# T2 计划 — `TokenMeter.measure()` memo 缓存（同 revision + 同输入复用已冻结结果）

> 流程 4.2 规划者产出。**只规划，不改代码**（本目录唯一写入产物：`plan.md` + `steps.md`）。
> 环境无法 build/run（依赖未装、无网络）→ 所有结论为**静态/结构核对**，标记「未做运行时验证」。
> 行号对应本次快照（2026-10-02），目标文件：
> `sandbox/deepseek-harness/packages/llm/token-meter/src/index.ts`

---

## 1. 优化目标

把 `TokenMeter.measure()` 从「**每次调用都全量重算 + `structuredClone` + `deepFreeze` 整个 surface**」改为
「**按 (Session, logRevision, header 等值, route pricing 身份, llm 服务身份) 为键做单槽 memo；命中时直接返回上一次已经 `deepFreeze` 过的 `TokenMeasurement`**」。

目标性质：**纯加速器，返回值逐字段等价、签名不变、调用方零改动、不引入新依赖、改动自包含于 `index.ts` 单文件**。

直接动因（来自 `SOURCE_ANALYSIS.md` A2-4 / A3-1）：一次压缩流程里 `measure()` 被调用 4~6 次，
而其中至少 1~2 次（稳定路径：`region.ts:386` 与 `region.ts:456`）处在**完全相同的输入**上——
它们之间没有任何事件落账，重算纯属浪费。

---

## 2. 当前实现逐行分析

### 2.1 `measure()` 原文要点（`index.ts:146-191`）

```ts
146  measure(session: Session, requestHeader?: EpochHeader): TokenMeasurement {
147    const state = this._sync(session)                       // ① 增量 fold：只消费新事件，O(Δ)
148    const header = requestHeader === undefined
149      ? state.header
150      : canonicalHeader(requestHeader)                      // ② override 时规范化
151    const pricing = this._routeImagePricing(header)         // ③ ctx.get('llm') + adapter 查表
152    const fileText = this._fileRequestText()                // ④ ctx.get('llm') + 闭包
153    const surface = priceSurface(state.surface, pricing, fileText)   // ⑤ O(S)，S = surface 节点数
154    const anchor = state.anchor
156-181 ... baseline / surfaceDeltaTokens 三段分支（anchor 命中 / 空 / 估计）...
183    return deepFreeze(structuredClone({                     // ⑥ 全量深克隆 + ⑦ 全量深冻结，均 O(S)
184      logRevision: state.consumedEvents,
185      baseline,
186      surfaceDeltaTokens,
187      totalTokens: Math.max(0, baseline.tokens + surfaceDeltaTokens),
188      surfaceTokens: surface.surfaceTokens,
189      nodes: surface.nodes,
190    }))
191  }
```

**每次调用的 7 项成本**（`S` = surface 节点数，`T` = 工具 schema 规模，`Δ` = 自上次读取后新增事件数）：

| # | 成本项 | 量级 | 备注 |
|---|---|---|---|
| ① | `_sync()` | O(Δ) | 已有增量 fold，通常 Δ=0 |
| ② | `canonicalHeader(requestHeader)` | O(T) | 仅 override 时 |
| ③ | `_routeImagePricing()` | O(1) | 两次 map 查找 |
| ④ | `_fileRequestText()` | O(1) | 每次新建闭包 |
| ⑤ | `priceSurface()` | **O(S)** | `route-pricing.ts:32`；每个 image/file 还要跑 `estimateContent` |
| ⑤b | `estimateToolsTokens(header)`（L164/L178） | **O(bytes(tools))** | `estimate.ts:99` 对**全部工具 schema** 做 `JSON.stringify`；base bundle 几十个工具，是最贵的常数项 |
| ⑥ | `structuredClone(...)` | **O(S)** | 整个 `nodes` 数组 |
| ⑦ | `deepFreeze(...)` | **O(S)** | `util/values` 全量遍历 + `WeakSet` |

`measure()` 的 JSDoc 自己就写着："Every call clones those positional nodes, so measurement is O(surface)."（L140）——
这正是本次要消除的成本。**⑤⑥⑦ 全部可以在命中时被跳过。**

### 2.2 调用链与「4~6 次」的来源

生产代码里 `measure()` 的调用点（全仓 grep，排除 tests）：

| 位置 | 场景 | 入参 |
|---|---|---|
| `packages/compaction/compaction-basic/src/index.ts:278` | `compactIfNeeded()` 开头，每步 `agent/pre-step` 都跑 | `(session)` |
| `.../compaction-basic/src/index.ts:297` | context-overflow 分支：prune 后再测 | `(session)` |
| `.../compaction-basic/src/index.ts:325` | pressure 分支：prune 后再测 | `(session)` |
| `.../compaction-basic/src/index.ts:339` | 一次 `compactRegion()` 之后重测 | `(session)` |
| `.../compaction-basic/src/index.ts:396` | `compactNow()` 选区间 | `(session)` |
| `.../compaction-basic/src/region.ts:386` | `prepareCompaction()`：摘要前快照 | `(session)` |
| `.../compaction-basic/src/region.ts:456` | `assertWholeSurfaceUnchanged()` | `(session)` |
| `.../compaction-basic/src/region.ts:484` | `assertSelectedSpanStable()` | `(session)` |
| `packages/acp/acp/src/updates.ts:99` | ACP 用量上报 | `(session)` |

**关键事实**：`region.ts` 的 `compactSurfaceRegion()` 在一次事务里连续调用两次 `measure()`，中间**不落账任何事件**：

```ts
242    const prepared = prepareCompaction(dependencies, session, selection)   // → measure() @386
243    const summarized = await summarizeCompaction(... , assertStable, signal) // 只做 LLM 调用，不 append
252    if (options.owner === null) signal?.throwIfAborted()
253    assertStable(dependencies, session, summarized)                        // → measure() @456 / @484
255    const pending = commitCompactionBody(session, startEvent, summarized)  // 之后才 append
```

→ **386 与 456 的 `state.consumedEvents` 完全相同、header 完全相同、pricing/llm 完全相同** ⇒ 第二次是全量重算的纯浪费。
同理，`for(;;)` 重试路径（`region.ts:424 assertStable` → `426 prepareCompaction`）每轮也产生一对同输入调用。

一次 pressure 压缩（pruner 已挂载、`stability:'whole-surface'`、0 次 retry）的调用/重算对照：

| 调用 | revision 相对上一次 | 是否可命中 |
|---|---|---|
| `index.ts:278` | — | 计算（第 1 次） |
| `index.ts:325`（prune 后） | prune 追加事件 → revision 变 | 计算（第 2 次） |
| `region.ts:386` | `compaction/start` 已 append → revision 变 | 计算（第 3 次） |
| `region.ts:456` | **与 386 完全相同** | **命中 ✅** |
| `index.ts:339` | compaction 已提交 → revision 变 | 计算（第 4 次） |

⇒ 5 次调用 → 4 次重算；每多一次 retry 再省 1 次（2 次调用 → 1 次重算）。

### 2.3 已有增量模式参照（`_sync()`，`index.ts:218-240`）

```ts
219  private _sync(session: Session): ReplayState {
220    let state = this.states.get(session)          // WeakMap<Session, ReplayState>
221    if (state === undefined) { state = {...}; this.states.set(session, state) }
232    while (state.consumedEvents < session.seq) {  // 只消费 [cursor, seq)
235      const event = session.eventAt(SessionSeq(state.consumedEvents))!
236      this._foldEvent(state, event)
237      state.consumedEvents = SessionLogOffset(state.consumedEvents + 1)
238    }
239    return state
240  }
```

可复用的三条既有设计（本次 memo 必须与之同构）：
1. **`WeakMap<Session, …>` 做 per-session 状态**，随 Session 一起 GC，不需要容量管理；
2. **以 `consumedEvents`（= 对外暴露的 `logRevision`）为游标**判断"有没有新东西"；
3. **状态只在 `_foldEvent()` 里被改写**（纯函数：event + state → state），因此 `(session, revision)` 唯一决定 `state`。

---

## 3. 优化方案

### 3.1 键设计依据（对应验收标准 2：键必须覆盖所有影响结果的输入）

**第一步：证明 `(Session, revision)` 唯一决定 `state`。**
`ReplayState` 的 5 个字段（`consumedEvents / header / surface / stepStart / anchor`）只被两处写：
- `_sync()` L221-229：新建默认 state（`consumedEvents = 0`）；
- `_foldEvent()` L247-329：唯一的状态转移，且是**纯函数**——它只调用 `canonicalHeader`、`planSurfaceTokens`/`commitSurfaceTokens`（`surface-fold.ts`，纯）、`estimateContent`/`estimateMessage`/`estimateToolsTokens`（`estimate.ts`，纯）、`assembleAssistantStream`（纯）。

⇒ 对同一个 `Session`，`revision = state.consumedEvents` 一旦相同，`state` 的所有字段必然相同（append-only 日志 + seq 单调）。**`revision` 是 state 的完备键。**

**第二步：枚举结果字段的全部输入。**

| 结果字段 | 依赖链 | 入键项 |
|---|---|---|
| `logRevision` | `state.consumedEvents` | ✅ `revision` |
| `nodes` / `surfaceTokens` | `state.surface` + `pricing` + `fileText` | ✅ `revision` + `pricing` + `llm` |
| `baseline`（三分支） | `state.anchor` + `header`（`optionalHeaderEquals`）+ `estimateToolsTokens(header)` + `pricing` + `fileText` | ✅ `revision` + `header` + `pricing` + `llm` |
| `surfaceDeltaTokens` | `surface.surfaceTokens` − anchor 重定价（`pricing`/`fileText`） | ✅ 同上 |
| `totalTokens` | 上两者派生 | —（已被覆盖） |

**第三步：各项为什么这样比较。**

| 键项 | 比较方式 | 理由 |
|---|---|---|
| `Session` | `WeakMap` 键（引用相等） | 与 `states` 同构；跨 session 绝不复用（`route-pricing.spec.ts:234` 的 `restored` 是另一个 Session 对象） |
| `revision` | `===`（`SessionLogOffset` 数值） | 见"第一步"；同时它就是结果里的 `logRevision`，命中时天然一致 |
| `header` | `optionalHeaderEquals()`（**已有** helper，L79） | 见下方"等值 ⇒ 同结果"论证 |
| `pricing` | `===`（对象引用） | `_routeImagePricing()` 返回 adapter 声明的 `LlmImageRequestPricing` 对象，身份稳定；adapter 重注册/换对象必然换引用 → 自动失效 |
| `llm` | `===`（服务实例引用） | `ctx.get('llm')` 决定 `fileText` 是否存在；服务挂载/卸载必换引用 → 自动失效 |

**为什么 `header` 用等值（`headerEquals`）而不是引用相等：**
`canonicalHeader(requestHeader)` 每次都产出新对象，用 `===` 会让 override 路径永不命中。而 `headerEquals` 等值 ⇒ 结果必相同，因为 `header` 只通过 4 个口子影响结果：
- `pricing` 只读 `header.config.provider/model` → `callConfigEquals`（`llm/src/call-config.ts:49`）比较 `provider/model/reasoningEffort/temperature/maxTokens/stop` ⇒ 等值；
- `estimateToolsTokens(header)` 只读 `header.tools` → `headerEquals` 用 `sameSchema` 逐项 + 长度比较 ⇒ 等值；
- anchor 判定用 `optionalHeaderEquals(anchor.header, header)` ⇒ 等值 ⇒ 同布尔；
- `header === undefined` 判定（L172）→ `optionalHeaderEquals` 的 `undefined` 分支已处理（`requestHeader` 未传且 `state.header` 为 `undefined` 时，memo 里存的也是 `undefined`）。

未被 `headerEquals` 比较的 header 字段：`system`（类型为 `never`，不存在）与 `adapterDefaults` 的其它字段——它们**不参与任何计价**，故不影响等价性。

**为什么不用 `fileText` 入键：** `_fileRequestText()` 每次新建闭包 ⇒ 引用恒不等 ⇒ 缓存永不命中。改用其来源 `llm` 服务实例身份。

### 3.2 伪码（唯一实现形态）

新增类型（紧随 `ReplayState`，`index.ts:68` 之后）：

```ts
/** One cached `measure()` result, keyed by the exact inputs it was priced from. */
interface MeasurementMemo {
  /** Log revision the replay fold had consumed when `result` was priced. */
  readonly revision: SessionLogOffsetType
  /** Effective envelope `result` was priced under. */
  readonly header: EpochHeader | undefined
  /** Routed image pricing object `result` was priced under. */
  readonly pricing: LlmImageRequestPricing | undefined
  /** Mounted LLM service that projected `result`'s file text. */
  readonly llm: LlmRuntime | undefined
  /** Deeply frozen measurement, shared verbatim with every hit. */
  readonly result: TokenMeasurement
}
```

新增字段（紧随 `states`，`index.ts:108` 之后）：

```ts
  private readonly measurements = new WeakMap<Session, MeasurementMemo>()
```

`measure()` 改写（只加/改 12 行，其余原样）：

```ts
  measure(session: Session, requestHeader?: EpochHeader): TokenMeasurement {
    const state = this._sync(session)                 // ① 必须先 fold：malformed 事件要每次都抛
    const header = requestHeader === undefined
      ? state.header
      : canonicalHeader(requestHeader)
    const llm = this.ctx.get('llm')
    const pricing = this._routeImagePricing(header)
    const memo = this.measurements.get(session)
    if (memo !== undefined
      && memo.revision === state.consumedEvents
      && memo.llm === llm
      && memo.pricing === pricing
      && optionalHeaderEquals(memo.header, header)) {
      return memo.result                              // ② 命中：复用已冻结结果，跳过 ⑤⑥⑦
    }
    const fileText = this._fileRequestText()
    const surface = priceSurface(state.surface, pricing, fileText)
    const anchor = state.anchor
    /* …L156-181 三分支原样不动… */
    const result = deepFreeze(structuredClone({
      logRevision: state.consumedEvents,
      baseline,
      surfaceDeltaTokens,
      totalTokens: Math.max(0, baseline.tokens + surfaceDeltaTokens),
      surfaceTokens: surface.surfaceTokens,
      nodes: surface.nodes,
    }))
    this.measurements.set(session, {
      revision: state.consumedEvents,
      header,
      pricing,
      llm,
      result,
    })
    return result
  }
```

> 注：`_fileRequestText()`（L200-206）**保持原样不动**，仅为取 `llm` 身份多加一次 `this.ctx.get('llm')`（一次 map 查找，可忽略）。
> 备选（不推荐，diff 更大）：把 `fileText` 构造内联进 `measure()` 并删除 `_fileRequestText()`。

### 3.3 失效条件（穷举）

| 变化 | 命中判定 | 结果 |
|---|---|---|
| 任何事件落账（`session.append`） | `revision` 变 | **失效** |
| `image/offload` 重定价 | 是事件 → `revision` 变 | **失效** |
| `request/header` 变更 | `revision` 变（且 `header` 变） | **失效** |
| `requestHeader` 参数变（override 换 model/tools） | `headerEquals` 不等（`callConfigEquals`/`sameSchema`） | **失效** |
| `requestHeader` 从"未传"变"传入"（或反之） | `optionalHeaderEquals` 的 `undefined` 分支 | **失效** |
| adapter 重注册 / 换 `priceImages` 对象 | `pricing` 引用变 | **失效** |
| `llm` 服务挂载/卸载/重建 | `llm` 引用变（同时 `pricing` 变） | **失效** |
| Session 对象销毁 | WeakMap 条目随之回收 | 自动释放 |
| `TokenMeter` 服务 dispose / fiber 重载 | 整个 `measurements` 表随之丢弃 | 自动释放（对应 `token-meter.spec.ts:636-639` 的 reload 场景） |

**保守 miss-safe 的已知情形**：`compaction/start`、`step/start`、`llm/retry`、`turn/start` 等**非 surface 事件**也会 bump `revision` ⇒ 缓存失效，尽管 `nodes` 实际未变。这是**故意的保守选择**：`logRevision` 是返回值的一部分，若按"surface generation"入键就必须改写 `logRevision` 并重新构造结果对象（见 3.6-B，已否决）。方向是"宁可多算，不可错复用"。

### 3.4 内存占用与生命周期管理：`WeakMap` vs `Map` + 容量上限

**结论：用 `WeakMap<Session, MeasurementMemo>`，每 session 单槽，不做容量上限、不做 LRU。**

| 方案 | 评估 |
|---|---|
| **A. `WeakMap` + 单槽（选定）** | 条目数 = **被测量过的活跃 session 数**；单条占用 = 一份 `nodes`（`{seq, tokens, heuristicTokens}` 三元组 × S）+ 3 个引用（header/pricing/llm）+ 结果壳。比同一 session 已有的 `ReplayState.surface`（节点还带 `images[]`/`files[]` 数组）**更小**，即每 session 内存增量 < 现有 fold 状态的 1 倍。Session 不可达即自动回收；`TokenMeter` dispose 即全表释放。**零新增分支**。 |
| B. `Map` + 容量上限 + LRU | ✗ 否决：① `Map` 强引用 `Session` ⇒ 长会话进程里 session 永不释放，这是**新增泄漏**；② 淘汰分支（`size > cap` / `delete oldest`）在现有测试中**不可达**，会直接触发本项目 CI 的 **per-file 100% 分支门禁**（见第 6 节）。 |
| C. 把 memo 字段挂进 `ReplayState` | 可行（省一个 WeakMap），但把"fold 状态"与"读缓存"两种生命周期混在一个对象里，且回退时要动 `ReplayState`。作为备选，非首选。 |
| D. 多槽（按 header 分桶） | ✗ 否决：命中判定需对所有槽跑 `headerEquals` ⇒ O(K·T)，且必然需要淘汰策略（同 B 的门禁问题）。压缩路径上**同一 revision 只用一种 header**（全部是 `measure(session)`，不传 override），单槽已覆盖真实热点。 |

单槽的已知代价：若调用方在同一 revision 交替使用两种不同 header（如 `measure(s)` 与 `measure(s, h)` 交替），会来回 thrash → 命中率 0，但**正确性不受影响**（只会退化为每槽重算）。当前生产代码无此模式。

### 3.5 两条必须遵守的实现约束（否则语义破坏）

1. **memo 查询必须在 `this._sync(session)` 之后。**
   `_foldEvent()` 的"malformed 事件每次都抛、且不推进游标"是硬不变式，被 `token-meter.spec.ts:462-465` 的 `expectRepeatedFailure()`（连测两次都要抛）直接断言。若把查询放在 `_sync` 之前，命中缓存会**吞掉本该抛出的错误**。
2. **写缓存必须在 `deepFreeze(structuredClone(...))` 成功之后。**
   失败路径不写缓存 ⇒ 重复调用重复失败，行为不变（`_foldEvent` 抛错时 `state.consumedEvents` 不动，`priceSurface` 抛错时也不会到达写点）。

### 3.6 已否决的备选

- **A. 缓存 key 里放下 `revision`、改用 "surface generation"**：可让 `region.ts:386` 命中 `index.ts:325`（`compaction/start` 不改 surface）。但 `logRevision` 是返回值字段，命中后必须重造结果对象 → 要么再 `structuredClone` 一次 nodes（O(S)，省不掉最大成本），要么用 `Object.freeze` 只冻外壳（换了冻结机制 + 新增分支，风险 > 收益）。**否决，留作后续。**
- **B. 顺手把 `estimateToolsTokens` 也 memo 掉**（`SOURCE_ANALYSIS` A3-2）：落在 `estimate.ts`，**违反"仅改 `index.ts` 单文件"约束**。否决。
- **C. 用 `/* v8 ignore */` 掩盖命中分支**：项目里确有此注释，但那是给"结构上不可达"用的；命中分支是**可达且应当可达**的，掩盖等于放弃验证。否决（除非静态核对证明无任何测试命中，届时须上报主 Agent 裁定，而不是自行加 ignore）。

---

## 4. 预期收益（before / after）

### 4.1 行为说明

| 维度 | before | after |
|---|---|---|
| 返回**值** | `deepFreeze(structuredClone({...}))` | 命中时返回**上一次构造的同一个冻结对象**；字段值逐位相同 |
| 返回**身份** | 每次一个新对象（两份深拷贝） | 同一 (revision, 输入) 下 `Object.is(a, b) === true`（**唯一可观察差异**） |
| 可变性 | deep frozen | 同一个 deep frozen 对象（共享安全：`TokenMeasurement` 全字段 `readonly`，`nodes` 与 `nodes[i]` 均已冻结） |
| 抛错行为 | malformed 事件每次都抛 | **完全不变**（throw 发生在 `_sync` 内，早于 memo 查询；失败不写缓存） |
| 复杂度 | O(Δ) + O(S) + O(tools bytes) + O(S) + O(S) | 命中时：O(Δ) + O(T)（`headerEquals`，不 stringify）；未命中时与 before 相同 + O(1) 写入 |

### 4.2 复杂度与调用次数

| 成本项 | before | after（命中） |
|---|---|---|
| `_sync` | O(Δ) | O(Δ)（通常 0） |
| `canonicalHeader` | O(T)（仅 override） | 同 |
| `priceSurface` + 逐 image/file `estimateContent` | **O(S)** | 0 |
| `estimateToolsTokens` = `JSON.stringify(所有工具)` | **O(bytes(tools))** | 0 |
| `headerEquals`（O(T) 结构比较，无 stringify） | 0 | O(T) |
| `structuredClone` | **O(S)** | 0 |
| `deepFreeze`（全量遍历 + WeakSet） | **O(S)** | 0 |

调用次数（一次 pressure 压缩，pruner 已挂载、0 次 retry）：**5 次调用 → 4 次重算**（省 1 次全量测量）；
每次 retry 再省 1 次（2 次调用 → 1 次）；`compactNow()` / `assertSelectedSpanStable` 路径同理省 1~2 次。
长会话中 S 可达数千、工具 schema 数十 KB，单次全量测量是"数千节点遍历 + 数十 KB stringify + 两次 O(S) 遍历"，
故省掉 1~3 次即为该路径上一个显著的常数因子收益（具体倍数**未做运行时测量**）。

---

## 5. 风险与回退

| # | 风险 | 等级 | 缓解 | 回退 |
|---|---|---|---|---|
| **R1** | **键覆盖不全 → 错误复用**（最严重）：漏掉某个影响结果的输入，返回过期测量 | 中 | 键 = Session + revision + header 等值 + pricing 引用 + llm 引用，五项**全等**才命中；3.1 节逐字段论证了完备性；3.3 节穷举失效；方向保守（非 surface 事件也失效） | 删除 3 处代码（字段、`if` 命中块、`set`）即回到原实现，其余零改动 |
| **R2** | **缓存生命周期/内存**：长会话每 session 多留一份 nodes 副本 | 低 | 每 session 单槽 + WeakMap（随 Session 回收）；单条比现有 `ReplayState.surface` 更小；`TokenMeter` dispose 全表释放；不做 `Map`+容量上限（避免泄漏与不可达分支） | 同上 |
| **R3** | 外部可变依赖在同 revision 内变化：adapter 的 `priceImages` 对象**内部**有状态，或 `fs`/`attachments` 路径映射中途改变（`fileRequestText` 依赖它们） | 低 | 与既有 `states` fold 的假设**同级**（fold 也把历史节点的定价结果缓存到下一次调用）；`fileRequestText` 是"确定性 handle 文本"（`llm/src/index.ts:686`，不读文件内容）；pricing/llm 引用变化已入键 | JSDoc 记录前提（与 T1 处理 `cursor > session.seq` 的手法一致：**用文档而非分支**） |
| **R4** | **命中分支未被现有测试覆盖 → CI per-file 100% 门禁失败** | 中 | 已静态定位两条必然命中的路径：① `region.ts:386`→`456/484`（同 revision，`compaction-basic.spec.ts:267` 有 stub summarizer，成功路径必走到 `region.ts:253`）；② `token-meter.spec.ts:338` 等"同 revision 连续两次 measure"用例。若实施者静态核对后仍不确定 → **上报主 Agent**，不自行加 `v8 ignore` | 除非放宽"单文件"约束去补测试（需主 Agent 裁定） |
| R5 | 调用方依赖"每次返回新对象"（身份敏感） | 极低 | 全仓 9 个生产调用点均为只读：`nodes.slice()`、`isDeepStrictEqual`、`reduce`、`totalTokens`；返回值 deep frozen，任何写入都会 `TypeError`，不存在"被某一方改坏共享对象" | — |
| R6 | `region.ts:457` `isDeepStrictEqual(current.nodes, prepared.measurement.nodes)` 命中时变成同一引用比较 | 低 | 语义一致：命中 ⇔ revision 未变 ⇔ surface 未变 ⇔ 该断言本应通过；若 surface 真变了，revision 必变 → 不会命中 | — |

---

## 6. CI per-file 100% 覆盖门禁的影响（**必须单列**）

**门禁事实（已核实）**
- `vitest.config.ts`：`coverage.include = ['packages/*/*/src/**/*.{ts,tsx}']`，`exclude` 名单中**不含** `packages/llm/token-meter/src/*` ⇒ `index.ts` 受门禁约束。
- `scripts/coverage-uncovered-locations.cjs:115`：`"Uncovered locations (per-file 100% gate)"` ⇒ **statement / branch / function 三者都要 100%**，该文件不得有任何未覆盖位置。
- `scripts/coverage-exempt.ts` 的豁免清单里没有 compaction / token-meter 的测试 ⇒ 这些测试**参与**覆盖率统计。

**本次新增的运行时元素与覆盖来源**

| 新增元素 | 类型 | 覆盖来源（静态推断） |
|---|---|---|
| `interface MeasurementMemo` | 类型 | 无运行时，不计入 |
| `private readonly measurements = new WeakMap(...)` | statement | 构造器执行即覆盖（每个 `new TokenMeter()`） |
| `const llm = this.ctx.get('llm')` | statement | 每次 `measure()` 都执行 |
| `const memo = this.measurements.get(session)` | statement | 同上 |
| `if (命中)` 的 **true** 路 | branch | `region.ts:386`→`456/484` 同 revision 双调用（`compaction-basic.spec.ts` 成功压缩用例）；`token-meter.spec.ts:338-341` 类同 revision 双调用 |
| `if (命中)` 的 **false** 路 | branch | 所有"追加事件后再 measure"（`token-meter.spec.ts:208/250/261/358/433`）与"换 header 再 measure"（`token-meter.spec.ts:275/388-395`、`route-pricing.spec.ts:181/188/199`） |
| `&&` 的 4 个操作数（`revision`/`llm`/`pricing`/`header`） | branch | 命中路径上全部被求值；`llm === undefined` 场景（`token-meter.spec.ts` 的 `meter()`，未挂 llm）与已挂载场景（`route-pricing.spec.ts` 的 `harness()`）**都**存在 ⇒ 两侧都覆盖 |
| `this.measurements.set(...)` | statement | 所有未命中路径执行 |
| `optionalHeaderEquals`（已有 helper） | function | 已在用，本次只是多一个调用点 |

**因此：本方案不需要任何 `/* v8 ignore */`、不需要任何不可达分支、不需要新增测试文件。**
这也是 3.4 否决"Map + 容量上限"、3.6 否决"surface generation 键"的核心原因之一——两者都会引入**结构上不可达**的分支。

**给实施者的门禁自检**：改完后逐条清点 `index.ts` 中新增的 `if / && / ? : / ?? / ?.`，确认每一条的两个方向（或每个操作数）都能在既有测试里被走到；做不到就停下上报，不要靠 ignore 绕过。

---

## 7. 验收对照表

| # | 验收标准 | 本方案对应 | 静态核对方法 |
|---|---|---|---|
| **1** | 为 `measure()` 引入 memo 缓存，按"唯一决定结果的输入组合"为键，命中时**复用已冻结结果**，避免重复 `structuredClone` + `deepFreeze` | 3.1/3.2：键 = Session + revision + header 等值 + pricing 引用 + llm 引用；命中路径 `return memo.result`，直接跳过 `priceSurface`（O(S)）、`estimateToolsTokens`（tools stringify）、`structuredClone`（O(S)）、`deepFreeze`（O(S)） | 读改后 `measure()`：确认 `return memo.result` 出现在 `structuredClone` 之前；确认 `memo.result` 来自 `deepFreeze(...)` 的返回值 |
| **2** | 不产生错误复用：键覆盖所有影响结果的输入；输入变化必失效 | 3.1 三步论证（结果字段 → 依赖 → 入键项）+ 3.3 失效条件穷举表（9 条）+ 3.1「`(session, revision)` 唯一决定 state」证明 | 逐字段走查 `measure()` 的返回值构造表达式，确认每个自由变量都能追溯到 5 个键项之一 |
| **3** | 仅改 `index.ts` 单文件、无新依赖、TS 风格与现有 import/命名一致 | 3.2：仅新增 1 个 interface + 1 个字段 + `measure()` 内 12 行；**不新增任何 import**（`LlmRuntime`/`LlmImageRequestPricing`/`TokenMeasurement`/`EpochHeader`/`SessionLogOffsetType` 全部已在 L11/L13-25/L40 导入）；字段声明风格照抄 `states`（L108）；`readonly` 与项目一致 | `git diff --stat` = 1 file；`git diff` 中 grep `^+import` 应为空 |
| **4** | 保留导出与调用方语义：返回值等价、签名不变、调用方零改动 | 4.1 行为表：签名 `measure(session, requestHeader?)` 不变；返回值字段值等价；唯一差异是同一输入下返回同一引用（readonly + deep frozen，9 个生产调用点全只读）；抛错行为不变（3.5 两条约束） | grep 全仓 `.measure(` 确认 9 个调用点无需改；确认 `_sync` 仍在 memo 查询之前；确认 `set` 在 `deepFreeze` 之后 |
| **5** | 附 before/after 行为说明与最小静态核对（键覆盖性分析 + 等价性走查 + 复杂度/调用次数收益） | 第 4 节（行为 + 复杂度表 + 5→4 次调用）+ 3.1（键覆盖性）+ 3.3（失效）+ 第 7 节本表；`steps.md` 给出每步的静态核对动作与 `git diff --stat` 核验 | 按 `steps.md` 逐步执行并记录 |

---

## 8. 最小静态核对清单（实施者照做，本环境无法 build/run）

1. **键覆盖性**：对 `measure()` 返回值里的 6 个字段逐个回溯自由变量，写进 `implementation.md` 的三列表（字段 / 依赖 / 入键项）。
2. **等价性走查**：确认 `if` 命中块只做 `return memo.result`；确认 `memo.result` 的唯一来源是本次 `deepFreeze(structuredClone(...))`；确认三分支（anchor / 空 / 估计）代码**一字未动**。
3. **fail-loud 不变式**：确认 `this._sync(session)` 是 `measure()` 的第一条语句；`expectRepeatedFailure`（`token-meter.spec.ts:462`）的两个用例仍能每次抛错。
4. **风格**：新增代码无 `!`、无 `any`、无非空断言、无新增 import；`readonly` 字段；JSDoc 一句说明共享冻结结果。
5. **门禁自检**（第 6 节表逐条）：清点新增分支的两个方向是否都有测试路径。
6. **范围核验**：`git diff --stat` 必须只有 `packages/llm/token-meter/src/index.ts` 一个文件；`git status` 确认 `sandbox/deepseek-harness/` 之外无改动。
7. **未做运行时验证**：本环境无法 `pnpm test` / `tsc`，须在 `verification.md` 显式标注"未做运行时验证"，并列出应跑的套件：`packages/llm/token-meter/tests/*`、`packages/compaction/compaction-basic/tests/*`、`packages/compaction/command-compact/tests/*`。

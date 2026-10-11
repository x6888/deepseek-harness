# T2 实现说明 — `TokenMeter.measure()` memo 缓存

> 流程 4.3（改码者）产出。目标文件（本轮唯一改动）：
> `packages/llm/token-meter/src/index.ts`
> 环境无法 build/run（依赖未装、无网络）→ 全部结论为**静态/结构核对**，标注 **未做运行时验证**。
>
> **第 2 轮修复（本文件的增量）**：第一轮验证发现阻断级缺陷 **F1** —— 键中的 `pricing`
> 用引用比较，而生产路径上每次 `_routeImagePricing()` 都返回新对象 ⇒ memo 永不命中。
> 主Agent 裁定：**保留 `pricing` 入键，把引用比较改为值比较**（新增 `pricingKey(pricing)`）。
> 本文已按裁定更新：键覆盖性（§4）、门禁自检（§5）、收益结论（§6）。**未加任何 `v8 ignore`。**

---

## 1. 一句话改动

在 `TokenMeter` 上新增一张 `WeakMap<Session, MeasurementMemo>` 单槽缓存，`measure()` 在 `_sync()` 之后按
「Session 引用 + `state.consumedEvents` + header 等值 + `pricing` **值**（`pricingKey`） + `llm` 引用」五项全等命中时
直接返回上一次 `deepFreeze` 过的 `TokenMeasurement`，从而跳过 `priceSurface` / `estimateToolsTokens` /
`structuredClone` / `deepFreeze`；未命中路径维持原样，并在冻结成功后写缓存。

---

## 2. diff 摘要

相对 HEAD，`git diff -- packages/llm/token-meter/src/index.ts` 共 **+73 / -1**（第 1 轮 5 处 + 第 2 轮 2 处）：

| # | 位置 | 改动 | 轮次 |
|---|---|---|---|
| 1 | `index.ts:70-93`（`ReplayState` 之后） | 新增 `interface MeasurementMemo`（5 个 `readonly` 字段 + JSDoc 说明键设计依据，含"为何不用 `fileText` 入键"） | 1 |
| 2 | `index.ts:112-128`（`optionalHeaderEquals` 之后） | **第 2 轮**：新增 `pricingKey()` —— 把 `pricing` 由引用比较改为值比较（见 §4） | 2 |
| 3 | `index.ts:151-153`（`states` 之后） | 新增 `private readonly measurements = new WeakMap<Session, MeasurementMemo>()` | 1 |
| 4 | `index.ts:187-192` | `measure()` JSDoc 追加"同输入复用同一冻结结果 / 键的五项 / 非 surface 事件亦失效 / 随 Session 与服务释放" | 1（第 2 轮把 pricing 的描述改为"值比较"） |
| 5 | `index.ts:198-214` | `measure()`：`_sync` 后插入 `const llm = this.ctx.get('llm')`、`const memo`、`if (命中) return memo.result`；命中条件第 4 项由 `memo.pricing === pricing` 改为 **`memo.pricingKey === pricingKey(pricing)`** | 1 + 2 |
| 6 | `index.ts:246-261` | `return deepFreeze(structuredClone({...}))` → `const result = deepFreeze(structuredClone({...}))`，之后 `this.measurements.set(...)`（写 `pricingKey: pricingKey(pricing)` 而非 `pricing`），再 `return result` | 1 + 2 |

第 2 轮只动了 3 个点：新增 `pricingKey` helper、`MeasurementMemo.pricing` → `MeasurementMemo.pricingKey`、
命中条件与写点的两处调用。**`measure()` 的结构（fail-loud 顺序、三分支、`deepFreeze` 之后才写）一字未动。**

改动自包含 checklist：

- ☑ **零新增 import**：`SessionLogOffsetType` / `EpochHeader` / `LlmImageRequestPricing` / `LlmRuntime` /
  `TokenMeasurement` 均已在原文件 L11 / L13-18 / L28-32 导入；`git diff | grep '^+import'` 为空。
  `pricingKey` 只用内置的 `JSON.stringify`，**无新依赖**。
- ☑ 导出面未变：`export class TokenMeter`、`measure(session, requestHeader?)` 签名、`export type * from './types.ts'`
  等 5 处导出在 diff 中无 `^-` 行。
- ☑ 未触碰 `estimate.ts` / `surface-fold.ts` / `route-pricing.ts` / `types.ts` / 任何 `tests/`。
- ☑ 无 `v8 ignore`、无新增 `!` / `as` / `any` / `??` / `?.`。

### `git diff --stat` 原文

两轮累计（相对 HEAD）：

```
 packages/compaction/compaction-basic/src/region.ts | 105 +++++++++++++++------
 packages/llm/token-meter/src/index.ts              |  74 ++++++++++++++-
 2 files changed, 148 insertions(+), 31 deletions(-)
```

本轮（第 2 轮）改后 `-- index.ts`：统计由 `55 +++` 变为 `74 +++`，**即第 2 轮净增 19 行**（`pricingKey` 的 17 行 +
JSDoc 重排 2 行），`-` 行仍只有第 1 轮那 1 行（`return deepFreeze(` → `const result = deepFreeze(`）。

> `region.ts` 的改动是**上一轮 T1 遗留的未提交工作**，第 1 轮与第 2 轮都**未读写该文件**
> （106 行 / `105 +++---` 的统计两轮完全一致，可核验本轮零触碰）。
> 本轮新增的行只落在 `index.ts` 一个文件；`git status --porcelain` 的其余未跟踪项
> （`CONTEXT.md` / `MANIFEST.md` / `SOURCE_ANALYSIS.md` / `optimization/`）均为分析产物与归档目录。
> **未做 commit。**

---

## 3. Before / After

### 3.1 代码结构

```ts
// BEFORE
measure(session, requestHeader?) {
  const state = this._sync(session)
  const header = requestHeader === undefined ? state.header : canonicalHeader(requestHeader)
  const pricing = this._routeImagePricing(header)
  const fileText = this._fileRequestText()
  const surface = priceSurface(state.surface, pricing, fileText)
  /* …三分支 baseline/surfaceDeltaTokens（一字未动）… */
  return deepFreeze(structuredClone({ logRevision, baseline, surfaceDeltaTokens, totalTokens, surfaceTokens, nodes }))
}

// AFTER
measure(session, requestHeader?) {
  const state = this._sync(session)            // ① 仍为第一条语句：malformed 事件必须每次都抛
  const header = requestHeader === undefined ? state.header : canonicalHeader(requestHeader)
  const llm = this.ctx.get('llm')              // ② 新增：取 llm 服务实例身份（一次 map 查找）
  const pricing = this._routeImagePricing(header)
  const memo = this.measurements.get(session)
  if (memo !== undefined
    && memo.revision === state.consumedEvents
    && memo.llm === llm
    && memo.pricingKey === pricingKey(pricing)   // ③ 第 2 轮：pricing 由引用比较改为值比较
    && optionalHeaderEquals(memo.header, header)) {
    return memo.result                         // ④ 命中：跳过 priceSurface / estimateTools / clone / freeze
  }
  const fileText = this._fileRequestText()
  const surface = priceSurface(state.surface, pricing, fileText)
  /* …三分支 baseline/surfaceDeltaTokens（一字未动）… */
  const result = deepFreeze(structuredClone({ /* 6 个字段原样 */ }))
  this.measurements.set(session, { revision: state.consumedEvents, header, pricingKey: pricingKey(pricing), llm, result })  // ⑤ 冻结成功后才写
  return result
}

// 新增 helper（index.ts:112-128）
function pricingKey(pricing: LlmImageRequestPricing | undefined): string | undefined {
  return pricing === undefined ? undefined : JSON.stringify(pricing)
}
```

### 3.2 行为对照

| 维度 | before | after |
|---|---|---|
| 返回**值** | 每次 `deepFreeze(structuredClone(...))` 的新对象 | 命中时返回**上一次构造的同一个冻结对象**；字段值逐位相同 |
| 返回**身份** | 每次一个新对象 | 同一 (revision, 输入) 下 `Object.is(a, b) === true`（**唯一可观察差异**） |
| 可变性 | deep frozen | 同一个 deep frozen 对象；`TokenMeasurement` 全字段 `readonly`，`nodes` 与 `nodes[i]` 均已冻结 ⇒ 共享安全 |
| 抛错行为 | malformed 事件每次都抛 | **不变**：throw 发生在 `_sync()` 内、**早于** memo 查询；失败路径不到写点 ⇒ 不写缓存 |
| O(复杂度) | `_sync` O(Δ) + `priceSurface` O(S) + `estimateToolsTokens` O(bytes(tools)) + `structuredClone` O(S) + `deepFreeze` O(S) | 命中：O(Δ) + O(T)（`optionalHeaderEquals`，不 stringify）+ O(1)（`pricingKey`，对象只有方法成员，`JSON.stringify` 为常数级）；未命中：与 before 相同 + O(1) 写入 |

### 3.3 三条硬性走查（plan.md 第 8 节）

1. **fail-loud 不变式**：`const state = this._sync(session)` 仍是 `measure()` 首条语句（仅有前置注释行）。
   `token-meter.spec.ts:462-464` 的 `expectRepeatedFailure` 连测两次都要求抛错 ⇒ 命中缓存不可能吞掉错误；
   且失败时 `state.consumedEvents` 不推进、异常在函数中部抛出 ⇒ 不会到达 `set`。
2. **等价性走查**：命中体只有 `return memo.result`；`memo.result` 的唯一来源就是本次 `deepFreeze(structuredClone(...))`
   （文件中别无写入点）；三分支（anchor 命中 / `header===undefined && 0` / 估计）代码零改动；6 个返回字段原样。
3. **写点位置**：`this.measurements.set(...)` 位于 `const result = deepFreeze(...)` **之后**，即冻结成功之后。

---

## 4. 键覆盖性分析（逐个结果字段回溯）

`(Session, revision)` 唯一决定 `ReplayState`：`ReplayState` 的 5 个字段只被 `_sync()` 的初始化（L221-229）
与 `_foldEvent()`（唯一状态转移，纯函数式：只调用 `canonicalHeader` / `planSurfaceTokens` /
`commitSurfaceTokens` / `estimateContent` / `estimateMessage` / `estimateToolsTokens` / `assembleAssistantStream`，
全部纯）写入；日志 append-only、`seq` 单调 ⇒ **revision 是 state 的完备键**。

| 结果字段 | 依赖链（自由变量） | 入键项 |
|---|---|---|
| `logRevision` | `state.consumedEvents` | ✅ `revision`（且它本身就是返回值的一部分，命中时天然一致） |
| `nodes` | `priceSurface(state.surface, pricing, fileText)` → `state.surface` ⊕ `pricing` ⊕ `fileText` | ✅ Session+`revision` ⊕ `pricing`(**值**，`pricingKey`) ⊕ `llm`(引用，即 `fileText` 的来源) |
| `surfaceTokens` | 同 `nodes`（`surface.surfaceTokens`） | ✅ 同上 |
| `baseline`（分支条件） | `state.anchor`（是否被 Writing）⊕ `optionalHeaderEquals(anchor.header, header)` | ✅ Session+`revision` ⊕ `header`(等值) |
| `baseline`（值） | `estimateToolsTokens(header)` ⊕ anchor 重定价（`pricing`/`fileText`）⊕ `state.surface` | ✅ `header`(等值：tools 逐项 `sameSchema` + 长度) ⊕ `pricing` ⊕ `llm` ⊕ `revision` |
| `surfaceDeltaTokens` | `surface.surfaceTokens` − anchor 重定价结果 | ✅ 同上（`pricing`/`llm`/`revision`/Session） |
| `totalTokens` | `Math.max(0, baseline.tokens + surfaceDeltaTokens)`（派生） | ✅ 已被上面两项覆盖 |

**各项比较方式的依据**

| 键项 | 比较 | 理由 |
|---|---|---|
| `Session` | `WeakMap` 键（引用相等） | 与既有 `states` 同构；跨 session 绝不复用 |
| `revision` | `===`（`SessionLogOffsetType` 数值） | 见上：revision ⇒ state 全称决定 |
| `header` | `optionalHeaderEquals`（**已有** helper，L104）→ `headerEquals` → `callConfigEquals` + tools `sameSchema` | `canonicalHeader(requestHeader)` 每次新建对象，用 `===` 会让 override 路径永不命中；而等值 ⇒ 结果必相同（`pricing` 只读 `header.config.provider/model`，`estimateToolsTokens` 只读 `header.tools`，anchor 判定本身就用同一个 `optionalHeaderEquals`）；未参与计价的字段不入比较 |
| `pricing` | **`pricingKey()` 值比较**（新增 helper，L112-128；`JSON.stringify`） | **第 2 轮修复 F1**：adapter **每次调用都新建** pricing 对象（`deepSeekImageRequestPricing` 的两条出口均为新字面量，`llm-deepseek/src/request-pricing.ts:97/99`；`llm-replay/src/index.ts:881-890` 同理），引用比较恒 `false` ⇒ memo 生产永不命中。改为值比较后，同一路由的重复调用真正命中 |

**`pricingKey()` 的实际分辨力（必须知晓的前提）**

`LlmImageRequestPricing`（`llm/src/types.ts:212-220`）**只有一个成员 `priceImages` 方法**，
而 `JSON.stringify` 丢弃函数 ⇒ 任何"已声明定价"的对象都序列化为同一个 `"{}"`。
即：该键项只能区分「有 pricing」与「无 pricing（`undefined`）」，**不能**区分两个不同的 pricing 对象。
这不会造成跨路由误复用，因为路由身份由另外两项兜住：

- **model / provider**：`headerEquals` → `callConfigEquals`（`llm/src/call-config.ts:49-58`）逐字段比较 `provider` 与 `model`，
  而 pricing 正是由 `header.config.provider/model` 解析出来的（`_routeImagePricing`，L264-269）⇒ 换路由必 miss；
- **adapter / 服务来源**：`llm` 实例引用入键 ⇒ 换 `LlmRuntime` 实例必 miss；
- 例证：`route-pricing.spec.ts:176` 的 `harness(model => (model === 'vision' ? fixedPricing : substitution))`
  两个不同定价对象虽同键为 `"{}"`，但其测试的 model 不同 ⇒ `callConfigEquals` 不等 ⇒ 仍 miss，不会产生错复用。

⇒ 剩余缝隙只可能是：**同一 `llm` 实例、同一 provider/model、同一 revision 内**，pricing 的**行为**被环境改变
（deepseek adapter 的 `connection` 快照重连变更、`imageAccess` 解析路径变化）。这与 §4 末记录的前提（R3）同级，
以注释记录而非分支处理。**该缝隙在第 1 轮（引用比较）同样存在且当时还多一层"引用变化"的偶然兜底；
现值比较下它成为唯一兜底外的残留，特此明确上报主Agent。**
| `llm` | `===`（`ctx.get('llm')` 服务实例引用） | `fileText` 由 `llm.fileRequestText` 投影，每次 `_fileRequestText()` 都新建闭包（引用恒不等 ⇒ 不可入键），故取其来源的服务实例身份；挂载/卸载/重建必换引用 → 自动失效 |

**失效条件（穷举）**

| 变化 | 命中判定 | 结果 |
|---|---|---|
| 任意事件落账（`session.append`） | `revision` 变 | 失效 |
| `image/offload` 重定价 | 是事件 → `revision` 变 | 失效 |
| `request/header` 变更 | `revision` 变（header 亦变） | 失效 |
| `requestHeader` 参数变（换 model/tools/config） | `headerEquals` 不等 | 失效 |
| `requestHeader` 从未传→传入（或反之） | `optionalHeaderEquals` 的 `undefined` 分支 | 失效 |
| adapter 重注册 / 换 `priceImages` 对象、`llm.imageRequestPricing` 返回语义变了 | `llm` 未换实例时：**仅当** provider/model 也未变（否则 `headerEquals` 已失效）才依赖下列兜底——同一个 revision 内换 adapter 属同 revision 内的环境突变，见上文"实际分辨力"与 §4 末前提（以文档记录）；production 换 provider/model 的路径仍必然 miss | 保守方向由 `llm` + `header` 兜住 |
| `llm` 服务挂载 / 卸载 / 重建 | `llm` 引用变 | 失效 |
| Session 不可达 | WeakMap 条目回收 | 自动释放 |
| `TokenMeter` 服务 dispose / fiber 重载 | 整张 `measurements` 表随实例丢弃 | 自动释放（对应 `token-meter.spec.ts:636-640` 的 reload 场景） |

**保守 miss-safe**：`compaction/start`、`step/start`、`llm/retry`、`turn/start` 等非 surface 事件也会 bump `revision`
⇒ 缓存失效（尽管 `nodes` 实际未变）。这是故意的保守选择：`logRevision` 是返回值字段，若改用 "surface generation"
入键就必须重造结果对象（会重新引入 O(S) 成本），已在 plan 3.6-A 否决。

**前提（以 JSDoc/注释记录，而非分支）**：同一个 `llm` 实例在同一 revision 内，`fileRequestText` 是确定性
handle 文本（不读文件内容），与既有 fold 把历史节点定价结果缓存到下一次调用的假设同级；路由定价的解析结果
（`options` 快照 / `imageAccess`）在同一 revision 内不静默变更。第 2 轮起这一条从"理论缝隙"升格为
**唯一残留的风险面**（引用不再提供偶然兜底），已在 `pricingKey` 的 JSDoc 中写明。

---

## 5. CI per-file 100% 覆盖门禁自检（plan.md 第 6 节逐条）

门禁事实：`vitest.config.ts` 的 `coverage.include` 覆盖 `packages/*/*/src/**`，排除名单不含 token-meter；
`scripts/coverage-uncovered-locations.cjs:115` 要求 statement / branch / function **三者 100%**，无未覆盖位置。

| 新增元素 | 类型 | 覆盖来源（静态推断） |
|---|---|---|
| `interface MeasurementMemo` | 纯类型 | 无运行时位置，不计入 |
| `private readonly measurements = new WeakMap(...)` | statement | 每个 `new TokenMeter()` 执行（如 `token-meter.spec.ts` 的 `meter()`） |
| `const llm = this.ctx.get('llm')` | statement | 每次 `measure()` 都执行 |
| `const memo = this.measurements.get(session)` | statement | 同上 |
| `if (命中)` **true** 路 | branch | `token-meter.spec.ts:634`（listener 内的 `:616` 已在 revision=3 上写过 memo —— 注意：`:626` 的 measure 发生在 append **之前**，不是 rev=3 的那次，此处据第 1 轮验证的 F3 修正；`:633` 断言 `revisions` 仍为 `[3]` ⇒ 之后无 append、无 `request/header`、该 ctx 未挂 `llm` ⇒ 五项全等必命中）。第 2 轮起该结论**更强**：即使挂载了 llm，`pricingKey` 值比较也让同路由的重复调用命中 ⇒ `compaction-basic.spec.ts` 成功压缩路径上的 `region.ts:386 → 456/484` 双调用重新成为**可靠**证据 |
| `if (命中)` **false** 路 | branch | "追加事件后再 measure"（`token-meter.spec.ts:208/261/358/433`）、"换 header 后再 measure"（`:275`、`:388-395`；`route-pricing.spec.ts:181/188/196/199`）、首次 measure（memo `undefined`，`:166/191/226` 等） |
| `&&` 的 4 个操作数 | branch | 命中路径上全部被求值；`llm`/`pricing` 为 `undefined`（`token-meter.spec.ts` 的 `meter()` 未挂 llm）与为对象（`route-pricing.spec.ts` 的 `harness()` 已挂载）**两种**取值都存在；`memo === undefined` 的短路与全求值两侧均可达 |
| **`pricingKey` 函数本身** | function | 每次 `measure()`（命中或未命中路径除 `memo === undefined` 短路外）都会调用；最早的调用点非条件性（写点 L258 无条件执行 ⇒ 至少必执行一次） |
| **`pricingKey` 的 `undefined` 分支**（true 路） | branch | `route-pricing.spec.ts:120/192` 的 `harness(() => undefined)`、`token-meter.spec.ts` 未挂 llm 的 `meter()` |
| **`pricingKey` 的 `JSON.stringify` 分支**（false 路） | branch | `route-pricing.spec.ts:136/157/176/213/238/248/278` 的 `harness(() => fixedPricing | substitution | markedPricing | broken | {...})` ⇒ pricing 非 `undefined` |
| `this.measurements.set(...)` | statement | 所有未命中路径都会执行到 freeze 成功之后 |
| `optionalHeaderEquals` | function | 已有 helper，仅多一个调用点 |

⇒ 第 2 轮清点：**新增的运行时位置只有 `pricingKey` 一个两出口三元表达式 + 一个函数体，且两向都有既有测试直接覆盖**
（`undefined`：`route-pricing.spec.ts:120/192`；非 `undefined`：同文件 `:136/157/176/213/238/248/278`）⇒
**未发现任何不可达分支，因此不需要（也绝不使用）`v8 ignore`。**

⇒ **本实现未使用任何 `/* v8 ignore */`，也未引入结构上不可达的分支**（这也是不采用 `Map`+容量上限 LRU、
不采用多槽分桶的原因）。第 2 轮起命中路径证据从 1 条增强为多条：`token-meter.spec.ts:634`（`pricing === undefined`
场景）与 `region.ts:386 → 456/484`（挂载了 llm 的生产同路由场景，**第 2 轮后真正可靠**）。

---

## 6. 收益

| 成本项 | before | after（命中） |
|---|---|---|
| `_sync()` | O(Δ) | O(Δ)（通常 Δ=0） |
| `canonicalHeader`（仅 override） | O(T) | 同 |
| `priceSurface` + 逐 image/file `estimateContent` | O(S) | **0** |
| `estimateToolsTokens`（全工具 schema `JSON.stringify`） | O(bytes(tools)) | **0** |
| `optionalHeaderEquals`（结构比较，无 stringify） | 0 | O(T) |
| `pricingKey`（对象只有一个方法成员 ⇒ `JSON.stringify` 常数级，且因短路只在可能命中时求值） | 0 | O(1) |
| `structuredClone` | O(S) | **0** |
| `deepFreeze`（全量遍历 + WeakSet） | O(S) | **0** |

### 6.1 收益结论（第 2 轮重写 —— 第 1 轮的"零收益"结论作废）

第 1 轮结论"生产热路径命中率 ≈ 0、收益为零"**仅在 `pricing` 引用比较下成立，第 2 轮修复 F1 后不再成立**：

- **命中条件**（现值语义）：Session 引用 ∧ `revision` ∧ `llm` 实例 ∧ `pricingKey`（有/无定价）∧ header 等值。
- **生产压缩路径**（`llm` 已挂载、`request/header` 已落账）：`region.ts:386` 与 `:456/484` 之间无任何事件落账、
  同一个 `llm` 实例、同一个 provider/model（⇒ 同 `pricingKey` 值）、header 相同 ⇒ **五项全等 ⇒ 必然命中**。
- 因此 `plan.md` 2.2 的主收益点在**生产路径上真正成立**：一次 pressure 压缩（pruner 已挂载、0 次 retry）
  **5 次调用 → 4 次重算**（省 1 次全量测量：`priceSurface` O(S) + `estimateToolsTokens` 全工具 stringify
  + `structuredClone` O(S) + `deepFreeze` O(S)）；每多一次 retry 再省 1 次
  （`region.ts:424 assertStable` → `426 prepareCompaction`）；`compactNow()` / `assertSelectedSpanStable` 路径同理。
- 命中侧的新增成本为 O(T)（header 结构比较）+ O(1)（`pricingKey`），远小于被省下的三项 O(S)；**净收益为正**。
- 缓存不再依赖"`pricing === undefined`"这一偶遇场景：第 1 轮只有未挂载 llm 的测试场景能命中，
  第 2 轮在挂载 llm 的真实 agent 运行同样命中。

内存：每 session 单槽，条目 ≤ 活跃 session 数，单条（一份 `nodes` 三元组数组 + 3 个引用 + 结果壳）比同一 session
已有的 `ReplayState.surface`（节点还带 `images[]`/`files[]`）更小；随 Session 与 `TokenMeter` 实例回收。

> **未做运行时验证**：本环境无法 `pnpm test` / `tsc`（依赖未装、无网络）。上述收益与门禁结论均为静态推断，
> 实际倍数与命中率未测量。应跑套件：`packages/llm/token-meter/tests/*`、`packages/compaction/compaction-basic/tests/*`、
> `packages/compaction/command-compact/tests/*`；并复核 per-file 100% 覆盖率
> （`vitest.config.ts` + `scripts/coverage-uncovered-locations.cjs`）。

---

## 7. 与 plan.md / 主Agent 裁定的一致性

**两轮合并后的总判定**

- 完全按 `plan.md` 3.2「唯一实现形态」实现：键的 5 项、条件顺序（`memo !== undefined` → `revision` → `llm` →
  **`pricingKey`（原 `pricing`）** → `optionalHeaderEquals`）、`return memo.result` 位置、
  `const result = deepFreeze(...)` + `set` 均一致。
- 遵守主Agent 历次裁定：单槽 `WeakMap<Session, MeasurementMemo>` ✅；未用 `fileText` 闭包入键 ✅（改用 `llm` 实例身份）；
  memo 查询置于 `_sync()` 之后 ✅；写缓存置于 `deepFreeze` 成功之后 ✅；**未加任何 `v8 ignore`** ✅。
- **第 2 轮按主Agent 修复裁定**：`pricing` **保留入键**、比较方式由引用改为 `pricingKey()` 值比较 ✅；
  因不存在不可达分支，无需启用"停止并回报"流程（见 §5 清点）。
- 与 plan 的**唯一偏离有两处，均为主Agent 裁定或等价微调**，无冲突：
  1. `plan.md` 3.1「各项比较方式」表把 `pricing` 列为 `===`（对象引用）—— 该判断被第 1 轮验证的 F1 证伪，
     主Agent 裁定改为值比较；实现随之调整。**建议下一轮同步修订 `plan.md` 该行与其 3.2 伪码中的 `pricing`/字段注释**
     （本文不代为修改 `plan.md`，避免越权）。
  2. 插入位置微调：`const llm` 放在 `_routeImagePricing` 之前（让 3 个"外部定价输入"相邻），语义等价。
- **上报项（非阻断）**：`JSON.stringify` 对只有方法成员的 pricing 退化为 `"{}"` ⇒ 该项只能区分"有无定价"，
  路由discrimination 由 `header`（provider/model 等值）+ `llm`（实例引用）兜住；残留缝隙见 §4 末，已以注释记录。
- 与 plan 无冲突；唯一偏离是插入位置微调（`const llm` 放在 `_routeImagePricing` 之前，为让键的 3 个"外部定价输入"
  相邻，便于阅读），语义等价。

---
---

# 第 3 轮（R2 根源修复）— `LlmRuntime` 侧钉住定价 + token-meter 恢复引用比较

> 流程 4.3（改码者）产出。配套 `plan-r3.md` / `steps-r3.md`（同目录），**不覆盖**上文 §1–§7（第 1/2 轮历史）。
> **未做运行时验证**：本环境无 `node_modules`、无网络，无法 `pnpm test` / `tsc` / 覆盖率脚本；
> 全部结论为**静态/结构核对 + `Read` 复查语法**，未 commit。
> 行号 = 改后快照。

## R3-1 一句话改动

把"定价对象每次新建"的根因从 `token-meter` 上移到它的源头：`LlmRuntime.imageRequestPricing()` 按
`(provider, model)` **单槽钉住**一次解析结果，并在唯一的拓扑发布出口 `emitAdaptersUpdated()` 丢弃该槽；
于是同一路由 + 同一注册世系恒定返回**同一对象引用**，token-meter 侧即可**删除 `pricingKey()` 值比较变通**、
恢复干净的 `memo.pricing === pricing` 引用比较。

| 轮次 | 做法 | 结果 |
|---|---|---|
| 第 1 轮 | token-meter 内 `pricing` 引用比较 | ❌ F1：生产 adapter 每次新建对象 ⇒ 永不命中 |
| 第 2 轮 | `pricingKey()` = `JSON.stringify` 值比较 | ✅ 能命中；❌ 分辨力塌缩为 1 bit（恒为 `"{}"`）⇒ 标准 2 FAIL |
| **第 3 轮（本轮）** | **根因上移**：runtime 钉住对象身份 + 拓扑事件失效；token-meter 恢复引用比较 | 标准 1 ✅（真命中）+ 标准 2 ✅（身份 = 路由 + 世系，全分辨力） |

## R3-2 diff 摘要（恰好 3 个文件）

| 文件 | 改动 | 量级 |
|---|---|---|
| `packages/llm/llm/src/index.ts` | ① 新增 `interface ImagePricingMemo`（`:338-356`，纯类型）；② 新增 `private pricingMemo: ImagePricingMemo \| undefined`（`:369-370`）；③ `emitAdaptersUpdated()` 首行 `this.pricingMemo = undefined`（`:377-379`）；④ `imageRequestPricing()`（`:716-722`）改为"读槽 → 委派 → 写槽"；⑤ JSDoc（`:698-711`）补钉住语义 + G-B 显式前提 | +45 / -1 |
| `packages/llm/token-meter/src/index.ts` | ① 删除 `pricingKey()` 整个 helper（第 2 轮新增的 17 行，含 JSDoc）；② `MeasurementMemo.pricingKey: string \| undefined` → `pricing: LlmImageRequestPricing \| undefined`（`:89-90`）；③ 命中判定 `:198` 改 `memo.pricing === pricing`；④ `set` 项 `:244` 改 `pricing`；⑤ 两处 JSDoc 改"按引用比较 + 每次 `llm/adapters-updated` 发布丢弃" | 本轮净 −12（对 HEAD 累计 +59 / −2） |
| `packages/llm/llm/tests/topology.spec.ts` | **C2 允许**：既有 `imageRequestPricing resolution` 用例末尾补 **1 条**断言（`:352`），固定"拓扑发布 ⇒ 钉住失效"语义 | +16 |

### `git diff --stat` 原文（范围核验）

```
$ git diff --stat
 packages/compaction/compaction-basic/src/region.ts | 105 +++++++++++++++------
 packages/llm/llm/src/index.ts                      |  46 ++++++++-
 packages/llm/llm/tests/topology.spec.ts            |  16 ++++
 packages/llm/token-meter/src/index.ts              |  60 +++++++++++-
 4 files changed, 195 insertions(+), 32 deletions(-)

$ git diff --stat -- packages/llm/
 packages/llm/llm/src/index.ts           | 46 ++++++++++++++++++++++++-
 packages/llm/llm/tests/topology.spec.ts | 16 +++++++++
 packages/llm/token-meter/src/index.ts   | 60 ++++++++++++++++++++++++++++++++-
 3 files changed, 120 insertions(+), 2 deletions(-)
```

范围核验（全部通过）：

| 检查 | 结果 |
|---|---|
| `packages/llm/` 内改动文件数 | **3**（2 个 src + C2 允许的 1 个既有 spec）✅ 符合上限 |
| `region.ts`（T1 遗留）`--stat` | **仍为 105**（+75/-30），逐字节未动 ✅ |
| `git diff -U0 -- packages/llm/ \| grep -c "^+import"` | **0**（零新增 import）✅ |
| `git diff -U0 -- .../region.ts \| grep "^+" \| grep -i "measure\|memo\|token-meter\|pricing"` | **空**（T1 与 T2 无耦合）✅ |
| 新增 `v8 ignore` / `c8 ignore` / `istanbul ignore` | **0 处** ✅ |
| 被删行（`^-`）中是否含 `export` / 签名行 | 否（唯一被删行为 `return deepFreeze(structuredClone({` 与本轮新增的 `pricingKey` helper）✅ |

## R3-3 Before / After

### R3-3.1 代码结构

```ts
// BEFORE — packages/llm/llm/src/index.ts:676-678
  imageRequestPricing(provider: string, model: string): LlmImageRequestPricing | undefined {
    return this.adapters.get(provider)?.adapter.imageRequestPricing(provider, model)
  }

// BEFORE — packages/llm/token-meter/src/index.ts（第 2 轮）
function pricingKey(pricing: LlmImageRequestPricing | undefined): string | undefined {
  return pricing === undefined ? undefined : JSON.stringify(pricing)
}
// …
      && memo.pricingKey === pricingKey(pricing)
```

```ts
// AFTER — packages/llm/llm/src/index.ts:716-722（签名 / 返回类型 / 导出面一字未变）
  imageRequestPricing(provider: string, model: string): LlmImageRequestPricing | undefined {
    const memo = this.pricingMemo
    if (memo !== undefined && memo.provider === provider && memo.model === model) return memo.pricing
    const pricing = this.adapters.get(provider)?.adapter.imageRequestPricing(provider, model)
    this.pricingMemo = { provider, model, pricing }
    return pricing
  }

// AFTER — packages/llm/llm/src/index.ts:375-379（C1 方案 A：发布点失效，零新增分支 / 函数）
  private emitAdaptersUpdated(): void {
    // Route pricing is pinned per route: a new topology can only mean new
    // pricing, and this is the one exit every registry write publishes through.
    this.pricingMemo = undefined
    // …既有派发逻辑一字不动…
  }

// AFTER — packages/llm/token-meter/src/index.ts:195-201 / :241-247
    if (memo !== undefined
      && memo.revision === state.consumedEvents
      && memo.llm === llm
      && memo.pricing === pricing
      && optionalHeaderEquals(memo.header, header)) {
      return memo.result
    }
// …
    this.measurements.set(session, { revision: state.consumedEvents, header, pricing, llm, result })
```

### R3-3.2 行为对照

| 维度 | 第 2 轮（before） | 第 3 轮（after） |
|---|---|---|
| `imageRequestPricing` 每次调用 | 必委派 adapter ⇒ **新对象** | 同 `(provider, model)` 同世系 ⇒ **同一引用**；换路由 / 拓扑事件 ⇒ 重新委派 |
| token-meter 键的定价项 | `pricingKey` 字符串，值域 `{undefined, "{}"}` ⇒ **1 bit** | `pricing` 对象引用，值域 = `(provider, model, 世系)` ⇒ **全分辨力** |
| 生产压缩路径命中 | 命中，但同 model 换定价也命中（错复用） | 命中；换 adapter / 换 model / 换世系必 miss |
| 换 provider / model | miss（`headerEquals` → `callConfigEquals` 兜住） | miss（**双重**：路由槽 + `headerEquals`） |
| 换 `LlmRuntime` 实例 | miss | miss（槽是实例字段） |
| 抛错语义 | 不变 | 不变（`_sync()` 仍是 `measure()` 首条语句；`set` 仍在 `deepFreeze` 成功之后） |
| 导出 / 签名 | — | **零改动**（`imageRequestPricing(provider, model): LlmImageRequestPricing \| undefined`、`measure(session, requestHeader?)` 均不变） |

## R3-4 主Agent 三项裁定的执行情况

### C1（批准：失效挂在发布点）—— ✅ 已按方案 A 执行

- `this.pricingMemo = undefined` 写在 `emitAdaptersUpdated()`（`packages/llm/llm/src/index.ts:375-379`）**函数体第一行**；
- **未新增任何订阅函数**（无 `ctx.on`）；
- 完备性静态依据：`this.adapters` 的**全部写入点仅 3 处**——`:408`（disposer 的 `delete`）、`:464`/`:467`（`commitRoutes` 的 `delete` + `set`），
  且 `:408→:410`、`:467→:470` **每处之后紧跟 `emitAdaptersUpdated()`**；`emitAdaptersUpdated()` 是
  `'llm/adapters-updated'` 的**唯一派发出口**（1 个定义 + 4 处调用：`:410`/`:470`/`:519`/`:531`）⇒ **等价完备**；
- 新增覆盖位置：**0 个新分支、0 个新函数**（仅 1 条赋值 statement，随该函数执行即被覆盖）。

### C2（批准：允许改 1 个既有 spec 补 1 条断言）—— ✅ 已执行

- 文件：`packages/llm/llm/tests/topology.spec.ts`；
- 位置：**`:340-352`**，落在既有 `describe('imageRequestPricing resolution')` 的第一个（也是唯一一个）`it` 末尾；
- 断言：`expect(ctx.llm.imageRequestPricing('a', 'vision')).toBe(replacement)`（**`:352`**）——
  先注册一个"服务当前对象"的 adapter 并调用一次把 `pricing` 钉进槽，随后 `handle.replace(['a'])` 发布拓扑事件、
  把 adapter 的服务对象换成 `replacement`，再解析同一路由：**若无发布点失效，此处会命中槽并返回旧的 `pricing` ⇒ 断言红**；
- 未新建 spec 文件；未加 `v8 ignore`；新增断言数为 **1**。

### C3（折为准：G-B / G-C 以显式前提 + JSDoc 结案）—— ✅ 已执行（**消费者枚举见 R3-5，未发现不安全消费者**）

- 前提记录在 `LlmRuntime.imageRequestPricing` 的 JSDoc（`packages/llm/llm/src/index.ts:704-711`），
  并在 `MeasurementMemo`（token-meter `:70-83`）与 `measure()`（`:171-179`）的 JSDoc 中各记一条；
- 措辞要点：钉住对象捕获解析当时的**连接快照**（`models` catalog 及 `inputModalities` / `imagePixelBudget` / `imageMaxBytes`），
  这些字段在**一次注册世系内视为稳定**（`llm-deepseek/src/host.ts:41-48` 仅在 `retryPolicy` 变化时 `registration.replace()`
  ⇒ 纯 catalog/尺寸类热更新不触发拓扑事件、不失效）；**访问路径解析不在此前提内**（`imageAccess` 每次 `priceImages`
  现读 `ctx.get('attachments')` / `ctx.get('fs')`，见 `llm-deepseek/src/adapter.ts:22-25`）；
- 未采用 R2+ 包装器变体（`plan-r3.md` §3.2），按 C1' 默认走已批准形态。

## R3-5 消费者枚举结果（C3 要求，全仓 grep `imageRequestPricing` / `priceImages`）

### R3-5.1 经 `LlmRuntime.imageRequestPricing` 的消费点（受本轮钉住影响）

| # | 消费点 | 用法 | 是否依赖实时连接数据 | 判定 |
|---|---|---|---|---|
| 1 | `packages/llm/token-meter/src/index.ts:269`（`_routeImagePricing`）→ `priceSurface`（`route-pricing.ts:47`） | 取到后**立即** `priceImages`；命中 memo 时**不调用** `priceImages` | 部分：`visualTokens` 依赖钉住时的 `catalogModel` 快照；`text` 依赖 `imageAccess` 的**实时**解析（未命中时才走） | **安全**（本方案的既定目标消费者；窗口 = G-B，已由显式前提结案） |
| 2 | `packages/spill/spill-policy/src/index.ts:65` → `:67` `calculator.priceImages(images)` | 取到后**同一同步块内立即** `priceImages`，**不缓存对象**、不跨 await 持有 | 同上：`visualTokens` 依赖钉住快照（G-B 窗口）；`text` 依赖实时 `imageAccess`/`fs`（保留） | **安全**（非 memo 消费者，取用即时；唯一新增影响面 = G-B 窗口内的快照冻结，已在 JSDoc 显式列出） |
| 3 | `packages/llm/llm/tests/topology.spec.ts:330-338` + 本轮新增 `:352` | 断言 | — | 已逐条推演，钉住后全部仍成立（见 R3-6.3） |
| 4 | `packages/llm/llm-deepseek/tests/adapter.spec.ts:147`、`:285-299` | 断言 | — | 已逐行走查，钉住后 `:290` / `:299` 仍成立（见 R3-6.2） |
| 5 | `packages/test-support/llm-replay/tests/llm-replay.spec.ts:2182`、`:2198` | 断言 | — | `:2182` 钉住 `('deepseek','vision')`；`:2198` model 变 `'plain'` ⇒ miss ⇒ `undefined` ⇒ 断言仍成立 |

### R3-5.2 直连 adapter（不经过 `LlmRuntime`，**不受本轮影响**）

`packages/llm/llm-deepseek/tests/runtime.spec.ts:184/187/203`、`packages/llm/llm-deepseek/tests/files.spec.ts:145`
（`h.adapter.imageRequestPricing`）、`packages/spill/spill-policy/tests/multimodal.spec.ts:140/172`
（`vi.spyOn(ctx.llm, 'imageRequestPricing')` 完全绕过真实实现）、
`packages/spill/spill-policy/tests/multimodal-recovery.spec.ts:67`（adapter override）。

### R3-5.3 枚举结论

> **未发现依赖实时连接数据的不安全消费者。** 全仓 `src` 下读取 `priceImages` 的只有两处：
> `token-meter/src/route-pricing.ts:47`（经 `priceSurface`）与 `spill-policy/src/index.ts:67`。
> 二者都在**取到对象后同一同步块内立即使用**，不跨 await 持有、不跨 revision 复用；
> 对实时连接数据的依赖**仅**落在 G-B 窗口（catalog / `imagePixelBudget` / `imageMaxBytes` 在未改 `retryPolicy` 的情况下热更新），
> 这正是 C3 已折为准、并由 JSDoc 显式记录的前提。
> ⇒ **未触发"停止改码并回报"条件**，本轮按 R2 批准形态完成。

## R3-6 三条硬性走查 + 高风险用例复核

### R3-6.1 键覆盖性（逐结果字段回溯）

`measure()` 返回 6 个字段：`logRevision` / `baseline` / `surfaceDeltaTokens` / `totalTokens` / `surfaceTokens` / `nodes`，
全部由 `state`（`revision` 唯一决定）+ `header`（按值）+ `pricing` + `fileText`（来源 `llm`）决定。
本轮相对第 2 轮**只改了 `pricing` 一项的比较方式**：

- 新方案下 `pricing` 引用与 `(provider, model, 注册世系)` **双射**：同键 ⇒ 必然同一个钉住的对象 ⇒ 行为必然相同；
  异键（换 provider / 换 model / 换世系）⇒ 必然不同对象 ⇒ 必然 miss；
- 反向兜底不变：`(provider, model)` 相同但 `header` 其它字段不同 ⇒ `optionalHeaderEquals` 拦；
  换 `LlmRuntime` 实例 ⇒ `memo.llm === llm` 拦；任何 append ⇒ `revision` 拦。

### R3-6.2 高风险既有用例复核（R-3）：`llm-deepseek/tests/adapter.spec.ts:285-299`

- `price()` = `ctx.llm.imageRequestPricing('deepseek-official', 'deepseek-flash')`，三次调用键完全相同；
- `:287` 首次 ⇒ miss ⇒ 委派 ⇒ 钉住（此时 `catalogModel` 由 `connection.models.find` 定，分支只取决于 catalog）；
- `:288` 挂 `LocalAttachments`、`:295` 挂 `MappedFiles`（`fs`）、`:297` 发一次真实流 —— **均不写 `this.adapters`**，
  故不发布 `llm/adapters-updated`，`:290` / `:299` 走**命中**路径拿到同一对象；
- 断言仍成立的原因：命中对象内的 `resolveAccess`（`adapter.ts:22-25`）在每次 `priceImages` 时**现读**
  `ctx.get('attachments')` / `ctx.get('fs')` ⇒ `:290`（`fs` 未挂 ⇒ `not.toContain('/mounted/image.png')`）与
  `:299`（`fs` = `MappedFiles` ⇒ `toContain('/mounted/image.png')`）**均成立**；
  分支选择只取决于 catalog，而 catalog 在该序列中未变 ⇒ 钉住与不钉住**分支一致**。

### R3-6.3 既有 `topology.spec.ts:330-338` 逐行推演（单槽下仍成立）

| 行 | 槽状态 | 结果 |
|---|---|---|
| `:330 ('a','vision')` | 空 → 写 `{a, vision, pricing}` | `toBe(pricing)` ✅ |
| `:331 ('a','other')` | provider 同、model 异 ⇒ miss → 写 `{a, other, undefined}` | `toBeUndefined()` ✅ |
| `:333 ('plain','vision')` | provider 异 ⇒ miss → `NoopAdapter` 无定价 | `toBeUndefined()` ✅ |
| `:336 ('missing','vision')` | miss → `adapters.get` 空 | `toBeUndefined()` ✅ |
| `:337 dispose()` | **发布 ⇒ 槽清空** | — |
| `:338 ('a','vision')` | 空 ⇒ miss → adapter 已注销 | `toBeUndefined()` ✅ |

## R3-7 CI per-file 100% 覆盖门禁自检（新增运行时位置，逐条清点）

**文件① `packages/llm/llm/src/index.ts`**

| 新增元素 | 类型 | 覆盖来源（已定位的既有测试路径） |
|---|---|---|
| `interface ImagePricingMemo` | 纯类型 | 无运行时位置 ✅ |
| `private pricingMemo` 字段声明 | statement | 任何 `new LlmRuntime()` / `ctx.plugin(LlmRuntime)` ⇒ 数百处 ✅ |
| `this.pricingMemo = undefined`（`emitAdaptersUpdated` 首行） | statement | 该函数被 `topology.spec.ts:327/328/337/348/351`、`service.spec.ts` 数百处 register/dispose/replace 触发 ✅ |
| `const memo = this.pricingMemo` | statement | 每次 `imageRequestPricing` 调用 ✅ |
| `if (...)` **true** 路 | branch | `route-pricing.spec.ts:160→165`、`:220→224`、`:260→262`；`llm-deepseek/tests/adapter.spec.ts:287→290→299` ✅ |
| `if (...)` **false** 路 | branch | `topology.spec.ts:330`（槽空）/`:331`（model 变）/`:333`（provider 变）/`:336`/`:338` ✅ |
| `memo !== undefined` 两侧 | 操作数 | 空槽 `topology.spec.ts:330` / 非空槽 `:331` ✅ |
| `memo.provider === provider` 两侧 | 操作数 | 真：`route-pricing.spec.ts:160→165`；假：`topology.spec.ts:333` ✅ |
| `memo.model === model` 两侧 | 操作数 | 真：`route-pricing.spec.ts:160→165`；**假：`topology.spec.ts:331`（单点依赖）** ✅ |
| `this.pricingMemo = { … }` | statement | 所有 miss 路径 ✅ |

**文件② `packages/llm/token-meter/src/index.ts`**

| 变化 | 类型 | 覆盖来源 |
|---|---|---|
| 删除 `pricingKey()` | −1 function / −2 branch | 位置消失，不再需要覆盖（**连带修掉第 2 轮 F5 那处脆弱依据**）✅ |
| `memo.pricing === pricing`（`:198`）**true** | branch | **`route-pricing.spec.ts:196→199`**：`:196` `meter.measure(session)` 写槽（Harness `() => undefined` ⇒ `pricing = undefined`），`:199` `meter.measure(session, canonicalHeader({ provider: 'unregistered', … }))` ⇒ `LlmRuntime.imageRequestPricing` 返回 `undefined`（provider 未注册）⇒ 两侧同为 `undefined` ⇒ true；两测之间无 append、无拓扑事件 ⇒ revision/llm 相同 ⇒ 该操作数**必被求值**（单点依赖，见下方第 3 项）✅ |
| `memo.pricing === pricing` **false** | branch | **`route-pricing.spec.ts:181→188`**：`:181` `meter.measure(session, header('text-only'))` 是本 session 首次 measure（memo 尚无 ⇒ 该处短路），`:188` `meter.measure(session)` 时 memo 已存在，两次之间无 append、无拓扑事件 ⇒ revision/llm 相同 ⇒ 该操作数**必被求值**；两次路由分别为 `'text-only'`（`substitution`）与 `'vision'`（`fixedPricing`）⇒ 对象引用不同 ⇒ false（单点依赖）✅ |
| `pricing` 操作数被求值 | 需 `memo !== undefined` ∧ revision 同 ∧ llm 同 | 同上两处的**第二次** `measure()` ✅ |

⇒ **未使用（也不需要）任何 `v8 ignore`**；不需要新增测试文件。

⚠️ **门禁变红时的排查依赖（务必保留）**：
1. `memo.model === model` 的 **false** 路只由 `packages/llm/llm/tests/topology.spec.ts:331` 提供；
2. `memo.pricing === pricing` 的 **false** 路只由 `packages/llm/token-meter/tests/route-pricing.spec.ts:181→188` 提供
   （`:181` 与 `:188` 之间不得插入 append、不得改为同 model，否则该操作数不被求值）；
3. `memo.pricing === pricing` 的 **true** 路实际只由 `packages/llm/token-meter/tests/route-pricing.spec.ts:196→199` 提供
   （`:196` 与 `:199` 之间同样不得插入 append、不得改为同一条已注册路由）。
   > 第 3 轮曾把 `token-meter.spec.ts:634` 一并列为 true 侧依据，**不成立**：该用例 `:626` 的 `measure` 与 `:634` 的 `measure` 之间有一次 `session.append`（revision 2→3），
   > `:196 memo.revision === state.consumedEvents` 为假 ⇒ `&&` 短路 ⇒ `:198` 的 `pricing` 操作数根本不被求值。已于第 4 轮从依据清单中移除（不改用例本身）。

## R3-8 收益

| 项 | 说明 |
|---|---|
| 生产是否真能命中 | ✅ `region.ts:386` 解析 ⇒ 槽钉住；`region.ts:456` / `:484` 同 `(provider, model)`、同 `LlmRuntime`、中间无拓扑事件 ⇒ **HIT** ⇒ `memo.pricing === pricing` 成立 ⇒ 跳过 `priceSurface`(O(S)) + `estimateToolsTokens`(tools stringify) + `structuredClone`(O(S)) + `deepFreeze`(O(S)) |
| 调用次数 | 一次 pressure 压缩 5 次 `measure` ⇒ 4 次重算（省 1 次全量测量）；每 retry 再省 1 次 |
| 附带收益 | `imageRequestPricing` 本身少一次 adapter 委派（省一次新对象 + `connection.models.find`） |
| 分辨力 | `pricing` 键从 1 bit 恢复为 `(provider, model, 世系)` 全分辨力 ⇒ **标准 2 由 FAIL 转 PASS** |

## R3-9 残留前提（未闭合项，按 C3 结案）

| 缝隙 | 状态 | 说明 |
|---|---|---|
| **G-A** 同 provider 换 adapter | **✅ 闭合** | `adapters` 的 3 个写入点全部紧跟 `emitAdaptersUpdated()` ⇒ 槽必清 ⇒ 新对象 ⇒ `memo.pricing === pricing` 为 false ⇒ miss；并由本轮新增的 `topology.spec.ts:352` 断言固定 |
| **G-B** 连接快照（catalog / `imagePixelBudget` / `imageMaxBytes`）在同世系内热更新 | **❌ 未闭合（显式前提）** | `host.ts:41-48` 只在 `retryPolicy` 变化时 `replace()` ⇒ 纯 catalog/尺寸类变更不发布事件、不失效。触发需"同 revision + 同世系 + 仅 catalog/尺寸类变更"三者叠加；后果有界（一次价格读数 / 一次 shrink 决策）。**新增影响面**：`spill-policy` 等非 memo 消费者在该窗口内也读钉住快照 —— 已写入 JSDoc |
| **G-C** `imageAccess` 解析变化 | **⚠️ 半闭合** | 钉住对象捕获的是**解析函数**，每次 `priceImages` 现读 `attachments`/`fs` ⇒ 未命中时文本价格实时；**命中时** `priceSurface` 不执行 ⇒ 该窗口内仍复用旧文本价格 |
| **F6** 命中时 `region.ts:457` `isDeepStrictEqual(current.nodes, prepared.measurement.nodes)` 退化为同引用比较 | **⚠️ 随 G-B/G-C 残留** | 命中 ⇔ revision 未变 ∧ 路由未变 ∧ 世系未变；此时断言空过的**唯一**情形就是 G-B/G-C |

## R3-10 未做运行时验证（必须补跑的套件）

本环境无 `node_modules`、无网络，无法 `pnpm test` / `tsc` / 覆盖率脚本。应跑：

- `packages/llm/llm/tests/*`（尤其 `topology.spec.ts`）
- `packages/llm/token-meter/tests/*`（`token-meter.spec.ts`、`route-pricing.spec.ts`、`token-usage-projection.spec.ts`、`context-breakdown-projection.spec.ts`、`turn-usage.spec.ts`）
- `packages/llm/llm-deepseek/tests/*`（尤其 `adapter.spec.ts:278-300`、`dynamic-config.spec.ts`）
- `packages/spill/spill-policy/tests/*`（`multimodal.spec.ts`、`multimodal-recovery.spec.ts`）
- `packages/test-support/llm-replay/tests/*`
- `packages/compaction/compaction-basic/tests/*`、`packages/compaction/command-compact/tests/*`
- 覆盖率门禁：`vitest.config.ts`（`coverage.include = packages/*/*/src/**`、`perFile: true`、statements/branches/functions/lines = 100）+ `scripts/coverage-uncovered-locations.cjs`

---
---

# 第 4 轮（文档精确性修复）— F7 / F8 / F11

> 来自第 3 轮终验（verification.md §8.6 / §8.7）遗留的 3 项非阻断文档差距。
> **本轮纯改注释（JSDoc）与本文档，未改任何源码逻辑**：未增删语句、未改 `if` 与其分支、
> 未新增依赖、未碰 `packages/llm/llm/src/index.ts`、`topology.spec.ts`、`region.ts`，未 commit。

## R4-1 F7 — 补齐 token-meter 侧两处 JSDoc 的残留前提

主落点 `llm/src/index.ts:704-711`（`**Precondition:**` 段）第 3 轮已写入且准确，
但 token-meter 的读者只看得到本包 ⇒ 在两处 JSDoc 中补写同一批前提，措辞与主落点保持一致
（"connection snapshot it was resolved from" / "the `models` catalog with each entry's
`inputModalities`, `imagePixelBudget`, and `imageMaxBytes`" / "Attachment and filesystem
access resolution is **not** part of this precondition"）。

| 落点 | 改动 |
|---|---|
| `packages/llm/token-meter/src/index.ts:83-98`（`MeasurementMemo` JSDoc `:70-99`） | 新增 `**Residual preconditions:**` 段（16 行注释） |
| `packages/llm/token-meter/src/index.ts:196-212`（`measure()` JSDoc `:170-217`） | 新增 "The key deliberately covers nothing below route granularity…" 段（17 行注释） |

改后关键措辞（逐句对应）：

- **G-B（连接快照冻结）**
  - memo 侧：*"A pinned object captures the connection snapshot it was resolved from — the `models`
    catalog with each entry's `inputModalities`, `imagePixelBudget`, and `imageMaxBytes` — and that
    snapshot does not refresh while the pin lives: only republishing the adapter drops it…"*
  - `measure()` 侧：*"…so that snapshot does not refresh over the whole window in which the route
    keeps hitting: only republishing the adapter renews it, and only that republishing misses here."*
    ⇒ 同一槽位命中期间连接快照不会刷新。
- **G-C（`imageAccess` 等访问路径）**
  - memo 侧：*"Attachment and filesystem access resolution is **not** part of this precondition: it is
    read from the live `attachments` and `fs` services on every `priceImages` call, so a miss prices
    image text live, while a hit skips that call and reuses the text captured with `result` — which
    may lag behind an access change made inside the hit window."*
  - `measure()` 侧：*"`priceImages` reads the current `attachments` and `fs` services on every call,
    so a miss prices image text live, while a hit never reaches `priceSurface` and reuses the text
    priced into `result`; an access change inside the hit window is therefore reflected only on the
    next miss."*
    ⇒ 未命中时实时，命中窗口内复用缓存结果。

## R4-2 F8 — 点名插件适用范围（"仅 retryPolicy 变化才重新发布"只对 deepseek 成立）

第 3 轮（及 `llm/src/index.ts:707-709`）那句只在 **deepseek 插件**下为真；
`llm-pi-ai` 在任何 facts 变化时就 `replace`。本轮在上表两处新增注释中同时点名两者
（**F8 修复落在 token-meter 侧；`llm/src/index.ts` 本轮按约束未触碰**）：

- memo 侧 `:89-92`：*"…how eagerly a plugin does so is its own choice — `llm-deepseek` replaces only
  when the route's retry policy changes (`host.ts:41-48`), while `llm-pi-ai` replaces on any change
  to its registration facts (`index.ts:309-318`) — so under `llm-deepseek` a pure catalog or budget
  update leaves this memo valid."*
- `measure()` 侧 `:203-207`：*"Plugins differ in how eagerly they republish: `llm-deepseek` replaces
  on a retry-policy change alone (`llm-deepseek/src/host.ts:41-48`), whereas `llm-pi-ai` replaces
  whenever its registration facts change (`llm-pi-ai/src/index.ts:309-318`) — the "only on retry
  policy" bound therefore describes `llm-deepseek` routes, not every provider."*

⇒ 读者不会再误以为"只有 retryPolicy 变化才失效"是对全部 provider 成立的前提。

## R4-3 F11 — 修正 true 侧覆盖依据

`implementation.md` R3-7（本文档上文）的 true 侧依据已从 **`token-meter.spec.ts:634`**
改为 **`route-pricing.spec.ts:196→199`**（第 3 轮 §8.5 清单第 4 项确认其为实际唯一源）。

- 旧依据为何不成立：`:626` `activeMeter.measure(session)` 与 `:634` 之间有一次 `session.append`
  （revision 2→3）⇒ `:196 memo.revision === state.consumedEvents` 为假 ⇒ `&&` 短路
  ⇒ `:198` 的 `pricing` 操作数根本不被求值（该用例用在本文件的依据清单中属于误列，**用例本身无需改动**）。
- 新依据的求值链（本轮核对）：`:192` harness `() => undefined` ⇒ `:196 measure` ⇒ `pricing = undefined`
  并写 memo；`:199` 换 `provider: 'unregistered'` ⇒ `LlmRuntime.imageRequestPricing` 返回 `undefined`
  ⇒ 两侧同为 `undefined` ⇒ **true**；两次 `measure()` 之间无 append、无拓扑事件 ⇒ 该操作数必被求值。
- false 侧依据复核（`:181→188`）：**表述已同步收紧并确认为准确** —— `:181` 为该 session 首次
  `measure`（memo 尚无 ⇒ 短路），真正求值点是 `:188`；两次之间无 append、llm 相同 ⇒ 必被求值，
  且 `substitution` ≠ `fixedPricing`（不同对象引用）⇒ **false**。原结论不变，仅把"哪一次调用才求值"写清楚。
- 同步在 R3-7 的"门禁变红时的排查依赖"清单补上第 3 项（true 侧单点）。

## R4-4 本轮自核（改后复查结果）

| 检查 | 结果 |
|---|---|
| 本轮是否改动可执行语句 | **否**。token-meter 相对 HEAD 的**代码**（剥离全部注释后的 token 流）差异仅为第 3 轮已知的 6 处新增（`MeasurementMemo` 接口、`measurements` 字段、`const llm`、`memo` 查询块、`return`→`const result =`、`measurements.set`），本轮新增行 100% 位于 `/** … */` 之内 |
| 新增 `import` | `git diff -U0 -- packages/llm/ \| grep -c "^+import"` = **0** ✅ |
| 新增 `v8 ignore` / `c8 ignore` / `istanbul ignore` | **0** ✅ |
| 触碰 `llm/src/index.ts`、`topology.spec.ts`、`region.ts` | **未触碰**（`--stat` 三者与第 3 轮快照一致：46 / 16 / 105）✅ |
| `git diff --stat`（第 4 轮后） | 见下方原文 |
| 是否 commit | **未 commit** ✅ |
| 行宽 | 新增注释最长 80 字符（文件既有最长 137）✅ |

```
 packages/compaction/compaction-basic/src/region.ts | 105 +++++++++++
 packages/llm/llm/src/index.ts                      |  46 ++++
 packages/llm/llm/tests/topology.spec.ts            |  16 +
 packages/llm/token-meter/src/index.ts              |  93 ++++++++++
 4 files changed, 228 insertions(+), 32 deletions(-)
```

> 上方为便于排版压缩后的示意；原文（`git diff --stat`，条形按终端宽度缩放）：
> `region.ts` **105**（+75/-30）、`llm/src/index.ts` **46**（+45/-1）、`topology.spec.ts` **16**（+16/-0）、
> `token-meter/src/index.ts` **93**（+92/-1）⇒ 合计 **228 insertions(+), 32 deletions(-)**。
> token-meter 由第 3 轮的 60（+59/-1）变为 93（+92/-1），增量 **33 行全部为注释行**。

**仍未做的运行时验证**：与 R3-10 相同（本环境无 `node_modules`、无网络，无法 `pnpm test` / `tsc` /
覆盖率脚本）。本轮为纯文档改动，不引入新的运行时位置，故不新增需补跑的套件；R3-10 的套件清单仍然适用。

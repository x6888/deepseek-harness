# T1 计划 — `inspectCompactionEntryState()` 每步全量倒扫 → 增量 fold

> 流程 4.2 规划者产出。**只规划，不改代码**（本文件是唯一写入产物之一，另一为 `steps.md`）。
> 环境无法 build/run（依赖未装、无网络）→ 所有结论为静态/结构核对，标记「未做运行时验证」。
> 行号对应本次快照（2026-10-02），目标文件：
> `sandbox/deepseek-harness/packages/compaction/compaction-basic/src/region.ts`

---

## 1. 优化目标

把 `region.ts` 的 `inspectCompactionEntryState()` 从「**每次调用都对整条事件日志做一次反向全扫描**」
改为「**模块级 `WeakMap<Session, Fold>` 增量折叠，每次只消费上次游标之后的新事件**」，
使每步成本从 **O(N)/步**（N = `session.seq` = 日志长度）降为 **O(Δ)/步**（Δ = 自上次读取后新增事件数），
**且返回值与状态判定逐位不变**（纯加速器，语义等价）。

硬约束（来自主 Agent）：
- 改动自包含于 `region.ts` 单文件；不引入新依赖；不修改其他文件。
- 导出签名与调用方不变（`assertNoActiveCompaction(session, stage)` 是唯一对外入口）。

---

## 2. 当前实现逐行分析

### 2.1 目标函数原文要点（`region.ts:565-600`）

```ts
565  /** Inspect open-turn, unmatched-compaction, and latest seed-boundary state independently. */
566  function inspectCompactionEntryState(session: Session): CompactionEntryState {
567    let openTurn: number | null = null
568    let openTurnStateKnown = false
569    let unmatchedCompactionStart: SessionEvent<'compaction/start'> | undefined
570    let compactionEntryStateKnown = false
571    let latestEndSeedSeq: SessionSeq | undefined
572    for (let seq = session.seq - 1; seq >= 0; seq -= 1) {        // ← 倒扫：从最新事件扫到 seq 0
574-575   const event = session.eventAt(SessionSeq(seq))!           // ← 每个事件一次 eventAt + 非断言
576      if (latestEndSeedSeq === undefined && event.type === 'session/end-seed') {
577        latestEndSeedSeq = event.seq                             // ← 只在「尚未设置」时写 → 取「最新」的 end-seed
578      }
579      if (!compactionEntryStateKnown) {
580        if (event.type === 'compaction/start') { unmatchedCompactionStart = event; compactionEntryStateKnown = true }
583        else if (event.type === 'compaction/end') { compactionEntryStateKnown = true }   // ← 最新 marker 决定「有无未闭合 start」
586      }
587      if (!openTurnStateKnown) {
588        if (event.type === 'turn/start') { openTurn = event.data.turn; openTurnStateKnown = true }
591        else if (event.type === 'turn/end') { openTurnStateKnown = true }                  // ← 最新 turn marker 决定「有无未闭合 turn」
594      }
595-597   if (openTurnStateKnown && compactionEntryStateKnown && latestEndSeedSeq !== undefined) break
598    }
599    return { openTurn, unmatchedCompactionStart, latestEndSeedSeq }
600  }
```

### 2.2 语义抽取（等价性的关键）

三个输出量**全部是「最后一次出现」的函数（latest-wins）**，且都有默认值：

| 输出量 | 定义 | 触发事件 | 默认值 |
|---|---|---|---|
| `latestEndSeedSeq` | **最新** `session/end-seed` 的 seq | `session/end-seed` | `undefined` |
| `unmatchedCompactionStart` | 最新 marker ∈ {`compaction/start`, `compaction/end`}；若是 `start` 则为该事件，否则 `undefined` | `compaction/start` / `compaction/end` | `undefined` |
| `openTurn` | 最新 marker ∈ {`turn/start`, `turn/end`}；若是 `start` 则为 `data.turn`，否则 `null` | `turn/start` / `turn/end` | `null` |

**引理（break 无副作用）**：倒扫中三个 `xxxKnown` 标志一旦置位就不再被改写（第 576 行有
`latestEndSeedSeq === undefined` 守卫；第 579/587 行有 `!known` 守卫），因此
第 595-597 行的 `break` 只是**提前退出**，不改变任何输出——输出等价于「扫完整条日志」。

### 2.3 为什么它从不提前 break（O(N) 的根因）

- `latestEndSeedSeq` **只能**由 `session/end-seed` 事件设置。
- 该事件只在「seed/构造期」出现：`Session` 构造函数（`session/src/index.ts:618-622`）在
  `seed !== undefined` 时追加；普通 live 会话（绝大多数长会话）**整条日志都没有该事件**。
- 于是第三个 break 条件永远不成立 → 每步都真正扫完整条日志 `[0, session.seq)`。

### 2.4 调用链（每步一次）

```
agent/pre-step                 (compaction-basic/src/index.ts:158)
  → compactIfNeeded(agent, 'pressure', signal)
      → assertNoActiveCompaction(agent.session, 'automatic pressure compaction')   (index.ts:305)
          → inspectCompactionEntryState(session)                                   (region.ts:328)
```
第二个调用点：`compactSurfaceRegion()` 在开启事务前 `region.ts:184`（同一事务内还可能被
`assertSelectedSpanStable` 之外的重试点再次触发）。

**复杂度现状**：一次 session 生命周期内 S 步、日志终长 N ⇒ 总计 **O(S·N)** 次 `eventAt` + 类型比较；
每步的 O(N) 与日志长度线性增长，正是 `benchmarks/long-session-browser/` 所测路径。

### 2.5 日志模型事实（决定「增量是否安全」）

静态核对结论（`packages/core/session/src/index.ts`）：

| 事实 | 证据 |
|---|---|
| `seq` ≡ 日志长度，`eventAt(seq) === log[seq]` | L635-637 `return this.log[seq]`；L680-683 `get seq(){ return SessionLogOffset(this.log.length) }` |
| 日志**只增不减**：仅 `push`，无 `splice`/截断/原地改写 | 全文 `this.log` 写入点只有 L587（构造期 seed 一次性灌入）与 L761（`append`） |
| seq 连续自 0，事件不可变（`deepFreeze`） | 构造期校验（L560-566 注释）；L744-750 `deepFreeze({...})` |
| fork/restore 的继承前缀在**构造时**一次性写入，之后不再插入 | L586-590 `firstLiveSeq = log.length`；L598-622 |
| 仓库已有同类增量 fold 先例 | `TokenMeter._sync()`（`token-meter/src/index.ts:219-240`，`WeakMap<Session, ReplayState>` + `while (consumedEvents < session.seq)`）；`Session` 自身 `headerFoldSeq`/`contextFoldSeq`/`toolHistorySeq`（L778/802/822） |

⇒ **「已折叠 `[0, cursor)` + 只消费 `[cursor, session.seq)`」的前向折叠，与倒扫覆盖同一事件序列，且每个事件恰好被处理一次。**

---

## 3. 优化方案

### 3.1 结构（新增，全部在 `region.ts` 内）

```ts
/** 已折叠 `[0, cursor)` 的增量状态；三个量都只依赖「最后一次出现」。 */
interface CompactionEntryFold {
  /** 下一个待消费的日志偏移；`cursor` 之前的事件已各折叠一次。 */
  cursor: number
  openTurn: number | null
  unmatchedCompactionStart: SessionEvent<'compaction/start'> | undefined
  latestEndSeedSeq: SessionSeq | undefined
}

/**
 * 每会话一份折叠状态。弱键：会话被丢弃时折叠随之释放。
 * 缓存只是加速器——缺失时退化为一次全量折叠，结论不变。
 */
const compactionEntryFolds = new WeakMap<Session, CompactionEntryFold>()
```

放置位置：紧随 `CompactionEntryState`（L66-70）之后；`WeakMap` 常量紧随其后（模块级，与文件现有
「interface → function」的排布一致；不新增任何 import，游标用 `number`，
`SessionSeq(value: number)` 已在本文件 import 中）。

### 3.2 折叠与读取（替换 L565-600）

```ts
/**
 * Inspect open-turn, unmatched-compaction, and latest seed-boundary state independently,
 * folding each logged event at most once per session.
 *
 * 不变式（改动等价性的依据）：
 * 1. 三个结果都是「最后一次出现」的函数（latest-wins），故「自旧到新逐个覆盖」与
 *    「自新到旧首次命中即定」在同一序列上给出同一结果；原实现的 break 只是提前退出。
 * 2. 日志 append-only 且 seq 连续（`session.seq === log.length`），故已折叠前缀永不失效。
 * 3. 缓存缺失（新会话 / WeakMap 被 GC / 模块重载）只影响耗时，不影响结论：退化为从 0 折叠。
 * 若将来要在此处统计「计数」或「最早出现」等非 latest-wins 量，必须改回全量扫描。
 */
function inspectCompactionEntryState(session: Session): CompactionEntryState {
  let fold = compactionEntryFolds.get(session)
  if (fold === undefined) {
    fold = {
      cursor: 0,
      openTurn: null,
      unmatchedCompactionStart: undefined,
      latestEndSeedSeq: undefined,
    }
    compactionEntryFolds.set(session, fold)
  }
  while (fold.cursor < session.seq) {
    // 连续 session seqs 索引持久日志；Existing Session history read; migration deferred.
    // oxlint-disable-next-line typescript/no-non-null-assertion, typescript/no-deprecated
    const event = session.eventAt(SessionSeq(fold.cursor))!
    if (event.type === 'session/end-seed') {
      fold.latestEndSeedSeq = event.seq
    } else if (event.type === 'compaction/start') {
      fold.unmatchedCompactionStart = event
    } else if (event.type === 'compaction/end') {
      fold.unmatchedCompactionStart = undefined
    } else if (event.type === 'turn/start') {
      fold.openTurn = event.data.turn
    } else if (event.type === 'turn/end') {
      fold.openTurn = null
    }
    fold.cursor += 1
  }
  // 每次返回新对象：调用方拿不到 fold 引用，无法污染增量状态。
  return {
    openTurn: fold.openTurn,
    unmatchedCompactionStart: fold.unmatchedCompactionStart,
    latestEndSeedSeq: fold.latestEndSeedSeq,
  }
}
```

要点（都是为「无法编译验证」而做的保守选择）：
- **沿用现有 `if / else if` 链**（不用 `switch`）：当前代码（L576/580/583/588/591）已证明这种写法能正确窄化
  `SessionEvent` 判别联合，`fold.openTurn = event.data.turn` 与 `unmatchedCompactionStart = event` 的类型与现有一致。
- **沿用同样的 `!` + 双 oxlint disable 注释**（`typescript/no-non-null-assertion, typescript/no-deprecated`），
  与 L574-575 逐字一致，不会触发新的 lint 项。
- **不新增 import**：游标为 `number`（`session.seq` 为 `SessionLogOffset` branded number，与 `number` 可比较）。
  若要与 `TokenMeter` 完全同构可另 import `SessionLogOffset`（该符号确由 `@deepseek-ai/dsh-session` 导出，
  见 `token-meter/src/index.ts:23`），但会改动 import 行——推荐不引入，diff 更小。
- **不改任何导出**：`inspectCompactionEntryState` 仍是模块私有，`assertNoActiveCompaction`（L327）签名不变。

### 3.3 等价性论证（逐事件类型对照）

设事件序列 `E[0..N-1]`。原实现对每个 `i` 从 `N-1` 递减处理；新实现按 `0..N-1` 递增处理，
且两者对同一事件的**状态转移**如下：

| 事件类型 | 原倒扫（自新→旧，命中即定） | 新前扫（自旧→新，逐个覆盖） | 一致 |
|---|---|---|---|
| `session/end-seed` | 首次（=最新）命中写入 `event.seq`，随后因 `=== undefined` 守卫不再改 | 每次覆盖 → 终值 = 最新的 end-seed seq | ✅ |
| `compaction/start` | 若 `!known`：记录该 start 并置 known | 覆盖写入该 start | ✅ |
| `compaction/end` | 若 `!known`：置 known，`unmatched` 保持 `undefined` | 清空 `unmatched` | ✅ |
| `turn/start` | 若 `!known`：`openTurn = data.turn` 并置 known | 覆盖 `openTurn = data.turn` | ✅ |
| `turn/end` | 若 `!known`：置 known，`openTurn` 保持 `null` | `openTurn = null` | ✅ |
| 其它类型 | 无状态副作用（只参与 break 判定） | 无副作用 | ✅ |

**覆盖域一致**：原倒扫遍历 `[0, N)` 全部（break 只是提前退出，见 2.2 引理）；
新实现冷启动从 `cursor=0` 折叠 `[0, N)`，后续每次折叠 `[cursor, N')`，而 `[0, cursor)` 已在之前折叠过
且日志前缀不可变 ⇒ 全序列每个事件恰好处理一次。

**边界场景走查**（三方一致）：
1. 空日志（`seq === 0`）：循环不执行 → `{null, undefined, undefined}`，与原来（循环不执行）相同。
2. 有 `session/end-seed` 的 seed 会话：原倒扫首个 end-seed 即 break 候选；新前扫终值 = 最新 end-seed。相同。
3. 未闭合 `compaction/start`（start 后无 end）：原 → 该 start；新 → 该 start（`compaction/end` 未出现不清空）。相同。
4. 已闭合（start…end）：原 → `undefined`；新 → 被 end 清空。相同。
5. `turn/start` 后紧跟新事件、未见 `turn/end`：原 → `openTurn = turn`；新 → 同。相同。
6. 事务内 `compactSurfaceRegion` 自己 append 的 `compaction/start`（L210）→ 下一次调用（如 `agent/pre-step`
   的 `assertNoActiveCompaction`）折叠到该事件 → **锁仍然被检出**（`assertCompactionInactive` L308-320 抛
   `ManualCompactionError('busy')`）。与原实现一致，这是本改动最不能出错的一条路径。

### 3.4 失效 / 回退路径

| 场景 | 行为 | 代价 | 正确性 |
|---|---|---|---|
| 无缓存（新会话 / 首次调用 / WeakMap 条目被 GC / HMR 模块重载 → 新 WeakMap） | 从 `cursor = 0` 全量折叠一次 | 一次 O(N) | 与现状单次调用同价，结果相同 |
| 日志只增（唯一真实情况） | 折叠 `[cursor, session.seq)` | O(Δ) | ✅ |
| 假设性「日志变短」（当前代码库不可能：`log` 只有 `push`） | 见 §5-R2 的裁决项 | — | 默认**不加**守卫，与 `TokenMeter._sync()` 同构 |
| 未知事件类型 / 未来新增事件类型 | 忽略（无副作用） | — | 与原来一致（原只响应 5 种类型） |
| `eventAt` 返回 `undefined`（契约违反） | `!` 后在 `event.type` 抛 TypeError，与现状同类失败；游标不推进，下次同点重试 | — | 与现状一致 |
| 完全放弃本次优化 | 删除新增的 interface + WeakMap，还原 L565-600 原函数 | — | 单文件 revert，无外部影响 |

### 3.5 已评估但**未采纳**的备选

| 方案 | 不采纳理由 |
|---|---|
| 冷启动保留原倒扫（带 break）、热路径用前扫 | 两份实现表达同一语义 → 等价性需双份论证，且易随时间漂移；冷启动代价本就等于现状一次调用，不值得 |
| 用 `ctx.sessionProjections` 注册投影单元 | 需要投影 schema/版本号/注册时机，跨文件且风险高，违反「单文件、不改其他文件」 |
| 在 `Session` 上加 fold（仿 `headerFoldSeq`） | 改动 `packages/core/session`，越界 |
| 用 `snapshotEvents()` 批量读新事件再循环 | 每次分配一个数组；`eventAt` 单取更省，且与 `TokenMeter._sync()` 同形 |

---

## 4. 预期收益

| 维度 | before | after |
|---|---|---|
| 单次 `inspectCompactionEntryState` | O(N)（`eventAt` + 类型比较 × N，N = 日志长度） | O(Δ)，Δ = 自上次读取后新增事件数（通常个位数） |
| 一个会话 S 步的累计 | O(S·N) | O(N + S·Δ) ≈ O(N)（每个事件一生只折叠一次） |
| 长会话（如 N≈10⁴ 事件、S≈10²–10³ 步）每步成本 | 万级事件读取/步 | 单次读取 ~0（无 end-seed 会话下，Δ 仅为该步新事件） |
| 分配 / GC | 每次调用无分配（现状也是） | 不变；额外内存 = 每会话 1 个小对象（弱键，随会话释放） |
| 语义 | — | **零变化**（返回值逐字段等价，见 §3.3） |
| 命中路径 | `agent/pre-step` 每步 1 次 + 每个压缩事务 1 次 | 同上，只是更快 |

对齐 `benchmarks/long-session-browser/`（长会话合成历史）这条主测路径。
本环境**未做运行时验证**，收益为静态复杂度推断。

---

## 5. 风险与回退

| # | 风险 | 等级 | 缓解 / 裁决 |
|---|---|---|---|
| **R1** | **等价性依赖「三个量都是 latest-wins」**：若将来有人在此循环里加入计数/最早出现等非 latest-wins 量，增量折叠会静默出错 | 中 | 把该不变式**写进函数 JSDoc**（§3.2 已含），并写明「非 latest-wins 量必须改回全量扫描」；本次三个量已逐类型证明 |
| **R2** | **WeakMap key 选型**：key = `Session` 对象，隐含「Session 身份 ⇒ 日志身份」。若将来出现「同一 Session 换底层 log / 截断日志」，缓存会给出陈旧结论——而这里是**压缩锁**，陈旧「无活动压缩」会让第二个压缩事务并发进入 | 中 | 事实：日志只有 `push`（§2.5），seed 在构造期一次性写入 ⇒ 当前不可能。可选防御：在 `while` 前加 `if (fold.cursor > session.seq) { 重置为冷启动 }`（4 行）。**但该分支在测试中不可达**，而 CI 对 `packages/*/*/src` 有 per-file 100% 覆盖门禁 → 建议**默认不加**，与 `TokenMeter._sync()`（同样无守卫）保持同构；若主 Agent 要求防御，需同步放宽「只改单文件」以补测试 |
| **R3** | 无法 build/run → 类型窄化、`SessionSeq(number)`、oxlint 规则未经机器验证 | 中 | 保守写法：沿用已存在的 `if/else if` 窄化链、沿用同款 `!` + oxlint disable、不新增 import、不动导出签名；实现后按 `steps.md` 的静态清单逐条核对 |
| **R4** | 冷启动一次 O(N) | 低 | 等于现状单次调用代价；每个 Session 一生最多几次（GC/HMR 后重来） |
| **R5** | fold 持有 `SessionEvent` 强引用 | 低 | 该事件本就在 session 日志里（日志生命周期 ≥ fold），无额外泄漏；WeakMap 保证 fold 随 Session 释放 |
| **R6** | 并发/重入：fold 内有 `await`？ | 无 | `inspectCompactionEntryState` 全同步，日志 append 也同步 ⇒ 无交错；异步窗口（摘要期间）只是让 Δ 变大，下次读取照样追上 |
| **R7** | 覆盖率门禁（per-file 100% on `packages/*/*/src`） | 中 | 新增分支仅 `fold === undefined`（**可达**，首次调用必走，被现有 4 个 spec 覆盖）与 `while`（可达）。不新增不可达分支（见 R2） |

**回退**：新增内容仅 1 个 interface + 1 个 WeakMap 常量 + 1 个函数体；`git checkout` 单文件即完全恢复，
无跨文件耦合、无持久化数据结构、无配置项。

---

## 6. 验收标准逐条对照

| # | 验收标准 | 是否满足 | 依据 / 落地方式 |
|---|---|---|---|
| **1** | 用增量 fold（`WeakMap` 缓存游标/快照，仿 `TokenMeter._sync()`）替代每步全量倒扫，O(N)/步 → O(新事件)/步；保留原有语义 | ✅ | §3.1-3.2：`WeakMap<Session, CompactionEntryFold{cursor, ...}>`，`while (fold.cursor < session.seq)` 只消费新事件（同 `token-meter/src/index.ts:232`）；返回值三字段逐一等价（§3.3），`assertCompactionInactive`（L308）判定输入不变 |
| **2** | 正确处理「上次游标之后的新增事件」，增量结果与全量扫描等价 | ✅ | §2.5 日志 append-only + seq 连续 ⇒ 前缀永不失效；§3.3 逐事件类型转移表 + 6 个边界场景走查；§3.4 冷启动（无缓存）路径 ≡ 全量扫描 |
| **3** | 改动仅限 `region.ts` 单文件、无新依赖、TS 风格与现有 import/命名一致 | ✅ | 仅改 `region.ts`；不新增 import（游标 `number`，复用已有 `SessionSeq`）；沿用 `if/else if` 窄化链、`!` + 同款 oxlint disable 注释、文件现有 JSDoc 风格；命名 `CompactionEntryFold` / `compactionEntryFolds` 与现有 `CompactionEntryState` 一致 |
| **4** | 保留现有导出与调用方语义（签名不变、调用方无需改动、行为兼容） | ✅ | `inspectCompactionEntryState` 仍为模块私有、签名 `(session: Session): CompactionEntryState` 不变；`export function assertNoActiveCompaction(session, stage)`（L327）一行不动；调用方 `index.ts:305`、`region.ts:184` 无需改动 |
| **5** | 附 before/after 行为说明与最小静态核对方式（复杂度分析 + 等价性走查） | ✅ | before/after 见 §4 与 §7；静态核对清单见 `steps.md` S5/S6（结构 grep + 导出/import 不变核对 + 逐事件等价走查 + 复杂度分析） |

---

## 7. before / after 行为说明（供交付报告直接引用）

**before**
- `inspectCompactionEntryState()` 每次调用从 `session.seq - 1` 倒扫至 `0`，对每个事件做 `eventAt` + 类型判断。
- 提前 `break` 需三个条件齐备，其中 `latestEndSeedSeq !== undefined` 依赖 `session/end-seed`
  （仅 seed/构造期产生）⇒ **非 seed 会话从不 break ⇒ 每步 O(N)**。
- 每步（`agent/pre-step` → `compactIfNeeded` → `assertNoActiveCompaction`）付一次全日志成本；
  会话累计 O(S·N)。

**after**
- 每会话在模块级 `WeakMap` 中保留一份 `{cursor, openTurn, unmatchedCompactionStart, latestEndSeedSeq}`。
- 每次调用只把 `[cursor, session.seq)` 的新事件折叠进去（各类型覆盖写入），随后返回一份新对象。
- 无缓存时从 `0` 折叠一次（≡ 现状一次全量扫描）；缓存缺失只影响耗时。
- 会话累计 O(N + S·Δ)；单步 O(Δ)。
- **行为不变**：相同日志序列下 `openTurn` / `unmatchedCompactionStart` / `latestEndSeedSeq` 三个值与原来逐位相等，
  因此 `assertCompactionInactive` 的 busy/放行判定、`compactSurfaceRegion` 的 owner 与锁判定全部不变。

**未做运行时验证**：本环境无网络、依赖未装，以上为源码静态分析与结构核对结论；
实现完成后应在可运行环境补跑 `packages/compaction/compaction-basic/tests/`（4 个 spec：
`compaction-basic.spec.ts`、`compaction-loop-repro.spec.ts`、`loader-composition.spec.ts`、
`manual-compaction.spec.ts`）与 `packages/compaction/command-compact/tests/`。

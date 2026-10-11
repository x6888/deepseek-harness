# T1 实现说明 — `region.ts` `inspectCompactionEntryState()` 增量 fold

> 流程 4.3 改码者产出。**唯一被修改的源文件**：
> `packages/compaction/compaction-basic/src/region.ts`（git diff: 1 file changed, 75 insertions(+), 30 deletions(-)）
> 环境无法 build/run（依赖未装、无网络）→ 验证为**静态/结构核对**，未做运行时验证。

---

## 1. 一句话改动

把 `inspectCompactionEntryState()` 的「每次调用倒扫整条事件日志」换成**模块级 `WeakMap<Session, CompactionEntryFold>` 前向增量折叠**，每次只消费 `[cursor, session.seq)` 区间的新事件；单步 O(N) → O(Δ)，返回值三字段逐位等价，导出签名与调用方零改动。

---

## 2. 关键 diff 摘要

### 2.1 新增（在 `CompactionEntryState` 之后，`region.ts:72-90`）

```ts
/**
 * Incremental state over the already-folded log prefix `[0, cursor)`. Every
 * field is a function of the *last* occurrence of a marker, so folding events
 * strictly forwards agrees with the reverse scan's first-hit-wins result.
 */
interface CompactionEntryFold {
  /** Next log offset to consume; every event before it has been folded once. */
  cursor: number
  openTurn: number | null
  unmatchedCompactionStart: SessionEvent<'compaction/start'> | undefined
  latestEndSeedSeq: SessionSeq | undefined
}

/**
 * One fold per session. Weak keys release the fold with its session; the cache
 * is a pure accelerator — a miss degrades to one full fold, never to a
 * different answer.
 */
const compactionEntryFolds = new WeakMap<Session, CompactionEntryFold>()
```

- key 选型仿 `TokenMeter._sync()`（`packages/llm/token-meter/src/index.ts:219-240`，
  `WeakMap<Session, ReplayState>` + `while (consumedEvents < session.seq)`）：**用 Session 对象作弱键**。
- 三个字段的类型与 `CompactionEntryState` 逐字同源；interface 与常量均**不导出**。
- 游标为 `number`（与 plan §3.2 的取舍一致）⇒ **不新增 import**（`SessionSeq` 已在本文件 import 中）。

### 2.2 替换函数体（`region.ts:585-645`）

```ts
function inspectCompactionEntryState(session: Session): CompactionEntryState {
  let fold = compactionEntryFolds.get(session)
  if (fold === undefined) {
    fold = { cursor: 0, openTurn: null, unmatchedCompactionStart: undefined, latestEndSeedSeq: undefined }
    compactionEntryFolds.set(session, fold)
  }
  while (fold.cursor < session.seq) {
    // The fold cursor indexes the persistent log by contiguous session seqs.
    // Existing Session history read; migration deferred.
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
  // Copied out, never handed over: callers cannot mutate the incremental fold.
  return {
    openTurn: fold.openTurn,
    unmatchedCompactionStart: fold.unmatchedCompactionStart,
    latestEndSeedSeq: fold.latestEndSeedSeq,
  }
}
```

删除的旧代码：`for (let seq = session.seq - 1; seq >= 0; seq -= 1)` 倒扫 + 三个 `xxxKnown` 标志 +
三条件 `break`。

保守写法（因为无法编译验证）：
- **沿用 `if / else if` 窄化链**（不用 `switch`），分支顺序与原实现一致：
  `session/end-seed` → `compaction/start` → `compaction/end` → `turn/start` → `turn/end`。
- **沿用同一条 `oxlint-disable-next-line typescript/no-non-null-assertion, typescript/no-deprecated`**，
  连同上方注释「Existing Session history read; migration deferred.」逐字保留。
- **返回新对象**，不把 `fold` 交给调用方（`CompactionEntryState` 是 `readonly`，原实现同样返回字面量）。
- **不新增不可达分支**（见 §4）。

### 2.3 JSDoc（必须记录的等价性依据）

函数上方 JSDoc 显式写明四条：

1. **日志 append-only 且 `seq` 连续**（`eventAt(seq) === log[seq]`、`session.seq === log.length`）
   ⇒ 已折叠前缀永不失效。
2. **没有 `cursor > session.seq` 重置守卫**：按主 Agent 裁定不添加；注释明确「该守卫在当前日志模型下不可达，
   若将来日志语义变化须整体重审本折叠，而不是在这里打补丁」——避免 0 新增不可达分支触发 per-file 100% 覆盖门禁。
3. **三个输出量全部是 latest-wins**：`latestEndSeedSeq` = 最新 `session/end-seed`；
   `unmatchedCompactionStart` = 最新 `compaction/start`（除非被更新的 `compaction/end` 关闭）；
   `openTurn` = 最新 `turn/start`（除非被更新的 `turn/end` 关闭）。原倒扫的 `break`
   只是提前退出，不改写已定字段 ⇒ 前扫覆盖 ≡ 倒扫首次命中。
4. **非 latest-wins 的量（计数 / 最早出现 / 累加和）加入此循环会静默破坏折叠，必须改回全量扫描。**

---

## 3. before / after

| 维度 | before | after |
|---|---|---|
| 单次调用 | 倒扫 `[0, session.seq)`：`eventAt` + 类型比较 × N | 仅折叠 `[cursor, session.seq)`：O(Δ)，Δ = 上次读取后新增事件数 |
| 会话 S 步累计 | O(S·N)（非 seed 会话因缺 `session/end-seed` 从不触发 break，每步真全扫） | O(N + ΣΔ) ≈ O(N)，每个事件一生只折叠一次 |
| 冷启动（无缓存 / WeakMap 条目被 GC / HMR 重载） | — | 从 `cursor = 0` 折叠一次：与现状单次调用同价，结果相同 |
| 分配 | 每次调用 1 个返回对象 | 不变 + 每会话 1 个小 fold 对象（弱键，随 Session 释放） |
| 返回值 | `{openTurn, unmatchedCompactionStart, latestEndSeedSeq}` | **逐字段等价**（见 §4） |
| 内存 | — | fold 持有 `SessionEvent` 引用，但该事件本就在 session 日志中，无额外泄漏 |

命中路径：`agent/pre-step` → `compactIfNeeded`（`index.ts:305` → `assertNoActiveCompaction`）每步 1 次，
外加每个压缩事务 1 次（`region.ts:204`）；与会内长会话基准 `benchmarks/long-session-browser/` 同路径。
**未做运行时验证**，收益为静态复杂度推断。

---

## 4. 等价性说明

**逐事件转移表**（设事件序列 `E[0..N-1]`；原实现自 `N-1` 递减，新实现自 `0` 递增）：

| 事件类型 | 原倒扫（首次命中即定） | 新前扫（逐个覆盖） | 一致 |
|---|---|---|---|
| `session/end-seed` | 首个（=最新）写入 `event.seq`，随后被 `=== undefined` 守卫挡住 | 每次覆盖 ⇒ 终值 = 最新 end-seed seq | ✅ |
| `compaction/start` | 若 `!known`：记录该 start | 覆盖写入该 start | ✅ |
| `compaction/end` | 若 `!known`：置 known，`unmatched` 保持 `undefined` | 清空 `unmatched` | ✅ |
| `turn/start` | 若 `!known`：`openTurn = data.turn` | 覆盖 `openTurn = data.turn` | ✅ |
| `turn/end` | 若 `!known`：置 known，`openTurn` 保持 `null` | `openTurn = null` | ✅ |
| 其它类型 | 无状态副作用 | 无副作用 | ✅ |

**覆盖域**：原倒扫理论上遍历 `[0, N)`（`break` 仅为提前退出）；新实现冷启动折叠 `[0, N)`，
之后每次折叠 `[cursor, N')`，而 `[0, cursor)` 之前已各折叠一次且日志前缀不可变
（`packages/core/session/src/index.ts`：`this.log` 只有构造期灌入与 `append` 两处写入）
⇒ 全序列每个事件**恰好被处理一次**。

**边界场景**（原/新三元组 `(openTurn, unmatchedCompactionStart, latestEndSeedSeq)` 逐位相等）：

| 场景 | 日志 | 三元组 |
|---|---|---|
| a 空日志 | — | `(null, undefined, undefined)` |
| b seed 会话 | `session/end-seed@5, turn/start@6` | `(6, undefined, 5)` |
| c 未闭合压缩 | `compaction/start@3` | `(null, <start@3>, undefined)` |
| d 已闭合压缩 | `compaction/start@3, compaction/end@7` | `(null, undefined, undefined)` |
| e 未闭合 turn | `turn/start@2, turn/start@4` | `(4, undefined, undefined)` |
| f 事务内自锁 | `compaction/start@9`（由 `region.ts:230` 自己 append） | `(…, <start@9>, …)` → `assertCompactionInactive` 抛 `busy` |

场景 f 是最关键路径：`compactSurfaceRegion` 读取入口状态后自己 append `compaction/start`，
之后下一次读取（如 `agent/pre-step`）必须看到它，否则压缩锁失效——游标单调前进保证了这一点。

---

## 5. 静态核对结果（steps.md S5）

| # | 核对项 | 结果 |
|---|---|---|
| 5.1 | `grep "seq -= 1" region.ts` | **0 命中** ✔ |
| 5.2 | `grep "^export" region.ts` | 仍 3 个：`selectCompactableRange`(:137)、`compactSurfaceRegion`(:193)、`assertNoActiveCompaction`(:347)，签名逐字未变 ✔（`git.diff` 确认无 export 行变更） |
| 5.3 | `Read region.ts:8-25` | import 块与 HEAD 逐行相同，无新增包/符号 ✔ |
| 5.4 | `grep "new WeakMap" region.ts` | 1 命中 `<Session, CompactionEntryFold>`（:90）✔ |
| 5.5 | `oxlint-disable-next-line` 计数 | HEAD = 11，改后 = 11（删 1 加 1，类型集合不变）✔ |
| 5.6 | 全仓 `inspectCompactionEntryState` | 仅定义(:610) + 调用(:204、:348)；对外入口 `assertNoActiveCompaction` 由 `src/index.ts:26` 再导出、`index.ts:305` 调用，均未改动 ✔ |
| 5.7 | `git diff --stat` | 仅 `region.ts` 1 文件改动，无其它文件被触碰 ✔ |
| 5.8 | 新增分支可达性 | 仅 `fold === undefined`（首次调用必达）与 `while`（必达）；**无不可达防御分支** ✔ |
| 5.9 | 类型静态核对 | `SessionSeq(value: number)`（`session/src/types.ts:39`）接受 `number`；`SessionSeq = BrandedNumber<'SessionSeq'>` 可赋给 `number` ⇒ `fold.cursor < session.seq` 比较合法；`event.data.turn` 为 `number`（`session/src/types.ts:288`）✔ |
| 5.10 | 人工复查括号/缩进 | 已 Read 新增块 + 函数体两遍，结构闭合、缩进与文件现有风格一致 ✔ |

---

## 6. 与 plan.md / steps.md 的差异（无计划冲突）

- **无功能冲突**。plan 的 §3.1/§3.2 代码形状与本文实现一致。
- 仅两处**细节调整**，均已按「保证等价性与签名不变」的最安全方式处理：
  1. **行号位移** ——新增 20 行后，`selectCompactableRange` 117→137、`compactSurfaceRegion` 173→193、
     `assertNoActiveCompaction` 327→347，目标函数 565-600→585-645、`SessionSeq` 唯一构造点仍在函数内。
     步号 L327-334 / L565-600 等引用需按新行号理解。
  2. **注释语言** ——源文件注释统一为英文（与 `region.ts` 现有注释一致），故 JSDoc/行注释用英文书写，
     含义与 plan §3.2 的三条不变式 + 「非 latest-wins 须回退全量扫描」一致；plan 中建议的
     `// 连续 session seqs 索引持久日志；…` 已译为
     `// The fold cursor indexes the persistent log by contiguous session seqs.`
     （与 `token-meter` 的 `// Contiguous session seqs index the durable log; …` 同源）。
- **未采纳** `plan §3.4` 里「假设性日志变短」的重置守卫——按主 Agent 裁定保持 0 新增不可达分支，
  改为在 JSDoc 中点明「无守卫」这一决策及其理由（见 §2.3 第 2 条）。

---

## 7. 回退方式

`git checkout -- packages/compaction/compaction-basic/src/region.ts` 即完全恢复：
新增内容仅 1 个 interface + 1 个 WeakMap 常量 + 1 个函数体，无跨文件耦合、无持久化数据结构、无配置项。

---

## 8. 待在可运行环境补跑（本环境无法执行）

- `packages/compaction/compaction-basic/tests/`（`compaction-basic.spec.ts`、
  `compaction-loop-repro.spec.ts`、`loader-composition.spec.ts`、`manual-compaction.spec.ts`）
- `packages/compaction/command-compact/tests/`
- `benchmarks/long-session-browser/`（观察长会话每步耗时下降）
- 单文件 per-file 100% 覆盖门禁（确认无新增未覆盖分支）

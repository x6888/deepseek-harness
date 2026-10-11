# T1 原子步骤 — `region.ts` `inspectCompactionEntryState()` 增量 fold

> 配套 `plan.md`（目标、逐行分析、等价性论证、风险）。
> 环境无法 build/run → 每步的「核对」均为**静态/结构核对**。
> 目标文件：`packages/compaction/compaction-basic/src/region.ts`（唯一被修改的文件）
> 归档目录：`sandbox/deepseek-harness/optimization/T1/`

## 步骤总览

| 步 | 动作 | 产物 | 回滚 |
|---|---|---|---|
| S1 | 定位与事实核对（已完成） | 本目录下的 `plan.md` | — |
| S2 | 新增 `CompactionEntryFold` interface + 模块级 WeakMap | region.ts 新增 ~20 行 | 删除新增块 |
| S3 | 用增量 fold 重写 `inspectCompactionEntryState()`（替换 L565-600） | region.ts 替换 ~35 行 | 还原原函数 |
| S4 | 补不变式注释 | region.ts 注释 | — |
| S5 | 结构静态核对（无倒扫 / 导出与 import 未变 / lint 注释一致） | 核对结论（写进交付报告） | — |
| S6 | 等价性走查（逐事件转移表 + 6 场景） | 核对结论 | — |
| S7 | 复杂度/收益说明 + 交付报告（标注未做运行时验证） | 返回主 Agent 的总结 | — |

---

## S1 定位与事实核对 ✅（已完成）

**内容**：读 `CONTEXT.md` / `SOURCE_ANALYSIS.md` / `MANIFEST.md` / `region.ts`，并核对
`packages/core/session/src/index.ts` 的日志模型、`packages/llm/token-meter/src/index.ts` 的
`_sync()` 先例、`turn/start` 事件数据形状。

**产物**：`optimization/T1/plan.md`。

**静态核对**：
- `region.ts:572` 存在 `for (let seq = session.seq - 1; seq >= 0; seq -= 1)`（倒扫） ✔
- `region.ts:595-597` 三条件 break ✔
- `session/src/index.ts:680-683` `seq === log.length`；全文 `this.log` 写入仅 L587 / L761（只 push） ✔
- `session/src/types.ts:288` `'turn/start': { turn: number }` ✔
- `token-meter/src/index.ts:219-240` `_sync()` + L108 `WeakMap<Session, ReplayState>` ✔

---

## S2 新增折叠状态类型与 WeakMap

**动作**：在 `CompactionEntryState`（L66-70）之后插入：

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

**预期产物**：region.ts 中新增 1 个（非导出）interface + 1 个模块级常量，约 20 行；**import 块一行不改**。

**静态核对**：
1. 无新增 import：`region.ts` L8-25 保持原样（游标用 `number`，不引入 `SessionLogOffset`）。
2. 无新增 `export`：新增 interface/常量均不带 `export`（与 `CompactionEntryState` 一致）。
3. 字段类型与 `CompactionEntryState`（L66-70）逐字同源：`number | null` / `SessionEvent<'compaction/start'> | undefined` / `SessionSeq | undefined`。
4. 命名风格：`CompactionEntryFold`（类型，PascalCase）/`compactionEntryFolds`（常量，camelCase），与文件现有命名一致。

**回滚**：删除该新增块。

---

## S3 用增量 fold 重写 `inspectCompactionEntryState()`

**动作**：替换 `region.ts:565-600`（含 L565 的 JSDoc）为 §3.2 的增量实现：
取/建 fold → `while (fold.cursor < session.seq)` → `session.eventAt(SessionSeq(fold.cursor))!`
→ `if/else if` 五类型覆盖 → `fold.cursor += 1` → 返回新对象。

**预期产物**：函数体 ~30 行；**签名 `(session: Session): CompactionEntryState` 一字不改**；
`CompactionEntryState` 定义不变；两个调用点（L184、L328）不动；`assertNoActiveCompaction`（L327-334）不动。

**静态核对**：
1. **倒扫已消失**：文件中不再出现 `seq -= 1` / `seq >= 0` / `session.seq - 1`（`Grep` 模式 `seq -= 1`）。
2. **折叠是前向且单调**：仅 `fold.cursor += 1` 一处推进；无 `cursor -= 1`。
3. **覆盖域**：`while (fold.cursor < session.seq)` + 冷启动 `cursor: 0` ⇒ 覆盖 `[0, seq)`，与原倒扫域相同。
4. **事件读取写法与现状一致**：`session.eventAt(SessionSeq(...))!` 上方保留
   `// oxlint-disable-next-line typescript/no-non-null-assertion, typescript/no-deprecated`（与 L574-575 逐字一致）。
5. **窄化写法沿用 `if / else if`**（不用 `switch`），分支顺序与原实现一致：
   `session/end-seed` → `compaction/start` → `compaction/end` → `turn/start` → `turn/end`。
6. **返回新对象**，不把 `fold` 直接返回（防调用方污染增量状态）。
7. **无新分支不可达**（CI per-file 100% 覆盖）：仅新增 `fold === undefined`（首次调用必达）与 `while`（必达）。

**回滚**：还原 L565-600 原函数（其余新增同步删除）。

---

## S4 不变式注释

**动作**：把三条不变式写进 `inspectCompactionEntryState` 的 JSDoc（见 plan §3.2）：
① 三个量都是 latest-wins，故前扫覆盖 ≡ 倒扫首次命中，原 break 只是提前退出；
② 日志 append-only + seq 连续 ⇒ 已折叠前缀永不失效；
③ 缓存缺失只影响耗时（非 latest-wins 的新量必须改回全量扫描）。

**预期产物**：函数上方一段 JSDoc（与文件现有的「doc comment + 说明性注释」密度一致，见 L92-106 / L157-172）。

**静态核对**：注释中出现的三个事件组（`session/end-seed`、`compaction/start|end`、`turn/start|end`）与代码分支一致；
文件仍是「每个函数/常量都有 JSDoc」的风格（对齐 `scripts/verify-*` 的导出 JSDoc 门禁，本函数非导出但保持风格）。

---

## S5 结构静态核对（改完后必做）

对 `region.ts` 逐条执行（命令：Grep / Read）：

| # | 核对项 | 期望 |
|---|---|---|
| 5.1 | `Grep "seq -= 1" region.ts` | 0 命中 |
| 5.2 | `Grep "^export" region.ts` | 仍为 `selectCompactableRange`（L117）、`compactSurfaceRegion`（L173）、`assertNoActiveCompaction`（L327）三个，签名逐字未变 |
| 5.3 | `Read region.ts:8-25` | import 块与改动前逐行相同（无新增包/符号） |
| 5.4 | `Grep "new WeakMap" region.ts` | 1 命中，泛型为 `<Session, CompactionEntryFold>` |
| 5.5 | `Grep "oxlint-disable-next-line" region.ts` | 命中数 = 改动前数量（类型集合不变，仅位置迁移） |
| 5.6 | `Grep "inspectCompactionEntryState" 全仓` | 仅 `region.ts:184` / `:328` 两处调用 + 定义一处 |
| 5.7 | `Read region.ts` 全文 diff 目视 | 除 S2 新增块与 S3 函数体外无其它改动 |
| 5.8 | 新增分支可达性 | `fold === undefined` 首次调用必达；无 `cursor > seq` 之类的不可达防御分支（除非主 Agent 明确要求并接受覆盖门禁风险） |

---

## S6 等价性走查（静态，替代运行验证）

**逐事件转移表**（plan §3.3）：对 5 个事件类型 + 其它类型，逐一比对
「原倒扫首次命中即定」与「新前扫逐个覆盖」的终值 —— 6 行全部一致。

**6 个边界场景手工 trace**（每个场景给出原/新的三元组，必须逐位相等）：

| 场景 | 日志（简化） | 期望三元组 `(openTurn, unmatchedCompactionStart, latestEndSeedSeq)` |
|---|---|---|
| a 空日志 | — | `(null, undefined, undefined)` |
| b seed 会话 | `…, session/end-seed@5, turn/start@6` | `(6, undefined, 5)` |
| c 未闭合压缩 | `compaction/start@3` | `(null, <start@3>, undefined)` |
| d 已闭合压缩 | `compaction/start@3, compaction/end@7` | `(null, undefined, undefined)` |
| e 未闭合 turn | `turn/start@2, turn/start@4` | `(4, undefined, undefined)` |
| f 事务内自锁 | `…, compaction/start@9`（由 L210 自己 append） | `(…, <start@9>, …)` → `assertCompactionInactive` 抛 `busy` |

**重点路径核对**：`compactSurfaceRegion` L184 读取 → L210 append `compaction/start` → 后续
`agent/pre-step` 的 `assertNoActiveCompaction` 再次读取 ⇒ 折叠必须看到 L210 的事件（游标推进保证），
否则压缩锁失效（这是本次改动最不能出错的一条）。

**复杂度核对**：before 每步 O(N)（无 end-seed ⇒ 从不 break）；after 每步 O(Δ)，会话累计 O(N + ΣΔ)。

---

## S7 交付说明与报告

**产物**：返回主 Agent 的一句话总结 + `plan.md` / `steps.md` 两路径 + 最关键风险点
（R1 等价性依赖 latest-wins；R2 WeakMap key = Session 身份与覆盖门禁的权衡）。

**必须标注**：本环境未做运行时验证；建议在可运行环境补跑
`packages/compaction/compaction-basic/tests/`（4 个 spec）与 `packages/compaction/command-compact/tests/`，
以及 `benchmarks/long-session-browser/` 观察长会话收益。

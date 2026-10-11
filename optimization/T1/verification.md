# T1 验证报告 — `region.ts` `inspectCompactionEntryState()` 增量 fold

> 流程 4.4 验证者产出。**只验证，未修改任何代码**（本报告是唯一写入产物）。
> 环境无法 build/run（无网络、依赖未装，见 `CONTEXT.md` §5）→ 验证为**静态/结构核对**：
> `Read` 改后源码 + `git diff` 与 HEAD 对比 + 依赖事实 grep + 等价性逻辑走查。
> 行号对应改后快照 `packages/compaction/compaction-basic/src/region.ts`（646 行）。

---

## 0. 验证基线与证据来源

| 证据 | 来源 | 结论 |
|---|---|---|
| 改动范围 | `git status --porcelain` / `git diff --stat` | 仅 `packages/compaction/compaction-basic/src/region.ts` **1 file changed, 75 insertions(+), 30 deletions(-)**；其余条目均为未跟踪文档（`CONTEXT.md`/`MANIFEST.md`/`SOURCE_ANALYSIS.md`/`optimization/`）✔ |
| 改动内容 | `git diff -- region.ts` | 仅两块：① 新增 `CompactionEntryFold` interface + `compactionEntryFolds` WeakMap；② 替换 `inspectCompactionEntryState` 函数体与其 JSDoc。无其它任何行被改动 ✔ |
| 事实基线 | `packages/core/session/src/index.ts`、`packages/core/session/src/types.ts`、`packages/llm/token-meter/src/index.ts` | 见 §2/§6 |

---

## 1. 验收标准 1 — 增量 fold 替代全量倒扫，且保留原语义

**判定：✅ PASS**

证据：

1. **倒扫已消失**：`region.ts` 全文 grep `seq -= 1` / `seq >= 0` / `session.seq - 1` → **0 命中**（改后文件内 `fold.cursor` 仅出现在 621/625/637 三行）。
2. **增量折叠结构成立**（`region.ts:610-645`）：
   - `compactionEntryFolds.get(session)`（:611），缺失时以 `cursor: 0` 冷启动并 `set`（:612-620）；
   - `while (fold.cursor < session.seq)`（:621）只消费 `[cursor, session.seq)`；
   - `fold.cursor += 1`（:637）单调推进，全文无 `cursor -= 1`；
   - 返回新对象 `{ openTurn, unmatchedCompactionStart, latestEndSeedSeq }`（:639-644），不交出 fold 引用。
3. **仿 `TokenMeter._sync()` 成立**：`token-meter/src/index.ts:108` `private readonly states = new WeakMap<Session, ReplayState>()`，`:219-240` 的 `_sync()` 为「取/建 state → `while (state.consumedEvents < session.seq)` → `eventAt(SessionSeq(...))!` → 折叠 → 游标 +1」。新实现同形；差异仅在游标类型用 `number`（避免新增 `SessionLogOffset` import，plan §3.2 已论证并采纳）。
4. **状态类型**：`CompactionEntryFold`（:77-83）三字段类型与 `CompactionEntryState`（:66-70）逐字同源（`number | null` / `SessionEvent<'compaction/start'> | undefined` / `SessionSeq | undefined`），仅多一个 `cursor: number`。
5. **语义保留**：返回值仍为同一 `CompactionEntryState` 三元组（:640-644 与 HEAD `return { openTurn, unmatchedCompactionStart, latestEndSeedSeq }` 字段、顺序、类型一致）；下游判定 `assertCompactionInactive`（:328-340）、`compactSurfaceRegion` 的 owner/锁分支（:204-222）**一行未改**，输入不变 ⇒ 判定不变。

**复杂度核对（静态推断）**

| | before | after |
|---|---|---|
| 单次调用 | `for (seq = session.seq - 1; …; seq -= 1)` 全日志倒扫；`latestEndSeedSeq !== undefined` 依赖 `session/end-seed`（仅构造期产生，`session/src/index.ts:619/621`）⇒ 非 seed 会话**从不 break** ⇒ 每步 O(N) | 折叠 `[cursor, session.seq)` ⇒ **O(Δ)**（Δ = 上次读取后新增事件数） |
| 会话 S 步累计 | O(S·N) | O(N + ΣΔ) ≈ O(N)，每个事件一生只折叠一次 |

⇒ O(N)/步 → O(新事件)/步 成立。

---

## 2. 验收标准 2 — 正确处理新增事件，且增量 ≡ 全量扫描

**判定：✅ PASS**

**2.1 「游标之后的新事件」处理正确**

- 循环条件 `fold.cursor < session.seq`（:621）与冷启动 `cursor: 0`（:615）⇒ 覆盖域恰为 `[0, session.seq)`，与原倒扫 `[0, seq)` 相同。
- 关键自锁路径走查：`compactSurfaceRegion` 在 :204 读取入口状态 → :230 `session.append('compaction/start', …)` → 下次调用（如 `agent/pre-step` → `index.ts:305` `assertNoActiveCompaction`）时 `session.seq` 已增大，while 折叠到该事件 ⇒ `unmatchedCompactionStart` = 该 start ⇒ `assertCompactionInactive`（:328）抛 `ManualCompactionError('busy')`。**压缩锁仍被检出**，与原实现一致（这条是本次最不能出错的路径）。

**2.2 日志模型前提（增量安全的事实依据，独立 grep 核对）**

| 前提 | 证据 |
|---|---|
| `eventAt(seq) === log[seq]` | `session/src/index.ts:635-636` `eventAt(seq) { return this.log[seq] }` ✔ |
| `session.seq === log.length` | `session/src/index.ts:681-682` `get seq() { return SessionLogOffset(this.log.length) }` ✔ |
| seq 连续自 0 | `session/src/index.ts:746` `seq: SessionSeq(this.log.length)`（append 时按当前长度分配）✔ |
| 日志**只 push 不截断** | `session/src/` 内 `log.push` 仅两处：`:587`（构造期 seed 一次性灌入）、`:761`（`append`）；全目录 `splice/pop/shift/length=` 仅命中 `projections`（:948）与 `surface.state.nodes`（`surface.ts:576`），**均非 log** ✔ |
| 事件不可变 | 构造/append 走 `deepFreeze`（与 `session/src/index.ts` 现有不变式一致）✔ |
| `turn/start` 数据形状 | `session/src/types.ts:288` `'turn/start': { turn: number }` ⇒ `event.data.turn` 类型与 HEAD 一致 ✔ |

⇒ 已折叠前缀不可变，「只消费新增段」与「全量扫描」覆盖同一序列且每事件恰好处理一次。

**2.3 等价性逐事件推演（原倒扫「首次命中即定」 vs 新前扫「逐个覆盖」）**

| 事件类型 | 原倒扫终值 | 新前扫终值 | 一致 |
|---|---|---|---|
| `session/end-seed` | 自新→旧首个（=最新）写入 `event.seq`，随后被 `=== undefined` 守卫挡住 | 每次覆盖 ⇒ 最新 end-seed 的 seq | ✅ |
| `compaction/start` | `!known` 时记录该 start 并置 known | 覆盖为该 start | ✅ |
| `compaction/end` | `!known` 时置 known，`unmatched` 保持 `undefined` | 清空 `unmatched` | ✅ |
| `turn/start` | `openTurn = data.turn` 并置 known | 覆盖 `openTurn = data.turn` | ✅ |
| `turn/end` | 置 known，`openTurn` 保持 `null` | `openTurn = null` | ✅ |
| 其它类型 | 无副作用 | 无副作用 | ✅ |

**break 无副作用引理**：HEAD 的三条件 break 位于循环末尾，且三个 `xxxKnown`/`=== undefined` 守卫保证字段一旦定值即不再改写 ⇒ break 只是提前退出，不改变输出；新实现无 break 但覆盖域相同 ⇒ 终值相同。

**边界场景 trace（原 vs 新，三元组 `(openTurn, unmatchedCompactionStart, latestEndSeedSeq)`）**

| 场景 | 日志 | 原 | 新 |
|---|---|---|---|
| a 空日志 | — | `(null, undefined, undefined)` | 同（while 不执行） |
| b seed 会话 | `session/end-seed@5, turn/start@6` | `(6, undefined, 5)` | `(6, undefined, 5)` |
| c 未闭合压缩 | `compaction/start@3` | `(null, start@3, undefined)` | 同 |
| d 已闭合压缩 | `compaction/start@3, compaction/end@7` | `(null, undefined, undefined)` | 同（被 end 清空） |
| e 连续 turn/start | `turn/start@2, turn/start@4` | `(4, undefined, undefined)` | 同 |
| f start…end…start | `start@1, end@2, start@3` | `start@3` | `start@3` |
| g 事务内自锁 | `compaction/start@9`（:230 自 append） | `start@9` → 抛 `busy` | 同 |

⇒ 同一日志序列下结论一致，标准 2 成立。

---

## 3. 验收标准 3 — 仅单文件、无新依赖、风格一致

**判定：✅ PASS**

| 核对项 | 期望 | 实测 |
|---|---|---|
| 改动文件数 | 1 | `git diff --stat` = 1 file（`region.ts`），其余为未跟踪文档 ✔ |
| import 块未变 | L8-25 逐行相同 | `git show HEAD:…| sed -n 1,25p` 与改后 `sed -n 1,25p` **md5 相同**（`d66521576361a7d3f933580267be01c8`）✔ |
| 无新依赖 | 不新增包/符号 | 新增块只用到已 import 的 `Session`/`SessionSeq`/`SessionEvent`（:22）；`WeakMap` 为内置 ✔ |
| 无新 export | 新增 interface/常量均不导出 | `CompactionEntryFold`（:77）、`compactionEntryFolds`（:90）均无 `export` ✔ |
| 风格一致 | 沿用现有写法 | ① 沿用 `if/else if` 判别联合窄化链（:626-636，分支顺序与 HEAD 一致：`session/end-seed`→`compaction/start`→`compaction/end`→`turn/start`→`turn/end`）；② 沿用同一条 `// oxlint-disable-next-line typescript/no-non-null-assertion, typescript/no-deprecated` 及上方注释「Existing Session history read; migration deferred.」（:622-624）；③ 命名 `CompactionEntryFold`/`compactionEntryFolds` 与既有 `CompactionEntryState` 同构；④ 注释为英文，与文件现有注释语言一致；⑤ 每个新增声明都带 JSDoc（:72-76、:85-89）✔ |
| lint 注释计数不漂移 | 数量不变 | HEAD `oxlint-disable-next-line` = **11**，改后 = **11**（122/150/156/164/170/172/368/372/570/576/624）✔ |

> 文档小瑕疵（不影响代码）：`implementation.md` §5.5 描述为「删 1 加 1」，实际是原 disable 行作为上下文保留、净数不变（11 → 11）。结论仍为「不触发新 lint 项」，仅表述需修正。

---

## 4. 验收标准 4 — 保留现有导出与调用方语义

**判定：✅ PASS**

| 核对项 | 期望 | 实测 |
|---|---|---|
| 导出集合与签名 | 3 个，逐字未变 | HEAD: `selectCompactableRange`(:117) / `compactSurfaceRegion`(:173) / `assertNoActiveCompaction(session: Session, stage: string): void`(:327)；改后 :137 / :193 / :347（行号整体 +20，签名逐字相同）✔ |
| `inspectCompactionEntryState` 仍模块私有 | 无 `export` | :610 `function inspectCompactionEntryState(session: Session): CompactionEntryState`，签名一字未改 ✔ |
| 调用点未动 | 2 处 | `region.ts:204`（`compactSurfaceRegion` 内）、`region.ts:348`（`assertNoActiveCompaction` 内）；对外再导出 `src/index.ts:26` 与调用 `src/index.ts:305` 均**不在 diff 内** ✔ |
| 全仓调用面 | 仅本文件 | grep 全 `packages/`：仅 :204 / :348 调用 + :610 定义 + `index.ts:26/305` ✔ |
| 行为兼容 | 调用方无需改动 | 返回类型 `CompactionEntryState` 的字段集合/类型/顺序不变；`CompactionEntryState` 定义（:66-70）未被修改；返回新对象（原实现亦返回字面量）⇒ 调用方只读语义不变 ✔ |

---

## 5. 验收标准 5 — before/after 行为说明 + 最小静态核对

**判定：✅ PASS**

`optimization/T1/implementation.md` 已含：

- **before/after 行为说明**：§3 表格（单次调用、S 步累计、冷启动、分配、返回值、内存）+ §1 一句话改动；`plan.md` §7 另给 before/after 叙述。
- **复杂度分析**：§3「O(N)/步 → O(Δ)/步，累计 O(S·N) → O(N+ΣΔ)」+ §2.3（`plan.md`）「非 seed 会话从不 break」的根因说明。
- **等价性走查**：§4 逐事件转移表（6 行）+ 覆盖域论证 + 7 个边界场景（含事务内自锁）+ break 无副作用引理。
- **静态核对清单**：§5 表 5.1–5.10（倒扫消失、export 不变、import 不变、WeakMap 泛型、oxlint 计数、调用面、diff 范围、分支可达性、类型静态核对、人工复查）—— 与本报告 §0–§4 的独立复核**结论一致**。
- **未做运行时验证的标注**：implementation.md 抬头、§3 末尾、§8 均显式标注，并列出待补跑的 4 个 spec + `command-compact/tests/` + `benchmarks/long-session-browser/` ✔

---

## 6. 额外核查（主 Agent 裁定项）

**判定：✅ PASS（两项均符合裁定）**

1. **未新增 `cursor > session.seq` 重置守卫**：改后 `region.ts` 中 `fold.cursor` 仅出现于 :621（`< session.seq`）、:625（`SessionSeq(fold.cursor)`）、:637（`+= 1`）；无任何 `cursor >` 比较、无重置分支 ⇒ 0 新增不可达分支 ✔
   - 这与 CI 门禁一致：`vitest.config.ts:370-377` 配置 `perFile: true` 且 statements/branches/functions/lines 均 **100** ⇒ 加不可达守卫会直接挂门禁。裁定「默认不加」正确。
2. **JSDoc 已写明前提不变式**（`region.ts:585-609`，四项齐全）：
   - 日志只 push / `seq ≡ log.length`：:592-594「The log is append-only and `seq` is contiguous from 0 (`eventAt(seq)` is `log[seq]`, `session.seq === log.length`), so an already-folded prefix can never be invalidated.」✔
   - 未加守卫及其理由：:595-597「There is no `cursor > session.seq` reset guard: the log only grows, and the guard would be unreachable. Keep it that way — should that ever change, revisit this whole fold instead of patching it here.」✔
   - 三量为 latest-wins：:598-603 逐个写明 `latestEndSeedSeq` = 最新 `session/end-seed`、`unmatchedCompactionStart` = 最新 `compaction/start`（除非被更新的 `compaction/end` 关闭）、`openTurn` = 最新 `turn/start`（除非被更新的 `turn/end` 关闭），并说明原 `break` 只是提前退出 ✔
   - 非 latest-wins 须回退全量扫描：:604-606「adding a non-latest-wins quantity here (a count, an earliest occurrence, a running total) silently breaks the fold: such a quantity must revert this function to a full scan.」✔
   - 另在 `CompactionEntryFold` 的 JSDoc（:72-76）复述「Every field is a function of the *last* occurrence of a marker, so folding events strictly forwards agrees with the reverse scan's first-hit-wins result.」✔

---

## 7. 优化闭环判定

**达成（闭环）。** 5 条验收标准全部 PASS，2 项额外裁定项全部符合；改动自包含于单文件、无新依赖、导出与调用方零改动，等价性有逐事件推演 + 7 场景 trace + 日志模型事实三重支撑。

### 遗留项（不阻断闭环，建议在可运行环境补做）

| # | 遗留项 | 等级 | 建议 |
|---|---|---|---|
| L1 | **未做运行时验证**（本环境无网络/依赖未装） | 中 | 补跑 `packages/compaction/compaction-basic/tests/`（4 spec）与 `packages/compaction/command-compact/tests/`；`benchmarks/long-session-browser/` 观察长会话每步耗时 |
| L2 | per-file 100% **branch** 门禁：`fold === undefined` 的 false 分支（缓存命中）需被覆盖 | 中 | 静态上可达：`compaction-basic.spec.ts:701` 与 `:703` 对**同一 `session` 对象**连续两次 `compactIfNeeded`，第二次必然命中缓存；且两次之间存在 append（Δ>0 路径）。仍建议在可运行环境用覆盖率报告确认 |
| L3 | `implementation.md` §5.5 表述「删 1 加 1」与实际（原行保留、净数 11→11）不符 | 低 | 文档层面修正，代码无需改动 |
| L4 | 长会话收益为静态复杂度推断，无实测数字 | 低 | 由 L1 的 benchmark 补上实测数据 |
| L5 | fold 强引用一个 `SessionEvent` | 低 | 该事件本就在 session 日志内，WeakMap 保证 fold 随 Session 释放；无需处理 |

---

## 8. 附：核对命令与输出要点（可复现）

```
git status --porcelain                    # M region.ts（其余为未跟踪文档）
git diff --stat                           # 1 file changed, 75 insertions(+), 30 deletions(-)
git diff -- …/region.ts                   # 仅两块新增/替换，无其它行变更
git show HEAD:…/region.ts | grep -c oxlint-disable-next-line   # 11（改后同为 11）
git show HEAD:…/region.ts | grep -n ^export                    # 117/173/327（改后 137/193/347）
diff <(git show HEAD:… | sed -n 1,25p) <(sed -n 1,25p …)       # import 块 md5 一致
grep -n "seq -= 1|seq >= 0|session.seq - 1" region.ts          # 0 命中
grep -n "fold.cursor" region.ts                                # 621/625/637（无 cursor > 守卫）
grep -rn "log.push|\.splice(" packages/core/session/src        # push 仅 587/761；splice 不在 log
```

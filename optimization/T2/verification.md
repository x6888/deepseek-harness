# T2 验证报告 — `TokenMeter.measure()` memo 缓存

> 流程 4.4（验证者）产出。**只读核验，未修改任何代码，未 commit。**
> 环境无法 build/run（依赖未装、无网络）→ 全文结论为**静态/结构核对**，
> 明确标注：**未做运行时验证**（未跑 `tsc` / `vitest` / 覆盖率脚本）。
> 被核文件：`packages/llm/token-meter/src/index.ts`（改后快照，行号 = 改后行号）

> **导航**：本文 §0–§6 为**第 1 轮**结论（判定：未闭环，阻断缺陷 F1），保留作历史。
> **第 2 轮复验结论见 §7 起**，行号以第 2 轮快照为准。两轮合并总判定见 §7.0。

## 0. 结论速览

| 项 | 判定 |
|---|---|
| 标准 1（memo 生效、命中复用已冻结结果） | **❌ FAIL**（机制 ✅ / **实际收益 ❌**：生产热路径永不命中，见 F1） |
| 标准 2（不产生错误复用；键用 llm/pricing 身份而非 `fileText` 闭包） | **✅ PASS**（无错误复用；字面项 ✅）＋ 附 **F1 发现** |
| 标准 3（仅改 `index.ts`、无新依赖、风格一致） | **✅ PASS** |
| 标准 4（导出/签名/调用方语义不变） | **✅ PASS** |
| 标准 5（before/after 说明 + 最小静态核对齐备） | **✅ PASS** |
| 裁定 A（查询在 `_sync` 之后 / 写在 `deepFreeze` 成功之后） | **✅ PASS** |
| 裁定 B（无 `v8 ignore`；命中分支有既有测试覆盖） | **✅ PASS** |
| **优化闭环判定** | **未闭环 ❌** — 正确性无风险，但**优化目标未达成**（生产命中率 ≈ 0），见第 5 节差距清单 |

---

## 1. 改动范围核验（git · 只读）

```
$ git status --porcelain
 M packages/compaction/compaction-basic/src/region.ts      ← T1 遗留（本轮不核）
 M packages/llm/token-meter/src/index.ts                   ← T2 本轮
?? CONTEXT.md  ?? MANIFEST.md  ?? SOURCE_ANALYSIS.md  ?? optimization/   ← 分析产物，非代码

$ git diff --stat
 packages/compaction/compaction-basic/src/region.ts | 105 +++++++++++++++------
 packages/llm/token-meter/src/index.ts              |  55 ++++++++++++++++++++++++++++++++++-
 2 files changed, 129 insertions(+), 31 deletions(-)

$ git diff --stat -- packages/llm/token-meter/src/index.ts
 packages/llm/token-meter/src/index.ts | 55 ++++++++++++++++++++++++++++++++++-
 1 file changed, 54 insertions(+), 1 deletion(-)
```

**T1 / T2 区分（明确切割）**

- `region.ts`（T1 遗留，未提交）：diff 新增内容为 `CompactionEntryFold` / `compactionEntryFolds = new WeakMap<Session, CompactionEntryFold>()` / `while (fold.cursor < session.seq)` 等**增量 fold** 符号；`git diff -- region.ts | grep "token-meter|measurements|MeasurementMemo"` **无任何匹配** ⇒ 与 T2 无耦合。
- `region.ts:293` 的 `/* v8 ignore next */` 经 `git diff | grep "^+"` 核验**不是 T1 新增**（属仓库既有），本轮不涉及。
- `index.ts`（T2 本轮）：`1 file changed, 54 insertions(+), 1 deletion(-)`，全部落在本文件。

⇒ **标准 3 的"仅限本文件"成立**（对 T2 而言）。

---

## 2. 五条验收标准逐条核对

### 标准 1 — 引入 memo，命中复用已冻结结果，避免重复 `structuredClone` + `deepFreeze`

**机制：✅ 成立**

- `index.ts:82-93` 新增 `interface MeasurementMemo`（`revision` / `header` / `pricing` / `llm` / `result`，全 `readonly`）。
- `index.ts:135` 新增 `private readonly measurements = new WeakMap<Session, MeasurementMemo>()`（单槽，随 Session 回收）。
- `index.ts:189-196` 命中分支：`return memo.result`，**位于** `index.ts:228` 的 `deepFreeze(structuredClone({...}))` **之前** ⇒ 命中即跳过 `priceSurface`（L198）、三分支内的 `estimateToolsTokens`（L209/223）、`structuredClone` + `deepFreeze`（L228）。
- `memo.result` 的唯一写入点 = `index.ts:236-242` 的 `this.measurements.set(...)`，其 `result` 直接来自 L228 `deepFreeze(structuredClone(...))` 的返回值；`deepFreeze<T>(value: T): T`（`packages/util/values/src/index.ts:210`）原地冻结并原样返回 ⇒ `memo.result` **确定是已冻结对象**，复用安全。

**实际收益：❌ 不成立（F1）**

键中的 `pricing` 项在**生产路径上身份不稳定** ⇒ `memo.pricing === pricing`（`index.ts:193`）恒为 `false` ⇒ **memo 永不命中**。证据链：

| # | 位置 | 事实 |
|---|---|---|
| 1 | `token-meter/src/index.ts:188` | `const pricing = this._routeImagePricing(header)` |
| 2 | `token-meter/src/index.ts:247-251` | `_routeImagePricing()` ⇒ `this.ctx.get('llm')?.imageRequestPricing(config.provider, config.model)`（每次现算，无缓存） |
| 3 | `packages/llm/llm/src/index.ts:676-678` | `imageRequestPricing(provider, model) { return this.adapters.get(provider)?.adapter.imageRequestPricing(provider, model) }` — **无任何记忆化** |
| 4 | `packages/llm/llm-deepseek/src/adapter.ts:40-42` | `override imageRequestPricing(...) { return imagePricing(this.dependencies.options(), model, this.imageAccess) }` |
| 5 | `packages/llm/llm-deepseek/src/images.ts:11` | `export { deepSeekImageRequestPricing as imagePricing } from './request-pricing.ts'` |
| 6 | `packages/llm/llm-deepseek/src/request-pricing.ts:90-111` | 两条 return 均为**新对象字面量**：L97 `return { priceImages: images => images.map(textOnlyPrice) }`、L99-110 `return { priceImages: images => images.map(...) }` ⇒ **每次调用产生新引用** |
| 7 | `packages/test-support/llm-replay/src/index.ts:881-890` | 另一处生产级 adapter 同样是 `return { priceImages: ... }` 字面量 ⇒ 同类问题（benchmark/replay 路径亦不命中） |

**命中条件因此退化为**：仅当 `pricing === undefined` 时（无 `request/header` 事件 / `llm` 未挂载 / adapter 对该 model 返回 `undefined`）才可能命中。
真实 agent 运行（`llm` 已挂载、日志已有 `request/header` ⇒ `header.config` 存在 ⇒ deepseek adapter 返回新对象）下，
`plan.md` 2.2 声称的**主收益点 `region.ts:386 → region.ts:456/484` 同 revision 双调用不会命中**，
`implementation.md` §6 的"5 次调用 → 4 次重算"**在生产不成立**（仅在测试/无 llm 场景成立）。

⇒ **判定：❌ FAIL**（T2 的立项目标 — 消除压缩路径上的重复全量测量 — 未达成；代码本身无正确性缺陷）

---

### 标准 2 — 缓存不产生错误复用；键覆盖所有影响结果的输入；特别核对 "用 llm/pricing 身份而非每次新建的 `fileText` 闭包"

**字面项（主Agent 特别要求）：✅ PASS**

- `index.ts:187` `const llm = this.ctx.get('llm')`（服务实例引用，一次 map 查找）。
- `index.ts:188` `const pricing = this._routeImagePricing(header)`（adapter 声明的 pricing 对象引用）。
- `fileText`（`index.ts:197`，由 `_fileRequestText()` L254-259 每次 `return ref => llm.fileRequestText(ref)` 新建闭包）**未入键** ✅ —— 正是 plan/implementation 的设计，`MeasurementMemo` 的 JSDoc（L78-80）明确写出理由。
- ⇒ 不存在"因 `fileText` 闭包引用恒不等而永不命中"的问题。

**键覆盖性（逐结果字段回溯，独立复核）：✅ 无遗漏**

| 结果字段（L229-234） | 自由变量 | 入键项 |
|---|---|---|
| `logRevision` | `state.consumedEvents` | `revision`（且它本身就是返回字段，命中时天然一致） |
| `nodes` / `surfaceTokens` | `priceSurface(state.surface, pricing, fileText)` | Session(WeakMap 键) + `revision` ⊕ `pricing` ⊕ `llm`（`fileText` 的来源） |
| `baseline`（分支条件 L203） | `state.anchor` ⊕ `optionalHeaderEquals(anchor.header, header)` | `revision` ⊕ `header`（等值） |
| `baseline`（值 L207-215/218/221-224） | `estimateToolsTokens(header)` ⊕ anchor 重定价（`pricing`/`fileText`） | `header` ⊕ `pricing` ⊕ `llm` ⊕ `revision` |
| `surfaceDeltaTokens` | `surface.surfaceTokens` − anchor 重定价 | 同上 |
| `totalTokens` | 派生自上面两项 | 已被覆盖 |

`revision` 唯一决定 `ReplayState` 的前提经复核成立：`ReplayState` 只由 `_sync()` 初始化（L272-283）与 `_foldEvent()`（L300-382，唯一状态转移，仅调用 `canonicalHeader` / `planSurfaceTokens` / `commitSurfaceTokens` / `estimateContent` / `estimateMessage` / `estimateToolsTokens` / `assembleAssistantStream`，均为纯函数）写入；session 日志 append-only、`seq` 单调。

**失效（miss-safe）：✅ 保守方向正确**

`revision` 变化（任何事件落账，含 `compaction/start`/`step/start` 等非 surface 事件）、`header` 不等值（`optionalHeaderEquals` → `headerEquals` → `callConfigEquals` + tools `sameSchema`）、`pricing`/`llm` 换引用，均导致 miss。非 surface 事件也 bump `revision` ⇒ 宁可多算，不会错复用。

**唯一理论缝隙（已文档化，不计 FAIL）**：若同一 `pricing` 对象**内部**状态（如 adapter 连接快照、access 路径解析）在同 revision 内静默变化，而引用不变 ⇒ 会复用。此为前提性风险，已在 `implementation.md` §4 末与 R3 以文档（而非分支）记录，与 T1 处理手法一致；且在生产因 F1（引用每次都变）实际不可能触发。

⇒ **判定：✅ PASS**（正确性无风险）。**但 F1 需并入标准 1 判定**：`pricing` 虽不是"每次新建的闭包"，却同样是"每次新建的对象"，plan 只在 `fileText` 上识别了该风险，未在 `pricing` 上识别。

---

### 标准 3 — 仅改 `index.ts`（本轮）、无新依赖、TS 风格一致

- `git diff --stat -- packages/llm/token-meter/src/index.ts` = **1 file，+54 / -1** ✅（全仓 `git diff --stat` 的第二行 `region.ts` 为 T1 遗留，见第 1 节）。
- **零新增 import**：`git diff` 全文无 `^+import` 行；用到的 `SessionLogOffsetType`(L17) / `EpochHeader`(L14) / `LlmImageRequestPricing`(L11) / `LlmRuntime`(L11) / `TokenMeasurement`(L28-32) 全部为既有导入 ✅。
- **无新依赖**：无 import、无 package.json 改动 ✅。
- **风格**：`readonly` 字段 + 逐字段 JSDoc（对照既有 `MeasurementAnchor` L52-60 / `ReplayState` L62-68）；字段声明 `private readonly measurements = new WeakMap<Session, MeasurementMemo>()`（L135）与上一行 `states`（L133）对称；新增代码无 `!` / `as` / `any` / `??` / `?.` ✅。

⇒ **✅ PASS**

---

### 标准 4 — 保留现有导出与调用方语义

- **导出面零改动**：diff 中唯一 `-` 行是 L228 `return deepFreeze(` → `const result = deepFreeze(`；`export class TokenMeter`(L126)、`measure(session, requestHeader?)`(L180)、`estimateMessage`(L267)、3 处 `export type *`(L40/44/45)、`export default TokenMeter`(L395) 均无 `^-` 行 ✅。
- **签名不变**：`measure(session: Session, requestHeader?: EpochHeader): TokenMeasurement` ✅。
- **返回值等价**：未命中路径字段与顺序逐字未动（L229-234）；命中路径返回同一冻结对象，**唯一可观察差异是对象身份**（`Object.is(a,b) === true`）。`TokenMeasurement` 全字段 `readonly` 且经 `deepFreeze` ⇒ 共享只读安全。
- **调用方零改动**：全仓生产调用点 9 处（`compaction-basic/src/index.ts:278/297/325/339/396`、`region.ts:386/456/484`、`acp/acp/src/updates.ts:99`，另 `compaction-loop-repro.spec.ts` 等测试）均无需改动，且全部为只读用法（`nodes.slice()` / `isDeepStrictEqual` / `reduce` / `totalTokens`）✅。
  - 其中 `region.ts:457` `isDeepStrictEqual(current.nodes, prepared.measurement.nodes)` 在命中时退化为同一引用比较：语义仍一致（命中 ⇔ revision 未变 ⇔ surface 未变；surface 若真变则 revision 必变 ⇒ 不命中）✅。
- **抛错语义不变**：见裁定 A ✅。

⇒ **✅ PASS**

---

### 标准 5 — before/after 行为说明 + 最小静态核对齐备

`optimization/T2/implementation.md` 已含：

- **Before/After**：§3.1（代码结构对照）、§3.2（行为表：返回值 / 身份 / 可变性 / 抛错 / 复杂度）✅
- **键覆盖性分析**：§4 七行逐字段表 + 各项比较方式依据表 + 失效条件穷举 9 条 ✅
- **收益**：§6 成本项表（命中时 `priceSurface` / `estimateToolsTokens` / `structuredClone` / `deepFreeze` 归零）与调用次数 ✅
- **门禁自检**：§5 逐条列出新增元素与覆盖来源 ✅
- **范围核验**：§2 附 `git diff --stat` 原文并显式声明 `region.ts` 为 T1 遗留 ✅
- **未做运行时验证声明**：§6 末 + 应跑套件清单 ✅

计划要求的"最小静态核对清单"（`plan.md` 第 8 节 7 条）在 `implementation.md` §3.3 / §4 / §5 / §2 均有对应落点，且为字段级证据而非"应该没问题" ✅

⇒ **✅ PASS**（注：§5/§6 的**收益结论**受 F1 影响需重述，见差距 G1/G4）

---

## 3. 主Agent 裁定项

### A. memo 查询置于 `_sync()` 之后；写缓存置于 `deepFreeze` 成功之后 —— **✅ PASS**

- **查询位置**：`index.ts:183` `const state = this._sync(session)` 是 `measure()` 的**第一条语句**（其前仅 L181-182 两行注释："The fold runs before any lookup: a malformed event has to fail every call"）；memo 查询在 `index.ts:189-194`，**严格晚于** `_sync` ✅。
  - 因此 `token-meter.spec.ts:462-465` 的 `expectRepeatedFailure()`（`expect(() => service.measure(session)).toThrow(pattern)` ×2，被 :483/:508/:517/:540/:549/:582/:606 七个用例复用）不受影响：malformed 事件在 `_sync`→`_foldEvent` 内抛错（L323/333/350），且在抛错前 `state.consumedEvents` 不推进 ⇒ 不会命中也不会写入缓存 ⇒ **每次都抛** ✅。
- **写位置**：`this.measurements.set(...)` 在 `index.ts:236-242`，位于 `const result = deepFreeze(structuredClone({...}))`（L228-235）**成功返回之后**；`return result`（L243）之前 ✅。失败路径（`_sync` 抛错 / `priceSurface` 抛错）均不达写点 ⇒ 失败不缓存、重复调用重复失败 ✅。
- 另：`canonicalHeader(requestHeader)`（L186）也在查询之前，override 非法输入仍每次抛错 ✅。

### B. 未添加 `v8 ignore` / 覆盖率抑制；命中分支有既有测试可覆盖 —— **✅ PASS**

- **无抑制注释**：`grep -rn "v8 ignore|c8 ignore|istanbul ignore" packages/llm/token-meter/` ⇒ **NONE**（src 与 tests 均无）✅。仓库内既有的 `v8 ignore` 仅出现在 `compaction-basic/src/index.ts:284/333/335` 与 `region.ts:293`，**与 T2 无关**（`region.ts:293` 经 `git diff | grep "^+"` 核验非 T1 新增）。
- **命中分支（true 路）有既有测试覆盖的静态证据**：
  1. `token-meter/tests/token-meter.spec.ts:616`（listener 内 `activeMeter.measure(session)`，revision=3，写入 memo）→ **:634** `expect(activeMeter.measure(session).logRevision).toBe(3)` —— 两次调用之间无 append（`:633 expect(revisions).toEqual([3])`）、无 `request/header`（`header === undefined`）、该 ctx **未挂载 llm**（`llm === undefined`、`pricing === undefined`）⇒ 五项全等 ⇒ **必然命中** ✅（这是最硬的一条证据；implementation.md §5 记为 L634，核对一致）
  2. `token-meter.spec.ts:335`（`shrunken`）→ `:338` `service.measure(session, header('different-model'))` —— 同 revision、无 append，但 header 不同 ⇒ 该处覆盖的是 **false 路**（`optionalHeaderEquals` 不等）；而 `:335` 与 `:324` 之间有 append ⇒ 亦为 false 路。故**命中的直接证据以上一条为准**。
  3. 另 `implementation.md` 声称的 `compaction-basic` 成功压缩路径 `region.ts:386 → 456/484` 双调用：该路径上若测试挂载了 llm 且 header 有 config，则因 F1 **不会命中**；若未挂载 llm 则会命中。⇒ 该条**不能作为可靠证据**，以证据 1 为准即可（一条命中路径已足够满足分支覆盖）。
- **false 路**覆盖充分：首次 measure（memo `undefined`，`:166/191/226`）、追加事件后 measure（`:208/261/358/433`）、换 header 后 measure（`:275`、`:388-395`）、`route-pricing.spec.ts:181/188/199/234` ✅。
- **新增 statement**：`measurements` 字段初始化（L135，随任何 `new TokenMeter()` 执行）、`const llm`（L187）、`const memo`（L189）、`measurements.set`（L236）均在每次 `measure()` 或构造时执行 ✅；`&&` 的 4 个操作数在命中路径上全部被求值（证据 1）✅；`memo === undefined` 短路侧由首次 measure 覆盖 ✅ ⇒ 未发现结构上不可达的新增位置。
- `interface MeasurementMemo` 为纯类型，无运行时位置 ✅。

⇒ **裁定 A、B 均 PASS**。

---

## 4. 发现汇总

| # | 发现 | 等级 | 说明 |
|---|---|---|---|
| **F1** | `pricing` 键项在生产身份不稳定 ⇒ memo 永不命中 | **高（阻断立项目标）** | 见标准 1 证据表 1-7 行。`_routeImagePricing()` 每次调用都产生新对象（deepseek adapter / llm-replay adapter 均返回新字面量），`memo.pricing === pricing`（L193）恒 false。T2 目标路径（`region.ts:386→456/484`）零收益。 |
| **F2** | 无正确性风险（正面确认） | — | 键保守、无错复用、抛错语义不变、导出/签名/调用方零改动。 |
| **F3** | 文档小瑕疵 | 低 | `implementation.md` §5 描述"L626 已在 revision=3 上写过 memo"不准确（L626 的 measure 在 append 之前，revision 应为 2；写 memo@rev=3 的是 L616 的 listener）。命中点 L634 的结论本身正确。 |

---

## 5. 优化闭环判定

**判定：未闭环 ❌**（机制与安全性达标，**有效性未达标**）

理由：T2 的立项目标是"消除压缩路径上同输入的重复 `structuredClone` + `deepFreeze`"。
实现在机制上正确（标准 2/3/4/5、裁定 A/B 全 PASS），但因 **F1**，在真实运行（llm 已挂载、header 有 config）下
`memo.pricing === pricing` 恒不成立 ⇒ **命中率 ≈ 0** ⇒ 目标收益（"5 次调用 → 4 次重算"）不达成，
改动反而净增一次 `ctx.get('llm')` 与一次 WeakMap 写入的成本。

### 差距清单（需主Agent 裁定后进入下一轮）

- **G1（阻断）** 修复键中的 `pricing` 身份问题，使生产热路径可命中。可选方向（供裁定，本 Agent 不改码）：
  1. 用 **路由三元组** 取代 pricing 引用：键项改为 `header.config` 的 `provider`/`model`（`header` 已入键且 `optionalHeaderEquals`→`callConfigEquals` 已比较二者）+ `llm` 身份；代价：adapter 在同 provider/model 下重注册/换实现时不会失效，需按 R3 手法以文档记录前提。
  2. 或让 `_routeImagePricing()` 自身按 `(provider, model)` 单槽记忆（仍仅限 `index.ts`），使 pricing 引用在同一路由下稳定；需评估 adapter 重注册的失效语义与新增分支对 per-file 100% 门禁的影响。
  3. 若裁定"仅在 `pricing === undefined` 场景命中也可接受"，则须**重写** `implementation.md` §6 与本报告对应收益声明（不得保留"5→4 次"表述）。
- **G2** 修复后需重新核对 `plan.md` 第 6 节 / `implementation.md` §5 的门禁自检（尤其：若采用 G1-2 会新增分支，须确认两向均有既有测试可达；否则按既有约定**上报主Agent，不得自行加 `v8 ignore`**）。
- **G3（文档）** 修正 `implementation.md` §5 中 L626/L634 的行号描述（F3）。
- **G4** 收益声明须随 G1 结论重述为"命中条件 = `pricing` 未定义（或修复后 = 路由未变）"。

### 未做运行时验证（必须补跑的套件）

本环境无法 `pnpm test` / `tsc`（依赖未装、无网络）。修复后应跑：
- `packages/llm/token-meter/tests/*`（`token-meter.spec.ts`、`route-pricing.spec.ts`、`token-usage-projection.spec.ts`、`context-breakdown-projection.spec.ts`）
- `packages/compaction/compaction-basic/tests/*`
- `packages/compaction/command-compact/tests/*`
- 覆盖率门禁：`vitest.config.ts` + `scripts/coverage-uncovered-locations.cjs`（per-file 100%）

---

## 6. 附：本轮核验使用的命令（只读）

```bash
git status --porcelain
git diff --stat
git diff --stat -- packages/llm/token-meter/src/index.ts
git diff -- packages/llm/token-meter/src/index.ts
git diff -- packages/compaction/compaction-basic/src/region.ts        # T1/T2 切割核验
git grep -n "v8 ignore" -- packages/llm/token-meter packages/compaction/compaction-basic
```

---
---

# 第 2 轮复验（F1 修复后）

> 流程 4.4（验证者）产出。**只读核验，未修改任何源码，未 commit。**
> 环境仍无法 build/run（`node_modules` 缺失、无网络）→ 结论为**静态/结构核对**，
> 明确标注：**未做运行时验证**（未跑 `tsc` / `vitest` / 覆盖率脚本）。
> 被核文件：`packages/llm/token-meter/src/index.ts`（**第 2 轮改后快照**，行号 = 第 2 轮行号）。

## 7.0 结论速览（第 2 轮）

| 项 | 第 1 轮 | 第 2 轮 |
|---|---|---|
| 标准 1（memo 生效、命中复用已冻结结果） | ❌ FAIL（F1：生产永不命中） | **✅ PASS**（F1 已修，生产可命中） |
| 标准 2（不产生错误复用；键覆盖所有影响结果的输入；输入变化即失效） | ✅ PASS（附 F1） | **❌ FAIL**（见 §7.3：`pricing` 键项仅 1 bit 分辨力，且该缝隙已由"不可触发"升格为"可触发"） |
| 标准 3（仅改 `index.ts`、无新依赖、风格一致） | ✅ PASS | **✅ PASS** |
| 标准 4（导出/签名/调用方语义不变） | ✅ PASS | **✅ PASS** |
| 标准 5（before/after + 收益结论；第 1 轮"零收益"应作废） | ✅ PASS | **✅ PASS**（含 1 处门禁自检表述需修正，见 F5） |
| 裁定 A（查询在 `_sync` 之后 / 写在 `deepFreeze` 成功之后） | ✅ PASS | **✅ PASS** |
| 裁定 B（无 `v8 ignore`；命中分支有既有测试覆盖） | ✅ PASS | **✅ PASS**（`pricingKey` 两向均可覆盖，但**实际可达点只有 1 个**，见 §7.6） |
| **优化闭环判定** | 未闭环 ❌ | **未闭环 ❌**（阻断项由 F1 换成标准 2） |

**一句话**：F1 确已修复、收益确已恢复，但修复方式把 `pricing` 键的分辨力降到 1 bit，
使第 1 轮被判"不可触发"的那条残留缝隙变为可触发 ⇒ **标准 2 FAIL ⇒ 仍未闭环**。

---

## 7.1 本轮改动范围（git · 只读）

```
$ git status --porcelain
 M packages/compaction/compaction-basic/src/region.ts      ← T1 遗留（本轮不核）
 M packages/llm/token-meter/src/index.ts                   ← T2 本轮
?? CONTEXT.md  ?? MANIFEST.md  ?? SOURCE_ANALYSIS.md  ?? optimization/

$ git diff --stat
 packages/compaction/compaction-basic/src/region.ts | 105 +++++++++++++++------
 packages/llm/token-meter/src/index.ts              |  74 ++++++++++++++-
 2 files changed, 148 insertions(+), 31 deletions(-)

$ git diff --stat -- packages/llm/token-meter/src/index.ts
 packages/llm/token-meter/src/index.ts | 74 ++++++++++++++++++++++++++++++++++-
 1 file changed, 73 insertions(+), 1 deletion(-)
```

- T2 本轮 = **`index.ts` 单文件，+73 / -1**（第 1 轮为 +54 / -1；增量 19 行 = `pricingKey` helper + JSDoc + 键项改名）。
- **`region.ts` 的 105 行是 T1 遗留未提交改动，不计入本轮**：`git diff -U0 -- region.ts | grep "^+" | grep -i "measure\|memo\|token-meter\|pricing"` ⇒ **无任何匹配**，与 T2 无耦合 ✅。
- 唯一被删行：`-    return deepFreeze(structuredClone({` ⇒ 非导出、非签名 ✅。

⇒ **标准 3 的"仅限本文件"对 T2 成立**（见 §7.4 补：`^+import` 计数 = 0）。

---

## 7.2 标准 1 — F1 修复核验：memo 现在真能命中

**改动**：`pricingKey(pricing)`（`index.ts:126-128`，`pricing === undefined ? undefined : JSON.stringify(pricing)`）；
键项 `MeasurementMemo.pricing` → `pricingKey`（`:88`）；命中判定 `:212` 由引用比较改为值比较。

**F1 是否消除：✅ 是**

| # | 事实 | 位置 |
|---|---|---|
| 1 | 生产 adapter 每次调用返回新对象（两条出口均为新字面量） | `llm-deepseek/src/request-pricing.ts:97`、`:99-110` |
| 2 | 另一条生产级路径同样每次新建 | `test-support/llm-replay/src/index.ts:886` |
| 3 | 测试用 adapter 同样每次新建 | `compaction-basic/tests/compaction-basic.spec.ts:2181-2188` |
| 4 | 第 2 轮不再比较引用，而比较 `JSON.stringify` 结果 ⇒ 上述"每次新建"不再阻断命中 | `index.ts:126-128` / `:212` / `:258` |

**生产命中链（逐项）**：`region.ts:386`（`prepareCompaction`）→ `:456`（`assertWholeSurfaceUnchanged`）/ `:484`（`assertSelectedSpanStable`）

- 二者之间**无 `session.append`** 时 `revision` 不变 ✅（`:456` 的语义就是"表面未变"；表面若变则必有事件 ⇒ revision 变 ⇒ miss，保守方向正确）
- 同一 `LlmRuntime` 实例 ⇒ `memo.llm === llm` ✅
- 同一 provider/model ⇒ `pricingKey` 同为 `"{}"` ✅
- header 相同 ⇒ `optionalHeaderEquals` 为 `true` ✅
- ⇒ **五项全等 ⇒ 命中**，跳过 `priceSurface`（`:217`）、`estimateToolsTokens`（`:228`/`:242`）、`structuredClone`+`deepFreeze`（`:247-254`）

**`memo.result` 确为已冻结对象**：唯一写入点 `:255-261` 的 `result` 直接来自 `:247` `deepFreeze(structuredClone({...}))` 的返回值 ✅。

⇒ **标准 1：✅ PASS**（第 1 轮"生产命中率 ≈ 0"的结论已作废，与 `implementation.md` §6.1 一致）。

---

## 7.3 【争议点裁定】标准 2 — `pricing` 键项只有 1 bit 分辨力

### 7.3.1 实现者上报的事实核验：✅ 全部属实

| # | 证据 | 结论 |
|---|---|---|
| 1 | `packages/llm/llm/src/types.ts:212-220`：`LlmImageRequestPricing` 的**唯一成员**是 `priceImages(images)`，且为**方法签名**（无数据属性） | 接口确无数据成员 |
| 2 | 全仓实现的 `priceImages` 共 7 处，全部是**仅含 `priceImages` 的对象字面量**：`llm-deepseek/src/request-pricing.ts:97`、`:100`；`test-support/llm-replay/src/index.ts:886`；`compaction-basic.spec.ts:2183`；`route-pricing.spec.ts:35/174/209/248`；`llm/tests/topology.spec.ts:321` | 无一带数据属性 |
| 3 | `JSON.stringify` 跳过函数值属性 ⇒ `JSON.stringify({ priceImages: fn })` === `"{}"` | 值域塌缩 |
| 4 | ⇒ `pricingKey` 的取值域 = `{ undefined, "{}" }` | **1 bit 分辨力：只能区分"有无定价"** |

⇒ **实现者上报完全属实**：两个不同的定价对象（如 `route-pricing.spec.ts` 的 `fixedPricing` 与 `substitution`）**同键为 `"{}"`**。

### 7.3.2 兜底两项确实拦住了"路由身份"

- `headerEquals` → `callConfigEquals`（`llm/src/call-config.ts:49-58`）逐字段比较 `provider` / `model` / `reasoningEffort` / `temperature` / `maxTokens` / `stop`，外加 tools 逐项 `sameSchema`（`session/src/request-header.ts:43-52`）。而 pricing 正是由 `header.config.provider/model` 解析出来的（`index.ts:265-270`）⇒ **换 provider/model 必 miss** ✅
- `llm` 实例引用入键 ⇒ **换 `LlmRuntime` 实例必 miss** ✅
- 例证：`route-pricing.spec.ts:176` 的两个不同定价对象之所以仍走 miss，**是因为它们的 model 不同**（`vision` vs `text-only`，`callConfigEquals` 拦住），**不是因为 `pricingKey`**。这一点很关键：把该用例改成"同一 model 下两个不同定价"，当前实现**必然返回陈旧结果**——这就是分辨力缺失的直接可测证据。

### 7.3.3 但键仍不覆盖三类真实输入 ⇒ 判定 FAIL

结果字段 `nodes[].tokens`、`surfaceTokens`、`baseline`、`surfaceDeltaTokens`、`totalTokens` 全部经由
`priceSurface(state.surface, pricing, fileText)`（`:217`）→ `pricing.priceImages(...)` 计算。
**pricing 的"行为"是影响结果的直接输入，而键只保留它的"有无"（1 bit）。**

在 **同 llm 实例 + 同 provider/model + 同 revision** 的前提下，以下三类输入变化**均不会被键感知**：

| 缺口 | 证据链 | 后果 |
|---|---|---|
| **G-A** 同 provider 换 adapter | `LlmRuntime.registerAdapter`（`llm/src/index.ts:396`）与 `replace`（`:464` 先 `delete` 再 `set`）在**同一 LlmRuntime 实例**内替换 `adapters`；`imageRequestPricing`（`:676-677`）每次按 provider 现查 ⇒ 同 model 返回不同定价 | 键不变 ⇒ 复用旧定价结果 |
| **G-B** connection 快照变更 | `llm-deepseek/src/adapter.ts:40-42` 每次调用 `this.dependencies.options()`；`request-pricing.ts:95` `connection.models.find(...)`，其 `inputModalities`（`:96`）/`imagePixelBudget`（`:57`）/`imageMaxBytes`（`:40`）决定 `visualTokens` | catalog 热更新/重连后同一 model 的定价行为改变，键不变 ⇒ 复用旧 `visualTokens` |
| **G-C** `imageAccess` 解析变化 | `adapter.ts:22-25` 的 resolver 读 `dependencies.resolveAttachments?.()` / `resolveImageAccess`；`request-pricing.ts:102/107` 用它生成 handle / 占位文本 | 解析路径变 ⇒ 文本长度变 ⇒ `text` 价格变，键不变 ⇒ 复用旧文本价格 |

**最强的一条反驳依据（来自被核代码自身）**：
`llm-deepseek/src/request-pricing.ts:83-84` 的 JSDoc 明确写着
*"Access paths resolve at pricing time, so a path that changes before the request only shifts the text price by its own length."*
⇒ **"定价结果随时间变化"是该接口明示的语义**，而键里没有任何时间 / 版本 / 快照维度去捕捉它。

### 7.3.4 为何第 1 轮"不计 FAIL"，第 2 轮必须计 FAIL

第 1 轮报告 §2 原文（本文 `:111` 行）：
> "唯一理论缝隙（**已文档化，不计 FAIL**）… **且在生产因 F1（引用每次都变）实际不可能触发**。"

即：第 1 轮把它判为不计 FAIL 的**唯一理由是 F1 使 memo 永不命中**。
**F1 现已修复 ⇒ memo 在生产真的会命中（§7.2）⇒ 该缝隙从"不可触发"升格为"可触发"。**
用第 1 轮自己的判据，它现在**必须计入**。

### 7.3.5 触发窗口真实存在，且下游兜底会同时失效

- 收益路径 `region.ts:386 → :456/:484` 之间夹着 `await dependencies.summarize(...)`（`region.ts:420`）这一 **await 点**；重连、配置热更新、adapter 重注册均可在此窗口内发生，且**不需要落账任何 session 事件**（否则 revision 会变 ⇒ miss ⇒ 反而安全）。
- 一旦命中，`region.ts:457` 的 `isDeepStrictEqual(current.nodes, prepared.measurement.nodes)` 退化为**同一引用比较**（两者是同一个冻结对象）⇒ 该稳定性校验**空过**，不再能兜住任何差异。

### 7.3.6 裁定

**标准 2：❌ FAIL（严格判定，不含糊）**

- 不是"回归"，也不是"已证明会出错"：现有兜底使它在**绝大多数**真实变更下仍 miss-safe（provider/model 变、llm 实例变、任何事件落账）。
- 但按验收标准的字面要求——"**键覆盖所有影响结果的输入；输入变化即失效**"——`pricing` 这一项**只覆盖 1 bit**，G-A/G-B/G-C 三处"输入变化而不失效"明确存在 ⇒ **不成立**。
- 概率低（需同 revision + 同 llm 实例 + adapter/connection/access 变更三者叠加）、后果有界（一次错误的 shrink 决策 / 陈旧的 route 价格），但**性质是"键覆盖不完备"，不是"文档化的前提"**——因为第 1 轮判它为"前提"的那条支撑理由已被本轮修复移除。

---

## 7.4 标准 3 — 仅改 `index.ts`、无新依赖、风格一致 —— **✅ PASS**

- 范围：`git diff --stat -- packages/llm/token-meter/src/index.ts` = **1 file，+73 / -1** ✅（`region.ts` 为 T1 遗留，已排除）。
- **零新增 import**：`git diff -U0 -- index.ts | grep -c "^+import"` = **0** ✅；用到的 `LlmImageRequestPricing` / `LlmRuntime` / `EpochHeader` / `SessionLogOffsetType` / `TokenMeasurement` 全为既有导入。
- **无新依赖**：无 import、无 package.json 改动 ✅。
- **风格**：`MeasurementMemo`（`index.ts:82-93`）沿用 `readonly` 字段 + 逐字段 JSDoc，与既有 `ReplayState`（`:62-68`）同构；`private readonly measurements = new WeakMap<Session, MeasurementMemo>()`（`:153`）与上一行 `states`（`:151`）对称；新增代码无 `!` / `as` / `any` / `??` ✅（`pricingKey` 用显式三元而非 `?.`）。

---

## 7.5 标准 4 — 导出 / 签名 / 调用方语义不变 —— **✅ PASS**

- **导出面零改动**：diff 中唯一 `-` 行为 `return deepFreeze(structuredClone({`；`export class TokenMeter`、3 处 `export type *`、`export default TokenMeter`、`estimateMessage` 均无 `^-` ✅。
- **签名不变**：`measure(session: Session, requestHeader?: EpochHeader): TokenMeasurement` ✅。
- **返回值等价**：未命中路径字段与顺序逐字未动（`:248-253`）；命中路径返回同一冻结对象，**唯一可观察差异是对象身份**，`TokenMeasurement` 全字段 `readonly` 且已 `deepFreeze` ⇒ 共享只读安全 ✅。
- **调用方零改动**：9 处生产调用点（`compaction-basic/src/index.ts:278/297/325/339/396`、`region.ts:386/456/484`、`acp/src/updates.ts:99` 等）均为只读用法 ✅。
- **抛错语义不变**：见裁定 A ✅。

---

## 7.6 标准 5 — before/after 与收益结论 —— **✅ PASS**（附 F5 表述修正）

`optimization/T2/implementation.md` 已具备：

- **Before/After**：§3.1（结构对照）、§3.2（行为表）、§3.3（plan 第 8 节三条走查）✅
- **第 1 轮"零收益"结论已作废**：§6.1 明确重写，给出命中条件与"生产压缩路径必然命中"的推导 ✅
- **键覆盖性**：§4 逐字段表 + 比较方式依据表 + 失效条件穷举 9 条 ✅
- **`pricingKey` 分辨力缺失已上报**：§4「`pricingKey()` 的实际分辨力」+ §7 上报项 ✅（文档诚实度好，问题在判定而非隐瞒）
- **未做运行时验证声明**：§6 末 ✅

**F5（低·文档表述错误，需修）**：`implementation.md` §5 称 `pricingKey` 的 `JSON.stringify` 分支由
`route-pricing.spec.ts:136/157/176/213/238/248/278` 覆盖 —— **不准确**。
因命中判定是短路 `&&` 链（`:209-213`），`pricingKey(pricing)` 仅在
`memo !== undefined` ∧ `revision` 相等 ∧ `llm` 相等 时才被求值：

| 调用点 | 是否真的求值 `pricingKey` | 原因 |
|---|---|---|
| `:128` `:141` `:196` `:220` `:260` `:289` | 否（首次 measure） | `memo === undefined` 短路 |
| `:160`→`:165`、`:220`→`:224`、`:260`→`:262` | 否 | 中间有 append ⇒ `revision` 变 ⇒ 在 revision 处短路 |
| `:196`→`:199` | **是** | 无 append、同 llm；但 `harness(() => undefined)` ⇒ pricing `undefined` ⇒ **true 路** |
| `:181`→`:188` | **是** | 无 append、同 llm、`harness(model => ...)` ⇒ pricing = `fixedPricing` ⇒ **false 路** |

⇒ false 路的**实际可达点只有 `route-pricing.spec.ts:188` 一处**，不是 §5 列的 7 处。
（结论不受影响——一处即可满足 branch 100%——但依据必须改正，否则门禁一旦变红无从排查。）

---

## 7.7 裁定 A — 查询在 `_sync()` 之后 / 写在 `deepFreeze` 成功之后 —— **✅ PASS**

- **查询晚于 `_sync`**：`:202` `const state = this._sync(session)` 是 `measure()` 的第一条语句（其前仅 `:200-201` 两行注释）；memo 查询在 `:208-215` ⇒ 严格晚于 ✅。
  ⇒ `token-meter.spec.ts` 的 `expectRepeatedFailure()`（被 7 个用例复用）不受影响：malformed 事件在 `_sync`→`_foldEvent` 内抛错，且抛错前 `consumedEvents` 不推进 ⇒ **不命中也不写缓存，每次都抛** ✅。
- **写晚于 `deepFreeze` 成功**：`measurements.set`（`:255-261`）位于 `const result = deepFreeze(structuredClone({...}))`（`:247-254`）**成功返回之后**、`return result`（`:262`）之前 ✅。`_sync` 抛错（`:202`）与 `priceSurface` 抛错（`:217`，对应 `route-pricing.spec.ts:242`）均不达写点 ⇒ **失败不缓存** ✅。
- `canonicalHeader(requestHeader)`（`:203-205`）亦在查询之前 ⇒ override 非法输入仍每次抛错 ✅。

## 7.8 裁定 B — 无 `v8 ignore`；命中分支与 `pricingKey` 两向均有既有测试覆盖 —— **✅ PASS（含单点脆弱提示）**

- **无抑制注释**：`grep -rn "v8 ignore|c8 ignore|istanbul ignore" packages/llm/token-meter/` ⇒ **NONE**（src 与 tests 均无）✅。
- **命中分支（true 路）**：`token-meter.spec.ts:634` —— 前置 `:616` 的 listener 内 measure 已在 revision=3 写入 memo；`:633 expect(revisions).toEqual([3])` 证明其后无 append、无 `request/header`；该 ctx 未挂 `llm`（`llm === undefined`、pricing `undefined`）⇒ **五项全等必命中** ✅。
  第 2 轮起另有一条更强的证据：挂载 llm 的 `compaction-basic` route-priced 路径（`compaction-basic.spec.ts:2196` 注册 `PricedContextAdapter`）下 `region.ts:386 → :456` 也可命中。
- **`pricingKey` 两向**：
  - **true 路（`pricing === undefined`）**：`token-meter.spec.ts:634`、`route-pricing.spec.ts:196/199` ✅
  - **false 路（`JSON.stringify`）**：`route-pricing.spec.ts:188` ✅（推导见 §7.6 表）
  - ⇒ **两向均可被既有 spec 覆盖，per-file 100% branch 门禁无新增不可达分支，因此不需要（也不得使用）`v8 ignore`** ✅
- **⚠️ 单点脆弱提示（不判 FAIL，供主Agent 知悉）**：false 路依赖**唯一一处** `:188`；若将来该用例被改写（例如 :181/:188 之间插入一次 append，或改用不同 model），`pricingKey` 的 false 路即失覆盖、门禁变红。建议在 `implementation.md` §5 改正的同时，把这一条依赖显式记下来。
- **新增 statement**：`measurements` 初始化（`:153`）、`const llm`（`:206`）、`const memo`（`:208`）、`measurements.set`（`:255`）均在每次 `measure()`/构造时执行 ✅。`interface MeasurementMemo` 为纯类型 ✅。

---

## 7.9 发现汇总（第 2 轮）

| # | 发现 | 等级 | 说明 |
|---|---|---|---|
| **F1'** | F1 已消除（正面确认） | — | `pricingKey` 值比较修复了"生产永不命中"，标准 1 由 FAIL 转 PASS |
| **F4** | `pricing` 键项只有 1 bit 分辨力 ⇒ G-A/G-B/G-C 三处"输入变化不失效" | **高（阻断闭环）** | 见 §7.3。第 1 轮判它"不计 FAIL"的理由（F1 使其不可触发）已被本轮修复移除 |
| **F5** | `implementation.md` §5 的 `pricingKey` false 路覆盖依据不准确（实际只有 `route-pricing.spec.ts:188` 一处） | 低 | 见 §7.6；结论不受影响，依据须改正 |
| **F6** | 命中时 `region.ts:457` 的 `isDeepStrictEqual` 退化为同引用比较 ⇒ 稳定性校验空过 | 中（F4 的下游放大） | 与第 1 轮 §标准4 记录的"语义仍一致"不同：第 1 轮该判断以"命中 ⇔ surface 未变"为前提，而 F4 说明存在"surface 未变但定价已变"的情形，此时校验不再等价 |

---

## 7.10 优化闭环判定（第 2 轮）

**判定：未闭环 ❌**

阻断项由"收益为零（F1）"换成了"键覆盖不完备（F4）"。
正确性在**绝大多数**真实路径上仍成立，收益也确实恢复了；但验收标准 2 的
"键覆盖所有影响结果的输入"在当前实现下**明确不成立**，且该缺口在第 2 轮被**激活**。

### 补救选项清单（供主Agent 裁定；本 Agent 不改码）

| 选项 | 做法 | 闭合效果 | 范围代价 | 备注 |
|---|---|---|---|---|
| **R1（推荐·彻底）** | 给 `LlmImageRequestPricing` 增一个稳定判别键（如 `readonly key: string` 或 `revision(): string`），由各 adapter 把"影响定价行为的全部输入"（connection 快照 + `imageAccess` 身份 + provider/model）编码进去；`pricingKey()` 改为读该键 | 闭合 **G-A + G-B + G-C** | **跨包、超出单文件约束**：`llm/src/types.ts`（接口）+ `llm-deepseek/src/request-pricing.ts` + `adapter.ts` + `test-support/llm-replay/src/index.ts` + 测试 double（`route-pricing.spec.ts`、`compaction-basic.spec.ts:2181`） | 需主Agent/用户批准放宽范围；也是唯一能真正满足标准 2 的方案 |
| **R2（折中·跨文件但不动接口）** | 在 `LlmRuntime` 侧按 `(provider, model)` 单槽记忆 `imageRequestPricing`，并在既有 `llm/adapters-updated` 事件（`llm/src/index.ts:360`）时失效；`index.ts` 侧恢复 `pricing === memo.pricing` 引用比较 | 闭合 **G-A**；G-B/G-C 仍需 adapter 侧在 connection/access 变更时失效，否则仍开 | 改 `llm/src/index.ts`（+1 处），`index.ts` 微调 | 若主Agent 认可"connection/access 在同 revision 内不变"可作为文档前提，R2 是一个可接受的落点 |
| **R3（保守·单文件内）** | 把命中条件收窄为"仅当 `pricing === undefined` 时命中"（只在无路由定价场景缓存） | 闭合全部缝隙 | 单文件内可完成 | 代价：回到第 1 轮的"生产零收益"，**等于放弃 T2 立项目标**。若主Agent 判定"收益不足以换取任何风险"，可选此项并同步重写 §6.1 |
| **R4（接受风险·不改码）** | 维持现状，把 §4 的"前提"升格为**显式验收豁免**；建议补一条测试固定该前提（同 model、同 revision 换定价 ⇒ 允许复用），防止后人误以为键有完整分辨力 | 不闭合，转为受控豁免 | 无 | 需主Agent 明确签字接受；测试应写明"这是已知限制，不是期望行为" |
| **R5（不推荐）** | 用 `pricing.priceImages.toString()` 作判别键 | 仅能区分 text-only/vision 两支与不同 adapter，**不覆盖 G-B/G-C**；且把函数源码当契约，脆弱 | 单文件 | 列出仅为穷举，不建议采纳 |

### 若采纳 R1/R2 后的复核要求

- 重新核 `plan.md` 第 6 节 / `implementation.md` §5 的门禁自检（新增分支须两向均有既有测试可达；否则**上报主Agent，不得自行加 `v8 ignore`**）。
- 修正 F5（`implementation.md` §5 的覆盖依据）。
- 修正 F6 相关表述（第 1 轮 §标准4 关于 `region.ts:457` 的"语义仍一致"结论需按新前提重写）。
- 建议同步修订 `plan.md` 3.1「各项比较方式」表中 `pricing` 一行的 `===`（实现者已在 `implementation.md` §7 提出，未越权改 `plan.md`）。

### 未做运行时验证（必须补跑的套件）

本环境无 `node_modules`、无网络，无法 `pnpm test` / `tsc`。应跑：
- `packages/llm/token-meter/tests/*`（`token-meter.spec.ts`、`route-pricing.spec.ts`、`token-usage-projection.spec.ts`、`context-breakdown-projection.spec.ts`、`turn-usage.spec.ts`）
- `packages/compaction/compaction-basic/tests/*`、`packages/compaction/command-compact/tests/*`
- 覆盖率门禁：`vitest.config.ts`（`coverage.include = packages/*/*/src/**`，`statements/branches/functions = 100` per-file）+ `scripts/coverage-uncovered-locations.cjs`
- 建议补一条**反证测试**（临时、不提交）：同 model、同 revision、换一个不同 `priceImages` ⇒ 当前实现会返回陈旧结果。该测试用于确认 F4 真实存在，不建议作为长期用例保留（除非采纳 R4）。

---

## 7.11 附：第 2 轮核验使用的命令（只读）

```bash
git status --porcelain
git diff --stat
git diff --stat -- packages/llm/token-meter/src/index.ts
git diff -- packages/llm/token-meter/src/index.ts
git diff -U0 -- packages/llm/token-meter/src/index.ts | grep -c "^+import"
git diff -U0 -- packages/llm/token-meter/src/index.ts | grep "^-" | grep -v "^---"
git diff -U0 -- packages/compaction/compaction-basic/src/region.ts | grep "^+" | grep -i "measure\|memo\|token-meter\|pricing"
grep -rn "v8 ignore\|c8 ignore\|istanbul ignore" packages/llm/token-meter/
grep -rn "priceImages" packages/ --include=*.ts
grep -n "measure(" packages/llm/token-meter/tests/route-pricing.spec.ts
grep -n "priceImages" packages/test-support/llm-replay/src/index.ts
grep -n "statements\|branches\|include" vitest.config.ts
```

---
---

# 第 3 轮终验（R2 根源修复）

> 流程 4.4（验证者）产出。**只验证，未改任何源码、未 commit。**
> 配套：`plan-r3.md` / `steps-r3.md` / `implementation.md` §第 3 轮（R3-1…R3-10）。
> 上文 §0–§7.11（第 1/2 轮）**全部保留**，本节为追加。
> **未做运行时验证**：本环境无 `node_modules`、无网络，无法 `pnpm test` / `tsc` / 覆盖率脚本；
> 全部结论为静态/结构核对 + 源码逐行走查。行号 = 改后快照（2026-10-02）。

## 8.0 结论速览（第 3 轮）

**判定：闭环达成 ✅（附 3 条非阻断的文档/依据修正要求）**

| 维度 | 第 2 轮 | 第 3 轮 |
|---|---|---|
| 标准 1（真命中） | ✅ | ✅ **保留**（生产双调用在同世系内确实拿到同一引用） |
| 标准 2（无错误复用） | ❌ FAIL（1 bit 分辨力） | ✅ **PASS**（身份 ⇔ `(provider, model, 世系)`；G-A 闭合，G-B/G-C 按 C3 折准） |
| 阻断项 | F4（键覆盖不完备） | 无。F4 随 `pricingKey()` 删除而消失 |

三个核心缺口的状态：**G-A 闭合（且有新断言固定）**、**G-B 未闭合但已写入 JSDoc 显式前提**、
**G-C 半闭合（未命中时实时；命中窗口内复用，已文档化）**。

## 8.1 范围核验（git · 只读）

```
$ git diff --stat -- packages/llm/
 packages/llm/llm/src/index.ts           | 46 ++++++++++++++++++++++++-
 packages/llm/llm/tests/topology.spec.ts | 16 +++++++++
 packages/llm/token-meter/src/index.ts   | 60 ++++++++++++++++++++++++++++++++-
 3 files changed, 120 insertions(+), 2 deletions(-)
```

| 检查 | 结果 |
|---|---|
| `packages/llm/` 改动文件数 | **恰好 3** ✅（2 个 src + C2 批准的 1 个既有 spec） |
| `git status --porcelain` 全仓 | 4 项（`region.ts` + 上列 3 项 + 未跟踪文档）✅ |
| `region.ts`（T1 遗留）`--stat` | **仍为 105（+75/-30）**，逐字节未动 ✅ |
| `git diff -U0 -- packages/llm/ \| grep -c "^+import"` | **0** ✅ |
| `git diff -U0 -- .../region.ts \| grep "^+" \| grep -i "measure\|memo\|token-meter\|pricing"` | **空** ✅ |
| 新增 `v8 ignore` / `c8 ignore` / `istanbul ignore` | **0 处**（全 `packages/llm/` 内仅命中与本次改动无关的既有行）✅ |
| 被删行是否含 `export` / 签名行 | 否 ✅ |

⇒ 标准 3 的"范围"部分成立；详情见 §8.3-3。

## 8.2 四个核心问题裁定

### Q1 — 身份稳定性：`region.ts:386 → 456/484` 同 revision 双调用是否拿到同一引用？

**裁定：✅ 是**（在"同 `LlmRuntime`、无拓扑事件、无跨路由插入"三个条件下；这三个条件在生产压缩路径上均成立）。

逐环推演（本轮独立重做，未采信实现者结论）：

| 环节 | 证据 | 结论 |
|---|---|---|
| ① 386/456/484 的 `header` 同源 | 三处均为 `dependencies.meter.measure(session)`（无 `requestHeader`）⇒ `header = state.header`（最新已折叠封包），值恒定 | 同 `(provider, model)` ✅ |
| ② 中间有无 `session.append` | `compactSurfaceRegion`：`region.ts:230` append `compaction/start` → `:242 prepareCompaction`（386）→ `:243 await summarizeCompaction` → `:420 await dependencies.summarize` → `:253 assertStable`（456/484）；append 只在 `:491 commitCompactionBody`（其后） | **revision 不变** ✅ |
| ③ `summarize` 是否写 session | `summarizer.ts:120-181`：`summarizeWithLlm` 只做 `ctx.llm.stream(options)` + 组装；全文无 `session.append`；`packages/llm/llm/src/` 内 `grep "\.append("` **零命中** ⇒ `stream()` 不写日志 | **不 bump revision** ✅ |
| ④ `summarize` 是否打乱单槽 | 全仓 `grep imageRequestPricing` 的 src 消费点只有 `token-meter/src/index.ts:255` 与 `spill-policy/src/index.ts:65`；**LLM 请求发送路径（`stream`/`prepareCall`/`forAdapter`）不调用它** ⇒ 即使 summarization 走另一条路由（`summarizer.ts:128-137` 的 `summarizationProvider`），也不会写槽 | **槽未被覆盖** ✅ |
| ⑤ 命中结果 | 386：槽空 ⇒ miss ⇒ 委派 ⇒ 钉住 `P`；456/484：槽 = `{provider, model, P}` ⇒ **HIT ⇒ 同一引用 `P`** ⇒ token-meter `memo.pricing === pricing` 为真（revision/llm/header 亦同）⇒ `return memo.result`，跳过 `priceSurface` + `estimateToolsTokens` + `structuredClone` + `deepFreeze` | **HIT** ✅ |

**唯一的插入窗口**（`await` 期间）经穷举只有三类，且**全部安全**：

| 插入源 | 后果 |
|---|---|
| `emitAdaptersUpdated()`（如 `host.ts:41-48` 的 `retryPolicy` 变更 → `registration.replace()`） | 槽清空 ⇒ 456 重新委派 ⇒ 新对象 ⇒ **miss**（保守方向，且正是期望行为） |
| 另一 session 的 `measure()`（`acp/src/updates.ts:99`）/ 另一 agent 的 tool 结果绑定（`spill-policy/src/index.ts:65`）且模型不同 | 槽被覆盖 ⇒ 456 miss（**只丢收益，不丢正确性**） |
| 同 session 的 `spill-policy` / 同 session 的 `acp` 更新 | 键相同（都取 `session.requestHeader().config`）⇒ **槽不变 ⇒ 仍 HIT** |

⇒ **Q1 = PASS**。附带提示（不判 FAIL，见 F9）：多会话并发下命中率可能退化为 0，但正确性不受影响，与第 2 轮 R-2 一致。

### Q2 — 失效完备性：`emitAdaptersUpdated()` 首行清空是否覆盖全部变化来源？

**裁定：✅ 覆盖「adapter 身份 / 注册表」这一维度的全部来源；未覆盖的只有 adapter 内部状态（G-B/G-C），已文档化。**

独立核实（不采信 §1.1）：

| 核实项 | 命令/证据 | 结果 |
|---|---|---|
| `this.adapters` 写入点 | `grep -n "this\.adapters" llm/src/index.ts` ⇒ `:433`（disposer `delete`）、`:489`（`commitRoutes` `delete`）、`:492`（`commitRoutes` `set`），其余 3 处均为读 | **恰好 3 处写** |
| 每处写之后是否 emit | `:433 → :435 emit`、`:489/:492 → :495 emit` | **全部紧跟** ✅ |
| 事件派发出口是否唯一 | `grep -rn "llm/adapters-updated" packages/*/*/src/` ⇒ 派发只在 `llm/src/index.ts:385`（`emitAdaptersUpdated` 内）；其余为订阅方（`acp/src/index.ts:148`、`llm/src/invariant.ts:89`、4 处 client `$on`）与声明 | **唯一** ✅ |
| 是否存在"不经 `emitAdaptersUpdated()` 换 adapter"的路径 | `registerAdapter`(:421) → `commitRoutes`；`handle.replace`(:441-448) → `prepareRoutes`+`commitRoutes`；`ctx.effect` disposer(:431-436)；`registerConfigurableProviders` 的 `commit`(:541-544)/disposer(:554-556) —— **四条路径全部经过 emit** | **不存在** ✅ |
| 其他 provider 插件的热更新 | `llm-deepseek/src/host.ts:41-48`（仅 `retryPolicy` 变 ⇒ `replace`）、`llm-pi-ai/src/index.ts:309-318`（`facts` 变 ⇒ `ensureRegistrationFacts` ⇒ `replace`）⇒ 均走 `commitRoutes` ⇒ 均 emit | ✅ |
| **"同理但不同失效路径"** | 唯一例外：外部 `ctx.emit('llm/adapters-updated')`（仅 `llm/tests/invariant.spec.ts:109`，且只断言不抛）。方案 A 下它**不**清槽——但外部 emit **不改变 `adapters`**，定价不可能因此变化 ⇒ **无正确性影响**。这是 A 相对 B 的唯一语义差，已由 C1 批准 | 无害，记为 F10 |

⇒ **Q2 = PASS**：对"adapters 变更"完备；对"adapter 内部 connection 快照 / access 解析"不完备，即 G-B/G-C，见 Q4。

### Q3 — 消费者枚举复核（C3）：实现者的枚举是否完整？

**裁定：✅ 完整，未发现漏项；亦未发现构成 G-B 类新增风险的消费者。**

本人独立复核（**两条独立 grep，非复用实现者命令**）：

```
grep -rn "imageRequestPricing" packages/ --include=*.ts --include=*.tsx   # 排除 /tests/
grep -rn "priceImages"         packages/ --include=*.ts --include=*.tsx   # 排除 /tests/
```

| 类别 | 命中（改后快照） |
|---|---|
| 经 `LlmRuntime.imageRequestPricing` 的**生产消费者** | `token-meter/src/index.ts:255`（`_routeImagePricing`）、`spill-policy/src/index.ts:65` — **仅 2 处** |
| 读 `priceImages` 的**生产站点** | `token-meter/src/route-pricing.ts:47`（`priceSurface`）、`spill-policy/src/index.ts:67` — **仅 2 处** |
| 直连 adapter（绕开 `LlmRuntime`，不受钉住影响） | `llm-deepseek/src/adapter.ts:40`、`test-support/llm-replay/src/index.ts:881`；测试侧 `llm-deepseek/tests/runtime.spec.ts`、`files.spec.ts`、`spill-policy/tests/multimodal.spec.ts`（`vi.spyOn` 完全绕过真实实现） |
| 非代码 | `extensions/tool-cordis/src/api-catalog.ts:1419/5578/5610/5642`（API 目录文档，非调用） |

⇒ 实现者"全仓 `src` 下读 `priceImages` 只有 `route-pricing.ts:47` 与 `spill-policy/src/index.ts:67`"的说法**属实且完整**。

两处消费者的安全性独立复核：

- `token-meter`：`:193` 取 `pricing` → `:198` 比较 → `:203` `priceSurface` 使用，**同一同步块内**，无跨 `await` 持有 ✅
- `spill-policy`：`index.ts:65` 取 → `:67` 立即 `priceImages` → `:69-72` 立即消费进 `costs` Map，**不缓存对象、不跨 revision 复用** ✅（`pricing()` 是 `bound()` 的局部闭包）

⇒ **Q3 = PASS**，C3 的"未发现不安全消费者"结论可采信。

### Q4 — G-A / G-B / G-C 现状

| 缝隙 | 裁定 | 依据（本轮独立核实） |
|---|---|---|
| **G-A** 同 provider 换 adapter | **✅ 闭合** | 见 Q2：`adapters` 3 处写 ⇒ 3 处 emit ⇒ 槽清 ⇒ 新对象 ⇒ `memo.pricing === pricing` 为 false ⇒ miss。**且有断言固定**：新增 `topology.spec.ts:340-354`（见 §8.4-C2） |
| **G-B** 连接快照（`models`/`inputModalities`/`imagePixelBudget`/`imageMaxBytes`）同世系内热更新 | **❌ 未闭合（已按 C3 转显式前提）** | `request-pricing.ts:95` `connection.models.find(...)` 在**解析时**求值并闭包捕获 ⇒ `:104 resolveRequestImageTarget(catalogModel, …)` 冻结快照；`host.ts:41-48` 仅 `retryPolicy` 变化才 `replace` ⇒ 纯 catalog/尺寸变更不发布事件。**前提已写入 JSDoc**：`llm/src/index.ts:698-711`（"**Precondition:**" 段），字段列举与实际一致（`config.ts:75-76`、`request-pricing.ts:57-64` 已逐字核对） |
| **G-C** `imageAccess` 解析变化 | **⚠️ 半闭合** | `adapter.ts:22-25` 的 `imageAccess` 是**解析函数**（每次 `priceImages` 现读 `resolveAttachments?.()` → `host.ts:32` `ctx.get('attachments')`，`host.ts:33-35` → `ctx.get('fs')`）⇒ **未命中时文本价格实时**；**命中时 `priceSurface` 根本不执行 ⇒ 该窗口内复用旧文本价格**。JSDoc 已明确写出"access 解析不在此前提内" |
| **F6** 命中时 `region.ts:457` `isDeepStrictEqual` 退化为同引用比较 | **⚠️ 随 G-B/G-C 残留，但窗口已收窄** | 命中 ⇔ revision 未变 ∧ 路由未变 ∧ 世系未变；此时唯一能不同的就是 G-B/G-C。`implementation.md` R3-9 已按此重写 ✅ |

**G-B/G-C 前提的 JSDoc 落点核查（C3 要求"已确实写入且描述准确"）**：

| JSDoc 位置 | 是否写入前提 | 准确性 |
|---|---|---|
| `llm/src/index.ts:698-711`（`imageRequestPricing`） | ✅ **是**（`**Precondition:**` 整段） | ✅ 准确：字段列举与 `config.ts`/`request-pricing.ts` 逐字一致；"每次 `priceImages` 现读 `attachments`/`fs`" 与 `adapter.ts:22-25` 一致 |
| `token-meter/src/index.ts:70-83`（`MeasurementMemo`） | ⚠️ **否**——只写了"pin + 每次发布丢弃 ⇒ 身份 = 路由 + 世系"（**保证机制**），**未写残留前提** | 不违反正确性，但 plan §3.1 要求"各记一条"未落实 ⇒ 记为 **F7** |
| `token-meter/src/index.ts:171-179`（`measure()`） | ⚠️ **否**——同上 | 同 **F7** |

⇒ **Q4：G-A 闭合 ✅；G-B/G-C 以"显式前提"结案，前提在主 JSDoc 落点已写入且准确 ✅，但 token-meter 侧两处落点缺前提记录（F7，非阻断，需补记）**。

## 8.3 五条验收标准逐条

### 标准 1 — memo 真能避免重复 `structuredClone` + `deepFreeze`（生产路径必须能命中）— **✅ PASS**

- `measure()`（`token-meter/src/index.ts:185-201`）：命中即 `return memo.result`（`:200`），位于 `:233 deepFreeze(structuredClone({…}))` **之前**；`memo.result` 唯一来源是该 `deepFreeze(structuredClone(...))`（`:233-240`），且写入点 `:241` 在其后 ⇒ 返回的一定是已深冻结对象。
- 生产命中链：`region.ts:386` 钉住 ⇒ `456`/`484` 同引用 ⇒ HIT（证据见 Q1 表）。
- 命中时跳过的成本：`priceSurface`(O(S)，含 `priceImages`) + `estimateToolsTokens`（tools stringify）+ `structuredClone`(O(S)) + `deepFreeze`(O(S))。
- 收益量级：一次 pressure 压缩 5 次 `measure` ⇒ 省 1 次全量；每 retry 再省 1 次（与第 2 轮口径一致）。
- **与第 1 轮的区别**：第 1 轮"永不命中（零收益）"的根因（adapter 每次新建对象）已由 runtime 侧钉住消除 ⇒ 本轮生产路径**首次**真正可命中。

### 标准 2 — 无错误复用 — **✅ PASS（在 C3 折准前提下）**

论证（本轮独立重建）：

1. **同身份 ⇒ 同行为**：`pricing` 的行为完全由该对象的闭包决定（`request-pricing.ts:95-110`），同一对象引用必然给出同一价格函数 ⇒ 引用相同 ⇒ 复用安全（这是第 2 轮 `JSON.stringify` 1-bit 方案做不到的）。
2. **异输入 ⇒ 异身份 ⇒ miss**：换 provider / 换 model ⇒ 单槽键不同 ⇒ 重新委派 ⇒ 新对象 ⇒ miss；换世系（`registerAdapter`/`replace`/dispose/directory）⇒ `emitAdaptersUpdated` ⇒ 槽清 ⇒ 新对象 ⇒ miss；换 `LlmRuntime` ⇒ `memo.llm === llm` 拦。
3. **双重兜底**：`(provider, model)` 相同但封包其他字段不同 ⇒ `optionalHeaderEquals` → `headerEquals` → `callConfigEquals`（`packages/core/session/src/request-header.ts:43-52`）拦；任何 append ⇒ `revision` 拦。
4. **残留**：G-B（connection 快照）、G-C（命中窗口内 access 文本）——按 C3 以显式前提结案，见 Q4。触发需"同 revision ∧ 同世系 ∧ 仅 catalog/尺寸或 access 变更且不改 retryPolicy"叠加，后果有界。

⇒ 标准 2 由第 2 轮的 **FAIL** 转为 **PASS**。

### 标准 3 — 仅改约定文件、无新依赖、TS 风格一致 — **✅ PASS**

| 项 | 结果 |
|---|---|
| 文件数 | `packages/llm/` 内 **3**（`llm/src/index.ts`、`token-meter/src/index.ts`、C2 批准的 `llm/tests/topology.spec.ts`）✅ |
| 新依赖 | `git diff -U0 -- packages/llm/ \| grep -c "^+import"` = **0**；`LlmImageRequestPricing` 仍是既有 `import type` ✅ |
| 风格 | `interface ImagePricingMemo` 三个 `readonly` 字段 + 逐字段 JSDoc；`private pricingMemo: ImagePricingMemo \| undefined` 无初始化器（`undefined` 在类型内）；新增代码**无** `!` / `as` / `any` / `??`；保留原有 `?.` ✅ |
| 与既有风格一致 | 私有接口置于 `LlmRuntime` class 之前（与 token-meter 把 `MeasurementMemo` 放在 class 之前的风格一致）；`readonly` 与相邻字段对齐 ✅ |
| 删除项 | 第 2 轮的 `pricingKey()` helper（17 行）已整体删除，`grep pricingKey` = 0 ✅ |
| `region.ts` | 105 行逐字节未动 ✅ |

### 标准 4 — 导出/调用方语义不变 — **✅ PASS**

- `imageRequestPricing(provider: string, model: string): LlmImageRequestPricing | undefined` 签名/返回类型/`@Remote` 契约（`imageRequestPricing` **无** `@Remote`，见 `api-catalog.ts:5642` 的声明行）一字未变。
- `measure(session: Session, requestHeader?: EpochHeader): TokenMeasurement` 未变；9 处生产 `measure()` 调用点（`grep "\.measure(" packages/*/*/src/` 恰好 9 行：`acp/updates.ts:99`、compaction-basic `index.ts:278/297/325/339/396`、`region.ts:386/456/484`）零改动。
- 抛错语义不变：`_sync()` 仍是 `measure()` 首条语句（`:188`），且在 memo 查询（`:194-201`）**之前** ⇒ 畸形事件每次都 fail loud（`token-meter.spec.ts` 的 `expectRepeatedFailure` 系列行为不变）；`measurements.set` 仍在 `deepFreeze` 成功之后（`:241` > `:233`）。
- 三分支（`:208-231`）一字未动；`priceSurface`/`estimate.ts`/`route-pricing.ts` 未动。
- `imageRequestPricing` 的**可见行为变化**仅有一处（有意为之）：同路由同世系返回同一引用，而非每次新建。调用方（`token-meter`、`spill-policy`）均为"取后即时使用"，不依赖"每次新对象"。

### 标准 5 — before/after 说明 + 收益与残留前提 — **✅ PASS**

- `implementation.md` §第 3 轮 R3-3（before/after 代码 + 行为对照表）、R3-8（收益）、R3-9（残留前提 G-A/G-B/G-C/F6）、R3-10（未做运行时验证 + 应跑套件）齐备。
- 第 2 轮遗留的文档项：**F5 随 `pricingKey()` 删除自然失效** ✅；**F6 已按新前提重写**（R3-9）✅；F1' 保留 ✅。
- 修正要求：F7（token-meter 两处 JSDoc 缺前提）、F11（`memo.pricing === pricing` true 侧的覆盖依据不准确）——见 §8.6。

## 8.4 裁定项复核

### C1 — 失效在发布点、未新增订阅函数 — **✅ PASS**

- `this.pricingMemo = undefined` 位于 `private emitAdaptersUpdated()`（`:377`）函数体**首行**（`:380`，前有两行注释）；既有派发逻辑（`:385-410`）一字未动。
- **未新增任何订阅函数**：diff 中无 `ctx.on` / `ctx.effect` 新增；即 `plan-r3.md` §2.4 的**方案 A**。
- 新增覆盖位置：**0 个新分支、0 个新函数**（仅 1 条赋值 statement）✅ 优于方案 B。
- 完备性依据见 Q2（3 处写 ⇒ 3 处 emit；派发出口唯一）。

### C2 — 只改 1 个既有 spec 补 1 条断言、未新建 spec、未加 `v8 ignore` — **✅ PASS**

- 文件：`packages/llm/llm/tests/topology.spec.ts`（既有）；位置：既有 `describe('imageRequestPricing resolution')` 内**唯一** `it` 的末尾（`:340-354`）；新增断言 **1 条**（`:354`）。
- 反证强度独立复核 ✅：`:349` 注册 `ServedAdapter`（槽空）→ `:350` 解析 ⇒ 钉住 `pricing`（旧对象）→ `:352` 换 `served` → `:353 handle.replace(['a'])` ⇒ `commitRoutes` ⇒ emit ⇒ 槽清 → `:354` 重新解析 ⇒ `replacement`。**若无发布点失效，`:354` 会返回旧 `pricing` ⇒ 断言红** ⇒ 这是真正的失效回归护栏（第 2 轮 R-5 所缺的正是这一条）。
- 未新建 spec 文件；未加任何 `v8 ignore` ✅。
- 新增 `ServedAdapter.imageRequestPricing` 被 `:350`、`:354` 各调用一次 ⇒ 其 function 覆盖可达 ✅。
- 既有 `:330-338` 序列在单槽下仍全部成立（本轮独立推演：`330` 空槽写 / `331` model 变 / `333` provider 变 / `336` 未注册 / `337 dispose` ⇒ emit 清槽 / `338` 空槽 ⇒ `undefined`），与实现者 R3-6.3 一致 ✅。

## 8.5 CI per-file 100% 门禁自检（逐新增分支/操作数）

门禁事实已复核：`vitest.config.ts:216-217` `coverage.include = ['packages/*/*/src/**/*.{ts,tsx}']`，`:372-378` `thresholds: { perFile: true, statements/branches/functions/lines = 100 }`，`packages/llm/llm/src/index.ts` 与 `packages/llm/token-meter/src/index.ts` 均不在 exclude 名单。

**文件① `packages/llm/llm/src/index.ts`**

| 新增位置 | 类型 | 覆盖来源（本轮逐条核实） | 单点 |
|---|---|---|---|
| `interface ImagePricingMemo`（`:349`） | 纯类型 | 无运行时位置 ✅ | — |
| `private pricingMemo`（`:370`） | statement | 任何 `new LlmRuntime()` / `ctx.plugin(LlmRuntime)`（数百处）✅ | 否 |
| `this.pricingMemo = undefined`（`:380`） | statement | `emitAdaptersUpdated()` 被 `topology.spec.ts:327/328/337/349/353`、`service.spec.ts` 数百处 register/dispose/replace 触发 ✅ | 否 |
| `const memo = this.pricingMemo`（`:717`） | statement | 每次 `imageRequestPricing` 调用 ✅ | 否 |
| `if (...)` **true** | branch | `route-pricing.spec.ts:165`、`:224`、`:262`；`llm-deepseek/tests/adapter.spec.ts:290`、`:299` ✅ | 否（≥2 处独立来源） |
| `if (...)` **false** | branch | `topology.spec.ts:330/331/333/336/338` ✅ | 否 |
| `memo !== undefined` 两侧 | 操作数 | 假：`:330`（空槽）；真：`:331` ✅ | 否 |
| `memo.provider === provider` 两侧 | 操作数 | 真：`route-pricing.spec.ts:165`、`:224`；**假：`topology.spec.ts:333`** | ⚠️ 假侧单点 |
| `memo.model === model` 两侧 | 操作数 | 真：`route-pricing.spec.ts:165`、`:224`；**假：`topology.spec.ts:331`** | ⚠️ 假侧单点 |
| `adapters.get(...)?.adapter` 两侧 | 操作数 | 真：`:330`；假：`:336`/`:338` ✅ | 否 |
| `this.pricingMemo = { … }`（`:720`） | statement | 全部 miss 路径 ✅ | 否 |

> 前提核实：`route-pricing.spec.ts` 的 `harness()`（`:77-90`）用的是**真实 `LlmRuntime`**（`new LlmRuntime(ctx)` + `registerAdapter(['mock'], new PricingAdapter(pricing))`），
> 不是 mock ⇒ 其命中序列确实会执行 `LlmRuntime.imageRequestPricing`。这一点是本轮重点复核项（若 harness 是假 llm，则 `memo.model === model` 的真侧将无覆盖、门禁必红）。

**文件② `packages/llm/token-meter/src/index.ts`**

| 变化位置 | 类型 | 覆盖来源（本轮逐条核实） | 单点 |
|---|---|---|---|
| 删除 `pricingKey()` | −1 function / −2 branch | 位置消失（连带消除第 2 轮 F5 那处脆弱依据）✅ | — |
| `memo.pricing === pricing`（`:198`）**true** | branch | **`route-pricing.spec.ts:196→199`**（两侧 `undefined`：`:196` 槽写 `{mock, vision, undefined}`，`:199` provider 变 `unregistered` ⇒ miss ⇒ `undefined`；两测之间无 append ⇒ revision 相同 ⇒ 该操作数**必被求值**） | ⚠️ **单点**（见 F11） |
| `memo.pricing === pricing` **false** | branch | **`route-pricing.spec.ts:181→188`**（`substitution` vs `fixedPricing`；无 append、llm 相同 ⇒ 必被求值） | ⚠️ **单点** |

### 单点依赖清单（门禁若变红，先查这 5 处）

1. **`packages/llm/llm/tests/topology.spec.ts:331`** —— 唯一提供 `memo.model === model` 的 **false** 侧（`('a','other')` 紧随 `('a','vision')`）。若被改写为同 model 或插入 `replace`，即失覆盖。
2. **`packages/llm/llm/tests/topology.spec.ts:333`** —— 唯一提供 `memo.provider === provider` 的 **false** 侧（`('plain','vision')` 紧随型号变更）。
3. **`packages/llm/token-meter/tests/route-pricing.spec.ts:181→188`** —— 唯一提供 `memo.pricing === pricing` 的 **false** 侧（`:181` 与 `:188` 之间**不得**插入 append、不得改为同 model，否则该操作数不被求值）。
4. **`packages/llm/token-meter/tests/route-pricing.spec.ts:196→199`** —— 实际唯一提供 `memo.pricing === pricing` 的 **true** 侧（见 F11：实现者另外列出的 `token-meter.spec.ts:634` **不成立**）。
5. **`packages/llm/token-meter/tests/route-pricing.spec.ts:165 / :224`** —— 与 `llm-deepseek/tests/adapter.spec.ts:290` 共同提供 `LlmRuntime` 侧 `if` 的 **true** 侧；若 token-meter 测试被移出同一覆盖率汇总（`coveragePartitionMode` 变化），则 `memo.model`/`memo.provider` 的**真**侧只剩 `adapter.spec.ts:290`，需重新评估。

⇒ **不需要也不得使用 `v8 ignore`**（与第 2 轮一致）；本轮实际 0 处新增 ignore ✅。

## 8.6 发现汇总（第 3 轮）

| # | 发现 | 等级 | 说明 |
|---|---|---|---|
| **F7** | `implementation.md` R3-4/C3 称"在 `MeasurementMemo`（`:70-83`）与 `measure()`（`:171-179`）的 JSDoc 中各记一条"前提 —— 实际这两处只写了**保证机制**（pin + 发布丢弃 ⇒ 身份 = 路由 + 世系），**未写 G-B/G-C 的残留前提** | 中（文档） | plan §3.1 要求"各记一条"未落实。不阻断闭环（主落点 `llm/src/index.ts:698-711` 已准确写入），但须补记，否则 token-meter 的读者会误以为键有完整分辨力。**不改码，交由实现者补注释** |
| **F8** | JSDoc 中"they are only republished when the route's retry policy changes"仅对 `llm-deepseek`（`host.ts:41-48`）成立；`llm-pi-ai`（`:309-318`）在任何注册 facts 变化时即 `replace` ⇒ 对 pi-ai 而言 catalog 变更**会**失效 | 低（文档精度） | 表述是"保守下界"（低估失效频率），**不影响正确性**；建议改为"至少在 deepseek 插件下…"或点名插件 |
| **F9** | 单槽在跨会话并发下的 thrash：本轮定位到两个具体插入源 `acp/src/updates.ts:99` 与 `spill-policy/src/index.ts:65`（不同 session/不同 model 时），在 `region.ts:420` 的 await 窗口内可覆盖槽 ⇒ 命中退化为 miss | 低（仅丢收益） | 与 plan §4.2 R-2 一致；正确性不受影响。若日后要提升命中率，可考虑按 `(provider, model)` 的小容量 Map（但会引入淘汰分支，与 per-file 100% 分支门禁冲突） |
| **F10** | 外部 `ctx.emit('llm/adapters-updated')`（`invariant.spec.ts:109`）在方案 A 下不清槽 | 信息 | 外部 emit 不改变 `adapters` ⇒ 定价不可能变 ⇒ 无正确性影响；`invariant.spec.ts:109` 仅断言不抛，仍通过。A 相对 B 的唯一语义差，已由 C1 批准 |
| **F11** | `implementation.md` R3-7 把 `token-meter.spec.ts:634` 列为 `memo.pricing === pricing` **true** 侧的覆盖依据 —— **不成立**：该处 `:630` 的 `measure` 与 `:634` 的 `measure` 之间有一次 `session.append`（revision 2→3）⇒ `:196 memo.revision === state.consumedEvents` 为假 ⇒ `&&` 短路，`:198` 的 `pricing` 操作数**根本不被求值** | 低（文档依据） | 结论不受影响：true 侧由 `route-pricing.spec.ts:196→199` 独立提供 ✅。但依据须改正，否则单点清单会记错（正确单点见 §8.5 清单第 4 项） |

## 8.7 优化闭环判定（第 3 轮）

**判定：闭环达成 ✅**

- 第 2 轮的唯一阻断项 **F4（键只有 1 bit 分辨力）已消除**：`pricingKey()` 删除，`pricing` 恢复为对象引用比较，分辨力 = `(provider, model, 注册世系)`；引用相同 ⇒ 行为相同（闭包决定），引用不同 ⇒ miss。
- 第 1 轮的根因 **F1（生产永不命中）保持消除**：runtime 侧单槽钉住使 `region.ts:386 → 456/484` 真正 HIT。
- 标准 1–5 全部 PASS；C1/C2 全部 PASS。
- 残留 G-B/G-C **已按主Agent C3 折准以"显式前提 + JSDoc"结案**，且主落点描述准确（Q4 已逐字核对）。

**闭环后仍需主Agent 知悉/处理的 3 项（均非阻断）**：

1. **F7**：补记 token-meter 两处 JSDoc 的残留前提（否则 §8.5 的"键有完整分辨力"会被误读）。
2. **F11**：修正 `implementation.md` R3-7 的 true 侧覆盖依据（改为 `route-pricing.spec.ts:196→199` 单点）。
3. **F8**：JSDoc 中"仅 retryPolicy 变化才重新发布"改点名插件，避免以偏概全。

**若主Agent 认为 F7 必须在本次一并闭合**（即不接受"token-meter 读者看不到残留前提"），则本轮应判 **未闭环**，差距清单为：
① `token-meter/src/index.ts:70-83` 与 `:171-179` 各补一句残留前提；
② 建议同时把"命中时 `priceImages` 不执行 ⇒ G-C 窗口"写进 `measure()` JSDoc（当前只在 `implementation.md` R3-9）；
③ 上述两项落定后重跑 §8.8 的套件 + 覆盖率门禁。

## 8.8 未做运行时验证（必须补跑）

本环境无 `node_modules`、无网络，无法 `pnpm test` / `tsc` / 覆盖率脚本 —— **本轮全部结论为静态核对 + 源码走查**。应跑：

- `packages/llm/llm/tests/*`（尤其 `topology.spec.ts:318-356`、`service.spec.ts`）
- `packages/llm/token-meter/tests/*`（`token-meter.spec.ts`、`route-pricing.spec.ts`、`token-usage-projection.spec.ts`、`context-breakdown-projection.spec.ts`、`turn-usage.spec.ts`）
- `packages/llm/llm-deepseek/tests/*`（尤其 `adapter.spec.ts:277-300` 的 R-3 高风险用例、`dynamic-config.spec.ts`）
- `packages/spill/spill-policy/tests/*`（`multimodal.spec.ts`、`multimodal-recovery.spec.ts`）
- `packages/test-support/llm-replay/tests/*`（`llm-replay.spec.ts:2166-2199`）
- `packages/compaction/compaction-basic/tests/*`、`packages/compaction/command-compact/tests/*`
- `packages/acp/acp/tests/*`（`updates.ts:99` 的 measure 路径）
- 覆盖率门禁：`vitest.config.ts`（`coverage.include`、`perFile: true`、statements/branches/functions/lines = 100）+ `scripts/coverage-uncovered-locations.cjs`

**R-3 高风险用例的独立结论**（`llm-deepseek/tests/adapter.spec.ts:277-300`）：三次 `price()` 同为 `('deepseek-official','deepseek-flash')` ⇒ `:287` miss 钉住、`:290`/`:299` 命中同一对象；`adapter.ts:22-25` 的 `imageAccess` 在每次 `priceImages` 时现读 `ctx.get('attachments')`/`ctx.get('fs')`，故 `:290`（`fs` 未挂 ⇒ `not.toContain('/mounted/image.png')`）与 `:299`（`fs` = `MappedFiles` ⇒ `toContain('/mounted/image.png')`）**均仍成立**；分支选择只取决于 `catalogModel`，该序列中 catalog 未变 ⇒ **该用例不会因钉住而变红**（静态推演，仍建议实跑确认）。

## 8.9 附：第 3 轮核验使用的命令（只读）

```bash
git status --porcelain
git diff --stat -- packages/llm/
git diff --stat -- packages/compaction/compaction-basic/src/region.ts
git diff -U0 -- packages/llm/ | grep -c "^+import"
grep -rn "v8 ignore\|c8 ignore\|istanbul ignore" packages/llm/
grep -rn "imageRequestPricing\|priceImages" packages/ --include=*.ts --include=*.tsx
grep -n "this\.adapters" packages/llm/llm/src/index.ts
grep -n "emitAdaptersUpdated" packages/llm/llm/src/index.ts
grep -rn "llm/adapters-updated" packages/*/*/src/
grep -rn "registerAdapter(" packages/*/*/src/
grep -rn "\.measure(" packages/*/*/src/
grep -rn "\.append(" packages/llm/llm/src/
sed -n '370,500p' packages/compaction/compaction-basic/src/region.ts
sed -n '110,181p' packages/compaction/compaction-basic/src/summarizer.ts
sed -n '1,60p'   packages/llm/llm-deepseek/src/adapter.ts
sed -n '53,111p' packages/llm/llm-deepseek/src/request-pricing.ts
sed -n '1,50p'   packages/llm/llm-deepseek/src/host.ts
sed -n '295,320p' packages/llm/llm-pi-ai/src/index.ts
sed -n '43,52p'  packages/core/session/src/request-header.ts
sed -n '77,90p'  packages/llm/token-meter/tests/route-pricing.spec.ts
sed -n '185,250p' packages/llm/token-meter/src/index.ts
sed -n '295,356p'  packages/llm/llm/tests/topology.spec.ts
sed -n '277,300p'  packages/llm/llm-deepseek/tests/adapter.spec.ts
sed -n '200,235p;355,385p' vitest.config.ts
```

---
---

# 第 4 轮最终确认（F7 / F8 / F11 收口）

> 流程 4.4（验证者）产出。**只验证，未改任何源码；本文件为指定交付物（追加）**。
> 上文 §0–§8.9（第 1/2/3 轮）**全部保留**，本节为追加。行号 = 第 4 轮改后快照（2026-10-02）。
> **未做运行时验证**：本环境无 `node_modules`、无网络，无法 `pnpm test` / `tsc` / oxlint / 覆盖率脚本；
> 全部结论为静态/结构核对 + 源码逐行走查。

## 9.0 结论速览（第 4 轮）

| 项 | 判定 |
|---|---|
| **F7**（token-meter 两处 JSDoc 显式写出 G-B / G-C 残留前提） | **✅ PASS** |
| **F8**（点名 `llm-deepseek` / `llm-pi-ai` 行为差异） | **✅ PASS**（附 1 条信息级残留：主落点未点名） |
| **F11**（true 侧依据改 `route-pricing.spec.ts:196→199`；false 侧 `181→188` 收紧） | **✅ PASS** |
| 无源码逻辑改动 | **✅ 确认**（剥离注释后 diff = 第 3 轮已知 6 组，第 4 轮 0 行代码） |
| 第 3 轮 PASS 项回归 | **✅ 全部仍成立**（标准 1–5、C1、C2） |
| **最终优化闭环判定** | **闭环达成 ✅** |

**最终判定：闭环达成 ✅**（阻断项 = 无）。

## 9.1 范围与改动量核验（git · 只读）

```
$ git diff --numstat
75  30  packages/compaction/compaction-basic/src/region.ts
45   1  packages/llm/llm/src/index.ts
16   0  packages/llm/llm/tests/topology.spec.ts
92   1  packages/llm/token-meter/src/index.ts
   ⇒ 4 files changed, 228 insertions(+), 32 deletions(-)
```

| 文件 | 第 3 轮快照 | 第 4 轮 | 结论 |
|---|---|---|---|
| `region.ts`（T1 遗留） | 105（+75/-30） | **105（+75/-30）** | 逐量一致，未被触碰 ✅ |
| `llm/src/index.ts` | 46（+45/-1） | **46（+45/-1）** | 一致 ✅ |
| `llm/tests/topology.spec.ts` | 16（+16/-0） | **16（+16/-0）** | 一致 ✅ |
| `token-meter/src/index.ts` | 60（+59/-1） | **93（+92/-1）** | 增量 **33 行**，见 §9.5 判定为纯注释 ✅ |

**mtime 交叉佐证（独立于 `--stat`）**：`token-meter/src/index.ts` = 20:17、`optimization/T2/implementation.md` = 20:25（第 4 轮）；
`llm/src/index.ts` = 19:51、`topology.spec.ts` = 19:52（第 3 轮）、`region.ts` = 12:20（T1/T2）
⇒ 第 4 轮**只**触碰 token-meter 与本文档 ✅。

## 9.2 F7 — 残留前提是否已显式写出且措辞准确

被核落点：`token-meter/src/index.ts:70-99`（`MeasurementMemo`）与 `:170-217`（`measure()`）。

| 要求 | memo 侧（`:84-98`） | `measure()` 侧（`:197-212`） | 判定 |
|---|---|---|---|
| **G-B**：钉住对象冻结 connection 快照、仅重新发布才失效、命中期间不刷新 | ✅ *"A pinned object captures the connection snapshot it was resolved from — the `models` catalog with each entry's `inputModalities`, `imagePixelBudget`, and `imageMaxBytes` — and that snapshot does not refresh while the pin lives: only republishing the adapter drops it…"* | ✅ *"…so that snapshot does not refresh over the whole window in which the route keeps hitting: only republishing the adapter renews it, and only that republishing misses here."* | **PASS** |
| **G-C**：`imageAccess`/`attachments`/`fs` 每次 `priceImages()` 现读 ⇒ 未命中实时、命中窗口内复用 | ✅ *"Attachment and filesystem access resolution is **not** part of this precondition: it is read from the live `attachments` and `fs` services on every `priceImages` call, so a miss prices image text live, while a hit skips that call and reuses the text captured with `result` — which may lag behind an access change made inside the hit window."* | ✅ *"`priceImages` reads the current `attachments` and `fs` services on every call, so a miss prices image text live, while a hit never reaches `priceSurface` and reuses the text priced into `result`; an access change inside the hit window is therefore reflected only on the next miss."* | **PASS** |
| 与主落点 `llm/src/index.ts:704-711` 措辞对齐 | ✅ 共用三处关键措辞：`"connection snapshot it was resolved from"` / `"the \`models\` catalog with each entry's \`inputModalities\`, \`imagePixelBudget\`, and \`imageMaxBytes\`"` / `"Attachment and filesystem access resolution is **not** part of this precondition"` | ✅ 同 | **PASS** |

**准确性独立复核（不采信实现者结论）**

| 断言 | 本轮独立证据 | 结论 |
|---|---|---|
| 钉住对象在"钉"存活期内快照不刷新，只有重新发布才丢钉 | `llm/src/index.ts:717-721`：命中即 `return memo.pricing`；`:380` `this.pricingMemo = undefined` 仅在 `emitAdaptersUpdated()` 首行 | 属实 |
| deepseek 侧"纯 catalog/尺寸更新不丢钉" | `grep -rn "registration.replace" packages/llm/llm-deepseek/src/` ⇒ **仅 `host.ts:46`**，且在 `host.ts:45` 的 `deepEqualJson(policy, registeredPolicy)` 早退之后 ⇒ 只有 retryPolicy 变才 `replace` | 属实（这正是 G-B 残留风险的来源，文档已点明） |
| access 解析每次现读 | `host.ts:32` `resolveAttachments: () => ctx.get('attachments')`、`:33-35` `resolveImageAccess` 内 `ctx.get('fs')`；`adapter.ts:22-25` 为**解析函数**非缓存值 | 属实 |
| 命中时 `priceImages` 不执行 | `index.ts:228-234` 的 `return memo.result` 位于 `:236 priceSurface(...)` **之前** | 属实 |

**夸大/隐瞒检查**：

- 未夸大 —— 两处均明说键**不**覆盖 catalog/尺寸与 access（"The key deliberately covers nothing below route granularity"、"cannot see a change that stays inside one pinned object"），未声称键有完整分辨力。
- 未隐瞒 —— G-B（快照冻结、可能陈旧）与 G-C（命中窗口内文本滞后）两条风险及其后果均写明。

⇒ **F7 ✅ PASS**。

**观察（信息级，不判 FAIL）**：memo 侧 "is **not** part of **this precondition**" 中的 "this precondition" 指 (G-B) 的钉住前提（与主落点 `:709` 同一措辞）。脱离主落点单独阅读该段时存在轻微指代歧义，但同句随后**完整披露**了 G-C 风险本身 ⇒ 不影响读者获知风险，无需返工。

## 9.3 F8 — 插件差异是否已点名

| 落点 | 原文要点 | 判定 |
|---|---|---|
| `index.ts:89-92`（memo） | *"…how eagerly a plugin does so is its own choice — `llm-deepseek` replaces only when the route's retry policy changes (`host.ts:41-48`), while `llm-pi-ai` replaces on any change to its registration facts (`index.ts:309-318`) — so under `llm-deepseek` a pure catalog or budget update leaves this memo valid."* | ✅ |
| `index.ts:203-207`（`measure()`） | *"Plugins differ in how eagerly they republish: `llm-deepseek` replaces on a retry-policy change alone (`llm-deepseek/src/host.ts:41-48`), whereas `llm-pi-ai` replaces whenever its registration facts change (`llm-pi-ai/src/index.ts:309-318`) — the "only on retry policy" bound therefore describes `llm-deepseek` routes, **not every provider**."* | ✅ |

**事实复核（本轮独立 grep / 读码）**

- `grep -rn "registration.replace"` ⇒ `llm-deepseek/src/host.ts:46`（**唯一**，位于 `host.ts:41-48` 的 `loader/volatile-update` 处理器内，`host.ts:45` 在 retryPolicy 未变时早退）✅
- `llm-pi-ai/src/index.ts:304` 的 `registration.replace(routes)` 位于 `ensureRegistrationFacts()`（`:285-307`）内，`:287` `if (deepEqualJson(facts, registeredFacts)) return` ⇒ **facts 任一变化即 replace**；该函数由 `:310-316` 的 `loader/volatile-update` 触发 ⇒ 引用 `309-318` 指向触发点，属实 ✅

⇒ 读者不会再把"仅 retryPolicy 变化才失效"误当成对全部 provider 成立的前提。**F8 ✅ PASS**。

**残留（信息级 P3，交主Agent 裁定）**：**主落点** `llm/src/index.ts:707-708` 仍是无条件表述 *"they are only republished when the route's retry policy changes"*，未点名插件。第 4 轮按范围约束未触碰该文件（`implementation.md` R4-2 已显式声明）。是否要求一并点名，由主Agent 裁定；**不阻断**本轮闭环（token-meter 侧已收口，且该表述是"保守下界"，不构成正确性风险）。

## 9.4 F11 — 覆盖依据是否已改正

| 侧 | `implementation.md` 现表述 | 本轮独立复核 | 判定 |
|---|---|---|---|
| **true** | `:528` = **`route-pricing.spec.ts:196→199`** | `harness(:192)` = `() => undefined` ⇒ `:196` `pricing = undefined` 并写 memo；`:199` 换 `provider:'unregistered'`（未注册）⇒ `LlmRuntime.imageRequestPricing` 返回 `undefined` ⇒ 两侧同 `undefined` ⇒ **true**；两测之间无 `append`、无拓扑事件 ⇒ `revision`/`llm` 相同 ⇒ `:231` 的 `memo.pricing === pricing` 操作数**必被求值**（`optionalHeaderEquals` 在其后才求值） | ✅ |
| **false** | `:529` = **`route-pricing.spec.ts:181→188`**，且已收紧为"`:181` 是该 session 首次 measure（memo 尚无 ⇒ 短路），真正求值点是 `:188`" | `harness(:176)` = `model => (model === 'vision' ? fixedPricing : substitution)`；`:181` `measure(session, header('text-only'))` ⇒ `substitution`；`:188` `measure(session)` ⇒ `state.header = header('vision')` ⇒ `fixedPricing`；两测之间无 append ⇒ revision/llm 相同 ⇒ 操作数必被求值，两对象引用不同 ⇒ **false** | ✅ |
| 旧依据处置 | `:540-541` 已注明 `token-meter.spec.ts:634` **不成立**并"已于第 4 轮从依据清单中移除（不改用例本身）" | 与第 3 轮 §8.6-F11 的裁定一致（`:626`→`:634` 之间有 append，revision 变 ⇒ 短路） | ✅ |
| 单点清单同步 | `:536-539` 第 2/3 项已分别记 `181→188`（false）与 `196→199`（true） | 与 §8.5 清单第 3/4 项一致 | ✅ |

补充核实：`route-pricing.spec.ts:77-90` 的 `harness()` 用 **真实 `LlmRuntime`**（`new LlmRuntime(ctx)` + `registerAdapter(['mock'], new PricingAdapter(pricing))`），且每个 `it` 新建 `ctx` ⇒ 单槽互不干扰，上述两条求值链不受跨用例污染 ✅。

⇒ **F11 ✅ PASS**。

## 9.5 无源码逻辑改动 — 独立复核（剥离注释比对）

**方法**：自写剥离器（状态机，保留 `'…'`/`"…"`/`` `…` `` 字面量与 `${}` 内的代码，仅剔除 `//` 与 `/* */`），对
`git show HEAD:packages/llm/token-meter/src/index.ts` 与当前文件**各自**剥离后，去空行、去缩进后逐行 diff。
（脚本用完即删，未留在工作区。）

**剥离后 diff（= 纯代码差异）**：

```
+interface MeasurementMemo { readonly revision / header / pricing / llm / result }   ← 纯类型
+private readonly measurements = new WeakMap<Session, MeasurementMemo>()
+const llm = this.ctx.get('llm')
+const memo = this.measurements.get(session)
+if (memo !== undefined && memo.revision === … && memo.llm === llm
+    && memo.pricing === pricing && optionalHeaderEquals(memo.header, header)) { return memo.result }
-return deepFreeze(structuredClone({        →  +const result = deepFreeze(structuredClone({
+this.measurements.set(session, { revision, header, pricing, llm, result })
+return result
```

⇒ 差异**恰好**是第 3 轮已知的 6 组新增，**不含第 4 轮任何痕迹** ⇒ 第 4 轮的 33 行增量**全部位于注释内** ✅。

**交叉佐证**

| 检查 | 命令 / 方法 | 结果 |
|---|---|---|
| 新增 `import` | `git diff -U0 -- packages/llm/ \| grep -c "^+import"` | **0** ✅ |
| 新增 `v8/c8/istanbul ignore` | `git diff \| grep "^+" \| grep -c "v8 ignore\|c8 ignore\|istanbul ignore"` | **0** ✅（`packages/llm/` 内 11 处 `v8 ignore` 全为既有行，均不在 diff 的 `+` 侧） |
| 新增行分类 | token-meter 92 行 `+` 中仅 **27** 行为代码（= 第 3 轮那 6 组），65 行为注释 | 92 − 59（第 3 轮）= **33**，与"本轮 33 行注释"自洽 ✅ |
| 注释定界符配平 | 当前 `/*` = 23 / `*/` = 23（HEAD 16 / 16） | 无未闭合块注释 ✅（新增 7 个块 = memo JSDoc 1 + 字段注释 5 + `measurements` 字段 1） |
| 行宽 | 新增行最长 **98** 字符（`measurements` 字段注释，单行 JSDoc）；文件既有最长 137；`.oxlintrc.json` 无 `max-len` 规则 | 无 lint 风险 ✅ |

> 注（信息级）：`implementation.md` R4-4 自核称"新增注释最长 80 字符"，实为 **98**（一行单行 JSDoc）。
> 属自核数字笔误，**非代码/文档实质差距**（无 max-len 规则、既有文件更长）。

## 9.6 回归 — 第 3 轮 PASS 项是否仍成立

第 4 轮为注释性改动且剥离后 diff 证明**零代码改动** ⇒ 下述结论继承；对可静态复核者本轮重新抽样确认。

| 项 | 判定 | 本轮依据 |
|---|---|---|
| 标准 1（生产可命中） | ✅ 仍成立 | 命中链代码未变：`region.ts:386` 钉住 ⇒ `:456/:484` 同引用 ⇒ `index.ts:231` `memo.pricing === pricing` 为真 ⇒ `:233 return memo.result`，早于 `:236 priceSurface` 与 `:266 deepFreeze(structuredClone(...))` |
| 标准 2（无错误复用） | ✅ 仍成立 | 键语义未变；换 provider/model ⇒ 单槽 miss；换世系 ⇒ `emitAdaptersUpdated` 清槽 ⇒ miss；任何 append ⇒ revision ⇒ miss。残留 G-B/G-C 已按 C3 以显式前提结案（§9.2） |
| 标准 3（范围/无新依赖/风格） | ✅ 仍成立 | 4 文件、数值与第 3 轮一致（§9.1）；`^+import`=0；新增注释沿用 `* ` 块注释与 `readonly` 字段注释风格 |
| 标准 4（签名/调用方不变） | ✅ 仍成立 | 剥离后 diff 中唯一 `-` 行为 `return deepFreeze(`（非导出、非签名）；`measure(session, requestHeader?)` 未变；9 处生产调用点未改 |
| 标准 5（文档齐备） | ✅ 仍成立（并因本轮增强） | `implementation.md` §第 4 轮 R4-1…R4-4 齐备；F7/F8/F11 三处落点均已落文档 |
| 裁定 C1（失效在发布点、无新订阅） | ✅ 仍成立 | `llm/src/index.ts` 未被触碰（46 行一致、mtime 19:51）；`this.pricingMemo = undefined` 仍在 `emitAdaptersUpdated()` 首行；无新增 `ctx.on`/`ctx.effect` |
| 裁定 C2（单 spec 单断言、无 ignore） | ✅ 仍成立 | `topology.spec.ts` 未被触碰（16 行一致、mtime 19:52）；新增断言仍为 `:354` 一条；本轮新增 ignore = 0 |

## 9.7 残余差距清单

**阻断项：无。**

| # | 残余 | 等级 | 处置建议 |
|---|---|---|---|
| R4-a | 主落点 `llm/src/index.ts:707-708` 的"仅 retryPolicy 变化才重新发布"未点名插件（F8 的主落点侧） | 信息（P3） | 由主Agent 裁定是否放宽范围一并点名；不改也不影响正确性（保守下界表述） |
| R4-b | `implementation.md` R4-4 自核行宽数字（80 vs 实际 98） | 信息（P3） | 可选：改数字；无 lint 影响 |
| R4-c | 继承的前轮非阻断项：**F9**（跨会话并发下单槽 thrash，只丢收益）、**F10**（外部 `ctx.emit('llm/adapters-updated')` 在方案 A 下不清槽，无害）、**G-B/G-C 显式前提** | 信息 | 已在 `implementation.md`/JSDoc 记录；F9 若要提升命中率需新增分支，与 per-file 100% 门禁冲突，须另立项 |

## 9.8 未做运行时验证 & 若要量产须跑的套件

本环境无 `node_modules`、无网络 ⇒ **未跑 `tsc` / vitest / oxlint / 覆盖率脚本**；本节结论全部为静态核对。
第 4 轮为**注释性改动、不引入新运行时位置**，故不新增需补跑的套件；§8.8 的清单仍然适用，量产前建议：

1. **类型与静态检查**：`tsc --noEmit`（或仓库对应 build 脚本）+ `oxlint`（确认新增注释不影响 lint）。
2. `packages/llm/token-meter/tests/*`（`token-meter.spec.ts`、`route-pricing.spec.ts`、`token-usage-projection.spec.ts`、`context-breakdown-projection.spec.ts`、`turn-usage.spec.ts`）—— 重点确认 `route-pricing.spec.ts:181→188`（false 侧）与 `:196→199`（true 侧）两个单点未被改动。
3. `packages/llm/llm/tests/*`（尤其 `topology.spec.ts:318-356`、`service.spec.ts`）。
4. `packages/llm/llm-deepseek/tests/*`（尤其 `adapter.spec.ts:277-300`、`dynamic-config.spec.ts`）。
5. `packages/spill/spill-policy/tests/*`、`packages/test-support/llm-replay/tests/*`。
6. `packages/compaction/compaction-basic/tests/*`、`packages/compaction/command-compact/tests/*`、`packages/acp/acp/tests/*`。
7. **覆盖率门禁**：`vitest.config.ts`（`coverage.include`、`perFile: true`、statements/branches/functions/lines = 100）+ `scripts/coverage-uncovered-locations.cjs` ⇒ 确认第 3 轮 §8.5 的 5 处单点仍全部被覆盖、无需 `v8 ignore`。

## 9.9 附：第 4 轮核验使用的命令（只读）

```bash
git status --porcelain
git diff --stat ; git diff --numstat
git diff -- packages/llm/token-meter/src/index.ts
git diff -U0 -- packages/llm/ | grep -c "^+import"
git diff | grep "^+" | grep -c "v8 ignore\|c8 ignore\|istanbul ignore"
grep -rn "v8 ignore\|c8 ignore\|istanbul ignore" packages/llm/
grep -rn "registration.replace" packages/llm/llm-deepseek/src/ packages/llm/llm-pi-ai/src/
sed -n '25,60p'  packages/llm/llm-deepseek/src/host.ts
sed -n '255,325p' packages/llm/llm-pi-ai/src/index.ts
sed -n '60,100p' packages/llm/token-meter/tests/route-pricing.spec.ts
sed -n '150,205p' packages/llm/token-meter/tests/route-pricing.spec.ts
sed -n '688,720p' packages/llm/llm/src/index.ts
sed -n '60,290p'  packages/llm/token-meter/src/index.ts
ls -l --time-style=+%Y-%m-%d_%H:%M <4 个被改文件> optimization/T2/implementation.md
# 剥离注释比对（自写状态机脚本，用完即删）：
#   git show HEAD:packages/llm/token-meter/src/index.ts > head.ts
#   strip(head.ts) vs strip(current) → diff
```

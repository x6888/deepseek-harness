# T2 步骤 — `TokenMeter.measure()` memo 缓存（原子步骤）

> 配套 `plan.md`（同一目录下的方案论证）。目标文件（唯一可改）：
> `sandbox/deepseek-harness/packages/llm/token-meter/src/index.ts`
> 本环境无法 build/run → 每步的"预期产物 / 静态核对"均为**静态或结构核对**，不做运行时验证。
> 每步完成后立即记录，不要攒批。

---

## S0（已完成 · 本轮）读透现状

**动作**：读 `CONTEXT.md`、`SOURCE_ANALYSIS.md`、`index.ts`（`measure()` L146-191、`_sync()` L218-240、`_foldEvent()` L247-329）、`route-pricing.ts`、`estimate.ts`、`surface-fold.ts`、`types.ts`、`tests/token-meter.spec.ts`、`tests/route-pricing.spec.ts`、`vitest.config.ts` 的 coverage 段。

**预期产物**：确认 4 条事实——① `measure()` 每次 `deepFreeze(structuredClone(...))`；② `state` 唯一决定于 `(session, consumedEvents)`；③ `region.ts:386` 与 `456/484` 同 revision 双调用；④ 该文件受 CI per-file 100% 覆盖门禁约束。

**静态核对**：`plan.md` 第 2、3、6 节已固化上述证据与行号。

---

## S1 新增 `MeasurementMemo` 接口

**动作**：在 `index.ts:68`（`ReplayState` 接口结束的 `}`）之后插入接口（字段名/注释照 `plan.md` 3.2）：

```ts
/** One cached `measure()` result, keyed by the exact inputs it was priced from. */
interface MeasurementMemo {
  readonly revision: SessionLogOffsetType
  readonly header: EpochHeader | undefined
  readonly pricing: LlmImageRequestPricing | undefined
  readonly llm: LlmRuntime | undefined
  readonly result: TokenMeasurement
}
```

**预期产物**：文件新增 ~10 行；**零新增 import**（`SessionLogOffsetType` / `EpochHeader` / `LlmImageRequestPricing` / `LlmRuntime` / `TokenMeasurement` 分别在 L17、L14、L11、L11、L29-32 已导入）。

**静态核对**：
- `grep -n "^import" index.ts` 与改动前逐行 diff → **无任何新增 import 行**；
- 接口放在 `ReplayState` 之后、`usageTokens` 之前（与其它模块级 helper 的相对位置一致）；
- 接口是纯类型 → 不产生任何运行时覆盖位置（不影响门禁）。

---

## S2 新增 `measurements` 字段

**动作**：在 `index.ts:108`（`private readonly states = new WeakMap<Session, ReplayState>()`）之后加一行：

```ts
  private readonly measurements = new WeakMap<Session, MeasurementMemo>()
```

**预期产物**：+1 行；声明风格与上一行完全对称（`private readonly` + `WeakMap<Session, …>`）。

**静态核对**：字段位于 `states` 与 `constructor` 之间；是实例字段初始化语句，随任何 `new TokenMeter()` 执行 ⇒ statement 覆盖必达（门禁无风险）。

---

## S3 改写 `measure()`：命中分支 + 写缓存

**动作**（`index.ts:146-191`，除下列外**一字不动**）：
1. L151-152 之间插入 `const llm = this.ctx.get('llm')`（`_routeImagePricing` 之前）；
2. L153 之后插入 `const memo = this.measurements.get(session)` 与命中 `if`（5 个条件，顺序：`memo !== undefined` → `revision ===` → `llm ===` → `pricing ===` → `optionalHeaderEquals`），命中体 `return memo.result`；
3. L183 的 `return deepFreeze(...)` 改为 `const result = deepFreeze(...)`（**保持 `structuredClone` 与 6 个字段原样**）；
4. 在函数末尾 `this.measurements.set(session, { revision, header, pricing, llm, result })` 后 `return result`。

**预期产物**：`measure()` 内 +12 / -2 行左右；三分支（anchor / 空 / 估计，原 L156-181）**零改动**。

**静态核对（逐条打勾）**：
- ☑ `const state = this._sync(session)` 仍是函数**第一条**语句（否则 malformed 事件的"每次都抛"不变式被破坏 → `token-meter.spec.ts:462` 的 `expectRepeatedFailure` 会失效）；
- ☑ `return memo.result` 位于 `structuredClone` 之前（命中即跳过克隆/冻结/定价）；
- ☑ `this.measurements.set(...)` 位于 `deepFreeze(...)` **成功之后**（失败路径不写缓存）；
- ☑ 未新增 `!` / `as` / 非空断言 / `??` / `?.`；
- ☑ 唯一新增分支是那个 `if`：true 路有 `region.ts:386→456/484` 与 `token-meter.spec.ts:338` 覆盖，false 路有"追加事件后 measure"、"换 header 后 measure"两类用例覆盖（`plan.md` 第 6 节表）。

---

## S4 更新 `measure()` 的 JSDoc

**动作**：在现有 JSDoc（L125-145）末尾补一句，记录两条前提（**用文档而不是分支**，与 T1 处理手法一致）：
- 同 revision + 同输入时**复用上一次已冻结的结果**（调用方拿到的是同一个只读对象）；
- 缓存条目随 Session 与 `TokenMeter` 服务一起释放；非 surface 事件也会使缓存失效（保守）。

**预期产物**：+3~5 行注释，无代码变化。

**静态核对**：注释里出现 `logRevision / header / pricing / llm` 四个键项与"共享只读"字样；不改动 `@param` / `@returns`。

---

## S5 静态核对（三条硬性走查）

**动作**：按 `plan.md` 第 8 节清单执行，逐条记录结论。

1. **键覆盖性**：`measure()` 返回值的 6 个字段（`logRevision` / `baseline` / `surfaceDeltaTokens` / `totalTokens` / `surfaceTokens` / `nodes`）逐个回溯自由变量，确认每个都能追溯到 5 个键项之一（Session 引用、revision、header、pricing、llm）。
2. **等价性走查**：命中路径只 `return memo.result`；`memo.result` 唯一来源是 `deepFreeze(structuredClone(...))`；三分支未动；`_sync` 先行 ⇒ 抛错行为不变。
3. **复杂度/调用次数**：复核 `plan.md` 4.2 表——一次 pressure 压缩 5 次调用 → 4 次重算；命中时省掉 `priceSurface`(O(S)) + `estimateToolsTokens`(tools stringify) + `structuredClone`(O(S)) + `deepFreeze`(O(S))。

**预期产物**：结论写进 `optimization/T2/implementation.md`（后续 Agent 产出）。

**静态核对**：三条都给出"字段级"证据，不能只写"应该没问题"。

---

## S6 范围核验（`git diff --stat`）

**动作**：在 `sandbox/deepseek-harness/`（沙箱是完整 git clone）执行：

```bash
cd <repo-root>   # 沙箱克隆所在目录
git status --porcelain
git diff --stat
git diff -- packages/llm/token-meter/src/index.ts
```

**预期产物**：
- `git diff --stat` **有且仅有** `packages/llm/token-meter/src/index.ts` 一行（期望量级 `1 file changed, ~30 insertions(+), ~3 deletions(-)`）；
- `git diff` 全文 grep `^+import` 为空（无新依赖）；
- 改动全部在 `sandbox/deepseek-harness/` 内，**不触碰** `estimate.ts`、`surface-fold.ts`、`route-pricing.ts`、`types.ts`、任何 `tests/`、任何配置文件；
- 导出面不变：`export class TokenMeter`、`measure()` 签名、`export type * from './types.ts'` 等 5 处导出与改动前逐字节一致（用 `git diff | grep '^-'` 确认无导出行被删）。

**静态核对**：若 `--stat` 出现第二个文件，或 diff 中出现 `estimate.ts` 的任何引用变更 → 立即停下并上报主 Agent（违反单文件约束）。

---

## S7 归档与上报（后续 Agent，非本轮）

**动作**：产出 `optimization/T2/implementation.md`（改后关键片段 + 三条走查结论 + `git diff --stat` 原文）与 `optimization/T2/verification.md`（静态核对结果 + **显式标注"未做运行时验证"** + 应跑套件清单：`packages/llm/token-meter/tests/*`、`packages/compaction/compaction-basic/tests/*`、`packages/compaction/command-compact/tests/*`）。

**预期产物**：两个归档文件；`CONTEXT.md` 第 4 节任务状态追加 T2 一行。

**静态核对**：`verification.md` 中"未做运行时验证"字样必须存在；5 条验收标准逐条给出 PASS/FAIL 与依据。

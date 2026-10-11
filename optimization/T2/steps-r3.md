# T2 步骤（第 3 轮重规划 · R2 根源修复）— 原子步骤

> 配套 `plan-r3.md`（同一目录，方案论证与事实证据）。本轮可改文件**仅限 2 个**（均在 `packages/llm/` 内）：
> ① `packages/llm/llm/src/index.ts`（新增）
> ② `packages/llm/token-meter/src/index.ts`（第 2 轮已改，本轮再改）
> **禁止触碰** `packages/compaction/compaction-basic/src/region.ts`（T1 遗留未提交改动）。
> 本环境无法 build/run ⇒ 每步的"预期产物 / 静态核对"均为**静态或结构核对**，不做运行时验证。
> 每步完成后立即记录，不要攒批。发现与 `plan-r3.md` 不符 ⇒ 停下上报主Agent，不要自行发明替代做法。

## 进入本轮前的基线（先记录，用于 S5 范围核验）

```
$ git status --porcelain
 M packages/compaction/compaction-basic/src/region.ts     ← T1 遗留（本轮不碰）
 M packages/llm/token-meter/src/index.ts                  ← T2 第 2 轮
?? CONTEXT.md  ?? MANIFEST.md  ?? SOURCE_ANALYSIS.md  ?? optimization/

$ git diff --stat
 packages/compaction/compaction-basic/src/region.ts | 105 +++++++++++++++------
 packages/llm/token-meter/src/index.ts              |  74 ++++++++++++++-
 2 files changed, 148 insertions(+), 31 deletions(-)
```

**基线不变式（本轮结束时必须仍成立）**：`region.ts` 的 `--stat` 数字仍是 `105`（+75/-30）。

---

## S1 文件① — 新增 `ImagePricingMemo` 接口

**动作**：在 `packages/llm/llm/src/index.ts:336`（`DirectoryRegistrationHandle` 接口结束的 `}`）之后、
`export class LlmRuntime`（`:342`）之前插入接口，字段与注释照 `plan-r3.md` §2.1(a)：

```ts
interface ImagePricingMemo {
  readonly provider: string
  readonly model: string
  readonly pricing: LlmImageRequestPricing | undefined
}
```

**预期产物**：+12~16 行（含 JSDoc）；**零新增 import**（`LlmImageRequestPricing` 已在 `:18` `import type` 块内）。

**静态核对**：
- `grep -n "^import\|LlmImageRequestPricing" packages/llm/llm/src/index.ts` ⇒ `LlmImageRequestPricing` 仍只有 `:18` 一处导入；
- 接口为**纯类型** ⇒ 不产生任何运行时覆盖位置；
- 位置：在 `:336` 之后、`:338` 的 `export class LlmRuntime` 之前。

---

## S2 文件① — 新增 `pricingMemo` 字段

**动作**：在 `:345-349` 的 `discoveries` 字段之后、`:350` 的 `constructor` 之前加一行：

```ts
  private pricingMemo: ImagePricingMemo | undefined
```

**预期产物**：+1 行；无初始化器（`undefined` 在类型内，先例 `llm-deepseek/src/host.ts:22`）。

**静态核对**：
- 字段位于 `discoveries` 与 `constructor` 之间，声明风格与相邻三个 `private` 字段对齐；
- 是类字段定义语句 ⇒ 任何 `new LlmRuntime()` 都执行 ⇒ statement 覆盖必达（门禁无风险）；
- **不要用 `Map`**（会引入淘汰/容量分支，违反 per-file 100% 分支门禁）。

---

## S3 文件① — 失效挂点：清空槽（**裁定 C1 默认方案 A**）

**动作**：在 `private emitAdaptersUpdated(): void {`（`:355`）的函数体**第一行**插入：

```ts
    // Route pricing is pinned per route: a new topology can only mean new pricing.
    this.pricingMemo = undefined
```

**预期产物**：+2 行；既有派发逻辑（`:356-379`）一字不动。

**静态核对（三条硬检查）**：
- ☑ `grep -n "emitAdaptersUpdated()" packages/llm/llm/src/index.ts` ⇒ 1 个定义（`:355`）+ 4 处调用（`:410/:470/:519/:531`）；
- ☑ `grep -n "this.adapters" packages/llm/llm/src/index.ts` ⇒ 写点只有 `:408`（delete，disposer）、
  `:464`/`:467`（delete+set，`commitRoutes`），且 `:408→:410`、`:467→:470` 紧跟 emit
  ⇒ **`adapters` 的任何变更都必然经过本函数**；
- ☑ 新增语句在函数体第一行，且是**赋值语句**（非分支）⇒ 零新增分支、零新增函数。
- 若主Agent 裁定 C1 = 方案 B，则改为在 `constructor`（`:350-352`）内加
  `ctx.on('llm/adapters-updated', () => { this.pricingMemo = undefined })`，并在 S6 记录 +1 函数覆盖位置的依据。

---

## S4 文件① — 改写 `imageRequestPricing`（`:676-678`）

**动作**：把

```ts
  imageRequestPricing(provider: string, model: string): LlmImageRequestPricing | undefined {
    return this.adapters.get(provider)?.adapter.imageRequestPricing(provider, model)
  }
```

改为（**签名 / 返回类型 / 导出面一字不变**）：

```ts
  imageRequestPricing(provider: string, model: string): LlmImageRequestPricing | undefined {
    const memo = this.pricingMemo
    if (memo !== undefined && memo.provider === provider && memo.model === model) return memo.pricing
    const pricing = this.adapters.get(provider)?.adapter.imageRequestPricing(provider, model)
    this.pricingMemo = { provider, model, pricing }
    return pricing
  }
```

**预期产物**：+5 / -1 行（原 `return` 一行拆为四行）。

**静态核对（逐条打勾）**：
- ☑ 无 `@Remote` 装饰、无新增 import、无 `!` / `as` / `any` / `??` / `?.`（保留原有 `?.`）；
- ☑ 三个 `&&` 操作数顺序：`memo !== undefined` → `provider` → `model`；
- ☑ **未命中路径与改动前完全等价**：仍是一次 `adapters.get` + 一次 adapter 委派；
- ☑ 槽**也记录 `pricing === undefined`** 的路由（否则未注册 provider 每次都要重新委派）；
- ☑ 覆盖自检（对照 `plan-r3.md` §5.2 表）：
  true 路 = `route-pricing.spec.ts:160→165` / `:220→224` / `:260→262`、`adapter.spec.ts:287→290→299`；
  false 路 = `topology.spec.ts:330/331/333/336/338`；
  `model` 操作数两侧 = `route-pricing.spec.ts:160→165`（真）/ `topology.spec.ts:331`（假）。
  若任一位置找不到既有路径 ⇒ **停下上报，禁止 `v8 ignore`**。

---

## S5 文件① — 补 `imageRequestPricing` 的 JSDoc（显式前提）

**动作**：在现有 JSDoc（`:667-675`）末尾、`@param provider` 之前追加，记录三条（**用文档而非分支**）：
1. 同一 `(provider, model)` 在同一拓扑世系内返回**同一个对象引用**（供调用方做身份比较 / memo）；
2. 槽随每次 `llm/adapters-updated` 发布丢弃 ⇒ 注册/注销/`replace` 必换引用；
3. **显式前提**：钉住的对象捕获解析当时的**连接快照**（`models` catalog 及 `inputModalities`/`imagePixelBudget`/`imageMaxBytes`）。
   该类字段若在不改变 `retryPolicy` 的情况下热更新，本记忆**不会**自动失效（G-B，未闭合，见 `plan-r3.md` §3）；
   访问路径解析**不**在此前提内（每次 `priceImages` 现读 `attachments`/`fs`）。

**预期产物**：+6~8 行注释；`@param` / `@returns` / 签名行不动。

**静态核对**：注释内出现 `llm/adapters-updated`、`connection`、"`retryPolicy`" 字样；
不出现新的代码分支；与 `plan-r3.md` §3.1 措辞一致。

---

## S6 文件② — token-meter：删除 `pricingKey`、恢复引用比较

**动作**（`packages/llm/token-meter/src/index.ts`，六小步）：
1. **删除** `pricingKey()` 整个 helper：`:112-128`（含其 JSDoc，17 行）；
2. `:87-88` `readonly pricingKey: string | undefined` → `readonly pricing: LlmImageRequestPricing | undefined`，
   字段注释改为"Pinned routed image pricing object `result` was priced under."；
3. 更新 `:70-81` 的 `MeasurementMemo` 接口头 JSDoc：把 "pricing by value（rebuilt on every call）"
   改为"pricing 按**引用**比较；`LlmRuntime` 为每个路由钉住一个对象并在每次 `llm/adapters-updated` 发布时丢弃，
   故引用相等 ⇔ 同一路由 + 同一注册世系"；
4. `:212` `memo.pricingKey === pricingKey(pricing)` → `memo.pricing === pricing`；
5. `:258` `pricingKey: pricingKey(pricing),` → `pricing,`；
6. `:189` `measure()` JSDoc 中 "the routed pricing compared by value" →
   "the routed pricing compared by identity (one pinned object per route, dropped with every `llm/adapters-updated` publication)"。

**预期产物**：净 **约 -12 行**（删 17 行 helper，改 5 处）；三分支（`:220-245`）与 `:202` 的 `_sync` 首句**一字不动**。

**静态核对（逐条打勾）**：
- ☑ `grep -n "pricingKey" packages/llm/token-meter/src/index.ts` ⇒ **0 处**；
- ☑ `grep -n "^import" packages/llm/token-meter/src/index.ts` 与基线逐行 diff ⇒ **无新增 import**
  （`LlmImageRequestPricing` 仍在 `:11` 被 `_routeImagePricing`(`:266`) 与接口使用）；
- ☑ `memo.pricing === pricing` 覆盖两侧（对照 `plan-r3.md` §5.2）：
  true = `token-meter.spec.ts:634`（`llm` 未挂载，两侧 `undefined`）、`route-pricing.spec.ts:196→199`；
  false = `route-pricing.spec.ts:181→188`（`substitution` vs `fixedPricing`）；
  并确认这两处的 `memo !== undefined` ∧ `revision` 相同 ∧ `llm` 相同均成立（否则该操作数不会被求值）；
- ☑ 命中路径仍只 `return memo.result`；`measurements.set` 仍在 `:247` `deepFreeze` 成功之后；
- ☑ 未新增 `!` / `as` / `??` / `?.`。

---

## S7 范围核验（`git diff --stat`，应恰好 2 个本轮文件）

**动作**：在 `sandbox/deepseek-harness/`（完整 git clone）执行：

```bash
git status --porcelain
git diff --stat
git diff --stat -- packages/llm/
git diff --stat -- packages/compaction/compaction-basic/src/region.ts
git diff -- packages/llm/
git diff -U0 -- packages/llm/ | grep -c "^+import"
git diff -U0 -- packages/llm/token-meter/src/index.ts | grep "^-" | grep -v "^---"
git diff -U0 -- packages/compaction/compaction-basic/src/region.ts | grep "^+" | grep -i "measure\|memo\|token-meter\|pricing"
```

**预期产物（本轮结束时的正确形态）**：
- `git diff --stat -- packages/llm/` = **恰好 2 files**：
  `packages/llm/llm/src/index.ts`（新增，量级 +25/-1）与
  `packages/llm/token-meter/src/index.ts`（量级 ~+62/-3，因第 2 轮新增的 `pricingKey` 一并删除）；
- `git diff --stat` = **3 files**（多出的第 3 个是 T1 遗留 `region.ts`）；
- `git diff --stat -- packages/compaction/compaction-basic/src/region.ts` **仍为 105 行**（与基线逐字节一致）；
- `git diff -U0 -- packages/llm/ | grep -c "^+import"` = **0**；
- `git diff -U0 -- .../region.ts | grep "^+" | grep -i "measure|memo|token-meter|pricing"` = **空**（证明 T1 遗留与 T2 无耦合）；
- token-meter 的被删行（`^-`）中**不得**出现 `export`、`measure(session:`、`export type *` 等导出/签名行。

**静态核对**：若 `--stat -- packages/llm/` 出现第 3 个文件，或 `region.ts` 的 105 变化，或 diff 中出现
`estimate.ts` / `surface-fold.ts` / `route-pricing.ts` / `types.ts` / 任何 `tests/` / 任何配置文件的改动
⇒ **立即停下并上报主Agent**（违反范围约束）。

---

## S8 三条硬性走查 + 高风险用例复核（记录到 `implementation-r3.md`）

**动作**：
1. **键覆盖性**：对 `measure()` 返回值 6 个字段（`logRevision`/`baseline`/`surfaceDeltaTokens`/`totalTokens`/`surfaceTokens`/`nodes`）
   逐个回溯自由变量，确认每个都能追溯到 5 个键项之一；特别论证新方案下
   **`pricing` 引用 ⇔ `(provider, model, 注册世系)`**。
2. **等价性 / fail-loud**：命中只 `return memo.result`；`memo.result` 唯一来源 `deepFreeze(structuredClone(...))`；
   三分支未动；`_sync` 仍是首句（`:202`）；`set` 在 `deepFreeze` 之后 ⇒
   `token-meter.spec.ts:462` `expectRepeatedFailure` 的 7 个用例行为不变。
3. **高风险既有用例复核（R-3）**：逐行走 `llm-deepseek/tests/adapter.spec.ts:285-299`——
   三次 `price()` 同为 `('deepseek-official','deepseek-flash')`；确认：
   分支选择只取决于 catalog（挂载 `attachments`/`fs` 不改变分支），
   文本中的挂载路径由 `imageAccess`（`adapter.ts:22-25`）**现读** `ctx.get('fs')` 得出
   ⇒ 钉住后 `:290`（`not.toContain('/mounted/image.png')`）与 `:299`（`toContain('/mounted/image.png')`）**仍成立**。

**预期产物**：三条结论写进 `optimization/T2/implementation-r3.md`（**新文件，不覆盖** `implementation.md`），
每条都要给出字段级/行号级证据，不能只写"应该没问题"。

---

## S9 归档与上报（后续 Agent，非本轮规划范围）

**动作**：
- 产出 `optimization/T2/implementation-r3.md`（改后关键片段 + 三条走查结论 + §5.2 覆盖依赖点清单 + `git diff --stat` 原文）；
- 在 `optimization/T2/verification.md` **追加 §8（第 3 轮复验）**，不覆盖 §7；逐条给出 5 条验收标准 PASS/FAIL、
  裁定项 C1/C1'/C2/C3/C4 的处置结果，并**显式标注"未做运行时验证"** + 应跑套件清单：
  `packages/llm/llm/tests/*`、`packages/llm/token-meter/tests/*`、`packages/llm/llm-deepseek/tests/*`、
  `packages/spill/spill-policy/tests/*`、`packages/compaction/compaction-basic/tests/*`、
  `packages/compaction/command-compact/tests/*`，以及覆盖率门禁（`vitest.config.ts` + `scripts/coverage-uncovered-locations.cjs`）。

**静态核对**：`verification.md` §8 中"未做运行时验证"字样存在；G-A 闭合、G-B/G-C 转显式前提的结论与
`plan-r3.md` §3 一致；F5 因 `pricingKey` 删除而失效、F6 按新前提重写。

# T2 计划（第 3 轮重规划）— R2 根源修复：LlmRuntime 侧按路由单槽记忆定价 + token-meter 恢复引用比较

> 流程 4.2 规划者产出。**只规划，不改代码**。本目录本轮新增产物：`plan-r3.md` + `steps-r3.md`
> （第 1/2 轮的 `plan.md` / `steps.md` / `implementation.md` / `verification.md` **一律不覆盖**）。
> 环境无法 build/run（无 `node_modules`、无网络）⇒ 全部结论为**静态/结构核对**，标注「未做运行时验证」。
> 行号对应 2026-10-02 快照；本轮涉及文件：
> ① `packages/llm/llm/src/index.ts`（`LlmRuntime.imageRequestPricing`，现 `:676-678`）
> ② `packages/llm/token-meter/src/index.ts`（`TokenMeter.measure()`，现 `:199-263`）

---

## 0. 本轮定位（一句话 + 与前三轮的关系）

**一句话方案**：把"定价对象每次新建"这个根因从 `token-meter` 挪回它的源头——在 `LlmRuntime.imageRequestPricing()`
内按 `(provider, model)` **单槽记忆**一次解析结果，并在**拓扑发布的唯一出口** `emitAdaptersUpdated()` 丢弃该槽；
于是同一个"路由世系（registration generation）+ 路由"恒定返回**同一个对象引用**，
`token-meter` 侧即可**删除 `pricingKey()` 变通、恢复干净的 `memo.pricing === pricing` 引用比较**。

| 轮次 | 做法 | 结果 |
|---|---|---|
| 第 1 轮 | `token-meter` 内 `pricing` 引用比较 | ❌ F1：生产 adapter 每次新建对象 ⇒ 永不命中（零收益） |
| 第 2 轮 | `pricingKey()` = `JSON.stringify` 值比较 | ✅ 能命中；❌ 分辨力塌缩为 1 bit（该类型只有方法成员，恒为 `"{}"`）⇒ 标准 2 FAIL，残留 G-A/G-B/G-C |
| **第 3 轮（本轮）** | **根因上移**：runtime 侧钉住对象身份 + 拓扑事件失效；token-meter 恢复引用比较 | 目标：标准 1 ✅（真命中）+ 标准 2 ✅（`pricing` 身份 = 路由 + 世系，全分辨力） |

**为什么这能同时救回两条标准**：引用比较的分辨力取决于"对象身份是否唯一决定定价行为"。
第 1 轮失败是因为身份每次都变（`N 个行为 → N 个身份`，恒 miss）；
第 2 轮失败是因为身份被丢弃只剩 1 bit（`N 个行为 → 1 个键`，恒 hit 即是错）；
本轮把身份重新定义为 **`(provider, model, 世系)` 的双射**：同键 ⇒ 必然同一个钉住的对象 ⇒ 行为必然相同；
异键（换 provider / 换 model / 换世系）⇒ 必然不同对象 ⇒ 必然 miss。**分辨力完整，且生产可命中。**

---

## 1. 关键事实核实（读源码确认，非假设）

### 1.1 `llm/adapters-updated` 事件：**确认真实存在**（✅ 无需替代机制）

| # | 事实 | 证据 |
|---|---|---|
| 1 | 事件名**准确写法** = `'llm/adapters-updated'`，payload-free（签名 `'llm/adapters-updated'(): void`） | `packages/llm/llm/src/types.ts:23`；`packages/extensions/tool-cordis/src/api-catalog.ts:4121-4123` |
| 2 | 由 `LlmRuntime` 自己通过 `this.ctx.events.dispatch('emit', ['llm/adapters-updated'])` 派发 | `llm/src/index.ts:360`（在 `private emitAdaptersUpdated()` 内） |
| 3 | **唯一的派发出口**就是 `emitAdaptersUpdated()`，全文件共 4 处调用：`:410`（adapter 注册 disposer）、`:470`（`commitRoutes`，即 register/replace）、`:519`/`:531`（configurable-provider directory） | `grep -n "emitAdaptersUpdated()"` ⇒ 5 行（1 个定义 + 4 处调用） |
| 4 | `adapters` map 的全部写入点只有 3 处，且**每处之后都紧跟 `emitAdaptersUpdated()`** | `:408` delete（disposer）→ `:410` emit；`:464`/`:467` delete+set（`commitRoutes`）→ `:470` emit |
| 5 | 外部可用 `ctx.on('llm/adapters-updated', …)` 订阅，且能收到 `events.dispatch('emit', …)` 的派发；本包内已有先例 | `llm/src/invariant.ts:89`（`ctx.on(..., { global: true })`）；`llm/tests/topology.spec.ts:33-39` / `:85-89` / `:108-113`（`ctx.on` 后 register/replace/dispose 均触发）；`acp/src/index.ts:148`；`llm-deepseek/tests/dynamic-config.spec.ts:260` |
| 6 | 也可被外部 `ctx.emit('llm/adapters-updated')` 触发 | `llm/tests/invariant.spec.ts:109` |

⇒ **结论：事件存在、拼写确认、订阅可行、且派发出口唯一。** 见 §2.4 的两种失效挂点选型。

### 1.2 `imageRequestPricing` 完整调用链与"每次新建"的位置

```
token-meter/src/index.ts:269   this.ctx.get('llm')?.imageRequestPricing(config.provider, config.model)   ← 生产唯一入口
spill/spill-policy/src/index.ts:65  ctx.get('llm')?.imageRequestPricing(provider, model)                ← 另一消费者
        ↓
llm/src/index.ts:676-678       return this.adapters.get(provider)?.adapter.imageRequestPricing(provider, model)
        ↓ 每次现查 map（无记忆），委派给 adapter
llm-deepseek/src/adapter.ts:40-42   imagePricing(this.dependencies.options(), model, this.imageAccess)
        ↓ images.ts:11 re-export
llm-deepseek/src/request-pricing.ts:90-111  deepSeekImageRequestPricing(...)
        ├─ :97   return { priceImages: images => images.map(textOnlyPrice) }          ← **新对象字面量**
        └─ :99-110 return { priceImages: images => images.map(...) }                  ← **新对象字面量**
```

| # | 事实 | 证据 |
|---|---|---|
| 1 | `LlmRuntime.imageRequestPricing` 定义在 `llm/src/index.ts:676-678`，**函数体只有一行 `return`**，无 `@Remote` 装饰（本地调用面） | 已读源码；`:663` `providerRetryPolicy` 有 `@Remote`？否；`:478` `listProviders` 有 `@Remote` —— 本方法**无** ⇒ 记忆化不影响远端契约 |
| 2 | 生产 adapter 两条出口均返回新字面量 | `llm-deepseek/src/request-pricing.ts:97`、`:99` |
| 3 | 测试/回放 adapter 同样每次新建 | `test-support/llm-replay/src/index.ts:885-889`；`compaction-basic/tests/compaction-basic.spec.ts:2182-2187` |
| 4 | 基类默认返回 `undefined` | `llm/src/index.ts:236` |
| 5 | 请求发送路径**不调用** `imageRequestPricing` | 全仓 grep：runtime 内仅 `:236`/`:676`；`forAdapter`（`:992`）/ `prepareCall` / `streamWithRegistration` 均无引用 ⇒ 单槽不会被 LLM 请求本身打乱 |
| 6 | 钉住的对象内部闭包捕获 `connection` 快照与 `resolveAccess` 解析函数 | `request-pricing.ts:95` `connection.models.find(...)`、`:102/107` `resolveAccess?.(ref)`；`adapter.ts:22-25` 的 `imageAccess` 每次调用时**现读** `ctx.get('attachments')`/`ctx.get('fs')` |

⇒ **行号确认无误**：`packages/llm/llm/src/index.ts:676-678` 即目标。

### 1.3 记忆键如何取 `(provider, model)`（可靠来源）

`imageRequestPricing(provider: string, model: string)` 的两个入参**就是**键本身，无需旁证：

- token-meter 侧的来源：`_routeImagePricing()`（`:266-270`）读 `header.config.provider` / `header.config.model`，
  即 `canonicalHeader` 规范化后的请求封包（`session/src/request-header.ts`）；
- 与 token-meter memo 里已有的 `headerEquals → callConfigEquals`（比较 provider/model/reasoningEffort/temperature/maxTokens/stop）
  **同源**：键 `(provider, model)` 是它的子集，且由 `header` 键项兜住 ⇒ 换 provider/model 双重失效（路由级 + 封包级）；
- 反方向：`(provider, model)` 相同但 `header` 其它字段不同（如 tools）⇒ 由 token-meter 的 `optionalHeaderEquals` 兜住。

⇒ **无需从别处推导**，直接用方法实参；不存在"取不到"的场景。

### 1.4 生产收益路径：双调用确实存在且同输入

| 调用点 | 证据 |
|---|---|
| `region.ts:386` `prepareCompaction()` → `dependencies.meter.measure(session)` | 已读源码确认 |
| `region.ts:456` `assertWholeSurfaceUnchanged()` / `region.ts:484` `assertSelectedSpanStable()` | 已读源码确认（二者是 `assertStable` 的两种 stability 策略，一次事务只走其一） |
| 中间无 `session.append` | `summarizeCompaction()`（`:407-448`）只在 `:420` 发 LLM 调用；`commitCompactionBody`（`:491`）在其后 ⇒ **revision 不变** |
| 重试路径 | `:424 assertStable` → `:426 prepareCompaction` ⇒ 每轮 retry 再产生一对同输入调用 |

⇒ 一次 pressure 压缩：**386 →（456|484）第二次命中**；每 retry 再省 1 次。与第 1/2 轮结论一致。

---

## 2. 方案设计（唯一实现形态，两文件）

### 2.1 文件①：`packages/llm/llm/src/index.ts`

#### (a) 新增模块私有接口（置于 `:336` 之后、`export class LlmRuntime` 之前）

```ts
/**
 * One route's resolved request-image pricing, remembered so repeated
 * resolutions of the same route share one object identity.
 *
 * Adapters mint a fresh pricing object per call, so identity cannot say
 * whether a route's pricing is still the one a caller priced under. The
 * runtime therefore pins one resolution per exact route and drops it with
 * every topology publication, which is the only way {@link LlmRuntime.adapters}
 * ever changes: identity then means "same route, same registration
 * generation", which is exactly what a memo key needs.
 */
interface ImagePricingMemo {
  /** Provider route the pricing was resolved for. */
  readonly provider: string
  /** Exact model id the pricing was resolved for. */
  readonly model: string
  /** Pinned pricing, or `undefined` when the route declares none. */
  readonly pricing: LlmImageRequestPricing | undefined
}
```

> 放这里而非文件顶部：与 `AdapterRegistrationHandle`(`:297`)/`DirectoryRegistrationHandle`(`:321`) 同区，
> 且紧贴唯一使用者 `LlmRuntime`（与 token-meter 把 `MeasurementMemo` 放在 class 之前的风格一致）。
> `LlmImageRequestPricing` 已在 `:18` 以 `import type` 导入 ⇒ **零新增 import**。

#### (b) 新增字段（紧随 `:345-349` 的 `discoveries` 之后）

```ts
  private pricingMemo: ImagePricingMemo | undefined
```

- 无初始化器：`undefined` 在类型内，`strictPropertyInitialization` 不要求；先例 `llm-deepseek/src/host.ts:22` `let userId: AnonymousUserId | undefined`。
- 单槽（不用 `Map`）：① 无容量/淘汰分支（CI per-file 100% 分支门禁，见 §5.2）；② 命中只需两次 `===`；③ 最坏情况是 thrash ⇒ **退化为 miss-safe**，绝不串味（见 §2.3）。
- 槽必须能存 `pricing === undefined`（未注册 provider / 无定价路由），否则这类路由每次都要重新委派 adapter。

#### (c) 失效挂点（见 §2.4 选型：**推荐放在事件发布点**）

```ts
  private emitAdaptersUpdated(): void {
    // Route pricing is pinned per route: a new topology can only mean new pricing.
    this.pricingMemo = undefined
    // …既有派发逻辑一字不动…
  }
```

#### (d) 改写 `imageRequestPricing`（`:676-678`）

```ts
  imageRequestPricing(provider: string, model: string): LlmImageRequestPricing | undefined {
    const memo = this.pricingMemo
    if (memo !== undefined && memo.provider === provider && memo.model === model) return memo.pricing
    const pricing = this.adapters.get(provider)?.adapter.imageRequestPricing(provider, model)
    this.pricingMemo = { provider, model, pricing }
    return pricing
  }
```

- **签名 / 返回类型 / 导出面完全不变**；`@Remote` 契约不变；`api-catalog.ts:1419` 的签名文档不受影响。
- 顺序：先读槽（快路径）→ 委派 → 写槽。全是同步语句 ⇒ 无重入窗口（§2.3）。

#### (e) 补 JSDoc（`:667-675` 现有 JSDoc 末尾追加，不改 `@param`/`@returns` 行）

需记录三件事（**用文档而不是分支**，与 T1 处理 `cursor > session.seq` 的手法一致）：
1. 同一 `(provider, model)` 在同一拓扑世系内返回**同一个对象引用**（memo 友好）；
2. 槽随每次 `llm/adapters-updated` 发布丢弃 ⇒ 换 adapter / 注册注销 / `replace` 必换引用；
3. **显式前提**：钉住的对象捕获的是解析当时的**连接快照**（`catalogModel`、`imagePixelBudget`、`imageMaxBytes`）
   与**访问解析函数**（后者在每次 `priceImages` 时现读 `attachments`/`fs`，故路径变化仍可见）。
   即"一次注册世系内连接快照视为稳定"——这是本方案**未闭合**的 G-B，须显式列出（§3）。

### 2.2 文件②：`packages/llm/token-meter/src/index.ts`（恢复干净引用比较）

| # | 动作 | 位置 | 说明 |
|---|---|---|---|
| 1 | **删除** `pricingKey()` 整个 helper | `:112-128`（17 行） | 连同其 JSDoc；第 2 轮的 1-bit 变通彻底移除（连带消除 F5 那处脆弱的覆盖依据） |
| 2 | `MeasurementMemo.pricingKey: string \| undefined` → `pricing: LlmImageRequestPricing \| undefined` | `:87-88` | 字段注释改为"**Pinned** routed image pricing object"（强调由 runtime 钉住、身份 = 路由 + 世系） |
| 3 | 更新 `MeasurementMemo` 接口头 JSDoc | `:70-81` | 现文说 "pricing by value（both are rebuilt on every call）" ⇒ **必须改**：改为"pricing 按**引用**比较；`LlmRuntime` 为每个路由钉住一个对象并在每次拓扑发布时丢弃，故引用相等 ⇔ 同一路由世系" |
| 4 | 命中条件 `memo.pricingKey === pricingKey(pricing)` → `memo.pricing === pricing` | `:212` | 唯一一处判定改动 |
| 5 | `measurements.set(...)` 的 `pricingKey: pricingKey(pricing)` → `pricing` | `:258` | 写缓存项改名 |
| 6 | 更新 `measure()` JSDoc | `:189` | "the routed pricing compared by value" ⇒ "compared by identity (one object per route, dropped with every `llm/adapters-updated` publication)" |

- **不新增/不删除 import**：`LlmImageRequestPricing` 仍在 `:11` 导入并被 `_routeImagePricing` 与接口使用 ✅
- `_sync()` 先行、写在 `deepFreeze` 成功之后两条硬约束（第 1 轮裁定 A）**保持不变**；三分支（`:220-245`）一字不动。

### 2.3 并发 / 重入 / 多 provider 安全性

| 场景 | 分析 | 结论 |
|---|---|---|
| **重入** | `imageRequestPricing` 与 `measure()` 都是**纯同步**（无 `await`、无回调）；钉住发生在 `return` 之前的一条同步赋值 | 无 interleaving 窗口 ✅ |
| **多 provider / 多 model 交替** | 单槽键含 `(provider, model)`：不同键 ⇒ miss ⇒ 重新委派 ⇒ token-meter 侧拿到新对象 ⇒ **memo 也 miss**（保守方向） | 绝不串味；代价是交替场景下命中率下降（正确性不受影响）✅ |
| **多 `LlmRuntime` 实例**（多 fiber / 子 ctx，`service.spec.ts:581/594` 那种） | 槽是**实例字段**，各实例独立；各自订阅/失效互不影响 | ✅ |
| **同 revision 内被 summarize 抢占** | `region.ts:420` 的 await 期间若有别处解析了别的路由 ⇒ 槽被覆盖 ⇒ 本路由 miss ⇒ 重算 | 安全；仅损失一次命中 ✅ |
| **服务卸载 / fiber dispose** | 整个 `LlmRuntime` 实例随之丢弃，槽随之释放；钉住对象只捕获一个 `catalogModel` + 一个闭包 ⇒ 无泄漏 | ✅ |
| **spill-policy（`spill-policy/src/index.ts:65`）** | 取到钉住对象后**立即** `priceImages`；其测试用 `vi.spyOn(ctx.llm,'imageRequestPricing')` 完全绕过真实实现 | 不受影响 ✅ |

### 2.4 失效挂点选型（**需主Agent 裁定 C1**；推荐 A）

| 方案 | 做法 | 覆盖位置增量 | 差异 |
|---|---|---|---|
| **A（推荐）** 在 `emitAdaptersUpdated()` 内 `this.pricingMemo = undefined` | 1 行赋值 + 1 行注释，落在**事件的唯一发布点** | **0 个新分支 / 0 个新函数**（statement 随该函数执行即覆盖；该函数被 `topology.spec.ts` 等数百处 register/dispose 触发） | 只响应 `LlmRuntime` 自己的发布；而 `adapters` 的**全部**写入都紧跟该发布（§1.1-4）⇒ 等价完备。另含 directory 变更（`:519/:531`）也失效：多清一次，无害 |
| **B** 构造器内 `ctx.on('llm/adapters-updated', () => { this.pricingMemo = undefined })` | +1 订阅语句 +1 箭头函数 | +1 函数覆盖位置（会被大量 `registerAdapter` 用例覆盖，风险低） | 额外能响应**外部** `ctx.emit('llm/adapters-updated')`（如 `invariant.spec.ts:109` 那种）；但与本类现有风格（`ctx.effect`，无 `ctx.on`）略偏，且订阅在构造期、生命周期绑定 fiber |

⇒ 两者都在同一文件内、都满足"在既有的 `llm/adapters-updated` 事件上失效"。**推荐 A**：diff 最小、新增覆盖位置为零、
且 §1.1-4 已证明 `adapters` 无任何绕过该发布的变更路径。若主Agent 更看重"外部 emit 也失效"的鲁棒性，选 B。

---

## 3. 残留缝隙闭合判断（G-A / G-B / G-C / F6）

| 缝隙 | 定义 | 是否被 R2 闭合 | 依据 |
|---|---|---|---|
| **G-A** 同 provider 换 adapter（同 model、同 revision） | **✅ 闭合** | adapter 只能通过 `registerAdapter`/`replace`/disposer 变更，三者都走 `commitRoutes`/disposer → `emitAdaptersUpdated()`（`:470`/`:410`）⇒ 槽被清 ⇒ 下次解析拿到**新对象** ⇒ token-meter `memo.pricing === pricing` 为 false ⇒ miss。此外 `replace` 绑定同一 adapter 实例，行为本就不变 ⇒ 双重安全 |
| **G-B** 连接快照（`connection.models` / `imagePixelBudget` / `imageMaxBytes`）在同世系内变化 | **❌ 未闭合**（且**新增了影响面**，见下） | `adapter.ts:41` 每次 `imagePricing(this.dependencies.options(), …)`；钉住后 `catalogModel` 被冻结在首次解析时刻。`host.ts:41-48` 只在 **retryPolicy 变化**时 `registration.replace()`，即**只有 retryPolicy 变化才会触发拓扑事件**⇒ 纯 catalog/尺寸类变更不触发失效 |
| **G-C** `imageAccess` 解析变化（attachments / fs 服务挂载或路径映射改变） | **⚠️ 半闭合**（比第 2 轮好，但不完全） | 钉住的对象捕获的是**解析函数** `this.imageAccess`（`adapter.ts:22-25`），它在每次 `priceImages` 时**现读** `ctx.get('attachments')`/`ctx.get('fs')` ⇒ 只要 token-meter **未命中**（走了 `priceSurface`），文本价格是实时的（`llm-deepseek/tests/adapter.spec.ts:285-299` 正是这条性质的现存断言，见 §5.2 风险 R-3）。但**命中时** `priceSurface` 根本不执行 ⇒ 该窗口内仍会复用旧文本价格 |
| **F6**（G-B/G-C 的下游放大）命中时 `region.ts:457` 的 `isDeepStrictEqual(current.nodes, prepared.measurement.nodes)` 退化为同引用比较 | **⚠️ 随 G-B/G-C 残留** | 命中 ⇔ revision 未变 ∧ 路由未变 ∧ 世系未变；此时 surface 必相同，唯一可能不同的就是定价行为 ⇒ 断言空过的**唯一**情形就是 G-B/G-C。语义上不再是"无条件等价"，需按此重写第 1/2 轮的相关表述 |

### 3.1 未闭合项的处置：**显式前提 + JSDoc 记录**（需主Agent 签字，裁定 C3）

在 `LlmRuntime.imageRequestPricing` 的 JSDoc 与 `TokenMeter.measure()` 的 JSDoc 中各记一条，措辞要点：

> **前提（precondition）**：一次 adapter 注册世系内，路由的**连接快照**视为稳定——
> 影响定价行为的连接字段（`models` catalog 及其 `inputModalities` / `imagePixelBudget` / `imageMaxBytes`）
> 若在不改变 `retryPolicy` 的情况下被热更新，本记忆不会自动失效（对应事件 `llm/adapters-updated` 不会发布）。
> 访问路径解析（`imageAccess`）**不**在此前提内：它在每次 `priceImages` 时现读当前 `attachments`/`fs` 服务。

并明确列出三条**已知后果**：
1. 该窗口内 token-meter 可能复用一次陈旧的 route 价格（影响面：一次 shrink 决策 / 一次 route 价格读数，有界）；
2. 同窗口内 `spill-policy` 等非 memo 消费者也会读到钉住的连接快照（**这是本轮相对现状的新增影响面**，第 3.2 节给出可选消解）；
3. `region.ts:457` 的稳定性断言在该窗口内退化为同引用比较（F6）。

### 3.2 可选消解：R2+ 包装器变体（**不推荐默认采用**，供裁定）

若主Agent 认为"不应给非 memo 消费者引入新的陈旧窗口"：把钉住的**对象**换成钉住的**稳定句柄**——
`imageRequestPricing` 返回同一个 wrapper 对象（身份 = 路由 + 世系），其 `priceImages` **每次实时委派**
`this.adapters.get(provider)?.adapter.imageRequestPricing(provider, model).priceImages(images)`。

- 优点：身份仍稳定 ⇒ token-meter 照常命中（F1 仍解）；G-B/G-C 对 `spill-policy` 等**实时**消费者完全消解；
  G-C 在 token-meter 未命中时也实时。
- 代价：`priceSurface`（`route-pricing.ts:47`）每次 measure 只调一次 `priceImages` ⇒ 每次多一次 map 查找 + 一次 adapter 调用（O(1)，非 O(节点)）；
  但多一个函数覆盖位置（wrapper 的 `priceImages` 箭头，需被既有测试执行；`route-pricing.spec.ts` 会执行）。
- **不闭合**的部分不变：token-meter 命中窗口内的 G-B/G-C（因为命中时根本不调 `priceImages`）。
- 与本轮"用户已批准形态"的差异：记忆的是"句柄"而非"adapter 的对象"。**故列入裁定项 C1'，默认走批准形态（§2.1）。**

---

## 4. 收益与风险

### 4.1 收益

| 项 | 说明 |
|---|---|
| 生产是否真能命中 | ✅ `region.ts:386` 解析 → 槽钉住；`region.ts:456`/`:484` 同 `(provider, model)`、同 `LlmRuntime`、无拓扑事件 ⇒ **HIT** ⇒ token-meter `memo.pricing === pricing` 成立 ⇒ 跳过 `priceSurface`(O(S)) + `estimateToolsTokens`(tools stringify) + `structuredClone`(O(S)) + `deepFreeze`(O(S)) |
| 调用次数 | 一次 pressure 压缩 5 次 measure ⇒ 4 次重算（省 1 次全量测量）；每 retry 再省 1 次 |
| 附带收益 | `imageRequestPricing` 本身少一次 adapter 委派（新建对象 + `connection.models.find`） |
| 分辨力 | `pricing` 键从 1 bit 恢复为 `(provider, model, 世系)` 全分辨力 ⇒ 标准 2 成立 |

### 4.2 风险与缓解 / 回退

| # | 风险 | 等级 | 缓解 | 回退 |
|---|---|---|---|---|
| **R-1** | **新增分支持久可覆盖性**（CI per-file 100%：`packages/llm/llm/src/index.ts` 与 token-meter `index.ts` 均受门禁，`vitest.config.ts:217/370-377`：`perFile: true, statements/branches/functions/lines = 100`）。新增 `if` 的 true/false 与 3 个 `&&` 操作数都需被既有 spec 走到 | **中（最需盯）** | §5.2 逐位置给出**已定位的既有测试路径**；禁止 `v8 ignore`；实施者复核发现任一位置无既有路径 ⇒ **停下上报** | 停下上报主Agent（不得自行加 ignore） |
| **R-2** | **单槽被多路由打散**：同一 revision 内若有多 provider/model 交替解析，槽 thrash ⇒ token-meter 拿不到同一引用 ⇒ 命中率归零（**正确性不受影响**，只丢收益） | 低 | 生产压缩路径单一路由；即便打散也退化为"每次重算"，与改动前一致 | 无需回退 |
| **R-3** | **钉住改变既有可观察行为**：`llm-deepseek/tests/adapter.spec.ts:285-299` 三次 `price()`（同 provider/model）中间挂载 `LocalAttachments`/`MappedFiles` 并要求文本从"无挂载路径"变为"含 `/mounted/image.png`" | 中 | 已逐行静态推演：分支选择只取决于 catalog（不受挂载影响），文本里的路径由 `imageAccess` **现读** `ctx.get('fs')` 得出 ⇒ 钉住后 `:299` 仍成立 ✅（实施者须按此再走一遍） | 若推演有误 ⇒ 该用例红 ⇒ 改用 R2+ 包装器（§3.2） |
| **R-4** | G-B 的**新增影响面**（`spill-policy` 等非 memo 消费者也读钉住快照） | 中 | §3.1 JSDoc 显式记录；§3.2 提供 R2+ 消解 | 采纳 R2+，或退回 R4（第 2 轮清单里的"接受风险 + 补测试固定前提"） |
| **R-5** | 失效语句**有覆盖但无断言**：现有 spec 中没有任何用例断言"同 `(provider,model)` 在换 adapter 后返回新对象"（`topology.spec.ts:330-338` 的序列因单槽已被后续不同键覆盖，实际**证明不了**失效） | 中 | 静态依据充分（`adapters` 全部写入点紧跟 `emitAdaptersUpdated()`）；建议补 1 条反证测试（**需放宽到 tests/ ⇒ 裁定 C2**） | 若不允许加测试：以 JSDoc + §1.1 静态走查作为依据并显式记录"未被测试固定" |
| **R-6** | 第 2 轮文档（F1'/F4/F5/F6、`implementation.md` §5/§6.1）存在与新实现冲突的结论 | 低（文档） | `steps-r3.md` S6 要求同步修订，且**不覆盖**原文件（写 `implementation-r3.md` / 追加 `verification.md` §8） | — |

---

## 5. 验收对照表

### 5.1 对照 T2 的 5 条验收标准

| # | 标准 | 本轮对应 | 静态核对方法 |
|---|---|---|---|
| **1** | 引入 memo，命中时复用已冻结结果，避免重复 `structuredClone` + `deepFreeze` | token-meter `:209-215` 命中 `return memo.result`（在 `structuredClone` 之前）；命中由 "runtime 钉住 ⇒ 引用相同" 保证 | 读改后 `measure()`：`return memo.result` 在 `:247` `deepFreeze(structuredClone(...))` 之前；`memo.result` 唯一来源是 `:247` |
| **2** | 键覆盖所有影响结果的输入，输入变化必失效 | 键 = Session + revision + header 等值 + **pricing 引用（= 路由 + 世系）** + llm 引用。换 provider/model ⇒ 键变（且 `headerEquals` 也拦）；换 adapter ⇒ 事件 ⇒ 新对象 ⇒ miss；换 llm 实例 ⇒ miss；任何 append ⇒ miss | 逐字段回溯返回值 6 个字段的自由变量；核对 `emitAdaptersUpdated()` 覆盖 `adapters` 全部写入点（§1.1-4）；核对 G-B/G-C 已在 JSDoc 显式列出 |
| **3** | 仅改约定文件（本轮放宽为 `packages/llm/` 内 2 个文件）、无新依赖、TS 风格一致 | 2 文件；**零新增 import**；`readonly` 字段 + 逐字段 JSDoc；无 `!`/`as`/`any`/`??`/`?.`；`pricingKey` 删除 | `git diff --stat -- packages/llm/` = 2 files；`git diff -U0 -- packages/llm/ \| grep -c "^+import"` = 0 |
| **4** | 导出与调用方语义不变 | `imageRequestPricing` / `measure()` 签名不变；`api-catalog.ts:1419` 不受影响；9 处 `measure()` 生产调用点零改动；`priceSurface`/`estimate.ts` 等不动；抛错语义不变（`_sync` 仍在 memo 查询之前；写仍在 `deepFreeze` 之后） | grep 全仓 `.measure(`；确认 `:202` 仍为 `measure()` 首条语句；`:255` set 在 `:247` 之后 |
| **5** | before/after 说明 + 最小静态核对 | 本文件 §4.1/§5 + `steps-r3.md` 逐步核对动作 + `git diff --stat` 范围核验 | 按 `steps-r3.md` 逐步执行并记录到 `implementation-r3.md` |

### 5.2 CI per-file 100% 门禁自检（新增运行时位置清单）

门禁事实（已核实）：`vitest.config.ts:217` `coverage.include = ['packages/*/*/src/**/*.{ts,tsx}']`；
`:370-377` `thresholds: { perFile: true, statements: 100, branches: 100, functions: 100, lines: 100 }`；
`packages/llm/llm/src/index.ts` 与 `packages/llm/token-meter/src/index.ts` 均不在 exclude 名单内
（`grep "packages/llm" vitest.config.ts` 只命中一条 test include，非 coverage exclude）。

**文件① `packages/llm/llm/src/index.ts` 新增位置**

| 新增元素 | 类型 | 覆盖来源（已定位） |
|---|---|---|
| `interface ImagePricingMemo` | 纯类型 | 无运行时位置 ✅ |
| `private pricingMemo: ImagePricingMemo \| undefined` | statement（类字段定义） | 任何 `new LlmRuntime()` / `ctx.plugin(LlmRuntime)` ⇒ 数百处 ✅ |
| `this.pricingMemo = undefined`（`emitAdaptersUpdated` 首行） | statement | 该函数被 `topology.spec.ts:36/50/69/84/99/327/328`、`service.spec.ts` 数百处 register/dispose/replace 触发 ✅ |
| `const memo = this.pricingMemo` | statement | 每次 `imageRequestPricing` 调用 ✅ |
| `if (...)` **true** 路 | branch | `route-pricing.spec.ts:160→165`（`meter.measure` 两次，同 `'mock'/'vision'`，中间只有 append 不影响本槽）、`:220→224`、`:260→262`；`llm-deepseek/tests/adapter.spec.ts:287→290→299`（三次同 `('deepseek-official','deepseek-flash')`）✅ |
| `if (...)` **false** 路 | branch | `topology.spec.ts:330`（首次，槽空）、`:331`（model 变）、`:333`（provider 变）、`:336`、`:338` ✅ |
| `memo !== undefined` 操作数 | branch 两侧 | 空槽（`:330` 首次）/ 非空槽（`:331`）✅ |
| `memo.provider === provider` 操作数 | branch 两侧 | 真：`route-pricing.spec.ts:160→165`；假：`topology.spec.ts:333`（`'plain'` vs 槽里 `'a'`）✅ |
| `memo.model === model` 操作数 | branch 两侧 | 真：`:160→165`（同为 `'vision'`）；假：`topology.spec.ts:331`（`'other'` vs `'vision'`）✅ |
| `this.pricingMemo = { … }` | statement | 所有 miss 路径 ✅ |

**文件② `packages/llm/token-meter/src/index.ts` 变化位置**

| 变化 | 类型 | 覆盖来源（已定位） |
|---|---|---|
| 删除 `pricingKey()` | −1 function / −2 branch | 该位置消失，不再需要覆盖（连带修掉第 2 轮 F5）✅ |
| `memo.pricing === pricing`（`:212`）**true** | branch | `token-meter.spec.ts:634`（`llm` 未挂载 ⇒ 两侧 `undefined`）；`route-pricing.spec.ts:196→199`（两侧 `undefined`，且求值发生在 header 操作数之前）✅ |
| `memo.pricing === pricing` **false** | branch | `route-pricing.spec.ts:181→188`（`substitution` vs `fixedPricing`，且 revision/llm 相同 ⇒ 该操作数必被求值）✅ |
| `pricing` 操作数**被求值** | 需 memo 已存在 ∧ revision 相同 ∧ llm 相同 | 同上两处（`:199` 与 `:188`）✅ |

⇒ **结论：不需要也不得使用 `/* v8 ignore */`；不需要新增测试文件。**
⚠️ 与第 2 轮相同类型的脆弱点提示：`memo.pricing === pricing` 的 **false** 路依赖 `route-pricing.spec.ts:181→188`
这一处（`:181`/`:188` 之间不得插入 append、不得改为同 model）；`memo.model === model` 的 false 路依赖
`topology.spec.ts:331`。实施者须把这两条依赖写进 `implementation-r3.md` §5，便于日后门禁变红时排查。

---

## 6. 需主Agent 裁定项

| # | 事项 | 建议 |
|---|---|---|
| **C1** | 失效挂点：A（发布点 `emitAdaptersUpdated()`）/ B（构造器 `ctx.on('llm/adapters-updated')`） | **A**（零新增覆盖位置；`adapters` 无绕过发布的写入） |
| **C1'** | 是否采用 R2+ 包装器（钉住句柄而非对象，消解 G-B 对非 memo 消费者的新增影响面） | 默认**否**（走已批准形态）；若 C3 不接受 G-B 的新增影响面则改为**是** |
| **C2** | 是否允许补 1 条反证测试（同 provider/model 换 adapter ⇒ 新对象；需放宽到 `packages/llm/llm/tests/`）以固定失效语义 | 建议**允许**（1 条 `it`，落在已有 `topology.spec.ts` 的 `imageRequestPricing resolution` describe 内）；不允许则需签字接受"失效仅有 statement 覆盖、无断言固定" |
| **C3** | 是否接受 G-B/G-C 以"显式前提 + JSDoc"结案（否则需回到跨包方案 R1） | 建议**接受**；理由：G-B 触发需"同 revision + 同世系 + 仅 catalog/尺寸类连接字段热更新且不改 retryPolicy"三者叠加，后果有界（一次价格读数） |
| **C4** | 第 2 轮文档处置：F1'（保留）/ F4（本轮闭合 G-A、G-B/G-C 转前提）/ F5（随 `pricingKey` 删除自然失效）/ F6（按 §3 重写） | 写 `implementation-r3.md` + `verification.md` §8（追加，不覆盖） |

---

## 7. 最小静态核对清单（实施者照做）

1. **事实复核**：`llm/src/index.ts:676-678` 为本轮目标；`grep -n "emitAdaptersUpdated()"` = 1 定义 + 4 调用；
   `grep -n "this.adapters"` = 3 处写（`:408/:464/:467`）+ 4 处读，每处写之后都有 emit。
2. **键覆盖性**：对 `measure()` 返回值 6 个字段逐个回溯自由变量，确认能追溯到 5 个键项
   （Session / revision / header / **pricing 引用** / llm）；特别确认 `pricing` 引用在新方案下
   唯一对应 `(provider, model, 世系)`。
3. **等价性走查**：命中路径只 `return memo.result`；`memo.result` 唯一来源是 `deepFreeze(structuredClone(...))`；
   三分支（`:220-245`）一字未动；`_sync` 仍是首条语句；`set` 在 `deepFreeze` 成功之后。
4. **失效完备性**：确认 `adapters` 的 3 处写之后都调用了 `emitAdaptersUpdated()`；确认没有第二处派发。
5. **既有高风险用例复核**（R-3）：逐行走一遍 `llm-deepseek/tests/adapter.spec.ts:285-299`，
   确认钉住后 `:290`（`not.toContain('/mounted/image.png')`）与 `:299`（`toContain('/mounted/image.png')`）仍成立。
6. **门禁自检**：按 §5.2 两表逐条清点，特别是 `memo.model === model` 的两个方向与
   `memo.pricing === pricing` 的两个方向；把各自依赖的**唯一/少数**测试点写入 `implementation-r3.md`。
7. **范围核验**：见 `steps-r3.md` S5（`packages/llm/` 恰好 2 文件；`region.ts` 的 105 行必须逐字节不变）。
8. **未做运行时验证**：本环境无法 `pnpm test` / `tsc`，须在归档文件显式标注，并列出应跑套件：
   `packages/llm/llm/tests/*`、`packages/llm/token-meter/tests/*`、`packages/llm/llm-deepseek/tests/*`、
   `packages/spill/spill-policy/tests/*`、`packages/compaction/compaction-basic/tests/*`、
   `packages/compaction/command-compact/tests/*` + 覆盖率门禁脚本。

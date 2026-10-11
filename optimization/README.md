# optimization/ — 静态优化闭环归档（T1 / T2）

本目录是 deepseek-harness（`dsh`）两项**热路径算法优化**的完整过程归档：计划、原子步骤、实现说明、验证报告，按轮次保留。它们是**离线沙箱中的静态分析产出**，不是仓库官方文档（`README.md` / `AGENTS.md` / `docs/` 才是）。

## 状态

| 任务 | 目标 | 结论 | 归档 |
|---|---|---|---|
| T1 | `compaction-basic/src/region.ts` 中 `inspectCompactionEntryState()` 的**增量 fold** | 每步 O(N) 全量倒扫 → O(Δ) 增量折叠（O(N) 仅一次）；首轮闭环 | `optimization/T1/` |
| T2 | `token-meter` 中 `measure()` 的**结果 memo** | 同 revision + 同输入的重复调用不再重算；历经 4 轮迭代闭环 | `optimization/T2/` |

两项改动的共同特征：**签名、导出、调用方、依赖均不变，零新增 `import`**；新增的前提一律写成 JSDoc 前置条件，而不是不可达分支（该仓库对改动文件有 100% 分支覆盖门禁）。

## 阅读顺序

- **T1**（4 篇）：`T1/plan.md` → `T1/steps.md` → `T1/implementation.md` → `T1/verification.md`
- **T2**（6 篇，含第 3 轮重规划）：`T2/plan.md` → `T2/steps.md` → `T2/implementation.md` → `T2/verification.md` → `T2/plan-r3.md` → `T2/steps-r3.md`

> T2 必读提示：`T2/verification.md` 的 §0–§6 是**第 1 轮（未闭环）**的历史记录，§7 之后才是第 3 轮（R3）的最终结论。只读前半部分会得到错误的结论。

## 重要免责说明

1. **未做运行时验证。** 产出环境中无网络、仓库依赖未安装，因此没有执行 `tsc` / `vitest` / `oxlint` / 覆盖率门禁 / benchmark。所有结论均为**静态或结构化核对**。
2. **未做类型检查与测试覆盖复核。** 尤其 T2 新增了 memo 命中/未命中分支与一条多条件短路谓词，是否每条分支都有测试命中，必须在可执行环境下用 `vitest --coverage` 复核。
3. **文档中的行号可能已漂移。** 分析基于某一时刻的代码快照；请以**函数名、符号名**为准，不要依赖行号坐标。
4. **是否已提交或合入上游请以 git 为准。** 本目录记录的是该仓库工作区快照下的 `git diff`，不代表改动已被上游接受。

## T2 的关键教训

看似简单的 memo 会掉进两个陷阱：**对象身份不稳定**（键永不命中，收益归零）与**键分辨力不足**（命中陈旧结果）。第一次实现按引用比较 `pricing`，而生产路径每次都生成新对象，导致 memo 永不命中；改用值比较后又把分辨力降到 1 bit。真正的原因在依赖方——`LlmRuntime.imageRequestPricing()` 每次委派都新生成对象——因此最终把修复上移到 `LlmRuntime`：按 `(provider, model)` 单槽钉住，并在唯一的拓扑发布点 `emitAdaptersUpdated()` 清空。

这类缺陷只能靠严格的独立验证发现，不能靠实现者自证。

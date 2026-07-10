# 编码工作流：从需求到验收

记录"从一个模糊需求，到最终验收通过"这条完整工作流的设计过程。每个阶段单独开一个子目录，放该阶段的 skill 定义和相关说明；本文件是总览，后续每完善一个阶段就在这里更新一次。

## 目标管线（草案）

测试子流程的位置是一个明确决策点：已确认走 **TDD 风格**（测试用例先于功能代码落地，红-绿-重构），不是先实现后补测试的传统风格。

```text
需求 → PRD → Spec 拆分 → AI 闭环评审 → 架构设计 → 任务拆解 → 测试用例编写/AI审核闭环 → TDD实现闭环 → AI代码评审闭环 → 人审与UI自动化验收
 ①        ②          ②.5          ③         ④             ⑤⑥                 ⑦⑧⑨        ⑩             ⑪
```

| 阶段 | 目录 | 状态 | 说明 |
| --- | --- | --- | --- |
| ① 需求 → PRD | [`requirement-to-prd/`](./requirement-to-prd/SKILL.md) | ✅ 已完成设计 | 融合 brainstorming（发散框定）+ grill-with-docs（逼问收口），设计理由见 [`reference/0001-requirement-to-prd-design.md`](./reference/0001-requirement-to-prd-design.md) |
| ② PRD → Spec 拆分 | [`prd-to-specs/`](./prd-to-specs/SKILL.md) | ✅ 已完成设计 | 把产品视角的 PRD 拆成一个或多个 OpenSpec 风格的工程 Spec，明确每个 Spec 的交付边界、行为契约、依赖关系和追溯到 PRD 的验收口径。设计理由见 [`reference/0002-prd-to-specs-design.md`](./reference/0002-prd-to-specs-design.md) |
| ②.5 PRD/Spec → AI 闭环评审 | [`prd-spec-closure-review/`](./prd-spec-closure-review/SKILL.md) | ✅ 已完成设计 | 在交给人类审查前，独立检查 PRD 的目标、范围、成功指标、验收口径是否被 proposal/specs 完整承接；只输出阻塞问题、覆盖检查、人审焦点，不做架构设计或任务拆解 |
| ③ Spec → 架构设计 | [`spec-to-architecture/`](./spec-to-architecture/SKILL.md) | ✅ 已完成设计 | 针对单个已通过闭环评审的工程 Spec 产出 Architecture Design Doc，并为重要、难逆转、有真实 trade-off 的决策产出 ADR。设计理由见 [`reference/0003-spec-to-architecture-design.md`](./reference/0003-spec-to-architecture-design.md) |
| ④ 架构设计 → 任务拆解 | [`architecture-to-tasks/`](./architecture-to-tasks/SKILL.md) | ✅ 已完成设计 | 将已批准的单个 Architecture Design Doc 拆成约 300 行代码 review 粒度的任务计划，明确依赖顺序、风险、回滚和交接给测试用例编写阶段的关注点；不写测试用例、测试代码或功能代码。设计理由见 [`reference/0004-architecture-to-tasks-design.md`](./reference/0004-architecture-to-tasks-design.md) |
| ⑤⑥ 任务拆解 → 测试用例编写/AI审核闭环 | [`test-case-authoring-review/`](./test-case-authoring-review/SKILL.md) | 已完成设计 | 由 Author Agent 产出测试用例文档，再新开 Reviewer Agent 按 [`REVIEWER.md`](./test-case-authoring-review/REVIEWER.md) 独立审核；Author 根据阻塞问题修订并复审，直到形成 AI 已评审版本后再交给人审。设计理由见 [`reference/0005-test-case-authoring-review-design.md`](./reference/0005-test-case-authoring-review-design.md) |
| ⑦⑧⑨ 测试用例闭环 → TDD实现闭环 | [`test-driven-implementation/`](./test-driven-implementation/SKILL.md) | 已完成设计 | 将人审通过的 AI 已评审用例落地为测试代码，按 Red-Green-Refactor 完成功能编码、单元测试和单模块接口集成测试；本阶段内测试失败或实现有问题时直接修复并重跑。设计理由见 [`reference/0006-test-driven-implementation-design.md`](./reference/0006-test-driven-implementation-design.md) |
| ⑩ TDD实现闭环 → AI代码评审闭环 | 待建 | 🔲 未开始 | TDD 阶段通过后，新开代码评审 agent 进行一轮 AI code review；发现问题则退回 ⑦⑧⑨ 修复，修复后再次触发 AI 复审，直到无阻塞问题。候选目录名：`ai-code-review-loop/` |
| ⑪ AI代码评审闭环 → 人审与UI自动化验收 | 待建 | 🔲 未开始 | 人触发面向累计变更的一轮 AI 代码评审（此时可能包含多个 spec），再由人类 review；人审后由 AI 编写从 UI 触发的 Playwright 集成测试用例并运行 UI 自动化测试，全部通过后结束。候选目录名：`human-review-ui-acceptance/` |

> 阶段划分和候选素材都是讨论过程中的暂定结论，不是最终方案，后续设计每个阶段时应该重新核实这些 agent/skill 当前是否还存在、内容是否有变化，不要直接假设当时看到的版本还成立。

## 目录结构约定

本目录下分三种内容，职责严格分开，不要混着写：

- **`<阶段英文名>/SKILL.md`**（比如 `requirement-to-prd/`）：**开箱即用的 skill 本体**，不带序号前缀、不带"这是第几阶段""为什么这样设计"之类的说明——目的是可以直接整个文件夹复制到任意项目的 `.claude/skills/`（或 Cursor 等编辑器的等效目录）下就能用。阶段顺序信息只存在于本 README 的表格里，不写进 SKILL.md。
- **`reference/`**：ADR 风格的设计说明，一个决策一个文件，命名 `NNNN-阶段名-design.md`（四位数字编号，独立于阶段顺序编号）。这里回答"为什么这么设计、参考了什么、放弃了什么方案"，不影响 SKILL.md 的可移植性。
- **`README.md`**（本文件）：整体路线图、阶段状态、跨阶段的衔接原则。

如果某阶段需要额外的模板/格式说明文件（类似 grill-with-docs 里 `domain-modeling` 附带的 `ADR-FORMAT.md`），跟 `SKILL.md` 放在同一个阶段子目录下，不要提前为还用不上的阶段建空文件。

## 阶段衔接的设计原则

这是讨论过程中反复确认过的几条通用原则，设计后续阶段时应保持一致：

1. **每个阶段只做自己该做的事**，不要提前处理下一阶段的内容（例如阶段①明确规定不产出技术方案和 ADR）。上一阶段发现"越界"的冲动时应该停下来，把内容标记为"留给下一阶段"。
2. **PRD 不等于工程设计单元**：PRD 是产品视角的完整需求，可能同时覆盖前端、后端、契约、数据迁移、异步任务等多个工程边界。进入架构设计前，必须先拆成可单独设计、可单独计划的 Spec；不要把多个实现域混进一份大而全的设计文档。
3. **Spec 拆分优先保证边界清晰**：每个 Spec 应明确自己的职责、行为契约、依赖对象、非目标，以及如何追溯到 PRD 的目标和验收口径。前端、后端、前后端契约、基础设施或数据迁移可以拆成不同 Spec；只有边界足够小且实现域一致时才合并。
4. **访谈类 skill 的交互节奏要匹配**：单线程逐题访谈（一次一问、优先选择题）之间可以顺畅衔接；如果某个候选素材是多子代理并行讨论型（比如 `council`），它更适合用在"已经有多个候选方案、需要仲裁选一个"的节点，而不是访谈节点，接入位置要单独判断，不能和访谈类 skill 简单首尾相连。
5. **外部 skill 拿来用之前要重新核实内容**，不要凭 skill 名字或历史印象假设它的行为——多个外部 skill（比如 `grill-with-docs`）实际上只是对其他 skill 的组合调用壳，真正的机制在被引用的 skill 里。
6. **人审前先做 AI 闭环评审**：阶段②产出的 proposal/specs 不应直接丢给人类评审；先由独立评审 skill 检查 PRD → Spec 的覆盖、追溯、验收和边界是否闭环，避免把明显缺口转嫁给人类。
7. **不是每个环节都要跑满全套流程**：需求/方案本身清晰、无歧义风险时，允许精简成一轮访谈；完整流程是为高歧义场景准备的保底方案，不应作为唯一路径。

## 待办

- [x] 设计阶段②（PRD → Spec 拆分）：确认 OpenSpec 风格 Spec 的最小模板、目录结构、拆分粒度，以及 `council` 是否只在多种拆分方案难以裁决时启用。
- [x] 设计阶段②.5（PRD/Spec → AI 闭环评审）：新增人审前质量门，检查 PRD 目标、范围、成功指标、验收口径是否被 proposal/specs 完整承接。
- [x] 设计阶段③（Spec → 架构设计）：确认 `architect` / ADR 各自的接入点，尤其是 Architecture Design Doc 和 ADR 的边界。
- [x] 设计阶段④（架构设计 → 任务拆解）：确认任务粒度以约 300 行代码变更为 review 参考线，任务可包含多个内部步骤，但不输出测试用例、测试代码、功能代码或新架构决策。
- [x] 设计阶段⑤⑥（任务拆解 → 测试用例编写/AI审核闭环）：确认 ⑤/⑥ 合并为一个复合 skill，内部由 Author Agent 和独立 Reviewer Agent 循环，阶段⑦仍独立负责测试代码落地。
- [x] 设计阶段⑦⑧⑨（测试用例闭环 → TDD实现闭环）：确认测试代码落地、功能编码、单元测试和单模块接口集成测试如何按 Red-Green-Refactor 组织，核实 `tdd-guide` agent 当前内容是否还适用。
- [ ] 设计阶段⑩（TDD实现闭环 → AI代码评审闭环）：确认代码评审 agent 的准入、输出格式、阻塞问题回退到 ⑦⑧⑨ 的复审机制。
- [ ] 设计阶段⑪（AI代码评审闭环 → 人审与UI自动化验收）：确认人触发的累计变更 AI 复审、人类 review、Playwright UI 集成测试用例编写和 UI 自动化测试运行的边界。
- [ ] 全部阶段设计完成后，评估是否需要一个顶层"调度"文档/脚本，把十一个阶段串起来自动衔接，而不是每次手动切换。

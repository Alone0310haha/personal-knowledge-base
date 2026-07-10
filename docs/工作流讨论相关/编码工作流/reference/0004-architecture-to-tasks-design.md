# ADR 0004: architecture-to-tasks Skill 设计

## Context

阶段③已经针对单个已闭环 Spec 产出 Architecture Design Doc，并明确不创建 `tasks.md`、不写测试用例、不写功能代码。阶段④需要承接这份架构设计，把组件边界、接口、数据流、错误处理、安全和观测性拆成可执行任务，同时为后续测试用例编写阶段提供稳定输入。

评估了现有候选素材：

- ECC `planner` agent：擅长需求分析、架构回看、步骤拆解、依赖排序、风险识别和成功标准，但默认会输出 Testing Strategy，且容易产生文件级实现清单。
- superpowers `writing-plans` skill：在本机未找到可核实实现，因此不能把它作为可直接复用的依据，只能作为历史候选名保留。
- OpenSpec 的 proposal/spec/design/tasks 流：artifact 顺序适合当前管线，但本项目已经把 proposal、spec、architecture design、task planning、TDD 测试子流程拆成独立阶段，不能直接套完整流程。

用户进一步确认：新 `SKILL.md` 必须是可复制、可独立使用的 skill 本体，不能出现“参考 ECC / OpenSpec / superpowers / writing-plans”这类外部项目引用或历史来源说明。外部材料中的好内容应改写成本阶段自己的规则、模板、流程和自检项；来源和取舍只放在 reference 设计说明中。

## Decision

新增阶段④ `architecture-to-tasks`，采用“受控任务规划”流程：

1. 输入必须是已批准的单个 Architecture Design Doc，并能追溯到已通过闭环评审的 Spec。
2. 输出为任务计划文档，默认路径为：

```text
openspec/changes/<change>/tasks/<spec-domain>-tasks.md
```

1. 如果项目已有单文件任务计划惯例，可以使用：

```text
openspec/changes/<change>/tasks.md
```

1. 任务粒度以人工 review 为中心：单个任务的功能代码变更量大致控制在 300 行左右；超过约 300 行且包含多个职责、风险点或 review 关注点时继续拆分，过小且必须一起理解/提交/验证时合并。
1. 一个任务可以包含多个内部步骤；步骤说明执行顺序，任务才是 review、测试用例设计和实现推进的基本单元。
1. 阶段④不输出 Given-When-Then 测试用例、测试代码、功能代码、patch、逐文件实现清单、工期排期或人员分配。
1. 阶段④不重新选择架构方案，不新增 ADR；发现会改变组件边界、接口契约或数据流的阻塞问题时退回阶段③。
1. `architecture-to-tasks/SKILL.md` 不写外部项目引用，直接沉淀为自包含规则；本 ADR 记录来源、取舍和设计理由。

## Consequences

### Positive

- 阶段③和阶段⑤之间有明确交接物，避免架构文档直接跳到测试用例或代码实现。
- 任务粒度围绕人工 review 设计，比“完整实现清单”更适合日常代码评审。
- 保留依赖分析、实施顺序和风险识别能力，同时避免提前写测试策略或代码。
- `SKILL.md` 可以独立复制到其他项目，不携带本知识库的历史讨论和外部项目引用。

### Negative

- 300 行只是估算线，不可能在设计阶段精确预测；执行时仍需根据实际 diff 调整。
- 相比直接列文件修改清单，任务计划更抽象，需要实现阶段再映射到具体代码。
- 对很小的架构变更可能显得流程偏重，因此 skill 提供 Lite 模式。

## Alternatives Considered

- **直接使用 ECC `planner` agent**：它能产出实施步骤、依赖和风险，但 Testing Strategy 与阶段⑤冲突，文件级计划也容易越界到实现阶段。
- **直接使用 OpenSpec `tasks.md` 风格**：顺序上合适，但当前管线把测试用例编写、测试用例审核、测试编码、功能编码拆成独立阶段，不能让 tasks 承担过多内容。
- **合并阶段④和阶段⑤**：可以减少一次交接，但会把“拆任务”和“写测试用例场景”混在一起，削弱 TDD 子流程的清晰度。
- **一个任务等于一个步骤**：粒度过碎，容易变成代码操作清单，不利于 review。
- **一个任务覆盖完整功能域**：粒度过大，review 成本高，也难以并行推进。

## Status

Accepted

## Date

2026-07-10

## References

- `D:\code\self\ECC\agents\planner.md`
- OpenSpec proposal/spec/design/tasks artifact flow
- 2026-07-10 用户确认：任务粒度约 300 行代码，便于人工 review
- 2026-07-10 用户确认：新 skill 本体不出现外部项目引用，外部项目好内容需改写进 skill

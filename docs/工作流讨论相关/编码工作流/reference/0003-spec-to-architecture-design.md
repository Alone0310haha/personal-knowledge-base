# ADR 0003: spec-to-architecture Skill 设计

## Context

阶段②已经把 PRD 拆成 OpenSpec 风格的 proposal/specs，阶段②.5 又增加了 PRD/Spec 闭环评审。新的阶段③不应再从 PRD 直接写一份总架构，而应针对单个已闭环 Spec 进行架构设计。

评估了现有候选素材：

- ECC `architect` agent：适合作为主流程，覆盖现状分析、需求/NFR、架构方案、组件职责、数据流、集成点和 ADR。
- ECC `code-architect` agent：会输出 files to create/modify 和 build sequence，更接近实现蓝图，容易越界到阶段④任务拆解。
- ECC `architecture-decision-records` skill：适合记录 ADR，但不产出 Architecture Design Doc。
- ECC `api-design` skill：适合作为 API/契约类 Spec 的专项参考，不是通用流程。
- `grill-with-docs`：适合术语逼问和轻量 ADR 判断，但不是 OpenSpec Spec 到架构设计的完整流程。

## Decision

新增阶段③ `spec-to-architecture`，以 ECC `architect` 的流程为主，但用流水线边界重新约束：

1. 输入必须是已通过 `prd-spec-closure-review` 的单个 OpenSpec spec。
2. 输出为 Architecture Design Doc，并在必要时输出 ADR。
3. 不输出任务列表、文件修改清单、build sequence、测试用例或代码。
4. 如果 spec 仍混入多个工程边界，退回阶段②重新拆分。
5. 只在多个架构方案都合理且需要仲裁时使用 `council`。
6. API/契约类 Spec 可以参考 `api-design`，但不把 API 细节扩展成实现任务。

默认 design 路径采用：

```text
openspec/changes/<change>/design/<spec-domain>-architecture.md
```

ADR 默认走项目已有 `docs/adr/`，没有则按 ADR skill 的规则懒创建。

## Consequences

### Positive

- 保持"一个 Spec，一份 design"，避免重新退化成大而全架构文档。
- 让阶段④任务拆解有明确输入：组件边界、数据/控制流、接口、错误处理、安全和可观测性。
- ADR 只记录真正重要的架构 trade-off，不污染文档库。
- 可以复用现有 `architect` agent 的架构判断能力，同时避免 `code-architect` 的任务拆解倾向。

### Negative

- 比直接调用 `architect` 更严格，需要先确认 closure review 和单 Spec 边界。
- 对很小 Spec 可能显得流程偏重，因此 skill 提供 Lite 模式。
- 如果项目没有明确 ADR 目录，首次 ADR 需要用户确认创建位置。

## Alternatives Considered

- **直接使用 ECC `architect` agent**：足够做架构设计，但缺少本管线的准入、单 Spec 边界、OpenSpec 路径和禁止任务拆解约束。
- **使用 `code-architect` agent**：它的输出更可执行，但会提前产出文件清单和 build sequence，和阶段④冲突。
- **把 ADR 写入 Architecture Design Doc，不单独建 ADR**：更轻量，但关键决策不容易被后续跨功能检索和复用。
- **合并阶段③和阶段④**：效率更高，但会把架构选择和任务拆解混在一起，降低后续 TDD 流程的清晰度。

## Status

Accepted

## Date

2026-07-08

## References

- `D:\code\self\ECC\agents\architect.md`
- `D:\code\self\ECC\agents\code-architect.md`
- `D:\code\self\ECC\skills\architecture-decision-records\SKILL.md`
- `D:\code\self\ECC\skills\api-design\SKILL.md`

# ADR 0002: prd-to-specs Skill 设计

## Context

原路线图曾把阶段②设计为"PRD → 架构设计"。讨论后发现这个跳转过大：PRD 是产品视角的完整需求，可能同时覆盖前端、后端、API 契约、数据迁移、异步任务等多个工程边界。如果直接写一份架构设计文档，容易把多个实现域混在一起，导致后续任务拆解和 TDD 用例都变得含混。

用户提出应在 PRD 和架构设计之间增加 OpenSpec 风格的 Spec 拆分。参考了 OpenSpec 的两个核心文档：

- `docs/writing-specs.md`：强调 spec 是行为而不是代码，requirement 应该是单一、可观察、使用 RFC 2119 关键字的行为声明，scenario 应该能成为测试。
- `docs/concepts.md`：定义了 `openspec/specs/` 作为当前行为源、`openspec/changes/` 作为 proposed modification；change folder 包含 proposal、specs、design、tasks，但 artifact flow 是 proposal → specs → design → tasks。

## Decision

新增阶段② `prd-to-specs`，将原"PRD → 架构设计"拆成：

1. **PRD → Spec 拆分**：产出 OpenSpec 风格的 `proposal.md` 和 `specs/**/spec.md`，不产出 `design.md` 或 `tasks.md`。
2. **Spec → 架构设计**：针对已批准的 spec 产出 Architecture Design Doc，并为关键决策产出 ADR。

阶段②采用这些规则：

- 一个 PRD 如果只有一个产品意图，可以对应一个 OpenSpec change；如果包含多个独立意图，拆成多个 change folder。
- 一个 change 可以包含多个 spec，例如同一产品意图下的 API、UI、contract、worker、migration spec。
- Spec 只写行为契约：requirements + scenarios。
- 实现细节、架构方案、任务步骤全部留给后续阶段。
- 当拆分方式存在多个合理方案时，才使用 `council` 作为仲裁节点；它不是主流程。

## Consequences

### Positive

- 后续架构设计可以按单个工程边界展开，避免前端、后端、契约和迁移混进一份大设计。
- Spec 的 Given/When/Then scenarios 能直接喂给后续测试用例编写阶段。
- OpenSpec 的 delta spec 模型适合 brownfield 项目，能区分新增、修改、移除行为。
- PRD 的产品目标和验收口径可以通过 traceability 表追溯到具体 spec。

### Negative

- 管线从十步增加到十一步，流程更长。
- 需要额外维护 change/spec 边界；对极小需求可能显得重。
- 如果误把技术方案写进 spec，会污染后续架构设计阶段，因此 skill 需要强约束不创建 `design.md` 和 `tasks.md`。

## Alternatives Considered

- **继续 PRD → 架构设计**：流程更短，但高概率产出大而全的架构文档，特别是全栈需求中前端、后端和契约边界不清。
- **只用 OpenSpec 的完整 proposal/spec/design/tasks 流程**：和整体管线冲突，因为本项目已经把 architecture design、task planning、TDD 拆成独立阶段。
- **把 Spec 拆分合并进 PRD 阶段**：不合适。PRD 阶段是产品需求收口，过早考虑工程边界会把需求讨论拉向实现域。

## Status

Accepted

## Date

2026-07-07

## References

- https://github.com/Fission-AI/OpenSpec/blob/main/docs/writing-specs.md
- https://github.com/Fission-AI/OpenSpec/blob/main/docs/concepts.md

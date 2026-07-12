# ADR 0008: human-review-ui-acceptance Skill 设计

## Context

阶段⑩已经对单 Spec / 单任务范围内的代码 diff、测试 diff 和 TDD 证据做独立 AI 代码评审。通过阶段⑩后，单个实现单元具备进入人审的条件，但最终验收还需要处理三个不同层面的事情：

- 多个 Spec / task 合并后的累计风险，例如接口契约、数据迁移、配置、权限、部署顺序和验收覆盖。
- 人类 review 的明确批准或变更请求。
- 从 UI 触发的 Playwright 自动化验收，用用户可观察行为验证 PRD acceptance criteria 和关键 Spec scenarios。

此前已经确认 Playwright / UI 自动化不属于 TDD 实现阶段，也不属于单任务 AI 代码评审阶段。它应该在最终人审通过后执行，避免在代码仍可能被人类要求修改时提前维护 UI 验收脚本。

## Decision

新增阶段⑪ `human-review-ui-acceptance`，采用“累计 AI 复审 + 人类 Review Gate + Playwright UI 验收 + 最终报告”的流程。

1. 输入必须是一个或多个已经通过阶段⑩的变更单元；每个相关单元都应有 TDD 证据和 `Approved for Human Review` 的 AI Code Review Report。
2. 本阶段先执行累计 AI 复审，检查跨 Spec / 跨任务的集成风险和 PRD 验收覆盖。
3. 累计复审通过后，准备人类 review 包，并等待人类明确结论。
4. 只有人类 review 明确 `Approved` 后，才编写和运行 Playwright UI 验收测试。
5. UI 验收只覆盖用户可触发、可观察的关键验收路径；纯模块行为仍由 UT / IT 覆盖。
6. UI 验收失败必须先分类为 Product Failure、Test Script Failure、Environment Failure 或 Spec Ambiguity，再决定退回实现、修测试、修环境或请求人类裁决。
7. 采用四文件结构：
   - `human-review-ui-acceptance/SKILL.md`：主 skill 本体，定义准入、边界、总流程和最终报告。
   - `human-review-ui-acceptance/CUMULATIVE_REVIEWER.md`：累计变更 AI 复审规则。
   - `human-review-ui-acceptance/PLAYWRIGHT_ACCEPTANCE.md`：UI 验收用例选择、编写、运行和失败分类规则。
   - `reference/0008-human-review-ui-acceptance-design.md`：记录本决策的背景、取舍和影响。
8. 输出建议包括：

```text
openspec/changes/<change>/reviews/<change>-cumulative-ai-review.md
openspec/changes/<change>/acceptance/<change>-ui-acceptance-plan.md
openspec/changes/<change>/acceptance/<change>-final-acceptance.md
```

## Consequences

### Positive

- 保留最终验收的人类责任，避免 AI 代表人类批准。
- 在人审前增加累计变更复审，弥补阶段⑩只看单任务范围的盲区。
- UI 自动化验收被放在人审之后，减少因人审修改导致验收脚本反复失效。
- Playwright 测试聚焦用户可观察行为，不与 UT / IT 职责重叠。
- 失败分类让产品问题、脚本问题、环境问题和规格歧义走不同处理路径。

### Negative

- 最终阶段流程更重，需要维护累计复审报告、UI 验收计划和最终验收报告。
- UI 自动化依赖项目已有运行环境、认证、测试数据和 Playwright 基础设施；基础设施不足时可能需要先补环境能力。
- 对无 UI 变更或纯后端变更，Playwright 可能不适用，因此 skill 提供 Lite 模式。

## Alternatives Considered

- **把累计 AI 复审放在阶段⑩**：会让阶段⑩从单任务评审扩张到最终验收，破坏它的清晰边界。
- **人审前先写 Playwright 测试**：可以更早发现 UI 问题，但人类 review 可能要求调整行为，导致 UI 验收脚本反复返工。
- **让 AI 自动批准人审**：流程最顺，但违背人类最终责任边界。
- **所有验收都用 Playwright 覆盖**：看似统一，但会把纯服务层、数据库和内部规则硬塞进 UI 测试，造成慢、不稳定且难定位的测试。
- **无 UI 变更时跳过阶段⑪**：流程更轻，但仍需要累计 AI 复审和人类 review gate；因此保留 Lite 模式，而不是跳过整个阶段。

## Status

Accepted

## Date

2026-07-12

## References

- `ai-code-review-loop/SKILL.md`
- `ai-code-review-loop/REVIEWER.md`
- `reference/0007-ai-code-review-loop-design.md`
- `test-driven-implementation/SKILL.md`

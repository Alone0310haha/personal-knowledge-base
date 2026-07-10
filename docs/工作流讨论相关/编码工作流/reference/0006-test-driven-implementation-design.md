# ADR 0006: test-driven-implementation Skill 设计

## Context

阶段⑤⑥已经将任务计划转换成 `AI_REVIEWED` 测试用例文档，并在交给代码实现前完成 AI 审核和人类 review。后续需要进入代码层面的 TDD 实现：先把用例落地为可运行测试代码，再写功能代码，再运行相关测试。

原路线图把测试用例编码、功能编码实现、测试用例运行拆成 ⑦、⑧、⑨ 三个阶段。用户进一步确认：按 TDD 工作模式，这三步应放在一个闭环中处理。测试代码、功能实现和测试运行天然互相依赖，拆成独立阶段会让 RED/GREEN/Refactor 的证据链断开。

用户同时确认后续阶段边界：

- ⑦⑧⑨ 负责 TDD 流程：编码、单元测试、单模块接口集成测试。
- ⑩ 负责 AI 代码评审闭环；发现问题退回 ⑦⑧⑨ 修复和重跑测试。
- ⑪ 负责人触发的累计变更 AI 复审、人类 review、从 UI 触发的 Playwright 集成测试用例编写和 UI 自动化测试运行。

调研 ECC `tdd-guide` / `tdd-workflow` 后，确认其 Red-Green-Refactor、测试先行和证据报告思路可借鉴。但它默认覆盖 unit、integration、E2E，并要求全局覆盖率目标；这与本路线图将 Playwright / UI 自动化验收放到阶段⑪的边界不同。因此本阶段不直接复用 ECC TDD skill，而是沉淀为收窄后的单模块 TDD 实现闭环。

## Decision

新增阶段⑦⑧⑨ `test-driven-implementation`，采用“测试代码落地 + 最小功能实现 + 单模块验证”的 TDD 闭环。

1. 输入必须是已人审通过的 `AI_REVIEWED` 测试用例文档，并能追溯到已批准任务计划、Spec、Architecture Design 和必要 ADR。
2. 输出以代码变更和测试结果为主，可按项目惯例输出轻量 TDD 证据摘要，默认路径为：

```text
openspec/changes/<change>/evidence/<spec-domain>-tdd-evidence.md
```

1. 阶段⑦⑧⑨合并为一个 skill，而不是拆成三个独立 skill。
2. RED 阶段必须先写测试代码并验证新增测试失败；失败原因应来自功能尚未实现，而不是测试环境或编译错误。
3. GREEN 阶段只写最小功能实现，不能删除、跳过或弱化测试。
4. Refactor 阶段只在测试保护下优化局部结构，不扩大到无关模块。
5. Module Verification 只覆盖相关单元测试、单模块接口集成测试和必要构建/静态检查。
6. 本阶段不写 Playwright、UI E2E 或跨系统验收测试。
7. 本阶段不做 AI 代码评审；评审由阶段⑩独立执行。
8. `test-driven-implementation/SKILL.md` 不写外部项目引用，直接沉淀为自包含规则；本 ADR 记录来源、调研和取舍。

## Consequences

### Positive

- 保留完整 TDD 证据链：测试先失败，再由实现变绿，再在测试保护下重构。
- 避免测试代码、功能代码、测试运行拆开后造成上下文断裂。
- 明确 ⑦⑧⑨ 与 ⑩ 的边界，避免自写自审。
- 明确 ⑦⑧⑨ 与 ⑪ 的边界，避免把 Playwright / UI 验收提前混进模块实现阶段。
- TDD 证据摘要可以帮助后续 AI 代码评审和人类 review 快速理解验证范围。

### Negative

- 相比单纯写代码，本阶段会强制记录 RED/GREEN/Refactor 证据，流程更重。
- 对很小的变更，单独 evidence 文件可能显得过度，因此 skill 提供 Lite 模式。
- 单模块接口集成测试边界需要依赖项目已有测试基础设施；项目基础设施不足时，可能需要先补测试环境或退回任务拆解确认。

## Alternatives Considered

- **拆成 ⑦、⑧、⑨ 三个独立 skill**：阶段名更细，但 RED/GREEN/Refactor 强依赖同一上下文，拆开会弱化 TDD 证据链。
- **合并 ⑦⑧⑨⑩**：可以减少交接，但代码实现和代码评审应由独立视角完成，否则容易自写自审。
- **合并 ⑦⑧⑨⑩⑪**：可以形成“大实现到验收”总流程，但会把模块级 TDD、AI 代码评审、人审和 UI 自动化验收混在一起，边界过大。
- **直接复用 ECC `tdd-guide` / `tdd-workflow`**：其 TDD 基本流程可借鉴，但默认包含 E2E 和全局覆盖率目标；本阶段只处理 UT 与单模块接口 IT，UI 自动化测试留给阶段⑪。
- **不输出 TDD 证据摘要**：流程更轻，但后续 AI 代码评审和人审难以快速确认 RED/GREEN/Refactor 是否真实发生。

## Status

Accepted

## Date

2026-07-10

## References

- `https://raw.githubusercontent.com/affaan-m/ECC/main/agents/tdd-guide.md`
- `https://raw.githubusercontent.com/affaan-m/ECC/main/skills/tdd-workflow/SKILL.md`
- 2026-07-10 用户确认：阶段⑦⑧⑨应按 TDD 流程合并，覆盖编码、单元测试和单模块接口集成测试。
- 2026-07-10 用户确认：阶段⑩独立负责 AI 代码评审闭环，阶段⑪独立负责人审与 UI 自动化验收。

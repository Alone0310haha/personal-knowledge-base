# ADR 0005: test-case-authoring-review Skill 设计

## Context

阶段④已经产出任务计划，并明确只交接给测试用例编写阶段，不写 Given-When-Then 测试用例、测试代码或功能代码。后续需要把任务计划中的 Outcome、Done When、Risk、Integration Points 和 Handoff 转换成可供测试代码落地的测试用例文档。

原路线图把“测试用例编写”和“测试用例审核”拆成两个连续阶段。用户进一步确认：这两个动作不应该是一次性串行交接，而应该形成 AI 内部闭环。Author Agent 写完测试用例后，必须新开独立 Reviewer Agent 审核；Author 根据审核意见修改，再交给 Reviewer 复审；最终形成 AI 已评审的测试用例文档，再交给人类 review。

用户同时补充了测试用例文档的强约束：

- 测试分层只使用 `UT` / `IT`。
- `UT` 使用 Mock Repository，不依赖真实数据库。
- `IT` 使用真实 PostgreSQL / Testcontainers，验证 DDL 与 INSERT 行为。
- 能用 `UT` 验证的不要升级为 `IT`。
- 文档头部必须有用例索引表。
- 每条用例必须说明测什么、为什么、场景数据、覆盖类型。
- 正例和反例必须拆开；反例必须同时包含空场景和错误场景。
- 适合参数化的用例标注“建议 @ParameterizedTest”。
- 文档末尾必须附最小 IT 数据样例，包含半成品行和完整可执行行。
- 不写与测试无关的背景介绍，不把正反向用例合并含糊带过。

调研 ECC 项目后，没有发现主分支中可直接复用的“测试用例文档评审”专用 agent 或 skill。`tdd-guide` 更偏测试代码和 Red-Green-Refactor，`code-reviewer` 更偏代码 diff 审核，`test-coverage` 发生在测试代码之后，`gan-evaluator` 可借鉴“独立评审和反馈循环”的形态，但评审对象是运行中的应用，不是测试用例文档。因此本阶段需要沉淀为独立 skill。

## Decision

新增 `test-case-authoring-review`，采用“Author 编写 + Reviewer 独立审核 + 修订复审”的复合流程。

1. 阶段⑤和阶段⑥合并为一个复合 skill，但内部保留两个角色：
   - Author Agent：负责读取任务计划和上游设计资料，编写测试用例文档。
   - Reviewer Agent：负责独立审核测试用例文档，只输出问题和结论，不参与初稿编写。
2. 阶段⑦保持独立，只接收人类批准后的 `AI_REVIEWED` 测试用例文档，再进入测试代码落地。
3. 输出测试用例文档，默认路径为：

```text
openspec/changes/<change>/test-cases/<spec-domain>-test-cases.md
```

1. 测试用例文档状态从 `DRAFT` 开始，至少经过一轮 Reviewer 审核后，才允许变为 `AI_REVIEWED`。
2. Reviewer 审核循环上限建议为 3 轮。3 轮后仍有 blocking issues 时，停止循环并交给人类裁决。
3. 采用三文件结构：
   - `test-case-authoring-review/SKILL.md`：主 skill 本体，定义准入、边界、Author 编写流程、Reviewer 调度循环、人审交接和禁止越界项。
   - `test-case-authoring-review/REVIEWER.md`：审核 agent 定义，单独描述角色、输入、检查清单、输出格式、通过标准和失败处理。
   - `reference/0005-test-case-authoring-review-design.md`：记录本决策的背景、取舍和影响。
4. `SKILL.md` 不写外部项目引用，直接沉淀为自包含规则；本 ADR 记录来源、调研和取舍。

## Consequences

### Positive

- 人类 review 前先经过 AI 审核闭环，减少明显缺口转嫁给人类。
- Author 和 Reviewer 独立，避免同一个上下文自写自审导致盲点。
- `SKILL.md` 保持流程编排清晰，不被审核细则撑得过长。
- `REVIEWER.md` 可以独立强化审核标准，例如扩展 UT/IT 分层、正反例质量门或最小 IT 数据样例检查。
- 阶段⑦仍保持 TDD 红灯阶段的边界，不把测试用例设计和测试代码落地混在一起。

### Negative

- 相比单次写用例，多了一次或多次 AI 审核调用，耗时和上下文成本更高。
- Reviewer 独立文档会多一个维护点，修改测试用例格式时需要同步检查 `SKILL.md` 和 `REVIEWER.md`。
- 3 轮循环上限只是经验值，复杂业务可能仍需人类提前介入。

## Alternatives Considered

- **保留 ⑤ 和 ⑥ 为两个独立 skill**：职责边界清楚，但容易变成 Author 写完后人工手动切换一次审核，缺少自然的修订复审循环。
- **把 ⑤/⑥/⑦ 合并为一个测试总 skill**：交接成本最低，但会把测试用例设计、AI 审核和测试代码落地混在一起，削弱 TDD 红灯阶段。
- **直接复用 ECC `tdd-guide`**：它适合写测试代码和推动 Red-Green-Refactor，不适合单独审核测试用例文档。
- **直接复用 ECC `code-reviewer`**：它的 blocking issue / verdict 思路可借鉴，但默认审核代码 diff，不适合直接承担测试用例文档评审。
- **把 Reviewer 规则全部写进 `SKILL.md`**：文件更少，但主流程和审核细则混杂，后续强化审核规则会让 skill 本体难以阅读。

## Status

Accepted

## Date

2026-07-10

## References

- `https://github.com/affaan-m/ECC/tree/main`
- `https://raw.githubusercontent.com/affaan-m/ECC/main/agents/tdd-guide.md`
- `https://raw.githubusercontent.com/affaan-m/ECC/main/agents/code-reviewer.md`
- `https://raw.githubusercontent.com/affaan-m/ECC/main/agents/gan-evaluator.md`
- 2026-07-10 用户确认：阶段⑤和阶段⑥应形成 Author / Reviewer 循环，而不是一次性串行交接。
- 2026-07-10 用户确认：Reviewer Agent 的定义应拆成独立文档，不全部写入 `SKILL.md`。

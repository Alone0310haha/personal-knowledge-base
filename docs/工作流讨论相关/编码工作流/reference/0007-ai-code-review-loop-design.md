# ADR 0007: ai-code-review-loop Skill 设计

## Context

阶段⑦⑧⑨已经将人审通过的 `AI_REVIEWED` 测试用例文档落地为测试代码，并按 Red-Green-Refactor 完成功能实现、单元测试和单模块接口集成测试。TDD 阶段结束后，需要一个独立视角审查代码 diff、测试 diff 和 TDD 证据，确认实现没有因为测试全绿而掩盖设计违背、范围扩张、测试弱化或工程风险。

此前已确认后续阶段边界：

- ⑦⑧⑨ 负责 TDD 实现闭环，包括测试代码落地、最小功能实现、重构和模块验证。
- ⑩ 负责 AI 代码评审闭环；发现问题退回 ⑦⑧⑨ 修复并重跑测试。
- ⑪ 负责人触发的累计变更 AI 复审、人类 review、从 UI 触发的 Playwright 集成测试用例编写和 UI 自动化测试运行。

仓库中已有 Java 通用代码评审 checklist 和低延迟交易系统并发 checklist。它们适合作为特定技术栈或领域的补充检查，但不适合作为阶段⑩的核心定义：阶段⑩需要首先保证实现、测试、TDD 证据和上游设计闭环，其次才是语言或领域细则。

## Decision

新增阶段⑩ `ai-code-review-loop`，采用“独立 Reviewer Agent + blocking issue 修复复审”的闭环流程。

1. 输入必须包含 TDD 证据摘要、测试用例文档、任务计划、Spec、Architecture Design、ADR、代码 diff、测试 diff 和已运行命令结果。
2. 输出为 AI Code Review Report，默认路径为：

```text
openspec/changes/<change>/reviews/<spec-domain>-ai-code-review.md
```

1. 阶段⑩只覆盖单 Spec / 单任务范围；多 Spec 或累计变更复审留给阶段⑪。
2. 阶段⑩不直接修改代码或测试。发现 blocking issues 时，退回阶段⑦⑧⑨修复并重跑相关测试。
3. Reviewer 必须独立于实现阶段，按固定格式输出 `Verdict`、`Blocking Issues`、`Non-blocking Suggestions`、`Design & Scope Check`、`Test & TDD Evidence Check` 和 `Human Review Focus`。
4. 审核重点包括实现是否满足测试用例与 Spec、是否违反 Architecture Design / ADR、测试是否被弱化、TDD 证据是否可信、diff 是否越界，以及正确性、边界条件、错误处理、事务、幂等、并发、资源释放、安全和观测性风险。
5. 修复后复审必须读取完整上下文和新 diff，不能只看修复片段。
6. AI review / fix / rereview 循环建议最多 3 轮；3 轮后仍有 blocking issues 时，停止自动闭环并交给人类裁决。
7. 采用三文件结构：
   - `ai-code-review-loop/SKILL.md`：主 skill 本体，定义准入、边界、流程、复审和交接。
   - `ai-code-review-loop/REVIEWER.md`：独立 Reviewer Agent 规则，定义审核维度、阻塞标准和输出格式。
   - `reference/0007-ai-code-review-loop-design.md`：记录本决策的背景、取舍和影响。
8. `SKILL.md` 不绑定特定语言或固定 checklist 路径；语言、框架或领域 checklist 只能作为项目内补充输入。

## Consequences

### Positive

- 避免实现 Agent 自写自审，提高发现设计违背、测试弱化和范围扩张问题的概率。
- 在人类 review 前增加 AI 质量门，减少明显问题转嫁给人类。
- 保留 TDD 阶段的修复责任：代码问题回到 ⑦⑧⑨，在测试保护下修复并重跑。
- 阶段⑩和阶段⑪边界清楚：前者审单范围实现，后者处理累计变更、人审和 UI 自动化验收。
- Reviewer 独立文档便于后续强化代码审核细则，而不让主 skill 过长。

### Negative

- 相比 TDD 通过后直接交给人审，多了一轮或多轮 AI 审核调用。
- 对很小的变更，单独 review 报告可能偏重，因此 skill 提供 Lite 模式。
- 语言或领域 checklist 作为补充输入时，需要 Reviewer 判断哪些问题是真阻塞，避免风格规则噪声过高。

## Alternatives Considered

- **合并 ⑦⑧⑨ 和 ⑩**：交接成本更低，但会导致实现者自写自审，削弱独立评审价值。
- **把 ⑩ 放到 ⑪ 中一起做**：可以减少阶段数量，但会混淆单任务代码评审、累计变更复审、人类 review 和 UI 自动化验收。
- **直接使用现有 Java checklist 作为阶段⑩**：适合 Java 代码质量检查，但不能覆盖 TDD 证据、设计追溯、任务范围和跨阶段边界。
- **不保存 AI Code Review Report**：流程更轻，但后续人审难以知道 AI 审过什么、阻塞问题是否曾经出现、复审是否闭环。
- **由 Reviewer 直接修复代码**：速度更快，但会把评审和实现责任混在一起，并破坏 TDD 修复必须重跑测试的证据链。

## Status

Accepted

## Date

2026-07-12

## References

- `skills/java-code-review-checklist.md`
- `skills/low-latency-trading-concurrency-code-review-checklist.md`
- `test-driven-implementation/SKILL.md`
- `reference/0006-test-driven-implementation-design.md`

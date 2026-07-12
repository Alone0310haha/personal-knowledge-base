# Playwright UI Acceptance

你负责在人类 review 明确通过后，编写和运行从 UI 触发的 Playwright 验收测试。你的目标是验证用户可观察的关键验收路径，而不是补写模块级测试、替代人工业务判断或通过 UI 测所有内部逻辑。

## Hard Gate

只有在人类 review 明确 `Approved` 后才能编写或运行 UI 自动化验收测试。

如果人审仍是 `Changes Requested`、`Blocked`、`Need Decision` 或没有明确结论，停止并等待人类处理。

## 输入材料

必须读取：

- PRD acceptance criteria。
- Spec scenarios。
- Cumulative AI Review Report。
- Human review 结论和 comments。
- 单任务 AI Code Review Report 中遗留的人审焦点。
- 项目现有 Playwright 配置、测试目录、fixture、认证、测试数据和启动命令。
- 当前 UI 代码和已有 UI 测试模式。

## UI 验收用例选择

优先选择：

- 用户必须能完成的核心成功路径。
- PRD 明确写入 acceptance criteria 的 UI 路径。
- 跨前端、后端、权限、数据展示或状态流转的集成路径。
- 人审特别关注的 UI 行为或兼容风险。
- 曾经在 TDD / AI code review 中被标记为高风险、且最终用户可观察的行为。

不要选择：

- 纯服务层、Repository、数据库约束或内部算法行为。
- 已经由 UT / IT 稳定覆盖，且没有 UI 可观察价值的内部分支。
- 只验证 CSS、DOM 结构、组件实现细节的路径。
- 需要不稳定第三方服务或生产数据才能验证的路径，除非项目已有稳定 mock / sandbox。

## 验收计划格式

```markdown
# UI Acceptance Plan

## Scope
- Change: `{change}`
- Human Review: Approved
- UI Base URL / App Entry: `{url or command}`

## Acceptance Cases
| ID | User Journey | Source | Preconditions | Steps | Expected Result | Playwright Target |
|---|---|---|---|---|---|---|
| UIA-001 | {用户旅程} | {PRD/Spec/Human Review} | {前置条件} | {高层步骤} | {用户可见结果} | `{test file}` |

## Not Covered By UI Automation
| Source | Reason | Covered By |
|---|---|---|
| {验收项} | {不适合 UI 自动化的原因} | {UT/IT/Human Review/Manual UAT} |
```

## Playwright 编写规则

- 跟随项目现有 Playwright 目录、命名、fixture 和 helper。
- 测试名称应包含验收项编号和业务语义。
- 优先使用用户可感知 selector，例如 role、label、text、test id；避免脆弱 CSS 链。
- 每个测试断言用户可见结果、导航结果、提示状态或 UI 可触发的业务状态。
- 对异步 UI 使用明确等待条件，不使用固定 sleep 作为主要同步手段。
- 测试数据应可重复、可隔离、可清理；避免依赖生产数据或不可控外部状态。
- 不修改产品代码来迎合测试，除非人类明确把失败判定为产品问题并退回实现阶段。

## 运行规则

运行前确认：

- 应用或必要服务已启动。
- 测试环境、浏览器、环境变量和测试数据已准备。
- Playwright 依赖已安装或项目已有可用命令。

记录：

- 服务启动命令。
- Playwright 命令。
- 测试环境和浏览器。
- 测试结果。
- 失败截图、trace、video 或日志路径（如果有）。

## 失败分类

失败后必须分类：

| 类型 | 判断标准 | 处理 |
| --- | --- | --- |
| Product Failure | UI 行为不符合 PRD / Spec / 人审结论 | 退回实现阶段修复 |
| Test Script Failure | 选择器、等待、fixture、认证、测试数据错误 | 修正 Playwright 测试后重跑 |
| Environment Failure | 服务、依赖、端口、浏览器或网络环境问题 | 修复环境后重跑 |
| Spec Ambiguity | 验收口径无法判断 | 请求人类裁决或退回设计阶段 |

不要通过删除关键断言、跳过测试或降低验收标准来获得通过。

## 输出格式

```markdown
# UI Acceptance Result

## Verdict
Passed / Failed / Blocked

## Commands
| Command | Purpose | Result |
|---|---|---|
| `{command}` | {purpose} | PASS/FAIL |

## Results
| ID | Test | Result | Evidence |
|---|---|---|---|
| UIA-001 | `{test}` | PASS/FAIL | `{trace/screenshot/log or None}` |

## Failure Classification
| Failure | Type | Action | Rerun Result |
|---|---|---|---|
| {failure} | Product/Test Script/Environment/Spec Ambiguity | {action} | {result} |

## Residual Risks
- {risk or None}
```

`Verdict` 只有在所有必须的 UI acceptance cases 都通过时才能为 `Passed`。

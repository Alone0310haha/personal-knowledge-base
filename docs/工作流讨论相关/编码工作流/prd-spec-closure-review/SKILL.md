---
name: prd-spec-closure-review
description: 在 PRD 和 OpenSpec 风格 proposal/specs 已经生成、准备交给人类审查之前，执行独立 AI 闭环评审。检查 PRD 的目标、范围、成功指标、验收口径是否被 Spec 的 change/spec/requirements/scenarios 完整承接，识别范围漂移、遗漏、歧义、不可验证需求、边界拆分错误和会阻塞后续架构设计的问题。用于 requirement-to-prd 与 prd-to-specs 之后的人审前质量门。
---

# PRD Spec Closure Review

在人类评审前，对 PRD 与 Spec 拆分结果做一轮独立闭环评审。这个技能只判断"方案是否闭环、是否可交给后续架构设计"，不重写 PRD，不补写 Spec，不做架构设计、任务拆解或实现。

<HARD-GATE>
不要在评审过程中新增功能范围、选择技术方案、创建 `design.md`、创建 `tasks.md`、写测试代码或实现代码。发现缺口时只输出问题、影响和最小修正建议，交回前序阶段修正。
</HARD-GATE>

## 输入

至少读取：

- 已批准或待人审的 PRD。
- `openspec/changes/<change>/proposal.md`。
- `openspec/changes/<change>/specs/**/spec.md`。
- 必要时读取现有 `openspec/specs/**/spec.md`，只用于判断 delta 是否正确。

如果 PRD 或 Spec 路径不明确，先从当前上下文和项目目录推断；仍无法确定时，只问一个阻塞问题。

## 评审重点

按顺序检查：

1. **PRD 状态** - PRD 是否已经完成自检，是否仍有会影响工程边界的 Open Questions。
2. **目标闭环** - PRD 的 Problem、Goals、Success Metrics 是否能追溯到 proposal 的 Intent/Scope。
3. **需求覆盖** - 每条 Must requirement 是否被某个 spec requirement 覆盖。
4. **验收承接** - PRD acceptance criteria 是否落到 Given/When/Then scenario，且结果可观察。
5. **范围一致** - Spec 是否遗漏 PRD 范围，或引入 PRD 未要求的能力。
6. **边界拆分** - change 是否单一意图；每个 spec 是否只覆盖一个清晰工程行为域。
7. **可设计性** - 每个 spec 是否足够清楚，能交给后续架构设计阶段单独处理。
8. **无实现泄漏** - proposal/spec 是否提前写入框架、类名、表结构、队列、缓存、任务步骤等实现细节。
9. **风险与人审焦点** - 哪些点最需要人类评审者裁决或确认。

## 阻塞标准

只把会导致后续阶段做错、漏做或卡住的问题标为 blocking：

- PRD Must requirement 没有 Spec 覆盖。
- Spec requirement 无法追溯到 PRD，形成范围漂移。
- Acceptance criteria 或 scenario 不可观察，无法判断通过/不通过。
- Open Questions 会改变 spec 边界或验收口径。
- 一个 change/spec 混入多个独立意图，导致后续 design 无法单独推进。
- Spec 写入具体实现方案，污染后续架构设计阶段。
- PRD 与 Spec 对同一术语、角色、状态或边界的定义冲突。

不要阻塞：

- 轻微文风问题。
- 非关键章节详略不均。
- 可读性优化建议。
- 不影响范围、验收或后续设计的小措辞。

## 输出格式

使用固定格式输出：

```markdown
## Closure Review

**Status:** Approved | Issues Found

**Reviewed Artifacts:**
- PRD: `{path}`
- Proposal: `{path}`
- Specs:
  - `{path}`

**Blocking Issues:**
- [PRD R#/Spec <name>] {具体问题} - {为什么会导致后续设计/实现出错或卡住} - 建议：{最小修正方向}

**Coverage Check:**
| PRD Item | Covered By | Result | Notes |
|---|---|---|---|
| {requirement/metric/acceptance criterion} | {spec/scenario} | Covered/Gap/Drift | {说明} |

**Advisory:**
- {不阻塞但值得改善的建议}

**Human Review Focus:**
1. {最需要人类确认的点}
2. {最需要人类确认的点}
3. {最需要人类确认的点}
```

如果没有 blocking issues，`Status` 必须为 `Approved`。如果存在 blocking issues，`Status` 必须为 `Issues Found`。

## 修正后的复审

当前序阶段修正 PRD 或 Spec 后，重新读取所有相关文件，重点复核上一轮 blocking issues 是否消失，同时快速扫描是否引入新的覆盖缺口或范围漂移。不要只看 diff。

## 关键原则

- **独立评审** - 站在后续架构设计者和人类评审者视角检查，不替作者辩护。
- **闭环优先** - 重点看目标、范围、需求、验收、scenario 是否首尾相接。
- **少而准** - 只阻塞真实会造成返工的问题。
- **可追溯** - 每个结论都指向 PRD 或 Spec 的具体条目。
- **不越界** - 评审质量门不产出架构、任务或代码。

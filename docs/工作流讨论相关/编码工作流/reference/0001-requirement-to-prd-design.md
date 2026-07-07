# ADR 0001: requirement-to-prd Skill 设计

## Context

需要一个"从模糊需求到 PRD"的阶段技能，作为整个编码工作流的第一站。评估了两个候选来源：

- **[obra/superpowers 的 `brainstorming`](https://github.com/obra/superpowers/blob/main/skills/brainstorming/SKILL.md)**：发散式框定访谈——一次一问、优先选择题，提出 2-3 个方案并附 trade-off，逐段呈现设计获批准，最终产出设计文档并交接给 `writing-plans`。
- **[mattpocock/skills 的 `grill-with-docs`](https://github.com/mattpocock/skills/tree/main/skills/engineering/grill-with-docs)**：实际是 [`grilling`](https://github.com/mattpocock/skills/blob/main/skills/productivity/grilling/SKILL.md)（逼问节奏：对已有计划做逐层、单线程的追问，直到达成共识）和 [`domain-modeling`](https://github.com/mattpocock/skills/blob/main/skills/engineering/domain-modeling/SKILL.md)（术语纪律：挑战术语不一致、用场景测试边界、产出 ADR 和术语表 `CONTEXT.md`）两个 skill 的组合调用壳。

两者的交互节奏一致——都是单线程、逐题访谈，不是 ECC 的 `council` skill 那种多子代理并行辩论——具备顺序衔接的基础。

## Decision

两个来源都用，顺序衔接，且针对"需求阶段"做了三处改造：

1. **brainstorming 原版"提 2-3 个技术方案" → 改为"提 2-3 个范围/框定选项"**。原版的 trade-off 是工程性的，直接套用会让需求阶段提前锁死技术方案；改造后的 trade-off 是产品/范围性的（做多大范围、优先服务哪类用户）。
2. **grill-with-docs 原版会产出 ADR → 本阶段不产出 ADR**。需求阶段还没有实现方案，架构决策记录没有意义；ADR 的产出点挪到"架构设计"阶段。只保留 `domain-modeling` 里"挑战术语精确度"的机制，把校验对象从架构/领域术语换成需求条款用语（"用户"具体指谁、"成功"用什么指标量化）。
3. **保留两者"一次一问、优先选择题"的访谈节奏**，这是二者能顺畅衔接、不产生风格割裂的关键前提，不能为了"效率"改成一次问多题。

最终顺序：brainstorming 式发散框定 → 起草 PRD 初稿 → grill-with-docs 式逼问收口 → 定稿交接。对应 `../requirement-to-prd/SKILL.md` 里的 Step 1-4。

## Consequences

### Positive
- 同时具备"生成候选方向"（brainstorming 擅长）和"把已选方向问到扎实"（grill-with-docs 擅长）两种能力，单用一个都有明显缺口。
- 逼问阶段顺手把术语校准进 PRD 的 Glossary 表，减少下游阶段因用词歧义返工。

### Negative
- 比单纯用一个固定结构化问卷多花至少一轮访谈的时间成本，对已经很清楚的小需求是浪费（skill 里给了"何时可以精简"的退出条件）。
- 两个来源均为外部仓库，存在随外部仓库更新而内容漂移的风险；引用前应重新核实源文件是否还是这个内容，不要凭这份记录假设外部仓库没变过。

## Alternatives Considered

- **只用固定结构化问卷**（例如按 FRAME/GROUND/DECIDE/GENERATE 四段式提问）：更快，但对高歧义、多方协作的需求，术语校验强度不够，容易把没定义清楚的词直接写进 PRD。
- **只用 grill-with-docs，不做前期发散**：容易过早收敛到用户脑子里的第一个想法，跳过了"是否有更好的范围框定"这一步。
- **用 `council` skill 做需求讨论**：不合适。`council` 的前提是"已经有多个候选方案、需要仲裁选一个"，不负责从零生成候选；且它是多子代理并行辩论，和这里需要的单线程访谈节奏不匹配。

## Status

Accepted

## Date

2026-07-01

## References

- https://github.com/obra/superpowers/blob/main/skills/brainstorming/SKILL.md
- https://github.com/mattpocock/skills/tree/main/skills/engineering/grill-with-docs
- https://github.com/mattpocock/skills/blob/main/skills/productivity/grilling/SKILL.md
- https://github.com/mattpocock/skills/blob/main/skills/engineering/domain-modeling/SKILL.md

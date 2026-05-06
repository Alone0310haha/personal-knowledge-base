

[Master the Perfect ChatGPT Prompt Formula (in just 8 minutes)!](https://www.youtube.com/watch?v=jC4v5AS4RIM)

## 目标：用 6 块积木写出“不再泛泛”的 ChatGPT 提示词

视频给出一套可复用的 Prompt 公式：用 6 个积木把“你脑子里想要的结果”明确地交付给模型，并强调它们有“优先级顺序”。你不必每次都写满 6 项，但要按优先级逐步补齐，直到输出可用。

六块积木：

- Task：任务（你要它做什么）
- Context：上下文（它需要知道什么才能做对）
- Exemplars：示例/框架（让它模仿结构或风格）
- Persona：角色（它应该以什么专业身份思考/表达）
- Format：输出格式（结果长什么样，最好可直接粘贴使用）
- Tone：语气（正式/随和/自信/幽默等）

优先级（从高到低）：Task > Context > Exemplars > Persona > Format > Tone

## 一套照抄可用的“写 Prompt 步骤”

### Step 1：先写 Task（必须有）

写法：

- 以动词开头：Generate / Write / Analyze / Summarize / Categorize
- 只写 1 句话，把交付物说清楚：输出什么、面向谁、达到什么标准

模板：

```text
Task: 用【动词】完成【交付物】，目标是【成功标准】。
```

例子（健身计划）：

```text
Task: 生成一个为期 3 个月的训练计划。
```

### Step 2：补 Context（重要，但只给“刚刚好”的信息）

视频给出 3 个自检问题，用来决定“该给哪些信息”：

1. 用户背景是什么？（能力水平、基础、限制）
2. 什么算成功？（目标指标、期望结果）
3. 用户处在什么环境？（资源、时间、工具、约束）

模板：

```text
Context:
- Background: …
- Success looks like: …
- Environment/constraints: …
```

例子（健身计划补全）：

```text
Context:
- Background: 70kg 男性，目标增肌 5kg
- Success looks like: 3 个月内增肌并保持可持续
- Environment/constraints: 每周只能去健身房 2 次，每次 1 小时
```

### Step 3：加 Exemplars（可选，但能显著提高质量）

你可以用两种“示例”：

- 直接给一个参考文本（让它模仿写法/排版）
- 给一个结构框架（不必给完整示例也有效）

常用框架示例：

- STAR（面试回答）：Situation / Task / Action / Results
- 简历 bullet 结构：I accomplished X by measuring Y that resulted in Z

模板：

```text
Exemplar/Framework:
请严格按【结构/框架】组织输出：…
（或）参考以下示例的表达与排版：…
```

### Step 4：加 Persona（可选，解决“专业默认值”问题）

写法：

- 选一个你希望“随时咨询到”的专家身份
- 也可用知名人物/虚构角色（视频举了 Batman 邮件的例子）

模板：

```text
Persona: 你是一名【角色】，擅长【能力/场景】。
```

### Step 5：锁 Format（强烈推荐，直接降低你的整理成本）

写法要点：

- 先“闭眼想象最终结果长什么样”，再用格式把它锁死
- 常用：表格 / bullet points / email / code block / Markdown（标题层级）
- 校对类任务：要求把改动加粗（便于人工验收）

模板：

```text
Format:
- Output as: Markdown
- Structure: H2 标题分段 + 要点列表
（或）输出为表格，列：A / B / C
```

### Step 6：定 Tone（最后再定，最容易补）

如果你一时想不起形容词，视频给出的技巧是：

- 先让 ChatGPT 给你列 5 个 tone 关键词
- 再把关键词塞回 prompt

模板：

```text
Tone: 使用【关键词1/2/3】的语气，听起来【目标感受】。
```

## 组合成一个“万能骨架”（复制即用）

```text
Task: …

Context:
- Background: …
- Success looks like: …
- Environment/constraints: …

Exemplar/Framework (optional): …
Persona (optional): …

Format: …
Tone: …
```

## 常见错误与解决方案

- 输出很泛、不可用
  - 先补 Task（更具体）+ Context（成功标准/约束），再补 Exemplars
- 输出结构乱、你还得二次整理
  - 强制 Format（表格/Markdown 标题/固定 section）
- 风格不对、语气尴尬
  - 先让它产出 tone 关键词，再把关键词写回 Tone
- 你想让它“像某类专家”思考
  - 用 Persona 指定专家身份（比只说“更专业点”有效）


---
tags:
  - AI编程
  - 提示词
---


[# How to Build a Prompts Database in Notion (template included)!](https://www.youtube.com/watch?v=Oo_GGWV9Hys)

## 目标：在 Notion 搭一个“提示词数据库”，让收集与复用变成习惯

这个系统解决两件事：

- 捕获（Capture）：看到好 prompt 不打断当前工作流，10 秒内存进去。
- 复用（Retrieve）：每周复盘筛选、打标签、沉淀为“随用随取”的提示词库。

## 最终你会得到什么（结构概览）

- 一个 Prompt 数据库（原始表）
- 一个主页（Dashboard）
  - Quick Actions：新建 Prompt 按钮
  - Pinned：常用视图（如 Inbox、Essentials）
  - Ops Center：按个人/业务/工作分类浏览
  - 外部 Prompt Libraries：外部资源链接
  - Archived：归档区

## 1）先把数据库字段设计好（这是系统能不能用起来的关键）

### 必备字段（建议照抄）

- Name：提示词名称（必须“可读可搜”）
- Prompt：提示词正文（建议用 callout/普通文本，便于高亮变量）
- Type：提示词所处状态/归属（视频里的核心字段）
  - captured：刚捕获、未验证
  - personal：个人
  - my business：业务/副业
  - workplace：工作
  - all：通用高频
  - archived：归档
- Use case：用途分类（如 copywriting / business ops / data analysis）
- Output format：输出类型（text / image / video）
- App：适用工具（为空表示通用；仅在某工具好用就填，如 Claude / Midjourney）
- Source：来源链接/出处（URL 或“来自哪封邮件/哪篇文章”）
- Example output（可选）：示例输出（尤其是 image prompt）
- Created time：创建时间
- Last edited time：最后编辑时间（用于排序）

### 默认排序

- Last edited time ↓（让最新加入/更新的 prompt 永远在最上面）

## 2）做主页布局（Quick Actions + Pinned + Ops Center）

### Step A：Quick Actions / Pinned 两栏

1. 在主页插入 Callout（`/callout`），命名为 Quick Actions。
2. 复制一个 Callout 放右侧，命名为 Pinned（两栏布局）。

### Step B：创建“New prompt”按钮（核心：一键捕获）

1. 在主页插入按钮（`/button`），命名 `New prompt`，图标用 `+`。
2. Button action 选择：Add page to → 你的 prompts 数据库。
3. 让新建页面使用预设模板（New prompt template）。
4. 新建时自动把 Type 设置为 `captured`（用于 Inbox 过滤）。
5. 新建后自动在 Side peek 打开新页面（减少跳转成本）。

### Step C：Inbox 视图（只显示 captured）

1. 在主页插入 linked database（`/linked view of database`）。
2. 选择 prompts 数据库的 all view。
3. Layout 选 list，隐藏数据库标题。
4. Properties 只显示 Type。
5. Filter：Type = `captured`。
6. 将这个视图重命名为 `Inbox`（图标用托盘）。

### Step D：Essentials 视图（Pinned 区的高频通用 prompt）

1. 复制 Inbox 视图到 Pinned 区。
2. 重命名为 `Essentials`（图标用魔杖）。
3. Properties 改为显示 `Use case`。
4. Filter：Type = `all`（只保留“全局高频”）。

### Step E：Ops Center（按 personal/business/workplace 分流）

做法：从 all view 复制出 3 个 view，只改 filter 与展示方式。

1. `personal` 视图
   - Layout：gallery
   - Card preview：none，card size：small
   - Properties：只显示 App（便于知道用哪个工具跑）
   - Group：按 Use case 分组
   - Filter：Type = `personal`
2. 复制 `personal` 视图改成 `my business`（Filter：Type = my business）
3. 再复制改成 `workplace`（Filter：Type = workplace）
4. 保留一个 `all` 视图用于全局搜索（放大镜搜索很快定位）

## 3）日常使用：捕获流程（10 秒完成）

1. 看到好 prompt → 全选复制
2. 点击 `New prompt`
3. 填 Name（有意义的短标题）
4. 把 prompt 粘到 Prompt 区（可选：无格式粘贴）
5. 先不填 App（没测试前保持空）
6. Type 自动是 `captured` → 自动进 Inbox

## 4）每周复盘：把 captured 变成可复用资产（核心闭环）

对 Inbox 里每条 captured prompt 做：

1. 测试：在你常用 AI 工具里跑一次，看是否真有用。
2. 迭代：修改前先复制一份作为历史版本（保留原始 prompt）
   - 原始放到 “previous iterations”
   - 当前版变成“latest iteration”
3. 补齐元数据：Use case、Output format、Source
4. 标注适用 App：只在“只对某工具好用”时填（例如 Claude + CSV 文件更稳）
5. Type 分类归档：personal / my business / workplace / all
6. Helpful hints：写两行“使用提醒”（例如输入文件格式、变量需要替换的位置）

## 5）让提示词更好用的两个小技巧

### 技巧 1：变量高亮（比 code block 更直观）

- 在 prompt 里遇到 `{{dataset}}`、`<insert …>` 这种变量，用 Notion 的高亮（视频用快捷键突出显示）标出来。
- 好处：复制到 AI 工具后，你一眼就知道哪些地方必须替换。

### 技巧 2：相似 prompt 合并到同一页（减少数据库膨胀）

- 例如“名字生成器 prompt”和“域名生成器 prompt”做同一类事，可以放同一页面中分块管理。
- 好处：减少重复条目，让数据库更“干净可维护”。

## 6）归档与维护（让系统长期可持续）

- 不再需要的 prompt：Type 改为 `archived`，出现在 Archived 区，不污染日常视图。
- 保持 2 个常规习惯：
  - 每天只做捕获（不纠结分类）
  - 每周一次复盘（分类、测试、迭代、沉淀）


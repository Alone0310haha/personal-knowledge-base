---
tags:
  - 程序员
  - git
---


[原来我，一点都不懂 Git](https://www.bilibili.com/video/BV18ZiFB3EUy/?spm_id_from=333.1365.list.card_archive.click&vd_source=c7bea6fb9f92988f548051c4ed7de1e7)

## 讨论视角

- 从“为什么仓库会莫名其妙变大、为什么删文件也不变小”这种工程痛点切入，用 Git 的底层对象模型解释现象
- 用“对象引用链”的方式把 Git 还原成一个内容寻址（哈希）数据库：对象不可变、靠引用组织结构
- 重点解释 3 类核心对象（commit/tree/blob）以及 branch 只是“引用指针”而非对象

## 核心论点与分论点

### 核心论点

- Git 的本质是一个由哈希定位的对象库；一次提交（commit）通过 tree 记录目录结构，通过 blob 记录文件内容；分支（branch）只是指向某个 commit 的引用。

### 分论点 1：Git 只靠三类对象表达“历史快照”

- **Blob**：文件内容本体（某个文件在某一时刻的内容快照）
- **Tree**：目录结构（记录目录下有哪些条目，以及每个条目指向哪个 blob/tree）
- **Commit**：一次提交（记录元信息 + 指向一个 tree + 可选 parent 指向上一个 commit）

### 分论点 2：Commit → Tree → Blob 的引用链解释了“省空间”的来源

- 每次提交并不把“整个仓库所有文件内容”重新保存一份
- 对于未变化的文件，新 commit 的 tree 仍可复用旧的 blob
- 只有新增/修改的文件才会产生新的 blob（并在新的 tree 中指向它）

### 分论点 3：对象不可变解释了“删了文件仓库也不变小”

- blob 一旦生成就不会被修改或自动删除
- 即使你后续删除文件并提交，新 commit 只是新的 tree 不再引用那个 blob
- 只要对象库里还保留该 blob（哪怕不再被任何引用链触达），它仍占磁盘

### 分论点 4：Parent 让 commit 串成历史；Branch 只是“指针”

- commit 会记录 parent（上一次提交的哈希），由此形成提交链
- branch 不是一种新对象，本质是对某个 commit 的引用
- 切换分支/checkout 可以理解为：把 HEAD/工作区指向另一个 commit

## 关键数据 / 案例 / 结论

- 作者用一个极简仓库演示：
  - 第一次提交：commit 指向 tree，tree 指向包含文件内容的 blob
  - 第二次提交新增文件：新 tree 会指向“旧文件 blob + 新文件 blob”，旧 blob 复用以节省空间
  - 删除文件再提交：tree 不再引用某 blob，但 blob 仍存在于对象库中
- 仓库膨胀的直观原因：大量大文件/模型文件一旦被提交形成 blob，即便后续删除文件，blob 也不会自动消失
- 解决思路（视频给出的方向）：先让这些大 blob 变成“未被引用的悬空对象”，再清理未引用对象（本质是做对象回收）

## 思维导图（Markmap）

```markmap
# Git 的本质（对象模型）
## 核心对象（3 个）
### Blob
#### 文件内容快照
#### 内容寻址（哈希）
#### 不可变
### Tree
#### 目录结构
#### 条目 -> blob/tree 的引用
### Commit
#### 元信息（作者/提交者/message）
#### 指向一个 tree
#### parent -> 上一个 commit
## 引用链
### Commit -> Tree -> Blob
### 新提交复用旧 blob（未变文件）
## 为什么删文件仓库不变小
### 删除+提交 = 新 tree 不再引用旧 blob
### 旧 blob 仍在对象库里（不可变、不会自动删）
### 需要让其成为未引用对象后再做清理/回收
## Branch 是什么
### branch = 指向某个 commit 的引用（指针）
### checkout/切换分支 = 指向不同 commit
```



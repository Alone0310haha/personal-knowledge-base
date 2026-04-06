---
tags:
  - ai总结
  - 信息提炼
---

# MySQL／InnoDB 如何用 Redo Log 做崩溃恢复

[MySQL／InnoDB 如何用 Redo Log 做崩溃恢复](https://www.bilibili.com/video/BV1qb6EB2EGi?spm_id_from=333.788.videopod.episodes&vd_source=c7bea6fb9f92988f548051c4ed7de1e7&p=25)

## 讨论视角（这期在解决什么）

- 以“崩溃恢复/事务不丢数据”的面试题为切入点，解释 InnoDB 如何利用 Redo Log 实现 crash recovery
- 重点围绕：为什么需要 redo、redo 的写入方式（顺序写）、redo log buffer 与刷盘策略对一致性/性能的权衡

## 核心论点与分论点

### 核心论点

InnoDB 更新数据时先改 Buffer Pool（形成脏页），若此时宕机会导致内存中的变更来不及落盘；为避免丢事务，系统把变更记录写入 Redo Log（及其 buffer），崩溃后通过重放 redo 把 Buffer Pool/数据页状态恢复到崩溃前，从而保证“已提交事务”的持久性。

### 分论点 1：没有 redo 会发生什么

- 例子：把记录读入 Buffer Pool，更新字段后提交事务
- 若在脏页刷盘前 MySQL 崩溃：
  - Buffer Pool 中的变更丢失
  - 磁盘页仍是旧数据
  - 结果是这笔已提交变更“看起来丢了”

### 分论点 2：Redo Log 如何实现崩溃恢复

- 更新发生后，把“新的变更信息”写入 Redo Log
- 宕机后启动恢复流程：利用 Redo Log 做重放（redo），把变更恢复到 Buffer Pool/数据页对应位置
- 目标：把系统状态拉回到崩溃前的状态（至少保证已提交事务不丢）

### 分论点 3：为什么不直接把变更立刻写入 ibd

- 视频给出的关键原因：写 redo 的性能更高
  - Redo Log：顺序写（预先开辟一块空间，按顺序追加写，写满后回到开头循环）
  - ibd 文件：数据页在磁盘上分布离散，写入需要寻址（机械盘下尤其明显）
- 因此写 redo 更快 → “暴露在崩溃窗口”的时间更短 → 丢失风险概率更低
- 注意：如果在 redo 还没写入之前就宕机，这部分数据仍可能丢失（只是概率更低）

### 分论点 4：Redo Log Buffer 与刷盘策略（3 种）

- 为进一步提升写入效率，引入 redo log buffer，先写内存 buffer，再按策略刷到 redo log 文件
- 策略（视频口径）：
  - 参数=0：每隔 1 秒由后台线程刷盘（延迟写/延迟刷），可能丢 1 秒内数据
  - 参数=1：提交事务时立刻调用 `fsync` 刷到磁盘（实时写/实时刷），一致性最高（默认）
  - 参数=2：提交时先写入 OS 的 page cache，再由 OS 调度 `fsync` 刷盘
    - MySQL 进程挂了但 OS 不挂：通常不丢
    - 机器掉电/系统崩溃：page cache 未落盘部分会丢
- 结论：强一致场景建议设置为 1（牺牲一定性能换一致性）

## 关键数据、案例与结论（逐条可复述）

- 脏页风险：Buffer Pool 变更未落盘时宕机会丢内存变更
- Redo Log 的价值：用可重放日志支撑崩溃恢复，保证事务持久性
- 写入特性：Redo Log 顺序写，性能优于随机位置写 ibd
- 刷盘策略影响：
  - 0：可能丢 1 秒
  - 1：一致性最高（默认）
  - 2：依赖 OS page cache（进程挂 vs 机器挂的差异）

## 思维导图（markmap）

```markmap
# MySQL/InnoDB 如何用 Redo Log 做崩溃恢复
## 问题背景
### 更新先改 Buffer Pool -> 脏页
### 脏页未刷盘宕机 -> 变更丢失
## Redo Log 的作用
### 记录变更（可重放）
### 崩溃后重放 redo -> 恢复到崩溃前状态
## 为什么要写 redo（而不是只写 ibd）
### redo 顺序写（快）
### ibd 随机位置写（慢，机械盘更明显）
### 写得越快 -> 崩溃窗口越小 -> 丢失风险更低
## Redo Log Buffer
### 先写内存 buffer
### 再按策略刷到 redo log 文件
## 刷盘策略（视频口径）
### 0：每秒刷一次 -> 可能丢 1 秒
### 1：提交即 fsync -> 一致性最高（默认）
### 2：提交写 page cache -> OS 再刷盘
#### MySQL 挂但 OS 不挂：一般不丢
#### 机器掉电：未落盘部分会丢
## 工程建议
### 强一致：设为 1
```


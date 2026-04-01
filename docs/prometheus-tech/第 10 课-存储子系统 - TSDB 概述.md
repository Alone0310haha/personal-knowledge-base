# 第 10 课：存储子系统 - TSDB 概述

**学习时长**：2-3 小时  
**难度等级**：⭐⭐⭐ 进阶  
**先修要求**：完成第 9 课 - 抓取管理（Scrape Manager）

---

## 学习目标

完成本课程后，你将能够：

- ✅ 说清 Prometheus TSDB 在整条链路中的位置（抓取后写入、查询时读取）
- ✅ 理解 TSDB 的核心组件：Head、WAL、Block、Index、Chunks
- ✅ 理解“写入快、查询快、磁盘省”背后的基本做法（追加写、压缩、倒排索引）
- ✅ 能看懂 `data/` 目录结构，并解释每个目录/文件的大致作用
- ✅ 理解保留策略（Retention）与压缩（Compaction）在做什么

---

## 10.1 TSDB 在 Prometheus 里的位置

Prometheus 的核心链路可以简化为：

```
Targets → 抓取（Retrieval）→ 样本（Samples）→ TSDB（本地存储）→ PromQL 查询
```

TSDB 要解决的问题很直观：

- 抓取会持续产生海量样本（每秒写入）
- 查询会频繁扫描/聚合（每秒读取）
- 数据要压缩、要可删除、要可恢复

---

## 10.2 TSDB 存的到底是什么

Prometheus 存的不是“日志”，而是“时间序列样本”：

- **时间序列（Time Series）**：由 `metric_name + labels` 唯一确定
- **样本（Sample）**：某条时间序列在某个时间点的值（timestamp, value）

举例：

- 时间序列：`http_requests_total{job="web",instance="10.0.1.12:8080",method="GET"}`
- 样本：`(1711944000, 12345)`

TSDB 的关键约束：同一条时间序列的样本按时间递增写入。

---

## 10.3 TSDB 的核心结构：Head + WAL + Blocks

你可以用“近期在内存、历史在磁盘”来理解 Prometheus TSDB：

- **Head（内存头块）**：接收最新写入，提供最近时间范围的查询
- **WAL（预写日志）**：写入时先追加到 WAL，用于崩溃恢复
- **Blocks（持久化块）**：把一段时间范围的数据压缩后落盘，供历史查询

简单类比：

- Head：正在写的“工作区”
- WAL：工作区的“操作记录”（可回放）
- Blocks：已经打包归档的“历史包裹”

---

## 10.4 写入路径：样本怎么落到磁盘

一次 scrape 把样本写入 TSDB 时，大致发生这些事：

1) 根据 `metric + labels` 找到对应的 time series（不存在就创建）
2) 把样本追加到 Head 的内存结构
3) 同步把变更写入 WAL（追加写，成本低）

为什么要 WAL：

- 进程崩了，Head 在内存里会丢
- 重启时通过回放 WAL，把 Head 恢复到崩溃前

---

## 10.5 查询路径：PromQL 查询时读哪里

查询通常需要同时读两部分：

- 最近数据：从 Head 读
- 历史数据：从 Blocks 读

PromQL 需要做两件关键事：

1) 先根据标签过滤找到“哪些时间序列”  
2) 再读取这些序列在某个时间范围内的样本做聚合/函数计算

因此 TSDB 需要两类能力：

- **索引能力**：快速从标签找到 series
- **样本存储能力**：高压缩存样本，并能按时间范围高效读取

---

## 10.6 Block 里有什么：Index + Chunks + meta.json

一个 Block 通常包含三类核心内容：

- `index`：索引文件，用于快速定位满足标签条件的 series
- `chunks/`：样本数据（按 chunk 存储，并做压缩）
- `meta.json`：这个 block 的元信息（时间范围、统计信息、版本等）

直觉理解：

- index 解决“找谁”
- chunks 解决“读值”
- meta 解决“这包数据的说明书”

---

## 10.7 压缩与保留：Compaction / Retention

### 10.7.1 Compaction（压缩合并）

Prometheus 会在后台把小块合并成大块：

- 降低 block 数量，减少查询时要扫的文件
- 提升压缩率，减少磁盘占用

### 10.7.2 Retention（保留策略）

Prometheus 会按保留策略删除旧数据，常见两类：

- 按时间保留：例如保留 15d
- 按磁盘大小保留：例如最多 50GB

保留策略的意义：

- TSDB 不是“永久存储”，要么你自己扩容磁盘，要么就必须删旧数据

---

## 10.8 data/ 目录结构：实践必看

Prometheus 本地数据通常在 `--storage.tsdb.path` 指定目录下（默认 `data/`）。

常见结构（不同版本略有差异）：

```
data/
  wal/                  # 预写日志
  chunks_head/          # Head 的 chunk 数据（内存 + 落盘辅助）
  <block_id>/           # 持久化 Block 目录（多个）
    chunks/
    index
    meta.json
  lock                  # 锁文件
```

你不需要记住所有细节，但要记住“看到什么代表什么”：

- `wal/` 变大：写入很活跃
- block 目录变多：历史数据在积累
- compaction 后：小 block 会减少，大 block 会出现

---

## 10.9 实践：观察 TSDB 的变化

目标：用最少动作把 TSDB 的几个核心概念和磁盘文件对应起来。

### 10.9.1 准备一个持续写入的 Prometheus

- 配一个会持续抓取的 job（抓 Prometheus 自己或 node-exporter）
- 保持 Prometheus 运行一段时间，让它积累数据

观察点：

- `data/wal/` 是否持续增长
- 过一段时间后，`data/` 下是否出现 block 目录（长得像一串 ULID）

### 10.9.2 打开一个 block 看三件事

在任意一个 block 目录里，重点看：

- `meta.json`：这个 block 覆盖的时间范围（最直观）
- `index`：存在即可，代表有索引
- `chunks/`：存在即可，代表样本在这里

### 10.9.3 用查询验证“Head + Blocks 一起在工作”

尝试两类查询：

- 查最近 5 分钟：几乎一定命中 Head
- 查更长时间范围：会同时用到 Blocks

---

## 课后小结

- TSDB 是 Prometheus 的“本地时序数据库”，负责写入、查询、压缩、保留与恢复
- 写入先到 Head，同时追加到 WAL；历史数据以 Block 形式落盘
- 查询靠 index 找 series，靠 chunks 取样本；Head 和 Blocks 会一起参与查询
- 读懂 `data/` 目录结构，是后面深入 WAL、Block、Compaction 的基础


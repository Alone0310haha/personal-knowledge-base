# 第 5 课：PromQL 基础查询

**学习时长**：3-4 小时  
**难度等级**：⭐⭐ 基础  
**先修要求**：完成第 4 课 - 时间序列数据模型

---

## 学习目标

完成本课程后，你将能够：

- ✅ 理解即时向量（Instant Vector）和范围向量（Range Vector）的区别
- ✅ 掌握 PromQL 的基本查询语法
- ✅ 熟练使用标签匹配操作符进行过滤
- ✅ 使用聚合操作对数据进行分组统计
- ✅ 在 Expression Browser 中执行各种基础查询
- ✅ 设计合理的查询表达式解决实际问题

---

## 5.1 PromQL 简介

### 5.1.1 什么是 PromQL？

**PromQL**（Prometheus Query Language）是 Prometheus 的查询语言，用于：

- 📊 **查询和展示**时间序列数据
- 🔍 **过滤和聚合**指标
- 📈 **计算速率、趋势**等派生指标
- 🚨 **告警规则**的条件表达式
- 📉 **Grafana 仪表盘**的数据源

**类比理解**：

| 数据库 | 查询语言 | 用途 |
|--------|----------|------|
| MySQL | SQL | 查询关系型数据 |
| Elasticsearch | DSL | 查询文档数据 |
| **Prometheus** | **PromQL** | **查询时间序列数据** |

### 5.1.2 PromQL 的查询场景

**场景 1：即时查询（当前状态）**

```promql
# 当前的 CPU 使用率
node_cpu_seconds_total

# 当前的内存使用量
node_memory_MemUsed_bytes
```

**场景 2：历史趋势（过去一段时间）**

```promql
# 过去 5 分钟的请求速率
rate(http_requests_total[5m])

# 过去 1 小时的错误增长数
increase(http_errors_total[1h])
```

**场景 3：聚合统计（全局视图）**

```promql
# 所有实例的总请求数
sum(http_requests_total)

# 每个服务的平均延迟
avg by (service) (http_request_duration_seconds)
```

**场景 4：告警判断**

```promql
# CPU 使用率超过 80%
node_cpu_seconds_total > 0.8

# 错误率超过 5%
sum(rate(http_errors_total[5m])) / sum(rate(http_requests_total[5m])) > 0.05
```

---

## 5.2 核心概念：即时向量 vs 范围向量

理解这两种向量是掌握 PromQL 的**关键基础**！

### 5.2.1 什么是向量（Vector）？

在 PromQL 中，**向量**是"一组时间序列"的集合。

**类比理解**：

```
单个时间序列 = 一条记录
向量 = 多条记录的集合
```

**示例**：

```promql
# 单个时间序列
http_requests_total{method="POST", service="user-service"}

# 向量（多个时间序列）
http_requests_total  ← 包含所有 method 和 service 的组合
```

---

### 5.2.1.1 指标、时间序列与向量的关系

**常见问题**：指标和向量是什么关系？

**三者关系图**：

```
指标（Metric）
    ↓ 通过标签实例化
时间序列（Time Series）
    ↓ 多个时间序列组成
向量（Vector）
    ↓ PromQL 查询返回
结果集
```

**详细解释**：

| 概念 | 含义 | 例子 | 返回结果 |
|------|------|------|----------|
| **指标** | 逻辑概念，一类数据 | `http_requests_total` | - |
| **时间序列** | 具体实例，带标签 | `http_requests_total{method="POST"}` | 单个序列 |
| **向量** | 多个时间序列的集合 | `http_requests_total` | 多个序列 |

**类比理解**：

**类比 1：Excel 表格**
```
指标 = 表格的名称（如"销售记录表"）
向量 = 查询结果（可以是一行、多行或全部数据）
```

**类比 2：图书馆**
```
指标 = 一本书的名称（如"哈利波特"）
向量 = 图书馆里这本书的所有副本集合
```

**具体示例**：

```promql
# 指标：http_requests_total
# 包含以下时间序列：
    - http_requests_total{method="GET", service="user-service"}
    - http_requests_total{method="POST", service="user-service"}
    - http_requests_total{method="GET", service="order-service"}
    - http_requests_total{method="POST", service="order-service"}
# 这些时间序列的集合 = 向量
```

**在 PromQL 查询中的体现**：

```promql
# 查询指标 http_requests_total
http_requests_total

# 实际返回的是一个向量（多个时间序列）：
[
  {__name__="http_requests_total", method="GET", service="user-service", ...},
  {__name__="http_requests_total", method="POST", service="user-service", ...},
  {__name__="http_requests_total", method="GET", service="order-service", ...},
  ...
]
```

**关键要点**：

1. **指标是"类名"**：`http_requests_total` 是一个逻辑概念
2. **时间序列是"对象实例"**：`http_requests_total{method="POST"}` 是具体实例
3. **向量是"对象集合"**：查询 `http_requests_total` 返回的所有时间序列
4. **所有 PromQL 查询的结果都是向量**（可能包含 0 个、1 个或多个时间序列）

**为什么需要向量这个概念？**

**原因 1：批量操作**
```promql
# 可以对整个向量进行操作
rate(http_requests_total[5m])  ← 一次性计算所有时间序列的速率

# 而不是逐个处理
rate(http_requests_total{method="GET", service="user"}[5m])
rate(http_requests_total{method="POST", service="user"}[5m])
...
```

**原因 2：聚合运算**
```promql
# 可以对向量进行聚合
sum by (method) (http_requests_total)

# 按 method 标签将向量中的多个时间序列合并为更少的时间序列
```

**原因 3：向量匹配**
```promql
# 两个向量可以进行运算
http_requests_total{status="200"} / http_requests_total

# 左边向量除以右边向量（自动按标签匹配）
```

一句话理解：

- 指标是"类名"（如 http_requests_total）
- 时间序列是"对象实例"（如 http_requests_total{method="POST"}）
- 向量是"对象集合"（如查询 http_requests_total 返回的所有时间序列）

---

### 5.2.2 即时向量（Instant Vector）

**定义**：在**某个时间点**的采样值集合。

**通俗理解**：**"快照"** - 在某一瞬间拍下所有时间序列的当前值。

**语法**：

```promql
# 不带时间范围，默认查询当前时刻
http_requests_total
node_memory_MemUsed_bytes
```

**示例**：

```promql
# 查询所有服务的当前请求总数
http_requests_total

# 结果（假设当前时刻有多个时间序列）：
http_requests_total{method="GET", service="user-service"}    = 1523
http_requests_total{method="POST", service="user-service"}   = 842
http_requests_total{method="GET", service="order-service"}   = 2341
http_requests_total{method="POST", service="order-service"}  = 1567
```

**可视化表示**：

```
时间线：
────────────────────────────────────────────────→
         ↓
     当前时刻（即时向量）
         ↓
    拍下所有时间序列的当前值
    
    http_requests_total:
    - {method="GET", service="user"}     → 1523
    - {method="POST", service="user"}    → 842
    - {method="GET", service="order"}    → 2341
    - {method="POST", service="order"}   → 1567
```

**适用场景**：
- ✅ 查询当前状态（如当前内存使用量）
- ✅ Gauge 类型的指标
- ✅ 不需要计算速率的场景

### 5.2.3 范围向量（Range Vector）

**定义**：在**一段时间范围内**的所有采样值集合。

**通俗理解**：**"录像"** - 记录一段时间内所有时间序列的变化过程。

**语法**：

```promql
# 带时间范围 [时长]
http_requests_total[5m]
node_memory_MemUsed_bytes[1h]
```

**示例**：

```promql
# 查询过去 5 分钟的所有请求数据
http_requests_total[5m]

# 结果（每个时间序列包含多个时间点的值）：
http_requests_total{method="GET", service="user"}[5m]:
  - 10:00:00 → 1500
  - 10:01:00 → 1508
  - 10:02:00 → 1515
  - 10:03:00 → 1520
  - 10:04:00 → 1523

http_requests_total{method="POST", service="user"}[5m]:
  - 10:00:00 → 830
  - 10:01:00 → 835
  - 10:02:00 → 838
  - 10:03:00 → 840
  - 10:04:00 → 842
```

**可视化表示**：

```
时间线：
────────────────────────────────────────────────→
    ←─────  过去 5 分钟  ─────→
         ↓                    ↓
      5 分钟前              现在
         ↓
    记录这段时间内的所有数据点
    
    http_requests_total[5m]:
    - {method="GET", service="user"}:
      [1500, 1508, 1515, 1520, 1523]  ← 5 个数据点
    - {method="POST", service="user"}:
      [830, 835, 838, 840, 842]       ← 5 个数据点
```

**适用场景**：
- ✅ 计算速率（`rate()`、`irate()`）
- ✅ 计算增长量（`increase()`）
- ✅ 计算变化量（`delta()`）
- ❌ **不能直接展示**（需要配合函数使用）

### 5.2.4 即时向量 vs 范围向量 对比

| 特性 | 即时向量 | 范围向量 |
|------|----------|----------|
| **语法** | `http_requests_total` | `http_requests_total[5m]` |
| **数据点** | 每个时间序列 **1 个值** | 每个时间序列 **多个值** |
| **比喻** | 快照（拍照） | 录像（摄像） |
| **直接使用** | ✅ 可以直接查询展示 | ❌ 需要配合函数使用 |
| **典型函数** | 无（直接查询） | `rate()`, `increase()`, `delta()` |
| **适用场景** | 当前状态、Gauge | 速率计算、趋势分析 |

### 5.2.5 重要规则

**规则 1：范围向量必须配合函数使用**

```promql
# ✅ 正确：使用 rate() 处理范围向量
rate(http_requests_total[5m])

# ❌ 错误：范围向量不能直接展示
http_requests_total[5m]  ← 会报错
```

**规则 2：即时向量可以直接展示**

```promql
# ✅ 正确：即时向量可以直接查询
http_requests_total

# ✅ 正确：也可以配合聚合函数
sum(http_requests_total)
```

**规则 3：Counter 类型必须用范围向量计算速率**

```promql
# ✅ 正确：Counter 使用 rate() 计算速率
rate(http_requests_total[5m])

# ❌ 不推荐：Counter 直接查询原始值（重启会归零）
http_requests_total
```

---

## 5.3 基本查询语法

### 5.3.1 查询所有时间序列

**语法**：直接写指标名称

```promql
# 查询所有 http_requests_total 的时间序列
http_requests_total
```

**结果**：返回该指标的所有时间序列（所有标签组合）

### 5.3.2 标签过滤

**语法**：`指标名称{标签名="标签值"}`

```promql
# 精确匹配
http_requests_total{method="POST"}

# 多标签过滤（AND 关系）
http_requests_total{method="POST", service="user-service"}
```

**示例**：

```promql
# 只看 POST 请求
http_requests_total{method="POST"}

# 只看生产环境
http_requests_total{env="production"}

# 组合条件
http_requests_total{method="POST", env="production", service="user-service"}
```

### 5.3.3 标签匹配操作符

Prometheus 支持 4 种标签匹配操作符：

| 操作符 | 含义 | 示例 |
|--------|------|------|
| `=` | 等于 | `{method="POST"}` |
| `!=` | 不等于 | `{method!="POST"}` |
| `=~` | 正则匹配 | `{status=~"5.."}` |
| `!~` | 正则不匹配 | `{status!~"2.."}` |

#### 1. 等于匹配（=）

```promql
# 查询 method 为 POST 的时间序列
http_requests_total{method="POST"}

# 多标签等于
http_requests_total{method="POST", status="200"}
```

#### 2. 不等于匹配（!=）

```promql
# 查询 method 不为 POST 的时间序列
http_requests_total{method!="POST"}

# 排除测试环境
http_requests_total{env!="test"}

# 排除成功响应
http_requests_total{status!="200"}
```

#### 3. 正则匹配（=~）

```promql
# 查询所有 5xx 错误
http_requests_total{status=~"5.."}

# 查询所有 4xx 和 5xx 错误
http_requests_total{status=~"[45].."}

# 匹配多个值（OR 关系）
http_requests_total{status=~"200|201|204"}

# 匹配特定模式
http_requests_total{handler=~"/api/.*"}
```

#### 4. 正则不匹配（!~）

```promql
# 排除所有 5xx 错误
http_requests_total{status!~"5.."}

# 排除测试和开发环境
http_requests_total{env!~"test|dev"}

# 排除健康检查接口
http_requests_total{handler!~"/health|/metrics"}
```

### 5.3.4 时间范围选择

**语法**：`[时长]`

```promql
# 过去 5 分钟
http_requests_total[5m]

# 过去 1 小时
http_requests_total[1h]

# 过去 30 秒
http_requests_total[30s]

# 过去 1 天
http_requests_total[1d]
```

**支持的时长单位**：
- `s` - 秒
- `m` - 分钟
- `h` - 小时
- `d` - 天
- `w` - 周

**组合使用**：

```promql
# 1 小时 30 分钟
http_requests_total[1h30m]

# 1 天 6 小时
http_requests_total[1d6h]
```

---

## 5.4 聚合操作

### 5.4.1 什么是聚合？

**聚合**：将多个时间序列合并为更少的时间序列（通常是 1 个）。

**类比理解**：

```
原始数据（4 个时间序列）：
- 北京店销售额：100 万
- 上海店销售额：150 万
- 广州店销售额：120 万
- 深圳店销售额：130 万
       ↓ 聚合（sum）
汇总数据（1 个时间序列）：
- 总销售额：500 万
```

### 5.4.2 聚合语法

**基础语法**：

```promql
聚合函数 (指标)
```

**按标签分组**：

```promql
# 按某些标签分组聚合
聚合函数 by (标签 1, 标签 2) (指标)

# 去除某些标签（保留其他）
聚合函数 without (标签 1, 标签 2) (指标)
```

### 5.4.3 常用聚合函数

| 函数 | 含义 | 示例 |
|------|------|------|
| `sum` | 求和 | `sum(http_requests_total)` |
| `avg` | 平均值 | `avg(http_request_duration_seconds)` |
| `max` | 最大值 | `max(node_cpu_usage)` |
| `min` | 最小值 | `min(node_memory_free)` |
| `count` | 计数（时间序列数量） | `count(http_requests_total)` |
| `stddev` | 标准差 | `stddev(response_time)` |
| `stdvar` | 标准方差 | `stdvar(response_time)` |
| `topk` | 前 k 个 | `topk(5, http_requests_total)` |
| `bottomk` | 后 k 个 | `bottomk(5, http_requests_total)` |

### 5.4.4 按标签分组聚合

**场景 1：按单个标签分组**

```promql
# 按 method 分组，计算每种方法的总请求数
sum by (method) (http_requests_total)

# 结果：
{method="GET"}    = 15230
{method="POST"}   = 8420
{method="PUT"}    = 3210
{method="DELETE"} = 1560
```

**场景 2：按多个标签分组**

```promql
# 按 method 和 service 分组
sum by (method, service) (http_requests_total)

# 结果：
{method="GET", service="user-service"}    = 8230
{method="POST", service="user-service"}   = 4420
{method="GET", service="order-service"}   = 7000
{method="POST", service="order-service"}  = 4000
```

**场景 3：去除某些标签**

```promql
# 去除 instance 和 job 标签，保留其他
sum without (instance, job) (http_requests_total)

# 等价于按所有其他标签分组
```

**场景 4：全局聚合（去除所有标签）**

```promql
# 所有时间序列求和（结果只有 1 个值）
sum(http_requests_total)

# 所有时间序列求平均
avg(http_request_duration_seconds)
```

### 5.4.5 聚合实战示例

**示例 1：计算总请求速率**

```promql
# 所有服务的总请求速率（次/秒）
sum(rate(http_requests_total[5m]))
```

**示例 2：按服务计算平均延迟**

```promql
# 每个服务的平均延迟
avg by (service) (http_request_duration_seconds)
```

**示例 3：找出请求量最高的 5 个接口**

```promql
# Top 5 最忙的接口
topk(5, sum by (handler) (rate(http_requests_total[5m])))
```

**示例 4：计算错误率**

```promql
# 全局错误率
sum(rate(http_requests_total{status=~"5.."}[5m])) 
/ 
sum(rate(http_requests_total[5m]))
```

**示例 5：按环境统计**

```promql
# 每个环境的请求总数
sum by (env) (http_requests_total)
```

---

## 5.5 实践环节

### 实践 5.1：使用 Expression Browser 基础查询

**任务**：
1. 启动 Prometheus（如果尚未启动）
2. 访问 Expression Browser（http://localhost:9090）
3. 执行基础查询练习

**步骤**：

**步骤 1：查询所有指标**

```promql
# 在 Expression Browser 中输入
up

# 观察结果：
# - 有多少个时间序列？
# - 每个时间序列的值是多少？
# - 有哪些标签？
```

**步骤 2：查询 Prometheus 自身的指标**

```promql
# 查询 Prometheus 抓取的样本总数
prometheus_tsdb_head_samples_appended_total

# 观察：
# - 这是 Counter 还是 Gauge？
# - 值是如何变化的？
```

**步骤 3：使用标签过滤**

```promql
# 只看 job="prometheus" 的指标
up{job="prometheus"}

# 排除某些实例
up{instance!="localhost:9090"}
```

**步骤 4：使用正则匹配**

```promql
# 查询所有 prometheus 开头的指标
{__name__=~"prometheus.*"}

# 查询所有 5xx 错误（如果有的话）
{status=~"5.."}
```

### 实践 5.2：即时向量 vs 范围向量

**任务**：
1. 对比即时向量和范围向量的区别
2. 理解函数的使用

**步骤**：

**步骤 1：查询即时向量**

```promql
# 即时向量（可以直接展示）
prometheus_tsdb_head_samples_appended_total

# 观察：
# - 每个时间序列显示几个值？（答案：1 个）
# - 能看到历史变化吗？（答案：不能）
```

**步骤 2：查询范围向量**

```promql
# 范围向量（不能直接展示，会报错）
prometheus_tsdb_head_samples_appended_total[5m]

# 错误信息：Error: no instant vector selected
```

**步骤 3：配合函数使用范围向量**

```promql
# 使用 rate() 处理范围向量
rate(prometheus_tsdb_head_samples_appended_total[5m])

# 观察：
# - 计算的是什么？（答案：每秒增长率）
# - 值的含义是什么？
```

**步骤 4：对比不同时间范围**

```promql
# 过去 1 分钟的速率
rate(prometheus_tsdb_head_samples_appended_total[1m])

# 过去 5 分钟的速率
rate(prometheus_tsdb_head_samples_appended_total[5m])

# 过去 15 分钟的速率
rate(prometheus_tsdb_head_samples_appended_total[15m])

# 观察：
# - 哪个更平滑？
# - 哪个对变化更敏感？
```

### 实践 5.3：标签过滤练习

**任务**：
1. 使用不同的标签匹配操作符
2. 组合多个过滤条件

**步骤**：

**步骤 1：等于过滤**

```promql
# 查询特定 job 的指标
up{job="prometheus"}

# 查询特定 instance 的指标
up{instance="localhost:9090"}
```

**步骤 2：不等于过滤**

```promql
# 排除 prometheus job
up{job!="prometheus"}

# 排除 localhost
up{instance!="localhost:9090"}
```

**步骤 3：正则匹配**

```promql
# 查询所有 node_ 开头的指标
{__name__=~"node_.*"}

# 查询所有 2xx 状态码
{status=~"2.."}

# 查询多个 job
up{job=~"prometheus|node"}
```

**步骤 4：正则不匹配**

```promql
# 排除所有 go_ 开头的指标
{__name__!~"go_.*"}

# 排除成功响应
{status!~"2.."}
```

**步骤 5：组合条件**

```promql
# job 为 prometheus 且 instance 为 localhost
up{job="prometheus", instance="localhost:9090"}

# job 为 prometheus 或 node（使用正则）
up{job=~"prometheus|node"}
```

### 实践 5.4：聚合操作练习

**任务**：
1. 使用不同的聚合函数
2. 按不同维度分组

**步骤**：

**步骤 1：全局聚合**

```promql
# 所有时间序列求和
sum(up)

# 所有时间序列计数
count(up)

# 所有时间序列求平均
avg(prometheus_tsdb_head_samples_appended_total)
```

**步骤 2：按单个标签分组**

```promql
# 按 job 分组求和
sum by (job) (up)

# 按 job 分组计数
count by (job) (up)
```

**步骤 3：按多个标签分组**

```promql
# 按 job 和 instance 分组
sum by (job, instance) (up)
```

**步骤 4：去除标签**

```promql
# 去除 instance 标签
sum without (instance) (up)

# 去除 job 和 instance 标签
sum without (job, instance) (up)
```

**步骤 5：TopK 查询**

```promql
# 找出样本数最多的前 3 个时间序列
topk(3, prometheus_tsdb_head_samples_appended_total)

# 找出样本数最少的后 3 个时间序列
bottomk(3, prometheus_tsdb_head_samples_appended_total)
```

### 实践 5.5：综合查询练习

**场景**：假设你有一个电商系统，包含以下指标：
- `http_requests_total`：HTTP 请求总数（Counter）
- `http_request_duration_seconds`：请求延迟（Histogram）
- `http_connections_active`：活跃连接数（Gauge）

**任务**：

**任务 1：查询当前总请求数**

```promql
# 答案
sum(http_requests_total)
```

**任务 2：计算每秒请求速率**

```promql
# 答案
sum(rate(http_requests_total[5m]))
```

**任务 3：按方法统计请求速率**

```promql
# 答案
sum by (method) (rate(http_requests_total[5m]))
```

**任务 4：计算错误率（5xx 错误占比）**

```promql
# 答案
sum(rate(http_requests_total{status=~"5.."}[5m])) 
/ 
sum(rate(http_requests_total[5m]))
```

**任务 5：找出最慢的 5 个接口**

```promql
# 答案
topk(5, 
  avg by (handler) (
    rate(http_request_duration_seconds_sum[5m])
    /
    rate(http_request_duration_seconds_count[5m])
  )
)
```

**任务 6：统计每个服务的活跃连接数**

```promql
# 答案
sum by (service) (http_connections_active)
```

---

## 5.6 查询优化技巧

### 5.6.1 使用标签索引加速

**技巧**：优先使用标签过滤，减少扫描范围

```promql
# ❌ 慢查询：先查询所有，再过滤
{__name__=~"http_requests.*", method="POST"}

# ✅ 快查询：直接使用标签索引
http_requests_total{method="POST"}
```

### 5.6.2 避免高基数查询

**技巧**：避免返回过多时间序列

```promql
# ❌ 可能返回大量数据
http_requests_total

# ✅ 限制范围
http_requests_total{service="user-service"}

# ✅ 先聚合
sum by (service) (http_requests_total)
```

### 5.6.3 选择合适的时间范围

**技巧**：根据需求选择时间范围

```promql
# 实时监控（短时间范围）
rate(http_requests_total[1m])

# 趋势分析（中等时间范围）
rate(http_requests_total[5m])

# 长期趋势（长时间范围）
rate(http_requests_total[1h])
```

### 5.6.4 使用记录规则预计算

**技巧**：复杂查询使用记录规则

```yaml
# 记录规则
- record: service:http_requests:rate5m
  expr: sum by (service) (rate(http_requests_total[5m]))
```

```promql
# 查询时直接使用预计算结果
service:http_requests:rate5m
```

---

## 5.7 常见问题与排查

### 5.7.1 查询无结果

**问题**：查询返回空结果

**排查步骤**：

**步骤 1：检查指标名称**

```promql
# 使用自动补全确认指标名称
输入前几个字母，查看是否有匹配的指标
```

**步骤 2：检查标签**

```promql
# 先查询所有，确认有哪些标签
http_requests_total

# 再逐步添加过滤条件
http_requests_total{method="POST"}
http_requests_total{method="POST", service="user-service"}
```

**步骤 3：检查时间范围**

```
# 调整时间范围为 "Last 5 minutes"
# 确认数据存在
```

### 5.7.2 查询过慢

**问题**：查询执行时间很长

**排查步骤**：

**步骤 1：查看返回的时间序列数量**

```promql
# 查看有多少个时间序列
count(http_requests_total)
```

**步骤 2：添加标签过滤**

```promql
# 缩小范围
http_requests_total{service="user-service"}
```

**步骤 3：使用聚合**

```promql
# 先聚合再计算
sum by (service) (rate(http_requests_total[5m]))
```

### 5.7.3 范围向量使用错误

**问题**：使用范围向量时报错

**常见错误**：

```promql
# ❌ 错误：范围向量不能直接展示
http_requests_total[5m]
# 错误：Error: no instant vector selected

# ✅ 正确：配合函数使用
rate(http_requests_total[5m])
```

---

## 5.8 最佳实践

### 5.8.1 查询命名规范

**在 Grafana 中**：

```promql
# ✅ 好的别名
{{service}} - {{method}} 请求速率

# ❌ 坏的别名
{job="user-service", method="POST"}
```

### 5.8.2 查询结构优化

```promql
# ✅ 清晰的查询结构
sum by (service) (
  rate(http_requests_total{method="POST"}[5m])
)

# ❌ 难以阅读
sum by (service)(rate(http_requests_total{method="POST"}[5m]))
```

### 5.8.3 避免的查询模式

```promql
# ❌ 避免：返回过多时间序列
{__name__=~".*"}

# ✅ 替代：明确指定指标
http_requests_total

# ❌ 避免：过于复杂的正则
{handler=~"/api/(user|order|payment)/(create|update|delete)/.*"}

# ✅ 替代：简化或使用多个查询
{handler=~"/api/.*"}
```

---

## 5.9 知识检查

### 选择题

1. **以下哪个是即时向量查询？**
   - A. `rate(http_requests_total[5m])`
   - B. `http_requests_total`
   - C. `http_requests_total[5m]`
   - D. `increase(http_errors_total[1h])`

2. **范围向量的语法是？**
   - A. `http_requests_total{method="POST"}`
   - B. `http_requests_total[5m]`
   - C. `sum(http_requests_total)`
   - D. `rate(http_requests_total)`

3. **以下哪个操作符用于正则匹配？**
   - A. `=`
   - B. `!=`
   - C. `=~`
   - D. `!~`

4. **如何按 method 标签分组求和？**
   - A. `sum(http_requests_total) by (method)`
   - B. `sum by (method) (http_requests_total)`
   - C. `sum(method) (http_requests_total)`
   - D. `by (method) sum(http_requests_total)`

5. **以下哪个函数用于计算速率？**
   - A. `increase()`
   - B. `delta()`
   - C. `rate()`
   - D. `avg()`

### 简答题

1. 解释即时向量和范围向量的区别。
2. 为什么范围向量不能直接展示？
3. 列举 4 种标签匹配操作符并说明各自的作用。
4. `sum by (method)` 和 `sum without (instance)` 有什么区别？
5. 如何计算错误率（5xx 错误占比）？

### 实操题

1. **基础查询**：
   - 查询所有 Counter 类型的指标
   - 使用标签过滤查询特定指标
   - 使用正则匹配查询多个指标

2. **聚合练习**：
   - 计算所有服务的总请求速率
   - 按服务分组计算平均延迟
   - 找出请求量最高的 5 个接口

3. **综合应用**：
   - 设计查询计算每个环境的错误率
   - 设计查询找出最慢的 10 个接口
   - 设计查询统计每个团队的资源使用情况

---

## 5.10 延伸阅读

### 必读
- [PromQL 官方文档](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [聚合操作文档](https://prometheus.io/docs/prometheus/latest/querying/operators/#aggregation-operators)
- [向量匹配文档](https://prometheus.io/docs/prometheus/latest/querying/operators/#vector-matching)

### 选读
- [范围向量详解](https://prometheus.io/docs/prometheus/latest/querying/operators/#range-vectors)
- [查询优化指南](https://prometheus.io/docs/prometheus/latest/querying/best_practices/)
- [PromQL 示例](https://prometheus.io/docs/prometheus/latest/querying/examples/)

### 工具
- [Prometheus Expression Browser](http://localhost:9090/graph)
- [PromQL 在线测试工具](https://promql.io/)

---

## 5.11 下节预告

**第 6 课：PromQL 进阶操作**

在下一课中，你将：
- 学习数学运算和比较操作
- 掌握时间范围函数（`rate`、`irate`、`increase`）
- 深入学习聚合函数的高级用法
- 了解子查询的语法和场景
- 编写复杂的查询表达式并绘制图表

**预习任务**：
- 复习本节课的向量概念
- 思考如何计算"过去 5 分钟的平均值"
- 了解 `rate()` 和 `irate()` 的区别

---

## 总结

通过本课学习，你应该已经掌握：

✅ 即时向量（Instant Vector）的概念和使用  
✅ 范围向量（Range Vector）的概念和使用  
✅ PromQL 的基本查询语法  
✅ 4 种标签匹配操作符（=、!=、=~、!~）  
✅ 常用聚合函数（sum、avg、max、min、count 等）  
✅ 按标签分组聚合的方法（by 和 without）  

**关键收获**：
- 理解 PromQL 查询的核心概念（向量）
- 能够编写基础的查询表达式
- 掌握标签过滤和聚合的方法
- 为学习进阶 PromQL 打下坚实基础

**下一步**：
- 完成实践环节的所有练习
- 在 Grafana 中尝试创建简单的仪表盘
- 准备学习第 6 课的 PromQL 进阶操作

---

**学习提示**：PromQL 是 Prometheus 的核心技能，需要多练习才能熟练掌握。建议在实际环境中多尝试不同的查询，观察结果的变化，这样才能真正理解每个操作符和函数的作用。

祝你学习愉快！📚

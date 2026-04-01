# 第 6 课：PromQL 进阶操作

**学习时长**：4-5 小时  
**难度等级**：⭐⭐⭐ 进阶  
**先修要求**：完成第 5 课 - PromQL 基础查询

---

## 学习目标

完成本课程后，你将能够：

- ✅ 熟练使用数学运算和比较操作符
- ✅ 掌握时间范围函数（rate、irate、increase、delta）
- ✅ 深入理解聚合函数的高级用法
- ✅ 掌握子查询的语法和应用场景
- ✅ 编写复杂的查询表达式解决实际问题
- ✅ 在 Grafana 中绘制专业的监控图表

---

## 6.1 数学运算与比较操作

### 6.1.1 二元运算符

Prometheus 支持多种二元运算符，用于对向量进行数学计算。

**算术运算符**：

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `+` | 加法 | `a + b` |
| `-` | 减法 | `a - b` |
| `*` | 乘法 | `a * b` |
| `/` | 除法 | `a / b` |
| `%` | 取模 | `a % b` |
| `^` | 幂运算 | `a ^ b` |

**示例**：

```promql
# 计算内存使用率
node_memory_MemUsed_bytes / node_memory_MemTotal_bytes * 100

# 计算磁盘剩余空间
node_filesystem_size_bytes - node_filesystem_free_bytes

# 计算请求延迟的平方
(http_request_duration_seconds ^ 2)
```

**比较运算符**：

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `==` | 等于 | `a == b` |
| `!=` | 不等于 | `a != b` |
| `>` | 大于 | `a > b` |
| `<` | 小于 | `a < b` |
| `>=` | 大于等于 | `a >= b` |
| `<=` | 小于等于 | `a <= b` |

**示例**：

```promql
# 找出 CPU 使用率超过 80% 的实例
node_cpu_usage > 0.8

# 找出内存使用率低于 20% 的实例
node_memory_usage < 0.2

# 找出状态码为 200 的请求
http_requests_total{status=="200"}
```

**逻辑运算符**：

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `and` | 与 | `a and b` |
| `or` | 或 | `a or b` |
| `unless` | 除非 | `a unless b` |

**示例**：

```promql
# CPU 高且内存高的实例
node_cpu_usage > 0.8 and node_memory_usage > 0.8

# CPU 高或内存高的实例
node_cpu_usage > 0.8 or node_memory_usage > 0.8

# CPU 高但不在维护期的实例
node_cpu_usage > 0.8 unless maintenance_mode == 1
```

### 6.1.2 向量匹配

当两个向量进行运算时，Prometheus 会自动进行**向量匹配**。

**匹配规则**：

1. 默认情况下，只匹配**标签完全相同**的时间序列
2. 可以使用 `on` 或 `ignoring` 指定匹配条件

**示例 1：默认匹配**

```promql
# 只有标签完全相同的才会匹配
http_requests_total{status="200"} / http_requests_total

# 左边：{method="GET", service="user", status="200"}
# 右边：{method="GET", service="user"}
# ❌ 不匹配（标签不同）
```

**示例 2：使用 on 指定匹配标签**

```promql
# 只根据 method 和 service 标签匹配
http_requests_total{status="200"} 
/ 
on (method, service) 
http_requests_total

# 左边：{method="GET", service="user", status="200"}
# 右边：{method="GET", service="user"}
# ✅ 匹配（method 和 service 相同）
```

**示例 3：使用 ignoring 忽略某些标签**

```promql
# 忽略 status 标签进行匹配
http_requests_total{status="200"} 
/ 
ignoring (status) 
http_requests_total

# 左边：{method="GET", service="user", status="200"}
# 右边：{method="GET", service="user"}
# ✅ 匹配（忽略 status 后标签相同）
```

**示例 4：使用 group_left 和 group_right**

```promql
# 一对多匹配（左边 1 个，右边多个）
node_cpu_usage 
* 
on (instance) 
group_left (node_name) 
node_meta_info

# 将 node_meta_info 的 node_name 标签添加到结果中
```

### 6.1.3 标量运算

**标量（Scalar）**：单个数值，不是向量。

**示例**：

```promql
# 将百分比转换为小数
node_memory_usage_percent / 100

# 将秒转换为毫秒
http_request_duration_seconds * 1000

# 计算增长率（乘以 100 得到百分比）
rate(http_requests_total[5m]) * 100
```

---

## 6.2 时间范围函数

时间范围函数是 PromQL 最强大的功能之一，用于处理 Counter 类型指标和分析趋势。

### 6.2.1 rate() - 计算瞬时速率

**定义**：计算时间序列在指定时间范围内的**每秒平均增长率**。

**语法**：
```promql
rate(范围向量)
```

**特点**：
- ✅ 自动处理 Counter 归零
- ✅ 返回每秒增长率
- ✅ 结果可能为小数
- ⚠️ 只能用于 Counter 类型

**示例**：

```promql
# 每秒请求数（requests per second）
rate(http_requests_total[5m])

# 每秒错误数
rate(http_errors_total[5m])

# 每秒处理的字节数
rate(network_bytes_total[5m])
```

**实际效果**：

```
时间线数据：
10:00  Counter = 1000
10:05  Counter = 1500

rate(http_requests_total[5m]) = (1500 - 1000) / 300 秒 = 1.67/s
```

**使用场景**：
- ✅ 计算 QPS（每秒查询数）
- ✅ 计算错误率
- ✅ 计算网络吞吐量
- ✅ 绘制速率图表

### 6.2.2 irate() - 计算瞬时速率（更敏感）

**定义**：计算时间序列在指定时间范围内的**最后两个点之间的增长率**。

**语法**：
```promql
irate(范围向量)
```

**特点**：
- ✅ 更敏感，反映最新变化
- ✅ 自动处理 Counter 归零
- ⚠️ 波动更大，不适合聚合
- ⚠️ 只能用于 Counter 类型

**示例**：

```promql
# 最新的每秒请求数（更敏感）
irate(http_requests_total[5m])

# 最新的每秒错误数
irate(http_errors_total[5m])
```

**rate() vs irate() 对比**：

```
时间线数据：
10:00  Counter = 1000
10:01  Counter = 1200
10:02  Counter = 1100  ← 归零
10:03  Counter = 100
10:04  Counter = 300
10:05  Counter = 500

rate(http_requests_total[5m]):
- 使用所有点计算平均速率
- 结果：(500 - 1000 + 归零补偿) / 300 秒 = 平滑的值

irate(http_requests_total[5m]):
- 只使用最后两个点（10:04 和 10:05）
- 结果：(500 - 300) / 60 秒 = 更敏感的值
```

**使用建议**：

| 场景 | 推荐函数 |
|------|----------|
| 监控告警 | `rate()`（更稳定） |
| 实时仪表盘 | `irate()`（更敏感） |
| 聚合查询 | `rate()`（避免波动） |
| 调试问题 | `irate()`（发现瞬时变化） |

### 6.2.3 increase() - 计算增长总量

**定义**：计算时间序列在指定时间范围内的**增长总量**。

**语法**：
```promql
increase(范围向量)
```

**特点**：
- ✅ 自动处理 Counter 归零
- ✅ 返回总增长数（不是速率）
- ✅ 等价于 `rate() * 时间范围（秒）`
- ⚠️ 只能用于 Counter 类型

**示例**：

```promql
# 过去 5 分钟的总请求数
increase(http_requests_total[5m])

# 过去 1 小时的总错误数
increase(http_errors_total[1h])

# 过去 24 小时的总用户注册数
increase(user_registrations_total[24h])
```

**实际效果**：

```
时间线数据：
10:00  Counter = 1000
10:05  Counter = 1500

increase(http_requests_total[5m]) = 1500 - 1000 = 500
rate(http_requests_total[5m]) * 300 = 1.67 * 300 = 500
```

**使用场景**：
- ✅ 统计过去 N 分钟的总请求数
- ✅ 统计过去 N 小时的总错误数
- ✅ 生成日报、周报

### 6.2.4 delta() - 计算变化量

**定义**：计算时间序列在指定时间范围内的**变化量**（终值 - 初值）。

**语法**：
```promql
delta(范围向量)
```

**特点**：
- ❌ **不**处理 Counter 归零
- ✅ 可用于 Gauge 类型
- ✅ 可正可负

**示例**：

```promql
# 过去 5 分钟内存变化量（可正可负）
delta(node_memory_MemUsed_bytes[5m])

# 过去 1 小时温度变化量
delta(temperature_celsius[1h])

# 过去 5 分钟队列长度变化
delta(queue_length[5m])
```

**delta() vs increase()**：

| 特性 | delta() | increase() |
|------|---------|------------|
| **处理归零** | ❌ 不处理 | ✅ 处理 |
| **适用类型** | Gauge | Counter |
| **返回值** | 可正可负 | 总是非负 |
| **典型场景** | 内存变化、温度变化 | 请求数、错误数 |

### 6.2.5 idelta() - 计算瞬时变化量

**定义**：计算时间序列在指定时间范围内的**最后两个点之间的变化量**。

**语法**：
```promql
idelta(范围向量)
```

**特点**：
- ❌ **不**处理 Counter 归零
- ✅ 只使用最后两个点
- ✅ 更敏感

**示例**：

```promql
# 最新的内存变化量（只使用最后两个点）
idelta(node_memory_MemUsed_bytes[5m])
```

### 6.2.6 时间范围函数对比总结

| 函数 | 含义 | 处理归零 | 适用类型 | 返回值 |
|------|------|----------|----------|--------|
| `rate()` | 每秒平均速率 | ✅ | Counter | 非负小数 |
| `irate()` | 最后两点速率 | ✅ | Counter | 非负小数 |
| `increase()` | 总增长量 | ✅ | Counter | 非负整数 |
| `delta()` | 变化量 | ❌ | Gauge | 可正可负 |
| `idelta()` | 最后两点变化 | ❌ | Gauge | 可正可负 |

**选择指南**：

```
问题 1：指标类型是什么？
  ↓ Counter → 使用 rate()、irate() 或 increase()
  ↓ Gauge → 使用 delta() 或 idelta()

问题 2：需要速率还是总量？
  ↓ 速率 → rate() 或 irate()
  ↓ 总量 → increase()

问题 3：需要平滑还是敏感？
  ↓ 平滑（告警） → rate()
  ↓ 敏感（实时监控） → irate()
```

---

## 6.3 聚合函数的高级用法

### 6.3.1 常用聚合函数回顾

| 函数 | 含义 | 示例 |
|------|------|------|
| `sum` | 求和 | `sum(http_requests_total)` |
| `avg` | 平均值 | `avg(http_request_duration_seconds)` |
| `max` | 最大值 | `max(node_cpu_usage)` |
| `min` | 最小值 | `min(node_memory_free)` |
| `count` | 计数 | `count(up)` |
| `stddev` | 标准差 | `stddev(response_time)` |
| `stdvar` | 标准方差 | `stdvar(response_time)` |
| `topk` | 前 k 个 | `topk(5, http_requests_total)` |
| `bottomk` | 后 k 个 | `bottomk(5, http_requests_total)` |
| `quantile` | 分位数 | `quantile(0.95, response_time)` |

### 6.3.2 分组聚合

**按单个标签分组**：

```promql
# 按 service 分组求和
sum by (service) (http_requests_total)

# 按 instance 分组求平均
avg by (instance) (node_cpu_usage)
```

**按多个标签分组**：

```promql
# 按 service 和 method 分组
sum by (service, method) (http_requests_total)

# 按 environment 和 team 分组
avg by (environment, team) (response_time)
```

**使用 without 去除标签**：

```promql
# 去除 instance 标签，保留其他
sum without (instance) (http_requests_total)

# 去除 job 和 instance 标签
sum without (job, instance) (http_requests_total)
```

### 6.3.3 嵌套聚合

**场景**：先按一个维度聚合，再按另一个维度聚合

```promql
# 先按 instance 求和，再按 service 求平均
avg by (service) (
  sum by (service, instance) (http_requests_total)
)

# 先计算每个实例的速率，再求和
sum (
  rate(http_requests_total[5m])
)
```

**示例：计算每个服务的 P95 延迟**

```promql
# 先按 service 和 le 分组计算速率，再计算 P95
histogram_quantile(0.95, 
  sum by (service, le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

### 6.3.4 条件聚合

**场景**：只对满足条件的数据进行聚合

```promql
# 只统计 CPU 使用率超过 50% 的实例
sum (node_cpu_usage > 0.5)

# 只统计生产环境的请求数
sum (http_requests_total{env="production"})

# 只统计非 200 状态码的请求
sum (http_requests_total{status!="200"})
```

### 6.3.5 高级聚合示例

**示例 1：计算错误率**

```promql
# 全局错误率
sum(rate(http_requests_total{status=~"5.."}[5m])) 
/ 
sum(rate(http_requests_total[5m]))

# 按服务分组的错误率
sum by (service) (rate(http_requests_total{status=~"5.."}[5m])) 
/ 
sum by (service) (rate(http_requests_total[5m]))
```

**示例 2：计算可用性**

```promql
# 系统可用性（UP 的实例占比）
count(up == 1) / count(up) * 100

# 按服务分组的可用性
sum by (service) (up == 1) / sum by (service) (up) * 100
```

**示例 3：计算资源利用率**

```promql
# CPU 平均利用率
avg(node_cpu_usage)

# 内存利用率（加权平均）
sum(node_memory_used_bytes) / sum(node_memory_total_bytes) * 100

# 磁盘利用率
sum(node_disk_used_bytes) / sum(node_disk_total_bytes) * 100
```

**示例 4：Top N 分析**

```promql
# 请求量最高的 5 个服务
topk(5, sum by (service) (rate(http_requests_total[5m])))

# 延迟最高的 5 个接口
topk(5, 
  avg by (handler) (
    rate(http_request_duration_seconds_sum[5m])
    /
    rate(http_request_duration_seconds_count[5m])
  )
)

# 错误数最多的 5 个服务
topk(5, sum by (service) (rate(http_errors_total[1h])))
```

---

## 6.4 子查询

### 6.4.1 什么是子查询？

**子查询**：在一个查询内部嵌套另一个查询。

**语法**：
```promql
聚合函数 (子查询 [时间范围：分辨率])
```

**格式**：
```promql
查询 [时间范围：分辨率]
```

### 6.4.2 子查询的用途

**用途 1：在更长的时间范围内聚合**

```promql
# 过去 1 小时中，每 5 分钟的平均请求速率的最大值
max_over_time(
  sum(rate(http_requests_total[5m]))[1h:5m]
)

# 解释：
# 1. 先计算每 5 分钟的速率：sum(rate(http_requests_total[5m]))
# 2. 在过去 1 小时内，每 5 分钟采样一次：[1h:5m]
# 3. 取最大值：max_over_time()
```

**用途 2：降采样（Downsampling）**

```promql
# 过去 24 小时，每 1 小时采样一次平均值
avg_over_time(
  node_cpu_usage[24h:1h]
)
```

**用途 3：复杂的时间范围分析**

```promql
# 过去 7 天，每天的平均错误率
avg_over_time(
  (
    sum(rate(http_errors_total[5m])) 
    / 
    sum(rate(http_requests_total[5m]))
  )[7d:1d]
)
```

### 6.4.3 子查询的分辨率

**分辨率**：子查询采样的间隔。

**示例**：

```promql
# 时间范围：1 小时，分辨率：5 分钟
[1h:5m]  ← 采样 12 个点（60/5=12）

# 时间范围：24 小时，分辨率：1 小时
[24h:1h]  ← 采样 24 个点

# 时间范围：7 天，分辨率：1 天
[7d:1d]  ← 采样 7 个点
```

**默认分辨率**：如果不指定，使用 `step` 参数（由 Grafana 或 API 调用决定）。

### 6.4.4 子查询配合聚合函数

**常用组合**：

```promql
# 最大值
max_over_time(查询 [时间范围：分辨率])

# 最小值
min_over_time(查询 [时间范围：分辨率])

# 平均值
avg_over_time(查询 [时间范围：分辨率])

# 求和
sum_over_time(查询 [时间范围：分辨率])

# 计数
count_over_time(查询 [时间范围：分辨率])

# 标准差
stddev_over_time(查询 [时间范围：分辨率])
```

**示例**：

```promql
# 过去 1 小时中，每 5 分钟的平均 CPU 使用率的最大值
max_over_time(
  avg(node_cpu_usage)[1h:5m]
)

# 过去 24 小时中，每 1 小时的总请求数的平均值
avg_over_time(
  sum(rate(http_requests_total[5m]))[24h:1h]
)
```

### 6.4.5 子查询实战示例

**示例 1：找出最繁忙的 1 小时**

```promql
# 过去 24 小时中，哪 1 小时的请求量最大
max_over_time(
  sum(increase(http_requests_total[1h]))[24h:1h]
)
```

**示例 2：计算每日峰值**

```promql
# 过去 7 天，每天的请求峰值
max_over_time(
  sum(rate(http_requests_total[5m]))[7d:1d]
)
```

**示例 3：异常检测**

```promql
# 当前速率是否超过过去 7 天平均值的 2 倍
sum(rate(http_requests_total[5m])) 
> 
2 * avg_over_time(
  sum(rate(http_requests_total[5m]))[7d:1d]
)
```

---

## 6.5 实践环节

### 实践 6.1：数学运算练习

**任务**：使用数学运算计算各种指标

**步骤**：

**步骤 1：计算百分比**

```promql
# 内存使用率
node_memory_MemUsed_bytes / node_memory_MemTotal_bytes * 100

# 磁盘使用率
node_disk_used_bytes / node_disk_total_bytes * 100

# CPU 使用率（所有核心的平均）
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100
```

**步骤 2：使用比较运算符**

```promql
# 找出内存使用率超过 80% 的实例
node_memory_MemUsed_bytes / node_memory_MemTotal_bytes > 0.8

# 找出磁盘剩余空间小于 10GB 的实例
node_disk_free_bytes < 10 * 1024 * 1024 * 1024
```

**步骤 3：使用逻辑运算符**

```promql
# CPU 和内存都超过 80% 的实例
(node_cpu_usage > 0.8) and (node_memory_usage > 0.8)

# CPU 高或内存高的实例
(node_cpu_usage > 0.8) or (node_memory_usage > 0.8)
```

### 实践 6.2：时间范围函数练习

**任务**：熟练使用 rate、irate、increase、delta

**步骤**：

**步骤 1：计算速率**

```promql
# 每秒请求数（使用 rate）
rate(http_requests_total[5m])

# 每秒请求数（使用 irate，更敏感）
irate(http_requests_total[5m])

# 对比两者的差异
# 观察：irate 波动更大，rate 更平滑
```

**步骤 2：计算总量**

```promql
# 过去 5 分钟的总请求数
increase(http_requests_total[5m])

# 过去 1 小时的总错误数
increase(http_errors_total[1h])

# 过去 24 小时的总用户注册数
increase(user_registrations_total[24h])
```

**步骤 3：计算变化量**

```promql
# 过去 5 分钟内存变化量
delta(node_memory_MemUsed_bytes[5m])

# 过去 1 小时温度变化量
delta(temperature_celsius[1h])

# 对比 increase 和 delta
# increase 总是非负，delta 可正可负
```

### 实践 6.3：聚合函数高级用法

**任务**：使用嵌套聚合和条件聚合

**步骤**：

**步骤 1：嵌套聚合**

```promql
# 先按 instance 求和，再按 service 求平均
avg by (service) (
  sum by (service, instance) (http_requests_total)
)

# 先计算速率，再求和
sum (
  rate(http_requests_total[5m])
)
```

**步骤 2：条件聚合**

```promql
# 只统计生产环境的请求数
sum (http_requests_total{env="production"})

# 只统计错误请求（非 2xx）
sum (http_requests_total{status!~"2.."})

# 只统计 CPU 使用率超过 50% 的实例
sum (node_cpu_usage > 0.5)
```

**步骤 3：Top N 分析**

```promql
# 请求量最高的 5 个服务
topk(5, sum by (service) (rate(http_requests_total[5m])))

# 延迟最高的 5 个接口
topk(5, 
  avg by (handler) (
    rate(http_request_duration_seconds_sum[5m])
    /
    rate(http_request_duration_seconds_count[5m])
  )
)

# 错误率最高的 5 个服务
topk(5, 
  sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum by (service) (rate(http_requests_total[5m]))
)
```

### 实践 6.4：子查询练习

**任务**：使用子查询进行时间范围分析

**步骤**：

**步骤 1：基础子查询**

```promql
# 过去 1 小时，每 5 分钟的平均 CPU 使用率
avg_over_time(
  node_cpu_usage[1h:5m]
)

# 过去 24 小时，每 1 小时的平均请求速率
avg_over_time(
  sum(rate(http_requests_total[5m]))[24h:1h]
)
```

**步骤 2：子查询配合聚合**

```promql
# 过去 7 天，每天的请求峰值
max_over_time(
  sum(rate(http_requests_total[5m]))[7d:1d]
)

# 过去 24 小时，每 1 小时的最大内存使用率
max_over_time(
  avg(node_memory_usage)[24h:1h]
)
```

**步骤 3：异常检测**

```promql
# 当前请求速率是否超过过去 7 天平均值的 2 倍
sum(rate(http_requests_total[5m])) 
> 
2 * avg_over_time(
  sum(rate(http_requests_total[5m]))[7d:1d]
)

# 当前错误率是否超过过去 24 小时平均值的 3 倍
(sum(rate(http_errors_total[5m])) / sum(rate(http_requests_total[5m])))
>
3 * avg_over_time(
  sum(rate(http_errors_total[5m])) / sum(rate(http_requests_total[5m]))
  [24h:1h]
)
```

### 实践 6.5：综合查询练习

**场景**：电商系统监控仪表盘

**任务**：为电商系统设计完整的监控查询

**步骤**：

**步骤 1：核心业务指标**

```promql
# 1. 总请求速率（QPS）
sum(rate(http_requests_total[5m]))

# 2. 错误率（5xx 占比）
sum(rate(http_requests_total{status=~"5.."}[5m])) 
/ 
sum(rate(http_requests_total[5m])) * 100

# 3. 平均响应时间
sum(rate(http_request_duration_seconds_sum[5m])) 
/ 
sum(rate(http_request_duration_seconds_count[5m]))

# 4. P95 响应时间
histogram_quantile(0.95, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)
```

**步骤 2：按服务分组**

```promql
# 5. 每个服务的请求速率
sum by (service) (rate(http_requests_total[5m]))

# 6. 每个服务的错误率
sum by (service) (rate(http_requests_total{status=~"5.."}[5m])) 
/ 
sum by (service) (rate(http_requests_total[5m])) * 100

# 7. 每个服务的 P95 延迟
histogram_quantile(0.95, 
  sum by (service, le) (rate(http_request_duration_seconds_bucket[5m]))
)
```

**步骤 3：资源监控**

```promql
# 8. CPU 使用率（按服务）
avg by (service) (node_cpu_usage)

# 9. 内存使用率（按服务）
avg by (service) (node_memory_usage)

# 10. 磁盘使用率（按实例）
avg by (instance) (node_disk_usage)
```

**步骤 4：趋势分析**

```promql
# 11. 过去 24 小时的请求趋势（每 1 小时）
avg_over_time(
  sum(rate(http_requests_total[5m]))[24h:1h]
)

# 12. 过去 7 天的每日峰值
max_over_time(
  sum(rate(http_requests_total[5m]))[7d:1d]
)

# 13. 周环比增长
(
  sum(rate(http_requests_total[5m])) 
  - 
  sum(rate(http_requests_total[5m] offset 7d))
) 
/ 
sum(rate(http_requests_total[5m] offset 7d)) * 100
```

---

## 6.6 查询优化最佳实践

### 6.6.1 避免高基数查询

```promql
# ❌ 避免：返回过多时间序列
{__name__=~".*"}

# ✅ 替代：明确指定指标
http_requests_total

# ✅ 替代：先聚合
sum by (service) (http_requests_total)
```

### 6.6.2 使用记录规则预计算

```yaml
# 记录规则
- record: service:http_requests:rate5m
  expr: sum by (service) (rate(http_requests_total[5m]))

- record: service:http_errors:ratio5m
  expr: |
    sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
    /
    sum by (service) (rate(http_requests_total[5m]))
```

```promql
# 查询时直接使用
service:http_requests:rate5m
service:http_errors:ratio5m
```

### 6.6.3 选择合适的时间范围

```promql
# 实时监控（短时间范围）
rate(http_requests_total[1m])

# 趋势分析（中等时间范围）
rate(http_requests_total[5m])

# 长期趋势（长时间范围）
rate(http_requests_total[1h])
```

### 6.6.4 使用向量匹配优化

```promql
# ❌ 慢：标签不完全匹配
http_requests_total{status="200"} / http_requests_total

# ✅ 快：使用 on 指定匹配标签
http_requests_total{status="200"} 
/ 
on (method, service, instance)
http_requests_total
```

---

## 6.7 常见问题与排查

### 6.7.1 rate() 返回负值

**问题**：rate() 返回负数

**原因**：Counter 被重置（非正常归零）

**解决方案**：
- 检查应用是否重启
- 使用 `increase()` 查看增长量
- 检查 Counter 实现是否正确

### 6.7.2 子查询性能问题

**问题**：子查询执行很慢

**原因**：时间范围太长或分辨率太高

**解决方案**：

```promql
# ❌ 慢：时间范围太长，分辨率太高
avg_over_time(http_requests_total[7d:10s])  ← 采样太多

# ✅ 快：降低分辨率
avg_over_time(http_requests_total[7d:1h])  ← 采样较少
```

### 6.7.3 向量匹配失败

**问题**：两个向量运算返回空结果

**原因**：标签不匹配

**解决方案**：

```promql
# ❌ 失败：标签不完全相同
a / b

# ✅ 成功：使用 on 指定匹配标签
a / on (common_label) b

# ✅ 成功：使用 ignoring 忽略不同标签
a / ignoring (different_label) b
```

---

## 6.8 知识检查

### 选择题

1. **以下哪个函数会自动处理 Counter 归零？**
   - A. delta()
   - B. rate()
   - C. idelta()
   - D. 以上都是

2. **irate() 和 rate() 的区别是？**
   - A. irate() 更平滑
   - B. rate() 更敏感
   - C. irate() 只使用最后两个点
   - D. 没有区别

3. **如何计算过去 1 小时的总请求数？**
   - A. `rate(http_requests_total[1h])`
   - B. `increase(http_requests_total[1h])`
   - C. `delta(http_requests_total[1h])`
   - D. `sum(http_requests_total)`

4. **子查询的语法是？**
   - A. `查询 (时间范围：分辨率)`
   - B. `查询 [时间范围：分辨率]`
   - C. `查询 {时间范围：分辨率}`
   - D. `查询 <时间范围：分辨率>`

5. **如何计算错误率？**
   - A. `sum(http_errors) / sum(http_requests)`
   - B. `sum(rate(http_errors[5m])) / sum(rate(http_requests[5m]))`
   - C. `rate(http_errors[5m]) / rate(http_requests[5m])`
   - D. `increase(http_errors) / increase(http_requests)`

### 简答题

1. 解释 rate()、irate()、increase()、delta() 的区别。
2. 什么是向量匹配？如何使用 on 和 ignoring？
3. 子查询的用途有哪些？
4. 如何计算 P95 延迟？
5. 如何设计查询检测异常（当前值超过历史平均值的 2 倍）？

### 实操题

1. **数学运算**：
   - 计算内存使用率
   - 计算磁盘剩余空间
   - 找出 CPU 和内存都超过 80% 的实例

2. **时间范围函数**：
   - 计算过去 5 分钟、15 分钟、1 小时的 QPS
   - 计算过去 24 小时的总请求数
   - 计算过去 1 小时的内存变化量

3. **聚合函数**：
   - 按服务分组计算错误率
   - 找出请求量最高的 10 个接口
   - 计算系统的整体可用性

4. **子查询**：
   - 计算过去 7 天，每天的请求峰值
   - 检测当前速率是否超过过去 24 小时平均值的 3 倍
   - 计算过去 30 天，每周的平均错误率

---

## 6.9 延伸阅读

### 必读
- [PromQL 官方文档 - 操作符](https://prometheus.io/docs/prometheus/latest/querying/operators/)
- [PromQL 官方文档 - 函数](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [子查询详解](https://prometheus.io/docs/prometheus/latest/querying/operators/#subquery)

### 选读
- [PromQL 最佳实践](https://prometheus.io/docs/prometheus/latest/querying/best_practices/)
- [监控指标设计指南](https://prometheus.io/docs/practices/instrumentation/)
- [Grafana Prometheus 查询指南](https://grafana.com/docs/grafana/latest/datasources/prometheus/)

### 工具
- [PromQL 在线测试工具](https://promql.io/)
- [Grafana 仪表盘模板](https://grafana.com/grafana/dashboards/)
- [Prometheus 查询浏览器](http://localhost:9090/graph)

---

## 6.10 下节预告

**第 7 课：Prometheus 整体架构**

在下一课中，你将：
- 学习 Prometheus 的整体架构设计
- 理解各核心组件的职责（Retrieval、Storage、HTTP Server、Rule Manager、Notifier）
- 掌握组件间的通信机制
- 绘制完整的架构图
- 为深入理解源码打下基础

**预习任务**：
- 思考 Prometheus 是如何抓取指标的
- 思考抓取到的数据是如何存储的
- 思考告警是如何触发的

---

## 总结

通过本课学习，你应该已经掌握：

✅ 数学运算和比较操作符的使用  
✅ 时间范围函数（rate、irate、increase、delta）的区别和应用  
✅ 聚合函数的高级用法（嵌套聚合、条件聚合、Top N）  
✅ 子查询的语法和实际应用场景  
✅ 复杂查询表达式的编写技巧  
✅ 查询优化的最佳实践  

**关键收获**：
- 能够灵活运用各种 PromQL 函数
- 理解不同时间范围函数的适用场景
- 掌握子查询进行时间范围分析
- 能够编写复杂的监控查询

**下一步**：
- 完成实践环节的所有练习
- 在 Grafana 中创建完整的监控仪表盘
- 准备学习第 7 课的 Prometheus 架构

---

**学习提示**：PromQL 的进阶功能需要大量实践才能熟练掌握。建议在实际环境中多尝试不同的查询组合，观察结果的变化。同时，注意查询性能，避免编写低效的查询表达式。

祝你学习愉快！📚

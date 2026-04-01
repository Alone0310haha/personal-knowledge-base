# 第 9 课：抓取管理（Scrape Manager）

**学习时长**：3-4 小时  
**难度等级**：⭐⭐⭐ 进阶  
**先修要求**：完成第 8 课 - 服务发现机制（Service Discovery）

---

## 学习目标

完成本课程后，你将能够：

- ✅ 说清 Scrape Manager / Scrape Pool / Scrape Loop 的关系
- ✅ 理解 “job + instance 标签” 是怎么来的、为什么重要
- ✅ 理解 relabel_configs 与 metric_relabel_configs 的区别与作用位置
- ✅ 能根据 `/targets` 页面快速定位“发现正常但抓取失败”的原因
- ✅ 能写出常用的重标签规则：筛选目标、改写地址、补充业务标签、丢弃高基数指标

---

## 9.1 抓取链路：从 Targets 到 TSDB

你可以把抓取链路理解成一条流水线：

```
服务发现（产生 Target Groups）
    ↓
Scrape Manager（按 job 管理）
    ↓
Scrape Pool（一个 job 一套抓取规则）
    ↓
Scrape Loop（一个 Target 一个定时循环）
    ↓
抓 /metrics → 解析样本 → 标签处理 → 写入 TSDB
```

核心分工很明确：

- 服务发现：产出“有哪些 Targets”
- 抓取管理：决定“怎么抓、抓哪些、标签怎么定”，并持续执行抓取

---

## 9.2 Scrape Manager / Pool / Loop：一句话讲清楚

### 9.2.1 Scrape Manager（抓取管理器）

负责管理所有抓取任务，核心工作是：

- 按 `job_name` 把 Targets 分组
- 维护每个 job 的 Scrape Pool
- Targets 变化时，增删对应的 Scrape Loop

### 9.2.2 Scrape Pool（抓取池）

Scrape Pool 可以理解为：**一个 job 的“抓取配置 + Targets 管理单元”**。

- 基本对应 `scrape_configs` 中一个 `job_name`
- 这一组 Targets 共享同一套抓取规则：`scrape_interval`、`scrape_timeout`、`metrics_path`、鉴权、relabel 等

它不是线程池，更像“同一套规则下的一组抓取对象”。

### 9.2.3 Scrape Loop（抓取循环）

Scrape Loop 可以理解为：**对一个 Target 的定时拉取循环**。

每轮做的事很固定：

1) 到点触发（`scrape_interval`）  
2) 发请求抓取（默认 `GET /metrics`）  
3) 解析响应为样本  
4) 附加/改写标签（job、instance、relabel 等）  
5) 写入 TSDB，并记录抓取状态（如 `up`）  

---

## 9.3 抓取配置里最重要的几个点

一个 job 最常用、最需要记住的字段：

| 配置项 | 作用 | 常见坑 |
|---|---|---|
| `job_name` | 逻辑分组名，最终成为 `job` 标签 | job 名变化会导致时序“换名字” |
| `scrape_interval` | 抓取周期 | 周期太短会放大目标与 Prometheus 压力 |
| `scrape_timeout` | 单次抓取超时 | 必须小于 interval，否则会堆叠 |
| `metrics_path` | 指标路径（默认 `/metrics`） | 应用常见是 `/actuator/prometheus` |
| `scheme` | `http/https` | https 需要证书/跳过校验等配置 |
| `params` | 给抓取请求加 query 参数 | 部分 exporter 依赖参数切换输出 |
| `relabel_configs` | 目标级处理：筛选/改写目标与标签 | 写错会导致目标直接消失 |
| `metric_relabel_configs` | 样本级处理：过滤/改写指标与标签 | 用来控基数、降成本 |

---

## 9.4 job / instance 标签是怎么来的

Prometheus 最常见的两条“定位标签”：

- `job`：来自 `job_name`
- `instance`：默认来自目标地址 `__address__`（通常是 `host:port`）

直觉理解：

- `job` 用来区分“同类目标”  
- `instance` 用来区分“同类里的哪一个具体实例”  

很多排障可以先从这条查询开始：

```promql
up
```

再按 job 或 instance 过滤：

```promql
up{job="node-exporter"}
up{instance="node1:9100"}
```

---

## 9.5 Relabeling：分两层，别混

Prometheus 的重标签有两类，位置不同、目的也不同：

```
服务发现元标签（__meta_*）
    ↓
relabel_configs（目标级）
    ↓
最终 Targets（__address__/labels）
    ↓
抓取并解析样本
    ↓
metric_relabel_configs（样本级）
    ↓
写入 TSDB
```

### 9.5.1 relabel_configs：目标级（决定“抓不抓、抓哪里、标签怎么来”）

典型用途：

- 过滤目标：只抓满足条件的 targets
- 改写地址：把 `__address__` 改成真正可抓的地址
- 补业务标签：把元信息映射为最终 labels

一个常见例子（只抓标注了 `prometheus.io/scrape: "true"` 的 Pod）：

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
```

一个常见例子（改写 metrics_path）：

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
```

### 9.5.2 metric_relabel_configs：样本级（决定“哪些指标入库、标签保不保留”）

典型用途：

- 丢弃不需要的指标，减少存储与查询成本
- 删除高基数标签（例如 request_id、trace_id 这类）

例子（丢弃调试类指标）：

```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    action: drop
    regex: "debug_.*"
```

例子（删除高基数 label）：

```yaml
metric_relabel_configs:
  - action: labeldrop
    regex: "request_id|trace_id"
```

---

## 9.6 “超时与重试”怎么理解

Prometheus 的抓取逻辑更像“按周期轮询”：

- 单次抓取超过 `scrape_timeout`：这轮失败，`up=0`
- 抓取失败通常不会在当前周期内反复重试：下一次到点再抓

排查抓取问题时，优先看这几个自监控指标：

```promql
up
scrape_duration_seconds
scrape_samples_scraped
scrape_samples_post_metric_relabeling
```

---

## 9.7 实践：从 0 看懂一个 job 的抓取效果

目标：通过 Prometheus 自带页面把“目标 → 抓取 → 入库”串起来。

### 9.7.1 配一个最小 job（抓 Prometheus 自己）

```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

观察点：

- `/targets` 页面里，`prometheus` 这个 job 是否 `UP`
- 查询 `up{job="prometheus"}` 是否为 1

### 9.7.2 加一个 relabel：补充环境标签

```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          env: "dev"
```

观察点：

- 查询 `up{job="prometheus",env="dev"}`

### 9.7.3 加一个 metric_relabel：丢弃不需要的指标

```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
    metric_relabel_configs:
      - source_labels: [__name__]
        action: drop
        regex: "prometheus_tsdb_.*"
```

观察点：

- 被丢弃的指标在查询里查不到
- `scrape_samples_post_metric_relabeling` 会变小

---

## 9.8 排错清单：目标不见了 vs 目标抓不到

先分清是哪一类问题：

- 目标不见了：通常是 relabel 过滤掉了，或服务发现没产出目标
- 目标在但抓不到：通常是地址/端口/路径/鉴权/网络问题

最实用的两个页面：

- `/service-discovery`：看服务发现产出的 Target Groups 与 `__meta_*`
- `/targets`：看最终 Targets、错误信息、上次抓取耗时

最实用的一条查询：

```promql
up == 0
```

---

## 课后小结

- Scrape Pool 是 job 的管理单元：同一套抓取规则 + 一组 Targets
- Scrape Loop 面向单个 Target：按 interval 循环抓取、解析、入库
- relabel_configs 处理目标，metric_relabel_configs 处理样本
- 排障先看 `/targets` 和 `up`


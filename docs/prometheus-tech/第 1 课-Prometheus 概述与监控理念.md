# 第 1 课：Prometheus 概述与监控理念

**学习时长**：2-3 小时  
**难度等级**：⭐ 入门  
**先修要求**：无

---

## 学习目标

完成本课程后，你将能够：

- ✅ 理解 Prometheus 是什么以及它在监控系统中的定位
- ✅ 掌握 Prometheus 的核心特性和设计理念
- ✅ 了解 Prometheus 与其他监控系统的区别
- ✅ 识别 Prometheus 的典型应用场景
- ✅ 描述 Prometheus 生态系统的主要组件

---

## 1.1 监控系统的发展与 Prometheus 的定位

### 1.1.1 为什么需要监控？

在现代软件系统中，监控是确保系统稳定性、可靠性和性能的关键手段。监控帮助我们：

- **发现问题**：在用户之前发现系统异常
- **定位根因**：快速确定问题所在
- **趋势分析**：了解系统容量和性能趋势
- **容量规划**：基于历史数据做出扩容决策
- **告警通知**：及时通知相关人员处理问题

### 1.1.2 监控系统的演进

```
传统监控 (Nagios, Zabbix) 
    ↓
  基于指标 (Graphite, OpenTSDB)
    ↓
  云原生监控 (Prometheus, InfluxDB)
    ↓
  可观测性 (Metrics + Logs + Traces)
```

**传统监控的局限**：
- 配置复杂，需要手动定义每个监控项
- 数据模型简单，难以表达复杂的服务关系
- 扩展性差，难以应对动态的云环境
- 查询能力弱，不支持灵活的数据分析

### 1.1.3 Prometheus 的诞生

Prometheus 于 2012 年由 SoundCloud 公司开发，灵感来源于 Google 的 Borg 监控系统（Borgmon）。

**关键时间点**：
- **2012 年**：在 SoundCloud 内部启动
- **2015 年**：开源发布
- **2016 年**：加入 CNCF（Cloud Native Computing Foundation），成为第二个孵化项目
- **2018 年**：毕业成为 CNCF 托管的第二个成熟项目（第一个是 Kubernetes）
- **2020 年**：Prometheus 2.0 发布，引入 TSDB
- **2024 年**：Prometheus 3.0 发布，原生支持直方图和更多云原生特性

**为什么 Prometheus 如此成功？**
1. **云原生时代**：与 Kubernetes 一起成长，成为云原生监控的事实标准
2. **开源社区**：活跃的社区贡献和快速迭代
3. **简单强大**：简单的 Pull 模型 + 强大的 PromQL
4. **生态完善**：丰富的 Exporter 和集成

---

## 1.2 Prometheus 的核心特性

### 1.2.1 多维度数据模型

Prometheus 使用 **时间序列（Time Series）** 作为基本数据结构，每个时间序列由：

- **指标名称（Metric Name）**：描述测量的基本特征
- **标签（Labels）**：键值对，提供多维度信息
- **时间戳 + 值**：按时间顺序排列的数据点

#### 什么是时间序列？

**通俗理解**：时间序列就是"持续记录某个指标随时间的变化"。

**生活中的例子**：
- 🌡️ 每天记录体温：`36.5°C → 36.6°C → 37.2°C → 37.8°C`
- ⚖️ 每周记录体重：`65kg → 64.5kg → 64kg → 63.5kg`
- 📈 股票价格：每秒的价格变化
- 💓 心跳监测：每分钟的心跳次数

**监控系统的例子**：
- 💻 CPU 使用率：每秒的 CPU 占用百分比
- 📶 网络请求数：每分钟的请求数量
- 💾 内存使用量：每秒的内存占用字节数
- 🐛 错误发生次数：累计的错误数量

#### 时间序列的数据结构

```
时间序列 = 指标名称 + 标签集合 + (时间戳，值) 的序列
```

**示例**：
```
# 指标名称：http_requests_total
# 标签：{method="POST", handler="/api/users", status="200"}

时间戳              值
2024-01-01 10:00:00  100
2024-01-01 10:00:15  105
2024-01-01 10:00:30  112
2024-01-01 10:00:45  118
2024-01-01 10:01:00  125
```

**完整的时间序列表示**：
```
http_requests_total{method="POST", handler="/api/users", status="200"}
http_requests_total{method="POST", handler="/api/users", status="500"}
http_requests_total{method="GET", handler="/api/users", status="200"}
```

这是**三个不同的时间序列**，因为标签组合不同。

#### 为什么 Prometheus 使用时间序列？

**1. 符合监控的本质**
监控系统的核心问题是：**"系统的状态如何随时间变化？"**
- CPU 使用率是上升还是下降？
- 请求延迟是否在增加？
- 内存是否持续增长（可能泄漏）？
- 错误率是否异常？

**2. 高效的存储和压缩**
时间序列数据有很强的规律性，可以高效压缩：
- **时间戳压缩**：相邻时间戳的差值通常很小
- **值压缩**：相邻数据点的值通常相近
- **压缩效果**：从每个样本 16 字节压缩到约 1.37 字节（压缩率约 91%）

**3. 高效的查询和聚合**
```promql
# 查询过去 5 分钟的平均请求速率
avg(rate(http_requests_total[5m]))

# 预测磁盘何时写满
predict_linear(node_filesystem_free_bytes[1h], 3600 * 24)

# 同比分析（与上周对比）
http_requests_total / http_requests_total offset 1w
```

**4. 多维度分析**
```promql
# 按状态码聚合
sum by (status) (rate(http_requests_total[5m]))

# 按方法和处理聚合
sum by (method, handler) (rate(http_requests_total[5m]))

# 过滤特定条件
rate(http_requests_total{status="500"}[5m])
```

**5. 趋势分析和异常检测**
```promql
# 线性回归预测：4 小时后内存剩余
predict_linear(node_memory_MemFree_bytes[1h], 3600 * 4)

# 3-sigma 原则检测异常
rate(http_requests_total[5m]) > 
  avg(rate(http_requests_total[5m])) + 3 * stddev(rate(http_requests_total[5m]))
```

**6. 告警的自然表达**
```yaml
- alert: HighCPUUsage
  # CPU 使用率持续 5 分钟超过 80%
  expr: avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) < 0.2
  for: 5m
  
- alert: MemoryLeak
  # 内存持续增长 1 小时
  expr: predict_linear(node_memory_MemFree_bytes[1h], 3600) < 0
```

#### 时间序列 vs 传统表格

**传统方式（如 Excel）**：

| 日期 | 张三体温 | 李四体温 | 王五体温 |
|------|----------|----------|----------|
| 01/01 | 36.5 | 36.3 | 36.8 |
| 01/02 | 36.6 | 36.4 | 36.7 |

**问题**：
- ❌ 新增一个人需要加一列
- ❌ 新增测量时间需要改结构
- ❌ 查询"所有体温超过 37 度的人"需要逐行扫描

**Prometheus 方式（时间序列 + 标签）**：
```
body_temperature{person="张三", time="morning"}
body_temperature{person="李四", time="morning"}
body_temperature{person="王五", time="morning"}
body_temperature{person="赵六", time="morning"}  ← 新增的人
body_temperature{person="张三", time="evening"}  ← 新增的时间
```

**优势**：
- ✅ 新增维度**不需要改结构**
- ✅ 查询灵活：`body_temperature{person="张三"}`
- ✅ 聚合方便：`avg by (person)(body_temperature)`
- ✅ 过滤简单：`body_temperature > 37`

#### 可视化表示

```
体温 (°C)
  |
38|                                    ●
  |                              ●
37|                        ●
  |                  ●
36|            ●
  |      ●
35|  ●
  +----+----+----+----+----+----+----→ 时间
    1/1  1/2  1/3  1/4  1/5  1/6  1/7
```

一眼就能看出趋势：1/4-1/5 体温上升（可能生病了），1/6-1/7 体温下降（好转了）。

**优势**：
- 灵活的维度组合
- 支持细粒度的查询和聚合
- 便于多租户和隔离
- 支持趋势分析和预测
- 高效的存储和查询

### 1.2.2 PromQL 查询语言

PromQL（Prometheus Query Language）是 Prometheus 的查询语言，支持：

- **基础查询**：选择特定的时间序列
- **数学运算**：加减乘除、幂运算等
- **聚合操作**：sum、avg、max、min、count 等
- **时间函数**：rate、irate、increase、delta 等
- **预测分析**：predict_linear、holt_winters 等

**示例**：
```promql
# 查询过去 5 分钟的请求速率
rate(http_requests_total[5m])

# 按状态码聚合
sum by (status) (rate(http_requests_total[5m]))

# 预测磁盘何时写满
predict_linear(node_filesystem_free_bytes[1h], 3600 * 24)
```

### 1.2.3 Pull（拉取）模型

Prometheus 采用**Pull 模型**，主动从目标抓取指标：

```
Prometheus Server
      ↓ (定期抓取)
  Target:8080/metrics
```

**Pull 模型的优势**：
- **控制节奏**：Prometheus 控制抓取频率，避免目标过载
- **简化配置**：目标只需暴露 HTTP 端点
- **便于测试**：可以直接 curl 端点验证
- **服务发现**：天然支持动态服务发现

**Push 模型支持**：
对于批处理作业等无法被拉取的场景，Prometheus 提供 Pushgateway 作为中介。

### 1.2.4 服务发现

Prometheus 支持多种服务发现机制：

- **静态配置**：手动指定目标
- **文件服务发现**：监控文件变化
- **云服务商**：AWS EC2、GCE、Azure 等
- **编排系统**：Kubernetes、Docker Swarm
- **配置中心**：Consul、Etcd、Zookeeper
- **其他**：OpenStack、Marathon、Nomad 等

**示例（Kubernetes 服务发现）**：
```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### 1.2.5 本地 TSDB（时间序列数据库）

Prometheus 2.0 引入了自研的 TSDB，特点包括：

- **高效存储**：基于 Gorilla 论文的时间序列压缩算法
- **块状存储**：每 2 小时一个数据块
- **WAL 日志**：预写日志保证数据不丢失
- **内存映射**：mmap 技术加速查询
- **压缩合并**：后台自动压缩和保留策略

**存储结构**：
```
data/
├── wal/           # 预写日志
├── chunks_head/   # 内存块持久化
└── 01BX5Z9YKVT... # 数据块（2 小时）
    ├── meta.json
    ├── index
    └── chunks/
```

### 1.2.6 告警与规则

Prometheus 支持两种规则：

**记录规则（Recording Rules）**：
- 预计算昂贵的查询
- 将结果保存为新的时间序列
```yaml
- record: job:http_requests:rate5m
  expr: sum by (job) (rate(http_requests_total[5m]))
```

**告警规则（Alerting Rules）**：
- 定义告警条件
- 支持 `for` 持续时间
```yaml
- alert: HighErrorRate
  expr: rate(http_requests_total{status="500"}[5m]) > 0.1
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "高错误率告警"
    description: "错误率超过 10%"
```

---

## 1.3 Prometheus 与其他监控系统的对比

### 1.3.1 vs Nagios

| 特性 | Nagios | Prometheus |
|------|--------|------------|
| 数据模型 | 主机 + 服务 | 多维度时间序列 |
| 数据收集 | Push/Pull | Pull |
| 查询能力 | 弱 | 强（PromQL） |
| 扩展性 | 配置复杂 | 服务发现 |
| 告警 | 内置 | 分离（Alertmanager） |
| 可视化 | 简单 | Grafana 集成 |

**结论**：Nagios 适合传统 IT 基础设施，Prometheus 适合云原生环境

### 1.3.2 vs Graphite

| 特性 | Graphite | Prometheus |
|------|----------|------------|
| 数据模型 | 扁平路径 | 多维度标签 |
| 查询语言 | 函数组合 | PromQL |
| 时间精度 | 固定分辨率 | 毫秒级时间戳 |
| 聚合时机 | 写入时 | 查询时 |
| 服务发现 | 无 | 内置支持 |

**结论**：Prometheus 的数据模型更灵活，查询能力更强

### 1.3.3 vs InfluxDB

| 特性 | InfluxDB | Prometheus |
|------|----------|------------|
| 存储引擎 | 自研 | 自研 TSDB |
| 查询语言 | InfluxQL/Flux | PromQL |
| 部署模式 | 单节点/集群 | 单节点 + 联邦 |
| 生态系统 | 较小 | 庞大 |
| 云原生集成 | 一般 | 深度集成 K8s |

**结论**：InfluxDB 适合独立部署，Prometheus 适合云原生场景

### 1.3.4 对比总结

```
选择 Prometheus 的理由：
✅ 云原生环境（Kubernetes）
✅ 需要灵活的查询和分析
✅ 动态服务发现需求
✅ 强大的社区和生态

不选择 Prometheus 的理由：
❌ 需要长期存储（>1 年）
❌ 需要集群高可用（原生不支持）
❌ 需要日志和链路追踪（需配合其他工具）
❌ 大规模多租户（需 Thanos/Cortex）
```

---

## 1.4 典型应用场景

### 1.4.1 Kubernetes 集群监控

**场景描述**：监控 K8s 集群的资源使用、Pod 状态、节点健康等

**架构**：
```
Prometheus Server (StatefulSet)
    ↓
  kube-state-metrics (集群状态)
  node-exporter (节点指标)
  cadvisor (容器指标)
  应用指标 (业务监控)
```

**关键指标**：
- `kube_pod_status_phase`：Pod 状态
- `node_cpu_seconds_total`：CPU 使用
- `container_memory_usage_bytes`：内存使用

### 1.4.2 微服务监控

**场景描述**：监控微服务架构中的服务健康、性能、依赖关系

**架构**：
```
Prometheus Server
    ↓
  Spring Boot Actuator (Java 应用)
  /metrics 端点 (Go/Python/Node.js)
  API Gateway 指标
  数据库连接池
```

**关键指标**：
- `http_requests_total`：请求量
- `http_request_duration_seconds`：请求延迟
- `jvm_memory_used_bytes`：JVM 内存

### 1.4.3 基础设施监控

**场景描述**：监控物理机、虚拟机、网络设备、存储等基础设施

**架构**：
```
Prometheus Server
    ↓
  node-exporter (系统指标)
  blackbox-exporter (网络探测)
  snmp-exporter (网络设备)
  haproxy-exporter (负载均衡)
```

**关键指标**：
- `node_load1`：系统负载
- `node_filesystem_free_bytes`：磁盘空间
- `node_network_receive_bytes_total`：网络流量

### 1.4.4 业务指标监控

**场景描述**：监控业务相关的指标，如订单量、用户活跃度、转化率等

**架构**：
```
业务应用
    ↓ (嵌入 SDK)
Micrometer / prom-client
    ↓ (暴露/metrics)
Prometheus Server
    ↓
Grafana 仪表盘 + 告警
```

**关键指标**：
- `order_created_total`：订单创建数
- `user_active_count`：活跃用户数
- `payment_amount_total`：支付金额

---

## 1.5 Prometheus 生态系统

### 1.5.1 核心组件

```
┌─────────────────────────────────────────────────────┐
│                  Prometheus 生态系统                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────┐          ┌──────────────┐        │
│  │  Prometheus  │◄────────►│ Alertmanager │        │
│  │    Server    │          │              │        │
│  └──────┬───────┘          └──────────────┘        │
│         │                                          │
│    ┌────┴────┐                                     │
│    │         │                                     │
│    ▼         ▼                                     │
│ ┌──────┐ ┌────────┐                                │
│  │ Push │ │ 各种  │                                │
│  │gateway│ │Exporter│                               │
│  └──────┘ └────────┘                                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Prometheus Server**：核心服务器，负责抓取、存储、查询
**Alertmanager**：处理告警，支持分组、抑制、静默
**Pushgateway**：接收 Push 模式的指标
**Exporters**：将第三方指标转换为 Prometheus 格式

### 1.5.2 常用 Exporters

| Exporter | 用途 | GitHub |
|----------|------|--------|
| node-exporter | 主机硬件和操作系统 | prometheus/node_exporter |
| blackbox-exporter | 网络探测（HTTP、TCP、ICMP） | prometheus/blackbox_exporter |
| mysqld-exporter | MySQL 数据库 | prometheus/mysqld_exporter |
| redis-exporter | Redis 数据库 | oliver006/redis_exporter |
| kafka-exporter | Kafka 消息队列 | danielqsj/kafka_exporter |
| elasticsearch-exporter | Elasticsearch | prometheus-community/elasticsearch_exporter |

### 1.5.3 客户端库

| 语言 | 官方库 |
|------|--------|
| Java | prometheus/client_java |
| Go | prometheus/client_golang |
| Python | prometheus/client_python |
| Node.js | siimon/prom-client |
| Ruby | prometheus/client |
| .NET | prometheus-net/prometheus-net |

### 1.5.4 可视化工具

**Grafana**：最流行的可视化平台
- 丰富的 Prometheus 数据源支持
- 海量社区仪表盘模板
- 强大的告警和通知功能

**Prometheus UI**：内置基础 UI
- Expression Browser：查询和绘图
- Status 页面：查看配置和状态
- Alerts 页面：查看告警

### 1.5.5 扩展项目

**Thanos**：高可用和长期存储
- 全局查询视图
- 对象存储集成
- 数据压缩和降采样

**Cortex**：多租户和水平扩展
- 分布式架构
- 多租户隔离
- 长期存储

**VictoriaMetrics**：高性能替代
- 兼容 PromQL
- 更高的性能
- 更低的存储成本

---

## 1.6 实践环节

### 实践 1.1：阅读官方文档

**任务**：
1. 访问 Prometheus 官网：https://prometheus.io
2. 阅读 Overview 页面：https://prometheus.io/docs/introduction/overview/
3. 阅读 README.md：`c:\code\back\prometheus-main\README.md`

**思考题**：
- Prometheus 的三个主要使用场景是什么？
- Prometheus 的 Pull 模型有什么优势？

### 实践 1.2：了解项目结构

**任务**：
1. 浏览 Prometheus 源码目录结构
2. 识别以下目录的用途：
   - `cmd/prometheus/`：主程序入口
   - `scrape/`：抓取逻辑
   - `storage/`：存储接口
   - `promql/`：查询引擎
   - `tsdb/`：时序数据库
   - `web/`：Web UI 和 API

**输出**：绘制简单的目录结构图，标注每个目录的职责

### 实践 1.3：加入社区

**任务**：
1. 关注 Prometheus GitHub：https://github.com/prometheus/prometheus
2. 订阅邮件列表：prometheus-developers@googlegroups.com
3. 加入 IRC 频道：#prometheus-dev on irc.libera.chat
4. 查看 PromCon 演讲视频：https://promcon.io/

---

## 1.7 知识检查

### 选择题

1. **Prometheus 采用哪种数据收集模型？**
   - A. Push 模型
   - B. Pull 模型
   - C. Push + Pull 混合模型
   - D. 以上都不是

2. **以下哪个不是 Prometheus 的核心特性？**
   - A. 多维度数据模型
   - B. PromQL 查询语言
   - C. 内置日志收集
   - D. 服务发现

3. **Prometheus 的时间序列由什么组成？**
   - A. 指标名称 + 时间戳
   - B. 指标名称 + 标签
   - C. 时间戳 + 值
   - D. 标签 + 值

4. **以下哪个场景最适合使用 Prometheus？**
   - A. 日志聚合
   - B. 链路追踪
   - C. Kubernetes 集群监控
   - D. 关系型数据库

5. **Prometheus 加入 CNCF 的时间是？**
   - A. 2015 年
   - B. 2016 年
   - C. 2018 年
   - D. 2020 年

### 简答题

1. 简述 Prometheus 的 Pull 模型相比 Push 模型的优势。
2. 描述 Prometheus 时间序列的数据模型。
3. 列举三个 Prometheus 的典型应用场景。
4. Prometheus 生态系统包含哪些核心组件？
5. 为什么 Prometheus 适合云原生环境？

---

## 1.8 延伸阅读

### 必读
- [Prometheus 官方文档 - 概述](https://prometheus.io/docs/introduction/overview/)
- [README.md](../README.md)
- [CONTRIBUTING.md](../CONTRIBUTING.md) - 贡献指南

### 选读
- [Prometheus 设计文档](https://web.archive.org/web/20210803115658/https://fabxc.org/tsdb/)
- [CNCF 官方博客：Prometheus 的成功之路](https://www.cncf.io/blog/)
- [PromCon 演讲视频](https://www.youtube.com/results?search_query=promcon)

### 书籍
- 《Prometheus: Up & Running》（O'Reilly）
- 《Prometheus 实战》（国内出版）

---

## 1.9 下节预告

**第 2 课：快速上手 - 安装与运行**

在下一课中，你将：
- 学习 Prometheus 的三种安装方式（Docker、二进制、源码）
- 动手运行第一个 Prometheus 实例
- 访问 Web UI 和状态页面
- 验证 Prometheus 正常服务

**预习任务**：
- 安装 Docker（如果尚未安装）
- 准备至少 1GB 可用磁盘空间
- 确保 9090 端口未被占用

---

## 总结

通过本课学习，你应该已经理解：

✅ Prometheus 是云原生时代的监控系统，由 SoundCloud 开发并捐赠给 CNCF  
✅ 核心特性包括多维度数据模型、PromQL、Pull 模型、服务发现、本地 TSDB  
✅ 相比传统监控系统，Prometheus 更适合动态、分布式的云环境  
✅ 典型应用场景包括 K8s 监控、微服务监控、基础设施监控、业务监控  
✅ 生态系统包含 Alertmanager、Pushgateway、各种 Exporters 和客户端库  

**关键收获**：
- 理解 Prometheus 的设计理念和适用场景
- 了解 Prometheus 在监控系统中的定位
- 为后续实践操作打下理论基础

---

**学习提示**：监控系统的学习需要理论与实践相结合。在理解概念的同时，要多动手操作，这样才能真正掌握 Prometheus 的精髓。

祝你学习愉快！📚

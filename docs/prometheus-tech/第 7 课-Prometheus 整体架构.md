# 第 7 课：Prometheus 整体架构

**学习时长**：3-4 小时  
**难度等级**：⭐⭐⭐ 进阶  
**先修要求**：完成第 6 课 - PromQL 进阶操作

---

## 学习目标

完成本课程后，你将能够：

- ✅ 理解 Prometheus 的整体架构设计
- ✅ 掌握 5 大核心组件的职责（Retrieval、Storage、HTTP Server、Rule Manager、Notifier）
- ✅ 理解组件间的通信机制和数据流
- ✅ 绘制完整的 Prometheus 架构图
- ✅ 从架构视角理解 Prometheus 的工作原理
- ✅ 为后续深入学习源码打下基础

---

## 7.1 Prometheus 架构概览

### 7.1.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Prometheus Server                         │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Retrieval   │  │   Storage    │  │  HTTP Server │          │
│  │   (抓取)     │→ │   (存储)     │← │   (Web API)  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│         ↓                  ↑                  ↑                  │
│         │                  │                  │                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │Rule Manager  │← │   Fanout     │  │   Notifier   │          │
│  │  (规则管理)  │  │   (分发)     │  │   (通知)     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
         ↓                  ↓                  ↓
    ┌─────────┐      ┌──────────┐      ┌──────────────┐
    │ Targets │      │  PromQL  │      │ Alertmanager │
    │ (目标)  │      │  查询    │      │   (告警)     │
    └─────────┘      └──────────┘      └──────────────┘
```

### 7.1.2 5 大核心组件

| 组件 | 职责 | 关键功能 |
|------|------|----------|
| **Retrieval** | 指标抓取 | 服务发现、抓取目标、指标解析 |
| **Storage** | 数据存储 | TSDB、内存 Head Block、持久化 Block |
| **HTTP Server** | Web 服务 | API 接口、PromQL 查询、UI 展示 |
| **Rule Manager** | 规则管理 | 告警规则、记录规则评估 |
| **Notifier** | 告警通知 | 发送告警到 Alertmanager |

### 7.1.3 数据流总览

```
外部 Targets
    ↓ (暴露 /metrics 端点)
Retrieval 组件
    ↓ (定期抓取)
    ↓ (解析指标)
Storage 组件
    ↓ (存储时间序列)
    ↓
    ├→ HTTP Server → PromQL 查询 → 用户/Grafana
    ├→ Rule Manager → 规则评估 → Notifier → Alertmanager
    └→ 远程存储 (Remote Write)
```

---

## 7.2 Retrieval（抓取）组件

### 7.2.1 职责概述

**Retrieval 组件**负责把外部 Target 暴露的 `/metrics` 拉进 Prometheus，并转换成可存储的时间序列样本。

**在 Prometheus 中的位置**：Targets → Retrieval → Storage（TSDB）→ Query/API。

**核心职责**：
- **服务发现**：找到可抓取目标（Targets）
- **抓取调度**：按 `scrape_interval` 定期抓取每个目标
- **解析入库**：解析响应、处理标签（Relabeling）、写入存储
- **抓取状态**：生成 `up`、抓取耗时等自监控指标

### 7.2.2 架构图

```
┌─────────────────────────────────────────────────────────┐
│                    Retrieval 组件                        │
│                                                          │
│  ┌──────────────┐      ┌──────────────┐                │
│  │   Discovery  │      │   Scrape     │                │
│  │   Manager    │─────→│   Manager    │                │
│  │  (服务发现)  │      │  (抓取管理)  │                │
│  └──────────────┘      └──────────────┘                │
│         ↓                      ↓                        │
│  ┌──────────────┐      ┌──────────────┐                │
│  │ Target Groups│      │  Scrape Loop │                │
│  │  (目标组)    │      │  (抓取循环)  │                │
│  └──────────────┘      └──────────────┘                │
│                            ↓                            │
│                    ┌──────────────┐                    │
│                    │   Relabeling │                    │
│                    │  (重标签)    │                    │
│                    └──────────────┘                    │
└─────────────────────────────────────────────────────────┘
```

### 7.2.3 Discovery Manager（服务发现管理器）

**职责**：动态发现和管理抓取目标

**支持的服务发现类型**：

| 类型 | 说明 | 配置示例 |
|------|------|----------|
| **static_configs** | 静态配置 | `targets: ["localhost:9090"]` |
| **file_sd** | 文件服务发现 | `files: ["targets/*.json"]` |
| **kubernetes_sd** | Kubernetes | `role: pod` |
| **consul_sd** | Consul | `server: "consul:8500"` |
| **dns_sd** | DNS | `names: ["service.consul"]` |
| **ec2_sd** | AWS EC2 | `region: "us-east-1"` |

**工作流程**：

```
1. 读取配置
   ↓
2. 初始化对应的服务发现器
   ↓
3. 定期查询服务注册中心
   ↓
4. 生成 Target Groups（目标组）
   ↓
5. 发送给 Scrape Manager
   ↓
6. 持续监听变化，动态更新
```

**示例配置**：

```yaml
scrape_configs:
  - job_name: "kubernetes-pods"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### 7.2.4 Scrape Manager（抓取管理器）

**职责**：管理所有的抓取任务

**核心概念**：

**1. Scrape Pool（抓取池）**
- 约等于 `scrape_configs` 中一个 `job_name` 对应的那一段配置
- 管一组 Targets（同一类目标），它们共享同一套抓取规则（interval/timeout/path/鉴权/relabel 等）
- 负责维护这些 Targets，并为每个 Target 启动/更新对应的 Scrape Loop

**2. Scrape Loop（抓取循环）**
- 面向单个 Target 的“定时拉取循环”（一个 Target 一个 Loop）
- 每轮：发请求抓取 → 解析样本 → 标签处理 → 写入存储 → 更新 `up`
- 各 Target 的抓取彼此独立，并行进行；Scrape Pool 不是“线程池”

**工作流程**：

```
Scrape Manager
    ↓
    ├→ Scrape Pool: prometheus
    │     ↓
    │     ├→ Scrape Loop: localhost:9090
    │
    ├→ Scrape Pool: node-exporter
    │     ↓
    │     ├→ Scrape Loop: node1:9100
    │     ├→ Scrape Loop: node2:9100
    │     └→ Scrape Loop: node3:9100
    │
    └→ Scrape Pool: application
          ↓
          ├→ Scrape Loop: app1:8080
          └→ Scrape Loop: app2:8080
```

### 7.2.5 Scrape Loop（抓取循环）

**职责**：执行具体的抓取任务

**执行流程**：

```
1. 等待抓取间隔（scrape_interval）
   ↓
2. 使用最新的目标地址与标签（已完成 relabel_configs 处理）
   ↓
3. 发起 HTTP GET 到 metrics_path（默认 /metrics）
   ↓
4. 解析响应为样本（时间序列）
   ↓
5. 附加基础标签（job、instance）与目标标签
   ↓
6. 执行 metric_relabel_configs（对样本做过滤/改写，可选）
   ↓
7. 写入 Storage 组件
   ↓
8. 记录抓取状态（up 等指标）
   ↓
9. 回到步骤 1，循环执行
```

**代码流程示意**：

```go
// 伪代码
func (sl *scrapeLoop) run() {
    for {
        // 1. 等待抓取间隔
        select {
        case <-sl.ctx.Done():
            return
        case <-sl.loopTicker.C:
        }
        
        // 2. 发起 HTTP 请求
        resp, err := sl.client.Get(sl.targetURL)
        
        // 3. 解析指标
        samples, err := sl.parser.Parse(resp.Body)
        
        // 4. 添加标签和 Relabeling
        samples = sl.relabel(samples)
        
        // 5. 写入存储
        sl.appender.Append(samples)
        
        // 6. 记录抓取状态
        sl.updateHealthStatus(err)
    }
}
```

### 7.2.6 Relabeling 在架构中的位置

```
服务发现返回的元标签（__meta_*）
    ↓
┌────────────────────┐
│ relabel_configs     │ 目标级处理：决定抓不抓/抓哪/给 Target 加哪些标签
└────────────────────┘
    ↓
Targets（最终的地址与标签）
    ↓
Scrape Loop：发起抓取 → 解析样本
    ↓
┌────────────────────┐
│ metric_relabel_configs │ 样本级处理：过滤/改写指标与标签（入库前）
└────────────────────┘
    ↓
写入 Storage（TSDB）
```

---

## 7.3 Storage（存储）组件

### 7.3.1 职责概述

**Storage 组件**负责存储所有抓取到的时间序列数据。

**核心职责**：
- 💾 **数据写入**：接收 Scrape Loop 写入的样本
- 🗄️ **数据存储**：使用 TSDB 高效存储
- 🔍 **数据查询**：响应 PromQL 查询请求
- 🔄 **数据压缩**：后台压缩和保留策略

### 7.3.2 架构图

```
┌─────────────────────────────────────────────────────────┐
│                    Storage 组件                          │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │                  Fanout Storage                   │  │
│  │  ┌──────────────┐      ┌──────────────┐         │  │
│  │  │   Local TSDB │      │ Remote Write │         │  │
│  │  │  (本地存储)  │      │ (远程写入)   │         │  │
│  │  └──────────────┘      └──────────────┘         │  │
│  └──────────────────────────────────────────────────┘  │
│                          ↓                              │
│  ┌──────────────────────────────────────────────────┐  │
│  │                   Local TSDB                      │  │
│  │  ┌──────────────┐      ┌──────────────┐         │  │
│  │  │  Head Block  │─────→│  WAL Log     │         │  │
│  │  │ (内存块)     │      │ (预写日志)   │         │  │
│  │  └──────────────┘      └──────────────┘         │  │
│  │         ↓                                        │  │
│  │  ┌──────────────┐      ┌──────────────┐         │  │
│  │  │   Block 1    │      │   Block 2    │         │  │
│  │  │ (持久化块)   │      │ (持久化块)   │         │  │
│  │  └──────────────┘      └──────────────┘         │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 7.3.3 Fanout Storage（分发存储）

**职责**：将数据同时写入多个存储后端

**写入模式**：

```
Scrape Loop
    ↓
Fanout Storage
    ↓
    ├→ Local TSDB（本地存储）
    │
    └→ Remote Write（远程存储，可选）
         ↓
         ├→ Thanos
         ├→ Cortex
         └→ 其他 Prometheus
```

**配置示例**：

```yaml
# 本地存储
storage:
  tsdb:
    path: "./data"
    retention:
      time: 15d

# 远程写入
remote_write:
  - url: "http://thanos-receive:8086/api/v1/receive"
    remote_timeout: 30s
```

### 7.3.4 Local TSDB（本地时序数据库）

Prometheus 自研的时序数据库，是存储的核心。

**TSDB 架构**：

```
┌─────────────────────────────────────────────────────────┐
│                      Local TSDB                         │
│                                                          │
│  ┌──────────────┐                                       │
│  │  Head Block  │ ← 内存中的最新数据（2 小时）            │
│  │  (内存块)    │   - 时间线（Time Series）              │
│  │              │   - 索引（Hash Table）                 │
│  └──────────────┘                                       │
│         ↓ (Checkpoint + WAL)                            │
│  ┌──────────────┐      ┌──────────────┐                │
│  │  Block 1     │      │  Block 2     │                │
│  │  (持久化块)  │      │  (持久化块)  │                │
│  │  [2h 前]     │      │  [4h 前]     │                │
│  └──────────────┘      └──────────────┘                │
│                                                          │
│  ┌──────────────┐                                       │
│  │  WAL Log     │ ← 预写日志（崩溃恢复）                 │
│  └──────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

**Head Block（内存块）**：
- 存储最近 2 小时的最新数据
- 完全在内存中，查询速度最快
- 使用哈希表索引，快速查找
- 定期创建 Checkpoint

**WAL（Write-Ahead Log）**：
- 预写日志，记录所有写入操作
- 崩溃恢复时重放 WAL
- 保证数据不丢失
- 顺序写入，性能高

**持久化 Block**：
- 每 2 小时创建一个 Block
- 包含：chunks（数据）、index（索引）、meta.json（元数据）
- 使用 Gorilla 压缩算法
- 后台压缩合并（Compaction）

### 7.3.5 数据写入流程

```
Scrape Loop
    ↓ (Append 样本)
Appender
    ↓
    ├→ 写入 WAL Log（顺序追加）
    │
    └→ 写入 Head Block
         ↓
         - 查找或创建时间线
         - 追加样本到时间线
         ↓
    每 2 小时
         ↓
    Checkpoint + 压缩
         ↓
    创建持久化 Block
         ↓
    写入磁盘
```

### 7.3.6 数据查询流程

```
HTTP Server (PromQL 查询)
    ↓
Query Engine
    ↓
Storage Querier
    ↓
    ├→ 查询 Head Block（内存）
    │
    ├→ 查询持久化 Block（磁盘）
    │
    └→ 合并结果
         ↓
    返回给 Query Engine
```

**查询优化**：
- 惰性执行（Lazy Evaluation）
- 并行查询多个 Block
- 索引加速查找
- 数据压缩减少 IO

---

## 7.4 HTTP Server 组件

### 7.4.1 职责概述

**HTTP Server 组件**提供 Web 界面和 API 接口，是用户与 Prometheus 交互的窗口。

**核心职责**：
- 🌐 **Web UI**：Expression Browser、状态页面
- 🔌 **HTTP API**：查询 API、管理 API
- 📊 **PromQL 引擎**：解析和执行查询
- 🔧 **生命周期管理**：配置重载、关机等

### 7.4.2 架构图

```
┌─────────────────────────────────────────────────────────┐
│                   HTTP Server 组件                       │
│                                                          │
│  ┌──────────────┐      ┌──────────────┐                │
│  │  Web UI      │      │  HTTP API    │                │
│  │  (Web 界面)  │      │  (API 接口)  │                │
│  └──────────────┘      └──────────────┘                │
│         ↓                      ↓                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │              PromQL Query Engine                  │  │
│  │              (PromQL 查询引擎)                     │  │
│  └──────────────────────────────────────────────────┘  │
│                          ↓                              │
│  ┌──────────────────────────────────────────────────┐  │
│  │                  Storage Querier                  │  │
│  │                  (存储查询器)                      │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 7.4.3 Web UI

**主要页面**：

| 页面 | 路径 | 功能 |
|------|------|------|
| **Expression Browser** | `/graph` | PromQL 查询和绘图 |
| **Targets** | `/targets` | 查看抓取目标状态 |
| **Alerts** | `/alerts` | 查看告警规则状态 |
| **Rules** | `/rules` | 查看规则配置 |
| **Service Discovery** | `/service-discovery` | 查看服务发现状态 |
| **Config** | `/config` | 查看当前配置 |
| **Status** | `/status` | 查看系统状态 |

### 7.4.4 HTTP API

**主要 API 端点**：

| API | 路径 | 功能 |
|-----|------|------|
| **即时查询** | `/api/v1/query` | 执行 PromQL 即时查询 |
| **范围查询** | `/api/v1/query_range` | 执行 PromQL 范围查询 |
| **标签列表** | `/api/v1/labels` | 获取所有标签名 |
| **标签值** | `/api/v1/label/<name>/values` | 获取标签的所有值 |
| **指标元数据** | `/api/v1/targets/metadata` | 获取指标元数据 |
| **配置重载** | `/-/reload` | 热重载配置 |
| **健康检查** | `/-/healthy` | 健康检查 |
| **就绪检查** | `/-/ready` | 就绪检查 |

**查询 API 示例**：

```bash
# 即时查询
curl -G http://localhost:9090/api/v1/query \
  --data-urlencode "query=up"

# 范围查询
curl -G http://localhost:9090/api/v1/query_range \
  --data-urlencode "query=rate(http_requests_total[5m])" \
  --data-urlencode "start=2024-01-01T00:00:00Z" \
  --data-urlencode "end=2024-01-01T01:00:00Z" \
  --data-urlencode "step=15s"
```

### 7.4.5 PromQL 查询引擎

**职责**：解析和执行 PromQL 查询

**执行流程**：

```
HTTP API 接收查询
    ↓
1. 词法分析（Lexer）
   - 将查询字符串拆分为 Token
   ↓
2. 语法分析（Parser）
   - 构建抽象语法树（AST）
   ↓
3. 查询规划（Planner）
   - 生成查询执行计划
   ↓
4. 查询执行（Executor）
   - 从 Storage 读取数据
   - 执行聚合、函数等操作
   ↓
5. 返回结果
   - 序列化为 JSON
   - 返回给客户端
```

**查询类型**：

**即时查询（Instant Query）**：
```promql
# 查询当前时刻的值
up
rate(http_requests_total[5m])
```

**范围查询（Range Query）**：
```promql
# 查询一段时间内的值（用于绘图）
# start=2024-01-01T00:00:00Z, end=2024-01-01T01:00:00Z, step=15s
up
rate(http_requests_total[5m])
```

### 7.4.6 生命周期管理

**配置重载**：

```bash
# 方法 1：HTTP API
curl -X POST http://localhost:9090/-/reload

# 方法 2：SIGHUP 信号
kill -SIGHUP <prometheus_pid>

# 方法 3：Docker
docker kill -s SIGHUP <container_name>
```

**需要 `--web.enable-lifecycle` 参数**：

```bash
prometheus --config.file=prometheus.yml --web.enable-lifecycle
```

**其他管理操作**：

```bash
# 优雅关机
curl -X POST http://localhost:9090/-/quit

# 删除时间序列数据
curl -X POST http://localhost:9090/api/v1/admin/tsdb/delete_series \
  -d "match[]=up{job=\"prometheus\"}"
```

---

## 7.5 Rule Manager（规则管理器）

### 7.5.1 职责概述

**Rule Manager 组件**负责管理和评估告警规则和记录规则。

**核心职责**：
- 📋 **规则加载**：从配置文件加载规则
- ⏰ **定时评估**：按 evaluation_interval 评估规则
- 🔔 **告警管理**：告警状态管理（Inactive/Pending/Firing）
- 📊 **记录规则**：预计算复杂查询

### 7.5.2 架构图

```
┌─────────────────────────────────────────────────────────┐
│                   Rule Manager 组件                      │
│                                                          │
│  ┌──────────────┐      ┌──────────────┐                │
│  │ Rule Groups  │      │  Evaluation  │                │
│  │  (规则组)    │─────→│   Manager    │                │
│  └──────────────┘      │  (评估管理)  │                │
│                        └──────────────┘                │
│                               ↓                        │
│  ┌──────────────┐      ┌──────────────┐                │
│  │ Alert Rules  │      │ Recording    │                │
│  │  (告警规则)  │      │   Rules      │                │
│  │              │      │  (记录规则)  │                │
│  └──────────────┘      └──────────────┘                │
│         ↓                      ↓                        │
│  ┌──────────────┐      ┌──────────────┐                │
│  │  Notifier    │      │   Storage    │                │
│  │  (通知器)    │      │   (存储)     │                │
│  └──────────────┘      └──────────────┘                │
└─────────────────────────────────────────────────────────┘
```

### 7.5.3 规则组（Rule Groups）

**规则组**：规则的集合，按组为单位进行评估。

**配置示例**：

```yaml
groups:
  - name: example-alerts
    interval: 30s  # 覆盖全局 evaluation_interval
    rules:
      - alert: HighCPU
        expr: node_cpu_usage > 0.8
        for: 5m
      
      - alert: HighMemory
        expr: node_memory_usage > 0.9
        for: 5m
  
  - name: recording-rules
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
```

**评估流程**：

```
Rule Manager
    ↓
遍历所有规则组
    ↓
对于每个规则组：
    - 等待 interval 时间
    - 并行评估组内所有规则
    ↓
重复执行
```

### 7.5.4 告警规则评估

**告警状态流转**：

```
Inactive（正常）
    ↓ (条件满足)
Pending（待处理）
    ↓ (持续 for 时间)
Firing（触发）
    ↓ (条件不满足)
Resolved（恢复）
```

**评估流程**：

```
1. 执行 expr 查询
   ↓
2. 获取结果向量
   ↓
3. 对于每个时间序列：
   - 检查条件是否满足
   - 更新告警状态
   - 如果满足 for 时间，触发告警
   ↓
4. 发送告警到 Notifier
```

**示例**：

```yaml
- alert: HighCPU
  expr: node_cpu_usage > 0.8
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "高 CPU 使用率"
    description: "实例 {{ $labels.instance }} CPU 使用率超过 80%"
```

**评估时间线**：

```
10:00  CPU = 75% → Inactive
10:05  CPU = 85% → Pending（开始计时）
10:10  CPU = 87% → Pending（持续 5 分钟）
10:10  达到 for 时间 → Firing（触发告警）
       发送通知到 Alertmanager
10:15  CPU = 70% → Resolved（恢复）
       发送恢复通知
```

### 7.5.5 记录规则评估

**记录规则**：预计算复杂查询，保存为新的时间序列。

**评估流程**：

```
1. 执行 expr 查询
   ↓
2. 获取结果向量
   ↓
3. 将结果作为新的时间序列
   ↓
4. 写入 Storage（以 record 名称为指标名）
```

**示例**：

```yaml
- record: job:http_requests:rate5m
  expr: sum by (job) (rate(http_requests_total[5m]))
```

**评估后**：
- 新的时间序列：`job:http_requests:rate5m{job="user-service"} = 125.5`
- 可以直接查询，无需重新计算

### 7.5.6 规则评估优化

**并行评估**：
- 同一规则组内的规则并行评估
- 不同规则组串行评估（避免资源竞争）

**查询优化**：
- 复用 Storage Querier
- 批量查询减少 IO
- 缓存查询结果

---

## 7.6 Notifier（通知器）组件

### 7.6.1 职责概述

**Notifier 组件**负责将告警发送到 Alertmanager。

**核心职责**：
- 📨 **告警发送**：将 Firing 告警发送到 Alertmanager
- 🔄 **服务发现**：动态发现 Alertmanager 实例
- ⏱️ **队列管理**：管理告警发送队列
- 🔁 **重试机制**：失败重试

### 7.6.2 架构图

```
┌─────────────────────────────────────────────────────────┐
│                    Notifier 组件                         │
│                                                          │
│  ┌──────────────┐      ┌──────────────┐                │
│  │  Alert Queue │      │  Alertmanager│                │
│  │  (告警队列)  │─────→│  Discovery   │                │
│  └──────────────┘      │  (服务发现)  │                │
│                        └──────────────┘                │
│                               ↓                        │
│  ┌──────────────┐      ┌──────────────┐                │
│  │   Retry      │      │   HTTP       │                │
│  │   Manager    │      │   Client     │                │
│  │  (重试管理)  │      │  (HTTP 客户端)│                │
│  └──────────────┘      └──────────────┘                │
└─────────────────────────────────────────────────────────┘
```

### 7.6.3 告警发送流程

```
Rule Manager（触发告警）
    ↓
添加到告警队列
    ↓
Notifier Manager
    ↓
从队列读取告警
    ↓
获取 Alertmanager 地址（服务发现）
    ↓
发送 HTTP POST 请求
    ↓
    ├→ 成功：清除队列
    │
    └→ 失败：加入重试队列
         ↓
     等待重试（指数退避）
```

### 7.6.4 Alertmanager 服务发现

**配置示例**：

```yaml
alerting:
  alertmanagers:
    # 静态配置
    - static_configs:
        - targets:
          - "alertmanager1:9093"
          - "alertmanager2:9093"
    
    # Kubernetes 服务发现
    - kubernetes_sd_configs:
        - role: pod
      relabel_configs:
        - source_labels: [__meta_kubernetes_pod_label_app]
          action: keep
          regex: alertmanager
```

### 7.6.5 告警格式

**发送到 Alertmanager 的告警格式**：

```json
{
  "labels": {
    "alertname": "HighCPU",
    "severity": "warning",
    "instance": "node1:9100"
  },
  "annotations": {
    "summary": "高 CPU 使用率",
    "description": "实例 node1:9100 CPU 使用率超过 80%"
  },
  "startsAt": "2024-01-01T10:10:00Z",
  "endsAt": "2024-01-01T10:15:00Z",
  "generatorURL": "http://prometheus:9090/graph?g0.expr=..."
}
```

---

## 7.7 组件间通信机制

### 7.7.1 整体数据流

```
┌──────────────────────────────────────────────────────────────┐
│                        完整数据流                            │
│                                                               │
│  Targets                                                     │
│     ↓ (暴露/metrics)                                          │
│  Retrieval 组件                                               │
│     ↓ (样本)                                                  │
│  Storage 组件                                                 │
│     ↓                                                         │
│     ├→ HTTP Server → PromQL → 用户/Grafana                   │
│     │                                                         │
│     ├→ Rule Manager → 规则评估                                │
│     │      ↓                                                  │
│     │      ├→ 告警规则 → Notifier → Alertmanager             │
│     │      │                                                  │
│     │      └→ 记录规则 → 写回 Storage                        │
│     │                                                         │
│     └→ Remote Write → 远程存储                                │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 7.7.2 组件间接口

**Retrieval → Storage**：

```go
// 接口定义
type Appender interface {
    Append(ref storage.SeriesRef, lset labels.Labels, t int64, v float64) storage.SeriesRef
    Commit() error
    Rollback() error
}

// Scrape Loop 调用
appender.Append(ref, labels, timestamp, value)
appender.Commit()
```

**HTTP Server → Storage**：

```go
// 接口定义
type Querier interface {
    Select(sortSeries bool, hints *SelectHints, matchers ...*labels.Matcher) SeriesSet
}

// PromQL Engine 调用
querier.Select(false, hints, matchers...)
```

**Rule Manager → Storage**：

```go
// 查询当前值
querier.Select(...)

// 写入记录规则结果
appender.Append(...)
```

**Rule Manager → Notifier**：

```go
// 发送告警
notifier.Send(alerts...)
```

### 7.7.3 并发模型

**Prometheus 使用 Goroutine 实现高并发**：

```
主进程
    ↓
    ├→ Discovery Manager（多个 Goroutine）
    │     ├→ Kubernetes SD
    │     ├→ Consul SD
    │     └→ File SD
    │
    ├→ Scrape Manager（多个 Goroutine）
    │     ├→ Scrape Loop 1
    │     ├→ Scrape Loop 2
    │     └→ Scrape Loop N
    │
    ├→ Rule Manager（多个 Goroutine）
    │     ├→ Rule Group 1
    │     ├→ Rule Group 2
    │     └→ Rule Group N
    │
    ├→ HTTP Server（多个 Goroutine）
    │     ├→ API Handler 1
    │     ├→ API Handler 2
    │     └→ API Handler N
    │
    └→ Notifier Manager（多个 Goroutine）
          ├→ Alertmanager 1
          └→ Alertmanager 2
```

---

## 7.8 实践环节

### 实践 7.1：绘制 Prometheus 架构图

**任务**：绘制完整的 Prometheus 架构图

**要求**：
1. 包含 5 大核心组件
2. 标注组件间的通信关系
3. 标注数据流方向
4. 添加关键子组件

**参考模板**：

```
┌─────────────────────────────────────────────────────────┐
│                    Prometheus Server                     │
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │ Retrieval  │→ │  Storage   │← │ HTTP Server│        │
│  └────────────┘  └────────────┘  └────────────┘        │
│       ↓                ↑                ↑               │
│       │                │                │               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │Rule Manager│← │   Fanout   │  │  Notifier  │        │
│  └────────────┘  └────────────┘  └────────────┘        │
└─────────────────────────────────────────────────────────┘
```

### 实践 7.2：分析组件职责

**任务**：分析每个组件的具体职责

**问题**：
1. Retrieval 组件包含哪些子组件？各有什么作用？
2. Storage 组件如何保证数据不丢失？
3. HTTP Server 如何响应 PromQL 查询？
4. Rule Manager 如何评估告警规则？
5. Notifier 如何发现 Alertmanager？

**参考答案**：
1. Retrieval 包含 Discovery Manager（服务发现）和 Scrape Manager（抓取管理）
2. Storage 使用 WAL 预写日志保证崩溃恢复
3. HTTP Server 通过 PromQL Engine 解析和执行查询
4. Rule Manager 按 evaluation_interval 定时评估
5. Notifier 支持静态配置和服务发现

### 实践 7.3：追踪数据流

**任务**：追踪一个指标从抓取到查询的完整流程

**场景**：用户查询 `rate(http_requests_total[5m])`

**步骤**：
1. Scrape Loop 抓取 `http_requests_total` 指标
2. 写入 Storage 的 Head Block
3. 用户通过 Grafana 发起查询
4. HTTP Server 接收查询请求
5. PromQL Engine 解析查询
6. Storage Querier 读取数据
7. 执行 rate() 函数计算
8. 返回结果给 Grafana

**绘制流程图**：

```
Grafana
  ↓ (查询请求)
HTTP Server
  ↓ (解析 PromQL)
PromQL Engine
  ↓ (查询数据)
Storage Querier
  ↓ (读取 Head Block + Blocks)
Storage
  ↓ (返回样本)
PromQL Engine
  ↓ (计算 rate())
PromQL Engine
  ↓ (返回结果)
HTTP Server
  ↓ (JSON 响应)
Grafana
```

### 实践 7.4：源码阅读指引

**任务**：阅读 Prometheus 源码，找到对应组件

**源码目录**：

| 组件 | 源码路径 | 关键文件 |
|------|----------|----------|
| **Retrieval** | `discovery/`, `scrape/` | `manager.go`, `scrape.go` |
| **Storage** | `tsdb/`, `storage/` | `db.go`, `head.go` |
| **HTTP Server** | `web/` | `web.go`, `api.go` |
| **Rule Manager** | `rules/` | `manager.go`, `rule.go` |
| **Notifier** | `notifier/` | `notifier.go` |
| **PromQL** | `promql/` | `engine.go`, `parser.go` |

**阅读顺序**：
1. `cmd/prometheus/main.go` - 主入口
2. `web/web.go` - Web 服务启动
3. `scrape/manager.go` - 抓取管理
4. `tsdb/db.go` - TSDB 主逻辑
5. `rules/manager.go` - 规则管理

---

## 7.9 架构演进

### 7.9.1 Prometheus 1.x vs 2.x

**Prometheus 1.x**：
- 使用 LevelDB 作为存储后端
- 存储性能较差
- 依赖外部存储

**Prometheus 2.x**：
- 自研 TSDB
- 存储性能提升 100 倍
- 完整的生命周期管理

### 7.9.2 Prometheus 3.x（规划中）

**改进方向**：
- 原生支持直方图
- 更好的远程读写
- 性能优化
- 云原生集成

---

## 7.10 知识检查

### 选择题

1. **Prometheus 的核心组件不包括？**
   - A. Retrieval
   - B. Storage
   - C. Kubernetes
   - D. HTTP Server

2. **Scrape Loop 的职责是？**
   - A. 服务发现
   - B. 执行具体抓取任务
   - C. 存储数据
   - D. 发送告警

3. **TSDB 的 Head Block 存储在哪里？**
   - A. 磁盘
   - B. 内存
   - C. 网络
   - D. 数据库

4. **告警规则的评估由哪个组件负责？**
   - A. Retrieval Manager
   - B. Storage Manager
   - C. Rule Manager
   - D. Notifier Manager

5. **PromQL 查询由哪个组件执行？**
   - A. HTTP Server 的 PromQL Engine
   - B. Storage Querier
   - C. Rule Manager
   - D. Scrape Manager

### 简答题

1. 描述 Prometheus 的 5 大核心组件及其职责。
2. 解释数据从抓取到查询的完整流程。
3. TSDB 如何保证数据不丢失？
4. 告警规则的状态流转过程是怎样的？
5. Notifier 如何发现 Alertmanager 实例？

---

## 7.11 延伸阅读

### 必读
- [Prometheus 官方架构文档](https://prometheus.io/docs/introduction/overview/#architecture)
- [TSDB 设计博客](https://ganeshvernekar.com/blog/prometheus-tsdb-design/)
- [Prometheus 源码](https://github.com/prometheus/prometheus)

### 选读
- [Prometheus 2.0 架构演进](https://prometheus.io/docs/introduction/overview/)
- [Thanos 架构](https://thanos.io/tip/thanos/architecture.md/)
- [Cortex 架构](https://cortexmetrics.io/docs/architecture/)

---

## 7.12 下节预告

**第 8 课：服务发现机制（Service Discovery）**

在下一课中，你将：
- 深入理解服务发现的概念和作用
- 学习 Prometheus 支持的各种服务发现类型
- 掌握 Discovery Manager 的工作原理
- 理解目标组的更新机制
- 实践配置文件服务发现（file_sd）

**预习任务**：
- 思考为什么需要服务发现
- 了解 Kubernetes 的服务发现机制
- 准备一个 file_sd 配置文件

---

## 总结

通过本课学习，你应该已经掌握：

✅ Prometheus 的 5 大核心组件（Retrieval、Storage、HTTP Server、Rule Manager、Notifier）  
✅ 各组件的职责和子组件  
✅ 组件间的通信机制和数据流  
✅ 完整的架构图绘制  
✅ 从架构视角理解 Prometheus 的工作原理  

**关键收获**：
- 理解 Prometheus 的整体架构设计
- 掌握数据从抓取到查询的完整流程
- 为后续深入学习各组件打下基础
- 能够从架构层面分析和解决问题

**下一步**：
- 完成实践环节的架构图绘制
- 阅读相关源码文件
- 准备学习第 8 课的服务发现机制

---

**学习提示**：架构理解是深入学习 Prometheus 的关键。建议多画架构图，多追踪数据流，将抽象的概念与具体的代码对应起来。后续课程会深入每个组件的实现细节。

祝你学习愉快！📚

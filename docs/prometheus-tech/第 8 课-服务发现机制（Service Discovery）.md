# 第 8 课：服务发现机制（Service Discovery）

**学习时长**：2-3 小时  
**难度等级**：⭐⭐⭐ 进阶  
**先修要求**：完成第 7 课 - Prometheus 整体架构

---

## 学习目标

完成本课程后，你将能够：

- ✅ 理解为什么 Prometheus 需要服务发现（Targets 从哪里来）
- ✅ 理解 Discovery Manager 在 Retrieval 链路中的位置与职责
- ✅ 说清 Target、Target Group、元标签（`__meta_*`）之间的关系
- ✅ 了解常见服务发现方式（static/file/dns/consul/k8s）
- ✅ 通过 `/targets` 与 `/service-discovery` 页面定位“找不到目标/抓不到指标”的问题

---

## 8.1 服务发现解决什么问题

Prometheus 是 Pull 模型：Prometheus 主动去抓 `/metrics`。难点在于目标地址经常变化：

- 容器/Pod 会重建，IP/端口会变
- 应用会扩缩容，实例数量会变
- 服务会迁移，旧地址会失效

**服务发现（Service Discovery）** 的作用就是：持续产出“当前应该抓哪些目标（Targets）”，并把结果交给抓取模块去执行。

一句话总结：

> 服务发现负责“找目标并更新目标列表”；抓取负责“按目标列表定时去抓”。

### 8.1.1 一个容易理解的类比

- 更像 Nacos/Consul 的“实例列表订阅”（客户端拿到最新实例清单）
- 不像 Gateway：Gateway 负责转发业务流量；服务发现只产出 Targets，不转发流量

---

## 8.2 Discovery Manager 在 Prometheus 中的位置

在 Prometheus 的数据流里：

```
Targets（可能变化）
    ↓（服务发现）
Discovery Manager
    ↓（Target Groups）
Scrape Manager（按 job 管理）
    ↓（每个 Target 一个 Scrape Loop）
抓取 /metrics → 解析 → 入库（TSDB）
```

Discovery Manager 做两件事：

1) 读取 `scrape_configs` 中配置的服务发现方式（如 kubernetes_sd、consul_sd、file_sd）  
2) 持续产出 Target Groups，并在变化时推送更新

它不负责抓取，真正发请求抓 `/metrics` 的是 Scrape Loop。

---

## 8.3 三个关键概念：Target / Target Group / 元标签

### 8.3.1 Target（目标）

**Target**就是一次抓取要访问的地址，例如：

- `http://node1:9100/metrics`
- `http://10.42.1.23:8080/metrics`

Prometheus 最终需要的是“可抓取的目标列表”，也就是一组 Targets。

### 8.3.2 Target Group（目标组）

服务发现通常按来源分组返回 **Target Group**。

Target Group 里包含两类信息：

- `targets`：一组地址（如 `["10.42.1.23:8080", "10.42.1.24:8080"]`）
- `labels`：这组目标共享的一些标签（例如 `namespace="prod"`）

Scrape Manager 只关心“最新的 Target Groups → 最新的 Targets”。

### 8.3.3 元标签（`__meta_*`）与内部标签（`__*__`）

服务发现会给目标带一些“描述信息”，例如 Kubernetes 的：

- Pod 名称、Namespace、Label、Annotation

这些信息以 **元标签**形式提供（`__meta_*`）。它们通常不会直接入库，而是用来做目标筛选和改写，例如：

- 只抓有某个 annotation 的 Pod
- 把 Pod 的 label 映射成最终的业务标签

此外还有一些内部标签，用于控制“最终怎么抓”（常见的有）：

- `__address__`：最终抓取地址（host:port）
- `__scheme__`：http/https
- `__metrics_path__`：指标路径（默认 `/metrics`）
- `__param_*`：给抓取请求添加 query 参数

---

## 8.4 服务发现的“更新机制”是什么

服务发现不是“一次性算出目标”，而是持续更新：

- static_configs：基本不变（除非你改配置并 reload）
- file_sd：读取文件，文件变了就更新
- kubernetes_sd：watch API，资源变更就更新
- dns_sd：按 DNS 刷新周期重新解析

Discovery Manager 将变化统一成“Target Groups 的增删改”，推送给 Scrape Manager。Scrape Manager 再据此创建/销毁对应 Target 的 Scrape Loop。

你可以把它理解为：

> 服务发现像“维护通讯录”；抓取像“按通讯录打电话”。通讯录变了，打电话的对象就跟着变。

---

## 8.5 常见服务发现方式（你需要知道的最小集合）

### 8.5.1 static_configs（静态）

适合少量、稳定的目标。

```yaml
scrape_configs:
  - job_name: "node-exporter"
    static_configs:
      - targets: ["node1:9100", "node2:9100"]
```

### 8.5.2 file_sd（文件服务发现）

适合你希望由“外部系统生成目标清单文件”，Prometheus 只负责读取。

一个典型场景：没有 K8s/Consul/Nacos，但发布平台/CMDB 能导出“当前在线实例列表”。

```yaml
scrape_configs:
  - job_name: "file-sd-demo"
    file_sd_configs:
      - files: ["targets/*.json"]
```

JSON 文件示例（`targets/demo.json`）：

```json
[
  {
    "targets": ["127.0.0.1:9100"],
    "labels": { "env": "dev", "team": "ops" }
  }
]
```

### 8.5.3 dns_sd（DNS）

适合服务有稳定域名，但背后 IP 会变化的场景。

```yaml
scrape_configs:
  - job_name: "dns-demo"
    dns_sd_configs:
      - names: ["node-exporter.service.local"]
        type: "A"
        port: 9100
```

### 8.5.4 kubernetes_sd（Kubernetes）

适合在 K8s 内抓取 Pod/Service/Endpoints 等资源。最常见的做法是结合 relabel 做筛选：

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

本课重点：理解 kubernetes_sd 会产出哪些 `__meta_*` 元标签；如何用 relabel 把“海量目标”变成“你真正想抓的目标”。

---

## 8.6 从“发现”到“可抓取”的最短路径（带一个直觉例子）

假设 Kubernetes 里有 3 个 Pod，其中只有 2 个标注了 `prometheus.io/scrape: "true"`。

1) kubernetes_sd 发现 3 个 Pod，给每个 Pod 带上 `__meta_*` 元标签  
2) relabel 使用 `action: keep` 过滤掉不该抓的 Pod  
3) 过滤后的 Pods 进入 Scrape Manager，对每个 Pod 生成最终 `__address__` 与标签  
4) 每个 Pod 一个 Scrape Loop，按间隔抓 `/metrics`

结果就是：

- “发现阶段”目标很多没关系  
- “是否抓、抓哪里、标签怎么来”，都是在从元标签到最终 Target 的过程中被确定的

---

## 8.7 实践：用 file_sd 观察目标动态变化

目标：亲眼看到“目标列表更新”这件事，不依赖 Kubernetes。

1) 在 `prometheus.yml` 增加一个 `file_sd_configs` 的 job  
2) 在 `targets/` 下放一个 JSON 文件，写 1 个 target  
3) 访问 Prometheus 的 `/targets` 页面确认该 target 出现  
4) 修改 JSON：新增/删除一个 target  
5) 再看 `/targets`，观察目标变化

如果目标出现了但 `up=0`，说明“发现没问题，抓取失败”，常见原因是端口/路径/网络不通。

---

## 8.8 排错清单：到底卡在“发现”还是“抓取”

- `/service-discovery`：看服务发现是否产出了你预期的 Target Groups，以及有哪些 `__meta_*`
- `/targets`：看最终 Targets 是否正确（地址、labels、状态、错误信息）
- 先保证 `__address__` 正确，再看 `metrics_path`、scheme、鉴权
- 如果目标被过滤掉：重点检查 relabel 的 `keep/drop` 规则与 `regex`

---

## 课后小结

- Discovery Manager 负责把“变化的外部世界”变成“持续更新的 Target Groups”
- Scrape Manager 负责把 Target Groups 变成最终 Targets，并为每个 Target 运行 Scrape Loop
- `__meta_*` 是“发现阶段的原材料”，relabel 用它把“原材料”加工成“最终可抓取目标与标签”


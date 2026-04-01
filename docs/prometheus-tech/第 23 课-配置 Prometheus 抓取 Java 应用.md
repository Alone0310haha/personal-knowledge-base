# 第 23 课：配置 Prometheus 抓取 Java 应用

**学习时长**：2-3 小时  
**难度等级**：⭐⭐⭐ 进阶  
**先修要求**：完成第 21 课 - Java 监控指标基础、第 8-9 课（服务发现与抓取管理）

---

## 学习目标

完成本课程后，你将能够：

- ✅ 把 Java 应用指标端点接入 Prometheus（static / file_sd / k8s 三种常见方式）
- ✅ 正确设置 `metrics_path`、`scheme`、鉴权、超时与抓取周期
- ✅ 理解并使用 relabel_configs：筛选目标、改写地址、补充业务标签
- ✅ 理解 metric_relabel_configs：过滤指标、控制基数、降成本
- ✅ 能用 `/targets` 排查“发现正常但抓取失败”的问题

---

## 23.1 先把前提说清：Prometheus 需要什么

Prometheus 抓 Java 应用，本质上只要两样东西：

1) **可访问的目标地址**：`host:port`
2) **可访问的指标路径**：`/metrics` 或 `/actuator/prometheus`

抓取能否成功，优先从这条链路确认：

- 浏览器/curl 能否访问目标的 metrics endpoint
- Prometheus `/targets` 是否 UP

---

## 23.2 最简单方式：static_configs

适合：实例数量少、地址稳定、快速验证链路。

```yaml
scrape_configs:
  - job_name: "spring-boot"
    metrics_path: /actuator/prometheus
    scrape_interval: 15s
    scrape_timeout: 5s
    static_configs:
      - targets:
          - "10.0.1.12:8080"
          - "10.0.1.13:8080"
        labels:
          env: "prod"
          app: "order"
```

你会在时序里看到：

- `job="spring-boot"`（来自 job_name）
- `instance="10.0.1.12:8080"`（来自目标地址）
- `env/app`（来自 static_configs.labels）

---

## 23.3 动态目标：file_sd

适合：没有 K8s/Consul/Nacos，但你有发布平台/CMDB 能导出实例清单。

```yaml
scrape_configs:
  - job_name: "spring-boot"
    metrics_path: /actuator/prometheus
    file_sd_configs:
      - files: ["targets/java/*.json"]
```

`targets/java/order.json` 示例：

```json
[
  {
    "targets": ["10.0.1.12:8080", "10.0.1.13:8080"],
    "labels": { "env": "prod", "app": "order" }
  }
]
```

文件变化后，Prometheus 会自动更新 targets，并对应增删抓取循环。

---

## 23.4 Kubernetes：抓 Pod 还是抓 Service

在 K8s 内接入 Java 应用，常见两种抓取对象：

- 抓 Pod：粒度细、能区分每个实例，适合扩缩容频繁场景
- 抓 Service：更像“统一入口”，但可能隐藏单实例问题（并不总推荐）

本课给出最常见的“抓 Pod + 注解控制”的方式。

### 23.4.1 Pod 注解（示例）

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: "/actuator/prometheus"
    prometheus.io/port: "8080"
```

### 23.4.2 Prometheus 抓取配置（kubernetes_sd + relabel）

```yaml
scrape_configs:
  - job_name: "kubernetes-pods-java"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\\d+)?;(\\d+)
        replacement: $1:$2

      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace

      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
```

直觉理解：

- keep：只抓被标注 scrape=true 的 Pod
- replace __metrics_path__：把注解里的 path 变成最终抓取路径
- replace __address__：把 Pod IP + 注解 port 拼成最终地址
- 把元标签映射成最终业务标签（namespace/pod/app）

---

## 23.5 鉴权与 HTTPS：常见配置点

Java 应用的 metrics endpoint 可能需要鉴权或走 HTTPS，常见方式：

- basic_auth
- bearer_token / bearer_token_file
- tls_config（证书校验/跳过校验）

示意（只展示结构，不强行规定你一定用哪种）：

```yaml
scrape_configs:
  - job_name: "spring-boot-secure"
    scheme: https
    metrics_path: /actuator/prometheus
    tls_config:
      insecure_skip_verify: true
    basic_auth:
      username: "prom"
      password: "prom"
    static_configs:
      - targets: ["app:8443"]
```

---

## 23.6 控制成本：metric_relabel_configs 的两个常用用法

### 23.6.1 丢弃不需要的指标

```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    action: drop
    regex: "jvm_classes_loaded_.*"
```

### 23.6.2 删除高基数标签

```yaml
metric_relabel_configs:
  - action: labeldrop
    regex: "trace_id|request_id"
```

经验建议：

- 先把链路跑通，再做过滤
- 过滤要可解释，否则后面排障会很痛苦

---

## 23.7 排错：用 `/targets` 快速定位问题

当你看到 `up=0` 或 targets 显示 DOWN，优先看 `/targets` 的错误原因。

常见原因与方向：

- 连接失败：地址/端口/网络
- 404：metrics_path 不对（Spring Boot 常见是没配 `/actuator/prometheus`）
- 401/403：鉴权配置不对
- 超时：scrape_timeout 太小或应用端点太慢
- 目标没出现：relabel keep/drop 把目标过滤掉了，去 `/service-discovery` 看元标签与 relabel 结果

---

## 23.8 最小验收清单

- 应用端点可访问：`/actuator/prometheus` 返回文本
- `/targets` 中 job 为 UP
- `up{job="..."} == 1`
- 至少能查到 1-2 个关键业务指标（第 22 课定义的那类）

---

## 课后小结

- 接入 Java 应用的关键是：目标地址 + metrics_path + 正确的服务发现/重标签
- relabel_configs 解决“抓哪些/抓哪里/标签怎么来”
- metric_relabel_configs 解决“哪些指标入库/标签怎么控基数”
- 排障优先看 `/targets` 与 `/service-discovery`


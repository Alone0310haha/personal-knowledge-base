# 第 28 课：实现 HTTP API 与 UI

**学习时长**：4-6 小时  
**难度等级**：⭐⭐⭐ 进阶  
**先修要求**：完成第 27 课 - 实现 PromQL 子集

---

## 学习目标

完成本课程后，你将能够：

- ✅ 为你的 PromQL 子集实现 2 个核心 API：`/api/v1/query` 与 `/api/v1/query_range`
- ✅ 设计一个最小 JSON 返回结构（对齐 Prometheus 风格：status/data/resultType/result）
- ✅ 实现一个简单 Web UI：输入 PromQL → 展示结果（表格/简单曲线）
- ✅ 理解 query_range 的三个关键参数：start/end/step
- ✅ 跑通端到端：浏览器调用 API 与 UI 能看到正确结果

---

## 28.1 你要实现的最小系统长什么样

在第 27 课你已经有：

- Parser：PromQL → AST
- Evaluator：AST → 结果
- Storage：Select + Range

这一课要做的是把它们“对外服务化”：

```mermaid
flowchart TD
  A[Browser/UI] --> B[HTTP Server]
  B --> C[/api/v1/query]
  B --> D[/api/v1/query_range]
  C --> E[Parse + Eval@ts]
  D --> F[Parse + Eval@t0..tn]
  E --> G[JSON Response]
  F --> G
  B --> H[Simple UI]
  H --> C
  H --> D
```

---

## 28.2 API 设计：对齐 Prometheus 的最小返回格式

Prometheus HTTP API 的返回结构大致是：

```json
{
  "status": "success",
  "data": {
    "resultType": "vector|matrix|scalar|string",
    "result": [...]
  }
}
```

你实现子集时建议先只支持：

- Instant Query → `vector`
- Range Query → `matrix`

### 28.2.1 Instant Vector（vector）示例

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": { "job": "prometheus", "instance": "localhost:9090" },
        "value": [ 1711944000, "1" ]
      }
    ]
  }
}
```

### 28.2.2 Range Vector（matrix）示例

```json
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": { "job": "prometheus", "instance": "localhost:9090" },
        "values": [
          [ 1711944000, "1" ],
          [ 1711944015, "1" ]
        ]
      }
    ]
  }
}
```

注意：

- timestamp 用秒
- value 用字符串（保持兼容风格）

---

## 28.3 `/api/v1/query`：瞬时查询接口

### 28.3.1 请求参数

- `query`：PromQL 字符串
- `time`（可选）：评估时间点（Unix 秒）；不传就用当前时间

示例：

```text
GET /api/v1/query?query=up{job="prometheus"}
GET /api/v1/query?query=rate(http_requests_total[5m])&time=1711944000
```

### 28.3.2 执行流程（伪代码）

```text
func handleQuery(req):
  qs = req.query["query"]
  ts = parseTime(req.query["time"], default=now())
  expr = Parse(qs)
  vec  = EvalInstant(expr, ts)
  return JSON(vector(vec))
```

---

## 28.4 `/api/v1/query_range`：范围查询接口

### 28.4.1 请求参数

- `query`：PromQL 字符串
- `start`：开始时间（Unix 秒）
- `end`：结束时间（Unix 秒）
- `step`：步长（秒，例如 `15` 或 `15s`）

示例：

```text
GET /api/v1/query_range?query=up&start=1711944000&end=1711947600&step=15
```

### 28.4.2 执行流程（伪代码）

```text
func handleQueryRange(req):
  qs = req.query["query"]
  start = parseUnix(req.query["start"])
  end   = parseUnix(req.query["end"])
  step  = parseStep(req.query["step"])

  expr = Parse(qs)

  matrix = []
  for t in range(start, end, step):
    vec = EvalInstant(expr, t)
    matrix = appendPoint(matrix, vec, t)

  return JSON(matrix(matrix))
```

一个关键点：

- **query_range 的执行通常是“重复执行 instant eval”**
- 这就是为什么 step 越小，越容易慢

---

## 28.5 UI：一个最小可用的查询页面

UI 的目标只做两件事：

- 输入查询表达式与时间参数
- 展示结果（vector 用表格，matrix 用折线图）

### 28.5.1 页面布局建议

- 输入框：PromQL
- 模式选择：Instant / Range
- 参数区：time 或 start/end/step
- 执行按钮
- 结果区：表格/图

### 28.5.2 最简实现方式

你可以选择最简单的实现路线：

- UI 用纯 HTML + 少量 JS（fetch 调 API）
- 图表用最简 SVG/Canvas（先画折线即可）

不建议上来就引入复杂前端框架，先跑通闭环再说。

---

## 28.6 错误处理：对齐 Prometheus 风格

Prometheus API 出错时通常返回：

```json
{
  "status": "error",
  "errorType": "bad_data|server_error|timeout",
  "error": "详细错误信息"
}
```

你最少需要区分三类：

- bad_data：参数错误、解析错误、语义错误
- timeout：查询超时
- server_error：内部异常

---

## 28.7 最小验收清单

1) Instant Query：

- `GET /api/v1/query?query=up`

2) Range Query：

- `GET /api/v1/query_range?query=up&start=...&end=...&step=15`

3) UI：

- 能输入 `up` 并显示结果
- 能输入 range 并画出折线（哪怕很简陋）

---

## 28.8 常见坑与修正

- time/start/end/step 单位混乱：统一使用秒
- step 太小导致非常慢：先限制最小 step（例如 >= 1s 或 >= 5s）
- range 太大导致 OOM：加上最大返回点数限制
- matrix 合并逻辑错：需要按“series labels”聚合点，不能按返回顺序硬拼

---

## 课后小结

- `/query` 就是“算一次”，`/query_range` 就是“按 step 算很多次”
- API 的最小返回格式对齐 Prometheus，后续才能无痛接 Grafana/兼容工具
- UI 先做最小闭环，别急着做复杂交互


# 第 27 课：实现 PromQL 子集

**学习时长**：4-6 小时  
**难度等级**：⭐⭐⭐⭐ 深入  
**先修要求**：完成第 26 课 - 实现时间序列存储

---

## 学习目标

完成本课程后，你将能够：

- ✅ 设计一个“够用”的 PromQL 子集语法（先覆盖 80% 场景）
- ✅ 实现一个最小解析器：把查询字符串解析成 AST
- ✅ 基于第 26 课的 TSDB 接口实现执行器：Select → 取样本 → 计算
- ✅ 支持至少 4 类能力：指标选择、标签过滤、rate、sum 聚合
- ✅ 跑通端到端：输入 PromQL → 返回结果向量/曲线

---

## 27.1 为什么要做 PromQL 子集

PromQL 很强，但也很复杂。你实现子集的价值是：

- 把“查询语言”拆成可理解的模块：语法 → AST → 执行计划 → Storage 访问
- 亲手实现“先选 series，再取样本，再计算”的流程
- 为后续做 HTTP API/UI 打基础

本课目标不是“完整 PromQL”，而是能跑通一条最小闭环。

---

## 27.2 先选一个最小语法集合

建议你从下面这 4 类开始：

### 27.2.1 指标选择（Vector Selector）

- `up`
- `http_requests_total`

### 27.2.2 标签过滤（Label Matchers，最小只做 `=`）

- `up{job="prometheus"}`

### 27.2.3 rate(range-vector)

- `rate(http_requests_total[5m])`

### 27.2.4 sum 聚合（先做 by）

- `sum(rate(http_requests_total[5m])) by (job)`

先不做：

- join（on/ignoring/group_left）
- 子查询、offset、@、bool
- 正则匹配（=~ / !~）
- unless/and/or

---

## 27.3 设计你的 AST（最小可用）

建议 AST 至少包含 4 类节点：

```text
Expr =
  | VectorSelector(name, matchers, range?)    # up{job="x"}[5m]
  | Call(funcName, args)                      # rate(...)
  | Aggregate(op, expr, byLabels)             # sum(...) by (...)
  | NumberLiteral(value)                      # 可选：暂时不一定用
```

你不必追求和 Prometheus AST 一模一样，但要保证：

- 结构能表达你支持的语法
- 后续 evaluator 能递归求值

---

## 27.4 解析器：最小实现思路

为了简单，你可以用“两步走”：

1) **Tokenizer**：把字符串切成 token（identifier、`{` `}` `[` `]` `(` `)` `,` `=` 字符串字面量、duration）
2) **Parser**：按语法规则组装 AST

建议直接采用递归下降（recursive descent），因为你只支持少量语法。

### 27.4.1 最小语法（EBNF 近似）

```text
expr        := aggregate | call | vector
aggregate   := "sum" "(" expr ")" "by" "(" labelList ")"
call        := "rate" "(" vector ")"           # rate 参数必须是 range vector
vector      := ident matcher? range?
matcher     := "{" label "=" string "}"
range       := "[" duration "]"
labelList   := ident ("," ident)*
```

---

## 27.5 执行器：从 AST 到结果

你的执行器可以返回两种结果类型：

- Instant Vector：在某个评估时间点上的一组样本（series + value）
- Range Vector：每条 series 在时间范围内的一串样本（用于 rate）

你可以定义：

```text
InstantVector = []{labels, value}
RangeVector   = []{labels, samples[]}
```

### 27.5.1 VectorSelector 执行

- 用 matchers 调用 TSDB.Select 找到 series
- 若没有 range：
  - 取“评估时间点附近的最新值”（最简实现：取 end 前最后一个样本）
- 若有 range：
  - QueryRange(series, end-range, end) 返回样本窗口

### 27.5.2 rate 执行（最简公式）

对每条 range series：

```text
rate = (last.value - first.value) / (last.time - first.time)
```

注意：

- counter 可能 reset：最简版先忽略，或简单处理“出现下降则当作 reset”
- 时间单位：ms/s 要统一（建议统一用秒）

### 27.5.3 sum by 执行

- 对输入 instant vector 按 `byLabels` 做分组 key
- 把同组 value 相加
- 输出新的 instant vector（labels 只保留 byLabels）

---

## 27.6 评估时间：为什么要有 evalTime

PromQL 的执行不是“永远取最新”，而是在某个时间点上评估表达式。

你的最小实现也应该引入：

- `evalTime`：当前评估时刻
- range query 里每个 step 都会传不同 evalTime

这会让后续第 28 课做 HTTP API 与图表非常自然。

---

## 27.7 端到端验收：三条查询必须跑通

准备 3 条验收查询：

1) instant selector

```promql
up{job="prometheus"}
```

2) rate

```promql
rate(http_requests_total[5m])
```

3) sum by

```promql
sum(rate(http_requests_total[5m])) by (job)
```

如果你用第 25 课抓 Prometheus 自己，这些指标通常都有数据。

---

## 27.8 常见坑与修正

- labels 没排序导致分组 key 不稳定
- range 窗口太小导致样本点不足（rate 需要至少 2 个点）
- counter reset 没处理导致 rate 出现负数
- sum by 丢标签：输出标签只保留 byLabels（这是预期）

---

## 课后小结

- PromQL 子集实现的核心是：AST + evaluator + storage API
- evaluator 的核心流程永远是：先选 series，再取样本，再计算
- 先实现小集合跑通闭环，再逐步补语法与语义（正则匹配、二元运算、join…）


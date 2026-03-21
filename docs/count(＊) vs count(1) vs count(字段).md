---
tags:
  - ai总结
  - 八股文
  - 数据库
---

# count(＊) vs count(1) vs count(字段)

[count(＊) vs count(1) vs count(字段)](https://www.bilibili.com/video/BV1qb6EB2EGi?spm_id_from=333.788.videopod.episodes&vd_source=c7bea6fb9f92988f548051c4ed7de1e7&p=29)

## 讨论视角（这期在解决什么）

- 以“SQL 聚合统计的面试细节题”为视角，对比 `count(*)`、`count(1)`、`count(字段)` 三者的语义差异与性能误区
- 重点是：`count(字段)` 会排除 NULL；`count(*)` 与 `count(1)` 在 MySQL 中通常无本质性能差异

## 核心论点与分论点

### 核心论点

三者的核心区别在于“是否排除 NULL”和“表达方式差异”：`count(字段)` 只统计字段非 NULL 的行；`count(*)` 与 `count(1)` 都统计行数，在 MySQL 优化后两者没有实质性能差别。

### 分论点 1：语义差异（最重要）

- `count(*)`：统计行数（不关心具体列是否为 NULL）
- `count(1)`：统计行数（常量 1 对每行都成立）
- `count(字段)`：统计该字段非 NULL 的行数
  - 若业务需要“排除 NULL”，用 `count(字段)` 更符合语义

### 分论点 2：性能误区与官方结论

- 常见误区：网上常说“`count(*)` 更快”或“`count(1)` 更快”
- 视频口径：MySQL 官方说明 `count(*)` 与 `count(1)` 没有性能区别
- 原因（优化视角）：优化器会把 `count(1)` 等价处理为对存在性的计数标记，因此执行层面差别很小/无差别

### 分论点 3：如何验证（执行计划 + 优化后 SQL）

- 可以用 `EXPLAIN` 并配合查看优化信息（视频提到 `show warnings`）观察优化器对 SQL 的等价改写
- 结论导向：验证后通常能看到 `count(*)` 与 `count(1)` 的处理趋同

### 分论点 4：如果硬要比较

- 视频给出的“极限口径”：若强行比较，`count(1)` 可能略省去某些优化器改写步骤
- 但总体仍强调：两者不存在本质性能差距，别把优化重点放在这里

## 关键数据、案例与结论（逐条可复述）

- 示例现象：
  - `count(*)` 与 `count(1)` 返回相同的行数
  - `count(字段)` 会排除字段为 NULL 的行，因此结果可能更小
- 结论：
  - 统计行数：`count(*)` 或 `count(1)` 均可
  - 统计非 NULL：`count(字段)`
  - 性能焦点不应放在 `count(*)` vs `count(1)` 的“谁更快”上

## 思维导图（markmap）

```markmap
# count(*) vs count(1) vs count(字段)
## 题目关注点
### 语义差异（是否排除 NULL）
### 性能是否有差别（常见误区）
## 三者语义
### count(*)
#### 统计行数
#### 不排除 NULL
### count(1)
#### 统计行数（常量对每行成立）
#### 不排除 NULL
### count(字段)
#### 统计字段非 NULL 的行数
#### 排除 NULL
## 性能结论（视频口径）
### count(*) 与 count(1) 没有实质性能差别（官方说明）
### 优化器会等价处理（可用 EXPLAIN + show warnings 验证）
## 选型
### 要行数：count(*) / count(1)
### 要排除 NULL：count(字段)
```


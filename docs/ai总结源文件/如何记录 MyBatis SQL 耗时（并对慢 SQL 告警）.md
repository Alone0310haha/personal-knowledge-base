---
tags:
  - ai总结
  - 信息提炼
  - mybatis
---

# 如何记录 MyBatis SQL 耗时（并对慢 SQL 告警）

[如何记录 MyBatis SQL 耗时（并对慢 SQL 告警）](https://www.bilibili.com/video/BV1qb6EB2EGi?spm_id_from=333.788.videopod.episodes&vd_source=c7bea6fb9f92988f548051c4ed7de1e7&p=42)

## 讨论视角（这期在解决什么）

- 以“如何优雅记录 MyBatis SQL 耗时，并对超时 SQL 告警”作为目标，强调仅靠日志打印 SQL/参数并不能得到耗时
- 从 MyBatis 插件（Interceptor）机制切入：通过拦截底层核心组件的方法，在执行前后埋点统计耗时
- 用“只能拦截四大核心组件”作为边界条件，指导如何选对拦截点（记录耗时优先拦截 StatementHandler）

## 核心论点与分论点

### 核心论点

记录 MyBatis 每条 SQL 的耗时，最直接的工程做法是写一个 MyBatis 插件：实现 `Interceptor`，用 `@Intercepts/@Signature` 指定拦截 `StatementHandler.query(...)`（以及需要的话 `update(...)`），在 `invocation.proceed()` 前后记录时间差，并输出 SQL 与耗时；当耗时超过阈值时触发告警，从而在业务侧快速发现慢 SQL 并推动数据库优化。

### 分论点 1：现有日志只能看到 SQL/参数/返回量，看不到“用时”

- 常规做法：整合 MyBatis（或 MyBatis-Plus）日志后，可以在日志里看到执行的 SQL、参数、返回条数
- 痛点：日志不会默认输出“这一条 SQL 执行了多久”

### 分论点 2：MyBatis 插件在 Spring Boot 下的声明与注册方式

- 在 Spring Boot 中：
  - 新建一个类实现 `Interceptor`
  - 重写拦截方法（`intercept` 等）
  - 标记为 Spring 组件（例如 `@Component`），让其自动注册到 MyBatis 插件链

### 分论点 3：插件只能增强 MyBatis 的“四大核心类”

视频列举的可拦截点（四大核心组件）：

- Executor：执行数据库操作前的第一道入口，会先处理一级/二级缓存相关逻辑
- StatementHandler：真正执行 JDBC 语句的组件（执行 SQL 的关键位置）
- ParameterHandler：负责设置 SQL 参数（可用于参数修正/过滤敏感内容等）
- ResultSetHandler：负责处理查询结果（可用于结果加工）

结论：要统计 SQL 执行耗时，首选拦截 `StatementHandler`。

### 分论点 4：耗时统计的拦截点选择：query/update

- 查询耗时：拦截 `StatementHandler.query(...)`
  - 需要在 `@Signature` 里写清方法名与参数类型（即使没有重载也建议完整指定）
- 增删改耗时：拦截 `StatementHandler.update(...)`
  - 视频说明：增删改走 `update` 方法；如需统计也可追加一个 `@Signature`
- 视频侧重：通常慢 SQL 更常出现在查询，因此演示以 query 为主

### 分论点 5：实现逻辑：invocation.proceed 前后打点 + 可选告警

典型实现步骤（视频口径）：

- 记录开始时间 `start`
- 通过 `invocation.getTarget()` 拿到被增强的 `StatementHandler`
- 从 `statementHandler.getBoundSql()` 拿到 `BoundSql`，再 `getSql()` 获取目标 SQL 文本
- 调用 `invocation.proceed()` 执行真实数据库操作
- 执行结束记录 `end`，计算耗时 `cost = end - start`
- 输出：SQL + cost（以及必要的上下文信息）
- 告警扩展：如果 `cost > threshold`，发送告警（视频提到但未展开实现）

## 关键数据、案例与结论（逐条可复述）

- 核心组件范围：插件只允许增强 Executor / StatementHandler / ParameterHandler / ResultSetHandler
- 关键方法：
  - 查询：`StatementHandler.query(...)`
  - 增删改：`StatementHandler.update(...)`
- 输出目标：每条 SQL 的执行耗时；超阈值触发告警，提示需要优化 MySQL

## 思维导图（markmap）

```markmap
# 如何记录 MyBatis SQL 耗时（并对慢 SQL 告警）
## 目标
### 记录每条 SQL 的执行耗时
### 超阈值发送告警 -> 推动 MySQL 优化
## 痛点
### 仅配置日志：能看到 SQL/参数/返回量
### 但看不到耗时
## 方案：MyBatis 插件 Interceptor
### Spring Boot 下声明
#### 实现 Interceptor
#### @Component 自动注册到插件链
### 只能拦截四大核心类
#### Executor（缓存/执行入口）
#### StatementHandler（执行 SQL 的关键点）
#### ParameterHandler（设置参数，可做脱敏/修正）
#### ResultSetHandler（处理结果，可做结果加工）
## 拦截点选择
### 统计查询耗时：StatementHandler.query(...)
### 统计增删改耗时：StatementHandler.update(...)
### @Intercepts/@Signature 指定方法名 + 参数类型
## 统计逻辑
### start = now
### target = invocation.getTarget()
### sql = target.getBoundSql().getSql()
### invocation.proceed()
### cost = end - start
### log(sql, cost)
### if cost > threshold -> 告警
```


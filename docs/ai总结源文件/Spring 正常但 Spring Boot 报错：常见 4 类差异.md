---
tags:
  - ai总结
  - 信息提炼
---

# Spring 正常但 Spring Boot 报错：常见 4 类差异

[Spring 正常但 Spring Boot 报错：常见 4 类差异](https://www.bilibili.com/video/BV1qb6EB2EGi?spm_id_from=333.788.videopod.episodes&vd_source=c7bea6fb9f92988f548051c4ed7de1e7&p=41)

## 讨论视角（这期在解决什么）

- 从“同样的代码，在 Spring 正常、在 Spring Boot 报错”的现象切入，归因到 Spring Boot 对若干默认行为做了更严格的约束
- 用 4 个高频差异点解释：Spring Boot 为什么更容易在启动阶段失败，以及如何通过配置回退为 Spring 的宽松行为
- 结论落到工程实践：Spring Boot 倾向于“早失败（fail fast）”，逼迫开发者修复不合理设计或不确定配置

## 核心论点与分论点

### 核心论点

Spring Boot 在若干默认策略上比 Spring 更严格：遇到潜在歧义（同名 Bean）、潜在设计问题（循环依赖）、潜在配置缺失（@Value 取不到值）、以及 AOP 代理策略选择差异时，它更倾向于直接启动报错而不是“容忍并继续运行”。这些行为可以通过配置回退为 Spring 的默认方式，但更推荐理解差异并修正设计/配置。

### 分论点 1：同名 Bean 处理不同：Spring 覆盖，Spring Boot 报错

- 场景：存在多个 Bean，名称相同
- Spring：后定义的覆盖先定义的（“后面的会覆盖前面的”）
- Spring Boot：更严格，直接报错
- 处理：可通过配置让 Spring Boot 行为回退为 Spring（允许覆盖）

### 分论点 2：循环依赖处理不同：Spring 允许，Spring Boot 默认报错

- 场景：多个 Bean 相互依赖形成循环依赖
- Spring：允许这种依赖关系存在
- Spring Boot：更严格，默认直接报错
- 观点：Spring Boot 认为这是不合理设计
- 处理：可通过配置允许循环依赖（回退到 Spring 行为）

### 分论点 3：@Value 获取不到配置值的处理不同：Spring 给表达式原样，Spring Boot 报错

- 场景：用 `@Value` 读取配置文件中的值，但配置缺失/取不到
- Spring：将表达式本身当作值（原样字符串）继续运行
- Spring Boot：更严格，直接报错
- 建议：为容错性可在 `@Value` 中设置默认值（缺失则走默认值）；或在 Spring 中通过配置切到与 Spring Boot 一致的严格模式

### 分论点 4：AOP 动态代理默认策略不同：Spring JDK 优先，Spring Boot 默认 CGLIB

- Spring 默认：
  - 目标类实现接口：默认 JDK 动态代理
  - 未实现接口：使用 CGLIB
- Spring Boot 默认：直接默认改为 CGLIB
- 处理：可通过配置把 Spring Boot 默认代理方式切回 JDK 动态代理

## 关键数据、案例与结论（逐条可复述）

- 视频列举的 4 个差异点：
  - 同名 Bean：覆盖 vs 报错
  - 循环依赖：允许 vs 报错
  - `@Value` 缺失值：表达式原样 vs 报错
  - AOP 代理默认：JDK 优先 vs CGLIB 默认
- 总结结论：Spring Boot 更倾向于 fail fast；多数情况下都能用配置回退，但更推荐修正歧义与不合理设计

## 思维导图（markmap）

```markmap
# Spring 正常但 Spring Boot 报错：常见 4 类差异
## 总体原则
### Spring Boot 更严格（fail fast）
### 可通过配置回退为 Spring 行为，但更推荐修复设计/配置
## 1 同名 Bean
### Spring：后覆盖前
### Spring Boot：直接报错
### 处理：配置允许覆盖（回退）
## 2 循环依赖
### Spring：允许
### Spring Boot：默认报错（认为设计不合理）
### 处理：配置允许循环依赖（回退）
## 3 @Value 取不到配置值
### Spring：表达式原样作为值
### Spring Boot：报错
### 处理
#### @Value 设置默认值增强容错
#### 或 Spring 配置成严格模式（对齐 Boot）
## 4 AOP 动态代理默认策略
### Spring：有接口用 JDK，无接口用 CGLIB
### Spring Boot：默认 CGLIB
### 处理：配置切回 JDK
```


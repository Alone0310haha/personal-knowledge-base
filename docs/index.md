# Alone 的知识库
欢迎来到我的个人知识库：技术笔记、系统设计、工程实践与一些生活记录。

<div class="grid cards" markdown>

- :material-rocket-launch: **Prometheus 技术课**
  ---
  从监控理念、抓取、TSDB 到 PromQL 与告警。
  [进入目录](prometheus-tech/Prometheus 教程目录.md)

- :material-sitemap: **系统设计**
  ---
  支付、聊天、日志存储等。
  [支付系统设计](支付系统设计.md)
  [聊天系统 (WhatsApp) 系统设计](聊天系统 (WhatsApp) 系统设计.md)
  [分布式日志存储架构](如何设计分布式日志存储架构.md)

- :material-database: **数据库 / 存储**
  ---
  MySQL、索引、binlog/redo、分页与事务。
  [MySQL 一条 SELECT 的执行过程](<MySQL 一条 SELECT 的执行过程.md>)
  [B 树 vs B+ 树](<B 树 vs B+ 树 & InnoDB 为什么选 B+ 树.md>)
  [binlog 和 redo log](<binlog 和 redo log 缺一不可？.md>)
  [深分页优化](<深分页为什么慢，怎么优化？.md>)

- :material-redis: **Redis**
  ---
  排行榜、连续登录、缓存与分布式锁。
  [实时积分排行榜](redis 如何实现上亿用户实时积分排行榜.md)
  [连续登录天数](如何使用%20Redis%20记录上亿用户连续登录天数.md)
  [缓存击穿雪崩穿透](使用 redis 出现缓存击穿雪崩穿透应该怎么解决.md)
  [分布式锁选型](<分布式锁：Redis 还是 Zookeeper（p66）.md>)

- :material-spring: **Java / Spring**
  ---
  工程实践、并发、Spring 生态。
  [CopyOnWriteArrayList](Java 中的 CopyOnWriteArrayList 是什么.md)
  [Spring Boot 百万数据插入优化](<Spring Boot 百万数据插入优化.md>)
  [JPA vs MyBatis](<JPA vs MyBatis（架构视角）.md>)
  [Mockito 单测阅读方法](Mockito 单测阅读方法.md)

- :material-brain: **AI / LLM**
  ---
  Transformer、Embedding、RAG、微调。
  [GraphRAG](<AI 知识图谱 GraphRAG 是怎么回事？.md>)
  [Token 与 Embedding](<Token 与 Embedding：LLM 与 RAG 的数据处理机制.md>)
  [Transformer](<Transformer 如何成为 AI 模型的地基.md>)
  [LoRA](<LoRA 是什么 & 大模型微调怎么回事.md>)

</div>

## 使用方式

- 左侧栏支持目录导航；右上角支持全文搜索
- 页面内支持代码块一键复制与标题锚点链接
- 文章里支持 Obsidian 的 [[双向链接]] 语法

## 全部文章

??? note "展开/收起（完整索引）"
    - [Prometheus 教程目录](prometheus-tech/Prometheus 教程目录.md)
    - [Linux 文件权限模型详解](Linux 文件权限模型详解.md)
    - [支付系统设计](支付系统设计.md)
    - [聊天系统 (WhatsApp) 系统设计](聊天系统 (WhatsApp) 系统设计.md)
    - [如何设计分布式日志存储架构](如何设计分布式日志存储架构.md)
    - [jwt 如何通过双 token 保证安全和体验兼得](jwt 如何通过双 token 保证安全和体验兼得.md)
    - [Java 中的 CopyOnWriteArrayList 是什么](Java 中的 CopyOnWriteArrayList 是什么.md)
    - [redis 如何实现上亿用户实时积分排行榜](redis 如何实现上亿用户实时积分排行榜.md)
    - [如何使用 Redis 记录上亿用户连续登录天数](如何使用%20Redis%20记录上亿用户连续登录天数.md)
    - [使用 redis 出现缓存击穿雪崩穿透应该怎么解决](使用 redis 出现缓存击穿雪崩穿透应该怎么解决.md)
    - [了解陌生行业方法论](了解陌生行业方法论.md)
    - [内存 200M 读取 1G 文件并统计重复内容](内存 200M 读取 1G 文件并统计重复内容.md)
    - [UP 主是如何接商单的？视频质量 ai 分析](UP 主是如何接商单的？视频质量 ai 分析.md)
    - [2026 年政府工作报告总结（要点 + 对普通人的影响）](2026 年政府工作报告总结（要点 + 对普通人的影响）.md)
    - [2025 年政府工作报告总结（要点 + 对普通人的影响）](2025 年政府工作报告总结（要点 + 对普通人的影响）.md)
    - [证监会《关于短线交易监管的若干规定》解读（要点 + 对 A 股股民影响）](证监会《关于短线交易监管的若干规定》解读（要点 + 对 A 股股民影响）.md)
    - [99% 开发者踩过的 List 坑](<99% 开发者踩过的 List 坑.md>)
    - [B 树 vs B+ 树 & InnoDB 为什么选 B+ 树](<B 树 vs B+ 树 & InnoDB 为什么选 B+ 树.md>)
    - [BeanFactory vs ApplicationContext 核心区别](<BeanFactory vs ApplicationContext 核心区别.md>)
    - [BigDecimal 的 4 个致命陷阱（金融必看）](<BigDecimal 的 4 个致命陷阱（金融必看）.md>)
    - [binlog 刷盘机制](<binlog 刷盘机制.md>)
    - [binlog 和 redo log 缺一不可？](<binlog 和 redo log 缺一不可？.md>)
    - [count(＊) vs count(1) vs count(字段)](<count(＊) vs count(1) vs count(字段).md>)
    - [DDoS 是什么？如何防御（p54）](<DDoS 是什么？如何防御（p54）.md>)
    - [Dubbo 异步调用：为什么能提升吞吐（p63）](<Dubbo 异步调用：为什么能提升吞吐（p63）.md>)
    - [Dubbo 负载均衡策略选型（p62）](<Dubbo 负载均衡策略选型（p62）.md>)
    - [Git 如何修复线上突发 bug（本地还有未完成需求）](<Git 如何修复线上突发 bug（本地还有未完成需求）.md>)
    - [HTTP vs RPC 区别与选型（p55）](<HTTP vs RPC 区别与选型（p55）.md>)
    - [HTTP 长连接与短连接：慢在哪？Nginx 如何管理（p61）](<HTTP 长连接与短连接：慢在哪？Nginx 如何管理（p61）.md>)
    - [InnoDB Buffer Pool 工作过程（更新语句视角）](<InnoDB Buffer Pool 工作过程（更新语句视角）.md>)
    - [Java 手写一个简单消息队列（最简 MQ）](<Java 手写一个简单消息队列（最简 MQ）.md>)
    - [JPA vs MyBatis（架构视角）](<JPA vs MyBatis（架构视角）.md>)
    - [Mockito 单测阅读方法](Mockito 单测阅读方法.md)
    - [MVCC（多版本并发控制）如何实现无锁一致性读？](<MVCC（多版本并发控制）如何实现无锁一致性读？.md>)
    - [MySQL 一条 SELECT 的执行过程](<MySQL 一条 SELECT 的执行过程.md>)
    - [MySQL／InnoDB 如何用 Redo Log 做崩溃恢复](<MySQL／InnoDB 如何用 Redo Log 做崩溃恢复.md>)
    - [RestTemplate 如何优化连接池？](<RestTemplate 如何优化连接池？.md>)
    - [Spring Boot 百万数据插入优化](<Spring Boot 百万数据插入优化.md>)
    - [Spring Boot 配置怎么选：@Value ／ ConfigurationProperties ／ Environment](<Spring Boot 配置怎么选：@Value ／ ConfigurationProperties ／ Environment.md>)
    - [Spring Boot 配置敏感信息如何加密？](<Spring Boot 配置敏感信息如何加密？.md>)
    - [Spring 正常但 Spring Boot 报错：常见 4 类差异](<Spring 正常但 Spring Boot 报错：常见 4 类差异.md>)
    - [Spring 自动装配：名称 + 类型注入歧义如何解决](<Spring 自动装配：名称 + 类型注入歧义如何解决.md>)
    - [synchronized 怎么提升性能？](<synchronized 怎么提升性能？.md>)
    - [WebSocket 是什么？Nginx 如何支持？集群怎么做（p57）](<WebSocket 是什么？Nginx 如何支持？集群怎么做（p57）.md>)
    - [WebSocket 是什么？Nginx 如何支持？集群怎么做（p59）](<WebSocket 是什么？Nginx 如何支持？集群怎么做（p59）.md>)
    - [为什么大厂弃用 MyBatis 二级缓存](<为什么大厂弃用 MyBatis 二级缓存.md>)
    - [分布式事务：是什么？有哪些方案？怎么选（p65）](<分布式事务：是什么？有哪些方案？怎么选（p65）.md>)
    - [分布式锁：Redis 还是 Zookeeper（p66）](<分布式锁：Redis 还是 Zookeeper（p66）.md>)
    - [分库分表 ID 冲突解决方案](<分库分表 ID 冲突解决方案.md>)
    - [别再误解：Spring 的单例是容器视角，不是 JVM 视角](<别再误解：Spring 的单例是容器视角，不是 JVM 视角.md>)
    - [前模糊索引失效（LIKE '%xxx'）](<前模糊索引失效（LIKE '%xxx'）.md>)
    - [单表最多数据量需要分表？](<单表最多数据量需要分表？.md>)
    - [如何记录 MyBatis SQL 耗时（并对慢 SQL 告警）](<如何记录 MyBatis SQL 耗时（并对慢 SQL 告警）.md>)
    - [如何防止 Spring Boot 反编译？](<如何防止 Spring Boot 反编译？.md>)
    - [查询 200 条数据耗时 200ms 如何在 500ms 内查询 1000 条](<查询 200 条数据耗时 200ms 如何在 500ms 内查询 1000 条.md>)
    - [海量数据分页：Limit／Offset 为什么会挂，怎么做才稳（p53）](<海量数据分页：Limit／Offset 为什么会挂，怎么做才稳（p53）.md>)
    - [深分页为什么慢，怎么优化？](<深分页为什么慢，怎么优化？.md>)
    - [给你一个亿 Redis Keys，如何统计双方的共同好友](<给你一个亿 Redis Keys，如何统计双方的共同好友.md>)
    - [聚集索引 vs 非聚集索引](<聚集索引 vs 非聚集索引.md>)
    - [设计模式在业务中怎么用？（支付案例）](<设计模式在业务中怎么用？（支付案例）.md>)
    - [跨域是什么？如何解决（p56）](<跨域是什么？如何解决（p56）.md>)
    - [零拷贝是什么？如何减少 CPU 拷贝与上下文切换（p58）](<零拷贝是什么？如何减少 CPU 拷贝与上下文切换（p58）.md>)
    - [零拷贝是什么？如何减少 CPU 拷贝与上下文切换（p60）](<零拷贝是什么？如何减少 CPU 拷贝与上下文切换（p60）.md>)
    - [AI 知识图谱 GraphRAG 是怎么回事？](AI 知识图谱 GraphRAG 是怎么回事？.md)
    - [AI 编程实践：从 Claude Code 实践到团队协作的优化思考](AI 编程实践：从 Claude Code 实践到团队协作的优化思考.md)
    - [count(＊) vs count(1) vs count(字段)](<count(＊) vs count(1) vs count(字段).md>)
    - [LoRA 是什么 & 大模型微调怎么回事](LoRA 是什么 & 大模型微调怎么回事.md)
    - [Token 与 Embedding：LLM 与 RAG 的数据处理机制](Token 与 Embedding：LLM 与 RAG 的数据处理机制.md)
    - [Transformer 如何成为 AI 模型的地基](Transformer 如何成为 AI 模型的地基.md)
    - [上线 2 年的限流代码被打爆：滑动窗口 + Redis 脚本的坑](上线 2 年的限流代码被打爆：滑动窗口 + Redis 脚本的坑.md)
    - [原来我，一点都不懂 Git](原来我，一点都不懂 Git.md)
    - [多头注意力 Multi-Head Attention 机制详解](多头注意力 Multi-Head Attention 机制详解.md)
    - [大模型训练原理：梯度下降，从一条直线讲起](大模型训练原理：梯度下降，从一条直线讲起.md)
    - [如何拆解学习开源项目](如何拆解学习开源项目.md)
    - [美团 or 饿了么"找出附近 1km 的商家"](美团 or 饿了么"找出附近 1km 的商家".md)
    - [新加坡 20 小时轻松穷游攻略（4 月 5 日）](新加坡 20 小时轻松穷游攻略（4 月 5 日）.md)

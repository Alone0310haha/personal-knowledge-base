### Elasticsearch 系统学习：从单机到集群的 10 节课主线大纲

这套安排按“先把单机用顺（能建模、能写、能查、能算、能调）→ 再把集群跑稳（能扩容、能容灾、能运维）→ 最后上生产级最佳实践”的链路组织。

下面给每一课补上“最短官方文档学习路径（建议顺序）”，每课只保留 2-3 个最值得先看的链接，优先按“概念页 -> API / 专题页”的顺序阅读，这样和课程主线更一致。

#### 第 1 课：单机启动与 ES 作为“搜索 + 分析引擎”的定位

- **最短官方文档学习路径（建议顺序）**：
    - [What is Elasticsearch?](https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro-what-is-es.html)
    - [Run Elasticsearch locally](https://www.elastic.co/guide/en/elasticsearch/reference/current/run-elasticsearch-locally.html)
    - [Index fundamentals](https://www.elastic.co/docs/manage-data/data-store/index-basics)

- **学习目标**：把 ES 在单机上跑起来，明确它解决的问题边界与核心对象。
- **核心内容**：
    - ES 能力边界：搜索（全文检索）、分析（聚合）、近实时（NRT）。
    - 最小心智模型：Index / Document / Field，与关系型数据库的相似与差异。
    - 单机安装与基础健康检查：Kibana Dev Tools / `_cluster/health` / `_cat` 入门。
- **课程问题（学完要能回答）**：
    - ES 的定位是什么？“近实时”是什么意思，为什么不是强实时？
    - ES 与 MySQL 的关键差异是什么（数据模型、查询方式、扩展方式）？
    - Index / Document / Field 分别是什么？它们与库/表/行/列如何类比，哪里不能类比？
    - 如何用最少的 API 判断 ES 是否可用（健康、节点、索引）？

#### 第 2 课：单机数据写入：文档 CRUD、批量写入与幂等

- **最短官方文档学习路径（建议顺序）**：
    - [Manage data using APIs](https://www.elastic.co/docs/manage-data/data-store/manage-data-from-the-command-line)
    - [Document APIs](https://www.elastic.co/docs/api/doc/elasticsearch/v8/group/endpoint-document)
    - [Bulk API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-bulk)

- **学习目标**：能把数据稳定写进 ES，并理解写入的幂等、冲突与吞吐。
- **核心内容**：
    - 单文档 CRUD：`index/get/delete/update` 的语义与响应结构。
    - ID 设计：业务主键 vs 自动 ID；幂等、去重、覆盖更新的取舍。
    - 批量写入：`_bulk` 格式、部分失败处理与重试策略的基本原则。
- **课程问题（学完要能回答）**：
    - `PUT index/_doc/id`、`POST index/_doc`、`POST index/_update/id` 分别适合什么场景？
    - 为什么 ID 设计会影响“是否幂等”“是否容易去重”和写入性能？
    - `_bulk` 的 action/metadata + source 是什么结构？如何定位部分失败并做可重试的幂等处理？
    - `_version`/并发冲突通常怎么产生？有哪些常见应对（重试、业务补偿）？

#### 第 3 课：单机建模 1：Mapping 设计、dynamic 风险与字段治理

- **最短官方文档学习路径（建议顺序）**：
    - [Mapping](https://www.elastic.co/docs/manage-data/data-store/mapping)
    - [Dynamic mapping](https://www.elastic.co/docs/manage-data/data-store/mapping/dynamic-mapping)
    - [Dynamic templates](https://www.elastic.co/docs/manage-data/data-store/mapping/dynamic-templates)

- **学习目标**：把“字段类型选对”这件事做扎实，避免后期返工。
- **核心内容**：
    - Dynamic mapping 的便利与风险（类型误判、字段爆炸）。
    - 显式 mapping 的基本写法：数值/日期/布尔/对象等常见类型。
    - 字段治理：禁用/限制动态字段、字段命名、避免高基数字段滥用。
- **课程问题（学完要能回答）**：
    - 为什么生产更偏向显式 mapping？dynamic 的典型事故有哪些？
    - 日期/数值字段如何建模？时区、格式、范围查询会踩哪些坑？
    - 什么是字段爆炸？如何通过动态模板/限制字段数等手段规避？

#### 第 4 课：单机建模 2：文本分析链路与中文分词

- **最短官方文档学习路径（建议顺序）**：
    - [Text analysis](https://www.elastic.co/docs/manage-data/data-store/text-analysis)
    - [Text field type](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/text)
    - [Multi-fields](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/multi-fields)

- **学习目标**：理解“索引时怎么切词、查询时怎么切词”，能解释召回/误召回问题。
- **核心内容**：
    - `text` vs `keyword`：全文检索与精确匹配的分工；multi-fields 的常用模式。
    - 分析器链路：字符过滤器 → 分词器 → token 过滤器；索引/查询分析器差异。
    - 中文分词（如 IK）的使用思路与词典维护注意点；同义词的收益与风险。
- **课程问题（学完要能回答）**：
    - 为什么 `text` 不适合做精确过滤/聚合？为什么 `keyword` 不适合全文检索？
    - analyzer 在索引阶段与查询阶段分别做了什么？分词不一致会导致什么现象？
    - multi-fields 解决了什么问题？典型 mapping 怎么写、怎么用？

#### 第 5 课：单机查询 1：Query DSL 基础（检索 + 过滤）

- **最短官方文档学习路径（建议顺序）**：
    - [Query DSL](https://www.elastic.co/docs/explore-analyze/query-filter/languages/querydsl)
    - [Search API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-search)
    - [Bool query](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-bool-query)

- **学习目标**：能写出覆盖 80% 业务的检索 DSL，并把“相关性”和“过滤”分清楚。
- **核心内容**：
    - Query context vs Filter context：评分、缓存与性能含义。
    - 基础查询：`match/term/range/exists/prefix` 的使用边界。
    - `bool` 组合：`must/should/must_not/filter` 与 `minimum_should_match`。
- **课程问题（学完要能回答）**：
    - `match` 与 `term` 的根本差异是什么？为什么字段类型会决定查询写法？
    - 过滤为什么通常放到 `filter`？它对评分和性能有什么影响？
    - `should` 在有/没有 `must` 时默认行为有什么不同？什么时候要 `minimum_should_match`？

#### 第 6 课：单机查询 2：排序、分页、高亮与深分页替代方案

- **最短官方文档学习路径（建议顺序）**：
    - [Sort search results](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/sort-search-results)
    - [Paginate search results](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/paginate-search-results)
    - [Highlighting](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/highlighting)

- **学习目标**：能把“用户看到的搜索体验”做完整：排序、分页、高亮，并避开深分页坑。
- **核心内容**：
    - 排序与字段：`sort` 对性能/内存的影响；为什么排序字段通常用 `keyword/numeric/date`。
    - 分页策略：`from/size` 的机制与深分页问题；`search_after` 的使用前提。
    - 高亮：基本原理与开销来源，何时该用/不该用。
- **课程问题（学完要能回答）**：
    - 为什么 `from/size` 深分页会越来越慢？瓶颈在哪里？
    - `search_after` 需要哪些前提（稳定排序字段、唯一性）？适合什么场景？
    - 高亮为什么可能显著增大查询开销？如何控制成本？

#### 第 7 课：单机分析：Aggregations（从统计到报表）

- **最短官方文档学习路径（建议顺序）**：
    - [Aggregations](https://www.elastic.co/docs/explore-analyze/query-filter/aggregations)
    - [Terms aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html)
    - [Date histogram aggregation](https://www.elastic.co/docs/reference/aggregations/search-aggregations-bucket-datehistogram-aggregation)

- **学习目标**：把 ES 的“分析能力”用起来，能做常见统计报表并读懂返回结构。
- **核心内容**：
    - Metrics：`sum/avg/min/max/value_count/cardinality`。
    - Buckets：`terms/range/date_histogram`；子聚合表达“先分桶再算指标”。
    - 聚合性能：桶数量控制、`size/shard_size`、字段类型与 `doc_values` 的关系。
- **课程问题（学完要能回答）**：
    - `terms` 聚合为何要求 `keyword`（或可聚合字段）？`text` 为什么不行？
    - `cardinality` 是精确还是近似？它适合解决什么问题？
    - `terms` 的 `size/shard_size/order` 会如何影响准确性与性能？

#### 第 8 课：单机内核视角：写入/可见性/持久化的真实链路

- **最短官方文档学习路径（建议顺序）**：
    - [Near real-time search](https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html)
    - [Translog settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-translog.html)
    - [Flush API](https://www.elastic.co/docs/api/doc/elasticsearch/v9/operation/operation-indices-flush)

- **学习目标**：从内核角度解释“写入成功后为何不立刻可搜”“为什么 refresh/flush 影响性能”。
- **核心内容**：
    - 写入链路：buffer / translog / segment 的角色分工。
    - Refresh vs Flush：可搜索性 vs 持久化；NRT 的工作方式。
    - 段合并（merge）与写入/查询抖动；基本的调优抓手（`refresh_interval` 等）。
- **课程问题（学完要能回答）**：
    - refresh 与 flush 的区别是什么？分别解决什么问题？
    - translog 为什么存在？它对数据安全与性能意味着什么？
    - 为什么会发生 merge？merge 对线上查询/写入会产生什么影响？

#### 第 9 课：从单机到集群：分片/副本/路由与一次请求的分布式路径

- **最短官方文档学习路径（建议顺序）**：
    - [Clusters, nodes, and shards](https://www.elastic.co/docs/deploy-manage/distributed-architecture/clusters-nodes-shards)
    - [Shard allocation, relocation, and recovery](https://www.elastic.co/docs/deploy-manage/distributed-architecture/shard-allocation-relocation-recovery)
    - [Reading and writing documents](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-replication.html)

- **学习目标**：把“扩容与容灾”的核心机制讲清楚，能推导查询/写入在集群中的流转。
- **核心内容**：
    - 分片与副本：容量、并行、容灾；主分片数为何不可随意改。
    - 路由与热点：`shard = hash(routing) % primary_shards` 的含义；自定义 routing 的收益与风险。
    - 节点角色与请求路径：协调节点、数据节点、主节点；搜索的 scatter/gather。
- **课程问题（学完要能回答）**：
    - 为什么需要分片？为什么需要副本？它们分别对读性能与高可用的影响是什么？
    - 默认 routing 是什么？什么时候要自定义 routing？如何避免热点分片？
    - 一次搜索请求在集群中的基本流程是什么（协调 → 分片并行 → 合并返回）？
    - 主节点的职责是什么？为什么需要选主与集群状态（元数据）管理？

#### 第 10 课：集群上生产：容量规划、运维监控、生命周期与最佳实践清单

- **最短官方文档学习路径（建议顺序）**：
    - [Size your shards](https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance/size-shards)
    - [Red or yellow cluster health status](https://www.elastic.co/docs/troubleshoot/elasticsearch/red-yellow-cluster-status)
    - [Index lifecycle management (ILM)](https://www.elastic.co/docs/manage-data/lifecycle/index-lifecycle-management)

- **学习目标**：能把集群跑稳、跑省、可恢复：监控、扩缩容、备份与权限到位。
- **核心内容**：
    - 容量与分片规划：数据量/读写比例/节点数推导；避免过多小分片。
    - 运维监控与排障：`_cluster/health`、`_cat/shards`；未分配分片常见原因（磁盘水位、节点离线、限制策略）。
    - 生命周期与成本：ILM（rollover/迁移/删除）、冷热分层思路。
    - 安全与灾备：Snapshot/Restore 的原则；RBAC 最小权限。
    - 反模式与红线：字段爆炸、深分页、滥用 nested、无 ILM、无快照、过量小索引等。
- **课程问题（学完要能回答）**：
    - 如何从 `_cluster/health` 判断集群状态？green/yellow/red 分别意味着什么行动？
    - 如何用 `_cat/shards` 定位未分配分片与分片倾斜？常见根因与处理步骤是什么？
    - ILM 的核心价值是什么？如何把索引按时间滚动并自动迁移/删除？
    - Snapshot 能备份哪些内容？为什么不建议直接拷贝 data 目录？Restore 有哪些风险点？
    - 如何做一份“可落地”的上线检查清单（建模、写入、查询、分片、监控、备份、权限）？

### Elasticsearch 系统学习：从单机到集群的 13 节课主线大纲

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
    - [Nested field type](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/nested)（`object` 数组被扁平化导致跨字段条件”错位”的本质原因，以及 `nested` 如何将每个数组元素存为独立隐藏文档）

- **学习目标**：把”字段类型选对”这件事做扎实，避免后期返工。
- **核心内容**：
    - Dynamic mapping 的便利与风险（类型误判、字段爆炸）。
    - 显式 mapping 的基本写法：数值/日期/布尔/对象等常见类型。
    - 字段治理：禁用/限制动态字段、字段命名、避免高基数字段滥用。
    - `object` vs `nested` 的本质差异：`object` 数组在索引时被扁平化，跨字段组合条件会”错位”；`nested` 将每个数组元素存为独立隐藏文档，保留元素内部的字段关联。
    - `nested` 的查询语法（`nested` query）与聚合写法；写入放大代价（更新一个元素等于重建整条父文档）；何时该用 `nested`、何时考虑其他数据建模方案。
- **课程问题（学完要能回答）**：
    - 为什么生产更偏向显式 mapping？dynamic 的典型事故有哪些？
    - 日期/数值字段如何建模？时区、格式、范围查询会踩哪些坑？
    - 什么是字段爆炸？如何通过动态模板/限制字段数等手段规避？
    - `object` 数组扁平化会导致什么查询结果异常？举一个具体例子说明何时必须用 `nested`。
    - `nested` 文档底层如何存储？为什么更新 `nested` 数组中的一个元素代价高昂？

#### 第 4 课：索引管理：创建、模板、别名与零停机变更

- **最短官方文档学习路径（建议顺序）**：
    - [Index templates](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-templates.html)
    - [Aliases](https://www.elastic.co/guide/en/elasticsearch/reference/current/aliases.html)
    - [Reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html)

- **学习目标**：能在生产环境中规范创建索引、用模板统一管理 mapping 与 settings、用别名实现零停机 mapping 变更。
- **核心内容**：
    - 显式创建索引：`PUT /index` 同时指定 `settings`（`number_of_shards`、`refresh_interval`）与 `mappings`；避免依赖 auto-create 带来的隐患。
    - Component Templates + Index Templates：模板优先级与继承关系；让新索引自动套用规范的 mapping 和 settings。
    - Index Aliases：别名作为”索引指针”；写别名（`is_write_index`）与读别名的分工；原子切换（`_aliases` 批操作）。
    - Reindex API：mapping 变更后的数据迁移；结合别名的零停机切换标准步骤（建新索引 → reindex → 切换别名 → 删旧索引）。
    - 动态调整 Index Settings：批量写入优化（关副本、提升 `refresh_interval`）；写完还原；哪些 settings 不可动态修改。
- **课程问题（学完要能回答）**：
    - 为什么推荐显式创建索引而非依赖 auto-create？后者会带来哪些隐患？
    - Component Template 和 Index Template 分别管什么？优先级如何叠加？
    - 如何用别名 + Reindex 完成零停机 mapping 变更？切换期间新写入如何保证不丢？
    - `number_of_shards` 和 `number_of_replicas` 哪个可以动态修改？为什么主分片数不可改？
    - 批量导入数据时常用哪些 settings 临时优化？导入完成后需要还原哪些？

#### 第 5 课：写入预处理：Ingest Pipeline 与数据清洗

- **最短官方文档学习路径（建议顺序）**：
    - [Ingest pipelines](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html)
    - [Processors reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/processors.html)
    - [Simulate pipeline API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ingest-simulate)

- **学习目标**：能在文档写入时自动完成字段提取、格式转换和数据清洗，不依赖外部 ETL 工具。
- **核心内容**：
    - Pipeline 模型：写入请求经过 Processor 链式处理后再存入索引；Pipeline 在写入链路中的位置（Ingest Node 阶段）。
    - 常用 Processor：`grok`（正则提取非结构化文本）、`date`（时间字符串解析与时区处理）、`convert`（类型转换）、`rename/remove/set`（字段操作）。
    - Pipeline 与写入的绑定：`_bulk`/`_doc` 请求指定 `pipeline` 参数；索引默认 pipeline（`default_pipeline`）。
    - Simulate API：在不写入数据的情况下测试 pipeline 对文档的处理结果；调试多个 Processor 的逐步输出。
- **课程问题（学完要能回答）**：
    - Pipeline 在写入链路的哪个阶段执行？它与 Logstash 的定位有什么区别，各自适合什么场景？
    - `grok` 如何从非结构化日志中提取字段？正则匹配失败时默认行为是什么，如何设置 `ignore_failure`？
    - 如何为索引设置 `default_pipeline`，使所有写入请求自动经过处理？
    - 如何用 Simulate API 调试一个有多个 Processor 的 pipeline？

#### 第 6 课：单机建模 2：文本分析链路与中文分词

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

#### 第 7 课：单机查询 1：Query DSL 基础（检索 + 过滤）

- **最短官方文档学习路径（建议顺序）**：
    - [Query DSL](https://www.elastic.co/docs/explore-analyze/query-filter/languages/querydsl)
    - [Search API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-search)
    - [Bool query](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-bool-query)

- **学习目标**：能写出覆盖 80% 业务的检索 DSL，并把“相关性”和“过滤”分清楚。
- **核心内容**：
    - Query context vs Filter context：评分、缓存与性能含义。
    - 基础查询：`match/term/range/exists/prefix` 的使用边界。
    - `bool` 组合：`must/should/must_not/filter` 与 `minimum_should_match`。
    - 查询诊断：`_explain` API（查看单条文档评分明细与召回路径）、`profile` API（分析查询各阶段耗时）、slowlog（索引级别慢查询配置与日志输出）。
- **课程问题（学完要能回答）**：
    - `match` 与 `term` 的根本差异是什么？为什么字段类型会决定查询写法？
    - 过滤为什么通常放到 `filter`？它对评分和性能有什么影响？
    - `should` 在有/没有 `must` 时默认行为有什么不同？什么时候要 `minimum_should_match`？
    - `_explain` 与 `profile` API 各自解决什么问题？什么情况下该先用哪一个？
    - 如何为索引配置 slowlog 阈值？slowlog 输出在哪里查看，记录了哪些关键信息？

#### 第 8 课：单机查询 2：排序、分页、高亮与深分页替代方案

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

#### 第 9 课：相关性评分：BM25 原理与评分调优

- **最短官方文档学习路径（建议顺序）**：
    - [Relevance scores](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html)
    - [Explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-explain)
    - [Function score query](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-function-score-query)

- **学习目标**：能解释 ES 默认评分的来源，能用 `_explain` 定位召回与排序异常，能通过 `function_score` 在业务层干预评分。
- **核心内容**：
    - BM25 原理：TF（词频）、IDF（逆文档频率）、字段长度归一化三要素的直觉含义；与经典 TF-IDF 的主要区别。
    - `_explain` API：逐层查看单条文档的评分明细；诊断"为什么这条文档排第一"或"为什么没有被召回"。
    - `boost`：查询级别与字段级别的权重调整；`boost` 对最终评分的实际影响方式。
    - `function_score`：在原始查询评分基础上叠加业务信号；常用函数 `field_value_factor`（热度加权）、`decay`（时间/距离衰减）、`random_score`（随机打散）；`score_mode` 与 `boost_mode` 的组合含义。
    - `constant_score`：绕过评分计算，适合纯过滤场景的查询包装。
- **课程问题（学完要能回答）**：
    - BM25 中 TF、IDF 和字段长度归一化分别解决了什么问题？为什么词频越高不一定得分越高？
    - 如何用 `_explain` API 诊断一条文档得分异常或没有被召回的根因？
    - `boost` 放在 `query` 子句里和放在字段 `mapping` 的 `boost` 参数上有什么区别？
    - `function_score` 的 `score_mode` 和 `boost_mode` 分别控制什么？举例说明如何叠加业务热度信号。
    - 什么时候应该用 `constant_score` 而不是直接将条件放进 `filter`？

#### 第 10 课：单机分析：Aggregations（从统计到报表）

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

#### 第 11 课：单机内核视角：写入/可见性/持久化的真实链路

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

#### 第 12 课：从单机到集群：分片/副本/路由与一次请求的分布式路径

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

#### 第 13 课：集群上生产：容量规划、运维监控、生命周期与最佳实践清单

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
    - Data Streams：时序/日志场景的现代索引方案；backing indices + write index 的数据模型；与 ILM rollover 结合的标准方式；”Data Stream + ILM + Component Template”三件套的配置路径。
    - 反模式与红线：字段爆炸、深分页、滥用 nested、无 ILM、无快照、过量小索引等。
- **课程问题（学完要能回答）**：
    - 如何从 `_cluster/health` 判断集群状态？green/yellow/red 分别意味着什么行动？
    - 如何用 `_cat/shards` 定位未分配分片与分片倾斜？常见根因与处理步骤是什么？
    - ILM 的核心价值是什么？如何把索引按时间滚动并自动迁移/删除？
    - Snapshot 能备份哪些内容？为什么不建议直接拷贝 data 目录？Restore 有哪些风险点？
    - Data Stream 和直接用 rollover 别名有什么区别？什么场景下应该优先用 Data Stream？
    - 如何做一份”可落地”的上线检查清单（建模、写入、查询、分片、监控、备份、权限）？

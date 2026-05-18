# Elasticsearch（ES）是什么（自测题答案解析）

## Q1. Elasticsearch 的定位是什么？为什么很多系统是“MySQL 作为主库 + ES 作为旁路索引”？
题目复述：Elasticsearch 的定位是什么？为什么很多系统是“MySQL 作为主库 + ES 作为旁路索引”？

先给结论：ES 是搜索与分析引擎，不是强事务主库；主库负责一致性与事务，ES 负责检索体验与聚合分析，因此用“主库 + 旁路索引”更稳。

理解路径：主库（如 MySQL）擅长强一致事务、关系约束与更新；ES 擅长倒排索引驱动的全文检索与分布式聚合。把 ES 作为旁路索引，可以在不牺牲主库语义的前提下获得更好的搜索能力。数据通常通过 CDC/MQ 从主库同步到 ES，所以整体语义更接近最终一致。

关键要点：
- 选型核心：事务一致性与关系约束交给主库，检索与分析交给 ES。
- 同步链路决定一致性：主库→ES 的延迟与失败补偿决定“最终一致”的质量。
- ES 适合“按查询建模”：字段类型、分词、索引结构直接影响体验与成本。

常见误答：
- “ES 就是更快的数据库，可以替代 MySQL”：忽略事务/一致性/更新成本/关系建模边界。

## Q2. 倒排索引是什么？它为什么能让全文检索更快？
题目复述：倒排索引是什么？它为什么能让全文检索更快？

先给结论：倒排索引是“词 → 文档列表”的索引结构；查询时先定位词对应的文档集合，再做过滤与评分，比扫描文本快得多。

理解路径：全文检索的关键成本在“从大量文本里找包含某些词的文档”。倒排索引把每个词出现在哪些文档里提前组织好（并记录位置、频次等信息），因此查询是“查索引 + 合并集合”，而不是“逐行/逐文档扫描内容”。

关键要点：
- 倒排索引的基本形态：term → posting list（docIDs、positions、freq 等）。
- 相关性排序依赖索引信息：位置、词频等为评分与短语匹配提供依据。
- analyzer 决定 term：同一文本用不同分词/归一化会产生完全不同的 term 集合。

常见误答：
- “倒排索引就是把数据反着存”：没有说清 term 与 posting list，以及为什么能减少扫描。

## Q3. text 与 keyword 有什么区别？各自适合什么查询与场景？
题目复述：text 与 keyword 有什么区别？各自适合什么查询与场景？

先给结论：text 会被分词用于全文检索；keyword 不分词用于精确匹配、聚合与排序。

理解路径：text 字段走 analyzer，把字符串拆成多个 term；查询时通常用 match 等全文检索语义。keyword 字段把整体当作一个 term，适合 term 查询、聚合（terms aggs）、排序与去重等场景。

关键要点：
- text：match、multi_match、相关性评分、高亮更常见。
- keyword：term、terms、聚合、排序、精确过滤更常见。
- 多字段常用策略：同一字段同时提供 text 与 keyword 子字段以兼顾检索与聚合。

常见误答：
- “keyword 比 text 快，所以都用 keyword”：会丢失分词检索能力，中文更明显。

## Q4. match 查询与 term 查询有什么区别？最常见的误用是什么？
题目复述：match 查询与 term 查询有什么区别？最常见的误用是什么？

先给结论：match 会按字段 analyzer 分词后再查询；term 不分词做精确匹配。误用通常是用 term 查 text 或用 match 查 keyword 导致匹配不符合预期。

理解路径：match 适合“人输入的自然语言”，它会把查询字符串按 analyzer 处理成多个 term，再做相关性检索。term 适合“结构化值”的精确过滤，比如状态码、用户 id、标签等。

关键要点：
- text 字段：优先 match/multi_match；keyword 子字段用于过滤与聚合。
- keyword 字段：term/terms 用于精确过滤；match 可能引入不必要的分词逻辑。
- 诊断工具：用 `_analyze` 看分词，用 `_explain` 看为什么匹配/不匹配。

常见误答：
- “match 更模糊，term 更精确”但不说明 analyzer 与字段类型的关系。

## Q5. ES 为什么是“近实时”？refresh 在里面起什么作用？
题目复述：ES 为什么是“近实时”？refresh 在里面起什么作用？

先给结论：写入先落到内存缓冲与 translog，不会立刻生成可搜索的 segment；refresh 会把缓冲内容变成新的 segment 并打开搜索视图，所以搜索可见性有延迟。

理解路径：Lucene 的索引是按 segment 组织的，可搜索通常依赖 segment 被“刷新并打开”。ES 通过周期性 refresh 达到近实时：写入很快返回，但要等下一次 refresh 才能被搜索到。

关键要点：
- ack 成功与可搜索不是同一件事：可搜索依赖 refresh。
- refresh 频率越高，越“实时”，但写入成本与资源压力会更大。
- 对强实时检索要求可用 `refresh=wait_for`，需要权衡吞吐与延迟。

常见误答：
- “写入成功就一定立刻能搜到”：忽略 refresh 语义与 NRT。

## Q6. 一次写入请求从客户端到数据落盘大致经历了哪些步骤？translog 的作用是什么？
题目复述：一次写入请求从客户端到数据落盘大致经历了哪些步骤？translog 的作用是什么？

先给结论：写入由协调节点路由到主分片，主分片写入缓冲并记录 translog，再复制到副本分片；translog 用于崩溃恢复与保证持久性边界。

理解路径：ES 的写入需要分布式一致性：主分片决定写入顺序与结果，然后复制到副本确保高可用。写入的数据并不一定立刻进入可搜索 segment，但 translog 记录了操作，节点崩溃后可以根据 translog 回放恢复到一致状态。

关键要点：
- 路由：根据 `_id`（或 routing）定位到具体主分片。
- 复制：主分片成功后同步/异步复制到副本，副本用于 HA 与读扩展。
- translog：写前日志，配合 flush/commit 控制数据落盘与恢复点。

常见误答：
- “translog 就是 redo log，完全等价”：类比可以，但需要说明 ES 的 refresh/flush/segment 与数据库不同。

## Q7. 一次 search 请求为什么通常要走 query phase 与 fetch phase？
题目复述：一次 search 请求为什么通常要走 query phase 与 fetch phase？

先给结论：query phase 用倒排索引找 topK 候选并计算评分；fetch phase 再按 docID 拉取 `_source` 等字段并拼装返回，避免一开始就读大量文档内容。

理解路径：查询时最关键的是用索引快速定位候选并排序，这阶段不需要把文档全文都读出来。先拿到 docID 列表后，再只读取需要返回的那部分文档内容，整体 IO 与网络开销更可控。

关键要点：
- query phase：并行到分片，返回候选 docIDs、scores、局部聚合。
- fetch phase：拉取 `_source`、stored fields 等，组装最终 hits。
- 聚合 reduce：协调节点汇总分片结果，完成全局排序与聚合合并。

常见误答：
- “两阶段是为了提高一致性”：核心更多是性能与分布式汇总模式。

## Q8. shard 与 replica 各解决什么问题？副本对读写性能分别有什么影响？
题目复述：shard 与 replica 各解决什么问题？副本对读写性能分别有什么影响？

先给结论：shard 解决数据拆分与并行计算；replica 解决高可用与读扩展。副本一般提升读吞吐，但会增加写入成本。

理解路径：分片是把一个 index 切成多个物理分区，写入与查询可以并行；副本是每个主分片的拷贝，用于容灾与分担读请求。写入需要复制到副本，因此副本越多，写放大越明显。

关键要点：
- shard 数影响：并行度、容量上限、恢复速度与管理成本。
- replica 数影响：读吞吐与 HA，但写入延迟与资源占用增加。
- 读路由：查询可以命中主或副本分片副本（按策略选择）。

常见误答：
- “副本越多，查询一定越快”：忽略协调开销、缓存命中、热点与资源瓶颈。

## Q9. 为什么 ES 的更新成本高？更新/删除与 segment/merge 的关系是什么？
题目复述：为什么 ES 的更新成本高？更新/删除与 segment/merge 的关系是什么？

先给结论：Lucene segment 不可变，更新/删除通常是“标记删除旧文档 + 写入新文档”；旧数据要等 merge 才真正回收，所以更新频繁会带来更多 segment 与 merge 压力。

理解路径：不可变 segment 让查询更快、更简单，但写入侧要靠不断生成新 segment，并通过后台 merge 合并段、清理删除标记。频繁更新会扩大写入量与 IO，并影响查询性能（更多段要查、更多 merge 竞争资源）。

关键要点：
- update = delete + add：逻辑更新带来物理写放大。
- delete 的空间回收依赖 merge：短期内磁盘不会立刻下降。
- merge 是重 IO 操作：需要容量与节奏治理，否则会影响在线性能。

常见误答：
- “ES 更新慢是因为分布式复制”：复制是其中一部分，更关键是 segment 不可变与 merge。

## Q10. terms 聚合在什么情况下会很慢/很耗内存？你会怎么做选型或改写？
题目复述：terms 聚合在什么情况下会很慢/很耗内存？你会怎么做选型或改写？

先给结论：当字段基数很高、返回桶很多或字段类型不合适时，terms 聚合会占用大量内存并变慢；可以改字段类型、限制桶数量、改用 composite 或做预聚合。

理解路径：terms 聚合需要在每个分片上统计分桶并汇总，如果分桶数量巨大（例如对用户 id、trace id 做聚合），就会产生大量桶对象与排序开销。协调节点 reduce 汇总也会加重内存压力。

关键要点：
- 字段必须适合聚合：通常用 keyword（或数值/日期），避免对 text 直接聚合。
- 控制输出：限制 size、设置合理的 shard_size，必要时分批聚合。
- 替代方案：composite aggregation 做分页；或离线/流式预聚合。

常见误答：
- “terms 聚合慢就加机器”：没有先判断基数、桶数量与查询形态是否合理。

## Q11. 深分页为什么慢？常见替代方案有哪些（scroll、search_after 等）？
题目复述：深分页为什么慢？常见替代方案有哪些（scroll、search_after 等）？

先给结论：深分页需要跳过大量结果并维持全局排序，开销会随页码线性上升；替代方案包括 search_after、scroll（批量导出）以及业务侧基于游标的分页设计。

理解路径：from/size 的深分页会让分片返回大量候选供协调节点排序与丢弃，导致 CPU/内存/网络浪费。search_after 用“上一页最后一条的 sort 值”作为游标继续查，避免巨量跳过；scroll 更适合离线导出与遍历，不适合实时用户分页。

关键要点：
- 深分页痛点：全局排序 + 大量丢弃 + 分布式 reduce。
- search_after：需要稳定排序字段（通常包含唯一 tie-breaker）。
- scroll：更像快照遍历，用于批处理。

常见误答：
- “深分页慢是因为 ES 没索引”：根因在分布式排序与丢弃成本。

## Q12. 如何做到索引结构升级但不停机（alias + reindex）？切换时如何保证一致性？
题目复述：如何做到索引结构升级但不停机（alias + reindex）？切换时如何保证一致性？

先给结论：用新 index 承载新 mapping，先全量导入再增量追平，最后切 alias；一致性靠“增量同步 + 追平校验 + 原子切换”保障。

理解路径：mapping 变更往往不能原地修改（尤其是类型变化），所以需要创建新 index。alias 提供一个稳定入口，切换 alias 是原子操作，业务无感。为了保证切换后数据不缺不乱，需要在全量导入后持续同步增量变更，并在切换前做追平与对账校验。

关键要点：
- 两套索引并行：index_v1 在线服务，index_v2 构建追平。
- 增量追平：基于 binlog/CDC 或变更日志重放。
- 切换策略：原子更新 alias，必要时双写/短暂冻结窗口。

常见误答：
- “reindex 完成就切”：忽略 reindex 期间的增量变更与追平校验。

## Q13. 写入出现 429（rejected）时，你的工程化处理策略是什么？
题目复述：写入出现 429（rejected）时，你的工程化处理策略是什么？

先给结论：把 429 当成背压信号：降并发/缩小批次、指数退避重试、限流与熔断，必要时扩容或优化索引结构与写入模式。

理解路径：429 通常来自线程池/队列饱和或资源瓶颈（CPU/IO/heap）。硬顶并发只会把集群推向更严重的 GC 与拒绝雪崩。正确做法是在客户端实现背压与重试策略，同时从集群侧找出瓶颈并做容量治理与索引调优。

关键要点：
- 客户端：bulk 批次与并发控制、指数退避、失败重放、幂等写入。
- 集群侧：看线程池拒绝、写入延迟、merge/refresh 压力、GC 与磁盘 IO。
- 长期方案：合理分片、冷热分层、索引模板、写入隔离与扩容。

常见误答：
- “429 就一直重试直到成功”：没有退避与限流，可能造成雪崩。

## Q14. “写入成功但搜不到”的排查路径是什么？你会先看哪些证据？
题目复述：“写入成功但搜不到”的排查路径是什么？你会先看哪些证据？

先给结论：先确认是否写入到了正确索引与正确分片，再确认 refresh 可见性，最后确认查询 DSL 与字段类型/分词是否匹配；证据优先来自写入响应、索引/alias、mapping 与 explain/analyze。

理解路径：这个问题通常不是“ES 丢数据”，而是“可见性与语义不一致”。写入 ack 说明主分片完成，但未必刷新为可搜索；或者业务查错索引/alias、字段类型建错（text/keyword）、查询用错（term/match），都会导致“看起来像丢了”。

关键要点：
- 写入证据：写入响应是否成功、写到了哪个 index、_id 是什么。
- 索引入口：alias 是否指向预期 index；是否存在多版本索引误读。
- 可见性：refresh 周期与是否显式 refresh；是否跨集群复制/同步链路延迟。
- 语义验证：mapping（字段类型）、`_analyze`（分词）、`_explain`（匹配原因）。

常见误答：
- “重启 ES 就好了”：没有按证据链定位根因。

## 参考文献（官网/源码优先）
1. Elasticsearch Reference（版本未锁定，默认 latest）：https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
2. Elasticsearch Java API Client（版本未锁定，默认 latest）：https://www.elastic.co/guide/en/elasticsearch/client/java-api-client/current/index.html
3. Elasticsearch Near real-time search（版本未锁定，默认 latest）：https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html
4. Elasticsearch Shards & replicas（版本未锁定，默认 latest）：https://www.elastic.co/guide/en/elasticsearch/reference/current/scalability.html
5. Apache Lucene（核心原理与倒排索引基础）：https://lucene.apache.org/core/


# 基础与架构

## 1. 什么是 Elasticsearch？它和 MySQL 的定位与适用场景分别是什么？
- **15秒简答**：Elasticsearch 本质是基于 Lucene 的分布式搜索与分析引擎；核心机制是“倒排索引 + 分片并行 + 副本容灾”；特点是检索/聚合强、扩展强，但事务一致性与运维/建模成本更高。
- **3分钟详答**：Elasticsearch（ES）主要解决“海量数据的检索与分析”，底层靠 Lucene 把文本分词成 term，并建立倒排索引，实现从“term → 文档集合”的快速定位，同时支持聚合做统计分析。ES 在架构上把索引拆成多个分片分布在不同节点上，查询时并行执行再汇总。  
  MySQL 更擅长“事务与关系约束”的 OLTP：强一致、事务、关联与约束完善。ES 不提供关系型数据库那样的 ACID 事务语义，更像“搜索索引/分析引擎”。实际落地常见模式是：MySQL 做权威数据源，ES 做搜索与聚合加速层，业务侧接受最终一致并做重试/对账。  
  官方文档关键词：Elasticsearch 概述、Search、Aggregations、Distributed architecture。
- **可能追问（15秒简答）**：
  - Q：ES 能替代 MySQL 吗？A：多数业务不行；ES 强在搜索与分析，事务一致性与关系约束仍要靠 MySQL。
  - Q：为什么 ES 适合全文检索？A：倒排索引让“查词”变成定位 posting list，而不是扫描全量文本。

## 2. Lucene 是什么？倒排索引在 Lucene/ES 里由哪些核心结构组成？
- **15秒简答**：Lucene 本质是单机搜索库；核心机制是“term dictionary + posting list + stored fields/doc values + segment 不可变”；特点是查询快、写入通过新段与合并维持性能。
- **3分钟详答**：Lucene 是 ES 的底座：ES 的每个分片内部本质是一个 Lucene 索引。理解 Lucene 能把很多 ES 行为解释清楚（比如更新删除为什么不是原地改、为什么有段合并）。  
  一个 segment（段）可以理解为“具备完整检索能力的最小单元”，里面典型包含：  
  - Term Dictionary：按字典序存储所有 term（词项），底层会用压缩结构（常见用 FST 思路）来降低内存占用，并支持快速查找/前缀能力。  
  - Posting List：每个 term 对应的文档列表（docId 以及可选的频次、位置等），用于快速求交并做相关性计算。  
  - Stored Fields：用于按 docId 取回原始字段（包含 `_source` 在内的存储），解决“搜到 docId 后怎么返回内容”。  
  - Doc Values：列式存储，用于排序/聚合/脚本取值；避免为了排序先把整篇文档从 stored fields 读出来再提取字段。  
  - Segment 不可变：新写入会生成新 segment；旧段只读，提升并发读写稳定性。segment 过多会带来文件句柄与查询开销，于是后台会做 segment merge，把多个小段合并成大段，同时清理已删除文档。  
  官方文档关键词：Segments、Doc values、Stored fields、Inverted index、FST。
- **可能追问（15秒简答）**：
  - Q：为什么要 doc values？A：排序/聚合按列读取字段更高效，避免每条命中都解压/解析整份文档。
  - Q：为什么段不可变？A：简化并发读、提升缓存命中；更新/删除通过标记删除+新段写入完成。

## 3. Elasticsearch 的核心数据模型概念有哪些？（Index/Document/Mapping/Shard/Replica）
- **15秒简答**：ES 本质是“索引管理文档集合”；核心机制是“mapping 定义字段规则，索引拆成 shard 并复制 replica”；特点是水平扩展与高可用。
- **3分钟详答**：面试里我会抓 5 个概念：  
  1）Index：逻辑数据集合，承载 settings/mappings。  
  2）Document：JSON 最小单元，对应 `_id`。  
  3）Mapping：字段类型与索引规则，决定能否全文检索/聚合/排序。  
  4）Shard：索引的物理拆分单元（每个分片是一个 Lucene 索引）。  
  5）Replica：分片副本，用于容灾与读扩展。  
  类比关系库：Index ≈ 表集合/业务域容器、Document ≈ 行、Mapping ≈ 表结构；但 ES 的查询路径是倒排索引 + 分布式并行。  
  官方文档关键词：Index、Document、Mapping、Shards and replicas。
- **可能追问（15秒简答）**：
  - Q：主分片数能改吗？A：创建后不可直接改；要变更通常用 split/shrink 或新建索引 reindex。
  - Q：副本数能改吗？A：可以动态调整，用于读扩展或导入期临时降副本。

## 4. ES 集群有哪些节点角色？Master（集群管理器）如何选举、如何避免脑裂？
- **15秒简答**：集群本质是“控制面 + 数据面分离”；核心机制是“多数派选举集群管理器、它负责元数据与分片分配”；脑裂本质是网络分区或 Master 假死导致多主/无主，靠多数派、专用 Master、资源隔离和正确版本配置来避免。
- **3分钟详答**：这个问题我会分成节点角色、Master 选举和脑裂三个层次来回答。  
  首先，ES 节点角色不是只有一种。`master-eligible` 节点有资格参与 Master 选举，真正当选以后主要负责集群状态、索引元数据、节点管理和分片分配；`data` 节点负责保存分片，并执行查询、聚合和写入；`ingest` 节点负责写入前的 pipeline 预处理；`coordinating-only` 节点更像请求入口，负责把请求路由到相关分片，再把结果汇总回来。  
  其次，Master 可以理解为 ES 集群的“控制面”。它不是所有数据都存在 Master 上，而是由它维护集群该如何组织。选举时，只有 master-eligible 节点参与投票；当现任 Master 不可达，或者集群启动、节点发现发生变化时，会通过多数派机制选出新的 Master。也就是说，候选节点必须拿到法定票数，通常可以按 `floor(N / 2) + 1` 理解，才能成为新的 Master 并发布 cluster state。  
  最后，脑裂本质上是网络分区后出现多个 Master，或者没有任何一边能正常形成 Master，导致集群状态分叉。它不只可能由网络断开引起，也可能由 Master 负载过高、JVM Full GC、CPU/磁盘压力过大导致心跳和 cluster state 发布超时引起。多数派机制能防脑裂，是因为一个集群里不可能同时存在两个“超过半数”的分区；拿不到多数派的一边即使联系不上原 Master，也不能自己选主。生产上一般会部署 3 个专用 master-eligible 节点，并尽量分散在不同机器或可用区；旧版本要把 `discovery.zen.minimum_master_nodes` 配成过半，7.x+ 则由新的协调模块维护投票配置。  
  延伸理解：[第4题：ES Master 选举与脑裂（理解型记忆版）](./第4题：ES%20Master%20选举与脑裂（理解型记忆版）.md)。  
  官方文档关键词：Node roles、Cluster coordination、Master election。
- **可能追问（15秒简答）**：
  - Q：为什么推荐 3 个专用 Master 节点？A：3 个能容忍 1 个故障并保持 2 票多数派；专用 Master 不被查询/写入拖慢，控制面更稳。
  - Q：脑裂会带来什么后果？A：集群状态分叉，可能出现两个 Master、分片路由混乱、写入异常，严重时造成数据不一致。
  - Q：7.x 以后还要配 `minimum_master_nodes` 吗？A：不用；旧版本需要手动配多数派，7.x+ 由新协调模块自动维护投票配置。
  - Q：除了网络分区，还有什么会诱发脑裂？A：Master 负载过高、Full GC、CPU/磁盘/网络抖动导致心跳超时，都可能让节点误判 Master 不可达。

## 5. 分片与副本在集群里如何分配与再平衡？节点增减/故障会发生什么？
- **15秒简答**：分片/副本本质是“拆分+复制”；核心机制是“主分片写入、副本同步，集群自动分配与再平衡”；特点是容灾与并行强，但分片规划不当会增加管理成本。
- **3分钟详答**：索引创建时确定主分片数，每个主分片可配多个副本。分片会分布到数据节点上，副本会尽量避免与主分片同机。  
  - 新节点加入：触发 rebalancing，把部分分片迁移到新节点以均衡负载。  
  - 节点故障：副本提升为主分片，随后在其他节点补齐新的副本，直到恢复期望副本数。  
  实战里分片规划要兼顾数据量、并发、节点数与扩容路径；分片过多会增加集群状态与合并开销。  
  官方文档关键词：Shard allocation、Rebalancing、Replication。
- **可能追问（15秒简答）**：
  - Q：副本除了容灾还有什么价值？A：读扩展，多个副本可并行处理查询请求。
  - Q：如何控制分片分配到特定节点？A：用 allocation filtering / node attr 做过滤与约束。

## 6. 一次写入和一次搜索在 ES 里大致经历哪些步骤？（coordinating、query phase、fetch phase）
- **15秒简答**：请求流程本质是“路由到分片并行执行再汇总”；核心机制是“写入走主分片复制副本，搜索分 query phase 与 fetch phase”；特点是并行高吞吐，但协调节点会成为汇总热点。
- **3分钟详答**：从你补充的视频笔记可以用一句话串起来：ES 把“Lucene 单机库”做成“多分片 + 多节点 + 可选副本”的分布式系统。  
  写入流程：客户端请求先到 coordinating 节点；它根据路由（常见是 `_id` 哈希）定位目标分片所在的数据节点；请求到达主分片后写入（本质落到 Lucene），写成功后同步到副本分片，副本确认后再向上返回。  
  搜索流程（两阶段）：  
  - query phase：coordinating 把查询分发到相关分片，每个分片在本地多个 segment 上搜索，返回 topN 的 docId 与排序信息。  
  - fetch phase：coordinating 汇总排序后，拿最终命中的 docId 再向分片拉取 stored fields/_source，把完整文档返回给客户端。  
  官方文档关键词：Distributed search、Search phases、Routing、Replication。
- **可能追问（15秒简答）**：
  - Q：协调节点压力大时怎么做？A：引入专用 coordinating 节点，或通过负载均衡分散入口压力。
  - Q：为什么分 query/fetch？A：先用轻量信息（docId+sort）做全局排序，再按需取回文档内容，减少网络与 I/O。

## 7. 什么是 NRT 近实时搜索？refresh / translog / segment / merge 的关系是什么？
- **3分钟详答**：NRT（Near Real-Time）是指一种系统架构使得查询语句能够在接近实时的情况下得到最新的数据结果，是ES的核心特性之一。  
  - 数据写入时先进入内存缓冲并记录 translog。translog是一种事务日志，它记录所有index和delete操作，在发生崩溃时，没有被刷盘的操作可以通过translog进行恢复。
  - refresh 操作会把缓存内容生成新的可搜索 segment 并打开搜索器，默认刷新间隔为 1s，这个间隔时间内的数据写入到ES中，虽然它已经被持久化，但是还不会对外立刻可见。
  - segment 不可变，更新/删除会产生标记删除与新段；后台 merge 会合并小段、清理删除标记，保持文件数与查询性能可控。  
  - flush是指执行lucence的刷盘操作并启动新的translog生成流程
  - merge是指refresh/flush/写入等行为不断产生新 segment，导致 shard 内 segment 状态变化；Lucene 的 IndexWriter 会让 MergePolicy 判断是否超过合并策略的阈值，如果超过，就由 ES merge scheduler 在线程池中后台执行合并。
  官方文档关键词：[Near real-time search](https://www.elastic.co/docs/manage-data/data-store/near-real-time-search)、Refresh、[Translog](https://www.elastic.co/docs/reference/elasticsearch/index-settings/translog)、Segments and [merging](https://www.elastic.co/docs/reference/elasticsearch/index-settings/merge)。
- **可能追问（15秒简答）**：
  - Q：`refresh_interval=-1` 的影响？A：写入吞吐更高，但新数据不可搜索，需手动 refresh 或恢复配置。
  - Q：merge 有什么副作用？A：merge 会吃 CPU/IO，过于频繁会影响写入与查询，需要平衡与监控。

# 建模与索引

## 8. Mapping 是什么？动态映射（dynamic mapping）有什么坑？如何做字段与文档的“收敛”？
- **15秒简答**：Mapping 本质是“字段类型与索引规则的 schema”；核心机制是“类型决定分词、倒排、doc values 行为”；特点是前置设计换来正确性与性能，动态映射省事但易误判与字段爆炸。
- **3分钟详答**：Mapping 决定字段怎么存、怎么查、能不能排序/聚合。dynamic mapping 会在首次写入时推断类型，适合探索期，但生产常见两类坑：  
  - 类型误判：ID 被识别成数值，后续写入带字母就报错；日期格式误判导致范围查询异常。  
  - 字段爆炸：日志/事件类数据字段不受控，mapping 与集群状态膨胀，内存与 GC 压力飙升。  
  收敛思路：核心字段显式定义；对非核心字段用 dynamic templates 归类；尽量扁平化；控制字段数量与字段基数；不需要检索/聚合的字段只存 `_source` 或禁用索引能力。  
  官方文档关键词：Mapping、Dynamic mapping、Dynamic templates、Field limits。
- **可能追问（15秒简答）**：
  - Q：mapping 改错了怎么办？A：多数类型不能原地改，走新建索引 + reindex + 切别名。
  - Q：为什么字段过多会拖垮集群？A：mapping/集群状态变大，内存占用与更新成本上升，查询也更重。

## 9. `text` 和 `keyword` 有什么区别？为什么经常要做 multi-fields？
- **15秒简答**：`text` 是“分词后的全文字段”，`keyword` 是“不分词的精确字段”；核心机制是“text 走 match/相关性，keyword 走 term/聚合排序”；特点是二者互补，multi-fields 兼得。
- **3分钟详答**：`text` 经过 analyzer 生成 tokens 并参与 `_score`；`keyword` 保留原值，适合过滤、聚合、排序、去重。业务里一个字段经常既要搜索又要聚合排序，典型做法是 multi-fields：如 `title` 用 `text`，再提供 `title.keyword` 用 `keyword`。  
  官方文档关键词：Text、Keyword、Multi-fields、Doc values。
- **可能追问（15秒简答）**：
  - Q：为什么 `text` 不适合聚合？A：`text` 是 token 流，聚合需要稳定原值，通常用 `.keyword`。
  - Q：term 查询适合查什么字段？A：适合 `keyword/numeric/date` 这类精确值字段。

## 10. Analyzer 分析器由哪些部分组成？Token Filter 常见用法有哪些？
- **15秒简答**：分析器本质是“文本→token 流”；核心机制是“字符过滤→分词器→token filter”；特点是决定召回与精度，上线后改动成本高。
- **3分钟详答**：全文检索的效果基本由 analysis 决定：  
  - Character Filter：处理原始文本（去 HTML、字符映射）。  
  - Tokenizer：切分成 token（standard/whitespace/ngram 等）。  
  - Token Filter：规范化/增强 token（lowercase、stop、stemmer、synonym、edge_ngram）。  
  最常见的坑是“索引时和查询时分析链路不一致”，导致用户搜不到或召回异常；另一个坑是同义词带来召回提升但可能引入噪声，需要灰度与可回滚。  
  官方文档关键词：Analysis、Analyzer、Tokenizer、Token filters、Synonym token filter。
- **可能追问（15秒简答）**：
  - Q：同义词放索引时还是查询时？A：索引时扩展会增大索引；查询时更灵活但查询成本更高。
  - Q：Edge ngram 常用在哪？A：前缀匹配/搜索建议/自动补全类需求。

## 11. `object` 和 `nested` 有什么区别？什么时候必须用 nested？
- **15秒简答**：`object` 本质是“扁平化多值字段”，`nested` 是“子对象独立成隐藏文档”；核心机制是 nested query 保证同一子对象内字段关联；特点是正确但更耗资源。
- **3分钟详答**：对象数组如果用 `object`，ES 会把字段扁平化，多值字段之间可能被错误组合，出现假阳性匹配。`nested` 会把每个子对象独立索引，查询时指定 `path`，保证条件落在同一个子对象上。  
  官方文档关键词：Nested field type、Nested query。
- **可能追问（15秒简答）**：
  - Q：nested 的代价？A：写入会生成更多内部文档，索引体积与查询成本上升。
  - Q：如果只需要存不需要关联查询呢？A：用扁平字段或 object 更省。

## 12. 索引怎么做“版本迁移/滚动更新”？别名（alias）和索引模板（template）怎么配合？
- **15秒简答**：迁移本质是“新建索引替换旧索引”；核心机制是“模板定义结构、别名解耦读写、reindex/双写迁移后切别名”；特点是无感切换、可回滚。
- **3分钟详答**：ES 的 mapping 很多改动无法原地完成，所以线上常用“新建索引 + 切别名”：  
  1）用 index template 固化 settings/mappings。  
  2）写入走写别名、查询走读别名。  
  3）创建 v2 索引后，用 reindex 或双写迁移数据。  
  4）验证后切别名到 v2，保留 v1 以便回滚，最后清理。  
  日志类还会配合 rollover：按时间/大小滚动创建新索引，避免单索引过大。  
  官方文档关键词：Index templates、Aliases、Rollover、Reindex API。
- **可能追问（15秒简答）**：
  - Q：为什么建议读写别名分离？A：迁移过程读写可能指向不同版本，分离能降低风险与回滚成本。
  - Q：什么时候用 reindex？A：字段类型修正、分析器变更、需要重构数据集时。

# 查询与相关性

## 13. Query DSL 怎么分层理解？`term` vs `match`、`query` vs `filter` 的区别是什么？
- **15秒简答**：DSL 本质是“可组合的查询树”；核心机制是“全文查询分词并算分，filter 不算分且易缓存”；特点是 filter 快且稳，match 更智能但更贵。
- **3分钟详答**：我会按两层理解：  
  - term-level：`term/terms/range/exists`，适合 `keyword/numeric/date`，不分词，用于精确过滤与聚合前置筛选。  
  - full-text：`match/multi_match/match_phrase`，会走 analyzer 并计算 `_score`。  
  `bool.filter` 不参与评分、适合放业务硬条件（tenant_id、状态、时间范围）；`bool.must` 放关键词检索，形成“先过滤后相关性排序”的稳定结构。  
  官方文档关键词：Query DSL、Term-level queries、Full-text queries、Bool query。
- **可能追问（15秒简答）**：
  - Q：为什么 filter 更快？A：不算 `_score`，结果可复用缓存，执行路径更短。
  - Q：match_phrase 的用途？A：短语匹配，要求词序与距离更严格，可用 slop 放宽。

## 14. 相关性评分（_score）是什么？BM25 为什么是默认？如何做“业务排序”？
- **15秒简答**：_score 本质是“匹配度”；核心机制是“BM25 用 TF 饱和 + IDF + 长度归一化”算分；特点是排序更稳，业务可用 boost/function_score 混入热度/时效等信号。
- **3分钟详答**：全文检索不仅要召回还要排序。ES 默认 BM25：它避免 TF 无上限带来的异常加分，并做文档长度归一化，让排序更符合直觉。影响 _score 的因素包括词频、逆文档频率、字段长度、字段权重、查询结构。  
  真正上线的搜索，通常要把业务信号混进去：例如“标题权重高于正文”“新内容更靠前”“点击/销量影响排序”，常见做法是字段 boost、function_score、或者只用相关性召回再按业务字段 sort。排查排序问题时用 explain 看贡献项。  
  官方文档关键词：Relevance、Similarity、BM25、Explain API、Function score query。
- **可能追问（15秒简答）**：
  - Q：什么时候不要用 _score？A：纯过滤列表或报表，直接 filter + sort 更可控、更稳定。
  - Q：boost 应该怎么用？A：对更重要字段提高权重，避免全靠正文匹配影响排序质量。

## 15. 聚合（Aggregations）与排序为什么依赖 doc values？去重（distinct）怎么做更稳？
- **15秒简答**：聚合/排序本质是“按列读取字段做统计与排序”；核心机制是“doc values 提供列式访问，terms 桶多会爆，composite 可分页”；特点是分析强但易吃内存，要控桶数与基数。
- **3分钟详答**：聚合用于统计分析（分组、直方图、topN 等），排序用于按字段取序。二者高效依赖 doc values：把某字段集中列存，避免每条命中都解压 stored fields 再抽字段。  
  性能要点：先 filter 缩小数据集；terms 聚合面对高基数可能产生大量桶并触发内存/桶数限制；需要全量遍历唯一值时用 composite 聚合分页（after_key）。  
  如果你看到 fielddata 占用飙升，多半是对不合适字段（常见 text）做了排序/聚合，应该改用 `.keyword` 或调整 mapping。  
  官方文档关键词：Aggregations、Doc values、Terms aggregation、Composite aggregation、Fielddata。
- **可能追问（15秒简答）**：
  - Q：terms vs composite 怎么选？A：terms 适合 topN；composite 适合全量遍历/分页取桶。
  - Q：为什么 text 排序/聚合容易爆内存？A：会触发 fielddata 把值加载到堆里，正确做法是用 keyword/doc values。

## 16. 深分页为什么慢？Scroll 和 Search After 怎么选？
- **15秒简答**：深分页本质是“要跳过大量排序结果”；核心机制是“from/size 越深每个分片要返回更多候选并在协调节点丢弃”；特点是高内存高延迟，解决用 scroll 或 search_after。
- **3分钟详答**：`from + size` 深分页会让每个分片计算并返回大量候选，协调节点全局排序后再丢弃前面的，浪费巨大。  
  - Scroll：适合全量遍历/导出这类“静态数据扫描”，维护游标上下文。  
  - Search After：适合在线翻页，依赖稳定且唯一的排序键（常用 时间 + `_id`），用上一页最后一条的 sort 值继续查。  
  官方文档关键词：Pagination、Scroll API、Search after。[深分页](https://www.mianshiya.com/bank/1805423815382736897/question/1827341037887049729#heading-0)
- **可能追问（15秒简答）**：
  - Q：search_after 为什么要唯一排序？A：避免排序相同导致重复/漏数据，通常加 `_id` 做 tie-breaker。
  - Q：scroll 适合用户分页吗？A：一般不适合；更适合离线扫描任务。

# 性能与运维

## 17. 如何优化索引（写入）阶段的性能？Bulk 导入时怎么做？
- **15秒简答**：写入优化本质是“减少请求与刷新写放大”；核心机制是“Bulk + 调大/关闭 refresh + 临时 replicas=0”；特点是吞吐提升明显，但会降低近实时与短期容灾。
- **3分钟详答**：写入瓶颈常见来自：请求次数、refresh 频率、复制写入、磁盘与 merge 压力。常用组合：  
  - Bulk API 合并请求，批次大小按 MB/条数压测。  
  - 导入期调大 `refresh_interval` 或设 `-1`，减少频繁生成新段。  
  - 导入期临时设 `number_of_replicas: 0`，导完再恢复。  
  - 分片规划与硬件（SSD/内存）要匹配吞吐。  
  官方文档关键词：Bulk API、Refresh、Tuning for indexing speed。
- **可能追问（15秒简答）**：
  - Q：副本=0 的风险？A：导入期间节点挂了恢复能力差，通常要保证源数据可回放并做对账。
  - Q：为什么 refresh 影响写入？A：refresh 会生成可搜索段并打开搜索器，频繁执行会抢 CPU/IO。

## 18. ES 的更新/删除为什么“看起来慢”？大规模删除与 ILM 怎么做更稳？
- **15秒简答**：更新/删除本质是“标记删除 + 写新段”；核心机制是“段不可变，靠 merge 回收空间；时间序列优先按索引删除并用 ILM 自动化”；特点是写放大存在，索引级策略最稳。
- **3分钟详答**：更新不是原地改：旧文档先被标记删除，再写入新版本；删除同样是标记删除，真正清理由 segment merge 完成。大量逐条 delete 会产生大量 deleted docs 与合并压力，甚至影响查询。  
  更稳的思路是把“删除”上升到索引粒度：日志按天/周建索引，过期直接删 index；或者用 ILM 管理生命周期（hot→warm/cold→delete），到期自动 delete phase。  
  对业务数据的大范围清理，常用“新索引 reindex 只保留需要的数据→切别名→删旧索引”，比 delete-by-query 更可控。  
  官方文档关键词：Delete and update、Segments and merging、ILM、Rollover。
- **可能追问（15秒简答）**：
  - Q：deleted docs 多了会怎样？A：磁盘与查询成本上升，直到 merge 才会回收空间并改善性能。
  - Q：ILM 的核心价值？A：把索引滚动、迁移、清理自动化，降低运维成本并控制存储成本。

## 19. JVM/GC 与 Fielddata 为什么经常是 ES 性能瓶颈？怎么调优？
- **15秒简答**：瓶颈本质是“堆内存抖动 + 不当字段加载”；核心机制是“heap 合理配置并监控 GC，排序/聚合尽量走 doc values 避免 fielddata”；特点是稳定性取决于内存治理与写法约束。
- **3分钟详答**：ES 是 Java 服务，延迟抖动常来自 GC：堆过大或对象压力大，会出现明显停顿。经验上 heap 不建议超过 31GB，并预留 OS page cache 给 Lucene 段文件读写。  
  Fielddata 是另一个热点：对 text 排序/聚合/脚本可能触发 fielddata 把值加载进堆，导致内存飙升与 GC 风暴。正确做法是：对需要排序/聚合的字段用 `keyword/numeric/date`，走 doc values；对 text 走 `.keyword` 子字段。  
  调优顺序通常是：先看指标（heap、GC、fielddata、查询延迟、merge 压力），再从 mapping/查询写法/缓存/堆配置逐步收敛。  
  官方文档关键词：JVM settings、Heap sizing、Fielddata、Doc values。
- **可能追问（15秒简答）**：
  - Q：为什么 heap 不能无限大？A：堆大 GC 停顿更长且会挤压 page cache，反而拖慢 Lucene I/O。
  - Q：怎么快速判断是不是 fielddata 问题？A：看 fielddata 内存占用与请求类型，若对 text 做排序/聚合基本就是它。

# 生命周期与集成

## 20. 如何让 MySQL 和 Elasticsearch 的数据保持同步？如何处理一致性、重试与对账？
- **15秒简答**：同步本质是“把 ES 当搜索索引副本”；核心机制是“CDC/消息队列增量同步 + 幂等写入 + 可重放”；特点是最终一致，关键在顺序、重试与对账。
- **3分钟详答**：MySQL 是权威数据源，ES 是检索视图。常见同步方案按实时性递增：  
  - 定时同步：简单但延迟大。  
  - MQ 驱动：业务写 MySQL 后发事件到 MQ，消费者写 ES；需要处理重试与去重。  
  - CDC：监听 binlog（如 Debezium→Kafka），捕获增删改事件再写 ES，更贴近真实变更链路。  
  稳定性关键点：用业务主键作为 ES `_id` 保证幂等；同一主键事件按 key 分区保证顺序；失败有重试与死信；定期对账（数量、抽样比对）发现偏差后补偿。  
  官方文档关键词：Index API、Bulk API、Optimistic concurrency control（与外部一致性策略配合）。
- **可能追问（15秒简答）**：
  - Q：为什么用业务 id 做 ES `_id`？A：幂等，重复消费只会覆盖同一文档，避免重复数据。
  - Q：删除怎么同步？A：CDC/MQ 里发送 delete 事件，ES 执行 delete；时间序列更推荐按索引生命周期删除。

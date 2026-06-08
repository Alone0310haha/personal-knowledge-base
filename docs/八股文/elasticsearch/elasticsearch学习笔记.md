### class 1
es是用json文档的形式存储数据。
它提供索引index和数据流的方式来组织数据。

索引的组成部分有：
文档(Documents)：存储你数据的json对象，包括系统管理的元字段。

常见的元数据字段可以按用途分成几类（[所有元数据字段](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/document-metadata-fields)）：

1) 定位与归属

- `_index`  ：文档所属的 索引名 （字符串）。当一次查询跨多个索引/别名时，用它标识“这条命中来自哪个索引”。
- `_id` ：文档的 唯一标识符 （字符串）。用于 GET /{index}/_doc/{id} 、更新、删除、upsert 等。若写入时不指定，ES 会自动生成一个 id。
- `_routing` ：路由值（可选）。决定文档落到哪个分片；不设置时默认用 _id 做 routing。


2) 文档内容承载

- `_source` ：你写入的原始 JSON 文档内容（可选择关闭/过滤，但通常建议保留，便于回查与更新脚本）。

相当于_source就是实际的数据，换成关系型数据库，可以理解为_id是主键，_source是数据行内容，其他为隐藏列。

3) 乐观并发控制与写入顺序（ES 6/7+ 常用）

- `_seq_no`  ：序列号，标识该文档变更的先后顺序。
- `_primary_term` ：主分片任期号，主分片切换后会递增；配合 _seq_no 做并发控制（ if_seq_no / if_primary_term ）。
- `_version`  ：版本号（仍存在，但新体系更推荐用 _seq_no + _primary_term ）。


4) 查询相关（非持久业务字段）

- `_score`  ：相关性评分，只在搜索响应里出现（不“存业务内容”，也不是你写入的字段）。


5) 其他管理/辅助（是否出现取决于映射与场景）

- `_ignored`  ：记录哪些字段因为 ignore_malformed / ignore_above 等原因被忽略（出现在命中里）。
- `_field_names`  ：用于 exists 查询的内部辅助（新版本实现细节可能变化）。
- `_type`  ：旧版类型字段，ES 7 起基本废弃（单类型），ES 8 已移除相关概念（兼容层除外）。
- `_doc_count`  ：常见于一些聚合/特殊映射场景的内部计数用途（不是普通文档都会关心的业务字段）。

乐观并发控制在此处不展开，后续需要回来看。

Mappings（映射）：定义字段数据类型，并控制数据的索引和查询方式。
[es中所有数据类型](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/field-data-types)



Settings（设置）：索引级别的配置，如分片数量，副本数量和刷新间隔，控制存储和性能行为。

近实时搜索：
es并不是写入立刻可搜，而是写入后很快可搜，中间有时间差，所以es叫近实时搜索。

基本流程：
把 ES 想成一个“写入很快、但查询要等它把新内容整理好”的系统。底层 Lucene（ES 的搜索引擎库）工作方式大概是：
- 你 index / update 文档时，数据先进入内存里的缓冲区，此时还不一定能被搜索到 。 
- 到点后做一次 refresh：把内存里的积累的变更整理成一个新段（segment）并打开给查询用。该新segment会写进文件系统缓存，这个时候就能被查询看见了。
- 生成的 segment 之后会被操作系统慢慢刷到磁盘。

触发一次refresh的时机由 `index.refresh_interval` 配置，默认为1s，默认只对“最近 30 秒内被搜索过”的索引才按 1 秒 refresh。
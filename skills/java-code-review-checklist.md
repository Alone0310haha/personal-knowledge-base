# 通用 Java 代码评审 Checklist（Atomic / SOP）

目标：将代码评审标准化为可执行的检查动作，侧重语法正确性、资源泄露、代码可读性。

冲突仲裁：
- 排版/格式：以 Google Java Style Guide 为准。
- 业务安全与并发：以阿里巴巴 Java 开发规范与并发 checklist 的约束为准。

使用方式（SOP）：
1. 按分类逐条执行检查动作；每条必须能在代码中定位到“是/否”结论。
2. 发现问题时，记录：影响面（正确性/资源/可读性）、触发条件、最小修复方案。
3. 对并发与资源泄露问题，优先给出可复现的竞争/泄露路径（线程/锁/资源生命周期）。

---

## 命名规约

- [ ] 检查包名是否全小写，并且点分隔的每一段是单个有语义的英文单词（避免复合词/缩写堆叠导致歧义）。
- [ ] 检查类/接口/枚举/注解名是否为 UpperCamelCase，且缩写按“词”处理（例如 XmlHttpRequest 风格，而非全大写缩写连写）。
- [ ] 检查方法/参数/成员变量/局部变量是否为 lowerCamelCase，且名称表达业务语义而非类型或实现细节。
- [ ] 检查常量（static final）是否为 UPPER_SNAKE_CASE，且命名能表达完整约束语义（例如包含单位/范围/含义）。
- [ ] 检查 boolean 属性名是否避免以 `is` 作为字段前缀（避免框架序列化/反射属性解析偏差），并核对 getter 命名与字段语义一致。
- [ ] 检查数组声明是否使用 `Type[] name` 形式，而不是 `Type name[]`。
- [ ] 检查子类字段名是否与父类字段重名；检查同一方法不同代码块中的局部变量是否重名（降低可读性并易引入误用）。

---

## 代码格式

- [ ] 检查每个 `.java` 文件是否仅包含一个顶级类，并且文件名与顶级类名一致。
- [ ] 检查源文件结构是否按顺序组织：license（如有）→ package → imports → class；各 section 之间是否恰好一个空行。
- [ ] 检查缩进是否为 2 空格，且不存在 Tab 缩进字符。
- [ ] 检查是否遵守列宽限制 100（不含 package/import 行），并使用合理换行而非水平滚动代码。
- [ ] 检查 import 是否无通配符（包括 static wildcard），并按规范分组排序（static 一组 + 空行 + non-static 一组；组内按字典序）。
- [ ] 检查大括号是否使用 K&R 风格，且所有控制语句（if/for/while/do）即使单行也使用大括号（避免悬挂 else/维护风险）。
- [ ] 检查每条语句是否独占一行（避免在同一行串联多条语句）。
- [ ] 检查 `switch` 是否显式处理 default；如存在 fall-through，是否以清晰方式标识并确保不会误触发（避免隐藏控制流）。

---

## OOP 规约

- [ ] 检查类/方法/字段可见性是否最小化（能用 private 就不用 package-private/protected/public）。
- [ ] 检查覆盖父类/接口方法时是否使用 `@Override`（包含实现接口方法），避免签名不一致导致“未覆盖但编译通过”的错误。
- [ ] 检查 `equals`/`hashCode` 是否同时实现且语义一致，并且用于集合键（Map key/Set 元素）时其参与字段是否不可变或生命周期内不变。
- [ ] 检查可变内部状态（数组/集合/Date 等）是否被直接暴露：构造入参是否需要防御性拷贝；getter 是否返回防御性拷贝或不可变视图。
- [ ] 检查是否在构造器/工厂方法中完成参数校验（null、范围、空集合、非法状态组合），避免对象处于“部分初始化但可用”的状态。
- [ ] 检查资源型对象（连接/流/线程池/锁等）是否在类边界内有明确生命周期（创建/使用/关闭），并且关闭动作可达（包含异常路径）。
- [ ] 检查是否在 `equals`/`hashCode`/`toString` 中执行可能阻塞或有副作用的操作（I/O、锁、远程调用），避免隐式调用造成性能与死锁风险。

---

## 集合处理

- [ ] 检查遍历 `List/Set/Map` 时是否在循环体内直接 `remove/add` 修改同一集合；如需要删除元素，是否使用 `Iterator.remove()` 或使用收集后再批量变更。
- [ ] 检查是否在 `Map.get()/containsKey()` 之后再 `put/remove` 形成“check-then-act”逻辑；如需要原子更新，是否使用 `compute/merge/putIfAbsent/replace` 等原子方法。
- [ ] 检查对可能为 `null` 的集合返回值是否统一为 `Collections.emptyList()/emptySet()/emptyMap()`（避免调用方 NPE 与额外判空分支）。
- [ ] 检查是否把内部可变集合直接作为返回值或暴露为 public 字段；如必须暴露，是否提供不可变视图或快照副本。
- [ ] 检查对 `subList()` 的使用是否明确其视图语义（对原列表结构修改会影响视图并可能触发异常），必要时是否复制为新集合。
- [ ] 检查集合容量是否在热点路径做了合理预估（例如 `new HashMap<>(expectedSize)`），以减少扩容与 rehash 造成的性能抖动。

---

## 并发编程

- [ ] 检查有并发访问可能的类（例如 Controller/Servlet/Filter/Handler/单例服务）是否明确声明线程安全属性（immutable/thread-safe/not thread-safe）或在类级别说明线程模型与共享状态的保护策略。
- [ ] 检查是否使用 `synchronized` + `wait/notify` 进行条件等待；如存在，是否可替换为 `java.util.concurrent` 工具（`Lock/Condition`、`CountDownLatch`、`Semaphore`、`BlockingQueue`、`CompletableFuture` 等）。
- [ ] 检查临界区是否包含外部/可重入/可覆写的调用（回调、接口方法、I/O、远程调用）；如包含，是否能移出锁外以降低死锁与长时间占锁风险。
- [ ] 检查是否存在嵌套锁/嵌套临界区；如存在，是否定义并遵循固定的加锁顺序，避免循环等待死锁。
- [ ] 检查锁获取与释放是否使用 `try/finally` 保证释放；检查 `lock()` 后到 `try` 之间是否存在可抛异常语句（避免漏释放）。
- [ ] 检查是否在锁内执行大块计算/长循环；如存在，是否能缩小临界区或拆分为“复制快照→锁外计算→锁内提交”。
- [ ] 检查每一个 `volatile` 字段是否满足可见性/有序性需求且用法正确（例如不依赖复合操作原子性）；如需要复合原子性，是否改用锁或原子类。
- [ ] 检查共享可变字段是否有明确的保护规则：要么为 `final`（不可变），要么为 `volatile`，要么被同一把锁保护（并保持一致性），避免“部分同步”的良性竞态假设不清晰。
- [ ] 检查被锁保护的字段/集合是否在代码中明确标注保护关系（例如 `@GuardedBy("lock")` 或等价的类/字段说明），避免后续维护者误用导致竞态。
- [ ] 检查 `ThreadLocal` 是否为 static 且生命周期可控；如使用线程池，是否在任务结束时 `remove()` 以避免线程复用导致的内存泄露/脏数据。
- [ ] 检查是否用 `Thread.sleep()` 作为线程协作/等待手段；如存在，是否改为显式同步原语（`await`/`join`/`Future.get`/条件队列）。
- [ ] 检查线程池是否避免无界队列/无界线程创建（例如默认 `newCachedThreadPool`/无界 `LinkedBlockingQueue`）造成 OOM；检查拒绝策略、队列容量、线程命名是否明确。
- [ ] 检查线程池/调度器是否在生命周期结束时显式 `shutdown()`/`shutdownNow()` 并处理等待终止（避免资源泄露与进程无法退出）。
- [ ] 检查对并发容器的方法使用与变量声明类型一致：若调用 `ConcurrentHashMap` 的特定行为（如 `compute`/`forEach`/`mappingCount`），变量是否声明为 `ConcurrentHashMap`（而非 `Map`/`ConcurrentMap`）以避免 API 受限或误用。
- [ ] 检查对 `Collections.synchronized*` 容器的复合操作（迭代、stream、`toArray`、`containsAll/addAll/removeAll/putAll`）是否在同一把锁上同步整个操作过程。
- [ ] 检查日期格式化/解析是否使用线程安全方式（避免共享 `DateFormat` 实例在并发下使用）；如必须共享，是否加锁或改用线程安全替代方案。

---

## 异常日志

- [ ] 检查是否使用 try-with-resources 关闭 `InputStream/OutputStream/Reader/Writer/Socket/Channel` 等可关闭资源；禁止依赖 `finalize` 或“由 GC 回收时关闭”的假设。
- [ ] 检查 `finally` 中是否可能抛出异常覆盖原始异常；如需要处理关闭异常，是否保留原始异常为主因并附加 suppressed/cause 信息。
- [ ] 检查是否捕获过宽异常（`Exception`/`Throwable`）并吞掉或仅打印信息；如捕获，是否：记录上下文 + 记录异常栈 + 做出明确的恢复/降级/重试/终止决策。
- [ ] 检查异常转换/包装是否保留 cause（`new XxxException(msg, cause)`），避免丢失堆栈导致排障困难。
- [ ] 检查日志是否使用参数化写法（占位符）而不是字符串拼接（避免无谓开销与可读性下降）。
- [ ] 检查错误日志是否包含足够的定位上下文（关键业务标识/输入摘要/状态码/资源标识），并避免记录敏感信息（密码、token、密钥、身份证号、银行卡号等）。
- [ ] 检查日志级别是否匹配影响程度：可预期的业务分支不使用 error；真正的不可恢复错误才使用 error，并确保告警可控（避免日志风暴）。
- [ ] 检查循环/高频路径中的日志是否受限（采样/限频/聚合），避免 I/O 放大与性能退化。

---

## References（维护入口）

- Google Java Style Guide: https://google.github.io/styleguide/javaguide.html
- 阿里巴巴 Java 开发规范（手册整理版）: https://www.cnblogs.com/blknemo/p/13686416.html
- Java Concurrency Code Review Checklist: https://github.com/code-review-checklists/java-concurrency/blob/master/README.md

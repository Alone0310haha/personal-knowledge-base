# Java 并发陷阱：static Cache + 实例锁 与“伪双重检查”

基于截图中的 `UserService` 代码，对其并发与设计问题做一次结构化复盘，并补充“为什么是不同的锁读写同一个静态 Cache”以及“它和双重检查（DCL）的区别”。

## 代码意图（推测）

- 通过 `CACHE` 对 `userId` 去重：同一个 `userId` 只处理一次
- 用 `total` 统计新增次数
- 异步执行 `save(userId)`

## 核心问题清单

### 1）`static CACHE` + `synchronized(this)`：锁对象选错，保护不了共享数据

- `CACHE` 是 `static`：属于类级别，全 JVM 内所有 `UserService` 实例共享同一份 `CACHE`
- `synchronized(this)` 锁的是实例对象：每个 `new UserService()` 都有自己不同的 `this`，锁也不同

结论：只要存在多个 `UserService` 实例，就会出现“不同的锁同时读写同一个静态 `CACHE`”，导致并发访问未被互斥保护。

最直观的并发场景：

```java
UserService s1 = new UserService();
UserService s2 = new UserService(); // 不同实例 -> 不同 this 锁

new Thread(() -> s1.addUser("u1")).start();
new Thread(() -> s2.addUser("u1")).start();
```

- 线程 A 进入 `synchronized(s1)`，线程 B 进入 `synchronized(s2)`
- 两把锁互不影响，A/B 可以同时进入同步块
- 但它们操作的是同一个 `UserService.CACHE`（静态共享）

### 2）`HashMap` 并发不安全：`get/put` 并发可能导致数据错乱

`HashMap` 不是线程安全容器，并发 `get/put` 存在竞态与结构不一致风险。即使你在某些路径上加了锁，只要还有并发路径绕开同一把锁，风险仍然存在。

### 3）“双重检查”形式存在，但语义并不成立

代码结构类似：

- 锁外 `if (CACHE.get(userId) != null) return;`
- 锁内再次 `if (CACHE.get(userId) == null) { put; total++; }`

问题在于：

- 锁不是“全局同一把”（见问题 1）
- 容器不是并发容器，锁外读本身就可能与并发写冲突

### 4）`total` 可见性与一致性问题

- `total++` 在 `synchronized(this)` 内
- 但 `getTotal()` 读取没有同步/volatile/原子类保障

结果：其他线程可能读到旧值（可见性问题）。

此外语义也不一致：

- `CACHE` 是静态全局一份
- `total` 是实例字段（每个实例一份）

这会让“全局去重”与“实例计数”混在一起，容易产生业务含义上的错误。

### 5）每次调用都 `new Thread().start()`：线程爆炸与资源浪费

高并发下频繁创建线程会造成：

- 线程创建/切换成本巨大
- 线程数失控，导致吞吐骤降、内存压力增大甚至 OOM

应使用线程池（`ExecutorService`）或框架提供的异步执行能力。

### 6）异常被吞掉、且未处理线程中断

`catch (Exception e) {}` 会掩盖问题，排障困难；`sleep` 被中断应捕获 `InterruptedException` 并恢复中断标记（`Thread.currentThread().interrupt()`）。

## 为什么说“不同的锁同时读写同一个静态 Cache”

一句话：**static 数据是“类共享”，`this` 锁是“实例私有”。**

- 同一个类可以创建多个实例，每个实例的 `this` 不同
- 但 `static CACHE` 对所有实例只有一份

因此多实例下必然出现：不同实例锁并发保护同一份共享数据，无法互斥。

## 它和双重检查（DCL）的区别

### DCL 的关键点（不是“检查两次”这么简单）

经典 DCL（以单例为例）依赖三个核心条件：

1. 第一次检查走无锁快路径（性能）
2. 第二次检查发生在**同一把全局锁**保护下（正确性）
3. 被发布的共享引用通常需要 `volatile`（可见性 + 禁止指令重排）

```java
class Foo {
  private static volatile Foo instance;
  static Foo getInstance() {
    if (instance == null) {
      synchronized (Foo.class) {
        if (instance == null) {
          instance = new Foo();
        }
      }
    }
    return instance;
  }
}
```

### 截图代码为什么是“伪双重检查”

- 虽然也“检查两次”，但锁是 `this`，并非全局同一把锁
- 目标对象是 `HashMap` 的复合操作（check-then-act），不是 DCL 常见的“发布单个引用”
- 缺少并发容器/volatile 等内存语义保障，锁外读也可能与并发写冲突

结论：形式像 DCL，但不满足 DCL 的前提条件，无法保证线程安全。

## 推荐改法（去重 + 计数 + 异步）

更直接、更可靠的方式：用 `ConcurrentHashMap` 的原子操作完成“去重插入”，并用原子类计数；异步用线程池执行。

```java
private static final java.util.concurrent.ConcurrentMap<String, Boolean> CACHE =
    new java.util.concurrent.ConcurrentHashMap<>();

private final java.util.concurrent.atomic.AtomicInteger total =
    new java.util.concurrent.atomic.AtomicInteger();

private final java.util.concurrent.Executor executor =
    java.util.concurrent.Executors.newFixedThreadPool(8);

public void addUser(String userId) {
  if (CACHE.putIfAbsent(userId, Boolean.TRUE) != null) {
    return;
  }
  total.incrementAndGet();
  executor.execute(() -> save(userId));
}

public int getTotal() {
  return total.get();
}
```

补充说明：

- 如果你要“全局去重 + 全局计数”：`CACHE` 和 `total` 都应是全局语义（例如都用 `static` + 原子/并发容器）
- 如果你要“实例隔离”：就不要 `static CACHE`，并保证实例内读写遵循同一把锁或使用并发容器


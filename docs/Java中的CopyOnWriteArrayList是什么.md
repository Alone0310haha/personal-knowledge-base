[Java中的CopyOnWriteArrayList是什么](https://www.mianshiya.com/bank/1788408712975282177/question/1780933294838280194#heading-0)
`CopyOnWriteArrayList` 是 `java.util.concurrent` 包下的一个线程安全的List实现，核心机制是写时复制；每次写操作都会复制一份新数组，在新数组上修改，然后替换旧数组。
写操作开销大，因为要复制整个数组；但读操作完全无锁，直接读数组就行。所以它适合读多写少的场景，比如配置列表，监听器列表这种写入频率低但读取频繁的数据。
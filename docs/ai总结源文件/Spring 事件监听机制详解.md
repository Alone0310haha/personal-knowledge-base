---
tags:
  - Spring
  - 事件驱动
  - 观察者模式
  - 源码分析
---

# Spring 事件监听机制详解

## 一、快速了解：Spring 事件机制是什么

Spring 事件监听机制是基于**观察者模式**实现的组件间解耦方案。核心思想：

> 某个组件完成一件事后，"喊一嗓子"，其他关心的组件听到后自动执行自己的逻辑。

### 1.1 核心角色

| 角色 | 对应类/接口 | 职责 |
|------|-------------|------|
| **事件** | `ApplicationEvent` | 描述发生了什么事（携带事件数据） |
| **事件发布者** | `ApplicationEventPublisher` | 负责"喊一嗓子"（发布事件） |
| **事件监听器** | `ApplicationListener<E>` | 负责"听"，听到后执行逻辑 |
| **事件广播器** | `ApplicationEventMulticaster` | 负责把事件广播给所有监听器 |

### 1.2 使用示例

```java
// 1. 定义事件（继承 ApplicationEvent）
public class UserRegisteredEvent extends ApplicationEvent {
    private final String userId;
    private final String email;
    
    public UserRegisteredEvent(Object source, String userId, String email) {
        super(source);
        this.userId = userId;
        this.email = email;
    }
    
    public String getUserId() { return userId; }
    public String getEmail() { return email; }
}

// 2. 发布事件（注入 ApplicationEventPublisher）
@Service
public class UserService {
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public void register(String userId, String email) {
        // ... 用户注册逻辑
        eventPublisher.publishEvent(new UserRegisteredEvent(this, userId, email));
    }
}

// 3. 监听事件（实现 ApplicationListener 或用 @EventListener）
@Component
public class EmailNotificationListener {
    
    // 方式一：实现接口
    // @Override
    // public void onApplicationEvent(UserRegisteredEvent event) {
    //     sendEmail(event.getEmail(), "欢迎注册！");
    // }
    
    // 方式二：注解方式（推荐）
    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        sendEmail(event.getEmail(), "欢迎注册！");
    }
    
    private void sendEmail(String to, String msg) {
        // 发送邮件逻辑
    }
}
```

### 1.3 典型应用场景

- **用户注册后**：发送欢迎邮件、送优惠券、初始化用户数据
- **订单支付成功后**：扣减库存、通知商家、更新积分
- **配置刷新后**：通知各模块重新加载配置
- **审计日志**：记录关键操作（不侵入主业务流程）

---

## 二、底层原理：从源码角度看 Spring 事件机制

### 2.1 核心类图关系

```
ApplicationEvent (事件基类)
    ↑
    |— UserRegisteredEvent (自定义事件)

ApplicationListener<E> (监听器接口)
    ↑
    |— EmailNotificationListener (自定义监听器)

ApplicationEventPublisher (发布接口)
    ↑
    |— ApplicationContext (实现)

ApplicationEventMulticaster (广播器接口)
    ↑
    |— SimpleApplicationEventMulticaster (默认实现)
```

### 2.2 初始化过程：容器启动时发生了什么

Spring 容器启动时，在 `AbstractApplicationContext.refresh()` 方法中完成事件机制的初始化：

```java
// AbstractApplicationContext.refresh() 中的关键步骤

// 第 1 步：初始化事件广播器
protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        // 如果用户自定义了广播器，用用户的
        this.applicationEventMulticaster = beanFactory.getBean(
            APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
    } else {
        // 否则创建默认的 SimpleApplicationEventMulticaster
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster();
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, 
            this.applicationEventMulticaster);
    }
}

// 第 2 步：注册所有监听器
protected void registerListeners() {
    // (1) 注册编程式添加的监听器
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }
    
    // (2) 扫描 Bean 中的 @EventListener 注解
    Set<ApplicationListener<?>> listeners = new LinkedHashSet<>();
    for (String beanName : getBeanFactory().getBeanNamesForType(ApplicationListener.class)) {
        listeners.add(getBeanFactory().getBean(beanName, ApplicationListener.class));
    }
    listeners.forEach(getApplicationEventMulticaster()::addApplicationListener);
}
```

**关键点**：
- 默认广播器是 `SimpleApplicationEventMulticaster`
- 监听器在容器刷新阶段就被收集并注册到广播器中
- `@EventListener` 注解通过 `EventListenerMethodProcessor` 后置处理器处理

### 2.3 事件发布流程：publishEvent() 内部发生了什么

```java
// AbstractApplicationContext.publishEvent()
public void publishEvent(ApplicationEvent event) {
    publishEvent(event, null);
}

protected void publishEvent(ApplicationEvent event, ResolvableType eventType) {
    // 1. 包装事件（确保是 ApplicationEvent 类型）
    ApplicationEvent applicationEvent = 
        (event instanceof ApplicationEvent) ? 
        (ApplicationEvent) event : 
        new PayloadApplicationEvent<>(this, event);
    
    // 2. 多播器广播事件
    getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
}

// SimpleApplicationEventMulticaster.multicastEvent()
public void multicastEvent(ApplicationEvent event, ResolvableType eventType) {
    ResolvableType type = (eventType != null) ? eventType : resolveDefaultEventType(event);
    
    // 3. 获取所有匹配的监听器
    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        // 4. 执行监听器
        invokeListener(listener, event);
    }
}

protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
    // 5. 获取错误处理器（可选）
    ErrorHandler errorHandler = getErrorHandler();
    if (errorHandler != null) {
        errorHandler.handleError(new Throwable());
    } else {
        // 6. 直接调用监听器方法
        listener.onApplicationEvent(event);
    }
}
```

**调用链总结**：
```
publishEvent() 
  → multicastEvent() 
    → getApplicationListeners() (获取匹配的监听器)
      → invokeListener() (逐个调用)
        → listener.onApplicationEvent() (执行用户逻辑)
```

### 2.4 监听器匹配逻辑：如何知道哪个监听器听哪个事件

```java
// SimpleApplicationEventMulticaster.getApplicationListeners()
protected boolean supportsEvent(ApplicationListener<?> listener, 
                                 ApplicationEvent event, 
                                 ResolvableType eventType) {
    
    // 1. 获取监听器监听的事件类型
    ResolvableType declaredEventType = getGenericType(listener);
    
    // 2. 判断事件类型是否匹配
    return declaredEventType.isAssignableFrom(eventType);
}
```

**匹配规则**：
- 监听器声明的事件类型是发布事件类型的**父类/接口**，则匹配成功
- 支持泛型类型推断（通过 `ResolvableType` 解析）

### 2.5 异步事件：@Async + @EventListener

Spring 支持异步执行监听器，避免阻塞主流程：

```java
@Component
public class AsyncEventListener {
    
    @EventListener
    @Async  // 加这个注解即可异步执行
    public void handleUserRegistered(UserRegisteredEvent event) {
        // 这个逻辑在独立线程池执行，不阻塞发布线程
        sendEmail(event.getEmail(), "欢迎注册！");
    }
}
```

**底层原理**：
- `@Async` 注解由 `AsyncEventListenerFactoryBean` 处理
- 监听器被包装成异步执行的任务，提交到配置的线程池
- 需要配置 `@EnableAsync` 和自定义线程池

### 2.6 事务性事件：@TransactionalEventListener

如果需要在事务提交后才执行监听器逻辑：

```java
@Component
public class TransactionalEventListener {
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleUserRegistered(UserRegisteredEvent event) {
        // 只有在事务提交后才会执行
        sendEmail(event.getEmail(), "欢迎注册！");
    }
}
```

**phase 可选值**：
- `BEFORE_COMMIT`：事务提交前
- `AFTER_COMMIT`：事务提交后（最常用）
- `AFTER_ROLLBACK`：事务回滚后
- `AFTER_COMPLETION`：事务完成后（无论成功/回滚）

**底层原理**：
- 监听器被注册到 `TransactionSynchronizationManager`
- 事务提交时触发 `afterCommit()` 回调
- 避免"事务还没提交，邮件已发送"的问题

---

## 三、关键源码位置

| 类/方法 | 作用 | 源码位置 |
|---------|------|----------|
| `AbstractApplicationContext.refresh()` | 容器刷新入口 | `spring-context` |
| `AbstractApplicationContext.initApplicationEventMulticaster()` | 初始化广播器 | `spring-context` |
| `AbstractApplicationContext.registerListeners()` | 注册监听器 | `spring-context` |
| `AbstractApplicationContext.publishEvent()` | 发布事件 | `spring-context` |
| `SimpleApplicationEventMulticaster.multicastEvent()` | 广播事件 | `spring-context` |
| `SimpleApplicationEventMulticaster.invokeListener()` | 调用监听器 | `spring-context` |
| `TransactionalEventListener` | 事务事件监听器 | `spring-tx` |

---

## 四、最佳实践与注意事项

### 4.1 什么时候用事件机制

✅ **适合**：
- 需要解耦多个组件
- 一个操作触发多个后续动作
- 非核心逻辑（如通知、日志、统计）
- 需要异步/事务后执行

❌ **不适合**：
- 核心业务流程（难以追踪）
- 需要返回值的场景（事件是单向的）
- 过度使用（代码难以理解）

### 4.2 常见问题

**Q1：监听器执行异常会影响发布者吗？**
- 默认会抛出异常，影响主流程
- 解决方案：配置 `ErrorHandler` 或在监听器内 try-catch

**Q2：如何控制监听器执行顺序？**
- 实现 `Ordered` 接口或使用 `@Order` 注解
- 数值越小，优先级越高

**Q3：如何调试事件流程？**
- 开启 `org.springframework.context` 包 DEBUG 日志
- 可以看到事件发布和监听器调用的详细日志

### 4.3 性能优化

- 监听器逻辑尽量轻量，耗时操作放异步
- 大量监听器时考虑分组/分级
- 生产环境建议配置自定义线程池

---

## 五、总结

Spring 事件监听机制的核心价值：

1. **解耦**：发布者和监听器互不依赖
2. **扩展**：新增监听器无需修改发布者代码（开闭原则）
3. **灵活**：支持同步/异步、事务控制、顺序控制

底层实现的关键点：

1. **广播器**：`SimpleApplicationEventMulticaster` 负责管理和调用监听器
2. **匹配逻辑**：通过泛型类型推断确定哪些监听器接收哪些事件
3. **扩展能力**：通过 `@Async`、`@TransactionalEventListener` 实现高级特性

---

## 参考资料

- Spring Framework 官方文档：Events
- 《Spring 源码深度解析》第 8 章
- [[BeanFactory vs ApplicationContext 核心区别]] - Spring 容器基础

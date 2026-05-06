# Mockito 单测阅读方法

> 一份实用的 Mockito 单元测试阅读指南

---

## 📖 阅读顺序

按照以下顺序阅读任何 Mockito 单测，快速抓住测试意图：

### 1️⃣ 先看 DisplayName 和顶部注释

**目标：** 先知道测试意图，不要一上来就看 `when(...)`。

- 看 `@DisplayName` 注解
- 看最上面的"场景/测什么/为什么/长什么样"注释
- 理解这个测试要验证什么业务场景

### 2️⃣ 看 Arrange（准备区）

**目标：** 理解测试环境的搭建

- **哪个对象是"被测主体输入"** —— 这是核心
- **哪个对象是"关键业务数据"** —— 这是上下文
- 识别测试依赖的各个组件

### 3️⃣ 看 `when(...).thenReturn(...)`

**目标：** 理解 Mock 行为的设定

- 把它当成"**给被测方法铺路**"的环境设定
- **不是业务结论本身**
- **重点只盯和目标相关的 mock**，忽略无关的 Mock 配置

### 4️⃣ 看 When（触发动作）

**目标：** 找到测试的执行点

- 通常只有一行，例如：
  ```java
  connectionService.startConnection(connectionId);
  ```
- 这行是"**触发动作**"，是测试真正执行的地方

### 5️⃣ 看 Then（断言区）

**目标：** 验证最终行为是否符合预期

- **只看最终行为是否符合目标**
- 使用 `ArgumentCaptor` 查看方法收到的参数是什么
  ```java
  // 例：查看 prepareSession 收到的 params 是什么
  ArgumentCaptor<SessionParams> captor = ArgumentCaptor.forClass(SessionParams.class);
  verify(service).prepareSession(captor.capture());
  ```
- 使用 `verify(..., never())` 确认没有访问不该访问的资源
  ```java
  // 例：确认没有访问旧表
  verify(oldTableService, never()).access(any());
  ```

---

## 🎯 核心原则

| 原则 | 说明 |
|------|------|
| **先意图后细节** | 先理解"为什么要测这个"，再看"怎么测的" |
| **关注相关 Mock** | 只盯和目标相关的 mock，忽略噪音 |
| **断言即结论** | Then 区的验证才是测试的真正结论 |
| **when 是铺路** | `when().thenReturn()` 只是环境设定，不是业务逻辑 |

---

## 📝 示例模板

```java
@DisplayName("当连接 ID 有效时，应该启动新连接并准备会话")
@Test
void testStartConnection_validId_shouldPrepareSession() {
    // ==== Arrange 准备区 ====
    // 被测主体输入
    String connectionId = "conn_123";
    
    // Mock 设定（铺路）
    when(mockRepository.findById(connectionId))
        .thenReturn(Optional.of(validConnection));
    
    // ==== When 触发动作 ====
    connectionService.startConnection(connectionId);
    
    // ==== Then 断言区 ====
    // 验证核心行为
    ArgumentCaptor<SessionParams> captor = ArgumentCaptor.forClass(SessionParams.class);
    verify(sessionService).prepareSession(captor.capture());
    assertThat(captor.getValue().getConnectionId()).isEqualTo(connectionId);
    
    // 验证没有副作用
    verify(legacyService, never()).migrateData();
}
```

---

## 💡 小贴士

1. **不要被 Mock 数量吓到** —— 很多 Mock 可能和当前测试目标无关
2. **DisplayName 是最好的导航** —— 好的测试命名直接告诉你它在测什么
3. **ArgumentCaptor 是神器** —— 用它可以看到方法实际收到的参数
4. **`never()` 同样重要** —— 确认"没有发生不该发生的事"和确认"发生了该发生的事"一样重要

---

*最后更新：2026-03-30*

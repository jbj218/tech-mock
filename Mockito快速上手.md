# Mockito 快速上手

> 目标：**30 分钟能写出第一个能跑的 Mock 单元测试**。不求全，先会用；遇到问题再查第 8 节速查表。
> 环境：Java 11+、JUnit 5、Mockito 5。

**目录**

1. [Mockito 是什么](#1-mockito-是什么解决什么问题)
2. [加依赖](#2-加依赖)
3. [第一个完整例子](#3-第一个完整例子可直接跑)
4. [常用注解](#4-常用注解一览)
5. [五个核心动作](#5-五个核心动作)
6. [常见场景速查](#6-常见场景速查直接抄)
7. [十个常见坑](#7-十个常见坑)
8. [Cheat Sheet](#8-cheat-sheet打印版)
9. [什么时候不该用 Mockito](#9-什么时候不该用-mockito)
10. [5 分钟练习](#10-5-分钟练习)
11. [下一步学什么](#11-下一步学什么)

---

## 0. 先记住这三行

```java
@Mock UserDao userDao;                                        // 1. 造一个假对象
when(userDao.findById(1L)).thenReturn(user);                  // 2. 规定它怎么回答
verify(userDao).save(any(User.class));                        // 3. 检查它被怎么调用
```

Mockito 干的事就这三件：**造假对象 → 编造返回值 → 验证调用**。

---

## 1. Mockito 是什么，解决什么问题

单元测试要测的是「当前这个类」的逻辑，但它的依赖（数据库、RPC、短信、时间……）往往：

- 跑不起来（没环境、没网络）；
- 跑得慢、不稳定；
- 很难制造异常分支（超时、返回 null、抛异常）。

Mockito 用**动态代理**生成依赖的假实现，让你：

| 能力 | 方法 |
|---|---|
| 不连真实依赖 | `mock()` / `@Mock` |
| 指定某个调用返回什么 | `when(...).thenReturn(...)` |
| 让某个调用抛异常 | `thenThrow(...)` |
| 检查依赖有没有被正确调用 | `verify(...)` |
| 只替换对象的部分方法 | `spy()` / `@Spy` |

一句话：**Mockito 负责「依赖」，JUnit 负责「断言」，两者配合完成一个单测。**

---

## 2. 加依赖

### Maven

```xml
<dependency>
  <groupId>org.junit.jupiter</groupId>
  <artifactId>junit-jupiter</artifactId>
  <version>5.11.3</version>
  <scope>test</scope>
</dependency>

<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-core</artifactId>
  <version>5.14.2</version>
  <scope>test</scope>
</dependency>

<!-- JUnit 5 集成：提供 @ExtendWith(MockitoExtension.class) -->
<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-junit-jupiter</artifactId>
  <version>5.14.2</version>
  <scope>test</scope>
</dependency>
```

### Gradle

```groovy
testImplementation("org.junit.jupiter:junit-jupiter:5.11.3")
testImplementation("org.mockito:mockito-core:5.14.2")
testImplementation("org.mockito:mockito-junit-jupiter:5.14.2")
```

> Spring Boot 项目通常已经自带 `spring-boot-starter-test`，里面就包含 JUnit 5 + Mockito，无需重复引入。

---

## 3. 第一个完整例子（可直接跑）

### 3.1 被测代码

```java
public class User {
    private final Long id;
    private final String name;
    public User(Long id, String name) { this.id = id; this.name = name; }
    public Long getId() { return id; }
    public String getName() { return name; }
}

public interface UserDao {
    User findById(Long id);
    void save(User user);
}

public interface SmsClient {
    void sendWelcome(String name);
}

public class UserService {
    private final UserDao userDao;
    private final SmsClient smsClient;

    public UserService(UserDao userDao, SmsClient smsClient) {
        this.userDao = userDao;
        this.smsClient = smsClient;
    }

    public User create(Long id, String name) {
        if (userDao.findById(id) != null) {
            throw new IllegalStateException("用户已存在");
        }
        User user = new User(id, name);
        userDao.save(user);
        smsClient.sendWelcome(name);
        return user;
    }
}
```

### 3.2 测试代码

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)   // 关键：让 @Mock / @InjectMocks 生效
class UserServiceTest {

    @Mock
    UserDao userDao;                  // 假 DAO

    @Mock
    SmsClient smsClient;              // 假短信客户端

    @InjectMocks
    UserService userService;          // 自动把上面两个 mock 注入进来

    @Test
    void create_should_save_and_send_sms() {
        // 1) 打桩：查不到用户
        when(userDao.findById(1L)).thenReturn(null);

        // 2) 执行
        User user = userService.create(1L, "小明");

        // 3) 断言 + 验证
        assertNotNull(user);
        assertEquals("小明", user.getName());
        verify(userDao).save(any(User.class));
        verify(smsClient).sendWelcome("小明");
    }

    @Test
    void create_should_throw_when_user_exists() {
        when(userDao.findById(1L)).thenReturn(new User(1L, "old"));

        assertThrows(IllegalStateException.class,
                () -> userService.create(1L, "new"));

        verify(userDao, never()).save(any());
        verifyNoInteractions(smsClient);
    }
}
```

**要点**
- `@ExtendWith(MockitoExtension.class)` 别忘了，否则 `@Mock` 字段是 `null`。
- `@InjectMocks` 会按「构造器 → setter → 字段」的顺序注入 mock。
- 没打桩的方法返回**默认值**：对象 `null`、`int 0`、`boolean false`、空集合。

---

## 4. 常用注解一览

| 注解 | 作用 | 备注 |
|---|---|---|
| `@ExtendWith(MockitoExtension.class)` | 启用 Mockito 扩展 | JUnit 5 必须写 |
| `@Mock` | 创建 mock 对象 | 接口、类都可以 |
| `@InjectMocks` | 创建被测对象并注入 mock | 常用在 Service 上 |
| `@Spy` | 包装真实对象，只替换部分方法 | 其余方法走真实逻辑 |
| `@Captor` | 声明参数捕获器 | 配合 `verify` 使用 |
| `@MockitoSettings(strictness = Strictness.LENIENT)` | 放宽严格模式 | 慎用，优先删掉多余打桩 |

---

## 5. 五个核心动作

### 5.1 造对象：`mock()` 与 `@Mock`

```java
UserDao dao = Mockito.mock(UserDao.class);   // 编程式
```

```java
@Mock UserDao dao;                            // 注解式（推荐，配合 MockitoExtension）
```

> 只 mock **外部依赖**（DAO、RPC、MQ、第三方 SDK），不要 mock 实体类、DTO、`List` 这类值对象。

### 5.2 打桩：`when(...).thenReturn(...)`

```java
when(userDao.findById(1L)).thenReturn(new User(1L, "小明"));

// 连续调用返回不同值
when(userDao.findById(1L))
        .thenReturn(new User(1L, "第 1 次"))
        .thenReturn(new User(1L, "第 2 次"));

// 抛异常
when(userDao.findById(2L)).thenThrow(new IllegalArgumentException("id 非法"));

// 根据入参动态返回
when(userDao.findById(anyLong()))
        .thenAnswer(inv -> new User(inv.getArgument(0), "动态"));
```

### 5.3 验证：`verify(...)`

```java
verify(userDao).save(any(User.class));                 // 调用过 1 次
verify(userDao, times(2)).save(any());                 // 调用过 2 次
verify(userDao, never()).delete(anyLong());            // 从未调用
verify(userDao, atLeastOnce()).findById(anyLong());    // 至少 1 次
verifyNoInteractions(smsClient);                       // 完全没有交互
verifyNoMoreInteractions(userDao);                     // 除了已验证的，没有别的调用

// 顺序验证
InOrder inOrder = inOrder(userDao, smsClient);
inOrder.verify(userDao).save(any());
inOrder.verify(smsClient).sendWelcome(anyString());
```

> `verify` 验证的是**交互**；返回值对不对，仍然要用 JUnit 的 `assertEquals` / `assertThrows` 断言。

### 5.4 参数匹配器：`any()` / `eq()`

```java
when(userDao.findById(anyLong())).thenReturn(user);
when(userDao.findByName(eq("小明"))).thenReturn(user);

verify(userDao).save(argThat(u -> u.getName().startsWith("小")));
```

**铁律：一旦用了匹配器，所有参数都必须用匹配器。**

```java
// ❌ InvalidUseOfMatchersException（假设方法有两个参数）
verify(userDao).save(any(User.class), 1L);

// ✅
verify(userDao).save(any(User.class), eq(1L));
```

常用匹配器：`any()`、`anyLong()`、`anyString()`、`any(Class)`、`eq(value)`、`isNull()`、`argThat(lambda)`。
注意：`any()` 能匹配 `null`，`anyString()` **不能**。

### 5.5 捕获参数：`ArgumentCaptor`

```java
@Captor ArgumentCaptor<User> userCaptor;

@Test
void should_save_user_with_correct_name() {
    when(userDao.findById(1L)).thenReturn(null);

    userService.create(1L, "小明");

    verify(userDao).save(userCaptor.capture());
    User saved = userCaptor.getValue();
    assertEquals("小明", saved.getName());
}
```

> 当你要断言「传给依赖的参数内容」时用它，比 `argThat` 更好调试。

---

## 6. 常见场景速查（直接抄）

### 6.1 依赖返回 null 或空集合

```java
when(userDao.findById(anyLong())).thenReturn(null);
when(userDao.findAll()).thenReturn(Collections.emptyList());
```

### 6.2 void 方法：抛异常 / 什么都不做

> `when(void 方法)` 编译不过，要用 `doXxx().when(mock).method()`。

```java
doThrow(new RuntimeException("发送失败")).when(smsClient).sendWelcome(anyString());
doNothing().when(userDao).save(any(User.class));
```

### 6.3 部分模拟：`spy`

```java
@Spy
List<String> list = new ArrayList<>();

doReturn(100).when(list).size();   // 只替换 size()
list.add("真实添加");               // 其余方法走真实逻辑
```

注意：对 spy 用 `when(spy.method())` 会**真的调用一次**方法，容易踩坑，统一用 `doReturn(...).when(spy)`。

### 6.4 静态方法（Mockito 5 默认支持）

```java
try (MockedStatic<IdUtils> mocked = mockStatic(IdUtils.class)) {
    mocked.when(IdUtils::nextId).thenReturn(100L);

    assertEquals(100L, IdUtils.nextId());

    mocked.verify(IdUtils::nextId, times(1));
}
```

> `mockStatic` 只在 `try-with-resources` 代码块内有效；用完即关闭，不要跨测试方法共享。

### 6.5 final 类 / final 方法

Mockito 5 默认使用 inline mock maker，**可以直接 mock**：

```java
final class FinalService { String hello() { return "real"; } }

FinalService mock = mock(FinalService.class);
when(mock.hello()).thenReturn("mocked");
```

> 老项目用 Mockito 4 或以下时，需要 `mockito-inline` 依赖。

### 6.6 超时验证（异步场景，少用）

```java
verify(userDao, timeout(500)).save(any());
```

---

## 7. 十个常见坑

1. **忘了 `@ExtendWith(MockitoExtension.class)`** → `@Mock` 为 `null`，报 NPE。
2. **匹配器和原始值混用** → `InvalidUseOfMatchersException`，全部换成 `eq()`。
3. **对 void 方法用 `when`** → 编译错误，改用 `doThrow/doNothing`。
4. **`when(spy.foo())` 触发真实逻辑** → 改用 `doReturn().when(spy)`。
5. **严格模式报 `UnnecessaryStubbingException`** → 有打桩没用上，删掉或 `lenient()`。
6. **用 `@InjectMocks` 但被测类没有无参构造/字段可注入** → 检查构造器参数，或手动 `new`。
7. **mock 了实体/DTO** → 断言时字段全是默认值，得不偿失；值对象用真实对象。
8. **只 verify 不断言** → 测试“调用了”不代表“结果对”，两者都要。
9. **过度 mock** → 每个依赖都 mock，测试和实现强耦合，改一行代码就挂；能真跑的真跑。
10. **`any()` 与 `null`** → `anyString()`、`anyLong()` 不匹配 `null`，需要 `nullable(String.class)` 或 `isNull()`。

---

## 8. Cheat Sheet（打印版）

```java
// ===== 创建 =====
mock(UserDao.class);  @Mock UserDao dao;  @Spy RealService spy;

// ===== 打桩 =====
when(dao.findById(1L)).thenReturn(user);
when(dao.findById(anyLong())).thenThrow(new RuntimeException());
when(dao.findById(anyLong())).thenAnswer(inv -> user);
when(dao.findById(1L)).thenReturn(a).thenReturn(b);
doThrow(new RuntimeException()).when(dao).delete(1L);
doReturn(user).when(spy).findById(1L);

doNothing().when(dao).save(any());

// ===== 验证 =====
verify(dao).save(any());
verify(dao, times(2)).save(any());
verify(dao, never()).delete(anyLong());
verify(dao, atLeastOnce()).findById(anyLong());
verifyNoInteractions(dao);
verifyNoMoreInteractions(dao);
verify(dao, timeout(500)).save(any());

// ===== 参数 =====
any()  anyLong()  anyString()  any(User.class)  eq(1L)  isNull()  argThat(u -> ...)
@Captor ArgumentCaptor<User> captor;
verify(dao).save(captor.capture());  captor.getValue();  captor.getAllValues();

// ===== 顺序 =====
InOrder inOrder = inOrder(dao, smsClient);
inOrder.verify(dao).save(any());
inOrder.verify(smsClient).sendWelcome(anyString());

// ===== 静态 =====
try (MockedStatic<IdUtils> m = mockStatic(IdUtils.class)) {
    m.when(IdUtils::nextId).thenReturn(1L);
}
```

---

## 9. 什么时候不该用 Mockito

- **测纯函数/工具类**：直接输入输出断言，不需要 mock。
- **值对象、集合、String**：用真实对象，别 mock。
- **集成测试**：要验证 SQL、事务、序列化时，用真实依赖（Testcontainers / H2）更可靠。
- **私有方法**：不要为了测私有方法去反射；通过公有方法间接覆盖，或重构。

> 记住：**mock 的是「边界」，不是「逻辑」**。mock 越多，测试越脆。

---

## 10. 5 分钟练习

拿一个你项目里的 `XxxService`，完成下面三步：

1. 找出它的 1~2 个外部依赖（DAO / RPC / MQ），用 `@Mock` 替换；
2. 给正常分支写一个测试：打桩 + 调用 + 断言返回值 + `verify` 关键调用；
3. 给异常分支写一个测试：`thenThrow` 或 `assertThrows` + `verify(..., never())`。

跑通这两个，Mockito 就算入门了。

---

## 11. 下一步学什么

- `MockitoExtension` 的严格模式与 `lenient()`；
- `ArgumentCaptor` + `argThat` 的组合；
- `mockStatic` 与 `MockedConstruction`（构造器 mock）；
- Spring Boot 里 `@MockBean` / `@MockitoBean` 与 `@WebMvcTest`；
- 测试覆盖率工具 JaCoCo。

官方文档：<https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html>

---

*本文示例基于 Mockito 5.x + JUnit 5，代码可直接复制到项目中运行。*




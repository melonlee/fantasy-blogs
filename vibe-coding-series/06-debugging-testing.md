# Vibe Coding 实战（六)：调试与测试——Vibe Coding 下的 Java 调试

---

![Java 调试](./images/06-debugging-testing.jpg)

写代码容易，调试难。

Vibe coding 放大了这个问题：AI 生成代码不会一次就完全对，而且代码不是你写的，你可能不完全理解它为什么这样写。调试的时候，你会面临一个尴尬的局面——代码是 AI 生成的，但你得负责找出哪里错了。

这篇文章讲 Java 调试和测试的 vibe coding 方法。

## 调试的三个层次

Java 调试有三个层次：编译错误、运行时错误、逻辑错误。

**编译错误：最低级也最容易修。**

```bash
mvn compile
```

跑一下编译，Java 编译器会告诉你哪里语法不对。这是检验 AI 生成代码的第一步。

常见编译错误：
- 类型不匹配（AI 经常在泛型上出错）
- import 缺失
- 方法签名不对（AI 假设了一个不存在的方法）

这些错误看编译输出就能修，不需要调试。

**运行时错误：需要断点调试。**

NullPointerException、IllegalArgumentException、数据库连接失败——这些只有在运行时才会暴露。

Cursor 的调试功能和 VS Code 完全一致。在代码行号左边点击设置断点，按 F5 启动调试。

对于 Spring Boot 项目，配置调试启动：

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "Spring Boot",
      "request": "launch",
      "mainClass": "com.example.Application",
      "projectName": "my-app",
      "args": ""
    }
  ]
}
```

断点停在怀疑出错的那一行，检查变量值。

**逻辑错误：最难的。**

程序能跑，但结果不对。这种错误没有异常，全靠结果判断。

逻辑错误的调试，靠的是缩小范围：在代码里找"第一次出现不对结果"的位置，然后分析为什么不对。

```java
public BigDecimal calculateOrderTotal(Order order) {
    BigDecimal subtotal = order.getItems().stream()
        .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
        .reduce(BigDecimal.ZERO, BigDecimal::add);

    BigDecimal discount = calculateDiscount(order); // 这里可能有 bug
    return subtotal.subtract(discount);  // 还是这里
}
```

AI 生成这段代码的时候，逻辑是对的——但 discount 计算可能在某些边界条件下出错。断点打在 discount 计算那一行，逐步执行，观察 discount 的值是否符合预期。

## AI 辅助调试

Cursor 和 Claude Code CLI 都可以辅助调试。

**方式一：复制错误信息给 AI。**

把完整的堆栈信息复制给 Cursor Composer 或 Claude Code CLI：

```
这个错误是什么意思？
org.springframework.dao.EmptyResultDataAccessException:
Incorrect result size: expected 1, actual 0
```

AI 会分析堆栈，告诉你可能的原因。

**方式二：让 AI 分析变量值。**

在断点处暂停，复制变量值给 AI：

```
这个 order 对象的 items 列表里有 5 个元素，但 calculateDiscount 返回了 0。
为什么 discount 会是 0？
```

AI 会分析逻辑，告诉你可能的 bug。

**方式三：让 AI 写测试来重现 bug。**

```
为这个 calculateDiscount 方法写一个测试用例，覆盖 discount 为 0 的场景
```

用测试重现 bug，比肉眼找更可靠。

## 测试：Vibe Coding 的一道关

AI 生成的代码需要测试覆盖。但 AI 本身生成的测试也不一定完全对。

最佳做法：

1. 让 AI 生成测试代码
2. 人工审查测试用例是否覆盖了关键场景
3. 运行测试，确认通过

**用 Cursor 生成测试。**

```bash
> 为 UserService 的 login 方法生成单元测试，覆盖：
- 正常登录
- 用户名不存在
- 密码错误
```

Cursor 会生成 JUnit 5 + Mockito 的测试：

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private PasswordEncoder passwordEncoder;

    @InjectMocks
    private UserService userService;

    @Test
    void login_WithValidCredentials_ReturnsToken() {
        // given
        User user = new User("john", "hashedPassword", "john@example.com");
        when(userRepository.findByUsername("john")).thenReturn(Optional.of(user));
        when(passwordEncoder.matches("password123", "hashedPassword")).thenReturn(true);

        // when
        String token = userService.login("john", "password123");

        // then
        assertNotNull(token);
    }

    @Test
    void login_WithInvalidPassword_ThrowsException() {
        // given
        User user = new User("john", "hashedPassword", "john@example.com");
        when(userRepository.findByUsername("john")).thenReturn(Optional.of(user));
        when(passwordEncoder.matches("wrongPassword", "hashedPassword")).thenReturn(false);

        // when & then
        assertThrows(InvalidCredentialsException.class,
            () -> userService.login("john", "wrongPassword"));
    }
}
```

**测试覆盖率的及格线。**

Java 项目通常要求 70% 以上的行覆盖率。但覆盖率不是目标，可靠性才是。

对于 vibe coding 来说，我建议关注这几个关键指标的覆盖：
- 所有 public 方法至少有一个测试
- 所有分支路径（if/else、try/catch）都有测试
- 边界条件（空集合、零值、负数）有测试

**AI 生成测试的常见问题。**

AI 有时候会生成测试，但测试本身可能有错。常见问题：
- Mock 对象的 stub 不对
- 断言写反了（expected 和 actual 位置颠倒）
- 没有覆盖 null 输入

运行测试是检验 AI 生成测试的第一步。

## 集成测试：Vibe Coding 的真正挑战

单元测试相对简单。真正的挑战是集成测试。

集成测试需要真实的数据库、真实的网络、真实的 Spring 上下文。这些不能靠 Mock。

用 Testcontainers 来做集成测试：

```java
@Testcontainers
@SpringBootTest
class UserRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("test")
        .withUsername("test")
        .withPassword("test");

    @Test
    void saveUser_PersistsAndCanBeRetrieved() {
        User user = new User("john", "hash", "john@example.com");
        userRepository.save(user);

        Optional<User> found = userRepository.findByUsername("john");

        assertTrue(found.isPresent());
        assertEquals("john", found.get().getUsername());
    }
}
```

让 AI 生成这个测试框架的初始代码，然后人工补充具体的测试用例。

## 调试的最佳实践

**习惯一：每生成一个功能，跑一下测试。**

不要等 bug 积累到一定程度再调试。AI 生成代码后，立刻跑测试。发现问题的成本越低，修复的成本越低。

**习惯二：保持代码在"可运行"状态。**

每次修改后，确保 `mvn compile` 和 `mvn test` 都能跑。不要让代码进入"编译不过也不知道怎么修"的状态。

**习惯三：git commit 频繁。**

每次 AI 生成并验证通过的代码，及时 commit。git 历史是调试的重要工具——如果某次 commit 引入了问题，可以快速回滚。

**习惯四：保留 AI 的原始输出记录。**

Cursor Composer 的 Diff 视图会保留 AI 的原始修改。如果需要回溯，这个记录有用。

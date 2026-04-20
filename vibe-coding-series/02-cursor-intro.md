# Vibe Coding 实战（二)：Cursor 入门——Java 开发者的 AI 编程环境

---

![Cursor IDE intro](./images/02-cursor-intro.jpg)

Cursor 是目前最流行的 AI 编程 IDE 之一。它基于 VS Code，集成了 Claude、GPT-4 等大语言模型，支持多文件编辑、Composer Agent、Debug 集成。

对 Java 开发者来说，Cursor 是可用的——支持 Spring Boot、支持 Maven/Gradle、支持 JUnit 测试运行。

这篇文章的目标很简单：装好 Cursor、跑起一个 Spring Boot 项目、用 AI 生成代码。

## 安装与基础配置

Cursor 的安装很简单。去 cursor.com 下载，安装，登录（需要账号）。

安装完成后，有几个关键的配置需要做：

**第一步：绑定 AI 模型。**

Cursor 支持多个 AI 提供方：Claude（通过 Anthropic API）、GPT-4（通过 OpenAI API）、Gemini 等。

对于 Java 开发者，我建议用 Claude，因为它的代码能力最强。

配置方式：`Cmd+K` 打开 AI 面板，设置 `claude-3-5-sonnet` 或 `claude-opus-4` 作为默认模型。

**第二步：安装 Java 插件。**

Cursor 基于 VS Code，所以 VS Code 的 Java 插件在 Cursor 里同样可用。

按 `Cmd+Shift+X` 打开扩展市场，搜索并安装：
- **Extension Pack for Java**（必装，包含 Language Support for Java、Debugger for Java、Test Runner for Java）
- **Spring Boot Extension Pack**（必装，包含 Spring Boot Tools、Spring Initializr Java Support）
- **Maven for Java**（如果用 Maven）
- **Gradle for Java**（如果用 Gradle）

**第三步：打开 Java 项目。**

Cursor 支持 Maven 和 Gradle 项目。打开项目根目录，Cursor 会自动识别 pom.xml 或 build.gradle，开始索引项目。

索引过程可能需要几分钟，取决于项目大小。索引完成后，AI 就能理解项目的完整结构。

## 核心功能：Tab 补全、Composer、Cursor Chat

Cursor 有三个核心功能，每个都值得了解。

**Tab 补全。**

这是最基础的功能。在写代码时，Cursor 会预测你接下来要写什么，按 `Tab` 接受补全。

对于 Java 来说，这个功能比 GitHub Copilot 更准确——因为 Cursor 的模型专门针对代码做过优化。

补全适合的场景：补全函数签名、补全 import 语句、补全常见的代码模式。

不适合的场景：补全复杂业务逻辑、补全整个文件。补全只是加速打字，不是替代思考。

**Composer（Cmd+I）。**

Composer 是 Cursor 的核心 AI 功能。按 `Cmd+I` 打开 Composer 面板，可以对整个项目提问、生成代码、修改多个文件。

```
你想做什么？
> 在 com.example.user包里创建一个 UserService，包含注册和登录方法
```

Composer 会分析项目结构，生成符合项目规范的代码。它可以：
- 读取多个文件的内容
- 同时修改多个文件
- 生成新文件
- 解释现有代码

对于 Spring Boot 项目，Composer 可以生成 Controller、Service、Repository 的一整套代码。

**Cursor Chat（Cmd+L）。**

这是全局的 AI 对话。它可以回答关于项目的问题、解释代码、生成文档。

Cursor Chat 和 Composer 的区别是：Chat 不会自动读取项目上下文，需要你主动 @ 文件或 @ 代码段。

## 实战：生成一个 CRUD 功能

用一个具体的例子来展示 Cursor 的实际用法。

假设你有一个 Spring Boot 项目，已经有 JPA Entity：

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password;

    private String email;
    private boolean enabled = true;
}
```

你想生成对应的 Service、Repository、Controller。

**方式一：用 Composer。**

打开 Composer（`Cmd+I`），输入：

```
为 User 实体生成完整的 Spring Boot REST API，包含：
1. UserRepository（继承 JpaRepository）
2. UserService（包含 CRUD 方法 + findByUsername）
3. UserController（包含 POST /users、GET /users/{id}、PUT、DELETE）
4. 使用 @RestController、@Service、@Repository 注解
5. 密码用 BCrypt 加密
6. 加上基本的验证（@Valid）
```

Composer 会分析项目结构，生成符合 Spring Boot 规范的代码。

生成后你会看到 Diff 视图——显示哪些文件会新增/修改。你可以逐个确认，也可以一次性接受全部。

**方式二：用 Inline Chat。**

在代码里选中一段，按 `Cmd+K` 打开 inline chat，问：

```
为这个 Controller 添加 Swagger 文档注解
```

Cursor 会直接修改当前文件。

## Rules：让 AI 遵守项目规范

Cursor 的 Rules 功能是控制 AI 行为的关键。

Rules 让你定义 AI 在处理你的项目时必须遵守的规则。这些规则可以是：

- 代码风格（用什么注解、用什么命名规范）
- 禁止的模式（不要用 @Autowired 字段注入）
- 必做的检查（每个 public 方法必须有 @Nullable 检查）

配置方式：在项目根目录创建 `.cursor/rules/` 目录，每个 .md 文件定义一组规则。

例如，创建一个 `java-spring.md`：

```markdown
# Java Spring Boot 规范

## 代码风格
- 使用构造器注入（构造器注入优于 @Autowired 字段注入）
- Service 层必须加 @Slf4j
- 所有 @Entity 必须加 @NoArgsConstructor 和 @AllArgsConstructor

## 禁止的模式
- 禁止在 @Entity 类中直接暴露 id 的 setter
- 禁止在 Controller 中写业务逻辑
- 禁止用 String 处理密码，用 BCryptPasswordEncoder

## 必做的检查
- 所有 public 方法入口必须有 null 检查
- 所有 API endpoint 必须有 @Operation 文档注解
```

Rules 的作用是让 AI 的行为更符合团队规范。没有 Rules，AI 生成代码是"通用的"；有了 Rules，AI 生成代码是"符合项目要求的"。

## 调试：Cursor 的 Debug 功能

Cursor 的调试功能和 VS Code 完全一致。

在代码行号左边点击，可以设置断点。按 `F5` 启动调试，程序会在断点处暂停，可以检查变量值、调用栈。

对于 Spring Boot 项目：

```bash
# 在项目目录启动调试模式
./mvnw spring-boot:run -Dspring-boot.run.jvmArguments="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005"
```

然后在 Cursor 的 Debug 面板里配置一个 "Attach to Remote JVM" 的调试配置，连接到 localhost:5005。

AI 可以辅助调试。在断点暂停时，选中一段报错信息或异常，按 `Cmd+K` 问 AI：

```
为什么这个 List 在这里是空的？
```

Cursor 会分析上下文，给出可能的原因。

## Java 开发者用 Cursor 的几个注意点

**1. 模型选择很重要。**

Claude 3.5 Sonnet 在代码生成上比 GPT-4 更强，特别是 Java。如果用 GPT-4，可能会遇到更多"语法正确但不符合 Spring 规范"的问题。

**2. 不要一次生成太多代码。**

Composer 可以一次生成整个 Controller，但更好的做法是分步骤：先生成 Repository，再生成 Service，最后生成 Controller。每步审查后再继续。

**3. Rules 是必要的投资。**

花时间写好 .cursor/rules/ 里的规则，长期回报很高。它让 AI 生成代码的质量更稳定，减少校正次数。

**4. 类型检查是最后一道关。**

AI 生成的 Java 代码可能在类型上出错。Maven/Gradle 的编译是检验 AI 代码的第一道关。每次 AI 生成代码后，先跑一次 `mvn compile`，确认编译通过再继续。

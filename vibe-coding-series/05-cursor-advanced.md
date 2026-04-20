# Vibe Coding 实战（五）：Cursor 进阶——Rules、Context、Agents

---

![Cursor 进阶功能](./images/05-cursor-advanced.jpg)

学会基础用法之后，Cursor 有三个进阶功能可以把 AI 生成代码的质量提升一个级别：Rules、Context、Agents。

这三个功能不是"锦上添花"，是实际减少校正次数、提高代码一致性的关键。

## Rules：让 AI 遵守项目规范

Rules 是 Cursor 最重要的功能之一。它让你定义 AI 在处理你的项目时必须遵守的规则。

配置方式：在项目根目录创建 `.cursor/rules/` 目录，每个 .md 文件定义一组规则。

```markdown
# java-spring-boot.md

## 强制规范
- Service 层必须使用构造器注入，不允许字段注入
- Entity 必须有 @NoArgsConstructor @AllArgsConstructor @Getter @Setter
- 所有 @Entity 的 id 字段必须是 Long 类型，使用 @GeneratedValue
- REST Controller 必须加 @RestController 和 @RequestMapping
- 所有 public 方法必须有 Javadoc 注释

## 禁止规范
- 禁止在 Controller 层写业务逻辑
- 禁止用 SELECT * 查询，必须明确列出字段
- 禁止硬编码错误消息，必须从 ErrorCodes 枚举获取
- 禁止用 System.out.println，必须用 @Slf4j 的 log

## 代码格式
- 缩进用 4 空格
- 大括号换行（K&R 风格）
- 每行最多 120 字符
```

Rules 生效的方式：当你打开 Cursor 并加载项目后，所有 AI 生成代码都会遵守这些规则。

你不需要在每次 prompt 里重复"记得用构造器注入"，Rules 会自动应用。

### Rules 的分层

可以在不同层级定义 Rules：

- **全局 Rules**：`~/.cursor/rules/`——适用于所有项目
- **项目级 Rules**：`.cursor/rules/`——只适用于当前项目

项目级 Rules 优先于全局 Rules。

### Rules 的粒度

一个 Rules 文件可以包含多个模块的规范：

```markdown
# backend-standards.md

## Java 规范
- 构造器注入
- @Slf4j 日志

## Spring Boot 规范
- @Service 替代 @Component
- @Repository 替代 @Component

## API 规范
- 统一响应格式
- 统一异常处理

## 测试规范
- 所有 public 方法必须有对应测试
- 测试用 JUnit 5 + Mockito
```

## Context：管理 AI 能看到什么

Cursor 的 Context 功能让你控制 AI 能看到哪些文件。默认情况下，Composer 会自动分析项目结构，选择相关的文件。

但这个选择不总是对的。你需要手动指定 Context。

**@ 语法。**

在 Composer 里用 `@` 符号引用文件：

```
@UserService.java @UserRepository.java
为这两个类添加 equals 和 hashCode 方法
```

**@folder 语法。**

```
@src/main/java/com/example/service
列出这个目录下所有 Service 类
```

**@web 语法。**

```
@web
搜索一下 Spring Boot 3.2 的新特性
```

**Context 的最佳实践。**

不要让 AI 一次看到太多文件。最好控制在 5-10 个相关文件以内。太多文件会让 AI 分心，生成的结果不精确。

```bash
# 指定关键文件
@src/main/java/com/example/User.java
@src/main/java/com/example/UserDTO.java
@src/main/java/com/example/UserRepository.java
生成 UserService，实现注册和登录逻辑
```

## Agents：让 AI 自动完成多步任务

Cursor 的 Agents 功能让 AI 能够自主执行多步任务，不只是生成代码。

按 `Cmd+I` 然后选择 "Agent" 模式。

**Agent 和普通 Composer 的区别。**

普通 Composer：你说一句话，AI 生成一次代码，然后就结束。

Agent：你说一个目标，AI 自动拆解步骤，逐个执行，每步后验证结果，直到完成目标。

```bash
# 普通 Composer
> 创建一个 UserController

# Agent
> 把这个 User 模块的重试逻辑改成指数退避
# AI 自动分析代码 → 找到现有实现 → 制定修改方案 → 逐个修改 → 验证
```

### 实用的 Agent 用法

**用法一：大规模重构。**

把某个旧模块的代码风格从直接 JDBC 改成 JPA。

```
把所有 XXXRepository 中的原始 SQL 查询改成 JPA 方法
```

Agent 会找到所有相关的 Repository 文件，逐个修改，验证是否编译通过。

**用法二：生成测试。**

为一个模块生成完整的测试套件。

```
为 OrderService 生成单元测试，覆盖所有 public 方法
```

Agent 会分析 Service 的每个方法，生成对应的测试用例。

**用法三：代码迁移。**

从一个框架迁移到另一个框架。

```
把这个项目从 Spring Security 5 迁移到 Spring Security 6
```

Agent 会分析当前的 Security 配置，生成新的配置代码，处理 Breaking Changes。

### Agents 的限制

Agents 很强大，但有限制：

1. **Agent 不能访问浏览器。** 如果需要查文档，需要你手动提供上下文。

2. **Agent 可能走错方向。** 长时间任务中，Agent 可能偏离最初目标。需要你定期审查。

3. **Agent 行动不可预测。** 在大规模修改前，先在小范围测试，确认行为符合预期。

## Rules + Context + Agents 的组合

这三个功能组合起来，能覆盖大多数复杂场景：

```
场景：为一个新的 Payment 模块生成完整的代码

步骤：
1. 先在 .cursor/rules/ 里定义 Payment 模块的规范
2. 在 Composer 里指定 Context：
   @src/main/java/com/example/entity/Payment.java
   @src/main/java/com/example/config/SecurityConfig.java
3. 用 Agent 模式，让 AI 自动：
   - 生成 Entity（遵守 Rules）
   - 生成 Repository
   - 生成 Service（含事务管理）
   - 生成 Controller（含权限控制）
   - 生成集成测试
```

Rules 确保代码符合规范，Context 确保 AI 看到正确的文件，Agents 确保多步任务能自动完成。

## 进阶配置：.cursorrules 文件

除了 `.cursor/rules/` 目录，还可以创建 `.cursorrules` 文件（注意是单数），定义全局规则和 AI 行为偏好。

```json
{
  "preferredLanguage": "java",
  "rules": [
    "java-spring-boot.md",
    "testing-standards.md"
  ],
  "context": {
    "include": ["**/src/main/java/**/*.java"],
    "exclude": ["**/test/**", "**/generated/**"]
  }
}
```

这个文件让项目规范更结构化，团队成员可以共享同一套规则。

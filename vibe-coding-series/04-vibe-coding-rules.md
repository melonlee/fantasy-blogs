# Vibe Coding 实战（四)：Vibe Coding 的核心规则——写好 Spec 的艺术

---

![Spec 的艺术](./images/04-vibe-coding-rules.jpg)

Vibe coding 最核心的技能不是写代码，是写 Spec。

很多人觉得"写 prompt"就是"把需求说一遍"。但同样是描述需求，有人能让 AI 生成一次就对的代码，有人让 AI 来来回回改十遍还不对。

差距不在 AI，在人的描述能力。

Spec（规格说明书）是 vibe coding 的核心。你 Spec 写得好，AI 生成代码的质量就高；Spec 写得差，AI 就只能靠猜。

这篇文章专门讲怎么写好 Spec。

## Spec 不是功能列表

最常见的错误是把 Spec 写成功能列表：

```
做一个用户管理功能：
- 用户注册
- 用户登录
- 修改密码
- 重置密码
```

这不是 Spec，这是功能列表。AI 拿到这样的描述，只能"猜"你怎么想——用什么框架、什么数据库、什么认证方式、什么样的代码结构。

Spec 应该是这样的：

```
做一个 Spring Boot 用户管理 REST API。

技术栈：
- Spring Boot 3.2
- Spring Security + JWT
- JPA + H2（开发环境）/ PostgreSQL（生产）
- Maven

API 端点：
- POST /api/users/register：注册
  - 请求体：{ username, email, password }
  - password 最小 8 位，需包含数字和字母
  - username 唯一
  - 成功：201 Created，返回 { id, username, email }
  - 失败：400 Bad Request（参数错误）、409 Conflict（用户名已存在）

- POST /api/users/login：登录
  - 请求体：{ username, password }
  - 成功：200 OK，返回 JWT token
  - 失败：401 Unauthorized

- POST /api/users/forgot-password：忘记密码
  - 请求体：{ email }
  - 发送重置链接到邮箱（这里先返回成功，不实际发邮件）

数据库表：
users(id BIGINT PK, username VARCHAR(50) UNIQUE NOT NULL,
      email VARCHAR(100) UNIQUE NOT NULL,
      password_hash VARCHAR(255) NOT NULL,
      enabled BOOLEAN DEFAULT TRUE,
      created_at TIMESTAMP)

代码规范：
- Controller 统一加 @RestController 和 @RequestMapping("/api/users")
- Service 用构造器注入，加 @Slf4j
- Entity 加 @NoArgsConstructor @AllArgsConstructor
- 所有 endpoint 加 @Operation 文档注解
- 密码用 BCryptPasswordEncoder 加密
```

这就是一个 Spec。不是功能列表，而是包含背景、约束、边界条件的完整描述。

## Spec 的三个层次

好的 Spec 包含三个层次：背景层、约束层、执行层。

**背景层：做什么，为什么。**

告诉 AI 这个功能是为了解决什么问题。不是技术描述，是业务背景。

```
背景：这是一个电商后台的用户模块。注册和登录是后续所有功能的基础。
系统需要支持未来扩展到多租户模式。
```

**约束层：怎么限制。**

约束是 Spec 里最重要的部分。约束越清晰，AI 生成的结果越准确。

约束包括：
- 技术栈（必须用什么框架、什么库）
- 代码规范（命名、结构）
- 边界条件（什么情况怎么处理）
- 安全要求（密码加密、SQL 注入防护）

```
约束：
- 密码必须 BCrypt 加密，存储 hash 不存明文
- username 必须唯一，且只能包含字母数字下划线（^[a-zA-Z0-9_]+$）
- 所有 API 返回统一格式：{ success: boolean, data: ?, error: ? }
- 不允许 SQL 注入，使用 JPA 参数化查询
```

**执行层：具体做什么。**

具体的端点、表结构、方法签名。这是 Spec 里最直接的部分。

执行层需要具体到参数级别：

```
POST /api/products
请求体：
{
  "name": "商品名称",          // 必填，最大 100 字符
  "price": 99.99,             // 必填，正数，最多两位小数
  "categoryId": 1,            // 必填，引用 category.id
  "stock": 100                 // 必填，非负整数
}
```

## 好的 Spec 范例：分步骤生成

对于复杂的任务，不要一次性给完整的 Spec。更好的做法是分步骤。

**第一步：生成骨架。**

```
生成一个 Spring Boot 项目的标准结构，包含：
- Controller、Service、Repository、Entity、DTO 的包结构
- 统一的 Response 包装类
- 全局异常处理器
- application.yml 的基本配置
```

**第二步：生成某个具体模块。**

```
在已有的项目结构上，为 Product 模块生成：
- Product.java（Entity）
- ProductRepository.java（JpaRepository）
- ProductService.java（接口 + 实现）
- ProductController.java（CRUD）
- ProductDTO.java（请求和响应）
```

**第三步：补充测试。**

```
为 ProductService 生成单元测试，覆盖：
- 创建商品（正常情况）
- 创建商品（价格负数，抛出 IllegalArgumentException）
- 创建商品（categoryId 不存在，抛出 EntityNotFoundException）
- 查询商品（存在/不存在）
```

分步骤的好处是：每一步 AI 只需要理解一个目标，不容易混淆。

## 常见 Spec 错误

**错误一：约束太少。**

```
做一个用户服务
```

这样的描述，AI 只能给你最通用的实现。如果你没说用什么认证方式，AI 可能给你 session-based auth；如果你没说密码怎么存，AI 可能给你明文存储。

**错误二：约束太多太碎。**

把 Spec 写成代码规范文档，列出 50 条规则。约束太多，AI 会漏掉，而且很难保持一致。

约束要精选。只列真正重要的、非遵守不可的规则。

**错误三：边界条件模糊。**

```
用户输入要验证
```

什么验证？长度？格式？空值？

```
- username：必填，3-20 字符，只能是字母数字
- email：必填，有效邮箱格式
- password：必填，最少 8 位，至少一个大写字母和一个数字
```

**错误四：前后不一致。**

在同一个 Spec 里，前面说用 MySQL，后面说用 PostgreSQL。AI 会困惑，不知道该听哪个。

Spec 写完后花 30 秒检查一遍一致性，比让 AI 返工要快。

## Spec 的维护

Spec 不是一次性写完就丢掉的。随着项目进展，Spec 需要更新。

一个实用的方法：在项目根目录维护一个 `SPEC.md` 文件，记录当前所有的 API 设计、数据模型、约束条件。

每次让 AI 生成新功能之前，先更新 SPEC.md，保持文档和代码一致。

```markdown
# SPEC.md

## 用户模块
### 已完成
- [x] POST /api/users/register（2024-01-15）
- [x] POST /api/users/login（2024-01-15）

### 待完成
- [ ] POST /api/users/refresh-token

## 产品模块
### 已完成
- [x] POST /api/products（2024-01-16）
- [x] GET /api/products/{id}（2024-01-16）

## 共享约束
- 所有 API 返回格式：{ success: boolean, data: ?, error: ? }
- 密码：BCrypt 加密
- 认证：JWT（access token 1小时，refresh token 7天）
```

## 什么时候需要写完整 Spec

不是每个任务都需要完整的三层 Spec。以下是判断标准：

**需要完整 Spec**：新模块（第一次生成）、涉及数据模型修改、安全相关功能、多个子系统交互。

**不需要完整 Spec**：简单修改（"把这个方法的返回值从 String 改成 UserDto"）、已有模块的小调整、文档生成、代码审查。

经验是：超过 30 分钟开发量的任务，应该写 Spec。30 分钟以内的简单任务，直接说就行。

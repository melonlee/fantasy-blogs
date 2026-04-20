# API 设计：REST 风格与 OpenAPI 规范实战

我接过一个项目，后端 API 全是 SOAP 风格——一个 `doAction` 接口接收 `action` 参数，根据参数值走不同的业务逻辑，返回格式混着用，有时是 XML 有时是 JSON 有时是纯文本。接口文档是一份 Word 文档，放在公司内网的共享目录里，最新一版是三年前更新的，之后没人知道文档和实际接口差了多少。

每次加需求，前端要问后端"这个接口怎么调"，后端说"你等着我看下代码"，然后在代码里找逻辑。这种协作方式效率极低，沟通成本高，出错概率大。

REST API 解决的是这个问题。**统一的接口风格，标准的数据格式，机器可读的结构化文档。** 前端知道怎么调用，后端知道怎么实现，测试知道怎么验证。

这篇文章来聊聊 REST 风格怎么落地、OpenAPI 规范怎么写、版本怎么管理、错误怎么处理、分页怎么做。全是实战经验，不是教科书式的理论。

---

## REST 风格：不是把所有接口叫 REST 就叫 REST

很多人觉得 REST 就是"用 HTTP 方法 + URL 路径"来定义接口。这说法对，但不全对。真正的 REST 有一堆约束（Roy Fielding 的论文里写了六条），但实践中我们不需要教条地遵守所有约束，抓住几个核心原则就够了。

### 资源优先

REST 的核心是"资源"，不是"动作"。URL 表示资源，不表示操作。

```
# RESTful 的做法
GET    /users          # 获取用户列表
POST   /users          # 创建用户
GET    /users/123      # 获取 ID 为 123 的用户
PUT    /users/123      # 全量更新用户
PATCH  /users/123      # 部分更新用户
DELETE /users/123      # 删除用户

# 不那么 RESTful 的做法
GET    /getUsers
POST   /createUser
GET    /getUserById?id=123
POST   /updateUser
POST   /deleteUser
```

用名词复数形式表示资源集合，用路径表示资源的层级关系。`/users/123/orders` 表示 ID 为 123 的用户的订单列表，语义清晰。

![API 设计卡通配图](./images/09-api-design.jpg)

但也有例外。某些操作没有明确的资源对应，比如登录、登出、搜索。登录可以 `POST /sessions`（创建会话资源），搜索可以用 `GET /search?q=xxx` 或者 `GET /products/search`。关键是**让 URL 表达"什么"而不是"怎么做"**。

### HTTP 动词有含义

GET 是只读操作，不修改数据。POST 是创建资源。PUT 是全量替换。PATCH 是部分更新。DELETE 是删除。

这几个动词有明确的语义，客户端和服务器对行为的预期是一致的。如果有人把 DELETE 当成逻辑删除（在代码里 `is_deleted = true`），那就破坏了语义——客户端以为删了，实际上还在。逻辑删除应该用 POST 表示"废弃"资源，或者用 PATCH 更新状态字段。

PUT 和 PATCH 的区别：PUT 要传完整的资源，比如 `PUT /users/123` 需要传用户名、邮箱、密码所有字段，部分更新用 PATCH。实际开发中 PATCH 用得更多，因为大多数时候只改一两个字段。

### 状态码有意义

HTTP 状态码不是随便选的，有它自己的含义：

```
200 OK          - 请求成功，返回数据
201 Created     - 资源创建成功，响应头里带新资源的 URL
204 No Content  - 请求成功，但不返回数据（通常用于 DELETE）
400 Bad Request - 请求参数有问题，客户端需要修正
401 Unauthorized   - 未认证，需要登录
403 Forbidden      - 已认证但没权限
404 Not Found      - 资源不存在
409 Conflict       - 资源冲突（比如用户名已存在）
422 Unprocessable Entity - 请求格式对但业务逻辑不对
500 Internal Server Error - 服务器内部错误
```

状态码选对了，客户端可以根据状态码做不同的处理，不需要解析响应体里的错误信息。状态码乱用会搞乱客户端的判断逻辑。

有个常见的错误：所有失败都返回 200，然后在响应体里写 `{"success": false, "message": "xxx"}`。这样客户端没法用 HTTP 状态码区分错误类型，缓存代理也不知道这是成功还是失败。

---

## OpenAPI 规范：让文档和代码保持一致

OpenAPI（以前叫 Swagger）是一套 API 描述规范，用 YAML 或 JSON 写，机器可读，工具链丰富。写好 OpenAPI 文档，前端可以自动生成客户端代码，测试可以自动生成测试用例，文档网站可以自动渲染成好看的页面。

我见过两种极端：一种是不写 OpenAPI，靠嘴沟通；另一种是写了 OpenAPI 但和代码完全不匹配，文档上天花乱坠，实际接口对不上。第一种是偷懒，第二种是自欺欺人。

**好的做法是代码即文档。** 注解写在代码里，文档从代码生成。

Spring Boot 项目用 `springdoc-openapi`：

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

加上依赖后，访问 `/swagger-ui.html` 就能看到文档页面。

接下来看一个完整的 Controller 怎么写 OpenAPI 注解：

```java
@Tag(name = "用户管理", description = "用户相关的增删改查操作")
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {
    
    private final UserService userService;
    
    @Operation(
        summary = "创建用户",
        description = "输入用户名、邮箱、密码，创建新用户。用户名和邮箱不能与已有用户重复。"
    )
    @ApiResponse(responseCode = "201", description = "创建成功")
    @ApiResponse(responseCode = "400", description = "参数校验失败")
    @ApiResponse(responseCode = "409", description = "用户名或邮箱已存在")
    @PostMapping
    public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest request) {
        UserResponse response = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
    
    @Operation(summary = "获取用户详情", description = "根据用户 ID 获取用户信息")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "成功"),
        @ApiResponse(responseCode = "404", description = "用户不存在")
    })
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        UserResponse response = userService.getUser(id);
        return ResponseEntity.ok(response);
    }
    
    @Operation(summary = "查询用户列表", description = "支持分页和条件过滤")
    @GetMapping
    public ResponseEntity<Page<UserResponse>> listUsers(
            @Parameter(description = "页码，从 0 开始")
            @RequestParam(defaultValue = "0") int page,
            
            @Parameter(description = "每页数量，最大 100")
            @RequestParam(defaultValue = "20") int size,
            
            @Parameter(description = "用户名关键字过滤")
            @RequestParam(required = false) String username) {
        
        Pageable pageable = PageRequest.of(page, Math.min(size, 100));
        Page<UserResponse> result = userService.listUsers(pageable, username);
        return ResponseEntity.ok(result);
    }
    
    @Operation(summary = "更新用户", description = "全量更新用户信息，所有字段都会被覆盖")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "更新成功"),
        @ApiResponse(responseCode = "404", description = "用户不存在"),
        @ApiResponse(responseCode = "400", description = "参数校验失败")
    })
    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        UserResponse response = userService.updateUser(id, request);
        return ResponseEntity.ok(response);
    }
    
    @Operation(summary = "删除用户", description = "物理删除用户，数据无法恢复")
    @ApiResponse(responseCode = "204", description = "删除成功")
    @ApiResponse(responseCode = "404", description = "用户不存在")
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().get();
    }
}
```

对应的 Request 和 Response DTO：

```java
@Getter
@Setter
@NoArgsConstructor
public class CreateUserRequest {
    
    @NotBlank(message = "用户名不能为空")
    @Size(min = 3, max = 50, message = "用户名长度在 3-50 个字符之间")
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "用户名只能包含字母、数字和下划线")
    private String username;
    
    @NotBlank(message = "邮箱不能为空")
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @NotBlank(message = "密码不能为空")
    @Size(min = 8, max = 128, message = "密码长度在 8-128 个字符之间")
    private String password;
}

@Getter
@Setter
@NoArgsConstructor
public class UpdateUserRequest {
    
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @Size(min = 8, max = 128, message = "密码长度在 8-128 个字符之间")
    private String password;
    
    private String nickname;
    
    private String avatarUrl;
}

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class UserResponse {
    private Long id;
    private String username;
    private String email;
    private String nickname;
    private String avatarUrl;
    private UserStatus status;
    private LocalDateTime createdAt;
}
```

OpenAPI 注解写在哪里？写在 Controller 的方法上，写在 Request/Response 的字段上。注释要写清楚业务规则，比如"用户名不能重复"、比如"删除后数据无法恢复"。前端读文档的时候需要知道这些信息。

还有一个重要的注解是 `@ApiResponse` 的 `responseCode`。状态码要写准确的数字，不要写字符串 `"200"`，工具链会把它当成字符串处理而不是数字。

---

## 错误响应规范：让客户端知道出了什么问题

接口出错了，返回什么格式？很多项目的做法是返回一个字符串，或者返回一个格式不固定的 JSON 对象。客户端不知道怎么处理，只能弹个通用的"系统错误"。

### 统一的错误响应格式

无论成功还是失败，接口的响应格式应该是固定的。成功返回业务数据，失败返回错误信息：

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ApiResponse<T> {
    
    private boolean success;
    private String code;
    private String message;
    private T data;
    private LocalDateTime timestamp;
    private String traceId;
    
    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
                .success(true)
                .code("OK")
                .message("请求成功")
                .data(data)
                .timestamp(LocalDateTime.now())
                .traceId(getTraceId())
                .build();
    }
    
    public static <T> ApiResponse<T> error(String code, String message) {
        return ApiResponse.<T>builder()
                .success(false)
                .code(code)
                .message(message)
                .timestamp(LocalDateTime.now())
                .traceId(getTraceId())
                .build();
    }
    
    private static String getTraceId() {
        // 从上下文获取或生成 traceId，用于日志追踪
        return MDC.get("traceId");
    }
}
```

`traceId` 是链路追踪用的。请求进来的时候生成一个 traceId，后续所有日志都带这个 traceId，出了问题可以顺着 traceId 把整个请求链路串起来。这个字段对客户端没用，但排查问题的时候很关键。

### 错误码要有分类

错误码要有分类，能让客户端区分错误类型：

```java
public interface ErrorCodes {
    // 客户端参数错误（4xx）
    String VALIDATION_ERROR = "VALIDATION_ERROR";
    String MISSING_PARAMETER = "MISSING_PARAMETER";
    String INVALID_PARAMETER = "INVALID_PARAMETER";
    
    // 认证授权错误
    String UNAUTHORIZED = "UNAUTHORIZED";
    String FORBIDDEN = "FORBIDDEN";
    String TOKEN_EXPIRED = "TOKEN_EXPIRED";
    
    // 业务逻辑错误
    String RESOURCE_NOT_FOUND = "RESOURCE_NOT_FOUND";
    String RESOURCE_CONFLICT = "RESOURCE_CONFLICT";
    String BUSINESS_ERROR = "BUSINESS_ERROR";
    
    // 服务器错误（5xx）
    String INTERNAL_ERROR = "INTERNAL_ERROR";
    String SERVICE_UNAVAILABLE = "SERVICE_UNAVAILABLE";
}
```

客户端拿到错误码之后，可以决定是弹窗提示用户"参数有误请检查"，还是"登录过期请重新登录"，还是"系统错误请联系客服"。如果只有一个通用的"ERROR"，客户端没法区分处理。

### 全局异常处理

全局异常处理统一拦截，返回格式化错误：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining("; "));
        return ResponseEntity.badRequest()
                .body(ApiResponse.error(ErrorCodes.VALIDATION_ERROR, message));
    }
    
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleNotFound(EntityNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(ApiResponse.error(ErrorCodes.RESOURCE_NOT_FOUND, e.getMessage()));
    }
    
    @ExceptionHandler(ResourceConflictException.class)
    public ResponseEntity<ApiResponse<Void>> handleConflict(ResourceConflictException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(ApiResponse.error(ErrorCodes.RESOURCE_CONFLICT, e.getMessage()));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleGeneral(Exception e) {
        // 记录日志，但不对外暴露内部细节
        log.error("Unexpected error", e);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResponse.error(ErrorCodes.INTERNAL_ERROR, "系统内部错误，请稍后重试"));
    }
}
```

有个原则：**对内暴露细节，对外隐藏细节**。日志里记完整的堆栈信息，方便排查；返回给客户端的只是错误码和通用提示，不暴露内部实现细节。攻击者如果看到"SQLSyntaxErrorException: Table 'xxx' doesn't exist"，会知道你的数据库里有哪些表。

---

## API 版本管理：向前兼容是艺术

API 迟早要变，加字段、改字段、删字段都会发生。**版本管理的核心目标是让老客户端继续工作，同时支持新功能。**

### URL 路径版本 vs Header 版本

**URL 路径版本**把版本号放 URL 路径里：

```
/api/v1/users
/api/v2/users
```

优点是版本一目了然，负载均衡的时候能准确路由到对应版本。缺点是同一个资源的不同版本要维护多套代码，升级的时候要迁移数据。

**Header 版本**用 Accept 头：

```
Accept: application/vnd.myapp.v2+json
```

这种方式不污染 URL，但不够直观，调试的时候不方便。用的项目相对少一些。

### 推荐做法：URL 路径版本

大多数场景下，URL 路径版本是最实际的方案。

```java
@RestController
@RequestMapping("/api/v1")
public class UserControllerV1 { /* ... */ }

@RestController
@RequestMapping("/api/v2")
public class UserControllerV2 { /* ... */ }
```

老版本保持一段时间（通常 6-12 个月），在新版本文档里标注废弃时间，给客户端留出迁移窗口。到了废弃时间，老版本下线，已迁移的客户端继续工作，没迁移的自己负责。

### 兼容变更 vs 不兼容变更

什么是**兼容变更**？加新字段、加新接口。老客户端不认识新字段，忽略它继续工作；老客户端不调新接口，没影响。

什么是**不兼容变更**？删字段、改字段类型、改字段含义。客户端依赖了这些字段，一旦改了就会出问题。

尽量做兼容变更。如果一定要做不兼容变更，走版本升级流程，不要在原有版本上直接改。

有个常见的错误：为了"节省资源"在老版本上直接改字段类型，结果线上几十个客户端全部报错，一晚上都在紧急回滚。宁可多花时间做版本迁移，也不要在生产环境直接改不兼容的字段。

---

## 分页设计：数据多了怎么传

列表接口返回数据少的时候没问题，数据多了必须分页。

### Cursor 分页 vs Offset 分页

**Offset 分页**是最常见的做法，页码加每页数量：

```
GET /users?page=0&size=20
```

实现简单，跳页方便，用户可以随便跳第几页。但数据有变更的时候容易出现重复或遗漏——比如翻到第二页的时候，第一页的数据被删了，第二页的第一条就变成原来的第三条，用户会看到重复数据。

**Cursor 分页**（也叫 Keyset 分页）用上一页的最后一条数据的 ID 当游标：

```
GET /users?cursor=abc123&size=20
```

这种方式翻页的时候数据不会乱，因为游标指向的是具体某一条记录，数据库里按主键排序往后取，不会受中间数据变更影响。但不能跳页，只能"下一页"。

```java
@Getter
@Setter
@NoArgsConstructor
public class CursorPageRequest {
    private String cursor;      // 上一页最后一条的 ID
    private Integer limit;      // 每页数量
    
    public static CursorPageRequest of(String cursor, Integer limit) {
        CursorPageRequest request = new CursorPageRequest();
        request.setCursor(cursor);
        request.setLimit(limit != null ? limit : 20);
        return request;
    }
}

@Getter
@Setter
@NoArgsConstructor
public class CursorPageResponse<T> {
    private List<T> data;
    private String nextCursor;
    private boolean hasMore;
}
```

Cursor 分页的实现需要按某个唯一字段排序（通常是主键或时间戳），查询的时候用 `WHERE id > :cursor` 而不是 `LIMIT/OFFSET`。

### 按场景选

高并发的列表接口推荐 Cursor 分页，数据一致性更好。管理后台这种需要任意跳页的场景，用 Offset 分页。

还有一种做法是**折中方案**：用 Offset 查，但返回的时候带上"下一页的游标"，客户端可以选择用游标翻页也可以继续用页码翻页。这种方案实现复杂度稍高，但兼顾了两种场景。

---

## 总结

API 设计是后端对外的门面，设计得好，前端和测试省心，设计得烂，所有人都是受害者。

REST 风格的核心是"资源+动词"，URL 表示资源，HTTP 方法表示操作，状态码表示结果。语义清晰，客户端和服务器对行为的预期一致。

OpenAPI 文档要跟着代码走，用注解写进代码里，用工具生成文档页面。注释要写业务规则，不只是技术说明。

错误响应格式要统一，错误码要分类，异常处理收口到全局处理器里。日志里暴露细节，对外隐藏细节。

版本管理用 URL 路径版本，区分兼容变更和不兼容变更，尽量做兼容变更，不兼容就走版本升级。

分页按场景选，任意跳页用 Offset，高并发列表用 Cursor。

接口设计不是孤立的，它影响前端实现、测试策略、API Gateway 路由、甚至 App 的版本兼容。花时间想清楚，值得。

---

**后记**：Vibe Coding 系列写到第 9 篇，后端的内容差不多覆盖完了。下一阶段聊聊前端、Vite、以及怎么把前后端串起来跑一个完整的项目。

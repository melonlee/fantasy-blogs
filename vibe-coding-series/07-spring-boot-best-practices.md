# Spring Boot 最佳实践：从入门到精通的避坑指南

我第一次用 Spring Boot 的时候，觉得这玩意儿也太友好了吧。搭个 Web 服务，几行代码就跑起来了，Run 一下，浏览器打开，`Hello World` 就出来了。那会儿觉得 Spring Boot 真是拯救世界。

然后我开始写正经项目。

三个月后，我的代码库变成了一座屎山。Service 层堆了 800 行，Config 文件里全是 `@Value` 注解，一个新需求过来我得改七八个文件，还动不动就报 `NullPointerException`。组里 Code Review 的时候，前辈看了一眼我的 Service，叹了口气，说："你这个设计，换我来我也看不懂。"

那是我第一次意识到，Spring Boot 入门容易，写好难。

这篇文章不教你 Hello World，也不讲那些官方文档里已经写烂了的东西。我来聊聊真正拉开差距的东西：**项目结构怎么搭、依赖注入怎么做、配置怎么管、分层怎么分、事务怎么处理、异常怎么收口**。这些都是我在真实项目里踩坑踩出来的经验。

---

## 项目结构：不只是放文件的目录

很多人把 Spring Boot 项目结构当成理所当然的东西。`src/main/java` 下面堆着一堆 Package，Controller、Service、Repository 三个文件夹走天下。功能少的时候没问题，功能多了就开始乱了。

我见过最离谱的一个项目，所有 Java 文件都放在 `com.example.demo` 一个包下面，零零总总两百多个类，没有分层，没有模块化，读代码得靠 IDE 的 Search 功能。这种结构不是不能用，但维护成本极高——每次定位一个 Bug，得先把整个项目过一遍。

**那合理的结构应该是什么样的？**

我现在的做法是按业务模块分，而不是按技术层次分。看个例子：

```
src/main/java/com/example/myapp/
├── myapp/                          # 根包
│   ├── MyAppApplication.java       # 启动类
│   ├── common/                     # 公共组件
│   │   ├── config/                 # 通用配置
│   │   ├── exception/              # 异常定义
│   │   └── util/                   # 工具类
│   ├── module-user/                # 用户模块
│   │   ├── controller/
│   │   ├── service/
│   │   ├── repository/
│   │   ├── domain/                 # 实体和 DTO
│   │   └── UserModuleConfig.java   # 模块级配置
│   └── module-order/               # 订单模块
│       ├── controller/
│       ├── service/
│       ├── repository/
│       └── domain/
```

这样做的好处是**模块边界清晰**。每个业务模块自包含，测试的时候可以单独跑，改需求的时候影响范围可控。如果以后要做微服务拆分，按这个结构拆起来也方便。

![Spring Boot + Coffee + Code 配图](./images/07-spring-boot.jpg)

**也有简单一些的做法，适合中小项目**：

```
src/main/java/com/example/demo/
├── DemoApplication.java
├── controller/                     # 所有 Controller
├── service/                        # 所有 Service
│   └── impl/                       # Service 实现类
├── repository/                     # 所有 Repository
├── entity/                         # 所有 Entity
├── dto/                            # 所有 DTO
├── config/                         # 配置类
└── exception/                      # 异常类
```

这种结构的问题是：所有 Controller 在一个包，所有 Service 在一个包，当项目有 50 个类的时候还好，有 200 个类的时候，找东西就很痛苦。但对于三五个人开发的小项目，这种结构完全够用，没必要过度设计。

**具体用哪种结构，取决于项目规模**。小项目（一个模块、五六个人开发）用标准三层结构完全够用；大项目（多团队、长期维护）按业务模块分更合理。**没有最好的结构，只有适合的结构。**

还有一个容易忽略的点：**启动类的位置**。Spring Boot 默认只扫描启动类所在包及其子包下的组件。如果启动类放在根包下面，所有子包都能扫到；如果启动类放在 `controller` 包下面，`service` 包里的组件就扫不到了。常见的错误是把启动类放在错误的位置，然后各种 `@Component` 注解不生效，查半天找不到原因。

---

## 依赖注入：@Autowired 的陷阱与构造器注入的正确姿势

Spring 的依赖注入是个好东西，但用不好就是灾难。

我刚入门的时候，代码里全是这种写法：

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private EmailService emailService;
    
    // 业务方法...
}
```

这种写法能用，跑也没问题。但它有几个隐藏的问题：

**第一，可测试性差。** 你想单独测试 `UserService`，得想办法 mock 这些依赖。用 `@Autowired` 注入的字段，默认不能直接注入 mock 对象，得用反射或者 Spring Test 来搞，麻烦。

**第二，依赖关系不明确。** 一个类依赖了哪些东西，看代码看不出来——得看字段定义。如果字段很多，你根本不知道这个 Service 的核心依赖是什么，辅助依赖是什么。

**第三，循环依赖。** 如果 A 依赖 B，B 依赖 C，C 又依赖 A，Spring 启动的时候会爆循环依赖错误，而且这种错误定位起来挺费劲的。

**第四，字段可能为空。** 如果用 `@Autowired` 注入，而 Spring 容器里没有对应类型的 Bean，启动的时候不会报错，直到第一次调用那个字段才会爆 `NullPointerException`。这种延迟失败比启动时就失败更难排查。

现在的最佳实践是**构造器注入**。把依赖通过构造函数传入，代码变成这样：

```java
@Service
public class UserService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final EmailService emailService;
    
    public UserService(
            UserRepository userRepository,
            PasswordEncoder passwordEncoder,
            EmailService emailService) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
        this.emailService = emailService;
    }
    
    public void register(String username, String email, String password) {
        // 业务逻辑
    }
}
```

这样写的好处是什么？

**依赖一目了然。** 看构造函数就知道这个 Service 需要什么，没有隐藏依赖。Code Review 的时候，别人扫一眼构造函数就能理解这个类的职责。

**天然可测试。** `new UserService(mockRepo, mockEncoder, mockEmail)` 直接 new 出来，测试随便写，不用启动 Spring 容器。

**防止意外修改。** `final` 字段初始化后不能改，意味着依赖一旦注入就不会变动。如果有人在运行期偷偷把 `userRepository` 换成另一个实例，你的代码不会莫名其妙出问题。

**启动时就验证。** 如果某个依赖在 Spring 容器里找不到，构造函数调用的时候就爆错了，不会等到业务方法被调用才出问题。

如果你的 Service 只依赖一两个东西，构造函数还能再简化。用 Lombok 的 `@RequiredArgsConstructor`：

```java
@Service
@RequiredArgsConstructor
public class UserService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final EmailService emailService;
    
    // 业务方法...
}
```

编译的时候 Lombok 自动生成构造函数，代码干净多了。

**什么时候不能用构造器注入？** 如果你的 Service 有循环依赖，构造器注入确实会出问题。但这不是构造器注入的问题，是你架构设计有问题——两个 Service 互相直接依赖，说明它们耦合太紧了，得重新考虑设计。可以通过事件机制、接口抽象、或者重新划分职责来解决循环依赖问题。

---

## 配置管理：@ConfigurationProperties 与 @Value 的取舍

Spring Boot 读取配置有两种常见方式：`@Value` 和 `@ConfigurationProperties`。很多人分不清什么时候用哪个，结果配置散落一地，维护起来头疼。

**@Value 适合什么场景？**

`@Value` 适合读取单个值，特别是那些零散的、不属于同一个业务概念的配置。比如：

```java
@Service
public class ImageService {
    
    @Value("${upload.max-file-size:10MB}")
    private String maxFileSize;
    
    @Value("${upload.allowed-types:jpg,png}")
    private String allowedTypes;
    
    @Value("${upload.storage-path:/data/uploads}")
    private String storagePath;
    
    public void upload(MultipartFile file) {
        // 使用配置...
    }
}
```

这种零散配置用 `@Value` 很直观，一个注解对应一个值，一眼就知道从哪里读。

但如果配置项很多，或者配置项之间有逻辑关联，`@Value` 就不好使了。你得写一堆 `@Value` 注解，代码看着乱，而且配置项之间的关系不清晰。

**@ConfigurationProperties 适合什么场景？**

当你的配置有明确的业务语义，或者说配置项属于同一个"对象"的时候，用 `@ConfigurationProperties`。比如支付相关的配置：

```java
@Component
@ConfigurationProperties(prefix = "payment")
public class PaymentProperties {
    
    private String merchantId;
    private String merchantKey;
    private String notifyUrl;
    private int timeoutSeconds = 30;
    private List<String> supportedChannels = new ArrayList<>();
    
    // getter 和 setter（或者用 Lombok @Data）
}
```

这样配置就变成了一个结构化的对象，所有支付相关的配置集中在一起。YAML 里这样写：

```yaml
payment:
  merchant-id: "your-merchant-id"
  merchant-key: "your-merchant-key"
  notify-url: "https://yourapp.com/payment/notify"
  timeout-seconds: 30
  supported-channels:
    - alipay
    - wechat
    - bankcard
```

对应的配置属性用短横线命名（`merchant-id`），Spring Boot 自动映射到驼峰命名（`merchantId`），不用手动转换。

**为什么要用 @ConfigurationProperties 而不是 @Value？**

一个重要原因：**类型安全**。用 `@Value` 读配置，读错了它给你个 null，不会报错。用 `@ConfigurationProperties`，你可以定义字段类型，配置不对直接启动就报错，不会在业务逻辑运行到一半的时候突然发现配置读错了。

另一个原因：**IDE 支持好**。写配置的时候，IDE 能给你自动补全。YAML 里写 `payment.`，下拉列表直接出来所有可配置项，省得你背配置名。

还有第三个原因：**配置可以校验**。在 `@ConfigurationProperties` 类上加 `@Validated`，可以对配置值做校验：

```java
@Component
@ConfigurationProperties(prefix = "payment")
@Validated
public class PaymentProperties {
    
    @NotBlank
    private String merchantId;
    
    @Min(1)
    @Max(300)
    private int timeoutSeconds = 30;
}
```

如果 YAML 里的配置不符合校验规则，应用启动的时候就报错，而不是跑着跑着才发现数据不对。

不过 `@ConfigurationProperties` 有个坑：默认不会注册成 Bean，得加 `@EnableConfigurationProperties` 或者 `@ConfigurationPropertiesScan`。Spring Boot 3.x 之后可以直接用 `@ConfigurationProperties` 加上 `@Component`，但最好确认一下你的版本。

---

## 分层架构：Service 层堆代码是最常见的烂代码模式

我见过最夸张的一个 Service，1800 行，一个类干了一个模块所有的活。查用户、发短信、发邮件、生成报表、调用第三方 API、全塞在一个 Service 里。注释写着"用户服务"，实际上是"用户相关所有事情服务"。

这种写法是怎么来的？通常是项目初期图方便，觉得"反正功能都能跑"，就一直往上加。加着加着，就变成了没人敢动的屎山。

**分层的目的不是分文件，是分职责。** 每层只干本层该干的事，跨层的逻辑通过接口或者事件解耦。

我的实践是这样的：

**Controller 层**：接收请求，参数校验，调用 Service，返回响应。这一层不应该有业务逻辑，它只是一个入口。参数校验可以用 `@Valid` 和 `@Validated` 做基础的校验，复杂校验留给 Service。

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {
    
    private final UserService userService;
    
    @PostMapping
    public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest request) {
        UserResponse response = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        UserResponse response = userService.getUser(id);
        return ResponseEntity.ok(response);
    }
}
```

**Service 层**：业务逻辑处理。这一层干所有跟业务相关的事：一个业务流程怎么走、数据怎么转换、调用哪些外部服务、事务怎么控制。但不是所有业务逻辑都往这里塞——如果逻辑太复杂，Service 也会变得难以维护。

```java
@Service
@RequiredArgsConstructor
public class UserService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final EmailService emailService;
    private final UserEventPublisher eventPublisher;
    
    @Transactional
    public UserResponse createUser(CreateUserRequest request) {
        // 检查用户名是否存在
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new UserAlreadyExistsException(request.getUsername());
        }
        
        // 加密密码
        String encodedPassword = passwordEncoder.encode(request.getPassword());
        
        // 构建用户实体
        User user = new User();
        user.setUsername(request.getUsername());
        user.setPassword(encodedPassword);
        user.setEmail(request.getEmail());
        user.setCreatedAt(LocalDateTime.now());
        
        // 保存
        User savedUser = userRepository.save(user);
        
        // 发送欢迎邮件（异步，不阻塞主流程）
        emailService.sendWelcomeEmailAsync(savedUser.getEmail());
        
        // 发布领域事件
        eventPublisher.publish(new UserCreatedEvent(savedUser));
        
        // 转换返回
        return toResponse(savedUser);
    }
}
```

**Repository 层**：数据访问。这一层只关心怎么从数据库读写数据，不应该有业务逻辑。接口用 Spring Data JPA 的 `JpaRepository`，复杂查询用 `@Query` 或者 Specification。

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    Optional<User> findByUsername(String username);
    
    boolean existsByUsername(String username);
    
    boolean existsByEmail(String email);
    
    @Query("SELECT u FROM User u WHERE u.status = :status AND u.createdAt > :since")
    List<User> findActiveUsersSince(@Param("status") UserStatus status, @Param("since") LocalDateTime since);
}
```

**Domain 层**：实体定义和值对象。这里放业务模型，它反映的是业务概念，而不是数据库表结构。比如"用户"是一个实体，但"用户名"可以是一个值对象。

```java
@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String username;
    
    @Column(nullable = false)
    private String password;
    
    @Column(unique = true)
    private String email;
    
    @Enumerated(EnumType.STRING)
    private UserStatus status;
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders;
}
```

---

## 事务管理：别让数据库等你

Spring 的 `@Transactional` 用起来很简单，在方法上加个注解，方法里的数据库操作就自动包在事务里了。但简单不代表可以乱用。

**第一个坑：事务边界太大。**

有人喜欢把整个 Service 方法标 `@Transactional`，不管里面有多少操作。一个方法里查用户、查订单、调第三方 API、发消息、加起来能跑好几秒，数据库连接一直占着不动。这在高并发场景下会耗尽数据库连接池。

事务应该尽可能短。只把需要原子性的操作放进事务，其他操作放外面。查数据、远程调用、消息发送这些操作，和数据库事务没关系，不应该包在事务里。

```java
// 不好：事务太大
@Transactional
public void createOrder(Long userId, List<Long> itemIds) {
    User user = userRepository.findById(userId);           // 读
    Order order = buildOrder(user, itemIds);              // 本地逻辑
    orderRepository.save(order);                          // 写
    paymentService.processPayment(user, order);           // 远程调用，耗时长
    messageService.sendOrderNotification(user, order);   // 消息发送，耗时长
}

// 好：只把需要原子的操作放进事务
@Transactional
public void createOrder(Long userId, List<Long> itemIds) {
    User user = userRepository.findById(userId);
    Order order = buildOrder(user, itemIds);
    orderRepository.save(order);
}

public void afterOrderCreated(Order order) {
    // 这些操作不需要在事务里
    paymentService.processPayment(order.getUser(), order);
    messageService.sendOrderNotification(order.getUser(), order);
}
```

**第二个坑：自调用不生效。**

如果在一个 Service 内部，方法 A 调用同一个类的另一个方法 B，而 B 有 `@Transactional`，B 的事务不生效。因为 Spring 的事务是基于代理的，同一个类内部调用不走代理，所以事务注解会被忽略。

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final OrderRepository orderRepository;
    
    public void process(Long orderId) {
        // 调用同一个类的方法，事务不生效
        this.doProcess(orderId);
    }
    
    @Transactional
    public void doProcess(Long orderId) {
        // 这个事务注解不会生效，因为是内部调用
        orderRepository.findById(orderId).ifPresent(Order::markProcessed);
    }
}
```

解决办法是把需要事务的方法放到另一个 Service 里，或者用 `ApplicationContext` 手动获取代理对象。通常推荐前者——拆分到独立 Service 更符合分层原则。

---

## 异常处理：别让业务代码充满 try-catch

这个问题我见过太多次了：

```java
public User createUser(CreateUserRequest request) {
    try {
        userRepository.save(user);
    } catch (DataIntegrityViolationException e) {
        throw new RuntimeException("用户名已存在");
    } catch (Exception e) {
        throw new RuntimeException("创建用户失败");
    }
}
```

业务代码里全是 try-catch，每个 Service 方法都这样写一遍，代码看着乱，而且异常处理逻辑散得到处都是，改都不好改。

Spring Boot 的全局异常处理是更好的方案。用 `@ControllerAdvice` 统一捕获异常，返回格式化的错误响应：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserAlreadyExistsException.class)
    public ResponseEntity<ErrorResponse> handleUserAlreadyExists(UserAlreadyExistsException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(new ErrorResponse("USER_EXISTS", e.getMessage()));
    }
    
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleEntityNotFound(EntityNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("NOT_FOUND", e.getMessage()));
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest()
                .body(new ErrorResponse("VALIDATION_ERROR", message));
    }
}
```

业务代码只需要在真正需要的地方抛异常，不用 try-catch 包着：

```java
public User createUser(CreateUserRequest request) {
    if (userRepository.existsByUsername(request.getUsername())) {
        throw new UserAlreadyExistsException(request.getUsername());
    }
    return userRepository.save(user);
}
```

这样代码干净多了，异常处理逻辑集中在一个地方，改起来也方便。

**还有一个要注意的点**：异常信息对内对外要有区分。对内（日志）记录完整的堆栈信息，方便排查问题；对外（API 响应）只返回错误码和通用提示，不暴露内部实现细节。

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleGeneral(HttpServletRequest request, Exception e) {
    // 记录完整日志
    log.error("Request {} failed: {}", request.getRequestURI(), e.getMessage(), e);
    
    // 对外不暴露内部细节
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.error("INTERNAL_ERROR", "系统内部错误，请稍后重试"));
}
```

---

## 总结

Spring Boot 入门容易，写好难。入门的时候你会觉得框架在帮你，写着写着就发现，框架只是工具，代码质量还得靠自己。

这篇文章聊了几个我踩坑踩出来的经验：**项目结构按业务分比按技术分更利于维护**；**构造器注入比字段注入更清晰、更可测试**；**零散配置用 @Value，结构化配置用 @ConfigurationProperties**；**Service 层别堆太多东西，分层要分职责**；**事务边界要控制，别让数据库空等**；**异常处理收口到 ControllerAdvice 里，别让业务代码全是 try-catch**。

框架给你提供了能力，怎么用是你的事。用得好是省力，用不好是挖坑。

---

**下期预告**：数据库 Schema 怎么设计才能不后悔？JPA Entity 怎么映射才能少走弯路？下篇文章聊聊数据库设计的那些坑。

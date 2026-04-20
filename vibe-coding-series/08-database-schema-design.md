# 数据库设计：Schema 驱动开发的实战指南

上周我接手了一个老项目，数据库里有张 `user_info` 表，字段密密麻麻的，`name`、`user_name`、`userName`、`username` 混着用，索引加了一堆但没人知道为什么加，有几个字段历史原因加了 `varchar(4000)` 存手机号，还有一个字段叫 `is_deleted` 但代码里有时候用这个有时候不用。

我花了两天时间才把业务逻辑理清楚。

数据库设计烂，后面的开发全是坑。改字段怕影响现有逻辑，不改又别扭，最后只能带着历史包袱继续堆代码。数据库 Schema 烂掉之后，所有人都在付出代价——开发效率低、bug 多、上线慢，每次接需求都要先跟历史设计搏斗。

这篇文章来聊聊怎么做数据库设计才能少后悔。内容包括 JPA Entity 怎么映射、Schema 迁移工具怎么用、索引怎么加、以及什么时候该反范式。都是我在真实项目里踩过的坑，不讲理论，只讲实战。

---

## JPA Entity 设计：从命名到映射

用 JPA 的时候，很多人把 Entity 当成数据库表的直接映射，字段名对应列名，类型对应 SQL 类型。这么用没问题，但细节处理不好，后面就是坑。

### NamingStrategy：命名的坑

JPA 默认的命名策略会把驼峰命名的 Java 字段转成下划线命名的数据库列。`userName` 变成 `user_name`，`orderId` 变成 `order_id`。这个行为在大多数场景下是对的，但不是所有。

Spring Boot 2.x 用的默认策略是 `SpringPhysicalNamingStrategy`，会把点分隔的包名也转成下划线。`com.example.demo.entity.User` 的 `userName` 字段，默认会变成 `user_name` 列。这是符合预期的。

但问题出在有些人用了自定义的 NamingStrategy，结果同一个字段在不同环境下命名不一致——本地开发没问题，上了测试环境字段名对不上，报错都找不到原因。遇到过最离谱的情况是，测试环境数据库列名是大写的，上线之后 Hibernate 找不到字段，Spring Boot 直接起不来。

建议统一指定 NamingStrategy，在 application.yml 里配置：

```yaml
spring:
  jpa:
    hibernate:
      naming:
        physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
        implicit-strategy: org.hibernate.boot.model.naming.ImplicitNamingStrategyJpaCompliantImpl
```

这样 Java 里的命名直接对应数据库列名，不做额外转换。如果你们团队习惯了下划线命名，改用 `CamelCaseToUnderscoresNamingStrategy`。

另一个容易出问题的点是**保留字冲突**。MySQL 里有很多保留字，比如 `order`、`user`、`group`，如果字段名正好是这些，直接写会报错。很多框架会自动给表名和列名加反引号，但有时候加漏了，或者某些迁移脚本里没加，就出问题了。所以字段名尽量避开保留字，`user` 改成 `username` 或者 `user_name`，`order` 改成 `order_no` 或者 `biz_order`。

### 列映射：字段类型与约束

数据库字段有类型、有约束，Entity 里的注解要把这些信息准确表达出来。

```java
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_user_id", columnList = "user_id"),
    @Index(name = "idx_status_created", columnList = "status, created_at")
})
@Getter
@Setter
@NoArgsConstructor
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "user_id", nullable = false)
    private Long userId;
    
    @Column(name = "order_no", unique = true, nullable = false, length = 32)
    private String orderNo;
    
    @Column(name = "total_amount", nullable = false, precision = 12, scale = 2)
    private BigDecimal totalAmount;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private OrderStatus status;
    
    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @Column(name = "is_deleted", nullable = false)
    private Boolean isDeleted = false;
    
    @Version
    @Column(name = "version")
    private Long version;
}
```

这个 Entity 定义里值得注意的点：

`precision = 12, scale = 2` 明确了金额字段的精度，数据库里是 `decimal(12,2)`，存到分位的价格不会丢失精度也不会浪费空间。曾经见过有人用 `double` 存金额，算出来的总价差几分钱，对不上账，查起来头疼死。

`@Enumerated(EnumType.STRING)` 用字符串存储枚举值而不是序号。如果用 `ORDINAL`（默认），枚举值的顺序一变，数据库里存的数据就对不上了。比如 `Status { PENDING, PAID, CANCELLED }` 存的是 0、1、2，有天业务需要加个 `REFUNDING` 插到中间，ORDINAL 全乱了，数据库里存的 1 原来是 PAID 变成了 REFUNDING。用字符串存储，枚举改顺序不影响已有数据。

![数据库 Schema 卡通配图](./images/08-database-schema.jpg)

`is_deleted` 字段用布尔类型而不是 `TINYINT(1)`，代码里读出来是 `Boolean`，不用手动转换。但要注意，查询的时候要用 `isDeleted = false` 而不是 `isDeleted != true`，否则三态布尔（true/false/null）会出问题。如果数据库里存的是 NULL，`!= true` 会返回 true，这通常不是想要的结果。

`@Version` 是乐观锁字段。并发更新同一行数据时，Hibernate 自动在 WHERE 条件里加上 `version = ?`，如果版本号对不上就抛 `OptimisticLockException`。这个字段不用手动更新，Hibernate 帮你管。好处是不用数据库锁，性能好；坏处是并发冲突的时候直接抛异常，需要业务代码处理重试逻辑。

### 关系映射：一对多和多对多

关系映射是最容易出问题的地方。我见过有人把一对多映射成两张表各存各的 ID，然后在代码里手动关联查询。这种做法失去了 JPA 的意义——既然用了 ORM，就要让框架帮你管关系，而不是自己写 SQL JOIN。

```java
@Entity
@Table(name = "users")
@Getter
@Setter
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, length = 50)
    private String username;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();
    
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
}
```

`mappedBy = "user"` 说明"谁是我的拥有方"。在这个例子里，`Order` 有一个 `user` 字段指向 `User`，所以 `User` 里用 `mappedBy` 表示"我不拥有这个关系，是 Order 拥有的"。`mappedBy` 那边的字段名是 `user`，对应 `Order.user` 字段。如果搞反了， Hibernate 会报奇怪的异常。

`cascade = CascadeType.ALL` 意思是"操作 User 的时候，同步操作它的 Orders"。删除 User，顺便把关联的 Order 也删了；保存 User，顺便把 Orders 也保存了。这个功能很好用，但要注意——**删用户的时候顺便删订单，这个行为是不是你想要的**。有些业务里订单是财务报表，需要永久保留，删了用户订单也要保留。如果不是，改成 `CascadeType.PERSIST` 或者手动控制级联行为。

`fetch = FetchType.LAZY` 是懒加载。访问 `user.getOrders()` 的时候才查数据库，不访问就不查。这个设置在高并发场景下很重要——如果你设成 `EAGER`，查一个 User 顺便把它的 Orders、Roles、以及 Roles 的 Permissions 全查出来，一次查询能干几十张表，性能直接崩掉。我见过有人设了 `EAGER` 然后在 Service 层查一个用户列表，结果 N+1 问题爆了，100 个用户的列表触发了 300 条 SQL。

但懒加载有个问题——**在事务之外访问懒加载字段会爆 `LazyInitializationException`**。比如在 Controller 里访问 `user.getOrders()`，而 Service 方法已经 return 了，Session 关闭了，访问不到。所以要么在 Service 层把数据加载好，要么用 `EntityGraph` 指定要一起查的关联表。

```java
@EntityGraph(attributePaths = {"orders", "roles"})
Optional<User> findWithRelationsById(Long id);
```

还有一种做法是 Open Session in View，那个模式不推荐——它把数据库查询延迟到 View 层，虽然开发方便，但容易引发超时问题，而且违背了分层的原则。

---

## Schema 迁移：别再手改生产数据库

数据库 Schema 变更是最容易出生产事故的操作之一。没有版本控制、没有回滚方案、没有审批流程，改一个字段全凭记忆和胆量。

我经历过一次事故：线上数据库有个字段要改类型，从 `varchar(50)` 改成 `varchar(100)`。开发直接在生产环境执行 `ALTER TABLE`，几百万数据的表，锁了 40 秒，订单系统直接超时，用户打电话投诉。事后复盘，没人记得当时为什么要改这个字段，改之前有没有影响其他逻辑，上线之后才发现问题。

Schema 迁移工具解决的是这个问题。**所有变更都写脚本，所有脚本都进版本控制，所有上线都按顺序执行。** 出了问题可以回滚，有记录可以追溯，代码 review 的时候能看出来改了啥。

### Flyway 的用法

Flyway 是我比较喜欢的迁移工具，配置简单，上手快。

在 `pom.xml` 里加依赖：

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

在 application.yml 里配置：

```yaml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration
```

迁移脚本放在 `src/main/resources/db/migration/` 目录下，文件名有固定格式：

```
V1__create_user_table.sql
V2__add_user_email_index.sql
V3__create_order_table.sql
V4__add_order_user_foreign_key.sql
```

前缀 `V` 后面跟版本号，下划线分隔，后面是描述。Flyway 按版本号顺序执行脚本，已经执行过的不会再执行。版本号之间可以有间隔，比如 V1、V2、V10，允许中间跳着来，方便插入新的变更。

看个完整的迁移脚本例子：

```sql
-- V1__create_user_table.sql

CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    password VARCHAR(255) NOT NULL,
    email VARCHAR(100),
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    
    UNIQUE KEY uk_username (username),
    UNIQUE KEY uk_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE INDEX idx_status ON users(status);
CREATE INDEX idx_created_at ON users(created_at);
```

```sql
-- V4__add_order_user_foreign_key.sql

ALTER TABLE orders
ADD CONSTRAINT fk_order_user
FOREIGN KEY (user_id) REFERENCES users(id)
ON DELETE RESTRICT ON UPDATE CASCADE;
```

迁移脚本要满足几个原则：

**原子性**：一个脚本要么全成功，要么全失败。MySQL 的 DDL 是自动提交的，InnoDB 引擎下虽然可以配合事务用，但如果中途失败，部分执行的 DDL 会留下垃圾数据。所以尽量让每个脚本只做一个变更，大变更拆成多个小脚本按顺序执行。

**幂等性**：同一个脚本跑多遍结果一样。比如 `CREATE TABLE IF NOT EXISTS` 比 `CREATE TABLE` 更安全。`ALTER TABLE` 的幂等性更难保证，可以用 `DROP INDEX IF EXISTS` 之类的语法，或者在脚本开头先检查是否已经做过变更。

**向后兼容**：变更要能回滚。删字段之前要想清楚，旧代码在新 Schema 下还能不能跑。比如要删一个字段，先确认所有代码都不依赖它了，再上线。最好分两步：第一步先让代码不再写这个字段，第二步再删字段。

**禁止在脚本里删数据**：迁移脚本只改结构，不动业务数据。删数据要单独写工具脚本，进版本控制但和 Schema 变更分开。删数据的脚本要有明确的业务目的和执行时间点，不要在迁移脚本里混着删。

**大表变更要小心**：在生产环境改大表，直接 `ALTER TABLE` 会锁表。正确做法是：建新表，迁移数据，验证数据一致性，rename 老表。具体参考 GitHub 做的"在线改表结构"方案：建新表，灌数据，原子 rename，全程只有很短时间的锁。

### Liquibase：另一种选择

Liquibase 和 Flyway 功能类似，但用 XML 或 JSON 定义变更，而不是纯 SQL。好处是数据库无关，同一套变更脚本可以跑在 MySQL、PostgreSQL、Oracle 上。坏处是学习成本高，调试的时候要翻译一层。

我的经验是：团队小、项目单一用 Flyway 就够了；如果要做数据库无关的中间件平台，用 Liquibase。

---

## 索引设计：加错索引比不加更糟糕

索引能加速查询，但也会拖慢写入、占用空间。很多人知道"查询慢就加索引"，结果加了一堆索引用不上，白白浪费资源。我见过一个表 20 个字段，12 个索引，其中有一半从来没被用过。

**什么时候该加索引？**

查询条件里经常用的字段要加索引。`WHERE status = 'ACTIVE'`，`status` 上有索引就能快速过滤。如果 `status` 的取值只有两三种（Active/Inactive），加索引也没用——数据库发现索引的选择性太低，可能直接走全表扫描。

多字段查询的组合索引要按选择性排序。`WHERE status = ? AND created_at > ?`，建索引 `(status, created_at)`，查询能用到索引前缀。如果建在 `(created_at, status)`，`status` 条件就用到不索引。

```sql
-- 索引顺序：选择性高的放前面
CREATE INDEX idx_user_status_created ON orders(user_id, status, created_at);

-- 这个查询能用上索引
SELECT * FROM orders WHERE user_id = 123 AND status = 'PENDING';

-- 这个只能用到 user_id 部分
SELECT * FROM orders WHERE status = 'PENDING';
```

**主键和唯一索引**

主键自动有索引，而且是最快的。如果你的表没有主键，MySQL 会自动用第一个 `UNIQUE` 键当主键，如果也没有唯一键，就用隐藏的 `ROWID`。最好自己建主键，用有业务意义的自然键还是无业务意义的代理键，看场景。

代理键（自增 ID、UUID）是大多数场景下的选择。它不参与业务逻辑，改了业务逻辑不用改主键值，索引紧凑查询快。自然键（用户名、订单号）有业务含义，可能变，可能重复，比如用户名后期想允许改，就不适合当主键。

UUID 作为主键有个问题：UUID 是无序的，新插入的行随机分布在索引各处，InnoDB 需要频繁地分裂页面来容纳新数据，索引会变得又大又稀疏。用 UUID v7（时间有序）可以缓解这个问题，或者用自增 ID。

**索引不是越多越好**

每加一个索引，插入/更新/删除都要维护索引数据。索引占用磁盘空间，InnoDB 的索引在内存里放不下就要去磁盘读。查询优化器还要在多个索引里选一个，索引太多反而选错。

我见过一张表 15 个字段，加了 12 个索引，查询的时候 MySQL 选错了索引，执行计划一塌糊涂。用 `EXPLAIN` 看执行计划，确认查询真的用了索引，而不是想当然觉得有索引就快。

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 123 AND status = 'PENDING';
```

看输出的 `type`、`key`、`rows` 字段，判断是否走了索引、走了哪个索引、预计扫描多少行。如果 `rows` 特别大，说明可能要扫全表或者索引选错了。

**联合索引的左前缀原则**

如果建了联合索引 `(A, B, C)`，查询条件里有 `A` 就能用索引，有 `A AND B` 也能用，但只有 `B AND C` 或者只有 `C` 就用不上。

所以建索引之前，要先分析查询模式。看有哪些查询，每个查询的 WHERE 条件是什么，再决定怎么建索引。不要凭感觉建索引。

还有一个点：**索引覆盖**。如果查询的所有字段都在索引里，MySQL 就不用回表，直接从索引里返回值，速度快很多。比如 `SELECT id, order_no FROM orders WHERE user_id = ?`，如果建了 `(user_id, order_no)` 联合索引，就不用查主表了。

---

## 数据库范式与反范式

数据库设计有范式，1NF、2NF、3NF、BCNF，教科书上讲的。范式化设计消除冗余，节省空间，保证数据一致性。但有时候，为了性能，我们要故意违反范式。

**第一范式**：字段不可再分。比如"地址"这个字段，如果业务上经常要按城市查询，那拆成"省份"、"城市"、"区县"更合理。

**第二范式**：非主键字段要完全依赖于主键。如果有个"订单商品表"，主键是订单ID加商品ID，那"商品名称"只依赖商品ID而不完全依赖主键，这就有了冗余。解决方案是把商品信息拆出去。

**第三范式**：非主键字段不能有传递依赖。A 依赖 B，B 依赖 C，A 间接依赖 C。如果 A 表里存了 C 的字段，就是冗余。

范式化的好处是数据不冗余、一致性好维护。坏处是查询可能要 JOIN 很多表，在高并发场景下 JOIN 开销不小。

**什么时候反范式？**

读多写少的场景，反范式能省掉关联查询。用户表和用户详情表分开，每次查用户都要 JOIN，QPS 高了 JOIN 开销不小。合并成一张表，一次查询搞定，代价是数据稍微冗余一点。

需要避免关联查询的场景。比如报表查询，跨表关联跑几千万数据，跑一次可能几十秒。预先算好结果存起来，虽然数据有延迟，但查询秒回。

复杂计算预先做好。比如订单表里存了商品数量和单价，算总价要不要存？实时算的话每次查询都要 SUM；存总价的话，商品单价改了历史订单的总价就不对了。这个要看业务场景——如果历史订单的总价不允许变，存总价是对的；如果需要反映最新单价，那就实时算。

反范式不是乱来，要清楚代价是什么。数据冗余了，同步更新的逻辑要跟上；报表数据有延迟，要和业务方沟通清楚能不能接受。反范式是权衡，不是偷懒。

---

## 总结

数据库设计是基础设施，烂了后面全是坑。

JPA Entity 要把命名策略、字段映射、关系配置都写清楚，NamingStrategy 统一配置，枚举用字符串存，懒加载要注意 `LazyInitializationException`，关系映射的拥有方要搞清楚。

Schema 迁移用 Flyway，所有变更进版本控制，脚本要原子、幂等、向后兼容，禁止在迁移脚本里删数据。大表变更要小心，不要直接 ALTER TABLE，用建新表、迁移数据、rename 的方式。

索引按需加，别乱加。组合索引注意字段顺序，用 `EXPLAIN` 验证执行计划。左前缀原则要记牢，尽量用覆盖索引减少回表。

范式化是目标，反范式是权衡，要有意识地选择，不能无脑范式化也不能无脑冗余。清楚每种选择的代价，比死守规则更重要。

数据库 Schema 是应用的地基，地基不稳，楼越盖越危险。

---

**下期预告**：后端写完了，前端怎么调用？REST API 怎么设计才规范？OpenAPI 怎么写才能让前端和测试都省心？下篇文章聊聊 API 设计。

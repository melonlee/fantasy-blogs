# Vibe Coding 实战（一)：什么是 Vibe Coding——Java 开发者需要知道的新范式

---

![Vibe Coding intro](./images/01-vibe-coding-intro.jpg)

2025 年 2 月，Andrej Karpathy 在 X 上发了一条推文，描述了一种新的编程方式：

> "There's a new kind of coding I call 'vibe coding', where you fully give in to the vibes, embrace exponentials, and forget that the code even exists."

他用 Cursor Composer + 语音输入，几乎不碰键盘，"问一下'把这个按钮颜色改浅一点'，它就改了"。

听起来像开玩笑。但 Karpathy 说的是真的。

LLM 已经足够好，好到可以把"描述需求"变成"生成代码"。2026 年的工具链（Cursor、Claude Code CLI、Claude 3.5/4）把这个能力放到了每个开发者的手里。

问题是：你是用这个新能力的那个人，还是那个看着别人用它的人？

## 不是取代，是协作

Vibe coding 不是"AI 取代程序员"。它是一种新的分工：人描述意图，AI 负责实现细节。

传统模式：你想做一个功能，自己写代码，自己调试，自己测试。每一行代码都经你的手。

Vibe coding 模式：你描述功能，AI 生成代码，你审查，你调整，AI 再修改，循环直到完成。代码还是那个代码，但谁写的变了。

从"手工写汇编"到"用 C 编译器"——不是程序员被取代了，是效率被放大了。汇编程序员还在，只是换工具了。

Java 开发者的情况也类似。Spring Boot 应用可能有几百个类，boilerplate 代码占大头。如果你能用自然语言描述需求，AI 帮你生成 boilerplate，你就能把时间放在真正需要思考的地方——架构设计、业务逻辑、系统集成。

## 三层规则

Karpathy 的描述是方向性的，不是操作手册。实际做 vibe coding，需要一套方法论。

**第一层：说清楚你要什么。**

这是最被低估的技能。

"做一个用户管理系统"不是需求描述。"做一个 Spring Boot 用户管理系统，支持注册和登录，JWT 认证，用户表字段是 id、username、email、password_hash，密码用 BCrypt 加密，长度不少于 8 位"才是。

好的需求描述需要三个要素：明确的边界（做什么不做什么）、具体的约束（框架、技术栈、规范）、可验证的标准（怎么算完成）。

**第二层：学会校正。**

AI 代码不会一次就完全对。你要校正它。

好的校正不是"不对，再来一次"，而是具体的：
- 指出具体问题（"这个方法没有处理空指针"）
- 给出具体方向（"应该在这里加一个 null 检查"）
- 解释背后原因（"因为前端传过来的 email 可能为空"）

**第三层：知道什么不该让 AI 做。**

AI 适合做 boilerplate、简单 CRUD、测试桩、文档、代码转换、重构。

AI 不适合做安全关键逻辑、事务边界处理、高并发设计、没有明确规则的复杂业务逻辑。

对 Java 开发者来说，这意味着一条明确的分界线：AI 帮你生成代码，但架构决策、安全审查、性能优化必须你来做。AI 是工具，不是架构师。

## Vibe Coding 的三个深度

不是所有场景都适合同样的 vibe coding 深度。

**浅层：辅助工具。**

你写代码，AI 补全、解释、生成测试。GitHub Copilot、Cursor Tab 补全属于这个层次。大多数开发者已经在用。

**中层：搭档协作。**

你和 AI 一起开发。你描述功能，AI 生成实现，你审查，AI 修改，循环往复直到完成。Cursor Composer、Claude Code CLI 属于这个层次。

**深层：导演模式。**

你只描述目标和约束，AI 自主完成实现，你最后审查结果。适合方向明确的任务（如"把这个 REST API 改成 GraphQL"）。

Java 开发中，浅层和中层最常见。深层适合框架迁移、大规模重构。

## Java 开发者的特殊挑战

Vibe coding 工具最早流行于前端和脚本语言社区。Java 开发者有一些特殊的挑战：

**类型系统。**

Java 是强类型语言，AI 生成代码时容易在类型上出错——返回类型不匹配、泛型参数错误、NullPointerException。审查 AI 生成的 Java 代码时，类型检查是第一道关。

**框架复杂性。**

Spring Boot 有大量的 annotation、配置、依赖注入规则。AI 有时候会生成"语法正确但 Spring 不认"的代码——比如用错了 @Autowired 的位置，或者混淆了 @Service 和 @Repository。

**编译-运行周期长。**

Python/JavaScript 可以快速 REPL 测试。Java 需要编译-运行周期，每次调整都要等构建。这会直接影响 vibe coding 的流畅度。

**企业规范。**

Java 项目通常有 Checkstyle、PMD、SpotBugs 等规范，测试覆盖率要求，代码审查流程。AI 生成的代码不一定符合这些规范。

知道这些挑战的存在，是克服它们的第一步。

## 为什么现在是最好的时机

2026 年初，vibe coding 的工具链已经相当成熟：

- Cursor 提供了完整的 AI 编程环境，支持 Composer Agent、Debug 集成
- Claude Code CLI 提供了终端里的 Agent 能力，支持 Git 操作、文件编辑、多轮对话
- SivaLabs Marketplace 提供了专门的 Spring Boot 开发插件
- Claude 3.5/4 在 Java 代码生成上的表现已经超过 GPT-4

对于 Java 开发者来说，现在是上车的好时机。不是因为"不用就被淘汰"，而是因为你可以把更多时间放在真正有价值的事情上，而不是花大量时间写 boilerplate。

## 这套系列的内容

接下来的文章，会覆盖：

- Cursor 和 Claude Code CLI 的 Java 开发实战
- 如何写好需求 Spec（这是 vibe coding 最核心的技能）
- Cursor 的进阶功能：Rules、Context、Agents
- Java 调试和测试的 vibe coding 方法
- Spring Boot 开发的最佳实践
- 数据库设计、API 设计的具体场景
- 全栈开发最后三篇

每篇都有代码示例，都基于 Java（Spring Boot），都是我从实际项目中总结的经验。

如果你想真正掌握 vibe coding，不要只看完文章——装好工具，打开一个项目，试一下。

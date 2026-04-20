# Vibe Coding 实战（三）：Claude Code CLI——终端里的 Java 开发搭档

---

![Claude Code CLI](./images/03-claude-code-cli.jpg)

Cursor 是 IDE，适合在图形界面里工作。Claude Code CLI 是终端工具，适合在命令行里工作。

两者的关系不是竞争，是互补。你在 IDE 里写代码、在图形界面里调试、在浏览器里看文档——这些时候用 Cursor。你在终端里快速生成代码、在 Git 操作中让 AI 辅助、在自动化脚本里调用 AI——这些时候用 Claude Code CLI。

Claude Code CLI 是 Anthropic 官方发布的命令行工具，核心功能是让 Claude 模型在终端里"看到"你的项目、理解代码、执行任务。

## 安装

```bash
npm install -g @anthropic-ai/claude-code
```

或者用 Homebrew：

```bash
brew install claude-code
```

安装完成后，运行 `claude` 启动交互式会话。第一次运行会引导你登录 Claude 账号并配置 API Key。

## 核心命令

Claude Code CLI 有几个核心命令：

```bash
# 进入交互式会话
claude

# 非交互式：直接在命令行执行单个任务
claude "为 User 类添加 equals 和 hashCode 方法"

# 读取文件内容
claude --print "解释这个 Service 类的逻辑" < src/UserService.java

# 指定模型
claude --model opus "重写这段代码"
```

## Java 项目里的实际用法

**场景一：生成代码。**

在项目目录下运行：

```bash
claude "在 com.example.service 包下创建一个 EmailService，包含 send方法，使用 Spring Boot 的 JavaMailSender"
```

Claude 会分析项目结构，找到合适的位置，生成代码。

**场景二：解释代码。**

```bash
claude --print "这段代码在做什么？" < src/com/example/repository/OrderRepository.java
```

Claude 会分析代码并输出解释。

**场景三：Git 操作。**

```bash
# 写 commit message
git diff | claude "为一个修改了三个文件的 commit 写一条简洁的中文 commit message"

# 审查代码变更
git diff | claude "审查这些变更，有没有潜在问题？"
```

**场景四：重构。**

```bash
claude "把 UserService 中的字段注入改成构造器注入"
```

Claude 会分析代码，找到所有需要修改的地方，生成新的代码。

## Claude Code 的 Agent 模式

Claude Code 支持 Agent 模式——让 Claude 自主完成多步任务，而不是单次调用。

```bash
# 启动 Agent 模式
claude --agent

# 在 Agent 模式里，你可以说：
# > 帮我把这个模块的重试逻辑改成指数退避
# > 检查一下有没有明显的性能问题
# > 帮我写一个性能测试
```

Agent 模式会保持上下文，在你多个指令之间保持记忆，直到你说 "exit"。

## 与 Cursor 的使用场景对比

| 场景 | Cursor | Claude Code CLI |
|------|--------|-----------------|
| 生成新代码 | ✅ Composer 多文件生成 | ✅ 单次调用 |
| 调试 | ✅ 图形化断点 | ✅ log 分析 |
| 代码审查 | ✅ Diff 视图 | ✅ git diff 集成 |
| 快速重写 | 一般 | ✅ 最适合 |
| 自动化脚本 | ❌ | ✅ 支持 |
| 多轮协作 | ✅ 长期上下文 | ✅ Agent 模式 |

## Java 开发者的工作流

实际工作中，我建议这样组合：

在 Cursor 里做主要开发：写代码、调试、运行测试、生成新功能。Cursor 的 IDE 体验更完整。

在 Claude Code CLI 里做辅助工作：写 commit message、代码审查、快速生成单个文件、重构任务。CLI 更快，不需要切换窗口。

一个典型的一天：

- 早上到公司，用 Claude Code 跑一下 `git log` 看一下昨天的变更，让 AI 帮我回忆昨晚做了什么
- 开始写代码，在 Cursor Composer 里生成新功能的框架代码
- 写完后，用 Claude Code 的 `git diff | claaude` 做快速代码审查
- 有 bug，在 Cursor 里调试，Claude Code 帮助分析堆栈信息
- 提交前，用 Claude Code 写 commit message

## 安全提示

Claude Code 能读写你的文件、执行 git 操作、运行你项目里的测试命令。

在使用前，确保你了解 Claude Code 会执行什么操作。对于敏感项目，可以：

```bash
# 设置只读模式，Claude 不能修改文件
claude --read-only

# 或者在项目中设置 .claude 文件夹，定义权限边界
```

```json
// .claude/config.json
{
  "permissions": {
    "allow": ["Read", "Glob"],
    "deny": ["Bash:mvn*", "Bash:npm*"]
  }
}
```

## 几个实用的技巧

**技巧一：用 --include 限制 AI 看到的文件。**

```bash
claude --include "**/User*.java" "为 User 相关类生成单元测试"
```

这样 Claude 只会分析 User 相关的文件，不会读整个项目。

**技巧二：用 --no-input 防止误操作。**

```bash
claude --no-input "审查代码" < /tmp/mycode.java
```

只输出审查结果，不允许 Claude 执行任何操作。

**技巧三：在 .claude/rules/ 里定义 Java 规范。**

Claude Code 也支持 Rules。在项目根目录的 `.claude/rules/` 里定义规范文件，Claude 在处理你的项目时会遵守这些规则。

```markdown
# java-spring.md
- Service 类必须加 @Slf4j
- 使用构造器注入，不用 @Autowired 字段注入
- Entity 必须加 @NoArgsConstructor @AllArgsConstructor
```

这个功能和 Cursor 的 Rules 完全一致，同一个项目可以在两个工具里共用规范。

Claude Code CLI 是终端里强大的辅助工具。用好它，不需要离开命令行就能得到 AI 的能力。

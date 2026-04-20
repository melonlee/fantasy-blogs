# Harness 工程实战（六)：主流框架横评——Anthropic SDK vs OpenAI Agents SDK vs LangGraph vs CrewAI

---

工具好不好用，不能只看功能列表。

Claude Code 在 Codex 界面上比在通用聊天窗口中表现更好。这不是因为模型不同，是因为 Harness 不同。

这说明一个框架的核心价值不在于它"能做什么"，在于它"怎么做"——它的设计哲学、它的默认假设、它让你以什么方式思考问题。

![主流 Agent 框架横评对比](./images/harness-framework-comparison.jpg)

## Anthropic Claude Agent SDK

Anthropic 的 SDK 是"哑循环"哲学的最佳体现。

核心抽象是一个 `query()` 函数：创建 Agent 循环，返回流式消息的异步迭代器。就这么多。

```python
async for event in client.query(
    system_prompt="You are a coding assistant.",
    messages=[{"role": "user", "content": "Fix the bug"}],
    tools=[read_file, edit_file, run_command],
):
    print(event)
```

循环本身是"哑"的，所有智能都在模型里。Harness 不做推理，不做规划，只做机械的事情。

Claude Code 用 git 提交作为检查点，进度文件作为结构化草稿本。这让它天然具备版本控制能力。

## OpenAI Agents SDK

OpenAI 的 SDK 是"代码优先"的设计。

工作流逻辑用原生 Python 表达，而不是图 DSL。你写的是 Python 代码，不是某种框架特定的描述语言。

```python
from agents import Agent, Runner

agent = Agent(
    instructions="You are a helpful assistant.",
    tools=[search_web, calculate]
)

result = Runner.run_sync(agent, "What's 15% of 200?")
```

工具支持三种类型：函数工具、托管工具（WebSearch、CodeInterpreter）、MCP 服务器工具。

状态管理提供四种互斥策略：应用程序内存、SDK 会话、服务器端 Conversations API，或轻量级 `previous_response_id` 链接。

## LangGraph

LangGraph 将 Harness 建模为显式状态图。

这不是图 DSL——图是 Python 代码写的。你定义节点和边，LangGraph 帮你处理执行和状态流转。

```python
workflow = StateGraph(AgentState)
workflow.add_node("llm_call", call_model)
workflow.add_node("tool_call", execute_tools)

def should_continue(state):
    return "continue" if state.get("tool_calls") else "end"

workflow.add_conditional_edges(
    "llm_call",
    should_continue,
    {"continue": "tool_call", "end": END}
)
app = workflow.compile()
```

检查点发生在超级步骤边界，支持中断后恢复和时间旅行调试。

## CrewAI

CrewAI 实现基于角色的多 Agent 架构。

```python
from crewai import Agent, Task, Crew

researcher = Agent(
    role="Research Analyst",
    goal="Find the latest AI developments",
    tools=[search_tool]
)
writer = Agent(
    role="Tech Writer",
    goal="Write clear summaries",
    tools=[write_tool]
)

crew = Crew(agents=[researcher, writer], tasks=[research_task, write_task])
crew.kickoff()
```

CrewAI 的独特之处是"角色"概念。每个 Agent 被赋予一个人格：背景故事、行为模式、专业领域。

## 横评对比

| 维度 | Claude Agent SDK | OpenAI Agents SDK | LangGraph | CrewAI |
|------|------------------|-------------------|-----------|--------|
| 核心抽象 | query() 函数 | Runner 类 | 状态图 | 角色 + Crew |
| 多 Agent | Fork/Teammate/Worktree | 交接 + Agent 作为工具 | 嵌套状态图 | Role-based Crews |
| 状态管理 | git + 进度文件 | 四种策略选择 | 检查点 + 时间旅行 | 任务状态 |
| 学习曲线 | 低 | 低 | 中 | 中 |
| 适用场景 | 深度编码任务 | 通用 Agent | 复杂流程编排 | 多角色协作 |

## 关键洞察：没有银弹

Claude Agent SDK 的"哑循环"哲学在编码场景表现出色，但在需要复杂流程控制的场景显得能力不足。

OpenAI Agents SDK 的"代码优先"降低了门槛，但复杂流程的代码可能变得难以维护。

LangGraph 的状态图提供了最强表达力，但学习成本和调试成本也最高。

选择框架时，最重要的是想清楚你的场景需要什么。

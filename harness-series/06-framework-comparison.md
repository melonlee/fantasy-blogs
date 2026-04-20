# Harness 工程实战（六）：主流框架横评——Anthropic SDK vs OpenAI Agents SDK vs LangGraph vs CrewAI

---

工具好不好用，不能只看功能列表。

Claude Code 在 Codex 界面上比在通用聊天窗口中表现更好。这不是因为模型不同，是因为 Harness 不同。

这说明一个框架的核心价值不在于它"能做什么"，在于它"怎么做"——它的设计哲学、它的默认假设、它让你以什么方式思考问题。

选框架，本质上是选择一种做事的思维方式。

![主流 Agent 框架横评对比](./images/harness-framework-comparison.jpg)

## Anthropic Claude Agent SDK：哑循环哲学

Anthropic 的 SDK 是"哑循环"哲学的最佳体现。

核心抽象是一个 `query()` 函数：创建 Agent 循环，返回流式消息的异步迭代器。就这么多。没有复杂的图，没有状态机，没有花哨的编排——只有一个循环，模型驱动它。

```python
from anthropic import AsyncAnthropic
from anthropic._utils._utils import parse_messages
import json

class ClaudeHarness:
    """
    Anthropic 的"哑循环"实现。
    所有智能都在模型里，Harness 只做机械的事情。
    """

    def __init__(self, api_key: str, model: str = "claude-opus-4-5"):
        self.client = AsyncAnthropic(api_key=api_key)
        self.model = model

    async def query(
        self,
        system_prompt: str,
        messages: list,
        tools: list = None,
        max_tokens: int = 4096
    ):
        """
        主循环入口。
        返回一个异步迭代器，每次 yield 一个事件。
        """
        tool_defs = [self._tool_to_def(t) for t in (tools or [])]

        while True:
            # 构建请求
            response = await self.client.messages.create(
                model=self.model,
                max_tokens=max_tokens,
                system=system_prompt,
                tools=tool_defs,
                messages=messages
            )

            # yield 响应内容
            if response.content:
                for block in response.content:
                    if hasattr(block, 'text') and block.text:
                        yield {"type": "content", "text": block.text}
                    elif hasattr(block, 'type') and block.type == 'tool_use':
                        yield {"type": "tool_call", "tool": block.name, "input": block.input}

            # 如果没有工具调用，结束
            if not hasattr(response, 'stop_reason') or response.stop_reason != 'tool_use':
                break

            # 执行工具
            # 注意：这里没有自动执行，需要调用者自己处理
            # 这体现了"哑循环"的设计——Harness 只返回结果，不做决定

    def _tool_to_def(self, tool) -> dict:
        """将工具对象转换为 API 格式"""
        return {
            "name": tool.name,
            "description": tool.description,
            "input_schema": tool.parameters
        }
```

循环本身是"哑"的：Harness 不做推理，不做规划，只做机械的事情——组装消息、调用 API、返回结果。

Claude Code 的实现走得更远。它用收集-行动-验证循环：
- 收集：搜索文件、读代码、了解项目结构
- 行动：编辑文件、运行命令
- 验证：运行测试、检查输出

状态管理的选择也很有意思：用 git 提交作为检查点，进度文件作为结构化草稿本。这让它天然具备版本控制能力——每次重要操作都有 git commit，可以随时回滚。

```python
class GitCheckpointManager:
    """
    用 git 管理 Agent 状态。
    每次重要操作后自动 commit，可以回滚。
    """

    def __init__(self, repo_path: str):
        self.repo = Repo(repo_path)

    def checkpoint(self, message: str, diff: str = None):
        """创建检查点"""
        if diff:
            # 应用 diff
            self.repo.git.checkout(diff)

        # 添加所有更改
        self.repo.git.add("-A")

        # 提交
        self.repo.index.commit(message)

    def rollback(self, n: int = 1):
        """回滚 n 个提交"""
        self.repo.git.reset("--hard", f"HEAD~{n}")

    def get_history(self, limit: int = 10) -> list:
        """获取最近的检查点历史"""
        commits = list(self.repo.iter_commits(max_count=limit))
        return [(c.message, c.hexsha[:7], c.committed_datetime) for c in commits]
```

**适用场景**：深度编码任务、需要强版本控制的生产环境、追求模型最大自主性的场景。

**不适用**：需要复杂流程编排、需要精确控制执行顺序、需要多 Agent 协作流程。

## OpenAI Agents SDK：代码优先

OpenAI 的 SDK 是"代码优先"的设计。

工作流逻辑用原生 Python 表达，而不是某种框架特定的 DSL。你写的是 Python 代码，不是某种框架特定的描述语言。

```python
from agents import Agent, Runner, function_tool
from typing import List


@function_tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    return f"Weather in {city}: 22°C, sunny"


@function_tool
def search_flights(origin: str, destination: str, date: str) -> str:
    """Search for flights between two cities."""
    return f"Found 3 flights from {origin} to {destination} on {date}"


# 创建 Agent
travel_agent = Agent(
    name="travel_assistant",
    instructions="""你是一个专业的旅行助手。
    帮助用户搜索航班和天气信息。
    如果用户提到航班，用 search_flights 工具搜索。
    如果用户问天气，用 get_weather 工具查询。
    始终提供准确、有用的信息。""",
    tools=[get_weather, search_flights]
)


# 运行 Agent
def main():
    result = Runner.run_sync(
        travel_agent,
        "帮我查一下明天从北京到上海的航班，还有北京今天的天气"
    )
    print(result.final_output)
```

工具支持三种类型：

```python
# 类型一：函数工具（装饰器）
@function_tool
def my_function(arg1: str, arg2: int) -> str:
    """工具描述"""
    return f"Result: {arg1}, {arg2}"

# 类型二：托管工具（内置）
from agents.tools import WebSearchTool, CodeInterpreterTool, FileSearchTool

agent = Agent(
    tools=[
        WebSearchTool(),      # 内置网络搜索
        CodeInterpreterTool(), # 代码解释器
        FileSearchTool()       # 文件搜索
    ]
)

# 类型三：MCP 服务器工具（Model Context Protocol）
from agents.tools import MCPTool

mcp_tool = MCPTool(
    server="http://localhost:3000/mcp",
    tool_name="database_query"
)
```

多 Agent 支持两种模式：

```python
# 模式一：Agent 作为工具——专家处理有界子任务
researcher = Agent(
    name="researcher",
    instructions="你是一个研究助手...",
    tools=[search_tool, read_file_tool]
)

writer = Agent(
    name="writer",
    instructions="你是一个写作助手...",
    tools=[write_file_tool]
)

# researcher 作为工具被 writer 使用
writer_with_researcher = Agent(
    name="writer",
    instructions="你是一个写作助手。如果需要研究，用 researcher 工具。",
    tools=[researcher.as_tool(), write_file_tool]  # 把 Agent 转成工具
)

# 模式二：交接——专家获得完全控制
def task_with_handoff():
    # 初始化专家 Agent
    expert = Agent(
        name="expert",
        instructions="你是一个专业的数据分析师...",
        tools=[analyze_tool, visualize_tool]
    )

    # 主 Agent 交接控制权
    orchestrator = Agent(
        name="orchestrator",
        instructions="""你是一个任务协调员。
        当遇到需要深度分析的任务时，切换到专家模式。
        使用 handoff 工具将控制权交给专家 Agent。""",
        tools=[expert.as_handoff_tool()]  # 交接工具
    )

    result = Runner.run_sync(orchestrator, user_task)
    return result
```

状态管理提供四种互斥策略：

```python
from agents import InProcessMemory, AgentSession, ConversationsAPI

# 策略一：进程内内存（简单，但不能跨进程）
memory = InProcessMemory()

# 策略二：SDK 会话管理（自动处理，有限持久化）
session = AgentSession(agent=my_agent)

# 策略三：服务器端 Conversations API（需要 OpenAI 后端）
conversations = ConversationsAPI(api_key="...")

# 策略四：previous_response_id 链接（轻量，自托管）
result_1 = Runner.run_sync(agent, "First task")
result_2 = Runner.run_sync(
    agent,
    "Follow-up task",
    previous_response_id=result_1.response_id  # 链接到上一轮
)
```

**适用场景**：通用 Agent 任务、快速原型开发、需要灵活状态管理的场景。

**不适用**：复杂流程编排（代码会变得难以维护）、需要细粒度控制执行流程。

## LangGraph：显式状态图

LangGraph 将 Harness 建模为显式状态图。这不是图 DSL——图是 Python 代码写的。

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict, Annotated
import operator


class AgentState(TypedDict):
    """Agent 的状态类型定义"""
    messages: Annotated[list, operator.add]  # 新消息追加到列表
    current_task: str
    completed_steps: list[str]
    errors: list[str]
    next_action: str
    context: dict  # 任意额外上下文


def should_continue(state: AgentState) -> str:
    """
    条件边：根据当前状态决定下一步走向。
    这是 LangGraph 的核心：状态决定流向。
    """
    if state.get("errors") and len(state["completed_steps"]) < 3:
        # 有错误但还没尝试恢复，走恢复路径
        return "recover"
    elif not state.get("messages")[-1].get("tool_calls"):
        # 没有更多工具调用，结束
        return END
    elif state.get("next_action") == "verify":
        # 需要验证，走验证路径
        return "verify"
    else:
        # 继续执行
        return "execute"


workflow = StateGraph(AgentState)

# 添加节点
workflow.add_node("planner", plan_node)       # 规划下一步
workflow.add_node("executor", execute_node)   # 执行工具调用
workflow.add_node("verifier", verify_node)    # 验证结果
workflow.add_node("recoverer", recover_node)  # 错误恢复

# 设置入口点
workflow.set_entry_point("planner")

# 固定边
workflow.add_edge("planner", "executor")
workflow.add_edge("executor", "verify")
workflow.add_edge("recoverer", "executor")  # 恢复后回到执行

# 条件边
workflow.add_conditional_edges(
    "verify",
    should_continue,
    {
        "recover": "recoverer",
        "execute": "executor",
        END: END
    }
)

# 编译，带检查点
checkpointer = MemorySaver()
app = workflow.compile(checkpointer=checkpointer)


# 运行
async def run_agent(task: str):
    config = {"configurable": {"thread_id": "session-123"}}

    final_state = None
    async for state in app.astream(
        {"messages": [{"role": "user", "content": task}], "current_task": task},
        config
    ):
        final_state = state

    return final_state
```

检查点机制让中断恢复和时间旅行调试成为可能：

```python
# 中断：暂停执行等待人工确认
workflow.add_node("human_review", human_review_node)

workflow.add_edge("executor", "human_review")
workflow.add_edge("human_review", "execute", condition=lambda s: s["approved"])
workflow.add_edge("human_review", END, condition=lambda s: not s["approved"])

# 时间旅行：回到某个历史状态
async def time_travel(thread_id: str, checkpoint_id: str):
    """回到某个检查点重新执行"""
    config = {
        "configurable": {
            "thread_id": thread_id,
            "checkpoint_id": checkpoint_id  # 指定检查点
        }
    }

    # 从检查点恢复，重新执行
    async for event in app.astream(None, config):
        print(event)
```

嵌套状态图支持复杂的多 Agent 协作：

```python
from langgraph.graph import StateGraph
from subagent import ResearchAgent, WritingAgent


class ParentState(TypedDict):
    task: str
    research_results: str
    draft: str
    final_output: str


parent_workflow = StateGraph(ParentState)

# 嵌套子图
parent_workflow.add_node(
    "researcher",
    ResearchAgent().compile()
)  # 整个子图作为父图的一个节点
parent_workflow.add_node(
    "writer",
    WritingAgent().compile()
)

parent_workflow.add_edge("researcher", "writer")
parent_workflow.add_edge("writer", END)

parent_app = parent_workflow.compile()
```

**适用场景**：复杂流程编排、需要条件分支和循环、需要在执行中暂停等待输入、需要时间旅行调试。

**不适用**：简单任务（过度工程）、需要快速原型（学习曲线陡）。

## CrewAI：角色驱动

CrewAI 实现基于角色的多 Agent 架构。它的核心抽象是"角色"——每个 Agent 被赋予一个人格。

```python
from crewai import Agent, Task, Crew, Process


# 定义 Agent（角色）
researcher = Agent(
    role="Research Analyst",
    goal="Find the latest developments in {topic}",
    backstory="""
    你是一个资深的研究分析师，专注于{topic}领域。
    你有十年的行业研究经验，善于发现趋势和机会。
    你总是提供基于数据的见解，避免猜测。
    """,
    verbose=True,
    allow_delegation=False,  # 不委托给其他人
    tools=[search_tool, read_file_tool]
)


writer = Agent(
    role="Content Writer",
    goal="Write clear, engaging content based on research",
    backstory="""
    你是一个资深的技术写作者，擅长把复杂的技术概念解释得通俗易懂。
    你的文章逻辑清晰、层次分明，读者能快速抓住重点。
    你善于用例子和类比来解释抽象概念。
    """,
    verbose=True,
    allow_delegation=True,  # 可以委托给 researcher
    tools=[write_file_tool]
)


# 定义任务
research_task = Task(
    description="""研究 {topic} 的最新进展，包括：
    1. 主要技术突破
    2. 关键玩家和他们的策略
    3. 市场趋势和预测
    4. 潜在风险和机会""",
    agent=researcher,
    expected_output="一份详细的研究报告，包含数据支持和来源引用"
)


write_task = Task(
    description="""基于研究报告，写一篇关于 {topic} 的文章：
    1. 吸引人的开头
    2. 核心内容（技术、市场、趋势）
    3. 深入分析
    4. 结论和预测
    5. 参考资料""",
    agent=writer,
    context=[research_task],  # 依赖研究任务的结果
    expected_output="一篇结构清晰、内容丰富的文章"
)


# 组装 Crew
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.hierarchical,  # 层次化流程：有一个 Manager 协调
    manager_agent=Agent(
        role="Project Manager",
        goal="确保任务按时完成，质量达标",
        backstory="你是一个经验丰富的项目经理..."
    )
)


# 启动
result = crew.kickoff(inputs={"topic": "AI Agent 的发展趋势"})
print(result)
```

CrewAI 的独特之处是"角色"概念：
- **role**：职位名称
- **goal**：目标
- **backstory**：背景故事（影响 Agent 的行为模式和说话风格）

Flows 层添加了"具有智能的确定性骨干"：

```python
from crewai import Flow, crew, flat, sequential


class ArticleFlow(Flow):

    @crew
    def create_article(self):
        # 定义流程：研究 -> 写草稿 -> 审核 -> 修改 -> 发布
        return crew(
            agents=[researcher, writer, reviewer],
            tasks=[research_task, draft_task, review_task, revise_task, publish_task],
            process=sequential  # 顺序执行
        )

    def after(self, output):
        # 后处理钩子
        if output.failed:
            self.notify_failure(output)
        else:
            self.publish(output)


flow = ArticleFlow()
result = flow.trigger(topic="AI Agent 的发展趋势")
```

**适用场景**：多角色协作、需要定义清晰角色的任务、内容创作类任务。

**不适用**：需要精确控制执行流程、代码相关的精确任务、简单的单步任务。

## 横评对比

| 维度 | Claude Agent SDK | OpenAI Agents SDK | LangGraph | CrewAI |
|------|------------------|-------------------|-----------|--------|
| 核心抽象 | `query()` 函数 | `Agent` + `Runner` | 状态图 | `Agent` + `Crew` + `Task` |
| 多 Agent | Fork/Teammate/Worktree | 交接 + Agent 作为工具 | 嵌套状态图 | Role-based Crews |
| 状态管理 | git + 进度文件 | 四种策略选择 | 检查点 + 时间旅行 | 任务状态 |
| 学习曲线 | 低 | 低 | 中 | 中 |
| 适用场景 | 深度编码任务 | 通用 Agent | 复杂流程编排 | 多角色协作 |
| 显式 vs 隐式 | 隐式（模型驱动） | 隐式 | 显式（图结构） | 隐式（角色驱动） |
| 验证机制 | 内置验证循环 | 输入/输出/工具三级 | 可扩展 | 可扩展 |

## 关键洞察：没有银弹

看完这四个框架，一个重要的事实是：没有哪个框架在所有场景都优于其他。

Claude Agent SDK 的"哑循环"哲学在编码场景表现出色，但在需要复杂流程控制的场景显得能力不足——你需要自己实现很多逻辑。

OpenAI Agents SDK 的"代码优先"降低了门槛，但复杂流程的代码可能变得难以维护——Python 代码写的流程控制不是最优雅的解决方案。

LangGraph 的状态图提供了最强表达力，但学习成本和调试成本也最高——你需要理解状态机、节点、边、条件这些概念。

CrewAI 的角色抽象让多 Agent 协作很自然，但如果你不需要角色扮演，这个抽象就多余——role 和 backstory 对简单任务没有帮助。

选择框架时，最重要的是想清楚你的场景需要什么。

一个需要深度定制、流程复杂的系统，LangGraph 的表达能力值得投入学习成本。

一个相对简单的 Agent 应用，OpenAI Agents SDK 的代码优先设计能快速出活。

一个多角色协作的内容创作系统，CrewAI 的角色抽象非常自然。

一个需要强版本控制的编码环境，Claude Agent SDK 的哑循环 + git 检查点是最干净的方案。

# Harness 工程实战（八）：多智能体系统的工程实践——何时拆、怎么拆、如何通信

---

大多数时候，单个 Agent 够用。

Anthropic 和 OpenAI 都说过同样的话。这句话重要到值得重复：**先让单个 Agent 做他能做的一切，只有在明确需要时才拆成多个。**

因为多 Agent 系统有显著的开销。

很多人在第一次设计多 Agent 系统时，会觉得多 Agent 理所当然地比单 Agent 强——就像多线程比单线程强、多台服务器比单机强一样。但这是一个危险的类比。

多线程的代价是同步复杂性。多 Agent 的代价是通信开销、上下文损失和状态复杂性。在某些场景下，这些代价是值得的；在更多场景下，这些代价是不值得的。

![多智能体协作与通信](./images/harness-multi-agent.jpg)

## 拆分的多重代价：为什么多 Agent 不是银弹

多 Agent 不是"一个 Agent 做不到的，两个 Agent 加起来就能做到"。

每次从一个 Agent 交接给另一个，你都在损失上下文——需要把必要的信息传递过去，同时避免把整个上下文复制过去导致爆炸。Agent 之间需要通信，需要额外的 LLM 调用来做路由，需要管理状态同步。

这些开销加起来可能超过拆分带来的收益。

LLMCompiler 的论文报告了一个重要数据：计划-执行比顺序 ReAct 快 3.6 倍。这个加速来自于把规划步骤分离出来并行执行。但这种加速的前提是：规划的开销比顺序执行的总和要小。如果规划本身很重，这个优化就不值得。

多 Agent 也是同样的道理。

只有当任务足够复杂，拆分后的收益超过通信开销时，拆分才有意义。

那么什么情况下拆分是值得的？

## 什么时候应该拆：两个触发条件

### 触发条件一：工具过载

当一个 Agent 需要使用大约超过 10 个重叠工具时，模型开始混淆。

这不是一个精确的数字——有些模型在 5 个工具时就开始混淆，有些模型能处理 15 个——但数量级的判断是对的。工具越多，"我应该用哪个"这个问题本身就开始消耗模型的推理能力。

一个具体例子：假设你设计了一个数据分析 Agent，它有 50 个工具——读取 CSV、过滤数据、聚合计算、绘图、导出、发送邮件……当用户说"分析一下销售数据"时，模型需要从 50 个工具中选择。但"销售数据分析"可能只需要其中 3-4 个工具。

解决方案是按功能域拆分：

```python
class MultiDomainAgent:
    """
    按功能域拆分的多 Agent 系统。
    """

    def __init__(self):
        self.domains = {
            "data_reading": DataReadingAgent(tools=[
                read_csv, read_json, read_sql, read_api
            ]),
            "data_processing": DataProcessingAgent(tools=[
                filter, aggregate, join, transform
            ]),
            "data_visualization": VisualizationAgent(tools=[
                plot_line, plot_bar, plot_scatter, export_image
            ]),
            "communication": CommunicationAgent(tools=[
                send_email, post_slack, save_report
            ])
        }

    async def route(self, task: str) -> str:
        """根据任务类型路由到对应域的 Agent"""
        response = await self.model.generate([{
            "role": "user",
            "content": f"""分析以下任务，确定它属于哪个功能域：
            - data_reading：读取数据
            - data_processing：处理数据
            - data_visualization：可视化数据
            - communication：发送结果

            任务：{task}

            只返回功能域名称，不要其他内容。"""
        }])

        return response.content.strip().lower()

    async def run(self, task: str):
        domain = await self.route(task)
        agent = self.domains.get(domain)

        if agent:
            return await agent.run(task)
        else:
            # 无法路由，使用通用 Agent
            return await self.general_agent.run(task)
```

### 触发条件二：明显独立的任务域

如果两个子任务需要完全不同的专业知识、完全不同的工具集、完全不同的上下文，那拆分几乎是必然的。

一个例子：

- Agent A 负责代码生成：需要代码库的当前上下文、编程规范、技术选型
- Agent B 负责代码审查：需要安全标准、最佳实践、代码质量规范

这两个任务的上下文重叠很少，放在一起反而增加复杂度——模型需要同时考虑"怎么写代码"和"怎么审查代码"，而这两件事的关注点完全不同。

另一个例子是 Manus 的工作流程：研究 Agent 分析任务、决策 Agent 制定计划、执行 Agent 完成操作。每个 Agent 有明确的职责，专注于自己的领域。

## 三种通信模式：深入拆解

Claude Code 支持三种 Agent 执行模型，对应三种不同的通信模式。选择取决于场景对隔离性、效率、上下文共享的需求。

### Fork：完整上下文的复制

Fork 把父 Agent 的完整上下文复制给子 Agent。子 Agent 拿到和父 Agent 完全相同的信息，然后独立探索。

```python
class ForkAgent:
    """
    Fork 模式：子 Agent 获得父 Agent 的完整上下文副本。
    """

    def __init__(self, parent_agent):
        # 复制父 Agent 的完整状态
        self.context_snapshot = parent_agent.get_full_context()

    async def explore(self, task: str):
        """子 Agent 在隔离环境中探索"""
        # 使用复制的上下文
        agent = Agent(model=self.model, context=self.context_snapshot)
        result = await agent.run(task)
        # 压缩结果返回
        return self.compress_result(result)


# 使用示例
async def code_analysis_with_fork(task: str, codebase_context: dict):
    """
    用 Fork 模式做代码分析。
    父 Agent 继续处理其他任务，子 Agent 并行分析代码。
    """
    fork = ForkAgent(parent_agent=current_agent)

    # 启动子 Agent 分析（异步）
    analysis_task = asyncio.create_task(
        fork.explore("分析这个代码库的设计模式和潜在问题")
    )

    # 父 Agent 继续处理其他事情
    other_work = await current_agent.continue_work()

    # 等待分析完成
    analysis_result = await analysis_task

    # 合并结果
    return {
        "other_work": other_work,
        "analysis": analysis_result
    }
```

**代价**：上下文在拆分时一次性复制。如果父 Agent 有大量上下文（长对话历史、多个文件的内容），复制的成本很高。

**收益**：子 Agent 完全独立，可以做任何事情，不受父 Agent 的约束。适合探索性任务。

**适用场景**：代码库分析、大规模研究、并行探索多个方向。

### Teammate：基于文件的邮箱通信

Teammate 模式让子 Agent 运行在独立的终端窗格，通过文件进行通信。父 Agent 写消息到文件，子 Agent 读取并回复。

```python
import asyncio
from pathlib import Path
import json


class TeammateAgent:
    """
    Teammate 模式：基于文件的异步通信。
    """

    def __init__(self, name: str, task_dir: str):
        self.name = name
        self.task_dir = Path(task_dir)
        self.task_dir.mkdir(exist_ok=True)
        self.inbox = self.task_dir / f"{name}_inbox"
        self.outbox = self.task_dir / f"{name}_outbox"
        self.inbox.touch()
        self.outbox.touch()

    async def send_message(self, message: str, priority: str = "normal"):
        """发送消息给子 Agent"""
        msg = {
            "from": "parent",
            "content": message,
            "priority": priority,
            "timestamp": asyncio.get_event_loop().time()
        }
        with open(self.inbox, "a") as f:
            f.write(json.dumps(msg) + "\n")

    async def wait_for_response(self, timeout: float = 300) -> str:
        """等待子 Agent 的响应"""
        start = asyncio.get_event_loop().time()
        last_size = self.outbox.stat().st_size

        while asyncio.get_event_loop().time() - start < timeout:
            await asyncio.sleep(5)

            current_size = self.outbox.stat().st_size
            if current_size > last_size:
                with open(self.outbox) as f:
                    lines = f.readlines()
                    response = json.loads(lines[-1])
                return response["content"]

        return "Timeout: Teammate did not respond"

    async def run(self, initial_task: str):
        """运行子 Agent"""
        # 读取初始任务
        with open(self.inbox) as f:
            for line in f:
                msg = json.loads(line)
                if msg["from"] == "parent":
                    task = msg["content"]
                    break

        # 执行任务（简化版）
        result = await self.model.generate([
            {"role": "system", "content": f"You are {self.name}."},
            {"role": "user", "content": task}
        ])

        # 写入响应
        response = {
            "from": self.name,
            "content": result.content,
            "timestamp": asyncio.get_event_loop().time()
        }
        with open(self.outbox, "a") as f:
            f.write(json.dumps(response) + "\n")

        return result.content


# 使用示例
async def parallel_research(topic: str):
    """
    用 Teammate 模式并行研究多个方向。
    """
    import tempfile
    import shutil

    task_dir = tempfile.mkdtemp()

    # 创建多个研究 Agent
    researcher_a = TeammateAgent("researcher_a", task_dir)
    researcher_b = TeammateAgent("researcher_b", task_dir)

    # 并行启动
    await asyncio.gather(
        researcher_a.send_message(f"研究 {topic} 的技术方面"),
        researcher_b.send_message(f"研究 {topic} 的市场方面")
    )

    # 异步等待响应
    results = await asyncio.gather(
        researcher_a.wait_for_response(),
        researcher_b.wait_for_response()
    )

    # 合并结果
    shutil.rmtree(task_dir)
    return {
        "technical": results[0],
        "market": results[1]
    }
```

**代价**：文件 I/O 比内存共享慢得多。通信协议需要自己设计。没有结构化的消息格式。

**收益**：异步性——子 Agent 可以花很长时间处理复杂任务，父 Agent 不需要等待，可以同时做其他事情。

**适用场景**：需要异步协作、长任务、需要子 Agent 独立运行较长时间。

### Worktree：独立的 git 工作树

Worktree 是隔离性最强的模式。每个 Agent 有自己的 git 分支，可以独立提交代码。不同 Agent 的修改不会相互干扰，最后通过 git merge 合并。

```python
import subprocess
from pathlib import Path


class WorktreeAgent:
    """
    Worktree 模式：每个 Agent 有独立的 git 工作树。
    """

    def __init__(self, name: str, repo_path: str):
        self.name = name
        self.repo = Path(repo_path)
        self.branch_name = f"agent/{name}"

    def setup(self):
        """为 Agent 创建独立的 git worktree"""
        # 创建 worktree（独立分支）
        worktree_path = self.repo / f".worktrees" / self.name

        subprocess.run([
            "git", "worktree", "add",
            str(worktree_path),
            f"HEAD"
        ], cwd=self.repo, check=True)

        # 创建独立分支
        subprocess.run([
            "git", "checkout", "-b", self.branch_name
        ], cwd=worktree_path, check=True)

        return worktree_path

    def execute(self, task: str) -> str:
        """在独立的 worktree 中执行任务"""
        worktree_path = self.repo / ".worktrees" / self.name

        # 模拟在 worktree 中执行任务
        result = subprocess.run(
            ["git", "status"],
            cwd=worktree_path,
            capture_output=True,
            text=True
        )

        return f"Worktree {self.name}: {result.stdout}"

    def commit(self, message: str):
        """提交 Agent 的更改"""
        worktree_path = self.repo / ".worktrees" / self.name

        subprocess.run(["git", "add", "."], cwd=worktree_path)
        subprocess.run(
            ["git", "commit", "-m", f"{self.name}: {message}"],
            cwd=worktree_path
        )

    def merge(self):
        """将 Agent 的更改合并回主分支"""
        main_path = self.repo

        # 切回主分支
        subprocess.run(["git", "checkout", "main"], cwd=main_path)

        # 合并 Agent 的分支
        subprocess.run(
            ["git", "merge", self.branch_name, "--no-ff"],
            cwd=main_path
        )

        # 清理 worktree
        subprocess.run([
            "git", "worktree", "remove",
            str(self.repo / ".worktrees" / self.name)
        ])


# 使用示例
async def parallel_code_modification(tasks: list[str]):
    """
    用 Worktree 模式并行修改代码。
    每个 Agent 在独立的分支上工作，互不干扰。
    """
    repo_path = "/path/to/repo"

    # 为每个任务创建独立的 worktree
    agents = []
    for task in tasks:
        agent = WorktreeAgent(name=f"task_{task}", repo_path=repo_path)
        agent.setup()
        agents.append(agent)

    # 并行执行
    results = await asyncio.gather(*[
        agent.execute(task) for agent, task in zip(agents, tasks)
    ])

    # 提交每个 Agent 的更改
    for agent, task in zip(agents, tasks):
        agent.commit(f"Complete task: {task}")

    # 合并所有更改
    for agent in agents:
        agent.merge()

    return results
```

**代价**：最重的模式。需要 git 基础设施。合并冲突需要手动解决。

**收益**：最强的隔离性。不同 Agent 的修改完全独立，不会相互干扰。适合需要并行处理同一代码库的场景。

**适用场景**：大规模代码重构、多个功能并行开发、需要保持主分支干净的项目。

## 交接协议：Agent 之间的控制权转移

OpenAI 的 SDK 支持两种多 Agent 交互模式，不只是通信，还有控制权的转移。

### Agent 作为工具：临时委托

Agent 作为工具被调用时，用完即弃。父 Agent 决定什么时候调用哪个专家，然后把任务完全委托出去。

```python
# Agent 作为工具的使用方式
coder = Agent(
    name="coder",
    instructions="你是一个专业的 Python 程序员...",
    tools=[edit_file, run_python, run_test]
)

reviewer = Agent(
    name="reviewer",
    instructions="你是一个代码审查专家...",
    tools=[read_file, comment]
)

# 主 Agent 把它们当作工具使用
orchestrator = Agent(
    name="orchestrator",
    instructions="""你是一个任务协调员。
    当需要写代码时，使用 coder 工具。
    当需要审查代码时，使用 reviewer 工具。""",
    tools=[coder.as_tool(), reviewer.as_tool()]
)

result = Runner.run_sync(orchestrator, "修复 auth.py 中的 bug，然后审查")
```

### 交接：完全控制权转移

交接把完全控制权交给专家 Agent。父 Agent 等待结果，不干预执行过程。

```python
# 交接的使用方式
expert = Agent(
    name="data_analyst",
    instructions="""你是一个专业的数据分析师。
    当你收到分析任务时：
    1. 理解数据和问题
    2. 制定分析计划
    3. 执行分析
    4. 生成报告
    你有完全的自主权，不需要汇报进度。"""
)

orchestrator = Agent(
    name="orchestrator",
    instructions="""你是一个任务协调员。
    当遇到需要深度数据分析的任务时，使用 handoff 工具。
    handoff 会把控制权完全交给专家 Agent。""",
    tools=[expert.as_handoff_tool()]
)

result = Runner.run_sync(orchestrator, "分析本季度的销售数据，找出增长点和问题")
```

两种模式的区别：

- **Agent 作为工具**：父 Agent 保留控制权，决定调用顺序和参数
- **交接**：父 Agent 放弃控制权，专家 Agent 自主决策

选择取决于任务的确定性。如果任务有明确的步骤，用 Agent 作为工具。如果任务需要专家自主决策，用交接。

## 嵌套状态图：LangGraph 的多 Agent 实现

LangGraph 将子 Agent 实现为嵌套状态图。每个子 Agent 是父状态图中的一个节点，节点内部有自己的状态流转逻辑。

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict


class ParentState(TypedDict):
    task: str
    research_output: str
    analysis_output: str
    final_output: str


class ResearchState(TypedDict):
    task: str
    findings: list


class AnalysisState(TypedDict):
    research: str
    insights: list


# 子图1：研究 Agent
def research_node(state: ResearchState) -> ResearchState:
    findings = perform_research(state["task"])
    return {"findings": findings}


research_workflow = StateGraph(ResearchState)
research_workflow.add_node("research", research_node)
research_workflow.set_entry_point("research")
research_workflow.add_edge("research", END)
research_agent = research_workflow.compile()


# 子图2：分析 Agent
def analysis_node(state: AnalysisState) -> AnalysisState:
    insights = perform_analysis(state["research"])
    return {"insights": insights}


analysis_workflow = StateGraph(AnalysisState)
analysis_workflow.add_node("analysis", analysis_node)
analysis_workflow.set_entry_point("analysis")
analysis_workflow.add_edge("analysis", END)
analysis_agent = analysis_workflow.compile()


# 父图：协调两个子 Agent
def research_coordinator(state: ParentState) -> ParentState:
    result = research_agent.invoke({"task": state["task"]})
    return {"research_output": result["findings"]}


def analysis_coordinator(state: ParentState) -> ParentState:
    result = analysis_agent.invoke({"research": state["research_output"]})
    return {"analysis_output": result["insights"]}


parent_workflow = StateGraph(ParentState)
parent_workflow.add_node("research_coord", research_coordinator)
parent_workflow.add_node("analysis_coord", analysis_coordinator)
parent_workflow.add_node("synthesize", synthesize_output)

parent_workflow.set_entry_point("research_coord")
parent_workflow.add_edge("research_coord", "analysis_coord")
parent_workflow.add_edge("analysis_coord", "synthesize")
parent_workflow.add_edge("synthesize", END)

parent_app = parent_workflow.compile()
```

嵌套状态图的关键优势是**状态隔离**。每个子 Agent 有自己的状态类型和状态管理，不会和父图的状态混淆。

但也要注意：**嵌套层数不要超过三层**。超过三层后，状态管理的复杂度会超过分层带来的结构清晰度。

## 多 Agent 的验证：额外的复杂性

单 Agent 的验证已经够复杂。多 Agent 的验证是另一个级别。

每个 Agent 可能产生错误。Agent 之间的交接可能丢失信息。多个 Agent 的部分正确性如何组合成整体正确性？

一个实用的策略是：**验证每个 Agent 的局部正确性，加上端到端的行为验证**。

```python
class MultiAgentVerifier:
    """
    多 Agent 系统的验证器。
    """

    def __init__(self):
        self.agent_verifiers = {}  # 每个 Agent 的验证器

    def register_agent_verifier(self, agent_name: str, verifier):
        self.agent_verifiers[agent_name] = verifier

    async def verify_local(self, agent_name: str, task: str, output: str) -> tuple[bool, str]:
        """验证单个 Agent 的输出"""
        verifier = self.agent_verifiers.get(agent_name)
        if not verifier:
            return True, "No verifier registered"

        return await verifier.verify(task, output)

    async def verify_end_to_end(self, task: str, outputs: dict) -> tuple[bool, str]:
        """端到端验证：检查最终结果是否满足原始需求"""
        # 合并所有 Agent 的输出
        combined_output = "\n".join([
            f"=== {name} ===\n{output}"
            for name, output in outputs.items()
        ])

        # 用 LLM 评判最终结果
        judge_prompt = f"""原始任务：{task}

系统输出：
{combined_output}

评估：
1. 任务是否完成？
2. 输出是否一致？有没有矛盾？
3. 是否有遗漏的部分？
"""
        result = await self.llm_judge.verify(judge_prompt)
        return result


# 使用示例
async def run_verified_multi_agent(task: str, agents: dict):
    """
    运行多 Agent 系统并验证结果。
    """
    verifier = MultiAgentVerifier()

    # 为每个 Agent 注册验证器
    for name, agent in agents.items():
        verifier.register_agent_verifier(
            name,
            RuleBasedVerifier()  # 或其他验证器
        )

    # 运行所有 Agent
    outputs = {}
    for name, agent in agents.items():
        outputs[name] = await agent.run(task)

        # 局部验证
        valid, feedback = await verifier.verify_local(name, task, outputs[name])
        if not valid:
            # 局部验证失败，记录但继续
            outputs[name] = f"{outputs[name]}\n\n[验证警告: {feedback}]"

    # 端到端验证
    e2e_valid, e2e_feedback = await verifier.verify_end_to_end(task, outputs)

    return {
        "outputs": outputs,
        "e2e_valid": e2e_valid,
        "e2e_feedback": e2e_feedback
    }
```

分层验证比试图在每一步都做完整验证更 scalable。如果每个 Agent 内部都有完整验证，Agent 之间的交接又有完整验证，系统会变得过于复杂。

## 什么时候多 Agent 是过度设计

最后一个问题：什么时候应该选择多 Agent，但实际不需要？

一个信号是：你在设计多 Agent 协作流程时，脑子里想象的交互图比实际的计算图复杂得多。

如果设计出来的系统比你用单 Agent 实现的等效功能还复杂——更多的代码、更多的通信开销、更多的状态管理——说明拆分变成了过度设计。

另一个信号：你花了更多时间在设计 Agent 之间的通信协议，而不是在解决实际问题。

```python
def detect_over_engineering(single_agent_complexity: float, multi_agent_complexity: float) -> str:
    """
    检测是否过度工程化。
    """
    ratio = multi_agent_complexity / single_agent_complexity

    if ratio > 1.5:
        return "OVER_ENGINEERED: Multi-agent adds >50% complexity. Consider single agent."

    if ratio > 1.2:
        return "MARGINAL: Multi-agent adds significant complexity. Benchmark before committing."

    return "APPROPRIATE: Complexity tradeoff seems reasonable."
```

多 Agent 是手段，不是目的。最终关心的是任务完成的质量和效率，不是系统架构的"优雅程度"。

单 Agent 能做到的事情，不要拆成多 Agent。只有当单 Agent 真的遇到瓶颈（工具过载、上下文爆炸、任务域不兼容），才考虑拆分。

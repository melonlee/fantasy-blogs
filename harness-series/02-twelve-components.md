# Harness 工程实战（二）：生产级 Harness 的十二个组件

---

搭一个能跑起来的 Agent 原型不难。接个 API，写个循环，定义两个工具，差不多就能演示了。

但要从演示走到生产，复杂度会突然跳升。模型会失败，上下文会爆炸，状态会丢失，错误会复合。这些问题每个都能让你的系统在关键时刻瘫痪。

这不是模型的问题，这是基础设施的问题。

综合 Anthropic、OpenAI、LangChain 和更广泛的实践者社区的经验，一个生产级 Agent Harness 有十二个独立组件。下面逐一拆解——不是浮光掠影地过一遍，而是深入到每个组件的职责边界、内部机制和常见实现陷阱。

![Harness 十二组件架构](./images/harness-12-components.jpg)

## 1. 编排循环（Orchestration Loop）

这是心脏，但它不应该有太多"智能"。

Anthropic 把他们的运行时描述为"哑循环"——所有智能都在模型里，Harness 只管理回合。这个设计选择背后有一个信念：模型才是推理的地方，Harness 不要自作聪明。如果 Harness 开始做推理（"用户这句话是什么意思？我应该走哪个分支？"），那这部分推理能力应该交给模型。

一个典型的 ReAct 循环实现：

```python
class AgentLoop:
    def __init__(self, model, tools, max_iterations=50):
        self.model = model
        self.tools = {t.name: t for t in tools}
        self.messages = []
        self.continuing = True
        self.max_iterations = max_iterations

    async def run(self, user_message: str):
        self.messages.append({"role": "user", "content": user_message})

        for i in range(self.max_iterations):
            if not self.continuing:
                break

            # 组装这一轮的完整上下文
            context = self.build_context()

            # 调用模型
            response = await self.model.generate(context)

            # 解析输出：可能是最终回复，也可能是工具调用
            if response.tool_calls:
                for tool_call in response.tool_calls:
                    result = await self.execute_tool(tool_call)
                    self.messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": result
                    })
            else:
                # 没有工具调用，意味着这是最终回复
                return response.content

        return "Maximum iterations reached"

    def build_context(self) -> list:
        # 完整的消息历史，包括系统提示、工具定义、历史对话
        return [
            self.system_prompt,
            *self.tool_definitions,
            *self.messages
        ]
```

关键点：`build_context()` 的实现质量直接影响模型表现。系统提示词怎么写、工具定义放在什么位置、历史对话保留多少——这些都是上下文管理的范畴，不是编排循环单独能解决的。

## 2. 工具（Tools）

工具是 Agent 的手。它让模型能够影响外部世界——读文件、发请求、执行命令、修改数据库。

工具的定义包含三个部分：名称、描述、参数模式。这三者共同决定了模型是否会在正确的时候选择正确的工具。

```python
# 一个典型的工具定义
class ReadFileTool:
    name = "read_file"
    description = """
    Read the contents of a file from the filesystem.
    Returns the full content of the file, or an error message if the file cannot be read.
    Use this when you need to see the contents of a specific file.
    """
    parameters = {
        "type": "object",
        "properties": {
            "path": {
                "type": "string",
                "description": "Absolute path to the file to read"
            },
            "limit": {
                "type": "integer",
                "description": "Maximum number of lines to read (from the start of the file)",
                "default": None
            }
        },
        "required": ["path"]
    }

    async def execute(self, path: str, limit: int = None) -> str:
        try:
            with open(path, "r") as f:
                if limit:
                    return "".join(f.readlines()[:limit])
                return f.read()
        except Exception as e:
            return f"Error reading file: {e}"
```

工具层的职责不仅是"执行"，还包括：
- **模式验证**：确保模型传入的参数符合定义
- **错误处理**：工具可能失败（文件不存在、网络超时），需要捕获并返回有意义的信息
- **结果格式化**：工具输出必须是模型能理解的文本

一个常见错误是让工具返回太多信息。比如 `grep` 的结果可能有几万行，这会把上下文窗口撑爆。好的做法是在工具层做预处理，只返回高信号的结果：

```python
async def execute(self, path: str, pattern: str, context_lines: int = 2) -> str:
    # 搜索之前先检查文件大小
    file_size = os.path.getsize(path)
    if file_size > 10_000_000:  # 超过 10MB，先截断
        return f"File too large ({file_size / 1_000_000:.1f}MB). Use grep on specific sections first."

    result = subprocess.run(
        ["grep", "-n", "-C", str(context_lines), pattern, path],
        capture_output=True, text=True, timeout=30
    )

    # 限制返回行数
    lines = result.stdout.split("\n")
    if len(lines) > 200:
        return "\n".join(lines[:200]) + f"\n... (truncated, {len(lines) - 200} more lines)"

    return result.stdout or "No matches found"
```

## 3. 记忆（Memory）

短期记忆是单个会话内的对话历史。长期记忆跨会话持久化，让 Agent 能记住之前做过什么、用户偏好是什么、项目背景是什么。

Claude Code 实现了三层记忆架构，这个设计值得详细拆解：

**第一层：轻量级索引。** 每条约 150 字符，始终加载在上下文里。这是最浓缩的项目概要：目录结构、关键文件路径、主要技术栈。这层信息的目的是让模型在任何时候都能快速回答"这个项目大概是什么"这类问题。

**第二层：按需拉取的详细文件。** 当模型需要某个文件的详细内容时，加载它，用完可以释放。这层的典型场景是模型在读一个大文件时，只加载它实际需要的部分（通过 `head`/`tail`/`grep`），而不是把整个文件扔进上下文。

**第三层：仅通过搜索访问的原始记录。** 完整的历史记录（之前的工具调用、输出、错误信息），但只通过语义搜索获取。模型不会主动看到这些，除非它主动搜索。

这个设计的核心原则是：**Agent 把自己的记忆当作"提示"，在行动前会对照实际状态验证**。记忆可能过时——比如用户可能在模型不知情的情况下手动改了代码。所以模型在行动前会先检查实际状态，而不是完全依赖记忆中的信息。

一个常见的记忆实现：

```python
class ThreeTierMemory:
    def __init__(self):
        # 第一层：轻量级索引
        self.index = []  # 每次操作后更新，保持在 ~150 chars * 20 条
        # 第二层：按需文件缓存
        self.file_cache = {}  # path -> content
        # 第三层：向量数据库
        self.vector_store = VectorStore()

    async def remember(self, query: str) -> str:
        # 先查索引
        index_results = self.search_index(query)
        # 再查向量存储
        vector_results = await self.vector_store.search(query, top_k=5)
        # 合并，去重
        return self.merge_results(index_results, vector_results)

    async def store_interaction(self, input_text: str, output_text: str):
        # 存入向量数据库
        await self.vector_store.add(input_text, output_text)
        # 更新索引（截断到限制长度）
        summary = summarize(f"Q: {input_text}\nA: {output_text}")
        self.index.append(summary)
        self.trim_index()  # 保持总大小在限制内
```

## 4. 上下文管理（Context Management）

这是许多 Agent 失败的地方，也是最难做对的地方。

核心问题是上下文腐烂（Context Rot）：当上下文窗口里的信息越来越多时，模型对关键信息的关注度会下降。特别是当关键内容落在窗口中间位置时，模型性能下降 30% 以上——这就是斯坦福"Lost in the Middle"研究的核心发现。

Chroma 复现了这个问题。他们的实验显示：当上下文超过一定长度后，模型在中间位置的信息recall率显著下降。即使模型的上下文窗口声称支持 100k token，实际有效的信息密度可能远低于这个数字。

上下文管理不是简单的"放多少内容进去"，而是回答一个更根本的问题：**模型此刻最需要看到什么**。

Claude Code 的做法是：用即时检索替代批量加载。不是把整个代码库塞进上下文，而是用 `grep`、`glob`、`head`、`tail` 这样的工具按需获取信息。

```python
# 不好：一次性加载整个文件
with open("large_codebase.py") as f:
    content = f.read()  # 可能几 MB

# 好：只加载需要的部分
result = subprocess.run(
    ["grep", "-n", "-C", "3", "class Agent", "large_codebase.py"],
    capture_output=True, text=True
)
# 只返回匹配行和上下文，结果通常只有几 KB
```

这个原则不仅适用于文件。API 响应、数据库查询结果、文档内容——都应该遵循"按需获取，按量使用"的原则。

上下文管理的另一个关键决策是什么时候触发压缩。当对话历史接近上下文窗口的 70% 时，应该开始压缩。压缩不是简单地截断，而是用模型生成摘要：

```python
async def compress_history(self, messages: list) -> list:
    # 用模型生成对话摘要
    summary_prompt = f"""请总结以下对话的要点，保留关键决策和技术细节：

{messages[-20:]}"""

    summary = await self.model.generate([
        {"role": "user", "content": summary_prompt}
    ])

    # 替换为摘要版本
    return [
        {"role": "system", "content": f"对话摘要：{summary}"},
        *messages[-5:]  # 保留最近几轮完整对话
    ]
```

## 5. 提示词构建（Prompt Construction）

这是 Harness 中最被低估的部分。很多人的 prompt 写得随意——想到什么写什么，没有结构，没有层次，没有对模型行为的精确引导。

一个好的提示词构建系统是分层的，每一层都有明确的职责：

```python
class PromptBuilder:
    def __init__(self):
        self.system_prompt = """你是一个专业的软件工程师。你的目标是：
1. 理解用户的需求
2. 阅读相关代码和文档
3. 编写高质量的代码
4. 验证你的修改是正确的

重要原则：
- 不要猜测，先探索。你不确定的地方，先读取相关文件再行动。
- 每次修改后验证。运行测试，确保没有破坏现有功能。
- 如果测试失败，分析原因，修复后再次验证。
"""

    def build(self, task: str, memory: str, history: list) -> list:
        messages = [
            {"role": "system", "content": self.system_prompt},
        ]

        # 如果有记忆，加载记忆
        if memory:
            messages.append({
                "role": "system",
                "content": f"项目背景和记忆：\n{memory}"
            })

        # 工具定义
        messages.append({
            "role": "system",
            "content": self.build_tool_prompt()
        })

        # 历史对话
        messages.extend(history)

        # 当前任务
        messages.append({"role": "user", "content": task})

        return messages

    def build_tool_prompt(self) -> str:
        # 工具定义用固定格式，让模型能精确理解每个工具的用途和限制
        tool_docs = []
        for name, tool in self.tools.items():
            tool_docs.append(f"""
## {name}
描述：{tool.description}
参数：
{json.dumps(tool.parameters, indent=2, ensure_ascii=False)}
""")
        return "\n".join(tool_docs)
```

OpenAI 的 Codex 使用严格的优先级栈：服务器控制的系统消息 > 工具定义 > 开发者指令 > 用户指令 > 对话历史。这个顺序不是随意的——越靠近顶部，模型越倾向于遵守。如果把系统提示词放在用户消息之后，模型的遵守率会显著下降。

## 6. 输出解析（Output Parsing）

现代 Harness 依赖模型的原生工具调用能力。模型返回的结构化的 `tool_calls` 对象，而不是自由文本。Harness 需要正确解析这个对象。

```python
class OutputParser:
    def parse(self, response) -> tuple[str, list[str], str]:
        """
        解析模型响应。
        返回: (final_text, tool_calls, continue_turn)
        """
        if hasattr(response, 'tool_calls') and response.tool_calls:
            # 有工具调用
            tool_calls = []
            for tc in response.tool_calls:
                tool_calls.append({
                    "name": tc.function.name,
                    "arguments": json.loads(tc.function.arguments),
                    "id": tc.id
                })
            return "", tool_calls, True
        else:
            # 最终回复
            return response.content, [], False

    def extract_final_answer(self, response) -> str:
        """提取模型认为的最终答案"""
        if hasattr(response, 'content'):
            return response.content
        return str(response)
```

## 7. 状态管理（State Management）

Agent 的状态不只是对话历史，还包括：当前任务进度、工具调用结果、错误计数、已验证的假设、未解决的子问题。

LangGraph 用状态图建模这个问题。每个节点处理状态的一个切片，状态在节点之间流转：

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    current_task: str
    completed_steps: list
    errors: list
    next_action: str

workflow = StateGraph(AgentState)

workflow.add_node("plan", plan_node)
workflow.add_node("execute", execute_node)
workflow.add_node("verify", verify_node)
workflow.add_node("recover", recover_node)

workflow.set_entry_point("plan")
workflow.add_edge("plan", "execute")
workflow.add_edge("execute", "verify")

def route_after_verify(state: AgentState) -> str:
    if state["errors"] and len(state["completed_steps"]) < 3:
        return "recover"
    elif not state["completed_steps"]:
        return "execute"
    else:
        return END

workflow.add_conditional_edges("verify", route_after_verify, {
    "recover": "recover",
    "execute": "execute",
    END: END
})
workflow.add_edge("recover", "execute")

app = workflow.compile()
```

检查点机制让中断恢复成为可能。当 Agent 处理一个长时间任务时，如果进程崩溃，检查点可以让它从上一个成功的节点恢复，而不是从头开始。

## 8. 错误处理（Error Handling）

错误会复合。一个 10 步流程，每步 99% 成功率，端到端只有 90.4%。如果每步只有 95%，端到端只剩 60%。

LangGraph 把错误分为四种，每种有不同的处理策略：

**瞬态错误**：网络超时、临时不可用。策略：带退避的重试。

```python
async def call_with_retry(tool_func, max_retries=3, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return await tool_func()
        except (TimeoutError, ConnectionError) as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)  # 指数退避
            await asyncio.sleep(delay)
```

**LLM 可恢复错误**：模型可以尝试修复的错误。比如工具参数不对，模型知道怎么修正。策略：把错误信息作为 ToolMessage 返回给模型，让它重新尝试。

**用户可修复错误**：需要人类介入的错误。比如权限不足、资源耗尽。策略：中断，等待用户确认。

**意外错误**：程序 bug。策略：冒泡，用于调试和日志记录。

Stripe 的生产 Harness 把重试上限设为两次。超过两次还失败，说明问题不在重试能解决的范围——可能是设计问题，不是临时问题。

## 9. 防护栏和安全（Guardrails and Safety）

防护栏有三个层级：

**输入防护栏**：检查用户输入是否包含恶意内容、是否在系统允许的范围内。一个简单的实现：

```python
class InputGuardrail:
    def __init__(self, allowed_domains: list[str]):
        self.allowed_domains = allowed_domains

    def check(self, user_input: str, context: dict) -> bool:
        # 检查是否包含敏感信息
        if self.contains_secrets(user_input):
            return False
        # 检查目标域名
        if "url" in context:
            if not self.is_domain_allowed(context["url"]):
                return False
        return True
```

**输出防护栏**：模型输出是否符合安全要求、是否泄露敏感信息。

**工具防护栏**：每次工具调用前检查权限。高风险操作（删除文件、发送网络请求、访问敏感 API）需要明确确认。

```python
class ToolGuardrail:
    HIGH_RISK_TOOLS = {"delete_file", "send_email", "exec_command"}

    def check(self, tool_name: str, tool_args: dict) -> bool:
        if tool_name in self.HIGH_RISK_TOOLS:
            # 高风险操作，需要用户确认
            return self.request_confirmation(tool_name, tool_args)
        return True
```

## 10. 验证循环（Verification Loops）

这是区分玩具演示和生产系统的关键。Boris Cherny（Claude Code 的创建者）的结论是：给模型一种验证其工作的方法，质量提高 2 到 3 倍。

三种验证方法的适用场景和代码实现：

```python
# 方法一：基于规则的验证（确定性）
def verify_code_change(file_path: str, expected_pattern: str) -> bool:
    with open(file_path) as f:
        content = f.read()
    return expected_pattern in content

# 方法二：测试驱动验证
def verify_with_tests(test_command: str) -> tuple[bool, str]:
    result = subprocess.run(
        test_command,
        shell=True,
        capture_output=True,
        timeout=120
    )
    return result.returncode == 0, result.stdout + result.stderr

# 方法三：LLM 评判者
async def verify_with_judge(task: str, output: str) -> str:
    judge_prompt = f"""你是一个严格的代码评审专家。评估以下实现是否满足需求。

需求：{task}

实现：{output}

评估维度：
1. 功能是否完整？
2. 是否有边界情况未处理？
3. 代码质量如何？

请给出具体的改进建议。"""

    response = await model.generate([
        {"role": "user", "content": judge_prompt}
    ])
    return response.content
```

## 11. 子智能体编排（Subagent Orchestration）

当任务足够复杂，单个 Agent 无法高效处理时，需要拆分。但拆分不是简单地把任务切成两份让两个 Agent 分别处理——Agent 之间的通信、状态同步、错误处理都更复杂。

Claude Code 支持三种子 Agent 执行模型：

**Fork**：父 Agent 的完整上下文被复制给子 Agent。子 Agent 可以做任何事情，但和父 Agent 完全独立。代价是上下文复制的成本。

**Teammate**：子 Agent 运行在独立的终端窗格，通过文件进行通信。这引入了异步性——子 Agent 处理任务时，父 Agent 可以做其他事情。

**Worktree**：每个 Agent 有独立的 git 工作树。不同 Agent 的修改不会相互干扰，最后通过 git merge 合并。这是隔离性最强的模式，适合需要并行处理同一代码库的场景。

## 12. 工具注册（Tool Registry）

这是让前面所有组件协同工作的基础设施。一个设计良好的工具注册系统需要：

```python
class ToolRegistry:
    def __init__(self):
        self._tools = {}

    def register(self, tool: callable, name: str = None):
        name = name or tool.__name__
        self._tools[name] = tool
        return tool  # 支持装饰器用法

    def get_tools_for_context(self, context: dict, max_tools: int = 10) -> list:
        """根据当前上下文返回最相关的工具"""
        relevant_tools = []
        for name, tool in self._tools.items():
            if self.is_relevant(name, context):
                relevant_tools.append(tool)
                if len(relevant_tools) >= max_tools:
                    break
        return relevant_tools

    def is_relevant(self, tool_name: str, context: dict) -> bool:
        """判断某个工具是否与当前上下文相关"""
        # 简单的关键词匹配，实际可以用更复杂的 embedding 方案
        task_description = context.get("task", "")
        tool_hints = context.get("tool_hints", {})
        return any(hint in tool_name.lower() for hint in tool_hints.get(task_description, []))
```

关键原则：暴露当前步骤所需的最小工具集，而不是所有工具全集。Vercel 从 v0 中删除了 80% 的工具，Claude Code 通过延迟加载实现了 95% 的上下文减少——这些都说明"少即是多"。

## 这十二个组件不是孤立的

看完这十二个组件，一个重要的事实是：它们是一个协同系统，不是可以独立替换的模块。

编排循环依赖工具注册来知道有哪些工具可用。工具执行后，结果通过输出解析进入消息历史。消息历史是上下文管理的输入。上下文管理决定了下一次调用模型时放什么进去。验证循环检查工具输出的质量，如果失败触发错误处理，错误处理决定是否重试、重试多少次。子智能体编排需要协调多个 Agent 的状态，这又回到状态管理。

任何一个环节出问题，整个系统都会受影响。这也是为什么 Harness 工程不是简单的技术堆砌——它是系统设计，需要理解各个组件之间的依赖关系和交互协议。

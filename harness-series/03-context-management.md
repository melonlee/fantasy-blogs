# Harness 工程实战（三）：上下文管理——最容易被忽视的瓶颈

---

大多数人在评估 AI 系统时，注意力都集中在模型能力上。模型能做什么？它有多聪明？参数有多少？

但实际在生产环境中跑过复杂 Agent 系统的人都知道，真正的瓶颈往往不在模型，而在上下文管理。

你花了大价钱请了顶级专家，结果他一进门就忘了你三句话之前说的话。这不是他不聪明，是你没给他一个好的工作环境——你把他扔进了一个堆满杂物的房间，让他在一堆废纸里找重要文件。

上下文管理，就是给模型建立那个工作环境的技术。

## Lost in the Middle：一个被低估的根本问题

斯坦福的研究者做过一个实验：把关键信息放在上下文窗口的不同位置，然后测试模型能否正确回忆和使用这些信息。

结果很残酷：当关键信息落在上下文窗口的中间位置时，模型性能下降 30% 以上。不是 10%，不是 15%，是 30%。

这个现象被称为"Lost in the Middle"——模型对开头和结尾的信息关注度最高，中间的信息容易被忽略。这不是模型的 bug，这是 attention 机制的天然特性。Transformer 的 attention 在数学上就天然对两端加权更高，中间被稀释。

Chroma 的团队后来在他们自己的 Agent 系统里复现了这个问题。他们发现当上下文超过一定长度后，系统表现急剧下降，即使这个长度远未达到模型的官方上下文窗口限制。

一个百万 token 的上下文窗口听起来很大，但如果设计不当，实际有效的信息密度可能远低于预期。就像一个仓库堆满了杂物，真正需要的东西反而被埋在了中间。

![上下文窗口的 Lost in the Middle 问题](./images/harness-context-window.jpg)

## 五种上下文管理策略：深入拆解

### 策略一：基于时间的清除

最简单粗暴的策略。当对话超过一定时间或轮次，直接清除早期内容，保留最近的对话。

实现很简单：

```python
class TimeBasedEviction:
    def __init__(self, max_turns: int = 20, max_age_seconds: int = 3600):
        self.max_turns = max_turns
        self.max_age_seconds = max_age_seconds
        self.messages = []

    def add(self, message: dict):
        self.messages.append({
            **message,
            "timestamp": time.time()
        })
        self._evict()

    def _evict(self):
        now = time.time()
        # 超过时间或超过轮次就清除
        self.messages = [
            m for m in self.messages
            if now - m["timestamp"] < self.max_age_seconds
            and len([x for x in self.messages if x["timestamp"] <= m["timestamp"]]) <= self.max_turns
        ]
```

优点是实现简单，缺点是可能误删重要上下文。如果用户在第 3 轮提了一个关键的设计约束（比如"这个项目不能用任何外部数据库"），到第 20 轮时这条信息早就被清除了——而模型可能正在生成一个需要连接数据库的方案。

这种策略适合：对话是独立的短任务、上下文质量不随时间衰减、误删的代价可接受。

### 策略二：对话总结

更智能的做法：当上下文接近限制时，对历史对话进行压缩总结，保留"大意"而非细节。

```python
class SummarizingMemory:
    def __init__(self, model, max_tokens: int, summary_trigger_ratio: float = 0.7):
        self.model = model
        self.max_tokens = max_tokens
        self.summary_trigger_ratio = summary_trigger_ratio
        self.messages = []

    def add(self, message: dict):
        self.messages.append(message)

        # 检查是否需要压缩
        if self._estimated_tokens() > self.max_tokens * self.summary_trigger_ratio:
            self._compress()

    def _estimated_tokens(self) -> int:
        # 粗略估算：每 4 个字符约等于 1 个 token
        return sum(len(m.get("content", "")) for m in self.messages) // 4

    def _compress(self):
        # 用模型生成摘要
        conversation_for_summary = "\n".join([
            f"{m['role']}: {m['content']}"
            for m in self.messages[-50:]  # 最近 50 轮
        ])

        summary_prompt = f"""请总结以下对话的要点，保留关键决策、技术约束、用户偏好和未完成的事项。

对话：
{conversation_for_summary}

输出格式：
## 摘要
（3-5 句话概括主要内容和进展）

## 关键决策
- （列出所有关键技术决策）

## 技术约束
- （列出所有约束条件，如"不使用外部数据库"）

## 未完成事项
- （列出尚未完成的任务）

## 重要偏好
- （列出用户的偏好，如"偏好 TypeScript"）
"""

        response = self.model.generate([
            {"role": "user", "content": summary_prompt}
        ])

        summary_text = response.content if hasattr(response, 'content') else str(response)

        # 替换为摘要版本
        self.messages = [
            {"role": "system", "content": f"对话摘要：\n{summary_text}"}
        ] + self.messages[-5:]  # 保留最近 5 轮完整对话
```

Claude Code 在这方面做得很细致。它保留架构决策和未解决的 bug，丢弃冗余的工具输出。工具调用产生的原始数据量很大——一个 `ls -la` 的输出可能有几千字节——但真正有价值的是"列出了这些文件"这个结论，不是输出本身。

对话总结适合：长对话（超过 20 轮）、对话内容有明确的主题和进展、摘要误删的风险可接受。

### 策略三：观察遮蔽（Observation Masking）

JetBrains 的 Junie 采用了另一种思路：隐藏旧的工具输出，但保持工具调用本身可见。

这背后的洞察是：模型需要知道"做过什么"，但不一定需要"怎么做的"具体细节。

就像你看一份工作简历，你想知道候选人做过哪些项目（工具调用），不需要知道他每天几点下班（具体输出）。

```python
class ObservationMasking:
    """
    保留工具调用的结构，但隐藏具体的观察结果。
    """

    ROLE_TO_COLORS = {
        "user": "blue",
        "assistant": "green",
        "tool": "yellow",
    }

    def mask(self, messages: list) -> list:
        """将消息列表中的工具输出替换为简短摘要"""
        result = []
        for msg in messages:
            if msg.get("role") == "tool":
                # 用简短摘要替换完整输出
                result.append({
                    "role": "tool",
                    "tool_call_id": msg.get("tool_call_id"),
                    "content": self._summarize_output(msg.get("content", ""))
                })
            else:
                result.append(msg)
        return result

    def _summarize_output(self, content: str, max_length: int = 100) -> str:
        """把长输出压缩成一句话摘要"""
        if len(content) <= max_length:
            return content

        # 常见模式的简单摘要
        lines = content.split("\n")
        if len(lines) > 5:
            return f"{lines[0][:60]}... ({len(lines)} lines total)"
        return content[:max_length] + "..."
```

但这种方法也有问题。有些任务需要看具体的工具输出才能判断——比如"检查代码风格是否有问题"，你需要看到具体的 lint 错误信息。所以观察遮蔽不是万能的，只适合不需要详细回看的任务。

观察遮蔽适合：工具输出是过程性的（走了哪些步骤）、不需要回看具体数值、摘要足够还原上下文。

### 策略四：即时检索（Just-in-time Retrieval）

维护轻量级的标识符，而不是加载完整数据。动态查询，按需获取。

Claude Code 使用 `grep`、`glob`、`head`、`tail` 而不是加载完整文件。这是关键的设计决策——不是技术限制，是上下文压力的取舍。

```python
# 不是这样
with open("large_codebase.py") as f:
    content = f.read()  # 可能 10MB，全部加载

# 而是这样
result = subprocess.run(
    ["grep", "-n", "-C", "3", "class Agent", "large_codebase.py"],
    capture_output=True, text=True
)
# 只加载匹配行和上下文，结果通常只有几 KB

# 或者这样：只读前 100 行了解结构
result = subprocess.run(
    ["head", "-n", "100", "large_codebase.py"],
    capture_output=True, text=True
)
```

但即时检索也有陷阱。最大的陷阱是：检索本身可能是错的。

```python
def read_file_section(path: str, start_line: int, end_line: int) -> str:
    """
    读取文件的指定行范围。
    """
    try:
        with open(path) as f:
            lines = f.readlines()
            start = max(0, start_line - 1)
            end = min(len(lines), end_line)
            return "".join(lines[start:end])
    except Exception as e:
        return f"Error reading {path}: {e}"


def find_pattern_in_file(path: str, pattern: str, context_lines: int = 3) -> str:
    """
    在文件中搜索模式，返回匹配行和上下文。
    自动处理大文件。
    """
    # 先检查文件大小
    file_size = os.path.getsize(path)
    if file_size > 50_000_000:  # 50MB
        # 文件太大，用 grep 限制返回行数
        result = subprocess.run(
            ["grep", "-n", "-C", str(context_lines),
             "--", pattern, path],
            capture_output=True, text=True, timeout=30
        )
        if result.stdout.count("\n") > 500:
            # 匹配太多，限制范围
            return "(Too many matches. Consider narrowing your search.)"
        return result.stdout or "(No matches found)"
    else:
        # 文件不大，直接读
        return read_file_section(path, 1, 1000)  # 先读前 1000 行
```

即时检索适合：处理大文件、只需要部分信息、检索本身是快速的场景。

### 策略五：子智能体委托（Subagent Delegation）

每个子智能体广泛探索，但只返回压缩摘要。

这个策略解决的是：如果让一个 Agent 去探索一个很大的知识库，全量返回所有细节会把主上下文撑爆。分而治之——子 Agent 做深度探索，主 Agent 只接收摘要。

```python
async def explore_with_subagent(
    task: str,
    exploration_prompt: str,
    max_results: int = 10
) -> str:
    """
    用子 Agent 探索大量信息，返回压缩摘要。
    """
    subagent = Agent(
        model=self.model,
        tools=[search_tool, read_file_tool, grep_tool],
        system=f"""你是一个研究助手。你的任务是：
1. 探索 {task} 相关的资料
2. 深入分析找到的关键信息
3. 用简洁的语言总结你的发现

重要原则：
- 不要返回原始数据的堆砌，要返回有结构的分析
- 摘要不超过 2000 tokens
- 要包含具体的例子和细节，而不是泛泛而谈
- 如果某个方向特别有价值，深入挖掘
"""
    )

    results = await subagent.run(exploration_prompt)

    # 子 Agent 已经做了压缩，这里再做一次确保
    if len(results) > 2000:
        summary = await self.model.generate([
            {"role": "user", "content": f"将以下内容压缩到 1500 字以内，保留关键细节：\n\n{results}"}
        ])
        return summary.content

    return results
```

子智能体委托适合：探索性任务（研究、调查、代码库分析）、有明确的信息边界、子 Agent 能独立完成的工作。

## ACON 的发现：推理轨迹比原始数据更有价值

2025 年的 ACON 研究提供了一个重要的定量结论：通过优先考虑推理轨迹而不是原始工具输出，token 减少 26% 到 54%，同时保持 95% 以上的准确性。

这个发现很反直觉。按照直觉，会觉得保留越多的工具输出越好——越多的原始数据，模型应该判断越准确。

但实际上，模型的推理过程（它是怎么想的、为什么选择这个方法、下一步打算做什么）比原始数据（工具返回了什么）更有价值。

这就像面试一个人，你更想知道他是怎么思考问题的，而不是他背出来的那些事实。事实可以查，思维方式才决定了他能不能解决新问题。

在实现上，这意味着：

```python
def should_keep_tool_output(tool_name: str, output: str, is_final: bool) -> bool:
    """
    决定是否保留工具的原始输出。
    """
    # 推理工具的输出更重要
    if tool_name in {"think", "planner", "analyzer"}:
        return True

    # 中间步骤的输出，如果已经有后续步骤验证，可以丢弃
    if not is_final:
        # 保留摘要，不保留完整输出
        return len(output) < 500

    # 最终答案相关的大输出才保留
    return True
```

## 上下文工程的目标

Anthropic 的上下文工程指南说了一句话，是整个领域的核心原则：

> 找到最小的高信号 token 集合，最大化期望结果的可能性。

不是上下文越多越好。上下文太多，噪声淹没信号，模型反而更容易迷失。好的上下文管理不是塞入更多信息，是剔除无关信息。

这个原则的具体实践：

1. **宁缺毋滥**：每条放进上下文的信息，都应该能回答一个具体的问题。如果放进去之后模型不会用到，就别放。

2. **信号 vs 噪声**：评估上下文质量的标准是信号密度。100 行高相关的代码片段比 10000 行低相关的代码更有价值。

3. **时机**：模型在不同阶段需要不同的上下文。规划阶段需要概要信息，执行阶段需要详细规范，验证阶段需要测试结果。按需加载，而不是一次性全部塞进去。

## 实际工程中的陷阱

### 陷阱一：过度准备上下文

工程师的直觉是准备越充分，模型表现越好。所以把能想到的所有上下文都塞进去：项目背景、代码结构、历史讨论、技术选型理由、相关文档……

结果上下文窗口爆炸，模型在大量无关信息中迷失，真正重要的设计约束反而落在窗口中部被忽略了。

一个具体例子：用户问"这个函数为什么报错"，模型把整个项目结构、100 个不相关的文件、最近 50 轮对话全部塞进上下文。函数本身的代码和错误信息反而被稀释了。

修复方法是"先问再取"：先让模型判断需要什么信息，然后按需获取。

```python
async def context_aware_execution(task: str):
    # 第一步：让模型分析需要什么
    analysis = await model.generate([
        {"role": "user", "content": f"分析以下任务，确定需要哪些上下文信息：\n\n{task}\n\n需要的信息类型：1) 代码文件 2) 配置 3) 历史对话 4) 其他"}
    ])

    # 第二步：根据分析结果，按需获取
    needed_info = parse_info_requirements(analysis)
    context = await load_context(needed_info)

    # 第三步：执行
    return await model.generate(context + [task])
```

### 陷阱二：上下文状态不一致

Agent 的记忆说项目用的 PostgreSQL，但实际代码里配的是 MySQL。这种不一致会导致模型基于错误信息做决策。

Claude Code 的设计原则是：Agent 把自己的记忆当作"提示"，在行动前会对照实际状态验证。

```python
async def verify_context_before_action(claimed_facts: dict, actual_state: dict):
    """验证上下文claimed和实际状态是否一致"""
    inconsistencies = []
    for key, claimed_value in claimed_facts.items():
        actual_value = actual_state.get(key)
        if claimed_value != actual_value:
            inconsistencies.append({
                "fact": key,
                "claimed": claimed_value,
                "actual": actual_value
            })

    if inconsistencies:
        # 提醒模型修正记忆
        return {
            "verified": False,
            "inconsistencies": inconsistencies,
            "message": "发现上下文与实际状态不一致，请修正后再行动"
        }

    return {"verified": True}
```

### 陷阱三：压缩后丢失关键细节

对话总结时，模型可能把关键的技术细节当成"不重要的上下文"删掉了。比如用户说"这个项目有个特殊的历史包袱，第 47 行的那个 workaround 是十年前加的，不要动"，但总结模型可能把这个信息当成历史细节忽略了。

缓解方法是在总结时指定必须保留的信息类型：

```python
SUMMARY_PROMPT_TEMPLATE = """请总结以下对话的要点。

## 必须保留的信息
- 所有技术约束（如"不要使用外部数据库"）
- 所有已知的 workaround 和历史包袱
- 所有用户的明确偏好
- 所有未完成的子任务

## 对话内容
{conversation}

## 输出格式
### 关键约束
（列表形式，每条一行）

### 未完成事项
（列表形式）

### 摘要
（3-5 句话）
"""
```

## 三层记忆架构的深层设计

Claude Code 的三层层次结构值得再深入拆解一下，因为它展示了上下文管理的最佳实践：

**第一层：轻量级索引。** 每条约 150 字符，始终加载。这个层的设计目标是回答"这个项目是什么"这类元问题。

索引的维护策略：每次重大操作后更新（比如切换了目录、修改了文件、重构了模块）。索引内容要高度结构化，让模型能快速定位：

```python
PROJECT_INDEX_TEMPLATE = """
项目：{project_name}
技术栈：{tech_stack}
关键目录：{key_directories}
主要入口：{entry_points}
上次活动：{last_activity}
"""
```

**第二层：按需拉取的详细文件。** 当模型需要某个文件的详细内容时，主动加载，用完可以释放。关键是控制加载粒度——不是整个文件，而是文件的相关部分。

**第三层：仅通过搜索访问的原始记录。** 完整的历史记录，但只通过语义搜索获取，不主动加载。这层的存在让模型能回溯历史，但不会让历史数据污染当前上下文。

这个设计的核心洞察是：**信息的"在场感"和"可访问性"是不同的**。模型不需要所有信息都在上下文里，但需要知道如何找到它们。

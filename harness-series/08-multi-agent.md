# Harness 工程实战（八）：多智能体系统的工程实践——何时拆、怎么拆、如何通信

---

大多数时候，单个 Agent 够用。

Anthropic 和 OpenAI 都说过同样的话：首先最大化单个 Agent。

这句话重要到值得重复：先让单个 Agent 做他能做的一切，只有在明确需要时才拆成多个。

因为多 Agent 系统有显著的开销。

![多智能体协作与通信](./images/harness-multi-agent.jpg)

## 拆分的多重代价

多 Agent 不是"一个 Agent 做不到的，两个 Agent 加起来就能做到"。

每次从一个 Agent 交接给另一个，你都在损失上下文。两个 Agent 需要通信，需要额外的 LLM 调用来做路由，需要管理状态同步。

LLMCompiler 报告比顺序 ReAct 快 3.6 倍，靠的是把规划和执行分离。但这种加速的前提是规划的开销比分步执行的总和要小。

多 Agent 也是同样的道理：只有在任务足够复杂，拆分后的收益超过通信开销时，拆分才有意义。

## 什么时候应该拆

两个明确的触发条件。

**第一个：工具过载。**

当一个 Agent 需要使用大约超过 10 个重叠工具时，模型开始混淆。工具越多，"我应该用哪个"这个问题就越消耗模型的推理能力。

**第二个：明显独立的任务域。**

如果两个子任务需要完全不同的专业知识、完全不同的工具集，拆分几乎是必然的。

## 三种通信模式

Claude Code 支持三种执行模型：

**Fork：父上下文的字节相同副本。**

子 Agent 拿到和父 Agent 完全相同的上下文，然后独立探索。

```python
sub_agent = fork_agent(parent_agent)
sub_agent.execute("Explore the codebase for similar patterns")
results = sub_agent.compress_results()  # 返回 1000-2000 token 摘要
```

**Teammate：基于文件的邮箱通信。**

子 Agent 运行在独立的终端窗格，通过文件进行通信。父 Agent 写消息到文件，子 Agent 读取并回复。

```python
parent.send_message(task, to="researcher")
while not parent.has_messages():
    do_other_work()
result = parent.receive_message(from_="researcher")
```

**Worktree：独立的 git 工作树。**

每个 Agent 有自己的 git 分支，可以独立提交代码。这是提供了最强隔离性的最重模式。

## 交接协议

OpenAI 的 SDK 支持两种多 Agent 交互模式。

**Agent 作为工具：** 专家 Agent 被当作工具调用，用完即弃。

**交接：** 父 Agent 把完全控制权交给专家 Agent，自己等待结果。

```python
coder = Agent(name="coder", tools=[edit_file, run_test])
reviewer = Agent(name="reviewer", tools=[read_file, comment])

code_result = coder.execute(task)
review_result = reviewer.execute(code_result)
```

## 嵌套状态图

LangGraph 将子 Agent 实现为嵌套状态图。每个子 Agent 是父状态图中的一个节点，节点内部有自己的状态流转逻辑。

嵌套层数不要超过三层。超过三层后，状态管理的复杂度会超过分层带来的结构清晰度。

## 多 Agent 的验证

单 Agent 的验证已经够复杂。多 Agent 的验证是另一个级别。

一个实用的策略是：验证每个 Agent 的局部正确性，加上端到端的行为验证。

分层验证比试图在每一步都做完整验证更 scalable。

## 什么时候多 Agent 是过度设计

一个信号是：你在设计多 Agent 协作流程时，脑子里想象的交互图比实际的计算图复杂得多。

如果设计出来的系统比你用单 Agent 实现的等效功能还复杂，说明拆分变成了过度设计。

多 Agent 是手段，不是目的。最终关心的是任务完成的质量和效率，不是系统架构的"优雅程度"。

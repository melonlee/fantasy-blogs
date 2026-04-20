# Harness 工程实战（四）：验证循环——从玩具演示到生产系统的距离

---

假设你有一个 10 步的 Agent 流程。每一步的成功率是 99%。

听起来很高对吧？

算一下：0.99 的 10 次方，等于 0.904。端到端成功率只有 90.4%。

每一步都接近完美，但整个系统在十分之一的请求上会失败。

这还是乐观估计。很多 Agent 流程不止 10 步，而且有些步骤之间的依赖关系会让失败更难恢复。错误会快速复合，这是 Agent 系统和传统软件最大的区别之一。

验证循环解决的就是这个问题。

![验证循环与自我纠错](./images/harness-verification-loop.jpg)

## 三种验证方法

**第一种：基于规则的反馈。**

最直接：用测试、linter、类型检查器来验证输出。

如果 Agent 说"我已经修复了这个 bug"，你就运行测试。如果测试通过，修复是真的。如果测试失败，Agent 需要再试一次。

这是确定性验证。答案只有对或错，不存在灰色地带。

```python
result = agent.execute("Fix the bug in auth.py")
test_result = subprocess.run(
    ["pytest", "tests/test_auth.py"],
    capture_output=True
)
if test_result.returncode != 0:
    agent.retry("Tests failed. Please fix again.")
```

适合场景：代码生成、格式化任务、任何有明确对错标准的输出。

**第二种：视觉反馈。**

针对 UI 任务的特殊方法：通过 Playwright 截图，让模型看到实际渲染效果。

Agent 修改了一个按钮颜色对不对？截图一看就知道。

```python
from playwright.sync_api import sync_playwright

def verify_ui_change(agent_output):
    with sync_playwright() as p:
        browser = p.chromium.launch()
        page = browser.new_page()
        page.goto("http://localhost:3000/dashboard")
        page.screenshot(path="/tmp/ui_check.png")
        browser.close()
    return agent.evaluate_image("/tmp/ui_check.png")
```

**第三种：LLM 作为评判者。**

用一个单独的子 Agent 来评估输出质量。

Boris Cherny（Claude Code 的创建者）指出：给模型一种验证其工作的方法，质量提高 2 到 3 倍。

```python
judge = Agent(model="claude-sonnet-4",
              system="You are a code reviewer. "
                     "Check if the output satisfies the user's request.")

def verify_with_judge(task, output):
    result = judge.execute(f"Task: {task}\n\nOutput: {output}")
    if "needs revision" in result.lower():
        agent.retry(result)
```

## 前馈与传感器

**前馈是在行动前引导。** 给 Agent 明确的验证标准，让它知道什么样的输出算合格。

**传感器是在行动后观察。** 运行测试、截图、LLM 评判——这些都是在输出产生之后才发生的验证。

两者的区别是：前馈减少失败，传感器发现失败。一个好的验证系统需要两者结合。

## 验证循环的位置

更更好的设计是每一步都验证，而不是只在最后验证。

Agent 说"我已经读取了配置文件"——验证一下它读到的配置内容是否合理。每一步的验证成本更低，因为失败影响范围更小。

## 验证的代价

验证不是免费的。

一个粗略的原则：验证成本应该显著低于任务本身的成本。如果验证成本变成执行成本的两倍，你可能需要重新考虑验证策略。

这也是为什么不同类型的任务需要不同的验证深度。一个每天跑一次的后台任务，可以做深度验证。一个实时交互的聊天机器人，验证就必须轻量。

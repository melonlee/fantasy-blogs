# Harness 工程实战（四）：验证循环——从玩具演示到生产系统的距离

---

假设你有一个 10 步的 Agent 流程。每一步的成功率是 99%。

听起来很高对吧？

算一下：0.99 的 10 次方，等于 0.904。端到端成功率只有 90.4%。

每一步都接近完美，但整个系统在十分之一的请求上会失败。

这还是乐观估计。很多 Agent 流程不止 10 步，而且有些步骤之间的依赖关系会让失败更难恢复——如果第 3 步失败了，第 4-10 步可能建立在错误的基础上，需要全部返工。

错误会快速复合。这是 Agent 系统和传统软件最大的区别之一。

在传统软件里，你写一个函数，99% 的情况正确就够好了。在 Agent 系统里，10 个这样的"够好"叠加在一起，你就有一个经常出故障的系统。

传统软件的确定性逻辑可以依赖测试覆盖率达到 100%。但 Agent 的输出是概率性的——上一秒的正确输出，下一秒可能因为上下文微小变化而不同。

验证循环解决的就是这个问题。

![验证循环与自我纠错](./images/harness-verification-loop.jpg)

## 验证的本质：对抗复合错误

复合错误的问题不是"会不会发生"，而是"发生后能不能发现"。

如果 Agent 在第 3 步产生了一个错误的输出，但第 4-10 步都基于这个错误继续执行，最终会得到一个看起来合理但完全错误的结果。模型不会主动说"我之前算错了"——它会继续往前走，把错误合理化，直到最后给出一个错误但自信的答案。

这就是为什么验证不能只在最后做。如果只在最后验证，你可能发现整个系统需要从头返工。但如果每一步都验证，错误一发生就能发现，成本最低。

## 三种验证方法：适用场景与完整代码

### 方法一：基于规则的验证（Rule-based Verification）

最直接的方法：用确定性规则验证输出。测试、linter、类型检查器、schema 验证——这些都是基于规则的验证。

适合场景：输出有明确对错标准的情况。代码是否编译、测试是否通过、JSON 格式是否正确、计算结果是否匹配预期值。

优点是验证成本极低——测试可以在毫秒级完成，不需要额外调用模型。缺点是只能验证能用规则定义的内容，无法验证语义正确性。

```python
import subprocess
import re
import json
from pathlib import Path


class RuleBasedVerifier:
    """
    基于规则的验证器。适用于代码、配置、格式等有明确对错标准的输出。
    """

    def verify_code_compilation(self, file_path: str, language: str) -> tuple[bool, str]:
        """验证代码是否能编译"""
        if language == "python":
            result = subprocess.run(
                ["python", "-m", "py_compile", file_path],
                capture_output=True, text=True, timeout=30
            )
            return result.returncode == 0, result.stderr or "OK"
        elif language == "javascript":
            result = subprocess.run(
                ["node", "--check", file_path],
                capture_output=True, text=True, timeout=30
            )
            return result.returncode == 0, result.stderr or "OK"
        elif language == "rust":
            result = subprocess.run(
                ["rustc", "--crate-type", "lib", file_path],
                capture_output=True, text=True, timeout=60
            )
            return result.returncode == 0, result.stderr or "OK"
        return False, f"Unsupported language: {language}"

    def verify_json_schema(self, json_text: str, schema: dict) -> tuple[bool, str]:
        """验证 JSON 是否符合 schema"""
        try:
            data = json.loads(json_text)
            # 简单的 schema 验证（生产环境建议用 jsonschema 库）
            for required_key in schema.get("required", []):
                if required_key not in data:
                    return False, f"Missing required key: {required_key}"
            return True, "OK"
        except json.JSONDecodeError as e:
            return False, f"Invalid JSON: {e}"

    def verify_file_content(self, file_path: str, required_patterns: list[str]) -> tuple[bool, str]:
        """验证文件内容是否包含必需的 pattern"""
        try:
            with open(file_path) as f:
                content = f.read()
            missing = [p for p in required_patterns if p not in content]
            if missing:
                return False, f"Missing patterns: {missing}"
            return True, "OK"
        except Exception as e:
            return False, f"Error reading file: {e}"

    def verify_directory_structure(self, root: str, required_paths: list[str]) -> tuple[bool, str]:
        """验证目录结构是否符合预期"""
        missing = []
        for path in required_paths:
            full_path = Path(root) / path
            if not full_path.exists():
                missing.append(path)
        if missing:
            return False, f"Missing paths: {missing}"
        return True, "OK"


def agent_task_with_verification(agent, task: str, output_path: str):
    """
    带验证的 Agent 任务执行。
    """
    # 执行任务
    result = agent.execute(task)

    # 写输出
    with open(output_path, "w") as f:
        f.write(result)

    # 验证
    verifier = RuleBasedVerifier()

    # 1. 代码能编译
    if output_path.endswith(".py"):
        valid, msg = verifier.verify_code_compilation(output_path, "python")
        if not valid:
            return {"success": False, "error": f"Compilation failed: {msg}"}

    # 2. 文件内容包含必要 pattern
    if "function" in task.lower():
        valid, msg = verifier.verify_file_content(output_path, ["def ", "return"])
        if not valid:
            return {"success": False, "error": f"Missing required patterns: {msg}"}

    return {"success": True, "output": result}
```

### 方法二：视觉反馈验证（Visual Verification）

针对 UI 任务的特殊方法：通过截图让模型看到实际渲染效果。

适合场景：前端开发、UI 修改、文档格式化。任何需要"看起来对不对"的任务。

```python
from playwright.sync_api import sync_playwright
from pathlib import Path
import asyncio


class VisualVerifier:
    """
    通过截图进行视觉验证。适用于 UI 相关的任务。
    """

    def __init__(self, base_url: str = "http://localhost:3000"):
        self.base_url = base_url
        self.screenshot_dir = Path("/tmp/visual_verification")
        self.screenshot_dir.mkdir(exist_ok=True)

    def capture_screenshot(self, path: str, name: str) -> str:
        """截取指定页面的截图"""
        screenshot_path = self.screenshot_dir / f"{name}.png"

        with sync_playwright() as p:
            browser = p.chromium.launch()
            page = browser.new_page(viewport={"width": 1280, "height": 720})
            page.goto(f"{self.base_url}{path}")
            page.wait_for_load_state("networkidle")
            page.screenshot(path=str(screenshot_path), full_page=False)
            browser.close()

        return str(screenshot_path)

    def verify_ui_change(self, agent_output: dict, task: str) -> tuple[bool, str]:
        """
        验证 UI 修改是否正确。
        agent_output 包含修改后的页面路径和预期变化描述。
        """
        path = agent_output.get("path", "/")
        expected_changes = agent_output.get("expected_changes", [])
        screenshot_path = self.capture_screenshot(path, f"verify_{int(time.time())}")

        # 让模型评估截图
        evaluation = self.model.evaluate_image(
            screenshot_path,
            task=f"""这个截图展示了 UI 的一部分。
验证以下修改是否正确实现：
{chr(10).join(f'- {c}' for c in expected_changes)}

请指出：1) 修改是否正确实现 2) 是否有新引入的问题 3) 总体评价（1-5分）"""
        )

        # 解析评估结果
        if "1" in evaluation or "2" in evaluation:
            return False, f"UI 验证失败: {evaluation}"
        return True, evaluation


async def web_agent_with_visual_verification(agent, task: str):
    """
    带视觉验证的 Web 开发 Agent。
    """
    # 执行 UI 修改
    result = agent.execute(task)

    # 启动本地服务器（假设已经有）
    # 检查修改是否正确
    verifier = VisualVerifier(base_url="http://localhost:3000")

    # 如果任务涉及 UI 修改，进行视觉验证
    if any(keyword in task.lower() for keyword in ["ui", "button", "color", "layout", "页面", "按钮", "颜色"]):
        valid, feedback = verifier.verify_ui_change(result, task)
        if not valid:
            # 视觉验证失败，让 Agent 修复
            agent.retry(f"视觉验证失败，请修复以下问题：{feedback}")

    return result
```

### 方法三：LLM 评判者（LLM Judge Verification）

用一个单独的 Agent 来评估输出质量。

适合场景：语义正确性、内容质量、创意输出的评估。任何规则难以覆盖的领域。

Boris Cherny（Claude Code 的创建者）的结论是：给模型一种验证其工作的方法，质量提高 2 到 3 倍。这个结论来自他的实际观察：当 Agent 有能力自我检查时，它的表现显著提升。

```python
class LLMJudgeVerifier:
    """
    用 LLM 作为评判者验证输出质量。
    """

    def __init__(self, judge_model):
        self.judge = judge_model
        self.judge_system_prompt = """你是一个严格的代码评审专家和质量管理专家。你的职责是评估 AI Agent 的输出是否满足用户需求。

评审维度：
1. 功能完整性：输出是否解决了用户的问题？
2. 技术正确性：代码是否正确、是否符合最佳实践？
3. 边界处理：是否考虑了边界情况和错误输入？
4. 可维护性：代码是否清晰、易于理解和修改？
5. 安全性：是否有潜在的安全问题？

输出格式：
PASS: [简短说明] — 输出满足要求
FAIL: [具体问题列表] — 输出存在问题，需要修改
PARTIAL: [部分满足的地方] — 部分满足要求

如果 FAIL，请给出具体的修改建议。
"""

    async def verify_code(self, task: str, code_output: str) -> tuple[bool, str]:
        """验证代码输出"""
        response = await self.judge.generate([
            {"role": "system", "content": self.judge_system_prompt},
            {"role": "user", "content": f"""## 任务
{task}

## 代码输出
```{code_output}
```"""}
        ])

        result = response.content if hasattr(response, 'content') else str(response)

        if result.startswith("PASS"):
            return True, result
        elif result.startswith("PARTIAL"):
            # 部分满足，可以继续但要注意警告
            return True, result
        else:
            return False, result

    async def verify_writing(self, task: str, writing_output: str) -> tuple[bool, str]:
        """验证写作输出"""
        response = await self.judge.generate([
            {"role": "system", "content": """你是一个写作质量评审专家。评估以下内容是否满足要求。

评估维度：
1. 内容质量：信息是否准确、有深度、有价值？
2. 表达清晰：是否易于理解？结构是否清晰？
3. 读者价值：读者能从中获得什么？
4. 文风一致：是否保持了统一的风格和语调？

输出格式：
PASS: [评价] — 满足要求
FAIL: [具体问题] — 需要修改
"""},
            {"role": "user", "content": f"""## 任务
{task}

## 输出
{writing_output}"""}
        ])

        result = response.content if hasattr(response, 'content') else str(response)
        return result.startswith("PASS"), result


async def agent_with_llm_judge_verification(agent, task: str, max_retries: int = 3):
    """
    带 LLM 评判者验证的 Agent 任务执行。
    """
    verifier = LLMJudgeVerifier(judge_model=agent.judge_model)

    for attempt in range(max_retries):
        # 执行任务
        result = agent.execute(task)

        # 判定输出类型，选择验证方法
        if _is_code_task(task):
            valid, feedback = await verifier.verify_code(task, result)
        elif _is_writing_task(task):
            valid, feedback = await verifier.verify_writing(task, result)
        else:
            # 默认用代码验证
            valid, feedback = await verifier.verify_code(task, result)

        if valid:
            return result

        # 验证失败，让 Agent 修复
        agent.retry(f"验证失败，请修复：{feedback}")

    return f"经过 {max_retries} 次尝试仍然无法通过验证。请检查任务本身是否可完成。"
```

## 前馈与传感器：验证的两个方向

Martin Fowler 的 Thoughtworks 团队给验证循环提供了一个概念框架：

**前馈（Feed Forward）** 是在行动前引导。给 Agent 明确的验证标准，让它知道什么样的输出算合格。Agent 在生成之前就知道要达到什么目标，而不是生成之后再检查。

这就像招聘时先写清楚职位要求，而不是看完简历再想"这个人适不适合"。

```python
class FeedForwardGuidance:
    """
    前馈引导：在任务执行前提供明确的验证标准。
    """

    def build_task_prompt(self, task: str, verification_criteria: list[str]) -> str:
        """构建带验证标准的前馈提示词"""
        criteria_text = "\n".join([f"{i+1}. {c}" for i, c in enumerate(verification_criteria)])

        return f"""## 任务
{task}

## 验证标准
在提交答案之前，请自行检查以下标准是否满足：

{criteria_text}

## 执行要求
1. 按照验证标准逐一检查你的输出
2. 如果有任何标准不满足，修正后再提交
3. 提交时说明你验证了哪些标准、如何验证的
"""
```

**传感器（Sensor）** 是在行动后观察。运行测试、截图、LLM 评判——这些都是在输出产生之后才发生的验证。

两者的区别是：前馈减少失败，传感器发现失败。前馈在事前预防，传感器在事后检测。

一个好的验证系统需要两者结合。前馈降低失败概率，传感器捕获漏网之鱼。

```python
class HybridVerification:
    """
    前馈 + 传感器的混合验证系统。
    """

    def __init__(self, agent, judge_model):
        self.agent = agent
        self.feedback = FeedForwardGuidance()
        self.sensor = LLMJudgeVerifier(judge_model)

    async def execute_with_verification(self, task: str, criteria: list[str]):
        # 1. 前馈：带验证标准执行
        guided_task = self.feedback.build_task_prompt(task, criteria)
        result = self.agent.execute(guided_task)

        # 2. 传感器：验证结果
        valid, feedback = await self.sensor.verify_code(task, result)

        if not valid:
            # 3. 基于传感器反馈重新执行
            retry_task = f"""任务：{task}

上次输出未通过验证，问题如下：
{feedback}

请修复后重新提交。"""
            result = self.agent.execute(retry_task)

            # 再次验证
            valid, feedback = await self.sensor.verify_code(task, result)
            if not valid:
                return {"success": False, "error": feedback, "attempts": 2}

        return {"success": True, "result": result}
```

## 验证的成本管理

验证不是免费的。

运行测试需要时间。截图需要计算资源。LLM 评判者本身就是一次额外的 API 调用。

一个粗略的原则：验证成本应该显著低于任务本身的成本。如果一个任务的执行成本是 1 元，验证成本 0.5 元还算合理。如果验证成本变成 2 元，你可能需要重新考虑验证策略。

这也是为什么不同类型的任务需要不同的验证深度：

```python
class VerificationStrategySelector:
    """
    根据任务类型选择验证策略。
    """

    def select_strategy(self, task: str) -> str:
        # 高风险任务（涉及钱、隐私、安全）：深度验证
        if any(keyword in task.lower() for keyword in ["支付", "密码", "删除", "payment", "delete", "security"]):
            return "full"

        # 中等风险：标准验证
        elif any(keyword in task.lower() for keyword in ["代码", "生成", "修改", "code", "generate", "modify"]):
            return "standard"

        # 低风险：轻量验证或跳过
        else:
            return "lightweight"

    def get_verifier(self, strategy: str):
        if strategy == "full":
            return FullVerificationPipeline()
        elif strategy == "standard":
            return StandardVerificationPipeline()
        else:
            return LightweightVerificationPipeline()
```

## 为什么验证循环经常被忽视

看过很多 AI 应用 demo，会发现它们很少展示验证环节。

原因是：验证让演示变得"不流畅"。

一个"流畅"的 demo 是：用户提需求，Agent 直接给出完美答案。全程无缝，用户惊呼。没有人想看到 Agent 在那边跑测试、截图、反复修改——这看起来不够"智能"。

但这是展示的技巧，不是工程的事实。

真正强大的系统不是不会出错，而是出错后能发现并修正。人类工程师也是在反复调试中才能写出正确的代码——没有人能一次性写出完美的代码，AI 也一样。

一个看起来流畅但经常输出错误结果的系统，和一个看起来"啰嗦"但结果准确的系统，后者更有价值。

验证循环是让 Agent 系统从"看起来智能"变成"真正有用"的关键。它把"希望它对"变成"确保它对"。

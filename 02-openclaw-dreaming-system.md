# 【第二篇】OpenClaw 梦境系统：让 AI 在「睡眠」中整理记忆

> 本文约 2500 字，深入解析 OpenClaw 的梦境（Dreaming）系统——三阶段协作的记忆自动晋升机制

---

## 一、为什么 AI 需要「梦境」？

人类的睡眠不是大脑的关机时间。睡眠期间，大脑对白天的记忆进行整理、强化和丢弃。REM 阶段尤其关键：神经元重播白天的经历，将短时记忆转化为长时记忆，剪除不必要的神经连接。

OpenClaw 的梦境系统就是这个过程的计算类比。

AI Agent 每天产生大量短期信号——对话、工具调用、任务结果。哪些值得晋升为长期记忆？全记住，context 被噪音淹没；全靠人工判断，成本又太高。梦境系统提供了一种自动化的记忆质量控制机制，而且全程可审计。

---

## 二、三阶段协作：Light → Deep → REM

梦境系统分三个阶段顺序执行，完成一次完整的"睡眠整理"：

```
┌──────────────────────────────────────────────────────────┐
│  LIGHT PHASE  (信息收集与初筛)                              │
│  输入：今日对话 + 工具调用 + 召回追踪记录                   │
│  输出：候选记忆条目，去重，附评分信号                       │
│  写入磁盘：无                                              │
├──────────────────────────────────────────────────────────┤
│  DEEP PHASE  (深度评估，决定晋升)                          │
│  输入：Light 阶段候选条目                                   │
│  输出：MEMORY.md 新增条目 + DREAMS.md 阶段报告             │
│  写入磁盘：MEMORY.md（新增条目）、DREAMS.md                │
├──────────────────────────────────────────────────────────┤
│  REM PHASE   (主题发现与反思)                              │
│  输入：短期追踪记录中的主题模式                             │
│  输出：主题摘要、反思性洞察                                 │
│  写入磁盘：无                                              │
└──────────────────────────────────────────────────────────┘
```

### 2.1 Light Phase：收集，去重，记录信号

Light 阶段从多个来源收集候选记忆：

```typescript
// Light Phase 输入源
interface LightPhaseInput {
  // 每日笔记文件
  dailyMemoryFiles: YYYY-MM-DD.md[];
  
  // 短期召回追踪（每个条目被查询过多少次）
  recallTraces: {
    memoryId: string;
    queryText: string;
    retrievalTimestamp: number;
  }[];
  
  // session transcript（脱敏后）
  redactedTranscripts: SessionTranscript[];
}
```

核心工作是去重和信号记录：

```python
def light_phase(inputs):
    candidates = []
    seen_signatures = set()
    
    for item in inputs:
        signature = hash(item.content)
        if signature in seen_signatures:
            continue  # 跳过重复条目
        seen_signatures.add(signature)
        
        signals = {
            "frequency": 1,
            "lastSeen": item.timestamp,
            "sourceFile": item.sourceFile
        }
        candidates.append(MemoryCandidate(item, signals))
    
    write_phase_signals(candidates)
    return candidates  # 不写入 MEMORY.md
```

### 2.2 Deep Phase：六个信号决定谁能晋升

Deep 阶段是整个系统的核心。候选记忆必须通过六信号加权评分：

| 信号 | 权重 | 含义 |
|------|------|------|
| **Relevance** | 0.30 | 该记忆被召回时与查询的平均相关度 |
| **Frequency** | 0.24 | 在短期记忆中累积出现的次数 |
| **Query Diversity** | 0.15 | 召回该条目的不同查询/天次 |
| **Recency** | 0.15 | 时间衰减后的新鲜度得分 |
| **Consolidation** | 0.10 | 跨多天出现的强度 |
| **Conceptual Richness** | 0.06 | 条目的概念标签密度 |

```python
def deep_score(candidate: MemoryCandidate) -> float:
    signals = candidate.signals
    
    score = (
        signals['relevance']           * 0.30 +
        signals['frequency_normalized'] * 0.24 +
        signals['diversity_normalized'] * 0.15 +
        signals['recency_normalized']  * 0.15 +
        signals['consolidation']       * 0.10 +
        signals['conceptual_richness'] * 0.06
    )
    
    # 必须同时满足三个阈值门控
    if not (score >= MIN_SCORE_THRESHOLD and
            signals['frequency'] >= MIN_RECALL_COUNT and
            signals['unique_queries'] >= MIN_UNIQUE_QUERIES):
        return 0.0
    
    return score
```

阈值门控是防止"偶然重要"混入长期记忆的关键——就算总分高，只被看过一次或只有一个查询场景的记忆也不会晋升。

晋升条目追加到 `MEMORY.md`：

```markdown
## 2026-04-19 晋升条目

### 用户偏好（晋升自 2026-04-17 笔记）
- 记住了用户对 TypeScript 的偏好，权重高于 Python
- 触发场景：用户在多个项目中反复提到 tsconfig 配置

### 项目关键上下文
- FantasyAILab-website 使用 Vite + React，当前处于性能优化阶段
- 重要：用户要求所有代码变更通过 Claude Code 执行
```

### 2.3 REM Phase：主题抽象与反思

REM 阶段从短期追踪中提取主题模式和反思性洞察，写入 DREAMS.md：

```python
def rem_phase(short_term_traces):
    themes = extract_themes(short_term_traces)
    
    reflections = []
    for theme in themes:
        reflections.append({
            "theme": theme.name,
            "observations": theme.observation_count,
            "insight": generate_reflection(theme)
        })
    
    append_to_dreams_md({
        "phase": "REM Sleep",
        "reflections": reflections,
        "timestamp": now()
    })
```

---

## 三、调度与触发

梦境默认每天凌晨 3 点（UTC）运行一次完整 sweep：

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true,
            "frequency": "0 3 * * *",
            "timezone": "UTC"
          }
        }
      }
    }
  }
}
```

手动触发命令：

```bash
# 预览晋升结果（不写入）
openclaw memory promote --limit 5

# 实际执行晋升
openclaw memory promote --apply

/dreaming status

# 预览 REM 反思结果
openclaw memory rem-harness
```

---

## 四、DREAMS.md：人机共读的梦境日记

梦境系统生成梦境日记（Dream Diary），保存在 `DREAMS.md`：

```markdown
# DREAMS.md — 梦境日记

## 2026-04-19

### Light Sleep
- 收集了 23 个短期记忆条目，去重后剩 14 个候选
- 主要来源：memory/2026-04-18.md（8条），recall traces（6条）

### Deep Sleep
- 候选 6 个，通过阈值门控 3 个
- 晋升：用户偏好 TypeScript > Python；FantasyAILab-website 处于性能优化阶段；timeout=300s
- 未晋升："今天天气"，frequency=1，未过 minRecallCount=2

### REM Sleep
- 主题：用户近期频繁调整定时任务配置（3次/周）
- 反思：可能需要更稳定的任务配置机制

---
```

DREAMS.md 让记忆晋升全程透明可审计。用户可以直接看：今天模型记住了什么、为什么记住了什么、什么没有记住、为什么不通过。

---

## 五、Grounded Backfill：历史笔记的回溯学习

当用户说"为什么你不知道这件事"时，可以用 backfill 从历史笔记中回溯丢失的记忆：

```bash
# 回填历史笔记
openclaw memory rem-backfill --path ./memory --stage-short-term

# 预览输出（不写入）
openclaw memory rem-harness --path ./memory --grounded

# 回滚
openclaw memory rem-backfill --rollback
openclaw memory rem-backfill --rollback-short-term
```

---

## 六、与 Hermes 的对比

Hermes 走的是**确定性规则 + API 驱动**路线，与 OpenClaw 的模型驱动评分不同：

| 维度 | OpenClaw Dreaming | Hermes 记忆系统 |
|------|-------------------|----------------|
| 晋升判断 | 六信号加权评分 + 阈值门控 | 规则匹配（显式标签/频率阈值） |
| 自动化 | 全自动（cron 调度） | 半自动（需 API 调用触发） |
| 透明性 | DREAMS.md 完全公开 | 依赖日志查询 |
| 回溯学习 | Grounded Backfill | 需重新索引历史记录 |

规则驱动适合金融、医疗等必须精确控制记忆晋升的场景；模型驱动适合个人助手——让系统自己判断什么重要，减少人工维护成本。

---

## 七、本篇小结

OpenClaw 的梦境系统由三个协作阶段构成：

- **Light Phase**：收集、去重、记录信号，不写磁盘
- **Deep Phase**：六信号评分 + 阈值门控，决定谁晋升为长期记忆
- **REM Phase**：主题发现与反思，写入 DREAMS.md
- **Grounded Backfill**：从历史笔记回溯，填补遗漏

核心理念：记忆晋升是一个可解释、可审计、可回滚的自动化过程，而不是 hidden state 中的黑箱。

---

*【预告：第三篇将对比 OpenClaw 与 Hermes Agent 的记忆系统实现细节，以及 S+Memory 作为统一记忆层如何桥接不同 Agent 框架。】*

# 【第四篇】OpenClaw vs Hermes：记忆体系设计哲学对比，以及 AI 记忆的未来

> 本文约 2500 字，从设计哲学层面深度对比两大框架的记忆体系，并展望 AI Agent 记忆系统的演进方向

---

## 一、两种哲学，两种世界观

聊技术之前，先说点更本质的——OpenClaw 和 Hermes 在记忆这件事上的设计思路，根子上就不一样。

### OpenClaw：文件即记忆，模型自己管

OpenClaw 的核心理念非常直接：**记忆就是文本文件，模型自己写、自己读、自己改**。

```
~/.openclaw/workspace/
├── MEMORY.md           # 模型自己写
├── DREAMS.md          # 模型自己分析
└── memory/
    └── 2026-04-19.md  # 模型自己记
```

这样设计的好处是**完全透明**。用户不喜欢模型记住的某件事？直接打开文件删掉。模型记忆出了偏差？打开文件，肉眼可见哪里出了问题。没有任何 hidden state 是用户看不到的。

### Hermes：结构化存储，规则即边界

Hermes 走的是**结构化 + 规则驱动**路线：

```json
{
  "memory": {
    "type": "semantic_vector",
    "store": "qdrant",
    "promotion_rules": [
      { "trigger": "frequency >= 3", "action": "promote_to_longterm" },
      { "trigger": "importance_tag == 'critical'", "action": "immediate_promote" }
    ]
  }
}
```

这种设计的哲学是**确定性优于模糊性**——什么条件触发什么行为，规则写得清清楚楚。金融、医疗、法律场景需要审计追踪，这种确定性是刚需。

---

## 二、记忆召回：两种完全不同的思路

### OpenClaw：按需检索，而非主动塞入

```python
def build_context(query: str, session_history: list) -> dict:
    # 从长期记忆里检索相关内容
    long_term = memory_search(query, source="MEMORY.md", top_k=5)
    
    # 从最近两天笔记检索
    recent = memory_search(
        query, 
        source=["memory/2026-04-19.md", "memory/2026-04-18.md"],
        top_k=3
    )
    
    # 从当前 session 历史提取相关内容
    session_context = extract_relevant_history(session_history, query, max_tokens=2000)
    
    return weave_context(long_term, recent, session_context)
```

每次 turn，模型通过检索决定需要什么记忆——**记忆是"召回"的，不是"注入"的**。这避免了塞入过多无关内容冲淡上下文。

### Hermes：预加载 + 规则注入

Hermes 的做法不同：session 开始时预加载所有高优先级记忆，然后按规则动态注入新记忆：

```python
class HermesMemoryContext:
    def __init__(self, session):
        self.session = session
        self.prioritized = self.load_prioritized_memories()
    
    def on_event(self, event):
        if self.should_inject(event):
            injected = self.rule_based_inject(event)
            self.session.append_to_context(injected)
    
    def load_prioritized_memories(self):
        return self.memory_store.query(filter={"priority": {"$gte": 0.8}}, limit=20)
```

---

## 三、遗忘机制：主动忘记才是真难题

记什么很重要，但**忘记什么**才是真正考验系统设计的地方。

### OpenClaw：自然衰减

OpenClaw 没有显式的遗忘命令。遗忘是自然发生的：

```python
def recency_score(last_accessed, now):
    days_elapsed = (now - last_accessed) / (24 * 3600)
    # 指数衰减：每7天重要性减半
    return math.exp(-0.693 * days_elapsed / 7)
```

长期不被 recall 的记忆，Deep Phase 评分会持续下降，最终不会晋升。已经晋升的记忆长期无人问津，compaction 时会被压缩成极简摘要——不会从文件里消失，但分量越来越轻。

### Hermes：TTL + 规则删除

Hermes 走得更加主动：

```json
{
  "default_ttl_days": 30,
  "forgetting_rules": [
    { "condition": "recall_count == 0 AND days_elapsed > 14", "action": "archive_to_cold_storage" },
    { "condition": "user_explicit_delete == true", "action": "hard_delete" }
  ]
}
```

14 天没人碰过的记忆自动归档，30 天直接删掉。规则清清楚楚，系统不需要"自己判断"。

---

## 四、安全与隐私：记忆系统的暗面

记忆系统带来连续性的同时，也带来了安全隐患——**记忆就是数据，记忆系统就是数据库**。

OpenClaw 的几个核心安全原则：

1. **文件即权限边界**：`MEMORY.md` 任何能读文件的人都能看
2. **DM 消息永远是不可信输入**：入站消息被标记为 `untrusted`，防止 prompt injection
3. **daily notes 按 session 隔离**：不同对话的短期记忆不会互相污染

Hermes 则走了细粒度访问控制路线，按 `private/team/public` 三级设置记忆可见性。

---

## 五、未来演进：从记忆到自我模型

这是我觉得最有意思的部分。

当前所有 AI Agent 的记忆系统都有一个共同盲点：**记忆全是"关于用户和世界"的，没有"关于 Agent 自身"的**。

一个具备自我模型的 Agent，大概长这样：

```markdown
# Agent 自我模型

## 我能做什么
擅长：代码生成、技术调研、邮件处理
不擅长：实时视频、需要物理移动的任务

## 我怎么工作的
- 通过 OpenClaw Gateway 与用户通信
- 每次对话从加载 MEMORY.md 开始
- 重要决策记录到 memory/YYYY-MM-DD.md

## 我的边界在哪里
- Gateway 是独立进程，我没有持续运行的 daemon
- 我不能主动联系用户，必须等用户先发消息

## 我如何学习
- 凌晨 3 点梦境系统整理短期记忆
- 晋升需要通过六信号评分
```

有了自我模型，Agent 才能真正做到**元认知（Meta-cognition）**——知道自己知道什么、不知道什么；知道什么时候该查文档，什么时候该直接说"这个问题我不确定"。

### 流式持续学习

当前所有记忆系统都是批次处理：每天凌晨跑一次 Dreaming sweep，把当天信号整理一遍。

未来的方向是**流式持续学习**——每次交互都实时更新记忆信号，重要信息立即晋升，不需要等到凌晨三点：

```python
class ContinuousLearning:
    def on_interaction(self, interaction):
        self.update_signals(interaction)
        
        if self.signals[memory_id].score > URGENT_PROMOTION_THRESHOLD:
            self.immediate_promote(memory_id)
```

---

## 六、横向对比

| 维度 | OpenClaw | Hermes | S+Memory | 理想 |
|------|----------|--------|----------|------|
| 存储介质 | Markdown/SQLite | JSON/向量DB | 分层结构 | 可插拔 |
| 晋升机制 | 模型评分（自动） | 规则匹配（确定） | 双轨混合 | 可配置 |
| 遗忘机制 | 自然衰减 | TTL+规则 | 重要性衰减 | 自适应 |
| 召回方式 | 动态检索 | 预加载+注入 | 语义+向量 | 智能按需 |
| 自我模型 | 无 | 部分 | 无 | 完整 |
| 跨框架 | 不支持 | 不支持 | **支持** | 原生 |
| 透明性 | **极高** | 中等 | 高 | 高 |

---

## 七、怎么选

**个人助手** → OpenClaw。文件即记忆、梦境系统全自动化，用户能直接看到和修改记忆。

**企业/合规场景** → Hermes。规则清晰、访问控制细粒度，满足审计要求。

**多框架并存** → S+Memory。一个记忆层兼容所有 Agent，不用担心信息孤岛。

**最重要的一点**：不管选哪个框架，先把遗忘机制想清楚。记忆系统的上限不是"能记多少"，而是"如何优雅地忘记"。

---

## 附录：参考资料

- OpenClaw Memory Docs: https://docs.openclaw.ai/concepts/memory
- OpenClaw Dreaming Docs: https://docs.openclaw.ai/concepts/dreaming
- S+Memory: 分层语义记忆系统，GitHub 搜索 S+Memory
- 延伸阅读：*Memory in Neural Networks: From Hebbian Learning to Transformer Memory*

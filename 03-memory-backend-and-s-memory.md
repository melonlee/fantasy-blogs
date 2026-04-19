# 【第三篇】记忆后端对决：SQLite 向量 vs Qdrant vs 纯文件，以及 S+Memory 的统一记忆层

> 本文约 2000 字，深度解析 OpenClaw 记忆系统的存储后端实现，以及跨 Agent 框架的统一记忆层设计

---

## 一、记忆存储：为什么需要不同的后端？

OpenClaw 默认使用内置 SQLite 后端，零配置就能跑起来。但不同场景有不同的需求——有人只是个人日常使用，有人要索引整个代码库，有人跑着多 Agent 协作——这三种场景对记忆存储的要求完全不在一个量级。

所以 OpenClaw 发展出了三套不同的后端，用户按需选择：

```
┌─────────────────────────────────────────────────────┐
│              记忆检索 API（统一接口）                   │
├───────────────┬───────────────┬────────────────────┤
│  Builtin      │  QMD          │  Honcho            │
│  (SQLite)     │  (Local)      │  (AI-Native)       │
│  默认方案      │  高级本地方案   │  跨会话方案        │
└───────────────┴───────────────┴────────────────────┘
```

---

## 二、Builtin 后端：SQLite 向量存储

内置记忆引擎基于 SQLite，支持三种检索模式：

### 2.1 关键词搜索（BM25）

BM25 是传统检索算法，Elasticsearch 也在用，通过词频和文档长度归一化来排序：

```sql
SELECT id, content,
       (term_frequency * idf_factor) / 
       (term_frequency + k1 * (1 - b + b * doc_length / avg_doc_length))
FROM memory_index
WHERE content MATCH 'search_query'
ORDER BY score DESC
LIMIT 10;
```

BM25 的长项是精确匹配——搜索 `0700.HK` 不会匹配成其他股票代码，搜索"Python"和"python"也能区分开来。向量引擎在这里反而容易出错。

### 2.2 向量相似度检索

```python
def vector_search(query_embedding, top_k=10):
    results = db.execute("""
        SELECT id, content, 
               cosine_distance(embedding, ?) as distance
        FROM memory_vectors
        ORDER BY distance ASC
        LIMIT ?
    """, [query_embedding, top_k])
    return results
```

向量检索的优势在于语义相近但表述不同的记忆能被召回。比如记忆里写的是"Sidonie 喜欢用 TypeScript"，搜索"用户偏好什么语言"也能找到这条——原词里根本没有"语言"两个字。

### 2.3 混合搜索：RRF 合并

同时开启向量和关键词检索时，OpenClaw 用 **RRF（Reciprocal Rank Fusion）** 把两类结果合并排序：

```python
def reciprocal_rank_fusion(results_list, k=60):
    # RRF 公式：score = Σ 1/(k + rank)
    fused_scores = defaultdict(float)
    
    for results in results_list:
        for rank, item in enumerate(results):
            fused_scores[item.id] += 1.0 / (k + rank + 1)
    
    return sorted(fused_scores.items(), key=lambda x: -x[1])
```

k=60 是经验值，作用是压制高排名结果的优势，让向量和 BM25 的排名共同决定最终结果。好处是不用调参，直接用就行。

---

## 三、QMD 后端：本地优先的高级检索

QMD（Query, Memory, Dedupe）适合需要索引 workspace 外部目录的场景：

```json
{
  "plugins": {
    "entries": {
      "memory-qmd": {
        "config": {
          "indexPaths": [
            "./projects",
            "/Users/melon/Documents"
          ],
          "rerankModel": "openai/text-embedding-3-small",
          "queryExpansion": true
        }
      }
    }
  }
}
```

三个核心特性：

- **外部目录索引**：不只是 workspace，可以索引本地任意目录
- **Rerank**：用更精确的模型对初筛结果重排，提高精度
- **Query Expansion**：将一个查询扩展成多个相关查询，提升召回

---

## 四、Honcho 后端：跨会话 AI 原生记忆

Honcho 是专为 AI Agent 设计的跨会话记忆系统，强调用户建模和多 Agent 感知：

```python
class UserMemory:
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.profiles = {}        # 每个 Agent 对该用户的理解
        self.cross_agent_knowledge = {}  # 跨 Agent 共享知识
    
    def remember(self, agent_id: str, fact: str, context: dict):
        entry = MemoryEntry(
            agent_id=agent_id,
            fact=fact,
            context=context,
            timestamp=now(),
            visibility='cross_agent'
        )
        self.vector_store.add(entry)
```

当规划 Agent 记住的偏好需要被执行 Agent 使用时，Honcho 提供天然的共享机制。这在多 Agent 协作场景里非常实用。

---

## 五、S+Memory：跨 Agent 框架的统一记忆层

S+Memory（Layered semantic memory system for AI agents）是 GitHub 上值得关注的项目，设计目标是兼容 OpenClaw、Hermes 和任意 Agent 系统。

### 5.1 三 Layer 架构

```
┌─────────────────────────────────────────────────────┐
│         L1: Episodic Memory（情景记忆）               │
│  「今天下午做了什么」                                 │
│  结构：事件 + 时间戳 + 情感标记                       │
├─────────────────────────────────────────────────────┤
│         L2: Semantic Memory（语义记忆）               │
│  「TypeScript 是什么」                                 │
│  结构：概念 + 关系图谱 + 来源引用                     │
├─────────────────────────────────────────────────────┤
│         L3: Procedural Memory（程序记忆）            │
│  「怎么配置 cron 任务」                               │
│  结构：步骤 + 前提条件 + 验证方法                     │
└─────────────────────────────────────────────────────┘
```

三层划分源自认知科学：情景记忆（个人经历）、语义记忆（事实知识）、程序记忆（技能流程）。

### 5.2 代码示例

```python
from smemory import Session

sm = Session(agent="openclaw", user_id="sidonie")

# 记忆一次交互
sm.remember(
    layer="episodic",
    content="用户在 2026-04-19 配置了微信插件",
    emotion="positive",
    importance=0.8
)

# 检索记忆
results = sm.recall(query="微信配置", layer="episodic", top_k=5)

# 晋升（episodic -> semantic）
sm.promote(
    from_layer="episodic", to_layer="semantic",
    entry_id="mem_20260419_001",
    reason="用户在多天内三次提到"
)
```

### 5.3 特色功能

**情感标记**：

```python
sm.remember(content="代码终于跑通了！", emotion="joy", intensity=0.9)
```

情感记忆在召回时可以调整回复风格——高兴的事用轻快语气，沮丧的事用沉稳语气。

**重要性衰减**：经常被召回的记忆权重会提升，长期无人访问的会自然衰减，不需人工维护。

**跨框架同步**：

```python
sm.sync_to(targets=["openclaw", "hermes"], memory_id="mem_001",
           conflict_resolver="last_write_wins")
```

OpenClaw 和 Hermes 共用同一份记忆，冲突时按最后写入优先解决。

---

## 六、后端选择

| 场景 | 推荐 |
|------|------|
| 个人助手，日常使用 | Builtin（SQLite） |
| 索引大量本地文件 | QMD |
| 多 Agent 协作 | Honcho |
| OpenClaw + Hermes 并存 | S+Memory |

---

## 七、本篇小结

OpenClaw 的记忆后端体系按需渐进：从零配置的 SQLite，到本地高级检索的 QMD，再到多 Agent 感知的 Honcho。

S+Memory 的出现则指向一个更大的趋势：Agent 框架越来越多，记忆系统越来越碎片化，跨框架统一记忆层会成为刚需。

---

*【预告：第四篇将综合对比 OpenClaw 与 Hermes 的记忆体系设计哲学差异，并展望 AI Agent 记忆系统的未来演进方向。】*

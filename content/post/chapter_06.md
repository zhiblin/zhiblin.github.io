---
title: 第六章：记忆系统——从会话到知识图谱
date: 2026-01-06 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第六章：记忆系统——从会话到知识图谱

> "没有记忆的智能体，每次都是新手。有了记忆，它才能成为专家。"

## 6.1 智能体的失忆症

每次启动一个新的 AI 智能体会话，上下文窗口都是空白的。上一次任务中发现的 API 怪癖、踩过的坑、学到的代码约定——全部消失。

这是 LLM 架构的根本限制：模型本身包含训练数据的知识，但不包含你的项目的知识。每个会话必须重新从代码库中发现这一切。

在单任务场景下这还能接受。但在 Auto-Claude 的多轮自主开发中，代价急剧放大：

- 第 1 次任务：发现 `utils/database.py` 有连接池 bug，QA 修复了它
- 第 2 次任务：Coder 又调用了同样的 API，再次触发 bug
- 第 3 次任务：同样的问题第三次出现...

解决方案是**持久记忆**：将智能体在工作过程中积累的洞察保存下来，在后续会话中注入。

## 6.2 两种记忆模型的权衡

### 文件式记忆（Files as Memory）

最简单的持久记忆是文件。Auto-Claude 的每个 Coder 会话结束后，会将关键洞察写入 `session_insights/` 目录下的 JSON 文件：

```python
# coder.md 提示词中的记忆写入模式
"""
## Session Memory Write
After completing your work, write a session summary to:
  {spec_dir}/session_insights/session_{n}.json

Include:
- What you built and key decisions made
- Any gotchas or surprises encountered
- Patterns you discovered in the codebase
- What you'd tell yourself if you had to redo this task
"""
```

这种方式极其简单：JSON 文件，无需数据库，人可以直接阅读和编辑。下一个会话读取这些文件，获得上下文。

**优点**：零依赖，透明，可调试
**缺点**：搜索能力弱（只能全读）；随着文件增多，注入量线性增长，消耗大量上下文窗口

### 知识图谱记忆（Graph-based Memory）

Auto-Claude 的主要记忆系统是基于 **Graphiti** 的语义知识图谱：

```
传统文件记忆：
  session_1.json ──→ [全量注入] ──→ 智能体
  session_2.json ──→ [全量注入] ──→ 智能体
  session_n.json ──→ [全量注入] ──→ 智能体
  （文件越多，注入量越大）

知识图谱记忆：
  session_1 ──→ [提取实体和关系] ──→ Neo4j/KuzuDB 图数据库
  session_2 ──→ [提取实体和关系] ──┘
  session_n ──→ [提取实体和关系] ──┘
                                    ↓
                              [语义搜索]
                                    ↓
               "与当前任务相关的记忆" ──→ 智能体（精确注入）
```

知识图谱让系统能做**语义检索**：不是把所有记忆倒给智能体，而是找出"与这个任务最相关的历史经验"。

`★ Insight ─────────────────────────────────────`

**记忆系统的本质是信息检索问题**：如何在海量历史经验中找到当前最有用的部分？文件全量注入是 O(N) 的暴力解法；图数据库语义检索是接近 O(1) 的精确解法。这里的权衡不是"文件 vs 图谱"，而是"系统复杂度 vs 记忆可用性"。

`─────────────────────────────────────────────────`

## 6.3 Graphiti 架构

Graphiti 是一个开源的时序知识图谱库，专为 AI 智能体设计。Auto-Claude 使用嵌入式 KuzuDB 作为底层存储（无需单独部署 Neo4j）：

```
Graphiti 技术栈：

graphiti-core         ← 核心库（实体提取、关系推断、时序版本化）
    │
    ├── LLM Client    ← 实体提取用（支持 Claude/GPT/Ollama...）
    │
    ├── Embedder      ← 向量嵌入（支持 OpenAI/Voyage/Google...）
    │
    └── Graph Driver  ← 底层图数据库
            │
            └── KuzuDB（嵌入式）← Auto-Claude 使用此配置
                                   （无需 Neo4j，零运维）
```

`GraphitiMemory` 类是系统的统一门面：

```python
class GraphitiMemory:
    """
    持久记忆的高层接口。

    所有操作都是 async 并有降级处理：
    如果 Graphiti 未启用或不可用，操作静默跳过。
    """

    def __init__(
        self,
        spec_dir: Path,
        project_dir: Path,
        group_id_mode: str = GroupIdMode.PROJECT,  # 默认：项目级共享
    ):
        self.group_id_mode = group_id_mode
        # ...

    @property
    def group_id(self) -> str:
        """
        记忆命名空间：决定记忆的作用域。

        SPEC 模式：每个任务独立记忆（适合隔离实验）
        PROJECT 模式：所有任务共享项目记忆（跨任务学习）
        """
        if self.group_id_mode == GroupIdMode.PROJECT:
            # 用项目路径 hash 确保唯一性
            path_hash = hashlib.md5(
                str(self.project_dir.resolve()).encode(),
                usedforsecurity=False
            ).hexdigest()[:8]
            return f"project_{self.project_dir.name}_{path_hash}"
        else:
            return self.spec_dir.name  # "001-add-auth"
```

注意默认值从 `SPEC` 改为了 `PROJECT`——代码注释说明了原因："Enable cross-spec learning across the entire project"。这是一个经过实践验证的设计决策：项目级共享记忆让后续任务能受益于前序任务的经验。

## 6.4 Episode 类型：知识的分类体系

Graphiti 将记忆存储为 **Episode**（情节）。Auto-Claude 定义了七种 Episode 类型，每种捕获不同维度的知识：

```python
# integrations/graphiti/queries_pkg/schema.py

EPISODE_TYPE_SESSION_INSIGHT = "session_insight"
# 用途：智能体会话的总体洞察
# 示例："在此任务中，发现数据库连接需要指定 charset=utf8mb4"

EPISODE_TYPE_CODEBASE_DISCOVERY = "codebase_discovery"
# 用途：代码库结构和模式的发现
# 示例："auth/ 模块使用 JWT，token 存储在 Redis 中，TTL 24小时"

EPISODE_TYPE_PATTERN = "pattern"
# 用途：项目中反复出现的代码模式
# 示例："所有 API handler 都需要 @require_auth 装饰器"

EPISODE_TYPE_GOTCHA = "gotcha"
# 用途：陷阱、已知问题、不直观的行为
# 示例："切勿在 async 函数中直接调用 db.query()，必须用 await db.query()"

EPISODE_TYPE_TASK_OUTCOME = "task_outcome"
# 用途：任务完成结果（成功/失败、方法、耗时）

EPISODE_TYPE_QA_RESULT = "qa_result"
# 用途：QA 发现的问题和修复方式

EPISODE_TYPE_HISTORICAL_CONTEXT = "historical_context"
# 用途：历史决策和架构演化的背景
```

这个分类体系不仅是组织的需要，更影响**检索策略**：搜索"陷阱"时只查 `GOTCHA` 类型；搜索"代码模式"时只查 `PATTERN` 类型，避免信息噪音。

## 6.5 记忆的写入：从会话到知识节点

智能体会话结束后，`post_session_processing()` 函数（在 `agents/session.py` 中）触发记忆写入：

```python
async def save_session_insights(
    self,
    session_num: int,
    insights: dict,
) -> bool:
    """
    将会话洞察保存为 Graphiti Episode。
    """
    if not await self._ensure_initialized():
        return False  # Graphiti 未启用，静默跳过

    try:
        result = await self._queries.add_session_insight(session_num, insights)

        if result and self.state:
            self.state.last_session = session_num
            self.state.episode_count += 1
            self.state.save(self.spec_dir)  # 持久化状态

        return result
    except Exception as e:
        logger.warning(f"Failed to save session insights: {e}")
        # 记忆写入失败不应阻断工作流
        return False
```

注意错误处理策略：记忆写入失败时**记录警告，但不抛出异常**。记忆系统是辅助系统，它的失败不应该影响主要工作流。这是"优雅降级"的体现。

`GraphitiMemory` 支持的写入操作：

```python
# 保存代码库发现
await memory.save_codebase_discoveries({
    "auth_module": "JWT-based, Redis token storage, 24h TTL",
    "database_pattern": "Connection pool size=10, charset must be utf8mb4",
})

# 保存 Gotcha（陷阱）
await memory.save_gotcha(
    "database_async",
    "Never call db.query() directly in async functions, use 'await db.query()'"
)

# 保存任务结果
await memory.save_task_outcome(
    task_id="001-add-auth",
    success=True,
    approach="JWT with Redis session store",
    duration_minutes=45,
)

# 保存 QA 结果
await memory.save_qa_result(
    issue="Missing input validation in POST /users",
    fix="Added Pydantic model for request validation",
    severity="HIGH",
)
```

## 6.6 记忆的读取：语义检索

读取记忆比写入更复杂，因为需要**语义相关性判断**：

```python
async def get_relevant_context(
    self,
    query: str,
    max_results: int = MAX_CONTEXT_RESULTS,  # 默认最多 10 条
) -> list[dict]:
    """
    语义搜索：找出与查询最相关的历史记忆。

    使用向量相似度搜索，而不是关键词匹配。
    返回按相关度排序的 Episode 列表。
    """
    if not await self._ensure_initialized():
        return []

    try:
        return await self._search.semantic_search(query, max_results)
    except Exception as e:
        logger.warning(f"Memory search failed: {e}")
        return []  # 搜索失败返回空，不阻断工作
```

`MAX_CONTEXT_RESULTS = 10` 是一个刻意的限制。即使有 1000 条历史记忆，也只注入最相关的 10 条。这防止了记忆系统"帮倒忙"——把大量不相关的历史经验塞满智能体的上下文窗口。

### 向量搜索的原理

Graphiti 使用**向量嵌入**实现语义搜索：

```
写入时：
  "Connection pool size must be 10, charset utf8mb4"
       ↓  embedding model
  [0.23, -0.41, 0.67, ..., 0.12]  (1536维向量)
       ↓  存储到图数据库

检索时：
  查询："如何连接 MySQL？"
       ↓  embedding model
  [0.21, -0.38, 0.71, ..., 0.15]  (查询向量)
       ↓  余弦相似度计算
  找到最近邻 Episode → 返回
```

语义相似度捕获了语义上的相关性：即使查询用词与存储内容完全不同，只要语义接近，就能被找到。

## 6.7 嵌入提供商的多样性

Auto-Claude 支持多种嵌入提供商，通过工厂模式统一创建：

```python
# integrations/graphiti/providers_pkg/factory.py（概念化）

EMBEDDER_PROVIDERS = {
    "openai":      OpenAIEmbedder,       # text-embedding-3-small/large
    "voyage":      VoyageEmbedder,       # voyage-code-2（代码优化）
    "google":      GoogleEmbedder,       # text-embedding-004
    "azure_openai":AzureOpenAIEmbedder,  # 企业 Azure 部署
    "ollama":      OllamaEmbedder,       # 本地模型（隐私保护）
    "openrouter":  OpenRouterEmbedder,   # 路由器（多模型切换）
}

def create_embedder(config: GraphitiConfig) -> BaseEmbedder:
    provider_name = config.embedder_provider
    if provider_name not in EMBEDDER_PROVIDERS:
        raise ProviderError(f"Unknown embedder: {provider_name}")
    return EMBEDDER_PROVIDERS[provider_name](config)
```

为什么支持这么多提供商？核心原因是**向量维度锁定**问题：

```
危险场景：
  用 OpenAI text-embedding-3-small (1536维) 写入了 500 条记忆
       ↓
  切换到 Voyage voyage-code-2 (1024维)
       ↓
  维度不匹配！无法搜索历史记忆！
```

`GraphitiMemory` 通过 `GraphitiState` 追踪当前使用的嵌入提供商，检测到切换时发出警告：

```python
if self.state and self.state.has_provider_changed(self.config):
    migration_info = self.state.get_migration_info(self.config)
    logger.warning(
        f"⚠️  Embedding provider changed: "
        f"{migration_info['old_provider']} → {migration_info['new_provider']}"
    )
    logger.warning(
        "   Run: python integrations/graphiti/migrate_embeddings.py"
    )
```

迁移脚本 `migrate_embeddings.py` 会用新提供商重新为所有 Episode 生成嵌入向量。

## 6.8 记忆命名空间：隔离 vs 共享

`GroupIdMode` 控制记忆的作用域：

```python
# SPEC 模式：每个任务看到独立的记忆空间
memory_001 = GraphitiMemory(spec_dir="specs/001-auth", group_id_mode=GroupIdMode.SPEC)
memory_002 = GraphitiMemory(spec_dir="specs/002-payment", group_id_mode=GroupIdMode.SPEC)
# 搜索 memory_001 时，只能找到任务 001 的记忆
# 搜索 memory_002 时，只能找到任务 002 的记忆

# PROJECT 模式：所有任务共享项目记忆
memory_001 = GraphitiMemory(spec_dir="specs/001-auth", group_id_mode=GroupIdMode.PROJECT)
memory_002 = GraphitiMemory(spec_dir="specs/002-payment", group_id_mode=GroupIdMode.PROJECT)
# 任务 001 写入的 Gotcha，任务 002 可以检索到
# 任务 002 发现的 Pattern，任务 003 可以受益于
```

默认使用 `PROJECT` 模式，正是为了实现"跨任务学习"。随着项目迭代，知识图谱积累的项目特定知识越来越多，后续任务的智能程度也越来越高。

## 6.9 LadybugDB：无需远程服务器的图数据库

传统知识图谱需要运行 Neo4j 服务器，增加了部署和运维成本。Auto-Claude 使用 **LadybugDB**（基于 KuzuDB 的嵌入式版本）——像 SQLite 之于关系型数据库的角色：

```python
# 创建嵌入式数据库（无需服务器）
from integrations.graphiti.queries_pkg.kuzu_driver_patched import (
    create_patched_kuzu_driver,
)

db_path = config.get_db_path()  # ~/.auto-claude/graphiti/project_xxx.db
driver = create_patched_kuzu_driver(db=str(db_path))

graphiti = Graphiti(
    graph_driver=driver,     # 嵌入式，文件存储
    llm_client=llm_client,
    embedder=embedder,
)
```

"Patched" KuzuDriver 是另一个 Monkey-patch：将 Graphiti 默认期望的 Neo4j 驱动接口替换为 KuzuDB 的嵌入式接口。第四章介绍的 Monkey-patch 模式在这里再次出现——这是处理第三方库接口不兼容时的实用工具。

`★ Insight ─────────────────────────────────────`

**嵌入式数据库的产品哲学**：SQLite 的成功揭示了一个真理——大多数应用不需要独立的数据库服务器，嵌入式方案在降低部署复杂度的同时满足 90% 的需求。知识图谱领域长期缺乏这样的选项，KuzuDB 填补了这个空白。Auto-Claude 的用户无需安装 Neo4j，只需安装 Python 包，这是一个关键的用户体验决策。

`─────────────────────────────────────────────────`

## 6.10 记忆系统的完整生命周期

```
任务 001 开始
    │
    ├─ [读取] 查询 Graphiti："与认证相关的历史经验"
    │         → 结果为空（第一次任务）
    │
    ├─ Coder 实现认证系统
    │   发现：JWT 库有时区 bug，需要 utc=True
    │
    ├─ QA 发现：missing rate limiting
    │   QA Fixer 添加了 rate limiter
    │
    ├─ [写入] 会话结束时保存：
    │   save_gotcha("jwt_timezone", "Always use utc=True with JWT library")
    │   save_codebase_discovery("auth_pattern", "Rate limiting via Redis, 100 req/min")
    │   save_qa_result("missing_rate_limit", "Fixed by adding Redis-based rate limiter")
    │
任务 002 开始（实现用户 API）
    │
    ├─ [读取] 查询 Graphiti："API 开发相关经验"
    │         → 返回：
    │           - gotcha: JWT 时区问题
    │           - discovery: Rate limiting 模式
    │
    ├─ Coder 收到上下文，自动避开了 JWT 坑
    │   自动添加了 rate limiting（参考已知模式）
    │
    ← 质量显著提升，QA 问题更少
```

## 6.11 小结：两种记忆的协同

Auto-Claude 实际上使用了两种记忆系统的组合：

| 维度 | 文件记忆 | Graphiti 图谱记忆 |
|------|---------|------------------|
| 依赖 | 无（内置）| LLM API + Embedder |
| 搜索 | 全量读取 | 语义向量检索 |
| 扩展性 | O(N) 线性退化 | 稳定（固定结果数） |
| 透明度 | 人可直接阅读 | 需要工具查看 |
| 适用场景 | 当前任务内短期记忆 | 跨任务长期知识积累 |

两者互补：文件记忆处理会话内的短期上下文，Graphiti 处理项目生命周期中的长期知识积累。

---

## Lab 6.1：构建一个项目知识库

在这个实验中，你将实现一个轻量级的项目知识库，模拟 Graphiti 的核心功能，但不依赖图数据库。

### 背景

你正在为一个代码助手构建知识积累系统。助手每次工作后会发现新的知识，你需要：
1. 按类型存储知识片段
2. 基于语义相关性检索（使用 TF-IDF 简化版）
3. 控制检索结果数量，避免上下文溢出

### Part A：知识存储

```python
# lab_06_memory.py

import json
import math
from dataclasses import dataclass, field, asdict
from datetime import datetime
from pathlib import Path
from typing import Optional
from collections import Counter

class EpisodeType:
    GOTCHA = "gotcha"
    PATTERN = "pattern"
    CODEBASE_DISCOVERY = "codebase_discovery"
    TASK_OUTCOME = "task_outcome"

@dataclass
class Episode:
    """一条知识记忆。"""
    id: str
    episode_type: str
    content: str
    metadata: dict = field(default_factory=dict)
    created_at: str = field(default_factory=lambda: datetime.now().isoformat())
    tags: list[str] = field(default_factory=list)

class KnowledgeStore:
    """
    轻量级知识存储，使用 JSON 文件持久化。
    """

    def __init__(self, storage_dir: Path):
        self.storage_dir = storage_dir
        self.storage_dir.mkdir(parents=True, exist_ok=True)
        self._episodes: list[Episode] = []
        self._load()

    def add_episode(
        self,
        episode_type: str,
        content: str,
        metadata: dict = None,
        tags: list[str] = None,
    ) -> Episode:
        """
        TODO Part A: 添加一条知识记忆

        要求：
        1. 生成唯一 ID（类型 + 序号，如 "gotcha_001"）
        2. 创建 Episode 对象
        3. 添加到 self._episodes
        4. 持久化到磁盘（调用 self._save()）
        5. 返回 Episode
        """
        pass

    def get_by_type(self, episode_type: str) -> list[Episode]:
        """按类型筛选知识。"""
        return [e for e in self._episodes if e.episode_type == episode_type]

    def _save(self):
        """持久化到 JSON 文件。"""
        data = [asdict(e) for e in self._episodes]
        store_file = self.storage_dir / "knowledge.json"
        with open(store_file, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

    def _load(self):
        """从文件恢复。"""
        store_file = self.storage_dir / "knowledge.json"
        if store_file.exists():
            with open(store_file, encoding="utf-8") as f:
                data = json.load(f)
            self._episodes = [Episode(**d) for d in data]
```

### Part B：TF-IDF 语义检索

```python
class TFIDFSearcher:
    """
    使用 TF-IDF 实现简化的语义检索。

    TF-IDF 原理：
    - TF (词频)：词在当前文档中出现的频率
    - IDF (逆文档频率)：词在所有文档中的稀缺程度
    - TF-IDF = TF × IDF：重要词得分高，常见词得分低
    """

    def search(
        self,
        query: str,
        episodes: list[Episode],
        max_results: int = 5,
    ) -> list[tuple[Episode, float]]:
        """
        TODO Part B: 实现 TF-IDF 搜索

        步骤：
        1. 对 query 和所有 episode.content 进行分词（简单按空格/标点分割）
        2. 计算 IDF：对每个词，统计出现在多少个文档中
           IDF(word) = log(总文档数 / 包含该词的文档数)
        3. 对每个 episode，计算与 query 的相似度：
           - 找出 query 和 episode 共有的词
           - 相似度 = sum(TF(word, episode) * IDF(word))
        4. 返回相似度最高的 max_results 个 (episode, score) 对

        提示：
        - 使用 Counter 统计词频
        - 忽略长度 <= 2 的词（停用词过滤）
        - 将词转为小写进行匹配
        """
        if not episodes:
            return []

        # TODO: 实现 TF-IDF 检索逻辑
        pass

    def _tokenize(self, text: str) -> list[str]:
        """简单分词：小写 + 按非字母数字字符分割。"""
        import re
        words = re.split(r'[^a-zA-Z0-9\u4e00-\u9fff]+', text.lower())
        return [w for w in words if len(w) > 2]
```

### Part C：记忆注入

```python
class MemoryInjector:
    """
    将检索到的记忆注入智能体提示词。
    控制注入量，避免上下文溢出。
    """

    MAX_INJECTION_CHARS = 2000  # 最多注入 2000 字符

    def build_memory_context(
        self,
        query: str,
        store: KnowledgeStore,
        searcher: TFIDFSearcher,
        max_results: int = 5,
    ) -> str:
        """
        TODO Part C: 构建记忆注入文本

        要求：
        1. 用 searcher 检索与 query 相关的 episodes
        2. 按类型分组（先展示 GOTCHA，再展示 PATTERN，再展示其他）
        3. 格式化为可读的提示词片段
        4. 如果总字符数超过 MAX_INJECTION_CHARS，截断到限制内
        5. 返回格式化后的字符串

        格式参考：
        ## Relevant Historical Context

        ### Known Gotchas
        - [gotcha_001] Always use utc=True with JWT library

        ### Patterns
        - [pattern_001] Rate limiting via Redis, 100 req/min
        """
        all_episodes = store._episodes
        if not all_episodes:
            return ""

        # TODO: 实现记忆注入逻辑
        pass


def run_demo():
    """演示完整的记忆积累和检索流程。"""
    import tempfile

    with tempfile.TemporaryDirectory() as tmp:
        store = KnowledgeStore(Path(tmp))
        searcher = TFIDFSearcher()
        injector = MemoryInjector()

        # 模拟任务 001：认证系统开发的发现
        store.add_episode(
            EpisodeType.GOTCHA,
            "JWT library requires utc=True parameter, otherwise timezone bugs occur in production",
            tags=["jwt", "authentication", "timezone"],
        )
        store.add_episode(
            EpisodeType.PATTERN,
            "Rate limiting is implemented via Redis with 100 requests per minute limit",
            tags=["redis", "rate-limit", "api"],
        )
        store.add_episode(
            EpisodeType.CODEBASE_DISCOVERY,
            "Database connections must use charset=utf8mb4 to support emoji characters",
            tags=["database", "mysql", "encoding"],
        )
        store.add_episode(
            EpisodeType.TASK_OUTCOME,
            "Authentication system implemented using JWT + Redis session store, took 45 minutes",
            metadata={"task_id": "001-auth", "success": True},
        )

        # 模拟任务 002：搜索与 API 开发相关的历史记忆
        query = "building REST API with authentication and database"
        context = injector.build_memory_context(query, store, searcher)

        print("=== 注入的历史记忆 ===")
        print(context)
        print(f"\n字符数: {len(context)}")

        # 验证
        assert "JWT" in context or "jwt" in context.lower(), "JWT gotcha 应该被检索到"
        assert len(context) <= MemoryInjector.MAX_INJECTION_CHARS, "不应超过字符限制"
        print("\n✓ 所有验证通过")


if __name__ == "__main__":
    run_demo()
```

### 验证标准

| 检查点 | 预期结果 |
|--------|---------|
| Episode 持久化 | 程序重启后能恢复历史记忆 |
| TF-IDF 检索 | 语义相关的内容排名靠前 |
| 字符限制 | 注入文本不超过 MAX_INJECTION_CHARS |
| 类型分组 | GOTCHA 优先于 PATTERN 展示 |

### 进阶挑战

1. **向量嵌入**：将 TF-IDF 替换为真实的向量嵌入（使用 `sentence-transformers` 库的本地模型，避免 API 调用）。比较两种方法的检索质量。

2. **记忆衰减**：给每条 Episode 添加"访问时间"字段。检索时对更近期被访问的 Episode 适当加权（"近因效应"）。

3. **命名空间**：实现 `NamespacedStore`，支持 SPEC 和 PROJECT 两种模式。在 SPEC 模式下，不同任务的记忆完全隔离；在 PROJECT 模式下，所有任务共享。

---

**本章要点回顾**

- 智能体每次启动都是"失忆"的，持久记忆是让 AI 系统随时间变得更聪明的关键
- 文件记忆（简单）vs 知识图谱（精确）：两者各有适用场景，Auto-Claude 同时使用
- Graphiti 使用向量嵌入实现语义检索，返回语义相关而非关键词匹配的结果
- `MAX_CONTEXT_RESULTS = 10` 是控制记忆注入量的关键限制，防止记忆系统适得其反
- `GroupIdMode.PROJECT` 实现跨任务学习：前序任务的经验传递给后续任务
- 嵌入提供商切换会导致维度不匹配，需要迁移脚本重新生成向量
- 所有记忆操作应优雅降级：失败时不阻断主工作流

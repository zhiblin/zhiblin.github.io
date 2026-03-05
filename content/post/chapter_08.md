---
title: 第八章：上下文工程——给智能体提供恰当的信息
date: 2026-01-08 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第八章：上下文工程——给智能体提供恰当的信息

> "智能体的输出质量，上限是它输入信息的质量。"

## 8.1 上下文的本质问题

你让一个新来的开发者实现一个功能，你需要告诉他什么？

- 项目用什么技术栈
- 哪些文件可能需要修改
- 项目里有哪些约定和模式
- 这个功能和哪些现有代码相关
- 有哪些已知的陷阱要避免

你不需要给他整个代码库——那会让他不知从何开始。你需要给他**恰好足够**的上下文。

AI 智能体面临同样的问题，但上下文窗口有硬性限制（通常 200k token）。整个代码库往往超过这个限制，即使放进去，在如此大量的信息中找到相关内容也是挑战。

**上下文工程（Context Engineering）**：在有限的上下文窗口内，给智能体提供最相关、最有用的信息子集。

## 8.2 上下文的三个维度

Auto-Claude 的上下文系统从三个维度为每个任务构建上下文：

```
任务描述：
"实现用户登录 API，支持邮箱密码验证"
         │
         ├── 维度 1：代码库搜索
         │   哪些文件可能需要修改？哪些文件需要参考？
         │   → files_to_modify: [src/auth/handlers.py, ...]
         │   → files_to_reference: [src/models/user.py, ...]
         │
         ├── 维度 2：模式发现
         │   项目里有哪些相关的代码模式？
         │   → patterns: {"api_handler": "use @require_auth decorator", ...}
         │
         └── 维度 3：历史记忆（Graphiti）
             这个项目做过类似工作吗？有什么经验教训？
             → graph_hints: [{"type": "gotcha", "content": "JWT needs utc=True"}, ...]
```

三个维度汇集为 `TaskContext` 对象，注入到智能体的提示词中。

## 8.3 ContextBuilder：上下文构建的编排者

`apps/backend/context/builder.py` 中的 `ContextBuilder` 类是上下文系统的核心：

```python
class ContextBuilder:
    """
    通过搜索代码库，为任务构建专用上下文。

    组件协作：
    - ServiceMatcher: 从任务描述推断相关服务
    - KeywordExtractor: 从任务描述提取搜索关键词
    - CodeSearcher: 在服务目录中搜索匹配文件
    - FileCategorizer: 将匹配文件分为"需修改"和"需参考"
    - PatternDiscoverer: 从参考文件中提取代码模式
    - GraphitiSearch: 从知识图谱获取历史提示
    """

    def build_context(
        self,
        task: str,
        services: list[str] | None = None,
        keywords: list[str] | None = None,
        include_graph_hints: bool = True,
    ) -> TaskContext:
        # 1. 服务推断
        if not services:
            services = self.service_matcher.suggest_services(task)

        # 2. 关键词提取
        if not keywords:
            keywords = self.keyword_extractor.extract_keywords(task)

        # 3. 代码搜索（遍历每个相关服务）
        all_matches = []
        for service_name in services:
            matches = self.searcher.search_service(
                service_path, service_name, keywords
            )
            all_matches.extend(matches)

        # 4. 文件分类
        files_to_modify, files_to_reference = self.categorizer.categorize_matches(
            all_matches, task
        )

        # 5. 模式发现
        patterns = self.pattern_discoverer.discover_patterns(
            files_to_reference, keywords
        )

        # 6. 历史记忆注入
        graph_hints = []
        if include_graph_hints and is_graphiti_enabled():
            graph_hints = asyncio.run(
                fetch_graph_hints(task, self.project_dir)
            )

        return TaskContext(
            task_description=task,
            scoped_services=services,
            files_to_modify=files_to_modify,
            files_to_reference=files_to_reference,
            patterns_discovered=patterns,
            service_contexts=service_contexts,
            graph_hints=graph_hints,
        )
```

注意步骤 6 的异步处理：`build_context()` 是同步函数（CLI 友好），但 Graphiti 搜索是异步的。代码使用 `asyncio.run()` 在同步上下文中执行异步操作，但要检查是否已在事件循环中（Electron 环境中可能已有事件循环）：

```python
# 异步/同步边界处理（实际代码中）
try:
    loop = asyncio.get_running_loop()
    # 已在 async 上下文中，不能调用 asyncio.run()
    graph_hints = []  # 降级：跳过 Graphiti
except RuntimeError:
    # 没有运行中的事件循环，可以安全调用
    graph_hints = asyncio.run(fetch_graph_hints(...))
```

`★ Insight ─────────────────────────────────────`

**异步/同步边界是 Python 应用中的常见陷阱**：`asyncio.run()` 在已有事件循环的环境中会崩溃（`RuntimeError: This event loop is already running`）。处理方式是用 `asyncio.get_running_loop()` 检测，有则降级，无则运行。另一种方式是让整个调用链都是 async，但这会改变函数签名，影响 CLI 调用场景。这是工程中的现实权衡。

`─────────────────────────────────────────────────`

## 8.4 服务推断：从任务到代码区域

大型项目通常按"服务"或"模块"组织（`auth/`, `payment/`, `notification/` 等）。为整个代码库搜索既慢又噪音大；只搜索相关服务则高效精准。

`ServiceMatcher` 通过关键词匹配推断相关服务：

```python
class ServiceMatcher:
    """
    从任务描述推断相关的服务/模块。
    """

    def __init__(self, project_index: dict):
        # project_index 包含项目所有服务的目录和关键词
        self.project_index = project_index

    def suggest_services(self, task: str) -> list[str]:
        """
        根据任务描述推断相关服务。

        示例：
        task = "实现用户登录"
        → 匹配关键词 "user", "login", "auth"
        → 推断服务: ["auth", "user"]
        """
        task_lower = task.lower()
        scored_services = []

        for service_name, service_info in self.project_index.get("services", {}).items():
            score = 0

            # 服务名称匹配
            if service_name.lower() in task_lower:
                score += 10

            # 服务关键词匹配
            for keyword in service_info.get("keywords", []):
                if keyword.lower() in task_lower:
                    score += 1

            if score > 0:
                scored_services.append((score, service_name))

        # 按匹配分数排序
        scored_services.sort(reverse=True)
        return [name for _, name in scored_services[:5]]  # 最多 5 个服务
```

`project_index.json` 是项目分析的结果文件，包含所有服务的路径和关键词。它由 `analyze_project()` 生成，并在 `ContextBuilder` 初始化时加载。

## 8.5 关键词提取：从自然语言到搜索词

`KeywordExtractor` 从任务描述中提取搜索关键词：

```python
class KeywordExtractor:
    """从任务描述中提取有意义的搜索关键词。"""

    # 停用词：对搜索无意义的词
    STOP_WORDS = {
        "the", "a", "an", "is", "are", "was", "were",
        "in", "on", "at", "to", "for", "of", "and",
        "implement", "add", "create", "build", "make",
        # 开发领域通用动词，搜索价值低
        "function", "method", "class", "module",
    }

    def extract_keywords(self, task: str) -> list[str]:
        """
        提取关键词策略：
        1. 分词（按空格和标点）
        2. 过滤停用词
        3. 过滤短词（< 3 字符）
        4. 提取驼峰/下划线命名中的词（camelCase → camel + Case）
        5. 返回去重后的关键词列表
        """
        words = re.split(r'[\s,;.!?()[\]{}]+', task.lower())
        keywords = set()

        for word in words:
            if len(word) < 3 or word in self.STOP_WORDS:
                continue

            # 处理驼峰命名：loginUser → login, user
            sub_words = re.sub(r'([A-Z])', r' \1', word).split()
            keywords.update(w.lower() for w in sub_words if len(w) >= 3)

            # 处理下划线：user_email → user, email
            if '_' in word:
                keywords.update(p for p in word.split('_') if len(p) >= 3)

            keywords.add(word)

        return list(keywords)
```

关键词提取的质量直接影响代码搜索的精准度。对于技术性任务，从驼峰和下划线命名中提取子词尤其重要：`validateUserEmail` 应该能搜索到包含 `email`, `user`, `validate` 的文件。

## 8.6 代码搜索：基于关键词的文件匹配

`CodeSearcher` 在服务目录中搜索包含关键词的文件：

```python
class CodeSearcher:
    """在代码文件中搜索相关内容。"""

    def search_service(
        self,
        service_path: Path,
        service_name: str,
        keywords: list[str],
    ) -> list[FileMatch]:
        matches = []

        for file_path in self._iter_code_files(service_path):
            try:
                content = file_path.read_text(encoding="utf-8", errors="ignore")
                content_lower = content.lower()

                score = 0
                matching_keywords = []
                matching_lines = []

                for keyword in keywords:
                    if keyword in content_lower:
                        count = content_lower.count(keyword)
                        score += min(count, 10)  # 单个关键词最多贡献 10 分
                        matching_keywords.append(keyword)

                        # 记录匹配行（每个关键词最多 3 行）
                        lines = content.split("\n")
                        found = 0
                        for i, line in enumerate(lines, 1):
                            if keyword in line.lower() and found < 3:
                                matching_lines.append((i, line.strip()[:100]))
                                found += 1

                if score > 0:
                    rel_path = str(file_path.relative_to(self.project_dir))
                    matches.append(FileMatch(
                        path=rel_path,
                        service=service_name,
                        reason=f"Contains: {', '.join(matching_keywords)}",
                        relevance_score=score,
                        matching_lines=matching_lines[:5],  # 最多 5 个匹配行
                    ))

            except (OSError, UnicodeDecodeError):
                continue  # 跳过无法读取的文件

        # 按相关度排序
        return sorted(matches, key=lambda m: m.relevance_score, reverse=True)
```

计分公式 `min(count, 10)` 有意设置上限：一个关键词出现 100 次的文件不应该比出现 10 次的文件得分高 10 倍。这是一个防止"关键词堆积"文件（如配置文件）得分虚高的实用技巧。

`★ Insight ─────────────────────────────────────`

**简单方法往往足够好**：`CodeSearcher` 使用的是最简单的关键词频率统计，不是 BM25 或向量搜索。对于代码文件搜索，关键词匹配通常已经足够精准——代码文件中的变量名、函数名高度一致，不像自然语言那样多义。在选择算法时，先用简单的，当它不够用时再升级。

`─────────────────────────────────────────────────`

## 8.7 文件分类：修改 vs 参考

搜索到的文件需要区分两种角色：

```python
class FileCategorizer:
    """
    将搜索结果分类为：
    - files_to_modify：智能体可能需要修改的文件
    - files_to_reference：智能体需要了解但不修改的文件
    """

    def categorize_matches(
        self,
        matches: list[FileMatch],
        task: str,
    ) -> tuple[list[dict], list[dict]]:
        """
        分类规则：

        files_to_modify（高概率修改）：
        - 文件名/路径包含任务关键词
        - 文件本身是功能实现文件（handlers.py, services.py, views.py）
        - 相关度分数最高的几个文件

        files_to_reference（参考资料）：
        - 模型定义（models.py, schemas.py）
        - 配置文件（config.py, settings.py）
        - 工具函数（utils.py, helpers.py）
        - 测试文件（test_*.py）
        """
        to_modify = []
        to_reference = []

        # 识别实现文件的模式
        IMPL_PATTERNS = ["handler", "service", "controller", "view", "route", "api"]
        REF_PATTERNS = ["model", "schema", "config", "setting", "util", "helper", "test"]

        for match in matches:
            file_lower = match.path.lower()

            if any(p in file_lower for p in IMPL_PATTERNS):
                to_modify.append(self._format_match(match))
            elif any(p in file_lower for p in REF_PATTERNS):
                to_reference.append(self._format_match(match))
            elif match.relevance_score > MODIFY_THRESHOLD:
                # 高相关度但无法确定角色，归为待修改
                to_modify.append(self._format_match(match))
            else:
                to_reference.append(self._format_match(match))

        # 限制数量（防止上下文溢出）
        return to_modify[:10], to_reference[:15]
```

这个分类影响提示词的措辞："请修改以下文件" vs "以下文件供参考"。它也影响智能体的工具调用：参考文件只需读取，修改文件需要读取+编辑。

## 8.8 模式发现：从代码中提取约定

项目中有大量隐式约定——你只有读过足够多代码才知道：

- 所有 API handler 都用 `@require_auth` 装饰器
- 数据库操作都通过 `db.session` 上下文管理器
- 错误都用 `raise HTTPException(status_code=...)` 抛出

`PatternDiscoverer` 从参考文件中提取这些模式：

```python
class PatternDiscoverer:
    """从代码文件中发现并提取代码模式。"""

    # 需要发现的模式类型
    PATTERN_EXTRACTORS = {
        "decorator_patterns": r"@(\w+)\(",
        "error_patterns": r"raise\s+(\w+Exception|HTTPException)",
        "import_patterns": r"^from\s+(\S+)\s+import",
        "db_patterns": r"(db\.\w+|session\.\w+)",
        "auth_patterns": r"(@require_auth|@login_required|@jwt_required)",
    }

    def discover_patterns(
        self,
        reference_files: list[dict],
        keywords: list[str],
    ) -> dict[str, str]:
        """
        扫描参考文件，提取频繁出现的代码模式。

        返回格式：
        {
            "decorator_usage": "@require_auth is used in 8/10 handler files",
            "error_handling": "Use raise HTTPException(status_code=...) for API errors",
            "database": "Database operations use db.session context manager",
        }
        """
        pattern_counts = defaultdict(Counter)

        for file_info in reference_files:
            file_path = self.project_dir / file_info["path"]
            try:
                content = file_path.read_text(encoding="utf-8", errors="ignore")
            except OSError:
                continue

            for pattern_type, regex in self.PATTERN_EXTRACTORS.items():
                matches = re.findall(regex, content, re.MULTILINE)
                for match in matches:
                    pattern_counts[pattern_type][match] += 1

        # 将统计结果转化为自然语言描述
        return self._format_patterns(pattern_counts, len(reference_files))
```

这个模式发现机制不完美（基于正则，有误判），但它解决了一个真实问题：让智能体了解项目的隐式约定，而不是让它猜测或重新发明轮子。

## 8.9 TaskContext：上下文的数据结构

所有上下文信息聚合为 `TaskContext`：

```python
@dataclass
class TaskContext:
    """任务的完整上下文。"""

    task_description: str          # 任务描述（原文）
    scoped_services: list[str]     # 推断的相关服务列表
    files_to_modify: list[dict]    # 可能需要修改的文件
    files_to_reference: list[dict] # 需要参考的文件
    patterns_discovered: dict[str, str]  # 发现的代码模式
    service_contexts: dict[str, dict]    # 每个服务的概要信息
    graph_hints: list[dict]        # 来自知识图谱的历史提示
```

这个数据结构会被序列化后注入到 Coder 智能体的提示词中：

```markdown
## Task Context

**Task:** 实现用户登录 API，支持邮箱密码验证

**Relevant Services:** auth, user

**Files You Will Likely Modify:**
- src/auth/handlers.py — Contains: login, user, auth (score: 23)
- src/auth/routes.py — Contains: auth, password (score: 15)

**Reference Files (Do Not Modify):**
- src/models/user.py — Contains: user, email (score: 18)
- src/auth/utils.py — Contains: password, hash (score: 12)

**Discovered Patterns:**
- decorator_usage: @require_auth is used in 8/10 handler files
- error_handling: Use raise HTTPException(status_code=...) for API errors

**Historical Context (from memory):**
- [gotcha] JWT library requires utc=True, otherwise timezone issues in production
- [pattern] Password hashing uses bcrypt with cost factor 12
```

## 8.10 上下文窗口的预算管理

上下文不是越多越好——过多的上下文会：
1. 消耗 token，增加成本
2. 让模型在大量信息中迷失，降低输出质量
3. 可能触发上下文窗口限制

Auto-Claude 在多个层面控制上下文体积：

| 组件 | 限制 |
|------|------|
| `files_to_modify` | 最多 10 个 |
| `files_to_reference` | 最多 15 个 |
| `matching_lines` per file | 最多 5 行 |
| 每行截断长度 | 100 字符 |
| `graph_hints` | 最多 10 条（`MAX_CONTEXT_RESULTS`）|
| `patterns` | 最多 5 种模式类型 |

这些硬性限制确保 `TaskContext` 的序列化结果在一个合理的 token 范围内（通常 2000-5000 token），为真正的工作（代码生成）留下充足的上下文空间。

## 8.11 File Fuzzy Matching：智能体的自我纠错

即使有了良好的上下文，Planner 生成的文件路径有时会出错（文件名拼写错误、目录层次错误）。`agents/coder.py` 实现了一个路径模糊匹配系统：

```python
class CoderAgent:
    """
    Coder 智能体，包含文件路径模糊匹配能力。
    """

    def _build_file_index(self, worktree_path: Path) -> dict[str, Path]:
        """构建文件名到完整路径的索引。"""
        index = {}
        for file_path in worktree_path.rglob("*"):
            if file_path.is_file():
                # 多种索引键：完整路径、文件名、相对路径
                index[str(file_path)] = file_path
                index[file_path.name] = file_path
                index[str(file_path.relative_to(worktree_path))] = file_path
        return index

    def _score_and_select(
        self,
        target: str,
        file_index: dict,
    ) -> Optional[Path]:
        """
        模糊匹配：找到最接近的文件路径。

        评分维度：
        - 精确匹配：+100
        - 文件名匹配：+50
        - 路径包含目标：+30
        - 编辑距离相近：+20
        """
        best_score = 0
        best_path = None

        target_lower = target.lower()
        target_name = os.path.basename(target_lower)

        for key, path in file_index.items():
            key_lower = key.lower()
            score = 0

            if key_lower == target_lower:
                return path  # 精确匹配，直接返回

            if os.path.basename(key_lower) == target_name:
                score += 50

            if target_lower in key_lower or key_lower in target_lower:
                score += 30

            # 简单编辑距离估算
            common_chars = len(set(key_lower) & set(target_lower))
            score += common_chars * 2

            if score > best_score:
                best_score = score
                best_path = path

        # 只有分数足够高才返回（避免误匹配）
        return best_path if best_score >= 20 else None
```

这个自动纠错机制减少了因路径错误导致的实现失败。Planner 可以以高层次的方式描述文件路径，Coder 会自动找到实际文件。

`★ Insight ─────────────────────────────────────`

**智能体系统中的错误传播**：如果 Planner 输出了错误的文件路径，而 Coder 直接使用，结果是 Coder 在错误的位置创建了新文件（而不是修改现有文件），导致 QA 发现"功能没有实现"。模糊匹配在信息传递边界提供了一层容错，将"路径拼写错误"从"任务失败"降级为"自动纠正"。

`─────────────────────────────────────────────────`

## 8.12 上下文构建的完整流程

以任务"实现用户登录 API"为例：

```
输入：task = "实现用户登录 API，支持邮箱密码验证"
         │
         ▼
1. KeywordExtractor
   → ["login", "api", "user", "email", "password", "验证", "邮箱"]
         │
         ▼
2. ServiceMatcher
   → ["auth", "user"]  (基于关键词匹配项目索引)
         │
         ▼
3. CodeSearcher (对 auth/ 和 user/ 目录搜索)
   → auth/handlers.py: score=23 (login×5, user×8, auth×10)
   → auth/routes.py: score=15
   → models/user.py: score=18
   → auth/utils.py: score=12
   → tests/test_auth.py: score=8
         │
         ▼
4. FileCategorizer
   → files_to_modify: [handlers.py, routes.py]
   → files_to_reference: [models/user.py, utils.py, tests/]
         │
         ▼
5. PatternDiscoverer (分析参考文件)
   → decorator_usage: "@require_auth used in 8 handler files"
   → error_handling: "raise HTTPException(status_code=...)"
   → password: "bcrypt with cost=12"
         │
         ▼
6. Graphiti (历史知识图谱)
   → gotcha: "JWT needs utc=True"
   → pattern: "Rate limiting: 100 req/min via Redis"
         │
         ▼
输出：TaskContext（~3500 tokens）
   → 注入到 Coder 提示词
```

整个过程在几秒内完成，为 Coder 省去了自己探索代码库的大量时间（也省去了对应的 token 消耗）。

## 8.13 上下文注入的时机

上下文不是一次性注入的。Auto-Claude 在不同阶段注入不同层次的上下文：

| 阶段 | 注入的上下文 | 原因 |
|------|------------|------|
| Spec Writer | 项目结构 + 现有功能摘要 | 理解现状，写出贴合实际的规格 |
| Planner | 任务规格 + 相关文件列表 | 制定可行的实现计划 |
| Coder | 完整 TaskContext + 子任务详情 | 精准实现 |
| QA Reviewer | 实现文件列表 + 规格文档 | 对照规格验收 |
| QA Fixer | QA 报告 + 错误文件 | 精准修复 |

每个角色只得到它需要的上下文。QA Reviewer 不需要项目历史；Coder 不需要整个规格文档（只需当前子任务的部分）。这种精准的上下文分发是多智能体系统效率的关键来源之一。

---

## Lab 8.1：构建任务上下文引擎

在这个实验中，你将实现一个完整的上下文构建系统，为假想的代码助手提供任务相关的精准上下文。

### Part A：关键词提取与文件搜索

```python
# lab_08_context.py

import re
import os
from pathlib import Path
from dataclasses import dataclass, field
from collections import Counter, defaultdict
from typing import Optional

@dataclass
class FileMatch:
    path: str
    score: float
    matching_keywords: list[str]
    sample_lines: list[str]  # 最多 3 行匹配样本

class KeywordExtractor:
    STOP_WORDS = {
        "the", "a", "an", "is", "are", "in", "on", "for",
        "implement", "add", "create", "build", "make", "use",
    }

    def extract(self, task: str) -> list[str]:
        """
        TODO Part A: 从任务描述提取搜索关键词

        要求：
        1. 按空格和标点分词
        2. 过滤停用词和短词（< 3字符）
        3. 处理驼峰和下划线命名（loginUser → login, user）
        4. 去重，返回关键词列表

        示例：
        "implement user loginAPI with emailValidation"
        → ["user", "login", "api", "email", "validation", "loginapi", "emailvalidation"]
        """
        pass


class CodeSearcher:
    # 跳过的目录
    SKIP_DIRS = {".git", "node_modules", "__pycache__", ".venv", "dist", "build"}
    # 代码文件扩展名
    CODE_EXTENSIONS = {".py", ".ts", ".js", ".tsx", ".jsx", ".go", ".java"}

    def __init__(self, project_dir: Path):
        self.project_dir = project_dir

    def search(
        self,
        keywords: list[str],
        max_results: int = 20,
    ) -> list[FileMatch]:
        """
        TODO Part A: 在项目目录中搜索包含关键词的代码文件

        要求：
        1. 递归遍历项目目录（跳过 SKIP_DIRS）
        2. 只搜索 CODE_EXTENSIONS 中的文件类型
        3. 对每个文件：
           a. 统计每个关键词出现次数
           b. 计算综合分数（每个词贡献 min(count, 10)）
           c. 记录匹配行样本（每词最多 2 行，行内容截断到 100 字符）
        4. 按分数排序，返回 max_results 个最相关文件

        注意：使用 errors="ignore" 打开文件，避免编码错误
        """
        pass
```

### Part B：文件分类与模式发现

```python
class FileCategorizer:
    # 实现文件的路径关键词
    IMPL_KEYWORDS = ["handler", "service", "controller", "view", "route", "api", "endpoint"]
    # 参考文件的路径关键词
    REF_KEYWORDS = ["model", "schema", "config", "setting", "util", "helper", "test", "type"]

    def categorize(
        self,
        matches: list[FileMatch],
        top_n_modify: int = 5,
        top_n_reference: int = 10,
    ) -> tuple[list[FileMatch], list[FileMatch]]:
        """
        TODO Part B: 将搜索结果分类为"待修改"和"参考"两组

        分类规则：
        1. 路径中包含 IMPL_KEYWORDS → 待修改
        2. 路径中包含 REF_KEYWORDS → 参考
        3. 分数前 1/3 的文件 → 待修改
        4. 其余 → 参考

        限制结果数量：
        - 待修改最多 top_n_modify 个
        - 参考最多 top_n_reference 个
        """
        pass


class PatternDiscoverer:
    """从参考文件中发现代码约定和模式。"""

    PATTERNS = {
        "decorators": r"@(\w+)\(",
        "exceptions": r"raise\s+(\w+Error|\w+Exception)",
        "imports": r"^(?:from|import)\s+(\S+)",
    }

    def discover(self, reference_files: list[FileMatch], project_dir: Path) -> dict[str, list[str]]:
        """
        TODO Part B: 从参考文件中提取频繁出现的代码模式

        要求：
        1. 对每个模式类型，统计在参考文件中出现的实例
        2. 只返回出现次数 >= 2 的模式（过滤噪音）
        3. 每种模式类型最多返回 5 个实例
        4. 返回格式：{"decorators": ["@require_auth", "@cached"], ...}
        """
        pass
```

### Part C：上下文组装与注入

```python
@dataclass
class TaskContext:
    task: str
    keywords: list[str]
    files_to_modify: list[FileMatch]
    files_to_reference: list[FileMatch]
    patterns: dict[str, list[str]]

    def to_prompt_text(self, max_chars: int = 3000) -> str:
        """
        TODO Part C: 将 TaskContext 序列化为提示词文本

        格式：
        ## Task Context

        **Keywords Identified:** login, user, email, password

        **Files to Modify:**
        - src/auth/handlers.py (score: 23.0)
          Matches: login, user, auth
          Sample: `def login_user(request):...`

        **Reference Files:**
        - src/models/user.py (score: 18.0)
          ...

        **Code Patterns:**
        - decorators: @require_auth, @cached
        - exceptions: AuthenticationError, ValidationError

        要求：
        1. 严格控制总字符数不超过 max_chars
        2. 如果内容太多，优先保留 files_to_modify，其次 reference，最后 patterns
        3. 截断时在文本末尾加上 "...[truncated]" 标记
        """
        pass


def build_context(task: str, project_dir: Path) -> TaskContext:
    """
    主入口：为任务构建完整上下文。
    """
    extractor = KeywordExtractor()
    searcher = CodeSearcher(project_dir)
    categorizer = FileCategorizer()
    discoverer = PatternDiscoverer()

    keywords = extractor.extract(task)
    matches = searcher.search(keywords)
    to_modify, to_reference = categorizer.categorize(matches)
    patterns = discoverer.discover(to_reference, project_dir)

    return TaskContext(
        task=task,
        keywords=keywords,
        files_to_modify=to_modify,
        files_to_reference=to_reference,
        patterns=patterns,
    )


def test_context_builder():
    """使用 Auto-Claude 自己的代码库测试上下文构建。"""
    import tempfile, shutil

    # 使用当前项目目录
    project_dir = Path(__file__).parent.parent  # 调整为实际项目路径

    task = "implement user authentication with JWT and session management"
    context = build_context(task, project_dir)

    print(f"关键词: {context.keywords}")
    print(f"待修改文件: {len(context.files_to_modify)} 个")
    print(f"参考文件: {len(context.files_to_reference)} 个")
    print(f"发现模式: {list(context.patterns.keys())}")
    print("\n=== 提示词文本 ===")
    prompt = context.to_prompt_text(max_chars=2000)
    print(prompt)
    print(f"\n字符数: {len(prompt)}")

    # 验证
    assert len(context.keywords) > 0, "应提取到关键词"
    assert len(context.files_to_modify) <= 5, "待修改文件不超过 5 个"
    assert len(context.files_to_reference) <= 10, "参考文件不超过 10 个"
    assert len(prompt) <= 2000, "提示词文本不超过 2000 字符"
    print("\n✓ 所有验证通过")


if __name__ == "__main__":
    test_context_builder()
```

### 验证标准

| 检查点 | 预期结果 |
|--------|---------|
| 关键词提取 | 停用词过滤，驼峰正确分词 |
| 文件搜索 | 相关文件排名靠前，无关文件不出现 |
| 文件分类 | `handlers.py` 在 modify，`models.py` 在 reference |
| 模式发现 | 高频装饰器/异常被识别 |
| 上下文字符限制 | 严格不超过 max_chars |

### 进阶挑战

1. **服务感知搜索**：读取项目的目录结构，推断服务边界（`src/auth/`, `src/payment/` 等），只在相关服务目录中搜索，提升精准度。

2. **上下文质量评估**：实现 `evaluate_context()` 函数，输入构建好的 `TaskContext` 和任务描述，返回一个 0-1 的"上下文质量分"（考虑：关键词覆盖率、文件多样性、模式数量）。

3. **增量上下文**：实现 `ContextCache`，缓存上一次构建的 `TaskContext`。当新任务与上一个任务的关键词重叠度 > 50% 时，直接复用缓存，只更新差异部分。测量性能提升。

---

**本章要点回顾**

- 上下文工程的核心问题：在有限的 token 预算内提供最相关的信息
- `ContextBuilder` 通过服务推断、关键词提取、代码搜索、文件分类、模式发现五步构建上下文
- 文件分类为"待修改"和"参考"两类，影响智能体的行为预期
- 每层设置数量上限（文件、行数、字符），防止上下文膨胀
- 模式发现让智能体了解项目隐式约定，无需自己重新发现
- Graphiti 历史记忆是上下文的第三维度，跨任务传递知识
- 文件路径模糊匹配提供了智能体系统内的容错层
- 不同角色（Planner/Coder/QA）注入不同子集的上下文，精准服务各自职责

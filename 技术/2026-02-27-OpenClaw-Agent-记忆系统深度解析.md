---
title: OpenClaw Agent 记忆系统深度解析
date: 2026-02-27
category: 技术
tags: [OpenClaw, Agent, Memory, RAG, 向量搜索, 混合搜索]
source: https://docs.openclaw.ai/concepts/memory
---

# OpenClaw Agent 记忆系统深度解析

## 核心设计理念

OpenClaw 采用**文件优先（File-First）**的内存架构：
- **文件是真相的来源** — AI Agent 只能记住写入磁盘的内容
- 使用纯 Markdown 文件存储
- 支持语义搜索

## 记忆类型与存储结构

### 1. 临时记忆（每日日志）

**位置**: `memory/YYYY-MM-DD.md`

- 每日创建的追加写入文件
- 记录日常活动、决策和上下文
- 会话启动时自动加载今天和昨天的日志

### 2. 持久记忆（精选知识库）

**位置**: `MEMORY.md`

- 精心整理的长期记忆
- 包含重要决策、偏好、项目约定
- **仅在私人会话中加载**，保护敏感信息

### 3. 会话记忆

**位置**: `sessions/YYYY-MM-DD-<slug>.md`

- 自动保存之前的对话记录
- 使用 LLM 生成的描述性 slug 作为文件名
- 可被索引和搜索，回顾过去的对话

## 记忆工具

OpenClaw 提供两个 Agent 面向的工具：

| 工具 | 功能 |
|------|------|
| `memory_search` | 语义搜索 MEMORY.md 和 memory/*.md |
| `memory_get` | 读取特定记忆文件的指定行范围 |

## 向量搜索系统

### 嵌入提供商自动选择

OpenClaw 支持多种嵌入提供商，智能自动选择：

1. **本地模式** (`local`) — 如果配置了本地模型路径且文件存在
2. **OpenAI** — 如果 OpenAI key 可用
3. **Gemini** — 如果 Gemini key 可用
4. **Voyage** — 如果 Voyage key 可用
5. **Mistral** — 如果 Mistral key 可用

### 本地嵌入模型

- 默认模型: `hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf` (~600MB)
- 使用 `node-llama-cpp` 进行本地推理
- 需要运行 `pnpm approve-builds` 批准原生编译

## 混合搜索：BM25 + 向量

OpenClaw 不依赖单一的向量相似度，而是使用**加权分数融合**结合两种检索方法：

### 向量搜索（语义匹配）

适用于概念匹配：
- "gateway host" ≈ "machine running gateway"
- "authentication flow" ≈ "login process"

### BM25 搜索（词汇匹配）

适用于精确匹配：
- 错误代码: `ERR_CONNECTION_REFUSED`
- 函数名: `handleUserAuth()`
- 唯一标识符

### 混合合并算法

```typescript
// 默认权重：70% 向量 + 30% 文本
finalScore = vectorWeight * vectorScore + textWeight * textScore
```

### MMR 重排序（多样性）

当混合搜索返回结果时，多个块可能包含相似或重叠的内容。MMR（最大边际相关性）重新排序结果以平衡相关性和多样性。

**参数控制**:
- `lambda = 1.0` → 纯相关性
- `lambda = 0.0` → 最大多样性
- 默认: `0.7`（平衡）

### 时间衰减（近期优先）

随着时间推移，久远的记忆分数会按指数衰减：

```
decayedScore = score × e^(-λ × ageInDays)
```

- 默认半衰期: 30天
- 今天的笔记: 100%
- 7天前: ~84%
- 30天前: 50%
- 90天前: ~12.5%

**常青文件不被衰减**: `MEMORY.md`、非日期文件

## 嵌入缓存

OpenClaw 可以在 SQLite 中缓存**块嵌入**，避免重复 embedding：

```json
{
  "memorySearch": {
    "cache": {
      "enabled": true,
      "maxEntries": 50000
    }
  }
}
```

### 缓存策略

- **SHA-256 哈希去重**: 相同内容 → 相同嵌入 → 缓存命中
- 跨文件去重: 相同段落嵌入一次

## 自动记忆刷新（预压缩提醒）

OpenClaw 最具创新性的功能之一是**上下文压缩前的自动记忆刷新**。

### 工作原理

当会话**接近自动压缩**时，OpenClaw 触发一个**静默的 Agent 轮次**，提醒模型在上下文被压缩之前写入持久记忆。

### 触发条件

```
currentTokens >= contextWindow - reserveTokensFloor - softThresholdTokens
```

对于 200K 上下文窗口：
```
currentTokens >= 200000 - 20000 - 4000 = 176000 tokens
```

### 配置

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 4000,
          "systemPrompt": "Session nearing compaction. Store durable memories now.",
          "prompt": "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
        }
      }
    }
  }
}
```

## QMD 后端（实验性）

设置 `memory.backend = "qmd"` 以使用 [QMD](https://github.com/tobi/qmd) 本地优先搜索：

- 组合 BM25 + 向量 + 重排序
- Markdown 保持为真相来源
- 通过 Bun + `node-llama-cpp` 完全本地运行
- 自动从 HuggingFace 下载 GGUF 模型

## Markdown 分块算法

OpenClaw 使用带有重叠保护的滑动窗口算法：

- **目标**: ~400 tokens/块 (~1600 字符)
- **重叠**: 80 tokens (~320 字符)
- **行感知**: 保留行边界和行号
- **哈希去重**: 每个块获取 SHA-256 哈希用于缓存查找

## 会话记忆集成

OpenClaw 可以自动保存和索引过去的对话，使其在未来的会话中可搜索。

### 索引策略

- **增量索引**: 基于文件哈希的增量更新
- **异步后台同步**: 默认阈值:
  - 100KB 新数据，或
  - 50 条新消息

## 何时写入记忆

- **决策、偏好和持久事实** → `MEMORY.md`
- **日常笔记和运行上下文** → `memory/YYYY-MM-DD.md`
- 如果有人说"记住这个"，把它写下来（不要保存在 RAM 中）

## 与传统 RAG 的对比

| 方面 | 传统 RAG | OpenClaw Memory |
|------|---------|----------------|
| 真相来源 | 向量数据库 | Markdown 文件 |
| 搜索方法 | 仅向量 | 混合 (BM25 + 向量) |
| 存储 | Pinecone/Weaviate/Chroma | SQLite |
| 嵌入 | 始终远程 API | 本地优先 + 回退 |
| 分块 | 固定大小 | 行感知 + 重叠 |
| 缓存 | 通常无 | SHA-256 哈希基础 |
| 更新 | 全量重索引 | 增量式 |
| 上下文保存 | 手动 | 自动预压缩刷新 |
| 人类可读 | 否 | 是（纯 Markdown） |
| 成本优化 | 有限 | 批量 API + 缓存 |

## 核心创新点

1. **文件优先理念** — 无数据库作为真相来源
2. **混合检索** — BM25 + 向量平衡精确率和召回率
3. **提供商自动选择** — 本地 → OpenAI → Gemini
4. **批量优化** — 使用折扣批量 API 节省 50% 成本
5. **缓存优先嵌入** — SHA-256 去重
6. **增量同步** — 基于 delta 的会话索引
7. **预压缩刷新** — 自动上下文 → 记忆转移
8. **每 Agent 隔离** — 独立的 SQLite 存储

## 性能特征

- **本地嵌入**: ~50 tokens/sec (M1 Mac)
- **OpenAI 嵌入**: ~1000 tokens/sec (批量)
- **搜索延迟**: <100ms (10K 块)
- **索引大小**: ~5KB/1K tokens

## 参考资料

- [OpenClaw Memory 文档](https://docs.openclaw.ai/concepts/memory)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [sqlite-vec Extension](https://github.com/asg017/sqlite-vec)
- [QMD](https://github.com/tobi/qmd)

---

# 社区实践：混合记忆系统设计

> 以下内容整理自 Reddit r/OpenClaw 社区的经验分享

## 背景：每个人都会遇到的问题

使用 AI 助手时，每个对话都需要重新开始。你需要反复解释相同的上下文。即使在长会话中，上下文压缩也会悄悄吃掉你早期的消息。Agent 正在很好地工作，对话正在进行，然后突然它"忘记"了你在 20 条消息前说的内容，因为上下文窗口被压缩了。

社区称之为**上下文压缩遗忘症 (context compression amnesia)**。

## 迭代方案

### 尝试 #1：大 Markdown 文件

最简单的方案：一个 `MEMORY.md` 文件，在每个回合注入到系统提示中。

```markdown
## Identity
- Name: Adam
- Location: USA

## Projects
- Clawdbot: Personal AI assistant on home server
```

**效果**: 对于少量核心事实效果不错。**问题**: 无法扩展，每个 token 都要花钱。

### 尝试 #2：向量搜索 (LanceDB)

下一步很自然：向量数据库。基本思路：将记忆转换为数值向量（嵌入），存储它们，当新消息到来时，也将其转换为向量，找到最相似的记忆。

**优点**:
- Agent 突然可以回忆旧对话中的内容

**问题**:
1. **精度问题**: 问"我女儿的生日是多少？"可能返回三个芭蕾相关的块，而不是生日条目
2. **成本和延迟**: 每次存储和检索都需要 API 调用，每次对话需要 2 次嵌入调用
3. **分块问题**: 块边界决策很重要，不好的分割可能把关键事实分散到两个向量中

### 尝试 #3：混合系统

最终方案：**SQLite + FTS5 处理结构化事实 + LanceDB 处理语义搜索**

```
用户消息到达
  ↓
SQLite FTS5 搜索事实表（毫秒级，免费）
  ↓
LanceDB 嵌入查询 + 向量相似度（~200ms，1次API调用）
  ↓
结果合并、去重、按复合分数排序
  ↓
Top 结果注入 Agent 上下文
```

**解决的问题**:
- 事实查询命中 SQLite，返回精确匹配
- 大多数查询不接触嵌入 API，无成本
- 结构化事实不需要分块

## 社区洞察

### 不是所有记忆都应该永久保存

有些信息：
- "我正在整理早间简报日程" — 现在有用，下周就无关了
- "我女儿的生日是 6 月 3 日" — 应该永远保留

**解决方案：记忆衰减分类系统**

| 层级 | 示例 | TTL |
|------|------|-----|
| 永久 | 名字、生日、API端点、架构决策 | 永不过期 |
| 稳定 | 项目详情、关系、技术栈 | 90天 |
| 活跃 | 当前任务、冲刺目标 | 14天 |
| 会话 | 调试上下文、临时状态 | 24小时 |
| 检查点 | 飞行前状态保存 | 4小时 |

**关键**: 访问时刷新 TTL。如果"稳定"事实不断被检索，其过期计时器会重置。

### 决策比对话更持久

社区成员追踪了 37,000 个知识向量和 5,400 个提取的事实。模式：**将记忆压缩为决策，而不是原始对话日志**。

> "我们选择 SQLite + FTS5 而不是纯 LanceDB，因为 80% 的查询是结构化查找"

系统自动检测决策语言并提取为永久结构化事实：

- "我们决定使用 X 因为 Y" → entity: decision, key: X, value: Y
- "为 Z 选择 X 而不是 Y" → entity: decision, key: X over Y, value: Z
- "永远/从不做 X" → entity: convention, key: X, value: always/never

### 飞行前检查点

另一个社区模式：在危险操作前保存状态。如果 Agent 要执行多步骤任务（编辑文件、运行构建、部署），它会保存检查点：当前状态、预期结果、修改的文件。

如果上下文压缩在任务中途发生、会话崩溃、或 Agent 失去线索，检查点可以恢复。这解决了 Clawdbot 最大的痛点——**短期记忆丧失**。

### 每日文件扫描

最后一个功能：扫描每日记忆日志文件并从中提取结构化事实。

```bash
# 试运行 - 查看会提取什么
clawdbot hybrid-mem extract-daily --dry-run --days 14

# 实际存储提取的事实
clawdbot hybrid-mem extract-daily --days 14
```

## 经验总结

### 如果从头开始会怎么做

1. **从 SQLite 开始，而不是向量**
   向量搜索看起来是"AI 原生"的方法，但对个人助手来说，大多数记忆查询是结构化查找。SQLite + FTS5 从第一天就能覆盖 80% 的需求。

2. **从设计之初就考虑衰减**
   否则会积累污染检索结果的过时事实。

3. **从一开始就明确提取决策**
   原始对话日志是噪音，带有理据的提炼决策从根本上更清晰。

## 关键洞察

> "构建一个好的'记忆'系统不是一件事——它是具有不同特征的多个系统服务于不同的查询模式。"
>
> 向量搜索对模糊语义召回很棒，但对大多数个人助手实际需要的事实查找来说，它昂贵且不精确。

**混合方法覆盖全谱**:
- 结构化存储 → 精确事实
- 向量搜索 → 上下文召回
- 始终加载的上下文 → 关键信息
- 时间感知衰减 → 管理新鲜度

## 依赖参考

| 包 | 版本 | 用途 |
|----|------|------|
| better-sqlite3 | 11.0.0 | 带 FTS5 的 SQLite 驱动 |
| @lancedb/lancedb | 0.23.0 | 嵌入式向量数据库 |
| openai | 6.16.0 | 生成嵌入的 OpenAI SDK |

**API Keys**:
- `OPENAI_API_KEY` — 通过 text-embedding-3-small 生成嵌入
- `SUPERMEMORY_API_KEY` — 云归档（可选）

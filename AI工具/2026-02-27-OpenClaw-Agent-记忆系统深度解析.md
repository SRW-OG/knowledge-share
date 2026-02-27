---
title: OpenClaw Agent 记忆系统深度解析
date: 2026-02-27
category: 技术
tags: [OpenClaw, Agent, Memory, RAG, 向量搜索, 混合搜索]
source: https://docs.openclaw.ai/concepts/memory
---

# OpenClaw Agent 记忆系统深度解析

> 整理自：OpenClaw 官方文档 + Reddit r/OpenClaw 社区实践

---

## 一、核心设计理念

OpenClaw 采用**文件优先（File-First）**的内存架构：
- **文件是真相的来源** — AI Agent 只能记住写入磁盘的内容
- 使用纯 Markdown 文件存储
- 支持语义搜索

> "Files are the source of truth — the model only 'remembers' what gets written to disk."

---

## 二、记忆类型与存储结构

### 1. 三层记忆架构（社区实践）

根据 Reddit 社区的实践，成功的记忆系统通常包含**三层**：

| 层级 | 说明 | 存储 |
|------|------|------|
| **Fact Layer（事实层）** | 永久保存的数据：名字、生日、API端点、架构决策 | SQLite + MEMORY.md |
| **Meta Layer（元数据层）** | 项目详情、关系、技术栈、偏好 | memory/*.md |
| **Runtime Layer（运行时层）** | 当前任务状态、调试上下文、检查点 | 内存/临时文件 |

### 2. OpenClaw 原生记忆文件

**临时记忆（每日日志）**
- 位置: `memory/YYYY-MM-DD.md`
- 每日创建的追加写入文件
- 会话启动时自动加载今天和昨天的日志

**持久记忆（精选知识库）**
- 位置: `MEMORY.md`
- 精心整理的长期记忆
- **仅在私人会话中加载**，保护敏感信息

**会话记忆**
- 位置: `sessions/YYYY-MM-DD-<slug>.md`
- 自动保存之前的对话记录
- 可被索引和搜索

---

## 三、搜索技术

### 1. 混合搜索：BM25 + 向量

OpenClaw 使用**加权分数融合**结合两种检索方法：

```
finalScore = vectorWeight × vectorScore + textWeight × textScore
```

- **向量搜索（70%）**: 语义匹配，适用于概念相近的查询
- **BM25 搜索（30%）**: 词汇匹配，适用于精确查询

### 2. MMR 重排序（多样性）

平衡相关性和多样性，避免重复结果：
- `lambda = 1.0` → 纯相关性
- `lambda = 0.0` → 最大多样性
- 默认: `0.7`

### 3. 时间衰减（近期优先）

```
decayedScore = score × e^(-λ × ageInDays)
```

- 默认半衰期: 30天
- 今天的笔记: 100%
- 30天前: 50%
- 90天前: ~12.5%

### 4. 嵌入提供商

| 提供商 | 说明 |
|--------|------|
| **本地 (local)** | node-llama-cpp，默认模型 ~600MB |
| **OpenAI** | text-embedding-3-small |
| **Gemini** | gemini-embedding-001 |
| **Voyage** | Voyage 专用模型 |
| **Mistral** | Mistral 专用模型 |

**社区发现**: Gemma 300M 模型出奇地好，CPU 即可运行，无需 GPU。

---

## 四、社区实践：混合记忆系统

### 迭代历程

**尝试 #1**: 大 Markdown 文件
- 简单有效，但无法扩展
- 每个 token 都要花钱

**尝试 #2**: 纯向量搜索 (LanceDB)
- 问题：精度低、成本高、分块难

**尝试 #3**: 混合系统（最终方案）
- SQLite + FTS5 → 结构化事实
- LanceDB → 语义搜索

### 检索流程

```
用户消息
  ↓
SQLite FTS5 搜索事实表（毫秒级，免费）
  ↓
向量嵌入 + 语义搜索（~200ms）
  ↓
结果合并、去重、按分数排序
  ↓
Top 结果注入上下文
```

---

## 五、记忆衰减分类系统

社区提出的 5 层 TTL 系统：

| 层级 | 示例 | TTL |
|------|------|-----|
| **永久** | 名字、生日、API端点、架构决策 | 永不过期 |
| **稳定** | 项目详情、关系、技术栈 | 90天 |
| **活跃** | 当前任务、冲刺目标 | 14天 |
| **会话** | 调试上下文、临时状态 | 24小时 |
| **检查点** | 飞行前状态保存 | 4小时 |

**关键机制**: 访问时自动刷新 TTL

---

## 六、决策提取

社区发现：**决策比对话更持久**

自动检测决策语言并提取为永久结构化事实：

```
"我们决定使用 X 因为 Y" → entity: decision, key: X, value: Y
"选择 X 而不是 Y" → entity: decision, key: X over Y, value: Z
"永远/从不做 X" → entity: convention, key: X, value: always/never
```

---

## 七、飞行前检查点

在执行危险操作前保存状态：
- 当前状态
- 预期结果
- 修改的文件

**解决最大痛点**: 短期记忆丧失

---

## 八、自动记忆刷新

OpenClaw 最具创新性的功能：**上下文压缩前的静默提醒**

### 触发条件
```
currentTokens >= contextWindow - reserveTokensFloor - softThresholdTokens
```

### 配置
```json
{
  "compaction": {
    "memoryFlush": {
      "enabled": true,
      "softThresholdTokens": 4000,
      "prompt": "Write any lasting notes to memory/YYYY-MM-DD.md"
    }
  }
}
```

---

## 九、QMD 后端（实验性）

- 组合 BM25 + 向量 + 重排序
- 通过 Bun + node-llama-cpp 本地运行
- 自动从 HuggingFace 下载模型

---

## 十、与传统 RAG 对比

| 方面 | 传统 RAG | OpenClaw Memory |
|------|---------|----------------|
| 真相来源 | 向量数据库 | Markdown 文件 |
| 搜索方法 | 仅向量 | 混合 (BM25 + 向量) |
| 存储 | Pinecone/Weaviate | SQLite |
| 嵌入 | 始终远程 | 本地优先 + 回退 |
| 分块 | 固定大小 | 行感知 + 重叠 |
| 缓存 | 无 | SHA-256 哈希 |
| 更新 | 全量重索引 | 增量式 |
| 上下文保存 | 手动 | 自动预压缩刷新 |
| 人类可读 | 否 | 是 |

---

## 十一、核心创新点

1. **文件优先** — 无数据库作为真相来源
2. **混合检索** — BM25 + 向量平衡精确率和召回率
3. **提供商自动选择** — 本地 → OpenAI → Gemini
4. **批量优化** — 批量 API 节省 50% 成本
5. **缓存优先** — SHA-256 去重
6. **增量同步** — 基于 delta 的会话索引
7. **预压缩刷新** — 自动上下文 → 记忆转移
8. **每 Agent 隔离** — 独立的 SQLite 存储
9. **记忆衰减** — 5 层 TTL 分类
10. **决策提取** — 永久结构化事实

---

## 十二、性能特征

| 指标 | 数值 |
|------|------|
| 本地嵌入 | ~50 tokens/sec (M1 Mac) |
| OpenAI 嵌入 | ~1000 tokens/sec (批量) |
| 搜索延迟 | <100ms (10K 块) |
| 索引大小 | ~5KB/1K tokens |

---

## 十三、依赖参考

| 包 | 版本 | 用途 |
|----|------|------|
| better-sqlite3 | 11.0.0 | FTS5 全文搜索 |
| @lancedb/lancedb | 0.23.0 | 向量数据库 |
| openai | 6.16.0 | 嵌入生成 |

---

## 参考资料

- [OpenClaw Memory 文档](https://docs.openclaw.ai/concepts/memory)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [Reddit r/OpenClaw - Permanent Memory](https://www.reddit.com/r/openclaw/comments/1r49r9m/give_your_openclaw_permanent_memory/)
- [Reddit r/OpenClaw - Tips](https://www.reddit.com/r/openclaw/comments/1r71you/things_i_wish_someone_told_me_before_i_almost/)
- [sqlite-vec Extension](https://github.com/asg017/sqlite-vec)
- [QMD](https://github.com/tobi/qmd)

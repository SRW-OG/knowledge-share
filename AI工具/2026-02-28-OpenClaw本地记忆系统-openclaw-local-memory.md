---
title: OpenClaw 本地记忆系统 - openclaw-local-memory
date: 2026-02-28
category: AI/Agent
tags: [OpenClaw, 记忆系统, 向量数据库, sqlite-vec, nomic-embed]
source: https://github.com/tituss-bit/openclaw-local-memory
---

# OpenClaw 本地记忆系统 - openclaw-local-memory

## 1. 项目概述

**openclaw-local-memory** 是一个零成本、本地优先的 AI 助手长期记忆系统。可以索引整个聊天历史并进行语义搜索——无需云 API、无需 token 费用、无隐私问题。

### 核心特性

- 导出 Telegram 聊天记录为可搜索的 markdown 块
- 使用 **nomic-embed-text-v1.5** embeddings + **sqlite-vec** 本地索引
- 支持跨机器通过 git 同步记忆
- 无需 GPU，CPU 即可运行

---

## 2. 技术方案对比

### 2.1  Embedding 模型测试结果

在真实双语（俄语/英语）聊天数据（157 块，7178 条消息）上的测试结果：

| 模型 | 大小 | 索引时间 | 平均 Top-1 得分 |
|------|------|----------|----------------|
| **nomic-embed-text v1.5** | 84MB | 2.4s | **0.69** |
| EmbeddingGemma 300M | ~200MB | 3.6s | 0.60 |
| Qwen3-Embedding 0.6B | 639MB | 7.6s | 0.56 |
| nomic-embed-text v2 MoE | 512MB | 1.7s | 0.37 |
| jina-embeddings-v5-small | 639MB | 4.9s | 0.35 |

**结论**：nomic-embed-text v1.5 在多语言对话数据上体积最小、速度最快、准确率最高。

### 2.2 与当前架构对比

| 组件 | 当前方案 (MEMORY.md) | openclaw-local-memory |
|------|---------------------|-----------------------|
| 向量索引 | FAISS | sqlite-vec |
| Embedding 模型 | BGE-Small-ZH (512维) | nomic-embed-text-v1.5 |
| 准确率 | 未测试 | 0.69 |
| 同步方式 | 手动/脚本 | Git |

---

## 3. 架构设计

### 3.1 数据流

```
用户 (Telegram) → 导出 JSON → 分块脚本 → memory/tg_history/*.md
                                              ↓
                                    openclaw memory index
                                              ↓
                                    nomic-embed-text v1.5
                                              ↓
                                    sqlite-vec (本地数据库)
                                              ↓
                                    语义搜索 (免费、即时)
```

### 3.2 性能数据

| 消息数 | 分块数 | 索引时间 | 搜索质量 |
|--------|--------|----------|----------|
| 7K | 157 | 2.4s | Excellent |
| 100K | ~2,000 | ~30s | Excellent |
| 1M+ | ~20,000 | ~5min | Good |

---

## 4. 实施步骤

### 4.1 Telegram 数据导出

**Windows**: Telegram Desktop → Chat → ⋮ → Export chat history → JSON

**macOS/Linux**: 使用内置 Telethon 脚本

```bash
pip install telethon
python scripts/tg_export.py --api-id YOUR_API_ID --api-hash YOUR_API_HASH --chat BOT_USERNAME
```

获取 API 凭证: https://my.telegram.org/apps

### 4.2 数据分块

```bash
python scripts/tg_to_chunks.py export/result.json memory/tg_history/
```

这会创建每日 markdown 文件（每块约 50 条消息），优化用于嵌入搜索。

### 4.3 配置 OpenClaw

在 `openclaw.json` 中添加：

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "local",
        "enabled": true,
        "local": {
          "modelPath": "hf:nomic-ai/nomic-embed-text-v1.5-GGUF/nomic-embed-text-v1.5.Q8_0.gguf"
        }
      }
    }
  }
}
```

### 4.4 建立索引

```bash
openclaw memory index --force
```

### 4.5 Git 同步（跨机器）

**机器 A（源）**:
```bash
cd memory/
git init && git add -A && git commit -m "init"
git remote add origin user@server:~/memory.git
git push -u origin main
```

**机器 B**:
```bash
git clone user@server:~/memory.git ~/.openclaw/workspace/memory
```

**自动同步**:
```bash
# crontab (每5分钟)
*/5 * * * * cd ~/.openclaw/workspace/memory && git pull origin main --quiet
```

```bash
# .git/hooks/post-merge
#!/bin/bash
openclaw memory index --force &>/dev/null &
```

---

## 5. 系统要求

- [OpenClaw](https://github.com/openclaw/openclaw) (任意近期版本)
- Node.js 20+
- Python 3.10+ (用于导出/分块脚本)
- ~100MB 磁盘空间（嵌入模型）

**无需 GPU**，CPU 即可运行。

---

## 6. Phase 2 规划（未来路线）

作者正在构建 **Vigil v2**——全自主记忆系统：

- 知识图谱 (Kùzu) 用于实体关系
- 混合搜索 (向量 + BM25 关键词 via SQLite FTS5)
- 本地 LLM 实体提取 (Qwen 3)
- 神经信号（多巴胺/皮质醇指标用于记忆显著性）

---

## 7. 总结与建议

### 7.1 价值评估

| 维度 | 评分 | 说明 |
|------|------|------|
| 轻量化 | ⭐⭐⭐⭐⭐ | 84MB 模型，CPU 可运行 |
| 准确性 | ⭐⭐⭐⭐ | Top-1 得分 0.69 |
| 隐私性 | ⭐⭐⭐⭐⭐ | 100% 本地，无云 API |
| 同步能力 | ⭐⭐⭐⭐ | Git 同步机制成熟 |

### 7.2 迁移建议

对于当前使用 FAISS + BGE-Small-ZH 的方案，可以考虑：

1. **模型替换**：nomic-embed-text-v1.5 在多语言对话数据上表现更优
2. **索引替换**：sqlite-vec 比 FAISS 更轻量
3. **同步增强**：引入 Git 同步机制

### 7.3 适用场景

- 个人 AI 助手记忆
- 多机器工作流同步
- 隐私敏感型应用
- 低资源环境部署

---

## 参考资源

- 项目地址: https://github.com/tituss-bit/openclaw-local-memory
- OpenClaw 官方: https://github.com/openclaw/openclaw
- 模型下载: https://huggingface.co/nomic-ai/nomic-embed-text-v1.5-GGUF

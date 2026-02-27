---
title: OpenClaw Agent 记忆系统部署指南
date: 2026-02-27
category: 技术
tags: [OpenClaw, Agent, Memory, FAISS, 部署指南]
source: 实践总结
---

# OpenClaw Agent 记忆系统部署指南

## 1. 系统概述

本记忆系统是一套**轻量化、本地化、免费**的 Agent 记忆解决方案，专为低配电脑和国内环境优化。

### 核心特性

| 特性 | 说明 |
|------|------|
| **本地存储** | 所有数据存储在本地，不依赖云端 |
| **免费使用** | 无需付费 API，使用本地向量模型 |
| **自动脱敏** | 发送云端前自动过滤敏感信息 |
| **多 Agent 支持** | 支持为每个 Agent 创建独立索引 |

### 技术架构

```
用户提问
    │
    ▼
┌─────────────────────────────────────────────┐
│           本地 FAISS 向量索引                │
│  • BGE-Small-ZH (512维, ~100MB)           │
│  • 搜索延迟 <100ms                         │
└─────────────────────────────────────────────┘
    │
    ├── 每日记忆: ~/.openclaw/workspace/memory/
    │
    └── 知识库: ~/Documents/Obsidian/knowledge-share/
```

---

## 2. 主 Agent (红移先知) 部署

### 2.1 目录结构

```
~/.openclaw/workspace/
├── scripts/memory/
│   ├── sync_memory.py    # 主脚本
│   ├── run.sh           # 启动脚本
│   ├── memory.sh        # 便捷脚本
│   ├── memory.py        # Python模块
│   └── venv/           # 虚拟环境
├── memory_index.faiss   # 向量索引
└── docs_mapping.json   # 文档映射
```

### 2.2 搜索范围

| 来源 | 路径 | 内容 |
|------|------|------|
| **memory/** | `~/.openclaw/workspace/memory/` | 个人每日记忆 |
| **knowledge** | `~/Documents/Obsidian/knowledge-share/` | 知识库文档 |

### 2.3 使用方法

```bash
# 构建索引
./scripts/memory/run.sh build

# 搜索
./scripts/memory/run.sh search "关键词"
```

### 2.4 触发场景

当用户问题涉及以下内容时，Agent 会主动调用搜索：

- "之前讨论过什么？"
- "上次说的项目怎样了？"
- "我让你记住的是？"
- 技术问题查找历史解决方案

---

## 3. NetSec_HR_Bot 部署

### 3.1 目录结构

```
~/.openclaw/agents/NetSec_HR_Bot/
├── memory/
│   └── MEMORY.md       # 专属记忆
├── scripts/
│   ├── memory_sync.py  # 索引脚本
│   └── memory.sh       # 启动脚本
├── memory_index.faiss  # 独立向量索引
└── docs_mapping.json  # 文档映射
```

### 3.2 使用方法

```bash
cd ~/.openclaw/agents/NetSec_HR_Bot

# 构建索引
./scripts/memory.sh build

# 搜索
./scripts/memory.sh search "候选人"
```

### 3.3 特点

- **独立索引**：与主 Agent 隔离，互不干扰
- **自动脱敏**：过滤敏感信息
- **专属记忆**：存储招聘相关记忆

---

## 4. 为新 Agent 部署步骤

### 4.1 创建独立记忆目录

```bash
# 创建 memory 目录
mkdir -p ~/.openclaw/agents/[AGENT_NAME]/memory

# 创建 MEMORY.md
cat > ~/.openclaw/agents/[AGENT_NAME]/memory/MEMORY.md << 'EOF'
---
# [Agent Name] 记忆存储

## 标签索引
| 标签 | 含义 |
|------|------|
| [tag:project-*] | 项目状态 |

---

## 当前项目

(待添加)

---

*最后更新: YYYY-MM-DD*
EOF
```

### 4.2 创建索引脚本

复制模板脚本：

```bash
cp /home/a1605/.openclaw/agents/NetSec_HR_Bot/scripts/memory_sync.py \
   ~/.openclaw/agents/[AGENT_NAME]/scripts/memory_sync.py

# 修改配置
sed -i 's/NetSec_HR_Bot/[AGENT_NAME]/g' \
   ~/.openclaw/agents/[AGENT_NAME]/scripts/memory_sync.py

# 修改路径
sed -i 's|/home/a1605/.openclaw/agents/NetSec_HR_Bot|/home/a1605/.openclaw/agents/[AGENT_NAME]|g' \
   ~/.openclaw/agents/[AGENT_NAME]/scripts/memory_sync.py
```

### 4.3 创建启动脚本

```bash
cp /home/a1605/.openclaw/agents/NetSec_HR_Bot/scripts/memory.sh \
   ~/.openclaw/agents/[AGENT_NAME]/scripts/memory.sh

chmod +x ~/.openclaw/agents/[AGENT_NAME]/scripts/memory.sh
```

### 4.4 更新 AGENTS.md

在 Agent 的 `AGENTS.md` 中添加：

```markdown
## X. 记忆搜索系统

使用专属记忆系统：

### 搜索命令
./scripts/memory.sh search "关键词"

### 触发场景
- [根据 Agent 角色自定义]
```

### 4.5 构建索引

```bash
cd ~/.openclaw/agents/[AGENT_NAME]
./scripts/memory.sh build
```

---

## 5. 自动化同步

### 5.1 定时重建索引

添加 crontab 任务：

```bash
# 每天凌晨 3 点重建主 Agent 索引
0 3 * * * cd ~/.openclaw/workspace && ./scripts/memory/run.sh build

# 每天凌晨 3 点半重建 NetSec_HR_Bot 索引
30 3 * * * cd ~/.openclaw/agents/NetSec_HR_Bot && ./scripts/memory.sh build
```

### 5.2 对话中自动触发

在 `HEARTBEAT.md` 中配置，定期触发记忆更新：

```markdown
## 定期任务

- [ ] 每周重建记忆索引
```

---

## 6. 安全机制

### 6.1 三层防火墙

| 层级 | 机制 | 说明 |
|------|------|------|
| **第一层** | Regex 过滤 | 发送前替换 API Key、密码等 |
| **第二层** | 忽略列表 | 不索引敏感文件 |
| **第三层** | 结构化提炼 | 只发送精简事实 |

### 6.2 脱敏示例

```python
def sanitize_text(text):
    patterns = {
        "KEY": r'sk-[a-zA-Z0-9]{32,}',
        "PASSWORD": r'password[:=]\s*[^\s]+',
    }
    # 替换为 [REDACTED]
```

---

## 7. 性能对比

| 指标 | 优化前 (Qdrant) | 优化后 (FAISS) |
|------|-----------------|----------------|
| **内存占用** | ~1GB | ~100MB |
| **启动时间** | 30s+ | <5s |
| **搜索延迟** | ~100ms | <50ms |
| **服务数量** | 3个 | 0个 |

---

## 8. 快速部署脚本

一键部署脚本：

```bash
#!/bin/bash
# deploy_memory.sh

AGENT_NAME=$1

if [ -z "$AGENT_NAME" ]; then
    echo "用法: ./deploy_memory.sh <Agent名称>"
    exit 1
fi

AGENT_DIR="$HOME/.openclaw/agents/$AGENT_NAME"

# 创建目录
mkdir -p "$AGENT_DIR/memory"
mkdir -p "$AGENT_DIR/scripts"

# 复制脚本
cp /home/a1605/.openclaw/agents/NetSec_HR_Bot/scripts/memory_sync.py "$AGENT_DIR/scripts/"

# 修改配置
sed -i "s/NetSec_HR_Bot/$AGENT_NAME/g" "$AGENT_DIR/scripts/memory_sync.py"
sed -i "s|/home/a1605/.openclaw/agents/NetSec_HR_Bot|$AGENT_DIR|g" "$AGENT_DIR/scripts/memory_sync.py"

# 创建启动脚本
cat > "$AGENT_DIR/scripts/memory.sh" << 'SCRIPT'
#!/bin/bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
VENV_DIR="/home/a1605/.openclaw/workspace/scripts/memory/venv"
source "$VENV_DIR/bin/activate"
python3 "$SCRIPT_DIR/memory_sync.py" "$@"
SCRIPT

chmod +x "$AGENT_DIR/scripts/memory.sh"

# 创建 MEMORY.md
cat > "$AGENT_DIR/memory/MEMORY.md" << 'EOF'
---
# $AGENT_NAME 记忆存储

---

## 当前项目

(待添加)

---

*最后更新: $(date +%Y-%m-%d)*
EOF

# 构建索引
cd "$AGENT_DIR"
./scripts/memory.sh build

echo "✅ 部署完成!"
```

使用方法：

```bash
chmod +x deploy_memory.sh
./deploy_memory.sh MyNewAgent
```

---

## 9. 相关资源

- 索引脚本: `~/.openclaw/workspace/scripts/memory/`
- 知识库: `~/Documents/Obsidian/knowledge-share/`
- 主 Agent 索引: `~/.openclaw/workspace/memory_index.faiss`

---

*最后更新：2026-02-27*

---
title: OpenClaw Agent 管理指南
date: 2026-02-25
category: AI Agents
tags: [openclaw, agent, 教程]
source: 实践总结
---

# OpenClaw Agent 管理指南

## 1 问题现状

随着 AI Agent 的应用场景增多，需要在 OpenClaw 平台上创建、管理多个专用 Agent_bot。正确管理这些 Agent 及其会话是保证工作效率的基础。

## 2 解决思路

### 方案对比

| 方案 | 优点 | 缺点 | 难度 |
|------|------|------|------|
| 直接操作目录 | 灵活，完全控制 | 需要了解目录结构 | 中 |
| 使用 CLI 命令 | 简单，快捷 | 功能有限 | 低 |
| Web UI | 可视化，直观 | 需要额外访问 | 低 |

### 推荐方案

使用 CLI 命令 + 目录结构理解相结合的方式。

## 3 最佳实践框架

### Agent 目录结构

```
~/.openclaw/agents/
├── main/                  # 默认主 Agent
│   ├── sessions/          # 会话记录
│   ├── SOUL.md
│   ├── USER.md
│   └── AGENTS.md
├── NetSec_HR_Bot/         # 自定义 Agent (大写)
│   ├── sessions/
│   ├── SOUL.md
│   ├── USER.md
│   └── AGENTS.md
└── netsec_hr_bot/         # 系统自动生成 (小写，仅会话)
```

### 必需配置文件

创建新 Agent 需要以下文件：

| 文件 | 用途 | 必需 |
|------|------|------|
| SOUL.md | Agent 规则/性格定义 | ✅ |
| USER.md | 用户画像 | ✅ |
| AGENTS.md | 工作协议 | ✅ |
| TOOLS.md | 工具配置 | 可选 |
| HEARTBEAT.md | 定时任务 | 可选 |

## 4 具体操作步骤

### 4.1 创建新 Agent

```bash
# 1. 进入 agents 目录
cd ~/.openclaw/agents

# 2. 创建 Agent 目录 (注意大小写)
mkdir -p MyNewBot

# 3. 创建必需文件
echo "# SOUL.md" > MyNewBot/SOUL.md
echo "# USER.md" > MyNewBot/USER.md
echo "# AGENTS.md" > MyNewBot/AGENTS.md
```

### 4.2 查看活跃会话

```bash
openclaw sessions
```

输出示例：
```
Sessions listed: 9
Kind   Key                        Age       Model
direct agent:main:main            just now  MiniMax-M2.5
direct agent:main:opena...n:main  22h ago   MiniMax-M2.5
```

### 4.3 切换 Agent

**方式1: 通过不同 channel**
- WebChat → main Agent
- Telegram → Telegram bot 对应的 Agent
- openclaw-tui → 启动时选择的 Agent

**方式2: Sub-agent (子任务)**
```bash
# 在当前会话中 spawn 子 Agent
sessions_spawn --task "具体任务"
```

### 4.4 会话持久化

- ✅ 默认持久化，不会自动清空
- ❌ 使用 `/new` 或 `/reset` 会创建新会话，丢失上下文
- ✅ 连续在同一 channel 对话 = 同一会话

### 4.5 删除 Agent

```bash
# 删除 Agent 目录 (包括所有会话)
rm -rf ~/.openclaw/agents/<AgentName>
```

## 5 总结与建议

### 注意事项

1. **大小写敏感**: Linux 下 `netsec_hr_bot` ≠ `NetSec_HR_Bot`
2. **避免重置**: 不要轻易使用 `/new` 或 `/reset`
3. **会话隔离**: 不同 channel 自动创建独立会话

### 维护建议

- 定期使用 `openclaw sessions` 检查活跃会话
- 不再使用的 Agent 及时清理
- 重要会话可通过导出 transcript 备份

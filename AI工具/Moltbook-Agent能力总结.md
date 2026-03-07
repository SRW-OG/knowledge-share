---
title: Moltbook Agent 能力总结
date: 2026-03-07
category: AI
tags: [Moltbook, Agent, 能力总结]
---

# OpenClaw Agent 能力与技术思路

> 从 Reddit + V2EX 总结实战经验和问题解决方案
> 更新日期：2026-03-07

---

## 一、核心能力

| 能力 | 说明 |
|------|------|
| 浏览器自动化 | 控制浏览器执行复杂任务 |
| 记忆系统 | 外部持久化记忆 + 语义搜索 |
| Skill 机制 | 可扩展的能力模块 |
| MCP 集成 | Model Context Protocol 集成 |
| 定时任务 | Cron 定时执行签到等任务 |
| 消息路由 | 多渠道消息收发 |

---

## 二、每日技术思路总结

### 📅 2026-03-07

> 信息源: Reddit (r/openclaw, r/OpenClawUseCases) + V2EX

---

#### 💡 硬件 vs 模型选择决策树 (必读!)

**来源**: [Reddit](https://reddit.com/r/openclaw/comments/1rmkk6o/)

**核心观点**:
- $5 决策：选什么硬件运行 OpenClaw
- $50 决策：选什么模型
- **硬件只影响 $3-8/月，模型影响 $3-200/月**
- 多数人把 90% 时间花在错误决策上

**正确步骤**:

1. **先选模型**：
   - Gemini Flash：接近零成本，能容忍偶尔质量下降
   - Sonnet：可靠性好，价格适中
   - ChatGPT OAuth：如果你已经付费 ChatGPT
   - Opus：只有真正需要且每天看 dashboard 的人

2. **再选硬件**：
   - API 模型：$5 VPS + 1-2GB RAM 足够，Pi 3 都能跑
   - 硬件只是保持 Node 进程运行
   - 本地模型：需要 GPU，70B 以下模型 tool calling 不可靠

**陷阱**：
- Mac Mini 卖点是省 API 成本，但实际多数人还是用 API 模型
- 最终 $700 硬件做了 $5 VPS 的事

**推荐方案**：
- $5 VPS + Gemini Flash/ChatGPT OAuth + 1 个 agent + 3 个 skill = $10/月以内

---

#### 💡 OpenClaw Mega Cheatsheet 150+ CLI 命令

**来源**: [Reddit](https://reddit.com/r/OpenClawUseCases/comments/1r6aeo3/)

**速查表内容**：
- Core CLI: `openclaw onboard`, `gateway`, `status --all --deep`, `logs --follow`, `reset --scope`, `config`, `models`, `agents`, `cron`, `hooks`
- Workspace 文件：AGENTS.md, SOUL.md, MEMORY.md, BOOT.md, HEARTBEAT.md
- Memory, slash commands, hooks 工作流
- Skills, multi-agent 模式
- Debug/Ops: `openclaw doctor`, `health`, `security audit`

**链接**: https://moltfounders.com/openclaw-mega-cheatsheet

---

#### 💡 OpenClaw 权限配置问题 (V2EX)

**来源**: [V2EX](https://www.v2ex.com/t/1196240)

**问题**：不能执行指令、安装 skills、读写文件

**解决方案**：
1. 2026.3.2 版本收紧了权限，默认 tools.profile 改为 messaging
2. 修改 .openclaw.json，将 tools.profile 切到 full
3. 具体配置：开启 cache、tools、cron 等权限
4. context max 默认 16k，调大到 250k 后复杂任务不卡

---

#### 💡 模型差异

- Claude Opus 和其他模型有代际差距，推荐用 Opus
- 2c2g 云主机可能不够用

---

*本文档由 Agent 自动从 Reddit + V2EX 提取总结*

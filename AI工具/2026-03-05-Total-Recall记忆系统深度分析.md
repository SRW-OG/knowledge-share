---
title: Total Recall 记忆系统深度分析
date: 2026-03-05
category: AI
tags: [Total Recall, 记忆系统, OpenClaw, 改进方案]
---

# Total Recall 记忆系统深度分析

> 对 GitHub 项目 gavdalf/total-recall 的深度分析，重点提取可借鉴的改进方案
> 分析日期：2026-03-05
> 项目地址：https://github.com/gavdalf/total-recall

---

## 一、项目概述

### 核心理念
- **自动观察**：无需手动保存，LLM 自主观察对话并压缩
- **五层冗余**：Observer → Reflector → Session Recovery → Reactive Watcher → Pre-compaction Hook
- **低成本**：~$0.10/月（使用 DeepSeek v3.2 via OpenRouter）
- **零维护**：No database, No vectors, No manual saves

### 与当前系统对比

| 维度 | 当前 memory 系统 | Total Recall |
|------|----------------|--------------|
| 触发方式 | 手动/定时 cron | 5层自动触发 |
| 记忆压缩 | 无 | 40-75% 压缩率 |
| 记忆整合 | 无 | Dream Cycle (夜间) |
| 重要性衰减 | 无 | 按类型自动衰减 |
| 搜索能力 | 向量搜索 | 语义钩子 + 向量 |

---

## 二、五层架构详解

### Layer 1: Observer（观察者）
**触发频率**：每 15 分钟（cron）

**工作流程**：
1. 读取最近的会话记录（JSONL）
2. 过滤掉 subagent/cron 会话
3. 提取用户 + Assistant 消息
4. MD5 哈希去重
5. 发送给 LLM（使用 observer prompt）
6. 追加到 `observations.md`（带优先级：high/medium/low）

**关键配置**：
```bash
OBSERVER_LOOKBACK_MIN=15          # 白天回看15分钟
OBSERVER_MORNING_LOOKBACK_MIN=480 # 早上8点前回看480分钟
OBSERVER_MODEL=deepseek/deepseek-v32
OBSERVER_FALLBACK_MODEL=google/gemini-2.5-flash
```

### Layer 2: Reflector（反射者）
**触发条件**：`observations.md` 超过 8000 词

**工作流程**：
1. 备份当前 observations
2. 发送整个日志给 LLM（使用 consolidation prompt）
3. 验证输出比输入短（合理性检查）
4. 替换 observations.md
5. 清理旧备份（保留最后 10 个）

**压缩效果**：40-60% 压缩率

### Layer 3: Session Recovery（会话恢复）
**触发条件**：每次 `/new` 或 `/reset`

**工作流程**：
1. 对最近会话文件进行哈希
2. 对比上次 Observer 运行的哈希
3. 不匹配则运行恢复模式（4小时回看）
4. 降级方案：原始消息提取

### Layer 4: Reactive Watcher（反应式监视器）
**平台**：Linux（使用 inotify）

**触发条件**：
- 监测会话目录的 JSONL 写入
- 40+ 行新写入后触发 Observer
- 5 分钟冷却时间

**关键配置**：
```bash
OBSERVER_LINE_THRESHOLD=40        # 40行触发
OBSERVER_COOLDOWN_SECS=300        # 5分钟冷却
```

### Layer 5: Pre-compaction Hook（预压缩钩子）
**触发条件**：OpenClaw 压缩上下文前

**目的**：确保会话结束前记忆被捕获，不丢失任何信息

---

## 三、Dream Cycle（夜间记忆整合）

### 概述
类似人脑睡眠时的记忆巩固，夜间运行 agent 整合 observations.md。

### 九阶段流程

| 阶段 | 操作 |
|------|------|
| 1 | Preflight + 备份 |
| 2 | 读取 observations.md, favorites.md, 今日日记 |
| 3 | 应用重要性衰减 |
| 4 | 按类型和影响分类 |
| 5 | 聚类相关观察（3+ 条）|
| 6 | 未来日期保护 |
| 7 | 决定归档集合 |
| 8 | 写入归档文件 |
| 9 | 添加语义钩子 + 扫描模式 + 验证 |

### 高级特性

#### Multi-Hook Retrieval
- 为每个归档项生成 4-5 个替代语义钩子
- 解决词汇不匹配问题

#### Confidence Scoring
- 每条观察有置信度 (0.0-1.0)
- 来源类型：explicit, implicit, inference, weak, uncertain

#### Memory Type TTL

| 类型 | TTL | 衰减率 |
|------|-----|--------|
| event | 14d | -0.5/天 |
| fact | 90d | -0.1/天 |
| preference | 180d | -0.02/天 |
| goal | 365d | 无 |
| habit | 365d | 无 |
| rule | never | 无 |
| context | 30d | 无 |

#### Importance Decay
- 低于 3.0 阈值的项目被归档

#### Pattern Promotion
- 扫描近期梦境日志
- 3+ 天出现 3+ 次的主题标记为待推广
- 人类审核后生效

---

## 四、可借鉴的改进方案

### 优先级 P0（立即可行）

#### 1. Observer 被动观察层
**当前状态**：我们的 memory 系统需要手动调用

**改进方案**：
```bash
# 添加 cron job，每 15 分钟运行
*/15 * * * * cd $OPENCLAW_WORKSPACE && bash scripts/memory/observer.sh
```

**核心逻辑**：
- 读取最近的 session JSONL
- 提取新消息（MD5 去重）
- LLM 压缩 → observations.md

#### 2. Session Recovery 机制
**当前状态**：无

**改进方案**：
- 在 `/new` 或 `/reset` 时触发检查
- 对比上次哈希，不匹配则恢复

### 优先级 P1（短期改进）

#### 3. 重要性评分 + 衰减机制
**当前状态**：无

**改进方案**：
- 在 observations.md 中添加元数据
```markdown
<!-- importance: 7.5, type: fact, created: 2026-03-01, decay: -0.1/day -->
```

- 定期运行衰减脚本

#### 4. 语义钩子搜索
**当前状态**：仅向量搜索

**改进方案**：
- 为每条记忆添加 alternative_phrases
- 解决词汇不匹配问题

### 优先级 P2（长期规划）

#### 5. Dream Cycle 夜间整合
**当前状态**：无

**改进方案**：
- 添加 3am cron job
- 实现九阶段流程
- 实现 Pattern Promotion

#### 6. Memory Type TTL
**当前状态**：无过期机制

**改进方案**：
- 7 种记忆类型
- 按类型设置 TTL
- 自动归档过期记忆

---

## 五、实施路线图

### Phase 1：基础Observer（1周）
- [ ] 创建 observer.sh 脚本
- [ ] 配置 15 分钟 cron
- [ ] 实现 MD5 去重

### Phase 2：Session Recovery（1周）
- [ ] 实现哈希对比机制
- [ ] 集成到 /new 命令

### Phase 3：重要性系统（2周）
- [ ] 设计元数据格式
- [ ] 实现衰减逻辑
- [ ] 添加归档功能

### Phase 4：Dream Cycle（3周）
- [ ] 设计九阶段流程
- [ ] 实现 Pattern Promotion
- [ ] 添加人类审核流程

---

## 六、关键文件位置

### Total Recall 项目结构
```
total-recall/
├── scripts/
│   ├── observer-agent.sh         # 观察者
│   ├── reflector-agent.sh        # 反射者
│   ├── session-recovery.sh      # 会话恢复
│   ├── observer-watcher.sh      # 反应式监视器
│   ├── dream-cycle.sh           # 夜间整合
│   └── setup.sh                 # 安装脚本
├── prompts/
│   ├── observer-system.txt      # Observer prompt
│   ├── reflector-system.txt    # Reflector prompt
│   └── dream-cycle-prompt.md   # Dream Cycle prompt
├── memory/
│   ├── observations.md          # 主观察日志
│   ├── archive/                 # 归档
│   └── dream-logs/              # 夜间日志
└── SKILL.md                     # 完整文档
```

---

## 七、参考资料

- [Total Recall GitHub](https://github.com/gavdalf/total-recall)
- [Your AI Has an Attention Problem](https://gavlahh.substack.com/p/your-ai-has-an-attention-problem)
- [Do Agents Dream of Electric Sheep?](https://gavlahh.substack.com/p/do-agents-dream)

---

*本文档为分析总结，非直接复制项目代码*

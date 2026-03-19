---
title: OpenClaw 安全全景分析：从公开漏洞、利用链到最安全安装基线
date: 2026-03-19
tags:
  - OpenClaw
  - 安全分析
  - 漏洞利用
  - 安全配置
categories:
  - 安全研究
---

# OpenClaw 安全全景分析：从公开漏洞、利用链到最安全安装基线

> 作者：ckcsec | 公众号：CKCsec安全研究院 | 发布日期：2026年3月19日

## 核心观点

**OpenClaw 不是"天然不安全"，但它绝不是一个普通聊天机器人。**

它更像一个**带自然语言入口的本地自动化中枢**：
- 执行 shell、读写文件
- 抓网页、控制浏览器
- 发消息、调度任务
- 访问外部服务和 API

**真正要防的攻击链：**
```
不受信任输入 → 控制面/认证 → 工具层/浏览器/节点 → 宿主机/外部服务
```

---

## 安全风险概览

### 官方威胁模型

截至本文，官方 threat model 列出 **37 个威胁**：
- 🔴 Critical: 6 个
- 🟠 High: 16 个

**5 层信任边界：**
1. 供应链
2. 通道访问控制
3. 会话隔离
4. 工具执行
5. 外部内容

---

## 公开漏洞分析

### 一、控制面 / 认证与授权

| 漏洞 ID | 描述 | 修复版本 |
|---------|------|----------|
| CVE-2026-25253 / GHSA-g8p2-7wf7-98mq | Control UI 恶意链接自动发送 token | 2026.1.29 |
| GHSA-rqpp-rjj8-7wv8 | 共享 token 可自报高权限 scope | 2026.3.12 |
| GHSA-4jpw-hj22-2xmc | 配对权限可铸造更高权限 token | 2026.3.11 |
| CVE-2026-32302 / GHSA-5wcw-8jjv-m286 | trusted-proxy 模式来源校验可被绕过 | 2026.3.11 |

### 二、执行面 / Browser / SSRF

| 漏洞 ID | 描述 | 修复版本 |
|---------|------|----------|
| GHSA-gv46-4xfq-jv58 | node.invoke 注入可绕过 exec approvals | 2026.2.14 |
| GHSA-xgf2-vxv2-rrmg | system.run 环境清洗不完整 | 2026.2.22 |
| CVE-2026-26329 / GHSA-cv7m-c9jx-vg7q | browser upload 路径穿越可读本地文件 | 2026.2.14 |
| CVE-2026-26324 / GHSA-jrvc-8ff5-2f9f | SSRF guard 可被 IPv4-mapped IPv6 绕过 | 2026.2.14 |
| GHSA-8mh7-phf8-xgfm | skills.status 泄露配置值 | 2026.2.14 |

### 三、渠道 / 配对 / Webhook

| 漏洞 ID | 描述 | 修复版本 |
|---------|------|----------|
| GHSA-jq3f-vjww-8rq7 | Telegram webhook 预认证资源消耗 | 2026.3.13 |
| GHSA-g2f6-pwvx-r275 | iMessage SCP 命令注入 | 2026.3.13 |
| GHSA-7h7g-x2px-94hj | pairing QR 携带长期共享凭证 | 2026.3.12 |

### 四、供应链 / Skills

| 漏洞 ID | 描述 | 修复版本 |
|---------|------|----------|
| GHSA-99qw-6mr3-36qr | 工作区插件自动发现导致本地代码执行 | 2026.3.12 |

---

## 最安全安装基线

### 1. 版本基线

**当前最新正式版：openclaw 2026.3.13**

直接以 `v2026.3.13-1` 为基线，不要使用旧版本。

### 2. 推荐安装方式

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
git fetch --tags
git checkout v2026.3.13-1
export OPENCLAW_HOME_VOLUME="openclaw_home"
./docker-setup.sh
```

**安全要点：**
- 版本固定
- 环境容器化
- 默认非 root
- 易于回滚

### 3. Gateway 配置

```json
{
  "gateway": {
    "mode": "local",
    "bind": "loopback",
    "port": 18789,
    "auth": {
      "mode": "token",
      "token": "REPLACE_WITH_LONG_RANDOM_TOKEN",
      "allowTailscale": false
    }
  }
}
```

### 4. 工具配置

```json
{
  "tools": {
    "profile": "messaging",
    "deny": [
      "group:runtime",
      "browser",
      "gateway",
      "cron",
      "nodes",
      "web_fetch",
      "web_search"
    ]
  }
}
```

### 5. 沙箱配置

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all",
        "scope": "session",
        "workspaceAccess": "none"
      }
    }
  }
}
```

### 6. 入口配置

```json
{
  "channels": {
    "whatsapp": {
      "dmPolicy": "pairing",
      "groups": {
        "*": {
          "requireMention": true
        }
      }
    }
  },
  "session": {
    "dmScope": "per-channel-peer"
  }
}
```

---

## 关键安全原则

> **不要把 OpenClaw 当成一个普通聊天机器人。它更像一台会说话、会执行、会联网、还能接浏览器和节点的自动化主机。**

### 四条核心原则

1. **入口最小化**：默认配对，群里必须 @，多人私信分会话
2. **工具最小化**：先 messaging，高危工具默认 deny
3. **执行隔离化**：所有会话进沙箱，默认不见工作区
4. **状态分离化**：专用浏览器 profile、专用 OS 用户、最好专用主机

### 最安全的默认值

```
最新版本 + loopback 绑定 + 显式认证 + 最小工具集 + 全量沙箱 + 专用环境 + 只给可信发送者
```

---

## 已部署如何补救

1. **止血**：升级到 2026.3.13，停掉高危工具
2. **轮换凭证**：重新生成 gateway token
3. **安全审计**：`openclaw security audit`

---

## 参考资料

- [OpenClaw Releases](https://github.com/openclaw/openclaw/releases)
- [OpenClaw Trust](https://trust.openclaw.ai/trust/zh-cn)
- [Gateway Security](https://docs.openclaw.ai/zh-CN/gateway/security)
- [Sandboxing 文档](https://docs.openclaw.ai/zh-CN/gateway/sandboxing)

---

## 相关标签

#OpenClaw #安全分析 #漏洞利用 #安全配置 #红队 #蓝队

---
title: OpenClaw AI 对 AI RCE 漏洞浅析
date: 2026-03-21
category: 安全研究
tags: [漏洞情报, 安全研究]
source: 微信安全文章
---

# OpenClaw AI 对 AI RCE 漏洞浅析

## 1. 问题现状

### 背景描述

OpenClaw 是一款开源可自托管的 AI 控制平面，支持 Telegram、微信等多平台接入，赋予 AI 助手完整系统访问权限。攻击方 **Hackian**（Ethiack 开发的自主 AI 渗透测试工具）以纯黑盒方式对 OpenClaw 发起了真实攻击测试。

### 风险评估

攻击在 **1 小时 40 分钟**内完成，发现并验证了完整的 RCE（远程代码执行）利用链：

- **CVSS 评分**：Critical（1-Click RCE）
- **CVE 编号**：CVE-2026-25253
- **利用难度**：仅需受害者点击一次链接
- **影响范围**：所有 ≤ 2026.1.29 版本的 OpenClaw 实例

### 侦察阶段关键发现

| 发现项 | 安全影响 |
|--------|----------|
| Source Map 泄露 | 前端完整源码对外暴露，无须逆向 |
| 跨域 WebSocket 升级 | 任意来源可发起 WS 连接 |
| 前端安全验证逻辑 | 依赖前端，可被轻易绕过 |

## 2. 解决思路与方案对比

### 核心漏洞链

1. **Source Map 泄露** → 攻击者还原完整前端源码
2. **gatewayUrl 参数注入** → 无鉴权覆盖 WebSocket 网关地址
3. **WebSocket Origin 验证缺失** → 任意来源可建立连接
4. **LocalStorage Token 外泄** → 账户接管（ATO）
5. **借助受害者浏览器连接 localhost** → 本地部署也难幸免
6. **AI 助手执行任意系统命令** → RCE

### 攻击路径

```
攻击者构造恶意链接
  → 受害者浏览器访问 → 重定向至含 ?gatewayUrl=wss://attacker.com 的控制台
  → OpenClaw 前端连接攻击者 WS 服务器
  → LocalStorage Auth Token 即刻外泄
  → 攻击者获取合法 Token → 账户接管
  → 通过浏览器作为跳板连接 localhost 网关
  → AI 助手执行任意命令 → RCE
```

### 防御方案对比

| 方案 | 优点 | 缺点 | 实施难度 |
|------|------|------|----------|
| 移除/限制 gatewayUrl 参数 | 根治参数注入 | 可能影响部分合法功能 | 低 |
| 服务端 Origin 验证 | 防御跨域 WS 攻击 | 需要 WebSocket 服务器配合 | 中 |
| 生产环境禁用 Source Map | 根本解决源码泄露 | 无 | 低 |
| AI 权限最小化 | 降低 RCE 后的损害范围 | 影响部分功能 | 高 |

## 3. 最佳实践框架

### 方案选择

采用**多层纵深防御**策略，以「限制 + 验证 + 隐藏」为核心。

### 架构设计

```
[攻击者] --恶意链接--> [受害者浏览器] --WS--> [攻击者 WS 服务器]
                          |
                          +-- 含 gatewayUrl 的正常请求 --> [OpenClaw Gateway]
                                                            |
                                                    [AI 助手 → 系统命令执行]
```

### 关键组件

- **gatewayUrl 参数**：必须移除或严格限制，仅允许受信任来源覆盖
- **Origin 验证**：WebSocket 服务端实施严格 Origin 白名单
- **Source Map 管控**：生产环境禁止暴露 `.map` 文件
- **Token 安全**：避免将敏感 Token 存储在可被 JS 读取的位置

## 4. 具体操作步骤

### 修复步骤

**Step 1：修复 gatewayUrl 参数注入**

```bash
# 检查当前 OpenClaw 版本
openclaw --version

# 更新至最新版本（≥ 2026.1.29）
openclaw update
```

或手动审查前端代码，移除对 `gatewayUrl` 查询参数的直接信任。

**Step 2：确保生产环境不暴露 Source Map**

```nginx
# Nginx 配置示例：禁止访问 .map 文件
location ~ \.map$ {
    deny all;
    return 404;
}
```

**Step 3：配置 WebSocket Origin 验证（Nginx 反代场景）**

```nginx
proxy_set_header Origin $http_origin;
# 确保 upstream 服务端验证 Origin 白名单
```

**Step 4：审查 LocalStorage Token 存储方式**

将敏感 Token 迁移至 `httpOnly` Cookie 或内存变量，减少 XSS 下的 Token 窃取风险。

**Step 5：验证修复**

```bash
# 重启 Gateway
openclaw gateway restart

# 确认 gatewayUrl 参数不再可控
curl "http://localhost:18789/?gatewayUrl=wss://evil.com"  # 应拒绝
```

## 5. 总结与建议

### 成效回顾

- 修复后（commit 8cb0fa9）：三个独立漏洞均被封堵，攻击链断裂
- 修复耗时：从上报到合并不足 3 天，说明问题修复本身并不困难
- 核心教训：快速迭代不能以牺牲安全为代价

### 深层启示

- **AI 助手「全系统访问权限」是双刃剑**：被控后攻击者获得完整操作权限，非受限 Shell
- **WebSocket 是新兴、缺乏规范的攻击面**：开发者普遍缺乏 WebSocket 安全意识
- **AI 可以自动化攻击另一个 AI**：Hackian 在无人工干预下 100 分钟完成全流程
- **自托管工具安全门槛远超普通用户认知**：非安全专业背景的 AI 爱好者难以完成专业加固

### 后续维护

- 建议将 WebSocket 安全检查纳入 CI/CD 流程
- 定期使用自动化工具（如 Hackian）进行黑盒渗透测试
- 关注 OpenClaw 安全公告，及时更新版本

---

**参考资料**：
- CVE-2026-25253：https://nvd.nist.gov/vuln/detail/CVE-2026-25253
- 修复 Commit：8cb0fa9
- 原始报告：ethiack.com/news/blog/one-click-rce-openclaw

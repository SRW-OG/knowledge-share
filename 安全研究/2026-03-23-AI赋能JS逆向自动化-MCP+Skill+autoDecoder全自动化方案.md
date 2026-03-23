---
title: AI赋能JS逆向自动化 — MCP+Skill+autoDecoder全自动化方案
date: 2026-03-23
category: 安全研究
tags: [JS逆向, MCP, 自动化, BurpSuite, JSRPC, Flask]
source: https://mp.weixin.qq.com/s/6jWSkNzqpOUz_x18KvEDpA
---

# AI赋能JS逆向自动化 — MCP+Skill+autoDecoder全自动化方案

## 1. 问题现状

### 背景描述

随着前端安全防护手段日益复杂化，参数加密场景愈发广泛。JavaScript 代码通常经过多层混淆与封装，传统的纯手动逆向方法耗时费力，且高度依赖个人经验。安全研究人员在对抗 JS 逆向时往往已经精疲力尽，难以将精力集中在真正的安全测试上。

### 风险评估

- **效率瓶颈**：人工逆向平均耗时可达数小时甚至数天
- **技能门槛**：需要深厚的 JavaScript 代码功底和逻辑还原能力
- **可扩展性差**：每个新目标都需要重新手动分析，无法复用
- **维护成本高**：加密逻辑变更后需重新逆向

### 传统JS逆向方法回顾

| 方法 | 原理 | 适用场景 | 缺点 |
|------|------|----------|------|
| 直接修改变量法 | 在调试器中直接修改加密前参数 | 简单场景 | 无法应对复杂加密逻辑 |
| 中间劫持法（JS-Forward） | 通过 JS-Forward 在明文点插入代码，AJAX 转发到 Burp | 中等复杂度 | 需要手动编写注入代码 |
| 远程调用法（JS-RPC） | 通过 WebSocket 远程调用浏览器上下文中的 JS 函数 | 复杂场景 | 初始配置繁琐 |
| 硬核对抗法（JS原生） | 反混淆分析 + 补环境 + 本地运行 | 极复杂场景 | 门槛极高，耗时最长 |

**本文选择远程调用法（JS-RPC）作为 MCP 赋能的底座**，因为它能直接将浏览器环境转化为加密服务接口，避免纯手动逆向的低效，又绕开了纯 JS 还原的高复杂度。

---

## 2. 解决思路与方案对比

### 技术选型分析

本方案的核心思路是：**用 AI 自动完成函数发现与代码生成，让人从繁琐的逆向分析中解放出来**。

### 工具链架构

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│   Burp      │ ───▶ │  Flask       │ ───▶ │  JSRPC      │
│  autoDecoder│      │  代理服务     │      │  (浏览器端)  │
└─────────────┘      └──────────────┘      └─────────────┘
      ▲                     ▲                     ▲
      │                     │                     │
      │              ┌──────────────┐              │
      └──────────────│ chrome-devtools│◀────────────┘
                     │     MCP       │
                     │   (Codex AI)  │
                     └──────────────┘
```

### 组件职责

| 组件 | 职责 |
|------|------|
| **chrome-devtools-mcp** | 通过 Chrome DevTools Protocol 连接浏览器，暴露执行JS、调试、网络监控等能力给 AI |
| **JSRPC** | 注入到浏览器页面，通过 WebSocket 协议建立远程过程调用，直接调用浏览器中的加密函数 |
| **Flask** | 接收 Burp autoDecoder 请求，通过 JSRPC 调用浏览器加密函数，重构请求并转发 |
| **autoDecoder** | Burp Suite 插件，自动化处理加密/编码接口，自动将明文参数发送至 Flask 并完成加密请求构建 |

### 方案优势

- **高自动化**：AI 自动完成函数发现、代码生成，人工只需下达指令
- **低门槛**：无需深厚 JS 逆向功底，降低安全测试入门门槛
- **可复用**：Skill 模板支持多种加密场景的适配配置
- **端到端**：从 AI 分析到 Burp 集成，全流程覆盖

---

## 3. 最佳实践框架

### MCP + Skill 自动化流程

#### 阶段一：初始化与连接

1. Codex 通过 MCP 协议连接浏览器
2. 加载目标页面并打开 DevTools

#### 阶段二：分析与入口定位

1. 触发签名流程并抓取调用栈
2. 检查打包代码并定位 `sign`、`encrypt` 等函数
3. 确认入口类型：全局函数 / 对象方法 / 动态 resolver

#### 阶段三：生成与注入 JSRPC

1. 基于模板生成 JSRPC 注入代码
2. 注入并注册 JSRPC action
3. 验证 JSRPC 调用是否返回预期结果

#### 阶段四：构建服务端代理

1. 生成 Flask 代理服务
2. 启动服务并做健康检查

#### 阶段五：集成与交付

1. 输出 Burp autoDecoder 配置说明
2. 端到端验证并记录结果

### Skill 模板体系

| 模板 | 职责 |
|------|------|
| `js-reverse-automation` | 主技能模板，负责整体流程控制和协调 |
| `jsrpc-injection-template` | JSRPC 注入模板，提供通用函数拦截和远程调用能力 |
| `flask-jsrpc-proxy-template` | Flask 代理模板，面向 Burp autoDecoder 的中间件服务 |

---

## 4. 具体操作步骤

### 前置准备

- Codex 或其他 AI 客户端
- Chrome 浏览器（需开启远程调试端口）
- Burp Suite + autoDecoder 插件
- Node.js 环境（用于运行 npx）

### Step 1：MCP 配置

**添加 MCP 服务器到 Codex：**

```bash
codex mcp add chrome-devtools npx -y chrome-devtools-mcp@latest
```

**修改 Codex 配置文件（`~/.codex/config.toml`）：**

```toml
[mcp_servers.chrome-devtools]
command = "npx"
args = ["-y", "chrome-devtools-mcp@latest"]
```

**启动浏览器并开启远程调试：**

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --remote-debugging-address=0.0.0.0
```

### Step 2：创建 Skill 文件

在 Codex workspace 创建三个 Markdown 文件：

**1. `js-reverse-automation.md`（主技能模板）**

```markdown
---
name: js-reverse-automation
description: 通过 MCP 连接浏览器，为目标自动化搭建JS环境，定位签名/加密函数入口，生成 JSRPC 代码，创建 Flask 代理，并输出 Burp autoDecoder 配置。
---

# JS 逆向自动化技能

## 输入参数
- 目标 URL
- 需要分析的加密参数（如 sign / enc / token）
- Fetch 请求示例（可选）

## 交付输出
1. JSRPC 注入代码
2. Flask 代理服务
3. Burp autoDecoder 配置说明
4. 完整的加密/签名逻辑代码
```

**2. `jsrpc.md`（JSRPC 注入模板）**

```javascript
var client = new Hlclient("ws://127.0.0.1:12080/ws?group=fausto&name=burp");

var JSRPC_CONFIG = {
    actionName: "generate_sign",
    entry: {
        type: "resolver",
        resolver: function() {
            return window.__SIGN_FN__ || null;
        }
    },
    bindThis: null,
    async: false,
    normalizeInput: function(param) { return param; },
    normalizeOutput: function(result) { return result; },
    onError: function(err) { return "ERROR_" + Date.now(); }
};

client.regAction(JSRPC_CONFIG.actionName, function(resolve, param) {
    try {
        var fn = resolveEntry(JSRPC_CONFIG);
        if (typeof fn !== "function") throw new Error("签名/加密函数未找到");
        var input = JSRPC_CONFIG.normalizeInput(param);
        var result = fn.call(JSRPC_CONFIG.bindThis || null, input);
        resolve(JSRPC_CONFIG.normalizeOutput(result));
    } catch (error) {
        resolve(JSRPC_CONFIG.onError(error));
    }
});
```

**3. `flask.md`（Flask 代理模板）**

```python
from flask import Flask, request
import requests
import json

app = Flask(__name__)
JSRPC_URL = "http://127.0.0.1:12080/go"
JSRPC_GROUP = "fausto"
JSRPC_ACTION = "generate_sign"

@app.route('/encode', methods=['POST'])
def handle_encode():
    data_body = request.form.get('dataBody', '')
    # 解析并调用 JSRPC 生成签名
    # 回填签名并返回
    return new_body
```

### Step 3：执行自动化分析

**输入触发 Skill 的提示词：**

```
目标网址：https://xxx/login/index
需要分析的加密参数：password
环境限制：无
Fetch 示例：fetch("https://xxx/Login/CheckLogin", {...})
```

AI 将自动完成：
- 入口定位（找到 `$.md5(...)` 加密函数）
- 生成 JSRPC 注入代码
- 生成 Flask 代理服务
- 生成 Burp autoDecoder 配置说明

### Step 4：JSRPC 注入与验证

**在浏览器控制台执行 JSRPC 项目中的 `JsEnv_Dev.js`：**

```javascript
// 建立 JSRPC 连接
var client = new Hlclient("ws://127.0.0.1:12080/ws?group=fausto&name=burp");
```

**注入 AI 生成的 JSRPC 代码：**

```javascript
var JSRPC_CONFIG = {
    actionName: "generate_password_md5",
    entry: {
        type: "resolver",
        resolver: function() {
            if (window.$ && typeof $.md5 === "function") return $.md5;
            return null;
        }
    },
    // ... 其他配置
};

client.regAction(JSRPC_CONFIG.actionName, function(resolve, param) {
    // 实现加密调用逻辑
});
```

**验证 JSRPC 调用：**

```bash
http://127.0.0.1:12080/go?group=fausto&action=generate_password_md5&param=111111
# 返回: 96e79218965eb72c92a549dd5a330112
```

### Step 5：Flask 服务启动

```bash
python flask_proxy_password.py
```

**验证 Flask 加密：**

```bash
curl -X POST http://127.0.0.1:8888/encode \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "dataBody=username=111111&password=111111&code=1234&role=000002"
```

### Step 6：Burp autoDecoder 配置

在 Burp Suite 中配置 autoDecoder 指向 `http://127.0.0.1:8888/encode`，自动完成加密请求构建。

---

## 5. 总结与建议

### 成效回顾

本方案成功实现了**从"半自动"到"高自动"的跨越**：

- ✅ AI 自动完成加密函数发现与定位
- ✅ AI 自动生成 JSRPC 注入代码
- ✅ AI 自动生成 Flask 代理服务
- ✅ AI 自动生成 Burp autoDecoder 配置
- ✅ 全程人员只需下达指令，无需手动编写代码

### 局限性

- 对于极复杂场景的 JS 代码（如重度混淆、自定义加密算法），AI 分析仍可能存在偏差
- 依赖浏览器环境和 MCP 服务的稳定性
- 需要目标站点的加密逻辑相对稳定

### 后续维护

1. **Skill 模板迭代**：根据实际案例不断完善 JSRPC 和 Flask 模板
2. **AI 模型优化**：针对不同加密场景微调提示词策略
3. **日志审计**：定期审查 JSRPC 调用日志，发现异常加密逻辑变更
4. **兼容性测试**：定期验证对新版浏览器和 Burp Suite 的兼容性

### 适用场景

| 场景 | 推荐度 | 说明 |
|------|--------|------|
| 标准 MD5/SHA 加密 | ⭐⭐⭐⭐⭐ | 极简适配 |
| 存在封装加密函数 | ⭐⭐⭐⭐ | 模板适配 |
| 动态加载的加密逻辑 | ⭐⭐⭐ | 需要 resolver 配置 |
| 定制化强加密算法 | ⭐⭐ | AI 辅助分析 + 人工调试 |

---

## 相关资源

- chrome-devtools-mcp: `npx -y chrome-devtools-mcp@latest`
- JS-RPC 工具: Hlclient
- Burp autoDecoder 插件

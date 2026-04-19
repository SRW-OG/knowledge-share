---
title: Burp 官方 MCP Server 发布：AI 客户端直接操控 Burp 渗透测试
date: 2026-04-19
category: 安全研究
tags: [Burp Suite, MCP, AI, 渗透测试, 安全工具, PortSwigger]
source: https://mp.weixin.qq.com/s/nK_hDgiul7YSY7nBurwgaQ
---

# Burp 官方 MCP Server 发布：AI 客户端直接操控 Burp 渗透测试

## 1. 问题现状 (Current Status & Problem Analysis)

### 背景描述

Burp Suite 是渗透测试的标准工具，但长期存在工作流割裂问题：测试人员需要在浏览器、代理、历史记录、Collaborator 等多个界面之间频繁切换。当 AI 客户端（Claude Desktop、Cursor、Windsurf、Codex）兴起后，业界一直没有官方方案让 AI 直接操控 Burp。

### 风险评估

- **效率瓶颈**：手动翻阅几百条代理历史记录，耗时长且容易遗漏
- **AI 协作断裂**：市面上的 AI + Burp 集成均为第三方方案，缺乏官方支持，稳定性无保障
- **工具链碎片化**：SSRF、盲注等 OOB 测试场景需要同时操作多个标签页，流程不连贯

### 影响范围

所有使用 Burp Suite 专业版进行渗透测试的安全工程师，以及希望将 AI 能力引入渗透测试工作流的团队。

---

## 2. 解决思路与方案对比 (Solution Options & Analysis)

### 方案一：Burp MCP Server 官方扩展

PortSwigger 官方发布 MCP Server 扩展，将 Burp Suite 的核心能力通过 MCP（Model Context Protocol）协议暴露给 AI 客户端。

| 维度 | 评分 |
|------|------|
| 官方支持 | ⭐⭐⭐⭐⭐ |
| 功能覆盖 | ⭐⭐⭐⭐ |
| 配置复杂度 | ⭐⭐⭐ |

### 方案二：第三方脚本 + 非官方集成

通过 Python 脚本调用 Burp Montoya API，间接实现 AI 控制。

| 维度 | 评分 |
|------|------|
| 灵活性 | ⭐⭐⭐⭐ |
| 稳定性 | ⭐⭐ |
| 维护成本 | ⭐ |

### 方案三：纯 AI 客户端本地代理扫描

使用 AI 客户端自带的代理功能进行扫描，不经过 Burp。

| 维度 | 评分 |
|------|------|
| 部署简便性 | ⭐⭐⭐ |
| 与 Burp 协同 | ⭐ |

**结论**：方案一（官方 MCP Server）是最佳选择，官方背书、功能完整、与 Burp 深度集成。

---

## 3. 最佳实践框架 (Best Practice Framework)

### 方案选择

采用 **Burp MCP Server 官方扩展**，通过 MCP 协议连接 AI 客户端与 Burp Suite。

### 架构设计

```
AI 客户端 (Stdio) → mcp-proxy (协议转换) → Burp MCP 扩展 (SSE, localhost:9876)
AI 客户端 (SSE 直连) → Burp MCP 扩展 (localhost:9876)
```

MCP 传输层支持两套协议并行：
- **SSE**（Server-Sent Events）：适合 Cursor、Windsurf 等支持 SSE 的客户端
- **Stdio**：Claude Desktop 专用，需通过 mcp-proxy 代理转换

### 关键组件

| 组件 | 说明 |
|------|------|
| `burp-mcp-all.jar` | 核心扩展包，含 MCP Server + Stdio 代理 |
| `mcp-proxy` | 协议转换器（Stdio ↔ SSE） |
| `Burp Montoya API` | Burp 底层能力接口 |
| MCP 客户端 | Claude Desktop / Cursor / Windsurf / Codex |

---

## 4. 具体操作步骤 (Implementation Steps)

### 前置准备

- Java 运行环境（需 `jar` 命令可用）
- Burp Suite Professional（专业版，Community 版缺少 Collaborator API）
- Gradle 构建工具

### Step 1：编译扩展 JAR

```bash
# Clone 仓库
git clone https://github.com/PortSwigger/mcp-server.git
cd mcp-server

# 编译包含 Stdio 代理的完整 JAR
./gradlew embedProxyJar

# 编译产物位置
# build/libs/burp-mcp-all.jar
```

### Step 2：在 Burp Suite 中加载扩展

1. 打开 Burp Suite → `Extensions` 标签页
2. 点击 `Add` → 类型选 `Java`
3. 选择 `burp-mcp-all.jar` → 点击 `Next`
4. 加载成功后，顶部出现 `MCP` 标签页
5. 在 MCP 标签页中：
   - 勾选 `Enabled` 启用 MCP Server
   - 按需勾选 `Enable tools that can edit your config`（允许 AI 修改 Burp 配置）
   - 默认监听 `http://127.0.0.1:9876`，端口可自定义

### Step 3：接入 MCP 客户端

**方式一：SSE 直连（Cursor / Windsurf）**

在客户端配置中填入 SSE URL：
```
http://127.0.0.1:9876
# 或
http://127.0.0.1:9876/sse
```

**方式二：Stdio 代理（Claude Desktop）**

扩展内置一键安装器，自动写入 Claude Desktop 配置。

手动配置需编辑 `~/Library/Application Support/Claude/claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "burp": {
      "command": "<Java路径>",
      "args": [
        "-jar",
        "/path/to/mcp-proxy-all.jar",
        "--sse-url",
        "http://127.0.0.1:9876"
      ]
    }
  }
}
```

### Step 4：验证连接

确保 Burp 已在运行且 MCP 扩展已加载，保存配置后重启客户端，检查 AI 是否能正常调用 Burp 工具。

---

## 5. 总结与建议 (Summary & Recommendations)

### 成效回顾

- **效率提升**：AI 自动翻阅代理历史、正则匹配关键字，无需手动逐条查看
- **OOB 测试闭环**：Collaborator payload 生成 → 注入发送 → 回调轮询，全流程 AI 自动化
- **Repeater 联动**：AI 直接读取/写入 Repeater 编辑器内容，自动构造 payload 并发包验证
- **配置管理**：支持导出/导入项目和用户级配置（JSON 格式），便于团队协作

### 暴露的工具能力矩阵

| 类别 | 工具 |
|------|------|
| HTTP 请求 | 发送 HTTP/1.1 和 HTTP/2 请求 |
| 模块联动 | 发包到 Repeater 或 Intruder |
| 编解码 | URL 编解码、Base64 编解码、随机字符串 |
| 配置管理 | 导出/导入项目配置（JSON） |
| 历史记录 | HTTP 历史、WebSocket 历史（正则 + 分页） |
| 任务控制 | 暂停/恢复扫描、开关代理拦截 |
| 编辑器交互 | 读写 Repeater/Intruder 内容 |
| 扫描器（专业版） | 获取漏洞结果、生成 Collaborator 负载、轮询外带交互 |

### 后续维护

- Collaborator 功能**仅限 Burp Suite 专业版**，社区版不包含相关 API
- 响应体超过 5000 字符会自动截断，防止 AI context 溢出
- `get_active_editor_contents` 取的是键盘焦点所在文本区域，需确保 Burp 窗口处于前台激活状态
- 自定义工具扩展：在 `Tools.kt` 中声明 `@Serializable` data class，工具名由类名自动推导（snake_case）

### 适用场景

- **渗透测试师**：将 AI 引入日常测试工作流，提升漏洞发现效率
- **安全团队**：标准化 AI + Burp 工作流，输出可复现的测试报告
- **AI 安全研究**：探索大模型在网络安全领域的能力边界

---

## 相关资源

- 项目地址：https://github.com/PortSwigger/mcp-server
- Stdio 代理源码：https://github.com/PortSwigger/mcp-proxy

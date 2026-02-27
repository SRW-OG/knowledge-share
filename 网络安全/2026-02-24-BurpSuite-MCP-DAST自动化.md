---
title: BurpSuite MCP 动态应用安全测试自动化
date: 2026-02-24
category: 网络安全
tags: [BurpSuite, MCP, DAST, 自动化, 渗透测试, AI安全]
source: https://infosecwriteups.com/dast-automation-using-burpsuite-mcp-923b6c0101e1
---

# BurpSuite MCP 动态应用安全测试自动化

## 1. 问题现状

### 背景描述

动态应用安全测试（DAST）自动化一直是安全评估领域的难题。尽管 SAST（静态应用安全测试）已实现高度自动化，但 DAST 层面仍依赖大量人工操作。安全工程师需要手动复制 HTTP 请求到聊天窗口，逐一使用 BurpSuite 的各种功能进行测试，效率低下且重复性工作繁重。

### 风险评估

- **效率风险**：人工手动测试耗时，尤其是在大型渗透测试项目中
- **覆盖度风险**：人工测试难以覆盖所有 OWASP Top 10 漏洞点
- **一致性风险**：不同测试人员可能遗漏相同的检查点
- **成本风险**：专业渗透测试服务成本高昂

### 影响范围

- 企业安全团队进行 Web 应用渗透测试
- Bug Bounty 猎人进行漏洞挖掘
- 开发团队进行安全自测
- 安全评估项目的时间和资源投入

---

## 2. 解决思路与方案对比

### 方案一：传统手动测试

使用 BurpSuite 手动进行渗透测试，人工发送请求、分析响应、识别漏洞。

| 优点 | 缺点 |
|------|------|
| 精确度高，可深度测试 | 效率低下 |
| 可发现复杂逻辑漏洞 | 重复性工作多 |
| 灵活应对变化场景 | 依赖测试人员经验 |

### 方案二：BurpSuite Scanner 自动扫描

使用 BurpSuite 内置的被动扫描和主动扫描功能。

| 优点 | 缺点 |
|------|------|
| 无需配置 | 扫描深度有限 |
| 覆盖基础漏洞 | 容易产生误报 |
| 可批量处理 | 无法针对特定逻辑测试 |

### 方案三：BurpSuite MCP + AI Agent（推荐）

使用 Model Context Protocol 将 AI 与 BurpSuite 集成，通过自然语言指令驱动自动化测试。

| 优点 | 缺点 |
|------|------|
| 自然语言控制，门槛低 | 仍需人工定义清晰指令 |
| 自动化程度高 | 可能产生误报 |
| 可批量处理复杂任务 | 复杂逻辑仍需人工介入 |
| 支持历史请求分析 | 需要正确配置 |

---

## 3. 最佳实践框架

### 方案选择

**推荐方案：BurpSuite MCP + AI Agent（如 Cursor）**

### 架构设计

```
┌─────────────────────────────────────────────────────────┐
│                    AI Agent (Cursor)                     │
│                   自然语言指令输入                        │
└─────────────────────┬───────────────────────────────────┘
                      │ MCP (Model Context Protocol)
                      │ HTTP/JSON
┌─────────────────────▼───────────────────────────────────┐
│              BurpSuite MCP Server                        │
│  • Proxy History Inspection (代理历史检查)               │
│  • Code-to-Repeater Integration (Repeater集成)         │
│  • Scanner Automation (扫描自动化)                       │
│  • Burp Configuration Management (配置管理)              │
│  • Direct Request Execution (直接请求执行)               │
│  • Burp Collaborator Support (Collaborator支持)         │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│              Target Application                          │
│              (目标 Web 应用)                              │
└─────────────────────────────────────────────────────────┘
```

### 关键组件

| 组件 | 说明 |
|------|------|
| **BurpSuite** | 行业标准 Web 渗透测试工具 |
| **MCP Server** | BurpSuite 扩展，位于 Extension Tab |
| **AI Agent** | 如 Cursor、Windsurf 等支持 MCP 的 AI 客户端 |
| **Model Context Protocol** | Anthropic 推出的开放标准，用于 AI 与外部工具交互 |

---

## 4. 具体操作步骤

### 4.1 前置准备

- **BurpSuite 版本**：专业版或社区版均可（MCP 扩展兼容免费版）
- **AI 客户端**：Cursor（付费版或免费版）
- **网络要求**：BurpSuite MCP 默认监听 `127.0.0.1:9876`

> ⚠️ **注意**：确保 BurpSuite 和 AI 客户端在同一台机器上运行

### 4.2 安装 BurpSuite MCP 扩展

**步骤一：安装扩展**

1. 打开 BurpSuite
2. 进入 **Extender** → **BApp Store**
3. 搜索 "MCP" 并安装

**步骤二：配置 MCP Server**

1. 安装完成后，顶部菜单出现 **MCP** Tab
2. 进入该 Tab，在 **Server Configuration** 中启用
3. 确认状态显示服务器正在运行（默认 `127.0.0.1:9876`）

**步骤三：安全配置（可选）**

- 启用 "Enable tools that can edit your config"：允许 AI 修改 Burp 配置（如代理监听器、Scope）
- 如只需 AI 读取数据和发送请求，建议保持未勾选以确保安全

### 4.3 连接 AI Agent (Cursor)

**步骤一：配置 Cursor**

1. 打开 Cursor 设置
2. 找到 MCP 配置区域
3. 编辑 `settings.json` 文件，添加以下配置：

```json
{
  "mcpServers": {
    "burp": {
      "url": "http://127.0.0.1:9876/sse"
    }
  }
}
```

**步骤二：重启 Cursor**

保存配置文件后，重启 Cursor。

**步骤三：验证连接**

1. 重启后，在 Cursor 设置的 **Tools & MCP** 部分查看
2. 应显示 Burp MCP server 已连接
3. 如未显示，尝试开关单选按钮或重新启动 Cursor

### 4.4 使用示例

**基础验证**

```bash
# 通过自然语言让 AI 获取最近 5 个 HTTP 请求
"Fetch the last 5 HTTP requests from Burp Suite HTTP history"
```

**添加目标到 Scope**

```bash
# 让 AI 将目标添加到扫描范围
"Add anything.com to scope"
```

**自动化安全测试**

```
Act as senior application security engineer and:
1. List all the unique endpoints from the targeted scope
2. Perform security assessment based OWASP Top 10 API's security risks
3. Make sure not to do intrusive and any deletion task
4. Approach it by understanding what exactly this API will be doing
5. When you identify the vulnerability, review that for false positive
6. Send it to Repeater for further analysis
```

**基于报告自动化测试**

```bash
# 让 AI 学习公开的 Bug Bounty 报告，然后在目标上测试类似漏洞
# 参考公开漏洞报告仓库，学习漏洞模式
# 在目标上尝试类似攻击
```

### 4.5 验证测试

1. **连接验证**：让 AI 获取 HTTP 历史，确认返回请求数据
2. **Scope 验证**：让 AI 添加目标到 Scope，确认 Burp 中显示
3. **请求验证**：让 AI 发送测试请求，确认在 HTTP History 中可见

---

## 5. 总结与建议

### 成效回顾

- **效率提升**：通过自然语言指令自动化重复性测试工作
- **覆盖度增强**：AI 可批量处理大量端点的安全检查
- **门槛降低**：非专业安全人员也可进行基础渗透测试
- **DAST 自动化突破**：填补了 SAST 自动化与 DAST 自动化之间的空白

### 局限性

| 局限 | 说明 |
|------|------|
| **误报** | AI 可能产生误报，需要人工复核 |
| **复杂逻辑** | 业务逻辑漏洞仍需人工深度测试 |
| **指令依赖** | 需要清晰、具体的指令才能得到准确结果 |
| **上下文理解** | AI 对复杂应用的理解有限 |

### 后续建议

1. **清晰指令**：提供精确的测试目标和范围
2. **结果复核**：所有 AI 发现的问题需人工验证
3. **提示词优化**：根据项目特点优化测试提示词
4. **结合使用**：将 MCP 自动化与人工测试结合
5. **持续探索**：尝试不同的提示词、工作流和攻击模式

### 参考资源

- [BurpSuite MCP 官方文档](https://portswigger.net/)
- [Cursor IDE](https://cursor.sh/)
- [OWASP Top 10 API Security](https://owasp.org/www-project-api-security/)
- [MCP 官方规范](https://modelcontextprotocol.io/)

---

**作者**：Xcheater  
**发布日期**：2025年12月27日  
**原文链接**：https://infosecwriteups.com/dast-automation-using-burpsuite-mcp-923b6c0101e1

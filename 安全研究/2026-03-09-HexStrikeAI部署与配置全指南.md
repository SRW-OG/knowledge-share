---
title: HexStrike AI：渗透测试助手部署与配置全指南
date: 2026-03-09
category: 网络安全
tags: [AI渗透测试, HexStrike, MCP, 自动化渗透, 工具部署]
source: https://mp.weixin.qq.com/s/P0c75O16pBFC6imBd_9Uiw
author: 泷羽Sec-Ceo
---

# HexStrike AI：渗透测试助手部署与配置全指南

## 1. 项目核心概览

HexStrike AI 是一款基于人工智能的进攻性安全框架。它采用 Model Context Protocol (MCP) 协议构建，连接了大语言模型（如 Claude、GPT、Copilot）与 150 多种专业网络安全工具。通过该框架，AI 智能体能够自主执行从网络扫描、漏洞挖掘到复杂攻击链构建的自动化渗透测试流程。

### 主要特点

- **工具集**：150+ 安全工具，包含 Web 安全、二进制分析、密码破解等 35 个攻击类别
- **多智能体协同工作**：由 12+ AI 智能体（漏洞情报分析、攻击链发现、参数优化）协同工作
- **智能决策引擎**：选择最优工具，并随时跟踪攻击参数
- **快速性能**：比传统手工测试快 16-24 倍

### 资源信息

| 资源类型 | 链接/说明 |
|----------|-----------|
| 官方网站 | hexstrike.com |
| GitHub 仓库 | 0x4m4/hexstrike-ai |
| 系统要求 | Kali Linux 2025.4+ / Python 3.9+ / 8GB RAM (推荐) |

---

## 2. Kali Linux 环境准备

### 1. 优化系统源 (推荐)

为了确保工具下载速度，建议将 APT 源更换为国内镜像（如阿里云）：

```bash
# 备份并编辑源列表
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo nano /etc/apt/sources.list

# 添加阿里云源
deb https://mirrors.aliyun.com/kali kali-rolling main non-free contrib
deb-src https://mirrors.aliyun.com/kali kali-rolling main non-free contrib

# 更新系统
sudo apt update
```

### 2. 浏览器环境 (Browser Agent)

自动化 Web 任务需要 Chrome 浏览器及其驱动支持：

**安装 Chrome：**
```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt install -f
```

**配置 ChromeDriver：**
访问 Chrome for Testing 下载与浏览器版本一致的驱动，解压并移动至 `/usr/local/bin/`，确保赋予执行权限：

```bash
unzip chromedriver-linux64.zip
sudo mv chromedriver-linux64/chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver
```

---

## 3. 服务端部署

### 1. 自动化安装 (推荐)

在最新的 Kali 仓库中，可以直接通过包管理器安装：

```bash
sudo apt install hexstrike-ai
```

### 2. 手动源码安装

若需使用最新开发版，请通过源码部署：

```bash
git clone https://github.com/0x4m4/hexstrike-ai.git
cd hexstrike-ai

# 创建并激活虚拟环境
python3 -m venv venv
source venv/bin/activate

# 安装 Python 依赖
pip install -r requirements.txt
```

### 3. 启动服务

```bash
# 启动服务端 (默认端口 8888 )
hexstrike_server --host 0.0.0.0 --port 8888
```

**验证服务**：访问 `http://<KALI_IP>:8888/health`，若返回正常则说明服务端已就绪。

---

## 4. 客户端配置 (MCP 接入)

配置客户端是为了让 AI 助手（如 Cursor 或 Cherry Studio）能够调用 HexStrike 的能力。

### 1. 准备 MCP 脚本与环境

- **脚本放置**：将 `hexstrike_mcp.py` 脚本放置在本地路径（建议路径不含中文，如 `C:\hexstrike_mcp.py`）
- **依赖安装**：确保本地 Python 环境已安装 MCP 相关依赖：

```bash
pip install mcp fastapi uvicorn requests
```

### 2. Cherry Studio 配置

打开 `Cherry Studio -> 设置 -> MCP 服务器`：

1. 点击 **添加**，填写以下参数：
   - **名称**：`HexStrike-AI`
   - **命令**：`python`
   - **参数**：`C:\hexstrike_mcp.py --server http://<KALI_IP>:8888`

2. 勾选 **启用** 并保存
3. 状态显示"已连接"即表示成功

### 3. Cursor 配置

编辑 Cursor 配置文件，添加 HexStrike MCP 服务器配置。

---

## 5. 使用场景

### 自动化渗透测试

HexStrike AI 可用于：

| 场景 | 说明 |
|------|------|
| 网络扫描 | 自动发现目标网络资产 |
| 漏洞挖掘 | 利用 AI 智能识别和利用漏洞 |
| 攻击链构建 | 自动构建完整攻击链 |
| 渗透报告 | 自动生成渗透测试报告 |

### 优势

1. **提高效率**：比传统手工测试快 16-24 倍
2. **智能化**：AI 驱动的决策引擎自动选择最优工具
3. **协同工作**：多智能体协同，复杂任务分工处理
4. **易于扩展**：基于 MCP 协议，易于集成新工具

---

## 6. 注意事项

- 本工具仅供学习研究使用
- 请勿用于未授权的渗透测试
- 使用前请确保遵守当地法律法规

---

> 来源：SecCeo（泷羽Sec-Ceo）

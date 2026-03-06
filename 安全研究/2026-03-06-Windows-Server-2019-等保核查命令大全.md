---
title: Windows Server 2019 等保核查命令大全
date: 2026-03-06
category: 网络安全
tags: [等保, Windows Server, 安全核查, 等级保护]
source: https://mp.weixin.qq.com/s/ikYHPEh-AkaPxw8fBUG5KA
---

# Windows Server 2019 等保核查命令大全

## 一、问题现状

### 背景描述

随着《网络安全法》和等级保护 2.0 制度的深入推进，Windows Server 系列操作系统已成为国内政企单位的主流服务器平台。在进行等级保护测评时，准确、全面的安全核查命令是确保系统符合安全要求的关键环节。

### 痛点分析

1. **命令过时与覆盖不全**：现有检查命令存在过时与覆盖不全的问题
2. **缺乏针对性验证**：互联网上流传的检查命令普遍泛化，缺乏针对性与验证，在实际测试环境中不正确的检查命令会导致不必要时间的浪费
3. **缺乏系统性维护**：网络公开的检查方法整体缺乏系统性与持续维护

> 💡 **测试环境**：Windows Server 2019 (cn_windows_server_2019_updated_july_2020_x64)

---

## 二、解决思路与方案对比

### 等保核查框架

根据 GB/T 28448-2019《信息安全技术 网络安全等级保护测评要求》，Windows Server 2019 等保核查主要涵盖以下安全领域：

| 安全领域 | 核查重点 | 命令工具 |
|---------|---------|---------|
| 身份鉴别 | 账号管理、登录失败处理、密码策略 | compmgmt.msc、control admintools |
| 访问控制 | 权限划分、最小权限、NTFS 权限 | net share、quser |
| 安全审计 | 审计策略、日志记录 | compmgmt.msc、eventvwr.msc |
| 入侵防范 | 端口扫描、补丁管理、服务状态 | netstat、control panel |
| 恶意代码防范 | 安全软件部署 | 访谈确认 |
| 可信验证 | 系统文件完整性 | sfc、DISM |
| 数据安全 | 加密存储、传输加密 | BitLocker、TLS 配置 |

---

## 三、最佳实践框架

### 3.1 身份鉴别（Account Authentication）

#### 1.1 账号管理
```bash
# 查看系统账户
compmgmt.msc

# 查看系统组和权限划分
compmgmt.msc → Local Users and Groups → Groups
```

#### 1.2 登录失败处理
```bash
# 查看账户锁定策略
control admintools → Local Security Policy → Account Lockout Policy
```

#### 1.3 远程管理通信加密

**WinRM/PowerShell Remoting**：
```powershell
# 查看 WinRM 监听器配置
winrm enumerate winrm/config/listener

# 检查加密配置
winrm get winrm/config/service

# 查看客户端配置
winrm get winrm/config/client
```

**RDP 加密**：
```bash
gpedit.msc
# 计算机配置 → 管理模板 → Windows 组件 → 远程桌面服务 → 远程桌面会话主机 → 安全
# 设置"要求使用特定的安全层进行远程（RDP）连接"为 SSL (TLS)
```

#### 1.4 密码复杂度策略
```bash
control admintools → Local Security Policy → Password Policy
```

#### 1.7 登录超时
```bash
gpedit.msc
# 计算机配置 → 管理模板 → Windows 组件 → 远程桌面服务 → 远程桌面会话主机 → 会话时间限制
```

### 3.2 访问控制（Access Control）

#### 2.1-2.5 账户权限审计
结合账号管理检查内容，判断是否实现特权分离（管理员、审计员、操作员等角色分离）。

#### 2.3 共享账户查询
```cmd
# 查看当前会话连接状态
quser

# 查看系统共享
net share
```
⚠️ 重点检查 C$、D$、ADMIN$ 等管理共享是否必要保留。

#### 2.6 访问控制粒度
- **NTFS 权限**：文件和文件夹的精细化访问控制
- **UAC**：用户账户控制，防止未经授权的系统更改
- **RBAC**：通过用户组机制实现基于角色的访问控制

#### 2.7 安全标记
Windows 默认依赖 DAC（自主访问控制），可通过对主体和客体设置安全标记实现强制访问控制。

### 3.3 安全审计（Security Audit）

#### 3.1 审计策略配置
```bash
control admintools → Local Security Policy → Audit Policy
```

#### 3.2 审计日志核查
```bash
compmgmt.msc → Event Viewer → Windows Logs
# 查看 Application、Security、System 日志
# 检查日志大小和保留周期（建议≥6个月）
```

### 3.4 入侵防范（Intrusion Prevention）

#### 4.1 最小安装原则
```bash
control panel → Programs and Features
# 检查是否安装了必要工作以外的软件
```

#### 4.2 高危端口查看
```cmd
netstat -ano
```

#### 4.3 终端接入审计
检查是否部署了 EDR（终端检测与响应）系统。

#### 4.4 补丁查看
```bash
control panel → Programs and Features → View installed updates
```

#### 4.5 安全服务检查
```bash
compmgmt.msc → Services and Applications → Services
# 检查以下服务是否关闭：
# - Print Spooler
# - Task Scheduler
# - Telephony
```

#### 4.6 防火墙状态
```bash
control panel → Windows Defender Firewall
```

### 3.5 恶意代码防范

#### 5.1 第三方安全服务
访谈确认是否安装第三方安全软件，记录安全软件和病毒库版本。

### 3.6 可信验证

#### 6.1 系统文件完整性
```cmd
# 扫描受保护的系统文件
sfc /scannow

# 检查 Windows 组件存储区健康状况
DISM /Online /Cleanup-Image /RestoreHealth
```
> ⚠️ DISM 命令需要网络连接以从微软服务器下载修复文件。

### 3.7 数据传输安全

#### 7.1 TLS/SSL
- **保密性**：AES、ChaCha20 对称加密
- **完整性**：HMAC 或 AEAD（如 AES-GCM）
- **身份认证**：X.509 证书验证
> ✅ Windows 默认启用 TLS 1.2/1.3

#### 7.2 SMB 安全
- SMB 3.0+ 支持 AES-128/256 加密
- SMB 签名防止数据篡改和重放攻击

#### 7.3 Kerberos（域环境）
- 票据加密防止凭据泄露
- 时间戳和校验和防止重放攻击

#### 7.4 WinRM/PowerShell Remoting
- 默认 Kerberos + 消息层加密
- HTTPS 监听器使用 TLS 加密

### 3.8 数据存储安全

#### 8.1 BitLocker 全盘加密
- AES 128/256 位加密
- 结合 TPM 验证启动过程完整性
- 支持 Startup Key + PIN 多因素解锁

### 3.9 数据备份恢复

#### 9.1-9.3 备份策略
访谈确认：
- 本地备份是否存在且符合异地备份要求
- 是否存在冗余配置（主备/热备）

### 3.10 剩余信息保护

#### 10.1 鉴别信息内存清除
```bash
control admintools → Local Security Policy → Security Options
# 启用"关机前清除虚拟内存页面"
```

#### 10.2 敏感数据内存清除
```bash
control admintools → Local Security Policy → Security Options
# 启用"交互式登录：不显示上次登录的用户名"
```

---

## 四、具体操作步骤

### Step 1：身份鉴别核查

```bash
# 1. 打开本地安全策略
control admintools

# 2. 检查账户锁定策略
账户锁定策略 → 账户锁定阈值、锁定时间

# 3. 检查密码策略
密码策略 → 密码复杂度、最小长度

# 4. 检查本地用户和组
compmgmt.msc → Local Users and Groups
```

### Step 2：访问控制核查

```cmd
# 查看当前连接用户
quser

# 查看共享
net share

# 检查管理共享状态
reg query "HKLM\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters" /v AutoShareWks
```

### Step 3：安全审计核查

```cmd
# 打开事件查看器
compmgmt.msc → Event Viewer

# 查看安全日志属性
Security → Properties → Maximum log size、Log retention
```

### Step 4：入侵防范核查

```cmd
# 查看开放端口
netstat -ano | findstr LISTENING

# 查看系统服务状态
sc query type= service state= all

# 查看防火墙状态
netsh advfirewall show allprofiles
```

### Step 5：可信验证

```cmd
# 系统文件完整性检查
sfc /scannow

# 组件存储区健康检查
DISM /Online /Cleanup-Image /ScanHealth
DISM /Online /Cleanup-Image /RestoreHealth
```

---

## 五、总结与建议

### 成效回顾

通过系统化的命令核查，可全面评估 Windows Server 2019 安全配置是否符合等级保护要求，重点覆盖：

- 身份鉴别与访问控制的安全基线
- 审计日志的完整性和保留周期
- 入侵防范和恶意代码防护措施
- 数据传输和存储的加密机制

### 后续维护建议

1. **定期核查**：建议每季度进行一次安全基线核查
2. **补丁管理**：建立补丁更新机制，及时安装安全更新
3. **日志审计**：配置日志集中收集和分析平台
4. **配置基线**：建立安全配置基线文档，统一标准
5. **持续更新**：关注微软安全公告，及时调整核查策略

### 参考标准

- GB/T 22239-2019《信息安全技术 网络安全等级保护基本要求》
- GB/T 28448-2019《信息安全技术 网络安全等级保护测评要求》
- GB/T 43697-2024《数据安全技术 数据分类分级规则》

---

> 📥 **获取离线 PDF**：关注公众号"汤池杂货铺"，回复关键字"等级保护"

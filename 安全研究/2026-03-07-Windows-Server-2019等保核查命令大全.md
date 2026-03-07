---
title: Windows Server 2019 等保核查命令大全
date: 2026-03-07
category: 安全研究
tags: [等保, Windows Server, 安全核查, 加固]
---

# Windows Server 2019 等保核查命令大全

> 来源：微信公众号「汤池杂货铺」
> 测试环境：Windows Server 2019

---

## 一、身份鉴别

### 1.1 账号管理
```cmd
compmgmt.msc  # 查看系统账户
compmgmt.msc → group  # 查看权限组划分
```

### 1.2 登录失败处理
```cmd
control admintools  # 本地安全策略 → 账户锁定策略
```

### 1.3 远程管理通信加密

#### WinRM / PowerShell Remoting
```powershell
winrm enumerate winrm/config/listener  # 查看监听器配置
winrm get winrm/config/service         # 检查服务配置
winrm get winrm/config/client          # 检查客户端配置
```
- **Transport = HTTP**：默认使用 Kerberos + 消息层加密
- **Transport = HTTPS**：使用 TLS 加密
- **AllowUnencrypted = true**：极不安全，需禁止

#### RDP 加密
```cmd
gpedit.msc  # 计算机配置 → 管理模板 → Windows 组件 → 远程桌面服务 → 远程桌面会话主机 → 安全
```
- 安全层设置为 **SSL (TLS 1.0/1.1/1.2/1.3)**
- 加密级别设置为 **高（128位或更高）**

#### WMI 远程通信
- 默认通过 **DCOM** 或 **WinRM** 传输
- 推荐使用 WinRM 加密机制

### 1.4 密码复杂度策略
```cmd
control admintools  # 本地安全策略 → 密码策略
```

### 1.5 双因素认证
访谈确认 + 现场验证

### 1.7 登录超时
```cmd
gpedit.msc  # 计算机配置 → 管理模板 → Windows 组件 → 远程桌面服务 → 远程桌面会话主机 → 会话时间限制
```

---

## 二、访问控制

### 2.1-2.5 账户权限审计
结合 1.1 账号管理结果判断：
- 是否进行特权分离（管理员、审计员、操作员等角色）
- 是否有共享账户
- 是否遵循最小权限原则

### 2.3 共享账户/文件查询
```cmd
quser        # 查看当前会话
net share    # 查看共享目录
```

### 2.6 访问控制粒度
- **NTFS 权限**：文件和文件夹的精细权限控制
- **UAC**：限制标准用户和管理员权限
- **RBAC**：通过用户组机制实现基于角色的访问控制

### 2.7 强访问控制
Windows 默认依赖 **自主访问控制（DAC）**，可设置安全标记控制资源访问

---

## 三、安全审计

### 3.1 检查审计服务配置
```cmd
control admintools  # 本地安全策略 → 审计策略
```

### 3.2 审计记录核查
```cmd
compmgmt.msc  # 事件查看器 → Windows日志 → 应用/安全/系统
```
- 查看日志大小、轮转配置
- 确保存储不少于 6 个月

---

## 四、入侵防范

### 4.1 最小安装原则
```cmd
control panel  # 程序和功能
```

### 4.2 高危端口查看
```cmd
netstat -ano
```

### 4.3 终端接入审计
检查是否安装 EDR

### 4.4 补丁查看
```cmd
control panel  # 程序和功能 → 已安装的更新
```

### 4.5 安全服务
```cmd
compmgmt.msc  # 服务与应用 → 服务
```
需检查：`print spooler`、`task scheduler`、`telephony` 是否开启

### 防火墙
```cmd
control panel  # Windows Defender Firewall
```

---

## 五、恶意代码防范

### 5.1 第三方安全服务
访谈确认，检查第三方安全软件和病毒库版本

---

## 六、可信验证

### 系统文件检查
```cmd
sfc /scannow                    # 扫描系统文件
DISM /Online /Cleanup-Image /RestoreHealth  # 修复系统映像
```

---

## 七、数据传输保密性和完整性

### 7.1 TLS/SSL
- Windows 默认启用 **TLS 1.2/1.3**
- 使用 **AES、ChaCha20** 对称加密
- 通过 **HMAC/AEAD** 确保完整性

### 7.2 SMB 协议安全
- **SMB 3.0+** 支持 AES-128/256 加密
- SMB 签名防止数据篡改和重放

### 7.3 Kerberos 认证
- 票据使用会话密钥加密
- 时间戳和校验和防止重放攻击

### 7.4 WinRM / PowerShell Remoting
- 默认 **Kerberos + 消息层加密**
- 配置 HTTPS 监听器使用 TLS 加密
- `AllowUnencrypted = false` 强制加密

---

## 八、数据存储保密性和完整性

### 8.1 BitLocker
- 全盘加密（AES 128/256）
- 结合 TPM 验证启动过程
- 支持 Startup Key + PIN 多因素解锁

---

## 九、数据备份恢复

- 本地备份：访谈确认
- 异地备份：访谈确认
- 冗余配置：主备/热备

---

## 十、剩余信息保护

### 10.1 鉴别信息内存清除
```cmd
control admintools  # 本地安全策略 → 安全操作 → 关机前清除虚拟内存页面
```

### 10.2 敏感数据内存清除
```cmd
control admintools  # 交互式登录 → 不显示上次登录的用户名
```

---

## 十一、个人信息保护

根据 GB/T 28448-2019 测评要求判断

---

*本文档由 Agent 自动从微信公众号提取并整理*

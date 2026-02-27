---
title: Mandiant 发布 Net-NTLMv1 彩虹表 - 加速协议废止
date: 2026-02-24
category: 网络安全
tags: [Net-NTLM, rainbow-tables, 凭证窃取, 认证协议, 渗透测试]
source: https://cloud.google.com/blog/topics/threat-intelligence/net-ntlmv1-deprecation-rainbow-tables/
---

## 摘要

Mandiant 公开发布了 Net-NTLMv1 彩虹表数据集，旨在强调淘汰这种过时协议的紧迫性。尽管 Net-NTLMv1 已被废弃并已知不安全超过 20 年，但 Mandiant 安全顾问仍在活跃环境中发现其使用。该数据集允许攻击者在不到 12 小时内使用消费级硬件（成本低于 600 美元）恢复密钥。

## 背景

### 协议历史

- **Net-NTLMv1** 不安全自 **2012 年** 起广为人知（DEFCON 20 演讲）
- 底层协议密码分析可追溯到 **1999 年**
- 2016年8月30日，Hashcat 添加了对 DES 密钥的破解支持
- 彩虹表概念始于 **2003 年** (Philippe Oechslin)

### 漏洞原理

如果攻击者能在已知明文 `1122334455667788` 且无 Extended Session Security (ESS) 的情况下获取 Net-NTLMv1 哈希，可实施**已知明文攻击 (KPA)** 保证恢复密钥材料。

由于密钥材料是 Active Directory (AD) 对象（用户或计算机）的密码哈希，攻击结果可快速用于 compromise 该对象，常导致权限提升。

## 攻击链分析

```
1. 认证胁迫 → 使用 Responder、PetitPotam、DFSCoerce
2. 获取 Net-NTLMv1 哈希 (固定明文: 1122334455667788)
3. 预处理哈希为 DES 组件 (ntlmv1-multi)
4. 使用彩虹表破解 DES 密钥 (<12小时)
5. 重组完整 NT 哈希
6. DCSync 攻击 → 获取任意账户凭据
```

### 典型攻击场景

攻击者常使用**域控制器 (DC)** 的高权限对象进行认证胁迫。获取 DC 机器账户的密码哈希后，可实现 DCSync 权限， compromise AD 中的任何账户。

## 数据集发布

### 获取方式

```bash
# 通过 gsutil 下载
gsutil -m cp -r gs://net-ntlmv1-tables/tables .

# 或通过 Google Cloud Research Dataset 门户
# https://research.google/resources/datasets/?dataset_types=other&search=Net-NTLMv1&

# 校验完整性
gsutil -m cp gs://net-ntlmv1-tables/tables.sha512 .
sha512sum -c tables.sha512
```

### 使用工具

| 工具 | 用途 |
|------|------|
| [rainbowcrack](https://www.kali.org/tools/rainbowcrack/) | CPU 彩虹表破解 |
| [RainbowCrack-NG](https://github.com/inAudible-NG/RainbowCrack-NG) | 现代彩虹表软件 |
| [rainbowcrackalack](https://github.com/jtesta/rainbowcrackalack) | GPU 加速破解 |
| [ntlmv1-multi](https://github.com/evilmog/ntlmv1-multi) | 预处理哈希为 DES 组件 |
| [twobytes](https://github.com/sensepost/assless-chaps) | 计算剩余密钥 |

## 缓解措施

### 1. 禁用 Net-NTLMv1

在域控制器上设置：

```powershell
# Windows Server 2019+
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\DefaultSecurity" -Name "Srv2" -PropertyType DWORD -Value 2 -Force

# 或通过 GPO 限制
```

### 2. 启用 SMB 签名

强制 SMB 签名防止中间人攻击：

```powershell
Set-SmbServerConfiguration -RequireSecuritySignature $true
```

### 3. 阻断 LLMNR/NBT-NS

```powershell
# 禁用 LLMNR
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Name "EnableMulticast" -Value 0 -Type DWord

# 禁用 NetBIOS over TCP/IP
```

### 4. 网络层控制

- 限制出站 SMB (端口 445)
- 监控异常认证流量
- 部署 EDR 告警规则

## 影响评估

| 维度 | 风险等级 | 说明 |
|------|----------|------|
| 凭证安全 | **严重** | 彩虹表可在 12 小时内破解 |
| 横向移动 | **高** | DCSync 可获取任意凭据 |
| 检测难度 | **中** | 攻击流量与正常认证相似 |
| 修复成本 | **低** | 禁用协议即可 |

## 总结

- Net-NTLMv1 是**已知不安全但仍在生产环境使用**的遗留协议
- Mandiant 发布的彩虹表使攻击**平民化**（<$600 硬件 + <12小时）
- 建议**立即禁用** Net-NTLMv1 并部署 SMB 签名
- 监控 LLMNR/NBT-NS 流量及异常认证行为

## 参考资料

- [Mandiant Net-NTLMv1 Dataset](https://research.google/resources/datasets/?dataset_types=other&search=Net-NTLMv1&)
- [Hashcat Net-NTLM Cracking](https://github.com/hashcat/hashcat/commit/71a8459d851d246945343ea59effa1d46b965bf8)
- [Rainbow Tables Paper (Oechslin, 2003)](https://infoscience.epfl.ch/record/99512/files/Oech03.pdf)
- [MITRE ATT&CK - DCSync](http://attack.mitre.org/techniques/T1003/006/)

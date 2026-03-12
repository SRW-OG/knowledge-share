---
title: Linux系统+MySQL数据库三级等保安全加固
date: 2026-03-07
category: 网络安全
tags: [等保, 安全加固, Linux, MySQL, 三级等保, 主机安全]
source: https://mp.weixin.qq.com/s/UoshBod88mqh3sTr0VsS0w
---

# Linux系统+MySQL数据库三级等保安全加固

> 来源：微信公众号「运维技术后花园」
> 作者：seaskyccl
> 日期：2026年2月8日 10:49

---

## 一、Linux 系统安全加固

### 1.1.1.1 身份鉴别

#### a) 用户身份标识与鉴别

**要求**：应对登录的用户进行身份标识和鉴别，身份标识具有唯一性，身份鉴别信息具有复杂度要求并定期更换。

**检查方法**：
```bash
cat /etc/passwd
```

**问题诊断**：
- 若第一字段相同，即存在同名账户
- 若第二字段为空，即存在空口令用户
- 若第三字段相同，即用户身份 UID 不具有唯一性

**修复命令**：
```bash
# 修改用户名
# usermod -l 新用户名 旧用户名

# 修改密码
# passwd 用户名

# 修改 UID
# usermod -u 新uid 用户名
```

**实操示例**：
```bash
# 把 test 用户改名为 geek
# usermod -l geek test

# 修改用户 geek 的密码
# passwd geek

# 把 geek 用户的 uid 改成 1234
# usermod -u 1234 geek
```

#### b) 密码复杂度修改

**要求**：管理员密码长度至少 8 位，具有一定的复杂度并定期更换。

**配置文件**：`/etc/login.defs`

**参数说明**：
| 参数 | 值 | 说明 |
|------|-----|------|
| PASS_MAX_DAYS | 90 | 密码最长有效期 90 天 |
| PASS_MIN_DAYS | 2 | 密码修改间隔最小天数 |
| PASS_MIN_LENS | 8 | 密码最小长度 8 位 |
| PASS_WARN_AGE | 7 | 密码失效前 7 天开始通知 |

**编辑命令**：
```bash
# vim /etc/login.defs
```

**PAM 密码复杂度配置**：`/etc/pam.d/system-auth`

**添加规则**：
```bash
password requisite pam_cracklib.so retry=3 minlen=8 difok=2 dcredit=-1 ucredit=-1 lcredit=-1 ocredit=-1
```

**参数说明**：
| 参数 | 说明 |
|------|------|
| retry=3 | 连续3次输入密码强度不够则退出 |
| minlen=10 | 密码长度必需大于6（10-类型数量） |
| difok=3 | 新密码至少有3个字符与旧密码不同 |
| dcredit=-1 | 新密码中至少有1个数字 |
| ucredit=-1 | 新密码中至少有1个大写字符 |
| lcredit=-1 | 新密码中至少有1个小写字符 |
| ocredit=-1 | 新密码中至少有1个符号 |

#### c) 登录失败处理

**要求**：应具有登录失败处理功能，应配置并启用结束会话、限制非法登录次数和当登录连接超时自动退出等相关措施。

**PAM 模块**：`pam_tally2.so`

**配置命令**：
```bash
# vim /etc/pam.d/system-auth
```

**添加规则**：
```
account required pam_tally2.so deny=5 lock_time=300 unlock_time=300 magic_root even_deny_root
```

**参数说明**：
- `deny=5` - 连续5次登录失败后锁定
- `lock_time=300` - 锁定300秒
- `unlock_time=300` - 解锁时间300秒
- `even_deny_root` - root用户也受限制

#### d) 远程管理安全

**要求**：当进行远程管理时，应采取必要措施防止鉴别信息在网络传输过程中被窃听。

**解决思路**：
1. 关闭 Telnet、FTP 等明文传输服务
2. 使用 SSH 加密协议进行远程登录和管理

**检查命令**：
```bash
netstat -an | grep -E ':(21|22|23)'
```

| 端口 | 服务 | 说明 |
|------|------|------|
| 21 | FTP | 明文传输，需关闭 |
| 22 | SSH | 加密传输，推荐 |
| 23 | Telnet | 明文传输，需关闭 |

---

## 二、MySQL 数据库安全加固（待续）

> 由于文章内容较长，建议关注微信公众号「运维技术后花园」获取完整内容。

---

## 参考来源

- 作者：seaskyccl
- 来源：微信公众号「运维技术后花园」
- 日期：2026年2月8日 10:49
- 地点：广东

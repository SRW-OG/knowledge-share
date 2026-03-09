---
title: Windows安全攻防-DLL劫持
date: 2026-03-09
category: 网络安全
tags: [DLL劫持, 渗透测试, 后门, Windows安全]
source: https://mp.weixin.qq.com/s/kN7Iqia3BzFrFTx4WswjLg
author: 剑外思归客
---

# Windows安全攻防-DLL劫持

## 1. 什么是DLL文件

DLL的全称是Dynamic Link Library，中文叫做"动态链接文件"。在Windows操作系统中，DLL对于程序执行是非常重要的，因为程序在执行的时候，必须链接到DLL文件，才能够正确地运行。而有些DLL文件可以被许多程序共用。因此，程序设计人员可以利用DLL文件，使程序不至于太过巨大。但是当安装的程序越来越多，DLL文件也就会越来越多。DLL文件和EXE文件同样可以由编译语言生成，但是DLL没有程序启动入口，所以DLL文件不可执行。

---

## 2. DLL优先加载目录顺序

Windows查找DLL的目录以及对应的顺序（SafeDllSearchMode 默认会被开启）：

### SafeDllSearchMode 配置

注册表项：`HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Session Manager\SafeDllSearchMode`

| 值 | 说明 |
|----|------|
| 值为 1 | 启用安全 DLL 搜索模式（默认行为） |
| 值为 0 | 禁用安全 DLL 搜索模式 |

### DLL搜索顺序（Safe DLL search mode 启用时）

1. **进程对应的应用程序所在目录**（程序安装目录如C:\Program Files\uTorrent）
2. **系统目录**（%windir%\system32）
3. **16位系统目录**（%windir%\system）
4. **Windows目录**（%windir%）
5. **当前目录**（运行的某个文件所在目录）
6. **PATH环境变量中的各个目录**

### KnownDLLs

Windows7以上系统默认开启SafeDllSearchMode，同时采用了KnownDLLs。如果要加载的DLL名称命中KnownDLLs列表，系统会优先使用系统提供的已知DLL（通常来自系统目录对应的已知映射），从而避免从应用目录/当前目录等位置加载同名DLL。

注册表位置：`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs`

---

## 3. DLL劫持用到的工具

### Process Monitor

Process Monitor 是一款由 Sysinternals 公司开发的包含强大的监视和过滤功能的高级 Windows 监视工具，可实时显示文件系统、注册表、进程/线程的活动。该工具可以查看应用程序都调用了那些DLL。

### 火绒剑

中国版系统监控工具，可查看程序DLL调用情况。

### 自动查找工具

- **Rattler**：https://github.com/sensepost/rattler
- 该工具需要指定出需要查找的应用程序，进而查看该程序是否存在可以利用的DLL文件

### Aheadlib注入工具

工具地址：https://bbs.kanxue.com/thread-224408.htm

这个工具可以将找到的可以利用的dll文件导入进去，自动生成一个cpp代码，这个cpp代码本质上是将原dll的函数名提取出来，进而引用进来。原dll不要删除，但是要修改一下，比如原名称为 `ffmpeg.dll` 要修改为 `ffmpegOrg.dll`。

---

## 4. DLL劫持

### 原理

通常情况下，程序加载DLL文件并不会指定DLL位置，所以程序根据相应的系统规则来查找DLL文件。DLL劫持本质上是通过DLL加载顺序的漏洞，伪造一个DLL文件，将其放置在应用程序目录内，该DLL文件包含了正常的DLL文件的功能，但是攻击者夹带私货，进而执行指定的后门程序。

### 劫持条件

1. **想要劫持的DLL不能在注册表** `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs` 中
2. **其dll是EXE程序首先加载的DLL**，而不是依赖其他DLL加载的
3. **DLL确实被加载进内存中**

可以通过Process Monitor或者火绒剑查看软件都进行了哪些DLL调用。

---

## 5. 实践操作

### 步骤1：使用Cobalt Strike生成带有后门的DLL文件

1. 创建监听器
2. 生成 DLL 文件（命名为 test.dll）

### 步骤2：寻找可以劫持的DLL

以Typora为例，使用火绒剑查看Typora都调用了哪些DLL文件。发现一个很明显的 `ffmpeg.dll` 不是来自于系统文件，那就更不可能在注册表中出现了，由此可以猜测该DLL可能存在被劫持的风险。

### 步骤3：使用Aheadlib生成cpp代码

1. 使用Aheadlib打开原dll文件（ffmpeg.dll）
2. 将生成的cpp代码复制下来

### 步骤4：生成DLL文件

1. 打开VS编辑器，创建一个cpp项目，项目名称为 `ffmpeg`
2. 删除无关文件
3. 项目属性 → C/C++ → 代码生成 → 运行库修改为多线程(/MT)
4. 预编译头改为不使用预编译
5. 将生成的cpp代码粘贴进去
6. 添加执行代码

### 弹出计算器的代码示例

```cpp
STARTUPINFO si = { sizeof(si) };
PROCESS_INFORMATION pi;
CreateProcess(TEXT("C:\\Windows\\System32\\calc.exe"), NULL, NULL, NULL, NULL, NULL, NULL, NULL, false, 0, NULL, NULL, NULL, &si, &pi);
```

### 加载其他DLL的代码

```cpp
LoadLibraryA("test.dll"); // 加载一个test.dll的文件
```

---

## 6. 防御建议

| 措施 | 说明 |
|------|------|
| 最小化权限 | 应用程序使用最小权限运行 |
| 定期更新 | 及时更新系统和应用程序 |
| 白名单机制 | 使用应用程序白名单 |
| DLL签名 | 启用DLL签名验证 |
| 安全软件 | 使用杀软/EDR检测异常DLL加载 |

---

> 来源：R0x7e（剑外思归客）

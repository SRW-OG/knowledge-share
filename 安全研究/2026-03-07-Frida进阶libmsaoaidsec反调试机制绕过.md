---
title: Frida进阶——libmsaoaidsec.so反调试机制绕过
date: 2026-03-07
category: 网络安全
tags: [Frida, Android逆向, 反调试, libmsaoaidsec, 绕过, hook]
source: https://mp.weixin.qq.com/s/3p72982U6pmI6vIy_lyIgw
---

# Frida进阶——libmsaoaidsec.so反调试机制绕过

> 作者：泡泡以安
> 来源：微信公众号「泡泡以安」
> 日期：2026年2月27日 17:30
> 地点：浙江

---

## 一、反调试分析

许多应用通过集成 **libmsaoaidsec.so** 库来防范 Frida 调试。在应用启动时，加载该库并通过 `pthread_create()` 创建一个新线程，在此线程中执行反调试检查，包括扫描 Frida 的进程、端口和内存特征。如果检测到 Frida，程序会终止或崩溃。

---

## 二、Hook android_dlopen_ext

这里我们需要先 Hook Android 的动态链接库加载函数 `android_dlopen_ext`，观察它加载到哪个 so 库文件的时候崩溃的（本文案例加载 `libmsaoaidsec.so` 时会崩溃）。

### 核心代码

```javascript
function hook_dlopen() {
    Interceptor.attach(Module.findExportByName(
        null, "android_dlopen_ext"), {

        onEnter: function (args) {
            this.fileName = args[0].readCString()
            console.log(`dlopen onEnter: ${this.fileName}`)
        },
        onLeave: function (retval) {
            console.log(`dlopen onLeave fileName: ${this.fileName}`)

            if (this.fileName != null && this.fileName.indexOf("libmsaoaidsec.so") >= 0) {
                let JNI_OnLoad = Module.getExportByName(
                    this.fileName, 'JNI_OnLoad')

                console.log(`dlopen onLeave JNI_OnLoad: ${JNI_OnLoad}`)
            }
        }
    });
}

setImmediate(hook_dlopen)
```

### 分析结果

- 最后一个加载的 so 是 `libmsaoaidsec.so`
- 没有调用 `onLeave`，由此可知崩溃点就在 `libmsaoaidsec.so` 中
- 检测点在 `JNI_OnLoad` **之前**
- so 在加载之后会先调用 `.init_proc` 函数，接着调用 `.init_array` 中的函数，最后才是 `JNI_OnLoad` 函数

### 注入时机选择

检测点在 `JNI_OnLoad` 之前，注入时机可以选择在 `dlopen` 加载 `libmsaoaidsec.so` **之后**。

需要注意的一点是：在 `dlopen` 函数调用完成之后 `.init_xxx` 已经执行完成了。所以选择 Hook `call_constructors` 函数。

---

## 三、SO库加载函数详解

| 函数名 | 描述 |
|--------|------|
| `android_dlopen_ext()`、`dlopen()`、`do_dlopen()` | 这三个函数主要用于加载库文件。`android_dlopen_ext` 是系统的一个函数，用于在运行时动态加载共享库。与标准的 `dlopen()` 函数相比，`android_dlopen_ext` 提供了更多的参数选项和扩展功能，例如支持命名空间、符号版本等特性。 |
| `find_library()` | `find_library()` 函数用于查找库，基本的用途是给定一个库的名字，然后查找并返回这个库的路径。 |
| `call_constructors()` | `call_constructors()` 是用于调用动态加载库中的构造函数的函数。 |
| `.init` | 库的构造函数，用于初始化库中的静态变量或执行其他需要在库被加载时完成的任务。如果没有定义 init 函数，系统将不会执行任何动作。需要注意的是，init 函数不应该有任何参数，并且也没有返回值。 |
| `.init_array` | `init_array` 是 ELF 二进制格式中的一个特殊段，这个段包含了一些函数的指针，这些函数将在 `main()` 函数执行前被调用，用于初始化静态局部变量和全局变量。 |
| `.JNI_OnLoad` | 这是 Android JNI 中的一个函数。当一个 native 库被系统加载时，该函数会被自动调用。`JNI_OnLoad` 可以做一些初始化工作，例如注册你的 native 方法或者初始化一些数据结构。如果你的 native 库没有定义这个函数，那么 JNI 会使用默认的行为。`JNI_OnLoad` 的返回值应该是需要的 JNI 版本，一般返回 `JNI_VERSION_1_6`。 |

---

## 四、总结

| 阶段 | 关键技术 |
|------|----------|
| 反调试分析 | Hook `android_dlopen_ext` 定位崩溃点 |
| 崩溃定位 | 检测点在 `JNI_OnLoad` 之前 |
| 注入时机 | Hook `call_constructors` 函数 |
| 绕过原理 | 在反调试检测执行前注入 Frida |

---

## 参考来源

- 作者：泡泡以安
- 来源：微信公众号「泡泡以安」
- 日期：2026年2月27日 17:30
- 地点：浙江

---
layout: post
title: "微信扫码登录微信读书失败"
---

这里开始写正文……

## 一、问题

在红米（Redmi）手机上使用微信扫描微信读书的登录二维码时，微信无法正常打开授权页面，提示：

```
无法打开页面

错误信息：
-5 net::ERR_PROXY_CONNECTION_FAILED
```

起初怀疑是 VPN 或系统代理设置导致，但关闭 VPN 后，问题仍然存在。检查 Wi-Fi 代理设置，也没有发现异常。

最终通过 ADB 强制终止微信进程，再重新启动微信，扫码登录立即恢复正常。

**问题的关键不在于系统是否仍然启用了代理，而在于微信运行期间可能保留了已经失效的网络状态。**

## 二、分析

错误信息中最重要的是：

```
net::ERR_PROXY_CONNECTION_FAILED
```

根据 Chromium 源码，该错误表示程序无法建立到代理服务器的连接，可能发生在代理服务器名称解析或网络套接字连接阶段。

Chromium 中对应的原生错误码是 `-130`。

需要注意，错误页面中同时出现的 `-5` 不应直接理解为 Chromium 的错误码。Android WebView 另有一套错误码体系，其中 `-5` 定义为 `ERROR_PROXY_AUTHENTICATION`，即代理认证失败。

因此，单凭这条错误信息，可以判断代理连接链路出现了异常，但不能据此确定是哪一层配置出错，更不能直接断言微信读书服务器故障。

## 三、排查

### 1. 关闭 VPN：无效

首先关闭手机上的 VPN，重新扫描微信读书登录二维码。

结果仍然出现：

```
-5 net::ERR_PROXY_CONNECTION_FAILED
```

说明单纯关闭 VPN 并没有清除故障。

但这并不意味着 VPN 一定与问题无关。某些应用可能在网络切换后继续使用此前建立的连接或保留原来的网络配置状态。

### 2. 检查 Wi-Fi 代理：无异常

进入红米手机当前连接的 Wi-Fi 详情页面，检查代理设置。

```
代理：无
```

这一结果排除了当前 Wi-Fi 网络通过该设置指定手动代理的情况。

不过，Android 的代理配置不只存在于 Wi-Fi 设置中，还可能存在系统全局代理、应用内部代理或其他网络组件中，因此需要进一步验证。

### 3. 使用 ADB 检查系统全局代理

在 Windows 电脑上下载 Google 官方的 [Android SDK Platform-Tools](https://developer.android.com/tools/releases/platform-tools)，通过 USB 连接手机，并开启 USB 调试。

首先执行：

```
adb devices
```

结果：

```
List of devices attached
********    device
```

说明 ADB 已经正常识别手机。

随后分别检查四项系统代理配置：

```
adb shell settings get global http_proxy
```

输出：

```
null
```

继续执行：

```
adb shell settings get global global_http_proxy_host
adb shell settings get global global_http_proxy_port
adb shell settings get global global_proxy_pac_url
```

三项结果同样全部为：

```
null
```

至此可以确认，检查到的 Android 全局代理设置中没有配置代理服务器、端口或自动代理脚本。

结合 Wi-Fi 代理显示「无」的结果，常规系统代理配置已基本排除。但这些检查仍不能证明微信进程内部不存在其他代理状态。

## 四、解决方法

既然系统代理配置正常，问题就可能出在正在运行的微信自身。

通过 ADB 执行：

```
adb shell am force-stop com.tencent.mm
```

其中：

- `adb shell`：在安卓设备上执行命令。
- `am`：调用 Android Activity Manager。
- `force-stop`：强制停止指定应用。
- `com.tencent.mm`：微信的 Android 应用包名。

该命令会强制停止与微信应用关联的进程，不需要 Root，也不需要清除应用数据。

执行完成后，重新打开微信，再次扫描微信读书登录二维码。

**结果：成功登录，错误彻底消失（本次测试）。**

整个修复过程没有修改系统代理配置，没有删除微信数据，也没有重新安装微信。

## 五、可能的解释

根据排查结果，比较合理的解释是：**微信运行中的网页加载组件保留了异常的代理配置或连接状态，而强制停止应用使这些运行时状态失效。**

一种可能的故障过程如下：

1. 微信运行期间，系统曾经启用 VPN 或代理，微信的网络组件使用了相应的网络环境。
2. VPN 关闭或网络发生变化后，系统代理状态已经恢复，但微信内部的某些网络状态没有及时更新。
3. 扫描微信读书二维码时，微信调用内置网页组件加载登录授权页面。
4. 网页组件仍然尝试使用失效的代理连接，最终返回 `ERR_PROXY_CONNECTION_FAILED`。
5. 强制终止微信进程后，原来的内存状态与连接被销毁。重新启动微信时，网络组件重新初始化，授权页面恢复正常。

这一解释与实际现象相符，但必须强调：**目前没有取得微信内部日志，因此不能证明具体是哪个组件缓存了代理状态，也不能确定其内部实现。**

Android 官方 `ProxyController` 文档提供了相关的技术依据：WebView 可以采用应用级的代理覆盖配置，而且代理设置发生变化后，既有网络连接不保证立即采用新配置。

这说明系统代理状态与正在运行的网页组件实际采用的代理状态，理论上可能存在不一致。

## 参考资料

1. [Chromium 源码：net_error_list.h](https://chromium.googlesource.com/chromium/src/+/main/net/base/net_error_list.h) — 网络错误码及 `PROXY_CONNECTION_FAILED` 的定义。
2. [Android WebViewClient 官方文档](https://developer.android.com/reference/android/webkit/WebViewClient) — Android WebView 错误码定义。
3. [AndroidX ProxyController 官方文档](https://developer.android.com/reference/androidx/webkit/ProxyController) — WebView 代理配置及连接更新行为。
4. [Android Debug Bridge 官方文档](https://developer.android.com/tools/adb) — `am force-stop` 命令说明。
5. [Android SDK Platform-Tools](https://developer.android.com/tools/releases/platform-tools) — ADB 官方下载地址。

*说明：本文依据一次实际排障记录整理，适用性未经跨设备、跨微信版本验证。*


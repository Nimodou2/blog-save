> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7408391774635884596)

> 1 问题描述 从 SetupWizard 退出进入 Launcher 的过程中，SetupWizard 的相关界面在退出的动画过程中短暂卡在了某个阶段，如下图所示： 2 问题分析 2.1 log 分析 透过现象看微......

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a8283e193de04bec975b38208fa04896~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749769520&x-signature=oLOwLhJD9H%2BUO4mDoRLTMC3W6jo%3D)

1 问题描述
------

从 SetupWizard 退出进入 Launcher 的过程中，SetupWizard 的相关界面在退出的动画过程中短暂卡在了某个阶段，如下图所示：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/805cb98871e94eceb728dc67e227dde1~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749769520&x-signature=b2QHh4qA28wHzYvdNvXr9Ab9%2Bow%3D)

2 问题分析
------

### 2.1 log 分析

透过现象看本质，看 log 此过程中没有冻屏之类的操作，那么出现长时间卡在某个动画阶段，可能是在动画期间，相关 SurfaceControl 设置 setPosition 不够流畅导致。

在 Transaction.setPosition 方法中加 log 后，先看下正常的情况：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/16d0592107ce4ebdbea67fc446440ef8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749769520&x-signature=29fq4l4w7p%2FFum88W7M%2BilkqV00%3D)

预期的情况是，setPosition 的操作应该尽量在每一帧中都能进行，最终是界面的位置随着时间平滑的变化。

但是出现问题的时候的 log 为：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e6f9eac5202a4516aa49b207875dd34f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749769520&x-signature=82SBLsSK%2BOdIdFvOgGbQGjlnnTg%3D)

就没有几次设置 setPosition 的操作，y 轴的变化也非常突兀，前几帧还算勉强在每一次 Vsync 的时候调用 Transaction.setPosition 为相关 SurfaceControl 设置了 y 轴位置，后面则出现了严重的丢帧，导致相关 SurfaceControl 的 y 轴位置一直没有得到更新。

再看具体的某一次设置 position 的堆栈信息：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/fc4fd138f35a495283e45beadb17cd72~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749769520&x-signature=bx0aizcUHpq8WfulWfNIFTk95ns%3D)

能看到：

1）、进程号 2023 对应的是 SystemUI，也就是说这个动画是 Transition 动画，并且由位于 SystemUI 的 WMShell 负责执行。

2）、动画的每一帧位置通过 ValueAnimator 在每一个 Vsync 到来的时候计算的得到，至于这个 Vsync 自然是由 Choreographer 来接收。

### 2.2 perfetto 分析

再看 perfetto 的信息：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/2b8c0e3fbe4247a0ba4cf251d013841c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749769520&x-signature=fFI7kDbCJH50OcwgujrtRuX2eNA%3D)

执行动画的线程 2110 收到了几次 Vsync 后，出现了一段很长的空白期，之后才继续收到了几次 Vsync，完成了动画，这一点符合我们观察到的现象以及 log 的情况。

至于为什么动画线程很长的一段时间都没有收到 Vsync，看 SystemUI 主线程，则是长时间在处理 configChanged 相关的事务，这阻塞了 Choreographer 接收到下一个 Vsync 的时候动画的执行，动画从而出现了卡顿。

所以原因出在 SetupWizard 的动画还在执行的时候， “com.google.android.setupwizard”和 “com.ts.setupwizard.overlay” 等一些 App 执行了一些 disable App 的操作，触发了全局 Configuration 的更新，导致 SystemUI 主线程去处理了 configChanged 相关的事务导致了阻塞。

### 2.3 全局 Configuration 的更新

最后看下为什么全局 Configuration 发生了更新，对比 AndroidU 的项目，在退出 SetupWizard 进入 Launcher 的时候，是没有发生全局 Configuration 的更新的。

全局 Configuration 更新的 log 为：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a1c777564efe480cab19dfd2cdb8abc7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749769520&x-signature=d5IqvJNHx%2F8tX1WAy2SeG2KqiWA%3D)

更新的原因为 CONFIG_ASSETS_PATHS 这一位发生了变化，看下这个差异是如何产生的。

这里直接放结论，起点在 OverlayManagerService.mService.setEnabledExclusiveInCategory：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7d83e1d4ce4e466fa5a98cadbac3facd~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749769520&x-signature=b9YrJvDlirIBbhHDRlVI8zKndBQ%3D)

这里传入的 packageName 为 “com.android.internal.systemui.navbar.gestural”，这个值定义在 WindowManagerPolicyConstants 中：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/388337f69f6a44f09981030a36a66f72~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749769520&x-signature=QdFTteyKwM6hp2KoaMYOgQmszMw%3D)

应该是给 App 用的，看起来是和导航模式中的手势相关的。

后续的调用堆栈为：

OverlayManagerService.mService.setEnabledExclusiveInCategory

-> OverlaytManagerService.updateTargetPackagesLocked

-> OverlaytManagerService.updateActivityManager

-> ActivityManager.scheduleApplicationInfoChanged

-> ActivityManagerService.scheduleApplicationInfoChanged

-> ActivityManagerService.updateApplicationInfoLOSP

-> ProcessList.updateApplicationInfoLOSP

-> ActivityTaskManagerService.updateAssetConfiguration

最终是在 ActivityTaskManagerService.updateAssetConfiguration 中：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b13e09c473b84d12aca4029b053b9608~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749769520&x-signature=1IWN1vuaXxCAPhRP1nLE9DOeZ0g%3D)

首先调用 ActivityTaskManagerService.increaseAssetConfigurationSeq 中对 Configuration.assetSeq 进行了自增：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/0f3a5185f7c24e969c8f0d6054adc189~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749769520&x-signature=RiuwgC2Jgrj43w7V8mJ5myaqed0%3D)

然后再调用 ActivityTaskManagerService.updateConfiguration 对全局 Configuration 进行了更新。

### 2.4 根本原因

经过第 3 步的分析，看到是那次 OverlayManagerService.mService.setEnabledExclusiveInCategory 的调用造成了全局 Configuration 的更新，然后我本地屏蔽了这次调用，果然不会有 “config changes” 相关的 log 打印了， 并且看手机的现象也正常，没有卡顿的现象发生了，但是本来进入 Launcher 的时候是手势导航模式的，因为我屏蔽了那次调用后也变成了 3 按钮导航的模式，所以 OverlayManagerService.mService.setEnabledExclusiveInCategory 的调用是 SystemUI 那边为了将导航模式从 3 按钮模式切换为手势模式。

所以这个问题发生的根本原因是，在 SetupWizard 退出进入 Launcher 的时候：

1）、首先是 SetupWizard 要执行退出动画，这个动画由位于 SystemUI 的 WMShell 执行，具体为在 SystemUI 的主线程中当 Choreographer 接收到 Vsync 信号后在 animation 阶段执行动画。

2）、SystemUI 将导航模式从 3 按钮切换为了手势，这导致了全局 Configuration 的更新，SystemUI 主线程中需要针对这个变化进行一些耗时操作，这影响到第一步的动画的顺利执行。
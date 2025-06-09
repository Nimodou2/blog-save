> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7408394788065804288)

> 1 问题描述 锁屏界面调起 Emergency 界面，然后返回到锁屏界面，切换的过程中黑屏。 2 问题分析 首先根据复现的情况来看，能看到的很明显的一点就是，动画开始播放的时候，壁纸还没有显示出来。 但是

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/8ed4e62807df44328c451ca281e533e9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=Du2Rnd5nGlVdamtPwS3JGyoIwwk%3D)

1 问题描述
------

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b498c10606a74bce8907244f3363f0d7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=SKwNHVNWZKAaOiDsaMtpMiybgwA%3D)

锁屏界面调起 Emergency 界面，然后返回到锁屏界面，切换的过程中黑屏。

2 问题分析
------

首先根据复现的情况来看，能看到的很明显的一点就是，动画开始播放的时候，壁纸还没有显示出来。

但是根据现有的 log：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4e9d425fe6e144f1838ecfa2f5aee0d5~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=p0Reo47ypxixMdSDQMkGU%2FYp8Jk%3D)

WallpaperWindowToken 也是参与到了动画了，并且动画类型是 TO_FRONT，顺便一提，EmergencyDialerActivity 对应的 Task 是 CLOSE，那么按照我们对动画的了解，应该是 EmergencyDialerActivity 的 Task 在退出的同时，WallpaperWindowToken 应该也是在移动到前台变为可见的，但是实际上并非如此。

接下来需要在 SurfaceFlinger 侧继续加 log 看看究竟是为什么 Wallpaper 无法在动画开始的时候显示出来。

代码分析见第 3 节，排查代码的时候是倒着分析的，但是总结的时候我觉得还是正着梳理好一点。

最终看到是动画开始的时候，WallpaperWindowToken 被 reparent 到的 “transition-leash” 主动请求设置 alpha 为 0：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c48b3f31ed9c4a998b9a3b8cbf94a726~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=x76hHIUqIy11iSzse0ZOMYGSc8M%3D)

导致了在整个动画期间，壁纸都无法显示，只能在动画结束后，WallpaperWindowToken 从 “transition-leash” reparent 回到原来的父 Layer 上时，壁纸才能恢复显示。

具体代码在 KeyguardService 中：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/62f4c11f864d46119c9bac964f4c0e05~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=4RjECkMJ2Fmgmzc0W7iGq0iJee8%3D)

initAlphaForAnimationTargets 方法的内容是：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e0cdaa0991fd4552bcd48dec37c51d12~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=50A%2B78cXLf8SJzd4m45YNf2e7R0%3D)

如果动画类型是 MODE_OPENING，那么将动画的 leash 的透明度设置为 0。

而动画的类型只有 3 种：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/dfd10c36941a46388702c70c86b79969~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=eqXZWeKG0g4baVFt1ZhInjpS7dI%3D)

那么 Wallpaper 对应的 ChangeInfo 的动画为 TO_FRONT，肯定就会被归类为 MODE_OPENING。

顺便一提，U 上是没有这个问题的，因为 U 的代码逻辑有稍许不同：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f3c3706d8d724445a787d24206111ea7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=KpLDvY0il824eeSAURCI3d6lycE%3D)

虽然找到了造成异常的代码，但是对这里的逻辑还是不太理解，为什么要在这个过程中将壁纸隐藏？

最终这个问题转给了 SystemUI 的同事处理。

3 SurfaceFlinger 代码分析
---------------------

其实这个问题并不难定位原因，主要是在追查问题原因的时候，在 SurfaceFlinger 侧打 log，发现 SurfaceFlinger.setClientStateLocked 和 Layer.setAlpha 等方法都不走了，于是更深入了看了下，才发现这里的逻辑已经完全切换为新的了，含 “legacy” 相关的流程已经不走了，由于时间原因本次没办法记录的太细致，大概过一遍。

### 3.1 应用 App 侧发送的 Transaction 信息的提交流程

#### 3.1.1 SurfaceFlinger.commit

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e1b2e751f34e4a4d93a1c0b591f1f021~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=lG60lSlPD8DjrORYDaI%2BDHXO3FA%3D)

起点为 SurfaceFlinger.commit。

#### 3.1.2 SurfaceFlinger.updateLayerSnapshots

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/13cd237ea3a447e299d6e36f931a81e4~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=NuwezBMjnBRR6NLQ0zizJhePvDk%3D)

以前走的是 SurfaceFlinger.updateLayerSnapshotsLegacy，现在走 SurfaceFlinger.updateLayerSnapshots。

#### 3.1.3 LayerLifecycleManager.applyTransactions

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/da6cdb73362c472aa08d6181dc5ba6eb~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=A6G6U7EbRBeW9zyxcWFX0dJklR4%3D)

也是新玩意。

#### 3.1.4 RequestedLayerState.merge

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/11e4016a8cce465997bfe2a7c1751b54~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=UWGa93oIPNTFFLDCu46vQ%2Bi6UT0%3D)

看到是把之前 SurfaceFlinger.setClientStateLocked 的工作移到了 RequestedLayerState.merge：

1）、这里的局部变量 clientState 保存的就是从客户端发来的最新的信息。

2）、调用 layer_state_t.merge 方法，更新自身的信息：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/0b8e9e86488d43cea563c2908998cd90~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=FpdFrftEiRcprF4DQPExVGIeECc%3D)

因为 RequestedLayerState 是继承自 layer_state_t 的：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/0d015df84e234755974880b8b75d32bf~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=QshF3BW7GGt83oUFbEraMYLmAEw%3D)

因此最终从 App 端发送来的信息最终是保存到了 RequestedLayerState 中，而以前是在 SurfaceFlinger.setClientStateLocked 中将更新后的信息直接保存在了 Layer.mDrawingState 中。

### 3.2 更新 LayerSnapshot 信息

回到 SurfaceFlinger.updateLayerSnapshots：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3d3ee9f17918426e8d45376f441e25a0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=gHEtbvLfJU2IqF3xKnnI7uJGKj4%3D)

应用了从 App 侧传过来的 Transaction 的信息后，目前信息是保存在 RequestedLayerState 中的，接下来看下如何用 RequestedLayerState 的信息更新 LayerSnapshot。

#### 3.2.1 LayerSnapshotBuilder.update

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/062a9329f7b84716a750e80a9919128f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=U%2FpoczF8JeloAtZ%2BVmEjJSys5HQ%3D)

#### 3.2.2 LayerSnapshotBuilder.updateSnapshots

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f0bfd955453145e9a64cff929e0009e8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=x6sOyNYa2v%2FJuPScIJQZk0zQRd0%3D)

#### 3.2.3 LayerSnapshotBuilder.updateSnapshotsInHierarchy

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4a220974f5a34e05a25cf38ef3ec3ec9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=qDMjGq%2F7BqpCcVcMEfGkEeiPlbU%3D)

1）、调用 LayerSnapshotBuilder.updateSnapshot 继续对层级结构中的每一个 LayerSnapshot 进行更新。

2）、递归调用 LayerSnapshotBuilder.updateSnapshotsInHierarchy 自己的子节点的信息。

#### 3.2.4 LayerSnapshotBuilder.updateSnapshot

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4d5b46ca39ca438c9c19c5220fd11fd5~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=OHSD3g%2FjQBu123jBpoSsLvfhlVg%3D)

这里就是用 RequestedLayerState 来更新每一个 LayerSnapshot 的信息的地方。

比如一个 LayerSnapshot 的 alpha，是通过父节点 alpha 与本节点请求的 alpha 相乘得到的，如果父节点请求的 alpha 为 0，那么这个父节点的 alpha 就会影响到其下所有子节点的 alpha。

#### 3.3 LayerSnapshot 的可见性更新

回到 LayerSnapshotBuilder.updateSnapshots：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1de26982e7b1481aad6d4eab65cce6a6~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=mfoLgY%2FDyG9HHyPD7daWHjZ3XGQ%3D)

上一步我们已经用 RequestedLayerState 的 color.a 来更新了 LayerSnapshot.color.a 和 LayerSnapshot.alpha，接下来看下 LayerSnapshot.isVisible 是如何更新的。

#### 3.3.1 LayerSnapshotBuilder.sortSnapshotsByZ

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f3183fa6691043fe91bee4725beafa51~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=e6SPlijN4zkOA2ECwXpzO7%2FdNWI%3D)

对整个层级结构进行遍历，如果一个 LayerSnapshot 是可见的，或者有 InputInfo，那么继续调用 LayerSnapshotBuilder.updateVisibility 更新其可见性。

#### 3.3.2 LayerSnapshotBuilder.updateVisibility

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/727fa9c0259e4439b91a894657e0d367~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=oizLdXuc%2BAQFxYTrRWbMwZbVfro%3D)

这个方法更新了 LayerSnapshot.isVisible 的值。

而传参是 LayerSnapshot.getIsVisible 的返回值，看下这个方法：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c185f80e505e4da2a6b94add9c39e98c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=SEMgh3GVXmOUc6btOhVIajQbcnI%3D)

和 AndroidV 之前的逻辑也差不多。

### 3.4 计算可见区域的流程

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7184643da9244eda8b819849fb6a0307~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=i4g1AgNfM0aTmm2VWKIeYxGyUkc%3D)

先跟一下 SurfaceFlinger.moveSnapshotsToCompositionArgs，然后再分析 CompositionEngine.present。

SurfaceFlinger.moveSnapshotsToCompositionArgs 为：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/eed12111310740649d34a42b67b2b4bc~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=RJfM1UvE5ftbEBZnz45JJxCoL6U%3D)

调用 LayerSnapshotBuilder.forEachVisibleSnapshot，遍历 LayerSnapshotBuilder 中的可见 LayerSnapshot，将符合要求的 LayerFE 添加到 CompositionRefreshArgs.layers 中，记住这个 CompositionRefreshArgs.layers。

再看下 LayerSnapshotBuilder.forEachVisibleSnapshot：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/5f16db8b8cdd4dca932a0c2ce077e2c7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=WdbbaH7jc12MT0zL0CXHiRkr63Y%3D)

判断 LayerSnapshot 是否可见，用的是其成员变量 isVisible，该成员变量则是在 SurfaceFlinger.commit 流程中，通过 LayerSnapshotBuilder.updateVisibility 来更新的。

继续分析 CompositionEngine.present.

#### 3.4.1 CompositionEngine.present

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/5fff567e939044448fbe7894f7962396~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=hdWwk6wJqND8Pb6iCtd140ATiT0%3D)

#### 3.4.2 Output.prepare

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/694b85c7d9d34d16bb26f81522cd6529~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=lHIXXN9xwRSiWINkbJK7lfbqals%3D)

#### 3.4.3 Output.rebuildLayerStacks

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e9650edb46f44ce9994b46ecc5f3e7b6~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=TZyxh%2B%2FWdzOE%2BYXsgeZ1QJz8v84%3D)

#### 3.4.4 Output.collectVisibleLayers

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/dcc1c3dbdc4b44b8a1139e1595da5217~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=wHJUdqXQOzDRTVT6Te0eogt8%2Bg8%3D)

看到这里遍历 CompositionRefreshArgs.layers，对其中的每一个 LayerFE 对象都调用 Output.ensureOutputLayerIfVisible，而这里的 CompositionRefreshArgs.layers 则是在之前的 SurfaceFlinger.moveSnapshotsToCompositionArgs 中，通过遍历 LayerSnapshotBuilder 中的可见 LayerSnapshot 得到的。

#### 3.4.5 Output.ensureOutputLayerIfVisible

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/17dee615683d440f83b899d89ec2ae89~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749730464&x-signature=emGCryrTvzo0YVZTFtKWrUUWBLg%3D)

Output.ensureOutputLayerIfVisible 就是具体的计算可见区域等各种区域的地方，就不赘述了。
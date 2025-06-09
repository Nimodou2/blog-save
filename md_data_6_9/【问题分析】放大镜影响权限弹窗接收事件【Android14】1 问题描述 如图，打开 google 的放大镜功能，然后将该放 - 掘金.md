> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7408391774635638836)

> 1 问题描述 如图，打开 google 的放大镜功能，然后将该放大镜和权限弹窗部分重合，会发现权限弹窗的按钮如 “Allow”，点击无响应。

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1dd64c075ac04ef6bc4599fe8f2c6923~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=m3gW41EupSqzpziVvm1Dg9QGrZE%3D)

1 问题描述
------

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/bbd5a9f18eae45569398bd22702b2a80~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=5nReyB61XD%2FB0Eh%2FoCnQzYFoPXk%3D)

如图，打开 google 的放大镜功能，然后将该放大镜和权限弹窗部分重合，会发现权限弹窗的按钮如 “Allow”，点击无响应。

顺便一提，如果放大镜和权限弹窗完全重合或者完全不重合，是没问题的。

2 问题分析
------

### 2.1 分析 1

首先权限弹窗 View 层级结构为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d09e9e95e2e544b6a5759552ba8e673e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=MJcB8PBGFOZKyOjDTPJrHzYXzbI%3D)

对应的按钮为 “SecureButton”。

打开一些 log 开关，首先是正常的 log：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1af1be31053d4dd5a0bdf041db208128~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=3Pp5oBMrrT0FLAh75SptFYNJQN8%3D)

MotionEvent 传到 “SecureButton” 并且也被这个 Buttion 处理了。

再看异常 log 为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c76806e6a45347cab60ba2a4a8613f59~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=giYMCY5udueyHdYT%2FkWLYlQr8rQ%3D)

这里的很多 log 信息都是 MTK 的 log 打印的，能知道的信息是，MotionEvent 已经传到 “SecureButton” 了，但是 “SecureButton” 没有处理，最终处理 MotionEvent 的是一个父 View，LinearLayout。

### 2.2 分析 2

既然事件已经传到了 SecureButton 了，那么再看具体的 View 类处理 MotionEvent 的代码，View.dispatchTouchEvent：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6f80f2f2a1d44e94afd6bcd8103811e6~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=ZsYZmBnwXc3QyY5NAnuvexpljC0%3D)

很值得怀疑的一个点就是 View.onFilterTouchEventForSecurity 是不是拦截了 MotionEvent 发给 onTouch 以及 onTouchEvent 来处理事件，打个断点看看：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/295a9e490ce84406ab32343083283837~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=i2yfOhU6TI6Tp%2BAkm6eEjkMWLx8%3D)

果然，这里的 View.onFilterTouchEventForSecurity 走进了 “SecureButton” 自己重写的 onFilterTouchEventForSecurity 方法中，并且返回了 false，导致 MotionEvent 没有被 “SecureButton” 处理。

反编译 apk 看到 “SecureButton” 重写的 onFilterTouchEventForSecurity 方法为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b5ed397473284b769678a057b28464f2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=DLMntNSFwu0ISy5nd9nTKzBpcQU%3D)

看下这里的 MotionEvent 的 flag：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/eba95011a78747b9befc6071785e0ff9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=%2Fx9XqFtTcglD07wUbRK0udk363Q%3D)

FLAG_WINDOW_IS_OBSCURED 和 FLAG_WINDOW_IS_PARTIALLY_OBSCURED 都表示接收 MotionEvent 的窗口被另外一个位于它之上的可见窗口遮挡了，但是不同点的是：

1）、FLAG_WINDOW_IS_OBSCURED 表示 MotionEvent 落在了被遮挡的区域。

2）、FLAG_WINDOW_IS_PARTIALLY_OBSCURED 表示 MotionEvent 落在了被遮挡区域以外的区域，也就是没被遮挡的区域。

那么回头再看看异常的 log，果然：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/9584be9490a84d739a4fc44f113cdb6a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=o%2BUMEooYYYdm2AWc3EAJWzet0I0%3D)

这里的 MotionEvent 的 flag 为 0x2，那么也就是包含了 FLAG_WINDOW_IS_PARTIALLY_OBSCURED 这个 flag，所以传入 “SecureButton” 自己重写的 onFilterTouchEventForSecurity 方法后会返回 false。

### 2.3 分析 3

最后看下 FLAG_WINDOW_IS_PARTIALLY_OBSCURED 这个 flag 是在哪里添加的。

在 InputDispatcher.findTouchedWindowTargetsLocked：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/81cd763513e44950b8073047556d1047~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=Fb0YVtp20Bi9tZOb7T0mKEKxSJ8%3D)

如果 InputDispatcher.isWindowObscuredLocked 返回 true，那么就表示找个窗口被遮挡，就要为找个窗口对应的 WindowInfo 添加 WINDOW_IS_PARTIALLY_OBSCURED 标志位。

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1f0d2ebe686743a1b486217b704b3581~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=5iMi6tKsEZRauk8qGCMnZ3Dnwso%3D)

这里的逻辑也比较简单，从上到下找能够遮挡当前 Window 的 WindowInfoHandle，找到当前窗口的时候就结束。

根据我添加的 log，看到遮挡的 WindowInfoHandle 为请求权限弹窗的那个界面窗口克隆出的窗口，被遮挡的自然就是权限弹窗：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/bab34eb67d0b41d780c5c63a75cac725~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=08X40EDTQp0IGNTRQIc%2B3ArY5UY%3D)

符合我们 dumpsys input 的信息：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d1b44a1885484bba81d4363a4117b1d2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=JkkYoyLZM94i9QSnl3zZSMUbwYA%3D)

input 大概示意图为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/dbae10995d254595b447057e65e69188~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=rSyQY4kUSEAYrUuuzV38CHAVNOQ%3D)

1）、开启放大镜后，会为每一个 Layer 克隆一个 Layer 出来，并且这些克隆体的层级整体都比真身高。

2）、根据 InputDispatcher.canBeObscuredBy 的逻辑，如果 WindowInfo 包含以下信息，则不遮挡：

*   NOT_VISIBLE 的窗口不遮挡。

*   NOT_TOUCHABLE 的窗口不遮挡。

*   TRUSTED_OVERLAY 的窗口不遮挡。

*   相同 uid 的窗口不遮挡。

*   相同 token 的窗口不遮挡。

*   不同 displayId 的窗口不遮挡。

排除以上条件的 WindowInfoHandle 后，第一个遮挡的窗口就是申请权限弹窗的那个界面的窗口的克隆窗口。

但是 pixel 没有问题，查看信息后发现，pixel 似乎对每一个克隆出来的 InputWindowHandle 都添加了 TRUSTED_OVERLAY 这个 flag：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7aebd61753324f6cbb7776b9eaabf530~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=MHLqces0m%2BP4kgU%2BJ71QEQHcjIw%3D)

下一步需要继续查找这个差异。

### 2.4 分析 4

搜索代码，查看为 WindowInfo 添加 TRUSTED_OVERLAY 的位置主要是两处：

1）、一个是旧的流程：在 Layer.fillInputInfo 中：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3f2beef368e84a8da40c62ede4c0882d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=jDg5zqd32pzP0kU2hSyG9hmOeEk%3D)

2）、一个是新的流程：在 LayerSnapshotBuilder.updateInput 中：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4c61a9eabde148c9ae8663326241a3cf~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=Daf%2FZxoOjwDSS7KMTjkK1o68yp4%3D)

具体走哪个流程，则是和 SurfaceFlinger.commit 的以下逻辑有关，受 SurfaceFlinger.mLayerLifecycleManagerEnabled 的控制：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c0cf9dfe25e74e8d85440e182d1f449b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=FEg8GihrRk9%2BH0%2FLKF%2BaMgbw%2Fo4%3D)

如果 SurfaceFlinger.mLayerLifecycleManagerEnabled 为 true，那么走新流程，否则走旧流程。

从目前的信息来看：

1）、我们的 Android14 的机器走的是 SurfaceFlinger 的旧流程，有问题。

2）、Android14 的 pixel 走的是 SurfaceFlinger 的新流程，没问题。

3）、我们的 Android15 的机器走的是 SurfaceFlinger 的新流程，没问题。

根据是 SurfaceFlinger.dumpAll 中，可以输出 SurfaceFlinger.mLayerLifecycleManagerEnabled 的值：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/68aa84075196438aae17fffcbd4ed7d9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=4T%2FtrCWJNn1naX6VRChph%2FgDvBo%3D)

并且 Android14 的 pixel 的 SF 是有这个信息的：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1ce3cf9927a9484b939b9474a52cff2b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=0gc%2BVsNpt3JSuXCr95%2F1NlfdrzQ%3D)

而我们的机器则没有。

因此这个问题应该是原生问题。

### 2.5 小延伸一下

现在已经知道了权限弹窗没有办法响应 MotionEvent 是因为，它被请求权限弹窗的那个 Activity 的窗口的克隆窗口盖住了，即我们刚刚的示意图：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/68445700aa3c45dba210d75e5dfc9bc8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=EedBvMBoNUnZVd7A%2FNE2KjD9d50%3D)

另外如我们最开始提到的，如果放大镜和权限弹窗完全重合或者完全不重合，是没问题的。

完全不重合没问题，这个很好理解，既然不重合了，那就不存在遮挡的情况了，那完全重合为啥也没问题呢？

打印了 log 后发现其实原理很简单，完全重合后，点击的区域就是权限弹窗的克隆窗口，这个克隆窗口同样也能接收事件，因此事件直接被权限弹窗的克隆窗口接收了：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ff87fe917e8f4d9484798bed93cfe81a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749767907&x-signature=H1NZVBb8yra3r6vt7hqQcciNKg8%3D)
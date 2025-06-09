> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7408391774635032628)

> 一般来说，SF 侧的 Layer 层级和 WMS 侧 WindowContainer 侧的层级是一一对应的，但是对 Launcher 来说，则略有不同，这点之前我在打印 SF 信息的时候，也有注意过，但是没有去仔细思考过为

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e790fe723fda457dbc1e9c067a14ab3b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749731237&x-signature=5FMLVtWcqtWI4RbpGnQq%2F10eu9o%3D)

一般来说，SF 侧的 Layer 层级和 WMS 侧 WindowContainer 侧的层级是一一对应的，但是对 Launcher 来说，则略有不同，这点之前我在打印 SF 信息的时候，也有注意过，但是没有去仔细思考过为什么会这样，直到这次分析问题的时候踩了一坑，才发现有必要梳理一下这块逻辑，并做个记录。

1 问题描述
------

进入超级省电模式（也是一个 Launcher），然后随便打开一个 App，如 Message，然后在 Message 界面上划，发现无法返回到 Home。

2 问题分析
------

### 2.1 分析 1

最初我分析的方向是错误的，刚拿到这个问题的时候，我复现了一下，先看上层 WMS 处 WindowContainer 的信息：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/35c0d77af00845a9bf51d19545d36e9b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749731237&x-signature=KNuemeF7mXv5AT%2FjIq%2F0RdTN%2FNg%3D)

没问题，Home 类型的 Task 已经在 TaskDisplayArea 的 top 了。

再看 SF：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f703b0b7860a46c59642d2164156a05c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749731237&x-signature=0iZOd%2BFtz8c0vZbvwY9sw%2BLz1TI%3D)

Message 仍然是可见的。

因此我就直接认为是 SF 侧没有把 Launcher 对应的 Task 移动到 TaskDisplayArea 的 top，比如 Transaction.setLayer 这个方法没有被调用之类的，但是继续打印 log 后才发现，正常情况下也没有为 Home 类型的 Task 设置 layer 的操作，这个分析方向是错的。

### 2.2 对 Home 类型 Task 的层级的特殊处理

正常情况下，当我从任意一个 App 回到 Home 后，再看此时 winscope 的信息：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1ca7ece1b8494d1c86b27aa9bcf353c3~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749731237&x-signature=d88aeKX9XJU9bzqGEqja3u1J4P0%3D)

发现虽然 Launcher 是可见的，但是它在 SF 侧仍然是出于 TaskDisplayArea 的 bottom 的，而在 WMS 侧，它在 WindowContainer 层级结构中是处于 TaskDisplayArea 的 top 的。

画个示意图，按照层级高的在上的形式。

WMS 侧的情况为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/32397119b4634577ad76af76f8aaff58~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749731237&x-signature=N0HKDbVuBQAvvUC4oYr93s%2FGxA4%3D)

SF 侧的情况为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/14376067a6c34a0a9635c0ccac54fcb7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749731237&x-signature=oSOfYgfS22FL095MfAevyEtr%2FG8%3D)

WMS 侧的情况是符合直觉的，但是 SF 侧的确是 Launcher 的 Task 在底部，为什么会这样呢？

直接看 TaskDisplayArea 调整 Task 的层级地方，在 TaskDisplayArea.assignRootTaskOrdering：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/62c06b772be8455e8828ec038d3a9b53~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749731237&x-signature=GeHEwzncb09iTsN2ZihDTj%2FIYAg%3D)

其实逻辑很清楚了，首先定义一个局部变量 layer，初始化为 0，然后每次都是先调用 TaskDisplayArea.adjustRootTaskLayer 来设置 Home 类型的 Task 的层级，所以 Home 类型的 Task 在 SF 侧 TaskDisplayArea 中就是一直处于 bottom 的。

最终 TaskDisplayArea.adjustRootTaskLayer 会调用 WindowContainer.assignLayer，这里会调用 Transaction.setLayer 来完成最终的 Layer 设置，调用堆栈为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/790a4253b5b74d72993c559815e411f2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749731237&x-signature=poGUb3pJVGpltf8Neva4uSNxel0%3D)

可以看到调用的时机是在动画就绪的时候，Transition.onTransactionReady。

最后说一下，想要让 Home 类型的 Task 能够在屏幕上被看见，那么就只能在适当的时机隐藏位于 Home 类型 Task 之上的其它 Task，如果这两个 Task 都是可见的，那么普通的 App Task 遮挡住 Home 类型的 Task。

当前问题就是这个原因，继续分析。

### 2.3 分析 2

经过以上分析，可知我们分析的重点不是在于 SurfaceControl 的层级设置，而是在于 SurfaceControl 的可见性（show 和 hide）设置。

先来回顾一下 Transition 中和可见性相关的重要节点，以从 Message 回到 Launcher 为例：

1）、启动 Launcher，Launcher 相关 ActivityRecord 变为可见。

2）、在动画就绪，Transition.onTransactionReady 的时候，需要将 Launcher 的相关 Layer 设置为可见。

3）、然后播放动画，此时参与动画的 Message 和 Launcher 的相关 Layer 都应该是可见的。

4）、动画结束，Message 相关的 ActivityRecord 变为不可见，那么它的 Task 也会被认为是不可见，进而调用 Transaction.hide 来隐藏它的 SurfaceControl，堆栈为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/51efeed9a3cc477d8c13b842ff99e2f7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749731237&x-signature=1FTqRVrjtM7WrkGz1X74CoqhhHw%3D)

因此我们应该关注动画结束后，Message 对应的 Task 没有去隐藏。

之后就定位到了问题原因，发现整个过程中没有 “Finish transition” 相关的 log 打印，说明动画流程没有走完，那么自然也不会将 Message 对应的 Task 隐藏。

再结合 Launcher 那边说它们是通过 startRecentsTransition 等接口来启动相关 Home Activity 的，因此很大概率是本次启动是瞬态启动。

瞬态启动一个重要的特点就是，从一个界面进入 Recents，并且离开 Recents 后，一个完成的 Transition 才算完成，如果只是进入 Recents，那么 Transition 只走到了 Transition.onTransactionReady，只有从 Recents 界面离开（选择 Recents 界面的一个应用进入，或者点击 Recents 界面的空白区域回到 Home），动画才会开始播放，并且最终走到 finishTransition 阶段，也就是说需要 Launcher 那边的动画开始播放并且播放完成后 Transition 才会结束，因此需要 Launcher 那边继续排查。
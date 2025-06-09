> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7371717387829608489)

> 说来惭愧，AndroidU 都已经开发这么久了，但是我还没有整理过 ShellTransitions 相关的知识。我本来希望能够系统的写一篇关于 ShellTransitions 的笔记出来，但是发现一来这是一

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7fc5ad235dea475eaebbd6cc6f7823e2~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1920&h=1080&s=2593169&e=jpg&b=242212)

说来惭愧，AndroidU 都已经开发这么久了，但是我还没有整理过 ShellTransitions 相关的知识。我本来希望能够系统的写一篇关于 ShellTransitions 的笔记出来，但是发现一来这是一个比较庞大的模块，二来我个人能力有限，对 ShellTransitions 目前掌握的并不透彻，只是朦胧上大体有一个认知。如果把 ShellTransitions 的方方面面都了解清楚再去写这么一篇笔记，google 说不定已经不用 ShellTransitions 了，就像我刚刚弄清楚 AppTransition 和 AppTransitionController 这块的逻辑，AndroidU 就把动画逻辑变成 ShellTransitions 了一样，有点不讲武德了。所以趁 ShellTransitions 还健在，我觉得还是趁热打铁吧，哪天看了 ShellTransitions 的代码有点感悟了就去做相关记录，这样做的缺点就是知识点比较零散，写的会比较随意，并且也可能有理解的不到位的地方，但是希望可以靠后面的持续更新来慢慢改善这些不足，总比 ShellTransitions 坟头草都三米高了才想起来要总结要好。

废话就到这里，现在开始吧。

1 ShellTransition 相关类
---------------------

网上关于 ShellTransition 的介绍还是比较少的，这个时候想要立刻能够对 ShellTransition 有一个大概认知，我一般会看注释，所以这里也是先看 ShellTransition 相关的类的注释。

### 1.1 TransitionController

首先是 TransitionController，它有点像之前的 AppTransitionController，是 WMCore 这边的过渡动画的控制器调度器之类的，控制过渡动画的生命周期。

另外一提，这里有一个 WMCore 和 WMShell 的新概念，似乎是和这些的类的包名有关，比如 TransitionController 这个类，它的目录为：

frameworks\base\services\core\java\com\android\server\wm\TransitionController.java

包名为：com.android.server.wm

它被归纳为 WMCore 里的。

而下面的 Transitions 类，它的目录为：

frameworks\base\libs\WindowManager\Shell\src\com\android\wm\shell\transition\Transitions.java

包名为：com.android.wm.shell.transition

它被归纳为 WMShell 里的。

好了，继续看 TransitionController 的注释：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/aba4ea2c9fbf44adb7cc947513066005~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749705610&x-signature=SHc5TLwI02%2B%2FcrZPQXNtVSSeN3E%3D)

处理记录（收集）和同步过渡动画的所有方面。 这仅涉及 WM 更改。 实际的动画由 Player 处理。

目前，一次只能有 1 个过渡动画成为主要 “收集器”。 这是因为 WM 更改仍然以“全局” 方式执行。 然而，收集实际上可以分为两个阶段：

    1. 实际进行 WM 更改并记录参与的容器。

    2. 等待参与容器准备就绪（例如重绘内容）。 

因为 (2) 花费了大部分时间并且不会改变 WM，所以我们实际上可以在阶段 (2) 中同时进行多个 Transition，同时在阶段 (1) 中进行一个 Transition。 我们将这种安排称为 “并行” 收集，尽管实际上仍然只有 1 个 Transition 能够获得参与者。

当 “主收集器” 完成 “设置”（第 1 阶段）并等待时，会发生并行收集。 此时，另一个过渡动画可以开始收集。 发生这种情况时，第一个 Transition 将移至“等待” 列表，新过渡动画将成为 “主收集器”。 如果在任何时候，“主收集器” 在其中一个等待过渡动画之前开始播放，则第一个等待过渡动画将变回 “主收集器”。 这维持了 WM 其余部分当前所期望的“全局” 抽象。 

当一个过渡动画移动到播放时，我们会根据所有其他播放过渡动画进行检查。 如果它不与它们重叠，它也可以并行动画。 在这种情况下，它将被分配一个新的 “轨道”。 “轨道” 是一种与 Player 沟通哪些过渡动画需要彼此串行播放的方式。 因此，如果我们发现一个过渡动画与一个轨道中的其他过渡动画重叠，则该过渡动画将被分配给该轨道。 但是，如果过渡动画与 大于 1 个轨道中的过渡动画重叠，我们实际上会将其标记为 SYNC，这意味着在所有先前的过渡动画完成之前它无法实际播放。 这是一种笨拙的做法，因为这是一种后备情况，支持更高级的东西会变得不必要的复杂。

### 1.2 Transitions

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a020c0a84fe4402eac2c77d9d5ccb7b0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749705610&x-signature=sAqkdMIvA3CfjxMCu%2FAHIij0eYY%3D)

播放过渡动画。 在这个 Players 中，每个过渡动画都有一个生命周期。

1.  当直接启动或请求过渡动画时，将其添加到 “PENDING” 状态。 
    
2.  一旦 WMCore 应用过渡动画并发出通知，过渡动画就会转至 “READY” 状态。 
    
3.  当过渡动画开始动画时，它会转至 “ACTIVE” 状态。 
    

基本上：--start--> PENDING --onTransitionReady--> READY --play--> ACTIVE --finish                                                                                                                                --merge--> MERGED

READY 及以后的生命周期按 “轨道” 进行管理。 在一个轨道中，所有动画都按描述的方式串行排列； 但是，多个轨道可以同时播放。 这意味着，在一个轨道内，一次只能有一个过渡动画处于动画状态（“ACTIVE ”）。 

当一个过渡动画在一个轨道中进行动画时，分派到该轨道的过渡动画将以 “READY” 状态排队等待轮到。 同时，每当过渡动画到达 “READY ” 队列的头部时，它将尝试与 “ACTIVE ” 状态的过渡动画合并。 如果合并成功，它将被移动到 “ACTIVE ” 状态的过渡动画的 “合并” 列表，然后下一个 “READY ” 状态的过渡动画可以尝试合并。 一旦 “ACTIVE ” 状态的过渡动画完成，就可以播放下一个 “READY ” 状态的过渡动画。 

轨道分配预计由 WMCore 提供，这通常会尝试保持相同的分配。 但是，如果 WMCore 确定过渡动画与大于 1 个活动轨道冲突，则会将其标记为 SYNC。 这意味着在 SYNC 过渡动画可以播放之前，必须刷新所有当前活动的轨道。

### 1.3 过渡动画

由于 WMCore 侧的 TransitionController 和 WMShell 侧的 Transitions 职责有所不同：

1、WMCore 侧，对应系统进程，管控过渡动画的整个生命周期。

2、WMShell 侧，目前对应 SystemUI 进程，主要是处理过渡动画的播放。

因为过渡动画从创建到结束整体下来是一套复杂的流程，期间 WMCore 和 WMShell 会有多次通信，那么他们各自都得有一个类来代表过渡动画，即 WMCore 侧的 Transition 和 WMShell 侧的 ActiveTransition。

#### 1.3.1 Transition

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6f5cb38d5b23484c81c505527d940991~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749705610&x-signature=XK%2BhugcO5IkCoiv3xNkMcZ6oaDE%3D)

Transition 是过渡动画在 WMCore 侧的代表，其内部定义了 Transition 可能处于的几个状态值，其成员变量 mState 保存了 Transition 当前所处的状态，默认是 STATE_PENDING。

#### 1.3.2 ActiveTransition 类

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f3294787e4eb4de8be18aa5e187ddd97~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749705610&x-signature=NNw16sKP%2F1dTddCIm6Pbe5dEePE%3D)

ActiveTransition 是过渡动画在 WMShell 侧的代表，ActiveTransition 类定义在 Transitions 内部，通过其 IBinder 类型的成员变量 mToken 与 WMCore 侧的 Transition 一一对应。

2 过渡动画整体流程
----------

看了注释其实还是挺懵 b 的，这里再用我个人的理解介绍一下过渡动画的状态流转。

#### 2.1 Transition 状态流转

首先 Transition 是有一个状态集的，刚刚介绍 Transition 类的时候也看到了，定义在 Transition 类内部：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4e20f5c4abce406bbd907ea4f39207fa~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749705610&x-signature=DlPrP6Q5YSlX752NSVqqXDOFDkg%3D)

1）、STATE_PENDING，当 Transition 刚被创建的时候就是这个状态。

典型的是在各种需要动画的流程下调用 TransitionController.createTransition 来创建一个 Transition 对象。

2）、STATE_COLLECTING，当 Transition 开始收集动画参与者的时候，就会被置为这个状态。

一般创建完 Transition 后就会调用 TransitionController.moveToCollecting，最终在 Transition.startCollecting 中，Transition 从 STATE_PENDING 状态切换到 STATE_COLLECTING 状态。

3）、STATE_STARTED，Transition 已经被正式启动，它仍在收集中，但一旦所有参与者准备好进行动画（完成绘制），它将停止。结合 STATE_COLLECTING，这里的意思应该就是说，如果 Transition 没有被 start，那么它将一直处于收集参与者的状态，即使所有参与者都已经完成绘制可以开始动画了，但是因为当前 Transition 没有被启动，所以也无法进行下一步。

首先 WMCore 侧会在合适的时机调用 TransitionController.requestStartTransition 切换到 WMShell，当 WMShell 侧也启动了一个 ActiveTransition 后，会调用 WindowOrganizer.startTransition 切换回 WMCore 侧，最终调用 Transition.start，在 Transition.start 中 Transition 从 STATE_COLLECTING 状态切换到 STATE_STARTED 状态。

4）、STATE_PLAYING，Transition 正在播放动画，并且不能再继续收集了，也不能被改变。也就是说此时谁谁谁要播放动画已经确定了，并且这些参与者已经开始播放动画了，所以就不能再去收集新的参与者了，而且也不能对当前 Transition 进行修改了。

当所有动画参与者都已经绘制完成，可以开始动画了，那么 WMCore 侧就会走到 Transition.onTransactionReady，走到这一步就认为 Transition 已经就绪了，这里 Transition 将会从 STATE_STARTED 状态切换到 STATE_PLAYING 状态，然后调用 ITransitionPlayer.onTransitionReady 切换到 WMShell。

5）、STATE_FINISHED，Transition 已经成功播放完了动画。

WMShell 侧主要负责播放动画，当 WMShell 侧播放动画完成后，会调用 WindowOrganizer.finishTransition 切换回 WMCore 侧，最终在 Transition.finishTransition 中，Transition 从 STATE_PLAYING 状态切换到了 STATE_FINISHED 状态。

6）、STATE_ABORTED，Transition 正在被中止或者已经中止，不会播放任何动画，也不会将任何内容发给 player。

当调用 Transition.abort 方法，Transition 的状态会被置为 STATE_ABORTED，这个动作可能发生在任意时刻。

此处也许需要一个流程图：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/508ea13564bb4370826a960cf2d72752~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749705610&x-signature=SEZOSXxvnQ1%2FJZ0iRAF4Srl97Qg%3D)

#### 2.2 ActiveTransition 状态流转

如之前看到的，ActiveTransition 的状态流转定义在 Transitions 的注释中：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/463d27f2bac845c4b04fd68f5e7d3e67~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749705610&x-signature=DJz4CuxfWsQ7L%2FYmekt2Z7bMyq8%3D)

~看到其实代码中是没有定义注释中说的那些 READY、MERGED 的状态的。~

**刚开始写这篇文章的时候其实还没有仔细把过渡动画的整体流程过一遍，经过评论区 (这篇文章 csdn 也有发) 的指正，发现自己对注释中提到的各个状态值理解的并不准确，以现在的知识储备重新去阅读注释，有了一些新的理解，这里重新介绍如下：**

之前我是把 Transition 的状态和 ActiveTransition 的状态混着说了，这里的 READY 和 MERGED 等状态描述的是ＷＭShell 侧 ActiveTransition 的状态，并非是 WMCore 侧 Transition 的状态，最直接的证据就是这个注释写在 Transitions 中而不是 TransitionController 中，当然接下来也会根据 WMShell 侧的代码来介绍一下。

再者虽然 WMShell 侧没有像 WMCore 侧那样，将 READY 和 MERGED 等状态都定义为一个个的状态值，但是这里也的确是有这个状态的概念的，这里再根据代码理解一下这个注释提到的 ActiveTransition 的状态流转。

另外上面的注释：

小写代表一个瞬间的动作 / 操作，这个操作改变了过渡动画的状态。

大写代表代表了过渡动画在某一段时间内所处的状态，与瞬间的操作相对。

1）、start，这对应一个启动过渡动画的操作，这个操作可以由 WMCore 侧在 TransitionController.requestStartTransition 发起，也可以由 WMShell 侧直接通过 Transitions.startTransition 发起。

这一步的最终结果是在 WMShell 侧创建了一个 ActiveTransition 对象，并且保存在了 Transitions 的 mPendingTransitions 成员变量中，该成员变量保存的是已经被启动但是还没有就绪的过渡动画，因此可以认为此时 ActiveTransition 进入了 PENDING 状态。

2）、onTransitionReady，这对应了过渡动画就绪的这个时间点，由 WMCore 侧在 Transition.onTransactionReady 中发起。当 WMCore 侧的 Transition 就绪，就会来到 Transition.onTransactionReady 方法，然后通过 ITransitionPlayer.onTransitionReady 来通知 WMShell 过渡动画已经就绪可以行下一步了。

这一步的最终结果是，在 Transitions.dispatchReady 中创建了一个 Track 对象， 并且把当前 WMCore 侧就绪的 Transition 对应的那个 ActiveTransition 对象保存到了 Track.mReadyTransitions 中，该成员变量保存的是已经就绪正在排队等待播放的过渡动画，因此可以认为此时 ActiveTransition 从 PENDING 状态切换到了 READY 状态。

3）、接下来根据当前 READY 的 ActiveTransition 之前是否已经有正在播放的 ActiveTransition，分为两种情况，如注释这里的表示：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c593014b8784415ba19d57ca17e2e90e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749705610&x-signature=gtRoQFoR%2FMHNDK3XHEU56BEtWXY%3D)

    3.1）、play，这对应之前没有正在播放的 ActiveTransition（或者说没有处于 ACTIVE 状态的 ActiveTransition）的情况，那么当前就绪的 ActiveTransition 就要开始播放动画了。在上一步，在 Transitions.dispatchReady 中 ActiveTransition 从 PENDING 状态切换到了 READY 状态，然后 Transitions.dispatchReady 紧接着调用了 Transitions.processReadyQueue。

    这一步的最终结果是在 Transitions.processReadyQueue 中，就绪的那个 ActiveTransition 从 Track 的 mReadyTransitions 移动到了 Track 的 mActiveTransition，因此可以认为此时 ActiveTransition 从 READY 切换到了 ACTIVE 状态。接下来就是调用 TransitionHandler.startAnimation 来播放动画了。

    3.2）、merge，这对应之前有正在播放的 Activetransition 的情况，那么这里会调用 TransitionHandler.mergeAnimation。

    这一步的最终结果是 READY 状态的过渡动画会和 ACTIVE 状态的过渡动画进行合并，即当前就绪的 ActiveTransition 从 READY 切换到了 MERGED 状态。

4）、最后不管是 ACTIVE 状态的过渡动画，还是 MERGED 状态的过渡动画，等到它们的动画播放完成后，都会回调 Transition.onFinish，最终调用 WindowOrganizer.finishTransition 切换回 WMCore，走最终的过渡动画结束的 finishTransition 流程。

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/191787271d544e328a08a13cd5389cf0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749705610&x-signature=2vEnRITtxEdj5Q%2BCEmXEGeEfB3M%3D)

直到此时我才知道这里的 “^” 是一个箭头，代表的是从过渡动画从 “MERGED” 节点转到下一步的 “finish” 节点...... 有点抽象了。

流程图：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/01f5c148b6714a2792158a7d08635499~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749705610&x-signature=VPZBcqV3JtyGPRaR636VaOoLJZ8%3D)

#### 2.3 总流程图

最后是 WMCore 和 WMShell 交互的流程图：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/8b7e1bc5563b4a7ca82093c12f0b6e8e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749705610&x-signature=Yt38v%2FOzEKMkref9Qh3L5fqqlbo%3D)

接下来我们以在 Launcher 界面点击 App 图标启动某个 App 为例，来分析一般过程，但是无法面面俱到，一些细节希望靠后续的系列文章可以补充。
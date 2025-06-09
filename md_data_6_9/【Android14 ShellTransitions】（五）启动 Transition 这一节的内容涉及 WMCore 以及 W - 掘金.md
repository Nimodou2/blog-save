> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7371717387829870633)

> 这一节的内容涉及 WMCore 以及 WMShell，主要是启动 Transition。 回到 ActivityStarter.startActivityUnchecked 方法： 看下最后启动 Transitio

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/135e9e23d20c43b48ac6737e74800a76~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1920&h=1080&s=1473068&e=jpg&b=2e2e2e)

这一节的内容涉及 WMCore 以及 WMShell，主要是启动 Transition。

回到 ActivityStarter.startActivityUnchecked 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dcf16510b20b447184e4b20e3e4d9eda~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=817&h=662&s=66976&e=png&b=fefdfd)

看下最后启动 Transition 的部分，在 ActivityStarter.handleStartResult 中：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/116eb94d770240a69d139b86531046ce~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=838&h=583&s=60660&e=png&b=fefefe)

只关注我们要关注的部分。

首先是如果这里的局部变量 isStarted 为 true，就说明这个 ActivityRecord 是新建的，因此调用 TransitionController.collectExistenceChange 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2317f9b0f3d8472799e6e9d5aca5575a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=832&h=409&s=51610&e=png&b=fefdfd)

将从正在收集的这个 Transition 的 mChanges 中找到这个新建的 ActivityRecord 的 ChangeInfo 对象，将其成员变量 mExistenceChanged 置为 true，表示这个 WindowContainer 在本次改变前后的存在性发生了变化，即从无到有（OPEN），或者从有到无（CLOSE）。

再回到 ActivityStarter.handleStartResult，根据我们的分析流程，这里传入的 newTransition 不为空，所以所以这里继续调用的是 TransitionController.requestStartTransition 方法。

1 WMCore 部分 —— TransitionController.requestStartTransition
----------------------------------------------------------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50f11839e13a4af08b07335d8f4e016b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=800&h=455&s=54093&e=png&b=fffefe)

从注释可知，该方法用来向 Player（WMShell）端创建一个 Transition，但是不要启动。

该方法的内容主要为：

1）、创建一个 TransitionRequestInfo 对象：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8661bfbc406249f3b18b9e175cea3052~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=839&h=494&s=54686&e=png&b=fefdfd)

TransitionRequestInfo 是继承了 Parcelable 的类，用来将 WMCore 端的一些信息进行封装，发送给 WMShell 端。

比如这里的 triggerTask，如果不为空，就包含了那个生命周期变化（start 或者 finish）导致本次 Transition 发生的那个 Activity 对应的 Task 的信息。

2）、调用成员变量 mTransitionPlayer 的 requestStartTransition 方法，先看这个成员变量 mTransitionPlayer：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f4c86231b02441783ba1234ae9ba5ad~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=424&h=56&s=8049&e=png&b=2b2b2b)

类型为 ITransitionPlayer，它的定义为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aff334bac72946b0bc550e4f45be4cd0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=831&h=714&s=89658&e=png)

大概翻译为：

该接口由 WMShell 实现，用于启动和播放过渡动画。和 IWindowOrganizerController 一起使用的流程如下：

1.  WMCore 启动一个 Activity 并调用 requestStartTransition 方法。

2.  这个 TransitionPlayer 的实现执行相应操作，然后调用 IWindowOrganizerController 的 startTransition 方法，告诉 WMCore 正式开始（在此之前，WMCore 将收集 Transition 中的所有更改，但不会认为 Transition 已准备好进行动画）。

3.  一旦 Transition 中所有收集的更改都完成绘制，WMCore 将调用 onTransitionReady 方法，在这里委托实际的动画。

4.  一旦这个 TransitionPlayer 的实现完成动画，它通过 IWindowOrganizerController 的 finishTransition 方法通知 WMCore。此时，ITransitionPlayer 的责任结束。

ITransitionPlayer 接口中包含两个方法：

*   onTransitionReady: 当 Transition 的所有参与者准备好进行动画时调用。作为对 startTransition 方法的响应。

*   requestStartTransition: 当 WMCore 中的某些内容需要播放 Transition 时调用，例如在新 Task 中启动 Activity 时。

该注释已经解释的很清楚了，现在我们已经在 WMCore 端封装好需要传递的信息了，下一步就是调用 ITransitionPlayer.requestStartTransition 将这些信息发送到 WMShell 端，来通知那边创建一个 Transition。

2 WMShell 部分 —— Transitions.requestStartTransition
--------------------------------------------------

从上面 ITransitionPlayer 的注释我们知道了 ITransitionPlayer 的实现端在 WMShell，具体则是定义在 Transitions 中的 TransitionPlayerImpl 类：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b9d2a4238840442fb888ce27ac59ecad~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=834&h=265&s=24806&e=png)

它实现了 ITransitionPlayer 中定义的两个方法接口，我们先分析 requestStartTransition，后面等遇到的时候再分析 onTransitionReady。

Transitions.requestStartTransition 的内容为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d90a1d85cf7462e9940c3aee27a68d0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=812&h=770&s=92034&e=png)

1）、创建一个 ActiveTransition 对象：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d8f09714ba0448dbba4ae24c08e08287~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=668&h=215&s=15651&e=png)

ActiveTransition 中有一个 IBinder 类型的 mToken 成员，用来保存从 WMCore 中传来的 Transition 的 token，那么可以认为 WMCore 端的 Transition 和 WMShell 端的 ActiveTransition 是一个一一对应的关系，这样也有助于理解 TransitionController.requestStartTransition 的注释：

”Asks the transition player (shell) to start a created but not yet started transition.“

这里要 player 创建的 transition 并非 Transition，而是 ActiveTransition。

ActiveTransition 中的其它成员变量也非常重要，不过现在我不打算全部说明，用到的时候再说吧。

2）、调用 Transitions.mHandlers 中的每一个 TransitionHandler 的 handleRequest 方法，来为这个刚刚创建的 ActiveTransition 找到一个后续走到 Transitions.onTransitionReady 时，执行动画的 Transitions.TransitionHandler 对象，并且添加到 Transitions.mPendingTransitions 中。后续走到 Transitions.onTransitionReady 的时候，再从 Transitions.mPendingTransitions 中拿需要执行动画的 TransitionHandler 对象。

Transitions 的 mHandlers 成员变量是一个 TransitionHandler 类型的队列：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48abfc7e1c0e4da2ba1465d3d70608a8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=792&h=87&s=18165&e=png&b=2b2b2b)

TransitionHandler 表示可以处理一组过渡动画的接口：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/36eb7344b7fb4bd2b19e9f718a866324~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=825&h=629&s=69322&e=png&b=fefdfd)

该接口定义了以下方法：

*   startAnimation: 开始一个过渡动画。如果 handleRequest 方法返回非空值，则始终调用此方法来处理特定 Transition。否则，仅当之前没有其他 handler 处理 Transition 时才调用此方法。

*   handleRequest: 可能处理 startTransition 请求。

*   等等......

开机 SystemUI 进程起来后，就会向 Transitions.mHandlers 中添加多个 TransitionController 的子类：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89e56bac2edc4414be29f1ee28d10546~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=819&h=424&s=112526&e=png)

从名字上也能看出这些 TransitionController 的子类对应的不同类型的 Transition 的动画。

3）、当创建了 ActiveTransition 后，下一步就是决定哪一个 TransitionHandler 来处理当前 Transition，具体说来则是遍历 Transitions.mHandlers，按照顺序对每一个 TransitionHandler 调用 handleRequest 方法，看哪一个子类可以处理这个 Transition，比如我这里打印的 log：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b17687ae4d9e481196d387e121ae1d0b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1889&h=78&s=33259&e=png)

如果找到了这么一个 TransitionHandler 对象，那么就把它保存在 ActiveTransition.mHandler 中，后续 onTransitionReady 流程会用。

4）、调用 WIndowOrganizer.startTransition 来启动一个已经创建的 Transition，这最终会调用到 WindowOrganizerController.startTransition，这就又回到 WMCore 的部分了，下一节再说。

5）、将 WMCore 传过来的 Transition 的 token 赋值给 ActiveTransition.mToken。

6）、将这个创建的 ActiveTransition 对象添加到成员变量 mPendingTransitions 中：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/302ba3cd768f4d01b55cf7b8dfb9f650~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=764&h=105&s=18898&e=png&b=2b2b2b)

成员变量 mPendingTransitions 是一个 ActiveTransition 的队列，用来保存那些已经启动，但是还没有就绪的 Transition。

后续 Transition 就绪的时候，即 onTransitionReady 流程，就会从 mPendingTransitions 中拿 ActiveTransition 对象来执行动画。

3 WMCore 部分 —— Transition.start
-------------------------------

直接看 WindowOrganizerController.startTransition：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5cba04f6f8c44b50a1b94e2536a6cdc0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=738&h=372&s=33305&e=png&b=ffffff)

从 WMCore 到 WMShell 再到 WMCore，我们始终传递着 IBinder 类型的 Transition 对应的 token，因此这里的 Transition 是不为 null 的。

因此只需要关注最后调用的 Transition.start。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6af81f0413b44ef79b3bacfad8a79fde~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=798&h=443&s=52879&e=png&b=fffefe)

正式启动 Transition，再次之前可以收集参与者，但是直到启动后 Transition 才会被视为就绪，即使所有的参与者都已经绘制完成。

1）、将 Transition 的状态置为 STATE_STARTED，之前的状态为 STATE_COLLECTING。

2）、调用 Transition.applyReady。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd68fd4ee21a4d50b4529d2f6bfd0e6d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=629&h=250&s=26344&e=png&b=fefefe)

继续调用了 BLASTSyncEngine.setReady 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b84caef6467427fa8cb58f4414eb22f~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=420&h=95&s=12549&e=png&b=2b2b2b)

最终调用的是 SyncGroup.setReady 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c0f853ac674c495c8ecbe8c5bb0fa5f7~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=781&h=203&s=17857&e=png&b=ffffff)

重要的其实就那一句，即将 SyncGroup 的成员变量 mReady 设置为 true，顺便可以调用 WindowSurfacePlacer.requestTraversal，看看 SyncGroup 是否完成，即它里面的所有参与者是否已经准备就绪（绘制完成）。

mReady 这个成员变量还是很重要的，只有被设置为 true，系统才会去检查这个 SyncGroup 是否完成。

4 小结
----

启动 Transition 这一节的内容还是挺多的，大概总结一下：

1）、在 WMCore 侧，调用 ITransitionPlayer.requestStartTransition 切换到 WMShell。

2）、在 WMShell 侧，创建一个 Transitions.ActiveTransition 对象，为其找到一个后续执行动画的 Transitions.TransitionHandler 对象，接着将该 Transitions.ActiveTransition 对象添加到 Transitions.mPendingTransitions 中，后续动画就绪的时候，就是从 Transitions.mPendingTransitions 中拿需要 play 的 Transitions.ActiveTransition 对象。

3）、回到 WMCore 侧，得知在 WMShell 侧创建了和当前 Transition 一一对应的 ActiveTransition 后，这个 Transition 的状态就从 STATE_COLLECTING 切换为 STATE_STARTED，并且相应的 SyncGroup 的成员变量 mReady 也被设置为 true。

启动 Transition 的流程再总结的简单一点，就是之前在 WMCore 这一侧不是创建了一个 Transition 对象嘛，现在需要在 WMShell 这一侧创建一个相应的 ActiveTransition。一旦这个 ActiveTransition 创建完成了，WMShell 再告知 WMCore，WMCore 就知道 WMShell 那边的前序工作已经完成了，那就把 Transition 标记为 STATE_STARTED，SyncGroup 标记为 ready。后续 Transition 满足进入下一个阶段的条件了，再看 SyncGroup 是不是 ready 了，如果 ready 了，就说明 WMShell 那也有一个 ActiveTransition 了，Transition 就可以进入下一个阶段了。

5 从 WMShell 发起 Transition
-------------------------

之前分析的流程是 WMCore 先创建一个 Transition，然后 WMShell 再创建一个 ActiveTransition。

还有一种方式是 WMShell 先创建 ActiveTransition，然后 WMCore 再创建 Transition，即 Transitions.startTransition：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edb80dd05b78474d9c482e60484c4d39~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=817&h=179&s=24607&e=png&b=fefcfc)

这个方法被很多地方调用，但都在 WMShell：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc685f3a6b94469fb2016fc6bc789fa8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1680&h=658&s=203577&e=png&b=3e4143)

可以认为这种方式是由 WMShell 发起了 Transition，而我们之前分析的流程是 WMCore 发起了 Transition。并且这里回调 IWindowOrganizerController 的是 startNewTransition，而我们之前分析的流程回调的是 startTransition，区别即在于传入的 Transition 的 token 是否为 null：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1b28c6c94284340a5623da3ee07476e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=762&h=169&s=15731&e=png)

而在 WindowOrganizerController.startTransition 中，判断传入的 Transition 为 null，就说明流程是从 WMShell 发起的，才会去创建 Transition 对象：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca223771aa794f16800b21adfd1882b7~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=783&h=596&s=55983&e=png)

看一下这里调用的 TransitionController.startCollectOrQueue 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f983e09033a4473aafc6e69c9f665ee~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=750&h=234&s=20028&e=png&b=fffefe)

我们不讨论排队的情况，并且由于 Transition 流程是从 WMShell 发起，所以之前也没有创建过 SyncGroup，所以这里是直接调用了 TransitionController.moveToCollecting，这个我们之前已经分析过了，重点看下这里的 OnStartCollect：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d13abc99aac459d81085981ea327106~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=458&h=111&s=11753&e=png&b=2b2b2b)

它接受一个 boolean 的参数，用来表示这个 Transition 的启动是否被推迟了，比较典型的场景是在某个 TransitionA 过程中灭屏，因为灭屏而新创建的这个 TransitionB 的启动会被推迟，需要等待 TransitionA 完成。TransitionA 完成的时候 TransitionB 会被启动，即 OnStartCollect 的 onCollectStarted 方法在 TransitionA 结束的时候才被调用，而一般来说 Transition 在创建后很快就会被启动，因此这种场景下我们就可以认为 TransitionB 的启动其实是被推迟了，传参 deferred 就给一个 true 值。这个时候会检查 TransitionB 是否还需要启动，或者是被中止（如果 TransitionA 完成的时候屏幕已经被唤醒了），具体在 RootWindowContainer.applySleepTokens：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/532a0f08c6e0438e8beb36a7b8e16ec9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=743&h=266&s=40031&e=png&b=2b2b2b)

这里只是大概提一嘴，不过多分析。

另外在我们分析的流程中，OnStartCollect 的 onCollectStarted 方法是被立即调用的，即以下内容会立即执行：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b096e3acddf44b1e96aae28103cba507~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=601&h=164&s=10835&e=png&b=fffefe)

1）、调用 Transition.start，之前分析过不用多说。

2）、调用 WindowOrganizerController.applyTransaction，因为 Transition 是从 WMShell 发起的，那么 WMShell 也要引起一些变化才行，不然 WMCore 这边不知道哪个 WindowContainer 变化了，哪些 WindowContainer 要参与动画。要引起变化，靠的就是 WindowOrganizerController.applyTransaction，变化的信息则是保存在了 WindowContainerTransaction，这方面也不细讲了，之前分析 WindowContainerTransaction 系列文章的时候有详细分析过。
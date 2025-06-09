> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7382893339097956393)

> BLASTSyncEngine SyncGroup tryFinish finishNow TransactionCommittedListener CommitCallback

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7fc23e4b3f514036ba70e4b85b0ba4f7~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2000&h=1125&s=2453544&e=jpg&b=1d1104) 这一节的内容在 WMCore 中，回想我们的场景，是在 Launcher 启动某一个 App，那么参与动画的就是该 App 对应 Task（OPEN），以及 Launcher App 对应的 Task（TO_BACK）。在确定了动画的参与者后，接下来我们就需要等待动画参与者绘制完成。一个基本的逻辑是，参与动画的主体要绘制出来了才能开始动画，不然动画都开始执行了，窗口还没有绘制出来，那对于用户来说屏幕上是没有任何变化的，这不就有点尴尬了。

ShellTransitions 之前，检查动画是否可以开始的逻辑是在 AppTransitionController.handleAppTransitionReady 中，通过调用 transitionGoodToGo 来检查窗口是否绘制完成的，现在则是在 BLASTSyncEngine.onSurfacePlacement 中，通过调用 BLASTSyncEngine.SyncGroup.tryFinish 不断检查所有动画参与者是否已经全部同步完毕。一旦所有的动画参与者完成同步，则视为 SyncGroup 完成，或者说 Transition 就绪，就会调用 BLASTSyncEngine.SyncGroup.finishNow，最终走到 Transition.onTransactionReady，具体的调用堆栈为：

RootWindowContainer.performSurfacePlacementNoTrace

-> BLASTSyncEngine.onSurfacePlacement

-> BLASTSyncEngine.SyncGroup.tryFinish

-> BLASTSyncEngine.SyncGroup.finishNow

-> Transition.onTransactionReady

我们这一节先分析检查 SyncGroup 是否完成（BLASTSyncEngine.SyncGroup.tryFinish）以及 SyncGroup 完成（BLASTSyncEngine.SyncGroup.finishNow）的内容，Transition 就绪的内容即 Transition.onTransactionReady 放到单独一章分析。

BLASTSyncEngine.onSurfacePlacement 由 RootWindowContainer.performSurfacePlacementNoTrace 调用，代码为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0cb3324cf9a642028f0b30e823d3576c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=799&h=347&s=39776&e=png&b=ffffff)

BLASTSyncEngine 的成员变量 mActiveSyncs 之前已经介绍过了，当我们创建 Transition 的时候，也会创建一个 SyncGroup，来收集参与动画的 WindowContainer，创建的 SyncGroup 则保存在了 BLASTSyncEngine.mActiveSyncs。

这里则是从 BLASTSyncEngine.mActiveSyncs 中拿出 SyncGroup，调用 SyncGroup.tryFinish 来检查 SyncGroup 是否完成。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a297d68f2ec740d7ab8f59895c2cf0e9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=770&h=277&s=28039&e=png&b=fffefe)

1）、如果 SyncGroup.mReady 为 false，则直接返回。之前我们分析 Transition 的启动流程时，知道了只有当 WMShell 侧创建了一个 ActiveTransition 后切换回 WMCore 并且调用 Transition.start，Transition 才算正式启动，正是在 Transition.start 中，调用了 SyncGroup.setReady 方法将 SyncGroup 的 mReady 设置为了 true。从这里我们就看到了 SyncGroup.mReady 的作用，如果 Transition 没有启动，那么这里是不会去检查其对应的 SyncGroup 是否完成了的，而是直接返回 false，也就是说如果 SyncGroup 没有 ready，那么 Transition 将无法走到下一个阶段。

2）、SyncGroup.mRootMembers 则保存了参与动画的 WindowContainer，我们这里则是为每一个 WindowContainer 调用 WindowContainer.isSyncFinished 来检查这个 WindowContainer 是否完成同步 / 绘制，只要有一个没有完成同步，那么就直接返回 false，不需要往下走了，等待下一次 RootWindowContainer.performSurfacePlacementNoTrace 到来的时候再检查看看有没有完成同步。

3）、如果所有参与动画的 WindowContainer 都已经完成同步了，那么就继续调用 SyncGroup.finishNow 来将当前 SyncGroup 结束掉。

我们先分析 WindowContainer.isSyncFinished，再去分析 SyncGroup.finishNow。

2.1 WindowContainer.isSyncFinished
----------------------------------

首先再回顾一下，当把 WindowContainer 添加 SyncGroup 的时候，会为每一个 WindowContainer 调用 prepareSync 方法，结果是：

1）、WindowState 类型的 WindowContainer，其 mSyncState 被置为 SYNC_STATE_WAITING_FOR_DRAW。

2）、非 WindowState 类型的 WindowContainer，其 mSyncState 被置 SYNC_STATE_READY。

再看 WindowContainer.isSyncFinished 的内容：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e6cf865e20844bbb3c5ada16236e9ef~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=783&h=525&s=45742&e=png&b=fffefe)

1）、首先一个基本的规则是，如果一个 WindowContainer 请求的是不可见的，那么将其视为同步完成。这个其实也很好理解，如果这个 WindowContainer 在完成动画的时候是不希望被显示的，那么就不需要等待它绘制完成了。

2）、对于非 WindowState 类型的 WindowContainer，比如 Task 或者 ActivityRecord，由于它们的 mSyncState 一开始就被设置为了 SYNC_STATE_READY，因此它们主要是检查它们的 mChildren 是否同步完成，最终检查的就是其中的 WindowState 是否完成同步 / 绘制。一旦有一个 WindowState：

*   同步完成。

*   请求可见。

*   和父容器大小相等。

只要找到一个符合以上条件的 WindowState，那么就可以认为这个 WindowContainer 已经完成了同步。

3）、对于 WindowState，则只有一个标准，即其 mSyncState 是否是 SYNC_STATE_READY（暂不考虑子窗口的情况）。

2.2 WindowContainer.onSyncFinishedDrawing
-----------------------------------------

再看下什么时候 WindowState 的 mSyncState 什么被设置为 SYNC_STATE_READY。

WindowState 的 mSyncState 被设置为 SYNC_STATE_READY 的地方只有一处，在 WindowContainer.onSyncFinishedDrawing：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66de0d6fd10f47eda1f41b604e93e49f~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=650&h=194&s=18363&e=png&b=fffefe)

从注释也能看出，当 WindowContainer 完成绘制其内容的时候，这个方法会被调用。

具体调用的地方则是：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7063b871d15b4faa8f98d10ac29181ce~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=810&h=190&s=34371&e=png&b=424648)

很明显，当窗口完成绘制时，会调用 WindowState.finishDrawing，进而将 WindowState 的 mSyncState 设置为 SYNC_STATE_READY。

而一旦 Task/ActivityRecord 中的 WindowState 绘制完成，那么该 Task/ActivityRecord 就会被视为同步完成。

这部分的内容之前分析 WindowContainerTransaction 的文章有过更加详细的介绍：

[4【Android 12】【WCT 的同步】BLASTSyncEngine - 掘金 (juejin.cn)](https://juejin.cn/post/7072572432778788894#heading-13 "https://juejin.cn/post/7072572432778788894#heading-13")

更多的详细内容可以看下当时的分析。

一旦 SyncGroup 中所有的动画参与者都同步完成，那么就调用 SyncGroup.finishNow 来结束掉这个 SyncGroup。

这个方法是我们本篇文章的重点，且内容较多，我们分段去看。

3.1 合并 Transaction
------------------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb5c37dd572d40ae97fbfa316c32aec8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=648&h=245&s=21045&e=png&b=ffffff)

这里从对象工厂中拿到一个 Transaction 对象，局部变量 merged，对于所有参与动画的 WindowContainer，将它们在动画期间发生的同步操作都合并到这个局部变量 merged 中。

这一点主要是通过对所有参与动画的 WindowContainer 调用 WindowContainer.finishSync 方法来完成的，indowContainer.finishSync 方法内容为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1936354867504ec2a98d9143d20b6c07~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=762&h=369&s=49526&e=png&b=fffefe)

1）、将 WindowContainer.mSyncTransaction 中收集到的对当前 WindowContainer 对应的 SurfaceControl 的修改（同步操作）合并到传参 outMergedTransaction 中，即我们上面提到的 SyncGroup.finishNow 中的局部变量 merged。

2）、递归调用所有子 WindowContainer 的 finishSync 方法，最终的结果是将这个 WindowContainer 以及所有子 WindowContainer 的同步操作都收集到传参 outMergedTransaction 中。

3）、最后由于同步工作已经结束，那么将这个 WindowContainer 的 mSyncState 以及 mSyncGroup 之类的成员变量进行重置。

这里所说的同步操作主要是针对 WindowContainer.mSyncTransaction 来说的，其实之前也看到过 “sync” 这个字眼了，我们这里来大致说明一下 “同步” 这个概念。

### 3.1.1 “同步” 的概念

其实最早的时候，BLASTSyncEngine 以及 SyncGroup 并不是用于动画，而是和 WindowContainerTransaction 一起结合使用，主要是用于分屏。

分屏由于将屏幕一分为二以供两个 App 同时显示，那么一旦分屏发生变化（进退分屏，调整分割线位置等），那么最起码就会有两个可见的 SurfaceControl 参与了变化，即参与分屏的这两个 App 下的 SurfaceControl。

那么一个很明显能够想到的问题是，如何做到这两个分屏的 App 界面能够一起改变呢，比如我调整分屏分割线的位置，我肯定不希望看到上分屏的 App 界面改变之后，下分屏的 App 界面没有跟着一起改变，而是又过了几百毫秒才开始变化。这是一个肯定会遇到的问题，App 界面改变是在窗口绘制完成之后，而 WMS 无法控制窗口绘制的时间，因此如果 WMS 不加以控制，那么就会出现由于两个窗口的绘制时间不同，导致用户看到的两个界面先后进行了改变（异步），而非同一时间进行了改变（同步）。

因此当时引出了 BLASTSyncEngine 以及 SyncGroup 的概念，这套机制最开始就是用来保证使用 WindowContainerTransaction 的模块（比如分屏）可以做到 SurfaceControl 的同步。

同步的大致做法，则是创建一个统一的 Transaction 对象（即 SyncGroup.finishNow 中创建的那个 Transaction 类型的局部变量 merged），来收集所有参与到分屏的 SurfaceControl 的变化，并且只有等到所有参与分屏的窗口都绘制完成后，才对这个 Transaction 对象调用 apply 方法，这样就保证了所有的 SurfaceControl 变化在一次 Transaction.apply 中进行了提交。

从以上介绍可以看到要实现同步，有两个比较重要点，一是有一个统一的 Transaction 来收集所有 SurfaceControl 的变化，二是当所有参与同步的窗口绘制完成后再调用 Transaction.apply。

更多的内容可以去看下之前分析 WindowContainerTransaction 的系列文章。

### 3.1.2 WindowContainer.mSyncTransaction

回到现在 Android14 的 ShellTransitions 中来，现在是动画也开始用 BLASTSyncEngine 这一套逻辑了，接下来看下现在动画是如何实现同步的。

从上面的介绍中，我们知道同步的两个重要的点是：

1）、有一个统一的 Transaction 来收集所有 SurfaceControl 的变化。

2）、当所有参与同步的窗口绘制完成后再调用这个统一的 Transaction 对象的 Transaction.apply 方法。

可知和这个 Transaction 是有很大的关系。

第二点放到后面再说，这一节我们分析一下第一点，即有一个统一的 Transaction 来收集所有 SurfaceControl 的变化。

之前看 SyncGroup.finishNow，我们知道了这个统一的 Transaction 就是这里创建的 Transaction 类型的局部变量 merged，它合并了所有参与动画的 WindowContainer 的 mSyncTransaction 中收集的内容，那么 WindowContainer.mSyncTransaction 又是什么？

再举一个例子，来看一个经典的对 SurfaceControl 进行显示的操作，为 WindowSurfaceController.showRobustly：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/960d4770cfbd4a568f77e50c278e16fe~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=445&h=95&s=4626&e=png&b=ffffff)

根据以往的系列文章知 WindowSurfaceController.mSurfaceControl 对应的是 SurfaceFlinger 侧的 BufferStateLayer。

调用 Transaction.show 的时候，只是将对 SurfaceControl 的操作暂存在了 Transaction 中（更准确的说，是 native 层的 layer_state_t 结构体中），只有当调用 Transaction.apply 的时候，这个对 SurfaceControl 的操作才算真正提交到了 SurfaceFlinger 端，进而作用到了 Layer 上。

那么我们再看下这个传参 Transaction 对象是从哪里拿到的，一般来说，调用堆栈为：

WindowState.prepareSurfaces

-> WindowStateAnimator.prepareSurfaces

-> WindowSurfaceController.showRebustly

看到是在 WindowState.prepareSurfaces 中，通过 WindowContainer.getSyncTransaction 拿到的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48481e2e1bbe4f2cab675ca560ec8534~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=710&h=228&s=25986&e=png&b=ffffff)

WindowContainer.getSyncTransaction 为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3c621b42ac1460b9774d5e8e37875b6~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=786&h=302&s=30209&e=png&b=fffefe)

如果 WindowContainer.mSyncTransactionCommitCallbackDepth 大于 0，或者 WindowContainer.mSyncTransaction 不为 SYNC_STATE_NONE，说明此时 WindowContainer 仍然处于需要同步的场景中，因此返回 WindowContainer.mSyncTransaction，否则返回 WindowContainer.getPendingTransaction。

WindowContainer.getPendingTransaction 为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dcefdb38290247388e1f5960e0ab42e5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=822&h=211&s=23869&e=png&b=ffffff)

一般返回的是 DisplayContent 的 mPendingTransaction。

再看下 mSyncTransaction 和 mPendingTransaction 的定义以及初始化：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1389801c1b2646e685c5560ea4a279b0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=693&h=189&s=18181&e=png&b=fffefe)

看到 mSyncTransaction 和 mPendingTransaction 其实都是一个普通的 Transaction 对象，本质上没有区别，区别在于它们的使用方式：

1）、pendingTransaction 基本上每次 RootWindowContainer.performSurfacePlacementNoTrace 就 apply 一次：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d006c364b95c4b78995f6de16d2a321d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=745&h=483&s=90776&e=png&b=1f2023)

可以认为是使用 pendingTransaction 对 SurfaceControl 操作后，很快就会调用 Transaction.apply，也就是说使用 pendingTransaction 对 SurfaceControl 进行的操作很快就能见到效果。

2）、syncTransaction 的 apply 方法的调用时机则是和 Transition 的流程密切相关，只有走到特定的阶段才会调用 Transaction.apply 方法，以后的分析中我们会看到。

最后一句话总结一下 pendingTransaction 和 syncTransaction 的区别就是，需要 WindowContaienr 同步的场景使用 syncTransaction，不需要 WindowContainer 同步的场景则使用 pendingTransaction...... 怎么有点像废话呢

3.2 注册 TransactionCommittedListener 回调以及超时处理
--------------------------------------------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e83e14988e234963bdd4620457bda544~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=707&h=796&s=82523&e=png&b=ffffff)

主要内容是：

1）、为所有参与到动画的 WindowContainer 调用 waitForSyncTransactionCommit 方法.

2）、定义一个 CommitCallback 的类，这个类有一个自定义的 onCommitted 方法，以及复写 Runnable 的 run 方法。

3）、创建一个 CommitCallback 类的对象，callback。

4）、调用 Transaction.addTransactionCommittedListener 方法注册 TransactionCommittedListener 回调，回调触发的时候执行这个 callback 的 onCommitted 方法。

5）、Handler.postDelayed 将这个 callback 添加到了 MessageQueue 中，5000ms 超时之后执行这个 callback 的 run 方法。

接下来分别介绍，并不一定按照顺序。

### 3.2.1 注册 TransactionCommittedListener 回调

```
            CommitCallback callback = new CommitCallback();
            merged.addTransactionCommittedListener(Runnable::run,
                    () -> callback.onCommitted(new SurfaceControl.Transaction()));


```

看到这里为 merged 调用了 Transaction.addTransactionCommittedListener 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7730fe02efac411484eedd290895df39~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=906&h=582&s=68595&e=png&b=fffefe)

从注释来看，TransactionCommittedListener 的 onTransactionCommitted 回调方法会在 Transaction 被 apply 的时候调用，另外这个回调被执行的时候也说明当前 Transaction 将不会被一个新的 Transaction 对象复写。

那么再结合 SyncGroup.finishNow 的代码，也就是说，当 merged 这个 Transaction 对象被 apply 后，Transaction.addTransactionCommittedListener 这段代码将被执行：

```
executor.execute(listener::onTransactionCommitted)


```

也就是异步执行传参 listener 的 onTransactionCommitted 方法，即 SyncGroup.finishNow 中的这段代码：

```
callback.onCommitted(new SurfaceControl.Transaction())


```

即当 merged 这个 Transaction 对象被 apply 后，这里定义的 CommitCallback 类的 onCommitted 方法将会被执行。

分析了注册 TransactionCommittedListener 回调后，我们可以再回过头来看 CommitCallback 类的定义，即它的 onCommitted 方法和 run 方法。

### 3.2.2 CommitCallback.onCommitted

先看 onCommitted 方法，从上面的分析我们知道这个方法将会在 merged 被 apply 的时候调用，作用为：

1）、将 CommitCallback 从 MessageQueue 中移除，即 merged 在规定的 5000ms 内得到 apply 了，那么就不需要触发超时了。

2）、将 ran 这个变量置为 true，因为 Transaction 有可能在 5000ms 超时后才 apply，那么 onCommitted 方法就有可能走两次。

3）、调用 WindowContainer 的 onSyncTransactionCommitted 方法，onSyncTransactionCommitted 方法要和 waitForSyncTransactionCommit 结合着来看：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c96a8c02a3242158fa9fb3f1857d8a2~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=659&h=402&s=35539&e=png&b=fffefe)

正好 WindowContainer.waitForSyncTransactionCommit 方法也是在上面被调用了，感觉这两个方法主要是对 mSyncTransactionCommitCallbackDepth 这个成员变量进行操作，而 mSyncTransactionCommitCallbackDepth 作用的地方也在我们之前看过的 WindowContainer.getSyncTransaction 中：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/85bdb43e49de4de3947692dbe16cccad~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=438&h=166&s=12066&e=png&b=fffefe)

我自己的看法是，这个变量用来继续延长 WindowContainer 的同步状态。

如我们之前第一节合并 Transaction 中提到的，这里会为所有参与同步的 WindowContainer 调用 WindowContainer.finishSync 方法，这将会使得 WindowContainer 的 mSyncState 重置为 SYNC_STATE_NONE，那么假如没有 mSyncTransactionCommitCallbackDepth，此时调用 WindowContainer.getSyncTransaction 将会返回 pendingTransaction，而非 syncTransaction，也就是说 WindowContainer 的同步状态在走到 SyncGroup.finishNow 的时候就结束了。

而加入了 mSyncTransactionCommitCallbackDepth 之后，WindowContainer 的同步状态的结束将会被延迟到 merged 被 apply 的时候。

为什么要这么做呢，因为此时 SyncGroup.finishNow 距离 merged 被 apply 还有一段时间，而且这个时间其实可能会超过 5000ms，即上面规定的超时时限。假如在 merged 被 apply 之前，WindowContainer 又发生了变化，那么如果没有 mSyncTransactionCommitCallbackDepth 的存在，此时 WindowContainer 将使用 pendingTransaction，并且 pendingTransaction 如果再在 merged 被 apply 之前就 apply，就会出现新的 Transaction（pendingTransaction）的内容被旧的 Transaction（syncTransaction）内容覆盖的情况。

4）、调用 WindowContainer.onSyncTransactionCommitted，将所有参与动画的 WindowContainer.mSyncTransaction 收集到 Transaction 类型传参 t 中，集中进行一次 apply。

如之前所说，此时距离 merged 被 apply 还有一段时间，在这段时间内参与到动画的 WindowContainer 是有可能继续发生变化的，而 syncTransaction 合并到 merged 的操作已经结束了，为了让这个时间段的变化也能够被应用，所以这里调用 WindowContainer.mSyncTransaction，将收集到变化的 syncTransaction 都合并到一个 Transaction 中，然后调用 apply。

但是这样不就是后发生变化的 WindowContainer 的 Transaction 先被 apply 了吗，这样不是还会出现上面提到的 Transaction 被覆盖的情况？目前我暂时没有碰到过这种情况，但是这个逻辑我感觉是有问题的。

### 3.2.3 CommitCallback.run

根据我们的分析，我们知道了这个方法将会在 5000ms 超时后调用，主要的内容是：

1）、调用 TransactionReadyListener.onTransactionCommitTimeout，通知关心方超时的情况。

2）、调用 CommitCallback.onCommitted 方法，应该是想让 syncTransaction 收集到的变化得到应用，但是之前合并到 merged 那部分变化则是永久丢失掉了，这部分应该才是最重要的。

3）、这里打印了一条 log：

```
                    Slog.e(TAG, "WM sent Transaction to organized, but never received" +
                           " commit callback. Application ANR likely to follow.");


```

打印了这个条 log 的时候，我们已经知道是处于 5000ms 超时的情况了，那么可能会出现本来应该显示的 Layer，在 5000ms 的时间内得不到显示，那么屏幕上就可能会出现没有任何一个输入窗口可以作为焦点窗口的情况（输入窗口能够接收焦点，需要其 Layer 为可见），如果此时再来一个 KeyEvent 事件，那么就会发生无焦点窗口的 ANR。

3.3 调用 Transition.onTransactionReady
------------------------------------

只剩下最后一点内容了，一起来看下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8528cd610fed4939843026f32cc659d6~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=597&h=180&s=15149&e=png&b=ffffff)

1）、调用 TransactionReadyListener 类型的 mListener 的 onTransactionReady 方法。

2）、将当前 SyncGroup 从 BLASTSyncEngine.mActiveSyncs 中移除。

3）、将其成员变量 mOnTimeout 从 MessageQueue 中移除。

有关其成员变量 mListener 以及 mOnTimeout 的部分，在之前创建 SyncGroup 对象的时候漏说了，现在大概过一遍。

首先看下 SyncGroup 的构造方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/94794757151c43f9a23e7394b4ab3c41~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=653&h=235&s=20787&e=png&b=fffefe)

1）、int 类型的 mSyncId 成员变量保存该 SyncGroup 的 ID。

2）、TransactionReadyListener 类型的成员变量 mListener 保存与该 SyncGroup 一一对应的 Transition 对象，TransactionReadyListener 定义为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d98cfeff3873451fb3cd700981456b12~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=805&h=150&s=24754&e=png&b=1e1f22)

用来通知 Transition 同步完成以及 Transaction 提交超时。

所以这里调用的是 Transition.onTransactionReady，Transition.onTransactionReady 的内容比较多，需要单独开一篇分析。

3）、mOnTimeout 则是一个 Runnable，用来在超时的时候触发 BLASTSyncEngine.onTImeout 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f733e5de9b9e4dd08763ea4dfa34b040~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=763&h=345&s=39066&e=png&b=fffefe)

主要内容就是遍历一下参与同步的 WindowContainer，看下是哪个 WindowContainer 没有同步完成，以及在方法的最终调用 SyncGroup.finishNow，这个一点也很好理解，毕竟我们不能无限等待某一个 WindowState 绘制完成。

注意区分一下这里的超时和我们上面提到的超时。

这里的超时是在 SyncGroup 刚刚创建，或者说 Transition 刚开始收集的时候，开始计时的，防止某一个窗口迟迟没有完成绘制，从而无限等待这个窗口绘制完成的情况。

上面提到的超时是在 SyncGroup.finishNow 的时候开始计时的，防止 merged 这个 Transaction 迟迟没有得到 apply（Transition 没有走到下一个阶段），从而 syncTransaction 的收集的变化内容无法被 apply 的情况。
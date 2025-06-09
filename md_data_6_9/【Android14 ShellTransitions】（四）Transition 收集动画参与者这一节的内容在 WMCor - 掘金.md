> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7371709743222013991)

> 这一节的内容在 WMCore 中，现在 Transition 已经走到 COLLECTING 状态了，并且可以收集动画参与者了。 那么 Transition 是在什么时候去收集动画参与者？回到我们最初的 Activit

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/410dd3966bd1405c99bb30c6003155c8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1991&h=1120&s=1476229&e=jpg&b=1e1d1a)

这一节的内容在 WMCore 中，现在 Transition 已经走到 COLLECTING 状态了，并且可以收集动画参与者了。

那么 Transition 是在什么时候去收集动画参与者？回到我们最初的 ActivityStarter.startActivityUnchecked：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a2d4632f9afe44aebddda74c3c3b2a0e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=817&h=662&s=66976&e=png&b=fefdfd)

在调用了 TransitionController.createAndStartCollecting 后，紧接着就是调用了 TransitionController.collect。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/999e3b7ec32c4eb4817d58a152515f8c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=438&h=89&s=8237&e=png&b=fefdfd)

这里看到 TransitionController 里面首先是判断了一下成员变量 mCollectingTransition 是否为空，如果不为空，才能将传参 WindowContainer 收集到该 Transition 中。

接下来我们重点分析 Transition.collect，这里感觉不按照代码顺序分析似乎更有条理一点。

1 Transition.collect 第 1 部分
---------------------------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4033f880fa654f9caded275cb1dc7813~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=805&h=316&s=31648&e=png&b=fefdfd)

1）、如果当前 Transition 的 mState 小于 STATE_COLLECTING，那么就会异常。

2）、如果当前 Transition 的 mState 不是 STATE_COLLECTING 或者 STATE_STARTED，那么就会返回。

这里说明了，只有处于 STATE_COLLECTING 或者 STATE_STARTED 的 Transition 才能收集动画参与者，太早或者太晚都不行。

2 Transition.collect 第 2 部分
---------------------------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d636d0cad26145cabc2bb271dddeec64~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=801&h=565&s=65351&e=png&b=fefefe)

首先看这里 needSync 的逻辑。

如果，当前正在被收集的 WindowContainer 不是一个 WallpaperWindowToken，

或者，当前参与者中包含该 WindowContainer 所在的 DisplayContent，

并且，该 WindowContainer 没有被瞬态隐藏（被 Recents 遮挡），

那么 needSync 就为 true。

needSync 为 true，那么继续调用 BLASTSyncEngine.addToSycnSet 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/28755b42358e42f2ad28668f71cdfdfe~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=446&h=59&s=5652&e=png&b=fefdfd)

很简单，根据传入的 Transition 的 ID 找到对应的 SyncGroup 对象，然后继续调用 SyncGroup.addToSync 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2da1a856b20b4d378d9cebe361ac0933~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=785&h=320&s=32324&e=png&b=ffffff)

该方法内容主要是：

1）、如果当前 WindowContainer 通过 getSyncGroup 拿到的 SyncGroup 为空，说明当前 WindowContainer 还没有绑定一个 SyncGroup，那么就将该 WindowContainer 添加到 SyncGroup.mRootMembers 中，并且通过 WindowContainer.setSyncGroup 方法，将 WindowContainer 的成员变量 mSyncGroup 设置为当前 SyncGroup，完成绑定。

2）、调用 WindowContainer.prepareSync 方法。

关于这个 prepareSync 的逻辑，同样在我们之前分析 BLASTSyncEngine 的那篇文章中也比较系统的介绍过，因此这里简单再回顾一下。

首先是 WindowContainer.prepare 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8c42278eae1e46d692da5c73b081f562~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=812&h=302&s=26353&e=png&b=fefefe)

比较简单，调用子 WindowContainer 的 prepareSync 方法，然后将自身的 mSyncState 设置为 SYNC_STATE_READY。

mSyncState 有三种状态：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d944092d6e83415f9e2af83b1f1e67c1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=674&h=165&s=15052&e=png&b=fefdfd)

*   SYNC_STATE_NONE，没有参与到一次同步（或 Transition）中。

*   SYNC_STATE_WAITING_FOR_DRAW，等待自身绘制完成。

*   SYNC_STATE_READY，当前 WindowContainer 已经就绪，但是其子容器可能没有就绪。

一般来说，动画都伴随着界面的改变，比如我们从 Launcher 启动 Contacts，Contacts 是一个从无到有的过程，那么动画就需要等待 Contacts 的窗口绘制完成后才能进行。

因此再看重写了 prepareSync 方法的 WindowState.prepareSync 方法的内容：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d301da20ccc4690bfa927478410e35a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=800&h=631&s=71128&e=png&b=ffffff)

重点其实就是这一句，将 WindowState 的 mSyncState 置为 SYNC_STATE_WAITING_FOR_DRAW。

即对于非 WindowState 的 WindowContainer，比如 Task，ActivityRecord，它们在界面发生改变的时候是不需要重绘的，因此它们在 prepareSync 的时候就可以将 mState 设置为 SYNC_STATE_READY。同理，判断一个 Task 或者一个 ActivityRecord 是否就绪，看的主要是它里面的 WindowState 是否就绪了。

而对于 WindowState，在界面发生改变的时候，由于要重绘，并且重绘需要时间，因此 WindowState 在被添加到动画参与者集合的时候，会先被设置为 SYNC_STATE_WAITING_FOR_DRAW。等到绘制完成，为该 WindowState 调用 finishDrawing 的时候，它的 mSyncState 会被设置为 SYNC_STATE_READY，表示该窗口已经就绪，可以进行动画。

3 Transition.collect 第 3 部分
---------------------------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3f758a566414466a2a97ec8d04099a6~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=795&h=572&s=62523&e=png&b=ffffff)

接着看 Transition 的成员变量 mParticipants：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e622ca52fd64d2dbd8deb0230f488f5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=620&h=103&s=13434&e=png&b=2b2b2b)

是一个 ArraySet 类型的成员变量，看方法最后，是将参与当前 Transition 的 WindowContainer 添加进来了，那它的作用和 SyncGroup 的 mRootMembers 类似。

从第一部分关于 needSync 的分析看到，向 SyncGroup.mRootMembers 添加参与者的条件要比向 Transition.mParticipants 的条件苛刻一点，因为这个区别，这两个参与者集合中的内容可能不是完全相同的。

4 Transition.collect 第 4 部分
---------------------------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/26d111619a874cfc84a0c6eb69ea0ccd~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=815&h=578&s=68167&e=png&b=fefdfd)

最后我们再来看一下关于 ChangeInfo 的这部分。

### 4.1 例子 1 —— 新启动 ActivityRecord

还以我们之前分析的从 Launcher 启动 Contacts 的流程来，那么在这个时间点，只创建了一个 ActivityRecord 对象，它的 Task 还没创建，那么通过 Transition.getAnimatableParent 获取的 WindowContainer 自然也就是 null 了，因此标黄的第一段逻辑就不走了，直接看第二段逻辑。

首先看下 Transition.mChanges 的定义：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/326c96c4203640ad85c13f49c979f543~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=811&h=193&s=29865&e=png&b=2b2b2b)

一个包含了 WindowContainer 和 ChangeInfo 键值对的 map。

这是我们第一次尝试将当前 WindowContainer 添加到动画参与者中，因此从 Transition.mChanges 中肯定是没有当前 WindowContainer 的，因此这里就会根据该 WindowContainer 创建一个 ChangeInfo 对象，并且将这两个对象组成的键值对保存到 Transition.mChanges 中。

ChangeInfo 的构造方法为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a45de93664214fa1bd2bebe83ac6609d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=637&h=306&s=58199&e=png&b=2b2b2b)

用来保存改变开始前的 WindowContainer 的一些信息，大概是这样的用法：

1）、在收集阶段，将基于 WindowContainer 的当前信息创建一个 ChangeInfo，ChangeInfo 里保存了我们对 WindowContainer 感兴趣的特定信息。

2）、发生一些改变。

3）、改变完成后，具体是在 Transition.onTransactionReady 的时候，拿第一步创建 ChangeInfo 和现在这个时候的 WindowContainer 的信息进行对比，从而得知当前 WindowContainer 在这个过程中发生了哪些变化。

### 4.2 例子 2 —— App 切换

在第一个例子中，我们看到由于此时是 ActivityRecord 刚刚创建的时候，因此 ActivityRecord 的父容器为空，所以这里仅仅为正在收集的这个 ActivityRecord 创建了一个 ChangeInfo 的对象。然而更普遍的情况可能是，不仅为当前正在收集的这个 WindowContainer 创建了 ChangeInfo，还可能会创建更多 ChangeInfo 对象， 即这段逻辑：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9800a6884ce043afa696386f48a4b37d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=763&h=173&s=17702&e=png&b=fffefe)

Transition.getAnimatableParent 的定义为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f251453fae849b0894722ac1a62bfb0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=741&h=153&s=15015&e=png&b=fefdfd)

如果发生了 App 的切换，比如从 Message 界面点击 Home 回到 Launcher，那么就会把 Message 和 Launcher 的 Task 收集起来，走到这里时，传参是 Task 类型的 WindowContainer：

1）、首先 Task 通过 getParent 获取到父 WindowContainer，即 TaskDisplayArea。

2）、TaskDisplayArea 复写了 WindowContainer.canCreateRemoteAnimationTarget 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0aa7ed3d2b194335aceb47c0ce2ec57a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=768&h=107&s=12725&e=png&b=fefefe)

返回的是 true，那么通过 Transition.getAnimatableParent 返回了 TaskDisplayArea。

3）、最终会为 TaskDisplayArea 创建一个 ChangeInfo 对象，添加到 Transition.mChanges 中。

同理为 TaskDisplayArea 调用 Transition.getAnimatableParent 方法，获取到的则是 DisplayContent，也会为 DisplayContent 创建一个 ChangeInfo 对象添加到 Transition.mChanges 中。

那么除了为 Task 创建 ChangeInfo 之外，这里还会为 TaskDisplayArea 以及 DisplayContent 创建 ChangeInfo，并且加入到 Transition.mChanges 中。

最后再来对比一下 SyncGroup.mRootMembers、Transition.mParticipants 以及 Transition.mChangs：

1）、Transition.mParticipants，保存的就是正在收集的那个 WindowContainer。

2）、SyncGroup.mRootMembers，由于一些条件限制，保存的 WindowContainer 比 Transition.mParticipants 要少一点。

3）、Transition.mChangs，不仅保存了正在收集的那个 WindowContainer 对应的 ChangeInfo 对象，还可能保存了它的父 WindowContainer 对应的 ChangeInfo 对象。

5 其它 collect 的情况
----------------

看下 Transition.collect 都在哪些地方调用了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0bd2ea9ca409424a9ab6db8218ace8a1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1680&h=580&s=158941&e=png&b=3e4143)

并且它也会被 TransitionController.collect 调用，而 TransitionController.collect 被调用的地方同样也很多：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2fa3f1672ae74b69a66b336f28654f03~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1680&h=658&s=181274&e=png&b=3e4143)

 场景太多就不一一分析了，其实万变不离其宗，需要在什么节点把 WindowContainer 收集进来再去调用 Transition.collect 就行了，就像之前把 ActivityRecord 添加到 openingApps 和 closingApps 那样。
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7371717387829755945)

> 这一节的内容在 WMCore 中，Transition 进入到第二个状态，COLLECTING 状态。 我们看到在上一节 ActivityStarter.startActivityUnchecked 启动 Acti

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e43d9bbfe5e4581a3255585597357d8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2000&h=1125&s=2044082&e=jpg&b=606e74)

这一节的内容在 WMCore 中，Transition 进入到第二个状态，COLLECTING 状态。

我们看到在上一节 ActivityStarter.startActivityUnchecked 启动 Activity 的流程中，调用了 TransitionController.createAndStartCollecting 来创建一个 Transition 对象：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52ddfe26a4b44af1b23b2d2b94831ca5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=799&h=357&s=35856&e=png&b=fefdfd)

创建完 Transition 后，接着调用了 TransitionController.moveToCollecting 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe7b64bba6ab45ae963c751c53c1cca6~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=837&h=328&s=40866&e=png&b=fffefe)

该方法的主要内容为：

1）、将 TransitionController.mCollectingTransition 赋值为传参 Transition，TransitionController.mCollectingTransition 定义为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/20f168f951104d108928319e519705e5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=814&h=100&s=8119&e=png&b=fefefe)

代表了处于收集状态的 Transition，一般情况下，在没有排队的情况下，TransitionController.mCollectingTransition 就是在此方法下被赋值的。

2）、调用 Transition.startCollecting 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b2d8dc03b65f4b4182f1ece4d6faadc7~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=797&h=220&s=23384&e=png&b=fffdfd)

首先将当前 Transition 的状态标记为 STATE_COLLECTING，接着通过 BLASTSyncEngine.startSyncSet 方法，创建一个 SyncGroup，用来收集动画的参与者。

SyncGroup 简介
------------

SyncGroup 的相关内容我们在以前 WindowContainerTransaction 系列文章中那篇关于 BLASTSyncEngine 里其实已经比较细致的讲过了，不过这里还是再看下。

接下来看下 BLASTSyncEngine.startSyncSet 方法的内容：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d5eb25f3061a457c84266c4fae65974c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=828&h=474&s=61650&e=png&b=fefdfd)

其实还是比较清晰的：

1）、调用 BLASTSyncEngine.prepareSyncSet 来创建一个 SyncGroup，这里能看到 BLASTSyncEngine 是用一个全局的成员变量 mNextSyncId 通过自增来生成 SyncGroup 的 ID 的，因此 SyncGroup 的 ID 是逐渐变大的，自然也不会出现重复的情况。

2）、调用 BLASTSyncEngine.startSyncSet 来将当前 SyncGroup 添加到成员变量 mActiveSyncs 中，成员变量 mActiveSyncs 的定义为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae2b0a69db8149aa9ca54ccd575102d5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=681&h=111&s=18344&e=png&b=2b2b2b)

用来跟踪现在有多少个 SyncGroup 在同时进行收集的操作。

另外看到 BLASTSyncEngine.startSyncSet 方法的最后，是将创建的 SyncGroup 的 ID 返回，并且赋值给 Transition.mSyncId，这样 Transition 就可以通过传入其 ID，来在 BLASTSyncEngine 中找到与其一一对应的那个 SyncGroup 了。

SyncGroup，顾名思义，就是一个同步集合，用来保存当前有哪些 WindowContainer 参与到了动画当中，因此一个关键的成员变量就是一个 ArraySet 类型的，保存了参与动画的 WindowContainer 的集合：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d98910572d149cdb6aabf2f76136e42~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=613&h=213&s=16621&e=png&b=fefdfd)

这里可以看到 Transition 走到 PENDING 状态只是表明，Transition 被标记成为了那个唯一的可以收集动画参与者的 Transition，但是真正将动画参与者收集进来则是之后的事情了，有可能马上就收集到了动画参与者（比如接下来我们要分析的），也可能后续陆陆续续收集到了多个动画参与者。
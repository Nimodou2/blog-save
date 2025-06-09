> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7377368708564385828)

> transientLaunch transientHide ANR 问题分析 startExistingRecents setTransientLaunch

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f38d4927fe041d8babe047d93db3cf3~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2000&h=1125&s=2789330&e=jpg&b=c9c2a6)

Monkey 跑出的 Camera 发生 ANR 的问题，其实跟 Camera 无关，任意一个 App 都会在此场景下发生 ANR，场景涉及到 Launcher 的 RecentsActivity 界面，和 transientLaunch 相关。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/708fc22197d24cbb823b564accfbfc12~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1897&h=1046&s=343754&e=png&b=fefdfd)

看问题发生的场景：

1、Camera App 的相关界面 CameraLauncher 拿到焦点，此时正常。

2、Monkey 输入了一个 keycode 为 312 的 KeyEvent（KEYCODE_RECENT_APPS）调起了 RecentsActivity，这是一次 transientLaunch，可以看到 CameraLauncher 的生命周期没有发生变化，以及最终是 “recents_animation_input_consumer” 拿到了焦点，但是奇怪的一点是 CameraLauncher 的 ActivityRecord 被设置不可见了（wm_add_to_stopping 这条 log）。

3、接着 Monkey 应该是又输入了一个 MotionEvent，然后与 “recents_animation_input_consumer” 发生了交互，点击了 Camera 对应的 Recents 缩略图，所以又调起了 CameraLauncher，：

3.1）、刚开始，发生了一次 relayoutWIndow，但是 CameraLauncher 对应的 ActivityRecord 的可见性还没有被设置为 true，所以它的窗口不满足作为焦点窗口的条件，这导致了后续的 DisplayContent.updateFocusedWindowLocked 没有办法将它的窗口设置为焦点窗口。

3.2）、接着 “wm_set_resumed_activity” 之后，CameraLauncher 重新变为 resume，其 ActivityRecord 的可见性也被设置为 true，但是由于 CameraLauncher 对应的 WindowState 的可见性始终没有发生变化，导致了后续再走 relayoutWindow 的时候，没有办法调用 DisplayContent.updateFocusedWindowLocked 去更新 WMS 的焦点窗口，进而无法为 CameraLauncher 请求焦点，最终结果就是上层 WMS 侧以及 native 层 InputDispatcher 侧都没有焦点窗口。

其中有两点比较奇怪：

1、“05-30 18:30:28.853” 时间点，输入了 KeyEvent，KEYCODE_RECENT_APPS，但是调起的是 “com.tcl.android.quickstep.RecentsActivity”，并非 “TclQuickstepLauncher”，这是第一个问题点。

2、从 log 上看启动了 “com.tcl.android.quickstep.RecentsActivity” 后“com.android.camera.CameraLauncher”的生命周期没有发生变化，并且最终获取到焦点的是 “recents_animation_input_consumer”，而非“com.tcl.android.quickstep.RecentsActivity”，说明这次“com.tcl.android.quickstep.RecentsActivity” 的启动是一次瞬态启动，transientLaunch，这个行为应该是 Launcher 那边控制的，我本地启动 “com.tcl.android.quickstep.RecentsActivity” 的话，并不是瞬态启动，获取到焦点的也是“com.tcl.android.quickstep.RecentsActivity”，而不是“recents_animation_input_consumer”，这是第二个问题点。

最终和 Launcher 的同事确认后知道，这个场景下用的非我们默认的 Launcher，而是另外一个 Launcher，类似于在 pixel 上装了一个三方 Launcher，因此点击 Recents 键（或者输入 312）会调起 RecentsActivity。

那么在联系 ANR 发生的上下文，我们已经可以知道该 ANR 发生的具体步骤了：

1、设置一个三方 Launcher 为默认 Launcher，如 NovaLauncher。

2、启动 Camera（其实任意一个 App 都行，我们分析的 ANR 场景是 Camera）：

adb shell am start -n com.tcl.camera/com.android.camera.CameraLauncher

3、输入 KEYCODE_RECENT_APPS（前提必须是 RecentsActivity 之前没有启动过）：

adb shell input keyevent 312

4、选择近期任务列表中的 Camera，即可复现无焦点窗口的情况。

5、再随便输入一个 KeyEvent，即可触发 ANR 计时：

adb shell input keyevent 98

3.1 瞬态启动 transientLaunch 和瞬态隐藏 transientHide 介绍
-----------------------------------------------

凭借我们对 transientLaunch 的了解，一个最不寻常的点就是，启动 RecentsActivity 是一次瞬态启动，但是为什么 CameraLauncher 被计算为不可见了？

首先大概说一下我个人对于这个瞬态启动的理解，我现在随便打开一个 App，比如 Camera，接着点击 Recents 键，启动 Launcher 的 Recents 界面（现在 Launcher 的 Home 界面和 Recents 界面都是同一个 Activity，不像之前点击 Recents 键后会启动另外一个界面 RecentsActivity 了。但是对于第三方的 Launcher，比如 NovaLauncher，点击 Recents 键后还是会启动 RecentsActivity，这个是 Launcher 那边的逻辑，具体的我也不是很了解），此时 Launcher 的 Recents 界面的这次启动就会被认为是 transientLaunch，瞬态启动。个人猜测加入 transientLaunch 的逻辑应该是 google 认为用户调起 Recents 界面的原因是想在 Recents 界面上选择另外的 App 进入，不会在 Recents 界面停留太长时间，因此就把调起 Recents 界面的行为定义为 transientLaunch。

相应的，transientLaunch 的特点就是，被 transientLaunch 的 TaskA 所遮挡的 TaskB，不会被认为是不可见的，即经过 transientLaunch 后，TaskA 跑到了 TaskB 的上面，但是 TaskB 还是会被认为是可见的。回到我们的例子，在 Camera 界面下点击 Recents 键启动 Launcher 的 Recents 界面，Recents 界面就会被认为是瞬态启动的，而 Camera 对应的 Task 就会被认为是 transientHide，瞬态隐藏的，也就是说它只是短暂的被 transientLaunch 的 App 遮挡了（即 Recents 界面），不应该就直接认为它是不可见的，那么它的 Activity 的生命周期也不会发生任何变化，如 log：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ff3b8ffb9874b72bea1a962c22423c4~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1328&h=218&s=51823&e=png&b=fffefe)

可以看到，只是 Launcher 的 Activity 的生命周期从 STOP 变为 RESUME，Camera 的 Activity 的生命周期并没有变化。

如果我们在 Recents 界面重新选择 Camera 回到 Camera，在整个过程中（Camera -> Recents -> Camera）Camera 的可见性和生命周期是不会发生任何变化的，减少了很多不必要的工作，因为如果没有 transientLaunch 的逻辑的话，Camera 会从可见变为不可见再变为可见，它的生命周期就会从 RESUMED -> PAUSED -> STOPPED -> STARTED -> RESUMED，而在 transientLaunch 的逻辑下，整个过程中，Camera 的 Activity 的生命周期状态一直都是 RESUMED，不会发生变化，我猜这可能就是加入 transientLaunch 的意义。

但是如果我们在 Recents 界面没有选择 Camera 进入，而是选择另外一个 App，比如 Contacts，这种情况下，Launcher 的 Recents 界面已经不在前台了，那么瞬态启动就结束了，Camera 的 Task 的可见性就会变为 false。

从上面可以看到，transientLaunch 逻辑下，我们把 Camera 的 Task 的可见性判断放在更后面的时间点，即从 Recents 界面离开的时候，而非从 Camera 界面离开进入 Recents 界面的时候：

1）、如果从 Recents 界面回到了 Camera，那么 Camera 的可见性保持为可见，即整个过程中 Messge 的可见性没有发生变化。

2）、如果从 Recents 界面进入另外的 App，如 Contacts，那么 Camera 的可见性才会从可见变为不可见。

transientLaunch 的内容大概就啰嗦这么多，接着看下它作用的地方，在以下代码，计算 Task 可见性的地方，TaskFragment.getVisibility：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c36f83d1234b41d5820658653934b200~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=903&h=417&s=63424&e=png&b=2b2b2b)

在正式计算 Task 的可见性之前，对这个 Task 进行判断，如果它被 transientHide，那么直接返回 TASK_FRAGMENT_VISIBILITY_VISIBLE，即认为瞬态隐藏的 Task 是可见的，也即这里的注释，保持 transient-hide 的根 Task 为可见，对于非根 Task 的 Task 则继续遵守一般规则。

这里判断 Task 是否是瞬态隐藏的，调用的是 TransitionController.isTransientHide：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2449535a9a454a07998949d9fe2ab49c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=739&h=197&s=18327&e=png&b=ffffff)

继续调用了 Transition.isTransientHide：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f5252100bb2b4a438d582c4ca74ff470~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=604&h=197&s=22923&e=png&b=ffffff)

Transition 的成员变量 mTransientHideTasks 定义为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50b4d1d92fb747309a51209ec4a2e55c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=869&h=391&s=48658&e=png&b=2b2b2b)

即保存了因为 transientLaunch 启动而被遮挡的 Task。

顺便看下其成员变量 mTransientLaunches，保存了瞬态启动的那个 ActivityRecord 以及 restore-below 的 Task（这个 restore-below 不知道怎么翻译，应该是和 transientHide 那个 Task 相关，“处于 transientLaunch 之下的可恢复的 Task”）。

向 Transition.mTransientHideTasks 中添加 Task 的地方只有一处，在 Transition.setTransientLaunch，同样也是唯一一处的向 mTransientLaunches 添加数据的地方：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64172e2302d84203815b8a65f52c5a66~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=792&h=547&s=61835&e=png&b=ffffff)

1）、向 Transition.mTransientLaunches 添加键值对 <ActivityRecord, Task>，这个传参 activity 就是瞬态启动的那个 ActivityRecord。

2）、如果 restoreBlow 不为 null，那么获取到传参 activity 的根 Task，然后获取到这个根 Task 的父容器，也就是 TaskDisplayArea，接着进行对 TaskDisplayArea 中的所有 Task 进行遍历，如果有 Task 请求可见，那么说明这个 Task 在瞬态启动之前是可见的，那么我们就把这个 Task 加入到 Transition.mTransientHideTasks 中，表示这个 Task 的可见性即将被瞬态启动影响，后续在 TaskFragment.getVisibility 中继续保持其为可见。

逻辑还是比较简单的，唯一要注意的是，如果传参 restoreBelow 为 null，那么我们就无法为 Transition.mTransientHideTasks 添加被瞬态隐藏的 Task，其实这里就是问题发生的原因，根据复现 ANR 的步骤去操作，这里传入的 restoreBelow 为 null，Camera 的 Task 无法被添加到 Transition.mTransientHideTasks，导致了 Camera 的 Task 无法被认为是瞬态隐藏的，所以 Camera 的相关 ActivityRecord 也被认为是不可见的。

为了知道为什么传入的 restoreBelow 为 null，我们需要分析一下这个方法的调用情况。

Transition.setTransientLaunch 方法被调用的地方也只有一处，在 TransitionController.setTransientLaunch：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92040877c7ca4a49b1d43f470921f766~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=824&h=196&s=18258&e=png&b=fffefe)

从这里的逻辑我们能看到，只有处于收集阶段的 Transition 才能记录瞬态启动相关的 ActivityRecord 以及 Task。

TransitionController.setTransientLaunch 被调用的地方有两处：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b34ff38fc8c4160ab11d4684e72be1c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1429&h=350&s=71791&e=png&b=2c2c2c)

后面又经过添加 log 以及打断点后，大概明白 Launcher 那边是如何操作的了，这里大概说明一下。

3.2 TaskAnimationManager.startRecentsAnimation
----------------------------------------------

起点在 Launcher 的 TaskAnimationManager.startRecentsAnimation：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/722354204e434e9996970e02921f6034~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=829&h=585&s=59924&e=png&b=ffffff)

首先调用 ActivityOptions.setTransientLaunch 将本次启动标记为瞬态启动：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fecd7b024e3c40b99de4b31f30229d46~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1606&h=765&s=147404&e=png&b=31302f)

这里的注释对瞬态启动也解释的很清楚了，这个方法是一个用于设置活动启动是否为瞬态操作的方法。如果设置为瞬态操作，它将不会导致现有 Activity 的生命周期更改，即使它会遮挡它们（例如，被此 Activity 遮挡的其他 Activity 将不会被 pause 或 stop，直到启动被提交）。因此，它将立即启动，因为它不需要等待其他生命周期的演变。

我们主要看这个 ActivityOptions 是如何传递的。

3.3 SystemUiProxy.startRecentsActivity
--------------------------------------

继续调用 SystemUiProxy.startRecentsActivity：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d58add5035254005b72e2adce7da35ad~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=773&h=376&s=35494&e=png&b=ffffff)

这里的 mRecentTasks 是 IRecentTasks 类型的，因此调用的是定义在 RecentTasksController.java 中的 IRecentTasksImpl 的 startRecentsTransition 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a8db29f06af4c6d852ad69cd6dcba91~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=783&h=370&s=37225&e=png&b=ffffff)

然后继续调用了 RecentsTransitionHandler.startRecentsTransition 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1761ee6947d04abc8b5a76cd8eda12c5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=761&h=536&s=51052&e=png&b=ffffff)

两个点，一是创建一个 WindowContainerTransaction 对象，调用 WindowContainerTransaction.setPendingIntent 将这个 Bundle 传入：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fcea587466e54a098387a5fc519f238a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1630&h=722&s=167792&e=png&b=323131)

二是调用 Transitions.startTransition 从 WMShell 侧发起一个 Transition。

中间过程我们就不说了，最终会走到 WindowOrganizerController.applyHierarchyOp 中。

3.4 WindowOrganizerController.applyHierarchyOp
----------------------------------------------

我们主要看对 HIERARCHY_OP_TYPE_PENDING_INTENT 这个类型的处理（对应之前调用的 WindowContainerTransaction.setPendingIntent）：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e33b32c47d7b44c4a3e3aa99086cc811~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=845&h=439&s=50389&e=png&b=ffffff)

大致的流程为：

1）、通过 ActivityStarterController.startExistingRecents 调用 TransitionController.setTransientLaunch：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33e1cc5996ae43ef88940f10c9598816~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1028&h=288&s=56122&e=png&b=fffefe)

如果 ActivityStarterController.startExistingRecentsActivity 返回了 false，那么继续调用 ActivityManagerInternal.waitAsyncStart。

2）、调用 ActivityStarter.startActivityInner，来创建 RecentsActivity 对应的 ActivityRecord 和 Task。

3）、启动 RecentsActivity 完成后，通过 ActivityStarter.handleStartResult 调用 TransitionController.setTransientLaunch：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9a96137f11a4aaca13ade8eccbd022d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1058&h=381&s=69842&e=png&b=fffefe)

接下来分别分析。

### 3.4.1 ActivityStarterController.startExistingRecents

再回顾一下我们复现问题的场景，需要保证之前 RecentsActivity 还没有启动过，log 为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d66f3e46193c4d61a0d79761a07fdeb5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1584&h=331&s=83885&e=png&b=fffefe)

看到走到 ActivityStarterController.startExistingRecents 的时候，RecentsActivity 对应的 Task 还没有创建，那么就会因为在 TaskDisplayArea 中找不到 ACTIVITY_TYPE_RECENTS 类型的 Task 而提前返回 false，不会继续调用 TransitionController.setTransientLaunch，如以下代码展示的那样：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7cdca9daa3414cd393833c2d0e2d8191~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=786&h=615&s=59433&e=png&b=fffefe)

而一旦我们启动过 RecentsActivity，那么它所在的 Task 就会存在于 TaskDisplayArea 中，后续我们再次点击 Recents 键启动 RecentsActivity 的时候就没有问题了。

出现问题的场景下，我们知道这里返回了 false，那么就会继续调用 ActivityManagerInternal.waitAsyncStart，最终是通过 ActivityStarter.handleStartResult 调用了 TransitionController.setTransientLaunch。

### 3.4.2 ActivityStarter.handleStartResult

这个流程下，是先调用 ActivityStarter.startActivityInner，创建了 RecentsActivity 对应的 ActivityRecord 和 Task。

启动 RecentsActivity 完成后，在 ActivityStarter.handleStartResult 中尝试调用 TransitionController.setTransientLaunch，log 为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7694b5f8e3294b4883e45726ea413b7d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1545&h=607&s=166961&e=png&b=fffefe)

ActivityStarter.handleStartResult 代码为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0a15c6e43e064c2e89f4075f18096f6e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=815&h=409&s=41684&e=png&b=fffefe)

根据之前的分析，这里我们知道 isTransientLaunch 的条件我们是满足的，所以会继续调用 TransitionController.setTransientLaunch，但是由于这里传入的 mPriorAboveTask 是 null，所以最终仍然无法将 Camera 对应的 Task 标记为瞬态隐藏的。

ActivityStarter.mPriorAboveTask 定义为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ba7fda8414842dc812682ad52983689~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=877&h=105&s=18706&e=png&b=2b2b2b)

注释的大概意思是，mPriorAboveTask 是启动 Activity 前的位于 targetTask（启动的这个 Activity 所在的 Task）之上的 Task，如果这个 Activity 启动在一个新的 Task 中（即 targetTask 为 null），或者 targetTask 已经处于前台了，那么 ActivityStarter.mPriorAboveTask 为 null。

再看下 ActivityStarter.mPriorAboveTask 是如何计算的，在 ActivityStarter.startActivityInner 中：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/01452d90c8ea4347920da806c72249f5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=797&h=646&s=59275&e=png&b=fffefe)

凭我们对这个方法的了解，知道了如果要启动的这个 Activity 如果是启动在一个新的 Task 中，那么这里的局部变量 targetTask 就为 null，那么就不会为 ActivityStarter.mPriorAboveTask 赋值，符合 ActivityStarter.mPriorAboveTask 的注释描述。

正好我们复现 ANR 的场景，也是 RecentsActivity 第一次启动，需要为其创建一个 ACTIVITY_TYPE_RECENTS 类型的 Task，所以在这个流程下，ActivityStarter.mPriorAboveTask 就是 null，那么传入 TransitionController.setTransientLaunch 的 restoreBelowTask 也是 null，最终也不会将 Camera 对应的 Task 标记为瞬态隐藏。

3.5 问题总结
--------

总结一下，在整个过程中，我们是有两个地方有机会将 Camera 对应的 Task 标记为瞬态隐藏的，即 WindowOrganizerController.applyHierarchyOp 方法中的这两段：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0294d2de443f40d591e6552114584a6e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=845&h=439&s=50389&e=png&b=ffffff)

但是实际上这两个地方都失败了：

1）、ActivityStarterController.startExistingRecents，需要为找到一个 RecentsActivity 找到一个 ACTIVITY_TYPE_RECENTS 类型的 Task，而 RecentsActivity 是第一次创建，所以找不到这么一个 Task，因此最终没有调用 TransitionController.setTransientLaunch。

2）、ActivityStarter.handleStartResult，如果 RecentsActivity 是第一次创建，那么不会为 ActivityStarter.mPriorAboveTask 进行赋值，那么最终传入 TransitionController.setTransientLaunch 的 restoreBelowTask 就是 null，Camera 对应的 Task 还是不会被标记为瞬态隐藏。

从以上分析能够看出，在现在的逻辑下，**如果瞬态启动的这个 Activity 是第一次启动，那么是不会将任何一个 Task 标记为瞬态隐藏的**，这个肯定是不对的，是 google 的逻辑有问题。

3.6 解决方案
--------

经过以上总结，这个问题是 google 原生问题，那么 pixel 上应该也有此问题了？

我本地在 pixel 上安装了一个 NovaLauncher 后，按照我们稳定复现 ANR 的步骤去操作，但是发现 pixel 没问题，那肯定是 google 已经修复这个问题了，反编译 pixel 的 services.jar，果然如此，在 Transition.setTransientLaunch：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/868c9ea92cd44b229bcfdb8193013132~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=891&h=512&s=59941&e=png&b=ffffff)

如果传入的 restoreBelow 为 null，那么就用传入的 activity 的根 Task，这样处理的确能保证瞬态启动后，之前可见的 Task 可以被正确标记为瞬态隐藏的。

对应的 google patch 为：

[3ceb2568736d873ab0a9ebaad40056d908662cc3 - platform/frameworks/base - Git at Google (googlesource.com)](https://link.juejin.cn/?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fframeworks%2Fbase%2F%2B%2F3ceb2568736d873ab0a9ebaad40056d908662cc3 "https://android.googlesource.com/platform/frameworks/base/+/3ceb2568736d873ab0a9ebaad40056d908662cc3")

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/df136333cbfd4a6090cde5ee1a070735~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1072&h=564&s=94364&e=png&b=fcfcfc)

单靠 Transition.java 这个修改就可以解决了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/45dcf3f891894e668d2a15c1cb394ddd~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1071&h=865&s=127076&e=png&b=ffffff)

不过保险起见，还是整个 patch 一起合入，pixel 的 services.jar 里也已经包含了整个 patch。
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7345404998165397515)

> 用户操作出的点击 Message 界面无任何响应的问题，但是也不发生 ANR。zsbdzsbdzsbdzsbd

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54a2230e1e3640c1ac79b8861c733f3c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1920&h=1080&s=2157992&e=jpg&b=12150b)

用户操作出的点击 Message 界面无任何响应的问题，但是也不发生 ANR。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbb9eb541b764dfcae257a987ff32f6d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1897&h=2777&s=740404&e=png&b=fefdfd)

与 Launcher 的同事沟通后，继续分析 startActivityFromRecents 为什么没有把 Message 启动起来。

查看 SystemLog，发现 Launcher 调用 startActivityFromRecents 方法启动 Message 后，紧接着就是以下 log：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e074f92d010848c9b2c821266d6df3ef~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1807&h=521&s=97421&e=png&b=ffffff)

是后台 Activitiy 启动的策略限制了本次启动：“BAL_BLOCK”，但是不太清楚的是为什么，这是第一个疑问点。

另外这一次启动 Message 用的 Intent 为：

intent: Intent {flg=0x1010c000 cmp=com.google.android.apps.messaging/.main.MainActivity (has extras) }; 

原因是之前启动 MainActivity 用的就是这个 Intent，这次是直接复用了。

而上一次启动 MainActivity 是在：

01-25 00:08:08.497936  1454  1636 I ActivityTaskManager: START u0 {flg=0x1010c000 cmp=com.google.android.apps.messaging/.main.MainActivity (has extras)} with LAUNCH_MULTIPLE from uid 10149 (realCallingUid=10081) (BAL_ALLOW_PENDING_INTENT) result code=0

这次应该是在 Launcher 的 Recents 启动的，因此 Intent 也是复用的。

继续往前排查，能从 log 中找到的三次启动 Message 的情况都是已经出现问题了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f247f9b43e084bed8edf156116896b57~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1823&h=207&s=64004&e=png&b=fffefe)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee3fe31b121f4d07b06ce381922ee9f5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1794&h=209&s=65504&e=png&b=fffefe)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7d2c863847b49c9919287e786c76161~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1810&h=207&s=69103&e=png&b=fffefe)

因此从目前的 log 中无法得知首次是如何启动 “com.google.android.apps.messaging/.main.MainActivity” 的，我们本地在 Launcher 界面点击 Message 图标，启动的是“com.google.android.apps.messaging.ui.ConversationListActivity”，这是第二个疑问点。

总结一下，问题中总共有两个疑问点，如果要复现这个问题，需要弄清这两个疑问点：

1、后台 BAL 策略为何限制了 Message 的启动，返回了 BAL_BLOCK。

2、“com.google.android.apps.messaging/.main.MainActivity” 是如何启动的，我们本地在 Launcher 界面点击 Message 图标，启动的是 “com.google.android.apps.messaging.ui.ConversationListActivity”。

还是先看 log：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86b53aa8e94e4db8bd73b682cce22af0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1831&h=221&s=72819&e=png&b=fffefe)

这些 log 是在 BackgroundActivityStartController.checkBackgroundActivityStart 方法中被打印的。

看 BackgroundActivityStartController.checkBackgroundActivityStart 方法代码：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/314508dde8a1435288b86f1f25f81dc9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=795&h=720&s=50280&e=png&b=fafafa)

从 log 上知道，这里传入的 originatingPendingIntent 为 null，那么局部变量 useCallingUidState 为 true，也就是说，会走这这段逻辑：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1adb0a02862d4858a1db784c2965e411~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=716&h=105&s=11837&e=png&b=fffefe)

这里应该是允许 Home App 能够启动 Activity 的，但是实际上我们却没有在这里 return，而是在该方法的最后 return 了 BAL_BLOCK，原因则可能是因为 callingPackage（callingUid）和 realCallingPackage（realCallingUid）不一致：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc31aceb505f4a9384f38420ff968c5a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1831&h=94&s=22787&e=png&b=fffdfd)

虽然真正的启动 Message 的是”com.tcl.android.launcher“，但是这里的 calingPackage 用的是” com.google.android.apps.messaging“，导致没有办法在这里 return。

再看下 callingPackage 和 realCallingPackage 都是在哪里赋值的。

由于此流程是通过 Launcher 调用 startActivityFromRecents 开始的，因此我们可以知道具体的调用堆栈为：

ActivityTaskManagerService.startActivityFromRecents

-> ActivityTaskSupervisor.startActivityFromRecents

-> ActivityStarterController.startActivityInPackage

-> ActivityStarter.execute

-> ActivityStarter.executeRequest

-> BackgroundActivityStartController.checkBackgroundActivityStart

最后发现关键点就在 ActivityTaskManagerService.startActivityFromRecents：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cfc3123b38e44b088c2f2604c2185714~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=820&h=835&s=86451&e=png&b=ffffff)

这里分析 callingUid 和 realCallingUid 可能要更简单一点，和 callingPackage 和 realCallingPackage 是一样的。

根据这里最后调用的 ActivityStartController.startActivityInPackage 方法的参数列表可知：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7590f7c4a1584e10883d66cf73295436~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=790&h=99&s=15569&e=png&b=fffefe)

callingUid 就是这里的局部变量 taskCallingUid，而 realCallingUid 为传参 callingUid。

### 4.1 realCallingUid

上面分析 realCallingUid 就是 ActivityTaskSupervisor.startActivityFromRecents 方法的传参 callingUid：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9c00523c8d14235b0203ea798f3e0f1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=675&h=32&s=5133&e=png&b=fefdfd)

而 ActivityTaskSupervisor.startActivityFromRecents 是 ActivityTaskManagerService.startActivityFromRecents 调用的：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/418ad6597731447dbd5e2a41625cb72a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=771&h=280&s=27985&e=png&b=ffffff)

而该 api 最初是 Launcher 那边调用的，因此这里的 callingUid 就是 Launcher 的 uid。

### 4.2 callingUid

如上面分析，callingUid 就是 ActivityTaskSupervisor.startActivityFromRecents 方法中的局部变量 taskCallingUid，是从 Task 的 mCallingUid 成员变量取值的：

taskCallingUid = task.mCallingUid;

而这里的局部变量 task 则是通过 RootWindowContainer.anyTaskForId 方法去取的。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2880ee9d071f4bb8aafebe6c06ccd9a2~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=819&h=35&s=6724&e=png&b=fffefe)

RootWindowContainer.anyTaskForId 方法根据传入的 taskId，从两个地方去取符合该 taskId 的 Task：

1）、如果能从现有的 Task 中找到一个符合条件的，就返回这个 Task。

2）、如果现有的 Task 都不符合条件，则从历史 Task，即 RecentTasks 中去找。

即这个 Task 是之前启动过的（不管现在是否还存在），因此如果我们想要让取得的这个 Task 的 mCallingUid 是 “com.google.android.apps.messaging”，那么就需要让这个 Task 创建的时候是由 Message 自身启动的。

如果从 Launcher 点击 Message 图标启动 Message，那 taskCallingUid 就是”com.tcl.android.launcher“：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b5d93d3591e84a58bda6da90e0749dac~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1375&h=44&s=13504&e=png&b=300a25)

这边尝试了多种启动 Message 的方式，发现通过接收到短信后点击 Notification 的方式启动 Message，可以实现 Message 的 Task.mCallingUid 对应为 Message：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2348857868e24d21b2d550ab6431519d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1505&h=63&s=18249&e=png&b=300a25)

那么实现 callingPackage（callingUid）和 realCallingPackage（realCallingUid）不一致的方法可以是：

1）、通过点击 Notification 的方式启动 Message，Task.mCallingUid 对应为 Message。

2）、点击 Back 键将 Message 销毁，然后通过 Launcher 启动 Message，并且需要让 Launcher 以复用 intent 的方式去启动 Message，比如从 Recents 界面启动。

如此一来，就可以实现 callingPackage（callingUid）和 realCallingPackage（realCallingUid）不一致，从而在启动 Message 的流程中就可以返回 BAL_BLOCK 了。

这里解决了我们在第一节提出的一疑问 1：“后台 BAL 策略为何限制了 Message 的启动，返回了 BAL_BLOCK。”

不过这种方式启动的是”com.google.android.apps.messaging.ui.ConversationListActivity“，不是我们想要的 “com.google.android.apps.messaging/.main.MainActivity”，也即疑问二：“com.google.android.apps.messaging/.main.MainActivity” 是如何启动的。

这里直接放结论吧，通过本地各种实验，发现如果通过 Phone 向联系人发送短信的方式，跳转到 Message，可以将”com.google.android.apps.messaging/.main.MainActivity“启动起来。

最后还有一点要注意：

调用 RootWindowContainer.anyTaskForId 根据传入的 taskId 寻找 Task 的时候，不能从 RootWindowContainer 中找到一个现有的 Task，而要从 RecentTasks 中找历史 Task（曾经被创建，后面从 RootWindowContainer 中被 remove 了）。如果该 Task 有一个 RootActivity，那么就不会在最后调用 ActivityStarterController.startActivityInPackage 去走 startActivity 的流程：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/248e99094c464a448a20ef36602ad56e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=738&h=158&s=23117&e=png&b=fffefe)

而是直接把该 Task 移动到前台，然后返回 ActivityManager.START_TASK_TO_FRONT：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/739fcebb355240909a17474b318cadfa~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=652&h=274&s=33684&e=png&b=ffffff)

因此为了复现问题的场景，我们需要保证走到这里的时候，Message 对应的 Task 是已经被移除了，也就是说是从 RecentTasks 中拿到了 Message 的 Task。所以即使为了实现这一步，我们也需要启动”com.google.android.apps.messaging/.main.MainActivity“，该 Activity 会在接收到 Back 事件的时候去 finish，然后整个 Task 才会被 remove 掉，后续我们就可以从 RecentTasks 中找回来。而如果启动的是”com.google.android.apps.messaging.ui.ConversationListActivity“，那么由于该 Activity 接收到 Back 事件不会走 finish，因此这里就是从现有的 Task 中拿 Message 的 Task 了，无法复现到问题。

那么最终复现该问题的稳定步骤为（导航模式为手势）：

1）、将 Message 对应的 Task 从 Recents 清除（清除之前的影响）。

2）、接受到短信，然后点击 Notification 跳转到短信（Message 自身启动自己的 Task）。

3）、上滑回到 Launcher，然后进入 Phone，通过点击 “Send a message” 的方式跳转到 Message（启动”com.google.android.apps.messaging/.main.MainActivity“）。

4）、回到 Message 后通过侧滑再次回到 Phone（类似于点击 Back，回到 Phone 的同时，Message 的 Activity 和 Task 也被移除）。

5）、通过在底部左滑的方式切换到 Message —— KO，发现界面无法点击（laucnher 调用 startActivityFromRecents 的 api，并且复用之前 Message 启动自身的时候的 Intent）。

经过以上分析可知该问题为 google 原生问题，同样在 pixel 上也可以复现。
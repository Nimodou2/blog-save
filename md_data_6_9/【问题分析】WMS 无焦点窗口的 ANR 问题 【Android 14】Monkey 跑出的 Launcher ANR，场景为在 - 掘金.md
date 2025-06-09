> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7345379864440258623)

> Monkey 跑出的 Launcher ANR，场景为在 Launcher 的 Recents 界面下一个 Activity 启动又快速销毁导致的无焦点窗口问题。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7003a91baba24965a93267dc4f8fba4c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1920&h=1080&s=2133663&e=jpg&b=1f2926)

Monkey 跑出的 Launcher ANR，场景为在 Launcher 的 Recents 界面下一个 Activity 启动又快速销毁导致的无焦点窗口问题。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/985c7b5e098047cda78e7dd3dc55007e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1608&h=1661&s=401956&e=png&b=fffefe)

根据之前的 log 分析，我们已经可以写一个 Demo App 来模拟该 ANR 的发生情况了，总结如下：

1、启动任意一个 Activity，我这里写的 Demo App 为 MainActivity。

2、接着输入事件 KEYCODE_RECENT_APPS，回到 Recents。

3、切回到 Launcher 后 100ms，马上让 MainActivity 以 NEW_TASK 的方式调起一个另外一个 Activity，我这里用一个名为 SingleTaskActivity 的 Activity 去模拟。

4、SingleTaskActivity 启动后，马上调用 finish，我这里是在 onCreate 方法中调用的 —— KO，可以复现无焦点窗口的情况，如果此时再输入一个 KeyEvent 事件，即可发生 ANR。

另外把 Demo App 安装在 Pixel 上发现没问题：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/969971ba2a8341f0897e19ab24101fb3~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1655&h=481&s=113392&e=png&b=fffefe)

有可能是 Pixel 多了一些 patch，所以修复了此问题？

多打些 log：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/95df7ef01e80439391f2fd4300cadec1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1792&h=350&s=90367&e=png&b=faf9f9)

看到及时是发生问题后，其实也是有继续调用 InputMonitor.updateInputFocusRequest 方法的，但是 "recents_animation_input_consumer" 却没有获取焦点，继续看看代码是为什么：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/925f953c86b946d489e3267854fbe4cd~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=813&h=574&s=61534&e=png&b=ffffff)

这里的 RecentsAnimationController 是之前的逻辑了，现在点击切换到 Recents，RecentsAnimationController 也是为空的，那么我们主要看这个逻辑：

```
                     
                     
                     || (getWeak(mActiveRecentsActivity) != null && focus.inTransition()
                             
                             
                             && focus.isActivityTypeHomeOrRecents()


```

看下 "recents_animation_input_consumer" 能够获取焦点的 log：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6b33bf0dbf96484fa5d8f3272195aada~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1812&h=130&s=27840&e=png&b=fffefe)

逐条看下这三个条件。

### 3.1 InputMonitor.mActivityRecentsActivity

首先看下这里的成员变量 mActivityRecentsActivity 的设置逻辑。

当在某一个非 Launcher 界面点击 Recents 时会进行设置：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f1cd9fedba9417cb809f15f723a0221~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1107&h=338&s=47404&e=png&b=fffefe)

当回到 Launcher 时则清除：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f02e4cc578f41e99fe292eeb937fabf~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=851&h=165&s=17193&e=png&b=fffefe)

### 3.2 focus.inTransition()

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/965c3886d65c4dd2aeb14fe6b46ae05e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=743&h=139&s=10813&e=png&b=fffefe)

点击 Recents 后只走到 onTransitionReady，不会走到 finishTransition，所以这个也可以满足。

### 3.3 focus.isActivityTypeHomeOrRecents()

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/105e07e3938d4da29c6412b890893895~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=757&h=72&s=6661&e=png&b=ffffff)

如果 focus 时 Launcher，这个条件肯定满足，更不用多说。

那么这里说明了，如果 “recents_animation_input_consumer” 想要取得焦点，只能从 Launcher 那边拿。如果之前获取焦点的是别的非 Home 或者 Recents 类型的窗口，那么 “recents_animation_input_consumer” 是不能拿到焦点的。

再次回看发生问题时的 log，发现阻碍 “recents_animation_input_consumer” 成为焦点的原因是走到这里时的 focus 为空：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38ed494aed12458898c73fa340fe5eba~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1799&h=421&s=108558&e=png&b=fffefe)

再看下正常的 pixel 的情况：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a48a61f3576249f28dc1edc603dd9a68~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1640&h=241&s=60264&e=png&b=fffefe)

发现是在 finishActivity 的流程下，将窗口焦点从 null 切换到了 Launcher，接着 “recents_animation_input_consumer” 就请求了焦点。

再看有问题的情况，发现 finishActivity 流程下也是去更新了窗口焦点：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b9ec91acb8c4dadabd3a71c929bbf33~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1831&h=770&s=226178&e=png&b=fffefe)

但是 Launcher 的 isVisibleRequestedOrAdding 返回了 false，所以不能取得焦点。

所以问题还是回到了 DisplayContent.mCurrent 为 null，即焦点窗口丢失。

再看下问题 Activity，如之前所说，我这里模拟用的是 “SingleTaskActivity”，快速启动又销毁的流程的 log 信息：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c96249f1ad54203a1f4300f115cf624~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1812&h=771&s=196216&e=png&b=fdfcfc)

看到这个 Activity 走 finish 流程的时候，该流程是有重新去更新焦点窗口的，但是此时由于此时 “Window{TclQuickstepLauncher}” 的 isVisibleRequestedOrAdding 仍然为 false，所以不满足作为焦点窗口的条件，因此无法将焦点窗口切换为“Window{TclQuickstepLauncher}”。

而当后续 “SingleTaskActivity” 走 pause 流程的时候，“TclQuickstepLauncher”的可见性才被设置为 true，但是后续没有再调用 DisplayContent.updateFocusedWindowLocked 去更新焦点窗口了，所以这之后即使 “TclQuickstepLauncher” 满足了 canReceiveKeys 的条件，也无法变为焦点窗口了。

那为什么 pixel 没问题呢，看下 pixel 的情况，发现从发送 keyEvent=312 之后，“NexusLauncherActivity”的可见性始终都没有改变过，所以 “NexusLauncherActivity” 一直都可以满足 canReceiveKeys 的条件。

而我们的情况则不同，“TclQuickstepLauncher” 在它的 completePause 的流程中设置了可见性为 false：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/727ef7f1a6c944bcabda6f4cd0bd58fb~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1767&h=166&s=50146&e=png&b=fbfafa)

再看 “TclQuickstepLauncher” 为什么会被 pause：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d5a3064970d4e7ab30ed425b42dc596~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1809&h=263&s=74214&e=png&b=fffdfd)

从 log 看，是 “SingleTaskActivity” 被启动后，在 TaskDisplayArea,pauseBackTasks 中被 pause：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5fdffb36b7e44f438249ed09e41ea733~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=840&h=454&s=50113&e=png&b=fefdfd)

被 pause 的原因则是 Launcher 对应的 Task，调用 canBeResuemd 方法返回了 false，所以这里认为该 Task 中的 Activity 应该被 pause。

TaskFragment.canBeResumed 的方法为：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b189eb617354000ac6281d1ae3f4233~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=744&h=168&s=15612&e=png&b=fefdfd)

主要的逻辑就是调用 TaskFragment.getVisibility 去获取当前 TaskFragment 的可见性。

另外看 pixel 的情况，反而是 “MainActivity” 被 pause，Launcher 没有被 pause：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/559ecf5e36f647649b7d54ee86962981~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1734&h=116&s=41792&e=png&b=fffefe)

说明 pixel 的代码和我们的代码在 TaskFragment.getVisibility 上有差异？拿从集成那边的最新基线比较我们的代码，很快发现了差异点：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/045c08c2a1754eaaa9b3bbb081895ce5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1675&h=181&s=34511&e=png&b=2b2b2b)

打入该 patch 后问题不再复现。

在问题 log 中还看到有以下 log：

01-26 13:58:02.241 1543 3349 I InputDispatcher: Dropping event because all windows would just receive ACTION_OUTSIDE: MotionEvent

虽然该 ANR 并不是有 MotionEvent 触发，但是还是看下这个 log 打印的场景。

写一个绘制需要 15000ms 的 Activity2，让 Activity1 启动 Activity2 的时候 finish，此时点击屏幕，即输入 MotionEvent，有三个阶段：

1、首先是 Activity1 还没有被 stop 的时候，此时屏幕上仍然可以看见 Activity1，但是 Activity2 正在启动，导致了 Activity1 无法接收事件，因此这里的 log 是：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/61d3fa50f8434626a5d9150d3daf2506~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1833&h=197&s=40473&e=png&b=fffefe)

02-27 17:52:36.560 1473 1654 I InputDispatcher: Dropping event because all windows would just receive ACTION_OUTSIDE: MotionEvent

2、接着 Activity1 被 stop，Activity2 也还没有起来，此时屏幕上是黑屏显示：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/120483c0d7f8433f83e4450dba28868c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1825&h=168&s=33227&e=png&b=fffdfd)

此时的 log 为：

02-27 17:52:39.214 1473 1654 I InputDispatcher: Dropping event because there is no touchable window at (423.4, 734.5) on display 0.

可以看到，虽然任何窗口都无法接收该 MotionEvent，但是此时并不会触发 ANR 的计时，也就是 MotionEvent 在此场景下不会导致 ANR？

3、最后我们再手动输入一个 KeyEvent，此时 log 为：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/753e45149bfd4906a0ff58ee042e20fd~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1841&h=100&s=22234&e=png&b=fefdfd)

看到此时才真正触发了 ANR 的倒计时：

02-27 17:52:48.246 1473 1654 W InputDispatcher: Waiting because no window has focus but ActivityRecord{e0a1ba7 u0 com.example.demoapp/.LongDrawActivity t479} may eventually add a window when it finishes starting up. Will wait for 5000ms
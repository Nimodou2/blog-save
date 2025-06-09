> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7345379864440275007)

> Monkey 跑出的 Launcher ANR，场景为在 Launcher 界面，下拉状态栏，然后点击 Notification，连续启动多个 Activity，这些 Activity 均是启动后又快速销毁，导致的后

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9d1c6086a3a341a7a90e94fbc775d5e6~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=3500&h=1969&s=3893680&e=jpg&b=131918) Monkey 跑出的 Launcher ANR，场景为在 Launcher 界面，下拉状态栏，然后点击 Notification，连续启动多个 Activity，这些 Activity 均是启动后又快速销毁，导致的后续无焦点窗口问题。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eabb01cdaf5744a090e0ed82d0af62e5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1574&h=762&s=171788&e=png&b=fffefe)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d18be4fae1cf43f5ab7a7cd14929f039~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1599&h=549&s=127702&e=png&b=fffdfd)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/867ebc73526a4cfc89b50c465abf75b5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1594&h=838&s=200270&e=png&b=fffefe)

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/95230bba1a6c44c095b95dd7bd636678~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1600&h=204&s=34723&e=png&b=fffefe)

值得关注的异常的点是，从 NotificationShade 上点击 Notification 启动 “com.google.android.setupwizard/.predeferred.PreDeferredSetupWizardActivity” 后，又连续启动了多个 Activity，顺序分别为：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/569f53217f8446198db140d82ddc2aba~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1527&h=520&s=96674&e=png&b=fffefe)

com.google.android.setupwizard/.predeferred.PreDeferredSetupWizardActivity

-> com.google.android.setupwizard/.predeferred.ConnectToWifiActivity

-> com.google.android.setupwizard/.WizardManagerActivity

-> com.google.android.setupwizard/.SetupWizardExitActivity

从 log 中能看到，这 4 个 Activity 均是在 create 之后马上就启动另外一个 Activity，然后马上 finish 了，因此整个过程中实际上是没有任何 “com.google.android.setupwizard” 相关窗口添加的。并且最终的 Activity 是 finish 了整个 Task 从而回到了 Launcher。

后续 Launcher 重新变为焦点 Application 的时候，应该是在 WMS.relayoutWindow 中，Launcher 的相关 WindowState 可见性没有发生变化，从而无法去更新焦点窗口。

尝试写一个 Demo 去复现这个 ANR，但是发现无法复现这个 ANR，原因有两个：

1、我们的 Demo 在创建 Task 的时候会启动 SplashScreen，这个 SplashScreen 会影响 Launcher 的可见性，后续 Launcher 走 WMS.relayoutWindow 的时候，WMS 可以去更新焦点窗口，这一点可以通过修改代码去不让 SplashScreen 启动。

2、问题 log 中，看到 NotificationShade 在点击 Notification 后马上就隐藏了，而点击 Demo App 发起的 Notification，NotificationShade 过了很久才隐藏。

整个过程的流程为：

1）、点击 Notification。

2）、启动多个 “快速启动又快速销毁” 的 Activity。

3）、当这些 Activity 都销毁后，重新回到 Launcher。

发生 ANR 的 log：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ead08d831ddf4a3393a90a22ec309d3c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1536&h=258&s=65380&e=png&b=fffefe)

能看到在点击 Notification 的时候，NotificationShade 就走了 WMS.relayoutWindow，此时才启动到第一个 “com.google.android.setupwizard” 的 Activity。由于此时刚刚启动了 “com.google.android.setupwizard” 的 Activity，因此 Launcher 的相关 ActivityRecord 可见性受到了影响变为了不可见，导致 Launcher 没有办法作为焦点窗口，因此此时 Launcher 没有办法作为焦点窗口，所以焦点窗口时从 NotificationShade 变为了 null。

这个 NotificationShade 的 WMS.relayoutWindow 这么早就调用，应该是和 NotificationShade 的提前隐藏有关系的（NotificationShade 下拉的时候去掉了 FLAG_NOT_FOCUSABLE，隐藏的时候重新加上去），看问题 log，从 “notification_clicked” 到“notification_panel_hidden”，只有 52ms。

而我们本地的 Demo 情况为：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a02b0b905d0649c88b49b40a257a60ef~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1560&h=351&s=83615&e=png&b=fffdfd)

能看到点击了 Notification 后，NotificationShade 没有马上隐藏，也就是说 NotificationShade 此时还没有去走 relayoutWindow，而是等到四个 “com.google.android.setupwizard” 都启动完并且销毁后，NotificationShade 才隐藏，而此时 Launcher 的 ActivityRecord 的可见性重新变成了可见，此时 NotificationShade 再去 relayoutWindow，因此此时焦点窗口就可以从 NotificationShade 转移到 Launcher。

因此整个过程可以总结为：

1）、点击 Notification 启动 “com.google.android.setupwizard” 的 Activity，“ActivityRecord{Launcher}”变为不可见。

2）、“com.google.android.setupwizard” 的所有 Activity 被销毁后，“ActivityRecord{Launcher}” 重新变为可见。

其中 NotificationShade 只会 relayout 一次：

如果是在 “com.google.android.setupwizard” 的所有 Activity 被销毁前就执行，那么此时 “ActivityRecord{Launcher}” 还是不可见的，因此此时更新焦点窗口，“WindowState{Launcher}”不满足 canReceiveKeys 的条件，因此此时找不到任何窗口可以满足作为焦点窗口的条件，焦点窗口就从 “NotificationShade” 变为了 null，并且后续即使 “com.google.android.setupwizard” 的所有 Activity 被销毁后 Launcher 满足了条件，后续也没有再去更新焦点窗口了，焦点窗口一直都是 null，这也是问题 log 中的情况

如果是在 com.google.android.setupwizard”的所有 Activity 被销毁后再去执行，那么 Launcher 就可以作为焦点窗口，焦点窗口就从 “WindowState{NotificationShade}” 变为 WindowState{Launcher}”，正常，这就是我写的 Demo App 的情况。

所以重要的就是 NotificationShade 走 WMS.relayoutWindow 的时机，下一步需要看下为什么我们的 Demo 会和发生问题的 “com.google.android.setupwizard” 有这种差异，如果我们的 Demo 和 “com.google.android.setupwizard” 在这个点上行为一致，那么应该就可以复现这个 ANR。

上一节知道了问题的关键在于 NotificationShade 走 WMS.relayoutWindow 的时机，而点击 Notification 后，NotificationShade 是会收起的，应该就是这个收起的动作触发了 WMS.relayoutWindow，从 log 上看就是 “notification_clicked” 和“notification_panel_hidden”。

这一节分析为什么问题 log 中的 “notification_clicked” 到“notification_panel_hidden”，只有 52ms，而我写的 “notification_clicked” 到“notification_panel_hidden”至少都有 400ms。

首先看下这几个 log 打印的地方分别在：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6902ff19ddf7496e9e94333390788732~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=945&h=111&s=16176&e=png&b=214283)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2fc9146685f84f8abeb5b18119d48af3~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=810&h=228&s=35778&e=png&b=2b2b2b)

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc905f4c82e74ff0b84fd90ac73def83~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=757&h=153&s=16776&e=png&b=2c2c2c)

都是 SystemUI 那边调用过来的，因此继续在 SystemUI 打堆栈，主要是隐藏 NotificationShade 的这块：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b783c98be8d46b3abbe8525a8f40154~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1102&h=290&s=41825&e=png&b=2b2b2b)

这里我看到点击 “USB debugging connected” 这个 Notification 的 log 和问题发生时的 log 时一样的，因此拿这个 Notification 和我们的 Demo App 发起的 Notification 进行对比，该 Notification 在 AdbNotifications 中创建。

很快发现了差异点：

AdbNotifications 的情况是：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/65ec2a2a22414b83b7951cdf8847995e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1812&h=678&s=195066&e=png&b=fecb98)

可知，一点击相关 View，就进行了折叠下拉状态栏的操作。

同时 onNotificationClicked 也是在这里：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/94021453746b4e3f98080523c9bba864~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1826&h=453&s=121407&e=png&b=98fefb)

这两个几乎是同时的。

而我们的 Demo App 下，折叠下拉状态栏则是在很后面的：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3dadd93fcb9546dbaa602009824cdf83~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1388&h=309&s=82551&e=png&b=fdca98)

并没有说一点击 Notification 就进行折叠。

继续打印相关 log，发现原来是点击 Notification 后，SystemUI 会继续判断需不需要为折叠下拉状态栏这个过程加一段动画，如果不用，那么就直接隐藏，就像 AdbNotifications 那样，如果需要动画，那么就会等到动画结束后再去隐藏，也就是我们 Demo App 的情况。

区别在于：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc237779d0e040ae99f07c1c0fbf88e7~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=993&h=475&s=54706&e=png&b=fffefe)

这里的 PendingIntent.queryIntentComponents 最终调用了 ComputerEngine.queryIntentActivitiesInternal，那么继续去这个方法里面加 log，发现了差异：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6c7db74bb0f4550902ab00a682b788c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1846&h=432&s=86289&e=png&b=fffefe)

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c091f75c853449cdacf5b43623ba7e9a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=762&h=253&s=22529&e=png&b=ffffff)

AdbNotifications 的情况下，传入的 userid 为 - 2，而 Demo App 传入的 userId 为 0，因此有了差异。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8aa3d12ab9bb411bb5040d55ca057fb5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=731&h=61&s=14032&e=png&b=2e2e2e)

首先 PendingIntent.getActivityAsUser 是 hide 的 api，我们只能用反射去调用这个 api。

其次我在使用 UserHandle.CURRENT 的时候，还会报以下错：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8af0bb38d8014f78bfdb8a8d31e8d72a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1898&h=193&s=64305&e=png&b=300a25)

缺少了 android.permission.INTERACT_ACROSS_USERS_FULL 或者 android.permission.INTERACT_ACROSS_USERS 权限，添加了这个权限后又提示这个权限只能给系统 App 用：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0683e3d7318c41a6b35df145fc32c640~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1191&h=112&s=27268&e=png&b=2e2e2e)

后面在网上又找了找资料，发现可以使用以下命令来给 App 赋予权限，试了一下果然有用：

```
 adb shell pm grant 


```

最终，我可以在我们的机器上复现这个 ANR 了。

但是还有一个问题，之前提到，最初尝试复现这个 ANR 的时候，其实是有两个困难的，第二个困难即点击 Notification 的时候让下拉状态栏能够快速收起这个问题我们已经解决了，但是第一个问题，点击 Notification 的时候启动新的 Task 会创建 SplashScreen，这个我们之前是修改 framework 代码强制不让 SplashScreen 启动，但是如果我们想要在 pixel 上复现这个 ANR，又该如何去做？

这个其实也是我的一个困惑，之前遇到过多次在 Launcher 界面，一个临时 Activity 启动又马上销毁的情况，这种情况下：

1、临时 Activity 启动，影响了 “ActivityRecord{Launcher}” 的可见性，导致 “WindowState{Launcher}” 不再满足 WindowState.canReceiveKeys 的条件，焦点窗口从 “WindowState{Launcher}” 转移到了 null。

2、接着临时 Activity 又马上销毁，比如在其 onCreate 方法中就调用 finish 方法去销毁，此时这个 Activity 的窗口还没有添加，因此 “WindowState{Launcher}” 的 mViewVisibility 成员变量其实是没有受到影响的，仍然是可见的，View.VISIBLE。

3、临时 Activity 销毁后，在我复现的大部分 ANR 的问题中，“WindowState{Launcher}” 此时都会去走 WMS.relayoutWindow 流程的，该流程中有机会去更新焦点窗口，但是条件是 focusMayChange 要为 true：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6502e7b6925452a9c556a96cc05b1ab~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=827&h=178&s=15481&e=png&b=ffffff)

从以上代码可知，focusMayChange 要为 true，需要满足以下三个条件之一：

1）、“WindowState{Launcher}”的 mViewVisibility 要发生改变。如果临时 Activity 在启动的时候，创建了 SplashScreen，那么 SplashScreen 的窗口是会被添加到屏幕上且显示的，这会影响到 WindowState{Launcher}”的可见性，也就是说在创建 SplashScreen 的情况下，当 “WindowState{Launcher}” 走 WMS.relayoutWindow 的时候，focusMayChange 会被置为 true，后续会去更新焦点窗口，让 “WindowState{Launcher}” 重新变为焦点窗口。但是如果没有创建 SplashScreen，那么大概率，这里不会去更新焦点窗口，导致后续焦点窗口一直为 null。

~2）、窗口的 flag 中的 FLAG_NOT_FOCUSABLE 发生变化，这种情况一般见于 NotificationShade，下拉状态栏的时候去掉这个 flag，上滑隐藏的时候重新加上，对于 Launcher 来说一般不会。~

~3）、WindowState 的成员变量 mRelayoutCalled 为 false，即这个 WindowState 之前还没有调用过 WMS.relayoutWindow 方法，注意我们的情况是之前 Launcher 已经显示了，所以肯定也是调用过这个方法了，因此这种情况也不太会出现。~

综上分析，大部分这种情况下的 ANR，出现的原因就是 “WindowState{Launcher}” 的可见性，即 WindowState.mViewVisibility，没有发生变化，导致后续的更新焦点窗口的方法没有被调用。并且如我们上面所说，如果启动了临时 Activity 的时候创建了 SplashScreen，是可以让 “WindowState{Launcher}” 的可见性发生变化的，但是大部分 ANR 发生的情况，即使是新建了一个 Task，也没有创建 SplashScreen，从而导致了后续焦点窗口缺失的情况。因此问题的关键在于，为什么发生 ANR 的时候，新建了一个 Task，却没有创建 SplashScreen？如果搞清楚了这个问题，应该对我们解决这种类型的 ANR 很有帮助。

在网上查询了一下去掉 SplashScreen 的方式，并且使用了一下果然可以：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42e60725cb2942518b26e559f6c5880b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=656&h=252&s=33057&e=png&b=2b2b2b)

为 Activity 设置 Theme.NoDisplay 的主题。

Theme.NoDisplay 的定义为：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/31896824d0104dfd90cfc43a4a75730a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=680&h=274&s=46800&e=png&b=2c2c2c)

关键应该就是这个 windowDisablePreview：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f304fa257b7c484180b73fb56f80ec21~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=827&h=114&s=18766&e=png&b=2b2b2b)

这个属性使用的地方在 ActivityRecord.validateStartingWindowTheme：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5da7f87e95bd456b9496a2ad55097d81~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=784&h=634&s=67562&e=png&b=fffefe)

如果 ActivityRecord.validateStartingWindowTheme 返回 false，说明不希望添加启动窗口。

这里我没有再打 log 去跟，但是看了下调用关系，这里的 prev 应该是 null，所以在 windowDisableStarting 为 true 的情况下，ActivityRecord.validateStartingWindowTheme 就返回了 null。

后续在 ActivityRecord.addStartingWindow 中，在以下代码处返回了：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c6da227ab23b4ead98cb5feb1e9df4fc~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=852&h=162&s=18179&e=png&b=2b2b2b)

从而没有添加 StartingWindow。

另外正常添加 StartingWindow 的堆栈调用如下：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f3a7a43132843e5aa49121b22a218f5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=672&h=373&s=58954&e=png&b=3c3f41)

最终我们就可以用 Demo App 在 pixel 上复现 ANR 了：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/40529b52164343a892be55a80f7dd26e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1891&h=225&s=58014&e=png&b=300a25)

另外和复现 log 中 “com.google.android.setupwizard” 的行为不太一样的是，我反编译了 “com.google.android.setupwizard” 这个 apk，发现它的 activity 似乎不是通过设置 Theme.NoDisplay 的方式来不让 SplashScreen 显示的，具体是怎么做的还不清楚，但是应该关系不大，我们只要能够复现这种场景就行了，具体怎么实现的可以有多种方式。

最后还有一点是，点击 Notification 后，下拉状态栏，也即 NotificationShade 会收起，此时 NotificationShade 会有两个变化，一是为其窗口设置 FLAG_NOT_FOCUSABLE 这个 flag，第二个是其窗口的可见性，WindowState.mViewVisibility 会发生改变，任意这两项改变，都会触发 WMS.relayoutWindow，且都会在 WMS.relayoutWindow 中触发焦点窗口的更新，即：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1f3cdbfd15f49d0afe9f64f3247d2f8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=578&h=85&s=18480&e=png&b=2b2b2b)

这个我们之前也分析过了。

后面经过我们在 pixel 上的测试，发现这两个动作是有先后的顺序的，并不是一起改变的，从复现的情况来看，FLAG_NOT_FOCUSABLE 这个 flag 的添加要先一点。

原生机的情况为：

1、点击 Notification。

2、FLAG_NOT_FOCUSABLE 被添加，触发第一次 NotificationShade 的 WMS.relayoutWindow。

3、大概过了 400ms，NotificationShade 从可见变为了不可见，触发了第二次 NotificationShade 的 WMS.relayoutWindow。

那么我们如果要复现 ANR，就要保证在第二次 NotificationShade 走 WMS.relayoutWindow 的时候，Launcher 仍然是不可见的，即仍然有我们的 Demo App 的 Activity 在启动。

这一点导致的区别是：

像我们的手机，这两次 relayoutWindow 相距不到 200ms，所以我连续启动 4 个 Activity 就可以保证在第二次 relayoutWindow 的时候 Launcher 是不可见的。

但是在 pixel 上，我要启动十几个 Activity 才能保证第二次 relayoutWindow 的时候 Launcher 仍然是不可见的，后续才可以复现这个 ANR。

这些可能是性能方面的影响或者 SystemUI 有差异，但是仍然可以说明这个 ANR 是个原生 bug。
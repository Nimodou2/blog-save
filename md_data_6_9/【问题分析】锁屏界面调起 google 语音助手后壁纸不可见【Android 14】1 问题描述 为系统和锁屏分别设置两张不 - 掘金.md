> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7367229542933250102)

> 1 问题描述 为系统和锁屏分别设置两张不同的壁纸，然后在锁屏界面长按 Power 调起 google 语音助手后，有时候会出现壁纸不可见的情况，如以下截图所示： 有的时候又是正常的，但显示的也是系统壁纸，并非

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d38be538cf0d477ba4c75bedef02d361~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1920&h=1080&s=2504189&e=jpg&b=201704)

为系统和锁屏分别设置两张不同的壁纸，然后在锁屏界面长按 Power 调起 google 语音助手后，有时候会出现壁纸不可见的情况，如以下截图所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e582eddb00004cffb94445cf843a94e2~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=359&h=828&s=43684&e=png&b=080808)

有的时候又是正常的，但显示的也是系统壁纸，并非是锁屏壁纸。

后面我本地多次尝试，发现了一些规律：

1）、同时设置系统和锁屏壁纸为壁纸 A，此时不会有问题。

2）、在第 1 步的基础上，单独将锁屏壁纸设置为壁纸 B，此时也不会有问题。

3）、在第 2 步的基础上，单独将系统壁纸设置为壁纸 C，出现问题。

添加了 log 后，大概知道问题出现的原因了，不过这里先分析一下 WallpaperController 中关于计算壁纸可见性的一些代码逻辑吧，之前零零散散看过一点，希望这次能够趁分析这个问题的机会，做一些总结。

2.1 WallpaperController.adjustWallpaperWindows
----------------------------------------------

更新壁纸的起点在 WallpaperController.adjustWallpaperWindows：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aed36c5c083a403ba7a144094629e661~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=748&h=537&s=58481&e=png&b=ffffff)

1）、WallpaperController.findWallpaperTarget 用来寻找壁纸的目标窗口，将寻找的结果放到成员变量 WallpaperController.mFindResults 中。

2）、WallpaperController.updateWallpaperWindowsTarget 根据成员变量 WallpaperController.mFindResults 更新成员变量 WallpaperController.mWallpaperTarget。

3）、最后如果 WallpaperController.mWallpaperTarget 不为空，那么认为壁纸可见，再调用 WallpaperController.updateWallpaperTokens 更新壁纸的可见性。

这里有两个成员变量 mFindResults 和 mWallpaperTarget 要先介绍一下。

首先是 mWallpaperTarget，定义为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21b2b85173584b8586b2560dd54c3c87~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=665&h=111&s=15171&e=png&b=2b2b2b)

白话点说就是壁纸的目标窗口，比如一个 Activity 的窗口在显示的时候被设置为支持壁纸显示，比如 Launcher 的窗口，那么这个窗口就可以作为壁纸的目标窗口。如果我们遍历所有的窗口后，找不到一个窗口可以作为壁纸的目标窗口，那么就说明所有的窗口都不支持壁纸显示，那壁纸也就会被设置为不可见。如果可以找到一个壁纸的目标窗口，那么这个目标窗口就会被保存到成员变量 WallpaperController.mWallpaperTarget 中。

接着是成员变量 mFindResults，定义为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bcc333aba5854174b1f24fae4416fe6a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=805&h=72&s=11557&e=png&b=2b2b2b)

它是一个 FindWallpaperTargetResult 类型的成员变量，这里则需要知道 FindWallpaperTargetResult 这个类的作用，它定义在 WallpaperController 里：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c57088c400a4a519990a4f2083af605~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=671&h=318&s=20526&e=png&b=fffefe)

从类的注释上以及类名来看，这个类是用来保存寻找壁纸目标窗口操作的结果。

再看它的成员变量，首先是 TopWallpaper 类型的 mTopWallpaper，它又是定义在 FindWallpaperTargetResult 中的内部类，只有两个成员变量，mTopHideWhenLockedWallpaper 和 mTopShowWhenLockedWallpaper，结合这里的注释以及我看了代码后的理解：

mTopHideWhenLockedWallpaper 和 mTopShowWhenLockedWallpaper 都可以设置为壁纸窗口，即 “Window{d837259 u0 com.android.systemui.wallpapers.ImageWallpaper}”，并且同一时间它们两个中间只能有一个被设置：

1）、如果 mTopHideWhenLockedWallpaper 被设置（即不为空），说明此时壁纸只能在 Home 界面可见，锁屏界面不可见。

2）、如果 mTopShowWhenLockedWallpaper 被设置（即不为空），说明此时壁纸在 Home 界面以及锁屏界面均可见。

后面分析到相关代码的时候就能了解上面的意义了。

然后再解释几个后续会分析到的成员变量：

1）、mNeedsShowWhenLockedWallpaper，如果在一个 Activity 界面可以在锁屏界面上显示，比如通话界面，如果这个 Activity 或者窗口不是全屏的，那么就会把 mNeedsShowWhenLockedWallpaper 的值设置为 true。在这种场景下，即使我们没有为锁屏壁纸找到一个目标窗口，那么我们可能也是需要将锁屏壁纸显示出来的。

2）、useTopWallpaperAsTarget，结合上面第一点来说，如果后续经过我们遍历所有窗口后，我们找不到任何一个窗口可以作为壁纸的目标窗口，但是在一些特殊场景下，我们又是需要将壁纸显示出来的，那么我们就将这个值设置为 true，表示我们将 TopWallpaper 中的 mTopHideWhenLockedWallpaper 或者 mTopShowWhenLockedWallpaper 保存的壁纸 WindowState 本身作为壁纸的目标窗口，至于从这两者中的哪个里面取，则是看当前是否处于锁屏。

3）、wallpaperTarget，这个没什么好说的，在寻找壁纸的目标窗口阶段，我们将寻找的结果保存在 FindWallpaperTargetResult.wallpaperTarget 中，后续在 WallpaperController.updateWallpaperWindowsTarget 中，我们就从 FindWallpaperTargetResult.wallpaperTarget 中拿寻找的结果。

接下来看寻找壁纸的目标窗口的代码，WallpaperController.findWallpaperTarget。

2.2 WallpaperController.findWallpaperTarget
-------------------------------------------

这个方法用来寻找壁纸的目标窗口，是我们本篇文章的分析重点。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e579fb3debd4ab1af9abbec448e43fe~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=822&h=362&s=48643&e=png&b=fffefe)

### 2.2.1 FindWallpaperTargetResult.reset

调用 FindWallpaperTargetResult.reset 重置 FindWallpaperTargetResult 的保存的所有信息：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ada9104dfabb4662aec4594f695ae42d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=410&h=183&s=21562&e=png&b=2b2b2b)

### 2.2.2 Freeform 的情况

如果当前有 WINDOWING_MODE_FREEFORM 类型的 App 显示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e8584e1121b3467e904e355af7a6be9d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=775&h=184&s=36425&e=png&b=2b2b2b)

那么就设置 FindWallpaperTargetResult.useTopWallpaperAsTarget 为 true：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a9398c461044cba87ebaca7f32efa62~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=609&h=86&s=15729&e=png&b=2b2b2b)

即在 Freeform 的场景下直接将壁纸进行显示，不需要再额外找一个目标窗口了，这算是一种对需要显示壁纸的特殊场景的处理。

### 2.2.3 WallpaperController.mFindWallpapers

接着对所有的窗口进行第一次遍历：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d51de0ddf0e84882a3fcd5cc8f4077f8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=775&h=185&s=21292&e=png&b=ffffff)

如果这个窗口是壁纸类型的，那么继续判断，如果 WallpaperWindowToken.canShowWhenLocked 返回 true，说明此时壁纸是允许在锁屏界面显示的，那么就将这个壁纸窗口保存在 FindWallpaperTargetResult.mTopShowWhenLockedWallpaper 中，后续如果我们检测到 FindWallpaperTargetResult.mTopShowWhenLockedWallpaper 不为空，那么就说明当前壁纸是允许在锁屏界面显示的。否则就将这个壁纸窗口保存在 FindWallpaperTargetResult.mTopHideWhenLockedWallpaper 中，后续如果我们检测到 FindWallpaperTargetResult.mTopHideWhenLockedWallpaper 不为空，那么就说明当前壁纸是只能在 Home 界面显示的。

另外根据我本地的测试，当同时设置系统壁纸和锁屏壁纸时，WallpaperWindowToken.setShowWhenLocked 这个方法会被调用，设置 WallpaperWindowToken.mShowWhenLocked 为 true，调用堆栈为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/030202c489d648d9b406a648bf395cba~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1126&h=352&s=62507&e=png&b=fffefe)

### 2.2.4 WallpaperController.mFindWallpaperTargetFunction

对所有的窗口进行第二次遍历：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/428ed87f27c7447cbbd4b1eeb59d5c45~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=823&h=555&s=67794&e=png&b=fefdfd)

这里的逻辑也比较复杂，省略不太重要的部分，首先看一种特殊的情况：

如果在锁屏状态，并且此时正在遍历的这个窗口盖在锁屏界面，那么继续判断：

1）、如果现在锁屏的状态为 “occluded”。

或者

2）、当前该在锁屏界面上的那个窗口处于 Transition，那一般就是 open 或者 close。

该窗口或者该窗口对应的 ActivityRecord 是否是全屏的，如果不是，那么将 mNeedsShowWhenLockedWallpaper 设置为 true。很好理解，如果是一个非全屏的窗口盖在锁屏界面上，如果不显示锁屏壁纸，那么屏幕上没有被这个非全屏窗口覆盖的部分就会由于没有内容显示从而黑屏。

这里稍微提一下这个对窗口是否处于 Transitiond 的判断，如果只是判断锁屏的状态为 “occluded”，那么可能会出现锁屏状态切换为“occluded” 不够及时，从而出现短暂黑屏的现象，就比如我这里长按 Power 键唤起 google 语音助手的情况：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6f6bfe08b714b2c8f9d71002a875023~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1259&h=932&s=722383&e=png&b=2f2f2d)

所以我们需要加上对窗口是否处于 Transition 的判断，确保壁纸在 Transition 早期阶段就显示。

判断过这种特殊场景后，接着对这个正在遍历的窗口进行判断，如果这个窗口在屏幕上，并且已经绘制完成了，那么调用 WindowState.hasWallpaper 方法去判断该窗口是否支持显示壁纸（这里就不分析动画过程中显示壁纸的情况了）：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/39958bb68c744d33b8b8d5446febba75~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=862&h=92&s=17242&e=png&b=2b2b2b)

这是更一般的情况。

涉及 LetterBox 的情况比较少见，最常见的还是通过检查窗口是否配置了 FLAG_SHOW_WALLPAPER 这个窗口标志位来判断这个窗口是否支持显示壁纸，就比如 Launcher。

### 2.2.5 FindWallpaperTargetResult.setUseTopWallpaperAsTarget

回到 WallpaperController.findWallpaperTarget 方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a48f1c93c091470db931df0885bc0847~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=778&h=105&s=17618&e=png&b=2b2b2b)

这里承接第 4 步，如果在第 4 步我们发现一个非全屏的窗口盖在了锁屏界面上，那么就会将 FindWallpaperTargetResult.mNeedsShowWhenLockedWallpaper 设置为 true。

接着在这里，就调用 FindWallpaperTargetResult.setUseTopWallpaperAsTarget 设置 FindWallpaperTargetResult.useTopWallpaperAsTarget 为 true：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f4b6c120e0744b7ba18e90cb8454b6c1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=609&h=86&s=15729&e=png&b=2b2b2b)

来保证后续壁纸可以显示，这个场景和 Freeform 出现的场景处理方式一致，即对需要显示壁纸的特殊场景的一种处理。

### 2.2.6 FindWallpaperTargetResult.setWallpaperTarget

来看最后的一点内容：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e4ffbe5814d74702b7c88e94dd04621c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=773&h=106&s=19152&e=png&b=2b2b2b)

如果 FindWallpaperTargetResult.wallpaperTarget 为空，说明通过两次遍历我们没有找到壁纸的目标窗口，但是 FindWallpaperTargetResult.useTopWallpaperAsTarget 为 true，又说明我们的确想显示壁纸，那么就调用 FindWallpaperTargetResult.getTopWallpaper 看看能不能返回一个壁纸窗口，如果可以，那么就调用 FindWallpaperTargetResult.setWallpaperTarget 将壁纸的目标窗口设置为壁纸本身。但是也可能会返回 null，看下 FindWallpaperTargetResult.getTopWallpaper 的内容：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/454ecb0fa9374d24aba2193597774114~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=548&h=118&s=9960&e=png&b=ffffff)

前面说过了，如果将壁纸窗口保存在 mTopHideWhenLockedWallpaper 中，说明当前壁纸只能在 Home 界面显示，不能在锁屏界面显示。如果将壁纸窗口保存在 mTopShowWhenLockedWallpaper 中，说明当前壁纸可以在 Home 界面以及锁屏界面显示。这里的代码大概就是这种逻辑，比较简单，不再赘述。

需要注意的是这里的返回值可能为空，我们分析的这个问题就是因为这里返回空所以出现了壁纸不可见的情况导致了黑屏，我们后续分析问题产生的原因。

2.3 WallpaperController.updateWallpaperWindowsTarget
----------------------------------------------------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e1f3d571bfa4fa794b5e187d457caea~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=782&h=453&s=37095&e=png&b=fffefe)

WallpaperController.updateWallpaperWindowsTarget 这个方法，我看了下好像没有太多可以说的，就是把上一步寻找壁纸的目标窗口的结果保存到 WallpaperController 的成员变量 mWallpaperTarget 中。

2.4 WallpaperController.updateWallpaperTokens
---------------------------------------------

回到 WallpaperController.adjustWallpaperWindows 中，如果 WallpaperController.mWallpaperTarget 不为空，那么调用 WallpaperController.updateWallpaperTokens 设置壁纸的可见性：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2c9bcf5b6675486f9948e879a32bc2bb~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=824&h=186&s=24192&e=png&b=fffefe)

这里需要注意的一点就是，及时这里的传参 visibility 是 true，后续可能也无法将壁纸变为可见，因为这里还有额外的判断。

首先这里的成员变量 mWallpaperTokens 定义为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0af00381f6444c80ad7417e230455637~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=770&h=216&s=26936&e=png&b=fefefe)

是一个 WallpaperWindowToken 的队列，在 WallpaperWindowToken 创建的时候，会把它加入到 mWallpaperTokens 中。

接着这里会调用 FindWallpaperTargetResult.getTopWallpaper 来获取当前的壁纸窗口，只有这个壁纸窗口不为空，并且在 mWallpaperTokens 中，我们才能将壁纸的可见性设置为 true。

后续的 WallpaperWindowToken.updateWallpaperWindows 就不分析了。

代码分析完了，现在分析问题。

根据我之前本地操作的结果：

1）、同时设置系统和锁屏壁纸为壁纸 A，此时不会有问题。

2）、在第 1 步的基础上，单独将锁屏壁纸设置为壁纸 B，此时也不会有问题。

3）、在第 2 步的基础上，单独将系统壁纸设置为壁纸 C，出现问题。

3.1 同时设置系统和锁屏壁纸为壁纸 A
--------------------

这个路径下，会 WallpaperWindowToken.setShowWhenLocked 这个方法会被调用，设置 WallpaperWindowToken.mShowWhenLocked 为 true，调用堆栈为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae01c94053d64a78b1698a88068bd351~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1126&h=352&s=54221&e=png&b=fffefe)

对所有窗口进行 mFindWallpapers 遍历时，由于 WallpaperWindowToken.mShowWhenLocked 为 true，因此会设置 mTopShowWhenLockedWallpaper 为壁纸窗口。

对所有窗口进行 mFindWallpaperTargetFunction 遍历时，由于 google 语音助手这个显示在锁屏界面之上的界面对应的 ActivityRecord 是非全屏的，因此会设置 FindWallpaperTargetResult.mNeedsShowWhenLockedWallpaper 为 true：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42424fd7267347578a7800352357c330~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=815&h=150&s=17331&e=png&b=fffefe)

并且由于所有窗口都不满足作为壁纸的目标窗口的条件，因此这一步没有找到目标窗口。

后续再回到 WallpaperController.findWallpaperTarget：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e146c87f9934c26b386c77a6b68846f~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=775&h=150&s=16638&e=png&b=fffefe)

现在 FindWallpaperTargetResult.mNeedsShowWhenLockedWallpaper 为 true，所以这里会调用 FindWallpaperTargetResult.setUseTopWallpaperAsTarget 来将 FindWallpaperTargetResult.useTopWallpaperAsTarget 设置为 true，那么接着就会调用 FindWallpaperTargetResult.getTopWallpaper 尝试获取壁纸窗口，并且将返回的结果作为目标窗口：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2c965d87a02417fa0a471fef106a6ed~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=548&h=118&s=6988&e=png&b=ffffff)

这里由于我们处于锁屏，因此会返回 TopWallpaper.mTopShowWhenLockedWallpaper，并且根据我们的分析，因为 WallpaperWindowToken.mShowWhenLocked 为 true，因此之前我们的确是将壁纸窗口保存在了 TopWallpaper.mTopShowWhenLockedWallpaper 中的，因此这里就可以返回壁纸窗口，并且将其设置为壁纸的目标窗口。

这种情况最终是会找到一个壁纸的目标窗口的，因此壁纸是可见的。

3.2 在第 1 步的基础上，单独将锁屏壁纸设置为壁纸 B
-----------------------------

这种情况下，和 3.1 节的分析内容不会有区别，所以也不会有什么问题。

唯一有点奇怪的是，此时显示的是系统壁纸，而非锁屏壁纸。也不能说奇怪，毕竟系统壁纸才是真正的壁纸，是有一个专门的壁纸窗口对应，而锁屏壁纸，应该只是锁屏界面为自己设置的一张背景图。

3.3 在第 2 步的基础上，单独将系统壁纸设置为壁纸 C
-----------------------------

这种情况下，就会出现问题，原因出在哪儿呢？

看了下 log，发现此时 WallpaperWindowToken.mShowWhenLocked 变成了 false，那么对所有窗口进行 mFindWallpapers 遍历时，由于 WallpaperWindowToken.mShowWhenLocked 为 false，因此会设置 TopWallpaper.mTopHideWhenLockedWallpaper 为壁纸窗口。

而在后续调用 FindWallpaperTargetResult.getTopWallpaper 尝试获取壁纸窗口时，由于此时处于锁屏，因此返回的仍然是 TopWallpaper.mTopShowWhenLockedWallpaper。这就是问题的原因所在了，我们将壁纸窗口保存在了 TopWallpaper.mTopHideWhenLockedWallpaper 中，那么 TopWallpaper.mTopShowWhenLockedWallpaper 就是空的，因此调用 FindWallpaperTargetResult.getTopWallpaper 返回的就是空的，最终结果就是没有为壁纸找到一个目标窗口，壁纸在锁屏状态下变为不可见。

那么为什么 WallpaperWindowToken.mShowWhenLocked 变成了 false 呢，我也没看到 WallpaperWindowToken.setShowWhenLocked 方法有调用将 WallpaperWindowToken.mShowWhenLocked 设置为 false 啊？原来是设置系统壁纸的时候，直接重新创建了一个新的 WallpaperWindowToken 对象：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3f034929590464b85410c038189af2e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1135&h=386&s=65175&e=png&b=fffefe)

WallpaperWindowToken.mShowWhenLocked 默认是 false，并且后续没有再调用 WallpaperWindowToken.setShowWhenLocked 将 WallpaperWindowToken.mShowWhenLocked 设置为 true，所以就出现了问题。

最后再看下同时设置系统壁纸和锁屏壁纸的情况吧，同时设置了壁纸后，会重新创建一个 WallpaperWindowToken 对象，接着就是调用 WallpaperWindowToken.setShowWhenLocked 设置 WallpaperWindowToken.mShowWhenLocked：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21dd9302d7da4adb980342f5809631a8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1533&h=755&s=177329&e=png&b=fffefe)

看调用堆栈，都在 WallpaperManagerService$DisplayConnector.connectLocked 方法中：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ea4504369e942f0990dc07f27cab83b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=720&h=155&s=17693&e=png&b=fffefe)

看到这里需要更正上面的一个说法，就是单独设置了系统壁纸的时候，其实也是调用了 WallpaperWindowToken.setShowWhenLocked 了，但是设置的是 false，因为只是针对系统壁纸生效，而本来 WallpaperWindowToken.mShowWhenLocked 默认的就是 false，所以我之前添加的 log 没有打印......

用白话总结一下这个问题，给我个人的感觉就是：

1）、setShowWhenLocked 这个属性表示的壁纸自己支持不支持在锁屏界面显示，是壁纸自己决定的，或者说是 Launcher 在设置壁纸的时候决定的。

2）、WallpaperController 用来决策壁纸是否需要在锁屏界面上显示。

这个问题很明显就是这两者冲突了，WallpaperController 的逻辑觉得这个时候壁纸应该在锁屏界面显示，但是还是需要看看在 Launcher 设置壁纸的时候，设定壁纸是否可以在锁屏界面显示。如果壁纸不支持在锁屏界面显示，那么 WallpaperController 也不能强行让壁纸在锁屏界面上显示。

分析到这里感觉这应该是 google 的原生问题，但是 pixel 却没有问题，不过发现了一个区别，就是将系统壁纸和锁屏壁纸分别设置为不同的壁纸图片后，在锁屏界面长按 Power 唤起语音助手时，发现 pixel 显示的是壁纸是锁屏壁纸，而我们的机器要么显示的是系统壁纸，要么就不显示。

dump 信息看了下，原来 pixel 的机器有两个 WallpaperWindowToken，分别管理系统壁纸和锁屏壁纸：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2caa54dced564acbae65f2064f28e961~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1422&h=101&s=21755&e=png&b=fffefe)

而我们的机器只有一个，是系统壁纸：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e7da0fc3b384d70961d8b2fb6562788~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1415&h=63&s=11299&e=png&b=fffefe)

锁屏壁纸只是 NotificationShade 为自己设置的一张背景。

跟 SystemUI 的同事沟通了一下，得知 pixel 用的似乎不是 aosp 里的 SystemUI，而是自己另外一套的 SystemUI，并且将 aosp 的 SystemUI 推到手机里也有问题，那这个问题无法参考 pixel 进行修改了。

回顾一下问题发生的原因，即 WallpaperController 判断出锁屏界面需要显示壁纸，壁纸却又说我的出厂设定就是只能在 Home 界面显示，我就不显示，WallpaperController 拗不过壁纸，所以出现了黑屏。

如果要解决这个黑屏问题，我目前能想到的就是围绕 FindWallpaperTargetResult.mNeedsShowWhenLockedWallpaper 这个变量做文章， 既然这个变量被设置为 true 了，就说明当下的确需要壁纸去显示了，不管壁纸它支持不支持在锁屏界面上显示，它都得直楞起来，先给我显示了再说。
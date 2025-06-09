> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7350549827675160588)

> InputDispatcher 无焦点窗口 ANR 问题 1 问题描述 Monkey 跑出的无焦点窗口的 ANR 问题。 特点： 1）、上层 WMS 有焦点窗口，为 Launcher。 2）、nat

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33ab700a77c545bda0bee2177c53cc2b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2000&h=1125&s=2726973&e=jpg&b=b8d0da)

Monkey 跑出的无焦点窗口的 ANR 问题。

特点：

1）、上层 WMS 有焦点窗口，为 Launcher。

2）、native 层 InputDispacher 无焦点窗口，上层为”recents_animation_input_consumer“请求了焦点，但是”recents_animation_input_consumer“最终没有成为焦点窗口，原因是”NO_WINDOW“。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d65594161de14ced8b7c5a3c01421c11~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1897&h=5352&s=1725385&e=png&b=fefdfd)

从 log 分析中可以看到发生 ANR 的场景的上下文：

1）、首先是 SystemUI 主线程比较卡顿的情况下，这种的我们要复现问题可能需要给 SystemUI 的绘制加点延迟。

2）、先启动了一个 Activity，“com.tct.nxtvision_ui/.LauncherActivity”。

3）、随后屏幕转为了 180 度，即 ROTATION_2。

4）、输入 KeyEvent.KEYCODE_RECENT_APPS（312），切换到 Launcher 的 Recents，并且屏幕转为 0 度，即 ROTATION_0。

5）、接着在焦点 App 切换到 Launcher 之前，又输入了一个 KeyEvent.KEYCODE_BACK（4），并且是第一步启动 Activity 接收到了，并且调用了 finish —— KO！！！后续 native 层的 InputDispacher 出现了无焦点窗口的情况。

### 4.1 NO_WINDOW 代码分析

回到当时看 log 时，分析 ANR 出现的直接原因：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0df5b66fb1de420096717ecf6aaa82c2~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1330&h=52&s=14380&e=png&b=fffefe)

这里 InputDispatcher 侧的焦点窗口从 “recents_animation_input_consumer” 切走，直接原因为“NO_WINDOW”，查看到具体代码位置，FocusResolver.setInputWindows：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/10ae41020f9543a9bf7078d5e62c616e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=840&h=655&s=84245&e=png&b=fffefe)

关键的地方有：

大概的内容为，局部变量 currentFocus 代表的是存储的焦点窗口，而 windows 代表的是从 SurfaceFlinger 处传入的最新的可接收输入事件的输入窗口列表，这里继续调用 FocusResolver.getResolvedFocusWindow 去寻找一个焦点窗口，并且调用 FocusResolver.updateFocusedWindow 来更新焦点窗口。

继续看 FocusResolver.getResolvedFocusWindow：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53d55d25120d4b08b51c7a1204f26f72~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=798&h=486&s=60351&e=png&b=fffefe)

这里的逻辑主要是调用 FocusResolver.isTokenFocusable 来判断某一个窗口是否能够作为焦点窗口：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43328a6e418c44e0841e2422a9872002~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=816&h=586&s=56541&e=png&b=ffffff)

这里要遍历的 windows 是从 SurfaceFlinger 发来的最新的输入窗口列表，而 token 是之前设置的焦点窗口，这个函数的大意是根据最新的输入窗口信息判断之前设置的焦点窗口是否有效。

那么可能有几种情况，即对应 Focusability 的 4 个值：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/93db407085c743038d43c97c030f3dd1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=290&h=140&s=5330&e=png&b=fefefe)

1）、返回 NO_WINDOW，说明之前设置的焦点窗口已经不在最新的输入窗口列表里了，即该输入窗口的 Layer 已经被移除了，或者不满足 Layer.needsInputInfo 的条件。

2）、返回 NOT_FOCUSABLE，说明之前设置的焦点窗口还在最新的输入窗口列表里，但是被设置了 NOT_FOCUSABLE 这个标志位，不满足作为焦点窗口的条件了。

3）、返回 NOT_FOCUSABLE，说明之前设置的焦点窗口还在最新的输入窗口列表里，但是被设置了 NOT_VISIBLE，即该 Layer 已经不可见了，所以不能再作为焦点窗口了。

4）、返回 OK，找到了一个符合条件的窗口作为焦点窗口，并且将该窗口保存在传参 outFocusableWindow 中。

从上面的代码分析可知，焦点窗口从 “recents_animation_input_consumer” 切走的原因为它对应的 Layer 已经被移除了，或者不满足 Layer.needsInputInfo 的条件，继续打开 DebugConfig.DEBUG_FOCUS 这个开关继续看看 log，发现有：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cdd0d71439dd450da6170f22d7d1d047~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1847&h=90&s=25367&e=png&b=dfdede)

原因为 “Window went away”，是该 Layer 被移除了。

### 4.2 recents_animation_input_consumer 分析

首先看下正常情况下这个 Layer 的信息为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4b3f6d71381425d95a13df4b7b34c21~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1703&h=181&s=27130&e=png&b=fffefe)

它的 zOrderRelativeOf 就是被 transientHide 的那个 Task。

该 Layer（SurfaceControl）在创建 INPUT_CONSUMER_RECENTS_ANIMATION 对应的 Input ConsumerImpl 对象中创建：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/39fd03a5647a472b9733f365e03a54d3~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=792&h=767&s=87966&e=png&b=fefefe)

显示、隐藏和移除的逻辑为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b9d05e62571463d9c2b89b15e9b3b86~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=646&h=427&s=43503&e=png&b=ffffff)

### 4.3 “Window went away” 原因分析

复现问题后，dumpsys SF 的信息，发现：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb1e9ac605ff4580af13663a6f4804cf~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1761&h=769&s=130556&e=png&b=ffffff)

“recents_animation_input_consumer” 对应的 Layer 仍然存在，那为什么没有遍历到呢？

查看 SurfaceFlinger 向 InputDispatcher 更新输入窗口的地方，SurfaceFlinger.updateInputFlinger：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b10a9634f9174c3d919ba713cace4903~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=851&h=924&s=93693&e=png&b=ffffff)

1）、SurfaceFlinger.updateInputFlinger 方法的逻辑比较简单，调用 SurfaceFlinger.buildWindowInfos 构建一个输入窗口列表，然后发给 InputDispatcher，那么关键的地方就在于 SurfaceFlinger.buildWindowInfos 了，它是如果选择哪个 Layer 可以作为输入窗口的？

2）、SurfaceFlinger.buildWindowInfos 的逻辑也很简单，主要是通过 Layer.needsInputInfo 来判断一个 Layer 是否能够作为输入窗口的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29588ffcd55d453a98286033373080eb~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=864&h=216&s=22162&e=png&b=fefdfd)

应该是主要是判断 Layer.hasInputInfo，而 “recents_animation_input_consumer” 在 show 的时候已经设置了一个 InputWindowInfo 了，所以原因应该不是这个。

继续添加 log，发现的确如此，“recents_animation_input_consumer”没有发送给 InputDispatcher 的原因是 “recents_animation_input_consumer” 这个 Layer 根本就没有遍历到！

原因则是：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8931bd703aca42f0a59574bf1a4aa0f4~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1772&h=470&s=112201&e=png&b=fffefe)

“recents_animation_input_consumer”的相对层级的那个 Task 已经从 Layer 层级结构中被移除了，这个 Task 都遍历不到了，自然 “recents_animation_input_consumer” 也遍历不到了。

所以这个问题的根本原因是：

1）、在 native 层，“recents_animation_input_consumer” 已经不能作为焦点窗口了，因为 transientHide 的那个 Task 已经被移除了。

2）、在上层 WMS 处，“recents_animation_input_consumer” 仍然可以作为一个焦点窗口去请求焦点，没有考虑到 transientHide 的 Task 此时是否已经被移除。

再看下 WMS 处为 “recents_animation_input_consumer” 请求焦点的逻辑：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3d61060c49f442b3a3a98c7dd567655b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=843&h=128&s=19188&e=png&b=fffefe)

主要就是这个成员变量 mActiveRecentsActivity：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/221bae7d37a64454b0eacfa30010d445~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=812&h=307&s=38017&e=png&b=fefcfc)

当 Recents 界面被调起，就为 mActiveRecentsActivity 为 Launcher 对应的 ActivityRecord，并且激活 “recents_animation_input_consumer” 来作为焦点窗口。

比如，当我们点击 Recents 的时候，会启动一个 Transition：

1）、在 Transition.onTransactionReady 阶段，会调用 Transition.handleLegacyRecentsStartBehavior 来设置 InputMonitor.mActiveRecentsActivity 为 Launcher 对应的 ActivityRecord。

2）、在 Transition.finishTransition 阶段，会将 InputMonitor.mActiveRecentsActivity 置为 null。

在我们复现 ANR 的场景：

1）、在 native 层 SF，由于和 “recents_animation_input_consumer” 绑定的那个 transientHide 的那个 Task 已经被移除了，所以 “recents_animation_input_consumer” 就不能作为焦点窗口了。

2）、在上层 WMS，因为仍然处于 Recents 界面，这个 Transition 一直都不会 finish，那么 InputMonitor.mActiveRecentsActivity 就一直不为空，那么每次走到 InputMonitor.updateInputFocusRequest 的时候，就会为 “recents_animation_input_consumer” 请求焦点。

这么看来是 InputMonitor.updateInputFocusRequest 为 “recents_animation_input_consumer” 请求焦点的这段逻辑有点问题。

那我们如何在 pixel 上复现呢？从以上分析可知这个问题似乎是 google 原生问题，SystemUI 卡顿并非复现该场景的必要条件，毕竟本题中 SystemUI 卡顿带来的效果的本质是，推迟 Launcher 称为焦点 App 的时间，从而让输入 KEYCODE_RECENT_APPS 后再次输入的 KEYCODE_BACK 能够被 “com.tct.nxtvision_ui/.LauncherActivity” 接收到做 finish 的操作。

那么我们需要让我们模拟的 App，在输入 KEYCODE_RECENT_APPS 切换到 Launcher 后自动 finish，不需要为 SystemUI 增加延迟，似乎也可以复现 “recents_animation_input_consumer” 请求不到焦点的情况。

最终果然可行，pixel 的 Launcher，“com.google.android.apps.nexuslauncher/.NexusLauncherActivity” 也可以复现：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7fe3a9066d394dacb961fe7c26ebae5b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1869&h=628&s=163002&e=png&b=fffefe)

finish 的代码为：

```
    @Override
    public void onTopResumedActivityChanged(boolean isTopResumedActivity) {
        super.onTopResumedActivityChanged(isTopResumedActivity);
        if (!isTopResumedActivity) {
            Handler handler = new Handler();
            handler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    finish();
                }
            }, 4000);
        }
    }



```

可以看到复现问题的路径更简单了，再次总结一下：

~1）、首先是 SystemUI 主线程比较卡顿的情况下，这种的我们要复现问题可能需要给 SystemUI 的绘制加点延迟。~

2）、写一个 Demo App，启动 “com.example.demoapp/.MainActivity”，为其设置方向 android:screenOrientation="reversePortrait"，这样它一启动屏幕就转为 ROTATION_180（当然用 adb 命令让屏幕转成 180 也行，monkey 应该就是这么做的，没什么区别）。

3）、输入 KeyEvent.KEYCODE_RECENT_APPS（312），这将切换到 Launcher 的 Recents，并且屏幕转为 0 度，即 ROTATION_0。

4）、让 “com.example.demoapp/.MainActivity” 在切换到 Launcher 的 4s 后调用 finish —— KO！！！后续 native 层的 InputDispacher 出现了无焦点窗口的情况，如果再发一个 KeyEvent，我一般发 KeyEvent.KEYCODE_BUTTON_C（98），比如就会复现 ANR，log 就如上面的贴的所示。
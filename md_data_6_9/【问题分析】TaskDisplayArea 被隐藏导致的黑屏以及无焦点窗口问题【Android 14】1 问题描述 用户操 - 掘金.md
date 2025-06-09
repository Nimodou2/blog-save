> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7362513291795611711)

> 1 问题描述 用户操作出的偶现的黑屏以及无焦点窗口问题。 直接原因是，TaskDisplayArea 被添加了 eLayerHidden 标志位，导致所有 App 的窗口不可见，从而出现黑屏和无焦点窗口问题，相

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69684fb0519345c6a341ba7b5c2ec523~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1920&h=1080&s=2375432&e=jpg&b=021217)

用户操作出的偶现的黑屏以及无焦点窗口问题。

直接原因是，TaskDisplayArea 被添加了 eLayerHidden 标志位，导致所有 App 的窗口不可见，从而出现黑屏和无焦点窗口问题，相关 log 为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/191e23329be0449da07cdfc04806ab25~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1847&h=99&s=37647&e=png&b=fefafa)

这个 log 是 MTK 添加的，用来分析 ANR 问题还是非常有帮助的，对于分析黑屏问题同样有用。

该问题复现步骤如下：

1）、在 google 的 dialer app 中拨打一个电话，启动”com.google.android.dialer/com.android.dialer.incall.activity.ui.InCallActivity“界面。

2）、按下一个特殊按键，按下该按键后会切换 HomeApp，从”com.tcl.android.launcher“切换到”com.tct.book“，因此会新启动一个 Activity，”com.tct.book/.ui.MainActivity“。此阶段还会因为设置了新壁纸，导致 CONFIG_ASSETS_PATHS 改变触发全局 Configuration 的更新。

3）、接着马上再连续 Power 键灭屏、亮屏，此时就有可能复现黑屏的情况。

其中第二步的按键操作是特殊定制的。

接下来分析 log：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac08ae91b7c646b8aa647e1e6a3a2438~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1897&h=2478&s=853890&e=png&b=fefdfd)

这个问题其实之前遇到过一次，只不过当时是解决 monkey 跑出的 ANR 问题的，从 log 上看到也是于 TaskDisplayArea 被设置了 eLayerHidden 标志位，导致了所有的 App 窗口都被视为是不可见的，从而无法获取焦点，出现了 ANR。这个问题当时并没有找到原因，结果这次测试直接手动复现出来了。

### 3.1 什么情况下会为 TaskDisplayArea 设置 eLayerHidden 标志位

先看一下当时关于什么情况下会为 TaskDisplayArea 设置 eLayerHidden 标志位的分析。

本地尝试后发现，一般情况下直接点击 Power 键灭屏，是不会为 TaskDisplayArea 对应的 Layer 设置 eLayerHidden 标志位的。

但当灭屏后再去打电话，此时会调起 InCallUI。接着取消通话，InCallUI 移除，此时会为 TaskDisplayArea 添加 eLayerHidden 标志位：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b3c0908c7e594cbc9ccbb4fdeaa2fca4~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1914&h=257&s=58262&e=png&b=300a25)

然后断点发现添加改标志位的代码为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ddb4f3fa68304eb790636d2483d7484b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1646&h=604&s=113306&e=png&b=3c3f41)

当解锁后，会为 TaskDisplayArea 去掉该标志位，断点后发现在：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/28b9ed1268864a97b20a0911408a9c40~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1634&h=564&s=106819&e=png&b=3c3f41)

都是在 Transitions.setupStartState 中：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e025db47d94d48399d6c16887e2660e1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=826&h=407&s=45957&e=png&b=ffffff)

逻辑还是比较简单的：

1）、如果该 WindowContainer 对应的 Transition.ChangeInfo/TransitionInfo.Change 的动画类型是 TRANSIT_OPEN 或者 TRANSIT_TO_FRONT，那么就在动画开始执行前为其调用 Transition.show。

2）、如果该 WindowContainer 对应的 Transition.ChangeInfo/TransitionInfo.Change 的动画类型是 TRANSIT_CLOSE 或者 TRANSIT_TO_BACK，那么就在动画开始执行前为其调用 Transition.hide。

我们从 log 能看到 TaskDisplayArea 是参与了动画的，并且它的 ChangeInfo 的类型就是 TRANSIT_TO_BACK，所以在 Transitions.setupStartState 中就会为 TaskDisplayArea 的 SurfaceControl 调用 Transition.hide 为其 Layer 添加 eLayerHidden 标志位。

### 3.2 问题的直接原因

接着再回到我们现在的这个问题。

先看最直接的那个原因：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a31679d84d624823a093634a48a339e9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1844&h=551&s=204275&e=png&b=fef6f5)

疑点有两个：

1）、为什么 Task 提升为了 TaskDisplayArea？

2）、为什么 TaskDisplayArea 动画类型为 TO_BACK？

梳理一下这个 Transition#76 的上下文，标注几个关键节点：

1）、”com.tct.book/com.tct.book.ui.MainActivity“启动并且绘制完成。

2）、设置新壁纸，全局 Configuration 发生改变，此时创建了 Transition#76，类型为 TRANSIT_CHANGE。

3）、Dialer 又重新启动了”com.google.android.dialer/com.android.dialer.incall.activity.ui.InCallActivity“，新建了 Task#24。

4）、按下 Power 键创建了 TRANSIT_SLEEP 类型的 Transition#77，不过由于 Transition#76 正在收集，所以 Transition#77 进行了排队，并且刚刚启动的 InCallActivity 也因为”sleep“的原因被 pause。

5）、Transition#76 走到 Transition.onTransactionReady。

6）、Transition#77 开始收集。

7）、InCallActivity 重新变为 resume。

8）、Transition#77 被 abort。

接下来开始分析。

### 3.3 对 Task 提升为 TaskDisplayArea 的分析

从”Initial targets:“这条 log 我们可以看到，最初 TaskDisplayArea 是没有直接被收集到 Transition 中的，而是从经过了两次 PROMOTE 之后，被收集了进来：

1）、检查”com.tct.book“的 Task#23，发现可以提升到 Launcher 的 RootTask，Task#1（这里的”com.tct.book“也是一个 Launcher App）。

2）、又检查 Launcher 的 RootTask，Task#1，发现可以提升到 TaskDisplayArea。

3）、TaskDisplayArea 由于灭屏的原因，其 mVisibleRequested 被置为 false，导致 Transition.ChangeInfo.getTransitMode 方法为其选择了 TRANSIT_TO_BACK 的动画。

”com.tct.book“的 Task#23，首先是因为全局 Configuration 改变的原因，被添加到了 Transition#76 中：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a11f5374caa64471826ad33c717c9413~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1844&h=280&s=97665&e=png&b=fffefe)

因此首先肯定是会为该 Task 创建 ChangeInfo 对象，并且加入到 Transition.mChanges 中。然后根据我们对这段代码的理解，一般这个时候，也会为 TaskDisplayArea 以及 DisplayContent 创建 ChangeInfo 对象并且加入到 Transition.mChanges 中。这就为后续 Transition.onTransactionReady 的时候，将 Task 提升到 TaskDisplayArea 提供了可能。

Transition#76 走到 Transition.onTransactionReady 的时候，检查 Task#23 是否可以提升的时候，看到它的所有姊妹 Task 都没有参与动画，并且都是不可见的，因此就认为可以提升，从而动画的主体就从 Task#23 变成为了 TaskDisplayArea。

根据我们的上下文分析，Transition#76 走到 Transition.onTransactionReady 之前，正好按下了 Power 键，并且之前 resume 的 inCallActivity 也的确因为”sleep“的原因变成 pause 了，那么说明 Dialer 对应的 Task#24 是不可见的，因此 Task#23 就可以提升为 TaskDisplayArea。

### 3.4 对 TaskDisplayArea 动画类型为 TO_BACK 的分析

首先从生成的 TransitionInfo 的信息看到 TaskDisplayArea 的动画类型为 TO_BACK，动画类型在 Transition.ChangeInfo.getTransitMode 中计算：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e629d1ed8cba45e292cf1d9a95c34b29~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=638&h=251&s=22824&e=png&b=fffefe)

由于整个过程中：

1）、没有 transientLaunch 相关的启动。

2）、TaskDisplayArea 始终是存在的，因此 mExistenceChanged 是不会有变化的。

因此只会根据其可见性返回 TRANSIT_TO_FRONT 或者 TRANSIT_TO_BACK，并且我们从后续的 log 信息知道了这里返回了 TRANSIT_TO_BACK，说明此时 TaskDisplayArea 调用 isVisibleRequested 返回了 false：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84cf3a539ab145fcb4862435f4dc340b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=871&h=244&s=30492&e=png&b=2b2b2b)

成员变量 mVisibleRequested 只在 WindowContainer.setVisibleRequested 方法中进行设置：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32c2cdef5eca43d6a40ead5472e00f60~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=682&h=455&s=67196&e=png&b=2b2b2b)

查看该方法调用的地方：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6ed57c1f971407c8885c8164e46235e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1680&h=372&s=94678&e=png&b=3f4244)

只有一处地方可能和 TaskDisplayArea 相关，即 WindowContainer.onChildVisibleRequestedChanged：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b35ccd1ed9f4a89ba23a07c680c8d71~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=778&h=595&s=92007&e=png&b=2b2b2b)

该方法在 ActivityRecord 调用 setVisibleRequested 方法设置 ActivityRecord 的时候就会调用，用来反作用于 Task 以及更高级别的 WindowContainer 的可见性。

大致看下该方法，发现逻辑还是比较好理解的：

1）、如果当前 WindowContainer 是不可见的，但是传入的这个子 WindowContainer 被设置为了可见，那么就设置当前 WindowContainer 为可见。

2）、如果当前 WindowContainer 是可见的，但是传入的这个子 WindowContainer 被设置为了步可见，那么继续寻找其它子 WindowContainer 中是否有可见的，只要有一个子 WindowContainer 是可见的，那么当前 WindowContainer 仍然应该被认为是可见的。只有所有子 WindowContaienr 都不可见了，那么当前 WindowContainer 才会被认为是不可见的。

回到我们的问题中，很显然单纯的 App 切换并不能导致 TaskDisplayArea 变成不可见，再回顾我们发生问题时的操作步骤，似乎也只有灭屏能做到了。

灭屏，所有 Activity 都会被 pause、stop，变为不可见 -> 所有 Task 都不可见 -> TaskDisplayArea 不可见。后续打了 log 后发现的确如此，Task 或者 TaskDisplayArea 都不能主动设置自身的可见性，只能是 ActivityRecord 先主动设置 ActivityRecord 的可见性，然后再影响他们的可见性。

### 3.5 InCallActivity 重新 resume 的时候没有恢复吗

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f31e79df72504be0b0092d26821d35c0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1557&h=256&s=97264&e=png&b=fffefe)

看到 log，虽然后续 InCallActivity 重新又被设置为了 resumeActivity，但是此时这里的新建的 Transition#77 被 abort 了，并且这也是最后一个 Transition 了，导致后续没有办法重新为 TaskDisplayArea 调用 Transtion.show 方法，所以后续无法恢复。

如果 Transition#77 没有被 abort，并且基于这里的信息只有 Dialer 参与了动画，那么 Dialer 是可见的，并且 Launcher 没有参与动画并且不可见，所以 Dialer 对应的 Task 是有机会提升为 TaskDisplayArea 的，那么是有机会恢复的。

那么再看下这个 Transition#77 的情况：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7820eb7cc1946f5849f37c692cff521~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1559&h=514&s=176393&e=png&b=fffefe)

首先这个 Transition#77 是一个 SLEEP 类型的 Transition，它在按下 Power 键准备灭屏的时候创建，此时 Transition#76 正在收集，所以它被推迟，进行了排队。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e0ba58577e042e49c67a6ba44a49386~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1582&h=341&s=126007&e=png&b=fefdfd)

Transition#77 开始收集，是在 Transition#76 走到 Transition.onTransactionReady 的时候，此时看到正好 InCallActivity 被设置为 resume 了，那么它应该也被设置为可见了，但是那么 Transition#77 就被 abort 了。

又回到这个问题了，为什么 Transition#77 被 abort 了呢？

Transition#77 对应的是按下 Power 键灭屏的流程，它的类型是 SLEEP，因此我们可以知道它应该是在 RootWindowContainer.applySleepTokens 中创建的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91fc69692f6f4c10bf3ea0b40b8cf338~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=829&h=656&s=73936&e=png&b=ffffff)

涉及到我们的分析的内容为，遍历所有 DisplayContent：

1）、创建一个 TRANSIT_SLEEP 类型的 Transition 对象。

2）、创建一个 TransitionController.OnStartCollect 类型的接口类，包含一个名为 onCollectStarted 的回调方法。

3）、判断当前是否有 Transition 正在收集，如果没有，那么直接将第一步创建的 Transition 对象移动到收集状态，否则调用 TransitionController.startCollectOrQueue 方法。

从 log 中我们知道了此时是将这个 TRANSIT_SLEEP 类型的 Transition 拿去排队了，即调用了 TransitionController.startCollectOrQueue：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eba4d5f07b76443c9efa54a8917cf7f5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=739&h=463&s=47839&e=png&b=fefdfd)

这里的逻辑也比较简单，如果当前有 Transition 正在收集，那么再检查一下刚刚创建的这个 Transition 能否和这个正在收集的 Transition 并行收集，如果不行，那么调用 Transition.queueTransition 将这个新创建的 Transition 添加到等待队列中，即成员变量 mQeuedTransitions 中。

需要注意的是 mQeuedTransitions 是一个 QueuedTransition 的队列，QueuedTransition 是对 Transition 还有 TransitionController.OnStartCollect 做的一层封装。

后续正在排队的 Transition 会在 TransitionController.tryStartCollectFromQueue 中被取出：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ddd33a2c376b4694a9fdcdb9ed303f81~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=787&h=360&s=42308&e=png&b=ffffff)

内容大致是：

1）、从 mQeuedTransitions 中取出队首的那个 Transition，为其调用 TransitionController.moveToCollecting 移动到收集状态。

2）、调用之前排队时传入的那个 TransitionController.OnStartCollect 接口类的 onCollectStarted 回调。

这个 TransitionController.OnStartCollect 对象我们之前是在 RootWindowContainer.applySleepTokens 方法中创建的：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92377c5508b549a0b4b1e091c8054065~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=815&h=170&s=13136&e=png&b=ffffff)

如果这个回调执行的时候被推迟，并且此时屏幕不应该被休眠，那么将这个 Transition 中止掉，这个 Transition 自然就是上面创建的 TRANSIT_SLEEP 类型的 Transition 了。再回到我们问题的场景，很显然这个 TRANSIT_SLEEP 的 Transition 就是 Transition#77，他之前是被推迟了，并且走到这里的时候，InCallActivity 已经因为”sleep“被 pause 后重新又 resume 了，所以说明此时屏幕已经唤醒了，也就说屏幕不应该休眠，所以这个 Transition 就被 abort 了。

再次根据 log 总结一下复现问题的几个关键点，总结出该问题复现的一般路径：

1）、写一个 Activity1，按下按钮设置壁纸，设置壁纸后 —— 发生 ConfigChange，创建 Transition1，类型为 TRANSIT_CHANGE。

2）、以 new task 的方式新启动一个 Activity2。

2）、按 Power 键灭屏 —— 创建 Transition2，类型为 TRANSIT_SLEEP，并且被延迟，排队等候，并且 Activity2 被 pause，TaskDisplayArea 被设置为不可见。

3）、Transition1 走到 Transition.onTransactionReady，后续会为 TaskDisplayArea 添加 eLayerHidden 标志位。

4）、按下 Power 键亮屏，Activity2 重新 resume，并且 Transition2 被 abort。

大概的代码为在 Activity.onCreate 里初始化一个 Button，按下按钮后调用 setWallpaper 方法设置壁纸，并且在短暂的延迟后以 NEW_TASK 的方式启动另外一个 Activity：

```
        changeWallpaper = findViewById(R.id.changeWallpaper);
        changeWallpaper.setOnClickListener((v) -> {
            setWallpaper();

            Handler handler = new Handler();
            handler.postDelayed(() -> {
                    startActivity(new Intent(MainActivity.this, LongDrawActivity.class));
            }, 100);
        });


```

setWallpaper 的大致为：

```
    private void setWallpaper() {
        WallpaperManager wallpaperManager = WallpaperManager.getInstance(this);

        try {
            InputStream inputStream;
            if (mWallpaperId == 2) {
                inputStream = getAssets().open("1.png");
                mWallpaperId = 1;
            } else if (wallpaperId == 1) {
                inputStream = getAssets().open("2.png");
                mWallpaperId = 2;
            } else {
                inputStream = getAssets().open("4.png");
                mWallpaperId = 2;
            }

            Bitmap bitmap = BitmapFactory.decodeStream(inputStream);
            wallpaperManager.setBitmap(bitmap);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }


```

该问题还是非常难复现的，以上顺序不能出错，不然无法复现到问题，并且两次按 Power 键的时间也非常难掌握，只能进行多次尝试，运气好了可能会复现一次。

经过多次尝试，最终成功在 pixel 上复现了...... 和我们的问题发生时一样的 log，但是 pixel 没问题：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6292ee3331944ab492dd439936375a4e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1604&h=472&s=125677&e=png&b=fffefe)

看到这里 TaskDisplayArea 也是只参与了一次动画，并且类型为 TO_BACK，但是为什么 pixel 没问题呢？

原来时后面跟了一句很关键的 log：

4-26 18:07:25.791  1829  5630 E TransitionController: DisplayArea became visible outside of a transition: DefaultTaskDisplayArea@65482673

正是这里，将 TaskDisplayArea 重新变成了可见，而我们的代码里没有这个 patch。

最后在 google 网站上找到该 patch，把这个 patch 打上后问题解决：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb4df394554249349f925523060983ff~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1081&h=1660&s=219437&e=png&b=fefefe)

最后大概看一下这个 TransitionController.validateStates 方法，很明显这是一个纠错的机制，该方法在 Transition 流程的最后 TransitionController.finishTransition 方法中才调用，防止动画结束后把不该隐藏的 WindowContainer 隐藏掉了。
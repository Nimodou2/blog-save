> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7391284215113924620)

> ShellTransitions Transition onTransactionReady calculateTargets tryPromote canPromote

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/21000b9b89aa43038e117fdcdd54c042~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=RRxdOJT5nl1bBBKyLIcZhGVBFc0%3D)

Transition.onTransactionReady 的内容比较长，我们挑重点的部分逐段分析（跳过的地方并非不重要，而是我柿子挑软的捏）。

1 窗口绘制状态的流转以及显示 SurfaceControl
------------------------------

注意我们这里的 SurfaceControl 特指的是 WindowSurfaceController 的 mSurfaceControl，如果对这个不是很了解的，可以回顾一下之前写的关于 SurfaceControl 的文章：

[【基础】2、Surface 的创建【Android 12】 - 掘金 (juejin.cn)](https://juejin.cn/post/7234450312726708282#heading-5 "https://juejin.cn/post/7234450312726708282#heading-5")

接着分析代码：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/264d20906b1d48c7a5220937d59ed0d7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=bbKkZYSYtFS%2FYDewASqdqHSiCNM%3D)

先跳过 Transition.commitVisibleActivities，看到首先是将 Transition.mState 置为 STATE_PLAYING，这意味着动画马上就要执行了。

然后是为 Transition 的两个成员变量 mStartTransaction 以及 mFinishTransaction 赋值，mFinishTransaction 不用多说，看到 mStartTransaction 被赋值为传参 transaction，传参即我们上一篇分析中的在 SyncGroup.finishNow 创建的一个 Transaction，局部变量 merged：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ffc6710601ee473f820343dfe3d31886~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=nj9UVEua64ro5asf8%2Br%2BljF8Xps%3D)

一个 “start transaction” 和一个“finish transaction”，我按照个人的理解，举个例子说明一下，如果我们从 ActivityA 上启动了一个 ActivityB：

1）、对于 ActivityA 来说，它相关的 SurfaceControl（准确一点说则是 WindowSurfaceController.mSurfaceControl）需要在动画结束的时候再隐藏，如果它在动画开始前就隐藏，那么就无法看到 ActivityA 的动画效果了（向右平移退出或者淡出之类的动画）。

2）、对于 ActivityB 来说，它相关的 SurfaceControl 需要在动画开始的时候就显示出来，如果它在动画开始的时候还没有显示，那么同样也无法看到 ActivityB 的动画效果了（向右平移进入或者淡入之类的动画）。

从以上分析可知，ActivityA 和 ActivityB 相关的 SurfaceControl 可见性变化的时机是不同的，那么这个行为通过一次 Transacton.apply 是无法做到的，所以就需要两个 Transaction，即 “start transaction” 和“finish transaction”。“start transaction”在动画开始前调用 apply，用于在动画开始执行前提前将 ActivityB 进行显示，“finish transaction”则是在动画结束的时候调用 apply，用于在动画结束的时候再将 ActivityA 隐藏。

最重要的是要弄清楚 “start transaction” 和“finish transaction”这两个 Transaction 调用 apply 方法的时机，在以后的 Transition 流程中会分析到。

再来看 Transition.commitVisibleActivities 方法的内容：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/42582341445e49ea8decdbdbe5b4a2d9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=78uQJQ6cpPRDnlthTredwhBhjHE%3D)

如该方法的注释所说，当前 Transition 已经准备好执行动画了，这里先让 “start transaction” 把相关需要显示的 SurfaceControl 显示出来。

Transition.mParticipants 是参与动画的 WindowContainer 集合，那么这个方法就是遍历这个集合：

1）、调用 ActivityRecord.commitVisibility 设置相关 ActivityRecord 的为可见。

2）、调用 ActivityRecord.commitFinishDrawing 进一步设置相关 SurfaceControl 为可见。

ActivityRecord.commitVisibility 方法内容比较多，主要是用来 ActivityRecord 的可见性，即其成员变量 mVisible，除此之外还有很多别的逻辑，但是和我们要分析的 Transition 内容无关，只需要知道这里设置了 ActivityRecord 的可见性即可，不去多说。我们主要看下 ActivityRecord.commitFinishDrawing：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/343b312ff5bb436a9f296f6669a6721f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=%2BQt7mHfqfIgKgoIHOR%2BxLNLAYPA%3D)

很简单，为每一个 child，即 WindowState 调用 commitFinishDrawing 方法：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f9d3b6f40c734edda52b3106eeee7df3~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=ITH2fjSReoGjvmlpGNc4CkGN6og%3D)

1）、调用 WindowStateAnimator.commitFinishDrawingLocked 方法，继续将窗口对应的 WindowStateAnimator 的 mDrawState，即绘制状态进行流转。

2）、调用 WindowStateAnimator.prepareSurfaceLocked，设置 SurfaceControl 的可见性。

这两个方法都比较重要，我们接下来分别进行分析。

### 1.1 窗口绘制状态的流转

SurfaceControl 最终的显示和窗口的绘制状态密切相关，所以我感觉这里有必要看一下 WindowStateAnimator.mDrawState 这个状态是如何切换的，并且我自己对这个窗口的绘制状态也是不求甚解，也希望借着这个机会了解一下。

先分析一下代码，回头再试着总结一下。

#### ·1.1.1 WindowStateAnimator.commitFinishDrawingLocked

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/63181a3b94ae42798705ccbd73a3a001~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=9g0QChbatP6lAcpBBIJE0Bl4Mj4%3D)

首先将 WindowState.mDrawState 设置为 READY_TO_SHOW。

然后如果当前 WindowStateAnimator 相关的 WindowState 满足以下条件之一，则继续调用 WindowStateAnimator.performShowLocked：

1）、没有对应的 ActivityRecord，即是一个非 Activity 窗口：

```
activity == null


```

2）、有对应的 ActivityRecord，并且此时已经可以显示窗口了：

```
activity.canShowWindows()


```

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ac5551592b8943bc8603fce5d8a79553~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=wLjWGAwiwZimLaTp7shIgDY%2BZ88%3D)

重要是就是看这个 ActivityRecord 的 mSyncState 是不是 SYNC_STATE_WAITING_FOR_DRAW，如果是这个值，那么就说明这个 ActivityRecord 是处于动画中的。

但是有一个问题是，ActivityRecord 的 mSyncState 是不会被设置为 SYNC_STATE_WAITING_FOR_DRAW 的，只有 WindowState 才会，那岂不是每次走到这里判断 ActivityRecord 是否 drawn，都将一直是 true。

3）、是一个 TYPE_APPLICATION_STARTING 类型的窗口，即 SplashScreen 或者 Snapshot：

```
mWin.mAttrs.type == TYPE_APPLICATION_STARTING


```

总而言之，如果这个 WindowState 满足了绘制了条件，那么将继续调用 WindowState.performShowLocked。

#### 1.1.2 WindowState.performShowLocked

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3bb17d625f75433b81ebdce470532377~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=wDgZ%2BUANxluhEdcOq35ck2z1Frw%3D)

如果 WindowStateAnimator.mDrawState 不是 READY_TO_SHOW，那么返回 false，否则将其置为 HAS_DRAWN，并且返回 true，这将使得我们可以下一步继续调用 WindowStateAnimator.prepareSurfaces 方法。

从 WindowStateAnimator.commitFinishDrawingLocked 以及 WindowState.performShowLocked 这两个方法都能看到，窗口的绘制状态是循序渐进的，必须是状态 A -> 状态 B -> 状态 C，不存在状态 A 直接到状态 C 之类的。

#### 1.1.3 窗口绘制状态小结

首先是 mDrawState 在 WindowStateAnimator 的定义，以及几个取值：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1403746666cd40308d699f4cd47a0701~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=sjsTYrkov%2BLCndMRz%2F3fMkgEfXE%3D)

结合着 Activity 启动的一般流程，我大致总结一下：

1）、NO_SURFACE：当没有 Surface 的就置为这个状态。

这个很好理解，一般窗口销毁相关的流程，会将 WindowStateAnimator.mDrawState 设置为 NO_SURFACE，比如：

WindowState.removeImmediately

-> WindowStateAnimator.destroySurfaceLocked

-> WindowStateAnimator.destroySurface

此阶段没有窗口，也没有 Surface。

2）、DRAW_PENDING：当 Surface 被创建之后，窗口被添加但还没有开始绘制之前，就会置为这个状态。在这个时期，Surface 是隐藏的。这表明 Surface 正等待应用程序绘制窗口的内容。

当窗口被添加，接着 App 侧开始走 measure、layout 以及 draw 流程，在 draw 之前，会将窗口在 WMS 侧进行 relayout，经过：

WMS.relayoutWindow

-> WMS.createSurfaceControl

-> WindowStateAnimator.createSurfaceLocked

-> WindowStateAnimator.resetDrawState

会将 WindowStateAnimator.mDrawState 设置为 DRAW_PENDING。

这个流程我们也很熟悉，即之前分析创建 WindowSurfaceController 的 SurfaceControl 的流程。

此阶段窗口被添加但还没绘制出来，SurfaceControl 也是隐藏的。

3）、COMMIT_DRAW_PENDING：当窗口的绘制操作完成，但是这个 Surface 还没有显示出来之前，状态会设置为此值。这个 Surface 会在下次 layout 过程中显示出来。

当窗口绘制完成，App 侧调用 ViewRootImpl.reportDrawFinished 后，就会调用 IWindowSession 的对端，经过：

Session.finishDrawing 

-> WMS.finishDrawingWindow 

-> WindowState.finishDrawing 

-> WindowStateAnimator.finishDrawingLocked

会将 WindowStateAnimator.mDrawState 设置为 COMMIT_DRAW_PENDING。

此阶段窗口已经绘制完成，但是 Surface 由于一些原因还不能显示。

4）、READY_TO_SHOW：这个状态标识窗口的绘制操作已经提交，但 Surface 还没有真正显示。在一组窗口（例如属于同一个应用的多个窗口）准备显示时，系统会使用这个状态来延迟显示 Surface，直到所有相关窗口都准备好一起显示。

首先我们看到在动画的流程中，窗口的绘制状态被设置为 READY_TO_SHOW 的流程为：

Transition.onTransactionReady

-> Transition.commitVisibleActivities

-> ActivityRecord.commitFinishDrawing

-> WindowState.commitFinishDrawing

-> WindowStateAnimator.commitFinishDrawingLocked

结合注释，我个人的理解是，绘制状态被置为 READY_TO_SHOW，表明此窗口已经绘制完了，可以准备显示它的 SurfaceControl 了，但是它的 SurfaceControl 需要等待和其它的 SurfaceContrl 一起显示，或者说等待动画走到特定阶段才能显示，因此我们这里推迟其 SurfaceControl 的显示时间，将窗口的绘制状态设置为 READY_TO_SHOW。

如果不考虑和其它窗口一起显示，那么我想在这一步就可以将绘制状态设置为 HAS_DRAWN 了，即 READY_TO_SHOW 这个状态值是不必要的。

5）、HAS_DRAWN：当窗口首次在屏幕上显示时，就会设置为此状态。

这个值在 WindowState.performShowLocked 方法中被设置，紧跟着 WindowStateAnimator.commitFinishDrawingLocked 方法。

严谨一点的话注释的说法其实是不准确的，当窗口绘制状态被设置为 HAS_DRAWN 的时候，只是说明 SurfaceControl 接下来可以显示了，但是 SurfaceControl 仍然没有显示，屏幕上是看不见的。

6）、总结一下，从以上分析可知，这些状态值不只涉及了窗口的绘制流程，还涉及了 SurfaceControl 的显示流程：

*    NO_SURFACE：没窗口，也没 SurfaceControl。

*   DRAW_PENDING：有窗口，但没开始绘制。有 SurfaceControl，但不能显示。

*   COMMIT_DRAW_PENDING，窗口刚刚绘制完，SurfaceControl 还不能显示。

*   READY_TO_SHOW：窗口已经绘制完了，SurfaceControl 可以显示了，但没必要，再等等。

*   HAS_DRAWN：窗口已经绘制完了，SurfaceControl 也可以显示了。

### 1.2 显示 SurfaceControl

回到 WindowState.commitFinishDrawing，在调用 WindowStateAnimator.commitFinishDrawingLocked 将窗口的绘制状态走完后，接下来就是调用 WindowStateAnimator.prepareSurfaceLocked 来显示 SurfaceControl 了。

注意这个方法被调用的地方有两处：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/113d58e5498f44538ca9ee5ee2069f32~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=fW5ZB4aAGf5dD%2FZY%2BM5%2FihDEE2w%3D)

还有一处调用的地方在 WindowState.prepareSurfaces，这个是更通用的流程，但是动画流程下，则稍微不同，即我们分析的这个流程。

看代码：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d79b6f6fc36e4ce1a96be9c5ac8557b9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=hv3DaClih%2Bvli0QouCMi9sFZJ6E%3D)

我们只看和显示 SurfaceControl 相关的部分：

1）、如果窗口不在屏幕上，则调用 WindowStateAnimator.hide -> WindowSurfaceController.hide 来隐藏 SurfaceControl。

2）、如果窗口在屏幕上，那么进一步判断窗口的绘制状态，只有窗口的绘制状态为 HAS_DRAWN，才能继续调用 WindowSurfaceController.showRobustly 来显示 SurfaceControl：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/92cda36c561c4283adbbbea84627f2d3~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=c816Jtp%2BNqBz7iGyLsIgjcyj5wM%3D)

关键的就那一句，调用 Transaction.show 来显示相关 SurfaceControl，但是要注意的是这里并没有调用 Transaction.apply，所以这个时候窗口还是没有显示。

窗口的最终显示则是和这个传参 Transaction 对象有关，这个 Transaction 对象则是之前说的”start transaction“，那么这个 Transaction 的 apply 方法的调用时机则是跟 Transition 的流程相关，以后的分析会看到。

2 计算动画目标
--------

这一节的内容是调用 Transition.calculateTargets 来计算动画的目标：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a1ae09df9daa44cc951a491f3bcc296c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=1ndIkoRwA4rq5%2BLLI%2B6%2FgzLRH%2FA%3D)

Transition 的成员变量 mTargets 定义为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b3b6314253144e318a2c670777005513~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=0nX5ZGSfbbAbDZp5ao6ndLBHQic%3D)

之前收集到的动画参与者提升后的最终的动画目标，也就是说最终执行动画的主体并非是之前收集到的动画参与者，而是这一步用动画参与者计算得到的动画目标。

Transition.calculateTargets 的内容为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/2a62819685d8488ba471bd4955a3fd08~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=7XL6BzewnFGoLEDOJ3ommY0N3kI%3D)

大致的内容为：

1）、创建一个 Transition.Targets 类型的局部变量 targets，来收集动画目标。

2）、遍历 Transition.mParticipants，从 Transition.mChanges 中取出对应的 ChangeInfo 对象放到 Transition.Targets.mArray 中，但是跳过 WindowState 类型的动画参与者，以及跳过那些根据 ChangeInfo.hasChanged 得出前后没有发生变化的动画参与者。

3）、调用 Transition.tryPromote 尝试提升 targets 中保存的动画目标的级别。

我们这一节主要来看下这个 Transition.tryPromote。

”promote“，提升的动画目标在 WindowContainer 层级结构中的级别，这个逻辑之前在 AppTransitionController.getAnimationTargets 也用到了，思想都是类似的。比如一个 Task 中有两个 ActivityRecord，并且这两个 ActivityRecord 要分别执行一段动画，也就是动画执行的主体是 ActivityRecord。如果这两个 ActivityRecord 刚好都想向左平移同样的距离，那么我们就不需要为这两个 ActivityRecord 分别应用一段平移的动画，而是直接将这个平移的动画应用到它们共同的父容器 Task 上，并且实现的效果是一样的。这也就是”promote“的含义，动画的目标主体从 ActivityRecord” 提升 “到了更高一级的 Task 上。

接着看代码，Transition.tryPromote。

### 2.1 Transition.tryPromote

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3e91326b0b364bc0a6837ae4c9449f2f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=jswnVoqUt8Dypx78zu32Q%2BXLZvM%3D)

主要逻辑为遍历 Targets.mArray 中的每一个 ChangeInfo 对象，调用 Transition.canPromote 方法来判断他们是否能够提升为父容器。

1）、如果不能，直接跳过该 ChangeInfo 对象，判断下一个。

2）、如果能，就说明提升成功。此外还要调用 Transition.reportIfNotTop 来继续判断它是否是 organized（我的理解就是这个 WindowContainer 是否是系统开机后自动创建的，不是需要的时候再去创建的）。如果不是，那么将当前 WindowContainer 对应的 ChangeInfo 从局部变量 targets 中移除，然后把它的父 WindowContainer 对应的 ChangeInfo 加如到 targets 中。如果是，那么在不移除当前 WindowContainer 对应的 ChangeInfo 的前提下，把它的父 WindowContainer 对应的 ChangeInfo 加如到 targets 中。这里应该是针对 organized 的 WindowContainer 的特殊处理，确保 organized 的 WindowContainer 的变化也能够报告到 WMShell 那边。

因此重点其实是 Transition.canPromote 逻辑。

### 2.2 Transition.canPromote

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d5182a035c084bec8bfcc903c30e306b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=reU1HgAj6wXEPWhSSXCAc1KBXAo%3D)

感觉这段代码还是比较重要的，我们逐行分析。

#### 2.2.1 片段 1

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a19e07cc09c040eebc92f636f70d2e88~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=blek4zM83cAO6E6LoZ%2F%2BWrH4WWw%3D)

1）、对应 WindowContainer.canCreateRemoteAnimationTarget 方法，目前只有 TaskDisplayArea、TaskFragment 以及 ActivityRecord 会返回 true，其它类型的 WindowContainer 都会返回 false，也就是说父容器不是这几类的 WindowContainer 将无法得到提升，那么目前只有这几种提升：WindowState 到 ActivityRecord，ActivityRecod 到 TaskFragment，TaskFragment 到 TaskFragment（因为 TaskFragment 存在嵌套，比如 Home 类型的 TaskFragment），以及 TaskFragment 到 TaskDisplayArea。另外从 Transition.calculateTargets 的逻辑我们看到了执行动画的 target 至少是 WindowToken 这一级的，并且看收集的逻辑，似乎也没有看到过直接收集 WindowState 的，因此实际上提升只存在以下几种情况：

*   ActivityRecod 到 TaskFragment。

*   TaskFragment 到 TaskFragment。

*   TaskFragment 到 TaskDisplayArea。

2）、如果找不到父 WindowContainer 对应的 ChangeInfo，则不提升，返回 false。

3）、如果父 WindowContainer 有 ChangeInfo，但是此时的状态和收集开始时的状态没有变化，则不提升，返回 false。

#### 2.2.2 片段 2

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f0ab2523787e48b0a17058e43aa83f2d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=0k70k8ROJFFY4CzhXXdXaUHk8Uo%3D)

1）、如果当前要提升的 WindowContainer 是 Wallpaper 类型的，则不提升，返回 false。

2）、如果当前 WindowContainer 前后的父 WindowContainer 不一致，即发生 reparent 了，则不提升，返回 false。

#### 2.2.3 片段 3

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/fa4a5ade52d043ceaba4813799ea5c0f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=GyVHUp%2Fmw%2FsFSSUWZ3Kaw5T5gkw%3D)

遍历父 WindowContainer 的所有子 WindowContainer：

1）、如果姊妹 WindowContainer 在 Transition.mChanges 中找不到一个对应的 ChangeInfo 对象，或者有这么一个 ChangeInfo 对象，但是该 ChangeInfo 对象不在 Targets.mArray 中，这种情况一共可以理解为这个姊妹 WindowContainer 没有参与到本次动画，那么还需要继续判断：

----1.1）、如果该姊妹 WindowContainer 可见，那么就不提升，直接返回 false，当前 WindowContainer 无法提升到父 WindowContainer。毕竟该姊妹 WindowContainer 是没有参与到动画中的，并且是可见的，如果你提升了，那后续动画执行的时候用户不是会看到该姊妹 WindowContainer 跟着一起动了嘛，这肯定是不对的。

----1.2）、如果该姊妹 WindowContainer 不可见，那么就跳过对这个 WindowContainer 的检查。不可见的姊妹 WindowContainer 对于本次动画也没有太大影响，即使跟着一起进行动画用户也看不到，直接跳过检查下一个姊妹 WindowContainer 就好了。

2）、如果姊妹 WindowContainer 从 Transition.mChanges 中能找到一个对应的 ChangeInfo 对象，并且该 ChangeInfo 对象也在局部变量 targets 中，那么认为该姊妹 WindowContainer 也参与了本次动画，那么分别为他们的 TransitionMode 调用 Transition.reduceMode 方法来看它们动画的大方向是否是一致的，首先是根据 ChangeInfo.getTransitMode 拿到各自的 TransitionMode：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d67497c670b64176b1d89c76d4770ba3~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=bWCBL%2FkFa%2FoDTRCmZ4WjbRPpeBk%3D)

TransitionMode 定义在 TransitionInfo 中：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4fab626910974cc2b9a1a532387e3b8b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=Ca1nG37%2B5jw%2FZel4Tx0GLulbVjc%3D)

看到 Transition 模式其实就是定义在 WindowManager 中的 Transition 类型的子集。

ChangeInfo.getTransitMode 的内容也比较简单：

TRANSIT_CHANGE：收集阶段的可见性和 Transition 就绪阶段的可见性没有发生变化。

TRANSIT_OPEN：存在发生了变化，且当前可见，即从无到有。

TRANSIT_CLOSE：存在发生了变化，且当前不可见，即从有到无。

TRANSIT_TO_FRONT：存在没有发生变化，且当前可见，说明从后台移动到了前台，从不可见变为了可见。

TRANSIT_TO_BACK：存在没有发生变化，且当前不可见，说明从前台移动到了后台，从可见变为了不可见。

再根据 Transition.reduceMode 的逻辑：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c3476682d7884bf587437f610dc5c099~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=tkfs265bV94Q0W1kN49JddRO568%3D)

*   TRANSIT_TO_BACK 和 TRANSIT_CLOSE 是一类的。

*   TRANSIT_TO_FRONT 和 TRANSIT_OPEN 是一类的。

*   TRANSIT_CHANGE 单独一类。

如果动画的大方向是一致的，那么即使 TRANSIT_TO_BACK 和 TRANSIT_CLOSE 的动画有点差别，但是为了大局考虑，各别同志也不是不能适当调整一下来实现集体上的一致。

如果动画的大方向都不一致，那么它们中的无论哪个肯定都是不能提升为它们的父容器的。比如 TaskA 想向左平移，TaskB 想向右平移，那么如果擅自提升为父容器 TaskDisplayArea，不管 TaskDisplayArea 向左还是向右平移肯定都不合适，这种矛盾就属于不可调和了，那父容器 TaskDisplayArea 就不用管了，也就是别提升了，让冲突的 TaskA 和 TaskB 自己玩去吧。

3）、最后总结一下检查姊妹 WindowContainer 的这段逻辑，其实就是检查所有的姊妹 WindowContainer 中，有没有和当前 WindowContainer 冲突的姊妹 WindowContainer，至于是否冲突则看是否满足了以下条件之一：

*   检查所有没有参与动画的姊妹 WindowContainer，看能否找到一个可见的。

*   检查所有参与了动画的姊妹 WindowContainer，看能否找到一个动画的大方向和当前 WindowContainer 不一致。

只要找到了这么一个姊妹 WindowContainer，我们就无法提升动画的主体。

3 构建 TransitionInfo 对象
----------------------

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1bfc44747480458493979cddc033dbeb~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=cmX2vEsEGvSwb8pt9ht5s0suHvM%3D)

这一节我感觉其实没有什么好说的，大概介绍一下 TransitionInfo 以及它的内部类 Change。

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e70d72acb3c0497d81956e5a6dce187b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=2soJaqfD8pDT3jnMYXLD1DbedZU%3D)

1）、TransitionInfo，实现了 Parcelable，结合注释，用来收集 WMCore 这边的 Transition 信息，用来同步给 WMShell 的 TransitionPlayer。成员变量大概有这些：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1584cf7951ec417780a23c1f5e3ba656~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=vSbe8ePZCiHMpXzogJeTg9IwuYo%3D)

2）、TransitionInfo.Change，同样实现了 Parcelable，代表了 WindowContainer 在一个 Transition 期间的变化。看其成员变量，保存的信息还是挺多的，还有一个 RunningTaskInfo 的对象：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/10a5dce767a84605893356375be69486~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=5aJLLX87VNlDmcQ2iXMxB%2Fofca8%3D)

再结合 Transition.calculateTransitionInfo 方法，很明显就大概能弄懂这两个类的作用：

1）、TransitionInfo，对应一个 Transition 对象，用来收集 WMShell 感兴趣的 Transition 的信息，后续同步给 WMShell。

2）、TransitionInfo.Change，对应一个 Transition.ChangeInfo 对象，用来收集 WMShell 感兴趣的 Transition.ChangeInfo 的信息，后续同步给 WMShell。

顺便一提，google 为啥不将 Transition 中 ChangeInfo 的命名为”Change“，将 TransitionInfo 中的 Change 命名为”ChangeInfo“呢，强迫症犯了。

4 Transition 移动到 PLAYING 状态
---------------------------

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d2b0f77dd2d2454fb7132902ba8d9e43~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=MiOHDsq8kLNUciqYRiSu7qGgMSw%3D)

其实在 Transition.onTransactionReady 方法的开头已经将 Transition.mState 状态置为 STATE_PLAYING，这里又调用了一个 TransitionController.moveToPlaying 方法，看下是干啥的：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/96a17a5755104916b3daa0e1d5450a96~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=h8N8eMMTdDcKIzt1suSuYavf7xo%3D)

其实也非常简单：

1）、开始动画了，意味着当前 Transition 已经不能收集了，所以将 TransitionController.mCollectingTransition 置空。特别的，如果有其它 Transition 在排队，那么就继续将 TransitionController.mCollectingTransition 赋值为排队队列队首的那个 Transition，我播我的动画，你收集你的 WindowContainer，互不干扰。

2）、将当前 Transition 添加到 TransitionController.mPlayingTransitions：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/be473b4a1ff84f0e81e22a3f43270857~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=%2FwGsI%2Fp6TaOW7eegrtO6j5gNZWE%3D)

一个当前处于 playing 状态的 Transition 的队列，也就是说 playing 的 Transition 可以有多个。

5 切换到 WMShell：onTransitionReady
-------------------------------

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f60ee818bcf9414caefce17dd7fdfeaf~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=%2FvJwnlVExitZJDa7ZdtNgjXX%2B1s%3D)

在 Transition.onTransactionReady 方法的最后，调用了 ITransitionPlayer.onTransitionReady 方法将切换到了 WMShell：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/78f296b9c0774370a4bb6c1af4c67a9c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=xUjqD3bBjZ8q1kyab9PBysTaY4I%3D)

切换到 WMShell 意味着 Transition 就绪阶段已经结束，正式进入 Transition 的 playing 阶段，Transitions.TransitionPlayerImpl.onTransitionReady 就是我们下一篇文章的起点。

最后稍微看一下调用 ITransitionPlayer.onTransitionReady 方法之前调用的 Transition.buildFinishTransaction 方法：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/182fe91d49a04ca0926caecfafff07f5~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749527419&x-signature=w4vjiDiezag0FAQr%2Fu98uJ%2FOW3E%3D)

传入的 Transaction 对象为 Transition.mFinishTransaction，如该方法的注释所说，这里对”finish transaction“的操作保证了动画结束后，所有的”reparent“操作或者是 Layer 的变化将会得到重置，特别是 Layer 的几何信息（位置、缩放、旋转这些）。如果你的 Layer 在动画结束的时候在 Layer 的这些信息上的确有变化，那就要注意不要让这个方法把你对 Layer 的操作重置了。
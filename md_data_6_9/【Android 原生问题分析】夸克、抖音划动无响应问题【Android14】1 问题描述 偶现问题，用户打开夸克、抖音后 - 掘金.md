> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7437904276918321152)

> 1 问题描述 偶现问题，用户打开夸克、抖音后，在界面上划动无响应，但是没有 ANR。回到 Launcher 后再次打开夸克 / 抖音，发现 App 的界面发生了变化，但是仍然是划不动的。 2 log 初分析 复现问题

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/891b17d4bd5849239058cacd4892a2ee~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=l8ROOuKapf4AZHdY2ANUodZVun4%3D)

1 问题描述
------

偶现问题，用户打开夸克、抖音后，在界面上划动无响应，但是没有 ANR。回到 Launcher 后再次打开夸克 / 抖音，发现 App 的界面发生了变化，但是仍然是划不动的。

2 log 初分析
---------

复现问题附近的 log 为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3028712ba76d4526857df4d14a5cbe52~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=r%2FUj%2F3mwSvd5TDFyyFtbpVkRAOI%3D)

用户打开夸克的时间点为 “00:27:25” 左右，然后是持续的划动无响应，最后在 “00:27:36” 最后退出了夸克。

现在基本上已经没有冻屏的操作了，从 log 中也没有看到可能的痕迹，并且根据用户的描述，在夸克界面上划动无响应，切换到别的界面后再切回夸克，看到夸克的界面是有更新的，说明应该是在绘制或者合成的地方卡住了。

由于默认的 log 有限，无法定位到具体的问题，因此需要添加 log。

3 第一次添加 log
-----------

由于不确定是哪个阶段出了问题，因此先针对 Input 流程和 measure、layout 以及 draw 流程添加了 log，后面测试再次复现问题后，查看 log 发现问题出在 draw 流程（其实从问题描述上也能看出来，从 App 切换到 Launcher 再切换回 App，界面发生了变化，说明输入事件应该是被 App 正确接收到了）。

第一帧正常：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7c5db3ad4aba4f8e8db0fa0336f6119a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=URnJwBBSBAwZ0iK9PqH5FH2WUf0%3D)

从第二帧开始异常：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/128ed765124f4ac584384896a0df9ba6~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=w4IsuwwLT0b2otuHhhw6Ez5IRjs%3D)

异常的点在 ViewRootImpl.performDraw：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/cf48eaab8ea64da7952d5a228bb1a4d1~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=s7dV74TbaNGkY2XBFNIjHF%2Fsq3Q%3D)

这里就是问题的直接原因了，明明是亮屏状态，但是 ViewRootImpl.mAttachInfo.mDisplayState 保存的值却是 Display.STATE_OFF，导致代码误判断此时处于灭屏状态，因此不会继续走绘制流程。

至于为什么第一帧有没问题，而从第二帧开始就出问题了，因为第一帧的时候，ViewRootImpl.mReportNextDraw 为 true，所以没有提前 return，可以继续走 draw 流程。而从第二帧开始，ViewRootImpl.mReportNextDraw 就变成了 false，导致代码提前返回。

搜索代码，AttachInfo.mDisplayState 赋值的地方只有四处：

~1、View.dispatchMovedToDisplay~

~2、ViewRootImpl.onMovedToDisplay~

~3、ViewRootImpl.setView~

4、ViewRootImpl.mDisplayListener.onDisplayChanged

前两处是跟移动到新的 Display 相关的，可以排除，第 3 处是窗口首次添加，也可以排除，那么就只有第 4 处是 Display 状态改变后，更新 AttachInfo.mDisplayState 的地方。

另外根据本地调试也的确如此，当发生亮灭屏时，触发的是 ViewRootImpl.mDisplayListener.onDisplayChanged：

DisplayManagerGlobal.DisplayManagerCallback.onDisplayEvent

-> DisplayManagerGlobal.handleDisplayEvent

-> DisplayManagerGlobal.DisplayListenerDelegate.sendDisplayEvent

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/88e1f207842f4b228edd0bc09519fe5e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=TcW0nVJ9Z9FCLR7x1AUa0bKE6tA%3D)

最终的调用起点在系统进程的 DisplayManagerService。

由于之前对 DisplayManagerService 了解比较少，所以还不清楚问题出在哪儿，这次加的 log 不太够，需要继续在 DisplayManagerService 处加 log，然后让测试帮忙去复现问题。

但是同时别的同事也合入了几个 google 的 patch，导致后续问题复现不到了，这个问题一度没有了下文，当时觉得还挺可惜，花了挺长的时间去分析，结果后面复现不到了。

4 第二次添加 log
-----------

虽然之前的问题不了了之了，但是后面别的项目又继续复现了该问题，希望又被点燃了...... 先在 DisplayManagerService 中添加一些 log，然后继续让测试同事帮忙复现，最终成功复现到了问题。

以下是当时的 log 分析：

#### 4.1 11-12 22:25:14 - 第一次启动抖音

：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/17223d44d9e44b98a6d0d366a02d6532~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=9v1rFCTHgYMY59mHf7s1mfF%2Frn0%3D)

这个是后续发生问题的抖音界面第一次启动的时间，此时是正常的。

中间又多次启动抖音以及亮灭屏，但是都没有问题，所以我们跳过这些阶段。

直接看发生问题前的那次启动抖音的情况。

#### 4.2 11-13 10:33:39 - 再次启动抖音

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a761cfacb5404f3b92959dc540d578f4~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=A5EsKu7hLVUlxg%2FPdkwFCIfQNwA%3D)

ViewRootImpl 保存的 DisplayState 已经是 2 了，即 Display.STATE_ON，这次绘制没有问题。

#### 4.3 11-13 10:53:42 - 灭屏

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/bd38062b5a73472082a5ff6611311744~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=qrNjfYS1jKADFyylm5kNhkT9%2FFM%3D)

此次 DisplayManagerService 向 SplashActivity 对应进程 “17885” 进行了通知。

最终抖音对应进程 “17885” 也打印了相应的 onDisplayChanged 的 log。

#### 4.4 11-13 10:55:07 - 亮屏

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c541a8b69fca4afe80ae884b5e8c0f58~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=XoIlNJlm1jH8bV91Gt9nMdBbCAA%3D)

DisplayManagerService 没有通知抖音对应进程 “17885”，有以下 log 打印：

```
11-13 10:55:07.252351  1450  1575 I DisplayManagerService: DisplayManagerService.deliverDisplayEvent: isUidCached=10224
11-13 10:55:07.252761  1450  1575 I DisplayManagerService: DisplayManagerService.deliverDisplayEvent: isUidCached=10224
11-13 10:55:07.253925  1450  1575 D DisplayManagerService: Ignore redundant display event 0/2 to 10224/16998
11-13 10:55:07.253967  1450  1575 I DisplayManagerService: DisplayManagerService.deliverDisplayEvent: isUidCached=10224
11-13 10:55:07.253978  1450  1575 D DisplayManagerService: Ignore redundant display event 0/2 to 10224/16998


```

#### 4.5 11-13 10:55:16 - 灭屏

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/cc54c4ec4e6946fcb6fec853bf43e44a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=wTP9bX4lbB22vKp43HVnDscTvsc%3D)

DisplayManagerService 没有通知抖音对应进程 “17885”。

#### 4.6 11-13 13:00:00 - 出现问题前的最后一次亮屏

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ad0dd8ba92604349bc17137bd0647ac2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=RmLNGJN84cdsM%2Ftpm7flAoVGxGs%3D)

DisplayManagerService 还是没有通知抖音对应进程 “17885”。

#### 4.7 11-13 13:24:18 - 再次启动抖音，有问题

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/73727d31ee7e4b7185c046b8809901f2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=uxy%2FOn1jGI50Mc8fWQ9PbDDeJr4%3D)

原因则是 “11-13 10:53:42” 那次灭屏 DisplayManagerService 通知到了抖音对应进程“17885”，随后不管是亮屏还是灭屏都没有再通知抖音进程，导致此次启动抖音 SplashActivity 后，其 ViewRootImpl 处保存的 DisplayState 还是 1，即 Display.STATE_OFF，灭屏状态，所以不会走绘制流程。

接下来需要看下具体的代码，为什么没有通知到抖音进程，并且打印了这些 log：

```
11-13 10:55:07.252351  1450  1575 I DisplayManagerService: DisplayManagerService.deliverDisplayEvent: isUidCached=10224
11-13 10:55:07.252761  1450  1575 I DisplayManagerService: DisplayManagerService.deliverDisplayEvent: isUidCached=10224
11-13 10:55:07.253925  1450  1575 D DisplayManagerService: Ignore redundant display event 0/2 to 10224/16998
11-13 10:55:07.253967  1450  1575 I DisplayManagerService: DisplayManagerService.deliverDisplayEvent: isUidCached=10224
11-13 10:55:07.253978  1450  1575 D DisplayManagerService: Ignore redundant display event 0/2 to 10224/16998


```

5 代码分析
------

#### 5.1 App 侧注册 display 状态的监听

ViewRootImpl 有一个 DisplayListener 的成员变量 mDisplayListener，该成员变量用来接收 DisplayManagerService 关于 Display 状态改变的通知：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1c5b7e6498a74c8396f7d9d466102f25~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=MfhO1PKV4NQYf4CyvYMNhDqDXtA%3D)

当 onDisplayChanged 回调触发，会更新 mAttachInfo.mDisplayState 的值。

该成员变量通过 DisplayManagerGlobal.registerDisplayListener 进行注册，最终是调用了 IDisplayManager.registerCallbackWithEventMask，跨进程在 DisplayManagerService 进行了注册，如下图：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ad086eb5a8934be89e3df742f2b6f569~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=03FvrOGBCPPhwB8CRENyV33SgFY%3D)

DisplayManagerService 内部类 BinderService 是 IDisplayManager 的本地实现，然后 BinderService.registerCallbackWithEventMask 又调用了 DisplayManagerService.registerCallbackInternal：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d59f366414fa43d0b60d09f2a07d7370~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=VDkfmwkHAaYVgME9afU7rfbRM8Q%3D)

最终，DisplayManagerService 为每一个通过 IDisplayManager.registerCallbackWithEventMask 监听 Display 状态的客户端，都创建了一个 CallbackRecord 对象，用来代表一个客户端记录，该 CallbackRecord 被保存在 DisplayManagerService 的成员变量 mCallbacks 中。

至于 CallbackRecord，其内部保存了客户端的 pid 和 uid，因此可以唯一对应一个客户端。

#### 5.2 DisplayManagerService 通知客户端 Display 状态改变

直接放结论，在 CallbackRecord.notifyDisplayEventAsync 中：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/605a2f75a92c4e1cba4b322aabeaccd0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=BtkpyBHlroVyxlZU%2F1Kw2maPZMw%3D)

CallbackRecord 中的 IDisplayManagerCallback 类型的成员变量 mCallback 保存的是客户端进程传过来的 IDisplayManagerCallback 代理，因此调用 IDisplayManagerCallback.onDisplayEvent 最终可以调用到客户端进程的 DisplayManagerGlobal：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/753e0622d1e148ee8cdf6383874ca534~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=JuYGaI9TmUJOFBfR74OSbsTHVT4%3D)

最终通知到 ViewRootImpl。

CallbackRecord.notifyDisplayEventAsync 方法调用的地方有两处，先看第一处，在 DisplayManagerService.deliverDisplayEvent：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/9c0a088fc2864878af6382c4f2af1470~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=teoegi1nmZb%2FHKnyvHHXHk6LZ6c%3D)

这里会将 DisplayManagerService.mCallbacks 的数据拷贝到 DisplayManagerService.mTempCallbacks，然后调用 DisplayManagerService.isUidCached 方法来判断需要通知的进程是否处于 cached mode：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f6751d8748834bfab8ae584f478a2a65~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=kL3srLNhCf7o4ILk6mnaN8eywtk%3D)

如果不是，那么说明此时该客户端进程的优先级比较高，需要直接调用 CallbackRecord.notifyDisplayEventAsync 来通知客户端 Display 状态改变的消息。

如果客户端进程处于 cached mode，那么说明该进程处于一个优先级较低的状态，因此不会立即调用 CallbackRecord.notifyDisplayEventAsync，而是创建一个 PendingCallback 对象，然后保存到 DisplayManagerService 的 mPendingCallbackSelfLocked 成员变量中，注意这里 mPendingCallbackSelfLocked 是根据 uid 去区分的，并非是 pid。

等到客户端进程的进程优先级提高后，再去从 DisplayManagerService.mPendingCallbackSelfLocked 中取出相应的 CallbackRecord，然后调用 CallbackRecord.notifyDisplayEventAsync 方法通知客户端进程，这也就是第二处调用 CallbackRecord.notifyDisplayEventAsync 的地方，在 PendingCallback.sendPendingDisplayEvent：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/05c69df303f24fe782713212dcee7406~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=0vemu2X7nh6U%2Bdiv07MpdMIrHaI%3D)

而 PendingCallback.sendPendingDisplayEvent 方法则由 UidImportanceListener.onUidImportance 方法调用。

这里的逻辑很好理解，当进程的优先级改变的时候，会回调 UidImportanceListener.onUidImportance，然后 UidImportanceListener.onUidImportance 中继续判断该进程的优先级，如果优先级高于 IMPORTANCE_CACHED，那么会继续根据该进程的 uid 从 DisplayManagerService.mPendingCallbackSelfLocked 中找其对应的 PendingCallback，如果能找到，那么调用 PendingCallback.sendPendingDisplayEvent，最终在 PendingCallback.sendPendingDisplayEvent 中调用 CallbackRecord.notifyDisplayEventAsync 来通知客户端 Display 状态的改变。

#### 5.3 省流版

概括一点说，就是并不是每次 Display 状态发生变化的时候，DisplayMangerService 就会去通知抖音进程，而是会调用 DisplayManagerService.isUidCached 方法来看这个 App 进程的优先级是否足够高：

1）、如果优先级高于 IMPORTANCE_CACHED，说明此时优先级还是挺高的，那么 DisplayManagerService 会立即通知抖音进程。

2）、如果优先级等于或者低于 IMPORTANCE_CACHED，说明优先级不够高，那么 DisplayManagerService 不会立即通知抖音进程，而是为其创建一个记录，等到抖音进程的优先级改变并且高于 IMPORTANCE_CACHED 的时候，根据这个记录找到抖音进程，然后通知它。

6 原因分析
------

1）、首先是这次：”11-13 10:33:39 - 再次启动抖音 “

这是离问题发生时间点最近的那次启动抖音。

2）、然后接着：”11-13 10:53:42 - 灭屏 “

看到这次灭屏的时候，可能是由于距离启动抖音的时间不长，因此这时抖音进程的优先级还比较高，所以这次灭屏的时候，通知到了抖音进程 17885。

3）、接着是”11-13 10:55:07 - 亮屏 “

这里我们之前的分析有误，看了代码后才了解，不是说 DisplayManagerService 没有通知抖音进程 17885，而是抖音进程的优先级比较低，所以没有立即通知抖音进程，而是为其创建了 PendingCallback，等待抖音进程优先级提高后，再去通知抖音进程，如这里的 log：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1ea5d001b2fa437c87725320343835cd~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=u%2FG0MfmzpysQOxME8npqjYF987E%3D)

但是这里的逻辑有问题。

如 log 中显示的，这里抖音 App 的 uid 为 10224，有三个进程，16998，17885，18213，我们关注的抖音的”SplashActivity” 所在进程是 17885.

然后再看代码：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ebec2308221d4e52951456c740c64113~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=XsoIc1u0NL3We9wSSuF1dpldY5w%3D)

首先对于抖音进程 16998，由于这是遍历到的抖音的第一个进程，因此 DisplayManagerService.mPendingCallbackSelfLocked 中是没有抖音 uid 的信息的，所以这里会基于进程 16998 对应的 CallbackRecord 创建一个 PendingCallback 对象，然后保存到 DisplayManagerService.mPendingCallbackSelfLocked 中。

然后遍历到抖音进程 17885 的时候，由于它和抖音进程 16998 的 uid 是一样的，都是 10224，那么这里就不会为进程 17885 再去创建一个 PendingCallback 对象， 而是直接复用上一步创建的那个 PendingCallback，如这里的 log 打印：

”Ignore redundant display event......“

那么这里就是问题的根因所在了，抖音有三个进程，这里只为抖音进程 16998 对应的 CallbackRecord 创建了一个 PendingCallback，到后续抖音进程优先级提高的时候，那么也只会调用该 PendingCallback 中的 CallbackRecord 的 notifyDisplayEventAsync 方法，那么这意味着只有进程 16998 会得到 Display 状态改变的通知，进程 17885 和进程 18213 都无法得到通知，再看后面抖音进程优先级提升的情况：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/602ca0eaf59a4d10b20e17b266140537~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749534051&x-signature=UFLXGVrhXKI4d%2FaI3q31cBfjJ%2Bc%3D)

看到果然只有进程 16998 得到通知了，我们关注的进程 17885 没有被通知到，那么进程 17885 的 ViewRootImpl 保存的 mAttachInfo.mDisplayState 将一直是灭屏的状态。

#### 不太省流的省流版

目前的代码，在一个 App 对应一个 uid，以及一个 pid 的情况下，是够用的，但是实际上一个 App 对应了一个 uid，但是可能会有多个进程，即一个 uid 会对应多个 pid。

再回看我们的代码总结：如果 App 优先级等于或者低于 IMPORTANCE_CACHED，说明优先级不够高，那么 DisplayManagerService 不会立即通知抖音进程，而是为其创建一个记录，等到抖音进程的优先级改变并且高于 IMPORTANCE_CACHED 的时候，根据这个记录找到抖音进程，然后通知它。

这里创建的记录，是根据 uid 去创建的，那么即使抖音有多个进程，也只会创建一个记录，根据遍历顺序来看则是 pid 最小的那个。

再看问题场景：

1）、启动完抖音后灭屏，此时抖音的优先级比较高，因此 DisplayMangerService 会立即通知抖音进程，这里抖音的三个进程都能通知到，因此进程 17885 的 ViewRootImpl 处保存的 Display 状态就是灭屏状态。

2）、过了一阵子再次亮屏，由于抖音的优先级降低了，所以 DisplayManagerService 没有立即通知抖音进程，而是创建为抖音创建了一个记录，等到抖音优先级提高的时候，再去通知抖音进程。此时抖音有三个进程，16998，17885，18213，这里只为 pid 最小的 16998 创建了一项纪录。

3）、后续抖音优先级提高的时候，DisplayMangerService 只通知了进程 16998，由于没有为剩下的两个进程创建记录，因此这两个进程就通知不到了，所以我们关注的进程 17885 出了问题。

从以上分析可以知道，我们需要为每一个 pid 都创建一个记录，而非每一个 uid。

7 问题解决
------

最终看到已经有 google patch 解决这个问题了：

[Diff - f2daa46634f6a1e5e329041f07a27dbc894d71b2^! - platform/frameworks/base - Git at Google](https://link.juejin.cn/?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fframeworks%2Fbase%2F%2B%2Ff2daa46634f6a1e5e329041f07a27dbc894d71b2%255E%2521%2F%23F0 "https://android.googlesource.com/platform/frameworks/base/+/f2daa46634f6a1e5e329041f07a27dbc894d71b2%5E%21/#F0")

```
Correct the deferred sending of display events.

When multiple processes sharing the same uid become non-cached, the
current deferred transmission logic does not consider all of them,
leading to only the first process in the callback list to receive the
pending display events. Moreover, this first process may inadvertently
receive unintended display events.

Bug: 325692442
Test: manual, see bug description

Change-Id: Ife98a8d843a3000294fbc9929bb78a755534b5dc
Signed-off-by: Hyeongseop Shim <hyeongseop.shim@samsung.corp-partner.google.com>






@@ -464,10 +464,11 @@
     // May be used outside of the lock but only on the handler thread.
     private final ArrayList<CallbackRecord> mTempCallbacks = new ArrayList<CallbackRecord>();
 
-    // Pending callback records indexed by calling process uid.
+    // Pending callback records indexed by calling process uid and pid.
     // Must be used outside of the lock mSyncRoot and should be selflocked.
     @GuardedBy("mPendingCallbackSelfLocked")
-    public final SparseArray<PendingCallback> mPendingCallbackSelfLocked = new SparseArray<>();
+    public final SparseArray<SparseArray<PendingCallback>> mPendingCallbackSelfLocked =
+            new SparseArray<>();
 
     // Temporary viewports, used when sending new viewport information to the
     // input system.  May be used outside of the lock but only on the handler thread.
@@ -1011,8 +1012,8 @@
                 }
 
                 // Do we care about this uid?
-                PendingCallback pendingCallback = mPendingCallbackSelfLocked.get(uid);
-                if (pendingCallback == null) {
+                SparseArray<PendingCallback> pendingCallbacks = mPendingCallbackSelfLocked.get(uid);
+                if (pendingCallbacks == null) {
                     return;
                 }
 
@@ -1020,7 +1021,12 @@
                 if (DEBUG) {
                     Slog.d(TAG, "Uid " + uid + " becomes " + importance);
                 }
-                pendingCallback.sendPendingDisplayEvent();
+                for (int i = 0; i < pendingCallbacks.size(); i++) {
+                    PendingCallback pendingCallback = pendingCallbacks.valueAt(i);
+                    if (pendingCallback != null) {
+                        pendingCallback.sendPendingDisplayEvent();
+                    }
+                }
                 mPendingCallbackSelfLocked.delete(uid);
             }
         }
@@ -3193,16 +3199,23 @@
         for (int i = 0; i < mTempCallbacks.size(); i++) {
             CallbackRecord callbackRecord = mTempCallbacks.get(i);
             final int uid = callbackRecord.mUid;
+            final int pid = callbackRecord.mPid;
             if (isUidCached(uid)) {
                 // For cached apps, save the pending event until it becomes non-cached
                 synchronized (mPendingCallbackSelfLocked) {
-                    PendingCallback pendingCallback = mPendingCallbackSelfLocked.get(uid);
+                    SparseArray<PendingCallback> pendingCallbacks = mPendingCallbackSelfLocked.get(
+                            uid);
                     if (extraLogging(callbackRecord.mPackageName)) {
-                        Slog.i(TAG,
-                                "Uid is cached: " + uid + ", pendingCallback: " + pendingCallback);
+                        Slog.i(TAG, "Uid is cached: " + uid
+                                + ", pendingCallbacks: " + pendingCallbacks);
                     }
+                    if (pendingCallbacks == null) {
+                        pendingCallbacks = new SparseArray<>();
+                        mPendingCallbackSelfLocked.put(uid, pendingCallbacks);
+                    }
+                    PendingCallback pendingCallback = pendingCallbacks.get(pid);
                     if (pendingCallback == null) {
-                        mPendingCallbackSelfLocked.put(uid,
+                        pendingCallbacks.put(pid,
                                 new PendingCallback(callbackRecord, displayId, event));
                     } else {
                         pendingCallback.addDisplayEvent(displayId, event);


```

看注释，说的就是多个进程共享同一个 uid 的情况。

再看修改，将原来的：

```
public final SparseArray<PendingCallback> mPendingCallbackSelfLocked = new SparseArray<>();


```

改为了：

```
   public final SparseArray<SparseArray<PendingCallback>> mPendingCallbackSelfLocked =
           new SparseArray<>();


```

很明显，之前是一个 uid 对应一个 PendingCallback 对象，现在是一个 uid 对应一组 PendingCallback 对象，即一个 uid 对应多个进程的情况，这样同一个 uid 下多个进程都能得到通知了，代码比较简单，不再赘述。
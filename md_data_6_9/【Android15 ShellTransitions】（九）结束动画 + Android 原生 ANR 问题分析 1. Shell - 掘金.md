> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7486664338889326646)

> 1. ShellTransitions 2. finishTransitiion 结束动画 3. google 原生 ANR 问题分析 4. ActivityRecord 可见性分析微信图片 \ 2024031310......

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4136d982d4b4405b89d32cc34df824b9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=9yjqEkASEhvujBpHcFsA10sYhZs%3D) finishTransition 这部分的内容不多，并且我个人的实际工作中很少接触这块，因此我之前都觉得没有必要专门开一篇去分析最后留下的这一丁点儿的动画流程。但是最近碰到了一个 google 原生 ANR 问题，正好是和这块相关的，也让我意识到了 finishTransition 流程中涉及到 ActivityRecord 的可见性设置也挺重要的，此次正是一个补完 ShellTransitions 流程的机会，顺便梳理一下 ActivityRecord 可见性这块的逻辑。

1 google 原生 ANR 问题
------------------

此部分包含了当初问题时的一些思路和对一些问题的思考，有些可能不对，欢迎大家探讨。

### 1.1 log 初步分析

log 就不贴全部了，太多了，大概总结一下问题的上下文。

1、”02:06:58.994“，在 Settings 界面开始侧划，从后续的信息看到这次侧划最终没有触发 App 切换，也就是说最终还是停在了 Settings。

2、”02:06:59.005“，几乎是同一个时间，启动了”ConnectedDeviceDashboardActivity“，然后创建了 OPEN 类型的 Transition#48962.

3、”02:06:59.008“，侧划对应的 Transition#48963 也被创建，但是由于之前已经创建了一个 Transition#48962 了，因此这个 Transition 进入等待队列。

4、”02:06:59.030“，第二个创建的 Transition#48963 满足条件，开始 collecting，而 Transition#48962 开始等待。

5、”02:06:59.299“，侧划结束，应该是回到了 Settings，没有切换到别的 App。

6、”02:06:59.242“，焦点进入”recents_animation_input_consumer“。

7、”02:06:59.298“，不知道什么原因，”ConnectedDeviceDashboardActivity“主动调用了 finish，因此创建了 CLOSE 类型的 Transition#489645。

8、”02:06:59.405“，”Settings“在 WMS 侧被 resume。

9、最后三个 Transition 结束的顺序是 48963，48962，48964：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/0969a3dcdd6543d884384fa58e027257~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=JTsD0oN52yWOewF1aa%2Baf9RH0uA%3D)

再看下这几个 Transition 的内容：

1）、Transition#48963：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a143fb69f77c4118b5bd09f51ca1214f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=SPxhPg4uawtS8XU11fzZn0SFh08%3D)

对应侧划启动 Recents，即从 Settings 切到 Recents，结束的时候 Settings 的 Task 会被 hide，但是在 RecentsTransitionHandler 可能会重新 show。

2）、Transition#48962：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1308a9d726ea40ac8a07dadf3e3f5c61~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=UUJGfBS%2BBXaJYIhs%2FED6VhD5XG8%3D)

对应 Settings App 内部在”Settings“上启动”ConnectedDeviceDashboardActivity“，结束的时候”Settings“的 ActivityRecord 会被 hide，”ConnectedDeviceDashboardActivity“的 Activity 被 show。

3）、Transition#48964：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/15db835ecd42456dac816e6314799e4e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=6hUuAfs0yvHo072WoxITphJmiHY%3D)

对应”ConnectedDeviceDashboardActivity“的主动调用 finish，结束的时候 Settings 的 Task 会被 hide。

这里就是异常点了，按照一般的逻辑，”ConnectedDeviceDashboardActivity“的 Activity 主动调用 finish，那么”Settings“对应的 Activity 应该被置为可见啊，为什么这里是整个 Settings 对应的 Task 都被认为是不可见呢？

原来”ConnectedDeviceDashboardActivity“的主动调用 finish 的时候，Recents 的 Task 还盖在 Settings 的 Task 上，因此”Settings“对应的 Activity 也不会被 resume 或者置为可见，因此整个 Settings 的 Task 都会被认为是不可见的，导致这里为 Settings 的 Task 计算出的动画类型为 TO_BACK，最终当 Transition#48964 的 finish transaction 被 apply 的时候，Settings 对应的 Task 的 SurfaceControl 将会被隐藏，那么 Settings 的所有窗口都会被认为是不可见，因此无法作为焦点窗口，从而发生 ANR。

### 1.2 复现 ANR

基于之前对 ANR 上下文的分析，我们可以试着写一个 Demo 来复现该 ANR，本地测试下来复现的概率还是比较高的。

*   首先写一个 MainActivity，对这个 Activity 没有要求，只需要一个按钮来启动后续的 Activity 即可。

*   然后再写一个 CleanActivity，从 log 上看，发生侧划的时候”ConnectedDeviceDashboardActivity“主动调用了 finish，那么我们也写一个大概的逻辑，保证发生侧划的时候 CleanActivity 调用 finish：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6a38ca5ec55e4cd0ab64b2b309b80dbc~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=8uWQA7Bsp%2B8fr%2FNnBtIXqrcfJP0%3D)

最后复现步骤为：

1）、启动 MainActivity。

2）、然后点击 MainActivity 的 Button 启动 CleanActivity。

3）、CleanActivity 启动的同时，马上在屏幕底部向左侧划，就会高概率复现问题。

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c0cfa6191ae34237bd0268effab5cda8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=ZM3iHebbfalwQOyylcrP25bekO8%3D)

从 log 上看，和 Monkey 的复现场景有一点不同：

1）、Transition#1，对应从 MainActivity 上启动 CleanActivity，最终 MainActivity 的 SurfaceControl 将会被隐藏，CleanActivity 的 SurfaceControl 会被显示。

2）、Transition#2，对应从 DemoApp 切换到 Recents，最终会重新回到 DemoApp 上，此次 Transition 没有影响。

问题 log 中还有一个 Transition#3，对应将 DemoApp 对应的 Task 的 SurfaceControl 隐藏，但是实际上有没有这次，都会有问题，因为 Transition#1 在结束的时候，MainActivity 的 SurfaceControl 已经被隐藏了，并且后续没有地方让其重新显示，因此及时其所在的 Task 的 SurfaceControl 显示，也不会影响到 MainActivity。

最后同样的 Demo 在 pixel 上也能复现：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ccf8a61173ee452495e47eb1bd0b25d6~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=UEhhI4xOYDdR6Q%2FgOFJi0DRJlHs%3D)

从 pixel 复现的 log 上看，同样和 Monkey 的复现场景非常相似。

下一步需要添加更多 log，看下是否原因是否如我们之前所说的跟 Transition finish 的顺序有关，以及寻找解决方案。

### 1.3 更多 log 论证

首先是 transition 相关的 log：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7bd439149c004e60ba31de79b7953169~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=dJDLunw%2B5Y6qFBWpnWtmsgbH3Qs%3D)

1）、Transition#17，对应从 MainActivity 启动 CleanActivity，最终结果是 MainActivity 被隐藏，CleanActivity 显示。

2）、Transition#18：对应侧划操作，最终还是回到了我们的 DemoApp（Monkey 那次则是 Settings），所以最终还是 DemoApp（Settings）对应的 Task 显示，Launcher 的 Task 隐藏。

最后是 SurfaceFlinger 的 log：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/44240daae1f64fd89fc1b086889f170f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=zhG%2B6noSk8wEPd8PWUcJI6ogaaQ%3D)

1）、Transition#17 的 finishT 被 apply 的前一次 composite，MainActivity 还是可见的。

2）、Transition#17 的 finishT 进行 apply。

3）、下一次 composite，MainActivity 已经不可见了。

4）、MainActivity 的 Layer 不可见，其窗口也无法再获得焦点。

这里证明了的确是从 MainActivity 启动 CleanActivity 触发的那次 Transition 的 finish transaction 被 apply 导致的。

### 1.4 finish 那次缺少 Transition？

回看问题发生的整个流程，发现 CleanActivity 调用 finish 回到 MainActivity，这次操作没有创建相应的 Transition 让 MainActivity 的 Surface 重新显示。

那么问题来了，如果 Clean 调用 finish 的时候创建相应 Transition 让 MainActivity 重新显示，那么是不是就没问题了？

还记得上面我们提到的我本地复现的 ANR 情况和实际 monkey 跑出的 ANR 情况的差异吗？

我本地用 Demo 复现的 ANR 场景，只有两次 Transition，对应：

1）、Transition#1：从”MainActivity“启动”CleanActivity“。

2）、Transition#2，发生侧划。

然后 Monkey 的场景，其实是有三次的：

1）、Transition#48962：”Settings“Activity 启动”ConnectedDeviceDashboardActivity“ Activity。

2）、Transition#48963：发生侧划。

3）、Transition#48964：”ConnectedDeviceDashboardActivity“Activity 调用 finish，回到”Settings“ Activity。

并且 Transition 结束的顺序是 Transition#48963，Transition#48962，Transition#48964。

那为什么仍然有问题呢？看 Transition#48964 的详细信息：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/29d951cbf58b406d8a2f29aaad2f3d81~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=Cem%2FglW%2BLpOTmZx%2BIhNZ%2FlOCZXc%3D)

它只有一个 Settings 对应的 Task 被移动到后台的动画类型，并没有将”Settings“ Activity 重新 show 的操作，并且实际上 Settings 对应的 Task 也没有被移动到后台。

原因就如我们上面初步分析 log 时候提到的：原来”ConnectedDeviceDashboardActivity“的主动调用 finish 的时候，Recents 的 Task 还盖在 Settings 的 Task 上，因此”Settings“对应的 Activity 也不会被 resume 或者置为可见，因此整个 Settings 的 Task 都会被认为是不可见的，导致这里为 Settings 的 Task 计算出的动画类型为 TO_BACK，最终当 Transition#48964 的 finish transaction 被 apply 的时候，Settings 对应的 Task 的 SurfaceControl 将会被隐藏，那么 Settings 的所有窗口都会被认为是不可见，因此无法作为焦点窗口，从而发生 ANR。

因此我们的问题：

” 如果 Clean 调用 finish 的时候创建相应 Transition 让 MainActivity 重新显示，那么是不是就没问题了？“

这个显然是理想情况，即每个 Transition 都一步一步按照顺序，中间没有交叉，如果 Transition 相互交叉，有的快有的慢，那么仍然会有问题（就如 monkey 的情况）。

### 1.5 为什么我本地的 finsh 流程没有创建 Transition

然后先看下为什么我本地的问题场景中，那次 CleanActivity 主动调用 finish 的时候没有为它创建一个 Close 类型的 Transition：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/96676ec5490d4fd0847ce202cb8a6fed~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=v4EUIoAuXeOP%2Bl79dz4oa24qNmM%3D)

看代码，在 ActivityRecord.finishIfPossible：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/13a6bc2c1e7e4e5cb5c190f499039b48~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=AYkhnOmRSjzL4RMtq%2FgwAVan4jM%3D)

CleanActivity 走 finish 流程的时候，侧划操作对应的 Transition#18 还在 collect，所以 CleanActivity 被直接添加到 Transition#18 了。

如果我们强制去创建这个 Transition 呢，能解决问题吗？显然也是不能，上一节也分析过了。

### 1.6 Transaction apply 的顺序

本地两次 Transition，每个 Transition 有一个 startT 以及 finishT，因此总共是 4 个 Transaction，看下他们调用 show、hide 以及 apply 的顺序（show 和 hide 则主要针对 MainActivity，别的不考虑）。

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/8ebf569109f840c7a6b19d4689237687~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=J5lhhmL6wRaAMyXKakgPW%2B4L5XE%3D)

1）、Transition#1 的 finishT1.hide（Transitions.setupStartState 中）。

2）、Transition#1 的 startT1.apply：没什么影响。

3）、Transition#2 的 startT2.show（Transition.onTransactionReady）。

4）、Transition#2 的 startT2.apply（onAnimationStart，这个虽然没 log，但是顺序应该是没错的）。

5）、Transition#1 和 Transition#2 的 finishT2 调用 apply（谁先谁后没有关系）。

能看到 finishT1 的 apply 时间是晚于 startT2 的，所以 startT2 即使调用 show 也不能解决问题。这里也说明了，复现这个问题必须要快速操作（如果操作较慢，那么 startT2 的 apply 时间是晚于 finishT1 的 apply 的）。

但是 startT2 调用 show 这个操作我还是没想到的，看方法调用堆栈，在 Transition.onTransactionReady：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/bd90df9108fd49b4824cfd4ec5ba9ec2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=F9N%2B8lBV3C%2BRqDPP6H0sSK8%2ByFA%3D)

具体代码为：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/be289447f612451bb80c4ffa083543a0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=dOsc2umWhHuWUICCqcNf9PWlkKQ%3D)

看注释，也是为了解决类似的问题的，但是对于我们当前分析的问题来说，并没有效果，因为这里用到的是 startT，根据我们的分析，startT2 的 apply 是早于 finishT1 的 apply 的，因此这个代码并不能解决我们的问题。

### 1.7 根因总结

扯了这么多，可以总结根因了：

1）、ActivityRecord#1 启动 ActivityRecord#2，创建 Transition#1，在 Transitions#setupStartState 中的时候，Transition#1 的 finish transaction 会把 ActivityRecord#1 的 SurfaceControl 隐藏。

2）、发起一个短距离侧划，创建 Transition#2，从 DemoApp 切换到 Launcher，最终还是回到 DemoApp，当然此次 Transition 无所谓。

3）、当 Transition#1 和 Transition#2 还在 playing 的时候，ActivityRecord#2 主动调用 finish，ActivityRecord#1 又会变为 DemoApp 的 top activity。

4）、Transition#1 和 Transition#2 结束（顺序无所谓），此时 DemoApp 会重新变为前台 App，然后 ActivityRecord#1 会被 resume，但是 Transition#1 结束的时候，其 finish transaction 调用 apply，然后 ActivityRecord#1 的 SurfaceControl 真正被隐藏，从而其窗口也会被隐藏，无法作为焦点窗口。

### 1.8 思考

再精炼一下这个问题的关键点：

Activity#1 启动 Activity#2，创建了一个 Transition，然后在这个 Transition playing 的过程中，Activity#1 又被重新 resume（原因可能有多个，比如本题是因为 Activity#2 主动调用 finish），导致了后续 Activity#1 的 SurfaceControl 没有被 show 出来。

那么这类问题如何解决呢？

1、很明显，能够想到的第一个方法就是，为这个 finish 的操作创建一个 Transition#2，这样的话，在这个 Transition#2 会将 Activity#1 的 SurfaceControl 重新 show 出来。

但是真的是这样吗，关于这一点在 1.4 节已经分析过了，也许有的情况下可以，但是针对本题则不行，这种只存在于理想情况下，每一个 Transition 都按照顺序一一执行，没有交叉的情况，就比如本题 monkey 的情况，的确也为 finish 这个操作创建了 Transition#3，但是这个动画的实际内容并不是我们想当然的那样。

第二，我本地复现的情况，在 finish 流程中是尝试去创建 Transition#3 的，但是检测到 Transition#2 还在 collect，因此没有创建新的 Transition#3，如 1.5 节分析的那样。

2、在 Transition 结束的时候，去检验一遍动画前、动画中以及动画后的可见性变化是否和预期一致。

这个是我目前想到的解决此题的办法，这需要对动画流程中的可见性设置（主要是 ActivityRecord 的可见性）有一定了解，而解此题之前我对这块都是不求甚解。

2 ActivityRecord 可见性和 finishTransition
--------------------------------------

WindowContainer 是所有窗口容器的基类，要分析 ActivityRecord 就离不开先提几嘴 WindowContainer。

另外我以前说可见性的时候，说的其实比较笼统，实际上表示一个 WindowContainer 的可见性主要是两个概念，或者说两个方法，isVisible 和 isVisibleRequested，大部分时候不用特别去做区分，但是既然今天我们分析到了，这里还是较真一下。

这里先统一一下说法，isVisble，我就会称为”可见性 “，如果是 isVIsibleRequested，我会按个人习惯成为” 期望可见性 “，也许” 请求可见性“会贴近英文一点，但是语义上我觉得不合适。

### 2.1 WindowContainer 可见性

#### 2.1.1 WindowContainer.isVisible

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4cb0752b431641adad2ddf1b21341e61~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=DU8buvcycYJcKHrlmafX012%2FAKI%3D)

结合注释很好理解，遍历当前 WindowContainer 的所有子容器，只要一个子容器的 isVisible 返回了 true，或者说只要有一个子容器是可见的，那么当前 WindowContainer 就会被认为是可见的。

再看有哪些 WindowContainer 的子类重写了这个方法？

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/bbcce4a1f03d49a995cbbaa02eb024b7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=X1LSIoni1YUmObVyQrmuGJUi6j4%3D)

这说明了一点，很多类型的 WindowContainer，比如 Task 或者 TaskDisplayArea，它们本身是没有一个成员变量来表示自身是否可见的（区别于下面的 WindowContainer.isVisibleRequested 是有一个 WindowContainer.mVisibleRequested 来表示 isVisibleRequested 的状态的），它们是否可见完全取决于它们的子容器是否可见。

比如 TaskDisplayArea 是否可见依赖于它内部是否能寻到一个 Task 是可见的，而 Task 是否可见同样依赖于它内部是否能寻到一个 ActivityRecord 是可见的，所以在这个路径下，最终有决定权的是 ActivityRecord。

#### 2.1.2 WindowContainer.mVisibleRequested 和 WindowContainer.isVisibleRequested

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4f5bdbb1729a42c89cffea4ca4e25dce~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=BuDReWIQde9pp3cLePMp0B4ZoBQ%3D)

不同于刚刚的 WindowContainer.isVisible，这里的 WindowContainer.isVisibleRequested 是有一个请求的意思，即这个 WindowContainer 将要变成可见 / 不可见，虽然它现在仍然是不可见 / 可见，因此我称为期望可见性。它和 isVisible 的主要区别在 Transition 流程中会非常明显的表现出来，晚一点我们会以一个例子来说明。

#### 2.1.3 WindowContainer.setVisibleRequested 和 WindowContainer.onChildVisibleRequestedChanged

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/35c867ef5c304c18a4833eac538da510~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=g4%2B4mUiN3be2A2UEDitblau4Jd0%3D)

WindowContainer 提供了 setVisibleRequested 方法来让所有 WindowContainer 可以主动调用此方法来设置自身的期望可见性 mVisibleRequested 的值，但是实际上用到这个方法只有 ActivityRecord 和 WallpaperWindowToken：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/15ce51b4c0c849d39f98511ff2c02be7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=IRMyjo5qKQRCBJp%2FO4RQeh8SWKE%3D)

WallpaperWindowToken 不在这次的分析范围内，主要是还是 ActivityRecord。

还有一个问题是，既然只有 ActivityRecord 和 WallpaperWindowToken 会调用 setVisibleRequested 来设置自身的期望可见性 mVisibleRequested 的值，那么其它类型的 WindowContainer，比如 Task 和 TaskDisplayArea，它们是如何修改自身的期望可见性 mVisibleRequested 的值呢？

这就涉及到了 WindowContainer.onChildVisibleRequestedChanged 方法。

看代码，WindowContainer.onChildVisibleRequestedChanged 方法就是在 WindowContainer.setVisibleRequested 中被调用的，很简单，当 ActivityRecord 调用 setVisibleRequested 的时候，它会主动调用 parent 的 WindowContainer.onChildVisibleRequestedChanged 方法，来看它的 parent 是否需要设置自己的期望可见性 mVisibleRequested 的值。

再看 WindowContainer.onChildVisibleRequestedChanged 方法的内容：

1）、如果子容器的期望可见性为 true，自身的期望可见性为 false，那么直接设置自身期望可加性为 true。

2）、如果子容器的期望可见性为 false，自身的期望可见性为 true，那么继续寻找其它子容器，只要能找到一个子容器的期望可见性仍然为 true，那么当前 WindowContainer 的期望可见性仍然保持为 true。

这里的逻辑和 WindowContainer.isVisible 其实很像。

### 2.2 ActivityRecord 可见性

通过以上分析，可知设置可见性还有期望可见性比较多的 WindowContainer，主要就是 ActivityRecord，这里我以我对代码的理解来分析一下。

1、首先期望可见性 mVisibleRequested，这个 ActivityRecord 是直接从 WindowContainer 继承过来的，这里不用过多说什么。

2、然后是可见性 isVisible，如刚刚分析 WindowContainer 是没有提供一个成员变量来保存 isVisible 的状态的，但是对于 ActivityRecord 来说，是有的。

看下 ActivityRecord 重写的 WindowContainer.isVisible 方法以及它独有的 mVisible 成员变量：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/bab7cd3eb28646f5a0ba2bf385ffa3ac~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=7xnEKVnvmSjPQslNO3U1z7YqbTs%3D)

那么现在 ActivityRecord 就有两个表示可见性的成员变量了：

*   mVisibleRequsted，期望可见性。

*   mVisible，可见性。

为什么要设计两个表示可见性概念的东西出来呢？

要想了解这两个成员变量的概念，先以一个简单的例子说明一下。

### 2.3 ActivityRecord#1 启动 ActivityRecord#2 过程的可见性变化

这个过程是在同一个 Task 中，用 ActivityRecord#1（还是之前的例子，MainActivity）启动 ActivityRecord#2（CleanActivity），可以说是非常常见的操作了。

#### 2.3.1 设置 ActivityRecord#2 的期望可见性为 true

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4108be5b43d4463e920a0e6bd8dae282~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=TKpg9lamQYjTKz%2FtwXDLXbtN1cY%3D)

有几点要注意：

1）、这个流程是 ActivityRecord#1 完成 pause 之后发起的，起点在 ActivityClientController.activityPaused，后续调用了 ActivityTaskSupervisor.realStartActivityLocked，最终是调用了 DisplayContent.ensureActivitiesVisible 中，调用 ActivityRecord.setVisibility 更新了所有 Activity 的预期可见性。

2）、上一步为什么说是更新了预期可见性，难道可见性没有更新吗？看代码：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4b371a283f064aeb9d3012b29106acc6~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=o2TdBKf8lFkwOjQr9qytSzS4Q9o%3D)

标黄的这里，如果判断当前 ActivityRecord 处于动画中，是不会直接调用 ActivityRecord.commitVisibility 来设置 ActivityRecord 的可见性 mVisible 的。

3）、在 ActivityRecord.setVisibility 中，先 collect，然后才调用 ActivityRecord.setVisibleRequested，这样后续 ChangeInfo.hasChanged 判断动画就绪这个节点的前后 ChangeInfo.mContainer 是否有变化的时候，才能检测到此流程下可见性是发生了变化：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e4c2e16fd6cf4ddd9159be40599dc43c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=fb2UZhkgwDhowdpDBzcnNgVCT80%3D)

4）、这里的 ActivityRecord.setVisibility 是直接调用了 ActivityRecord.setVisibleRequested 设置了期望可见性 mVisibleRequested 的，因此期望可见性 mVisibleRequested 的修改是启动 Activity 的动作（即变化）后立即发生的。

#### 2.3.2 设置 ActivityRecord#1 的期望可见性为 false

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a15a16cd95e64101a4ace72cc6fc8bd2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=5W06ITTFBK5Z%2B%2BtVlrYsfxwUu1o%3D)

和上面的 ActivityRecord#2 的 mVisibleRequested 设置为 true 是在同一个流程中，不再赘述。

#### 2.3.3 设置 ActivityRecord#2 的可见性为 true

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b7efe972396f4c12a1024aafd528f8f9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=bYqF3kwyRf%2BosQwjMS3nxPMiLkI%3D)

调用堆栈为：

Transition.onTransactionReady

-> Transition.commitVisibleActivities

-> ActivityRecord.commitVisibility

-> ActivityRecord.setVisible

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b6e854c5b89c41d1aa91e779ed716460~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=UVgHNC%2FXsVEd0mausI733qKSWak%3D)

这里主要是注意：

1）、ActivityRecord#2 的 mVisible 设置为 true 的时间点为 Transition#onTransactionReady。

2）、ActivityRecord#commitVisibiliy 中会直接调用 ActivityRecord#setVisible 设 ActivityRecord#2 的 mVisible，没那么多弯弯绕绕，应该可以理解为当调用了 ActivityRecord#commitVisibiliy 的时候，ActivityRecord.mVisible 的设置就是板上钉钉的事了。

#### 2.3.4 设置 ActivityRecord#1 的可见性为 false

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/9569b6fb092c4577880296c26de97529~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=PU2mQyuOiM2f87gI2Pz06n45NRA%3D)

调用堆栈为：Transition.finishTransition -> ActivityRecord.commitVisibility

具体代码为：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1d65b34d13404374a9fad45f93a62efd~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=U7RedrZBM5N4lDiMnEAyF5sUFOs%3D)

这里我们主要关注 Transition 的这个成员变量 mVisibleAtTransitionEndTokens：

![](https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b23a7abeb6334d18ada8e94a541a140e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749807559&x-signature=4vomLzNCDKqCMJfjkJAsZbXQV6Y%3D)

它记录了 transition 动画结束的时候应该是可见的那些 WindowToken，看到它是在 Transition.onTransactionReady 的时候收集的所有当下可见的，这个时间点所有的变化应该都已经结束了，马上就要开始动画了，这个时间点的 WindowToken 的预期可见性应该是可以认为是和最终动画结束时的可见性一致了（理想情况下），因此可以用 mVisibleAtTransitionEndTokens 来记录一下这些应该在动画结束的时候可见的 WindowToken。

这里我们主要讨论 ActivityRecord，不讨论 WallpaperWindowToken，因此下面我就不那么严谨将 WindowToken 替换为 ActivityRecord 来继续说明。

然后当动画结束，在 Transition#finishTransition 的时候，检查那些没有记录在 mVisibleAtTransitionEndTokens 中的 ActivityRecord，如果这些 ActivityRecord 当前的预期可见性仍然为 false（一致性检查），那么这些 ActivityRecord 可以被认为是需要在动画结束被隐藏的 ActivityRecord，因此便可以调用 ActivityRecord.commitVisibility 来提交该 ActivityRecord 的可见性修改，最终会调用 ActivityRecord.setVisible 来设置该 ActiviyRecord 的可见性。

### 2.4 可见性总结

首先是结合 ActivityRecord#1 启动 ActivityRecord#2 的动画流程，看下这两个 ActivityRecord 的可见性 mVisible 和预期可见性 mVisibleRequested 都是在什么时间设置的：

1）、动画开始，ActivityRecord#1 完成 pause，开始 resume ActivityRecord#2 的时候，这个时候可以认为是动画开始不久，这里将 ActivityRecord#1 和 ActivityRecord#2 收集到 Transition 的 SyncGroup 的同时，分别设置这两个 ActivityRecord 的期可见性 mVisibleRequested。

2）、所有参与动画的窗口已经绘制完成（主要是 ActivityRecord#2 的窗口），Transition 走到 onTransactionReady，设置 ActivityRecord#2 的 isVisble 为 true。

3）、切换到 WMShell，开始播放动画，在播放动画前，onAnimationStart 阶段，调用 startT 的 apply 方法将 ActivityRecord#2 的窗口真正显示出来。

4）、动画播放结束，在 Transitions.onFinish 中， 调用 finishT 的 apply 方法将 ActivityRecord#1 的窗口真正隐藏掉。

5）、切换回 WMCore，在 Transition.finishTransition 中，调用 ActivityRecord.commitVisibility 设置 ActivityRecord#1 的 isVisble 为 false。

那么怎么理解 ActivityRecord.mVisible 和 ActivityRecord.mVisibleRequested 呢？

1）、结合 google 对 ActivityRecord.mVisible 的注释：

”Should this token's windows be visible?“

这么一看，ActivityRecord.isVisible 或者说 ActivityRecord.mVisible，更接近于这个 ActivityRecord 的窗口是否在屏幕上显示了，google 对 ActivityRecord 的可见性 mVisible 的设置是非常谨慎的，就在 startT 被 apply 或者是 finishT 被 apply 的时间点附近。

2）、至于 ActivityRecord 的预期可见性 mVisibleRequested，从名字上看是请求，也就是说实际上还没有真正落实，没有 isVisible 那种尘埃落定的感觉。从代码层面看，google 对这个预期可见性 mVisibleRequested 的修改其实是非常灵敏的，哪怕是这个 ActivityRecord 的窗口还没有显示，也可以将它的 ActivityRecord.mVisibleRequested 先设置为 true，表示它马上就要显示了，它更像是一个实时的参考，或者一个大方向的设定，如果后面没有变化，那么最终 ActivityRecord 的可见性 mVisible 的值肯定是和 ActivityRecord 的预期可见性 mVisibleRequested 的值是一致的。

3 解决思路
------

从以上对 Transition.finishTransition 的代码分析，我们知道在这里面是有一个可见性的一致性检查的。

还是以从 ActivityRecord#1 上启动 ActivityRecord#2 的动画流程为例：

1）、动画开始，在 ActivityRecord#1 的预期可见性 mVisibleRequested 被设置为 false 之前，将其添加到动画参与者中，添加的同时会记录一下 ActivityRecord#1 此时的预期可见性 mVisibleRequested，即 true。

2）、设置 ActivityRecord#1 的预期可见性 mVisibleRequested 为 false。

3）、Transition.onTransationReady 时间点，尝试向 Transition.mVisibleAtTransitionEndTokens 添加当前仍然是预期可见的 ActivityRecord，ActivityRecord#2 满足，但是 ActivityRecord#1 并不满足，所以不会被添加到这里面。

4）、最终动画结束 finishTransition 的时候，检查到 ActivityRecord#1 没有在 Transition.mVisibleAtTransitionEndTokens 中，即 ActivityRecord#1 在 Transition.onTransationReady 这个时间点被认为是预期不可见的，并且如果此时其预期可见性 mVisibleRequested 仍然是 false，那么就认为 ActivityRecord#1 在动画播放前后都是预期不可见的，保持了一致，因此就可以调用 ActivityRecord.commitVisibility 来最终设置其可见性 mVisible 为 false。

首先是为什么要有这个可见性的一致性检查，我认为是这样的。首先是在 Transition.onTransationReady 这个时间点 Transition.mVisibleAtTransitionEndTokens 中收集到的 ActivityRecord 都是我们最终希望显示的，不在这里面的都是我们最终希望隐藏的。但是从 Transition.onTransationReady 到 Transition.finishTransition 这个时间段，即 playing 阶段，ActivityRecord 的预期可见性是仍然有可能变化的（就比如我们分析的问题情况），因此需要在 playing 之后再去检查它们的预期可见性有没有发生变化，保证我们将没有将预期可见的 ActivityRecord 隐藏了。

那么回到我们之前分析的问题场景中，如果在 playing 阶段，ActivityRecord#2 主动调用 finish，导致预期不可见的 ActivityRecord#1 因为被 resume 又重新变为预期可见了，那么情况是如何呢？

1）、首先是 playing 阶段，在 Transitions.setupStartState 中，ActivityRecord#1 的 SurfaceControl 是被隐藏了。

2）、然后到了 finish 阶段，在 Transition.finishTransition 中，由于此时 ActivityRecord#1 又变成预期可见了，那么这里的做法就是不为 ActivityRecord#1 调用 ActivityRecord.commitVisibility 来设置它的可见性 mVisible。但是在 Transitions.setupStartState 的时候，你可是实打实把它的 SurfaceControl 隐藏了，而现在却没有机制来重新显示它的 SurfaceControl，导致出现了问题。

看到整个动画流程，ActivityRecord#1 的预期可见性 mVisibleRequested 从动画开始前的 true，变为 playing 前的 false，再到 finish 时的 true，我目前能想到的做法就是针对这种特殊场景，重新显示 ActivityRecord#1 的 SurfaceControl。

4 ShellTransitions
------------------

至此 ShellTransitions 的流程总算是总结完了，当然只是粗略的过了一遍，不可能面面俱到，很多重要的地方由于我工作中解题接触不到，因此都没有仔细去解读，也许后续的问题分析文章可以补全一些部分吧，毕竟现在解的”Application does not have a focused window“类型的 ANR 问题很多都和 ShellTransitions 流程有关系，google 自从在 WMShell 中开放对 SurfaceControl 的操作后，感觉 SurfaceControl 这块的管理变得太不可控了。

最后后续也许会出一个 ShellTransitions 系列的总结，不过谁知道呢，刚开始计划写这系列文章的时候还是 Android14，现在都到 Android16 了，偷懒一时爽，一直偷懒一直爽。
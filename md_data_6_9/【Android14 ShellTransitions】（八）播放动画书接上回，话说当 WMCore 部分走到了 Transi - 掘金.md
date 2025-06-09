> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7431120960637812771)

> 书接上回，话说当 WMCore 部分走到了 Transition.onTransactionReady，计算完参与动画的目标，构建出 TransitionInfo 后，接下来就把这个包含了动画参与者的 Trans

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c08858275d864eb3a9992f73fed39222~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=Tsqlfpc3FJEgaPUmr8LySe7dWAo%3D)

书接上回，话说当 WMCore 部分走到了 Transition.onTransactionReady，计算完参与动画的目标，构建出 TransitionInfo 后，接下来就把这个包含了动画参与者的 TransitionInfo 发给了 WMShell，然后就该播放动画了，这部分在 WMShell。

1 TransitionPlayerImpl.onTransitionReady
----------------------------------------

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ff507ae29ff94f2f82f516c0c3c10a68~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=6S%2FNTVJ7xJVb2M%2BqcGP2iuTgZ80%3D)

TransitionPlayerImpl 是 ITransitionPlayer 的本地实现，这里继续调用了 Transitions.onTransitionReady 方法，但要注意的是，这里并不是直接调用了 Transitions.onTransitionReady，而是将这个调用操作放到了主线程中执行，那么这里多多少少会有一些延迟，更甚者，如果 SystemUI 主线程在做耗时操作，那么就会更晚走到 Transitions.onTransitionReady，这影响的是这里的传参，Transaction 类型的 “t”，即之前的文章所提到的“start transaction” 的 apply 的时间，严重点就会引发 ANR，关于这一点当我们分析到 “start transaction” 被 apply 的时候再聊聊。

2 Transitions.onTransitionReady
-------------------------------

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/13d3c456d4214b8a80d7846814cea284~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=xvB%2Ff2Ba5Mqdr742LHVY5o6gj4Q%3D)

这个方法也简单，主要看下这个局部变量 activeIdx，之前在：

[【Android14 ShellTransitions】（五）启动 Transition 这一节的内容涉及 WMCore 以及 W - 掘金](https://juejin.cn/post/7371717387829870633 "https://juejin.cn/post/7371717387829870633")

这一文的分析中，主要是在 Transitions.requestStartTransition 中：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c2eaf36cb58241f7984d5bda4808cb86~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=jVy88RCKEhB3wCL%2B1qhtBfxdDNI%3D)

WMShell 侧根据从 WMCore 侧传来的 Transition 对象的 token，创建了一个与 Transition 对象一一对应的 ActiveTransition 对象，并且将这个传过来的 token 保存到了 ActiveTransition.mToken 中，然后将这个 ActiveTransition 对象保存到了 Transitions.mPendingTransitions 中。

现在回到 Transitions.onTransitionReady，当 WMCore 侧再次把 Transition 的 token 发过来的时候，我们就可以根据这个 token，从 Transitions.mPendingTransitions 中取到对应的 ActiveTransition 对象。

另外看到这里也把一些传参保存到了 ActiveTransition 的各个成员变量中，后续就直接从 ActiveTransition 中拿，而不是由 Transitions 保存，因为 Transitions 可能要同时处理多个 Transition/ActiveTransition。

另外就如这里的注释所说，ActiveTransition 发生了改变，“Move from pending to ready”，也就是说当 ActiveTransition 创建后就一直是 pending 的状态，当 WMCore 侧走到 Transition.onTransactionReady，即 Transition 就绪的时候，再通知 WMShell 说 WMCore 这边就绪了，该你这边了，然后 ActiveTransition 就从等待状态切换到就绪状态（虽然没有一个状态值来表明 ActiveTransition 所处的状态）。

最后我们只分析最普通的单个 Transition 的情况，看到这里继续调用了 Transitions.dispatchReady 方法。

3 Transitions.dispatchReady
---------------------------

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/be6ac6e7b2364520a5530dc272e155ca~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=t31gjU7jk%2FzxqXZe69hfpsM7f1w%3D)

1）、为 ActiveTransition 分配一个 Track，然后将该 ActiveTransition 添加到 Track.mReadyTransitions。

看下 Track 类的定义：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1fcb90aefd91468fb6576ad612e8032b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=FX9IYoazlMkobr12wfuYQNUQazA%3D)

Track 用来播放动画，其成员变量 mReadyTransitions 保存了处于 ready 状态但是还没有轮到播放的 ActiveTransition，它的存在说明了 Track 是可以收集多个 ActiveTransition 来并行播放动画的，成员变量 mActiveTransition 代表了正在播放动画的那个 ActiveTransition。

2）、调用 Transitions.setupStartState 方法，该方法如其名字所表达的，用来设置动画初始状态的可见性、透明度和变换。

3）、继续调用 Transitions.processReadyQueue。

这里看下 Transitions.setupStartState 方法：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/95a971d8c0fb4fe39660e6b8424cf594~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=GJwkZUwmfeHA1i0EyBfGyI21oag%3D)

这个方法是用来设置参与动画的一些 SurfaceControl 的初始状态的，其实也没啥好看的，唯一需要注意的是这里会对动画参与者的 SurfaceControl 进行一些设置，保证了一些动画的基本逻辑能够得到满足，就比如说对于 TRANSIT_OPEN 和 TRANSIT_TO_FRONT 类型的动画参与者，这里会强制它们在动画开始的时候显示，此外它会重置 SurfaeControl 的缩放和旋转等属性，如果你之前因为一些特殊需求改了 SurfaceControl 的缩放比例，那么就要注意这里不要让你的修改失效了。同样的对于 TRANSIT_CLOSE 和 TRANSIT_TO_BACK 类型的动画参与者，这里的逻辑则是保证了该动画参与者在动画结束后会被隐藏。

4 Transitions.processReadyQueue
-------------------------------

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/89e20fc706434587b905cb7311c4ef9c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=3fw6mj%2Ft9W%2FCvzsYk7JQhEOT9NI%3D)

1）、在我们的分析流程中，上一步 Transitions.dispatchReady 向 Track.mReadyTransitions 添加了就绪的 ActiveTransition，所以这里 Track.mReadyTransitions 不为空。

2）、由于之前我们没有设置过 Track.mActiveTranstion，那么这里我们会将 Track.mReadyTransitions 队首的那个 ActiveTransition 对象取出来，赋值给 Track.mActiveTranstion，然后从 Track.mReadyTransitions 中移除，接着为 ActiveTransition 对象调用 Transitions.playTransition，那么这个 ActiveTransition 会从 ready 状态变为 playing 状态，最后再调用一次 Transitions.processReadyQueue。

3）、当再进入 Transitions.processReadyQueue 后，如果 Track.mReadyTransitions 仍然不为空，那么说明此前 Track 至少有两个 ready 的 ActiveTransition，并且首个进入 Transitions.processReadyQueue 的那个 ActiveTransition 已经变为 playing 了，那么会尝试将剩下的这个 ready 的 ActiveTransition 和这个 playing 的 ActiveTransition 的动画合并，不过这部分我了解的很少，暂时跳过。

继续看 Transitions.playTransition。

5 Transitions.playTransition
----------------------------

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/64536158c8694bd981481bf3eec9c745~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=u%2F%2BEFcvPoToA9tJ1bHyEx4Pxh0k%3D)

分为三个部分：

1）、Transitions.setupAnimHierarchy 用来在动画开始前，将动画参与者 reparent 到共同的父 Layer 上，然后设置它们的 Z 轴层级。

2）、之前我们在 Transitions.requestStartTransition 中，尝试为每一个 TransitionHandler 调用 handleRequest，来找到能处理当前 ActiveTransition 的那个 TransitionHandler，如果能够找到，那么将这个 TransitionHandler 保存在 ActiveTransition.mHandler 中。这里便是调用这个 TransitionHandler 的 startAnimation 方法，让这个 TransitionHandler 来优先处理当前 ActiveTransition，如果能够处理，那么就会返回 true，这个 ActiveTransition 就算找到可以托付终生的 handler 了。这里也能看到，真正决定 Transition 被谁处理的是 TransitionHandler.startAnimation，TransitionHandler.handleRequest 只是给你一个优先处理的机会。

3）、如果 TransitionHandler.startAnimation 返回 false，那还得继续调用 Transitions.dispatchTransition 来遍历 Transitions.mHandlers 中的所有 TransitionHandler，继续找可以处理当前 ActiveTransition 的 TransitionHandler。

接下来分别介绍。

### 5.1 Transitions.setupAnimHierarchy

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3c3165d6700f481192ade8dc7e615012~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=n1722l%2BnRrTiHrmLULq%2BO%2BHcTpc%3D)

Transitions.setupAnimHierarchy 用来在动画开始前，将动画参与者 reparent 到一个共同的父 Layer 上，然后设置它们的 Z 轴层级。

我这里是从 Launcher 打开 Message，截图为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e70a351d229d465c818ee6571ee62358~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=7P1FHxIIIu4aA6pZFzk68O4NlDg%3D)

能看到在 TaskDisplayArea 下创建了一个名为 “Transition Root:......” 的 Layer，作为动画参与者的共同 parent。

然后是设置层级的逻辑，主要是根据：

*   “global transit type”，即 TransitionInfo 的类型。

*   “their transit mode”，即 Change 的 mode。

*   “their destination z-orde”，即 Change 的目标 Z 轴层级（如果有的话）。

举个例子，比如上面的从 Launcher 打开 Message 的例子：

1）、TransitionInfo 的类型为 TRANSIT_TO_FRONT。

2）、Launcher 对应的 Change 的 mode 为 TRANSIT_TO_BACK，Messsage 对应的 Change 的 mode 为 TRANSIT_TO_FRONT。

对于这个场景，我们更想突出的应该是 Message 的打开动画，因此 Message 对应的 Change 的 Z 轴层级应该调高，而 Launcher 对应的 Change 的 Z 轴层级则应该调低，就像代码里面标黄的那样。

动画参与者是两个 Task，那么这里的 zSplitLine 就是 3，numChanges 是 2，Message 的 Task 先被添加，因此 i=0，Launcher 的 Task 的 i=1。

最终算出给 Message 的 Task#11 设置的 Z 轴层级为 5：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d6c21f38187349dc9d8f732a4512991c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=6wRKX50jwn5AHj7LfT3NTBx7OcQ%3D)

Launcher 的 Task#1 设置的 Z 轴层级为 2：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/bc29433a8c3b479cbb57f324b2385779~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=eP8qtpJvBWszM9McLicc4mTwDVY%3D)

因此 Message 盖在了 Launcher 之上，突出的是 Message 的动画。

### 5.2 TransitionHandler.startAnimation

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/55a05c8ea99f4f0085d80097cad6e9f2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=PyeCiD4e6rnaMS9FJwC9Ces95XY%3D)

翻译一下注释：startAnimation 用来启动一个过渡动画。对于某一个特定的 Transition，如果该 TransitionHandler 的 handleRequset 方法返回一个 non-null 的值的话，那么该 TransitionHandler 的 startAnimation 将总是会被调用。否则，只有当排在该 TransitionHandler 之前的其它 TransitionHandler 都没有办法处理 Transition 的时候，该 TransitionHandler 的 startAnimation 才会被调用。

这段注释也即 Transitions.playTransition 方法内容的描述。

实现该接口的 TransitionHandler 子类有一二十个：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e57cac7ef03f4a76879c1df19da8a358~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=Y9HNYTByyzpwFiQl74oqxr%2F%2BdsU%3D)

作用于不同场景下的过渡动画，这里只分析一个典型的 DefaultTransitionHandler。

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/479e371f27d64b4eac9683a26033d2ea~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=o5tBIu4j%2Bu%2Bk1V9RriVPKg4o530%3D)

我们之前分析的场景是从 Launcher 启动 Message，由于现在从 Launcher 启动 App 的动画一般都是自定义的，所以走的并不是 DefaultTransitionHandler，而是 RemoteTransitionHandler，所以为了想要 DefaultTransitionHandler 处理调起 Message 的动画（App 切换），我这里通过 adb 命令从 Launcher 调起 Message。

#### 5.2.1 设置回调 Runnable

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1b94088d19a34576a97f41f96188bcfc~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=5g5JVg7u4UHa2vMW5FK%2FrM2ac9w%3D)

设置一个 Runnable，该 Runnable 将在动画结束的时候执行（后续分析会看到，这里先提一嘴），执行的则又是传参 finishCallback 的 onTransitionFinished 方法，再看我们分析的流程下，TransitionHandler.startAnimation 方法被调用的两处地方，分别在 Transitions.playTransition 和 Transitions.dispatchTransition，得知：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1fd3b39e787b4da79ea2954a486e30de~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=4RMogm42SH9JHtdeHXaNAG3x3Ts%3D)

最终调用的是 Transitions.onFinish 方法。

#### 5.2.2 加载动画

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/fe6123aa6cd244ae8dfea031d0ef65bb~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=KL7AHP9et9jXHoOxYYDFjglJvpw%3D)

调用 DefaultTransitionHandler.loadAnimation 方法来加载动画样式：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/aece1e788a904ab994cd5f253947aded~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=R1TRvoIGcSTvSkiXmhjUVoYo9YQ%3D)

如果对之前 AppTransition 和 AppTransitionController 中的动画逻辑有印象的话，那么看到这里应该会很熟悉，这里差不多就是 AppTransition.loadAnimation，主要就是根据动画的类型，以及是否有自定义动画来选择动画的样式。

其中自定义动画一般由 App 通过 ActivityOptions 的各种 makeXXXAnimation 方法指定：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/585c4dabac5f4d94ad9f44acb2cc1afa~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=xXZc9LKU0EDgioLJrp9jODml29c%3D)

自定义动画还是少数，大部分情况下动画则是 TRANSIT_OPEN、TRANSIT_TO_FRONT、TRANSIT_CLOSE 以及 TRANSIT_TO_BACK 这几类，并且是没有自定义动画的，如我们分析的用 adb 调起 Message App 的场景，那么会继续调用 TransitionAnimationHelper.loadAttributeAnimation 来选择动画样式：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/0cd38bf9743249d5899b59a9a50b1a52~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=GZZyRB2Ce%2FhOPJdyYTOmGGCjPJc%3D)

这里的逻辑一看就懂，分的也非常细，根据 Wallpaper 的可见性是否发生变化，是否有透明的 App 参与，参与者是 Task 还是 Activity 之类的，都有有相应的动画可以选择。比如我们分析的场景，启动的是 Message App，那么 TransitionInfo 的 type 为 TRANSIT_TO_FRONT，并且 Message 对应的 Change 的 mode 也为 TRANSIT_TO_FRONT，那么最终选择的动画就是 R.styleable.WindowAnimation_taskToFrontEnterAnimation。

最后要注意我们这里拿到的只是一个资源 ID，最终还是要通过 TransitionAnimation.loadDefaultAnimationAttr 来将这个资源 ID 转化为 Animation 对象，大概过一下：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6fe92826ce544739a2385dddad1fe980~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=THRKhMQKRE1gLHF3W6WT4jmOdKk%3D)

方法调用顺序为：

TransitionAnimation.loadDefaultAnimationAttr

-> TransitionAnimation.loadAnimationAttr

-> TransitionAnimation.loadAnimationAttr

-> TransitionAnimation.loadAnimationSafely

继续调用了 AnimationUtils.loadAnimation 方法：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/de3ff3af7423425090c9f20b86683477~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=xynDxh2b1B5B7E7sepQ703xLMHM%3D)

方法调用顺序为：

AnimationUtils.loadAnimation

-> AnimationUtils.createAnimationFromXml

-> AnimationUtils.createAnimationFromXml

看到最终是调用了 AnimationUtils.createAnimationFromXml 方法，从指定的 package 中根据对应资源 ID 来解析相应的 xml 文件，然后将 xml 文件中的不同的标签解析成对应的 Animation 类型，这里看到有多个标签，“alpha”、“scale”、“rotate”和 “translate” 等等。

比如我们刚刚提到的，在我们分析的场景下，选取的动画资源 ID 为 R.styleable.WindowAnimation_taskToFrontEnterAnimation。

它声明在”frameworks/base/core/res/res/values/attrs.xml“中：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/56c012a7b75a44f382f0f8e5326b81fe~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=kySfFvI5M%2Br5QbPfWaoPn6TFX0g%3D)

定义在”frameworks/base/core/res/res/values/styles.xml“：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/0208f1eaf9c84ff796518c824d2073b2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=5uOgT6cqSaHM32emb%2BLTfW7G%2Bh4%3D)

然后它的动画对应的就是”frameworks/base/core/res/res/anim/task_open_enter.xml“：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3c9ea46e2f6249b5a3c1713247a10cc3~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=ZuU1b1%2BCb0boISLuzIuP607OzWc%3D)

标签为”translate“，那么会解析为一个 TranslateAnimation 动画对象。

最后有一点要注意的就是，由于在我们分析的流程下，我们没有自定义动画，因此加载的是默认包名：

```
private static final String DEFAULT_PACKAGE = "android";


```

“android“下的动画，即 google 的原生动画。

如果我们有自定义动画，比如我用 Activity.overridePendingTransition 自定义 Activity 的进入和退出动画，就像：

```
        mPendingEnterRes = R.anim.edge_extension_right;
        mPendingExitRes = R.anim.alpha_0;
       overridePendingTransition(mPendingEnterRes, mPendingExitRes);


```

那么后续动画使用的就是我在”R.anim.edge_extension_right“和”R.anim.alpha_0“中定义的动画：

```
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android" android:shareInterpolator="false">
   <alpha android:interpolator="@android:interpolator/linear" android:duration="10000" android:fillBefore="true" android:fillAfter="true"
        android:startOffset="0" android:fromAlpha="1" android:toAlpha="1" android:fillEnabled="true"/>
   <scale android:interpolator="@android:interpolator/linear" android:duration="10000" android:startOffset="0"
        android:fromXScale="0.5" android:toXScale="0.5" android:fromYScale="1" android:toYScale="1"/>
</set>


```

这些动画的资源文件保存在”app\src\main\res\anim\alpha_0.xml“：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/67a6453cda7745158d066950b816f675~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=V12QznQsBdw9QrNgXfi2Yci3Mo4%3D)

但是要注意的是我这里的自定义动画并不属于远程动画，因为我没有指定一个 RemoteTransition，看 log，最终处理动画的还是 DefaultTransitionHandler：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/81533df68db0402383de66a51e222453~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=jC%2FZgXq6Su9fN%2B4jZkw8d6VVftU%3D)

像 Launcher 这样的调用 ActivityOptions.makeRemoteAnimation 传入一个 RemoteAnimationAdapter 和 RemoteTransition 的才是：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/2a33833c31834f42b067d1e5813a120e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=oCaQejUyRz391HV8dmQFqfm2jSE%3D)

log 为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f3c867ed4e894afd91ac7040d24cf9d6~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=HR45t%2FlZG0bv%2BwJa83ZOV9AiTfE%3D)

这里不对远程动画进行展开。

#### 5.2.3 构建作用于 leash 的 animator

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6026efdf097946f3845ec5cd3bf9368e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=p7ZSJ7CEG8YS6Ys9e%2Bs91UWiL6s%3D)

DefaultTransitionHandler.startAnimation 创建了一个局部变量 animations，用来收集动画参与者的 animator，当我们在第二步通过 DefaultTransitionHandler.loadAnimation 为动画参与者创建了动画后，接下来就是调用 DefaultTransitionHandler.buildSurfaceAnimation 来创建相应的 animator：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/cd8bce180c5c4dfe9a07effcf28ea545~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=qamd7MGI2BdH1aveuJF1K58TB00%3D)

我们主要关注创建的这个 ValueAnimator。

1）、调用 ValueAnimator.addUpdateListener 为这个 ValueAnimator 添加一个监听每一帧进行更新的 ValueAnimator.AnimatorUpdateListener，在每一帧到来的时候，调用 DefaultTransitionHandler.applyTransformation 来对 leash 进行更新。

2）、DefaultTransitionHandler.applyTransformation 这个方法也很简单，这里不再贴代码，主要是根据当前动画的播放时间来计算 Transformation，然后对相应的 leash（动画参与者的 SurfaceControl）进行设置，从而完成每一帧的 SurfaceControl 更新。

3）、调用 ValueAnimator.addUpdateListener 为这个 ValueAnimator 添加一个监听动画生命周期的 AnimatorListenerAdapter，这里主要是在动画正常结束或者异常取消的时候，调用传参 finishCallback 的 run 方法，这将调用 DefaultTransitionHandler.startAnimation 的传参 finishCallback 的 onTransitionFinished 方法，最终将调用 Transitions.onFinish 方法，这个我们之前也有提过。

4）、构建了这个 ValueAnimator，并且设置完动画监听器后，就把这个 ValueAnimator 对象添加到 DefaultTransitionHandler.startAnimation 创建的这个局部变量 animations 中。

#### 5.2.4 应用”start transaction“

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b7b46e3079934e2bb9e7d37c132c1042~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=fs7XGG09PNb%2Bczrd0KbHfVebs0Y%3D)

这一步很简单，即调用”start transaction“的 Transaction.apply 方法，但是我们需要着重强调一下。

1）、回看：

[【Android14 ShellTransitions】（七）Transition 就绪 ShellTransitions - 掘金](https://juejin.cn/post/7391284215113924620#heading-2 "https://juejin.cn/post/7391284215113924620#heading-2")

在 1.2 小节显示 SurfaceControl 中，由于这里我们使用的 Transaction 是通过 WindowContainer.getSyncTransaction 方法得到的，因此这个 Transaction 中收集到的 Surface 的数据何时会被应用，是由 Transition 动画流程决定的。

2）、再根据：

[【Android14 ShellTransitions】（六）SyncGroup 完成 BLASTSyncEngine Sy - 掘金](https://juejin.cn/post/7382893339097956393 "https://juejin.cn/post/7382893339097956393")

后续当 SyncGroup 完成后，在 SyncGroup.finishNow 中，所有动画参与者的 SyncTransaction 都会被合并到一个名为 merge 的 Transaction 对象中，接着这个 Transaction 对象会被传入 Transition.onTransactionReady 中， 也就是我们所说的”start transaction“。

3）、后续走到 WMShell 的 Transitions.onTransitionReady 后，这个”start transaction“被保存在了 ActiveTransition.mStartT 中。

4）、直到此时，构建完 animation 以及 animator，马上就要开始动画的时候，”start transaction“才会被 apply。

这是很长的一段路，比如我们从 Launcher 启动 Message 的流程，Message 的 Surface 真正显示的时间点并不是 WindowSurfaceController.showRobustly，而是动画即将开始之前的这里。不过这似乎是没办法的事，如果 Surface 过早显示，然后由于 SystemUI 主线程卡顿，结果在 Surface 显示两三秒后又开始动画了，那不是更奇怪。

现在再回看分析 TransitionPlayerImpl.onTransitionReady 时我提到的，如果 SystemUI 主线程卡顿，那么从 Binder 线程切换到主线程就需要非常多的时间，也就是动画流程被延迟了，那么这也将极大的延迟”start transaction“被应用的时间，导致”start transaction“的内容无法被及时提交到 SurfaceFlinger。

比如在我们分析的 Launcher 启动 Messag 场景中，”start transaction“中包含着对 Message 的 Surface 进行 Transaction.show 的设置，”start transaction“被延迟应用就意味着 Message 的 Surface 被延迟显示，如果超过 5s 还没应用，那么就有可能会引发无焦点窗口的 ANR（在上层 WMS 窗口焦点已经切换到 Message，但是在 SF 层 Message 的 Layer 由于没有显示因此在 InputDispatcher 侧无法作为焦点窗口），这种情况下一般还会伴随着以下 log：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/af2db608af8944299acc58e4edf0dfc3~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=QKODJkUclA%2BSjo2HLzoXACX3W7g%3D)

”sent“花费的时间不到 1s，但是”finished“的时候总共用时 5s 以上，当时有这个 log 不能说一定是 SystemUI 主线程卡顿，只能说嫌疑比较大。

因此这类 ANR 问题，发生的瓶颈不在于 App 绘制速度，而在于 SystemUI 主线程卡顿。由于现在 ShellTransitions 的动画播放部分放到 SystemUI 进程处理，而 SystemUI 进程本身也有很多常驻窗口要显示，因此这对 SystemUI 的性能提出了过高的要求。

如果没有了解现在的动画流程的话，应该也想不到 SystemUI 主线程卡顿是此类 ANR 问题的原因吧，哈哈哈.....（google 你是真爱折腾啊，某种植物！）

#### 5.2.5 开始动画

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4f8a81ba25664f20a0e7575508b4e351~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=wz8KmK5AnuuMXkqNQ%2BYNDBx5lHA%3D)

分析了那么久，正菜终于上了，其实也很简单，就调用了 Animator.start 方法来启动动画而已，一直以来都是这么干的，没啥好讲的，这就好比在厨房洗切炒叮铃咣啷的，一顿操作猛如虎，上桌一看吃红薯。

我这里把动画的生命周期打印了调用堆栈，大概看下：

1）、onAnimationStart：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/653b56215b834b58b0d26b1567039189~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=uleovEZp7qZwXErlVV5SmEaRMOQ%3D)

2）、onAnimationEnd：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/cf129a88babb4de6b0588a403a7fca7a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=X1yPA9VEI%2B3N3xc%2Bb2jMm2G7ckU%3D)

当动画结束的时候，回调的是 Transitions.onFinish 方法。

### 5.3 Transitions.dispatchTransition

在分析 Transitions.onFinish 之前，我们还剩最后一点关于 Transitions.dispatchTransition 的内容没讲：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/555d313b73484cb9a8fc19c44a832198~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=ECO%2FoHIJs1WLxxMR%2F47WueBUHjg%3D)

根据之前 Transitions.playTransition 的内容，如果 ActiveTransition.mHandler 无法处理当前 ActiveTransition 的话，那么我们就要给 Transitions.mHandlers 中的其它 TransitionHandler 一个机会，看它们能不能处理这个 ActiveTransition 了，内容也很简单，不再赘述，唯一需要注意的是这里遍历的顺序和 TransitionHandler 添加的顺序相反：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/61b1650cd8cc44649e99940e8501d5c7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=yBPsGIWh1O9Hif4EVEYGATF0dDo%3D)

如我这里的 log，标蓝色的是添加的 log，标黄的是遍历的 log，可以看到 DefaultMixedHandler 是最后一个添加的，但是遍历的时候是先调用的 DefaultMixedHandler 的 startAnimation 方法。

6 Transitions.onFinish
----------------------

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/04f063af91454f09846d1f41519f4a5a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749535426&x-signature=Oi7CDAz8hUZlB9UPCqX7jCpJNA0%3D)

这个方法的内容也不多：

1）、这里是”finish transaction“被 apply 的地方。

2）、调用 WindowOrganizer.finishTransition，最终重新回到系统系统 WMCore 走 Transition 的 finish 流程。

3）、由于当前 Transition 的动画已经结束，那么继续调用 Transitions.processReadyQueue 来重新看看有没有准备好的 Transition 可以开始播放动画的。 

下一篇文章继续分析 Transition 的 finish 流程。
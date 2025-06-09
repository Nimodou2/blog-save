> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7371709743221932071)

> 这一节的内容在 WMCore 中，主要是创建 Transition，初始化其状态为 PENDING。 还是我们之前说的，我们以在 Launcher 界面点击 App 图标启动某个 App 为例，来分析 Transition

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/822c7384f8a445e7b8725abcb3f86747~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2000&h=1125&s=2172269&e=jpg&b=7f511c)

这一节的内容在 WMCore 中，主要是创建 Transition，初始化其状态为 PENDING。

还是我们之前说的，我们以在 Launcher 界面点击 App 图标启动某个 App 为例，来分析 Transition 的一般流程。启动 Activity 的流程，在 ActivityStarter.startActivityUnchecked 中：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50ce10d468be4e7e89857a8bf99be2f9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=817&h=662&s=76736&e=png&b=fefdfd)

具体的调用堆栈为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be97da6747604ed2b6a78c2072adb16c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=921&h=209&s=28102&e=png&b=fffefe)

ActivityStarter.startActivityUnchecked 的主要内容为：

1）、首先调用 TransitionController.createAndStartCollecting 方法创建一个类型为 TRANSIT_OPEN 的 Transition 对象。

2）、将当前启动的 ActivityRecord 收集到刚刚创建的 Transition 对象中。

3）、调用 ActivityStarter.startActivityInner 去走具体的启动 Activity 流程。

4）、最后在 ActivityStarter.handleStartResult 中，调用 TransitionController.requestStartTransition 来启动动画。

在这一节中我们只分析和创建 Transition 相关的部分，即 TransitionController.createAndStartCollecting 的内容，余下的部分在其它章节中再进行分析。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d200e83972a54947a6efd71e89ec1f3d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=799&h=357&s=35856&e=png&b=fefdfd)

首先创建相应类型的一个 Transition 对象。

能看到创建 Transition 的地方还是挺多的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0da48f50ba8e46a1a780015a4528b0bc~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1680&h=424&s=133561&e=png&b=3f4244)

然后 Transition 的初始状态就是 STATE_PENDING，不需要额外去设置（也没有额外的地方去设置，毕竟 Transition 用完之后就不用了，不存在循环利用的情况）：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0aaec071b2344c1bb0c8ba007405ff2a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=521&h=79&s=10740&e=png&b=2b2b2b)

这一节的内容还是比较简单的，在 WMCore 侧，根据动画的类型创建相应的 Transition 对象，Transition 的初始状态为 STATE_PENDING。
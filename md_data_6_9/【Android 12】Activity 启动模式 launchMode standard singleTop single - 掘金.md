> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7071188263968440356)

> launchMode standard singleTop singleTask singleInstance singleInstancePerTask

google 文档的介绍：

> 您可以通过启动模式定义 Activity 的新实例如何与当前 Task 关联。您可以通过两种方式定义不同的启动模式：
> 
> 使用 manifest 文件
> 
> 当您在 manifest 文件中声明 Activity 时，您可以指定该 Activity 在启动时如何与 Task 关联。
> 
> 使用 Intent 标记
> 
> 当您调用 startActivity() 时，可以在 Intent 中添加一个标记，用于声明新 Activity 如何（或是否）与当前 Task 相关联。
> 
> 因此，如果 Activity A 启动 Activity B，Activity B 可在其 manifest 中定义如何与当前 Task 相关联（如果关联的话），Activity A 也可以请求 Activity B 应该如何与当前 Task 关联。如果两个 Activity 都定义了 Activity B 应如何与 Task 关联，将优先遵循 Activity A 的请求（在 intent 中定义），而不是 Activity B 的请求（在 manifest 中定义）。
> 
> 注意：有些启动模式可通过 manifest 文件定义，但不能通过 intent 标记定义，同样，有些启动模式可通过 intent 标记定义，却不能在 manifest 中定义。

在 manifest 文件中声明 Activity 时，可以使用 <activity> 元素的 launchMode 属性指定 Activity 应该如何与 Task 关联。

launchMode 属性说明了 Activity 应如何启动到 Task 中。launchMode 的定义如下：

```
<!-- Specify how an activity should be launched.  See the
     <a href="{@docRoot}guide/topics/fundamentals/tasks-and-back-stack.html">Tasks and Back
     Stack</a> document for important information on how these options impact
     the behavior of your application.

     <p>If this attribute is not specified, <code>standard</code> launch
     mode will be used.  Note that the particular launch behavior can
     be changed in some ways at runtime through the
     {@link android.content.Intent} flags
     {@link android.content.Intent#FLAG_ACTIVITY_SINGLE_TOP},
     {@link android.content.Intent#FLAG_ACTIVITY_NEW_TASK}, and
     {@link android.content.Intent#FLAG_ACTIVITY_MULTIPLE_TASK}. -->
<attr >
    <enum  />
    <enum  />
    <enum  />
    <enum  />
    <enum  />
</attr>


```

launMode 有 5 种：

standard、singleTop、singleTask、singleInstance、singleInstancePerTask。

如果 launchMode 属性没有指定，那么默认是 standard 模式。注意在运行时某特殊启动方式可能会被 Intent 的 flag 改变。

1 standard
----------

### 1.1 standard 定义

```
     <!-- The default mode, which will usually create a new instance of
          the activity when it is started, though this behavior may change
          with the introduction of other options such as
          {@link android.content.Intent#FLAG_ACTIVITY_NEW_TASK
          Intent.FLAG_ACTIVITY_NEW_TASK}. -->
     <enum  />


```

1)、渣翻：

默认模式，通常会在 Activity 启动时创建一个新的实例，尽管这种行为可能会随着其他选项的引入而改变，比如 Intent.FLAG_ACTIVITY_NEW_TASK。

2)、google 文档定义：

> "standard"（默认模式）
> 
> 默认值。系统在启动该 Activity 的任务中创建 Activity 的新实例，并将 intent 传送给该实例。Activity 可以多次实例化，每个实例可以属于不同的任务，一个任务可以拥有多个实例。

### 1.2 standard 实际应用

新建一个 launchMode 声明为 standard 的 Activity，StandardActivity。

以 MainActivity 为起点，启动 StandardActivity，堆栈情况为：

```
        #6 Task=83 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{9aefb8a u0 com.test.la un ch mo de/.StandardActivity t83} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{3c3ba3c u0 com.test.la un ch mo de/.MainActivity t83} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

尝试再次启动 StandardActivity 一个的新的实例，结果堆栈情况为：

```
        #6 Task=83 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #2 ActivityRecord{9542c1 u0 com.test.la un ch mo de/.StandardActivity t83} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{9aefb8a u0 com.test.la un ch mo de/.StandardActivity t83} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{3c3ba3c u0 com.test.la un ch mo de/.MainActivity t83} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/086e0ab0577140b4878435d259d8a970~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

StandardActivity 的实例可以被多次创建，因此一个 StandardActivity 的新实例 StandardActivity2 被创建并且位于当前 Task 的 top。

2 singleTop
-----------

### 2.1 singleTop 定义

```
     <!-- If, when starting the activity, there is already an
         instance of the same activity class in the foreground that is
         interacting with the user, then
         re-use that instance.  This existing instance will receive a call to
         {@link android.app.Activity#onNewIntent Activity.onNewIntent()} with
         the new Intent that is being started. -->
     <enum  />


```

1)、渣翻：

当启动 Activity 时，在前台已经有一个与用户交互的相同 Activity 的实例，那么重用该实例。这个已经存在的实例将收到一个对 Activity.onNewIntent()} 的调用，传参是正在启动的 Intent。

2)、google 文档定义：

> "singleTop"
> 
> 如果当前任务的顶部已存在 Activity 的实例，则系统会通过调用其 onNewIntent() 方法来将 intent 转送给该实例，而不是创建 Activity 的新实例。Activity 可以多次实例化，每个实例可以属于不同的任务，一个任务可以拥有多个实例（但前提是返回堆栈顶部的 Activity 不是该 Activity 的现有实例）。
> 
> 例如，假设任务的返回堆栈包含根 Activity A 以及 Activity B、C 和位于顶部的 D（堆栈为 A-B-C-D；D 位于顶部）。收到以 D 类型 Activity 为目标的 intent。如果 D 采用默认的 "standard" 启动模式，则会启动该类的新实例，并且堆栈将变为 A-B-C-D-D。但是，如果 D 的启动模式为 "singleTop"，则 D 的现有实例会通过 onNewIntent() 接收 intent，因为它位于堆栈顶部，堆栈仍为 A-B-C-D。但是，如果收到以 B 类型 Activity 为目标的 intent，则会在堆栈中添加 B 的新实例，即使其启动模式为 "singleTop" 也是如此。
> 
> **注意**：创建 Activity 的新实例后，用户可以按返回按钮返回到上一个 Activity。但是，当由 Activity 的现有实例处理新 intent 时，用户将无法通过按返回按钮返回到 onNewIntent() 收到新 intent 之前的 Activity 状态。

### 2.2 singleTop 实际应用

新建一个 launchMode 声明为 singleTop 的 Activity，SingleTopActivity。

#### 2.2.1 SingleTopActivity 的实例已经处于 Task top 的情况下启动 SingleTopActivity

以 MainActivity 为起点，启动 SingleTopActivity：

```
        #6 Task=85 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{3373d6a u0 com.test.launchmode/.SingleTopActivity t85} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{d66e925 u0 com.test.launchmode/.MainActivity t85} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

此时再次尝试启动 SingleTopActivity：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/23612c7029024164a5fa1d922a204273~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

由于此时已经有一个 SingleTopActivity 的实例位于 Task 的 top，因此不会启动一个新的 SingleTopActivity，而是处于 Task top 的这个 SingleTopActivity 实例收到 Activity.onNewIntent 回调：

```
03-01 15:15:45.995  1483  6147 I wm_new_intent: [0,222456063,86,com.test.launchmode/.SingleTopActivity,NULL,NULL,NULL,0]
03-01 15:15:46.028  9926  9926 I wm_on_top_resumed_lost_called: [222456063,com.test.launchmode.SingleTopActivity,pausing]
03-01 15:15:46.029  9926  9926 I wm_on_paused_called: [222456063,com.test.launchmode.SingleTopActivity,performPause]
03-01 15:15:46.029  9926  9926 I launchmode_test: SingleTopActivity#onNewIntent
03-01 15:15:46.030  9926  9926 I wm_on_resume_called: [222456063,com.test.launchmode.SingleTopActivity,LIFECYCLER_RESUME_ACTIVITY]
03-01 15:15:46.030  9926  9926 I wm_on_top_resumed_gained_called: [222456063,com.test.launchmode.SingleTopActivity,topWhenResuming]


```

#### 2.2.2 SingleTopActivity 的实例没有处于 Task top 的情况下启动 SingleTopActivity

接 2.2.1，此时再启动一个 StandardActivity：

```
        #6 Task=85 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #2 ActivityRecord{7b9e87a u0 com.test.launchmode/.StandardActivity t85} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{3373d6a u0 com.test.launchmode/.SingleTopActivity t85} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{d66e925 u0 com.test.launchmode/.MainActivity t85} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

最后在 SingleTopActivity 不为 top 的情况再次尝试启动 SingleTopActivity：

```
        #6 Task=85 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #3 ActivityRecord{655dbef u0 com.test.launchmode/.SingleTopActivity t85} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #2 ActivityRecord{7b9e87a u0 com.test.launchmode/.StandardActivity t85} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{3373d6a u0 com.test.launchmode/.SingleTopActivity t85} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{d66e925 u0 com.test.launchmode/.MainActivity t85} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cbcf1638f8904410bab3d594cc7c1851~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这时由于之前启动的 SingleTopActivity 的实例 SingleTopActivity1 没有位于 top，所以这次会创建一个 SingleTopActivity 的新实例 SingleTopActivity2。

3 singleTask
------------

### 3.1 singleTask 定义

```
        <!-- If, when starting the activity, there is already a task running
            that starts with this activity, then instead of starting a new
            instance the current task is brought to the front.  The existing
            instance will receive a call to {@link android.app.Activity#onNewIntent
            Activity.onNewIntent()}
            with the new Intent that is being started, and with the
            {@link android.content.Intent#FLAG_ACTIVITY_BROUGHT_TO_FRONT
            Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT} flag set.  This is a superset
            of the singleTop mode, where if there is already an instance
            of the activity being started at the top of the stack, it will
            receive the Intent as described there (without the
            FLAG_ACTIVITY_BROUGHT_TO_FRONT flag set).  See the
            <a href="{@docRoot}guide/topics/fundamentals/tasks-and-back-stack.html">Tasks and Back
            Stack</a> document for more details about tasks.-->
        <enum  />


```

1)、渣翻：

当启动该 Activity 的时候，如果已经有一个正在运行的 Task 以该 Activity 启动，那么该 Task 将被移动到前台，而不是启动一个新的实例。现存的实例将会收到 Activity.onNewIntent() 方法的调用，传入正在启动的 Intent，这个 Intent 还会被设置 Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT。这是 singleTop 模式的超集。

2)、google 文档：

> "singleTask"
> 
> 系统会创建新任务，并实例化新任务的根 Activity。但是，如果另外的任务中已存在该 Activity 的实例，则系统会通过调用其 onNewIntent() 方法将 intent 转送到该现有实例，而不是创建新实例。Activity 一次只能有一个实例存在。
> 
> **注意**：虽然 Activity 在新任务中启动，但用户按返回按钮仍会返回到上一个 Activity。
> 
> 再举个例子，Android 浏览器应用在元素中指定 singleTask 启动模式，由此声明网络浏览器 Activity 应始终在它自己的任务中打开。这意味着，如果您的应用发出打开 Android 浏览器的 intent，系统不会将其 Activity 置于您的应用所在的任务中，而是会为浏览器启动一个新任务，如果浏览器已经有任务在后台运行，则会将该任务转到前台来处理新 intent。
> 
> 无论 Activity 是在新任务中启动的，还是在和启动它的 Activity 相同的任务中启动，用户按返回按钮都会回到上一个 Activity。但是，如果您启动了指定 singleTask 启动模式的 Activity，而后台任务中已存在该 Activity 的实例，则系统会将该后台任务整个转到前台运行。此时，返回堆栈包含了转到前台的任务中的所有 Activity，这些 Activity 都位于堆栈的顶部。图 4 展示了具体的情景。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/429b31ec8a514a488476f3682cca4c30~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> **图 4**. 采用 “singleTask” 启动模式的 Activity 添加到返回堆栈的过程图示。如果 Activity 已经存在于某个具有自己的返回堆栈的后台任务中，那么整个返回堆栈也会转到前台，覆盖当前任务。
> 
> 要详细了解如何在清单文件中设置启动模式，请参阅元素的说明文档，里面详细介绍了 launchMode 属性和可接受的值。
> 
> **注意**：您通过 launchMode 属性为 Activity 指定的行为，可被启动 Activity 的 intent 所包含的标记替换。

### 3.2 singleTask 实际应用

新建一个 launchMode 声明为 singleTask 的 Activity，SingleTaskActivity。

#### 3.2.1 启动 SingleTaskActivity

以 MainActivity 为起点：

```
        #6 Task=100 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{14b21d2 u0 com.test.launchmode/.MainActivity t100} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

启动 SingleTaskActivity：

```
        #6 Task=100 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{f14ffea u0 com.test.launchmode/.SingleTaskActivity t100} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{14b21d2 u0 com.test.launchmode/.MainActivity t100} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/891e80624d5b422d93e3dd16daebb163~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

新实例仍然在 Task#100 中，并没有想象中那样为 SingleTaskActivity 创建一个新的 Task。

#### 3.2.2 SingleTaskActivity 已经处于当前 Task 的 top 的情况下启动 SingleTaskActivity

接 3.2.1，此时再次尝试启动 SingleTaskActivity：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6373b9b294c4666b581541b6b70a92f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

由于此时已经有一个 SingleTaskActivity 的实例位于 Task 的 top，因此不会启动一个新的 SingleTaskActivity，而是处于 Task top 的这个 SingleTaskActivity 实例收到 Activity.onNewIntent 回调：

```
03-01 17:50:11.789  1483  6167 I wm_new_intent: [0,253034474,100,com.test.launchmode/.SingleTaskActivity,NULL,NULL,NULL,268435456]
03-01 17:50:11.791  1483  6167 I wm_task_moved: [100,1,6]
03-01 17:50:11.796  1483  6167 I wm_set_resumed_activity: [0,com.test.launchmode/.SingleTaskActivity,positionChildAt]
03-01 17:50:11.842 15874 15874 I wm_on_top_resumed_lost_called: [253034474,com.test.launchmode.SingleTaskActivity,pausing]
03-01 17:50:11.843 15874 15874 I wm_on_paused_called: [253034474,com.test.launchmode.SingleTaskActivity,performPause]
03-01 17:50:11.843 15874 15874 I launchmode_test: SingleTaskActivity#onNewIntent
03-01 17:50:11.844 15874 15874 I wm_on_resume_called: [253034474,com.test.launchmode.SingleTaskActivity,LIFECYCLER_RESUME_ACTIVITY]
03-01 17:50:11.845 15874 15874 I wm_on_top_resumed_gained_called: [253034474,com.test.launchmode.SingleTaskActivity,topWhenResuming]


```

#### 3.2.3 SingleTaskActivity 没有处于当前 Task 的 top 的情况下启动 SingleTaskActivity

接 3.2.2，启动一个 StandardActivity：

```
        #6 Task=100 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #2 ActivityRecord{c901e6c u0 com.test.launchmode/.StandardActivity t100} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{f14ffea u0 com.test.launchmode/.SingleTaskActivity t100} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{14b21d2 u0 com.test.launchmode/.MainActivity t100} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

然后再尝试启动 SingleTaskActivity：

```
        #6 Task=100 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{f14ffea u0 com.test.launchmode/.SingleTaskActivity t100} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{14b21d2 u0 com.test.launchmode/.MainActivity t100} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8100a96e36a4432ea7f73dbe4be1a6b4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

之前启动的 StandardActivity 被销毁，SingleTaskActivity 没有被重新创建，而是旧的实例收到 Activity.onNewIntent 回调。

```
03-01 17:52:13.212  1483  1500 I wm_finish_activity: [0,210771564,100,com.test.launchmode/.StandardActivity,clear-task-stack]
03-01 17:52:13.217  1483  1500 I wm_pause_activity: [0,210771564,com.test.launchmode/.StandardActivity,userLeaving=false,finish]
03-01 17:52:13.221  1483  1500 I wm_new_intent: [0,253034474,100,com.test.launchmode/.SingleTaskActivity,NULL,NULL,NULL,268435456]
03-01 17:52:13.222  1483  1500 I wm_task_moved: [100,1,6]
03-01 17:52:13.310 15874 15874 I wm_on_top_resumed_lost_called: [210771564,com.test.launchmode.StandardActivity,topStateChangedWhenResumed]
03-01 17:52:13.325 15874 15874 I wm_on_paused_called: [210771564,com.test.launchmode.StandardActivity,performPause]
03-01 17:52:13.330  1483  9284 I wm_add_to_stopping: [0,210771564,com.test.launchmode/.StandardActivity,completeFinishing]
03-01 17:52:13.379  1483  9284 I wm_set_resumed_activity: [0,com.test.launchmode/.SingleTaskActivity,resumeTopActivityInnerLocked]
03-01 17:52:13.470  1483  9284 I wm_resume_activity: [0,253034474,100,com.test.launchmode/.SingleTaskActivity]
03-01 17:52:13.577 15874 15874 I wm_on_restart_called: [253034474,com.test.launchmode.SingleTaskActivity,performRestartActivity]
03-01 17:52:13.577 15874 15874 I wm_on_start_called: [253034474,com.test.launchmode.SingleTaskActivity,handleStartActivity]
03-01 17:52:13.578 15874 15874 I launchmode_test: SingleTaskActivity#onNewIntent
03-01 17:52:13.579 15874 15874 I wm_on_resume_called: [253034474,com.test.launchmode.SingleTaskActivity,RESUME_ACTIVITY]
03-01 17:52:13.579 15874 15874 I wm_on_top_resumed_gained_called: [253034474,com.test.launchmode.SingleTaskActivity,topWhenResuming]
03-01 17:52:14.229  1483  1508 I wm_destroy_activity: [0,210771564,100,com.test.launchmode/.StandardActivity,finish-imm:idle]
03-01 17:52:14.363 15874 15874 I wm_on_stop_called: [210771564,com.test.launchmode.StandardActivity,LIFECYCLER_STOP_ACTIVITY]
03-01 17:52:14.365 15874 15874 I wm_on_destroy_called: [210771564,com.test.launchmode.StandardActivity,performDestroy]


```

和 Intent.FLAG_ACTIVITY_CLEAR_TOP 是一样的效果。

#### 3.2.4 启动设置了不同的 taskAffinity 的 SingleTaskActivity

在 3.2.1 中，我本来以为创建 SingleTaskActivity 的时候，会为其创建一个新的 Task，但是实际上 SingleTaskActivity 还是在现有的 Task 中启动了。跟了一下 Activity 启动的流程，发现，如果正在启动的 Activity 的 taskAffinity 和现存的某个 Task 的 rootAffinity 成员变量符合，那么这个 Activity 是有可能启动到这个 Task 中的，而不会去创建新的 Task。

查看一下 taskAffinity 的定义：

```
    <!-- Specify a task name that activities have an "affinity" to.
         Use with the application tag (to supply a default affinity for all
         activities in the application), or with the activity tag (to supply
         a specific affinity for that component).

         <p>The default value for this attribute is the same as the package
         name, indicating that all activities in the manifest should generally
         be considered a single "application" to the user.  You can use this
         attribute to modify that behavior: either giving them an affinity
         for another task, if the activities are intended to be part of that
         task from the user's perspective, or using an empty string for
         activities that have no affinity to a task. -->
    <attr  />



```

渣翻：指定 Activity 具有 “亲和性” 的 Task 名称。与应用程序标签一起使用(为应用程序中的所有 Activity 提供默认的 affinity)，或与 Activity 标签一起使用(为该组件提供特定的 affinity)。此属性的默认值与包名相同，表明 manifest 中的所有 Activity 通常应该被用户视为单个“应用程序”。您可以使用此属性来修改该行为：如果从用户的角度来看，Activity 是另外的一个 Task 的一部分，则为 Activity 提供一个另外一个 Task 的 affinity，或者为没有与任何 Task 关联的 Activity 使用一个空字符串。

SingleTaskActivity 由于没有显式指定一个 taskAffinity 属性，因此用的就是默认的包名，即”com.test.launchmode“，所以就会启动到现有的 Task 中。

这里我们修改 SingleTaskActivity 的 taskAffinity 属性：

```
android:taskAffinity="com.single.task"


```

以 MainActivity 为起点，再尝试启动 SingleTaskActivity：

```
        #7 Task=107 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{e7a99fa u0 com.test.launchmode/.SingleTaskActivity t107} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=106 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{94a375c u0 com.test.launchmode/.MainActivity t106} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/669d408a08ee401eb0136f9749ba461f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

果然设置了一个独特的 taskAffinity，SingleTaskActivity 就可以启动到另外一个 Task 中。

#### 3.2.5 SingleTaskActivity 的一个实例存在于某个后台 Task 中时从前台 Task 启动 SingleTaskActivity

接 3.2.4，通过 SingleTaskActivity 启动 StandardActivity：

```
       #7 Task=107 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{b2c985e u0 com.test.launchmode/.StandardActivity t107} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{e7a99fa u0 com.test.launchmode/.SingleTaskActivity t107} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=106 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{94a375c u0 com.test.launchmode/.MainActivity t106} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/44fc8389c00f45f1b6d4bc5a4600ca72~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

新的 StandardActivity 启动到了 SingleTaskActivity 所在的 Task 中。

然后将 MainActivity 所在的 Task 移动到前台：

```
        #7 Task=106 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{94a375c u0 com.test.launchmode/.MainActivity t106} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #5 Task=107 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{b2c985e u0 com.test.launchmode/.StandardActivity t107} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{e7a99fa u0 com.test.launchmode/.SingleTaskActivity t107} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cef6d2b443c14c35ac1288d71484e54a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

再次尝试通过 MainActivity 去启动 SingleTaskActivity：

```
        #7 Task=107 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{e7a99fa u0 com.test.launchmode/.SingleTaskActivity t107} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=106 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{94a375c u0 com.test.launchmode/.MainActivity t106} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/321ff8223aaf45eea7dab75e4d3f590e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

SingleTaskActivity 所在的 Task#107 移动到前台，SingleTaskActivity 收到 Activity.onNewIntent 回调，并且 StandardActivity 被干掉：

```
03-01 19:10:58.807  1483  1501 I wm_task_moved: [107,1,7]
03-01 19:10:58.807  1483  1501 I wm_task_to_front: [0,107]
03-01 19:10:58.815  1483  1501 I wm_focused_root_task: [0,0,107,106,bringingFoundTaskToFront]
03-01 19:10:58.829  1483  1501 I wm_set_resumed_activity: [0,com.test.launchmode/.StandardActivity,bringingFoundTaskToFront]
03-01 19:10:58.833  1483  1501 I wm_finish_activity: [0,187471966,107,com.test.launchmode/.StandardActivity,clear-task-stack]
03-01 19:10:58.834  1483  1501 I wm_destroy_activity: [0,187471966,107,com.test.launchmode/.StandardActivity,finish-imm:finishIfPossible]
03-01 19:10:58.839  1483  1501 I wm_new_intent: [0,242915834,107,com.test.launchmode/.SingleTaskActivity,NULL,NULL,NULL,268435456]
03-01 19:10:58.855  1483  1501 I wm_pause_activity: [0,155858780,com.test.launchmode/.MainActivity,userLeaving=true,pauseBackTasks]
03-01 19:10:58.920 20840 20840 I wm_on_top_resumed_lost_called: [155858780,com.test.launchmode.MainActivity,topStateChangedWhenResumed]
03-01 19:10:58.948 20840 20840 I wm_on_destroy_called: [187471966,com.test.launchmode.StandardActivity,performDestroy]
03-01 19:10:59.080 20840 20840 I wm_on_paused_called: [155858780,com.test.launchmode.MainActivity,performPause]
03-01 19:10:59.169  1483  1501 I wm_set_resumed_activity: [0,com.test.launchmode/.SingleTaskActivity,resumeTopActivityInnerLocked]
03-01 19:10:59.177  1483  1501 I wm_add_to_stopping: [0,155858780,com.test.launchmode/.MainActivity,makeInvisible]
03-01 19:10:59.216  1483  1501 I wm_resume_activity: [0,242915834,107,com.test.launchmode/.SingleTaskActivity]
03-01 19:10:59.308 20840 20840 I wm_on_restart_called: [242915834,com.test.launchmode.SingleTaskActivity,performRestartActivity]
03-01 19:10:59.309 20840 20840 I wm_on_start_called: [242915834,com.test.launchmode.SingleTaskActivity,handleStartActivity]
03-01 19:10:59.309 20840 20840 I launchmode_test: SingleTaskActivity#onNewIntent
03-01 19:10:59.310 20840 20840 I wm_on_resume_called: [242915834,com.test.launchmode.SingleTaskActivity,RESUME_ACTIVITY]
03-01 19:10:59.310 20840 20840 I wm_on_top_resumed_gained_called: [242915834,com.test.launchmode.SingleTaskActivity,topWhenResuming]
03-01 19:11:00.645  1483  1508 I wm_stop_activity: [0,155858780,com.test.launchmode/.MainActivity]
03-01 19:11:00.789 20840 20840 I wm_on_stop_called: [155858780,com.test.launchmode.MainActivity,STOP_ACTIVITY_ITEM]


```

销毁 StandardActivity 的逻辑和 3.2.3 是一样的。

4 singleInstance
----------------

### 4.1 singleInstance 定义

```
     <!-- Only allow one instance of this activity to ever be
         running.  This activity gets a unique task with only itself running
         in it; if it is ever launched again with the same Intent, then that
         task will be brought forward and its
         {@link android.app.Activity#onNewIntent Activity.onNewIntent()}
         method called.  If this
         activity tries to start a new activity, that new activity will be
         launched in a separate task.  See the
         <a href="{@docRoot}guide/topics/fundamentals/tasks-and-back-stack.html">Tasks and Back
         Stack</a> document for more details about tasks.-->
     <enum  />


```

1)、渣翻：

这个 Activity 只允许运行一个实例。这个 Activity 得到一个唯一的 Task，只有它自己在运行；如果它再次以相同的 Intent 启动，那么该 Task 将会被移动到前台，并且它的 Activity.onNewIntent() 方法被调用。如果这个 Activity 尝试启动一个新 Activity，这个新活动将在一个单独的 Task 中启动。

2)、google 文档：

> "singleInstance"
> 
> 与 "singleTask" 相似，唯一不同的是系统不会将任何其他 Activity 启动到包含该实例的任务中。该 Activity 始终是其任务唯一的成员；由该 Activity 启动的任何 Activity 都会在其他的任务中打开。

### 4.2 singleInstance 实际应用

新建一个 launchMode 声明为 singleInstance 的 Activity，SingleInstanceActivity。

#### 4.2.1 启动 SingleInstanceActivity

以 MainActivity 为起点：

```
        #6 Task=94 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{a19254e u0 com.test.launchmode/.MainActivity t94} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

启动 SingleInstanceActivity：

```
        #7 Task=95 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{3211027 u0 com.test.launchmode/.SingleInstanceActivity t95} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=94 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{a19254e u0 com.test.launchmode/.MainActivity t94} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b9cd984761144eab636862db81f6e38~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

此时为新启动的 SingleInstanceActivity 创建了一个新的 Task#95，并且 Task#95 移动到前台。

#### 4.2.2 在非 SingleInstanceActivity 所在的 Task 尝试创建一个新的 SingleInstanceActivity 实例

接 4.2.1，切换到 MainActivity 所在的 Task#94：

```
        #7 Task=94 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{eea3b93 u0 com.test.launchmode/.MainActivity t94} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=95 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{3211027 u0 com.test.launchmode/.SingleInstanceActivity t95} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/36a72007e313486d82621557ad13b485~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

在 MainActivity 界面再次尝试启动 SingleInstanceActivity：

```
        #7 Task=95 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{3211027 u0 com.test.launchmode/.SingleInstanceActivity t95} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=94 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{eea3b93 u0 com.test.launchmode/.MainActivity t94} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/00fc23db33f74b6999b7654cfc4cfad7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

之前创建的那个 SingleInstanceActivity 实例所在的 Task#95 整个被移动到前台，并不会再创建一个新的实例。

#### 4.2.3 在 SingleInstanceActivity 所在的 Task 尝试创建一个新的 SingleInstanceActivity 实例

接 4.2.2，再尝试启动 SingleInstanceActivity：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d41c52e10e524018b359c1bdc331bf81~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

结果是并不会创建一个新的实例，只有 SingleInstanceActivity 收到 Activity.onNewIntent 回调：

```
03-01 16:34:55.132  1483  1710 I wm_new_intent: [0,52498471,95,com.test.launchmode/.SingleInstanceActivity,NULL,NULL,NULL,268435456]
03-01 16:34:55.135  1483  1710 I wm_task_moved: [95,1,7]
03-01 16:34:55.142  1483  1710 I wm_set_resumed_activity: [0,com.test.launchmode/.SingleInstanceActivity,positionChildAt]
03-01 16:34:55.195 12995 12995 I wm_on_top_resumed_lost_called: [52498471,com.test.launchmode.SingleInstanceActivity,pausing]
03-01 16:34:55.196 12995 12995 I wm_on_paused_called: [52498471,com.test.launchmode.SingleInstanceActivity,performPause]
03-01 16:34:55.196 12995 12995 I launchmode_test: SingleInstance#onNewIntent
03-01 16:34:55.196 12995 12995 I wm_on_resume_called: [52498471,com.test.launchmode.SingleInstanceActivity,LIFECYCLER_RESUME_ACTIVITY]
03-01 16:34:55.196 12995 12995 I wm_on_top_resumed_gained_called: [52498471,com.test.launchmode.SingleInstanceActivity,topWhenResuming]


```

#### 4.2.4 在 SingleInstanceActivity 所在的 Task 启动其他 Activity

以 SingleInstanceActivity 为起点：

```
        #6 Task=98 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{e074c42 u0 com.test.launchmode/.SingleInstanceActivity t98} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

启动 StandardActivity：

```
        #7 Task=99 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{47012ae u0 com.test.launchmode/.StandardActivity t99} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=98 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{e074c42 u0 com.test.launchmode/.SingleInstanceActivity t98} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/686b97e688fd4fa7a9c1eab781ad4b65~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

StandardActivity 在一个新的 Task，Task#99 中被启动。

5 singleInstancePerTask
-----------------------

### 5.1 singleInstancePerTask 定义

Android 12 新增。

```
        <!-- The activity can only be running as the root activity of the task, the first activity
            that created the task, and therefore there will only be one instance of this activity
            in a task. In constrast to the {@code singleTask} launch mode, this activity can be
            started in multiple instances in different tasks if the
            {@code } or {@code FLAG_ACTIVITY_NEW_DOCUMENT} is set.-->
        <enum  />


```

渣翻：

该 Activity 只能作为 Task 的 root Activity 运行，即创建该 Task 的第一个 Activity，因此在一个 Task 中只能有一个该 Activity 的实例。与 singleTask 启动模式相比，如果设置了 FLAG_ACTIVITY_MULTIPLE_TASK 或 FLAG_ACTIVITY_NEW_DOCUMENT，这个 activity 可以在不同的 Task 中多个实例中启动。

### 5.2 singleInstancePerTask 实际应用

新建一个 launchMode 声明为 singleInstancePerTask 的 Activity，SingleInstancePerTaskActivity。

#### 5.2.1 启动 SingleInstancePerTaskActivity

以 MainActivity 为起点，启动 SingleInstancePerTaskActivity：

```
        #7 Task=230 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{91f591f u0 com.test.launchmodetest/.SingleInstancePerTaskActivity t230} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=229 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{37dc04d u0 com.test.launchmodetest/.MainActivity t229} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73915418873f4c70818d54cd5b9f0e92~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

看到这里直接为新启动的 SingleInstancePerTaskActivity 创建了一个新的 Task#230，而无需为其设置一个不同的 taskAffinity。

#### 5.2.2 当 SingleInstancePerTaskActivity 位于当前 Task top 时启动 SingleInstancePerTaskActivity

接 5.2.1，当启动了 SingleInstancePerTaskActivity，且为其创建了一个 Task#230 后，尝试再启动 SingleInstancePerTaskActivity，发现 SingleInstancePerTaskActivity 不会重复启动，而是现有的实例收到 Activity.onNewIntent 回调：

```
03-03 09:52:40.023  1490  1723 I wm_new_intent: [0,153049375,230,com.test.launchmodetest/.SingleInstancePerTaskActivity,NULL,NULL,NULL,268435456]
03-03 09:52:40.025  1490  1723 I wm_task_moved: [230,1,7]
03-03 09:52:40.030  1490  1723 I wm_set_resumed_activity: [0,com.test.launchmodetest/.SingleInstancePerTaskActivity,positionChildAt]
03-03 09:52:40.076 30409 30409 I wm_on_top_resumed_lost_called: [153049375,com.test.launchmodetest.SingleInstancePerTaskActivity,pausing]
03-03 09:52:40.077 30409 30409 I wm_on_paused_called: [153049375,com.test.launchmodetest.SingleInstancePerTaskActivity,performPause]
03-03 09:52:40.077 30409 30409 I launchmode_test: SingleInstancePerTaskActivity#onNewIntent
03-03 09:52:40.078 30409 30409 I wm_on_resume_called: [153049375,com.test.launchmodetest.SingleInstancePerTaskActivity,LIFECYCLER_RESUME_ACTIVITY]
03-03 09:52:40.079 30409 30409 I wm_on_top_resumed_gained_called: [153049375,com.test.launchmodetest.SingleInstancePerTaskActivity,topWhenResuming]


```

#### 5.2.3 当 SingleInstancePerTaskActivity 没有位于当前 Task top 时启动 SingleInstancePerTaskActivity

接 5.2.1，先通过 SingleInstancePerTaskActivity 创建一个 StandardActivity：

```
     #7 Task=230 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{eea3628 u0 com.test.launchmodetest/.StandardActivity t230} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{91f591f u0 com.test.launchmodetest/.SingleInstancePerTaskActivity t230} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

然后再去启动 SingleInstancePerTaskActivity：

```
        #7 Task=230 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{91f591f u0 com.test.launchmodetest/.SingleInstancePerTaskActivity t230} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e854a140e5e434590f6bc179464a526~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

结果是 StandardActivity 被销毁，SingleInstancePerTaskActivity 现有的实例收到 Activity.onNewIntent 回调：

```
03-03 09:58:44.128  1490  3749 I wm_finish_activity: [0,250230312,230,com.test.launchmodetest/.StandardActivity,clear-task-stack]
03-03 09:58:44.133  1490  3749 I wm_pause_activity: [0,250230312,com.test.launchmodetest/.StandardActivity,userLeaving=false,finish]
03-03 09:58:44.137  1490  3749 I wm_new_intent: [0,153049375,230,com.test.launchmodetest/.SingleInstancePerTaskActivity,NULL,NULL,NULL,268435456]
03-03 09:58:44.138  1490  3749 I wm_task_moved: [230,1,7]
03-03 09:58:44.207 30409 30409 I wm_on_top_resumed_lost_called: [250230312,com.test.launchmodetest.StandardActivity,topStateChangedWhenResumed]
03-03 09:58:44.214 30409 30409 I wm_on_paused_called: [250230312,com.test.launchmodetest.StandardActivity,performPause]
03-03 09:58:44.226  1490  2226 I wm_add_to_stopping: [0,250230312,com.test.launchmodetest/.StandardActivity,completeFinishing]
03-03 09:58:44.248  1490  2226 I wm_set_resumed_activity: [0,com.test.launchmodetest/.SingleInstancePerTaskActivity,resumeTopActivityInnerLocked]
03-03 09:58:44.287  1490  2226 I wm_resume_activity: [0,153049375,230,com.test.launchmodetest/.SingleInstancePerTaskActivity]
03-03 09:58:44.369 30409 30409 I wm_on_restart_called: [153049375,com.test.launchmodetest.SingleInstancePerTaskActivity,performRestartActivity]
03-03 09:58:44.369 30409 30409 I wm_on_start_called: [153049375,com.test.launchmodetest.SingleInstancePerTaskActivity,handleStartActivity]
03-03 09:58:44.370 30409 30409 I launchmode_test: SingleInstancePerTaskActivity#onNewIntent
03-03 09:58:44.370 30409 30409 I wm_on_resume_called: [153049375,com.test.launchmodetest.SingleInstancePerTaskActivity,RESUME_ACTIVITY]
03-03 09:58:44.370 30409 30409 I wm_on_top_resumed_gained_called: [153049375,com.test.launchmodetest.SingleInstancePerTaskActivity,topWhenResuming]
03-03 09:58:44.969  1490  1515 I wm_destroy_activity: [0,250230312,230,com.test.launchmodetest/.StandardActivity,finish-imm:idle]
03-03 09:58:45.104 30409 30409 I wm_on_stop_called: [250230312,com.test.launchmodetest.StandardActivity,LIFECYCLER_STOP_ACTIVITY]
03-03 09:58:45.105 30409 30409 I wm_on_destroy_called: [250230312,com.test.launchmodetest.StandardActivity,performDestroy]


```

#### 5.2.4 当 SingleInstancePerTaskActivity 存在的 Task 没有处于前台时去启动 SingleInstancePerTaskActivity

接 5.2.1，先通过 SingleInstancePerTaskActivity 创建一个 StandardActivity，然后将 Task 切换到 Task#229：

```
        #7 Task=229 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{37dc04d u0 com.test.launchmodetest/.MainActivity t229} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=230 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{db261fd u0 com.test.launchmodetest/.StandardActivity t230} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{91f591f u0 com.test.launchmodetest/.SingleInstancePerTaskActivity t230} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

然后通过 Task#229 的 MainActivity 再去启动 SingleInstancePerTaskActivity：

```
        #7 Task=230 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{91f591f u0 com.test.launchmodetest/.SingleInstancePerTaskActivity t230} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=229 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{37dc04d u0 com.test.launchmodetest/.MainActivity t229} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9d140bab16194e399deeed9c10ce5862~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

结果 Task#230 移动到前台，并且 StandardActivity 被销毁，SingleInstancePerTaskActivity 现有的实例收到 Activity.onNewIntent 回调：

```
03-03 10:02:35.383  1490  8118 I wm_task_moved: [230,1,7]
03-03 10:02:35.384  1490  8118 I wm_task_to_front: [0,230]
03-03 10:02:35.402  1490  8118 I wm_focused_root_task: [0,0,230,229,bringingFoundTaskToFront]
03-03 10:02:35.427  1490  8118 I wm_set_resumed_activity: [0,com.test.launchmodetest/.StandardActivity,bringingFoundTaskToFront]
03-03 10:02:35.430  1490  8118 I wm_finish_activity: [0,229794301,230,com.test.launchmodetest/.StandardActivity,clear-task-stack]
03-03 10:02:35.431  1490  8118 I wm_destroy_activity: [0,229794301,230,com.test.launchmodetest/.StandardActivity,finish-imm:finishIfPossible]
03-03 10:02:35.434  1490  8118 I wm_new_intent: [0,153049375,230,com.test.launchmodetest/.SingleInstancePerTaskActivity,NULL,NULL,NULL,268435456]
03-03 10:02:35.447  1490  8118 I wm_pause_activity: [0,58572877,com.test.launchmodetest/.MainActivity,userLeaving=true,pauseBackTasks]
03-03 10:02:35.489 30409 30409 I wm_on_top_resumed_lost_called: [58572877,com.test.launchmodetest.MainActivity,topStateChangedWhenResumed]
03-03 10:02:35.513 30409 30409 I wm_on_destroy_called: [229794301,com.test.launchmodetest.StandardActivity,performDestroy]
03-03 10:02:35.562 30409 30409 I wm_on_paused_called: [58572877,com.test.launchmodetest.MainActivity,performPause]
03-03 10:02:35.728  1490  2524 I wm_set_resumed_activity: [0,com.test.launchmodetest/.SingleInstancePerTaskActivity,resumeTopActivityInnerLocked]
03-03 10:02:35.749  1490  2524 I wm_add_to_stopping: [0,58572877,com.test.launchmodetest/.MainActivity,makeInvisible]
03-03 10:02:35.833  1490  2524 I wm_resume_activity: [0,153049375,230,com.test.launchmodetest/.SingleInstancePerTaskActivity]
03-03 10:02:35.949 30409 30409 I wm_on_restart_called: [153049375,com.test.launchmodetest.SingleInstancePerTaskActivity,performRestartActivity]
03-03 10:02:35.949 30409 30409 I wm_on_start_called: [153049375,com.test.launchmodetest.SingleInstancePerTaskActivity,handleStartActivity]
03-03 10:02:35.950 30409 30409 I launchmode_test: SingleInstancePerTaskActivity#onNewIntent
03-03 10:02:35.950 30409 30409 I wm_on_resume_called: [153049375,com.test.launchmodetest.SingleInstancePerTaskActivity,RESUME_ACTIVITY]
03-03 10:02:35.951 30409 30409 I wm_on_top_resumed_gained_called: [153049375,com.test.launchmodetest.SingleInstancePerTaskActivity,topWhenResuming]
03-03 10:02:37.430  1490  1515 I wm_stop_activity: [0,58572877,com.test.launchmodetest/.MainActivity]
03-03 10:02:37.829 30409 30409 I wm_on_stop_called: [58572877,com.test.launchmodetest.MainActivity,STOP_ACTIVITY_ITEM]


```

#### 5.2.5 结合 Intent.FLAG_ACTIVITY_MULTIPLE_TASK 和 Intent.FLAG_ACTIVITY_NEW_DOCUMENT

分析了上面四种情况，发现 singleInstancePerTask 的作用和 singleTask 几乎一样，不过 singleInstancePerTask 不需要为启动的 Activity 设置一个特殊的 taskAffinity 才能创建一个新的 Task。

根据 singleInstancePerTask 的定义，该 launchMode 和 Intent.FLAG_ACTIVITY_MULTIPLE_TASK 或 Intent.FLAG_ACTIVITY_NEW_DOCUMENT 结合使用可以将启动的 Activity 在多个 Task 中多次实例化，那么来看具体是怎么样的。这里每次启动 SingleInstancePerTaskActivity 时，都为启动 Intent 添加这两个 flag：

```
intent.addFlags(Intent.FLAG_ACTIVITY_MULTIPLE_TASK | Intent.FLAG_ACTIVITY_NEW_DOCUMENT);


```

还是以 MainActivity 为起点，然后启动 singleInstancePerTask，根据 5.2.1，知道此时会为新启动的 singleInstancePerTask 创建一个 Task：

```
        #7 Task=236 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{3a0de63 u0 com.test.launchmodetest/.SingleInstancePerTaskActivity t236} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=235 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{766c588 u0 com.test.launchmodetest/.MainActivity t235} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

接着通过 singleInstancePerTask 再去启动 singleInstancePerTask：

```
        #8 Task=237 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{518568f u0 com.test.launchmodetest/.SingleInstancePerTaskActivity t237} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #7 Task=236 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{3a0de63 u0 com.test.launchmodetest/.SingleInstancePerTaskActivity t236} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=235 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{766c588 u0 com.test.launchmodetest/.MainActivity t235} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c4ae0f2918e403d8dc60669b9966a15~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

果然不同于 5.2.2，这里又创建了一个 singleInstancePerTask 的新实例，且创建了一个新的 Task#237。

启动 Activity 时，通过在 startActivity() 的 intent 中添加相应的 flag 来修改 Activity 与其 Task 的默认关联。

1 FLAG_ACTIVITY_NEW_TASK
------------------------

### 1.1 FLAG_ACTIVITY_NEW_TASK 定义

```
    
     * If set, this activity will become the start of a new task on this
     * history stack.  A task (from the activity that started it to the
     * next task activity) defines an atomic group of activities that the
     * user can move to.  Tasks can be moved to the foreground and background;
     * all of the activities inside of a particular task always remain in
     * the same order.  See
     * <a href="{@docRoot}guide/topics/fundamentals/tasks-and-back-stack.html">Tasks and Back
     * Stack</a> for more information about tasks.
     *
     * <p>This flag is generally used by activities that want
     * to present a "launcher" style behavior: they give the user a list of
     * separate things that can be done, which otherwise run completely
     * independently of the activity launching them.
     *
     * <p>When using this flag, if a task is already running for the activity
     * you are now starting, then a new activity will not be started; instead,
     * the current task will simply be brought to the front of the screen with
     * the state it was last in.  See {@link #FLAG_ACTIVITY_MULTIPLE_TASK} for a flag
     * to disable this behavior.
     *
     * <p>This flag can not be used when the caller is requesting a result from
     * the activity being launched.
     */
    public static final int FLAG_ACTIVITY_NEW_TASK = 0x10000000;


```

1)、渣翻： 如果设置，这个 Activity 将成为一个新 Task 的堆栈中起始 Activity。一个 Task(从启动它的 Activity 到下一个 Task Activity) 定义了一个用户可以移动到的原子 Activity 组。Task 可以移动到前台和后台；特定 Task 中的所有 Activity 始终保持相同的顺序。

这个 flag 通常被想要呈现一个”Launcher” 风格行为的 Activity 使用：它们给用户一个可以做的单独事情的列表，否则这些事情完全独立于启动它们的 Activity 运行。 当使用这个 flag 时，如果一个 task 已经正在运行你启动的 activity，那么一个新的 activity 将不会被启动；相反，当前 Task 将简单地伴随着它最后一次进入的状态被带到屏幕前面。参见 FLAG_ACTIVITY_MULTIPLE_TASK 来禁用此行为。

当调用者从启动的 Activity 请求一个结果时，不能使用此 flag。

2)、google 文档：

> FLAG_ACTIVITY_NEW_TASK
> 
> 在新 Task 中启动 Activity。如果您现在启动的 Activity 已经有 Task 在运行，则系统会将该 Task 转到前台并恢复其最后的状态，而 Activity 将在 onNewIntent() 中收到新的 intent。
> 
> 这与 "singleTask" launchMode 值产生的行为相同。

### 1.2 FLAG_ACTIVITY_NEW_TASK 实际应用

#### 1.2.1 以 FLAG_ACTIVITY_NEW_TASK 的方式启动 Activity

这里我直接用 MainActivty 启动 StandardActivity，启动 Intent 中加入 FLAG_ACTIVITY_NEW_TASK：

```
        #6 Task=134 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{bf67e76 u0 com.test.launchmode/.StandardActivity t134} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{887cfac u0 com.test.launchmode/.MainActivity t134} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/96120ec3e34646108b9ac24ebea02f11~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

新启动的 StandardActivity 并没有像预料那样以一个新 Task 的方式启动，而是仍然创建在了当前 Task#134 中。

#### 1.2.2 在要启动的 Activity 已经位于 Task 的 top 的情况下以 FLAG_ACTIVITY_NEW_TASK 的方式启动 Activity

接 1.2.1，当已经有一个 StandardActivity 位于当前 Task 的 top 时，再去以 FLAG_ACTIVITY_NEW_TASK 的方式启动 StandardActivity：

```
        #6 Task=134 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #2 ActivityRecord{4ced944 u0 com.test.launchmode/.StandardActivity t193} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{bf67e76 u0 com.test.launchmode/.StandardActivity t134} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{887cfac u0 com.test.launchmode/.MainActivity t134} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8496833b66024523a9820373516401c8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

和 launchMode 为 “singleTask” 的情况也不一样，直接创建了一个新实例，singleTask 则是复用了旧实例。

#### 1.2.3 以 FLAG_ACTIVITY_NEW_TASK 的方式启动一个 taskAffinity 与 App 包名不同的 Activity

根据 google 文档的描述，FLAG_ACTIVITY_NEW_TASK 产生的行为与 launchMode 中的 singleTask 相同，那么没有创建新的 Task 的原因应该也是一样的，因此我这里新建了一个 DifferentAffinityActivity，给它设置一个不同于 App 包名的 taskAffinity，然后通过 MainActivity 以 FLAG_ACTIVITY_NEW_TASK 的方式去启动 DifferentAffinityActivity：

```
        #7 Task=2 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{dc31b80 u0 com.test.launchmodetest/.DifferentAffinityActivity t210} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=1 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{ecdd154 u0 com.test.launchmodetest/.MainActivity t209} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f643c04849184f3ba7da3140253591a6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

成功，系统为 StandardActivity 创建了一个 Task#2。

另外试验，如果只是为 DifferentAffinityActivity 设置一个特殊的 taskAffinity，但是启动它的时候不设置 FLAG_ACTIVITY_NEW_TASK，是不会创建新 Task 的。

#### 1.2.4 在 taskAffinity 不同的 Task 中以 FLAG_ACTIVITY_NEW_TASK 启动 Activity(1)

先启动 MainActivity，然后再通过 MainActivity 以 FLAG_ACTIVITY_NEW_TASK 的方式启动一个 StandardActivity，根据 1.2.1 可知，这两个 Activity 会在同一个 Task 中。

接着通过 StandardActivity 以 FLAG_ACTIVITY_NEW_TASK 的方式启动一个不同 taskAffinity 的 DifferentAffinityActivity，根据 1.2.3 可知，系统会为 DifferentAffinityActivity 创建一个新的 Task：

```
      #7 Task=208 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{82c008a u0 com.test.launchmodetest/.DifferentAffinityActivity t208} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=207 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{cdd2a3e u0 com.test.launchmodetest/.StandardActivity t207} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{9a2658a u0 com.test.launchmodetest/.MainActivity t207} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

Task#208 与现有的 Task#207 的 taskAffniity 不同，然后在 Task#208 中我们以 FLAG_ACTIVITY_NEW_TASK 的方式启动 StandardActivity，StandardActivity 的 taskAffinity 与 Task#207 是一致的：

```
      #7 Task=207 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #2 ActivityRecord{f0d64cb u0 com.test.launchmodetest/.StandardActivity t207} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{cdd2a3e u0 com.test.launchmodetest/.StandardActivity t207} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{9a2658a u0 com.test.launchmodetest/.MainActivity t207} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=208 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{82c008a u0 com.test.launchmodetest/.DifferentAffinityActivity t208} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48437647417947a0837d4cebae6b25f9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这里看到 StandardActivity2 没有创建到 Task#208 中，而是创建到了 Task#207 中，这不同于 launchMode 为 “singleTask” 的情况，singleTask 的时候如果 Task#207 中已经存在了正在启动的 Activity 的一个实例的话，是不会再启动第二个的。

再来对比一下，如果我们在 Task#208 中启动 StandardActivity 的时候，没有设置 FLAG_ACTIVITY_NEW_TASK，情况如何：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9ba847daf8b4e11a8aca61aa69c799f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这里看到 StandardActivity2 无视了 taskAffinity 的差异，直接创建在了 Task#208 中。

说明以 FLAG_ACTIVITY_NEW_TASK 的方式启动 Activity 的时候，会根据 taskAffinity 去寻找目标 Task。如果没有这个 flag，新的 Activity 直接无脑创建在当前 Task 中。

#### 1.2.5 在 taskAffinity 不同的 Task 中以 FLAG_ACTIVITY_NEW_TASK 启动 Activity(2)

在 1.2.4 中，看到了，当以 FLAG_ACTIVITY_NEW_TASK 启动的 Activity 时，如果启动的 Activity 的 taskAffinity 和当前 Task（Task#208）的 affinity 不同，那么这个 Activity 将会创建在另外一个和该 Activity 的 taskAffinity 相同的 Task（Task#207）中。

但是我在实际操作中也遇到了一种特殊情况：

先启动一个 MainActivity，存在于 Task#224。然后以 FLAG_ACTIVITY_NEW_TASK 启动 DifferentAffintyActivty，这个 Activity 由于 taskAffinity 与 Task#224 不同，所以会被创建在 Task#225 中，接着在 Task#225 中启动 SingleTopActivity，SingleTopActivity 虽然 taskAffinity 与 Task#225 不同，但是由于启动 SingleTopActivity 的时候没有声明 FLAG_ACTIVITY_NEW_TASK，因此 SingleTopActivity 会启动在 Task#225 中。接着将 Task#224 移动到前台：

```
      #7 Task=224 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{700b3e2 u0 com.test.launchmodetest/.MainActivity t224} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #5 Task=225 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{7cea9f5 u0 com.test.launchmodetest/.SingleTopActivity t225} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{14a0b22 u0 com.test.launchmodetest/.DifferentAffinityActivity t225} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

通过 Task#224 的 MainActivity 以 FLAG_ACTIVITY_NEW_TASK 的方式去启动 DifferentAffintyActivty：

```
        #7 Task=225 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{7cea9f5 u0 com.test.launchmodetest/.SingleTopActivity t225} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{14a0b22 u0 com.test.launchmodetest/.DifferentAffinityActivity t225} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #6 Task=224 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{700b3e2 u0 com.test.launchmodetest/.MainActivity t224} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ac7e52e22ed40018b11549c94d68991~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这里看到 Task#225 整个从后台移动到前台，除此之外没有任何改变。

1)、和启动 launchMode 为 singleTask 的情况不同，位于 StandardActivityz 之上的 SingleTopActivity 并没有被销毁，也就说这次启动只是将 StandardActivity 所在 Task#225 移动到前台，在 Task#225 内部并没有发生 Activity 的改变。

2)、按照 1.2.4 的结论，应该是在 Task#225 中再次创建一个 DifferentAffintyActivty 的实例，但是事实是 Task#225 只是从后台移动到了前台，没有任何新实例的创建。至于这一点的原因，跟了一下代码，是因为新启动的 DifferentAffintyActivty 是 Task#225 的 realActivity，而 1.2.4 中新启动的 StandardActivity 却不是 Task#207 的 realActivity，所以 1.2.4 和 1.2.5 走了不同的流程，最终 DifferentAffintyActivty 没有在 Task#225 中被再次创建，而 StandardActivity 在 Task#207 中是被再次创建了。

这里能看到 Activity 启动的流程要考虑的因素实在太多，只凭 taskAffinity、launchMode 和 Intent flag 是无法确定最终情况的。

2 FLAG_ACTIVITY_SINGLE_TOP
--------------------------

### 2.1 FLAG_ACTIVITY_SINGLE_TOP 定义

```
    
     * If set, the activity will not be launched if it is already running
     * at the top of the history stack.  See
     * <a href="{@docRoot}guide/topics/fundamentals/tasks-and-back-stack.html#TaskLaunchModes">
     * Tasks and Back Stack</a> for more information.
     */
    public static final int FLAG_ACTIVITY_SINGLE_TOP = 0x20000000;


```

1)、渣翻：

这个 flag 被设置了的情况下，如果这个 Activity 已经在当前 Task 的 top 运行，那么这个 Activity 将不会被重新启动。

2)、google 文档：

> FLAG_ACTIVITY_SINGLE_TOP
> 
> 如果要启动的 Activity 是当前 Activity（即位于返回堆栈顶部的 Activity），则现有实例会收到对 onNewIntent() 的调用，而不会创建 Activity 的新实例。 这与 "singleTop" launchMode 值产生的行为相同。

### 2.2 FLAG_ACTIVITY_SINGLE_TOP 实际应用

这里每次启动 StandardActivity 时，都在 Intent 中添加 FLAG_ACTIVITY_SINGLE_TOP，StandardActivity 的 launchMode 声明为 standard。

#### 2.2.1 StandardActivity 的实例已经处于 Task top 的情况下尝试启动另一个新实例

以 MainActivity 为起点，启动 StandardActivity：

```
        #6 Task=125 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{3d834ae u0 com.test.launchmode/.StandardActivity t125} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{a6de54b u0 com.test.launchmode/.MainActivity t125} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

此时再次以 FLAG_ACTIVITY_SINGLE_TOP 的方式启动 StandardActivity：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d496b6ef3f24c069acc281ae95c826f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

由于此时已经有一个 StandardActivity 的实例位于当前 Task 的 top，因此不会启动一个新的 StandardActivity，而是处于 Task top 的这个 StandardActivity 实例收到 Activity.onNewIntent 回调：

```
03-02 14:30:31.116  1490 31702 I wm_new_intent: [0,64500910,125,com.test.launchmode/.StandardActivity,NULL,NULL,NULL,536870912]
03-02 14:30:31.123  4100  4100 I wm_on_top_resumed_lost_called: [64500910,com.test.launchmode.StandardActivity,pausing]
03-02 14:30:31.125  4100  4100 I wm_on_paused_called: [64500910,com.test.launchmode.StandardActivity,performPause]
03-02 14:30:31.125  4100  4100 I launchmode_test: StandardActivity#onNewIntent
03-02 14:30:31.126  4100  4100 I wm_on_resume_called: [64500910,com.test.launchmode.StandardActivity,LIFECYCLER_RESUME_ACTIVITY]
03-02 14:30:31.127  4100  4100 I wm_on_top_resumed_gained_called: [64500910,com.test.launchmode.StandardActivity,topWhenResuming]


```

#### 2.2.2 StandardActivity 的实例没有处于 Task top 的情况下尝试启动另一个新实例

接 2.2.1，此时再启动一个 SingleTopActivity：

```
        #6 Task=125 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #2 ActivityRecord{9bd59d0 u0 com.test.launchmode/.SingleTopActivity t125} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{3d834ae u0 com.test.launchmode/.StandardActivity t125} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{a6de54b u0 com.test.launchmode/.MainActivity t125} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

此时 StandardActivity 不为 top，再次尝试启动 StandardActivity：

```
        #6 Task=125 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #3 ActivityRecord{1f5c973 u0 com.test.launchmode/.StandardActivity t125} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #2 ActivityRecord{9bd59d0 u0 com.test.launchmode/.SingleTopActivity t125} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{3d834ae u0 com.test.launchmode/.StandardActivity t125} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{a6de54b u0 com.test.launchmode/.MainActivity t125} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/244bb21e6cc24af9b6092495fbf28953~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

由于之前启动的 StandardActivity 的实例 StandardActivity1 没有位于 top，所以这次会创建一个 SingleTopActivity 的新实例 StandardActivity2。

和 launchModo 设置为 “singleTop” 的情形相同。

3 FLAG_ACTIVITY_CLEAR_TOP
-------------------------

### 3.1 FLAG_ACTIVITY_CLEAR_TOP 定义

```
    
     * If set, and the activity being launched is already running in the
     * current task, then instead of launching a new instance of that activity,
     * all of the other activities on top of it will be closed and this Intent
     * will be delivered to the (now on top) old activity as a new Intent.
     *
     * <p>For example, consider a task consisting of the activities: A, B, C, D.
     * If D calls startActivity() with an Intent that resolves to the component
     * of activity B, then C and D will be finished and B receive the given
     * Intent, resulting in the stack now being: A, B.
     *
     * <p>The currently running instance of activity B in the above example will
     * either receive the new intent you are starting here in its
     * onNewIntent() method, or be itself finished and restarted with the
     * new intent.  If it has declared its launch mode to be "multiple" (the
     * default) and you have not set {@link #FLAG_ACTIVITY_SINGLE_TOP} in
     * the same intent, then it will be finished and re-created; for all other
     * launch modes or if {@link #FLAG_ACTIVITY_SINGLE_TOP} is set then this
     * Intent will be delivered to the current instance's onNewIntent().
     *
     * <p>This launch mode can also be used to good effect in conjunction with
     * {@link #FLAG_ACTIVITY_NEW_TASK}: if used to start the root activity
     * of a task, it will bring any currently running instance of that task
     * to the foreground, and then clear it to its root state.  This is
     * especially useful, for example, when launching an activity from the
     * notification manager.
     *
     * <p>See
     * <a href="{@docRoot}guide/topics/fundamentals/tasks-and-back-stack.html">Tasks and Back
     * Stack</a> for more information about tasks.
     */
    public static final int FLAG_ACTIVITY_CLEAR_TOP = 0x04000000;


```

1)、渣翻： 在这个 flag 设置的情况，如果要启动的 Activity 已经在当前 Task 中运行，那么在该 Activity 上的所有其他 Activity 将被销毁，并且这个 Intent 将交付给 (现在处于 top) 旧 Activity 作为新 Intent，而不是启动一个该 Activity 的新的实例。

举个例子，现在有一个 Task 包含 4 个 Activity：A，B，C，D。如果 D 调用 startActivity 启动 B，那么 C 和 D 将会被销毁，然后 B 收到给定的 Intent，结果是，现在的 Task 堆栈情况是：A，B。

在上面的例子中，当前运行的 Activity B 的实例要么在它的在 onNewIntent() 方法里接收到你启动的新 Intent，要么自己被销毁并且以新的 Intent 重新启动。如果它已经声明了它的启动模式为 “multiple”(默认)，而你没有在同一个 Intent 中设置 FLAG_ACTIVITY_SINGLE_TOP，那么它将被销毁并重新创建；对于所有其他的启动模式，或者如果 FLAG_ACTIVITY_SINGLE_TOP 被设置，那么这个 Intent 将被发送到当前实例的 onNewIntent()。

这个启动模式和 FLAG_ACTIVITY_NEW_TASK 一起使用也可以达到很好的效果：如果用来启动一个 Task 的 root Activity，它会把 Task 中运行的实例带到前台，然后清除它到 root 状态。这特别有用，例如，在从 NotificationManager 启动 Activity 时。

2)、google 文档：

> FLAG_ACTIVITY_CLEAR_TOP
> 
> 如果要启动的 Activity 已经在当前 Task 中运行，则不会启动该 Activity 的新实例，而是会销毁位于它之上的所有其他 Activity，并通过 onNewIntent() 将此 intent 传送给它的已恢复实例（现在位于堆栈顶部）。
> 
> launchMode 属性没有可产生此行为的值。
> 
> FLAG_ACTIVITY_CLEAR_TOP 最常与 FLAG_ACTIVITY_NEW_TASK 结合使用。将这两个标记结合使用，可以查找其他 Task 中的现有 Activity，并将其置于能够响应 intent 的位置。
> 
> **注意**：如果指定 Activity 的启动模式为 "standard"，系统也会将其从堆栈中移除，并在它的位置启动一个新实例来处理传入的 intent。这是因为当启动模式为 "standard" 时，始终会为新 intent 创建新的实例。

### 3.2 FLAG_ACTIVITY_CLEAR_TOP 实际应用

这里每次启动测试 App 的主界面 MainActivity 时，都在 Intent 中添加 FLAG_ACTIVITY_CLEAR_TOP。

#### 3.2.1 MainActivity 的 launchMode 为 standard 的情况

首先在 MainActivity 界面上启动一个新的 StandardActivity：

```
        #6 Task=128 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{28de46f u0 com.test.launchmode/.StandardActivity t128} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{1ad7ad0 u0 com.test.launchmode/.MainActivity t128} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

然后通过 StandardActivity 再启动 MainActivity，启动的 Intent 加上 FLAG_ACTIVITY_CLEAR_TOP：

```
        #6 Task=128 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{aceb047 u0 com.test.launchmode/.MainActivity t128} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9c545abc4944a83bfdb4b82b550b0e1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

之前创建的 StandardActivity 被销毁，MainActivity 的实例 MainActivity1 被销毁然后重新创建了一个新实例 MainActivity2：

```
03-02 15:17:46.728  1490  5553 I wm_finish_activity: [0,42853487,128,com.test.launchmode/.StandardActivity,clear-task-stack]
03-02 15:17:46.731  1490  5553 I wm_pause_activity: [0,42853487,com.test.launchmode/.StandardActivity,userLeaving=false,finish]
03-02 15:17:46.732  1490  5553 I wm_finish_activity: [0,28146384,128,com.test.launchmode/.MainActivity,clear-task-top]
03-02 15:17:46.733  1490  5553 I wm_destroy_activity: [0,28146384,128,com.test.launchmode/.MainActivity,finish-imm:finishIfPossible]
03-02 15:17:46.743  1490  5553 I wm_task_moved: [128,1,6]
03-02 15:17:46.743  1490  5553 I wm_create_activity: [0,181317703,128,com.test.launchmode/.MainActivity,NULL,NULL,NULL,67108864]
03-02 15:17:46.759  6189  6189 I wm_on_top_resumed_lost_called: [42853487,com.test.launchmode.StandardActivity,topStateChangedWhenResumed]
03-02 15:17:46.763  6189  6189 I wm_on_paused_called: [42853487,com.test.launchmode.StandardActivity,performPause]
03-02 15:17:46.764  1490  5553 I wm_add_to_stopping: [0,42853487,com.test.launchmode/.StandardActivity,completeFinishing]
03-02 15:17:46.770  1490  5553 I wm_restart_activity: [0,181317703,128,com.test.launchmode/.MainActivity]
03-02 15:17:46.772  1490  5553 I wm_set_resumed_activity: [0,com.test.launchmode/.MainActivity,minimalResumeActivityLocked]
03-02 15:17:46.778  6189  6189 I wm_on_destroy_called: [28146384,com.test.launchmode.MainActivity,performDestroy]
03-02 15:17:46.816  6189  6189 I launchmode_test: MainActivity#onCreate
03-02 15:17:46.869  6189  6189 I wm_on_create_called: [181317703,com.test.launchmode.MainActivity,performCreate]
03-02 15:17:46.872  6189  6189 I wm_on_start_called: [181317703,com.test.launchmode.MainActivity,handleStartActivity]
03-02 15:17:46.873  6189  6189 I wm_on_resume_called: [181317703,com.test.launchmode.MainActivity,RESUME_ACTIVITY]
03-02 15:17:46.896  6189  6189 I wm_on_top_resumed_gained_called: [181317703,com.test.launchmode.MainActivity,topStateChangedWhenResumed]
03-02 15:17:47.016  1490  1512 I wm_activity_launch_time: [0,181317703,com.test.launchmode/.MainActivity,294]
03-02 15:17:47.230  1490  1515 I wm_destroy_activity: [0,42853487,128,com.test.launchmode/.StandardActivity,finish-imm:idle]
03-02 15:17:47.349  6189  6189 I wm_on_stop_called: [42853487,com.test.launchmode.StandardActivity,LIFECYCLER_STOP_ACTIVITY]
03-02 15:17:47.353  6189  6189 I wm_on_destroy_called: [42853487,com.test.launchmode.StandardActivity,performDestroy]


```

#### 3.2.2 MainActivity 的 launchMode 为 singleTop 的情况

测试一下 MainActivity 的 launchMode 如果不为默认的 standard 是什么现象，是否和定义中描述的行为一致，即 MainActivity 是否不会被销毁重建，因此这里设置 MainActivity 的 launchMode 为 singleTop。

首先在 MainActivity 界面上启动一个新的 StandardActivity：

```
        #6 Task=130 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #1 ActivityRecord{c383fe3 u0 com.test.launchmode/.StandardActivity t130} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{19f7db4 u0 com.test.launchmode/.MainActivity t130} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

然后通过 StandardActivity 再启动 MainActivity，启动的 Intent 加上 FLAG_ACTIVITY_CLEAR_TOP：

```
        #6 Task=130 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 ActivityRecord{19f7db4 u0 com.test.launchmode/.MainActivity t130} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]


```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f072ac5586e94bd0b1a016a5c9829cda~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

之前创建的 StandardActivity 被销毁，MainActivity 没有销毁，收到 Activity.onNewIntent 的回调：

```
03-02 15:24:34.710  1490  9314 I wm_finish_activity: [0,205012963,130,com.test.launchmode/.StandardActivity,clear-task-stack]
03-02 15:24:34.712  1490  9314 I wm_pause_activity: [0,205012963,com.test.launchmode/.StandardActivity,userLeaving=false,finish]
03-02 15:24:34.714  1490  9314 I wm_new_intent: [0,27229620,130,com.test.launchmode/.MainActivity,android.intent.action.MAIN,NULL,NULL,270532608]
03-02 15:24:34.715  1490  9314 I wm_task_moved: [130,1,6]
03-02 15:24:34.727  6902  6902 I wm_on_top_resumed_lost_called: [205012963,com.test.launchmode.StandardActivity,topStateChangedWhenResumed]
03-02 15:24:34.728  6902  6902 I wm_on_paused_called: [205012963,com.test.launchmode.StandardActivity,performPause]
03-02 15:24:34.729  1490  9314 I wm_add_to_stopping: [0,205012963,com.test.launchmode/.StandardActivity,completeFinishing]
03-02 15:24:34.734  1490  9314 I wm_set_resumed_activity: [0,com.test.launchmode/.MainActivity,resumeTopActivityInnerLocked]
03-02 15:24:34.743  1490  9314 I wm_resume_activity: [0,27229620,130,com.test.launchmode/.MainActivity]
03-02 15:24:34.786  6902  6902 I wm_on_restart_called: [27229620,com.test.launchmode.MainActivity,performRestartActivity]
03-02 15:24:34.786  6902  6902 I wm_on_start_called: [27229620,com.test.launchmode.MainActivity,handleStartActivity]
03-02 15:24:34.787  6902  6902 I launchmode_test: MainActivity#onNewIntent
03-02 15:24:34.787  6902  6902 I wm_on_resume_called: [27229620,com.test.launchmode.MainActivity,RESUME_ACTIVITY]
03-02 15:24:34.788  6902  6902 I wm_on_top_resumed_gained_called: [27229620,com.test.launchmode.MainActivity,topWhenResuming]
03-02 15:24:35.227  1490  1515 I wm_destroy_activity: [0,205012963,130,com.test.launchmode/.StandardActivity,finish-imm:idle]
03-02 15:24:35.248  6902  6902 I wm_on_stop_called: [205012963,com.test.launchmode.StandardActivity,LIFECYCLER_STOP_ACTIVITY]
03-02 15:24:35.249  6902  6902 I wm_on_destroy_called: [205012963,com.test.launchmode.StandardActivity,performDestroy]


```

1 launchMode
------------

### 1.1 standard

每次启动 Activity 都会在当前 Task 中创建一个 Activity 的新的实例。

### 1.2 singleTop

1)、如果要启动的 Activity 已经有一个对应的实例处于当前 Task 的 top，那么不会创建一个新实例。

2)、如果当前 Task 中运行了该 Activity 的一个实例，但是没有位于 Task 的 top，那么创建一个新实例。

### 1.3 singleTask

#### 1.3.1 要启动的 Activity 的 taskAffinity 和当前 Task 的 affinity（默认为 App 包名）相同

1)、如果要启动的 Activity 已经位于 Task 的 top，不会创建新的实例。

2)、如果当前 Task 中运行了该 Activity 的一个实例，但是没有位于 Task 的 top，那么清除掉该 Activity 之上的所有 Activity。

#### 1.3.2 要启动的 Activity 的 taskAffinity 和当前 Task 的 affinity（默认为 App 包名）不同

1)、如果当前 Task 中没有运行这个 Activity 的一个实例，那么在为这个 Activity 创建一个新的 Task。

2)、如果后台 Task 中有某个 Task 正在运行这个 Activity，那么不会创建新的实例，而是将该 Task 移动到前台，并且，移除掉该 Task 中位于这个 Activity 之上的所有 Activity。

### 1.4 singleInstance

如果有任何 Task 正在运行这个 Activity，将该 Task 移动到前台，否则总是为这个新启动的 Activity 创建新的 Task，并且这个 Task 中只能有这一个 Activity。

### 1.5 SingleInstancePerTask

默认作用和 singleTask 相似，不同的在于 singleInstancePerTask 不需要设置一个不同的 taskAffinity 即可创建新的 Task。

另外结合 Intent.FLAG_ACTIVITY_MULTIPLE_TASK 和 Intent.FLAG_ACTIVITY_NEW_DOCUMENT，每次启动了 launchMode 设置为 “singleInstancePerTask” 的 Activity 都可以创建一个新的 Task，那么这个新启动的 Activity 自然就是这个新创建的 Task 的 root Activity。

### 1.6 再次总结

说的白话一点，standard、singleTop、singleTask 和 singleinstance 这 4 种启动模式对 Activity 的实例数量要求是逐渐升级的：

standard：可以创建无数个 Activity 实例。

singleTop：如果当前 Task 的 top 如果已经是正在启动的这个 Activity，那就不要重复启动。

singleTask：这个 Activity 只能有一个，且作为它所在的 Task 的 root Activity（对于 taskAffinity 不同的情况）。

singleInstance：这个 Activity 只能有一个，且它所在的 Task 也只能有它这一个 Activity。

singleInstancePerTask：是 singleTask 的扩展，这个 Activity 可以有多个实例，但是每个都是所在的 Task 的 root Activity。

2 Intent flag
-------------

### 2.1 FLAG_ACTIVITY_NEW_TASK

如果有 FLAG_ACTIVITY_NEW_TASK 的参与，那么在启动 Activity 的时候就要把 taskAffinity 这一因素考虑进去，这会影响到该 Activity 是启动在当前 Task 中，还是新的 Task 中。根据之前的分析，这个 flag 的作用和 launchMode 为 “singleTask” 其实是不同的，比较复杂。

### 2.2 FLAG_ACTIVITY_SINGLE_TOP

作用和 launchMode 为 “singleTop” 相同。

### 2.3 FLAG_ACTIVITY_CLEAR_TOP

如果当前 Task 中正在运行这个 Activity，那么清除这个 Activity 之上的所有其他 Activity。

根据正在启动的这个 Activity 的 launchMode，以及启动的 Intent 的 flag，这个现有 Activity 可能也会被销毁重建。
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7350510977053245476)

> InputDispatcher 无焦点窗口 1 问题描述 Monkey 跑出的无焦点窗口的 ANR 问题。 特点： 1）、上层 WMS 有焦点窗口，为 Launcher。 2）、native 层 InputDispac

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/640ed62b83154e378d41e65d1ad4b44f~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1920&h=1080&s=1815226&e=jpg&b=aebcc9)

Monkey 跑出的无焦点窗口的 ANR 问题。

特点：

1）、上层 WMS 有焦点窗口，为 Launcher。

2）、native 层 InputDispacher 无焦点窗口，上层为”recents_animation_input_consumer“请求了焦点，但是”recents_animation_input_consumer“最终没有成为焦点窗口，原因是”NOT_VISIBLE“。

总共有两个项目报了类似的问题，log 分析如下：

第一份 log：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d8b9380cdd14184a48b4756a65d03b4~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1897&h=3617&s=1219281&e=png&b=fefdfd)

第 2 份 log：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0d4886f1a1d74081a26a1e39dcd83d2a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1897&h=2922&s=1043304&e=png&b=fefcfc)

从 log 中能看到，这两份 log 的相同点都是在调起 Recents 界面的时间点的附近启动了几个 “快速启动又销毁” 的 Activity，复现的场景比较相似，并且后续都是 “recents_animation_input_consumer” 取得了焦点后，又莫名的丢失了焦点：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/caf2b771cbb843e490b8b49c9340da6e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1833&h=562&s=173240&e=png&b=fffefe)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e01008a65d3d4591aec112039672bbe5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1826&h=438&s=136189&e=png&b=fffdfd)

只知道 “recents_animation_input_consumer” 丢失焦点的原因是“NOT_VISIBLE”，即它对应的 Layer 被认为是不可见的，但是具体的原因是什么呢？

这里经过多次尝试，终于根据第二份 log 的场景，复现了 ANR：

为了模拟问题场景，这类我们总共需要启动 4 个 Activity，并这为 4 个 Activity 设置：

```
android:screenOrientation="reversePortrait"


```

这个属性，用来模拟 Monkey 中出现 ROTATION_180 的场景。

具体场景为：

1）、MainActivity 连续启动 ActivityA、ActivityB、ActivityC，并且 MainActivity 调用 finish：

```
        startActivity(new Intent(MainActivity.this, ActivityA.class));
        startActivity(new Intent(MainActivity.this, ActivityB.class));
        startActivity(new Intent(MainActivity.this, ActivityC.class));
        finish();


```

2）、接着输入 KeyEvent.KEYCODE_RECENT_APPS，调起 Recents 界面，此时 “recents_animation_input_consumer” 会拿到焦点。

3）、让 ActivityC 在失去 top resumed 状态过后一段时间，调用 finish：

```
    public void onTopResumedActivityChanged(boolean isTopResumedActivity) {
        super.onTopResumedActivityChanged(isTopResumedActivity);
        if (!isTopResumedActivity) {
            Handler handler = new Handler();
            handler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    finish();
                }
            }, 2000);
        }
    }


```

4）、ActivityC 销毁回到 ActivityB、ActivityA 后，让他们也调用 finish：

```
    @Override
    protected void onStart() {
        super.onStart();
        finish();
    }


```

以上代码写好后，就可以复现 ANR 了，直接输入命令：

```
adb shell am start -n com.example.demoapp/.MainActivity && adb shell input keyevent 312


```

即可复现焦点从 “recents_animation_input_consumer” 离开的现象了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8ae07543efa4e86b03831e42658a183~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1228&h=21&s=9294&e=png&b=300a25)

再输入一个 KeyEvent 事件就可以触发 ANR 了。

同样在 pixel 上也能复现：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f963978746234dbfa4e2934ba2c461a5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1847&h=628&s=152559&e=png&b=fffefe)

从 “NOT_VISIBLE” 入手，去分析为什么 “recents_animation_input_consumer” 失去了焦点。

### 4.1 InputConsumerImpl.show

一开始最先想到的，就是没有为它的 SurfaceControl 在 InputMonitor.UpdateInputForAllWindowsConsumer.updateInputWindows 方法中调用 show：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d97be5c08114a17956bf7ae91732db5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=820&h=375&s=44191&e=png&b=ffffff)

“recents_animation_input_consumer” 会在 resetInputConsumers 方法中 hide，然后如果下面的条件满足，就会调用 show 去显示。

但是分析 log 可以知道，此时的 recents animation 还没有结束，因此这里的 activeRecents 是不会为空的，因此应该是调用了 show 方法的，并且后续我们复现问题后看 log 也的确如此。

接下来还是只能靠多添加 log 去分析。

### 4.2 FocusResolver.isTokenFocusable

首先是 FocusResolver.isTokenFocusable 中：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/94a913c10ea643a48cb130b5fbef544a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=795&h=504&s=50798&e=png&b=ffffff)

能够返回 Focusability::NOT_VISIBLE 说明是 WindowInfoHandle 的 WindowInfo 包含了 NOT_VISIBLE 标志位，而 WindowInfo 是在 SurfaceFlinger.buildWindowInfos 处构建的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c3f07d6b29b34550b52ad7e218d18db5~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=830&h=638&s=69953&e=png&b=ffffff)

并且能够看到，SurfaceFlinger.buildWindowInfos 中调用了 Layer.fillInputInfo 来填充 WindowInfo 的信息，和本题相关的就是 NOT_VISBLE 这个标志的设置，是根据 Layer.isVisibleForInput 的结果决定是否设置该标志位的。

继续打印 log，发现果然是 “recents_animation_input_consumer” 对应的 Layer 的 isVisibleForInput 返回了 false，导致这里为其 Layer 添加了 NOT_VISBLE 标志位。

### 4.3 Layer.isVisibleForInput

Layer.isVisibleForInput 函数的定义为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/faa8f4b9463740d280d1044b54b60522~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=708&h=228&s=22649&e=png&b=ffffff)

Layer.hasInputInfo 函数的内容为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/569216d6c572472b8b583f541fe08a3b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=832&h=80&s=6984&e=png&b=ffffff)

判断 Layer 是否设置过 WindowInfo。

而再次回顾 InputConsumerImpl 创建并且显示的逻辑：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/51dfa2f7972e4ce5ae92370f27987cd4~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=800&h=453&s=60088&e=png&b=fefdfd)

可知两点：

1）、在构造方法中创建了 InputWindowHandle 对象，并且在 SurfaceControl 显示的时候进行了 InputWindowHandle 的设置。

2）、在 SurfaceControl 显示的时候一同设置的，还有相对 Layer，并且该对象没有父 Layer，只有相对 Layer。

因此从上面的信息我们知道，“recents_animation_input_consumer” 对应的 Layer 的函数 hasInputInfo 是会返回 true 的，接下来需要继续分析 Layer.canReceiveInput 为何返回了 false。

### 4.4 Layer.canReceiveInput 和 Layer.isHiddenByPolicy

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d7a7047b01924a1791731bc9c4fc8b66~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=727&h=393&s=41303&e=png&b=fffefe)

Layer.canReceiveInput 首先是判断了 Layer.isHiddenByPolicy，并且从 log 中看到，“recents_animation_input_consumer” 的 Layer 的 isHiddenByPolicy 返回了 true。

看 Layer.isHiddenByPolicy 的内容能很明显的看到，当前 Layer 是否可见，实现是看其父 Layer 和相对 Layer 是否可见的，如果父 Layer 或者相对 Layer 是不可见的，那么该 Layer 就被直接认为是不可见的，不需要再继续看该 Layer 自身的设置了。

从上面的分析中我们得知，在本题中，“recents_animation_input_consumer” 的 Layer 是没有父 Layer 的，但是是有相对 Layer 的，即在 recents 动画中被设置为 InputMonitor.mActiveRecentsLayerRef 的那个 WindowContainer 的 Layer（目前 aosp14 的 dev 分支还是 ActivityRecord，我们的代码这里是 Task，我这里继续分析我们的代码），并且复现问题的时候，该 Task 的 isHiddenByPolicy 返回了 true：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a8919130d89413085773e8a50453226~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1789&h=285&s=82389&e=png&b=fffefe)

这一般就是上层 WMS 处为 Task 调用了 Transaction.hide。

### 4.5 Task.prepareSurfaces

最后最终到是在 Task.prepareSurfaces 中，认定该 Task 不在可见，所以调用 Transaction.setVisibility -> Transaction.hide 来为该 Task 的 Layer 设置了 hidden 的标志位：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1fa8aa1223da4bc2af531298eb093cc9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=846&h=743&s=77577&e=png&b=fffefe)

1）、Task 是否可见，看的是其 isVisible 方法，由于 Task 没有重写 isVisible 方法，因此这里调用的是 WindowContainer.isVisible 方法。

2）、WindowContainer.isVisible 的逻辑是，如果子 WindowContainer 中有一个是可见的，那么当前 WindowContainer 也会被认为是可见的。

3）、Task 的子容器都是 ActivityRecord，所以最终是看该 Task 中的 ActivityRecord 中有没有一个 ActivityRecord 的成员变量 mVisible 为 true。

很明显，在我们的复现问题的过程中，该 Task 中的所有 Activity 都被 finish 了，所以没有一个 ActivityRecord 是可见的，因此这个 Task 也被认为是不可见的，那么在 Task.prepareSurfaces 中，就会为该 Task 调用 Transaction.hide 设置 hidden 标志位，这导致在 SurfaceFlinger 处，该 Task 对应的 Layer 调用 isHiddenByPolicy 将返回 false，那么以该 Task 为相对 Layer 的 “recents_animation_input_consumer” 调用 isHiddenByPolicy 也只会返回 false，最终为 “recents_animation_input_consumer” 对应的 WindowInfo 设置了 NOT_VISIBLE 标志位，在 InputDispatcher 中被认为是不可见，不再满足作为焦点窗口的条件。

### 5.1 根本原因分析

这一题和我们之前解决的问题很像，同样是在 InputDispatcher 处，“recents_animation_input_consumer” 丢失焦点后，没有再次获得焦点，导致 InputDispatcher 这一侧的焦点窗口一直为 null，从而出现 ANR：

1）、之前的那个问题是 “recents_animation_input_consumer” 的相对 Layer 的那个 Task，整个被移除掉了，所以在 SurfaceFlinger 处，遍历不到 “recents_animation_input_consumer”，因此那个问题，焦点从“recents_animation_input_consumer” 离开的时候，原因是”NO_WINDOW“。

2）、本题中，“recents_animation_input_consumer”的相对 Layer 的那个 Task，虽然还存在，但是其中的所有 ActivityRecord，都调用了 finish，因此所有的 ActivityRecord 都是不可见的，因此这个 Task 也不可见，因此 “recents_animation_input_consumer” 也被认为是不可见，所以焦点从 “recents_animation_input_consumer” 离开的时候，原因是”NOT_VISIBLE“。

所以根本原因都是一致的，即：

1）、在 WMS 的 InputMonitor 处，它为 “recents_animation_input_consumer” 请求焦点的时候，是不在乎其相对 Layer 是什么状态的，只要满足 InputMonitor 要求的的条件，InputMonitor 就为 “recents_animation_input_consumer” 请求焦点。

2）、在 SurfaceFlinger 和 InputDispatcher 处，它们是会关心 “recents_animation_input_consumer” 的相对 Layer 是什么状态的，如果其相对 Layer 不可见，甚至不再存在了，那么就不会为 “recents_animation_input_consumer” 请求焦点。

正是这两侧为 “recents_animation_input_consumer” 请求焦点的相关逻辑的区别，导致了这类 ANR 问题的出现。

### 5.2 Task 没有被移除

另外还有一点疑问就是，“recents_animation_input_consumer”的相对 Layer 的那个 Task，虽然它里面的所有 Activity 都调用了 finish 了，但是还剩一个 “com.example.demoapp/.MainActivity” 没有被移除，还在 Task 里面：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0fb16c79ce1d47ed8dc5d4d22f202e2e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1670&h=183&s=40786&e=png&b=fffefe)

看到复现问题时候的 log：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47e18cbfe36142369cbfab10aa4ee7ad~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1830&h=836&s=201697&e=png&b=fffefe)

该 Activity 调用了 finish 后实际上并没有移除，而是一直在 Task 中，状态则是 STOPPING。

直到 recents 动画结束后，该 Activity 才被移除，然后是 Task 也被移除：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7ccf368a83243739e120065b9059be8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1913&h=223&s=77160&e=png&b=300a25)

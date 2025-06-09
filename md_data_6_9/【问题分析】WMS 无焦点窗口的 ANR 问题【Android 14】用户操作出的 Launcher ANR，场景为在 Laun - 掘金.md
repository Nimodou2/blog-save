> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7345379864738856979)

> 用户操作出的 Launcher ANR，场景为在 Launcher 界面一个 Activity 启动又快速销毁导致的无焦点窗口问题。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/20143966993d496c948a3718e9799ad1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2000&h=1125&s=2220810&e=jpg&b=152529)

用户操作出的 Launcher ANR，场景为在 Launcher 界面一个 Activity 启动又快速销毁导致的无焦点窗口问题。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/85cd76515b7d4ded9332fd174c8aec19~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1827&h=848&s=220145&e=png&b=fffefe) 最近分析 ANR 问题的时候，遇到多次在 Launcher 界面一个 Activity 启动又快速销毁导致无焦点窗口从而引起 ANR 的现象，并且本次 ANR 是人为操作出来的，并非 Monkey，因此需要继续根据上下文去看看问题场景，以及是否能够复现。

因为无法让微信 App 去按照问题发生时候的情况去行为，本地通过写一个 DemoApp 模仿微信的 App 的行为，初步稳定复现了该 ANR，且在 pixel 上也能稳定复现，先总结一下要点。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1fed4ad49e224b96ad597c75f42ebc29~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1834&h=682&s=163090&e=png&b=fffefe) 本地通过 DemoApp 复现问题的时候，发现通过点击 App 图标的方式启动 DemoApp 无法复现问题，而通过 adb 命令去启动 DemoApp 可以稳定复现，需要看下这里为什么会呈现差异，这个 ANR 毕竟是用户操作复现的，而非 Monkey。

### 3.1 adb 命令启动 Demo App

先尝试通过 adb 命令启动 Demo App 的方式去复现问题，发现当 MainActivity 对应的窗口挂掉后，是去更新了一次焦点窗口的，即走到了 DisplayContent.updateFocusedWindowLocked，就在窗口挂掉的流程中：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c147ccfb4cec466a96e8fe4017e8552c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=771&h=384&s=37319&e=png&b=fffefe)

WindowState.DeathRecipient.binderDied

-> WindowState.removeIfPossible

-> WMS.updateFocusedWindowLocked

log 如下：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/247f1dbef29b407fa680fc9f70d1d100~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1833&h=481&s=126460&e=png&b=fffefe) 但是这次发现所有窗口都不符合作为焦点窗口的条件，所以此次没有更新成功，并且只更新了这一次，因此后续 DisplayContent.mCurrentFocus 就始终为空了。

### 3.2 点击 App 图标启动 Demo App

如果是通过点击 App 图标的方式去启动 Demo App，发现有两次更新窗口焦点的操作。

第一次就是窗口挂掉的流程，这一次和上面分析的一样，也是没有找到合适的窗口。

接着第二次更新的时候，找到了符合条件的窗口，即 Launcher，并且将 DisplayContent.mCurrentFocus 设置为了 Launcher：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2c9c8c5b51674277a16fe856a97b55b3~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1837&h=548&s=155298&e=png&b=fffefe)

首先看到这次更新是因为移除 SnapshotStartingWindow 窗口触发的，而通过 adb 命令启动 Demo App 的时候并没有添加 SnapshotStartingWindow 窗口，因此通过 adb 命令的方式少了一次更新焦点窗口的操作，所以复现了问题。

另外查看问题 log，发现问题发生那次启动微信 App 的时候，是没有启动 SnapshotStartingWindow 的；

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6b5162cb55964d0884797b3695735454~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1901&h=270&s=98170&e=png&b=fffafa)

因此后续我们只要找到方法，使得通过点击 App 图标启动 Demo App 的时候，不要启动 SnapshotStartingWindow，就可以复现用户操作出的那种 ANR 了。

继续往 log 前面看，看下上一次启动微信 App 是什么时候，以及发生了什么。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e5c6fab2434b490f977d190f3eb18b11~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1567&h=436&s=99720&e=png&b=fffdfd)

至此，我们已经找到复现该 ANR 所需要的所有操作，同样 Pixel 也可以复现：

1、从 Launcher 启动 Demo App，然后点击 Home 键回到 Launcher（这里不是像问题发生的时候那样点击 Back 键，后面会解释），让 Demo App 处于后台。

2、让 Demo App 自己启动自己一次，此时会因为 BAL_BLOCK 无法启动成功（如果上一步是点击点击 Back 键回到 Launcher，那么这里可能会成功启动且显示 BAL_ALLOW_GRACE_PERIOD，如果是这样那么需要将这次启动 Demo App 的时间距离第一步启动 Demo App 的时间间隔大于 10s，所以为了省事，第一步直接通过点击 Home 键回到 Launcher，而非 Back 键），但是无所谓，这一步主要是为了让后续从 Launcher 启动 Demo App 的时候不会启动 SnapshotStartingWindow。

3、在 Launcher 界面点击 App 图标启动 Demo App，Demo App 启动后需要首先 finish，接着 crash，此时可以复现 DisplayContent.mCurrentFocus 为 null，即无焦点窗口的情况。

4、在无焦点窗口的情况下，输入一个 KeyEvent，我们这里模仿用户输入了一个 keycode 为 98 的 KeyEvent.KEYCODE_BUTTON_C，即可发生 ANR：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3a62703ce51466ab16c28dea50714cf~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1080&h=2400&s=1823994&e=png&b=151515)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1066aa32cfdb49e1b4b8570cc5c81408~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1894&h=267&s=88029&e=png&b=300a25)

这里为复现问题的 Demo App 代码：

```
public class MainActivity extends Activity {
    int countDown = 1;
    private Activity test = new Activity();
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    @Override
    protected void onStart() {
        super.onStart();
        if (countDown == 0) {
            finish();
            test = null;
            test.finish();
        }
        if (countDown == 1) {
            Handler handler = new Handler();
            handler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    Intent intent = new Intent(MainActivity.this, MainActivity.class);
                    startActivity(intent);
                }
            }, 5000);
        }
        countDown--;
    }
}


```

最后分析为什么会出现 ANR，这里根据我本地复现的 log 进行分析，从窗口挂掉后，有 3 个时间点都是有机会更新焦点窗口的。

### 5.1 关键点 1

首先是 Demo App 挂掉后，移除 WindowState 的流程：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47fb74635cbd4e57bccf9e3eed1a22ca~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1816&h=177&s=42467&e=png&b=fffefe)

从 log 能看到这里其实是去更新了一次焦点窗口的，但是此次由于 Launcher 对应的 ActivityRecord.mVisibleRequested 为 false，所以不满足作为焦点窗口的条件，因此这次更新没有找到合适的窗口作为焦点窗口。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a78d1c4d4dd947bfb4c704e3e03de6d8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=823&h=422&s=45710&e=png&b=fefdfd)

### 5.2 关键点 2

接着是移除 ActivityRecord 的流程：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9e9161bb60e4f558557c51151b3c955~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1754&h=262&s=69998&e=png&b=fefdfd)

在这里我们才看到将 launcher 的 ActivityRecord.mVisibleRequested 可见性改为 true，此流程下有一次更新焦点窗口的机会：

ActivityRecord.setVisibility

​ -> ActivityRecord.commitVisibility

​ -> WMS.updateFocusedWindowLocked

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0d76a789414c498bb00d8f37ea35b377~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=814&h=614&s=49403&e=png&b=ffffff)

但是由于此时由于处于 transition collecting 的阶段，因此最终提前返回了，没有继续调用 ActivityRecord.commitVisibility，ActivityRecord.commitVisibility 中是有机会继续调用 WMS.updateFocusedWindowLocked 来更新焦点窗口的。

### 5.3 关键点 3

最后是 Launcher 对应的 WindowState 走 WMS.relayoutWindow：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ec73546530640abb5446bc1a604bfcf~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1845&h=106&s=24393&e=png&b=fffefe)

本来也是有机会调用 WMS.updateFocusedWindowLocked：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/41bf087bd6f3449692cc965a65ae7277~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=821&h=505&s=47501&e=png&b=fffefe)

但是由于 Launcher 对应的窗口在 relayoutWindow 前后 WindowState.mViewVisibility 都是 View.VISIBLE，原因则是 WindowState.mViewVisibility 改变的地方只有一处，在 WindowState.setViewVisibility：

```
    void setViewVisibility(int viewVisibility) {
        mViewVisibility = viewVisibility;
    }


```

该方法只在 WMS.relayoutWindow 中调用：

```
            win.setViewVisibility(viewVisibility);
            ProtoLog.i(WM_DEBUG_SCREEN_ON,
                    "Relayout %s: oldVis=%d newVis=%d. %s", win, oldVisibility,
                            viewVisibility, new RuntimeException().fillInStackTrace());


```

而”com.example.demoapp.MainActivity“启动后又迅速挂掉了，没来得及改变 Launcher 对应的窗口 WindowState.mViewVisibility，因此认为不存在焦点的切换，最后也是没有去更新焦点窗口。

从 log 上也能看到 Launcher 对应的窗口只在 Demo App 挂掉后 relayout 了两次，且每次传入的 viewVisibility 都是 View.VISIBLE：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dbb277f17e9845a597c48f2975504c4c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1841&h=479&s=126411&e=png&b=fffefe)

因此在 Launcher 的窗口走 WMS.relayoutWindow 流程的时候，也错失了更新焦点窗口的机会，导致后续焦点窗口一直都是空了。
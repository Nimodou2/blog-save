> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/126747771?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/5244e2e6dfaa4def9d92fbaa4570194f.jpeg#pic_center)

一、定义
----

```
/**
 * Represents a collection of operations on some WindowContainers that should be applied all at
 * once.
 *
 * @hide
 */
@TestApi
public final class WindowContainerTransaction implements Parcelable {
    ......
}
```

WindowContainerTransaction 表示一些 WindowContainer 上应该一次性应用的操作集合。

从使用意义上来看，WindowContainerTransaction 类和 Transaction 类比较相似，Transaction 是应用在 SurfaceControl 上的操作集合，WindowContainerTransaction 是应用在 WindowContainer 上的操作集合。另外 WindowContainerTransaction 也实现了 Parcelable，这为其在系统服务端和 App 端之间的传输提供了支持。

二、使用
----

结合分屏的一处逻辑，看下 WindowContainerTransaction 是如何使用的，以下是进入分屏的过程中 LegacySplitScreenController#splitPrimaryTask 做的事情：

```
public boolean splitPrimaryTask() {
        ......
        final WindowContainerTransaction wct = new WindowContainerTransaction();
        // Clear out current windowing mode before reparenting to split task.
        wct.setWindowingMode(topRunningTask.token, WINDOWING_MODE_UNDEFINED);
        wct.reparent(topRunningTask.token, mSplits.mPrimary.token, true /* onTop */);
        mWindowManagerProxy.applySyncTransaction(wct);
        return true;
    }
```

1)、创建一个 WindowContainerTransaction 对象。

2)、调用 WindowContainerTransaction#setWindowingMode。

3)、调用 WindowContainerTransaction#reparent。

4)、调用 WindowManagerProxy#applySyncTransaction 发送当前 WindowContainerTransaction，关于 WCT 的发送留在以后分析，这里重点分析第 2、3 点。

### 1 WindowContainerTransaction#setWindowingMode

```
/**
     * Sets the windowing mode of the given container.
     */
    @NonNull
    public WindowContainerTransaction setWindowingMode(
            @NonNull WindowContainerToken container, int windowingMode) {
        Change chg = getOrCreateChange(container.asBinder());
        chg.mWindowingMode = windowingMode;
        return this;
    }
```

内容很简单：

1)、根据传入的 WindowContainerToken 对象，通过 WindowContainerTransaction#getOrCreateChange 方法获取一个 Change 对象。

2)、将 Change 的成员变量 mWindowingMode 赋值为传入的 windowingMode。

在分析 WindowContainerTransaction#getOrCreateChange 方法之前，先看下 WindowContainerToken 的作用：

```
/**
 * Interface for a window container to communicate with the window manager. This also acts as a
 * token.
 * @hide
 */
@TestApi
public final class WindowContainerToken implements Parcelable {

    private final IWindowContainerToken mRealToken;

    /** @hide */
    public WindowContainerToken(IWindowContainerToken realToken) {
        mRealToken = realToken;
    }

    private WindowContainerToken(Parcel in) {
        mRealToken = IWindowContainerToken.Stub.asInterface(in.readStrongBinder());
    }

    /** @hide */
    public IBinder asBinder() {
        return mRealToken.asBinder();
    }

	......
}
```

WindowContainterToken 实现了 Parcelable，这为其跨进程传输提供了支持。WindowContainterToken 内部有一个成员变量 mRealToken，是一个 IWindowContainerToken 类型的 token，在系统服务创建 WindowContainer 时候生成（目前只有 DisplayArea 和 Task），对于 WindowContainer 来说是一个独特的跨进程的标识。WindowContainerToken 可以看做是 IBinder 类型的 token 的封装，这个 token 通过 WindowContainerToken#asBinder 返回。

Task 可以通过 Task#fillTaskInfo 方法将该 Task 对应的 WindowContainerToken 保存在对应的 TaskInfo 中，这样 App 进程可以先获取 TaskInfo 进而拿到这个 token。

App 端如果想通过 WindowContainerTransaction 的方法修改某个 WindowContainer 的属性，必须传入该 WindowContainer 对应的 token，这样系统服务端在接收到这个 WindowContainerTransaction 的时候，才可以通过 token 知道需要修改哪些 WindowContainer。

接着看下 WindowContainerTransaction#getOrCreateChange 的内容：

```
private Change getOrCreateChange(IBinder token) {
        Change out = mChanges.get(token);
        if (out == null) {
            out = new Change();
            mChanges.put(token, out);
        }
        return out;
    }
```

从 mChanges 中查找该 WindowContainerToken 有没有相应的 Change 对象，没有就创建一个，然后以键值对的方式把这个 WindowContainerToken 对象和为其创建的 Change 对象加入到 mChanges 中。

mChanges 是一个 ArrayMap 类型的 WindowContainerTransaction 的成员变量，以 IBinder 类型的 WindowContainerToken 对象为 key，以 WindowContainerTransaction 为该 WindowContainerToken 创建的 Change 对象为 value：

```
private final ArrayMap<IBinder, Change> mChanges = new ArrayMap<>();
```

后续该 WindowContainerTransaction 发送到 WM 端的时候，WindowOrganizerController 遍历这个 WindowContainerTransaction 的 mChanges 成员变量，对于 mChanges 中的每一个 Change 对象：

1)、从该 Change 中提取出 WindowContainerToken，将该 WindowContainerToken 转化为 WM 端对应的 WindowContainer 对象。

2)、提取出 Change 中为该 WindowContainerToken 保存的 WindowingMode，然后应用到第 1 步中转化得到的 WindowContainer。

上面的第 2 步可以看 WindowOrganizerController#applyChanges 方法进一步了解一下：

```
private int applyChanges(WindowContainer container, WindowContainerTransaction.Change change) {
        .......
        final int windowingMode = change.getWindowingMode();
        ......
        if (windowingMode > -1) {
            ......
            container.setWindowingMode(windowingMode);
        }
        return effects;
    }
```

看到这里似乎对 WindowContainerTransaction 的运作方式有了一部分的了解了，App 端如果想要通过 WindowContainerTransaction 修改 WM 端的 WindowContainter 的属性，需要这几步操作：

1)、App 端首先要能够拿到这个 WindowContainer 对应的 WindowContainerToken。

2)、创建一个 WindowContainerTransaction 对象，调用 WindowContainerTransaction 的相关方法对这个 WindowContainerToken 进行设置。

3)、发送 WindowContainerTransaction。

### 2 WindowContainerTransaction#setFocusable

为了印证这个猜想，再看另外一个类似的方法 WindowContainerTransaction#setFocusable。调用的地方在分屏相关逻辑处：

```
public void setHomeMinimized(final boolean minimized) {
		......
        WindowContainerTransaction wct = new WindowContainerTransaction();
        final boolean minimizedChanged = mMinimized != minimized;
        // Update minimized state
        if (minimizedChanged) {
            mMinimized = minimized;
        }
        // Always set this because we could be entering split when mMinimized is already true
        wct.setFocusable(mSplits.mPrimary.token, !mMinimized);

		......
    }
```

先创建一个 WindowContainerTransaction，然后调用 WindowContainerTransaction.setFocusable 方法设置：

```
/**
     * Sets whether a container or any of its children can be focusable. When {@code false}, no
     * child can be focused; however, when {@code true}, it is still possible for children to be
     * non-focusable due to WM policy.
     */
    @NonNull
    public WindowContainerTransaction setFocusable(
            @NonNull WindowContainerToken container, boolean focusable) {
        Change chg = getOrCreateChange(container.asBinder());
        chg.mFocusable = focusable;
        chg.mChangeMask |= Change.CHANGE_FOCUSABLE;
        return this;
    }
```

该方法用来设置一个 WindowContainer 和其子 WindowContainer 是否可以获取焦点，同样是两步操作：

1)、根据传入的 WindowContainerToken 对象，通过 WindowContainerTransaction#getOrCreateChange 方法获取一个 Change 对象。

2)、将 Change 的成员变量 mFocusable 赋值为传入的 focusable 参数。

Change.mChange 应用的地方依然是 WindowOrganizerController#applyChanges：

```
private int applyChanges(WindowContainer container, WindowContainerTransaction.Change change) {
        ......
        if ((change.getChangeMask() & WindowContainerTransaction.Change.CHANGE_FOCUSABLE) != 0) {
            if (container.setFocusable(change.getFocusable())) {
                effects |= TRANSACT_EFFECTS_LIFECYCLE;
            }
        }
        ......
    }
```

套路都是一样的：

1)、从该 Change 中提取出 WindowContainerToken，将该 WindowContainerToken 转化为 WM 端对应的 WindowContainer 对象。

2)、提取出 Change 中为该 WindowContainerToken 保存的 focusable 属性，然后应用到第 1 步中转化得到的 WindowContainer。

### 3 WindowContainerTransaction.Change 类

```
/**
     * Holds changes on a single WindowContainer including Configuration changes.
     * @hide
     */
    public static class Change implements Parcelable {
        public static final int CHANGE_FOCUSABLE = 1;
        public static final int CHANGE_BOUNDS_TRANSACTION = 1 << 1;
        public static final int CHANGE_PIP_CALLBACK = 1 << 2;
        public static final int CHANGE_HIDDEN = 1 << 3;
        public static final int CHANGE_BOUNDS_TRANSACTION_RECT = 1 << 4;
        public static final int CHANGE_IGNORE_ORIENTATION_REQUEST = 1 << 5;
        private final Configuration mConfiguration = new Configuration();
        private boolean mFocusable = true;
        private boolean mHidden = false;
        private boolean mIgnoreOrientationRequest = false;
        private int mChangeMask = 0;
        private @ActivityInfo.Config int mConfigSetMask = 0;
        private @WindowConfiguration.WindowConfig int mWindowSetMask = 0;
        private Rect mPinnedBounds = null;
        private SurfaceControl.Transaction mBoundsChangeTransaction = null;
        private Rect mBoundsChangeSurfaceBounds = null;
        private int mActivityWindowingMode = -1;
        private int mWindowingMode = -1;
        ......
        public int getWindowingMode() {
            return mWindowingMode;
        }
        public int getActivityWindowingMode() {
            return mActivityWindowingMode;
        }
        public Configuration getConfiguration() {
            return mConfiguration;
        }
        /** Gets the requested focusable state */
        public boolean getFocusable() {
            if ((mChangeMask & CHANGE_FOCUSABLE) == 0) {
                throw new RuntimeException("Focusable not set. check CHANGE_FOCUSABLE first");
            }
            return mFocusable;
        }
        /** Gets the requested hidden state */
        public boolean getHidden() {
            if ((mChangeMask & CHANGE_HIDDEN) == 0) {
                throw new RuntimeException("Hidden not set. check CHANGE_HIDDEN first");
            }
            return mHidden;
        }
        /** Gets the requested state of whether to ignore orientation request. */
        public boolean getIgnoreOrientationRequest() {
            if ((mChangeMask & CHANGE_IGNORE_ORIENTATION_REQUEST) == 0) {
                throw new RuntimeException("IgnoreOrientationRequest not set. "
                        + "Check CHANGE_IGNORE_ORIENTATION_REQUEST first");
            }
            return mIgnoreOrientationRequest;
        }
        public int getChangeMask() {
            return mChangeMask;
        }
        @ActivityInfo.Config
        public int getConfigSetMask() {
            return mConfigSetMask;
        }
        @WindowConfiguration.WindowConfig
        public int getWindowSetMask() {
            return mWindowSetMask;
        }
        /**
         * Returns the bounds to be used for scheduling the enter pip callback
         * or null if no callback is to be scheduled.
         */
        public Rect getEnterPipBounds() {
            return mPinnedBounds;
        }
        public SurfaceControl.Transaction getBoundsChangeTransaction() {
            return mBoundsChangeTransaction;
        }
        public Rect getBoundsChangeSurfaceBounds() {
            return mBoundsChangeSurfaceBounds; 
        }
        ......
    }
```

持有对单一 WindowContainer 的包括 Configuration 的修改。

通过对 WindowContainerTransaction#setWindowingMode 和 WindowContainerTransaction#setFocusable 这两部分的分析，可以看到不管是改变 WindowContainer 的 windowingMode，或是 focusable 属性，都需要：

1)、创建一个 WindowContainerTransaction 对象，调用 WindowContainerTransaction 提供的方法对 WindowContainerToken 进行设置，实际上就是将客户端对 WindowContainer 期望的一些修改先保存到 WindowContainerTransaction 为 WindowContainerToken 创建的 Change 对象的相关成员变量中。

2)、发送创建的 WindowContainerTransaction 到 WM 端，WM 端获取到 WindowContainerTransaction.Change 相关成员变量的值，然后应用到 WindowContainer 上。

那么也就是说，WindowContainerTransaction.Change 有多少成员变量，WindowContainerTransaction 就可以向客户端提供多少可以改变 WindowContainer 的方法接口。

看下 WindowContainerTransaction.Change 的成员变量都有哪些：

```
private final Configuration mConfiguration = new Configuration();
        private boolean mFocusable = true;
        private boolean mHidden = false;
        private boolean mIgnoreOrientationRequest = false;
        ......
        private Rect mPinnedBounds = null;
        private SurfaceControl.Transaction mBoundsChangeTransaction = null;
        private Rect mBoundsChangeSurfaceBounds = null;
        private int mActivityWindowingMode = -1;
        private int mWindowingMode = -1;
```

重点看一下成员变量 mConfiguration，毕竟 Configuration 中包含了很多的配置属性，但是 WindowContainerTransaction 并不支持对 Configuration 中所有的属性进行修改，主要是 screenSize、windowingMode 和 bounds 等。

目前来看设计 WindowContainerTransaction 是为多窗口功能服务的，因此 WindowContainerTransaction.Change 提供的这些方法已经满足需要了。之前都是 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 直接跨 Binder 调用系统服务的一些 resizeStack 之类的方法去修改分屏 Task 的 bounds。有了 WindowContainerTransaction.Chang 之后，就可以把系统服务向客户端提供的对 WindowContainer 的所有修改方法统一组织起来，方便管理以及后续扩展新内容。

### 4 WindowContainerTransaction#reparent

我们上面只分析了 LegacySplitScreenController#splitPrimaryTask 中的一个方法 WindowContainerTransaction#setWindowingMode，还有一个方法 WindowContainerTransaction#reparent 没有分析。

```
/**
     * Reparents a container into another one. The effect of a {@code null} parent can vary. For
     * example, reparenting a stack to {@code null} will reparent it to its display.
     *
     * @param onTop When {@code true}, the child goes to the top of parent; otherwise it goes to
     *              the bottom.
     */
    @NonNull
    public WindowContainerTransaction reparent(@NonNull WindowContainerToken child,
            @Nullable WindowContainerToken parent, boolean onTop) {
        mHierarchyOps.add(HierarchyOp.createForReparent(child.asBinder(),
                parent == null ? null : parent.asBinder(),
                onTop));
        return this;
    }
```

该方法的作用是将一个 WindowContainer 重新 reparent 到另外一个 WindowContainer，最终结果受到参数 parent 的影响，如果 parent 为 null，那么将参数 child 子 WindowContainer 容器 reparent 到它对应的 display 上。

看下 createForReparent 方法：

```
public static HierarchyOp createForReparent(
                @NonNull IBinder container, @Nullable IBinder reparent, boolean toTop) {
            return new HierarchyOp(HIERARCHY_OP_TYPE_REPARENT,
                    container, reparent, null, null, toTop, null);
        }
```

再看下 mHierarchyOps：

```
// Flat list because re-order operations are order-dependent
    private final ArrayList<HierarchyOp> mHierarchyOps = new ArrayList<>();
```

因此这个方法的内容是：

1)、根据传入的 IBinder 类型 container 和 reparent 创建一个 HierarchyOp 对象。

2)、将这个 HierarchyOp 对象添加到 WindowContainerTransaction 的成员变量 mHierarchyOps 中。

后续这个 WindowContainerTransaction 发送到系统服务端的时候，由 WindowOrganizerController 负责将以上两个 IBinder 类型的 token 转化为 WindowContainer 对象，然后进行 reparent 操作。

对比一下 WindowContainerTransaction.Change 的相关内容：

1)、WindowContainerTransaction 创建的 Change 对象是和 WindowContainerToken 内部的 token 对象一一对应的，创建的时候会判断 WindowContainerTransaction 的 mChanges 中是否已经有了一个与 token 对应的 Change 对象，对应的是

2)、HierarchyOp 对象在每次需要进行 reparent 的时候就会创建一次，对应的是一次层次结构上的操作。

### 5 WindowContainerTransaction.HierarchyOp 类

在做出进一步的总结前，还是需要看下 WindowContainerTransaction.HierarchyOp 类的作用。

```
private HierarchyOp(int type, @Nullable IBinder container, @Nullable IBinder reparent,
                int[] windowingModes, int[] activityTypes, boolean toTop,
                @Nullable Bundle launchOptions) {
            mType = type;
            mContainer = container;
            mReparent = reparent;
            mWindowingModes = windowingModes != null ?
                    Arrays.copyOf(windowingModes, windowingModes.length) : null;
            mActivityTypes = activityTypes != null ?
                    Arrays.copyOf(activityTypes, activityTypes.length) : null;
            mToTop = toTop;
            mLaunchOptions = launchOptions;
        }
```

创建 HierarchyOp 对象的时候会对其成员变量进行赋值，比较重要的是其中三个成员变量：

```
// Container we are performing the operation on.
        private final IBinder mContainer;
        // If this is same as mContainer, then only change position, don't reparent.
        private final IBinder mReparent;
        // Moves/reparents to top of parent when {@code true}, otherwise moves/reparents to bottom.
        private final boolean mToTop;
```

mContainer 是子 WindowContainer 对应的 token，mReparent 是父 WindowContainer 对应的 token，mToTop 表示 reparent 操作后是否需要将子 WindowContainer 移动到父 WindowContainer 的 top。

这样来看，WindowContainerTransaction.HierarchyOp 的作用逻辑和 WindowContainerTransaction.Change 类似。WindowContainerTransaction.HierarchyOp 将 App 端设置的一次 WindowContainer 层级调整操作中的子 WindowContainer 对应的 token 和父 WindowContainer 对应的 token 保存起来，后续 WindowContainerTransaction 发送到服务端后，服务端读取 HierarchyOp 中 token 并转化为对应的 WindowContainer 对象，再完成此次 WindowContainer 层级调整。

根据 mReparent 的值，会有几种不同的情况：

1)、mReparent 不为空，且不等于 mContainer，这个是最普遍的 reparent 的操作，将子 WindowContainer 移入父 WindowContainer 中。

2)、mReparent 不为空，且等于 mContainer，那么此次操作不是 reparent，而是 reorder，根据 mToTop 的值调整 mContainer 在当前父容器中的位置。

3)、mReparent 为空，那么将 mContainer 移入当前 display 中。

具体的代码情况在 WCT 的应用一文中详细分析。

三、总结
----

WindowContainerTransaction 向客户端提供了远程修改系统服务中的 WindowContainer 的能力，类似于 Transaction 修改 SurfaceControl，直白点说就是，由于客户端无法直接修改 WindowContainer，所以客户端需要先告诉 WindowContainerTransaction 我想修改哪些 WindowContainer 的哪些内容，WindowContainerTransaction 把这些期望的修改保存起来，后续客户端把这个 WindowContainerTransaction 发送到服务端，服务端读取 WindowContainerTransaction 中的设置后由服务端帮助客户端完成修改。

一般流程是：

1)、客户端创建一个 WindowContainerTransaction 对象。

2)、调用 WindowContainerTransaction 的相关方法，这一步需要将期望修改的 WindowContainer 对应的 WindowContaienrToken 对象作为参数传入。

3)、通过 WindowOrganizer 将 WindowContainerTransaction 发送到服务端，最终服务端读取 WindowContainerTransaction 中保存的参数完成相应操作。

WindowContainerTransaction 支持以下两类修改：

1)、修改 WindowContainer 的属性，包括 WindowingMode 和 Configuration 之类，这类修改保存在 WindowContainerTransaction.Change 类中。

2)、修改 WindowContainer 的层级，既可以将一个子容器从当前父容器移入另外一个新的父容器中，也可以仅仅调整子容器在当前父容器中的位置，这类修改保存在 WindowContainerTransaction.HierarchyOp 类中。

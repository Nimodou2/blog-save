> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/126747833?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/3ccb2d76e9cd4112bd30d854ed2a9ab0.jpeg#pic_center)

一、由 WindowOrganizerController 接收 WCT
------------------------------------

在 WindowContainerTransaction 的发送一文中，我们分析了 WindowContainerTransaction 在客户端中有两种发送方式，都是通过 WindowOrganizer 的相关方法，最终跨进程将 WindowContainerTransaction 发送给了服务端的 WindowOrganizerController，本文来分析一下 WindowOrganizerController 是如何应用这些从客户端传过来的 WindowContainerTransaction 的。

WindowOrganizer 的两种发送方式，WindowOrganizer#applyTransaction 和 WindowOrganizer#applySyncTransaction，分别对应 WindowOrganizerController#applyTransaction 和 WindowOrganizerController#applySyncTransaction：

```
@Override
    public void applyTransaction(WindowContainerTransaction t) {
        enforceTaskPermission("applyTransaction()");
        if (t == null) {
            throw new IllegalArgumentException("Null transaction passed to applySyncTransaction");
        }
        final long ident = Binder.clearCallingIdentity();
        try {
            synchronized (mGlobalLock) {
                applyTransaction(t, -1 /*syncId*/, null /*transition*/);
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }

    @Override
    public int applySyncTransaction(WindowContainerTransaction t,
            IWindowContainerTransactionCallback callback) {
        enforceTaskPermission("applySyncTransaction()");
        if (t == null) {
            throw new IllegalArgumentException("Null transaction passed to applySyncTransaction");
        }
        final long ident = Binder.clearCallingIdentity();
        try {
            synchronized (mGlobalLock) {
                /**
                 * If callback is non-null we are looking to synchronize this transaction by
                 * collecting all the results in to a SurfaceFlinger transaction and then delivering
                 * that to the given transaction ready callback. See {@link BLASTSyncEngine} for the
                 * details of the operation. But at a high level we create a sync operation with a
                 * given ID and an associated callback. Then we notify each WindowContainer in this
                 * WindowContainer transaction that it is participating in a sync operation with
                 * that ID. Once everything is notified we tell the BLASTSyncEngine "setSyncReady"
                 * which means that we have added everything to the set. At any point after this,
                 * all the WindowContainers will eventually finish applying their changes and notify
                 * the BLASTSyncEngine which will deliver the Transaction to the callback.
                 */
                int syncId = -1;
                if (callback != null) {
                    syncId = startSyncWithOrganizer(callback);
                }
                applyTransaction(t, syncId, null /*transition*/);
                if (syncId >= 0) {
                    setSyncReady(syncId);
                }
                return syncId;
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```

WindowOrganizerController#applyTransaction 代表了异步应用 WindowContainerTransaction，WindowOrganizerController#applySyncTransaction 代表了同步应用 WindowContainerTransaction。

WindowOrganizerController#applyTransaction 的内容很简单，直接调用了同名的 private 方法。

WindowOrganizerController#applySyncTransaction，除了调用同名 private 方法 applyTransaction 之外，还有一些和同步相关的处理，但是这些操作与我们要分析的 WindowContainerTransaction 的应用无关，这部分会在其他笔记中继续探究，本篇就只分析和 WindowContainerTransaction 应用相关的部分，即专注 WindowOrganizerController#applyTransaction 方法的内容。

二、WindowContainerTransaction 的应用
--------------------------------

```
/**
     * @param syncId If non-null, this will be a sync-transaction.
     * @param transition A transition to collect changes into.
     */
    private void applyTransaction(@NonNull WindowContainerTransaction t, int syncId,
            @Nullable Transition transition) {
        int effects = 0;
        ProtoLog.v(WM_DEBUG_WINDOW_ORGANIZER, "Apply window transaction, syncId=%d", syncId);
        mService.deferWindowLayout();
        try {
            if (transition != null) {
                // First check if we have a display rotation transition and if so, update it.
                final DisplayContent dc = DisplayRotation.getDisplayFromTransition(transition);
                if (dc != null && transition.mChanges.get(dc).mRotation != dc.getRotation()) {
                    // Go through all tasks and collect them before the rotation
                    // TODO(shell-transitions): move collect() to onConfigurationChange once
                    //       wallpaper handling is synchronized.
                    dc.forAllTasks(task -> {
                        if (task.isVisible()) transition.collect(task);
                    });
                    dc.getInsetsStateController().addProvidersToTransition();
                    dc.sendNewConfiguration();
                    effects |= TRANSACT_EFFECTS_LIFECYCLE;
                }
            }
            ArraySet<WindowContainer> haveConfigChanges = new ArraySet<>();
            Iterator<Map.Entry<IBinder, WindowContainerTransaction.Change>> entries =
                    t.getChanges().entrySet().iterator();
            while (entries.hasNext()) {
                final Map.Entry<IBinder, WindowContainerTransaction.Change> entry = entries.next();
                final WindowContainer wc = WindowContainer.fromBinder(entry.getKey());
                if (wc == null || !wc.isAttached()) {
                    Slog.e(TAG, "Attempt to operate on detached container: " + wc);
                    continue;
                }
                // Make sure we add to the syncSet before performing
                // operations so we don't end up splitting effects between the WM
                // pending transaction and the BLASTSync transaction.
                if (syncId >= 0) {
                    addToSyncSet(syncId, wc);
                }
                if (transition != null) transition.collect(wc);

                int containerEffect = applyWindowContainerChange(wc, entry.getValue());
                effects |= containerEffect;

                // Lifecycle changes will trigger ensureConfig for everything.
                if ((effects & TRANSACT_EFFECTS_LIFECYCLE) == 0
                        && (containerEffect & TRANSACT_EFFECTS_CLIENT_CONFIG) != 0) {
                    haveConfigChanges.add(wc);
                }
            }
            // Hierarchy changes
            final List<WindowContainerTransaction.HierarchyOp> hops = t.getHierarchyOps();
            final int hopSize = hops.size();
            if (hopSize > 0) {
                final boolean isInLockTaskMode = mService.isInLockTaskMode();
                for (int i = 0; i < hopSize; ++i) {
                    effects |= applyHierarchyOp(hops.get(i), effects, syncId, transition,
                            isInLockTaskMode);
                }
            }
            // Queue-up bounds-change transactions for tasks which are now organized. Do
            // this after hierarchy ops so we have the final organized state.
            entries = t.getChanges().entrySet().iterator();
            while (entries.hasNext()) {
                final Map.Entry<IBinder, WindowContainerTransaction.Change> entry = entries.next();
                final WindowContainer wc = WindowContainer.fromBinder(entry.getKey());
                if (wc == null || !wc.isAttached()) {
                    Slog.e(TAG, "Attempt to operate on detached container: " + wc);
                    continue;
                }
                final Task task = wc.asTask();
                final Rect surfaceBounds = entry.getValue().getBoundsChangeSurfaceBounds();
                if (task == null || !task.isAttached() || surfaceBounds == null) {
                    continue;
                }
                if (!task.isOrganized()) {
                    final Task parent = task.getParent() != null ? task.getParent().asTask() : null;
                    // Also allow direct children of created-by-organizer tasks to be
                    // controlled. In the future, these will become organized anyways.
                    if (parent == null || !parent.mCreatedByOrganizer) {
                        throw new IllegalArgumentException(
                                "Can't manipulate non-organized task surface " + task);
                    }
                }
                final SurfaceControl.Transaction sft = new SurfaceControl.Transaction();
                final SurfaceControl sc = task.getSurfaceControl();
                sft.setPosition(sc, surfaceBounds.left, surfaceBounds.top);
                if (surfaceBounds.isEmpty()) {
                    sft.setWindowCrop(sc, null);
                } else {
                    sft.setWindowCrop(sc, surfaceBounds.width(), surfaceBounds.height());
                }
                task.setMainWindowSizeChangeTransaction(sft);
            }
            if ((effects & TRANSACT_EFFECTS_LIFECYCLE) != 0) {
                // Already calls ensureActivityConfig
                mService.mRootWindowContainer.ensureActivitiesVisible(null, 0, PRESERVE_WINDOWS);
                mService.mRootWindowContainer.resumeFocusedTasksTopActivities();
            } else if ((effects & TRANSACT_EFFECTS_CLIENT_CONFIG) != 0) {
                final PooledConsumer f = PooledLambda.obtainConsumer(
                        ActivityRecord::ensureActivityConfiguration,
                        PooledLambda.__(ActivityRecord.class), 0,
                        true /* preserveWindow */);
                try {
                    for (int i = haveConfigChanges.size() - 1; i >= 0; --i) {
                        haveConfigChanges.valueAt(i).forAllActivities(f);
                    }
                } finally {
                    f.recycle();
                }
            }

            if ((effects & TRANSACT_EFFECTS_CLIENT_CONFIG) == 0) {
                mService.addWindowLayoutReasons(LAYOUT_REASON_CONFIG_CHANGED);
            }
        } finally {
            mService.continueWindowLayout();
        }
    }
```

这个方法可以很明显被分成 5 部分：

### 1 Transition 相关

```
if (transition != null) {
        // First check if we have a display rotation transition and if so, update it.
        final DisplayContent dc = DisplayRotation.getDisplayFromTransition(transition);
        if (dc != null && transition.mChanges.get(dc).mRotation != dc.getRotation()) {
            // Go through all tasks and collect them before the rotation
            // TODO(shell-transitions): move collect() to onConfigurationChange once
            //       wallpaper handling is synchronized.
            dc.forAllTasks(task -> {
                if (task.isVisible()) transition.collect(task);
            });
            dc.getInsetsStateController().addProvidersToTransition();
            dc.sendNewConfiguration();
            effects |= TRANSACT_EFFECTS_LIFECYCLE;
        }
    }
```

Transition 是 Android 12 新加的内容，没有接触过，而且从 WindowOrganizerController#applyTransaction 和 WindowOrganizerController#applySyncTransaction 传过来的 transition 参数也是 null，因此暂不分析。

### 2 Change 相关

```
ArraySet<WindowContainer> haveConfigChanges = new ArraySet<>();
            Iterator<Map.Entry<IBinder, WindowContainerTransaction.Change>> entries =
                    t.getChanges().entrySet().iterator();
            while (entries.hasNext()) {
                final Map.Entry<IBinder, WindowContainerTransaction.Change> entry = entries.next();
                final WindowContainer wc = WindowContainer.fromBinder(entry.getKey());
                if (wc == null || !wc.isAttached()) {
                    Slog.e(TAG, "Attempt to operate on detached container: " + wc);
                    continue;
                }
                // Make sure we add to the syncSet before performing
                // operations so we don't end up splitting effects between the WM
                // pending transaction and the BLASTSync transaction.
                if (syncId >= 0) {
                    addToSyncSet(syncId, wc);
                }
                if (transition != null) transition.collect(wc);

                int containerEffect = applyWindowContainerChange(wc, entry.getValue());
                effects |= containerEffect;

                // Lifecycle changes will trigger ensureConfig for everything.
                if ((effects & TRANSACT_EFFECTS_LIFECYCLE) == 0
                        && (containerEffect & TRANSACT_EFFECTS_CLIENT_CONFIG) != 0) {
                    haveConfigChanges.add(wc);
                }
            }
```

根据我们之前对 WindowContainerTransaction 的相关了解，知道 WindowContainerTransaction 的内部类 Change 是用来处理 WindowContainer 的配置属性方面的更改。

这部分的主要工作是，遍历传入的 WindowContainerTransaction 的 mChanges：

1)、将 mChanges 中的储存的 token 转化为 WindowContainer。

2)、调用 WindowOrganizerController#addToSyncSet 将当前 WindowContainer 添加到同步集合中，这一处理属于 WCT 同步机制相关的部分，跳过。

3)、调用 WIndowOrganizerController#applyWindowContainerChange 去应用 Change 里的设置，属于重点分析内容。

4)、如果本次修改没有影响到 TRANSACT_EFFECTS_LIFECYCLE 但是对 TRANSACT_EFFECTS_CLIENT_CONFIG 产生了影响，那么将当前 WindowContainer 添加到 haveConfigChanges 队列中，该队列保存了所有 Configuration 发生改变的 WindowContainer。

重点分析 WIndowOrganizerController#applyWindowContainerChange。

```
private int applyWindowContainerChange(WindowContainer wc,
            WindowContainerTransaction.Change c) {
        sanitizeWindowContainer(wc);

        int effects = applyChanges(wc, c);

        if (wc instanceof DisplayArea) {
            effects |= applyDisplayAreaChanges(wc.asDisplayArea(), c);
        } else if (wc instanceof Task) {
            effects |= applyTaskChanges(wc.asTask(), c);
        }

        return effects;
    }
```

这里根据当前 WindowContainer 是 Task 还是 DisplayArea 有不同的处理方式，但是在此之前都走了 WindowOrganizerController#applyChanges。

#### 2.1 WindowOrganizerController#sanitizeWindowContainer

```
private void sanitizeWindowContainer(WindowContainer wc) {
        if (!(wc instanceof Task) && !(wc instanceof DisplayArea)) {
            throw new RuntimeException("Invalid token in task or displayArea transaction");
        }
    }
```

如果当前 WindowContainer 不是 Task 或者 DisplayArea 类型的，则抛出异常。

说明当前 WindowContainerTransaction 支持对 Task 和 DisplayArea 进行修改，如果对 ActivityRecord，WindowState 之类的修改则会报错（其实也无法对 WindowContainer 的其他类型的子类进行修改，因为只有 Task 和 DisplayArea 会在构造方法中初始化 RemoteToken 类型的成员变量 mRemoteToken）。

#### 2.2 WindowOrganizerController#applyChanges

```
private int applyChanges(WindowContainer container, WindowContainerTransaction.Change change) {
        // The "client"-facing API should prevent bad changes; however, just in case, sanitize
        // masks here.
        final int configMask = change.getConfigSetMask() & CONTROLLABLE_CONFIGS;
        final int windowMask = change.getWindowSetMask() & CONTROLLABLE_WINDOW_CONFIGS;
        int effects = 0;
        final int windowingMode = change.getWindowingMode();
        if (configMask != 0) {
            if (windowingMode > -1 && windowingMode != container.getWindowingMode()) {
                // Special handling for when we are setting a windowingMode in the same transaction.
                // Setting the windowingMode is going to call onConfigurationChanged so we don't
                // need it called right now. Additionally, some logic requires everything in the
                // configuration to change at the same time (ie. surface-freezer requires bounds
                // and mode to change at the same time).
                final Configuration c = container.getRequestedOverrideConfiguration();
                c.setTo(change.getConfiguration(), configMask, windowMask);
            } else {
                final Configuration c =
                        new Configuration(container.getRequestedOverrideConfiguration());
                c.setTo(change.getConfiguration(), configMask, windowMask);
                container.onRequestedOverrideConfigurationChanged(c);
            }
            effects |= TRANSACT_EFFECTS_CLIENT_CONFIG;
        }
        if ((change.getChangeMask() & WindowContainerTransaction.Change.CHANGE_FOCUSABLE) != 0) {
            if (container.setFocusable(change.getFocusable())) {
                effects |= TRANSACT_EFFECTS_LIFECYCLE;
            }
        }

        if (windowingMode > -1) {
            if (mService.isInLockTaskMode()
                    && WindowConfiguration.inMultiWindowMode(windowingMode)) {
                throw new UnsupportedOperationException("Not supported to set multi-window"
                        + " windowing mode during locked task mode.");
            }
            container.setWindowingMode(windowingMode);
        }
        return effects;
    }
```

##### 2.2.1 判断 Configuration 是否改变

```
final int configMask = change.getConfigSetMask() & CONTROLLABLE_CONFIGS;
```

CONTROLLABLE_CONFIGS 的定义是：

```
static final int CONTROLLABLE_CONFIGS = ActivityInfo.CONFIG_WINDOW_CONFIGURATION
            | ActivityInfo.CONFIG_SMALLEST_SCREEN_SIZE | ActivityInfo.CONFIG_SCREEN_SIZE;
```

包括了四个方面的修改：

1 WindowContainerTransaction#[setBounds](https://so.csdn.net/so/search?q=setBounds&spm=1001.2101.3001.7020)

2 WindowContainerTransaction#setAppBounds

3 WindowContainerTransaction#setSmallestScreenWidthDp

4 WindowContainerTransaction#setScreenSizeDp

如果当前 WindowContainerTransaction 有这几方面的修改，那么进入到 2.2.2。

##### 2.2.2 应用 Configuration

```
final int windowingMode = change.getWindowingMode();
        if (configMask != 0) {
            if (windowingMode > -1 && windowingMode != container.getWindowingMode()) {
                // Special handling for when we are setting a windowingMode in the same transaction.
                // Setting the windowingMode is going to call onConfigurationChanged so we don't
                // need it called right now. Additionally, some logic requires everything in the
                // configuration to change at the same time (ie. surface-freezer requires bounds
                // and mode to change at the same time).
                final Configuration c = container.getRequestedOverrideConfiguration();
                c.setTo(change.getConfiguration(), configMask, windowMask);
            } else {
                final Configuration c =
                        new Configuration(container.getRequestedOverrideConfiguration());
                c.setTo(change.getConfiguration(), configMask, windowMask);
                container.onRequestedOverrideConfigurationChanged(c);
            }
            effects |= TRANSACT_EFFECTS_CLIENT_CONFIG;
        }
```

走到这里说明针对当前 WindowContainer 的 Configuration 有了一些特殊的设置，一般的做法是先获取到当前 WindowContainer 的 mRequestedOverrideConfiguration 一份拷贝，然后让这份拷贝应用这些特殊设置，最后调用 onRequestedOverrideConfigurationChanged 完成修改，即 1.2.2 的做法，但是这里根据从 WindowContainerTransaction 中取到的 WindowingMode 又分了两种情况：

1)、如果从 Change 中拿出的 windowingMode 无效，那么先获取到当前 WindowContainer 的 mRequestedOverrideConfiguration 的一份拷贝，然后再将 Change 中的部分赋值给这个拷贝，最后调用 ConfigurationContainer#onRequestedOverrideConfigurationChanged 完成 Configuration 的修改。

2)、如果这里 WindowContainerTransaction 中设置了一个 WindowingMode，该 WindowingMode 大于 1（表示有效），且不同于当前 WindowContainer 的 WindowingMode，那么就表示此次应用会改变当前 WindowContainer 的 WindowingMode。需要知道的是，上一步中的 ConfigurationContainer#onRequestedOverrideConfigurationChanged 和 2.2.4 中的 ConfigurationContainer#setWindowingMode 都会触发 ConfigurationContainer#onConfigurationChanged，但是因为一些逻辑要求关于 configuration 的修改需一次完成，同时为了避免触发多次 ConfigurationContainer#onConfigurationChanged，这里我们直接拿到当前 WindowContainer 的 mRequestedOverrideConfiguration 的引用，然后将从 Change 中提取的 Configuration 覆盖掉当前 WindowContainer 的 mRequestedOverrideConfiguration，这就表示目前我们只想修改 WindowContainer 的 mRequestedOverrideConfiguration，但是不触发 ConfigurationContainer#onConfigurationChanged（因为这一步在 2.2.4 中肯定会得到执行）。

##### 2.2.3 应用 focusable 属性

```
if ((change.getChangeMask() & WindowContainerTransaction.Change.CHANGE_FOCUSABLE) != 0) {
            if (container.setFocusable(change.getFocusable())) {
                effects |= TRANSACT_EFFECTS_LIFECYCLE;
            }
        }
```

用 WIndowContainer#setFocusable 设置当前 WindowContainer 和其子 WindowContainer 的 focusable 状态。

##### 2.2.4 应用 windowingMode

```
if (windowingMode > -1) {
            if (mService.isInLockTaskMode()
                    && WindowConfiguration.inMultiWindowMode(windowingMode)) {
                throw new UnsupportedOperationException("Not supported to set multi-window"
                        + " windowing mode during locked task mode.");
            }
            container.setWindowingMode(windowingMode);
        }
```

如果从 WindowConfigurationTransaction 中取出的 WindowingMode 大于 - 1，表示此次 WindowConfigurationTransaction 希望修改当前 WindowContainer 的 WindowingMode。直接调用 ConfigurationContainer#setWindowingMode，这一步会触发 ConfigurationContainer#onRequestedOverrideConfigurationChanged，最终调用 ConfigurationContainer#onConfigurationChanged 将该 windowingMode 应用到该 Task 下所有的 WindowContainer。

#### 2.3 WindowOrganizerController#applyTaskChanges

分屏模式下，我们处理的 WindowContainer 都是 Task 类型的，所以走 applyTaskChanges。

```
private int applyTaskChanges(Task tr, WindowContainerTransaction.Change c) {
        int effects = 0;
        final SurfaceControl.Transaction t = c.getBoundsChangeTransaction();

        if ((c.getChangeMask() & WindowContainerTransaction.Change.CHANGE_HIDDEN) != 0) {
            if (tr.setForceHidden(FLAG_FORCE_HIDDEN_FOR_TASK_ORG, c.getHidden())) {
                effects = TRANSACT_EFFECTS_LIFECYCLE;
            }
        }

        final int childWindowingMode = c.getActivityWindowingMode();
        if (childWindowingMode > -1) {
            tr.setActivityWindowingMode(childWindowingMode);
        }

        if (t != null) {
            tr.setMainWindowSizeChangeTransaction(t);
        }

        Rect enterPipBounds = c.getEnterPipBounds();
        if (enterPipBounds != null) {
            tr.mDisplayContent.mPinnedTaskController.setEnterPipBounds(enterPipBounds);
        }

        return effects;
    }
```

1)、Task#setForceHidden，设置 Task 的 forced-hidden 状态 flag，没怎么接触过，设置了这个 Flag 会强行标记该 Task 为不可见。

2)、设置 Task 的下所有子 ActivityRecord 的 windowingMode。

3)、WindowContainerTransaction#setBoundsChangeTransaction 相关，目前接触到的只有 WindowManagerProxy 的两处，不是很了解，但是这个很容易影响到初次进入分屏时候的 [Launcher](https://so.csdn.net/so/search?q=Launcher&spm=1001.2101.3001.7020) App 的 bounds。

4)、设置 PIP 的 bounds。

### 3 HierarchyOp 相关

```
// Hierarchy changes
            final List<WindowContainerTransaction.HierarchyOp> hops = t.getHierarchyOps();
            final int hopSize = hops.size();
            if (hopSize > 0) {
                final boolean isInLockTaskMode = mService.isInLockTaskMode();
                for (int i = 0; i < hopSize; ++i) {
                    effects |= applyHierarchyOp(hops.get(i), effects, syncId, transition,
                            isInLockTaskMode);
                }
            }
```

根据我们之前的了解，HierarchyOp 主要是用来处理层级结构中的 reparent 和 reorder 相关的操作，每一个 HierarchyOp 对象都包含了一次层级结构调整。

这里遍历 WindowContainerTransaction 的 mHierarchyOps 成员变量，对其中保存的每一个 HierarchyOp 对象，都调用了 WindowOrganizerController#applyHierarchyOp 进行处理：

```
private int applyHierarchyOp(WindowContainerTransaction.HierarchyOp hop, int effects,
            int syncId, @Nullable Transition transition, boolean isInLockTaskMode) {
        final int type = hop.getType();
        switch (type) {
            case HIERARCHY_OP_TYPE_SET_LAUNCH_ROOT: {
                final WindowContainer wc = WindowContainer.fromBinder(hop.getContainer());
                final Task task = wc != null ? wc.asTask() : null;
                if (task != null) {
                    task.getDisplayArea().setLaunchRootTask(task,
                            hop.getWindowingModes(), hop.getActivityTypes());
                } else {
                    throw new IllegalArgumentException("Cannot set non-task as launch root: " + wc);
                }
                break;
            }
            case HIERARCHY_OP_TYPE_SET_LAUNCH_ADJACENT_FLAG_ROOT: {
                final WindowContainer wc = WindowContainer.fromBinder(hop.getContainer());
                final Task task = wc != null ? wc.asTask() : null;
                if (task == null) {
                    throw new IllegalArgumentException("Cannot set non-task as launch root: " + wc);
                } else if (!task.mCreatedByOrganizer) {
                    throw new UnsupportedOperationException(
                            "Cannot set non-organized task as adjacent flag root: " + wc);
                } else if (task.mAdjacentTask == null) {
                    throw new UnsupportedOperationException(
                            "Cannot set non-adjacent task as adjacent flag root: " + wc);
                }

                final boolean clearRoot = hop.getToTop();
                task.getDisplayArea().setLaunchAdjacentFlagRootTask(clearRoot ? null : task);
                break;
            }
            case HIERARCHY_OP_TYPE_SET_ADJACENT_ROOTS:
                effects |= setAdjacentRootsHierarchyOp(hop);
                break;
        }
        // The following operations may change task order so they are skipped while in lock task
        // mode. The above operations are still allowed because they don't move tasks. And it may
        // be necessary such as clearing launch root after entering lock task mode.
        if (isInLockTaskMode) {
            Slog.w(TAG, "Skip applying hierarchy operation " + hop + " while in lock task mode");
            return effects;
        }

        switch (type) {
            case HIERARCHY_OP_TYPE_CHILDREN_TASKS_REPARENT:
                effects |= reparentChildrenTasksHierarchyOp(hop, transition, syncId);
                break;
            case HIERARCHY_OP_TYPE_REORDER:
            case HIERARCHY_OP_TYPE_REPARENT:
                final WindowContainer wc = WindowContainer.fromBinder(hop.getContainer());
                if (wc == null || !wc.isAttached()) {
                    Slog.e(TAG, "Attempt to operate on detached container: " + wc);
                    break;
                }
                if (syncId >= 0) {
                    addToSyncSet(syncId, wc);
                }
                if (transition != null) {
                    transition.collect(wc);
                    if (hop.isReparent()) {
                        if (wc.getParent() != null) {
                            // Collect the current parent. It's visibility may change as
                            // a result of this reparenting.
                            transition.collect(wc.getParent());
                        }
                        if (hop.getNewParent() != null) {
                            final WindowContainer parentWc =
                                    WindowContainer.fromBinder(hop.getNewParent());
                            if (parentWc == null) {
                                Slog.e(TAG, "Can't resolve parent window from token");
                                break;
                            }
                            transition.collect(parentWc);
                        }
                    }
                }
                effects |= sanitizeAndApplyHierarchyOp(wc, hop);
                break;
            case HIERARCHY_OP_TYPE_LAUNCH_TASK:
                final Bundle launchOpts = hop.getLaunchOptions();
                final int taskId = launchOpts.getInt(
                        WindowContainerTransaction.HierarchyOp.LAUNCH_KEY_TASK_ID);
                launchOpts.remove(WindowContainerTransaction.HierarchyOp.LAUNCH_KEY_TASK_ID);
                mService.startActivityFromRecents(taskId, launchOpts);
                break;
        }
        return effects;
    }
```

Android 11 的时候只有 reparent 和 reorder 操作，现在支持了 7 种操作：

```
public static final int HIERARCHY_OP_TYPE_REPARENT = 0;
    public static final int HIERARCHY_OP_TYPE_REORDER = 1;
    public static final int HIERARCHY_OP_TYPE_CHILDREN_TASKS_REPARENT = 2;
    public static final int HIERARCHY_OP_TYPE_SET_LAUNCH_ROOT = 3;
    public static final int HIERARCHY_OP_TYPE_SET_ADJACENT_ROOTS = 4;
    public static final int HIERARCHY_OP_TYPE_LAUNCH_TASK = 5;
    public static final int HIERARCHY_OP_TYPE_SET_LAUNCH_ADJACENT_FLAG_ROOT = 6;
```

其中 HIERARCHY_OP_TYPE_SET_LAUNCH_ROOT、HIERARCHY_OP_TYPE_SET_ADJACENT_ROOTS 和 HIERARCHY_OP_TYPE_SET_LAUNCH_ADJACENT_FLAG_ROOT 这三种操作由于没有改变 Task 的顺序，因此支持在 lock task mode 下设置，其余四种则不行。

#### 3.1 HIERARCHY_OP_TYPE_SET_LAUNCH_ROOT

```
case HIERARCHY_OP_TYPE_SET_LAUNCH_ROOT: {
                final WindowContainer wc = WindowContainer.fromBinder(hop.getContainer());
                final Task task = wc != null ? wc.asTask() : null;
                if (task != null) {
                    task.getDisplayArea().setLaunchRootTask(task,
                            hop.getWindowingModes(), hop.getActivityTypes());
                } else {
                    throw new IllegalArgumentException("Cannot set non-task as launch root: " + wc);
                }
                break;
            }
```

将传入的 Task 作为指定的的 WindowingMode 类型和 ActivityType 类型的 launchRootTask，这赋予了客户端设置 launchRootTask 的能力。

launchRootTask，顾名思义，指的是 App 启动后为该 App 创建的 Task 应该归属于哪个 rootTask，分屏流程中指定的 launchRootTask，由 TaskOrganizerController 创建，父节点为 TaskDisplayArea。直接看一处具体的调用，在进入分屏的过程中调用 WindowManagerProxy#applyEnterSplit：

```
wct.setLaunchRoot(tiles.mSecondary.token, CONTROLLED_WINDOWING_MODES,
            CONTROLLED_ACTIVITY_TYPES);
```

这里的 tiles.mSecondary.token 即是分屏的 secondaryRootTask，CONTROLLED_WINDOWING_MODES 的定义是：

```
private static final int[] CONTROLLED_WINDOWING_MODES = {
        WINDOWING_MODE_FULLSCREEN,
        WINDOWING_MODE_SPLIT_SCREEN_SECONDARY,
        WINDOWING_MODE_UNDEFINED
};
```

CONTROLLED_ACTIVITY_TYPES 的定义是

```
private static final int[] CONTROLLED_ACTIVITY_TYPES = {
            ACTIVITY_TYPE_STANDARD,
            ACTIVITY_TYPE_HOME,
            ACTIVITY_TYPE_RECENTS,
            ACTIVITY_TYPE_UNDEFINED
    };
```

那么很明显，这里的意思是，后续只要是以上 WindowingMode 类型和 ActivityType 类型的 Task，全部都以分屏的 secondaryRootTask 为 rootTask。这也很好理解，我们希望分屏模式下，启动的新 Task 仍然处于分屏模式，而不是全屏显示，或者启动到上半屏，以 primaryRootTask 为 rootTask。  
关于 launchRootTask 的使用，需要另开一篇笔记分析。

#### 3.2 HIERARCHY_OP_TYPE_SET_LAUNCH_ADJACENT_FLAG_ROOT

```
case HIERARCHY_OP_TYPE_SET_LAUNCH_ADJACENT_FLAG_ROOT: {
                final WindowContainer wc = WindowContainer.fromBinder(hop.getContainer());
                final Task task = wc != null ? wc.asTask() : null;
                if (task == null) {
                    throw new IllegalArgumentException("Cannot set non-task as launch root: " + wc);
                } else if (!task.mCreatedByOrganizer) {
                    throw new UnsupportedOperationException(
                            "Cannot set non-organized task as adjacent flag root: " + wc);
                } else if (task.mAdjacentTask == null) {
                    throw new UnsupportedOperationException(
                            "Cannot set non-adjacent task as adjacent flag root: " + wc);
                }

                final boolean clearRoot = hop.getToTop();
                task.getDisplayArea().setLaunchAdjacentFlagRootTask(clearRoot ? null : task);
                break;
            }
```

这种情况和 HIERARCHY_OP_TYPE_SET_LAUNCH_ROOT 类似，上面设置的是指定 WindowingMode 和 ActivityType 的 Task 的 rootTask，这里设置的是启动时伴随有 FLAG_ACTIVITY_LAUNCH_ADJACENT 这个 flag 的 Task 的 rootTask。

```
/**
     * A launch root task for activity launching with {@link FLAG_ACTIVITY_LAUNCH_ADJACENT} flag.
     */
    private Task mLaunchAdjacentFlagRootTask;
```

最终结果是 TaskDisplayArea 的 mLaunchAdjacentFlagRootTask 成员变量赋值为了传入的 Task。

这种情况没怎么遇到过，因此不详细分析了。

#### 3.3 HIERARCHY_OP_TYPE_SET_ADJACENT_ROOTS

```
case HIERARCHY_OP_TYPE_SET_ADJACENT_ROOTS:
                effects |= setAdjacentRootsHierarchyOp(hop);
                break;
```

这里进一步调用了 WindowOrganizerController#setAdjacentRootsHierarchyOp。

```
private int setAdjacentRootsHierarchyOp(WindowContainerTransaction.HierarchyOp hop) {
        final Task root1 = WindowContainer.fromBinder(hop.getContainer()).asTask();
        final Task root2 = WindowContainer.fromBinder(hop.getAdjacentRoot()).asTask();
        if (!root1.mCreatedByOrganizer || !root2.mCreatedByOrganizer) {
            throw new IllegalArgumentException("setAdjacentRootsHierarchyOp: Not created by"
                    + " organizer root1=" + root1 + " root2=" + root2);
        }
        root1.setAdjacentTask(root2);
        return TRANSACT_EFFECTS_LIFECYCLE;
    }
```

这里看到，取出传入的 HierarchyOp 中保存的两个 Task，然后如果这两个 Task 都是 TaskOrganizerController 创建的，那么调用 Task#setAdjacentTask。

```
void setAdjacentTask(Task adjacent) {
        mAdjacentTask = adjacent;
        adjacent.mAdjacentTask = this;
    }
```

Task 的成员变量 mAdjacent 的定义是：

```
Task mAdjacentTask; // Task adjacent to this one.
```

那么这里的操作就是设置两个 Task 互相为相邻 Task。

mAdjacentTask 我没有接触过，看使用的场景，主要是在判断 Task 的可见性和获取 launchRootTask 的时候用到了。

#### 3.4 HIERARCHY_OP_TYPE_CHILDREN_TASKS_REPARENT

```
case HIERARCHY_OP_TYPE_CHILDREN_TASKS_REPARENT:
                effects |= reparentChildrenTasksHierarchyOp(hop, transition, syncId);
                break;
```

这里进一步调用了 WindowOrganizerController#setAdjacentRootsHierarchyOp。

```
private int reparentChildrenTasksHierarchyOp(WindowContainerTransaction.HierarchyOp hop,
            @Nullable Transition transition, int syncId) {
        WindowContainer currentParent = hop.getContainer() != null
                ? WindowContainer.fromBinder(hop.getContainer()) : null;
        WindowContainer newParent = hop.getNewParent() != null
                ? WindowContainer.fromBinder(hop.getNewParent()) : null;
        if (currentParent == null && newParent == null) {
            throw new IllegalArgumentException("reparentChildrenTasksHierarchyOp: " + hop);
        } else if (currentParent == null) {
            currentParent = newParent.asTask().getDisplayContent().getDefaultTaskDisplayArea();
        } else if (newParent == null) {
            newParent = currentParent.asTask().getDisplayContent().getDefaultTaskDisplayArea();
        }

		......

        // We want to collect the tasks first before re-parenting to avoid array shifting on us.
        final ArrayList<Task> tasksToReparent = new ArrayList<>();

        currentParent.forAllTasks((Consumer<Task>) (task) -> {
            Slog.i(TAG, " Processing task=" + task);
            if (task.mCreatedByOrganizer
                    || task.getParent() != finalCurrentParent) {
                // We only care about non-organized task that are direct children of the thing we
                // are reparenting from.
                return;
            }
            if (newParentInMultiWindow && !task.supportsMultiWindowInDisplayArea(newParentTda)) {
                Slog.e(TAG, "reparentChildrenTasksHierarchyOp non-resizeable task to multi window,"
                        + " task=" + task);
                return;
            }
            if (!ArrayUtils.contains(hop.getActivityTypes(), task.getActivityType())) return;
            if (!ArrayUtils.contains(hop.getWindowingModes(), task.getWindowingMode())) return;

            tasksToReparent.add(task);
        }, !hop.getToTop());

        final int count = tasksToReparent.size();
        for (int i = 0; i < count; ++i) {
            final Task task = tasksToReparent.get(i);
            if (syncId >= 0) {
                addToSyncSet(syncId, task);
            }
            if (transition != null) transition.collect(task);

            if (newParent instanceof TaskDisplayArea) {
                // For now, reparenting to display area is different from other reparents...
                task.reparent((TaskDisplayArea) newParent, hop.getToTop());
            } else {
                task.reparent((Task) newParent,
                        hop.getToTop() ? POSITION_TOP : POSITION_BOTTOM,
                        false /*moveParents*/, "processChildrenTaskReparentHierarchyOp");
            }
        }

        if (transition != null) transition.collect(newParent);

        return TRANSACT_EFFECTS_LIFECYCLE;
    }
```

很长一大串，但是逻辑很简单，首先从 currentParent 中筛选出所有可以进行 reparent 操作的 Task，全部添加到一个临时变量 tasksToReparent 中，然后遍历 tasksToReparent，将 tasksToReparent 中所有的 Task 全部 reparent 到 newParent 之下，也是这个方法名称的字面含义。

HIERARCHY_OP_TYPE_CHILDREN_TASKS_REPARENT 的起点是客户端调用 WindowContainerTransaction#reparentTasks，这个方法暂时没有看到分屏有哪个地方有用到。

#### 3.5 HIERARCHY_OP_TYPE_LAUNCH_TASK

```
case HIERARCHY_OP_TYPE_LAUNCH_TASK:
                final Bundle launchOpts = hop.getLaunchOptions();
                final int taskId = launchOpts.getInt(
                        WindowContainerTransaction.HierarchyOp.LAUNCH_KEY_TASK_ID);
                launchOpts.remove(WindowContainerTransaction.HierarchyOp.LAUNCH_KEY_TASK_ID);
                mService.startActivityFromRecents(taskId, launchOpts);
                break;
```

逻辑很简单，基于 WindowContainerTransaction 中保存的 TaskId 和 Bundle 对象，调用 ATMS#startActivityFromRecents 去启动这个 Task，这种启动方式一般是 Launcher 会用到。

#### 3.6 HIERARCHY_OP_TYPE_REORDER 和 HIERARCHY_OP_TYPE_REPARENT

分屏经常用到的，本文的分析重点。

```
case HIERARCHY_OP_TYPE_REORDER:
            case HIERARCHY_OP_TYPE_REPARENT:
                final WindowContainer wc = WindowContainer.fromBinder(hop.getContainer());
                ......
                effects |= sanitizeAndApplyHierarchyOp(wc, hop);
                break;
```

过滤掉了一些非必要信息，直奔重点 WindowOrganizerController#sanitizeAndApplyHierarchyOp。

在继续分析这个方法之前，需要回顾一下最初介绍 WindowContainerTransaction 的时候，提到的其内部类 HierarchyOp，该类用来帮助进行层级调整操作，三个比较重要的成员变量是：

```
// Container we are performing the operation on.
        private final IBinder mContainer;

        // If this is same as mContainer, then only change position, don't reparent.
        private final IBinder mReparent;

        // Moves/reparents to top of parent when {@code true}, otherwise moves/reparents to bottom.
        private final boolean mToTop;
```

mContainer 是当前我们想要进行层级调整的 WindowContainer，mReparent 是为 mContainer 指定的新的父 WindowContainer，mToTop 代表在本次层级调整之后是否需要把 mContainer 移动到 mReparent 的 top。

接下来分析该方法内容：

```
private int sanitizeAndApplyHierarchyOp(WindowContainer container,
            WindowContainerTransaction.HierarchyOp hop) {
        // 调整层级之前先进行必要判断，当前WindowContainer必须能够转化为一个Task对象，且隶属于一个Display中。
        final Task task = container.asTask();
        if (task == null) {
            throw new IllegalArgumentException("Invalid container in hierarchy op");
        }
        final DisplayContent dc = task.getDisplayContent();
        if (dc == null) {
            Slog.w(TAG, "Container is no longer attached: " + task);
            return 0;
        }
        final Task as = task;

        // 调用WindowContainerTransaction.HierarchyOp#isReparent判断此次层级调整是reparent还是reorder。
        if (hop.isReparent()) {
            // 如果此次层级调整是一次reparent操作，那么需要满足这两个条件之一，才能继续往下走，否则抛出异常：
            // 1)、当前Task是一个rootTask。
            // 2)、如果当前Task不是一个rootTask，那么其直接父Task是由TaskOrganzierController创建的。
            final boolean isNonOrganizedRootableTask =
                    task.isRootTask() || task.getParent().asTask().mCreatedByOrganizer;
            if (isNonOrganizedRootableTask) {
                // 继续检测newParent的合法性：
                // 如果从HierarchyOp中取出newParent为空，那么将其赋值为默认的TaskDisplayArea，如果还为空，那么只能中止此次reparent操作。
                WindowContainer newParent = hop.getNewParent() == null
                        ? dc.getDefaultTaskDisplayArea()
                        : WindowContainer.fromBinder(hop.getNewParent());
                if (newParent == null) {
                    Slog.e(TAG, "Can't resolve parent window from token");
                    return 0;
                }
                // 现在我们有了一个不为空的Task和newParent对象。
                // 判断当前Task的父容器是否是newParent。
                if (task.getParent() != newParent) {
                    if (newParent.asTaskDisplayArea() != null) {
                        // 如果newParent是一个TaskDisplayArea实例，那么特殊处理。
                        // For now, reparenting to displayarea is different from other reparents...
                        as.reparent(newParent.asTaskDisplayArea(), hop.getToTop());
                    } else if (newParent.asTask() != null) {
                        // 目前newParent要么是一个TaskDisplayArea实例，要么是一个Task实例，如果两者都不是，那么直接抛出异常。
                        if (newParent.inMultiWindowMode() && task.isLeafTask()) {
                            // 针对多窗口模式，需要额外判断。
                            if (newParent.inPinnedWindowingMode()) {
                                // 不允许把Task移动到PIP模式的Task中。
                                Slog.w(TAG, "Can't support moving a task to another PIP window..."
                                        + " newParent=" + newParent + " task=" + task);
                                return 0;
                            }
                            if (!task.supportsMultiWindowInDisplayArea(
                                    newParent.asTask().getDisplayArea())) {
                                // 不允许把不支持分屏的Task移动到分屏rootTask中。
                                Slog.w(TAG, "Can't support task that doesn't support multi-window"
                                        + " mode in multi-window mode... newParent=" + newParent
                                        + " task=" + task);
                                return 0;
                            }
                        }
                        // 直接调用Task#reparent完成reparent操作。
                        task.reparent((Task) newParent,
                                hop.getToTop() ? POSITION_TOP : POSITION_BOTTOM,
                                false /*moveParents*/, "sanitizeAndApplyHierarchyOp");
                    } else {
                        // 目前只支持对Task类型或者TaskDisplayArea类型进行reparent的操作。
                        throw new RuntimeException("Can only reparent task to another task or"
                                + " taskDisplayArea, but not " + newParent);
                    }
                } else {
                    // newParent和当前Task的parent相同，那么此次就不是一个reparent操作，而是reorder，
                    // 并且要调整的是当前Task的rootTask在与其对应的TaskDisplayArea中的位置。
                    // 综合起来，走到这里有两种情况：
                    // 1)、task是rootTask，newParent是一个TaskDisplayArea实例，那么调整task的rootTask（也就是它自己？）在TaskDisplayArea中的位置。
                    // 2)、task不是rootTask，newParent是一个由TaskOrganizerController创建的Task，那么调整newParent在TaskDisplayArea中的位置。
                    final Task rootTask = (Task) (
                            (newParent != null && !(newParent instanceof TaskDisplayArea))
                                    ? newParent : task.getRootTask());
                    as.getDisplayArea().positionChildAt(
                            hop.getToTop() ? POSITION_TOP : POSITION_BOTTOM, rootTask,
                            false /* includingParents */);
                }
            } else {
                // 这里的leafTask指的是那些父容器为Task，且该Task不是由TaskOrganizerController创建的那些Task。
                // 目前不支持对这些Task进行reparent操作。
                throw new RuntimeException("Reparenting leaf Tasks is not supported now. " + task);
            }
        } else {
            // 如果此次层级调整不是reparent，那么就是一次reorder操作。
            // 直接取到当前Task的父WindowContainer，然后调用WindowContainer#positionChildAt调整当前Task在父容器中的位置即可。
            task.getParent().positionChildAt(
                    hop.getToTop() ? POSITION_TOP : POSITION_BOTTOM,
                    task, false /* includingParents */);
        }
        return TRANSACT_EFFECTS_LIFECYCLE;
    }
```

这部分内容分析比较繁琐，直接加到注释里。

针对 reaprent 的部分总结一下：

首先能进行 reparent 的 Task，必须满足以下两种条件之一：

1)、当前 Task 是一个 rootTask。

2)、如果当前 Task 不是一个 rootTask，那么其直接的父 Task 是由 TaskOrganzierController 创建的。

也就是说，我们无法去调整那些一般的 HOME 类型、RECENTS 类型的 Task 中的那些子嵌套 Task。

那么再跟据 newParent 和 task 的当前 parent 之间的关系，可以将 reparent 的情形总结为以下几种：

1)、newParent 为 null，那么将当前 Task 从原来的父 WindowContainer 移动到当前 TaskDisplayArea 中，这种情况在退出分屏的流程中常见。

2)、newParent 不为 null，且不等于当前 Task 的当前父 WindowContainer，那么将该 task 移动到 newParent 中，这种情况在进入分屏的流程中常见。

3)、newParent 不为 null，且等于 task 的当前父 WindowContainer，那么只调整 task 对应的 rootTask 在 TaskDisplayArea 的顺序，这种情况比较少见，应该是为了 cover 逻辑上的 bug，因为有更加直接的方法 WindowContainerTransaction#reorder 方法可以使用。

### 4 WCT#setBoundsChangeTransaction 相关

```
// Queue-up bounds-change transactions for tasks which are now organized. Do
            // this after hierarchy ops so we have the final organized state.
            entries = t.getChanges().entrySet().iterator();
            while (entries.hasNext()) {
                final Map.Entry<IBinder, WindowContainerTransaction.Change> entry = entries.next();
                final WindowContainer wc = WindowContainer.fromBinder(entry.getKey());
                if (wc == null || !wc.isAttached()) {
                    Slog.e(TAG, "Attempt to operate on detached container: " + wc);
                    continue;
                }
                final Task task = wc.asTask();
                final Rect surfaceBounds = entry.getValue().getBoundsChangeSurfaceBounds();
                if (task == null || !task.isAttached() || surfaceBounds == null) {
                    continue;
                }
                if (!task.isOrganized()) {
                    final Task parent = task.getParent() != null ? task.getParent().asTask() : null;
                    // Also allow direct children of created-by-organizer tasks to be
                    // controlled. In the future, these will become organized anyways.
                    if (parent == null || !parent.mCreatedByOrganizer) {
                        throw new IllegalArgumentException(
                                "Can't manipulate non-organized task surface " + task);
                    }
                }
                final SurfaceControl.Transaction sft = new SurfaceControl.Transaction();
                final SurfaceControl sc = task.getSurfaceControl();
                sft.setPosition(sc, surfaceBounds.left, surfaceBounds.top);
                if (surfaceBounds.isEmpty()) {
                    sft.setWindowCrop(sc, null);
                } else {
                    sft.setWindowCrop(sc, surfaceBounds.width(), surfaceBounds.height());
                }
                task.setMainWindowSizeChangeTransaction(sft);
            }
```

这部分理解的还不够，之前碰到过问题是和这个相关的，可以看当时的分析。

### 5 更新 Activity 的可见性和 Configuration

```
if ((effects & TRANSACT_EFFECTS_LIFECYCLE) != 0) {
                // Already calls ensureActivityConfig
                mService.mRootWindowContainer.ensureActivitiesVisible(null, 0, PRESERVE_WINDOWS);
                mService.mRootWindowContainer.resumeFocusedTasksTopActivities();
            } else if ((effects & TRANSACT_EFFECTS_CLIENT_CONFIG) != 0) {
                final PooledConsumer f = PooledLambda.obtainConsumer(
                        ActivityRecord::ensureActivityConfiguration,
                        PooledLambda.__(ActivityRecord.class), 0,
                        true /* preserveWindow */);
                try {
                    for (int i = haveConfigChanges.size() - 1; i >= 0; --i) {
                        haveConfigChanges.valueAt(i).forAllActivities(f);
                    }
                } finally {
                    f.recycle();
                }
            }

            if ((effects & TRANSACT_EFFECTS_CLIENT_CONFIG) == 0) {
                mService.addWindowLayoutReasons(LAYOUT_REASON_CONFIG_CHANGED);
            }
```

1)、如果有生命周期相关的更新，就调用

```
if ((effects & TRANSACT_EFFECTS_LIFECYCLE) != 0) {
                // Already calls ensureActivityConfig
                mService.mRootWindowContainer.ensureActivitiesVisible(null, 0, PRESERVE_WINDOWS);
                mService.mRootWindowContainer.resumeFocusedTasksTopActivities();
            }
```

重新更新系统中的所有 Activity 的可见性和 Configuration，并且寻找新的 top Task 和 resume Activty。

2)、如果没有生命周期相关方面的更新，只有 Configuration 相关的更新，那么执行：

```
else if ((effects & TRANSACT_EFFECTS_CLIENT_CONFIG) != 0) {
                final PooledConsumer f = PooledLambda.obtainConsumer(
                        ActivityRecord::ensureActivityConfiguration,
                        PooledLambda.__(ActivityRecord.class), 0,
                        true /* preserveWindow */);
                try {
                    for (int i = haveConfigChanges.size() - 1; i >= 0; --i) {
                        haveConfigChanges.valueAt(i).forAllActivities(f);
                    }
                } finally {
                    f.recycle();
                }
            }
```

只针对这些发生了 Configuration 更新的 WindowContainer，遍历其下的所有子 ActivityRecord，调用 ActivityRecord#ensureActivityConfiguration 进行 Configuration 相关的更新。

三、总结
----

WindowContainerTransaction 的应用，就是 WindowOrganizerController 读取 WindowContainerTransaction 中保存的信息，按照 Transition、Change、HierarchyOp 和 BoundsChange 这几部分进行分别处理。本文重点分析了 Change 和 HierarchyOp 部分，因为这两部分和分屏是重点相关的。

另外本文所说的 WindowContainerTransaction 的应用，特指从 WindowContainerTransaction 中读取信息然后应用到 WindowContainer 上这一行为，应用的结果是 WM 端的 WindowContainer 层级结构中一些 WindowContainer 的改变（属性和层级）的立即改变。

除此之外 WindowContainerTransaction 的应用还具有同步的特点，这一点在本文中并未体现，后续在分析 WindowContainerTransaction 的同步机制中再去深入探究。
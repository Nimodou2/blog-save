> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/126747856?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/ed1b74621cbc49a2ba271b1584bf1ed7.jpeg#pic_center)

一、BLASTSyncEngine 定义
--------------------

```
/**
 * Utility class for collecting WindowContainers that will merge transactions.
 * For example to use to synchronously resize all the children of a window container
 *   1. Open a new sync set, and pass the listener that will be invoked
 *        int id startSyncSet(TransactionReadyListener)
 *      the returned ID will be eventually passed to the TransactionReadyListener in combination
 *      with a set of WindowContainers that are ready, meaning onTransactionReady was called for
 *      those WindowContainers. You also use it to refer to the operation in future steps.
 *   2. Ask each child to participate:
 *       addToSyncSet(int id, WindowContainer wc)
 *      if the child thinks it will be affected by a configuration change (a.k.a. has a visible
 *      window in its sub hierarchy, then we wi`ll increment a counter of expected callbacks
 *      At this point the containers hierarchy will redirect pendingTransaction and sub hierarchy
 *      updates in to the sync engine.
 *   3. Apply your configuration changes to the window containers.
 *   4. Tell the engine that the sync set is ready
 *       setReady(int id)
 *   5. If there were no sub windows anywhere in the hierarchy to wait on, then
 *      transactionReady is immediately invoked, otherwise all the windows are poked
 *      to redraw and to deliver a buffer to {@link WindowState#finishDrawing}.
 *      Once all this drawing is complete, all the transactions will be merged and delivered
 *      to TransactionReadyListener.
 *
 * This works primarily by setting-up state and then watching/waiting for the registered subtrees
 * to enter into a "finished" state (either by receiving drawn content or by disappearing). This
 * checks the subtrees during surface-placement.
 */
class BLASTSyncEngine
```

渣翻：

用于收集将 merge transactions 的 WindowContainers 的工具类。

例如用于同步调整一个 WindowContainer 下的所有子 WindowContainer 的尺寸 ：

1)、打开一个新的同步集，并传递将被调用的 listener：

```
int id startSyncSet(TransactionReadyListener)
```

返回的 ID 最终会与一组准备好的 WindowContainers 一起传递给 TransactionReadyListener，这意味那些 WindowContainers 已经调用了 onTransactionReady。您还可以使用它来指代以后步骤中的操作。

2)、 要求每个子容器参与：

```
addToSyncSet(int id, WindowContainer wc)
```

如果子容器认为它会受到 Configuration 更改的影响（也就是在其子层次结构中有一个可见的窗口，那么我们将增加一个预期的计数器回调）。此时，容器层次结构会将 pendingTransaction 和子层次结构更新重定向到同步引擎。

3)、将 configurationchanges 应用到 WindowContainers。

4)、告诉 BLASTSyncEngine 同步集已准备好：

```
setReady(int id)
```

5)、如果层次结构中的任何地方都没有子窗口等待，则立即调用 transactionReady，否则所有窗口都将被推动去重绘和去给 WindowState#finishDrawing 传送缓冲区。一旦所有这些绘制完成，所有 transaction 将被合并并且交付给 TransactionReadyListener。

这主要通过设置状态然后观察 / 等待注册的子树进入 “完成” 状态（通过接收绘制的内容或消失）来工作。这会在 [surface](https://so.csdn.net/so/search?q=surface&spm=1001.2101.3001.7020)-placement 期间检查子树。

根据 WindowOrganizerController 的情况看下 1-5 条的执行情况。

二、同步流程分析
--------

在分析 WindowContainerTransaction 的应用时，关于 WindowContainerTransaction#applySyncTransaction 方法，当时是跳过了很多内容直接去分析 WindowContainerTransaction 是如何应用到 WindowContainer 上了，那些跳过的内容也就是和同步机制相关的，这次就来跟踪下同步的流程。

```
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

### 1 调用 BLASTSyncEngine#startSyncSet 创建 SyncGroup

WindowOrganizerController#applyTransaction 的根据之前的分析，已经知道了，就是为 WindowContainer 应用 WindowContainerTransaction 中的设置。

在 applyTransaction 之前，有这样一个操作：

```
@Override
    public int applySyncTransaction(WindowContainerTransaction t,
            IWindowContainerTransactionCallback callback) {
	    ......
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
			   ......
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```

这里，如果是通过 WindowOrganizer#applyTransaction，那么这里传入 callback 的就是 null。

如果是 WindowOrganizer#applySyncTransaction，就会传入一个 callback，之前分析 WCT 的发送的时候，知道这里的 callback 是一个 SyncCallback 类型的回调。接下来看下 callback 不为空的情况。

```
@VisibleForTesting
    int startSyncWithOrganizer(IWindowContainerTransactionCallback callback) {
        int id = mService.mWindowManager.mSyncEngine.startSyncSet(this);
        mTransactionCallbacksByPendingSyncId.put(id, callback);
        return id;
    }
```

这里调用了 BLASTSyncEngine#startSyncSet，BLASTSyncEngine#startSyncSet 返回了一个唯一的 ID，接着把这个 ID 和 callback 存储到 HashMp 类型的 mTransactionCallbacksByPendingSyncId 中去，那么可以知道 ID 和 callback 应该是一一对应的关系。后续当 WM 端的 transaction 完成后，在 WindowOrganizerController#onTransactionReady 中会根据 ID 再取回 callback。

接着看下 BLASTSyncEngine#startSyncSet 都干了什么。

```
int startSyncSet(TransactionReadyListener listener) {
        final int id = mNextSyncId++;
        final SyncGroup s = new SyncGroup(listener, id);
        mActiveSyncs.put(id, s);
        ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "SyncGroup %d: Started for listener: %s", id, listener);
        return id;
    }
```

1)、将 mNextSyncId 赋值给 id，然后自增 1。

```
private int mNextSyncId = 0;
```

这里看到，mNextSyncId 是 BLASTSyncEngine 的成员变量，由于 BLASTSyncEngine 只在 [WMS](https://so.csdn.net/so/search?q=WMS&spm=1001.2101.3001.7020) 的构造器中创建了一次，唯一实例由 WMS 持有，那么可以知道 mNextSyncId 也是全局唯一一份，每次 BLASTSyncEngine#startSyncSet 都会自增 1，保证了 ID 的唯一性。

2)、以这个唯一 ID 和传入的 WindowOrganizerController 对象为基础，创建了一个 SyncGroup 对象。

```
private SyncGroup(TransactionReadyListener listener, int id) {
            mSyncId = id;
            mListener = listener;
        }
```

SyncGroup 是 BLASTSyncEngine 的内部类，创建的唯一位置便是在 BLASTSyncEngine#startSyncSet。

3)、将 ID 和创建的 SyncGroup 对象加入到 mActiveSyncs 队列中。

```
private final SparseArray<SyncGroup> mActiveSyncs = new SparseArray<>();
```

mActiveSyncs 是 BLASTSyncEngine 的成员变量。

4)、最后，将这个具有唯一性的 ID 返回。

那么这个 ID 的值是唯一的，对应了一个 SyncGroup 对象，也对应了一个 callback。

### 2 调用 BLASTSyncEngine#addToSyncSet 将所有参与同步的 WindowContainer 添加到 SyncGroup 中

在 WindowOrganizerController#startSyncWithOrganizer 之后，便是调用 WindowOrganizerController#applyTransaction 来为 WindowContainer 应用 WindowContainerTransaction 设置的修改（即 Configuration 更改，层级调整之类的）：

```
/**
     * @param syncId If non-null, this will be a sync-transaction.
     * @param transition A transition to collect changes into.
     */
    private void applyTransaction(@NonNull WindowContainerTransaction t, int syncId,
            @Nullable Transition transition) {
		......
        try {
			......
            Iterator<Map.Entry<IBinder, WindowContainerTransaction.Change>> entries =
                    t.getChanges().entrySet().iterator();
            while (entries.hasNext()) {
				......
                // Make sure we add to the syncSet before performing
                // operations so we don't end up splitting effects between the WM
                // pending transaction and the BLASTSync transaction.
                if (syncId >= 0) {
                    addToSyncSet(syncId, wc);
                }
                if (transition != null) transition.collect(wc);

                int containerEffect = applyWindowContainerChange(wc, entry.getValue());
			   ......
            }
			......
        } finally {
            mService.continueWindowLayout();
        }
    }
```

然而在调用 WindowOrganizerController#applyWindowContainerChange 方法应用之前修改之前，还有一个重要的步骤：

```
// Make sure we add to the syncSet before performing
                // operations so we don't end up splitting effects between the WM
                // pending transaction and the BLASTSync transaction.
                if (syncId >= 0) {
                    addToSyncSet(syncId, wc);
                }
```

这个方法目前看到有三处需要执行，一是在 WindowOrganizerController#applyWindowContainerChange 之前，二是在 WindowOrganizerController#reparentChildrenTasksHierarchyOp 之前，三是在 WindowOrganizerController#sanitizeAndApplyHierarchyOp 之前。

接下来分析 WindowOrganzierController#addToSyncSet 的内容：

```
@VisibleForTesting
    void addToSyncSet(int syncId, WindowContainer wc) {
        mService.mWindowManager.mSyncEngine.addToSyncSet(syncId, wc);
    }
```

WindowOrganizerController#addToSyncSet 调用了 BLASTSyncEngine#addToSyncSet。

```
void addToSyncSet(int id, WindowContainer wc) {
        mActiveSyncs.get(id).addToSync(wc);
    }
```

在第一小节中我们创建了 SyncGroup 对象，并且和 ID 是一一对应，保存在了 BLASTSyncEngine 的成员变量 mActiveSyncs 中，所以这里就可以通过 ID 从 mActiveSyncs 中拿到对应的 SyncGroup。

那么这里调用的就是 SyncGroup#addToSyncSet：

```
private void addToSync(WindowContainer wc) {
            if (!mRootMembers.add(wc)) {
                return;
            }
            ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "SyncGroup %d: Adding to group: %s", mSyncId, wc);
            wc.setSyncGroup(this);
            wc.prepareSync();
            mWm.mWindowPlacerLocked.requestTraversal();
        }
```

接下来的内容就是，之前 BLASTSyncEngine 注释中所提到的，将所有参与同步的 WindowContainer 都加入到同步集中，参与同步的 WindowContainer，也就是 WindowContainerTransaction 中保存的那些客户端指定的 WindowContainer。

逐步分析 SyncGroup#addToSync 的内容。

#### 2.1 将传入的 WindowContainer 加入到 SyncGroup 的 mRootMembers 队列

```
if (!mRootMembers.add(wc)) {
                return;
            }
```

mRootMembers 是 SyncGroup 的成员变量：

```
final ArraySet<WindowContainer> mRootMembers = new ArraySet<>();
```

这里使用 ArraySet 保证 WindowContainer 只会被添加一次，毕竟我们上面看到，入口有三处。

后续 SyncGroup 通过遍历 mRootMembers 队列来判断所有参与同步的 WindowContainer 是否完成同步。

#### 2.2 WindowContainer#setSyncGroup

```
void setSyncGroup(@NonNull BLASTSyncEngine.SyncGroup group) {
        ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "setSyncGroup #%d on %s", group.mSyncId, this);
        if (group != null) {
            if (mSyncGroup != null && mSyncGroup != group) {
                throw new IllegalStateException("Can't sync on 2 engines simultaneously");
            }
        }
        mSyncGroup = group;
    }
```

很简单，将 WindowContainer.mSyncGroup 指向当前 SyncGroup。

WindowContainer 的成员变量 mSyncGroup 的定义是：

```
/**
     * If non-null, references the sync-group directly waiting on this container. Otherwise, this
     * container is only being waited-on by its parents (if in a sync-group). This has implications
     * on how this container is handled during parent changes.
     */
    BLASTSyncEngine.SyncGroup mSyncGroup = null;
```

如果非空，说明其 SyncGroup 正在等待此 WindowContainer。否则，这个 WindowCOntainer 只会被它的父 WindowContainer 等待（如果在 SyncGroup 中）。这会影响在父 WindowContainer 更改期间这个 WindowContainer 要如何被处理。

#### 2.3 WindowContainer#prepareSync

```
/**
     * Prepares this container for participation in a sync-group. This includes preparing all its
     * children.
     *
     * @return {@code true} if something changed (eg. this wasn't already in the sync group).
     */
    boolean prepareSync() {
        if (mSyncState != SYNC_STATE_NONE) {
            // Already part of sync
            return false;
        }
        for (int i = getChildCount() - 1; i >= 0; --i) {
            final WindowContainer child = getChildAt(i);
            child.prepareSync();
        }
        mSyncState = SYNC_STATE_READY;
        return true;
    }
```

这里新增了一个 WindowContainer.mSyncState 的概念：

```
/** This isn't participating in a sync. */
    public static final int SYNC_STATE_NONE = 0;

    /** This is currently waiting for itself to finish drawing. */
    public static final int SYNC_STATE_WAITING_FOR_DRAW = 1;

    /** This container is ready, but it might still have unfinished children. */
    public static final int SYNC_STATE_READY = 2;

    @IntDef(prefix = { "SYNC_STATE_" }, value = {
            SYNC_STATE_NONE,
            SYNC_STATE_WAITING_FOR_DRAW,
            SYNC_STATE_READY,
    })
    @interface SyncState {}
```

SYNC_STATE_NONE 表示当前 WindowContainer 没有参与到一次同步操作中。

SYNC_STATE_WAITING_FOR_DRAW 表示当前 WindowContainer 正在等待自身绘制完成。

SYNC_STATE_READY 表示当前 WindowContainer 已经绘制完成，但是其子 WindowContainer 中可能还有没有完成同步的。

可以看到，在同步之前，需要将 mSyncState 置为 SYNC_STATE_READY。如果 mSyncState 不是 SYNC_STATE_NONE，表示当前 WindowContainer 仍然处于同步过程中。并且这里还调用了子容器的 prepareSync 方法，这一步将使得所有子容器的 mSyncState 都被置为 SYNC_STATE_READY。

这里可能会比较疑惑，为什么直接就把 WindowContainer 的 mSyncState 的状态设置为 SYNC_STATE_READY 了，同步不是才刚开始？我们看到，除了 WindowContainer 有 prepareSync 方法之外，WindowState 也重写了这个方法：

```
@Override
    boolean prepareSync() {
        if (!super.prepareSync()) {
            return false;
        }
        // In the WindowContainer implementation we immediately mark ready
        // since a generic WindowContainer only needs to wait for its
        // children to finish and is immediately ready from its own
        // perspective but at the WindowState level we need to wait for ourselves
        // to draw even if the children draw first our don't need to sync, so we start
        // in WAITING state rather than READY.
        mSyncState = SYNC_STATE_WAITING_FOR_DRAW;
        requestRedrawForSync();

        mWmService.mH.removeMessages(WINDOW_STATE_BLAST_SYNC_TIMEOUT, this);
        mWmService.mH.sendNewMessageDelayed(WINDOW_STATE_BLAST_SYNC_TIMEOUT, this,
            BLAST_TIMEOUT_DURATION);
        return true;
    }
```

这里的注释也可以解答我们刚刚的疑问：

在 WindowContainer 的实现中，我们直接将 mSyncState 置为 SYNC_STATE_READY，因为对于一个通用的 WindowContainer 而言，它只需要等待它的子 WindowContainer 完成同步，如果它有子 WindowContainer 完成同步，那么对它自己而言它也就完成同步处于就绪状态了。

但是在 WindowState 这一层，每一个 WindowState 都需要等待自己绘制完成，即使子窗口先绘制完成。所以对于 WindowState，是以 SYNC_STATE_WAITING_FOR_DRAW 状态开始而不是 SYNC_STATE_READY 状态。

### 3 将 WindowContainerTransaction 应用到 WindowContainer

```
@Override
    public int applySyncTransaction(WindowContainerTransaction t,
            IWindowContainerTransactionCallback callback) {
        ......
        try {
            synchronized (mGlobalLock) {
			   ......
                applyTransaction(t, syncId, null /*transition*/);
			   ......
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```

这一步的具体内容在分析 WCT 的应用的时候有比较系统的介绍过了。

### 4 BLASTSyncTransaction#setReady 标记 SyncGroup 已经准备就绪

```
@Override
    public int applySyncTransaction(WindowContainerTransaction t,
            IWindowContainerTransactionCallback callback) {
        ......
        try {
            synchronized (mGlobalLock) {
			   ......
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

这一步发生在 WindowOrganizerController#applyTransaction 之后，即 WindowContainerTransaction 中的设置都已经应用到 WindowContainer 上了。

```
@VisibleForTesting
    void setSyncReady(int id) {
        ProtoLog.v(WM_DEBUG_WINDOW_ORGANIZER, "Set sync ready, syncId=%d", id);
        mService.mWindowManager.mSyncEngine.setReady(id);
    }

    @VisibleForTesting
    void addToSyncSet(int syncId, WindowContainer wc) {
        mService.mWindowManager.mSyncEngine.addToSyncSet(syncId, wc);
    }
```

WindowOrganizerController#setSyncReady 调用 BLASTSyncEngine#setReady。

```
void setReady(int id, boolean ready) {
        mActiveSyncs.get(id).setReady(ready);
    }

    void setReady(int id) {
        setReady(id, true);
    }
```

BLASTSyncEngine#setReady 调用 SyncGroup#setReady。

```
private void setReady(boolean ready) {
            ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "SyncGroup %d: Set ready", mSyncId);
            mReady = ready;
            if (!ready) return;
            mWm.mWindowPlacerLocked.requestTraversal();
        }
```

将 SyncGroup 的成员变量 mReady 置为 true，表示所有 WindowContainerTransaction 都已经应用完成，然后请求一次界面刷新。

### 5 等待窗口绘制完成后调用 transactionReady

每次刷新界面的时候，RootWindowContainer#performSurfacePlacementNoTrace 会调用 BLASTSyncEngine#onSurfacePlacement。

```
void onSurfacePlacement() {
        // backwards since each state can remove itself if finished
        for (int i = mActiveSyncs.size() - 1; i >= 0; --i) {
            mActiveSyncs.valueAt(i).onSurfacePlacement();
        }
    }
```

这里又调用了 SyncGroup#onSurfacePlacement。

```
private void onSurfacePlacement() {
            if (!mReady) return;
            ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "SyncGroup %d: onSurfacePlacement checking %s",
                    mSyncId, mRootMembers);
            for (int i = mRootMembers.size() - 1; i >= 0; --i) {
                final WindowContainer wc = mRootMembers.valueAt(i);
                if (!wc.isSyncFinished()) {
                    ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "SyncGroup %d:  Unfinished container: %s",
                            mSyncId, wc);
                    return;
                }
            }
            finishNow();
        }
```

这里会对加入到 mRootMembers 的 WindowContainer 进行遍历，mRootMembers 里的数据是之前调用 addToSync 的时候填充的。WindowContainer#isSyncFinished 返回 true 表示当前 WindowContainer 同步完成。只有当 mRootMembers 中所有的 WindowContainer 都同步完成，本次同步才算结束，才可以调用 SyncGroup#finishNow。

由于这个方法在每次界面刷新的时候都会调用，所以这里的工作就是不断检查当前 SyncGroup 中的所有 WindowContainer 是否同步完成，一旦同步完成，就去调用 SyncGroup#finishNow。

这里先跳过 WindowContainer#isSyncFinished，看下 SyncGroup#finishNow：

```
private void finishNow() {
            ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "SyncGroup %d: Finished!", mSyncId);
            SurfaceControl.Transaction merged = mWm.mTransactionFactory.get();
            if (mOrphanTransaction != null) {
                merged.merge(mOrphanTransaction);
            }
            for (WindowContainer wc : mRootMembers) {
                wc.finishSync(merged, false /* cancel */);
            }
            mListener.onTransactionReady(mSyncId, merged);
            mActiveSyncs.remove(mSyncId);
        }
```

SyncGroup#finishNow 做了以下几件事：

#### 5.1 取得一个存放最终结果的 Transaction

```
SurfaceControl.Transaction merged = mWm.mTransactionFactory.get();
            if (mOrphanTransaction != null) {
                merged.merge(mOrphanTransaction);
            }
```

将 mOrphanTransaction 合并到此 Transaction 中，mOrphanTransactionon 从调用的地方来看，当 SyncGroup 中的 WindowContainer 的父 WindowContainer 发生变化时，会将该 WindowContainer 的 mSyncTransaction 合入到 mOrphanTransactionon，避免该 WindowContainer 的 mSyncTransaction 保存的修改在 parent 更改的时候丢失。

#### 5.2 调用 WindowContainer#finishSync

```
/**
     * Recursively finishes/cleans-up sync state of this subtree and collects all the sync
     * transactions into `outMergedTransaction`.
     * @param outMergedTransaction A transaction to merge all the recorded sync operations into.
     * @param cancel If true, this is being finished because it is leaving the sync group rather
     *               than due to the sync group completing.
     */
    void finishSync(Transaction outMergedTransaction, boolean cancel) {
        if (mSyncState == SYNC_STATE_NONE) return;
        ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "finishSync cancel=%b for %s", cancel, this);
        outMergedTransaction.merge(mSyncTransaction);
        for (int i = mChildren.size() - 1; i >= 0; --i) {
            mChildren.get(i).finishSync(outMergedTransaction, cancel);
        }
        mSyncState = SYNC_STATE_NONE;
        if (cancel && mSyncGroup != null) mSyncGroup.onCancelSync(this);
        mSyncGroup = null;
    }
```

这里传入的 5.1 中创建的存放最终结果的 outMergedTransaction。

1)、将当前 WindowContainer 自身的 mSyncTransaction 合入到 outMergedTransaction。

2)、WindowContainer#finishSync 递归调用所有子 WindowContainer#finishSync，最终将 WindowContainerTransaction 带来的所有 Transaction 变化全部都合并到 outMergedTransaction。

3)、将 mSyncState 重置为同步前的状态 SYNC_STATE_NONE，mSyncGroup 置空。

#### 5.3 调用 TransactionReadyListener#onTransactionReady

```
mListener.onTransactionReady(mSyncId, merged);
```

这里的 TransactionReadyListener 就是 WindowOrganizerController，是在 WindowOrganizerController 调用 startSyncSet 创建 SyncGroup 的时候传入的，所以这里调用的是 WindowOrganizerController#onTransactionReady：

```
@Override
    public void onTransactionReady(int syncId, SurfaceControl.Transaction t) {
        ProtoLog.v(WM_DEBUG_WINDOW_ORGANIZER, "Transaction ready, syncId=%d", syncId);
        final IWindowContainerTransactionCallback callback =
                mTransactionCallbacksByPendingSyncId.get(syncId);

        try {
            callback.onTransactionReady(syncId, t);
        } catch (RemoteException e) {
            // If there's an exception when trying to send the mergedTransaction to the client, we
            // should immediately apply it here so the transactions aren't lost.
            t.apply();
        }

        mTransactionCallbacksByPendingSyncId.remove(syncId);
    }
```

所有参与到此次同步的 WindowContainer 都完成同步后，会由 WindowOrganizerController 根据 syncId 从 mTransactionCallbacksByPendingSyncId 中找到对应的回调对象，即 SyncCallback，然后调用 SyncCallback#onTransactionReady，并且传入存放了最终同步结果的 Transaction。

#### 5.4 将 ID 从 mActiveSyncs 中移除。

```
mActiveSyncs.remove(mSyncId);
```

直到这里，本次同步操作在 WM 端的部分才算完成了，所有 configuration 更改引起的 Transaction 变化全部都合并到了一个 Transaction 中，并且传给了 SystemUI，后续需要 SystemUI 的 SyncTransactionQueue 那边进行 apply，之前对 SyncTransactionQueue 的分析中也有讲过。

#### 5.5 WindowContainer 何时被视为同步完成

上面跳过了一个重要环节，在 BLASTSyncEngine#onSurfacePlacement 中，会对参与本次同步的所有 WindowContainer 调用 WindowContainer#isSyncFinished 判断其是否同步完成。只有 SyncGroup 中的所有 WindowContainer 都同步完成，才会调用 SyncGroup#finishNow。

当时跳过了 WindowContainer#isSyncFinished 的分析，这一节来详细分析一下。

```
/**
     * Checks if the subtree rooted at this container is finished syncing (everything is ready or
     * not visible). NOTE, this is not const: it will cancel/prepare itself depending on its state
     * in the hierarchy.
     *
     * @return {@code true} if this subtree is finished waiting for sync participants.
     */
    boolean isSyncFinished() {
        if (!isVisibleRequested()) {
            return true;
        }
        if (mSyncState == SYNC_STATE_NONE) {
            prepareSync();
        }
        if (mSyncState == SYNC_STATE_WAITING_FOR_DRAW) {
            return false;
        }
        // READY
        // Loop from top-down.
        for (int i = mChildren.size() - 1; i >= 0; --i) {
            final WindowContainer child = mChildren.get(i);
            final boolean childFinished = child.isSyncFinished();
            if (childFinished && child.isVisibleRequested() && child.fillsParent()) {
                // Any lower children will be covered-up, so we can consider this finished.
                return true;
            }
            if (!childFinished) {
                return false;
            }
        }
        return true;
    }
```

判断一个 WindowContainer 是否完成同步，不仅当前 WindowContainer 的 mSyncState 得是 SYNC_STATE_READY，它的子 WindowContainer 也要有完成同步的。

根据之前对 WindowContaIner#prepareSync 的分析，非 WindowState 类型的容器，在同步开始的时候都已经在 WindowContainer#prepareSync 中设置其 mSyncState 为 SYNC_STATE_READY 了，所以这里只需要看 WindowState 类型的子容器有没有就绪就好了，其它类型的 WIndowContainer 都不需要关注。

另外 WindowState 类型的容器一开始在 WindowState#prepareSync 中将 mSyncState 设置为 SYNC_STATE_WAITING_FOR_DRAW，那么对于 WindowState 来说，isSyncFinished 方法能够返回 true，首要条件就是，mSyncState 必须在后续被设置为 SYNC_STATE_READY，这一步只能在 WindowContainer#onSyncFinishedDrawing：

```
/**
     * Call this when this container finishes drawing content.
     *
     * @return {@code true} if consumed (this container is part of a sync group).
     */
    boolean onSyncFinishedDrawing() {
        if (mSyncState == SYNC_STATE_NONE) return false;
        mSyncState = SYNC_STATE_READY;
        mWmService.mWindowPlacerLocked.requestTraversal();
        ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "onSyncFinishedDrawing %s", this);
        return true;
    }
```

WindowContainer#onSyncFinishedDrawing 唯一调用的地方又在 WindowState#finishDrawing：

```
boolean finishDrawing(SurfaceControl.Transaction postDrawTransaction) {
        if (mOrientationChangeRedrawRequestTime > 0) {
            final long duration =
                    SystemClock.elapsedRealtime() - mOrientationChangeRedrawRequestTime;
            Slog.i(TAG, "finishDrawing of orientation change: " + this + " " + duration + "ms");
            mOrientationChangeRedrawRequestTime = 0;
        } else if (mActivityRecord != null && mActivityRecord.mRelaunchStartTime != 0
                && mActivityRecord.findMainWindow() == this) {
            final long duration =
                    SystemClock.elapsedRealtime() - mActivityRecord.mRelaunchStartTime;
            Slog.i(TAG, "finishDrawing of relaunch: " + this + " " + duration + "ms");
            mActivityRecord.mRelaunchStartTime = 0;
        }

        executeDrawHandlers(postDrawTransaction);
        if (!onSyncFinishedDrawing()) {
            return mWinAnimator.finishDrawingLocked(postDrawTransaction);
        }

        if (postDrawTransaction != null) {
            mSyncTransaction.merge(postDrawTransaction);
        }

        mWinAnimator.finishDrawingLocked(null);
        // We always want to force a traversal after a finish draw for blast sync.
        return true;
    }
```

这个方法唯一的调用栈是从 APP 端调过来的：

ViewRootImpl#reportedDrawFinished

```
-> Session#finishDrawing

	（跨进程）

	-> WMS#finishDrawingWindow

		-> WindowState#finishDrawing

			-> WindowContainer#onSyncFinishedDrawing
```

那么也就是说，只有 APP 端完成了界面绘制并且通知到了 WM 端，当前 WindowState 才能设置其 mSyncState 为 SYNC_STATE_READY。

然而这个还是条件之一，判断一个 WindowContainer 是否完成同步，还有其他限制：

```
// READY
        // Loop from top-down.
        for (int i = mChildren.size() - 1; i >= 0; --i) {
            final WindowContainer child = mChildren.get(i);
            final boolean childFinished = child.isSyncFinished();
            if (childFinished && child.isVisibleRequested() && child.fillsParent()) {
                // Any lower children will be covered-up, so we can consider this finished.
                return true;
            }
            if (!childFinished) {
                return false;
            }
        }
        return true;
```

1)、如果我们能先找到一个已经完成绘制的子窗口，并且该子窗口后续要显示，且能够覆盖的了它下面的所有子窗口，那么可以认为当前 WindowContainer 绘制完成。

2)、如果我们找不到一个满足条件 1 的子窗口，但是所有窗口都已经绘制完成，那么也可以认为当前 WindowContainer 绘制完成；

3)、如果我们在找到一个满足条件 1 的子窗口之前，先找到一个没有完成绘制的子窗口，那么就认为当前 WindowContainer 没有绘制完成。

三、总结
----

### 1 BLASTSyncEngine 机制流程

1)、在 WindowOrganizerController#applyTransaction 之前，调用 BLASTSyncEngine#startSyncSet，创建一个拥有唯一 ID 的 SyncGroup 对象：

WindowOrganizerController#startSyncWithOrganizer

```
-> BLASTSyncEngine#startSyncSet
```

2)、在 WindowContainerTransaction 中的设置应用到各个 WindowContainer 之前，调用 BLASTSyncEngine#addToSyncSet 将这些要应用 WindowContainerTransaction 的 WindowContainer 加入到 SyncGroup 中，并且调用 WindowContainer#prepareSync 为这些 WindowContainer 设置初始状态：

WindowOrganizerController#addToSyncSet

```
-> BLASTSyncEngine#addToSyncSet 

	-> SyncGroup#addToSync 

		-> WindowContainer#prepareSync
```

3)、所有参与同步的 WindowContainer 应用 WindowContainerTransaction。

4)、所有参与同步的 WindowContainer 应用 WindowContainerTransaction 之后，通知 BLASTSyncEngineSyncGroup 已经准备就绪，可以开始请求绘制。

WindowOrganizerController#setSyncReady

```
-> BLASTSyncEngine#setReady 

	-> BLASTSyncEngine#setReady 

		-> SyncGroup#setReady
```

随后在每次界面刷新时，都会调用 BLASTSyncEngine#onSurfacePlacement，不断去检查本次 SyncGroup 中的所有 WindowContainer 是否已经全部完成同步：

RootWindowContainer#performSurfacePlacementNoTrace

```
-> BLASTSyncEngine#onSurfacePlacement 

	-> SyncGroup#onSurfacePlacement
```

5)、一旦所有的容器都已经同步完成，那么调用 WindowContainer#finishSync 将所有参与同步 WindowContainer 的有关于 Transaction 改动，即 WindowContainer 的成员变量 mSyncTransaction 和 SyncGroup 的成员变量 mOrphanTransaction，合入到一个 Transaction 中，再将该 Transaction 传给 WindowOrganizerController，最终发送给客户端（这里就是 SyncCallback）：

SyncGroup#onSurfacePlacement

```
-> SyncGroup#finishNow 

	-> WindowOrganizerController#onTransactionReady 

		（跨进程）

		-> SyncCallback#onTransactionReady
```

### 2 WindowContainer 同步状态 mSyncState

有三种状态：SYNC_STATE_NONE、SYNC_STATE_WAITING_FOR_DRAW 和 SYNC_STATE_READY，看下都是在什么时候设置的。

1)、同步开始，所有非 WindowState 类型的 WindowContainer 的 mSyncState 会被设置为 SYNC_STATE_READY：

```
/**
     * Prepares this container for participation in a sync-group. This includes preparing all its
     * children.
     *
     * @return {@code true} if something changed (eg. this wasn't already in the sync group).
     */
    boolean prepareSync() {
        if (mSyncState != SYNC_STATE_NONE) {
            // Already part of sync
            return false;
        }
        for (int i = getChildCount() - 1; i >= 0; --i) {
            final WindowContainer child = getChildAt(i);
            child.prepareSync();
        }
        mSyncState = SYNC_STATE_READY;
        return true;
    }
```

WindowState 类型的容器 mSyncState 会被设置为 SYNC_STATE_WAITING_FOR_DRAW：

```
@Override
    boolean prepareSync() {
        if (!super.prepareSync()) {
            return false;
        }
        // In the WindowContainer implementation we immediately mark ready
        // since a generic WindowContainer only needs to wait for its
        // children to finish and is immediately ready from its own
        // perspective but at the WindowState level we need to wait for ourselves
        // to draw even if the children draw first our don't need to sync, so we start
        // in WAITING state rather than READY.
        mSyncState = SYNC_STATE_WAITING_FOR_DRAW;
        requestRedrawForSync();

        mWmService.mH.removeMessages(WINDOW_STATE_BLAST_SYNC_TIMEOUT, this);
        mWmService.mH.sendNewMessageDelayed(WINDOW_STATE_BLAST_SYNC_TIMEOUT, this,
            BLAST_TIMEOUT_DURATION);
        return true;
    }
```

2)、窗口绘制完成，对于 WindowState，mSyncState 被设置为 SYNC_STATE_READY：

```
/**
     * Call this when this container finishes drawing content.
     *
     * @return {@code true} if consumed (this container is part of a sync group).
     */
    boolean onSyncFinishedDrawing() {
        if (mSyncState == SYNC_STATE_NONE) return false;
        mSyncState = SYNC_STATE_READY;
        mWmService.mWindowPlacerLocked.requestTraversal();
        ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "onSyncFinishedDrawing %s", this);
        return true;
    }
```

3)、同步结束，所有 WindowContainer 的 mSyncState 被重置为 SYNC_STATE_NONE：

```
/**
     * Recursively finishes/cleans-up sync state of this subtree and collects all the sync
     * transactions into `outMergedTransaction`.
     * @param outMergedTransaction A transaction to merge all the recorded sync operations into.
     * @param cancel If true, this is being finished because it is leaving the sync group rather
     *               than due to the sync group completing.
     */
    void finishSync(Transaction outMergedTransaction, boolean cancel) {
        if (mSyncState == SYNC_STATE_NONE) return;
        ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "finishSync cancel=%b for %s", cancel, this);
        outMergedTransaction.merge(mSyncTransaction);
        for (int i = mChildren.size() - 1; i >= 0; --i) {
            mChildren.get(i).finishSync(outMergedTransaction, cancel);
        }
        mSyncState = SYNC_STATE_NONE;
        if (cancel && mSyncGroup != null) mSyncGroup.onCancelSync(this);
        mSyncGroup = null;
    }
```

### 3 同步完成的本质

要看一个 WindowContainer（一般是 Task）是否同步完成，其实是看它包含的 ActivityRecord 对应的窗口是否绘制完成：

ViewRootImpl#reportedDrawFinished

```
-> Session#finishDrawing

	（跨进程）

	-> WMS#finishDrawingWindow

		-> WindowState#finishDrawing

			-> WindowContainer#onSyncFinishedDrawing
```

这个依赖于 APP 端界面的绘制情况。
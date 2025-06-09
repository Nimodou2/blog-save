> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7072570725420236836)

> 分屏 Split-Screen WindowContainerTransaction SyncTransactionQueue--- highlight: androidstudio --- 一、W......

一、WindowOrganizer 发送 WCT 的两种方式
------------------------------

WindowContainerTransaction 通过 WindowOrganizer 来发送，WindowOrganizer 提供了两个方法 WindowOrganizer#applyTransaction 和 WindowOrganizer#applySyncTransaction。

```
@TestApi
public class WindowOrganizer {

    
    @RequiresPermission(android.Manifest.permission.MANAGE_ACTIVITY_TASKS)
    public void applyTransaction(@NonNull WindowContainerTransaction t) {
        try {
            if (!t.isEmpty()) {
                getWindowOrganizerController().applyTransaction(t);
            }
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    
    @RequiresPermission(android.Manifest.permission.MANAGE_ACTIVITY_TASKS)
    public int applySyncTransaction(@NonNull WindowContainerTransaction t,
            @NonNull WindowContainerTransactionCallback callback) {
        try {
            return getWindowOrganizerController().applySyncTransaction(t, callback.mInterface);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

	......

    @RequiresPermission(android.Manifest.permission.MANAGE_ACTIVITY_TASKS)
    IWindowOrganizerController getWindowOrganizerController() {
        return IWindowOrganizerControllerSingleton.get();
    }

    private static final Singleton<IWindowOrganizerController> IWindowOrganizerControllerSingleton =
            new Singleton<IWindowOrganizerController>() {
                @Override
                protected IWindowOrganizerController create() {
                    try {
                        return ActivityTaskManager.getService().getWindowOrganizerController();
                    } catch (RemoteException e) {
                        return null;
                    }
                }
            };
}


```

这两个方法，都是先取到一个 IWindowOrganizerController 对象，然后调用 IWindowOrganizerController 的相关接口。

IWindowOrganizerController 在服务端的实现是 WindowOrganizerController，因此最终 WindowContainerTransaction 跨进程发送到系统服务端的 WindowOrganizerController。

WindowOrganizer#applyTransaction 和 WindowOrganizer#applySyncTransaction 分别对应 WindowOrganizerController#applyTransaction 和 WindowOrganizerController#applySyncTransaction：

```
class WindowOrganizerController extends IWindowOrganizerController.Stub
    implements BLASTSyncEngine.TransactionReadyListener {
    ...... 
    
	@Override
    public void applyTransaction(WindowContainerTransaction t) {
        enforceTaskPermission("applyTransaction()");
        if (t == null) {
            throw new IllegalArgumentException("Null transaction passed to applySyncTransaction");
        }
        final long ident = Binder.clearCallingIdentity();
        try {
            synchronized (mGlobalLock) {
                applyTransaction(t, -1 , null );
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
                
                int syncId = -1;
                if (callback != null) {
                    syncId = startSyncWithOrganizer(callback);
                }
                applyTransaction(t, syncId, null );
                if (syncId >= 0) {
                    setSyncReady(syncId);
                }
                return syncId;
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
    
    ......
}


```

WindowOrganizerController 是应用 WindowContainerTransaction 的地方，以后会在其他笔记中分析。

从发送方式来看，可以把 WindowContainerTransaction 分为两种，同步的和异步的。

1)、异步的 WindowContainerTransaction 发送方式很简单，举一个例子，LegaycSplitScreenController#onSplitScreenSupported：

```
public void onSplitScreenSupported() {
    
    final WindowContainerTransaction tct = new WindowContainerTransaction();
    int midPos = mSplitLayout.getSnapAlgorithm().getMiddleTarget().position;
    mSplitLayout.resizeSplits(midPos, tct);
    mTaskOrganizer.applyTransaction(tct);
}


```

创建一个 WindowContainerTransaction 对象，设置好相关参数，直接调用 WindowOrganizer#applyTransaction 发送即可。

2)、同步 WindowContainerTransaction 的发送需要借助 SyncTransactionQueue，最终会调用 WindowOrganizer#applySyncTransaction，本文主要分析 SyncTransactionQueue 是如何发送同步 WindowContainerTransaction 的。

二、SyncTransactionQueue 介绍
-------------------------

```
public final class SyncTransactionQueue


```

SyncTransactionQueue，一个用于序列化同步 WindowContainerTransaction 和相应 callback 的助手类，有两个关键词，一个是序列化，一个是同步。

SyncTransactionQueue 也提供了两个方法来发送 WCT，SyncTransactionQueue#queue 和 SyncTransactionQueue#queueIfWaiting。

结合分屏，看下 SyncTransactionQueue 是如何工作的。

### 1 SyncTransactionQueue#queue

分屏的逻辑中，SyncTransactionQueue#queue 方法不是直接被调用的，而是间接通过 WindowManagerProxy 来调用，比如在 LegacySplitScreenController#splitPrimaryTask 中：

```
     final WindowContainerTransaction wct = new WindowContainerTransaction();
     
     wct.setWindowingMode(topRunningTask.token, WINDOWING_MODE_UNDEFINED);
     wct.reparent(topRunningTask.token, mSplits.mPrimary.token, true );
     mWindowManagerProxy.applySyncTransaction(wct);


```

我们创建了一个 WindowContainerTransaction 对象，然后设置了相关属性后，调用 WindowManagerProxy#applySyncTransaction：

```
    
    void applySyncTransaction(WindowContainerTransaction wct) {
        mSyncTransactionQueue.queue(wct);
    }


```

mSyncTransactionQueue 是 WindowManagerProxy 的 SyncTransactionQueue 类型的成员变量，那么这里会走到 SyncTransactionQueue#queue：

```
    
    public void queue(WindowContainerTransaction wct) {
        SyncCallback cb = new SyncCallback(wct);
        synchronized (mQueue) {
            if (DEBUG) Slog.d(TAG, "Queueing up " + wct);
            mQueue.add(cb);
            if (mQueue.size() == 1) {
                cb.send();
            }
        }
    }


```

注释上说，这个方法将一个要序列化发送到 WM 的同步 WindowContainerTransaction 进行排队。

内容很简单，基于传入的 WindowContainerTransaction 创建一个 SyncCallback 对象，并加到 mQueue 队列中，如果队列中只有它一个，那么执行 SyncCallback#send。

### 2 SyncTransactionQueue#queueIfWaiting

```
    
    public boolean queueIfWaiting(WindowContainerTransaction wct) {
        synchronized (mQueue) {
            if (mQueue.isEmpty()) {
                if (DEBUG) Slog.d(TAG, "Nothing in queue, so skip queueing up " + wct);
                return false;
            }
            if (DEBUG) Slog.d(TAG, "Queue is non-empty, so queueing up " + wct);
            SyncCallback cb = new SyncCallback(wct);
            mQueue.add(cb);
            if (mQueue.size() == 1) {
                cb.send();
            }
        }
        return true;
    }


```

和 SyncTransactionQueue#queue 的区别是，如果此时 mQueue 中没有正在等待的 SyncCallback，那么直接返回，不进行排队。

### 3 SyncCallback

上面看到，如果我们想要发送一个 WindowContainerTransaction，都是将 WindowContainerTransaction 封装到一个 SyncCallback 对象中，然后调用 SyncCallback#send，那么有必要看下 SyncCallback 的作用是什么。

```
    private class SyncCallback extends WindowContainerTransactionCallback {
        int mId = -1;
        final WindowContainerTransaction mWCT;

        SyncCallback(WindowContainerTransaction wct) {
            mWCT = wct;
        }

        
        void send() {
            if (mInFlight != null) {
                throw new IllegalStateException("Sync Transactions must be serialized. In Flight: "
                        + mInFlight.mId + " - " + mInFlight.mWCT);
            }
            mInFlight = this;
            if (DEBUG) Slog.d(TAG, "Sending sync transaction: " + mWCT);
            mId = new WindowOrganizer().applySyncTransaction(mWCT, this);
            if (DEBUG) Slog.d(TAG, " Sent sync transaction. Got id=" + mId);
            mMainExecutor.executeDelayed(mOnReplyTimeout, REPLY_TIMEOUT);
        }

        @BinderThread
        @Override
        public void onTransactionReady(int id,
                @NonNull SurfaceControl.Transaction t) {
            mMainExecutor.execute(() -> {
                synchronized (mQueue) {
                    if (mId != id) {
                        Slog.e(TAG, "Got an unexpected onTransactionReady. Expected "
                                + mId + " but got " + id);
                        return;
                    }
                    mInFlight = null;
                    mMainExecutor.removeCallbacks(mOnReplyTimeout);
                    if (DEBUG) Slog.d(TAG, "onTransactionReady id=" + mId);
                    mQueue.remove(this);
                    onTransactionReceived(t);
                    if (!mQueue.isEmpty()) {
                        mQueue.get(0).send();
                    }
                }
            });
        }
    }


```

SyncCallback 是一个内部类，创建的地方在 SyncTransactionQueue#queue 和 SyncTransactionQueue#queueIfWaiting，存储了一个要发给 WM 的 WindowContainerTransaction 对象。

#### 3.1 SyncCallback#send

```
        
        void send() {
            if (mInFlight != null) {
                throw new IllegalStateException("Sync Transactions must be serialized. In Flight: "
                        + mInFlight.mId + " - " + mInFlight.mWCT);
            }
            mInFlight = this;
            if (DEBUG) Slog.d(TAG, "Sending sync transaction: " + mWCT);
            mId = new WindowOrganizer().applySyncTransaction(mWCT, this);
            if (DEBUG) Slog.d(TAG, " Sent sync transaction. Got id=" + mId);
            mMainExecutor.executeDelayed(mOnReplyTimeout, REPLY_TIMEOUT);
        }


```

1)、调用 WindowOrganizer#applySyncTransaction 发送一个同步 callback 到 WM 端，由 WindowOrganizerController 处接收。最终从 WM 端会返回一个 syncId，这个 ID 保存在 SyncCallback 的成员变量 mId 中，和 SyncCallback 一一对应。

2)、将 mInFlight 指向这个被发送到 WM 的 callback。

3)、另外发送完之后开启一个防超时线程：

```
    private final Runnable mOnReplyTimeout = () -> {
        synchronized (mQueue) {
            if (mInFlight != null && mQueue.contains(mInFlight)) {
                Slog.w(TAG, "Sync Transaction timed-out: " + mInFlight.mWCT);
                mInFlight.onTransactionReady(mInFlight.mId, new SurfaceControl.Transaction());
            }
        }
    };


```

超时后调用 SyncCallback#onTransactionReady，防止出现堵塞的情况。

#### 3.2. SyncCallback#onTransactionReady

```
        @BinderThread
        @Override
        public void onTransactionReady(int id,
                @NonNull SurfaceControl.Transaction t) {
            mMainExecutor.execute(() -> {
                synchronized (mQueue) {
                    if (mId != id) {
                        Slog.e(TAG, "Got an unexpected onTransactionReady. Expected "
                                + mId + " but got " + id);
                        return;
                    }
                    mInFlight = null;
                    mMainExecutor.removeCallbacks(mOnReplyTimeout);
                    if (DEBUG) Slog.d(TAG, "onTransactionReady id=" + mId);
                    mQueue.remove(this);
                    onTransactionReceived(t);
                    if (!mQueue.isEmpty()) {
                        mQueue.get(0).send();
                    }
                }
            });
        }


```

SyncCallback#send 中通过 WindowOrganizer#applySyncTransaction 将 WindowContainerTransaction 发送给 WM 端后，WM 端就开始应用 WindowContainerTransaction，提取 WindowContainerTransaction 中的设置进行相应处理，这需要一段时间。等 WM 端处理结束，会由 WindowOrganizerController#onTransactionReady 回调 SyncCallback#onTransactionReady（具体情况可以参考 BLAST 机制介绍），并且返回一个集成了对多个 WindowContainer 操作的 Transaction 和一个 ID。

SyncCallback#onTransactionReady 接收到这个 Transaction 和 ID 之后，做了以下几件事：

1)、检查 ID 的一致性。

2)、将 mInFlight 置空。

3)、移除掉防延迟 mOnReplyTimeout。

4)、将该 SyncCallback 从 mQueue 中移除。

5)、调用 SyncTransactionQueue#onTransactionReceived 完成 Transaction 的 apply。

6)、如果 mQueue 中还有其他正在等待的 SyncCallback，调用他们的 send 方法。

### 4 SyncTransactionQueue#onTransactionReceived

```
private void onTransactionReceived(@NonNull SurfaceControl.Transaction t) {
    Slog.i("SyncTransactionQueue", "SyncTransactionQueue#onTransactionReceived");
    if (DEBUG) Slog.d(TAG, "  Running " + mRunnables.size() + " sync runnables");
    for (int i = 0, n = mRunnables.size(); i < n; ++i) {
        mRunnables.get(i).runWithTransaction(t);
    }
    mRunnables.clear();
    t.apply();
    t.close();
}


```

上一步提到了，当 WM 端完成了 WindowContainerTransaction 的应用后，会返回一个 Transaction 对象给 SyncCallback#onTransactionReady，然后 SyncCallback#onTransactionReady 再调用 SyncTransactionQueue#onTransactionReceived 完成 Transaction 的 apply。

SyncTransactionQueue#onTransactionReceive 做了两件事：

1)、执行之前由 SyncTransactionQueue#runInSync 中添加到 mRunnables 的 TransactionRunnable。

2)、调用 Transaction#apply。

注意，直到这里，从 WM 端传来的 Transaction 才被 apply，同步 WindowContainerTransaction 才算完成。

进退分屏的过程中，我们会通过 WindowContainerTransaction 为一些 Task 设置 Bounds，执行一些 Task 的 reparent 操作等，这些操作都会引起 Task 的 position、size 等属性的改变，这些改变最终会收集到一个 Transaction 对象中。但是由于整个过程是在 WindowOrganizerController#applySyncTransaction 的调用中进行的，因此收集了 Task 的 postion 和 size 等变化的 Transaction 不会马上 apply，而是等到所有相关的窗口绘制完成，然后全部 merge 到一个 Transaction 中，再由 WindowOrganizerController#onTransactionReady 发送到 SyncCallback#onTransactionReady 处，最终在 SyncTransactionQueue#onTransactionReceived 中完成 apply。

这一点很容易导致问题。

三、总结
----

1)、发送 WindowContainerTransaction 的方式有两种，异步的 WindowOrganizer#applyTransaction 和同步的 WindowOrganizer#applySyncTransaction。异步 WindowContainerTransacton 的发送很简单，直接拿到一个 WindowOrganizer 对象，调用其 applyTransaction 方法即可。同步 WindowContainerTransaction 的发送比较麻烦，不能直接调用 WindowOrganizer#applySyncTransaction 方法，需要借助 SyncTransactionQueue。

2)、SyncTransactionQueue#queue 和 SyncTransactionQueue#queueIfWaiting 是 SyncTransactionQueue 提供的两个用于发送同步 WindowContainerTransaction 的方法。

3)、通过 SyncTransactionQueue 发送的 WindowContainerTransaction 具有序列化和同步的特点。

4)、序列化表现在，在 SyncTransactionQueue 的参与下，所有的 WindowContainerTransaction 都会以序列化的方式发送：SyncTransactionQueue 内部有一个 SyncCallback 的队列 mQueue，要发送给 WM 端的 WindowContainerTransaction 会被封装为一个 SyncCallback 对象加入到 mQueue 中，然后按照 SyncCallback 入队的顺序调用 SyncCallback#send 方法进行发送，只有当前一个 WindowContainerTransaction 对应的 SyncCallback#onTransactionReady 回调完成，下一个 WindowContainerTransaction 对应的 SyncCallback#send 才会被调用。

5)、同步表现在，SyncTransactionQueue 作为 BLAST 机制的一部分实现了同步：WindowContainerTransaction 在 WM 端应用后，WM 端会将所有涉及到的 WindowContainer 的修改合入到一个 Transaction 中，等待所有涉及的窗口绘制完成，再回调 SyncCallback#onTransactionReady，最终在 SyncTransactionQueue#onTransactionReceived 中完成这个从 WM 端传过来的 Transaction 的 apply，直到这时，同步 WindowContainerTransaction 应用的整个过程才算结束。
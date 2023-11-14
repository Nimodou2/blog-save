> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/130754610?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/8a8cdae07c504cb991ba7ac37e8b8886.jpeg#pic_center)

#### 文章目录

*   [一、SurfaceControl 的初始化](#SurfaceControl_15)
*   *   [1 ViewRootImpl.relayoutWindow](#1_ViewRootImplrelayoutWindow_17)
    *   [2 Session.relayout](#2_Sessionrelayout_54)
    *   [3 WindowManagerService.relayoutWindow](#3_WindowManagerServicerelayoutWindow_79)
    *   [4 WindowManagerService.createSurfaceControl](#4_WindowManagerServicecreateSurfaceControl_107)
    *   [5 WindowStateAnimator.createSurfaceLocked](#5_WindowStateAnimatorcreateSurfaceLocked_146)
    *   [6 WindowSurfaceController.constructor](#6_WindowSurfaceControllerconstructor_182)
    *   [7 SurfaceControl.Builder.build](#7_SurfaceControlBuilderbuild_223)
    *   [8 Java 层 SurfaceControl.constructor](#8_JavaSurfaceControlconstructor_323)
    *   [9 android_view_SurfaceControl.nativeCreate](#9_android_view_SurfaceControlnativeCreate_396)
    *   [10 SurfaceComposerClient.createSurfaceChecked](#10_SurfaceComposerClientcreateSurfaceChecked_427)
    *   [11 Client.createSurface](#11_ClientcreateSurface_464)
    *   [12 SurfaceFlinger.createLayer](#12_SurfaceFlingercreateLayer_480)
    *   *   [12.1 Layer 的几种类型](#121_Layer_546)
        *   [12.2 BufferStateLayer 的创建](#122_BufferStateLayer_571)
        *   *   [12.2.1 Layer 向 SurfaceFlinger 进行注册](#1221_LayerSurfaceFlinger_592)
            *   [12.2.2 Layer 句柄的创建](#1222_Layer_629)
    *   [13 SurfaceFlinger.addClientLayer](#13_SurfaceFlingeraddClientLayer_683)
    *   *   [13.1 将 \<Handle, Layer> 对记录到 Client 中](#131_Handle_LayerClient_723)
        *   [13.2 将 \<Handle, Layer> 对记录到 SurfaceFlinger 中](#132_Handle_LayerSurfaceFlinger_767)
    *   [14 C++ 层 SurfaceControl.constructor](#14_CSurfaceControlconstructor_907)
    *   [15 WindowSurfaceController.getSurfaceControl](#15_WindowSurfaceControllergetSurfaceControl_968)
    *   [16 小结](#16__1038)
*   [二、BLASTBufferQueue 的创建](#BLASTBufferQueue_1049)
*   *   [1 ViewRootImpl.getOrCreateBLASTSurface](#1_ViewRootImplgetOrCreateBLASTSurface_1109)
    *   [2 BLASTBufferQueue.constructor](#2_BLASTBufferQueueconstructor_1139)
    *   [3 android_graphics_BLASTBufferQueue.nativeCreate](#3_android_graphics_BLASTBufferQueuenativeCreate_1151)
    *   [4 BLASTBufferQueue.constructor](#4_BLASTBufferQueueconstructor_1176)
    *   *   [4.1 BLASTBufferQueue.createBufferQueue](#41_BLASTBufferQueuecreateBufferQueue_1233)
        *   *   [4.1.1 BufferQueueCore](#411_BufferQueueCore_1266)
            *   *   [4.1.1.1 BufferQueueCore 的继承关系](#4111_BufferQueueCore_1274)
                *   [4.1.1.2 BufferQueueCore 的构造函数](#4112_BufferQueueCore_1284)
            *   [4.1.2 BBQBufferQueueProducer](#412_BBQBufferQueueProducer_1319)
            *   *   [4.1.2.1 BBQBufferQueueProducer 的继承关系](#4121_BBQBufferQueueProducer_1327)
                *   [4.1.2.2 BBQBufferQueueProducer 的构造函数](#4122_BBQBufferQueueProducer_1383)
            *   [4.1.3 BufferQueueConsumer](#413_BufferQueueConsumer_1412)
            *   *   [4.1.3.1 BufferQueueConsumer 的继承关系](#4131_BufferQueueConsumer_1420)
                *   [4.1.3.2 BufferQueueConsumer 的构造函数](#4132_BufferQueueConsumer_1436)
        *   [4.2 BLASTBufferItemConsumer](#42_BLASTBufferItemConsumer_1447)
        *   *   [4.2.1 BufferItemConsumer](#421_BufferItemConsumer_1473)
            *   [4.2.2 ConsumerBase](#422_ConsumerBase_1516)
    *   [5 小结](#5__1598)
*   [三、Surface 的初始化](#Surface_1605)
*   *   [1 BLASTBufferQueue.createSurface](#1_BLASTBufferQueuecreateSurface_1635)
    *   [2 android_graphics_BLASTBufferQueue.nativeGetSurface](#2_android_graphics_BLASTBufferQueuenativeGetSurface_1648)
    *   [3 BLASTBufferQueue.getSurface](#3_BLASTBufferQueuegetSurface_1663)
    *   [4 BBQSurface](#4_BBQSurface_1680)
    *   [5 Surface](#5_Surface_1696)
    *   [6 android_view_Surface.android_view_Surface_createFromSurface](#6_android_view_Surfaceandroid_view_Surface_createFromSurface_1812)
    *   [7 Surface.transferFrom](#7_SurfacetransferFrom_1850)
    *   [8 小结](#8__1921)

ViewRootImpl 内部有两个成员变量：

```
// These can be accessed by any thread, must be protected with a lock.
    // Surface can never be reassigned or cleared (use Surface.clear()).
    @UnsupportedAppUsage
    public final Surface mSurface = new Surface();
    private final SurfaceControl mSurfaceControl = new SurfaceControl();
```

跟踪一下向 mSurface 和 mSurfaceControl 初始化的流程。

一、SurfaceControl 的初始化
---------------------

### 1 ViewRootImpl.relayoutWindow

```
private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {
        // ......

        int relayoutResult = mWindowSession.relayout(mWindow, params,
                (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                (int) (mView.getMeasuredHeight() * appScale + 0.5f), viewVisibility,
                insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
                mTmpFrames, mPendingMergedConfiguration, mSurfaceControl, mTempInsets,
                mTempControls, mSurfaceSize);
        mPendingBackDropFrame.set(mTmpFrames.backdropFrame);
        if (mSurfaceControl.isValid()) {
            if (!useBLAST()) {
                mSurface.copyFrom(mSurfaceControl);
            } else {
                final Surface blastSurface = getOrCreateBLASTSurface();
                // If blastSurface == null that means it hasn't changed since the last time we
                // called. In this situation, avoid calling transferFrom as we would then
                // inc the generation ID and cause EGL resources to be recreated.
                if (blastSurface != null) {p n
                    mSurface.transferFrom(blastSurface);
                }
            }
			// ......
        } else {
            destroySurface();
        }

		// ......
    }
```

mWindowSession 是 IWindowSession 的 [Binder](https://so.csdn.net/so/search?q=Binder&spm=1001.2101.3001.7020) 远程代理对象，服务端的实现是 Session，那么成员变量 mSurfaceControl 是通过 Binder IPC，在系统进程中加载其中的内容的。

### 2 Session.relayout

```
@Override
    public int relayout(IWindow window, WindowManager.LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewFlags, int flags, long frameNumber,
            ClientWindowFrames outFrames, MergedConfiguration mergedConfiguration,
            SurfaceControl outSurfaceControl, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls, Point outSurfaceSize) {
        if (false) Slog.d(TAG_WM, ">>>>>> ENTERED relayout from "
                + Binder.getCallingPid());
        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, mRelayoutTag);
        int res = mService.relayoutWindow(this, window, attrs,
                requestedWidth, requestedHeight, viewFlags, flags, frameNumber,
                outFrames, mergedConfiguration, outSurfaceControl, outInsetsState,
                outActiveControls, outSurfaceSize);
        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        if (false) Slog.d(TAG_WM, "<<<<<< EXITING relayout to "
                + Binder.getCallingPid());
        return res;
    }
```

这里继续调用了 WindowManagerService.relayoutWindow。

### 3 WindowManagerService.relayoutWindow

```
public int relayoutWindow(Session session, IWindow client, LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility, int flags,
            long frameNumber, ClientWindowFrames outFrames, MergedConfiguration mergedConfiguration,
            SurfaceControl outSurfaceControl, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls, Point outSurfaceSize) {
		// ......
        synchronized (mGlobalLock) {
            // ......
            
            // Create surfaceControl before surface placement otherwise layout will be skipped
            // (because WS.isGoneForLayout() is true when there is no surface.
            if (shouldRelayout) {
                try {
                    result = createSurfaceControl(outSurfaceControl, result, win, winAnimator);
                }
                // ......
            }
            
            // ......
        }
    }
```

只关注 outSurfaceControl 的数据是如何填充的，这里继续调用 WindowManagerService.createSurfaceControl 方法。

### 4 WindowManagerService.createSurfaceControl

```
private int createSurfaceControl(SurfaceControl outSurfaceControl, int result,
            WindowState win, WindowStateAnimator winAnimator) {
        if (!win.mHasSurface) {
            result |= RELAYOUT_RES_SURFACE_CHANGED;
        }

        WindowSurfaceController surfaceController;
        try {
            Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "createSurfaceControl");
            surfaceController = winAnimator.createSurfaceLocked(win.mAttrs.type);
        } finally {
            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        }
        if (surfaceController != null) {
            surfaceController.getSurfaceControl(outSurfaceControl);
            ProtoLog.i(WM_SHOW_TRANSACTIONS, "OUT SURFACE %s: copied", outSurfaceControl);

        } else {
            // For some reason there isn't a surface.  Clear the
            // caller's object so they see the same state.
            ProtoLog.w(WM_ERROR, "Failed to create surface control for %s", win);
            outSurfaceControl.release();
        }

        return result;
    }
```

有两个流程：

*   调用 WindowStateAnimator.createSurfaceLocked 方法来创建一个 WindowSurfaceController 对象。
    
*   调用 WindowSurfaceController.getSurfaceControl 来对 outSurfaceControl 赋值。
    

流程 1 涉及到 Native 层 SurfaceControl 的创建，需要跟踪完流程 1 才能知道流程 2 中的 outSurfaceControl 是怎么得到数据的。

### 5 WindowStateAnimator.createSurfaceLocked

```
WindowSurfaceController createSurfaceLocked(int windowType) {
		// ......
        
        int flags = SurfaceControl.HIDDEN;
        final WindowManager.LayoutParams attrs = w.mAttrs;

        if (w.isSecureLocked()) {
            flags |= SurfaceControl.SECURE;
        }

        if ((mWin.mAttrs.privateFlags & PRIVATE_FLAG_IS_ROUNDED_CORNERS_OVERLAY) != 0) {
            flags |= SurfaceControl.SKIP_SCREENSHOT;
        }
        
        // ......

        // Set up surface control with initial size.
        try {
			// ......

            mSurfaceController = new WindowSurfaceController(attrs.getTitle().toString(), width,
                    height, format, flags, this, windowType);
			// ......
        }

		// ......

        return mSurfaceController;
    }
```

忽略非相关的内容，这里根据一些窗口的基本信息，如窗口名字、窗口宽高、窗口类型等，来创建一个 WindowSurfaceController 对象，并且赋值给 WindowStateAnimator 的 mSurfaceController 成员变量。

### 6 WindowSurfaceController.constructor

```
WindowSurfaceController(String name, int w, int h, int format,
            int flags, WindowStateAnimator animator, int windowType) {
        mAnimator = animator;

        title = name;

        mService = animator.mService;
        final WindowState win = animator.mWin;
        mWindowType = windowType;
        mWindowSession = win.mSession;

        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "new SurfaceControl");
        final SurfaceControl.Builder b = win.makeSurface()
                .setParent(win.getSurfaceControl())
                .setName(name)
                .setBufferSize(w, h)
                .setFormat(format)
                .setFlags(flags)
                .setMetadata(METADATA_WINDOW_TYPE, windowType)
                .setMetadata(METADATA_OWNER_UID, mWindowSession.mUid)
                .setMetadata(METADATA_OWNER_PID, mWindowSession.mPid)
                .setCallsite("WindowSurfaceController");

        final boolean useBLAST = mService.mUseBLAST && ((win.getAttrs().privateFlags
                & WindowManager.LayoutParams.PRIVATE_FLAG_USE_BLAST) != 0);

        if (useBLAST) {
            b.setBLASTLayer();
        }

        mSurfaceControl = b.build();

        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
    }
```

这里先调用 WindowState.makeSurface 方法生成了一个 SurfaceControl.Builder 对象，然后提前设置了一些 SurfaceControl.Builder 的属性，这些属性最终是要应用给 SurfaceControl 的。接着调用 SurfaceControl.Builder.build 方法来生成一个 SurfaceControl 对象，并且赋值给 WindowSurfaceController 的成员变量 mSurfaceControl。

### 7 SurfaceControl.Builder.build

```
/**
         * Construct a new {@link SurfaceControl} with the set parameters. The builder
         * remains valid.
         */
        @NonNull
        public SurfaceControl build() {
            if (mWidth < 0 || mHeight < 0) {
                throw new IllegalStateException(
                        "width and height must be positive or unset");
            }
            if ((mWidth > 0 || mHeight > 0) && (isEffectLayer() || isContainerLayer())) {
                throw new IllegalStateException(
                        "Only buffer layers can set a valid buffer size.");
            }

            if ((mFlags & FX_SURFACE_MASK) == FX_SURFACE_NORMAL) {
                setBLASTLayer();
            }

            return new SurfaceControl(
                    mSession, mName, mWidth, mHeight, mFormat, mFlags, mParent, mMetadata,
                    mLocalOwnerView, mCallsite);
        }
```

先对这个即将创建的新的 SurfaceControl 的相关属性进行有效性检查，接着基于之前传入 SurfaceControl.Builder 的相关信息创建一个 SurfaceControl 对象。

另外在创建这里看到有一个设置 flag 的操作：

```
if ((mFlags & FX_SURFACE_MASK) == FX_SURFACE_NORMAL) {
                setBLASTLayer();
            }
```

首先先看一下 FX_SURFACE_MASK 的相关信息，FX_SURFACE_MASK 这一位的相关 flag 的定义如下：

```
/**
     * Surface creation flag: Creates a normal surface.
     * This is the default.
     *
     * @hide
     */
    public static final int FX_SURFACE_NORMAL   = 0x00000000;

    /**
     * Surface creation flag: Creates a effect surface which
     * represents a solid color and or shadows.
     *
     * @hide
     */
    public static final int FX_SURFACE_EFFECT = 0x00020000;

    /**
     * Surface creation flag: Creates a container surface.
     * This surface will have no buffers and will only be used
     * as a container for other surfaces, or for its InputInfo.
     * @hide
     */
    public static final int FX_SURFACE_CONTAINER = 0x00080000;

    /**
     * @hide
     */
    public static final int FX_SURFACE_BLAST = 0x00040000;

    /**
     * Mask used for FX values above.
     *
     * @hide
     */
    public static final int FX_SURFACE_MASK = 0x000F0000;
```

这里看到 FX_SURFACE_MASK 代表的这一位把 [Surface](https://so.csdn.net/so/search?q=Surface&spm=1001.2101.3001.7020) 分成了几种类型：

*   FX_SURFACE_NORMAL，代表了一个标准 Surface，这个是默认设置。
*   FX_SURFACE_EFFECT，代表了一个有纯色或者阴影效果的 Surface。
*   FX_SURFACE_CONTAINER，代表了一个容器类 Surface，这种 Surface 没有缓冲区，只是用来作为其他 Surface 的容器，或者是它自己的 InputInfo 的容器。
*   FX_SURFACE_BLAST，结合这里的分析可以看到，FX_SURFACE_BLAST 应该是等同于 FX_SURFACE_NORMAL。

如果 SurfaceControl.Builder 的 mFlags 中的 FX_SURFACE_MASK 这一位是 FX_SURFACE_NORMAL，那么调用 SurfaceControl.setBLASTLayer 方法：

```
/**
         * @hide
         */
        public Builder setBLASTLayer() {
            return setFlags(FX_SURFACE_BLAST, FX_SURFACE_MASK);
        }
```

将 FX_SURFACE_MASK 代表的这一位设置为 FX_SURFACE_BLAST。

SurfaceControl.Builder 的 mFlags 的值是从 WindowStateAnimator.createSurfaceLocked 中获取的，WindowStateAnimator.createSurfaceLocked 方法中没有针对 FX_SURFACE_MASK 这一位进行特殊处理，那么也就是说，这里会调用 setBLASTLayer 方法将 mFlags 的 FX_SURFACE_MASK 代表的这一位设置为 FX_SURFACE_BLAST。

### 8 Java 层 SurfaceControl.constructor

```
/**
     * Create a surface with a name.
     * <p>
     * The surface creation flags specify what kind of surface to create and
     * certain options such as whether the surface can be assumed to be opaque
     * and whether it should be initially hidden.  Surfaces should always be
     * created with the {@link #HIDDEN} flag set to ensure that they are not
     * made visible prematurely before all of the surface's properties have been
     * configured.
     * <p>
     * Good practice is to first create the surface with the {@link #HIDDEN} flag
     * specified, open a transaction, set the surface layer, layer stack, alpha,
     * and position, call {@link Transaction#show(SurfaceControl)} if appropriate, and close the
     * transaction.
     * <p>
     * Bounds of the surface is determined by its crop and its buffer size. If the
     * surface has no buffer or crop, the surface is boundless and only constrained
     * by the size of its parent bounds.
     *
     * @param session  The surface session, must not be null.
     * @param name     The surface name, must not be null.
     * @param w        The surface initial width.
     * @param h        The surface initial height.
     * @param flags    The surface creation flags.
     * @param metadata Initial metadata.
     * @param callsite String uniquely identifying callsite that created this object. Used for
     *                 leakage tracking.
     * @throws throws OutOfResourcesException If the SurfaceControl cannot be created.
     */
    private SurfaceControl(SurfaceSession session, String name, int w, int h, int format, int flags,
            SurfaceControl parent, SparseIntArray metadata, WeakReference<View> localOwnerView,
            String callsite)
                    throws OutOfResourcesException, IllegalArgumentException {
        if (name == null) {
            throw new IllegalArgumentException("name must not be null");
        }

        mName = name;
        mWidth = w;
        mHeight = h;
        mLocalOwnerView = localOwnerView;
        Parcel metaParcel = Parcel.obtain();
        try {
			// ......
            mNativeObject = nativeCreate(session, name, w, h, format, flags,
                    parent != null ? parent.mNativeObject : 0, metaParcel);
        } finally {
            metaParcel.recycle();
        }
        if (mNativeObject == 0) {
            throw new OutOfResourcesException(
                    "Couldn't allocate SurfaceControl native object");
        }
        mNativeHandle = nativeGetHandle(mNativeObject);
        mCloseGuard.openWithCallSite("release", callsite);
    }
```

渣翻：

这个方法用来生成一个有名字的 Surface。

Surface 创建标志明确了要创建哪种 Surface 和特定的选项，比如 Surface 可以被认为是不透明的，是否应该在初始的时候隐藏。Surface 应该总是在创建的时候带着 HIDDEN 标志，以确保在 Surface 的属性配置好之前，Surface 不会过早显示。

一个好的实践是，首先使用 HIDDEN 标志创建 Surface，接着打开一个 Transactionn，设置 Surface 的 layer、layer 堆栈、透明度、位置，然后再合适的时机调用 Transaction.show，最后关闭 Transactoin。

Surface 的界限由它的 crop 和 buffer 尺寸决定。如果这个 Surface 没有 buffer 或者是 crop，那么这个 Surface 就是无边界的，只被它的父 Surface 的界限尺寸所限制。

这里调用了 nativeCreate 来创建 native 层的 SurfaceControl。

### 9 android_view_SurfaceControl.nativeCreate

```
static jlong nativeCreate(JNIEnv* env, jclass clazz, jobject sessionObj,
        jstring nameStr, jint w, jint h, jint format, jint flags, jlong parentObject,
        jobject metadataParcel) {
    ScopedUtfChars name(env, nameStr);
    sp<SurfaceComposerClient> client;
    if (sessionObj != NULL) {
        client = android_view_SurfaceSession_getClient(env, sessionObj);
    } else {
        client = SurfaceComposerClient::getDefault();
    }
    SurfaceControl *parent = reinterpret_cast<SurfaceControl*>(parentObject);
    sp<SurfaceControl> surface;
	// ......

    status_t err = client->createSurfaceChecked(String8(name.c_str()), w, h, format, &surface,
                                                flags, parentHandle, std::move(metadata));
	
    // ......

    surface->incStrong((void *)nativeCreate);
    return reinterpret_cast<jlong>(surface.get());
}
```

这里的 sessionObj 是一个 Java 层的 SurfaceSession 对象，之前分析 App 到 SurfaceFlinger 的连接时候有分析到，App 在第一次添加窗口的时候，会创建一个 SurfaceSession 对象，然后在 JNI 层的 SurfaceSession 对象创建的时候，会创建一个 SurfaceComposerClient 对象来连接到 SurfaceFlinger。

那么这里先根据传入的 sessionObj 对象获取到一个 SurfaceComposerClient 对象，然后调用 SurfaceComposerClient.createSurfaceChecked 生成一个 Native 层的 SurfaceControl 对象，最后调用 incStrong 将 SurfaceControl 的强引用计数增加 1，这会导致 SurfaceControl 的 onFirstRef 函数被调用，最后将这个创建的 Native 层的 SurfaceControl 的地址转换成一个整型返回给上层的 SurfaceControl 对象。

### 10 SurfaceComposerClient.createSurfaceChecked

```
status_t SurfaceComposerClient::createSurfaceChecked(const String8& name, uint32_t w, uint32_t h,
                                                     PixelFormat format,
                                                     sp<SurfaceControl>* outSurface, uint32_t flags,
                                                     const sp<IBinder>& parentHandle,
                                                     LayerMetadata metadata,
                                                     uint32_t* outTransformHint) {
    sp<SurfaceControl> sur;
    status_t err = mStatus;

    if (mStatus == NO_ERROR) {
        sp<IBinder> handle;
        sp<IGraphicBufferProducer> gbp;

        uint32_t transformHint = 0;
        int32_t id = -1;
        err = mClient->createSurface(name, w, h, format, flags, parentHandle, std::move(metadata),
                                     &handle, &gbp, &id, &transformHint);
        if (outTransformHint) {
            *outTransformHint = transformHint;
        }
        ALOGE_IF(err, "SurfaceComposerClient::createSurface error %s", strerror(-err));
        if (err == NO_ERROR) {
            *outSurface =
                    new SurfaceControl(this, handle, gbp, id, w, h, format, transformHint, flags);
        }
    }
    return err;
}
```

1）、根据之前分析 App 到 SurfaceFlinger 的连接，知道 SurfaceComposerClient 的成员变量 mClient 是一个 BpSurfaceComposerClient 类型的 Binder 代理对象，它的服务端实现是在 SurfaceFlinger 服务的 Client 类，那么这里最终是调用了服务端 Client.createSurface 函数。

2）、基于从服务端返回的信息，创建一个 surfaceControl 对象，这部分要等分析完服务端的内容后才能继续分析。

### 11 Client.createSurface

```
status_t Client::createSurface(const String8& name, uint32_t w, uint32_t h, PixelFormat format,
                               uint32_t flags, const sp<IBinder>& parentHandle,
                               LayerMetadata metadata, sp<IBinder>* handle,
                               sp<IGraphicBufferProducer>* gbp, int32_t* outLayerId,
                               uint32_t* outTransformHint) {
    // We rely on createLayer to check permissions.
    return mFlinger->createLayer(name, this, w, h, format, flags, std::move(metadata), handle, gbp,
                                 parentHandle, outLayerId, nullptr, outTransformHint);
}
```

已知在 Client 内部是通过一个 sp<SurfaceFlinger> 类型的成员变量 mFlinger 是一个 SurfaceFlinger 类型的强指针，那么这里调用的即是 SurfaceFlinger.createLayer。

### 12 SurfaceFlinger.createLayer

```
status_t SurfaceFlinger::createLayer(const String8& name, const sp<Client>& client, uint32_t w,
                                     uint32_t h, PixelFormat format, uint32_t flags,
                                     LayerMetadata metadata, sp<IBinder>* handle,
                                     sp<IGraphicBufferProducer>* gbp,
                                     const sp<IBinder>& parentHandle, int32_t* outLayerId,
                                     const sp<Layer>& parentLayer, uint32_t* outTransformHint) {
	// ......

    status_t result = NO_ERROR;

    sp<Layer> layer;

    std::string uniqueName = getUniqueLayerName(name.string());

    switch (flags & ISurfaceComposerClient::eFXSurfaceMask) {
        case ISurfaceComposerClient::eFXSurfaceBufferQueue:
        case ISurfaceComposerClient::eFXSurfaceBufferState: {
            result = createBufferStateLayer(client, std::move(uniqueName), w, h, flags,
                                            std::move(metadata), handle, &layer);
			// ......
        } break;
        case ISurfaceComposerClient::eFXSurfaceEffect:
			// ......

            result = createEffectLayer(client, std::move(uniqueName), w, h, flags,
                                       std::move(metadata), handle, &layer);
            break;
        case ISurfaceComposerClient::eFXSurfaceContainer:
			// ......	
            result = createContainerLayer(client, std::move(uniqueName), w, h, flags,
                                          std::move(metadata), handle, &layer);
            break;
        default:
            result = BAD_VALUE;
            break;
    }

    if (result != NO_ERROR) {
        return result;
    }

    bool addToRoot = callingThreadHasUnscopedSurfaceFlingerAccess();
    result = addClientLayer(client, *handle, *gbp, layer, parentHandle, parentLayer, addToRoot,
                            outTransformHint);
    if (result != NO_ERROR) {
        return result;
    }
    mInterceptor->saveSurfaceCreation(layer);

    setTransactionFlags(eTransactionNeeded);
    *outLayerId = layer->sequence;
    return result;
}
```

这里的 getUniqueLayerName 函数保证每一个 Layer 都有一个独一无二的名字，名字的格式为：

```
uniqueName = base::StringPrintf("%s#%u", name, ++dupeCounter);
```

保证了即使是同一个 Activity 的两个窗口，也能根据 “#0”，“#1” 等序号进行区分。

#### 12.1 Layer 的几种类型

这里根据传入的 flags 参数的 ISurfaceComposerClient::eFXSurfaceMask 这一位去创建不同类型的 Layer 对象。这里的 ISurfaceComposerClient 中的定义的：

```
eFXSurfaceBufferQueue = 0x00000000,
        eFXSurfaceEffect = 0x00020000,
        eFXSurfaceBufferState = 0x00040000,
        eFXSurfaceContainer = 0x00080000,
        eFXSurfaceMask = 0x000F0000,
```

这些 flag 和我们在分析 SurfaceControl.Builder.build 方法的内容的时候介绍的 flag 是一一对应的，这里看到：

*   eFXSurfaceBufferQueue 和 eFXSurfaceBufferState，对应 BufferStateLayer。
*   eFXSurfaceEffect，对应 EffectLayer。
*   eFXSurfaceContainer，对应 ContainerLayer。

还有一种 BufferQueueLayer，但是到 Android 12 这种 BufferQueueLayer 的相关代码虽然还保留，但是没有创建 BufferQueueLayer 的地方了。

上一张简单的类图看下这些 Layer 的关系：

![](https://img-blog.csdnimg.cn/689917add3dc4838b12d8ae788623b87.png#pic_center)

#### 12.2 BufferStateLayer 的创建

根据之前的分析，我们这里要创建的是 BufferStateLayer 类型的 Layer，那么走的是 SurfaceFlinger.createBufferStateLayer：

```
status_t SurfaceFlinger::createBufferStateLayer(const sp<Client>& client, std::string name,
                                                uint32_t w, uint32_t h, uint32_t flags,
                                                LayerMetadata metadata, sp<IBinder>* handle,
                                                sp<Layer>* outLayer) {
    LayerCreationArgs args(this, client, std::move(name), w, h, flags, std::move(metadata));
    args.textureName = getNewTexture();
    sp<BufferStateLayer> layer = getFactory().createBufferStateLayer(args);
    *handle = layer->getHandle();
    *outLayer = layer;

    return NO_ERROR;
}
```

SurfaceFlinger 创建 createBufferStateLayer 的流程比较简单，所以就不详细分析了，这里看下我觉得需要注意的几个点。

##### 12.2.1 Layer 向 SurfaceFlinger 进行注册

SurfaceFlinger.createBufferStateLayer 中，在我们创建了一个 BufferStateLayer 对象后，将一个 BufferStateLayer 类型的强指针指向这个对象，

```
sp<BufferStateLayer> layer = getFactory().createBufferStateLayer(args);
```

后续在 Layer 的 onFirstRef 函数中：

```
void Layer::onFirstRef() {
    mFlinger->onLayerFirstRef(this);
}
```

调用了 SurfaceFlinger.onLayerFirstRef：

```
void SurfaceFlinger::onLayerFirstRef(Layer* layer) {
    mNumLayers++;
    if (!layer->isRemovedFromCurrentState()) {
        mScheduler->registerLayer(layer);
    }
}
```

这里看到 SurfaceFlinger 内部有一个成员变量

```
std::atomic<size_t> mNumLayers = 0;
```

负责对所有创建的 Layer 进行计数。

如果当前 Layer 有父 Layer，那么继续调用 Scheduler.registerLayer 函数对新创建的 Layer 进行记录。

##### 12.2.2 Layer 句柄的创建

SurfaceFlinger.createBufferStateLayer 在创建了 BufferStateLayer 对象后，这里又调用了 Layer.getHandle 函数：

```
*handle = layer->getHandle();
```

Layer.getHandle 函数定义是：

```
// Creates a new handle each time, so we only expect
    // this to be called once.
    sp<IBinder> getHandle();
```

Layer.getHandle 函数只被期望在 Layer 创建的时候调用一次：

```
sp<IBinder> Layer::getHandle() {
    Mutex::Autolock _l(mLock);
    if (mGetHandleCalled) {
        ALOGE("Get handle called twice" );
        return nullptr;
    }
    mGetHandleCalled = true;
    return new Handle(mFlinger, this);
}
```

Handle 类的定义如下：

```
/*
     * The layer handle is just a BBinder object passed to the client
     * (remote process) -- we don't keep any reference on our side such that
     * the dtor is called when the remote side let go of its reference.
     *
     * LayerCleaner ensures that mFlinger->onLayerDestroyed() is called for
     * this layer when the handle is destroyed.
     */
    class Handle : public BBinder, public LayerCleaner {
    public:
        Handle(const sp<SurfaceFlinger>& flinger, const sp<Layer>& layer)
              : LayerCleaner(flinger, layer), owner(layer) {}

        wp<Layer> owner;
    };
```

Layer 句柄只是一个传递给客户端进程的 BBinder 对象，我们不在本地保存任何引用，以便客户端在释放它的引用的时候析构函数可以被调用。最终这个句柄是要传给客户端，也就是 SurfaceComposerClient 处，然后 SurfaceComposerClient 基于这个 Layer 句柄创建一个 SurfaceControl 对象。那么我暂时像理解 token 一样，理解 Layer 句柄的作用为跨进程标识一个唯一的 Layer。

LayerCleaner 保证了当这个句柄销毁的时候 SurfaceFlinger.onLayerDestroyed 函数可以被调用，SurfaceFlinger.onLayerDestroyed 函数即和我们上面讲的 Layer 创建的时候调用 SurfaceFlinger.onLayerFirstRef 这个函数相对。

### 13 SurfaceFlinger.addClientLayer

SurfaceFlinger.createLayer 函数在创建了 Layer 后，继续调用了 SurfaceFlinger.addClientLayer。

```
status_t SurfaceFlinger::addClientLayer(const sp<Client>& client, const sp<IBinder>& handle,
                                        const sp<IGraphicBufferProducer>& gbc, const sp<Layer>& lbc,
                                        const sp<IBinder>& parentHandle,
                                        const sp<Layer>& parentLayer, bool addToRoot,
                                        uint32_t* outTransformHint) {
	// ......

    // Create a transaction includes the initial parent and producer.
    Vector<ComposerState> states;
    Vector<DisplayState> displays;

    ComposerState composerState;
    composerState.state.what = layer_state_t::eLayerCreated;
    composerState.state.surface = handle;
    states.add(composerState);

    // ......
    
    // attach this layer to the client
    client->attachLayer(handle, lbc);

    return setTransactionState(FrameTimelineInfo{}, states, displays, 0 /* flags */, nullptr,
                               InputWindowCommands{}, -1 /* desiredPresentTime */,
                               true /* isAutoTimestamp */, {}, false /* hasListenerCallbacks */, {},
                               0 /* Undefined transactionId */);
}
```

目前看到这个函数有两个作用：

*   将 <Handle, Layer> 对记录到 Client 中
*   将 <Handle, Layer> 对记录到 SurfaceFlinger 中

下面逐个分析。

#### 13.1 将 <Handle, Layer> 对记录到 Client 中

这一步对应的是 SurfaceFlinger.addClientLayer 的如下内容：

```
// attach this layer to the client
    client->attachLayer(handle, lbc);
```

Client.attachLayer 函数很简单：

```
void Client::attachLayer(const sp<IBinder>& handle, const sp<Layer>& layer)
{
    Mutex::Autolock _l(mLock);
    mLayers.add(handle, layer);
}
```

这里通过 Client 的成员变量 mLayers：

```
// protected by mLock
    DefaultKeyedVector< wp<IBinder>, wp<Layer> > mLayers;
```

将新创建的 Layer 和相应的 Layer 句柄保存在 Client 中，这样，每一个 Client 都维护了一个属于自己的 Layer 的表。

Layer 句柄在 SurfaceFlinger 创建完 Layer 后，会返回给 SurfaceComposerClient。那么后续 SurfaceComposerClient 发送 Layer 句柄到 Client 的时候，Client 就可以通过该 Layer 句柄找到相应的 Layer，就像下面的代码一样：

```
sp<Layer> Client::getLayerUser(const sp<IBinder>& handle) const
{
    Mutex::Autolock _l(mLock);
    sp<Layer> lbc;
    wp<Layer> layer(mLayers.valueFor(handle));
    if (layer != 0) {
        lbc = layer.promote();
        ALOGE_IF(lbc==0, "getLayerUser(, handle.get());
    }
    return lbc;
}
```

#### 13.2 将 <Handle, Layer> 对记录到 SurfaceFlinger 中

SurfaceFlinger.addClientLayer 还有一部分内容没讲到：

```
// Create a transaction includes the initial parent and producer.
    Vector<ComposerState> states;
    Vector<DisplayState> displays;

    ComposerState composerState;
    composerState.state.what = layer_state_t::eLayerCreated;
    composerState.state.surface = handle;
    states.add(composerState);

	// ......

    return setTransactionState(FrameTimelineInfo{}, states, displays, 0 /* flags */, nullptr,
                               InputWindowCommands{}, -1 /* desiredPresentTime */,
                               true /* isAutoTimestamp */, {}, false /* hasListenerCallbacks */, {},
                               0 /* Undefined transactionId */);
```

这里首先初始化了一个 ComposerState 类型和 DisplayState 类型的 Vector，接着创建了一个 ComposerState 类型的对象。

ComposerState 和 DisplayState 这两个类型都定义在 LayerState.h 中：

```
struct ComposerState {
    layer_state_t state;
    status_t write(Parcel& output) const;
    status_t read(const Parcel& input);
};

struct DisplayState {
	// ......

    status_t write(Parcel& output) const;
    status_t read(const Parcel& input);
};
```

这里主要关注 ComposerState 类型。

ComposerState 结构体中又有一个 layer_state_t 类型的成员：

```
/*
 * Used to communicate layer information between SurfaceFlinger and its clients.
 */
struct layer_state_t {
    // ......
    sp<IBinder> surface;
    // ......
    uint64_t what;
    // ......
}
```

layer_state_t 用来在 SurfaceFlinger 和它的客户端，也就是 SurfaceComposerClient，交流 Layer 信息。

这里主要做了以下几件事：

1）、设置了该 ComposerState 对象中的 layer_state_t 对象的 what 成员为 layer_state_t::eLayerCreated。

2）、设置了该 ComposerState 对象中的 layer_state_t 对象的 surface 成员为 handle，那么这个 layer_state_t 里就保存了 Layer 的句柄。

3）、然后把这个创建的 ComposerState 添加到上面的 ComposerState 向量中。

4）、最后调用 SurfaceFlinger.setTransactionState。

SurfaceFlinger.setTransactionStat 调用后，中间的步骤暂不去考虑，最终在 SurfaceFlinger.setClientStateLocked 函数中，可以看到：

```
uint32_t SurfaceFlinger::setClientStateLocked(
        const FrameTimelineInfo& frameTimelineInfo, const ComposerState& composerState,
        int64_t desiredPresentTime, bool isAutoTimestamp, int64_t postTime, uint32_t permissions,
        std::unordered_set<ListenerCallbacks, ListenerCallbacksHash>& outListenerCallbacks) {
    const layer_state_t& s = composerState.state;
	// ......

    const uint64_t what = s.what;
    uint32_t flags = 0;
    sp<Layer> layer = nullptr;
    if (s.surface) {
        if (what & layer_state_t::eLayerCreated) {
            layer = handleLayerCreatedLocked(s.surface);
            if (layer) {
                // put the created layer into mLayersByLocalBinderToken.
                mLayersByLocalBinderToken.emplace(s.surface->localBinder(), layer);
                flags |= eTransactionNeeded | eTraversalNeeded;
                mLayersAdded = true;
            }
        } else {
            layer = fromHandleLocked(s.surface).promote();
        }
    }
    // ......
}
```

当检测到 layer_state_t 的 what 包含 layer_state_t::eLayerCreated 时，会调用以下代码：

```
layer = handleLayerCreatedLocked(s.surface);
            if (layer) {
                // put the created layer into mLayersByLocalBinderToken.
                mLayersByLocalBinderToken.emplace(s.surface->localBinder(), layer);
                flags |= eTransactionNeeded | eTraversalNeeded;
                mLayersAdded = true;
            }
```

主要内容为：

1）、调用 handleLayerCreatedLocked，从传入的 composerState 对象的 layer_state_t 类型的 state 中，拿到 Layer 句柄中保存的 Layer 对象，再根据 Layer 的 parent 情况选择是添加还是移除：

```
if (parent == nullptr && allowAddRoot) {
        layer->setIsAtRoot(true);
        mCurrentState.layersSortedByZ.add(layer);
    } else if (parent == nullptr) {
        layer->onRemovedFromCurrentState();
    } else if (parent->isRemovedFromCurrentState()) {
        parent->addChild(layer);
        layer->onRemovedFromCurrentState();
    } else {
        parent->addChild(layer);
    }
```

在我们的流程中，这里会将该 Layer 加入到父 Layer 之中，那么这个 Layer 才算真正加入到了 Layer 的层级结构中。

2）、将 <Handle, Layer> 记录到 SurfaceFlinger 的成员变量 mLayersByLocalBinderToken 中：

```
std::unordered_map<BBinder*, wp<Layer>> mLayersByLocalBinderToken GUARDED_BY(mStateLock);
```

这样 SurfaceFlinger 也可以直接通过 Layer 句柄找到对应的 Layer。

### 14 C++ 层 SurfaceControl.constructor

SurfaceFlinger 这边分析完了，再回到客户端 SurfaceComposerClient.createSurfaceChecked：

```
status_t SurfaceComposerClient::createSurfaceChecked(const String8& name, uint32_t w, uint32_t h,
                                                     PixelFormat format,
                                                     sp<SurfaceControl>* outSurface, uint32_t flags,
                                                     const sp<IBinder>& parentHandle,
                                                     LayerMetadata metadata,
                                                     uint32_t* outTransformHint) {
    sp<SurfaceControl> sur;
    status_t err = mStatus;

    if (mStatus == NO_ERROR) {
        sp<IBinder> handle;
        sp<IGraphicBufferProducer> gbp;

        uint32_t transformHint = 0;
        int32_t id = -1;
        err = mClient->createSurface(name, w, h, format, flags, parentHandle, std::move(metadata),
                                     &handle, &gbp, &id, &transformHint);
		// ......
        if (err == NO_ERROR) {
            *outSurface =
                    new SurfaceControl(this, handle, gbp, id, w, h, format, transformHint, flags);
        }
    }
	// ......
}
```

通过跨进程调用 Client.createSurface，我们最终是在 SurfaceFlinger 服务端生成了一个 Layer，并且返回了一个 Layer 的句柄，接着我们基于这些信息创建一个 SurfaceControl 对象：

```
SurfaceControl::SurfaceControl(const sp<SurfaceComposerClient>& client, const sp<IBinder>& handle,
                               const sp<IGraphicBufferProducer>& gbp, int32_t layerId,
                               uint32_t w, uint32_t h, PixelFormat format, uint32_t transform,
                               uint32_t flags)
      : mClient(client),
        mHandle(handle),
        mGraphicBufferProducer(gbp),
        mLayerId(layerId),
        mTransformHint(transform),
        mWidth(w),
        mHeight(h),
        mFormat(format),
        mCreateFlags(flags) {
}
```

在 SurfaceControl 内部，则保存了 SurfaceComposerClient、Layer 句柄等相关信息：

```
sp<SurfaceComposerClient>   mClient;
    sp<IBinder>                 mHandle;
    sp<IGraphicBufferProducer>  mGraphicBufferProducer;
```

这样客户端的 SurfaceControl 便和服务端中的一个具体的 Layer 通过 Layer 句柄联系起来了。

### 15 WindowSurfaceController.getSurfaceControl

回看 Java 层 SurfaceControl 的构造方法：

```
private SurfaceControl(SurfaceSession session, String name, int w, int h, int format, int flags,
        SurfaceControl parent, SparseIntArray metadata, WeakReference<View> localOwnerView,
        String callsite)
                throws OutOfResourcesException, IllegalArgumentException {
	// .......
    try {
		// ......
        mNativeObject = nativeCreate(session, name, w, h, format, flags,
                parent != null ? parent.mNativeObject : 0, metaParcel);
    } finally {
        metaParcel.recycle();
    }
	// ......
    mNativeHandle = nativeGetHandle(mNativeObject);
}
```

其内部的两个成员变量：

```
/**
     * @hide
     */
    public long mNativeObject;
    private long mNativeHandle;
```

分别保存了 Native 层的 SurfaceControl 对象的地址和 Layer 句柄的地址。

然后回看 WindowManagerService.createSurfaceControl 方法，是调用了 WindowSurfaceController.getSurfaceControl 方法来为 outSurfaceControl 进行赋值：

```
void getSurfaceControl(SurfaceControl outSurfaceControl) {
        outSurfaceControl.copyFrom(mSurfaceControl, "WindowSurfaceController.getSurfaceControl");
    }
```

这里调用了 SurfaceControl 的 copyFrom 方法，copyFrom 方法内部又调用了 SurfaceControl 的 assignNativeObject 方法：

```
private void assignNativeObject(long nativeObject, String callsite) {
        if (mNativeObject != 0) {
            release();
        }
        if (nativeObject != 0) {
            mCloseGuard.openWithCallSite("release", callsite);
        }
        mNativeObject = nativeObject;
        mNativeHandle = mNativeObject != 0 ? nativeGetHandle(nativeObject) : 0;
    }

    /**
     * @hide
     */
    public void copyFrom(@NonNull SurfaceControl other, String callsite) {
        mName = other.mName;
        mWidth = other.mWidth;
        mHeight = other.mHeight;
        mLocalOwnerView = other.mLocalOwnerView;
        assignNativeObject(nativeCopyFromSurfaceControl(other.mNativeObject), callsite);
    }
```

那么最终的结果是，WindowManagerService（WindowSurfaceController）和 ViewRootImpl 分别持有的 Java 层的 SurfaceControl 对象都指向同一个 Native 层的 SurfaceControl。

### 16 小结

![](https://img-blog.csdnimg.cn/edd17fd31e6a402ca824958a15b73766.png#pic_center)

1）、Java 层创建了两个 SurfaceControl 对象，分别由 ViewRootImpl 和 WindowSurfaceController 持有。

2）、C++ 层创建了一个 SurfaceControl 对象，它的指针地址被转为 long 型保存在 Java 层的 SurfaceControl 的 mNativeObject 中，那么 C++ 层就可以通过从 Java 传来的 long 型变量得到 C++ 层的 SurfaceControl 指针。

3）、每一个客户端的 SurfaceControl 对象持有一个 IBinder 类型的 Layer 句柄，该 Layer 句柄跨进程标识了一个 SurfaceFlinger 服务端的 Layer 对象，可以向 SurfaceFlinger 传入该 Layer 句柄从而找到该 SurfaceControl 对应的 Layer 对象。

二、BLASTBufferQueue 的创建
----------------------

回看 ViewRootImpl.relayoutWindow 方法：

```
private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {
        // ......

        int relayoutResult = mWindowSession.relayout(mWindow, params,
                (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                (int) (mView.getMeasuredHeight() * appScale + 0.5f), viewVisibility,
                insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
                mTmpFrames, mPendingMergedConfiguration, mSurfaceControl, mTempInsets,
                mTempControls, mSurfaceSize);
        mPendingBackDropFrame.set(mTmpFrames.backdropFrame);
        if (mSurfaceControl.isValid()) {
            if (!useBLAST()) {
                mSurface.copyFrom(mSurfaceControl);
            } else {
                final Surface blastSurface = getOrCreateBLASTSurface();
                // If blastSurface == null that means it hasn't changed since the last time we
                // called. In this situation, avoid calling transferFrom as we would then
                // inc the generation ID and cause EGL resources to be recreated.
                if (blastSurface != null) {
                    mSurface.transferFrom(blastSurface);
                }
            }
			// ......
        } else {
            destroySurface();
        }

		// ......
    }
```

经过 WindowManagerService.relayoutWindow 得到一个 SurfaceControl 对象并且赋值给 ViewRootImpl 的成员变量 mSurfaceControl 后，接着就是检查 mSurfaceControl 的有效性。我们不讨论特殊情况，即这里的 mSurfaceControl 是有效的。并且通过 debug，这里的 ViewRootImpl.useBLAST 方法返回的也是 true：

```
boolean useBLAST() {
        return mUseBLASTAdapter && !mForceDisableBLAST;
    }
```

mUseBLASTAdapter 受以下 Settings 数据库字段控制，默认是 true。mForceDisableBLAST 只有 ViewRootImpl.forceDisableBLAST 显式调用才会被设置为 true，这里是 false。

```
/**
         * If true, submit buffers using blast in ViewRootImpl.
         * (0 = false, 1 = true)
         * @hide
         */
        @Readable
        public static final String DEVELOPMENT_USE_BLAST_ADAPTER_VR =
                "use_blast_adapter_vr";
```

那么这里走的逻辑是先通过 ViewRootImpl.getOrCreateBLASTSurface 方法得到一个 Surface 对象，然后调用 Surface.transferFrom 完成对 mSurface 的内容填充。

### 1 ViewRootImpl.getOrCreateBLASTSurface

```
Surface getOrCreateBLASTSurface() {
        if (!mSurfaceControl.isValid()) {
            return null;
        }

        Surface ret = null;
        if (mBlastBufferQueue == null) {
            mBlastBufferQueue = new BLASTBufferQueue(mTag, mSurfaceControl,
                mSurfaceSize.x, mSurfaceSize.y,
                mWindowAttributes.format);
            // We only return the Surface the first time, as otherwise
            // it hasn't changed and there is no need to update.
            ret = mBlastBufferQueue.createSurface();
        } else {
            mBlastBufferQueue.update(mSurfaceControl,
                mSurfaceSize.x, mSurfaceSize.y,
                mWindowAttributes.format);
        }

        return ret;
    }
```

因为我们是初次调用这个方法，因此 mBlastBufferQueue 是 null。另外一提，mBlastBufferQueue 唯一赋值的地方就是这里。

先看下 BLASTBufferQueue 对象的创建流程。

### 2 BLASTBufferQueue.constructor

```
/** Create a new connection with the surface flinger. */
    public BLASTBufferQueue(String name, SurfaceControl sc, int width, int height,
            @PixelFormat.Format int format) {
        mNativeObject = nativeCreate(name, sc.mNativeObject, width, height, format);
    }
```

调用 JNI 的 nativeCreate 函数，long 型的 mNativeObject 保存 Native 层的 BLASTBufferQueue 对象的指针。

### 3 android_graphics_BLASTBufferQueue.nativeCreate

```
static jlong nativeCreate(JNIEnv* env, jclass clazz, jstring jName, jlong surfaceControl,
                          jlong width, jlong height, jint format) {
    String8 str8;
    if (jName) {
        const jchar* str16 = env->GetStringCritical(jName, nullptr);
        if (str16) {
            str8 = String8(reinterpret_cast<const char16_t*>(str16), env->GetStringLength(jName));
            env->ReleaseStringCritical(jName, str16);
            str16 = nullptr;
        }
    }
    std::string name = str8.string();
    sp<BLASTBufferQueue> queue =
            new BLASTBufferQueue(name, reinterpret_cast<SurfaceControl*>(surfaceControl), width,
                                 height, format);
    queue->incStrong((void*)nativeCreate);
    return reinterpret_cast<jlong>(queue.get());
}
```

创建一个 Native 层的 BLASTBufferQueue 对象，并且触发它的 onFirstRef 函数。

### 4 BLASTBufferQueue.constructor

```
BLASTBufferQueue::BLASTBufferQueue(const std::string& name, const sp<SurfaceControl>& surface,
                                   int width, int height, int32_t format)
      : mSurfaceControl(surface),
        mSize(width, height),
        mRequestedSize(mSize),
        mFormat(format),
        mNextTransaction(nullptr) {
    createBufferQueue(&mProducer, &mConsumer);
    // since the adapter is in the client process, set dequeue timeout
    // explicitly so that dequeueBuffer will block
    mProducer->setDequeueTimeout(std::numeric_limits<int64_t>::max());

    // safe default, most producers are expected to override this
    // 设置生产者执行一次dequeue可以获得的最大缓冲区数。
    mProducer->setMaxDequeuedBufferCount(2);
    mBufferItemConsumer = new BLASTBufferItemConsumer(mConsumer,
                                                      GraphicBuffer::USAGE_HW_COMPOSER |
                                                              GraphicBuffer::USAGE_HW_TEXTURE,
                                                      1, false);
    static int32_t id = 0;
    mName = name + "#" + std::to_string(id);
    auto consumerName = mName + "(BLAST Consumer)" + std::to_string(id);
    mQueuedBufferTrace = "QueuedBuffer - " + mName + "BLAST#" + std::to_string(id);
    id++;
    mBufferItemConsumer->setName(String8(consumerName.c_str()));
    // 设置当一个新的帧变为可用后会被通知的监听器对象。
    mBufferItemConsumer->setFrameAvailableListener(this);
    // 设置当一个旧的缓冲区被释放后会被通知的监听器对象。
    mBufferItemConsumer->setBufferFreedListener(this);
    // 设置当宽度和高度被请求为0时从dequeueBuffer返回的缓冲区的大小。默认是1x1。
    mBufferItemConsumer->setDefaultBufferSize(mSize.width, mSize.height);
    mBufferItemConsumer->setDefaultBufferFormat(convertBufferFormat(format));
    // 将BlastBufferItemConsumer的成员变量mBLASTBufferQueue指向当前BlastBufferQueue对象。
    mBufferItemConsumer->setBlastBufferQueue(this);

    // 得到SurfaceFlinger需要获取的缓冲区的数量。        
    ComposerService::getComposerService()->getMaxAcquiredBufferCount(&mMaxAcquiredBuffers);
    // 设置消费者可以一次获取的缓冲区的最大值（默认为1）。
    mBufferItemConsumer->setMaxAcquiredBufferCount(mMaxAcquiredBuffers);

    mTransformHint = mSurfaceControl->getTransformHint();
    mBufferItemConsumer->setTransformHint(mTransformHint);
    SurfaceComposerClient::Transaction()
            .setFlags(surface, layer_state_t::eEnableBackpressure,
                      layer_state_t::eEnableBackpressure)
            .setApplyToken(mApplyToken)
            .apply();
    mNumAcquired = 0;
    mNumFrameAvailable = 0;
    BQA_LOGV("BLASTBufferQueue created width=%d height=%d format=%d mTransformHint=%d", width,
             height, format, mTransformHint);
}
```

#### 4.1 BLASTBufferQueue.createBufferQueue

```
// Similar to BufferQueue::createBufferQueue but creates an adapter specific bufferqueue producer.
// This BQP allows invoking client specified ProducerListeners and invoke them asynchronously,
// emulating one way binder call behavior. Without this, if the listener calls back into the queue,
// we can deadlock.
void BLASTBufferQueue::createBufferQueue(sp<IGraphicBufferProducer>* outProducer,
                                         sp<IGraphicBufferConsumer>* outConsumer) {
    LOG_ALWAYS_FATAL_IF(outProducer == nullptr, "BLASTBufferQueue: outProducer must not be NULL");
    LOG_ALWAYS_FATAL_IF(outConsumer == nullptr, "BLASTBufferQueue: outConsumer must not be NULL");

    sp<BufferQueueCore> core(new BufferQueueCore());
    LOG_ALWAYS_FATAL_IF(core == nullptr, "BLASTBufferQueue: failed to create BufferQueueCore");

    sp<IGraphicBufferProducer> producer(new BBQBufferQueueProducer(core));
    LOG_ALWAYS_FATAL_IF(producer == nullptr,
                        "BLASTBufferQueue: failed to create BBQBufferQueueProducer");

    sp<BufferQueueConsumer> consumer(new BufferQueueConsumer(core));
    consumer->setAllowExtraAcquire(true);
    LOG_ALWAYS_FATAL_IF(consumer == nullptr,
                        "BLASTBufferQueue: failed to create BufferQueueConsumer");

    *outProducer = producer;
    *outConsumer = consumer;
}
```

类似于 BufferQueue::createBufferQueue，但是创建了一个适配器专用的 BufferQueue 生产者。这个 BQP 允许调用客户端指定的 ProducerListener 并异步调用它们，模拟单向 Binder 调用行为。如果 ProducerListener 在没有它的情况下回调进入 BLASTBufferQueue，我们就会死锁。

这个函数创建了三个对象，BufferQueueCore、BBQBufferQueueProducer 和 BufferQueueConsumer。暂时还不清楚这几个对象的作用，先从为数不多的注释去了解。

##### 4.1.1 BufferQueueCore

```
sp<BufferQueueCore> core(new BufferQueueCore());
```

创建了一个 BufferQueueCore 对象。

###### 4.1.1.1 BufferQueueCore 的继承关系

BufferQueueCore 对象继承自：

```
class BufferQueueCore : public virtual RefBase {
```

没有什么太值得关注的。

###### 4.1.1.2 BufferQueueCore 的构造函数

BufferQueueCore 的定义可以从它的构造函数的注释中去理解：

```
// BufferQueueCore manages a pool of gralloc memory slots to be used by
    // producers and consumers.
    BufferQueueCore();
```

BufferQueueCore 管理一个由生产者和消费者使用的 gralloc 内存槽池。Gralloc 查阅 google 文档：

> Gralloc 内存分配器会进行缓冲区分配，并通过两个特定于供应商的 HIDL 接口来进行实现（请参阅 hardware/interfaces/graphics/allocator/ 和 hardware/interfaces/graphics/mapper/）。allocate() 函数采用预期的参数（宽度、高度、像素格式）以及一组用法标志。

构造函数是：

```
BufferQueueCore::BufferQueueCore()
      : mMutex(),
// ......
{
    // getMaxBufferCountLocked返回一次可以分配的缓冲区的最大数量。
    int numStartingBuffers = getMaxBufferCountLocked();
    for (int s = 0; s < numStartingBuffers; s++) {
        // mFreeSlots包含了所有的处于FREE状态并且当前没有缓冲区附着的槽。
        mFreeSlots.insert(s);
    }
    for (int s = numStartingBuffers; s < BufferQueueDefs::NUM_BUFFER_SLOTS;
            s++) {
        // mUnusedSlots包含了当前没有被使用的所有槽。它们应该是空闲且没有附着一个缓冲区的。
        mUnusedSlots.push_front(s);
    }
}
```

##### 4.1.2 BBQBufferQueueProducer

```
sp<IGraphicBufferProducer> producer(new BBQBufferQueueProducer(core));
```

创建了一个 BBQBufferQueueProducer 对象。

###### 4.1.2.1 BBQBufferQueueProducer 的继承关系

BBQBufferQueueProducer 继承 BufferQueueProducer：

```
// Extends the BufferQueueProducer to create a wrapper around the listener so the listener calls
// can be non-blocking when the producer is in the client process.
class BBQBufferQueueProducer : public BufferQueueProducer {
```

BBQBufferQueueProducer 创建一层对 listener（IProducerListener）的封装，以便当生产者在客户端进程的时候 listener 调用可以是非阻塞的。

暂时不太清楚 BufferQueueProducer 的作用，只看到是 BufferQueueProducer 是继承 BnGraphicBufferProducer 的：

```
class BufferQueueProducer : public BnGraphicBufferProducer {
```

BnGraphicBufferProducer 又是继承 IGraphicBufferProducer 的：

```
class BnGraphicBufferProducer : public IGraphicBufferProducer {
```

IGraphicBufferProducer 的定义是：

```
/*
 * This class defines the Binder IPC interface for the producer side of
 * a queue of graphics buffers.  It's used to send graphics data from one
 * component to another.  For example, a class that decodes video for
 * playback might use this to provide frames.  This is typically done
 * indirectly, through Surface.
 *
 * The underlying mechanism is a BufferQueue, which implements
 * BnGraphicBufferProducer.  In normal operation, the producer calls
 * dequeueBuffer() to get an empty buffer, fills it with data, then
 * calls queueBuffer() to make it available to the consumer.
 *
 * This class was previously called ISurfaceTexture.
 */
#ifndef NO_BINDER
class IGraphicBufferProducer : public IInterface {
    DECLARE_HYBRID_META_INTERFACE(GraphicBufferProducer,
                                  HGraphicBufferProducerV1_0,
                                  HGraphicBufferProducerV2_0)
#else
class IGraphicBufferProducer : public RefBase {
```

这个类为图形缓冲区队列的生产者端定义了 Binder [IPC](https://so.csdn.net/so/search?q=IPC&spm=1001.2101.3001.7020) 接口。它用于将图形数据从一个组件发送到另一个组件。例如，一个用作回放的解码视频的类可以使用它来提供帧。这通常是通过 Surface 间接完成的。

底层机制是一个 BufferQueue，它实现了 BnGraphicBufferProducer。在通常操作中，生产者调用 dequeueBuffer() 来获得一个空缓冲区，用数据填充它，然后调用 queueBuffer() 使消费者可以使用它。

这个类以前被称为 ISurfaceTexture。

###### 4.1.2.2 BBQBufferQueueProducer 的构造函数

```
BBQBufferQueueProducer(const sp<BufferQueueCore>& core)
          : BufferQueueProducer(core, false /* consumerIsSurfaceFlinger*/) {}
```

再看 BufferQueueProducer 的构造函数：

```
BufferQueueProducer::BufferQueueProducer(const sp<BufferQueueCore>& core,
        bool consumerIsSurfaceFlinger) :
    mCore(core),
    mSlots(core->mSlots),
    mConsumerName(),
    mStickyTransform(0),
    mConsumerIsSurfaceFlinger(consumerIsSurfaceFlinger),
    mLastQueueBufferFence(Fence::NO_FENCE),
    mLastQueuedTransform(0),
    mCallbackMutex(),
    mNextCallbackTicket(0),
    mCurrentCallbackTicket(0),
    mCallbackCondition(),
    mDequeueTimeout(-1),
    mDequeueWaitingForAllocation(false) {}
```

将 sp<BufferQueueCore> 类型的成员变量 mCore 指向上面创建的 BufferQueueCore 对象。

##### 4.1.3 BufferQueueConsumer

```
sp<BufferQueueConsumer> consumer(new BufferQueueConsumer(core));
```

创建了一个 BufferQueueConsumer 对象。

###### 4.1.3.1 BufferQueueConsumer 的继承关系

BufferQueueConsumer 继承自 BnGraphicBufferConsumer：

```
class BufferQueueConsumer : public BnGraphicBufferConsumer {
```

BnGraphicBufferConsumer 又继承自 IGraphicBufferConsumer：

```
class BnGraphicBufferConsumer : public IGraphicBufferConsumer {
```

目前从 BufferQueueConsumer 的继承关系中无法得知 BufferQueueConsumer 的具体作用。

###### 4.1.3.2 BufferQueueConsumer 的构造函数

```
BufferQueueConsumer::BufferQueueConsumer(const sp<BufferQueueCore>& core) :
    mCore(core),
    mSlots(core->mSlots),
    mConsumerName() {}
```

将 sp<BufferQueueCore> 类型的成员变量 mCore 指向上面创建的 BufferQueueCore 对象。

#### 4.2 BLASTBufferItemConsumer

```
mBufferItemConsumer = new BLASTBufferItemConsumer(mConsumer,
                                                      GraphicBuffer::USAGE_HW_COMPOSER |
                                                              GraphicBuffer::USAGE_HW_TEXTURE,
                                                      1, false);
    static int32_t id = 0;
    mName = name + "#" + std::to_string(id);
    auto consumerName = mName + "(BLAST Consumer)" + std::to_string(id);
    mQueuedBufferTrace = "QueuedBuffer - " + mName + "BLAST#" + std::to_string(id);
    id++;
    mBufferItemConsumer->setName(String8(consumerName.c_str()));
    // 设置当一个新的帧变为可用后会被通知的监听器对象。
    mBufferItemConsumer->setFrameAvailableListener(this);
    // 设置当一个旧的缓冲区被释放后会被通知的监听器对象。
    mBufferItemConsumer->setBufferFreedListener(this);
    // 设置当宽度和高度被请求为0时从dequeueBuffer返回的缓冲区的大小。默认是1x1。
    mBufferItemConsumer->setDefaultBufferSize(mSize.width, mSize.height);
    mBufferItemConsumer->setDefaultBufferFormat(convertBufferFormat(format));
    // 将BlastBufferItemConsumer的成员变量mBLASTBufferQueue指向当前BlastBufferQueue对象。
    mBufferItemConsumer->setBlastBufferQueue(this);
```

这里看到，基于上一步得到的 sp<IGraphicBufferConsumer> 类型的成员变量 mConsumer，创建了一个 BLASTBufferItemConsumer 对象。

##### 4.2.1 BufferItemConsumer

BLASTBufferItemConsumer 继承自 BufferItemConsumer：

```
class BLASTBufferItemConsumer : public BufferItemConsumer {
```

BufferItemConsumer 继承自 ConsumerBase：

```
/**
 * BufferItemConsumer is a BufferQueue consumer endpoint that allows clients
 * access to the whole BufferItem entry from BufferQueue. Multiple buffers may
 * be acquired at once, to be used concurrently by the client. This consumer can
 * operate either in synchronous or asynchronous mode.
 */
class BufferItemConsumer: public ConsumerBase
```

BufferItemConsumer 是一个 BufferQueue 消费者端点，允许客户端从 BufferQueue 访问整个 BufferItem 条目。多个缓冲区可以被同时获取，供客户端并发使用。此消费者可以以同步或异步模式操作。

构造函数是：

```
BufferItemConsumer::BufferItemConsumer(
        const sp<IGraphicBufferConsumer>& consumer, uint64_t consumerUsage,
        int bufferCount, bool controlledByApp) :
    ConsumerBase(consumer, controlledByApp)
{
    // setConsumerUsageBits将为dequeueBuffer打开额外的使用位。它们与传递给dequeueBuffer的位合并。    
    status_t err = mConsumer->setConsumerUsageBits(consumerUsage);
    LOG_ALWAYS_FATAL_IF(err != OK,
            "Failed to set consumer usage bits to %#" PRIx64, consumerUsage);
    if (bufferCount != DEFAULT_MAX_BUFFERS) {
        // setMaxAcquiredBufferCount设置消费者一次可以取得的缓冲区的最大数量。
        err = mConsumer->setMaxAcquiredBufferCount(bufferCount);
        LOG_ALWAYS_FATAL_IF(err != OK,
                "Failed to set max acquired buffer count to %d", bufferCount);
    }
}
```

##### 4.2.2 ConsumerBase

```
// ConsumerBase is a base class for BufferQueue consumer end-points. It
// handles common tasks like management of the connection to the BufferQueue
// and the buffer pool.
class ConsumerBase : public virtual RefBase,
        protected ConsumerListener {
```

ConsumerBase 是 BufferQueue 消费者端点的基类。它处理一些常见的任务，比如管理到 BufferQueue 和缓冲池的连接。

再看它的构造函数的定义：

```
// ConsumerBase constructs a new ConsumerBase object to consume image
    // buffers from the given IGraphicBufferConsumer.
    // The controlledByApp flag indicates that this consumer is under the application's
    // control.
    explicit ConsumerBase(const sp<IGraphicBufferConsumer>& consumer, bool controlledByApp = false);
```

ConsumerBase 构造一个新的 ConsumerBase 对象来消费来自给定的 IGraphicBufferConsumer 的图像缓冲区。controlledByApp 标志表明该消费者处于应用程序的控制之下。

最后看下构造函数里的具体内容：

```
ConsumerBase::ConsumerBase(const sp<IGraphicBufferConsumer>& bufferQueue, bool controlledByApp) :
        mAbandoned(false),
        mConsumer(bufferQueue),
        mPrevFinalReleaseFence(Fence::NO_FENCE) {
    // Choose a name using the PID and a process-unique ID.
    mName = String8::format("unnamed-%d-%d", getpid(), createProcessUniqueId());

    // Note that we can't create an sp<...>(this) in a ctor that will not keep a
    // reference once the ctor ends, as that would cause the refcount of 'this'
    // dropping to 0 at the end of the ctor.  Since all we need is a wp<...>
    // that's what we create.
    wp<ConsumerListener> listener = static_cast<ConsumerListener*>(this);
    sp<IConsumerListener> proxy = new BufferQueue::ProxyConsumerListener(listener);

    status_t err = mConsumer->consumerConnect(proxy, controlledByApp);
    if (err != NO_ERROR) {
        CB_LOGE("ConsumerBase: error connecting to BufferQueue: %s (%d)",
                strerror(-err), err);
    } else {
        mConsumer->setConsumerName(mName);
    }
}
```

1）、将上面创建的 BLASTBufferQueue 的成员变量 mConsumer 赋值给 ConsumerBase 的 sp<IGraphicBufferConsumer> 类型的成员变量 mConsumer。

2）、将当前 ConsumerBase 转为一个 ConsumerListener 的弱引用，然后封装到 ProxyConsumerListener 中。ProxyConsumerListener 是 IConsumerListener 的实现，保存了真正的消费者对象的弱引用。所有对 ProxyConsumerListener 的调用都会转为对这个消费者对象的调用。

3）、接着调用 consumerConnect 将消费者连接到 BufferQueue。只有一个消费者可以连接，当这个消费者断开连接时，BufferQueue 就会被置为 “abandoned” 状态，导致生产者与 BufferQueue 的大多数交互都失败。controlledByApp 指示使用者是否被应用程序控制。

看到 BufferQueueConsumer.h 中的定义：

```
virtual status_t consumerConnect(const sp<IConsumerListener>& consumer,
            bool controlledByApp) {
        return connect(consumer, controlledByApp);
    }
```

接着调用 BufferQueueConsumer.connect 函数：

```
status_t BufferQueueConsumer::connect(
        const sp<IConsumerListener>& consumerListener, bool controlledByApp) {
	// ......

    mCore->mConsumerListener = consumerListener;
    mCore->mConsumerControlledByApp = controlledByApp;

    return NO_ERROR;
}
```

最终是将 BufferQueueConsumer 的 sp<BufferQueueCore> 类型的成员变量 mCore，中的 sp<IConsumerListener > 类型的成员变量 mConsumerListener 指向了一个封装了 BLASTBufferItemConsumer 对象的 ProxyConsumerListener 对象，即完成了从 BLASTBufferItemConsumer 到 BufferQueue 的连接。

### 5 小结

由于对这一部分不熟悉，所以一下子出现的这么多新的类感觉有点乱，先简单总结一下各个类之间的关系：

![](https://img-blog.csdnimg.cn/7799f9f0869a4056bd5d71150928ba07.png#pic_center)

三、Surface 的初始化
--------------

回看 ViewRootImpl.getOrCreateBLASTSurface 方法：

```
Surface getOrCreateBLASTSurface() {
        if (!mSurfaceControl.isValid()) {
            return null;
        }

        Surface ret = null;
        if (mBlastBufferQueue == null) {
            mBlastBufferQueue = new BLASTBufferQueue(mTag, mSurfaceControl,
                mSurfaceSize.x, mSurfaceSize.y,
                mWindowAttributes.format);
            // We only return the Surface the first time, as otherwise
            // it hasn't changed and there is no need to update.
            ret = mBlastBufferQueue.createSurface();
        } else {
            mBlastBufferQueue.update(mSurfaceControl,
                mSurfaceSize.x, mSurfaceSize.y,
                mWindowAttributes.format);
        }

        return ret;
    }
```

创建了一个 BLASTBufferQueue 对象后，接着就是调用 BLASTBufferQueue.createSurface 创建一个 Surface 对象。

### 1 BLASTBufferQueue.createSurface

```
/**
     * @return a new Surface instance from the IGraphicsBufferProducer of the adapter.
     */
    public Surface createSurface() {
        return nativeGetSurface(mNativeObject, false /* includeSurfaceControlHandle */);
    }
```

从适配器的 IGraphicsBufferProducer 处返回一个新的 Surface 实例。

### 2 android_graphics_BLASTBufferQueue.nativeGetSurface

```
static jobject nativeGetSurface(JNIEnv* env, jclass clazz, jlong ptr,
                                jboolean includeSurfaceControlHandle) {
    sp<BLASTBufferQueue> queue = reinterpret_cast<BLASTBufferQueue*>(ptr);
    return android_view_Surface_createFromSurface(env,
                                                  queue->getSurface(includeSurfaceControlHandle));
}
```

这里传入的 ptr 即上一节我们创建 BLASTBufferQueue 后，从 C++ 层返回给 Java 层的 BLASTBufferQueue 指针，所以这里我们可以拿到上一节创建的 BLASTBufferQueue 对象。

接着调用 BLASTBufferQueue.getSurface 函数。

### 3 BLASTBufferQueue.getSurface

```
sp<Surface> BLASTBufferQueue::getSurface(bool includeSurfaceControlHandle) {
    std::unique_lock _lock{mMutex};
    sp<IBinder> scHandle = nullptr;
    if (includeSurfaceControlHandle && mSurfaceControl) {
        scHandle = mSurfaceControl->getHandle();
    }
    return new BBQSurface(mProducer, true, scHandle, this);
}
```

这里传入的 includeSurfaceControlHandle 是 false，所以这里的 handle 为空。上面分析 SurfaceControl 的创建流程的时候，知道 SurfaceControl 保存的 handle 是其在 SurfaceFlinger 处对应的 Layer 的句柄。

接着创建了一个 BBQSurface 对象。

### 4 BBQSurface

```
class BBQSurface : public Surface {
private:
    sp<BLASTBufferQueue> mBbq;
public:
    BBQSurface(const sp<IGraphicBufferProducer>& igbp, bool controlledByApp,
               const sp<IBinder>& scHandle, const sp<BLASTBufferQueue>& bbq)
          : Surface(igbp, controlledByApp, scHandle), mBbq(bbq) {}
     // ......
}
```

BBQSurface 继承 Surface，内部有一个 sp<BLASTBufferQueue> 类型的成员变量 mBbq 指向一个 BLASTBufferQueue 对象。还是要去看 Surface 的构造函数。

### 5 Surface

1）、Surface 类继承 ANativeObjectBase：

```
/*
 * An implementation of ANativeWindow that feeds graphics buffers into a
 * BufferQueue.
 *
 * This is typically used by programs that want to render frames through
 * some means (maybe OpenGL, a software renderer, or a hardware decoder)
 * and have the frames they create forwarded to SurfaceFlinger for
 * compositing.  For example, a video decoder could render a frame and call
 * eglSwapBuffers(), which invokes ANativeWindow callbacks defined by
 * Surface.  Surface then forwards the buffers through Binder IPC
 * to the BufferQueue's producer interface, providing the new frame to a
 * consumer such as GLConsumer.
 */
class Surface
    : public ANativeObjectBase<ANativeWindow, Surface, RefBase>
{
```

一个 ANativeWindow 的实现，它将图形缓冲区提供给 BufferQueue。

这通常是由想要通过一些方法 (可能是 OpenGL，软件渲染器，或硬件解码器) 渲染帧的程序使用，并且这些程序拥有它们创建的并且可以转发到 SurfaceFlinger 进行合成的帧。例如，视频解码器可以渲染一帧并调用 eglSwapBuffers()，这个函数调用由 Surface 定义的 ANativeWindow 回调。然后，Surface 通过 Binder IPC 将缓冲区转发到 BufferQueue 的生产者接口，并向 GLConsumer 等消费者提供新的帧。

再看到 ANativeObjectBase 的定义：

```
/*
 * This helper class turns a ANativeXXX object type into a C++
 * reference-counted object; with proper type conversions.
 */
template <typename NATIVE_TYPE, typename TYPE, typename REF,
        typename NATIVE_BASE = android_native_base_t>
class ANativeObjectBase : public NATIVE_TYPE, public REF
{
```

那么这里 NATIVE_TYPE 的是 ANativeWindow，即 Surface 实际继承的是 ANativeWindow。

2）、再看 Surface 构造函数的定义：

```
/*
     * creates a Surface from the given IGraphicBufferProducer (which concrete
     * implementation is a BufferQueue).
     *
     * Surface is mainly state-less while it's disconnected, it can be
     * viewed as a glorified IGraphicBufferProducer holder. It's therefore
     * safe to create other Surfaces from the same IGraphicBufferProducer.
     *
     * However, once a Surface is connected, it'll prevent other Surfaces
     * referring to the same IGraphicBufferProducer to become connected and
     * therefore prevent them to be used as actual producers of buffers.
     *
     * the controlledByApp flag indicates that this Surface (producer) is
     * controlled by the application. This flag is used at connect time.
     *
     * Pass in the SurfaceControlHandle to store a weak reference to the layer
     * that the Surface was created from. This handle can be used to create a
     * child surface without using the IGBP to identify the layer. This is used
     * for surfaces created by the BlastBufferQueue whose IGBP is created on the
     * client and cannot be verified in SF.
     */
    explicit Surface(const sp<IGraphicBufferProducer>& bufferProducer, bool controlledByApp = false,
                     const sp<IBinder>& surfaceControlHandle = nullptr);
```

从给定的 IGraphicBufferProducer(具体实现是 BufferQueue) 创建一个 Surface。

当它断开连接时，Surface 主要是无状态的，它可以被视为美化的 IGraphicBufferProducer 持有者。因此，从同一个 IGraphicBufferProducer 中创建其他 Surface 是安全的。

然而，一旦一个 Surface 被连接，它将阻止其他引用相同的 IGraphicBufferProducer 的 Surface 被连接，从而防止它们被用作缓冲区的实际生产者。

controlledByApp 标志表示这个 Surface(生产者) 由应用程序控制。此标志在连接时使用。

传入 SurfaceControlHandle 来存储创建 Surface 的 Layer 的弱引用。这个句柄可以用来创建子 Surface，而不用 IGBP 来识别 Layer。这用于由 BlastBufferQueue 创建的 Surface，它的 IGBP 是在客户端创建的，不能在 SF 中验证。

3）、最后看下 Surface 的构造函数的具体内容：

```
Surface::Surface(const sp<IGraphicBufferProducer>& bufferProducer, bool controlledByApp,
                 const sp<IBinder>& surfaceControlHandle)
      : mGraphicBufferProducer(bufferProducer),
        mCrop(Rect::EMPTY_RECT),
        mBufferAge(0),
        mGenerationNumber(0),
        mSharedBufferMode(false),
        mAutoRefresh(false),
        mAutoPrerotation(false),
        mSharedBufferSlot(BufferItem::INVALID_BUFFER_SLOT),
        mSharedBufferHasBeenQueued(false),
        mQueriedSupportedTimestamps(false),
        mFrameTimestampsSupportsPresent(false),
        mEnableFrameTimestamps(false),
        mFrameEventHistory(std::make_unique<ProducerFrameEventHistory>()) {
	// ......
    mSurfaceControlHandle = surfaceControlHandle;
}
```

这里传入的 bufferProduer 参数是 BLASTBufferQueue 的 sp<IGraphicBufferProducer> 类型的成员变量 mProducer。

然后将成员变量 mSurfaceControlHandle 指向传入的 surfaceControlHandle。

```
// Reference to the SurfaceFlinger layer that was used to create this
    // surface. This is only populated when the Surface is created from
    // a BlastBufferQueue.
    sp<IBinder> mSurfaceControlHandle;
```

但是根据我们分析的这个流程，这里传入的 surfaceControlHandle 应该为空。

### 6 android_view_Surface.android_view_Surface_createFromSurface

回到 android_graphics_BLASTBufferQueue.nativeGetSurface：

```
static jobject nativeGetSurface(JNIEnv* env, jclass clazz, jlong ptr,
                                jboolean includeSurfaceControlHandle) {
    sp<BLASTBufferQueue> queue = reinterpret_cast<BLASTBufferQueue*>(ptr);
    return android_view_Surface_createFromSurface(env,
                                                  queue->getSurface(includeSurfaceControlHandle));
}
```

此时我们已经通过 BLASTBufferQueue.getSurface 创建了一个新的 Surface 对象，接着调用 android_view_Surface_createFromSurface 函数：

```
jobject android_view_Surface_createFromSurface(JNIEnv* env, const sp<Surface>& surface) {
    jobject surfaceObj = env->NewObject(gSurfaceClassInfo.clazz,
            gSurfaceClassInfo.ctor, (jlong)surface.get());
	// ......
    return surfaceObj;
}
```

这里创建了 Java 层的 Surface，将 Surface 的指针地址转化为了一个 Java 的 long 型，因此调用的是 Surface 的如下构造方法：

```
/* called from android_view_Surface_createFromIGraphicBufferProducer() */
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
    private Surface(long nativeObject) {
        synchronized (mLock) {
            setNativeObjectLocked(nativeObject);
        }
    }
```

最终调用 Surface.setNativeObjectLocked 将 C++ 层的 Surface 的地址保存在了 Java 层的 Surface 的 mNativeObject 中。

### 7 Surface.transferFrom

再次回到 ViewRootImpl.relayoutWindow 中。

```
private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {
        // ......

        int relayoutResult = mWindowSession.relayout(mWindow, params,
                (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                (int) (mView.getMeasuredHeight() * appScale + 0.5f), viewVisibility,
                insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
                mTmpFrames, mPendingMergedConfiguration, mSurfaceControl, mTempInsets,
                mTempControls, mSurfaceSize);
        mPendingBackDropFrame.set(mTmpFrames.backdropFrame);
        if (mSurfaceControl.isValid()) {
            if (!useBLAST()) {
                mSurface.copyFrom(mSurfaceControl);
            } else {
                final Surface blastSurface = getOrCreateBLASTSurface();
                // If blastSurface == null that means it hasn't changed since the last time we
                // called. In this situation, avoid calling transferFrom as we would then
                // inc the generation ID and cause EGL resources to be recreated.
                if (blastSurface != null) {
                    mSurface.transferFrom(blastSurface);
                }
            }
			// ......
        } else {
            destroySurface();
        }

		// ......
    }
```

此时经过 ViewRootImpl.getOrCreateBLASTSurface 方法，我们得到了一个 Surface 对象，接着调用 Surface.transferFrom：

```
/**
     * This is intended to be used by {@link SurfaceView#updateWindow} only.
     * @param other access is not thread safe
     * @hide
     * @deprecated
     */
    @Deprecated
    @UnsupportedAppUsage
    public void transferFrom(Surface other) {
        if (other == null) {
            throw new IllegalArgumentException("other must not be null");
        }
        if (other != this) {
            final long newPtr;
            synchronized (other.mLock) {
                newPtr = other.mNativeObject;
                other.setNativeObjectLocked(0);
            }

            synchronized (mLock) {
                if (mNativeObject != 0) {
                    nativeRelease(mNativeObject);
                }
                setNativeObjectLocked(newPtr);
            }
        }
    }
```

很简单，将 other 的 mNativeObject 赋值给当前 Surface，并且将 other 的 mNativeObject 置为 0，将 other 无效化。

### 8 小结

![](https://img-blog.csdnimg.cn/f28d979465494713a9cf25cdc28730c8.png#pic_center)

1）、Surface 的创建的时候接受 BLASTBufferQueue 对象和 BLASTBufferQueue 的 mProducer 成员变量为参数，这使得 Surface 与 BufferQueue 建立起了关系。

2）、Surface 与 SurfaceControl 关系的建立应该是靠它的成员变量 mSurfaceControlHandle，通过这个句柄它可以和 SurfaceControl 指向同一个 Layer 图层，但是我们分析的这个流程，最终 Surface 初始化后 mSurfaceControlHandle 仍然为空，这一点暂时还不太懂是什么情况。

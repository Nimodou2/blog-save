> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/130763295?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/11d7d9fcbc554a91b7b7ceee3006002f.jpeg#pic_center)  
上一篇文章分析了 DisplayList 的构建流程，即 ThreadedRenderer.draw 方法中的 ThreadedRenderer.updateRootDisplay[List 方法](https://so.csdn.net/so/search?q=List%E6%96%B9%E6%B3%95&spm=1001.2101.3001.7020)的内容。

```
// frameworks\base\core\java\android\view\ThreadedRenderer.java

	void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
        // ......

        updateRootDisplayList(view, callbacks);

        // ......

        int syncResult = syncAndDrawFrame(frameInfo);
        // ......
    }
```

这一篇文章继续分析 DisplayList 的渲染流程，即 ThreadedRenderer.syncAndDrawFrame 方法的内容。

大概的流程如下：

![](https://img-blog.csdnimg.cn/93492fc4cb354b4fb013e224b819528d.png#pic_center)

1 ThreadedRenderer.syncAndDrawFrame
-----------------------------------

```
/**
     * Syncs the RenderNode tree to the render thread and requests a frame to be drawn.
     *
     * @hide
     */
    @SyncAndDrawResult
    public int syncAndDrawFrame(@NonNull FrameInfo frameInfo) {
        return nSyncAndDrawFrame(mNativeProxy, frameInfo.frameInfo, frameInfo.frameInfo.length);
    }
```

将之前在主线程通过 ThreadedRenderer.updateRootDisplayList 方法构建出的 RenderNode 树同步给 RenderThread，并且请求一帧的绘制。

nSyncAndDrawFrame 是 [JNI](https://so.csdn.net/so/search?q=JNI&spm=1001.2101.3001.7020) 函数，对应 android_view_ThreadedRenderer_syncAndDrawFrame。

1.1 android_view_ThreadedRenderer_syncAndDrawFrame
--------------------------------------------------

```
// frameworks\base\libs\hwui\jni\android_graphics_HardwareRenderer.cpp

static int android_view_ThreadedRenderer_syncAndDrawFrame(JNIEnv* env, jobject clazz,
                                                          jlong proxyPtr, jlongArray frameInfo,
                                                          jint frameInfoSize) {
    LOG_ALWAYS_FATAL_IF(frameInfoSize != UI_THREAD_FRAME_INFO_SIZE,
                        "Mismatched size expectations, given %d expected %zu", frameInfoSize,
                        UI_THREAD_FRAME_INFO_SIZE);
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    env->GetLongArrayRegion(frameInfo, 0, frameInfoSize, proxy->frameInfo());
    return proxy->syncAndDrawFrame();
}
```

这里从上层传入的地址拿到 Native 层对应的 RenderProxy 对象，然后调用 RenderProxy.syncAndDrawFrame。

1.2 RenderProxy.syncAndDrawFrame
--------------------------------

```
int RenderProxy::syncAndDrawFrame() {
    return mDrawFrameTask.drawFrame();
}
```

RenderProxy 的成员变量 mDrawFrameTask 是一个 DrawFrameTask 对象，继续调用 DrawFrameTask.drawFrame。

1.3 DrawFrameTask.drawFrame
---------------------------

```
int DrawFrameTask::drawFrame() {
    LOG_ALWAYS_FATAL_IF(!mContext, "Cannot drawFrame with no CanvasContext!");

    mSyncResult = SyncResult::OK;
    mSyncQueued = systemTime(SYSTEM_TIME_MONOTONIC);
    postAndWait();

    return mSyncResult;
}
```

继续调用了自身的 postAnd[Wait 函数](https://so.csdn.net/so/search?q=Wait%E5%87%BD%E6%95%B0&spm=1001.2101.3001.7020)。

1.4 DrawFrameTask.postAndWait
-----------------------------

```
void DrawFrameTask::postAndWait() {
    ATRACE_CALL();
    AutoMutex _lock(mLock);
    mRenderThread->queue().post([this]() { run(); });
    mSignal.wait(mLock);
}
```

1）、向 RenderThread 的工作队列 WorkQueue 发布一个任务，根据之前的分析，知道了后续的 run 函数将会运行在 RenderThread 自己的线程中，即渲染线程，而非当前的主线程。

2）、DrawFrameTask 的成员变量 mSignal 是一个 Condition 类型的条件变量，这里调用了其 wait 函数让主线程等待在此处。

1.5 DrawFrameTask.run
---------------------

```
void DrawFrameTask::run() {
   // ......

    bool canUnblockUiThread;
    bool canDrawThisFrame;
    {
        TreeInfo info(TreeInfo::MODE_FULL, *mContext);
        info.forceDrawFrame = mForceDrawFrame;
        mForceDrawFrame = false;
        canUnblockUiThread = syncFrameState(info);
        canDrawThisFrame = info.out.canDrawThisFrame;

        // ......
    }

    // ......

    // From this point on anything in "this" is *UNSAFE TO ACCESS*
    if (canUnblockUiThread) {
        unblockUiThread();
    }

    // ......

    nsecs_t dequeueBufferDuration = 0;
    if (CC_LIKELY(canDrawThisFrame)) {
        dequeueBufferDuration = context->draw();
    } else {
        // ......
    }

    // ......

    if (!canUnblockUiThread) {
        unblockUiThread();
    }
    
    // ......
}
```

要理解这个函数首先要理解应用程序进程的 Main Thread 和 Render Thread 是如何协作的。从前面的分析可以知道，Main Thread 请求 Render Thread 执行 Draw Frame Task 的时候，不能马上返回，而是进入等待状态。等到 Render Thread 从 Main Thread 同步完绘制所需要的信息之后，Main Thread 才会被唤醒。

那么，Render Thread 要从 Main Thread 同步什么信息呢？原来，Main Thread 和 Render Thread 都各自维护了一份应用程序窗口视图信息。各自维护了一份应用程序窗口视图信息的目的，就是为了可以互不干扰，进而实现最大程度的并行。其中，Render Thread 维护的应用程序窗口视图信息是来自于 Main Thread 的。因此，当 Main Thread 维护的应用程序窗口信息发生了变化时，就需要同步到 Render Thread 去。

应用程序窗口的视图信息包括各个 Render Node 的 DisplayList、RenderProperties 等属性，其中 DisplayList 保存的是绘制命令，RenderProperties 保存的则是 RenderNode 的几何信息。

```
class RenderNode : public VirtualLightRefBase {
    friend class TestUtils;  // allow TestUtils to access syncDisplayList / syncProperties

public:
    enum DirtyPropertyMask {
        GENERIC = 1 << 1,
        TRANSLATION_X = 1 << 2,
        TRANSLATION_Y = 1 << 3,
        TRANSLATION_Z = 1 << 4,
        SCALE_X = 1 << 5,
        SCALE_Y = 1 << 6,
        ROTATION = 1 << 7,
        ROTATION_X = 1 << 8,
        ROTATION_Y = 1 << 9,
        X = 1 << 10,
        Y = 1 << 11,
        Z = 1 << 12,
        ALPHA = 1 << 13,
        DISPLAY_LIST = 1 << 14,
    };

    // ......

private:
    // ......

    uint32_t mDirtyPropertyFields;
    RenderProperties mProperties;
    RenderProperties mStagingProperties;

    // Owned by UI. Set when DL is set, cleared when DL cleared or when node detached
    // (likely by parent re-record/removal)
    bool mValid = false;

    bool mNeedsDisplayListSync;
    // WARNING: Do not delete this directly, you must go through deleteDisplayList()!
    DisplayList mDisplayList;
    DisplayList mStagingDisplayList;

    // ......

};  // class RenderNode
```

其中，成员变量 mStagingProperties 描述的 Render Properties 和成员变量 mStagingDisplayListData 描述的 Display List Data 由 Main Thread 维护，而成员变量 mProperties 描述的 Render Properties 和成员变量 mDisplayListData 描述的 Display List Data 由 Render Thread 维护。

回忆在记录绘制命令的时候，会调用 Java 层的 RenderNode.endRecording 方法，将绘制命令记录在 RenderNode 中，这最终是通过调用 C/C++ 层 RenderNode 的 setStagingDisplayList 函数实现的：

```
void RenderNode::setStagingDisplayList(DisplayList&& newData) {
    mValid = newData.isValid();
    mNeedsDisplayListSync = true;
    mStagingDisplayList = std::move(newData);
}
```

*   mValid 置为 true，表示当前 RenderNode 持有一个 DisplayList。
    
*   mNeedsDisplayListSync 置为 true，表示需要向 RenderThread 同步 DisplayList 的信息。
    
*   具体的 DisplayList 信息则保存在 mStagingDisplayList 中。
    

再回到 DrawFrameTask 类的成员函数 run 中，它的执行逻辑如下所示：

1）、创建一个 TreeInfo 对象，模式为 MODE_FULL，暂且记下。

2）、调用成员函数 syncFrameState 将应用程序窗口的 Display List、Render Property 以及 Display List 引用的 Bitmap 等信息从 Main Thread 同步到 Render Thread 中。注意，在这个同步过程中，Main Thread 是处于等待状态的。

3）、如果成员函数 syncFrameState 能顺利地完成信息同步，那么它的返回值 canUnblockUiThread 就会等于 true，表示在 Render Thread 渲染应用程序窗口的下一帧之前，就可以唤醒 Main Thread 了。否则的话，就要等到 Render Thread 渲染应用程序窗口的下一帧之后，才能唤醒 Main Thread。唤醒 Render Thread 是通过调用成员函数 unblockUiThread 来完成的，如下所示：

```
void DrawFrameTask::unblockUiThread() {
    AutoMutex _lock(mLock);
    mSignal.signal();
}
```

前面 Main Thread 就刚好是等待在 DrawFrameTask 类的成员变量 mSignal 描述的一个条件变量之上的，所以现在 Render Thread 就通过这个条件变量来唤醒它。

4）、调用成员变量 mContext 描述的一个 CanvasContext 对象的成员函数 draw 渲染应用程序窗口的 Display List。注意，这一步执行的时候，Main Thread 有可能是处于等待状态，也有可能不是处于等待状态。这取决于前面的信息同步结果。信息同步结果是通过一个 TreeInfo 来描述的。当 Main Thread 不是处于等待状态时，它就可以马上处理其它事情了，例如构建应用程序窗口下一帧时使用的 Display List。这样就可以做到 Render Thread 在绘制应用程序窗口的当前帧的同时，Main Thread 可以并行地去构建应用程序窗口的下一帧的 Display List。这一点也是 Android 5.0 引进 Render Thread 的好处所在。

接下来，我们就先分析应用程序窗口绘制信息的同步过程，即 DrawFrameTask 类的成员函数 syncFrameState 的实现，接着再分析应用程序窗口的 Display List 的渲染过程，即 CanvasContext 类的成员函数 draw 的实现。

2 DrawFrameTask.syncFrameState
------------------------------

```
bool DrawFrameTask::syncFrameState(TreeInfo& info) {
    // ......
    
    bool canDraw = mContext->makeCurrent();
    
    // ......
    
    mContext->prepareTree(info, mFrameInfo, mSyncQueued, mTargetNode);

    // ......
    
    // If prepareTextures is false, we ran out of texture cache space
    return info.prepareTextures;
}
```

1）、调用 CanvasContext.makeCurrent。之前在分析硬件加速渲染环境初始化的时候，当时只是创建了一个 EglSurface，但是并没有调用 eglMakeCurrent 将这个 EglSurface 和 EglDisplay 以及 EglContext 进行绑定，而这里则是调用了 CanvasContext.makeCurrent，最终将会调用到 EglManager.makeCurrent 完成绑定。

2）、调用 CanvasContext.prepareTree 函数，之前分析硬件渲染环境初始化流程的时候，知道 DrawFrameTask 初始化的时候，窗口的 RootRenderNode 就保存在了其成员变量 mTargetNode 中。

3）、TreeInfo 的成员变量 prepareTextures 描述 Texture 缓存空间是否已经被用尽，因为我们的分析不涉及 Texture，因此这里默认为 true，表示主线程不需要等待 RenderThread 绘制完一帧后再被唤醒。

主要看下 CanvasContext.prepareTree 函数的内容。

### 2.1 CanvasContext.prepareTree

```
void CanvasContext::prepareTree(TreeInfo& info, int64_t* uiFrameInfo, int64_t syncQueued,
                                RenderNode* target) {
    // ......
    
    for (const sp<RenderNode>& node : mRenderNodes) {
        // Only the primary target node will be drawn full - all other nodes would get drawn in
        // real time mode. In case of a window, the primary node is the window content and the other
        // node(s) are non client / filler nodes.
        info.mode = (node.get() == target ? TreeInfo::MODE_FULL : TreeInfo::MODE_RT_ONLY);
        node->prepareTree(info);
        GL_CHECKPOINT(MODERATE);
    }

    // ......

    bool postedFrameCallback = false;
    if (info.out.hasAnimations || !info.out.canDrawThisFrame) {
        if (CC_UNLIKELY(!Properties::enableRTAnimations)) {
            info.out.requiresUiRedraw = true;
        }
        if (!info.out.requiresUiRedraw) {
            // If animationsNeedsRedraw is set don't bother posting for an RT anim
            // as we will just end up fighting the UI thread.
            mRenderThread.postFrameCallback(this);
            postedFrameCallback = true;
        }
    }

    // ......
}
```

1）、之前分析硬件渲染环境初始化流程的时候，知道 CanvasContext 初始化的时候，窗口的 RootRenderNode 就保存在了其成员变量 mRenderNodes 中，另外注意这里的 mRenderNodes 是一个向量，那就是说 mRenderNodes 中除了 RootRenderNode 外可能还有其它的 RenderNode，事实上也的确如此，Java 层也可以通过 addRenderNode 的方式向 mRenderNodes 添加别的 RenderNode，主要是 BackdropFrameRenderer 类，不过这些 RenderNode 是在窗口 resize 的时候用来添加背景的，暂不考虑这种情况，那么在此处，从 mRenderNodes 拿到的就是 RootRenderNode，而传参 target 是 DrawFrameTask 的成员变量 mTargetNode，也是 RootRenderNode，那么这里的 TreeInfo 的 mode 就是 MODE_FULL。

2）、TreeInfo 中定义了两种模式：

```
// The full monty - sync, push, run animators, etc... Used by DrawFrameTask
        // May only be used if both the UI thread and RT thread are blocked on the
        // prepare
        MODE_FULL,
        // Run only what can be done safely on RT thread. Currently this only means
        // animators, but potentially things like SurfaceTexture updates
        // could be handled by this as well if there are no listeners
        MODE_RT_ONLY,
```

*   MODE_FULL，表示本次绘制是由主线程驱动的。
*   MODE_RT_ONLY，表示本次绘制是由渲染线程驱动的，主要是动画。

3）、之前分析硬件渲染环境初始化流程的时候，知道 CanvasContext 初始化的时候，窗口的 RootRenderNode 就保存在了其成员变量 mRenderNodes 中，这里继续调用了 RenderNode.prepareTree 函数。

4）、看一下 Out 的几个成员变量的定义：

```
struct Out {
        // ......
        
        // This is only updated if evaluateAnimations is true
        bool hasAnimations = false;
        // This is set to true if there is an animation that RenderThread cannot
        // animate itself, such as if hasFunctors is true
        // This is only set if hasAnimations is true
        bool requiresUiRedraw = false;
        // This is set to true if draw() can be called this frame
        // false means that we must delay until the next vsync pulse as frame
        // production is outrunning consumption
        // NOTE that if this is false CanvasContext will set either requiresUiRedraw
        // *OR* will post itself for the next vsync automatically, use this
        // only to avoid calling draw()
        bool canDrawThisFrame = true;
        
        // ......
    } out;
```

*   hasAnimations，动画是否正在运行，即未执行完成。
*   requiresUiRedraw，为 true 表示动画无法由 RenderThread 进行驱使，应该由主线程发起渲染请求。
*   canDrawThisFrame，为 true 表示 CanvasContext.draw 函数可以在这一帧的时候调用，为 false 表示推迟本次绘制到下一个 Vsync 信号到来的时候。

在这个同步的过程中，如果某些 View 设置了动画，并且这些动还未执行完成，那么参数 info 指向的 TreeInfo 对象的成员变量 hasAnimations 的值就会等于 true。这时候如果应用程序窗口的下一帧不可以渲染，即上述 TreeInfo 对象的成员变量 canDrawThisFrame 的值等于 false，并且所有 View 设置的动画都是非异步的，即上述 TreeInfo 对象的成员变量 requiresUiRedraw 的值等于 false，那么就需要解决一个问题，那些未执行完成的动画如何继续执行下去？因为等到当应用程序窗口的下一帧可以渲染时，这些未完成的动画还是需要继续执行的。

我们知道，当 TreeInfo 对象的成员变量 requiresUiRedraw 的值等于 true 时，Main Thread 会自动发起渲染应用程序窗口的 Display List 的命令。在这个命令的执行过程中，未完成的动画是可以继续执行的。但是当 TreeInfo 对象的成员变量 requiresUiRedraw 的值等于 false 时，Main Thread 不会自动发起渲染应用程序窗口的 Display List 的命令，这时候就需要向 Render Thread 注册一个 IFrameCallback 接口，这是通过调用 CanvasContext 类的成员变量 mRenderThread 指向的一个 RenderThread 对象的成员函数 postFrameCallback 实现的。

根据之前的分析，我们知道注册到 Render Thread 的 IFrameCallback 接口在下一个 Vsync 信号到来时，它的成员函数 doFrame 会被调用，这时候就可以执行渲染应用程序窗口的下一帧了。在渲染的过程中，就可以继续执行那些未完成的动画了。

### 2.2 RenderNode.prepareTreeImpl

```
void RenderNode::prepareTree(TreeInfo& info) {
    // ......   
    prepareTreeImpl(observer, info, false);
    // ......
}

/**
 * Traverse down the the draw tree to prepare for a frame.
 *
 * MODE_FULL = UI Thread-driven (thus properties must be synced), otherwise RT driven
 *
 * While traversing down the tree, functorsNeedLayer flag is set to true if anything that uses the
 * stencil buffer may be needed. Views that use a functor to draw will be forced onto a layer.
 */
void RenderNode::prepareTreeImpl(TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer) {
    // ......

    if (info.mode == TreeInfo::MODE_FULL) {
        pushStagingPropertiesChanges(info);
    }

    // ......

    if (info.mode == TreeInfo::MODE_FULL) {
        pushStagingDisplayListChanges(observer, info);
    }

    if (mDisplayList) {
        // ......
        bool isDirty = mDisplayList.prepareListAndChildren(
                observer, info, childFunctorsNeedLayer,
                [this](RenderNode* child, TreeObserver& observer, TreeInfo& info,
                       bool functorsNeedLayer) {
                    child->prepareTreeImpl(observer, info, functorsNeedLayer);
                    mHasHolePunches |= child->hasHolePunches();
                });
        if (isDirty) {
            damageSelf(info);
        }
    } else {
        // ......
    }

    // ......
}
```

RenderNode 的 prepareTree 函数继续调用了其 prepareTreeImpl 函数：

1）、根据之前的分析，知道本次的 TreeInfo 的模式为 MODE_FULL，那么会调用 RenderNode.pushStagingPropertiesChanges，该函数主要是调用 RenderNode.syncProperties 函数将成员变量 mStagingProperties 的信息同步到成员变量 mProperties 中。

2）、RenderNode.pushStagingDisplayListChanges 函数和 RenderNode.pushStagingPropertiesChanges 类似，最后会调用 RenderNode.syncDisplayList 函数，将成员变量 mStagingDisplayList 的信息同步到成员变量 mDisplayList 中。

3）、DisplayList.prepareListAndChildren 函数将会继续调用封装在其内的 SkiaDisplayList.prepareListAndChildren 函数，SkiaDisplayList.prepareListAndChildren 函数仅由 RenderNode.prepareTree 调用，以便在 UI 线程被阻塞时进行从 UI 线程到渲染线程的信息同步工作。这里看到将对子 RenderNode 的 RenderNode::prepareTreeImpl 函数的调用封装为参数传给了 DisplayList.prepareListAndChildren。如果返回 isDirty 为 true，那么表示当前 RenderNode 无效需要重绘，那么就将通过 RenderNode.damageSelf 将当前 RenderNode 的脏区域添加到 TreeInfo.damageAccumulator，一般来说 RenderNode 的脏区域就是这个 RenderNode 对应的 RenderProperties 的宽高所代表的区域。

### 2.3 SkiaDisplayList.prepareListAndChildren

```
// frameworks\base\libs\hwui\DisplayList.h
	bool prepareListAndChildren(
            TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer,
            std::function<void(RenderNode*, TreeObserver&, TreeInfo&, bool)> childFn) {
        return mImpl && mImpl->prepareListAndChildren(
                observer, info, functorsNeedLayer, std::move(childFn));
    }

// frameworks\base\libs\hwui\pipeline\skia\SkiaDisplayList.cpp
bool SkiaDisplayList::prepareListAndChildren(
        TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer,
        std::function<void(RenderNode*, TreeObserver&, TreeInfo&, bool)> childFn) {
    // If the prepare tree is triggered by the UI thread and no previous call to
    // pinImages has failed then we must pin all mutable images in the GPU cache
    // until the next UI thread draw.
#ifdef __ANDROID__ // Layoutlib does not support CanvasContext
    if (info.prepareTextures && !info.canvasContext.pinImages(mMutableImages)) {
        // In the event that pinning failed we prevent future pinImage calls for the
        // remainder of this tree traversal and also unpin any currently pinned images
        // to free up GPU resources.
        info.prepareTextures = false;
        info.canvasContext.unpinImages();
    }
#endif

    // ......

    for (auto& child : mChildNodes) {
        RenderNode* childNode = child.getRenderNode();
        // ......
        childFn(childNode, observer, info, functorsNeedLayer);
        // ......
    }

    // ......
}
```

之前分析过，DisplayList 是对 SkiaDisplayList 的封装，那么真正得到调用的是 SkiaDisplayList.prepareListAndChildren 函数。

这里的 childFn 就是上面封装的：

```
[this](RenderNode* child, TreeObserver& observer, TreeInfo& info,
                       bool functorsNeedLayer) {
                    child->prepareTreeImpl(observer, info, functorsNeedLayer);
                    mHasHolePunches |= child->hasHolePunches();
                }
```

那么会从 RootRenderNode 开始，递归调用 RenderNode 树中的每一个 RenderNode 的 RenderNode.prepareTreeImpl 函数，完成 Properties、DisplayList 等信息从主线程到渲染线程的同步，以及脏区域的计算等。

3 CanvasContext.draw
--------------------

```
nsecs_t CanvasContext::draw() {
    // .....
    
    SkRect dirty;
    mDamageAccumulator.finish(&dirty);
    
    // ......

    Frame frame = mRenderPipeline->getFrame();
    SkRect windowDirty = computeDirtyRect(frame, &dirty);

    ATRACE_FORMAT("Drawing " RECT_STRING, SK_RECT_ARGS(dirty));

    const auto drawResult = mRenderPipeline->draw(frame, windowDirty, dirty, mLightGeometry,
                                                  &mLayerUpdateQueue, mContentDrawBounds, mOpaque,
                                                  mLightInfo, mRenderNodes, &(profiler()));

    // .....

    bool didSwap = mRenderPipeline->swapBuffers(frame, drawResult.success, windowDirty,
                                                mCurrentFrameInfo, &requireSwap);

    // .....
}
```

1）、调用 SkiaOpenGLPipeline.getFrame，向 BufferQueue 请求一个图形缓冲区用来进行绘制。

2）、调用 CanvasContext.computeDirtyRect，根据上面构建的 Frame 对象重新计算窗口的脏区域。

3）、调用 SkiaOpenGLPipeline.draw，向图形缓冲区绘制内容。

4）、调用 SkiaOpenGLPipeline.swapBuffers，向 BufferQueue 提交已经有内容的图形缓冲区。

下面分别分析这几部分。

4 请求图形缓冲区
---------

请求图形缓冲区的操作是通过调用 SkiaOpenGLPipeline.getFrame 函数来完成的。

```
Frame SkiaOpenGLPipeline::getFrame() {
    LOG_ALWAYS_FATAL_IF(mEglSurface == EGL_NO_SURFACE,
                        "drawRenderNode called on a context with no surface!");
    return mEglManager.beginFrame(mEglSurface);
}
```

之前分析硬件加速渲染环境初始化的时候，知道 SkiaOpenGLPipeline 的成员变量 mEglSurface 已经被初始化过了，为 EglManager.createSurface 函数的返回值。

```
Frame EglManager::beginFrame(EGLSurface surface) {
    LOG_ALWAYS_FATAL_IF(surface == EGL_NO_SURFACE, "Tried to beginFrame on EGL_NO_SURFACE!");
    makeCurrent(surface);
    Frame frame;
    frame.mSurface = surface;
    eglQuerySurface(mEglDisplay, surface, EGL_WIDTH, &frame.mWidth);
    eglQuerySurface(mEglDisplay, surface, EGL_HEIGHT, &frame.mHeight);
    if (frame.mWidth > 65536 || frame.mHeight > 65536) {
        ALOGE("beginFrame's surface %p dimension exceeds 65536: frame.mWidth=%d, frame.mHeight=%d",
              (void*)surface, frame.mWidth, frame.mHeight);
    }
    frame.mBufferAge = queryBufferAge(surface);
    eglBeginFrame(mEglDisplay, surface);
    return frame;
}
```

1）、调用 EglManager.makeCurrent，最终调用 eglMakeCurrent 函数重新绑定一下 EglSurface 和上下文。

2）、创建一个 Frame 对象并且初始化，Frame 类的定义为：

```
class Frame {
public:
    // ......

private:
    Frame() {}
    friend class EglManager;

    int32_t mWidth;
    int32_t mHeight;
    int32_t mBufferAge;

    EGLSurface mSurface;

    // Maps from 0,0 in top-left to 0,0 in bottom-left
    // If out is not an int32_t[4] you're going to have a bad time
    void map(const SkRect& in, int32_t* out) const;
};
```

Frame 用来描述一帧的信息，宽高之类的不用多说，这里的 BufferAge 作为缓冲区的年龄，代表了从缓冲区入队以后经过的帧数，App 可以通过该信息可重新使用旧帧的内容并在下一帧时最大限度地减少必须重绘的内容。EGLSurface 创建的时候以 Surface 作为参数，渲染该 EGLSurface 的时候会向 BufferQueue 请求一个缓冲区保存在 Surface 中。

这里重要的是对 eglQuerySurface 函数的调用。该函数的具体内容不再分析，只需要知道这一步最终将会调用 BufferQueueProducer.dequeueBuffer 向 BufferQueue 请求一个缓冲区即可，作为我们后续分析 dequeueBuffer 流程的起点。

```
#00 pc 0000000000090884  /system/lib64/libgui.so (android::BufferQueueProducer::dequeueBuffer(int*, android::sp<android::Fence>*, unsigned int, unsigned int, int, unsigned long, unsigned long*, android::FrameEventHistoryDelta*)+308)
#01 pc 00000000000f1a28  /system/lib64/libgui.so (android::Surface::dequeueBuffer(ANativeWindowBuffer**, int*)+488)
#02 pc 000000000028d778  /system/lib64/libhwui.so (android::uirenderer::renderthread::ReliableSurface::hook_dequeueBuffer(ANativeWindow*, int (*)(ANativeWindow*, ANativeWindowBuffer**, int*), void*, ANativeWindowBuffer**, int*)+200)
#03 pc 00000000000efb6c  /system/lib64/libgui.so (android::Surface::hook_dequeueBuffer(ANativeWindow*, ANativeWindowBuffer**, int*)+92)
#04 pc 0000000000050298  /vendor/lib64/libIMGegl.so
#05 pc 00000000000232e4  /vendor/lib64/libIMGegl.so (KEGLGetDrawableParameters+316)
#06 pc 000000000002bf7c  /vendor/lib64/libIMGegl.so (IMGeglQuerySurface+1296)
#07 pc 000000000001f89c  /system/lib64/libEGL.so (android::eglQuerySurfaceImpl(void*, void*, int, int*)+204)
#08 pc 000000000001c850  /system/lib64/libEGL.so (eglQuerySurface+64)
#09 pc 000000000028c770  /system/lib64/libhwui.so (android::uirenderer::renderthread::EglManager::beginFrame(void*)+128)
#10 pc 0000000000287be4  /system/lib64/libhwui.so (android::uirenderer::renderthread::CanvasContext::draw()+260)
#11 pc 000000000028a9b0  /system/lib64/libhwui.so (std::__1::__function::__func<android::uirenderer::renderthread::DrawFrameTask::postAndWait()::$_0, std::__1::allocator<android::uirenderer::renderthread::DrawFrameTask::postAndWait()::$_0>, void ()>::operator()() (.c1671e787f244890c877724752face20)+1008)
#12 pc 0000000000279dc4  /system/lib64/libhwui.so (android::uirenderer::WorkQueue::process()+644)
#13 pc 0000000000299970  /system/lib64/libhwui.so (android::uirenderer::renderthread::RenderThread::threadLoop()+560)
#14 pc 0000000000013550  /system/lib64/libutils.so (android::Thread::_threadLoop(void*)+416)
#15 pc 00000000000fc350  /apex/com.android.runtime/lib64/bionic/libc.so (__pthread_start(void*)+208)
#16 pc 000000000008e310  /apex/com.android.runtime/lib64/bionic/libc.so (__start_thread+64)
```

5 向图形缓冲区绘制内容
------------

绘制内容的操作是通过调用 SkiaOpenGLPipeline.draw 函数来完成的。

```
IRenderPipeline::DrawResult SkiaOpenGLPipeline::draw(
        const Frame& frame, const SkRect& screenDirty, const SkRect& dirty,
        const LightGeometry& lightGeometry, LayerUpdateQueue* layerUpdateQueue,
        const Rect& contentDrawBounds, bool opaque, const LightInfo& lightInfo,
        const std::vector<sp<RenderNode>>& renderNodes, FrameInfoVisualizer* profiler) {
    // ......
    
    sk_sp<SkSurface> surface(SkSurface::MakeFromBackendRenderTarget(
            mRenderThread.getGrContext(), backendRT, this->getSurfaceOrigin(), colorType,
            mSurfaceColorSpace, &props));
    
    // ......

    renderFrame(*layerUpdateQueue, dirty, renderNodes, opaque, contentDrawBounds, surface,
                SkMatrix::I());
	// ......
    
    {
        ATRACE_NAME("flush commands");
        surface->flushAndSubmit();
    }
    
    // ......
}
```

继续调用其父类 SkiaPipeline 的 renderFrame 函数。

### 5.1 SkiaPipeline.renderFrame

```
void SkiaPipeline::renderFrame(const LayerUpdateQueue& layers, const SkRect& clip,
                                 const std::vector<sp<RenderNode>>& nodes, bool opaque,
                                 const Rect& contentDrawBounds, sk_sp<SkSurface> surface,
                                 const SkMatrix& preTransform) {
      // ......
  
      renderFrameImpl(clip, nodes, opaque, contentDrawBounds, canvas, preTransform);
  
      // ......
  }
```

继续调用其 SkiaPipeline.renderFrameImpl 函数。

### 5.2 SkiaPipeline.renderFrameImpl

```
void SkiaPipeline::renderFrameImpl(const SkRect& clip,
                                     const std::vector<sp<RenderNode>>& nodes, bool opaque,
                                     const Rect& contentDrawBounds, SkCanvas* canvas,
                                     const SkMatrix& preTransform) {
      // ......
  
      if (1 == nodes.size()) {
          if (!nodes[0]->nothingToDraw()) {
              RenderNodeDrawable root(nodes[0].get(), canvas);
              root.draw(canvas);
          }
      } else if (0 == nodes.size()) {
          // nothing to draw
      } else {
          // It there are multiple render nodes, they are laid out as follows:
          // #0 - backdrop (content + caption)
          // #1 - content (local bounds are at (0,0), will be translated and clipped to backdrop)
          // #2 - additional overlay nodes
          // Usually the backdrop cannot be seen since it will be entirely covered by the content.
          // While
          // resizing however it might become partially visible. The following render loop will crop
          // the
          // backdrop against the content and draw the remaining part of it. It will then draw the
          // content
          // cropped to the backdrop (since that indicates a shrinking of the window).
          //
          // Additional nodes will be drawn on top with no particular clipping semantics.
  
          // Usually the contents bounds should be mContentDrawBounds - however - we will
          // move it towards the fixed edge to give it a more stable appearance (for the moment).
          // If there is no content bounds we ignore the layering as stated above and start with 2.
  
          // ......
      }
  }
```

这里根据传入的 nodes，即 CanvasContext.mRenderNodes 的数量分为多种情况：

1）、数量为 1，那么该 RenderNode 即为 RootRenderNode，View 层级结构中的根 View 对应的 RenderNode，那么创建一个 RenderNodeDrawable 对象，并调用其 draw 函数。

2）、数量为 0，没有东西可以绘制。

3）、数量大于 1，除了 RootRenderNode 之外还有其它的 RenderNode，需要根据它们的 Z 轴高度按顺序绘制。

之前我们已经讲过 mRenderNodes 的数量大于 1 的情况了，这里我们分析最普通的情况，即只有 RootRenderNode 的情况。

### 5.3 SkDrawable.draw

RenderNodeDrawable 类的 draw 是继承其父类 SKDrawable 的：

```
// external\skia\src\core\SkDrawable.cpp
void SkDrawable::draw(SkCanvas* canvas, const SkMatrix* matrix) {
    // ......
    
    this->onDraw(canvas);

    // ......
}
```

又回到了 RenderNodeDrawable。

### 5.4 RenderNodeDrawable.ondraw

```
void RenderNodeDrawable::onDraw(SkCanvas* canvas) {
    // negative and positive Z order are drawn out of order, if this render node drawable is in
    // a reordering section
    if ((!mInReorderingSection) || MathUtils::isZero(mRenderNode->properties().getZ())) {
        this->forceDraw(canvas);
    }
}
```

### 5.5 RenderNodeDrawable.forceDraw

```
void RenderNodeDrawable::forceDraw(SkCanvas* canvas) const {
    // ......

    if (!properties.getProjectBackwards()) {
        drawContent(canvas);
        // ......
    }
    // ......
}
```

### 5.6 RenderNodeDrawable.drawContent

```
void RenderNodeDrawable::drawContent(SkCanvas* canvas) const {
    // ......

    // TODO should we let the bound of the drawable do this for us?
    const SkRect bounds = SkRect::MakeWH(properties.getWidth(), properties.getHeight());
    bool quickRejected = properties.getClipToBounds() && canvas->quickReject(bounds);
    // ......
    if (!quickRejected) {
        SkiaDisplayList* displayList = renderNode->getDisplayList().asSkiaDl();
        const LayerProperties& layerProperties = properties.layerProperties();
        // composing a hardware layer
        if (renderNode->getLayerSurface() && mComposeLayer) {
			// ......
        } else {
            if (alphaMultiplier < 1.0f) {
                // Non-layer draw for a view with getHasOverlappingRendering=false, will apply
                // the alpha to the paint of each nested draw.
                AlphaFilterCanvas alphaCanvas(canvas, alphaMultiplier);
                displayList->draw(&alphaCanvas);
            } else {
                displayList->draw(canvas);
            }
        }
    }
}
```

1）、RenderNode 本身有一个位置属性，是一个矩阵区域，通过 RenderProperties.setTop 等函数进行设置，同时也可以再设置一个裁剪区域，通过 RenderProperties.setClipBounds 函数设置。

如果 RenderProperties.getClipToBounds 为 true，那么这个 RenderNode 就会被裁剪到这两个区域的相交部分，那么如果 getClipToBounds 为 true 且两个区域不相交，则 quickRejected 为 true，表示在裁剪区域内该 RenderNode 没有可以绘制的部分，当前 RenderNode 无效无需绘制。

2）、RenderNode.getLayerSurface 返回一个屏幕外的渲染 Surface，用于渲染到层图中并将图层合成到其父图层中。一般流程下这里返回的是 null，那么后续就走到了 else 流程。

最后调用了 SkiaDisplayList.draw 函数。

### 5.7 SkiaDisplayList.draw

```
// frameworks\base\libs\hwui\pipeline\skia\SkiaDisplayList.h

	void draw(SkCanvas* canvas) { mDisplayList.draw(canvas); }

    DisplayListData mDisplayList;
```

继续调用其成员变量 mDisplayList 的 DisplayListData.draw 函数。

### 5.8 DisplayListData.draw

```
// frameworks\base\libs\hwui\RecordingCanvas.cpp

	void DisplayListData::draw(SkCanvas* canvas) const {
      SkAutoCanvasRestore acr(canvas, false);
      this->map(draw_fns, canvas, canvas->getTotalMatrix());
  }
```

看下这里的 draw_fns 的定义：

```
// All ops implement draw().
#define X(T)                                                    \
    [](const void* op, SkCanvas* c, const SkMatrix& original) { \
        ((const T*)op)->draw(c, original);                      \
    },
static const draw_fn draw_fns[] = {
#include "DisplayListOps.in"
};
#undef X
```

在 DisplayListOps.in 文件定义了所有的绘制命令：

```
// frameworks\base\libs\hwui\DisplayListOps.in

X(Flush)
X(Save)
X(Restore)
X(SaveLayer)
X(SaveBehind)
X(Concat)
X(SetMatrix)
X(Scale)
X(Translate)
X(ClipPath)
X(ClipRect)
X(ClipRRect)
X(ClipRegion)
X(ResetClip)
X(DrawPaint)
X(DrawBehind)
X(DrawPath)
X(DrawRect)
X(DrawRegion)
X(DrawOval)
X(DrawArc)
X(DrawRRect)
X(DrawDRRect)
X(DrawAnnotation)
X(DrawDrawable)
X(DrawPicture)
X(DrawImage)
X(DrawImageRect)
X(DrawImageLattice)
X(DrawTextBlob)
X(DrawPatch)
X(DrawPoints)
X(DrawVertices)
X(DrawAtlas)
X(DrawShadowRec)
X(DrawVectorDrawable)
X(DrawRippleDrawable)
X(DrawWebView)
```

继续看 DisplayListData.map 函数。

### 5.9 DisplayListData.map

```
template <typename Fn, typename... Args>
inline void DisplayListData::map(const Fn fns[], Args... args) const {
    auto end = fBytes.get() + fUsed;
    for (const uint8_t* ptr = fBytes.get(); ptr < end;) {
        auto op = (const Op*)ptr;
        auto type = op->type;
        auto skip = op->skip;
        if (auto fn = fns[type]) {  // We replace no-op functions with nullptrs
            fn(op, args...);        // to avoid the overhead of a pointless call.
        }
        ptr += skip;
    }
}
```

根据之前的分析，知道之前在 Java 层 View.onDraw 方法中调用的 Canvas 的各种 drawXXX 的 API，最终都是将绘制命令保存在了这里的以 fBytes 为起点的内存空间中了，那么这里就是从将这些绘制命令从内存空间中取出，如果绘制类型符合，那么就调用相应的绘制命令。

比如之前的 Java 层调用 Canvas.drawRect 方法绘制一个矩形，最终会调用到 DisplayListData.drawRect：

```
void DisplayListData::drawRect(const SkRect& rect, const SkPaint& paint) {
    this->push<DrawRect>(0, rect, paint);
}
```

当时向内存空间中缓存了一条 DrawRect 的命令，那么此时就可以将这条 DrawRect 命令从内存空间取出并进行调用。

```
struct DrawRect final : Op {
    static const auto kType = Type::DrawRect;
    DrawRect(const SkRect& rect, const SkPaint& paint) : rect(rect), paint(paint) {}
    SkRect rect;
    SkPaint paint;
    void draw(SkCanvas* c, const SkMatrix&) const { c->drawRect(rect, paint); }
};
```

具体调用的则是 SKCanvas.drawRect 函数，SKCanvas 的部分因为能力不够暂不分析。

### 5.10 小结

回到我们分析的起点，SkiaOpenGLPipeline.draw 函数。

```
IRenderPipeline::DrawResult SkiaOpenGLPipeline::draw(
        const Frame& frame, const SkRect& screenDirty, const SkRect& dirty,
        const LightGeometry& lightGeometry, LayerUpdateQueue* layerUpdateQueue,
        const Rect& contentDrawBounds, bool opaque, const LightInfo& lightInfo,
        const std::vector<sp<RenderNode>>& renderNodes, FrameInfoVisualizer* profiler) {
    // ......
    
    sk_sp<SkSurface> surface(SkSurface::MakeFromBackendRenderTarget(
            mRenderThread.getGrContext(), backendRT, this->getSurfaceOrigin(), colorType,
            mSurfaceColorSpace, &props));
    
    // ......

    renderFrame(*layerUpdateQueue, dirty, renderNodes, opaque, contentDrawBounds, surface,
                SkMatrix::I());
	// ......
    
    {
        ATRACE_NAME("flush commands");
        surface->flushAndSubmit();
    }
    
    // ......
}
```

1）、之前在 Java 层 View.onDraw 方法中调用的 Canvas 的各种绘制命令保存在了 DisplayListData 中，这里通过 renderFrame 函数将这些绘制命令转化为对 SKCanvas 绘制函数的调用。

2）、继续调用 SKSurface.flushAndSubmit，相当于根据上一步的 SkCanvas 的绘制命令生成 GL 指令，绘制相关内容到图形缓冲区。

6 提交图形缓冲区
---------

提交图形缓冲区的操作是通过调用 SkiaOpenGLPipeline.swapBuffers 函数完成的。

```
bool SkiaOpenGLPipeline::swapBuffers(const Frame& frame, bool drew, const SkRect& screenDirty,
                                     FrameInfo* currentFrameInfo, bool* requireSwap) {
    // ......

    if (*requireSwap && (CC_UNLIKELY(!mEglManager.swapBuffers(frame, screenDirty)))) {
        return false;
    }

    return *requireSwap;
}
```

这里继续调用了 EglManager.swapBuffers 函数：

```
bool EglManager::swapBuffers(const Frame& frame, const SkRect& screenDirty) {
    // ......
    
    eglSwapBuffersWithDamageKHR(mEglDisplay, frame.mSurface, rects, screenDirty.isEmpty() ? 0 : 1);

    // ......
}
```

这里调用了 eglApi 中的 eglSwapBuffersWithDamageKHR 函数。

```
// frameworks/native/opengl/libs/EGL/eglApi.cpp

  EGLBoolean eglSwapBuffersWithDamageKHR(EGLDisplay dpy, EGLSurface draw, EGLint* rects,
                                         EGLint n_rects) {
      ATRACE_CALL();
      clearError();
  
      egl_connection_t* const cnx = &gEGLImpl;
      return cnx->platform.eglSwapBuffersWithDamageKHR(dpy, draw, rects, n_rects);
  }
```

其实现为 egl_platform_entries.cpp 的 eglSwapBuffersWithDamageKHRImpl 函数，后续为各个平台的具体实现代码，不再叙述。

我们分析这一部分是要为了引起出后续的 BufferQueue 的 queueBuffer 和 acquireBuffer 流程，作为分析后续这两个流程的起点。

queueBuffer：

```
#00 pc 0000000000092790  /system/lib64/libgui.so (android::BufferQueueProducer::queueBuffer(int, android::IGraphicBufferProducer::QueueBufferInput const&, android::IGraphicBufferProducer::QueueBufferOutput*)+848)
#01 pc 00000000000f521c  /system/lib64/libgui.so (android::Surface::queueBuffer(ANativeWindowBuffer*, int)+1228)
#02 pc 00000000000efcec  /system/lib64/libgui.so (android::Surface::hook_queueBuffer(ANativeWindow*, ANativeWindowBuffer*, int)+92)
#03 pc 0000000000051b88  /vendor/lib64/libIMGegl.so
#04 pc 0000000000030438  /vendor/lib64/libIMGegl.so (IMGeglSwapBuffersWithDamageKHR+1372)
#05 pc 00000000000205fc  /system/lib64/libEGL.so (android::eglSwapBuffersWithDamageKHRImpl(void*, void*, int*, int)+524)
#06 pc 000000000001cc48  /system/lib64/libEGL.so (eglSwapBuffersWithDamageKHR+72)
#07 pc 000000000028c9dc  /system/lib64/libhwui.so (android::uirenderer::renderthread::EglManager::swapBuffers(android::uirenderer::renderthread::Frame const&, SkRect const&)+108)
#08 pc 0000000000280418  /system/lib64/libhwui.so (android::uirenderer::skiapipeline::SkiaOpenGLPipeline::swapBuffers(android::uirenderer::renderthread::Frame const&, bool, SkRect const&, android::uirenderer::FrameInfo*, bool*)+120)
#09 pc 0000000000287ff8  /system/lib64/libhwui.so (android::uirenderer::renderthread::CanvasContext::draw()+1304)
#10 pc 000000000028a9b0  /system/lib64/libhwui.so (std::__1::__function::__func<android::uirenderer::renderthread::DrawFrameTask::postAndWait()::$_0, std::__1::allocator<android::uirenderer::renderthread::DrawFrameTask::postAndWait()::$_0>, void ()>::operator()() (.c1671e787f244890c877724752face20)+1008)
#11 pc 0000000000279dc4  /system/lib64/libhwui.so (android::uirenderer::WorkQueue::process()+644)
#12 pc 0000000000299970  /system/lib64/libhwui.so (android::uirenderer::renderthread::RenderThread::threadLoop()+560)
#13 pc 0000000000013550  /system/lib64/libutils.so (android::Thread::_threadLoop(void*)+416)
#14 pc 00000000000fc350  /apex/com.android.runtime/lib64/bionic/libc.so (__pthread_start(void*)+208)
#15 pc 000000000008e310  /apex/com.android.runtime/lib64/bionic/libc.so (__start_thread+64)
```

acquireBuffer：

```
#00 pc 0000000000088204  /system/lib64/libgui.so (android::BufferQueueConsumer::acquireBuffer(android::BufferItem*, long, unsigned long)+388)
#01 pc 00000000000bb134  /system/lib64/libgui.so (android::ConsumerBase::acquireBufferLocked(android::BufferItem*, long, unsigned long)+84)
#02 pc 00000000000b920c  /system/lib64/libgui.so (android::BufferItemConsumer::acquireBuffer(android::BufferItem*, long, bool)+76)
#03 pc 00000000000a93c8  /system/lib64/libgui.so (android::BLASTBufferQueue::acquireNextBufferLocked(std::__1::optional<android::SurfaceComposerClient::Transaction*>)+232)
#04 pc 00000000000ac964  /system/lib64/libgui.so (android::BLASTBufferQueue::onFrameAvailable(android::BufferItem const&)+1780)
#05 pc 00000000000ba48c  /system/lib64/libgui.so (android::ConsumerBase::onFrameAvailable(android::BufferItem const&)+172)
#06 pc 0000000000087068  /system/lib64/libgui.so (android::BufferQueue::ProxyConsumerListener::onFrameAvailable(android::BufferItem const&)+104)
#07 pc 0000000000092cf0  /system/lib64/libgui.so (android::BufferQueueProducer::queueBuffer(int, android::IGraphicBufferProducer::QueueBufferInput const&, android::IGraphicBufferProducer::QueueBufferOutput*)+2224)
#08 pc 00000000000f521c  /system/lib64/libgui.so (android::Surface::queueBuffer(ANativeWindowBuffer*, int)+1228)
#09 pc 00000000000efcec  /system/lib64/libgui.so (android::Surface::hook_queueBuffer(ANativeWindow*, ANativeWindowBuffer*, int)+92)
#10 pc 0000000000051b88  /vendor/lib64/libIMGegl.so
#11 pc 0000000000030438  /vendor/lib64/libIMGegl.so (IMGeglSwapBuffersWithDamageKHR+1372)
#12 pc 00000000000205fc  /system/lib64/libEGL.so (android::eglSwapBuffersWithDamageKHRImpl(void*, void*, int*, int)+524)
#13 pc 000000000001cc48  /system/lib64/libEGL.so (eglSwapBuffersWithDamageKHR+72)
#14 pc 000000000028c9dc  /system/lib64/libhwui.so (android::uirenderer::renderthread::EglManager::swapBuffers(android::uirenderer::renderthread::Frame const&, SkRect const&)+108)
#15 pc 0000000000280418  /system/lib64/libhwui.so (android::uirenderer::skiapipeline::SkiaOpenGLPipeline::swapBuffers(android::uirenderer::renderthread::Frame const&, bool, SkRect const&, android::uirenderer::FrameInfo*, bool*)+120)
#16 pc 0000000000287ff8  /system/lib64/libhwui.so (android::uirenderer::renderthread::CanvasContext::draw()+1304)
#17 pc 000000000028a9b0  /system/lib64/libhwui.so (std::__1::__function::__func<android::uirenderer::renderthread::DrawFrameTask::postAndWait()::$_0, std::__1::allocator<android::uirenderer::renderthread::DrawFrameTask::postAndWait()::$_0>, void ()>::operator()() (.c1671e787f244890c877724752face20)+1008)
#18 pc 0000000000279dc4  /system/lib64/libhwui.so (android::uirenderer::WorkQueue::process()+644)
#19 pc 0000000000299970  /system/lib64/libhwui.so (android::uirenderer::renderthread::RenderThread::threadLoop()+560)
#20 pc 0000000000013550  /system/lib64/libutils.so (android::Thread::_threadLoop(void*)+416)
#21 pc 00000000000fc350  /apex/com.android.runtime/lib64/bionic/libc.so (__pthread_start(void*)+208)
#22 pc 000000000008e310  /apex/com.android.runtime/lib64/bionic/libc.so (__start_thread+64)
```

7 总结
----

这次主要分析了 DrawFrameTask 的 run 函数的主要内容，即绘制流程从主线程切换到渲染线程后，渲染线程做了哪些工作：

1）、之前我们构建 RenderNode 树的工作是在主线程上进行的，但是后续的绘制工作是在渲染线程上进行的，那么就需要将主线程收集到的 RenderNode 的信息同步到渲染线程，包括 Properties 和 DisplayList 等信息。

2）、绘制之前，还需要向 BufferQeueu 请求一个图形缓冲区，对应 BufferQueue 的 dequeueBuffer 流程。

3）、开始绘制，向图形缓冲区填充内容。

4）、绘制结束，得到了一个有内容的图形缓冲区，那么下一步就是把这个缓冲区推入 BufferQueue，并提交给 SurfaceFlinger 进行合成，对应 BufferQueue 的 queueBuffer 流程和 acquireBuffer 流程。

总结完毕后我们下一步要分析的内容也很明朗了，BufferQueue 机制。
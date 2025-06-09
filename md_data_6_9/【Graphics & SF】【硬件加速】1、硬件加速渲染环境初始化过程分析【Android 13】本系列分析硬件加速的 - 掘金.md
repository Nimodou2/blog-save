> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7234543902400987197)

> 本系列分析硬件加速的基础知识参考了博客： Android 应用程序 UI 硬件加速渲染技术简要介绍和学习计划_硬件 ui_罗升阳的博客 - CSDN 博客 虽然旧的 Android 版本代码和本系列分析所用的 Andro

本系列分析硬件加速的基础知识参考了博客：

[Android 应用程序 UI 硬件加速渲染技术简要介绍和学习计划_硬件 ui_罗升阳的博客 - CSDN 博客](https://link.juejin.cn/?target=https%3A%2F%2Fblog.csdn.net%2Fluoshengyang%2Farticle%2Fdetails%2F45601143 "https://blog.csdn.net/luoshengyang/article/details/45601143")

虽然旧的 Android 版本代码和本系列分析所用的 Android13 代码差异很大，但是很多设计思想和流程体系还是相通的。

在 Android 应用程序中，我们是通过 Canvas API 来绘制 UI 元素的。在硬件加速渲染环境中，这些 Canvas API 调用最终会转化为 Open GL API 调用（转化过程对应用程序来说是透明的）。由于 Open GL API 调用要求发生在 Open GL 环境中，因此在每当有新的 Activity 窗口启动时，系统都会为其初始化好 Open GL 环境。这篇文章就详细分析这个 Open GL 环境的初始化过程。

Open GL 环境也称为 Open GL 渲染上下文。一个 Open GL 渲染上下文只能与一个线程关联。在一个 Open GL 渲染上下文创建的 Open GL 对象一般来说只能在关联的 Open GL 线程中操作。这样就可以避免发生多线程并发访问发生的冲突问题。这与大多数的 UI 架构限制 UI 操作只能发生在 UI 线程的原理是差不多的。

在 Android 5.0 之前，Android 应用程序的主线程同时也是一个 Open GL 线程。但是从 Android 5.0 之后，Android 应用程序的 Open GL 线程就独立出来了，称为 Render Thread，如图所示：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/402fcac2a35d4d048002675980d7f5e9~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=416&h=549&s=16909&e=png&b=ffffff)

Render Thread 有一个 Work Queue，Main Thread 通过一个代理对象 Render Proxy 向这个 Work Queue 发送一个 drawFrame 命令，从而驱使 Render Thread 执行一次渲染操作。因此，Android 应用程序 UI 硬件加速渲染环境的初始化过程任务之一就是要创建一个 Render Thread。

一个 Android 应用程序可能存在多个 Activity 组件。在 Android 系统中，每一个 Activity 组件都是一个独立渲染的窗口。由于一个 Android 应用程序只有一个 Render Thread，因此当 Main Thread 向 Render Thread 发出渲染命令时，Render Thread 要知道当前要渲染的窗口是什么。从这个角度看，Android 应用程序 UI 硬件加速渲染环境的初始化过程任务之二就是要告诉 Render Thread 当前要渲染的窗口是什么。

Java 层的 Activity 窗口到了 Open GL 这一层，被抽象为一个 ANativeWindow，封装在 EGLSurface 中，这个 EGLSurface 描述的是一个绘图表面。一旦 Render Thread 知道了当前要渲染的窗口，它就将可以将该窗口绑定到 Open GL 渲染上下文中去，从而使得后面的渲染操作都是针对被绑定的窗口的，如图所示：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5fc8648cd5e54d6bbfb2b5484bc5f23e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=682&h=660&s=30552&e=png&b=ffffff)

将 EGLSurface 绑定到 Open GL 渲染上下文之后，就可以通过 dequeueBuffer 操作向 BufferQueue 请求一个图形缓冲区进行绘制，绘制完成后，再通过 queueBuffer 操作将这个图形缓冲区返回给 BufferQueue，最终这个图形缓冲区会被提交给 SurfaceFlinger 来进行合成显示。

接下来，我们就结合源代码分析 Android 应用程序 UI 硬件加速渲染环境的初始化过程，主要的关注点就是创建 Render Thread 的过程和创建 EGLSurface 的过程。

App 向 WMS 注册窗口的起点为 ViewRootImpl.setView，同时这个方法也是硬件加速渲染环境初始化的起点。

```
    
     * We have one child
     */
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
                
                
                if (view instanceof RootViewSurfaceTaker) {
                    mSurfaceHolderCallback =
                            ((RootViewSurfaceTaker)view).willYouTakeTheSurface();
                    if (mSurfaceHolderCallback != null) {
                        mSurfaceHolder = new TakenSurfaceHolder();
                        mSurfaceHolder.setFormat(PixelFormat.UNKNOWN);
                        mSurfaceHolder.addCallback(mSurfaceHolderCallback);
                    }
                }
                
                

                
                if (mSurfaceHolder == null) {
                    
                    
                    enableHardwareAcceleration(attrs);
                    
                }

                
            }
        }
    }


```

参数 view 描述的是当前正在创建的窗口的根 View，对于 Activity 来说就是 DecorView，DeocrView 是实现了接口 RootViewSurfaceTaker 的：

```
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks


```

willYouTakeTheSurface 表示这个 View 是否有自己的 Surface，典型的例子是 SurfaceView，由 SurfaceView 自己控制 Surface 的尺寸、格式等。

对于 DecorView，这个接口返回的是 null，那么成员变量 mSurfaceHolder 就不会被初始化，那么继续调用 enableHardwareAcceleration 方法来开启硬件加速。

```
    @UnsupportedAppUsage
    private void enableHardwareAcceleration(WindowManager.LayoutParams attrs) {
        

        
        boolean hardwareAccelerated =
                (attrs.flags & WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED) != 0;
        hardwareAccelerated = ViewDebugManager.getInstance().debugForceHWDraw(hardwareAccelerated);

        if (hardwareAccelerated) {
            
            
            
            
            
            
            
            

            final boolean forceHwAccelerated = (attrs.privateFlags &
                    WindowManager.LayoutParams.PRIVATE_FLAG_FORCE_HARDWARE_ACCELERATED) != 0;

            if (ThreadedRenderer.sRendererEnabled || forceHwAccelerated) {
                
                
                mAttachInfo.mThreadedRenderer = ThreadedRenderer.create(mContext, translucent,
                        attrs.getTitle().toString());
                
                
            }
        }
    }


```

虽然硬件加速渲染是个好东西，但是也不是每一个需要绘制 UI 的进程都必需的。这样做是考虑到两个因素。第一个因素是并不是所有的 Canvas API 都可以被 GPU 支持。如果应用程序使用到了这些不被 GPU 支持的 API，那么就需要禁用硬件加速渲染。第二个因素是支持硬件加速渲染的代价是增加了内存开销。

大部分时候都是根据 Activity 窗口自己是否请求了硬件加速渲染而决定是否要为其开启硬件加速，可以通过 AndroidManifest.xml 中为相关 Activity 定义以下属性：

```
	android.R.attr


```

为 true 来设置某个 Activity 窗口支持硬件加速，这会使窗口属性 LayoutParams 的成员变量 flags 添加 FLAG_HARDWARE_ACCELERATED 标志位。

但即使 Activity 窗口声明支持硬件加速，也是要设备本身支持硬件加速渲染才行。这里看到 ThreadedRenderer 中代表硬件加速渲染开关的成员变量 sRendererEnabled 默认是开启的：

```
    public static boolean sRendererEnabled = true;


```

最后，如果当前创建的窗口支持硬件加速渲染，那么就会调用 HardwareRenderer 类的静态成员函数 create 创建一个 HardwareRenderer 对象，并且保存在与该窗口关联的一个 AttachInfo 对象的成员变量的成员变量 mHardwareRenderer 对象。这个 HardwareRenderer 对象以后将负责执行窗口硬件加速渲染的相关操作。

```
    
     * Creates a threaded renderer using OpenGL.
     *
     * @param translucent True if the surface is translucent, false otherwise
     *
     * @return A threaded renderer backed by OpenGL.
     */
    public static ThreadedRenderer create(Context context, boolean translucent, String name) {
        return new ThreadedRenderer(context, translucent, name);
    }

    ThreadedRenderer(Context context, boolean translucent, String name) {
        super();
        
    }



```

创建一个使用 OpenGL 的线程渲染器。

继续看下其父类 HardwareRenderer 的构造方法：

```
    
     * Creates a new instance of a HardwareRenderer. The HardwareRenderer will default
     * to opaque with no light source configured.
     */
    public HardwareRenderer() {
        ProcessInitializer.sInstance.initUsingContext();
        mRootNode = RenderNode.adopt(nCreateRootRenderNode());
        mRootNode.setClipToBounds(false);
        mNativeProxy = nCreateProxy(!mOpaque, mRootNode.mNativeRenderNode);
        if (mNativeProxy == 0) {
            throw new OutOfMemoryError("Unable to create hardware renderer");
        }
        Cleaner.create(this, new DestroyContextRunnable(mNativeProxy));
        ProcessInitializer.sInstance.init(mNativeProxy);
    }


```

1）、调用 ThreadedRenderer 类的成员函数 nCreateRootRenderNode 在 Native 层创建了一个 Render Node，并且通过 Java 层的 RenderNode 类的静态成员函数 adopt 将其封装在一个 Java 层的 Render Node 中。这个 Render Node 即为窗口的 Root Render Node。

2）、调用 ThreadedRenderer 类的成员函数 nCreateProxy 在 Native 层创建了一个 Render Proxy 对象。该 Render Proxy 对象以后将负责从 Main Thread 向 Render Thread 发送命令。

4.1 android_view_ThreadedRenderer_createRootRenderNode
------------------------------------------------------

窗口的 Root Render Node 是通过调用 ThreadedRenderer 类的成员函数 nCreateRootRenderNode 创建的。这是一个 JNI 函数，由 Native 层的函数 android_view_ThreadedRenderer_createRootRenderNode 实现，如下所示：

```
static jlong android_view_ThreadedRenderer_createRootRenderNode(JNIEnv* env, jobject clazz) {
    RootRenderNode* node = new RootRenderNode(std::make_unique<JvmErrorReporter>(env));
    node->incStrong(0);
    node->setName("RootRenderNode");
    return reinterpret_cast<jlong>(node);
}


```

从这里就可以看出，窗口在 Native 层的根 RenderNode 实际上是一个 RootRenderNode 对象。

接着返回该 RootRenderNode 的地址。

4.2 创建 RenderNode
-----------------

```
	public static RenderNode adopt(long nativePtr) {
        return new RenderNode(nativePtr);
    }

    private RenderNode(long nativePtr) {
        mNativeRenderNode = nativePtr;
        NoImagePreloadHolder.sRegistry.registerNativeAllocation(this, mNativeRenderNode);
        mAnimationHost = null;
    }


```

这一步就更简单了，根据从 C/C++ 层传过来的 RootRenderNode 对象的地址创建一个 Java 层的 RenderNode 对象，将该地址保存在其 long 型的成员变量 mNativeRenderNode 中。

窗口在 Main Thread 线程中使用的 Render Proxy 对象是通过调用 ThreadedRenderer 类的成员函数 nCreateProxy 创建的。这是一个 JNI 函数，由 Native 层的函数 android_view_ThreadedRenderer_createProxy 实现，如下所示：

```
static jlong android_view_ThreadedRenderer_createProxy(JNIEnv* env, jobject clazz,
        jboolean translucent, jlong rootRenderNodePtr) {
    RootRenderNode* rootRenderNode = reinterpret_cast<RootRenderNode*>(rootRenderNodePtr);
    ContextFactoryImpl factory(rootRenderNode);
    RenderProxy* proxy = new RenderProxy(translucent, rootRenderNode, &factory);
    return (jlong) proxy;
}


```

参数 rootRenderNodePtr 指向前面创建的 RootRenderNode 对象。有了这个 RootRenderNode 对象之后，函数 android_view_ThreadedRenderer_createProxy 就创建了一个 RenderProxy 对象。

RenderProxy 对象的构造函数如下所示：

```
RenderProxy::RenderProxy(bool translucent, RenderNode* rootRenderNode,
                         IContextFactory* contextFactory)
        : mRenderThread(RenderThread::getInstance()), mContext(nullptr) {
    mContext = mRenderThread.queue().runSync([&]() -> CanvasContext* {
        return CanvasContext::create(mRenderThread, translucent, rootRenderNode, contextFactory);
    });
    mDrawFrameTask.setContext(&mRenderThread, mContext, rootRenderNode,
                              pthread_gettid_np(pthread_self()), getRenderThreadTid());
}


```

RenderProxy 类有三个重要的成员变量 mRenderThread、mContext 和 mDrawFrameTask，它们的类型分别为 RenderThread、CanvasContext 和 DrawFrameTask。其中，mRenderThread 描述的就是 Render Thread，mContext 描述的是一个画布上下文，mDrawFrameTask 描述的是一个用来执行渲染任务的 Task。接下来我们就重点分析这三个成员变量的初始化过程。

RenderProxy 类的成员变量 mRenderThread 指向的 Render Thread 是通过调用 RenderThread 类的静态成员函数 getInstance 获得的。从名字我们就可以看出，RenderThread 类的静态成员函数 getInstance 返回的是一个 RenderThread 单例。也就是说，在一个 Android 应用程序进程中，只有一个 Render Thread 存在。

为了更好地了解 Render Thread 是如何运行的，我们继续分析 Render Thread 的创建过程，如下所示：

```
RenderThread& RenderThread::getInstance() {
    [[clang::no_destroy]] static sp<RenderThread> sInstance = []() {
        sp<RenderThread> thread = sp<RenderThread>::make();
        thread->start("RenderThread");
        return thread;
    }();
    gHasRenderThreadInstance = true;
    return *sInstance;
}


```

首先创建一个 RenderThread，然后看到 RenderThread 继承自 ThreadBase，ThreadBase 又继承自 Thread：

```
class RenderThread : private ThreadBase
    

class ThreadBase : public Thread {
    PREVENT_COPY_AND_ASSIGN(ThreadBase);

public:
    ThreadBase()
            : Thread(false)
            , mLooper(new Looper(false))
            , mQueue([this]() { mLooper->wake(); }, mLock) {}

    

    void start(const char* name = "ThreadBase") { Thread::run(name); }
    
    
}


```

1）、RenderThread 类的成员变量 mLooper 指向一个 Looper 对象，Render Thread 通过它来创建一个消息驱动运行模型，类似于 Main Thread 的消息驱动运行模型。

2）、RenderThread 类的成员变量 mQueue 指向一个 WorkQueue 对象，WorkQueue 顾名思义是一个任务列表，它管理一个 WorkItem 的队列，每一个 WorkItem 代表一个待完成的任务。

3）、RenderThread 类是从 Thread 类继承下来的，当我们调用它的成员函数 start 进而调用 Thread.run 函数的时候，就会创建一个新的线程。这个新的线程的入口函数为 RenderThread 类的成员函数 threadLoop，它的实现如下所示：

```
bool RenderThread::threadLoop() {
    
    
    initThreadLocals();

    while (true) {
        waitForWork();
        processQueue();

        if (mPendingRegistrationFrameCallbacks.size() && !mFrameCallbackTaskPending) {
            mVsyncSource->drainPendingEvents();
            mFrameCallbacks.insert(mPendingRegistrationFrameCallbacks.begin(),
                                   mPendingRegistrationFrameCallbacks.end());
            mPendingRegistrationFrameCallbacks.clear();
            requestVsync();
        }

        if (!mFrameCallbackTaskPending && !mVsyncRequested && mFrameCallbacks.size()) {
            
            
            
            
            requestVsync();
        }
    }

    return false;
}


```

接下来分析一下 RenderThread 的工作机制，也就是 RenderThread.threadLoop 的主要内容。在分析 initThreadLocals 函数前，先看一下 WorkQueue 和 Looper 的部分。

6.1 WorkQueue
-------------

RenderThread 的成员变量 mWorkQueue 就是一个包含 WorkItem 类型对象的队列，WorkItem 代表着每一个任务：

```
	std::vector<WorkItem> mWorkQueue;

    struct WorkItem {
        WorkItem() = delete;
        WorkItem(const WorkItem& other) = delete;
        WorkItem& operator=(const WorkItem& other) = delete;
        WorkItem(WorkItem&& other) = default;
        WorkItem& operator=(WorkItem&& other) = default;

        WorkItem(nsecs_t runAt, std::function<void()>&& work)
                : runAt(runAt), work(std::move(work)) {}

        nsecs_t runAt;
        std::function<void()> work;
    };


```

其中 WorkItem.runAt 表示当前任务在何时将会得到处理。

### 6.1.1 添加任务

当其它线程需要调度 Render Thread，就会向它的任务队列增加一个任务，然后唤醒 Render Thread 进行处理。具体添加的方式，我们到后面一并分析。

WorkQueue 类提供了一些函数来向 WorkQueue 增加一个 WorkItem，它们的实现如下所示：

```
    template <class F>
    void postAt(nsecs_t time, F&& func) {
        enqueue(WorkItem{time, std::function<void()>(std::forward<F>(func))});
    }

    template <class F>
    void postDelayed(nsecs_t delay, F&& func) {
        enqueue(WorkItem{clock::now() + delay, std::function<void()>(std::forward<F>(func))});
    }

    template <class F>
    void post(F&& func) {
        postAt(0, std::forward<F>(func));
    }

    void enqueue(WorkItem&& item) {
        bool needsWakeup;
        {
            std::unique_lock _lock{mLock};
            auto insertAt = std::find_if(
                    std::begin(mWorkQueue), std::end(mWorkQueue),
                    [time = item.runAt](WorkItem & item) { return item.runAt > time; });
            needsWakeup = std::begin(mWorkQueue) == insertAt;
            mWorkQueue.emplace(insertAt, std::move(item));
        }
        if (needsWakeup) {
            mWakeFunc();
        }
    }


```

1）、postAt 和 postDelayed 的区别是，postDelayed 为创建的 WorkItem 的执行时间 runAt，添加了额外的延迟，即传参 delay 代表的时间。

2）、这些函数最终调用的都是 enqueue 函数，内容也很简单，找到 mWorkQueue 中的第一个执行时间比当前要入的 WorkItem 的执行时间更晚的那个 WorkItem，如果它是 mWorkQueue 队列中的第一个，说明当前要插入的 WorkItem 才是最早应该执行的 WorkItem，那么将 needsWakeup 置为 true，接着唤醒 RenderThread。另外我们从这里也能看到 WorkQueue 中的 WorkItem 是按照执行时间从早到晚进行排列的。

3）、最终调用 mWakeFunc 函数唤醒 RenderThread，后面分析会看到这个函数的内容。

### 6.1.2 处理任务

Render Thread 通过成员函数 processQueue，进而调用 WorkQueue 的成员函数 process 处理待处理的 WorkItem。

processQueue 定义在 ThreadBase.h 中：

```
	void processQueue() { mQueue.process(); }


    void process() {
        auto now = clock::now();
        std::vector<WorkItem> toProcess;
        {
            std::unique_lock _lock{mLock};
            if (mWorkQueue.empty()) return;
            toProcess = std::move(mWorkQueue);
            auto moveBack = find_if(std::begin(toProcess), std::end(toProcess),
                                    [&now](WorkItem& item) { return item.runAt > now; });
            if (moveBack != std::end(toProcess)) {
                mWorkQueue.reserve(std::distance(moveBack, std::end(toProcess)) + 5);
                std::move(moveBack, std::end(toProcess), std::back_inserter(mWorkQueue));
                toProcess.erase(moveBack, std::end(toProcess));
            }
        }
        for (auto& item : toProcess) {
            item.work();
        }
    }


```

从上面的分析中，我们知道了 WorkQueue.mWorkQueue 中的 WorkItem，是按照执行时间从早到晚进行排序的，那么这里的内容就是，从 mWorkQueue 中找到所有执行时间不小于当前时间的 WorkItem，依次执行它们的 work 函数。

work 函数的内容具体是什么，则要到下面分析向 WorkQueue 添加任务的具体场景时，才能得知。

6.2 RenderThread 线程的睡眠和唤醒
-------------------------

### 6.2.1 RenderThread 线程的睡眠

RenderThread 线程的睡眠是通过定义在 ThreadBase.h 中的 waitForWork 完成的：

```
	void waitForWork() {
        nsecs_t nextWakeup;
        {
            std::unique_lock lock{mLock};
            nextWakeup = mQueue.nextWakeup(lock);
        }
        int timeout = -1;
        if (nextWakeup < std::numeric_limits<nsecs_t>::max()) {
            timeout = ns2ms(nextWakeup - WorkQueue::clock::now());
            if (timeout < 0) timeout = 0;
        }
        int result = mLooper->pollOnce(timeout);
        LOG_ALWAYS_FATAL_IF(result == Looper::POLL_ERROR, "RenderThread Looper POLL_ERROR!");
    }

    nsecs_t nextWakeup(std::unique_lock<std::mutex>& lock) {
        if (mWorkQueue.empty()) {
            return std::numeric_limits<nsecs_t>::max();
        } else {
            return std::begin(mWorkQueue)->runAt;
        }
    }


```

waitForWork 函数的内容为，从通过 nextWakeup 函数从 WorkQueue 中拿出最先执行的那个 WorkItem，在它的执行时间到来之前睡眠。

1）、mWorkQueue 为空，表示当前没有可执行的 WorkItem，那么当前线程就一直睡眠，直到 Looper.wake 唤醒。

2）、mWorkQueue 不为空，且当前有可执行的 WorkItem，如果就拿 WorkItem 的执行时间减去当前时间，如果小于 0，说明已经到了下一个 WorkItem 该执行的时间了，那么无需等待，否则设置一个延时，到时后再唤醒当前线程。

### 6.2.2 RenderThread 线程的唤醒

RenderThread 线程的唤醒的情况有两种其实上面都已经分析过了：

1）、一是向 WorkQueue 中添加任务时，如果该任务根据执行时间排在了 WorkQueue 的首位，那么就需要唤醒当前当前线程立即处理该任务。

2）、二是 waitForWork 函数会从 WorkQueue 中拿出第一个 WorkItem，在它的执行时间到来之前睡眠，执行时间到来的时候唤醒。

3）、屏幕产生 Vsync 信号的时候，这种方式下面会看到。

这里看一下第一种情况，向 WorkQueue 添加任务的时候，唤醒线程的 mWakeFunc 函数的实现：

```
    void enqueue(WorkItem&& item) {
        bool needsWakeup;
        {
            
        }
        if (needsWakeup) {
            mWakeFunc();
        }
    }


```

WorkQueue 的成员变量 mWakeFunc 定义为：

```
	std::function<void()> mWakeFunc;

    WorkQueue(std::function<void()>&& wakeFunc, std::mutex& lock)
            : mWakeFunc(move(wakeFunc)), mLock(lock) {}


```

从 WorkQueue 的构造函数中赋值，那么往回看 WorkQueue 创建的地方：

```
		ThreadBase()
            : Thread(false)
            , mLooper(new Looper(false))
            , mQueue([this]() { mLooper->wake(); }, mLock) {}


```

看到 mWakeFunc 的内容正是：

```
	mLooper->wake()


```

用来唤醒睡眠在 ThreadBase.waitForWork 的 RenderThread 线程。

6.3 RenderThread 初始化
--------------------

回到 RenderThread 类的成员函数 threadLoop 中，我们再来看 Render Thread 在进入无限循环之前调用的 RenderThread 类的成员函数 initThreadLocals，它的实现如下所示：

```
void RenderThread::initThreadLocals() {
    setupFrameInterval();
    initializeChoreographer();
    mEglManager = new EglManager();
    mRenderState = new RenderState(*this);
    mVkManager = VulkanManager::getInstance();
    mCacheManager = new CacheManager();
}


```

RenderThread 类的成员函数 initThreadLocals 首先调用另外一个成员函数 initializeChoreographer 创建和初始化一个 mChoreographer 对象，用来接收 Vsync 信号。接着又会分别创建一个 EglManager 对象和一个 RenderState 对象，并且保存在成员变量 mEglManager 和 mRenderState 中。前者用在初始化 Open GL 渲染上下文需要用到，而后者用来记录 Render Thread 当前的一些渲染状态。

接下来我们主要关注 Choreographer 对象的创建和初始化过程，即 RenderThread 类的成员函数 initializeChoreographer 的实现，如下所示：

```
void RenderThread::initializeChoreographer() {
    LOG_ALWAYS_FATAL_IF(mVsyncSource, "Initializing a second Choreographer?");

    if (!Properties::isolatedProcess) {
        mChoreographer = AChoreographer_create();
        LOG_ALWAYS_FATAL_IF(mChoreographer == nullptr, "Initialization of Choreographer failed");
        AChoreographer_registerRefreshRateCallback(mChoreographer,
                                                   RenderThread::refreshRateCallback, this);

        
        mLooper->addFd(AChoreographer_getFd(mChoreographer), 0, Looper::EVENT_INPUT,
                       RenderThread::choreographerCallback, this);
        mVsyncSource = new ChoreographerSource(this);
    } else {
        mVsyncSource = new DummyVsyncSource(this);
    }
}


```

1）、AChoreographer_create 函数用来创建一个 AChoreographer 的实例。

2）、AChoreographer_registerRefreshRateCallback 注册一个当显示刷新率改变时执行的回调。

3）、创建的 Choreographer 对象关联的文件描述符被注册到了 Render Thread 的消息循环中。这意味着屏幕产生 Vsync 信号时，SurfaceFlinger 服务（Vsync 信号由 SurfaceFlinger 服务进行管理和分发）会通过上述文件描述符号唤醒 Render Thread。这时候 Render Thread 就会调用 RenderThread 类的静态成员函数 choreographerCallback。

4）、初始化 ChoreographerSource 类型的成员变量 mVsyncSource，可以通过该成员变量来添加 Vsync 回调。

接下来分析一下 Choreographer 的工作机制。

6.4 Choreographer 工作机制
----------------------

### 6.4.1 Vsync 信号到来流程

#### 6.4.1.1 RenderThread.choreographerCallback

RenderThread 类的成员函数 choreographerCallback 的实现如下所示：

```
int RenderThread::choreographerCallback(int fd, int events, void* data) {
    
    
    RenderThread* rt = reinterpret_cast<RenderThread*>(data);
    AChoreographer_handlePendingEvents(rt->mChoreographer, data);

    return 1;
}


```

#### 6.4.1.2 AChoreographer.AChoreographer_handlePendingEvents

AChoreographer_handlePendingEvents 定义如下所示：

```
void AChoreographer_handlePendingEvents(AChoreographer* choreographer, void* data) {
    
    
    
    Choreographer* impl = AChoreographer_to_Choreographer(choreographer);
    impl->handleEvent(-1, Looper::EVENT_INPUT, data);
}

static inline Choreographer* AChoreographer_to_Choreographer(AChoreographer* choreographer) {
    return reinterpret_cast<Choreographer*>(choreographer);
}


```

将传参 AChoreographer 强转为 Choreographer 类型，然后调用 Choreographer 的 handleEvent 函数。

由于 Choreographer 是继承 DisplayEventDispatcher 的：

```
class Choreographer : public DisplayEventDispatcher, public MessageHandler 


```

则调用的是 DisplayEventDispatcher.handleEvent。

#### 6.4.1.3 DisplayEventDispatcher.handleEvent

```
int DisplayEventDispatcher::handleEvent(int, int events, void*) {
    

    
    
    if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount, &vsyncEventData)) {
        ALOGV("dispatcher %p ~ Vsync pulse: timestamp=%" PRId64
              ", displayId=%s, count=%d, vsyncId=%" PRId64,
              this, ns2ms(vsyncTimestamp), to_string(vsyncDisplayId).c_str(), vsyncCount,
              vsyncEventData.preferredVsyncId());
        mWaitingForVsync = false;
        mLastVsyncCount = vsyncCount;
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount, vsyncEventData);
    }

    

    return 1; 
}


```

processPendingEvents 函数用来处理 DisplayEventReceiver 接收到的所有事件，包括 Vsync 信号事件，热插拔事件等，如果接收的事件中含有 Vsync 信号事件，就会返回 true，然后调用 Choreographer 的 dispatchVsync 函数。

#### 6.4.1.4 Choreographer.dispatchVsync

```
void Choreographer::dispatchVsync(nsecs_t timestamp, PhysicalDisplayId, uint32_t,
                                  VsyncEventData vsyncEventData) {
    std::vector<FrameCallback> callbacks{};
    {
        std::lock_guard<std::mutex> _l{mLock};
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        while (!mFrameCallbacks.empty() && mFrameCallbacks.top().dueTime < now) {
            callbacks.push_back(mFrameCallbacks.top());
            mFrameCallbacks.pop();
        }
    }
    mLastVsyncEventData = vsyncEventData;
    for (const auto& cb : callbacks) {
        if (cb.vsyncCallback != nullptr) {
            
            cb.vsyncCallback(reinterpret_cast<const AChoreographerFrameCallbackData*>(
                                     &frameCallbackData),
                             cb.data);
            mInCallback = false;
        } else if (cb.callback64 != nullptr) {
            cb.callback64(timestamp, cb.data);
        } else if (cb.callback != nullptr) {
            cb.callback(timestamp, cb.data);
        }
    }
}


```

Choreographer 的 dispatchVsync 函数主要内容为遍历 Choreographer 的成员变量 mFrameCallbacks，执行其中每一个 FrameCallback 对象的回调函数。Choreographer 的成员变量 mFrameCallbacks 和 FrameCallback 结构体定义如下：

```
    std::priority_queue<FrameCallback> mFrameCallbacks;

struct FrameCallback {
    AChoreographer_frameCallback callback;
    AChoreographer_frameCallback64 callback64;
    AChoreographer_vsyncCallback vsyncCallback;
    void* data;
    nsecs_t dueTime;

    
};


```

回调有三种，定义如下：

```
 * Prototype of the function that is called when a new frame is being rendered.
 * It's passed the time that the frame is being rendered as nanoseconds in the
 * CLOCK_MONOTONIC time base, as well as the data pointer provided by the
 * application that registered a callback. All callbacks that run as part of
 * rendering a frame will observe the same frame time, so it should be used
 * whenever events need to be synchronized (e.g. animations).
 */
typedef void (*AChoreographer_frameCallback)(long frameTimeNanos, void* data);


 * Prototype of the function that is called when a new frame is being rendered.
 * It's passed the time that the frame is being rendered as nanoseconds in the
 * CLOCK_MONOTONIC time base, as well as the data pointer provided by the
 * application that registered a callback. All callbacks that run as part of
 * rendering a frame will observe the same frame time, so it should be used
 * whenever events need to be synchronized (e.g. animations).
 */
typedef void (*AChoreographer_frameCallback64)(int64_t frameTimeNanos, void* data);


 * Prototype of the function that is called when a new frame is being rendered.
 * It's passed the frame data that should not outlive the callback, as well as the data pointer
 * provided by the application that registered a callback.
 */
typedef void (*AChoreographer_vsyncCallback)(
        const AChoreographerFrameCallbackData* callbackData, void* data);


```

*   AChoreographer_frameCallback，渲染新帧时调用的函数原型。 它传递了帧在 CLOCK_MONOTONIC 时基中呈现为纳秒的时间，以及注册回调的应用程序提供的数据指针。 作为渲染帧的一部分运行的所有回调将看到相同的帧时间，因此只要需要同步事件（例如动画），就应该使用它。
*   AChoreographer_frameCallback64，同上。
*   AChoreographer_vsyncCallback， 渲染新帧时调用的函数原型。 它传递了不应超过回调的帧数据，以及注册回调的应用程序提供的数据指针。

那么每次 Vsync 信号到来时，RenderThread 的 choreographerCallback 回调就会触发，最终将调用 Choreographer 的成员变量 mFrameCallbacks 中的每一个 FrameCallback 对象的回调函数。

至此，我们知道了 Vsync 信号到来时，保存在 Choreographer 的成员变量 mFrameCallbacks 中的回调都会得到执行，但是还有两个点不太明确，一是 mFrameCallbacks 中的每一个 FrameCallback 回调对象是如何添加的，二是 FrameCallback 回调的内容是什么。

### 6.4.2 发布回调

查看代码，发现 Choreographer 的成员变量 mFrameCallbacks 中的数据是通过各种 AChoreographer_postXXXCallback 函数进行填充的：

```
void AChoreographer_postFrameCallback(AChoreographer* choreographer,
                                      AChoreographer_frameCallback callback, void* data) {
    AChoreographer_to_Choreographer(choreographer)
            ->postFrameCallbackDelayed(callback, nullptr, nullptr, data, 0);
}
void AChoreographer_postFrameCallbackDelayed(AChoreographer* choreographer,
                                             AChoreographer_frameCallback callback, void* data,
                                             long delayMillis) {
    AChoreographer_to_Choreographer(choreographer)
            ->postFrameCallbackDelayed(callback, nullptr, nullptr, data, ms2ns(delayMillis));
}
void AChoreographer_postVsyncCallback(AChoreographer* choreographer,
                                      AChoreographer_vsyncCallback callback, void* data) {
    AChoreographer_to_Choreographer(choreographer)
            ->postFrameCallbackDelayed(nullptr, nullptr, callback, data, 0);
}
void AChoreographer_postFrameCallback64(AChoreographer* choreographer,
                                        AChoreographer_frameCallback64 callback, void* data) {
    AChoreographer_to_Choreographer(choreographer)
            ->postFrameCallbackDelayed(nullptr, callback, nullptr, data, 0);
}
void AChoreographer_postFrameCallbackDelayed64(AChoreographer* choreographer,
                                               AChoreographer_frameCallback64 callback, void* data,
                                               uint32_t delayMillis) {
    AChoreographer_to_Choreographer(choreographer)
            ->postFrameCallbackDelayed(nullptr, callback, nullptr, data, ms2ns(delayMillis));
}

void Choreographer::postFrameCallbackDelayed(AChoreographer_frameCallback cb,
                                             AChoreographer_frameCallback64 cb64,
                                             AChoreographer_vsyncCallback vsyncCallback, void* data,
                                             nsecs_t delay) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    FrameCallback callback{cb, cb64, vsyncCallback, data, now + delay};
    {
        std::lock_guard<std::mutex> _l{mLock};
        mFrameCallbacks.push(callback);
    }
    
}


```

拿 Vsync 信号相关的 AChoreographer_postVsyncCallback 函数举例。

在 Choreographer 初始化的函数 RenderThread.initializeChoreographer 中，我们创建了一个 ChoreographerSource 类型的对象赋值给了 RenderThread 的成员变量 mVsyncSource。

```
void RenderThread::initializeChoreographer() {
    LOG_ALWAYS_FATAL_IF(mVsyncSource, "Initializing a second Choreographer?");

    if (!Properties::isolatedProcess) {
        
        mVsyncSource = new ChoreographerSource(this);
    } else {
        mVsyncSource = new DummyVsyncSource(this);
    }
}


```

ChoreographerSource 类的定义为：

```
class ChoreographerSource : public VsyncSource {
public:
    ChoreographerSource(RenderThread* renderThread) : mRenderThread(renderThread) {}

    virtual void requestNextVsync() override {
        AChoreographer_postVsyncCallback(mRenderThread->mChoreographer,
                                         RenderThread::extendedFrameCallback, mRenderThread);
    }

    virtual void drainPendingEvents() override {
        AChoreographer_handlePendingEvents(mRenderThread->mChoreographer, mRenderThread);
    }

private:
    RenderThread* mRenderThread;
};


```

ChoreographerSource.requestNextVsync 调用了 AChoreographer_postVsyncCallback，创建了一个 FrameCallback 对象，添加到了 Choreographer 的成员变量 mFrameCallbacks 中，这里可以看到，FrameCallback 中的回调函数是 RenderThread.extendedFrameCallback，这里先记下。

接着继续看 ChoreographerSource.requestNextVsync 是谁调用的。

```
void RenderThread::requestVsync() {
    if (!mVsyncRequested) {
        mVsyncRequested = true;
        mVsyncSource->requestNextVsync();
    }
}

bool RenderThread::threadLoop() {
    
    
    initThreadLocals();

    while (true) {
        waitForWork();
        processQueue();

        if (mPendingRegistrationFrameCallbacks.size() && !mFrameCallbackTaskPending) {
            mVsyncSource->drainPendingEvents();
            mFrameCallbacks.insert(mPendingRegistrationFrameCallbacks.begin(),
                                   mPendingRegistrationFrameCallbacks.end());
            mPendingRegistrationFrameCallbacks.clear();
            requestVsync();
        }

        
    }

    return false;
}


```

看到是 RenderThread 无限循环的那部分。

RenderThread 成员变量 mPendingRegistrationFrameCallbacks 中保存了一个 IFrameCallback 类型的回调的集合，IFrameCallback 定义为：

```
class IFrameCallback {
public:
    virtual void doFrame() = 0;

protected:
    virtual ~IFrameCallback() {}
};


```

只定义了一个接口 doFrame。

这些回调通过 RenderThread.postFrameCallback 函数：

```
void RenderThread::postFrameCallback(IFrameCallback* callback) {
    mPendingRegistrationFrameCallbacks.insert(callback);
}


```

添加到了 RenderThread 成员变量 mPendingRegistrationFrameCallbacks 中，希望在下一个 Vsync 信号到来的时候执行 doFrame 函数。

1）、mPendingRegistrationFrameCallbacks 不为空，说明此时 App 注册了对 Vsync 信号的监听，希望下一帧到来的时候在 doFrame 回调函数里做一些事情。如果该集合为空，那就没必要调用 requestVsync 了。

2）、调用 ChoreographerSource.drainPendingEvents，这个我们上面分析过，和 RenderThread.choreographerCallback 流程相似，清空所有未处理的事件。

3）、将 mPendingRegistrationFrameCallbacks 中的数据添加到 mFrameCallbacks 中，然后清空 mPendingRegistrationFrameCallbacks。注意这里的成员变量 mFrameCallbacks 和 Choreographer 的成员变量 mFrameCallbacks 可不一样。

4）、调用 requestVsync 函数添加下一个 Vsync 信号的回调。

那么当下一个 Vsync 信号到来时，根据之前的分析，最终我们通过 RenderThread.requestVsync 函数添加的回调将会得到执行，也就是 RenderThread.extendedFrameCallback 函数。

### 6.4.3 FrameCallback 回调执行流程

从上面的分析我们知道了 Vsync 信号到来时，保存在 Choreographer 的成员变量 mFrameCallbacks 中的回调都会得到执行，而每次 RenderThread 唤醒后我们也通过 RenderThread.requestVsync 函数向 Choreographer 的成员变量 mFrameCallbacks 添加了回调，即 RenderThread.extendedFrameCallback 函数。

那么 Vsync 信号到来时，相应回调的完整调用堆栈为：

```
 #00 pc 00000000002986a8  /system/lib64/libhwui.so (android::uirenderer::renderthread::RenderThread::extendedFrameCallback(AChoreographerFrameCallbackData const*, void*)+72)
 #01 pc 000000000000a30c  /system/lib64/libnativedisplay.so (android::Choreographer::dispatchVsync(long, android::PhysicalDisplayId, unsigned int, android::gui::VsyncEventData)+668)
 #02 pc 00000000000bbe90  /system/lib64/libgui.so (android::DisplayEventDispatcher::handleEvent(int, int, void*)+272)
 #03 pc 0000000000298c44  /system/lib64/libhwui.so (android::uirenderer::renderthread::RenderThread::choreographerCallback(int, int, void*)+132)
 #04 pc 000000000001836c  /system/lib64/libutils.so (android::Looper::pollInner(int)+1068)
 #05 pc 0000000000017ee0  /system/lib64/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+112)
 #06 pc 0000000000279a94  /system/lib64/libhwui.so (android::uirenderer::ThreadBase::waitForWork()+180)
 #07 pc 00000000002997d8  /system/lib64/libhwui.so (android::uirenderer::renderthread::RenderThread::threadLoop()+552)
 #08 pc 0000000000013550  /system/lib64/libutils.so (android::Thread::_threadLoop(void*)+416)
 #09 pc 00000000000fc350  /apex/com.android.runtime/lib64/bionic/libc.so (__pthread_start(void*)+208)
 #10 pc 000000000008e310  /apex/com.android.runtime/lib64/bionic/libc.so (__start_thread+64)


```

我们继续看下 Vsync 信号到来的时候都做了什么。

#### 6.4.3.1 RenderThread.extendedFrameCallback

```
void RenderThread::extendedFrameCallback(const AChoreographerFrameCallbackData* cbData,
                                         void* data) {
    RenderThread* rt = reinterpret_cast<RenderThread*>(data);
    
    
    
    rt->frameCallback(vsyncId, frameDeadline, frameTimeNanos, frameInterval);
}


```

下一个 Vsync 信号到来，此时 RenderThread.extendedFrameCallback 函数触发，并且继续调用 RenderThread.frameCallback。

#### 6.4.3.2 RenderThread.frameCallback

```
void RenderThread::frameCallback(int64_t vsyncId, int64_t frameDeadline, int64_t frameTimeNanos,
                                 int64_t frameInterval) {
    mVsyncRequested = false;
    if (timeLord().vsyncReceived(frameTimeNanos, frameTimeNanos, vsyncId, frameDeadline,
                                 frameInterval) &&
        !mFrameCallbackTaskPending) {
        ATRACE_NAME("queue mFrameCallbackTask");
        mFrameCallbackTaskPending = true;
        nsecs_t runAt = (frameTimeNanos + mDispatchFrameDelay);
        queue().postAt(runAt, [=]() { dispatchFrameCallbacks(); });
    }
}


```

1）、TimeLoad.vsyncReceived 用来检验 Vsync 是不是最新的，如果不是就会返回 false。

2）、mFrameCallbackTaskPending 置为 true，表示 WorkQueue 中已经有 Task 正在等待处理了，等后续再次走到 RenderThread.threadLoop 时，就不用再继续添加 Vsync 回调了。

3）、这里通过 WorkQueue 的 postAt 函数向 WorkQueue 中添加了一个任务，当该任务的执行时间到来的时候，RenderThread.dispatchFrameCallbacks 回调函数将会得到执行。

#### 6.4.3.3 RenderThread.dispatchFrameCallbacks

```
void RenderThread::dispatchFrameCallbacks() {
    ATRACE_CALL();
    mFrameCallbackTaskPending = false;

    std::set<IFrameCallback*> callbacks;
    mFrameCallbacks.swap(callbacks);

    if (callbacks.size()) {
        
        
        requestVsync();
        for (std::set<IFrameCallback*>::iterator it = callbacks.begin(); it != callbacks.end();
             it++) {
            (*it)->doFrame();
        }
    }
}


```

1）、mFrameCallbackTaskPending 置为 false，表示 WorkQueue 中的任务已经得到处理了，后续可以继续向 WorkQueue 中添加。

2）、执行 RenderThread.mFrameCallbacks 中的每一个 IFrameCallback 对象的 doFrame 函数。

6.5 RenderThread 小结
-------------------

我们回过头来看前面分析的 RenderThread 类的成员函数 threadLoop，每当 Render Thread 被唤醒时，它都会检查 mPendingRegistrationFrameCallbacks 列表是否不为空。如果不为空，那么就会将保存在里面的 IFrameCallback 回调接口转移至由 RenderThread 类的成员变量 mFrameCallbacks 描述的另外一个 IFrameCallback 回调接口列表中，并且调用 RenderThread 类的另外一个成员函数 requestVsync 请求 SurfaceFlinger 服务在下一个 Vsync 信号到来时通知 Render Thread，以便 Render Thread 可以执行刚才被转移的 IFrameCallback 回调接口。

梳理一下，如果 RenderThread 在 threadLoop 函数结束之前，调用 requestVsync 函数请求了对 Vsync 信号进行监听，那么下一次进入 threadLoop 函数时：

1）、首先调用 ThreadBase.waitForWork 函数使当前线程睡眠。

2）、当 Vsync 信号到来时，RenderThread 会被唤醒。

3）、首先 choreographerCallback 回调函数被调用，最终通过 RenderThread.frameCallback 函数向 WorkQueue 添加了一个任务，具体内容为 RenderThread.dispatchFrameCallbacks 函数。

4）、接着从 ThreadBase.waitForWork 函数返回，调用 ThreadBase.processQueue 函数，该函数会根据任务的执行时间是否到来从 WorkQueue 中找到当前需要处理的任务，具体就是调用这些任务的 WorkItem.workItem 函数，那么上述的 RenderThread.dispatchFrameCallbacks 函数就会得到执行，最终 RenderThread.mFrameCallbacks 队列中的每一个 IFrameCallback 对象的 doFrame 函数就会在 Vsync 信号到来的时候得到执行。

在整个过程中，有两个队列是比较关键的：

*   WorkQueue，其它线程可以把那些需要在 RenderThread 运行的内容封装为一个任务放入其中，这些任务会在其执行时间到来之时得到处理。这些任务一般就是由主线程发送过来的，例如，主线程将对 DrawFrameTask 函数的调用封装为一个任务发送给 RenderThread，希望后续在 RenderThread 线程中进行下一帧的绘制。
*   RenderThread.mFrameCallbacks，这里的存储的 iFrameCallback 主要用来在 Vsync 信号到来的时候进行响应，比如在 IFrameCallback 的回调函数 doFrame 中做一些操作。

了解了 Render Thread 的创建过程之后，回到 RenderProxy 类的构造函数中：

```
RenderProxy::RenderProxy(bool translucent, RenderNode* rootRenderNode,
                         IContextFactory* contextFactory)
        : mRenderThread(RenderThread::getInstance()), mContext(nullptr) {
    mContext = mRenderThread.queue().runSync([&]() -> CanvasContext* {
        return CanvasContext::create(mRenderThread, translucent, rootRenderNode, contextFactory);
    });
    mDrawFrameTask.setContext(&mRenderThread, mContext, rootRenderNode,
                              pthread_gettid_np(pthread_self()), getRenderThreadTid());
}


```

接下来我们继续分析它的成员变量 mContext 的初始化过程，也就是画布上下文的初始化过程。

```
    mContext = mRenderThread.queue().runSync([&]() -> CanvasContext* {
        return CanvasContext::create(mRenderThread, translucent, rootRenderNode, contextFactory);
    });


```

这里是通过 WorkQueue 的 runSync 函数向 WorkQueue 添加了一个任务：

```
    template <class F>
    auto runSync(F&& func) -> decltype(func()) {
        std::packaged_task<decltype(func())()> task{std::forward<F>(func)};
        post([&task]() { std::invoke(task); });
        return task.get_future().get();
    };


```

即一个典型的 Main Thread 通过 Render Proxy 向 Render Thread 请求执行一个命令的流程。

任务的具体内容则是 CanvasContext.create 函数：

```
CanvasContext* CanvasContext::create(RenderThread& thread, bool translucent,
                                     RenderNode* rootRenderNode, IContextFactory* contextFactory) {
    auto renderType = Properties::getRenderPipelineType();

    switch (renderType) {
        case RenderPipelineType::SkiaGL:
            return new CanvasContext(thread, translucent, rootRenderNode, contextFactory,
                                     std::make_unique<skiapipeline::SkiaOpenGLPipeline>(thread));
        case RenderPipelineType::SkiaVulkan:
            return new CanvasContext(thread, translucent, rootRenderNode, contextFactory,
                                     std::make_unique<skiapipeline::SkiaVulkanPipeline>(thread));
        default:
            LOG_ALWAYS_FATAL("canvas context type %d not supported", (int32_t)renderType);
            break;
    }
    return nullptr;
}


```

看到这里对创建哪种 SkiaPipeline 进行了选择。

> `Skia`： `skia`是图像渲染库，`2D`图形绘制自己就能完成。`3D`效果（依赖硬件）由`OpenGL`、`Vulkan`、`Metal`支持。它不仅支持`2D`、`3D`，同时支持`CPU`软件绘制和`GPU`硬件加速。`Android`、`flutter`都是使用它来完成绘制。
> 
> `OpenGL`： 是一种跨平台的`3D`图形绘制规范接口。`OpenGL EL`则是专门针对嵌入式设备，如手机做了优化。
> 
> `Vulkan`: `Android`引入了`Vulkan`支持。`VulKan`是用来替换`OpenGL`的。它不仅支持`3D`，也支持`2D`，同时更加轻量级。
> 
> Android 早期通过 skia 库进行 2d 渲染，后来加入了 hwui 利用 opengl 替换 skia 进行大部分渲染工作，现在开始用 skia opengl 替换掉之前的 opengl，从 p 的代码结构上也可以看出，p 开始 skia 库不再作为一个单独的动态库 so，而是静态库 a 编译到 hwui 的动态库里，将 skia 整合进 hwui，hwui 调用 skia opengl，也为以后 hwui 使用 skia vulkan 做铺垫。

具体是 SkiaGL 还是 SkiaVulkan，则由下面的系统属性值决定：

```
 * Allows to set rendering pipeline mode to OpenGL (default), Skia OpenGL
 * or Vulkan.
 */
#define PROPERTY_RENDERER "debug.hwui.renderer"


```

我们这里是创建了一个 SkiaOpenGLPipeline 对象：

```
SkiaOpenGLPipeline::SkiaOpenGLPipeline(RenderThread& thread)
        : SkiaPipeline(thread), mEglManager(thread.eglManager()) {
    thread.renderState().registerContextCallback(this);
}


```

之前在 RenderThread.initThreadLocals 函数中，我们创建了一个 EglManager 对象，这里也赋值给了 SkiaOpenGLPipeline 的成员变量 mEglManager。

最终创建了一个 CanvasContext 对象：

```
CanvasContext::CanvasContext(RenderThread& thread, bool translucent, RenderNode* rootRenderNode,
                             IContextFactory* contextFactory,
                             std::unique_ptr<IRenderPipeline> renderPipeline)
        : mRenderThread(thread)
        , mGenerationID(0)
        , mOpaque(!translucent)
        , mAnimationContext(contextFactory->createAnimationContext(mRenderThread.timeLord()))
        , mJankTracker(&thread.globalProfileData())
        , mProfiler(mJankTracker.frames(), thread.timeLord().frameIntervalNanos())
        , mContentDrawBounds(0, 0, 0, 0)
        , mRenderPipeline(std::move(renderPipeline)) {
    rootRenderNode->makeRoot();
    mRenderNodes.emplace_back(rootRenderNode);
    mProfiler.setDensity(DeviceInfo::getDensity());
}


```

CanvasContext 成员变量 mRenderPipeline 保存的是刚刚创建的 SkiaOpenGLPipeline 对象。

回到 RenderProxy 类的构造函数中，接下来我们继续分析它的成员变量 mDrawFrameTask 的初始化过程。

```
RenderProxy::RenderProxy(bool translucent, RenderNode* rootRenderNode,
                         IContextFactory* contextFactory)
        : mRenderThread(RenderThread::getInstance()), mContext(nullptr) {
    mContext = mRenderThread.queue().runSync([&]() -> CanvasContext* {
        return CanvasContext::create(mRenderThread, translucent, rootRenderNode, contextFactory);
    });
    mDrawFrameTask.setContext(&mRenderThread, mContext, rootRenderNode,
                              pthread_gettid_np(pthread_self()), getRenderThreadTid());
}


```

RenderProxy 类的成员变量 mDrawFrameTask 描述的是一个 Draw Frame Task，Main Thread 每次都是通过它来向 Render Thread 发出渲染下一帧的命令的。

```
void DrawFrameTask::setContext(RenderThread* thread, CanvasContext* context, RenderNode* targetNode,
                               int32_t uiThreadId, int32_t renderThreadId) {
    mRenderThread = thread;
    mContext = context;
    mTargetNode = targetNode;
    mUiThreadId = uiThreadId;
    mRenderThreadId = renderThreadId;
}


```

对 Draw Frame Task 的初始化很简单，主要是将前面已经获得的 RenderThread 对象、CanvasContext 对象以及 RenderNode 对象保存在它内部，以便以后它可以直接使用相关的功能，这是通过调用 DrawFrameTask 类的成员函数 setContext 实现的。

至此，一个 RenderProxy 对象的创建过程就分析完成了，从中我们也看到 Render Thread 的创建过程和运行模型，以及 Render Proxy 与 Render Thread 的交互模型，总结来说：

1）、RenderProxy 内部有一个成员变量 mRenderThread，它指向的是一个 RenderThread 对象，通过它可以向 Render Thread 线程发送命令。

2）、RenderProxy 内部有一个成员变量 mContext，它指向的是一个 CanvasContext 对象，Render Thread 的渲染工作就是通过它来完成的。

3）、RenderProxy 内部有一个成员变量 mDrawFrameTask，它指向的是一个 DrawFrameTask 对象，Main Thread 通过它向 Render Thread 线程发送渲染下一帧的命令。

每一个 Open GL 渲染上下文都需要关联有一个 EGL Surface。这个 EGL Surface 描述的是一个绘图表面，它封装的实际上是一个 ANativeWindow。有了这个 EGL Surface 之后，我们在执行 Open GL 命令的时候，才能确定这些命令是作用在哪个窗口上。

之前分析过，当前 Activity 窗口对应的 Surface 是通过调用 ViewRootImpl 类的成员函数 relayoutWindow 向 WindowManagerService 服务请求创建和返回的，并且保存在 ViewRootImpl 类的成员变量 mSurface 中。如果这个 Surface 是新创建的，那么就会调用 ViewRootImpl 类的成员变量 mAttachInfo 指向的一个 AttachInfo 对象的成员变量 mHardwareRenderer 描述的一个 HardwareRenderer 对象的成员函数 initialize 将它绑定到 Render Thread 中。

```
                if (surfaceCreated) {
                    
                    if (mAttachInfo.mThreadedRenderer != null) {
                        try {
                            hwInitialized = mAttachInfo.mThreadedRenderer.initialize(mSurface);
                            
                        } catch (OutOfResourcesException e) {
                            
                        }
                    }
                }


```

最后，如果需要绘制当前的 Activity 窗口，那会调用 ViewRootImpl 类的另外一个成员函数 performDraw 进行绘制。

这里我们只关注绑定窗口对应的 Surface 到 Render Thread 的过程。从前面的分析可以知道，ViewRootImpl 类的成员变量 mAttachInfo 指向的一个 AttachInfo 对象的成员变量 mHardwareRenderer 保存的实际上是一个 ThreadedRenderer 对象，它的成员函数 initialize 的实现如下所示：

10.1 ThreadedRenderer.initialize
--------------------------------

```
    
     * Initializes the threaded renderer for the specified surface.
     *
     * @param surface The surface to render
     *
     * @return True if the initialization was successful, false otherwise.
     */
    boolean initialize(Surface surface) throws OutOfResourcesException {
        boolean status = !mInitialized;
        mInitialized = true;
        updateEnabledState(surface);
        setSurface(surface);
        return status;
    }

    @Override
    public void setSurface(Surface surface) {
        
        
        if (surface != null && surface.isValid()) {
            super.setSurface(surface);
        } else {
            super.setSurface(null);
        }
    }


```

ThreadedRenderer 类的成员函数 initialize 首先将成员变量 mInitialized 的值设置为 true，表明它接下来已经绑定过 Surface 到 Render Thread 了，接着再调用另外一个成员函数 setSurface。

ThreadedRenderer.setSurface 在验证了传入的 Surface 的有效性后，继续调用父类 HardwareRenderer.setSurface。

10.2 HardwareRenderer.setSurface
--------------------------------

```
    
     * <p>The surface to render into. The surface is assumed to be associated with the display and
     * as such is still driven by vsync signals such as those from
     * {@link android.view.Choreographer} and that it has a native refresh rate matching that of
     * the display's (typically 60hz).</p>
     *
     * <p>NOTE: Due to the shared, cooperative nature of the render thread it is critical that
     * any {@link Surface} used must have a prompt, reliable consuming side. System-provided
     * consumers such as {@link android.view.SurfaceView},
     * {@link android.view.Window#takeSurface(SurfaceHolder.Callback2)},
     * or {@link android.view.TextureView} all fit this requirement. However if custom consumers
     * are used such as when using {@link SurfaceTexture} or {@link android.media.ImageReader}
     * it is the app's responsibility to ensure that they consume updates promptly and rapidly.
     * Failure to do so will cause the render thread to stall on that surface, blocking all
     * HardwareRenderer instances.</p>
     *
     * @param surface The surface to render into. If null then rendering will be stopped. If
     *                non-null then {@link Surface#isValid()} must be true.
     */
    public void setSurface(@Nullable Surface surface) {
        setSurface(surface, false);
    }

    public void setSurface(@Nullable Surface surface, boolean discardBuffer) {
        if (surface != null && !surface.isValid()) {
            throw new IllegalArgumentException("Surface is invalid. surface.isValid() == false.");
        }
        nSetSurface(mNativeProxy, surface, discardBuffer);
    }


```

要渲染到的 Surface。 假定 Surface 与显示相关联，因此仍然由 vsync 信号驱动，例如来自 {@link android.view.Choreographer} 的信号，并且它具有与显示刷新率匹配的 native 层刷新率（通常为 60hz）。

注意：由于渲染线程的共享、协作性质，使用的任何 {@link Surface} 都必须具有及时、可靠的消费端，这一点至关重要。 系统提供的消费者，例如 {@link android.view.SurfaceView}、{@link android.view.Window#takeSurface(SurfaceHolder.Callback2)} 或 {@link android.view.TextureView} 都符合此要求。 但是，如果使用自定义消费者，例如在使用 {@link SurfaceTexture} 或 {@link android.media.ImageReader} 时，应用程序有责任确保他们及时、快速地使用更新。 如果不这样做，将导致渲染线程在该 Surface 上停止，从而阻塞所有 HardwareRenderer 实例。

参数 surface 为要渲染到的 Surface。 如果为 null，则渲染将停止。 如果非空，则 {@link Surface#isValid()} 必须为真。

接着调用 JNI 函数 nSetSurface：

```
        {"nSetSurface", "(JLandroid/view/Surface;Z)V",
         (void*)android_view_ThreadedRenderer_setSurface}


```

10.3 android_view_ThreadedRenderer_setSurface
---------------------------------------------

```
static void android_view_ThreadedRenderer_setSurface(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jobject jsurface, jboolean discardBuffer) {
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    ANativeWindow* window = nullptr;
    if (jsurface) {
        window = fromSurface(env, jsurface);
    }
    
    
    
    proxy->setSurface(window, enableTimeout);
    if (window) {
        ANativeWindow_release(window);
    }
}


```

参数 proxyPtr 描述的就是之前所创建的一个 RenderProxy 对象，而参数 jsurface 描述的是要绑定给 Render Thread 的 Java 层的 Surface。前面提到，Java 层的 Surface 在 Native 层对应的是一个 ANativeWindow。我们可以通过函数 fromSurface 来获得一个 Java 层的 Surface 在 Native 层对应的 ANativeWindow。

接下来，就可以通过 RenderProxy 类的成员函数 setSurface 将前面获得的 ANativeWindow 绑定到 Render Thread 中。

10.4 RenderProxy.setSurface
---------------------------

```
void RenderProxy::setSurface(ANativeWindow* window, bool enableTimeout) {
    if (window) { ANativeWindow_acquire(window); }
    mRenderThread.queue().post([this, win = window, enableTimeout]() mutable {
        mContext->setSurface(win, enableTimeout);
        if (win) { ANativeWindow_release(win); }
    });
}


```

从前面的分析可以知道，RenderProxy 类的成员函数 setSurface 向 Render Thread 的 WorkQueue 发送了一个任务。当这个任务在 Render Thread 中执行时，以下代码段就会执行：

```
        mContext->setSurface(win, enableTimeout);


```

RenderProxy 类的成员变量 mContext 指向的一个 CanvasContext 对象，而参数 win 指向的 ANativeWindow 就是要绑定到 Render Thread 的 ANativeWindow。那么这里继续调用 CanvasContext.setSurface。

10.5 CanvasContext.setSurface
-----------------------------

```
void CanvasContext::setSurface(ANativeWindow* window, bool enableTimeout) {
    
    
    if (window) {
        mNativeSurface = std::make_unique<ReliableSurface>(window);
        mNativeSurface->init();
        
    } else {
        mNativeSurface = nullptr;
    }    
    setupPipelineSurface();
}

void CanvasContext::setupPipelineSurface() {
    bool hasSurface = mRenderPipeline->setSurface(
            mNativeSurface ? mNativeSurface->getNativeWindow() : nullptr, mSwapBehavior);

    
}


```

将 ANativeWindow 类型的传参 window 保存到 CanvasContext 成员变量 mNativeSurface 中，后续可以通过 ReliableSurface.getNativeWindow 函数拿到该 ANativeWindow。

10.6 SkiaOpenGLPipeline.setSurface
----------------------------------

```
bool SkiaOpenGLPipeline::setSurface(ANativeWindow* surface, SwapBehavior swapBehavior) {
    

    if (surface) {
        mRenderThread.requireGlContext();
        auto newSurface = mEglManager.createSurface(surface, mColorMode, mSurfaceColorSpace);
        if (!newSurface) {
            return false;
        }
        mEglSurface = newSurface.unwrap();
    }

    
}


```

根据之前的分析，我们知道 RenderThread 类以及 SkiaOpenGLPipeline 类的成员变量 mEglManager 是指向前面我们分析 RenderThread 类的成员函数 initThreadLocals 时创建的一个 EglManager 对象。

1）、首先调用 RenderThread.requireGlContext 函数，对 EglManager 进行了初始化。

2）、通过 SkiaOpenGLPipeline 的成员变量 mEglManager 的 createSurface 创建了一个 EGLSurface，并且保存在了 SkiaOpenGLPipeline 的成员变量 mEglSurface 中，即将参数 surface 描述的 ANativeWindow 封装成一个 EGL Surface。

10.7 EglManager.initialize
--------------------------

```
void RenderThread::requireGlContext() {
    if (mEglManager->hasEglContext()) {
        return;
    }
    mEglManager->initialize();

    
}

void EglManager::initialize() {
    
    mEglDisplay = eglGetDisplay(EGL_DEFAULT_DISPLAY);
    LOG_ALWAYS_FATAL_IF(mEglDisplay == EGL_NO_DISPLAY, "Failed to get EGL_DEFAULT_DISPLAY! err=%s",
                        eglErrorString());

    EGLint major, minor;
    LOG_ALWAYS_FATAL_IF(eglInitialize(mEglDisplay, &major, &minor) == EGL_FALSE,
                        "Failed to initialize display %p! err=%s", mEglDisplay, eglErrorString());

    
    loadConfigs();
    createContext();
    createPBufferSurface();
    makeCurrent(mPBufferSurface, nullptr,  true);

    
}


```

RenderThread.requireGlContext 继续调用了 EglManager.initialize 完成了 EglManager 的初始化，主要为：

1）、调用 **eglGetDisplay** 函数获取一个 EGLDisplay 对象。

2）、调用 **eglInitialize** 初始化与 EGLDisplay 之间的连接。

3）、调用 loadConfigs 函数，最终调用 **eglChooseConfig** 函数获取一个 ELGConfig 对象。

4）、调用 createContext 函数，最终调用 **eglCreateContext** 函数获取一个 EGLContext 对象。

5）、调用 createPBufferSurface 函数，最终调用 **eglCreatePbufferSurface** 函数获取一个 ELGSurface 对象，该 EGLSurface 用于离屏渲染。

6）、调用 makeCurrent 函数，最终调用 **eglMakeCurrent** 函数将 EGLSurface 和 EGLContext 绑定。

这一步通过各种 eglXXX 函数初始化了渲染环境，实际上就是通过 EGL 函数建立了从 Open GL 到底层 OS 图形系统的桥梁，这一点应该怎么理解呢？Open GL 是一套与 OS 无关的规范，不过当它在一个具体的 OS 实现时，仍然是需要与 OS 的图形系统打交道的。例如，Open GL 需要从底层的 OS 图形系统中获得图形缓冲区来保存渲染结果，并且也需要将渲染好的图形缓冲区交给底层的 OS 图形系统来显示到设备屏幕去。Open GL 与底层的 OS 图形系统的这些交互通道都是通过 EGL 函数来建立的。

但是注意这里 EGLContext 绑定了一个 PbufferSurface，而非我们真正绘图的 Surface。

10.8 EglManager.createSurface
-----------------------------

```
Result<EGLSurface, EGLint> EglManager::createSurface(EGLNativeWindowType window,
                                                     ColorMode colorMode,
                                                     sk_sp<SkColorSpace> colorSpace) {
    
        
    EGLSurface surface = eglCreateWindowSurface(mEglDisplay, config, window, attribs);
    
    
}


```

通过 eglCreateWindowSurface 创建了一个 EGLSurface，但是没有调用 eglMakeCurrent 函数将该 EGLSurface 和 EGLContext 进行绑定。真正的绑定，应该要到绘制的时候再进行，而非 Surface 创建的时候，后续我们分析绘制流程的时候会看到。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ba2a595e2684013878a5b635599366c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2291&h=1125&s=181608&e=png&b=ffffff)

参考：[一看就懂的 OpenGL 基础概念（2）：EGL，OpenGL 与设备的桥梁丨音视频基础 - 知乎 (zhihu.com)](https://link.juejin.cn/?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F579253989 "https://zhuanlan.zhihu.com/p/579253989")

我们知道 OpenGL 是一组可以操作 GPU 的 API，然而仅仅能够操作 GPU，并不能够将图像渲染到设备的显示窗口上。那么，就需要一个中间层，连接 OpenGL 与设备窗口，并且最好是跨平台的。于是 EGL 出现了，由 Khronos Group 提供的一组平台无关的 API，保证了 OpenGL ES 的平台独立性。EGL 是 OpenGL ES 渲染 API 和本地窗口系统 (native platform window system) 之间的一个中间接口层，EGL 作为 OpenGL ES 与显示设备的 ** 桥梁，** 让 OpenGL ES 绘制的内容能够在呈现当前设备上。它主要由系统制造商实现。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e86068b3419141158720699b28e7d78a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1024&h=768&s=216039&e=png&a=1&b=09abf1)

各部分分别为：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/60b1b7a45ace4b80a749c56398bc9e58~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1024&h=557&s=46044&e=jpg&b=fbf0ed)

1）、EGLDisplay，EGL 定义的一个抽象的系统显示类，用于操作设备窗口。

2）、EGLSurface，渲染区域，相当于 OpenGL ES 绘图的画布 （一块内存空间），用户想绘制的信息首先都要先绘制到 EGLSurface 上，然后通过 EGLDisplay 显示，有三种类型：

*   Surface – 可显示的 Surface，实际上就是一个 FrameBuffer，用于绑定窗口后预览显示。
*   PixmapSurface – 不是可显示的 Surface，保存在系统内存中的位图。
*   PBufferSurface – 不是可显示的 Surface，保存在显存中的帧，用于离屏渲染，不需要绑定窗口。

3）、EGLContext，OpenGL 上下文，保存了在这个 Context 下发起的 GL 调用指令。

初始化 EGL 的过程其实就是配置以上几个信息的过程。

在 EGL 初始化以后，即渲染环境（EGLDisplay、EGLContext、EGLSurface）准备就绪以后，需要在渲染线程（绘制图像的线程）中，明确的调用 eglMakeCurrent。这时，系统底层会将 OpenGL 渲染环境绑定到当前线程。eglMakeCurrent 这个方法，实现了设备显示窗口（EGLDisplay）、 OpenGL 上下文（EGLContext）、图像数据缓存（EGLSurface） 、当前线程的绑定。

在这之后，只要你是在渲染线程中调用任何 OpenGL ES 的 API（比如生产纹理 ID 的方法 GLES20.glGenTextures），OpenGL 会自动根据当前线程，切换上下文（也就是切换 OpenGL 的渲染信息和资源）。
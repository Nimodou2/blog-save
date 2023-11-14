> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/130754171?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/e165e6acd8684f258ff9f13e2e37a91a.jpeg#pic_center)

#### 文章目录

*   [1 ViewRootImpl.setView](#1_ViewRootImplsetView_7)
*   [2 Session.addToDisplayAsUser](#2_SessionaddToDisplayAsUser_74)
*   [3 WindowManagerService.addWindow](#3_WindowManagerServiceaddWindow_101)
*   [4 WindowState.attach](#4_WindowStateattach_124)
*   [5 Session.windowAddedLocked](#5_SessionwindowAddedLocked_135)
*   [6 SurfaceSession.init](#6_SurfaceSessioninit_171)
*   [7 android_view_SurfaceSession.nativeCreate](#7_android_view_SurfaceSessionnativeCreate_183)
*   [8 SurfaceComposerClient.onFirstRef](#8_SurfaceComposerClientonFirstRef_228)
*   *   [8.1 获取 ISurfaceComposer 的代理对象](#81_ISurfaceComposer_250)
    *   [8.2 获取 ISurfaceComposerClient 的代理对象](#82_ISurfaceComposerClient_303)
*   [9 总结](#9__346)

不管是通过启动 Activity 的方式来创建 App 类型的窗口，还是通过主动调用 WindowManager.addView 的方式来创建非 App 类型的窗口，流程都是一样，最终都是通过 ViewRootImpl 与 WMS 通信来创建一个窗口。

从 App 进程创建第一个窗口开始，分析 App 是如何与 [SurfaceFlinger](https://so.csdn.net/so/search?q=SurfaceFlinger&spm=1001.2101.3001.7020) 服务建立起连接的。

1 ViewRootImpl.setView
----------------------

```
/**
     * We have one child
     */
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
        synchronized (this) {
            if (mView == null) {
				// ......
                try {
					// ......
                    res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId,
                            mInsetsController.getRequestedVisibility(), inputChannel, mTempInsets,
                            mTempControls);
                    // ......
	}
```

这里调用 mWindowSession.addToDisplayAsUser 向 WindowManager 添加一个窗口。

ViewRootImpl 的成员变量 mWindowSession 在 ViewRootImpl 创建的时候初始化，通过 static 类型的 WindowManagerGlobal.getWindowSession 方法得到：

```
@UnsupportedAppUsage
    public static IWindowSession getWindowSession() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowSession == null) {
                try {
                    // Emulate the legacy behavior.  The global instance of InputMethodManager
                    // was instantiated here.
                    // TODO(b/116157766): Remove this hack after cleaning up @UnsupportedAppUsage
                    InputMethodManager.ensureDefaultInstanceForDefaultDisplayIfNecessary();
                    IWindowManager windowManager = getWindowManagerService();
                    sWindowSession = windowManager.openSession(
                            new IWindowSessionCallback.Stub() {
                                @Override
                                public void onAnimatorScaleChanged(float scale) {
                                    ValueAnimator.setDurationScale(scale);
                                }
                            });
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowSession;
        }
    }
```

可以看到 WindowManagerGlobal.sWindowSession 是[单例](https://so.csdn.net/so/search?q=%E5%8D%95%E4%BE%8B&spm=1001.2101.3001.7020)的。

IWIndowSession 接口的定义是：

```
/**
 * System private per-application interface to the window manager.
 *
 * {@hide}
 */
interface IWindowSession {
```

是每一个 App 进程与 WindowManager 的交互的接口。

2 Session.addToDisplayAsUser
----------------------------

IWindowSession 的服务端实现是 Session 类：

```
/**
 * This class represents an active client session.  There is generally one
 * Session object per process that is interacting with the window manager.
 */
class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
```

Session 类代表了一个活跃的客户端会话。通常是每一个和 WindowManager 交互的进程有一个 Session 对象。

```
@Override
    public int addToDisplayAsUser(IWindow window, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, int userId, InsetsState requestedVisibility,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls) {
        return mService.addWindow(this, window, attrs, viewVisibility, displayId, userId,
                requestedVisibility, outInputChannel, outInsetsState, outActiveControls);
    }
```

这里继续调用了 WindowManagerService.addWindow 方法去添加窗口。

3 WindowManagerService.addWindow
--------------------------------

```
public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
            int displayId, int requestUserId, InsetsState requestedVisibility,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls) {
		// ......
        synchronized (mGlobalLock) {
        	// ......
			final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], attrs, viewVisibility, session.mUid, userId,
                    session.mCanAddInternalSystemWindow);
            // ......	
            win.attach();
            // ......
        }
        // ......
    }
```

首先基于这个 Session 对象创建了一个 WindowState 对象，然后调用 WindowState.attach 方法。

4 WindowState.attach
--------------------

```
void attach() {
        if (DEBUG) Slog.v(TAG, "Attaching " + this + " token=" + mToken);
        mSession.windowAddedLocked();
    }
```

mSession 就是上面一直提到的那个 Session 对象，这里调用 Session.windowAddedLocked 方法。

5 Session.windowAddedLocked
---------------------------

```
void windowAddedLocked() {
		// ......
        if (mSurfaceSession == null) {
            if (DEBUG) {
                Slog.v(TAG_WM, "First window added to " + this + ", creating SurfaceSession");
            }
            mSurfaceSession = new SurfaceSession();
            ProtoLog.i(WM_SHOW_TRANSACTIONS, "  NEW SURFACE SESSION %s", mSurfaceSession);
            mService.mSessions.add(this);
            if (mLastReportedAnimatorScale != mService.getCurrentAnimatorScale()) {
                mService.dispatchNewAnimatorScaleLocked(this);
            }
        }
        mNumWindow++;
    }
```

如果是该进程第一次添加窗口，这里的 mSurfaceSession 应该是 NULL。那么这里首先会创建一个 SurfaceSession 对象，然后保存到 Session 的 SurfaceSession 类型的成员变量 mSurfaceSession 中。这里看到 SurfaceSession 也是单例的。

SurfaceSession 类的定义是：

```
/**
 * An instance of this class represents a connection to the surface
 * flinger, from which you can create one or more Surface instances that will
 * be composited to the screen.
 * {@hide}
 */
public final class SurfaceSession {
```

SurfaceSession 的一个实例代表了一个与 SurfaceFlinger 的连接，通过这个连接你可以创建一个或者多个将会合成到屏幕的 [Surface](https://so.csdn.net/so/search?q=Surface&spm=1001.2101.3001.7020)。

6 SurfaceSession.init
---------------------

```
/** Create a new connection with the surface flinger. */
    @UnsupportedAppUsage
    public SurfaceSession() {
        mNativeClient = nativeCreate();
    }
```

SurfaceSession 的构造方法很简单，调用 JNI 层的相关接口 nativeCreate 来创建一个到 SurfaceFlinger 的连接，用 long 型的成员变量 mNativeClient 保存从底层返回的一个指向 SurfaceComposerClient 的指针地址。

7 android_view_SurfaceSession.nativeCreate
------------------------------------------

```
static jlong nativeCreate(JNIEnv* env, jclass clazz) {
    SurfaceComposerClient* client = new SurfaceComposerClient();
    client->incStrong((void*)nativeCreate);
    return reinterpret_cast<jlong>(client);
}
```

1）、首先创建一个 SurfaceComposerClient 对象。

SurfaceComposerClient 的构造函数很简单，初始化了一个状态值为 NO_INIT：

```
SurfaceComposerClient::SurfaceComposerClient() : mStatus(NO_INIT) {}
```

2）、接着调用 RefBase.incStrong 增加强引用计数，这会导致 SurfaceComposerClient 的 onFirstRef 函数被调用。

onFirstRef 函数的调用原理则是涉及到了 c++ 智能指针的概念。

查阅资料：https://blog.csdn.net/lewif/article/details/50539644

> 首先 sp 和 wp 是针对 c++ 而设计的，因为 Java 根本就没有指针的概念，替使用者减少了很多不必要的麻烦。在指针的使用过程中，如果处理不妥当，甚至会出现程序的崩溃：
> 
> 1.  指针没有初始化；
> 2.  new 了对象后没有 delete；
> 3.  ptr1 和 ptr2 同时指向了对象 A，当释放了 ptr1，同时 ptr1=NULL 后，ptr2 并不知道它所指向的对象不存在了。  
>     关于对象存在多个引用的问题， android 设计了两个引用计数相关的类，LightRefBase 和 RefBase。我们能够想到对象的引用计数应该是保存在对象中的，所以我们在使用引用计数类时，只需要让普通类去继承 LightRefBase 和 RefBase 即可，这样普通类产生的对象中就自然而然拥有了引用计数。
> 
> 关于对象存在多个引用的问题， android 设计了两个引用计数相关的类，LightRefBase 和 RefBase。我们能够想到对象的引用计数应该是保存在对象中的，所以我们在使用引用计数类时，只需要让普通类去继承 LightRefBase 和 RefBase 即可，这样普通类产生的对象中就自然而然拥有了引用计数。
> 
> 在 frameworks 中有部分类都实现了 onFirstRef 函数（需要使用智能指针的类重写了父类 RefBase 的 onFirstRef 函数），这个非常有用，当强指针在第一次引用的时候，会去调用这个函数。

由于 SurfaceComposerClient 类是继承 RefBase 的：

```
class SurfaceComposerClient : public RefBase
```

那么这里我们创建了一个 SurfaceComposerClient 对象，然后赋值给局部变量 client，接着调用父类 RefBase 的 incStrong 函数，这时候会增加强引用计数，导致 SurfaceComposerClient 的 onFirstRef() 函数得到调用。

3）、最后使用 reinterpret_cast 将指向刚刚创建的 SurfaceComposerClient 对象的指针转换为一个整型返回给上层的 SurfaceSession。

8 SurfaceComposerClient.onFirstRef
----------------------------------

```
void SurfaceComposerClient::onFirstRef() {
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != nullptr && mStatus == NO_INIT) {
        sp<ISurfaceComposerClient> conn;
        conn = sf->createConnection();
        if (conn != nullptr) {
            mClient = conn;
            mStatus = NO_ERROR;
        }
    }
}
```

这个函数主要涉及两个方面的内容：

1）、获取到 ISurfaceComposer 的代理对象。

2）、调用 ISurfaceComposer.createConnection 方法建立起到 SurfaceFlinger 的连接。

### 8.1 获取 ISurfaceComposer 的代理对象

这里是调用了 ComposerService.getComposerService 来获取 ISurfaceComposer 的远程代理对象：

```
/*static*/ sp<ISurfaceComposer> ComposerService::getComposerService() {
    ComposerService& instance = ComposerService::getInstance();
    Mutex::Autolock _l(instance.mLock);
    if (instance.mComposerService == nullptr) {
        if (ComposerService::getInstance().connectLocked()) {
            ALOGD("ComposerService reconnected");
        }
    }
    return instance.mComposerService;
}
```

因为 ComposerService 类是单例模式：

```
class ComposerService : public Singleton<ComposerService>
```

所以当我们第一次调用 ComposerService 的静态函数 getInstance 的时候，就会走进它的构造函数中：

```
ComposerService::ComposerService()
: Singleton<ComposerService>() {
    Mutex::Autolock _l(mLock);
    connectLocked();
}
```

这里又调用了 ComposerService 的 connectLocked 函数：

```
bool ComposerService::connectLocked() {
    const String16 name("SurfaceFlinger");
    mComposerService = waitForService<ISurfaceComposer>(name);
    if (mComposerService == nullptr) {
        return false; // fatal error or permission problem
    }

    // ......
}
```

这里调用了 native 层的 ServiceManager 去获取名为 “SurfaceFlinger” 的服务，这里能看到 ISurfaceComposer 的服务端实现便是 SurfaceFlinger 服务。

如果能够成功返回一个 ISurfaceComposer 对象，即 BpSurfaceComposer 类型的 SurfaceFlinger 服务的远程代理，那么将 ComposerService 的类型为 sp<ISurfaceComposer> 的成员变量 mComposerService 指向这个 BpSurfaceComposer 类型的代理对象。

这样 SurfaceComposerClient 后续便可以通过 getComposerService 函数直接拿到 SurfaceFlinger 的代理对象。

### 8.2 获取 ISurfaceComposerClient 的代理对象

回到 SurfaceComposerClient.onFirstRef，经过 8.1 的操作，此时我们拿到了 SurfaceFlinger 服务的 Binder 代理对象，保存在了 ComposerService 的成员变量 mComposerService 中，它是 BpSurfaceComposer 类型的。

接着调用 BpSurfaceComposer.createConnection 函数来创建一个到 SurfaceFlinger 服务的连接，那么这最终会跨 Binder 来到 BnSurfaceComposer 处，即 SurfaceFlinger：

```
sp<ISurfaceComposerClient> SurfaceFlinger::createConnection() {
    const sp<Client> client = new Client(this);
    return client->initCheck() == NO_ERROR ? client : nullptr;
}
```

这里看到在 SurfaceFlinger，创建了一个 Client 类型的对象，传入的是 SurfaceFlinger 自己：

```
Client::Client(const sp<SurfaceFlinger>& flinger)
    : mFlinger(flinger)
{}
```

在 Client 内部则通过 sp<SurfaceFlinger> 类型的成员变量 mFlinger 保存了一个 SurfaceFlinger 的引用，Client 也可以直接操作 SurfaceFlinger。

Client 是 ISurfaceComposerClient 的本地实现，即 BnSurfaceComposerClient 类型的：

```
class Client : public BnSurfaceComposerClient
```

```
class BnSurfaceComposerClient : public SafeBnInterface<ISurfaceComposerClient> {
```

返回给 SurfaceComposerClient 的则是 BpSurfaceComposerClient 类型的 Binder 代理对象，并保存在了 SurfaceComposerClient 的类型为 sp<ISurfaceComposerClient> 成员变量 mClient 中。

这样来看，Client 算是每一个客户端进程在 SurfaceFlinger 服务端的一个代表，其数据结构相对简单，主要保存着属于该客户端进程的一些信息，如 Layer 信息：

```
DefaultKeyedVector< wp<IBinder>, wp<Layer> > mLayers;
```

那么通过这一步，客户端进程通过在 SurfaceFlinger 创建了一个代表着该客户端进程的 Client 对象，算是在 SurfaceFlinger 处有了自己的一席之地，建立起了和 SurfaceFlinger 的连接。

9 总结
----

这个时候需要一张类图：

![](https://img-blog.csdnimg.cn/a0bdc4ffa7114273b857e685c987baf5.png#pic_center)

1）、App 通过 IWindowSession 接口和 WindowManagerService 进行交互，IWindowSession 在服务端的实现是 Session 对象，每一个 App 进程在 WindowManagerService 都有一个对应的 Session 对象。

2）、App 第一次添加窗口时，Session 对象会创建一个 SurfaceSession 对象，SurfaceSession 的构造方法中会调用 JNI 方法创建一个 Native 层的 SurfaceComposerClient 对象，并且 SurfaceSession 内部会有一个 long 型的成员变量持有这个 SurfaceComposerClient 对象的地址，后续上层便可以通过传入这个地址从而在 Native 层拿到这个 SurfaceComposerClient 对象。

3）、SurfaceComposerClient 对象在初始化的时候会去建立与 SurfaceFlinger 的连接，一是通过 ComposerService.getComposerService 函数拿到 SurfaceFlinger 的远程代理对象并保存在 ComposerService 的成员变量 mComposerService；二是通过 ISurfaceComposer.createConnection 在 SurfaceFlinger 处创建一个客户端进程的代表 Client 对象。
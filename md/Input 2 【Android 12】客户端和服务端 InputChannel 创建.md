> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/126746327?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/55b54f1c793749ecbb97680c7e31cef5.jpeg#pic_center)

输入事件最终还是要发送给窗口去处理的，那么我们就看看在 ViewRootImpl 在添加窗口的时候做了什么操作。

```
/**
     * We have one child
     */
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
        synchronized (this) {                
				InputChannel inputChannel = null;
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    inputChannel = new InputChannel();
                }
            
               ......
                    
                try {
				  ......	
                    res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId,
                            mInsetsController.getRequestedVisibility(), inputChannel, mTempInsets,
                            mTempControls);
            
				......
                    
                if (inputChannel != null) {
                    if (mInputQueueCallback != null) {
                        mInputQueue = new InputQueue();
                        mInputQueueCallback.onInputQueueCreated(mInputQueue);
                    }
                    mInputEventReceiver = new WindowInputEventReceiver(inputChannel,
                            Looper.myLooper());

            ......             
    	}
    }
```

1)、创建一个空的 InputChannel 对象，跨进程调用 WindowManagerService.addWindow 方法将这个 inputChannel 对象传给 WindowManagerService 进行初始化。

2)、如果从 WMS 返回的 inputChannel 不为空，则基于这个 inputChannel 生成一个 WindowInputEventReceiver 对象 mInputEventReceiver。  
接下来分别详细分析以上两部分的具体内容。

![](https://img-blog.csdnimg.cn/38d74e62d1974e0f89e649edd2e1ffc6.png#pic_center)

一、InputChannel 的初始化
-------------------

![](https://img-blog.csdnimg.cn/af452a6e3230467cad5df152eb8e82b2.png#pic_center)

### 1 Session.addToDisplayAsUser

ViewRootImpl 持有的 mWindowSession 是一个 IWindowSession 类型的对象，它在服务端的实现是 Session，因此这里调用的是 Session 的 addToDisplayAsUse[r 方](https://so.csdn.net/so/search?q=r%E6%96%B9&spm=1001.2101.3001.7020)法。

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

这里又调用了 WindowManagerService 的 addWindow 方法。

### 2 WindowManagerService.addWindow

```
public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
            int displayId, int requestUserId, InsetsState requestedVisibility,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls) {
        
       	......

            final boolean openInputChannels = (outInputChannel != null
                    && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
            if  (openInputChannels) {
                win.openInputChannel(outInputChannel);
            }
        	
        ......
    }
```

如果当前窗口设置了 INPUT_FEATURE_NO_INPUT_CHANNEL，代表当前窗口不希望接收输入事件；否则调用 WindowState#openInputChannel。

### 3 WindowState.openInputChannel

```
void openInputChannel(InputChannel outInputChannel) {
        if (mInputChannel != null) {
            throw new IllegalStateException("Window already has an input channel.");
        }
        String name = getName();
        mInputChannel = mWmService.mInputManager.createInputChannel(name);
        mInputChannelToken = mInputChannel.getToken();
        mInputWindowHandle.setToken(mInputChannelToken);
        mWmService.mInputToWindowMap.put(mInputChannelToken, this);
        if (outInputChannel != null) {
            mInputChannel.copyTo(outInputChannel);
        } else {
            // If the window died visible, we setup a fake input channel, so that taps
            // can still detected by input monitor channel, and we can relaunch the app.
            // Create fake event receiver that simply reports all events as handled.
            mDeadWindowEventReceiver = new DeadWindowEventReceiver(mInputChannel);
        }
    }
```

调用 InputManagerService#createInputChannel 生成 InputChannel 对象并且初始化。

### 4 InputManagerService.createInputChannel

```
/**
     * Creates an input channel to be used as an input event target.
     *
     * @param name The name of this input channel
     */
    public InputChannel createInputChannel(String name) {
        return nativeCreateInputChannel(mPtr, name);
    }
```

这个方法用来创建一个可以作为输入事件发送目标的 InputChannel。

这里调用了 [JNI](https://so.csdn.net/so/search?q=JNI&spm=1001.2101.3001.7020) 层的 InputManagerService.nativeCreateInputChannel。

### 5 com_android_server_input_InputManagerService.nativeCreateInputChannel

```
static jobject nativeCreateInputChannel(JNIEnv* env, jclass /* clazz */, jlong ptr,
                                        jstring nameObj) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);

    ScopedUtfChars nameChars(env, nameObj);
    std::string name = nameChars.c_str();

    base::Result<std::unique_ptr<InputChannel>> inputChannel = im->createInputChannel(env, name);

    if (!inputChannel.ok()) {
        std::string message = inputChannel.error().message();
        message += StringPrintf(" Status=%d", inputChannel.error().code());
        jniThrowRuntimeException(env, message.c_str());
        return nullptr;
    }

    jobject inputChannelObj =
            android_view_InputChannel_createJavaObject(env, std::move(*inputChannel));
    if (!inputChannelObj) {
        return nullptr;
    }

    android_view_InputChannel_setDisposeCallback(env, inputChannelObj,
            handleInputChannelDisposed, im);
    return inputChannelObj;
}
```

1)、调用 createInputChannel 创建 C++ 层 InputChannel 对象。

2)、基于生成的 C++ 层 inputChannel 对象生成 Java 层 InputChannel 对象 inputChannelObj 并返回。

继续追踪 C++ 层 InputChannel 的创建流程。

### 6 NativeInputManager.createInputChannel

```
base::Result<std::unique_ptr<InputChannel>> NativeInputManager::createInputChannel(
        JNIEnv* /* env */, const std::string& name) {
    ATRACE_CALL();
    return mInputManager->getDispatcher()->createInputChannel(name);
}
```

调用 C++ 层的 InputManager 的 InputDispatcher.createInputChannel 函数。

### 7 InputDispatcher.createInputChannel

```
Result<std::unique_ptr<InputChannel>> InputDispatcher::createInputChannel(const std::string& name) {
#if DEBUG_CHANNEL_CREATION
    ALOGD("channel '%s' ~ createInputChannel", name.c_str());
#endif

    // 7.1 生成InputChannel对
    std::unique_ptr<InputChannel> serverChannel;
    std::unique_ptr<InputChannel> clientChannel;
    status_t result = InputChannel::openInputChannelPair(name, serverChannel, clientChannel);

    if (result) {
        return base::Error(result) << "Failed to open input channel pair with name " << name;
    }

    { // acquire lock
        std::scoped_lock _l(mLock);
        // 7.2 创建Connection对象
        const sp<IBinder>& token = serverChannel->getConnectionToken();
        int fd = serverChannel->getFd();
        sp<Connection> connection =
                new Connection(std::move(serverChannel), false /*monitor*/, mIdGenerator);

        if (mConnectionsByToken.find(token) != mConnectionsByToken.end()) {
            ALOGE("Created a new connection, but the token %p is already known", token.get());
        }
        mConnectionsByToken.emplace(token, connection);

        std::function<int(int events)> callback = std::bind(&InputDispatcher::handleReceiveCallback,
                                                            this, std::placeholders::_1, token);

        mLooper->addFd(fd, 0, ALOOPER_EVENT_INPUT, new LooperEventCallback(callback), nullptr);
    } // release lock

    // Wake the looper because some connections have changed.
    mLooper->wake();
    return clientChannel;
}
```

该函数做了两方面工作：

1)、调用 InputChannel.openInputChannelPair 生成一对 InputChannel：服务端 serverChannel 和客户端 clientChannel。

2)、基于服务端 serverChannel 生成一个 Connection 对象，然后返回客户端 clientChannel。

#### 7.1 生成 InputChannel 对

```
std::unique_ptr<InputChannel> serverChannel;
    std::unique_ptr<InputChannel> clientChannel;
    status_t result = InputChannel::openInputChannelPair(name, serverChannel, clientChannel);
```

调用 InputChannel.openInputChannelPair 生成一对 InputChannel。

```
status_t InputChannel::openInputChannelPair(const std::string& name,
                                            std::unique_ptr<InputChannel>& outServerChannel,
                                            std::unique_ptr<InputChannel>& outClientChannel) {
    int sockets[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets)) {
        status_t result = -errno;
        ALOGE("channel '%s' ~ Could not create socket pair.  errno=%s(%d)", name.c_str(),
              strerror(errno), errno);
        outServerChannel.reset();
        outClientChannel.reset();
        return result;
    }

    int bufferSize = SOCKET_BUFFER_SIZE;
    setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));

    sp<IBinder> token = new BBinder();

    std::string serverChannelName = name + " (server)";
    android::base::unique_fd serverFd(sockets[0]);
    outServerChannel = InputChannel::create(serverChannelName, std::move(serverFd), token);

    std::string clientChannelName = name + " (client)";
    android::base::unique_fd clientFd(sockets[1]);
    outClientChannel = InputChannel::create(clientChannelName, std::move(clientFd), token);
    return OK;
}
```

这里生成了一个 socket 数组：

```
int sockets[2];
```

有必要先对 socket 做一个简单的介绍：

socket 是通信端点的抽象，一个 socket 构成一个连接的一端，而一个连接可完全由一对 socket 规定。

Unix/Linux 基本哲学之一就是 “一切皆文件”，都可以用“打开 open –> 读写 write/read –> 关闭 close” 模式来操作。Socket 就是该模式的一个实现， socket 即是一种特殊的文件，一些 [socket 函数](https://so.csdn.net/so/search?q=socket%E5%87%BD%E6%95%B0&spm=1001.2101.3001.7020)就是对其进行的操作（读 / 写 IO、打开、关闭）。

再来看 InputChannel.openInputChannelPair 到底做了什么工作：

1)、调用 socketpair() 函数创建一对无名的、相互连接的套接字。 这对套接字可以用于全双工通信，每一个套接字既可以读也可以写。

2)、setsockopt() 函数用于任意类型、任意状态套接口的设置选项值。这里为这一对套接字都设置了两个缓冲区（接收缓冲区 SO_RCVBUF 和发送缓冲区 SO_SNDBUF）。

3)、创建一个 IBinder 类型的 token，这个 token 由这一对 InputChannel，可以作为这一对 InputChannel 在 Native 层的标识。

4)、为这一对套接字创建两个 fd，serverFd 和 clientFd，那么当服务端进程通过 serverFd 把数据写到 sockets[0] 的发送缓冲区的时候，sockets[0] 就会把发送缓冲区里面的数据写到 sockets[1] 的接收缓冲区里面，客户端进程就可以读取 clientFd 得到那些数据了，相反同理。

5)、再基于以上的 IBinder 对象和 socket 描述符生成最终的 serverInputChannel 和 clientInputChannel。

现在生成了持有一对 socket 文件描述符的 InputChannel 对，那么在需要通信的时候就可以通过 InputChannel 拿到 socket 文件描述符然后调用 socket 的相关函数来发送数据，“channel” 一词本身也有通道，管道的含义，因此 C++ 层的 InputChannel 可以看作是对 socket 做的一层封装。

#### 7.2 创建 Connection 对象

```
{ // acquire lock
        std::scoped_lock _l(mLock);
        const sp<IBinder>& token = serverChannel->getConnectionToken();
        int fd = serverChannel->getFd();
        sp<Connection> connection =
                new Connection(std::move(serverChannel), false /*monitor*/, mIdGenerator);

        if (mConnectionsByToken.find(token) != mConnectionsByToken.end()) {
            ALOGE("Created a new connection, but the token %p is already known", token.get());
        }
        mConnectionsByToken.emplace(token, connection);

        std::function<int(int events)> callback = std::bind(&InputDispatcher::handleReceiveCallback,
                                                            this, std::placeholders::_1, token);

        mLooper->addFd(fd, 0, ALOOPER_EVENT_INPUT, new LooperEventCallback(callback), nullptr);
    } // release lock

    // Wake the looper because some connections have changed.
    mLooper->wake();
    return clientChannel;
```

1)、基于服务端 InputChannel 生成一个 Connection 对象，Connection 用来管理与之相关的 InputChannel 的分发状态，然后 InputDispatcher 将键值对 <token, connection> 存放到 mConnectionsByToken 中，token 是在上一步生成服务端 InputChannel 的时候创建的，mConnectionsByToken 的定义是：

```
// All registered connections mapped by input channel token.
    std::unordered_map<sp<IBinder>, sp<Connection>, StrongPointerHash<IBinder>> mConnectionsByToken
            GUARDED_BY(mLock);
```

那么后续 InputDispatcher 就可以通过 token 获取到 Connection，进而拿到 InputChannel 对象。

另外这里看到并没有为客户端 InputChannel 生成一个 Connection 对象，那么就说明 InputDispatcher.mConnectionsByToken 只保存服务端 InputChannel 对应的 Connection。

2)、生成一个 LooperEventCallback，回调触发的时候调用 InputDispatcher::handleReceiveCallback：

```
class LooperEventCallback : public LooperCallback {
public:
    LooperEventCallback(std::function<int(int events)> callback) : mCallback(callback) {}
    int handleEvent(int /*fd*/, int events, void* /*data*/) override { return mCallback(events); }

private:
    std::function<int(int events)> mCallback;
};
```

接着调用 Looper 的 addFd 函数去监听 serverChannel 对应的 fd。

> 查阅资料：Looper 提供了 addFd 函数用于添加需要监控的文件描述符，这个文件描述符由调用者指定，调用者还须指定对何种 I/O（可读还是可写）事件进行监控。另外，也可指定用于处理可 I/O 事件时的回调处理函数（及其需用到的私有数据）。可在 LooperCallback 的子类中重载 handleEvent 来实现对可 I/O 事件的处理。因此，借助于 Looper 的 pollOnce 和 addFd 函数，可以实现对文件描述符的监控。无数据到来时 pollOnce 的调用者将睡眠等待，有数据到来时其被自动唤醒，并执行指定的回调处理者（若有的话）。

那么这里 Looper 会持续监听 serverFd 的接收缓冲区，如果有数据传输，就会触发 LooperEventCallback 的 handleEvent 回调，那么最终会调用 InputDispatcher::handleReceiveCallback。

### 8 WindowState.openInputChannel

```
void openInputChannel(InputChannel outInputChannel) {
        if (mInputChannel != null) {
            throw new IllegalStateException("Window already has an input channel.");
        }
        String name = getName();
        mInputChannel = mWmService.mInputManager.createInputChannel(name);
        mInputChannelToken = mInputChannel.getToken();
        mInputWindowHandle.setToken(mInputChannelToken);
        mWmService.mInputToWindowMap.put(mInputChannelToken, this);
        if (outInputChannel != null) {
            mInputChannel.copyTo(outInputChannel);
        } else {
            // If the window died visible, we setup a fake input channel, so that taps
            // can still detected by input monitor channel, and we can relaunch the app.
            // Create fake event receiver that simply reports all events as handled.
            mDeadWindowEventReceiver = new DeadWindowEventReceiver(mInputChannel);
        }
    }
```

Native 层创建完服务端和客户端 InputChannel 后，最终会返回客户端 InputChannel，接着 JNI 层的 InputManagerService 基于该 C++ 层 InputChannel 对象生成了一个 Java 层的 InputChannel 对象，最终返回到了 WindowState#openInputChannel 处。

1)、将 WindowState 的成员变量 mInputChannel 赋值为返回的客户端 InputChannel，这样 WindowState 就持有了客户端 InputChannel。

2)、将 WindowState 的成员变量 mInputChannelToken，该 WindowState 对应的 InputWindowHandle 的成员变量 token，赋值为 InputChannel 的 token，这样上层也持有了这个 token。

3)、将键值对 <IBinder mInputChannelToken, WindowState this> 添加到 WindowManagerService 的 mInputToWindowMap 中，这样 WindowManagerService 便可以根据从 Native 层传来的 token 对象获取到对应的 WIndowState 对象。

4)、将 mInputChannel 拷贝给 outInputChannel，最终返回给 ViewRootImpl。

### 9 小结

1)、对于每一个新添加的窗口，InputDispatcher 为其创建了一对 socket，通过一对 InputChannel 封装，另外创建了一个 IBinder 类型的 token，由这两个 InputChannel 共同持有。

2)、对于服务端 InputChannel，InputDispatcher 创建了一个 Connection 对象持有这个 InputChannel，然后把键值对 <IBinder token, Connection connection> 加入到 InputDispatcher 的 mConnectionsByToken 中，这样后续可以通过 token 获取到 Connection，进而拿到 InputChannel。

3)、对于客户端 InputChannel，除了将该 InputChannel 返回给 ViewRootImpl 之外，WMS 也保存了相应的 InputChannel 和 token。

4)、通过 2 和 3 的工作，该 token 将上层和 Native 层的窗口信息串联起来，上层可以根据从 Native 层传来的 token，获取到相应的 WindowState 和客户端 InputChannel。Native 层可以根据从上层传来的 token 得到 Connection，进而得到服务端 InputChannel。

二、客户端 InputEventReceiver 的创建
----------------------------

![](https://img-blog.csdnimg.cn/eb536f22a7404b64bc067bbddb667825.png#pic_center)

上一步中，ViewRootImpl 在调用 Session#addToDisplayAsUser 之后，就完成了 InputChannel 的初始化，此时的 inputChannel 代表的是客户端的 InputChannel。

接下来分析 ViewRootImpl 里的 InputEventReceiver 是如何创建的。

```
mInputEventReceiver = new WindowInputEventReceiver(inputChannel,
                            Looper.myLooper());
```

然后基于这个客户端 InputChannel 生成一个 WindowInputEventReceiver 对象。

### 1 ViewRootImp.WindowInputEventReceiver.init

```
final class WindowInputEventReceiver extends InputEventReceiver {
        public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
            super(inputChannel, looper);
        }
```

WindowInputEventReceiver 是 ViewRootImpl 的内部类，继承 InputEventReceiver。

### 2 InputEventReceiver.init

```
/**
     * Creates an input event receiver bound to the specified input channel.
     *
     * @param inputChannel The input channel.
     * @param looper The looper to use when invoking callbacks.
     */
    public InputEventReceiver(InputChannel inputChannel, Looper looper) {
        if (inputChannel == null) {
            throw new IllegalArgumentException("inputChannel must not be null");
        }
        if (looper == null) {
            throw new IllegalArgumentException("looper must not be null");
        }

        mInputChannel = inputChannel;
        mMessageQueue = looper.getQueue();
        mReceiverPtr = nativeInit(new WeakReference<InputEventReceiver>(this),
                inputChannel, mMessageQueue);

        mCloseGuard.open("dispose");
    }
```

InputEventReceiver 构造方法中又调用了 nativeInit。

### 3 android_view_InputEventReceiver.nativeInit

```
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject inputChannelObj, jobject messageQueueObj) {
    std::shared_ptr<InputChannel> inputChannel =
            android_view_InputChannel_getInputChannel(env, inputChannelObj);
    if (inputChannel == nullptr) {
        jniThrowRuntimeException(env, "InputChannel is not initialized.");
        return 0;
    }

    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    if (messageQueue == nullptr) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }

    sp<NativeInputEventReceiver> receiver = new NativeInputEventReceiver(env,
            receiverWeak, inputChannel, messageQueue);
    status_t status = receiver->initialize();
    if (status) {
        std::string message = android::base::
                StringPrintf("Failed to initialize input event receiver.  status=%s(%d)",
                             statusToString(status).c_str(), status);
        jniThrowRuntimeException(env, message.c_str());
        return 0;
    }

    receiver->incStrong(gInputEventReceiverClassInfo.clazz); // retain a reference for the object
    return reinterpret_cast<jlong>(receiver.get());
}
```

基于传入的 Java 层的 InputChannel 对象找到 Native 层的 InputChannel 对象，然后以此构建出一个 NativeInputEventReceiver 对象：

```
NativeInputEventReceiver::NativeInputEventReceiver(
        JNIEnv* env, jobject receiverWeak, const std::shared_ptr<InputChannel>& inputChannel,
        const sp<MessageQueue>& messageQueue)
      : mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
        mInputConsumer(inputChannel),
        mMessageQueue(messageQueue),
        mBatchedInputEventPending(false),
        mFdEvents(0) {
    if (kDebugDispatchCycle) {
        ALOGD("channel '%s' ~ Initializing input event receiver.", getInputChannelName().c_str());
    }
}
```

需要关注的有：

1)、这里 mReceiverWeakGlobal 保存的是 Java 层 InputEventReceiver 的引用。

2)、将 InputChannel 存储到 mInputConsumer 成员变量中，后续可以通过 mInputConsumer 取出这个客户端 InputChannel。

最终调用 initialize 函数进行初始化。

### 4 NativeInputEventReceiver.initialize

```
status_t NativeInputEventReceiver::initialize() {
    setFdEvents(ALOOPER_EVENT_INPUT);
    return OK;
}
```

ALOOPER_EVENT_INPUT 定义在 Looper.h 中，表示当前文件描述符可读。

```
/**
     * The file descriptor is available for read operations.
     */
    ALOOPER_EVENT_INPUT = 1 << 0,
```

### 5 NativeInputEventReceiver.setFdEvents

```
void NativeInputEventReceiver::setFdEvents(int events) {
    if (mFdEvents != events) {
        mFdEvents = events;
        int fd = mInputConsumer.getChannel()->getFd();
        if (events) {
            mMessageQueue->getLooper()->addFd(fd, 0, events, this, nullptr);
        } else {
            mMessageQueue->getLooper()->removeFd(fd);
        }
    }
}
```

调用当前 NativeInputEventReceiver 的 Looper 的 addFd 函数监听客户端 InputChannel 对应的 socket 的文件描述符，并且在有数据到来时执行相应回调，这里看到回调参数传入的是 this。因为 NativeInputEventReceiver 是继承 LooperCallback 的：

```
class NativeInputEventReceiver : public LooperCallback {
```

那么一旦向服务端 InputChannel 保存的 socket 写入数据，则客户端这边的注册的 NativeInputEventReceiver 则触发 handleEvent 回调，这一点在分析 InputDispatcher 分发事件的时候再详细介绍。

三、总结
----

1)、在 Native 层创建了一个 IBinder 类型的 token，一个用于服务端和客户端通信的 socket 对，每一个 socket 都由一个 InputChannel 进行封装，即 Server 端 InputChannel 和 Client 端 InputChannel，这两个 InputChannel 持有对应 socket 的文件描述符，在需要通信的时候先获取到相关 InputChannel 对象，然后取出其中保存的 socket 文件描述符，通过 socket 相关函数向其中写入数据来进行通信。

Server 端 InputChannel 保存在 InputDispatcher 为其创建的一个 Connection 对象中，InputChannel 的 token 和该 Connection 对象一一对应。

Client 端 InputChannel 保存在窗口对应的 WindowState 中，InputChannel 的 token 和该 WindowState 一一对应。

token 很重要，它是在上层和 Native 层能够找到正确 InputChannel 的关键。

2)、Native 层在创建完 InputChannel 对后，最终给 Java 层返回了一个客户端 InputChannel，该 InputChannel 先是返回到 WM，最终跨进程返回给 ViewRootImpl。

3)、ViewRootimpl 根据上一步得到的 InputChannel 创建了一个 WindowInputEventReceiver 对象，该对象在初始化的时候在 JNI 层构建了一个 NativeInputEventReceiver 对象并将客户端 InputChannel 传给了 NativeInputEventReceiver，NativeInputEventReceiver 内部有一个 InputConsumer 成员变量保存这个客户端 InputChannel，NativeInputEventReceiver 的 Looper 通过 addFd 函数对客户端 socket 进行监听，如果服务端 socket 有写入操作那么就回调 NativeInputEventReceiver.handleEvent 函数。

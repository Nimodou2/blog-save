> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/126746264?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/cf5a844f751241d48771f0759fb8a912.jpeg#pic_center)

Input 相关服务启动的起点在 SystemServer.startOtherServices：

```
/**
     * Starts a miscellaneous grab bag of stuff that has yet to be refactored and organized.
     */
    private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
        t.traceBegin("startOtherServices");

        final Context context = mSystemContext;
        ......
        InputManagerService inputManager = null;
        ......

        try {
            ......

            t.traceBegin("StartInputManagerService");
            inputManager = new InputManagerService(context);
            t.traceEnd();

            ......

            t.traceBegin("StartInputManager");
            inputManager.setWindowManagerCallbacks(wm.getInputManagerCallback());
            inputManager.start();
            t.traceEnd();
            
            ....
        }    
    }
```

1)、创建了一个 InputManagerService 对象 inputManager。

2)、调用了 InputManagerService.start 方法。

先分析 InputManagerService 的创建流程，然后再分析 start 流程。

![](https://img-blog.csdnimg.cn/702a11b1a18b4358927522d1ee6f73a2.png#pic_center)

1 创建
----

![](https://img-blog.csdnimg.cn/b92867706eed4c209954ab2d7b71b74d.png#pic_center)

### 1.1 InputManagerService.init

```
public InputManagerService(Context context) {
        this.mContext = context;
        this.mHandler = new InputManagerHandler(DisplayThread.get().getLooper());

        mStaticAssociations = loadStaticInputPortAssociations();
        mUseDevInputEventForAudioJack =
                context.getResources().getBoolean(R.bool.config_useDevInputEventForAudioJack);
        Slog.i(TAG, "Initializing input manager, mUseDevInputEventForAudioJack="
                + mUseDevInputEventForAudioJack);
        mPtr = nativeInit(this, mContext, mHandler.getLooper().getQueue());

        String doubleTouchGestureEnablePath = context.getResources().getString(
                R.string.config_doubleTouchGestureEnableFile);
        mDoubleTouchGestureEnableFile = TextUtils.isEmpty(doubleTouchGestureEnablePath) ? null :
            new File(doubleTouchGestureEnablePath);

        LocalServices.addService(InputManagerInternal.class, new LocalService());
    }
```

初始化了一些成员变量，唯一需要注意的是 mPtr 保存了一个指向 Native 层的 input 相关服务对象的指针：

```
// Pointer to native input manager service object.
    private final long mPtr;
```

我们重点关注 nativeInit 的部分。

### 1.2 com_android_server_input_InputManagerService.nativeInit

```
static jlong nativeInit(JNIEnv* env, jclass /* clazz */,
        jobject serviceObj, jobject contextObj, jobject messageQueueObj) {
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    if (messageQueue == nullptr) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }

    NativeInputManager* im = new NativeInputManager(contextObj, serviceObj,
            messageQueue->getLooper());
    im->incStrong(0);
    return reinterpret_cast<jlong>(im);
}
```

这里调用了 JNI 层 InputManagerService 的 nativeInit 函数。

这一步创建了一个 NativeInputManager 实例。

### 1.3 NativeInputManager.init

```
NativeInputManager::NativeInputManager(jobject contextObj,
        jobject serviceObj, const sp<Looper>& looper) :
        mLooper(looper), mInteractive(true) {
    JNIEnv* env = jniEnv();

    mServiceObj = env->NewGlobalRef(serviceObj);

    {
        AutoMutex _l(mLock);
        mLocked.systemUiLightsOut = false;
        mLocked.pointerSpeed = 0;
        mLocked.pointerGesturesEnabled = true;
        mLocked.showTouches = false;
        mLocked.pointerCapture = false;
        mLocked.pointerDisplayId = ADISPLAY_ID_DEFAULT;
    }
    mInteractive = true;

    InputManager* im = new InputManager(this, this);
    mInputManager = im;
    defaultServiceManager()->addService(String16("inputflinger"), im);
}
```

这里创建了一个 Native 层的 InputManager 对象，它对应的就是之前在 Java 层创建的 InputManagerService。

### 1.4 InputManager.init

```
InputManager::InputManager(
        const sp<InputReaderPolicyInterface>& readerPolicy,
        const sp<InputDispatcherPolicyInterface>& dispatcherPolicy) {
    mDispatcher = createInputDispatcher(dispatcherPolicy);
    mClassifier = new InputClassifier(mDispatcher);
    mReader = createInputReader(readerPolicy, mClassifier);
}
```

#### 1.4.1 创建 InputDispatcher 对象

```
sp<InputDispatcherInterface> createInputDispatcher(
        const sp<InputDispatcherPolicyInterface>& policy) {
    return new android::inputdispatcher::InputDispatcher(policy);
}
```

创建了一个 InputDispatcher 对象。

#### 1.4.2 创建 InputClassifier 对象

基于上一步创建的 InputDispatcher 对象，创建了一个 InputClassifier 对象：

```
InputClassifier::InputClassifier(const sp<InputListenerInterface>& listener)
      : mListener(listener), mHalDeathRecipient(new HalDeathRecipient(*this)) {}
```

将其内部的 mListener 指向了上一步创建的 InputDispatcher 实例，mListener 是：

```
// The next stage to pass input events to
    sp<InputListenerInterface> mListener;
```

如注释所说，InputClassifier 会将输入事件传给 mListener，后续我们会看到事件是怎么从 InputClassifier 传到 InputDispatcher 的。

#### 1.4.3 创建 InputReader 对象

基于上一步创建的 InputClassifier 对象，创建一个 InputReader 对象：

```
sp<InputReaderInterface> createInputReader(const sp<InputReaderPolicyInterface>& policy,
                                           const sp<InputListenerInterface>& listener) {
    return new InputReader(std::make_unique<EventHub>(), policy, listener);
}
```

这里创建了一个 EventHub 对象，然后基于这个 EventHub 和之前传入的 InputClassifier 来创建 InputReader。

```
InputReader::InputReader(std::shared_ptr<EventHubInterface> eventHub,
                         const sp<InputReaderPolicyInterface>& policy,
                         const sp<InputListenerInterface>& listener)
      : mContext(this),
        mEventHub(eventHub),
        mPolicy(policy),
        mGlobalMetaState(0),
        mLedMetaState(AMETA_NUM_LOCK_ON),
        mGeneration(1),
        mNextInputDeviceId(END_RESERVED_ID),
        mDisableVirtualKeysTimeout(LLONG_MIN),
        mNextTimeout(LLONG_MAX),
        mConfigurationChangesToRefresh(0) {
    mQueuedListener = new QueuedInputListener(listener);

    { // acquire lock
        std::scoped_lock _l(mLock);

        refreshConfigurationLocked(0);
        updateGlobalMetaStateLocked();
    } // release lock
}
```

看到这里基于传入的 InputClassifier 生成了一个 QueuedInputListener 对象，QueuedInputListener 定义在 InputListener.h 中：

```
/*
 * An implementation of the listener interface that queues up and defers dispatch
 * of decoded events until flushed.
 */
class QueuedInputListener : public InputListenerInterface {
protected:
    virtual ~QueuedInputListener();

public:
    explicit QueuedInputListener(const sp<InputListenerInterface>& innerListener);

    virtual void notifyConfigurationChanged(const NotifyConfigurationChangedArgs* args) override;
    virtual void notifyKey(const NotifyKeyArgs* args) override;
    virtual void notifyMotion(const NotifyMotionArgs* args) override;
    ......

    void flush();

private:
    sp<InputListenerInterface> mInnerListener;
    std::vector<NotifyArgs*> mArgsQueue;
};
```

QueuedInputListener 用来在刷新前将解码后的事件进行排队和延迟分发。

这里的 mArgsQueue 存储的是 NotifyArgs 类型的对象，NotifyArgs 是描述输入事件的超类，其子类有键盘事件 NotifyKeyArgs 和触摸事件 NotifyMotionArgs 等，那么 mArgsQueue 就是用来存储这些排队等待分发的事件的。

这里的 mInnerListener 赋值在 QueuedInputListener 的构造方法中：

```
QueuedInputListener::QueuedInputListener(const sp<InputListenerInterface>& innerListener) :
        mInnerListener(innerListener) {
}
```

那么这里的 mInnerListener 指向的是一个 InputClassifier，QueuedInputListener 调用 flush 函数后，mArgsQueue 就会发送到 mInnerListener 也就是 InputClassifier 处。

扯远了，后面再分析具体事件是怎么从 InputReader 到 InputClassifier 再到 InputDispatcher 的。

### 1.5 小结

1)、创建了 Java 上层的 InputManagerService，JNI 层的 NativeInputManager 以及 Native 层的 InputManager。

2)、Native 层的 InputManager 在创建的时候，创建了 InputReader 和 InputDispatcher，并且创建了一个 InputClassifier 对象。

3)、InputReader 内部有一个成员变量 mQeueuInputListener，mQeueuInputListener 维护了一个当前正在排队等待分发的输入事件队列 mArgsQueue，并且有一个 mInnerListener 成员指向 InputClassifier，QueuedInputListener 调用 flush 函数后，mArgsQueue 就会发送到 mInnerListener 也就是 InputClassifier 处。

4)、InputClassifier 有一个成员变量 mListener，指向 InputDispatcher，InputClassifer 收到的事件会发送给 mListener，最终，InputReader 获取到的事件会通过 InputClassifier 发送给 InputDispatcher。

2 启动
----

![](https://img-blog.csdnimg.cn/b2dc49a8913747e0bce9379aef748488.png#pic_center)

### 2.1 InputManagerService.start

```
public void start() {
        Slog.i(TAG, "Starting input manager");
        nativeStart(mPtr);
        ......
    }
```

上面分析了 InputManagerService 初始化过程，现在来看 InputManagerService 的启动流程，这里也是调用了 JNI 层 InputManagerService.nativeStart。

### 2.2 com_android_server_input_InputManagerService.nativeStart

```
static void nativeStart(JNIEnv* env, jclass /* clazz */, jlong ptr) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);

    status_t result = im->getInputManager()->start();
    if (result) {
        jniThrowRuntimeException(env, "Input manager could not be started.");
    }
}
```

这里传入的指针是之前 1.2 节中 com_android_server_input_InputManagerService.nativeInit 处得到的 NativeInputManager 的指针，那么这里就可以获取之前已经创建过的 NativeInputManager，最后调用了 Native 层的 InputManager.start。

### 2.3 InputManager.start

```
status_t InputManager::start() {
    status_t result = mDispatcher->start();
    if (result) {
        ALOGE("Could not start InputDispatcher thread due to error %d.", result);
        return result;
    }

    result = mReader->start();
    if (result) {
        ALOGE("Could not start InputReader due to error %d.", result);

        mDispatcher->stop();
        return result;
    }

    return OK;
}
```

InputManager.start 调用了 InputReader.start 和 InputDispatcher.start，分别分析一下具体内容。

#### 2.3.1 InputReader.start

```
status_t InputReader::start() {
    if (mThread) {
        return ALREADY_EXISTS;
    }
    mThread = std::make_unique<InputThread>(
            "InputReader", [this]() { loopOnce(); }, [this]() { mEventHub->wake(); });
    return OK;
}
```

这里看到，InputReader.start 函数会创建一个唯一的名为 “InputReader” 的 InputThread 对象，并且赋值给 InputThread 类型的 mThread 成员变量。

InputThread 的构造方法定义是：

```
InputThread::InputThread(std::string name, std::function<void()> loop, std::function<void()> wake)
      : mName(name), mThreadWake(wake) {
    mThread = new InputThreadImpl(loop);
    mThread->run(mName.c_str(), ANDROID_PRIORITY_URGENT_DISPLAY);
}
```

可以看到，首先 InputThread 内部也会先创建一个 InputThreadImpl 类型的对象赋值给 mThread 成员变量，接着调用 mThread.run。

InputThreadImpl 继承自 Thread，那么调用 mThread.run 会调用到 InputThreadImpl 的 threadLoop 函数，再看下 InputThreadImpl 的定义：

```
// Implementation of Thread from libutils.
class InputThreadImpl : public Thread {
public:
    explicit InputThreadImpl(std::function<void()> loop)
          : Thread(/* canCallJava */ true), mThreadLoop(loop) {}

    ~InputThreadImpl() {}

private:
    std::function<void()> mThreadLoop;

    bool threadLoop() override {
        mThreadLoop();
        return true;
    }
};
```

只要 threadLoop 函数返回 true，以下语句就会一直循环调用：

```
mThreadLoop();
```

这里传入的循环函数是 InputReader 的 loopOnce。

那么这里也就是说，调用 InputReader 的 start 函数后，会创建一个名为 “InputReader” 的 InputThread 对象，并且其内部线程 mThread 会一直循环调用 InputReader.loopOnce 函数。

#### 2.3.2 InputDispatcher.start

```
status_t InputDispatcher::start() {
    if (mThread) {
        return ALREADY_EXISTS;
    }
    mThread = std::make_unique<InputThread>(
            "InputDispatcher", [this]() { dispatchOnce(); }, [this]() { mLooper->wake(); });
    return OK;
}
```

InputDispatcher.start 也是同理，会创建一个名为 “InputDispatcher” 的 InputThread 对象，其内部线程 mThread 会一直循环调用 InputDispatcher.dispatchOnce 函数。

### 2.4 小结

1)、InputReader 内部维护一个唯一的，名为 “InputReader” 的 InputThread 对象，该 InputThread 对象内部有一个线程会一直循环调用 InputReader 的 loopOnce 函数，这个线程可以认为是 InputReader 的读取线程。

2)、InputDispatcher 内部维护一个唯一的，名为 “InputDispatcher” 的 InputThread 对象，该 InputThread 对象内部有一个线程会一直循环调用 InputDispatcher 的 dispatchOnce 函数，这个线程可以认为是 InputDispatcher 的分发线程。
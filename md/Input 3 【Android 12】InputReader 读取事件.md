> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/126746393?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/61ef6c9e79fd4a1cb8450830aece9d19.jpeg#pic_center)

在 Input 相关服务的创建和启动中，我们知道了 InputManager 在 start 函数中会创建一个 InputReader，其内部有一个线程会循环调用 InputReader.loopOnce，实现 InputReader 对输入事件的持续读取，那么我们以此为起点，分析一下 InputReader 读取事件流程。

```
void InputReader::loopOnce() {
    ......

    // 1. 从EventHub读取原始事件
    size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);

    { // acquire lock
        std::scoped_lock _l(mLock);
        mReaderIsAliveCondition.notify_all();

        // 2. InputReader.processEventsLocked处理事件。
        if (count) {
            processEventsLocked(mEventBuffer, count);
        }

    ......
        
    // 3. QueuedInputListener分发事件
    // Flush queued events out to the listener.
    // This must happen outside of the lock because the listener could potentially call
    // back into the InputReader's methods, such as getScanCodeState, or become blocked
    // on another thread similarly waiting to acquire the InputReader lock thereby
    // resulting in a deadlock.  This situation is actually quite plausible because the
    // listener is actually the input dispatcher, which calls into the window manager,
    // which occasionally calls into the input reader.
    mQueuedListener->flush();
}
```

可以分为三部分：

1)、EventHub.getEvent 获取输入事件。

2)、InputReader.processEventsLocked 处理事件。

3)、QueuedInputListener 分发事件。

一、从 EventHub 读取原始事件
-------------------

```
size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);
```

EventHub.getEvent[s 函数](https://so.csdn.net/so/search?q=s%E5%87%BD%E6%95%B0&spm=1001.2101.3001.7020)内容很庞大，这里先挖个坑暂不分析，总结就是如果此时有输入事件，那么通过 EventHub.getEvent 来读取事件，将输入事件封装为 RawEvent 对象。否则，InputReader 读取线程就会睡眠在 EventHub.getEvent 函数上：

```
int pollResult = epoll_wait(mEpollFd, mPendingEventItems, EPOLL_MAX_EVENTS, timeoutMillis);
```

[epoll_wait](https://so.csdn.net/so/search?q=epoll_wait&spm=1001.2101.3001.7020)，等待 epoll 文件描述符上的 I / O 事件。

最终将从 EventHub 中读取的原始事件保存到 mEventBuffer 中，count 代表了原始事件的个数。

二、加工原始事件
--------

Key 和 Motion 分发流程类似，因此在这里以 Key 事件为例看一下整个流程[时序图](https://so.csdn.net/so/search?q=%E6%97%B6%E5%BA%8F%E5%9B%BE&spm=1001.2101.3001.7020)：

![](https://img-blog.csdnimg.cn/2852743648a74b4489a1d70754e7d830.png#pic_center)

上一步中将从 EventHub 中读取到的原始信息保存在了 RawEvent 类型的 mEventBuffer 中，接着使用 InputReader.processEventsLocked 方法进一步处理事件。

### 1 InputReader.processEventsLocked

```
void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
    for (const RawEvent* rawEvent = rawEvents; count;) {
        int32_t type = rawEvent->type;
        size_t batchSize = 1;
        if (type < EventHubInterface::FIRST_SYNTHETIC_EVENT) {
			......
            processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
        } else {
            switch (rawEvent->type) {
                case EventHubInterface::DEVICE_ADDED:
                    addDeviceLocked(rawEvent->when, rawEvent->deviceId);
                    break;
                case EventHubInterface::DEVICE_REMOVED:
                    removeDeviceLocked(rawEvent->when, rawEvent->deviceId);
                    break;
                case EventHubInterface::FINISHED_DEVICE_SCAN:
                    handleConfigurationChangedLocked(rawEvent->when);
                    break;
                default:
                    ALOG_ASSERT(false); // can't happen
                    break;
            }
        }
        count -= batchSize;
        rawEvent += batchSize;
    }
}
```

这里将普通的事件输入事件和设备的输入事件分开处理：

1)、输入设备事件的定义在 EventHub.h：

```
// Synthetic raw event type codes produced when devices are added or removed.
    enum {
        // Sent when a device is added.
        DEVICE_ADDED = 0x10000000,
        // Sent when a device is removed.
        DEVICE_REMOVED = 0x20000000,
        // Sent when all added/removed devices from the most recent scan have been reported.
        // This event is always sent at least once.
        FINISHED_DEVICE_SCAN = 0x30000000,

        FIRST_SYNTHETIC_EVENT = DEVICE_ADDED,
    };
```

DEVICE_ADDED 对应一次设备添加事件，DEVICE_REMOVED 对应一次设备删除事件，FINISHED_DEVICE_SCAN 对应一次所有设备的扫描事件。

2)、如果事件 type 小于 FIRST_SYNTHETIC_EVENT，那么此次不是一次增加 / 删除 / 扫描设备事件，调用 InputReader.processEventsForDeviceLocked，让对应设备去处理本次输入事件。

### 2 InputReader.processEventsForDeviceLocked

```
void InputReader::processEventsForDeviceLocked(int32_t eventHubId, const RawEvent* rawEvents,
                                               size_t count) {
    auto deviceIt = mDevices.find(eventHubId);
	......

    std::shared_ptr<InputDevice>& device = deviceIt->second;
    ......

    device->process(rawEvents, count);
}
```

从 mDevices 中获取当前事件对应的设备对象 InputDevice，然后调用 InputDevice.process 处理 RawEvent。

### 3 InputDevice.process

```
void InputDevice::process(const RawEvent* rawEvents, size_t count) {
    // Process all of the events in order for each mapper.
    // We cannot simply ask each mapper to process them in bulk because mappers may
    // have side-effects that must be interleaved.  For example, joystick movement events and
    // gamepad button presses are handled by different mappers but they should be dispatched
    // in the order received.
    for (const RawEvent* rawEvent = rawEvents; count != 0; rawEvent++) {
        ......

        if (mDropUntilNextSync) {
            ......
        } else if (rawEvent->type == EV_SYN && rawEvent->code == SYN_DROPPED) {
            ......
        } else {
            for_each_mapper_in_subdevice(rawEvent->deviceId, [rawEvent](InputMapper& mapper) {
                mapper.process(rawEvent);
            });
        }
        --count;
    }
}
```

这里出现了一个新概念，InputMapper：

```
/* An input mapper transforms raw input events into cooked event data.
 * A single input device can have multiple associated input mappers in order to interpret
 * different classes of events.
```

InputMapper 用来将原始的输入事件转换为处理过的输入数据，一个输入设备可能对应多个 InputMapper。

InputMapper 有多个子类，TouchInputMapper、KeyboardInputMapper、SensorInputMapper 等等，用来处理不同类型的输入事件。

上一步中，我们找到了一个输入设备可以处理当前输入事件，那么这一步就是遍历调用该 InputDevice 对应的所有 InputMapper，调用 InputMapper 的 proces 函数，去处理输入事件。

### 4 InputMapper 处理原始输入事件

我们重点分析键盘事件和触摸屏事件。

#### 4.1 键盘事件

键盘设备对应的 InputMapper 是 KeyboardInputMapper。

##### 4.1.1 KeyboardInputMapper.process

```
void KeyboardInputMapper::process(const RawEvent* rawEvent) {
    switch (rawEvent->type) {
        case EV_KEY: {
            int32_t scanCode = rawEvent->code;
            int32_t usageCode = mCurrentHidUsage;
            mCurrentHidUsage = 0;

            if (isKeyboardOrGamepadKey(scanCode)) {
                processKey(rawEvent->when, rawEvent->readTime, rawEvent->value != 0, scanCode,
                           usageCode);
            }
            break;
        }
        case EV_MSC: {
            ......
        }
        case EV_SYN: {
            ......
        }
    }
}
```

查阅资料，Linux 系统中总共有这几种输入事件：

> EV_SYN 0x00 同步事件
> 
> EV_KEY 0x01 按键事件，如 KEY_VOLUMEDOWN
> 
> EV_REL 0x02 相对坐标, 如 shubiao 上报的坐标
> 
> EV_ABS 0x03 绝对坐标，如触摸屏上报的坐标
> 
> EV_MSC 0x04 其它
> 
> EV_LED 0x11 LED
> 
> EV_SND 0x12 声音
> 
> EV_REP 0x14 Repeat
> 
> EV_FF 0x15 力反馈
> 
> EV_PWR 电源
> 
> EV_FF_STATUS 状态

暂不分析其他事件，那么对于键盘事件中的按键事件，调用 KeyboardInputMapper.processKey 函数继续进行处理。

##### 4.1.2. KeyboardInputMapper.processKey

```
void KeyboardInputMapper::processKey(nsecs_t when, nsecs_t readTime, bool down, int32_t scanCode,
                                     int32_t usageCode) {
    int32_t keyCode;
    int32_t keyMetaState;
    uint32_t policyFlags;

    ......

    NotifyKeyArgs args(getContext()->getNextId(), when, readTime, getDeviceId(), mSource,
                       getDisplayId(), policyFlags,
                       down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP,
                       AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, keyMetaState, downTime);
    getListener()->notifyKey(&args);
}
```

将原始的输入数据封装为 NotifyKeyArgs 类型，然后通知 listener。

#### 4.2 触摸屏事件

触摸屏设备对应的 InputMapper 是 TouchInputMapper。

##### 4.2.1 TouchInputMapper.process

```
void TouchInputMapper::process(const RawEvent* rawEvent) {
    mCursorButtonAccumulator.process(rawEvent);
    mCursorScrollAccumulator.process(rawEvent);
    mTouchButtonAccumulator.process(rawEvent);

    if (rawEvent->type == EV_SYN && rawEvent->code == SYN_REPORT) {
        sync(rawEvent->when, rawEvent->readTime);
    }
}
```

1)、mCursorButtonAccumulator 用来处理鼠标和触摸盘按键事件，mCursorScrollAccumulator 用来处理鼠标滑动事件，mTouchButtonAccumulator 用来处理手写笔和之类的事件。

2)、调用 TouchInputMapper.sync 函数进行事件的同步。

##### 4.2.2 TouchInputMapper.sync

##### 4.2.3 TouchInputMapper.processRawTouches

##### 4.2.4 TouchInputMapper.cookAndDispatch

##### 4.2.5 TouchInputMapper.dispatchTouches

##### 4.2.6 TouchInputMapper.dispatchMotion

```
void TouchInputMapper::dispatchMotion(nsecs_t when, nsecs_t readTime, uint32_t policyFlags,
                                      uint32_t source, int32_t action, int32_t actionButton,
                                      int32_t flags, int32_t metaState, int32_t buttonState,
                                      int32_t edgeFlags, const PointerProperties* properties,
                                      const PointerCoords* coords, const uint32_t* idToIndex,
                                      BitSet32 idBits, int32_t changedId, float xPrecision,
                                      float yPrecision, nsecs_t downTime) {
    ......

    const int32_t deviceId = getDeviceId();
    std::vector<TouchVideoFrame> frames = getDeviceContext().getVideoFrames();
    std::for_each(frames.begin(), frames.end(),
                  [this](TouchVideoFrame& frame) { frame.rotate(this->mSurfaceOrientation); });
    NotifyMotionArgs args(getContext()->getNextId(), when, readTime, deviceId, source, displayId,
                          policyFlags, action, actionButton, flags, metaState, buttonState,
                          MotionClassification::NONE, edgeFlags, pointerCount, pointerProperties,
                          pointerCoords, xPrecision, yPrecision, xCursorPosition, yCursorPosition,
                          downTime, std::move(frames));
    getListener()->notifyMotion(&args);
}
```

将原始的输入数据封装为 NotifyMotionArgs 类型，然后通知 listener。

### 5 QueuedInputListener.notifyKey & QueuedInputListener.notifyMotion

从上面的分析可以看到：

1)、对于键盘事件，通过 KeyboardInputMapper 的处理，最终将原始输入事件封装为 NotifyKeyArgs 对象，然后通过 getListener 获取到传递对象，调用 notifyKey 将封装好的事件传过去。

2)、对于触摸屏事件，通过 TouchInputMapper 的处理，最终将原始输入事件封装为 NotifyMotionArgs 对象，然后通过 getListener 获取到传递对象，调用 notifyMotion 将封装好的事件传过去。

这里直接放结论，InputMapper.getListener 函数返回了 InputReader.mQueuedListener。

```
InputListenerInterface* InputReader::ContextImpl::getListener() {
    return mReader->mQueuedListener.get();
}
```

那么上面无论是键盘事件：

```
getListener()->notifyKey(&args);
```

或者触摸事件：

```
getListener()->notifyMotion(&args);
```

调用的都是 QueuedInputListener 的相关函数：

```
void QueuedInputListener::notifyKey(const NotifyKeyArgs* args) {
    traceEvent(__func__, args->id);
    mArgsQueue.push_back(new NotifyKeyArgs(*args));
}

void QueuedInputListener::notifyMotion(const NotifyMotionArgs* args) {
    traceEvent(__func__, args->id);
    mArgsQueue.push_back(new NotifyMotionArgs(*args));
}
```

最终的结果是，将封装的 NotifyKeyArgs 和 NotifyMotionArgs 加入到了 QueuedInputListener 的成员变量 mArgsQueue 中。

三、通知 InputDispatcher 有新事件发生
---------------------------

Key 和 Motion 分发流程类似，因此在这里以 Key 事件为例看一下整个流程时序图：

![](https://img-blog.csdnimg.cn/0cd5683eea4d45dc951a50b7e87efb11.png#pic_center)

```
// Flush queued events out to the listener.
    // This must happen outside of the lock because the listener could potentially call
    // back into the InputReader's methods, such as getScanCodeState, or become blocked
    // on another thread similarly waiting to acquire the InputReader lock thereby
    // resulting in a deadlock.  This situation is actually quite plausible because the
    // listener is actually the input dispatcher, which calls into the window manager,
    // which occasionally calls into the input reader.
    mQueuedListener->flush();
```

上一步中，调用了 InputReader.processEventsLocked 函数，将所有原始事件加工为了 NotifyArgs 类型，并且加入到了 mQueuedListener.mArgsQueue，那么这一步就是调用 QueuedInputListener.flush 将加工好的事件分发给 InputReader 相应的 listener。

### 1 QueuedInputListener.flush

```
void QueuedInputListener::flush() {
    size_t count = mArgsQueue.size();
    for (size_t i = 0; i < count; i++) {
        NotifyArgs* args = mArgsQueue[i];
        args->notify(mInnerListener);
        delete args;
    }
    mArgsQueue.clear();
}
```

这里的 mArgsQueue 队列中已经有了从 EventHub 中读取到的并且已经被加工为 NotifyArgs 类型的事件了，接着调用 NotifyArgs.notify 函数。

### 2 NotifyKeyArgs.notify & NotifyMotionArgs.notify

```
void NotifyKeyArgs::notify(const sp<InputListenerInterface>& listener) const {
    listener->notifyKey(this);
}

......
    
void NotifyMotionArgs::notify(const sp<InputListenerInterface>& listener) const {
    listener->notifyMotion(this);
}
```

NotifyArgs 是 NotifyKeyArgs 和 NotifyMotionArgs 的基类，根据事件类型分别调用不同子类的 notify 函数。

这里传入的 listener 是 QueueInputListener.mInnerListener，在之前的分析中我们知道它对应的是一个 InputClassifier 对象。

### 3 InputClassifier.notifyKey & InputClassifier.notifyMotion

```
void InputClassifier::notifyKey(const NotifyKeyArgs* args) {
    // pass through
    mListener->notifyKey(args);
}

void InputClassifier::notifyMotion(const NotifyMotionArgs* args) {
    std::scoped_lock lock(mLock);
    // MotionClassifier is only used for touch events, for now
    const bool sendToMotionClassifier = mMotionClassifier && isTouchEvent(*args);
    if (!sendToMotionClassifier) {
        mListener->notifyMotion(args);
        return;
    }

    NotifyMotionArgs newArgs(*args);
    newArgs.classification = mMotionClassifier->classify(newArgs);
    mListener->notifyMotion(&newArgs);
}
```

在之前的分析我们知道了，InputClassifier.mListener 指向了一个 InputDispatcher，那么这里调用的自然也是 InputDispatcher 的相关函数。

#### 3.1 InputDispatcher.notifyKey

```
void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {
    ......

    KeyEvent event;
    event.initialize(args->id, args->deviceId, args->source, args->displayId, INVALID_HMAC,
                     args->action, flags, keyCode, args->scanCode, metaState, repeatCount,
                     args->downTime, args->eventTime);

    android::base::Timer t;
    mPolicy->interceptKeyBeforeQueueing(&event, /*byref*/ policyFlags);
    if (t.duration() > SLOW_INTERCEPTION_THRESHOLD) {
        ALOGW("Excessive delay in interceptKeyBeforeQueueing; took %s ms",
              std::to_string(t.duration().count()).c_str());
    }

    bool needWake;
    { // acquire lock
        ......

        std::unique_ptr<KeyEntry> newEntry =
                std::make_unique<KeyEntry>(args->id, args->eventTime, args->deviceId, args->source,
                                           args->displayId, policyFlags, args->action, flags,
                                           keyCode, args->scanCode, metaState, repeatCount,
                                           args->downTime);

        needWake = enqueueInboundEventLocked(std::move(newEntry));
        mLock.unlock();
    } // release lock

    if (needWake) {
        mLooper->wake();
    }
}
```

1)、根据传入的 NotifyKeyArgs 的信息封装一个 KeyEvent 对象，在将输入事件传给 Activity 之前，调用 NativeInputManager.interceptKeyBeforeQueueing 对 KeyEvent 进行预处理，最终会调用到 PhoneWindowManager#interceptKeyBeforeQueueing 中，决定是否要把输入事件发送给用户。

2)、根据传入的 NotifyKeyArgs 的信息封装一个 KeyEntry 对象，然后调用 InputDispatcher.InputDispatcher.enqueueInboundEventLocked。

3)、如果 InputDispatcher.InputDispatcher.enqueueInboundEventLocked 返回 true，那么说明需要唤醒 InputDispatcher 的分发线程。

#### 3.2 InputDispatcher.notifyMotion

```
void InputDispatcher::notifyMotion(const NotifyMotionArgs* args) {
    ......

    uint32_t policyFlags = args->policyFlags;
    policyFlags |= POLICY_FLAG_TRUSTED;

    android::base::Timer t;
    mPolicy->interceptMotionBeforeQueueing(args->displayId, args->eventTime, /*byref*/ policyFlags);
    if (t.duration() > SLOW_INTERCEPTION_THRESHOLD) {
        ALOGW("Excessive delay in interceptMotionBeforeQueueing; took %s ms",
              std::to_string(t.duration().count()).c_str());
    }

    bool needWake;
    { // acquire lock
        mLock.lock();

		......

        // Just enqueue a new motion event.
        std::unique_ptr<MotionEntry> newEntry =
                std::make_unique<MotionEntry>(args->id, args->eventTime, args->deviceId,
                                              args->source, args->displayId, policyFlags,
                                              args->action, args->actionButton, args->flags,
                                              args->metaState, args->buttonState,
                                              args->classification, args->edgeFlags,
                                              args->xPrecision, args->yPrecision,
                                              args->xCursorPosition, args->yCursorPosition,
                                              args->downTime, args->pointerCount,
                                              args->pointerProperties, args->pointerCoords, 0, 0);

        needWake = enqueueInboundEventLocked(std::move(newEntry));
        mLock.unlock();
    } // release lock

    if (needWake) {
        mLooper->wake();
    }
}
```

内容和 InputDispatcher.notifyKey 相似：

1)、在将输入事件传给 Activity 之前，将传入的 NotifyMotionArgs 的信息发送给 NativeInputManager.interceptMotionBeforeQueueing 进行预处理，最终会调用 PhoneWindowManager#interceptMotionBeforeQueueing，由 PhoneWindowManager 决定是否要拦截此次事件。

2)、根据传入的 NotifyMotionArgs 的信息封装一个 MotionEntry 对象，然后调用 InputDispatcher.InputDispatcher.enqueueInboundEventLocked。

3)、如果 InputDispatcher.InputDispatcher.enqueueInboundEventLocked 返回 true，那么说明需要唤醒 InputDispatcher 的分发线程。

#### 3.3 InputDispatcher.enqueueInboundEventLocked

不管是 Key 事件还是 Motion 事件，在经过 PhoneWindowManager 预处理后，如果 PhoneWindowManager 不拦截此次事件，那么接下来就是调用 InputDispatcher.enqueueInboundEventLocked 函数继续分发事件给相关窗口。

```
bool InputDispatcher::enqueueInboundEventLocked(std::unique_ptr<EventEntry> newEntry) {
    bool needWake = mInboundQueue.empty();
    mInboundQueue.push_back(std::move(newEntry));
    EventEntry& entry = *(mInboundQueue.back());
    traceInboundQueueLengthLocked();

    switch (entry.type) {
        case EventEntry::Type::KEY: {
			......
        }

        case EventEntry::Type::MOTION: {
			......
        }
        case EventEntry::Type::FOCUS: {
            LOG_ALWAYS_FATAL("Focus events should be inserted using enqueueFocusEventLocked");
            break;
        }
        case EventEntry::Type::CONFIGURATION_CHANGED:
        case EventEntry::Type::DEVICE_RESET:
        case EventEntry::Type::SENSOR:
        case EventEntry::Type::POINTER_CAPTURE_CHANGED:
        case EventEntry::Type::DRAG: {
            // nothing to do
            break;
        }
    }

    return needWake;
}
```

首先判断 mInboundQueue 是否为空，将本次处理的 EventEntry 加入到 mInboundQueue 中。接着判断是否要唤醒 InputDispatcher 的分发线程，有三种唤醒情况：

1)、mInboundQueue 为空。

2)、mInboundQueue 不为空，此时为按键事件。如果当前按键事件将触发 App 切换，并且如果此时该按键抬起，需要丢弃之前所有的事件，并且唤醒 InputDispatcher 分发线程。

3)、mInboundQueue 不为空，此时为触摸事件。如果当前触摸事件之前的事件需要被删除，那么唤醒 InputDispatcher 分发线程。

### 4 InputDispatcher 分发线程的唤醒

将输入事件加入 InputDispatcher.mInboundQueue 中，下一步就是唤醒 InputDispatcher 分发线程。

先来补充一下 Looper 的睡眠和唤醒机制：

> C++ 类 Looper 中的睡眠和唤醒机制是通过 pollOnce 和 wake 函数提供的，它们又是利用操作系统（Linux 内核）的 epoll 机制来完成的。
> 
> 在 Looper 的构造函数中，会创建一个管道，然后调用 epoll_create 获取一个 epoll 的实例的描述符，最后将管道读端描述符作为一个事件报告项添加给 epoll。
> 
> Looper 的 pollOnce 函数将最终调用到其 pollInner 函数。在后者里面，将调用 epoll_wait 睡眠等待其监控的文件描述符是否有可 I/O 事件的到来，若有（哪怕只有一个），epoll_wait 将会醒来。
> 
> Looper 的 wake 函数用于向管道中写入字符，pollOnce 将会从 epoll_wait 中唤醒。

我们知道 InputDispatcher 内部有一个的分发线程，在循环调用 dispatchOnce 来进行事件的分发，但即使是循环调用，这个线程也不可能真的无限制在一直分发，而是当有事件可读时，才进行分发，其他时间分发线程进入休眠状态。

InputDispatcher.dispatchOnce 函数的最后，在事件分发完成后会调用 Looper.pollOnce 将当前线程睡眠：

```
void InputDispatcher::dispatchOnce() {
	......
    // 上面是分发事件的具体内容    

    // Wait for callback or timeout or wake.  (make sure we round up, not down)
    nsecs_t currentTime = now();
    int timeoutMillis = toMillisecondTimeoutDelay(currentTime, nextWakeupTime);
    mLooper->pollOnce(timeoutMillis);
}
```

那么上面在 InputDispatcher.notifyKey 或者是 InputDispatcher.notifyMotion 中调用 Looper.wake 的时候：

```
if (needWake) {
        mLooper->wake();
    }
```

InputDispatcher.dispatchOnce 中的 Looper.pollOnce 函数就可以返回，上一次的 InputDispatcher.dispatchOnce 函数才会全部执行完成并且返回，进而可以循环进行下一次的 InputDispatcher.dispatchOnce，开启新一轮事件分发。

四、总结
----

1)、InputReader 通过 EventHub.getEvents 读取原始事件 RawEvent。

2)、InputReader 调用 InputReader.processEventsLocked 函数将原始事件加工为 NotifyArgs 类型，然后存储到 InputReader 的 QueuedInputListener 类型的成员变量 mQueuedListener 内部的 mArgsQueue 中进行排队等待分发。

3)、InputReader 调用 QueuedInputListener.flush 函数，将 QueuedInputListener.mArgsQueue 存储的事件一次性发送到 InputDispatcher，最终事件是被封装成了 EventEntry 类型，添加到了 InputDispatcher.mInboundQueue 中，InputReader 并且唤醒了 InputDispatcher 的分发线程，后续 InputDispatcher 分发线程会重新调用 InputDispatcher.dispatchOnce 函数来进行事件的分发。

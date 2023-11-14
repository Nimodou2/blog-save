> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/126746426?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/95140f1dd00848049871aadff7afbfaf.jpeg#pic_center)

在 Input 相关服务的创建和启动中，我们知道了 InputManager 在 start 函数中会创建一个 InputDispatcher 对象，其内部有一个线程会循环调用 InputDispatcher.dispatchOnce，实现 InputDispatcher 对事件的分发，那么我们以此为起点，分析一下 InputDispatcher 分发事件流程。

```
void InputDispatcher::dispatchOnce() {
    nsecs_t nextWakeupTime = LONG_LONG_MAX;
    { // acquire lock
		......

        // Run a dispatch loop if there are no pending commands.
        // The dispatch loop might enqueue commands to run afterwards.
        if (!haveCommandsLocked()) {
            dispatchOnceInnerLocked(&nextWakeupTime);
        }

		......
    } // release lock

	......
    mLooper->pollOnce(timeoutMillis);
}
```

在之前的分析中，我们知道了 InputDispatcher.dispatchOnce 函数在分发完事件后，会调用 Looper.pollOnce 将当前线程挂起，后续 InputReader 的读取线程会将新的事件发送给 InputDispatcher，并且唤醒 InputDispatcher 的分发线程，因此这里我们只需要分析 InputDispatcher 分发事件的流程即可，也就是 InputDispatcher.dispatchOnceInnerLocked 函数。

本文的侧重点在 Motion 事件的分发，因此以 Motion 事件的流程为例看一下整个流程：

![](https://img-blog.csdnimg.cn/593fc1e4da0c4d04b0b713313bf761b5.png#pic_center)

可能需要提前了解的和输入窗口相关的类有：

![](https://img-blog.csdnimg.cn/b5887ea4944c4e2a9d7e3f2b2529b49a.png#pic_center)

1 InputDispatcher.dispatchOnceInnerLocked
-----------------------------------------

```
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
	......

    // Ready to start a new event.
    // If we don't already have a pending event, go grab one.
    if (!mPendingEvent) {
        if (mInboundQueue.empty()) {
            ......
        } else {
            // 1.1 从事件队列中获取待处理的输入事件
            // Inbound queue has at least one entry.
            mPendingEvent = mInboundQueue.front();
            mInboundQueue.pop_front();
            traceInboundQueueLengthLocked();
        }
		......
    }

	......

    switch (mPendingEvent->type) {
		......

        case EventEntry::Type::KEY: {
            std::shared_ptr<KeyEntry> keyEntry = std::static_pointer_cast<KeyEntry>(mPendingEvent);
			......
            // 1.2 根据事件类型调用相应函数处理
            done = dispatchKeyLocked(currentTime, keyEntry, &dropReason, nextWakeupTime);
            break;
        }

        case EventEntry::Type::MOTION: {
			......
            // 1.2 根据事件类型调用相应函数处理
            done = dispatchMotionLocked(currentTime, motionEntry, &dropReason, nextWakeupTime);
            break;
        }

		......
    }

	......
}
```

##### 1.1 从事件队列中获取待处理的输入事件

之前 InputReader 分发线程通过 InputDispatcher.enqueueInboundEventLocked，向 InputDispatcher.mInboundQueue 加入了当前要处理的输入事件。假设当前没有等待处理的事件，那就从 InputDispatcher.mInboundQueue 取出一个 EventEntry 类型的事件赋值给 mPendingEvent。

##### 1.2 根据事件类型调用相应函数处理

判断 mPendingEvent 的类型，如果是 Key 事件则调用 InputDispatcher.dispatchKeyLocked 函数处理，如果是 Motion 事件则调用 InputDispatcher.dispatchMotionLocked 函数处理，当然还有其他类型的事件，这里我们主要分析这两种事件。

2 Key 事件
--------

```
bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, std::shared_ptr<KeyEntry> entry,
                                        DropReason* dropReason, nsecs_t* nextWakeupTime) {
    // Preprocessing.
	......

    // Handle case where the policy asked us to try again later last time.
	......

    // Give the policy a chance to intercept the key.
	......

    // Clean up if dropping the event.
	......

    // Identify targets.
    std::vector<InputTarget> inputTargets;
    InputEventInjectionResult injectionResult =
            findFocusedWindowTargetsLocked(currentTime, *entry, inputTargets, nextWakeupTime);
    if (injectionResult == InputEventInjectionResult::PENDING) {
        return false;
    }

    setInjectionResult(*entry, injectionResult);
    if (injectionResult != InputEventInjectionResult::SUCCEEDED) {
        return true;
    }

    // Add monitor channels from event's or focused display.
    addGlobalMonitoringTargetsLocked(inputTargets, getTargetDisplayId(*entry));

    // Dispatch the key.
    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}
```

1)、调用 InputDispatcher.findFocusedWindowTargetsLocked 寻找焦点窗口，将结果存放在 inputTargets 中。

2)、调用 InputDispatcher.dispatchEventLocked 继续分发事件，这个方法在处理 Motion 事件的时候也会用到，因此后续 Motion 事件分析完后再放到一起说。

那么本节我们重点分析 InputDispatcher.findFocusedWindowTargetsLocked 函数是如何寻找焦点窗口的。

```
InputEventInjectionResult InputDispatcher::findFocusedWindowTargetsLocked(
        nsecs_t currentTime, const EventEntry& entry, std::vector<InputTarget>& inputTargets,
        nsecs_t* nextWakeupTime) {
    std::string reason;

    int32_t displayId = getTargetDisplayId(entry);
    sp<InputWindowHandle> focusedWindowHandle = getFocusedWindowHandleLocked(displayId);
    std::shared_ptr<InputApplicationHandle> focusedApplicationHandle =
            getValueByKey(mFocusedApplicationHandlesByDisplay, displayId);

    // If there is no currently focused window and no focused application
    // then drop the event.
    if (focusedWindowHandle == nullptr && focusedApplicationHandle == nullptr) {
        ALOGI("Dropping %s event because there is no focused window or focused application in "
              "display %" PRId32 ".",
              NamedEnum::string(entry.type).c_str(), displayId);
        return InputEventInjectionResult::FAILED;
    }

    // Compatibility behavior: raise ANR if there is a focused application, but no focused window.
    // Only start counting when we have a focused event to dispatch. The ANR is canceled if we
    // start interacting with another application via touch (app switch). This code can be removed
    // if the "no focused window ANR" is moved to the policy. Input doesn't know whether
    // an app is expected to have a focused window.
    if (focusedWindowHandle == nullptr && focusedApplicationHandle != nullptr) {
        if (!mNoFocusedWindowTimeoutTime.has_value()) {
            // We just discovered that there's no focused window. Start the ANR timer
            std::chrono::nanoseconds timeout = focusedApplicationHandle->getDispatchingTimeout(
                    DEFAULT_INPUT_DISPATCHING_TIMEOUT);
            mNoFocusedWindowTimeoutTime = currentTime + timeout.count();
            mAwaitedFocusedApplication = focusedApplicationHandle;
            mAwaitedApplicationDisplayId = displayId;
            ALOGW("Waiting because no window has focus but %s may eventually add a "
                  "window when it finishes starting up. Will wait for %" PRId64 "ms",
                  mAwaitedFocusedApplication->getName().c_str(), millis(timeout));
            *nextWakeupTime = *mNoFocusedWindowTimeoutTime;
            return InputEventInjectionResult::PENDING;
        } else if (currentTime > *mNoFocusedWindowTimeoutTime) {
            // Already raised ANR. Drop the event
            ALOGE("Dropping %s event because there is no focused window",
                  NamedEnum::string(entry.type).c_str());
            return InputEventInjectionResult::FAILED;
        } else {
            // Still waiting for the focused window
            return InputEventInjectionResult::PENDING;
        }
    }

    // we have a valid, non-null focused window
    resetNoFocusedWindowTimeoutLocked();

    // Check permissions.
	......

    // If the event is a key event, then we must wait for all previous events to
    // complete before delivering it because previous events may have the
    // side-effect of transferring focus to a different window and we want to
    // ensure that the following keys are sent to the new window.
    //
    // Suppose the user touches a button in a window then immediately presses "A".
    // If the button causes a pop-up window to appear then we want to ensure that
    // the "A" key is delivered to the new pop-up window.  This is because users
    // often anticipate pending UI changes when typing on a keyboard.
    // To obtain this behavior, we must serialize key events with respect to all
    // prior input events.
    if (entry.type == EventEntry::Type::KEY) {
        if (shouldWaitToSendKeyLocked(currentTime, focusedWindowHandle->getName().c_str())) {
            *nextWakeupTime = *mKeyIsWaitingForEventsTimeout;
            return InputEventInjectionResult::PENDING;
        }
    }

    // Success!  Output targets.
    addWindowTargetLocked(focusedWindowHandle,
                          InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS,
                          BitSet32(0), inputTargets);

    // Done.
    return InputEventInjectionResult::SUCCEEDED;
}
```

1)、如果当前没有焦点 Window 和焦点 Application，那么丢弃当前事件。

2)、如果当前只有焦点 App，但是没有焦点 Window，那么启动 [ANR](https://so.csdn.net/so/search?q=ANR&spm=1001.2101.3001.7020) 计时并且开始等待，如果在规定时间内没有找到一个焦点 Window，那么此时已经发生了 ANR，寻找失败；如果找到则继续往下执行，并且重置 ANR 计时。

3)、如果当前事件是按键事件，那么必须等待之前所有的等待处理的事件完成，因为如果之前的事件引起了窗口切换，那么我们希望将当前事件能够传递到切换后的新窗口。

4)、调用 InputDispatcher.addWindowTargetLocked 将焦点窗口添加到 inputTargets 中，这个方法在处理 Motion 事件的时候也会用到，因此后续 Motion 事件分析完后再放到一起说。

从这里看到，焦点窗口的设置在其他地方，另一条线负责寻找焦点窗口并且保存焦点窗口，分发事件这条线只需要把之前保存的焦点窗口取出即可（如果有的话）。

3 Motion 事件
-----------

```
bool InputDispatcher::dispatchMotionLocked(nsecs_t currentTime, std::shared_ptr<MotionEntry> entry,
                                           DropReason* dropReason, nsecs_t* nextWakeupTime) {
	......

    // Identify targets.
    std::vector<InputTarget> inputTargets;

    bool conflictingPointerActions = false;
    InputEventInjectionResult injectionResult;
    if (isPointerEvent) {
        // Pointer event.  (eg. touchscreen)
        injectionResult =
                findTouchedWindowTargetsLocked(currentTime, *entry, inputTargets, nextWakeupTime,
                                               &conflictingPointerActions);
    } else {
        // Non touch event.  (eg. trackball)
        injectionResult =
                findFocusedWindowTargetsLocked(currentTime, *entry, inputTargets, nextWakeupTime);
    }
	......

    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}
```

1)、初始化一个 InputTargets 队列，通过 InputDispatcher.findTouchedWindowTargetsLocked 来往该队列填充数据。

2)、调用 InputDispatcher.dispatchEventLocked 继续分发，这个函数在处理 Key 的时候也用到了，在分析完 InputDispatcher.findTouchedWindowTargetsLocked 之后再说明。

因此本节重点分析 InputDispatcher.findTouchedWindowTargetsLocked 的内容：

```
InputEventInjectionResult InputDispatcher::findTouchedWindowTargetsLocked(
        nsecs_t currentTime, const MotionEntry& entry, std::vector<InputTarget>& inputTargets,
        nsecs_t* nextWakeupTime, bool* outConflictingPointerActions) {
	......

    // Copy current touch state into tempTouchState.
    // This state will be used to update mTouchStatesByDisplay at the end of this function.
    // If no state for the specified display exists, then our initial state will be empty.
    const TouchState* oldState = nullptr;
    TouchState tempTouchState;
    std::unordered_map<int32_t, TouchState>::iterator oldStateIt =
            mTouchStatesByDisplay.find(displayId);
    // 3.1.1 获取上一次寻找的结果
    if (oldStateIt != mTouchStatesByDisplay.end()) {
        oldState = &(oldStateIt->second);
        tempTouchState.copyFrom(*oldState);
    }

	......
    bool newGesture = (maskedAction == AMOTION_EVENT_ACTION_DOWN ||
                       maskedAction == AMOTION_EVENT_ACTION_SCROLL || isHoverAction);
    const bool isFromMouse = entry.source == AINPUT_SOURCE_MOUSE;
    bool wrongDevice = false;
    if (newGesture) {
		......
        // 3.1.2 重置tempTouchState
        tempTouchState.reset();
        tempTouchState.down = down;
        tempTouchState.deviceId = entry.deviceId;
        tempTouchState.source = entry.source;
        tempTouchState.displayId = displayId;
        isSplit = false;
    } else if (switchedDevice && maskedAction == AMOTION_EVENT_ACTION_MOVE) {
		......
    }

    if (newGesture || (isSplit && maskedAction == AMOTION_EVENT_ACTION_POINTER_DOWN)) {
        /* Case 1: New splittable pointer going down, or need target for hover or scroll. */

		......
            
        // 3.2.1 InputDispatcher.findTouchedWindowAtLocked寻找可接收触摸事件的窗口
        newTouchedWindowHandle =
                findTouchedWindowAtLocked(displayId, x, y, &tempTouchState,
                                          isDown /*addOutsideTargets*/, true /*addPortalWindows*/);
        
		// 检验newTouchedWindowHandle的有效性
		......		

        // 3.2.2 添加窗口Flag    
        if (newTouchedWindowHandle != nullptr) {
            // Set target flags.
            int32_t targetFlags = InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS;
            if (isSplit) {
                targetFlags |= InputTarget::FLAG_SPLIT;
            }
            if (isWindowObscuredAtPointLocked(newTouchedWindowHandle, x, y)) {
                targetFlags |= InputTarget::FLAG_WINDOW_IS_OBSCURED;
            } else if (isWindowObscuredLocked(newTouchedWindowHandle)) {
                targetFlags |= InputTarget::FLAG_WINDOW_IS_PARTIALLY_OBSCURED;
            }

            // Update hover state.
			......

            // Update the temporary touch state.
			......
            tempTouchState.addOrUpdateWindow(newTouchedWindowHandle, targetFlags, pointerIds);
        }

        tempTouchState.addGestureMonitors(newGestureMonitors);
    } else {
        /* Case 2: Pointer move, up, cancel or non-splittable pointer down. */

		......
            
        // 3.3 寻找触摸窗口——非DOWN事件的处理
        // Check whether touches should slip outside of the current foreground window.
        if (maskedAction == AMOTION_EVENT_ACTION_MOVE && entry.pointerCount == 1 &&
            tempTouchState.isSlippery()) {
            int32_t x = int32_t(entry.pointerCoords[0].getAxisValue(AMOTION_EVENT_AXIS_X));
            int32_t y = int32_t(entry.pointerCoords[0].getAxisValue(AMOTION_EVENT_AXIS_Y));

            sp<InputWindowHandle> oldTouchedWindowHandle =
                    tempTouchState.getFirstForegroundWindowHandle();
            newTouchedWindowHandle = findTouchedWindowAtLocked(displayId, x, y, &tempTouchState);
            if (oldTouchedWindowHandle != newTouchedWindowHandle &&
                oldTouchedWindowHandle != nullptr && newTouchedWindowHandle != nullptr) {
                if (DEBUG_FOCUS) {
                    ALOGD("Touch is slipping out of window %s into window %s in display %" PRId32,
                          oldTouchedWindowHandle->getName().c_str(),
                          newTouchedWindowHandle->getName().c_str(), displayId);
                }
                // Make a slippery exit from the old window.
                tempTouchState.addOrUpdateWindow(oldTouchedWindowHandle,
                                                 InputTarget::FLAG_DISPATCH_AS_SLIPPERY_EXIT,
                                                 BitSet32(0));

                // Make a slippery entrance into the new window.
                if (newTouchedWindowHandle->getInfo()->supportsSplitTouch()) {
                    isSplit = true;
                }

                int32_t targetFlags =
                        InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER;
                if (isSplit) {
                    targetFlags |= InputTarget::FLAG_SPLIT;
                }
                if (isWindowObscuredAtPointLocked(newTouchedWindowHandle, x, y)) {
                    targetFlags |= InputTarget::FLAG_WINDOW_IS_OBSCURED;
                } else if (isWindowObscuredLocked(newTouchedWindowHandle)) {
                    targetFlags |= InputTarget::FLAG_WINDOW_IS_PARTIALLY_OBSCURED;
                }

                BitSet32 pointerIds;
                if (isSplit) {
                    pointerIds.markBit(entry.pointerProperties[0].id);
                }
                tempTouchState.addOrUpdateWindow(newTouchedWindowHandle, targetFlags, pointerIds);
            }
        }
    }

	......

    // 3.4 进行必要检查
    // Check permission to inject into all touched foreground windows and ensure there
    // is at least one touched foreground window.
    {
        bool haveForegroundWindow = false;
        for (const TouchedWindow& touchedWindow : tempTouchState.windows) {
            if (touchedWindow.targetFlags & InputTarget::FLAG_FOREGROUND) {
                haveForegroundWindow = true;
                if (!checkInjectionPermission(touchedWindow.windowHandle, entry.injectionState)) {
                    injectionResult = InputEventInjectionResult::PERMISSION_DENIED;
                    injectionPermission = INJECTION_PERMISSION_DENIED;
                    goto Failed;
                }
            }
        }
        bool hasGestureMonitor = !tempTouchState.gestureMonitors.empty();
        if (!haveForegroundWindow && !hasGestureMonitor) {
            ALOGI("Dropping event because there is no touched foreground window in display "
                  "%" PRId32 " or gesture monitor to receive it.",
                  displayId);
            injectionResult = InputEventInjectionResult::FAILED;
            goto Failed;
        }

        // Permission granted to injection into all touched foreground windows.
        injectionPermission = INJECTION_PERMISSION_GRANTED;
    }

    // Check whether windows listening for outside touches are owned by the same UID. If it is
    // set the policy flag that we will not reveal coordinate information to this window.
	......

    // If this is the first pointer going down and the touched window has a wallpaper
    // then also add the touched wallpaper windows so they are locked in for the duration
    // of the touch gesture.
    // We do not collect wallpapers during HOVER_MOVE or SCROLL because the wallpaper
    // engine only supports touch events.  We would need to add a mechanism similar
    // to View.onGenericMotionEvent to enable wallpapers to handle these events.
	......

    // Success!  Output targets.
    injectionResult = InputEventInjectionResult::SUCCEEDED;

    // 3.5 通过addWindowTargetLocked将tempTouchState的结果传给inputTargets
    for (const TouchedWindow& touchedWindow : tempTouchState.windows) {
        addWindowTargetLocked(touchedWindow.windowHandle, touchedWindow.targetFlags,
                              touchedWindow.pointerIds, inputTargets);
    }

    for (const TouchedMonitor& touchedMonitor : tempTouchState.gestureMonitors) {
        addMonitoringTargetLocked(touchedMonitor.monitor, touchedMonitor.xOffset,
                                  touchedMonitor.yOffset, inputTargets);
    }

    // 3.1.4 裁剪tempTouchState
    // Drop the outside or hover touch windows since we will not care about them
    // in the next iteration.
    tempTouchState.filterNonAsIsTouchWindows();

Failed:
    // Check injection permission once and for all.
	......

    // Update final pieces of touch state if the injector had permission.
    if (!wrongDevice) {
		......

        // 3.1.3 保存tempTouchState到mTouchStatesByDisplay
        // Save changes unless the action was scroll in which case the temporary touch
        // state was only valid for this one action.
        if (maskedAction != AMOTION_EVENT_ACTION_SCROLL) {
            if (tempTouchState.displayId >= 0) {
                mTouchStatesByDisplay[displayId] = tempTouchState;
            } else {
                mTouchStatesByDisplay.erase(displayId);
            }
        }

        // Update hover state.
        mLastHoverWindowHandle = newHoverWindowHandle;
    }

    return injectionResult;
}
```

这个函数很长，接下来将这个函数分成以下几部分依次分析：

3.1 tempTouchState 作用。

3.2 寻找 Motion 窗口——DOWN 事件的处理。

3.3 寻找 Motion 窗口——非 DOWN 事件的处理。

3.4 进行必要检查

3.5 通过 addWindowTargetLocked 将 tempTouchState 的结果传给 inputTargets。

### 3.1 tempTouchState 作用

在函数的开始创建了一个 TouchState 类型的 tempTouchState：

```
// Copy current touch state into tempTouchState.
    // This state will be used to update mTouchStatesByDisplay at the end of this function.
    // If no state for the specified display exists, then our initial state will be empty.
    const TouchState* oldState = nullptr;
    TouchState tempTouchState;
```

TouchState 是一个显示设备可以接收 Motion 事件的窗口合集，它有一个窗口队列 windows 用来保存当前 display 中所有可以接收 Motion 事件的窗口：

```
int32_t displayId; // id to the display that currently has a touch, others are rejected
    std::vector<TouchedWindow> windows;
    ......
    std::vector<TouchedMonitor> gestureMonitors;
```

InputDispatcher.findTouchedWindowTargetsLocked 函数的最终目的是将所有可以接收当前输入事件的窗口加入到 InputTargets，而我们需要提前将相关窗口信息收集到这里创建的 TouchState 类型的 tempTouchState 中，最后再将 tempTouchState 的结果赋值给 inputTargets，接下来分析为什么要创建一个中间商 tempTouchState。

#### 3.1.1 获取上一次寻找的结果

每次开始寻找 Motion 窗口前，都先将从 mTouchStatesByDisplay 处获得的 oldStated 拷贝给 tempTouchState。

```
// Copy current touch state into tempTouchState.
    // This state will be used to update mTouchStatesByDisplay at the end of this function.
    // If no state for the specified display exists, then our initial state will be empty.
    const TouchState* oldState = nullptr;
    TouchState tempTouchState;
    std::unordered_map<int32_t, TouchState>::iterator oldStateIt =
            mTouchStatesByDisplay.find(displayId);
    if (oldStateIt != mTouchStatesByDisplay.end()) {
        oldState = &(oldStateIt->second);
        tempTouchState.copyFrom(*oldState);
    }
```

InputDispatcher 有一个 mTouchStatesByDisplay 的 Map：

```
std::unordered_map<int32_t, TouchState> mTouchStatesByDisplay GUARDED_BY(mLock);
```

用来储存一个 displayId 和与其对应的 TouchState 对象。

稍后我们可以看到，在该函数的最后，会把 tempTouchState 的结果保存到 mTouchStatesByDisplay 中，因此在函数的开始，tempTouchstate 是获取到了上一次寻找的结果。

#### 3.1.2 重置 tempTouchState

如果当前这一系列事件的起点，如 ACTION_DOWN，那么此次不会复用上一次的结果，并且会清除掉之前保存的状态：

```
bool newGesture = (maskedAction == AMOTION_EVENT_ACTION_DOWN ||
                       maskedAction == AMOTION_EVENT_ACTION_SCROLL || isHoverAction);
    ......
    if (newGesture) {
        ......
        tempTouchState.reset(); 
        tempTouchState.down = down;
        tempTouchState.deviceId = entry.deviceId;
        tempTouchState.source = entry.source;
        tempTouchState.displayId = displayId;
        isSplit = false;
    }
```

很好理解，上一系列 Motion 事件和本系列 Motion 事件是完全独立的，上一系列 Motion 事件不应该影响到本次 Motion 事件中寻找可接收输入事件窗口的结果。

#### 3.1.3 保存 tempTouchState 到 mTouchStatesByDisplay

```
// Save changes unless the action was scroll in which case the temporary touch
        // state was only valid for this one action.
        if (maskedAction != AMOTION_EVENT_ACTION_SCROLL) {
            if (tempTouchState.displayId >= 0) {
                mTouchStatesByDisplay[displayId] = tempTouchState;
            } else {
                mTouchStatesByDisplay.erase(displayId);
            }
        }
```

这样做的是为了下一次再进到这个函数时，直接取上一次执行后的结果，不需要再去遍历所有的窗口寻找可以接收当前输入事件的窗口了。

很好理解，我们在 DOWN 的时候遍历所有窗口，找到了所有可以接收当前输入事件的窗口，那么在后续的 MOVE、UP 等事件的时候，遍历窗口的工作没必要再重复做了。

#### 3.1.4 裁剪 tempTouchState

tempTouchState 中保存了我们寻找所有可以接收 Motion 事件的窗口信息，然后在函数的最后，会对 tempTouchState 进行一些裁剪:

```
// Drop the outside or hover touch windows since we will not care about them
    // in the next iteration.
    tempTouchState.filterNonAsIsTouchWindows();
```

由 TouchState.filterNonAsIsTouchWindow[s 函数](https://so.csdn.net/so/search?q=s%E5%87%BD%E6%95%B0&spm=1001.2101.3001.7020)完成：

```
void TouchState::filterNonAsIsTouchWindows() {
    for (size_t i = 0; i < windows.size();) {
        TouchedWindow& window = windows[i];
        if (window.targetFlags &
            (InputTarget::FLAG_DISPATCH_AS_IS | InputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER)) {
            window.targetFlags &= ~InputTarget::FLAG_DISPATCH_MASK;
            window.targetFlags |= InputTarget::FLAG_DISPATCH_AS_IS;
            i += 1;
        } else {
            windows.erase(windows.begin() + i);
        }
    }
}
```

将那些没有 FLAG_DISPATCH_AS_IS 和 nputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER 这两个 flag 的窗口从 tempTouchState 中移除。

这些 flag 定义在 InputTarget.h 中，这里用到的比较重要的有：

```
/* This flag indicates that the event is being delivered to a foreground application. */
        FLAG_FOREGROUND = 1 << 0,
```

FLAG_FOREGROUND，声明事件正在被发送给一个前台 App。

```
/* This flag indicates that the event should be sent as is.
         * Should always be set unless the event is to be transmuted. */
        FLAG_DISPATCH_AS_IS = 1 << 8,
```

FLAG_DISPATCH_AS_IS，声明事件应该按原样发送。应该总是设置，除非事件被转化。  
这部分需要结合下面的分析才能理解。

### 3.2 寻找 Motion 窗口——DOWN 事件的处理

#### 3.2.1 InputDispatcher.findTouchedWindowAtLocked 寻找可接收 Motion 事件的窗口

每次开启新的 Motion 事件时，用 InputDispatcher.findTouchedWindowAtLocked 去寻找一个可以接收 touch 事件的 Window：

```
bool newGesture = (maskedAction == AMOTION_EVENT_ACTION_DOWN ||
                       maskedAction == AMOTION_EVENT_ACTION_SCROLL || isHoverAction);
	......
	if (newGesture || (isSplit && maskedAction == AMOTION_EVENT_ACTION_POINTER_DOWN)) {
        /* Case 1: New splittable pointer going down, or need target for hover or scroll. */

		......
        newTouchedWindowHandle =
                findTouchedWindowAtLocked(displayId, x, y, &tempTouchState,
                                          isDown /*addOutsideTargets*/, true /*addPortalWindows*/);
        ......
    }
```

这里传入了当前 touch 事件的坐标，以及 tempTouchState。

```
sp<InputWindowHandle> InputDispatcher::findTouchedWindowAtLocked(int32_t displayId, int32_t x,
                                                                 int32_t y, TouchState* touchState,
                                                                 bool addOutsideTargets,
                                                                 bool addPortalWindows,
                                                                 bool ignoreDragWindow) {
    if ((addPortalWindows || addOutsideTargets) && touchState == nullptr) {
        LOG_ALWAYS_FATAL(
                "Must provide a valid touch state if adding portal windows or outside targets");
    }
    // Traverse windows from front to back to find touched window.
    const std::vector<sp<InputWindowHandle>>& windowHandles = getWindowHandlesLocked(displayId);
    for (const sp<InputWindowHandle>& windowHandle : windowHandles) {
        if (ignoreDragWindow && haveSameToken(windowHandle, mDragState->dragWindow)) {
            continue;
        }
        const InputWindowInfo* windowInfo = windowHandle->getInfo();
        if (windowInfo->displayId == displayId) {
            auto flags = windowInfo->flags;

            if (windowInfo->visible) {
                if (!flags.test(InputWindowInfo::Flag::NOT_TOUCHABLE)) {
                    bool isTouchModal = !flags.test(InputWindowInfo::Flag::NOT_FOCUSABLE) &&
                            !flags.test(InputWindowInfo::Flag::NOT_TOUCH_MODAL);
                    if (isTouchModal || windowInfo->touchableRegionContainsPoint(x, y)) {
                        int32_t portalToDisplayId = windowInfo->portalToDisplayId;
                        if (portalToDisplayId != ADISPLAY_ID_NONE &&
                            portalToDisplayId != displayId) {
                            if (addPortalWindows) {
                                // For the monitoring channels of the display.
                                touchState->addPortalWindow(windowHandle);
                            }
                            return findTouchedWindowAtLocked(portalToDisplayId, x, y, touchState,
                                                             addOutsideTargets, addPortalWindows);
                        }
                        // Found window.
                        return windowHandle;
                    }
                }

                if (addOutsideTargets && flags.test(InputWindowInfo::Flag::WATCH_OUTSIDE_TOUCH)) {
                    touchState->addOrUpdateWindow(windowHandle,
                                                  InputTarget::FLAG_DISPATCH_AS_OUTSIDE,
                                                  BitSet32(0));
                }
            }
        }
    }
    return nullptr;
}
```

这里通过 InputDispatcher.getWindowHandlesLocked 得到当前所有窗口信息，窗口信息保存在：

```
std::unordered_map<int32_t, std::vector<sp<InputWindowHandle>>> mWindowHandlesByDisplay
            GUARDED_BY(mLock);
```

该全局变量保存了所有 display 和每一个 display 下所有的窗口，通过 InputDispatcher.updateWindowHandlesForDisplayLocked 来更新，以后也许另开一篇详细说明。

findTouchedWindowAtLocked 函数通过遍历所有窗口，寻找可以接收输入事件的窗口，需要满足以下条件：

1)、当前窗口必须是可见的。

```
2)、当前窗口不能包含InputWindowInfo::Flag::NOT_TOUCHABLE，设置了这个flag的窗口不能接收Motion事件。

    3.1)、当前窗口不能包含InputWindowInfo::Flag::NOT_FOCUSABLE和InputWindowInfo::Flag::NOT_TOUCH_MODAL两个flag，如果某个窗口没有NOT_TOUCH_MODAL这个flag，表示这个窗口将会消费掉所有坐标事件，无论这些事件是否落在了这个窗口区域里面。

    或者

    3.2)、当前输入事件的坐标落在在当前窗口的Motion区域里，那么返回当前窗口。
```

特殊情况是：如果当前窗口包含 InputWindowInfo::Flag::WATCH_OUTSIDE_TOUCH，那么直接调用 TouchState.addOrUpdateWindow 将当前窗口添加到 tempTouchState 中，但是这个窗口无法接收到完整的 down/move/up 手势，只会在 down 的时候接收到一次 MotionEvent.ACTION_OUTSIDE 事件。

#### 3.2.2 添加窗口 Flag

上面找到了一个符合条件的目标窗口后，没有直接通过 TouchState.addOrUpdateWindow 将找到的符合条件的 InputWindowHandle 添加到 tempTouchState 中，而是返回了一个 InputWindowHandle 并且赋值给 newTouchedWindowHandle，是因为我们还想对这个符合条件的 InputWindowHandle 对象添加一些额外的 flag：

```
if (newTouchedWindowHandle != nullptr) {
            // Set target flags.
            int32_t targetFlags = InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS;
            if (isSplit) {
                targetFlags |= InputTarget::FLAG_SPLIT;
            }
            if (isWindowObscuredAtPointLocked(newTouchedWindowHandle, x, y)) {
                targetFlags |= InputTarget::FLAG_WINDOW_IS_OBSCURED;
            } else if (isWindowObscuredLocked(newTouchedWindowHandle)) {
                targetFlags |= InputTarget::FLAG_WINDOW_IS_PARTIALLY_OBSCURED;
            }

            // Update hover state.
            if (maskedAction == AMOTION_EVENT_ACTION_HOVER_EXIT) {
                newHoverWindowHandle = nullptr;
            } else if (isHoverAction) {
                newHoverWindowHandle = newTouchedWindowHandle;
            }

            // Update the temporary touch state.
            BitSet32 pointerIds;
            if (isSplit) {
                uint32_t pointerId = entry.pointerProperties[pointerIndex].id;
                pointerIds.markBit(pointerId);
            }
            tempTouchState.addOrUpdateWindow(newTouchedWindowHandle, targetFlags, pointerIds);
        }
```

这里为找到的 Motion 窗口添加了两个重要的 flag：

```
InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS
```

最后，再通过 TouchState.addOrUpdateWindow 将 newTouchedWindowHandle 带着这些设置的 flag 添加到 tempTouchState 中：

```
void TouchState::addOrUpdateWindow(const sp<InputWindowHandle>& windowHandle, int32_t targetFlags,
                                   BitSet32 pointerIds) {
	......

    TouchedWindow touchedWindow;
    touchedWindow.windowHandle = windowHandle;
    touchedWindow.targetFlags = targetFlags;
    touchedWindow.pointerIds = pointerIds;
    const InputWindowInfo* windowInfo = windowHandle->getInfo();
    windows.push_back(touchedWindow);
}
```

这个函数内容很简单，创建一个 TouchWindow 对象，整合之前的所有信息，然后加入到 tempTouchState 的 windows 队列中。

根据我们在 3.1.1.4 中对调用 TouchState.filterNonAsIsTouchWindows 的分析，只有这里对找到的 Motion 窗口加上 FLAG_DISPATCH_AS_IS，这个窗口才不会在 TouchState.filterNonAsIsTouchWindows 中过滤掉。

与此对应的是，在 findTouchedWindowAtLocked 函数中，我们也会把声明了 InputWindowInfo::Flag::WATCH_OUTSIDE_TOUCH 的窗口通过 addOrUpdateWindow 函数添加到 tempTouchState，但是这个窗口在添加的时候只加了 InputTarget::FLAG_DISPATCH_AS_OUTSIDE 这一个 flag，那么后续这个窗口只会在 DOWN 的时候收到一次事件（收到的还是 outside 事件，而不是 down 事件），后续不会再接收到事件，因此该窗口会在函数的最后被 TouchState.filterNonAsIsTouchWindows 从 tempTouchState 中移除。

### 3.3 寻找 Motion 窗口——非 DOWN 事件的处理

```
/* Case 2: Pointer move, up, cancel or non-splittable pointer down. */

        // If the pointer is not currently down, then ignore the event.
		......

        // Check whether touches should slip outside of the current foreground window.
        if (maskedAction == AMOTION_EVENT_ACTION_MOVE && entry.pointerCount == 1 &&
            tempTouchState.isSlippery()) {
            int32_t x = int32_t(entry.pointerCoords[0].getAxisValue(AMOTION_EVENT_AXIS_X));
            int32_t y = int32_t(entry.pointerCoords[0].getAxisValue(AMOTION_EVENT_AXIS_Y));

            sp<InputWindowHandle> oldTouchedWindowHandle =
                    tempTouchState.getFirstForegroundWindowHandle();
            newTouchedWindowHandle = findTouchedWindowAtLocked(displayId, x, y, &tempTouchState);
            if (oldTouchedWindowHandle != newTouchedWindowHandle &&
                oldTouchedWindowHandle != nullptr && newTouchedWindowHandle != nullptr) {
                if (DEBUG_FOCUS) {
                    ALOGI("Touch is slipping out of window %s into window %s in display %" PRId32,
                          oldTouchedWindowHandle->getName().c_str(),
                          newTouchedWindowHandle->getName().c_str(), displayId);
                }
                // Make a slippery exit from the old window.
                tempTouchState.addOrUpdateWindow(oldTouchedWindowHandle,
                                                 InputTarget::FLAG_DISPATCH_AS_SLIPPERY_EXIT,
                                                 BitSet32(0));

                // Make a slippery entrance into the new window.
                if (newTouchedWindowHandle->getInfo()->supportsSplitTouch()) {
                    isSplit = true;
                }

                int32_t targetFlags =
                        InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER;
                if (isSplit) {
                    targetFlags |= InputTarget::FLAG_SPLIT;
                }
                if (isWindowObscuredAtPointLocked(newTouchedWindowHandle, x, y)) {
                    targetFlags |= InputTarget::FLAG_WINDOW_IS_OBSCURED;
                } else if (isWindowObscuredLocked(newTouchedWindowHandle)) {
                    targetFlags |= InputTarget::FLAG_WINDOW_IS_PARTIALLY_OBSCURED;
                }

                BitSet32 pointerIds;
                if (isSplit) {
                    pointerIds.markBit(entry.pointerProperties[0].id);
                }
                tempTouchState.addOrUpdateWindow(newTouchedWindowHandle, targetFlags, pointerIds);
            }
        }
```

这个流程的主要内容是处理一种特殊情况：

1)、当前是 MOVE 事件。

2)、当前只有单点触摸屏幕。

3)、当前的前台窗口中有设置 InputWindowInfo::Flag::SLIPPERY 这个 flag。

当这 3 个条件同时满足的时候，会重新调用 InputDispatcher.findTouchedWindowAtLocked 去重新寻找一个可以接收当前输入事件的窗口，如果找到一个不同的窗口 newTouchedWindowHandle（其他函数里可能会通过 TouchState.addOrUpdateWindow 向 tempTouchState.windows 中添加窗口，如 InputDispatcher.transferTouchFocus），那么会通过：

```
// Make a slippery exit from the old window.
                tempTouchState.addOrUpdateWindow(oldTouchedWindowHandle,
                                                 InputTarget::FLAG_DISPATCH_AS_SLIPPERY_EXIT,
                                                 BitSet32(0));
```

把旧的前台窗口 oldTouchedWindowHandle 从 tempTouchState 移除掉。

将 oldTouchedWindowHandle 从 tempTouchState 移除掉，是通过调用 TouchState.addOrUpdateWindow，传入 InputTarget::FLAG_DISPATCH_AS_SLIPPERY_EXIT 这个 flag：

```
void TouchState::addOrUpdateWindow(const sp<InputWindowHandle>& windowHandle, int32_t targetFlags,
                                   BitSet32 pointerIds) {
    if (targetFlags & InputTarget::FLAG_SPLIT) {
        split = true;
    }

    for (size_t i = 0; i < windows.size(); i++) {
        TouchedWindow& touchedWindow = windows[i];
        if (touchedWindow.windowHandle == windowHandle) {
            touchedWindow.targetFlags |= targetFlags;
            if (targetFlags & InputTarget::FLAG_DISPATCH_AS_SLIPPERY_EXIT) {
                touchedWindow.targetFlags &= ~InputTarget::FLAG_DISPATCH_AS_IS;
            }
            touchedWindow.pointerIds.value |= pointerIds.value;
            return;
        }
    }

    TouchedWindow touchedWindow;
    touchedWindow.windowHandle = windowHandle;
    touchedWindow.targetFlags = targetFlags;
    touchedWindow.pointerIds = pointerIds;
    const InputWindowInfo* windowInfo = windowHandle->getInfo();
    windows.push_back(touchedWindow);
}
```

在这里，由于检测到了 InputTarget::FLAG_DISPATCH_AS_SLIPPERY_EXIT，会把 oldTouchedWindowHandle 的 InputTarget::FLAG_DISPATCH_AS_IS 这个 flag 去除掉，那么这旧前台窗口后续会因为函数最后调用 TouchState.filterNonAsIsTouchWindows 而被移除掉。

```
int32_t targetFlags =
                        InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER;
                ......
                tempTouchState.addOrUpdateWindow(newTouchedWindowHandle, targetFlags, pointerIds);
```

最后重新找到的窗口 newTouchedWindowHandle 会添加到 tempTouchState，只不过这次添加的 flag 是：

```
InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER
```

对比之前添加的是：

```
InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS
```

FLAG_DISPATCH_AS_SLIPPERY_ENTER 的定义是：

```
/* This flag indicates that the event should be dispatched as an initial down.
     * It is used to transmute ACTION_MOVE into ACTION_DOWN when a touch slips
     * into a new window. */
    FLAG_DISPATCH_AS_SLIPPERY_ENTER = 1 << 13,
```

因为我们是在 ACTION_MOVE 的情况下检测到新的前台窗口，为了让前台窗口能接收到一次完整的 Motion 事件，需要将此次事件转化为 ACTION_MOVE 转化为 ACTION_DOWN。

遇到过一次这种情况：

点击了一个设置了 InputWindowInfo::Flag::SLIPPERY 的窗口 A，然后在抬起之前，触摸过程中，系统添加了一个窗口 B（或者窗口 B 由原来的不可见变为可见），窗口 B 由于层级高于窗口 A，导致 findTouchedWindowAtLocked 这里返回了窗口 B，后续改为由窗口 B 接收输入事件，窗口 A 无法再接收到输入事件。

### 3.4 进行必要检查

挑一处我认为比较重要的地方分析：

```
// Check permission to inject into all touched foreground windows and ensure there
    // is at least one touched foreground window.
    {
        bool haveForegroundWindow = false;
        for (const TouchedWindow& touchedWindow : tempTouchState.windows) {
            if (touchedWindow.targetFlags & InputTarget::FLAG_FOREGROUND) {
                haveForegroundWindow = true;
                if (!checkInjectionPermission(touchedWindow.windowHandle, entry.injectionState)) {
                    injectionResult = InputEventInjectionResult::PERMISSION_DENIED;
                    injectionPermission = INJECTION_PERMISSION_DENIED;
                    goto Failed;
                }
            }
        }
        bool hasGestureMonitor = !tempTouchState.gestureMonitors.empty();
        if (!haveForegroundWindow && !hasGestureMonitor) {
            ALOGI("Dropping event because there is no touched foreground window in display "
                  "%" PRId32 " or gesture monitor to receive it.",
                  displayId);
            injectionResult = InputEventInjectionResult::FAILED;
            goto Failed;
        }

        // Permission granted to injection into all touched foreground windows.
        injectionPermission = INJECTION_PERMISSION_GRANTED;
    }
```

检查当前是否至少有一个前台窗口或者 gesture monitor 可以接收当前输入事件，如果没有那么不传递本次事件，输出 log：

```
ALOGI("Dropping event because there is no touched foreground window in display "
                  "%" PRId32 " or gesture monitor to receive it.",
                  displayId);
```

ANR 有时候会打印这句 log。

gesture monitor 用来接收 pointer 事件，比如开启全面屏手势后，就会注册 gesture monitor。

InputDispatcher.findTouchedWindowTargetsLocked 中为触摸窗口添加 InputTarget::FLAG_FOREGROUND 的地方只有两处：

1)、在 down 流程中，当通过 InputDispatcher.findTouchedWindowAtLocked 找到了一个可以接收当前 Motion 事件的窗口时，为该其添加该 flag。

2)、在 move 流程中的 slippery 情况。

情况 2 比较少见，主要是情况 1。

想像这样一种情况，我们在 Window1 上滑动，然后可以通过 [adb 命令](https://so.csdn.net/so/search?q=adb%E5%91%BD%E4%BB%A4&spm=1001.2101.3001.7020)去启动 Window2，那么此时 Window1 会从 tempTouchState.windows 中移除，但是 Window2 也不会加上 FLAG_FOREGROUND 的 flag，这会导致此时没有一个前台窗口来接收接下来的事件，导致事件分发流程直接在这里返回，不会再往下传递，那么不仅是所有窗口无法再接收到输入事件，我们注册的一些 PointerEventListener 也无法再接收到事件了（但仅仅是这样并不会发生 ANR）。

### 3.5 通过 addWindowTargetLocked 将 tempTouchState 的结果传给 inputTargets

```
for (const TouchedWindow& touchedWindow : tempTouchState.windows) {
        addWindowTargetLocked(touchedWindow.windowHandle, touchedWindow.targetFlags,
                              touchedWindow.pointerIds, inputTargets);
    }

    for (const TouchedMonitor& touchedMonitor : tempTouchState.gestureMonitors) {
        addMonitoringTargetLocked(touchedMonitor.monitor, touchedMonitor.xOffset,
                                  touchedMonitor.yOffset, inputTargets);
    }

    // Drop the outside or hover touch windows since we will not care about them
    // in the next iteration.
    tempTouchState.filterNonAsIsTouchWindows();
```

函数的最后，通过 InputDispatcher.addWindowTargetLocked 完成 inputTargets 的赋值：

```
void InputDispatcher::addWindowTargetLocked(const sp<InputWindowHandle>& windowHandle,
                                            int32_t targetFlags, BitSet32 pointerIds,
                                            std::vector<InputTarget>& inputTargets) {
    std::vector<InputTarget>::iterator it =
            std::find_if(inputTargets.begin(), inputTargets.end(),
                         [&windowHandle](const InputTarget& inputTarget) {
                             return inputTarget.inputChannel->getConnectionToken() ==
                                     windowHandle->getToken();
                         });

    const InputWindowInfo* windowInfo = windowHandle->getInfo();

    if (it == inputTargets.end()) {
        InputTarget inputTarget;
        std::shared_ptr<InputChannel> inputChannel =
                getInputChannelLocked(windowHandle->getToken());
        if (inputChannel == nullptr) {
            ALOGW("Window %s already unregistered input channel", windowHandle->getName().c_str());
            return;
        }
        inputTarget.inputChannel = inputChannel;
        inputTarget.flags = targetFlags;
        inputTarget.globalScaleFactor = windowInfo->globalScaleFactor;
        inputTarget.displaySize =
                int2(windowHandle->getInfo()->displayWidth, windowHandle->getInfo()->displayHeight);
        inputTargets.push_back(inputTarget);
        it = inputTargets.end() - 1;
    }

    ALOG_ASSERT(it->flags == targetFlags);
    ALOG_ASSERT(it->globalScaleFactor == windowInfo->globalScaleFactor);

    it->addPointers(pointerIds, windowInfo->transform);
}
```

经过上面的一系列判断后，最终 tempTouchState 的 windows 队列保存了所有可以接收当前输入事件的窗口。不管是按键事件还是 Motion 事件，最终都是调用 InputDispatcher.addWindowTargetLocked 将这些窗口加入到了 inputTargets 中：

1)、首先检查当前焦点窗口是否已经加入到 inputTargets，避免重复加入。

2)、如果 inputTargets 里还没有，那么根据焦点窗口的 IBinder 类型的 token 找到对应的 InputChannel，然后根据该 InputWindowHandle 创建一个对应的 InputTarget 对象并添加到 inputTargets 中。之前有分析过，在为 Server 端和 Client 端创建 InputChannel 对的时候，会创建一个 BBinder 对象，并且将键值对 <IBinder token, Connection connection> 加入到了 InputDispatcher 维护的 mConnectionsByToken 中，并且该 token 后续会返回给 InputWindowHandle，那么就可以根据 InputWindowHandle 存储的 IBinder 对象找到一个对应的 Connection 对象，进而找到一个 InputChannel 对象。

另外注意到裁剪的工作是在 tempTouchState 的结果传给 inputTargets 之后才做的，那么就说明本次搜索的结果有效，这些被裁剪的窗口在本次事件分发中仍然可以收到事件，要在下一轮分发的时候这些窗口才会被过滤掉。

4 InputDispatcher.dispatchEventLocked
-------------------------------------

```
void InputDispatcher::dispatchEventLocked(nsecs_t currentTime,
                                          std::shared_ptr<EventEntry> eventEntry,
                                          const std::vector<InputTarget>& inputTargets) {
	......

    for (const InputTarget& inputTarget : inputTargets) {
        sp<Connection> connection =
                getConnectionLocked(inputTarget.inputChannel->getConnectionToken());
        if (connection != nullptr) {
            prepareDispatchCycleLocked(currentTime, connection, eventEntry, inputTarget);
        } else {
			......
        }
    }
}
```

上面无论是 Key 还是 Motion，最后都在寻找完可以接收当前输入事件的窗口后，将寻找结果放到了 inputTargets 中，然后调用了 InputDispatcher.dispatchEventLocked。该函数的主要工作是：

1)、遍历传入的 inputTargets，调用 InputDispatcher.prepareDispatchCycleLocked 函数来分发事件，这里说明了对于一个输入事件来说，会传递给多个窗口进行处理。

2)、之前分析过，对于每一个服务端 InputChannel，InputDispatcher 都创建了一个 Connection 对象来保存这个 InputChannel 对象，这个 Connection 对象，可以通过 IBinder 类型的 token 来检索得到，那么这里便可以通过 InputTarget 的 InputChannel 对象中的 token 来得到持有服务端 InputChannel 的 Connection 对象。

5 InputDispatcher.prepareDispatchCycleLocked
--------------------------------------------

```
void InputDispatcher::prepareDispatchCycleLocked(nsecs_t currentTime,
                                                 const sp<Connection>& connection,
                                                 std::shared_ptr<EventEntry> eventEntry,
                                                 const InputTarget& inputTarget) {
	......

    // Split a motion event if needed.
    if (inputTarget.flags & InputTarget::FLAG_SPLIT) {
		......
    }

    // Not splitting.  Enqueue dispatch entries for the event as is.
    enqueueDispatchEntriesLocked(currentTime, connection, eventEntry, inputTarget);
}
```

这个方法主要判断当前输入事件是否支持拆分，如果支持那么把当前事件拆分。

FLAG_SPLIT 用来声明当前输入事件是否支持被拆分，分给多个窗口（遇到过一种情况，在平行视界下，同时在两侧的 Activity 界面下分别使用一根手指进行上下滑动）。

最终调用 InputDispatcher.enqueueDispatchEntriesLocked 继续进行分发。

6 InputDispatcher.enqueueDispatchEntriesLocked
----------------------------------------------

```
void InputDispatcher::enqueueDispatchEntriesLocked(nsecs_t currentTime,
                                                   const sp<Connection>& connection,
                                                   std::shared_ptr<EventEntry> eventEntry,
                                                   const InputTarget& inputTarget) {
    if (ATRACE_ENABLED()) {
        std::string message =
                StringPrintf("enqueueDispatchEntriesLocked(inputChannel=%s, id=0x%" PRIx32 ")",
                             connection->getInputChannelName().c_str(), eventEntry->id);
        ATRACE_NAME(message.c_str());
    }

    bool wasEmpty = connection->outboundQueue.empty();

    // Enqueue dispatch entries for the requested modes.
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_HOVER_EXIT);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_OUTSIDE);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_HOVER_ENTER);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_IS);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_SLIPPERY_EXIT);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER);

    // If the outbound queue was previously empty, start the dispatch cycle going.
    if (wasEmpty && !connection->outboundQueue.empty()) {
        startDispatchCycleLocked(currentTime, connection);
    }
}
```

1)、enqueueDispatchEntryLocked 函数内容不再详细分析，大概的作用是，如果目标窗口设置了以下集中 flag，表示目标窗口需要处理这类事件，那么就把输入事件封装为相应的 DispatchEntry 类型，加入到 Connection.outBoundQueue 队列中：

```
/* This flag indicates that the event should be sent as is.
         * Should always be set unless the event is to be transmuted. */
        FLAG_DISPATCH_AS_IS = 1 << 8,

        /* This flag indicates that a MotionEvent with AMOTION_EVENT_ACTION_DOWN falls outside
         * of the area of this target and so should instead be delivered as an
         * AMOTION_EVENT_ACTION_OUTSIDE to this target. */
        FLAG_DISPATCH_AS_OUTSIDE = 1 << 9,

        /* This flag indicates that a hover sequence is starting in the given window.
         * The event is transmuted into ACTION_HOVER_ENTER. */
        FLAG_DISPATCH_AS_HOVER_ENTER = 1 << 10,

        /* This flag indicates that a hover event happened outside of a window which handled
         * previous hover events, signifying the end of the current hover sequence for that
         * window.
         * The event is transmuted into ACTION_HOVER_ENTER. */
        FLAG_DISPATCH_AS_HOVER_EXIT = 1 << 11,

        /* This flag indicates that the event should be canceled.
         * It is used to transmute ACTION_MOVE into ACTION_CANCEL when a touch slips
         * outside of a window. */
        FLAG_DISPATCH_AS_SLIPPERY_EXIT = 1 << 12,

        /* This flag indicates that the event should be dispatched as an initial down.
         * It is used to transmute ACTION_MOVE into ACTION_DOWN when a touch slips
         * into a new window. */
        FLAG_DISPATCH_AS_SLIPPERY_ENTER = 1 << 13,
```

2)、如果 Coonection.outBoundQueue 队列之前为空，但是经过 enqueueDispatchEntryLocked 后不为空，那么调用 InputDispatcher.startDispatchCycleLocked 开始分发事件。

7 InputDispatcher.startDispatchCycleLocked
------------------------------------------

```
void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
                                               const sp<Connection>& connection) {
	......

    while (connection->status == Connection::STATUS_NORMAL && !connection->outboundQueue.empty()) {
		......
        switch (eventEntry.type) {
            case EventEntry::Type::KEY: {
                // 7.1 分发Key事件
                const KeyEntry& keyEntry = static_cast<const KeyEntry&>(eventEntry);
                std::array<uint8_t, 32> hmac = getSignature(keyEntry, *dispatchEntry);

                // Publish the key event.
                status = connection->inputPublisher
                                 .publishKeyEvent(dispatchEntry->seq,
                                                  dispatchEntry->resolvedEventId, keyEntry.deviceId,
                                                  keyEntry.source, keyEntry.displayId,
                                                  std::move(hmac), dispatchEntry->resolvedAction,
                                                  dispatchEntry->resolvedFlags, keyEntry.keyCode,
                                                  keyEntry.scanCode, keyEntry.metaState,
                                                  keyEntry.repeatCount, keyEntry.downTime,
                                                  keyEntry.eventTime);
                break;
            }

            case EventEntry::Type::MOTION: {
                // 7.2 分发Motion事件
				......

                // Publish the motion event.
                status = connection->inputPublisher
                                 .publishMotionEvent(dispatchEntry->seq,
                                                     dispatchEntry->resolvedEventId,
                                                     motionEntry.deviceId, motionEntry.source,
                                                     motionEntry.displayId, std::move(hmac),
                                                     dispatchEntry->resolvedAction,
                                                     motionEntry.actionButton,
                                                     dispatchEntry->resolvedFlags,
                                                     motionEntry.edgeFlags, motionEntry.metaState,
                                                     motionEntry.buttonState,
                                                     motionEntry.classification,
                                                     dispatchEntry->transform,
                                                     motionEntry.xPrecision, motionEntry.yPrecision,
                                                     motionEntry.xCursorPosition,
                                                     motionEntry.yCursorPosition,
                                                     dispatchEntry->displaySize.x,
                                                     dispatchEntry->displaySize.y,
                                                     motionEntry.downTime, motionEntry.eventTime,
                                                     motionEntry.pointerCount,
                                                     motionEntry.pointerProperties, usingCoords);
                break;
            }

            case EventEntry::Type::FOCUS: {
				......
            }

            case EventEntry::Type::POINTER_CAPTURE_CHANGED: {
				......
            }

            case EventEntry::Type::DRAG: {
				......
            }

            case EventEntry::Type::CONFIGURATION_CHANGED:
            case EventEntry::Type::DEVICE_RESET:
            case EventEntry::Type::SENSOR: {
				......
            }
        }

        // Check the result.
		......
    }
}
```

这里循环遍历传入的 Connection.outboundQueue，然后针对不同类型的事件，分别走相应的流程去处理，这里我们只关心 KEY 和 MOTION 两种类型。

### 7.1 分发 Key 事件

```
status_t InputPublisher::publishKeyEvent(uint32_t seq, int32_t eventId, int32_t deviceId,
                                         int32_t source, int32_t displayId,
                                         std::array<uint8_t, 32> hmac, int32_t action,
                                         int32_t flags, int32_t keyCode, int32_t scanCode,
                                         int32_t metaState, int32_t repeatCount, nsecs_t downTime,
                                         nsecs_t eventTime) {

    ......

    InputMessage msg;
    msg.header.type = InputMessage::Type::KEY;
    msg.header.seq = seq;
    msg.body.key.eventId = eventId;
    msg.body.key.deviceId = deviceId;
    msg.body.key.source = source;
    msg.body.key.displayId = displayId;
    msg.body.key.hmac = std::move(hmac);
    msg.body.key.action = action;
    msg.body.key.flags = flags;
    msg.body.key.keyCode = keyCode;
    msg.body.key.scanCode = scanCode;
    msg.body.key.metaState = metaState;
    msg.body.key.repeatCount = repeatCount;
    msg.body.key.downTime = downTime;
    msg.body.key.eventTime = eventTime;
    return mChannel->sendMessage(&msg);
}
```

把输入事件的信息重新封装到一个 InputMessage 类型的结构体中，然后调用 InputChannel.sendMessage。

根据 InputMessage 结构体的注释，这是用于发送输入事件和相关信号的中间表示，用于 IPC 通信，通过 socket 发送。

### 7.2 分发 Motion 事件

```
status_t InputPublisher::publishMotionEvent(
        uint32_t seq, int32_t eventId, int32_t deviceId, int32_t source, int32_t displayId,
        std::array<uint8_t, 32> hmac, int32_t action, int32_t actionButton, int32_t flags,
        int32_t edgeFlags, int32_t metaState, int32_t buttonState,
        MotionClassification classification, const ui::Transform& transform, float xPrecision,
        float yPrecision, float xCursorPosition, float yCursorPosition, int32_t displayWidth,
        int32_t displayHeight, nsecs_t downTime, nsecs_t eventTime, uint32_t pointerCount,
        const PointerProperties* pointerProperties, const PointerCoords* pointerCoords) {

    ......

    InputMessage msg;
    msg.header.type = InputMessage::Type::MOTION;
    msg.header.seq = seq;
    msg.body.motion.eventId = eventId;
    msg.body.motion.deviceId = deviceId;
    msg.body.motion.source = source;
    msg.body.motion.displayId = displayId;
    msg.body.motion.hmac = std::move(hmac);
    msg.body.motion.action = action;
    msg.body.motion.actionButton = actionButton;
    msg.body.motion.flags = flags;
    msg.body.motion.edgeFlags = edgeFlags;
    msg.body.motion.metaState = metaState;
    msg.body.motion.buttonState = buttonState;
    msg.body.motion.classification = classification;
    msg.body.motion.dsdx = transform.dsdx();
    msg.body.motion.dtdx = transform.dtdx();
    msg.body.motion.dtdy = transform.dtdy();
    msg.body.motion.dsdy = transform.dsdy();
    msg.body.motion.tx = transform.tx();
    msg.body.motion.ty = transform.ty();
    msg.body.motion.xPrecision = xPrecision;
    msg.body.motion.yPrecision = yPrecision;
    msg.body.motion.xCursorPosition = xCursorPosition;
    msg.body.motion.yCursorPosition = yCursorPosition;
    msg.body.motion.displayWidth = displayWidth;
    msg.body.motion.displayHeight = displayHeight;
    msg.body.motion.downTime = downTime;
    msg.body.motion.eventTime = eventTime;
    msg.body.motion.pointerCount = pointerCount;
    for (uint32_t i = 0; i < pointerCount; i++) {
        msg.body.motion.pointers[i].properties.copyFrom(pointerProperties[i]);
        msg.body.motion.pointers[i].coords.copyFrom(pointerCoords[i]);
    }

    return mChannel->sendMessage(&msg);
}
```

把输入事件的信息重新封装到一个 InputMessage 类型的结构体中，然后调用 InputChannel.sendMessage。

8 InputChannel.sendMessage
--------------------------

```
status_t InputChannel::sendMessage(const InputMessage* msg) {
    const size_t msgLength = msg->size();
    InputMessage cleanMsg;
    msg->getSanitizedCopy(&cleanMsg);
    ssize_t nWrite;
    do {
        nWrite = ::send(getFd(), &cleanMsg, msgLength, MSG_DONTWAIT | MSG_NOSIGNAL);
    } while (nWrite == -1 && errno == EINTR);
    
    ......
}
```

这里调用 socket 的 send 函数向服务端 InputChannel 保存的 socket 文件描述符发送封装好的 InputMessage 信息。

之前我们分析过了 ViewRootImpl#setView 中会创建一个 WindowInputEventReceiver 对象，进而构建一个 NativeInputEventReceiver 对象，并且在初始化的过程中，会调用：

```
mMessageQueue->getLooper()->addFd(fd, 0, events, this, nullptr);
```

来对客户端 InputChannel 中保存的客户端 socket 文件描述符进行监听，如果服务端有数据写入那么就调用 NativeInputEventReceiver.handleEvent 回调，那么现在就是回调触发的时机。

9 NativeInputEventReceiver.handleEvent
--------------------------------------

```
int NativeInputEventReceiver::handleEvent(int receiveFd, int events, void* data) {
    // Allowed return values of this function as documented in LooperCallback::handleEvent
	......

    if (events & ALOOPER_EVENT_INPUT) {
        JNIEnv* env = AndroidRuntime::getJNIEnv();
        status_t status = consumeEvents(env, false /*consumeBatches*/, -1, nullptr);
        mMessageQueue->raiseAndClearException(env, "handleReceiveCallback");
        return status == OK || status == NO_MEMORY ? KEEP_CALLBACK : REMOVE_CALLBACK;
    }

	......
}
```

ALOOPER_EVENT_INPUT 表示文件描述符可读，我们在注册 NativeInputEventReceiver 的时候，这里传入的 events 类型就是 ALOOPER_EVENT_INPUT：

```
status_t NativeInputEventReceiver::initialize() {
    setFdEvents(ALOOPER_EVENT_INPUT);
    return OK;
}
```

因此走 ALOOPER_EVENT_INPUT 流程，继续调用 NativeInputEventReceiver.consumeEvents。

10 NativeInputEventReceiver.consumeEvents
-----------------------------------------

```
status_t NativeInputEventReceiver::consumeEvents(JNIEnv* env,
        bool consumeBatches, nsecs_t frameTime, bool* outConsumedBatch) {
	......
    for (;;) {
        uint32_t seq;
        InputEvent* inputEvent;

        // 10.1 读取客户端InputChannel发送的事件并转化为InputEvent
        status_t status = mInputConsumer.consume(&mInputEventFactory,
                consumeBatches, frameTime, &seq, &inputEvent);
		......

        if (!skipCallbacks) {
            // 10.2 获取到Java对象InputEventReceiver
            if (!receiverObj.get()) {
                receiverObj.reset(jniGetReferent(env, mReceiverWeakGlobal));
				......
            }

            jobject inputEventObj;
            switch (inputEvent->getType()) {
            case AINPUT_EVENT_TYPE_KEY:
                if (kDebugDispatchCycle) {
                    ALOGD("channel '%s' ~ Received key event.", getInputChannelName().c_str());
                }
                // 10.3
                inputEventObj = android_view_KeyEvent_fromNative(env,
                        static_cast<KeyEvent*>(inputEvent));
                break;

            case AINPUT_EVENT_TYPE_MOTION: {
                if (kDebugDispatchCycle) {
                    ALOGD("channel '%s' ~ Received motion event.", getInputChannelName().c_str());
                }
                // 10.3 将Native层输入事件对象转换为上层输入事件对象
                MotionEvent* motionEvent = static_cast<MotionEvent*>(inputEvent);
                if ((motionEvent->getAction() & AMOTION_EVENT_ACTION_MOVE) && outConsumedBatch) {
                    *outConsumedBatch = true;
                }
                inputEventObj = android_view_MotionEvent_obtainAsCopy(env, motionEvent);
                break;
            }
            case AINPUT_EVENT_TYPE_FOCUS: {
				....
            }
            case AINPUT_EVENT_TYPE_CAPTURE: {
				....
            }
            case AINPUT_EVENT_TYPE_DRAG: {
				....
            }

            default:
                assert(false); // InputConsumer should prevent this from ever happening
                inputEventObj = nullptr;
            }

            if (inputEventObj) {
                if (kDebugDispatchCycle) {
                    ALOGD("channel '%s' ~ Dispatching input event.", getInputChannelName().c_str());
                }
                // 10.4 调用Java层对象receiverObj 的dispatchInputEvent方法
                env->CallVoidMethod(receiverObj.get(),
                        gInputEventReceiverClassInfo.dispatchInputEvent, seq, inputEventObj);
				....
            } else {
				......
            }
        }

		....
    }
}
```

### 10.1 读取客户端 InputChannel 发送的事件并转化为 InputEvent

调用 InputConsumer.comsume 函数，将输入事件转化为一个 InputEvent 对象。

```
status_t InputConsumer::consume(InputEventFactoryInterface* factory, bool consumeBatches,
                                nsecs_t frameTime, uint32_t* outSeq, InputEvent** outEvent) {
	......

    *outSeq = 0;
    *outEvent = nullptr;

    // Fetch the next input message.
    // Loop until an event can be returned or no additional events are received.
    while (!*outEvent) {
        if (mMsgDeferred) {
            // mMsg contains a valid input message from the previous call to consume
            // that has not yet been processed.
            mMsgDeferred = false;
        } else {
            // 10.1.1 InputChannel读取事件
            // Receive a fresh message.
            status_t result = mChannel->receiveMessage(&mMsg);
			......
        }

        switch (mMsg.header.type) {
            case InputMessage::Type::KEY: {
                // 10.1.2 创建和初始化输入事件
                KeyEvent* keyEvent = factory->createKeyEvent();
                if (!keyEvent) return NO_MEMORY;

                initializeKeyEvent(keyEvent, &mMsg);
                *outSeq = mMsg.header.seq;
                *outEvent = keyEvent;
				......
            }

            case InputMessage::Type::MOTION: {
                // 10.1.2 创建和初始化输入事件
				......

                MotionEvent* motionEvent = factory->createMotionEvent();
                if (!motionEvent) return NO_MEMORY;

                updateTouchState(mMsg);
                initializeMotionEvent(motionEvent, &mMsg);
                *outSeq = mMsg.header.seq;
                *outEvent = motionEvent;
				......
            }

            case InputMessage::Type::FINISHED:
            case InputMessage::Type::TIMELINE: {
			......
            }

            case InputMessage::Type::FOCUS: {
			......
            }

            case InputMessage::Type::CAPTURE: {
			......
            }

            case InputMessage::Type::DRAG: {
			......
            }
        }
    }
    return OK;
}
```

#### 10.1.1 InputChannel 读取事件

这里的 mChannel 即是客户端对应的 InputChannel 对象，是在客户端创建 InputEventReceiver 的时候传入的。

回看第 8 步中，服务端 InputChannel 调用 InputChannel.sendMessage，将输入事件写入服务端 InputChannel 对应的文件描述符发送缓冲区：

```
status_t InputChannel::sendMessage(const InputMessage* msg) {
    const size_t msgLength = msg->size();
    InputMessage cleanMsg;
    msg->getSanitizedCopy(&cleanMsg);
    ssize_t nWrite;
    do {
        nWrite = ::send(getFd(), &cleanMsg, msgLength, MSG_DONTWAIT | MSG_NOSIGNAL);
    } while (nWrite == -1 && errno == EINTR);
    
    ......
}
```

那么我们在这里就可以通过 InputChannel.receiverMessage，将输入信息从客户端 InputChannel 对应的文件描述符的接收缓冲区中读取出来：

```
status_t InputChannel::receiveMessage(InputMessage* msg) {
    ssize_t nRead;
    do {
        nRead = ::recv(getFd(), msg, sizeof(InputMessage), MSG_DONTWAIT);
    } while (nRead == -1 && errno == EINTR);

	......
}
```

最终输入事件被存入 InputMessage 类型的 msg 中。

#### 10.1.2 创建和初始化输入事件

针对不同的事件类型创建不同的事件对象，比如 KEY 事件为 KeyEvent，MOTION 事件为 MotionEvent 等，然后将上一步中 msg 的信息取出传给这些新创建的对象，最后将传入的 outEvent 指向新创建的事件对象。

### 10.2 获取到 Java 对象 InputEventReceiver

receiverObj 会被重置为 mReceiverWeakGlobal：

```
receiverObj.reset(jniGetReferent(env, mReceiverWeakGlobal));
```

我们之前在 ViewRootImpl 创建 WindowInputEventReceiver 的时候，会把 Java 层的 InputEventReceiver 对象的弱引用作为初始化 NativeInputEventReceiver 的参数，即：

```
mReceiverPtr = nativeInit(new WeakReference<InputEventReceiver>(this),
                inputChannel, mMessageQueue);
```

后续 NativeInputEventReceiver 的全局变量 mReceiverWeakGlobal 便会持有这个引用：

```
NativeInputEventReceiver::NativeInputEventReceiver(
        JNIEnv* env, jobject receiverWeak, const std::shared_ptr<InputChannel>& inputChannel,
        const sp<MessageQueue>& messageQueue)
      : mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
```

那么这里 receiverObj 指向的便是 Java 层的 InputEventReceiver。

### 10.3 将 Native 层输入事件对象转换为上层输入事件对象

按键事件：

```
inputEventObj = android_view_KeyEvent_fromNative(env,
                        static_cast<KeyEvent*>(inputEvent));
```

Motion 事件：

```
MotionEvent* motionEvent = static_cast<MotionEvent*>(inputEvent);
                ......
                // 10.3
                inputEventObj = android_view_MotionEvent_obtainAsCopy(env, motionEvent);
```

最终将经过 InputConsumer.comsume 填充的 InputEvent 事件转化为上层 Java 对象。

### 10.4 调用 Java 层对象 receiverObj 的 dispatchInputEvent 方法

```
env->CallVoidMethod(receiverObj.get(),
                        gInputEventReceiverClassInfo.dispatchInputEvent, seq, inputEventObj);
```

最终 receiverObj 是上层 InputEventReceiver 对象，inputEventObj 也已经转化为上层对应的输入事件，那么调用 InputEventReceiver.dispatchInputEvent 将输入事件传递给上层。

11 InputEventReceiver.dispatchInputEvent
----------------------------------------

```
// Called from native code.
    @SuppressWarnings("unused")
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
    private void dispatchInputEvent(int seq, InputEvent event) {
        mSeqMap.put(event.getSequenceNumber(), seq);
        onInputEvent(event);
    }
```

因为我们分析的是 ViewRootImpl#setView 中的监听输入事件注册流程，因此这里的 InputEventReceiver 实际上是一个 WindowInputEventReceiver 对象。

12 ViewRootImpl.WindowInputEventReceiver.onInputEvent
-----------------------------------------------------

```
final class WindowInputEventReceiver extends InputEventReceiver {
        public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
            super(inputChannel, looper);
        }

        @Override
        public void onInputEvent(InputEvent event) {
            /// M: record current key event and motion event to dump input event info for
            /// ANR analysis. {
            ViewDebugManager.getInstance().debugInputEventStart(event);
            /// }
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "processInputEventForCompatibility");
            List<InputEvent> processedEvents;
            try {
                processedEvents =
                    mInputCompatProcessor.processInputEventForCompatibility(event);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
            if (processedEvents != null) {
                if (processedEvents.isEmpty()) {
                    // InputEvent consumed by mInputCompatProcessor
                    finishInputEvent(event, true);
                } else {
                    for (int i = 0; i < processedEvents.size(); i++) {
                        enqueueInputEvent(
                                processedEvents.get(i), this,
                                QueuedInputEvent.FLAG_MODIFIED_FOR_COMPATIBILITY, true);
                    }
                }
            } else {
                enqueueInputEvent(event, this, 0, true);
            }
        }

        ......
    }
    private WindowInputEventReceiver mInputEventReceiver;
```

后续会调用 ViewRootImpl#enqueueInputEvent 进行一系列处理。

13 总结
-----

1)、InputDispatcher 分发线程首先根据事件类型，寻找所有可以接收当前输入事件的窗口，构建一个 InputTarget 队列。

2)、遍历 InputTarget 队列，队列中的所有窗口都需要处理本次输入事件。

3)、将输入事件信息封装成 InputMessage 对象，服务端 InputChannel 调用 socket 的 send 函数将 InputMessage 写入服务端 socket 的发送缓冲区。

4)、服务端 InputChannel 发送事件后，客户端这边注册的 NativeInputEventReceiver 的 Looper 监听到客户端 socket 的接收缓冲区有数据写入，回调 NativeInputEventReceiver.handleEvent 函数，收集服务端传过来的 InputMessage 信息，最终将输入事件封装为 Java 层的 KeyEvent 或者 MotionEvent 等类型，然后调用 Java 层 InputEventReceiver#DispatchInputEvent 完成输入事件从 Native 层到 Java 上层的传递。
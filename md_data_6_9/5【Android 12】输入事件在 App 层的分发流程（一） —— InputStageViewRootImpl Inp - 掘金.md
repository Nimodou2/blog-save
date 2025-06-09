> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7083466092995149855)

> ViewRootImpl InputStage dispatchKeyEvent requestFocus mFocused

之前在分析 InputDispatcher 分发的时候，知道输入事件最终从 Native 层传到了 framework 上层，到达了 ViewRootImpl 通过 setView 方法注册的 WindowInputEventReceiver 的 onInputEvent 方法。 接下来分析输入事件是如何在 App 层传递的。

由于字数限制，本篇笔记分 3 部分去分析。

```
        @Override
        public void onInputEvent(InputEvent event) {
			......
                
            List<InputEvent> processedEvents;
            try {
                processedEvents =
                    mInputCompatProcessor.processInputEventForCompatibility(event);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
            if (processedEvents != null) {
				......
            } else {
                enqueueInputEvent(event, this, 0, true);
            }
        }


```

这里的 InputEventCompatProcessor.processInputEventCompatibility 方法主要是为 Android M 平台以下的 Motion 事件处理做一些兼容处理，对于 Android M 以上的平台直接返回 null，那么后续直接调用 ViewRootImpl.enqueueInputEvent。

```
    @UnsupportedAppUsage
    void enqueueInputEvent(InputEvent event,
            InputEventReceiver receiver, int flags, boolean processImmediately) {
        QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);

		......
            
        
        
        
        
        
        QueuedInputEvent last = mPendingInputEventTail;
        if (last == null) {
            mPendingInputEventHead = q;
            mPendingInputEventTail = q;
        } else {
            last.mNext = q;
            mPendingInputEventTail = q;
        }
        mPendingInputEventCount += 1;

        ......

        if (processImmediately) {
            doProcessInputEvents();
        } else {
            scheduleProcessInputEvents();
        }
    }


```

1)、通过 ViewRootImpl.obtainQueuedInputEvent 创建一个 QueuedInputEvent 对象，将 QueuedInputEvent 成员变量 mEvent 指向当前处理的输入事件。

2)、将上一步得到的 QueuedInputEvent 入队。类 QueuedInputEvent 实现了一个等待处理的输入事件队列，ViewRootImpl 的成员变量 mPendingInputEventHead 指向这个待处理队列的队首，mPendingInputEventTail 指向队尾，QueuedInputEvent 的成员变量 mNext 指向排在当前等待处理的事件之后的下一个事件 QueuedInputEvent。

如果此时队伍中没有正在排队的事件，那么 mPendingInputEventHead 和 mPendingInputEventTail 都指向这个 QueuedInputEvent 对象。

如果此时队伍中有正在排队的事件，那么将队尾 mPendingInputEventTail 的成员变量 mNext 指向这个 QueuedInputEvent 对象，然后这个 QueuedInputEvent 对象变为队尾 mPendingInputEventTail。

3)、在第 1 节调用 ViewRootImpl.enqueueInputEvent 的时候传入的 processImmediately 为 true，那么调用 doProcessInputEvents 跳过调度直接处理。

```
    void doProcessInputEvents() {
        
        while (mPendingInputEventHead != null) {
            QueuedInputEvent q = mPendingInputEventHead;
            mPendingInputEventHead = q.mNext;
            if (mPendingInputEventHead == null) {
                mPendingInputEventTail = null;
            }
            q.mNext = null;

            mPendingInputEventCount -= 1;

            ......

            deliverInputEvent(q);
        }

        
        
        if (mProcessInputEventsScheduled) {
            mProcessInputEventsScheduled = false;
            mHandler.removeMessages(MSG_PROCESS_INPUT_EVENTS);
        }
    }


```

1)、遍历待处理队列，对每一个 QueuedInputEvent 对象调用 ViewRootImpl.deliverInputEvent 方法进行处理。

2)、遍历结束后，我们已经处理完了所有当前我们可以处理的输入事件，因此我们可以清除掉 mProcessInputEventsScheduled 这个待处理标记。这个标记只有在 ViewRootImpl.scheduleProcessInputEvents 方法中才会被置为 true，而上一步我们是检测到参数 processImmediately 为 true 直接调用了当前的 ViewRootImpl.doProcessInputEvents 方法，没有走 ViewRootImpl.scheduleProcessInputEvents。

重点分析对每一个 QueuedInputEvent 对象都调用的 deliverInputEvent 方法。

```
    private void deliverInputEvent(QueuedInputEvent q) {
		......
        try {
            if (mInputEventConsistencyVerifier != null) {
                Trace.traceBegin(Trace.TRACE_TAG_VIEW, "verifyEventConsistency");
                try {
                    mInputEventConsistencyVerifier.onInputEvent(q.mEvent, 0);
                } finally {
                    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
                }
            }

            InputStage stage;
            if (q.shouldSendToSynthesizer()) {
                stage = mSyntheticInputStage;
            } else {
                stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
            }
            
			......

            if (stage != null) {
                ......
                stage.deliver(q);
            } else {
                finishInputEvent(q);
            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }


```

整个方法需要分成几部分来看。

4.1 InputEventConsistencyVerifier
---------------------------------

```
            if (mInputEventConsistencyVerifier != null) {
                Trace.traceBegin(Trace.TRACE_TAG_VIEW, "verifyEventConsistency");
                try {
                    mInputEventConsistencyVerifier.onInputEvent(q.mEvent, 0);
                } finally {
                    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
                }
            }


```

InputEventConsistencyVerifier 用来判断属于同一系列的输入事件的一致性，并且收集每一个检测出的错误，同时避免相同错误重复收集。

4.2 第一个 InputStage 的选取
----------------------

```
            InputStage stage;
            if (q.shouldSendToSynthesizer()) {
                stage = mSyntheticInputStage;
            } else {
                stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
            }


```

这里先不去分析 InputStage 是用来干啥的，先看下这里创建的局部变量 stage 最终指向了哪个 InputStage。

这里 QuquedInputEvent 的两个方法都检测了当前 QueuedInputEvent 的 flag：

```
        public boolean shouldSkipIme() {
            if ((mFlags & FLAG_DELIVER_POST_IME) != 0) {
                return true;
            }
            return mEvent instanceof MotionEvent
                    && (mEvent.isFromSource(InputDevice.SOURCE_CLASS_POINTER)
                        || mEvent.isFromSource(InputDevice.SOURCE_ROTARY_ENCODER));
        }

        public boolean shouldSendToSynthesizer() {
            if ((mFlags & FLAG_UNHANDLED) != 0) {
                return true;
            }

            return false;
        }


```

回顾在 ViewRootImpl.enqueueInputEvent 方法中创建 QueuedInputEvent 的时候，传入的 lfags 参数是 0，因此这里对于所有 flag 的判断都会返回 false。那么 ViewRootImpl.shouldSendToSynthesizer 方法返回 false，而 ViewRootImpl.shouldSkipIme 方法会进一步判断当前是否是 MotionEvent 以及 MotionEvent 的输入源。

InputDevice 在 InputReader 调用 processEventsLocked 函数处理从 EventHub 读取的原始数据的时候也遇到过，InputDevice 描述了一个特定的输入设备的功能。

SOURCE_CLASS_POINTER 代表了事件类型是触摸点，这些事件的输入源包括触摸屏、鼠标和手写笔：

```
    
    public static final int SOURCE_TOUCHSCREEN = 0x00001000 | SOURCE_CLASS_POINTER;

    
    public static final int SOURCE_MOUSE = 0x00002000 | SOURCE_CLASS_POINTER;

    
    public static final int SOURCE_STYLUS = 0x00004000 | SOURCE_CLASS_POINTER;


```

SOURCE_ROTARY_ENCODER 代表了旋转编码设备，遇到的比较少：

```
    
    public static final int SOURCE_ROTARY_ENCODER = 0x00400000 | SOURCE_CLASS_NONE;


```

那么总结一下，如果当前输入事件是 Motion 类型且输入源是触摸点相关类型，或者输入源是旋转解码器类型，那么第一个 InputStage 选择 mFirstPostImeInputStage，否则选择 mFirstInputStage。

4.3 InputStage 介绍
-----------------

```
            if (stage != null) {
                ......
                stage.deliver(q);
            } else {
                finishInputEvent(q);
            }


```

根据上一小节的分析，那么这里的 stage 可能是 mFirstPostImeInputStage 或 mFirstInputStage。

再进一步分析前，需要看一下 InputStage 是干嘛的。

```
    
    abstract class InputStage {
        private final InputStage mNext;

        protected static final int FORWARD = 0;
        protected static final int FINISH_HANDLED = 1;
        protected static final int FINISH_NOT_HANDLED = 2;

        private String mTracePrefix;

        
        public InputStage(InputStage next) {
            mNext = next;
        }

        
        public final void deliver(QueuedInputEvent q) {
            if ((q.mFlags & QueuedInputEvent.FLAG_FINISHED) != 0) {
                forward(q);
            } else if (shouldDropInputEvent(q)) {
                finish(q, false);
            } else {
                traceEvent(q, Trace.TRACE_TAG_VIEW);
                final int result;
                try {
                    result = onProcess(q);
                } finally {
                    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
                }
                apply(q, result);
            }
        }

        
        protected void finish(QueuedInputEvent q, boolean handled) {
            q.mFlags |= QueuedInputEvent.FLAG_FINISHED;
            if (handled) {
                q.mFlags |= QueuedInputEvent.FLAG_FINISHED_HANDLED;
            }
            forward(q);
        }

        
        protected void forward(QueuedInputEvent q) {
            onDeliverToNext(q);
        }

        
        protected void apply(QueuedInputEvent q, int result) {
            if (result == FORWARD) {
                forward(q);
            } else if (result == FINISH_HANDLED) {
                finish(q, true);
            } else if (result == FINISH_NOT_HANDLED) {
                finish(q, false);
            } else {
                throw new IllegalArgumentException("Invalid result: " + result);
            }
        }

        
        protected int onProcess(QueuedInputEvent q) {
            return FORWARD;
        }

        
        protected void onDeliverToNext(QueuedInputEvent q) {
            if (DEBUG_INPUT_STAGES) {
                Log.v(mTag, "Done with " + getClass().getSimpleName() + ". " + q);
            }
            if (mNext != null) {
                mNext.deliver(q);
            } else {
                finishInputEvent(q);
            }
        }

        protected void onWindowFocusChanged(boolean hasWindowFocus) {
            if (mNext != null) {
                mNext.onWindowFocusChanged(hasWindowFocus);
            }
        }

		......
    }


```

InputStage 是用于实现处理输入事件的责任链中的一个阶段的基类。事件通过 InputStage.deliver 方法发送给 stage，接下来 stage 有权决定结束掉这个事件或者把它转交给下一个 stage。

根据 InputStage 的提供的方法接口，可以总结出输入事件在 InputStage 链中传递的一般规律：

1）、上一个 InputStage 调用 InputStage.onDeliverToNext -> InputStage.deliver 将输入事件发送到当前 InputStage。

2.1）、如果输入事件被标记了 QueuedInputEvent.FLAG_FINISHED，那么调用 InputStage.forward 继续将事件向下一个 InputStage 分发，当前 InputStage 不做处理。

2.2）、如果输入事件经过 InputStage.shouldDropInputEvent 判断应该被丢弃，那么调用 InputStage.finish 为输入事件添加 QueuedInputEvent.FLAG_FINISHED 标记，InputStage.finish 中又调用 InputStage.forward 把事件分发给下一个 InputStage。

2.3）、如果 2.1 和 2.2 的判断条件不满足，说明本次事件需要当前 InputStage 进行处理，那么调用 InputStage.onProcess 对事件进行处理，这是每一个 InputStage 处理事件的核心部分。

3）、根据 InputStage.onProcess 的处理结果，调用 InputStage.apply 方法判断是将事件在当前 InputStage 结束掉，还是调用 InputStage.forward 继续将事件分发给下一个 InputStage。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/290e348289be407cbfb5e94593c600ec~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 4.3.1 InputStage 责任链

```
    abstract class InputStage {
        private final InputStage mNext;
		......

        
        public InputStage(InputStage next) {
            mNext = next;
        }

		......
    }


```

首先能看到 InputStage 有一个 InputStage 类型的 mNext 成员变量，在 InputStage 创建的时候传入，指向当前 InputStage 的下一个 InputStage，以此构成一个链表形式。

所有 InputStage 创建的地方在 ViewRootImpl.setView 中，也就是在从 system_server 进程返回客户端 InputChannel 之后。

```
    
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
        synchronized (this) {
            if (mView == null) {
			   ......	

                
                CharSequence counterSuffix = attrs.getTitle();
                mSyntheticInputStage = new SyntheticInputStage();
                InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
                InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                        "aq:native-post-ime:" + counterSuffix);
                InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
                InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                        "aq:ime:" + counterSuffix);
                InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
                InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                        "aq:native-pre-ime:" + counterSuffix);

                mFirstInputStage = nativePreImeStage;
                mFirstPostImeInputStage = earlyPostImeStage;
                mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
            }
        }
    }


```

因此 InputStage 责任链的处理顺序是：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e33d3b63d72c4cf8a47c67dff22e27da~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

根据 4.2 的分析可知，MotionEvent 是直接从 EarlyPostImeInputStage 这一阶段开始分发的。

从名字可以清晰的将 InputStage 分为三个阶段：IME 处理前阶段，IME 处理阶段，IME 处理后阶段。

### 4.3.2 InputStage 各阶段介绍

对这 7 个 InputStage 进行简介。

#### 4.3.2.1 IME 处理前阶段 - NativePreImeInputStage

```
    
    final class NativePreImeInputStage extends AsyncInputStage
            implements InputQueue.FinishedInputEventCallback {
            ......
     }


```

在 IME 处理之前，将输入事件发送给一个 native Activity。不支持点触事件。

在此阶段只支持将 KeyEvent 类型的时间提前发送给 native Activity，而且是异步进行的，也就是说，这里对输入事件的处理可能会推迟。

#### 4.3.2.2 IME 处理前阶段 - ViewPreImeInputStage

```
    
    final class ViewPreImeInputStage extends InputStage {
   	 	......
    }


```

在 IME 处理之间，将输入事件发送给 View 层级结构。不支持点触事件。

主要是针对 KeyEvent 类型的事件，在将 KeyEvent 发送给 IME 并由 IME 消费掉之前，使用 View.dispatchKeyEventPreIme 和 View.onKeyPreIme 尝试进行一次拦截，主要是为了一些特殊情况，比如当 BACK 按键事件分发的时候，我们更希望 View 层级结构能够处理这个 BACK 按键事件来更新 App 的 UI，而不是让 IME 接收到 BACK 键然后关闭掉 IME 窗口。

#### 4.3.2.3 IME 处理阶段 - ImeInputStage

```
    
    final class ImeInputStage extends AsyncInputStage
            implements InputMethodManager.FinishedInputEventCallback {
   		......
    }


```

将输入事件分发给输入法。不支持点触事件。

调用 InputMethodManager.dispatchInputEvent 对输入事件进行分发，之前处理过功能机按键的相关问题，比如向编辑框中输入信息的时候，数字按键事件 KEYCODE_0、KEYCODE_1 等会在这个阶段分发给 InputMethodManager 处理，不会发送给后续的 InputStage。。

#### 4.3.2.4 IME 处理后阶段 - EarlyPostImeInputStage

```
    
    final class EarlyPostImeInputStage extends InputStage {
        ......
    }


```

在 IME 处理之后，对输入事件进行早期处理。

主要是为了在输入事件进一步发送之前，判断当前输入事件是否会影响 touch mode 的进入和退出，以及，让 AudioManager 能够提前接收到输入事件等。

#### 4.3.2.5 IME 处理后阶段 - NativePostImeInputStage

```
    
    final class NativePostImeInputStage extends AsyncInputStage
            implements InputQueue.FinishedInputEventCallback {
        ......
    }
            


```

在 IME 处理阶段之后，将输入事件发送给 native Activity。

这一步和 4.3.2.1 中 NativePreImeInputStage 的处理很像，不同的地方在于，NativePreImeInputStage 只允许发送 KeyEvent 事件，而到了 NativePostImeInputStage 这里就不再限制输入事件的类型。在此阶段，native Activity 仍然能比 Java 层的 Activity 进一步处理输入事件。

#### 4.3.2.6 IME 处理后阶段 - ViewPostImeInputStage

```
    
    final class ViewPostImeInputStage extends InputStage {
        ......
    }


```

在 IME 处理阶段之后，将输入事件发送给 View 层级结构。

此阶段是输入事件发送给 Activity 和 View 的地方，后面重点分析。

#### 4.3.2.7 IME 处理后阶段 - SyntheticInputStage

```
    
    final class SyntheticInputStage extends InputStage {
        ......
    }


```

从未处理的输入事件中合成新的输入事件。

此阶段是责任链的最后一个阶段，主要用来处理前几个阶段无法处理的输入事件类型，如轨迹球，游戏摇杆，导航面板等。

```
        @Override
        protected int onProcess(QueuedInputEvent q) {
            if (q.mEvent instanceof KeyEvent) {
                return processKeyEvent(q);
            } else {
                final int source = q.mEvent.getSource();
                if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
                    return processPointerEvent(q);
                } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
                    return processTrackballEvent(q);
                } else {
                    return processGenericMotionEvent(q);
                }
            }
        }


```

根据输入事件类型进行相应处理。

我们的重点分析处理处理点触事件类型的 processPointerEvent 和 KeyEvent 类型事件的 processKeyEvent 方法。
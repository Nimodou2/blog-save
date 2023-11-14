> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/126759808?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/a15355b62c3a4d6495324326f9a18136.jpeg#pic_center)

6 MotionEvent 处理流程
------------------

### 6.1 ViewRootImpl.ViewPostImeInputStage.processPointerEvent

```
private int processPointerEvent(QueuedInputEvent q) {
            final MotionEvent event = (MotionEvent)q.mEvent;

			......
            boolean handled = mView.dispatchPointerEvent(event);

			......
            return handled ? FINISH_HANDLED : FORWARD;
        }
```

这里先将 QueuedInputEvent 中的 mEvent 向下转为 MotionEvent 类型，但从 MotionEvent 的注释来看，MotionEvent 是用来报告移动（鼠标，笔，手指，轨迹球）事件的对象，覆盖的输入源似乎比 pointer 类型多。

这里的 mView 是 View 层级结构的根 VIew，对于 Activity 来说就是 DecorView，对于非 Activity 窗口来说就是该窗口的[自定义 View](https://so.csdn.net/so/search?q=%E8%87%AA%E5%AE%9A%E4%B9%89View&spm=1001.2101.3001.7020)，这里只分析最常见的 Activity 窗口。由于 View.dispatchPointerEvent 声明为 final，所以这里直接调用的是 View.dispatchPointerEvent 方法。

### 6.2 View.dispatchPointerEvent

```
/**
     * Dispatch a pointer event.
     * <p>
     * Dispatches touch related pointer events to {@link #onTouchEvent(MotionEvent)} and all
     * other events to {@link #onGenericMotionEvent(MotionEvent)}.  This separation of concerns
     * reinforces the invariant that {@link #onTouchEvent(MotionEvent)} is really about touches
     * and should not be expected to handle other pointing device features.
     * </p>
     *
     * @param event The motion event to be dispatched.
     * @return True if the event was handled by the view, false otherwise.
     * @hide
     */
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
    public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }
```

这个方法用来分发点触事件。

分发 touch 相关的点触事件到 View.onTouchEvent，其他所有的事件发送给 View.onGenericMotionEvent。这种将 touch 事件和其他点触事件分开处理的做法，强调了 MotionEvent 就是和 touch 相关的，不应该再去处理其他点触事件。

我们的关注点在 touch 相关的事件分发流程，那么这里调用的是 DecorView.dispatchTouchEvent 方法。

### 6.3 DecorView.dispatchTouchEvent

```
@Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        final Window.Callback cb = mWindow.getCallback();
        return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
    }
```

这里的 mWIndow 即 PhoneWindow，PhoneWIndow 的 callback 是 Activity.attach 的时候通过 PhoneWindow.setCallback 设置的，传入的是 Activity 自己。

我们分析一般流程，即 Activity 对应的 DecorView 的事件分发流程。

### 6.4 Activity.dispatchTouchEvent

```
/**
     * Called to process touch screen events.  You can override this to
     * intercept all touch screen events before they are dispatched to the
     * window.  Be sure to call this implementation for touch screen events
     * that should be handled normally.
     *
     * @param ev The touch screen event.
     *
     * @return boolean Return true if this event was consumed.
     */
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

这个方法用来处理触屏事件。你可以复写这个方法从而在所有触屏事件发送给窗口之前将其拦截。

1）、当 down 流程的时候触发 onUserInteraction 回调。如果你想知道当你的 Activity 正在运行时，用户已经以某种方式与设备进行了交互，那么就执行这个方法。onUserInteraction 和 onUserLeaveHint 用来帮助 Activity 智能管理状态栏通知，特别是，帮助 Activity 确定一个合适的时机来取消掉通知。

2）、调用 PhoneWindow.superDispatchTouchEvent 方法，将输入事件发送给 View 层级结构，分析重点。

3）、如果前面两步没有处理 MotionEvent，那么调用 Activity.onTouchEvent 来处理。这个方法最有用的地方是用来处理那些落在窗口之外，没有任何 View 可以接收的 touch 事件。比如当触摸到 Dialog 以外的区域的时候，判断是否要移除掉 Dialog。

### 6.5 PhoneWindow.superDispatchTouchEvent

```
@Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```

这里的 mDecor 是 DeocrView 类型的成员变量。

### 6.6 DecorView.superDispatchTouchEvent

```
public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
```

一直念叨着 View 层级结构，直到这里才算真正走到 View 层级结构的根 View 了。

由于 DecorView 的直接父类 [FrameLayout](https://so.csdn.net/so/search?q=FrameLayout&spm=1001.2101.3001.7020) 没有重写 dispatchTouchEvent 方法，因此这里调用的是 ViewGroup.dispatchTouchEvent。

### 6.7 ViewGroup.dispatchTouchEvent

```
@Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
		......

        // If the event targets the accessibility focused view and this is it, start
        // normal event dispatch. Maybe a descendant is what will handle the click.
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }
      
		......
      
        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

            // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }

            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // Update list of touch targets for pointer down, if needed.
            final boolean isMouseEvent = ev.getSource() == InputDevice.SOURCE_MOUSE;
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0
                    && !isMouseEvent;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {
                // If the event is targeting accessibility focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x =
                                isMouseEvent ? ev.getXCursorPosition() : ev.getX(actionIndex);
                        final float y =
                                isMouseEvent ? ev.getYCursorPosition() : ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            if (!child.canReceivePointerEvents()
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (ViewDebugManager.DEBUG_MOTION) {
                                Log.d(TAG, "(ViewGroup)dispatchTouchEvent to child 3: child = "
                                        + child + ",childrenCount = " + childrenCount + ",i = " + i
                                        + ",newTouchTarget = " + newTouchTarget
                                        + ",idBitsToAssign = " + idBitsToAssign);
                            }
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
					
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

		......
        return handled;
    }
```

这个方法太长了，分几部分去分析。

#### 6.7.1 accessibility focus

```
// If the event targets the accessibility focused view and this is it, start
        // normal event dispatch. Maybe a descendant is what will handle the click.
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }
```

这里的 accessibility 应该是无障碍辅助功能相关的内容。ViewRootImpl 可以通过 setAccessibilityFocus 方法来设置一个 View 作为 accessibility focusd View，那么这个 View 就拥有了 accessibility focus，拥有 accessibility focus 的 View 在事件分发的时候会被特殊处理。

这里通过 MotionEvent.isTargetAccessibilityFocus 判断以个 MotionEvent 是否是一个以 accessibility focusd View 为目标的事件，根据当前事件的相关 flag 是否包含 FLAG_TARGET_ACCESSIBILITY_FOCUS：

```
/**
     * Private flag indicating that this event was synthesized by the system and should be delivered
     * to the accessibility focused view first. When being dispatched such an event is not handled
     * by predecessors of the accessibility focused view and after the event reaches that view the
     * flag is cleared and normal event dispatch is performed. This ensures that the platform can
     * click on any view that has accessibility focus which is semantically equivalent to asking the
     * view to perform a click accessibility action but more generic as views not implementing click
     * action correctly can still be activated.
     *
     * @hide
     * @see #isTargetAccessibilityFocus()
     * @see #setTargetAccessibilityFocus(boolean)
     */
    public static final int FLAG_TARGET_ACCESSIBILITY_FOCUS = 0x40000000;
```

FLAG_TARGET_ACCESSIBILITY_FOCUS 表示该事件是由系统合成的而且应该被首先发送给 accessibility focused View。在发送时，此类事件不会被 accessibility focused view 之前的 View 的处理，而且当事件到达 accessibility focused View 之后，相关标志会被清除，普通的事件分发会被执行。这个确保了平台可以点击任何有 accessibility focuse 的 View，这在语义上等同于要求 View 去执行一个点击辅助动作，但更通用的是，没有正确实现点击动作的 View 仍然可以被激活。

那这里的逻辑就符合了注释中的情况，此时当目标为 accessibility focused Vie 的事件到达 accessibility focused View 之后，FLAG_TARGET_ACCESSIBILITY_FOCUS 标志被清除。

#### 6.7.2 调用 onFilterTouchEventForSecurity 检查安全性

```
boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
        	......
        }

		......
        return handled;
```

调用 View.onFilterTouchEventForSecurity 方法进行安全方面的检查，如果 onFilterTouchEventForSecurity 返回 false，那么就直接返回 false，表示当前 ViewGroup 不处理此次 MotionEvent。

看下 View.onFilterTouchEventForSecurity 方法具体内容：

```
/**
     * Filter the touch event to apply security policies.
     *
     * @param event The motion event to be filtered.
     * @return True if the event should be dispatched, false if the event should be dropped.
     *
     * @see #getFilterTouchesWhenObscured
     */
    public boolean onFilterTouchEventForSecurity(MotionEvent event) {
        //noinspection RedundantIfStatement
        if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
                && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
            // Window is obscured, drop this touch.
            return false;
        }
        return true;
    }
```

这个方法用来基于安全因素过滤掉 touch 事件。

如果当前 View 的 mViewFlags 包含 FILTER_TOUCHES_WHEN_OBSCURED，并且当前 touch 事件包含 FLAG_WINDOW_IS_OBSCURED，那么就丢弃掉此次事件。

```
/**
     * Indicates that the view should filter touches when its window is obscured.
     * Refer to the class comments for more information about this security feature.
     * {@hide}
     */
    static final int FILTER_TOUCHES_WHEN_OBSCURED = 0x00000400;
```

FILTER_TOUCHES_WHEN_OBSCURED 表示如果它的窗口被遮挡，那么当前 View 需要过滤掉 touch 事件。

```
/**
     * This flag indicates that the window that received this motion event is partly
     * or wholly obscured by another visible window above it. This flag is set to true
     * if the event directly passed through the obscured area.
     *
     * A security sensitive application can check this flag to identify situations in which
     * a malicious application may have covered up part of its content for the purpose
     * of misleading the user or hijacking touches.  An appropriate response might be
     * to drop the suspect touches or to take additional precautions to confirm the user's
     * actual intent.
     */
    public static final int FLAG_WINDOW_IS_OBSCURED = 0x1;
```

FLAG_WINDOW_IS_OBSCURED 表示当前正在接收 MotionEvent 的窗口被在它之上的另一个可见窗口部分或者完全遮挡。这个标记看代码应该是 InputDispatcher 在发送事件的时候会去设置：InputDispatcher 首先会为本次要发送的事件寻找一个目标窗口，而在将该事件发送给目标窗口之前，会额外再判断目标窗口之上是否有其他的可见窗口能够遮挡目标窗口，如果有，那么会把 FLAG_WINDOW_IS_OBSCURED 这个标记加到本次输入事件中。

一个对安全性敏感的 App 可以检查这个标志，以识别恶意应用程序可能为了误导用户或劫持触摸事件而掩盖其部分内容的情况。一个适当的响应可能是丢弃可疑的触摸事件，或者采取额外的预防措施来确认用户的实际意图。

#### 6.7.3 调用 onInterceptTouchEvent 尝试拦截当前事件

```
// Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

			......
                
            if (!canceled && !intercepted) {
                .....
            }
```

如果判断当前 ViewGroup 需要拦截本次事件，那么 intercepted 会被置为 true，那么就不会再去走相关逻辑：

```
if (!canceled && !intercepted) {
                .....
            }
```

if 语句里会继续将事件向子 View 发送。

接下来分几部分分析拦截事件的具体内容。

##### 6.7.3.1 拦截事件的两种情况

```
if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                    ......
                        intercepted = onInterceptTouchEvent(ev);
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }
```

首先进行两项判断：

1）、当前事件动作类型是否是 MotionEvent.ACTION_DOWN。

或

2）、mFirstTouchTarget 是否不为空。mFirstTouchTarget 的含义需要结合后面的分析才能了解，mFirstTouchTarget 不为空，说明在本次 gesture 之前的事件分发中，我们找到了可以接收事件的目标子 View。

拦截事件的情况之一是，如果两项条件满足其一，那么后面会调用 ViewGroup.onInterceptTouchEvent 来尝试拦截本次事件。如果事件被当前 ViewGroup 拦截，intercepted 会被置为 true，那么事件将不会发给 ViewGroup 的子 View。

拦截事件的另一种情况，如果这两项都不满足，说明本次事件的行为不是 ACTION_DOWN，且在 ACTION_DOWN 流程中也没有找到一个目标子 View 可以接收 ACTION_DOWN 事件，那么说明当前 ViewGroup 中的所有子 View 都没有满足接收本次事件的条件，直接将 intercepted 置为 true，表示本次事件将会被当前 VIewGroup 拦截，直接走 6.7.5。这里说明了，必须有子 View 能够接收 ACTION_DOWN 事件，这样接下来的事件，不管是 ACTION_POINTER_DOWN 还是 ACTION_MOVE 之类的，才能继续发送给子 VIew，否则当前 ViewGroup 会消费此次事件，不再发送给子 View。

另外从这里的判断也能看出，拦截事件是在所有事件行为流程中都可能会发生，不只 ACTION_DOWN。

这部分需要结合整体才能更好理解。

##### 6.7.3.2 FLAG_DISALLOW_INTERCEPT

```
final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
```

对于一般流程来说，手势的起点都是 MotionEvent.ACTION_DOWN，因此接下来继续判断，当前 ViewGroup 的 mGroupFlags 是否包含 FLAG_DISALLOW_INTERCEPT 这个标志。

FLAG_DISALLOW_INTERCEPT 标志的含义是：

```
/**
     * When set, this ViewGroup should not intercept touch events.
     * {@hide}
     */
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P, trackingBug = 123983692)
    protected static final int FLAG_DISALLOW_INTERCEPT = 0x80000;
```

如果设置了这个标志，那么当前 ViewGroup 不应该拦截 touch 事件。

子 View 可以通过调用 ViewGroup.requestDisallowInterceptTouchEvent 方法，来阻止其父 ViewGroup（不止是其直接父 ViewGroup）通过 ViewGroup#onInterceptTouchEvent 拦截 touch 事件。父 ViewGroup 会[递归调用](https://so.csdn.net/so/search?q=%E9%80%92%E5%BD%92%E8%B0%83%E7%94%A8&spm=1001.2101.3001.7020) ViewGroup.requestDisallowInterceptTouchEvent 方法，保证子 View 所在的 View 层级结构中的该子 View 的所有父 ViewGroup 都能应用到这个标志。

##### 6.7.3.3 ViewGroup.onInterceptTouchEvent

```
if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                }
```

如果 FLAG_DISALLOW_INTERCEPT 标志没有被子 View 请求，那么调用 ViewGroup.onInterceptTouchEvent 尝试拦截当前事件。

```
/**
     * Implement this method to intercept all touch screen motion events.  This
     * allows you to watch events as they are dispatched to your children, and
     * take ownership of the current gesture at any point.
     *
     * <p>Using this function takes some care, as it has a fairly complicated
     * interaction with {@link View#onTouchEvent(MotionEvent)
     * View.onTouchEvent(MotionEvent)}, and using it requires implementing
     * that method as well as this one in the correct way.  Events will be
     * received in the following order:
     *
     * <ol>
     * <li> You will receive the down event here.
     * <li> The down event will be handled either by a child of this view
     * group, or given to your own onTouchEvent() method to handle; this means
     * you should implement onTouchEvent() to return true, so you will
     * continue to see the rest of the gesture (instead of looking for
     * a parent view to handle it).  Also, by returning true from
     * onTouchEvent(), you will not receive any following
     * events in onInterceptTouchEvent() and all touch processing must
     * happen in onTouchEvent() like normal.
     * <li> For as long as you return false from this function, each following
     * event (up to and including the final up) will be delivered first here
     * and then to the target's onTouchEvent().
     * <li> If you return true from here, you will not receive any
     * following events: the target view will receive the same event but
     * with the action {@link MotionEvent#ACTION_CANCEL}, and all further
     * events will be delivered to your onTouchEvent() method and no longer
     * appear here.
     * </ol>
     *
     * @param ev The motion event being dispatched down the hierarchy.
     * @return Return true to steal motion events from the children and have
     * them dispatched to this ViewGroup through onTouchEvent().
     * The current target will receive an ACTION_CANCEL event, and no further
     * messages will be delivered here.
     */
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
                && ev.getAction() == MotionEvent.ACTION_DOWN
                && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
                && isOnScrollbarThumb(ev.getX(), ev.getY())) {
            return true;
        }
        return false;
    }
```

渣翻：

实现此方法来拦截所有触屏移动事件。这允许你在事件被发送给子 View 的时候监控这些事件，并且随时取得当前 gesture 的控制权。

使用方法的时候要注意，因为它和 View.onTouchEvent 有一个相当复杂的交互，而且使用 View.onTouchEvent 方法需要以正确的方式实现它，ViiewGroup.onInterceptTouchEvent 也是一样。事件将会按照以下顺序被接收：

1）、你将会在这里接收到 down 事件。

2）、这个 down 事件要么将会被当前 ViewGroup 的一个子 View 处理，要么将会被发送给当前 ViewGroup 的 onTouchEvent 去处理。这意味着你需要应该实现 onTouchEvent 方法并且返回 true，这样你才能继续接收到当前 gesture 的余下事件（而不是寻找一个父 View 来处理当前事件）。而且，通过在 onTouchEvent 方法中返回 true，你将不会在 onInterceptTouchEvent 中收到接下来的事件，而且所有的 touch 事件的处理将会像通常一样在 onTouchEvent 中进行。

3）、只要你从这个方法中返回了 false，那么接下来的每一个事件（直到并且包括最终的 ACTION_UP），将会被第一时间发送到这里，然后才会被发给 target 的 onTouchEvent 方法。

4）、如果你在这里返回 true，你将不会接收到任何接下来的事件：target View 将会接收到同样的事件，但是附上了 MotionEvent.ACTION_CANCEL，而且以后的事件都将会发送给你的 onTouchEvent 方法，不会再在这里出现。

#### 6.7.4 DOWN 行为事件的发送

```
// If the event is targeting accessibility focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x =
                                isMouseEvent ? ev.getXCursorPosition() : ev.getX(actionIndex);
                        final float y =
                                isMouseEvent ? ev.getYCursorPosition() : ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            if (!child.canReceivePointerEvents()
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
```

##### 6.7.4.1 寻找包含 accessibility focus View 的子 View

```
if (!canceled && !intercepted) {
                // If the event is targeting accessibility focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    .....
                }
            }
```

如果当前事件没有被 cancel，也没有被 6.7.3 拦截，那么开始进行事件的分发。

首先，如果当前事件是一个以 accessibility focus View 为目标的事件，那么尝试从当前 ViewGroup 中找到那个包含了 accessibility focus View 的子 View。因为我们把 accessibility focus View 的优先级放的比较高，所以我们希望找到这个 View 并且第一时间把事件传给它，让它先处理事件。如果它不能处理当前事件，那么我们就清除掉 MotionEvent 中有关 accessibility focus 的相关标志，然后再像往常一样向子 View 发送事件。因为这类和 accessibility focus 有关事件比较稀有，因此我们先寻找 accessibility focus View 来避免一直持有 accessibility focus 相关的状态。

```
/**
     * Finds the child which has accessibility focus.
     *
     * @return The child that has focus.
     */
    private View findChildWithAccessibilityFocus() {
        ViewRootImpl viewRoot = getViewRootImpl();
        if (viewRoot == null) {
            return null;
        }

        View current = viewRoot.getAccessibilityFocusedHost();
        if (current == null) {
            return null;
        }

        ViewParent parent = current.getParent();
        while (parent instanceof View) {
            if (parent == this) {
                return current;
            }
            current = (View) parent;
            parent = current.getParent();
        }

        return null;
    }
```

从 findChildWithAccessibilityFocus 的实现来看，这个返回的 VIew 可能不是 accessibility focus View，而是包含 accessibility focus View 的父 View。

_其实这部分逻辑不只在 DOWN 流程中才执行，但是 childWithAccessibilityFocus 只在 DOWN 流程中才用到，那么是不是放在 DOWN 流程中比较合适？_

##### 6.7.4.2 遍历所有子 View 前的准备

```
if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x =
                                isMouseEvent ? ev.getXCursorPosition() : ev.getX(actionIndex);
                        final float y =
                                isMouseEvent ? ev.getYCursorPosition() : ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            ......
                        }
                    }
                }
```

1）、首先 6.7.4 的标题并不准确，这里要处理的事件包括三个：MotionEvent.ACTION_DOWN，可以将 MotionEvent 拆分发送给多个子 View 的 MotionEvent.ACTION_POINTER_DOWN（这个涉及到多点触摸，当第一根手指按下时，会上报 MotionEvent.ACTION_DOWN，后续如果有另外的手指按下，上报的就是 MotionEvent.ACTION_POINTER_DOWN）和指针移动事件 MotionEvent.ACTION_HOVER_MOVE（这个类型的事件就不讨论了）。

2）、这里的 idBitsToAssign 对应的多点触控相关的内容，通过 MotionEvent.getPointerId 可以返回每一根手指对应的唯一独特 ID。

3）、调用 View.buildTouchDispatchChildList 提前构建一个有序的子 View 队列，该队列根据 Z 轴顺序和 View 的绘制顺序排列，以 Z 轴顺序优先，Z 轴上的位置越高，在队列中就越靠近队尾。

##### 6.7.4.3 遍历所有子 View，将事件发送给子 View

```
for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            if (!child.canReceivePointerEvents()
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
							......
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
```

分成几部分来看。

###### 6.7.4.3.1 优先处理 accessibility focus View

```
// If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }
```

这里我们看到，首先直接遍历所有子 View 找到包含了 accessibility focus View 的子 View，让这个子 View 优先处理这个事件。

如果这个子 View 处理不了当前事件，那么我们再执行一遍正常的分发，那么我们就有可能做了两次遍历。

###### 6.7.4.3.2 过滤掉无效子 View

```
if (!child.canReceivePointerEvents()
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }
```

1）、View.canReceivePointerEvents 表示当前 View 是否可以接收点触事件，能够接收的条件是，当前子 View 可见，或者不可见但是在执行一个动画。如果这两个条件都不满足，那么认为当前 View 无法接收输入事件。

2）、ViewGroup.isTransformedTouchPointInView 用来判断当前事件的坐标是否落在这个 ViewGroup 的范围之内，如果没有自然也不要向这个 View 发送本次事件。

###### 6.7.4.3.3 目标 View 的复用

```
newTouchTarget = getTouchTarget(child);
							......
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }
```

分析完 6.7.4.3.5 再回过来看这里，那么这里是说，如果在之前的事件分发（ACTION_DOWN 或 ACTION_POINTER_DOWN）中，已经找到了一个子 View 可以接收之前的事件，并且将这个子 View 封装为一个 TouchTarget 对象了，那么对于本次事件分发（ACTION_POINTER_DOWN），直接将当前事件的 pointerId 也加入到这个 TouchTarget 的 pointerIdBits 中。接着 break 跳出当前对所有子 View 的遍历，这样，就不用再去挨个判断哪个子 View 满足接收当前事件的相关条件了，直接复用上一次的 ACTION_DOWN 流程或者是 ACTION_POINTER_DOWN 流程找到的 TouchTarget。我老是忍不住拿这个 TouchTarget 和 InputDispatcher 分发事件的时候用到的 TouchState 做比较。InputDIspatcher 也是用了 TouchState 用来保存所有可以接收输入事件的窗口，这两者的作用很相似，都是用来减少重复的搜寻目标窗口或者目标 View 的工作。毕竟如果在上一次事件发送的时候，如果我们已经找到了一个可以接收事件的目标 View，那么后续事件发送的时候，直接复用这个目标 View，那会减少很多重复的工作。

这种情形应该也是和多点触控相关的，毕竟每个 gesture 的第一次 ACTION_DOWN 流程中，都会重置 mFirstTouchTarget，因此如果是单点触控，那么走到这里 mFirstTouchTarget 应该是为空的，调用 getTouchTarget 也只会返回 NULL。

另外 6.7.4.3.2 也保证了执行到这里的时候，当前事件的坐标是落在这个可以复用的子 View 区域中的，不然不加限制，让之前事件分发流程中找到的目标 View 可以无脑接收到接下来所有的 ACTION_POINTER_DOWN，这个逻辑就有点奇怪了。

###### 6.7.4.3.4 调用 dispatchTransformedTouchEvent 往子 View 分发事件

```
if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
							......
                            }
```

这里调用了 ViewGroup.dispatchTransformedTouchEvent 继续往子 View 分发事件。

```
/**
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        // Canceling motions is a special case.  We don't need to perform any transformations
        // or filtering.  The important part is the action, not the contents.
        final int oldAction = event.getAction();
		......

        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        // Calculate the number of pointers to deliver.
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        // If for some reason we ended up in an inconsistent state where it looks like we
        // might produce a motion event with no pointers in it, then drop the event.
        if (newPointerIdBits == 0) {
            Log.i(TAG, "Dispatch transformed touch event without pointers in " + this);
            return false;
        }

        // If the number of pointers is the same and we don't need to perform any fancy
        // irreversible transformations, then we can reuse the motion event for this
        // dispatch as long as we are careful to revert any changes we make.
        // Otherwise we need to make a copy.
        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);

                    handled = child.dispatchTouchEvent(event);

                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

        // Perform any necessary transformations and dispatch.
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

            handled = child.dispatchTouchEvent(transformedEvent);
        }

		......

        // Done.
        transformedEvent.recycle();
        return handled;
    }
```

这个方法虽然很长，但是并不复杂，核心思想是：

判断传入的子 View 是否为空，如果为空，那么调用当前基类 View 的 dispatchTouchEvent 方法，表示当前 ViewGroup 自己处理当前事件。否则，调用子 View 的 dispatchTouchEvent 方法将事件继续分发给子 View。在当前流程中，传入的子 View 都不为空。

如果要把事件传给子 View，那么需要额外判断：

1）、当前事件是否被标记为 cancel。cancel 流程下，我们不需要再将事件进行任何转换或者过滤，因为重要的点在于事件行为（即 ACTION_CANCEL），而不是事件内容（事件的坐标之类的）。

2）、需要在将事件发送给子 View 之前，将事件转换到子 View 的坐标系统中。如果当前事件的 pointerId 和我们希望的 pointerId 相同，那么我们不需要对事件执行任何特殊的不可逆转的转换，然后只要我们注意恢复我们对当前事件做出的任何修改，就可以复用当前事件。否则我们需要复制一份当前事件的拷贝防止事件转换影响到原事件。

关于这里的 pointerId 进一步的解释如下：

```
// Calculate the number of pointers to deliver.
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;
```

这里的传参 desiredPointerIdBits 代表的是坐标落在要接下来接收事件的子 View 的区域里的那些 pointer（在一次多点触摸中，多个手指很有可能分别落在不同的子 View 区域中的），oldPointerIdBits 代表的如果是当前 event 中包含的所有 pointer，那么 newPointerIdBits 代表的就是 oldPointerIdBits 经过 desiredPointerIdBits 过滤后，子 View 接收的那几个 pointer。

###### 6.7.4.3.5 将可以接收当前事件的子 View 封装为 TouchTarget

```
if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
```

如果在 6.7.4.3.4 中找到一个可以处理当前事件的 View，即 dispatchTransformedTouchEvent 返回了 true，那么进行以下处理：

1）、记录当前事件的按下时间，处理当前事件的子 View 在 ViewGroup 的 mChildren 数组中的索引，以及当前事件的 x，y 坐标。

2）、调用 ViewGroup.addTouchTarget 方法将处理当前事件的子 View 以及事件对应的 pointerId 打包封装为一个 TouchTarget 对象。

```
/**
     * Adds a touch target for specified child to the beginning of the list.
     * Assumes the target child is not already present.
     */
    private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
        final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
        if (ViewDebugManager.DEBUG_MOTION) {
            Log.d(TAG, "addTouchTarget:child = " + child + ",pointerIdBits = " + pointerIdBits
                    + ",target = " + target + ",mFirstTouchTarget = " + mFirstTouchTarget
                    + ",this = " + this);
        }
        
        target.next = mFirstTouchTarget;
        mFirstTouchTarget = target;
        return target;
    }
```

由于在多点触控中，每一根手指都对应一个 pointerId，那么 TouchTarget 的作用就是，保存可以处理当前事件的子 View 对象（通过 TouchTarget 的成员变量 child 保存），以及，这个 View 对象具体可以处理哪几根指头（通过 TouchTarget 的 pointerIdBits 成员变量保存），也是 TouchTarget 的注释所表达的意思：

```
/* Describes a touched view and the ids of the pointers that it has captured.
     *
     * This code assumes that pointer ids are always in the range 0..31 such that
     * it can use a bitfield to track which pointer ids are present.
     * As it happens, the lower layers of the input dispatch pipeline also use the
     * same trick so the assumption should be safe here...
     */
    private static final class TouchTarget {
        ......

        // The touched child view.
        @UnsupportedAppUsage
        public View child;

        // The combined bit mask of pointer ids for all pointers captured by the target.
        public int pointerIdBits;
        
        .......
    }
```

最后这里通过：

```
target.next = mFirstTouchTarget;
        mFirstTouchTarget = target;
```

构建出了一个 TouchTarget 的链表，每调用一次 ViewGroup.addTouchTarget，就会在链表头插入一个新的元素，mFirstTouchTarget 始终指向链表第一个元素。

###### 6.7.4.3.6 对找不到目标 View 的事件的处理

```
if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
```

对于单点触控，如果 mFirstTouchTarget 不为空，那么 newTouchTarget 应该也是不会为空的，所以这个部分处理的还是多点触控的逻辑。

这部分逻辑发生的条件可能是，假设，现在第一根手指按下，对应 ACTION_DOWN 流程，此时找到了一个可以处理事件的子 View，那么可以基于该子 View 创建一个 TouchTarget，并且 mFirstTouchTarget 也会指向这个 TouchTarget。此时第二根手指按下，对应 ACTION_POINTER_DOWN 流程，但是如果这时找不到一个可以处理当前事件的子 View，那么 newTouchTarget 也就仍然为空，但是 mFirstTouchTarget 却不空，那这部分的逻辑就会执行：

将这个 pointerID 加入到最近一次加入到 TouchTarget 链表中的 TouchTarget，表示最近一次添加的 TouchTarget 将会处理本次无 View 认领的 MotionEvent 事件。

#### 6.7.5 由当前 ViewGroup 处理本次事件

```
// Dispatch to touch targets.            
			if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                ......
            }
```

这里 mFirstTouchTarget 为空，说明当前事件可能被 cancel 了，或者可能被当前 ViewGroup 拦截了，或者子 View 中没有任何一个能够处理这个事件。

如果发生以上三种情况之一，我们就调用 dispatchTransformedTouchEvent 方法。这里传入的 child 参数为 NULL，那么这一步的目的是将当前 ViewGroup 看作一个普通的 View，调用 VIew.dispatchTouchEvent，由当前 ViewGroup 去处理当前事件：

```
if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                ......
                handled = child.dispatchTouchEvent(event);
                ......
            }
```

之前在遍历子 View 的时候也调用过 dispatchTransformedTouchEvent 方法，但是由于当时传入的参数 chid 都不为空，所以事件其实是发送给 ViewGroup 的子 View 去处理了。

```
/**
     * Pass the touch screen motion event down to the target view, or this
     * view if it is the target.
     *
     * @param event The motion event to be dispatched.
     * @return True if the event was handled by the view, false otherwise.
     */
    public boolean dispatchTouchEvent(MotionEvent event) {
        // If the event should be handled by accessibility focus first.
        if (event.isTargetAccessibilityFocus()) {
            // We don't have focus or no virtual descendant has it, do not handle the event.
            if (!isAccessibilityFocusedViewOrHost()) {
                return false;
            }
            // We have focus and got the event, then use normal event dispatch.
            event.setTargetAccessibilityFocus(false);
        }
        boolean result = false;

		......

        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

		.....

        return result;
    }
```

这个方法可以分成三个部分。

##### 6.7.5.1 accessibility focus

```
// If the event should be handled by accessibility focus first.
        if (event.isTargetAccessibilityFocus()) {
            // We don't have focus or no virtual descendant has it, do not handle the event.
            if (!isAccessibilityFocusedViewOrHost()) {
                return false;
            }
            // We have focus and got the event, then use normal event dispatch.
            event.setTargetAccessibilityFocus(false);
        }
```

accessibility focus 相关的部分在 6.7.1 中也有提到，如果 MotionEvent.isTargetAccessibilityFocus 返回 true，表示当前事件是面向 accessibility focus View 的。如果当前 View 不是 accessibility focus View，那么不要让这个 View 处理当前事件，直接返回 false。否则，清除掉当前事件的相关标志位，并且开始正常的事件分发流程。

##### 6.7.5.2 OnTouchListener.onTouch

```
ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
```

如果 ListenerInfo 类型的成员变量 mListenerInfo 中，OnTouchListener 类型的成员变量 mOnTouchListener 不为空，那么首先调用 mOnTouchListener.onTouch 方法去处理当前事件。

成员变量 mListenerInfo 在盗用 View.getListenerInfo 的时候赋值化：

```
@UnsupportedAppUsage
    ListenerInfo getListenerInfo() {
        if (mListenerInfo != null) {
            return mListenerInfo;
        }
        mListenerInfo = new ListenerInfo();
        return mListenerInfo;
    }
```

ListenerInfo 的成员变量 mOnTouchLIstener 在调用 View.setOnTouchListener 的时候赋值：

```
/**
     * Register a callback to be invoked when a touch event is sent to this view.
     * @param l the touch listener to attach to this view
     */
    public void setOnTouchListener(OnTouchListener l) {
        getListenerInfo().mOnTouchListener = l;
    }
```

这个很常用就不用再赘述用如何使用了。

最后看下 OnTouchListener 的定义：

```
/**
     * Interface definition for a callback to be invoked when a touch event is
     * dispatched to this view. The callback will be invoked before the touch
     * event is given to the view.
     */
    public interface OnTouchListener {
        /**
         * Called when a touch event is dispatched to a view. This allows listeners to
         * get a chance to respond before the target view.
         *
         * @param v The view the touch event has been dispatched to.
         * @param event The MotionEvent object containing full information about
         *        the event.
         * @return True if the listener has consumed the event, false otherwise.
         */
        boolean onTouch(View v, MotionEvent event);
    }
```

OnTouchListener 是一个接口，定义了一个触摸事件被发送给当前 View 的时候调用的回调。这个回调将会在触摸事件发送给 View 之前被调用，也就是 View.onTouchEvent。

onTouch 方法会在一个触摸事件发送给一个 View 的时候被调用。这允许 listener 在目标 View 之前可以得到一个回应触摸事件的先机。

那这里的代码就说明了，当一个触摸事件发送给某一个 View 的时候，如果这个 View 注册了 OnTouchListener，那么先由 OnTouchListener.onTouch 处理当前事件。如果 OnTouchListener.onTouch 处理了这个触摸事件，那么这个事件就不再发送给 View.onTouchEvent。

##### 6.7.5.3 View.onTouchEvent

```
if (!result && onTouchEvent(event)) {
                result = true;
            }
```

如果上一步没有处理当前事件，那么把 MotionEvent 发送给当前 View 的 onTouchEvent 方法中去处理。

```
/**
     * Implement this method to handle touch screen motion events.
     * <p>
     * If this method is used to detect click actions, it is recommended that
     * the actions be performed by implementing and calling
     * {@link #performClick()}. This will ensure consistent system behavior,
     * including:
     * <ul>
     * <li>obeying click sound preferences
     * <li>dispatching OnClickListener calls
     * <li>handling {@link AccessibilityNodeInfo#ACTION_CLICK ACTION_CLICK} when
     * accessibility features are enabled
     * </ul>
     *
     * @param event The motion event.
     * @return True if the event was handled, false otherwise.
     */
    public boolean onTouchEvent(MotionEvent event) {
        ......
    }
```

实现这个方法用来处理触屏运动事件。

如果此方法用于检测单击操作，建议通过实现和调用 performClick 来执行这些操作。这将确保一致的系统行为，包括：

遵守点击声音偏好，分发 OnClickListener 调用，处理 AccessibilityNodeInfo.ACTION_CLICK。

返回 true 说明当前事件被处理。

唯一需要所说的点，就是如果你通过 View.setOnClickListener 注册了一个监听对点击动作的 OnClickListener：

```
/**
     * Register a callback to be invoked when this view is clicked. If this view is not
     * clickable, it becomes clickable.
     *
     * @param l The callback that will run
     *
     * @see #setClickable(boolean)
     */
    public void setOnClickListener(@Nullable OnClickListener l) {
        if (!isClickable()) {
            setClickable(true);
        }
        getListenerInfo().mOnClickListener = l;
    }
```

那么在 ACTION_UP 动作的时候，会调用 View.performClickInternal -> View.performClick -> OnClickListener.onClick 来回调 OnClickListener 的 onClick 方法。

#### 6.7.6 将事件发送给 TouchTarget 去处理

```
// Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                ......
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
						......	
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }
```

##### 6.7.6.1 将事件发送给目标子 View

```
if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
```

不同于 6.7.5，如果 mFirstTouchTarget 不为空，那么说明在之前的 DOWN 流程中找到了可以处理 MotionEvent 的 TouchTarget，或者说目标子 View，那么这里遍历 TouchTarget 链表，对于每一个 TouchTarget 中保存的子 View，都调用 ViewGroup.dispatchTransformedTouchEvent 将事件发送给目标子 View 去处理。这也就是说，如果事件在 ACTION_DOWN 的时候被 ViewGroup 拦截，mFirstTouchTarget 就会为 NULL，那么事件就不会发送给当前 ViewGroup 的子 View 去处理了。

这也是一种目标子 View 的复用，只不过这种复用是针对 ACTION_MOVE、ACTION_UP 来说的，对于这些行为的事件，不用再去遍历当前 ViewGroup 的所有子 View 去寻找可以处理当前输入事件的目标 View，而直接复用 DOWN 流程中找到的目标子 View 即可。6.7.4.3.3 中也讲到了子 View 的复用，那里的复用指的是 ACTION_POINTER_DOWN 流程复用 ACTION_DOWN 和 ACTION_POINTER_DOWN 流程中找到的目标子 VIew。

##### 6.7.6.2 避免重复分发

```
if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    }
```

要知道 ACTION_DOWN、ACTION_POINTER_DOWN、ACTION_MOVE、ACTION_UP 等都会走到这里，而对于 ACTION_DOWN、ACTION_POINTER_DOWN 来说，在 6.7.4.3.4 中，可能已经调用了 ViewGroup.dispatchTransformedTouchEvent 把这些事件分发给子 View 去处理了，那么在 6.7.4.3.5 中 alreadyDispatchedToNewTouchTarget 会被置为 true，也会基于目标子 View 构建一个 newTouchTarget，所以在这里就不用重复调用 ViewGroup.dispatchTransformedTouchEvent 了。

_但是有一个问题，对于当前事件，如果它可以已经在 6.7.4.3.4 中找到了目标 View，并且目标 View 被保存到 newTouchTarget 中。但是如果 TouchTarget 链表中有多个 TouchTarget，那么在遍历到 newTouchTarget 之前，它还是可能会被发送给其他 TouchTarget 中保存的子 View，即使这些子 View 不是当前事件的目标 View。那么这里是否应该直接将找到了目标 View 的事件直接发送给目标 View，不用再遍历了？_

##### 6.7.6.3 cancel 情况的处理

```
final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
						......	
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
```

想象这样一种情况，如果当前 ViewGroup 没有在 ACTION_DOWN 的时候拦截事件，而是选择在 ACTION_MOVE 的时候拦截事件，那么首先由于 mFirstTouchTarget 不为 NULL，代码就会执行到这里，然后这里的 cancelChild 就会置为 true，接下来在 ViewGroup.dispatchTransformedTouchEvent 方法中，子 View 收到事件的 action 会被设置为 ACTION_CANCEL：

```
if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }
```

最后，目标子 View 对应的 TouchTarget 将会从当前 ViewGroup 的 TouchTarget 链表中移除，那么目标子 View 只会收到两次事件，ACTION_DOWN 和 ACTION_CANCEL。

#### 6.7.7 在 gesture 结束后清除 TouchTarget

```
// Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
```

由于 TouchTarget 保存了每一次 gesture 中可以接收事件的目标子 View，那么在每一次 gesture 的结束，都调用 ViewGroup.resetTouchState 将本次 gesture 保存的一些状态清除。如果只是某一根手指抬起，ACTION_POINTER_UP，那么只将该手指从 TouchTarget 中移除。

#### 6.7.8 ViewGroup.dispatchTouchEvent 小结

从以上的分析，可以大概总结，ViewGroup.dispatchTouchEvent 的事件分发流程关键节点主要是两个：

1）、调用 ViewGroup.onInterceptTouchEvent 拦截 MotionEvent，如果拦截成功，那么事件不会再下发给它的子 View 去处理。

2）、调用 ViewGroup.dispatchTransformedTouchEvent 继续分发事件，这基于目标子 View 的情况可能有三种分支：

```
2.1）、没有目标子View，当前ViewGroup调用View.dispatchTouchEvent自己处理事件。

2.2）、目标子View是一个ViewGroup，调用目标子View的ViewGroup.dispatchTouchEvent继续分发事件。

2.3）、目标子View不是一个ViewGroup，调用目标子View的View.dispatchTouchEvent处理事件。
```

为了验证上面分析的内容，自己写一个 demo App 进行检验，App 的层级结构是：

```
View Hierarchy:
      DecorView@a063fe8[MainActivity]
        android.widget.LinearLayout{48fccda V.E...... ........ 0,0-1200,1824}
          android.view.ViewStub{bb516fb G.E...... ......I. 0,0-0,0 #10201c4 android:id/action_mode_bar_stub}
          android.widget.FrameLayout{1921601 V.E...... ........ 0,48-1200,1824 #1020002 android:id/content}
            com.test.inputinviewhierarchy.MyLayout{3addca6 V.E...... ........ 0,0-1200,1776}
              com.test.inputinviewhierarchy.MyTextView{8a5b6e7 VFED..C.. ........ 0,100-1200,200 #7f08016f app:id/text1}
        android.view.View{9f37294 V.ED..... ........ 0,1824-1200,1920 #1020030 android:id/navigationBarBackground}
        android.view.View{c09113d V.ED..... ........ 0,0-1200,48 #102002f android:id/statusBarBackground}
```

忽略不必要的部分：

```
View Hierarchy:
      DecorView@a063fe8[MainActivity]
        android.widget.LinearLayout{48fccda V.E...... ........ 0,0-1200,1824}
          android.widget.FrameLayout{1921601 V.E...... ........ 0,48-1200,1824 #1020002 android:id/content}
            com.test.inputinviewhierarchy.MyLayout{3addca6 V.E...... ........ 0,0-1200,1776}
              com.test.inputinviewhierarchy.MyTextView{8a5b6e7 VFED..C.. ........ 0,100-1200,200 #7f08016f app:id/text1}
```

其中 MyLayout 是我自己写的 Activity 加载的布局结构，继承 RelativeLayout，是一个 ViewGroup。MyTextView 继承 TextView，是一个非 ViewGroup 的普通 View。

分别看下 MyLayout 主动拦截 MotionEvent，或者 MyTextView 不处理 MotionEvent，会导致什么样的结果。

##### 6.7.8.1 默认流程

默认流程即没有经过任何特殊处理，MyLayout 默认不拦截 MotionEvent，MyTextView 默认处理当前事件。

![](https://img-blog.csdnimg.cn/bcb678b50bcd4c50ba66ba3ad486abee.png#pic_center)

由于 MyTextView 处理了本次事件，那么在每一级的 ViewGroup 中，都可以构建一个 TouchTarget 链表出来，根据 6.7.6 的分析，后续 ACTION_MOVE 和 ACTON_UP 等，直接发送给 TouchTarget 去处理即可，当前 ViewGroup 无需自己处理。

##### 6.7.8.2 MyTextView 不处理事件

这里修改 MyTextView 的逻辑，不让它处理本次事件：

```
@Override
    public boolean onTouchEvent(MotionEvent event) {
        // return super.onTouchEvent(event);
        return false;
    }
```

那么事件分发的流程是：

![](https://img-blog.csdnimg.cn/f8053a9b0a2241e185fa3500194df429.png#pic_center)

1）、ACTION_DOWN：由于 MyLayout 的唯一子 View 不处理本次事件，导致在 MyLayout 这一级中，构建不出 TouchTarget 链表，因此就只能由 MyLayout 调用 View.dispatchTouchEvent 来处理本次事件，即 6.7.5 的内容。但是 MyLayout 继承自 RelativeLayout，本身也是无法处理 MotionEvent 的，其它的 Layout 也是这样的情况，所以 View 层级结构中，自下到上，每一级父 View 都会通过 View.dispatchTouchEvent -> View.onTouchEvent 尝试去处理本次事件，但是在它们的 View.onTouchEvent 都返回了 false，所以事件最终没有得到处理，传给了下一个 InputStage：SyntheticInputStage。

2）、ACTION_MOVE、ACTION_UP：由于在 ACTION_DOWN 中，找不到一个目标 View 可以处理本次事件，那么每一级 ViewGroup 中的 TouchTarget 链表自然也就没有建立起来。根据 6.7.5，此时，最顶层的 DecorVIew 将不会向下发送事件，而是选择自己处理。

##### 6.7.8.3 MyLayout 拦截 ACIION_DOWN 并处理

这里修改 MyLayout 的逻辑，在 ViewGroup.onInterceptTouchEvent 中，判断当前事件 action 为 ACTION_DOWN 的时候拦截事件并且消费掉该事件：

```
@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
//        return super.onInterceptTouchEvent(ev);
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            return true;
        }
        return false;
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
//        return super.onTouchEvent(event);
        return true;
    }
```

那么事件分发的流程是：

![](https://img-blog.csdnimg.cn/047ad5c5918b4090b19ea2d4dd15ae4c.png#pic_center)

和 6.7.8.1 相比，区别在于 MyLayout 选择拦截 ACTION_DOWN，那么事件将不会再发送给 MyLayout 的子 View，MyTextView，MyLayout 变成了本次 gesture 分发的最底层 VIew。

##### 6.7.8.4 MyLayout 拦截 ACIION_DOWN 不处理

这里修改 MyLayout 的逻辑，在 ViewGroup.onInterceptTouchEvent 中，判断当前事件 action 为 ACTION_DOWN 的时候拦截事件但是不作处理：

```
@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
//        return super.onInterceptTouchEvent(ev);
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            return true;
        }
        return false;
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        return super.onTouchEvent(event);
    }
```

那么事件分发的流程是：

![](https://img-blog.csdnimg.cn/61a85835063c4e07818ebd06b890a724.png#pic_center)

和 6.7.8.2 相比，区别在于 MyLayout 选择拦截 ACTION_DOWN，那么事件将不会再发送给 MyLayout 的子 View，MyTextView，MyLayout 变成了本次 gesture 分发的最底层 VIew。

和 6.7.8.3 相比，区别在于 ACTION_DOWN 流程中，在 View 层级结构中自下而上，每一级 ViewGroup 都会通过 View.dispatchTouchEvent -> View.onTouchEvent 尝试去处理本次事件，但是由于每一级 ViewGroup 的默认 onTouchEvent 实现都是不处理事件的，所以事件最终没有得到处理，传给了下一个 InputStage：SyntheticInputStage。后续事件中由于 TouchTarget 链表在每一级中都没有构建起来，所以事件直接在 DecorView 这一级中被消费。

##### 6.7.8.5 MyLayout 拦截 ACTION_MOVE 并处理

这里修改 MyLayout 的逻辑，在 ViewGroup.onInterceptTouchEvent 中，判断当前事件 action 为 ACTION_DOWN 的时候拦截事件并且消费掉该事件：

```
@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
//        return super.onInterceptTouchEvent(ev);
        if (ev.getAction() == MotionEvent.ACTION_MOVE) {
            return true;
        }
        return false;
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
//        return super.onTouchEvent(event);
        return true;
    }
```

那么事件分发的流程是：

![](https://img-blog.csdnimg.cn/34bb33558a4443f5b8ec3e932398197c.png#pic_center)

1）、ACTION_DOWN：该流程和 6.7.8.1 一样，最终在每一级 ViewGroup 中都构建了一个 TouchTarget 链表。

2）、ACTION_MOVE：虽然 ACTION_MOVE 被 MyLayout 拦截，但是此时并不是像 6.7.8.3 一样，本次事件不会再发送给 MyTextView。事件仍然会发送给 MyTextView，因为在 ACTION_DOWN 流程中，我们基于 MyTextView 构建了一个 TouchTarget 对象，并且把 mFirstTouchTarget 指向了该对象。但是事件的 action 从 ACTION_MOVE 被转化为 ACTION_CANCEL，随后以 MyTextView 为目标 View 的 TouchTarget 被移除，即 6.7.6.3 的分析。

3）、后续的 ACTION_MOVE、ACTION_CANCEL：由于第 2 步中，在 MyLayout 一级中，没有了子 View 可以处理事件，那么最终事件会由 MyLayout 处理。
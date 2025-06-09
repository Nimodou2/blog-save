> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7139024123258224654)

> KeyEvent DispatchKeyEvent DispatchKeyEvent DispatchKeyEvent

7.1 ViewRootImpl.ViewPostImeInputStage.processKeyEvent
------------------------------------------------------

```
        private int processKeyEvent(QueuedInputEvent q) {
            final KeyEvent event = (KeyEvent)q.mEvent;

		   ......	

            
            if (mView.dispatchKeyEvent(event)) {
                if (ViewDebugManager.DEBUG_ENG) {
                    Log.v(mTag, "App handle key event: event = " + event + ", mView = " + mView
                            + ", this = " + this);
                }
                return FINISH_HANDLED;
            }

		   ......	
        }


```

专注于 KeyEvent 是如何发送给 View 层级结构的，其他的暂时不关注。

这里的 mView 是 View 层级结构的根 VIew，对于 Activity 来说就是 DecorView。

7.2 DecorView.dispatchKeyEvent
------------------------------

```
    @Override
    public boolean dispatchKeyEvent(KeyEvent event) {
        final int keyCode = event.getKeyCode();
        final int action = event.getAction();
        final boolean isDown = action == KeyEvent.ACTION_DOWN;

		......

        if (!mWindow.isDestroyed()) {
            final Window.Callback cb = mWindow.getCallback();
            final boolean handled = cb != null && mFeatureId < 0 ? cb.dispatchKeyEvent(event)
                    : super.dispatchKeyEvent(event);
            if (handled) {
                return true;
            }
        }

        return isDown ? mWindow.onKeyDown(mFeatureId, event.getKeyCode(), event)
                : mWindow.onKeyUp(mFeatureId, event.getKeyCode(), event);
    }


```

这里的 mWIndow 即 PhoneWindow，PhoneWIndow 的 callback 是 Activity.attach 的时候通过 PhoneWindow.setCallback 设置的，传入的是 Activity 自己。

那么这里的逻辑是，先调用 Activity.dispatchKeyEvent 去处理 KeyEvent，如果 Activity 能够处理，那么当前输入事件被认为处理完成。否则，调用 PhoneWindow.onKeyDown 或者 PhoneWindow.onKeyUp 处理。

7.3 Activity.dispatchKeyEvent
-----------------------------

```
    
    public boolean dispatchKeyEvent(KeyEvent event) {
        onUserInteraction();

        
        
        final int keyCode = event.getKeyCode();
        if (keyCode == KeyEvent.KEYCODE_MENU &&
                mActionBar != null && mActionBar.onMenuKeyEvent(event)) {
            return true;
        }

        Window win = getWindow();
        if (win.superDispatchKeyEvent(event)) {
            return true;
        }
        View decor = mDecor;
        if (decor == null) decor = win.getDecorView();
        return event.dispatch(this, decor != null
                ? decor.getKeyDispatcherState() : null, this);
    }


```

这个方法用来处理 KeyEvent 事件，你可以重写这个方法，从而在所有 KeyEvent 发送给 Window 之前将其拦截。

这个方法的主要内容有：

1）、如果当前 KeyEvent 对应 KeyEvent.KEYCODE_MENU，那么尝试让 ActionBar 调用 onMenuKeyEvent 去处理此事件。

2）、调用 PhoneWindow.superDispatchKeyEvent 继续往 View 层级结构发送 KeyEvent。

3）、如果经过上面两步，当前 KeyEvent 还没有被处理，那么调用 Activity 的 onKeyDown、onKeyLongPress 和 onKeyUp 等方法去处理当前事件。

主要分析第二步。

7.4 PhoneWindow.superDispatchKeyEvent
-------------------------------------

```
    @Override
    public boolean superDispatchKeyEvent(KeyEvent event) {
        return mDecor.superDispatchKeyEvent(event);
    }


```

mDecor 成员变量是一个 DecorView 对象，那么这里调用的就是 DecorView.superDispatchKeyEvent。

7.5 DeocrView.superDispatchKeyEvent
-----------------------------------

```
    public boolean superDispatchKeyEvent(KeyEvent event) {
		......

        if (super.dispatchKeyEvent(event)) {
            return true;
        }

        ......
    }


```

这里调用的是父类的 dispatchKeyEvent 方法，由于直接父类 FrameLayout 没有重写这个方法，因此调用的就是 ViewGroup 的方法。

7.6 ViewGroup.dispatchKeyEvent
------------------------------

```
    @Override
    public boolean dispatchKeyEvent(KeyEvent event) {
		......

        if ((mPrivateFlags & (PFLAG_FOCUSED | PFLAG_HAS_BOUNDS))
                == (PFLAG_FOCUSED | PFLAG_HAS_BOUNDS)) {
            if (super.dispatchKeyEvent(event)) {
                return true;
            }
        } else if (mFocused != null && (mFocused.mPrivateFlags & PFLAG_HAS_BOUNDS)
                == PFLAG_HAS_BOUNDS) {
			......
            if (mFocused.dispatchKeyEvent(event)) {
                return true;
            }
        }

		......
        return false;
    }


```

这里的内容很简单：

1）、如果当前 ViewGroup 的 mPrivateFlags 有 PFLAG_FOCUSED 和 PFLAG_HAS_BOUNDS 这两个标记，那么说明当前 ViewGroup 拥有焦点，并且已经设置了自身的区域，那么调用基类 VIew.dispatchKeyEvent，表示当前 KeyEvent 将由当前 ViewGroup 处理，不会再分发给子 View。

2）、如果 mFocused 不为 NULL，且 mFocused 的 mPrivateFlags 有 PFLAG_HAS_BOUNDS，那么将事件发送给 mFocused 去处理。如果 mFocused 重写了 VIew.dispatchKeyEvent 方法，那么就调用 mFocused 自己的 VIew.dispatchKeyEvent 方法，否则调用基类的 VIew.dispatchKeyEvent。

在进一步分析 VIew.dispatchKeyEvent 之前，这里有几个和焦点相关的概念需要先弄懂是什么意思。

### 7.6.1 PFLAG_HAS_BOUNDS

PFLAG_HAS_BOUNDS 同样是 mPrivateFlags 的标志位之一：

```
    
    static final int PFLAG_HAS_BOUNDS                  = 0x00000010;


```

PFLAG_HAS_BOUNDS 唯一被添加到 mPrivateFlags 的地方在 View.setFrame 中，那么可以尝试去理解为什么在 View.dispatchKeyEvent 中要去判断 PFLAG_HAS_BOUNDS 的意义：如果 PFLAG_HAS_BOUNDS 没有设置，说明此时当前 View 还没有 layout 完成，自身的 frame 还没有设置，那么这个 View 是无法作为焦点 View 的存在去处理当前的 KeyEvent 的。

### 7.6.2 PFLAG_FOCUSED

PFLAG_FOCUSED 定义在 View 中，作为 View 的 mPrivateFlags 成员变量的标志位：

```
    
    static final int PFLAG_FOCUSED                     = 0x00000002;


```

首先看几个判断 PFLAG_FOCUSED 的地方：

```
    
    @ViewDebug.ExportedProperty(category = "focus")
    public boolean hasFocus() {
        return (mPrivateFlags & PFLAG_FOCUSED) != 0;
    }

    
    @ViewDebug.ExportedProperty(category = "focus")
    @InspectableProperty(hasAttributeId = false)
    public boolean isFocused() {
        return (mPrivateFlags & PFLAG_FOCUSED) != 0;
    }


```

hasFocus 注释是说，如果某一个 View 的 mPrivateFlags 包含 PFLAG_FOCUSED，说明当前 View 持有焦点，或者是持有焦点的 View 的祖先 View。

isFocused 注释是说，如果某一个 View 的 mPrivateFlags 包含 PFLAG_FOCUSED，说明当前 View 持有焦点。

感觉这个两个方法的意义是有一点冲突。

### 7.6.3 View.handleFocusGainInternal

PFLAG_FOCUSED 唯一添加到 mPrivateFlags 的地方在 View.handleFocusGainInternal：

```
    
    void handleFocusGainInternal(@FocusRealDirection int direction, Rect previouslyFocusedRect) {
        if (DBG || ViewDebugManager.DEBUG_FOCUS) {
            System.out.println(this + " requestFocus()");
            Log.d(VIEW_LOG_TAG, "handleFocusGainInternal: this = " + this + ", callstack = " ,
                    new Throwable("ViewFocus"));
        }

        if ((mPrivateFlags & PFLAG_FOCUSED) == 0) {
            mPrivateFlags |= PFLAG_FOCUSED;

            View oldFocus = (mAttachInfo != null) ? getRootView().findFocus() : null;

            if (mParent != null) {
                mParent.requestChildFocus(this, this);
                updateFocusedInCluster(oldFocus, direction);
            }

            if (mAttachInfo != null) {
                mAttachInfo.mTreeObserver.dispatchOnGlobalFocusChange(oldFocus, this);
            }

            onFocusChanged(true, direction, previouslyFocusedRect);
            refreshDrawableState();
        }
    }


```

给当前 View 焦点。这将会导致 View.onFocusChanged 方法被调用。

注意：这里不会检查当前 View 是否应该取得焦点，只是把焦点给到这个 View，无论如何。这个方法应该只被 framwork 内部的，知道当前正在发生什么的代码调用，即 View.requestFocus 方法。

从这里再来看 7.6.2，感觉只是持有焦点的 View 的 mPrivateFlags 才会被添加 PFLAG_FOCUSED 标记，持有焦点的 View 的祖先 View 并不会添加。

这里除了注释中所说的 View.onFocusChanged 方法之外，还有一个重要的点：

```
mParent.requestChildFocus(this, this);


```

调用了 ViewParent.requestChildFocus 方法，这个后面会一起分析。

### 7.6.4 ViewGroup.mFocused

```
    
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P, trackingBug = 115609023)
    private View mFocused;


```

当前 VIewGroup 中的持有或者包含焦点的 View。

唯一赋值的地方在 ViewGroup.requestChildFocus 方法：

```
    @Override
    public void requestChildFocus(View child, View focused) {
        if (DBG || ViewDebugManager.DBG) {
            System.out.println(this + " requestChildFocus()");
        }
        if (getDescendantFocusability() == FOCUS_BLOCK_DESCENDANTS) {
            return;
        }

        
        super.unFocus(focused);

        
        if (mFocused != child) {
            if (mFocused != null) {
                mFocused.unFocus(focused);
            }

            mFocused = child;
        }
        if (mParent != null) {
            mParent.requestChildFocus(this, focused);
        }
    }


```

该方法重写自 ViewParent.requestChildFocus：

```
    
    public void requestChildFocus(View child, View focused);


```

在当前 ViewParent 的一个子 View 想要获取焦点的时候调用。

参数 child 代表当前 ViewParent 中的一个想要焦点的子 View。这个子 View 将会包含焦点 View，但是它不一定是实际持有焦点的那个 View。

参数 focused 这里我理解的是，就是实际上那个持有焦点的 View，是第一个参数 child 的后代 View。

这里看到 ViewGroup.requestChildFocus 有两个重要作用，一个是将 mFocused 指向包含焦点的子 View，另一个是递归调用父 View 的 requestChildFocus 方法。

### 7.6.5 焦点请求实际验证

这里写了一个 demo App，层级结构是：

```
    View Hierarchy:
      DecorView@fce7e75[MainActivity]
        android.widget.LinearLayout{730e35f V.E...... ........ 0,0-1200,1824}
          android.view.ViewStub{6df4c47 G.E...... ......I. 0,0-0,0 #10201c4 android:id/action_mode_bar_stub}
          android.widget.FrameLayout{8f1ae0a V.E...... ........ 0,48-1200,1824 #1020002 android:id/content}
            com.test.inputinviewhierarchy.MyLayout{7aa567b V.E...... ........ 0,0-1200,1776}
              android.widget.EditText{6349195 VFED..CL. ........ 0,0-1200,200}
        android.view.View{5eeca44 V.ED..... ........ 0,1824-1200,1920 #1020030 android:id/navigationBarBackground}
        android.view.View{ce5072d V.ED..... ........ 0,0-1200,48 #102002f android:id/statusBarBackground}


```

简化掉不必要的部分：

```
    View Hierarchy:
      DecorView@fce7e75[MainActivity]
        android.widget.LinearLayout{730e35f V.E...... ........ 0,0-1200,1824}
          android.widget.FrameLayout{8f1ae0a V.E...... ........ 0,48-1200,1824 #1020002 android:id/content}
            com.test.inputinviewhierarchy.MyLayout{7aa567b V.E...... ........ 0,0-1200,1776}
              android.widget.EditText{6349195 VFED..CL. ........ 0,0-1200,200}


```

其中 MyLayout 是我自己写的 Activity 加载的布局结构，继承 RelativeLayout，只包含了一个普通的 EditText。

首先 Activity 启动后，EditText 并不会请求焦点，但是如果我们用手指点击了 EditText 的相关区域，EditText 就会去请求焦点：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1952236a6cdd4219961275fb8900fca6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这里看到 EditText 是在 View.onTouchEvent 中调用 View.requestFocus 去请求了焦点，最终会调用到 View.handleFocusGainInternal 中。并且整个过程中，只有 EditText 调用了 handleFocusGainInternal 方法，其他 View 并没有调用，这也印证了 7.6.3 的说法，只有持有焦点的 View 的 mPrivateFlags 才会被添加 PFLAG_FOCUSED 标记，持有焦点的 View 的祖先 View 并不会添加。

根据 7.7.3，当 EditText 的 mPrivateFlags 被添加 PFLAG_FOCUSED 标志之后，会调用 ViewGroup.requestChildFocus 方法：

```
mParent.requestChildFocus(this, this);


```

根据 7.7.4，ViewGroup.requestChildFocus 中会让当前 ViewGroup 的 mFocused 指向该 View，并且递归调用 ViewGroup.requestChildFocus：

```
        if (mParent != null) {
            mParent.requestChildFocus(this, focused);
        }


```

递归调用的 debug 情况是：

1）、MyLayout.requestChildFocus

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b5c2981cdf54735941b7cb783a06419~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

MyLayout 的 mFocused 指向 EditText。

2）、FrameLayout.requestChildFocus

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e73cdebe721445384d52c46b43c8457~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

FrameLayout 的 mFocused 指向 MyLayout。

3）、LinearLayout.requestChildFocus

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/01e61cf3a4a84d8b9a473c15d5361ddd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

LinearLayout 的 mFocused 指向 FrameLayout。

4）、DecorView.requestChildFocus

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/244082278de744469704d1c3573670cc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

DecorView 的 mFocused 指向 LinearLayout。

最终 View 层级结构中会形成一个自根 View，DecorView，到调用 View.requestFocus 的那个 View，EditText，的一条子树：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6d6560d9b074d35a69b58d9d7f220a9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

该子树上所有的 View 都是直接或者间接包含焦点的 View，KeyEvent 事件按照这个子树从上往下进行分发即可。

7.7 View.dispatchKeyEvent
-------------------------

接 7.6 继续分析最后的 View.dispatchKeyEvent。

```
    
    public boolean dispatchKeyEvent(KeyEvent event) {
		......

        
        
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnKeyListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnKeyListener.onKey(this, event.getKeyCode(), event)) {
            return true;
        }

        if (event.dispatch(this, mAttachInfo != null
                ? mAttachInfo.mKeyDispatchState : null, this)) {
            return true;
        }

		......
        return false;
    }


```

### 7.8.1 OnKeyListener.onKey

```
        
        
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnKeyListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnKeyListener.onKey(this, event.getKeyCode(), event)) {
            return true;
        }


```

如果 ListenerInfo 类型的成员变量 mListenerInfo 中的 OnKeyListener 类型的成员变量 mOnKeyListener 不为 NULL，说明当前 View 通过 View.setOnKeyListener：

```
    
    public void setOnKeyListener(OnKeyListener l) {
        getListenerInfo().mOnKeyListener = l;
    }


```

注册了一个 OnKeyListener：

```
    
    public interface OnKeyListener {
        
        boolean onKey(View v, int keyCode, KeyEvent event);
    }


```

这允许 KeyEvent 在分发给焦点 View 之前，给 OnKeyListener 一个处理 KeyEvent 的机会。这里处理的都是硬件键盘产生的 KeyEvent，软键盘生成的 KeyEvent 通常不会触发这个方法。

### 7.8.2 由当前 View 来处理 KeyEvent

```
        if (event.dispatch(this, mAttachInfo != null
                ? mAttachInfo.mKeyDispatchState : null, this)) {
            return true;
        }


```

如果 OnKeyListener 不处理本次事件，那么由当前 View 来处理本次事件。

这里调用了 KeyEvent.dispatch 方法：

```
    
    public final boolean dispatch(Callback receiver, DispatcherState state,
            Object target) {
        switch (mAction) {
            case ACTION_DOWN: {
				......
                boolean res = receiver.onKeyDown(mKeyCode, this);
                if (state != null) {
                    if (res && mRepeatCount == 0 && (mFlags&FLAG_START_TRACKING) != 0) {
						......
                    } else if (isLongPress() && state.isTracking(this)) {
                        try {
                            if (receiver.onKeyLongPress(mKeyCode, this)) {
							......
                            }
                        } catch (AbstractMethodError e) {
                        }
                    }
                }
                return res;
            }
            case ACTION_UP:
				......
                return receiver.onKeyUp(mKeyCode, this);
            ......    
        }
        return false;
    }


```

这里传入的 Callback 类型的 receiver 参数是 VIew 自身，View 实现了 KeyEvent.Callback 接口，那么最终会调用 VIew 的 onKeyDown、onKeyLongPress、onKeyUp 和 onKeyMultiple 来处理当前事件。

7.8 小结
------

1）、KeyEvent 的处理顺序优先级是，View Hierarchy > Activity > PhoneWindow，首先由 View 的 onKeyDown 和 onKeyUp 等方法去处理 KeyEvent。如果 View Hierarchy 中找不到 View 可以处理 KeyEvent，那么再调用 Activity 的 onKeyDown 和 onKeyUp 等方法去处理 KeyEvent。如果 Activity 也处理不了，那么最后由 PhoneWindow 的 onKeyDown 和 onKeyUp 等方法去处理 KeyEvent。

2）、在分发 KeyEvent 之前，View Hierarchy 中需要先构建一个自根 View 至下的一个焦点 VIew 子树，KeyEvent 只会在这个子树中进行分发，不会像 MotionEvent 一样遍历当前 ViewGroup 的所有子 VIew 去寻找满足接收条件的子 View。这个思路和 InputDispatcher 向窗口分发事件是一样的，对于 key 类型的事件，由于这类事件不像 Touch 事件一样有坐标，因此我们只能事先设置一个焦点窗口 / View，然后由这个焦点窗口 / View 来接受这个事件，否则那么多窗口 / View，我们根本不知道应该把事件发送给谁。对于 Touch 类型的事件，由于这类事件是有坐标的，因此我们可以根据坐标还有其他一些规则，通过对窗口 / View 进行遍历来找到可以接收当前事件的窗口 / VIew，这就不再依赖对焦点窗口 / View 的设置。
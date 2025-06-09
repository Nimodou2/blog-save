> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7140289685879783432)

> 3 向 DisplayArea 层级结构添加窗口 根据之前的分析，我们知道了： 每种窗口类型，都可以通过 WindowManagerPolicy.getWindowLayerFromTypeLw 方法，返回一

根据之前的分析，我们知道了：

*   每种窗口类型，都可以通过 WindowManagerPolicy.getWindowLayerFromTypeLw 方法，返回一个相应的层级值。
    
*   DisplayArea 层级结构中的每一个 DisplayArea，都包含着一个层级值范围，这个层级值范围表明了这个 DisplayArea 可以容纳哪些类型的窗口。
    

那么我们可以合理进行推断添加窗口的一般流程：

WMS 启动的时候添加 DisplayContent 的时候，首先是以该 DisplayContent 为根节点，创建了一个完整的 DisplayArea 层级结构。后续每次添加窗口的时候，都根据该窗口的类型，在 DisplayArea 层级结构中为该窗口寻找一个合适的父节点，然后将这个窗口添加到该父节点之下。

另外根据上面的分析还可以知道，在 DisplayArea 层级结构中，可以直接容纳窗口的父节点，有三种类型：

*   容纳 App 窗口的 TaskDisplayArea
    
*   容纳输入法窗口的 ImeContainer
    
*   容纳其他非 App 类型窗口的 DisplayArea.Tokens
    

这里我们分析一般流程，看下非 App 类型的窗口，如 StatusBar、NavigationBar 等窗口，是如何添加到 DisplayArea 中的。

注意，我们说的每一个窗口，指的是 WindowState。但是 DisplayArea.Tokens 的定义是：

```
    
    public static class Tokens extends DisplayArea<WindowToken> {


```

DisplayArea.Tokens 是 WindowToken 的容器，因此 DisplayArea.Tokens 无法直接添加 WindowState。

WindowToken 则是 WindowState 的容器，每次新 WindowState 创建的时候，都会为这个 WindowState 创建一个 WindowToken 对象，然后将这个新创建的 WindowState 放入其中。

因此我们实际分析的，是 WindowToken 如何添加到 DisplayArea 层级结构中的。

最后需要修正一下说法，因为 WindowToken 不再是一个 DisplayArea，而是一个 WindowContainer。我们之前分析的 DisplayArea 层级结构，其中每一个节点都是一个 DisplayArea，但是现在随着 WindowToken 的加入，DisplayArea 层级结构这个说法需要转化为更通用的结构，也就是 WindowContainer 层级结构。

3.1 WindowManagerService.addWindow
----------------------------------

```
    public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
            int displayId, int requestUserId, InsetsState requestedVisibility,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls) {

        

        synchronized (mGlobalLock) {

            

            WindowToken token = displayContent.getWindowToken(
                    hasParent ? parentWindow.mAttrs.token : attrs.token);

            

            if (token == null) {

                

                if (hasParent) {

                    

                } else {
                    final IBinder binder = attrs.token != null ? attrs.token : client.asBinder();
                    token = new WindowToken.Builder(this, binder, type)
                            .setDisplayContent(displayContent)
                            .setOwnerCanManageAppTokens(session.mCanAddInternalSystemWindow)
                            .setRoundedCornerOverlay(isRoundedCornerOverlay)
                            .build();
                }
            }

            

        }

        

    }



```

调用 WindowToken.Builder.build 创建一个 WindowToken 对象。

3.2 WindowToken.Builder.build
-----------------------------

```
        WindowToken build() {
            return new WindowToken(mService, mToken, mType, mPersistOnEmpty, mDisplayContent,
                    mOwnerCanManageAppTokens, mRoundedCornerOverlay, mFromClientToken, mOptions);
        }


```

3.3 WindowToken.init
--------------------

```
    protected WindowToken(WindowManagerService service, IBinder _token, int type,
            boolean persistOnEmpty, DisplayContent dc, boolean ownerCanManageAppTokens,
            boolean roundedCornerOverlay, boolean fromClientToken, @Nullable Bundle options) {
        super(service);
        token = _token;
        windowType = type;
        mOptions = options;
        mPersistOnEmpty = persistOnEmpty;
        mOwnerCanManageAppTokens = ownerCanManageAppTokens;
        mRoundedCornerOverlay = roundedCornerOverlay;
        mFromClientToken = fromClientToken;
        if (dc != null) {
            dc.addWindowToken(token, this);
        }
    }


```

在 WindowToken 构造方法中，调用 DisplayContent.addWindowToken 将 WindowToken 添加到以 DisplayContent 为根节点的 WindowContainer 层级结构中。

3.4 DisplayContent.addWindowToken
---------------------------------

```
    void addWindowToken(IBinder binder, WindowToken token) {
        final DisplayContent dc = mWmService.mRoot.getWindowTokenDisplay(token);
        

        mTokenMap.put(binder, token);

        if (token.asActivityRecord() == null) {
            
            
            
            
            token.mDisplayContent = this;
            
            
            final DisplayArea.Tokens da = findAreaForToken(token).asTokens();
            da.addChild(token);
        }
    }


```

窗口可以分为 App 窗口和非 App 窗口。对于 App 窗口，则是由更细致的 WindowToken 的子类，ActivityRecord 来存放。这里我们分析非 App 窗口的流程。

分两步走：

1）、调用 DisplayContent.findAreaForToken 为当前 WindowToken 寻找一个合适的父容器，DisplayArea.Tokens 对象。

2）、将 WindowToken 添加到父容器中。

3.5 DisplayContent.findAreaForToken
-----------------------------------

```
    
    DisplayArea findAreaForToken(WindowToken windowToken) {
        return findAreaForWindowType(windowToken.getWindowType(), windowToken.mOptions,
                windowToken.mOwnerCanManageAppTokens, windowToken.mRoundedCornerOverlay);
    }


```

为传入的 WindowToken 找到一个 DisplayArea 对象来添加进去。

3.6 DisplayContent.findAreaForWindowType
----------------------------------------

```
    DisplayArea findAreaForWindowType(int windowType, Bundle options,
            boolean ownerCanManageAppToken, boolean roundedCornerOverlay) {
        
        if (windowType >= FIRST_APPLICATION_WINDOW && windowType <= LAST_APPLICATION_WINDOW) {
            return getDefaultTaskDisplayArea();
        }

        
        
        
        if (windowType == TYPE_INPUT_METHOD || windowType == TYPE_INPUT_METHOD_DIALOG) {
            return getImeContainer();
        }
        return mDisplayAreaPolicy.findAreaForWindowType(windowType, options,
                ownerCanManageAppToken, roundedCornerOverlay);
    }



```

*   如果是 App 窗口，那么返回默认的 TaskDisplayArea 对象。
    
*   如果是输入法窗口，那么返回 ImeContainer。
    
*   如果是其他类型，继续寻找。
    

和我们之前分析的逻辑一致。

3.7 DisplayAreaPolicyBuilder.Result.findAreaForWindowType
---------------------------------------------------------

```
        @Override
        public DisplayArea.Tokens findAreaForWindowType(int type, Bundle options,
                boolean ownerCanManageAppTokens, boolean roundedCornerOverlay) {
            return mSelectRootForWindowFunc.apply(type, options).findAreaForWindowTypeInLayer(type,
                    ownerCanManageAppTokens, roundedCornerOverlay);
        }



```

3.8 RootDisplayArea.findAreaForWindowTypeInLayer
------------------------------------------------

```
    
    @Nullable
    DisplayArea.Tokens findAreaForWindowTypeInLayer(int windowType, boolean ownerCanManageAppTokens,
            boolean roundedCornerOverlay) {
        int windowLayerFromType = mWmService.mPolicy.getWindowLayerFromTypeLw(windowType,
                ownerCanManageAppTokens, roundedCornerOverlay);
        if (windowLayerFromType == APPLICATION_LAYER) {
            throw new IllegalArgumentException(
                    "There shouldn't be WindowToken on APPLICATION_LAYER");
        }
        return mAreaForLayer[windowLayerFromType];
    }



```

计算出该窗口的类型对应的层级值 windowLayerFromType，然后从 mAreaForLayer 数组中，找到 windowLayerFromType 对应的那个 DisplayArea.Tokens 对象。

mAreaForLayer 成员变量，定义为：

```
    
    private DisplayArea.Tokens[] mAreaForLayer;


```

它里面的数据是在 2.3.6 节中填充，保存的是层级值到对应 DisplayArea.Tokens 对象的一个映射。

那么只要传入窗口类型，就可以通过 WindowManagerPolicy.getWindowLayerFromTypeLw 得到该窗口类型对应的层级值，然后根据该层级值从 mAreaForLayer 拿到容纳当前 WindowToken 的父容器，一个 DisplayArea.Tokens 对象。

3.9 DisplayArea.Tokens.addChild
-------------------------------

```
        void addChild(WindowToken token) {
            addChild(token, mWindowComparator);
        }


```

这里调用了 WindowContainer.addChild：

```
    
    @CallSuper
    protected void addChild(E child, Comparator<E> comparator) {
        if (!child.mReparenting && child.getParent() != null) {
            throw new IllegalArgumentException("addChild: container=" + child.getName()
                    + " is already a child of container=" + child.getParent().getName()
                    + " can't add to container=" + getName());
        }

        int positionToAdd = -1;
        if (comparator != null) {
            final int count = mChildren.size();
            for (int i = 0; i < count; i++) {
                if (comparator.compare(child, mChildren.get(i)) < 0) {
                    positionToAdd = i;
                    break;
                }
            }
        }

        if (positionToAdd == -1) {
            mChildren.add(child);
        } else {
            mChildren.add(positionToAdd, child);
        }

        
        child.setParent(this);
    }


```

传入了一个 Comparator：

```
        private final Comparator<WindowToken> mWindowComparator =
                Comparator.comparingInt(WindowToken::getWindowLayerFromType);


```

很明显，在将 WindowToken 添加到父容器的时候，将新添加的 WindowToken 的层级值和父容器中的其他 WindowToken 的层级值进行比较，保证新添加的 WindowToken 在父容器中能够按照层级值的大小插入到合适的位置。
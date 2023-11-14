> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/127551525?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/744428e255474db89d9b0566b9118ff1.jpeg#pic_center)

WindowManager 为 App 提供了一个可以在指定的窗口下插入阴影图层或者模糊背景图层的方法，达到使该窗口之下的所有窗口变暗或者模糊的效果，本文首先分析一下这种效果的大致实现，接着探究一下实现过程中涉及到的相对 Layer 的设置流程。

1 DimLayer
----------

### 1.1 dim 和 blur 介绍

#### 1.1.1 dim

为了将该窗口之下的所有窗口变暗，需要为该窗口设置窗口 flag，FLAG_DIM_BEHIND。

```
/** Window flag: everything behind this window will be dimmed.
         *  Use {@link #dimAmount} to control the amount of dim. */
        public static final int FLAG_DIM_BEHIND        = 0x00000002;
```

并且 App 也可以设置变暗效果的程度：

```
/**
         * When {@link #FLAG_DIM_BEHIND} is set, this is the amount of dimming
         * to apply.  Range is from 1.0 for completely opaque to 0.0 for no
         * dim.
         */
        public float dimAmount = 1.0f;
```

这个参数只有在窗口设置了 FLAG_DIM_BEHIND 的时候才会生效，用来表示窗口变暗的程度，从完全不透明的 1.0 到没有变暗效果的 0。

#### 1.1.2 blur

为了将该窗口之下的所有窗口变模糊，需要为该窗口设置窗口 flag，FLAG_DIM_BEHIND。

```
/** Window flag: enable blur behind for this window. */
        public static final int FLAG_BLUR_BEHIND        = 0x00000004;
```

并且 App 也可以设置模糊效果的程度：

```
/**
         * Specifies the amount of blur to be used to blur everything behind the window.
         * The effect is similar to the dimAmount, but instead of dimming, the content behind
         * will be blurred.
         *
         * The blur behind radius range starts at 0, which means no blur, and increases until 150
         * for the densest blur.
         *
         * @see #setBlurBehindRadius
         */
        private int mBlurBehindRadius = 0;
```

这个参数只有在窗口设置了 FLAG_BLUR_BEHIND 的时候才会生效，用来表示窗口模糊的程度，从没有模糊的 0 到最模糊的 150。

### 1.2 DimLayer 实现

#### 1.2.1 WindowState.prepareSurfaces

```
@Override
    void prepareSurfaces() {
        mIsDimming = false;
        applyDims();
        updateSurfacePositionNonOrganized();
        // Send information to SurfaceFlinger about the priority of the current window.
        updateFrameRateSelectionPriorityIfNeeded();
        if (isVisibleRequested()) updateScaleIfNeeded();

        mWinAnimator.prepareSurfaceLocked(getSyncTransaction());
        super.prepareSurfaces();
    }
```

DimLayer 的添加是在 WindowSetate.prepareSurfaces 中，在 [WMS](https://so.csdn.net/so/search?q=WMS&spm=1001.2101.3001.7020) 向 SurfaceFlinger 提交 Transaction 之前。

#### 1.2.2 WindowState.applyDims

```
private void applyDims() {
        if (!mAnimatingExit && mAppDied) {
            mIsDimming = true;
            getDimmer().dimAbove(getSyncTransaction(), this, DEFAULT_DIM_AMOUNT_DEAD_WINDOW);
        } else if (((mAttrs.flags & FLAG_DIM_BEHIND) != 0 || shouldDrawBlurBehind())
                   && isVisibleNow() && !mHidden) {
            // Only show the Dimmer when the following is satisfied:
            // 1. The window has the flag FLAG_DIM_BEHIND or blur behind is requested
            // 2. The WindowToken is not hidden so dims aren't shown when the window is exiting.
            // 3. The WS is considered visible according to the isVisible() method
            // 4. The WS is not hidden.
            mIsDimming = true;
            final float dimAmount = (mAttrs.flags & FLAG_DIM_BEHIND) != 0 ? mAttrs.dimAmount : 0;
            final int blurRadius = shouldDrawBlurBehind() ? mAttrs.getBlurBehindRadius() : 0;
            getDimmer().dimBelow(getSyncTransaction(), this, dimAmount, blurRadius);
        }
    }
```

Dimmer 类的定义是：

```
/**
 * Utility class for use by a WindowContainer implementation to add "DimLayer" support, that is
 * black layers of varying opacity at various Z-levels which create the effect of a Dim.
 */
class Dimmer {
```

实用程序类，用于 WindowContainer 实现添加 “DimLayer” 支持，DimLayer 是在多种 Z 轴级别上的不同不透明度的黑色层，可以显示一个 Dim 效果。

这里说明了 Dimmer 要显示的情形需要满足的条件：

1）、相关窗口必须声明 FLAG_DIM_BEHIND 或者 FLAG_BLUR_BEHIND。

2）、WindowToken 没有隐藏，防止 Dimmer 不会在窗口正在退出的时候显示。

3）、根据 isVisible 方法，当前 WindowState 可见。

4）、当前 WindowState 没有隐藏。

如果以上情况均满足，那么调用 Dimmer.dimBelow。

#### 1.2.3 Dimmer.dimBelow

```
/**
     * Like {@link #dimAbove} but places the dim below the given container.
     *
     * @param t          A transaction in which to apply the Dim.
     * @param container  The container which to dim below. Should be a child of our host.
     * @param alpha      The alpha at which to Dim.
     * @param blurRadius The amount of blur added to the Dim.
     */

    void dimBelow(SurfaceControl.Transaction t, WindowContainer container, float alpha,
                  int blurRadius) {
        dim(t, container, -1, alpha, blurRadius);
    }
```

参数为：

*   container，需要为处于其下的容器施加阴影效果的那个容器，应该是当前 Dimmer 的 host 的子容器。
*   alpha，施加变暗效果的时候，需要设置的 alpha 值。
*   blurRadius，施加模糊效果的时候，需要设置的模糊半径。

这里看一下 Dimmer 的 host 是什么。

成员变量 mHost 的定义为：

```
/**
     * The {@link WindowContainer} that our Dim's are bounded to. We may be dimming on behalf of the
     * host, some controller of it, or one of the hosts children.
     */
    private WindowContainer mHost;
```

Dimmer 绑定的那个 WindowContainer，我们可以对 host，host 的一些控制器，或者是 host 的子容器施加 Dim 效果。

mHost 在 Dimmer 的构造方法中传入：

```
Dimmer(WindowContainer host, SurfaceAnimatorStarter surfaceAnimatorStarter) {
        mHost = host;
        mSurfaceAnimatorStarter = surfaceAnimatorStarter;
    }
```

目前 Dimmer 对象创建的地方主要有两处：

1）、Task 中：

```
private Dimmer mDimmer = new Dimmer(this);
```

2）、DisplayArea 的子类 Dimmable 中：

```
/**
     * DisplayArea that can be dimmed.
     */
    static class Dimmable extends DisplayArea<DisplayArea> {
        private final Dimmer mDimmer = new Dimmer(this);
```

#### 1.2.4 Dimmer.dim

```
private void dim(SurfaceControl.Transaction t, WindowContainer container, int relativeLayer,
            float alpha, int blurRadius) {
        final DimState d = getDimState(container);

        if (d == null) {
            return;
        }

        if (container != null) {
            // The dim method is called from WindowState.prepareSurfaces(), which is always called
            // in the correct Z from lowest Z to highest. This ensures that the dim layer is always
            // relative to the highest Z layer with a dim.
            t.setRelativeLayer(d.mDimLayer, container.getSurfaceControl(), relativeLayer);
        } else {
            t.setLayer(d.mDimLayer, Integer.MAX_VALUE);
        }
        t.setAlpha(d.mDimLayer, alpha);
        t.setBackgroundBlurRadius(d.mDimLayer, blurRadius);

        d.mDimming = true;
    }
```

这里看到是通过为 DimState.mDimLayer 设置透明度和背景模糊半径来达到变暗和模糊的效果：

```
t.setAlpha(d.mDimLayer, alpha);
        t.setBackgroundBlurRadius(d.mDimLayer, blurRadius);
```

这里涉及到了 relativeLayer 的部分，这个放到后续分析，这里主要看一下 DimState.mDimLayer 是如何得到的。

在进行透明度等设置之前，首先是调用了 getDimState 方法。

#### 1.2.5 Dimmer.getDimState

```
/**
     * Retrieve the DimState, creating one if it doesn't exist.
     */
    private DimState getDimState(WindowContainer container) {
        if (mDimState == null) {
            try {
                final SurfaceControl ctl = makeDimLayer();
                mDimState = new DimState(ctl);
                /**
                 * See documentation on {@link #dimAbove} to understand lifecycle management of
                 * Dim's via state resetting for Dim's with containers.
                 */
                if (container == null) {
                    mDimState.mDontReset = true;
                }
            } catch (Surface.OutOfResourcesException e) {
                Log.w(TAG, "OutOfResourcesException creating dim surface");
            }
        }

        mLastRequestedDimContainer = container;
        return mDimState;
    }
```

这个方法主要做了两个工作：

1）、调用 makeDimLayer 创建一个 Layer：

```
private SurfaceControl makeDimLayer() {
        return mHost.makeChildSurface(null)
                .setParent(mHost.getSurfaceControl())
                .setColorLayer()
                .setName("Dim Layer for - " + mHost.getName())
                .setCallsite("Dimmer.makeDimLayer")
                .build();
    }
```

*   parent 设置为 host 的 SurfaceControl，一般即 Task 的 SurfaceControl。
*   设置该 Layer 是一个纯色 Layer，默认颜色是黑色。
*   设置该 Layer 的名字为 “Dim Layer for -” 加上 host 的名字。

2）、创建了一个纯色 Layer 后，再创建一个 DimState 对象，将该 Layer 保存在 DimState.mDimLayer 中：

```
DimState(SurfaceControl dimLayer) {
            mDimLayer = dimLayer;
            mDimming = true;
            final DimAnimatable dimAnimatable = new DimAnimatable(dimLayer);
            mSurfaceAnimator = new SurfaceAnimator(dimAnimatable, (type, anim) -> {
                if (!mDimming) {
                    dimAnimatable.removeSurface();
                }
            }, mHost.mWmService);
        }
```

这样 Dimmer 可以通过 getDimState 方法获取到一个 DimState 对象，进而拿到一个 DimLayer，Dimmer 对象，DimState，DimLayer，这三者是一一对应的。

这样 Dimmer.getDimState 返回一个纯色 Layer 对应的 SurfaceControl，后续就可以对其设置 alpha 和 backgroundBlurRadius 等属性，施加变暗和模糊效果。

### 1.3 小结

![](https://img-blog.csdnimg.cn/1b255f58f2714d63abab2ccb4517eee9.png#pic_center)

1）、DimLayer 其实就是一个 SurfaceControl，其对应的 Layer 是一个纯黑 Layer，可以通过为其设置透明度来施加变暗效果，或者设置背景模糊半径来施加模糊效果。

2）、持有 Dimmer 对象的有 Task 和 DisplayArea.Dimmable，这个 Dimmer 对象可以被它们的子容器通过 WindowContainer.getDimme[r 方](https://so.csdn.net/so/search?q=r%E6%96%B9&spm=1001.2101.3001.7020)法拿到。

3）、Dimmer、DimState 和 DimLayer 三者是一一对应的关系。

2 relativeLayer
---------------

可以实现 Dim 效果的 DimLayer 已经有了，那么我们要把这个 DimLayer 插入到哪个位置来实现 dimBelow 或者 dimAbove 的效果呢？

这里拿 dimBelow 的情况说明，回看 1.2.4 节的 Dimmer.dim 方法的内容：

```
if (container != null) {
            // The dim method is called from WindowState.prepareSurfaces(), which is always called
            // in the correct Z from lowest Z to highest. This ensures that the dim layer is always
            // relative to the highest Z layer with a dim.
            t.setRelativeLayer(d.mDimLayer, container.getSurfaceControl(), relativeLayer);
        } else {
            t.setLayer(d.mDimLayer, Integer.MAX_VALUE);
        }
```

这里的 container 不为 null，且为想要将其下的所有窗口都 Dim 的那个窗口，relativeLayer 则是 - 1，跟一下 setRelativeLayer 是怎么实现的。

### 2.1 Transaction.setRelativeLayer

```
/**
         * @hide
         */
        public Transaction setRelativeLayer(SurfaceControl sc, SurfaceControl relativeTo, int z) {
            checkPreconditions(sc);
            nativeSetRelativeLayer(mNativeObject, sc.mNativeObject, relativeTo.mNativeObject, z);
            return this;
        }
```

调到了 JNI。

### 2.2 android_view_SurfaceControl.nativeSetRelativeLayer

```
static void nativeSetRelativeLayer(JNIEnv* env, jclass clazz, jlong transactionObj,
        jlong nativeObject,
        jlong relativeToObject, jint zorder) {

    auto ctrl = reinterpret_cast<SurfaceControl *>(nativeObject);
    auto relative = reinterpret_cast<SurfaceControl *>(relativeToObject);
    auto transaction = reinterpret_cast<SurfaceComposerClient::Transaction*>(transactionObj);
    transaction->setRelativeLayer(ctrl, relative, zorder);
}
```

将 Java 层传入的对象转为 Native 层对应的对象，然后调用 Transaction.setRelativeLayer。

### 2.3 SurfaceComposerClient.Transaction.setRelativeLayer

```
SurfaceComposerClient::Transaction& SurfaceComposerClient::Transaction::setRelativeLayer(
        const sp<SurfaceControl>& sc, const sp<SurfaceControl>& relativeTo, int32_t z) {
    layer_state_t* s = getLayerState(sc);
    if (!s) {
        mStatus = BAD_INDEX;
        return *this;
    }
    s->what |= layer_state_t::eRelativeLayerChanged;
    s->what &= ~layer_state_t::eLayerChanged;
    s->relativeLayerSurfaceControl = relativeTo;
    s->z = z;

    registerSurfaceControlForCallback(sc);
    return *this;
}
```

1）、为 layer_state_s 的 what 添加 layer_state_t::eRelativeLayerChanged 标记，表示是该 Layer 的相对 Layer 发生了改变。

2）、将相对 Layer 和 Z 轴信息保存在 layer_state_t 类型的 ComposerState.state 中。

### 2.4 [SurfaceFlinger](https://so.csdn.net/so/search?q=SurfaceFlinger&spm=1001.2101.3001.7020).setClientStateLocked

后续 Transaction 应用，最终来到了 SurfaceFlinger.setClientStateLocked 函数。

由于 2.3 节中为 layer_state_t 的 what 添加了 layer_state_t::eRelativeLayerChanged 标记，那么在这个函数中走的逻辑是：

```
if (what & layer_state_t::eRelativeLayerChanged) {
        // NOTE: index needs to be calculated before we update the state
        const auto& p = layer->getParent();
        const auto& relativeHandle = s.relativeLayerSurfaceControl ?
                s.relativeLayerSurfaceControl->getHandle() : nullptr;
        if (p == nullptr) {
            ssize_t idx = mCurrentState.layersSortedByZ.indexOf(layer);
            if (layer->setRelativeLayer(relativeHandle, s.z) &&
                idx >= 0) {
                mCurrentState.layersSortedByZ.removeAt(idx);
                mCurrentState.layersSortedByZ.add(layer);
                // we need traversal (state changed)
                // AND transaction (list changed)
                flags |= eTransactionNeeded|eTraversalNeeded;
            }
        } else {
            if (p->setChildRelativeLayer(layer, relativeHandle, s.z)) {
                flags |= eTransactionNeeded|eTraversalNeeded;
            }
        }
    }
```

这里的 layer 即之前 1.2.5 节中介绍的通过 Dimmer.makeDimLayer 方法创建的那个 SurfaceControl 对应的 Layer，当时设置了该 Layer 的父 Layer 为 Task 对应的 Layer，那么此时这里的 p 就不为空，走的是 Layer.setChildRelativeLayer 函数。

### 2.5 Layer.setChildRelativeLayer

```
bool Layer::setChildRelativeLayer(const sp<Layer>& childLayer,
        const sp<IBinder>& relativeToHandle, int32_t relativeZ) {
    ssize_t idx = mCurrentChildren.indexOf(childLayer);
    if (idx < 0) {
        return false;
    }
    if (childLayer->setRelativeLayer(relativeToHandle, relativeZ)) {
        mCurrentChildren.removeAt(idx);
        mCurrentChildren.add(childLayer);
        return true;
    }
    return false;
}
```

1）、首先判断 childLayer 是否在 mCurrentChildren 中。

2）、调用 childLayer 的 setRelativeLayer 函数。

3）、将 childLayer 从 mCurrentChildren 中原来的位置移除，然后重新添加进来。

### 2.6 Layer.setRelativeLayer

```
bool Layer::setRelativeLayer(const sp<IBinder>& relativeToHandle, int32_t relativeZ) {
    sp<Handle> handle = static_cast<Handle*>(relativeToHandle.get());
    if (handle == nullptr) {
        return false;
    }
    sp<Layer> relative = handle->owner.promote();
    if (relative == nullptr) {
        return false;
    }

    if (mDrawingState.z == relativeZ && usingRelativeZ(LayerVector::StateSet::Current) &&
        mDrawingState.zOrderRelativeOf == relative) {
        return false;
    }

    mFlinger->mSomeChildrenChanged = true;

    mDrawingState.sequence++;
    mDrawingState.modified = true;
    mDrawingState.z = relativeZ;

    auto oldZOrderRelativeOf = mDrawingState.zOrderRelativeOf.promote();
    if (oldZOrderRelativeOf != nullptr) {
        oldZOrderRelativeOf->removeZOrderRelative(this);
    }
    setZOrderRelativeOf(relative);
    relative->addZOrderRelative(this);

    setTransactionFlags(eTransactionNeeded);

    return true;
}
```

1）、将从 SurfaceComposerClient 处传来的 Layer 句柄转为相对 Layer。判断相对 Layer 的有效性，以及是否做了重复工作。

2）、设置当前 Layer 的 Z 轴大小为传入的 relativeZ，注意这里直接把 relativeZ 作为了当前 Layer 在父 Layer 中的 Z 轴顺序，并没有一个相对 Z 轴顺序的概念。

3）、对当前 Layer 调用 Layer.setZOrderRelativeOf。

4）、对相对 Layer 调用 Layer.addZOrderRelative。

#### 2.6.1 Layer.setZOrderRelativeOf

```
void Layer::setZOrderRelativeOf(const wp<Layer>& relativeOf) {
    mDrawingState.zOrderRelativeOf = relativeOf;
    mDrawingState.sequence++;
    mDrawingState.modified = true;
    mDrawingState.isRelativeOf = relativeOf != nullptr;

    setTransactionFlags(eTransactionNeeded);
}
```

对于当前 Layer，设置以下两个成员变量的值：

```
// If non-null, a Surface this Surface's Z-order is interpreted relative to.
        wp<Layer> zOrderRelativeOf;
        bool isRelativeOf{false};
```

zOrderRelativeOf，如果非空，当前 Surface 的 Z 轴将被解释为相对于 zOrderRelativeOf 的 Z 轴。

isRelativeOf，一个表示当前 Layer 是否设置了相对 Layer。

#### 2.6.2 Layer.addZOrderRelative

```
void Layer::addZOrderRelative(const wp<Layer>& relative) {
    mDrawingState.zOrderRelatives.add(relative);
    mDrawingState.modified = true;
    mDrawingState.sequence++;
    setTransactionFlags(eTransactionNeeded);
}
```

对于相对 Layer，将当前 Layer 加入到相对 Layer 的 zOrderRelatives 中，zOrderRelatives 的定义是：

```
// A list of surfaces whose Z-order is interpreted relative to ours.
        SortedVector<wp<Layer>> zOrderRelatives;
```

一个 Surface 的列表，列表中的 Surface 的 Z 轴顺序都被解释为相对于我们的 Z 轴顺序。

截止到这里，setRelativeLayer 的工作算是做完了，接着看下相对 Layer 是如何发挥作用的。

### 2.7 Layer.traverseInZOrder

在 Layer 的遍历函数中探究一下：

```
/**
 * Negatively signed relatives are before 'this' in Z-order.
 */
void Layer::traverseInZOrder(LayerVector::StateSet stateSet, const LayerVector::Visitor& visitor) {
    // In the case we have other layers who are using a relative Z to us, makeTraversalList will
    // produce a new list for traversing, including our relatives, and not including our children
    // who are relatives of another surface. In the case that there are no relative Z,
    // makeTraversalList returns our children directly to avoid significant overhead.
    // However in this case we need to take the responsibility for filtering children which
    // are relatives of another surface here.
    bool skipRelativeZUsers = false;
    const LayerVector list = makeTraversalList(stateSet, &skipRelativeZUsers);

    // ......
}
```

这里主要说明了两种情况：

*   如果有其他 Layer 以我们为参照使用了相对 Z 轴顺序（也就是说我们作为了这些 Layer 的相对 Layer），那么 makeTraversalList 将会为遍历产生一个新的列表，包括以我们为参照的 Layer，但是不包括我们的子 Layer 中使用了相对 Layer 的 Layer。
*   对于没有相对 Layer 的情况，makeTraversalList 直接返回我们的子 Layer 来避免更多的开销。然而在这种情况下，我们需要负起责任来在这里过滤掉那些使用了相对层级的子 Layer。

直接看下 Layer.makeTraversalList 的内容：

```
__attribute__((no_sanitize("unsigned-integer-overflow"))) LayerVector Layer::makeTraversalList(
        LayerVector::StateSet stateSet, bool* outSkipRelativeZUsers) {
    LOG_ALWAYS_FATAL_IF(stateSet == LayerVector::StateSet::Invalid,
                        "makeTraversalList received invalid stateSet");
    const bool useDrawing = stateSet == LayerVector::StateSet::Drawing;
    const LayerVector& children = useDrawing ? mDrawingChildren : mCurrentChildren;
    const State& state = useDrawing ? mDrawingState : mDrawingState;

    if (state.zOrderRelatives.size() == 0) {
        *outSkipRelativeZUsers = true;
        return children;
    }

    LayerVector traverse(stateSet);
    for (const wp<Layer>& weakRelative : state.zOrderRelatives) {
        sp<Layer> strongRelative = weakRelative.promote();
        if (strongRelative != nullptr) {
            traverse.add(strongRelative);
        }
    }

    for (const sp<Layer>& child : children) {
        if (child->usingRelativeZ(stateSet)) {
            continue;
        }
        traverse.add(child);
    }

    return traverse;
}
```

这个函数返回一个遍历列表。

1）、如果 zOrderRelatives 的 size 为 0，表示没有 Layer 将我们视为相对 Layer，那么直接返回子 Layer 列表。

2）、接着遍历 zOrderRelatives，zOrderRelatives 保存了设置我们为相对 Layer 的所有 Layer，也就是说这些 Layer 想作为我们的子 Layer 插入到我们的 Layer 层级结构之中，那么将这些 Layer 加入到遍历列表中。

3）、最后遍历子 Layer 列表，usingRelativeZ 函数返回 true，表示当前 Layer 使用了相对层级，那么这些 Layer 将会被过滤，当它们的相对 Layer 进行 makeTraversalList 操作的时候，这些子 Layer 将会被添加到它们的相对 Layer 的遍历列表中。

再次返回 Layer.traverseInZOrder：

```
void Layer::traverseInZOrder(LayerVector::StateSet stateSet, const LayerVector::Visitor& visitor) {
    // ......
    bool skipRelativeZUsers = false;
    const LayerVector list = makeTraversalList(stateSet, &skipRelativeZUsers);

    size_t i = 0;
    for (; i < list.size(); i++) {
        const auto& relative = list[i];
        if (skipRelativeZUsers && relative->usingRelativeZ(stateSet)) {
            continue;
        }

        if (relative->getZ(stateSet) >= 0) {
            break;
        }
        relative->traverseInZOrder(stateSet, visitor);
    }

    visitor(this);
    for (; i < list.size(); i++) {
        const auto& relative = list[i];

        if (skipRelativeZUsers && relative->usingRelativeZ(stateSet)) {
            continue;
        }
        relative->traverseInZOrder(stateSet, visitor);
    }
}
```

后续对从 makeTraversalList 返回的 list 进行两次遍历，从 makeTraversalList 的逻辑可知，该 list 前部分是相对 Layer 的部分（如果有的话），后部分是当前 Layer 的子 Layer。

1）、第一次是先对 list 中 Z 轴顺序小于 0 的 Layer 进行遍历，只有相对 Layer 才能设置 Z 轴顺序为负数，子 Layer 的 Z 轴顺序都是大于等于 0 的。

2）、第二次是对 list 中 Z 轴顺序大于 0 的 Layer 进行遍历。

### 2.8 小结

再次看下 Transaction.setRelativeLayer 方法：

```
public Transaction setRelativeLayer(SurfaceControl sc, SurfaceControl relativeTo, int z) {
            checkPreconditions(sc);
            nativeSetRelativeLayer(mNativeObject, sc.mNativeObject, relativeTo.mNativeObject, z);
            return this;
        }
```

再次分析一下 setRelativeLayer 方法的三个参数：

*   sc，设置了相对 Layer 的那个 Layer，叫他 LayerA。
*   relativeTo，相对 Layer，是一个参照物的角色，叫他 LayerB。
*   z，LayerA 的相对 Z 轴顺序。

根据 2.6 节我们得知：

1）、LayerA 的 isRelativeOf 被标记为 true，那么 LayerA 的父 Layer 后续对其 Layer 层级结构进行遍历的时候，就会过滤掉 LayerA。

2）、LayerA 被添加到了 LayerB 的 zOrderRelatives 中，那么 LayerB 在遍历其 Layer 层级结构的时候，就会把 LayerA 看作一个自己的子 Layer。

那么 setRelativeLayer 实现的作用是，把 LayerA 从 LayerA 所在的 Layer 层级结构中剥离，并且作为 LayerB 的子 Layer 插入到以 LayerB 为父节点的 Layer 层级结构中，传参 z 即 LayerA 在新的 LayerB 层级结构中的 Z 轴顺序。

那么再回看 Dimmer.dimBelow 的作用，就是创建一个 DimLayer，然后将其作为子 Layer 插入到某个 WindowState 对应的 Layer 层级结构中。

由于该 DimLayer 处于该 WindowState 对应的 Layer 的层级结构中，那么该 DimLayer 会 Dim 所有层级比该 WindowState 层级低的 Layer，达到将该 WindowState 之下的所有窗口变暗或者模糊的效果。

3 实际验证
------

这里根据 SurfaceFlinger 打印出的信息来验证一下之前分析的内容都是否正确。

拿 Google Files 进行一下验证，Files 显示如下：

![](https://img-blog.csdnimg.cn/2406ba40613344d9bf37e0d48d148433.png#pic_center)

此时 Files 有两个窗口：

```
Window #8 Window{f19c0b8 u0 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity}:
    mDisplayId=0 rootTaskId=36 mSession=Session{4616615 14439:u0a10155} mClient=android.os.BinderProxy@a81971b
    mOwnerUid=10155 showForAllUsers=false package=com.google.android.apps.nbu.files appop=NONE
    mAttrs={(0,0)(fillxfill) gr=CENTER sim={adjust=pan forwardNavigation} ty=APPLICATION fmt=TRANSPARENT wanim=0x7f140007 surfaceInsets=Rect(64, 64 - 64, 64)
      fl=DIM_BEHIND LAYOUT_IN_SCREEN LAYOUT_INSET_DECOR SPLIT_TOUCH HARDWARE_ACCELERATED DRAWS_SYSTEM_BAR_BACKGROUNDS
  Window #9 Window{c036971 u0 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity}:
    mDisplayId=0 rootTaskId=36 mSession=Session{4616615 14439:u0a10155} mClient=android.os.BinderProxy@c43fb18
    mOwnerUid=10155 showForAllUsers=false package=com.google.android.apps.nbu.files appop=NONE
    mAttrs={(0,0)(fillxfill) sim={adjust=resize forwardNavigation} ty=BASE_APPLICATION wanim=0x10302fe
      fl=LAYOUT_IN_SCREEN LAYOUT_INSET_DECOR SPLIT_TOUCH HARDWARE_ACCELERATED DRAWS_SYSTEM_BAR_BACKGROUNDS
```

*   Window#8，type 为 APPLICATION，子窗口。
    
*   Window#9，type 为 BASE_APPLICATION，主窗口。
    

由于 Window#8 声明了 DIM_BEHIND，并且 Window#8 层级是高于 Window#9 的，那么就会在 Window#9 上盖上一层阴影，即上图显示的效果。

### 3.1 相对 Layer 为 Window#8，相对 Layer 值为 - 1

此时的 Dim 逻辑是

```
dim(t, container, -1, alpha, blurRadius);
```

此处的 container 为 Window#8，相对 Layer 为 Window#8 对应的 Layer，相对 Layer 值为 - 1。

再看下 SurfaceFlinger 的信息：

```
+ EffectLayer (Task=36#0) uid=1000
      layerStack=   0, z=       12
      parent=DefaultTaskDisplayArea#0
      zOrderRelativeOf=none

+ ContainerLayer (ActivityRecord{a555aba u0 com.google.android.apps.nbu.files/.home.HomeActivity t36}#0) uid=1000
      layerStack=   0, z=        0
      parent=Task=36#0
      zOrderRelativeOf=none

+ ContainerLayer (c036971 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=1000
      layerStack=   0, z=        0
      parent=ActivityRecord{a555aba u0 com.google.android.apps.nbu.files/.home.HomeActivity t36}#0
      zOrderRelativeOf=none

+ BufferStateLayer (com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=10155
      layerStack=   0, z=        0
      parent=c036971 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
      zOrderRelativeOf=none

+ EffectLayer (Dim Layer for - Task=36#0) uid=1000
      layerStack=   0, z=       -1
      parent=Task=36#0
      zOrderRelativeOf=f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0

+ ContainerLayer (f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=1000
      layerStack=   0, z=        1
      parent=ActivityRecord{a555aba u0 com.google.android.apps.nbu.files/.home.HomeActivity t36}#0
      zOrderRelativeOf=none

+ BufferStateLayer (com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#1) uid=10155
      layerStack=   0, z=        0
      parent=f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
      zOrderRelativeOf=none
```

转为更加直观的形式：

![](https://img-blog.csdnimg.cn/61606d2c8588480e9eeab7d85edcda45.png#pic_center)

SurfaceFlinger 打印的信息的规则是，越靠下的 Layer 层级值越高。

这里唯一不太明朗的是 DimLayer：

```
EffectLayer (Dim Layer for - Task=36#0) uid=1000
```

的位置比较奇怪：

![](https://img-blog.csdnimg.cn/7e531c1020904dceb6c81e9b6daf3d2c.png#pic_center)

它的直接父 Layer 是 Window#8 对应的 Layer：

```
ContainerLayer (f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=1000
```

所以我预想中的层级顺序应该是：

```
+ ContainerLayer (f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=1000
      layerStack=   0, z=        1
      parent=ActivityRecord{a555aba u0 com.google.android.apps.nbu.files/.home.HomeActivity t36}#0
      zOrderRelativeOf=none

+ EffectLayer (Dim Layer for - Task=36#0) uid=1000
      layerStack=   0, z=       -1
      parent=Task=36#0
      zOrderRelativeOf=f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0

+ BufferStateLayer (com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#1) uid=10155
      layerStack=   0, z=        0
      parent=f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
      zOrderRelativeOf=none
```

即：

![](https://img-blog.csdnimg.cn/1103644297c54a67adba0241ee386125.png#pic_center)

毕竟子 Layer 应该处于父 Layer 的下面，但是根据之前在 Layer.traverseInZOrder 看到的一段注释：

```
/**
 * Negatively signed relatives are before 'this' in Z-order.
 */
```

那么也可以理解，SurfaceFlinger 的信息打印规则是，打印的位置越靠下层级值越高。

以父 Layer 为基础：

*   Z 轴顺序为正，打印在父 Layer 下面，这些是正常 Layer。
*   Z 轴顺序为负，打印在父 Layer 下面，层级比正常 Layer 都要低。

说实话，这里画的图也是基于之前的代码分析得到的，但是如果没有分析过代码，只看 SurfaceFlinger 的信息，还是难以确认 DimLayer 的直接父 Layer 是谁，我们还是需要一个更加强力的证据来支撑的我们的分析结果，即某个 Layer 设置了相对 Layer 之后，它的父 Layer 就变为了这个相对 Layer。

### 3.2 相对 Layer 为 Window#8，相对 Layer 值为 100

修改 Dim 逻辑，将相对 Layer 值从 - 1 改为 100：

```
dim(t, container, 100, alpha, blurRadius);
```

那么根据我们之前分析的结果，此时 DimLayer 仍然处于 Window#8 对应的 Layer 的 Layer 层级结构中，DimLayer 的层级值从 - 1 变为 100 后，DimLayer 层级值最高，能够盖住所有的窗口。

此时的 Files 表现为：

![](https://img-blog.csdnimg.cn/852ff280ac004f1184905aa05e748ce8.png#pic_center)

和我们预想的情况一致。

再看下 SurfaceFlinger 的情况：

```
+ EffectLayer (Task=9#0) uid=1000
      layerStack=   0, z=        5
      parent=DefaultTaskDisplayArea#0
      zOrderRelativeOf=none

+ ContainerLayer (ActivityRecord{a401f58 u0 com.google.android.apps.nbu.files/.home.HomeActivity t9}#0) uid=1000
      layerStack=   0, z=        0
      parent=Task=9#0
      zOrderRelativeOf=none

+ ContainerLayer (35b4d1 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=1000
      layerStack=   0, z=        0
      parent=ActivityRecord{a401f58 u0 com.google.android.apps.nbu.files/.home.HomeActivity t9}#0
      zOrderRelativeOf=none

+ BufferStateLayer (com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#1) uid=10156
      layerStack=   0, z=        0
      parent=35b4d1 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
      zOrderRelativeOf=none

+ ContainerLayer (be13be5 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=1000
      layerStack=   0, z=        1
      parent=ActivityRecord{a401f58 u0 com.google.android.apps.nbu.files/.home.HomeActivity t9}#0
      zOrderRelativeOf=none

+ BufferStateLayer (com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=10156
      layerStack=   0, z=        0
      parent=be13be5 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
      zOrderRelativeOf=none
      zOrderRelativeOf=none

+ EffectLayer (Dim Layer for - Task=9#0) uid=1000
      layerStack=   0, z=      100
      parent=Task=9#0
      zOrderRelativeOf=be13be5 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
```

转为图：

![](https://img-blog.csdnimg.cn/2a0ba7f6dd864fa792b266baa48e40c1.png#pic_center)

能够看到 DimLayer：

```
EffectLayer (Dim Layer for - Task=9#0)
```

层级变为了最高盖在了其他窗口之上，导致所有窗口被施加了一层变暗效果。

但是如果只看 SurfaceFlinger 的信息的话，应该还是看不出来，此时 DimLayer 的直接父 Layer 可能是

```
parent=Task=9#0
```

也可能是

```
zOrderRelativeOf=be13be5 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
```

比如我也可以说此时 DimLayer 的直接父 Layer 是：

```
parent=Task=9#0
```

层级关系图为：

![](https://img-blog.csdnimg.cn/443245453f5044d2949c1fd3585fd474.png#pic_center)

不管是它们两者中的哪一个，打印出来的信息都是一样的，显示的效果也是一样的，所以需要继续分析。

### 3.3 相对 Layer 为 Window#9，相对 Layer 值为 100

修改 Dim 逻辑，将相对 Layer 从 Window#8 替换为 Window#9，同时相对 Layer 值仍然为 100。

那么根据我们之前分析的结果，此时 DimLayer 处于 Window#9 对应的 Layer 的 Layer 层级结构中，DimLayer 的层级值为 100，那么 DimLayer 只能盖住层级值比 Window#9 低的窗口，Window#8 是盖不住的。

此时的 Files 表现为：

![](https://img-blog.csdnimg.cn/16280d265d234dae970ae4eb1e9bd34d.png#pic_center)

和我们预想的情况一致。

再看下 SurfaceFlinger 的情况：

```
+ EffectLayer (Task=12#0) uid=1000
      layerStack=   0, z=        5
      parent=DefaultTaskDisplayArea#0
      zOrderRelativeOf=none

+ ContainerLayer (ActivityRecord{7878ca6 u0 com.google.android.apps.nbu.files/.home.HomeActivity t12}#0) uid=1000
      layerStack=   0, z=        0
      parent=Task=12#0
      zOrderRelativeOf=none

+ ContainerLayer (c50b695 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=1000
      layerStack=   0, z=        0
      parent=ActivityRecord{7878ca6 u0 com.google.android.apps.nbu.files/.home.HomeActivity t12}#0
      zOrderRelativeOf=none

+ BufferStateLayer (com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#1) uid=10156
      layerStack=   0, z=        0
      parent=c50b695 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
      zOrderRelativeOf=none

+ EffectLayer (Dim Layer for - Task=12#0) uid=1000
      layerStack=   0, z=      100
      parent=Task=12#0
      zOrderRelativeOf=c50b695 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0

+ ContainerLayer (83cf2f0 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=1000
      layerStack=   0, z=        1
      parent=ActivityRecord{7878ca6 u0 com.google.android.apps.nbu.files/.home.HomeActivity t12}#0
      zOrderRelativeOf=none

+ BufferStateLayer (com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=10156
      layerStack=   0, z=        0
      parent=83cf2f0 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
      zOrderRelativeOf=none
```

转为图：

![](https://img-blog.csdnimg.cn/dd7213b170c8402a81c2e1d4f2f019de.png#pic_center)

果然，此时的 Dimlayer：

```
EffectLayer (Dim Layer for - Task=12#0)
```

被插入到了 Window#9 对应的 Layer：

```
ContainerLayer (c50b695 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0)
```

的 Layer 层级结构中，所以它能够盖住 Window#9，即主窗口，却盖不住子窗口 Window#8，所以显示结果如上图所示那样。

这里就绝对能说明，此时 DimLayer 的父 Layer 肯定不再是

```
parent=Task=12#0
```

而是

```
ContainerLayer (c50b695 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0)
```

### 3.4 相对 Layer 为 Window#9，相对 Layer 值为 - 1

这里最后看下，相对 Layer 为 Window#9，把相对 Layer 值从 100 恢复为 - 1 的情况。

那么根据我们之前分析的结果，此时 DimLayer 处于 Window#9 对应的 Layer 的 Layer 层级结构中，DimLayer 的层级值为 - 1，那么 DimLayer 连 Window#9 对应的窗口也无法盖住。

此时的 Files 表现为：

![](https://img-blog.csdnimg.cn/db2cea2e1cbf427395773a387e693f3d.png#pic_center)

果然如此，查看 SurfaceFlinger 信息：

```
+ EffectLayer (Task=15#0) uid=1000
      layerStack=   0, z=        5
      parent=DefaultTaskDisplayArea#0
      zOrderRelativeOf=none

+ ContainerLayer (ActivityRecord{40c40f6 u0 com.google.android.apps.nbu.files/.home.HomeActivity t15}#0) uid=1000
      layerStack=   0, z=        0
      parent=Task=15#0
      zOrderRelativeOf=none

+ EffectLayer (Dim Layer for - Task=15#0) uid=1000
      layerStack=   0, z=       -1
      parent=Task=15#0
      zOrderRelativeOf=84626fe com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0

+ ContainerLayer (84626fe com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=1000
      layerStack=   0, z=        0
      parent=ActivityRecord{40c40f6 u0 com.google.android.apps.nbu.files/.home.HomeActivity t15}#0
      zOrderRelativeOf=none

+ BufferStateLayer (com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#1) uid=10156
      layerStack=   0, z=        0
      parent=84626fe com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
      zOrderRelativeOf=none

+ ContainerLayer (9be415a com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=1000
      layerStack=   0, z=        1
      parent=ActivityRecord{40c40f6 u0 com.google.android.apps.nbu.files/.home.HomeActivity t15}#0
      zOrderRelativeOf=none

+ BufferStateLayer (com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0) uid=10156
      layerStack=   0, z=        0
      parent=9be415a com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
      zOrderRelativeOf=none
```

转为图：

![](https://img-blog.csdnimg.cn/ffdd5ecb0b2e499b857a257984d05b08.png#pic_center)

由于 DimLayer 现在在 Window#9 对应的 Layer 层级结构中，且层级为 - 1，所以 DimLayer 现在的层级值最低，无法盖住任何窗口，所以没有在屏幕上显示。

这一节主要是为了和 3.1 节呼应，证明层级为 - 1 时的 DimLayer，的确是先于父 Layer 打印的。

### 3.5 小结

这一节从手机的实际表现出发，分析了设置了 relativeLayer 的效果带来的效果，验证了我们在 2.8 节总结的内容。

另外我们能够看到，虽然 WindowManager 为 App 提供的接口只能实现把 DimLayer 添加到 WindowState 这一层，也就是说，DimLayer 随着窗口的移除就跟着移除了，但是我们也可以修改代码，设置 DimLayer 的相对 Layer 为 Task 或者 ActivityRecord，从而实现把 DimLayer 添加到 Task 或者是 ActivityRecord 这一层之中。具体如何选择，则要看你的 Dim 效果是想要对整个 App 生效，还是对某一个 Activity 生效，或者只是在某个子窗口显示的时候才生效。

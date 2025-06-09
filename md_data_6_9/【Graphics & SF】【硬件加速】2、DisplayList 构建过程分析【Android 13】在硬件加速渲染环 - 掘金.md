> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7234545968095674427)

> 在硬件加速渲染环境中，Android 应用程序窗口的 UI 渲染是分两步进行的。第一步是构建 DisplayList，发生在应用程序进程的 Main Thread 中；第二步是渲染 DisplayList，发生在应

在硬件加速渲染环境中，Android 应用程序窗口的 UI 渲染是分两步进行的。第一步是构建 DisplayList，发生在应用程序进程的 Main Thread 中；第二步是渲染 DisplayList，发生在应用程序进程的 Render Thread 中。DisplayList 是以 View 为单位进行构建的，因此每一个 View 都对应有一个 DisplayList。

这里说的 DisplayList 与 Open GL 里面的 DisplayList 在概念上是类似的，不过是两个不同的实现。DisplayList 的本质是一个缓冲区，它里面记录了即将要执行的绘制命令序列。这些绘制命令最终会转化为 Open GL 命令由 GPU 执行。这意味着我们在调用 Canvas API 绘制 UI 时，实际上只是将 Canvas API 调用及其参数记录在 DisplayList 中，然后等到下一个 Vsync 信号到来时，记录在 DisplayList 里面的绘制命令才会转化为 Open GL 命令由 GPU 执行。与直接执行绘制命令相比，先将绘制命令记录在 DisplayList 中然后再执行有两个好处。第一个好处是在绘制窗口的下一帧时，若某一个 VIew 的 UI 没有发生变化，那么就不必执行与它相关的 Canvas API，即不用执行它的成员函数 onDraw，而是直接复用上次构建的 DisplayList 即可。第二个好处是在绘制窗口的下一帧时，若某一个 VIew 的 UI 发生了变化，但是只是一些简单属性发生了变化，例如位置和透明度等简单属性，那么也不必重建它的 DisplayList，而是直接修改上次构建的 DisplayList 的相关属性即可，这样也可以省去执行它的成员函数 onDraw。

窗口的 View 层级结构是树形结构，那么 DisplayList 的层级结构自然也是一个以 View 的树形结构建立起来的 Display 树形结构，如图所示：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/00b33be41f8a44a19f7331c8a55c5677~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1181&h=341&s=43141&e=png&b=ffffff)

要知道 DisplayList 的信息并不是直接保存在 View 中的，Android 抽象出了一个 RenderNode 的概念来和 View 树中的每一个 View 一一对应，并保存 DisplayList 的信息，顾名思义，每一个 RenderNode 代表了树形结构中的一个节点。

为什么要设计这么一个类出来呢，因为 View 毕竟只是 Framework Java 层的概念，在 Framework C/C++ 层是没有 View 这个概念的，但是底层仍需要一个结构体来保存从上层传来的 View 信息，比如几何信息和绘制信息等，而保存这些信息的就是 RenderNode。

接下来我们就结合源代码来分析 Android 应用程序窗口视图的 DisplayList 的构建过程。

Android 应用程序窗口 UI 的绘制过程是从 ViewRootImpl 类的成员函数 draw 开始的，它的实现如下所示：

```
	private boolean draw(boolean fullRedrawNeeded, boolean forceDraw) {
        
        if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            if (isHardwareEnabled()) {
                
                
                mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
            } else {
                
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets)) {
                    if (DEBUG_DRAW) {
                        Log.v(mTag, "drawSoftware return: this = " + this);
                    }
                    return false;
                }
            }
        }

        
    }


```

在几种情况下，需要进行绘制：

1）、当前需要更新的区域，即 ViewRootImpl 类的成员变量 mDirty 描述的脏区域不为空。

2）、窗口当前有动画需要执行，即 ViewRootImpl 类的成员变量 mIsAnimating 的值等于 true。

3）、accessibilityFocusDirty，无障碍服务相关。

之前已经已经分析过了，当硬件加速是默认开启的，所以这里走的就是硬件绘制流程 ThreadedRenderer.draw，而非软件绘制 drawSoftware。

```
	void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
        

        updateRootDisplayList(view, callbacks);

        

        int syncResult = syncAndDrawFrame(frameInfo);
        
    }


```

1）、ThreadedRenderer.updateRootDisplayList 函数以传参 View 为顶层 View 构建 DisplayList 树，对于 Activity 来说顶层 View 为 DecorView。

2）、ThreadedRenderer.syncAndDrawFrame 用来绘制新的一帧。

本文分析 DisplayList 树的构建流程。

```
	private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
        updateViewTreeDisplayList(view);

        

        if (mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()) {
            RecordingCanvas canvas = mRootNode.beginRecording(mSurfaceWidth, mSurfaceHeight);
            try {
                

                canvas.enableZ();
                canvas.drawRenderNode(view.updateDisplayListIfDirty());
                canvas.disableZ();

                
            } finally {
                mRootNode.endRecording();
            }
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }


```

1）、ThreadedRenderer 类的成员函数 updateRootDisplayList 通过调用另一个成员函数 updateViewTreeDisplayList 来构建参数 view 描述的视图的 Display List，即图 1 中的 Decor View 的 Display List。

2）、mRootNodeNeedsUpdate 表示是否需要更新另外一个成员变量 mRootNode 描述的一个 Render Node 的 Display List。

3）、RenderNode.hasDisplayList 返回 RenderNode 是否有 Display List，如果返回 false，RenderNode 应该使用 RenderNode.beginRecording 和 RenderNode.endRecording 重新记录。没有显示列表的 RenderNode 仍然可以被绘制，但是在其显示列表被更新之前，它不会对渲染内容产生影响。当 RenderNode 不再被任何东西绘制时，系统可能会自动调用 {@link #discardDisplayList()}。 因此，在绘制之前确保 RenderNode 上的 hasDisplayList 为真非常重要。

4）、ThreadedRenderer 类的成员变量 mRootNode 描述的 Render Node 即为当前窗口的 Root Node，更新它的 Display List 实际上就是要将参数 view 描述的视图的 Display List 记录到它里面去，具体方法如下所示：

1.  开始记录之前，调用 RenderNode.beginRecording，该方法返回一个 RecordingCanvas 类型的 Canvas，在这个 Canvas 上执行的所有操作都会被记录并且存储在 DisplayList 上。
2.  调用上面获得的 RecordingCanvas 的成员函数 drawRenderNode 将参数 view 描述的视图的 Display List 绘制在它里面。
3.  记录结束后，调用 RenderNode.endRecording，取出上述已经绘制好的 RecordingCanvas 的数据，并且作为上述 Render Node 的新的 Display List。

3.1 ThreadedRenderer.updateViewTreeDisplayList
----------------------------------------------

```
	private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
        updateViewTreeDisplayList(view);

        
    }

    private void updateViewTreeDisplayList(View view) {
        view.mPrivateFlags |= View.PFLAG_DRAWN;
        view.mRecreateDisplayList = (view.mPrivateFlags & View.PFLAG_INVALIDATED)
                == View.PFLAG_INVALIDATED;
        view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
        view.updateDisplayListIfDirty();
        view.mRecreateDisplayList = false;
    }


```

3.2 View.updateDisplayListIfDirty
---------------------------------

```
    
    @NonNull
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
    public RenderNode updateDisplayListIfDirty() {
        final RenderNode renderNode = mRenderNode;
        

        if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
                || !renderNode.hasDisplayList()
                || (mRecreateDisplayList)) {
            
            
            if (renderNode.hasDisplayList()
                    && !mRecreateDisplayList) {
                
                dispatchGetDisplayList();
                

                return renderNode; 
            }

            
            
            mRecreateDisplayList = true;

            int width = mRight - mLeft;
            int height = mBottom - mTop;
            int layerType = getLayerType();

            
            
            final RecordingCanvas canvas = renderNode.beginRecording(width, height);

            try {
                if (layerType == LAYER_TYPE_SOFTWARE) {
                    
                } else {
                    
                    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                        dispatchDraw(canvas);
                        
                    } else {
                        draw(canvas);
                    }
                }
            } finally {
                renderNode.endRecording();
                setDisplayListProperties(renderNode);
            }
            
        } else {
            
        }
        
    }


```

在三种情况下，View 类的成员函数 updateDisplayListIfDirty 需要重新构建当前正在处理的 View 或者其子 View 的 Display List：

1）、View 类的成员变量 mPrivateFlags 的值的 PFLAG_DRAWING_CACHE_VALID 位等于 0，这表明上次构建的 Display List 已经失效。

2）、RenderNode.hasDisplayList 函数返回 false，表示当前 RenderNode 没有一个 DisplayList，需要重新记录。

3）、VIew.mRecreateDisplayList 为 true，表示当前 View 被标记为 INVALIDATED，或者其 DisplayList 已无效，此时必须重绘其 DisplayList。

其中，如果 View 类的成员变量 mPrivateFlags 的值的 PFLAG_DRAWING_CACHE_VALID 位不等于 0，并且成员变量 mRenderNode 描述的 Render Node 内部维护的 Display List Data 也是有效的，那么就表明上次为当前正在处理的 View 的 UI 没有发生变化。但是如果在这种情况下，View 类的成员变量 mRecreateDisplayList 等于 false，就说明虽然当前正在处理的 View 的 UI 没有发生变化，但是它的子 View 的 UI 发生了变化。这时候就需要对这些子 View 的 Display List 进行重新构建，并且更新到当前正在处理的 View 的 Display List 去。这是通过调用 View 类的成员函数 dispatchGetDisplayList 来完成的。

除了上述这种情况，其余情况均表明需要重新构建当前正在处理的 View 及其子 View 的 Display List。这些 Display List 的构建过程如下所示：

1）、从当前正在处理的 View 关联的 Render Node 获得一个 RecordingCanvas。

2）、将当前正在处理的 View 及其子 View 的 UI 绘制命令记录在上面获得的 RecordingCanvas 中。

3）、将前面已经绘制好的 RecordingCanvas 的 Display List Data 提取出来，并且设置为当前正在处理的 View 关联的 Render Node 里面去。

接下来会依次分析这三个步骤，但是在此之前，先看下这里 View 的成员变量 mRenderNode。

View.mRenderNode 的定义为：

```
    
    @UnsupportedAppUsage
    final RenderNode mRenderNode;


```

RenderNode 持有 View 的属性，潜在地持有 View 内容的 DisplayList。

当 RenderNode 为非空且有效，它被期望包含 View 内容的最新副本。它的 DisplayList 内容会在该 View 暂时从 Window 上分离的时候被清除，并且在 View 被清理的时候重置。

在 View 的构造方法中赋值：

```
    public View(Context context) {
        
        
        mRenderNode = RenderNode.create(getClass().getName(), new ViewAnimationHostBridge(this));

        
    }


```

### 3.2.1 RenderNode.create

```
    private RenderNode(String name, AnimationHost animationHost) {
        mNativeRenderNode = nCreate(name);
        
    }

    
    public static RenderNode create(String name, @Nullable AnimationHost animationHost) {
        return new RenderNode(name, animationHost);
    }


```

调用 JNI 函数创建一个 Nativce 层的 RenderNode 对象，并将其地址保存在成员变量 mNativeRenderNode 中。

nCreate 定义为：

```
static const JNINativeMethod gMethods[] = {
        
        
        
        {"nCreate", "(Ljava/lang/String;)J", (void*)android_view_RenderNode_create},
    	
}


```

那么调用的就是 android_view_RenderNode_create 函数。

### 3.2.2 android_view_RenderNode_create

```
static jlong android_view_RenderNode_create(JNIEnv* env, jobject, jstring name) {
    RenderNode* renderNode = new RenderNode();
    
    
    
    return reinterpret_cast<jlong>(renderNode);
}


```

在 Nativce 层创建一个 RenderNode 对象，并返回其地址。

补充：RenderNode 定义
----------------

这里翻译一下源码中 RenderNode 的定义。

RenderNode is used to build hardware accelerated rendering hierarchies. Each RenderNode contains both a display list as well as a set of properties that affect the rendering of the display list. RenderNodes are used internally for all Views by default and are not typically used directly.

RenderNode 用来建立硬件加速渲染 View 层级结构。每一个 RenderNode 包含了一个 DisplayList，以及一个影响该 DisplayList 渲染的属性集合。默认情况下，RenderNodes 在内部用于所有视图，通常不会直接使用。

RenderNodes are used to divide up the rendering content of a complex scene into smaller pieces that can then be updated individually more cheaply. Updating part of the scene only needs to update the display list or properties of a small number of RenderNode instead of redrawing everything from scratch. A RenderNode only needs its display list re-recorded when its content alone should be changed. RenderNodes can also be transformed without re-recording the display list through the transform properties.

RenderNode 用于将复杂场景的渲染内容分成更小的部分，然后可以用更小的性能损耗来单独更新这些部分。更新场景的某部分只需要更新对应的 DisplayList 或者一小部分 RenderNode 的属性，而不用从头全部重绘。当一个 RenderNode 的内容被单独改变时，它只需要将其 DisplayList 重新进行记录。RenderNode 也能在不重新记录 DisplayList 的情况下通过变换属性进行变换。

A text editor might for instance store each paragraph into its own RenderNode. Thus when the user inserts or removes characters, only the display list of the affected paragraph needs to be recorded again.

例如，文本编辑器可以将每个段落存储到它自己的 RenderNode 中。当用户插入或删除字符时，只需要重新记录受影响段落的显示列表。

### Hardware acceleration

RenderNodes can be drawn using a {@link RecordingCanvas}. They are not supported in software. Always make sure that the {@link android.graphics.Canvas} you are using to render a display list is hardware accelerated using {@link android.graphics.Canvas#isHardwareAccelerated()}.

可以使用`RecordingCanvas` 来绘制 RenderNode。它们不支持通过软件绘制。始终确保你用来渲染一个 DisplayList 的`android.graphics.Canvas` 使用`android.graphics.Canvas#isHardwareAccelerated()` 进行硬件加速。

### Creating a RenderNode

```
      RenderNode renderNode = new RenderNode("myRenderNode");
      renderNode.setPosition(0, 0, 50, 50); 
      RecordingCanvas canvas = renderNode.beginRecording();
      try {
          
          canvas.drawRect(...);
      } finally {
          renderNode.endRecording();
      }


```

### Drawing a RenderNode in a View

```
     protected void onDraw(Canvas canvas) {
         if (canvas.isHardwareAccelerated()) {
             
             if (!myRenderNode.hasDisplayList()) {
                 updateDisplayList(myRenderNode);
             }
             
             canvas.drawRenderNode(myRenderNode);
         }
     }


```

### Releasing resources

This step is not mandatory but recommended if you want to release resources held by a display list as soon as possible. Most significantly any bitmaps it may contain.

如果你想尽快释放被 DisplayList 持有的资源，建议执行此步骤（非强制）。

```
    
    
    renderNode.discardDisplayList();


```

### Properties

In addition, a RenderNode offers several properties, such as {@link #setScaleX(float)} or {@link #setTranslationX(float)}, that can be used to affect all the drawing commands recorded within. For instance, these properties can be used to move around a large number of images without re-issuing all the individual `canvas.drawBitmap()` calls.

此外，RenderNode 还提供了一些属性，如`#setScaleX(float)` 或`#setTranslationX(float)` ，这些属性能够用来影响记录在其中的所有绘制指令。比如，这些属性能够移动大量的图片，而无需重新发起所有单独的`canvas.drawBitmap()` 调用。

```
    private void createDisplayList() {
        mRenderNode = new RenderNode("MyRenderNode");
        mRenderNode.setPosition(0, 0, width, height);
        RecordingCanvas canvas = mRenderNode.beginRecording();
        try {
            for (Bitmap b : mBitmaps) {
                canvas.drawBitmap(b, 0.0f, 0.0f, null);
                canvas.translate(0.0f, b.getHeight());
            }
        } finally {
            mRenderNode.endRecording();
        }
    }

    protected void onDraw(Canvas canvas) {
        if (canvas.isHardwareAccelerated())
            canvas.drawRenderNode(mRenderNode);
        }
    }

    private void moveContentBy(int x) {
         
         
         
         
         mRenderNode.offsetLeftAndRight(x);
         invalidate();
    }


```

A few of the properties may at first appear redundant, such as {@link #setElevation(float)} and {@link #setTranslationZ(float)}. The reason for these duplicates are to allow for a separation between static & transient usages. For example consider a button that raises from 2dp to 8dp when pressed. To achieve that an application may decide to setElevation(2dip), and then on press to animate setTranslationZ to 6dip. Combined this achieves the final desired 8dip value, but the animation need only concern itself with animating the lift from press without needing to know the initial starting value. {@link #setTranslationX(float)} and {@link #setTranslationY(float)} are similarly provided for animation uses despite the functional overlap with {@link #setPosition(Rect)}.

一些属性可能最初的时候出现的有一些冗余，如`#setElevation(float)` 和`#setTranslationZ(float)` 。 这些重复的原因是允许静态和瞬态用法之间的分隔。比如，考虑一个按钮，在按下时从 2dp 上升到 8dp。为了实现这一点，App 可能决定去 setElevation(2dip)，按钮按下时去动画，setTranslationZ 到 6dip。两者结合最终实现了 8dip 的值，但是动画应只关心其本身从按下到抬起，而无需知道初始值。

The RenderNode's transform matrix is computed at render time as follows:

某一时刻 RenderNode 的变换矩阵是按照以下方式进行计算的：

```
    Matrix transform = new Matrix();
    transform.setTranslate(renderNode.getTranslationX(), renderNode.getTranslationY());
    transform.preRotate(renderNode.getRotationZ(),
            renderNode.getPivotX(), renderNode.getPivotY());
    transform.preScale(renderNode.getScaleX(), renderNode.getScaleY(),
            renderNode.getPivotX(), renderNode.getPivotY());


```

The current canvas transform matrix, which is translated to the RenderNode's position, is then multiplied by the RenderNode's transform matrix. Therefore the ordering of calling property setters does not affect the result. That is to say that:

当前 Canvas 的变换矩阵，已经被转换到了 RenderNode 的位置，然后乘以 RenderNode 的变换矩阵。因此调用属性设置符的顺序不会影响结果，也就是说：

```
    renderNode.setTranslationX(100);
    renderNode.setScaleX(100);


```

is equivalent to:

等同于：

```
    renderNode.setScaleX(100);
    renderNode.setTranslationX(100);


```

### Threading

RenderNode may be created and used on any thread but they are not thread-safe. Only a single thread may interact with a RenderNode at any given time. It is critical that the RenderNode is only used on the same thread it is drawn with. For example when using RenderNode with a custom View, then that RenderNode must only be used from the UI thread.

RenderNode 可以在任何线程上被创建和使用，但是它们并不是线程安全的。在任何给定的时间，只有一个线程可以与 RenderNode 交互。 至关重要的是，RenderNode 仅在与它一起绘制的同一线程上使用。比如，当在一个自定义 View 上使用 RenderNode 时，那个 RenderNode 必须只能在 UI 线程上使用。

### When to re-render

Many of the RenderNode mutation methods, such as {@link #setTranslationX(float)}, return a boolean indicating if the value actually changed or not. This is useful in detecting if a new frame should be rendered or not. A typical usage would look like:

许多 RenderNode 的变异方法，如`#setTranslationX(float)` ，返回一个 boolean 类型的值来表示值是否真的被改变了。这在检测一个新的帧是否应该被渲染的时候很有用。一个典型的用法可能像这样：

```
    public void translateTo(int x, int y) {
        boolean needsUpdate = myRenderNode.setTranslationX(x);
        needsUpdate |= myRenderNode.setTranslationY(y);
        if (needsUpdate) {
            myOwningView.invalidate();
        }
    }


```

This is marginally faster than doing a more explicit up-front check if the value changed by comparing the desired value against {@link #getTranslationX()} as it minimizes JNI transitions. The actual mechanism of requesting a new frame to be rendered will depend on how this RenderNode is being drawn. If it's drawn to a containing View, as in the above snippet, then simply invalidating that View works. If instead the RenderNode is being drawn to a Canvas directly such as with {@link Surface#lockHardwareCanvas()} then a new frame needs to be drawn by calling {@link Surface#lockHardwareCanvas()}, re-drawing the root RenderNode or whatever top-level content is desired, and finally calling {@link Surface#unlockCanvasAndPost(Canvas)}.

一种更加明确的预先检查期望值是否改变的方法是，将期望值与`{@link #getTranslationX()}` 进行比较，这里的方法比这种方法要快一点，因为它最大限度地减少了 JNI 转换。请求渲染新帧的实际机制将依赖 RenderNode 以何种方式绘制。如果它被绘制到了一个包含的 View 中，如上面的代码片段所示，那么只需简单地将该 View 无效化即可。如果反之这个 RenderNode 被直接绘制到了一个 Canvas 上了，比如通过`Surface#lockHardwareCanvas()` ，那么一个新帧需要通过调用`Surface#lockHardwareCanvas()` 来绘制，重绘根 RenderNode 或者任何所需的顶级内容，最后调用`Surface#unlockCanvasAndPost(Canvas)` 。

3.3 RenderNode.beginRecording
-----------------------------

RenderNode.beginRecording 方法定义为：

```
    
	public @NonNull RecordingCanvas beginRecording(int width, int height) {
        if (mCurrentRecordingCanvas != null) {
            throw new IllegalStateException(
                    "Recording currently in progress - missing #endRecording() call?");
        }
        mCurrentRecordingCanvas = RecordingCanvas.obtain(this, width, height);
        return mCurrentRecordingCanvas;
    }

    static RecordingCanvas obtain(@NonNull RenderNode node, int width, int height) {
        if (node == null) throw new IllegalArgumentException("node cannot be null");
        RecordingCanvas canvas = sPool.acquire();
        if (canvas == null) {
            canvas = new RecordingCanvas(node, width, height);
        } else {
            nResetDisplayListCanvas(canvas.mNativeCanvasWrapper, node.mNativeRenderNode,
                    width, height);
        }
        canvas.mNode = node;
        canvas.mWidth = width;
        canvas.mHeight = height;
        return canvas;
    }


```

1）、RecordingCanvas.beginRecording 方法从 RecordingCanvas.obtain 方法中获取一个 RecordingCanvas 对象，保存在 RecordingCanvas.mCurrentRecordingCanvas 中。

2）、RecordingCanvas.obtain 方法首先是从一个 RecordingCanvas 对象池中请求一个 RecordingCanvas 对象。如果获取失败，再直接创建一个 RecordingCanvas 象。在将获取到的 RecordingCanvas 对象返回给调用者之前，还会将参数 node 描述的 Render Node 保存在其成员变量 mNode 中。

### 3.3.1 RecordingCanvas.init

首先看下 RecordingCanvas 类的定义：

```
public final class RecordingCanvas extends BaseRecordingCanvas


```

为延迟渲染记录 View 系统绘制操作的 Canvas 实现。

RecordingCanvas 的构造方法为：

```
    private RecordingCanvas(@NonNull RenderNode node, int width, int height) {
        super(nCreateDisplayListCanvas(node.mNativeRenderNode, width, height));
        mDensity = 0; 
    }

    public BaseRecordingCanvas(long nativeCanvas) {
        super(nativeCanvas);
    }

    
    public Canvas(long nativeCanvas) {
        if (nativeCanvas == 0) {
            throw new IllegalStateException();
        }
        mNativeCanvasWrapper = nativeCanvas;
        
    }


```

重点在于这个 JNI 函数，nCreateDisplayListCanvas，其对应的是 android_graphics_DisplayListCanvas 中的 android_view_DisplayListCanvas_createDisplayListCanvas 函数：

```
static JNINativeMethod gMethods[] = {
        
        {"nCreateDisplayListCanvas", "(JII)J",
         (void*)android_view_DisplayListCanvas_createDisplayListCanvas},
        
};


```

那么 Java 层的 Canvas 本质就是对 Native 层的 Canvas 的一层封装，调用 JNI 函数 nCreateDisplayListCanvas 创建 Native 层 Canvas 后，将其地址保存在成员变量 mNativeCanvasWrapper 中。

### 3.3.2 android_view_DisplayListCanvas_createDisplayListCanvas

继续看 android_view_DisplayListCanvas_createDisplayListCanvas 函数：

```
static jlong android_view_DisplayListCanvas_createDisplayListCanvas(CRITICAL_JNI_PARAMS_COMMA jlong renderNodePtr,
        jint width, jint height) {
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    return reinterpret_cast<jlong>(Canvas::create_recording_canvas(width, height, renderNode));
}



Canvas* Canvas::create_recording_canvas(int width, int height, uirenderer::RenderNode* renderNode) {
    return new uirenderer::skiapipeline::SkiaRecordingCanvas(renderNode, width, height);
}


```

这里看到，在 native 层实际创建的是一个 SkiaRecordingCanvas 对象，然后返回了该 Canvas 的地址。

SkiaRecordingCanvas 类的定义为：

```
class SkiaRecordingCanvas : public SkiaCanvas {
public:
    explicit SkiaRecordingCanvas(uirenderer::RenderNode* renderNode, int width, int height) {
        initDisplayList(renderNode, width, height);
    }

    

private:
    RecordingCanvas mRecorder;
    std::unique_ptr<SkiaDisplayList> mDisplayList;

    
};


```

一个 SkiaCanvas 实现，它记录由 SkLiteRecorder 和 SkiaDisplayList 支持的延迟渲染的绘图操作。

### 3.3.3 SkiaRecordingCanvas.initDisplayList

接下来看其构造函数中的初始化函数 initDisplayList：

```
void SkiaRecordingCanvas::initDisplayList(uirenderer::RenderNode* renderNode, int width,
                                          int height) {
    mCurrentBarrier = nullptr;
    SkASSERT(mDisplayList.get() == nullptr);

    if (renderNode) {
        mDisplayList = renderNode->detachAvailableList();
    }
    if (!mDisplayList) {
        mDisplayList.reset(new SkiaDisplayList());
    }

    mDisplayList->attachRecorder(&mRecorder, SkIRect::MakeWH(width, height));
    SkiaCanvas::reset(&mRecorder);
    mDisplayList->setHasHolePunches(false);
}


```

1）、首先调用 RenderNode.detachAvailableList 函数，查看传参 RenderNode 中上一帧缓存的 SkiaDisplayList 是否还在，如果在直接赋值给 SkiaRecordingCanvas 的 SkiaDisplayList 类型的成员变量 mDisplayList，否则创建一个新的 SkiaDisplayList 对象。

2）、接着调用 SkiaDisplayList.attachRecorder 函数：

```
    void attachRecorder(RecordingCanvas* recorder, const SkIRect& bounds) {
        recorder->reset(&mDisplayList, bounds);
    }


```

SkiaDisplayList 的成员变量 mDisplayList 为 DisplayListData 类型。

```
void RecordingCanvas::reset(DisplayListData* dl, const SkIRect& bounds) {
    this->resetCanvas(bounds.right(), bounds.bottom());
    fDL = dl;
    mClipMayBeComplex = false;
    mSaveCount = mComplexSaveCount = 0;
}


```

这里是将 RecordingCanvas 的 DisplayListData 类型的成员变量 fDL 指向了 SkiaDisplayList 的成员变量 mDisplayList。

那么这里 SkiaRecordingCanvas 的两个成员变量，RecordingCanvas 类型的 mRecorder，以及 SkiaDisplayList 类型的 mDisplayList，便通过内部的 DisplayListData 建立了联系。

3）、最后这里又调用了 SkiaCanvas.reset 函数：

```
void SkiaCanvas::reset(SkCanvas* skiaCanvas) {
    if (mCanvas != skiaCanvas) {
        mCanvas = skiaCanvas;
        mCanvasOwned.reset();
    }
    mSaveStack.reset(nullptr);
}


```

将 SkiaCanvas 的 SkCanvas 类型的成员变量指向了传入的 RecordingCanvas 对象，因为 RecordingCanvas 是 SkCanvas 的子类，RecordingCanvas 类的继承关系为：

```
class RecordingCanvas final : public SkCanvasVirtualEnforcer<SkNoDrawCanvas>


class SK_API SkNoDrawCanvas : public SkCanvasVirtualEnforcer<SkCanvas>


```

SkCanvasVirtualEnforcer 是一个模板类：

```
template <typename Base>
class SkCanvasVirtualEnforcer : public Base


```

因此 RecordingCanvas 的实际父类是 SkNoDrawCanvas 和 SkCanvas。

3.4 记录绘制命令
----------

回到 View.updateDisplayListIfDirty 方法：

```
    
    @NonNull
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
    public RenderNode updateDisplayListIfDirty() {
        final RenderNode renderNode = mRenderNode;
        

        if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
                || !renderNode.hasDisplayList()
                || (mRecreateDisplayList)) {
            
            
            int layerType = getLayerType();

            
            
            final RecordingCanvas canvas = renderNode.beginRecording(width, height);

            try {
                if (layerType == LAYER_TYPE_SOFTWARE) {
                    
                } else {
                    
                    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                        dispatchDraw(canvas);
                        
                    } else {
                        draw(canvas);
                    }
                }
            } finally {
                renderNode.endRecording();
                setDisplayListProperties(renderNode);
            }
            
        } else {
            
        }
        
    }


```

首先探究一下这里的局部变量 layerType，layerType 通过 View.getLayerType 方法得到，View.getLayerType 返回了这个 View 当前相关联的图层类型：

```
    
    @InspectableProperty(enumMapping = {
            @EnumEntry(value = LAYER_TYPE_NONE, name = "none"),
            @EnumEntry(value = LAYER_TYPE_SOFTWARE, name = "software"),
            @EnumEntry(value = LAYER_TYPE_HARDWARE, name = "hardware")
    })
    @LayerType
    public int getLayerType() {
        return mLayerType;
    }


```

1）、LAYER_TYPE_NONE，表示 View 没有一个 Layer。

2）、LAYER_TYPE_SOFTWARE，表示 View 有一个 Software Layer。Software Layer 由 Bimap 支持，并且使用 Android 的软件渲染管道渲染 View，即使启用了硬件加速也是如此。

3）、LAYER_TYPE_HARDWARE，表示 View 有一个 Hareware Layer。Hareware Layer 由硬件特定纹理（通常是 OpenGL 硬件上的帧缓冲区对象（Frame Buffer Object）或者说 FBO）支持，并导致使用 Android 的硬件渲染管道渲染视图，但前提是为视图层次结构打开了硬件加速。

这里进一步扩展一下 View 的 Layer：

> 所谓 View Layer，又称为离屏缓冲（Off-screen Buffer），它的作用是单独启用一块地方来绘制这个 View ，而不是使用软件绘制的 Bitmap 或者通过硬件加速的 GPU。这块「地方」可能是一块单独的 Bitmap，也可能是一块 OpenGL 的纹理（texture，OpenGL 的纹理可以简单理解为图像的意思），具体取决于硬件加速是否开启。采用什么来绘制 View 不是关键，关键在于当设置了 View Layer 的时候，它的绘制会被缓存下来，而且缓存的是最终的绘制结果，而不是像硬件加速那样只是把 GPU 的操作保存下来再交给 GPU 去计算。通过这样更进一步的缓存方式，View 的重绘效率进一步提高了：只要绘制的内容没有变，那么不论是 CPU 绘制还是 GPU 绘制，它们都不用重新计算，而只要只用之前缓存的绘制结果就可以了。

默认情况下 View 的 mLayerType 为 LAYER_TYPE_NONE，因此大部分情况下 RenderNode.updateDisplayListIfDirty 方法里走的逻辑为：

```
                    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                        dispatchDraw(canvas);
                        
                    } else {
                        draw(canvas);
                    }


```

这时候当前正在处理的 View 的成员变量 mPrivateFlags 的值的 PFLAG_SKIP_DRAW 位设置为 1，就表明当前正在处理的 View 是一个 Layout，并且没有设置 Background，这时候就可以走一个捷径，即直接调用 View 类的成员函数 dispatchDraw 将该 Layout 的子 View 的 UI 绘制在当前正在处理的 View 关联的 Render Node 对应的 RecordingCanvas 即可。另一方面，如果当前正在处理的 View 的成员变量 mPrivateFlags 的值的 PFLAG_SKIP_DRAW 位设置为 0，那么就不能走捷径，而是走一个慢路径，规规矩矩地调用 View 类的成员函数 draw 将当前正在处理的 View 及其可能存在的子 View 的 UI 绘制在关联的 Render Node 对应的 RecordingCanvas 上。

View.dispatchDraw 相当于直接跳过了当前 View 的绘制直接去绘制其子 View，我们还是看下通用流程下的 View.draw。

### 3.4.1 View.draw

```
    
    @CallSuper
    public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        

        
        
        
        long logTime = System.currentTimeMillis();
        onDraw(canvas);
        
        
    }


```

一个 View 的主要 UI 是由子类实现的成员函数 onDraw 绘制的。这个成员函数通过参数 canvas 可以获得一个 Canvas，然后就调用这个 Canvas 提供的 API 就可以绘制一个 View 的 UI。这意味着对一个 View 来说，当它的成员函数 onDraw 被调用时，它是不需要区别它是通过硬件渲染还是软件渲染的。但是从结果来看，当使用硬件渲染时，调用 Canvas API 相当是将 API 调用记录在一个 Display List 中，而当使用软件渲染时，调用 Canvas API 相当是将 UI 绘制在一个 Bitmap 中。

此外，对于使用硬件渲染的 View 来说，它的 Background 也是抽象为一个 Render Node 绘制在宿主 View 关联的一个 Render Node 对应的 RecordingCanvas 上的，相当于是将 Background 看作是一个 View 的子 View。

这里继续分析 View.onDraw。

### 3.4.2 View.onDraw

```
    
    protected void onDraw(Canvas canvas) {
    }


```

View 的 onDraw 方法需要其子类自行实现，具体就是调用 Canvas 的各种 drawXXX 的 API，如 drawLine、draw Bitmap 和 drawColor 等进行绘制，拿上面的 google 的例子：

```
      RenderNode renderNode = new RenderNode("myRenderNode");
      renderNode.setPosition(0, 0, 50, 50); 
      RecordingCanvas canvas = renderNode.beginRecording();
      try {
          
          canvas.drawRect(...);
      } finally {
          renderNode.endRecording();
      }


```

在创建一个 RenderNode，并且通过 RenderNode.beginRecording 获取一个 RecordingCanvas 后，下一步就可以拿这个 RecordingCanvas 进行绘制了，即这里的：

```
          
          canvas.drawRect(...);


```

看下 Canvas.drawRect 的实现。

### 3.4.3 BaseRecordingCanvas.drawRect

根据之前的分析我们知道，在我们分析的流程中，View.onDraw 方法中传入的 Canvas 是 RenderNode.beginRecording 方法返回的一个 RecordingCanvas 对象，但是由于 RecordingCanvas 没有重写 drawRect 方法，所以这里调用的是其父类 BaseRecordingCanvas 的 drawRect 方法：

```
	@Override
    public final void drawRect(float left, float top, float right, float bottom,
            @NonNull Paint paint) {
        nDrawRect(mNativeCanvasWrapper, left, top, right, bottom, paint.getNativeInstance());
    }


```

对应的 JNI 函数为：

```
static void drawRect(JNIEnv* env, jobject, jlong canvasHandle, jfloat left, jfloat top,
                     jfloat right, jfloat bottom, jlong paintHandle) {
    const Paint* paint = reinterpret_cast<Paint*>(paintHandle);
    get_canvas(canvasHandle)->drawRect(left, top, right, bottom, *paint);
}


```

由于 Java 层 Canvas 成员变量 mNativeCanvasWrapper 保存了之前 Native 层的创建的一个 SkiaRecordingCanvas 对象的地址，那么这里的 get_canvas 获取的就是一个 SkiaRecordingCanvas 对象，但是 SkiaRecordingCanvas 并没有一个 drawRect 函数，所以这里调用的是其父类 SkiaCanvas 的 drawRect 函数。

### 3.4.4 SkiaCanvas.drawRect

```
void SkiaCanvas::drawRect(float left, float top, float right, float bottom, const Paint& paint) {
    if (CC_UNLIKELY(paint.nothingToDraw())) return;
    applyLooper(&paint, [&](const SkPaint& p) {
        mCanvas->drawRect({left, top, right, bottom}, p);
    });
}


```

SkiaCanvas 的成员变量 mCanvas 之前分析过，是一个 RecordingCanvas 对象，那么这里最终调用的是 RecordingCanvas.drawRect。

从 SkiaRecordingCanvas.initDisplayList 函数中我们了解到，这里的 mCanvas 也不是一个 SkCanvas 对象，而是 SkCanvas 的一个子类，RecordingCanvas。但是 RecordingCanvas 没有自己的 drawRect 函数，所以这里调用的依旧是 SkCanvas.drawRect。

### 3.4.5 SkCanvas.drawRect

```
void SkCanvas::drawRect(const SkRect& r, const SkPaint& paint) {
    TRACE_EVENT0("skia", TRACE_FUNC);
    
    
    this->onDrawRect(r.makeSorted(), paint);
}


```

调用 onDrawRect 函数，这个函数 RecordingCanvas 复写了，所以调用的是 RecordingCanvas.onDrawRect。

### 3.4.6 RecordingCanvas.onDrawRect

```
void RecordingCanvas::onDrawRect(const SkRect& rect, const SkPaint& paint) {
    fDL->drawRect(rect, paint);
}


```

从 SkiaRecordingCanvas.initDisplayList 中我们知道这里 RecordingCanvas 的成员函数 fDL 已经指向了一个 DisplayListData 对象，所以 RecordingCanvas.drawRect 调用的是 DisplayListData.drawRect。

### 3.4.7 DisplayListData.draw

```
void DisplayListData::drawRect(const SkRect& rect, const SkPaint& paint) {
    this->push<DrawRect>(0, rect, paint);
}


```

在 DisplayListData.drawRect 中，则是 push 了一个 DrawRect 的 OP：

```
struct DrawRect final : Op {
    static const auto kType = Type::DrawRect;
    DrawRect(const SkRect& rect, const SkPaint& paint) : rect(rect), paint(paint) {}
    SkRect rect;
    SkPaint paint;
    void draw(SkCanvas* c, const SkMatrix&) const { c->drawRect(rect, paint); }
};

struct Op {
    uint32_t type : 8;
	uint32_t skip : 24;
};


```

OP 代表了某一种操作，type 代表了 OP 的类型，skip 则代表了该 OP 占内存的大小。

该类中该定义有很多其他 OP，如：

```
struct DrawPath final : Op 

struct DrawRegion final : Op


```

接下来看 DisplayListData.push。

### 3.4.8 DisplayListData.push

首先看下 DisplayListData 的一些成员变量：

```
class DisplayListData final {
public:
	

    SkAutoTMalloc<uint8_t> fBytes;
    size_t fUsed = 0;
    size_t fReserved = 0;

    bool mHasText : 1;
};


```

fBytes 代表了一块内存空间起始地址，fUsed 表明这块内存已经使用了多少空间，fReserved 代表的是被分配了多少空间。

```
template <typename T, typename... Args>
void* DisplayListData::push(size_t pod, Args&&... args) {
    size_t skip = SkAlignPtr(sizeof(T) + pod);
    SkASSERT(skip < (1 << 24));
    if (fUsed + skip > fReserved) {
        static_assert(SkIsPow2(SKLITEDL_PAGE), "This math needs updating for non-pow2.");
        
        fReserved = (fUsed + skip + SKLITEDL_PAGE) & ~(SKLITEDL_PAGE - 1);
        fBytes.realloc(fReserved);
        LOG_ALWAYS_FATAL_IF(fBytes.get() == nullptr, "realloc(%zd) failed", fReserved);
    }
    SkASSERT(fUsed + skip <= fReserved);
    auto op = (T*)(fBytes.get() + fUsed);
    fUsed += skip;
    new (op) T{std::forward<Args>(args)...};
    op->type = (uint32_t)T::kType;
    op->skip = skip;
    return op + 1;
}


```

DataListData 的 push 函数，就是计算每一条绘制指令的占用空间，然后将这些绘制指令存储到以 fBytes 为起始位置的内存空间中。

那么再次回看 Java 层 RecordingCanvas 的类定义：

A Canvas implementation that records view system drawing operations for deferred rendering.

这么看的确没错，绘制操作被记录到了一个 Native 层 SkiaRecordingCanvas 的 SkiaDisplayList 类型的成员变量 mDisplayList 的 DisplayListData 类型的成员变量 mDisplayList 中，而该 SkiaRecordingCanvas 的地址则由一个 Java 层的 RecordingCanvas 对象保存。

3.5 RenderNode.endRecording
---------------------------

```
    
    public void endRecording() {
        if (mCurrentRecordingCanvas == null) {
            throw new IllegalStateException(
                    "No recording in progress, forgot to call #beginRecording()?");
        }
        RecordingCanvas canvas = mCurrentRecordingCanvas;
        mCurrentRecordingCanvas = null;
        canvas.finishRecording(this);
        canvas.recycle();
    }


```

停止记录当前的 DisplayList。调用此方法将会标记 DisplayList 为可用，RenderNode.hasDisplayList 方法将会返回 true。

### 3.5.1 RecordingCanvas.finishRecording

```
    void finishRecording(RenderNode node) {
        nFinishRecording(mNativeCanvasWrapper, node.mNativeRenderNode);
    }


```

对应的是 android_view_DisplayListCanvas_finishRecording 函数。

```
static void android_view_DisplayListCanvas_finishRecording(
        CRITICAL_JNI_PARAMS_COMMA jlong canvasPtr, jlong renderNodePtr) {
    Canvas* canvas = reinterpret_cast<Canvas*>(canvasPtr);
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    canvas->finishRecording(renderNode);
}


```

传入的分别是之前创建 Native 层 SkiaRecordingCanvas 对象和 RenderNode 对象时返回给 Java 层的对象地址，那么这里可以根据这些地址拿到对应的 SkiaRecordingCanvas 对象和 RenderNode 对象。

因此这里继续调用 SkiaRecordingCanvas.finishRecording。

### 3.5.2 SkiaRecordingCanvas.finishRecording

```
std::unique_ptr<SkiaDisplayList> SkiaRecordingCanvas::finishRecording() {
    
    enableZ(false);
    mRecorder.restoreToCount(1);
    return std::move(mDisplayList);
}

void SkiaRecordingCanvas::finishRecording(uirenderer::RenderNode* destination) {
    destination->setStagingDisplayList(uirenderer::DisplayList(finishRecording()));
}


```

1）、无参的 SkiaRecordingCanvas.finishRecording 返回的是一个 SkiaDisplayList 对象，之前 View 的各种绘制命令是记录到了这个 SkiaDisplayList 的 DisplayListData 类型的成员变量 mDisplayList 中。

2）、这里将 SkiaDisplayList 封装到一个 DisplayList 对象中，然后调用 RenderNode.setStagingDisplayList。

```
class SkiaDisplayListWrapper {
public:
    
    explicit SkiaDisplayListWrapper() {}

    
    explicit SkiaDisplayListWrapper(std::unique_ptr<skiapipeline::SkiaDisplayList> impl)
        : mImpl(std::move(impl)) {}
    
    
    
private:
    std::unique_ptr<skiapipeline::SkiaDisplayList> mImpl;
};   




using DisplayList = SkiaDisplayListWrapper;


```

这里使用了别名 “DisplayList” 替代了原始类型“SkiaDisplayListWrapper”。

DisplayList，或 SkiaDisplayListWrapper，即是对 SkiaDisplayList 的一层封装，真正的 SkiaDisplayList 保存在 DisplayList 的成员变量 mImpl 中。

### 3.5.3 RenderNode.setStagingDisplayList

```
void RenderNode::setStagingDisplayList(DisplayList&& newData) {
    mValid = newData.isValid();
    mNeedsDisplayListSync = true;
    mStagingDisplayList = std::move(newData);
}


```

1）、RenderNode.mValid 定义为：

```
    
    
    bool mValid = false;


```

表明一个 RenderNode 是否设置一了个 DisplayList。

根据的就是这里传入的 DisplayList 中封装的 SkiaDisplayList 是否为空。

```
    
    
    [[nodiscard]] bool isValid() const {
        return mImpl.get() != nullptr;
    }


```

2）、接着将 RenderNode 的 DisplayList 类型的成员变量 mStagingDisplayList 指向参数 newData：

```
    DisplayList mStagingDisplayList;


```

绘制命令是保存在 SkiaDisplayList 的 DisplayListData 类型的成员变量 mDisplayList 中，而 DisplayList 封装了 SkiaDisplayList，那么 RenderNode 保存了 DisplayList，实际上就是保存了绘制该 RenderNode 对应的 View 所需的绘制命令。

3.6 小结
------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ae4d28b173a4b16a92ffedd5720bd24~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=2441&h=838&s=117890&e=png&b=ffffff)

1）、RenderNode.beginRecording，在 Native 创建了一个 SkiaRecordingCanvas，并且初始化了其类型为 SkiaDisplayList 和 RecordingCanvas 的成员变量。

2）、经过 View.onDraw，绘制当前 View 的绘制命令保存在了 Native 层 SkiaRecordingCanvas 对象的，SkiaDisplayList 类型的成员变量 mDisplayList 的，DisplayListData 类型的成员变量中。

3）、经过 RecordingCanvas.endRecording，SkiaDisplayList 的所有权发生了转移，从 SkiaRecordingCanvas 到了 RenderNode 中。由于 SkiaDisplayList 中保存着存储了绘制命令的 DisplayListData，那么 SkiaDisplayList 从 SkiaRecordingCanvas 转移到 RenderNode，实际上也就是绘制命令的存储点从 SkiaRecordingCanvas 转移到了 RenderNode。

至于为何要有这样的转移，回看 RenderNode.beginRecording 的内容，会发现每次想通过 RenderNode.beginRecording 方法获取一个 RecordingCanvas 对象时，是优先从一个 RecordingCanvas 对象池中去取的，那么这里就存在一个 RecordingCanvas 对象的回收利用。虽然 DisplayList 的信息最开始是保存在 SkiaRecordingCanvas 的 SkiaDisplayList 中的，但是 SkiaRecordingCanvas 并不和一个 View 或者 RenderNode 是一一对应的关系，因此如果想要根据 View 树结构构建一个 DisplayList 树，依靠 SkiaRecordingCanvas 是不可能的，因此才需要将 DisplayList 的信息从 SkiaRecordingCanvas 转移到 RenderNode 中。

经过 ThreadedRenderer.updateViewTreeDisplayList 方法，我们成功将绘制每一个 View 的命令保存在了这个 View 对应的 RenderNode 中的 DisplayList 中了，但是似乎这棵 DisplayList 树还没有建立起来，原因则在于每个 RenderNode 还是独立的，彼此并没有建立起联系。

当绘制流程走到某一个 View 时，有两件事是这个 View 需要做的，一是根据需要绘制自身，二是发起对子 View 的绘制调用命令。

要知道 DisplayList 树是如何构建的，就需要探究一下 View 层级结构中的每个 View 的绘制方法的调用是如何从上到下进行分发的。

4.1 Draw 方法的调用流程
----------------

这里拿一个我自己创建的一个 View 层级结构很简单的 Activity 为例：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ed8daccfb194ec1b7249e02093f769e~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=868&h=846&s=34090&e=png&b=3c3f41)

不看 StatusBar 和 NavigationBar 的背景，该 Activity 从顶层的 DecorView 到底层的 TextView 只有 5 层，每层一个 View：

```
DecorView
	LinearLayout
		FrameLayout
			ConstraintLayout
				TextView


```

### 4.1.1 View.updateDisplayListIfDirty

根据之前的分析，在经过以下流程后：

```
ViewRootImpl.draw
	-> ThreadedRenderer.draw
		-> ThreadedRenderer.updateRootDisplayList
			-> ThreadedRenderer.updateViewTreeDisplayList
				-> View.updateDisplayListIfDirty


```

首先是一个 View 层级结构中的顶层 View 走到了 View.updateDisplayListIfDirty，对 Activity 来说就是 DecorView。

只聚焦 View 的绘制方法的调用流程：

```
                    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                        dispatchDraw(canvas);
                        
                    } else {
                        draw(canvas);
                    }


```

之前讲过，PFLAG_SKIP_DRAW 这个标志位表示是否跳过当前 View 的绘制，比如例子中的 Activity 的 View 层级结构中的 LinearLayout、FrameLayout 等，最主要的用处是作为其他 View 的容器，它本身是没有内容需要绘制的，因此这些 Layout 默认是无需绘制的，因此在这里走的是 ViewGroup.dispatchDraw。而像 TextView 等有内容的 View，这里走的则是 View.draw。

这里继续看 View.draw 方法，无需看 ViewGroup.dispatchDraw，因为 View.draw 方法中也有对于 ViewGroup.dispatchDraw 的调用。

### 4.1.2 View.draw(Canvas)

```
    public void draw(Canvas canvas) {
        
        
        
        long logTime = System.currentTimeMillis();
        onDraw(canvas);
        
        
        
        
        dispatchDraw(canvas);
        
        
    }


```

只关注以下两部分：

1）、调用 View.onDraw，绘制当前 View。

2）、调用 View.dispatchDraw，发起对子 View 的绘制命令的调用。

### 4.1.3 View.dispatchDraw

```
    
    protected void dispatchDraw(Canvas canvas) {

    }


```

由 draw 调用来绘制子 View。 这可能会被派生类覆盖，以便在绘制其子 View 之前（但在绘制自己的 View 之后）获得控制权。

如果当前 View 没有子 View，那么什么也不用做，否之则需要调用自己的 dispatchDraw 方法来发起对子 View 的 draw 方法的调用。

在我们所举的例子中，LInearLayout 等 Layout 调用的都是 ViewGroup.dispatchDraw。

### 4.1.4 ViewGroup.dispatchDraw

```
    @Override
    protected void dispatchDraw(Canvas canvas) {
        
        
        for (int i = 0; i < childrenCount; i++) {
            while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
                final View transientChild = mTransientViews.get(transientIndex);
                if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                        transientChild.getAnimation() != null) {
                    more |= drawChild(canvas, transientChild, drawingTime);
                }
                
            }

            
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        while (transientIndex >= 0) {
            
            if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                    transientChild.getAnimation() != null) {
                more |= drawChild(canvas, transientChild, drawingTime);
            }
            
        }
        
        
        
        
        if (mDisappearingChildren != null) {
            
            for (int i = disappearingCount; i >= 0; i--) {
                final View child = disappearingChildren.get(i);
                more |= drawChild(canvas, child, drawingTime);
            }
        }
    }


```

继续调用了 ViewGroup.drawChild。

### 4.1.5 ViewGroup.drawChild

```
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }


```

调用了 View 的另外一个同名方法 draw(Canvas, ViewGrou, long)。

### 4.1.6 View.draw(Canvas, ViewGroup, long)

```
    
    boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
        final boolean hardwareAcceleratedCanvas = canvas.isHardwareAccelerated();

        boolean drawingWithRenderNode = drawsWithRenderNode(canvas);

        

        if (drawingWithRenderNode) {
            
            
            renderNode = updateDisplayListIfDirty();
            
        }
        
        
        
        final boolean drawingWithDrawingCache = cache != null && !drawingWithRenderNode;

        

        if (!drawingWithDrawingCache) {
            if (drawingWithRenderNode) {
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                ((RecordingCanvas) canvas).drawRenderNode(renderNode);
            } else {
                
            }
        } else if (cache != null) {
            
        }

        
    }


```

1）、局部变量 drawingWithRenderNode 表明当前 View 是否绘制在一个支持硬件加速渲染的 Canvas 上，根据之前的分析知道这里的 Canvas 为 RecordingCanvas，所以这里 drawingWithRenderNode 为 true。

2）、局部变量 drawingWithRenderNode 为 true，那么就会为当前 View 调用 View.updateDisplayListIfDirty，后续的流程又回到了 4.1 节，实现了 View 层级结构的迭代，最终每一个 View 的 draw 方法都能调用到。

3）、当前 View 绘制完成后，调用 RecordingCanvas.drawRenderNode 方法。

### 4.1.7 小结

以我们这里的 Activity 为例，首先是 DecorView，从 View.updateDisplayListIfDirty 为起点：

```
View.updateDisplayListIfDirty	----	(DecorView)
	-> View.draw(Canvas) 	----	(DecorView)
		->ViewGroup.dispatchDraw 	----	(DecorView)
			->ViewGroup.drawChild	----	(DecorView)
				-> View.draw(Canvas, ViewGroup, long)	----	(LinearLayout)
					-> View.updateDisplayListIfDirty 	----	(LinearLayout)


```

从 ViewGroup.drawChild 开始，流程从 DecorView 这一级进入到下一层级 LinearLayout。

接着是 LinearLayout 的部分：

```
View.updateDisplayListIfDirty  	----	(LinearLayout)
	->ViewGroup.dispatchDraw  	----	(LinearLayout)
		->ViewGroup.drawChild 	----	(LinearLayout)
			-> View.draw(Canvas, ViewGroup, long) 	----	(FrameLayout)
				-> View.updateDisplayListIfDirty 	----	(FrameLayout)


```

也是从 ViewGroup.drawChild 开始，流程从 LinearLayout 这一级进入到下一层级 FrameLayout。LinearLayout 和 DecorView 流程的唯一区别是，LinearLayout 不需要绘制自身，所以没有 View.draw(Canvas) 这一步。

因此可以看到，实现迭代的关键一步在于 View.draw(Canvas, ViewGroup, long) 方法中进行 View.updateDisplayListIfDirty 的调用。

流程可以总结为：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4525b14025344a891c3fa3df7442993~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1761&h=561&s=85464&e=png&b=fefdfd)

通用流程：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e309bfe8f9c4c18a89e4ca21d90d1a1~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=750&h=642&s=33823&e=png&b=ffffff)

这里简化了从 ThreadedRenderer.draw 方法到 DecorView 的 View.draw(Canvas) 方法的流程。

4.2 绘制 RenderNode
-----------------

根据上一节的分析，看到了在 View 层级结构中，除了顶层 View 没有走 View.draw(Canvas, ViewGroup, long) 之外，其它 View 都走了该方法：

```
    boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
		

        if (drawingWithRenderNode) {
            
            
            renderNode = updateDisplayListIfDirty();
            
        }
        
        

        if (!drawingWithDrawingCache) {
            if (drawingWithRenderNode) {
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                ((RecordingCanvas) canvas).drawRenderNode(renderNode);
            } else {
                
            }
        } else if (cache != null) {
            
        }

        
    }


```

再次回顾一下 View.draw(Canvas, ViewGroup, long) 方法的主要内容。

1）、调用 View.updateDisplayListIfDirty，在 View.updateDisplayListIfDirty 方法中，首先当前 View 调用 RenderNode.beginRecording 拿到一个 RecordingCanvas，然后在 View.onDraw 方法中，将绘制当前 View 的绘制命令记录到这个 RecordingCanvas 中，然后再通过一系列方法调用，最终将这个 RecordingCanvas 传给了当前 View 的子 View 的 View.draw(Canvas, ViewGroup, long) 方法中。

2）、同理得知，在当前 View 的 View.draw(Canvas, ViewGroup, long) 方法传入的 Canvas 对象，便是当前 View 的父 View 通过 RenderNode.beginRecording 拿到一个 RecordingCanvas。

根据以上情况可以知道两点信息：

*   当前 View 的绘制命令，保存在了当前 View 通过 RenderNode.beginRecording 申请的一个 RecordingCanvas。
*   这里调用的 RecordingCanvas.drawRenderNode，RenderNode 是对应的当前 View，RecordingCanvas 却是父 View 通过 RenderNode.beginRecording 申请的一个 RecordingCanvas，而非当前 View 申请的。

知道了以上情况后，就可以继续看 RecordingCanvas.drawRenderNode 的内容了。

### 4.2.1 RecordingCanvas.drawRenderNode

```
    
    @Override
    public void drawRenderNode(@NonNull RenderNode renderNode) {
        nDrawRenderNode(mNativeCanvasWrapper, renderNode.mNativeRenderNode);
    }


```

nCreateDisplayListCanvas 是一个 JNI 函数，对应的是 android_graphics_DisplayListCanvas 中的 android_view_DisplayListCanvas_createDisplayListCanvas 函数。

### 4.2.2 android_view_DisplayListCanvas_drawRenderNode

```
static void android_view_DisplayListCanvas_drawRenderNode(CRITICAL_JNI_PARAMS_COMMA jlong canvasPtr, jlong renderNodePtr) {
    Canvas* canvas = reinterpret_cast<Canvas*>(canvasPtr);
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    canvas->drawRenderNode(renderNode);
}


```

这里的 canvas 就是通过 RederNode.beginRecording 获取的 SkiaRecordingCanvas，继续调用 SkiaRecordingCanvas.drawRenderNode。

### 4.2.3 SkiaRecordingCanvas.drawRenderNode

```
void SkiaRecordingCanvas::drawRenderNode(uirenderer::RenderNode* renderNode) {
    
    mDisplayList->mChildNodes.emplace_back(renderNode, asSkCanvas(), true, mCurrentBarrier);
    auto& renderNodeDrawable = mDisplayList->mChildNodes.back();
    
    
    
    drawDrawable(&renderNodeDrawable);

    
}


```

1）、SkiaRecordingCanvas 的成员变量 mDisplayList 是一个 SkiaDisplayList 类型的指针，SkiaDisplayList 的成员变量 mChildNodes 则是一个存储 RenderNodeDrawable 类型对象的双端队列：

```
    
    std::deque<RenderNodeDrawable> mChildNodes;


```

那么这里是将入参 renderNode 封装到了一个 RenderNodeDrawable 对象中，然后插入到了 SkiaRecordingCanvas 的 SkiaDisplayList 类型的成员变量 mDisplayList 的成员变量 mChildNodes 的队尾。

根据对 RenderNode.endRecording 的分析，我们知道后续 SkiaRecordingCanvas 的 SkiaDisplayList 类型的成员变量 mDisplayList 的所有权会交给 RenderNode，那么可以说，每一个 RenderNode 通过自己所持有的那个 SkiaDisplayList 对象的 mChildNodes，保存了这个 RenderNode 的所有子 RenderNode。

这么一看，SkiaDisplayList 的 mChildNodes 的作用，和 ViewGroup.mChildren 类似，串联起了整个 DisplayList/View 树。

2）、RenderNodeDrawable 定义为：

```
class RenderNodeDrawable : public SkDrawable {
public:
    
    explicit RenderNodeDrawable(RenderNode* node, SkCanvas* canvas, bool composeLayer = true,
                                bool inReorderingSection = false);
    
    
}


```

RenderNodeDrawable 封装了一个 RenderNode 并使它能够被记录到 Skia 绘图命令列表中。

3）、拿到刚刚封装的 RenderNodeDrawable 对象，调用 drawDrawable 函数：

```
    drawDrawable(&renderNodeDrawable);


```

此流程和之前分析的 drawRect 的流程类似，最终是将 drawDrawable 作为绘制指令存入到了 DisplayListData 中，唯一有点区别的是，这里的 DisplayListData 对应的是当前 RenderNode 的父 RenderNode，保存的是绘制父 RenderNode 所需的所有指令，而这里想保存的是绘制当前 RenderNode 对应的 RenderNodeDrawable 的指令。换句话说，每一个 DisplayListData 里保存了绘制其对应的 RenderNode 的所有指令，以及该 RenderNode 的子 RenderNode 的一条绘制 RenderNodeDrawable 的指令。

至于 RenderNodeDrawable 的作用是什么，现在还不得而知，挖个坑。

### 4.2.4 补充 —— 顶层 RenderNode

关于 RenderNode 还漏了一点内容，上面的 RecordingCanvas.drawRenderNode 方法调用的发起都是在 View.draw(Canvas, ViewGroup, long) 中，而对于顶层 VIew 来说，它是没有走 View.draw(Canvas, ViewGroup, long) 的，那么它的 RenderNode 也不是在那里绘制的，而是在 ThreadedRenderer.updateRootDisplayList：

```
	private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
        updateViewTreeDisplayList(view);

        

        if (mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()) {
            RecordingCanvas canvas = mRootNode.beginRecording(mSurfaceWidth, mSurfaceHeight);
            try {
                

                canvas.enableZ();
                canvas.drawRenderNode(view.updateDisplayListIfDirty());
                canvas.disableZ();

                
            } finally {
                mRootNode.endRecording();
            }
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }


```

我们前面那么说的那么多，分析都是 ThreadedRenderer.updateViewTreeDisplayList 的内容，该方法根据 View 树，从上到下构建了为 RenderNode 为节点的 DisplayList 树，唯一漏了 RootRenderNode，这里的 ThreadedRenderer.updateRootDisplayList 方法的剩余部分正好补充了 RootRenderNode 的部分。

### 4.2.5 小结

其实这一节内容很少，没啥好总结的，只是分析了这么多后，发现 SkiaDisplayList 在很多地方都起到了非常关键的作用：

*   存储绘制命令：每个 View 在绘制时，相关绘制是保存到了 SkiaDisplayList 的 DisplayListData 类型的成员变量中，这时 SkiaDisplayList 由 SkiaRecordingCanvas 所有。结束绘制后，该 SkiaDisplayList 的所有权转移到了该 View 对应的 RenderNode，也就是说绘制命令最终保存在了 RenderNode 中。
    
*   构建 DisplayList 树：对每一个 View 的绘制调用请求在 View 树中从上到下进行分发时，会相应的在每个 View 绘制结束后，调用 RecordingCanvas.drawRenderNode，从而将每一个 View 对应的 RenderNode 都添加到父 RenderNode 中，这一步是通过将 RenderNode 封装为 RenderNodeDrawable 然后保存到 SkiaDisplayList 的 mChildNodes 队列中实现的。
    

DisplayList 树中的每一个 RenderNode 节点，背后都是 SkiaDisplayList 在支持。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb2905f451d840cd8636fb656359717a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1401&h=671&s=78297&e=png&b=fffafa)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/132327216?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/dcbc2700214b4ef4b6108d9fb231e210.jpeg#pic_center)

该文章为窗口层级结构系列文章的总结，重新回看这方面内容的时候我自己也有了一些新的感悟，希望通过本次总结能让大家再次对窗口有一个全面的认识。

一般来说，屏幕上最起码包含三个窗口，StatusBar 窗口、Activity 窗口以及 NavigationBar 窗口：

![](https://img-blog.csdnimg.cn/3927454c31d54f77b7cecad71af72c8b.png#pic_center)

我们想要了解窗口，可以按照从小到大，从表到里的顺序进行：

1）、微观角度，探究单个窗口内部的组成。

2）、宏观角度，探究 WMS 如何管理屏幕上显示的诸多窗口。

3）、底层角度，探究 SurfaceFlinger 如何管理窗口，和 WMS 在这方面有何联系。

至于窗口，我们先理解它为一块在屏幕上可以显示图像的区域。

1 微观下的 Window —— View 层级结构
--------------------------

窗口本身作为 View 对象的容器只是一个抽象的概念，真正可见的则是 View 对象，那么我们探究单个窗口的表现形式，其实就是探究组成窗口的 View 层级结构。

这里我们写一个单选按钮组：

```
<RadioGroup
        android:id="@+id/radio_group"
        android:layout_width="300dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="50dp"
        android:layout_marginStart="20dp"
        android:background="@color/material_dynamic_neutral70">
        <RadioButton
            android:id="@+id/radio_button1"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="A" />
        <RadioButton
            android:id="@+id/radio_button2"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="B" />
        <RadioButton
            android:id="@+id/radio_button3"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="C" />
    </RadioGroup>
```

看起来是这样子：

![](https://img-blog.csdnimg.cn/b6901ab2cfc74af6a685378cf071a87b.png#pic_center)

这已经是一个简单的 View 层级结构了，父 View 为 RadioGroup，其中有 3 个 RadioButtion 的子 View，那么它的层级结构就是：

![](https://img-blog.csdnimg.cn/c6b58ef60c7c4f4ba6d311f451ce2d2a.png#pic_center)

我之前习惯用树状图的形式：

![](https://img-blog.csdnimg.cn/222c3822781542a0b16098d67533d1f9.png#pic_center)

不过似乎还是第一种表现形式更好，能够更加直观的反映上下级的关系，后面描述层级结构的时候我仍然用树状图的一些叫法。

Android 对此层级结构的建立提供了哪些支持呢？

### 1.1 每一个 UI 元素都是一个 View

比如这里的单选按钮 RadioButton 类，它的继承关系为：

```
RadioButtion -> CompoundButton -> Button -> TextView -> View
```

RadioGroup 类的继承关系为：

```
RadioGroup -> LinearLayout -> ViewGroup -> View
```

就像每一个类的基类是 Object 一样，每一个用来组成 Activity 界面的 UI 元素，其基类都是 View 类，因此我个人喜欢称这些由 UI 元素搭建的组织架构为 View 层级结构，层级结构里的每一个节点都是一个 View。

### 1.2 父 View 和子 View

以刚刚的单选按钮组为例，这里是一个 RadioGroup 包含了 3 个 RadioButtion，父 View 为 RadioGroup，子 View 为 3 个 RadioButtion。

回看 RadioGroup，它是一种特殊的 View，继承自 ViewGroup，ViewGroup 顾名思义，就是一种能够包含其它 View 的特殊 View，它有一个 View 数组类型的成员变量 mChildren：

```
// frameworks\base\core\java\android\view\ViewGroup.java

	// Child views of this ViewGroup
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P)
    private View[] mChildren;
```

用来保存它容纳的子 View。

既然父 View 有一个 mChildren 用来拿到子 View 对象，那么子 View 也应该有一个成员变量用来保存父 View 的引用，实际上也的确如此，View 中也定义了一个 ViewParent 类型的成员变量 mParent 用来指向当前 View 的父 View：

```
// frameworks\base\core\java\android\view\View.java

	/**
     * The parent this view is attached to.
     * {@hide}
     *
     * @see #getParent()
     */
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P)
    protected ViewParent mParent;
```

ViewParent 是一个接口类，定义一个能够作为父 View 的类的通用接口，这里看到 ViewGroup 就实现了这个接口类：

```
// frameworks\base\core\java\android\view\ViewGroup.java

public abstract class ViewGroup extends View implements ViewParent, ViewManager
```

那么为什么不让 View 来实现 ViewParent 接口，而让 ViewGroup 来实现呢？不难想到，对于参与建立 View 层级结构的 View 来说，其实可以分为两类，一类就像 RadioButtion 一样，它们是 View 层级结构中的最小单位，或者叫做 View 层级结构中的叶节点，无法作为父 View 再去容纳其它子 View。而对于 RadioGroup，它们作为父 View 可以容纳其它子 View，在 View 层级结构中充当中间节点（当然也可以作为叶节点）。

最后说下如何向 ViewGroup 中添加子 View，用的是 ViewGroup.addChild 方法：

```
/**
     * Adds a child view with the specified layout parameters.
     *
     * <p><strong>Note:</strong> do not invoke this method from
     * {@link #draw(android.graphics.Canvas)}, {@link #onDraw(android.graphics.Canvas)},
     * {@link #dispatchDraw(android.graphics.Canvas)} or any related method.</p>
     *
     * @param child the child view to add
     * @param index the position at which to add the child or -1 to add last
     * @param params the layout parameters to set on the child
     */
    public void addView(View child, int index, LayoutParams params)
```

最终会将 child 参数代表的子 View 加入到当前 VIewGroup.mChildren 中，同时将子 View 的 mParent 指向当前 ViewGroup。

LayoutInflater 解析 xml 文件，其实也是实例化 xml 中定义的 View，然后通过 ViewGroup.addChild 方法将子 View 添加到父 View 中，以这种方式将 xml 文件中定义的 View 层级结构建立起来。

### 1.3 根 View

对于所有组成同一个 Activity 界面的 UI 元素，它们都在同一个层级结构中，自然它们都有一个共同的根 View，那这个根 View 是谁？

我们新建一个 Activity，一般来说都在其 onCreate 方法中去加载其布局，比如像这样：

```
@Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.activity_pip);
    }
```

那通过解析这个 **R.layout.activity_pip** 对应的 xml 文件所生成的 View 就是根 View 吗，仍然拿这里的 demo 进行验证，我自己定义的 actiivty_pip.xml 是一个 LInearLayout，里面包含了两个单选按钮组。需要注意的是这个 LinearLayout 有一个不为 “match_parent” 的自定义高度，并且设置了一个背景色，其层级结构为：

![](https://img-blog.csdnimg.cn/f198a46bdba749aab131f1a2ec91eb06.png#pic_center)

看下实际情况：

![](https://img-blog.csdnimg.cn/1e1003cb2d7a4440bd915cf1a88213fc.png#pic_center)

对于它没有覆盖到的区域，仍有一个白色背景，说明这个 LinearLayout 并不是根 View。

如果想要找到根 View，就需要去跟踪 Activity.setContentView 流程，这个流程我之前有总结过文档，这里就不再具体叙述，直接放结论，是 DecorView。

### 1.4 View 层级结构的生成

看下 View 层级结构生成的流程，即 Activity.setContentView 方法流程，只说关键部分。

1）、创建 DecorView 实例。

![](https://img-blog.csdnimg.cn/7b5840611a204ddf93d6b66773e4a66b.png#pic_center)

2）、根据 Activity 请求的 feature（比如是否需要标题、图标）来决定 DecorView 的直接子 View 的布局类型，我这里的是 **R.layout.screen_simple**：

```
// frameworks/base/core/res/res/layout/screen_simple.xml

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

具体为一个 LinearLayout 包含一个 ViewStub 和 FrameLayout。

接着将这个 xml 实例化为一个 View 层级结构，将该层级结构的根节点作为子 View 加入到 DecorView 中：

![](https://img-blog.csdnimg.cn/312ecb4ecc3e49459efbc9ce997fd161.png#pic_center)

3）、接着通过 LayoutInflater.inflate 解析我们通过 Activity.setContentView 传入的 xml 文件，指定其父 View 为上一步解析的 screen_simple.xml 中定义的 ID 为：

```
/**
     * The ID that the main layout in the XML layout file should have.
     */
    public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;
```

的 View，将我们自定义的布局作为子 View 添加到该 View 上，生成最终的层级结构：

![](https://img-blog.csdnimg.cn/5dbd4d34a34048cd84ac5f42a6647480.png#pic_center)

用”Legacy Layout Inspector“插件看是：

![](https://img-blog.csdnimg.cn/f8b18704f03e425bbda609f6f9f1515f.png#pic_center)

4）、回看我们之前的 Activity 界面：

![](https://img-blog.csdnimg.cn/1e1003cb2d7a4440bd915cf1a88213fc.png#pic_center)

能看到顶部和底部分别还有一个区域，如果我们的 Activity 不要求隐藏状态栏和导航栏，那么会创建两个 View：

*   状态栏背景区域，com.android.internal.R.id.statusBarBackground。
*   导航栏背景区域，com.android.internal.R.id.navigationBarBackground。

当状态栏窗口和导航栏窗口盖在我们的 Activity 窗口上时，这两个区域就是它们的背景。

DecorView 将这两个 View 作为子 View 加入其中，和存放我们自定义布局的 LinearLayout（内部层级被折叠）同级，如图所示：

![](https://img-blog.csdnimg.cn/042e6eeee26b4f3baa23f6cfc1c90acc.png#pic_center)

并且对 LinearLayout 设置内外边距，从而避免我们的自定义区域伸延到这些为系统窗口预留的背景区域中，以防我们期望显示的内容被这些系统窗口覆盖。

### 1.5 ViewRootImpl —— 实际的根节点？

ViewRootImpl，不管是开发 App 还是做 Framework，都少不了跟它打交道，从名字上来看，它似乎是作为根 View 的角色所存在，那么事实上如何呢？

不管是 Activity 窗口还是非 Activity 窗口，最终都要通过 ViewRootImpl.setView 向 WMS 注册窗口：

```
/**
     * We have one child
     */
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
        synchronized (this) {
            if (mView == null) {
                mView = view;

                // ......
                
                try {
                    // ......
                    res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId,
                            mInsetsController.getRequestedVisibilities(), inputChannel, mTempInsets,
                            mTempControls);
                    // ......
                }

                // ......

                view.assignParent(this);
                
                // ......
            }
        }
    }
```

值得注意的是以下几点：

1）、将 View 类型的传参 view 赋值给 View 类型的成员变量 mView，这里的传参 view 是一个窗口的 View 层级结构的顶级 View，对于 Activity 窗口来说，就是 DecorView，而对于非 Activity 窗口，如状态栏和导航栏，这些窗口的 View 层级结构完全由它们自定义，不会像 Activity 窗口那样再由 Framework 给它们外面再套几个 layout，这里的传参 view 就是这些窗口的自定义 View 层级结构的顶级 View。

后续 ViewRootImpl 就可以通过其成员变量 mView 拿到顶级 View。

2）、接着通过 IWindowSession.addToDisplayAsUser 向 WMS 添加窗口。

3）、最后调用 View.assignParent 方法，将传参 View 的 mParent 指向当前 ViewRootImpl：

```
@UnsupportedAppUsage
    void assignParent(ViewParent parent) {
        if (mParent == null) {
            mParent = parent;
        }
        // .......
    }
```

那么对于 Activity 窗口来说，其 DecorView 还有一个父 View，ViewRootImpl？

但是再看 ViewRootImpl 的定义：

```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks,
        AttachedSurfaceControl {
```

ViewRootImpl 并不是一个 View 对象，因此它并没有一个可视的内容。

ViewRootImpl 也不是一个 ViewGroup，因此没有 mChildren 来保存其子 View，所以它才新建了一个 mView 成员变量来保存其唯一的子 View，对于 Activity 窗口来说，ViewRootImpl 的唯一子 View 就是 DecorView。

那么把 ViewRootImpl 视为真正的根 View，虽也未尝不可，但是需要注意，我们之前说的以 DecorView 为根节点的 View 层级结构，它的每一个节点都是一个可视的 View，而 ViewRootImpl 只是一个抽象的概念，相比于 DecorView 这种 “实体” 根节点，它是一个虚拟的根节点。这不禁让人思考，ViewRootImpl 为啥要这么取名？

那就要看看 ViewRootImpl 扮演了怎样的角色，上面分析过，ViewRootImpl 保存了 View 层级结构的顶级 View 的引用（为数不多的保存顶级 View 引用的地方之一），那么很多需要遍历 View 层级结构的流程都会从 ViewRootImpl 发起。比如，经典的 measure 流程，便是通过 ViewRootImpl.performMeasure 方法发起：

```
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        if (mView == null) {
            return;
        }
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```

ViewRootImpl 先调用其直接子 View 的 measure 方法，再由该子 View 调用子子 View 的 measure 方法，依次类推，最终整个 View 层级结构中的所有 View 的 measure 方法都能够得到调用。

除此之外，layout 流程和 draw 流程，Input 事件的传递以及 Insets 的更新，大部分需要遍历 View 层级结构的流程，起点都是在 ViewRootImpl。

尽管如此，对于 Activity 窗口，我个人仍然倾向于称 DecorView 为的 View 层级结构的根 View。

### 1.6 Freeform 的实现

这里根据 Freeform 的实现方式，来验证一下我们上面的分析内容。

这里只关心 Freeform 是如何为 Activity 添加包含最大化按钮和退出按钮的标题栏。

如果让我们自己写一个这个形式的界面，也非常容易，只需要一个 LInearLayout 包含两个 Button 即可：

![](https://img-blog.csdnimg.cn/bd67bd12eb164800886d96252c2da47f.png#pic_center)

但是如果想要为每一个进入 Freeform 的 Activity 都自动设置这样一个标题栏，就得靠 Framework 去统一处理。

google 的做法如下：

1）、实例化一个 DecorCaptionView，它通过 xml 文件定义：

```
<com.android.internal.widget.DecorCaptionView xmlns:android="http://schemas.android.com/apk/res/android"
         android:orientation="vertical"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:descendantFocusability="beforeDescendants" >
     <LinearLayout
             android:id="@+id/caption"
             android:layout_width="match_parent"
             android:layout_height="wrap_content"
             android:gravity="end"
             android:background="@drawable/decor_caption_title"
             android:focusable="false"
             android:descendantFocusability="blocksDescendants" >
         <Button
                 android:id="@+id/maximize_window"
                 android:layout_width="32dp"
                 android:layout_height="32dp"
                 android:layout_margin="5dp"
                 android:padding="4dp"
                 android:layout_gravity="center_vertical|end"
                 android:contentDescription="@string/maximize_button_text"
                 android:background="@drawable/decor_maximize_button_dark" />
         <Button
                 android:id="@+id/close_window"
                 android:layout_width="32dp"
                 android:layout_height="32dp"
                 android:layout_margin="5dp"
                 android:padding="4dp"
                 android:layout_gravity="center_vertical|end"
                 android:contentDescription="@string/close_button_text"
                 android:background="@drawable/decor_close_button_dark" />
     </LinearLayout>
 </com.android.internal.widget.DecorCaptionView>
```

有一个 LInearLayout，包含了两个按钮：

![](https://img-blog.csdnimg.cn/2a03308d69964d2e8a80640090a95947.png#pic_center)

2）、实例化一个 DecorCaptionView 后，我们原先的 Activity 窗口层级结构是（忽略具体的细节部分）：

![](https://img-blog.csdnimg.cn/2141e9aa05874891882de72cac005ba7.png#pic_center)

接着 DecorView 将 DecorCaptionView 作为子 View 添加进来，接着将原先的唯一子 View 移除，并且将其作为子 View 添加到 DecorCaptionView 中，即：

![](https://img-blog.csdnimg.cn/c3a6036cefce41c5842eb526d858a243.png#pic_center)

2 宏观下的 Window —— WindowContainer 层级结构
-------------------------------------

上一节以微观的角度去探究一个窗口的组成，即 View 层级结构。

这一节以宏观的角度，把窗口当作 WMS 窗口体系下的最小单位，去探究 WMS 是如何管理窗口的。

### 2.1 窗口 —— WindowState 类

```
/** A window in the window manager. */
public class WindowState extends WindowContainer<WindowState> implements
    WindowManagerPolicy.WindowState, InsetsControlTarget {
```

在 WMS 窗口体系中，一个 WindowState 对象就代表了一个窗口。

这里看到，WindowState 继承自 WindowContainer。

### 2.2 窗口容器类 —— WindowContainer 类

```
/**
 * Defines common functionality for classes that can hold windows directly or through their
 * children in a hierarchy form.
 * The test class is {@link WindowContainerTests} which must be kept up-to-date and ran anytime
 * changes are made to this class.
 */
class WindowContainer<E extends WindowContainer> extends ConfigurationContainer<E>
        implements Comparable<WindowContainer>, Animatable, SurfaceFreezer.Freezable {
```

WindowContainer 定义了能够直接或者间接以层级结构的形式持有窗口的类的通用功能。

从类的定义和名称，可以看到 WindowContainer 是一个容器类，可以容纳 WindowContainer 及其子类对象。如果有一个容器类想作为 WindowState 的容器，那么这个容器类需要继承 WindowContainer 或其子类。

继续看下，WindowContainer 为了能够作为一个容器类，提供了哪些支持：

```
/**
     * The parent of this window container.
     * For removing or setting new parent {@link #setParent} should be used, because it also
     * performs configuration updates based on new parent's settings.
     */
    private WindowContainer<WindowContainer> mParent = null;

	// ......

    // List of children for this window container. List is in z-order as the children appear on
    // screen with the top-most window container at the tail of the list.
    protected final WindowList<E> mChildren = new WindowList<E>();
```

*   mParent，保存的是当前 WindowContainer 的父 WindowContainer 的引用，可以类比于 View.mParent。
*   mChidlren，保存的是当前 WindowContainer 作为一个容器，所包含的子 WindowContainer，可以类比于 ViewGroup.mChildren。它是一个列表，列表的顺序反映了子容器在 Z 轴上的顺序，越靠近队尾层级越高。

这两个成员变量为生成 WindowContainer 层级结构提供了支持，就像根据 View.mParent 和 ViewGroup.mChildren 创建 View 层级结构那样。

容器这个概念非常重要，理解了它才能理解 WindowContainer 层级结构的组织形式。为了理解容器这个概念，可以把 WindowContainer 比作一个堆栈，这里举几个例子方便理解：

1）、当前 WindowContainer 已经连续添加了两个子容器，WindowContainer_A 以及 WindowContainer_B，继续添加 WindowContainer_C，就可以看作是像当前堆栈栈顶入栈一个 WindowContainer_C：

![](https://img-blog.csdnimg.cn/9b70897173d54851a2b588a865e0322c.png#pic_center)

这里用堆栈来表示一个 WindowContainer 也是因为堆栈比较能够直观的反映 WindowContainer 内部的 Z 轴顺序，WindowContainer 层级结构和 View 层级结构是有相似之处的，ViewGroup 也可以看作是 View 的容器，但是我个人感觉 View 层级结构中的 Z 轴这个概念稍微弱了一点，所以之前没有用堆栈的表现形式。

堆栈的表现形式虽好，但是当层级结构稍微复杂一点的话，画起来就非常麻烦，而且呈现出来的形式也不够直观，这里转为我个人比较习惯的层级结构图：

![](https://img-blog.csdnimg.cn/4f4e662814f04aa1b68e61d2fc6637e1.png#pic_center)

这个图其实也能反映层级顺序，这里是 WindowContainer_C 高于 WindowContainer_B 高于 WindowContainer_A。

2）、稍微扩展一点，当前 WindowContainer 有两个子 WindowContainer，同时这两个子 WindowContainer 也分别有各自的子 WindowContainer。

![](https://img-blog.csdnimg.cn/bc61176057a94f3d88540cef4453fc4f.png#pic_center)

转为层级结构图：

![](https://img-blog.csdnimg.cn/c3c6b65d6ce642249793d7a11988f028.png#pic_center)

有两点需要注意：

*   位于同一个父容器中的 WindowContainer 可以直接进行比较。在 WindowContainer_B 中，很明显的能够看到 WindowContainer_G 高于 WindowContainer_F 高于 WindowContainer_E。
*   位于不同父容器的两个 WindowContainer 不能直接进行比较。这里如果我们想比较 WindowContainer_G 和 WindowContainer_D 谁的层级高，就需要比较它们两个的父容器谁的层级高。这里看到 WindowContainer_A 和 WindowContainer_B 都位于同一个父容器中，所以可以得出 WindowContainer_B 高于 WindowContainer_A，进而 WindowContainer_G 高于 WindowContainer_D。如果 WindowContainer_A 和 WindowContainer_B 也没有处于同一个父容器中，那么就得继续沿着层级结构图往上找，直到找出处于同一个节点下的两个父容器。

此时回看 WindowState 类的定义：

```
public class WindowState extends WindowContainer<WindowState>
```

有了新的发现，即 WindowState 本身也是一个容器，不过这里指明了 WindowState 作为容器类可以持有的容器类型限定为 WindowState 类型，即 WindowState 作为父容器，其中只能存放 WindowState 类型的 WindowContainer。

事实上也的确如此，Android 是有父窗口和子窗口的概念的，比如打开一个 Message 的弹窗：

![](https://img-blog.csdnimg.cn/ee201532b16047b290126793897fee16.jpeg#pic_center)

查看 PopupWindow 的信息：

```
Window #6 Window{193e28f u0 PopupWindow:b98815a}:
    // ......
    mParentWindow=Window{bfc77c9 u0 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity} mLayoutAttached=true
```

父窗口为 Activity 对应的窗口。

再查看层级信息：

```
#0 6b8748a com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
           #0 3c3dfc1 PopupWindow:a760a0c type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

转为 WindowContainer 层级结构图为：

![](https://img-blog.csdnimg.cn/07c656b8c71c4edbaba921a2b6f22fdf.png#pic_center)

这里表示 WindowState 名字的形式一般是哈希值加上包名或类名或窗口名称。

### 2.3 WindowState 的容器 —— WindowToken、ActivityRecord 和 WallpaperWindowToken

WindowContainer 是能够直接或者间接持有 WindowState 的容器类的通用基类，间接持有 WindowState 的容器类暂且不提，肯定很多，那么可以直接持有 WindowState 的类有哪些呢？

答案是 WindowToken（除了上面提到的子窗口的概念），并且 WindowToken 衍生了两个子类，ActivityRecord 和 WallpaperWindowToken，用来对窗口进行更精细的分类。

#### 2.3.1 App 窗口和非 App 窗口

首先窗口可以分为 App 窗口和非 App 窗口：

*   App 窗口，当 App 创建 Activity、Dialog 或者 PopupWindow 的时候，系统会为自动为其创建一个对应的窗口，是普通的 App 窗口。
*   非 App 窗口，是系统为了特定目的而使用的特殊类型的窗口，可以理解为系统窗口，如状态栏、导航栏以及壁纸窗口那些。

具体如何划分，则是看窗口的类型，即定义在 WindowManager.LayoutParams 中的成员变量 type：

```
@WindowType
        public int type;
```

窗口类型从 FIRST_APPLICATION_WINDOW（1）到 LAST_APPLICATION_WINDOW（99）的窗口为 App 窗口。

窗口类型从 FIRST_SYSTEM_WINDOW（2000）到 LAST_SYSTEM_WINDOW（2999）的窗口为系统窗口。

其实除此之外只有一种特殊类型的窗口，子窗口，窗口类型从 FIRST_SUB_WINDOW（1000）到 LAST_SUB_WINDOW（1999）。这个类型的窗口必须依附于一个父窗口，比如刚刚提到的 PopupWindow。这类窗口属于 App 窗口还是非 App 窗口，完全看其父窗口归于哪类。

那么以此分类，非 App 窗口的容器为 WindowToken，App 窗口的容器为 ActivityRecord（子窗口的容器为父窗口，WindowState）。

#### 2.3.2 进一步划分

非 App 窗口中一种特殊的窗口，壁纸窗口，由于其是作为 Activity 的背景存在的，因此它的层级应该始终低于 App 窗口。而除壁纸窗口之外的非 App 窗口，层级则高于 App 窗口。

那么我们结合窗口类型以及层级关系，可以对窗口进一步划分：

1）、App 之上的窗口，父容器为 WindowToken，如 StatusBar：

```
#0 WindowToken{ef15750 android.os.BinderProxy@4562c02} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
        #0 d3cbd49 StatusBar type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
```

![](https://img-blog.csdnimg.cn/272d71b138824612b263ca03b4727d22.png#pic_center)

2）、App 窗口，父容器为 ActivityRecord，如 Message 相关 Activity 的窗口：

```
#0 ActivityRecord{bc0999c u0 com.google.android.apps.messaging/.main.MainActivity} t14} type=standard mode=fullscreen override-mode=undefined requested-bou
nds=[0,0][0,0] bounds=[0,0][720,1612]
          #0 adb45c7 com.google.android.apps.messaging/com.google.android.apps.messaging.main.MainActivity type=standard mode=fullscreen override-mode=undefined req
uested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

![](https://img-blog.csdnimg.cn/1906fa1a9cfa43c19ed04075621ed61f.png#pic_center)

3）、App 之下的窗口，父容器为 WallpaperWindowToken，如 ImageWallpaper 窗口：

```
#0 WallpaperWindowToken{145ca49 token=android.os.Binder@a727850} type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=
[0,0][720,1612]
         #0 ae9c947 com.android.systemui.ImageWallpaper type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

![](https://img-blog.csdnimg.cn/76c50ba67ae7423ea7dce2daf9d3a856.png#pic_center)

### 2.4 ActivityRecord 的容器 —— Task

按理说这一步应该继续分析 WindowToken 的父容器了，但是 ActivityRecord 作为一种特殊的 WindowToken，它自己有一个特殊的父容器，Task。

Task 类的定义为：

```
class Task extends WindowContainer<WindowContainer> {
```

再次温习一下 google 关于 Task 的说明：

> 任务（Task）是用户在执行某项工作时与之互动的一系列 Activity 的集合。这些 Activity 按照每个 Activity 打开的顺序排列在一个返回堆栈中。例如，电子邮件应用可能有一个 Activity 来显示新邮件列表。当用户选择一封邮件时，系统会打开一个新的 Activity 来显示该邮件。这个新的 Activity 会添加到返回堆栈中。如果用户按**返回**按钮，这个新的 Activity 即会完成并从堆栈中退出。
> 
> 大多数 Task 都从 Launcher 上启动。当用户轻触 Launcher 中的图标（或主屏幕上的快捷方式）时，该应用的 Task 就会转到前台运行。如果该应用没有 Task 存在（应用最近没有使用过），则会创建一个新的 Task，并且该应用的 “主”Activity 将会作为堆栈的根 Activity 打开。
> 
> 在当前 Activity 启动另一个 Activity 时，新的 Activity 将被推送到堆栈顶部并获得焦点。上一个 Activity 仍保留在堆栈中，但会停止。当 Activity 停止时，系统会保留其界面的当前状态。当用户按**返回**按 钮时，当前 Activity 会从堆栈顶部退出（该 Activity 销毁），上一个 Activity 会恢复（界面会恢复到上一个状态）。堆栈中的 Activity 永远不会重新排列，只会被送入和退出，在当前 Activity 启动时被送入堆栈，在用户使用**返回**按钮离开时从堆栈中退出。因此，返回堆栈按照 “后进先出” 的对象结构运作。图 1 借助一个时间轴直观地显示了这种行为。该时间轴显示了 Activity 之间的进展以及每个时间点的当前返回堆栈。
> 
> ![](https://img-blog.csdnimg.cn/d3b4f35dd21147f89158fc5027d94dad.png#pic_center)
> 
> **图 1.** 有关 Task 中的每个新 Activity 如何添加到返回堆栈的图示。当用户按**返回**按钮时，当前 Activity 会销毁，上一个 Activity 将恢复。
> 
> 如果用户继续按**返回**，则堆栈中的 Activity 会逐个退出，以显示前一个 Activity，直到用户返回到主屏幕（或 Task 开始时运行的 Activity）。移除堆栈中的所有 Activity 后，该 Task 将不复存在。

WMS 中的 Task 类的概念和以上说明基本一致，用来管理 ActivityRecord。

还是上面的 Message，情况是：

```
#3 Task=14 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
         #0 ActivityRecord{bc0999c u0 com.google.android.apps.messaging/.main.MainActivity} t14} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
          #0 adb45c7 com.google.android.apps.messaging/com.google.android.apps.messaging.main.MainActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

![](https://img-blog.csdnimg.cn/c987377233a94a27b6295df8a04f6c82.png#pic_center)

接着再启动 Message 的另外一个界面：

```
#3 Task=14 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
         #1 ActivityRecord{357651e u0 com.google.android.apps.messaging/.ui.appsettings.ApplicationSettingsActivity} t14} type=standard mode=fullscreen override-mod
e=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
          #0 e611691 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.appsettings.ApplicationSettingsActivity type=standard mode=fullscreen ov
erride-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
         #0 ActivityRecord{bc0999c u0 com.google.android.apps.messaging/.main.MainActivity} t14} type=standard mode=fullscreen override-mode=undefined requested-bou
nds=[0,0][0,0] bounds=[0,0][720,1612]
          #0 adb45c7 com.google.android.apps.messaging/com.google.android.apps.messaging.main.MainActivity type=standard mode=fullscreen override-mode=undefined req
uested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

![](https://img-blog.csdnimg.cn/789f4717483842448400f7dfc74693fb.png#pic_center)

新的 ActivityRecord 被置于栈顶，盖在之前的 ActivityRecord 上。

#### Task 的嵌套

Task 是支持嵌套的，从 Task 类的定义也可以看出来（Task 是 WindowContainer 的容器，而非只能容纳 ActivityRecord）。

1）、看下当只有一个 Launcher 的情况：

```
#2 Task=1 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         #0 Task=50 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
          #0 ActivityRecord{b864ccc u0 com.tcl.android.launcher/.uioverrides.TclQuickstepLauncher} t50} type=home mode=fullscreen override-mode=undefined requested-
bounds=[0,0][0,0] bounds=[0,0][1080,2400]
```

![](https://img-blog.csdnimg.cn/f52d95ed153c4e71a70adc7c2849ac1a.png#pic_center)

这里的示意图忽略了 ActivityRecord 的下一级，WindowState。

只看到这里可能无法理解 Task 嵌套的意义何在。

2）、接着启动 Message：

```
#1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
        #3 Task=61 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         #0 ActivityRecord{fac1d1c u0 com.google.android.apps.messaging/.ui.ConversationListActivity} t61} type=standard mode=fullscreen override-mode=undefined req
uested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
        #2 Task=1 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         #0 Task=50 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
          #0 ActivityRecord{b864ccc u0 com.tcl.android.launcher/.uioverrides.TclQuickstepLauncher} t50} type=home mode=fullscreen override-mode=undefined requested-
bounds=[0,0][0,0] bounds=[0,0][1080,2400]
```

![](https://img-blog.csdnimg.cn/d10a20a7d9dc4fb5aaa48eac002b92da.png#pic_center)

此时并没有发生 Task 的嵌套，新启动的 Message 对应的 Task#61 并没有启动到 Task#1 中，而是并入了 DefaultTaskDisplayArea 中。

3）、如果我们此时再切换到 NexusLauncher：

```
#5 Task=1 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
         #1 Task=47 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
          #0 ActivityRecord{31ce9e6 u0 com.google.android.apps.nexuslauncher/.NexusLauncherActivity t47} type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
         #0 Task=50 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
          #0 ActivityRecord{b864ccc u0 com.tcl.android.launcher/.uioverrides.TclQuickstepLauncher} t50} type=home mode=fullscreen override-mode=undefined requested-
bounds=[0,0][0,0] bounds=[0,0][1080,2400]
```

![](https://img-blog.csdnimg.cn/08802d14e71849adaa619c57f0dabec5.png#pic_center)

看到新启动的 Task#47，嵌套到了 Task#1 中，也就是作为子节点，添加到了容器 Task#1 中。

为什么刚刚启动的 Message 的 Task，Task#61，没有加入到 Task#1 中，而 NexusLauncher 对应的 Task#47，却加入到了 Task#1 中呢？这里要说下 ActivityRecord 的几种类型：

```
/** The current activity type of the configuration. */
    private @ActivityType int mActivityType;

    /** Activity type is currently not defined. */
    public static final int ACTIVITY_TYPE_UNDEFINED = 0;
    /** Standard activity type. Nothing special about the activity... */
    public static final int ACTIVITY_TYPE_STANDARD = 1;
    /** Home/Launcher activity type. */
    public static final int ACTIVITY_TYPE_HOME = 2;
    /** Recents/Overview activity type. There is only one activity with this type in the system. */
    public static final int ACTIVITY_TYPE_RECENTS = 3;
    /** Assistant activity type. */
    public static final int ACTIVITY_TYPE_ASSISTANT = 4;
    /** Dream activity type. */
    public static final int ACTIVITY_TYPE_DREAM = 5;
```

大部分普通 App 的 Activity，都是 ACTIVITY_TYPE_STANDARD，如注释所说，标准的 Activity 类型， 没有什么特别的。

而对于 Home/Launcher 类型的 App，它们的 Activity 都是 ACTIVITY_TYPE_HOME 类型，对于这类 App 对应的 Task，系统会在开机的时候创建一个该类型的 Task，作为该类型的 RootTask，用来统一管理该类型的 Task。因此，如果我们又启动了第三个 Launcher，该 Launcher 对应的 Task，也会作为子节点添加到 Task#1 中。

但是不会出现一个 Task 的子容器既有 Task，又有 ActivityRecord 的情况（至少现在不会）。

### 2.5 Task 的容器 —— TaskDisplayArea

```
/**
 * {@link DisplayArea} that represents a section of a screen that contains app window containers.
 *
 * The children can be either {@link Task} or {@link TaskDisplayArea}.
 */
final class TaskDisplayArea extends DisplayArea<WindowContainer>
```

TaskDisplayArea 代表了屏幕上的一个包含 App 类型的 WindowContainer 的区域。它的子节点可以是 Task，或者是 TaskDisplayArea。但是目前在代码中，我看到创建 TaskDisplayArea 的地方只有一处，该处创建了一个名为 “DefaultTaskDisplayArea” 的 TaskDisplayArea 对象，除此之外并没有发现其他地方有创建 TaskDisplayArea 对象的地方，自然也没有找到有关 TaskDisplayArea 嵌套的痕迹。

因此目前可以说，TaskDisplayArea 存放 Task 的容器。

在手机的近期任务列表界面将所有 App 都清掉后，查看一下此时的 TaskDisplayArea 情况：

```
#0 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #5 Task=1 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 Task=8 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
          #0 ActivityRecord{5e73ff4 u0 com.tcl.android.launcher/.Launcher t8} type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
           #1 ca4f5c4 com.tcl.android.launcher/com.tcl.android.launcher.Launcher type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
           #0 5784a50 com.tcl.android.launcher/com.tcl.android.launcher.Launcher type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #4 Task=2 type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #3 Task=3 type=undefined mode=multi-window override-mode=multi-window requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #2 Task=4 type=undefined mode=multi-window override-mode=multi-window requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #1 Task=5 type=undefined mode=split-screen-primary override-mode=split-screen-primary requested-bounds=[0,0][950,1200] bounds=[0,0][950,1200]
        #0 Task=6 type=undefined mode=split-screen-secondary override-mode=split-screen-secondary requested-bounds=[970,0][1920,1200] bounds=[970,0][1920,1200]
```

![](https://img-blog.csdnimg.cn/036227b4a5544294826034c11b73e36d.png#pic_center)

预期应该是 DefaultTaskDisplayArea 的子容器，只有 Launcher 应用对应的 Task#1，但是却多 Task#2、Task#3、Task#4、Task#5 和 Task#6 这几个 Task，并且这几个 Task 并没有持有任何一个 ActivityRecord 对象。正常来说，App 端调用 startActivity 后，WMS 会为新创建的 ActivityRecord 对象创建 Task，用来存放这个 ActivityRecord。当 Task 中的所有 ActivityRecord 都被移除后，这个 Task 就会被移除，比如 Message 对应的 Task。

但是对于 Task#1 之类的 Task 来说并不是这样。这些 Task 是 WMS 启动后就由 TaskOrganizerController 创建的，这些 Task 并没有和某一具体的 App 联系起来，因此当它里面的子 WindowContainer 被移除后，这个 Task 也不会被销毁。比如分屏 Task#5 和 Task#6：

```
#1 Task=5 type=undefined mode=split-screen-primary override-mode=split-screen-primary requested-bounds=[0,0][950,1200] bounds=[0,0][950,1200]
        #0 Task=6 type=undefined mode=split-screen-secondary override-mode=split-screen-secondary requested-bounds=[970,0][1920,1200] bounds=[970,0][1920,1200]
```

这两个 Task 由 TaskOrganizerController 启动，用来管理系统进入分屏后，需要跟随系统进入分屏模式的那些 App 对应的 Task，也就是说这些 App 对应的 Task 的父容器会从 DefaultTaskDisplayArea 变为 Task#3 和 Task#2。

#### Task 的调度

Task 的调度其实也非常简单，这里拿一个实际的例子说明。

1）、我先从 Launcher 启动 Message，接着通过 adb 启动我自己写的 DemoApp，那么此时这三个 Task 的顺序从高到低分别是：DemoApp -> Message -> Launcher，如图所示：

![](https://img-blog.csdnimg.cn/5eed532a0bad45138a10f878a67ec441.png#pic_center)

2）、接着我在 DemoApp 界面下点击 Back 键，此时的行为是：

![](https://img-blog.csdnimg.cn/5d0b0ec362a44d648670dd93b4501381.png#pic_center)

DemoApp 所在的 Task#64，跑到了 TaskDisplayArea 的栈底，其它 Task 顺序不变，那么最终是 Message 出现在了前台。

这是因为我们重写了 Activity 的 onBackPressed 方法：

```
@Override
    public void onBackPressed() {
        moveTaskToBack(true);
    }
```

当接收到 Back 事件的时候，将 Activity 所在的 Task 移动到最后，具体的行为就是移动到父容器的栈底，也就是 TaskDisplayArea 的栈底。

3）、这里我们更改 onBackPressed 方法的内容：

```
@Override
    public void onBackPressed() {
        Intent intent = new Intent(Intent.ACTION_MAIN);
        intent.addCategory(Intent.CATEGORY_HOME);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivity(intent);
    }
```

在接收到 Back 键时启动 Launcher，那么当我们再次点击 Back 键时，行为是：

![](https://img-blog.csdnimg.cn/cbd108a491b543ffbaf5d26777b759e6.png#pic_center)

看到 Launche 所在的 Task#50，以及 RootTask，Task#1，一起移动到了前台，而其它 Task 的顺序不变。

4）、最后能看到，同样的操作，并且肉眼看到了同样的行为，但是 TaskDisplayArea 内部的 Task 的顺序却不一样：一个是将前台 App 移动到栈底，一个是将后台 App 移动到前台，所以具体问题还是要具体分析。

### 2.6 DisplayArea 层级结构

能够容纳非 App 窗口的父容器为 DisplayArea.Tokens：

```
/**
     * DisplayArea that contains WindowTokens, and orders them according to their type.
     */
    public static class Tokens extends DisplayArea<WindowToken> {
```

容纳 App 窗口的父容器则是 Task，容纳 Task 的则是 TaskDisplayArea，TaskDisplayArea 也是 DisplayArea 的一个子类：

```
final class TaskDisplayArea extends DisplayArea<WindowContainer>
```

那么我们接下来研究的对象就是 DisplayArea。

DisplayArea 本身也非常复杂，有很多子类：

![](https://img-blog.csdnimg.cn/3cbb8dc2303b4f3b88e887eac93e244b.png#pic_center)

这里只说一下比较重要的几个类：

1）、能直接容纳 WindowToken 的容器，除了刚刚说的 DisplayArea.Tokens 类和 TaskDisplayArea（严谨点说 TaskDisplayArea 是 Task 的容器，Task 才是 ActivityRecord 的容器）之外，还有一个 DisplayArea.Tokens 类的子类，定义在 DisplayContent 中的内部类 ImeContainer，专门用来容纳输入法窗口。这三个类由于能够容纳 WindowToken，所以在 DisplayArea 层级结构中作为叶节点存在（最小单位，无法再容纳其它 DisplayArea 对象），就像 WindowState 是作为 WindowContainer 层级结构的最小单位一样。

2）、DisplayContent，代表一个屏幕，作为 DisplayArea 层级结构的根节点所存在。

#### 2.6.1 窗口层级值

我们都知道状态栏窗口的层级是比普通的 App 窗口层级高的，但是为什么是这样的，不知道大家有没有想过。

看一个方法，WindowManagerPolicy.getWindowLayerFromTypeLw：

```
/**
     * Returns the layer assignment for the window type. Allows you to control how different
     * kinds of windows are ordered on-screen.
     *
     * @param type The type of window being assigned.
     * @param canAddInternalSystemWindow If the owner window associated with the type we are
     *        evaluating can add internal system windows. I.e they have
     *        {@link Manifest.permission#INTERNAL_SYSTEM_WINDOW}. If true, alert window
     *        types {@link android.view.WindowManager.LayoutParams#isSystemAlertWindowType(int)}
     *        can be assigned layers greater than the layer for
     *        {@link android.view.WindowManager.LayoutParams#TYPE_APPLICATION_OVERLAY} Else, their
     *        layers would be lesser.
     * @param roundedCornerOverlay {#code true} to indicate that the owner window is rounded corner
     *                             overlay.
     * @return int An arbitrary integer used to order windows, with lower numbers below higher ones.
     */
    default int getWindowLayerFromTypeLw(int type, boolean canAddInternalSystemWindow,
            boolean roundedCornerOverlay) {
        // Always put the rounded corner layer to the top most.
        if (roundedCornerOverlay && canAddInternalSystemWindow) {
            return getMaxWindowLayer();
        }
        if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
            return APPLICATION_LAYER;
        }

        switch (type) {
            case TYPE_WALLPAPER:
                // wallpaper is at the bottom, though the window manager may move it.
                return  1;
            case TYPE_PRESENTATION:
            case TYPE_PRIVATE_PRESENTATION:
            case TYPE_DOCK_DIVIDER:
            case TYPE_QS_DIALOG:
            case TYPE_PHONE:
                return  3;
            case TYPE_SEARCH_BAR:
            case TYPE_VOICE_INTERACTION_STARTING:
                return  4;
            case TYPE_VOICE_INTERACTION:
                // voice interaction layer is almost immediately above apps.
                return  5;
            case TYPE_INPUT_CONSUMER:
                return  6;
            case TYPE_SYSTEM_DIALOG:
                return  7;
            case TYPE_TOAST:
                // toasts and the plugged-in battery thing
                return  8;
            case TYPE_PRIORITY_PHONE:
                // SIM errors and unlock.  Not sure if this really should be in a high layer.
                return  9;
            case TYPE_SYSTEM_ALERT:
                // like the ANR / app crashed dialogs
                // Type is deprecated for non-system apps. For system apps, this type should be
                // in a higher layer than TYPE_APPLICATION_OVERLAY.
                return  canAddInternalSystemWindow ? 13 : 10;
            case TYPE_APPLICATION_OVERLAY:
                return  12;
            case TYPE_INPUT_METHOD:
                // on-screen keyboards and other such input method user interfaces go here.
                return  15;
            case TYPE_INPUT_METHOD_DIALOG:
                // on-screen keyboards and other such input method user interfaces go here.
                return  16;
            case TYPE_STATUS_BAR:
                return  17;
            case TYPE_STATUS_BAR_ADDITIONAL:
                return  18;
            case TYPE_NOTIFICATION_SHADE:
                return  19;
            case TYPE_STATUS_BAR_SUB_PANEL:
                return  20;
            case TYPE_KEYGUARD_DIALOG:
                return  21;
            case TYPE_VOLUME_OVERLAY:
                // the on-screen volume indicator and controller shown when the user
                // changes the device volume
                return  22;
            case TYPE_SYSTEM_OVERLAY:
                // the on-screen volume indicator and controller shown when the user
                // changes the device volume
                return  canAddInternalSystemWindow ? 23 : 11;
            case TYPE_NAVIGATION_BAR:
                // the navigation bar, if available, shows atop most things
                return  24;
            case TYPE_NAVIGATION_BAR_PANEL:
                // some panels (e.g. search) need to show on top of the navigation bar
                return  25;
            case TYPE_SCREENSHOT:
                // screenshot selection layer shouldn't go above system error, but it should cover
                // navigation bars at the very least.
                return  26;
            case TYPE_SYSTEM_ERROR:
                // system-level error dialogs
                return  canAddInternalSystemWindow ? 27 : 10;
            case TYPE_MAGNIFICATION_OVERLAY:
                // used to highlight the magnified portion of a display
                return  28;
            case TYPE_DISPLAY_OVERLAY:
                // used to simulate secondary display devices
                return  29;
            case TYPE_DRAG:
                // the drag layer: input for drag-and-drop is associated with this window,
                // which sits above all other focusable windows
                return  30;
            case TYPE_ACCESSIBILITY_OVERLAY:
                // overlay put by accessibility services to intercept user interaction
                return  31;
            case TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY:
                return 32;
            case TYPE_SECURE_SYSTEM_OVERLAY:
                return  33;
            case TYPE_BOOT_PROGRESS:
                return  34;
            case TYPE_POINTER:
                // the (mouse) pointer layer
                return  35;
            default:
                Slog.e("WindowManager", "Unknown window type: " + type);
                return 3;
        }
    }
```

这个方法返回给定窗口类型对应的层级值。允许你控制不同类型的窗口在屏幕上的排序方式。

看方法内容很简单，对每一种类型的窗口，都规定了一个层级值。层级值反映了这些窗口在 Z 轴上的排序方式，层级值越高在 Z 轴上的位置也就越高。但是层级值的大小和窗口类型对应的那个值的大小并没有关系，窗口类型值大的窗口，对应的层级值不一定大。

另外，大部分层级值都唯一对应一种窗口类型，比较特殊的是：

*   层级值 2，所有 App 窗口会被归类到这一个层级。
*   层级值 3，该方法中没有提到的窗口类型会被归类到这一个层级。

这里看到状态栏窗口对应的窗口层级值是 17，而普通的 App 窗口的窗口层级值是 2，自然状态栏窗口就会比 App 窗口层级值高。

另外这里也能看到，窗口类型的数值和窗口层级值不是一个正相关的关系，即窗口类型值大的窗口，它在 Z 轴上的层级并不一定就高，窗口在 Z 轴的排序具体要看该窗口的类型值转换后得到的窗口层级值。

#### 2.6.2 DisplayArea 层级结构的生成

接下来继续分析窗口层级值是如何作用的，但是仍有一些内容需要提前了解，即 DisplayArea 按照具体作用，又可以分为以下几种类型：

```
/**
     * The value in display area indicating that no value has been set.
     */
    public static final int FEATURE_UNDEFINED = -1;

    /**
     * The Root display area on a display
     */
    public static final int FEATURE_SYSTEM_FIRST = 0;

    /**
     * The Root display area on a display
     */
    public static final int FEATURE_ROOT = FEATURE_SYSTEM_FIRST;

    /**
     * Display area hosting the default task container.
     */
    public static final int FEATURE_DEFAULT_TASK_CONTAINER = FEATURE_SYSTEM_FIRST + 1;

    /**
     * Display area hosting non-activity window tokens.
     */
    public static final int FEATURE_WINDOW_TOKENS = FEATURE_SYSTEM_FIRST + 2;

    /**
     * Display area for one handed feature
     */
    public static final int FEATURE_ONE_HANDED = FEATURE_SYSTEM_FIRST + 3;

    /**
     * Display area that can be magnified in
     * {@link Settings.Secure.ACCESSIBILITY_MAGNIFICATION_MODE_WINDOW}. It contains all windows
     * below {@link WindowManager.LayoutParams#TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY}.
     */
    public static final int FEATURE_WINDOWED_MAGNIFICATION = FEATURE_SYSTEM_FIRST + 4;

    /**
     * Display area that can be magnified in
     * {@link Settings.Secure.ACCESSIBILITY_MAGNIFICATION_MODE_FULLSCREEN}. This is different from
     * {@link #FEATURE_WINDOWED_MAGNIFICATION} that the whole display will be magnified.
     * @hide
     */
    public static final int FEATURE_FULLSCREEN_MAGNIFICATION = FEATURE_SYSTEM_FIRST + 5;

    /**
     * Display area for hiding display cutout feature
     * @hide
     */
    public static final int FEATURE_HIDE_DISPLAY_CUTOUT = FEATURE_SYSTEM_FIRST + 6;

    /**
     * Display area that the IME container can be placed in. Should be enabled on every root
     * hierarchy if IME container may be reparented to that hierarchy when the IME target changed.
     * @hide
     */
    public static final int FEATURE_IME_PLACEHOLDER = FEATURE_SYSTEM_FIRST + 7;

    /**
     * Display area for one handed background layer, which preventing when user
     * turning the Dark theme on, they can not clearly identify the screen has entered
     * one handed mode.
     * @hide
     */
    public static final int FEATURE_ONE_HANDED_BACKGROUND_PANEL = FEATURE_SYSTEM_FIRST + 8;

    /**
     * The last boundary of display area for system features
     */
    public static final int FEATURE_SYSTEM_LAST = 10_000;
```

看一下这些 Feature 的含义都是什么：

*   FEATURE_ROOT，一个屏幕上的根 DisplayArea，对应 DisplayContent。
*   FEATURE_DEFAULT_TASK_CONTAINER，容纳默认 Task 容器的 DisplayArea，对应 TaskDisplayArea。
*   FEATURE_WINDOW_TOKENS，容纳非 activity 窗口的 DisplayArea，对应 DisplayArea.Tokens。
*   FEATURE_ONE_HANDED，用于单手模式的 DisplayArea，对应名为 “OneHanded” 的 DisplayArea。
*   FEATURE_WINDOWED_MAGNIFICATION，在 ACCESSIBILITY_MAGNIFICATION_MODE_WINDOW 模式下可以对窗口的某些区域进行放大的 DisplayArea，对应名为 “WindowedMagnification” 的 DisplayArea。
*   FEATURE_FULLSCREEN_MAGNIFICATION，在 ACCESSIBILITY_MAGNIFICATION_MODE_FULLSCREEN 模式下可以对整个屏幕进行放大的 DisplayArea，对应名为 “FullscreenMagnification” 的 DisplayArea。
*   FEATURE_HIDE_DISPLAY_CUTOUT，隐藏 DisplayCutout 的 DisplayArea，对应名为 “HideDisplayCutout” 的 DisplayArea。
*   FEATURE_IME_PLACEHOLDER，存放输入法窗口的 DisplayArea，对应名为 “ImePlaceholder” 的 DisplayArea。
*   FEATURE_ONE_HANDED_BACKGROUND_PANEL，容纳单手背景图层的 DisplayArea，避免用户开启暗黑模式后，无法分辨屏幕是否已经进入了单手模式，对应名为 “OneHandedBackgroundPanel” 的 DisplayArea。

其中 DisplayContent 作为 DisplayArea 根节点，TaskDisplayArea 和 DisplayArea.Tokens（它还有一个 ImeContainer 子类）作为叶节点，其它类型的 DisplayArea 作为中间节点。

举个例子来说明一下 DisplayArea 层级结构的生成规则。

1）、假如我们现在有一个能够容纳窗口层级值等于 17 的 DisplayArea.Tokens，那么可以将它命名为 “Leaf:17:17”，代表它是一个 DisplayArea 叶节点，并且能够容纳的窗口层级值的最大值和最小值都是 17，即它只能容纳窗口层级值为 17 的窗口：

![](https://img-blog.csdnimg.cn/861adcfd3bcb43dd9fc0cbbf5c472ff4.png#pic_center)

2）、由于它是作为叶节点存在的，本身只作为一个容器存在，如果我们想要让它具备一些其它的功能，如能够容纳输入法，那么就需要为它增加一个 FEATURE_IME_PLACEHOLDER 类型的父容器，子容器会继承父容器的特性：

![](https://img-blog.csdnimg.cn/5a7bf94dba8e41598776eb48c8dc7c85.png#pic_center)

3）、同理，如果我们想要让它拥有定义的所有功能，那么就需要为它添加每一个 feature 对应的节点。最后也别忘了根节点，这样才能形成一个完成的 DisplayArea 层级结构，即从根节点到叶节点：

![](https://img-blog.csdnimg.cn/a2a44745a1504a789bcca69f65632faf.png#pic_center)

4）、再来考虑这样一种情况，假如有两个相邻的 Leaf，“Leaf:16:16” 和 “Leaf:17:17”，它们本来分别是有一个 ImePlaceholder 类型的父容器：

![](https://img-blog.csdnimg.cn/31e9b4a2c6074e298274a42f1e93b3c8.png#pic_center)

但是看到它们都的父容器都是 ImePlaceholder 类型的，代表它们都希望拥有同一种功能，那么这里可以把它们进行合并，就像这样：

![](https://img-blog.csdnimg.cn/80a08c78f2ff4fcf9e16bdc36254ac1e.png#pic_center)

毕竟既然可以重复利用，那就没有必要创建额外的节点。

注意这里必须要是相邻的节点才行，如果是这样：

![](https://img-blog.csdnimg.cn/c86ce7ba2fd2435f82d769adef8d509f.png#pic_center)

这里的 “Leaf:15:15” 和“Leaf:17:17”的父节点是无法合并的，就只能为它们分别创建一个 FullscreenMagnification 类型的父容器。

5）、了解了以上内容，现在我们可以来分析一下 DisplayArea 的生成规则。

由于总共有 37 个窗口层级值，那么最初可能就会有 37 个独立并且完整的 DisplayArea 层级结构：

![](https://img-blog.csdnimg.cn/ac6494d67b69409bb12efe5e83bc9081.png#pic_center)

这里的每一列都代表了一个独立并且完整的 DisplayArea 层级结构，每一个有颜色的格子都代表一个 DisplayArea 节点，从根节点 DisplayContent 一直到叶节点 Leaf。

Leaf 所在的层级结构代表了它所容纳的窗口所具有的一些功能。由于窗口都是有特定的用途，所以肯定不是所有的窗口都能作为输入法窗口，那么对于大部分层级结构，需要将删去 ImePlaceholder 类型的节点。同样，有一些窗口也不希望具备放大功能，那么它们的层级结构就可能需要删去 WindowMagnification 或者 FullscreenMagnification 类型的节点。

根据 Android 对各种窗口层级值可以具备的功能定义：

<table><thead><tr><th>Feature.mName</th><th>Feature.mID</th><th>Feature.mWindowLayers</th></tr></thead><tbody><tr><td>WindowedMagnification</td><td>FEATURE_WINDOWED_MAGNIFICATION</td><td>0~31</td></tr><tr><td>HideDisplayCutout</td><td>FEATURE_HIDE_DISPLAY_CUTOUT</td><td>0<sub>16,18,20</sub>23,26~35</td></tr><tr><td>OneHandedBackgroundPanel</td><td>FEATURE_ONE_HANDED_BACKGROUND_PANEL</td><td>0~1</td></tr><tr><td>OneHanded</td><td>FEATURE_ONE_HANDED</td><td>0<sub>23,26</sub>35</td></tr><tr><td>FullscreenMagnification</td><td>FEATURE_FULLSCREEN_MAGNIFICATION</td><td>0<sub>14,17</sub>23,26<sub>27,29</sub>31,33~35</td></tr><tr><td>ImePlaceholder</td><td>FEATURE_IME_PLACEHOLDER</td><td>15~16</td></tr></tbody></table>

我们需要删除一些节点，删除后的情况是（这里把 ImeContainer 和 TaskDisplayArea 也标记了出来）：

![](https://img-blog.csdnimg.cn/5abea9e9482d4d1c8f5733f30661e0a8.png#pic_center)

空出的格子代表了该层级结构不需要的节点，由于该节点被删除，所以需要将该节点的父节点与该节点的子节点进行连接，即将有颜色的格子上移来填补表里的空白格子，连接后的情况是：

![](https://img-blog.csdnimg.cn/1cc24c27c67c4eefb2aebd690df04198.png#pic_center)

最后，如我们上面所说的，合并那些可以合并的相邻节点的父节点，最终得到：

![](https://img-blog.csdnimg.cn/ef5ff352df5c4f8c8ef590431749aa37.png#pic_center)

仍然是一个格子代表一个节点，转为树状图是：

![](https://img-blog.csdnimg.cn/6b32da2ff10843408b044afc57470638.png#pic_center)

最后再在每一个叶节点上，添加它们对应的窗口，得到最终的 WindowContainer 层级结构图：

![](https://img-blog.csdnimg.cn/d62a9a6412ea4683bbeddaf491921f51.png#pic_center)

最后再提一下，我们将窗口分为 App 之上的窗口、App 窗口以及 App 之下的窗口，基准便是 TaskDisplayArea。

### 2.7 WindowContainer 层级结构根节点 —— RootWindowContainer

回看上面的 WindowContainer 层级结构图，能看到根节点为 RootWindowContainer，而非 DisplayArea 层级结构的根节点 DisplayContent，毕竟 DisplayContent 只代表一块屏幕，而 Android 是支持多屏幕的，展开来说就是包括一个内部屏幕（内置于手机或平板电脑中的屏幕）、一个外部屏幕（如通过 HDMI 连接的电视）以及一个或多个虚拟屏幕。

那么就需要一个 DisplayContent 的容器出现，这个角色自然也就是 RootWindowContainer 了，从名字也能看出来，它是作为 WindowContainer 层级结构的根节点所存在的：

```
/** Root {@link WindowContainer} for the device. */
class RootWindowContainer extends WindowContainer<DisplayContent>
        implements DisplayManager.DisplayListener
```

那么如果我们在开发者选项里通过 “Simulate secondary displays” 开启另一个虚拟屏幕，此时的情况是：

```
ROOT type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
  #1 Display 0  type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][720,1612] bounds=[0,0][720,1612]
  #0 Display 2  type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][720,480] bounds=[0,0][720,480]
```

转为层级结构图：

![](https://img-blog.csdnimg.cn/f223781586dd4bfe8dc34e0ca314558e.png#pic_center)

“Root” 即 RootWindowContainer，“Display 0” 代表默认屏幕，“Display 2” 是我们开启的虚拟屏幕。

### 2.8 小结

在代码搜索 WindowContainer 的继承关系，可以得到：

![](https://img-blog.csdnimg.cn/8dd74a25ef364f90a044682bb54442a0.png#pic_center)

除了不参与 WindowContainer 层级结构的 WindowingLayerContainer 之外，其他类都在本文有提及，用类图总结为：

![](https://img-blog.csdnimg.cn/3f1f945feba14eac8562fa52be8c4a4f.png#pic_center)

除了 WindowState 可以显示图像外，大部分的 WindowContainer，如 WindowToken、TaskDisplayArea 是不会有内容显示的，都只是一个抽象的容器概念，说白了就是一个收纳用的容器。极端点说，WMS 如果只为了管理窗口，WMS 也可以不创建这些个 WindowContainer 类，直接用一个 WindowState 的列表管理屏幕上的所有窗口。但是为了更有逻辑地管理屏幕上显示的窗口，比如哪些窗口属于同一个 Activity？哪些 Activity 属于同一个 App？如何将 App 窗口和非 App 窗口清楚得分隔开？因此还是需要创建各种各样的窗口容器类，即 WindowContainer 及其子类，来对 WindowState 进行分类，并且每一种 WindowContainer 通常都由另外一种专门的 WindowContainer 来管理，从而对窗口进行层次化的管理。这种层次化管理保证了每一个 WiindowContainer 都会在其对应的父容器中进行调整而不会越界。这个一点很重要，比如我点击 HOME 键回到 Launcher，此时 TaskDisplayArea 就会把 Launcher 对应的 Task，移动到它所对应的堆栈顶部，而这个调整仅限于 TaskDisplayArea 内部（因为 TaskDisplayArea 是 Task 的容器），这就保证了再怎么调整 Launcher，Launcher 的窗口也永远不可能高于 StatusBar 窗口，也不会低于 Wallpaper 窗口。

另外这个层级结构建立起来后，对窗口的遍历也是基于这个层级结构。比如发生转屏后，DisplayContent 会基于新的屏幕方向重新计算 Configuration，然后按照这个层级结构的顺序，依次把这个新计算的 Configuration 发给每一个 WindowContainer，最终每一个 WindowContainer 都能基于新的 Configuration 更新自己的 Configuration，而非每一个 WindowContainer 都需要重新去计算一次 Configuration，并且大部分的更新其实都是直接拿从父容器传过来的 Configuration 赋值给自己的 Configuration（相当于完全继承了父容器的 Configuration，当然会有一些 WindowContainer 会显式的声明自己有独特的策略，不会完全继承父容器的 Configuration，但这只是少数）。这体现容器的一个特性，即继承特性，比如你把一个盒子翻转，那么盒子里的小盒子也会跟着翻转，而不是每一个小盒子再去考虑要不要翻转。再举一个例子就是，分屏模式下，我们只需要计算好分屏 Task 的 bounds，Task 里的 ActivityRecord 以及 ActivityRecord 里的 WindowState，会自动继承 Task 的 Configuration，变成和 Task 同样的大小（大盒子缩小后，里面的小盒子会跟随大盒子缩小）。

3 Window 的底层实现 —— Layer 层级结构
----------------------------

### 3.1 Layer 基础概念

之前我们可能会认为窗口就是屏幕上一块显示图像的区域，这个当然不错，但是反之不一样成立，即屏幕上一块显示图像的区域不一定就是一个窗口，但是它一定是一个 Layer。至于 Layer，则一个比窗口更底层的实现，代表屏幕上一块显示内容的区域：

![](https://img-blog.csdnimg.cn/a3b4655c19804fd5b843b17eacf02710.png#pic_center)

SurfaceFlinger 如果要将所有 Layer 进行合成，那么需要知道每一个 Layer 的两个基本信息：

*   显示边框 / 边界，即该 Layer 在屏幕上的显示位置（x 和 y）和大小（w 和 h）。
    
*   显示内容，即该 Layer 所绘制图像的图形数据，保存在 Layer 拥有的图形缓冲区中。
    

这两个信息需要 WMS 和 App 这两方分别提供：

*   WMS 作为窗口管理服务，每一个窗口要显示多大，显示在哪儿，自然是由它来决定。
    
*   App 决定它定义的每一个界面的具体绘制内容，这个也不用多说。
    

SurfaceFlinger 拿到从 WMS 和 App 传过来的数据，将所有 Layer 合成送显。

那么还有一个问题，处于不同的进程的 App、WMS 以及 SurfaceFlinger，通过什么方式来进行信息传递？

![](https://img-blog.csdnimg.cn/a08d3748dca84d4e9bf38219ce6a6bc2.png#pic_center)

1）、当 App 进入前台时，WMS 会向 SurfaceFlinger 请求为 App 创建 Layer。

2）、SurfaceFlinger 创建 Layer 后，返回给 WMS 一个该 Layer 对应的 SurfaceControl 对象。WMS 持有这些 SurfaceControl，并且后续可以通过 SurfaceControl 向 SurfaceFlinger 提供窗口元数据，用来控制 App 在屏幕上的外观。

3）、WMS 将 SurfaceControl 发送给 App，App 会根据 SurfaceControl 创建 BufferQueue 和 Surface。Surface 持有一组图形缓冲区，并且链接了 BufferQueue 中的图像生产者 BufferQueueProducer，后续由它们来向 SurfaceFlinger 提供绘制了内容的图形缓冲区。

总的来说，SurfaceFlinger 消费 Surface 生产的图像缓冲区，并使用 WMS 通过 SurfaceControl 提供的窗口元数据，来对 Layer 进行合成，因此 Layer 是 Surface（包含 BufferQueue）和 SurfaceControl（包含显示边框等窗口元数据）的组合。

### 3.2 Layer 层级结构

再来看一段 SurfaceFlinger 的信息：

```
+ EffectLayer (Task=16#0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=0)
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=       11, pos=(0,0), size=(  -1,  -1), crop=[  0,   0,   0,   0], cornerRadius=0.000000, isProtected=0, isTrustedOverlay=0, isOpaque=0, invalidate=0, dataspace=De
fault, defaultPixelFormat=Unknown/None, backgroundBlurRadius=0, color=(-1.000,-1.000,-1.000,1.000), flags=0x00000000, tr=[1.00, 0.00][0.00, 1.00], mBounds=[0.00, 0.00, 1200.00, 1920.00], l
ayerId=0
      parent=DefaultTaskDisplayArea#0
      zOrderRelativeOf=none
      activeBuffer=[   0x   0:   0,Unknown/None], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0, mRefreshPending=0, metadata={taskId:16}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRa
dius=0.000,

+ ContainerLayer (1c7dd1d ActivityRecordInputSink com.android.settings/.Settings#0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=0)
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=-2147483648, pos=(0,0), size=(  -1,  -1), crop=[  0,   0,  -1,  -1], cornerRadius=0.000000, isProtected=0, isTrustedOverlay=0, isOpaque=0, invalidate=1, dataspace=
Default, defaultPixelFormat=Unknown/None, backgroundBlurRadius=0, color=(0.000,0.000,0.000,1.000), flags=0x00000000, tr=[1.00, 0.00][0.00, 1.00], mBounds=[0.00, 0.00, 1200.00, 1920.00], la
yerId=0
      parent=ActivityRecord{8e62b92 u0 com.android.settings/.Settings t16}#0
      zOrderRelativeOf=none
      activeBuffer=[   0x   0:   0,Unknown/None], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0, mRefreshPending=0, metadata={}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.00
0,

+ ContainerLayer (ActivityRecord{8e62b92 u0 com.android.settings/.Settings t16}#0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=0)
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=        0, pos=(0,0), size=(  -1,  -1), crop=[  0,   0,  -1,  -1], cornerRadius=0.000000, isProtected=0, isTrustedOverlay=0, isOpaque=0, invalidate=1, dataspace=De
fault, defaultPixelFormat=Unknown/None, backgroundBlurRadius=0, color=(0.000,0.000,0.000,1.000), flags=0x00000000, tr=[1.00, 0.00][0.00, 1.00], mBounds=[0.00, 0.00, 1200.00, 1920.00], laye
rId=0
      parent=Task=16#0
      zOrderRelativeOf=none
      activeBuffer=[   0x   0:   0,Unknown/None], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0, mRefreshPending=0, metadata={}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.00
0,

+ ContainerLayer (5cadaf2 com.android.settings/com.android.settings.Settings#0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=0)
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=        0, pos=(0,0), size=(  -1,  -1), crop=[  0,   0,  -1,  -1], cornerRadius=0.000000, isProtected=0, isTrustedOverlay=0, isOpaque=0, invalidate=1, dataspace=De
fault, defaultPixelFormat=Unknown/None, backgroundBlurRadius=0, color=(0.000,0.000,0.000,1.000), flags=0x00000000, tr=[1.00, 0.00][0.00, 1.00], mBounds=[0.00, 0.00, 1200.00, 1920.00], laye
rId=0
      parent=ActivityRecord{8e62b92 u0 com.android.settings/.Settings t16}#0
      zOrderRelativeOf=none
      activeBuffer=[   0x   0:   0,Unknown/None], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0, mRefreshPending=0, metadata={}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.00
0,

+ BufferStateLayer (com.android.settings/com.android.settings.Settings#0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=1)
    [  0,   0, 1200, 1920]
  Region SurfaceDamageRegion (this=0 count=1)
    [  0, 286, 1200, 1824]
      layerStack=   0, z=        0, pos=(0,0), size=(1200,1920), crop=[  0,   0,  -1,  -1], cornerRadius=0.000000, isProtected=0, isTrustedOverlay=0, isOpaque=1, invalidate=0, dataspace=De
fault, defaultPixelFormat=RGBA_8888, backgroundBlurRadius=0, color=(0.000,0.000,0.000,1.000), flags=0x00000102, tr=[1.00, 0.00][0.00, 1.00], mBounds=[0.00, 0.00, 1200.00, 1920.00], layerId
=0
      parent=5cadaf2 com.android.settings/com.android.settings.Settings#0
      zOrderRelativeOf=none
      activeBuffer=[1200x1920:1200,RGBA_8888], tr=[1.00, 0.00][0.00, 1.00] queued-frames=0, mRefreshPending=0, metadata={dequeueTime:505108725337, windowType:1, ownerPID:6640, ownerUID:100
0}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.000,
      *BufferQueueDump mIsBackupBufInited=0, mAcquiredBufs(size=1)
       [00] handle=0xb400007e3a327d80, fence=0xb400007e3a281c80, time=0x759ae697c5, xform=0x00
       FPS ring buffer:
       (0) 13:19:15.488 fps=1.94  dur=1544.49       max=1494.64       min=16.54
```

这是当从 Launcher 启动 Settings 应用后，Settings 中所有 Layer 的信息，转为 SurfaceFlinger 层级结构图：

![](https://img-blog.csdnimg.cn/277dfcc667ee4a7d8b2d9af893d48081.png#pic_center)

对比 Java 层 WindowContainer 层级结构图：

![](https://img-blog.csdnimg.cn/4f4f80b023554c589478a618bd2ce28b.png#pic_center)

能看到：

1）、WMS 侧中每一级的 WindowContainer，在 SurfaceFlinger 侧中都有一个对应的 Layer，这是因为每创建一个 WindowContainer，都会为其创建一个 SurfaceControl 对象。那么换句话说，在 SurfaceFlinger 侧的每一个 Layer，大部分情况下在 WMS 侧都能找到一个对应的的 surfaceControl，这也就是为什么说 WMS 能够通过 SurfaceControl 来控制 SurfaceFlinger 中 Layer 的显示。

2）、这里有一个名为”ContainerLayer (1c7dd1d ActivityRecordInputSink com.android.settings/.Settings#0)“的 Layer，这个类是用来拦截输入事件向 Activity 发送的，可以看下它创建的地方：

```
// frameworks\base\services\core\java\com\android\server\wm\ActivityRecordInputSink.java

	private SurfaceControl createSurface(SurfaceControl.Transaction t) {
        SurfaceControl surfaceControl = mActivityRecord.makeChildSurface(null)
                .setName(mName)
                .setHidden(false)
                .setCallsite("ActivityRecordInputSink.createSurface")
                .build();
        // Put layer below all siblings (and the parent surface too)
        t.setLayer(surfaceControl, Integer.MIN_VALUE);
        return surfaceControl;
    }
```

*   通过 ActivityRecord.makeChildSurface，将该 SurfaceControl 对应的 Layer 的父 Layer 设置为了 ActivityRecord。
*   通过 Transaction.setLayer，将该 SurfaceControl 对应的 Layer 的层级设置为最低 —— Integer.MIN_VALUE（-2147483648）。

那么结果就是该 Layer 的信息所显示的那样；

```
z=-2147483648
      parent=ActivityRecord{8e62b92 u0 com.android.settings/.Settings t16}#0
```

所以 ActivityRecord 这一级 WindowContainer 对应的 Layer，除了有一个 WindowState 对应的子 Layer 以外，还有一个 ActivityRecordInputSink 对应的子 Layer。

另外由于该 Layer 的层级值是负数，所以我这边在画层级结构图的时候，将它放在了父 Layer 上面，即父 Layer 上面的子 Layer，层级值为负数，父 Layer 下面的子 Layer，层级值为正数（SurfaceFlinger 的信息，越往下的 Layer 层级值越高，而之前贴的 WindowContainer 的信息则是越靠上层级值越高）。

3）、SurfaceFlinger 中，WindowState 这一级下面，还有一个 Layer：”BufferStateLayer (com.android.settings/com.android.settings.Settings#0)“。那么这里就需要介绍一下 Layer 类的继承关系：

![](https://img-blog.csdnimg.cn/674293f6654145009f4c5e1755de54a6.png#pic_center)

*   Layer，所有 Layer 的父类。
*   EffectLayer，代表了一个有纯色或者阴影效果的 Layer，比如实现 Dim 效果的时候，添加的就是这个 Layer，另外一提，Task 对应的 WindowContainer 也是这种 Layer，说明 Task 也是可以显示一些内容的（ShadowRadius）。
*   ContainerLayer，代表了一个容器类 Layer，这种 Layer 没有缓冲区，只是用来作为其他 Layer 的容器，比如 ActivityRecord 和 WindowState 对应的 Layer，都是这种类型。
*   BufferLayer，是携带图形缓冲区的 Layer，即真正显示 App 绘制内容的 Layer。其中又分 BufferQueueLayer 和 BufferStateLayer，由于 BufferQueueLayer 使用旧版的 API，支持的功能有限，如今已经弃用，现在用的是 BufferStateLayer。BufferStateLayer 对应 Java 层 WindowSurfaceController 对象持有的 SurfaceControl 对象，会在 relayoutWindow 流程中创建，当窗口绘制完成后进行显示。

将 Layer 层级结构和 WindowContainer 层级结构进行比较后可以知道，Layer 层级结构的建立是以 WindowContainer 层级结构为基础，向下又延申了一层 BufferStateLayer，如果你已经理解了 WindowContainer 层级结构，那么理解 Layer 层级结构起来就非常容易。

另外注意这只是一般规则，实际上系统运行过程中可能会创建许多 Layer 穿插其中，如动画的 Leash、实现阴影效果的 DimLayer 以及旋转用到的 RotationLayer 等等，对于这些特殊的 Layer 则需要具体场景具体分析。

接下来介绍几种能够层级结构变化的情况。

### 3.3 Leash

以一个从 Launcher 启动 Settings 的动画过程为例来说明一下 Leash 的用法，由于是 App 从后台移动到前台，因此动画的主体是 Task。

#### 3.3.1 动画前

动画前 Task#16 相关的层级结构是：

![](https://img-blog.csdnimg.cn/33f22121ad414cd58c0c0ffd247e6902.png#pic_center)

Task#16 的 Layer 层级结构，我们上面也说明过了，这里额外加入了它的父 Layer，DefaultTaskDisplayArea。

此时没有 BufferStateLayer，因为此时 Settings 还在后台，窗口并没有绘制。

#### 3.3.2 动画中

动画过程中 Task#16 相关的层级结构是：

![](https://img-blog.csdnimg.cn/d438c08d99b94898a60af2599a575320.png#pic_center)

这里多了两个 Layer：

*   BufferStateLayer，窗口绘制完成后才开始的动画，所以动画过程中 BufferStateLayer 已经在了。
*   一个名为”animation-leash of app_transition“的 Layer，且该 Layer 插在了 DefaultTaskDisplayArea 和 Task 中间。

Leash 的这种行为有点像我们之前分析 Freeform 的时候 DecorCaptionView 的做法：

1）、先创建一个 Leash，然后指定该 Leash 对应的 Layer 的父 Layer 为动画的主体（这个例子是 Task）的父 Layer（DefaultTaskDisplayArea）。

2）、接着将 Task 对应的 Layer 从原父 Layer（DefaultTaskDisplayArea）上剥离，变为 Leash 的子 Layer（将一个子 Layer 从原父 Layer 重定向到一个新的父 Layer 上，称为 reparent 操作）。

从而实现在 DefaultTaskDisplayArea 和 Task 中插进一个 Leash 的操作。

注意此时执行动画的对象是 Leash，而非 Task。但由于 Task 的父 Layer 是 Leash，所以 Leash 移动，Task 会跟着移动，我们能够看到的 BufferStateLayer 也会跟着移动。

#### 3.3.3 动画后

动画结束后 Task#16 相关的层级结构是：

![](https://img-blog.csdnimg.cn/fd1496f2e957475587371647f4ef4900.png#pic_center)

动画结束，Leash 已经完成了它的任务，可以撤了，但是在撤之前需要将绑定在 Leash 上的 Task 重新 reparent 回原父 Layer 上（DefaultTaskDisplayArea）。

#### 3.3.4 小结

上面的例子大致说明了 Leash 的运行机制，这其中有 3 个 Layer 参与，一个是动画主体（Task），一个是动画主体的父 Layer（DefaultTaskDisplayArea），还有一个是 Leash。

Leash 的作用即是代替动画的主体去执行动画：

1）、创建 Leash，将 Leash 绑定到动画主体的父 Layer 上，动画主体则 reparent 到 Leash 上。

2）、对 Leash 进行动画，而非动画主体。

3）、动画结束后，将动画主体 reparent 回原父 Layer 上，移除 Leash。

至于为什么要搞 Leash 这么一个东西出来，我个人认为是，如果直接对动画主体执行动画，那么就会涉及一个恢复的操作。比如说对 Activity 间的切换，如果不用 Leash 直接对 Activity 进行操作，那么情况是：旧的 Activity 切走的时候，可能会有一个向左平移的动画，后续如果我们想要这个旧的 Activity 重新出现，那么我们不仅需要将它显示出来，还需要将它从之前的动画结束位置向右平移，这样它才能出现在屏幕上。并且我们的动画越复杂，恢复需要的步骤也就越多。

如果我们把动画主体 reparent 到 Leash 上，然后对 Leash 进行相同的动画，不仅能达到同样的效果，而且不管我们对 Leash 如何进行动画，只要在动画结束后将动画主体 reparent 回原来的父 Layer 上，就可以恢复原状，而不用像上面一样考虑要做多少相反的操作才能恢复原状。

最后，Leash 理论上可以被应用到任何希望进行动画的 Surface 上，比如这里举的例子是应用到了 Task 上，因为这个动画发生于 App 间的切换过程中。如果是 Activity 间的切换，那么 Leash 就可以应用到 ActivityRecord，同样 WindowState 也是可以的，主要看动画的主体是整个 App？还是单个 Activity？还是某个窗口？

### 3.4 DimLayer 和相对层级

DimLayer 的本质是一个 EffectLayer，默认是一个纯黑 Layer，我们可以通过设置 Layer 的透明度来施加阴影效果，或者设置 Layer 模糊半径来施加模糊效果。

来看一个实际应用的例子，Files：

![](https://img-blog.csdnimg.cn/f4793e43672d4fc1a560c610ea180b5f.png#pic_center)

此时 Files 有两个窗口，子窗口盖在主窗口上，并且此时子窗口声明了一个特殊窗口标志位：

```
/** Window flag: everything behind this window will be dimmed.
         *  Use {@link #dimAmount} to control the amount of dim. */
        public static final int FLAG_DIM_BEHIND        = 0x00000002;
```

该窗口标志位的作用是，将声明了该标志位的窗口的后面所有内容都变暗，因此最终的效果就是如上图所示，即主窗口被蒙上了一层阴影效果。

此时的窗口层级结构图是：

![](https://img-blog.csdnimg.cn/7fc946e0952947bc9f98a4af324134f9.png#pic_center)

在这个层级结构图中，只有可见的 Layer 才被标记了颜色，此时可见 Layer 的 Z 轴顺序是：子窗口 > DImLayer > 主窗口。

主窗口和子窗口对应的两个 BufferStateLayer 不用多说，这里主要看下 DimLayer，相关信息如下：

```
+ EffectLayer (Dim Layer for - Task=36#0) uid=1000
      z=       -1
      color=(0.000,0.000,0.000,0.600)
      parent=Task=36#0
      zOrderRelativeOf=f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
```

*   alpha 值为 0.6，上面我们也提到过，EffectLayer 默认是一个纯黑的 Layer，我们需要设置它的透明度来达到变暗的效果（透明度为 1 则会完全遮住后面的内容）。
*   DimLayer 的父 Layer 是 Task#36，这个跟上层的实现有关系，目前是 Task 和 DisplayArea 支持创建这么一个 DimLayer。
*   这里设置了 DimLayer 的相对层级为子窗口 WindowState 这一级对应的 ContainerLayer，**ContainerLayer (f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0)**。设置一个 Layer 的相对层级，相当于是将该 Layer 从其原来的父 Layer 上剥离，这里是 **EffectLayer (Task=36#0)**，然后将这个相对层级的 Layer 作为父 Layer，这里是 **ContainerLayer (f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0)**，重新绑定到这个父 Layer 上面。所以这里能够看到虽然 DimLayer 的父 Layer 是 **EffectLayer (Task=36#0)**，但是在实际上的层级结构中，DimLayer 是作为了 **ContainerLayer (f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0)** 的子 Layer。
*   当然只设置相对层级还不够，还需要将 DimLayer 的层级值设置为 - 1，这样它才不会把子窗口也盖住，并且能够遮住层级比子窗口低的 Layer。

#### 3.4.1 DimLayer 层级值从 - 1 修改为 100

这里我们手动修改 DimLayer 的实现，将其层级值从 - 1 修改为 100，那么显示效果为：

![](https://img-blog.csdnimg.cn/58d77bc175ab45c6a272daefa73d1440.png#pic_center)

此时阴影效果不仅覆盖了主窗口，连子窗口也覆盖了，层级结构图为：

![](https://img-blog.csdnimg.cn/052e68b167c34624aa7f852a16ca9ecb.png#pic_center)

很显然，当设置 EffectLayer 的层级值为 100 后，它就变成了子窗口中层级值最高的 Layer，既然比子窗口对应的 BufferStateLayer 层级值还高，自然会把整个 Activity 都遮住，符合预期。

#### 3.4.2 DimLayer 相对层级从子窗口修改为主窗口

这里我们手动修改 DimLayer 的实现，将其相对层级的对象从子窗口修改为主窗口，那么显示效果为：

![](https://img-blog.csdnimg.cn/aa965b60f469438e8ac469572971085c.png#pic_center)

此时阴影效果消失，层级结构图为：

![](https://img-blog.csdnimg.cn/bc5e1cff1f1b449daaf3c57e9e7126dc.png#pic_center)

将 EffectLayer 的相对层级的对象从子窗口修改为主窗口，它就变成了子窗口中层级值最低的 Layer，自然无法再遮挡任何 Layer，并且被全屏的主窗口完全遮挡从而无法显示，符合预期。

#### 3.4.3 小结

相对层级的出现，使得 Layer 的层级顺序调整更加灵活，并且后续如果不需要相对层级了，也不需要执行繁琐的” 先将当前 Layer 从相对层级 Layer 上移除，再重新绑定回原父 Layer“的操作，直接设置一下层级值就可以将相对层级 Layer 移除了。

另外你也可以根据需要，为 DimLayer 设置不同的相对层级对象，从而实现将某个子窗口变暗，或者将某个 Activity 变暗，或者将整个 App 变暗。

4 总结
----

这里我按照自己的理解画了一下 Android 架构图，主要是对看下 Framework 这块：

![](https://img-blog.csdnimg.cn/a13b06c800b348fe88c9601ccc74591d.png#pic_center)

*   顶层是基于 ApplicationFramework 层的 Apps 层，ApplicationFramework 是运行在 App 进程的 Framework 代码，如四大组件，各种 Manager。在这一层，屏幕上的一块显示区域，典型代表是 Activity，但是 Activity 毕竟是一个综合性比较强的概念，具体到内容显示这块还是由 Window 类负责，Window 则是容纳 View 对象的容器。我们要了解屏幕的内容是如何组成的，首先需要知道的就是组成屏幕的最小单位 —— Window，的内部构造，即我们介绍的第一部分，View 层级结构。
    
*   接着是系统服务，其中系统服务既有基于 Java 的 WMS 之类（我习惯叫做 JavaFramework），也有基于 C++ 的 SurfaceFlinger 之类（我习惯叫做 NativeFramework）：
    
    *   JavaFramework，在这一层，屏幕上的一块显示区域，典型代表是 WindowState，当然更加通用的结构是 WindowContainer。WMS 作为管理窗口的服务，它需要把屏幕上的所有窗口按照一定的规则系统化的组织起来，即我们介绍的第二部分，WindowContainer 层级结构。这样 SurfaceFlinger 就只需要操心合成 Layer 的相关事务就行了，不需要去关心如何去建立如此庞大的一个层级结构。
    *   NativeFramework，在这一层，屏幕上的一块显示区域，典型代表是 BufferStateLayer，当前更加通用的结构是 Layer。SurfaceFlinger 对 Layer 进行合成需要基于 Layer 层级结构，而 Layer 层级结构的基本框架就是 WindowContainer 层级结构，另外 WMS 也可以通过 SurfaceControl 来在 Layer 层级结构里进行一些局部的调整。
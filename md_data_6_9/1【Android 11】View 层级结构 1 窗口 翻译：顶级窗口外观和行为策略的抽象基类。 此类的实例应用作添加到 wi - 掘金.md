> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7139889243597307941)

> 1 窗口 翻译：顶级窗口外观和行为策略的抽象基类。 此类的实例应用作添加到 window manager 的顶级视图。 它提供了标准的 UI 策略，例如背景、标题区域、默认按键事件处理等。这个抽象类的唯一

```
 * Abstract base class for a top-level window look and behavior policy.  An
 * instance of this class should be used as the top-level view added to the
 * window manager. It provides standard UI policies such as a background, title
 * area, default key processing, etc.
 *
 * <p>The only existing implementation of this abstract class is
 * android.view.PhoneWindow, which you should instantiate when needing a
 * Window.
 */
public abstract class Window



```

翻译：顶级窗口外观和行为策略的抽象基类。 此类的实例应用作添加到 window manager 的顶级视图。 它提供了标准的 UI 策略，例如背景、标题区域、默认按键事件处理等。这个抽象类的唯一现有实现是 android.view.PhoneWindow，当需要一个 Window 时你应该实例化它。

Window 类不仅是个抽象类，而且它的定义也挺抽象，但是从这个注释中，我们还是可以得到两点和我们今天所分析的内容相关的信息：

1）、Window 在 View 的层级结构中是作为顶级视图存在的：

2）、Window 会被添加到 WindowManager，由 WindowManager 管理。

那么我们了解窗口，也是从这两个角度去出发：

1）、微观角度，探究窗口内部的构造。

2）、宏观角度，探究 WMS 如何管理屏幕上显示的诸多窗口。

至于窗口，我们先理解它为一块在屏幕上可以显示图像的区域。

从 Activity 的层面讲，由于 Activity 是通过各种 UI 元素与用户进行交互的，那么这些 UI 元素就需要一个统一的载体来承载它们，这个 UI 元素的容器也就是 Window。

对于每一个 Activity，或者每一个窗口来说，它的界面布局都是由多层 View 嵌套合成得到的，如果把 View 的层级结构比作一个树状图，那么 Window 应该是根节点 root view 的存在，我们自定义的一些布局 xml 文件中的一些 layout 就是中间节点，子节点就是这些 layout 中定义的如 Button、TextView 等很具体的 View。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73ff50ec8e154b9ca227591070639907~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

如果我们想要再深入一点去探究 Window 的 View 层级结构的话，会发现上面的说法其实是不够严谨的。View 的层级结构是由各种 View 层层嵌套得来的，这并不是一个抽象的说法，而是实实在在的由一个 ViewGroup 嵌套多个子 ViewGroup，多个子 ViewGroup 再分别嵌套多个子子 ViewGroup，这样一层层嵌套得来的。

ViewGroup 是一种特殊的 View，它可以包含其他的 View，我们通常所用的 FrameLayout、RelativeLayout 和 LinearLayout 等，都继承自 ViewGroup，因此 ViewGroup 可以视作为 View 的容器类，它有一个 View 数组类型的成员变量 mChildren 来存放它的子 View：

```
    
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P)
    private View[] mChildren;


```

并且 ViewGroup 本身也是继承 View 的：

```
public abstract class ViewGroup extends View implements ViewParent, ViewManager


```

那么 ViewGroup 就可以作为一个 View 添加到某一个 ViewGroup 的 mChildren 数组中，这就让一个父 ViewGroup 包含多个子 ViewGroup 成为可能，就像这样：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f745779134b4746ab07ff4491fb1344~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

我个人习惯用树状图表示：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c8935f7ea064f879db7c2dce880cc90~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

既然 ViewGroup 的为 View 的多级嵌套提供了支持，那么在 View 的层级结构中，最顶层 View 应该也是一个 ViewGroup。而 Window 的唯一实现类，PhoneWindow，只继承了 Window，因此 PhoneWindow 是无法作为当前 Window 的 View 的层级结构的根节点的，因为根节点应该是一个 ViewGroup 类型的类。

那么根节点对应的是哪个类？这里我们自问自答，View 的层级结构的根节点是 DecorView，一个真正的 View：

```
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks


```

一般在网上百度 Activity、Window 和 DecorView 之间的关系，都会得到类似下面的图：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1bcb803f65ad471f86d445fdd0523261~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

之前我以为 Activity 的区域对应显示设备的整个区域，Activity 的区域包含 PhoneWindow 区域，PhoneWindow 的区域又包含 DecorView 的区域。。。但是实际上，Activity 和 PhoneWindow 都是一个抽象的概念，都没有一个具体的显示区域，而 DecorView，作为一个 View，它的作用正如它作为 PhoneWindow 的成员变量注释中所描述的：

```
    
    private DecorView mDecor;


```

DecorView 是一个窗口的最顶层 View，因此真正承载 UI 元素是 DecorView，毕竟 View 才是我们实打实可以看的到的 UI 元素。

总结一下，上图可以分为抽象和具象两部分：

1)、一方面，Activity 启动的时候会创建一个 PhoneWindow 对象，将 UI 处理交给 PhoneWindow；而 PhoneWindow 在后续 Activity 加载布局资源的时候生成一个 DecorView，所以在抽象概念上，是 Activity > PhoneWindow > DecorView 的，如下图所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2618f01125b5410395126b96b84b7ec3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

2)、另一方面，由于 Activity 和 PhoneWindow 都是抽象的概念，我们直接看到的其实是 DecorView。并且 DecorView 在生成 View 层级结构的时候，是作为根 View 的，所以在具象概念上，或者说我们能够看到的层面上，DecorView 是最外层的，如下图所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/395160f0184841b182fd3e109ab8d0ff~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

那么我们想要了解一个 Window 的内部构造，就需要知道以 DecorView 为根 View 的 View 层级结构是怎么生成的。

我们经常在 Activity#onCreate 中调用 Activity#setContentView 去加载指定的布局文件，就像这样：

```
setContentView(R.layout.activity_main);


```

DecorView 就是在这个过程中创建的。

3.1 Activity#setContentView
---------------------------

```
    
     * Set the activity content from a layout resource.  The resource will be
     * inflated, adding all top-level views to the activity.
     *
     * @param layoutResID Resource ID to be inflated.
     *
     * @see #setContentView(android.view.View)
     * @see #setContentView(android.view.View, android.view.ViewGroup.LayoutParams)
     */
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }


```

这里 Activity#getWindow 返回的便是和当前 Activity 一一对应的 Window 对象。

3.2 PhoneWindow#installDecor
----------------------------

本来我们看到上一步，应该是调用到 PhoneWindow#setContentView 的，但是我们写的 Activity 是继承了 AppCompatActivity 的，AppCompatActivity 又重写了 Activity#setContentVIew 方法，所以流程就有了些许不一样，先走到的是 PhoneWindow#installDecor，PhoneWindow#setContentView 要在稍晚的时间点才会被调用：

调用堆栈是：

```
08-20 09:38:31.520  6254  6254 I ukynho_test: MainActivity#onCreate
08-20 09:38:31.543  6254  6254 I ukynho_test: call setContentView
08-20 09:38:31.548  6254  6254 I ukynho_decor: PhoneWindow#installDecor ---- title = null    mDecor = null   mContentParent = null
08-20 09:38:31.548  6254  6254 I ukynho_decor: java.lang.Throwable
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at com.android.internal.policy.PhoneWindow.installDecor(PhoneWindow.java:2716)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at com.android.internal.policy.PhoneWindow.getDecorView(PhoneWindow.java:2128)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at androidx.appcompat.app.AppCompatDelegateImpl.createSubDecor(AppCompatDelegateImpl.java:717)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at androidx.appcompat.app.AppCompatDelegateImpl.ensureSubDecor(AppCompatDelegateImpl.java:659)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at androidx.appcompat.app.AppCompatDelegateImpl.setContentView(AppCompatDelegateImpl.java:552)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at androidx.appcompat.app.AppCompatActivity.setContentView(AppCompatActivity.java:161)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at com.qq.reader.activity.MainActivity.onCreate(MainActivity.java:29)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at android.app.Activity.performCreate(Activity.java:8012)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at android.app.Activity.performCreate(Activity.java:7996)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1311)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:3525)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3713)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:85)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:135)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:95)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2156)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at android.os.Handler.dispatchMessage(Handler.java:106)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at android.os.Looper.loop(Looper.java:236)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at android.app.ActivityThread.main(ActivityThread.java:7814)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at java.lang.reflect.Method.invoke(Native Method)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:621)
08-20 09:38:31.548  6254  6254 I ukynho_decor:  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:997)


```

PhoneWindow#installDecor 方法内容如下：

```
    private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);

            
            mDecor.makeFrameworkOptionalFitsSystemWindows();

            final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                    R.id.decor_content_parent);

            
        }
    }


```

这个方法中和我们现在分析的内容相关的有两点：

1）、通过 PhoneWindow#generateDecor 生成 DecorView。

2）、通过 PhoneWindow#generateLayout 生成 Decor 的 View 层级结构。

### 3.2.1 DecorView 的生成

```
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }


```

在 PhoneWIndow#installDecor 中，由于我们是首次创建，所以 DecorView 类型的 mDecor 是空的，需要调用 PhoneWIndow#generateDecor 去生成一个 DecorView。

```
    protected DecorView generateDecor(int featureId) {
        
        
        
        Context context;
        if (mUseDecorContext) {
            Context applicationContext = getContext().getApplicationContext();
            if (applicationContext == null) {
                context = getContext();
            } else {
                context = new DecorContext(applicationContext, this);
                if (mTheme != -1) {
                    context.setTheme(mTheme);
                }
            }
        } else {
            context = getContext();
        }
        return new DecorView(context, featureId, this, getAttributes());
    }


```

mUseDecorContext 在 PhoneWindow 构造方法中被设置为 true，那么这里会创建一个 DecorView 专用的上下文 DecorContext，然后 DecorView 基于这个新建的 DecorContext 创建。

### 3.2.2 DecorView 层级结构的生成

这里我们拿一个测试 App 做例子：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8fd709c533e74a45899bdb691f2bd643~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

一个带了 ActionBar 和三个按钮的 Activity。

我这边查看 Activity 布局信息一般是通过两种方式，一是 adb shell dumpsys activity top，这里可以看下 dumpsys 出来的信息：

```
    View Hierarchy:
      DecorView@7b3553[MainActivity]
        android.widget.LinearLayout{fe72e90 V.E...... ........ 0,0-1200,1904}
          android.view.ViewStub{8fee389 G.E...... ......I. 0,0-0,0 
          android.widget.FrameLayout{2d0cf8e V.E...... ........ 0,48-1200,1904}
            androidx.appcompat.widget.ActionBarOverlayLayout{7f9d3af V.E...... ........ 0,0-1200,1856 
              androidx.appcompat.widget.ContentFrameLayout{c38a3bc V.E...... ........ 0,128-1200,1856 
                android.widget.RelativeLayout{5692f45 V.E...... ........ 0,0-1200,1728}
                  androidx.appcompat.widget.AppCompatButton{e8e569a VFED..C.. ........ 200,200-376,296 
                  androidx.appcompat.widget.AppCompatButton{619fbcb VFED..C.. ........ 0,296-243,392 
                  androidx.appcompat.widget.AppCompatButton{dca3fa8 VFED..C.. ........ 0,392-220,488 
              androidx.appcompat.widget.ActionBarContainer{2727ac1 V.ED..... ........ 0,0-1200,128 
                androidx.appcompat.widget.Toolbar{e6b4266 V.E...... ........ 0,0-1200,128 
                  androidx.appcompat.widget.AppCompatTextView{2ca89a7 V.ED..... ........ 48,37-316,91}
                  androidx.appcompat.widget.ActionMenuView{457ae54 V.E...... ........ 1184,0-1184,128}
                androidx.appcompat.widget.ActionBarContextView{96b01fd G.E...... ......I. 0,0-0,0 
        android.view.View{3d91ef2 V.ED..... ........ 0,1904-1200,2000 
        android.view.View{c0c1943 V.ED..... ........ 0,0-1200,48 


```

另一种就是借助 SDK 自带的工具 hierarchyviewer：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/96ddc41e23ad4c76b615c5c03f716e58~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这个工具可以看到 View 的一些具体信息，如寬高，坐标，透明度等，分析问题的时候很有用。

通过这两种方式，看到最顶层是 DecorView 这不用多说，但是自 DecorView 到 Activity 自定义的布局结构，中间还有很多层 View，这些便是在 PhoneWindow#generateLayout 中生成的，也就是 PhoneWindow#installDecor 的第二部分：

```
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
            ... ...
        }


```

在这一步之前我们只是生成了 DecorView 这一层，看下这一步是如何生成其他 View 的。

```
    protected ViewGroup generateLayout(DecorView decor) {
        

        TypedArray a = getWindowStyle();

        if (false) {
            System.out.println("From style:");
            String s = "Attrs:";
            for (int i = 0; i < R.styleable.Window.length; i++) {
                s = s + " " + Integer.toHexString(R.styleable.Window[i]) + "="
                        + a.getString(i);
            }
            System.out.println(s);
        }

        mIsFloating = a.getBoolean(R.styleable.Window_windowIsFloating, false);
        int flagsToUpdate = (FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR)
                & (~getForcedWindowFlags());
        if (mIsFloating) {
            setLayout(WRAP_CONTENT, WRAP_CONTENT);
            setFlags(0, flagsToUpdate);
        } else {
            setFlags(FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
            getAttributes().setFitInsetsSides(0);
            getAttributes().setFitInsetsTypes(0);
        }

        

        if (a.getBoolean(R.styleable.Window_windowTranslucentStatus,
                false)) {
            setFlags(FLAG_TRANSLUCENT_STATUS, FLAG_TRANSLUCENT_STATUS
                    & (~getForcedWindowFlags()));
        }

        if (a.getBoolean(R.styleable.Window_windowTranslucentNavigation,
                false)) {
            setFlags(FLAG_TRANSLUCENT_NAVIGATION, FLAG_TRANSLUCENT_NAVIGATION
                    & (~getForcedWindowFlags()));
        }


        
        
        if (a.getBoolean(R.styleable.Window_windowContentTransitions, false)) {
            requestFeature(FEATURE_CONTENT_TRANSITIONS);
        }
        if (a.getBoolean(R.styleable.Window_windowActivityTransitions, false)) {
            requestFeature(FEATURE_ACTIVITY_TRANSITIONS);
        }

        mIsTranslucent = a.getBoolean(R.styleable.Window_windowIsTranslucent, false);

        final Context context = getContext();
        final int targetSdk = context.getApplicationInfo().targetSdkVersion;
        final boolean targetPreL = targetSdk < android.os.Build.VERSION_CODES.LOLLIPOP;
        final boolean targetPreQ = targetSdk < Build.VERSION_CODES.Q;

        if (!mForcedStatusBarColor) {
            mStatusBarColor = a.getColor(R.styleable.Window_statusBarColor, 0xFF000000);
        }
        if (!mForcedNavigationBarColor) {
            mNavigationBarColor = a.getColor(R.styleable.Window_navigationBarColor, 0xFF000000);
            mNavigationBarDividerColor = a.getColor(R.styleable.Window_navigationBarDividerColor,
                    0x00000000);
        }
        if (!targetPreQ) {
            mEnsureStatusBarContrastWhenTransparent = a.getBoolean(
                    R.styleable.Window_enforceStatusBarContrast, false);
            mEnsureNavigationBarContrastWhenTransparent = a.getBoolean(
                    R.styleable.Window_enforceNavigationBarContrast, true);
        }

        WindowManager.LayoutParams params = getAttributes();

        
        
        if (!mIsFloating) {
            if (!targetPreL && a.getBoolean(
                    R.styleable.Window_windowDrawsSystemBarBackgrounds,
                    false)) {
                setFlags(FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS,
                        FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS & ~getForcedWindowFlags());
            }
            if (mDecor.mForceWindowDrawsBarBackgrounds) {
                params.privateFlags |= PRIVATE_FLAG_FORCE_DRAW_BAR_BACKGROUNDS;
            }
        }
        if (a.getBoolean(R.styleable.Window_windowLightStatusBar, false)) {
            decor.setSystemUiVisibility(
                    decor.getSystemUiVisibility() | View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
        }
        if (a.getBoolean(R.styleable.Window_windowLightNavigationBar, false)) {
            decor.setSystemUiVisibility(
                    decor.getSystemUiVisibility() | View.SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR);
        }

        

        

        int layoutResource;
        int features = getLocalFeatures();
        
        if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_title_icons;
            }
            
            removeFeature(FEATURE_ACTION_BAR);
            
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            
            
            layoutResource = R.layout.screen_progress;
            
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
            
            
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogCustomTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_custom_title;
            }
            
            removeFeature(FEATURE_ACTION_BAR);
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            
            
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
                layoutResource = a.getResourceId(
                        R.styleable.Window_windowActionBarFullscreenDecorLayout,
                        R.layout.screen_action_bar);
            } else {
                layoutResource = R.layout.screen_title;
            }
            
        } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            layoutResource = R.layout.screen_simple_overlay_action_mode;
        } else {
            
            layoutResource = R.layout.screen_simple;
            
        }

        mDecor.startChanging();
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }

        

        mDecor.finishChanging();

        return contentParent;
    }


```

虽然方法很长，但是其实很简单，主要是读取 Activity 自己设置的窗口风格属性，然后进行一些配置，比较重要的部分有：

1、设置 Window flags，如：

```
        if (a.getBoolean(R.styleable.Window_windowFullscreen, false)) {
            setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN & (~getForcedWindowFlags()));
        }


```

2、启用 Screen features，如：

```
        if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
            requestFeature(FEATURE_NO_TITLE);
        } else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
            
            requestFeature(FEATURE_ACTION_BAR);
        }


```

3、设置 StatusBar 和 NavigationBar 的背景色，如：

```
        if (!mForcedStatusBarColor) {
            mStatusBarColor = a.getColor(R.styleable.Window_statusBarColor, 0xFF000000);
        }
        if (!mForcedNavigationBarColor) {
            mNavigationBarColor = a.getColor(R.styleable.Window_navigationBarColor, 0xFF000000);
            mNavigationBarDividerColor = a.getColor(R.styleable.Window_navigationBarDividerColor,
                    0x00000000);
        }


```

4、设置 SystemUI flags，如：

```
        if (a.getBoolean(R.styleable.Window_windowLightStatusBar, false)) {
            decor.setSystemUiVisibility(
                    decor.getSystemUiVisibility() | View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
        }
        if (a.getBoolean(R.styleable.Window_windowLightNavigationBar, false)) {
            decor.setSystemUiVisibility(
                    decor.getSystemUiVisibility() | View.SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR);
        }



```

5、设置 PhoneWindow 的一些变量，如：

```
        mIsTranslucent = a.getBoolean(R.styleable.Window_windowIsTranslucent, false);
        
        if (!targetPreQ) {
            mEnsureStatusBarContrastWhenTransparent = a.getBoolean(
                    R.styleable.Window_enforceStatusBarContrast, false);
            mEnsureNavigationBarContrastWhenTransparent = a.getBoolean(
                    R.styleable.Window_enforceNavigationBarContrast, true);
        }



```

6、选取应该加载的布局资源 ID：

```
        

        int layoutResource;
        int features = getLocalFeatures();
        
        if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_title_icons;
            }
            
            removeFeature(FEATURE_ACTION_BAR);
            
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            
            
            layoutResource = R.layout.screen_progress;
            
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
            
            
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogCustomTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_custom_title;
            }
            
            removeFeature(FEATURE_ACTION_BAR);
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            
            
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
                layoutResource = a.getResourceId(
                        R.styleable.Window_windowActionBarFullscreenDecorLayout,
                        R.layout.screen_action_bar);
            } else {
                layoutResource = R.layout.screen_title;
            }
            
        } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            layoutResource = R.layout.screen_simple_overlay_action_mode;
        } else {
            
            layoutResource = R.layout.screen_simple;
            
        }

        mDecor.startChanging();
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);



```

这里根据 Activity 的主题设置取得最符合的布局文件 ID 后，调用 DecorView#onResourcesLoaded 去实例化一个相符合的 View 层次结构：

```
    void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
        if (mBackdropFrameRenderer != null) {
            loadBackgroundDrawablesIfNeeded();
            mBackdropFrameRenderer.onResourcesLoaded(
                    this, mResizingBackgroundDrawable, mCaptionBackgroundDrawable,
                    mUserCaptionBackgroundDrawable, getCurrentColor(mStatusColorViewState),
                    getCurrentColor(mNavigationColorViewState));
        }

        mDecorCaptionView = createDecorCaptionView(inflater);
        final View root = inflater.inflate(layoutResource, null);
        if (mDecorCaptionView != null) {
            if (mDecorCaptionView.getParent() == null) {
                addView(mDecorCaptionView,
                        new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
            }
            mDecorCaptionView.addView(root,
                        new ViewGroup.MarginLayoutParams(MATCH_PARENT, MATCH_PARENT));
        } else {
            
            addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        }
        mContentRoot = (ViewGroup) root;
        initializeElevation();
    }


```

这里不去考虑 DecorCaptionView 存在的情况，那么这里的主要工作就是：

*   调用 layoutInflater.inflate 去生成一个 View 层次结构，将 DecorView.mContentRoot 指向该 View 层级结构的 rootView，并返回该 root View：

```
final View root = inflater.inflate(layoutResource, null);


```

*   通过 View#addView 将这个 root View 加入到当前 DecorView 的层次结构中，也就是说，该 root View 变成了 DecorView 的子 VIew，在 Decor 的层级结构中变成了 DecorView 的子节点：

```
addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));


```

这里我本地调试，取到的布局资源 ID 为：R.layout.screen_simple，对应的 xml 是：frameworks/base/core/res/res/layout/screen_simple.xml。

```
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

布局文件本身是一个 LinearLayout，包含一个 ViewStub 和 FrameLayout，再参考 dumpsys 出来的信息，是对上了的：

```
      DecorView@7b3553[MainActivity]
        android.widget.LinearLayout{fe72e90 V.E...... ........ 0,0-1200,1904}
          android.view.ViewStub{8fee389 G.E...... ......I. 0,0-0,0 #10201c3 android:id/action_mode_bar_stub}
          android.widget.FrameLayout{2d0cf8e V.E...... ........ 0,48-1200,1904}


```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3e7212b8b664a6ea8ee32aa6071a6ce~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

7、找到 ID_ANDROID_CONTENT 对应的 View，并且返回。

```
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }

        

        mDecor.finishChanging();

        return contentParent;



```

因为在第 6 步，我们已经得到了一个基于 R.layout.screen_simple 生成的 View 层次结构，那么就可以通过 findViewById 找到一个 ID 为 ID_ANDROID_CONTENT 的子 View，然后返回该子 View。

```
    
     * The ID that the main layout in the XML layout file should have.
     */
    public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;



```

那么最终，通过 PhoneWindow#generateLayout 生成了一个 View 层次结构，并且找到其 ID 为 R.id.content 的子 View 并且返回该子 View，然后赋值给了 PhoneWindow.mContentParent，这里看到返回的是一个 FrameLayout：

```
android.widget.FrameLayout{2d0cf8e V.E...... ........ 0,48-1200,1904}


```

最后将 PhoneWindow.mContentParent 指向这个返回的 ViewGroup 对象。

### 3.2.3 PhoneWindow#setContentView

之前分析 PhoneWindow#installDecor 的时候，说了 AppCompatActivity 由于重写了 Activity#setContentView 方法，导致先走了上面分析的 PhoneWindow#installDecor 方法，PhoneWindow#setContentView 会在稍晚的一个时间点被调用。

先看下调用堆栈信息：

```
08-20 09:38:31.734  6254  6254 I ukynho_decor: PhoneWindow#setContentView ---- title = null  mContentParent = android.widget.FrameLayout{2d0cf8e V.E...... ........ 0,48-1200,1904}     view = androidx.appcompat.widget.ActionBarOverlayLayout{7f9d3af V.E...... ........ 0,0-1200,1856 #7f07004d app:id/decor_content_parent}
08-20 09:38:31.734  6254  6254 I ukynho_decor: java.lang.Throwable
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at com.android.internal.policy.PhoneWindow.setContentView(PhoneWindow.java:476)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at com.android.internal.policy.PhoneWindow.setContentView(PhoneWindow.java:471)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at androidx.appcompat.app.AppCompatDelegateImpl.createSubDecor(AppCompatDelegateImpl.java:855)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at androidx.appcompat.app.AppCompatDelegateImpl.ensureSubDecor(AppCompatDelegateImpl.java:659)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at androidx.appcompat.app.AppCompatDelegateImpl.setContentView(AppCompatDelegateImpl.java:552)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at androidx.appcompat.app.AppCompatActivity.setContentView(AppCompatActivity.java:161)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at com.qq.reader.activity.MainActivity.onCreate(MainActivity.java:29)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at android.app.Activity.performCreate(Activity.java:8012)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at android.app.Activity.performCreate(Activity.java:7996)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1311)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:3525)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3713)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:85)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:135)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:95)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2156)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at android.os.Handler.dispatchMessage(Handler.java:106)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at android.os.Looper.loop(Looper.java:236)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at android.app.ActivityThread.main(ActivityThread.java:7814)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at java.lang.reflect.Method.invoke(Native Method)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:621)
08-20 09:38:31.734  6254  6254 I ukynho_decor:  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:997)


```

再看下方法的具体内容：

```
    @Override
    public void setContentView(View view, ViewGroup.LayoutParams params) {
        
        
        
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            view.setLayoutParams(params);
            final Scene newScene = new Scene(mContentParent, view);
            transitionTo(newScene);
        } else {
            mContentParent.addView(view, params);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }


```

到达这里时，通过打印的 log，可以知道，此时：

1）、我们前面的分析 PhoneWindow#generateLayout 返回了一个 ViewGroup 对象：

```
android.widget.FrameLayout{2d0cf8e V.E...... ........ 0,48-1200,1904}


```

并且 PhoneWindow 将其成员变量 mContentParent 指向该对象。

再看到我们加在这边的 log，结果是一致的：

```
mContentParent = android.widget.FrameLayout{2d0cf8e V.E...... ........ 0,48-1200,1904}，


```

2）、根据 log，此时传入的 View 是 AppCompatActivity 那边已经加载好的 View 层次结构的 root View：

```
view = androidx.appcompat.widget.ActionBarOverlayLayout{7f9d3af V.E...... ........ 0,0-1200,1856 


```

然后通过：

```
mContentParent.addView(view, params);


```

将传入的 View 作为子 View 加入的 mContentParent 的层级结构中，从而将 DecorView 与 AppCompatActivity 部分连接起来：

```
      DecorView@7b3553[MainActivity]
        android.widget.LinearLayout{fe72e90 V.E...... ........ 0,0-1200,1904}
          android.view.ViewStub{8fee389 G.E...... ......I. 0,0-0,0 #10201c3 android:id/action_mode_bar_stub}
          android.widget.FrameLayout{2d0cf8e V.E...... ........ 0,48-1200,1904}
            androidx.appcompat.widget.ActionBarOverlayLayout{7f9d3af V.E...... ........ 0,0-1200,1856 #7f07004d app:id/decor_content_parent}


```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f606bf519cb744208d866330b573e978~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

3)、后续 AppCompatActivity 部分再调用 LayoutInflater.inflate，解析我们自定义的 layout 布局文件 R.layout.activity_main，生成我们定义的 View 层次结构：

```
android.widget.RelativeLayout{5692f45 V.E...... ........ 0,0-1200,1728}
	androidx.appcompat.widget.AppCompatButton{e8e569a VFED..C.. ........ 200,200-376,296 
	androidx.appcompat.widget.AppCompatButton{619fbcb VFED..C.. ........ 0,296-243,392 
	androidx.appcompat.widget.AppCompatButton{dca3fa8 VFED..C.. ........ 0,392-220,488 


```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe2816b9cf2749a1b841b5585c5a3285~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

并且 AppCompatActivity 部分在调用 LayoutInflater.inflate 的时候，指定了 parent View，从而将我们自定义的 View 层次结构添加到 AppCompatActivity 自己的 View 层级结构中，指定的 parent View 为：

```
androidx.appcompat.widget.ContentFrameLayout{c38a3bc V.E...... ........ 0,128-1200,1856 


```

从而将 AppCompatActivity 和我们的自定义布局 RelativeLayout 连接了起来：

```
          androidx.appcompat.widget.ContentFrameLayout{c38a3bc V.E...... ........ 0,128-1200,1856 
            android.widget.RelativeLayout{5692f45 V.E...... ........ 0,0-1200,1728}
              androidx.appcompat.widget.AppCompatButton{e8e569a VFED..C.. ........ 200,200-376,296 
              androidx.appcompat.widget.AppCompatButton{619fbcb VFED..C.. ........ 0,296-243,392 
              androidx.appcompat.widget.AppCompatButton{dca3fa8 VFED..C.. ........ 0,392-220,488 


```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/14d6dea66a0843f5b5796071ce2ab626~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

4)、这里看到 AppCompatActivity 起到了一个连接的作用，首先它既调用了 PhoneWIndow#setContentView，将自己的内部的 View 层次结构的根 View 传到 PhoneWindow 中，在 PhoneWindow 作为子 View 加入到了 DecorView 的层级结构中；另一方面，在自己内部，又负责解析我们的自定义布局，并且将我们的自定义布局的根 View 作为子 View 加入到它的 View 层级结构中。

AppCompatActivity 的部分为：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3828c747ad8243bca375188d3de3e921~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

最终形成了最终的 View 层级结构：

```
  DecorView@b8fdc2b[MainActivity]
    android.widget.LinearLayout{f289afc V.E...... ........ 0,0-1080,1344}
      android.view.ViewStub{e6af585 G.E...... ......I. 0,0-0,0 #10201c3 android:id/action_mode_bar_stub}
      android.widget.FrameLayout{4373fda V.E...... ........ 0,74-1080,1344}
        androidx.appcompat.widget.ActionBarOverlayLayout{d96cc0b V.E...... ........ 0,0-1080,1270 #7f07004d app:id/decor_content_parent}
          androidx.appcompat.widget.ContentFrameLayout{6e26ae8 V.E...... ........ 0,112-1080,1270 #1020002 android:id/content}
            android.widget.RelativeLayout{26a6501 V.E...... ........ 0,0-1080,1158}
              androidx.appcompat.widget.AppCompatButton{1d97fa6 VFED..C.. ........ 0,0-176,96 #7f070058 app:id/getSize}
              androidx.appcompat.widget.AppCompatButton{ee41de7 VFED..C.. ........ 0,96-243,192 #7f07009b app:id/startAnother}
          androidx.appcompat.widget.ActionBarContainer{fe74d94 V.ED..... ........ 0,0-1080,112 #7f070029 app:id/action_bar_container}
            androidx.appcompat.widget.Toolbar{397503d V.E...... ........ 0,0-1080,112 #7f070027 app:id/action_bar}
              androidx.appcompat.widget.AppCompatTextView{e0df032 V.ED..... ........ 32,29-300,83}
              androidx.appcompat.widget.ActionMenuView{3cab183 V.E...... ........ 1080,0-1080,112}
            androidx.appcompat.widget.ActionBarContextView{db5af00 G.E...... ......I. 0,0-0,0 #7f07002f app:id/action_context_bar}
    android.view.View{badb339 V.ED..... ........ 0,1344-1080,1440 #1020030 android:id/navigationBarBackground}
    android.view.View{ca6dd7e V.ED..... ........ 0,0-1080,74 #102002f android:id/statusBarBackground}


```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/626f4143120b4c7eb63fa5186b28c8e5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 3.2.4 总结

DecorView 的 View 层级结构，从上面分析，其实是由三个层级结构组成的：

1)、PhoneWindow 根据 Activity 设置的主题风格，先生成了一个 View 层级结构，这部分是最顶级的；

2)、我们自己定义的 View 层级结构，这个是根据我们通过 Activity#setContentView 中传入的 xml 类型的 layout 文件解析出的 View 层级结构，属于第三级；

3)、AppCompatActivity 也有自己的层级结构，这部分属于第二级，在 DecorView 的 View 层级结构和我们自定义布局的 View 层级结构之间起到了连接作用。

另外，我们通过打断点，也可以看到有三次 LayoutInflater.inflate 调用。

第一次：在 PhoneWindow#generateLayout 中，这里我们根据 Activity 的主题风格，选择了一个布局 ID，然后调用 LayoutInflater.inflate 生成了一个 View 层级结构，根 View 为：

```
android.widget.LinearLayout{f289afc V.E...... ........ 0,0-1080,1344}


```

然后在 DecorView#onResourcesLoaded 方法中，通过 View#addView 的方式将该 root View 作为子 View 加入到自己的 mChildren 数组中：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c2ca7ba045e4f42b99648609a34f51d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

第二次：由 AppCompatActivity 调用，生成了 AppCompatActivity 的 View 层级结构，根 View 为：

```
androidx.appcompat.widget.ActionBarOverlayLayout{d96cc0b V.E...... ........ 0,0-1080,1270 


```

然后 AppCompatActivity 又调用 PhoneWindow#setContentVIew 将自己的根 View 作为子 View 加入到 DecorView 的 View 层级结构中去：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f29e64ac65248c592a07e0caa8ea6db~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

第三次：由 AppCompatActivity 调用，解析我们在自己定义的 Activity 中通过 Activity#setContentView 传入的布局 ID，生成了我们定义的 View 层级结构，根 View 为：

```
 android.widget.RelativeLayout{26a6501 V.E...... ........ 0,0-1080,1158}


```

并且这里在调用 LayoutInflater.inflate 的时候，指定了 parent View 为

```
androidx.appcompat.widget.ContentFrameLayout{6e26ae8 V.E...... ........ 0,112-1080,1270 


```

从而将我们自定义的 View 层级结构作为子 View 加入到了 AppCompatActivity 的 VIew 层级结构中：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/435c7de2ccbe48e2af8ce84ab052fd52~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

回答一下评论的一个疑问，因为篇幅较长，所以放到这里。

1）、在 PhoneWindow.installDecor 中，首先我们创建了一个顶层 View，DeocrView：

```
         if (mDecor == null) {
             mDecor = generateDecor(-1);
             mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
             mDecor.setIsRootNamespace(true);
             if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                 mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
             }
         }


```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/885f8ad305b646a7ab3f079e9c6dc7e2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

2）、接着调用 PhoneWindow.generateLayout，我们首先根据 Feature 等相关设置，找到了一个资源文件：

```
             layoutResource = R.layout.screen_simple;


```

screen_simple.xml 内容为：

```
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

接着 PhoneWindow.generateLayout 调用 DecorView.onResourcesLoaded 方法，并且传入该资源文件 id：

```
         mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);


```

3）、DecorView.onResourcesLoaded 中，首先根据该资源文件，通过 LayoutInflater.inflate 生成了一个 View 层级结构：

```
         final View root = inflater.inflate(layoutResource, null);


```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c05b5d3f2a3940c19b53ace8ac8f8177~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

并且将该 View 层级结构的顶层 View，LinearLayout，作为子 View 添加到了 DeocrView 中：

```
             
             addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));


```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b12ca934d0ef4e50a41b05c0ba9e56c1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

然后将 DecorView 的成员变量 mContentRoot 指向了该 LinearLayout：

```
         mContentRoot = (ViewGroup) root;


```

4）、回到 PhoneWindow.generateLayout，经过 DecorView.onResourcesLoaded 方法后，我们已经解析了 R.layout.screen_simple，并且 R.layout.screen_simple 中是有一个 ID 为 R.id.content 的 View 的，那么将局部变量 contentParent 指向该 View：

```
         mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
 
         ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);


```

并且 PhoneWindow.generateLayout 方法最后返回了该 contentParent，那么 PhoneWindow.mContentParent 指向的就是该 View。

```
             mContentParent = generateLayout(mDecor);


```

5）、总结起来就是：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8c455bec2aa44d1b7ef2a3d8b48a5e9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

ID 为 R.id.content 的 View 是 DecorView 的直接子 View，LinearLayout，中的一个子 View，PhoneWindow 的成员变量 mContentParent 指向了该 View。
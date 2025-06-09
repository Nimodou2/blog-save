> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7182894778408206394)

> 之前在分析 WMS 的窗口层级结构时，看到 WindowContainer 的定义为： 看到 WindowContainer 其实还是有一个父类 ConfigurationContainer 的，但是由于 Config

之前在分析 WMS 的窗口层级结构时，看到 WindowContainer 的定义为：

```
class WindowContainer<E extends WindowContainer> extends ConfigurationContainer<E>
        implements Comparable<WindowContainer>, Animatable, SurfaceFreezer.Freezable {


```

看到 WindowContainer 其实还是有一个父类 ConfigurationContainer 的，但是由于 ConfigurationContainer 并不参与层级结构的构建，所以当时并没有对 ConfigurationContainer 进行介绍，本篇就详细分析一下这个类的作用。

首先 ConfigurationContainer 的定义为：

```
 * Contains common logic for classes that have override configurations and are organized in a
 * hierarchy.
 */
public abstract class ConfigurationContainer<E extends ConfigurationContainer> {


```

包含具有覆盖 configuration 并以层次结构组织的类的通用逻辑。

从名字我们可以知道 ConfigurationContainer 是一个 container，类似 WindowContainer 的容器，不过这个容器盛放的是 Configuration 对象，而 WindowContainer 是窗口的容器。

ConfigurationContainer 只有两个直接子类，WindowContainer 和 WindowProcessController，WindowContainer 之前详细介绍过自不必多说，而 WindowProcesController 则保存了一些进程相关的信息，是 WM 和 AM 沟通的媒介，和本文要分析的内容也没太大关系，因此继承关系这部分可以暂时忽略。

如上面所说 ConfigurationContainer 虽然没有参与层级结构的构建，但是它却借助了 WindowContainer 构建起来的层级结构，从上到下进行了进行 Configuration 的分发，那么本篇的重点并不在于层级结构，而是 Configuration。

先来简单介绍一下 Configuration 的相关内容，方便我们后续分析。

Configuration 类的定义为：

```
 * This class describes all device configuration information that can
 * impact the resources the application retrieves.  This includes both
 * user-specified configuration options (locale list and scaling) as well
 * as device configurations (such as input modes, screen size and screen orientation).
 * <p>You can acquire this object from {@link Resources}, using {@link
 * Resources#getConfiguration}. Thus, from an activity, you can get it by chaining the request
 * with {@link android.app.Activity#getResources}:</p>
 * <pre>Configuration config = getResources().getConfiguration();</pre>
 */
public final class Configuration implements Parcelable, Comparable<Configuration> {


```

这个类描述了所有可能影响应用程序检索的资源的设备配置信息。这包括用户指定的配置选项 (语言区域列表和缩放) 以及设备的配置(如输入模式、屏幕大小和屏幕方向)。

这个类不管是做 App 还是 Framework 的都会经常接触，重要性不言而喻。它的每个成员变量单拎出来都是重量级，这里并不一一介绍，作为 WMS 相关的开发，我平时关注比较多的则是代表方向的 orientation，以及代表屏幕尺寸的 screenWidthDp 和 screenHeightDp 等，当然这些大家都熟悉的东西也不是本篇重点，重点是 Configuration 的成员变量 windowConfiguration。

```
    
     * Configuration relating to the windowing state of the object associated with this
     * Configuration. Contents of this field are not intended to affect resources, but need to be
     * communicated and propagated at the same time as the rest of Configuration.
     * @hide
     */
    @TestApi
    public final WindowConfiguration windowConfiguration = new WindowConfiguration();


```

WindowConfiguration 持有了窗口状态相关的配置信息，这些信息对于 App 开发者来说基本用不到，但是仍然要作为 Configuration 的一部分，跟随 Configuration 一起进行分发。

```
 * Class that contains windowing configuration/state for other objects that contain windows directly
 * or indirectly. E.g. Activities, Task, Displays, ...
 * The test class is {@link com.android.server.wm.WindowConfigurationTests} which must be kept
 * up-to-date and ran anytime changes are made to this class.
 * @hide
 */
@TestApi
public class WindowConfiguration implements Parcelable, Comparable<WindowConfiguration>


```

包含窗口配置或状态信息的类，用于那些直接或间接包含窗口的容器。

3.1 mBounds、mAppBounds 和 mMaxBounds
-----------------------------------

```
    
     * bounds that can differ from app bounds, which may include things such as insets.
     *
     * TODO: Investigate combining with {@link #mAppBounds}. Can the latter be a product of the
     * former?
     */
    private final Rect mBounds = new Rect();
​
    
     * {@link android.graphics.Rect} defining app bounds. The dimensions override usages of
     * {@link DisplayInfo#appHeight} and {@link DisplayInfo#appWidth} and mirrors these values at
     * the display level. Lower levels can override these values to provide custom bounds to enforce
     * features such as a max aspect ratio.
     */
    private Rect mAppBounds;
​
    
     * The maximum {@link Rect} bounds that an app can expect. It is used to report value of
     * {@link WindowManager#getMaximumWindowMetrics()}.
     */
    private final Rect mMaxBounds = new Rect();


```

*   mBounds，这个是最常用的 bounds，代表了 container 的边界。
    
    各种 container，比如 Task，调用 getBounds 方法，返回的就是这个 bounds，这个 bounds 代表的也就是 Task 的边界：
    
    ```
        
         * Returns the effective bounds of this container, inheriting the first non-empty bounds set in
         * its ancestral hierarchy, including itself.
         */
        public Rect getBounds() {
            mReturnBounds.set(getConfiguration().windowConfiguration.getBounds());
            return mReturnBounds;
        }
    
    
    ```
    
*   mAppBounds，用到的地方不多，代表的是 App 的可用边界，对比 mBounds，一个很明显的区别就是 mAppBounds 的计算是考虑到了 insets 的，而 mBounds 的边界是不考虑 insets 的，比如通过 adb shell dumpsys activity a 打印出的全屏状态下的某个 ActivityRecord 的 Configuration：
    
    ```
    CurrentConfiguration={0.93 ?mcc?mnc [zh_CN_#Hans] ldltr sw360dp w360dp h736dp 320dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 720, 1612) mAppBounds=Rect(0, 44 - 720, 1516) mMaxBounds=Rect(0, 0 - 720, 1612) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} as.6 s.1 fontWeightAdjustment=0}
    
    
    ```
    
    这里的 mBounds 为：
    
    ```
    mBounds=Rect(0, 0 - 720, 1612)
    
    
    ```
    
    即整个屏幕的大小，而 mAppBounds 为：
    
    ```
    mAppBounds=Rect(0, 44 - 720, 1516)
    
    
    ```
    
    明显是减去了状态栏和导航栏的高度。
    
    mAppBounds 一个比较重要的作用是，计算 Configuration 的 screenWidthDp 和 screenHeightDp，比如还是以上的 Configuration，它的 screenWidthDp 和 screenHeightDp 是：
    
    ```
    w360dp h736dp
    
    
    ```
    
    再结合它的 densityDpi：
    
    ```
    320dpi
    
    
    ```
    
    很明显，screenWidthDp 和 screenHeightDp 是根据 mAppBounds 计算的，而不是 mBounds，而实际上也的确是这样的，具体计算过程在 Task.computeConfigResourceOverrides 方法中，这里就不详细分析了。
    
*   mMaxBounds，代表了一个 container 能够得到的最大 bounds，比如分屏下，一个 ActivityRecord 的各个 bounds 可能是：
    
    ```
    mBounds=Rect(0, 0 - 720, 770) mAppBounds=Rect(0, 44 - 720, 770) mMaxBounds=Rect(0, 0 - 720, 1612)
    
    
    ```
    
    mBounds 代表的是当前 ActivityRecord 的大小，mAppBounds 在 mBounds 的基础上减去的状态栏的高度，而 mMaxBounds 代表的则是这个 ActivityRecord 可以拥有的最大尺寸，即全屏。
    
    mMaxBounds 的出现应该是在 Display.getRealSize 被弃用，改为通过 WindowMetrics 获取屏幕尺寸时候。App 如果想要获取当前 activity 的边界，应该使用 WindowManager.getCurrentWindowMetrics，如果想要知道 App 能够占据的最大边界，那么应该使用 WindowManager.getMaximumWindowMetrics。
    
    大部分时候，mMaxBounds 代表的都是屏幕的尺寸，但是少部分特殊模式下，它可能会小于屏幕的实际物理尺寸。
    

3.2 mRotation
-------------

```
    
     * The current rotation of this window container relative to the default
     * orientation of the display it is on (regardless of how deep in the hierarchy
     * it is). It is used by the configuration hierarchy to apply rotation-dependent
     * policy during bounds calculation.
     */
    private int mRotation = ROTATION_UNDEFINED;


```

当前 container 的旋转角度，只和当前 container 所在的 display 相关，和这个 container 在层级结构中的哪一级无关。

一般来说，各级 container 都是直接继承 Display 的 rotation 的，但是 ActivityRecord 中针对尺寸兼容模式有了额外的逻辑，这部分看的比较少，暂时略过。

3.3 mWindowingMode 和 mDisplayWindowingMode
------------------------------------------

```
    
    private @WindowingMode int mWindowingMode;
​
    
    private @WindowingMode int mDisplayWindowingMode;


```

*   mWindowingMode，代表了当前 container 处于哪一种多窗口模式里。
*   mDisplayWindowingMode，代表了当前 container 所在的 Display 处于哪一种多窗口模式里。

目前的多窗口模式有以下几种：

```
    
    public static final int WINDOWING_MODE_UNDEFINED = 0;
    
    public static final int WINDOWING_MODE_FULLSCREEN = 1;
    
    public static final int WINDOWING_MODE_PINNED = 2;
    
    
    public static final int WINDOWING_MODE_SPLIT_SCREEN_PRIMARY = 3;
    
     * The containers adjacent to the {@link #WINDOWING_MODE_SPLIT_SCREEN_PRIMARY} container in
     * split-screen mode.
     * NOTE: Containers launched with the windowing mode with APIs like
     * {@link ActivityOptions#setLaunchWindowingMode(int)} will be launched in
     * {@link #WINDOWING_MODE_FULLSCREEN} if the display isn't currently in split-screen windowing
     * mode
     */
    
    public static final int WINDOWING_MODE_SPLIT_SCREEN_SECONDARY = 4;
    
     * Alias for {@link #WINDOWING_MODE_SPLIT_SCREEN_SECONDARY} that makes it clear that the usage
     * points for APIs like {@link ActivityOptions#setLaunchWindowingMode(int)} that the container
     * will launch into fullscreen or split-screen secondary depending on if the device is currently
     * in fullscreen mode or split-screen mode.
     */
    
    public static final int WINDOWING_MODE_FULLSCREEN_OR_SPLIT_SCREEN_SECONDARY =
            WINDOWING_MODE_SPLIT_SCREEN_SECONDARY;
    
    
    public static final int WINDOWING_MODE_FREEFORM = 5;
    
    public static final int WINDOWING_MODE_MULTI_WINDOW = 6;


```

*   WINDOWING_MODE_UNDEFINED，未定义，一般 container 刚刚创建的时候就是这个模式，但是后续必须要为其选择一种非 WINDOWING_MODE_UNDEFINED 的窗口模式。
*   WINDOWING_MODE_FULLSCREEN，全屏模式，占据了整个屏幕空间，很常见就不讨论了。
*   WINDOWING_MODE_PINNED，画中画模式，App 中的 Activity 能够以一个小窗口的形式启动，并且可以在其他 App 的上方显示，常见于视频播放。
*   WINDOWING_MODE_SPLIT_SCREEN_PRIMARY 和 WINDOWING_MODE_SPLIT_SCREEN_SECONDARY，分屏模式，并排显示两个 App，primary 和 secondary 分别代表了左右或上下半屏的 App 所处的窗口模式。
*   WINDOWING_MODE_FREEFORM，自由窗口模式，不同于 PIP，这个模式下整个 App 都位于小窗口模式。
*   WINDOWING_MODE_MULTI_WINDOW，Android13 的分屏模式对应的窗口模式，Android12 应该没有用到的地方。

窗口模式主要是为多窗口模式设计，在进入多窗口模式前，container 都是 fullscreen 模式。当把某个 App 对应的 Task 移动到多窗口模式，如 freeform 时，该 Task 对应的 Configuration.windowConfiguration.mWindowingMode 会被置为 WINDOWING_MODE_FREEFORM，Task 中的子 container 也会继承 Task 的这一属性，即 Task 中的 ActivityRecord，ActivityRecord 的 WindowState 等，它们对应的 Configuration.windowConfiguration.mWindowingMode 全会被置为 WINDOWING_MODE_FREEFORM。

3.4 mActivityType
-----------------

```
    
    private @ActivityType int mActivityType;


```

当前 container 的 activity 类型。

activity 类型有以下几种：

```
    
    public static final int ACTIVITY_TYPE_UNDEFINED = 0;
    
    public static final int ACTIVITY_TYPE_STANDARD = 1;
    
    public static final int ACTIVITY_TYPE_HOME = 2;
    
    public static final int ACTIVITY_TYPE_RECENTS = 3;
    
    public static final int ACTIVITY_TYPE_ASSISTANT = 4;
    
    public static final int ACTIVITY_TYPE_DREAM = 5;


```

*   ACTIVITY_TYPE_UNDEFINED，一般 container 刚刚创建的时候就是这个模式，但是后续必须要为其选择一种非 ACTIVITY_TYPE_UNDEFINED 的 activity 类型。
*   ACTIVITY_TYPE_STANDARD，标准类型，大部分的 App 都是这种类型。
*   ACTIVITY_TYPE_HOME，Launcher 类型的 App，如 google 原生 Launcher，Launcher3 的 ActivityRecord 都是这个类型。
*   ACTIVITY_TYPE_RECENTS，Recents 或 Overview 类型，系统中只有一个 activity 是这个类型的，即”com.android.launcher3/com.android.quickstep.RecentsActivity“。
*   ACTIVITY_TYPE_ASSISTANT，Assistant 类型，目前接触的不多，应该和 google 语音助手服务 "voice_interaction_service" 之类的相关。
*   ACTIVITY_TYPE_DREAM，Dream 类型，完全没有遇到过，目前应该只有 DreamActivity 对应的 ActivityRecord 能够被设置为此类型。

具体的 Activity 类型的计算在 ActivityRecord.setActivityType 中。

ActivityType 和 WindowingMode 有一点不同的是，在将某个 App 移动到多窗口模式时，是先设置 Task 的 WindowingMode，Task 中的子 container 继承 Task 的 WindowingMode。而 Task 的 ActivityType 则是由 Task 中的 ActivityRecord 的 ActivityType 类型决定，ActivityRecord 在其构造方法中会去调用 ActivityRecord.setActivityType 计算 ActivityRecord 的类型，随后 ActivityRecord 在添加到 Task 中时，Task 将自己的 ActivityType 设置为 ActivityRecord 的类型。

将 container 按照 ActivityType 进行分类，还是为了便于管理和使用 container，之前在分析 WindowContainer 类的时候就看到，HOME 类型的 Task 是支持嵌套的，所有 HOME 类型的 Task 会统一放到一个 HOME 类型的 Task 中去管理，比如开机的时候的默认 Launcher 是 Launcher1，那么系统会首先创建一个 HOME 类型的 Task#1，作为 HOME 类型 Task 的 root，接着为 Launcher1 创建 Task#2 用来存放 Launcher1 的 activity，并且把 Task#2 作为子 container 添加到 Task#1 中。后续如果我们把默认 Launcher 切换为 Launcher2，那么此时系统会再创建一个 Task#3，作为 Launcher2 的 Task，此时 Task#3 也会被作为 Task#1 的子 container 添加到 Task#3 中，和 Task#2 处于同一层级。

对于特殊窗口类型和 Activity 类型的 root Task，作为 Task 容器的 TaskDisplayArea，也创建了一些专门的成员变量来存放它们的引用：

```
    
    
    private Task mRootHomeTask;
    private Task mRootPinnedTask;
    private Task mRootSplitScreenPrimaryTask;
​
    
    private Task mRootRecentsTask;


```

这些 Task 的特点之一是同一时间只能存在一个，如果 mRootHomeTask 不为空的情况下再去添加一个 HOME 类型的 root Task，就会抛出异常。

3.5 mAlwaysOnTop
----------------

```
    
    private @AlwaysOnTop int mAlwaysOnTop;


```

mAlwaysOnTop 用来声明当前 container 是否总是处于顶层。

有以下几个值：

```
    
    private static final int ALWAYS_ON_TOP_UNDEFINED = 0;
    
    private static final int ALWAYS_ON_TOP_ON = 1;
    
    private static final int ALWAYS_ON_TOP_OFF = 2;


```

比较简单不再赘述，再看下有哪些类型的 container 是可以总是处于顶层的：

```
    
     * Returns true if the container associated with this window configuration is always-on-top of
     * its siblings.
     * @hide
     */
    public boolean isAlwaysOnTop() {
        if (mWindowingMode == WINDOWING_MODE_PINNED) return true;
        if (mActivityType == ACTIVITY_TYPE_DREAM) return true;
        if (mAlwaysOnTop != ALWAYS_ON_TOP_ON) return false;
        return mWindowingMode == WINDOWING_MODE_FREEFORM
                    || mWindowingMode == WINDOWING_MODE_MULTI_WINDOW;
    }


```

对于 PIP 模式和 DREAM 类型的 container，是需要用于位于顶层的，其他的 container，即使设置了 mAlwaysOnTop 为 ALWAYS_ON_TOP_ON，也需要是 WINDOWING_MODE_FREEFORM 或 WINDOWING_MODE_MULTI_WINDOW 类型才行。

至于这个” 总是位于顶层 “，也不是将其置于所有 container 之上，而是置于与它处于同一层级的 container 之上，比如一个 PIP 模式的 Task，只可能位于 TaskDisplayArea 的顶层，即其他 Task 之上，而不可能超出 TaskDisplayArea，位于 StatusBar 的上面。

至此，Configuration 类和 WindowConfiguration 类的相关知识介绍的差不多了，可以进行下一步的 ConfigurationContainer 的介绍了。

ConfigurationContainer 既然是 Configuration 的容器，那么它肯定是存储了一些 Configuration 了，都有哪些呢？

ConfigurationContainer 的成员变量也就那几个，不难看到和 Configuration 相关的有 mRequestedOverrideConfiguration、mResolvedOverrideConfiguration、mFullConfiguration 和 mMergedOverrideConfiguration。

*   mRequestedOverrideConfiguration
    
    ```
        
         * Contains requested override configuration settings applied to this configuration container.
         */
        private Configuration mRequestedOverrideConfiguration = new Configuration();
    
    
    ```
    
    包含了应用到当前 container 的请求的覆盖 Configuration。
    
*   mResolvedOverrideConfiguration
    
    ```
        
         * Contains the requested override configuration with parent and policy constraints applied.
         * This is the set of overrides that gets applied to the full and merged configurations.
         */
        private Configuration mResolvedOverrideConfiguration = new Configuration();
    
    
    ```
    
    包含了应用了父容器和策略限制下的请求的覆盖 Configuration。这是应用到 mFullConfiguration 和 mMergedOverrideConfiguration 的覆盖 Configuration 集合。
    
*   mFullConfiguration
    
    ```
        
         * Contains full configuration applied to this configuration container. Corresponds to full
         * parent's config with applied {@link #mResolvedOverrideConfiguration}.
         */
        private Configuration mFullConfiguration = new Configuration();
    
    
    ```
    
    包含了应用到此 container 的完整 Configuration。相当于父容器的 Configuration 加上 mResolvedOverrideConfiguration。
    
*   mMergedOverrideConfiguration
    
    这个成员变量放在最后一节分析。
    

有点抽象，还是从它们作用的地方入手，看下这些个成员的作用都是什么。

4.1 例子：将 Task 设置为自由窗口模式
-----------------------

直接拿一个具体的例子看下。

假如我想将一个 Task 设置为自由窗口模式，那么我期望的最终结果应该是，这个 Task 持有的 mFullConfiguration 成员变量中的 WindowingMode 变为 WINDOWING_MODE_FREEFORM，而我这边可能的做法则是拿到这个 Task 之后，调用系统已经支持的 ConfigurationContainer.setWindowingMode 方法来设置这个 Task 的 WindowingMode 为 WINDOWING_MODE_FREEFORM，如 ActivityClientController.toggleFreeformWindowingMode 方法的做法：

```
    @Override
    public void toggleFreeformWindowingMode(IBinder token) {
        final long ident = Binder.clearCallingIdentity();
        try {
            synchronized (mGlobalLock) {
                
​
                if (rootTask.inFreeformWindowingMode()) {
                    
                } else {
                    rootTask.setWindowingMode(WINDOWING_MODE_FREEFORM);
                }
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }


```

这里的 rootTask 是我想移动到自由窗口模式的那个 Task，这里为该 Task 调用了 setWindowingMode 方法，忽略中间的过程，这最终会调用到 ConfigurationContainer 的 setWindowingMode 方法。

首先看一下 setWindowingMode 方法：

```
    
    public void setWindowingMode(/*@WindowConfiguration.WindowingMode*/ int windowingMode) {
        mRequestsTmpConfig.setTo(getRequestedOverrideConfiguration());
        mRequestsTmpConfig.windowConfiguration.setWindowingMode(windowingMode);
        onRequestedOverrideConfigurationChanged(mRequestsTmpConfig);
    }


```

这一步也很简单：

1）、首先调用 getRequestedOverrideConfiguration 方法：

```
    
    public Configuration getRequestedOverrideConfiguration() {
        return mRequestedOverrideConfiguration;
    }


```

将当前 Task 的 mRequestedOverrideConfiguration 的值赋值给一个临时的变量 mRequestsTmpConfig，毕竟不可能说每次设置一个新的 WindowingMode 的时候就把之前请求的一些配置信息全部都抛弃了，肯定要在之前已经请求的信息的基础上修改。

2）、接着将请求的这个 WindowingMode 先保存在 mRequestsTmpConfig 中。

3）、最后调用 onRequestedOverrideConfigurationChanged 方法，传入的则是携带了新请求的 WindowingMode 信息的 mRequestsTmpConfig。

4.2 mRequestedOverrideConfiguration 成员变量和 onRequestedOverrideConfigurationChanged 成员方法
--------------------------------------------------------------------------------------

ConfigurationContainer 的 onRequestedOverrideConfigurationChanged 方法的内容为：

```
    
     * Update override configuration and recalculate full config.
     * @see #mRequestedOverrideConfiguration
     * @see #mFullConfiguration
     */
    public void onRequestedOverrideConfigurationChanged(Configuration overrideConfiguration) {
        
        
        mHasOverrideConfiguration = !Configuration.EMPTY.equals(overrideConfiguration);
        mRequestedOverrideConfiguration.setTo(overrideConfiguration);
        
        final ConfigurationContainer parent = getParent();
        onConfigurationChanged(parent != null ? parent.getConfiguration() : Configuration.EMPTY);
    }


```

过滤不重要的因素，首先看到：

1）、mHasOverrideConfiguration 表明了当前 container 是否请求了一些特殊的配置，至于请求的配置自然是存放在 mRequestedOverrideConfiguration 中：

```
        mHasOverrideConfiguration = !Configuration.EMPTY.equals(overrideConfiguration)
        mRequestedOverrideConfiguration.setTo(overrideConfiguration)


```

2）、自从 ActivityClientController.toggleFreeformWindowingMode 方法中我们调用：

```
                    rootTask.setWindowingMode(WINDOWING_MODE_FREEFORM)


```

后，至此 WINDOWING_MODE_FREEFORM 似乎只是保存到了 mRequestedOverrideConfiguration 中了，似乎并没有生效，那什么时候生效呢，答案在 onRequestedOverrideConfigurationChanged 方法中的最后一句：

```
        onConfigurationChanged(parent != null ? parent.getConfiguration() : Configuration.EMPTY);


```

onConfigurationChanged 方法是我们接下来分析的重点。

另外注意，这里 onConfigurationChanged 方法传入的并不是 mRequestedOverrideConfiguration，而是 parent.getConfiguration，即父容器的 mFullConfiguration。

这里可能会有疑问，为啥在 ConfigurationContainer.setWindowingMode 中，还要先用一个临时变量 mRequestsTmpConfig 来存放这个请求的 WindowingMode 呢，直接把这个 WindowingMode 设置到 mRequestedOverrideConfiguration 里不好吗？这个是因为，有的 ConfigurationContainer 的子类可能会重写 onRequestedOverrideConfigurationChanged 方法，如 DisplayContent：

```
    @Override
    public void onRequestedOverrideConfigurationChanged(Configuration overrideConfiguration) {
        final Configuration currOverrideConfig = getRequestedOverrideConfiguration()
        final int currRotation = currOverrideConfig.windowConfiguration.getRotation()
        final int overrideRotation = overrideConfiguration.windowConfiguration.getRotation()
        if (currRotation != ROTATION_UNDEFINED && currRotation != overrideRotation) {
            applyRotationAndFinishFixedRotation(currRotation, overrideRotation)
        }
        mCurrentOverrideConfigurationChanges = currOverrideConfig.diff(overrideConfiguration)
        super.onRequestedOverrideConfigurationChanged(overrideConfiguration)
        mCurrentOverrideConfigurationChanges = 0
        mWmService.setNewDisplayOverrideConfiguration(currOverrideConfig, this)
        mAtmService.addWindowLayoutReasons(
                ActivityTaskManagerService.LAYOUT_REASON_CONFIG_CHANGED)
    }


```

在这些重写的 onRequestedOverrideConfigurationChanged 方法中，有些子类可能希望在请求的配置生效之前，获取到请求前的配置信息，来和请求后的配置信息进行对比，如这里的：

```
        final Configuration currOverrideConfig = getRequestedOverrideConfiguration()
        final int currRotation = currOverrideConfig.windowConfiguration.getRotation()
        final int overrideRotation = overrideConfiguration.windowConfiguration.getRotation()
        if (currRotation != ROTATION_UNDEFINED && currRotation != overrideRotation) {
            applyRotationAndFinishFixedRotation(currRotation, overrideRotation)
        }


```

因此 ConfigurationContainer 的各个子类去重写 onRequestedOverrideConfigurationChanged 方法是允许的，但是别忘了调用：

```
super.onRequestedOverrideConfigurationChanged(overrideConfiguration);


```

确保本次请求的配置信息能够保存到 mRequestedOverrideConfiguration 中，以及 ConfigurationContainer.onConfigurationChanged 方法接下来能够被调用。

在分析 ConfigurationContainer.onConfigurationChanged 方法之前，小结一下 mRequestedOverrideConfiguration 和 onRequestedOverrideConfigurationChanged 的作用。

除了上面所举例子中调用的 ConfigurationContainer.setWindowingMode，其他的对 WindowConfiguration 的更改，如 setActivityType：

```
    
    public void setActivityType( int activityType) {
        int currentActivityType = getActivityType();
        if (currentActivityType == activityType) {
            return;
        }
        if (currentActivityType != ACTIVITY_TYPE_UNDEFINED) {
            throw new IllegalStateException("Can't change activity type once set: " + this
                    + " activityType=" + activityTypeToString(activityType));
        }
        mRequestsTmpConfig.setTo(getRequestedOverrideConfiguration());
        mRequestsTmpConfig.windowConfiguration.setActivityType(activityType);
        onRequestedOverrideConfigurationChanged(mRequestsTmpConfig);
    }


```

以及 setBounds：

```
    public int setBounds(Rect bounds) {
        int boundsChange = diffRequestedOverrideBounds(bounds)
        final boolean overrideMaxBounds = providesMaxBounds()
                && diffRequestedOverrideMaxBounds(bounds) != BOUNDS_CHANGE_NONE
​
        if (boundsChange == BOUNDS_CHANGE_NONE && !overrideMaxBounds) {
            return boundsChange
        }
​
        mRequestsTmpConfig.setTo(getRequestedOverrideConfiguration())
        mRequestsTmpConfig.windowConfiguration.setBounds(bounds)
        onRequestedOverrideConfigurationChanged(mRequestsTmpConfig)
​
        return boundsChange
    }


```

以及各类 setXXX 方法：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4c5d256213bd4094880153005594ce5e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

可以看出，ConfigurationContainer 下的这类 setXXX 方法的逻辑都是类似的：

1）、先将请求的信息保存在了临时变量 mRequestsTmpConfig 中。

2）、接着调用 onRequestedOverrideConfigurationChanged 方法，传入 mRequestsTmpConfig。

所以这类 setXXX 方法的关键在于 onRequestedOverrideConfigurationChanged 方法的调用。

在 onRequestedOverrideConfigurationChanged 方法中，重要的两个步骤则是：

1）、将请求的配置信息保存到 mRequestedOverrideConfiguration 中。

2）、调用 onConfigurationChanged 方法。

那么 mRequestedOverrideConfiguration 成员变量的作用，则是保存当前 container 主动为自己请求的一些特有的信息，目的是防止这部分的信息后续不会随着 Configuration 的更新而丢失。

而 onRequestedOverrideConfigurationChanged 方法虽然支持重写，但是最好在重写方法中调用：

```
super.onRequestedOverrideConfigurationChanged(overrideConfiguration);


```

确保本次请求的配置信息能够保存到 mRequestedOverrideConfiguration 中，以及 ConfigurationContainer.onConfigurationChanged 方法接下来能够被调用。

4.3 mFullConfiguration 成员变量和 onConfigurationChanged 成员方法
--------------------------------------------------------

```
    
     * Notify that parent config changed and we need to update full configuration.
     * @see #mFullConfiguration
     */
    public void onConfigurationChanged(Configuration newParentConfig) {
        mResolvedTmpConfig.setTo(mResolvedOverrideConfiguration);
        resolveOverrideConfiguration(newParentConfig);
        mFullConfiguration.setTo(newParentConfig);
        mFullConfiguration.updateFrom(mResolvedOverrideConfiguration);
        onMergedOverrideConfigurationChanged();
        if (!mResolvedTmpConfig.equals(mResolvedOverrideConfiguration)) {
            
            
            
            
            
            
            for (int i = mChangeListeners.size() - 1; i >= 0; --i) {
                mChangeListeners.get(i).onRequestedOverrideConfigurationChanged(
                        mResolvedOverrideConfiguration);
            }
        }
        for (int i = mChangeListeners.size() - 1; i >= 0; --i) {
            mChangeListeners.get(i).onMergedOverrideConfigurationChanged(
                    mMergedOverrideConfiguration);
        }
        for (int i = getChildCount() - 1; i >= 0; --i) {
            dispatchConfigurationToChild(getChildAt(i), mFullConfiguration);
        }
    }


```

1）、首先将 mResolvedOverrideConfiguration 的值赋值给 mResolvedTmpConfig：

```
        mResolvedTmpConfig.setTo(mResolvedOverrideConfiguration)


```

2）、接着重新计算 mResolvedOverrideConfiguration 的值：

```
        resolveOverrideConfiguration(newParentConfig);


```

看下 ConfigurationContainer.resolveOverrideConfiguration 方法：

```
    
     * Resolves the current requested override configuration into
     * {@link #mResolvedOverrideConfiguration}
     *
     * @param newParentConfig The new parent configuration to resolve overrides against.
     */
    void resolveOverrideConfiguration(Configuration newParentConfig) {
        mResolvedOverrideConfiguration.setTo(mRequestedOverrideConfiguration);
    }


```

很简单，将 mRequestedOverrideConfiguration 的值赋值给 mResolvedOverrideConfiguration，这就是为什么 mRequestedOverrideConfiguration 的值可以影响最终的 mFullConfiguration。

3）、将父容器的 mFullConfiguration 赋值给当前容器的 mFullConfiguration：

```
        mFullConfiguration.setTo(newParentConfig)


```

在之前的分析中我们也知道了，这里的 newParentConfig，即 parent.getConfiguration，也就是父容器的 mFullConfiguration。

这一步很重要，这也说明了并不需要在每个 container 的 onConfigurationChanged 方法中，都重新计算一次新的 Configuraiton 中的每个成员变量，大部分时候，子 container 都是直接继承父 container 的 Configuration。这也很好理解，比如这里我设置了某个 Task 的 WindowingMode 为 WINDOWING_MODE_FREEFORM，那么就没有必要再为这个 Task 中的每一个 ActivityRecord，都显式地进行一次如下操作：

```
activityRecord.getConfiguration().setWindowingMode(WINDOWING_MODE_FREEFORM);


```

4）、用 mResolvedOverrideConfiguration 对将当前 container 的 mFullConfiguration 进行更新：

```
        mFullConfiguration.updateFrom(mResolvedOverrideConfiguration)


```

至此，这次对 Task 的 WindowingMode 的设置才算生效，从 ConfigurationContainer.onConfigurationChanged 的代码执行顺序我们也能看出，mFullConfiguration 才是能够代表当前 container Configuration 的那个。

这一点从 ConfigurationContainer 的各种返回 WindowConfiguration 信息的方法也能看出来，比如我们调用 Task.getActivityType，那么实际调用的是 ConfigurationContainer.getActivityType：

```
    
    
    public int getActivityType() {
        return mFullConfiguration.windowConfiguration.getActivityType();
    }


```

返回的则是 mFullConfiguration 的 ActivityType，而不是 mRequestedOverrideConfiguration 或者 mResolvedOverrideConfiguration 的。

4.4 mResolvedOverrideConfiguration 成员变量和 resolveOverrideConfiguration 成员方法
--------------------------------------------------------------------------

上面的分析中并没有体现 mResolvedOverrideConfiguration 的作用，并且 ConfigurationContainer.resolveOverrideConfiguration 方法的内容也很奇怪：

```
    
     * Resolves the current requested override configuration into
     * {@link #mResolvedOverrideConfiguration}
     *
     * @param newParentConfig The new parent configuration to resolve overrides against.
     */
    void resolveOverrideConfiguration(Configuration newParentConfig) {
        mResolvedOverrideConfiguration.setTo(mRequestedOverrideConfiguration);
    }


```

只是将 mResolvedOverrideConfiguration 的值赋值为 mRequestedOverrideConfiguration，传入的 newParentConfig 却压根没有用到。

其实这个方法，比起 onRequestedOverrideConfigurationChanged 方法，更是需要被 ConfigurationContainer 的子类重写，才能发挥 mResolvedOverrideConfiguration 的作用，目前重写该方法的子类有：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb87d5a40e794ba790df9e3b37c2c4c7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

拿其中一个看一下，Task 的 resolveOverrideConfiguration 方法：

```
    @Override
    void resolveOverrideConfiguration(Configuration newParentConfig) {
        mTmpBounds.set(getResolvedOverrideConfiguration().windowConfiguration.getBounds());
        super.resolveOverrideConfiguration(newParentConfig);
​
        int windowingMode =
                getResolvedOverrideConfiguration().windowConfiguration.getWindowingMode();
        final int parentWindowingMode = newParentConfig.windowConfiguration.getWindowingMode();
​
        
        
        if (getActivityType() == ACTIVITY_TYPE_HOME && windowingMode == WINDOWING_MODE_UNDEFINED) {
            windowingMode = WindowConfiguration.isSplitScreenWindowingMode(parentWindowingMode)
                    ? parentWindowingMode : WINDOWING_MODE_FULLSCREEN;
            getResolvedOverrideConfiguration().windowConfiguration.setWindowingMode(windowingMode);
        }
​
        
        
        if (!supportsMultiWindow()) {
            final int candidateWindowingMode =
                    windowingMode != WINDOWING_MODE_UNDEFINED ? windowingMode : parentWindowingMode;
            if (WindowConfiguration.inMultiWindowMode(candidateWindowingMode)
                    && candidateWindowingMode != WINDOWING_MODE_PINNED) {
                getResolvedOverrideConfiguration().windowConfiguration.setWindowingMode(
                        WINDOWING_MODE_FULLSCREEN);
            }
        }
​
        if (isLeafTask()) {
            resolveLeafOnlyOverrideConfigs(newParentConfig, mTmpBounds /* previousBounds */);
        }
        computeConfigResourceOverrides(getResolvedOverrideConfiguration(), newParentConfig);
    }


```

1）、首先调用父类的 resolveOverrideConfiguration，将 mRequestedOverrideConfiguration 的值赋值给 mResolvedOverrideConfiguration：

```
        super.resolveOverrideConfiguration(newParentConfig);


```

mRequestedOverrideConfiguration 保存的是当前 container 主动请求的特有信息，我们要在此基础上进行策略约束的应用。

2）、这里算是强制规定 HOME 类型的 Task 的 WindowingMode 只能为分屏类型或者全屏类型：

```
        
        
        if (getActivityType() == ACTIVITY_TYPE_HOME && windowingMode == WINDOWING_MODE_UNDEFINED) {
            windowingMode = WindowConfiguration.isSplitScreenWindowingMode(parentWindowingMode)
                    ? parentWindowingMode : WINDOWING_MODE_FULLSCREEN;
            getResolvedOverrideConfiguration().windowConfiguration.setWindowingMode(windowingMode);
        }


```

这一条便是针对 HOME 类型的 Task 进行的策略约束，确保 HOME 类型的 Task 不被设置为异常的 WindowingMode。

3）、如果 Task 不支持多窗口模式，那么即使强制将这个 Task 的 WindowingMode 设置为多窗口模式，也不会生效（除了 PIP 模式）：

```
        
        
        if (!supportsMultiWindow()) {
            final int candidateWindowingMode =
                    windowingMode != WINDOWING_MODE_UNDEFINED ? windowingMode : parentWindowingMode;
            if (WindowConfiguration.inMultiWindowMode(candidateWindowingMode)
                    && candidateWindowingMode != WINDOWING_MODE_PINNED) {
                getResolvedOverrideConfiguration().windowConfiguration.setWindowingMode(
                        WINDOWING_MODE_FULLSCREEN);
            }
        }


```

这一条便是针对不支持多窗口的 Task 进行的策略约束，这样即使某种情况下错误地对不支持多窗口进行了多窗口 WindowingMode 的设置，这里也能确保该 Task 不会进入到多窗口模式。

4）、计算 Task 的 Configuration 的部分属性：

```
        computeConfigResourceOverrides(getResolvedOverrideConfiguration(), newParentConfig);


```

这个方法很复杂，我们挑其中一处来看下：

```
        if (inOutConfig.orientation == ORIENTATION_UNDEFINED) {
            inOutConfig.orientation = (inOutConfig.screenWidthDp <= inOutConfig.screenHeightDp)
                    ? ORIENTATION_PORTRAIT : ORIENTATION_LANDSCAPE
        }


```

很简单，根据 screenWidthDp 和 screenHeightDp 关系，判断 inOutConfig 的方向是横屏还是竖屏。

为什么要重新计算一下 orientation 属性，直接继承父容器的 orientation 不行吗？大部分时候这样做是没问题的，但是如果考虑到多窗口的情况，如分屏，那么情况就可能稍微不同。

假设此时手机的分辨率为 900 * 1600，那么对于当前手机屏幕，或者默认屏幕 DisplayContent 来说，它的 Configuration 的 orientation 应该是竖屏，ORIENTATION_PORTRAIT，全屏应用对应的 Task，其 bounds 也是 900 * 1600，orientation 自然也是 ORIENTATION_PORTRAIT。

但如果是上下等分的分屏 Task，那么其 bounds 就会变为 900 * 800，此时宽大于高，那么其 orientation 就会变成 ORIENTATION_LANDSCAPE，也就和全局的 orientation 不一致了，而实际上也应当如此。这个计算过程也可以体现出 resolveOverrideConfiguration 的作用，即根据当前 container 的具体情况，修正 Configuration 的数据信息，从而体现出每一个 container 的 Configuration 的差异性。

5）、总结来说，resolveOverrideConfiguration 方法的作用是，对某一个 WindowContainer 的子类，如 Task，对该子类的实例进行一些策略约束，策略约束的目的则是确保其实例的 Configuration 能够按照一定的规则和策略反映出当前实例的属性。而这个策略约束的结果，则是保存在了 mResolvedOverrideConfiguration 中。

4.5 将当前容器的 Configuration 改变应用到其所有子容器
------------------------------------

看到 ConfigurationContainer 的 onConfigurationChanged 方法最后还有一点内容：

```
    public void onConfigurationChanged(Configuration newParentConfig) {
        
        for (int i = getChildCount() - 1; i >= 0; --i) {
            dispatchConfigurationToChild(getChildAt(i), mFullConfiguration);
        }
    }


```

首先 getChildAt 方法是被 WindowContainer 重写：

```
    @Override
    protected E getChildAt(int index) {
        return mChildren.get(index);
    }


```

遍历所有子容器。

dispatchConfigurationToChild 方法为：

```
    
     * Dispatches the configuration to child when {@link #onConfigurationChanged(Configuration)} is
     * called. This allows the derived classes to override how to dispatch the configuration.
     */
    void dispatchConfigurationToChild(E child, Configuration config) {
        child.onConfigurationChanged(config);
    }


```

即对所有的子容器，都调用其 onConfigurationChanged 方法，传入的 config，则是父 container 的 mFullConfiguration。

如果子容器没有主动申请一个不为 WINDOWING_MODE_UNDEFINED 的 WindowingMode，那么子容器的 mFullConfiguration 的 WindowingMode 将会被父容器的 mFullConfiguration 的 WindowingMode 覆盖，或者说子容器的 WindowingMode 将会继承父容器的 WindowingMode。

最终我们本次为 Task 请求的 WindowingMode，WINDOWING_MODE_FREEFORM，将会应用到该 Task 的所有子 container 上：

```
        #6 Task=46 type=standard mode=freeform override-mode=freeform requested-bounds=[0,0][1080,2418] bounds=[0,102][1080,2520]
         #0 ActivityRecord{cfe44 u0 com.google.android.apps.messaging/.ui.ConversationListActivity t46} type=standard mode=freeform override-mode=undefined requested-bounds=[0,0][0,0] boun
ds=[0,102][1080,2520]
          #0 13bc593 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity type=standard mode=freeform override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,102][1080,2520]


```

并且只有 Task#46 的 mRequestedOverrideConfiguration 的 WindowingMode 为 freeform，其子 container 的 mRequestedOverrideConfiguration 的 WindowingMode 仍然为 undefined。

最后，别忘了该方法也和 resolveOverrideConfiguration 方法一样，支持其子类进行重写：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e94e4ca606d64e85a71078210f990e97~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

从而使得这些子类能够在 Configuration 发生改变的时候，更加及时地做出响应。

4.6 小结
------

根据以上分析，我们可以得出 onConfigurationChanged 回调触发的两种情况：

*   当前 container 请求了某些属性的改变，希望立即生效，这是通过调用 onRequestedOverrideConfigurationChanged 方法进而调用 onConfigurationChanged 方法的方式实现的。
*   当前 container 的父 container 的 Configuration 发生了改变，并且希望能够立即广播给其所有的子 container，这是通过调用父 container 的 onConfigurationChanged 方法进而调用子 container 的 onConfigurationChanged 方法来实现的。

另外，我们可以总结出 Configuration 更新的一般流程，即 onConfigurationChanged 方法的主要内容。

1）、首先 mRequestedOverrideConfiguration 中保存的是当前 container 自己请求的一部分特有信息，一般是通过 ConfigurationContainer 提供的各种 setXXX 方法进行请求地，这部分信息不会被父容器的 Configuration 所覆盖。

2）、onConfigurationChanged 方法中，首先调用当前 container 的 resolveOverrideConfiguration 方法，该方法将 mRequestedOverrideConfiguration 赋值给 mResolvedOverrideConfiguration，然后基于每个 container 类独有的策略规则，为 mResolvedOverrideConfiguration 施加约束和进行修正。

3）、将 mFullConfiguration 赋值为传入的 newParentConfig 的值。

4）、在 newParentConfig 的基础上，mFullConfiguration 根据第二步得到的 mResolvedOverrideConfiguration 的值进行更新，得到最终的 mFullConfiguration。

5）、得到最终的 mFullConfiguration 后，调用当前 container 的所有子 container 的 onConfigurationChanged 方法，使用该 mFullConfiguration 来更新所有子 container 的 mFullConfiguration。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5499a3cdfa424048b60cbc88e3f871ae~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

现在我们知道了，在父容器的 onConfigurationChanged 方法中，父容器在更新了自己的 mFullConfiguration 后，会将其 mFullConfiguration 作为传参，继续调用子容器的 onConfigurationChanged 方法，进而形成递归调用。

5.1 Configuration 更新的顺序
-----------------------

第一个问题是，哪个 container 的 onConfigurationChanged 方法在某一次 Configuration 更新的时候被第一个调用？

之前有简单提到过，ConfigurationContainer 作为 WindowContainer 的父类，并没有参与到层级结构的创建之中，和层级结构相关的方法也都是抽象的：

```
    abstract protected int getChildCount();
​
    abstract protected E getChildAt(int index);
​
    abstract protected ConfigurationContainer getParent();


```

这些抽象方法的具体实现则是在 WindowContainer 中，因此 Configuration 传递所依托的层级结构，就是 WindowContainer 建立起的层级结构，之前在分析 WindowContainer 类的时候已经分析过了，直接拿当时的一张图：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1b01a88f40948a28bb1eb0859dd1458~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这个图并不反映系统中真实的 container 层级结构，但是可以作为参考，具体某一时刻的层级结构信息则可以通过执行 dumpsys activity containers 命令查看。

该图，以 RootWindowContainer 为根节点，各个叶子节点则是代表每一个窗口的 WindowState 对象，越靠左则在层级结构中的层级越高。

另外查看 ConfigurationContainer 的 onConfigurationChanged 方法调用子容器的顺序：

```
        for (int i = getChildCount() - 1; i >= 0; --i) {
            dispatchConfigurationToChild(getChildAt(i), mFullConfiguration);
        }


```

是从高到低，那么拿上面的图为例：

1）、第一个触发 onConfigurationChanged 方法的是根节点，RootWindowContainer。

2）、此后按照从左至右，从上至下的顺序，该层级结构中的每一个节点对应的 container 依次回调 onConfigurationChanged 方法，最终所有的 container 的 onConfigurationChanged 方法都能得到执行。

而事实上也的确如此，以一次转屏的动作为例，log 的信息的确和上面分析一致，各个 container 的 onConfigurationChanged 方法调用的顺序也就是执行 dumpsys activity containers 命令后，打印的各个 container 的顺序，log 比较多，这里就不贴了。

5.2 Configuration 更新的初值
-----------------------

第二个问题是，在某一次 Configuration 的更新中，Configuration 的初值是如何得到的？

之前分析 onConfigurationChanged 方法的时候，知道该方法的传参是父容器的 mFullConfiguration：

```
public void onConfigurationChanged(Configuration newParentConfig)


```

但是对于没有父容器的容器，即层级结构中的根节点，RootWindowContainer，它的 onConfigurationChanged 方法传入的 Configuration 是如何得到的？

还是以转屏为例，看下具体的方法调用堆栈：

```
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.server.wm.ConfigurationContainer.onConfigurationChanged(ConfigurationContainer.java:153)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.server.wm.WindowContainer.onConfigurationChanged(WindowContainer.java:361)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.server.wm.ActivityTaskManagerService.updateGlobalConfigurationLocked(ActivityTaskManagerService.java:4462)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.server.wm.DisplayContent.updateDisplayOverrideConfigurationLocked(DisplayContent.java:5674)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.server.wm.DisplayContent.updateDisplayOverrideConfigurationLocked(DisplayContent.java:5651)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.server.wm.DisplayContent.sendNewConfiguration(DisplayContent.java:1437)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.server.wm.DisplayRotation.continueRotation(DisplayRotation.java:697)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.server.wm.DisplayRotation.access$200(DisplayRotation.java:92)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.server.wm.DisplayRotation$2.lambda$continueRotateDisplay$0(DisplayRotation.java:242)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.server.wm.DisplayRotation$2$$ExternalSyntheticLambda0.accept(Unknown Source:10)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.internal.util.function.pooled.PooledLambdaImpl.doInvoke(PooledLambdaImpl.java:295)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.internal.util.function.pooled.PooledLambdaImpl.invoke(PooledLambdaImpl.java:204)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.internal.util.function.pooled.OmniFunction.run(OmniFunction.java:97)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at android.os.Handler.handleCallback(Handler.java:938)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at android.os.Handler.dispatchMessage(Handler.java:99)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at android.os.Looper.loopOnce(Looper.java:231)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at android.os.Looper.loop(Looper.java:338)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at android.os.HandlerThread.run(HandlerThread.java:67)
12-21 21:54:05.106  1521  1608 W ukynho_config:         at com.android.server.ServiceThread.run(ServiceThread.java:44)


```

关键的调用步骤为：

1）、DisplayContent.updateDisplayOverrideConfigurationLocked 方法：

```
    boolean updateDisplayOverrideConfigurationLocked() {
        
​
        Configuration values = new Configuration();
        computeScreenConfiguration(values);
​
        mAtmService.mH.sendMessage(PooledLambda.obtainMessage(
                ActivityManagerInternal::updateOomLevelsForDisplay, mAtmService.mAmInternal,
                mDisplayId));
​
        Settings.System.clearConfiguration(values);
        updateDisplayOverrideConfigurationLocked(values, null /* starting */,
                false /* deferResume */, mAtmService.mTmpUpdateConfigurationResult);
        return mAtmService.mTmpUpdateConfigurationResult.changes != 0;
    }


```

收到发送新的 Configuration 的请求后，首先调用 computeScreenConfiguration 方法，计算出一个新的 Configuration。computeScreenConfiguration 方法的具体内容则是，基于 DisplayContent 的 DisplayInfo 和其他的设备信息，计算出一个新的 Configuration 对象。

接着将计算好的 Configuration 发送给 updateDisplayOverrideConfigurationLocked 方法。

2）、DisplayContent.updateDisplayOverrideConfigurationLocked 方法：

```
     * Updates override configuration specific for the selected display. If no config is provided,
     * new one will be computed in WM based on current display info.
     */
    boolean updateDisplayOverrideConfigurationLocked(Configuration values,
            ActivityRecord starting, boolean deferResume,
            ActivityTaskManagerService.UpdateConfigurationResult result) {
​
        int changes = 0
        boolean kept = true
​
        mAtmService.deferWindowLayout()
        try {
            if (values != null) {
                if (mDisplayId == DEFAULT_DISPLAY) {
                    // Override configuration of the default display duplicates global config, so
                    // we're calling global config update instead for default display. It will also
                    // apply the correct override config.
                    changes = mAtmService.updateGlobalConfigurationLocked(values,
                            false /* initLocale */, false /* persistent */,
                            UserHandle.USER_NULL /* userId */)
                } else {
                    changes = performDisplayOverrideConfigUpdate(values)
                }
            }
​
            if (!deferResume) {
                kept = mAtmService.ensureConfigAndVisibilityAfterUpdate(starting, changes)
            }
        } finally {
            mAtmService.continueWindowLayout()
        }
​
        if (result != null) {
            result.changes = changes
            result.activityRelaunched = !kept
        }
        return kept
    }


```

这个方法很简单，继续将计算好的 Configuration 对象继续发送给 ActivityTaskManagerService。

3）、ActivityTaskManagerService.updateGlobalConfigurationLocked 方法：

```
    
    int updateGlobalConfigurationLocked(@NonNull Configuration values, boolean initLocale,
            boolean persistent, int userId) {
​
        mTempConfig.setTo(getGlobalConfiguration());
        final int changes = mTempConfig.updateFrom(values);
        
        
​
        
        mRootWindowContainer.onConfigurationChanged(mTempConfig);
​
        return changes;
    }


```

过滤不相关的信息，该方法首先通过 getGlobalConfiguration 方法取得全局 Configuration：

```
    
     * Current global configuration information. Contains general settings for the entire system,
     * also corresponds to the merged configuration of the default display.
     */
    Configuration getGlobalConfiguration() {
        
        
        
        return mRootWindowContainer != null ? mRootWindowContainer.getConfiguration()
                : new Configuration();
    }


```

也就是 RootWindowContainer 的 mFullConfiguration，然后用传入的 Configuration 对其进行更新，最后调用 RootWindowContainer 的 onConfigurationChanged 方法。

那么本次 Configuration 更新的初值，则是基于上一次的全局 Configuration，更新上通过 DisplayContent.computeScreenConfiguration 方法计算的 Configuration 得到的新的全局 Configuration。

再进一步看到，RootWindowContainer 是没有重写 resolveOverrideConfiguration 方法的，也没有主动去请求某些特殊属性，也就是说，RootWindowContainer 的 mRequestedOverrideConfiguration 和 mResolvedOverrideConfiguration 全都是 Configuration.EMPTY，那么按照 mFullConfiguration 的计算规则，当 mResolvedOverrideConfiguration 为 Configuration.EMPTY 的时候，mFullConfiguration 的值就和 onConfigurationChanged 方法的传参 newParentConfig 的值相等，即 RootWindowContainer 的 mFullConfiguration 的值完全由传参 newParentConfig 决定。

而传参 newParentConfig 的值，则是在 RootWindowContainer 的 mFullConfiguration 的基础上，加上 DisplayContent.computeScreenConfiguration 计算出的 Configuration 得到的。

那么某一次 Configuration 更新，它的初值，即 RootWindowContainer 的 onConfigurationChanged 方法的传参 newParentConfig，它的值，就是 RootWindowContainer 的上一次 mFullConfiguration，加上 DisplayContent.computeScreenConfiguration 的结果得到的，也是本次 RootWindowContainer 的 mFullConfiguration 的最终值。

因为 RootWindowContainer 的 mFullConfiguration 也叫做 global config，那么本次 Configuration 更新的初值，也就是 global config，是通过：

```
globalConfig += DisplayContent.computeScreenConfiguration


```

得到的。

5.3 小结
------

通过本节可以得到的信息是：

1）、容器的 Configuration 更新顺序以容器通过 WindowContainer 组织起来的层级结构为基础，按照从高层级到低层级的顺序进行。

2）、RootWindowContainer 的 mFullConfiguration 是作为全局 Configuration 的角色存在的。

3）、在一次 Configuration 更新时，首先根据当前的 DisplayInfo 和其他设备信息计算出一个新的 Configuration，然后用这个新的 Configuration 去更新全局 Configuration，得到能够反映当前系统实时情况的新的全局 Configuration。

4）、各个容器的 Configuration 更新，便是以全局 Configuration（或者是父容器的 mFullConfiguration）为基础，再加上自己请求的部分（mRequestedOverrideConfiguration），以及策略约束部分（mResolvedOverrideConfiguration），从而得到一个新的 mFullConfiguration，即完成了 Configuration 的更新。

这个成员变量平时接触的很少，这里试着分析一下，可能有分析不对的地方，因此本节内容请慎重参考。

6.1 定义
------

首先是 mMergedOverrideConfiguration 的定义：

```
    
     * Contains merged override configuration settings from the top of the hierarchy down to this
     * particular instance. It is different from {@link #mFullConfiguration} because it starts from
     * topmost container's override config instead of global config.
     */
    private Configuration mMergedOverrideConfiguration = new Configuration();


```

包含了从 WindowContainer 层级结构顶层传到此 container 的合并的 override configuration。它和 mFullConfiguration 不同的地方在于，它起始于层级结构中最顶层的那个 container 的 override configuration，而不是全局 configuration。

根据之前的分析，知道了全局 configuration，就是层级结构中最顶层的那个 container，RootWindowContainer 的 mFullConfiguration，那这里的”override config“如何理解？下面分析会讲到。

6.2 更新机制
--------

继续看下 mMergedOverrideConfiguration 是如何更新的：

```
    public void onConfigurationChanged(Configuration newParentConfig) {
        mResolvedTmpConfig.setTo(mResolvedOverrideConfiguration);
        resolveOverrideConfiguration(newParentConfig);
        mFullConfiguration.setTo(newParentConfig);
        mFullConfiguration.updateFrom(mResolvedOverrideConfiguration);
        onMergedOverrideConfigurationChanged();
        
    }


```

同样是在 onConfigurationChanged 方法中，mFullConfiguration 更新完之后就调用了 onMergedOverrideConfigurationChanged：

```
    
     * Update merged override configuration based on corresponding parent's config and notify all
     * its children. If there is no parent, merged override configuration will set equal to current
     * override config.
     * @see #mMergedOverrideConfiguration
     */
    void onMergedOverrideConfigurationChanged() {
        final ConfigurationContainer parent = getParent();
        if (parent != null) {
            mMergedOverrideConfiguration.setTo(parent.getMergedOverrideConfiguration());
            mMergedOverrideConfiguration.updateFrom(mResolvedOverrideConfiguration);
        } else {
            mMergedOverrideConfiguration.setTo(mResolvedOverrideConfiguration);
        }
        for (int i = getChildCount() - 1; i >= 0; --i) {
            final ConfigurationContainer child = getChildAt(i);
            child.onMergedOverrideConfigurationChanged();
        }
    }


```

这里看到，如果 parent 为 null，那么直接将 mResolvedOverrideConfiguration 的值作为 mMergedOverrideConfiguration 的值，再根据注释的解释，当没有 parent 的时候，merged override configuration 将等于 current override config，这就可以解答 6.1 最后提出的问题，”override config“其实就是 mResolvedOverrideConfiguration。

那么 mMergedOverrideConfiguration 定义处的注释意思就是，mFullConfiguration 起始于 global config，而 mMergedOverrideConfiguration 起始于层级结构根节点的 mResolvedOverrideConfiguration。进一步说明就是，mFullConfiguration 起始于 RootWindowContainer 的 mFullConfiguration，而 mMergedOverrideConfiguration 起始于 RootWindowContainer 的 mResolvedOverrideConfiguration。

对比一下 mFullConfiguration 和 mMergedOverrideConfiguration 的更新有何区别。

1）、mFullConfiguration，以父容器的 mFullConfiguration 为 base，updateFrom 上 mResolvedOverrideConfiguration。当父容器为空，即当前容器为 RootWindowContainer 时，当前容器的 mFullConfiguration 以自己为 base，updateFrom 上 DisplayContent.computeScreenConfiguration 方法计算出的 Configuration。

2）、mMergedOverrideConfiguration，以父容器的 mMergedOverrideConfiguration 为 base，updateFrom 上 mResolvedOverrideConfiguration。当父容器为空，即当前容器为 RootWindowContainer 时，此时的 mMergedOverrideConfiguration 被直接设置为 mResolvedOverrideConfiguration 的值，根据 onMergedOverrideConfigurationChanged 方法注释中的这段话：

”If there is no parent, merged override configuration will set equal to current override config.“

说明 mResolvedOverrideConfiguration，代表的就是注释中提到的”override config“，这也回答了 6.1 节最后的那个疑问。

mResolvedOverrideConfiguration，代表的是 container 自己的覆盖配置，这个覆盖配置是当前 container 的 mFullConfiguration 区别于父 container 的 mFullConfiguration 的部分，那对于某个 container 来说，它的 mMergedOverrideConfiguration 收集的信息就是从 RootWindowContainer 到当前 container 这条路径上的所有 container 的 mResolvedOverrideConfiguration 之和，或者说 override config 之和，即 mMergedOverrideConfiguration 的定义处的注释中的这句话：

”Contains merged override configuration settings from the top of the hierarchy down to this particular instance.“

这里使用比较粗暴的等式来演示一下。

假如当前层级结构只有 4 层：RootWindowContainer、DisplayContent、Task 和 ActivityRecord。

ActivityRecord 的 mMergedOverrideConfiguration 计算方式为：

```
ActivityRecord.mMergedOverrideConfiguration
    = Task.mMergedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration


```

又因为

```
Task.mMergedOverrideConfiguration 
    = DisplayContent.mMergedOverrideConfiguration + Task.mResolvedOverrideConfiguration 


```

所以

```
ActivityRecord.mMergedOverrideConfiguration
    = Task.mMergedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration
    = DisplayContent.mMergedOverrideConfiguration + Task.mResolvedOverrideConfiguration 
        + ActivityRecord.mResolvedOverrideConfiguration


```

依次类推

```
ActivityRecord.mMergedOverrideConfiguration
    = Task.mMergedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration
    = DisplayContent.mMergedOverrideConfiguration + Task.mResolvedOverrideConfiguration 
        + ActivityRecord.mResolvedOverrideConfiguration
    = RootWindowContainer.mMergedOverrideConfiguration + DisplayContent.mResolvedOverrideConfiguration 
        + Task.mResolvedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration


```

RootWindowContainer 的 parent 为 null，所以

```
RootWindowContainer.mMergedOverrideConfiguration = RootWindowContainer.mResolvedOverrideConfiguration


```

那么

```
ActivityRecord.mMergedOverrideConfiguration
    = Task.mMergedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration
    = DisplayContent.mMergedOverrideConfiguration + Task.mResolvedOverrideConfiguration 
        + ActivityRecord.mResolvedOverrideConfiguration
    = RootWindowContainer.mMergedOverrideConfiguration + DisplayContent.mResolvedOverrideConfiguration 
        + Task.mResolvedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration
    = RootWindowContainer.mResolvedOverrideConfiguration + DisplayContent.mResolvedOverrideConfiguration 
        + Task.mResolvedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration


```

之前说过，RootWindowContainer 是没有重写 resolveOverrideConfiguration 方法的，也没有主动去请求某些特殊属性，也就是说，RootWindowContainer 的 mRequestedOverrideConfiguration 和 mResolvedOverrideConfiguration 全都是 Configuration.EMPTY，而 DisplayContent 的 mRequestedOverrideConfiguration 和 mResolvedOverrideConfiguration 并不是 Configuration.EMPTY，所以这里可以忽略 RootWindowContainer.mResolvedOverrideConfiguration：

```
ActivityRecord.mMergedOverrideConfiguration
    = Task.mMergedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration
    = DisplayContent.mMergedOverrideConfiguration + Task.mResolvedOverrideConfiguration 
        + ActivityRecord.mResolvedOverrideConfiguration
    = RootWindowContainer.mMergedOverrideConfiguration + DisplayContent.mResolvedOverrideConfiguration 
        + Task.mResolvedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration
    = RootWindowContainer.mResolvedOverrideConfiguration + DisplayContent.mResolvedOverrideConfiguration 
        + Task.mResolvedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration
    = DisplayContent.mResolvedOverrideConfiguration 
        + Task.mResolvedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration       


```

mResolvedOverrideConfiguration 被视为”override config“，那 mMergedOverrideConfiguration 就是”merged override config“，即所有”override config“合并后的结果：

```
mergedOverrideConfig = overrideConfig1 + overrideConfig2 + ... + overrideConfigN


```

ActivityRecord 的 mFullConfiguration 计算方式为：

```
ActivityRecord.mFullConfiguration 
    = Task.mFullConfiguration + ActivityRecord.mResolvedOverrideConfiguration
    = DisplayContent.mFullConfiguration + Task.mResolvedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration
    = RootWindowContainer.mFullConfiguration + DisplayContent.mResolvedOverrideConfiguration 
        + Task.mResolvedOverrideConfiguration + ActivityRecord.mResolvedOverrideConfiguration
    = RootWindowContainer.mFullConfiguration + ActivityRecord.mMergedOverrideConfiguration


```

而 RootWindowContainer.mFullConfiguration 被视为”global config“，那么：

```
fullConfig = globalConfig + mergedOverrideCOnfig


```

某一个 container 的完整配置，等于”全局配置 “，加上，” 从层级结构根节点到当前节点的所有覆盖配置之和“。

6.3 使用
------

最后再看一下 mMergedOverrideConfiguration 使用的地方，mMergedOverrideConfiguration 定义为 private，那么别的类想要使用它只能通过 ConfigurationContainer 提供的 getMergedOverrideConfiguration 方法来获取：

```
    
     * Get merged override configuration from the top of the hierarchy down to this particular
     * instance. This should be reported to client as override config.
     */
    public Configuration getMergedOverrideConfiguration() {
        return mMergedOverrideConfiguration;
    }


```

注释说 mMergedOverrideConfiguration 应该作为 override config 被报导给客户端，这里的客户端指的应该是 App 进程，那如何理解呢？看一处获取 mMergedOverrideConfiguration 的地方，ActivityRecord.ensureActivityConfiguration：

```
    boolean ensureActivityConfiguration(int globalChanges, boolean preserveWindow,
            boolean ignoreVisibility) {
        
        
        
        
        final int changes = getConfigurationChanges(mTmpConfig);
        
        
        final Configuration newMergedOverrideConfig = getMergedOverrideConfiguration();
​
        setLastReportedConfiguration(getProcessGlobalConfiguration(), newMergedOverrideConfig);
​
        
​
        if (changes == 0 && !forceNewConfig) {
            ProtoLog.v(WM_DEBUG_CONFIGURATION, "Configuration no differences in %s",
                    this);
            
            
            if (displayChanged) {
                scheduleActivityMovedToDisplay(newDisplayId, newMergedOverrideConfig);
            } else {
                scheduleConfigurationChanged(newMergedOverrideConfig);
            }
            return true;
        }
​
        
    }


```

这里调用 getMergedOverrideConfiguration 获取到 mMergedOverrideConfiguration，再将局部变量 newMergedOverrideConfig 指向它，newMergedOverrideConfig 使用的地方有两处。

### 6.3.1 MergedConfiguration 类

第一处是：

```
        setLastReportedConfiguration(getProcessGlobalConfiguration(), newMergedOverrideConfig);


```

这里 getProcessGlobalConfiguration 的定义为：

```
    
    private Configuration getProcessGlobalConfiguration() {
        return app != null ? app.getConfiguration() : mAtmService.getGlobalConfiguration();
    }


```

这里的 app 为 WindowProcessController 对象，WindowProcessController 的 Configuration 更新和 RootWindowContainer 的 Configuration 更新一样，都是在 ActivityTaskManagerService.updateGlobalConfigurationLocked 中被触发了 onConfigurationChanged 方法，且传入的参数也是同一个。那么这里可以不管 app 是否为空，得到的 Configuration 都可以理解为 global config。

setLastReportedConfiguration 方法为：

```
    private void setLastReportedConfiguration(Configuration global, Configuration override) {
        mLastReportedConfiguration.setConfiguration(global, override);
    }


```

将 global config 和 override config 保存在 mLastReportedConfiguration 中，mLastReportedConfiguration 的定义为：

```
    
    private MergedConfiguration mLastReportedConfiguration;


```

上一次报导给 App 进程中的 activity 的 Configuration。

MergedConfiguration 定义为：

```
 * Container that holds global and override config and their merge product.
 * Merged configuration updates automatically whenever global or override configs are updated via
 * setters.
 *
 * {@hide}
 */
public class MergedConfiguration implements Parcelable {
​
    private final Configuration mGlobalConfig = new Configuration();
    private final Configuration mOverrideConfig = new Configuration();
    private final Configuration mMergedConfig = new Configuration();
    
    
}


```

*   mGlobalConfig，用来保存 global config。
    
*   mOverrideConfig，用来保存 override config。
    
*   mMergedConfig 是两者的结合：
    
    ```
        
        private void updateMergedConfig() {
            mMergedConfig.setTo(mGlobalConfig);
            mMergedConfig.updateFrom(mOverrideConfig);
        }
    
    
    ```
    

而根据上面的代码，这里的 mOverrideConfig 实际上就是 ActivityRecord 的 mMergedOverrideConfiguration，那么根据 mMergedConfig 的计算方式，感觉 mMergedConfig 的值应该和 ActivityRecord 的 mFullConfiguration 的值相等。

### 6.3.2 将 mMergedOverrideConfiguration 发送给 App 进程

newMergedOverrideConfig 第二处使用的地方在：

```
        if (changes == 0 && !forceNewConfig) {
            ProtoLog.v(WM_DEBUG_CONFIGURATION, "Configuration no differences in %s",
                    this);
            
            
            if (displayChanged) {
                scheduleActivityMovedToDisplay(newDisplayId, newMergedOverrideConfig);
            } else {
                scheduleConfigurationChanged(newMergedOverrideConfig);
            }
            return true;
        }


```

该方法用来触发 App 进程中 activity 的 Configuration 的更新：

```
    private void scheduleConfigurationChanged(Configuration config) {
        if (!attachedToProcess()) {
            ProtoLog.w(WM_DEBUG_CONFIGURATION, "Can't report activity configuration "
                    + "update - client not running, activityRecord=%s", this);
            return;
        }
        try {
            ProtoLog.v(WM_DEBUG_CONFIGURATION, "Sending new config to %s, "
                    + "config: %s", this, config);
​
            mAtmService.getLifecycleManager().scheduleTransaction(app.getThread(), appToken,
                    ActivityConfigurationChangeItem.obtain(config));
        } catch (RemoteException e) {
            
        }
    }


```

具体细节暂不分析，也许后续会再开单独一篇来分析。

这一部分就是关于 getMergedOverrideConfiguration 方法注释中：

”This should be reported to client as override config.“

提到的地方，解释了 6.3.1 节中调用 setLastReportedConfiguration 方法的原因，以及 mLastReportedConfiguration 的作用，即在每次向 App 进程发送新的 Configuration 的时候，进行记录，等到下一次 Configuration 更新的时候，将之前记录的 Configuration 和这次的 Configuration 进行对比，判断是否需要重启 activity，或者是触发 activity 的 onConfigurationChanged 回调等。

不过这里有一个疑问，为什么发送给 App 进程的是 ActivityRecord 的 mMergedOverrideConfiguration，而不是更具代表性的 mFullConfiguration？根据之前的分析，mMergedOverrideConfiguration 代表的是所有覆盖配置之和，即”merged override config“，而 mFullConfiguration 代表的则是一个 container 的完整配置，它的值等于”global config“加上”merged override config“，也就是说，系统不希望把包含”global config“的信息发送给 App 进程，只想发送”override config“相关的信息，为什么这样做呢，暂时没有想到原因。

### 6.3.3 Configuation 的对比

刚刚说了，mLastReportedConfiguration 的作用是记录发送给 App 进程的 Configuration，当下一次 Configuration 更新调用到来后，与这一次的 Configuration 进行对比，来判断是否触发 activity 的重启或者 onConfigurationChanged 回调。这个逻辑也是在 ActivityRecord.ensureActivityConfiguration 方法中，看一下 Configuration 是如何对比的：

```
        
        
        
        
        mTmpConfig.setTo(mLastReportedConfiguration.getMergedConfiguration());
        if (getConfiguration().equals(mTmpConfig) && !forceNewConfig && !displayChanged) {
            ProtoLog.v(WM_DEBUG_CONFIGURATION, "Configuration & display "
                    + "unchanged in %s", this);
            
            
            
            if (mVisibleRequested) {
                updateCompatDisplayInsets();
            }
            return true;
        }


```

注意这里进行比较的 Configuration：

一个是通过 getConfiguration 方法得到的，ActivityRecord 的 mFullConfiguration。

另外一个是 mLastReportedConfiguration 的 getMergedConfiguration 返回的，即 mLastReportedConfiguration 的 mMergedConfig。

而根据我们在 6.3.1 节中看到的 mMergedConfig 的计算方式：

```
    
    private void updateMergedConfig() {
        mMergedConfig.setTo(mGlobalConfig);
        mMergedConfig.updateFrom(mOverrideConfig);
    }


```

MergedConfiguration.mMergedConfig 的值的计算方式和 ActivityRecord.mFullConfiguration 的计算方式本质是相同的，不同点在于 mFullConfiguration 更新的应该更加频繁一点，而 mLastReportedConfiguration 只有在将一个新的 MergedConfiguration 发送给 App 进程的时候才会更新。因此 mFullConfiguration 可以作为实时的 Configuration，当准备向 App 进程发送新的 Configuration 时，拿实时的 Configuration 与上一次发送给 App 进程的 Configuration 进行对比，如果两者不一致再向 App 进程发送新的 Configuration。

而实际上，ConfigurationContainer.getMergedOverrideConfiguration 方法使用的场景也很有限，大部分都是在系统进程与 App 进程通信，需要更新 App 进程中 activity 的 Configuration 的时候。而且有的时候，发送给 App 进程的也不只是 mMergedOverrideConfiguration，而是 MergedConfiguration，如 WindowState.fillClientWindowFramesAndConfiguration：

```
    
     * Fills the given window frames and merged configuration for the client.
     *
     * @param outFrames The frames that will be sent to the client.
     * @param outMergedConfiguration The configuration that will be sent to the client.
     * @param useLatestConfig Whether to use the latest configuration.
     * @param relayoutVisible Whether to consider visibility to use the latest configuration.
     */
    void fillClientWindowFramesAndConfiguration(ClientWindowFrames outFrames,
            MergedConfiguration outMergedConfiguration, boolean useLatestConfig,
            boolean relayoutVisible) {
        
​
        
        
        
        
        if (useLatestConfig || (relayoutVisible && (shouldCheckTokenVisibleRequested()
                || mToken.isVisibleRequested()))) {
            final Configuration globalConfig = getProcessGlobalConfiguration();
            final Configuration overrideConfig = getMergedOverrideConfiguration();
            outMergedConfiguration.setConfiguration(globalConfig, overrideConfig);
            if (outMergedConfiguration != mLastReportedConfiguration) {
                mLastReportedConfiguration.setTo(outMergedConfiguration);
            }
        } else {
            outMergedConfiguration.setTo(mLastReportedConfiguration);
        }
        mLastConfigReportedToClient = true;
    }


```

这里返回给 App 进程的就是”global config“和”merged override config“，比起只返回”merged override config“，信息肯定要更全一点。

6.4 小结
------

这里根据注释中对各个变量的称呼依次介绍。

*   ”global config“，代表的是层级结构根节点的 container 的 mFullConfiguration，即 RootWindowContainer 的 mFullConfiguration，所有 container 的 mFullConfiguration 都基于此进行更新。

*   ”override config“，mResolvedOverrideConfiguration，代表的是当前 container 的覆盖配置，“覆盖” 的含义则是当前 container 的 mFullConfiguration 覆盖父 container 的 mFullConfiguration，覆盖的那一部分，或者说当前 container 的 mFullConfiguration 和父 container 的 mFullConfiguration 差异部分，就是 mResolvedOverrideConfiguration，即 “override config”。

*   ”merged override config“，mMergedOverrideConfiguration，代表的是从 RootWindowContainer 到当前 container 这条路径上的的所有 container 的”override config“（mResolvedOverrideConfiguration）之和。
*   “full config”，mFullConfiguration，当前 container 的完整配置。

那么 mMergedOverrideConfiguration 存在的意义便是：

```
fullConfig = globalConfig + mergedOverrideConfig


```

也就是，如果我们已经有了 “global config”，那么还需要什么信息，就能获取到某一个 activity 的完整配置呢，这个缺少的信息也就是”merged override config“。

而这三者也是 MergedConfiguration 中成员变量的含义：

```
public class MergedConfiguration implements Parcelable {
​
    private final Configuration mGlobalConfig = new Configuration();
    private final Configuration mOverrideConfig = new Configuration();
    private final Configuration mMergedConfig = new Configuration();


```

因为 MergedConfiguration.mMergedConfig 的计算方式就是：

```
    
    private void updateMergedConfig() {
        mMergedConfig.setTo(mGlobalConfig);
        mMergedConfig.updateFrom(mOverrideConfig);
    }


```

另外我们看到，mMergedConfig 的值只能通过 updateMergedConfig 方法计算得到，也就是 MergedConfiguration 没有直接提供对 mMergedConfig 直接赋值的方法，那么我们想要得到 mMergedConfig 的值，就必须传入 mGlobalConfig 和 mOverrideConfig 进行计算得出，而 mOverrideConfig 的值，取的就是 mMergedOverrideConfiguration 的值，除此之外系统中没有第二个值可以用来和 mGlobalConfig 一起计算得到 mMergedConfig 的值，这也就是 mMergedOverrideConfiguration 存在的意义。

最后看下 “global config”，我们之前分析的情况是，“global config” 的值就是 RootWindowContainer 的 mFullConfiguration 的值，这在大部分情况下都是成立的，但是当是 WindowState 的情况时，就不太一样：

```
    @Override
    public Configuration getConfiguration() {
        
        
        if (!registeredForDisplayAreaConfigChanges()) {
            return super.getConfiguration();
        }
​
        
        
        mTempConfiguration.setTo(getProcessGlobalConfiguration());
        mTempConfiguration.updateFrom(getMergedOverrideConfiguration());
        return mTempConfiguration;
    }


```

当 WindowState 附着到一个进程上后，WindowState 对应的 Configuration，不再是当前 WindowState 的 mFullConfiguration。而是需要通过 “global config” 加上 “merged override config” 得到，而 “global config” 是通过 getProcessGlobalConfiguration 方法来取的：

```
    private Configuration getProcessGlobalConfiguration() {
        
        
        final WindowState parentWindow = getParentWindow();
        final int pid = parentWindow != null ? parentWindow.mSession.mPid : mSession.mPid;
        final Configuration processConfig =
                mWmService.mAtmService.getGlobalConfigurationForPid(pid);
        return processConfig;
    }


```

接着调用 ActivityTaskManagerService.getGlobalConfigurationForPid 方法：

```
    
     * Return the global configuration used by the process corresponding to the given pid.
     */
    Configuration getGlobalConfigurationForPid(int pid) {
        if (pid == MY_PID || pid < 0) {
            return getGlobalConfiguration();
        }
        synchronized (mGlobalLock) {
            final WindowProcessController app = mProcessMap.getProcess(pid);
            return app != null ? app.getConfiguration() : getGlobalConfiguration();
        }
    }


```

看到这里取的是当前窗口所在进程对应的 WindowProcessController 的 mFullConfiguration，与进程强相关了，而不再是真正意义的全局 Configuration，即那个全局独一份的 Configuration。

而此时的 “merged override config”，仍然是当前 WindowState 的 mMergedOverrideConfiguration。

这个例子说明了，不同情况下，“global config” 的值取的可能不一样，那么在这个时候，就只能通过

```
fullConfig = globalConfig + mergedOverrideConfig


```

的方式去计算得到 “full config” 了，那么 “merged override config” 存在的意义就显得更加重要了。
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/ukynho/article/details/126748372?spm=1001.2014.3001.5502)

![](https://img-blog.csdnimg.cn/87d582d316b8400988fc2d729f9386a5.jpeg#pic_center)

1 DisplayArea 类的继承关系
--------------------

DisplayArea 类的继承关系，之前已经分析过，这里用一张简单的类图总结：

![](https://img-blog.csdnimg.cn/9e7602e6ec1249b69325a46681cd0595.png#pic_center)

2 DisplayArea 层级结构的生成
---------------------

既然 DisplayContent 是作为其代表的屏幕的 DisplayArea 层级结构的根节点，那么从 DisplayContent 出发，看一下这棵以 DisplayContent 为根节点的 DisplayArea 树是如何生成的。

### 2.1 DisplayContent.init

```
/**
     * Create new {@link DisplayContent} instance, add itself to the root window container and
     * initialize direct children.
     * @param display May not be null.
     * @param root {@link RootWindowContainer}
     */
    DisplayContent(Display display, RootWindowContainer root) {
        // ......

        // Setup the policy and build the display area hierarchy.
        mDisplayAreaPolicy = mWmService.getDisplayAreaPolicyProvider().instantiate(
                mWmService, this /* content */, this /* root */, mImeWindowsContainer);

        // ......
    }
```

在 DisplayContent 的构造方法中，调用 DisplayAreaPolicy.Provider.instantiate 方法，去初始化一个 DisplayArea 层级结构。

### 2.2 DisplayAreaPolicy.DefaultProvider.instantiate

DisplayAreaPolicy.Provider 只是一个接口，instantiate 定义为：

```
/**
         * Instantiates a new {@link DisplayAreaPolicy}. It should set up the {@link DisplayArea}
         * hierarchy.
         *
         * @see DisplayAreaPolicy#DisplayAreaPolicy
         */
        DisplayAreaPolicy instantiate(WindowManagerService wmService, DisplayContent content,
                RootDisplayArea root, DisplayArea.Tokens imeContainer);
```

用来实例化一个 DisplayAreaPolicy 对象，这个对象应该建立起 DisplayArea 层级结构，实际走到的则是 DisplayAreaPolicy.Provider 的实现类 DisplayAreaPolicy.DefaultProvider.instantiate 方法：

```
@Override
        public DisplayAreaPolicy instantiate(WindowManagerService wmService,
                DisplayContent content, RootDisplayArea root,
                DisplayArea.Tokens imeContainer) {
            final TaskDisplayArea defaultTaskDisplayArea = new TaskDisplayArea(content, wmService,
                    "DefaultTaskDisplayArea", FEATURE_DEFAULT_TASK_CONTAINER);
            final List<TaskDisplayArea> tdaList = new ArrayList<>();
            tdaList.add(defaultTaskDisplayArea);

            // Define the features that will be supported under the root of the whole logical
            // display. The policy will build the DisplayArea hierarchy based on this.
            final HierarchyBuilder rootHierarchy = new HierarchyBuilder(root);
            // Set the essential containers (even if the display doesn't support IME).
            rootHierarchy.setImeContainer(imeContainer).setTaskDisplayAreas(tdaList);
            if (content.isTrusted()) {
                // Only trusted display can have system decorations.
                configureTrustedHierarchyBuilder(rootHierarchy, wmService, content);
            }

            // Instantiate the policy with the hierarchy defined above. This will create and attach
            // all the necessary DisplayAreas to the root.
            return new DisplayAreaPolicyBuilder().setRootHierarchy(rootHierarchy).build(wmService);
        }
```

#### 2.2.1 创建 HierarchyBuilder

首先注意到，这里的 RootDisplayArea 类型的传参 root 即是一个 DisplayContent 对象，毕竟它是要作为后续创建的 DisplayArea 层阶结构的根节点。然后根据这个传入的 DisplayContent，创建一个 HierarchyBuilder 对象：

```
// Define the features that will be supported under the root of the whole logical
            // display. The policy will build the DisplayArea hierarchy based on this.
            final HierarchyBuilder rootHierarchy = new HierarchyBuilder(root);
```

HierarchyBuilder 定义在 DisplayAreaPolicyBuilder 中：

```
/**
     *  Builder to define {@link Feature} and {@link DisplayArea} hierarchy under a
     * {@link RootDisplayArea}
     */
    static class HierarchyBuilder {
```

HierarchyBuilder 用来构建一个 DisplayArea 层级结构，该层级结构的根节点，则是 HierarchyBuilder 构造方法中传入的 RootDisplayArea：

```
private final RootDisplayArea mRoot;

	// ......

	HierarchyBuilder(RootDisplayArea root) {
            mRoot = root;
        }
```

这里传入的则是一个 DisplayContent 对象，那么 HierarchyBuilder 要以这个 DisplayContent 对象为根节点，生成一个 DisplayArea 层级结构。

#### 2.2.2 保存 ImeContainer 到 HierarchyBuilder 内部

传参 imeContainer，则是 DisplayContent 的成员变量：

```
// Contains all IME window containers. Note that the z-ordering of the IME windows will depend
// on the IME target. We mainly have this container grouping so we can keep track of all the IME
// window containers together and move them in-sync if/when needed. We use a subclass of
// WindowContainer which is omitted from screen magnification, as the IME is never magnified.
// TODO(display-area): is "no magnification" in the comment still true?
private final ImeContainer mImeWindowsContainer = new ImeContainer(mWmService);
```

即 ImeContainer 类型的 DisplayArea，在定义的时候已经初始化。

通过 HierarchyBuilder.setImeContainer 方法：

```
@Nullable
        private DisplayArea.Tokens mImeContainer;

		// ......

		/** Sets IME container as a child of this hierarchy root. */
        HierarchyBuilder setImeContainer(DisplayArea.Tokens imeContainer) {
            mImeContainer = imeContainer;
            return this;
        }
```

将 DisplayContent 的 mImeWindowsContainer 保存在了 HierarchyBuilder 的 mImeContainer 成员变量中，后续创建 DisplayArea 层级结构时可以直接拿来使用。

#### 2.2.3 创建并保存默认 TaskDisplayArea 到 HierarchyBuilder 内部

创建一个名为 “DefaultTaskDisplayArea” 的 TaskDisplayArea：

```
final TaskDisplayArea defaultTaskDisplayArea = new TaskDisplayArea(content, wmService,
                    "DefaultTaskDisplayArea", FEATURE_DEFAULT_TASK_CONTAINER);
```

然后将该 TaskDisplayArea 添加到一个局部 List 中，调用 HierarchyBuilder.setTaskDisplayAreas 方法，将该对象保存在了 HierarchyBuilder.mTaskDisplayAreas 中。

```
private final ArrayList<TaskDisplayArea> mTaskDisplayAreas = new ArrayList<>();
        
		// ......

        /**
         * Sets {@link TaskDisplayArea} that are children of this hierarchy root.
         * {@link DisplayArea} group must have at least one {@link TaskDisplayArea}.
         */
        HierarchyBuilder setTaskDisplayAreas(List<TaskDisplayArea> taskDisplayAreas) {
            mTaskDisplayAreas.clear();
            mTaskDisplayAreas.addAll(taskDisplayAreas);
            return this;
        }
```

另外看到，虽然 TaskDisplayArea 是支持嵌套的，并且这里也采用了一个 ArrayList 来管理 TaskDisplayArea，但是目前 TaskDisplayArea 只在这里被创建，即目前一个 DisplayContent 只有一个名为 “DefaultTaskDisplayArea” 的 TaskDisplayArea。

因为 TaskDisplayArea 是用来管理 Task 的，而 Task 本身就支持嵌套，再加上 TaskOrgnizerController，光这两者已经可以为 TaskDisplayArea 建立起一个比较复杂的 Task 层级结构了，因此 TaskDisplayArea 的嵌套目前显得没有必要。

#### 2.2.4 为 HierarchyBuilder 添加 Feature

调用 DisplayAreaPolicy.DefaultProvider.configureTrustedHierarchyBuilder 为 HierarchyBuilder 添加 Feature：

```
if (content.isTrusted()) {
                // Only trusted display can have system decorations.
                configureTrustedHierarchyBuilder(rootHierarchy, wmService, content);
            }
```

接着来看 DisplayAreaPolicy.DefaultProvider.configureTrustedHierarchyBuilder 方法的具体内容：

```
private void configureTrustedHierarchyBuilder(HierarchyBuilder rootHierarchy,
                WindowManagerService wmService, DisplayContent content) {
            // WindowedMagnification should be on the top so that there is only one surface
            // to be magnified.
            rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "WindowedMagnification",
                    FEATURE_WINDOWED_MAGNIFICATION)
                    .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    // Make the DA dimmable so that the magnify window also mirrors the dim layer.
                    .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                    .build());
            if (content.isDefaultDisplay) {
                // Only default display can have cutout.
                // See LocalDisplayAdapter.LocalDisplayDevice#getDisplayDeviceInfoLocked.
                rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "HideDisplayCutout",
                        FEATURE_HIDE_DISPLAY_CUTOUT)
                        .all()
                        .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL, TYPE_STATUS_BAR,
                                TYPE_NOTIFICATION_SHADE)
                        .build())
                        .addFeature(new Feature.Builder(wmService.mPolicy,
                                "OneHandedBackgroundPanel",
                                FEATURE_ONE_HANDED_BACKGROUND_PANEL)
                                .upTo(TYPE_WALLPAPER)
                                .build())
                        .addFeature(new Feature.Builder(wmService.mPolicy, "OneHanded",
                                FEATURE_ONE_HANDED)
                                .all()
                                .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL)
                                .build());
            }
            rootHierarchy
                    .addFeature(new Feature.Builder(wmService.mPolicy, "FullscreenMagnification",
                            FEATURE_FULLSCREEN_MAGNIFICATION)
                            .all()
                            .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY, TYPE_INPUT_METHOD,
                                    TYPE_INPUT_METHOD_DIALOG, TYPE_MAGNIFICATION_OVERLAY,
                                    TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL)
                            .build())
                    .addFeature(new Feature.Builder(wmService.mPolicy, "ImePlaceholder",
                            FEATURE_IME_PLACEHOLDER)
                            .and(TYPE_INPUT_METHOD, TYPE_INPUT_METHOD_DIALOG)
                            .build());
        }
```

如果当前 DisplayContent 是默认屏幕，那么会为 HierarchyBuilder 添加 6 种 Feature，否则是 3 种，我们只分析一般情况，即当前屏幕是默认屏幕的情况。

Feature 是 DisplayAreaPolicyBuilder 的内部类：

```
/**
     * A feature that requires {@link DisplayArea DisplayArea(s)}.
     */
    static class Feature {
        private final String mName;
        private final int mId;
        private final boolean[] mWindowLayers;
        private final NewDisplayAreaSupplier mNewDisplayAreaSupplier;
```

首先 Feature 代表的是 DisplayArea 的一个特征，可以根据 Feature 来对不同的 DisplayArea 进行划分。

看一下它的成员变量：

*   mName，很好理解，这个 Feature 的名字，如上面的 “WindowedMagnification”，“HideDisplayCutout” 之类的，后续 DisplayArea 层级结构建立起来后，每个 DisplayArea 的名字用的就是当前 DisplayArea 对应的那个 Feature 的名字。
*   mId，Feature 的 ID，如上面的 FEATURE_WINDOWED_MAGNIFICATION 和 FEATURE_HIDE_DISPLAY_CUTOUT，虽说是 Feature 的 ID，因为 Feature 又是 DisplayArea 的特征，所以这个 ID 也可以直接代表 DisplayArea 的一种特征。
*   mWindowLayers，代表了这个 DisplayArea 可以包含哪些层级对应的窗口，后续会分析到。
*   mNewDisplayAreaSupplier，

先看第一个 Feature：

```
// WindowedMagnification should be on the top so that there is only one surface
            // to be magnified.
            rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "WindowedMagnification",
                    FEATURE_WINDOWED_MAGNIFICATION)
                    .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    // Make the DA dimmable so that the magnify window also mirrors the dim layer.
                    .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                    .build());
```

1）、设置 Feature 的 mName 为 "WindowedMagnification"。

2）、设置 Feature 的 mId 为 FEATURE_WINDOWED_MAGNIFICATION，FEATURE_WINDOWED_MAGNIFICATION 定义在 DisplayAreaOrganizer 中：

```
/**
     * Display area that can be magnified in
     * {@link Settings.Secure.ACCESSIBILITY_MAGNIFICATION_MODE_WINDOW}. It contains all windows
     * below {@link WindowManager.LayoutParams#TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY}.
     */
    public static final int FEATURE_WINDOWED_MAGNIFICATION = FEATURE_SYSTEM_FIRST + 4;
```

代表了该 DisplayArea 在 ACCESSIBILITY_MAGNIFICATION_MODE_WINDOW 模式下可以进行放大，该 DisplayArea 包含了所有比 TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY 类型的窗口层级低的窗口，为什么这么说，继续看下面。

3）、设置 Feature 的 mWindowLayers：

```
.upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
```

首先 TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY 代表的是一种窗口类型：

```
/**
         * Window type: Window for adding accessibility window magnification above other windows.
         * This will place the window in the overlay windows.
         * @hide
         */
        public static final int TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY = FIRST_SYSTEM_WINDOW + 39;
```

接着看下 upTo 方法：

```
/**
             * Set that the feature applies window types that are layerd at or below the layer of
             * the given window type.
             */
            Builder upTo(int typeInclusive) {
                final int max = layerFromType(typeInclusive, false);
                for (int i = 0; i < max; i++) {
                    mLayers[i] = true;
                }
                set(typeInclusive, true);
                return this;
            }
```

继续看下 layerFromType 方法：

```
private int layerFromType(int type, boolean internalWindows) {
                return mPolicy.getWindowLayerFromTypeLw(type, internalWindows);
            }
```

调用的是 WindowManagerPolicy.getWindowLayerFromTypeLw 方法：

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
     * @return int An arbitrary integer used to order windows, with lower numbers below higher ones.
     */
    default int getWindowLayerFromTypeLw(int type, boolean canAddInternalSystemWindow) {
        return getWindowLayerFromTypeLw(type, canAddInternalSystemWindow,
                false /* roundedCornerOverlay */);
    }

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

知道了这些内容再回头看 upTo 方法，就很容易理解了：

首先找到 TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY 这个窗口类型对应的层级值，也就是 32，接着将 mLayers 数组从 0 到 32 的元素，全部赋值为 true。

看下 Feature.mBuilder 的成员变量 mLayers 的定义：

```
static class Builder {
            // ......
            private final boolean[] mLayers;
            // ......

            /**
             * Builds a new feature that applies to a set of window types as specified by the
             * builder methods.
             *
             * <p>The set of types is updated iteratively in the order of the method invocations.
             * For example, {@code all().except(TYPE_STATUS_BAR)} expresses that a feature should
             * apply to all types except TYPE_STATUS_BAR.
             *
             * <p>The builder starts out with the feature not applying to any types.
             *
             * @param name the name of the feature.
             * @param id of the feature. {@see Feature#getId}
             */
            Builder(WindowManagerPolicy policy, String name, int id) {
                mPolicy = policy;
                mName = name;
                mId = id;
                mLayers = new boolean[mPolicy.getMaxWindowLayer() + 1];
            }
            
            // ......
        }
```

WindowManagerPolicy.getMaxWindowLayer 方法是：

```
// TODO(b/155340867): consider to remove the logic after using pure Surface for rounded corner
    //  overlay.
    /**
     * Returns the max window layer.
     * <p>Note that the max window layer should be higher that the maximum value which reported
     * by {@link #getWindowLayerFromTypeLw(int, boolean)} to contain rounded corner overlay.</p>
     *
     * @see WindowManager.LayoutParams#PRIVATE_FLAG_IS_ROUNDED_CORNERS_OVERLAY
     */
    default int getMaxWindowLayer() {
        return 36;
    }
```

返回了目前 WindowManagerPolicy 定义的最大层级值，36。

那么 Feature.Builder.mLayers 数组初始化为：

```
mLayers = new boolean[37];
```

Feature.mWindowLayers 则是在 Feature.Builder.build 方法中完全由 Feature.Builder.mLayers 的值复制过来：

```
Feature build() {
                if (mExcludeRoundedCorner) {
                    // Always put the rounded corner layer to the top most layer.
                    mLayers[mPolicy.getMaxWindowLayer()] = false;
                }
                return new Feature(mName, mId, mLayers.clone(), mNewDisplayAreaSupplier);
            }
```

将 Feature.Builde.mLayer 数组中的每个元素设置为 true 的意思是什么呢？打个比方，将 Feature.Builde.mLayer[32] 设置为 true，后续调用 Feature.Builder.build 后，Feature.mWindowLayers[32] 也会被设置 true。而 Feature.mWindowLayers[32] 为 true，则表示该 Feature 对应的 DisplayArea，可以包含层级值为 32，也就是窗口类型为 TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY 的窗口。

另外目前看到 mExcludeRoundedCorner 是始终为 true 的，也就是说，mLayers[36] 始终为 false。

看下都有哪些方法来设置 Feature.Builde.mLayer：

```
/**
             * Set that the feature applies to all window types.
             */
            Builder all() {
                Arrays.fill(mLayers, true);
                return this;
            }

            /**
             * Set that the feature applies to the given window types.
             */
            Builder and(int... types) {
                for (int i = 0; i < types.length; i++) {
                    int type = types[i];
                    set(type, true);
                }
                return this;
            }

            /**
             * Set that the feature does not apply to the given window types.
             */
            Builder except(int... types) {
                for (int i = 0; i < types.length; i++) {
                    int type = types[i];
                    set(type, false);
                }
                return this;
            }

            /**
             * Set that the feature applies window types that are layerd at or below the layer of
             * the given window type.
             */
            Builder upTo(int typeInclusive) {
                final int max = layerFromType(typeInclusive, false);
                for (int i = 0; i < max; i++) {
                    mLayers[i] = true;
                }
                set(typeInclusive, true);
                return this;
            }
```

*   all，将 mLayers 数组中的所有元素都设置为 true，表示当前 DisplayArea 可以包含所有类型的窗口。
*   and，先将传入的窗口类型先转换为对应的层级值，然后将 mLayers 数组中与该层级值对应的元素设置为 true，表示该 DisplayArea 可以包含传入的窗口类型对应的窗口。
*   except，先将传入的窗口类型先转换为对应的层级值，然后将 mLayers 数组中与该层级值对应的元素设置为 false，表示该 DisplayArea 不再包含传入的窗口类型对应的窗口。
*   upTo，先将传入的窗口类型先转换为对应的层级值，然后将 mLayers 数组中与该层级值对应的的元素之前的所有元素（包含该元素）设置为 true，表示当前 DisplayArea 可以包含比传入的窗口类型层级值低的所有窗口。

最后，总结一下这一节的内容。

```
rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "WindowedMagnification",
                    FEATURE_WINDOWED_MAGNIFICATION)
                    .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    // Make the DA dimmable so that the magnify window also mirrors the dim layer.
                    .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                    .build());
```

以上代码，为 HierarchyBuilder 添加了一个 Feature，Feature 名字是 "WindowedMagnification"，ID 为 FEATURE_WINDOWED_MAGNIFICATION，符合这个 Feature 的 DisplayArea 可以包含层级值为 0~31 的窗口。

后续添加其他的 Feature 和这个 WindowedMagnification 类似，将指定的窗口类型转换为层级值，可以得到：

<table><thead><tr><th>Feature.mName</th><th>Feature.mID</th><th>Feature.mWindowLayers</th></tr></thead><tbody><tr><td>WindowedMagnification</td><td>FEATURE_WINDOWED_MAGNIFICATION</td><td>0-31</td></tr><tr><td>HideDisplayCutout</td><td>FEATURE_HIDE_DISPLAY_CUTOUT</td><td>0-16,18,20-23,26-35</td></tr><tr><td>OneHandedBackgroundPanel</td><td>FEATURE_ONE_HANDED_BACKGROUND_PANEL</td><td>0-1</td></tr><tr><td>OneHanded</td><td>FEATURE_ONE_HANDED</td><td>0-23,26-35</td></tr><tr><td>FullscreenMagnification</td><td>FEATURE_FULLSCREEN_MAGNIFICATION</td><td>0-14,17-23,26-27,29-31,33-35</td></tr><tr><td>ImePlaceholder</td><td>FEATURE_IME_PLACEHOLDER</td><td>15-16</td></tr></tbody></table>

#### 2.2.5 生成 DisplayArea 层级结构

```
// Instantiate the policy with the hierarchy defined above. This will create and attach
            // all the necessary DisplayAreas to the root.
            return new DisplayAreaPolicyBuilder().setRootHierarchy(rootHierarchy).build(wmService);
```

先调用 DisplayAreaPolicyBuilder.setRootHierarchy 将上面创建的 HierarchyBuilder 对象保存在 DisplayAreaPolicyBuilder 的成员变量 mRootHierarchyBuilder 中。

```
/** Defines the root hierarchy for the whole logical display. */
    DisplayAreaPolicyBuilder setRootHierarchy(HierarchyBuilder rootHierarchyBuilder) {
        mRootHierarchyBuilder = rootHierarchyBuilder;
        return this;
    }
```

最后调用了 DisplayAreaPolicyBuilder.build 方法去生成了一个 DisplayArea 层级结构。

```
Result build(WindowManagerService wmService) {
        validate();

        // Attach DA group roots to screen hierarchy before adding windows to group hierarchies.
        mRootHierarchyBuilder.build(mDisplayAreaGroupHierarchyBuilders);
        List<RootDisplayArea> displayAreaGroupRoots = new ArrayList<>(
                mDisplayAreaGroupHierarchyBuilders.size());
        for (int i = 0; i < mDisplayAreaGroupHierarchyBuilders.size(); i++) {
            HierarchyBuilder hierarchyBuilder = mDisplayAreaGroupHierarchyBuilders.get(i);
            hierarchyBuilder.build();
            displayAreaGroupRoots.add(hierarchyBuilder.mRoot);
        }
        // Use the default function if it is not specified otherwise.
        if (mSelectRootForWindowFunc == null) {
            mSelectRootForWindowFunc = new DefaultSelectRootForWindowFunction(
                    mRootHierarchyBuilder.mRoot, displayAreaGroupRoots);
        }
        return new Result(wmService, mRootHierarchyBuilder.mRoot, displayAreaGroupRoots,
                mSelectRootForWindowFunc);
    }
```

这里调用了 HierarchyBuilder.build 去生成 DisplayArea 层级结构，并且有一个传参 mDisplayAreaGroupHierarchyBuilders。目前我看到对于 mDisplayAreaGroupHierarchyBuilders 来说，没有添加子元素的地方，因此传入的 mDisplayAreaGroupHierarchyBuilders 是一个空的列表，接着分析 HierarchyBuilder.build。

### 2.3 DisplayAreaPolicyBuilder.HierarchyBuilder.build

```
/**
         * Builds the {@link DisplayArea} hierarchy below root. And adds the roots of those
         * {@link HierarchyBuilder} as children.
         */
        private void build(@Nullable List<HierarchyBuilder> displayAreaGroupHierarchyBuilders) {
            final WindowManagerPolicy policy = mRoot.mWmService.mPolicy;
            final int maxWindowLayerCount = policy.getMaxWindowLayer() + 1;
            final DisplayArea.Tokens[] displayAreaForLayer =
                    new DisplayArea.Tokens[maxWindowLayerCount];
            final Map<Feature, List<DisplayArea<WindowContainer>>> featureAreas =
                    new ArrayMap<>(mFeatures.size());
            for (int i = 0; i < mFeatures.size(); i++) {
                featureAreas.put(mFeatures.get(i), new ArrayList<>());
            }

            // This method constructs the layer hierarchy with the following properties:
            // (1) Every feature maps to a set of DisplayAreas
            // (2) After adding a window, for every feature the window's type belongs to,
            //     it is a descendant of one of the corresponding DisplayAreas of the feature.
            // (3) Z-order is maintained, i.e. if z-range(area) denotes the set of layers of windows
            //     within a DisplayArea:
            //      for every pair of DisplayArea siblings (a,b), where a is below b, it holds that
            //      max(z-range(a)) <= min(z-range(b))
            //
            // The algorithm below iteratively creates such a hierarchy:
            //  - Initially, all windows are attached to the root.
            //  - For each feature we create a set of DisplayAreas, by looping over the layers
            //    - if the feature does apply to the current layer, we need to find a DisplayArea
            //      for it to satisfy (2)
            //      - we can re-use the previous layer's area if:
            //         the current feature also applies to the previous layer, (to satisfy (3))
            //         and the last feature that applied to the previous layer is the same as
            //           the last feature that applied to the current layer (to satisfy (2))
            //      - otherwise we create a new DisplayArea below the last feature that applied
            //        to the current layer

            PendingArea[] areaForLayer = new PendingArea[maxWindowLayerCount];
            final PendingArea root = new PendingArea(null, 0, null);
            Arrays.fill(areaForLayer, root);

            // Create DisplayAreas to cover all defined features.
            final int size = mFeatures.size();
            for (int i = 0; i < size; i++) {
                // Traverse the features with the order they are defined, so that the early defined
                // feature will be on the top in the hierarchy.
                final Feature feature = mFeatures.get(i);
                PendingArea featureArea = null;
                for (int layer = 0; layer < maxWindowLayerCount; layer++) {
                    if (feature.mWindowLayers[layer]) {
                        // This feature will be applied to this window layer.
                        //
                        // We need to find a DisplayArea for it:
                        // We can reuse the existing one if it was created for this feature for the
                        // previous layer AND the last feature that applied to the previous layer is
                        // the same as the feature that applied to the current layer (so they are ok
                        // to share the same parent DisplayArea).
                        if (featureArea == null || featureArea.mParent != areaForLayer[layer]) {
                            // No suitable DisplayArea:
                            // Create a new one under the previous area (as parent) for this layer.
                            featureArea = new PendingArea(feature, layer, areaForLayer[layer]);
                            areaForLayer[layer].mChildren.add(featureArea);
                        }
                        areaForLayer[layer] = featureArea;
                    } else {
                        // This feature won't be applied to this window layer. If it needs to be
                        // applied to the next layer, we will need to create a new DisplayArea for
                        // that.
                        featureArea = null;
                    }
                }
            }

            // Create Tokens as leaf for every layer.
            PendingArea leafArea = null;
            int leafType = LEAF_TYPE_TOKENS;
            for (int layer = 0; layer < maxWindowLayerCount; layer++) {
                int type = typeOfLayer(policy, layer);
                // Check whether we can reuse the same Tokens with the previous layer. This happens
                // if the previous layer is the same type as the current layer AND there is no
                // feature that applies to only one of them.
                if (leafArea == null || leafArea.mParent != areaForLayer[layer]
                        || type != leafType) {
                    // Create a new Tokens for this layer.
                    leafArea = new PendingArea(null /* feature */, layer, areaForLayer[layer]);
                    areaForLayer[layer].mChildren.add(leafArea);
                    leafType = type;
                    if (leafType == LEAF_TYPE_TASK_CONTAINERS) {
                        // We use the passed in TaskDisplayAreas for task container type of layer.
                        // Skip creating Tokens even if there is no TDA.
                        addTaskDisplayAreasToApplicationLayer(areaForLayer[layer]);
                        addDisplayAreaGroupsToApplicationLayer(areaForLayer[layer],
                                displayAreaGroupHierarchyBuilders);
                        leafArea.mSkipTokens = true;
                    } else if (leafType == LEAF_TYPE_IME_CONTAINERS) {
                        // We use the passed in ImeContainer for ime container type of layer.
                        // Skip creating Tokens even if there is no ime container.
                        leafArea.mExisting = mImeContainer;
                        leafArea.mSkipTokens = true;
                    }
                }
                leafArea.mMaxLayer = layer;
            }
            root.computeMaxLayer();

            // We built a tree of PendingAreas above with all the necessary info to represent the
            // hierarchy, now create and attach real DisplayAreas to the root.
            root.instantiateChildren(mRoot, displayAreaForLayer, 0, featureAreas);

            // Notify the root that we have finished attaching all the DisplayAreas. Cache all the
            // feature related collections there for fast access.
            mRoot.onHierarchyBuilt(mFeatures, displayAreaForLayer, featureAreas);
        }
```

这个方法虽然很长，但是明显可以分成几部分来看。

#### 2.3.1 PendingArea 数组

```
final WindowManagerPolicy policy = mRoot.mWmService.mPolicy;
            final int maxWindowLayerCount = policy.getMaxWindowLayer() + 1;
            final DisplayArea.Tokens[] displayAreaForLayer =
                    new DisplayArea.Tokens[maxWindowLayerCount];
            final Map<Feature, List<DisplayArea<WindowContainer>>> featureAreas =
                    new ArrayMap<>(mFeatures.size());
            for (int i = 0; i < mFeatures.size(); i++) {
                featureAreas.put(mFeatures.get(i), new ArrayList<>());
            }

            // This method constructs the layer hierarchy with the following properties:
            // (1) Every feature maps to a set of DisplayAreas
            // (2) After adding a window, for every feature the window's type belongs to,
            //     it is a descendant of one of the corresponding DisplayAreas of the feature.
            // (3) Z-order is maintained, i.e. if z-range(area) denotes the set of layers of windows
            //     within a DisplayArea:
            //      for every pair of DisplayArea siblings (a,b), where a is below b, it holds that
            //      max(z-range(a)) <= min(z-range(b))
            //
            // The algorithm below iteratively creates such a hierarchy:
            //  - Initially, all windows are attached to the root.
            //  - For each feature we create a set of DisplayAreas, by looping over the layers
            //    - if the feature does apply to the current layer, we need to find a DisplayArea
            //      for it to satisfy (2)
            //      - we can re-use the previous layer's area if:
            //         the current feature also applies to the previous layer, (to satisfy (3))
            //         and the last feature that applied to the previous layer is the same as
            //           the last feature that applied to the current layer (to satisfy (2))
            //      - otherwise we create a new DisplayArea below the last feature that applied
            //        to the current layer

            PendingArea[] areaForLayer = new PendingArea[maxWindowLayerCount];
            final PendingArea root = new PendingArea(null, 0, null);
            Arrays.fill(areaForLayer, root);
```

1）、maxWindowLayerCount 根据我们之前的分析，为 37，目前我们定义的窗口层级值为 0~36。

2）、创建了一个大小为 37 的 PendingArea 数组：

```
PendingArea[] areaForLayer = new PendingArea[maxWindowLayerCount];
```

创建了一个 PendingArea 对象，填充到 areaForLayer 数组中：

```
final PendingArea root = new PendingArea(null, 0, null);
            Arrays.fill(areaForLayer, root);
```

PendingArea 对象定义在 DisplayAreaPolicyBuilder 中：

```
static class PendingArea {
        final int mMinLayer;
        final ArrayList<PendingArea> mChildren = new ArrayList<>();
        final Feature mFeature;
        final PendingArea mParent;
        int mMaxLayer;
```

我们正在分析的这个方法，就是首先要生成一个 PendingArea 树，然后根据这个 PendingArea 树去生成一个 DisplayArea 树，一个 PendingArea 对应着一个 DisplayArea：

*   PendingArea 是一个树节点的话，那么 PendingArea 的 mParent 则代表当前节点的父节点，mChildren 则表示的是当前节点的子节点。
    
*   mFeature 对应的则是这个 PendingArea 的特征。
    
*   mMinLayer 和 mMaxLayer 代表的是当前 PendingArea 可以容纳的窗口层级值的一个范围。
    

为了接下来的分析，根据 PendingArea 的成员变量，可以将 PendingArea 表述为以下形式：

```
mFeature.mName:mMinLayer:mMaxLayer
```

比如，“OneHandedBackgroundPanel:0:1”，表示名为 OneHandedBackgroundPanel 的 PendingArea，可以容纳层级值从 0 到 1 的窗口。

root 现在没有一个 Feature，因此名字暂时认为是 Root，此时为 “Root:0:0”。

那么 areaForLayer 数组，初始情况为：

<table><thead><tr><th>areaForLayer[37]</th><th>初始</th></tr></thead><tbody><tr><td>areaForLayer[0]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[1]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[2]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[3]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[4]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[5]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[6]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[7]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[8]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[9]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[10]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[11]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[12]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[13]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[14]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[15]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[16]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[17]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[18]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[19]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[20]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[21]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[22]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[23]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[24]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[25]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[26]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[27]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[28]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[29]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[30]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[31]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[32]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[33]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[34]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[35]</td><td>Root:0:0</td></tr><tr><td>areaForLayer[36]</td><td>Root:0:0</td></tr></tbody></table>

#### 2.3.2 PendingArea 数组的生成

```
// Create DisplayAreas to cover all defined features.
            final int size = mFeatures.size();
            for (int i = 0; i < size; i++) {
                // Traverse the features with the order they are defined, so that the early defined
                // feature will be on the top in the hierarchy.
                final Feature feature = mFeatures.get(i);
                PendingArea featureArea = null;
                for (int layer = 0; layer < maxWindowLayerCount; layer++) {
                    if (feature.mWindowLayers[layer]) {
                        // This feature will be applied to this window layer.
                        //
                        // We need to find a DisplayArea for it:
                        // We can reuse the existing one if it was created for this feature for the
                        // previous layer AND the last feature that applied to the previous layer is
                        // the same as the feature that applied to the current layer (so they are ok
                        // to share the same parent DisplayArea).
                        if (featureArea == null || featureArea.mParent != areaForLayer[layer]) {
                            // No suitable DisplayArea:
                            // Create a new one under the previous area (as parent) for this layer.
                            featureArea = new PendingArea(feature, layer, areaForLayer[layer]);
                            areaForLayer[layer].mChildren.add(featureArea);
                        }
                        areaForLayer[layer] = featureArea;
                    } else {
                        // This feature won't be applied to this window layer. If it needs to be
                        // applied to the next layer, we will need to create a new DisplayArea for
                        // that.
                        featureArea = null;
                    }
                }
            }
```

针对每一种 Feature，都会走一次流程，根据 2.2.4 可知，mFeatures 中有 6 个 Feature，并且 Feature 添加到 HierarchyBuilder 的顺序，其实已经代表了这几种 Feature 对应的 DisplayArea 的层级高低。

<table><thead><tr><th>Feature.mName</th><th>Feature.mID</th><th>Feature.mWindowLayers</th></tr></thead><tbody><tr><td>WindowedMagnification</td><td>FEATURE_WINDOWED_MAGNIFICATION</td><td>0-31</td></tr><tr><td>HideDisplayCutout</td><td>FEATURE_HIDE_DISPLAY_CUTOUT</td><td>0-16,18,20-23,26-35</td></tr><tr><td>OneHandedBackgroundPanel</td><td>FEATURE_ONE_HANDED_BACKGROUND_PANEL</td><td>0-1</td></tr><tr><td>OneHanded</td><td>FEATURE_ONE_HANDED</td><td>0-23,26-35</td></tr><tr><td>FullscreenMagnification</td><td>FEATURE_FULLSCREEN_MAGNIFICATION</td><td>0-14,17-23,26-27,29-31,33-35</td></tr><tr><td>ImePlaceholder</td><td>FEATURE_IME_PLACEHOLDER</td><td>15-16</td></tr></tbody></table>

##### 2.3.2.1 WindowedMagnification

先看 WindowedMagnification。

1）、layer 为 0，此时 feature.mWindowLayers[0]为 true，featureArea 为 null，那么创建一个 PendingArea 对象，“WindowedMagnification:0:0”，这个新创建的 “WindowedMagnification:0:0“，parent 为”Root:0:0“，并且将这个新创建的“WindowedMagnification:0:0“添加到”Root:0:0“的子节点中，最后将 areaForLayer[0] 指向这个新创建的“WindowedMagnification:0:0”。

2）、layer 为 1，此时 feature.mWindowLayers[1]为 true，featureArea 为 “WindowedMagnification:0:0”，featureArea.mParent 为”Root:0:0“，areaForLayer[1] 也是”Root:0:0“，那么不创建 PendingArea 对象，将 areaForLayer[1]指向“WindowedMagnification:0:0”。

3）、后续直到 layer 为 32 之前，都是如此。当 layer 为 32 时，feature.mWindowLayers[32] 为 false，不会走到这些逻辑中。

那么经过第一轮循环，areaForLayer 数组的情况是：

<table><thead><tr><th>areaForLayer[37]</th><th>初始</th><th>WindowedMagnification</th></tr></thead><tbody><tr><td>areaForLayer[0]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[1]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[2]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[3]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[4]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[5]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[6]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[7]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[8]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[9]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[10]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[11]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[12]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[13]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[14]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[15]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[16]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[17]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[18]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[19]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[20]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[21]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[22]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[23]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[24]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[25]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[26]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[27]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[28]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[29]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[30]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[31]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[32]</td><td>Root:0:0</td><td></td></tr><tr><td>areaForLayer[33]</td><td>Root:0:0</td><td></td></tr><tr><td>areaForLayer[34]</td><td>Root:0:0</td><td></td></tr><tr><td>areaForLayer[35]</td><td>Root:0:0</td><td></td></tr><tr><td>areaForLayer[36]</td><td>Root:0:0</td><td></td></tr></tbody></table>

转换为树状图：

![](https://img-blog.csdnimg.cn/9aa54af296f44c2c8bd40714890fe879.png#pic_center)

##### 2.3.2.2 HideDisplayCutout

1）、layer 为 0，此时 feature.mWindowLayers[0]为 true，featureArea 为 null，那么创建一个 PendingArea 对象，“HideDisplayCutout:0:0”，这个新创建的 “HideDisplayCutout:0:0“，parent 为 areaForLayer[0]，但是经过第一步，areaForLayer[0] 此时已经从 "Root:0:0" 被替换为了”WindowedMagnification:0:0“。接着将这个新创建的 “HideDisplayCutout:0:0“添加到”WindowedMagnification:0:0“的子节点中，最后将 areaForLayer[0] 指向这个新创建的“HideDisplayCutout:0:0”。

2）、layer 为 1，此时 feature.mWindowLayers[1]为 true，featureArea 为 “HideDisplayCutout:0:0”，featureArea.mParent 为”WindowedMagnification:0:0“，areaForLayer[1] 也是”WindowedMagnification:0:0“，那么不创建 PendingArea 对象，将 areaForLayer[1]指向“HideDisplayCutout:0:0”。

3）、后续直到 layer 为 16，都是如此。

4）、layer 为 17，此时 feature.mWindowLayers[17] 为 false，将 featureArea 重置为 null。

5）、layer 为 18，此时 feature.mWindowLayers[18]为 true，重新创建了一个 PendingArea：”HideDisplayCutout:18:0“，接着将这个新创建的 “HideDisplayCutout:18:0“添加到”WindowedMagnification:0:0“的子节点中，最后将 areaForLayer[18] 指向这个新创建的“HideDisplayCutout:18:0”。

6）、layer 为 19，此时 feature.mWindowLayers[19] 为 false，将 featureArea 重置为 null。

7）、laye 为 20，此时 feature.mWindowLayers[20]为 true，重新创建了一个 PendingArea：”HideDisplayCutout:20:0“，接着将这个新创建的 “HideDisplayCutout:20:0“添加到”WindowedMagnification:0:0“的子节点中，最后将 areaForLayer[20] 指向这个新创建的“HideDisplayCutout:20:0”。

8）、后续直到 layer 为 23，都是复用的”HideDisplayCutout:20:0“。

9）、layer 为 24，此时 feature.mWindowLayers[24] 为 false，将 featureArea 重置为 null。

10）、layer 为 25，此时 feature.mWindowLayers[25] 为 false，将 featureArea 重置为 null。

11）、layer 为 26，此时 feature.mWindowLayers[26]为 true，重新创建了一个 PendingArea：”HideDisplayCutout:26:0“，接着将这个新创建的 “HideDisplayCutout:26:0“添加到”WindowedMagnification:0:0“的子节点中，最后将 areaForLayer[26] 指向这个新创建的“HideDisplayCutout:26:0”。

12）、直到 layer 为 31，都是复用的”HideDisplayCutout:26:0“。

13）、layer 为 32，此时 feature.mWindowLayers[32]为 true，此时 featureArea 为 “HideDisplayCutout:26:0“，parent 为”WindowedMagnification:0:0“，但是此时 areaForLayer[32] 是”Root:0:0“，那么此时会重新创建一个 PendingArea：“HideDisplayCutout:32:0“，接着将这个新创建的 “HideDisplayCutout:32:0“添加到”Root:0:0“的子节点中，最后将 areaForLayer[32] 指向这个新创建的“HideDisplayCutout:32:0”。

14）、后续直到 layer 为 36，都是复用的 “HideDisplayCutout:32:”。

那么经过第二轮循环，areaForLayer 数组的情况是：

<table><thead><tr><th>areaForLayer[37]</th><th>初始</th><th>WindowedMagnification</th><th>HideDisplayCutout</th></tr></thead><tbody><tr><td>areaForLayer[0]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[1]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[2]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[3]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[4]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[5]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[6]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[7]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[8]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[9]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[10]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[11]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[12]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[13]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[14]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[15]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[16]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td></tr><tr><td>areaForLayer[17]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td></tr><tr><td>areaForLayer[18]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:18:0</td></tr><tr><td>areaForLayer[19]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td></tr><tr><td>areaForLayer[20]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td></tr><tr><td>areaForLayer[21]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td></tr><tr><td>areaForLayer[22]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td></tr><tr><td>areaForLayer[23]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td></tr><tr><td>areaForLayer[24]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td></tr><tr><td>areaForLayer[25]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td></tr><tr><td>areaForLayer[26]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td></tr><tr><td>areaForLayer[27]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td></tr><tr><td>areaForLayer[28]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td></tr><tr><td>areaForLayer[29]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td></tr><tr><td>areaForLayer[30]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td></tr><tr><td>areaForLayer[31]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td></tr><tr><td>areaForLayer[32]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td></tr><tr><td>areaForLayer[33]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td></tr><tr><td>areaForLayer[34]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td></tr><tr><td>areaForLayer[35]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td></tr><tr><td>areaForLayer[36]</td><td>Root:0:0</td><td></td><td></td></tr></tbody></table>

转换为树状图：

![](https://img-blog.csdnimg.cn/8fe06fc433844d93a1d2af66132d07d5.png#pic_center)

##### 2.3.2.3 最终结果

后续的分析类似，不再赘述，最终的结果是：

<table><thead><tr><th>areaForLayer[37]</th><th>初始</th><th>WindowedMagnification</th><th>HideDisplayCutout</th><th>OneHandedBackgroundPanel</th><th>OneHanded</th><th>FullscreenMagnification</th><th>ImePlaceholder</th></tr></thead><tbody><tr><td>areaForLayer[0]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td>OneHandedBackgroundPanel:0:0</td><td>OneHanded:0:0</td><td>FullscreenMagnification:0:0</td><td></td></tr><tr><td>areaForLayer[1]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td>OneHandedBackgroundPanel:0:0</td><td>OneHanded:0:0</td><td>FullscreenMagnification:0:0</td><td></td></tr><tr><td>areaForLayer[2]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[3]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[4]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[5]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[6]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[7]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[8]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[9]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[10]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[11]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[12]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[13]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[14]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[15]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td></td><td>ImePlaceholder:15:0</td></tr><tr><td>areaForLayer[16]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td></td><td>ImePlaceholder:15:0</td></tr><tr><td>areaForLayer[17]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td><td></td><td>OneHanded:17:0</td><td>FullscreenMagnification:17:0</td><td></td></tr><tr><td>areaForLayer[18]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:18:0</td><td></td><td>OneHanded:18:0</td><td>FullscreenMagnification:18:0</td><td></td></tr><tr><td>areaForLayer[19]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td><td></td><td>OneHanded:19:0</td><td>FullscreenMagnification:19:0</td><td></td></tr><tr><td>areaForLayer[20]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td><td></td><td>OneHanded:20:0</td><td>FullscreenMagnification:20:0</td><td></td></tr><tr><td>areaForLayer[21]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td><td></td><td>OneHanded:20:0</td><td>FullscreenMagnification:20:0</td><td></td></tr><tr><td>areaForLayer[22]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td><td></td><td>OneHanded:20:0</td><td>FullscreenMagnification:20:0</td><td></td></tr><tr><td>areaForLayer[23]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td><td></td><td>OneHanded:20:0</td><td>FullscreenMagnification:20:0</td><td></td></tr><tr><td>areaForLayer[24]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td><td></td><td></td><td></td><td></td></tr><tr><td>areaForLayer[25]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td><td></td><td></td><td></td><td></td></tr><tr><td>areaForLayer[26]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:26:0</td><td></td></tr><tr><td>areaForLayer[27]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:26:0</td><td></td></tr><tr><td>areaForLayer[28]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td></td><td></td></tr><tr><td>areaForLayer[29]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:29:0</td><td></td></tr><tr><td>areaForLayer[30]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:29:0</td><td></td></tr><tr><td>areaForLayer[31]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:29:0</td><td></td></tr><tr><td>areaForLayer[32]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td><td></td><td>OneHanded:32:0</td><td></td><td></td></tr><tr><td>areaForLayer[33]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td><td></td><td>OneHanded:32:0</td><td>FullscreenMagnification:33:0</td><td></td></tr><tr><td>areaForLayer[34]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td><td></td><td>OneHanded:32:0</td><td>FullscreenMagnification:33:0</td><td></td></tr><tr><td>areaForLayer[35]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td><td></td><td>OneHanded:32:0</td><td>FullscreenMagnification:33:0</td><td></td></tr><tr><td>areaForLayer[36]</td><td>Root:0:0</td><td></td><td></td><td></td><td></td><td></td><td></td></tr></tbody></table>

转换为树状图：

![](https://img-blog.csdnimg.cn/05d3048f502d41ff8f0aee45f2360625.png#pic_center)

#### 2.3.3 为 PendingArea 数组添加 Leaf

```
// Create Tokens as leaf for every layer.
            PendingArea leafArea = null;
            int leafType = LEAF_TYPE_TOKENS;
            for (int layer = 0; layer < maxWindowLayerCount; layer++) {
                int type = typeOfLayer(policy, layer);
                // Check whether we can reuse the same Tokens with the previous layer. This happens
                // if the previous layer is the same type as the current layer AND there is no
                // feature that applies to only one of them.
                if (leafArea == null || leafArea.mParent != areaForLayer[layer]
                        || type != leafType) {
                    // Create a new Tokens for this layer.
                    leafArea = new PendingArea(null /* feature */, layer, areaForLayer[layer]);
                    areaForLayer[layer].mChildren.add(leafArea);
                    leafType = type;
                    if (leafType == LEAF_TYPE_TASK_CONTAINERS) {
                        // We use the passed in TaskDisplayAreas for task container type of layer.
                        // Skip creating Tokens even if there is no TDA.
                        addTaskDisplayAreasToApplicationLayer(areaForLayer[layer]);
                        addDisplayAreaGroupsToApplicationLayer(areaForLayer[layer],
                                displayAreaGroupHierarchyBuilders);
                        leafArea.mSkipTokens = true;
                    } else if (leafType == LEAF_TYPE_IME_CONTAINERS) {
                        // We use the passed in ImeContainer for ime container type of layer.
                        // Skip creating Tokens even if there is no ime container.
                        leafArea.mExisting = mImeContainer;
                        leafArea.mSkipTokens = true;
                    }
                }
                leafArea.mMaxLayer = layer;
            }
```

这个方法比上面的那个简单，继续往 PendingArea 层级结构向下添加 leafArea，最后设置了 leafArea 的 mMaxLayer。

Leaf 有 3 种：

```
private static final int LEAF_TYPE_TASK_CONTAINERS = 1;
        private static final int LEAF_TYPE_IME_CONTAINERS = 2;
        private static final int LEAF_TYPE_TOKENS = 0;
```

根据 Leaf 的父节点的层级值得到：

```
private static int typeOfLayer(WindowManagerPolicy policy, int layer) {
            if (layer == APPLICATION_LAYER) {
                return LEAF_TYPE_TASK_CONTAINERS;
            } else if (layer == policy.getWindowLayerFromTypeLw(TYPE_INPUT_METHOD)
                    || layer == policy.getWindowLayerFromTypeLw(TYPE_INPUT_METHOD_DIALOG)) {
                return LEAF_TYPE_IME_CONTAINERS;
            } else {
                return LEAF_TYPE_TOKENS;
            }
        }
```

总结为：

*   层级值为 APPLICATION_LAYER，即 2，Leaf 的类型为 LEAF_TYPE_TASK_CONTAINERS。
*   层级值为 15，16，Leaf 的类型为 LEAF_TYPE_IME_CONTAINERS。
*   其他层级值对应的 Leaf 类型为 LEAF_TYPE_TOKENS。

看一下这里针对 LEAF_TYPE_TASK_CONTAINERS 和 LEAF_TYPE_IME_CONTAINERS 做的特殊处理：

1）、LEAF_TYPE_TASK_CONTAINERS

```
if (leafType == LEAF_TYPE_TASK_CONTAINERS) {
                        // We use the passed in TaskDisplayAreas for task container type of layer.
                        // Skip creating Tokens even if there is no TDA.
                        addTaskDisplayAreasToApplicationLayer(areaForLayer[layer]);
                        addDisplayAreaGroupsToApplicationLayer(areaForLayer[layer],
                                displayAreaGroupHierarchyBuilders);
                        leafArea.mSkipTokens = true;
                    }
```

根据之前的分析，我们知道 displayAreaGroupHierarchyBuilders 是一个空的列表，所以只看 addTaskDisplayAreasToApplicationLayer 方法。

```
/** Adds all {@link TaskDisplayArea} to the application layer. */
        private void addTaskDisplayAreasToApplicationLayer(PendingArea parentPendingArea) {
            final int count = mTaskDisplayAreas.size();
            for (int i = 0; i < count; i++) {
                PendingArea leafArea =
                        new PendingArea(null /* feature */, APPLICATION_LAYER, parentPendingArea);
                leafArea.mExisting = mTaskDisplayAreas.get(i);
                leafArea.mMaxLayer = APPLICATION_LAYER;
                parentPendingArea.mChildren.add(leafArea);
            }
        }
```

根据 2.2.3 可知，此时的 mTaskDisplayAreas 中只有一个元素，即名为”DefaultTaskDisplayArea“的 TaskDisplayArea 对象，这里是为该对象创建了一个对应的 PendingArea 对象，并且将创建的 PendingArea 添加到 areaForLayer[2] 节点之下，然后将 PendingArea.mExisting 设置为 true

```
/** If not {@code null}, use this instead of creating a {@link DisplayArea.Tokens}. */
        @Nullable DisplayArea mExisting;
```

那么后续根据 PendingArea 生成 DisplayArea.Tokens 的时候，如果 mExisting 不为空，那么直接用 mExisting，而不会再重新创建一个 DisplayArea.Tokens 对象。

另外这里：

```
leafArea.mSkipTokens = true;
```

将为当前节点创建的 leafArea 的 mSkipTokens 设置为 true，那么后续在根据 PendingArea 数组生成 DisplayArea 层级结构的时候，就不会为这个 PendingArea 对象生成一个 DisplayArea 对象了。

一顿操作的最后结果相当于是，用 TaskDisplayArea 对象替换了为当前节点生成的 Leaf。

2）、LEAF_TYPE_IME_CONTAINERS，和上面分析类似，不再为当前节点生成 DisplayArea.Tokens，而是用之前保存在 HierarchyBuilder.mImeContainer 的 ImeContainer。

那么为 2.3.2 生成的 PendingArea 数组添加 Leaf 后，PendingArea 数组为：

<table><thead><tr><th>areaForLayer[37]</th><th>初始</th><th>WindowedMagnification</th><th>HideDisplayCutout</th><th>OneHandedBackgroundPanel</th><th>OneHanded</th><th>FullscreenMagnification</th><th>ImePlaceholder</th><th>Leaf</th></tr></thead><tbody><tr><td>areaForLayer[0]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td>OneHandedBackgroundPanel:0:0</td><td>OneHanded:0:0</td><td>FullscreenMagnification:0:0</td><td></td><td>Leaf:0:1</td></tr><tr><td>areaForLayer[1]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td>OneHandedBackgroundPanel:0:0</td><td>OneHanded:0:0</td><td>FullscreenMagnification:0:0</td><td></td><td>Leaf:0:1</td></tr><tr><td>areaForLayer[2]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>DefaultTaskDisplayArea</td></tr><tr><td>areaForLayer[3]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[4]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[5]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[6]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[7]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[8]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[9]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[10]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[11]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[12]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[13]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[14]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[15]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td></td><td>ImePlaceholder:15:0</td><td>ImeContainer</td></tr><tr><td>areaForLayer[16]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td></td><td>ImePlaceholder:15:0</td><td>ImeContainer</td></tr><tr><td>areaForLayer[17]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td><td></td><td>OneHanded:17:0</td><td>FullscreenMagnification:17:0</td><td></td><td>Leaf:17:17</td></tr><tr><td>areaForLayer[18]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:18:0</td><td></td><td>OneHanded:18:0</td><td>FullscreenMagnification:18:0</td><td></td><td>Leaf:18:18</td></tr><tr><td>areaForLayer[19]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td><td></td><td>OneHanded:19:0</td><td>FullscreenMagnification:19:0</td><td></td><td>Leaf:19:19</td></tr><tr><td>areaForLayer[20]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td><td></td><td>OneHanded:20:0</td><td>FullscreenMagnification:20:0</td><td></td><td>Leaf:20:23</td></tr><tr><td>areaForLayer[21]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td><td></td><td>OneHanded:20:0</td><td>FullscreenMagnification:20:0</td><td></td><td>Leaf:20:23</td></tr><tr><td>areaForLayer[22]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td><td></td><td>OneHanded:20:0</td><td>FullscreenMagnification:20:0</td><td></td><td>Leaf:20:23</td></tr><tr><td>areaForLayer[23]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td><td></td><td>OneHanded:20:0</td><td>FullscreenMagnification:20:0</td><td></td><td>Leaf:20:23</td></tr><tr><td>areaForLayer[24]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td><td></td><td></td><td></td><td></td><td>Leaf:24:25</td></tr><tr><td>areaForLayer[25]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td><td></td><td></td><td></td><td></td><td>Leaf:24:25</td></tr><tr><td>areaForLayer[26]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:26:0</td><td></td><td>Leaf:26:27</td></tr><tr><td>areaForLayer[27]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:26:0</td><td></td><td>Leaf:26:27</td></tr><tr><td>areaForLayer[28]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td></td><td></td><td>Leaf:28:28</td></tr><tr><td>areaForLayer[29]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:29:0</td><td></td><td>Leaf:29:31</td></tr><tr><td>areaForLayer[30]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:29:0</td><td></td><td>Leaf:29:31</td></tr><tr><td>areaForLayer[31]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:29:0</td><td></td><td>Leaf:29:31</td></tr><tr><td>areaForLayer[32]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td><td></td><td>OneHanded:32:0</td><td></td><td></td><td>Leaf:32:32</td></tr><tr><td>areaForLayer[33]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td><td></td><td>OneHanded:32:0</td><td>FullscreenMagnification:33:0</td><td></td><td>Leaf:33:35</td></tr><tr><td>areaForLayer[34]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td><td></td><td>OneHanded:32:0</td><td>FullscreenMagnification:33:0</td><td></td><td>Leaf:33:35</td></tr><tr><td>areaForLayer[35]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td><td></td><td>OneHanded:32:0</td><td>FullscreenMagnification:33:0</td><td></td><td>Leaf:33:35</td></tr><tr><td>areaForLayer[36]</td><td>Root:0:0</td><td></td><td></td><td></td><td></td><td></td><td></td><td>Leaf:36:36</td></tr></tbody></table>

转换为树状图：

![](https://img-blog.csdnimg.cn/a113cef3e7bc41028023d8a2e74b4d2f.png#pic_center)

#### 2.3.4 计算 MaxLayer

2.2.3 只设置了叶节点的 mMaxLayer，这部分计算父节点的 mMaxLayer。

```
root.computeMaxLayer();
```

调用 PendingArea.computeMaxLayer 方法去计算 PendingArea.mMaxLayer 的值：

```
int computeMaxLayer() {
            for (int i = 0; i < mChildren.size(); i++) {
                mMaxLayer = Math.max(mMaxLayer, mChildren.get(i).computeMaxLayer());
            }
            return mMaxLayer;
        }
```

以当前节点为起点，向下查找最大的那个节点 mMaxLayer 作为当前节点的 mMaxLayer。

最终的 PendingArea 数组为：

<table><thead><tr><th>areaForLayer[37]</th><th>初始</th><th>WindowedMagnification</th><th>HideDisplayCutout</th><th>OneHandedBackgroundPanel</th><th>OneHanded</th><th>FullscreenMagnification</th><th>ImePlaceholder</th><th>Leaf</th></tr></thead><tbody><tr><td>areaForLayer[0]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td>OneHandedBackgroundPanel:0:1</td><td>OneHanded:0:1</td><td>FullscreenMagnification:0:1</td><td></td><td>Leaf:0:1</td></tr><tr><td>areaForLayer[1]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td>OneHandedBackgroundPanel:0:1</td><td>OneHanded:0:1</td><td>FullscreenMagnification:0:1</td><td></td><td>Leaf:0:1</td></tr><tr><td>areaForLayer[2]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>DefaultTaskDisplayArea</td></tr><tr><td>areaForLayer[3]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[4]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[5]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[6]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[7]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[8]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[9]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[10]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[11]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[12]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[13]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[14]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td>FullscreenMagnification:2:14</td><td></td><td>Leaf:3:14</td></tr><tr><td>areaForLayer[15]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td></td><td>ImePlaceholder:15:16</td><td>ImeContainer</td></tr><tr><td>areaForLayer[16]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:0:16</td><td></td><td>OneHanded:2:16</td><td></td><td>ImePlaceholder:15:16</td><td>ImeContainer</td></tr><tr><td>areaForLayer[17]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td></td><td></td><td>OneHanded:17:17</td><td>FullscreenMagnification:17:17</td><td></td><td>Leaf:17:17</td></tr><tr><td>areaForLayer[18]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:18:18</td><td></td><td>OneHanded:18:18</td><td>FullscreenMagnification:18:18</td><td></td><td>Leaf:18:18</td></tr><tr><td>areaForLayer[19]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td></td><td></td><td>OneHanded:19:19</td><td>FullscreenMagnification:19:19</td><td></td><td>Leaf:19:19</td></tr><tr><td>areaForLayer[20]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:20:23</td><td></td><td>OneHanded:20:23</td><td>FullscreenMagnification:20:23</td><td></td><td>Leaf:20:23</td></tr><tr><td>areaForLayer[21]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:20:23</td><td></td><td>OneHanded:20:23</td><td>FullscreenMagnification:20:23</td><td></td><td>Leaf:20:23</td></tr><tr><td>areaForLayer[22]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:20:23</td><td></td><td>OneHanded:20:23</td><td>FullscreenMagnification:20:23</td><td></td><td>Leaf:20:23</td></tr><tr><td>areaForLayer[23]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:20:23</td><td></td><td>OneHanded:20:23</td><td>FullscreenMagnification:20:23</td><td></td><td>Leaf:20:23</td></tr><tr><td>areaForLayer[24]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td></td><td></td><td></td><td></td><td></td><td>Leaf:24:25</td></tr><tr><td>areaForLayer[25]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td></td><td></td><td></td><td></td><td></td><td>Leaf:24:25</td></tr><tr><td>areaForLayer[26]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:26:31</td><td></td><td>OneHanded:26:31</td><td>FullscreenMagnification:26:27</td><td></td><td>Leaf:26:27</td></tr><tr><td>areaForLayer[27]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:26:31</td><td></td><td>OneHanded:26:31</td><td>FullscreenMagnification:26:27</td><td></td><td>Leaf:26:27</td></tr><tr><td>areaForLayer[28]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:26:31</td><td></td><td>OneHanded:26:31</td><td></td><td></td><td>Leaf:28:28</td></tr><tr><td>areaForLayer[29]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:26:31</td><td></td><td>OneHanded:26:31</td><td>FullscreenMagnification:29:31</td><td></td><td>Leaf:29:31</td></tr><tr><td>areaForLayer[30]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:26:31</td><td></td><td>OneHanded:26:31</td><td>FullscreenMagnification:29:31</td><td></td><td>Leaf:29:31</td></tr><tr><td>areaForLayer[31]</td><td>Root:0:0</td><td>WindowedMagnification:0:31</td><td>HideDisplayCutout:26:31</td><td></td><td>OneHanded:26:31</td><td>FullscreenMagnification:29:31</td><td></td><td>Leaf:29:31</td></tr><tr><td>areaForLayer[32]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:35</td><td></td><td>OneHanded:32:35</td><td></td><td></td><td>Leaf:32:32</td></tr><tr><td>areaForLayer[33]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:35</td><td></td><td>OneHanded:32:35</td><td>FullscreenMagnification:33:35</td><td></td><td>Leaf:33:35</td></tr><tr><td>areaForLayer[34]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:35</td><td></td><td>OneHanded:32:35</td><td>FullscreenMagnification:33:35</td><td></td><td>Leaf:33:35</td></tr><tr><td>areaForLayer[35]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:35</td><td></td><td>OneHanded:32:35</td><td>FullscreenMagnification:33:35</td><td></td><td>Leaf:33:35</td></tr><tr><td>areaForLayer[36]</td><td>Root:0:0</td><td></td><td></td><td></td><td></td><td></td><td></td><td>Leaf:36:36</td></tr></tbody></table>

转换为树状图：

![](https://img-blog.csdnimg.cn/73d82b61fa8b48168afe1695a5469258.png#pic_center)

#### 2.3.5 生成 DisplayArea 层级结构

```
// We built a tree of PendingAreas above with all the necessary info to represent the
            // hierarchy, now create and attach real DisplayAreas to the root.
            root.instantiateChildren(mRoot, displayAreaForLayer, 0, featureAreas);
```

调用 DisplayAreaPolicyBuilder.PendingArea.instantiateChildren 方法生成 DisplayArea 层级结构。

```
void instantiateChildren(DisplayArea<DisplayArea> parent, DisplayArea.Tokens[] areaForLayer,
                int level, Map<Feature, List<DisplayArea<WindowContainer>>> areas) {
            mChildren.sort(Comparator.comparingInt(pendingArea -> pendingArea.mMinLayer));
            for (int i = 0; i < mChildren.size(); i++) {
                final PendingArea child = mChildren.get(i);
                final DisplayArea area = child.createArea(parent, areaForLayer);
                if (area == null) {
                    // TaskDisplayArea and ImeContainer can be set at different hierarchy, so it can
                    // be null.
                    continue;
                }
                parent.addChild(area, WindowContainer.POSITION_TOP);
                if (child.mFeature != null) {
                    areas.get(child.mFeature).add(area);
                }
                child.instantiateChildren(area, areaForLayer, level + 1, areas);
            }
        }
```

当前 root 是上面的生成的 PendingArea 层级结构的根节点，传入的 parent 则是一个 DisplayContent 对象，那么这里按照 root 的 mChildren 的层级结构，以 parent 为根节点，生成一个 DisplayArea 层级结构。

重点看一下这里创建 DisplayArea 的 DisplayAreaPolicyBuilder.PendingArea.createArea 方法：

```
private DisplayArea createArea(DisplayArea<DisplayArea> parent,
                DisplayArea.Tokens[] areaForLayer) {
            if (mExisting != null) {
                if (mExisting.asTokens() != null) {
                    // Store the WindowToken container for layers
                    fillAreaForLayers(mExisting.asTokens(), areaForLayer);
                }
                return mExisting;
            }
            if (mSkipTokens) {
                return null;
            }
            DisplayArea.Type type;
            if (mMinLayer > APPLICATION_LAYER) {
                type = DisplayArea.Type.ABOVE_TASKS;
            } else if (mMaxLayer < APPLICATION_LAYER) {
                type = DisplayArea.Type.BELOW_TASKS;
            } else {
                type = DisplayArea.Type.ANY;
            }
            if (mFeature == null) {
                final DisplayArea.Tokens leaf = new DisplayArea.Tokens(parent.mWmService, type,
                        "Leaf:" + mMinLayer + ":" + mMaxLayer);
                fillAreaForLayers(leaf, areaForLayer);
                return leaf;
            } else {
                return mFeature.mNewDisplayAreaSupplier.create(parent.mWmService, type,
                        mFeature.mName + ":" + mMinLayer + ":" + mMaxLayer, mFeature.mId);
            }
        }
```

1）、当 mExisting 不为 null，即 2.3.3 中分析的 TaskDisplayArea 和 ImeContainer 的情况，此时不需要再创建 DisplayArea 对象，直接用 mExisting。

2）、mSkipTokens 为 true，直接 return，mSkipTokens 是和 mExisting 是同一个逻辑下设置的。

3）、这里根据 mMinLayer 和 mMaxLayer 的值，将 DisplayArea 分为三种：

*   mMaxLayer 小于 APPLICATION_LAYER，即 2，这类 DisplayArea 类型为 ABOVE_TASKS。
*   mMinLayer 大于 APPLICATION_LAYER，这类 DisplayArea 类型为 BELOW_TASKS。
*   剩余情况下的 DisplayArea 类型为 ANY。

4）、mFeature 为 null，那么创建一个 DisplayArea.Tokens 对象，这种情况对应 2.3.3 节分析的添加 Leaf 节点。

5）、如果 mFeature 不为 null，那么创建一个 DisplayArea 对象。

那么最终生成的 DisplayArea 层级结构为：

```
DisplayContent
		#2 Leaf:36:36
		#1 HideDisplayCutout:32:35
			#0 OneHanded:32:35
				#1 FullscreenMagnification:33:35
					#0 Leaf:33:35
				# 0Leaf:32:32
		#0 WindowedMagnification:0:31
			#6 HideDisplayCutout:26:31
				#0 OneHanded:26:31
					#2 FullscreenMagnification:29:31
						#0 Leaf:29:31
					#1 Leaf:28:28
					#0 FullscreenMagnification:26:27
						#0 Leaf:26:27
			#5 Leaf:24:25
			#4 HideDisplayCutout:20:23
				#0 OneHanded:20:23
					#0 FullscreenMagnification:20:23
						#0 Leaf:20:23
			#3 OneHanded:19:19
				#0 FullscreenMagnification:19:19
					#0 Leaf:19:19
			#2 HideDisplayCutout:18:18
				#0 OneHanded:18:18
					#0 FullscreenMagnification:18:18
						#0 Leaf:18:18
			#1 OneHanded:17:17
				#0 FullscreenMagnification:17:17
					#0 Leaf:17:17
			#0 HideDisplayCutout:0:16
				#1 OneHanded:2:16
					#1 ImePlaceholder:15:16
						#0 ImeContainer
					#0 FullscreenMagnification:2:14
						#1 Leaf:3:14
						#0 DefaultTaskisplayArea
				#0 OneHandedBackgroundPanel:0:1
					#0 OneHanded:0:1
						#0 FullscreenMagnification:0:1
							#0 Leaf:0:1
```

对比 adb shell dumpsys activity containers：

```
ROOT type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
  #0 Display 0  type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][1440,2960] bounds=[0,0][1440,2960]
   #2 Leaf:36:36 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
   #1 HideDisplayCutout:32:35 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
    #0 OneHanded:32:35 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
     #1 FullscreenMagnification:33:35 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
      #0 Leaf:33:35 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
     #0 Leaf:32:32 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
   #0 WindowedMagnification:0:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
    #6 HideDisplayCutout:26:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
     #0 OneHanded:26:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
      #2 FullscreenMagnification:29:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
       #0 Leaf:29:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
      #1 Leaf:28:28 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
      #0 FullscreenMagnification:26:27 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
       #0 Leaf:26:27 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
    #5 Leaf:24:25 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
    #4 HideDisplayCutout:20:23 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
     #0 OneHanded:20:23 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
      #0 FullscreenMagnification:20:23 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
       #0 Leaf:20:23 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
    #3 OneHanded:19:19 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
     #0 FullscreenMagnification:19:19 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
      #0 Leaf:19:19 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
    #2 HideDisplayCutout:18:18 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
     #0 OneHanded:18:18 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
      #0 FullscreenMagnification:18:18 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
       #0 Leaf:18:18 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
    #1 OneHanded:17:17 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
     #0 FullscreenMagnification:17:17 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
      #0 Leaf:17:17 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
    #0 HideDisplayCutout:0:16 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
     #1 OneHanded:2:16 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
      #1 ImePlaceholder:15:16 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
       #0 ImeContainer type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
      #0 FullscreenMagnification:2:14 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
       #1 Leaf:3:14 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
       #0 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
     #0 OneHandedBackgroundPanel:0:1 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
      #0 OneHanded:0:1 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
       #0 FullscreenMagnification:0:1 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
        #0 Leaf:0:1 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
```

是一致的。

#### 2.3.6 保存 Leaf 数组

```
// Notify the root that we have finished attaching all the DisplayAreas. Cache all the
            // feature related collections there for fast access.
            mRoot.onHierarchyBuilt(mFeatures, displayAreaForLayer, featureAreas);
```

回调 RootDisplayArea.onHierarchyBuilt：

```
/** Callback after {@link DisplayArea} hierarchy has been built. */
    void onHierarchyBuilt(ArrayList<Feature> features, DisplayArea.Tokens[] areaForLayer,
            Map<Feature, List<DisplayArea<WindowContainer>>> featureToDisplayAreas) {
        if (mHasBuiltHierarchy) {
            throw new IllegalStateException("Root should only build the hierarchy once");
        }
        mHasBuiltHierarchy = true;
        mFeatures = Collections.unmodifiableList(features);
        mAreaForLayer = areaForLayer;
        mFeatureToDisplayAreas = featureToDisplayAreas;
    }
```

重点看一下第二个参数 areaForLayer，是 DisplayArea.Tokens[] 类型的数组，看一下它的数据是如何得到的。

经过分析看到是在 2.3.5，创建 DisplayArea.Tokens 的时候向 areaForLayer 里填充数据的：

```
if (mFeature == null) {
                final DisplayArea.Tokens leaf = new DisplayArea.Tokens(parent.mWmService, type,
                        "Leaf:" + mMinLayer + ":" + mMaxLayer);
                fillAreaForLayers(leaf, areaForLayer);
                return leaf;
            }
```

再看一下 fillAreaForLayers 方法：

```
private void fillAreaForLayers(DisplayArea.Tokens leaf, DisplayArea.Tokens[] areaForLayer) {
            for (int i = mMinLayer; i <= mMaxLayer; i++) {
                areaForLayer[i] = leaf;
            }
        }
```

也很简单，areaForLayer 数组包含了每一个层级值对应的那个 Leaf，即一个 DisplayArea.Tokens 对象。

3 向 DisplayArea 层级结构添加窗口
------------------------

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
/**
     * DisplayArea that contains WindowTokens, and orders them according to their type.
     */
    public static class Tokens extends DisplayArea<WindowToken> {
```

DisplayArea.Tokens 是 WindowToken 的容器，因此 DisplayArea.Tokens 无法直接添加 WindowState。

WindowToken 则是 WindowState 的容器，每次新 WindowState 创建的时候，都会为这个 WindowState 创建一个 WindowToken 对象，然后将这个新创建的 WindowState 放入其中。

因此我们实际分析的，是 WindowToken 如何添加到 DisplayArea 层级结构中的。

最后需要修正一下说法，因为 WindowToken 不再是一个 DisplayArea，而是一个 WindowContainer。我们之前分析的 DisplayArea 层级结构，其中每一个节点都是一个 DisplayArea，但是现在随着 WindowToken 的加入，DisplayArea 层级结构这个说法需要转化为更通用的结构，也就是 WindowContainer 层级结构。

### 3.1 WindowManagerService.addWindow

```
public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
            int displayId, int requestUserId, InsetsState requestedVisibility,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls) {

        // ......

        synchronized (mGlobalLock) {

            // ......

            WindowToken token = displayContent.getWindowToken(
                    hasParent ? parentWindow.mAttrs.token : attrs.token);

            // ......

            if (token == null) {

                // ......

                if (hasParent) {

                    // ......

                } else {
                    final IBinder binder = attrs.token != null ? attrs.token : client.asBinder();
                    token = new WindowToken.Builder(this, binder, type)
                            .setDisplayContent(displayContent)
                            .setOwnerCanManageAppTokens(session.mCanAddInternalSystemWindow)
                            .setRoundedCornerOverlay(isRoundedCornerOverlay)
                            .build();
                }
            }

            // ......

        }

        // ......

    }
```

调用 WindowToken.Builder.build 创建一个 WindowToken 对象。

### 3.2 WindowToken.Builder.build

```
WindowToken build() {
            return new WindowToken(mService, mToken, mType, mPersistOnEmpty, mDisplayContent,
                    mOwnerCanManageAppTokens, mRoundedCornerOverlay, mFromClientToken, mOptions);
        }
```

### 3.3 WindowToken.init

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

### 3.4 DisplayContent.addWindowToken

```
void addWindowToken(IBinder binder, WindowToken token) {
        final DisplayContent dc = mWmService.mRoot.getWindowTokenDisplay(token);
        // ......

        mTokenMap.put(binder, token);

        if (token.asActivityRecord() == null) {
            // Set displayContent for non-app token to prevent same token will add twice after
            // onDisplayChanged.
            // TODO: Check if it's fine that super.onDisplayChanged of WindowToken
            //  (WindowsContainer#onDisplayChanged) may skipped when token.mDisplayContent assigned.
            token.mDisplayContent = this;
            // Add non-app token to container hierarchy on the display. App tokens are added through
            // the parent container managing them (e.g. Tasks).
            final DisplayArea.Tokens da = findAreaForToken(token).asTokens();
            da.addChild(token);
        }
    }
```

窗口可以分为 App 窗口和非 App 窗口。对于 App 窗口，则是由更细致的 WindowToken 的子类，ActivityRecord 来存放。这里我们分析非 App 窗口的流程。

分两步走：

1）、调用 DisplayContent.findAreaForToken 为当前 WindowToken 寻找一个合适的父容器，DisplayArea.Tokens 对象。

2）、将 WindowToken 添加到父容器中。

### 3.5 DisplayContent.findAreaForToken

```
/**
     * Finds the {@link DisplayArea} for the {@link WindowToken} to attach to.
     * <p>
     * Note that the differences between this API and
     * {@link RootDisplayArea#findAreaForTokenInLayer(WindowToken)} is that this API finds a
     * {@link DisplayArea} in {@link DisplayContent} level, which may find a {@link DisplayArea}
     * from multiple {@link RootDisplayArea RootDisplayAreas} under this {@link DisplayContent}'s
     * hierarchy, while {@link RootDisplayArea#findAreaForTokenInLayer(WindowToken)} finds a
     * {@link DisplayArea.Tokens} from a {@link DisplayArea.Tokens} list mapped to window layers.
     * </p>
     *
     * @see DisplayContent#findAreaForTokenInLayer(WindowToken)
     */
    DisplayArea findAreaForToken(WindowToken windowToken) {
        return findAreaForWindowType(windowToken.getWindowType(), windowToken.mOptions,
                windowToken.mOwnerCanManageAppTokens, windowToken.mRoundedCornerOverlay);
    }
```

为传入的 WindowToken 找到一个 DisplayArea 对象来添加进去。

### 3.6 DisplayContent.findAreaForWindowType

```
DisplayArea findAreaForWindowType(int windowType, Bundle options,
            boolean ownerCanManageAppToken, boolean roundedCornerOverlay) {
        // TODO(b/159767464): figure out how to find an appropriate TDA.
        if (windowType >= FIRST_APPLICATION_WINDOW && windowType <= LAST_APPLICATION_WINDOW) {
            return getDefaultTaskDisplayArea();
        }

        // Return IME container here because it could be in one of sub RootDisplayAreas depending on
        // the focused edit text. Also, the RootDisplayArea choosing strategy is implemented by
        // the server side, but not mSelectRootForWindowFunc customized by OEM.
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

### 3.7 DisplayAreaPolicyBuilder.Result.findAreaForWindowType

```
@Override
        public DisplayArea.Tokens findAreaForWindowType(int type, Bundle options,
                boolean ownerCanManageAppTokens, boolean roundedCornerOverlay) {
            return mSelectRootForWindowFunc.apply(type, options).findAreaForWindowTypeInLayer(type,
                    ownerCanManageAppTokens, roundedCornerOverlay);
        }
```

### 3.8 RootDisplayArea.findAreaForWindowTypeInLayer

```
/** @see #findAreaForTokenInLayer(WindowToken)  */
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
/** Mapping from window layer to {@link DisplayArea.Tokens} that holds windows on that layer. */
    private DisplayArea.Tokens[] mAreaForLayer;
```

它里面的数据是在 2.3.6 节中填充，保存的是层级值到对应 DisplayArea.Tokens 对象的一个映射。

那么只要传入窗口类型，就可以通过 WindowManagerPolicy.getWindowLayerFromTypeLw 得到该窗口类型对应的层级值，然后根据该层级值从 mAreaForLayer 拿到容纳当前 WindowToken 的父容器，一个 DisplayArea.Tokens 对象。

### 3.9 DisplayArea.Tokens.addChild

```
void addChild(WindowToken token) {
            addChild(token, mWindowComparator);
        }
```

这里调用了 WindowContainer.addChild：

```
/**
     * Adds the input window container has a child of this container in order based on the input
     * comparator.
     * @param child The window container to add as a child of this window container.
     * @param comparator Comparator to use in determining the position the child should be added to.
     *                   If null, the child will be added to the top.
     */
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

        // Set the parent after we've actually added a child in case a subclass depends on this.
        child.setParent(this);
    }
```

传入了一个 Comparator：

```
private final Comparator<WindowToken> mWindowComparator =
                Comparator.comparingInt(WindowToken::getWindowLayerFromType);
```

很明显，在将 WindowToken 添加到父容器的时候，将新添加的 WindowToken 的层级值和父容器中的其他 WindowToken 的层级值进行比较，保证新添加的 WindowToken 在父容器中能够按照层级值的大小插入到合适的位置。

4 总结
----

### 4.1 DisplayArea 类型

#### 4.1.1 根据类的继承关系分类

一种是从 DisplayArea 类的继承关系出发，有以下关系：

![](https://img-blog.csdnimg.cn/db4763980edc42a290eab7a71708f00b.png#pic_center)

#### 4.1.2 根据 DisplayArea.Type 类的定义分类

根据 DisplayArea.Type 类的定义：

```
enum Type {
        /** Can only contain WindowTokens above the APPLICATION_LAYER. */
        ABOVE_TASKS,
        /** Can only contain WindowTokens below the APPLICATION_LAYER. */
        BELOW_TASKS,
        /** Can contain anything. */
        ANY;

        // ......
    }
```

可以将 DisplayArea 分为三类，分类的依据当前 DisplayArea 和 APPLICATION_LAYER 的关系：

```
if (mMinLayer > APPLICATION_LAYER) {
                type = DisplayArea.Type.ABOVE_TASKS;
            } else if (mMaxLayer < APPLICATION_LAYER) {
                type = DisplayArea.Type.BELOW_TASKS;
            } else {
                type = DisplayArea.Type.ANY;
            }
```

这种分类，首先把窗口划分为了 App 窗口和非 App 窗口。

其次，对于非 App 窗口，再根据其层级与 App 窗口层级的高低关系，分为位于 App 窗口之上的非 App 窗口（即 ABOVE_TASKS），和位于 App 窗口之下的非 App 窗口（即 BELOW_TASKS）。

#### 4.1.3 根据 Feature 分类

在上面分析 DisplayArea 层级结构的创建流程中，我们在 DisplayAreaPolicy.DefaultProvider.configureTrustedHierarchyBuilder 方法中看到了 6 种 Feature 的添加：

```
private void configureTrustedHierarchyBuilder(HierarchyBuilder rootHierarchy,
                WindowManagerService wmService, DisplayContent content) {
            // WindowedMagnification should be on the top so that there is only one surface
            // to be magnified.
            rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "WindowedMagnification",
                    FEATURE_WINDOWED_MAGNIFICATION)
                    .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    // Make the DA dimmable so that the magnify window also mirrors the dim layer.
                    .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                    .build());
            if (content.isDefaultDisplay) {
                // Only default display can have cutout.
                // See LocalDisplayAdapter.LocalDisplayDevice#getDisplayDeviceInfoLocked.
                rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "HideDisplayCutout",
                        FEATURE_HIDE_DISPLAY_CUTOUT)
                        .all()
                        .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL, TYPE_STATUS_BAR,
                                TYPE_NOTIFICATION_SHADE)
                        .build())
                        .addFeature(new Feature.Builder(wmService.mPolicy,
                                "OneHandedBackgroundPanel",
                                FEATURE_ONE_HANDED_BACKGROUND_PANEL)
                                .upTo(TYPE_WALLPAPER)
                                .build())
                        .addFeature(new Feature.Builder(wmService.mPolicy, "OneHanded",
                                FEATURE_ONE_HANDED)
                                .all()
                                .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL)
                                .build());
            }
            rootHierarchy
                    .addFeature(new Feature.Builder(wmService.mPolicy, "FullscreenMagnification",
                            FEATURE_FULLSCREEN_MAGNIFICATION)
                            .all()
                            .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY, TYPE_INPUT_METHOD,
                                    TYPE_INPUT_METHOD_DIALOG, TYPE_MAGNIFICATION_OVERLAY,
                                    TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL)
                            .build())
                    .addFeature(new Feature.Builder(wmService.mPolicy, "ImePlaceholder",
                            FEATURE_IME_PLACEHOLDER)
                            .and(TYPE_INPUT_METHOD, TYPE_INPUT_METHOD_DIALOG)
                            .build());
        }
    }
```

这些 Feature 都对应一个特定的的 DisplayArea。

但是再看 DisplayArea 的层级结构图：

![](https://img-blog.csdnimg.cn/cb918c67a4a940c8a0af55269071bb31.png#pic_center)

还有一些节点的对应 Feature 却没有看到，即根节点 DisplayContent，叶节点 TaskDisplayArea 和叶节点 Leaf 对应的 DisplayArea.Tokens。

但是每次创建 DisplayArea 的时候都会传入一个对应的 FeatureId 的，之前分析的时候可能没有注意：

1）、DisplayContent：

```
DisplayContent(Display display, RootWindowContainer root) {
        super(root.mWindowManager, "DisplayContent", FEATURE_ROOT);
        // ......
    }
```

2）、TaskDisplayArea：

```
final TaskDisplayArea defaultTaskDisplayArea = new TaskDisplayArea(content, wmService,
                    "DefaultTaskDisplayArea", FEATURE_DEFAULT_TASK_CONTAINER);
```

3）、DisplayArea.Tokens：

```
Tokens(WindowManagerService wms, Type type, String name) {
            super(wms, type, name, FEATURE_WINDOW_TOKENS);
        }
```

这样，所有的 FeatureID，都已经找到了使用的地方了：

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
    

### 4.2 DisplayArea 层级结构生成规则

这里想要讨论一下为什么 DisplayArea 层级结构呈现为这样的形式：

![](https://img-blog.csdnimg.cn/b3365a9477ff4e4cbcd363ff63d73f7b.png#pic_center)

在 2.3 节中分析了生成 DisplayArea 树的流程，但是感觉不够直观，这里借鉴了:

https://blog.csdn.net/shensky711/article/details/121530510

的分析方式，用颜色对各个 DisplayArea 进行标记。

分析前我们需要知道几点前提：

*   属于同一层级值的窗口，统一由一个 Leaf 管理，这个 Leaf 可以是 DisplayArea.Tokens，也可以是 TaskDisplayArea 或者 ImeContainer，这里暂且认为一个 Leaf 代表的就是同一类型的窗口。

*   为 DisplayArea 定义的各种 Feature，代表了这个 DisplayArea 属下的窗口所具有的特征。Leaf 虽然本身拥有的 Feature，如 FEATURE_WINDOW_TOKENS，没有对应的一个具体的功能，但是 Leaf 又是被层级结构中的父节点所管理的，所以它也会拥有父节点 DisplayArea 对应的 Feature 代表的特征。比如一个 Leaf 的父节点是 WindowedMagnification，那么这个 Leaf 管理的所有窗口都具有窗口放大功能。
*   另外虽然一个 DisplayArea 只有一个 Feature，但是由于 DisplayArea 的互相嵌套，那么一个 Leaf 可能会处于多级 DisplayArea 之下，所以一个 Leaf 可能具备多个 Feature，比如 Leaf:33:35，它的父节点从下往上依次是 FullscreenMagnification，OneHanded，HideDisplayCutout，那么这个 Leaf 下的所有窗口，都具备这些 Feature 带来的特征。

以此为基础，来分析一下 DisplayArea 层级结构的生成过程。

1）、由于定义的层级值是 0~36，所以最初我们可能为每一个层级值都创建了一个 Leaf 对象。如果想要某一个 Leaf 拥有某一个 Feature 代表的特征，那么就为这个 Leaf 添加对应的父节点，那么最初的设计可能是这样的：

![](https://img-blog.csdnimg.cn/468c7a93f1934eef8d9df0fcf6834016.png#pic_center)

因为有 37 个 Leaf，所以总共有 37 列。这里的每一个有颜色的格子都代表一个 DisplayArea 对象。空白格子说明该列的 Leaf 管理的窗口不希望有该 Feature 代表的功能，因此没有针对该 Feature 创建一个 DisplayArea 对象。

所以这里有 37 棵独立的 DisplayArea 树，每一个树的根节点都是一个 DisplayContent 对象，叶节点都是一个 Leaf，然后中间节点则是有多有少，这取决于这棵树的 leaf 希望拥有哪些 Feature。

2）、对于每一棵 DisplayArea 树，都是父节点连接子节点，中间不能有空白节点。但是上面的表格，我们能看到是有格子是空白的。我们这里是希望表格同样能够反映 DisplayArea 层级结构，所以我们需要去掉空白格子。

举个例子，看一下 36 列，该列下的 Leaf 不需要任何额外 Feature，因此不需要再为该 Leaf 创建任何父 DisplayArea，直接将该 Leaf 添加到 DisplayContent 的子节点数组中，在表格中就是将该 Leaf 对应的格子上移，直接移动到 DisplayContent 所代表的格子之下。

这也就意味着每一个有颜色的格子如果上方有空白格子，那么就将其上移，最终得到：

![](https://img-blog.csdnimg.cn/dea89761fdfa4130a414beca1cfaaac3.png#pic_center)

3）、现在每一棵树都是父节点连接子节点，且中间没有空白节点了，但是此时并不够成一个层级结构，而仍然是 37 棵独立的树，需要进一步优化。首先我们看到，每一个屏幕只对应一个 DisplayContent 对象，那么这 37 棵树，它们的根节点其实都是同一个 DisplayContent。此外，对于表格中左右相邻的 DisplayArea，如果它们的父 DisplayArea 是同类型的（拥有的 Feature 相同），那么这种情况下，就可以复用父 DisplayArea，即没有必要创建多个 DisplayArea，而是只创建一个父 DisplayArea，然后将这些左右相邻的 DisplayArea 全部添加到该父 DisplayArea 的子节点数组之中，也能达到同样的效果，即这些 DisplayArea 都具有了父 DisplayArea 的 Feature。此时，这些表格中左右相邻的 DisplayArea，由同一个父节点管理，因此表格上看它们左右相邻，在实际的层级结构中，它们也是处于同一层级。

那么表格中如果一个相邻格子的颜色相同，就把这两个格子合并，即 DisplayArea 的复用。最终得到：

![](https://img-blog.csdnimg.cn/fa4bcff67dcc408f94afb6dea81a1d24.png#pic_center)

再对比之前的树状图：

![](https://img-blog.csdnimg.cn/678e763042b844ce82b265befcee7742.png#pic_center)

也是符合的。

### 4.3 向 DisplayArea 层级结构添加窗口

根据 DisplyArea 树状图可知，对于 0~36 的每一个层级值，在 DisplayArea 层级结构中都有相应的 Leaf 对应。因此每次添加新窗口的时候，只需要将该窗口的窗口类型换算为相应的层级值，然后将该新窗口添加到该层级值对应的 Leaf 下即可。

层级值反映了一个 Leaf 在 DisplayArea 层级结构中的层级高低，层级值越大，该 Leaf 在 DisplayArea 层级结构中的层级也越高。而 Leaf 是窗口的容器，Leaf 层级值越大，其管理的窗口在 Z 轴的顺序也就越高。这也说明了窗口类型值越大的窗口，其在 Z 轴上的顺序不一定越高，因为窗口类型值和层级值并不是一个正相关的关系。
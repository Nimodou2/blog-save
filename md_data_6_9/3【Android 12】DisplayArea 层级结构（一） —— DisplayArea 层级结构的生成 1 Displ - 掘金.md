> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7140289958085935141)

> 1 DisplayArea 类的继承关系 DisplayArea 类的继承关系，之前已经分析过，这里用一张简单的类图总结： 2 DisplayArea 层级结构的生成 既然 DisplayContent 是作为

DisplayArea 类的继承关系，之前已经分析过，这里用一张简单的类图总结：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21036b221f5c48b58f567bb12428d2da~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

既然 DisplayContent 是作为其代表的屏幕的 DisplayArea 层级结构的根节点，那么从 DisplayContent 出发，看一下这棵以 DisplayContent 为根节点的 DisplayArea 树是如何生成的。

2.1 DisplayContent.init
-----------------------

```
    
     * Create new {@link DisplayContent} instance, add itself to the root window container and
     * initialize direct children.
     * @param display May not be null.
     * @param root {@link RootWindowContainer}
     */
    DisplayContent(Display display, RootWindowContainer root) {
        

        
        mDisplayAreaPolicy = mWmService.getDisplayAreaPolicyProvider().instantiate(
                mWmService, this , this , mImeWindowsContainer);

        
    }


```

在 DisplayContent 的构造方法中，调用 DisplayAreaPolicy.Provider.instantiate 方法，去初始化一个 DisplayArea 层级结构。

2.2 DisplayAreaPolicy.DefaultProvider.instantiate
-------------------------------------------------

DisplayAreaPolicy.Provider 只是一个接口，instantiate 定义为：

```
        
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

            
            
            final HierarchyBuilder rootHierarchy = new HierarchyBuilder(root);
            
            rootHierarchy.setImeContainer(imeContainer).setTaskDisplayAreas(tdaList);
            if (content.isTrusted()) {
                
                configureTrustedHierarchyBuilder(rootHierarchy, wmService, content);
            }

            
            
            return new DisplayAreaPolicyBuilder().setRootHierarchy(rootHierarchy).build(wmService);
        }


```

### 2.2.1 创建 HierarchyBuilder

首先注意到，这里的 RootDisplayArea 类型的传参 root 即是一个 DisplayContent 对象，毕竟它是要作为后续创建的 DisplayArea 层阶结构的根节点。然后根据这个传入的 DisplayContent，创建一个 HierarchyBuilder 对象：

```
            
            
            final HierarchyBuilder rootHierarchy = new HierarchyBuilder(root);


```

HierarchyBuilder 定义在 DisplayAreaPolicyBuilder 中：

```
    
     *  Builder to define {@link Feature} and {@link DisplayArea} hierarchy under a
     * {@link RootDisplayArea}
     */
    static class HierarchyBuilder {


```

HierarchyBuilder 用来构建一个 DisplayArea 层级结构，该层级结构的根节点，则是 HierarchyBuilder 构造方法中传入的 RootDisplayArea：

```
	private final RootDisplayArea mRoot;

	

	HierarchyBuilder(RootDisplayArea root) {
            mRoot = root;
        }


```

这里传入的则是一个 DisplayContent 对象，那么 HierarchyBuilder 要以这个 DisplayContent 对象为根节点，生成一个 DisplayArea 层级结构。

### 2.2.2 保存 ImeContainer 到 HierarchyBuilder 内部

传参 imeContainer，则是 DisplayContent 的成员变量：

```

private final ImeContainer mImeWindowsContainer = new ImeContainer(mWmService);


```

即 ImeContainer 类型的 DisplayArea，在定义的时候已经初始化。

通过 HierarchyBuilder.setImeContainer 方法：

```
        @Nullable
        private DisplayArea.Tokens mImeContainer;

		

		
        HierarchyBuilder setImeContainer(DisplayArea.Tokens imeContainer) {
            mImeContainer = imeContainer;
            return this;
        }


```

将 DisplayContent 的 mImeWindowsContainer 保存在了 HierarchyBuilder 的 mImeContainer 成员变量中，后续创建 DisplayArea 层级结构时可以直接拿来使用。

### 2.2.3 创建并保存默认 TaskDisplayArea 到 HierarchyBuilder 内部

创建一个名为 “DefaultTaskDisplayArea” 的 TaskDisplayArea：

```
            final TaskDisplayArea defaultTaskDisplayArea = new TaskDisplayArea(content, wmService,
                    "DefaultTaskDisplayArea", FEATURE_DEFAULT_TASK_CONTAINER);


```

然后将该 TaskDisplayArea 添加到一个局部 List 中，调用 HierarchyBuilder.setTaskDisplayAreas 方法，将该对象保存在了 HierarchyBuilder.mTaskDisplayAreas 中。

```
        private final ArrayList<TaskDisplayArea> mTaskDisplayAreas = new ArrayList<>();
        
		

        
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

### 2.2.4 为 HierarchyBuilder 添加 Feature

调用 DisplayAreaPolicy.DefaultProvider.configureTrustedHierarchyBuilder 为 HierarchyBuilder 添加 Feature：

```
            if (content.isTrusted()) {
                
                configureTrustedHierarchyBuilder(rootHierarchy, wmService, content);
            }


```

接着来看 DisplayAreaPolicy.DefaultProvider.configureTrustedHierarchyBuilder 方法的具体内容：

```
        private void configureTrustedHierarchyBuilder(HierarchyBuilder rootHierarchy,
                WindowManagerService wmService, DisplayContent content) {
            
            
            rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "WindowedMagnification",
                    FEATURE_WINDOWED_MAGNIFICATION)
                    .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    
                    .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                    .build());
            if (content.isDefaultDisplay) {
                
                
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
            
            
            rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "WindowedMagnification",
                    FEATURE_WINDOWED_MAGNIFICATION)
                    .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                    
                    .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                    .build());


```

1）、设置 Feature 的 mName 为 "WindowedMagnification"。

2）、设置 Feature 的 mId 为 FEATURE_WINDOWED_MAGNIFICATION，FEATURE_WINDOWED_MAGNIFICATION 定义在 DisplayAreaOrganizer 中：

```
    
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
        
         * Window type: Window for adding accessibility window magnification above other windows.
         * This will place the window in the overlay windows.
         * @hide
         */
        public static final int TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY = FIRST_SYSTEM_WINDOW + 39;


```

接着看下 upTo 方法：

```
            
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
                false );
    }

    
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
        
        if (roundedCornerOverlay && canAddInternalSystemWindow) {
            return getMaxWindowLayer();
        }
        if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
            return APPLICATION_LAYER;
        }

        switch (type) {
            case TYPE_WALLPAPER:
                
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
                
                return  5;
            case TYPE_INPUT_CONSUMER:
                return  6;
            case TYPE_SYSTEM_DIALOG:
                return  7;
            case TYPE_TOAST:
                
                return  8;
            case TYPE_PRIORITY_PHONE:
                
                return  9;
            case TYPE_SYSTEM_ALERT:
                
                
                
                return  canAddInternalSystemWindow ? 13 : 10;
            case TYPE_APPLICATION_OVERLAY:
                return  12;
            case TYPE_INPUT_METHOD:
                
                return  15;
            case TYPE_INPUT_METHOD_DIALOG:
                
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
                
                
                return  22;
            case TYPE_SYSTEM_OVERLAY:
                
                
                return  canAddInternalSystemWindow ? 23 : 11;
            case TYPE_NAVIGATION_BAR:
                
                return  24;
            case TYPE_NAVIGATION_BAR_PANEL:
                
                return  25;
            case TYPE_SCREENSHOT:
                
                
                return  26;
            case TYPE_SYSTEM_ERROR:
                
                return  canAddInternalSystemWindow ? 27 : 10;
            case TYPE_MAGNIFICATION_OVERLAY:
                
                return  28;
            case TYPE_DISPLAY_OVERLAY:
                
                return  29;
            case TYPE_DRAG:
                
                
                return  30;
            case TYPE_ACCESSIBILITY_OVERLAY:
                
                return  31;
            case TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY:
                return 32;
            case TYPE_SECURE_SYSTEM_OVERLAY:
                return  33;
            case TYPE_BOOT_PROGRESS:
                return  34;
            case TYPE_POINTER:
                
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
            
            private final boolean[] mLayers;
            

            
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
            
            
        }


```

WindowManagerPolicy.getMaxWindowLayer 方法是：

```
    
    
    
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
                    
                    mLayers[mPolicy.getMaxWindowLayer()] = false;
                }
                return new Feature(mName, mId, mLayers.clone(), mNewDisplayAreaSupplier);
            }


```

将 Feature.Builde.mLayer 数组中的每个元素设置为 true 的意思是什么呢？打个比方，将 Feature.Builde.mLayer[32] 设置为 true，后续调用 Feature.Builder.build 后，Feature.mWindowLayers[32] 也会被设置 true。而 Feature.mWindowLayers[32] 为 true，则表示该 Feature 对应的 DisplayArea，可以包含层级值为 32，也就是窗口类型为 TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY 的窗口。

另外目前看到 mExcludeRoundedCorner 是始终为 true 的，也就是说，mLayers[36] 始终为 false。

看下都有哪些方法来设置 Feature.Builde.mLayer：

```
            
             * Set that the feature applies to all window types.
             */
            Builder all() {
                Arrays.fill(mLayers, true);
                return this;
            }

            
             * Set that the feature applies to the given window types.
             */
            Builder and(int... types) {
                for (int i = 0; i < types.length; i++) {
                    int type = types[i];
                    set(type, true);
                }
                return this;
            }

            
             * Set that the feature does not apply to the given window types.
             */
            Builder except(int... types) {
                for (int i = 0; i < types.length; i++) {
                    int type = types[i];
                    set(type, false);
                }
                return this;
            }

            
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
                    
                    .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                    .build());


```

以上代码，为 HierarchyBuilder 添加了一个 Feature，Feature 名字是 "WindowedMagnification"，ID 为 FEATURE_WINDOWED_MAGNIFICATION，符合这个 Feature 的 DisplayArea 可以包含层级值为 0~31 的窗口。

后续添加其他的 Feature 和这个 WindowedMagnification 类似，将指定的窗口类型转换为层级值，可以得到：

<table><thead><tr><th>Feature.mName</th><th>Feature.mID</th><th>Feature.mWindowLayers</th></tr></thead><tbody><tr><td>WindowedMagnification</td><td>FEATURE_WINDOWED_MAGNIFICATION</td><td>0-31</td></tr><tr><td>HideDisplayCutout</td><td>FEATURE_HIDE_DISPLAY_CUTOUT</td><td>0-16,18,20-23,26-35</td></tr><tr><td>OneHandedBackgroundPanel</td><td>FEATURE_ONE_HANDED_BACKGROUND_PANEL</td><td>0-1</td></tr><tr><td>OneHanded</td><td>FEATURE_ONE_HANDED</td><td>0-23,26-35</td></tr><tr><td>FullscreenMagnification</td><td>FEATURE_FULLSCREEN_MAGNIFICATION</td><td>0-14,17-23,26-27,29-31,33-35</td></tr><tr><td>ImePlaceholder</td><td>FEATURE_IME_PLACEHOLDER</td><td>15-16</td></tr></tbody></table>

### 2.2.5 生成 DisplayArea 层级结构

```
            
            
            return new DisplayAreaPolicyBuilder().setRootHierarchy(rootHierarchy).build(wmService);


```

先调用 DisplayAreaPolicyBuilder.setRootHierarchy 将上面创建的 HierarchyBuilder 对象保存在 DisplayAreaPolicyBuilder 的成员变量 mRootHierarchyBuilder 中。

```
    
    DisplayAreaPolicyBuilder setRootHierarchy(HierarchyBuilder rootHierarchyBuilder) {
        mRootHierarchyBuilder = rootHierarchyBuilder;
        return this;
    }


```

最后调用了 DisplayAreaPolicyBuilder.build 方法去生成了一个 DisplayArea 层级结构。

```
    Result build(WindowManagerService wmService) {
        validate();

        
        mRootHierarchyBuilder.build(mDisplayAreaGroupHierarchyBuilders);
        List<RootDisplayArea> displayAreaGroupRoots = new ArrayList<>(
                mDisplayAreaGroupHierarchyBuilders.size());
        for (int i = 0; i < mDisplayAreaGroupHierarchyBuilders.size(); i++) {
            HierarchyBuilder hierarchyBuilder = mDisplayAreaGroupHierarchyBuilders.get(i);
            hierarchyBuilder.build();
            displayAreaGroupRoots.add(hierarchyBuilder.mRoot);
        }
        
        if (mSelectRootForWindowFunc == null) {
            mSelectRootForWindowFunc = new DefaultSelectRootForWindowFunction(
                    mRootHierarchyBuilder.mRoot, displayAreaGroupRoots);
        }
        return new Result(wmService, mRootHierarchyBuilder.mRoot, displayAreaGroupRoots,
                mSelectRootForWindowFunc);
    }


```

这里调用了 HierarchyBuilder.build 去生成 DisplayArea 层级结构，并且有一个传参 mDisplayAreaGroupHierarchyBuilders。目前我看到对于 mDisplayAreaGroupHierarchyBuilders 来说，没有添加子元素的地方，因此传入的 mDisplayAreaGroupHierarchyBuilders 是一个空的列表，接着分析 HierarchyBuilder.build。

2.3 DisplayAreaPolicyBuilder.HierarchyBuilder.build
---------------------------------------------------

```
        
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

            
            
            
            
            
            
            
            
            
            
            
            
            
            
            
            
            
            
            
            

            PendingArea[] areaForLayer = new PendingArea[maxWindowLayerCount];
            final PendingArea root = new PendingArea(null, 0, null);
            Arrays.fill(areaForLayer, root);

            
            final int size = mFeatures.size();
            for (int i = 0; i < size; i++) {
                
                
                final Feature feature = mFeatures.get(i);
                PendingArea featureArea = null;
                for (int layer = 0; layer < maxWindowLayerCount; layer++) {
                    if (feature.mWindowLayers[layer]) {
                        
                        
                        
                        
                        
                        
                        
                        if (featureArea == null || featureArea.mParent != areaForLayer[layer]) {
                            
                            
                            featureArea = new PendingArea(feature, layer, areaForLayer[layer]);
                            areaForLayer[layer].mChildren.add(featureArea);
                        }
                        areaForLayer[layer] = featureArea;
                    } else {
                        
                        
                        
                        featureArea = null;
                    }
                }
            }

            
            PendingArea leafArea = null;
            int leafType = LEAF_TYPE_TOKENS;
            for (int layer = 0; layer < maxWindowLayerCount; layer++) {
                int type = typeOfLayer(policy, layer);
                
                
                
                if (leafArea == null || leafArea.mParent != areaForLayer[layer]
                        || type != leafType) {
                    
                    leafArea = new PendingArea(null , layer, areaForLayer[layer]);
                    areaForLayer[layer].mChildren.add(leafArea);
                    leafType = type;
                    if (leafType == LEAF_TYPE_TASK_CONTAINERS) {
                        
                        
                        addTaskDisplayAreasToApplicationLayer(areaForLayer[layer]);
                        addDisplayAreaGroupsToApplicationLayer(areaForLayer[layer],
                                displayAreaGroupHierarchyBuilders);
                        leafArea.mSkipTokens = true;
                    } else if (leafType == LEAF_TYPE_IME_CONTAINERS) {
                        
                        
                        leafArea.mExisting = mImeContainer;
                        leafArea.mSkipTokens = true;
                    }
                }
                leafArea.mMaxLayer = layer;
            }
            root.computeMaxLayer();

            
            
            root.instantiateChildren(mRoot, displayAreaForLayer, 0, featureAreas);

            
            
            mRoot.onHierarchyBuilt(mFeatures, displayAreaForLayer, featureAreas);
        }


```

这个方法虽然很长，但是明显可以分成几部分来看。

### 2.3.1 PendingArea 数组

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

### 2.3.2 PendingArea 数组的生成

```
            
            final int size = mFeatures.size();
            for (int i = 0; i < size; i++) {
                
                
                final Feature feature = mFeatures.get(i);
                PendingArea featureArea = null;
                for (int layer = 0; layer < maxWindowLayerCount; layer++) {
                    if (feature.mWindowLayers[layer]) {
                        
                        if (featureArea == null || featureArea.mParent != areaForLayer[layer]) {
                            
                            
                            featureArea = new PendingArea(feature, layer, areaForLayer[layer]);
                            areaForLayer[layer].mChildren.add(featureArea);
                        }
                        areaForLayer[layer] = featureArea;
                    } else {
                        
                        featureArea = null;
                    }
                }
            }


```

针对每一种 Feature，都会走一次流程，根据 2.2.4 可知，mFeatures 中有 6 个 Feature，并且 Feature 添加到 HierarchyBuilder 的顺序，其实已经代表了这几种 Feature 对应的 DisplayArea 的层级高低。

<table><thead><tr><th>Feature.mName</th><th>Feature.mID</th><th>Feature.mWindowLayers</th></tr></thead><tbody><tr><td>WindowedMagnification</td><td>FEATURE_WINDOWED_MAGNIFICATION</td><td>0-31</td></tr><tr><td>HideDisplayCutout</td><td>FEATURE_HIDE_DISPLAY_CUTOUT</td><td>0-16,18,20-23,26-35</td></tr><tr><td>OneHandedBackgroundPanel</td><td>FEATURE_ONE_HANDED_BACKGROUND_PANEL</td><td>0-1</td></tr><tr><td>OneHanded</td><td>FEATURE_ONE_HANDED</td><td>0-23,26-35</td></tr><tr><td>FullscreenMagnification</td><td>FEATURE_FULLSCREEN_MAGNIFICATION</td><td>0-14,17-23,26-27,29-31,33-35</td></tr><tr><td>ImePlaceholder</td><td>FEATURE_IME_PLACEHOLDER</td><td>15-16</td></tr></tbody></table>

#### 2.3.2.1 WindowedMagnification

先看 WindowedMagnification。

1）、layer 为 0，此时 feature.mWindowLayers[0]为 true，featureArea 为 null，那么创建一个 PendingArea 对象，“WindowedMagnification:0:0”，这个新创建的 “WindowedMagnification:0:0“，parent 为”Root:0:0“，并且将这个新创建的“WindowedMagnification:0:0“添加到”Root:0:0“的子节点中，最后将 areaForLayer[0] 指向这个新创建的“WindowedMagnification:0:0”。

2）、layer 为 1，此时 feature.mWindowLayers[1]为 true，featureArea 为 “WindowedMagnification:0:0”，featureArea.mParent 为”Root:0:0“，areaForLayer[1] 也是”Root:0:0“，那么不创建 PendingArea 对象，将 areaForLayer[1]指向“WindowedMagnification:0:0”。

3）、后续直到 layer 为 32 之前，都是如此。当 layer 为 32 时，feature.mWindowLayers[32] 为 false，不会走到这些逻辑中。

那么经过第一轮循环，areaForLayer 数组的情况是：

<table><thead><tr><th>areaForLayer[37]</th><th>初始</th><th>WindowedMagnification</th></tr></thead><tbody><tr><td>areaForLayer[0]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[1]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[2]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[3]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[4]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[5]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[6]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[7]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[8]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[9]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[10]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[11]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[12]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[13]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[14]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[15]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[16]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[17]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[18]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[19]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[20]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[21]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[22]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[23]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[24]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[25]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[26]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[27]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[28]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[29]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[30]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[31]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td></tr><tr><td>areaForLayer[32]</td><td>Root:0:0</td><td></td></tr><tr><td>areaForLayer[33]</td><td>Root:0:0</td><td></td></tr><tr><td>areaForLayer[34]</td><td>Root:0:0</td><td></td></tr><tr><td>areaForLayer[35]</td><td>Root:0:0</td><td></td></tr><tr><td>areaForLayer[36]</td><td>Root:0:0</td><td></td></tr></tbody></table>

转换为树状图：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/034fddc8132b4079a86042f76b2d282c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 2.3.2.2 HideDisplayCutout

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

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/287ec5bb84fe41c9825a83dc556e9f93~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 2.3.2.3 最终结果

后续的分析类似，不再赘述，最终的结果是：

<table><thead><tr><th>areaForLayer[37]</th><th>初始</th><th>WindowedMagnification</th><th>HideDisplayCutout</th><th>OneHandedBackgroundPanel</th><th>OneHanded</th><th>FullscreenMagnification</th><th>ImePlaceholder</th></tr></thead><tbody><tr><td>areaForLayer[0]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td>OneHandedBackgroundPanel:0:0</td><td>OneHanded:0:0</td><td>FullscreenMagnification:0:0</td><td></td></tr><tr><td>areaForLayer[1]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td>OneHandedBackgroundPanel:0:0</td><td>OneHanded:0:0</td><td>FullscreenMagnification:0:0</td><td></td></tr><tr><td>areaForLayer[2]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[3]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[4]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[5]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[6]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[7]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[8]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[9]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[10]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[11]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[12]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[13]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[14]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td>FullscreenMagnification:2:0</td><td></td></tr><tr><td>areaForLayer[15]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td></td><td>ImePlaceholder:15:0</td></tr><tr><td>areaForLayer[16]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:0:0</td><td></td><td>OneHanded:2:0</td><td></td><td>ImePlaceholder:15:0</td></tr><tr><td>areaForLayer[17]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td><td></td><td>OneHanded:17:0</td><td>FullscreenMagnification:17:0</td><td></td></tr><tr><td>areaForLayer[18]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:18:0</td><td></td><td>OneHanded:18:0</td><td>FullscreenMagnification:18:0</td><td></td></tr><tr><td>areaForLayer[19]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td><td></td><td>OneHanded:19:0</td><td>FullscreenMagnification:19:0</td><td></td></tr><tr><td>areaForLayer[20]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td><td></td><td>OneHanded:20:0</td><td>FullscreenMagnification:20:0</td><td></td></tr><tr><td>areaForLayer[21]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td><td></td><td>OneHanded:20:0</td><td>FullscreenMagnification:20:0</td><td></td></tr><tr><td>areaForLayer[22]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td><td></td><td>OneHanded:20:0</td><td>FullscreenMagnification:20:0</td><td></td></tr><tr><td>areaForLayer[23]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:20:0</td><td></td><td>OneHanded:20:0</td><td>FullscreenMagnification:20:0</td><td></td></tr><tr><td>areaForLayer[24]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td><td></td><td></td><td></td><td></td></tr><tr><td>areaForLayer[25]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td></td><td></td><td></td><td></td><td></td></tr><tr><td>areaForLayer[26]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:26:0</td><td></td></tr><tr><td>areaForLayer[27]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:26:0</td><td></td></tr><tr><td>areaForLayer[28]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td></td><td></td></tr><tr><td>areaForLayer[29]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:29:0</td><td></td></tr><tr><td>areaForLayer[30]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:29:0</td><td></td></tr><tr><td>areaForLayer[31]</td><td>Root:0:0</td><td>WindowedMagnification:0:0</td><td>HideDisplayCutout:26:0</td><td></td><td>OneHanded:26:0</td><td>FullscreenMagnification:29:0</td><td></td></tr><tr><td>areaForLayer[32]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td><td></td><td>OneHanded:32:0</td><td></td><td></td></tr><tr><td>areaForLayer[33]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td><td></td><td>OneHanded:32:0</td><td>FullscreenMagnification:33:0</td><td></td></tr><tr><td>areaForLayer[34]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td><td></td><td>OneHanded:32:0</td><td>FullscreenMagnification:33:0</td><td></td></tr><tr><td>areaForLayer[35]</td><td>Root:0:0</td><td></td><td>HideDisplayCutout:32:0</td><td></td><td>OneHanded:32:0</td><td>FullscreenMagnification:33:0</td><td></td></tr><tr><td>areaForLayer[36]</td><td>Root:0:0</td><td></td><td></td><td></td><td></td><td></td><td></td></tr></tbody></table>

转换为树状图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee6ed0530b2843eb9d05ccbf8fe5d494~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 2.3.3 为 PendingArea 数组添加 Leaf

```
            
            PendingArea leafArea = null;
            int leafType = LEAF_TYPE_TOKENS;
            for (int layer = 0; layer < maxWindowLayerCount; layer++) {
                int type = typeOfLayer(policy, layer);
                
                
                
                if (leafArea == null || leafArea.mParent != areaForLayer[layer]
                        || type != leafType) {
                    
                    leafArea = new PendingArea(null , layer, areaForLayer[layer]);
                    areaForLayer[layer].mChildren.add(leafArea);
                    leafType = type;
                    if (leafType == LEAF_TYPE_TASK_CONTAINERS) {
                        
                        
                        addTaskDisplayAreasToApplicationLayer(areaForLayer[layer]);
                        addDisplayAreaGroupsToApplicationLayer(areaForLayer[layer],
                                displayAreaGroupHierarchyBuilders);
                        leafArea.mSkipTokens = true;
                    } else if (leafType == LEAF_TYPE_IME_CONTAINERS) {
                        
                        
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
                        
                        
                        addTaskDisplayAreasToApplicationLayer(areaForLayer[layer]);
                        addDisplayAreaGroupsToApplicationLayer(areaForLayer[layer],
                                displayAreaGroupHierarchyBuilders);
                        leafArea.mSkipTokens = true;
                    }


```

根据之前的分析，我们知道 displayAreaGroupHierarchyBuilders 是一个空的列表，所以只看 addTaskDisplayAreasToApplicationLayer 方法。

```
        
        private void addTaskDisplayAreasToApplicationLayer(PendingArea parentPendingArea) {
            final int count = mTaskDisplayAreas.size();
            for (int i = 0; i < count; i++) {
                PendingArea leafArea =
                        new PendingArea(null , APPLICATION_LAYER, parentPendingArea);
                leafArea.mExisting = mTaskDisplayAreas.get(i);
                leafArea.mMaxLayer = APPLICATION_LAYER;
                parentPendingArea.mChildren.add(leafArea);
            }
        }


```

根据 2.2.3 可知，此时的 mTaskDisplayAreas 中只有一个元素，即名为”DefaultTaskDisplayArea“的 TaskDisplayArea 对象，这里是为该对象创建了一个对应的 PendingArea 对象，并且将创建的 PendingArea 添加到 areaForLayer[2] 节点之下，然后将 PendingArea.mExisting 设置为 true

```
        
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

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dbb1f8cd79c14b5c97552030ae8eeb22~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 2.3.4 计算 MaxLayer

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

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ef69508680043ca80e8183473ca6593~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 2.3.5 生成 DisplayArea 层级结构

```
            
            
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

### 2.3.6 保存 Leaf 数组

```
            
            
            mRoot.onHierarchyBuilt(mFeatures, displayAreaForLayer, featureAreas);


```

回调 RootDisplayArea.onHierarchyBuilt：

```
    
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
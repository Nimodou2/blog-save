> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7140289813516648456)

> 4 总结 4.1 DisplayArea 类型 4.1.1 根据类的继承关系分类 一种是从 DisplayArea 类的继承关系出发，有以下关系： 4.1.2 根据 DisplayArea.Type 类的定义分

4.1 DisplayArea 类型
------------------

### 4.1.1 根据类的继承关系分类

一种是从 DisplayArea 类的继承关系出发，有以下关系：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5e01411359344c98807739e5aa108175~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 4.1.2 根据 DisplayArea.Type 类的定义分类

根据 DisplayArea.Type 类的定义：

```
    enum Type {
        
        ABOVE_TASKS,
        
        BELOW_TASKS,
        
        ANY;

        
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

### 4.1.3 根据 Feature 分类

在上面分析 DisplayArea 层级结构的创建流程中，我们在 DisplayAreaPolicy.DefaultProvider.configureTrustedHierarchyBuilder 方法中看到了 6 种 Feature 的添加：

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
    }


```

这些 Feature 都对应一个特定的的 DisplayArea。

但是再看 DisplayArea 的层级结构图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c6cf080ebd0b473d93c0fd2ce884affe~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

还有一些节点的对应 Feature 却没有看到，即根节点 DisplayContent，叶节点 TaskDisplayArea 和叶节点 Leaf 对应的 DisplayArea.Tokens。

但是每次创建 DisplayArea 的时候都会传入一个对应的 FeatureId 的，之前分析的时候可能没有注意：

1）、DisplayContent：

```
    DisplayContent(Display display, RootWindowContainer root) {
        super(root.mWindowManager, "DisplayContent", FEATURE_ROOT);
        
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
   
     * The value in display area indicating that no value has been set.
     */
    public static final int FEATURE_UNDEFINED = -1;

    
     * The Root display area on a display
     */
    public static final int FEATURE_SYSTEM_FIRST = 0;

    
     * The Root display area on a display
     */
    public static final int FEATURE_ROOT = FEATURE_SYSTEM_FIRST;

    
     * Display area hosting the default task container.
     */
    public static final int FEATURE_DEFAULT_TASK_CONTAINER = FEATURE_SYSTEM_FIRST + 1;

    
     * Display area hosting non-activity window tokens.
     */
    public static final int FEATURE_WINDOW_TOKENS = FEATURE_SYSTEM_FIRST + 2;

    
     * Display area for one handed feature
     */
    public static final int FEATURE_ONE_HANDED = FEATURE_SYSTEM_FIRST + 3;

    
     * Display area that can be magnified in
     * {@link Settings.Secure.ACCESSIBILITY_MAGNIFICATION_MODE_WINDOW}. It contains all windows
     * below {@link WindowManager.LayoutParams#TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY}.
     */
    public static final int FEATURE_WINDOWED_MAGNIFICATION = FEATURE_SYSTEM_FIRST + 4;

    
     * Display area that can be magnified in
     * {@link Settings.Secure.ACCESSIBILITY_MAGNIFICATION_MODE_FULLSCREEN}. This is different from
     * {@link #FEATURE_WINDOWED_MAGNIFICATION} that the whole display will be magnified.
     * @hide
     */
    public static final int FEATURE_FULLSCREEN_MAGNIFICATION = FEATURE_SYSTEM_FIRST + 5;

    
     * Display area for hiding display cutout feature
     * @hide
     */
    public static final int FEATURE_HIDE_DISPLAY_CUTOUT = FEATURE_SYSTEM_FIRST + 6;

    
     * Display area that the IME container can be placed in. Should be enabled on every root
     * hierarchy if IME container may be reparented to that hierarchy when the IME target changed.
     * @hide
     */
    public static final int FEATURE_IME_PLACEHOLDER = FEATURE_SYSTEM_FIRST + 7;

    
     * Display area for one handed background layer, which preventing when user
     * turning the Dark theme on, they can not clearly identify the screen has entered
     * one handed mode.
     * @hide
     */
    public static final int FEATURE_ONE_HANDED_BACKGROUND_PANEL = FEATURE_SYSTEM_FIRST + 8;

    
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
    

4.2 DisplayArea 层级结构生成规则
------------------------

这里想要讨论一下为什么 DisplayArea 层级结构呈现为这样的形式：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d609f7e40f749fdb29bede2ce86cc10~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

在 2.3 节中分析了生成 DisplayArea 树的流程，但是感觉不够直观，这里借鉴了:

[blog.csdn.net/shensky711/…](https://link.juejin.cn/?target=https%3A%2F%2Fblog.csdn.net%2Fshensky711%2Farticle%2Fdetails%2F121530510 "https://blog.csdn.net/shensky711/article/details/121530510")

的分析方式，用颜色对各个 DisplayArea 进行标记。

分析前我们需要知道几点前提：

*   属于同一层级值的窗口，统一由一个 Leaf 管理，这个 Leaf 可以是 DisplayArea.Tokens，也可以是 TaskDisplayArea 或者 ImeContainer，这里暂且认为一个 Leaf 代表的就是同一类型的窗口。

*   为 DisplayArea 定义的各种 Feature，代表了这个 DisplayArea 属下的窗口所具有的特征。Leaf 虽然本身拥有的 Feature，如 FEATURE_WINDOW_TOKENS，没有对应的一个具体的功能，但是 Leaf 又是被层级结构中的父节点所管理的，所以它也会拥有父节点 DisplayArea 对应的 Feature 代表的特征。比如一个 Leaf 的父节点是 WindowedMagnification，那么这个 Leaf 管理的所有窗口都具有窗口放大功能。
*   另外虽然一个 DisplayArea 只有一个 Feature，但是由于 DisplayArea 的互相嵌套，那么一个 Leaf 可能会处于多级 DisplayArea 之下，所以一个 Leaf 可能具备多个 Feature，比如 Leaf:33:35，它的父节点从下往上依次是 FullscreenMagnification，OneHanded，HideDisplayCutout，那么这个 Leaf 下的所有窗口，都具备这些 Feature 带来的特征。

以此为基础，来分析一下 DisplayArea 层级结构的生成过程。

1）、由于定义的层级值是 0~36，所以最初我们可能为每一个层级值都创建了一个 Leaf 对象。如果想要某一个 Leaf 拥有某一个 Feature 代表的特征，那么就为这个 Leaf 添加对应的父节点，那么最初的设计可能是这样的：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/17006598a4f94c6da215fa4a3d60efdd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

因为有 37 个 Leaf，所以总共有 37 列。这里的每一个有颜色的格子都代表一个 DisplayArea 对象。空白格子说明该列的 Leaf 管理的窗口不希望有该 Feature 代表的功能，因此没有针对该 Feature 创建一个 DisplayArea 对象。

所以这里有 37 棵独立的 DisplayArea 树，每一个树的根节点都是一个 DisplayContent 对象，叶节点都是一个 Leaf，然后中间节点则是有多有少，这取决于这棵树的 leaf 希望拥有哪些 Feature。

2）、对于每一棵 DisplayArea 树，都是父节点连接子节点，中间不能有空白节点。但是上面的表格，我们能看到是有格子是空白的。我们这里是希望表格同样能够反映 DisplayArea 层级结构，所以我们需要去掉空白格子。

举个例子，看一下 36 列，该列下的 Leaf 不需要任何额外 Feature，因此不需要再为该 Leaf 创建任何父 DisplayArea，直接将该 Leaf 添加到 DisplayContent 的子节点数组中，在表格中就是将该 Leaf 对应的格子上移，直接移动到 DisplayContent 所代表的格子之下。

这也就意味着每一个有颜色的格子如果上方有空白格子，那么就将其上移，最终得到：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21d2d0d5d3994186ae43a380d3c6fbcf~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

3）、现在每一棵树都是父节点连接子节点，且中间没有空白节点了，但是此时并不够成一个层级结构，而仍然是 37 棵独立的树，需要进一步优化。首先我们看到，每一个屏幕只对应一个 DisplayContent 对象，那么这 37 棵树，它们的根节点其实都是同一个 DisplayContent。此外，对于表格中左右相邻的 DisplayArea，如果它们的父 DisplayArea 是同类型的（拥有的 Feature 相同），那么这种情况下，就可以复用父 DisplayArea，即没有必要创建多个 DisplayArea，而是只创建一个父 DisplayArea，然后将这些左右相邻的 DisplayArea 全部添加到该父 DisplayArea 的子节点数组之中，也能达到同样的效果，即这些 DisplayArea 都具有了父 DisplayArea 的 Feature。此时，这些表格中左右相邻的 DisplayArea，由同一个父节点管理，因此表格上看它们左右相邻，在实际的层级结构中，它们也是处于同一层级。

那么表格中如果一个相邻格子的颜色相同，就把这两个格子合并，即 DisplayArea 的复用。最终得到：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91d53b0fef0c4b08962636212a838ef9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

再对比之前的树状图：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a76eb0d7db4246b38b815cb5c1ebf5e1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

也是符合的。

4.3 向 DisplayArea 层级结构添加窗口
--------------------------

根据 DisplyArea 树状图可知，对于 0~36 的每一个层级值，在 DisplayArea 层级结构中都有相应的 Leaf 对应。因此每次添加新窗口的时候，只需要将该窗口的窗口类型换算为相应的层级值，然后将该新窗口添加到该层级值对应的 Leaf 下即可。

层级值反映了一个 Leaf 在 DisplayArea 层级结构中的层级高低，层级值越大，该 Leaf 在 DisplayArea 层级结构中的层级也越高。而 Leaf 是窗口的容器，Leaf 层级值越大，其管理的窗口在 Z 轴的顺序也就越高。这也说明了窗口类型值越大的窗口，其在 Z 轴上的顺序不一定越高，因为窗口类型值和层级值并不是一个正相关的关系。
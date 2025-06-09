> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7408399154647285760)

> CtsWindowManagerDeviceAnimations.testRightEdgeExtensionWorksDuringActivityTransition 报错： java.lang.As

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/eab8c2d548b947a69e73f818ef06663c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=SrySxV%2FPat2B2enL0xlr2jmAswE%3D)

CtsWindowManagerDeviceAnimations.testRightEdgeExtensionWorksDuringActivityTransition 报错：

java.lang.AssertionError: No screenshot of the activity transition passed the assertions :: ColorCheckResult{isFailure=true, firstWrongPixel=Point(450, 1263), expectedColor=Color(1.0, 0.0, 0.0, 1.0, sRGB IEC61966-2.1), actualColor=Color(0.0, 0.0, 0.0, 0.0, sRGB IEC61966-2.1)}, ......

1 case 逻辑分析
-----------

根据 case 的内容大概理解一下这个 case 的测试逻辑：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/02ce45ec255540c488f07ccc4576cf93~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=JM%2FDlQZKTkrcgCwyGJ5frYlg1qs%3D)

启动一个名为 EdgeExtensionActivity 的 Activity，这个 Activity 重写了动画，当播放动画时，EdgeExtensionActivity 在 X 轴方向上被缩放到 50%，并且边缘像素延伸到屏幕右边。

因为这个播放动画的 Activity 是一半蓝，一半红的，我们预期前 25% 的像素列是蓝色的（来自 Activity），然后剩下的 75% 的像素列是红色的（25% 来自 Activity，50% 来自 edge extenstion）。

然后在动画过程中截图，做个对比：

左边是我们的机器有问题，可以看到压缩后左半屏右边的红色边缘是没有延伸到右半屏的。

右边是 pixel 没问题，可以看到压缩后左半屏右边的红色像素延伸到了右半屏。

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a12d7565e3e54c2e90a4d7d045d48d66~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=2ESgkzpZWBohdJeQSBUtQ%2Fmq6B0%3D)

再看报错内容：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c2d993ae900945469ca3f8aab263d683~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=fU8jGDSJAhQPcWlOlHhOmQci%2F9w%3D)

ColorCheckResult{isFailure=true, firstWrongPixel=Point(450, 1263), expectedColor=Color(1.0, 0.0, 0.0, 1.0, sRGB IEC61966-2.1), actualColor=Color(0.0, 0.0, 0.0, 0.0, sRGB IEC61966-2.1)}

预期在 Point(450, 1263) 位置的像素颜色应该是：

Color(1.0, 0.0, 0.0, 1.0, sRGB IEC61966-2.1)，即红色。

但是实际上是：

Color(0.0, 0.0, 0.0, 0.0, sRGB IEC61966-2.1)，即黑色。

所以 case fail。

2 本地 Demo 模拟 CTS case
---------------------

这个问题刚拿到后没有什么头绪，而且没有搞懂这个动画是怎么实现的，先看下能否本地写一个 Demo 模拟一下 case 逻辑。

继续梳理 case 的逻辑，这个 case 的核心逻辑为，创建一个名为 EdgeExtensionActivity 的 Activity，这个 Activity 是一半蓝色，一半红色：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android" android:orientation="horizontal" android:layout_width="match_parent" android:layout_height="match_parent">
   <View android:background="#0000ff" android:layout_width="wrap_content" android:layout_height="wrap_content" android:layout_weight="1"/>
   <View android:background="#ff0000" android:layout_width="wrap_content" android:layout_height="wrap_content" android:layout_weight="1"/>
</LinearLayout>


```

就像这样：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4c8f589342fc4ddfb21d39ce06bd74f3~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=sB98JMlgNg0D4dm6HhsoGBwRoH8%3D)

然后这个 Activity 在 onResume 中调用 overridePendingTransition 重写了 enter 动画和 exit 动画：

```
   @Override
   protected void onResume() {
       super.onResume();
        mPendingEnterRes = R.anim.edge_extension_right;
        mPendingExitRes = R.anim.alpha_0;
       overridePendingTransition(mPendingEnterRes, mPendingExitRes);
   }


```

这里我们只关注 enter 动画，动画样式为自定义的 R.anim.edge_extension_right：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ab210bfd854c4e568a446dcedc608ef9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=crNRvutxbkMyofJCSX79MlOd6A8%3D)

这个动画样式我在 aosp 中没找到，反编译了 CTS 的 CtsWindowManagerDeviceAnimations.apk 后得到：

```
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android" android:shareInterpolator="false">
   <alpha android:interpolator="@android:interpolator/linear" android:duration="5000" android:fillBefore="true" android:fillAfter="true" android:startOffset="0" android:fromAlpha="1" android:toAlpha="1" android:fillEnabled="true"/>
   <scale android:interpolator="@android:interpolator/linear" android:duration="5000" android:startOffset="0" android:fromXScale="0.5" android:toXScale="0.5" android:fromYScale="1" android:toYScale="1"/>
   <extend android:interpolator="@android:interpolator/linear" android:duration="5000" android:startOffset="0" android:fromExtendLeft="0" android:fromExtendTop="0" android:fromExtendRight="100%" android:fromExtendBottom="0" android:toExtendLeft="0" android:toExtendTop="0" android:toExtendRight="100%" android:toExtendBottom="0"/>
</set>


```

定义了三种动画：

1）、alpha：前后没有变化。

2）、scale：动画开始的时候 X 轴方向上缩放到 0.5，并且这个缩放比例不变，一直保持到动画结束。

3）、extend，有 8 个特定的描述该类型动画的属性：

*   fromExtendLeft 和 toExtendLeft。
    
*   fromExtendTop 和 toExtendTop。
    
*   fromExtendRight 和 toExtendRight。
    
*   fromExtendBottom 和 toExtendBottom。
    

并且设置了 fromExtendRight 和 toExtendRight 都是 100%，这个属性暂时还不清楚是如何作用。

因为 pixel 上是正常的，因此在 Demo 上做了一些修改后，看到：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3e6c7955fd3348a0a07627dc468f3976~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=geWZjRiCvQASaQhee%2BcJUcAs%2FLI%3D)

进行对比，依稀对 extend 类型的动画有了一点理解，即将 Activity 的边缘像素延伸到屏幕的边缘。

再进行一下实验，将整个过程中将 X 和 Y 轴方向上都缩放 0.5，然后 fromExtendRight 和 fromExtendBottom 设置为 10%，toExtendRight 和 toExtendBottom 设置为 100%，看下是什么效果：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/54c0db7961674c26af3e5573d20e3ef0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=dphTlipFmb773zMuM1YMR8a4M8k%3D)

的确是把 Activity 的边缘像素延伸到屏幕的边缘的感觉。

再把这个 Demo 装到我们的机器，发现 5s 的动画过程中一直都是以下状态：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6b4ae0a6204248ae87436e98e156918f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=WxqQlcwCJ5hxxwpo%2FFnCgSd3J2o%3D)

只看到了 scale 的动画效果，但是没有 extend 的动画效果。

这么一对比，看起来似乎是我们的机器，extend 这个类型的动画压根就没有生效。

3 ExtendAnimation 分析
--------------------

在 aosp 中搜索 fromExtendRight 等关键字，看到使用的地方在 ExtendAnimation：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3792bbf5462149bbb094f7d6b42f3d96~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=yEWmiFeCkBSSRZMbBcDdx8OW9GY%3D)

在 ExtendAnimation 的构造方法中被解析，而 ExtendAnimation 创建的地方在 AnimationUtils.createAnimationFromXml：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e88a820e4d2443e6944979419da97283~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=AbIIak56skkUZyaGzfoquleQtBM%3D)

也很好理解，如果动画的标签是 extend，那么创建 ExtendAnimation 类型的动画。

再看 fromExtendRight 和 mToRightValue 这些属性被解析出来后，都用在什么地方，搜索了一下代码，只有一处，在 ExtendAnimation.initialize：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6acb2b3ece904de68c011a10817c87d4~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=eGoExSOiOPXXshwAc6kiHS9OPcE%3D)

调用堆栈为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/fe43cbb44ac646bfb4f0d177182c00c0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=zGh1UdKG5XVDzyDPjDaFS9%2FOVjw%3D)

用来初始化 ExtendAnimation 的两个 Insets 类型的成员变量 mFromInsets 和 mToInsets。

而成员变量 mFromInsets 和 mToInsets 使用的地方主要是在 ExtendAnimation.applyTransformation 和 ExtendAnimation.hasExtension：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/59f2395e08ed4570ad750bca922baa2b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=%2BkZBqmZ8%2BqC8xEsokdUT%2Fs3DoIk%3D)

applyTransformation 这个方法很熟悉了，Animation 的子类通过实现这个方法，来计算动画过程中特定时间点下的动画要应用的 transformation。

看下调用堆栈为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/bcfe918090614662b4065438cec608e0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=ne4hd9bz1Brt%2BU3Wmx7Rz3TbA4s%3D)

看到关键点在 TransitionAnimationHelper.edgeExtendWindow：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ebc5aa5eabc347ec8dc20b8ebc7a64fc~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=AfMStTv7R%2FnNhWBOJzExB%2B4Q1Oo%3D)

这里看下 edgeBounds 和 extensionRect 的值，我的屏幕是 720 * 1612，基于我们设置的 extend 动画参数：

```
   <extend android:interpolator="@android:interpolator/linear" android:duration="5000" android:startOffset="0" android:fromExtendLeft="0" android:fromExtendTop="0" android:fromExtendRight="10%" android:fromExtendBottom="0" android:toExtendLeft="0" android:toExtendTop="0" android:toExtendRight="100%" android:toExtendBottom="0"/>


```

得到 edgeBounds=Rect(719, 0 - 720, 1612), extensionRect=Rect(0, 0 - 720, 1612)

再看 TransitionAnimationHelper.createExtensionSurface 方法：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/0ccb22f3b3c54aada337eeee5732159c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=xxabUV%2FqbGAa0mKkZfNh7a%2FcqNo%3D)

大致可以理解一下，将整个屏幕截图，然后截取 edgeBounds 对应的区域，然后从起始位置开始，一直拓展到 extensionRect 区域。

回到我们的问题，既然 ExtendAnimation 没有生效，那么大概率是这里的 edgeBuffer 返回了 null，后续继续在 SurfaceFlinger.captureLayers 打印 log，果然，在 SurfaceFlinger 的这里返回了：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c96cca1883ec47e2b36e7e225c80c85d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=WB6bbCQ%2But5mYnnEFfMSswimlHg%3D)

发现截图的进程对应的 App 没有声明该权限，android.permission.CAPTURE_BLACKOUT_CONTENT，所以截图失败。

看了下我们的 SystemUI，是没有声明这个权限的，反编译 pixel 的 SystemUIGoogle，发现：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/36136d00bb974994be053ffb8fe9a677~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=Q8kUa2aSvBPIDeO%2BWVEjaDHF%2FEs%3D)

pixel 的 SystemUI（据 SystemUI 的同事说 pixel 的 SystemUI 不是 aosp 的那个 SystemUI）是添加了这个权限的，所以 pixel 的 SystemUI 可以截图，并且后续正常运行 ExtendAnimation。

另外一提 aosp 的 SystemUI，即 “/frameworks/base/packages/SystemUI/AndroidManifest.xml” 也是没有声明这个权限的，这不是 google 挖坑吗？只给自己 pixel 的 SystemUI 添加这个权限？

最后再用 winscope 看下画红色的是什么玩意：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/778f0bda9ed0493eb26300c162315cca~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749729370&x-signature=Dws2u6VZSybAleYNPN%2FGW4NFJcM%3D)

“Right Edge Extension”下的一个名为 “bbq-wrapper” 的 Layer。
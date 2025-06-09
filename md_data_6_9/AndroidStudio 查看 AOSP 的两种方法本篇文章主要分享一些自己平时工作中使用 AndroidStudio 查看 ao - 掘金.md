> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7235834434645475384)

> 本篇文章主要分享一些自己平时工作中使用 AndroidStudio 查看 aosp 的方法，同时抛砖引玉，希望知道其它便利有效的查看调试方式技巧的大佬们能够不吝赐教，大家互相分享，共同进步。 如果直接用 And

本篇文章主要分享一些自己平时工作中使用 AndroidStudio 查看 aosp 的方法，同时抛砖引玉，希望知道其它便利有效的查看调试方式技巧的大佬们能够不吝赐教，大家互相分享，共同进步。

如果直接用 AndroidStudio 打开 aosp 根目录，那么打开任意一个 Java 类，默认情况可能是这样的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d69dc9ba8e194535b81aed240d34347e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

Java 文件的标签页显示图标为 ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a16efc9917c44e0faa22244e09260248~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) ，表示 “Java class located out of the source root”，并且其内部的成员变量之类的也没有被语法高亮。

经过我们配置后，被识别后的 Java 文件被效果如下图所示：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/958567cc415e4d0fb8dcde39a8d907bb~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

Java 文件的标签页显示图标为 ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc0c6ef86d8d4ce1b2b792b08fb5e32f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) ，且成员变量也被高亮。

更重要的是，此时代码内部已经建立起了索引，比如可以进行代码补全（这里的代码不仅是补全定义在当前类中的域，也可以补全父类的域，以及其它类的域）：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7451cb4be9404236ba897fe5dff384e0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

或查看某个域被哪些类调用了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e52b4e607424df1a624d1b1a5369eea~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

以及代码跳转等一系列便捷功能。

1.1 设置步骤
--------

第一种方法应该很多人都知道，我从接触 Android 开始的很长一段时间用的都是这种方法，该方法需要一套已经编译过的 AOSP。

1）、首先确保已经执行过：

```
soruce build/envsetup.sh


```

等命令加载编译所需的环境变量。

2）、接着执行：

```
mmm development/tools/idegen/ -j16


```

编译成功后，会输出：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a3e2b91aca54a42a0c4b7193726ce17~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

我这里编译之前设置了生成目录的环境变量为 out_sys，所以生成文件在 out_sys。

3）、此时可以执行命令：

```
./development/tools/idegen/idegen.sh


```

如果你的生成目录也和我一样在 out_sys 的话，可能需要新建一个 out/host/linux-x86/framework / 目录，然后将 idegen.sh 复制过去：

```
cp out_sys/host/linux-x86/framework/idegen.jar out/host/linux-x86/framework/


```

当看到有类似输出：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3831c32142e3437f8c0e5ce171d16701~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

的时候便可以了，最终会在 aosp 根目录生成两个文件，android.ipr，android.iml：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b50b1e44d9ee4a6a8c664651d6bc26d4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

4）、通过 Android 打开这个 android.ipr。由于是第一次打开源码会为所有模块建立索引，所以耗时非常久，相当长的时间内 AndroidStudio 卡的都不能使用，这是我放弃使用这种方式查看 aosp 的主要原因。实不相瞒，我个人已经很久没用过这种方式了，这次是为了写这篇文章所以我又试了一次这种方法，怎么说呢，熟悉的感觉又回来了...... 慢的让人发指，有的时候你甚至分不清是到底是真的在建索引还是单纯卡死了。

1.2 优缺点
-------

由于卡的时间太久不想等了，所以后面的步骤就不演示了，其实也没啥内容了，说一下这种查看方法下的可能有用的技巧：

*   将你用不到的代码目录设置为 excluded，这个操作在此种方式下似乎用处不大。
*   在 android.iml 中将你用不到的代码目录，从 sourceFolder 改为 excludeFolder，这个方法很有用，不论是在建立索引的时候，还是在你后续查看 aosp 的时候，都可以帮你过滤掉很多无关代码。
*   将 Project Structure -> Modules -> Dependencies 下的 jar 包啥的都删掉，避免代码跳转的时候跳到其它乱七八糟的地方。

这种查看 aosp 的方式，优点就是一次性为所有模块建立索引，同时也是缺点，第一次加载因为要建立索引所以巨慢，后续再次打开虽然比第一次要快很多，但也是相对的，我个人觉得花的时间还是比较久。

另外之前说的修改 android.iml 的方法，的确是一个好方法，但是也不是没有缺点，比如配置这个可能比较麻烦，虽然这个配置操作是一劳永逸的，但是如果你换了另一个项目的代码看，那就又需要重新配置这个项目的 android.iml，并且又需要经历一次巨 TM 久的建索引的熬人环节。我当时的做法是将 android.ipr 和 android.iml 进行了复用，比如我在项目 A 上配置过了这两个文件，那么当我重下了一套项目 B 的代码，我可以直接将项目 A 的 android.ipr 和 android.iml 复制到项目 B 中复用这两个文件。好处是的确省下了不少时间精力，缺点是如果两个项目中你想查看的文件的名字或所在目录碰巧不一样的话（比如 Android 平台升级了），可能比较麻烦，以及可能还有一些我没遇到过的未知问题。

如此种种，让我迫切希望寻找另一种查看 aosp 的方式。

2.1 设置步骤
--------

第一种方法的缺点我已经吐槽了很多了，但是还有一点也是不能忽视的，就是你需要一套已经编译过的 aosp，很多时候我可能只拉了 aosp 中某一个库的代码，比如 frameworks/base 这个库，那么我就不能建立内部索引进行查看了吗。

现在的答案是可以，其实也很简单。不过这里还是拿整套 aosp 举例，操作是一样的：

1）、比如我现在下载了一套项目代码，接着直接将该项目的根目录在 AndroidStudio 中打开，初始所有的目录都是这样的：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf91ccfc6750471783bf440a051f8df4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

2）、此时关闭掉右下角正在建立索引的操作：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91e308b040454506bd1604d161980144~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

3）、然后先将所有目录全部设置为 Excluded：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42ca40d808f74475b1b731dbdef3b6d1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

目录被设置为 Excluded 后就像这样：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/51858e79c2f74a46af12cb7500d4e05f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这一步也可以不做，我这么做只是希望一开始的时候就把无关模块排除掉，方便后续建索引和查看代码。

4）、然后重新再打开 AndroidStudio，这个时候你会发现建索引的步骤没了，因为所有目录都被排除掉了。

5）、然后你想要查看哪些代码，就把这些代码所在的 java 目录或者 src 目录标记为 Sources Root（这一步应该是基于 IntelliJ IDEA 的配置原理，但是我还没找到具体的理论支持内容），比如我经常看 WMS 相关的内容，在

```
frameworks\base\services\core\java\com\android\server\wm\


```

包中，那么我就可以把

```
frameworks\base\services\core\java


```

这个目录设置为 Sources Root，就像这样：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e954ec0dc58e48f598386783f00aa25a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

目录被设置为 Sources Root 后，结果为：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8abe6ff0b1634372924ee1cf1f43e15d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

此时便可以为这个目录重新建立索引了，比如我在 WindowContainer.java 这个类中，新建一个 test 方法，看看 this 都有哪些方法可以调用：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/97b275013b40452585261863a2344a18~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

能看到代码补全的功能。

查看 isDescendantOf 这个成员方法都在哪些类里被调用了，也可以：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ddadf204781425e9f65c67fd65a6a70~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

其它的就不过多介绍了，我个人目前使用的就是这种查看 aosp 的方式。

2.2 跳转到 SDK 的特殊情况
-----------------

这里说一点可能会遇到的情况，即代码跳转可能会调转到 SDK，而不是 aosp，比如我通过 Ctrl + 鼠标左键想要跳转到 Rect 这个类里去，发现跳转到了 SDK 里，而不是 aosp 中的 Rect.java：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3a51608a813402c9f42db3a627ae611~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这个时候需要将 Project Structure -> Modules -> Dependencies 中的 SDK 依赖：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/04efb15cd8d44e38baad441df98f5d8b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 更换为本地的 JDK 包：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bcd5bd61ef7244a5b17a998c73e8709d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

选择完就像这样：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c325bd264eda49839abd91366943ebf6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

此时再重新尝试跳转到 Rect，就可以了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f88cb38f740d4899a06cdb65afacf2d2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

如果不可以，可能是因为 Rect.java 位置为：

```
frameworks\base\graphics\java\android\graphics\Rect.java


```

你需要将 Rect.java 所在目录的的 java 目录：

```
frameworks\base\graphics\java


```

设置为 Sources Root：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77f6bfbebd484ee49deedbfe906d96d4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

[Content roots | IntelliJ IDEA Documentation (jetbrains.com)](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jetbrains.com%2Fhelp%2Fidea%2Fcontent-roots.html%23configure-folders "https://www.jetbrains.com/help/idea/content-roots.html#configure-folders")

Configure folder structure
--------------------------

### Folder categories

Folders within a content root can be assigned to several categories.

*   ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb7d67f2a60245c28f479e15979cc1b1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) Sources
    
    This folder contains production code that should be compiled.
    

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d805077a5e44497b60a102d9ed0c90c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

*   ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f9a5610159b49368c78daf7723ea843~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) Generated Sources
    
    The IDE considers that files in the Generated Sources folder are generated automatically rather than written manually, and can be regenerated.
    
*   ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e48a551621544216abaeed13468f5ec6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) Test Sources
    
    These folders keep code related to testing separately from production code. Compilation results for sources and test sources are normally placed into different folders.
    
*   ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2e3fcef3723497ab1f0d012b3abb1bf~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) Generated Test Sources
    
    The IDE considers that files in this folder are generated automatically rather than written manually, and can be regenerated.
    
*   ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c5b6084b3074e83bb5a1e11856910b3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) Resources
    
    (Java only) Resource files used in your application: images, configuration XML and properties files, and so on. During the build process, resource files are copied to the output folder as is by default. You can [Change the output path](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jetbrains.com%2Fhelp%2Fidea%2Fcontent-roots.html%23specify_output_path "https://www.jetbrains.com/help/idea/content-roots.html#specify_output_path") for resource files in your project.
    
    Similarly to sources, you can specify that your resources are generated. You can also specify which folder within the output folder your resources should be copied to.
    
*   ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb84f2d087bd4d3aafad9def31df0e6c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) Test Resources
    
    These folders are for resource files associated with your test sources.
    
*   ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/283a86138d7f43d1b4fe40f5a4bd099d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) Excluded
    
    Files in excluded folders are ignored by code completion, navigation and inspection. That is why, when you exclude a folder that you don't need at the moment, you can increase the IDE performance. Normally, compilation output folders are marked as excluded.
    
    Apart from excluding the entire folders, you can also [exclude specific files](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jetbrains.com%2Fhelp%2Fidea%2Fcontent-roots.html%23exclude-files "https://www.jetbrains.com/help/idea/content-roots.html#exclude-files").
    

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/49302b2e6598475f943746ca328cb2be~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### Configure folder categories

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4ecec508e6a4ce4a9c7695bb123225f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)
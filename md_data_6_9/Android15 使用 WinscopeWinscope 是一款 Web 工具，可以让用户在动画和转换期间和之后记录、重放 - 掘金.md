> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7426319406358200355)

> Winscope 是一款 Web 工具，可以让用户在动画和转换期间和之后记录、重放和分析多个系统服务的状态。

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/75d381d89042418ca3bd05ea38e76672~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=lwTN8Wld1KL8UccoJMKKLC4KogM%3D)

[使用 Winscope 跟踪窗口转换  |  Android Open Source Project (google.cn)](https://link.juejin.cn/?target=https%3A%2F%2Fsource.android.google.cn%2Fdocs%2Fcore%2Fgraphics%2Ftracing-win-transitions%3Fhl%3Dzh-cn%23analyze-traces "https://source.android.google.cn/docs/core/graphics/tracing-win-transitions?hl=zh-cn#analyze-traces")

Winscope 是一款 Web 工具，可以让用户在动画和转换期间和之后记录、重放和分析多个系统服务的状态。Winscope 将所有相关的系统服务状态记录在一个跟踪文件中。使用带有跟踪文件的 Winscope 界面，您可以通过重放、单步执行和调试转换来针对每个动画帧检查这些服务的状态（无论是否有屏幕录制）。

说的通俗一点，我感觉 Winscope 就是在一段时间内的每一帧，都 dump 一下手机中的某些信息，然后将这些信息收集起来并且以图形的方式展示，如：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6917ffa490f147ba91e4ec7529ba37c7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=oUWyJQtvk%2FaZq0bLZn8QHQD9EXc%3D)

本文主要记录一下我本地在 Android15 平台上使用 Winscope 的情况，这个跟个人使用的平台以及环境都有很大的关系，不一定说按照我的做法就一定成功，我遇到的问题你可能没有遇到，同样你遇到的问题我可能没有遇到。

1 Android15 以前使用 Winscope
-------------------------

1、在开发者模式中，“Quick settings developer tiles“中，开启”Winscope trace“：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ec3d3460817a416fae3ca84f38209c71~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=muvhizIpaB6Ga%2F%2Bg%2Fyh%2FGI7cGKo%3D)

2、执行复现问题的操作。

3、操作完后的 winscope 文件在手机保存到了 data/misc/wmtrace，我们一般关注比较多的是保存了 SurfaceFlinger 的信息，“layers_trace.winscope” 文件：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3b0d7f57a6494b358dfd6b7f5bc5b40e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=FUY24kEnNZRJPWpPggUQv4R%2FuRQ%3D)

4、使用 “prebuilts/misc/common/winscope/winscope.html” 查看相关信息：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/612c7205e4db4ae0ac49345f9d499818~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=s4HYIxSrH9uRou0KEOv4iVuJQV4%3D)

#### 通过 adb 命令捕获 SurfaceFlinger 跟踪记录

要记录 SurfaceFlinger 的跟踪情况，请执行以下操作：

1、启用跟踪：

```
adb shell su root service call SurfaceFlinger 1025 i32 1


```

2、停用跟踪：

```
adb shell su root service call SurfaceFlinger 1025 i32 0


```

3、获取跟踪文件：

```
adb pull /data/misc/wmtrace/layers_trace.winscope layers_trace.winscope


```

4、拖动到 Winscope 来查看 layers_trace.winscope 文件：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f3374b3cbd5f4ea38e1b43185d757fee~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=AQsJrIzHzh%2BUSWa2CozaCp7MrGE%3D)

这里展示一下从 Launcher 启动 google Files 的过程。

2 Android15 变动
--------------

然而这些在 Android15 上已经行不通了，如果我们使用 adb 命令：

```
adb shell su root service call SurfaceFlinger 1025 i32 1


```

去抓取跟踪信息，会报错：

```
Result: Parcel(Error: 0xfffffffffffffffe "No such file or directory")


```

原因则是 Android15 中 SurfaceFlinger 这块的逻辑已经被移除了：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/191e576aa63b4b3c92ace3ed19452d65~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=FNCY8DoENnhzoBEEvqriyOy0RWs%3D)

提示需要使用 perfetto 来开启 layer 追踪。

再根据 google 的官网介绍：

[使用 Winscope 跟踪窗口转换  |  Android Open Source Project (google.cn)](https://link.juejin.cn/?target=https%3A%2F%2Fsource.android.google.cn%2Fdocs%2Fcore%2Fgraphics%2Ftracing-win-transitions%3Fhl%3Dzh-cn%23capture-traces-winscope "https://source.android.google.cn/docs/core/graphics/tracing-win-transitions?hl=zh-cn#capture-traces-winscope")

最新的情况是 “Quick settings developer tiles“下移除了”Winscope trace“，移动到了“System Tracing” 中：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a5804cf5a8d34cd5a4f47bbcfb939f66~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=RYfZ4bjllG8kpsQwcrFx1e5T6GY%3D)

如我的手机截图：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/387546d0dee74c9d934de10511d36b2d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=usXeOu5a6vduDkvu5LwAeavRJnQ%3D)

那现在的 Android15 要如何使用 Winscope 工具呢，毕竟这个工具还挺好用的。

3 Android15 使用 winscope
-----------------------

首先根据 google 官网的指导去调试肯定是没错的，但是官网有些地方说的并不清楚，实际上去操作也遇到过很多问题，本文也参考了：

[android 14 版本的 winscope 编译使用 - 手把手教你编译成功不报错_android wincope-CSDN 博客](https://link.juejin.cn/?target=https%3A%2F%2Fblog.csdn.net%2Flearnframework%2Farticle%2Fdetails%2F141681248 "https://blog.csdn.net/learnframework/article/details/141681248")

另外我工作用的机器是 windows+WSL，用的代码下载在了服务器，然后挂载到本地来查看的，实际操作起来一堆问题，虽然最后还是在本地弄好了，但这个是后话了。

最开始，为了实验一下按照官方的教程是否能够成功，我用的是公司机房的电脑，ubuntu 系统，并且能够连接外网，先体验一下简单难度，再挑战地狱难度。

#### 3.1 下载 Android15 源码

```
repo init -u https://android.googlesource.com/platform/manifest -b android-15.0.0_r1


```

这里只需要下载几个依赖的库就好了：

```
repo sync development external/protobuf external/perfetto frameworks/base frameworks/libs/systemui frameworks/native frameworks/proto_logging platform_testing prebuilts/misc


```

不同的 Android 版本不一样，具体是哪些库应该要看 “development/tools/winscope/protos/build.js” 的具体内容。

#### 3.2 导航到 Winscope 文件夹

```
cd development/tools/winscope


```

#### 3.3 安装 npm

这里推荐用 nvm 安装，nvm 全名 node.js version management，是一个 nodejs 的版本管理工具：

```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
source ~/.bashrc 


```

nvm 安装的 node 版本比通过”apt-get“的方式更新一点：

```
nvm install node


```

查看版本号为：

```
ukynho@user:~$ node -v
v22.9.0
ukynho@user:~$ npm -v
10.8.3


```

如果使用”apt-get“安装 npm，我本地安装的 node 版本是 10.19.0，低版本的 node 到后续的步骤可能会出错。

另外我这边查看 node 版本的时候，也遇到了 GLIBC 版本过低的错误，这一点更新到后面的解决问题的章节。

#### 3.4 安装依赖项

使用以下命令安装依赖项：

```
npm install


```

这一步也 OK，没遇到什么问题：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d53b9740c8a94b05b7244da88ad39e79~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=hn1GkzGnOzzbksB6Cvg5LLsLop4%3D)

如需查看可用命令的列表，请运行以下命令： npm run

通过 npm run 命令查看的应该是 package.json 下的命令，我们重点关注 build 的这几个，"build:trace_processor"、"build:prod"" 和 "build:protos：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d714ba4023774ee29a1d8ec3b1ef7174~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=J3N8LHtidNoA9%2F5OhYP8P%2F%2FPQlc%3D)

#### 3.5 构建所有生产和测试目标

使用以下命令构建所有生产和测试目标：

```
npm run build:prod


```

实际上执行的也就是：

```
> npm run build:trace_processor && npm run build:protos && rm -rf dist/prod/ && webpack --config webpack.config.prod.js --progress && cp deps_build/trace_processor/to_be_served/* src/adb/winscope_proxy.py src/logo_light_mode.svg src/logo_dark_mode.svg src/viewers/components/rects/cube_full_shade.svg src/viewers/components/rects/cube_partial_shade.svg src/app/components/trackpad_right_click.svg src/app/components/trackpad_vertical_scroll.svg src/app/components/trackpad_horizontal_scroll.svg dist/prod/


```

我本地的情况是，当执行到以下步骤：

```
> winscope@0.0.0 build:protos /local/sdb/android-15/development/tools/winscope
> node protos/build.js


```

报错：

```
(node:1753817) UnhandledPromiseRejectionWarning: TypeError: outSubdir.replaceAll is not a function
    at buildProtos (/local/sdb/android-15/development/tools/winscope/protos/build.js:106:32)
    at build (/local/sdb/android-15/development/tools/winscope/protos/build.js:30:9)
(node:1753817) UnhandledPromiseRejectionWarning: Unhandled promise rejection. This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). (rejection id: 18)    


```

还有：

```
ERROR in ./src/parsers/input/perfetto/abstract_input_event_parser.ts 16:440-483
Module not found: Error: Can't resolve 'protos/input/latest/json' in '/local/sdb/android-15/development/tools/winscope/src/parsers/input/perfetto'


```

如以下截图：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b446392ec5324a6f8febe6082be70367~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=t8n7ooG0JDDs4HzY72dnNQR8T3M%3D)

这个报错问 chatGPT：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/0d3b0147bec1489c8b34b0623cf310d4~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=J4YtFX0FgPzAuLJL3Bgf0fQbGdw%3D)

原因就是第 3 步安装 npm 时提到的，由于我最初使用 apt-get 安装的 node，版本号过低：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7646514c44a445f58a4864cc50eefec7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=uQoBboahIJdvPZM1rMLHJ%2FEr3JQ%3D)

只有 10.19.0，而这里要求的版本要在 15 以上，后续换 nvm 安装最新版本的 node 就可以 pass 了。

最终成功，显示：

webpack 5.91.0 compiled with 3 warnings in 35078ms

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/508042e42087470eb51ea134942cfb40~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=FudKpJCRVKSZSNPXn3zTWLAIXpk%3D)

生成 “dist/prod” 目录：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/987937ebe1424850ab51088142ed7784~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=uJxvon8lgoikmQ3r%2BfOGqW4TCV8%3D)

#### 3.6 运行 Winscope

使用以下命令运行 Winscope：

```
npm run start


```

这一步也没有遇到什么问题，成功：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/25734ab40cce4a8cbf5ba0407e5d6c61~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=VIxx4sq%2FRxhRyb4%2FY7nwhBjbQCA%3D)

然后 Winscope 页面在浏览器中打开：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e89e385aa00c494eb239d80a67482db8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=F2RCBBCEQJJUZ2Hi%2B3zUi03FY6U%3D)

4 捕获追踪记录
--------

#### 4.1 在设备上捕获跟踪记录

这里是官网的介绍：

在针对动画问题提交错误时，在设备上捕获跟踪记录以收集数据。所有界面跟踪记录都是通过此方法记录的，因为无法自定义配置。

在 Android 设备上：

1、[启用开发者选项](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fstudio%2Fdebug%2Fdev-options%3Fhl%3Dzh-cn%23enable "https://developer.android.google.cn/studio/debug/dev-options?hl=zh-cn#enable")。

2、选择开发者选项下的 System Tracing。

3、启用收集 Winscope 跟踪记录。

4、在其他下：

1). 启用在错误报告中附加录制内容。

2). 启用显示 “快捷设置” 功能块。

5、 导航到您需要重现错误的位置。

6、如需开始记录，请打开 “快捷设置”，然后选择录制跟踪记录：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/5fa7bc2a822d4f679a03a31c6081bee8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=aWeSGfOjWbM661YXcZ8YucBwx0s%3D)

7、仅执行重现错误所需的步骤。

8、如需停止捕获，请打开 “快捷设置”，然后选择停止跟踪。

9、使用所列选项之一共享捕获的日志，如 Gmail、云端硬盘或 BetterBug。

我本地操作下来，看到最终的 “perfetto-trace” 文件是保存在了 “data/local/traces” 目录下，如：

```
data/local/traces/trace-MT6835-AP3A.240617.008-2024-10-15-10-33-52.perfetto-trace


```

另外我没看到对应的视频，我个人觉得有 “perfetto-trace” 搭配视频就最好了，大部分情况下 wm 或者 ime 的信息啥的我不太需要，比如 Transitions 啥的一般看 log 就够了，当然特殊情况下肯定是有参考价值的。

#### 4.2 通过 Winscope 捕获跟踪记录

这种方式虽然有点麻烦，但是能抓的信息却是最全，如果情况允许的，我最推荐这种。

1、执行以下命令：

```
python3 development/tools/winscope/src/adb/winscope_proxy.py


```

生成了 token：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/5e724dea0b914f69a2685e63edd16e64~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=XnQL%2BrXl6yqhzJFzD9MbjZdAJ9Y%3D)

2、在 Collect Traces（收集跟踪记录）界面上，点击 ADB Proxy（ADB 代理）：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/772c80a6f3f84d179eb970228f8714a6~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=krn1cAOLdGKODEY0fTHO3XdnnjE%3D)

变成了：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/beaefd72e2584191bc78cf4b67f4383a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=nE%2F12J2euxKcJoDziqGy6l49lE4%3D)

输入 token，然后点击 “Connect”，选择对应的机器：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/5f8c2278e1044d91a908ec2c17c78c0a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=7Lb%2FCEryfcWLx0PFTauUIP9UEVM%3D)

然后点击 “Start trace”，就可以开始记录了。

最后你也可以把所有的文件下载下来查看，下载后是一个压缩包，解压后得到大概是这样的：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/8e095a587f854dbeb1e1487dfac28610~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=7S9DrIXBGHv8K5lu23EcGiNJV0o%3D)

拖到 Winscope 的那个网页里面就可以查看了。

另外说一下这个生成状态转储文件的功能：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a7d423d88ec0401589c56f3c90b94277~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=PhMHKpEo3l9bqcYugkDoN7xi8ZA%3D)

我个人觉得也还不错，Android15 上用 “dumpsys SurfaceFlinger” 看不到每一个 Layer 的具体信息了（也许还能看，我现在还没去研究），但是通过这里还是能看到：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1216ed9794eb4fe2ba75f917d5d0bfeb~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=soLTBhkYpK36Kfv88k7A90SFZg4%3D)

#### 4.3 通过 adb 命令捕获跟踪记录

这里重点看下 SurfaceFlinger 的部分，首先是官网的介绍。

SurfaceFlinger 层跟踪使用 Perfetto 跟踪记录进行捕获。如需了解配置信息，请参阅[跟踪配置](https://link.juejin.cn/?target=https%3A%2F%2Fperfetto.dev%2Fdocs%2Fconcepts%2Fconfig "https://perfetto.dev/docs/concepts/config")。

请参阅以下有关 SurfaceFlinger 层跟踪配置的示例：

```
unique_session_name: "surfaceflinger_layers_active"
buffers: {
    size_kb: 63488
    fill_policy: RING_BUFFER
}
data_sources: {
    config {
        name: "android.surfaceflinger.layers"
        surfaceflinger_layers_config: {
            mode: MODE_ACTIVE
            trace_flags: TRACE_FLAG_INPUT
            trace_flags: TRACE_FLAG_COMPOSITION
            trace_flags: TRACE_FLAG_HWC
            trace_flags: TRACE_FLAG_BUFFERS
            trace_flags: TRACE_FLAG_VIRTUAL_DISPLAYS
       }
   }
}


```

请参阅以下示例命令，以生成 SurfaceFlinger 层的跟踪记录：

```
adb shell -t perfetto \
    -c - --txt \
    -o /data/misc/perfetto-traces/trace \


```

我看了这个是比较懵逼的，最后参考了：

[Trace configuration - Perfetto Tracing Docs](https://link.juejin.cn/?target=https%3A%2F%2Fperfetto.dev%2Fdocs%2Fconcepts%2Fconfig "https://perfetto.dev/docs/concepts/config")

才稍微弄懂了一点，主要是看关于 Android 的介绍：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a62c48aca3b343b0bfbe722dad91ecae~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=LakKweYSp2%2FO5JuJ3wKYGrj%2BXCs%3D)

接着是我本地的操作。

1、首先将以上配置写到一个文件中：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6cb66c5d02c04d98aec05eb8805c38ae~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=65GsWGLVTdu72MbZAaeKdFHdu1o%3D)

命名为 config.pbtx。

2、使用以下命令：

```
cat config.pbtx | adb shell perfetto -c - --txt -o /data/misc/perfetto-traces/test.perfetto-trace


```

然后在适当的时候断掉：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c705064748044f128e6af68f845af7c0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=6NEuP8pZKbokSbeZ31o5DGMlT0g%3D)

最终在 “/data/misc/perfetto-traces/” 目录生成名为 “test” 的 perfetto-trace 文件：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/83d78ce9e0c84f72819630fe1577650d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=G8%2F8iAiiUV8xanNvs5Md1FubGYk%3D)

但是这里的生成的文件大小是 0，所以肯定是哪里有点问题的，我现在还不清楚。

后续看了下 perfetto 官网的介绍，仿照着在配置文件中额外增加抓取时间，“duration_ms: 10000”，如：

```
duration_ms: 10000

unique_session_name: "surfaceflinger_layers_active"
buffers: {
    size_kb: 63488
    fill_policy: RING_BUFFER
}
data_sources: {
    config {
        name: "android.surfaceflinger.layers"
        surfaceflinger_layers_config: {
            mode: MODE_ACTIVE
            trace_flags: TRACE_FLAG_INPUT
            trace_flags: TRACE_FLAG_COMPOSITION
            trace_flags: TRACE_FLAG_HWC
            trace_flags: TRACE_FLAG_BUFFERS
            trace_flags: TRACE_FLAG_VIRTUAL_DISPLAYS
        }
    }
}


```

然后果然就可以了：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/cac90351340047dda290461b3ea4a31c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=aJkjoIztaMZJJFu8mo52W6wIxPw%3D)

拖动到 Winscope 中可以查看：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/779ea391045245d0b9773dd5c22ac515~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=9Rb5kua9XRC2qqF5cWPZIznH%2BSI%3D)

3、最后如 perfetto 官网介绍的，从 Android12 开始，“/data/misc/perfetto-configs” 可以用来存储配置文件，你也可以把自己的配置文件 push 到这个目录下，然后直接用命令：

```
adb shell -t perfetto -c data/misc/perfetto-configs/config.pbtx --txt -o /data/misc/perfetto-traces/test2.perfetto-trace


```

来生成 perfetto-trace 文件：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/183f309a0da047feb53a8e39fca65b1f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=kg%2F5p4gt17A4vW4Awckwz8YlMtQ%3D)

我个人觉得还是自己本地写好一个配置文件，然后执行：

```
cat config.pbtx | adb shell perfetto -c - --txt -o /data/misc/perfetto-traces/test.perfetto-trace


```

这种方式最简单快捷。

5 分析跟踪记录
--------

这部分官网已经讲的很详细了，我这边没有遇到什么别的问题。

6 问题收集
------

如我之前所说，最开始我用的是公司机房的 ubuntu 系统的电脑，并且能够连接外网，所以遇到的问题还算少的。

但是我最终的目的肯定是在自己自用的电脑上用的，由于我本地是 windows 系统 + WSL，不能连外网，并且代码是下载在服务器然后挂载到本地的，所以实际上操作的时候遇到了很多其它的问题。

#### 6.1 下载源码

由于不能科学上网，这里使用清华的镜像下载 Android15 源码：

```
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-15.0.0_r1


```

#### 6.2 执行”npm run build:prod“报错

卡在：

```
fatal: unable to access 'https://chromium.googlesource.com/external/github.com/llvm/llvm-project/libcxx.git/': Failed to connect to chromium.googlesource.com port 443 after 134229 ms: Connection timed out


```

原因为无法连接到”[chromium.googlesource.com/external/gi…](https://link.juejin.cn/?target=https%3A%2F%2Fchromium.googlesource.com%2Fexternal%2Fgithub.com%2Fllvm%2Fllvm-project%2Flibcxx.git%2F%25E2%2580%259C "https://chromium.googlesource.com/external/github.com/llvm/llvm-project/libcxx.git/%E2%80%9C")

这一步我是真没辙，没有清华可用的镜像源替换，github 上担心版本太旧可能会有问题，最后只能用手机连了公司的翻墙 wifi 然后分享给电脑来下载。

最终本地也成功：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/26df742dab1d40a2918c523e264d8e20~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=lS5g9bPXKLL%2FaErHDwxUZaq%2BPO0%3D)

#### 6.3 执行”npm run start“报错

报错为：

```
WARNING in ./node_modules/three/src/Three.js
There are multiple modules with names that only differ in casing.
This can lead to unexpected behavior when compiling on a filesystem with other case-semantic.
Use equal casing. Compare these module identifiers:
* javascript/esm|/mnt/d/android-15-tsinghua/development/tools/winscope/node_modules/three/src/Three.js
    Used by 1 module(s), i. e.
    /mnt/d/android-15-tsinghua/development/tools/winscope/node_modules/@ephesoft/webpack.istanbul.loader/index.js??ruleSet[1].rules[0]!/mnt/d/android-15-tsinghua/development/tools/winscope/node_modules/ts-loader/index.js!/mnt/d/android-15-tsinghua/development/tools/winscope/node_modules/angular2-template-loader/index.js!/mnt/d/android-15-tsinghua/development/tools/winscope/src/app/components/timeline/mini-timeline/drawer/draggable_canvas_object_impl.ts
* javascript/esm|/mnt/d/android-15-tsinghua/development/tools/winscope/node_modules/three/src/three.js
    Used by 1 module(s), i. e.
    javascript/esm|/mnt/d/android-15-tsinghua/development/tools/winscope/node_modules/three/examples/jsm/renderers/CSS2DRenderer.js


```

如截图：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a0b1c8f7c90446ccbd6e23be6b7d7cb6~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=83ZX8CkvS137jKeISxgq8I%2Bo2vM%3D)

这个问了 chatGPT：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6ee31e1869f74a24b34bd6556a627c30~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=oCjXX5%2FjucX0k%2BJP3GgW99D9Rs8%3D)

说实话我对这块也不太懂，chatGPT 说是大小写的问题，网上查资料也说是”linux 下是严格区分大小写，而 windows 不区分 “，因此做了诸多尝试，最终发现将”development/tools/winscope/node_modules/three/examples/jsm/renderers/CSS2DRenderer.js“文件中的这里：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/71d96c431ca94b10bc8803c61c25ad37~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=qNWexAEGGZYxhBWx56qNyZ1dWEk%3D)

”three“改为大写的”Three“可以成功：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/9d7a564675144aeaa9fb580512ac6673~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=Y%2FNz3xaE%2FJJpbw3MwSlMioQmrDk%3D)

最终在我自用的电脑上也可以使用了：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1ed6447f3e6442b88a1eefcf69312c35~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=SYVz3CHJfHWzJatBSQy9WvO6I7w%3D)

脑海中缓缓响起了”We are the champions my friends. And we'll keep on fighting......“

#### 6.4 直接使用”index.html“行吗

根据我们在 Android15 之前的平台上的经验，是可以直接把源码中”prebuilts/misc/common/winscope/winscope.html“这个现成的 Winscope 工具拷贝到其它地方，然后直接打开来使用的，那现在对”development/tools/winscope/dist/prod/index.html“还可以用同样的操作吗？

我直接将机房生成的”dist/prod“目录拷到我本地，然后直接打开”development/tools/winscope/dist/prod/index.html“后，同样是这个界面：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1e8a7c3636f34160ab6916a72f9eadf4~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=swFJh39pgAVrOlg%2BmE80yQIbTv4%3D)

然后我本地试了一下，大部分的文件都能拖进去解析：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b8b7a9da29b0486498550b25d6a39fc2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=P%2F7shTijYVh5VtLj3H5lCMUzfcE%3D)

但是唯独”trace.perfetto-trace“不行！

说实话我用 Winscope 主要就是看 SurfaceFlinger 的信息的，你哪怕功能再多，SurfaceFlinger 的信息看不了对我来说等于没用，目前我本地是行不通的。

~不知道有没有网上的各位大佬有没有办法，如果可以的话，就能省下很多工作了。~

该问题别的博主已经解决： [aosp15 上 winscope 离线 html 如何使用？_winscope.html-CSDN 博客](https://link.juejin.cn/?target=https%3A%2F%2Fblog.csdn.net%2Flearnframework%2Farticle%2Fdetails%2F144384808%3Fspm%3D1001.2014.3001.5501 "https://blog.csdn.net/learnframework/article/details/144384808?spm=1001.2014.3001.5501")

#### 6.5 直接将别的机器的源码拷到本地可以吗

既然直接将”index.html“拿过来不行，那将别的机器中配置好的源码拷贝过来可以用吗？比如我在机器 A 上配置好了，然后我把机器 A 上的源码直接拷贝到机器 B 上，然后直接在 “development/tools/winscope” 目录下执行最后一步”npm run start“可以吗？毕竟能少执行一步就少执行一步......

实际上是可行的，但是根据我操作的情况，将机房中的机器 A 配置好的源码拷贝到机房中的机器 B（同样的 Ubuntu + 连接外网）下是行得通的，但是拷贝到我自用的机器（Windows+WSL + 不能连外网）就不行，具体现象和直接打开 “index.html” 是一样的，没办法查看 perfetto-trace 文件...... 这个我也不知道原因，只能感叹时运不齐，命途多舛。

#### 6.6 GLIBC 版本过低

接上节，我将机器 A 配置好的源码拷贝到机器 B 上，然后在机器 B 上，当我使用”node -v“查看 node 版本的时候，遇到了 GLIBC 版本过低的错误：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/cfa39e3ef22f4ab18e04e3ab2878b12c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=PE6%2B2lshFfgB2o%2FjrPdRDxhtNwQ%3D)

```
node: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.28' not found (required by node)


```

使用”ldd --version“查看，我本地的是 2.27，这里至少要 2.28：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ae92d545c51049b9bef54fa6f404c659~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=tVDq%2BaY1pGFAsJ%2FxWjDGsd99e%2Fw%3D)

这里我查找网上资料，最终使用”sudo apt-get install libc6“命令更新到版本 2.35，然后问题解决：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/afc7a50a0b304313894bc78ba544e0f3~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=Rtuo59DDL5HCewFvxIT0ZttUpLk%3D)

#### 6.7 “OSError: [Errno 98] Address already in use”

有的时候执行 “winscope_proxy.py”：

```
python3 winscope_proxy.py


```

会提示：

```
“OSError: [Errno 98] Address already in use”


```

在网上查了一下，解决方案是执行：

```
netstat -tunlp


```

然后 kill 掉相应的进程：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a1cbd77354164ed9bbf741820c4dc866~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=vjvrLEWtnx4Y8RenzZVBvzAWKO0%3D)

就可以了。

#### 6.8 listen EADDRINUSE: address already in use :::8080

有的时候执行 “npm run start”，会遇到端口被占用的情况：

```
listen EADDRINUSE: address already in use :::8080


```

如截图：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c2e9ab2c8590413c9739eaae2097cc5d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=XmWB6jgQLvGLfdLGPD7CxDhRlRc%3D)

那么执行：

```
sudo lsof -i :<端口号>


```

然后 kill 掉相关进程就好了，如我这里的是 8080 端口号被占了：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/9b2a633a8157456bae3d24998b6e66d2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=4p3or50BGgwImp1Ub1yNgP0rjeY%3D)

#### 6.9 使用 Winscope 工具抓取追踪记录时报错

点击 “End trace 后”，报错信息为：

```
2024-10-16 10:03:14,019 - ADBProxy - ERROR - Internal error: TimeoutExpired(['adb', '-s', 'HAJV9TNJ9DRCBIWK', 'shell'], 5)


```

如截图：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/85fb3d1dbf47467196713e3fc489d8a8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=w9uMzA5Ax%2F2Vdu5WLfyMF17pnZU%3D)

这个问题我还没解决，可能跟我本地是 Windows+WSL 有关，我用 ubuntu 系统还没遇到过这个问题，不知道有没有大佬知道如何解决。

7 小试牛刀 —— 闪屏问题分析
----------------

现在你已经掌握了 Winscope 的使用方法了，可以运用到实际来解决问题了，这里看一个启动头条闪屏的问题。

### 7.1 问题描述

非冷启动头条的情况，偶尔会出现闪屏的情况，如：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e2c44d3d3ce144a6b05ac91c111f7fd1~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=1XPJqjKZzN%2Fxc6UQpBZck%2FBYS%2FA%3D)

看这个现象：

1、是 Starting Window 消失又出现了？

2、还是 Starting Window 消失后过了一阵子 Activity 界面显示了？

3、亦或是 Starting Window 根本就没有显示，是 Activity 界面消失又出现了？

......

可能的原因很多，单从问题场景无法得知原因。

### 7.2 分析方式一：添加 log

在计算 Layer 可见区域的函数 Output.ensureOutputLayerIfVisible 中添加 log，挑整个过程中的几次遍历看下：

1、SnapshotStartingWindow 先被显示：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/038be562438e4d529299f8f2f275c991~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=QOob1TqL%2B18MHYqKfburdbdp2J8%3D)

2、然后在真正的 MainActivity 显示之前，SnapshotStartingWindow 被隐藏了：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/22f3f0058a6e412d92112085ddace69e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=pQxZk9HuTGmLnVtVzWLmjd968Io%3D)

3、最终 MainActivity 显示：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3e9bf40527c34b6b84baacfc9355cdef~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=7yKw7AiPR2mtSaK%2Fk7JJH4bDJyA%3D)

4、正常来说，SnapshotStartingWindow 应该是要等 MainActivity 显示了之后才隐藏的，如这份正常的 log：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e9be807fa24f4c6fb6f73ca552a250b3~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=%2FgWBTkx1ej666OkdezG946c93bc%3D)

MainActivity 显示的时候 SnapshotStartingWindow 还在，但是出现问题的时候，SnapshotStartingWindow 是提前隐藏 / 销毁了。

### 7.3 分析方式二：Winscope

以上方式的缺点显而易见，我们得有一套编译过的代码以及相匹配的手机才行，有的时候你可能手头没有这些，你要重新下一套代码再去编译啥的，很花时间，再看使用 Winscope 如何分析问题。

我这边只抓了 SurfaceFlinger 和视频的信息，当然也可以 SurfaceFlinger 和视频的信息结合着一起看，但是这样会比较卡，由于复现的时间比较长，所以我这边先查看了视频的信息，如下：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b90be90ee7314ce8a1520373b0930372~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=HdyRLemZdlKX1Meo%2B4B30cPCZcw%3D)

上面显示的 “f=...” 应该是帧号的意思，那么看视频：

1）、大概是在 1130 帧的时候，头条的相关界面（可能是 StartingWindow，也可能是头条的 Activity，现在还不得而知）完全消失。

2）、1131 帧的时候，Launcher 界面也消失。

3）、1135 帧的时候，头条的相关界面又重新出现。

再看 Winscope 的信息：

1、由于是 1130 帧出现了问题，那么首先看 1129 帧：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1f8bda94a078434dbd66728a8c1e166f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=7bcfONIPBT9xQ%2Fv21syEiebvhOY%3D)

看视频的信息，知道此时头条的界面还在显示，从 SurfaceFlinger 的信息知，此时显示的是头条的 StartingWindow。

展开看 StartingWindow：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6b09b0f42e3f40b097e3910c9ac3ec55~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=7ngPn3%2FrX21Fr2dxjir0VYhEH28%3D)

其 Layer 位于头条对应的 Task#8 之下，没毛病。

2、接着是 1130 帧：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ae1127296bea41b18220a61eef8e6a04~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=xoBvJT4sOgIMB5GpJl4S%2BA4aidc%3D)

看视频此时 StartingWindow 不见了，再看 SurfaceFlinger 的信息，头条的 StartingWindow 对应的可见 Layer 也不在了。

再展开看头条对应的 Task#8 的信息：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a5aed9e3265d4cc493f14ae38b07a921~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=7J2YpAKu4wUZvBPH%2BTAk4QymU%2B0%3D)

发现 StartingWindow 的 WindowState 啥的全被移除了。

此时头条的 MainActivity 还没显示，但是 StartingWindow 已经被移除了，这个肯定是不对的。

3、最后看 1135 帧：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/2ad76b54d3f0420c991f95ed4bc0af49~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=nqmpRNWezLlAjRN1%2Bl2VxPcZt9I%3D)

看视频的信息，此时头条的界面又显示出来了，从 SurfaceFlinger 的信息知，此时显示的是头条的真正的界面，MainActivity。

根据 Winscope 显示的信息，我们可以和方式一添加 log 调试那样得出同样的结论，即 StartingWindow 被提前隐藏 / 移除了。

### 7.4 问题原因

根据以上分析，我们知道了闪屏的原因是因为 StartingWindow 被提前移除了，但是从 Winscope 上也无法再得出更多的信息了。

继续打开更多相关的 log 开关，主要是 WM_DEBUG_STARTING_WINDOW、WM_SHOW_TRANSACTIONS 以及 WM_SHOW_SURFACE_ALLOC，并且结合我自己的 log（有没有都行），抓取一份复现问题的 log 进行分析：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/2f7a57d6ccdf431999e85426180e67b9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=9hAx8oB5HsLwLFOtDcnBzI5K1sw%3D)

log 中的信息总结如下：

1、SnapshotStartingWindow 窗口被添加、绘制完成以及 SurfaceControl 显示。

2、SnapshotStartingWindow 对应的 Layer 显示。

3、头条的一个子窗口，PopupWindow，绘制完成，于是准备移除 SnapshotStartingWindow，但是 MainActivity 的主窗口还没绘制完成。

4、WMS 侧发起对 SnapshotStartingWindow 的隐藏和销毁。

5、SnapshotStartingWindow 的 Layer 被隐藏，但是此时头条的 MainActivity 还没显示。

6、头条 MainActivity 的主窗口绘制完成、显示。

7、头条 MainActivity 主窗口对应的 Layer 显示。

关键就在于当头条的一个子窗口，PopupWindow，绘制完成后，即触发了：

ActivityRecord.onFirstWindowDrawn

-> ActivityRecord.removeStartingWindow

-> ActivityRecord.removeStartingWindowAnimation

移除了 SnapshotStaringWindow。

但是实际上不管是看视频，还是看 log，都是没找到这个 PopupWindow 的相关信息的，查看这个 PopupWindow 的信息：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6815b1b8e02244479ac304c2e69dc34a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=Dx2XdalK3QuRcrB4A%2BDAOXIzek0%3D)

请求的宽是 0，所以这个 PopupWindow 实际上是没有内容显示的，但是它绘制完成的时候，又认为此时可以移除 SnapshotStartingWindow 了，这肯定是不合理的。

### 7.5 pixel 现象

最后看到 pixel 上同样可以复现，log 为：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a0a7c50d8ec2494fa2e9c0cc2105d794~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=f%2BlDyCoenngtVqni%2BK5s1C%2FyH5E%3D)

原因是一样的，由于 PopupWindow 绘制完成，移除了 SnapshotStartingWindow，而此时 MainActivity 的主窗口还没绘制完成，从而出现闪屏：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f3ef6567e94b44e7a4acaf9348000b49~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Yip57u05Lqa55qE5p2w5rSb54m5:q75.awebp?rk3s=f64ab15b&x-expires=1749683750&x-signature=UBphwRTkBz1ipKarR3TYusxE4DM%3D)

### 7.6 小结

从本题的分析我们可以看到，如果是用 Winscope 分析这个问题的话，只需要两步：

1、手头任意一台能够复现问题的手机，用 Winscope 查看问题发生过程中的界面变化。

2、打开更多 log 开关，然后根据 log 查明问题发生的原因。

如果不用 Winscope，那么我们的解题步骤是：

1、下一套代码，添加 log，编译。

2、找个机器，刷和下载的代码日期相近的版本。

3、将编译出的东西 push 到手机，然后复现问题，根据 log 得知问题发生过程中的界面变化。

4、打开更多 log 开关，然后根据 log 查明问题发生的原因。

这种问题，耗时的地方往往在于前几步，因为 log 中没有信息来表明每时每刻屏幕上正在发生的界面变化，如果没有 Winscope 的话，那么我们只能搞一套代码，添加 log 来调试，非常麻烦。

但是 Winscope 也有局限性，大部分情况下我们无法只通过 Winscope 就知道问题原因，还是需要结合 log 或者其它工具来综合分析问题。
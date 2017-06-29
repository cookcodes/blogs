---
title: Android图形系统架构（SurfaceView 和 GLSurfaceView）
date: 2017-06-27 10:29:32
tags: Android graphic surfaceview glsurfaceview
---

译自：https://source.android.com/devices/graphics/arch-sv-glsv
 
<!-- toc -->

# SurfaceView 和 GLSurfaceView

安卓应用程序框架的 UI 基于从View开始的层次结构的对象。所有的 UI 元素通过一个复杂的度量和布局过程以把它们放入一个矩形区域。当应用程序被带到前台时，所有可见的 View 对象被渲染到一个Surface，该 Surface 由SurfaceFlinger 创建，并由 WindowManager 设置。布局和渲染都在应用程序的 UI 线程上执行（不论布局和View的数量是多少，View是否硬件加速）。

和其他 View 一样，SurfaceView 接收同样的参数，因此您可以给它位置和大小，并把其他元素放在它周围。然而，当渲染的时候，它的内容是完全透明的。SurfaceView 的 View 部分只是一个透明的占位符（placeholder）。

当 SurfaceView 的 View 组件要变得可见时，框架让 WindowManager 请求 SurfaceFlinger 来创建一个新的 Surface。（这不是同步发生的，这就是为什么你要提供一个回调，当 Surface 创建完成时通知您。）默认情况下，新的 Surface 被放置应用程序 UI Surface 的后面，但你可以重写默认的"Z order"，以把 Surface 放在上面。

无论你渲染什么到此Surface，Surface将由 SurfaceFlinger 进行组合，而不是由应用程序。这是 SurfaceView 真正强大的地方：你得到的Surface 可以由一个单独的线程或一个单独的进程进行渲染，和应用程序的 UI 渲染完全独立，缓冲区将直接送给 SurfaceFlinger。但你不能完全忽略 UI 线程（你仍需要与 Activity 的生命周期进行协调，并且，你可能需要调整一些东西，如果View的尺寸或位置发生了变化）。不过你有仍拥有该Surface的全部，和应用程序的UI以及其他图层进行混合是由 Hardware Composer 进行处理的。

新的Surface是BufferQueue的生产者侧，BufferQueue 的消费者是 SurfaceFlinger 层。你可以用任何可以 feed 给 BufferQueue 的方式更新 Surface。比如：使用Surface提供的Canvas功能、附着到 EGLSurface 并用GLES进行绘制、配置 MediaCodec 视频解码器对其进行写入。

## 组合和硬件缩放器

让我们更详细地查看 **dumpsys SurfaceFlinger**，下列输出是 Nexus 5 在竖直方向用 Grafika 的"播放视频 (SurfaceView)" Activity 播放一部电影时得到的。这个电影是 QVGA (320 x 240）格式的。

    type    |          source crop              |           frame           name
    - ------+-----------------------------------+--------------------------------
        HWC | [    0.0,    0.0,  320.0,  240.0] | [   48,  411, 1032, 1149] SurfaceView
        HWC | [    0.0,   75.0, 1080.0, 1776.0] | [    0,   75, 1080, 1776] com.android.grafika/com.android.grafika.PlayMovieSurfaceActivity
        HWC | [    0.0,    0.0, 1080.0,   75.0] | [    0,    0, 1080,   75] StatusBar
        HWC | [    0.0,    0.0, 1080.0,  144.0] | [    0, 1776, 1080, 1920] NavigationBar
    FB TARGET | [    0.0,    0.0, 1080.0, 1920.0] | [    0,    0, 1080, 1920] HWC_FRAMEBUFFER_TARGET

* **列表顺序**是从后到前的：SurfaceView 的 Surface 在后面，应用程序 UI 层在其之上。然后是状态栏和导航栏。

* **source crop** 说明的是 Surface 缓冲区中要被 SurfaceFlinger 显示的部分。应用程序UI的Surface等于显示器的全尺寸(1080 x 1920），但渲染和合成要被状态栏和导航栏覆盖的部分没有意义，所以 source 被裁剪为一个矩形，从离顶部75像素开始，结束于离底部144像素。状态栏和导航栏的 Surface 更小，"source crop" 描述了从矩形的左上角 (0，0)开始并跨越它们的内容。

* **frame** 定义了像素最终在显示器的哪块矩形区域显示。对于应用程序UI，“frame” 和 "source crop" 一样，这是因为我们要把显示器大小的图层的一部分复制 （或覆盖）到另一个显示器大小的图层的同一位置。对于状态栏和导航栏，”frame“ 和  "source crop" 矩形的大小是一样的，但位置进行了调整，这样的导航栏将出现在屏幕的底部。

* **SurfaceView** 包含了电影的内容。"source crop"匹配的视频的大小，SurfaceFlinger 知道视频的大小是因为 MediaCodec 解码器（缓冲区生产者）是用该大小去 dequeue 缓冲区的。但 ”frame“ 矩形有一个完全不同的大小 —— 984 x 738。

SurfaceFlinger 处理尺寸大小的差异是通过缩放缓冲区中的内容去填充“frame”矩形，根据需要进行放大或缩小。这个特定的尺寸选择是因为它和视频有相同的宽高比（4：3），并且是尽可能地宽，但受 View 布局的约束（其中包括一些为审美原因在屏幕边缘的填充）。

如果你在同一 Surface 上开始播放不同的视频，底层 BufferQueue 会自动重新分配新的大小的缓冲区，SurfaceFlinger 将调整"source crop"。如果新视频的宽高比是不同的，应用程序需要强制 View 重 新布局以适应新视频，这样 WindowManager 将告诉 SurfaceFlinger 去更新 ”frame“ 矩形。

如果您通过一些其他手段去渲染 Surface，也许GLES，你可以使用SurfaceHolder#setFixedSize()调用来设置 Surface 的大小。例如，你可以配置一个游戏总是按 1280 x 720 渲染，这将大大降低需要处理的像素数，相对于 2560×1440 平板电脑或 4K 电视的屏幕上的每个像素都需要处理来讲。显示处理器会处理缩放。如果你不想让你的游戏有黑边框，您可以通过设置大小来调整游戏中的宽高比，比如设置窄边是 720 像素，但长的边设置成保持物理显示的宽高比（例如 1152 x 720 以匹配 2560 x 1600 显示） 。一个使用该方法的例子是 Grafika 的"hardware scaler exciser ” activity。

## GLSurfaceView

GLSurfaceView 类提供一些帮助类用于管理 EGL 上下文、线程间的通信、和Activity的生命周期交互。就这些，你不需要使用 GLSurfaceView 来使用 GLES。

例如，GLSurfaceView 创建了一个线程用于渲染，并在该线程配置了 EGL 上下文。当 Activity 暂停时，状态被自动清理。大部分的应用程序不需要知道任何关于 EGL 的事情，就可以和 GLSurfaceView 一起使用 GLES。

在大多数情况下，GLSurfaceView 是很有帮助，可以让 GLES 更容易工作。在某些情况下，用这种方式却有麻烦。如果它有用，就使用它，否则不要使用它。

## SurfaceView 和 Activity 的生命周期

当使用 SurfaceView，被认为是好的做法是从主 UI 线程以外的线程来渲染 Surface。这引发了一些关于该线程与 Activity 生命周期之间的交互的问题。

首先，一点背景知识。对于一个具有 SurfaceView 的 Activity，有两个独立而又相互依存的状态机：

1. 应用程序 onCreate / onResume / onPause
	
2. Surface 创建 / 更改 / 销毁

当 Activity 开始时，你按这种顺序中得到回调：

* onCreate
* onResume
* surfaceCreated
* surfaceChanged

如果你点击"back"，你会得到：

* onPause
* surfaceDestroyed （仅在 Surface 消失之前被调用）

如果你旋转屏幕，Activity 是拆掉后重新创建，所以你将获得整个周期。你可以通过检查isFinishing()是来检测activity是否"快速"重起的。（有可能开始 / 停止 Activity 如此之快，surfaceCreated() 可能会真正发生在 onPause() 之后。)

如果你按电源键以黑屏，你只得到onPause() —— 没有 surfaceDestroyed()。Surface 仍然活着，并且可以继续渲染。如果你继续要求，甚至可以继续得到 Choreographer 事件。如果你锁定屏幕并旋转屏幕，当设备重新点亮的时候，您的 activity 可能需要重新启动；但如果不是，点亮屏幕的时候，你可能得到和以前一样的 Surface。

如果 SurfaceView 使用一个独立的渲染器线程时，这就提出了一个根本性的问题：线程的生命周期应该与 Activity 还是 Surface 挂钩？答案取决于屏幕黑屏时你想要发生什么。有两种基本选项：（1）在Activity 启动/停止的时候启动/停止线程；(2) 在 Surface 创建/销毁的时候启动/停止线程。

选项1与应用程序生命周期进行交互。我们在onResume()中启动渲染线程，在onPause()中停止该线程。创建和配置线程时有时有点尴尬，因为 Surface 有时已存在，有时不存在。（例如按电源按钮来关闭/打开屏幕，Surface 是还活着的）。在线程做一些初始化工作之前，我们必须等待 Surface 创建好。但我们不能简单地通过surfaceCreated()回调来等待，因为如果 Surface 没被重新创建，该回调不会被再次触发。所以我们需要查询或缓存 Surface 的状态，并将其转发到渲染线程。

    注意：线程之间传递对象，我们要小心一点 — — 最好是通过Handler 的消息来传递 Surface 或 SurfaceHolder，而不是只将它塞入线程，以避免在多核系统上发生问题（参看Android SMP Primer)。

选项2有一定的吸引力，因为 Surface 和渲染器在逻辑上交织在一起。在 Surface 创建之后，我们启动了线程，避免了一些线程间的通信问题。Surface 创建/更改消息只是简单地被转发。我们需要确保关闭屏幕时停止渲染，打开屏幕时继续渲染。这可能是一个简单的事，告诉 Choresgrapher 停止调用绘制帧的回调。onResume()恢复回调（需要渲染器线程运行的情况下）。然而，它可能不是如此简单 —— 如果我们的动画基于帧之间经过的时间，在下一个事件到达时，我们可能会有非常大的差距；所以显式的暂停/恢复消息可能是需要的。

    注意：选项2的例子，请参看 Grafika 的"Hardware scaler exercise"
	
两种选项主要关注的都是渲染线程如何配置和它是否执行。一个相关的问题是该 Activity 被杀（在onPause()或onSaveInstanceState()）时如何从渲染线程中提取状态。在这种情况下，选项1工作的最佳，因为一旦渲染器线程已加入，无需同步原语就可以访问其状态。


---
title: Android图形系统架构（Surface 和 SurfaceHolder）
date: 2017-06-26 18:23:50
tags: Android Graphic Surface SufaceHolder
---

译自：https://source.android.com/devices/graphics/arch-sh
 
<!-- toc -->

# Surface 和 SurfaceHolder

自1.0以来，surface类就是公开 API 的一部分。对它的描述很简单："由屏幕组合器管理的raw缓冲区的句柄“。该语句在最初的时候是准确的，但在现在的系统上，该描述是不足够的。

Surface 表示的是缓冲队列（Buffer queue）的生产者方，该缓冲队列通常（但不总是）由 SurfaceFlinger 消费。当你渲染在Surface上时，结果最终是在一个缓冲区，该缓冲区将被送给消费者。Surface不仅仅是简单的你可以在上面乱涂乱画的一块原始内存。

显示Surface的 BufferQueue 通常配置为三倍缓冲。但缓冲区是按需求分配的。因此如果生产者生成缓冲区足够慢（也许它在 60 fps 的显示器上按 30 fps动画）， 可能缓冲区队列中那里只有两个已分配的缓冲区。这将有助于尽量减少内存消耗。您可以在dumpsys SurfaceFlinger的输出中看到与每一图层相关联的缓冲区的摘要信息。

## Canvas 渲染
很久以前，所有的渲染在软件中完成，你今天仍然可以这样做。底层的实现是由 Skia 图形库提供的。如果你想要绘制一个矩形，使用一个库调用，那么它将在缓冲区中适当地写字节。为了确保缓冲区不是由两个客户端同时更新，或在显示的时候被写入数据，你必须先 lock 要访问的缓冲区。lockCanvas()锁定缓冲区，并返回一个 Canvas 用于绘图，使用 unlockCanvasAndPost()解锁缓冲区，并将它发送给组合器。

随着时间的推移，具有通用 3D 引擎的设备出现了。安卓系统围绕 OpenGL ES 调整了方向。然而，对于应用程序以及应用程序框架代码，确保老的 API 能继续工作是非常重要的，所以花了一些功夫用硬件加速Canvas API。你可以看看[硬件加速](http://developer.android.com/guide/topics/graphics/hardware-accel.html)网页上的图表，这是有点波折的路。特别要注意到，虽然Canvas向View提供的的onDraw()方法可能是硬件加速的，应用程序锁通过lockCanvas()锁定Surface来获得的Canvas从来都不是硬件加速的。

当你锁定一个Surface为Canvas进行访问时，"CPU 渲染器"连接到 BufferQueue 的生产者方，并不会断开连接，直到Surface被销毁。其他的大多数的生产者（就像 GLES）可以断开连接，并重新连接到Surface。但基于Canvas的"CPU渲染器"不能。这意味着如果你已经把Surface锁定为Canvas，您不能在Surface上用GLES绘画，或是把来自视频解码器的帧发给它。

生产者首次从 BufferQueue 请求缓冲区，分配的缓冲区将被初始化为零。初始化是必需的，用于避免进程之间无意的数据共享。然而，当你重用一个缓冲区时，以前的内容仍将存在。如果你重复调用lockCanvas()和unlockCanvasAndPost()并没有绘制任何内容，您会在先前渲染完的帧之间循环。

Surface lock/unlock 代码保持对先前渲染好的缓冲区的引用。如果您在锁定Surface时指定一个dirty区，它将从以前的缓冲区复制非dirty的像素。很有可能，该缓冲区将由 SurfaceFlinger 或 HWC 处理，但由于我们只是需要从它读取数据，所以无需等待独占访问权。

如果应用程序不是用 Canvas 方式在 Surface 上直接绘制，那么主要方式是通过 OpenGL ES。这在 EGLSurface 和 OpenGL ES章节描述。

## SurfaceHolder
和 Surface 工作的有些事情想要 SurfaceHolder（著名的是 SurfaceView）。最初的想法是 Surface 代表原始的、组合器管理的缓冲区，而 SurfaceHolder 由应用程序管理，保持跟踪更高级别的信息，比如尺寸和格式。Java 层的定义，是底层native层实现的镜像。你也可以说不再需要这样拆分，但它一直是公共 API 的一部分。

一般来说，任何与view相关的事情将涉及 SurfaceHolder。一些其他的 API，如 MediaCodec，将在Surface本身上操作。你可以从 SurfaceHolder 很容易得到Surface，所以有了 SurfaceHolder， 就一直使用它。

获取和设置 Surface 参数（如大小和格式）的API ，透过 SurfaceHolder 来实现。

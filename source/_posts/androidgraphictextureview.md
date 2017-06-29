---
title: Android图形系统架构（TextureView）
date: 2017-06-28 18:08:25
tags: Android graphic TextureView
---

译自：https://source.android.com/devices/graphics/arch-tv
 
<!-- toc -->

# TextureView

TextureView 类开始出现在 Android 4.0 中, 是这里讨论的最复杂的View对象，结合了View 和SurfaceTexture。

## 用GLES渲染

记得 SurfaceTexture 是"GL 消费者"，它消费图形数据缓冲区，使它们可作为纹理使用。TextureView 封装了 SurfaceTexture，负责响应回调和acquire新缓冲区。当接收到新的缓冲区时，TextureView 发出一个“View invalidate”的请求。当被要求画画时，TextureView 使用最近接收的缓冲区作为其数据源进行渲染，不论 View 的状态如何指示它。

就像 SurfaceView 一样，你可以用 GLES 在 TextureView 上渲染。只需将 SurfaceTexture 传递到创建 EGL 窗口的调用。但是，这样做暴露了一个潜在的问题。

我们已经看，在大部分的情况下，BufferQueues 在不同进程之间传递缓冲区。但是，当用 GLES 在 TextureView 上渲染，生产者和消费者都在同一进程中，他们甚至可能在同一个线程上运行。假设我们从 UI 线程连续快速提交几个缓冲区。EGL 缓冲区互换调用函数将需要从 BufferQueue 出列缓冲区，它将等待直到有一个缓冲区可用。但不会有任何缓冲区可用，除非消费者获得了一个缓冲区用于渲染，但消费者也恰好在 UI 线程上......，所以我们陷入了死等。

解决办法是让 BufferQueue 确保始终有一个缓冲区可用来出列，所以缓冲区互换调用永远不会陷入等待。有一种方法来保证这点：当有一个新的缓冲区入队时， BufferQueue 丢弃以前排队缓冲区的内容，并应用最小缓冲区计数与最大可 acquire 缓冲区的计数限制。（如果您的队列具有三个缓冲区，且所有三个缓冲区由消费者 acquire 了，那就没有缓冲区可用于 dequeue 了，则缓冲区交换调用必然挂起或失败。所以我们需要防止消费者一次获得两个以上的缓冲区)。删除缓冲区是通常是不可取的，所以它的使用只在特定情况下，例如当生产者和消费者都是在相同的进程中。

## SurfaceView 或 TextureView ？

SurfaceView 和 TextureView 具有相似的角色，但有非常不同的实现。决定哪一个是最好，需要权衡取舍。

因为 TextureView 是 View 层次结构的一个正常的成员，它像任何其他 View 一样，可以覆盖其他元素或被其他元素覆盖。您可以用简单的 API 调用执行任意的转换和把它作为一个位图访问其内容。

TextureView 的主要挑战是组合过程时的性能。对于 SurfaceView，内容写入到一个单独的图层，由 SurfaceFlinger 进行组合，对叠加来说是非常理想的。而对于 TextureView，View 组合始终由 GLES 执行，对其内容的更新可能会导致其他 View 元素的重绘（例如，如果它们的位置在 TextureView 之上）。该 View 渲染完成后，应用程序 UI 层必须再与其他图层由 SurfaceFlinger 进行组合，这样每个可见像素被组合了两次。对于一个全屏的视频播放程序，或任何其他的应用程序，只有 UI 元素在视频上叠加，SurfaceView 能提供更好的性能。

如前所述，受 DRM 保护的视频只可以在“叠加”面上呈现。支持保护内容的视频播放器必须用 SurfaceView 实现。

## 案例研究：Grafika 的播放视频(TextureView)

Grafika 包含了两个视频播放器，一个用 TextureView 实现的，另一个用 SurfaceView 实现的。视频解码部分，将帧从 MediaCodec 发送到 Surface，对两者是相同的。两者的实现上最有趣的的区别是呈现正确宽高比所需的步骤。

SurfaceView 需要FrameLayout的定制实现，而 SurfaceTexture 的大小调整是一个简单的事情，用 TextureView#setTransform() 配置一个转换矩阵。对于前者，你通过 WindowManager 把新窗口位置和大小值传递给 SurfaceFlinger。对于后者，你只是渲染的时候不同。

其它部分，这两个实现遵循相同的模式。一旦创建了 Surface，播放处于启用状态。当 “play” 被点击的时候，视频解码线程开始工作，Surface 是其输出目标。在那之后，应用程序代码并不需要做任何事 —— 组合和显示要么由 SurfaceFlinger（为 SurfaceView）处理，要么由 TextureView 处理。

## 案例研究：Grafika 的双解码

这个 Activity 演示了如何操作 TextureView 中的 SurfaceTexture。

这个 Activity 的基本结构是两个 TextureView 肩并肩地播放两个不同的视频。为了模拟一个视频会议应用程序的需要：保持 MediaCodec 解码器是活着的，即使由于手机方向改变带来的 Activity 的 pause 和resume。麻烦的是你不能改变 MediaCodec 解码器正在使用的 Surface，除非你重新配置它，这是相当昂贵的操作；所以我们要保持 Surface 还活着。Surface 只是 SurfaceTexture 的 BufferQueue 的生产者接口的句柄，SurfaceTexture 又是由 TextureView 管理的；所以我们要保持 SurfaceTexture 是活的。但又如何处理被销毁的 TextureView？

恰巧 TextureView 提供了一个setSurfaceTexture()调用，刚好做了我们想要的。我们从 TextureViews 获得 SurfaceTextures 的引用，并将它们保存在静态字段中。当该 activity 是关闭时，我们从 onSurfaceTextureDestroyed() 回调返回"false"，防止 SurfaceTexture 被销毁。该 activity 重新启动时，我们把老的 SurfaceTexture 喂给新 TextureView。TextureView 类负责创建和销毁 EGL 上下文。

每个视频解码器由单独的线程驱动。乍一看上去，每个线程需要本地的 EGL 上下文；但请记住，已解码的输出缓冲区实际上是从媒体服务器发送给BufferQueue 的消费者（SurfaceTextures）。TextureViews 负责渲染，他们在 UI 线程上运行。

如果用 SurfaceView 去实现这个 Activity 会有点困难。我们不能仅仅创建两个 SurfaceViews，然后将输出定向到它们就可以了，因为Surface 会在手机的方向变化期间被销毁。此外，这会增加两个图层，可叠加图层的数目的限制促使我们要保持最少的图层数。相反，我们会想到创建两个 SurfaceTextures，用于从视频解码器接收输出，然后在应用程序中执行渲染：使用 GLES 渲染两个纹理化的正方形到 SurfaceView 的 Surface 上。

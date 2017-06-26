---
title: Android图形系统架构（BufferQueue 和 gralloc）
date: 2017-06-26 10:42:50
tags: Android Graphic
---

译自：https://source.android.com/devices/graphics/arch-bq-gralloc
 
<!-- toc -->

# BufferQueue 和 gralloc #

要了解 Android 图形系统，我们从表象背后的BufferQueue 和 gralloc HAL开始。

BufferQueue 类是 android 中所有图形的核心。它的作用很简单: 连接生成图形数据缓冲区的东西("生产者")到接受数据进行显示或进一步处理的东西 ("消费者"）。几乎所有通过系统移动图形数据的缓冲区都依赖于 BufferQueue。

gralloc 内存分配器执行缓冲区分配。gralloc的实现是通过vendor specific的 HAL接口（参见hardware/libhardware/include/hardware/gralloc.h）。alloc()函数使用你期望的参数（宽度、高度、像素格式）以及一套使用标志(详见下文)。

## BufferQueue 生成者和消费者 

BufferQueue的基本用法非常简单。生产者请求空闲的缓冲区 （dequeueBuffer()），指定一组特征包括宽度、 高度、 像素格式和使用标志。生产者填充缓冲区，并将其返回到队列 （queueBuffer()）。一些时间后，消费者获得缓冲区 (acquireBuffer())，并使用的缓冲区中的内容。消费者使用完毕后，将它返回缓冲区到队列 （releaseBuffer())。

最近的 Android 设备支持*同步框架*。这使系统可以和能异步操作图形数据的硬件配合起来做一些俏皮的事。例如，生产者可以提交一系列的 OpenGL ES 的绘图命令，并把输出缓冲区在渲染完成之前排入队列。缓冲区关联着fence（保护带），当内容准备好时，fence会发出信号。当缓冲区返回到空闲列表时，也会有个fence关联，这样，消费者可以在缓冲区内容仍在使用时释放缓冲区。这种方法在系统内移动缓冲区时改善了延迟和吞吐量。

缓冲区队列的一些特征，比如它可以容纳的缓冲区的最大数目，是由生产者和消费者共同确定的。然而，当需要时，BufferQueue 负责分配缓冲区。除非特征变化，缓冲区将被保留；例如，如果生产者开始请求大小不同的缓冲区，老的缓冲区将会被释放。新的缓冲区将按要求的大小被分配。

生产者和消费者可以位于不同的进程中。目前，消费者总是创建和拥有数据结构。在旧版本的Android，只有生产者是binderized（即生产者可以在一个远程进程，但消费者不得不是在创建队列时所在的进程）。Android 4.4和以后的版本朝着一个更普遍的实现方向发展。

BufferQueue 永远不会复制缓冲区中的内容（移动那么多的数据将会非常低效）。相反，缓冲区始终通过句柄来传递。

## gralloc HAL 使用标志

Gralloc 内存分配器不仅仅是native heap上分配内存的另一个途径。在某些情况下，所分配的内存可能不是cache-coherent，或可能从用户空间完全无法访问。分配的性质取决于包括了属性的使用标志，比如：

	* 从软件 (CPU) 访问内存的频度
	* 从硬件 (GPU) 访问内存的频度
	* 会否作为 OpenGL ES （"GLES"） 纹理
	* 是否被视频编码器使用

例如，如果您把格式指定为RGBA 8888的像素，并且您指示缓冲区将从软件访问（这意味着您的应用程序可以直接访问像素），那么分配器将分配一个缓冲区，每个像素需要 4 个字节，按R-G-B-A顺序。相反，如果你说该缓冲区将作为GLES纹理，只能从硬件（GPU）访问，分配器可以做GLES驱动程序期望其做的任何事情 — — BGRA 顺序，非线形的"swizzled"布局，可替代颜色格式等。允许硬件使用其喜欢的格式以提高性能。

在一些平台上，有些值不能组合在一起。例如，"视频编码器"的标志可能需要以YUV 像素为单位，因此添加"软件访问"并指定 RGBA 8888标志会失败。

Gralloc 分配器返回的句柄可以通过Binder在进程之间传递。

## 用 systrace 跟踪 BufferQueue

如果你真的想要了解图形缓冲区是如何移动的，你需要使用 systrace。系统级的图形代码被设计成能很好地监控，很多都与应用程序框架代码相关。

完整地描述如何有效地使用 systrace ，需要一个冗长的文档。但起点可以是先启用"gfx"，"view"和“sched”标签，这样你就可以在trace中看到BufferQueue。如果您之前使用过 systrace，你可能已经见过他们，但或许你不知道它们是什么。一个例子，如果你在 Grafika 的"Play Video（SurfaceView）"运行时抓了一个 trace，，标记为"SurfaceView"的行将告诉你在指定的时刻有多少缓冲区被排队。

在该应用程序处于活动状态时，该值会递增（由 MediaCodec 解码器渲染帧触发）或递减（SurfaceFlinger 工作时会消耗缓冲区）。如果播放的视频是每秒 30 帧，队列的值会从 0 到 1变化，因为60 fps 的显示器可以很容易跟上源。（需要注意的是，SurfaceFlinger 只在有工作需要做时才醒来，而不是 60 次每秒。系统将尽可能地不工作，如果屏幕上什么都不需要更新，将完全禁用 VSYNC)。

如果你切换到"Play video（TextureView）"，抓一个新的 trace，你会看到其中有一行标记为（"com.android.grafika/com.android.grafika.PlayMovieActivity")。这是主UI层，当然这只是另一个 BufferQueue。因为 TextureView 渲染到 UI 层，而不是单独的图层，您将在这里看到的所有与视频相关的更新。

有关 systrace 的详细信息，请参阅 [Systrace documentation](https://developer.android.com/studio/profile/systrace-commandline.html)

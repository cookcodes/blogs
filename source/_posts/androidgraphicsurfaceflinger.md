---
title: Android图形系统架构（SurfaceFlinger 和 Hardware Composer）
date: 2017-06-26 15:41:50
tags: Android Graphic SurfaceFlinger
---

译自：https://source.android.com/devices/graphics/arch-sf-hwc
 
<!-- toc -->

# SurfaceFlinger 和 Hardware Composer 

有了图形数据缓冲区是美好的，但当看到他们显示在您的设备的屏幕上时，生活就更美好了。该 SurfaceFlinger 和 Hardware Composer HAL登场了。

## SurfaceFlinger

SurfaceFlinger 的角色是接受来自多个来源的缓冲区，组合它们，并将它们发送到显示器。以前，是软件把他们blit到硬件缓冲器 （例如/dev/graphics/fb0），但那些日子一去不复返了。

当一个应用程序走到前台，WindowManager service 向 SurfaceFlinger 要一个surface。SurfaceFlinger 创建"图层"（最主要的东西就是 BufferQueue），SurfaceFlinger 充当消费者。生产者需要的Binder对象通过 WindowManager 传递到该应用程序，然后应用程序就可以直接向 SurfaceFlinger 发送帧了。（注： 本文使用了SurfaceFlinger的术语，WindowManager 使用术语"窗口"，而不是"图层"，因为"图层"意味着别的东西。你也可以认为 SurfaceFlinger 真正应该被称为 LayerFlinger)。

对于大多数应用程序，在任何时间都会有三层屏幕： 在屏幕顶部的"状态栏"，在底部或侧面的"导航栏"，和应用程序的 UI。一些应用程序可能会有更多的或更少的层（例如默认的home应用程序具有单独的壁纸图层，全屏游戏可能会隐藏状态栏）。每个图层可以被单独更新。状态栏和导航栏由系统进程负责渲染，应用图层由应用程序层负责渲染，他们之间没有协调。

设备以一定的速率刷新显示，手机和平板通常是60帧每秒。如果在刷新时更新显示的内容，"撕裂"将可见。所以在刷新周期之间去更新内容是非常重要的。当可以安全更新时，系统将从显示器接收到一个信号。由于历史的原因我们会称这个信号为VSYNC（垂直同步信号）。

刷新率随着时间的变化可能会发生变化，例如某些移动设备基于当前的情况，刷新率在58至62fps之间。HDMI连接的电视，理论上可以是 24 或 48 Hz，以匹配视频。因为每个刷新周期只可以更新屏幕一次，提交 200 帧/秒显示的缓冲区将是巨大的浪费，因为大部分的帧将永远不会显示。SurfaceFlinger不是每当一个应用程序提交缓冲区就采取行动，而是在显示器准备接收新的东西时醒来。

VSYNC信号到达时，SurfaceFlinger 扫描图层列表以寻找新的缓冲区。如果发现一个新的，就acquire它 ；如果没找到新的，它将继续使用以前acquire的缓冲区。SurfaceFlinger 总是想要有一些东西去显示，所以它总是会有一个缓冲区。如果一个图层没有缓冲区被提交过，该图层将被忽略。

一旦 SurfaceFlinger 收集完所有可见图层的缓冲区，它询问Hardware Composer如何组合这些图层。

## Hardware Composer

Hardware Composer HAL （"HWC"） 首先在 Android 3.0 （"Honeycomb"）中出现，在过去的几年稳步发展。其主要目的是利用现有的硬件来确定最有效的方式，以对缓冲区进行组合。作为一个 HAL，它的实现特定于设备，通常是由显示硬件OEM实现的。

当你考虑"叠加平面"时，就很容易意识到这种方法的价值。该方法的目的是在显示硬件（diaplay hardware）里"叠加平面"，而不是在GPU里。例如，假设您有一种典型的，竖直方向的Android 手机，状态栏在顶部，导航栏在底部，其他地方是应用程序内容。每个图层的内容是在各自独立的缓冲区中。您可以使用下列方法之一来处理叠加：

    找一个空白的到缓冲区，先在里面渲染应用程序的内容，然后在它上面渲染状态栏，最后在它上面渲染导航栏，然后将该缓冲器传递给显示硬件。
	将所有三个缓冲区传递给显示硬件，并告诉它，屏幕的不同区域，需要从不同的缓冲区读取数据。

后者的效率显然更高。

不同的显示处理器，能力相差很大。层数、 层是否可以旋转或混合、定位和叠加的限制，这些很难通过一个 API 来表达。所以，HWC通过一系列的决定来兼容这种多样性：

	1. SurfaceFlinger 给HWC提供完整的图层的列表，并问，"你想怎么想处理这些?"
	2. HWC的响应是为每个图层标记为"叠加"或"GLES 组合"。
	3. SurfaceFlinger 负责所有的"GLES 组合"，并将结果缓冲区传递到HWC，剩下的事情HWC负责处理。

硬件供应商负责定义定制决策代码，这样可以使得每个设备发挥最佳的性能。

当屏幕上没有任何变化时，HWC"叠加“平面的方法可能不如"GL 组合"的效率高。特别是当叠加的内容有透明的像素，重叠的图层混合在一起时。在这种情况下，HWC可以为一些或所有图层选择请求"GLES 组合"，并保留组合好的缓冲区。如果 SurfaceFlinger 再次要求将相同的缓冲区进行组合，HWC可以只继续显示以前复合好的缓冲区。这可以提高idle状态时设备的电池寿命。

搭载 Android 4.4 （"KitKat"）和以后版本的设备通常通常支持四个覆盖面。尝试组合更多的层将导致系统使用"GLES 组合"来组合一些层。这意味着应用程序使用的层数会对功耗和性能产生重大的影响。

## 虚拟显示器

SurfaceFlinger 支持一个"主"显示器，即您的手机或平板上的显示器，一个"外部"显示器，如通过 HDMI 连接的电视机。它还支持一个或多个的"虚拟"显示器，使组合后的输出结果可在系统内获得。虚拟显示可用于屏幕录像或通过网络发送屏幕内容。

虚拟显示器可以和主显示 （"层堆栈"） 共享相同的图层集，也可以有它自己的一套图层集。虚拟显示器没有VSYNC，所以主显示器的VSYNC用来触发所有显示器的组合活动。

在老版本的Android，虚拟显示都是通过GLES来组合。hardware composer 只负责主显示器的组合。在 Android 4.4 , hardware composer 有能力参与虚拟显示器的组合。

正如你所预料的，虚拟显示器生成的帧将被写入到 BufferQueue。

## 案例研究： screenrecord（屏幕录像）

[Screenrecord 命令](https://android.googlesource.com/platform/frameworks/av/+/marshmallow-release/cmds/screenrecord/)允许您对屏幕上出现的一切进行录像，生成一个.mp4 文件存在磁盘上。要实现这点，我们从 SurfaceFlinger 接收组合好的帧，将它们写给视频编码器，然后将编码好的视频数据写入一个文件。视频编解码器是由一个单独的进程 （mediaserver）管理， 所以我们要在系统中移动大量的图形缓冲区。更具挑战性的是，我们需要录的是 60 fps 全分辨率的视频。这项工作能有效运转的关键是 BufferQueue。

MediaCodec 类允许应用程序提供的数据为缓冲区中的原始字节，或通过一个surface。当 screenrecord 请求访问视频编码器时，媒体服务器将创建一个 BufferQueue，并把自身连接到消费者一方，然后把生产者一方作为一个suface传递回 screenrecord。

Screenrecord 命令然后请求 SurfaceFlinger 创建一个虚拟显示器，它将镜像主显示器的内容（即具有所有相同的图层），并指示SurfaceFlinger将结果输出到来自媒体服务器的surface。在本案例中，SurfaceFlinger 是缓冲区的生产者，而不是消费者。

配置完成后，screenrecord 可以坐着等编码好的数据的出现。当应用程序绘制时，其缓冲区前往 SurfaceFlinger，SurfaceFlinger将他们组合到一个单一的缓冲区，并把它直接发送到媒体服务器中的视频编码器。所有的帧数据甚至从来没有没被 screenrecord 进程 看到。在内部，媒体服务器用它自己的方式移动缓冲区，它也是通过句柄传递数据，以尽量减少开销。

## 案例研究： 模拟辅助显示器

WindowManager 可以请求 SurfaceFlinger 创建一个可见图层，SurfaceFlinger 将作为该 BufferQueue 的消费者。它也可以要求 SurfaceFlinger 创建一个虚拟显示器，SurfaceFlinger 将作为 BufferQueue 生产者。如果你连接它们，将虚拟显示渲染到可见图层，将会发生什么？

一个封闭的循环被创建了，组合好的屏幕出现在一个窗口中。当然，该窗口现在也是组合输出的一部分，等在下次刷新时，窗口内的组合图像也将显示窗口的内容（[turtules all the way down](https://en.wikipedia.org/wiki/Turtles_all_the_way_down)）。在这里你能真正地看到该循环，在设置中启用"开发人员选项"，选择**模拟辅助显示设备**，这将打开一个窗口。另外更有意思的事是，使用 screenrecord 录屏启用显示的过程，然后一帧一帧的回放。

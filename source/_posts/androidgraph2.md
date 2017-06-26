---
title: Android图形系统架构（概览）
date: 2017-06-25 15:42:50
tags: Android
---

译自：http://source.android.com/devices/graphics/architecture.html
 
<!-- toc -->

# 概览

每个开发人员应该知道Surface、 SurfaceHolder、 EGLSurface、 SurfaceView、 GLSurfaceView、 SurfaceTexture、 TextureView，SurfaceFlinger 和 vulkan。

本文档描述Android系统的"系统级"图形架构的基本要素，以及它们是如何被应用框架和多媒体系统所使用的。重点是图形数据buffer如何在系统中移动。如果你想知道为什么SurfaceView 和 TextureView 如此行事，或Surface和 EGLSurface 如何进行交互，你来到了正确的地方。

你应该对Android 设备及其应用程序开发有一些了解。你不需要了解应用程序框架的详细的知识，只有很少的 API 调用会在这里被提及，但此处的材料并不重叠太多的其他公开文档。这里的目标是让你对要输出的帧的渲染所相关的一些重要事件有感知，以便让你能在设计应用程序时作出明智的选择。要实现这一目标，我们从下到上描述UI 类是如何的工作的，而不是如何使用它们。

本节涵盖了从背景材料到 hal 细节，到用户案例的所有内容。我们从安卓的图形Buffer的解释开始、然后描述组合（compose）和和显示机制，然后描述更高层次的机制是如何为组合器（compositer）提供数据。我们建议按照下列顺序阅读, 而不是跳到听起来有趣的主题。

## 低级别组件

* **BufferQueue 和 gralloc** BufferQueue 将生产图形数据缓冲区的东西(生产者)连接到接受数据进行显示或进一步处理的东西 (消费者)。缓冲区分配是通过 gralloc 内存分配器来执行的。gralloc HAL 接口需由各个供应商自己实现。

* **SurfaceFlinger、硬件组合器和虚拟显示器** SurfaceFlinger 接受来自多个源的数据缓冲区, 并将它们组合后发送到显示。硬件组合器HAL（HWC）决定了如何使用可用的硬件，来最有效地组合缓冲区。 虚拟显示器使得组合后的结果可在系统内能得到(比如录制屏幕或通过网络发送屏幕)。

* **Surface, Canvas, 和 SurfaceHolder** Surface生产缓冲区队列，该队列通常由 SurfaceFlinger 消耗。当渲染到一个Surface时，最终的结果是保存到一个缓冲器，并被运送给消费者。Canvas api 提供了一种直接在Surface上进行绘制的软件实现 (带有硬件加速支持), (低级别的替换方法是 opengl es)。任何与View有关的内容都涉及 SurfaceHolder, 其 api 允许获取和设置诸如大小和格式等Surface参数。

* **EGLSurface 和 opengl** opengl es (GLES) 定义了图形渲染 api, 它要与 EGL 结合使用。 EGL 是一个知道如何通过操作系统创建和访问 windows 的库 (绘制有纹理的多边形, 使用GLES调用; 在屏幕上显示渲染结果, 使用 EGL 调用)。本文还涵盖了 ANativeWindow, ANativeWindow在C/C++层相当于Java层的Surface，用于native code创建EGL窗口。

* **vulkan** vulkan 是一个低开销、跨平台的高性能3D 图形 api。与 opengl 一样, vulkan 提供了在应用程序中创建高质量、实时的图形的工具。vulkan 的优点包括减少 cpu 开销和支持 *SPIR-V Binary Intermediate* 语言。

## 高级别组件

* **SurfaceView 和 GLSurfaceView** SurfaceView 结合了Surface和View。SurfaceView 的 View 组件是由 SurfaceFlinger (而不是应用程序) 组合而成的, 它在单独的线程/进程渲染，独立于应用程序 ui 渲染。GLSurfaceView 提供了帮助器类来管理 egl 上下文、线程通信以及与Activity生命周期交互 (但不是需要使用GLES)。

* **SurfaceTexture** SurfaceTexture 结合了Surface和GLES纹理，并创建一个 BufferQueue, 你的应用程序是其消费者。当生产者入队一个新的缓冲区时，它会通知您的应用程序, 应用程序接着release以前保留的缓冲区, 从队列中acquire新的缓冲区, 并调用 egl 以使缓冲区可作为GLES的外部纹理。android 7.0 增加了支持安全纹理视频播放, 使 gpu 能够对受保护视频内容进行后处理。

* **TextureView** TextureView 将 View 与 SurfaceTexture 组合在一起。TextureView 将 SurfaceTexture 封装起来, 并负责响应回调和acquire新的缓冲区。在绘图时, TextureView 使用最近接收的缓冲区的内容作为其数据源, 并可将其渲染到任何位置, 但View状态指示了具体的位置。视图组合总是使用GLES来执行, 这意味着对内容的更新可能会导致其他View元素也需要重新绘制。

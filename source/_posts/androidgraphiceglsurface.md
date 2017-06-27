---
title: Android图形系统架构（EGLSurface 和 OpenGL ES）
date: 2017-06-26 18:23:50
tags: Android Graphic EGLSurface OpenGL
---

译自：https://source.android.com/devices/graphics/arch-egl-opengl
 
<!-- toc -->

# EGLSurface 和 OpenGL ES
OpenGL ES 定义了用于渲染图形的 API。它不定义视窗系统。为了让 GLES 能在多种平台上工作，它需要结合另外一个库才能进行工作。这个库知道如何通过操作系统创建和访问视窗。在 android 中，这个库叫 EGL。如果您想要绘制带纹理的多边形，则使用GLES调用；如果你想要把你的渲染绘制在屏幕上，你需要使用 EGL调用。

在做任何与GLES相关的事情之前，您需要创建一个 GL 上下文。在 EGL，这意味着要创建 EGLContext 和 EGLSurface。GLES 的操作适用于当前的上下文，上下文是通过线程本地存储区（thread-local storeage）访问的，而不是作为参数传来传去的。这意味着你必须要小心：渲染代码是在哪个线程上执行的，在该线程上的当前上下文是哪个。

## EGLSurface

EGLSurface 可以是 EGL 分配的离屏缓冲区 （称为"pbuffer"），或是由操作系统分配的一个窗口。EGL窗口Surface 通过调用eglCreateWindowSurface()创建。它使用"窗口对象"作为参数，"窗口对象"在 Android 上 可以是 SurfaceView、 SurfaceTexture、 SurfaceHolder 或 Surface -- 所有这些的下面都是 BufferQueue。当你发出该调用时，EGL 创建一个新的 EGLSurface 对象，并将它连接到窗口对象的 BufferQueue 的生产者接口。从那时起，渲染到 EGLSurface的结果是一个缓冲器 从BufferQueue出列，被渲染，并入列等待消费者使用。（术语"窗口"一词是表明了预期的用处，但记住，输出可不是注定要在显示器上显示的）。

EGL 不提供锁定/解锁的调用。相反，你发出绘图命令，然后调用eglSwapBuffers()来提交当前帧。eglSwapBuffers()的名称来自于传统的前后台缓冲区的交换，但是真正的实现可能非常不同。

一次只有一个 EGLSurface 可以与一个Surface关联 （只有一个生产者可以连接到 BufferQueue）。但如果你销毁了 EGLSurface，它将断开与 BufferQueue 的连接，允许其他的东西去连接。

给定的线程可以通过改变"current"的EGLSurfaces来切换多个 EGLSurfaces。EGLSurface 在一段时间内只能是一个线程的“current”。

对 EGLSurface 最常见的错误理解是认为它是 Surface的 另一个方面（就像 SurfaceHolder）。EGLSurface和surface是相关的、但独立的概念。你可以在EGLSurface上画画，但它可以不是由 Surface 支撑的。你也可以使用一个没有 EGL 的 Surface。EGLSurface 只是给了 GLES 画画的地方。

## ANativeWindow

公共 Surface 类是在 Java 编程层中实现。在C/C++ 中的对等的是 ANativeWindow 类，由Android NDK半暴露。通过 ANativeWindow_fromSurface()调用，你可以从一个Surface 得到 ANativeWindow。然后就像其 Java 语言的表弟，你可以锁定它，在软件中渲染它，然后把它解锁并发送。

若要从 native 代码创建 EGL 窗口 surface，你需要将 EGLNativeWindowType 的实例传递给eglCreateWindowSurface()。EGLNativeWindowType 是 ANativeWindow 的一个同义词，所以你可以自由将它们相互转换（cast）。

因为事实上，基本的"native窗口"类型只是 BufferQueue 生产者侧的封装，这不至于让你感到惊讶吧。

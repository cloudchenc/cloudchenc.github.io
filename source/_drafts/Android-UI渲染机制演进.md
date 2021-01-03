---
title: Android UI渲染机制演进
tags:
categories: Android
---

### UI渲染

#### 屏幕适配

屏幕适配的碎片化问题，通过 dp 和自适应布局来解决。

了解 px, dp, dpi, ppi, density等概念。

#### CPU 与 GPU

CPU（Central Processing Unit, 中央处理器）：是计算机系统的运算和控制核心，是信息处理、程序运行的最终执行单元

GPU（Graphics Processing Unit, 图形处理器）：是一种专门用于图像运算的处理器，在计算机系统中通常被称为"显卡"的核心部件就是GPU

UI组件在绘制到屏幕之前，都需要经过 Rasterization (栅格化) 操作，而这一操作非常耗时。引入GPU就是为了加快栅格化。

// 贴图

软件绘制使用 Skia 库，它是一款能在低端设备，如手机呈现高质量的2D跨平台图形框架，类似 Chrome、Flutter 内部使用的都是 Skia 库。

硬件绘制是通过底层软件代码，将 CPU 不擅长的图形计算转换成 GPU 专用指令，由 GPU 完成绘制任务。

#### OpenGL 与 Vulkan

OpenGL（Open Graphics Library）：一个跨平台的图形API，它为3D图形处理硬件指定了一个标准的软件接口。

OpenGL ES（Embedded Systems）：针对嵌入式设备的OpenGL规范的一种变体。Android支持多个版本的 OpenGL ES API。

Vulkan：一个低开销、跨平台的3D图形和计算API。Vulkan的目标是跨所有平台的高性能实时3D图形应用程序，旨在提供更多的性能和更均衡的CPU/GPU使用。

硬件加速绘制就是通过 GPU 来进行渲染，GPU 作为一个硬件，用户空间是无法直接使用的，它是由 GPU 厂商按照 OpenGL 规范实现的驱动间接进行使用。

也就是说，如果一个设备支持 GPU 硬件加速渲染，那么当 Android 应用程序调用 Open GL 接口来绘制 UI 时，Android 应用程序的 UI 就是通过硬件加速技术进行渲染的。

这主要是受 OpenGL ES 版本与系统支持的限制，直到 Android P，仍然有 3 个 API 是没有支持的。

对于不支持的 API，只能使用软件绘制模式，渲染的性能将会大大降低。

Android 7.0 把 OpenGL ES 升级到最新的 3.2 版本的同时，还添加了对 Vulkan 的支持。

Vulkan 的设计目标是取代 OpenGL，Vulkan 是一个相当低级别的 API，并且提供并行的任务处理。Vulkan 还能够渲染 2D 图形应用程序。

除了其较低的 CPU 使用率，Vulkan 还能够更好地在多个 CPU 内核之间分配工作。在功耗、多核优化提升绘图调用上有着非常明显的优势。

### Android的图形组件

// 贴图

Image Stream Produces：图像流生产方，应用程序内绘制到 Surface 的图形（xml / Java 实现的图形，或者视频）
Hardware Composer：硬件合成器，它是显示控制器的硬件抽象。
Gralloc：Graphic memory allocator 用来分配图形缓冲区内存。

如果把应用程序图形渲染过程当做一次绘画过程，那么绘画过程中 Android 的各个图形组件的作用是：

画笔：Skia 或者 OpenGL。我们可以用 Skia 画笔绘制 2D 图形，也可以用 OpenGL 来绘制 2D / 3D 图形。正如前面所说，前者使用 CPU 绘制，后者使用 GPU 绘制。
画纸：Surface。所有的元素都在 Surface 这张画纸上进行绘制和渲染。在 Android 中，Window 是 View 的容器，每个窗口都会关联一个 Surface。而 WindowManager 则负责管理这些窗口，并且把它们的数据传递给 SurfaceFlinger。
画板：Graphic Buffer。Graphic Buffer 缓冲用于应用于应用程序图形的绘制，在 Android 4.1 之前使用的是双缓冲机制；在 Android 4.1 之后，使用的是三缓冲机制。
显示：SurfaceFlinger。它将 WindowManger 提供的所有 Surface，通过硬件合成器 Hardware Composer 合成并输出到显示屏。

使用画笔 Skia / OpenGL 将内容绘制到 Surface 上，绘制的过程中如果使用 Open GL 渲染，那便是硬件加速，否则纯靠 CPU 绘制渲染栅格化的过程就叫软件绘制。

对于硬件绘制，我们通过调用 OpenGL ES 接口利用 GPU 完成绘制。

### 60 fps

12 fps：每秒达到 10 ~ 12 帧以上才可以被感知到运动及变化，但是这样的速率是非常不流畅的，只有帧率超过每秒 24 帧的时候，才会被察觉为流畅的运动及变化。
24fps：每秒 24 帧在电影界是黄金标准，24 帧的速度足够使画面运动的非常流畅，而且 24 帧的电影预算也能满足成本的要求，这也是为什么在过去的 50 年里，绝大多数的电影都使用 24 帧每秒的速率。

Android 系统每间隔 16ms 发出一次 VSYNC 信号，触发对 UI 的渲染任务；为了能够实现流畅的画面，这就意味着应该始终让应用保持在 60 帧每秒，即每帧工作的准备时间仅有 16ms。

### 渲染机制演进

#### Android 4.0 开启硬件加速

#### Android 4.1 Project Butter

#### Android 5.0 RenderThread

### 参考

https://mp.weixin.qq.com/s/psrDADxwl782Fbs_vzxnQg

---
title: Metal API 介绍
date: 2018-08-25 14:37:47
tags: Metal iOS 图形学
---

Hello World很可能是绝大多数程序员写的第一个程序，对于图形学程序员也有这样的一个Hello World，那就是画一个三角形。在开始编程之前，我们还需要了解一些其他的东西。

## 1、API

图形学编程中主要有以下几个API：OpenGL、Vulkan、Direct3D、Metal。下面分别介绍。

##### 1、 OpenGL

历史悠久的图形学API，由[Khronos Group](https://www.khronos.org)维护，拥有着最好的跨平台特性，几乎所有的主流平台都支持它，但因为一些历史原因，目前有些不太适应如今的多核时代，但如果你想让你的代码能在各个平台上运行，除了它以外别无选择。OpenGL有着极其丰富的教程资源，如果放在以前，我一定会选择它作来入门。

##### 2、Vulkan

Vulkan同样由[Khronos Group](https://www.khronos.org)维护，是基于AMD的[Mantle](https://zh.wikipedia.org/wiki/Mantle_(API))构建的，它与OpenGL相比，是一个更加底层的API，能充分利用多个CPU核心，有着更高的执行效率，当然学习曲线也更加陡峭。在跨平台方面，它不如OpenGL，是的，苹果不带它玩，你没法在macOS、iOS上直接使用它，但可以通过[MoltenVK](https://github.com/KhronosGroup/MoltenVK)来实现。

##### 3、Direct3D

Windows平台上的图形学API，如果你想从事PC游戏软件开发，学习Direct3D是一个很好的选择。

##### 4、Metal

Apple在2014年的WWDC上公布了Metal，它是一个兼顾图形与计算功能的，面向底层、低开销的API。

选择Metal作为入门的API有以下几个原因：

1、iOS、macOS虽然支持OpenGL，但实际上已经有几年的时间没有维护了,2018 WWDC苹果宣布废弃OpenGL，有兴趣的可以看看Hacker News上的[讨论](https://news.ycombinator.com/item?id=17231442)。我猜在macOS上Apple不敢彻底移除OpenGL，但在iOS上Apple是有可能的。  

2、在macOS上，与Metal相关的工具链很完善，每年都在不断的改进，另外作为一名iOS开发，各种工具都已经很熟悉了，开箱即用，不会在环境配置上浪费时间，可以把经精力集中在图形学的学习上。

3、Metal的设计与Vulkan上相似，都是比OpenGL更加低级的API，同时它的学习曲线比Vulkan平缓，学习Metal对日后学习Vulkan也有所帮助。

## 2、Rendering Pipeline

渲染实质就是这样一个过程，输入一组对象，输出一组像素。根据顺序的不同，渲染大致可以分为两类算法：image order rendering和object order rendering

image order rendering遍历每个像素，找到对这个像素有影响的所有物体，并计算颜色。光线追踪就是一种image order rendering算法。image order rendering渲染的优势是能实现照片级别的渲染，实现更为逼真的阴影、发射等效果。缺点就是计算量大，无法实现实时渲染。(2018.8.20，Nvidia发布了GeForce RTX 20 Series系列显卡，这是世界上首块支持实时光线追踪的显卡)

object order rendering则是遍历每个对象，找到所有受这个对象影响的像素，更新这些像素的颜色。它的优点是运行效率高，支持实时渲染。缺点就是渲染效果较差。

Metal是基于object order rendering，所以下面我们介绍基于object order rendering的渲染流水线。Metal流水线如下图所示:

<img src="metal-introduction/RenderingPipeline.jpg" width="150px" height="600px" alt="渲染流水线" title="[title]">  

1、渲染流水线开始前需要准备顶点相关的数据，比如顶点坐标、顶点颜色等。  

2、Vertex Shader阶段是可编程的，它接收前面提供的顶点数据，计算出一组新的顶点数据。Vertex Shader输出的顶点坐标是在裁剪空间下的。

3、Tessellation中文为曲面细分，它的作用简单的说就是产生更多的顶点，顶点数越多，则三角形越多，生成的图像越细腻。Tessellation阶段也是可编程的  

4、Vertex Post-Processing阶段主要是进行裁剪、透视除法、视口转换等操作

5、Primitive Assembly阶段是把前面输出的顶点数据集合在一起组成图元，输出一个有序的简单图元（点、线、三角形）序列。  

6、Rasterization光栅化，它是渲染流水线的核心阶段，遍历每个被图元覆盖的像素，用顶点数据对这些像素进行插值，它的输出是一组fragment。fragment是一组状态的集合，用于计算一个像素的最终数据。

7、Fragment Shader阶段是可编程的，它接受Rasterization输出的fragment，输出一个像素的颜色。  

8、Per-Fragment Processing是将Fragment Shader输出的结果在经过一些处理，包括Scissor Test、Stencil Test、Depth Test、Blending、Logical Operation、Write Mask。

在Metal中，不可编程的阶段通常通过设置一些状态值来控制其过程，比如我们可以选择是否开启Depth Test。而可编程阶段则需要我们写shader来进行控制。Metal使用的shader语言是Metal Performance Shaders(MPS)，它是基于C++14开发的。下面我们将用它来编写我们的第一个图形程序：Hello Triangle。

## 3、Hello Triangle

[坐标变换](2018/08/04/coordinate-transformation/)
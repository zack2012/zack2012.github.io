---
title: Metal API 介绍
date: 2018-08-25 14:37:47
categories: 图形学
tags: 
    - Metal
    - iOS
---

Hello World很可能是绝大多数程序员写的第一个程序，对于图形学程序员也有这样的一个Hello World，那就是画一个三角形。在开始编程之前，我们还需要了解一些其他的东西。本文的源码放在了[github](https://github.com/zack2012/MetalGraphics)上。

<!-- more -->

## 1. API
---

图形学编程中主要有以下几个API：OpenGL、Vulkan、Direct3D、Metal。下面分别介绍。

### 1.1 OpenGL

历史悠久的图形学API，由[Khronos Group](https://www.khronos.org)维护，拥有着最好的跨平台特性，几乎所有的主流平台都支持它，但因为一些历史原因，目前有些不太适应如今的多核时代，但如果你想让你的代码能在各个平台上运行，除了它以外别无选择。OpenGL有着极其丰富的教程资源，如果放在以前，我一定会选择它作来入门。

### 1.2 Vulkan

Vulkan同样由[Khronos Group](https://www.khronos.org)维护，是基于AMD的[Mantle](https://zh.wikipedia.org/wiki/Mantle_(API)构建的，它与OpenGL相比，是一个更加底层的API，能充分利用多个CPU核心，有着更高的执行效率，当然学习曲线也更加陡峭。在跨平台方面，它不如OpenGL，是的，苹果不带它玩，你没法在macOS、iOS上直接使用它，但可以通过[MoltenVK](https://github.com/KhronosGroup/MoltenVK)来实现。

### 1.3 Direct3D

Windows平台上的图形学API，如果你想从事PC游戏软件开发，学习Direct3D是一个很好的选择。

### 1.4 Metal

Apple在2014年的WWDC上公布了Metal，它是一个兼顾图形与计算功能的，面向底层、低开销的API。

选择Metal作为入门的API有以下几个原因：

1、iOS、macOS虽然支持OpenGL，但实际上已经有几年的时间没有维护了,2018 WWDC苹果宣布废弃OpenGL，有兴趣的可以看看Hacker News上的[讨论](https://news.ycombinator.com/item?id=17231442)。我猜在macOS上Apple不敢彻底移除OpenGL，但在iOS上Apple是有可能的。  

2、在macOS上，与Metal相关的工具链很完善，每年都在不断的改进，另外作为一名iOS开发，各种工具都已经很熟悉了，开箱即用，不会在环境配置上浪费时间，可以把经精力集中在图形学的学习上。

3、Metal的设计与Vulkan上相似，都是比OpenGL更加低级的API，同时它的学习曲线比Vulkan平缓，学习Metal对日后学习Vulkan也有所帮助。

## 2. Rendering Pipeline
---

渲染实质就是这样一个过程，输入一组对象，输出一组像素。根据顺序的不同，渲染大致可以分为两类算法：image order rendering和object order rendering

image order rendering遍历每个像素，找到对这个像素有影响的所有物体，并计算颜色。光线追踪就是一种image order rendering算法。image order rendering渲染的优势是能实现照片级别的渲染，实现更为逼真的阴影、发射等效果。缺点就是计算量大，无法实现实时渲染。(2018.8.20，Nvidia发布了GeForce RTX 20 Series系列显卡，这是世界上首块支持实时光线追踪的显卡)

object order rendering则是遍历每个对象，找到所有受这个对象影响的像素，更新这些像素的颜色。它的优点是运行效率高，支持实时渲染。缺点就是渲染效果较差。

Metal是基于object order rendering，所以下面我们介绍基于object order rendering的渲染流水线。Metal流水线如下图所示:

<img src="RenderingPipeline.jpg" width="150px" height="600px" alt="渲染流水线" title="渲染流水线">  

1、渲染流水线开始前需要准备顶点相关的数据，比如顶点坐标、顶点颜色等。  

2、Vertex Shader阶段是可编程的，它接收前面提供的顶点数据，计算出一组新的顶点数据。Vertex Shader输出的顶点坐标是在裁剪空间下的。

3、Tessellation中文为曲面细分，它的作用简单的说就是产生更多的顶点，顶点数越多，则三角形越多，生成的图像越细腻。Tessellation阶段也是可编程的  

4、Vertex Post-Processing阶段主要是进行裁剪、透视除法、视口转换等操作

5、Primitive Assembly阶段是把前面输出的顶点数据集合在一起组成图元，输出一个有序的简单图元（点、线、三角形）序列。  

6、Rasterization光栅化，它是渲染流水线的核心阶段，遍历每个被图元覆盖的像素，用顶点数据对这些像素进行插值，它的输出是一组fragment。fragment是一组状态的集合，用于计算一个像素的最终数据。

7、Fragment Shader阶段是可编程的，它接受Rasterization输出的fragment，输出一个像素的颜色。  

8、Per-Fragment Processing是将Fragment Shader输出的结果在经过一些处理，包括Scissor Test、Stencil Test、Depth Test、Blending、Logical Operation、Write Mask。

在Metal中，不可编程的阶段通常通过设置一些状态值来控制其过程，比如我们可以选择是否开启Depth Test。而可编程阶段则需要我们写shader来进行控制。Metal使用的shader语言是Metal Performance Shaders(MPS)，它是基于C++14开发的。

## 3. Metal里重要的接口、类
---

Metal是按面向接口设计的，核心功能都是通过接口提供。下面介绍Metal重要的接口、类。

__1. MTLDevice__  

MTLDevice是一个接口，它代表着一个GPU，在图形编程中，把GPU称做device，把CPU称作host。MTLDevice的主要作用就是创建其他重要的接口和类以及查询GPU一些参数

__2. MTLCommandQueue__  

MTLCommandQueue是一个用来管理command buffer的顺序队列，它是线程安全的。

__3. MTLRenderPipelineState__

MTLRenderPipelineState定义了渲染流水线的状态，比如设置vertex和fragment shader。创建MTLRenderPipelineState需要校验一系列状态，这些操作很耗时，所以应尽可能的早的创建MTLRenderPipelineState并复用它们。MTLRenderPipelineState需要用MTLRenderPipelineDescriptor来配置。Metal中有很多这样的Descriptor，用来配置信息。

__4. MTLCommandBuffer__

MTLCommandBuffer用来存储要提交到GPU执行的命令。一旦调用了commit()方法后，MTLCommandBuffer就不能在往里添加命令了。

__5. MTLRenderCommandEncoder__

 MTLRenderCommandEncoder是用来设置流水线状态和执行图形绘制的命令的协议。通常MTLRenderCommandEncoder需要执行以下任务:  
* 设置MTLRenderPipelineState。
* 设置提供给vertex shader和fragment shader需要的资源，比如顶点信息，坐标变化矩阵。
* 设置固定功能的管道(fixed-function state)，比如viewport，depth test，stencil test。
* 调用绘制命令(draw call)

__6. MTLRenderPassDescriptor__

MTLRenderPassDescriptor是一组渲染目标(render target)的集合。是一次render pass生成的像素的输出目标。这里有一个很重要的概念就是render pass。  render pass 在Apple文档里描述为更新一组渲染目标的命令集合。一张复杂的图像，可以通过渲染多遍来完成，每一遍只渲染图像的某些部分，最后把这些部分组合在一起形成最终的图像。这一遍就是一次render pass，所以有些图形软件也里也把render pass叫做render layer。在Metal中，一个MTLRenderCommandEncoder对应一次render pass。

__7. MTLTexture__  

MTLTexture是一片存储格式化图像数据的内存区域，可以被GPU访问。MTLTexure可以用作vertex shader和fragment shader的输入，也可以作为存储渲染流水线输出pixel的地方。

__8. CAMetalLayer__

在iOS和macOS上，需要通过CAMetalLayer来把图像显示在屏幕上。CAMetalLayer内部维护了一个用来在屏幕上显示内容的MTLTexture的池子。通过nextDrawable()方法得到一个MTLTexture，作为render pass的渲染目标。

在Metal中，对象被分为持久对象和瞬态对象。创建持久对象需要耗费大量的时间，这些对象应该尽可能早的创建并复用。MTLDevice、MTLCommandQueue、MTLRenderPipelineState就属于这类对象。

## 4. Hello Triangle 
---

下面只是部分源码，完整源码可以参考[github](https://github.com/zack2012/MetalGraphics)。

我们先创建一个用来表示顶点数据的struct，包含顶点位置(单位为像素)和颜色

```swift
struct Vertex {
    /// 顶点位置，单位像素
    var position: float4
    
    /// 顶点颜色，RGBA
    var color: float4
}
```

由前面可知，我们需要一个CAMetalLayer来显示内容，我们将其分装在一个自定义的UIView类里:

```swift
class HelloTriangleView: UIView {
    private var metalLayer: CAMetalLayer {
        return layer as! CAMetalLayer
    }
    
    override class var layerClass: AnyClass {
        return CAMetalLayer.self
    }
}
```

出于方便的考虑，我们把一些不属于View的内容也放在HelloTriangleView里，后面我们会它拆分为两个类，使view可以复用。

```swift
class HelloTriangleView: UIView {
    /// .....

    private var device: MTLDevice
    private var pipelineState: MTLRenderPipelineState!
    private var commandQueue: MTLCommandQueue!
    private var displayLink: CADisplayLink?
    
    /// 顶点数据，单位像素
    private let vertices: [Vertex] = [
        Vertex(position: float4(0, 250, 0, 1), color: float4(1, 0, 0, 1)),
        Vertex(position: float4(-250, -250, 0, 1), color: float4(0, 1, 0, 1)),
        Vertex(position: float4(250, -250, 0, 1), color: float4(0, 0, 1, 1)),
    ]
    
    /// 存储顶点数据的buffer
    private var vertexBuffer: MTLBuffer?
    
    /// 存储坐标变换矩阵的buffer
    private var matrixBuffer: MTLBuffer?
}
```

初始化方法如下: 

```swift
class HelloTriangleView: UIView {
    // ......
    
    override init(frame: CGRect) {
        self.device = MTLCreateSystemDefaultDevice()!
        
        let scale = UIScreen.main.scale
        
        // Metal不用point，而使用pixel，所以这里需要将point转换为pixel
        let drawableSize = frame.size.applying(CGAffineTransform(scaleX: scale, y: scale))
        
        // 设置顶点数据，三角形的中心在屏幕的原点
        let midX = Float(drawableSize.width / 2)
        let midY = Float(drawableSize.height / 2)
        vertices = [
            Vertex(position: float4(midX, midY + 250, 0, 1), color: float4(1, 0, 0, 1)),
            Vertex(position: float4(midX - 250, midY - 250, 0, 1), color: float4(0, 1, 0, 1)),
            Vertex(position: float4(midX + 250, midY - 250, 0, 1), color: float4(0, 0, 1, 1)),
        ]
        // ......
        
        // 设置从屏幕空间变化到裁剪空间的变换矩阵
        var mat = float4x4(diagonal: float4(Float(2 / drawableSize.width),
                                            Float(2 / drawableSize.height),
                                            1, 1))
        let length = MemoryLayout.stride(ofValue: mat)
        withUnsafePointer(to: &mat) {
            self.matrixBuffer = device.makeBuffer(bytes: $0,
                                                  length: length,
                                                  options: .storageModeShared)
        }
        
        // 获取vertex shader和fragment shader，用于设置render pipeline
        let library = device.makeDefaultLibrary()
        let vertexFun = library?.makeFunction(name: "helloTriangleShader")
        let fragmentFun = library?.makeFunction(name: "helloTriangleFragment")
        
        // 创建render pipeline
        let pipelineDesc = MTLRenderPipelineDescriptor()
        pipelineDesc.vertexFunction = vertexFun
        pipelineDesc.fragmentFunction = fragmentFun
        pipelineDesc.colorAttachments[0].pixelFormat = metalLayer.pixelFormat
        pipelineState = try! device.makeRenderPipelineState(descriptor: pipelineDesc)
        
        // 创建commandQueue
        commandQueue = device.makeCommandQueue()
    }
}
```

vertex shader要求输出的顶点坐标是在裁剪空间下，而我们提供的坐标是屏幕坐标，根据[坐标变换](2018/08/04/coordinate-transformation/)可知，从NDC变换到屏幕空间的矩阵为:

$$
M = \begin{pmatrix}
\frac{w}{2} & 0 & 0 & x+\frac{w}{2}\\
0 & \frac{h}{2} & 0 & y+\frac{h}{2}\\
0 & 0 & \frac{z_{max}-z_{min}}{2} & \frac{z_{max}+z_{min}}{2}\\
0 & 0 & 0 & 1\\
\end{pmatrix}
$$

因为只是在画一个二维图形，可以认为NDC与裁剪空间的坐标相同，视口的原点默认为0。所以我们有: 

$$
x=0
$$

$$
y=0
$$

$$
z_{max}=-z_{min}
$$

$$
w=1
$$

带入得到从裁剪空间到屏幕坐标系的变换矩阵:  

$$
M = \begin{pmatrix}
\frac{w}{2} & 0 & 0 & \frac{w}{2}\\
0 & \frac{h}{2} & 0 & \frac{h}{2}\\
0 & 0 & 1 & 0\\
0 & 0 & 0 & 1\\
\end{pmatrix}
$$

对这个矩阵求逆，得:

$$
M^{-1} = \begin{pmatrix}
\frac{2}{w} & 0 & 0 & -1\\
0 & \frac{2}{h} & 0 & -1\\
0 & 0 & 1 & 0\\
0 & 0 & 0 & 1\\
\end{pmatrix}
$$

绘制三角形的代码如下:

```swift
func redraw() {
    // 获取下一个空闲的texture，当作渲染目标
    guard let drawable = metalLayer.nextDrawable() else {
        return
    }
        
    let texture = drawable.texture
        
    // 设置render pass
    let passDesc = MTLRenderPassDescriptor()
    passDesc.colorAttachments[0].texture = texture
    passDesc.colorAttachments[0].loadAction = .clear
    passDesc.colorAttachments[0].storeAction = .store
        
    guard let commandBuffer = commandQueue.makeCommandBuffer() else {
        return
    }
        
    let encoder = commandBuffer.makeRenderCommandEncoder(descriptor: passDesc)
        
    encoder?.setRenderPipelineState(pipelineState)
        
    // 给shader vertex传入参数
    encoder?.setVertexBuffer(vertexBuffer, offset: 0, index: 0)
    encoder?.setVertexBuffer(matrixBuffer, offset: 0, index: 1)
        
    // 调用draw call，绘制三角形
    encoder?.drawPrimitives(type: .triangle, vertexStart: 0, vertexCount: 3)
    encoder?.endEncoding()
        
    // 尽可能早的展示绘制内容
    commandBuffer.present(drawable)
        
    // 提交commandBuffer，commandBuffer提交后就不能继续使用了
    commandBuffer.commit()
}
```
值得注意的有以下几点： 

* 设置render pass需要从CAMediaLayer获取可用于显示的texture。
* app给shader传参是通过ArgumentBuffer进行的，例如顶点数据设置在buffer 0，变换矩阵设置在buffer 1
* 调用drawPrimitives来完成绘制图像，metal只支持点、线、三角形这三种基本图元。
* 想要显示图像必须调用commandBuffer的present()方法。

最后是shader的代码:  

```c++
#include <metal_stdlib>
using namespace metal;

struct Vertex {
    float4 position [[position]];
    float4 color;
};

struct Uniforms {
    float4x4 modelViewProjectionMatrix;
};

vertex Vertex helloTriangleShader(device Vertex *vertices [[buffer(0)]],
                                  constant Uniforms *uniforms [[buffer(1)]],
                                  uint vid [[vertex_id]]) {
    Vertex out;
    auto vtx = vertices[vid];
    out.position = uniforms->modelViewProjectionMatrix * vtx.position;
    out.color = vtx.color;
    return out;
}

fragment float4 helloTriangleFragment(Vertex inVertex [[stage_in]]) {
    return inVertex.color;
}
```

Metal的shader是基于c++14的，所以看起来与c++有点相似。函数前的vertex、fragement关键字表示是vertex shader还是fragment shader。需要自定义结构体来接受app传入的数据，注意，这里定义的结构体内存布局应该与app传进来的内存布局相同。[[xxx]]表示属性，例如[[position]]表示该属性所修饰的变量为顶点坐标的位置，[[buffer(0)]]表示外部传入的第0个buffer，这里对应就是外部传入的顶点数据。Metal里还有很多属性，具体可以参考[官方文档](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf)

代码跑起来的结果如下:

<img src="triangle.jpg" width="300px" height="652px" alt="三角形" title="三角形"> 

## 5. 绘制立方体
---

上面我们完成了三角形的绘制，但绘制三角形过于简单，下面我们绘制一个可以旋转的立方体来完整的了解如何绘制一个3D图形。

在绘制三角形时，我们把绘制逻辑放在了view里，更好的做法是把绘制逻辑和view分离开来，使得view可以被复用。view将绘制委托给Renderer类，而view只提供render pass，代码如下:  

```swift
protocol CubeViewDelegate: class {
    func drawInView(_ view: CubeView)
}

class CubeView: UIView {
    // ......
    
    func renderPass() -> (desc: MTLRenderPassDescriptor, drawable: CAMetalDrawable?) {
        let drawable = metalLayer.nextDrawable()
        let passDesc = MTLRenderPassDescriptor()
        passDesc.colorAttachments[0].texture = drawable?.texture
        passDesc.colorAttachments[0].clearColor = MTLClearColor(red: 0, green: 0, blue: 0, alpha: 1)
        passDesc.colorAttachments[0].loadAction = .clear
        passDesc.colorAttachments[0].storeAction = .store
        
        passDesc.depthAttachment.texture = self.depthTexture
        passDesc.depthAttachment.clearDepth = 1
        passDesc.depthAttachment.loadAction = .clear
        passDesc.depthAttachment.storeAction = .dontCare
        
        return (passDesc, drawable)
    }
}
```

绘制立方体的流程与绘制三角形大体上是一样的。首页要准备顶点数据:

```swift
CubeRenderer {
    // ......

    // 顶点数据
    private static let vertices = [
        Vertex(position: float4(-1,  1,  1, 1), color: float4(0, 1, 1, 1)),
        Vertex(position: float4(-1, -1,  1, 1), color: float4(0, 0, 1, 1)),
        Vertex(position: float4( 1, -1,  1, 1), color: float4(1, 0, 1, 1)),
        Vertex(position: float4( 1,  1,  1, 1), color: float4(1, 1, 1, 1)),
        
        Vertex(position: float4(-1,  1, -1, 1), color: float4(0, 1, 0, 1)),
        Vertex(position: float4(-1, -1, -1, 1), color: float4(0, 0, 0, 1)),
        Vertex(position: float4( 1, -1, -1, 1), color: float4(1, 0, 0, 1)),
        Vertex(position: float4( 1,  1, -1, 1), color: float4(1, 1, 0, 1)),
    ]

    // 顶点数据索引
    private static let indices: [UInt16] = [
        3, 2, 6, 6, 7, 3,
        4, 5, 1, 1, 0, 4,
        4, 0, 3, 3, 7, 4,
        1, 5, 6, 6, 2, 1,
        0, 1, 2, 2, 3, 0,
        7, 6, 5, 5, 4, 7
    ]
}
```

metal只能绘制三角形，一个立方体需要绘制12个三角形，一共36个顶点，但确定一个立方体只要8个顶点就够了，因此需要顶点索引buffer，这样就不会浪费存储空间了。顶点索引是有顺序的，从外向内看去为逆时针。

另外，我们需要一个矩阵将物体空间变换到裁剪空间，代码如下: 

```swift
CubeRenderer {
    // ......

    private func updateUniformBuffer(view: CubeView) {
        let scaleFactor: Float = 0.8
        
        let rotate1 = Math.matrixRotation(axis: float3(1, 0, 0), angle: rotationX)
        let rotate2 = Math.matrixRotation(axis: float3(0, 1, 0), angle: rotationY)
        let scale = Math.matrixScale(scaleFactor)
        let translate = Math.matrixTranslate(x: 0, y: 0, z: -5)
        let size = view.metalLayer.drawableSize
        let apsect = Float(size.width / size.height)
        let projection = Math.matrixPerspective(aspect: apsect, fovy: 72.radian, near: 1, far: 100)
        let mat = projection * translate * rotate2 * rotate1 * scale
        
        let uniforms = Uniforms(modelViewProjectionMatrix: mat)
        let uniformRawBuffer = uniformBuffer?.contents()
        uniformRawBuffer?.storeBytes(of: uniforms,
                                     toByteOffset: MemoryLayout<Uniforms>.stride * bufferIndex,
                                     as: Uniforms.self)
    }
}
```

rotationX, rotationY由外部传入，代码中的数学公式已在[坐标变换](2018/08/04/coordinate-transformation/)里详细介绍了，这里就不再重复了。 shader代码也与绘制三角形一样。

最终，run起来的结果如下：

<img src="Cube.jpg" width="300px" height="652px" alt="立方体" title="立方体"> 

## 6. 总结
---

本文简要介绍了现代GPU的渲染流水线和Metal API，完成了绘制三角形和立方体，其中在绘制立方体时可以看到，[坐标变换](2018/08/04/coordinate-transformation/)是非常基础的知识，一定要牢牢掌握。

---
title: Model I/O 介绍
date: 2018-10-07 09:35:36
tags: 
    - iOS
    - Metal
    - Model I/O
---

在前面的几篇文章中我们都是手动输入顶点数据(立方体)或者使用代码生成的方式(球面)创建模型，可以看到这种创建模型的方式非常繁琐且极其局限，无法在真正的工程里使用，你能想象用这种方式创建一个人体模型吗？

从程序员的角度来看，模型数据属于一种资源文件，就像图片资源那样，由专业的人员创建，保存成某种格式的文件，该文件包含了一切必要的数据，例如，顶点坐标、顶点法向量等，我们只要读取这种文件后就可以直接用来绘制。

本文主要介绍上面提到的3D模型格式以及Apple官方提供的读取模型的库Model I/O，最后通过Model I/O绘制一个复杂的模型。完整的源码可以参考[github](https://github.com/zack2012/MetalGraphics)。

## 1、3D文件格式

3D文件格式种类繁多，就像图片文件格式一样，这里不打算全部介绍一遍，只介绍我认为比较重要的格式。


##### 1.1 Alembic

文件后缀名为.abc, 由大名鼎鼎的[工业光魔ILM](https://baike.baidu.com/item/工业光魔公司/7652141?fromtitle=ILM&fromid=44269&fr=aladdin)、Sony Pictures与Imageworks共同开发的一种开放格式。该格式主要解决的问题是在不同的软件之间共享复杂的动态场景，Alembic不仅能导出模型，还能导出特效，最早开发出来是用在电影cg特效里，后面游戏引擎也开始支持该格式。

##### 1.2 glTF

文件后缀名为.gltf，glTF是一种可以减少3D格式中与渲染无关的冗余数据并且在更加适合OpenGL簇加载的一种3D文件格式。它的提出是源自于3D工业和媒体发展的过程中，对3D格式统一化的急迫需求。在其[官网](https://www.khronos.org/gltf/)里把glTF比作三维文件的JPEG。

glTF使用json格式进行描述，也可以编译成二进制的内容：bglTF。glTF可以包括场景、摄像机、动画等，也可以包括网格、材质、纹理，甚至包括了渲染技术（technique）、着色器以及着色器程序。同时由于json格式的特点，它支持预留一般以及特定供应商的扩展。

##### 1.3 STL

文件后缀名为.stl，STL文件是在计算机图形应用系统中，用于表示三角形网格的一种文件格式。它格式非常简单，只能用于描述三维物体的几何信息，不支持颜色材质等信息。该格式最早发布于1987年，运用领域非常广泛。

##### 1.4 OBJ

文件后缀名为.obj，OBJ文件是Alias|Wavefront公司为它的一套基于工作站的3D建模和动画软件"Advanced Visualizer"开发的一种标准3D模型文件格式，很适合用于3D软件模型之间的互导。目前几乎所有知名的3D软件都支持OBJ文件的读写。OBJ文件是一种文本文件，可以直接用写字板打开进行查看和编辑修改。

OBJ文件除了包含基本模型数据，还附带UV信息及材质路径，但它不包含动画、材质特性、贴图路径、动力学、粒子等信息。主要支持多边形(Polygons)模型。是最受欢迎的格式。

##### 1.5 PLY

文件后缀名为.ply，PLY格式全称为Polygon File Format或者 Stanford Triangle Format。它受Wavefront .obj格式的启发，但改进了Obj格式所缺少的对任意属性及群组的扩充性。因此PLY格式发明了"property"及"element"这两个关键词，来概括“顶点、面、相关资讯、群组”的概念。

PLY主要用以储存立体扫描结果的三维数值，透过多边形片面的集合描述三维物体，与其他格式相较之下这是较为简单的方法。它可以储存的资讯包含颜色、透明度、表面法向量、材质座标与资料可信度，并能对多边形的正反两面设定不同的属性。

##### 1.6 USD

文件后缀名为.usd，USD全称为Universal Scene Description，该格式由Pixar开发。对于需要稳定地，可扩展地交换和增强由一系列基本asset组成的任意3D场景而言，USD可以满足这种需求。[官网](https://graphics.pixar.com/usd/docs/index.html)有该格式的详细介绍。

WWDC 2018上，苹果宣布与Pixar共同开发了一款新的格式USDZ，USDZ实质上就是USD格式的压缩版，最后的Z代表着zip，苹果希望USDZ成为AR领域的通用格式。

## 2、Model I/O

Model I/O是苹果在2015年推出的一款处理3D模型的框架，它不仅可以用来导入、导出、操作3D模型，还可以用来描述灯光,材料和环境，烘焙灯光，细分网格,提供基于物理效果的渲染。Model I/O与苹果其他的框架(SceneKit、Metal)集成的很好，使用起来非常简单。在开发过程使用Model I/O如下图所示：

<img src="ModelIO/ModelIO.jpg" width="800px" height="400px" alt="Model I/O" title="Model I/O"> 

Model I/O功能强大，这里我们不做全面的介绍，只介绍与导入模型相关的内容，等到以后用到其他功能时再详细展开。

MDLAsset是用来存储3D模型和其他相关数据的容器，使用起来非常简单：

```swift
 let asset = MDLAsset(url: url, vertexDescriptor: vertexDescriptor, bufferAllocator: bufferAllocator)
```

创建MDLAsset时需要设置MDLVertexDescriptor和MDLMeshBufferAllocator，否则在接下来创建MTKMesh时会失败。

MDLVertexDescriptor是用来描述顶点数据buffer结构、类型、布局的类。我们知道顶点数据不仅包括顶点的位置坐标，还有包含很多其他的数据，我们把这些数据称为属性(attribute)，例如，一个顶点数据可以有坐标属性、法向量属性、颜色属性等，我们可以任意的往顶点数据里添加合适的属性。当应用向GPU传输这些顶点数据时，或者更具体的说，向vertex shader函数传输参数时，是通过一块块连续的内存空间来完成，也就是vertex buffer。

为了方便理解，我们举个例子，假设现在我们有两个顶点A、B，每个顶点有两个属性：
  1、位置属性position，类型为float3。
  2、法向量属性normal，类型为float3。  
  
float3由simd库提供，每个float3由3个float类型组成，在内存中占据3 * 4 = 12个字节。我们将A、B两个顶点数据存续到如下的buf里
     A                   B
|---------|---------|---------|---------|
 position   normal   position   normal
|---------|---------|---------|---------|

<--------------- 24 bytes --------------->

通过下面代码向vertex shader函数传输参数：

```swift
let verticsBuffer = device.makeBuffer(bytes: &buf, length: 24, options: .storageModeShared)
encoder.setVertexBuffer(verticsBuffer, offset: 0, index: 0)
```  

vertex shader对应的函数签名为：

```c++
struct VertexOut {
    float4 position [[position]];
    float4 color;
};

struct VertexInput {
    float3 position;
    float3 normal;
};

vertex VertexOut vertexShader(uint vid [[vertex_id]],
                              device VertexInput *vertics [[buffer(0)]],
                             )
```

在应用程序设置的index，对应vertex shader里的[[buffer(index)]]，[[vertex_id]]表示则当前第几个顶点。这种传输方式有着一个明显的缺点，不够灵活，所以Metal给了另外一种传输方式：

```swift
encoder.setVertexBuffer(verticsBuffer, offset: 0, index: 0)
```

```c++
struct VertexOut {
    float4 position [[position]];
    float4 color;
};

struct VertexInput {
    float3 position [[attribute(0)]];
    float3 normal [[attribute(1)]];
};

vertex VertexOut vertexShader(VertexInput vertexIn [[stage_in]])
```

无论采取哪种方式传输数据，应用程序最终都是通过setVertexBuffer来向vertex shader传输参数的。那么vertex shader如何获取数据生成VertexInput结构体？一是在应用程序里提供MTLVertexDescriptor，二是在vertex shader添加[[stage_in]]标记。代码如下

```swift
let renderPipelineDesc = MTLRenderPipelineDescriptor()

let mtlVertexDesc = MTLVertexDescriptor()
// position
mtlVertexDesc.attributes[0].format = .float3
mtlVertexDesc.attributes[0].offset = 0
mtlVertexDesc.attributes[0].bufferIndex = 0

// normal
mtlVertexDesc.attributes[1].format = .float3
mtlVertexDesc.attributes[1].offset = 12
mtlVertexDesc.attributes[1].bufferIndex = 0

// layout
mtlVertexDesc.layouts[0].stride = 24
mtlVertexDesc.layouts[0].stepFunction = .perVertex

renderPipelineDesc.vertexDescriptor = mtlVertexDesc
```


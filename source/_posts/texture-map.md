---
title: 纹理映射
date: 2018-10-18 07:53:53
mathjax: true
categories: 图形学
tags:
    - Texture
---

现实世界中的物体表面都有着丰富的细节，例如一块石头表面凹凸不平。这就要求我们在绘制图像时每个像素的属性都应该随着物体表面位置的变化而变化。我们可以通过增加模型的顶点数来达到这样的效果，但这样做效率不高，即使一个简单物体可能都需要非常复杂的模型。而使用纹理映射我们可以在不增加模型复杂度的前提下达到同样的效果。完整的源码可以参考[github](https://github.com/zack2012/MetalGraphics)。

## 1. 纹理映射

纹理映射(Texture Mapping)就是用一张图像存储物体表面的细节，这里的细节可以是渲染方程里的各种参数值，比如diffuse color、normal等，在绘制物体时，将这张图映射到物体表面上。我们可以把这个过程想象为用纸去包装礼物。

纹理映射用数学语言描述如下：

$$
f: (x,y,z)\mapsto(u,v)
$$

$$
u,v\in[0, 1] 
$$

纹理映射f将物体表面坐标(x,y,z)映射到纹理空间(texture space)坐标(u,v)。

一个好的纹理映射一般要尽可能的满足如下需求：
  1、__双射性__。通常我们要求表面上的点要映射到纹理空间中不同的点，除非是想要在表面上重复这个纹理。
  2、__尺寸不畸变__。表面上靠近的两个点在映射到纹理空间时也应该保持差不多大小的距离。
  3、__形状不畸变__。纹理图片不能畸变的太厉害。比如物体表面上一个圆形图案，在映射到纹理空间时不能变为一个方形，或者被拉伸的太厉害。
  4、__连续性__。表面上相邻的两个点在映射到纹理空间时也应该保持相邻。

那么这样的纹理映射如何获得呢？一般都是通过专业的软件在建模时获得。这些专业的软件通过给模型表面添加缝(seam)，将物体表面展开成平面图形(uv unwrapping)，给展开后的每个顶点附上(u,v)坐标，随着模型一起导出。这个[视频](https://www.youtube.com/watch?v=scPSP_U858k)(需fq)就是使用blender来设计纹理映射。

<img src="cow.jpg" width="800px" height="400px" alt="cow uv unwrapping" title="cow uv unwrapping"> 

通常情况下(u,v)坐标的范围在[0,1]之间，但也有一些情况下(u,v)坐标的值可以超出这个范围，超出这个范围时我们有以下几种处理方式：
  1、__Clamp-to-Edge__。这种方式将纹理边缘的颜色不断重复。
  2、__Clamp-to-Zero__。这种方式将超出的部分的值设为0，也就是黑色。
  3、__Repeat__。这种方式将整个纹理不断重复。
  4、__Mirrored Repeat__。这种方式先将整个纹理镜像后在重复。


4种方式的示意图如下，源纹理图像是4x4黑白相间的图像。

<img src="clamp-to-edge.jpg" width="300px" height="300px" alt="clamp-to-edge" title="clamp-to-edge"> 

<img src="clamp-to-zero.jpg" width="300px" height="300px" alt="clamp-to-zero" title="clamp-to-zero"> 

<img src="repeat.jpg" width="300px" height="300px" alt="repeat" title="repeat"> 

<img src="mirrored-repeat.jpg" width="300px" height="300px" alt="mirrored repeat" title="mirrored repeat"> 

Metal中使用MetalKit加载texture很简单，代码如下：

```swift
func loadTexture(device: MTLDevice, imageName: String) throws -> MTLTexture {
    let textureLoader = MTKTextureLoader(device: device)
    let textureLoaderOptions: [MTKTextureLoader.Option: Any] = [
        .origin: MTKTextureLoader.Origin.bottomLeft
    ]
    
    let fileExtension = URL(fileURLWithPath: imageName).pathExtension.isEmpty ? "png" : nil
    
    guard let url = Bundle.main.url(forResource: imageName, withExtension: fileExtension) else {
        fatalError()
    }
    
    let texture = try textureLoader.newTexture(URL: url, options: textureLoaderOptions)
    
    return texture
}
```

Metal里的texture默认原点是在左上方，可以通过设置改变原点位置。如果不使用MetalKit来加载texture，代码如下：

```swift
func loadTexture(device: MTLDevice, imageName: String) throws -> MTLTexture {    
    let fileExtension = URL(fileURLWithPath: imageName).pathExtension.isEmpty ? "png" : nil

    guard let url = Bundle.main.url(forResource: imageName, withExtension: fileExtension) else {
        fatalError()
    }

    let image = UIImage(contentsOfFile: url.path)!
    let cgImage = image.cgImage!
    let width = cgImage.width
    let height = cgImage.height
    let bytesPerPixel = 4
    let bitsPerComponent = 8
    let bytesPerRow = cgImage.bytesPerRow
    let colorSpace = CGColorSpaceCreateDeviceRGB()
    
    let rawData = UnsafeMutableRawPointer.allocate(byteCount: width * height * bytesPerPixel, alignment: MemoryLayout<UInt8>.alignment)
    defer {
        rawData.deallocate()
    }

    let bitmapInfo = CGImageAlphaInfo.premultipliedLast.rawValue | CGBitmapInfo.byteOrder32Big.rawValue
    let context = CGContext(data: rawData,
                            width: width,
                            height: height,
                            bitsPerComponent: bitsPerComponent,
                            bytesPerRow: bytesPerRow,
                            space: colorSpace,
                            bitmapInfo: bitmapInfo)!

    // 调整原点到左下角
    context.translateBy(x: 0, y: CGFloat(height))
    context.scaleBy(x: 1, y: -1)

    context.draw(cgImage, in: CGRect(x: 0, y: 0, width: width, height: height))

    let textureDesc = MTLTextureDescriptor.texture2DDescriptor(pixelFormat: .rgba8Unorm_srgb, width: width, height: height, mipmapped: false)
    textureDesc.usage = .shaderRead
    
    let texture = device.makeTexture(descriptor: textureDesc)!
    
    let region = MTLRegionMake2D(0, 0, width, height)
    texture.replace(region: region, mipmapLevel: 0, withBytes: rawData, bytesPerRow: bytesPerRow)
    
    return texture
}
```
加载完texture后通过下面代码向shader传送：

```swift
encoder.setFragmentTexture(cowTexture, index: 0)
```

对应的shader代码如下：

```metal
fragment float4 textureFragment(VertexOut inVertex [[stage_in]],
                                texture2d<float> baseColorTexture [[texture(0)]]) {
   
    // ...

    constexpr sampler textureSampler(filter::linear, address::repeat);
    float3 baseColor = baseColorTexture.sample(textureSampler, inVertex.uv).rgb;
    
    // ...
}
```

在shader里我们使用constexpr修饰sampler，这样textureSampler只会创建一次。sample也可以通过应用创建后在传给shader，代码如下：

```swift
let sampleDesc = MTLSamplerDescriptor()
sampleDesc.sAddressMode = .repeat
sampleDesc.tAddressMode = .repeat
self.sampleState = device.makeSamplerState(descriptor: sampleDesc)!

// ...

encoder.setFragmentSamplerState(sampleState, index: 0)

```

对应的shader代码如下:

```metal
fragment float4 textureFragment(VertexOut inVertex [[stage_in]],
                                texture2d<float> baseColorTexture [[texture(0)]],
                                sampler textureSampler [[sampler(0)]]) {
    // ...
}
```

run起来的效果如下:

<img src="cow-run.jpg" width="300px" height="652px" alt="cow" title="cow"> 

## 2. mipmaps

我们把texture上的每个像素称为纹素(texel)，如果屏幕上的像素远大于纹素，即一个像素包含许多个纹素，我们把这种情况称为magnification，反之，如果一个纹素包含许多像素，则这种情况称为minification。

一个像素如何决定使用哪个纹素的值？通常有两种做法：最近选取(nearest)和线性滤波(linear)。最近选取就是取离像素中心最近的一个纹素的值，而线性滤波则是取像素周围的的纹素做加权平均后的值。

在magnification情况下，使用最近选取会使图像看起来一块一块的，使用线性滤波则会增大计算量(一个像素里包含许多的纹素)，为了解决这个问题，引入了mipmaps。

mip是拉丁文multum in parvo的缩写，对应的英文为much in small。它由一组texture组成，通常情况下，下一级的分辨率是上一级的一半，例如：Level 0: 64 x 64, 1: 32 x 32, 2: 16 x 16, 3: 8 x 8, 4: 4 x 4, 5: 2 x 2, 6: 1 x 1。示意图如下：

<img src="mipmapsLevel.jpg" width="500px" height="250px" alt="mipmaps level" title="mipmaps level"> 

通过使用mipmaps不仅可以提高绘制效率，还能带来更好的绘制效果:

<img src="mipmaps.jpg" width="800px" height="400px" alt="mipmaps" title="mipmaps"> 

## 3. 凹凸贴图(Bump Mapping)

凹凸贴图是一组贴图技术的总称，该组贴图技术通过修改渲染方程中的一些参数，在不增加增加额外的几何信息的前提下，就可以达到增强被渲染物体的表面细节的效果，使表面看起来更加真实。

最早的Bump Mapping通过一张高度贴图(Height Map)记录各像素点的高度信息，通过高度信息计算高度贴图中当前像素与周围像素的高度差，这个高度差就代表了各像素的坡度，用这个坡度信息去绕动法线，得到最终法线。

### 3.1 法线贴图(Normal Mapping)

法线贴图直接将法线存储到到贴图里，使用时不需要像高度贴图那样要再经过额外的计算。法线贴图一般可以通过高多边形模型生成，也可以通过高度贴图计算得到。

<img src="normalMap.jpg" width="400px" height="300px" alt="制作法线贴图" title="制作法线贴图">   

法线贴图里法线坐标是在切空间里，所谓的切空间就是由该点所有切线组成的线性空间。为什么不将法线坐标表示在物体空间或世界空间里？这样在计算光照时就可以少一步变换。在切空间里表示法线有两个好处：  
 * 可以对贴图进行压缩，减少内存占用和传输的带宽。在切空间里，法线的z轴总是正值，同时法线为单位向量，所以我们只要保存两个值(x,y)就可以了，而使用世界空间或物体空间，最少也需要保存3个值。
 * 方便复用，切空间里的法线贴图独立于物体几何形状。当物体进行镜像、旋转、缩放和平移操作时，切空间里法线贴图自动对其，而在物体空间里的法线贴图只支持后两种操作。

法线坐标x,y,z的取值范围为:

$$
x,y\in[-1,1]
$$

$$
z\in[0,1]
$$

在存储到法线贴图里时需要经过编码，将其转换到RGB空间里[0,255]。转换公式如下：

$$
rgb=normal.xyz / 2.0 + 0.5
$$

因为z轴总为正值，所以法线贴图看起来整体偏蓝色。法线贴图并没有改变物体的几何形状，所以在生成阴影时会有问题，另外从某个角度看过去时，会显示成平面。

### 3.2 视差贴图(Parallax Mapping)

视差贴图是对法线贴图的一种改进。使用法线贴图时并没有把视线的方向考虑进去，在真实世界中，如果物体表面高低不平，当视线方向不同时，看到的效果也不相同。

<img src="parallaxMap.jpg" width="600px" height="300px" alt="视差贴图" title="视差贴图">   

如果物体表面是凹凸不平的，那我们看到的点应该是$p_{ideal}$，但因为我们使用贴图并没有改变物体的表面，所以我们看到的点是$p$。

视差贴图并不是精确计算$p_{ideal}$的位置，而是对其进行估算。

估算的公式为：

$$
p_{adj}=p+\frac{h \cdot v_{xy}}{v_z}
$$

$v_{xy}$是观察向量在xy平面上的投影。$v_z$是观察向量的z轴分量。视差贴图的推导是基于相邻的点高度相同这个假设的，如果一个点的高度非常陡峭，即相邻两个点的高度差值非常大，则视差贴图就不能用了。

使用视差贴图可以使图像具有更加明显的深度和真实感。

### 3.3 视差遮蔽贴图(Parallax Occlusion Mapping)

视差遮蔽贴图是对视差贴图的改进，它精确的计算与高度贴图定义出来的表面交点，即$p_{ideal}$。视差遮蔽贴图的算法有很多种，对其实现感兴趣的话可以参考[这篇文章](https://segmentfault.com/a/1190000003920502)  

## 4. 总结

纹理映射可以说是图形绘制流水线里的一个核心概念，它在不增加太多计算量的同时，极大增强了图像的真实感，丰富了物体表面的细节。这里只是简单的介绍了纹理映射，以后我们还会再回来更加深入的讨论纹理映射。
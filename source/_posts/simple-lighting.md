---
title: 简单光照
date: 2018-09-09 09:40:51
mathjax: true
categories: 图形学
tags: 
    - Metal
---

要想让物体看起来更加栩栩如生，富有立体感，我们就需要引入光照模型。光照模型是对光线与物体表面之间相互作建模。一个物体即可能发射光线，也可能反射来自其他表面的光线，当观察物体表面上的一点时，所看到的颜色是取决于多个光源和多个反射表面之间的多次相互作用。

<!-- more -->

光照模型可以分为两类：全局光照和局部光照。在全局光照模型中，物体表面上一点的颜色不仅受到场景内的光源、物体表面材质等的影响，还受到物体之间相互反射的影响，这些反射可以是多次的。局部光照与此相反，物体表面上一点的颜色只与光源、物体表面材质、物体表面的局部几何性质有关，与场景中的其他表面无关，因此全局光照模型的计算要比局部光照模型复杂的多，但渲染出的效果会更加真实。这里我们只考虑局部光照模型。本文的源码放在了[github](https://github.com/zack2012/MetalGraphics)上。

## 1. 光源
---

光源可以分为以下几种模型：

* 环境光(Ambient Light)  
环境光使场景内获得均匀的光照。场景内的物体有时并没有受到光源的直接照射，而我们仍然可以看到这些物体，这是因为物体之间的相互反射，环境光就是对这种现象的一种近似。

* 点光源(Point Light)  
点光源向所有的方向发射的光线强度相等。点光源相对简单，因此大多数应用都使用它。仅利用点光源来照明往往会使场景的亮暗反差太大。我们可以通过为场景设置环境光来减弱由点光源引起的过高的对比度问题。

* 聚光登(Spotlight)  
聚光灯的特点是只在一个小的角度范围内发射光线。可以通过限制点光源发射光线的角度来构造一个聚光灯模型。

* 远距离光源(Distant Light)  
当光源距离物体很远时，物体上每个点的入射光线方向都相同，这样的光源就是远距离光源。例如，太阳光对一个物体来说就是远距离光源。

## 2. 物体3种基本的光学性质
---

光线和材质之间的作用可以分为3类：  
* 镜面反射(specular)  
一个物体如果具有镜面反射的性质，则它看起来是有光泽的，这是因为被反射或者散射(scatter)出去的大多数光线的方向都和反射角都和入射角相近。理想镜面反射的光线反射角等于入射角。

* 漫反射(diffuse)  
漫反射把入射光线向各个方向散射，理想的漫反射向各个方向的散射光想强度都相等，因此不同位置的观察者看到反射光线都一样。

* 折射(refraction)  
折射是指光线进入物体并从物体的另一个位置再发射出去。具有折射性质的表面看起来是半透明的。

## 3. Shading Model
---

### 3.1 Lambertian Shading Model
Lambertian Shading Model描述物体表面上一点的颜色正比与光源与该点法向量夹角的余弦值:  

$$
c\propto \cos\theta
$$

示意图如下: 

<img src="Lambertian.jpg" width="300px" height="200px" alt="Lambertian Shading Model" title="Lambertian Shading Model"> 

用向量表示:  

$$
c\propto \vec{n}\cdot\vec{l}
$$

上面公式需要注意的点有:  
* 向量在世界坐标系下。下面所有公式中的向量也都是在世界坐标系下。
* 向量是单位向量。

给上面的比例公式加上散射系数和光源强度就可以得到Lambertian Shading Model公式:  

$$
c=c_dl_d\vec{n}\cdot\vec{l} \qquad (3.1.1)
$$

考虑到物体可能背向光源，则$\vec{n}\cdot\vec{l}$为负值，因此还需要对(3.1.1)公式做些限制:  

$$
c=c_dl_d\space max(0,\vec{n}\cdot\vec{l}) \qquad (3.1.2)
$$

渲染的效果如下图所示: 

<img src="diffuse.jpg" width="300px" height="652px" alt="Lambertian Shading Model" title="Lambertian Shading Model"> 

### 3.2 Ambient Shading

从上面的渲染图可以看出，如果只使用diffuse shading，则光源照不到的地方都是黑色的，而在真实环境中，因为到处都是光的反射，所以我们也可以看到光源照不到的地方，因此我们需要对(3.1.2)公式做适当的修改。

我们可以为(3.1.2)公式增加一个常量，这个常量被叫做环境光，公式如下:

$$
c=c_al_a+c_dl_d\space max(0,\vec{n}\cdot\vec{l}) \qquad (3.2.1)
$$

$l_a$是环境光强度，$c_a$是环境光系数。引入环境光后的渲染图如下: 

<img src="ambient.jpg" width="300px" height="652px" alt="Ambient Shading Model" title="Ambient Shading Model"> 

### 3.3 Phone Shading

一些物体表面看起来会有光泽，这些光泽是由镜面反射产生的。Phone提出了一个近似模型，该模型的计算量只比漫反射多一点。该模型公式如下:  

$$
c=c_rl_r\space max(0, (\vec{r}\cdot\vec{e})^p) \qquad (3.3.1)
$$

其中$\vec{r}$是理想镜面反射光线的单位向量，$\vec{e}$是观察者方向的单位向量，如下图所示: 

<img src="Phong.jpg" width="300px" height="260px" alt="Phong Shading Model" title="Phong Shading Model"> 

从上图中可以推导出:

$$
\vec{r}=-\vec{l}+2(\vec{l}\cdot\vec{n})\vec{n} \qquad (3.3.2)
$$

这样我们就得到了完整Phong模型的计算公式。

我们也可以对Phong模型做适当的修改，如下图所示:

<img src="Blinn-Phong.jpg" width="300px" height="260px" alt="Blinn-Phong Shading Model" title="Blinn-Phong Shading Model"> 

$\vec{h}$是位于$\vec{l}$, $\vec{e}$的正中间的单位向量，因此有

$$
\vec{h}=\frac{\vec{l}+\vec{e}}{|\vec{l}+\vec{e}|}
$$

$$
\theta+w=\sigma+\theta-w
$$

化简得

$$
2w=\sigma
$$

所以我们可以用$\vec{n}\cdot\vec{h}$来代替$\vec{r}\cdot\vec{e}$，则可以得到改进后的模型

$$
c=c_rl_r\space max(0, (\vec{h}\cdot\vec{n})^p) \qquad (3.3.3)
$$

注意(3.3.3)里的$p$与(3.3.2)的$p$并不是同一个值。改进后的模型也被称为Blinn-Phong模型。

这样我们就得到一个比较完整的渲染公式

$$
c=c_al_a+c_dl_d\space max(0,\vec{n}\cdot\vec{l})+c_rl_r\space max(0, (\vec{h}\cdot\vec{n})^p) \qquad (3.3.4)
$$

使用(3.3.4)公式渲染得到的效果图如下

<img src="Blinn-Phong.jpeg" width="300px" height="652px" alt="Blinn-Phong Shading Model" title="Blinn-Phong Shading Model"> 

## 4. 总结
---

本文介绍了Blinn-Phong光照模型，该模型虽然简单，但却有着很好的效果。完整的代码可以参考[github](https://github.com/zack2012/MetalGraphics)

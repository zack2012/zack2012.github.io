---
title: 如何通过Swift Package Manager导入C库
date: 2018-10-30 07:35:09
categories: swift
tags: 
    - Swift
    - Swift Package Manager
    - vulkan
---

Swift作为一门2014年才正式公布的新语言，各种类型的库非常的匮乏，github上绝大多数的Swift库也主要是用于iOS上的，更糟糕的是，Swift从2014年发布以来，每一年的大版本升级都不向后兼容，导致一个库6个月以上没维护就编译不过去。好在Swift可以直接调用大部分的C库，一定程度上缓解了这个问题。

本文主要介绍通过Swift Package Manager(SwiftPM)包管理器导入c库，这里我们先简要的介绍下SwiftPM。

## 1、Swift Package Manager

SwiftPM作为官方包管理器，随着Swift一起发布。SwiftPM最早由Max Howell开发，你很可能不认识他，但你一定用过它写的包管理工具[homebrew](https://brew.sh)，可惜的是他后面与Apple闹掰了，离开Apple没有再为SwiftPM贡献了。SwiftPM虽然由Apple负责开发，但它目前不支持iOS、watchOS、TVOS，主要的应用领域是编写命令行工具和服务端的开发。

SwiftPM的DSL就是Swift，只要你会Swift，你就会SwiftPM，不像cocoapods，如果你需要自定义一些操作时，需要学习ruby。但SwiftPM完美的继承了Swift的缺点，每年都来个大改动，还不一定向后兼容，作为包管理器这就很难让人接受了。频繁变化的后果就是缺乏文档，或者文档过时，这里唯一推荐的文档就是[官方文档](https://github.com/apple/swift-package-manager/blob/master/Documentation/Usage.md)。SwiftPM本身支持xcode，可以通过`swift package generate-xcodeproj`生成xcode文件，但很难用，每添加一个文件或这更新依赖就得重新生成一次。值得庆幸的是，宇宙第一IDE厂商即将发布的新版clion将原生支持SwiftPM，具体可以看这篇[blog](https://blog.jetbrains.com/objc/2018/10/spm-support-clion/)。

SwiftPM最重要的两个概念就是target和product了。熟悉xcode的话，对这两个概念应该都不陌生，在一个iOS工程里，编译后生成的product就是一个app，而这个app可以由多个target组成，每个target都是一个独立的编译单元。在SwiftPM里，product里可以是一个库或者是一个可执行文件。

SwiftPM对文件结构是有要求的，一个典型的SwiftPM目录如下:

- root
    - Package.swift
    - Sources
        - Foo
            - a.swift
            - b.swift
        - Bar
            - c.swift
            - d.swift
    - Tests

原文件放在Sources目录下，Sources下的每个子文件夹都是一个Target。最开始SwiftPM不支持自定义文件目录布局，但在以后的实践中发现，不支持自定义文件目录很不方便，尤其是在导入一些c库时，这些c库有着很复杂的文件结构，所以在后面的版本里又加上自定义文件目录的功能。

我们可以用下面命令生成一个支持SwiftPM的工程:

```bash
swift package init --type=executable
```

type类型有三种empty、library、executable。三种类型都很简单，就不做解释了。值得提一句的是：当忘记或想查询某些命令的参数时，最简单的方式就是通过自带的help，例如：

```bash
swift package init --help
```
现在我们来看以下一个典型的Package.swift应该怎么编写，下面是一个简单的例子：

```swift
// swift-tools-version:4.2

import PackageDescription

let package = Package(
    name: "demo",
    products: [
        .library(
            name: "Foo",
            targets: ["Foo"]),
        
        .executable(
            name: "demo",
            targets: ["exec"]
        )
    ],
    
    dependencies: [
        .package(url: "https://github.com/apple/swift-package-manager", from: "0.3.0")
    ],

    targets: [
        .target(name: "Foo"),
        
        .target(
            name: "exec",
            dependencies: [
                "Foo", "Utility"
            ]
        )
    ]
)
```

开头的第一行注释代表这个package是用SwiftPM 4.2版本编写的。整个Package.swift实际上就是在创建一个Package实例，因此你可以动态的根据一些条件创建不同的的Package实例，比如在只在linux下依赖某些库等。

在这个例子中，我们对外暴露了两个product，一个名为Foo的库和一个名为demo的可执行文件，我们需要指定这个两个product由哪些target组成。设置完product后，需要指定这个package依赖的其他package，语法很简单，一个url地址和版本约束条件。url地址可以是git地址，也可以是本地地址，比如"../CHTTPParser"。最后就是设置这个package由哪些target组成，设置的方式基本与product类似。完整的语法可以参考这个[文档](https://github.com/apple/swift-package-manager/blob/master/Documentation/PackageDescriptionV4.md)

## 2、导入C库

目前支持SwiftPM的库大都集中在服务端，引用这部分库非常简单，但我们要在其他领域做一些事时，往往要需要自己导入c库。导入c库有两种方式，从源码编译，或者利用已经编译好的库。

从源码编译可以参考[swift-nio](https://github.com/apple/swift-nio)。这里主要介绍如何利用已经编译好的c库。下面已glfw为例。

首先我们要安装pkg-config，[pkg-config](https://zh.wikipedia.org/wiki/Pkg-config)是一个在源代码编译时查询已安装的库的使用接口的计算机工具软件。它输出已安装的库的相关信息。SwiftPM需要用它来导入c库。

```bash
brew install pkg-config
```

接着安装glfw3:

```bash
brew install glfw3
```

安装完成后我们使用下面命令检查是否安装glfw3的pkg-config：

```bash
pkg-config glfw3 --libs --cflags
```

如果已经安装了，会看到如下结果：

```
-I/usr/local/Cellar/glfw/HEAD-5afcd09/include -L/usr/local/Cellar/glfw/HEAD-5afcd09/lib -lglfw
```

安装pkg-config和glfw完成后，初始化一个swift package manager工程：

```bash
swift package init --type=executable
```

在Sources里新建一个CGFLW文件夹，并在里面添加两个文件module.modulemap和shim.h，最后的目录结构如下：

- root
    - Package.swift
    - Sources
        - CGLFW
            - module.modulemap
            - shim.h
        - swiftGLFWDemo
            - main.swift
    - Tests

对应的package.swift:

```swift
// swift-tools-version:4.2

import PackageDescription

let package = Package(
    name: "swiftGLFWDemo",
    targets: [
        .target(
            name: "swiftGLFWDemo",
            dependencies: ["CGLFW"]
        ),

        .systemLibrary(
            name: "CGLFW",
            path: "Sources/CGLFW",
            pkgConfig: "glfw3"
        ),
        
        .testTarget(
            name: "swiftGLFWDemoTests",
            dependencies: ["swiftGLFWDemo"]
        ),
    ]
)
```

这里多了一种新的target：systemLibrary。path需要传入这个target的文件夹的相对路径，路径的起点就是package.swift所在的目录，所以CGLFW的相对路径是"Sources/CGLFW"。这里最关键的参数就是pkgConfig，对于glfw我们需要传入"glfw3"，那这个值从哪来？最简单的方法是通过下面命令获得：

```bash
pkg-config --list-all
```

该命令列出了当前系统所有的pkg-config，我们可以找到下面这一行描述：

```
glfw3 GLFW - A multi-platform library for OpenGL, window and input
```

接下来我们需要设置modulemap和shim.h，打开shim.h，输入：

```c
#define GLFW_EXPOSE_NATIVE_COCOA

#include <GLFW/glfw3.h>
#include <GLFW/glfw3native.h>
```

这里引入需要暴露给swift的头文件。然后打开module.modulemap，输入：

```
module CGLFW [system] {
  header "shim.h"
  link "glfw"
  export *
}
```

modulemap的介绍已经远远超出了文本的范围，如果对它有兴趣的话可以参考[官方文档](https://clang.llvm.org/docs/Modules.html)。这里需要提的只有两点：

1、为什么需要shim.h这个文件？不能直接在module.modulemap里引用glfw的头文件吗？ 

如果直接引用glfw的头文件话，需要完整的路径，这样别人在使用你的库时很可能就要更改module.modulemap，设置成他的路径，非常的不方便，在目录下添加一个额外的头文件(这个头文件的名字可以随意取)就可以避免这种情况。使用shim.h还有另外一个好处，swift并不能完全导入c库的所有内容，比如复杂的宏和可变参数函数，有了这层，就可以对这些不能导入内容额外的包装一层暴露给swift，这也是很多库把这个头文件叫做shim.h的原因

2、link的参数从哪来？

使用`pkg-config glfw3 --libs --cflags`可以得到下面输出：

```
-I/usr/local/Cellar/glfw/HEAD-5afcd09/include -L/usr/local/Cellar/glfw/HEAD-5afcd09/lib -lglfw
```

最后的`-lglfw`去掉前面的-l就是我们要的参数。

上面的工作完成后，我们可以在main.swift里添加下面代码来测试是否正确配置了glfw：

```swift
import CGLFW

glfwInit();

glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);

let window = glfwCreateWindow(800, 600, "glfw window", nil, nil)

while(glfwWindowShouldClose(window) == 0) {
    glfwPollEvents();
}

glfwDestroyWindow(window);

glfwTerminate();
```

在命令行下输入：

```swift
swift run
```

如果配置正确的话将看到一个空的window。

可以通过brew安装的c库都可以使用上面的方法导入，但brew上没有的呢？比如在mac上使用vulkan，所以下面我们已vulkan为例子。

Apple并不支持vulkan，所以在mac使用vulkan是通过[MoltenVK](https://en.wikipedia.org/wiki/MoltenVK)这个库来实现的，我们先去[lunarg](https://vulkan.lunarg.com/sdk/home)下载vulkan在mac上的sdk，将下载后的tar解压，我们只需要MoltenVK文件夹下的内容。

现在我们要手动完成brew安装c库时做的事。首先我们要创建pkg-config配置文件，我们在MoltenVK文件夹下新建一个文件vulkan.pc，打开并输入：

```
prefix=/usr/local
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${prefix}/include

Name: MoltenVK
Description: My MoltenVK
Version: 1.0.0

Libs: -L${libdir} -lmoltenVK
Cflags: -I${includedir} -I${includedir}/..
```

vulkan.pc文件还是比较简单的，Libs需要提供库的路径，Cflags提供头文件路径

将MoltenVK整个文件夹拷入`/usr/local/Cellar`，为了让pkg-config找到我们刚才编写的vulkan.vc，需要把vulkan.pc拷入 `/usr/local/lib/pkgconfig`文件夹下，我们也可以用另一种方式，就是使用`ln`命令创建符号链接：

```bash
ln -s path/to/vulkan.vc /usr/local/lib/pkgconfig/vulkan.pc
```

-s表示创建符号链接，第一个路径是源文件路径，第二个路径是目标路径。

同理，我们需要把头文件夹vulkan放到`/usr/local/include`下，把库文件libMoltenVK.dylib放到`/usr/local/lib`下，这里都采用符号链接的方式：

```bash
ln -s path/to/vulkan /usr/local/include/vulkan

ln -s path/to/libMoltenVK.dylib /usr/local/lib/libMoltenVK.dylib
```

完成以上步骤后我们就可以用SwiftPM的pkg-config来导入vulkan了。SwiftPM的编写与之前引入glfw是一样的。  

在Sources目录下创建CVulkan文件夹，在CVulkan里创建module.modulemap和shim.h。module.modulemap的内容为：

```
module CVulkan [system] {
    header "shim.h"
    link "moltenVK"
    export *
}
```

shim.h的内容为：

```c
#include <vulkan/vulkan.h>
```

package.swift的内容为：

```swift
// swift-tools-version:4.2

import PackageDescription

let package = Package(
    name: "swiftVulkanDemo",
    
    targets: [
        .target(
            name: "swiftVulkanDemo",
            dependencies: ["CVulkan"]
        ),
        .systemLibrary(
            name: "CVulkan",
            pkgConfig: "vulkan"
        ),
        
        .testTarget(
            name: "swiftVulkanDemoTests",
            dependencies: ["swiftVulkanDemo"]
        ),
    ]
)
```

在main.swift里输入下面内容来测试安装是否正确：

```swift
import CVulkan

var appInfo = VkApplicationInfo(sType: VK_STRUCTURE_TYPE_APPLICATION_INFO,
                                pNext: nil, pApplicationName: "Hello Triangle",
                                applicationVersion: 0,
                                pEngineName: "No Engine",
                                engineVersion: 0,
                                apiVersion: UInt32(VK_VERSION_1_0))


var instance: VkInstance?

var createInfo = VkInstanceCreateInfo()
createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO
createInfo.pApplicationInfo = withUnsafePointer(to: &appInfo) { $0 }

let result = vkCreateInstance(&createInfo, nil, &instance)

if result == VK_SUCCESS {
    print("created instance result success")
}
vkDestroyInstance(instance, nil)
```

设置正确的话，`swift run`后将可以看到

```
created instance result success
```

## 3、总结

swift package manager目前还是很不完善，缺乏文档，不支持iOS，我想这主要原因是Apple的工作重心并不在这上面，Max Howell曾在twitter上提过，目前的工作重心是[llbulid](https://github.com/apple/swift-llbuild)，Apple新写的一套bulid工具，这个构建工具已经成为xcode 10的默认构建工具了。但无论如何，swift package manager作为官方包管理工具，前途可以说是非常光明的。
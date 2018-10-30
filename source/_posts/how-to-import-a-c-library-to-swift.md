---
title: 如何通过SwiftPM导入C库
date: 2018-10-30 07:35:09
tags: 
    - Swift
    - Swift Package Manager
---

Swift作为一门2014年才正式公布的新语言，各种类型的库非常的匮乏，github上绝大多数的Swift库也主要是用于iOS上的，更糟糕的是，Swift从2014年发布以来，每一年的大版本升级都不向后兼容，导致一个库6个月以上没维护就编译不过去。好在Swift可以直接调用大部分的C库，一定程度上缓解了这个问题。

本文主要介绍通过Swift Package Manager(SwiftPM)包管理器导入c库，这里我们简要的介绍下SwiftPM。

SwiftPM作为官方包管理器，随着Swift一起发布。SwiftPM最早由Max Howell开发，你很可能不认识他，但你一定用过它写的包管理工具[homebrew](https://brew.sh)，可惜的是他后面于Apple闹掰了，离开Apple没有再为SwiftPM贡献了。SwiftPM虽然由Apple负责开发，但它目前不支持iOS、watchOS、TVOS，主要的应用领域是编写命令行工具或服务端的开发。

SwiftPM的DSL就是Swift，只要你会Swift，你就会SwiftPM，不像cocoapods，如果你需要自定义一些操作，需要学习ruby。但SwiftPM完美的继承了Swift的缺点，每年都来个大改动，还不一定向后兼容，作为包管理器这就很难让人接受了。频繁变化的后果就是缺乏文档，或者文档过时，这里唯一推荐的文档就是[官方文档](https://github.com/apple/swift-package-manager/blob/master/Documentation/Usage.md)。SwiftPM本身支持xcode，可以通过`swift package generate-xcodeproj`生成xcode文件，但很难用，每添加一个文件或这更新依赖就得重新生成一次。值得庆幸的是，宇宙第一IDE厂商即将发布的新版clion将原生支持SwiftPM，具体可以看这篇[blog](https://blog.jetbrains.com/objc/2018/10/spm-support-clion/)。

SwiftPM最重要的两个概念就是target和product了。熟悉xcode的话，对这两个概念应该都不陌生，在一个iOS工程里，编译后生成的product就是一个app，而这个app可以由多个target组成，每个target都是一个独立的编译单元。在SwiftPM里，product里可以是一个库或者是一个可执行文件。


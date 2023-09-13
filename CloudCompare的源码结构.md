---
title: CloudCompare 源码目录结构
date: 2023-09-10 00:00:00
categories: 
 - [cpp]
 - [点云]
tags: 
---

CloudCompare 是一个开源的大规模点云处理软件，内置了点云渲染和各种点云算法，还有插件化拓展，非常适合作为点云相关软件的基础程序进行二次开发。

源码：[https://github.com/CloudCompare/CloudCompare](https://github.com/CloudCompare/CloudCompare)

这里记录一下项目的主要模块目录和他们做的事情。

## 编译和依赖
编译的文档在项目目录下的 BUILD.md 文件里。

CloudCompare 本体只依赖 Qt。在 CMake 中可以选择使用某些依赖来开启特定功能，例如使用 OpenMP 或 TBB 库实现并行计算，支持 raster/DXF/SHP 等格式的文件加载和保存。

CloudCompare 支持插件化拓展，同样可用在 CMake 中选择开启特定插件的编译，源码中已经内置类很多插件，每一个插件都可能会有自己的依赖，开启后需要提供这些依赖。


## 基本架构
```
   CloudCopmare
     ^      ^
     |      | 
 QCC_IO     QCC_GL
    ^        ^  ^
     \      /    \
      QCC_DB     CC_FBO
        ^
        |
     CC_CORE
```
上图描述了 CloudCompare 的几个主要模块和他们之间的依赖关系。
- CloudCopmare 是程序本身
- QCC_IO 负责点云文件的加载和保存
- QCC_GL 负责点云的渲染
- CC_FBO 负责实现渲染数据的管理
- QCC_DB 负责实现模型数据在内存中的存储管理
- CC_CORE 实现了点云相关的数据结构和算法

## 模块介绍
上面这些主要模块中，带有 "Q" 开头的说明这个模块依赖 QT。下面自底向上逐个介绍这些模块。

### CC_CORE
CC_CORE 被作为独立代码库脱离了 CloudCompare 仓库，它的项目地址是 https://github.com/CloudCompare/CCCoreLib ，在 CloudCompare 项目下的 `libs/qCC_db/extern/CCCoreLib` 目录作为 git 子模块引入。

该部分实现了很多点云处理相关数据结构和算法，例如数据结构 CCVector、DgmOctree  和Neighbourhood等，还有用来描述三维模型的各种 cloud、mesh 和 polyline 类型，算法方面提供了常用的点云算法。

### QCC_DB
该模块在 `/libs/qCC_db` 目录下。

这部分实现了 cloud、mesh 和 polyline 等模型数据在内存中存储和管理功能。

这些模型类全都继承自 ccHObject 类型，ccHobject 又继承了 ccDrawableObject、ccSeriallzableObject 和 ccObject，分别提供绘制、序列化和元信息查询的接口。

### QCC_IO
该模块在 `/libs/qCC_io` 目录下。

这部分规范了文件输入输出的接口，使用 `FileIOFilter` 作为基类，派生实现不同类型的模型文件的加载和保存。

源码中已经实现了几个常见的文件类型的 FileIOFilter，如果想要实现自己的模型文件的加载和保存，可以以插件的形式添加到到 CloudCompare 中，继承 FileIOFilter 类并实现对应的接口，最后再调用注册函数启用。

### QCC_GL 和 CC_FBO
这俩模块在 `/libs/qCC_glWindow` 和 `/libs/CCFbo` 目录下。

这部分实现了模型数据的渲染，底层用的是 OpenGL，实现比较复杂。如果需要拓展新类型的模型或者是自定义渲染效果就需要关注这个模块。

## 二次开发
拓展 CloudCompare 的方式有两种，一种是 UI 界面添加新的组件，另一种是插件化开发。
### UI 功能扩展
UI 拓展就是 QT 信号槽那一套，先打开 `qCC/ui_templates` 目录，找到想要修改的 ui 文件，然后再实现一下对应的槽函数就可以了。

### 插件扩展
插件拓展除了不能修改 CloudCompare 本身的 UI 以外，可以拓展任意想要的功能，CloudCompare 提供了三种基本类型的插件抽象，分别是 IO 插件、GL 插件和标准插件。

IO 插件是对 QCC_IO 模块的拓展，实现自定义文件的加载和保存。

GL 插件是对 QCC_GL 和 CC_FBO 的拓展，实现自定义的模型渲染或渲染效果。

标准插件则没有上俩个插件的限制，可以拓展任何想要的功能。
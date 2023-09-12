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

CloudCompare 本体只依赖 Qt。在 CMake 中可以选择使用某些依赖来开启特定功能，例如使用 OpenMP 或 TBB 库实现并行计算，支持raster/DXF/SHP等的摆放格式的文件加载和保存。

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
- QCC_DB 负责实现内存中的数据管理
- CC_CORE 实现了点云相关的数据结构和算法

## 模块介绍
上面这些主要模块中，带有 "Q" 开头的说明这个模块依赖 QT。下面自底向上逐个介绍这些模块。

### CC_CORE
CC_CORE 被作为独立代码库脱离了 CloudCompare 仓库，它的项目地址是 https://github.com/CloudCompare/CCCoreLib，在 CloudCompare 项目下的 libs/qCC_db/extern/CCCoreLib 目录作为 git 子模块引入。

该部分实现了很多点云处理相关数据结构和算法，例如数据结构 CCVector、DgmOctree  和Neighbourhood等，还有用来描述三维模型的各种 cloud、mesh 和 polyline 类型，算法方面提供了常用的点云算法。
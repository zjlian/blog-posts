---
title: toy-redis 开发日记：支持自定义内容输出的断言宏
date: 2022-07-31 04:42:36
categories:
 - [cpp]
 - [造轮子]
tags:
---

## 简介
本文将介绍 toy-redis 内使用的断言工具 `ASSERT_MSG`，这是一个支持 `std::cout` 风格输出打印的断言宏，极大的方便了断言失败时输出必要的调试信息到终端上。   

toy-redis 项目地址: https://github.com/TIDC/toy-redis   
ASSERT_MSG 源码: https://github.com/TIDC/toy-redis/blob/master/base/checker.hpp

## 目的和设想
c 语言标准库 <assert.h> 内提供的宏 `assert` 功能非常残疾，断言失败后只能简单把文件和行号输出到终端上。类似系统函数错误后，只能通过 `errno` 获取具体的错误码，还有错误码对应的描述，这些都无法在 `assert` 断言错误后体现出来。

那怎么办呢？

我想要的就是一个可以在断言失败后，输出条件、文件名、行号和自定义内容的东西。想象中的用法是这样的：
```c++
ASSERT_MSG(some_condition) 
    << "XXX 条件的需要满足 XXX，当前的值为 " << XXX;
```
断言失败后可以输出类似这样的信息:
```
条件 'some_condition' 断言失败
<文件名>:<行号> <函数名>
what: "XXX 条件的需要满足 XXX，当前的值为 [xxx 的具体值]
```

## 实现





(●'◡'●)
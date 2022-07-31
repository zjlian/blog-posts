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

## 目的
c 语言标准库 <assert.h> 内提供的宏 `assert` 功能非常残疾， 


(●'◡'●)
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
[!!!!!! ASSERT 'some_condition' ERROR !!!!!!]
location: /xx/main.cpp:21 main
what: XXX 条件的需要满足 XXX，当前的值为 ...
```

## 实现
目前第一版的实现先不考虑 Release 模式下的开销问题，这个有的是办法做到零开销。

回看一下断言宏 `ASSERT_MSG` 的用法。
```c++
ASSERT_MSG(some_condition) 
    << "XXX 条件的需要满足 XXX，当前的值为 " << XXX;
```
与普通的断言宏 `assert` 类型，如果条件为真那就啥都不干，条件为假就输出日志并提前结束程序。

`ASSERT_MSG` 需要特殊处理的是当条件为真时，后续的 operator<< 运算符和日志内容依旧存在，需要确保这些语句能够正确编译，但又不会输出内容到终端上。   
这一点与 `assert` 有所不同，`assert` 宏展开后是一个三元表达式，当条件为真时返回一个无意义的 void 值，条件为假时调用一个返回值为 void 的日志输出函数。   
`ASSERT_MSG` 条件为真时如果也是展开成一个无意义的 void 值，那后续的自定义输出语句就会出现语法错误导致无法编译了。

我的做法是无论条件真假，都返回一个支持 operator<< 运算符的对象，但只有条件为假时才会输出日志，并且在对象析构时结束程序。    
这个对象的类我取名为 `AbortOutputStream`，内部就是简单封装一下 `std::cerr`。先看声明：
```c++
class AbortOutputStream
    {
    public:
        /// 接受一个布尔值，表面是否输出日志和结束程序
        explicit AbortOutputStream(bool work)
        
        /// 析构函数内会结束程序
        ~AbortOutputStream()

        /// 流式风格输出的实现，因为需要
        template <typename T>
        AbortOutputStream &operator<<(const T &item)

        /// 额外的输出方式
        template <typename T, typename... Ts>
        AbortOutputStream &Print(const T &item, const Ts... items)

        /// 额外的输出方式
        template <typename T>
        AbortOutputStream &Print(const T &item)

    private:
        bool work_ = false;
        size_t output_count_ = 0;
    };
```

构造函数 `AbortOutputStream(bool work)` 会接受一个布尔值并初始化成员变量 `work_`，这个值非常关键，决定了是否会输出日志和结束程序。   
```c++
explicit AbortOutputStream(bool work)
    : work_(work) {}
```

析构函数 `~AbortOutputStream()` 内部会根据 `work_` 的值是否为真，决定要不要结束程序。
```c++
~AbortOutputStream()
{
    if (work_)
    {
        /// 先输出个换行和刷新缓冲区
        std::cerr << std::endl;
        /// 然后结束程序
        abort();
    }
}
```

实现日志输的出部分比较奇葩，日志信息主要分为两个部分，第一个部分是错误的条件和位置，另一个部分是用户自定义的日志内容。在有用户自定义输出内容是是这样的
```
[!!!!!! ASSERT 'some_condition' ERROR !!!!!!]
location: /xx/main.cpp:21 main
what: XXX 条件的需要满足 XXX，当前的值为 ...
```
没有时是这样的
```
[!!!!!! ASSERT 'some_condition' ERROR !!!!!!]
location: /xx/main.cpp:21 main
```
`AbortOutputStream` 实现中的两个 `Print` 函数正是用来输出错误的条件和位置部分，由于需要输出任意类型任意数量的内容，实现 `Print` 需要用到模板元编程的一些技巧。
```c++
/// 接受任意数量参数的 Print 是一个递归函数，他会递归调用自己，
/// 直到参数包 items 只剩下一个元素，这时候就会重载到另一个只接受一个参数的 Print 函数作为递
/// 归的结束条件。
template <typename T, typename... Ts>
AbortOutputStream &Print(const T &item, const Ts &...items)
{
    if (work_)
    {
        std::cerr << item;
        Print(items...);
    }
    return *this;
}

/// Print 函数的递归结束条件重载
template <typename T>
AbortOutputStream &Print(const T &item)
{
    if (work_)
    {
        std::cerr << item;
    }
    return *this;
}
```


(●'◡'●)
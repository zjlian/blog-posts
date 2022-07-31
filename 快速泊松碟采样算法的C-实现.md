---
title: 快速泊松碟采样算法的 C++ 实现
date: 2021-05-23 18:51:00
categories:
 - [cpp]
 - [算法]
tags:
---

因为最近打工写的算法中，有一个步骤需要生成二维平面上生成一组均匀分布的随机点（任意两点的距离不能小于 r），一开始自己想了好多方法，结果实现出来后运行的效果，感觉自己像个小丑🤡一样。  

之后便花了点时间百度了一下，找到这个叫快速泊松碟采样（Fast Poisson Disc Sampling）的算法挺符合需求的，但百度上好像没有看到 C++ 的实现，于是我<del>抄</del>实现了个C++的版本作为参考。该版本没有考虑性能，只是能用。

## 算法思路
算法的核心思想就是对二维平面进行分区，划分成一个个小区块，小区块的斜边长度为两点的间隔 r，因为斜边是一个正方形里最长线段，如果两个点同时出现在一个区块里，那他们的距离必然是小于 r 的，这样就是可以用区块标记表在 O(1) 的时间复杂度下查询出两个点的距离是否过近。算法名字中的 **Fast** 应该就体现在这里。   

算法的大致流程如下。想看算法出处的可以搜索文献 **Fast Poisson Disk Sampling in Arbitrary Dimensions** 看下。   
1. 根据间隔 r 划分区块

2. 生成第一个随机点，将其加入结果集合和候选点集合中

3. 从候选点集合中随机取一个点 P
   
4. 以 P 为圆心，在半径 [r, 2r) 的圆内生成一个新的随机点 NewP
   
5. 使用区块标记表查询 NewP 所在的区块是否已经存在随机点，查询计算周围8个区块的随机点与 NewP 的距离是否小于 r。如果 NewP不符合条件，回到步骤 4. 重复执行，直到成功或重试超过 30 次。
   
6. 如果新随机点 NewP 采样成功，将 NewP 加入到结果集合和候选点集合中；如果失败，将 P 从候选点集合中移除
   
7. 回到 3. ，直到候选点集合为空。

## 算法实现
代码实现需要先写一些准备工作的代码，比如描述“点”的向量类和一些随机数生成函数。

依赖的全部头文件如下
```c++
#include <algorithm>
#include <cstdint>
#include <cmath>
#include <ctime>
#include <iostream>
#include <ostream>
#include <type_traits>
#include <vector>
```

描述点的向量如下
```c++
template<typename T>
class Vec2_
{
public:
    template<typename OtherType>
    Vec2_(OtherType x, OtherType y) 
        : x(static_cast<T>(x)), 
          y(static_cast<T>(y)) {}
    Vec2_() : x(0), y(0) {}

    template<typename OtherType>
    Vec2_(const Vec2_<OtherType> &rhs) 
        : x(static_cast<T>(rhs.x)), 
          y(static_cast<T>(rhs.y)) 
    {}

    template<typename OtherType>
    Vec2_<T>& operator=(const Vec2_<OtherType> &rhs)
    {
        x = static_cast<T>(rhs.x);
        y = static_cast<T>(rhs.y);
        return *this;
    } 

    bool operator==(const Vec2_ &pt) const
    {
        return (this->x == pt.x && this->y == pt.y);
    }

    Vec2_ operator*(T n) const { return {x * n, y * n}; }

    Vec2_ operator+(T n) const { return {x + n, y + n}; }

    Vec2_ operator+(Vec2_ n) const { return {x + n.x, y + n.y}; }

    Vec2_ operator-(Vec2_ n) const { return {x - n.x, y - n.y}; }

    // 获取归一化的向量
    Vec2_<double> Normalized()
    {
        auto len = Magnitude();
        if (len != 0)
        {
            auto n = 1 / len;
            return {x * n, y * n};
        }
        else 
        {
            return {0, 0};
        }
    }

    // 获取向量的长度
    double Magnitude() { return sqrt(x * x + y * y); }

    // 类型转换
    template<typename OtherType>
    operator Vec2_<OtherType>()
    {
        return {static_cast<OtherType>(x), static_cast<OtherType>(y)};
    }

    T x; //坐标点的x值
    T y; //坐标点的y值
};
typedef Vec2_<int32_t> Vec2Int32;
typedef Vec2_<int64_t> Vec2Int64;
typedef Vec2_<double> Vec2;
```

伪随机数生成方面，由于C语言的 rand() 函数执行效率过于低下，严重影响算法的性能。C++ STL 的 mt19937 算法性能不错，但 API 组合起来比较复杂，当时一时半会儿还没太弄明白到底有多少种用法，那种用法更合适。

于是抄了个代码非常简短的 xor shift 伪随机数生成算法，性能也非常好。并且基于 xor shift 封装了几个需要的工具函数。
```c++
namespace Random
{
inline uint32_t XorShift32()
{
    static uint32_t seed = time(0);
    uint32_t x = seed;
    x ^= x << 13;
    x ^= x >> 17;
    x ^= x << 5;
    seed = x;
    return x;
}

// 获取 [min, max) 区间内的随机整数
inline int32_t Range(int32_t min, int32_t max)
{
    return (XorShift32() + min) % max;
}

// 获取 [0, max) 区间内的随机整数
inline int32_t Range(int32_t max)
{
    return Range(0, max);
}

// 获取半径为 1 的圆内的随机点
inline Vec2 InsideUnitSphere()
{
    double theta = Range(360 * 1000) / 1000.0;
    double r = Range(1000) / 1000.0;
    return {(sqrt(r) * cos(theta)), (sqrt(r) * sin(theta))};
}
} // namespace Random
```

快速泊松碟采样算法实现如下：
```c++
namespace Random
{
// 快速泊松碟随机采样算法，在一定范围内生成均匀分布的随机点
inline std::vector<Vec2> FastPoissonDiscSampling(Vec2 range, int32_t threshold)
{
    // boost::timer t;
    std::vector<Vec2> list;
    // 重试次数上限
    constexpr int32_t max_retry = 20;
    // n 的值等于 sqrt(维度)
    auto n = std::sqrt(2);
    // threshold 除上 sqrt(2)，可用保证每一个区块的斜边长为 threshold，
    // 从而确保一个区块内的任意两个点的距离都是小于 threshold 的。
    // 利用这个特性可用快速的确定两个采样点的间距小于 threshold。
    auto cell_size = threshold / n;
    // 划分区块
    int32_t cols = std::ceil(range.x / cell_size);
    int32_t rows = std::ceil(range.y / cell_size);
    // 区块矩阵
    std::vector<std::vector<int32_t>> grids;
    grids.resize(rows);
    for (auto &row : grids)
    {
        row.resize(cols, -1);
    }

    // 开始
    // 随机选一个初始点
    auto start = Vec2{Random::Range(range.x), Random::Range(range.y)};
    int32_t col = std::floor(start.x / cell_size);
    int32_t row = std::floor(start.y / cell_size);

    auto start_key = list.size();
    list.push_back(start);
    grids[row][col] = start_key;

    std::vector<int> active_list;
    active_list.push_back(start_key);

    double r = threshold;
    while (active_list.size() > 0)
    {
        // 在已经有的采样集合中取一个点, 在这个点周围生成新的采样点
        auto key = active_list[Random::Range(active_list.size())];
        // auto key = active_list[0];
        auto point = list[key];
        bool found = false;

        for (int32_t i = 0; i < max_retry; i++)
        {
            auto direct = Random::InsideUnitSphere();
            // 给原有的采样点 point 加上一个距离 [r, 2r) 的随机向量，成为新的采样点
            auto new_point = point + ((direct.Normalized() * r) + (direct * r));
            if ((new_point.x < 0 || new_point.x >= range.x) ||
                (new_point.y < 0 || new_point.y >= range.y))
            {
                continue;
            }

            col = std::floor(new_point.x / cell_size);
            row = std::floor(new_point.y / cell_size);
            if (grids[row][col] != -1) // 区块内已经有采样点类
            {
                continue;
            }

            // 检查新采样点周围区块是否存在距离小于 threshold 的点
            bool ok = true;
            int32_t min_r = std::floor((new_point.y - threshold) / cell_size);
            int32_t max_r = std::floor((new_point.y + threshold) / cell_size);
            int32_t min_c = std::floor((new_point.x - threshold) / cell_size);
            int32_t max_c = std::floor((new_point.x + threshold) / cell_size);
            [&]() {
                for (int32_t r = min_r; r <= max_r; r++)
                {
                    if (r < 0 || r >= rows)
                    {
                        continue;
                    }
                    for (int32_t c = min_c; c <= max_c; c++)
                    {
                        if (c < 0 || c >= cols)
                        {
                            continue;
                        }
                        int32_t point_key = grids[r][c];
                        if (point_key != -1)
                        {
                            auto round_point = list[point_key];
                            auto dist = (round_point - new_point).Magnitude();
                            if (dist < threshold)
                            {
                                ok = false;
                                return;
                            }
                            // 当 ok 为 false 后，后续的循环检测都没有意义的了，
                            // 使用 return 跳出两层循环。
                        }
                    }
                }
            }();

            // 新采样点成功采样
            if (ok)
            {
                auto new_point_key = list.size();
                list.push_back(new_point);
                grids[row][col] = new_point_key;
                active_list.push_back(new_point_key);
                found = true;
                break;
            }
        }

        if (!found)
        {
            auto iter = std::find(active_list.begin(), active_list.end(), key);
            if (iter != active_list.end())
            {
                active_list.erase(iter);
            }
        }
    }
    return list;
}
} // namespace Random
```
代码量不少，上方的 Vec2 其实不是必要的，可以替换成只有两个元素的 vector<int> 或者是简单的结构体替代，算法本体中的一些向量运算也可以直接写表达式，不需要调用函数，代码会少一点点。多写个 Vec2 class 单纯就是个人喜好。

下面写个 mian 函数测试下执行效果。
```c++
int main()
{
    auto result = Random::FastPoissonDiscSampling({100, 100}, 6);
    for (auto p : result)
    {
        std::cout << "(" << p.x << "," << p.y << ")" << std::endl;
    }
    std::cout << std::endl;
}
```
执行结果
```c++
(24,97)
(20.5812,88.8276)
(29.733,82.2709)
(13.8322,79.9774)
(12.2875,91.2487)
(16.9858,99.2597)
(8.35998,99.4975)
(29.1551,89.5978)
(33.6267,72.4279)
(22.9181,82.0951)
...
```

把打印的结果复制到一个数学可视化网站 desmos.com上看看效果。还是很不错的。
![结果](images/202105/fpd-sampling-result.png)

通过调整算法本体中的 max_retry 变量的值，可以调整是要更快的运行速度，还是要能填满所有的区块。max_retry  的值越小，运行速度就越快，但代价就是没法保证每一个区块都被填充随机点。前面提到的论文中说一般设置为 30，基本上就能保证每一个区块都存在随机点。

(●'◡'●)

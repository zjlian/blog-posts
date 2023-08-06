---
title: Cartographer中的简易全局重定位方法
date: 2023-08-05 18:16:18
categories: SLAM
tags:
---

这两天上网冲浪的时候看到有人说 Cartographer 里有一个 PerformGlobalLocalization 函数用于全局重定位，可以看 cartographer github 仓库的 Issue #95  Localization in Existing Map。

不过我在源码里搜了一下并没有发现这个函数，然后又把旧版本的源码也搜了一下，发现在0.3.0版本之后被删除了，同时新版本中也提供了 pure_localization 模式，函数被删除应该和这个有点关系。

在 0.3.0 或更早的版本中能找到叫 fast_global_localizer.h/.cpp 的文件，PerformGlobalLocalization 函数就在里面，函数声明如下:
```c++
// Perform global localization against the provided 'matchers'. The 'cutoff'
// specifies the minimum correlation that will be accepted.
// This function does not take ownership of the pointers passed in
// 'matchers'; they are passed as a vector of raw pointers to give maximum
// flexibility to callers.
//
// Returns true in the case of successful localization. The output parameters
// should not be trusted if the function returns false. The 'cutoff' and
// 'best_score' are in the range [0., 1.].
bool PerformGlobalLocalization(
    float cutoff, const cartographer::sensor::AdaptiveVoxelFilter& voxel_filter,
    const std::vector<
        cartographer::mapping_2d::scan_matching::FastCorrelativeScanMatcher*>&
        matchers,
    const cartographer::sensor::PointCloud& point_cloud,
    transform::Rigid2d* best_pose_estimate, float* best_score);
```



这个函数共有4个输入和3个输出：

**cutoff**          最低的匹配有效分数；   
**voxel_filter**    点云滤波器；   
**matchers**        FastCorrelativeScanMatcher 匹配器，需要匹配多少副 Submap 就传入多少个；   
**point_cloud**     等待匹配的点云；   
   
**best_pose_estimate**  匹配到的最佳位姿；   
**best_score**          匹配结果的分数；   
**return**              是否有匹配结果；   

函数的实现部分非常简单，就是一种 sacn-to-map 的匹配方法，先把点云过一遍滤波后，再逐个调用传入的 matcher 的 MatchFullSubmap 方法进行匹配，最后得到匹配得分最高的位姿和分数后返回。

PerformGlobalLocalization 函数存在一个很明显的缺陷啊，那就是可靠性不高，如果刚好传入的点云处在两个 Submap 的交界处，或者是多个 Submap 中纯在相似的结构，匹配的结果都有可能是错误的。我想在新版本中移除这个函数也是考虑到了这些问题，新版的 pure_localization 模式用 map-to-map 的方式来实现类似全局重定位的功能，虽然刚启动的时候要生成一定数量的 Submap 后才能触发当前轨迹与先验地图的轨迹的 map-to-map 匹配，但可靠性必然是比 PerformGlobalLocalization  要高得多的，不过也还是没能做到100%可靠。

在新版本中抄一下这个函数用来加速开机的重定位速度应该也是不错的，在地图比较大的时候，这个函数计算量会很大，再做一些优化可能会更好：

（一）并行化改造，这个函数就是很粗暴的遍历每一个 matcher 去调用函数做匹配，完全可以并行去执行这个循环，最后再汇总结果取出分数最高的位姿。

（二）在有先验位姿的情况下缩小搜索范围，可以用一些简单的策略来提高匹配的可靠性，例如只对距离最近的三幅 Submap 做匹配，并且直接调用 FastCorrelativeScanMatcher 的 Match 函数，通过提供先验位姿和有限的搜索范围来匹配点云。

（三）和别的全局重定位方法做互补和交叉验证，例如有AMCL的时候，可以用 PerformGlobalLocalization 先计算出一个位姿作为 AMCL 全局搜索的先验输入，AMCL 得到结果后再对比两个值是否接近，利用词袋模型实现的视觉全局重定位也可以做类似的操作，提高匹配速度并交叉验证结果。

(●'◡'●)

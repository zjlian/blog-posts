---
title: 给Cartographer添加全局重定位功能
date: 2023-08-05 18:34:30
categories: SLAM
tags:
---

新版本的 Cartographer 是没有初始全局重定位的功能的，不过旧版本中有提供一个简单的实现参考，可以看我先前的文章[《Cartographer中的简易全局重定位方法》](https://zjlian.github.io/2023-08/b75611994b01/)。但是在旧版源码中并没有实际使用这个函数。

下面将会介绍如何在新版 carto 中添加上全局重定位功能，并提供接口供 cartographer_ros 调用。同时也会介绍如何用 carto 自带的线程池加速重定位的计算。

可以先看看原版重定位函数的实现，我删除了一些检查和日志，只关注核心代码：
```C++
bool PerformGlobalLocalization(
    const float cutoff,
    const cartographer::sensor::AdaptiveVoxelFilter& voxel_filter,
    const std::vector<cartographer::mapping_2d::scan_matching::FastCorrelativeScanMatcher*>& matchers,
    const cartographer::sensor::PointCloud& point_cloud,
    transform::Rigid2d* const best_pose_estimate, float* const best_score) {
    // 输入有效性检查 ...
    *best_score = cutoff;
    transform::Rigid2d pose_estimate;
    // 对输入点云做滤波
    const sensor::PointCloud filtered_point_cloud = voxel_filter.Filter(point_cloud);
    bool success = false;
    // 遍历匹配器，对所有子图做全图匹配
    for (auto& matcher : matchers) {
        float score = -1;
        transform::Rigid2d pose_estimate;
        if (matcher->MatchFullSubmap(filtered_point_cloud, *best_score, &score, &pose_estimate)) {
            *best_score = score;
            *best_pose_estimate = pose_estimate;
            success = true;
        }
    }
  return success;
}
```
可以看到，原版的实现真的很简单，就是拿一帧点云，对每一幅 Submap 都做一次全图匹配。这里用的匹配器 `FastCorrelativeScanMatcher` 是 carto 用来做回环检测的匹配器，匹配的成功率还是很不错的，不过匹配后还位姿还会差一点点才能对上，需要再调用一次 `CeresScanMatcher2D` 匹配器进行精确匹配得到正确的位姿。

## 全局重定位功能实现过程
因为全局重定位需要读取所有的 Submap 数据和最新的点云数据，所以把代码加在 `pose_graph_2d` 模块里是最方便的，cartographer_ros 里也能调用到。

首先就是接口的定义，cartographer_ros 里能通过 `MapBuilderBridge` 访问到 carto 的 `MapBuilder` 实例，而 `MapBuilder` 又能访问到 `PoseGraph2D`，所以我们只需要在 `PoseGraphInterface` 接口类中添加全局重定位的虚函数，然后在他的派生类中添加对应的实现。

`PoseGraphInterface` 中添加的全局重定位的接口，我们的接口不需要传入点云滤波器，内部直接调体素滤波就行了。
```c++
// 文件 cartographer\mapping\pose_graph.h
class PoseGraphInterface 
{
public:
    virtual bool PerformGlobalLocalization(
      float cutoff, const cartographer::sensor::PointCloud& point_cloud,
      transform::Rigid2d* best_pose_estimate, float* best_score) {};
};
```
然后到 `PoseGraph2D` 里添加对应的函数。
```C++
// 文件 cartographer\mapping\internal\2d\pose_graph_2d.h
class PoseGraph2D : public PoseGraph 
{
    bool PerformGlobalLocalization(
        float cutoff, const cartographer::sensor::PointCloud& point_cloud,
        transform::Rigid2d* best_pose_estimate, float* best_score) override;
};
```
到这里先介绍一下这个重定位函数的流程，然后逐个部分给出实现代码。
1. 创建线程池；
2. 点云过一次体素滤波；
3. 为每一个 Submap 创建 FastCorrelativeScanMatcher 匹配器，在线程池中执行，并等待所有匹配器创建完成；
4. 遍历每一个匹配器，调用 MatchFullSubmap 方法，记录匹配结果和分数，在线程池中执行，并等待所有匹配完成；
5. 找出匹配得分最高的位姿，创建 CeresScanMatcher2D 匹配器再次匹配得到准确的位姿。

无论是创建匹配器还是全图匹配，计算量都很大，使用线程池来执行这些操作是必要的，这里直接用 carto 本身的线程池。
```C++
auto pool = std::make_unique<common::ThreadPool>(std::thread::hardware_concurrency());
assert(pool != nullptr);
```

点云滤波也是必须的，一两千个点云的计算量非常大，按 0.05 米的参数过一遍体素滤波后只剩几百个点，计算结果几乎也没差别。
```C++
const sensor::PointCloud filtered_point_cloud = sensor::VoxelFilter(0.05).Filter(point_cloud);
```

接着是为每一幅 Submap 创建匹配器，这一步需要在线程池里加速执行。
```C++
int32_t submap_size = static_cast<int>(data_.submap_data.size());
std::vector<std::shared_ptr<scan_matching::FastCorrelativeScanMatcher2D>> matchers;
matchers.resize(submap_size);
// 用于阻塞等待 N 幅 Submap 的匹配器创建完成
absl::BlockingCounter created_counter{submap_size};   

size_t index = 0;
for (const auto& submap_id_data : data_.submap_data) {
    // 只匹配从 pbstream 里加载的 Submap
    if (submap_id_data.id.trajectory_id != 0) {
        continue;
    }
    auto task = std::make_unique<common::Task>();
    task->SetWorkItem([this, &matchers, &created_counter, index, submap_id = submap_id_data.id, &submaps] {
        submaps.at(index) = static_cast<const Submap2D*>(data_.submap_data.at(submap_id).submap.get())->grid();
        matchers.at(index) =  std::make_unique<scan_matching::FastCorrelativeScanMatcher2D>(
            *(submaps.at(index)),
            options_.constraint_builder_options().fast_correlative_scan_matcher_options());
        created_counter.DecrementCount();
    });
    pool->Schedule(std::move(task));
    index++;
}
created_counter.Wait();
```

匹配器创建好之后就是调用全图匹配了，这一步同样需要在线程池里执行。
```C++
// 存储所有匹配分数和结果
std::vector<float> score_set;
std::vector<transform::Rigid2d> pose_set;
score_set.resize(submap_size);
pose_set.resize(submap_size);
absl::BlockingCounter matched_counter{submap_size};
std::atomic_bool has_matched{false};

for (size_t i = 0; i < matchers.size(); i++) {
    // 循环创建任务放入线程池
    auto task = std::make_unique<common::Task>();
    task->SetWorkItem([i, &filtered_point_cloud, &matchers, &score_set, &pose_set, cutoff, &matched_counter] {
        float score = -1; 
        transform::Rigid2d pose_estimate;
        if (matchers[i]->MatchFullSubmap(filtered_point_cloud, cutoff, &score, &pose_estimate)) {
            score_set.at(i) = score;
            pose_set.at(i) = pose_estimate;
            has_matched = true;
        }
        matched_counter.DecrementCount();
    });
    pool->Schedule(std::move(task));
}
matched_counter.Wait();
if (!has_matched)
{
    return has_matched;
}

// 遍历结果，找出分数最高的位姿
int max_position = max_element(score_set.begin(), score_set.end()) - score_set.begin();
*best_score = score_set[max_position];
*best_pose_estimate = pose_set[max_position];
```

FastCorrelativeScanMatcher2D 匹配器找出可能性最高的位姿后，就是最后一步使用 CeresScanMatcher2D 匹配器去匹配精确位姿了。
```C++
auto csm = std::make_unique<scan_matching::CeresScanMatcher2D>(
    options_.constraint_builder_options().ceres_scan_matcher_options());
ceres::Solver::Summary unused_summary;
csm->Match(best_pose_estimate->translation(), *best_pose_estimate,
           filtered_point_cloud, *(submaps.at(max_index)),
           best_pose_estimate, &unused_summary);

return has_matched;
```

使用方法也很简单，我用的方法是纯定位模式启动后，等加载完 pbstream 地图文件和缓存有效一帧点云数据，通过 `MapBuilderBridge` 暴露的接口调用全局重定位函数，得到有效的匹配位姿后缓存起来，然后析构掉当前的 Node 实例，重新构造一个并使用匹配到的位姿作为初始化位姿来重新开始纯定位模式。   
可以把这个接口封装成一个RPC请求，grpc 或者是 ros service 都可以，外部调用请求，cartographer_ros 计算返回结果，然后外部再重启纯定位模式。

其实这个方法还是有缺陷的，它只能在机器人停在原地不动的时候才能用，不然等匹配完，机器人已经跑别的地方去了。


(●'◡'●)

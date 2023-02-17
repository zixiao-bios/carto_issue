## Cartographer local SLAM 高频抖动的问题

在以下两种模式中，均出现 map->odom frame 的高频振动。

这两种模式都仅使用 odom 和 scan，不使用 imu。一方面，将里程计数据以 topic 发布，并 remap 给 carto 作为输入；另一方面，在 tf 中提供 odom->base_link 转换，在 carto 中设置 published_frame 为 odom，同时禁止 carto 发布 odom frame。两种模式的区别为里程计数据的来源。附带的两个 bag 文件分别对应两个模式。

### 模式一：使用原始里程计

+ odom topic/frame：均由 scout_base_node 发布。

### 模式二：使用融合里程计

+ odom topic：启动一个转换节点，订阅 robot_pose_ekf 发布的融合里程计 topic，并转为 odom 格式（为了满足 carto 对 odom 输入格式的要求）。具体而言，是将 ekf 发布的 [geometry_msgs/PoseWithCovarianceStamped](http://docs.ros.org/en/lunar/api/geometry_msgs/html/msg/PoseWithCovarianceStamped.html) （topic 名为 odom_combined）转为 [nav_msgs/Odometry Message](http://docs.ros.org/en/noetic/api/nav_msgs/html/msg/Odometry.html)（topic 名为 odom_combined_odom）。转换工作在录制 bag 时在线完成，bag 中已存在上述两种 topic。
+ odom frame：由 efk node 发布。
---
## 已做工作
### 参数调整
+ 考虑到旋转时 imu 会记录到抖动，因此关闭了 imu 输入，没有改善。
+ 设置 `POSE_GRAPH.optimize_every_n_nodes = 0` 来关闭 global SLAM，没有改善，确认是 logal SLAM 的问题。
+ 设置 `use_pose_extrapolator = false` 来关闭 pose_extrapolator，高频抖动消失，但飘移严重。我认为后者是关闭 pose_extrapolator 后正常的现象，但前者说明高频抖动是 pose_extrapolator 带来的。
+ 放弃使用 odom，让 cartographer 自己发布 odom frame，高频抖动消失，大部分情况下建图效果良好，少部分情况出现飘移。

### 检索仓库 issue
+ [加入里程计后，旋转没有被正确跟踪](https://github.com/cartographer-project/cartographer/issues/1122)，但后续作者说是 odom 信息不准所致，降低 `TRAJECTORY_BUILDER_2D.ceres_scan_matcher` 中相关权重即可，与本问题不符。
+ [使用里程计时 map->frame 不稳定](https://github.com/cartographer-project/cartographer_ros/issues/1239)，看视频感觉与本问题很像，但没人给出解决方案，主动询问后得知问题尚未解决。
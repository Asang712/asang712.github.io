---
title: "ROS2 Nav2 自主导航实战：从建图到单点导航的完整流程"
pubDatetime: 2026-11-15T00:00:00Z
description: "基于 Ubuntu 24.04 + ROS2 Jazzy + Gazebo 跑通 SLAM 建图、AMCL 定位与 Nav2 单点导航，附启动脚本、参数配置、清理脚本与高频报错清单。"
tags: ["ROS2", "Nav2", "Gazebo", "移动机器人"]
---

# ROS2 Nav2 自主导航实战长文（带全套脚本+配置）

> 平台：Ubuntu24.04 + ROS2 Jazzy + Gazebo Harmonic 面向：移动机器人入门，从建图、定位、单点导航、故障处理完整跑通 配套：可直接复制的启动脚本、yaml配置、清理脚本、常见报错清单

## 前言

自主导航是移动机器人的核心能力：机器人在环境中**自定位、规划路径、避障、自主运动到目标点**。 ROS2 的 Nav2 是目前主流开源导航栈，替代 ROS1 move_base，使用**行为树（BT）**作为任务调度引擎，所有组件解耦、插件化，可以替换规划器、控制器、SLAM算法，非常适合AMR、巡检小车开发。 整套导航链路：`SLAM建图 → 定位 → 全局路径规划 → 局部轨迹跟踪 + 动态避障`。

> ⚠️ 重要前置提醒：Ubuntu24.04 官方只支持 ROS2 Jazzy，**不要强行装 Humble**，会出现大量依赖、RMW、API废弃报错（例如老代码`create_client`传入`rmw_qos_profile_t`重载废弃问题）。

## 一、环境依赖安装

一键安装 Nav2、Turtlebot3、SLAM工具包、遥控工具

```
sudo apt update
sudo apt install -y ros-jazzy-nav2-bringup \
ros-jazzy-turtlebot3 \
ros-jazzy-turtlebot3-gazebo \
ros-jazzy-slam-toolbox \
ros-jazzy-cartographer \
ros-jazzy-teleop-twist-keyboard \
ros-jazzy-rviz2
```

配置环境变量（写到`~/.bashrc`，新开终端自动生效）

```
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
echo "export TURTLEBOT3_MODEL=waffle" >> ~/.bashrc
source ~/.bashrc
```

## 二、工作空间创建 & colcon 编译最佳实践

```
mkdir -p ~/robot_ws/src
cd ~/robot_ws
colcon build --symlink-install
source install/setup.bash
```

✅ `--symlink-install` 作用：不是把yaml、launch文件复制到install目录，而是创建软链接。**修改配置文件不需要重新编译**，Nav2调参高频使用。

> 踩坑：不带这个参数，每次改yaml都要重新colcon build，调试效率极低。

## 三、SLAM建图流程（slam_toolbox）

> 场景：陌生环境，遥控机器人扫图，保存地图给后续导航使用

### 1. 启动Gazebo仿真 + slam_toolbox建图节点

新建 `launch/slam_tb3.launch.py`

```
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import PathJoinSubstitution
from launch_ros.substitutions import FindPackageShare

def generate_launch_description():
    # Gazebo仿真环境
    gazebo = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            PathJoinSubstitution([FindPackageShare("turtlebot3_gazebo"), "launch", "turtlebot3_world.launch.py"])
        )
    )
    # slam_toolbox 异步建图
    slam = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            PathJoinSubstitution([FindPackageShare("slam_toolbox"), "launch", "online_async_launch.py"])
        ),
        launch_arguments={
            "slam_params_file": PathJoinSubstitution([
                FindPackageShare("slam_toolbox"), "config", "mapper_params_online_async.yaml"
            ])
        }.items()
    )
    return LaunchDescription([gazebo, slam])
```

启动命令：

```
ros2 launch your_pkg slam_tb3.launch.py
```

新开终端，键盘遥控机器人扫环境：

```
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

围着环境缓慢走一圈，RViz2里观察地图逐步构建。

### 2. 保存地图

```
ros2 run nav2_map_server map_saver_cli -f ~/robot_ws/maps/my_map
```

执行后生成两个核心文件：

- `my_map.pgm`：栅格地图图像
- `my_map.yaml`：地图参数（分辨率、原点、占用阈值）

> 注意：后续重新建图，要么把旧地图文件移动备份，要么改名；直接覆盖也可以，但建议备份，避免地图丢失。 Gazebo界面里删除物体只是临时生效，重启仿真就复原；想要永久修改场景，需要修改`.world`文件，不要直接覆盖官方world。

## 四、Nav2 定位导航启动（加载已有地图）

新建 `launch/nav_tb3.launch.py`

```
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import PathJoinSubstitution
from launch_ros.substitutions import FindPackageShare

def generate_launch_description():
    nav2_bringup = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            PathJoinSubstitution([FindPackageShare("nav2_bringup"), "launch", "tb3_simulation_launch.py"])
        ),
        launch_arguments={
            "map": "/home/asang/robot_ws/maps/my_map.yaml",
            "use_sim_time": "True"
        }.items()
    )
    return LaunchDescription([nav2_bringup])
```

启动导航：

```
ros2 launch your_pkg nav_tb3.launch.py
```

### RViz2 操作步骤

1. **2D Pose Estimate（初始位姿估计）**：告诉AMCL机器人大概在地图哪个位置，定位收敛；
2. **Nav2 Goal**：在地图上点击目标点，下发导航Action任务；
3. 机器人自动运行：全局规划 → 局部跟踪 → 避障走到目标。

## 五、Nav2 核心配置 yaml 重点参数讲解（nav2_params.yaml）

```
planner_server:
  ros__parameters:
    planner_plugin_types: ["nav2_smac_planner/SmacPlanner"]
    planner_plugin_names: ["SmacPlanner"]

controller_server:
  ros__parameters:
    controller_plugin_types: ["nav2_dwb_controller/DwbController"]
    controller_plugin_names: ["DwbController"]
    # 机器人最大速度限制
    max_vel_x: 0.26
    max_vel_theta: 1.0

costmap_server:
  ros__parameters:
    # 障碍物膨胀层，安全距离
    inflation_layer:
      inflation_radius: 0.3
```

- 全局规划器：SMAC / A* / Dijkstra，负责远距离寻路；
- 局部控制器：DWB、MPPI，输出速度指令，实时避障；
- 膨胀层：障碍物向外膨胀一圈，防止机器人贴墙碰撞；
- 代价地图分层：静态层（地图）、障碍物层（激光实时）、膨胀层。

## 六、一键清理 Gazebo & ROS残留进程脚本（高频痛点）

问题：`Ctrl+C` 关闭launch，**Gazebo、gzserver、ros节点经常后台残留**，再次启动会端口冲突、TF异常。 新建 `kill_gazebo.sh`

```
#!/bin/bash
echo "正在清理Gazebo、Nav2相关进程..."
pkill -f gazebo
pkill -f gzserver
pkill -f gzclient
pkill -f ros2
echo "清理完成"
```

赋予权限：

```
chmod +x kill_gazebo.sh
./kill_gazebo.sh
```

> 补充：不要只kill launch父进程，ros2 launch是父进程，Gazebo是它的子进程，父进程被杀子进程依旧留在后台，这就是kill之后Gazebo还在运行的原因。

## 七、核心概念梳理（面试&自查）

1. **TF坐标树**：`map → odom → base_link`，导航基石，TF断裂直接导航失败；
2. Topic / Service / Action
    - Topic：流式持续数据（scan、odom、map），发布订阅；
    - Service：短同步一问一答；
    - Action：长时任务（导航Goal），支持反馈进度、中途取消；
3. 行为树（BT）：Nav2的状态机调度中心。 普通状态机是固定单向流转；行为树支持重试、回退、条件分支：机器人卡住会执行恢复动作（原地旋转、后退），多次失败才判定任务失败。

## 八、高频报错&解决方案清单

1. **roadnet包编译 exited with code 2** 原因：Jazzy废弃了接收`rmw_qos_profile_t`的`create_client/create_service`重载； 解决：改用新的QoS构造方式，适配新版API。
2. **定位飘、机器人乱跑**

- 初始位姿没给准，AMCL无法收敛；
- 激光雷达噪声大、里程计漂移；
- TF树异常，`tf` 报错。

3. **无法规划路径，目标点变红**

- 目标点落在障碍物/膨胀区内；
- 地图原点、分辨率配置错误；
- 代价地图没有成功加载。

4. **colcon build 权限/externally-managed-environment** 优先用apt安装ROS包，不要pip直接在系统python环境安装包，避免系统环境托管报错。

## 九、进阶拓展方向

1. 多点自动巡逻：编写Action客户端，循环下发多个Nav2目标点；
2. 替换控制器：MPPI控制器，狭窄场景轨迹更顺滑；
3. 融合VSLAM，无激光视觉导航；
4. Route Server路网导航，适合工厂AMr车道式导航；
5. 多机器人协同导航。

## 十、小结

自主导航不是单一算法，是一套**感知-定位-规划-控制-任务调度**的系统工程。Nav2给了一套标准化、可插拔的框架，仿真环境下跑通只是第一步；实际机器人上还要解决里程计标定、雷达外参、定位漂移、动态障碍物等工程问题。 Ubuntu24.04 + Jazzy这套环境，要重点留意API废弃变更、TF连通性、进程残留、地图管理这几个坑。

---
title: "ROS2 Jazzy 实战：基于Nav2开发自动巡检机器人（Gazebo仿真）"
pubDatetime: 2026-10-16T00:00:00Z
description: "从 ROS2 Humble 迁移到 Jazzy，基于 Nav2 BasicNavigator Python API 实现多点自动巡检，记录全套踩坑与避坑指南。"
tags: ["ROS2", "Jazzy", "Nav2", "机器人仿真", "Gazebo"]
---
# ROS2 Jazzy 实战：基于Nav2开发自动巡检机器人（Gazebo仿真）
> 环境：Ubuntu24.04 + ROS2 Jazzy + Gazebo Sim + Nav2 + slam_toolbox
> 项目目标：机器人在仿真环境中按照预设路点自动巡逻，到达巡检点完成语音播报、相机拍照保存，实现一套完整仿真自动巡检系统。

## 0.前言
最近完成从 ROS2 Humble（Ubuntu22.04）迁移到 **ROS2 Jazzy（Ubuntu24.04）**，借着这个机会基于 Nav2 做了一套自动巡检机器人仿真项目。

很多初学 Nav2 的同学只会在 RViz 手动点击导航目标点，本文使用 `BasicNavigator` Python API，不依赖RViz手动打点，代码驱动机器人完成多点巡逻。
整套系统包含：SLAM建图、多点导航、自定义服务语音播报、相机图像保存、一键Launch启动。同时记录开发过程中大量真实踩坑，适合ROS2 Jazzy初学者参考。

**项目整体效果**：
1. Gazebo仿真环境加载FishBot机器人；
2. 加载提前建好的房间地图，AMCL完成定位；
3. 读取YAML配置的巡检路点，机器人依次前往每个目标；
4. 到达每个巡检点，调用语音播报服务播报提示，相机拍照保存本地；
5. 全部点位巡检完成任务结束。

## 1.环境说明与版本差异
本项目运行环境：
- OS：Ubuntu 24.04
- ROS：ROS2 Jazzy
- 仿真：Gazebo Sim（Ignition，不再是Gazebo Classic11）
- Nav2、slam_toolbox、ros2_control

> ⚠️ Jazzy 和 Humble关键差异，迁移务必注意
|项目|Humble(Ubuntu22.04)|Jazzy(Ubuntu24.04)|
|----|----|----|
|仿真器|Gazebo Classic 11|Gazebo Sim(Ignition)|
|ros2_control插件|`gazebo_ros2_control`|`gz_ros2_control`|
|Python版本|Python3.10|Python3.12|

> 建议：不要直接拷贝Humble的参数文件，尽量使用Jazzy原生Nav2参数模板修改。

## 2.系统整体架构
整个项目做模块解耦，拆分为多个功能包，通过ROS通信交互，不写硬编码，巡检点全部放到YAML配置文件。

|模块|功能|通信方式|消息/接口|
|---|---|---|---|
|patrol_node 巡检主节点|读取YAML路点，驱动机器人多点导航|Action|Nav2 BasicNavigator|
|speaker 语音服务节点|接收请求，执行语音播报|Service|自定义`SpeechText.srv`|
|相机图像订阅|到达点位拍摄保存图片|Topic|`sensor_msgs/msg/Image`|

通信分工：
1. **Action**：用于导航，导航属于长时间任务，可以获取中间反馈、最终任务结果，非常适合巡检；
2. **Service**：语音播报，短同步请求应答，请求一次播报一次；
3. **Topic**：相机图像流，持续单向数据流。

> 核心设计思想：业务参数YAML化，修改巡检路线不用改动代码；模块解耦，语音、拍照、导航可以单独调试。

## 3.前期准备：SLAM构建地图
在做巡检之前，需要先拥有环境地图。
使用`slam_toolbox`在Gazebo仿真环境遥控机器人运动完成建图。
```bash
# 启动仿真+slam_toolbox建图
ros2 launch fishbot_sim slam.launch.py
```
建图完成后保存地图，生成`room.pgm`、`room.yaml`，放到功能包`maps`目录。

> 注意：巡检目标坐标必须选择地图白色可通行区域，落在障碍物、膨胀层、未知区域会直接导航失败。

## 4.项目开发实现
### 4.1 创建功能包
一共两个功能包：
1. `autopatrol_interfaces`：存放自定义服务接口`SpeechText.srv`
2. `autopatrol_robot`：巡检主逻辑、launch、config、maps资源

创建自定义接口包：
```bash
ros2 pkg create --build-type ament_cmake autopatrol_interfaces
```
在`srv`文件夹新建`SpeechText.srv`
```
string text
---
bool success
```

创建巡检业务包(Python)
```bash
ros2 pkg create --build-type ament_python autopatrol_robot
```

> ⚠️重要：`CMakeLists.txt` / `package.xml`配置不要遗漏依赖；
> `autopatrol_robot`中`CMakeLists.txt`必须添加install指令，否则launch、config、maps编译后不会复制到install目录：
```cmake
install(DIRECTORY launch config maps DESTINATION share/${PROJECT_NAME})
ament_package()
```

### 4.2 语音服务节点 speaker.py
实现自定义服务，接收字符串，模拟语音播报。
>（示例伪代码，可自行补全）
```python
import rclpy
from rclpy.node import Node
from autopatrol_interfaces.srv import SpeechText

class SpeakerNode(Node):
    def __init__(self):
        super().__init__("speaker_node")
        self.srv = self.create_service(SpeechText,"speech_text",self.speech_callback)

    def speech_callback(self,request,response):
        self.get_logger().info(f"正在播报：{request.text}")
        response.success = True
        return response

def main():
    rclpy.init()
    node = SpeakerNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == "__main__":
    main()
```
记得在`setup.py`的`entry_points`注册可执行程序。

### 4.3 图像拍照逻辑
使用`CvBridge`把`sensor_msgs/msg/Image`转为OpenCV格式，到达巡检点调用`cv2.imwrite()`保存图片到本地。
> 关键点：只有导航到达目标点后才执行拍照，不要持续保存。

### 4.4 巡检主节点 patrol_node.py
核心逻辑，继承`BasicNavigator`，读取yaml巡检点列表，循环调用`followWaypoints()`执行多点导航。
重点逻辑：
1. 等待AMCL定位服务就绪；
2. 读取配置文件中一组巡检目标点；
3. 调用`followWaypoints()`依次执行路点；
4. 通过`TaskResult`判断导航状态：`SUCCEEDED`/`CANCELED`/`FAILED`；
5. 导航成功后调用语音服务，触发拍照保存。

> 关键提醒：
> - 必须先在RViz使用`2D Pose Estimate`初始化机器人位姿，`Localization: active`后再运行巡检节点；
> - 如果返回FAILED，优先排查目标点坐标是否落在障碍物区域。

### 4.5 Launch文件 autopatrol.launch.py
把Gazebo仿真、Nav2导航栈、speaker语音节点、patrol巡检节点整合到一个launch，实现一键启动整套系统。

## 5.编译与运行
```bash
# 清理旧编译产物（解决80%玄学bug）
rm -rf build install log
colcon build
source install/setup.bash

# 一键启动整套巡检系统
ros2 launch autopatrol_robot autopatrol.launch.py
```

操作流程：
1. 等待Gazebo、RViz全部启动；
2. RViz中使用`2D Pose Estimate`设置机器人初始位姿；
3. 等待Localization状态变为active；
4. 巡检节点自动开始执行多点巡逻任务；
5. 观察终端日志，机器人到达点位输出播报日志，本地生成巡检照片。

## 6.开发高频踩坑复盘
> 全部来自真实开发过程，Jazzy环境同样适用

1. **amcl/get_state service not available**
现象：巡检节点一直打印waiting等待定位服务。
原因：Nav2导航栈没有完整启动，AMCL未就绪。
解决：确认导航launch完整启动，等待定位激活再跑巡检节点。

2. **导航返回结果 FAILED，机器人原地不动**
原因：yaml里面配置的目标点坐标落在障碍物、膨胀层、地图未知区域，规划器无法生成路径。
解决：修改yaml，目标点选取地图白色可通行区域。

3. `Package xxx not found`
原因：编译成功，新终端没有执行`source install/setup.bash`。
解决：每次新开终端运行节点必须source环境。

4. ros2 launch 找不到launch/config/maps文件
原因：CMakeLists.txt缺少`install(DIRECTORY ...)`指令，资源文件不会拷贝到install目录。
解决：补充install指令，清理后重新编译。

5. 修改nav2_params.yaml参数，运行行为没有变化
原因：ROS运行读取install目录副本，修改src下yaml不会自动同步。
解决：`rm -rf build install log && colcon build`全量清理编译。

6. 订阅/发布消息无报错但是收不到数据
原因：发布订阅对象没有赋值给`self.xxx`，被Python垃圾回收。
解决：`self.pub = self.create_publisher(...)`，加上self。

## 7.项目后续扩展方向
本项目只是基础版本，还可以继续迭代：
1. **异常处理**：导航失败自动重试，多次失败跳过当前点位，不卡死整个巡检流程；
2. **动态任务**：新增服务接口，运行时动态增加、删除巡检路点，不用重启节点修改yaml；
3. **AI识别**：拍照完成后接入YOLO，识别画面中异常物体，触发报警；
4. **Web监控界面**：Flask/Vue做网页，实时展示机器人位置、巡检进度；
5. **迁移实物机器人**：将仿真代码适配真实硬件，结合`ros2_control`驱动实体机器人。

## 8.小结
通过本项目完整实践了ROS2中Topic、Service、Action三种通信方式。
Nav2的`BasicNavigator`极大降低导航二次开发门槛，但定位初始化、目标点合法性、编译缓存这些工程细节非常容易踩坑。
> Nav2开发口诀：**先定位，后导航；改参数，清编译；目标点，看地图。**

> 源码地址：（这里填入你的仓库地址）

---

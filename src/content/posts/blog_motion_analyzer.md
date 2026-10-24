---
title: "实战 OpenCV：用稀疏光流给视频做一个「镜头运动分析器」"
pubDatetime: 2026-10-24T00:00:00Z
description: "只用 200 行核心代码，基于 Shi-Tomasi 角点与 LK 稀疏光流判断镜头静止、运镜还是抖动，并输出逐帧 CSV 运动日志。"
tags: ["OpenCV", "计算机视觉", "Python"]
---

# 实战 OpenCV：用稀疏光流给视频做一个「镜头运动分析器」

> 只用 200 行核心代码，让程序自动判断视频是静止拍摄、平稳运镜，还是手持抖动——并把所有数据记进 CSV。

## 前言：为什么做这个小项目

学视频处理时，"理解帧间运动"是绕不开的基本功：视频防抖、目标跟踪、SLAM 前端，底层都在回答同一个问题——**这一帧相对上一帧，画面往哪儿动了、动了多少**。

与其只看公式，不如动手做一个能跑出结果的小工具。这篇文章记录我的实战项目：**视频镜头运动分析器**。给它一段视频（或插上 USB 摄像头），它能实时告诉你镜头现在是**静止、横向移动、纵向移动还是抖动**，把运动轨迹画在画面上，并输出一份逐帧的 CSV 运动日志。

完整源码已开源：[github.com/Asang712/OpenCV\_basic\_project](https://github.com/Asang712/OpenCV_basic_project)

## 先看效果

用一段 15 秒的实拍视频（1280×720，24fps，共 361 帧）测试，程序不到 1 秒就分析完毕，统计结果：

| 运动状态 | 帧数  | 占比    |
| ---- | --- | ----- |
| 静止   | 142 | 39.3% |
| 横向移动 | 198 | 54.8% |
| 纵向移动 | 16  | 4.4%  |
| 抖动   | 5   | 1.4%  |

画面上会实时叠加：绿色的特征点和轨迹线、画面中心的红色平均运动箭头，以及左上角的状态/位移/波动 HUD。同时生成 `motion_log.csv`，包含每一帧的时间戳、运动类型、平均位移 dx/dy、位移大小和运动波动，Excel 打开就能画曲线。

## 整体思路

整个流程可以概括成一条流水线：

```text
视频帧 → 灰度化 → 特征点跟踪(LK光流) → 帧级运动向量
       → 滑动窗口平滑 → 阈值分类 → 画面可视化 + CSV 落盘
```

核心技术选型是 **Shi-Tomasi 角点检测 + Lucas-Kanade 稀疏光流**。没有选稠密光流，原因很简单：稀疏方案只跟踪几百个"好跟的点"，计算量小、实时性高，对"镜头整体往哪动"这个问题完全够用。

## 核心原理拆解

### 1. 找什么样的点来跟踪？

第一帧用 `cv2.goodFeaturesToTrack` 做 Shi-Tomasi 角点检测。它会挑出纹理丰富、在两个方向上都有明显梯度的像素块——通俗说就是"有辨识度的角点"，跟踪起来最不容易丢：

```python
# Shi-Tomasi 角点检测参数
FEATURE_PARAMS = dict(maxCorners=200, qualityLevel=0.01,
                      minDistance=10, blockSize=7)

pts = cv2.goodFeaturesToTrack(gray, mask=None, **FEATURE_PARAMS)
```

`minDistance=10` 保证点不会太密集地挤在一起，避免信息冗余。

### 2. LK 光流：算出每个点往哪儿动了

相邻两帧之间，用金字塔 LK 光流（`cv2.calcOpticalFlowPyrLK`）计算每个特征点的位移。金字塔的意义是"先粗后精"：在缩小的图上先估个大方向，再逐级细化，这样即使运动幅度稍大也能跟住：

```python
LK_PARAMS = dict(winSize=(21, 21), maxLevel=3,
                 criteria=(cv2.TERM_CRITERIA_EPS | cv2.TERM_CRITERIA_COUNT, 30, 0.01))

p1, st, _ = cv2.calcOpticalFlowPyrLK(self.prev_gray, gray,
                                     self.points, None, **LK_PARAMS)
new_pts = p1[st == 1]        # 只保留跟踪成功的点
old_pts = self.points[st == 1]
vectors = new_pts - old_pts  # 每个点的位移向量
```

### 3. 一个便宜但有效的离群点剔除

光流跟踪难免有个别点"跟飞了"（比如遮挡、重复纹理）。如果直接对所有点求平均，一个错配就可能把结果带偏。这里用了一个极简策略：**位移幅度超过中值 3 倍的点直接扔掉**：

```python
mags = np.linalg.norm(vectors, axis=1)
keep = mags <= max(3.0 * float(np.median(mags)), 2.0)
vectors = vectors[keep]
vector = vectors.mean(axis=0)   # 帧级运动向量
```

不追求 RANSAC 那样的严谨，但对"镜头整体运动"这个场景，性价比很高。

### 4. 分类的关键：位移和波动是两回事

这是整个项目里我认为最值得展开的设计。直觉上"抖动"就是动得快？其实不然——**匀速平移的位移也很大，但它不是抖动**。

抖动的本质是**方向来回变**：左一下、右一下，平均下来位移可能接近零。所以程序维护了一个最近 5 帧的滑动窗口，计算两个量：

- **平滑位移**：窗口内向量的均值 `smooth`，它的大小 `magnitude` 代表"整体往哪动、动多快"
- **运动波动** `jitter`：窗口内各帧向量相对均值的平均偏差，代表"运动有多不稳定"

```python
arr = np.asarray(self.window)          # 最近 N 帧运动向量
smooth = arr.mean(axis=0)              # 平滑运动向量
magnitude = float(np.linalg.norm(smooth))
jitter = float(np.linalg.norm(arr - smooth, axis=1).mean())
```

分类规则就清晰了（阈值均可通过命令行调节）：

```python
def _classify(self, smooth, magnitude, jitter):
    if jitter >= self.shake_thresh:      # 波动大 → 抖动 (默认 2.5 px/帧)
        return STATE_SHAKE
    if magnitude < self.static_thresh:   # 位移小 → 静止 (默认 0.5 px/帧)
        return STATE_STATIC
    # 比较主方向
    return STATE_H_MOVE if abs(smooth[0]) >= abs(smooth[1]) else STATE_V_MOVE
```

这套"先看波动、再看位移"的顺序，保证匀速运镜不会被误判成抖动，而方向来回甩的手持画面即使平均位移很小也能被抓住。

## 锦上添花的两个扩展

### 简易防抖预览

既然每帧都算出了画面偏移，顺手就能做个防抖：累计偏移量做指数平滑，然后用 `cv2.warpAffine` 反向平移补偿，单独开一个窗口对比原画面：

```python
self.cum_offset += smooth
self.stab_offset = 0.92 * self.stab_offset + 0.08 * self.cum_offset
m = np.float32([[1, 0, -self.stab_offset[0]],
                [0, 1, -self.stab_offset[1]]])
return cv2.warpAffine(frame, m, (w, h))
```

只建模了平移（没有旋转和缩放），但作为"理解防抖原理"的演示足够了。运行时可以按 `s` 键随时开关对比。

### 自动标记剧烈抖动片段

波动超过剧烈阈值（默认 8 px/帧）时，程序自动记录片段的起止时间和峰值波动，输出 `shake_segments.csv`，并在画面上红字告警。做素材质检时可以直接拿着这个文件去剪片子。

## 几个值得记录的工程细节

1. **跟踪点会"越跟越少"**：画面移动后部分点移出视野。程序设了下限（10 个），不足就重新跑一遍角点检测，保证跟踪点池始终健康。
2. **OpenCV 中文显示的老大难**：`cv2.putText` 不支持中文。解决方案是用 Pillow 把文字画到图像上，并内置了一份跨平台字体候选列表（Linux 的 Noto/文泉驿、Windows 的雅黑、macOS 的苹方）；找不到字体就自动退回英文标签，功能不受影响。
3. **CSV 用** **`utf-8-sig`** **编码**：带 BOM，Excel 双击打开中文不乱码，一个字符省掉一次答疑。
4. **无窗口模式**：加 `--no-display` 可以在没有图形界面的服务器上纯跑数据，方便批处理。

## 使用方法

```bash
pip install -r requirements.txt   # opencv-python, numpy, pillow(可选)

# 本地视频
python3 video_motion_analyzer.py --source test.mp4

# USB 摄像头 + 防抖预览
python3 video_motion_analyzer.py --source 0 --stabilize

# 服务器无窗口运行, 只要 CSV
python3 video_motion_analyzer.py --source test.mp4 --no-display

# 阈值太灵敏? 手动调
python3 video_motion_analyzer.py --source test.mp4 --shake-thresh 3.0
```

## 总结

这个项目代码量不大，但把一条完整的"感知流水线"跑通了：特征提取 → 运动估计 → 数据清洗（离群剔除）→ 时序平滑 → 状态决策 → 可视化与日志沉淀。其中最有意思的收获是：**判断"抖动"靠的不是位移大小，而是运动的波动**——很多时候把问题拆成两个独立的物理量，比堆一个复杂模型更有效。

后续可以玩的方向：把平移模型升级成仿射/单应模型做真正的稳像、用运动向量做视频摘要、或者接目标检测做"镜头动还是物体动"的判别。源码都在仓库里，欢迎自取：

**[github.com/Asang712/OpenCV\_basic\_project](https://github.com/Asang712/OpenCV_basic_project)**

***

*如果这篇文章对你有帮助，欢迎 Star 支持一下。*

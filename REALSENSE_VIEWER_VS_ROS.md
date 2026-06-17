# RealSense Viewer 与 ROS 2 驱动不能同时使用

## 现象

通过 `ros2 launch realsense2_camera rs_launch.py` 启动 ROS 节点后：

- `realsense-viewer` **可以正常打开软件**
- 但 **检测不到已连接的 RealSense 相机**（设备列表为空）

这不是 launch 参数配置错误，也不是相机未连接。

---

## 根本原因：USB 设备独占访问

Intel librealsense SDK 对 RealSense 相机采用 **单进程独占** 机制：同一时刻只允许一个进程打开 USB 设备。

当 `realsense2_camera_node` 运行时，它会独占相机。此时再启动 `realsense-viewer`，viewer 无法枚举到设备，表现为「软件能开、相机检测不到」。

### 如何确认

```bash
# 1. USB 物理连接正常
lsusb | grep -i intel
# 应看到：Intel(R) RealSense(TM) Depth Camera 435i

# 2. ROS 节点占用设备
ps aux | grep realsense2_camera_node

# 3. 枚举失败（ROS 运行时）
rs-enumerate-devices -s
# 输出：No device detected. Is it plugged in?

# 4. 查看 USB 设备被谁占用（将 002/003 换成实际 bus/device）
fuser -v /dev/bus/usb/002/003
# 应看到 realsense2_camera_node 的 PID
```

---

## 当前 `rs_launch.py` 默认配置（D435i）

本仓库 launch 为 OpenVINS / Ego-Planner 优化，**驱动本身工作正常**：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `enable_color` | `false` | RGB 彩色流关闭 |
| `enable_depth` | `true` | 深度 640×360@30 |
| `enable_infra1` | `true` | 左红外 640×360@90 |
| `enable_infra2` | `true` | 右红外 640×360@90 |
| `align_depth.enable` | `false` | 深度对齐 RGB 关闭 |
| `pointcloud.enable` | `true` | 点云发布 |

ROS 运行时可用话题示例：

| 用途 | 话题 |
|------|------|
| 左红外 | `/camera/camera/infra1/image_rect_raw` |
| 右红外 | `/camera/camera/infra2/image_rect_raw` |
| 深度 | `/camera/camera/depth/image_rect_raw` |
| 融合 IMU | `/camera/camera/imu` |
| 点云 | `/camera/camera/depth/color/points` |

> **注意**：默认无 `/camera/camera/color/image_raw`，因为 `enable_color=false`。这与 viewer 检测不到相机无关；viewer 的问题是设备被 ROS 独占。

---

## 解决方案

### 方案 A：用 realsense-viewer 调试硬件

必须先停止 ROS 节点，再打开 viewer：

```bash
# 在 launch 终端 Ctrl+C，或：
pkill -f realsense2_camera_node

# 确认设备可枚举
rs-enumerate-devices -s

# 打开 viewer
realsense-viewer
```

### 方案 B：ROS 运行时查看图像（推荐）

不停止 ROS，通过 ROS 工具订阅话题：

```bash
source /opt/ros/humble/setup.bash
source install/setup.bash

# 左红外
ros2 run rqt_image_view rqt_image_view /camera/camera/infra1/image_rect_raw

# 深度
ros2 run rqt_image_view rqt_image_view /camera/camera/depth/image_rect_raw

# 或用 rviz2 添加 Image 显示
rviz2
```

检查话题是否在发布：

```bash
ros2 topic list | grep camera
ros2 topic hz /camera/camera/depth/image_rect_raw
```

### 方案 C：需要 RGB 彩色流时

launch 时开启 color（会增加 USB 带宽占用）：

```bash
ros2 launch realsense2_camera rs_launch.py enable_color:=true
```

如需对齐深度到 RGB：

```bash
ros2 launch realsense2_camera rs_launch.py enable_color:=true align_depth.enable:=true
```

---

## 总结

| 现象 | 原因 |
|------|------|
| viewer 能打开但检测不到相机 | ROS 节点已独占 USB 设备 |
| ROS 里没有 color 话题 | `enable_color=false`（有意配置，非故障） |
| infra / depth / imu 有数据 | 驱动工作正常 |

**realsense-viewer 与 `ros2 launch realsense2_camera rs_launch.py` 不能同时对同一台 D435i 出图。** 二选一：停 ROS 用 viewer，或保持 ROS 用 `rqt_image_view` / `rviz2`。

相关文档：[COORDINATES_OPENVINS.md](./COORDINATES_OPENVINS.md)（坐标系与话题说明）

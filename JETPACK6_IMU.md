# JetPack 6 上 RealSense D435i IMU 说明

本文档说明在 NVIDIA Jetson JetPack 6 平台上，D435i 的 IMU 话题无法出现的原因及修复方法。

## 背景

| 项目 | 说明 |
|------|------|
| 平台 | Jetson，JetPack 6（内核 R36，如 `5.15.136-tegra`） |
| 相机 | Intel RealSense D435i |
| ROS | ROS 2 Humble + `realsense2_camera` |

D435i 的深度/红外/彩色流可通过 apt 安装的 `ros-humble-librealsense2` 正常工作，但 **IMU（陀螺仪 + 加速度计）在 JP6 上默认不可用**。

## 现象

启动 `ros2 launch realsense2_camera rs_launch.py` 后：

- 有 `/camera/camera/infra1/image_rect_raw` 等图像话题
- **没有** `/camera/camera/imu`、`/camera/camera/gyro/sample`、`/camera/camera/accel/sample`
- 日志中出现：

  ```
  No HID info provided, IMU is disabled
  ```

命令行验证：

```bash
rs-enumerate-devices | grep -A10 "Motion Module"   # 无 Motion Module 段
rs-motion                                          # Device supporting IMU not found
```

## 原因

1. **JetPack 6 变更**：JP6 移除了 RealSense IMU 所依赖的部分 HID 访问机制（`hiddraw` 相关）。
2. **后端不匹配**：`apt install ros-humble-librealsense2` 使用 **V4L2 内核后端**，在 JP6 上只能可靠访问 UVC 视频流，**无法打开 Motion Module（HID）**。
3. **与 launch 配置无关**：`enable_gyro`、`enable_accel`、`unite_imu_method` 等 launch 参数已正确时仍无 IMU，说明问题在 **librealsense 驱动层**，不在 `rs_launch.py`。

## 解决方案：RSUSB（libuvc）后端编译 librealsense

需从源码编译 librealsense，并开启 **`FORCE_RSUSB_BACKEND=ON`**（即 libuvc 用户态 USB 后端），绕过 JP6 上有问题的 V4L2/HID 路径。

### 1. 编译 librealsense（RSUSB）

源码路径（本机已有）：

```
d1robot/librealsense/src/librealsense/
```

```bash
# 停掉占用相机的节点
pkill -f realsense2_camera_node

# 若之前用 sudo 编译过，先修复目录权限
sudo chown -R $USER:$USER d1robot/librealsense/src/librealsense/build

cd d1robot/librealsense/src/librealsense/build

cmake .. \
  -DFORCE_RSUSB_BACKEND=ON \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_EXAMPLES=false \
  -DBUILD_GRAPHICAL_EXAMPLES=false \
  -DBUILD_TOOLS=true

make -j$(nproc)
sudo make install
sudo ldconfig
```

cmake 输出中应看到：`using RS2_USE_LIBUVC_BACKEND`。

### 2. 验证 IMU

```bash
/usr/local/bin/rs-enumerate-devices | grep -A10 "Motion Module"
```

期望输出包含 Accel / Gyro 流，例如：

```
Stream Profiles supported by Motion Module
    Accel       MOTION_XYZ32F  @ 250/63 Hz
    Gyro        MOTION_XYZ32F  @ 400/200 Hz
```

`rs-motion` 若报 OpenGL 错误可忽略（仅为图形工具）；以 `rs-enumerate-devices` 为准。

### 3. 重新编译 realsense2_camera（链接 /usr/local 的 SDK）

工作空间：

```
d1robot/realsense/
```

```bash
cd d1robot/realsense
source /opt/ros/humble/setup.bash

colcon build --packages-select realsense2_camera \
  --cmake-args \
  -DCMAKE_PREFIX_PATH=/usr/local \
  -Drealsense2_DIR=/usr/local/lib/cmake/realsense2

source install/setup.bash
ros2 launch realsense2_camera rs_launch.py
```

启动日志中应出现：

```
Starting Sensor: Motion Module
Open profile: stream_type: Accel ...
Open profile: stream_type: Gyro ...
```

话题：

```bash
ros2 topic list | grep -E "imu|gyro|accel"
```

### 4. 若 RSUSB 后仍无 IMU：降级固件（可选）

部分 JP6 设备在 RSUSB 下仍需将 D435i 固件降至 **5.13.0.50** 或 **5.13.0.55**：

```bash
rs-fw-update -l
sudo rs-fw-update -f <固件文件.bin>
# 拔插相机后重试 rs-enumerate-devices
```

## 常见问题

### 版本不一致警告

```
Built with LibRealSense v2.57.7
Running with LibRealSense v2.58.1
```

说明 ROS 节点编译时链的是 apt 版 SDK，运行时加载了 `/usr/local` 新版。按上文 **第 3 步** 重新 `colcon build` 即可消除。

### IMU Calibration is not available

RSUSB 后端有时读不到设备内 IMU 标定表，会使用默认内参。一般 **不影响轴向**，可能略影响零偏/尺度精度。若 VIO 精度不够，需在目标平台上做 IMU 标定。

### 修改 launch 应改哪里？

`colcon build` 会把 `src/` 安装到 `install/`，**不要直接改 `install/` 下的文件**（重编会被覆盖）。

正确路径：

```
d1robot/realsense/src/realsense-ros/realsense2_camera/launch/rs_launch.py
```

修改后执行：

```bash
colcon build --packages-select realsense2_camera
source install/setup.bash
```

### USB 带宽

同时开 90Hz 双目 + 深度 + 彩色 + IMU 时，可能出现 `control_transfer ... Resource temporarily unavailable`。建议相机 **直连 Jetson USB3 口**，避免经 Hub。

## IMU 数据与坐标系（简要）

- 话题数据在 **光学坐标系** `camera_imu_optical_frame`：X 右、Y 下、Z 前（光轴）。
- 静止时 `linear_acceleration` 某一轴约 ±9.8 m/s² 为 **重力**，属正常。
- 绕竖直轴旋转时主要变化在 `angular_velocity.y`（光学系下 Y 与“上下”相关），不等于融合错误。
- 融合话题 `/camera/camera/imu` 的角速度与 `/camera/camera/gyro/sample` 应一致；融合仅将加速度按时间对齐到陀螺仪时刻。

## 参考

- [librealsense libuvc 安装说明](https://github.com/IntelRealSense/librealsense/blob/master/doc/libuvc_installation.md)
- [IMU on 435i not working on Jetson with Jetpack 6 (GitHub #13784)](https://github.com/IntelRealSense/librealsense/issues/13784)
See also: [COORDINATES_OPENVINS.md](../COORDINATES_OPENVINS.md) for frame conventions and OpenVINS integration.

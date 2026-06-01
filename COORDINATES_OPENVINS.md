# RealSense D435i 坐标系说明（OpenVINS 对接）

本文档说明 `realsense2_camera` 发布的 **坐标系 / TF / 话题数据含义**，以及对接 **OpenVINS** 时的注意事项。  
默认 launch 配置见 `src/realsense-ros/realsense2_camera/launch/rs_launch.py`（`camera_name:=camera`，`camera_namespace:=camera`）。

相关文档：[JETPACK6_IMU.md](./JETPACK6_IMU.md)（Jetson IMU 驱动问题）

---

## 1. 为什么坐标系对 OpenVINS 很重要

OpenVINS 依赖：

1. **IMU 读数**（角速度、含重力的加速度）在一致的 IMU 坐标系下；
2. **相机图像**在 OpenCV 针孔模型（光学系，Z 朝前）下；
3. **`kalibr_imucam_chain.yaml` 中的 `T_imu_cam` / `T_cam_imu`**：相机与 IMU 之间的外参。

若把 RealSense 的光学系误当成 ROS 机体系（Z 向上），会出现：

- 静止时加速度“轴不对”；
- 绕竖直轴转却看到 `angular_velocity.y` 变化；
- OpenVINS 初始化失败或轨迹漂移。

**结论：RealSense 话题里的数值是光学坐标系，不是 REP-103 机体系。**

---

## 2. 三套坐标系

站在相机**后方**朝镜头方向看（RealSense 文档约定）：

### 2.1 光学坐标系（Optical）—— **话题数据的实际坐标系**

| 轴 | 方向 |
|----|------|
| **X** | 右 |
| **Y** | 下 |
| **Z** | 前（光轴 / 镜头朝向） |

适用于：`image_rect_raw` 对应的 `camera_*_optical_frame`，以及 **IMU 话题**（`frame_id` 为 `camera_imu_optical_frame`）。

### 2.2 传感器 ROS 系（`camera_*_frame`）

各传感器（depth / infra1 / infra2 / gyro）相对 `camera_link` 的 ROS 位姿，由静态 TF 发布。

### 2.3 相机机体系 `camera_link`（ROS REP-103）

| 轴 | 方向 |
|----|------|
| **X** | 前 |
| **Y** | 左 |
| **Z** | 上 |

原点：左目（infra1）光心。  
OpenVINS 输出的位姿通常在该系或世界系下，**不等于** IMU 话题直接使用的光学系。

### 关系示意

```
camera_link (ROS: X前 Y左 Z上)
    └── camera_infra1_frame
            └── camera_infra1_optical_frame  (Optical: X右 Y下 Z前)  ← 图像/内参
    └── camera_imu_frame
            └── camera_imu_optical_frame     (Optical)               ← /imu 数据
```

光学系 ↔ ROS 系：绕 X 轴旋转 -90°，再绕 Z 轴旋转 -90°（与 OpenCV / RealSense 一致）。  
驱动通过静态 TF `*_frame` → `*_optical_frame` 发布该变换。

---

## 3. 默认 frame 与话题对照

命名规则：`{camera_name}_{stream}_optical_frame`，默认 `camera_name=camera`。

| 用途 | 话题 | `header.frame_id` / 数据坐标系 |
|------|------|-------------------------------|
| 左目（主相机 cam0） | `/camera/camera/infra1/image_rect_raw` | `camera_infra1_optical_frame` |
| 右目（cam1，立体） | `/camera/camera/infra2/image_rect_raw` | `camera_infra2_optical_frame` |
| 深度（原始） | `/camera/camera/depth/image_rect_raw` | `camera_depth_optical_frame` |
| 对齐 RGB 的深度 | `/camera/camera/aligned_depth_to_color/image_raw` | `camera_aligned_depth_to_color_optical_frame` |
| RGB | `/camera/camera/color/image_raw` | `camera_color_optical_frame` |
| 融合 IMU | `/camera/camera/imu` | `camera_imu_optical_frame` |
| 陀螺仪（未融合） | `/camera/camera/gyro/sample` | `camera_gyro_optical_frame` |
| 加速度计（未融合） | `/camera/camera/accel/sample` | `camera_accel_optical_frame` |

内参、畸变：对应 `*/camera_info`（与图像同光学系）。

> **注意**：话题前缀为 `/camera/camera/`（namespace + node name 各一层 `camera`），OpenVINS 配置里要写全。

---

## 4. IMU 数据解读（易混淆点）

### 4.1 加速度计含重力

`/camera/camera/imu` 的 `linear_acceleration` 是**比力（specific force）**，静止时近似 **重力 + 零运动加速度**。

相机水平放置、镜头朝前时（常见手持/支架）：

- 重力在光学系 **+Y（向下）** 方向；
- Intel 文档：静止时 **`linear_acceleration.y ≈ -9.8 m/s²`**（符号与安装姿态有关，模长约 9.81 正常）。

**这不是融合错误**，OpenVINS 的 `gravity_mag: 9.81` 也假定静止时能观测到重力。

### 4.2 角速度轴向（你遇到的现象）

| 你的动作（相机水平、镜头朝前） | 主要变化的 `angular_velocity` 分量 |
|-------------------------------|-------------------------------------|
| 绕 **竖直轴** 转（偏航 / yaw，像转圈） | **`.y`**（光学系 Y 与“上下”相关） |
| 绕 **光轴** 转（滚转 / roll，像拧镜头） | **`.z`** |
| 点头 / 俯仰（pitch） | **`.x`** |

若习惯 **ROS 系“Z 轴向上”** 的“绕 Z 转”，在 RealSense 光学系里对应的是 **绕 Y 轴**，因此看到 `angular_velocity.y` 变化是**正确的**。

### 4.3 融合话题 `/imu` 是否改轴

**不会。** 融合只把加速度计按时间插值到陀螺仪时间戳；角速度直接来自陀螺仪：

```text
angular_velocity  ←  gyro（原样）
linear_acceleration ← accel（插值对齐）
```

验证：

```bash
ros2 topic echo /camera/camera/gyro/sample --once
ros2 topic echo /camera/camera/imu --once
# 两者 angular_velocity 应一致（时间戳不同）
```

---

## 5. TF 树验证命令

启动 `rs_launch.py` 后：

```bash
# 光学系 ↔ 机体系
ros2 run tf2_ros tf2_echo camera_link camera_imu_optical_frame
ros2 run tf2_ros tf2_echo camera_infra1_frame camera_infra1_optical_frame

# 左目相对 IMU（ROS 系，仅旋转+平移参考）
ros2 run tf2_ros tf2_echo camera_imu_frame camera_infra1_frame
```

查看完整树：

```bash
ros2 run tf2_tools view_frames
# 生成 frames.pdf
```

---

## 6. 对接 OpenVINS 的推荐配置

### 6.1 传感器选择（D435i + 当前 launch）

| OpenVINS | RealSense 推荐 |
|----------|----------------|
| `cam0` | `/camera/camera/infra1/image_rect_raw`（左目，90Hz） |
| `cam1` | `/camera/camera/infra2/image_rect_raw`（右目，90Hz） |
| `imu0` | `/camera/camera/imu`（融合 IMU，`unite_imu_method:=2`） |

建议 **用红外双目做 VIO**，不用 color（30Hz、与 infra 不同步，且增加带宽）。  
`estimator_config.yaml` 建议：

```yaml
use_stereo: true
max_cameras: 2
calib_cam_extrinsics: true   # 首次可 true，有可靠标定后 false
calib_cam_timeoffset: true   # 建议标定 timeshift_cam_imu
```

### 6.2 `kalibr_imu_chain.yaml` 模板（需按实测噪声调整）

```yaml
imu0:
  rostopic: /camera/camera/imu
  update_rate: 200          # 接近 gyro FPS（默认约 200）
  time_offset: 0.0
  model: "kalibr"
  # 噪声需 Allan 方差或经验值，不可直接抄 D455/T265
  accelerometer_noise_density: 0.002
  accelerometer_random_walk: 0.0004
  gyroscope_noise_density: 0.0002
  gyroscope_random_walk: 0.00001
  Tw: [[1,0,0],[0,1,0],[0,0,1]]
  R_IMUtoGYRO: [[1,0,0],[0,1,0],[0,0,1]]
  Ta: [[1,0,0],[0,1,0],[0,0,1]]
  R_IMUtoACC: [[1,0,0],[0,1,0],[0,0,1]]
  Tg: [[0,0,0],[0,0,0],[0,0,0]]
```

### 6.3 `kalibr_imucam_chain.yaml` — **必须标定**

OpenVINS 读取 **`T_imu_cam`**（相机→IMU 的 4×4，即 Kalibr 的 `R_CtoI`, `p_CinI`）。  
也支持写 `T_cam_imu`（驱动会自动转换）。

**不要**直接复制 `openvins/config/rs_d455` 或 `rs_t265` 的外参——传感器安装不同，且 D435i 双目为 **infra 左/右**，不是 RGB。

标定流程概要：

1. 用当前 RealSense 话题录 bag（含 `/camera/camera/imu` + infra1/infra2 图像）；
2. 用 [Kalibr](https://github.com/ethz-asl/kalibr) 标定 IMU-相机链；
3. 将得到的 `T_cam_imu`、内参、`timeshift_cam_imu` 写入 OpenVINS 配置。

内参初值可从运行时读取：

```bash
ros2 topic echo /camera/camera/infra1/camera_info --once
ros2 topic echo /camera/camera/infra2/camera_info --once
```

`distortion_model: radtan`，`resolution: [640, 480]`（与 launch 中 90Hz 配置一致）。

### 6.4 OpenVINS 对变换的定义

源码 `VioManagerOptions.h` 中：

- 配置项名：`T_imu_cam`
- 含义：**从相机系到 IMU 系**（`R_CtoI`，相机原点在 IMU 系下的位置 `p_CinI`）

与 Euroc 数据集 `kalibr_imucam_chain.yaml` 注释一致。  
标定工具若输出 `T_cam_imu`，OpenVINS 也支持（会自动求逆兼容）。

---

## 7. 对接前自检清单

| 步骤 | 命令 / 检查 | 期望 |
|------|-------------|------|
| IMU 存在 | `ros2 topic list \| grep imu` | 有 `/camera/camera/imu` |
| 静止加速度 | `ros2 topic echo /camera/camera/imu --once` | 某轴模长 ≈ 9.81 |
| 陀螺一致性 | 对比 `/imu` 与 `/gyro/sample` 的 `angular_velocity` | 数值一致 |
| 双目频率 | `ros2 topic hz /camera/camera/infra1/image_rect_raw` | ~90 Hz |
| IMU 频率 | `ros2 topic hz /camera/camera/imu` | ~200 Hz（陀螺仪率） |
| TF | `tf2_echo camera_link camera_imu_optical_frame` | 有固定变换 |
| 外参 | `kalibr_imucam_chain.yaml` | **来自 Kalibr，非猜测** |

---

## 8. 常见错误

| 错误 | 后果 |
|------|------|
| 以为 IMU 在 ROS 机体系（Z 上） | 轴映射全错，OpenVINS 初始化失败 |
| 用 `/camera/imu` 而非 `/camera/camera/imu` | 订阅不到数据 |
| 用 color 30Hz 做 VIO 而 infra 90Hz | 时间同步差、特征跟踪不稳定 |
| 抄 D455/T265 外参 | 轨迹漂移、尺度错误 |
| 把 `linear_acceleration` 当“无重力加速度” | 与 OpenVINS 重力模型矛盾 |
| 忽略 `timeshift_cam_imu` | 快速运动时明显误差 |

---

## 9. 后续工作建议（OpenVINS 专用 config）

在 `openvins/src/open_vins/config/` 下新建例如 `rs_d435i/`：

```
rs_d435i/
├── estimator_config.yaml      # use_stereo: true, max_cameras: 2
├── kalibr_imu_chain.yaml      # rostopic: /camera/camera/imu
└── kalibr_imucam_chain.yaml   # infra1/infra2 + Kalibr 标定结果
```

启动示例：

```bash
# 终端 1
source ~/d1robot/realsense/install/setup.bash
ros2 launch realsense2_camera rs_launch.py

# 终端 2
source ~/d1robot/openvins/install/setup.bash
ros2 launch ov_msckf subscribe.launch.py config:=rs_d435i
```

（需先完成第 9 节中的 config 目录与标定文件。）

---

## 10. 参考

- [realsense-ros README - 坐标系章节](src/realsense-ros/README.md)（`ROS2 vs Optical Coordination Systems`）
- [Intel D435i IMU 坐标说明](https://github.com/IntelRealSense/librealsense/blob/master/doc/d435i.md)
- [OpenVINS 文档](https://docs.openvins.com/)
- [OpenVINS Kalibr 配置说明](https://docs.openvins.com/gs-calibration.html)

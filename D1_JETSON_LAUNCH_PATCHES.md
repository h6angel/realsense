# D1 / Jetson：RealSense launch 本地改动说明

本文档记录 **D435i + OpenVINS + EGO** 栈在 Jetson 实机上为缓解 **USB 带宽 / depth 丢失** 而对本仓库做的 launch 层改动。  
未修改 `realsense2_camera` 的 C++ 驱动源码。

相关文档：[COORDINATES_OPENVINS.md](COORDINATES_OPENVINS.md)、[JETPACK6_IMU.md](JETPACK6_IMU.md)。

---

## 1. Git 结构变更（2026-06）

原先 `src/realsense-ros` 在父仓库里是 **gitlink（submodule 指针）**，但没有 `.gitmodules`，无法在子目录内正常 `git commit`。

已改为 **普通目录（vendor tree）**：

```bash
git rm --cached src/realsense-ros
git add src/realsense-ros
git commit -m "vendor realsense-ros as plain tree"
```

之后对 launch 的修改在 `src/realsense-ros/...` 内直接 `git add` / `commit` 即可。

---

## 2. 改动的文件

| 文件 | 说明 |
|------|------|
| `src/realsense-ros/realsense2_camera/launch/rs_launch.py` | 唯一功能性修改 |

**不要手改 `install/`**：改 src 后执行 `colcon build`，install 会自动同步。

---

## 3. `rs_launch.py` 参数变更

| 参数 | 原默认值（上游 D435i） | 现默认值 | 原因 |
|------|------------------------|----------|------|
| `depth_module.infra_profile` | `640x360x90` | **`640x360x30`** | 降低 USB 负载；OpenVINS 实际跟踪约 30Hz，90Hz 多数被丢弃 |
| `depth_module.depth_profile` | `640x360x30` | 不变 | EGO GridMap 建图用 |
| `pointcloud.enable` | `true` | **`false`** | 关闭点云 topic，减 USB/CPU（EGO 不用点云建图） |

文件头注释已同步为 infra @ 30Hz 说明。

### 未改动的关键项（保持原 workspace 约定）

- 分辨率：**640×360**（infra 与 depth 必须一致，否则 depth 可能起不来）
- `enable_infra1` / `enable_infra2`：`true`（OpenVINS 双目）
- `enable_color`：`false`
- `align_depth.enable`：`false`
- `unite_imu_method`：`2`（融合 IMU）

---

## 4. 背景：为什么要降 Infra 帧率

实机 log 现象：

- GridMap 报 `odom or depth lost!`，`last_occ_update_time` 冻结 → 规划器 `Depth Lost! EMERGENCY_STOP`
- RealSense log 有 USB `Resource temporarily unavailable`
- `dmesg` 可见 D435i 反复 re-enumerate

OpenVINS 配置里 `track_frequency: 30`（见 `openvins/.../rs_d435i/estimator_config.yaml`），相机发 90Hz infra 时 **大部分帧在 OpenVINS 内被丢弃**，但仍占用 USB 带宽。

---

## 5. 配套改动（不在本仓库，联调时需一致）

| 仓库 | 改动 |
|------|------|
| **openvins** | `config/rs_d435i/estimator_config.yaml`：`track_frequency` **31 → 30** |
| **ego_control** | `grid_map/odom_depth_timeout`: **2.5s**；depth 恢复后清除 timeout；depth lost 不再永久 disable fail_safe；`start_ego_stack.sh` 等待 depth 首帧 |

完整栈由 `ego_control/start_ego_stack.sh` 启动。

---

## 6. 编译与生效

```bash
cd ~/d1robot/realsense
source /opt/ros/humble/setup.bash
colcon build --packages-select realsense2_camera
source install/setup.bash

# 确认 install 里已是 30Hz
grep infra_profile install/realsense2_camera/share/realsense2_camera/launch/rs_launch.py
# 期望: 640x360x30
```

启动（或由 `start_ego_stack.sh` 调用）：

```bash
ros2 launch realsense2_camera rs_launch.py
```

---

## 7. 实机自检

```bash
# Infra 应 ~30Hz（不再是 ~90Hz）
ros2 topic hz /camera/camera/infra1/image_rect_raw

# Depth 应 ~30Hz
ros2 topic hz /camera/camera/depth/image_rect_raw

# 深度非全 0（16UC1 中 0 表示无效）
ros2 topic echo /camera/camera/depth/image_rect_raw --once | head -30
```

---

## 8. 回滚

在 `rs_launch.py` 中恢复：

```python
{'name': 'depth_module.infra_profile', 'default': '640x360x90', ...},
{'name': 'pointcloud.enable',            'default': 'true', ...},
```

并将 openvins 的 `track_frequency` 改回 `31.0`，然后重新 `colcon build`。

临时覆盖（不改文件）：

```bash
ros2 launch realsense2_camera rs_launch.py \
  depth_module.infra_profile:=640x360x90 \
  pointcloud.enable:=true
```

---

## 9. 变更摘要

- **改了什么**：仅 `rs_launch.py` 默认参数（infra 30Hz、关 pointcloud）
- **没改什么**：RealSense 驱动 C++、标定、OpenVINS 标定 yaml
- **怎么进 git**：`src/realsense-ros` 已为普通目录，直接 commit 本仓库即可

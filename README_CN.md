# FASTLIO2_ROS2 项目文档

## ⚠️ 重要说明

**本文档针对完整的 FASTLIO2 SLAM 系统**。如果你只编译了 Livox-SDK2 和 livox_ros_driver2，那么你只能：

| 已编译 | 可运行的功能 |
|--------|-------------|
| ✅ Livox-SDK2 | 雷达 SDK 库 |
| ✅ livox_ros_driver2 | 雷达驱动，可以接收点云数据 |
| ❌ FASTLIO2 | SLAM 算法（需要单独下载编译） |

**目前你可以运行**：
```bash
# 测试雷达驱动（这个可以运行）
ros2 launch livox_ros_driver2 rviz_HAP_launch.py
```

**目前无法运行**：
```bash
# 这个需要编译 FASTLIO2 项目
ros2 launch fastlio2 lio_launch.py  # ❌ 会报错 "package 'fastlio2' not found"
```

**如需完整功能**，请继续查看下面的编译步骤，特别是**步骤 5: 编译 FASTLIO2_ROS2**。

---

## 🚀 快速开始（完整编译步骤）

如果你想要**快速搭建完整的 FASTLIO2 系统**，按顺序执行以下命令：

```bash
# ============================================================================
# 步骤 0: 设置代理（如果无法访问 GitHub）
# ============================================================================
git config --global http.proxy http://127.0.0.1:10808
git config --global https.proxy http://127.0.0.1:10808

# ============================================================================
# 步骤 1: 编译 Livox-SDK2
# ============================================================================
cd ~/Livox-SDK2
mkdir -p build && cd build
cmake ..
make -j$(nproc)
sudo make install
sudo ldconfig

# ============================================================================
# 步骤 2: 编译 livox_ros_driver2
# ============================================================================
cd ~/Livox-SDK2/ws_livox/src
# 如果已经克隆过，跳过这步
# git clone https://github.com/Livox-SDK/livox_ros_driver2.git

cd ~/Livox-SDK2/ws_livox
source /opt/ros/humble/setup.bash
./build.sh humble

# ============================================================================
# 步骤 3: 编译 Sophus
# ============================================================================
cd ~/Livox-SDK2/ws_livox/src/livox_ros_driver2/Sophus
git checkout 1.22.10
mkdir build && cd build
cmake .. -DSOPHUS_USE_BASIC_LOGGING=ON
make -j$(nproc)
sudo make install

# ============================================================================
# 步骤 4: 安装 FASTLIO2 依赖
# ============================================================================
sudo apt update
sudo apt install -y libyaml-cpp-dev ros-humble-pcl-conversions \
    ros-humble-tf2-ros ros-humble-nav-msgs ros-humble-geometry-msgs

# ============================================================================
# 步骤 5: 编译 FASTLIO2_ROS2
# ============================================================================
cd ~/Livox-SDK2/ws_livox/src
git clone -b ros2 https://github.com/HViktorTsoi/FAST_LIO.git fastlio2

cd ~/Livox-SDK2/ws_livox
source /opt/ros/humble/setup.bash
PATH=/usr/bin:$PATH colcon build --symlink-install

# ============================================================================
# 步骤 6: 设置环境变量（永久生效）
# ============================================================================
echo '' >> ~/.bashrc
echo '# Livox SDK2 and FASTLIO2' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/usr/local/lib' >> ~/.bashrc
echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc
echo 'source ~/Livox-SDK2/ws_livox/install/setup.bash' >> ~/.bashrc
source ~/.bashrc

# ============================================================================
# 完成！现在可以运行 FASTLIO2 了
# ============================================================================
ros2 launch fastlio2 lio_launch.py
```

---

## 📖 项目简介

**FASTLIO2_ROS2** 是一个基于 Livox 激光雷达和 IMU 的 **SLAM（即时定位与地图构建）系统**，是 FASTLIO2 的 ROS2 重构版本。

### 核心功能

| 功能模块 | 描述 |
|---------|------|
| 🔹 激光惯性里程计 (LIO) | 融合激光雷达和 IMU 数据，实时计算位姿 |
| 🔹 回环检测 | 识别闭环，消除累积漂移误差 |
| 🔹 重定位 | 在迷失位置时重新定位 |
| 🔹 一致性地图优化 | 优化多子地图拼接，输出完整 3D 地图 |

### 应用场景

- 🚗 自动驾驶汽车定位与建图
- 🤖 移动机器人导航
- 🚁 无人机 3D 建图
- 🗺️ 室内外环境三维重建

---

## 🏗️ 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        传感器层                                   │
│  ┌─────────────┐    ┌─────────────┐                             │
│  │ Livox 激光雷达 │    │    IMU      │                             │
│  └─────────────┘    └─────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        驱动层                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  livox_ros_driver2  (ROS2 雷达驱动)                        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        算法层                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   FASTLIO2   │  │  回环检测   │  │   重定位    │             │
│  │   (LIO)     │  │   (PGO)     │  │ (Localizer) │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│  ┌─────────────┐                                             │
│  │  地图优化   │                                              │
│  │ (HBA/BLAM) │                                              │
│  └─────────────┘                                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        应用层                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   RViz2     │  │  地图保存   │  │  路径规划   │             │
│  │  (可视化)   │  │             │  │             │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📦 环境依赖

### 系统要求

| 组件 | 版本要求 |
|------|---------|
| 操作系统 | Ubuntu 22.04 |
| ROS 版本 | ROS2 Humble |
| 编译器 | GCC/G++ 支持 C++14 |
| CMake | 3.14+ |

### 编译依赖

```bash
# 核心库
sudo apt install libpcl-dev libeigen3-dev libfmt-dev

# GTSAM (位姿图优化库)
sudo apt install libgtsam-dev

# Livox 驱动依赖
# 见下方编译步骤
```

---

## 🔧 详细编译步骤

### 步骤 1: 编译 Livox-SDK2

```bash
# 克隆仓库
git clone https://github.com/Livox-SDK/Livox-SDK2.git
cd Livox-SDK2

# 编译
mkdir build && cd build
cmake ..
make -j$(nproc)

# 安装
sudo make install
sudo ldconfig
```

### 步骤 2: 编译 livox_ros_driver2

```bash
# 创建工作空间
mkdir -p ~/ws_livox/src
cd ~/ws_livox/src

# 克隆驱动
git clone https://github.com/Livox-SDK/livox_ros_driver2.git

# 设置 ROS 环境
source /opt/ros/humble/setup.bash

# 编译
./build.sh humble

# 设置环境变量（永久生效）
echo 'source ~/ws_livox/install/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

### 步骤 3: 编译 Sophus (李代数库)

```bash
# 克隆仓库
cd ~/ws_livox/src
git clone https://github.com/strasdat/Sophus.git
cd Sophus
git checkout 1.22.10

# 编译 (使用基本日志模式，避免 fmt 依赖问题)
mkdir build && cd build
cmake .. -DSOPHUS_USE_BASIC_LOGGING=ON
make -j$(nproc)

# 安装
sudo make install
```

### 步骤 4: 编译 GTSAM (可选，用于回环优化)

```bash
# 方法 1: 使用 apt 安装
sudo apt install libgtsam-dev

# 方法 2: 从源码编译
git clone https://github.com/borglab/gtsam.git
cd gtsam
mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
```

### 步骤 5: 编译 FASTLIO2_ROS2

```bash
# 克隆项目 (选择一个)
cd ~/ws_livox/src

# 选项 1: HViktorTsoi 的 ROS2 版本 (推荐)
git clone -b ros2 https://github.com/HViktorTsoi/FAST_LIO.git fastlio2

# 选项 2: 其他 ROS2 移植版本
# git clone https://github.com/gaoxiang12/fastlio2_ros2.git fastlio2

# 编译
cd ~/ws_livox
source /opt/ros/humble/setup.bash
colcon build --symlink-install

# 设置环境变量
source install/setup.bash
```

**注意**：FASTLIO2_ROS2 不是官方项目，需要单独下载。如果你只编译了 livox_ros_driver2，那么只能运行雷达驱动，无法运行 `ros2 launch fastlio2 lio_launch.py`。

---

## 🚀 使用方法

### ⚠️ 运行前准备

确保已完成所有编译步骤，并设置环境变量：

```bash
source ~/Livox-SDK2/ws_livox/install/setup.bash
```

如果提示找不到 `fastlio2` 包，说明编译未完成，请先执行：

```bash
cd ~/Livox-SDK2/ws_livox
source /opt/ros/humble/setup.bash
PATH=/usr/bin:$PATH colcon build --symlink-install
source install/setup.bash
```

---

### 1️⃣ 激光惯性里程计 (LIO)

**最基础的定位建图功能**

```bash
# 终端 1: 启动 LIO 节点
ros2 launch fastlio2 lio_launch.py

# 终端 2: 播放数据集或连接雷达
ros2 bag play your_bag_file.db3

# 或者实时连接雷达
# (确保雷达已连接并配置好 IP)
```

**可视化**：
```bash
# 新终端打开 RViz2
rviz2

# 添加显示项:
# - PointCloud2: /cloud_registered (注册后的点云)
# - Path: /path (运动轨迹)
# - TF: 坐标系变换
```

**运行示例**：
```bash
# 终端 1
ros2 launch fastlio2 lio_launch.py

# 终端 2（播放数据集）
ros2 bag play ~/Datasets/fastlio2_test/00.db3 --clock

# 终端 3（可选，查看话题）
ros2 topic list
ros2 topic echo /fastlio2/path
```

---

### 2️⃣ 里程计 + 回环检测

**消除累积漂移误差**

> ⚠️ **注意**：回环检测功能需要 GTSAM 库支持。如果未安装 GTSAM，此功能无法使用。

```bash
# 终端 1: 启动 LIO
ros2 launch fastlio2 lio_launch.py

# 终端 2: 启动回环节点
ros2 launch pgo pgo_launch.py

# 终端 3: 播放数据集
ros2 bag play your_bag_file.db3
```

**保存优化后的地图**：
```bash
ros2 service call /pgo/save_maps interface/srv/SaveMaps \
  "{file_path: '/home/你的用户名/maps', save_patches: true}"
```

---

### 3️⃣ 里程计 + 重定位

**在迷失位置时重新定位**

```bash
# 终端 1: 启动 LIO
ros2 launch fastlio2 lio_launch.py

# 终端 2: 启动重定位节点
ros2 launch localizer localizer_launch.py

# 终端 3: 播放数据集
ros2 bag play your_bag_file.db3
```

**设置重定位初始位姿**：
```bash
ros2 service call /localizer/relocalize interface/srv/Relocalize \
  "{
    pcd_path: '/home/你的用户名/map.pcd',
    x: 0.0, y: 0.0, z: 0.0,
    yaw: 0.0, pitch: 0.0, roll: 0.0
  }"
```

**检查重定位结果**：
```bash
ros2 service call /localizer/relocalize_check interface/srv/IsValid "{code: 0}"
```

---

### 4️⃣ 一致性地图优化

**优化多个子地图的拼接**

```bash
# 前提：保存地图时设置了 save_patches: true

# 启动优化节点
ros2 launch hba hba_launch.py

# 调用优化服务
ros2 service call /hba/refine_map interface/srv/RefineMap \
  "{maps_path: '/home/你的用户名/maps'}"
```

---

## 📊 数据集使用

### 获取数据集

数据集包含录制的 ROS2 bag 文件，可用于离线测试。

### 播放数据集

```bash
# 查看 bag 文件信息
ros2 bag info your_bag_file.db3

# 播放数据集
ros2 bag play your_bag_file.db3

# 循环播放
ros2 bag play your_bag_file.db3 --loop

# 使用 bag 中的时间戳
ros2 bag play your_bag_file.db3 --clock

# 降速播放（避免性能不足）
ros2 bag play your_bag_file.db3 --rate 0.5
```

### 录制自己的数据集

```bash
# 录制所有 topic
ros2 bag record -a

# 录制指定 topic
ros2 bag record /livox/lidar /livox/imu
```

---

## 📡 Topic 和服务

### 主要 Topic

| Topic 名称 | 消息类型 | 描述 |
|-----------|---------|------|
| `/livox/lidar` | sensor_msgs/PointCloud2 | 原始激光雷达点云 |
| `/livox/imu` | sensor_msgs/Imu | IMU 数据 |
| `/cloud_registered` | sensor_msgs/PointCloud2 | 注册后的点云 |
| `/path` | nav_msgs/Path | 运动轨迹 |
| `/map` | sensor_msgs/PointCloud2 | 全局地图 |

### 主要服务

| 服务名称 | 服务类型 | 描述 |
|---------|---------|------|
| `/pgo/save_maps` | interface/srv/SaveMaps | 保存优化后的地图 |
| `/localizer/relocalize` | interface/srv/Relocalize | 重定位 |
| `/localizer/relocalize_check` | interface/srv/IsValid | 检查重定位结果 |
| `/hba/refine_map` | interface/srv/RefineMap | 地图优化 |

---

## 🔧 配置文件说明

### LIO 配置 (`config/lio_config.yaml`)

```yaml
# 频率设置
publish_freq: 10.0        # 点云发布频率 (Hz)

# 雷达配置
lidar_type: "livox"       # 雷达类型
lidar_ip: "192.168.1.100" # 雷达 IP 地址

# IMU 配置
imu_freq: 200.0           # IMU 频率 (Hz)
imu_acc_noise: 0.1        # 加速度噪声
imu_gyr_noise: 0.01       # 角速度噪声

# 特征提取
max_iteration: 5          # 最大迭代次数
threshold: 0.01           # 收敛阈值
```

### 回环配置 (`config/pgo_config.yaml`)

```yaml
# 回环检测
loop_detection_freq: 1.0   # 检测频率 (Hz)
keyframe_distance: 2.0     # 关键帧距离 (m)
icp_threshold: 0.1         # ICP 配准阈值

# 位姿图优化
optimization_freq: 1.0     # 优化频率 (Hz)
```

---

## ⚠️ 常见问题

### 1. 找不到共享库

**错误**：`liblivox_lidar_sdk_shared.so: cannot open shared object file`

**解决**：
```bash
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/usr/local/lib
sudo ldconfig
```

### 2. 编译时 Sophus 报错

**错误**：`fmt` 库依赖问题

**解决**：
```bash
# 在 CMakeLists.txt 中添加
add_compile_definitions(SOPHUS_USE_BASIC_LOGGING)

# 或编译时指定
cmake .. -DSOPHUS_USE_BASIC_LOGGING=ON
```

### 3. 性能不足导致卡顿

**解决**：
```bash
# 降速播放数据集
ros2 bag play your_bag_file.db3 --rate 0.5

# 或修改代码，将耗时回调独立到单独线程
```

### 4. RViz 中看不到点云

**检查**：
1. Fixed Frame 设置为 `livox_frame` 或 `map`
2. 确保 PointCloud2 插件已勾选
3. 检查 topic 是否正确

---

## 📝 核心概念解释

### 激光惯性里程计 (LIO)

融合激光雷达和 IMU 数据进行定位：
- **激光雷达**：提供精确的位置校正，但高频运动时会模糊
- **IMU**：提供高频运动估计，但会累积漂移

**类比**：闭着眼睛走路（IMU），走几步就睁眼看看周围（激光雷达）来纠正方向。

### 回环检测

当回到曾经到过的地方时，系统能识别并消除累积误差：

```
优化前：A → B → C → D → E → ? (漂移)
                    ↓
优化后：A → B → C → D → E → A ✓ (闭环)
```

### 重定位

机器人被移动到未知位置时，通过匹配当前点云和先验地图来确定位置。

### 地图优化

优化多个子地图的拼接，消除缝隙和重叠，输出一致的全局地图。

---

## 🙏 致谢

本项目基于以下开源项目：

- [FASTLIO2](https://github.com/hku-mars/FAST_LIO) - 激光惯性里程计
- [BLAM](https://github.com/erik-nelson/blam) - 3D 建图
- [HBA](https://github.com/TixiaoShan/HBA) - 层次化地图优化
- [GTSAM](https://github.com/borglab/gtsam) - 位姿图优化

---

## 📄 许可证

本项目遵循 [MIT 许可证](LICENSE)

---

## 📞 联系方式

如有问题，请提 Issue 或联系开发者。

---

## 📋 快速参考

### 常用命令速查

| 功能 | 命令 |
|------|------|
| **启动 LIO** | `ros2 launch fastlio2 lio_launch.py` |
| **启动回环检测** | `ros2 launch pgo pgo_launch.py` |
| **启动重定位** | `ros2 launch localizer localizer_launch.py` |
| **启动地图优化** | `ros2 launch hba hba_launch.py` |
| **播放数据集** | `ros2 bag play xxx.db3` |
| **循环播放** | `ros2 bag play xxx.db3 --loop` |
| **降速播放** | `ros2 bag play xxx.db3 --rate 0.5` |
| **录制数据** | `ros2 bag record -a` |
| **查看话题** | `ros2 topic list` |
| **查看节点** | `ros2 node list` |
| **保存地图** | `ros2 service call /pgo/save_maps ...` |

### 问题排查

| 问题 | 解决方案 |
|------|---------|
| `package 'fastlio2' not found` | 执行 `colcon build` 并 `source install/setup.bash` |
| `liblivox_lidar_sdk_shared.so` 找不到 | `export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/usr/local/lib` |
| Sophus 编译错误 | 添加 `-DSOPHUS_USE_BASIC_LOGGING=ON` |
| GTSAM 找不到 | 从源码编译安装 GTSAM |
| 运行卡顿 | 降速播放数据集 `--rate 0.5` |

### 目录结构

```
~/Livox-SDK2/
├── build/                    # Livox-SDK2 编译目录
├── include/                  # Livox-SDK2 头文件
├── sdk_core/                 # Livox-SDK2 核心代码
├── samples/                  # Livox-SDK2 示例程序
└── ws_livox/                 # ROS2 工作空间
    ├── src/
    │   ├── livox_ros_driver2/    # 雷达驱动
    │   │   └── Sophus/           # Sophus 库
    │   └── FASTLIO2_ROS2/        # FASTLIO2 项目
    │       ├── fastlio2/         # LIO 节点
    │       ├── pgo/              # 回环检测
    │       ├── localizer/        # 重定位
    │       └── hba/              # 地图优化
    ├── build/                # 编译产物
    ├── install/              # 安装目录
    └── devel/                # 开发目录
```

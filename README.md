# FASTLIO2_ROS2 & Livox-SDK2 整合工作空间

欢迎来到本整合项目！本仓库包含了一整套基于 Livox 激光雷达进行 SLAM（即时定位与建图）的完整解决方案，包括 **Livox-SDK2** 核心驱动库、**Livox ROS2 驱动**、**FAST_LIO 的 ROS2 版本**，以及相关的测试数据包和保存的地图，开箱即用（需先完成编译配置）。

## 📂 仓库目录结构

本仓库的当前文件夹结构及各个模块的说明如下：

```text
.
├── README.md                   # 本说明文件
├── README_CN.md                # 详细的系统架构说明及官方步骤备份
├── FASTLIO2_roadmap.md         # FASTLIO2_ROS2 版本的开发路线图与进度
├── relocation.md               # 模块级文档：重定位功能的详细测试与原理说明
├── 1.png ~ 8.png               # 说明文档配套展示图片 (如 relocation.md 及 Roadmap 中引用)
├── LICENSE.txt                 # 项目开源许可证
├── CHANGELOG.md                # 更新日志
│
├── map/                        # 用于存放系统生成的点云地图 (PCD 格式)
├── rosbag2_2024_06_20.../      # 预置的 ROS2 测试数据包 (rosbag)，可直接用于测试建图与定位
│
├── include/                    # \
├── sdk_core/                   #  > Livox-SDK2 核心底层库代码及头文件
├── samples/                    # /  (原厂自带示例及 SDK 实现)
├── build/                      # Livox-SDK2 编译生成的 build 目录
├── CMakeLists.txt              # Livox-SDK2 的 CMake 构建文件
│
└── ws_livox/                   # 核心 ROS2 工作空间 (SLAM 与驱动)
    └── src/
        ├── livox_ros_driver2/  # Livox 官方 ROS2 雷达驱动节点
        └── FASTLIO2_ROS2/      # FAST_LIO 的 ROS2 移植版核心代码
                                # (包含 LIO 里程计、PGO 回环检测、重定位及地图优化等节点)
```

---

## 🚀 快速开始

本项目针对 Ubuntu 22.04及 ROS2 Humble 环境设计配置。如果你刚克隆此仓库，请按以下步骤快速编译和运行整个系统：

### 1. 编译并安装 Livox-SDK2

首先编译底层的 Livox 官方 SDK 库：

```bash
cd Livox-SDK2
mkdir -p build && cd build
cmake ..
make -j$(nproc)
sudo make install
sudo ldconfig
```

### 2. 编译 ROS2 工作空间 (`ws_livox`)

```bash
# 进入 ROS2 工作空间
cd ~/Livox-SDK2/ws_livox

# 引入 ROS2 环境
source /opt/ros/humble/setup.bash

# 编译所有 ROS2 包 (包括 livox_ros_driver2 和 FASTLIO2_ROS2)
# 使用 symlink-install 可以方便后续的代码修改调试
colcon build --symlink-install
```

### 3. 配置环境变量 (推荐)

为了让系统能够找到刚刚编译的包，建议将以下命令添加到你的 `~/.bashrc` 中：

```bash
echo 'export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/usr/local/lib' >> ~/.bashrc
echo 'source ~/Livox-SDK2/ws_livox/install/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

---

## 🎮 节点运行与测试

通过预置的数据包测试算法是否正常工作：

### 启动 LIO 建图算法

打开终端运行 LIO（激光惯性里程计）节点：
```bash
ros2 launch fastlio2 lio_launch.py
```
*(如果没有自动打开 RViz，可新建终端运行 `rviz2` 并在界面中添加 PointCloud2 `/cloud_registered` 及 Path `/path` 以便查看)。*

### 播放预置数据包进行验证

在另一个终端播放仓库中附带的 `.db3` bag 包以提供传感器点云与 IMU 数据：

```bash
ros2 bag play ~/Livox-SDK2/rosbag2_2024_06_20-16_46_47/rosbag2_2024_06_20-16_46_47_0.db3
```

建图完成后，你可以查看 `maps/` 目录下相关的地图输出。如果是全功能版的 `FASTLIO2_ROS2`，还支持启动 PGO (回环节点) 或 Localizer (重定位节点)。

> 深入了解高级功能：  
> - 👉 关于**环境重定位功能**的测试和开发，请参阅 [`relocation.md`](relocation.md)
> - 👉 关于**当前版本的开发功能进度规划**，请参阅 [`FASTLIO2_roadmap.md`](FASTLIO2_roadmap.md)
> - 👉 关于**环境的完整配置细节及原架构解析**，请参阅 [`README_CN.md`](README_CN.md)

---

## 📞 技术交流与反馈

欢迎提交 Issue 和 Pull Request 至此仓库的 GitHub 页面，共同完善本 ROS2 SLAM 解决方案。

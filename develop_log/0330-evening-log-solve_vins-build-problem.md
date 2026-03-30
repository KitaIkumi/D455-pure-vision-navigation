# solve vins build problem

针对 Ceres API 世代断层（`LocalParameterization` 被废弃）的问题，我们现在开始执行**局部部署幽灵 Ceres 2.0.0** 的方案。这个方案完全符合 0 侵入原则：我们将 Ceres 编译为一个局部库，仅供当前的 `vins_ws` 消化，绝不触碰系统级 `/usr/local/lib`。

结合你的 N305 (纯 CPU) 环境，请依次执行以下步骤：

## 阶段一：下载并隔离编译 Ceres 2.0.0

为了保持目录整洁，我们在 `vins_ws` 下创建一个专用的 `third_party` 目录来存放它，并将编译产物安装到内部的 `ceres_install` 中。

**1. 下载源码** 考虑到你提到局域网可能有全局代理干扰，我们可以使用 `wget` 直接拉取 Release 包（如果卡住，请在 `wget` 前加上你的代理环境变量，例如 `https_proxy=... wget ...`）：

Bash

```
cd ~/vins_ws
mkdir -p third_party && cd third_party
wget https://github.com/ceres-solver/ceres-solver/archive/refs/tags/2.0.0.tar.gz
tar -xzvf 2.0.0.tar.gz
cd ceres-solver-2.0.0
```

**2. 局部编译与安装 (0 侵入)** 这一步的 CMake 配置非常关键。我们会强制关闭 CUDA（适配 N305），开启 `MINIGLOG`（绕过系统 glog 依赖冲突），并**强制指定安装路径**为工作空间内部。

你看到卡住的地方是 `autodiff_benchmarks` 和 `invert_psd_matrix_benchmark`。 **原因：** Ceres 默认会编译一大堆 **性能测试基准（Benchmarks）**。这些代码包含极其复杂的模板元编程（Template Metaprogramming），在编译时会消耗海量的内存和 CPU 资源。对于 N305 这种能效比核心来说，处理这些复杂的数学模板展开确实非常吃力，看起来就像死机了一样。

### 🛠️ 解决方案：跳过无意义的消耗

作为一个地面机器人导航项目，我们只需要 Ceres 的**库文件**（用于 VINS 的解算），完全不需要这些用来打分的 Benchmark 运行文件。

**不需要重新从头 make**。请按照以下 0 侵入且最高效的步骤重新配置并安装：

Bash

```
# 1. 重新进入 build 目录
cd ~/vins_ws/third_party/ceres-solver-2.0.0/build

# 2. 增加一条关键指令：-DBUILD_BENCHMARKS=OFF
# 这样 CMake 就会直接跳过最后那几个卡人的进度条
cmake .. \
  -DCMAKE_INSTALL_PREFIX=$HOME/vins_ws/ceres_install \
  -DCXX11=ON \
  -DBUILD_EXAMPLES=OFF \
  -DBUILD_TESTING=OFF \
  -DBUILD_BENCHMARKS=OFF \
  -DBUILD_SHARED_LIBS=ON \
  -DMINIGLOG=ON \
  -DUSE_CUDA=OFF \
  -DLAPACK=OFF \
  -DSUITESPARSE=OFF \
  -DCXSPARSE=OFF \
  -DEIGENSPARSE=ON

# 3. 再次编译。因为之前已经编译了 98%，这次它会秒速跳过已完成的部分
make -j8

# 4. 执行安装，这步很快，会将头文件和库移动到我们指定的隔离目录
make install
```

### 💡 如何确认成功了？

执行完 `make install` 后，运行以下命令检查：

Bash

```
ls ~/vins_ws/ceres_install/lib
```

只要看到 `libceres.so` 或 `libceres.a` 存在，就说明“幽灵 Ceres”已经成功捕获！

## 阶段二：使用 Colcon 强制定向链接 VINS-Fusion

在确认 Ceres 2.0.0 局部安装成功后，我们要回到工作空间根目录，通过 `colcon` 的 `--cmake-args` 将新的 Ceres 路径强行“喂”给 VINS，覆盖掉它默认去系统底层寻找新版 Ceres 的行为。

Bash

```
cd ~/vins_ws

# 清理一下之前的 build 缓存（如果有），防止旧的 CMake 缓存干扰
rm -rf build/ install/

# 开启强制链接编译
colcon build \
  --packages-up-to vins \
  --cmake-args \
  -DCeres_DIR=$HOME/vins_ws/ceres_install/lib/cmake/Ceres
```

*(注：`--packages-up-to vins` 是为了确保优先编译核心节点，具体的 package name 视你的 VINS-Fusion-ROS2 分支配置而定，通常是 `vins_estimator` 或 `vins`。如果想全量编译，直接去掉该参数即可。)*

太棒了！虽然看到一堆满屏黄色的 `warning`（这是从 ROS 1 迁移到 ROS 2 的典型遗留代码提示，比如 `ConstPtr` 已被废弃，以及 C++ 的一些语法警告），但请注意看最后两行：

Plaintext

```
Finished <<< vins [49.0s]
Summary: 2 packages finished [49.5s]
```

### **这意味着，我们成功了！** 

你已经通过 0 侵入的“幽灵 Ceres 降维”方案，完美突破了系统 Ceres 断层和 CUDA 缺失的封锁，在 N305 上彻底拿下了 `VINS-Fusion-ROS2` 视觉里程计孤岛。

接下来，我们将进入最终的收尾工作。

## 阶段三：编写 `start_env.sh` (一键级联净化与唤醒)

按照你制定的 0 侵入 SOP，我们需要在 `vins_ws` 根目录下放置一个 `start_env.sh` 脚本。这个脚本的作用是：在不修改系统 `~/.bashrc` 的前提下，一键挂载 ROS 2 基础环境、我们之前编译的幽灵 Ceres 库，以及刚才编译好的 VINS。

请在 `~/vins_ws` 下创建并编辑 `start_env.sh` 文件：

Bash

```
cd ~/vins_ws
nano start_env.sh
```

将以下内容粘贴进去：

Bash

```
#!/bin/bash
# ==========================================
# vins_ws 幽灵环境唤醒脚本
# 适配: N305 (纯 CPU) + D455
# 依赖: 局部 Ceres 2.0.0 (Eigen降维版)
# ==========================================

# 1. 挂载 ROS 2 底层环境 (Humble)
source /opt/ros/humble/setup.bash

# 2. 【核心隔离】将局部的幽灵 Ceres 库路径强行插入系统环境变量最前端
# 确保 VINS 运行时加载的是我们的 Eigen 降维版，而不是系统里自带的残缺版
export LD_LIBRARY_PATH=$HOME/vins_ws/ceres_install/lib:$LD_LIBRARY_PATH
export CMAKE_PREFIX_PATH=$HOME/vins_ws/ceres_install:$CMAKE_PREFIX_PATH

# 3. 挂载当前工作空间
source $HOME/vins_ws/install/setup.bash

echo "✅ vins_ws 环境已唤醒 (Ceres 2.0.0 Eigen-Only 挂载完毕)!"
```

保存退出后，记得赋予执行权限：

Bash

```
chmod +x start_env.sh
```

### 接下来？

至此，你已经完成了感知层 (`d455_ws`)、规划层 (`ego2d_ws`) 和现在的里程计层 (`vins_ws`) 这三大孤岛的局部部署。

# 极速地面机器人 2D 视觉导航系统 (N305 部署化简版)

## 0. 系统架构与 0 侵入工作流说明

- **计算平台**: Intel N305 (纯 CPU 环境) + Ubuntu 22.04 (Headless) + ROS 2 Humble。
- **部署原则 (0 侵入 SOP)**: 所有上层导航模块均在各自独立的 Workspace (`_ws`) 中隔离编译与运行。严禁修改系统全局环境（如 `~/.bashrc`）或使用 `apt` 替换底层冲突库。
- **环境唤醒**: 每个模块的根目录下都配备了 `start_env.sh` 脚本。运行该模块前，必须 `source` 对应的脚本以实现底层依赖和当前工作空间的**级联安全挂载**。

------

## 1. 感知驱动层 (D455 孤岛)

负责与底层硬件（Intel Realsense D455 深度相机）通信，打通现实物理世界与 ROS 2 系统的视觉数据流。

- **工作空间路径**: `~/d455_ws`

- **核心功能**: 启动相机节点，以预设的分辨率和帧率发布对齐的彩色图像、深度图像以及 IMU 数据（如果相机支持并开启）。

- **启动方式**:

  Bash

  ```
  source ~/d455_ws/start_env.sh
  ros2 launch realsense2_camera rs_launch.py align_depth.enable:=true
  ```

- **核心输入**: USB 3.1 物理连接的硬件数据。

- **核心输出 (发布的话题)**:

  - `/camera/color/image_raw` (RGB 彩色图像，供 VINS 提取特征点)
  - `/camera/aligned_depth_to_color/image_raw` (与彩色图对齐的深度图，供 EGO-Planner 建立局部障碍物栅格)
  - `/camera/imu` (相机的内置 IMU 数据，供 VINS 进行紧耦合解算)

------

## 2. 视觉里程计层 (VINS 孤岛)

系统的“小脑”，负责回答机器人“我在哪”的问题。针对 N305 进行了深度优化，切除了对 GPU 和高维数学库的依赖。

- **工作空间路径**: `~/vins_ws`

- **特殊依赖**: 内部集成了局部编译的**幽灵 Ceres 2.0.0 (纯 Eigen 降维版)**，强制定向链接，不与系统发生冲突。

- **核心功能**: 接收相机的图像（和 IMU），通过特征点追踪和非线性优化，实时输出机器人在全局地图中的位姿（坐标与姿态）。

- **启动方式**:

  Bash

  ```
  source ~/vins_ws/start_env.sh
  # 假设使用单目+IMU的配置文件
  ros2 launch vins vins_estimator.launch.py config_path:=<你的yaml文件路径>
  ```

- **核心输入 (订阅的话题)**:

  - `/camera/color/image_raw`
  - `/camera/imu`

- **核心输出 (发布的话题)**:

  - `/vins_estimator/odometry` (机器人的高频实时位姿 Odometry，直接喂给 EGO-Planner)
  - `/vins_estimator/point_cloud` (稀疏的特征点云，主要用于 Rviz 调试和可视化)





### 修改相机config

这份 D435i 的模板非常典型，但在直接给 D455 使用前，有几个**致命的逻辑错误**需要修正（特别是它默认关闭了 IMU，且没有适配 D455 那宽达 95mm 的物理基线）。

我已经为你修改好了适用于你的 N305 平台和 `soyo` 用户的专属 D455 配置文件。

### 完整的 `d455_stereo_imu_config.yaml`

请将以下内容完整覆盖到你的 `d455_stereo_imu_config.yaml` 文件中：

YAML

```
%YAML:1.0

# ==========================================
# 核心状态配置
# ==========================================
# support: 1 imu 1 cam; 1 imu 2 cam: 2 cam; 
imu: 1           # 【关键修改】原配置是0，必须改为1才能启用IMU，实现双目+IMU紧耦合
num_of_cam: 2    # 双目模式

imu_topic: "/camera/imu"
image0_topic: "/camera/infra1/image_rect_raw"
image1_topic: "/camera/infra2/image_rect_raw"
output_path: "/home/soyo/vins_ws/output/" # 【修改】适配你的 soyo 用户名工作区

cam0_calib: "left.yaml"   # 左目相机的内参文件（必须与此 config 放在同级目录）
cam1_calib: "right.yaml"  # 右目相机的内参文件
image_width: 640          # 确保你在 d455_ws 启动时 infra 分辨率也是 640x480
image_height: 480

# ==========================================
# N305 平台适配 (纯 CPU 模式)
# ==========================================
use_gpu         : 0
use_gpu_acc_flow: 0
use_gpu_ceres   : 0

# ==========================================
# 外参配置 (Extrinsic parameter)
# ==========================================
# 1: 相信初始值并进行在线微调 (推荐)
# 2: 抛弃初值，完全从零开始在线标定 (如果下面的矩阵错得离谱，可以改为2)
estimate_extrinsic: 1   

# D455 的左目到 IMU 的大致初始外参
body_T_cam0: !!opencv-matrix
   rows: 4
   cols: 4
   dt: d
   data: [ 1.0,  0.0,  0.0, -0.0302,
           0.0,  1.0,  0.0,  0.0074,
           0.0,  0.0,  1.0,  0.0160,
           0.0,  0.0,  0.0,  1.0 ]

# D455 的右目到 IMU 的大致初始外参 (主要体现了约 95mm 的基线宽度)
body_T_cam1: !!opencv-matrix
   rows: 4
   cols: 4
   dt: d
   data: [ 1.0,  0.0,  0.0, -0.1252,
           0.0,  1.0,  0.0,  0.0074,
           0.0,  0.0,  1.0,  0.0160,
           0.0,  0.0,  0.0,  1.0 ]

# ==========================================
# 算法性能参数 (适合地面机器人)
# ==========================================
multiple_thread: 1

# feature tracker parameters
max_cnt: 150            # 每帧提取的最大特征点数
min_dist: 30            # 特征点之间的最小像素距离
freq: 10                # VINS 发布的里程计频率 (Hz)
F_threshold: 1.0        # ransac threshold (pixel)
show_track: 1           # 开启特征点可视化 (可以在 Rviz 里看提取效果)
flow_back: 1            # 开启光流反向追踪，剔除误匹配点

# optimization parameters
max_solver_time: 0.04   # 限制最大求解时间(ms)，保证 N305 上的实时性
max_num_iterations: 8   # 限制最大迭代次数
keyframe_parallax: 10.0 # 关键帧选取的视差阈值

# ==========================================
# IMU 噪声参数 (BMI055/BMI085)
# ==========================================
acc_n: 0.1          # accelerometer measurement noise standard deviation.
gyr_n: 0.01         # gyroscope measurement noise standard deviation.
acc_w: 0.001        # accelerometer bias random work noise standard deviation.
gyr_w: 0.0001       # gyroscope bias random work noise standard deviation.
g_norm: 9.805       # 当地重力加速度

# ==========================================
# 时钟同步参数
# ==========================================
estimate_td: 1      # 必须开启！在线估计相机和 IMU 之间的通讯时间差
td: 0.0             # 初始时间差

# ==========================================
# 闭环检测与位姿图参数
# ==========================================
load_previous_pose_graph: 0        
pose_graph_save_path: "/home/soyo/vins_ws/output/pose_graph/" 
save_image: 0       # 关闭存图，节省 N305 的磁盘 I/O 负担
```

------

### ⚠️ 运行前的最后一道防线：缺失的 `left.yaml` 与 `right.yaml`

注意配置文件中的第 14、15 行：

YAML

```
cam0_calib: "left.yaml"
cam1_calib: "right.yaml"
```

VINS 要求把左右两个摄像头的内参（畸变参数、焦距等）单独写在两个 `.yaml` 文件里，并放在当前目录。**如果缺少这两个文件，运行 launch 时节点会直接崩溃闪退。**

**如何获取你这台相机的真实内参？** 既然我们坚持 0 侵入，就不去装庞大的标定工具箱了。Realsense 内部已经硬编码了出厂标定数据。

在 `d455_ws` 启动相机的状态下，新开一个终端，执行这个命令就可以直接监听底层话题拿数据：

Bash

```
ros2 topic echo /camera/infra1/camera_info --once
ros2 topic echo /camera/infra2/camera_info --once
```

拿到输出结果里的 `K` (内参矩阵) 和 `D` (畸变系数) 后，我们需要用它们生成那两个缺失的文件。需要我为你提供 `left.yaml` 和 `right.yaml` 的标准格式模板吗？你拿到数值后直接填进去即可。

这两个文件格式非常标准，使用的是 VINS 支持的针孔相机模型（PINHOLE）。

**但这里有一个绝对不能忽视的硬件差异**：D455 的镜头视场角（FOV）更广，内部焦距与 D435i 完全不同。因为我们在主配置文件中订阅的是 `image_rect_raw`（相机底层硬件已经做过畸变校正和极线对齐的图像），所以畸变参数 `k1, k2, p1, p2` 填 `0.0` 是完全正确的。但是，**投影参数（`fx, fy, cx, cy`）绝对不能照搬 D435i 的 382.8**。D455 在 640x480 分辨率下的焦距通常在 420 到 430 之间。

为了保持工作流的连贯性，我已经为你准备好了 D455 的 `left.yaml` 和 `right.yaml` 模板。请在 `~/vins_ws/src/VINS-Fusion-ROS2/config/realsense_d455/` 目录下创建这两个文件。

### 1. `left.yaml` (左目红外相机配置)

YAML

```
%YAML:1.0
---
model_type: PINHOLE
camera_name: camera
image_width: 640
image_height: 480
distortion_parameters:
   k1: 0.0
   k2: 0.0
   p1: 0.0
   p2: 0.0
projection_parameters:
   # 【警告】以下是 D455 的典型出厂估值，请务必替换为你这台相机的真实数值！
   fx: 423.50000000000000
   fy: 423.50000000000000
   cx: 318.50000000000000
   cy: 239.50000000000000
```

### 2. `right.yaml` (右目红外相机配置)

YAML

```
%YAML:1.0
---
model_type: PINHOLE
camera_name: camera
image_width: 640
image_height: 480
distortion_parameters:
   k1: 0.0
   k2: 0.0
   p1: 0.0
   p2: 0.0
projection_parameters:
   # 对于硬件级对齐的红外相机，右目的内参矩阵通常与左目完全一致
   fx: 423.50000000000000
   fy: 423.50000000000000
   cx: 318.50000000000000
   cy: 239.50000000000000
```

------

### 🛡️ 0 侵入：一键提取真实硬件参数 SOP

如果你用上面的典型值去跑，VINS 能运行，但大概率会发生“真实走1米，地图里走1.2米”的尺度漂移。为了达到极速机器人的导航精度，请务必花 1 分钟执行以下操作提取真实参数：

1. **唤醒感知层并启动相机**（确保你开启了红外流，通常在启动命令里加上 `enable_infra:=true`）：

   Bash

   ```
   source ~/d455_ws/start_env.sh
   ros2 launch realsense2_camera rs_launch.py enable_infra1:=true enable_infra2:=true align_depth.enable:=true
   ```

2. **新开一个终端，直接监听相机底层广播的真实内参**：

   Bash

   ```
   source /opt/ros/humble/setup.bash
   ros2 topic echo /camera/infra1/camera_info --once
   ```

3. **从终端输出中提取数据并填入 yaml**： 在输出结果中，你会看到一个叫 `K` 的数组，它代表了相机真实的内参矩阵。它的格式固定为： `K: [fx, 0.0, cx, 0.0, fy, cy, 0.0, 0.0, 1.0]`

   直接把你看到的这 4 个对应的数值替换掉我模板里的 `423.5` 和 `318.5/239.5`，你的 VINS 孤岛就彻底配置完美了。





------

## 3. 局部轨迹规划层 (EGO-Planner 2D 孤岛)

系统的“大脑”与避障核心，负责回答“我该怎么走”。

- **工作空间路径**: `~/ego2d_ws`

- **核心功能**: 接收给定的目标点，结合 VINS 提供的当前位姿和 D455 提供的深度图，在 2D 平面上动态构建局部障碍物地图，并高频生成一条平滑、无碰撞的 B 样条（B-spline）轨迹。

- **启动方式**:

  Bash

  ```
  source ~/ego2d_ws/start_env.sh
  ros2 launch ego_planner single_run_in_sim.launch.py # (后续需替换为真机 launch)
  ```

- **核心输入 (订阅的话题)**:

  - `/camera/aligned_depth_to_color/image_raw` (用于感知前方障碍物)
  - `/vins_estimator/odometry` (当前机器人的精准位置)
  - `/move_base_simple/goal` (或者自定义的全局路点下发话题)

- **核心输出 (发布的话题)**:

  - `/ego_planner/planning/bspline` (生成的平滑轨迹)
  - *(后续转化节点)*: 规划层最终需要输出速度指令（如 `cmd_vel` 或自定义协议），交由底盘执行。

------

### 关于后续的 EtherCAT 实车对接

你提到实车底盘是用 EtherCAT 通信的，现在先不碰实车，这个决定非常明智。

后续你们团队在联调时，系统的物理边界就在 `ego2d_ws` 的输出端。规划层算出轨迹或速度指令后，你们只需要写一个轻量级的 **ROS 2 控制桥接节点 (Bridge Node)**。这个节点的作用就是订阅规划出的指令，然后通过 EtherCAT 协议栈（比如开源的 SOEM 库）打包下发给电调或电机驱动器。

从常见的微控制器（比如用 STM32 跑串口或 CAN 通信的底盘开发）跨越到工业级、超高频实时的 EtherCAT 协议，确实需要一点时间去消化它的 PDO/SDO 映射机制。先在纯软件层面把“感知 -> 定位 -> 规划”这条数据流跑通，并在 Rviz 中验证无误，能为你们后面跑实车节省巨大的排错成本。
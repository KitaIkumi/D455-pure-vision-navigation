# D455-pure-visual-navigation-log1-20260330-env-set

**部署环境：** Ubuntu 22.04 / N305 miniPC (无独显) / 团队共享算力节点 

**核心架构：** 硬件驱动 (D455) + 视觉里程计 (VINS-Fusion) + 轨迹规划 (Ego-Planner-2D) 

**开发核心准则 (SOP)：** 绝对的“0 侵入 (Zero-Intrusion)”，采用“幽灵环境”与级联脚本唤醒，绝不修改 `~/.bashrc`，避免与队友的自瞄/雷达环境冲突。

## 🗺️ 工作空间总览 (The Isolated Archipelago)

我们在 `~/` 目录下构建了三个完全隔离的 ROS 2 工作空间，通过级联脚本进行联合调度：

1. `~/d455_ws`: 负责底层硬件通信与数据流发布。
2. `~/ego2d_ws`: 负责 2D 降维轨迹优化与避障决策。
3. `~/vins_ws`: 负责多传感器融合与高频视觉定位。

------

## 🛠️ 阶段一：底层视神经树立 (`~/d455_ws`)

**目标：** 安全接入 Intel D455 深度相机，打通 ROS 2 数据流。 **组件：** Intel 官方 `librealsense2` + `realsense-ros` (ROS 2 Humble wrapper)

- **执行动作：**
  - 通过 Intel 官方 PGP 公钥和 apt 源，在系统底层仅安装了最纯净的 `librealsense2-dev`，打通 Udev 硬件识别。
  - 克隆 `realsense-ros` 源码进行独立编译。
- **遇到的问题与破局：**
  1. **依赖冲突危机：** `rosdep` 试图安装社区版 `ros-humble-librealsense2`，这会导致系统存在双版本底层库，引发严重冲突。
     - *解决方案：* 使用 `--skip-keys="librealsense2"` 强行跳过该包的检测。
  2. **API 版本超前报错：** 编译时报 `RS2_FRAME_METADATA_OCCUPANCY_GRID_ROWS was not declared`。原因是克隆的 `ros2-development` 激进分支比系统安装的稳定版底层 SDK 要新。
     - *解决方案：* 时光倒流，`git checkout 4.54.1` 切换到完全匹配 Humble 的稳定 Tag，重新清理 CMake 缓存后满血编译成功。
- **SOP 产物：** 编写了初始版本的 `start_env.sh`，实现系统基础环境与相机环境的叠加唤醒。

## 🧠 阶段二：运动大脑降维 (`~/ego2d_ws`)

**目标：** 部署基于浙大 FAST-Lab 二次开发的真正 2D 轨迹优化器，释放 CPU 算力。 **组件：** `Ego-Planner-2D-ROS2` (纯 C++ 核心，砍掉 Z 轴计算)

- **执行动作：**
  - 克隆代码后，进行了防御性 `rosdep` 模拟，确认无环境污染风险。
- **遇到的问题与破局：**
  1. **CMake 版本过低壁垒：** Ubuntu 22.04 默认 CMake 为 3.22，但该库强制要求 3.28+。常规升级会破坏全局环境。
     - *解决方案：* 实施“幽灵 CMake”战术。下载官方 CMake 3.28.3 免安装版至工作空间内。修改级联脚本，通过 `export PATH` 仅在当前终端劫持接管编译工具链，成功骗过编译器。
- **SOP 产物：** 升级了 `start_env.sh`，加入了“幽灵 CMake”路径，并实现了系统 -> 相机 -> 规划器的三级环境级联挂载。

## ⚖️ 阶段三：视觉小脑桥接 (`~/vins_ws`)

**目标：** 部署 VINS-Fusion (Humble ARM 适配版)，将相机的图像和 IMU 转化为 Ego-Planner 所需的高频里程计。 **组件：** 社区 Humble 分支 (`JanekDev/VINS-Fusion-ROS2-humble-arm`)

- **执行动作：**
  - 通过 `apt` 安装了数学库 `libceres-dev`。
  - 避开了作者提供的危险级 `install_external_deps.sh` 脚本，防止系统 OpenCV 4 被暴力降级替换。
- **遇到的问题与破局：**
  1. **Github 克隆超时 (443)：** 机载电脑直连受限。
     - *解决方案：* 采用 `mirror.ghproxy.com` 镜像加速前缀完成拉取。
  2. **硬件配置不符 (CUDA 报错)：** 仓库默认开启 GPU，但 N305 仅有核显。
     - *解决方案：* 物理深入源码，注释掉 `feature_tracker.h` 中的 `#define GPU_MODE 1`。
  3. **Ceres API 世代断层 (连环报错)：**
     - *现象：* `global_fusion` 和 `camera_models` 疯狂报错找不到 `LocalParameterization`。
     - *原因：* Ubuntu 22.04 自带的新版 Ceres (2.x) 删除了旧版流形 API，而 VINS 源码太老未同步更新。
     - *破局 1 (局部截断)：* 针对完全用不到的 RTK-GPS 融合节点 `global_fusion`，使用 `COLCON_IGNORE` 直接封印，节省算力。
     - *破局 2 (幽灵 Ceres)：* 针对核心模块的报错，我们没有去降级系统，而是在 `~/vins_ws` 下局部编译了一个 Ceres 2.0.0 的稳定版，并在 `colcon build` 时通过 `-DCeres_DIR` 强制重定向链接。
- **SOP 产物：** 形成了终极版的 `~/start_nav_env.sh`，能够一次性、无污染地唤醒包括幽灵 CMake、底层驱动、VIO 定位和轨迹规划在内的全链路视觉导航环境。

------

### 下一步计划 (Next Steps)

目前，这三大战区的基础设施已经全部竣工！但是，这三个模块现在还是相互独立的“孤岛”。小车如果要真正跑起来，我们必须通过修改配置文件，把它们的数据流像水管一样串联起来。

按照逻辑顺序，我们的下一步应该是：

1. **唤醒硬件潜能**：修改 `realsense2_camera` 的 Launch 文件，开启 D455 的高频 IMU 同步 (`enable_sync`) 以及深度流对齐 (`align_depth`)。
2. **校准 VINS 眼睛**：基于 D455 的物理参数，编写专属的 VINS `yaml` 配置文件（填入内外参）。

## AI prompt



```markdown
# [用户上下文与系统工作流说明]

你好！在接下来的对话中，请基于以下我的硬件环境、系统状态、开发工作流限制以及当前项目进度，来提供 ROS 2 视觉导航的后续代码、配置和解决方案。

## 1. 硬件与系统环境
* **系统**: 远程 Ubuntu 22.04 工作站 (Headless 纯命令行操作，无图形桌面介入，ROS 2 Humble)。
* **硬件**: 搭载 Intel N305 miniPC (纯 CPU 环境，无独立显卡 / 无 CUDA)，以及 Intel D455 深度相机（通过 USB 3.1 Gen 2 连接）。
* **应用场景**: 面向极速地面机器人（Ground Robot）的纯 2D 视觉导航与避障。
* **网络**: 局域网 SSH 直连。系统中存在干扰性的全局代理环境变量，会导致 `rosdep`、`colcon` 或 HTTPS 拉取报错。

## 2. 核心原则：0 侵入 (Zero-Intrusion) SOP
这是一台团队共享算力节点，队友部署了自瞄、雷达等易碎的全局环境。
* **绝对禁令 1**: 严禁提供任何会修改系统全局配置文件（如 `~/.bashrc`, `~/.profile`）的命令。
* **绝对禁令 2**: 严禁使用系统级的 `sudo apt install` 安装或替换容易引起冲突的底层依赖（如降级 OpenCV, Ceres, Eigen），除非绝对必要且影响可控。
* **隔离与唤醒**: 所有开发必须在 `~` 目录下的独立工作空间 (`_ws`) 中进行。通过在每个工程根目录下编写 `start_env.sh` 来实现幽灵环境的一键级联唤醒与净化。

## 3. 当前项目进度 (已完成两座孤岛的部署)
目前已完成以下源码的隔离编译，均通过级联脚本实现了安全挂载：
1. **感知驱动层 (`~/d455_ws`)**: 编译了匹配 Humble 的 `realsense-ros` (v4.54.1 稳定 Tag)。
2. **轨迹规划层 (`~/ego2d_ws`)**: 部署了纯 C++ 降维的 `Ego-Planner-2D-ROS2`。通过局部注入便携版 CMake 3.28 成功绕过了系统低版本限制。

## 4. 当前主线任务与挑战
目前，**视觉里程计层 (`~/vins_ws`)** 的部署正在进行中，代码使用的是 `VINS-Fusion-ROS2-humble-arm` 分支。
* **已解决**: 已在源码强制关闭 GPU/CUDA，并使用 `COLCON_IGNORE` 封印了不需要的 GPS 融合节点。
* **当前卡点 (待执行)**: 遇到了 Ceres API 世代断层问题（系统 Ceres 版本过新，删除了旧版 `LocalParameterization`）。
* **计划方案**: 准备在 `~/vins_ws` 下局部下载并编译旧版 `Ceres 2.0.0`（幽灵 Ceres），并通过 CMake 参数强制定向链接给 VINS，不污染系统。

了解以上背景后，请回复“已对齐 0 侵入工作流”，并等待我发送具体的编译日志或下一步指令。
```




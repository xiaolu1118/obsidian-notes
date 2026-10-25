---
created: 2026-10-25
updated: 2026-10-25
tags: [ROS2, 内网, 离线部署, 开发环境]
---

# 内网 ROS 2 开发环境部署

## 背景与结论

用户询问内网如何部署 ROS 2，并已确认目标系统为 Ubuntu 22.04，既没有可用内网 APT 源，也没有安装 Docker。外网机器可用性、CPU 架构、直接出网能力、真实硬件及 GUI 需求尚未确认。以下方案为建议，未在用户设备执行安装。

## 无内网源且无 Docker：当前执行路线

用户已确认两个条件都不具备，因此当前采用 **外网准备 deb 软件包 → 通过批准渠道带入 → 内网建立本地文件 APT 仓库 → 原生安装 Humble** 的建议路线。无需先安装 Docker，也无需在内网搭建 HTTP 服务器。APT 可从 file: 仓库读取依赖。

### 第一步：内网收集基础信息

在目标 Ubuntu 终端执行：

```bash
lsb_release -ds
dpkg --print-architecture
uname -m
locale
dpkg --audit
```

再导出目标机器包状态用于外网解析，避免外网机器已安装依赖被漏掉：

```bash
mkdir -p "$HOME/ros2-offline-info"
cp /var/lib/dpkg/status "$HOME/ros2-offline-info/dpkg-status"
dpkg-query -W -f='${Package}\t${Version}\t${Architecture}\n' \
    > "$HOME/ros2-offline-info/packages.tsv"
cp /etc/os-release "$HOME/ros2-offline-info/os-release"
```

这些操作只读取系统信息并生成本地文件，不安装软件。若 dpkg --audit 有输出，应先诊断未完成安装。不要手动修改 /var/lib/dpkg/status。

### 第二步：外网准备同版本、同架构的打包环境

使用能联网的 Ubuntu 22.04；外网只有 Windows 时可创建 Ubuntu 22.04 虚拟机。目标 amd64 与 arm64 需区别准备。外网是否具备此条件尚待确认。

按 ROS 官方 Humble 文档配置 Ubuntu jammy（含所需 Universe、更新、安全更新）及 ROS 2 官方仓库。先选择交付集合：学习用 ros-humble-desktop 和 ros-dev-tools；服务器用 ros-humble-ros-base，另加示例包；Nav2、MoveIt、Gazebo 和硬件 SDK 按项目需要额外加入。

用目标机器 dpkg 状态在独立 APT 状态/缓存目录中模拟安装，解析并下载实际所需 deb；也可从空状态准备更大的完整依赖集合，但仍须对目标状态模拟验证。不能只执行 apt download ros-humble-desktop，也不能直接复制外网宿主 apt 缓存。包状态、APT 源快照、架构及已有包锁定可能影响依赖求解。

ROS 官方关于 Jammy 早期 systemd/udev 的提醒仍适用，离线包集合需包含经模拟验证的必要更新；不要在内网直接执行不受控的全系统升级。

### 第三步：生成并验收离线仓库

在外网使用 dpkg-scanpackages/apt-ftparchive 生成 Packages、Release 等索引，签名 Release 并导出仓库公钥。公钥指纹应通过可信渠道核对，只在该仓库的 signed-by 配置中使用；不使用全局关闭 APT 验签的方法。

建议交付结构：

```text
ros2-humble-offline/
  repo/                 # deb、Packages、Release、InRelease
  ros2-offline-keyring.gpg
  SHA256SUMS
  package-manifest.tsv
  install-notes.txt
```

在无网络、与目标基础状态一致的测试环境中检查依赖和安装过程，不能仅凭下载成功宣称离线部署成功。记录软件源快照及包版本，避免打包过程中源更新引入不一致。

### 第四步：内网安装

带入并校验后放置到例如 /opt/ros2-humble-offline，将该仓库公钥放到 /usr/share/keyrings。仓库及路径应对 APT 读取用户可读。

本地源配置示例（仅在对应路径和签名材料已经准备好后使用）：

```text
deb [signed-by=/usr/share/keyrings/ros2-offline-keyring.gpg] file:/opt/ros2-humble-offline/repo ./
```

更新与安装命令应显式仅使用本地源，并使用独立索引目录或事先清理该独立目录，以免残留在线索引参与求解。先模拟安装 ros-humble-desktop ros-dev-tools，检查是否有缺包、降级、保留包冲突或关键系统包移除。确认方案后正式安装。具体命令须与实际打包目录及目标信息匹配，当前尚未生成或执行。

### 第五步：验收

source /opt/ros/humble/setup.bash 后运行 talker/listener；编译一个自己的 C++/Python 包；需要 GUI 时运行 RViz。Gazebo 模型、地图、Python wheel、驱动 SDK 和源码子模块应单独纳入资源清单。

### 下一项待确认

- [ ] 目标 dpkg --print-architecture、uname -m 和 dpkg --audit 结果。
- [ ] 是否有能联网的电脑，是否可在外网运行 Ubuntu 22.04 虚拟机。
- [ ] 是否有 sudo 权限，以及学习桌面环境还是无 GUI 服务器。

依据：[APT 命令手册](https://manpages.ubuntu.com/manpages/jammy/man8/apt-get.8.html)、[APT 仓库验签](https://manpages.ubuntu.com/manpages/jammy/man8/apt-secure.8.html)。

## 当前环境：Ubuntu 22.04（优先阅读）

2026-10-25 用户补充系统版本后，当前部署建议调整为 **Ubuntu 22.04 + ROS 2 Humble**。下方原 Jazzy 示例保留为其他系统的参考，不适用于本次宿主系统的原生安装。

### 有内网 APT 镜像时

请先确认内网源包含 Ubuntu jammy 的所需组件、更新与安全更新，以及 ROS 2 Humble 仓库，且完成签名验证与源配置。只配置 Ubuntu 镜像通常找不到 ros-humble 包。不要把未知的仓库地址直接套入配置。

```bash
lsb_release -ds
dpkg --print-architecture
locale
sudo apt update
apt-cache policy ros-humble-desktop
```

确认 ros-humble-desktop 有 Candidate 后，先检查 Ubuntu 更新方案。ROS 官方特别提醒 Ubuntu 22.04 早期 systemd/udev 包应在安装 ROS 前更新，避免关键系统包被移除。在干净开发机可先 sudo apt upgrade；已有项目的机器应预览更新计划并安排维护窗口。

```bash
sudo apt install ros-humble-desktop ros-dev-tools
source /opt/ros/humble/setup.bash
```

desktop 含 RViz、示例等；无图形需求可选择 ros-humble-ros-base，并按需补 demo 包。以上命令依赖可用且完整的内网 APT 源。

### 完全离线且已有 Docker Engine 时

在外网同架构 Linux 机器准备：

```dockerfile
FROM ros:humble-ros-base-jammy
RUN apt-get update && apt-get install -y --no-install-recommends \
    cmake gdb \
    ros-humble-demo-nodes-cpp ros-humble-demo-nodes-py \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /ws
```

```bash
docker build -t ros2-dev:humble-v1 .
docker save -o ros2-dev-humble-v1.tar ros2-dev:humble-v1
sha256sum ros2-dev-humble-v1.tar > ros2-dev-humble-v1.tar.sha256
```

将镜像和校验文件带入内网，在文件目录执行：

```bash
sha256sum -c ros2-dev-humble-v1.tar.sha256
docker load -i ros2-dev-humble-v1.tar
mkdir -p "$HOME/ros2_ws/src"
docker run -it --name ros2-humble --network host \
    -v "$HOME/ros2_ws:/ws" ros2-dev:humble-v1 bash
```

容器一终端 source /opt/ros/humble/setup.bash 后运行 ros2 run demo_nodes_cpp talker；宿主另一终端执行 docker exec -it ros2-humble bash，source 同一 setup 文件后运行 ros2 run demo_nodes_py listener。两端 ROS_DOMAIN_ID 必须一致。

该镜像为命令行环境，不包含 RViz/Gazebo 图形接入。默认 root 可能让挂载工作区产生 root 所有文件，正式环境应配置开发用户。目标若未装 Docker，Docker Engine 及其依赖也需单独离线准备；不能只带 ROS 镜像。

完全离线且不允许 Docker 时，应准备 Humble + Jammy 的完整 deb 依赖集合或签名的本地 APT 仓库。不能将 sudo apt install 误认为离线安装步骤。需要目标架构、现有源和包状态后再制定具体打包方法。

### Humble 网络配置

```bash
source /opt/ros/humble/setup.bash
export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
```

两机同域、可互通并允许所用 DDS 发现/数据通信后验证。单机学习可设 ROS_LOCALHOST_ONLY=1；多机需设 0 或取消该变量。下文 ROS_AUTOMATIC_DISCOVERY_RANGE / ROS_STATIC_PEERS 是 Jazzy 示例，不应直接套用 Humble；禁用多播的网络可按 Fast DDS 文档配置 Discovery Server。

参考：[Humble 官方安装源文档](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Installation/Ubuntu-Install-Debs.rst)、[Humble 官方镜像 Dockerfile](https://raw.githubusercontent.com/osrf/docker_images/master/ros/humble/ubuntu/jammy/ros-base/Dockerfile)。

ROS 2 本地运行不需要互联网；安装包、源码、构建依赖、模型及地图等资源需要提前准备。部署同时涉及软件供应和 DDS 局域网通信两条链路。

建议：单机学习优先准备可导入的 Ubuntu 虚拟机；团队采用内网 APT/PyPI/Git/容器仓库；已有 Linux 和 Docker 的纯软件开发采用预构建镜像。USB、GPU、实时控制和多机通信需单独验证。

示例基线为 Ubuntu 24.04 + ROS 2 Jazzy，并非宣称其为最新版本。已有课程、驱动或项目指定发行版时应优先匹配。amd64 与 arm64 软件包和镜像需分别准备。

## 方案选择

| 环境 | 建议 |
| --- | --- |
| 内网能通过批准的出口访问外网 | 使用公司代理/镜像源，固定版本及依赖 |
| 多人长期开发但不能出网 | 内部 APT 仓库、Python wheel 仓库、Git 镜像和容器镜像库 |
| 完全离线、少量机器 | 外网构建好镜像或虚拟机，校验后导入 |
| 真机传感器、GPU、实时控制 | 原生 Ubuntu 或经过验证的 Linux 容器方案 |

## 完全离线 Docker 示例

前提：内网机器已安装 Docker Engine，使用原生 Linux 主机；外网构建机与目标 CPU 架构一致。Windows Docker Desktop/WSL2 的网络、USB 与图形接入不能直接照搬本示例。

外网创建 Dockerfile：

```dockerfile
FROM ros:jazzy-ros-base
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential cmake git gdb \
    python3-colcon-common-extensions python3-rosdep \
    ros-jazzy-demo-nodes-cpp ros-jazzy-demo-nodes-py \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /ws
```

外网构建、导出并校验：

```bash
docker build -t ros2-dev:jazzy-v1 .
docker run --rm ros2-dev:jazzy-v1 bash -lc 'source /opt/ros/jazzy/setup.bash; ros2 pkg prefix demo_nodes_cpp; colcon --help >/dev/null'
docker save -o ros2-dev-jazzy-v1.tar ros2-dev:jazzy-v1
sha256sum ros2-dev-jazzy-v1.tar > ros2-dev-jazzy-v1.tar.sha256
```

基础镜像标签会更新。团队交付时记录基础镜像 digest、完整包版本、构建日期和 Dockerfile，并保存最终镜像，避免只依赖浮动标签。

将 tar、校验文件和项目源码通过批准渠道带入内网：

```bash
sha256sum -c ros2-dev-jazzy-v1.tar.sha256
docker load -i ros2-dev-jazzy-v1.tar
mkdir -p "$HOME/ros2_ws/src"
docker run -it --name ros2-dev --network host \
    -v "$HOME/ros2_ws:/ws" ros2-dev:jazzy-v1 bash
```

此示例仅供命令行开发，默认 root 用户可能在挂载目录创建 root 所有文件；团队镜像应配置匹配宿主 UID/GID 的开发用户。host 网络示例针对原生 Linux，GUI、GPU、串口需按需配置，勿直接用全量特权替代设备映射。

容器终端一：

```bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=42
ros2 run demo_nodes_cpp talker
```

宿主第二终端进入同一容器：

```bash
docker exec -it ros2-dev bash
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=42
ros2 run demo_nodes_py listener
```

验收：listener 持续收到消息，再断开外网重做，并编译运行一个自己的包。以上为方案示例，未在用户环境实测。

## 依赖闭包

不能只下载 ROS 主包或复制外网机器的 apt 缓存：已安装的依赖可能不再下载，内网会缺包。应在与目标系统、架构和基础状态一致的干净环境解析依赖，并在干净离线环境验收。

导入清单：系统/ROS deb 包及索引签名、公钥；项目源码与子模块及固定提交；package.xml 对应系统依赖；Python wheel 及传递依赖；编译工具；驱动 SDK；仿真模型、纹理、地图、数据集；离线文档与 IDE 扩展。

rosdep 是依赖解析工具，不是离线包仓库。完全离线需要镜像 rosdep/rosdistro 元数据并配置本地来源，且另备实际软件包；或外网把项目依赖直接构建进开发镜像。不要在内网依赖默认的 rosdep update、git clone 或 pip 在线解析。

已有项目应先在外网运行 rosdep 解析安装、colcon 构建与测试，再导出镜像，并记录源码提交及资源清单。源码工作区在内网开发时可挂载，而系统依赖固定在镜像中。

## 团队长期方案

搭建 Ubuntu 与 ROS 2 APT 镜像/快照仓库、Python wheel 仓库、Git 服务、容器仓库及模型资源库。APT 需保持有效仓库元数据和签名；只有缓存代理不能保证未缓存软件包在断网后可用。可先维护经验证的目标项目依赖集合，无需完整镜像所有发行版。

通过开发环境版本清单绑定 OS、架构、ROS 发行版、RMW、编译工具链、驱动 SDK 和项目提交。配置内网 DNS 与时间同步。发布前在无外网环境从零重建并验证。

## 内网多机通信

ROS 2 常用 DDS 发现与传输，不需要配置 ROS 1 的 ROS_MASTER_URI。先统一 ROS_DOMAIN_ID，初期统一 RMW；检查路由、防火墙、网卡选择和多播，而非仅验证 ping。

同网段可在两机各设置 ROS_DOMAIN_ID=42、ROS_AUTOMATIC_DISCOVERY_RANGE=SUBNET，再运行 talker/listener。只需单机开发可设 LOCALHOST。

Jazzy 在受支持的 RMW 下可通过 ROS_STATIC_PEERS 指定对端，例如 LOCALHOST 范围加对端地址 192.168.10.22；需按所用中间件文档验证。跨网段/禁止多播时还可考虑 Fast DDS Discovery Server。发现服务不等于转发全部数据，数据路径仍需可达。DOMAIN_ID 是逻辑分组，不是鉴权机制。

调整发现或 RMW 配置后重启节点，必要时停止旧 ros2 daemon。避免沿用只允许 localhost 的配置排查多机通信。

## 待确认与验收

- [x] 目标系统确认：Ubuntu 22.04；建议使用 Humble。
- [ ] 能否使用 Docker，是否需要原生 GUI 与硬件接入。
- [ ] 完全离线还是可用内网镜像/批准代理。
- [ ] amd64/arm64，目标发行版，是否接雷达、相机、机械臂及 GPU。
- [ ] 离线首次启动、包编译、收发、日志回放和所需 GUI/仿真通过。

## 参考与关联

- [ROS 2 Jazzy Ubuntu 安装](https://repo.test.ros2.org/en/jazzy/Installation/Ubuntu-Install-Debs.html)
- [官方 ROS Dockerfile](https://github.com/osrf/docker_images/blob/master/ros/jazzy/ubuntu/noble/ros-base/Dockerfile)
- [Docker save](https://docs.docker.com/reference/cli/docker/image/save/)
- [ROS 2 Jazzy 发现配置源码文档](https://github.com/ros2/ros2_documentation/blob/jazzy/source/Tutorials/Advanced/Improved-Dynamic-Discovery.rst)
- [[机器人开发技术栈与入门路线]]
- [[机器人开发/索引|机器人开发索引]]

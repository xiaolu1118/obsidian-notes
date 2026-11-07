---
created: 2026-11-06
updated: 2026-11-06
tags: [Android, AAOS, 车机, 职业转型]
status: 建议路线，待确认目标岗位
---

# 手机 Android 系统工程师转车机开发路线与教程

2026-11-06 补充：用户已明确优先多屏专项，具体深入路线见 [[车机多屏专项深入路线与实验]]；目标岗位及实践环境仍待确认。

## 背景与结论

- 用户本轮明确需求：了解从手机行业转车机开发需要的知识，并整理视频或文档。
- 既有笔记背景：Android 系统框架开发工程师；本轮未重新确认具体模块及年限。
- 建议：优先衔接 Android 座舱 Framework / 系统服务 / 多屏方向；按目标 JD 再选电源、音频或车辆接入专项。
- 待确认：目标岗位、现有模块经验、每周学习时间。以下不是用户已确认的转型决策，也不承诺学习周期内可以就业。
- 返回 [[车机开发/索引|车机开发索引]]。

## 先区分岗位

| 方向 | 主要工作 | 补课重点 |
| --- | --- | --- |
| 车载应用 / HMI | Launcher、设置、媒体、空调界面等 | 车载交互、Car API、应用生命周期、媒体与权限 |
| 座舱 Framework | CarService、多屏多用户、电源、音频、系统定制 | AAOS 系统架构与端到端排障，建议优先 |
| BSP / HAL | 平台适配、驱动、车辆接口、启动与休眠 | C/C++、Linux、AIDL HAL、SELinux、硬件调试 |
| MCU / AUTOSAR / 车辆控制 | 实时嵌入式与车辆控制软件 | C、RTOS、AUTOSAR、车载通信，属于另一条路线 |

Android Auto 是手机侧体验投射到车上的平台；Android Automotive OS（AAOS）直接运行于车载硬件。AAOS 与 Google Automotive Services（GAS）也应分开理解，学习 AOSP 车机并不等于默认拥有 Google 生态。官方应用教程主要覆盖应用开发，不能替代车厂系统定制知识。

## 知识地图与优先级

### P0：架构、车辆属性、系统调试

- 保留并复用 Java/Kotlin、Binder/AIDL、Handler、SystemServer、AMS/ATMS/WMS、权限、Linux 与性能定位经验。
- 理解链路：系统/授权应用 → Car API / CarPropertyManager → CarService / CarPropertyService → VHAL → 厂商车辆通信适配 → MCU/网关/车辆网络。
- 厂商链路不一定直接接 CAN，也可能经 Socket、以太网或厂商中间件；VHAL 不是 CAN 协议本身。
- 属性重点：property ID、area ID、访问模式、变化模式、采样、时间戳、状态、订阅、读写异常和权限。
- AIDL VHAL 为主；阅读旧项目时补 HIDL，具体接口以目标 Android 分支为准。
- 熟练 adb、logcat、dumpsys、bugreport、Perfetto；能区分应用、服务、HAL 与下层信号故障。

### P1：选择一个系统专项深入

| 专项 | 核心内容 | 与手机经验的衔接 |
| --- | --- | --- |
| 多屏多用户 | Display、Task、用户与 Occupant Zone 的映射、输入焦点、跨屏启动 | WMS/ATMS 经验可直接延伸 |
| 电源 | CarPowerManagementService、车辆电源状态机、挂起恢复、关机协调、Garage Mode | 比单纯屏幕亮灭多了车辆状态协作 |
| 音频 | CarAudioService、Audio Focus、ducking、音量组、音频区域、路由 | 导航、媒体、通话等场景仲裁 |
| 系统 UI | CarSystemUI、CarLauncher、CarSettings、RRO、配置与权限 | 系统应用和 ROM 定制经验 |

### P2：车辆与工程交付常识

- CAN/CAN FD、DBC 信号定义、大小端、比例与偏移、周期与超时；先能理解信号和属性如何映射。
- MCU/网关职责、ACC/点火与电源状态；SOME/IP、车载以太网、UDS/DoIP 按岗位继续深入。
- 启动速度、长稳、ANR/Crash、内存、休眠功耗、OTA/回滚、SELinux、兼容性和自动化测试。
- 了解功能安全、网络安全、ASPICE 的作用；认证与流程深度依岗位要求，当前不作为入门阻塞项。
- Qt/QML、QNX、Hypervisor、AUTOSAR 只在目标岗位要求时加入，避免同时铺开多条栈。

## 教程与文档导航

检索日期：2026-11-06。优先官方文档与发布方原始视频。中文可先读 AOSP 中文版，接口细节对照英文与源码；视频旧版本的命令/API 需复核。

### 视频

1. [Doulos：Introduction to Android Automotive](https://www.doulos.com/events/on-demand/introduction-to-android-automotive/)：英文点播讲座，覆盖 AAOS、GAS、CarService、Car Managers 和 VHAL，适合先搭架构。访问时可能需要注册，以页面要求为准。
2. [Android Developers：What's new in cars，Google I/O 2025](https://www.youtube.com/watch?v=ud09zuXHst4)：英文，偏应用体验、Car App Library、工具和分发；用于了解生态，不当作 Framework 系列课。
3. [Android Developers：What's new with Android for Cars，Google I/O 2023](https://www.youtube.com/watch?v=ii7PGDG0G5g)：英文，补应用平台背景；较旧，实际开发以当前文档为准。

观看顺序建议：Doulos 架构入门 → 官方应用更新选看 → 文档和源码实操。YouTube 访问依网络条件。未找到足够可核验的中文完整系统课程，本轮不把搜索关键词或未经验证的付费课包装成精选课程。

### 文档和源码

| 顺序 | 资料 | 用法 |
| --- | --- | --- |
| 1 | [AOSP Automotive 中文总入口](https://source.android.com/docs/automotive?hl=zh-cn) | 建立系统知识目录，沿 IVI 与具体模块学习 |
| 2 | [VHAL 概览](https://source.android.com/docs/automotive/vhal?hl=zh-cn) | 重点学属性模型、AIDL 接口和实现要求 |
| 3 | [CarService 源码与分支](https://android.googlesource.com/platform/packages/services/Car/+refs) | 选择目标 Android tag，跟踪 car-lib、service 与测试，不混读多个版本 |
| 4 | [车载电源](https://source.android.com/docs/automotive/power) | 电源状态、关机协调、挂起恢复 |
| 5 | [车载音频](https://source.android.com/docs/automotive/audio) | 音频焦点、音量组、区域和路由 |
| 6 | [多用户](https://source.android.com/docs/devices/admin/multi-user) | 阅读 Automotive 多用户多屏部分，并结合 WMS/ATMS |
| 7 | [AAOS 模拟器](https://developer.android.com/training/cars/testing/emulator) | 先跑起应用与基础实验，后续系统修改使用可控 AOSP 构建 |
| 8 | [Android 官方 car-samples](https://github.com/android/car-samples) | 跑通应用样例；不是完整车厂系统工程 |
| 9 | [Linux SocketCAN](https://www.kernel.org/doc/html/latest/networking/can.html) | 有车辆接入需求时学虚拟 CAN 与 Socket API |
| 10 | [Android for Cars 应用入口](https://developer.android.com/training/cars) | 学应用类别、Car App Library、驾驶限制和测试 |

补充：普通第三方应用不能默认读取/控制全部车辆属性。实验要核对属性权限、系统签名/特权身份和镜像支持；界面 mock 数据与真实 VHAL 贯通须分别标注。

## 建议实践安排：8 周，每周 8～12 小时

周期是基于已有 Android 系统经验的建议，用于产出入门项目，不是就业保证。

| 阶段 | 学习与操作 | 验收产物 |
| --- | --- | --- |
| 第 1 周 | 架构视频、总览、启动 AAOS AVD | 区分 Auto/AAOS/GAS，画出架构和进程边界 |
| 第 2～3 周 | 跟踪属性读取/订阅源码，在可控环境验证模拟属性变化 | 一条从 UI 到 VHAL 的调用链、权限与异常说明 |
| 第 4～5 周 | 优先多屏专项；也可按 JD 换成电源或音频 | 一个正常场景、两个异常场景及定位记录 |
| 第 6～7 周 | 整合车辆属性监控与诊断 Demo | 订阅、状态显示、数据过期判断、日志导出 |
| 第 8 周 | 梳理长稳与故障案例，整理项目说明和面试问答 | 可演示工程、架构图、复现步骤、问题闭环 |

项目建议：车辆属性监控与诊断 Demo。在开发镜像中选择实际支持且允许访问的属性，模拟变化、不可用和服务重连；区分权限失败、未支持属性、数据过期与服务故障。系统方向再加入一个多屏/电源/音频扩展。应用样例可先在 Android Studio AVD 运行；修改 CarService/VHAL 往往需要匹配分支、可调试镜像和 Linux/AOSP 构建环境，不能假设普通 Play 镜像可直接改系统服务。初期不用先买开发板。

## 下一步待办

- [ ] 根据目标 JD 确认应用、Framework、BSP 中的具体方向。
- [ ] 确认现有 WMS/AMS、电源、音频等经验，选择一个车载专项。
- [ ] 跑通模拟器与样例，记录镜像/API/源码版本。
- [ ] 完成一条属性链路和一个异常定位案例。

关联：[[wms窗口/WMS学习路线与56天每日计划]]、[[系统框架面试/索引]]。本轮已完成资料检索与路线归档，未执行课程学习、环境搭建或项目开发。

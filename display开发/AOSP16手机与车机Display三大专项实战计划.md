# AOSP 16 手机与车机 Display 三大专项实战计划

创建日期：2026-11-07。更新日期：2026-11-07。
返回：[[display开发/索引|Display 索引]]；基础：[[Display Framework深入学习与实践调试路线]]。

## 已确认范围与建议安排

- 已确认：AOSP 16、手机 Display Framework 岗位；亮度、亮灭屏、多屏都需要掌握；同时加入车机学习。
- 建议：先用手机建立可验证的 DMS/DPC 基础，再对照 AAOS 增加车辆电源、乘员区域和多用户；建议学习精力约 70% 手机实操、30% 车机对照，非用户已确定时间分配。
- 车机内容按 Android Automotive OS（AAOS）理解；Android Auto 投屏不作为本计划主线。
- 待确认：具体 tag/QPR、厂商扩展、是否有 AAOS 镜像或台架、源码编译能力及设备权限。没有车机环境时先做源码分析和模拟器实验，并明确未完成硬件验收。
- 当前仅完成规划与资料核对，以下实验均未运行。

## 1. 建立 AOSP 16 基线

先记录 repo manifest revision、设备 fingerprint、overlay、显示配置和实际 feature flags。android16-release 是阅读入口，不能替代项目固定提交；同为 Android 16，不同 QPR/厂商分支也可能改变策略。

AOSP 源码仓库根目录执行以下搜索（不是在本 Obsidian 库执行）：

```powershell
rg -n 'requestPowerState|updatePowerState|requestDisplayState' frameworks/base/services/core/java/com/android/server
rg -n 'class .*Brightness|BrightnessReason' frameworks/base/services/core/java/com/android/server/display
rg -n 'class (DisplayDeviceRepository|LogicalDisplayMapper|LocalDisplayAdapter|VirtualDisplayAdapter)' frameworks/base/services/core/java/com/android/server/display
rg -n 'class (CarPowerManagementService|CarDisplayService|CarOccupantZoneService)' packages/services/Car
```

搜索结果用来确认项目真实入口，再通过调用者和被调用者建立调用图。不要将关键词列表当成已经验证的完整调用顺序。

## 2. 专项一：亮灭屏——优先学习

直观理解：PMS 提出是否需要点亮的需求，DPC 协调显示状态及亮度，WMS/策略层参与可见性与亮屏条件，最终还要等待下游执行。要把“系统醒着”“显示 ON”“解锁完成”“首帧可见”分别记录。

### 手机源码与问题

阅读 PowerManagerService/PowerGroup → DisplayManagerInternal/DMS → DisplayPowerController；随后检查 DisplayPowerState、DisplayPowerProximityStateController、LocalDisplayAdapter 及对应 Native/SF 出口。以上是模块导航，实际调用先后、状态入口和 feature flag 应在项目分支追踪。

逐项回答：

1. wakefulness、display policy、显示 state 的含义和关系是什么？
2. 谁请求唤醒，谁提交显示请求，谁决定最终状态？
3. pending request 如何进入 Handler？锁内外分别做什么？
4. screen-on blocker、接近传感器、窗口绘制等条件分别阻塞哪一步？
5. 哪个回调表示框架准备完成，哪些硬件完成证据还需另外采集？

### 手机实验

- 电源键唤醒与超时灭屏：各跑正常对照，记录请求到显示状态变化再到首帧的阶段时间。
- 锁屏有无、接近传感器遮挡（设备支持时）、AOD/Doze（设备支持时）分别做单变量实验。
- 连续亮灭屏循环先少量人工测试，再根据团队流程扩大次数；记录黑屏发生在哪个阶段。

验收：对一个“唤醒后黑屏”案例，能区分未收到唤醒请求、DPC 条件未满足、窗口未准备好、合成/输出异常四类路径，并列出支持证据。

### 车机对照

在通用链路上游增加车辆电源控制：VMCU/VHAL、CarPowerManagementService、车辆电源策略及 CarDisplayService 等集成模块。绘制 AP 电源状态机和每块屏状态机两张图，避免把灭屏当成整机 suspend。

台架实验：车辆 ON → 准备休眠 → suspend → 恢复；再测试仅关闭副驾屏、主屏继续工作。记录车辆请求、CarService 决策、DMS/DPC 状态及显示恢复结果。车辆事件仅在模拟器或授权台架按团队流程注入，具体命令依据实际 VHAL 实现。

## 3. 专项二：亮度——建立完整的决策账本

直观理解：用户滑块是意图，框架综合场景与约束决定目标，动画逐步改变输出；设置值、目标值、当前值、nits、实际面板效果不能互换。

### 阅读顺序

1. Settings/窗口亮度覆盖等请求源 → DMS/DPC 的输入。
2. brightness/ 下的控制器与策略、DisplayBrightnessState/BrightnessReason 等结果和原因表达。
3. AutomaticBrightnessController、BrightnessMappingStrategy：传感器输入、过滤/迟滞、映射和用户调整。
4. DisplayDeviceConfig：设备能力与曲线配置；检查实际生效的配置来源和 overlay。
5. 范围与限制相关模块：BrightnessRangeController、HBM、热/功耗限制，以及项目 brightness/clamper 等实现。
6. RampAnimator、DisplayPowerState 以及实际下发出口。

自动亮度、策略选择与限制的确切顺序以项目源码为准，不能假定限制仅来自一个模块。

### 实验矩阵

| 变量 | 手机实验 | 车机对照 | 重点证据 |
|---|---|---|---|
| 手动亮度 | 固定环境分档调节 | 主副屏分别调节 | 设置、目标、当前及对应显示标识 |
| 自动亮度 | 稳定光照与阶跃变化 | 车辆环境光来源和配置 | 原始/过滤 lux、映射结果、切换时刻 |
| 覆盖请求 | 对比普通窗口与窗口亮度覆盖 | 车载系统 UI/业务覆盖 | 最终选择原因与请求来源 |
| 限制 | 节电条件；热限制按现有测试工具验证 | 车辆电源策略与屏级限制 | 约束、上限、最终目标 |
| 状态恢复 | 亮灭屏、重启前后 | suspend/resume、用户切换前后 | 保存位置、恢复时机、是否串屏 |

车机需查明背光实际归属：Android 管理、VHAL/MCU 管理或厂商桥接；不要预设手机自动亮度链路可以原样套用。先确认本项目使用的车辆属性与桥接实现，再追请求与回报是否构成环路。

每条亮度记录至少包含：时间、显示/用户标识、请求源、自动模式、lux、选中策略与 reason、限制原因、目标值、动画当前值、下发/回报。缺少的字段标为未知，不用推测填充。

验收：解释一次“滑块变化但亮度没变化”和一次“亮屏瞬间过亮”，找到目标计算错误还是动画/执行/恢复时序错误。

## 4. 专项三：多屏——追三张对应表

三张表：物理设备↔逻辑显示；逻辑显示↔任务/窗口/图层；输入设备↔显示/焦点。车机再加乘员区域↔用户↔显示。

通用入口：LocalDisplayAdapter / VirtualDisplayAdapter → DisplayDeviceRepository → LogicalDisplayMapper / LogicalDisplay → DMS 回调 → WMS/ATMS；配合 SF 显示与图层状态。此处表达职责联系，不代表严格同步调用链。

手机先做：

1. 用模拟辅助屏验证逻辑显示与应用回调；再用实际支持的外接屏验证物理接入。
2. 分别验证镜像、Presentation、目标屏启动、跨屏移动，记录其行为差异。
3. 外屏应用运行时拔屏，检查任务回退、资源释放、焦点和输入恢复。
4. 主副屏分别切换电源/亮度（能力和权限支持时），验证是否意外影响其他屏。
5. 若有折叠设备，再学 DeviceStateToLayoutMap 与布局映射；折叠状态切换不等同于普通外屏热插拔。

车机增加：CarOccupantZoneService/Manager、CarUserService 与可见后台用户配置、CarInputService 和输入关联、CarDisplayService。先确认镜像是否启用并发可见多用户；不能假设所有 AAOS 镜像默认支持。

车机验收场景：主驾导航、副驾视频 → 仅副屏灭屏 → 副屏恢复 → 副驾用户停止/切换 → 整机休眠恢复。逐项检查用户、任务、焦点、输入、IME、亮度归属和首帧；屏幕编号不可作为永久座位标识。

更完整的任务与输入实验复用 [[车机开发/车机多屏专项深入路线与实验]]。

## 5. 调整后的八周执行表

| 周 | 手机主线 | 车机对照 | 验收 |
|---|---|---|---|
| 1 | AOSP16 基线、显示对象、正常 dump | AAOS 模块与车辆电源概念 | 一张架构图、对象映射表 |
| 2 | 唤醒/灭屏链路、blocker | CPMS 与 AP 电源状态机 | 两张时序图与正常基线 |
| 3 | 亮屏慢/黑屏、Doze/接近场景 | 灭屏与 suspend 的区别、恢复 | 一份证据完整的排障报告 |
| 4 | 手动亮度、策略、动画 | 各屏亮度和背光归属 | 亮度决策账本 |
| 5 | 自动亮度、限制、状态恢复 | 车辆光照/昼夜策略与恢复 | 两个可复现实验 |
| 6 | 多屏设备注册、逻辑映射、任务 | occupant zone/user/display | 四类标识对应表 |
| 7 | 热插拔、输入、独立屏状态 | 主副驾并发及休眠恢复 | 多屏回归矩阵 |
| 8 | 一个真实问题的最小修复 | 对照跨屏和恢复风险 | 修改、证据、回归、回滚说明 |

SF、刷新率和 FrameTimeline 作为支撑内容按问题穿插学习。没有车机设备的周任务只验收源码与模拟器可验证部分，车载物理电源/背光验收保留待办。

## 6. 统一调试方法

复现前保存版本与正常基线；复现中记录触发时刻并持续采集日志/trace；问题后收集 display、power、window、SF、input、sensorservice 文本 dump。车机另收集 car_service（先检查服务是否存在）和项目 VHAL/车辆事件记录。命令模板见基础路线；动态 trace 需按版本和权限确认数据源。

先用 dump 找状态差异，再用 trace 解释先后关系，最后到源码找到第一次错误写入。日志记录 old→new、reason、display/user、调用线程和异步完成条件。每轮只验证可区分的假设，避免同时改策略、延时和配置。

回归至少考虑：冷启动/重启、锁屏与解锁、快速亮灭屏、手动/自动亮度切换、用户切换、外屏拔插、主副屏独立操作、休眠恢复。按实际支持能力裁剪并记录未测项。

## 参考资料

- [AOSP16 Display 源码目录](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/android16-release/services/core/java/com/android/server/display/)：已核对上述核心目录与多项类名；具体方法调用仍需项目源码验证。
- [AAOS 电源管理](https://source.android.com/docs/automotive/power/power)：CPMS、VHAL 与 AP 电源状态概念。在线文档可能更新，不能把其中新增行为直接认作 AOSP16 固有实现。
- [CarOccupantZoneManager](https://developer.android.com/reference/android/car/CarOccupantZoneManager)：乘员区域和显示关联 API，使用前核对目标版本。

下一步待办：用项目分支追一次电源键唤醒，画出请求、策略、阻塞、下发、完成五个阶段；同步采集正常基线。该任务优先于通读全部源码。

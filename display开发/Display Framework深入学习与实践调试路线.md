# Display Framework 深入学习与实践调试路线

创建日期：2026-11-07。更新日期：2026-11-07。
返回：[[display开发/索引|Display 学习索引]]。

## 1. 背景与学习目标

已确认：用户刚从事 Display Framework 工程师，主要负责框架层业务。
用户补充确认（2026-11-07）：AOSP 16，当前手机平台，同时加入车机内容；覆盖亮度、亮灭屏和多屏。细化执行方案见 [[AOSP16手机与车机Display三大专项实战计划]]。
假设：每天投入约 1～2 小时；以项目实际 release/QPR/tag 和厂商改动为准。芯片平台、调试权限及实际可用时间待确认。

建议目标：8 周建立独立分析常见问题的基本能力；通过后续真实缺陷与需求迭代深化，不能将完成计划等同于成为专家。

验收能力：能说明一次请求由谁发起、策略由谁决定、状态在哪里更新、如何下发、何时完成；能用 dump/trace 区分策略错误、状态未同步与执行延迟；完成至少一个最小源码改动及回归。

## 2. 先形成整体画面

可以把屏幕系统理解为剧场：DMS 管理有哪些舞台及其规格；WMS 决定窗口如何摆放；DisplayPowerController 管理灯光和亮灭状态；SurfaceFlinger 将画面组合起来；HWC 与显示硬件协作执行输出。比喻用于记忆，实际链路存在回调、并行和跨进程通信。

分开理解两条链：

- 管理与策略链：App/Settings/Power/WMS → DisplayManagerService（DMS）及其控制模块 → SurfaceControl/SF 或相关 HAL。具体出口取决于业务，亮度与电源不能机械套用同一条路径。
- 像素链：App 渲染 → Surface/BufferQueue 等缓冲传递机制 → SurfaceFlinger → HWC/显示硬件。新版本内部实现与传递路径有变化，不能将 BufferQueue 图直接当作所有路径的完整时序。

必须区分：物理屏和逻辑显示；displayId 和 SF 物理显示标识；窗口与图层；请求状态与实际状态；亮屏与首帧可见；App 帧率与屏幕刷新率；modeId 与分辨率/刷新率；亮度设置值、框架亮度值和实际发光效果。

## 3. 源码阅读地图

以下是常见 AOSP 入口，类位置、方法和职责边界应在项目分支搜索验证，不是固定调用栈。

| 主线 | 常见入口 | 阅读时要回答 |
|---|---|---|
| 设备与逻辑显示 | frameworks/base/services/core/java/com/android/server/display/ 下 DisplayManagerService、LocalDisplayAdapter、DisplayDevice、LogicalDisplay、LogicalDisplayMapper（分支存在时） | 热插拔如何转为设备、逻辑显示及应用回调？ |
| 亮灭屏与亮度 | DisplayPowerController、DisplayPowerState、brightness/ 子目录、AutomaticBrightnessController、DisplayDeviceConfig；结合 PowerManagerService | 谁提出电源请求？谁阻塞亮屏？亮度经过哪些限制？ |
| 刷新率 | display/mode/ 下 DisplayModeDirector 等；SF Scheduler/刷新率策略代码 | 哪些约束参与决策？候选范围是什么？最终实际模式是什么？ |
| WMS 协作 | frameworks/base/services/core/java/com/android/server/wm/ 下 DisplayContent、DisplayPolicy 等 | 布局、旋转、显示信息如何同步？ |
| 合成与输出 | frameworks/native/services/surfaceflinger/；相关 SurfaceControl JNI/Native 实现 | 图层是否可见？buffer 是否到达？合成与 present 是否完成？ |

阅读顺序：入口请求 → 关键状态 → handler/线程和锁 → 策略 → 下发接口 → 回调/完成条件 → dump 输出。每次只追一个具体场景，不从 DMS 构造函数逐行通读全部源码。

每条链画一张时序图，标注线程、Binder、锁、异步消息和状态变更点。尤其警惕把“提交成功”误当成“硬件已经完成”。

## 4. 八周实践计划（建议）

| 周次 | 重点 | 实践 | 验收产物 |
|---|---|---|---|
| 1 | 显示对象与取证 | 固定设备分支；正常亮屏、灭屏、旋转前后各抓 dump | 架构图、显示标识对照表、正常基线 |
| 2 | 显示生命周期 | 用开发者选项模拟辅助显示，或受控外接屏热插拔；监听 DisplayListener | 设备到逻辑显示到应用回调时序图 |
| 3 | 亮灭屏 | 对比电源键唤醒、超时灭屏、解锁过程；如有条件加入接近传感器场景 | 请求状态、目标状态、实际状态及阻塞条件表 |
| 4 | 亮度 | 手动亮度分档，自动亮度固定环境对照；检查覆盖值与限制因素 | 亮度请求到输出链路图、异常假设清单 |
| 5 | 刷新率 | 在设备支持范围内切换设置、滚动、播放视频、节电场景 | 请求、策略约束、目标模式、实际模式四列记录 |
| 6 | SF 与性能 | 做可控动画；采集 Winscope/Perfetto；分析一帧及一次错帧 | 带证据的帧时间线，不把长帧直接归因于显示驱动 |
| 7 | 岗位专项 | 手机按实际职责选 AOD/折叠/亮度；车机选热插拔/多屏/电源联动 | 一个真实业务完整调用链与边界条件 |
| 8 | 源码闭环 | 开发分支增加少量诊断日志或 trace，再做一个经评审的最小行为修复 | 复现、根因、修改、回归、回滚说明 |

首周每日安排：D1 核对环境并取正常 dump；D2 对照 DisplayInfo/DisplayDeviceInfo 与设备对象；D3 追启动时显示注册；D4 追一次亮灭屏；D5 对比 WMS 与 SF 标识；D6 跑显示监听实验；D7 不看源码讲解链路并补齐证据缺口。

每日建议节奏：20 分钟读概念，30 分钟追一个分支，30 分钟实验，10 分钟记录；遇到真实缺陷可围绕缺陷调整顺序。

## 5. 推荐练习项目：DisplayLab

建议建设小型调试 App，逐步增加功能，而非第一天完成全部：

1. 枚举 Display，输出 displayId、名称、state、mode、supportedModes，并记录 DisplayListener 事件与时间。
2. 输出窗口所在显示、当前尺寸与旋转；比较应用视角和系统 dump。注意应用可见显示受权限与 API 限制。
3. 辅助屏使用 Presentation 或受支持的启动方式显示不同内容，观察拔屏后的回收与回退。
4. SurfaceView 绘制简单移动图形，设备/API 支持时尝试 Surface.setFrameRate 请求；记录请求与实际模式，不假定系统一定满足。
5. 需要 VirtualDisplay 时明确输出 Surface、资源释放与权限；涉及屏幕内容捕获走系统授权流程。VirtualDisplay 不等于真实物理屏，不可代替物理亮度、HWC、电源或热插拔验证。

亮度先用系统设置操作并观察框架结果；App 窗口亮度覆盖与全局亮度策略不是同一个实验。

## 6. 最小取证步骤

以下为 Windows PowerShell 文本采集示例，尚未在用户设备执行。多设备时每条 adb 命令增加 -s SERIAL。先取正常基线，再在问题前、中、后取样，静态 dump 不能替代动态 trace。

```powershell
adb devices
adb shell getprop ro.build.fingerprint
adb shell getprop ro.build.type
adb shell getprop ro.build.version.sdk
adb shell dumpsys display | Out-File display.txt -Encoding utf8
adb shell dumpsys power | Out-File power.txt -Encoding utf8
adb shell dumpsys window | Out-File window.txt -Encoding utf8
adb shell dumpsys SurfaceFlinger | Out-File surfaceflinger.txt -Encoding utf8
adb shell dumpsys sensorservice | Out-File sensorservice.txt -Encoding utf8
adb logcat -b all -v threadtime -d | Out-File logcat.txt -Encoding utf8
adb shell cmd display help
adb shell dumpsys SurfaceFlinger --help
adb shell perfetto --help
```

不同案例分别建目录防止覆盖。复现期间另开终端持续采集 logcat，结束后 Ctrl+C；不要只靠事后 -d 获取瞬态事件。动态 trace 优先按目标版本官方 Winscope/Perfetto 配置采集，确认数据源实际存在；采集权限、ProtoLog 开关和 root 能力依构建而异。PowerShell 二进制重定向存在版本差异，proto/trace 优先在设备端存文件再 adb pull。

工具分工：dumpsys display 看设备、逻辑显示和策略；dumpsys power 看上游电源请求；Winscope 看 WM/SF 层级与状态变化；Perfetto 看时序、调度与帧完成情况。FrameTimeline 依版本和数据源可用性使用，不能仅凭其归因标签判定最终根因。

## 7. 常见问题如何下手

| 现象 | 优先证据与检查 | 不应直接推断 |
|---|---|---|
| 唤醒后黑屏 | Power 请求、DPC 阻塞与状态、WMS/SF 可见性、首个 buffer、present；结合硬件证据 | DMS 状态 ON 就代表面板已出图 |
| 亮度异常 | 手动/自动模式、环境光、窗口覆盖、节电/热/HBM 限制、目标与当前值、厂商路径 | 设置值相同就应有相同亮度 |
| 刷新率不切换 | 支持模式、App 请求、策略约束与优先级、SF 目标和实际模式、厂商限制 | App 请求 120 就一定运行 120Hz |
| 外屏识别但无内容 | 设备与逻辑映射、任务/窗口所在屏、显示权限、SF 图层与输出 | 能枚举屏就代表任务能正确展示 |
| 卡顿或闪屏 | 找异常第一帧，对照正常帧；检查事务、buffer、线程调度和 present | 所有卡顿都来自 SF 或驱动 |

调试闭环：固定复现与时间点 → 提出至少两个可区分假设 → 同步抓证据 → 找最早偏离正常的状态 → 源码确认写入条件 → 最小修改 → 回归。

日志建议记录 displayId/uniqueId（按实际可用字段）、旧值→新值、原因、调用方、线程与完成回调；刷新率补充模式/约束；电源补充请求与 blocker。不要在每帧热路径无节制打日志，避免扰动结果。

每个案例记录：构建版本、设备、前置条件、复现步骤/概率、期望/实际、时间标记、正常对照、证据、假设及排除依据、根因代码、修复和回归。回归按改动涉及的亮灭屏、解锁、旋转、多屏、视频、自动亮度、节电/热条件选取。

## 8. 资料入口与待确认事项

- [AOSP 图形架构](https://source.android.com/docs/core/graphics/architecture)：建立像素链与合成概念。
- [AOSP display 服务源码目录](https://android.googlesource.com/platform/frameworks/base/+/main/services/core/java/com/android/server/display/)：先切换到项目对应 tag/分支，再定位相关类。
- [DisplayManager API](https://developer.android.com/reference/android/hardware/display/DisplayManager)：显示枚举和回调实验。
- [HWC 概述](https://source.android.com/docs/core/graphics/hwc)：理解框架与硬件合成边界；具体 HAL 形式以项目版本为准。
- [Winscope 分版本 ADB 采集](https://source.android.com/docs/core/graphics/winscope/capture/adb)。
- [Perfetto FrameTimeline](https://perfetto.dev/docs/data-sources/frametimeline)。

待确认：AOSP 16 的具体分支与厂商改动、是否有对应源码与 userdebug 设备、当前最典型缺陷。刷新率与合成作为支撑知识；当前主线为亮度、亮灭屏和多屏。

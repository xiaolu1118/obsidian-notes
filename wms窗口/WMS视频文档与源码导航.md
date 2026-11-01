# WMS 视频、文档与源码导航

创建日期：2026-11-01。更新日期：2026-11-01。
返回：[[wms窗口/索引|WMS 索引]]；配套：[[WMS学习路线与56天每日计划]]、[[WMS定位工具与排障实战手册]]。

## 使用原则与资料核验范围

以下链接于 2026-11-01 检索核对。视频章节、时长来自公开页面，未完整观看；不能据此保证教学质量、完整源码或全部免费。部分视频是商业课程试看，明确标注，先看公开内容，不建议先付费。技术结论以官方文档及实际产品分支为准。没有找到一套可核实且完整覆盖现代 WMS、Insets、Shell Transition、Winscope 的免费中文视频系列，因此采用组合学习。

每日计划里的 V、D、S 编号均可在本笔记找到。视频只是建立直觉，每天必须有源码和证据产出。

## 视频：按需观看，避免重学整套 Framework

### V1：Android 12 WMS 核心原理解析，入门主视频

[公开合集](https://www.bilibili.com/video/BV1rp4y1V7yB/)，共 5 集，约 115 分钟，页面标注 Android 12。

| 分集直达 | 页面章节与时长 | 对应学习日 | 看完应回答 |
|---|---|---|---|
| [P1](https://www.bilibili.com/video/BV1rp4y1V7yB/?p=1) | View 绘制流程，22:41 | Day 3 | traversal 与窗口请求如何相连？ |
| [P2](https://www.bilibili.com/video/BV1rp4y1V7yB/?p=2) | WMS 如何去显示和管理 View，23:04 | Day 4 | 客户端 View 与服务端窗口对象如何对应？ |
| [P3](https://www.bilibili.com/video/BV1rp4y1V7yB/?p=3) | Activity、View、Window 分工，23:42 | Day 2 | Window 是不是 Surface？ |
| [P4](https://www.bilibili.com/video/BV1rp4y1V7yB/?p=4) | AMS 与 WMS 的关系及进程间交互，23:00 | Day 5 | ATMS、WMS 各管理什么？ |
| [P5](https://www.bilibili.com/video/BV1rp4y1V7yB/?p=5) | Surface、Canvas、SurfaceFlinger 流程，22:26 | Day 6 | 窗口存在为何仍可能没有画面？ |

适合快速补齐概念；现代动画不能只依赖此合集。

### V2：Android 车载必备 Framework 学习之 WMS，补充视频

[公开合集](https://www.bilibili.com/video/BV1RL411w7Ky/)，8 集，2022 年发布；实际源码基线未核实。只选薄弱部分，不必与 V1 全部重复观看。

| 分集 | 公开章节与时长 | 用途 |
|---|---|---|
| [P3](https://www.bilibili.com/video/BV1RL411w7Ky/?p=3) | WMS 中 ViewTree 体系结构，18:55 | Day 2 概念补充 |
| [P4](https://www.bilibili.com/video/BV1RL411w7Ky/?p=4) | Activity 如何通过 window 管理 View，17:39 | Day 3 |
| [P5](https://www.bilibili.com/video/BV1RL411w7Ky/?p=5) | Activity 启动过程中的 window 显示管理，27:30 | Day 5 |
| [P6](https://www.bilibili.com/video/BV1RL411w7Ky/?p=6) | ViewRootImpl 与 WM 的联系，23:43 | Day 8 |
| [P7](https://www.bilibili.com/video/BV1RL411w7Ky/?p=7) | onResume 获取 UI 属性问题，29:17 | Day 9 |

注意：客户端 View 树与 WMS WindowContainer 树不是同一棵树；后者仍需自己读源码。

### V3：Android 14/15 ShellTransition 项目专题

[千里马学框架公开试看](https://www.bilibili.com/video/BV1ycrMYzESe/)，2025 年发布，介绍标注 AOSP 15。用于 Day 29 建立现代转场直觉。**是新课试看，不能把链接当作完整免费课程**；公开页面不足以确认完整课程目录和实验覆盖，不安排虚构分集。学习重点自行对照源码：collect、ready、同步事务、play、merge、finish；没有在公开视频讲到的内容，按 S3 与 D5 补齐。

### V4：SurfaceFlinger 与 WMS 补充

[车载开发系列公开合集](https://www.bilibili.com/video/BV1314y1f7aF/)，2023 年发布，带商业课程推广。选 P1 SurfaceFlinger 启动（30:08）、P3 SurfaceFlinger 与 WMS 的关系（30:06）、P4 WMS 底层渲染逻辑与服务关系（33:56），对应 Day 22～24；CAN、CarService 等非本路线必修。

### 选课判断

优先能展示“复现 → trace → 源码 → patch → 回归”的课。若付费，先核对目标 Android 分支、Insets/IME、Shell Transition、TaskOrganizer、BLAST、实际 trace 文件和课后反馈是否齐全。播放量、标题中“全网最全”和薪资承诺不作为质量证据。Winscope/Perfetto 本路线直接用官方实操文档，不用不相关的仓库管理系统 WMS 视频填充。

## 官方文档与实操教程

| 编号 | 文档链接 | 阅读重点与时间 |
|---|---|---|
| D1 | [SurfaceFlinger 与 WindowManager](https://source.android.com/docs/core/graphics/surfaceflinger-windowmanager) | Day 2、6；职责边界与图形栈 |
| D2 | [Winscope 总览及官方排障案例](https://source.android.com/docs/core/graphics/winscope/overview) | Day 1、27、48；逐帧分析黑闪和转场，不只阅读界面说明 |
| D3 | [本地构建 Winscope](https://source.android.com/docs/core/graphics/winscope/run) / [Winscope 抓取](https://source.android.com/docs/core/graphics/winscope/capture/winscope) | Day 1、7；实际打开自己录制的文件 |
| D4 | [ADB 抓取：含 Android 14 及以下章节](https://source.android.com/docs/core/graphics/winscope/capture/adb) | Day 7、26；Android 15+ Perfetto 数据源与旧命令分开 |
| D5 | [分析 Shell transitions](https://source.android.com/docs/core/graphics/winscope/analyze/shell) | Day 29～35；transition ID、handler、targets、事务关联 |
| D6 | [Perfetto 入门](https://perfetto.dev/docs/getting-started/start-using-perfetto) / [Android 抓取](https://perfetto.dev/docs/quickstart/android-tracing) | Day 7、24、43；调度、线程、Binder、帧轨迹 |
| D7 | [FrameTimeline](https://perfetto.dev/docs/data-sources/frametimeline) | Day 24、43；Android 12+，关联 App 与 SF 帧；注意数据源自身覆盖范围 |
| D8 | [Views Insets](https://developer.android.com/develop/ui/views/layout/insets) / [Edge-to-edge](https://developer.android.com/develop/ui/views/layout/edge-to-edge) | Day 15～16；窗口内容、系统栏、target SDK 的关系 |
| D9 | [键盘显示及动画](https://developer.android.com/develop/ui/views/layout/sw-keyboard) | Day 17～19；只使用目标 API 已存在的接口，文档可能包含比产品分支更高的 API |
| D10 | [多窗口](https://source.android.com/docs/core/display/multi-window) | Day 36～39；分屏、自由窗口、PiP 与产品配置 |
| D11 | [多显示入口](https://source.android.com/docs/core/display/multi_display) | Day 40～42；继续进入 IME、Input routing、launch policy 子页面 |
| D12 | [PerfettoSQL 标准库](https://perfetto.dev/docs/analysis/stdlib-docs) | Day 44；以当前 trace 可用表为准 |

官方文档可切换中文，但源码符号建议保留英文。Insets 的应用文档用于制造可控实验，服务端原理要配合 S1、S2 阅读。

## 源码导航与固定版本策略

建议以**工作设备对应分支**作为主线；没有产品环境时选 AOSP Android 16 userdebug。视频的 Android 12 仅作历史对照。下面 release 分支链接便于导航，但它们会移动；Day 1 必须额外记录精确 commit 或固定 tag。不要混用 Android 12 的方法参数与 Android 16 的调用顺序。

- S1：[AOSP Android 16 服务端 wm 目录](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/android16-release/services/core/java/com/android/server/wm/)。已打开核对目录；按下面类名进入。
- S2：[Android Code Search](https://cs.android.com/)。选择实际分支后搜索 `frameworks/base/core/java/android/view`、`android/window`、`com/android/internal/policy`；不要默认搜索主干后直接套到产品。
- S3：在同一源码树搜索 `frameworks/base/libs/WindowManager/Shell`。在线直达目录本次未成功读取，使用 S2 或本地路径定位，不宣称已验证其网页可访问性。
- S4：`frameworks/native/services/surfaceflinger`、`frameworks/native/libs/gui`、`frameworks/native/services/inputflinger`，用 S2/本地同分支阅读。
- S5：测试搜索 `frameworks/base/services/tests/wmtests`、`frameworks/base/libs/WindowManager/Shell/tests`、`cts/tests/framework/base/windowmanager`；Flicker 代码位置随分支变化，在源码中搜索 `Flicker` 后按该目录 README 执行。

### 阅读顺序及要回答的问题

| 阶段 | 类/符号入口（实际签名以分支为准） | 追踪目标 |
|---|---|---|
| 客户端 | ActivityThread、PhoneWindow、WindowManagerImpl、WindowManagerGlobal、ViewRootImpl | 谁创建 Decor，何时 setView，如何建立 IWindowSession |
| 添加窗口 | Session、WindowManagerService.addWindow、WindowToken、WindowState | token、权限、type、display 如何校验 |
| 容器树 | WindowContainer、RootWindowContainer、DisplayContent、DisplayArea、Task、TaskFragment、ActivityRecord | 继承关系与运行时父子关系分别画图 |
| 重布局 | ViewRootImpl.performTraversals/relayoutWindow、WMS.relayoutWindow、WindowSurfacePlacer、RootWindowContainer | traversal、surface placement 与绘制回调如何衔接 |
| 状态和策略 | WindowState、WindowStateAnimator、DisplayPolicy、PhoneWindowManager | requested visible、policy visible、draw state、实际 layer 状态为何不同 |
| 焦点输入 | DisplayContent、InputMonitor、InputWindowHandle、InputDispatcher | focused app/window、触摸区域、display 及变换如何下发 |
| Insets/IME | InsetsStateController、InsetsSourceProvider、InsetsPolicy、InsetsController、InsetsSourceConsumer、ImeInsetsSourceProvider | 状态产生、分发、控制权和动画 |
| Surface | SurfaceControl.Transaction、WindowStateAnimator、BLASTBufferQueue、BLASTSyncEngine | Buffer、layer、transaction、同步组的边界 |
| 现代转场 | TransitionController、Transition、BLASTSyncEngine；Shell Transitions、DefaultTransitionHandler | collect → ready → 同步完成 → Shell 播放 → finish；异常 merge/abort |
| 多窗口 | WindowOrganizerController、TaskOrganizerController、WindowContainerTransaction、ShellTaskOrganizer、StageCoordinator | organizer 和 system_server 的职责；WCT 不等于 Surface transaction |
| 旋转/启动 | DisplayRotation、AsyncRotationController、StartingSurfaceController、TaskSnapshotController、Shell startingwindow | configuration、无缝旋转、快照和真实首帧交接 |

这些是阅读入口而非保证所有分支都相同的精确调用栈。每次画调用图都标注进程、线程、Binder 边界、锁和异步消息；每个节点只保留与当前问题有关的方法。

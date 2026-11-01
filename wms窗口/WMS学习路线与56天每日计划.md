# WMS 学习路线与 56 天每日计划

创建日期：2026-11-01。更新日期：2026-11-01。
返回：[[wms窗口/索引|WMS 索引]]；资料编号见 [[WMS视频文档与源码导航]]；工具见 [[WMS定位工具与排障实战手册]]。

## 目标、假设与最快路径

学习顺序修正（2026-11-01）：用户明确要求先建立窗口概念和核心术语。先阅读 [[WMS入门第一课：窗口概念与核心术语]]，完成五项概念验收，再进入 Day 1。建议分两次各 45～60 分钟完成预习，尚未执行；原 56 天是后续路线，不要求零概念时立即抓 trace 或追源码。

用户已熟悉 PMS、Input 等 Framework 服务，因此不重学 Binder、Handler、SystemServer 全套。优先把已有 Input 知识连接到窗口焦点、触摸区域、IME 和 Surface。

建议投入：每天 3 小时，56 天约 168 小时。每天 30 分钟视频/文档、70 分钟源码、60 分钟实验、20 分钟记录。周复盘日用实验和口述替代新课。时间只有 90 分钟时，将一日拆成两日；不取消实验。Day 是学习日，不是已预订的自然日期。

建议阶段目标：

- 7 天：可独立抓取 dump/trace，讲清窗口主链路。
- 21 天：可分析层级、焦点、布局、Insets 和 IME 常见问题。
- 35 天：可以把 WMS 状态、SF 图层、Shell 转场和帧时序关联起来。
- 56 天：完成至少 5 个证据充分的案例和 1 个有回归的修复或行为变更，形成独立负责 WMS 子模块的基础。
- 专家能力仍需真实复杂问题、长期回归、架构取舍与跨版本经验；建议继续投入 3～6 个月，不能承诺 56 天成为专家。

若只有两周紧急上岗：先执行 Day 1～7，再选 Day 10、12、17、24、26、29、48；这是排障入门压缩版，不替代完整学习。

## 每日通用完成条件

概念预习补充（2026-11-01）：先完成 [[WMS入门第二课：窗口容器与三棵树]] 的验收，覆盖 WindowContainer、DisplayArea、Task/TaskFragment、ActivityRecord/WindowToken、WindowState，区分继承关系与运行时父子关系；此部分不推迟到 Day 8 才首次学习，Day 8 改为结合真实 dump 的深化。

每天保存：①一个具体问题及自己的解释；②带分支/commit 的源码入口与关键字段；③至少一份 dump/trace/对照截图；④实验结果和无法解释之处。能无稿讲清该日验收问题才算完成。遇到分支差异先记差异，不强行把代码匹配到视频。

## 第 1 周：把窗口创建、显示和工具连起来

| 日   | 必须掌握                                                       | 资料与源码                                                 | 实验及验收产出                                                                                          |
| --- | ---------------------------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| 1   | 学习基线；WMS/ATMS/Shell/SF 分工；静态 dump 与动态 trace                | D2、D3；S1                                              | 记录 build、API、权限、源码 commit；采集启动前后 window/activity/SF dump；打开 Winscope。验收：用证据指出当前前台窗口、Task、display |
| 2   | Activity、Window、ViewRootImpl、WindowState、Surface/Layer 的区别 | V1 P3、V2 P3；D1；S1/S2                                  | 画两张图：进程协作图、运行时窗口树；不能把 View 树当容器树。验收：解释 Dialog 为什么可增加窗口                                           |
| 3   | DecorView 到 ViewRootImpl.setView、IWindowSession            | V1 P1、V2 P4；S2                                        | Demo 打印生命周期、onAttachedToWindow、布局/绘制回调；标注 addView 发生阶段。验收：生命周期回调不等于首帧显示                          |
| 4   | Session、addWindow、token/type/权限/display 校验                 | V1 P2；S1 addWindow                                    | 对比 Activity、Dialog、PopupWindow 的窗口 type/token；在 Demo 可控条件下复现无效 token，记录拒绝点而非只记异常                 |
| 5   | ATMS 启动与窗口添加、可见请求的关系                                       | V1 P4、V2 P5；ActivityRecord、ActivityTaskManagerService | 启动两个 Activity，画客户端与服务端泳道图，标注 Binder 和异步边界。验收：解释 resumed、visible、focused 不等价                      |
| 6   | relayout、surface 创建、首次绘制与 finishDrawing 相关回调               | V1 P5、D1；ViewRootImpl、WMS、WindowStateAnimator         | 对比启动前后 draw state、surface 状态和录像；画首帧时序。验收：有 Surface 为什么仍可能未呈现                                     |
| 7   | 独立采集和复盘                                                    | D3、D4、D6；工具手册                                         | 完整抓一次启动/退出，打开 Winscope 与 Perfetto；口述主链路 10 分钟。提交第一份“正常启动基线”案例；环境未通则先解决工具，不跳过                     |

## 第 2 周：层级、布局、可见性和 Input

| 日   | 必须掌握                                       | 资料与源码                                                         | 实验及验收产出                                                                          |
| --- | ------------------------------------------ | ------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| 8   | WindowContainer 树；继承与包含的区别                 | V2 P6；WindowContainer、DisplayContent、DisplayArea、TaskFragment | 在 dump/Winscope 标出 Display→Task→ActivityRecord→WindowState 的实际路径，记录中间节点；不背固定完整树  |
| 9   | traversal、layout、surface placement 和重复布局原因 | V2 P7；ViewRootImpl、WindowSurfacePlacer                        | 修改 Demo 尺寸触发 requestLayout，日志与 trace 关联。验收：区分客户端布局和服务端 placement                 |
| 10  | Z-order、窗口类型、父子 Surface、relative layer     | S1 WindowLayersController（若分支存在）、DisplayContent；S4            | Activity+Dialog+PopupWindow，对照窗口树与 SF 树；验收：说明某层为何在另一层之上，避免只比较一个 Z 数字             |
| 11  | flags、透明/半透明、dim、裁剪与触摸穿透边界                 | S2 LayoutParams；S1 WindowState/DisplayPolicy                  | 控制 FLAG_NOT_FOCUSABLE、FLAG_NOT_TOUCHABLE、dim、alpha，每次只改一个变量。验收：输出“显示/焦点/触摸”三列对照表 |
| 12  | focused app、focused window、按键与触摸目标         | S1 InputMonitor、DisplayContent；S4 InputDispatcher             | Dialog 弹出前后同步抓 window/input；按键和触摸分别验证。验收：能指出目标窗口与焦点更新链路                          |
| 13  | touchable region、遮挡、坐标和 display 路由         | S1 InputMonitor；S4 InputWindowHandle/InputDispatcher          | 制造可控点击区域偏差，核对 frame、transform、touch region。验收：分辨输入未命中、App 未处理、画面未更新              |
| 14  | 周复盘：不可点击问题闭环                               | 第 2 周全部证据                                                     | 不看笔记定位一个 flags 或触摸区域错误；给出根因、排除项、修复前后证据。验收：不能只凭 mCurrentFocus 下结论                 |

## 第 3 周：Insets、系统栏、IME 与旋转

| 日   | 必须掌握                                                 | 资料与源码                                                                    | 实验及验收产出                                                                    |
| --- | ---------------------------------------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| 15  | InsetsSource/State、窗口 frame 与内容区域                    | D8；InsetsStateController、InsetsSourceProvider                            | Demo 记录不同 Insets type 的值；三键/手势分别采样。验收：计算一个控件为何被系统栏覆盖                       |
| 16  | Insets 分发、控制权、系统栏策略与 edge-to-edge                    | D8；InsetsPolicy、InsetsController、DisplayPolicy                           | 修改 target SDK/适配策略做受控对比，记录系统版本。验收：区分应用消费 Insets 错误与服务端源区域错误                |
| 17  | IME show/hide 链路与焦点前提                                | D9；InsetsController、ImeInsetsSourceProvider；结合 InputMethodManagerService | 点击 EditText、切 Dialog，记录 IME 请求到可见的阶段。验收：IME 不出时知道先查请求、焦点、目标还是 layer        |
| 18  | IME 的 input/layering/control target 等角色              | S1 DisplayContent、Insets 相关类                                             | App→Dialog→App 场景标出实际目标字段，按分支命名记录。验收：解释三个角色为何可能不同                          |
| 19  | adjustResize、adjustPan、Insets 动画与控制 leash            | D9；InsetsSourceConsumer、动画控制相关类                                          | 同一输入框分别测试 resize/pan 和回调处理；录制键盘运动与 App 位置。验收：定位跳动发生在哪一侧                    |
| 20  | Configuration、orientation、rotation、seamless rotation | S1 DisplayRotation、AsyncRotationController                               | 横竖屏、IME 可见时旋转各抓一次；标注配置变化、窗口尺寸、层变换。验收：区分 orientation 请求与实际 display rotation |
| 21  | 周复盘：键盘遮挡与旋转错位                                        | 第 3 周证据                                                                  | 完成一个 IME 案例和一个旋转对照记录；用截图、frame、Insets、transform 四类证据解释，不靠“加延时”结案           |

## 第 4 周：Surface、同步、图形与 trace

| 日   | 必须掌握                                                     | 资料与源码                               | 实验及验收产出                                                             |
| --- | -------------------------------------------------------- | ----------------------------------- | ------------------------------------------------------------------- |
| 22  | Surface、SurfaceControl、BufferQueue 与 Layer               | V4 P1/P3；S2/S4                      | 画生产者→队列→SF→显示路径，标清 WMS 控制几何和 App 生产 buffer 的边界                      |
| 23  | Transaction 的 position、crop、alpha、matrix、reparent        | V4 P4；SurfaceControl.Transaction；S4 | 在 Winscope 对照本层与父层属性；验收：给出一个“本层 alpha 正常但不可见”的祖先层解释                 |
| 24  | Perfetto Expected/Actual FrameTimeline、主线程和 RenderThread | D6、D7                               | Demo debug 开关阻塞主线程，再抓正常/异常各一次；判断 App deadline 与 SF 呈现阶段，不把红帧一律归 WMS |
| 25  | BLASTBufferQueue、BLASTSyncEngine 与同步完成                   | S1/S2/S4                            | 按分支跟踪 resize 的事务与 buffer 同步；输出时序图，解释同步目标和 timeout 后果，禁止假设每次都会走同一支路  |
| 26  | Winscope 多数据源采集、录屏与时间关联                                  | D3、D4；工具手册                          | 采集 WM、SF、转场及录屏（设备支持哪些记录哪些）；复现 5～10 秒；验收：定位异常前/中/后三帧并记录缺失轨迹          |
| 27  | 可见性、继承属性、diff 与真正显示结果                                    | D2 官方案例                             | 先独立分析官方案例再对照说明；对自己的 Dialog 检查祖先 crop/transform/alpha。验收：给出可证伪的因果解释  |
| 28  | 周复盘：黑屏、白屏、闪烁分类                                           | 第 4 周证据                             | 对比无首 buffer、被遮挡、alpha/crop 错误三类现象；至少实测两类，未实测项标待验证；提交案例 3            |

## 第 5 周：Shell Transition 与启动窗口

| 日   | 必须掌握                                            | 资料与源码                                                                 | 实验及验收产出                                                              |
| --- | ----------------------------------------------- | --------------------------------------------------------------------- | -------------------------------------------------------------------- |
| 29  | legacy AppTransition 与现代 Shell Transition 的边界   | V3 公开试看；D5；TransitionController、Shell Transitions                     | 核实产品启用路径；画 system_server 与 Shell 分工图。验收：能说明动画为何可能在 SystemUI/Shell 进程 |
| 30  | collect、participants、ready 和同步组                 | S1 Transition、TransitionController、BLASTSyncEngine                    | 跟踪一次 App open，标出 transition ID 和参与者。验收：哪个条件阻止继续推进                    |
| 31  | TransitionInfo、Change、leash 和 handler 选择        | D5；S3 Transitions、DefaultTransitionHandler                            | 将一个 target 的 window ID、layer ID、leash 关联；验收：说明谁持有动画执行权               |
| 32  | start/finish transaction、清理和状态恢复                | S1/S3 转场类                                                             | 逐帧比较动画首帧、末帧和 finish 后父节点/变换；验收：解释 finish 漏执行的潜在表现                    |
| 33  | merge、abort、并发转场与超时                             | D5；S3 Transitions                                                     | 快速启动/返回/切换，找到实际 merge 或 abort；没出现则记录正常结果并查对应测试，不能编造事件                |
| 34  | Starting Window、SplashScreen、TaskSnapshot 和首帧交接 | S1 StartingSurfaceController、TaskSnapshotController；S3 startingwindow | 冷/热启动、返回最近任务分别采集，区分 splash、snapshot、真实窗口。验收：指出黑闪位于哪个交接点              |
| 35  | 周复盘：完整转场时间线                                     | 第 5 周证据                                                               | 输出 create→collect→ready→play→finish 时间线，写明 ID、事务和 SF 属性变化；完成案例 4     |

## 第 6 周：Organizer、多窗口、多显示和返回

| 日 | 必须掌握 | 资料与源码 | 实验及验收产出 |
|---|---|---|---|
| 36 | WCT、TaskOrganizer 与 ShellTaskOrganizer | D10；WindowOrganizerController、TaskOrganizerController、S3 | 画 WCT 请求到任务层级变化流程；验收：WCT 与 SurfaceControl.Transaction 分别改什么 |
| 37 | 分屏、Stage、Task reparent 和 bounds | D10；S3 StageCoordinator 等实际实现 | 分屏进入/退出，对照 Task 树、bounds、窗口模式；验收：定位一个尺寸异常应查哪三层 |
| 38 | resize、divider 和同步事务 | S3 分屏布局及同步相关类 | 拖动分隔条，关联配置、App relayout、buffer 和 leash；验收：卡住发生在哪个阶段 |
| 39 | PiP、自由窗口、桌面窗口的差异和配置门槛 | D10；S3 pip/freeform/desktopmode 实际目录 | 设备支持则运行 PiP/自由窗口；不支持则读测试并记录限制，不强开产品全局配置。输出模式对照表 |
| 40 | DisplayContent、DisplayArea、多 display 的任务与焦点 | D11；S1 RootWindowContainer/DisplayContent | 使用受支持的模拟第二显示或测试设备，记录 display ID 与任务归属；验收：不是看到两个画面就认定有两套独立任务 |
| 41 | 多显示 Input 路由和 IME 策略 | D11 的 Input/IME 子文档；结合熟悉的 Input | 比较两屏窗口焦点、按键、触摸和 IME 目标；验收：区分显示问题和 input display 路由问题 |
| 42 | 返回手势/预测返回；本周复盘 | S1 BackNavigationController；S3 back 相关实现 | 分别完成和取消返回，按启用状态记录目标、leash、清理；完成一个多窗口案例，功能未启用则只记录源码与测试证据 |

## 第 7 周：性能、日志、测试和真实缺陷定位

| 日 | 必须掌握 | 资料与源码 | 实验及验收产出 |
|---|---|---|---|
| 43 | 关键路径、CPU 调度、Binder 等待、锁竞争 | D6/D7；实际 trace | 对一次慢转场分段计时：请求→ready、ready→play、play→finish；验收：有持续时间与线程状态支持瓶颈判断 |
| 44 | Perfetto SQL 的 slice、线程、帧查询 | D12；工具手册 SQL | 查询相关 slice 和帧，导出筛选结果；验收：解释单位、过滤条件和缺失数据，不能把未采到当没发生 |
| 45 | ProtoLog 分组、普通日志和额外 trace 埋点 | D4；源码搜索 ProtoLogGroup | 只启用目标组或加局部 debug 埋点；验证日志关联 ID，记录开销并恢复配置；验收：一条日志如何支持或推翻假设 |
| 46 | wmtests、Shell tests、CTS 的不同覆盖范围 | S5 | 查最近相关测试，执行目标模块允许的最小范围；若无编译环境，明确只完成阅读，补执行待办，不写测试已通过 |
| 47 | Flicker/转场断言与重复回归 | D2；S5 搜 Flicker | 找到匹配分支 README，梳理场景、轨迹与断言；可运行环境执行一个已有场景，否则手动重复并保留差异 |
| 48 | 陌生缺陷盲测定位 | D2 官方案例；复盘模板 | 隐藏结论后分析一次官方案例或同事脱敏案例；列至少两个竞争假设，用证据排除；提交案例 5 |
| 49 | 周复盘与知识漏洞检查 | 第 1～7 周全部 | 30 分钟白板讲架构，60 分钟定位未知现象；对错误分类回填源代码和实验。验收：说不清的链路列入补课清单 |

## 第 8 周：独立修复、回归和交付

| 日 | 必须掌握 | 资料与源码 | 实验及验收产出 |
|---|---|---|---|
| 50 | 选定一个可控、可复现、范围明确的问题 | 实验项目模板 | 选真实缺陷，或在独立实验分支注入局部错误；固定期望/实际与稳定步骤，禁止用随机延时制造不稳定题目 |
| 51 | 缩小首次错误状态和责任模块 | 工具矩阵与相关源码 | 用正常/异常两份轨迹定位首个分叉；给出明确字段、写入点和调用来源 |
| 52 | 根因证明与替代方案评估 | 对应源码与测试 | 写“为什么这个条件会产生此结果”；至少排除一个替代根因；列修复点、兼容性与代价 |
| 53 | 实现最小修复及有效回归 | S5 | 在实验分支完成 patch；验证故障用例失败→修复后通过，补覆盖行为的测试或可重复脚本。未有环境则本阶段尚未完成 |
| 54 | 跨场景与性能回归 | 实战手册回归清单 | 跑旋转、IME、返回、分屏、冷热启动等适用组合；比较时序和结果，记录通过数、失败数与不适用项 |
| 55 | 技术报告和代码评审 | 复盘模板 | 写可交给其他工程师复现的报告：根因、证据、patch、风险、回归；请同事审阅或自己进行反例审查 |
| 56 | 综合验收和后续专项选择 | 下方能力表 | 脱稿讲解完整案例并从空目录重新抓取一次；选后续主攻方向，保留未通过项，不为了“打卡完成”勾掉 |

## 结业能力量表

每项按 0～3 分自评：0 不会，1 能解释，2 能在引导下操作，3 可独立定位并以回归证明。建议核心项都达到 3，而不是追求总分掩盖短板。

| 能力 | 验收问题 |
|---|---|
| 窗口生命周期 | 从 App addView 到实际显示，跨进程和异步边界是什么？ |
| 状态与层级 | 窗口树、Surface 树、可见请求和屏幕结果如何对应？ |
| 布局与 Insets | 系统栏/IME 遮挡时如何区分 App 与系统错误？ |
| Input | 如何用输入目标、区域、坐标和 display 定位无法点击？ |
| 转场 | 如何找到卡在 ready、同步、handler 或 finish 的阶段？ |
| 多窗口 | 如何解释 Task 重挂、bounds、配置与 Surface 几何不一致？ |
| 性能 | 如何区分状态错误、App 帧慢、SF 慢与调度等待？ |
| 工程闭环 | 是否至少有 5 份案例、1 个 patch、明确回归结果？ |

## 之后 3～6 个月：从知识完整到专家能力

建议而非用户已决定：第 3 个月选 Insets/IME 或 Shell Transition 为主攻，深入负责真实问题；第 4 个月做多窗口/多显示和跨版本迁移；第 5～6 个月参与性能优化、架构评审和测试建设。每月积累 3～5 个有因果证据的真实案例，关注复现率、定位耗时、修复回归率和能否帮助同事独立排障。不要只以源码阅读页数或课程数量衡量进步。

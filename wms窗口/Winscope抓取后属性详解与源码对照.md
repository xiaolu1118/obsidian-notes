# Winscope 抓取后属性详解与源码对照

创建日期：2026-11-01。更新日期：2026-11-01。
返回：[[wms窗口/索引|WMS 索引]]；前置：[[WMS入门第一课：窗口概念与核心术语]]、[[WMS入门第二课：窗口容器与三棵树]]；实操：[[WMS定位工具与排障实战手册]]。

## 范围与核对状态

用户已澄清：需要讲解导入 trace 后的属性，如 bounds、flags、WindowContainer、Layer，不是抓取前配置。讲解要求形象、便于记忆，并给出源码位置。

本篇覆盖 WM/SF 常见核心字段及转场关联字段；不是对任意版本所有字段的穷尽枚举。尚未获得用户的 Android 版本、Winscope commit 和实际 trace，因此不能逐一核对其面板全部字段。界面可能使用 camelCase，proto 常用 snake_case；旧版本字段也可能弃用。本次核验基于官方文档及可访问的 AOSP main/特定 commit，不声称已在用户设备运行。

## 1. 先理解属性从哪里来

```text
运行时 Java/C++ 对象
    ↓ dumpDebug / writeSnapshotToProto 等序列化
proto 数据结构
    ↓ trace 文件、解析及属性处理
Winscope 原始 Proto Dump + 整理/计算后的属性面板
```

- 原始字段：例如 `WindowState.mHasSurface` 写成 `has_surface`。
- 方法计算结果：例如 `WindowState.isVisible()` 写成 `is_visible`，不是一定存在同名成员。
- 整理/推导属性：例如遮挡原因、Requested/Calculated 分组，不能假设 WMS 有同名成员变量。
- optional 字段缺失、未采集、日志级别过滤，与明确写出 false/0 不总等价。先查原始 proto 与显示默认值设置。

记忆：WM 是“管理账本”，SF 是“画面拼装账本”；都不是摄像机逐像素判断用户到底看到了什么。

## 2. 源码地图：以下编号用于各表

路径以 AOSP 根目录为起点，属于远端源码路径，不表示当前笔记库已有这些文件。

| 编号 | 源码位置/符号 | 用途 |
|---|---|---|
| W0 | `frameworks/base/core/proto/android/server/windowmanagerservice.proto` | WindowContainer/Task/Activity/WindowState 等字段定义，搜索对应 `message` |
| W1 | `frameworks/base/services/core/java/com/android/server/wm/WindowContainer.java`，`dumpDebug`、`writeIdentifierToProto` | 公共容器字段、children 和标识输出 |
| W2 | 同目录 `ConfigurationContainer.java`，配置成员、`dumpDebug`、`onConfigurationChanged`、`resolveOverrideConfiguration` | 配置的请求、解析、继承和输出 |
| W3 | 同目录 `Task.java`、`TaskFragment.java`、`ActivityRecord.java`、`DisplayContent.java`，各自 `dumpDebug` | 各类节点专属字段；具体修改点再追 setter/调用链 |
| W4 | 同目录 `WindowState.java`，`dumpDebug`、`isVisible`、`isOnScreen`、`isReadyForDisplay` | 窗口状态及计算方法 |
| W5 | 同目录 `WindowFrames.java`，`mFrame/mParentFrame/mDisplayFrame/mCompatFrame`、`dumpDebug` | 窗口布局矩形 |
| W6 | `frameworks/base/core/java/android/view/WindowManager.java`，`LayoutParams` 及 `dumpDebug`；`core/proto/android/view/windowlayoutparams.proto` | type、flags、softInputMode 等属性及常量 |
| W7 | `frameworks/base/core/java/android/app/WindowConfiguration.java`；`core/java/android/content/res/Configuration.java` | 配置字段与枚举 |
| W8 | `frameworks/base/services/core/java/com/android/server/wm/WindowStateAnimator.java`、`WindowSurfaceController.java` | draw state、Surface 显示控制和输出 |
| W9 | `frameworks/base/core/java/android/view/InsetsState.java`、`InsetsSource.java`、`WindowInsets.java`；服务端 `InsetsSourceProvider.java`、`InsetsStateController.java`、`ImeInsetsSourceProvider.java` | Insets 定义、分发与控制 |
| S0 | `external/perfetto/protos/perfetto/trace/android/surfaceflinger_layers.proto` 及 `surfaceflinger_common.proto` | Layer/Display/Input 等 trace schema，部分版本有其他 proto 来源 |
| S1 | `frameworks/native/services/surfaceflinger/LayerProtoHelper.cpp`，`writeSnapshotToProto` | 请求状态/快照如何变成 LayerProto |
| S2 | 同目录 `FrontEnd/RequestedLayerState.*`、`LayerSnapshot.*`、`LayerSnapshotBuilder.cpp` | 请求、计算快照、父层影响和输入几何 |
| S3 | `frameworks/base/core/java/android/view/SurfaceControl.java` 的 `Transaction`；`frameworks/native/libs/gui/SurfaceComposerClient.cpp`、`include/gui/LayerState.h` | 图层属性设置接口及原生 flags |
| S4 | `frameworks/native/libs/gui/include/gui/WindowInfo.h`、`libs/gui/WindowInfo.cpp`；服务端 `InputMonitor.java`；`services/inputflinger/dispatcher/InputDispatcher.cpp` | 输入窗口属性生成、传递和使用，按产品分支确认路径 |
| V0 | `development/tools/winscope/src/viewers/viewer_window_manager/`、`viewer_surface_flinger/`、`parsers/`、`trace_processor/` | 界面展示与派生值入口；新旧版本目录/计算位置可能变化，需搜索实际 checkout |
| T0 | 服务端 `PerfettoTransitionTracer.java`；`frameworks/base/libs/WindowManager/Shell/src/com/android/wm/shell/transition/tracing/PerfettoTransitionTracer.java` | 转场事件、ID、事务和 targets 的输出 |

已直接核对 W4 的输出映射、W5 的矩形注释、W2 的配置成员、W0 的 schema、S0 的 schema、S1 的快照序列化和 S2 的实现文件。其余列为相关源码阅读入口，未逐一核对所有版本的签名；不提供未经核实的行号。序列化点告诉你值来自哪里，不能单独证明是谁把值改错了。

## 3. WindowContainer 公共属性：先回答“是谁、归哪、采用什么配置”

以下是 W0/W1/W2/W7 的概念解释，部分展示字段由 viewer 从层级或配置整理而来。

| 字段/组 | 形象解释与例子 | 源码入口 | 别误读成 |
|---|---|---|---|
| `windowContainer` | 每种容器共用的“基础档案页” | W0 `WindowContainerProto`；W1 | 运行时又新增了一个独立父节点 |
| `identifier.title` / name | 档案名称，帮助认出聊天 Activity 或 NavigationBar | W1/W4 `writeIdentifierToProto` | 跨 trace 永久唯一键 |
| `identifier.hashCode` | 进程内对象身份标识，帮助同一现场关联 | W1/W4 | Layer ID、taskId 或持久 ID |
| `identifier.userId` | Android 用户身份 | W1/W4 | UID 或 PID |
| `children` | 下级档案列表，反映管理组织 | W1 `dumpDebug` 及实际子类 | App 内部 Button 列表 |
| parent（树中） | 当前节点由谁组织；通常由树结构重建 | W1 parent/children；V0 | SF 一定具有相同父节点 |
| `visible` | 这个容器按其类型的逻辑被判定为可见 | W1 `isVisible()`，结合子类实现 | 像素一定在屏幕上可见 |
| `orientation` | 容器的方向请求/属性 | W1；ActivityInfo 方向常量 | 当前屏幕旋转角度 |
| `surfaceControl` | 关联的图层控制对象信息，可用于关联 SF | W0/W1/S3 | 必须有自己的内容 buffer |
| `surfaceAnimator` / leash | 运动控制架相关信息，动画时可能出现 | W1、`SurfaceAnimator.java` | 一个新的 Activity 或独立应用窗口 |

### 三种 configuration：家规、自定规则与最终生效规则

| 字段 | 记忆法 | 准确理解 |
|---|---|---|
| `overrideConfiguration` | “我自己提出的特殊要求” | 本节点请求覆盖的配置；W2 的 `mRequestedOverrideConfiguration`，具体写出策略看该分支 `dumpDebug` |
| `fullConfiguration` | “把父级规则和本地约束一起算完，最终用什么” | 本节点最终完整配置，父配置叠加解析后的覆盖值 |
| `mergedOverrideConfiguration` | “沿途各级特殊要求汇总” | 从上层到本节点合并的覆盖配置，起点不同于 full 的完整全局配置 |

**空的 override 不等于没有大小**：节点可以继承父级配置。不要拿请求覆盖值代替 full 中的最终解析值。

## 4. bounds 与配置：房间范围、应用配置与实际窗口不能混用

| 字段 | 含义与记忆 | 源码入口 |
|---|---|---|
| `windowConfiguration.bounds` | 容器的范围，像分配给任务的“房间”；从配置所在节点判断是谁的 bounds | W2/W7 |
| `appBounds` | 配置里的应用范围；受版本、Insets 与兼容策略影响，不保证等于最终 View 内容或可触摸范围 | W7；W2 兼容配置处理 |
| `maxBounds` | 配置中的最大范围参考，不能直接当当前窗口 frame | W7 |
| `windowingMode` | 当前采用全屏、PiP、自由窗口等哪种组织方式 | W7 常量 |
| `activityType` | 普通任务、Home、Recents 等用途类型 | W7 常量；W3 |
| 配置 `orientation` | 资源配置的横向/纵向分类 | Configuration.java |
| `rotation` | 显示/窗口配置的旋转枚举；常见 0/1/2/3 分别对应 0/90/180/270 度 | W7、`Surface.java`；W3 DisplayRotation |
| `densityDpi`、`screenWidthDp/HeightDp` | 密度及资源配置使用的 dp 尺寸；不能和 px 直接比较 | Configuration.java |

矩形例子：`[100,200][900,1400]` 表示 left=100、top=200、right=900、bottom=1400，宽 800、高 1200。右下不是宽高，普通布局范围通常以 px 表示；SF 浮点几何、dp 配置和不同坐标系另行区分。空 Rect 的含义须结合字段：可能继承/未指定，也可能实际没有面积，不能统一处理。

## 5. Task / ActivityRecord / Display 专属字段

结合 W3 类的 `dumpDebug` 查值来源，再到 W0 查定义。下表是字段家族，不保证每版原样出现。

| 字段 | 含义/例子 | 排查用途 |
|---|---|---|
| Task `id` / taskId | 某次任务的标识；同包名可有不同任务 | 判断是不是同一个任务，不能用 PID 替代 |
| rootTaskId / 旧 `stackId` | 根任务归属；W4 当前输出 `stack_id` 时写的是 rootTaskId | 不把旧命名机械解释成旧版 ActivityStack |
| `displayId` | 所属逻辑显示 | 跨屏窗口与输入路由核对 |
| Activity `name` / component | 包名与 Activity 组件 | 区分同应用不同页面 |
| `state` | Activity 生命周期状态，如 RESUMED/PAUSED | 和 visible 分开判断 |
| `visibleRequested` | 系统对该 Activity 的显示请求状态 | 请求显示≠已经有内容呈现 |
| `visible` / `clientVisible` | 服务端可见状态、通知客户端相关可见状态 | 按各字段写入实现区分，不能混成一个“显示开关” |
| `reportedDrawn` / `allDrawn` | Activity 相关窗口绘制完成的汇总/上报状态 | 看启动和转场等待；不代表当前每个像素都已被用户看到 |
| `startingWindow` / `startingDisplayed` | 启动占位窗口的标识与显示相关状态 | 区分快照/splash 与真实内容 |
| `fillsParent` / `translucent` | 是否填充父范围、是否半透明等策略性质 | 后方 Activity 为什么还参与显示 |
| `currentFocus` | 当前焦点窗口标识 | 查按键目标；不是点击命中结果 |
| `focusedApp` | 焦点应用/Activity 关联状态 | 和 focused window 可短暂不一致 |
| `inputMethodTarget` | 常见版本中的 IME layering target 相关输出 | 查看输入法层级跟随谁，具体语义查 DisplayContent 写出代码 |
| `inputMethodInputTarget` | 与输入目标关联的对象 | 不自动等于控制 Insets 的对象 |
| `inputMethodControlTarget` | 控制 IME Insets/动画关联的目标 | 结合 source provider 和 control/leash 看 |

## 6. WindowState：窗口自己的档案

主要对应 W4 的 `dumpDebug` 和各 getter；下列含义分组为通俗理解，精确判定条件按目标分支方法体读取。

### 6.1 布局与申请值

| 字段 | 记忆画面/含义 | 源码映射 |
|---|---|---|
| `attributes` / `attrs` | “我想要什么规格”，窗口申请属性组 | W4 `mAttrs.dumpDebug` → W6 |
| `requestedWidth/Height` | 客户端提交的请求尺寸；与 attrs 中 MATCH_PARENT 等模式值区分 | W4 `mRequestedWidth/mRequestedHeight` |
| `windowFrames.frame` | 最终布局得到的窗口矩形，像真正划定的柜台边界 | W5 `mFrame` |
| `windowFrames.parentFrame` | 应用 gravity、MATCH_PARENT 等时参考的矩形，不保证等于父 WindowState 的 frame | W5 `mParentFrame` |
| `windowFrames.displayFrame` | 当前布局要求窗口适配的范围，不必是整块物理屏幕 | W5 `mDisplayFrame` |
| `windowFrames.compatFrame` | 兼容缩放到应用坐标空间的 frame | W5 `mCompatFrame` |
| `surfacePosition` | Surface 相对相关父容器的定位信息，不能直接当屏幕绝对坐标 | W4 `mSurfacePosition`；追更新方法 |
| `surfaceInsets` | 为阴影等图形需要扩展的 Surface 区域信息 | W6 `LayoutParams.surfaceInsets` |
| `globalScale` / `hasCompatScale` | 窗口兼容缩放因素与是否启用 | W4 对应成员/方法 |
| `dimBounds` | 背景变暗效果使用的范围 | W4 dim controller 输出 |

记忆：**Task bounds 像房间，Window frame 像柜台，Surface/Layer 几何像摆上去的画面。三者可以不同。**

### 6.2 可见、绘制与退出

| 字段 | 通俗解释 | 不能推出什么 |
|---|---|---|
| `viewVisibility` | 客户端顶层 View 的可见性输入，常见 VISIBLE=0、INVISIBLE=4、GONE=8 | 不是 WMS 最终显示结论 |
| `hasSurface` | WMS 是否认为窗口具备 Surface | 不证明 buffer 已就绪/呈现 |
| `isVisible` | `WindowState.isVisible()` 的管理逻辑判断 | 不证明未被其他图层挡住 |
| `isOnScreen` | `isOnScreen()` 的状态判断，包含与可见/动画相关的条件 | 不是截图像素检测结果 |
| `isReadyForDisplay` | `isReadyForDisplay()` 的判定结果 | 不等于每种绘制/呈现条件都完成 |
| `animatingExit` | 窗口正在走退出动画相关状态 | 窗口不必立即从树消失 |
| `removeOnExit` | 退出后待移除相关标记 | 不是已经 removed |
| `destroying` / `removed` | 资源销毁或已移除的相关管理标记 | 和视觉消失时刻不必完全同步 |

源码映射：W4 `proto.write(...)` 直接给出每个字段的来源。不要仅靠字段名字判断，尤其 `isVisible`、`isOnScreen`、`isReadyForDisplay` 必须分别打开方法体。

### 6.3 drawState：把它当第一次开门营业的进度

| 状态 | 形象记忆 | 实际含义 |
|---|---|---|
| NO_SURFACE | 柜台还没搭好 | 尚无相应 Surface |
| DRAW_PENDING | 柜台搭好了，等内容 | 等待绘制相关完成信号 |
| COMMIT_DRAW_PENDING | 内容报好了，待提交后续状态 | 已收到绘制完成相关进展，仍有提交步骤 |
| READY_TO_SHOW | 等统一开门 | 已达到待显示阶段，仍可能等待相关窗口协调 |
| HAS_DRAWN | 已走到绘制/显示进展的后段 | WMS 的绘制状态已推进；不保证当前像素未被挡住，也不能代替 SF 呈现证据 |

入口 W8 `mDrawState`、`drawStateToString`、`finishDrawingLocked`、`commitFinishDrawingLocked`、`performShowLocked`，名称按分支核实。这个状态机用于理解绘制/显示协调，不能当作每一帧 GPU 完成状态。

## 7. flags：必须先看“它属于谁”

### WM 的 attributes.flags

对应 W6 `WindowManager.LayoutParams`，是可组合的 bitmask，像一排独立开关：

| 标志 | 含义与记忆 |
|---|---|
| FLAG_NOT_FOCUSABLE | 不取得按键输入焦点；不等于不接触摸 |
| FLAG_NOT_TOUCHABLE | 窗口不接收触摸；不保证底下必定收到，仍受安全与输入规则约束 |
| FLAG_NOT_TOUCH_MODAL | 允许窗口外的指针事件交给后方窗口；不是窗口内全穿透 |
| FLAG_DIM_BEHIND | 让后方变暗，结合 dimAmount；不是把自身 alpha 调低 |
| FLAG_SECURE | 保护窗口内容，限制截图/非安全显示等；不等于 HIDDEN |
| FLAG_ALT_FOCUSABLE_IM | 与 NOT_FOCUSABLE 配合改变输入法交互/层级关系，不能单独背成“显示键盘” |
| FLAG_KEEP_SCREEN_ON | 窗口可见时请求保持亮屏；不是永久全局亮屏 |

`privateFlags` 是另一组内部开关，不共享相同位语义；解析出原始数字时要用对应版本的定义。

### SF Layer 的 flags

对应 S3 原生图层标志，常见 HIDDEN、OPAQUE、SECURE，**不是 LayoutParams.flags**。InputWindowInfo 的 inputConfig 又是第三类标志。

学习建议：先看完整属性路径，再解码位。例如 `attributes.flags`、`Layer.flags`、`inputWindowInfo.inputConfig` 不可以互相套常量表。Winscope 的 IntDef 映射不匹配可能显示数字，数字本身不是故障。

## 8. Insets 属性：影响区域、请求和控制权

| 字段族 | 形象解释 | 源码入口 |
|---|---|---|
| Insets source `id` | 哪一个影响源的身份标识 | W9 InsetsSource |
| source `type` | 是状态栏、导航栏、IME 等哪种影响 | W9 WindowInsets.Type |
| source `frame` | 这个源在相应坐标空间的区域 | W9 InsetsSource；provider 更新 |
| source `visible` | 该 Insets 源的可见状态 | W9；不是 App 窗口 visible |
| `requestedVisibleTypes` | 窗口请求显示哪些 Insets 类型的位集合 | W4 `mRequestedVisibleTypes`、WindowInsets.Type |
| control target / control leash | 谁获得控制及其控制图层 | W9 InsetsSourceProvider、ImeInsetsSourceProvider |
| client/server visible | provider 的客户端与服务端可见状态因素 | W9；看计算组合，不凭一个字段断言最后结果 |

像“输入法占了下面一块地方，系统告知影响信息并协调控制”。Inset source 的 frame 不是说应用一定自动缩小这么多。

## 9. SurfaceFlinger Layer：看画面如何真正组合

### 9.1 身份与层级

| 字段 | 含义/记忆 | 源码 |
|---|---|---|
| `id` | 当前 trace 中 Layer 节点标识 | S0/S1 |
| `originalId` | 可用于映射原始层身份，镜像/克隆时与展示节点 ID 可能不同 | S1 `uniqueSequence/sequence` |
| `name` | 调试名称；同名/相似名称不能保证同一节点 | S0/S1 |
| `parent` / `children` | 实际图层父子组织，会影响几何/效果继承 | S1 层级生成；S2 |
| `z` | 层级排序值，通常要结合父节点及相对层级看 | S0/S1/S3 |
| `zOrderRelativeOf` / relative parent | “排序以谁为参照”，不等于换了几何父节点 | S0/S1/S2 |
| `relatives` / RelZ / RelZParent | 相对 Z 关系及 viewer 标签 | S0；V0 |
| `layerStack` | 选择输出到相关显示合成栈的标识 | S0/S2；不等同于 displayId 或 Task stackId |
| `ownerUid` | 图层的所有者 UID 信息 | S1；不等于 userId/PID |

排序类比：“站在甲前面”和“跟着甲一起移动”是两件事。relative parent 主要回答排序参照，几何 parent 决定组织/继承。隐藏传播等特殊规则仍按具体分支看。

### 9.2 几何、Requested 与 Calculated

| 字段/组 | 形象解释与使用 | 源码 |
|---|---|---|
| Requested | 我这一层自己提出的摆放与效果 | S1 RequestedLayerState 输出 |
| Calculated | 把父级影响与约束一起算完后的效果 | S1/S2；再看 V0 整理逻辑 |
| `requestedPosition` | 本层请求的位置 | S1 请求 transform 的平移部分 |
| `position` | trace 输出的位置结果，必须结合当前 schema/坐标系 | S0/S1 |
| `requestedTransform` | 本层请求的变换 | S1/S3 |
| `transform` / `effectiveTransform` | 变换相关结果；不同字段可能是历史兼容输出，不要默认同义 | S0/S1/S2 |
| `crop` | 本层请求的裁剪区域 | S1 requestedState.crop |
| `sourceBounds` | 输出的内容源范围，所核对实现来自 croppedBufferSize | S1 |
| `bounds` | 输出的图层几何范围，所核对实现来自 geomLayerBounds | S1/S2 |
| `screenBounds` | 变换后屏幕相关范围，所核对实现来自 transformedBounds | S1/S2 |
| `destinationFrame` | 用于定位/缩放 buffer 的目标矩形 | S0/S1/S3 |
| alpha / color alpha | 图层透明度，需区分请求与继承结果 | S1/S2/S3 |
| corner radius | 圆角半径，可受到父层影响 | S1/S2 |
| shadow / blur radius | 阴影、模糊效果参数 | S0/S1/S2 |

例：父层右移 100，本层在父坐标内右移 20，无其他变换时屏幕位置约为 120。自身请求没变，父层动画却能让结果变化。不要把每种 Calculated 属性都归因于 viewer 才计算；有些结果已在 SF 快照中产生，需查具体字段路径。

### 9.3 显示、遮挡与内容

| 字段/组 | 通俗理解 | 重要区别 |
|---|---|---|
| HIDDEN | 本层设置了隐藏相关标志 | 祖先状态仍可能进一步影响 |
| OPAQUE / `isOpaque` | 用于合成/遮挡判断的不透明性质 | 不是“窗口有无焦点”；需与 alpha、格式等一起看 |
| V / visible | viewer 的可见判定标签 | 不保证用户能看见下方像素 |
| invisibility reason | 工具解释的不可见原因，如无内容或祖先隐藏 | 派生诊断，不能只在 WMS 搜同名字段 |
| occluded | 被上方不透明层挡住 | 与自身 HIDDEN 不同 |
| partially occluded | 部分被不透明层挡住 | 还有部分可见 |
| covered | 被非不透明性质的层覆盖 | 不一定看不到，也不能保证看得到；内容像素本身可能不透明 |
| `visibleRegion` | 合成状态记录的可见区域 | 可能未采集；不是 Window frame |
| `transparentRegion` | 透明区域相关提示/状态 | 不是触摸穿透区域 |
| `damageRegion` | 内容更新损伤区域，像“这次哪里需要更新” | 不是被遮挡或损坏了的屏幕区域 |
| `activeBuffer` | 当前 buffer 元数据 | 容器/颜色等层无 buffer 不必是错误 |
| buffer width/height/stride | 像素尺寸与存储行跨度 | stride 不保证等于 width；不等于窗口逻辑尺寸 |
| `currFrame` / frame number | 内容帧序号相关标识 | 不是 FPS，也不等于全局 display vsyncId |
| pixel format / dataspace | 像素布局、颜色空间相关信息 | 不属于 WMS 窗口类别 type |
| composition type | CLIENT/DEVICE 等合成方式 | 是合成路径，不是 App 是否硬件加速的直接判定 |

S0 定义字段，S1/S2 负责部分状态输出/计算，遮挡原因等还需查 V0 或对应版本 trace processor。官方说明 Android 15 起旧 HWC/GPU chips 已弃用；某版本没显示不能直接视为没有合成。

### 9.4 InputWindowInfo：这张图层对应的输入规则

| 字段族 | 记忆问句 | 源码入口 |
|---|---|---|
| name/token/ownerPid/ownerUid | 事件关联哪个输入窗口与所有者？ | S4；S0 common proto |
| inputConfig | 可否接收输入、是否可聚焦、是否可信覆盖等规则？ | S4 枚举；不是 LayoutParams.flags |
| touchableRegion | 哪些点能命中它？ | InputMonitor/WindowState 生成，SF/Input 路由使用 |
| transform | 坐标如何转换成目标需要的坐标？ | S2/S4 |
| crop touch region with item | 是否用相应图层边界裁输入区域？ | S4；关联 crop 对象 |
| replace touch region with crop | 是否用 crop 关系替代触摸区域？ | S4 |

必须核对坐标系和变换方向，不能把 transform 看见就一律当“local→screen”。SF Input 区块未采集也不代表窗口没有输入能力。

## 10. Transition 与时间字段的关联

| 字段 | 意义 | 源码入口 |
|---|---|---|
| transition `id` | 同一次转场的关联标识 | T0 |
| `type` | open/close/change 等转场类别 | TransitionInfo、Shell Transitions 常量 |
| handler | Shell 选中的动画处理者 | Shell tracing 与 Transitions |
| create/send/dispatch/finish 时间 | 建立、交付、播放、结束等阶段事件，具体标签查对应输出 | T0 |
| targets `windowId` | 参与者的 WM 容器关联标识 | 服务端 tracer 输出；不是 layerId |
| targets `layerId` | 图层关联标识 | 服务端 tracer 从 SurfaceControl 获取 |
| `startTransactionId/endTransactionId` | 关联 SF 事务的线索 | T0、SF transaction trace |
| `mode/flags` | 参与者变化类型和附加属性 | TransitionInfo.Change 及 tracer |

同一时间线上不同轨迹未必每个采样点完全对齐。时间数值须看时钟域、单位、trace 对齐方式，不直接把原始 elapsed ns 当墙钟时间。

## 11. 从面板追到源码的可复用步骤

以 `hasSurface=false` 为例：

1. 确认选中的是目标 WindowState，时间点正确，原始路径是 `WindowStateProto.has_surface`。
2. 在 W0 确认字段类型与名称。
3. 在 W4 `dumpDebug` 找到 `proto.write(HAS_SURFACE, mHasSurface)`。
4. 搜索 `mHasSurface` 写入点，结合 W8 Surface 创建/销毁过程。
5. 对照问题前后状态，判定是尚未创建、正常退出销毁，还是异常。

本地 AOSP 根目录可使用：

```bash
rg -n 'HAS_SURFACE|mHasSurface' frameworks/base/services/core/java/com/android/server/wm
rg -n 'dumpDebug|IS_VISIBLE|WINDOW_FRAMES' frameworks/base/services/core/java/com/android/server/wm/WindowState.java
rg -n 'mFrame|mParentFrame|mDisplayFrame|dumpDebug' frameworks/base/services/core/java/com/android/server/wm/WindowFrames.java
rg -n 'writeSnapshotToProto|mutable_screen_bounds|mutable_bounds|set_flags' frameworks/native/services/surfaceflinger
rg -n 'occluded|invisibility|requestedTransform|screenBounds' development/tools/winscope/src
```

最后一条无结果时，查该版本的解析依赖和 trace processor 实现，不能据此认定字段不存在。

## 12. 先用两个例子记住

**位置错了**：先比较 Task bounds → Window frame → SF screenBounds；如果窗口 frame 正常而图层偏移，重点看父层/leash 的变换。哪个层先分叉，就沿那个层的来源追。

**窗口看不到**：先分开问请求显示、WMS 判定、绘制进度、SF 内容、继承属性、遮挡和呈现时序。`hasSurface=true`、`isVisible=true`、`HAS_DRAWN` 不足以共同证明当前用户一定看到内容。

建议首轮只熟悉：身份、父子关系、bounds/frame、flags、visible、hasSurface、drawState、parent/z、alpha/crop/transform、buffer、input region。其余字段遇到真实问题再深入。

## 核验参考链接

- [官方 WM 分析与 IntDef 翻译](https://source.android.com/docs/core/graphics/winscope/analyze/wm)
- [官方 SF 属性分类、继承与遮挡](https://source.android.com/docs/core/graphics/winscope/analyze/sf)
- [WM proto 定义](https://android.googlesource.com/platform/frameworks/base/+/master/core/proto/android/server/windowmanagerservice.proto)
- [WindowState 输出映射](https://android.googlesource.com/platform/frameworks/base/+/master/services/core/java/com/android/server/wm/WindowState.java)
- [WindowFrames 定义与输出](https://android.googlesource.com/platform/frameworks/base/+/master/services/core/java/com/android/server/wm/WindowFrames.java)
- [ConfigurationContainer 配置模型](https://android.googlesource.com/platform/frameworks/base/+/master/services/core/java/com/android/server/wm/ConfigurationContainer.java)
- [WindowManager.LayoutParams 常量](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/view/WindowManager.java)
- [SF proto，固定 commit](https://android.googlesource.com/platform/external/perfetto/+/0b7a36337d8cd8bf93adf6dd3eec94eaa2e3d1f2/protos/perfetto/trace/android/surfaceflinger_layers.proto)
- [LayerProtoHelper 快照输出，固定 commit](https://android.googlesource.com/platform/frameworks/native/+/776399cd109d54ed2ac9dfff71a18fbf2a37f5b9/services/surfaceflinger/LayerProtoHelper.cpp)
- [LayerSnapshotBuilder](https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/services/surfaceflinger/FrontEnd/LayerSnapshotBuilder.cpp)
- [Winscope SF presenter，固定 commit](https://android.googlesource.com/platform/development/+/ac04a428b678979a7af55e16c1df0c48bec4569e/tools/winscope/src/viewers/viewer_surface_flinger/presenter.ts)

后续待确认：用户实际版本/trace 的完整字段清单。可据实际属性树逐项扩展，并记录具体对象、字段路径、值和源码 commit；本次没有将通用解释写成用户现场的根因结论。

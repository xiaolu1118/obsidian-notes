# WMS 入门第一课：窗口概念与核心术语

创建日期：2026-11-01。更新日期：2026-11-01。
返回：[[wms窗口/索引|WMS 索引]]；后续：[[WMS学习路线与56天每日计划]]。

## 学习顺序修正

形象记忆辅助：[[WMS概念形象记忆：商场与舞台]]。用户要求概念讲解通俗形象、便于记忆；后续使用“场景→类比→准确定义→区别→自测”的方式，不以列出术语代替解释。

用户已明确：先通俗理解窗口与核心关键字，再进入源码与工具。原计划首日同时安排环境、dump、Winscope，对概念尚未建立的学习者负担过重。

建议先用本课完成概念预习，可分两次，每次 45～60 分钟；先会用自己的话解释，再开始原 Day 1。56 天计划仍保留为后续路线，不把预习当作用户已完成。

## 1. 窗口是什么

从 WMS 入门角度，把窗口理解为：**向系统登记的一块界面单元，具有位置、大小、层级、显示状态和输入规则**。实际可见内容还需要图形链路生产和合成。

它不一定有桌面窗口的边框，也不一定占满屏幕、是矩形可见内容或属于一个普通 Activity。全屏应用的主界面、Dialog、状态栏、输入法都涉及窗口。普通 Button、TextView 通常只是某个窗口内部的 View，不各自向 WMS 注册。

例如聊天页面弹出确认 Dialog，下方又显示输入法，上方还有状态栏：屏幕同时包含多个由不同组件提供的窗口。WMS 必须统一协调谁在前面、占哪些区域、是否允许显示、输入该交给谁。应用各自决定内部按钮怎样排布，不适合自行决定它与其他应用或系统窗口的全局关系。

“一块界面单元”是概念模型，不能推导为一个窗口必定只有一个图层或一个 buffer。

## 2. 先分清三个视角

| 视角 | 典型对象 | 关心的问题 |
|---|---|---|
| 应用内部 | Activity、Window/PhoneWindow、DecorView、View、ViewRootImpl | 页面承载什么内容、内部怎么布局、怎样提交绘制与窗口请求 |
| 系统管理 | WMS、WindowState、WindowToken、Task、DisplayContent | 属于谁、在哪个显示上、大小层级、焦点和显示策略 |
| 图形呈现 | Surface、SurfaceControl、Layer、SurfaceFlinger | 内容如何生产、图层如何组织变换、最终如何合成 |

同一个用户界面在三个视角下有不同对象。不能用 Window、WindowState、Surface 三个名字互相替换。

## 3. 第一组：Activity、Window、View 和 ViewRootImpl

- **Activity**：应用交互组件，具有生命周期，并参与任务和导航管理。普通 Activity 通常通过关联的 Window 承载界面；不能简单认为每个 Activity 实例始终对应一个已显示窗口。
- **Window**：应用侧的窗口抽象，管理顶层界面的外观、内容和一些行为；常见实现是 **PhoneWindow**。名字有 Phone，不意味着只适用于手机。
- **DecorView**：普通 Activity 窗口的顶层 View，其下包含应用内容及与主题相关的结构。`setContentView` 提供的布局通常放入它的内容区域，并不天然就是整个窗口的根。
- **View**：窗口内部的控件或布局节点，如 Button、TextView、LinearLayout。
- **ViewRootImpl**：连接顶层 View 树与系统窗口会话的桥梁，负责协调 traversal、绘制、输入等。它名字带 View，但自身不是一个 View，也不是 ViewGroup。
- **WindowManager**：应用侧管理窗口的接口，可以 addView/updateViewLayout/removeView；不要和 system_server 内的 WMS 混为一谈。

概念关系示意（不是精确调用栈）：

```text
Activity
  └─ Window（常见 PhoneWindow）
       └─ DecorView
            └─ 应用布局、Button、TextView……

顶层 View 树 ← ViewRootImpl → 窗口会话 → WMS
```

例子：聊天页面增加一个 Button，通常只改 View 树；显示一个普通 Dialog，通常会增加一个独立窗口和对应的顶层 View 管理对象。

## 4. 第二组：WMS、WindowState、LayoutParams、Token

- **WMS / WindowManagerService**：系统窗口管理服务。协调窗口的生命周期、布局、层级、焦点和可见策略，并与 ATMS、Input、图形系统协作。它通常不负责绘制应用内部每个控件。
- **WindowState**：WMS 内描述和跟踪一个窗口的核心服务端对象，例如窗口属性、frame、输入关联和绘制状态。可类比“系统登记档案”，但它是参与逻辑执行的活对象，并非只是一份配置表。
- **WindowManager.LayoutParams**：应用提出的窗口属性，例如宽高、位置、type、flags、softInputMode。它表达请求，最终效果还受权限、策略和显示环境约束。
- **type**：窗口的类别，帮助系统决定权限、所属关系和层级策略。常见类别包括应用窗口、子窗口、系统窗口；子窗口通常要依附适当的父窗口。
- **flags**：窗口的行为选项，比如是否具备获得焦点的资格、是否接收触摸、是否需要背景变暗。不是一个“窗口模式”就包含了所有行为。
- **token**：多处窗口接口使用的 Binder 标识，帮助关联所属实体并校验请求。需要区分 Activity token、父窗口标识及服务端 WindowToken 对象；它们不能统称为同一种“权限票”。具体含义由接口和窗口 type 决定。
- **WindowToken**：服务端用于组织相关窗口的一类容器对象；ActivityRecord 在现代实现中也参与这一体系。初学先掌握“系统需要知道窗口属于谁”，之后再看继承结构。
- **IWindow / IWindowSession / Session**：应用与窗口服务协作的接口及服务端会话实现。先理解系统既接收应用请求，也需要向客户端回调；名称和方法签名后续结合分支学习。

不应把所有 addWindow 失败都归因于 token；权限、type、display、重复添加等也可能造成失败。

## 5. 第三组：Surface、SurfaceControl、Layer、SurfaceFlinger

- **Surface**：从应用绘制角度理解，是与内容生产相关的接口，通过图形链路向消费者提供图像 buffer。并非通常意义上的一张静态图片，也不等于 Window。
- **buffer**：某次生产出的图像数据。视频、滚动、动画通常需要不断更新内容；并非屏幕每次变化都要求 App 重画所有内容。
- **SurfaceControl**：控制图层相关状态的句柄/接口，可用事务设置位置、裁剪、透明度、层级与父子关系等。
- **Transaction**：一次提交一组图层状态变更的方式，例如移动、裁剪、重挂。不要理解为 App 每次都靠 Transaction 绘制按钮文字。
- **Layer**：SurfaceFlinger 侧合成层级中的节点，可有内容，也可以是组织其他节点的容器。多个层可组合出一个应用窗口的视觉结果。
- **SurfaceFlinger / SF**：接收和组织图层内容，与硬件合成和显示链路协作，产出最终屏幕画面。

便于入门的类比：App 生产内容，WMS 协调窗口规则和布局，SurfaceControl 控制图层属性，SF 负责图层合成。类比只解释主要职责；实际事务也可能由 Shell、App 等参与方提交。

重要结论：**窗口与图层不是固定一对一**。一个窗口的视觉内容可能涉及多个 Layer，例如 SurfaceView 的独立内容层；动画还可能增加父容器 leash。

## 6. 第四组：大小、遮挡、可见和输入

| 术语 | 通俗解释 | 容易误解的地方 |
|---|---|---|
| frame / bounds | 某对象在相应坐标系中的矩形范围 | Task、Window、View、Layer 的范围不保证相同 |
| Z-order | 前后叠放关系 | 不总能用两个节点各自的一个数字直接比较，父子和相对层级也重要 |
| alpha | 透明程度 | 本层正常，也可能被祖先透明度影响 |
| crop | 限制显示区域的裁剪 | 位置正确但内容被裁掉，仍可能看不见 |
| transform | 平移、缩放、旋转等几何变换 | 父层也可施加变换，输入坐标还需对应 |
| visibility | 显示相关状态 | App 请求显示、系统允许显示、有内容、实际未被遮挡，是不同条件 |
| focus | 通常决定按键等焦点事件的目标 | 触摸通常还依赖命中区域和输入规则，不能只凭焦点判断 |
| touchable region | 窗口可以命中触摸的区域 | 不保证等于肉眼看到的整个矩形 |
| Insets | 为处理系统栏、IME、刘海等影响而提供的区域信息 | 不简单等同于给应用预留的黑边；应用可以绘制到相关区域并自行处理重叠 |
| IME | Input Method Editor，输入法 | 焦点、输入连接、目标窗口、Insets 和动画需协同 |

最常用的判断分离：

1. 有窗口，不代表已经有图像内容。
2. 有图像内容，不代表实际可见，可能被裁剪、透明或遮挡。
3. 可见，不代表具备焦点。
4. 有焦点，不代表任意坐标的触摸都会交给它。

## 7. 第五组：Task、Display、ATMS、Shell 和 Transition

- **Task**：任务/导航组织单元，通常包含一个或多个 Activity，关联窗口模式、范围及导航行为。不是进程，也不是单个窗口；同一应用可有多个 Task。
- **Display / DisplayContent**：显示设备或逻辑显示及其 WMS 侧管理对象。先知道窗口需要归属某个 display，再学习多显示。
- **WindowContainer**：服务端组织窗口相关对象的容器基类。任务、Activity、窗口等形成管理树；这棵树与 App View 树、SF Layer 树分别承担不同职责。
- **ATMS / ActivityTaskManagerService**：主要处理 Activity 与 Task 的生命周期、启动及导航组织，与 WMS 紧密协同。
- **WM Shell**：承担多窗口交互和现代转场等职责的组件。不要把它与命令行 adb shell 混为一谈；具体运行安排依产品实现。
- **Transition**：打开应用、返回、进入分屏等状态变化的一次转场组织过程，不仅是一段视觉动画。
- **leash**：动画常用的临时控制父节点。可以把相关内容挂在其下，统一移动、缩放、淡入淡出，再在结束时恢复和清理。
- **relayout**：窗口重新协商/更新布局及相关绘制资源状态的过程；不只表示 App 内部执行一次 View.layout。

这些属于第二轮理解内容；第一遍无需记住所有继承关系、方法名和转场阶段。

## 8. 用一个场景串起来

聊天 Activity 打开 → 应用通过 Window 承载 View 内容 → 顶层 View 经 ViewRootImpl 与窗口服务协作 → WMS 用 WindowState 等对象管理窗口 → 应用生产图像内容，图形链路提交并合成 → 屏幕上出现聊天界面。

点“删除”弹出普通 Dialog → 新窗口加入 → 系统协调它相对聊天窗口的位置、层级、焦点及背景变暗 → 点 Dialog 内输入框 → 输入法请求与输入目标建立 → IME 窗口显示并更新 Insets → 应用按自己的适配策略处理布局。

如果此时“输入框被键盘盖住”，应分别问：IME 有没有显示？系统报的 Insets 对不对？应用有没有消费 Insets/调整内容？不能尚未分层就认定 WMS 布局出错。

## 9. 概念预习验收

用户补充要求：窗口容器也属于概念阶段必学。继续阅读 [[WMS入门第二课：窗口容器与三棵树]]，理解容器关系后再进入源码与抓取。

- [ ] 能解释 Button 为什么通常不是独立窗口，Dialog 为什么通常是。
- [ ] 能分别用一句话解释 Activity、Window、ViewRootImpl、WindowState。
- [ ] 能区分窗口树、View 树、Layer 树。
- [ ] 能解释“窗口存在、有 buffer、可见、有焦点、能点到”为什么不同。
- [ ] 能解释 WMS、ATMS、Input、SF 在同一个显示/交互场景中的职责。

完成后再去抓一份 dump，把抽象概念对应到真实对象。不要求第一课背完所有关键字。

## 参考资料

本课为稳定基础概念的通俗整理，未新增在线核验；版本相关实现以原资料导航及实际源码分支为准。

- [AOSP：SurfaceFlinger 与 WindowManager](https://source.android.com/docs/core/graphics/surfaceflinger-windowmanager)
- [Android：Window](https://developer.android.com/reference/android/view/Window)
- [Android：WindowManager.LayoutParams](https://developer.android.com/reference/android/view/WindowManager.LayoutParams)
- [Android：窗口 Insets](https://developer.android.com/develop/ui/views/layout/insets)

# WMS 入门第二课：窗口容器与三棵树

创建日期：2026-11-01。更新日期：2026-11-01。
返回：[[wms窗口/索引|WMS 索引]]；前置：[[WMS入门第一课：窗口概念与核心术语]]。

## 背景和学习目标

配套形象记忆：[[WMS概念形象记忆：商场与舞台]]。先用商场的分组管理建立画面，再阅读本课的继承图和运行时包含图。

用户明确要求概念阶段也包括窗口容器。学习目标是理解系统为什么需要容器，以及对象怎样组织；先不背完整类图和具体调用栈。本课为概念讲解，尚未执行设备实验。

## 1. 为什么不能只有一堆 WindowState

假设某显示上有聊天、浏览器、输入法、状态栏等窗口。仅用一个窗口列表可以登记它们，但很难表达以下操作：

- 把某任务及其 Activity 一起移到另一显示。
- 让分屏中的一组任务共享区域和窗口模式。
- 管理 Activity 下主窗口、Dialog、启动窗口等相关窗口。
- 在调整任务范围时，传播配置并触发相关布局更新。

因此系统需要“分组、嵌套和统一操作”的结构。**窗口容器是 WMS/ATMS 管理树中的节点，表达组织关系和管理边界。它不一定自己展示内容。**类比文件夹有助于理解分组，但容器比文件夹多了配置、层级、Surface 和状态处理逻辑。

## 2. WindowContainer 是公共能力基类

WindowContainer 不是屏幕上额外的一种可见窗口，而是多种服务端对象共享的组织基础。核心认识：

- parent：这个节点归谁管理。
- children：它管理哪些下级节点。
- child order：子节点的顺序参与层级与遍历逻辑，但不应据此直接推断所有最终 SF 遮挡关系。
- add/remove：加入或移出一棵树。
- reparent：换父容器，常见于任务重组/多窗口场景。
- configuration/bounds：在层级中解析和传播配置，子节点可覆盖部分配置；不是父节点一个值无条件覆盖全部后代。
- Surface 相关组织：很多容器参与图层组织，但逻辑树不能直接视为 SF 树。

不要只把“容器”理解成没有内容的中间盒子。WindowState 也继承容器体系，子窗口等场景仍可能有 children。

## 3. 自上而下认识容器

| 名称 | 通俗理解 | 需要避免的误解 |
|---|---|---|
| RootWindowContainer | 整套显示/窗口管理结构的根入口，组织各显示 | 不是某个 App 的根 View |
| DisplayContent | 某个 display 在 WMS 中对应的管理对象 | 不等于一个 Activity 的内容布局 |
| DisplayArea | 按显示策略组织窗口和任务的区域/分组节点 | 不一定是肉眼可见的矩形，也不等于分屏的一半 |
| TaskDisplayArea | 用来组织 Task 的一类 DisplayArea | 不是 Task；其下可有多个 root task |
| Task | 组织 Activity/子任务的任务单元，承载任务级管理信息 | 不等于 App、进程、Activity 或单个窗口 |
| root task | 在相应任务区域中处于根层级的 Task 所扮演的角色 | 不是必须另外创建的 RootTask Java 类 |
| TaskFragment | 组织 Activity 等内容的容器，可支持任务内嵌入/分区 | 不要求每个 Task 下都有一个额外 TaskFragment 节点 |
| ActivityRecord | system_server 中代表一个 Activity 的管理对象 | 不是 App 进程中的 Activity 实例，也不是一个实际窗口 |
| WindowToken | 组织相关窗口并关联身份的一类容器 | 不等于接口参数中的 Binder token 本身 |
| WindowState | 系统中一个窗口的核心管理对象 | 不等于客户端 Window，也不包含整棵应用 View 树 |

TaskFragment 要特别注意：Task 本身在现代代码中继承 TaskFragment，所以 Task 已具有其基础能力；运行时也可以有额外 TaskFragment 子节点用于组织 Activity。不是“有继承关系，就一定多一层对象”。

## 4. 必须把继承图和运行时父子图分开

继承表示“这个类是一种什么”，运行时父子表示“这个对象被谁装着”。两张图不可混画。

简化的现代继承关系（省略泛型及其他父类）：

```text
WindowContainer
├─ RootWindowContainer
├─ DisplayContent
├─ DisplayArea
│  └─ TaskDisplayArea
├─ TaskFragment
│  └─ Task
├─ WindowToken
│  └─ ActivityRecord
└─ WindowState
```

其中 Task→TaskFragment、TaskFragment→WindowContainer、ActivityRecord→WindowToken 已通过本次检索的 AOSP 源码核对；搜索结果包含 main 和 Android 16 QPR 分支，不据此冒充固定 android16-release commit 的完整验证。所有具体继承、泛型及中间层以产品分支为准。

一个用于理解普通应用路径的**运行时包含示意**：

```text
RootWindowContainer
└─ DisplayContent（默认显示）
   └─ ……显示区域策略节点……
      └─ TaskDisplayArea
         └─ Task（此处扮演 root task）
            └─ Task（可选的下级任务）
               ├─ ActivityRecord（聊天列表）
               │  └─ WindowState（列表主窗口）
               └─ ActivityRecord（聊天详情）
                  ├─ WindowState（详情主窗口）
                  └─ WindowState（普通 Dialog，视具体类型和归属）
```

这里没有画成每个 ActivityRecord 下再套一个 WindowToken：**ActivityRecord 本身就是 WindowToken 的子类**。root task 可直接承载 ActivityRecord，也可组织子 Task；嵌入场景可增加 TaskFragment。示意树不是每个版本和每台设备都会出现的完整结构。

系统栏、壁纸、IME 等非 Activity 窗口有对应 token/区域组织，不能强行塞进应用 Task 中。DisplayArea 的精确组织和相对层级由策略决定，初学先理解“应用分支之外还有系统窗口分支”，不背一个伪通用完整树。

## 5. 三棵树分别管什么

| 树 | 典型节点 | 核心用途 | 例子 |
|---|---|---|---|
| App View 树 | DecorView、布局、Button | 一个窗口内部的布局、绘制与交互 | 按钮在输入框右边 |
| WMS 容器树 | DisplayContent、Task、ActivityRecord、WindowState | 显示/任务/Activity/窗口的组织、配置、层级与策略 | 聊天任务属于哪个显示/任务区域 |
| SF Layer 树 | 容器 layer、buffer layer、动画 leash | 图层的合成组织与几何属性 | 整个聊天画面跟随父层缩放 |

一个 WindowState 内可以有很多 View；一个窗口的视觉结果也可能涉及多个 Layer。因此三棵树之间不能机械一一对应。尤其 SF 中的动画 leash 不能简单认为必定对应一个新增 WindowState。

## 6. 用三种变化理解容器存在的价值

### 聊天列表进入详情

典型同任务场景：Task 下增加或切换 ActivityRecord，每个 ActivityRecord 管理各自相关窗口；界面内的按钮仍位于各自 App View 树。

### 详情弹出 Dialog

通常新增 WindowState，不需要因此创建新 Task 或新 ActivityRecord。Dialog 的具体窗口 type/token 决定归属；不要因为视觉上“压在上面”就认定它是主 WindowState 的 child。普通附着于 Activity 的 Dialog 与子窗口类型不能混为一谈。

### 应用进入分屏或移动任务

可能涉及 Task 重挂、范围/配置更新、转场和 Surface 层级变化。概念上不是把 App 进程剪成两半，也不是逐个手动移动 Button。具体分屏组织由版本、Shell 和 organizer 实现决定。

## 7. 后面遇到这些动词时应该想到什么

| 术语 | 应该先想到的问题 |
|---|---|
| addChild | 新节点挂在哪个父节点下？ |
| removeChild | 只解除关系，还是还有后续资源清理？具体看方法语义 |
| reparent | 原父、新父是谁？配置、Surface 和焦点是否要随之更新？ |
| positionChildAt | 在同级节点中的顺序如何改变？ |
| bounds change | 改的是 Task 范围、Window frame，还是图层 crop/transform？ |
| configuration override | 哪个节点覆盖了父级配置，最终解析值是什么？ |
| traversal | 遍历的是哪棵树？App traversal 与服务端容器遍历不同 |

## 8. 本课验收

- [ ] 能解释系统为什么需要容器树，而不是只有窗口列表。
- [ ] 能分别用一句话解释 DisplayContent、DisplayArea、Task、ActivityRecord、WindowState。
- [ ] 能指出 TaskFragment 继承关系和实际嵌套关系的区别。
- [ ] 能解释为什么 ActivityRecord 下不一定要再画 WindowToken。
- [ ] 能解释 root task 是 Task 的角色，而不是另一类窗口。
- [ ] 能区分 View 树、WMS 容器树、SF Layer 树。

## 本次参考源码

- [WindowContainer：父子组织与添加](https://android.googlesource.com/platform/frameworks/base/+/master/services/core/java/com/android/server/wm/WindowContainer.java)
- [TaskFragment：容器职责和继承](https://android.googlesource.com/platform/frameworks/base/+/master/services/core/java/com/android/server/wm/TaskFragment.java)
- [Task：继承 TaskFragment，且可包含子 Task](https://android.googlesource.com/platform/frameworks/base/+/master/services/core/java/com/android/server/wm/Task.java)
- [ActivityRecord：继承 WindowToken](https://android.googlesource.com/platform/frameworks/base/+/android16-qpr2-release/services/core/java/com/android/server/wm/ActivityRecord.java)

以上为导航分支链接，可能随分支演进；实际学习应记录自己的源码 commit。本课链接与检索证据支持基础模型，不宣称在用户设备上已核对运行时结构。

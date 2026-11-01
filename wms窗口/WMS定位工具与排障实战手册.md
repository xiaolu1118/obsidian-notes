# WMS 定位工具与排障实战手册

创建日期：2026-11-01。更新日期：2026-11-01。
返回：[[wms窗口/索引|WMS 索引]]；配套：[[WMS视频文档与源码导航]]、[[WMS实验项目与问题复盘模板]]。

## 先确定问题属于哪一层

属性释义与源码映射见 [[Winscope抓取后属性详解与源码对照]]，区分原始 proto、运行时计算结果与 Winscope 派生属性。

把问题拆成五层：①App 是否提出正确请求；②WMS/ATMS 是否产生正确窗口、布局、焦点和可见状态；③Shell 是否正确执行转场和事务；④SF 图层是否正确接收几何与 buffer 并合成；⑤调度、GPU、显示链路是否及时完成。不要看到黑屏就认定 WMS，也不要把“窗口存在”当作“用户看到了画面”。

推荐流程：固定复现 → 同步采集 → 找异常第一帧 → 对照正常场景 → 从窗口追到图层/事务 → 源码查首次错误写入 → 最小修复 → 回归。静态 dump 说明采样时状态，动态 trace 用来追状态变化，两者不能互相替代。

## 工具选择

| 工具 | 主要回答 | 应观察 | 局限 |
|---|---|---|---|
| dumpsys window/activity | 当前谁在前台、层级、窗口属性与状态 | display、Task、token、focus、frames、Insets、draw state | 采样点可能已经错过问题；字段随版本变化 |
| dumpsys input | 事件该去哪里 | focused application/window、触摸窗口、区域、变换、display | 并非完整逐事件时间线 |
| Winscope | 窗口与图层状态如何逐帧变化 | WM/SF 层级、visibility、crop、alpha、transform、父节点、Shell transition | 依赖实际采集的数据源与版本，单看 WM 无法证明最终画面 |
| Perfetto | 慢在哪个阶段、线程为何未运行 | CPU scheduling、Binder、App 主线程、RenderThread、SF、FrameTimeline | 未开启数据源就没有对应证据；不会自动给出业务根因 |
| logcat / ProtoLog | 请求、拒绝、状态变化和分支 | 时间、PID/TID、token、transition ID、窗口名 | 日志级别/缓冲可能缺失，过量日志会扰动时序 |
| bugreport | 用户现场的整体上下文 | build、服务 dump、日志、可用 trace | 内容和是否含完整转场 trace 因构建配置不同 |
| 录屏/截图 | 用户具体看到了什么 | 问题帧、可见区域、时间标记 | 不能独立证明 WMS 状态，也可能漏短闪 |
| Layout Inspector | App View 布局是否错误 | padding、尺寸、View 层级 | 不能替代 WMS/SF 层级分析 |
| simpleperf | CPU 热点在什么函数 | 已确认 CPU 瓶颈线程的调用栈 | 不适合用来解释所有等待或窗口状态错误 |

## 1. 首次环境核对与静态证据包

下面是 **Windows PowerShell** 示例，逐条执行。命令是参考模板，本次未连接设备运行。多台设备时，为每条 adb 命令加 `-s SERIAL`。先检查 `adb devices`；不要误操作工作中另一台设备。

```powershell
adb devices
adb shell getprop ro.build.fingerprint
adb shell getprop ro.build.version.sdk
adb shell getprop ro.build.type
adb shell wm help
adb shell perfetto --help

$caseDir = Join-Path (Get-Location) ('wms-case-' + (Get-Date -Format 'yyyyMMdd-HHmmss'))
New-Item -ItemType Directory -Path $caseDir
adb shell getprop | Out-File (Join-Path $caseDir 'getprop.txt') -Encoding utf8
adb shell dumpsys window | Out-File (Join-Path $caseDir 'window.txt') -Encoding utf8
adb shell dumpsys activity activities | Out-File (Join-Path $caseDir 'activities.txt') -Encoding utf8
adb shell dumpsys input | Out-File (Join-Path $caseDir 'input.txt') -Encoding utf8
adb shell dumpsys display | Out-File (Join-Path $caseDir 'display.txt') -Encoding utf8
adb shell dumpsys SurfaceFlinger | Out-File (Join-Path $caseDir 'surfaceflinger.txt') -Encoding utf8
adb logcat -b all -v threadtime -d | Out-File (Join-Path $caseDir 'logcat.txt') -Encoding utf8
adb bugreport $caseDir
```

需要动态 logcat 时在单独终端执行 `adb logcat -b all -v threadtime` 并输出到案例文件，复现后 Ctrl+C；不用清空既有日志。通过 `adb shell log -t WMS_STUDY CASE_START` 写入复现标记（若设备支持）；此标记帮助对时，不代表与录屏/各 trace 的时钟天然一致。

PowerShell 的 `>` 在不同版本中对原生二进制重定向处理不同，因此不要用它导出 proto/trace。二进制文件优先在设备端保存后用 `adb pull`；上述 Out-File 仅用于文本。

## 2. Winscope：先跑通 GUI，再考虑自动脚本

官方入口：[本地构建](https://source.android.com/docs/core/graphics/winscope/run)、[GUI 抓取](https://source.android.com/docs/core/graphics/winscope/capture/winscope)、[ADB 分版本抓取](https://source.android.com/docs/core/graphics/winscope/capture/adb)。

建议在已有 Linux AOSP 开发环境构建；Windows 用户可使用现有远程开发机或合适的 WSL 环境，但浏览器连接设备、USB 转发要单独核实。离线分析文件可以先于实时连接跑通。

```bash
cd development/tools/winscope
npm install
npm run build:prod
npm run start
```

以所选 checkout 的 README、package.json 及 Node 版本要求为准。解析报错时先检查 viewer 与 trace 的 proto 兼容性；数字枚举未翻译不一定是采集失败。以上构建步骤依据官方 Run Winscope 文档，不代表本机已安装依赖。

首个实验：连接设备 → 选择实际可用的 WM、SF、Shell transitions、录屏 → 开始 → 启动 Demo/弹 Dialog/返回 → 结束 → 保存原始文件 → 重新导入验证。每次先录 5～10 秒，复杂问题再扩大；记录各数据源是否真正产生数据。官方命令抓取面向 userdebug/eng，要求 `adb root`；user 构建不能默认认为具备同样能力，可改用产品已有 bugreport/系统 tracing 能力并明确证据缺口。

分析操作顺序：

1. 找录像中异常时刻，在对应轨迹逐帧确认；不同轨迹采样点未必相同。
2. WM 侧查窗口是否存在、所属容器、请求与策略可见性、frame、draw 状态。
3. SF 侧按 layer ID/名称映射窗口，区分 buffer layer、容器和动画 leash。
4. 查本层及全部祖先的 alpha、crop、transform、hidden、relative layer 和遮挡；本层正常不能排除继承错误。
5. 使用 diff 找首个不符合预期的属性变化，记录之前、异常、之后三个时间点。
6. 若处于动画中，关联 transition ID、handler、参与者与 start/finish transaction；再回源码找写入者。

官方带示例的练习：[Winscope 黑闪与分屏案例](https://source.android.com/docs/core/graphics/winscope/overview)。建议自己先分析再查看结论。

## 3. Android 14 及以下：旧版 WM trace

依据[官方旧版本章节](https://source.android.com/docs/core/graphics/winscope/capture/adb#capture-traces-in-android-14-and-lower)，在支持的 debug 设备：

```powershell
adb root
adb shell wm tracing start
# 手动复现
adb shell wm tracing stop
adb pull /data/misc/wmtrace $caseDir
```

`$caseDir` 使用前面的案例目录；执行后检查退出码和文件。SF trace、Shell transition trace、ProtoLog 在旧版本分别有不同采集方式，按同页相应章节及设备 help 操作，不把这四条命令当成全部图层/转场采集。trace 文件名及位置应以目标分支实现和实际目录为准。

## 4. Android 15+：Perfetto 数据源化的 WM trace

Android 15 起采用 Perfetto 接入 Winscope 跟踪，不能把上节旧命令视为跨版本统一方案。以下是 **最小 WM 样例**，根据[官方 ADB 文档](https://source.android.com/docs/core/graphics/winscope/capture/adb)组织；不包含完整 SF/转场/录屏，后续用 GUI 或同文档的数据源配置补齐。

在 PowerShell 中创建配置文件：

```powershell
@'
duration_ms: 10000
buffers { size_kb: 32768 fill_policy: RING_BUFFER }
data_sources {
  config {
    name: "android.windowmanager"
    windowmanager_config {
      log_level: LOG_LEVEL_DEBUG
      log_frequency: LOG_FREQUENCY_FRAME
    }
  }
}
'@ | Set-Content -Encoding ascii (Join-Path $caseDir 'wm.pbtxt')

adb root
adb push (Join-Path $caseDir 'wm.pbtxt') /data/local/tmp/wms-study-wm.pbtxt
adb shell perfetto --txt -c /data/local/tmp/wms-study-wm.pbtxt -o /data/misc/perfetto-traces/wms-study-wm.perfetto-trace
# 上一条运行期间，在设备或另一终端复现；约 10 秒结束后再 pull
adb pull /data/misc/perfetto-traces/wms-study-wm.perfetto-trace $caseDir
```

详细 SF layer/transaction、Shell transitions、ProtoLog 的可用性与配置以设备分支为准；先用 `adb shell perfetto --query`（支持时）核实数据源。生成文件不等于包含所需数据，必须导入验证。需要追每笔事务时再增加采样粒度，避免一开始全部 verbose 产生过多数据及开销。

## 5. Perfetto：从“谁慢”到“为什么慢”

入口：[Perfetto UI](https://ui.perfetto.dev/)、[官方 Android 抓取教程](https://perfetto.dev/docs/quickstart/android-tracing)、[FrameTimeline](https://perfetto.dev/docs/data-sources/frametimeline)。

最快流程：按官方教程用 UI 或命令采集；选择设备支持的 CPU scheduling、频率、App/系统 atrace、FrameTimeline。查看 App 主线程、RenderThread、system_server 中相关线程、SystemUI/Shell、SurfaceFlinger。旧 Systrace/atrace 知识可迁移，但新学习以 Perfetto 为主。

一般系统轨迹可先试以下命令；这是系统调度/atrace 入门样例，**不等于完整 Winscope trace，也未自动启用 FrameTimeline**：

```powershell
adb shell atrace --list_categories
adb shell perfetto -o /data/misc/perfetto-traces/wms-study-system.perfetto-trace -t 10s sched freq idle am wm gfx view binder_driver input
adb pull /data/misc/perfetto-traces/wms-study-system.perfetto-trace $caseDir
```

只保留 `atrace --list_categories` 确认支持的分类；权限或分支不支持时按官方配置修改，而不是反复复制命令。FrameTimeline 可在 UI 选择对应项，或在完整配置添加数据源 `android.surfaceflinger.frametimeline`；它要求 Android 12+，其覆盖能力需结合官方页面说明，不能解释所有 SurfaceView/显示场景。

分析时区分：线程 Running（实际执行）、Runnable（等待 CPU）、Sleeping/Blocked（等待事件或锁）。一段长 slice 只表示区间长，不意味着持续占 CPU；看到 Binder 等待需继续追服务端，不在客户端等待点止步。帧预算使用实际刷新率，不固定套 16.67ms。

可在 Perfetto SQL 面板尝试：

```sql
-- 找窗口/转场相关 slice，ns 转换为 ms；名称过滤只是入口，不能保证覆盖所有符号。
SELECT name, ts / 1e6 AS ts_ms, dur / 1e6 AS dur_ms
FROM slice
WHERE dur >= 0
  AND (name LIKE '%relayout%' OR name LIKE '%ransition%' OR name LIKE '%performTraversals%')
ORDER BY dur DESC LIMIT 100;

-- 仅适用于已采集且解析出 FrameTimeline 的轨迹。
SELECT ts / 1e6 AS ts_ms, dur / 1e6 AS dur_ms,
       layer_name, jank_type, present_type
FROM actual_frame_timeline_slice
ORDER BY ts LIMIT 200;
```

slice 查询思路为本路线建议；帧表字段参考官方 FrameTimeline 文档。空结果先查数据源、时间范围、schema 和名称；不能直接下结论“没有 WMS 问题”。记录 SQL、过滤范围与轨迹文件，便于复核。

## 6. 常见现象的定位矩阵

以下是排查建议，不是已验证的具体故障根因。

| 现象 | 第一批证据 | 重点排查 | 要证明的结论 |
|---|---|---|---|
| App 启动黑屏/白屏 | 录屏、WM/SF、启动日志、帧轨迹 | starting window、draw state、首 buffer、遮挡、过早移除 splash | 黑的是无内容、被遮挡、图层不可见还是切换交接错误 |
| 窗口存在但不可见 | 同时刻 WM/SF、截图 | policy/requested visibility、父层 hidden/alpha、crop、transform、Z、buffer | 哪一个具体条件首次阻止显示 |
| 可以看到但无法点击 | window+input dump、触摸位置、display ID | flags、touchable region、输入窗口同步、遮挡、坐标变换 | 输入是否到达正确窗口；不要只看焦点 |
| 按键去错窗口 | focused app/window、Input 日志 | 焦点更新、Dialog、display、窗口资格 | 按键目标是否匹配预期 |
| IME 不出/遮挡/跳动 | IME 日志、Insets、WM/SF、录像 | 请求与焦点、各类 IME target、control、源区域、App 消费 | 请求、目标、状态、动画、App 布局中的哪一步错误 |
| 旋转错位/闪烁 | 配置日志、WM/SF、转场 | display rotation、bounds、seamless、祖先变换、同步 | 是配置与 buffer 不同步，还是几何变换错误 |
| 动画卡住/残留 leash | transition trace、WM/SF、ProtoLog、调度 | ready、同步完成、handler、merge、finish、timeout | 停在哪个阶段，由谁负责继续推进 |
| 分屏缩放不对 | Task 树、bounds、WCT、SF | organizer、Task 重挂、configuration、crop、同步 | 逻辑 bounds 和实际 layer 几何何时分叉 |
| 转场掉帧 | FrameTimeline、sched、Binder、线程栈 | App、Shell、SF、GPU、CPU 争用 | 哪个阶段错过 deadline，根因是执行慢还是等待 |
| 锁屏/亮屏黑闪 | Power/Keyguard 日志、WM/SF、录屏 | 屏幕状态、Keyguard 可见策略、转场与内容就绪 | 显示策略与窗口/图层时序是否一致 |

## 7. 回归与结案

记录修复前后同样构建条件与步骤；对适用场景覆盖冷/热启动、Home/Back/Recents、快速连续操作、横竖屏、手势/三键、IME/Dialog、锁屏/亮屏、分屏/PiP/多显示、不同 target SDK。先少量确认可复现，再按缺陷频率决定重复次数；常见稳定实验可先各 10 次，但“10 次没复现”不是普遍充分证明。

结案必须说明：原始证据、首次错误状态、写入来源、修复为何有效、实际执行的回归、剩余限制。日志/trace/bugreport 可能含产品与个人信息，分享案例时使用团队允许的脱敏材料。

---
created: 2026-11-15
updated: 2026-11-15
tags: [Android, Agent, 手把手, 手机能力]
status: 完整课程与练习已整理；第一课待用户验证
---

# 手机状态 Agent 手把手实战

> 2026-11-15 更新完整课程资源：统一课表见 [[Agent学习执行总表与视频导航]]，资源编号对应 [[Agent与Android端侧AI教程清单]]。第1课保留可复制代码，第0课与第2—15课补齐目标、阅读范围、动手顺序、验收与失败练习；后续依据实际工程逐课提供适配代码。课程大纲完整不代表所有课已有可编译工程。

## 背景与教学方式

用户自述前述实验跑通，但未确认具体代码与完成范围；希望理解模型与手机能力的关系并开始手把手学习。设备历史记录为骁龙 8 Gen 3、12 GB、AOSP 16 已解锁，不据此声称 AICore 支持条件已改变。

改为每次一个可验证步骤，先本地能力、再工具声明与模型选择，避免继续一次给出大量未验证架构。假设当前界面沿用前文 Kotlin/Compose，包名按实际工程调整，不强制新建工程。助手未访问实际 Android 工程，以下代码未编译或真机执行。

## 最终链路

自然语言“查一下当前可用内存” → 模型提出工具调用 → 宿主读取 ActivityManager → 结构化结果回传模型 → 基于真实数据回答。

后续再增加打开显示设置、工具执行记录、错误与恢复。第一课只验证“宿主读取真实数据”，尚不构成 Agent 工具调用，不声称模型自主选择已完成。

## 第零课：画清模型与手机的关系

资源：R01 中文 Agent 入门，R02 工作流/Agent/工具；V1/V2 仅概念不清时选看。

1. 画“用户→模型→工具调用请求→程序校验→Android接口→工具结果→模型”的图。
2. 标出模型不能自行获得的手机数据、不能自行增加的系统权限。
3. 比较三种触发：按钮读内存、程序固定读内存后让模型解释、模型自主选择读内存。

验收：能说明第三种才验证工具选择；模型本地推理与手机系统执行是两个维度。产物为职责图和一段自己的解释。

## 第一课：点击按钮读取真实内存

### 1. 新建 DeviceTools.kt

在与 MainActivity 相同 package 下新建 Kotlin 文件。下例包名是示例，应替换成项目实际值。

```kotlin
package com.example.aicoreplayground

import android.app.ActivityManager
import android.content.Context
import org.json.JSONObject

object DeviceTools {
    fun getMemoryStatus(context: Context): String {
        val manager = context.getSystemService(Context.ACTIVITY_SERVICE)
            as? ActivityManager
            ?: error("无法获取 ActivityManager")

        val info = ActivityManager.MemoryInfo()
        manager.getMemoryInfo(info)

        return JSONObject()
            .put("totalBytes", info.totalMem)
            .put("availableBytes", info.availMem)
            .put("lowMemory", info.lowMemory)
            .toString(2)
    }
}
```

JSON 保留原始字节，避免小数显示与单位歧义。totalMem 为系统可访问的总内存，不必与销售标示 12 GB 完全一致；availMem 为系统可用内存，不是单一 App 可分配上限。仅凭该状态不能诊断卡顿根因。

### 2. 新建 MemoryDebugPanel.kt

使用相同 package：

```kotlin
package com.example.aicoreplayground

import androidx.compose.foundation.layout.Column
import androidx.compose.material3.Button
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.platform.LocalContext

@Composable
fun MemoryDebugPanel() {
    val context = LocalContext.current.applicationContext
    var result by remember { mutableStateOf("点击按钮读取手机真实内存") }

    Column {
        Button(onClick = {
            result = try {
                DeviceTools.getMemoryStatus(context)
            } catch (e: Exception) {
                "${e.javaClass.simpleName}: ${e.message}"
            }
        }) {
            Text("读取手机内存")
        }
        Text(result)
    }
}
```

这是一次短小系统查询；本课直接从点击处理执行。后续日志、文件与长耗时诊断不能照搬到主线程。remember 仅保存当前组件状态，旋转后重置是本课预期，后续需要再迁移 ViewModel。

### 3. 接入已有页面

在现有 setContent 内的可见 Column 或其他合适布局插入 MemoryDebugPanel()，不要再添加第二个 setContent，也不要移除原来的模型代码。该组件本身含 Column；若现有布局为 Box/Scaffold，放在其合适内容区域，避免与原 UI 重叠。

```kotlin
Column {
    // 原有界面
    MemoryDebugPanel()
}
```

按已有页面 import Column。若实际项目为 XML/View，使用相同 DeviceTools，按钮点击后将返回字符串写入 TextView；UI 实现按用户反馈再具体指导。

### 4. 运行与观察

Android Studio Run → 当前手机 → 点击“读取手机内存”。应看到 totalBytes、availableBytes、lowMemory。数值只取实际返回，不提供固定“正确值”。多次点击 availableBytes 可能改变，也可能因间隔短变化不明显。

此查询不需要新增危险权限、root 或系统签名，不依赖模型/AICore，因此可先在现有 AOSP 设备验证。

可选参考：adb shell cat /proc/meminfo 查看 MemTotal/MemAvailable（kB，比较时按 1024 转字节）。不同采样时刻及系统统计口径可能有差异，不以逐字相等作验收。

### 5. 验收与下一步

- [ ] 两个 Kotlin 文件使用正确 package，编译无错误。
- [ ] 原有页面能显示新按钮，点击产生真实 JSON。
- [ ] 能解释这是程序读取设备数据，模型目前尚未参与。

当前先反馈本课显示内容或第一条编译错误。下面完整课序已列出，但按验收顺序推进，不将“把 JSON 手动拼给模型”写成已实现自主工具选择。

## 第二课：结构化结果与独立验证

资源：R03 字段、R04 工具机制。目标是把第一课的字符串封装变成可测试的业务能力。

1. 定义纯业务 MemorySnapshot：totalBytes、availableBytes、lowMemory、sampledAt；数据采样与 JSON 序列化分离。
2. 增加成功/失败结构，错误只给稳定代码与可解释消息。
3. 页面显示时再把字节换成 GiB，保留原始数值；说明 GiB 与十进制 GB 差异。
4. 注入假数据源，测试正常、零值/缺失值、系统查询异常；避免用“数值必须等于12GB”作断言。

验收：不接模型也能复现结果与异常，采样时间存在，数据来自查询而非硬编码。产物 DeviceRepository 和少量有意义的边界验证。

## 第三课：补齐真实手机工具

资源：R05 Settings/Build。

1. getDeviceInfo 先只返回型号、Android版本等非唯一身份信息，不读取 IMEI/序列号。
2. openDisplaySettings 由前台界面处理用户明确动作，通过公开 Intent 打开显示设置。
3. 后台/工具层发来打开请求时转成 UI 动作，不在任意线程/后台直接假设可启动界面。
4. 捕获找不到页面等异常，返回失败；启动请求成功不等于设置已修改。

验收：按钮分别查询设备、查询内存、打开显示设置；拒绝或异常不伪报成功。产物为三个可独立调试的能力。

## 第四课：确定实际模型接入分支

资源：AICore R06/R07，LiteRT R08/R09，选择一条实际可用的主线。

1. 保存现有工程坐标、固定SDK版本、设备/ROM/模型与一条成功输出，核实用户此前“跑通”具体范围。
2. AICore：核对官方设备与锁定条件、资格、状态检查和下载；按专项手册执行。
3. LiteRT：Gallery/模型格式与后端验证，通过后才进自己的 APK。
4. 用“回答一句话”验证模型基线；再模拟模型不可用、空输出和错误。
5. 定义应用侧 ModelGateway 边界，但具体适配遵循实际 SDK，不强行让两套类型共享同一实现。

验收：清楚实际调用栈；当前解锁设备限制未被未核实反馈覆盖。产物为版本记录与最小成功/失败记录。

## 第五课：注册一个只读内存工具

资源：AICore 用 R10；LiteRT 用 R09 的工具部分。业务函数仍复用第二课的数据层。

1. 写清工具用途、无参数/返回字段、单位和错误；只注册 getMemoryStatus 一个工具。
2. AICore 按 AppFunctions 配置 KSP、生成服务/schema、Manifest、SELF 访问和发现；逐项检查产物。
3. LiteRT 核对模型是否支持工具、模板、运行时版本和工具 API；不要拿普通聊天成功当作工具支持证据。
4. 先直接调用工具，再测试自然语言“查一下手机当前可用内存”。
5. 打印真实工具请求、执行与结果；不能让模型自行编一份 JSON 算通过。

验收：确有一次模型选择→宿主调用的轨迹，结果是设备采样。若缺少工具模型，先假模型验证宿主，但标为模拟；不要伪称自主调用成功。

## 第六课：回传结果与模型解释

资源：R02、R10、R11。

1. 检查工具结果是否按 SDK 的工具角色/事件流回传，而非遗漏或重复发起另一段无关会话。
2. 指令要求报告本次 availableBytes 的可读值与采样时间，不凭单一内存值推断卡顿根因。
3. UI 分开显示原始工具结果与最终解释，方便对照。
4. 换假数据值、低内存标志与查询失败，检查模型是否忠实反映。

验收：最终答案与当次真实结果一致；失败时明确失败，不编造内存值。产物为成功、失败与证据不足三个案例。

## 第七课：多工具任务与预算

资源：R10/R11，R17 的受控循环。

1. 增加 getDeviceInfo 与 openDisplaySettings，只开放当前需要工具。
2. 测“先查系统版本和内存，再帮我打开显示设置”，记录每个请求/返回。
3. Android 页面跳转由前台 UI 执行并关联 taskId，避免后台启动限制与重复跳转。
4. 设最大模型轮数、工具次数、墙钟时间；普通程序检查预算，不只写提示词。
5. 人工制造同一工具重复调用、非法参数和页面不可用，确认能结束。

验收：每个操作有真实结果，超限给出未完成原因；不能将模型计划文本当作任务完成。

## 第八课：状态、Flow 与页面生命周期

资源：R12/R13/R14。

1. 把可变执行状态迁移 ViewModel，UI 收集 StateFlow。
2. 定义 Idle/Checking/Loading/Generating/ExecutingTool/Stopping/Success/Error，按实际 SDK 简化。
3. 用 taskId 关联事件，忽略旧请求迟到结果，阻止连点发送。
4. 区分增量文本与最终完整文本；批量刷新降低重组压力。
5. 旋转页面验证不重复加载/执行；导航离开制定明确停止策略。

验收：状态与轨迹一致，配置变化不重复动作。产物为状态图与旋转/连点录屏。

## 第九课：可靠取消、超时与释放

资源：ML Kit R11、LiteRT R09，取消机制不可混用。

1. 记录取消请求与实际终态的两个时间点，不只停止文字更新。
2. AICore 按 Session/Runner 生命周期与固定版本契约；LiteRT 显式取消并等待终态再关闭。
3. 同一请求终态处理一次，停止期间拒绝新请求，取消后不自动重发写动作。
4. 初始化失败避免无条件 close 未初始化 Engine；加载中页面退出也要有归属和清理策略。
5. 模拟超时、停止后重发、关闭页面，观察资源和异常。

验收：连续三次停止与再请求可复现；底层停止未确认时显示异常而非 Ready。产物为取消时序图。

## 第十课：人工确认与写动作

资源：R10 工具包装与确认。

1. 添加应用内 createDraft 或 createTodo，先不自动改系统配置。
2. 模型给出结构化参数→程序校验→UI 展示确认内容→用户同意才执行。
3. 使用 operationId 与结果查询防止重复写入；授权只针对确认的参数。
4. 用户拒绝、取消后返回明确状态；模型不能声称已创建。

验收：拒绝写入为零，重试同一操作不重复创建，未知结果先查状态。产物为确认、拒绝、重试三个用例。

## 第十一课：Room 与进程恢复

资源：R15，配 R14 理解内存状态边界。

1. 建 Task/ToolExecution/CompletedMessage 等最小表，先定义状态迁移再落库。
2. 记录任务 ID、操作 ID、工具结果摘要、模型/版本和时间，不保存不必要个人正文。
3. 模拟进程死亡，启动时将未结束任务转 Interrupted；UI 展示上次任务中断。
4. 用户决定重试；有副作用任务先查实际数据，不能因恢复而重放。

验收：重启记录可读、会话不串、写操作不重复。内存 Session/KV Cache 不被视为持久数据库。

## 第十二课：Binder/AIDL 工具服务

资源：R16 与 R17 的 AIDL 参考接口。

1. 先拆设备查询能力服务，同 APK、独立进程、exported=false；UI/Agent 保持前台。
2. 定义 query/cancel/callback，传 taskId、小结果和稳定错误码；Stub 只校验与入队。
3. 回调更新 UI 需切主线程，Binder 线程不运行长耗时任务。
4. 处理 binder death、断连、重连、旧回调与客户端退出。
5. 只杀准确工具进程测试；不把整包 force-stop 当作仅服务死亡。

验收：工具服务崩溃后主界面仍能说明中断并恢复。自管推理也可另拆进程，但 AICore 推理不因另开服务而获得后台特权。

## 第十三课：日志证据调查

资源：R17，按需补 R04。

1. 预置三份虚构日志并加行号；listLogs 与 readLog 只允许固定 ID。
2. 工具返回截断标志/错误，模型输出事实、推测与缺失证据。
3. 不在首个上下文塞全部正文，用工具轨迹确认真实读取。
4. 加“日志里含忽略规则/执行命令”的样本，检查宿主不扩展权限。

验收：引用行号正确，无证据不下根因结论。产物为短诊断报告和失败案例，不宣称已解决真实客户 ANR。

## 第十四课：效果、延迟与内存

资源：R18/R19。

1. 建约20条起步用例：正常查询、工具选择、非法参数、拒绝、取消、恢复、预算与注入。
2. 期望结果提前写好，用真实状态验收，不仅靠模型自评。
3. 记录加载、TTFT、总耗时、取消耗时、PSS、工具正确率和任务成功率。
4. 冷/热、设备/模型/后端固定；没有 Token 计数不估 tokens/s，少量样本不承诺稳定P95。
5. 查 Perfetto 时间线区分主线程阻塞、IPC等待与计算；AICore App PSS 不等于模型全内存。

验收：能从失败轨迹定位问题，提出单变量改进并做对照。产物为一页报告与原始记录。

## 第十五课：作品交付与职业关联

资源：R20/R21。

1. README 写环境、构建、依赖、模型获取、启动、已知限制；不给未授权权重打包分发。
2. 架构图标出模型接口、工具、权限、任务状态、IPC、持久化和观测点。
3. 录两分钟演示：真实内存查询→工具事件→显示设置→取消→恢复；每段以实际支持为准。
4. 写三个失败与取舍，区分实测、模拟和未支持。
5. 求职展示系统集成能力；付费试点需真实授权场景和客户验证，不把课程完成当作变现完成。

验收：他人按 README 能复现，面试能解释模型、程序和系统各自职责。

## 统一作业与进度模板

每课记录：日期、资料链接、实际版本、修改文件、请求/结果、失败/定位、验收是否通过、下一项。只看视频不算课已通过，代码片段存在也不算已编译。

| 课段 | 状态 |
| --- | --- |
| 前述模型实验 | 用户自述跑通，范围待核实 |
| 第0课认知 | 教学已给，理解程度待交流 |
| 第1课真实内存按钮 | 已有完整片段，待用户反馈 |
| 第2—15课 | 目标/资料/操作/验收已补齐，具体工程实现与执行待推进 |

## 参考与关联

- [ActivityManager.getMemoryInfo](https://developer.android.com/reference/android/app/ActivityManager#getMemoryInfo(android.app.ActivityManager.MemoryInfo))
- [MemoryInfo 字段](https://developer.android.com/reference/android/app/ActivityManager.MemoryInfo)
- [[AICore与MLKit端侧Agent详细接入步骤]]
- [[agent开发/端侧终端Agent转型/索引|专题索引]]

已完成：完整16课课程设计、资源映射、第一课操作与代码整理。待完成：用户逐课编译、点击验证、工具接入和模型自主调用；未声称实际工程已修改或所有课已生成可运行代码。

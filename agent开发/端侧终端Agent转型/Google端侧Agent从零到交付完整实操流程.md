---
created: 2026-11-14
updated: 2026-11-14
tags: [Android, Agent, LiteRT-LM, AIDL, 实操路线]
status: 详细实施手册；未执行真机验收
---

# Google 端侧 Agent 从零到交付完整实操流程

## 0. 使用范围与最终成果

已确认：用户为 Android Framework 开发者，对 Agent 认知较少，选择 Google 技术路线；设备为骁龙 8 Gen 3、12 GB、AOSP 16、bootloader 已解锁。尚未确认 Gallery 安装、模型下载或 Kotlin 工程已跑通。本文记录完整实施流程，不把教程写成已完成成果。

建议作品：AndroidLogAgent。输入“检查有哪些日志，读取 ANR 示例，列出事实与缺失证据”；系统选择两个只读工具 listLogs/readLog、返回工具结果、继续生成最终回答，记录执行过程。随后将推理迁移到同 APK 的独立进程服务，通过 AIDL 调用。

采用 App → LiteRT-LM → 自管兼容模型 → CPU/GPU。AICore/Gemini Nano 的受支持接入不适用于当前已解锁设备。NPU、修改 system_server、真实跨应用控制和商业部署均不作为当前实验的前置任务。

**本文取代此前聊天中的零散代码作为执行顺序。代码块分为配置、接口与核心逻辑片段，并非已经编译验证的完整 Android 工程。** 学习顺序是：环境 → Gallery → 最小 APK → 生命周期 → 工具 → Agent 循环 → AIDL → 可靠性与指标 → 演示交付。

| 阶段 | 本阶段产物 | 通过后再继续 |
| --- | --- | --- |
| P0 | 设备与工具记录 | adb 可用，确认 arm64 与存储 |
| P1 | Gallery 离线推理记录 | 能生成新回复 |
| P2 | 自己的最小推理 APK | 能读取模型并生成 |
| P3 | 状态机与可靠停止 | 停止后能再次请求 |
| P4 | 两个可独立运行的工具 | 不使用模型也能验证工具 |
| P5 | 有步数上限的工具循环 | 真正调用工具并引用返回证据 |
| P6 | 独立进程推理服务 | Binder 断连、重连和取消可解释 |
| P7 | 评测、演示与复盘 | 可以用数据说明效果与局限 |

所有耗时和样本量均是学习建议，非厂商性能承诺。

## 1. 环境准备与证据留存

### 1.1 电脑

1. 安装能构建当前模板的 Android Studio，使用其配套 Gradle JDK，先不要手动拼接 AGP/Kotlin/JDK 版本。
2. SDK Manager 安装 Android 16 对应 SDK Platform、Platform-Tools，以及模板要求的 Build Tools。
3. 先不编译 LiteRT-LM C++ 源码；使用 Maven Android SDK。仅在实际构建要求时安装匹配 NDK。
4. PowerShell 可执行 adb；以下命令在你自己的实验目录执行，输出文件保存在当前目录。
5. 多设备时，所有 adb 命令显式加 -s 序列号。

```powershell
adb version
adb devices -l
adb shell getprop ro.product.model
adb shell getprop ro.soc.model
adb shell getprop ro.product.cpu.abi
adb shell getprop ro.build.version.release
adb shell getprop ro.build.fingerprint
adb shell df -h /data
```

预期设备状态为 device、ABI 为 arm64-v8a。unauthorized 时到手机确认调试授权。属性为空可能是 ROM 实现差异，保留完整输出，不单凭空值判断兼容性。

建议预留约 10 GB 存储，真实需求按模型与临时文件调整。正常电量、温度下实验，暂关其他重负载应用。

### 1.2 建立实验记录

记录：日期、机型/SoC、RAM、fingerprint、App 版本、SDK 解析版本、模型仓库/文件名/哈希/许可、后端、输入、输出和错误。不把 GGUF、task、litertlm 视为可互换格式。

```powershell
Get-FileHash -Algorithm SHA256 "$env:USERPROFILE\Downloads\model.litertlm"
```

模型下载需要网络，模型仓库可能要求账号和接受许可；离线推理实验不要求云端模型 API Key。下载授权与模型推理是不同阶段。

## 2. Gallery 安装与设备验证

1. 打开 [官方最新发布](https://github.com/google-ai-edge/gallery/releases/latest)。本次核验为 1.0.23。
2. 展开 Assets，先取通用 ai_edge_gallery_app.apk。专用 sm8650 包与 NPU 模型留到后续，不因芯片匹配就推定自编 AOSP 驱动兼容。
3. 安装，成功后在手机上打开。

```powershell
adb install -r "$env:USERPROFILE\Downloads\ai_edge_gallery_app.apk"
adb shell pm list packages com.google.ai.edge.gallery
```

仅遇到 INSTALL_FAILED_TEST_ONLY 时再加 -t；-t 允许测试 APK，不能绕过最低系统版本。签名冲突先核对旧版来源，不默认卸载清数据。

4. 进入 AI Chat/文本聊天，选应用列表提供的较小兼容模型；若跳转 Hugging Face，登录并接受许可。
5. 下载完成后加载，先短输入；如后端可选则 GPU，失败时尝试该模型支持的 CPU。界面名称随版本变化，以功能为准。
6. 输入“请用三句话解释 Android Binder”，生成完成后开启飞行模式并关闭 Wi-Fi，重新进入新会话再问“ANR 和 Java Crash 的区别”。
7. 保存模型、后端、结果、耗时和错误。只证明该文本路径离线，不推断联网工具也离线。

详细排障见 [[骁龙8Gen3运行GoogleAIEdgeGallery操作手册]]。Gallery 尚未通过时先定位模型/驱动，不在自己的工程同时引入新模型、新版本和新后端。

## 3. 新建 Android 工程与固定版本

### 3.1 创建

Android Studio → New Project → Empty Activity → Kotlin；项目 AndroidLogAgent，包名统一用 com.example.androidlogagent，minSdk 31 作为实验选择。先 Run 模板，确认基础 APK 能安装启动。

最终文件职责建议：

```text
app/src/main/
  AndroidManifest.xml
  java/com/example/androidlogagent/
    MainActivity.kt          UI 与状态展示
    AgentViewModel.kt        用户操作与配置变化
    InferenceController.kt  引擎、会话、取消与清理
    AgentCoordinator.kt     调查循环、预算、证据
    ToolDispatcher.kt       工具白名单与参数校验
    DemoLogRepository.kt    虚构日志数据
    TraceRecorder.kt        脱敏执行事件
    InferenceService.kt     P6 才增加
  aidl/com/example/androidlogagent/ipc/  P6 才增加
```

先只建 UI 和推理控制器，其余文件到对应阶段再添加。

### 3.2 Gradle

settings.gradle.kts 的 dependencyResolutionManagement.repositories 包含 google()、mavenCentral()。app/build.gradle.kts 加：

```kotlin
dependencies {
    implementation("com.google.ai.edge.litertlm:litertlm-android:latest.release")
}
```

保留模板已有 Compose/Activity/Lifecycle 依赖；需要 viewModelScope 时补 lifecycle-viewmodel-ktx，使用模板现有的同系列版本，不编造固定版本。

defaultConfig 中设 arm64：

```kotlin
ndk { abiFilters += "arm64-v8a" }
```

同步后，在项目根目录查询实际版本：

```powershell
.\gradlew.bat :app:dependencyInsight --dependency litertlm-android --configuration debugRuntimeClasspath
```

把输出实际解析版本写回 implementation，替换 latest.release，记录 Gradle/AGP/Kotlin 版本。版本号取实际解析结果；GitHub 当前 release v0.19.1 不自动证明 Google Maven 同名版本已经发布。

用 Android Studio External Libraries/下载 sources 查看该固定版本的 Engine、Conversation、ConversationConfig、Message 和 Tool API。官方网页与 main 分支可能比发布包更新，出现签名差异先按固定版本核对，避免同时粘贴不同代示例。

### 3.3 Manifest

在 application 内加：

```xml
<uses-native-library android:name="libvndksupport.so" android:required="false" />
<uses-native-library android:name="libOpenCL.so" android:required="false" />
```

它们不安装驱动。实验通过 adb 放模型且不联网时不必为模型推理加 INTERNET。此处不申请无障碍、系统签名、root、所有文件访问或后台常驻权限。

### 3.4 编译与安装

```powershell
.\gradlew.bat :app:assembleDebug
adb install -r .\app\build\outputs\apk\debug\app-debug.apk
```

在手机启动一次，确保目录初始化。每个阶段通过后保存一个本地 Git 提交用于回退；不提交模型、账号凭据或真实日志。

## 4. 模型选择、校验与导入

### 4.1 首轮只验证聊天

可使用官方 Kotlin 指南链接的 [Gemma3-1B-IT 文件列表](https://huggingface.co/litert-community/Gemma3-1B-IT/tree/main)。本次可见通用 gemma3-1b-it-int4.litertlm，页面标示约 584 MB，作为候选；实际以下载时模型卡和运行时兼容说明为准。

1. 浏览器登录 Hugging Face，接受 Gemma 许可。
2. 打开 Files，选通用 .litertlm；不先选带 sm8650/Tensor/mt 标签的后端专用包。
3. 下载实际二进制文件，避免保存登录 HTML 或 Git LFS 指针。
4. 可在电脑将有效文件副本命名为 model.litertlm，仅为统一实验路径；改名不转换格式。
5. 用 Get-FileHash 记录哈希。模型文件大小不代表运行时总内存。

聊天候选只用于 P2/P3，不能据它能回答问题就断定其支持 P5 的原生工具调用。

### 4.2 adb 导入

本手册包名与早期 NanoPlayground 不同，勿把两套路径混用。

```powershell
adb shell mkdir -p /sdcard/Android/data/com.example.androidlogagent/files/models
adb push "$env:USERPROFILE\Downloads\model.litertlm" /sdcard/Android/data/com.example.androidlogagent/files/models/model.litertlm
adb shell ls -lh /sdcard/Android/data/com.example.androidlogagent/files/models/model.litertlm
```

App 内得到同一路径：

```kotlin
val root = requireNotNull(context.getExternalFilesDir(null))
val model = java.io.File(root, "models/model.litertlm")
require(model.isFile && model.length() > 0L)
```

如 adb 被 ROM 限制，给 App 加 OpenDocument 选择器，获取 URI 后用 ContentResolver.openInputStream，在后台复制到应用目录的临时文件；复制完成、校验长度/哈希后再原子改名，不直接把 content:// 当作模型文件系统路径。复制失败删除本次临时文件，保留旧模型。卸载或清数据前先备份需要的模型和实验记录。

## 5. 单次生成：先建立可观察闭环

界面最少包含：模型路径、后端选择、加载按钮、输入框、发送按钮、输出区、状态和错误。初期只允许一个请求。

1. 后台创建 EngineConfig，传绝对 modelPath、Backend.GPU()，需要时配置应用可写 cacheDir。
2. 后台 initialize，成功记录加载耗时并进入 Ready。
3. Ready 才允许创建 Conversation；首轮输入固定为一句话。
4. 同步 sendMessage 只能在工作线程上运行；成功后把文本投递到主线程。
5. 本次 Conversation 使用完关闭；下一请求重建。引擎复用策略见下一节。
6. 加载异常显示原始类型与消息；Engine 未成功 initialize 时不要无条件 close 导致二次异常掩盖原因。

初始化、推理不能放进 Composable 函数体，重组可能重复执行。Compose 显示 StateFlow/不可变状态，动作交给 ViewModel。

GPU 失败时确认没有活跃会话，卸载已初始化 Engine，再用同一模型的 CPU 支持路径重建。无法初始化的模型/后端组合不做无限重试，也不把自动切后端掩盖成“正常成功”。

验收：自己的 APK 能读取指定文件、回答短问题、断网新请求成功。前面聊天的简化代码仅适用于正常完成路径，下一节整体替换生命周期管理，不能把它当作已具备可靠停止。

## 6. 流式、取消与生命周期完整流程

### 6.1 控制器与线程

ViewModel 持有 InferenceController，控制器仅使用 applicationContext。生命周期事件使用独立串行 Executor/actor，避免不同 Binder/UI 回调竞争原生句柄。

- 控制器工作队列：加载、建会话、发异步请求、请求取消、终态清理和卸载。
- SDK callback：只复制分片或异常并投递事件，不直接操作 UI，不在回调栈里重入 close。
- 主线程：根据状态渲染，不执行模型加载和关闭。
- 一个 Engine 同时最多一个活跃 Conversation。停止期间不接新请求。

### 6.2 状态与事件

```text
Unloaded --load--> Loading --success--> Ready
Loading --failure--> Error
Ready --send--> Generating --done--> Ready
Generating --stop--> Stopping --terminal callback--> Ready
Generating/Stopping --failure--> Error
Ready --unload--> Unloaded
任意状态 --dispose--> 先停止/等待，再释放并进入 Disposed
```

UiState 建议字段：phase、requestId、output、error、loadMs、ttftMs、totalMs、backend、modelId。内部还需 activeConversation、terminalHandled、stopRequested、disposeRequested；终态处理必须幂等。

### 6.3 开始生成

1. 校验 phase == Ready，trim 后输入非空；输入长度按所用模型上下文能力限制，先设保守字符上限，记录这不是精确 Token 上限。
2. 分配 UUID requestId，清空本次输出和计时，创建新 Conversation。支持的版本用 ConversationConfig(maxOutputToken = 256) 起步。
3. 状态改为 Generating，在工作队列调用 sendMessageAsync(prompt, callback)。注册回调前即完成活动请求字段设置，避免同步报错导致漏清理。
4. onMessage 复制文本并投递；核对 requestId 仍匹配，才追加到缓冲。分片不等于 Token；UI 可每 30—50ms 批量刷新，减少重组。
5. 第一段非空文本到达时只记录一次 TTFT。区分普通输出、工具调用和其他通道，调试时不要把整个对象当最终用户文本。
6. onDone/onError 投递终态事件；当前请求仅处理一次。关闭本次 Conversation 后才清空活动字段和恢复操作按钮。
7. 如果 close 异常，进入 Error，禁止把不明状态的 Engine 继续视为健康；保留原始失败和清理失败两份信息。

核心接口形状如下，必须嵌入上述串行控制器，而不是直接从多个按钮调用：

```kotlin
conversation.sendMessageAsync(prompt, object : MessageCallback {
    override fun onMessage(message: Message) {
        // 复制 message 文本；投递 Chunk(requestId, text)
    }
    override fun onDone() {
        // 投递 Terminal(requestId, null)
    }
    override fun onError(throwable: Throwable) {
        // 投递 Terminal(requestId, throwable)
    }
})
```

### 6.4 停止

1. 点击停止，校验请求仍活跃且未处理终态，改成 Stopping。
2. 在控制器队列对当前会话调用 cancelProcess()；记录这是取消请求而非已完成停止。
3. 保留分片输出，并标注未完成。等待 onDone/onError 终态后执行一次清理。
4. 使用 requestId 忽略迟到分片/旧终态。停止之后新请求采用新 Conversation，不沿用可能部分生成的历史。
5. 设一个可配置观察超时，例如 10 秒，属于实验策略；超时仍未到终态，进入“停止未确认”错误，阻止新请求，不同时强行 close 原生对象。
6. 如必须硬恢复，可在下一阶段对独立推理进程设计明确的重启流程。不要对应用主线程/任意 native 线程做强杀。

当前官方源码 Flow 路径 awaitClose 为空，不能用 job.cancel 代替显式底层取消。实际 SDK 固定版本需查对应源码。

### 6.5 旋转、退出与卸载

- 旋转：ViewModel 保留控制器，Activity 重建后重新观察状态；不得重新加载相同 Engine。
- 返回离开：界面与推理生命周期分离；定义是否离开即停止。首版建议主动请求停止，完成清理后卸载。
- ViewModel onCleared：设置 disposeRequested，有任务先请求取消，等待终态后关闭 Conversation，再关闭已初始化 Engine，最后关闭执行队列。
- 清理不能依赖已取消的 viewModelScope；控制器自有队列在释放前继续工作。加载中收到 dispose，加载返回后立即释放，不接新任务。
- 进程被杀：内存对象不会恢复。持久化记录任务 Interrupted，启动后显示“上次任务中断”，重新加载，不能静默重发副作用操作。

## 7. 两个只读工具：先不接模型独立验证

### 7.1 固定数据

DemoLogRepository 只保存三份虚构日志，文件 ID 与内容由程序预置：

| ID | 示例内容 |
| --- | --- |
| anr_demo | ActivityManager: ANR in com.example.demo；Reason: Input dispatching timed out |
| crash_demo | AndroidRuntime: FATAL EXCEPTION: main；NullPointerException in DemoActivity |
| normal_demo | ActivityTaskManager: Displayed com.example.demo/.MainActivity |

为每条记录加行号，例如 anr_demo:L1/L2；它是输出证据定位的依据。模型第一次只看到工具说明，不预先注入所有日志正文，才能检查是否真的调用了读取工具。

### 7.2 工具约定

```text
listLogs() → {ok:true, logs:[{id,type}, ...]}
readLog(id) → {ok:true, id, text, truncated:false}
未知 id → {ok:false, error:"NOT_FOUND"}
```

ToolDispatcher 只接受这两个名称。readLog 只接受固定 ID 白名单，不接任意路径、命令、URI。工具返回 JSON 使用 JSONObject/Gson 等序列化，不用字符串拼接日志正文。

1. UI 加两个调试按钮：列日志、读 anr_demo。
2. 校验正常列表、正常内容、未知 ID、空 ID、过长 ID。
3. 工具正文设上限，例如 8 KB 作为实验预算；截断时显式标注 truncated，不让模型误以为看到全量。
4. 保存 requestId、toolName、校验结果、开始/结束时间；只记录必要样例信息。
5. 所有工具行为来自真实程序调用；不让模型编造工具结果代替执行。

## 8. 接入 Agent 工具循环

### 8.1 先确认工具模型

普通聊天模型不自动支持原生工具调用。官方 Kotlin 指南以 FunctionGemma 为工具模型例子，但原始模型权重不能直接作为 LiteRT-LM 文件加载。

步骤：找到官方/官方链接社区提供的兼容工具模型包 → 核对模型卡、工具模板、支持后端和 SDK 版本 → 完成模型自己的工具示例 → 再替换为日志工具。记录文件和许可。若未找到与当前环境兼容的包，先采用 8.4 的受控 JSON 协议练循环，不宣称原生工具能力已验证。

### 8.2 声明与手动执行

为 listLogs/readLog 分别实现固定版本 SDK 的 OpenApiTool，getToolDescriptionJsonString 返回名称、描述、parameters/object/properties/required；工具调用的 execute 或 ToolDispatcher 只能访问预置数据。

ConversationConfig 注册 tools，并设 automaticToolCalling = false。教学阶段由宿主决定是否执行，便于观察验证、步数限制与停止。模型拿到工具说明后不代表工具自动有权限。

各版本 ToolSet/AutoToolSet 注解 API 可能变化，先用固定版本支持的 OpenApiTool 路径，别混用网页与 main 示例。

### 8.3 循环步骤

1. 设置指令：先获取日志列表；需要正文时读取；只依据工具返回内容回答；区分事实、候选根因、缺失证据；引用日志 ID 与行号；没有证据就说明不足。
2. 初始化 taskId、轮数 0、工具调用数 0、总时限。建议起步上限：6 轮模型交互、4 次工具执行、60 秒墙钟预算，按模型速度调整。
3. 将用户请求发送给同一 Conversation。
4. 每轮完整收到响应后检查 toolCalls。流式中不要根据未收完整的参数片段提前执行。
5. 没有 toolCalls：将文本作为最终答复候选，检查是否有证据，再结束。没有读正文却生成具体日志根因，应记为任务失败，不伪造一次读取补齐轨迹。
6. 有 toolCalls：逐个校验名称、参数类型、ID、重复调用、预算与取消状态。未知工具或非法参数不执行，返回受控错误结果并计入预算。
7. 执行工具，记录真实结果，并将结果作为工具消息回传同一个 Conversation。SDK 支持的 Content.ToolResponse/Message.tool 结构按实际版本构造，不能改成伪装用户消息混淆角色。
8. 继续下一轮，直到最终回答、取消、失败或预算耗尽。每轮和每次工具前都检查预算；计时超限时调用底层取消，不仅仅退出 UI。
9. SDK 默认自动调用也有内部循环上限，但不能替代自己的业务工具预算。

算法示意，非可直接编译代码：

```text
response = model(userRequest)
repeat while rounds < limit:
    if cancelled or deadline exceeded: cancel and stop
    if response has no tool calls: validate evidence; finish
    for each complete tool call:
        validate name + parameters + budget
        result = dispatch or controlled error
        append trace
    response = model(toolResults)
budget exhausted -> explain incomplete task and preserve trace
```

### 8.4 工具模型暂不可用时的学习替代

定义只允许两个 JSON 动作的提示协议：

```json
{"kind":"tool","name":"readLog","arguments":{"id":"anr_demo"}}
```

最终答复使用：

```json
{"kind":"final","answer":"...","evidence":["anr_demo:L1"]}
```

整个模型回复必须解析为一个 JSON 对象；拒绝未知 kind/字段、非法类型、超长内容和代码块包裹。最多允许一次格式纠正提示，然后失败；不正则截取任意大括号执行，不把模型文本作为 Shell。

把真实工具结果序列化为下一轮上下文并说明其为数据。日志中即使含“忽略规则/执行命令”也不赋予指令权限。模型选错、JSON 不合法均属于被评测的失败。

这是宿主约束的文本协议实验，不等同于模型原生 function calling 或严格约束解码；可帮助理解循环，但需要单独报告可靠性。

### 8.5 首次成功用例

输入：“先列出日志，再读取 ANR 示例，说明事实、不能确认的根因和下一步证据。”

期望轨迹：listLogs → readLog(anr_demo) → 输出包含 ANR/输入分发超时事实与行号，并说明缺少主线程栈/相关线程与时间段证据。工具顺序可因任务变化，但不得伪造未调用工具的数据。

验收：工具选择和结果回传可见，最终引用实际证据；只产生漂亮文本不算工具闭环通过。

## 9. 推理迁移到独立进程与 AIDL

单进程闭环通过后再拆。它是普通 App 内服务化练习，不是复刻 AICore 或新增 system_server 服务。

### 9.1 服务所有权

InferenceService 持有 InferenceController 与模型句柄；UI 进程不再同时加载模型。首版同 APK、同 UID、exported=false，不使用 isolatedProcess，以免丢失文件访问条件。

```xml
<service
    android:name=".InferenceService"
    android:process=":inference"
    android:exported="false" />
```

Gradle android.buildFeatures 开启 aidl = true。在 app/src/main/aidl/com/example/androidlogagent/ipc/ 下创建接口，包名与目录一致。

### 9.2 接口约定

IInferenceCallback.aidl：

```aidl
package com.example.androidlogagent.ipc;
oneway interface IInferenceCallback {
    void onState(String requestId, String state);
    void onChunk(String requestId, String text);
    void onTerminal(String requestId, String status, String error);
}
```

IInferenceService.aidl：

```aidl
package com.example.androidlogagent.ipc;
import com.example.androidlogagent.ipc.IInferenceCallback;
interface IInferenceService {
    void load(String requestId, String modelId, String backend,
              IInferenceCallback callback);
    void generate(String requestId, String prompt,
                  IInferenceCallback callback);
    void cancel(String requestId);
    void unload();
}
```

这些是本学习工程自定义接口，不是 Google SDK 接口。load 接受 modelId，由服务映射到固定路径；禁止客户端任意传路径。backend 只允许 CPU/GPU。方法入口快速校验并入队，不能占 Binder 线程做加载/推理。

### 9.3 实现步骤

1. Service.onCreate 初始化队列；onBind 返回 IInferenceService.Stub。
2. Stub 各方法检查输入、当前状态及 requestId，随后投递控制器。跨进程调用可能并发，所有状态由队列归并。
3. UI 用显式 Intent + bindService(BIND_AUTO_CREATE)，onServiceConnected 后获取 Stub.asInterface。
4. 客户端的 load/generate/unload 调用也放后台，避免同步 Binder 调用或异常拖住主线程。请求成功入队不等于加载或推理完成，等待 callback。
5. 对 Binder linkToDeath；处理 onServiceDisconnected、onBindingDied，清空旧代理，把任务记为 Interrupted。onBindingDied 按系统要求解绑并重新绑定，避免累积连接。
6. 服务对客户端 callback binder 监听死亡；客户端消失时取消其活跃任务。RemoteException 不在推理回调里无限重试。
7. chunk 合批发送，限制单条长度，例如 4 KB 的文本目标并按实际编码考虑大小；不跨 Binder 传模型或整份 bugreport。
8. 成功结束、失败、取消，每个 requestId 恰好一个 terminal。迟到回调按 ID 丢弃。
9. UI 退出时显式取消并按策略卸载、解绑。Service.onDestroy 尽力清理，但进程被杀不保证收到 onDestroy；应用应容忍重启。
10. App 启动代码可能在 :inference 进程执行，避免 Application.onCreate 重复初始化 UI 或再次加载模型。

### 9.4 故障实验

```powershell
adb shell ps -A
```

找到准确的 com.example.androidlogagent:inference PID；仅在开发 ROM 允许时针对该 PID 执行 kill，权限不足不要当作服务测试失败。预期 UI 检测死亡、标记中断、重新绑定，用户主动重新加载后可继续。不要使用整包 force-stop 来声称已验证“仅推理进程死亡”。

此阶段先用 bound service、保持实验前台；需要后台长任务时另行设计前台服务和平台限制，不承诺 bound service 永久存活。

## 10. 状态持久化、历史与恢复

先只持久化任务概要：taskId、用户输入摘要、模型/后端、状态、时间、工具名/参数摘要/错误、证据 ID、最终输出。可先调试 JSONL，再按需要使用 Room。

- 进程启动扫描未结束任务，改为 Interrupted，不假设 KV Cache 能恢复。
- 对话历史、工具轨迹、模型缓存分开处理；本阶段每任务新会话，避免无限历史膨胀。
- 只读任务可让用户明确点击重试；后续创建待办/修改设置等写操作必须加操作 ID、确认和执行结果查询，不能直接重放。
- 仅使用虚构日志即可完成学习。接真实 bugreport 前需确认授权、脱敏与留存范围。

## 11. 评测与性能记录

### 11.1 固定验收集

建议先建 20 条正常/异常用例，不是统计充分性的承诺：

| 类别 | 示例 | 通过标准 |
| --- | --- | --- |
| 事实提取 | 读取 anr_demo | 引用正确内容，不断言未知根因 |
| 工具选择 | 只问文件列表 | 调 listLogs，不无关读取 |
| 非法参数 | 不存在 ID、路径穿越 | 不执行越界读取，给受控错误 |
| 证据不足 | 日志缺主线程栈 | 明确不足与下一步信息 |
| 停止 | 生成中点停止 | 到终态再允许新任务 |
| 重复输入 | 连点发送 | 仅一个活跃任务 |
| 生命周期 | 旋转、页面重建 | 不重复加载/启动请求 |
| 模型失败 | 文件缺失或损坏 | 原因可见，不误报成功 |
| GPU 失败 | 后端不支持 | 明确失败，受控 CPU 对照 |
| 进程死亡 | 仅推理进程退出 | UI 标记中断并能重连 |
| 提示注入 | 日志含操作命令 | 当作数据，不执行日志指令 |
| 预算 | 模型反复读同一日志 | 按预算结束并说明未完成 |

把期望轨迹、事实与可接受变体写下来。对比“手动把日志贴给模型”的基线，不能将所有改进归因于 Agent。

### 11.2 指标

使用 SystemClock.elapsedRealtime，分别记加载时间、请求到首个非空输出 TTFT、请求总耗时、取消请求到终态耗时。没有 Token 计数时不报告 tokens/s，也不把回调次数作为 Token 数。

记录任务成功率、非法工具执行次数、证据引用正确率、预算超限次数。任务成功率分母只用定义好的固定任务，失败也保留。

```powershell
adb shell dumpsys meminfo com.example.androidlogagent
adb shell dumpsys meminfo com.example.androidlogagent:inference
adb shell dumpsys thermalservice
adb logcat -v threadtime > .\androidlogagent-logcat.txt
```

最后一条先启动再复现，Ctrl+C 结束。PID 过滤可能漏掉 lmkd/驱动事件，故故障现场先保存完整本地日志，再提取相关段落脱敏分享。先取日志，不默认 logcat -c 清除已有证据。

同设备、同模型、同输入和输出上限，对冷/热运行分开记录；五次短运行可作初步观察，不足以稳定估计 P95，样本不足时直接报告各次值或中位数。

持续温升与能耗实验需固定屏幕、充电、环境和负载；不要从单次 PSS 或温度读数推断完整能耗结论。

## 12. 排障决策表

| 问题 | 下一步 |
| --- | --- |
| Gradle 找不到依赖 | google()、网络、坐标、实际发布版本；不猜 release 与 Maven 等价 |
| 编译 API 不匹配 | 查固定版本 sources，统一示例版本 |
| 文件不存在/权限错误 | 包名路径、真实文件、模型长度，必要时 SAF 导入 |
| 格式/Tokenizer 错误 | 检查是否 .litertlm 兼容资产、HTML/指针误下载、后端专用包 |
| GPU 初始化失败 | Manifest、GPU/OpenCL 驱动、ROM、固定运行时；用支持的 CPU 对照 |
| native crash/内存压力 | 短上下文、小模型、加载数量、回调清理竞态、lmkd/tombstone |
| 停止按钮只停 UI | 显式取消、终态回调、SDK 版本行为；不提前 close |
| 工具不调用 | 核对模型工具能力、模板、schema、注册和实际轨迹 |
| 工具反复调用 | 检查结果回传角色、是否同一会话、预算与错误表达 |
| 引用编造 | 证据白名单/行号校验，任务标失败并保留样本 |
| AIDL 卡顿 | Stub 是否阻塞、回调是否大包/过频、主线程 Binder 调用 |
| SELinux denial | 具体 AVC/路径/域定位，不把全局 permissive 当默认修复 |

## 13. 最终交付与下一阶段

形成可复现目录：README、版本记录、架构图、debug APK、模型获取说明（遵守许可，不随意分发权重）、20 条用例、效果记录、失败复盘和两分钟演示。源码不包含真实客户日志或账号凭据。

演示建议：加载 → 发日志调查请求 → 展示工具轨迹与证据 → 停止 → 再次请求 → 服务死亡恢复。说明哪些是实测、哪些未支持，不能将虚构日志实验包装为实际根因自动诊断成果。

完成后再选一个分支：系统集成/资源调度继续 AIDL、队列、内存与权限；Agent 应用继续真实授权案例和效果评测；推理优化再进入 NPU、量化与后端适配。当前主线不要求同时完成所有分支。

## 14. 资料、核验边界与关联

2026-11-14 查阅：

- [LiteRT-LM Android/Kotlin](https://developers.google.com/edge/litert-lm/android)：API、模型与工具入口。
- [LiteRT-LM release](https://github.com/google-ai-edge/LiteRT-LM/releases/latest)：本次指向 v0.19.1，不代替 Maven 发布核验。
- [Conversation 源码](https://github.com/google-ai-edge/LiteRT-LM/blob/main/kotlin/java/com/google/ai/edge/litertlm/Conversation.kt)：取消与回调；使用时查所固定版本。
- [Tool 固定 tag 示例](https://github.com/google-ai-edge/LiteRT-LM/blob/v0.19.1/kotlin/java/com/google/ai/edge/litertlm/Tool.kt)：工具 API 版本对照。
- [Gemma3-1B-IT 模型列表](https://huggingface.co/litert-community/Gemma3-1B-IT/tree/main)：许可、实际文件与后端变体。
- [Android AIDL](https://developer.android.com/develop/background-work/services/aidl)：接口、线程与绑定。
- [Gallery 发布](https://github.com/google-ai-edge/gallery/releases/latest)。

工具模型和 NPU 的特定可用包尚未在用户真机验证；代码片段未编译。当前工作区为笔记库，未发现 Android 工程；本轮未安装 SDK、下载模型或运行 adb 设备操作。

关联：[[agent开发/端侧终端Agent转型/索引|专题索引]]、[[Google端侧大模型方案与实现流程]]、[[LiteRT-LM的Kotlin最小推理实验]]、[[骁龙8Gen3运行GoogleAIEdgeGallery操作手册]]。

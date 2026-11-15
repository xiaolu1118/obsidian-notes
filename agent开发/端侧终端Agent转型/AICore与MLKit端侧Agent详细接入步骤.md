---
created: 2026-11-15
updated: 2026-11-15
tags: [AICore, MLKit, GeminiNano, AppFunctions, Android]
status: 公开文档核验与实施指南，未真机验证
---

# AICore 与 ML Kit 端侧 Agent 详细接入步骤

## 0. 适用范围与当前设备结论

用户明确要求 AICore 方案的详细步骤，非继续替换为 LiteRT-LM。本文单独记录官方系统托管路线：应用 → ML Kit GenAI → AICore → Gemini Nano → 设备硬件。

用户设备已确认骁龙 8 Gen 3、12 GB、AOSP 16、bootloader 已解锁。官方 Direct model access 文档明确不支持解锁设备。因此现有设备可用于编写/编译客户端、假接口与 UI 测试，但不能作为本路线的受支持真机验证环境；芯片与 RAM 足够不等于资格满足。

本教程描述开发者在已支持设备上的应用接入。将 AICore 集成进自编 AOSP/量产 ROM 是不同任务：公开材料未提供一套任意 AOSP 可复现的完整部署流程，不能用安装 APK、搬运权重或自行写 Binder 服务代替系统组件、设备适配与服务端资格。具体 OEM 集成需对应正式合作资料。此为公开信息边界，不推断厂商内部实现。

不要求当前用户立即换机或刷机；先借用/使用官方支持的原厂锁定设备验证即可。重新锁定自定义 AOSP 不是通用修复，本指南不包含刷机/锁定操作。

## 1. 明确各层职责

| 组件 | 你做什么 | 平台做什么 |
| --- | --- | --- |
| App / ViewModel | UI、输入、任务、错误、授权和持久化 | 不替你定义业务规则 |
| ML Kit Direct model access | 提示、内容、输出验证，按需自行维护历史 | 调用端侧生成能力 |
| AgentRunner | 指令、工具、预算、确认、会话与效果评测 | 管理多轮与工具执行循环 |
| AICore | 检查支持与状态，使用公开 SDK | 模型托管、分发更新与推理管理 |
| AppFunctions | 声明自己的受控能力、校验输入与执行 | 类型化工具描述/调用机制 |

不用下载 Hugging Face .litertlm，不配置 EngineConfig，不手选 CPU/GPU/NPU；这些属于前面的自管运行时路线。云端 Gemini API Key 也不是本地 AICore 推理的接入条件。

## 2. 设备资格核验

1. 打开 [ML Kit GenAI 总览](https://developers.google.com/ml-kit/genai)，定位 Custom generation and agent device support；不要误看仅支持摘要/改写等 Task API 的列表。
2. 对照具体型号、系统/地区配置、内存规格和 AICore 更新状态，不仅看 SoC。
3. Agent 当前入门要求 Android 14/API 34 以上、受支持设备、AICore Developer Preview；项目 minSdk 26 以上、Kotlin 2.3 以上。minSdk 26 不表示 Android 8 真机能运行 Agent 预览。
4. 使用该设备受支持的原厂系统与 Google 服务环境、bootloader 锁定状态。锁定只是必要条件之一，不保证所有 API 可用。
5. 使用与预览报名一致的 Google 账号；准备 Wi-Fi 与足够存储空间。

可以用只读命令保存设备信息：

```powershell
adb devices -l
adb shell getprop ro.product.model
adb shell getprop ro.build.fingerprint
adb shell getprop ro.build.version.release
adb shell getprop ro.boot.flash.locked
adb shell pm path com.google.android.aicore
adb shell dumpsys package com.google.android.aicore > .\aicore-package.txt
```

boot 属性和包存在只用于辅助观察，不是资格证明；某些 ROM 的属性不适用或为空。官方运行时 checkStatus 才是功能状态入口。多设备用 adb -s 指定序列号。

## 3. 加入 AICore Developer Preview

依据 [官方预览说明](https://developers.google.com/ml-kit/genai/aicore-dev-preview)：

1. 电脑或手机浏览器登录测试设备使用的 Google 账号。
2. 加入 [aicore-experimental Google Group](https://groups.google.com/g/aicore-experimental)。
3. 打开 [Android AICore 测试计划](https://play.google.com/apps/testing/com.google.android.aicore)，同账号成为 tester。是否成功以该页面账户状态为准，工具无法读取登录后个人资格。
4. 到手机设置中的 Google 系统服务更新 AICore：Pixel 从设置头像进入，其他设备从 Google（服务）进入，再找 All services → System services → AICore。
5. 核对 listing 显示 Android AICore (Beta)，有更新时点击更新。官方指出标签生效可能约需一小时，别在尚未生效时反复清空 AICore。
6. 若 Beta 更新下载有问题，官方建议可清 Play Store 缓存后重试；不默认清 AICore 数据，后者会重新初始化配置和资产。
7. 更新完成出现 AICore 图标后打开，阅读并接受所需条款。
8. 如果账号或设备不支持，先停在资格问题，不往工程里添加更多依赖尝试绕过。

## 4. 先用 AICore 预览入口验证模型

1. 在 AICore 应用的模型选择器选择该设备提供的 Gemini Nano 版本。
2. Wi-Fi 下载模型，等待完成。已有稳定模型时可以按 UI 提供的选项先测试稳定版本；不保证每台设备都已预装。
3. 输入一个短问题，例如英文“Explain Android Binder in three short sentences”，先确认基础能力。
4. 再测试中文，实际记录语言表现，不把任务 API 的语言支持等同于任意生成接口的语言能力。
5. 首次推理包括加载，可能较慢；BUSY 时等待并减少并发，而不是高频重试。
6. 模型准备完成后断网、新输入再测试。只验证该文本推理路径；模型下载需要网络。
7. 记录型号、系统、AICore 版本、所选模型、结果和耗时。

此阶段通过后再开发自己的 APK。若官方入口尚不能生成，先查设备/账号/配置，避免同时引入 App 变量。

## 5. 新建 ML Kit 最小工程

### 5.1 创建和配置

Android Studio 新建 Kotlin Empty Activity，名称 AICorePlayground，包名 com.example.aicoreplayground。minSdk 可用 26，实际实验设备按上面的真机要求。保留模板 Compose、协程和 Lifecycle 依赖；后续 Agent 使用 Kotlin 2.3 以上并选择配套兼容的 AGP/KSP。

settings.gradle.kts 的 dependencyResolutionManagement.repositories 包含 google() 和 mavenCentral()。app/build.gradle.kts 添加当前官方 Direct model access 依赖：

```kotlin
dependencies {
    implementation("com.google.mlkit:genai-prompt:1.0.0-beta4")
}
```

本轮核验日期为 2026-11-15。版本变化时重新核验，不混用早期 Google AI Edge AICore 实验 SDK 与当前 ML Kit 代码。

不添加 LiteRT-LM Engine、OpenCL native library 声明、模型存储访问权限或自定义模型路径。本路线由系统管理模型。

### 5.2 最小界面

先做模型状态、检查按钮、下载按钮、输入框、生成按钮、输出区和错误区。状态未 AVAILABLE 时禁用生成；生成中禁止重复点击。所有操作经 ViewModel 的协程执行，UI 只显示状态；不要在 Compose 重组体里创建客户端和发请求。

### 5.3 构建、安装

```powershell
.\gradlew.bat :app:dependencyInsight --dependency genai-prompt --configuration debugRuntimeClasspath
.\gradlew.bat :app:assembleDebug
adb install -r .\app\build\outputs\apk\debug\app-debug.apk
```

本指南尚未创建/编译该工程；片段用于接入设计，不声称完整可运行 APK 已交付。先在 External Libraries 核对当前 SDK 的类、函数签名、错误和关闭方法。

## 6. 检查、下载、生成与流式

### 6.1 客户端和状态

在模型仓库层通过 Generation.getClient() 获得客户端，Generation 属于 com.google.mlkit.genai.prompt。FeatureStatus/DownloadStatus 的准确 import 以当前发布包为准，可由 IDE 导入，不从其他 Task API 复制错误类型。

```kotlin
val model = Generation.getClient()
val status = model.checkStatus()
```

| 状态 | 界面与程序行为 |
| --- | --- |
| UNAVAILABLE | 显示不支持或配置未就绪；进入资格/初始化排查，不生成 |
| DOWNLOADABLE | 显示下载按钮，由用户触发下载 |
| DOWNLOADING | 显示等待和可获得的进度，避免重复启动 |
| AVAILABLE | 允许短文本请求 |

状态查询也可能异常，单独捕获显示。不要把所有 UNAVAILABLE 都归因于“还没下载”。

### 6.2 下载

在 DOWNLOADABLE 分支收集 model.download()：

```kotlin
model.download().collect { event ->
    // DownloadStarted：进入下载状态
    // DownloadProgress：显示 totalBytesDownloaded
    // DownloadCompleted：重新 checkStatus，再决定是否可生成
    // DownloadFailed：记录 event.e，显示可解释的错误
}
```

只暴露字节数时不凭空显示百分比。下载失败与模型资格失败分开呈现。不要从非官方来源复制模型文件到 AICore 私有目录。

### 6.3 一次生成

在 AVAILABLE 且 App 顶部前台时执行：

```kotlin
val response = model.generateContent("Explain Android Binder in three sentences.")
val text = response.candidates.firstOrNull()?.text.orEmpty()
```

空候选/空文本不崩溃，保留响应与错误的脱敏诊断信息，不盲目自动重试。首次用短输入，之后按请求构建接口设置 maxOutputTokens 等参数，使用 countTokens/目标模型约束控制上下文，不直接上传整份 bugreport。

### 6.4 流式

```kotlin
val buffer = StringBuilder()
model.generateContentStream("Briefly explain Android ANR.").collect { chunk ->
    val delta = chunk.candidates.firstOrNull()?.text.orEmpty()
    buffer.append(delta)
    // 发布 buffer.toString() 给 UI；确认当前版本分片为增量
}
```

客户端放 ViewModel/仓库中按生命周期复用；用户取消、退出和关闭遵循该 ML Kit 版本 API。不要把 LiteRT-LM Conversation.cancelProcess 原样搬到这里，它不是同一个 SDK。

仅在模型资产已准备且业务确有需要时使用 warmup()，分开记录冷/热延迟；预热也占资源。

### 6.5 前台限制与重试

当前 ML Kit 总览明确推理只允许顶部前台应用，前台服务不能绕过该限制。实验保持 Activity 可见，不复制 LiteRT-LM 的独立后台推理服务设计来假设仍可运行。

- BUSY：有限次数指数退避，支持用户取消。
- PER_APP_BATTERY_USE_QUOTA_EXCEEDED：显示配额限制，停止循环重试。
- BACKGROUND_USE_BLOCKED：回到前台后由用户再发请求，别伪装后台常驻。

## 7. 从生成升级为 AgentRunner

Direct model access 已通过后才添加：

```kotlin
implementation("com.google.mlkit:genai-agent:1.0.0-alpha1")
```

这是本轮文档的预览依赖，非全机型量产保证。生成与 Agent 共享模型服务并不表示业务状态由系统自动维护。

在 ViewModel/明确所有者中创建：

```kotlin
val agent = Agent.builder()
    .setName("LogAssistant")
    .setInstructions("Use available evidence. Distinguish facts from guesses.")
    .setMaxTurns(6)
    .build()
val runner = AgentRunner(agent)
val session = runner.getOrCreateSession("log_demo_01")
```

Content.builder().text(userInput).build() 构建请求，再 session.runAsync(content) 收集事件。先无工具验证 Agent 会话；只看文本时可以 textTokens()，接工具时保留完整事件流。

会话 ID 遵循 SDK 约束，UUID 可用。旋转保留 ViewModel 的 session；进程死亡不保留内存会话。保存需要的已完成消息和任务概要，不把 sessionId 当作系统已持久化全部历史的证明。

## 8. 添加两个应用内 AppFunctions 工具

### 8.1 依赖与生成

按 Kotlin/AGP 兼容关系启用 KSP 插件；KSP 版本从对应官方发布选择，不直接等同 Kotlin 版本。添加文档当前示例：

```kotlin
implementation("androidx.appfunctions:appfunctions:1.0.0-alpha13")
ksp("androidx.appfunctions:appfunctions-compiler:1.0.0-alpha13")
```

先编译依赖，再实现类，检查 KSP 产物，避免“工具类写了但 schema/服务没生成”。

### 8.2 工具内容

第一版用三条固定虚构日志，沿用 list_logs()、read_log(id)，读操作只允许 anr_demo/crash_demo/normal_demo；返回结构含 ok、id、带行号正文、error 和 truncated。不接 shell、任意文件路径、真实系统日志或无障碍。

1. abstract class LogTools : AppFunctionService()。
2. 类上 AppFunctionServiceEntryPoint，指定 serviceName = "LogAppFunctionService"、appFunctionXmlFileName = "log_app_functions"。
3. 方法加 AppFunctionDeclaration(isDescribedByKDoc = true)，KDoc 描述工具用途与 @param。
4. 返回 primitive 或 AppFunctionSerializable 数据类，不把任意 Kotlin Map 当作已确认可生成的返回类型。
5. 工具加 AppFunctionAccessLevel，使用 AppFunctionMetadata.ACCESS_LEVEL_SELF，并按版本启用 compat enforcement/实验 opt-in；只供自己的 Agent 使用。
6. 先直接调用工具单测正常/未知 ID/空输入；预期错误返回受控结果，不返回设备敏感路径。

### 8.3 Manifest 注册生成服务

以下名称须与实际 KSP 输出一致，非手写同名空 Service：

```xml
<service
    android:name=".LogAppFunctionService"
    android:permission="android.permission.BIND_APP_FUNCTION_SERVICE"
    android:exported="true">
    <property android:name="android.app.appfunctions.schema"
        android:value="app_functions_schema.xsd" />
    <property android:name="android.app.appfunctions.v2"
        android:value="log_app_functions.xml" />
    <intent-filter>
        <action android:name="android.app.appfunctions.AppFunctionService" />
    </intent-filter>
</service>
```

exported=true 是当前官方服务注册形式，配合绑定权限与函数 SELF 访问限制；不能删掉权限后认为函数是私有的。API/元数据随 alpha 变化，按所固定版本核对。

### 8.4 注册到 Agent

两种方式选一种：

- 自动发现：enableAppFunctions(true)，按当前 API 用 addAppFunctionIds 限定实际生成 ID。
- 程序化发现：AppFunctionAgentTools.discover，检查/过滤/包装后 addTools。

不同时注册两遍，避免重复工具 schema。生成的工具 ID 以产物/发现结果为准，不假设永远等于简单方法名。发现结果为空时先查 KSP、manifest、生成 XML、名称和环境，不能让模型“自己发现未注册工具”。

## 9. 完整事件、确认、取消与停止

收集 session.runAsync(content) 的 Flow<AgentEvent>：

| 事件 | UI/记录 |
| --- | --- |
| TextMessage + isPartial=true | 追加文本增量 |
| TextMessage + isPartial=false | 用最终完整文本替换当前段，避免重复追加 |
| FunctionCall | 记录工具与参数，显示执行中 |
| FunctionResponse | 记录结果、更新工具状态 |
| 其他事件 | when 保留 else，不假定类型穷尽 |

不要同时 collect 一份 textTokens 又重复追加完整事件文本。事件输出仅用于观察，工具权限仍由执行层检查。

第一版只读；后续写入操作通过程序化工具包装加入确认、操作 ID 和幂等。确认弹窗由宿主实现，模型一句“用户已同意”不是授权证据。

同一 session 禁止并发 run。取消由该版本 Runner/Session 的协程及 session 生命周期契约处理；官方说明 session.close() 取消其在途协程并安排驱逐，runner.close() 释放其活跃会话。停止/退出期间不接受新任务；取消后对已执行工具查验结果，不能把关闭会话理解为回滚写入。独立拥有的 runner 在 ViewModel onCleared 关闭；共享 runner 不由单个会话页面随意关闭。

maxTurns 限制循环，同时增加业务总时限和工具预算。先用六轮、四次读取作学习目标，达到上限解释未完成，避免无限工具循环。不要套用 LiteRT-LM 原生句柄释放顺序作为 AICore SDK 的内部实现。

## 10. 第一次 Agent 验收

输入“先列出日志，再读取 ANR 示例，用事实、推测、缺失证据三个部分回答，并引用行号”。

必须观察到实际 FunctionCall/FunctionResponse，正文来自工具，最终引用真实行号；没有调用却直接回答具体日志内容不能算通过。

再验证：空输入、未知日志、工具失败、停止、重复发送、旋转、进后台、断网、进程重建和预算耗尽。记录每个用例、期望行为、实际结果及版本。

指标区分检查状态/模型准备耗时、首字延迟、请求总耗时、工具耗时、任务成功率和错误证据。AICore 在独立系统组件执行，App 的 PSS 不等于模型总内存；共享服务内存也不能简单全算给单一 App。

## 11. 排错顺序

| 现象 | 处理 |
| --- | --- |
| 当前解锁 AOSP 无法使用 | 按不受支持环境处理，先受支持真机验证，不继续搬 APK |
| UNAVAILABLE | 型号/ROM/账号预览/AICore 版本/配置与 bootloader 逐项核对 |
| 601 BINDING_FAILURE | 官方文档建议更新 AICore、必要时重装测试 App；先保留日志 |
| 606 FEATURE_NOT_FOUND | 可能配置未初始化，也可能不支持/已解锁；不是简单等下载即可 |
| 刚重置或刚装 AICore | 保持网络，等待配置初始化，必要时重启；避免不断清数据重置进度 |
| 下载 UNKNOWN/域名错误 | 网络、Wi-Fi、系统服务连通性与存储；和工具逻辑分开 |
| BUSY/电量配额 | 退避或暂停，避免刷请求 |
| BACKGROUND_USE_BLOCKED | 回顶部前台，不靠前台服务绕过 |
| AppFunctions 不可见 | KSP 产物、服务注册、schema、工具 ID、访问等级、SDK 兼容 |
| 文字重复 | 是否把最终完整文本再次追加 |

抓取日志，Ctrl+C 结束，分享前脱敏：

```powershell
adb logcat -v threadtime > .\aicore-playground-logcat.txt
adb shell dumpsys meminfo com.example.aicoreplayground
adb shell dumpsys package com.google.android.aicore > .\aicore-package.txt
```

## 12. 官方资料与完成状态

- [设备支持与限制](https://developers.google.com/ml-kit/genai)
- [预览报名与模型测试](https://developers.google.com/ml-kit/genai/aicore-dev-preview)
- [Direct model access](https://developers.google.com/ml-kit/genai/prompt/android/get-started)
- [AgentRunner 入门](https://developers.google.com/ml-kit/genai/agent/android/get-started)
- [AppFunctions 工具](https://developers.google.com/ml-kit/genai/agent/android/define-tools)
- [会话与生命周期](https://developers.google.com/ml-kit/genai/agent/android/manage-sessions)
- [事件与 UI](https://developers.google.com/ml-kit/genai/agent/android/streaming-events)

已完成：公开文档复核、AICore 独立教程和索引归档。未完成：用户设备资格变更、测试报名、模型下载、工程编译、真机生成或 Agent 验收。用户要求教程不代表同意换机、刷回原厂、锁 bootloader 或参加预览，本轮未执行这些操作。

关联：[[Google端侧大模型方案与实现流程]]、[[Google端侧Agent从零到交付完整实操流程]]、[[agent开发/端侧终端Agent转型/索引|专题索引]]。

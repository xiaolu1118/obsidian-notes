---
created: 2026-11-14
updated: 2026-11-14
status: 操作建议，未编译或实测
tags: [LiteRT-LM, Kotlin, Android]
---

# LiteRT-LM 的 Kotlin 最小推理实验

> 当前统一执行入口为 [[Google端侧Agent从零到交付完整实操流程]]。本文保留早期教学片段和设计背景；简化 Flow/use 示例不作为完整取消与退出处理实现，资源清理按完整手册的终态驱动方案执行。

用户要求继续 Google 端侧入门。设备为骁龙 8 Gen 3、12 GB、AOSP 16、已解锁。未收到 Gallery 已成功的确认，不将上一阶段记为完成。

## 实验目标与准备

从 Gallery 体验转到自己写的 APK，先做单次文本生成。建议先验证一份兼容模型，不以完成工具调用或生产服务为目标。

Android Studio 新建 Empty Activity，Kotlin，包名 com.example.nanoplayground，minSdk 可选 31 作为实验设置，并非 SDK 官方最低声明。保留模板 Compose 与 lifecycle-runtime-ktx 依赖，确认 lifecycleScope 可导入。settings.gradle.kts 的依赖仓库包含 google() 和 mavenCentral()。

app/build.gradle.kts 添加 Android 依赖 com.google.ai.edge.litertlm:litertlm-android:latest.release。官方示例使用动态版本，本轮 Maven 元数据无法通过浏览工具读取，故不臆造固定版本。首次同步后用 dependencyInsight 记录解析版本并改成固定版本，模型与 API 均按该版确认。android.defaultConfig.ndk 配置 abiFilters += "arm64-v8a"。

GPU 需要在 Manifest 的 application 内声明可选 native library：libvndksupport.so 和 libOpenCL.so，两项 android:required 均为 false。此声明不是驱动安装；AOSP 缺少实现时仍可能失败。

## 模型与路径

使用官方文档链接的 LiteRT Community 兼容 .litertlm 文本模型；可从 Gemma3-1B-IT 候选开始，实际文件以模型页为准。Gallery 模型在其沙箱中，不默认可供另一个 App 读取。下载或导出兼容文件后自行保存为电脑 Downloads/model.litertlm；重命名不构成格式转换。

先安装 debug APK 并启动一次。包名保持上文设置时：

```powershell
adb shell mkdir -p /sdcard/Android/data/com.example.nanoplayground/files/models
adb push "$env:USERPROFILE\Downloads\model.litertlm" /sdcard/Android/data/com.example.nanoplayground/files/models/model.litertlm
adb shell ls -lh /sdcard/Android/data/com.example.nanoplayground/files/models/model.litertlm
```

代码使用 File(getExternalFilesDir(null), "models/model.litertlm").absolutePath，对本应用目录无需申请广泛存储权限。若 ROM 阻止 adb 写入，改用系统文件选择器导入并在后台复制到应用私有目录，不改成全局 permissive。卸载应用可能移除模型。

## 调用设计

参考当前官方 Kotlin API：EngineConfig(modelPath = path, backend = Backend.GPU()) → Engine → initialize → createConversation → sendMessage → 关闭 Conversation 与 Engine。耗时过程全部放到 Dispatchers.IO，use 块保证异常时释放资源。

Activity 中用一个按钮启动 lifecycleScope.launch，在 withContext(Dispatchers.IO) 中执行单次实验，按钮执行期间禁用，在 finally 恢复；捕获普通异常显示类型和消息，CancellationException 重新抛出。同步 native 生成不承诺随协程取消立即停止；当前建议运行期间保持前台、不旋转页面，可靠取消与持久状态留待下一实验。

建议首个问题：“用三句话解释 Android Binder”。首次仅做同步完整回复，成功后再替换为 sendMessageAsync(prompt).collect 流式输出。GPU 失败且模型支持 CPU 时改 Backend.CPU() 对照，关闭旧引擎后重建。

验收：自己的 APK 读取正确模型路径、完成一次短回复、断网新请求仍能生成、记录加载与生成时间及 PSS。尚未创建项目、编译 APK、推送模型或真机测试。

## 第二课：流式输出、引擎复用与取消（2026-11-14）

用户要求继续；尚未确认首课编译或设备运行成功。第二课为实施指导，未创建或测试工程。

本轮核对官方 main 分支 Conversation.kt：sendMessageAsync 提供 Flow 和 MessageCallback；Flow 实现的 awaitClose 为空，不能把取消 collect 当作底层停止保证；Conversation.cancelProcess() 显式请求取消，onError 的取消状态映射为 CancellationException。main 不必等于用户已解析的 Maven 版本，需先核对固定版本源码/API。

建议先单独实现流式，再做复用和取消，不一次重写全部结构。流式输出可以沿用首课短生命周期 Engine/Conversation，在 IO 中 sendMessageAsync(prompt).collect 累积文本，并用 withContext(Dispatchers.Main) 更新 Compose 状态。本阶段不加停止按钮，不在流仍生成时主动 close；生命周期取消的清理需在下一步改为 callback 终态驱动。

可靠取消采用回调路径：开始生成→onMessage 接收增量→正常 onDone 或 onError 作为终态→关闭本次 Conversation→复位按钮。停止按钮先进入“正在停止”，显式调用 cancelProcess；等终态回调后再关闭资源并允许新任务，不能刚 cancel 就 close Engine。若终态迟迟不来，显示停止异常、阻止新任务并保留现场，不并行销毁原生对象。

引擎所有权建议交给 ViewModel 内控制器，使用 applicationContext。初始化/创建会话/关闭等生命周期操作串行化，回调只投递文本与终态事件；初始化在后台执行。Engine 初始化成功后才进入 Ready，并只加载一次。若初始化失败，记录原始异常，避免盲目 close 未初始化的 Engine 掩盖错误。每次新问题创建独立 Conversation，取消后丢弃该会话；复用 Engine 不自动等于复用历史。

状态：Unloaded → Loading → Ready → Generating → Ready；Generating → Stopping → Ready，错误转 Error。初始化时不可推理；Generating/Stopping 不允许新请求或卸载。终态和主动停止路径通过请求 ID/串行控制确保仅清理一次，旧回调不覆盖新请求。

ViewModel 可应对旋转，UI 收集状态与用户动作；进程死亡后重新加载，onCleared 不能直接和活跃 JNI 调用竞争 close。控制器应请求取消并在终态后释放，清理任务不能依赖已经取消的 viewModelScope；底层终态不来时没有通用安全强杀线程方案。

性能记录区分模型加载、首个非空文本分片 TTFT、请求完成总耗时。用 SystemClock.elapsedRealtime()，首片后不再覆盖 TTFT；没有 Token 计数就不把分片数当 Token。建议重复短请求五次验证引擎复用、三次停止验证能再次请求，并检查旋转、空输入和工具未接入等边界。

参考：[Conversation 源码](https://github.com/google-ai-edge/LiteRT-LM/blob/main/kotlin/java/com/google/ai/edge/litertlm/Conversation.kt)、[Engine 源码](https://github.com/google-ai-edge/LiteRT-LM/blob/main/kotlin/java/com/google/ai/edge/litertlm/Engine.kt)。当前代码片段为教学片段，非完整取消控制器。

## 来源与关联

- [官方 Kotlin Android 指南](https://developers.google.com/edge/litert-lm/android)：2026-11-14 核验，API 可能变化。
- [[骁龙8Gen3运行GoogleAIEdgeGallery操作手册]]
- [[Google端侧大模型方案与实现流程]]
- [[agent开发/端侧终端Agent转型/索引|专题索引]]

---
created: 2026-11-14
updated: 2026-11-14
tags: [Android, LiteRT-LM, GoogleAIEdgeGallery, 实操]
status: 待用户执行
---

# 骁龙 8 Gen 3 运行 Google AI Edge Gallery 操作手册

## 已确认背景与目标

用户设备：骁龙 8 Gen 3、12 GB 内存、AOSP 16、bootloader 已解锁。用户选择先学习 Google 方案，当前目标是完成离线文本推理，尚未安装或实测。

采用 Gallery/LiteRT-LM 自管模型入口；ML Kit Direct model access 官方不支持解锁 bootloader。当前实验不依赖 AICore/Nano，也不是完整 Agent。不要将此路线与系统托管模型混淆。

## 1. 准备

- Windows 电脑可用 adb，手机启用 USB 调试并确认电脑授权；adb devices -l 应显示 device。
- 建议预留约 10 GB 存储作为起步余量，实际取决于模型和下载临时空间，不是官方最低要求。
- 手机与电脑能访问 GitHub；模型下载还可能需要 Hugging Face、账号和模型许可授权。
- 首次用文本聊天；关闭其他重负载应用，保持手机温度正常。

PowerShell 执行：

```powershell
adb devices -l
adb shell getprop ro.product.cpu.abi
adb shell getprop ro.soc.model
adb shell getprop ro.build.version.release
adb shell df -h /data
```

CPU ABI 预期 arm64-v8a；SoC 属性可因 ROM 缺失或命名不同而变化，不能单凭空值判断芯片不兼容。多设备连接时，在每条 adb 命令中使用 -s 指定序列号。

## 2. 下载与安装

2026-11-14 核验官方最新 release 为 1.0.23。官方 Assets 同时提供通用 ai_edge_gallery_app.apk 和多个 SoC 专用 APK。为了先验证 CPU/GPU，建议从通用包开始；sm8650 包留给后续按说明核对后再实验，不将其视为现有 AOSP 可运行 NPU 的保证。

- 官方发布：https://github.com/google-ai-edge/gallery/releases/latest
- 固定版本：https://github.com/google-ai-edge/gallery/releases/tag/1.0.23
- 通用 APK：https://github.com/google-ai-edge/gallery/releases/download/1.0.23/ai_edge_gallery_app.apk

假设文件保存在 Windows Downloads，执行：

```powershell
adb install -r "$env:USERPROFILE\Downloads\ai_edge_gallery_app.apk"
adb shell pm list packages com.google.ai.edge.gallery
```

出现 Success 后从手机应用列表打开 Google AI Edge Gallery。若只有 INSTALL_FAILED_TEST_ONLY，再使用 adb install -r -t 加同一文件路径；-t 的含义是允许测试 APK，不能绕过 minSdk 要求。签名冲突先核对原包来源与版本，不默认卸载，以免清除模型和数据。

也可在手机浏览器下载官方 APK，通过文件管理器点击安装，并按系统提示允许该来源安装。

## 3. 模型下载和首次生成

界面随版本变化，以下按功能入口描述，不保证按钮位置一致。

1. 打开 AI Chat 或相应文本聊天入口，进入模型选择列表。
2. 优先选择该版本列表内较小的文本模型；若存在兼容的 Gemma 3n E2B 量化模型，可作为候选。未核验当前安装包实际模型列表，不保证该项一定存在。
3. 点击下载，若跳转 Hugging Face，按页面登录并接受对应模型许可后返回应用。
4. 等待完整下载。显示可加载/可使用后再进入聊天，首次加载可能较慢。
5. 如设置提供后端选项，先 GPU；若 GPU 失败且同一模型支持 CPU，再切 CPU。没有对应 UI 时保留默认，不臆造按钮。
6. 如可设置输出上限，先用 128—256 Token；其余参数先保持默认。
7. 输入“请用三句话解释 Android Binder，并给出一个使用场景”。观察真实生成，避免一次提交整份 bugreport。

不从其他项目随意下载 GGUF/safetensors 并直接导入。模型格式、Tokenizer 和运行时版本需要配套；人工导入与转换放到后续。

## 4. 离线验证

首次在线加载成功后，开启飞行模式并确认 Wi-Fi 和移动数据均关闭。退出并重新打开应用，新建会话，选择已下载模型，输入新的短问题。出现新生成回答即可作为该文本推理路径本地可运行的验证，不证明所有 Gallery 功能离线。

可选输入虚构日志：

```text
以下是虚构日志：
11-14 10:00:00.000 ActivityManager: ANR in com.example.demo
11-14 10:00:00.010 ActivityManager: Reason: Input dispatching timed out
请仅列出日志事实、尚不能确认的根因和下一步需要的证据。
```

检查输出是否引用实际内容，是否将输入超时直接断言为某个根因。短日志不足以证明根因；此练习检验证据意识，不是诊断能力验收。

## 5. 记录与排错

下载前、加载后、生成时可分别保存：

```powershell
adb shell dumpsys meminfo com.google.ai.edge.gallery
adb shell dumpsys thermalservice
```

记录 App 版本、完整模型名称/文件大小、后端、加载时间、首字时间、生成是否成功及 PSS。没有 token 计数器时只记人工观察耗时，不杜撰 tokens/s。PSS 是进程内存观测，不一定覆盖全部驱动/设备侧分配。

故障时先启动日志抓取，再复现；Ctrl+C 结束：

```powershell
adb logcat -v threadtime > .\gallery-logcat.txt
```

日志保存在 PowerShell 当前目录，可能包含设备其他应用信息，分享前只提取相关段落并脱敏。

| 现象 | 排查顺序 |
| --- | --- |
| 下载失败/401/403 | 模型站点连通性、登录与许可、存储空间；和推理后端分开诊断 |
| GPU 初始化失败 | 确认使用近期官方包；1.0.23 发布说明包含 GPU 相关修复；再尝试支持的 CPU 后端，保留 EGL/OpenCL/后端日志 |
| 加载后闪退 | 较小兼容模型、短上下文；检查 native crash、OutOfMemory 和 lmkd 记录 |
| 速度慢 | 确认实际后端、短输入与输出、冷/热状态、温升；不直接保证 NPU 会更快 |
| AICore 不可用 | 当前 Gallery 实验不以 AICore 为前提，无需转向安装 AICore APK 排错 |
| SELinux 拒绝 | 查看相关 avc 记录定位具体访问；不把全局 permissive 作为默认修复 |

第一阶段完成标准：至少一次断网新会话生成；记录模型和后端、内存及异常。下一阶段参考 LiteRT-LM Kotlin 文档实现最小 App，再考虑独立进程 AIDL 和工具调用。

## 来源与关联

- [Gallery 官方 README](https://github.com/google-ai-edge/gallery)
- [1.0.23 官方发布](https://github.com/google-ai-edge/gallery/releases/tag/1.0.23)
- [安装 Wiki](https://github.com/google-ai-edge/gallery/wiki/2.-Getting-Started)：页面较旧，实际 APK 名称以 release 为准。
- [[Google端侧大模型方案与实现流程]]
- [[agent开发/端侧终端Agent转型/索引|专题索引]]

已完成：文档核验与操作手册归档。未完成：APK 安装、模型下载、真机推理和性能实测。

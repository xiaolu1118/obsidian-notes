---
created: 2026-10-24
updated: 2026-11-15
tags: [学习资源, Agent, Android, 教程]
---

# Agent 与 Android 端侧 AI 教程清单

> 当前课序见 [[Agent学习执行总表与视频导航]]，逐课实践见 [[agent开发/端侧终端Agent转型/手机状态Agent手把手实战]]。2026-11-15 更新：以下新增完整资源编号，旧教程保留参考，不再把 Python/RAG 大课作为手机工具实验前置。

## 当前必修资源 R01—R21

“本次可读”仅表示网页可访问并核对主题，不表示样例已编译、设备已验证或整门课已学习。文档可切换中文时优先中文；SDK API 以实际固定版本为准。注册/模型/API费用按实际服务确定，本轮未购买课程或申请账号。

| 编号 | 资源与直达链接 | 阅读范围与本课作用 | 形式/核验 |
| --- | --- | --- | --- |
| R01 | [HF Agents Course 中文](https://huggingface.co/learn/agents-course/zh-CN/unit0/introduction) | 先入门单元与 Agent 基础，不急着全学多个框架 | 中文课程，本次入口可读 |
| R02 | [Building effective agents](https://www.anthropic.com/research/building-effective-agents) | 工作流、Agent、工具设计；写职责图 | 英文文章，本次可读；原文发布较早，概念参考，接口另查 |
| R03 | [MemoryInfo](https://developer.android.com/reference/android/app/ActivityManager.MemoryInfo) / [getMemoryInfo](https://developer.android.com/reference/android/app/ActivityManager#getMemoryInfo(android.app.ActivityManager.MemoryInfo)) | totalMem/availMem/lowMemory 与真实内存工具 | 官方 API，本轮前序已核对 |
| R04 | [Hello-Agents 源码教程](https://github.com/datawhalechina/hello-agents) | 定位工具、执行循环、错误与实验，不完整重学全部项目 | 中文教程仓库，本次可读；[站点](https://hello-agents.datawhale.cc/)仅返回前端壳，以仓库替代 |
| R05 | [Settings](https://developer.android.com/reference/android/provider/Settings#ACTION_DISPLAY_SETTINGS) / [Build](https://developer.android.com/reference/android/os/Build) | 公开设置入口与设备信息，分清打开页面与修改配置 | Settings 本次可读；Build 为补充参考，未逐字段复核 |
| R06 | [ML Kit GenAI 总览](https://developers.google.com/ml-kit/genai) | 支持设备、模型版本、前台与配额限制 | 官方文档，本次可读 |
| R07 | [AICore 预览](https://developers.google.com/ml-kit/genai/aicore-dev-preview) / [Direct access](https://developers.google.com/ml-kit/genai/prompt/android/get-started) | 资格、下载、状态、生成 | 官方文档，本轮前序已核对 |
| R08 | [Gallery](https://github.com/google-ai-edge/gallery) / [发布页](https://github.com/google-ai-edge/gallery/releases/latest) | 自管模型真机体验、后端与模型记录 | 官方仓库，本次仓库可读；版本安装详见操作手册 |
| R09 | [LiteRT-LM Kotlin](https://developers.google.com/edge/litert-lm/android) / [源码](https://github.com/google-ai-edge/LiteRT-LM) | Engine/Conversation、流式、工具、关闭与版本 | 官方文档，本次 Android 指南可读 |
| R10 | [AgentRunner](https://developers.google.com/ml-kit/genai/agent/android/get-started) / [工具与 AppFunctions](https://developers.google.com/ml-kit/genai/agent/android/define-tools) | AICore 分支的工具声明、发现、SELF 访问、确认 | 工具页本次可读；入门页本轮前序核对 |
| R11 | [会话生命周期](https://developers.google.com/ml-kit/genai/agent/android/manage-sessions) / [事件](https://developers.google.com/ml-kit/genai/agent/android/streaming-events) | session、工具事件、取消与文本增量/完整值 | 官方文档，本轮前序已核对；只用于 ML Kit 分支 |
| R12 | [Android 协程](https://developer.android.com/kotlin/coroutines) | suspend、Dispatcher、结构化并发与异常 | 官方文档，本次可读 |
| R13 | [Flow](https://developer.android.com/kotlin/flow) | 数据流、收集、UI 更新与取消的边界 | 官方文档，本次可读 |
| R14 | [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel) | 配置变化、状态所有权；不等于进程持久化 | 官方文档，本次可读 |
| R15 | [Room](https://developer.android.com/training/data-storage/room) | 实体/DAO/数据库，保存任务和执行状态 | 官方文档，本次可读 |
| R16 | [AIDL](https://developer.android.com/develop/background-work/services/aidl) | 接口、Stub、并发、回调与生命周期 | 官方文档，本次可读 |
| R17 | [[agent开发/端侧终端Agent转型/Google端侧Agent从零到交付完整实操流程]] | 虚构日志工具、预算、证据和故障练习 | 本地自编教材，未编译/真机验收 |
| R18 | [Perfetto](https://perfetto.dev/docs/) | Getting Started、系统跟踪、内存/CPU 与时间线 | 官方文档，本次可读 |
| R19 | [[agent开发/端侧终端Agent转型/手机状态Agent手把手实战]] | 固定用例、预期结果、失败分类与测量口径 | 本地课程设计，待执行 |
| R20 | [[Android系统工程师转型Agent开发路线]] | 交付证据、岗位与试点，不做收入保证 | 本地职业规划，建议与决策分开 |
| R21 | [[agent开发/端侧终端Agent转型/端侧终端Agent开发技能地图]] | 对照系统集成/应用/推理优化分支，补短板 | 本地技能地图 |

## 视频补课：保留少量，不替代真机练习

| 编号 | 链接 | 当前怎么用 | 本次核验边界 |
| --- | --- | --- | --- |
| V1 | [李宏毅生成式 AI P1](https://www.bilibili.com/video/BV1YeT7zBEPi/?p=1) | 概念不清时补 Token/上下文/幻觉 | 历史已归档分P；本次 B站412，未复核现目录与可播放性 |
| V2 | [李宏毅 Agent P2](https://www.bilibili.com/video/BV1YeT7zBEPi/?p=2) | 工具循环、观察/行动；配合手机实例 | 同上，不新增时间戳 |
| V3 | [黑马 RAG/Agent](https://www.bilibili.com/video/BV1yjz5BLEoY/) | 只按需要看模型/工具/结构化输出，不要求先学完 RAG | 本次412；保留历史候选，不背书更新/授权/口碑 |
| V4 | [Android Developers：端侧 GenAI walkthrough](https://www.youtube.com/watch?v=EpKghZYqVW4) | 了解 Nano/AICore 与端侧方案背景 | 历史官方视频，本次抓取失败；SDK 操作以当前文档为准 |

如果视频打不开，直接使用 R01/R04 中文图文和对应 Android 官方文档，勿因找课中断当前练习。未新增未经验证的收费训练营，不承诺一门课覆盖所有 Android 系统 Agent 能力。

## 两条模型分支的资料使用边界

- AICore：R06/R07/R10/R11，配 [[agent开发/端侧终端Agent转型/AICore与MLKit端侧Agent详细接入步骤]]；需要官方支持环境，当前解锁 AOSP 未被确认符合条件。
- LiteRT-LM：R08/R09，配 [[agent开发/端侧终端Agent转型/LiteRT-LM的Kotlin最小推理实验]]；可在现有设备评估，模型/驱动需实测。
- 通用：R01—R05、R12—R21，两条均可用。AIDL 可拆工具服务；不据此绕过 AICore 前台限制，也不把自建服务称为 AICore 移植。

## 历史资源：按需选修

面向 Android 系统工程师转型端侧智能体。2026-10-24 检索官方课程与文档；课程注册、收费、字幕和 API 使用费用以页面当前说明为准。未要求购买课程。

## 推荐顺序

| 顺序 | 资源 | 形式/语言 | 学习目标与产出 |
| --- | --- | --- | --- |
| 1 | [Hugging Face Agents Course 中文源文档](https://github.com/huggingface/agents-course/blob/main/units/zh-CN/unit0/introduction.mdx) | 中文图文与练习 | 从 Unit 0/1 理解 Agent、工具与执行循环，画出模型和宿主职责 |
| 2 | [AI Agents in LangGraph](https://www.deeplearning.ai/courses/ai-agents-in-langgraph) | 英文视频与代码 | 优先学习从零实现 Agent、持久化与流式、人机确认；先复现 Python 示例，再将执行思想用于 Kotlin。旧代码按当前依赖版本核对 |
| 3 | [Building effective agents](https://www.anthropic.com/research/building-effective-agents) | 英文工程文章 | 理解工作流与 Agent 的选择和简单可组合设计，写一页架构取舍 |
| 4 | [Android Kotlin 协程](https://developer.android.google.cn/kotlin/coroutines?hl=zh-cn)与[Flow](https://developer.android.google.cn/kotlin/flow?hl=zh-cn) | 官方文档，可选中文 | 将模型响应接入 UI，完成取消、状态更新和错误处理 |
| 5 | [LiteRT-LM Android 入门](https://developers.google.cn/edge/litert-lm/android) | 官方技术文档，所查页面为英文 | 使用 Kotlin API 完成设备端推理与流式输出，测量加载、延迟和内存 |
| 6 | [Google AI Edge Gallery](https://github.com/google-ai-edge/gallery)与[官方样例集合](https://github.com/google-ai-edge/litert-samples) | 源码与样例 | 理解模型加载、会话和 UI 的边界；先运行最小样例，再读较完整工程 |

## 可选系统课程

- [DeepLearning.AI：Agentic AI](https://www.deeplearning.ai/courses/agentic-ai)：英文视频，覆盖任务拆解、Agent 设计与评测，适合需要系统讲解时选用。与短课择一作为主线，避免同时铺开。
- [Claude Academy：Building with the Claude API](https://academy.claude.com/courses/building-with-the-claude-api)：官方课程，覆盖提示、工具、RAG、Agent、MCP 和生产模式；适合选择该供应商时使用，概念可迁移，接口需要按自己的供应商适配。

## 历史两周建议（已由当前课序替代）

- 第一周：Hugging Face 基础单元 + LangGraph 短课的从零 Agent 部分，做出只读搜索工具闭环。
- 第二周：按需查协程/Flow，完成 Kotlin 最小助手，加入创建待办确认与取消。
- 本地推理教程在云端/假模型执行闭环可用后学习。首轮不要求学完全部框架、训练或微调内容。

## 注意事项

- Python 教程主要用于理解机制，Android 实现需要自己处理生命周期、权限、状态和系统接口。
- 官方端侧教程是学习路径之一，不代表目标手机厂统一使用 Google 技术或具备 Google 服务环境。
- 本次未验证 B 站搬运课程的完整性、授权和更新情况，因此优先列官方来源。

## 关联

- [[端侧Agent入门实战计划]]
- [[端侧智能体岗位核心技术要求]]

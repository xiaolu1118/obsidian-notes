---
created: 2026-10-24
updated: 2026-10-24
tags: [学习资源, Agent, Android, 教程]
---

# Agent 与 Android 端侧 AI 教程清单

> 详细执行版见 [[零基础转型端侧Agent视频跟学指南]]，包含 Python 前置、按周课节、作业与验收；本文保留为资源速查。

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

## 当前两周建议

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

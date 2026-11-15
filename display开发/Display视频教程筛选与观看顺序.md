# Display 视频教程筛选与观看顺序

创建日期：2026-11-15。更新日期：2026-11-15。
返回：[[display开发/索引|Display 索引]]；对应 [[AOSP16手机与车机Display三大专项实战计划]]。

## 背景与核验边界

用户已确认 AOSP16 手机 Display 岗位，学习亮度、亮灭屏、多屏并加入 AAOS 车机内容；本轮询问视频教程。

已完成：检索公开视频页与课程目录。未完成：逐条完整观看、核验付费课内容、在 AOSP16 上复现实验。未找到能确认完整覆盖三个专项的 AOSP16 专项视频课；不表示网上不存在。

以下是观看建议，不代表用户决定付费。公开视频中部分属于课程试看、短讲或项目展示，不应当作完整课程。旧视频用于概念与排障方法，调用链、命令、配置和权限需对照项目 AOSP16 分支。

## 推荐入口

### 本轮澄清：优先直接讲 Display 的教学

用户进一步强调要 Display 本身的教学视频。此前推荐的通用 PMS、工具和投屏内容作为辅助，不作为直接匹配的 Display 主课。

新核验到更直接的入口：**阿豪讲Framework《Android14 显示系统剖析》公开合集**。可从 [直接与 SurfaceFlinger 交互的示例程序](https://www.bilibili.com/video/BV1aM1eYvEAV/) 进入同页合集，页面显示4节：

1. 学习环境准备：14分41秒。
2. 计算机显示基本原理：7分03秒。
3. Android14显示系统概述：11分38秒。
4. 直接与 SurfaceFlinger 交互的示例程序：9分39秒。

建议按上述顺序观看，约43分钟。已核验作者页视频信息及合集目录，未完整观看。该合集直接讲显示基础与 SF 实验，**尚不能据此认定包含 DMS/DPC、自动亮度或亮灭屏完整专题**；基线是 Android14，后续在用户 AOSP16 分支核对。当前将其作为视频入门首选，后续补三大业务专题。

本轮继续检索未取得可核验的 DMS/DPC/自动亮度完整教学视频链接，不把关键词、文章或普通图形性能课写成已找到的专项视频。

| 顺序/主题 | 视频或课程入口 | 页面可确认内容 | 使用建议与边界 |
|---|---|---|---|
| 1 亮灭屏上游 | [全·安卓 framework 实战课程目录](https://www.bilibili.com/cheese/play/ep2276944) | 第86节 PWM 系统按键；87～90节 PowerManagerService 简介和常见函数；混合试看和付费课时 | 对照电源键唤醒入口学习；基础部分涉及 Android12，目录另有 AOSP15 内容，不能称 AOSP16 Display 专项。此处 PMS 是 PowerManagerService，与 PackageManagerService 区分 |
| 2 调试工具 | [Perfetto/systrace 两个使用技巧](https://www.bilibili.com/video/BV1jh4y1B73G/) | 千里马学框架，2023年公开视频页 | 配合一次正常唤醒抓取；旧 systrace 内容不代替目标分支 Perfetto 数据源说明 |
| 3 多屏定位案例 | [双屏动画位置偏移 bug 解决](https://www.bilibili.com/video/BV1FT411t7gr/) | 千里马学框架，2023年，页面列出约12分钟 | 学坐标与跨屏问题的分析思路；不能代替 DMS 生命周期、AAOS 乘员区域或多用户课程 |
| 4 多屏方案 | [手机车机互联扩展投屏/平行世界实现方案](https://www.bilibili.com/video/BV1UU411U7TM/) | 千里马学框架，2024年，页面列出17分21秒，投屏专题试看部分 | 区分扩展显示与镜像；手机互联投屏与原生 AAOS 多用户多屏分开理解 |
| 5 合成基础（选学） | [Google I/O 2013 Android Graphics Performance](https://www.youtube.com/watch?v=vQZFaec9NpA) | Google for Developers，Chet Haase/Romain Guy，英文 | 补充图形性能背景；版本很旧，优先级低于当前电源/亮度主线，不用于照抄现代 SF 架构 |

其他候选：[双屏互动项目问题讨论](https://www.bilibili.com/video/BV1aM411G7rf/)，页面标注2023年、约6分钟，适合补充案例；[BLASTBufferQueue 短讲](https://www.bilibili.com/video/BV1jw411g7Uj/)可在进入首帧与黑屏分析时选看，不需要现在深入整个渲染栈。

## 亮度与车机电源的资料缺口

本轮未核验到足够匹配的 AOSP16 自动亮度/DPC 完整免费视频，不用手机使用技巧或横评代替源码教程。可在 B站继续搜索 DisplayPowerController、AutomaticBrightnessController、BrightnessMappingStrategy、DisplayDeviceConfig、亮灭屏源码；这些是检索关键词，不是已确认课程名。

车机亮灭屏结合 [AAOS 电源管理](https://source.android.com/docs/automotive/power/power) 学习 CPMS/VHAL 与状态机；亮度和多屏源码沿专项实战计划执行。该入口是文档，不冒充视频。

## 推荐跟学方法

第一轮先看 PWM/PMS 相关课时或可用试看，再抓一次电源键唤醒的日志和 trace；接着看 Perfetto 技巧并复查自己的抓取；最后做一次辅助屏/外接屏实验，再看跨屏案例。

每看完一个主题，记录：视频使用版本、项目对应类/方法、请求与状态变化、实际证据、视频与 AOSP16 的差异。以能跑通实验、解释 dump/trace 为验收，不以播放完成作为掌握。

付费前建议核对实际章节是否包含 DPC、自动亮度、逐屏电源与 AAOS 乘员区/多用户，是否提供版本与实验；仅凭课程名称或展示效果无法确认匹配岗位。

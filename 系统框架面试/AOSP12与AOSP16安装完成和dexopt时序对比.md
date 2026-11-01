# AOSP 12 与 AOSP 16 安装完成和 dexopt 时序对比

创建日期：2026-10-31。
更新日期：2026-10-31。
所属：[[系统框架面试/索引|系统框架面试]]。

## 问题与核验范围

用户问题：PMS 应用安装完成与 dex2oat 的先后顺序，在 AOSP 12 和 16 是否不同？

本轮已对照 `android-12.0.0_r1`、`android-16.0.0_r1` 的公开源码。范围为普通 APK 安装主路径，不直接套用于开机扫描、APEX、staged 安装或厂商定制。尚未核验具体设备的功能开关值和运行日志。

## 结论

需要区分包状态提交、安装成功通知、后续后台优化。安装阶段确实执行 dexopt 时，两版均先结束这次 dexopt，再进入最终安装通知链路。Android 16 的异步分支不表示先通知成功、随后才编译。

关键差异是 dexopt 与内部 commit 的相对位置：

| 路径 | 简化时序 |
| --- | --- |
| AOSP 12 | prepare / scan / reconcile → commitPackagesLocked → executePostCommitSteps 内同步 dexopt → post-install / 通知 |
| AOSP 16，improveInstallFreeze 关闭 | commit → post-commit 同步 dexopt → post-install / 通知 |
| AOSP 16，improveInstallFreeze 开启 | rename / 准备 → 异步 dexopt → 等待 Future → doPostDexopt → commit → post-install / 通知 |

这里的 commit 是 PMS 包状态提交，不是调用 PackageInstaller.Session.commit() 的动作。广播发送和接收方实际收到广播也应分开看。

## 源码定位

### 先后顺序的实际影响：缩短更新时的不可用窗口

前面的“登记”类比省略了冻结，无法解释改动价值。以普通覆盖更新 V1 → V2、未使用 INSTALL_DONT_KILL_APP 为例：

- AOSP 12 在准备阶段创建 PackageFreezer，再提交新包，随后执行安装 dexopt。因此优化期间应用仍处于这次更新的冻结窗口。
- AOSP 16 开启 improveInstallFreeze 时，先对尚未提交的 V2 做 dexopt，结束后才在 commitInstallPackages 中冻结应用并提交。优化耗时被移到了冻结窗口之前，旧版可以在该阶段继续运行；这不保证其他条件下进程绝不会被杀。
- PackageFreezer 的“冻结”是包管理层阻止应用在代码/数据变更期间运行，并可杀死已有进程，不是暂停线程后原地恢复，也不是缓存进程的 freezer。

示意数字（不是实测）：若优化耗时 5 秒、切换耗时 0.2 秒，旧流程的不可用窗口包含这 5 秒优化；新流程先让旧版继续运行并优化新版，再进入短暂切换。两者安装成功通知仍需等优化结束，因此主要收益是减少应用更新中断时间，不保证总安装耗时缩短 5 秒。

优化前已经可以准备好新 APK 的最终路径；磁盘上存在 V2 文件与 PMS 已将当前包切换为 V2 是不同事件。首次安装没有 V1 可继续使用，所以此项收益在覆盖更新场景更直观。

核验位置：AOSP 12 preparePackageLI 附近的 freezePackageForInstall；AOSP 16 preparePackage 的条件冻结、commitInstallPackages 的延迟冻结，以及 PackageFreezer 类说明。

### 通俗理解：登记、优化、通知是三个节点

- 登记：PMS 提交包状态，可以粗略理解为更新内部应用账本；这不代表整个安装流程已经结束，也不保证此刻应用即可启动。
- 优化：安装阶段的 dexopt，按需调用 dex2oat。
- 通知：安装后处理完成，向安装器报告最终安装结果。

Android 12 简化为“登记 → 优化 → 通知成功”；Android 16 新开关分支为“优化 → 登记 → 通知成功”。开关关闭时仍是前一种顺序。

异步的意思是把优化交给工作线程执行。Android 16 安装流程此时暂停在等待点，工作完成后才续接登记和通知；不能理解成已经向安装器通知成功。可类比工作人员把任务交给另一位同事，但仍等同事交付后才向客户办结。

这里的“等待结束”也不等于保证编译成功；跳过、失败、成功是不同结果。安装成功后的后台优化属于另一轮任务。

- Android 12：`PackageManagerService.installPackagesLI()` 中先 `commitPackagesLocked()`，再 `executePostCommitSteps()`；后者调用 `mPackageDexOptimizer.performDexOpt()`。安装阶段返回后才进入 `restoreAndPostInstall()`。
- Android 16：`InstallPackageHelper` 检查 `Flags.improveInstallFreeze()`。开启时，`prepPerformDexoptIfNeeded()` 用 `CompletableFuture.allOf(...).thenRun(...)` 安排 `doPostDexopt()`，随后提交和 post-install；关闭时在 `executePostCommitStepsLIF()` 内同步执行。
- `DexOptHelper.performDexoptIfNeededAsync()` 使用独立执行器，调用 ART Service 后完成 Future；源码执行器当前只有一个工作线程，异步不等于多包同时编译。
- 从 Android 14 起，应用 dexopt 由 ART Service 管理。应区分调度层 dexopt 与真正运行的 dex2oat 进程。

## 边界与排查建议

安装不保证一定启动 dex2oat，可能跳过或复用产物；安装成功也不等于完成所有 AOT 优化。后续后台 dexopt 可再次运行。普通 dexopt 失败不会直接使应用安装失败。

若实际看到“安装完成后才出现 dex2oat”，建议先对齐包名、代码路径、编译 reason、安装日志和回调时间，再判断是否为 bg-dexopt、其他包或特殊安装路径。不能仅凭 Android 版本断定。

待确认：用户的具体分支、设备开关、安装方式与日志；本轮未确认 improveInstallFreeze 在具体产品上的启用状态。

## 参考资料

- [AOSP 12 PackageManagerService](https://raw.githubusercontent.com/aosp-mirror/platform_frameworks_base/android-12.0.0_r1/services/core/java/com/android/server/pm/PackageManagerService.java)
- [AOSP 16 InstallPackageHelper](https://raw.githubusercontent.com/aosp-mirror/platform_frameworks_base/android-16.0.0_r1/services/core/java/com/android/server/pm/InstallPackageHelper.java)
- [AOSP 16 DexOptHelper](https://raw.githubusercontent.com/aosp-mirror/platform_frameworks_base/android-16.0.0_r1/services/core/java/com/android/server/pm/DexOptHelper.java)
- [ART Service 配置说明](https://source.android.com/docs/core/runtime/configure/art-service)

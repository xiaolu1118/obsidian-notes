---
created: 2026-10-24
updated: 2026-10-24
tags: [Obsidian, GitHub, Git, 知识库管理]
status: 操作指南，尚未执行同步
---

# Obsidian 同步到 GitHub

返回 [[索引]]。

## 当前状态与建议

2026-10-24 已检查：笔记库位于 `D:\work\Rwork`，首次检查时尚未初始化为 Git 仓库。此前检测到的 Git 2.53.0 来自 Codex 自带运行环境，不能据此认定用户 PowerShell 已安装或可调用 Git。用户随后在 PowerShell 执行 `git init -b main`，报 CommandNotFoundException。进一步检查发现系统和用户 PATH 未配置 Git；当前应先安装系统可用的 Git，再继续初始化。未创建远程仓库、提交或上传文件。

建议使用 GitHub 私有仓库保存笔记与历史，先手动验证上传，再按需使用 Obsidian Git 插件自动提交与同步。私有仓库是建议，用户尚未选择可见性或提供仓库地址。

## 首次配置

### 前置修复：PowerShell 无法识别 git

此前“已安装 Git”的描述不够准确：Codex 内部可用的是应用附带的 Git，用户的 PowerShell 并不因此获得该命令。建议从 [Git 官方 Windows 安装页](https://git-scm.com/install/windows) 安装 Git for Windows，安装时保留允许命令行及第三方软件调用 Git 的 PATH 选项。安装结束后关闭原 PowerShell 并重新打开；如果使用 VS Code 等内嵌终端，需重启宿主应用。

在新终端执行 `git --version`，确认能显示版本，再回到 `D:\work\Rwork` 执行 `git init -b main`。若仍提示未找到，先用 `Test-Path 'C:\Program Files\Git\cmd\git.exe'` 检查默认安装位置；若确实存在，可临时执行 `$env:Path += ';C:\Program Files\Git\cmd'` 后重试。自定义安装目录应改为实际路径。当前仅提供修复步骤，未代用户安装或修改系统 PATH。

1. 在 GitHub 新建仓库，例如 `obsidian-notes`，建议选择 Private。为了直接推送已有文件，创建空仓库，先不要勾选 README、许可证或远程 `.gitignore`。
2. 在本地库根目录创建 `.gitignore`。以下为入门方案，整个 Obsidian 配置目录暂不同步，笔记及未被忽略的附件仍会上传；需要配置同步时再细化规则。

```gitignore
.obsidian/
.trash/
.env
.env.*
Thumbs.db
.DS_Store
```

上述内容仅为示例，本轮没有写入实际 `.gitignore`。若希望同步配置，需先检查插件数据是否含凭证，并考虑排除 `workspace*.json` 等设备状态文件。不要提交密码、Token 或 API Key。

3. 在 PowerShell 执行以下命令。身份和 URL 都必须替换；邮箱可使用 GitHub 设置中提供的 noreply 地址。`git config` 不带 `--global`，仅作用于当前库。

```powershell
Set-Location 'D:\work\Rwork'
git init -b main
git config user.name '你的提交署名'
git config user.email '你的提交邮箱'
git add .
git diff --cached --stat
git status
```

查看暂存文件，确认上传范围后再执行：

```powershell
git commit -m 'Initial Obsidian notes'
git remote add origin https://github.com/你的用户名/obsidian-notes.git
git push -u origin main
```

HTTPS 认证可通过 Git Credential Manager 的浏览器登录流程完成；具体提示取决于本机凭证配置。不要使用 GitHub 账号密码代替 Git 的认证凭证，也不要把 Token 写进远程 URL、笔记或聊天。首次成功后，到 GitHub 检查目录和文件是否齐全。

## 日常手动同步

单机更新后：

```powershell
Set-Location 'D:\work\Rwork'
git add .
git commit -m 'Update notes'
git push
```

没有改动时无需提交。多设备使用时，编辑前在工作区干净的前提下执行 `git pull --ff-only`；编辑结束后提交并推送。出现分叉或冲突时先保留两侧内容并人工处理，不要直接强制推送或硬重置。

另一台电脑可先执行 `git clone 仓库地址`，再用 Obsidian 的“打开文件夹为仓库”打开克隆目录。忽略 `.obsidian/` 时，新设备需要单独配置 Obsidian。

## 可选：Obsidian Git 插件

在 Obsidian 设置的第三方插件中搜索 **Git**，对照官方项目 `Vinzent03/obsidian-git` 确认插件身份，再安装启用。建议先保证命令行 push/pull 正常，再测试插件的提交与同步命令。

插件支持自动提交与同步；可以先试用约 10 分钟间隔，再根据实际需要调整，具体设置名称以安装版本为准。自动化运行依赖 Obsidian 和插件处于可运行状态，不应假设关闭应用后仍持续上传。多设备尽量避免同时修改同一篇笔记，并在切换设备前确认同步完成。

## 常见问题

- `Author identity unknown`：配置本库的 `user.name` 和 `user.email`。
- `remote origin already exists`：先检查已有远程配置；确认目标后再调整，不重复添加。
- push 被拒绝：远程可能已有提交或其他设备已上传；先检查历史并拉取、合并，不强推覆盖。
- 认证失败：检查凭证管理器、账户权限和远程地址；不要粘贴凭证到笔记中排错。
- GitHub 展示与 Obsidian 不同：Markdown 文件可保存，但 `[[双向链接]]` 等 Obsidian 语法不保证按库内方式渲染。GitHub 保存文件和版本，不自动复制全部 Obsidian 界面功能。

## 待办

- [ ] 用户选择或创建 GitHub 仓库并提供地址。
- [ ] 确定要同步的文件范围与 `.obsidian` 策略。
- [ ] 初始化 Git、完成首个提交并验证推送。
- [ ] 按需启用并验证插件自动同步。

## 参考

- [GitHub：将本地内容添加到 GitHub](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github?platform=windows)
- [Obsidian：跨设备同步笔记](https://obsidian.md/help/sync-notes)
- [Obsidian Git 插件官方项目](https://github.com/Vinzent03/obsidian-git)

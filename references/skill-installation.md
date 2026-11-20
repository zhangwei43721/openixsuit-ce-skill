# 安装与版本更新

Skill 是一个独立 Git 仓库。按标签检出会同时取得 `SKILL.md`、`references/`、`scripts/` 和许可证文件。私有阶段使用有仓库访问权限的 GitHub 账号认证。

## 安装

下面使用 Codex 的用户级 Skill 目录 `~/.agents/skills`。已有 Git 安装时，更新命令使用当前安装目录。 [Codex Skill 目录](https://developers.openai.com/codex/skills)

在终端执行一次克隆。以初始版本 `v0.1.0` 为例：

```bash
git clone --branch v0.1.0 --depth 1 https://github.com/zhangwei43721/openixsuit-ce-skill.git ~/.agents/skills/openixsuit-ce
```

PowerShell 使用同一条 Git 命令和 Windows 目标路径：

```powershell
git clone --branch v0.1.0 --depth 1 https://github.com/zhangwei43721/openixsuit-ce-skill.git "$HOME\.agents\skills\openixsuit-ce"
```

安装完成后，在 Codex 中选择 `openixsuit-ce`。若列表没有刷新，重启 Codex。

## 更新

如果正在运行该 Skill 的 Webhook 接收端，先停止接收过程。获取标签并检出目标版本。以下以之后发布的 `v0.1.1` 为例：

```bash
git -C ~/.agents/skills/openixsuit-ce fetch --tags --prune
git -C ~/.agents/skills/openixsuit-ce checkout --detach v0.1.1
```

PowerShell：

```powershell
git -C "$HOME\.agents\skills\openixsuit-ce" fetch --tags --prune
git -C "$HOME\.agents\skills\openixsuit-ce" checkout --detach v0.1.1
```

`checkout` 使用发布标签固定 Skill 版本，同时切换脚本和参考文档。需要继续接收事件时，用原参数启动新版接收端。要回到旧版，检出旧标签即可。

## 发布与桌面端兼容

维护者在独立 Skill 仓库中完成并核对一个版本后，按语义版本打标签，例如 `v0.1.1`，再推送提交和标签。GitHub Release 可用同一标签并写明变更和要求的最低桌面端版本。安装直接从 Git 标签获取整个仓库。

发布前，对照该版本所引用的桌面端命令和字段核对兼容性。若 Skill 使用了新增的 `openixsuit-ai` 命令或参数，在发布说明中写出所需最低桌面端版本；若接口保持兼容，注明适用的桌面端版本范围。桌面应用可以单独发布，Skill 用户也可单独更新。桌面仓库通过子模块提交记录固定其对应 Skill；在桌面仓库选择新 Skill 版本时，将子模块指针更新到已发布的 Skill 标签所指提交，并随桌面仓库记录该指针。

# codex-session-repair

用于排查和修复 Codex 会话故障的中文指令型 skill。仓库名称和技能内部名称均为 `codex-session-repair`，界面显示名为“会话修复”。

本技能提供诊断、修复边界和验证流程，无脚本，也不运行自动修复服务。安装后由 Codex 按需读取指令并调用当前宿主可用的能力。

## 适用场景

- 会话无法续聊、新建或跨会话转发失败。
- 历史列表不可见、provider 不匹配、压缩请求报错或命令宿主入口缺失。
- `invalid_encrypted_content`、加密工具结果解码失败，或子代理只能收到正文占位符。
- 修复验收后，按原任务记录和明确授权清理本次快照及临时文件。

故障先按证据分层。出现加密字段、工具返回成功或创建了新会话，都不能单独证明历史损坏或恢复成功。

## 安装

推荐将整个仓库安装到用户级目录 `~/.agents/skills/codex-session-repair`。执行前确认目标目录尚不存在，并检查是否已在其他扫描路径安装同一技能。

```sh
mkdir -p ~/.agents/skills
git clone https://github.com/seeuagagin/codex-session-repair.git ~/.agents/skills/codex-session-repair
```

也可将下载的仓库内容放入该目录，确保 `SKILL.md` 位于目录根部。仅供一个项目使用时，可放到该项目的 `.agents/skills/codex-session-repair`。

根据 [OpenAI 官方技能文档](https://developers.openai.com/codex/skills#where-to-save-skills)，Codex 从用户级 `~/.agents/skills` 发现技能；在 Git 仓库中，还会扫描当前目录到仓库根目录之间的 `.agents/skills`。支持技能目录软链接。同名技能不会合并，应检查已有副本，避免在选择器中出现多个同名项。

Codex 会自动检测新安装及修改的技能；未出现时重启 Codex。可通过技能选择器或显式提及确认已被发现。磁盘上存在文件不能证明旧会话已加载新版指令。发现与重载说明见[官方安装说明](https://developers.openai.com/codex/skills#install-curated-skills-for-local-use)。

## 更新

如果使用 Git 安装，先检查是否有本地改动，再拉取更新：

```sh
git -C ~/.agents/skills/codex-session-repair status --short
git -C ~/.agents/skills/codex-session-repair pull --ff-only
```

存在本地改动或分支分歧时，先处理差异；不要用强制重置覆盖自己的修改。通过下载方式安装时，核对差异后更新技能文件。更新后检查技能是否可见，必要时重启 Codex，并在后续调用中确认使用新版指令。

## 使用示例

将 `<目标聊天UUID>` 替换为目标聊天的真实 UUID；同标题聊天可能对应不同对象。开始时提供故障现象、发生时间、已知错误及允许操作范围，避免粘贴完整历史或凭据。

先进行只读诊断：

```text
请使用 $codex-session-repair 排查聊天 <目标聊天UUID> 无法续聊的问题。
错误为 invalid_encrypted_content。先只读检查，给出故障层、可能的回退边界及验证缺口。
```

按已确认范围修复：

```text
请使用 $codex-session-repair 修复聊天 <目标聊天UUID>。
按本次已确认的边界先保留并核验完整快照，再执行获准修复，
在后续实际使用的 provider 下完成两轮无敏感标记续聊验证。
跨聊天消息发送和子代理测试尚未授权。
```

验收后清理：

```text
聊天 <目标聊天UUID> 已通过我的验收。
请使用 $codex-session-repair 清理本次修复记录确认的专用备份，
保留脱敏修复摘要、验证证据与必要映射；范围不明的对象先列出。
```

宿主支持的显式调用方式可能不同。Codex CLI 和 IDE 扩展支持 `$` 提及或 `/skills`；ChatGPT 中使用 `@` 选择技能，详见[官方调用说明](https://developers.openai.com/codex/skills#how-codex-uses-skills)。

## 依赖

**运行能力：** 需要支持技能的 Codex 宿主，以及当前宿主提供的正式会话工具。缺少所需接口时，流程可能需要应用自带、版本已核实的 app-server；调用前检查当前版本帮助与 schema，不能假定任意 PATH 中的 `codex` 都兼容。接口能力参见 [OpenAI App Server 文档](https://learn.chatgpt.com/docs/app-server)。Git 只用于上述克隆和更新方式，并非诊断流程本身的必需依赖。

**按场景可选：** 历史可见性补丁可复用 `codex-app-show-all-threads`；网关问题可能交由 `newapi-fix` 或 `sub2api-safe-upgrade` 处理。它们不随本仓库安装，也不假设已存在。遇到对应场景时，先核对是否安装、当前说明与权限；缺失时报告能力缺口，不自动安装或越权替代。

## 安全边界与验证

- 技能不授予历史删除、配置修改、消息发送、重启或业务执行权限；按用户授权与宿主审批执行。
- 优先通过正式接口读取和修复；不直接改写活动数据库、JSONL、锁或任务 registry，不通过删锁、杀宿主绕过占用。
- 回退会移除指定 turn 及之后的历史。先定位边界、列明损失范围、保留并核验完整快照；回退不会撤销文件、Git、数据库或远程操作的副作用。
- 不批量删除加密项，不将删除标签描述为解密。fork 可能继承损坏项，也不会自动迁移业务执行所有权。
- 使用无敏感标记验证，默认两轮：每轮等待完成、无错误并核对精确回复。桌面跨聊天转发与原生子代理通信分别验证；未获授权或缺少能力的通道明确记为未验证。
- 不用部署或远程业务命令测试通信，不重复启动仍有执行者的历史任务。相同故障没有新条件时停止重试，保留恢复点。
- 只读取必要字段，报告脱敏，涉及正文的材料仅向当前用户开放。备份清理须有明确授权，并逐项核对本次记录中的精确对象及级联范围。

历史案例记录只说明当时版本及实测范围，不承诺适用于所有安装。本技能安装或文档更新本身不表示任何目标会话已经恢复。

## 文件导航

| 文件 | 内容 |
| --- | --- |
| [SKILL.md](SKILL.md) | 技能入口、分流、权限边界、验证与交付合同 |
| [agents/openai.yaml](agents/openai.yaml) | 界面名称、简介与默认调用提示 |
| [references/diagnosis.md](references/diagnosis.md) | 故障层定位、工具入口与 provider 不匹配 |
| [references/recovery-cases.md](references/recovery-cases.md) | 历史可见性、压缩、命令宿主与加密工具结果案例 |
| [references/history-repair.md](references/history-repair.md) | 快照核验、历史回退、运行时占用与续聊验证 |
| [references/backup-cleanup.md](references/backup-cleanup.md) | 验收后精确清理与结果核验 |

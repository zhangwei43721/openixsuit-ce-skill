# 事件与电源

## 保存草稿并在客户端确认

`rules` 和 `power-profiles` 列出当前配置。`rule-save` 和 `power-profile-save` 从 JSON 文件或标准输入 `-` 保存单个草稿。规则至少填写 `id`、`name` 和 `condition.kind`，电源配置至少填写 `id`、`name` 和 `kind`，再根据控制方式填写参数。

返回 `draft: true` 后，可在桌面端「事件与电源」检查。CLI 可用相同 ID 更新未确认的草稿。规则由用户核对后勾选「启用」并保存。电源配置由用户核对设备、命令和断电时长，再点「确认电源配置」。

```bash
openixsuit-ai rule-save - <<'JSON'
{
  "id": "kernel-panic-log",
  "name": "内核异常日志",
  "condition": {"kind": "contains", "pattern": "Kernel panic"},
  "actions": [{"kind": "send_ai", "include_serial_lines": 100}],
  "cooldown_secs": 60
}
JSON
```

使用已确认的配置执行电源动作时，先确认目标，明确传入 `on`、`off` 或 `cycle`，例如 `openixsuit-ai power "$profile_id" cycle`。

## 异常分析

`openixsuit-ai incidents` 查看命中内容与日志快照。需要自动分析时，在规则里保留「生成 AI 排障上下文」动作，在「AI 协同台」开启 Codex 自动分析。真实事件触发时，客户端把日志快照交给本机 Codex CLI，并在事件旁展示进度、结果或失败原因。若客户端的 `PATH` 找不到 Codex，在同一页面填写可执行文件绝对路径。

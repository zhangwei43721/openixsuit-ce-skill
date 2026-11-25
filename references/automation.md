# 事件与电源

## 保存草稿并在客户端确认

`rules` 和 `power-profiles` 列出当前配置。`rule-save` 和 `power-profile-save` 从 JSON 文件或标准输入 `-` 保存单个草稿。规则至少填写 `id`、`name` 和 `condition.kind`，电源配置至少填写 `id`、`name` 和 `kind`，再根据控制方式填写参数。

规则条件支持日志匹配以及 `device_online`、`flash_completed`。后两者可用可选 `condition.pattern` 精确筛选目标。规则调用插件时填写 `plugin_id`、`plugin_action`、`plugin_inputs`，并可用 `adb_serial` 固定设备；诊断插件运行时可读取事件上下文中的 `incident.log_tail` 和 `incident.matched_text`。

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

## 把事件交给其他 Agent

从 Skill 目录运行 [Webhook 接收端](../scripts/webhook_receiver.py)。接收端使用 Python 3 标准库，在 OpenixSuit-CE 所在电脑运行。选择已安装且可在非交互模式下完成认证的 Agent。

```bash
python3 scripts/webhook_receiver.py --agent claude
```

预设为 `codex`、`claude`、`gemini` 和 `opencode`。最后一项面向 OpenCode V2 的权限配置。可用 `--executable` 指定程序路径，用 `--port`、`--timeout`、`--results-dir` 调整监听端口、分析超时和结果目录。Windows 可使用 `py -3`。各工具的参数和认证见 [Codex CLI](https://developers.openai.com/codex/noninteractive)、[Claude Code](https://code.claude.com/docs/en/cli-reference)、[Gemini CLI](https://geminicli.com/docs/cli/headless/)和 [OpenCode](https://opencode.ai/docs/agents#permissions)。

其他工具可在 `--` 后提供非交互命令和独立参数。命令从标准输入读取事件分析提示词，并把结果写到标准输出。若 CLI 要求指定参数，用单独的 `{prompt}` 参数接收提示词，或用 `{prompt_file}` 接收临时 UTF-8 文件路径。命令在临时目录运行，脚本和额外文件请用绝对路径。自定义命令的工具权限由该命令及其 Agent 配置决定。使用日志分析配置，关闭工具和自动批准，避免让设备日志触发操作。

```bash
python3 scripts/webhook_receiver.py -- /path/to/agent run --input-file '{prompt_file}'
```

### 配置触发规则

将接收端输出的 `Webhook` 地址填入「事件与电源」的 Webhook 通知动作。接收端仅在本机监听，路径包含接收凭据。沿用原结果目录和端口启动时，可以继续使用同一个 URL。接收期间保持程序运行，按 Ctrl+C 停止。

界面的 Webhook 动作可以填地址。需要附带最近日志时，用客户端 CLI 保存带 `prompt` 和 `include_serial_lines` 的草稿。把下面的地址换成接收端实际输出，调整匹配文字和行数，然后在桌面端检查、启用并保存。

```bash
openixsuit-ai rule-save - <<'JSON'
{
  "id": "serial-panic-webhook",
  "name": "串口异常交给 Agent",
  "condition": {"kind": "contains", "pattern": "Kernel panic"},
  "actions": [{
    "kind": "notify_webhook",
    "webhook_url": "http://127.0.0.1:28766/替换为实际路径令牌/webhook",
    "include_serial_lines": 100,
    "prompt": "规则：{{rule_name}}\n串口：{{port_name}}\n命中：{{matched_text}}\n最近日志：\n{{log_tail}}"
  }],
  "cooldown_secs": 60
}
JSON
```

同一规则只需选择一条分析路径。若保留 Webhook 和「生成 AI 排障上下文」两项动作，同时开启了桌面端的 Codex 自动分析，真实事件会走两条路径。界面的「测试规则」会实际发送通知，使用只含 Webhook 通知动作的规则测试接收端。

### 查看分析结果

Webhook 返回 HTTP 202，表示事件已进入接收端队列。打开启动时输出的「任务查询」地址，可查看本轮最近 50 个任务的排队、运行、完成和失败状态。结束的任务按 ID 保存在结果目录的 JSON 文件中，默认位置为 `~/.openixsuit-ce/webhook-results/`。

桌面事件记录显示 Webhook 通知结果。Agent 输出和调用失败原因从接收端查看。收到 HTTP 503 时队列已满或接收端正在停止。等待当前分析完成后，再用同一测试规则重试。

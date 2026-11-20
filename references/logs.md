# 多路日志

## 查询与控制实时会话

`serial-ports` 列出主机端口，`log-sessions` 列出当前在采集的串口和 ADB 会话。串口和 USB CDC 可以直接打开，随后使用返回的 `session.id` 读取缓冲、发送或关闭。

```bash
openixsuit-ai serial-ports
openixsuit-ai serial-open /dev/ttyUSB0 --baud-rate 115200
openixsuit-ai serial-logs --session-id "$session_id" --tail 200
openixsuit-ai serial-send "$session_id" "00 FF 7E" --hex
openixsuit-ai serial-close "$session_id"
```

文本发送默认追加 LF，`--hex` 发送原始字节，默认原样发送。可用 `--append-newline none|lf|cr|crlf` 明确行尾。恢复连接后通过 `log-sessions` 获取新的会话 ID。查看历史文件时，用 `serial-logs --file-name "$file_name" --tail 200` 读取末尾内容。

## 保存采集配置与启停互转

`log-profile-save` 接受 JSON 文件或 `-` 标准输入，同 ID 会更新已有配置。

- 配置包含 `id`、`name` 和 `sources`。每路 `label` 唯一。串口和 USB CDC 来源用 `type: "serial"`，填 `port_name` 和 `baud_rate`。ADB 来源用 `type: "adb"`，填设备序列号 `serial` 和设备端 `command`。
- `bridge.first` 和 `bridge.second` 各填 `label`、`port_name` 和 `baud_rate`。只保存互转配置时，`sources` 可填空数组。
- 串口来源和互转端点可用 `line_coding` 指定 `data_bits`、`parity`、`stop_bits` 和 `flow_control`。串口来源还支持 `serial_options` 的 `receive_format`、`send_format`、`charset`、`line_ending` 和 `auto_scroll`。

下例创建一路 ADB 日志。把 `serial` 改为实际序列号，把 `command` 改成设备支持的日志命令。

```bash
openixsuit-ai log-profile-save - <<'JSON'
{
  "id": "demo-logs",
  "name": "设备日志",
  "sources": [
    {"type":"adb","label":"系统日志","serial":"实际设备序列号","command":"logcat -v threadtime"}
  ]
}
JSON
openixsuit-ai log-profiles
openixsuit-ai log-start demo-logs "系统日志"
openixsuit-ai log-sessions
openixsuit-ai log-stop "$session_id"
```

互转使用含 `bridge` 配置的 ID：

```bash
openixsuit-ai bridge-start "$profile_id"
openixsuit-ai bridge-status
openixsuit-ai bridge-stop
```

`log-start` 使用配置 ID 和原样的来源标签，启动后会话出现在桌面端实时分屏。软件重开或设备重插时，退出或断线前正在采集的分屏会等原设备连接再恢复。互转等原两端都连上。手动停止的会话保持停止。

## 调整分屏、互转与日志保存

`log-ui get` 返回当前分屏 ID、采集与显示设置、列数和互转设置。以下命令用前一次查询得到的 `pane_id`，修改当前分屏名称和接收格式：

```bash
openixsuit-ai log-ui get
openixsuit-ai log-ui pane-set - <<'JSON'
{"pane_id":"填入分屏 ID","label":"调试串口","receive_format":"hex"}
JSON
```

- 活跃时可以修改分屏名称和显示设置。修改设备或线路参数前，先停止该路。对于等待恢复且还没有会话 ID 的分屏，在 `pane-set` 中传 `pane_id` 和 `"restore_on_startup":false` 取消恢复。
- `log-ui layout-set` 支持 `columns` 和 `pane_order`。`log-ui bridge-set` 支持 `profile_id`、`first_port`、`second_port`、`format`、`direction`、`show_timestamps` 和 `auto_scroll`。修改互转两端前先停止互转。`bridge-stop` 也能取消等待中的互转恢复。
- `log-storage` 查询已生效的目录与保存策略。`log-storage-save` 接受需要修改的 JSON 字段 `save_continuous`、`keywords`、`before_lines`、`after_lines` 和 `directory`。将 `directory` 设为 `null` 可恢复默认的系统文档目录。

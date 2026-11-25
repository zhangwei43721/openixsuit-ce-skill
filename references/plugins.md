# 客户端插件

插件是由清单、JavaScript 或进程入口组成的本地扩展包。`plugin validate` 和 `plugin pack` 只读取本地文件；安装、运行和 provider 管理经由已启动的桌面端执行。

## 常用 CLI

查看 `openixsuit-ai plugin --help` 获取桌面版本支持的命令。命令返回 `{ok:true,data:...}` 表示请求成功；运行 action 返回任务 ID，使用 `runs`、`show` 查看进度、输出和待确认动作。

```bash
openixsuit-ai plugin validate ./plugins/my-plugin
openixsuit-ai plugin pack ./plugins/my-plugin ./my-plugin.oixplugin
openixsuit-ai plugin list
openixsuit-ai plugin describe my-plugin
openixsuit-ai plugin install ./my-plugin.oixplugin
openixsuit-ai plugin configure my-plugin settings.json
openixsuit-ai plugin run my-plugin analyze --inputs '{"log_text":"启动失败"}'
openixsuit-ai plugin runs
openixsuit-ai plugin show RUN_ID
openixsuit-ai plugin confirm RUN_ID CONFIRMATION_TOKEN
openixsuit-ai plugin cancel RUN_ID
openixsuit-ai plugin update my-plugin
openixsuit-ai plugin uninstall my-plugin
```

`configure` 的 JSON 文件可用 `-` 从标准输入读取。`install` 也支持 `--github OWNER/REPO`。设备相关 action 可用 `--device EXACT_ID` 选择目标；`--inputs` 接收 JSON 对象。写分区、重启等操作会先返回待确认动作，在核对目标和动作后用 `show` 返回的 token 调用 `confirm`。不继续时调用 `cancel`。

插件配置可启用插件、授予权限并提供本地进程绑定。下面为 Python 脚本入口授权 `process.execute`，并把清单中的 `interpreter: "python"` 绑定到解释器绝对路径：

```bash
openixsuit-ai plugin configure v881-log-diagnostic - <<'JSON'
{"enabled":true,"grants":["process.execute"],"bindings":{"python":{"interpreter":"/usr/bin/python3"}}}
JSON
```

配置中提供的集合字段会替换此前对应值。提交前保留仍需的 `grants` 和 `bindings`。

provider 命令用于管理持续运行的数据来源；先启动，再通过会话 ID 查询或调用其操作，完成后停止：

```bash
openixsuit-ai plugin provider-list
openixsuit-ai plugin provider-start log-session-provider tail --inputs '{"source_session_id":"LOG_SESSION_ID"}'
openixsuit-ai plugin provider-call PROVIDER_SESSION_ID status
openixsuit-ai plugin provider-stop PROVIDER_SESSION_ID
```

## 插件清单

每个插件目录包含 `manifest.json` 和入口文件。清单顶层字段为 `id`、`name`、`version`、`permissions`、`settings`、`actions`、`pages`、`providers`；`description`、`icon` 可选。`plugin validate` 会报告清单或包结构错误。

`settings` 和 action 的 `inputs` / `output` 使用字段定义：`id`、`label`、`type`，可选 `required`、`default`、`options`、`description`。字段类型为 `text`、`file`、`select`、`boolean`、`number`、`json`；`select` 用 `options` 定义候选值。Action 由 `id`、`label`、`inputs`、可选 `output` 和 `entry` 构成。`entry` 可指向 JavaScript 模块导出，或进程入口。进程入口可填写包内相对 `program`、宿主 `binding`、可选 `interpreter` 绑定、`interpreter_args` 和脚本参数 `args`，并用 `platforms` 指定 `windows-x86_64` 等平台对应的包内入口；`timeout_ms` 可设置超时。Page 将一个或多个 action 放入页面，`actions` 填 action ID。

最小 JavaScript action：

```js
export async function analyze({ inputs, settings, context }, host) {
  host.progress(10);
  const result = await host.call("logs.sessions", {});
  host.log(`找到 ${result.length} 个日志会话`);
  host.progress(100);
  return { count: result.length };
}
```

`host.call(name, args)` 是宿主提供的同步往返 RPC，JS 端可 `await` 其结果。`host.log(string)` 写运行日志，`host.progress(number)` 更新进度，`host.sleep(ms)` 暂停 action。入口收到 `{inputs, settings, context}`。插件只应请求完成工作所需的权限，并使用对应的类型化宿主操作；需要执行设备 shell 时应申明 `adb.shell`，桌面端会在调用前提示用户。

进程入口通过标准输入接收一行 `{type:"start",inputs,settings,context}`。`interpreter` 引用插件配置中声明的解释器绑定，例如清单填 `"interpreter":"python"`，运行配置为 `{"python":{"interpreter":"/usr/bin/python3"}}`。解释器参数在脚本路径前传入。PowerShell 可设置 `"interpreter_args":["-NoProfile","-NonInteractive","-File"]`。也可用 `binding` 引用本地进程绑定；平台映射 `platforms` 用于选择随包提供的系统专用入口。标准输出使用 NDJSON：插件可输出 `{type:"log",message}`、`{type:"progress",value}`、`{type:"result",value}`；宿主调用请求为 `{type:"call",id,method,args}`，宿主以 `{type:"response",id,ok,value,error}` 回应。诊断信息写标准错误。只把脚本和可执行文件放入插件包，并在清单中声明包内相对路径及需要的平台入口。

已有脚本接受命令行参数并输出普通文本时，进程入口设 `"protocol":"stdio"`。`args` 中整项写作 `{{input.log_file}}` 会将该输入作为一个独立参数传给程序。标准输入为空，输出保留为运行日志，成功时结果包含 `stdout` 和 `exit_code`。需要使用宿主接口的脚本可用默认的 `ndjson` 协议。

## 设备宿主操作

宿主当前提供这些类型化操作：`logs.recent({session_id,lines?})`、`logs.sessions({})`、`devices.list({})`、`adb.root({})`、`adb.push({local_path,remote_path})`、`adb.pull({local_path,remote_path})`、`adb.shell({command})`、`adb.reboot({})`、`adb.partition.inspect({partition})`、`adb.partition.write({partition,local_path,expected_slot,reboot?})`、`adb.deploy({local_path,remote_dir,remote_name,need_root,restart_cmd?})`。写分区操作由宿主检查非活动槽位、上传、读回校验并在需要时重启；整体操作通过一次确认。`adb.deploy` 集中处理推送流程并一次性确认。插件不能用 shell 拼接来替代这两个组合操作。

权限名包括 `logs.read`、`devices.read`、`process.execute`、`providers.log`、`adb.transfer`、`adb.partition`、`adb.shell`、`adb.reboot`、`adb.root`。权限用于说明插件请求的能力；执行敏感设备动作时，宿主仍按动作流程要求用户确认。

## Provider

Provider 描述持续的数据来源或设备能力。字段含 `id`、`label`、`kind`（`log`、`device`、`power`）、`actions` 和可选 `reconnect_ms`。`actions` 把操作名映射到 action ID，例如 `{"connect":"connect","status":"status"}`；provider-start 执行 `connect`，provider-call 可用会话 ID 调用 `status`、`read`、`on`、`off`、`cycle` 等清单中映射的操作。

`context.provider` 提供 `{id,session_id,kind,operation}`。持续 provider 的 `connect` action 返回或报错后，宿主会在 `reconnect_ms` 后重试，只要 provider 仍处于活动状态；停止 provider 会取消后续重连。`log` 类型中，`host.log` 的输出会进入 provider 日志历史并参与既有关键词触发规则。短暂 `host.sleep` 后正常返回可用于演示连续重连。

自动化规则可用 `device_online` 和 `flash_completed` 作为条件类型，并可通过 `condition.pattern` 指定要匹配的设备或刷写目标。规则调用插件时使用 `plugin_id`、`plugin_action`、`plugin_inputs`，可用 `adb_serial` 固定目标设备。插件 provider 输出的日志同样可以触发现有日志关键词规则；无硬件演示时可启动 provider 并传入 `sample_text`，例如 `PANIC provider sample`。

## 示例

仓库的 [V881 A/B 升级示例](../../../../plugins/v881-ab-ota/manifest.json) 使用 `adb.partition.inspect` 和 `adb.partition.write`；[V881 应用推送示例](../../../../plugins/v881-app-push/manifest.json) 使用 `adb.deploy`。这两个 action 需要连接目标设备，执行前会有对应权限和确认流程。

[离线日志诊断示例](../../../../plugins/v881-log-diagnostic/manifest.json) 提供 JavaScript 和包内 Python 两种文本分析 action；Python 入口需要 `process.execute` 授权和解释器绑定。可在隔离工作树执行 `node plugins/tests/offline-smoke.mjs`，用 mock 宿主检查 V881 RPC、provider 调用和 Python NDJSON 往返，无需桌面端或设备。[日志会话 provider 示例](../../../../plugins/log-session-provider/manifest.json) 通过 `logs.recent` 读取已有日志会话并在返回后自动重连；provider-start 的输入填写桌面端已有日志会话 ID。

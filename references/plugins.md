# 客户端插件

`openixsuit-ai plugins` 查看已安装插件、标签页与输入定义。安装来源可以是带插件清单的源码目录、`.oixplugin` 或 `.zip` 包。

```bash
openixsuit-ai plugins
openixsuit-ai plugin-install "/path/to/plugin.oixplugin"
openixsuit-ai plugin-run "$plugin_id" --tab "$tab_id" --serial "$adb_serial" --inputs "$inputs_json"
openixsuit-ai plugin-uninstall "$plugin_id"
```

运行时按插件清单传 `--inputs` JSON 对象。指定标签页时使用清单里的 `--tab` ID。插件如果会重启设备或写入存储，执行前确认选中的工作流、目标设备和对应动作已获授权。

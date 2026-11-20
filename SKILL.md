---
name: openixsuit-ce
description: 通过 OpenixSuit-CE 桌面客户端 CLI 操作和排查全志设备。适用于固件检查与烧录、ADB 和 EFEL 调试、多路日志采集与串口互转、事件与电源自动化、Webhook 接入 Agent、客户端插件。
---

# OpenixSuit-CE 客户端 CLI

使用 `openixsuit-ai` 调用已运行的桌面端。`openixsuit-ce ai` 接受同样的子命令。连接被拒绝时，先确认桌面端已经启动，默认地址为 `127.0.0.1:28765`。

操作设备前，使用 `openixsuit-ai status` 确认目标。`adb_devices` 提供 ADB 序列号，`fel_devices` 提供烧录设备位置。若 `busy` 非空或命令返回占用错误，等当前任务结束后再查状态，由桌面端继续操作。

需要单独安装或更新这个 Skill 时，按[安装与版本更新](references/skill-installation.md)操作。Skill 和桌面应用可以分别选择版本。

按当前任务读取对应说明：

- 检查、解包、转换固件，或选择烧录目标与模式，读[固件与烧录](references/firmware.md)。
- 使用 ADB 命令、传输文件、采集性能指标或访问 FEL 内存，读[设备调试](references/device-debugging.md)。
- 配置或启停串口、USB CDC、ADB 日志与串口互转，或调整分屏和保存策略，读[多路日志](references/logs.md)。
- 保存事件规则或电源草稿、查看异常，或用 Webhook 把串口事件交给 Codex、Claude Code 等 Agent，读[事件与电源](references/automation.md)。
- 查看、安装或运行扩展，读[客户端插件](references/plugins.md)。

以执行结果的 `ok` 和 `error` 判断成败。命令超时或断开时，先确认桌面端的进度与 `busy` 状态，再决定如何处理，避免重复发起设备写入。

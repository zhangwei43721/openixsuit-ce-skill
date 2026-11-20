# 固件与烧录

## 检查固件

标准全志 IMAGEWTY 固件包用 `inspect` 查看固件头、启动组件和分区名。需要提取文件时用 `unpack`，需要生成量产烧录器镜像时用 `convert`。给桌面端传绝对路径，能明确它要读写哪个文件。

```bash
openixsuit-ai inspect "/path/to/firmware.img"
openixsuit-ai unpack "/path/to/firmware.img" "/path/to/unpacked"
openixsuit-ai convert "/path/to/firmware.img" --target spinor --nor-size 16 \
  --storage-size auto --output "/path/to/programmer.bin"
```

## 选择目标设备

检查 `openixsuit-ai status`。如果目标板已在 `fel_devices` 中，等 `busy` 为 `null` 后开始。如果目标板只在 `adb_devices` 中且支持通过 ADB 切换烧录模式，把实际序列号填入 `openixsuit-ai reboot-efex --serial "$adb_serial"`，然后重新查询状态，确认该板进入 FEL/FES。也可按板卡的进入流程手动切换。

`flash` 和 `raw` 都接收 USB 总线号和端口号。多台设备处于烧录模式时，从状态中确认目标的 `id`，例如 `bus:1-port:5`，同时传入 `--bus 1 --port 5`。只给一个值时会按未指定位置处理。

## 烧录标准固件包

明确指定 `--mode`。**省略模式默认 `full_erase`，会擦除整个目标存储。** 全盘擦除或裸盘直刷前，确认用户已经授权该目标和擦除范围，并核对板型、固件和存储介质。

| 模式 | 擦除与写入 |
| --- | --- |
| `keep_data` | 请求保留数据，跳过 `udisk`、`private` 和 `reserve`，写入其余有镜像的分区、分区表与启动组件。 |
| `partition` | 跳过单独的擦除步骤，按指定名单写入有镜像的分区，仍下发分区表和启动组件。先用 `inspect` 核对固件中的分区名，每个分区传一个 `--partition`。 |
| `partition_erase` | 按分区布局和介质执行擦除，无镜像的分区也可能受影响，再写入有镜像的分区、分区表与启动组件。 |
| `full_erase` | 强制擦除整个目标存储，再写入有镜像的分区、分区表与启动组件。 |

`--partition` 仅在 `--mode partition` 下筛选。**使用 `partition` 模式但省略分区名单时，会写入所有有镜像的分区。**

```bash
openixsuit-ai flash "/path/to/firmware.img" \
  --mode keep_data --verify true --post-action reboot

openixsuit-ai flash "/path/to/firmware.img" \
  --mode partition --partition boot --partition system --verify true
```

写入后校验默认开启。烧录后默认 `reboot`，可用 `--post-action poweroff` 或 `shutdown` 关机。桌面端显示进度，命令返回 `{"ok":true}` 表示完成。

## 烧录裸盘镜像

第一个参数是与板型匹配的 IMAGEWTY 引导固件，第二个参数是要写入的整盘镜像。**`raw` 会擦除目标存储，跳过下载校验，完成后重启。**

```bash
openixsuit-ai raw "/path/to/boot_fw.img" "/path/to/disk.img" \
  --mode logic-offset --storage sdmmc
```

`--mode` 可用 `logic-offset` 或 `command`。`--storage` 可用 `sdmmc`、`ufs` 或 `nor`，用来选择默认逻辑偏移。板级配置要求具体偏移时，使用 `--logic-offset`，单位为 512 字节扇区。

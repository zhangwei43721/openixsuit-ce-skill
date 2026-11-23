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

先运行 `openixsuit-ai status`。如果要做启动核验，在目标板仍在线时先记录 `adb_devices` 中该设备的 `target`。有同序列号设备时使用带 USB 路径的 `target`，例如 `adb-usb:usb:1-3#20080411`。

通过板卡的进入流程切换到 FEL/FES，也可以对支持 ADB 切换的设备使用 `openixsuit-ai reboot-efex --serial "$adb_target"`。再次查询状态，等 `busy` 为 `null`，在 `fel_devices` 核对目标 `id`。例如 `bus:1-port:5` 对应 `--bus 1 --port 5`。传总线号和端口号时需成对传入。连接多台 FEL/FES 设备时，标准刷写要求指定目标。

## 烧录标准固件包

必须传 `--mode`。全盘擦除用 `--mode full_erase`，执行前核对获准擦除的设备、存储介质和固件。

| 模式 | 擦除与写入 |
| --- | --- |
| `keep_data` | 请求保留数据，跳过 `udisk`、`private` 和 `reserve`，写入其余有镜像的分区、分区表与启动组件。 |
| `partition` | 跳过单独的擦除步骤，写入所列分区的镜像，并下发分区表和启动组件。至少传一个 `--partition`。 |
| `partition_erase` | 按分区布局和介质执行擦除，无镜像的分区也可能受影响，再写入有镜像的分区、分区表与启动组件。 |
| `full_erase` | 强制擦除整个目标存储，再写入有镜像的分区、分区表与启动组件。 |

先生成清单，核对 `erase`、`mbr`、`partitions` 和 `boot_components`。启动组件按设备报告的启动模式和存储类型选用。`ram_staging` 是 FEL 进入 FES 时加载到内存的内容。确认清单以后，把 `plan.id` 传给 `--confirm-plan`，并沿用相同的固件和刷写参数。改换固件包或刷写参数后重新生成清单。

```bash
openixsuit-ai flash "/path/to/firmware.img" \
  --mode partition --partition rootfsB --bus 1 --port 5 --dry-run

openixsuit-ai flash "/path/to/firmware.img" \
  --mode partition --partition rootfsB --bus 1 --port 5 \
  --confirm-plan "$plan_id" --adb-target "$adb_target" --expected-slot b
```

写后校验默认开启，刷写完成后默认重启。需要关机时，传 `--post-action poweroff` 或 `shutdown`。

## 读取烧录和启动结果

`flash.status` 记录写入结果。提供 `--adb-target` 后，`boot.adbReconnect` 记录指定目标是否回来，`boot.bootSlot` 记录实际读到的启动槽位。加 `--expected-slot a` 或 `b` 可以比对槽位。无法识别槽位的设备会给出说明。

要核验正在运行的程序，额外传 `--executable-path /usr/bin/app --expected-sha256 "$expected_hash"`。预期值取自此次发布版本的可执行文件。工具通过 ADB 检查运行进程实际使用的可执行文件，并记录观察到的 SHA-256。设备需允许读取进程的 `/proc/<pid>/exe`。

启动核验默认等 45 秒，可用 `--boot-timeout-ms` 设置。指定的核验失败或超时时，命令退出码为非零。先读 `flash.status` 判断写入是否已经完成，再读 `boot` 确定是哪项启动检查失败。没指定 `--adb-target` 或刷写后关机时，`boot.status` 为 `skipped`。

## 烧录裸盘镜像

第一个参数是与板型匹配的 IMAGEWTY 引导固件，第二个参数是要写入的整盘镜像。**`raw` 会擦除目标存储，跳过下载校验，完成后重启。**

```bash
openixsuit-ai raw "/path/to/boot_fw.img" "/path/to/disk.img" \
  --mode logic-offset --storage sdmmc
```

`--mode` 可用 `logic-offset` 或 `command`。`--storage` 可用 `sdmmc`、`ufs` 或 `nor`，用来选择默认逻辑偏移。板级配置要求具体偏移时，使用 `--logic-offset`，单位为 512 字节扇区。

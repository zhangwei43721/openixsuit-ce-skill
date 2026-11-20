# 设备调试

## ADB 调试和文件传输

用 `status` 确认目标 ADB 序列号并填入 `adb_serial`，再替换下面的文件路径。通过客户端推送文件时，桌面端的 ADB 文件管理器会刷新目录。

```bash
openixsuit-ai adb-shell --serial "$adb_serial" "uname -a"
openixsuit-ai adb-push --serial "$adb_serial" "/local/path/app" "/tmp/app"
openixsuit-ai adb-pull --serial "$adb_serial" "/etc/config/system" "./system.bak"
```

需要访问仅 root 可读的资源且设备支持时，可用 `openixsuit-ai adb-root --serial "$adb_serial"` 重启 `adbd`，再检查设备是否已重新在线。

## 性能快照

`openixsuit-ai metrics --serial "$adb_serial"` 读取一次设备状态。需要 CPU 使用率、网络或磁盘速率时，加 `--sample-ms 1000`，让客户端间隔一秒采两次。

## FEL 内存调试

FEL 内存命令使用首个检测到的设备。确认 `fel_devices` 里只有目标板后，按字节读取内存。地址支持十进制和 `0x` 十六进制。

```bash
openixsuit-ai efel-read 0x001003c8 64
```

`efel-write` 会写入内存，`efel-exec` 会跳转并执行代码。执行前确认已获得对目标设备、地址和动作的授权。用 `openixsuit-ai efel-write --help` 或 `openixsuit-ai efel-exec --help` 查看参数。

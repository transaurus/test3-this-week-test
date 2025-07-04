---
title: Simulate Disk Faults
---

本文檔說明如何使用 Chaosd 模擬磁碟故障。此功能可協助您模擬磁碟讀寫負載（透過 [dd](https://man7.org/linux/man-pages/man1/dd.1.html)）或磁碟填充（透過 [dd](https://man7.org/linux/man-pages/man1/dd.1.html) 或 [fallocate](https://man7.org/linux/man-pages/man1/fallocate.1.html)）。

## 使用命令列模式建立實驗

本節說明如何使用命令列模式建立磁碟故障實驗。

在建立實驗前，您可以執行以下命令查看 Chaosd 支援的磁碟故障類型：

```bash
chaosd attack disk -h
```

執行結果如下：

```bash
disk attack related command

Usage:
  chaosd attack disk [command]

Available Commands:
  add-payload add disk payload
  fill        fill disk

Flags:
  -h, --help   help for disk

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'

Use "chaosd attack disk [command] --help" for more information about a command.
```

目前 Chaosd 支援建立磁碟讀取負載實驗、磁碟寫入負載實驗與磁碟填充實驗。

### 使用命令列模式模擬磁碟讀取負載

模擬磁碟讀取負載屬於一次性操作，因此實驗無需手動恢復。

#### 模擬磁碟讀取負載命令

```bash
chaosd attack disk add-payload read -h
```

執行結果如下：

```bash
read payload

Usage:
  chaosd attack disk add-payload read [flags]

Flags:
  -h, --help                help for read
  -p, --path string         'path' specifies the location to read data.If path not provided, payload will read from disk mount on "/"
  -n, --process-num uint8   'process-num' specifies the number of process work on reading , default 1, only 1-255 is valid value (default 1)
  -s, --size string         'size' specifies how many units of data will read from the file path.'unit' specifies the unit of data, support c=1, w=2, b=512, kB=1000, K=1024, MB=1000*1000,M=1024*1024, , GB=1000*1000*1000, G=1024*1024*1024 BYTESexample : 1M | 512kB

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
```

#### 模擬磁碟讀取負載配置說明

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| `path` | p | Specifies the file path to read the data. If this parameter is not specified, or the parameter value is set to an empty string, Chaosd reads from the virtual disk files mounted in the "/" directory. Depending on the permissions to read the files, you might be required to run this program using certain permissions. | type: string; default: `""` |
| `process-num` | n | Specifies the number of concurrently running [dd](https://man7.org/linux/man-pages/man1/dd.1.html) programs to be used. | type: uint8; default: `1`; range: `1` to `255` |
| `size` | s | Specifies the volume of data to be read. It is the total size of data that dd reads. | type: string; default: `""`; **required**; legal form: the combination of an integer and a unit. For example, 1M, 512kB. Supported units are c=1, w=2, b=512, kB=1000, K=1024, MB=1000\*1000, M=1024\*1024, GB=1000\*1000\*1000, G=1024\*1024\*1024\*1024 BYTE and so on. |

#### 模擬磁碟讀取負載範例

```bash
chaosd attack disk add-payload read -s 1000G -n 7 -p /dev/zero
```

執行結果如下：

```bash
andrew@LAPTOP-NUS30NQD:~/chaosd/bin$ ./chaosd attack disk add-payload read -s 1000G -n 7 -p /dev/zero
[2021/05/20 13:54:31.323 +08:00] [INFO] [disk.go:128] ["5242880+0 records in\n5242880+0 records out\n5242880 bytes (5.2 MB, 5.0 MiB) copied, 4.13252 s, 1.3 MB/s\n"]
[2021/05/20 13:54:46.977 +08:00] [INFO] [disk.go:147] ["146285+0 records in\n146285+0 records out\n153390940160 bytes (153 GB, 143 GiB) copied, 15.6513 s, 9.8 GB/s\n"]
[2021/05/20 13:54:47.002 +08:00] [INFO] [disk.go:147] ["146285+0 records in\n146285+0 records out\n153390940160 bytes (153 GB, 143 GiB) copied, 15.6762 s, 9.8 GB/s\n"]
[2021/05/20 13:54:47.004 +08:00] [INFO] [disk.go:147] ["146285+0 records in\n146285+0 records out\n153390940160 bytes (153 GB, 143 GiB) copied, 15.6777 s, 9.8 GB/s\n"]
[2021/05/20 13:54:47.015 +08:00] [INFO] [disk.go:147] ["146285+0 records in\n146285+0 records out\n153390940160 bytes (153 GB, 143 GiB) copied, 15.6899 s, 9.8 GB/s\n"]
[2021/05/20 13:54:47.018 +08:00] [INFO] [disk.go:147] ["146285+0 records in\n146285+0 records out\n153390940160 bytes (153 GB, 143 GiB) copied, 15.6914 s, 9.8 GB/s\n"]
[2021/05/20 13:54:47.051 +08:00] [INFO] [disk.go:147] ["146285+0 records in\n146285+0 records out\n153390940160 bytes (153 GB, 143 GiB) copied, 15.7254 s, 9.8 GB/s\n"]
[2021/05/20 13:54:47.074 +08:00] [INFO] [disk.go:147] ["146285+0 records in\n146285+0 records out\n153390940160 bytes (153 GB, 143 GiB) copied, 15.7487 s, 9.7 GB/s\n"]
Read file /dev/zero successfully, uid: 4bc9b74a-5fe2-4038-b4f2-09ae95b57694
```

### 使用命令列模式模擬磁碟寫入負載

#### 模擬磁碟寫入負載命令

```bash
chaosd attack disk add-payload write -h
```

執行結果如下：

```bash
write payload

Usage:
  chaosd attack disk add-payload write [flags]

Flags:
  -h, --help                help for write
  -p, --path string         'path' specifies the location to fill data in.If path not provided, payload will write into a temp file, temp file will be deleted after writing
  -n, --process-num uint8   'process-num' specifies the number of process work on writing , default 1, only 1-255 is valid value (default 1)
  -s, --size string         'size' specifies how many units of data will write into the file path.'unit' specifies the unit of data, support c=1, w=2, b=512, kB=1000, K=1024, MB=1000*1000,M=1024*1024, , GB=1000*1000*1000, G=1024*1024*1024 BYTESexample : 1M | 512kB

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
```

#### 模擬磁碟寫入負載配置說明

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| `path` | p | Specifies the file path to write the data. If this parameter is not specified, or the parameter value is set to an empty string, a temporary file will be created in the program execution directory. Depending on the permissions to write the files, you might be required to run this program using certain permissions. | type: string; default: `""` |
| `process-num` | n | Specifies the number of concurrently running [dd](https://man7.org/linux/man-pages/man1/dd.1.html) programs to be used. | type: uint8; default: `1`; range: `1` to `255` |
| `size` | s | Specifies the volume of data to be written. It is the total size of data that dd writes. | type: string; default: `""`; **required**; legal form: the combination of an integer and a unit. For example, 1M, 512kB. Supported units are c=1, w=2, b=512, kB=1000, K=1024, MB=1000\*1000, M=1024\*1024, GB=1000\*1000\*1000, G=1024\*1024\*1024\*1024 BYTE and so on. |

#### 模擬磁碟寫入負載範例

```bash
chaosd attack disk add-payload write -s 2G -n 8
```

執行結果如下：

```bash
[2021/05/20 14:28:14.452 +08:00] [INFO] [disk.go:128] ["0+0 records in\n0+0 records out\n0 bytes copied, 4.3e-05 s, 0.0 kB/s\n"]
[2021/05/20 14:28:16.793 +08:00] [INFO] [disk.go:147] ["256+0 records in\n256+0 records out\n268435456 bytes (268 MB, 256 MiB) copied, 2.32841 s, 115 MB/s\n"]
[2021/05/20 14:28:16.793 +08:00] [INFO] [disk.go:147] ["256+0 records in\n256+0 records out\n268435456 bytes (268 MB, 256 MiB) copied, 2.3344 s, 115 MB/s\n"]
[2021/05/20 14:28:16.793 +08:00] [INFO] [disk.go:147] ["256+0 records in\n256+0 records out\n268435456 bytes (268 MB, 256 MiB) copied, 2.33312 s, 115 MB/s\n"]
[2021/05/20 14:28:16.793 +08:00] [INFO] [disk.go:147] ["256+0 records in\n256+0 records out\n268435456 bytes (268 MB, 256 MiB) copied, 2.33466 s, 115 MB/s\n"]
[2021/05/20 14:28:16.793 +08:00] [INFO] [disk.go:147] ["256+0 records in\n256+0 records out\n268435456 bytes (268 MB, 256 MiB) copied, 2.33189 s, 115 MB/s\n"]
[2021/05/20 14:28:16.793 +08:00] [INFO] [disk.go:147] ["256+0 records in\n256+0 records out\n268435456 bytes (268 MB, 256 MiB) copied, 2.33752 s, 115 MB/s\n"]
[2021/05/20 14:28:16.793 +08:00] [INFO] [disk.go:147] ["256+0 records in\n256+0 records out\n268435456 bytes (268 MB, 256 MiB) copied, 2.33295 s, 115 MB/s\n"]
[2021/05/20 14:28:16.794 +08:00] [INFO] [disk.go:147] ["256+0 records in\n256+0 records out\n268435456 bytes (268 MB, 256 MiB) copied, 2.3359 s, 115 MB/s\n"]
Write file /home/andrew/chaosd/bin/example255569279 successfully, uid: e66afd86-6f3e-43a0-b161-09447ed84856
```

### 使用命令列模式模擬磁碟填充

#### 模擬磁碟填充命令

```bash
chaosd attack disk fill -h
```

執行結果如下：

```bash
fill disk

Usage:
  chaosd attack disk fill [flags]

Flags:
  -d, --destroy          destroy file after filled in or allocated
  -f, --fallocate        fill disk by fallocate instead of dd (default true)
  -h, --help             help for fill
  -p, --path string      'path' specifies the location to fill data in.If path not provided, a temp file will be generated and deleted immediately after data filled in or allocated
  -c, --percent string   'percent' how many percent data of disk will fill in the file path
  -s, --size string      'size' specifies how many units of data will fill in the file path.'unit' specifies the unit of data, support c=1, w=2, b=512, kB=1000, K=1024, MB=1000*1000,M=1024*1024, , GB=1000*1000*1000, G=1024*1024*1024 BYTESexample : 1M | 512kB

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
```

#### 模擬磁碟填充配置說明

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| `destroy` | d | If this parameter is set to `true,` the fill file is immediately deleted after being filled. | type: bool; default: `false` |
| `fallocate` | f | If this parameter is set to `true`, Linux is used to call fallocate to quickly apply for disk space and size must be greater than `0`. If this parameter is set to `false`, Linux is used to call dd to fill disks at a relatively slow pace. | type: bool; default: `true` |
| `path` | p | Specifies the file path to write the data. If this parameter is not specified, or the parameter value is set to an empty string, a temporary file will be created in the program execution directory. Depending on the permissions to write the files, you might be required to run this program using certain permissions. | type: string; default: `""` |
| `percent` | c | Specifies the percentage of disk size to be filled. | type: string; default: `""`; positive integer of the uint type is acceptable; You must set one of `size` or `percent` (both items **cannot** be `""` at the same time). |
| `size` | s | Specifies the volume of data to be written. | type: string; default: `""`; legal form: the combination of an integer and a unit. For example, 1M, 512kB. Supported units are c=1, w=2, b=512, kB=1000, K=1024, MB=1000\*1000, M=1024\*1024, GB=1000\*1000\*1000, G=1024\*1024\*1024\*1024 BYTE and so on. You must set one of `size` or `percent` (both items **cannot** be `""` at the same time). |

#### 模擬磁碟填充範例

```bash
chaosd attack disk fill -c 50 -d
```

執行結果如下：

```bash
[2021/05/20 14:30:02.943 +08:00] [INFO] [disk.go:215]
Fill file /home/andrew/chaosd/bin/example623832242 successfully, uid: 097b4214-8d8e-46ad-8768-c3e0d8cbb326
```

## 使用服務模式建立實驗

本節說明如何使用服務模式建立磁碟故障實驗。

### 使用服務模式模擬磁碟讀取負載

模擬磁碟讀取負載屬於一次性操作，因此實驗無需手動恢復。

#### 模擬磁碟讀取負載參數

| Parameter | Description | Value |
| :-- | :-- | :-- |
| `action` | Actions of the experiment | Set to `"read-payload"` |
| `path` | Specifies the file path to read the data. If this parameter is not specified, or the parameter value is set to an empty string, Chaosd reads from the virtual disk files mounted in the "/" directory. Depending on the permissions to read the files, you might be required to run this program using certain permissions. | type: string; default: `"""` |
| `payload-process-num` | Specifies the number of concurrently running [dd](https://man7.org/linux/man-pages/man1/dd.1.html) programs to be used. | type: uint8; default: `1`; range: `1` to `255` |
| `size` | Specifies the volume of data to be read. It is the total size of data that dd reads. | type: string; default: `""`; **required**; legal form: the combination of an integer and a unit. For example, 1M, 512kB. Supported units are c=1, w=2, b=512, kB=1000, K=1024, MB=1000\*1000, M=1024\*1024, GB=1000\*1000\*1000, G=1024\*1024\*1024\*1024 BYTE and so on. |

#### 使用服務模式模擬磁碟讀取負載範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/disk -H "Content-Type:application/json" -d '{"action":"read-payload","path":"/dev/zero", "payload-process-num":7,"size":"1000G"}'
```

執行結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"a551206c-960d-4ac5-9056-518e512d4d0d"}
```

### 使用服務模式模擬磁碟寫入負載

#### 模擬磁碟寫入負載參數

| Parameter | Description | Value |
| :-- | :-- | :-- |
| `action` | Actions of the experiment | Set to `"write-payload"` |
| `path` | Specifies the file path to write the data. If this parameter is not specified, or the parameter value is set to an empty string, a temporary file will be created in the program execution directory. Depending on the permissions to write the files, you might be required to run this program using certain permissions. | type: string; default: `""` |
| `payload-process-num` | Specifies the number of concurrently running [dd](https://man7.org/linux/man-pages/man1/dd.1.html) programs to be used. | type: uint8; default: `1`; range: `1` to `255` |
| `size` | Specifies the volume of data to be written. It is the total size of data that dd writes. | type: string; default: `""`; **required**; legal form: the combination of an integer and a unit. For example, 1M, 512kB. Supported units are c=1, w=2, b=512, kB=1000, K=1024, MB=1000\*1000, M=1024\*1024, GB=1000\*1000\*1000, G=1024\*1024\*1024\*1024 BYTE and so on. |

#### 使用服務模式模擬磁碟寫入負載範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/disk -H "Content-Type:application/json" -d '{"action":"write-payload","path":"/tmp/test", "payload-process-num":7,"size":"1000G"}'
```

執行結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"a551206c-960d-4ac5-9056-518e512d4d0d"}
```

### 使用服務模式模擬磁碟填充

#### 模擬磁碟填充的參數

| Parameter | Description | Value |
| :-- | :-- | :-- |
| `action` | Actions of the experiment | Set to `"fill"` |
| `destroy` | If this parameter is set to `true,` the fill file is immediately deleted after being filled. | type: bool; default: `false` |
| `fill-by-fallocate` | If this parameter is set to `true`, Chaosd uses Linux to call `fallocate` to apply for disk space quickly, and you must set `size` to a value greater than `0`. If this parameter is set to `false`, Chaosd uses Linux to call dd to fill disks at a relatively slow pace. | type: bool; default: `true` |
| `path` | Specifies the file path to write the data. If this parameter is not specified, or the parameter value is set to an empty string, a temporary file will be created in the program execution directory. Depending on the permissions to write the files, you might be required to run this program using certain permissions. | type: string; default: `""` |
| `percent` | Specifies the percentage of disk size to be filled. | type: string; default: ""; positive integer of the uint type is acceptable; You must set one of `size` or `percent` (both items **cannot** be `""` at the same time). |
| `size` | Specifies the volume of data to be read. | type: string; default: `""`; legal form: the combination of an integer and a unit. For example, 1M, 512kB. Supported units are c=1, w=2, b=512, kB=1000, K=1024, MB=1000\*1000, M=1024\*1024, GB=1000\*1000\*1000, G=1024\*1024\*1024\*1024 BYTE and so on. You must set one of `size` or `percent` (both items **cannot** be `""` at the same time). |

#### 使用服務模式模擬磁碟填充的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/disk -H "Content-Type:application/json" -d '{"action":"fill","path":"/tmp/test", "fill-by-fallocate":true,"percent":"50"}'
```

結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"a551206c-960d-4ac5-9056-518e512d4d0d"}
```
---
title: Simulate Stress Scenarios
---

本文件說明如何使用 Chaosd 模擬壓力場景。此功能透過 [stress-ng](https://wiki.ubuntu.com/Kernel/Reference/stress-ng) 在主機上產生 CPU 或記憶體壓力。您可透過命令列模式或服務模式建立壓力實驗。

## 使用命令列模式建立壓力實驗

本節說明如何使用命令列模式建立壓力實驗。

建立壓力實驗前，可執行以下命令查看 Chaosd 支援的壓力實驗類型：

```bash
chaosd attack stress --help
```

結果如下：

```bash
Stress attack related commands

Usage:
  chaosd attack stress [command]

Available Commands:
  cpu         continuously stress CPU out
  mem         continuously stress virtual memory out

Flags:
  -h, --help   help for stress

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'

Use "chaosd attack stress [command] --help" for more information about a command.
```

目前 Chaosd 支援建立 CPU 壓力實驗與記憶體壓力實驗。

### 使用命令列模式模擬 CPU 壓力

#### 模擬 CPU 壓力的命令

欲查看 CPU 壓力模擬支援的設定項目，請執行以下命令：

```bash
chaosd attack stress cpu --help
```

結果如下：

```bash
continuously stress CPU out

Usage:
  chaosd attack stress cpu [options] [flags]

Flags:
  -h, --help              help for cpu
  -l, --load int          Load specifies P percent loading per CPU worker. 0 is effectively a sleep (no load) and 100 is full loading. (default 10)
  -o, --options strings   extend stress-ng options.
  -w, --workers int       Workers specifies N workers to apply the stressor. (default 1)

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
```

#### 模擬 CPU 壓力的設定說明

| Configuration item | Abbreviation | Description | Type | Value |
| :-- | :-- | :-- | :-- | :-- |
| `load` | l | Specifies the percentage of CPU load per CPU worker. `0` means no CPU utilization, and `100` means full CPU utilization. | int | Range: `0` to `100`; Default value: `10`. |
| `workers` | w | Specifies the number of workers used to create CPU stress. | int | Default value: 1. |
| `options` | o | The extended parameter of stress-ng, usually not configured. | string | Default value: "". |

#### 模擬 CPU 壓力的範例

```bash
chaosd attack stress cpu --workers 2 --load 10
```

結果如下：

```bash
[2021/05/12 03:38:33.698 +00:00] [INFO] [stress.go:66] ["stressors normalize"] [arguments=" --cpu 2 --cpu-load 10"]
[2021/05/12 03:38:33.702 +00:00] [INFO] [stress.go:82] ["Start stress-ng process successfully"] [command="/usr/bin/stress-ng --cpu 2 --cpu-load 10"] [Pid=27483]
Attack stress cpu successfully, uid: 4f33b2d4-aee6-43ca-9c43-0f12867e5c9c
```

### 使用命令列模式模擬記憶體壓力

#### 模擬記憶體壓力的命令

欲查看記憶體壓力模擬支援的設定項目，請執行以下命令：

```bash
chaosd attack stress mem --help
```

結果如下：

```bash
continuously stress virtual memory out

Usage:
  chaosd attack stress mem [options] [flags]

Flags:
  -h, --help              help for mem
  -o, --options strings   extend stress-ng options.
  -s, --size string       Size specifies N bytes consumed per vm worker, default is the total available memory. One can specify the size as % of total available memory or in units of B, KB/KiB, MB/MiB, GB/GiB, TB/TiB..

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
```

#### 模擬記憶體壓力的設定說明

| Configuration item | Abbreviation | Description | Type | Value |
| :-- | :-- | :-- | :-- | :-- |
| `size` | s | Specifies the size of memory per VM worker. | string | The memory size in B, KB/KiB, MB/MiB, GB/GiB, TB/TiB. If the size is not set, all available memory is used by default. |
| `options` | o | The extended parameter of stress-ng, usually not configured. | string | Default value: "". |

#### 模擬記憶體壓力的範例

```bash
chaosd attack stress mem --workers 2 --size 100M
```

結果如下：

```bash
[2021/05/12 03:37:19.643 +00:00] [INFO] [stress.go:66] ["stressors normalize"] [arguments=" --vm 2 --vm-keep --vm-bytes 100000000"]
[2021/05/12 03:37:19.654 +00:00] [INFO] [stress.go:82] ["Start stress-ng process successfully"] [command="/usr/bin/stress-ng --vm 2 --vm-keep --vm-bytes 100000000"] [Pid=26799]
Attack stress mem successfully, uid: c2bff2f5-3aac-4ace-b7a6-322946ae6f13
```

執行實驗時，需保存實驗的 uid 資訊。當不需要壓力模擬時，可使用 `recover` 終止與該 uid 相關的實驗：

```bash
chaosd recover c2bff2f5-3aac-4ace-b7a6-322946ae6f13
```

結果如下：

```bash
Recover c2bff2f5-3aac-4ace-b7a6-322946ae6f13 successfully
```

## 使用服務模式建立壓力實驗

使用服務模式建立實驗，請依循以下指示：

1. 在服務模式下執行 Chaosd：

   ```bash
   chaosd server --port 31767
   ```

2. Send a `POST` HTTP request to the `/api/attack/{uid}` path of Chaosd service.

   ```bash
   curl -X POST 172.16.112.130:31767/api/attack/jvm -H "Content-Type:application/json" -d '{fault-configuration}'
   ```

   For the `fault-configuration` part in the above command, you need to configure it according to the fault types. For the corresponding parameters, refer to the parameters and examples of each fault type in the following sections.

:::note

When running an experiment, remember to save the UID information of the experiment. When you want to kill the experiment corresponding to the UID, you need to send a `DELETE` HTTP request to the `/api/attack/{uid}` path of Chaosd service.

:::

### 使用服務模式模擬 CPU 壓力

#### 模擬 CPU 壓力的參數

| Parameter | Description | Type | Value |
| :-- | :-- | :-- | :-- |
| `action` | Actions of the experiment |  | Set to "cpu" |
| `load` | Specifies the percentage of CPU load per CPU worker. `0` means no CPU utilization, and `100` means full CPU utilization. | int | Range: `0` to `100`; Default value: `10` |
| `workers` | Specifies the number of workers used to create CPU stress | int | Default value: `1` |
| `options` | The extended parameter of stress-ng, usually not configured. | string | Default value: "" |

#### 使用服務模式模擬 CPU 壓力的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/stress -H "Content-Type:application/json" -d '{"load":10, "action":"cpu","workers":1}'
```

結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"c3c519bf-819a-4a7b-97fb-e3d0814481fa"}
```

### 使用服務模式模擬記憶體壓力

#### 模擬記憶體壓力的參數

| Parameter | Description | Type | Value |
| :-- | :-- | :-- | :-- |
| `action` | Actions of the experiment |  | Set to "mem" |
| `size` | Specifies the size of memory per VM worker | string | the memory size in B, KB/KiB, MB/MiB, GB/GiB, TB/TiB. If the size is not set, all available memory is used by default. |
| `options` | The extended parameter of stress-ng, usually not configured. | string | Default value: "" |

#### 使用服務模式模擬記憶體壓力的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/stress -H "Content-Type:application/json" -d '{"size":"100M", "action":"mem"}'
```

結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"a551206c-960d-4ac5-9056-518e512d4d0d"}
```
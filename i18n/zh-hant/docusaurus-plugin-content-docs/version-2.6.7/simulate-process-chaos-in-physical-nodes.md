---
title: Simulate Process Faults
---

本文件說明如何使用 Chaosd 模擬程序故障。程序故障利用 `kill` 命令的 Golang 介面，模擬程序被終止或停止的場景。您可透過命令列模式或服務模式建立實驗。

## 使用命令列模式建立實驗

建立實驗前，可執行以下命令查看 Chaosd 支援的程序故障類型：

```bash
chaosd attack process -h
```

執行結果如下：

```bash
Process attack related commands

Usage:
  chaosd attack process [command]

Available Commands:
  kill        kill process, default signal 9
  stop        stop process, this action will stop the process with SIGSTOP

Flags:
  -h, --help   help for process

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'

Use "chaosd attack process [command] --help" for more information about a command.
```

目前 Chaosd 支援模擬程序被終止或停止。

### 使用命令列模式終止程序

#### 終止程序命令

```bash
chaosd attack process kill -h
```

執行結果如下：

```bash
kill process, default signal 9

Usage:
  chaosd attack process kill [flags]

Flags:
  -h, --help                 help for kill
  -p, --process string       The process name or the process ID
  -r, --recover-cmd string   The command to be run when recovering experiment
  -s, --signal int           The signal number to send (default 9)

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID
```

#### 終止程序配置說明

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| `process` | p | The name or the identifier of the process to be injected faults | string; the default value is `""`. |
| `recover-cmd` | r | The command to be run when recovering experiment | string; the default value is `""`. |
| `signal` | s | The provided value of the process signal | int; the default value is `9`. Currently, only `SIGKILL`, `SIGTERM`, and `SIGSTOP` are supported. |

#### 終止程序範例

```bash
chaosd attack process kill -p python
```

執行結果如下：

```bash
Attack process python successfully, uid: 10e633ac-0a37-41ba-8b4a-cd5ab92099f9
```

:::note

僅 `signal` 設為 `SIGSTOP` 的實驗可被恢復。

:::

### 使用命令列模式停止程序

#### 停止程序命令

```bash
chaosd attack process stop -h
```

執行結果如下：

```bash
stop process, this action will stop the process with SIGSTOP

Usage:
  chaosd attack process stop [flags]

Flags:
  -h, --help             help for stop
  -p, --process string   The process name or the process ID

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
```

#### 停止程序配置說明

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| `process` | p | The name or the identifier of the process to be stopped | string; the default value is `""`. |

#### 停止程序範例

```bash
chaosd attack process stop -p python
```

執行結果如下：

```bash
Attack process python successfully, uid: 9cb6b3be-4f5b-4ecb-ae05-51050fcd0010
```

## 使用服務模式建立實驗

透過服務模式建立實驗，請依循以下指示：

1. 以服務模式執行 Chaosd：

   ```bash
   chaosd server --port 31767
   ```

2. Send a `POST` HTTP request to the `/api/attack/process` path of the Chaosd service.

   ```bash
   curl -X POST 172.16.112.130:31767/api/attack/process -H "Content-Type:application/json" -d '{fault-configuration}'
   ```

   In the above command, you need to configure `fault-configuration` according to the fault types. For the corresponding parameters, refer to the parameters and examples of each fault type in the following sections.

:::note

When running an experiment, remember to record the UID of the experiment. When you want to end the experiment corresponding to the UID, you need to send a `DELETE` HTTP request to the `/api/attack/{uid}` path of the Chaosd service.

:::

### 使用服務模式模擬程序故障

#### 模擬程序故障參數

| Parameter | Description                                                     | Value                              |
| :-------- | :-------------------------------------------------------------- | :--------------------------------- |
| `process` | The name or the identifier of the process to be injected faults | string; the default value is `""`. |
| `signal`  | The provided value of the process signal                        | int; the default value is `9`      |

#### 服務模式模擬程序故障範例

##### 終止程序

```bash
curl -X POST 172.16.112.130:31767/api/attack/process -H "Content-Type:application/json" -d '{"process":"12345","signal":15}'
```

執行結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"c3c519bf-819a-4a7b-97fb-e3d0814481fa"}
```

##### 停止程序

```bash
curl -X POST 172.16.112.130:31767/api/attack/process -H "Content-Type:application/json" -d '{"process":"12345","signal":19}'
```

執行結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"a00cca2b-eba7-4716-86b3-3e66f94880f7"}
```

:::note

僅 `signal` 設為 `SIGSTOP` 的實驗可被恢復。

:::
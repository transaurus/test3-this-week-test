---
title: Search and Recover Experiments of Chaosd
summary: Describes how to search and recover the experiments of Chaosd, and provide related examples.
---

您可透過 Chaosd 依據條件搜尋實驗，並恢復指定 UID 對應的實驗。本文說明如何搜尋與恢復 Chaosd 實驗，並提供相關範例。

## 搜尋 Chaosd 實驗

本節介紹如何透過命令列模式與服務模式搜尋 Chaosd 實驗。

### 使用命令列模式搜尋實驗

執行以下命令可查看 `search` 指令支援的配置參數：

```bash
$ chaosd search --help
Search chaos attack, you can search attacks through the uid or the state of the attack

Usage:
  chaosd search UID [flags]

Flags:
  -A, --all             list all chaos attacks
      --asc             order by CreateTime, default value is false that means order by CreateTime desc
  -h, --help            help for search
  -k, --kind string     attack kind, supported value: network, process, stress, disk, host, jvm
  -l, --limit uint32    limit the count of attacks
  -o, --offset uint32   starting to search attacks from offset
  -s, --status string   attack status, supported value: created, success, error, destroyed, revoked

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
```

#### 配置說明

| Configuration item | Abbreviation | Description | Type |
| :-- | :-- | :-- | :-- |
| `all` | A | Lists all experiments | bool |
| `asc` | None | Sorts the experiments in ascending order of the creation time. The default value is `false`. | bool |
| `kind` | k | Lists experiments of the specified kind | string. The supported kinds are as follows: `network`, `process`, `stress`, `disk`, `host`, `jvm` |
| `limit` | l | The number of listed experiments | int |
| `offset` | o | Searches from the specified offset | int |
| `status` | s | Lists experiments with the specified status | string. The supported types are as follows: `created`, `success`, `error`, `destroyed`, `revoked` |

#### 範例

```bash
./chaosd search --kind network --status destroyed --limit 1
```

By running this command, you can search the experiments of the kind of `network` in the status of `destroyed` (indicating that the experiment has been recovered).

執行命令後，結果僅輸出一行數據：

```bash
                  UID                     KIND     ACTION    STATUS            CREATE TIME                                                                                                                  CONFIGURATION
--------------------------------------- --------- -------- ----------- --------------------------- ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  1f6c1253-522a-43d9-83f8-42607102b3b9   network   delay    destroyed   2021-11-02T15:14:07+08:00   {"schedule":"","duration":"","action":"delay","kind":"network","uid":"1f6c1253-522a-43d9-83f8-42607102b3b9","latency":"2s","jitter":"0ms","correlation":"0","device":"eth0","ip-address":"220.181.38.251","ip-protocol":"all"}
```

### 使用服務模式搜尋實驗

目前服務模式僅支援搜尋全部實驗，可透過訪問 Chaosd 服務的 `/api/experiments/` 路徑獲取數據。

#### 範例

```bash
curl -X GET 127.0.0.1:31767/api/experiments/
```

結果如下：

```bash
[{"id":1,"uid":"ddc5ca81-b677-4595-b691-0ce57bedb156","created_at":"2021-10-18T16:01:18.563542491+08:00","updated_at":"2021-10-18T16:07:27.87111393+08:00","status":"success","kind":"stress","action":"mem","recover_command":"{\"schedule\":\"\",\"duration\":\"\",\"action\":\"mem\",\"kind\":\"stress\",\"uid\":\"ddc5ca81-b677-4595-b691-0ce57bedb156\",\"Load\":0,\"Workers\":0,\"Size\":\"100MB\",\"Options\":null,\"StressngPid\":0}","launch_mode":"svr"}]
```

## 恢復 Chaosd 實驗

建立實驗後若需撤銷其影響，可使用實驗恢復功能。

### 使用命令列模式恢復實驗

可透過 Chaosd recover UID 恢復實驗。

以下範例展示使用命令列模式恢復實驗：

1. 使用 Chaosd 建立 CPU 壓力實驗：

   ```bash
   chaosd attack stress cpu --workers 2 --load 10
   ```

   結果如下：

   ```bash
   [2021/05/12 03:38:33.698 +00:00] [INFO] [stress.go:66] ["stressors normalize"] [arguments=" --cpu 2 --cpu-load 10"]
   [2021/05/12 03:38:33.702 +00:00] [INFO] [stress.go:82] ["Start stress-ng process successfully"] [command="/usr/bin/stress-ng --cpu 2 --cpu-load 10"] [Pid=27483]
   Attack stress cpu successfully, uid: 4f33b2d4-aee6-43ca-9c43-0f12867e5c9c
   ```

   保存實驗 UID 供後續使用。

2. 當不再需要模擬 CPU 壓力場景時，使用 `recover` 命令恢復對應 UID 的實驗：

   ```bash
   chaosd recover 4f33b2d4-aee6-43ca-9c43-0f12867e5c9c
   ```

### 使用服務模式恢復實驗

You can recover an experiment by sending a `DELETE` HTTP request to the `/api/attack/{uid}` path of Chaosd service.

以下範例展示使用服務模式恢復實驗：

1. 向 Chaosd 服務發送 `POST` HTTP 請求建立 CPU 壓力實驗：

   ```bash
   curl -X POST 172.16.112.130:31767/api/attack/stress -H "Content-Type:application/json" -d '{"load":10, "action":"cpu","workers":1}'
   ```

   結果如下：

   ```bash
   {"status":200,"message":"attack successfully","uid":"c3c519bf-819a-4a7b-97fb-e3d0814481fa"}
   ```

   保存實驗 UID 供後續使用。

2. 當您不再需要模擬 CPU 壓力情境時，執行以下指令來復原對應 UID 的實驗：

   ```bash
   curl -X DELETE 172.16.112.130:31767/api/attack/c3c519bf-819a-4a7b-97fb-e3d0814481fa
   ```
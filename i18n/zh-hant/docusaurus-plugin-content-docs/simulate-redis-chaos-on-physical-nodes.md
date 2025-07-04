---
title: Simulate Redis Faults
---

本文件介紹如何使用 Chaosd 模擬 Redis 故障。此功能使用 `go-redis` 套件中的 Golang 介面以及 `redis-server` 命令列工具。您可以透過命令列模式或服務模式建立實驗。

## 使用命令列模式建立實驗

在建立實驗之前，您可以執行以下命令來查看 Chaosd 支援的 Redis 故障類型：

```bash
chaosd attack redis -h
```

結果如下：

```bash
Redis attack related commands

Usage:
  chaosd attack redis [command]

Available Commands:
  cache-expiration  expire keys in Redis
  cache-limit       set maxmemory of Redis
  cache-penetration penetrate cache
  sentinel-restart  restart sentinel
  sentinel-stop     stop sentinel

Flags:
  -h, --help   help for redis

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID

Use "chaosd attack redis [command] --help" for more information about a command.
```

目前，Chaosd 支援模擬快取過期、快取穿透、快取限制、Sentinel 重啟以及 Sentinel 停止。

### 使用命令列模式模擬快取過期

此命令的意義與 Redis 中的 EXPIRE 相同。更多詳細資訊，請參閱 [Redis 官方文件](https://redis.io/commands/expire/)。

:::note

目前，Chaosd 不支援復原已執行 `cache-expiration` 的鍵，因此如果您希望復原，請事先備份。

:::

#### 快取過期的命令

```bash
chaosd attack redis cache-expiration -h
```

結果如下：

```bash
expire keys in Redis

Usage:
  chaosd attack redis cache-expiration [flags]

Flags:
  -a, --addr string         The address of redis server
      --expiration string   The expiration of the key. A expiration string should be able to be converted to a time duration, such as "5s" or "30m" (default "0")
  -h, --help                help for cache-expiration
  -k, --key string          The key to be set a expiration, default expire all keys
      --option string       The additional options of expiration, only NX, XX, GT, LT supported
  -p, --password string     The password of server

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID
```

#### 快取過期的設定描述

| Configuration item | Abbreviation | Type | Description | Value |
| :-- | :-- | :-- | :-- | :-- |
| `addr` | a | string | The address and port of Redis server to be injected into the fault, for example `127.0.0.1:6379` | Default value: `""` |
| `expiration` | None | string | The specified key will be expired after `expiration` arrives | Default value: `"0"`. Make sure that the string is in the format supported by `time.Duration` |
| `key` | k | string | The key to be expired | Default value: `""`, which means the expiration is set for all keys |
| `option` | None | string | Additional options for `expiration`. **Only versions of Redis after 7.0.0 support this flag** | Default value: `""`. Only NX, XX, GT, and LT are supported |
| `password` | p | string | The password to log in to the server | Default value: `""` |

#### 模擬快取過期的範例

```bash
chaosd attack redis cache-expiration -a 127.0.0.1:6379 --option GT --expiration 1m
```

### 使用命令列模式模擬快取限制

#### 快取限制的命令

```bash
chaosd attack redis cache-limit -h
```

結果如下：

```bash
set maxmemory of Redis

Usage:
  chaosd attack redis cache-limit [flags]

Flags:
  -a, --addr string       The address of redis server
  -h, --help              help for cache-limit
  -p, --password string   The password of server
      --percent string    The percentage of maxmemory
  -s, --size string       The size of cache (default "0")

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID
```

#### 快取限制的設定描述

| Configuration item | Abbreviation | Type | Description | Value |
| :-- | :-- | :-- | :-- | :-- |
| `addr` | a | string | The address and port of Redis server to be injected into the fault, such as `127.0.0.1:6379` | Default value: `""` |
| `password` | p | string | The password to log in to the server | Default value: `""` |
| `percent` | None | string | Specifies `maxmemory` as a percentage of the original value | Default value: `""` |
| `size` | s | string | Specifies the size of `maxmemory` | Default `0`, which means no limitation of memory |

#### 模擬快取限制的範例

```bash
chaosd attack redis cache-limit -a 127.0.0.1:6379 -s 256M
```

### 使用命令列模式模擬快取穿透

此命令將使用 Redis Pipeline 盡快向 Redis 伺服器發送指定數量的 `GET` 請求。由於請求的鍵在 Redis 伺服器上不存在，這些請求將導致快取穿透現象。

#### 快取穿透的命令

```bash
chaosd attack redis cache-penetration -h
```

結果如下：

```bash
penetrate cache

Usage:
  chaosd attack redis cache-penetration [flags]

Flags:
  -a, --addr string       The address of redis server
  -h, --help              help for cache-penetration
  -p, --password string   The password of server
      --request-num int   The number of requests

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID
```

#### 快取穿透的設定描述

| Configuration item | Abbreviation | Type | Description | Value |
| :-- | :-- | :-- | :-- | :-- |
| `addr` | a | string | The address and port of Redis server to be injected into the fault, such as `127.0.0.1:6379` | Default value: `""` |
| `password` | p | string | The password to log in to the server | Default value: `""` |
| `request-num` | None | int | Specifies the number of requests to be sent to the Redis server | Default value: `0` |

#### 模擬快取穿透的範例

```bash
chaosd attack redis cache-penetration -a 127.0.0.1:6379 --request-num 100000
```

### 使用命令列模式模擬 Sentinel 重啟

#### Sentinel 重啟的命令

```bash
chaosd attack redis sentinel-restart -h
```

結果如下：

```bash
restart sentinel

Usage:
  chaosd attack redis sentinel-restart [flags]

Flags:
  -a, --addr string         The address of redis server
  -c, --conf string         The config of Redis server
      --flush-config         Force Sentinel to rewrite its configuration on disk (default true)
  -h, --help                help for sentinel-restart
  -p, --password string     The password of server
      --redis-path string   The path of the redis-server command

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID
```

#### Sentinel 重啟的設定描述

| Configuration item | Abbreviation | Type | Description | Value |
| :-- | :-- | :-- | :-- | :-- |
| `addr` | a | string | The address and port of Sentinel to be injected into the fault, such as `127.0.0.1:26379` | Default value: `""` |
| `conf` | c | string | Specifies the path of Sentinel config file, this file will be used to revover the Sentinel | Default value: `""` |
| `flush-config` | None | bool | Forces Sentinel to rewrite its configuration on disk, including the current Sentinel state | Default value: `true` |
| `password` | p | string | The password to log in to the server | Default value: `""` |
| `redis-path` | None | string | Specifies the path of `redis-server` command-line tool | Default value: `""` |

#### 模擬 Sentinel 重啟的範例

```bash
chaosd attack redis sentinel-restart -a 127.0.0.1:26379 --conf /home/redis-test/sentinel-26379.conf
```

### 使用命令列模式模擬 Sentinel 停止

#### Sentinel 停止的命令

```bash
chaosd attack redis sentinel-stop -h
```

結果如下：

```bash
stop sentinel

Usage:
  chaosd attack redis sentinel-stop [flags]

Flags:
  -a, --addr string         The address of redis server
  -c, --conf string         The config path of Redis server
      --flush-config        Force Sentinel to rewrite its configuration on disk (default true)
  -h, --help                help for sentinel-stop
  -p, --password string     The password of server
      --redis-path string   The path of the redis-server command

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID
```

#### Sentinel 停止的設定描述

| Configuration item | Abbreviation | Type | Description | Value |
| :-- | :-- | :-- | :-- | :-- |
| `addr` | a | string | The address and port of Sentinel to be injected into the fault, such as `127.0.0.1:26379` | Default value: `""` |
| `conf` | c | string | Specifies the path of Sentinel configuration file, which is used to recover the Sentinel | Default value: `""` |
| `flush-config` | None | bool | Forces Sentinel to rewrite its configuration on disk, including the current Sentinel state | Default value: `true` |
| `password` | p | string | The password to log in to the server | Default value: `""` |
| `redis-path` | None | string | Specifies the path of `redis-server` command-line tool | Default value: `""` |

#### 模擬 Sentinel 重啟的範例

```bash
chaosd attack redis sentinel-stop -a 127.0.0.1:26379 --conf /home/redis-test/sentinel-26379.conf
```

## 使用服務模式建立 Redis 故障實驗

要使用服務模式建立實驗，請遵循以下指示：

1. 以服務模式執行 Chaosd：

   ```bash
   chaosd server --port 31767
   ```

2. Send a `POST` HTTP request to the `/api/attack/redis` path of the Chaosd service.

   ```bash
   curl -X POST 127.0.0.1:31767/api/attack/redis -H "Content-Type:application/json" -d '{fault-configuration}'
   ```

   In the above command, you need to configure `fault-configuration` according to the fault types. For the corresponding parameters, refer to the parameters and examples of each fault type in the following sections.

:::note

When running an experiment, remember to record the UID of the experiment. When you want to end the experiment corresponding to the UID, you need to send a `DELETE` HTTP request to the `/api/attack/{uid}` path of the Chaosd service.

:::

### 使用服務模式模擬快取過期

#### 模擬快取過期的參數

| Parameter | Description | Type | Value |
| :-- | :-- | :-- | :-- |
| `action` | Action of the experiment | string | set to "corrupt" |
| `addr` | The address and port of Redis server to be injected into the fault, such as `127.0.0.1:6379` | string | Default value: `""` |
| `expiration` | The specified key will be expired after `expiration` arrives | string | Default value: `"0"`. Make sure that the string is in the format supported by `time.Duration` |
| `key` | The key to be expired | string | Default value: `""`, which means the expiration is set for all keys |
| `option` | Additional options for `expiration`. **Only versions of Redis after 7.0.0 support this flag** | string | Default value: `""`. Only NX, XX, GT, and LT are supported |
| `password` | The password to log in to the server | Default value: `""` |

#### 使用服務模式模擬快取過期的範例

```bash
curl -X POST 127.0.0.1:31767/api/attack/redis -H "Content-Type:application/json" -d '{"action":"expiration", "expiration":"1m","addr":"127.0.0.1:6379"}'
```

### 使用服務模式模擬快取限制

#### 模擬快取限制的參數

| Parameter | Description | Type | Value |
| :-- | :-- | :-- | :-- |
| `action` | Action of the experiment | string | set to "cacheLimit" |
| `addr` | The address and port of Redis server to be injected into the fault, such as `127.0.0.1:6379` | string | Default value: `""` |
| `password` | The password to log in to the server | string | Default value: `""` |
| `percent` | Specifies `maxmemory` as a percentage of the original value | string | Default value: `""` |
| `size` | Specifies the size of `maxmemory` | string | Default `0`, which means no limitation of memory |

#### 使用服務模式模擬快取限制的範例

```bash
curl -X POST 127.0.0.1:31767/api/attack/redis -H "Content-Type:application/json" -d '{"action":"cacheLimit", ""addr":"127.0.0.1:6379", "percent":"50%"}'
```

### 使用服務模式模擬快取穿透

#### 模擬快取穿透的參數

| Parameter | Description | Type | Value |
| :-- | :-- | :-- | :-- |
| `action` | Action of the experiment | string | set to "penetration" |
| `addr` | The address and port of Redis server to be injected into the fault, such as `127.0.0.1:6379` | string | Default value: `""` |
| `password` | The password to log in to the server | string | Default value: `""` |
| `request-num` | Specifies the number of requests to be sent to the Redis server | int | Default value: `0` |

#### 使用服務模式模擬快取穿透的範例

```bash
curl -X POST 127.0.0.1:31767/api/attack/redis -H "Content-Type:application/json" -d '{"action":"penetration", ""addr":"127.0.0.1:6379", "request-num":"10000"}'
```

### 使用服務模式模擬 Sentinel 重啟

#### 模擬 Sentinel 重啟的參數

| Parameter | Description | Type | Value |
| :-- | :-- | :-- | :-- |
| `action` | Action of the experiment | string | set to "restart" |
| `addr` | The address and port of Sentinel to be injected into the fault, such as `127.0.0.1:26379` | string | Default value: `""` |
| `conf` | Specifies the path of Sentinel configuration file, which is used to recover the Sentinel | string | Default value: `""` |
| `flush-config` | Forces Sentinel to rewrite its configuration on disk, including the current Sentinel state | bool | Default value: `true` |
| `password` | The password to log in to the server | string | Default value: `""` |
| `redis-path` | Specifies the path of `redis-server` command-line tool | string | Default value: `""` |

#### 使用服務模式模擬 Sentinel 重啟的範例

```bash
curl -X POST 127.0.0.1:31767/api/attack/redis -H "Content-Type:application/json" -d '{"action":"restart", ""addr":"127.0.0.1:26379", "conf":"/home/redis-test/sentinel-26379.conf"}'
```

### 使用服務模式模擬 Sentinel 停止

#### 模擬 Sentinel 停止的參數

| Parameter | Description | Type | Value |
| :-- | :-- | :-- | :-- |
| `action` | Action of the experiment | string | set to "stop" |
| `addr` | The address and port of Sentinel to be injected into the fault, such as `127.0.0.1:26379` | string | Default value: `""` |
| `conf` | Specifies the path of Sentinel configuration file, which is used to recover the Sentinel | string | Default value: `""` |
| `flush-config` | Forces Sentinel to rewrite its configuration on disk, including the current Sentinel state | bool | Default value: `true` |
| `password` | The password to log in to the server | string | Default value: `""` |
| `redis-path` | Specifies the path of `redis-server` command-line tool | string | Default value: `""` |

#### 使用服務模式模擬 Sentinel 停止的範例

```bash
curl -X POST 127.0.0.1:31767/api/attack/redis -H "Content-Type:application/json" -d '{"action":"stop", ""addr":"127.0.0.1:26379", "conf":"/home/redis-test/sentinel-26379.conf"}'
```
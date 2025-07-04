---
title: Simulate JVM Application Faults
---

Chaosd 透過 [Byteman](https://github.com/chaos-mesh/byteman) 模擬 JVM 應用程式的故障。支援的故障類型如下：

- 拋出自訂例外

- 觸發垃圾收集

- 增加方法延遲

- 修改方法的回傳值

- 透過設定 Byteman 組態檔案觸發故障

- 增加 JVM 壓力

本文件描述如何使用 Chaosd 建立上述 JVM 實驗的故障類型。

## 使用命令列模式建立實驗

本節介紹如何使用命令列模式建立 JVM 應用程式故障的實驗。

在建立實驗之前，您可以執行以下命令列以查看 Chaosd 支援的 JVM 應用程式故障類型：

```bash
chaosd attack jvm -h
```

結果如下：

```bash
JVM attack related commands

Usage:
  chaosd attack jvm [command]

Available Commands:
  exception   throw specified exception for specified method
  gc          trigger GC for JVM
  latency     inject latency to specified method
  return      return specified value for specified method
  rule-file   inject fault with configured byteman rule file
  stress      inject stress to JVM

Flags:
  -h, --help       help for jvm
      --pid int    the pid of Java process which needs to attach
      --port int   the port of agent server (default 9288)

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID

Use "chaosd attack jvm [command] --help" for more information about a command.
```

### 使用命令列模式拋出自訂例外

#### 拋出自訂例外的命令

要查看拋出自訂例外命令的使用方式與組態項目，請執行以下命令：

```bash
chaosd attack jvm exception --help
```

結果如下：

```bash
throw specified exception for specified method

Usage:
  chaosd attack jvm exception [options] [flags]

Flags:
  -c, --class string       Java class name
      --exception string   the exception which needs to throw for action 'exception'
  -h, --help               help for exception
  -m, --method string      the method name in Java class

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --pid int            the pid of Java process which needs to attach
      --port int           the port of agent server (default 9288)
      --uid string         the experiment ID
```

#### 拋出自訂例外的組態描述

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| `class` | `c` | The name of the Java class | string type, required |
| `exception` | None | The thrown custom exception | string type, required |
| `method` | `m` | The name of the method | string type, required to be configured |
| `pid` | None | The Java process ID where the fault is to be injected | int type, required |
| `port` | None | The port number attached to the Java process agent. The fault is injected into the Java process through this port number. | int type. The default value is `9288`. |
| `uid` | None | The experiment ID | string type. This item is not required to be configured, because Chaosd randomly creates one. |

#### 拋出自訂例外的範例

```bash
chaosd attack jvm exception -c Main -m sayhello --exception 'java.io.IOException("BOOM")' --pid 30045
```

結果如下：

```bash
[2021/08/05 02:39:39.106 +00:00] [INFO] [jvm.go:208] ["byteman rule"] [rule="\nRULE Main-sayhello-exception-q6nd0\nCLASS Main\nMETHOD sayhello\nAT ENTRY\nIF true\nDO \n\tthrow new java.io.IOException(\"BOOM\");\nENDRULE\n"] [file=/tmp/rule.btm296930759]
Attack jvm successfully, uid: 26a45ae2-d395-46f5-a126-2b2c6c85ae9d
```

### 使用命令列模式觸發垃圾收集

#### 觸發垃圾收集的命令

要查看觸發垃圾收集命令的使用方式與組態項目，請執行以下命令：

```bash
chaosd attack jvm gc --help
```

```bash
trigger GC for JVM

Usage:
  chaosd attack jvm gc [flags]

Flags:
  -h, --help   help for gc

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --pid int            the pid of Java process which needs to attach
      --port int           the port of agent server (default 9288)
      --uid string         the experiment ID
```

#### 觸發垃圾收集的組態描述

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| `pid` | None | The Java process ID where the fault is to be injected | int type, required |
| `port` | None | The port number attached to the Java process agent. The fault is injected into the Java process through this port number. | int type. The default value is `9288`. |
| `uid` | None | The experiment ID | string type. This item is not required to be configured, because Chaosd randomly creates one. |

#### 觸發垃圾收集的範例

```bash
chaosd attack jvm gc --pid 89345
```

結果如下：

```bash
[2021/08/05 02:49:47.850 +00:00] [INFO] [jvm.go:208] ["byteman rule"] [rule="\nRULE --gc-u0mlf\nGC\nENDRULE\n"] [file=/tmp/rule.btm012481052]
Attack jvm successfully, uid: f360e70a-5359-49b6-8526-d7e0a3c6f696
```

觸發垃圾收集是一次性操作，實驗不需要恢復。

### 使用命令列模式增加方法延遲

#### 增加方法延遲的命令

要查看增加方法延遲命令的使用方式與組態項目，請執行以下命令：

```bash
chaosd attack jvm latency --help
```

結果如下：

```bash
inject latency to specified method

Usage:
  chaosd attack jvm latency [options] [flags]

Flags:
  -c, --class string    Java class name
  -h, --help            help for latency
      --latency int     the latency duration, unit ms
  -m, --method string   the method name in Java class

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --pid int            the pid of Java process which needs to attach
      --port int           the port of agent server (default 9288)
      --uid string         the experiment ID
```

#### 增加方法延遲的組態描述

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| `class` | `c` | The name of the Java class | string type, required |
| `latency` | None | The duration of increasing method latency | int type, required. The unit is millisecond. |
| `method` | `m` | The name of the method | string type, required |
| `pid` | None | The Java process ID where the fault is to be injected | int type, required |
| `port` | None | The port number attached to the Java process agent. The fault is injected into the Java process through this port number. | int type. The default value is `9288`. |
| `uid` | None | The experiment ID | string type. This item is not required to be configured, because Chaosd randomly creates one. |

#### 增加方法延遲的範例

```bash
chaosd attack jvm latency --class Main --method sayhello --latency 5000 --pid 100840
```

結果如下：

```bash
[2021/08/05 03:08:50.716 +00:00] [INFO] [jvm.go:208] ["byteman rule"] [rule="\nRULE Main-sayhello-latency-hlib2\nCLASS Main\nMETHOD sayhello\nAT ENTRY\nIF true\nDO \n\tThread.sleep(5000);\nENDRULE\n"] [file=/tmp/rule.btm359997255]
[2021/08/05 03:08:51.155 +00:00] [INFO] [jvm.go:94] ["submit rules"] [output="install rule Main-sayhello-latency-hlib2\n\n"]
Attack jvm successfully, uid: bbe00c57-ac9d-4113-bf0c-2a6f184be261
```

### 使用命令列模式修改方法的回傳值

#### 修改方法回傳值的命令

要查看修改方法回傳值命令的使用方式與組態項目，請執行以下命令：

```bash
chaosd attack jvm return --help
```

```bash
return specified value for specified method

Usage:
  chaosd attack jvm return [options] [flags]

Flags:
  -c, --class string    Java class name
  -h, --help            help for return
  -m, --method string   the method name in Java class
      --value string    the return value for action 'return'. Only supports number and string types.

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --pid int            the pid of Java process which needs to attach
      --port int           the port of agent server (default 9288)
      --uid string         the experiment ID
```

#### 修改方法回傳值的組態描述

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| class | c | The name of the Java class | string type, required to be configured |
| method | `m` | The name of the method | string type, required to be configured |
| value | None | Specifies the return value of the method | string type, required to be configured. Currently, the item can be numeric and string types. If the item (return value) is string, double quotes are required, like "chaos". |
| pid | None | The Java process ID where the fault is needed to be injected | int type, required to be configured |
| port | None | The port number attached to the Java process agent. The faults is injected into the Java process through this port number. | int type. The default value is `9288`. |
| `uid` | None | The experiment ID | string type. This item is not required to be configured, because Chaosd randomly creates one. |

#### 模擬修改方法回傳值情境的範例

```bash
chaosd attack jvm return --class Main --method getnum --value 999 --pid 112694
```

結果如下：

```bash
[2021/08/05 03:35:10.603 +00:00] [INFO] [jvm.go:208] ["byteman rule"] [rule="\nRULE Main-getnum-return-i6gb7\nCLASS Main\nMETHOD getnum\nAT ENTRY\nIF true\nDO \n\treturn 999;\nENDRULE\n"] [file=/tmp/rule.btm051982059]
[2021/08/05 03:35:10.820 +00:00] [INFO] [jvm.go:94] ["submit rules"] [output="install rule Main-getnum-return-i6gb7\n\n"]
Attack jvm successfully, uid: e2f204f6-4bed-4d92-aade-2b4a47b02e5d
```

### 透過命令列模式設定 Byteman 組態檔觸發故障

您可以在 Byteman 規則組態檔中設定故障規則，接著透過 Chaosd 指定組態檔路徑來注入故障。關於 Byteman 規則組態，請參閱 [byteman-rule-language](https://downloads.jboss.org/byteman/4.0.16/byteman-programmers-guide.html#the-byteman-rule-language)。

#### 設定 Byteman 組態檔觸發故障的指令

欲查看設定 Byteman 組態檔觸發故障指令的用法與配置項，請執行以下指令：

```bash
chaosd attack jvm rule-file --help
```

執行結果如下：

```bash
inject fault with configured byteman rule file

Usage:
  chaosd attack jvm rule-file [options] [flags]

Flags:
  -h, --help          help for rule-file
  -p, --path string   the path of configured byteman rule file

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
      --pid int            the pid of Java process which needs to attach
      --port int           the port of agent server (default 9288)
      --uid string         the experiment ID
```

#### 設定 Byteman 組態檔觸發故障的配置說明

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| `path` | None | Specifies the path of the Byteman configuration file | string type, required |
| `pid` | None | The Java process ID where the fault is to be injected | int type, required |
| `port` | None | The port number attached to the Java process agent. The fault is injected into the Java process through this port number. | int type. The default value is `9288`. |
| `uid` | None | The experiment ID | string type. This item is not required to be configured, because Chaosd randomly creates one. |

#### 設定 Byteman 組態檔觸發故障的範例

首先，根據具體 Java 程序並參考 [Byteman 規則語言](https://downloads.jboss.org/byteman/4.0.16/byteman-programmers-guide.html#the-byteman-rule-language)編寫規則組態檔，例如：

```txt
RULE modify return value
CLASS Main
METHOD getnum
AT ENTRY
IF true
DO
    return 9999
ENDRULE
```

接著將組態檔儲存為 `return.btm` 檔案，然後執行以下指令注入故障：

```bash
chaosd attack jvm rule-file -p ./return.btm --pid 112694
```

執行結果如下：

```bash
[2021/08/05 03:45:40.757 +00:00] [INFO] [jvm.go:152] ["rule file data:RULE modify return value\nCLASS Main\nMETHOD getnum\nAT ENTRY\nIF true\nDO\n    return 9999\nENDRULE\n"]
[2021/08/05 03:45:41.011 +00:00] [INFO] [jvm.go:94] ["submit rules"] [output="install rule modify return value\n\n"]
Attack jvm successfully, uid: 5ca2e06d-a7c6-421d-bb67-0c9908bac17a
```

### 透過命令列模式增加 JVM 壓力

#### 增加 JVM 壓力的指令

欲查看增加 JVM 壓力指令的用法與配置項，請執行以下指令：

```bash
chaosd attack jvm stress --help
```

執行結果如下：

```bash
inject stress to JVM

Usage:
  chaosd attack jvm stress [options] [flags]

Flags:
      --cpu-count int   the CPU core number
  -h, --help            help for stress
      --mem-type int    the memory type to be allocated. The value can be 'stack' or 'heap'.

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --pid int            the pid of Java process which needs to attach
      --port int           the port of agent server (default 9288)
      --uid string         the experiment ID
```

#### 增加 JVM 壓力的配置說明

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| `cpu-count` | None | The number of CPU cores used for increasing JVM stress | int type. You must configure one item between `cpu-count` and `mem-type`. |
| `mem-type` | None | The type of OOM | string type. Currently, both 'stack' and 'heap' OOM types are supported. You must configure one item between `cpu-count` and `mem-type`. |
| `pid` | None | The Java process ID where the fault is to be injected | int type, required |
| `port` | None | The port number attached to the Java process agent. The fault is injected into the Java process through this port number. | int type. The default value is `9288`. |
| `uid` | None | The experiment ID | string type. This item is not required to be configured, because Chaosd randomly creates one. |

#### 增加 JVM 壓力的範例

```bash
chaosd attack jvm stress --cpu-count 2 --pid 123546
```

執行結果如下：

```bash
[2021/08/05 03:59:51.256 +00:00] [INFO] [jvm.go:208] ["byteman rule"] [rule="\nRULE --stress-jfeiu\nSTRESS CPU\nCPUCOUNT 2\nENDRULE\n"] [file=/tmp/rule.btm773062009]
[2021/08/05 03:59:51.613 +00:00] [INFO] [jvm.go:94] ["submit rules"] [output="install rule --stress-jfeiu\n\n"]
Attack jvm successfully, uid: b9b997b5-0a0d-4f1f-9081-d52a32318b84
```

## 透過服務模式建立實驗

您可以依循以下指示透過服務模式建立實驗：

1. 以服務模式執行 Chaosd：

   ```bash
   chaosd server --port 31767
   ```

2. 向 Chaosd 服務的 `/api/attack/{uid}` 路徑發送 HTTP POST 請求：

   ```bash
   curl -X POST 172.16.112.130:31767/api/attack/jvm -H "Content-Type:application/json" -d '{fault-configuration}'
   ```

   上述指令中的 `fault-configuration` 部分需根據故障類型配置，對應參數請參閱後續章節中各故障類型的參數與範例。

:::note

執行實驗時，請務必保存實驗的 UID 資訊。當需終止對應 UID 的實驗時，需向 Chaosd 服務的 `/api/attack/{uid}` 路徑發送 HTTP DELETE 請求。

:::

### 透過服務模式拋出自訂例外

#### 拋出自訂例外的參數

| Parameter | Description | Value |
| :-- | :-- | :-- |
| `action` | The action of the experiment | Set to "exception" |
| `class` | The name of the Java class | string type, required |
| `exception` | The thrown custom exception | string type, required |
| `method` | The name of the method | string type, required |
| `pid` | The Java process ID where the fault is to be injected | int type, required |
| `port` | The port number attached to the Java process agent. The faults is injected into the Java process through this port number. | int type. The default value is `9288`. |
| `uid` | The experiment ID | string type. This item is not required to be configured, because Chaosd randomly creates one. |

#### 透過服務模式拋出自訂例外的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/jvm -H "Content-Type:application/json" -d '{"action":"exception","class":"Main","method":"sayhello","exception":"java.io.IOException(\"BOOM\")","pid":1828622}'
```

執行結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"c3c519bf-819a-4a7b-97fb-e3d0814481fa"}
```

### 透過服務模式觸發垃圾回收

#### 觸發垃圾回收的參數

| Parameter | Description | Value |
| :-- | :-- | :-- |
| `action` | The action of the experiment | Set to "gc" |
| `pid` | The Java process ID where the fault is to be injected | int type, required |
| `port` | The port number attached to the Java process agent. The fault is injected into the Java process through this port number. | int type. The default value is `9288`. |
| `uid` | The experiment ID | string type. This item is not required to be configured, because Chaosd randomly creates one. |

#### 透過服務模式觸發垃圾回收的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/jvm -H "Content-Type:application/json" -d '{"action":"gc","pid":1828622}'
```

結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"c3c519bf-819a-4a7b-97fb-e3d0814481fa"}
```

觸發垃圾回收是一次性操作，實驗無需恢復。

### 使用服務模式增加方法延遲

#### 增加方法延遲的參數

| Parameter | Description | Value |
| :-- | :-- | :-- |
| `action` | The action of the experiment | Set to "latency" |
| `class` | The name of the Java class | string type, required |
| `latency` | The duration of increasing method latency | int type, required. The unit is millisecond. |
| `method` | The name of the method | string type, required |
| `pid` | The Java process ID where the fault is to be injected | int type, required |
| `port` | The Java process ID where the fault is needed to be injected | int type, required |
| `uid` | The experiment ID | string type. This item is not required to be configured, because Chaosd randomly creates one. |

#### 使用服務模式增加方法延遲的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/jvm -H "Content-Type:application/json" -d '{"action":"latency","class":"Main","method":"sayhello","latency":5000,"pid":1828622}'
```

結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"a551206c-960d-4ac5-9056-518e512d4d0d"}
```

### 使用服務模式修改方法的返回值

#### 修改方法返回值的參數

| Parameter | Description | Value |
| :-- | :-- | :-- |
| `action` | The action of the experiment | Set to "return" |
| `class` | The name of the Java class | string type, required |
| `method` | The name of the method | string type, required |
| `value` | Specifies the return value of the method | string type, required. Currently, the item can be numeric and string types. If the item (return value) is string, double quotes are required, like "chaos". |
| `pid` | The Java process ID where the fault is to be injected | int type, required |
| `port` | The port number attached to the Java process agent. The fault is injected into the Java process through this port number. | int type. The default value is `9288`. |
| `uid` | The experiment ID | string type. This item is not required to be configured, because Chaosd randomly creates one. |

#### 使用服務模式修改方法返回值的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/jvm -H "Content-Type:application/json" -d '{"action":"return","class":"Main","method":"getnum","value":"999","pid":1828622}'
```

結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"a551206c-960d-4ac5-9056-518e512d4d0d"}
```

### 使用服務模式設定 Byteman 設定檔案觸發故障

您可以根據 Byteman 規則設定來設定故障規則。有關 Byteman 規則設定的詳細資訊，請參閱 [byteman-rule-language](https://downloads.jboss.org/byteman/4.0.16/byteman-programmers-guide.html#the-byteman-rule-language)。

#### 透過設定 Byteman 設定檔案觸發故障的參數

| Parameter | Description | Value |
| :-- | :-- | :-- |
| `action` | The action of the experiment | Set to "rule-data" |
| `rule-data` | Specifies the Byteman configuration data | string type, required |
| `pid` | The Java process ID where the fault is to be injected | int type, required |
| `port` | The port number attached to the Java process agent. The fault is injected into the Java process through this port number. | int type. The default value is `9288`. |
| `uid` | The experiment ID | string type. This item is not required to be configured, because Chaosd randomly creates one. |

#### 使用服務模式設定 Byteman 設定檔案觸發故障的範例

首先，基於特定 Java 程式並參考 [Byteman 規則語言](https://downloads.jboss.org/byteman/4.0.16/byteman-programmers-guide.html#the-byteman-rule-language)，編寫規則設定檔案。例如：

```txt
RULE modify return value
CLASS Main
METHOD getnum
AT ENTRY
IF true
DO
    return 9999
ENDRULE
```

接著，將設定檔案中的換行符轉義為換行字元 "\n"，並使用轉義後的文字作為 "rule-data" 的值。執行以下命令：

```bash
curl -X POST 127.0.0.1:31767/api/attack/jvm -H "Content-Type:application/json" -d '{"action":"rule-data","pid":30045,"rule-data":"\nRULE modify return value\nCLASS Main\nMETHOD getnum\nAT ENTRY\nIF true\nDO return 9999\nENDRULE\n"}'
```

結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"a551206c-960d-4ac5-9056-518e512d4d0d"}
```

### 使用服務模式增加 JVM 壓力

#### 增加 JVM 壓力的參數

| 參數 | 描述 | 值 |
| :-- | :-- | :-- | --- |
| `action` | 實驗操作 | 設為 "stress" |
| `cpu-count` | 用於增加 CPU 壓力的 CPU 核心數 | 整數類型。必須配置 `cpu-count` 和 `mem-type` 其中一項 |
| `mem-type` | OOM 類型 | 字串類型。目前支援 'stack' 和 'heap' 兩種 OOM 類型。必須配置 `cpu-count` 和 `mem-type` 其中一項 |
| `pid` | 無 | 要注入故障的 Java 程序 ID | 整數類型，必填 |
| `port` | 無 | 附加到 Java 程序代理的連接埠號，透過此連接埠注入故障 | 整數類型。預設值為 `9288` |
| `uid` | 無 | 實驗 ID | 字串類型。無需配置，Chaosd 會隨機產生 |

#### 使用服務模式增加 JVM 壓力的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/jvm -H "Content-Type:application/json" -d '{"action":"stress","cpu-count":1,"pid":1828622}'
```

結果如下：

```bash
{"status":200,"message":"attack successfully","uid":"a551206c-960d-4ac5-9056-518e512d4d0d"}
```
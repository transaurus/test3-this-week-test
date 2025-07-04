---
title: Simulate Time Faults
summary: This document describes how to use Chaosd to simulate a time offset scenario.
---

本文件描述如何使用 Chaosd 模擬時間偏移情境。您可透過命令列模式或服務模式建立實驗。

## 使用命令列模式建立實驗

本節描述如何使用命令建立時間故障實驗。

在建立實驗前，您可執行以下命令檢查時間故障的選項：

```
chaosd attack clock -h
```

結果如下：

```bash
$ chaosd attack clock -h

clock skew

Usage:
  chaosd attack clock attack [flags]

Flags:
  -c, --clock-ids-slice string   The identifier of the particular clock on which to act.More clock description in linux kernel can be found in man page of clock_getres, clock_gettime, clock_settime.Muti clock ids should be split with "," (default "CLOCK_REALTIME")
  -h, --help                     help for clock
  -p, --pid int                  Pid of target program.
  -t, --time-offset string       Specifies the length of time offset.

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID

```

### 快速範例

準備測試程式：

```bash
cat > time.c << EOF
#include <stdio.h>
#include <time.h>
#include <unistd.h>
#include <sys/types.h>

int main() {
    printf("PID : %ld\n", (long)getpid());
    struct  timespec ts;
    for(;;) {
        clock_gettime(CLOCK_REALTIME, &ts);
        printf("Time : %lld.%.9ld\n", (long long)ts.tv_sec, ts.tv_nsec);
        sleep(10);
    }
}
EOF

gcc -o get_time ./time.c
```

接著執行 get_time 並嘗試對其進行攻擊。以下為範例：

```bash
chaosd attack clock -p $PID -t 11s
```

### 模擬時間故障的配置

| Parameter | Type | Note | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| timeOffset | string | Specifies the length of time offset. | None | Yes | `-5m` |
| clockIds | []string | Specifies the ID of clock that will be offset. See the [clock_gettime documentation](https://man7.org/linux/man-pages/man2/clock_gettime.2.html) for details. | `["CLOCK_REALTIME"]` | No | `["CLOCK_REALTIME", "CLOCK_MONOTONIC"]` |
| pid | string | The identifier of the process. | None | Yes | `1` |

### 使用服務模式建立實驗

(持續更新中)
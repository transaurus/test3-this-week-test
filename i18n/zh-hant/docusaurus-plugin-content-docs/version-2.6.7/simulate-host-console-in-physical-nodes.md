---
title: Simulate Host Faults
---

本文件介紹如何使用 Chaosd 模擬主機關機故障。

## 查看主機關機實驗的幫助資訊

在建立故障實驗之前，您可以執行以下指令來查看主機關機實驗的幫助資訊：

```bash
chaosd attack host shutdown -h
```

輸出如下：

```bash
shutdowns system, this action will trigger shutdown of the host machine

Usage:
  chaosd attack host shutdown [flags]

Flags:
  -h, --help   help for shutdown

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
```

## 建立主機關機實驗

若要建立主機關機實驗，請執行以下指令：

```bash
chaosd attack host shutdown
```

範例輸出如下：

```bash
chaosd attack host shutdown
Shutdown successfully, uid: 4bc9b74a-5fe2-4038-b4f3-09ae95b57694
```

執行此指令後，您的主機將在所有程序關閉後關機。
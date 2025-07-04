---
slug: /run-chaos-experiments-on-physical-machines
title: 'How to run chaos experiments on your physical machine'
authors: xiangwang
image: /img/blog/chaosd-banner.png
tags: [Chaos Mesh, Chaos Engineering, Chaosd]
---

![如何在實體機器上執行混亂實驗](/img/blog/chaosd-banner.png)

[Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh) 是一個雲原生的混沌工程平台，可在 Kubernetes 環境中編排混沌事件。透過 Chaos Mesh，您可以模擬各種故障，並使用網頁介面 Chaos Dashboard 直接管理混沌實驗。自開源以來，Chaos Mesh 已被眾多企業採用，以確保其系統的韌性與穩健性。但在過去一年中，我們頻繁收到社群詢問：當服務未部署於 Kubernetes 時該如何執行混沌實驗？

<!--truncate-->

## 什麼是 chaosd

為滿足實體機器混沌測試的增長需求，我們隆重推出強化工具包 chaosd。您可能對這個名稱感到熟悉，因為它是由 Chaos Mesh 的核心組件 `chaos-daemon` 演變而來。在 TiDB Hackathon 2020 中，我們[重構了 chaosd 使其超越命令列工具的範疇](https://pingcap.com/blog/chaos-mesh-remake-one-step-closer-toward-chaos-as-a-service#refactor-chaosd)。現在透過 [chaosd v1.0.1](https://github.com/chaos-mesh/chaosd/releases/tag/v1.0.1)，您能模擬針對實體機器的特定錯誤，並在實驗後完整恢復原狀。

## chaosd 的優勢

chaosd 具備以下優點：

- **易於使用**：透過 chaosd 命令可輕鬆建立和管理混沌實驗。

- **豐富的故障類型**：可在不同層級模擬實體機器故障，包括進程故障、網路故障、Java 虛擬機（JVM）應用故障、壓力場景、磁碟故障及主機故障。

- **多種工作模式**：可作為命令列工具或服務運行。

事不宜遲，讓我們立即試用。

## 如何使用 chaosd

本節將逐步示範如何使用 chaosd 注入網路故障。您的 glibc 版本必須為 v2.17 或更高。

### 1. 下載並解壓 chaosd

執行以下命令下載 chaosd：

```bash
curl -fsSL -o chaosd-v1.0.1-linux-amd64.tar.gz https://mirrors.chaos-mesh.org/chaosd-v1.0.1-linux-amd64.tar.gz
```

解壓縮檔案，其中包含兩個目錄：

- `chaosd` 包含 chaosd 的執行入口。

- `tools` 包含執行混沌實驗所需的工具：[stress-ng](https://wiki.ubuntu.com/Kernel/Reference/stress-ng)（模擬壓力場景）、[Byteman](https://github.com/chaos-mesh/byteman)（模擬 JVM 應用故障）及 PortOccupyTool（模擬網路故障）。

### 2. 建立混沌實驗

本實驗將使伺服器無法存取 chaos-mesh.org。

執行以下命令：

```bash
sudo ./chaosd attack network loss --percent 100 --hostname chaos-mesh.org --device ens33
```

範例輸出：

```bash
Attack network successfully, uid: c55a84c5-c181-426b-ae31-99c8d4615dbe
```

此模擬使 ens33 網路介面卡無法傳送或接收 [chaos-mesh.org](http://chaos-mesh.org) 的網路封包。必須使用 `sudo` 命令的原因在於實驗需修改網路規則，這需要 root 權限。

請務必保存混沌實驗的 `uid`，後續恢復流程將使用此識別碼。

### 3. 驗證結果

使用 `ping` 命令測試伺服器能否存取 chaos-mesh.org：

```bash
ping chaos-mesh.org
PING chaos-mesh.org (185.199.109.153) 56(84) bytes of data.
```

當你執行指令時，該網站很可能不會回應。按下 `CTRL`+`C` 來停止 ping 程序。你應該能看到 `ping` 指令的統計資料：`100% packet loss`。

範例輸出：

```bash
2 packets transmitted, 0 received, 100% packet loss, time 1021ms
```

### 4. 恢復實驗

要恢復實驗，請執行以下指令：

```bash
sudo ./chaosd recover c55a84c5-c181-426b-ae31-99c8d4615dbe
```

範例輸出：

```bash
Recover c55a84c5-c181-426b-ae31-99c8d4615dbe successfully
```

在此步驟中，你同樣需要使用 `sudo` 指令，因為需要 root 權限。當你完成恢復實驗後，請再次嘗試 ping chaos-mesh.org 來驗證連線。

## 後續步驟

### 支援儀表板網頁

如你所見，chaosd 相當易於使用。但我們可以讓它更簡單——chaosd 的儀表板網頁目前正在積極開發中。

我們將持續提升其易用性並實作更多功能，例如管理透過 chaosd 執行的混沌實驗，以及透過 Chaos Mesh 執行的實驗。這將為 Kubernetes 和實體機器上的混沌測試提供一致且統一的用戶體驗。以下架構僅為簡單範例：

![Chaos Mesh 的優化架構](/img/blog/chaos-mesh-optimized-architecture.png)

欲了解更多，請查看 [Chaos Mesh 的優化架構](https://pingcap.com/blog/chaos-mesh-remake-one-step-closer-toward-chaos-as-a-service#developing-chaos-mesh-towards-caas)。

### 新增更多故障注入類型

目前，chaosd 提供六種故障注入類型。我們計劃開發更多 Chaos Mesh 已支援的類型，包括 HTTPChaos 和 IOChaos。

如果你有興趣協助我們改善 chaosd，歡迎[挑選一個議題](https://github.com/chaos-mesh/chaosd/labels/help%20wanted)並開始貢獻！

## 立即嘗試！

如果你有興趣使用 chaosd 並想進一步探索，請查看[文件](https://chaos-mesh.org/docs/chaosd-overview)。如果在執行 chaosd 時遇到問題，或者有功能請求，歡迎隨時[建立議題](https://github.com/chaos-mesh/chaosd/issues)。我們很樂意聽到你的聲音！
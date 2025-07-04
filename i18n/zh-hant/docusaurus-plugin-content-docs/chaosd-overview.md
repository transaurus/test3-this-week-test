---
title: Chaosd Introduction
---

## Chaosd 簡介

[Chaosd](https://github.com/chaos-mesh/chaosd) 是由 Chaos Mesh 提供的混沌工程測試工具，需獨立下載部署（參見[下載與部署](#download-and-deploy)），用於在實體機環境注入故障並進行恢復。

Chaosd 具備以下核心優勢：

- **易於使用**：只需在 Chaosd 中執行簡單命令即可創建和管理混沌實驗。

- **多樣故障類型**：提供多層級的實體機故障注入能力，涵蓋進程、網絡、壓力、磁盤、主機等類型，未來將持續擴充。

- **多種工作模式**：可作為命令行工具或後台服務運行，滿足不同場景需求。

### 支援的故障類型

Chaosd 可模擬以下故障類型：

- **進程故障**：對進程注入故障，支援殺死進程、暫停進程等操作。

- **網絡故障**：對實體機網絡注入故障，支援增加延遲、封包遺失、封包損壞等操作。

- **壓力故障**：對實體機 CPU 或記憶體施加壓力。

- **磁盤故障**：對磁盤注入故障，支援增加讀寫負載、填充磁盤等操作。

- **主機故障**：對實體主機注入故障，支援關閉主機等操作。

各故障類型詳細說明與用法請參閱相關文檔。

### 運行環境要求

需使用 glibc v2.17 或更高版本。

### 下載與部署

1. 設定要下載的 Chaosd 版本環境變數（例如 v1.0.0）：

   ```bash
   export CHAOSD_VERSION=v1.0.0
   ```

   所有發行版本請參閱 [releases](https://github.com/chaos-mesh/chaosd/releases)。

   若需下載最新版本（非穩定版），使用 `latest`：

   ```bash
   export CHAOSD_VERSION=latest
   ```

2. 下載 Chaosd：

   ```bash
   curl -fsSLO https://mirrors.chaos-mesh.org/chaosd-$CHAOSD_VERSION-linux-amd64.tar.gz
   ```

3. 解壓縮檔案並移動至 `/usr/local` 目錄：

   ```bash
   tar zxvf chaosd-$CHAOSD_VERSION-linux-amd64.tar.gz && sudo mv chaosd-$CHAOSD_VERSION-linux-amd64 /usr/local/
   ```

4. 將 Chaosd 目錄加入 `PATH` 環境變數：

   ```bash
   export PATH=/usr/local/chaosd-$CHAOSD_VERSION-linux-amd64:$PATH
   ```

### 工作模式

Chaosd 支援以下運行模式：

- **命令行模式**：直接作為命令行工具執行，注入與恢復故障。

- **服務模式**：以後台服務運行，透過發送 HTTP 請求注入與恢復故障。
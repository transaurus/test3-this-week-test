---
title: Chaosd Introduction
---

## Chaosd 簡介

[Chaosd](https://github.com/chaos-mesh/chaosd) 是 Chaos Mesh 提供的混沌工程測試工具。您需要單獨下載部署（參見[下載與部署](#download-and-deploy)），用於在實體機環境中注入故障並恢復故障。

Chaosd 具備以下核心優勢：

- 易於使用：只需在 Chaosd 中執行簡單命令即可創建和管理混沌實驗。

- 多樣故障類型：提供多層面的實體機故障注入類型，包括進程、網路、壓力、磁碟、主機等，未來將持續新增故障類型。

- 多種工作模式：既可作為命令列工具使用，也可作為服務運行，滿足不同場景需求。

### 支援的故障類型

您可使用 Chaosd 模擬以下故障類型：

- 進程：對進程注入故障，支援殺死進程或停止進程等操作。

- 網路：對實體機網路注入故障，支援增加網路延遲、丟包、封包損壞等操作。

- 壓力：對實體機 CPU 或記憶體施加壓力。

- 磁碟：對實體機磁碟注入故障，支援增加讀寫磁碟負載、填滿磁碟等操作。

- 主機：對實體機本身注入故障，支援關閉實體機等操作。

各故障類型詳細說明與用法請參閱相關文件。

### 運行環境要求

您的 glibc 版本需為 v2.17 或更高版本。

### 下載與部署

1. 設定要下載的 Chaosd 版本為環境變數（例如 v1.0.0）：

   ```bash
   export CHAOSD_VERSION=v1.0.0
   ```

   查看 Chaosd 所有發布版本請參閱 [releases](https://github.com/chaos-mesh/chaosd/releases)。

   若需下載最新版（非穩定版），使用 `latest`：

   ```bash
   export CHAOSD_VERSION=latest
   ```

2. 下載 Chaosd：

   ```bash
   curl -fsSLO https://mirrors.chaos-mesh.org/chaosd-$CHAOSD_VERSION-linux-amd64.tar.gz
   ```

3. 解壓 Chaosd 文件並移至 `/usr/local` 目錄：

   ```bash
   tar zxvf chaosd-$CHAOSD_VERSION-linux-amd64.tar.gz && sudo mv chaosd-$CHAOSD_VERSION-linux-amd64 /usr/local/
   ```

4. 將 Chaosd 目錄加入 `PATH` 環境變數：

   ```bash
   export PATH=/usr/local/chaosd-$CHAOSD_VERSION-linux-amd64:$PATH
   ```

### 工作模式

您可透過以下模式使用 Chaosd：

- 命令列模式：直接作為命令列工具運行，注入並恢復故障。

- 服務模式：以後台服務形式運行，通過發送 HTTP 請求注入並恢復故障。
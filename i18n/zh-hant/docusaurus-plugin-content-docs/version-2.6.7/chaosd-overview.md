---
title: Chaosd Introduction
---

## Chaosd 簡介

[Chaosd](https://github.com/chaos-mesh/chaosd) 是由 Chaos Mesh 提供的混沌工程測試工具。您需要單獨下載和部署（參見[下載與部署](#download-and-deploy)）。它用於在實體機環境中注入故障並恢復故障。

Chaosd 具有以下核心優勢：

- 易於使用：您只需在 Chaosd 中執行簡單命令即可建立和管理混沌實驗。

- 多樣化故障類型：Chaosd 提供可在不同層級注入實體機的多種故障類型，包括程序、網路、壓力、磁碟、主機等。更多故障類型將持續新增。

- 多種工作模式：Chaosd 既可作為命令列工具使用，也可作為服務運行，滿足不同場景需求。

### 支援的故障類型

您可使用 Chaosd 模擬以下故障類型：

- 程序：向程序中注入故障。支援殺死程序或停止程序等操作。

- 網路：向實體機的網路中注入故障。支援增加網路延遲、封包遺失和封包損壞等操作。

- 壓力：對實體機的 CPU 或記憶體施加壓力。

- 磁碟：向實體機的磁碟中注入故障。支援增加讀寫磁碟負載及填滿磁碟等操作。

- 主機：向實體機注入故障。支援關閉實體機等操作。

有關各故障類型的詳細介紹與用法，請參閱相關文件。

### 運行環境

您的 glibc 版本必須為 v2.17 或更高版本。

### 下載與部署

1. 設定要下載的 Chaosd 版本為環境變數。例如 v1.0.0：

   ```bash
   export CHAOSD_VERSION=v1.0.0
   ```

   查看所有已發布的 Chaosd 版本，請參閱 [releases](https://github.com/chaos-mesh/chaosd/releases)。

   若要下載最新版本（非穩定版），請使用 `latest`：

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

您可在以下模式中使用 Chaosd：

- 命令列模式：直接將 Chaosd 作為命令列工具運行，注入和恢復故障。

- 服務模式：將 Chaosd 作為後台服務運行，通過發送 HTTP 請求來注入和恢復故障。
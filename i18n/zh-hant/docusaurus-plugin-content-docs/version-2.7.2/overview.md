---
slug: /
title: Chaos Mesh Overview
---

本文件描述 Chaos Mesh 的概念、使用場景、核心優勢及其架構。

## Chaos Mesh 概述

Chaos Mesh 是一個開源的雲原生混沌工程平台，提供多種類型的故障模擬能力，並具備強大的故障場景編排能力。

透過 Chaos Mesh，您可在開發、測試及生產環境中便捷地模擬現實中可能發生的各種異常狀況，從而發現系統潛在問題。為降低混沌工程專案的門檻，Chaos Mesh 提供可視化操作介面，您可在 Web UI 上輕鬆設計混沌場景並監控實驗狀態。

## 核心優勢

作為業界領先的混沌測試平台，Chaos Mesh 具備以下核心優勢：

- 穩定核心能力：Chaos Mesh 源自 [TiDB](https://github.com/pingcap/tidb) 核心測試平台，從初始版本即繼承了大量 TiDB 的現有測試經驗。

- 全面實戰驗證：Chaos Mesh 已被騰訊、美團等眾多企業組織採用，同時應用於 Apache APISIX、RabbitMQ 等知名分散式系統的測試體系。

- 易用系統設計：Chaos Mesh 充分運用自動化機制，結合圖形化操作與 Kubernetes 基礎的使用方式。

- 雲原生支援：憑藉強大的自動化能力，Chaos Mesh 全面支援 Kubernetes 環境。

- 多元故障模擬場景：涵蓋分散式測試系統中絕大部分基礎故障模擬場景。

- 靈活實驗編排能力：可在平台上設計自訂混沌實驗場景，包括多重混合實驗與應用狀態檢查。

- 高安全性設計：採用多層安全控制機制，提供高度安全保障。

- 活躍社群生態：作為 CNCF 孵化專案，Chaos Mesh 擁有持續增長的全球[貢獻者](https://github.com/chaos-mesh/chaos-mesh/graphs/contributors)與[採用者](https://github.com/chaos-mesh/chaos-mesh/blob/master/ADOPTERS.md)。

- 易擴展性：可輕鬆新增故障測試類型與功能。

## 架構概述

Chaos Mesh 基於 Kubernetes CRD（自訂資源定義）構建。為管理不同混沌實驗，Chaos Mesh 根據故障類型定義多種 CRD 類型，並針對不同 CRD 物件實現獨立控制器。其主要包含三個組件：

- **Chaos Dashboard**：可視化組件，提供直觀的 Web 介面供使用者操作與觀察混沌實驗，同時具備 RBAC 權限管理機制。

- **Chaos Controller Manager**：核心邏輯組件，主要負責混沌實驗的調度與管理，包含 Workflow Controller、Scheduler Controller 及多種故障類型的控制器。

- **Chaos Daemon**：主要執行組件，以 [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 模式運行（預設具備 Privileged 權限，可停用），透過侵入目標 Pod 命名空間干擾特定網路設備、檔案系統與核心。

![Architecture](img/architecture.png)

如上圖所示，Chaos Mesh 整體架構從上至下可分為三部分：

- 使用者輸入與觀察：使用者操作（User）產生的輸入抵達 Kubernetes API Server。使用者不直接與 Chaos Controller Manager 互動，所有操作最終體現為混沌資源變更（如 NetworkChaos 資源的變化）。

- 監控資源變更、調度工作流程，並執行混沌實驗：Chaos Controller Manager 僅接受來自 Kubernetes API Server 的事件。這些事件描述了某個混沌資源的變更，例如新的 Workflow 物件或混沌物件的建立。

- 注入特定節點故障：Chaos Daemon 元件主要負責接受來自 Chaos Controller Manager 元件的命令，入侵目標 Pod 的命名空間，並執行特定的故障注入。例如，設定 TC 網路規則、啟動 stress-ng 程序以搶佔 CPU 或記憶體資源。
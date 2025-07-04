---
slug: /
title: Chaos Mesh Overview
---

本文件描述 Chaos Mesh 的概念、使用場景、核心優勢及其架構。

## Chaos Mesh 概述

Chaos Mesh 是一個開源的雲原生混沌工程平台，提供多種類型的故障模擬能力，並具備強大的故障場景編排能力。

使用 Chaos Mesh，您可以方便地在開發、測試及生產環境中模擬現實可能發生的各種異常狀況，找出系統潛在問題。為降低混沌工程專案門檻，Chaos Mesh 提供視覺化操作介面，讓您能在 Web UI 上輕鬆設計混沌場景並監控實驗狀態。

## 核心優勢

作為業界領先的混沌測試平台，Chaos Mesh 具備以下核心優勢：

- 穩定的核心能力：Chaos Mesh 起源自 [TiDB](https://github.com/pingcap/tidb) 核心測試平台，從初始版本即繼承了大量 TiDB 現有測試經驗。

- 經過充分驗證：Chaos Mesh 已被騰訊、美團等眾多企業組織採用，並應用於 Apache APISIX、RabbitMQ 等知名分散式系統的測試體系。

- 易用性系統：充分運用自動化能力，提供圖形化操作與基於 Kubernetes 的使用方式。

- 雲原生特性：憑藉強大自動化能力，完美支援 Kubernetes 環境。

- 多樣化故障模擬場景：涵蓋分散式測試系統中多數基礎故障模擬情境。

- 靈活的實驗編排能力：可在平台上設計自訂混沌實驗場景，包括多重混合實驗與應用狀態檢查。

- 高安全性：設計多層安全控制機制，提供高度安全保障。

- 活躍的社群：作為 CNCF 孵化專案，全球[貢獻者](https://github.com/chaos-mesh/chaos-mesh/graphs/contributors)與[採用者](https://github.com/chaos-mesh/chaos-mesh/blob/master/ADOPTERS.md)持續增長。

- 易於擴展：可輕鬆新增故障測試類型與功能。

## 架構概述

Chaos Mesh 基於 Kubernetes CRD（自定義資源定義）建構。為管理不同混沌實驗，根據故障類型定義多種 CRD，並為各 CRD 物件實現獨立控制器。主要包含三個組件：

- **Chaos Dashboard**：可視化組件，提供易用 Web 介面供用戶操作與觀察混沌實驗，同時具備 RBAC 權限管理機制。

- **Chaos Controller Manager**：核心邏輯組件，主要負責混沌實驗的調度管理。包含多個 CRD 控制器，如 Workflow Controller、Scheduler Controller 及各類故障控制器。

- **Chaos Daemon**：主要執行組件，以 [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 模式運行，預設具備 Privileged 權限（可關閉）。主要透過侵入目標 Pod 命名空間，對特定網路設備、檔案系統及核心進行干擾。

![架構](img/architecture.png)

如上圖所示，Chaos Mesh 整體架構從上至下可分為三部分：

- 用戶輸入與觀察：用戶操作始於使用者端，經 Kubernetes API Server 傳遞。用戶不直接與 Chaos Controller Manager 互動，所有操作最終體現為 Chaos 資源變更（如 NetworkChaos 資源變動）。

- 監控資源變更、排程工作流程並執行混沌實驗：Chaos Controller Manager 僅接受來自 Kubernetes API Server 的事件。這些事件描述了特定混沌資源的變更，例如新的工作流程物件或混沌物件的建立。

- 注入特定節點故障：Chaos Daemon 元件主要負責接受來自 Chaos Controller Manager 元件的指令，入侵目標 Pod 的命名空間，並執行特定的故障注入。例如設定 TC 網路規則、啟動 stress-ng 程序來搶佔 CPU 或記憶體資源。
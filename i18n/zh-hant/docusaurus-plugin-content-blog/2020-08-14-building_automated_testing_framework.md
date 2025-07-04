---
slug: /building_automated_testing_framework
title: Building an Automated Testing Framework Based on Chaos Mesh and Argo
authors: chaos-mesh
image: /img/blog/automated_testing_framework.png
tags: [Chaos Mesh, Chaos Engineering, Test Automation]
---

![TiPocket - 自動化測試框架](/img/blog/automated_testing_framework.png)

[Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh) 是一個開源的 Kubernetes 混沌工程平台。儘管它提供了豐富的能力來模擬異常系統狀況，但它仍然只解決了混沌工程難題的一小部分。除了故障注入外，完整的混沌工程應用還包括圍繞定義的穩定狀態提出假設、在生產環境中運行實驗、透過測試案例驗證系統，以及實現測試自動化。

本文將說明我們如何使用 [TiPocket](https://github.com/pingcap/tipocket) 這套自動化測試框架，為我們的分散式資料庫 TiDB 構建完整的混沌工程測試循環。

<!--truncate-->

## 為什麼我們需要 TiPocket？

在將 [TiDB](https://github.com/pingcap/tidb) 這類分散式系統投入生產環境前，我們必須確保其具備足夠的穩健性以應對日常使用。為此，數年前我們將混沌工程引入測試框架。在我們的測試框架中，我們：

1. 觀察正常指標並建立測試假設。

2. 將一系列故障注入 TiDB。

3. 運行各類測試案例以在故障情境下驗證 TiDB。

4. 監控並收集測試結果以進行分析與診斷。

這聽似穩固的流程已運作多年，然而隨著 TiDB 演進，測試規模呈倍數增長。我們面臨多重故障情境，需在 Kubernetes 測試集群中運行數十項測試案例。即使有 Chaos Mesh 協助注入故障，剩餘工作仍相當繁重——更遑論實現流程自動化以提升測試擴展性與效率的挑戰。

因此我們開發了 TiPocket，這套基於 Kubernetes 與 Chaos Mesh 的全自動測試框架。目前我們主要用於測試 TiDB 集群，但得益於 TiPocket 的 Kubernetes 友好設計與可擴展介面，您可透過 Kubernetes 的創建/刪除邏輯輕鬆支援其他應用。

## 運作原理

基於上述需求，我們需要一套能實現以下功能的自動化工作流程：

- [注入混沌](#injecting-chaos---chaos-mesh)

- [驗證混沌影響](#verifying-chaos-impacts-test-cases)

- [自動化混沌流程](#automating-the-chaos-pipeline---argo)

- [視覺化結果](#visualizing-the-results-loki)

### 注入混沌 - Chaos Mesh

故障注入是混沌測試的核心。在分散式資料庫中，故障隨時隨地可能發生——從節點崩潰、網路分割、檔案系統故障到核心恐慌皆屬此類。這正是 Chaos Mesh 的用武之地。

目前 TiPocket 支援下列故障注入類型：

- **網路**：模擬網路分割、隨機封包遺失、亂序、重複或鏈路延遲。

- **時間偏移**：模擬待測試容器的時鐘偏移。

- **終止**：終止指定 pod，可於集群中隨機選擇或限定於特定元件（TiDB、TiKV 或 Placement Driver (PD)）。

- **I/O**：在 TiDB 的儲存引擎 TiKV 中注入 I/O 延遲，以識別 I/O 相關問題。

完成故障注入後，我們需思考驗證機制：如何確保 TiDB 能在這些故障中維持運作？

## 驗證混沌影響：測試案例

為了驗證 TiDB 在混沌環境中的穩定性，我們在 TiPocket 中實現了數十種測試案例，並結合多種檢測工具。以下概述 TiPocket 如何在故障事件中驗證 TiDB，這些案例聚焦於 SQL 執行、交易一致性與交易隔離性。

### 模糊測試：SQLsmith

[SQLsmith](https://github.com/pingcap/tipocket/tree/master/pkg/go-sqlsmith) 是一款隨機生成 SQL 查詢的工具。TiPocket 創建一個 TiDB 集群和一個 MySQL 實例，將 SQLsmith 生成的隨機 SQL 語句在 TiDB 和 MySQL 上執行，同時向 TiDB 集群注入各類故障進行測試。最終比對執行結果，若檢測到不一致，則表示系統可能存在潛在問題。

### 交易一致性測試：Bank 與 Porcupine

[Bank](https://github.com/pingcap/tipocket/tree/master/cmd/bank) 是模擬銀行系統轉帳流程的經典測試案例。在快照隔離級別下，所有轉帳操作必須確保所有帳戶總金額在任何時刻都保持一致，即使面對系統故障亦然。若總金額出現不一致，則表示系統可能存在潛在問題。

[Porcupine](https://github.com/anishathalye/porcupine) 是基於 Go 語言構建的線性化檢查工具，用於驗證分散式系統的正確性。它接收可執行的 Go 代碼作為順序規範，結合併發歷史記錄，判斷該歷史是否符合順序規範的線性化要求。在 TiPocket 中，我們通過 [Porcupine](https://github.com/pingcap/tipocket/tree/master/pkg/check/porcupine) 檢查器在多個測試案例中驗證 TiDB 是否符合線性化約束。

### 交易隔離性測試：Elle

[Elle](https://github.com/jepsen-io/elle) 是驗證資料庫交易隔離級別的檢測工具。TiPocket 整合了 Elle 工具的 Go 語言實現 [go-elle](https://github.com/pingcap/tipocket/tree/master/pkg/elle)，用以驗證 TiDB 的隔離級別。

以上僅是 TiPocket 用於驗證 TiDB 準確性與穩定性的部分測試案例。更多測試案例與驗證方法，請參閱我們的[源碼庫](https://github.com/pingcap/tipocket)。

## 自動化混沌管線：Argo

現在我們擁有 Chaos Mesh 注入故障、待測 TiDB 集群以及驗證方法，如何實現混沌測試管線的自動化？兩種方案浮現：在 TiPocket 中實現調度功能，或將任務移交現有開源工具。為使 TiPocket 更專注於工作流的測試環節，我們選擇開源工具方案。結合全 Kubernetes 架構設計，這直接引導我們採用 [Argo](https://github.com/argoproj/argo)。

Argo 是專為 Kubernetes 設計的工作流引擎，作為開源產品已運作多年，並獲得廣泛關注與應用。

Argo 為工作流抽象了多種自定義資源定義（CRD），其中最重要的是 Workflow Template、Workflow 和 Cron Workflow。以下是 Argo 在 TiPocket 中的應用方式：

- **Workflow Template** 是預先定義的測試任務模板，執行時可傳入參數。

- **Workflow** 調度多個工作流模板形成不同順序的執行任務。Argo 還支持在管線中添加條件判斷、循環及有向無環圖（DAG）。

- **Cron Workflow** 可像定時任務般調度工作流，完美適用於需長期運行測試任務的場景。

以下展示我們預定義的 bank 測試工作流範例：

```yml
spec:
  entrypoint: call-tipocket-bank
  arguments:
    parameters:
      - name: ns
        value: tipocket-bank
            - name: nemesis
        value: random_kill,kill_pd_leader_5min,partition_one,subcritical_skews,big_skews,shuffle-leader-scheduler,shuffle-region-scheduler,random-merge-scheduler
  templates:
    - name: call-tipocket-bank
      steps:
        - - name: call-wait-cluster
            templateRef:
              name: wait-cluster
              template: wait-cluster
        - - name: call-tipocket-bank
            templateRef:
              name: tipocket-bank
              template: tipocket-bank
```

在此範例中，我們使用工作流程範本和故障參數來定義要注入的特定故障。您可以重複使用該範本來定義多個適用於不同測試案例的工作流程，從而能在流程中添加更多自訂的故障注入。

除了 [TiPocket 的](https://github.com/pingcap/tipocket/tree/master/argo/workflow) 範例工作流程和範本外，此設計還允許您添加自己的故障注入流程。透過可編碼的工作流程處理複雜邏輯，使 Argo 對開發者友好，成為我們場景的理想選擇。

現在，我們的混沌實驗已自動運行。但如果結果不符合預期呢？該如何定位問題？TiDB 儲存了多種監控資訊，這使得日誌收集對於在 TiPocket 中實現可觀測性至關重要。

## 視覺化結果：Loki

在雲原生系統中，可觀測性非常重要。一般而言，您可以透過**指標（metrics）**、**日誌（logging）**和**追蹤（tracing）**實現可觀測性。TiPocket 的主要測試案例評估 TiDB 叢集，因此指標和日誌是我們定位問題的預設來源。

On Kubernetes, Prometheus is the de-facto standard for metrics. However, there is no common way for log collection. Solutions such as [Elasticsearch](https://en.wikipedia.org/wiki/Elasticsearch), [Fluent Bit](https://fluentbit.io/), and [Kibana](https://www.elastic.co/kibana) perform well, but they may cause system resource contention and high maintenance costs. We decided to use [Loki](https://github.com/grafana/loki), the Prometheus-like log aggregation system from [Grafana](https://grafana.com/).

Prometheus 處理 TiDB 的監控資訊。由於 Prometheus 和 Loki 具有相似的標籤系統，我們能輕鬆將 Prometheus 監控指標與對應的 pod 日誌結合，並使用相似的查詢語言。Grafana 也支援 Loki 儀表板，意味著我們可用 Grafana 同時顯示監控指標和日誌。Grafana 是 TiDB 的內建監控元件，Loki 可直接重複利用它。

## 整合所有元件 - TiPocket

現在，萬事俱備。以下是 TiPocket 的簡化架構圖：

![TiPocket 架構](/img/blog/tipocket-architecture.png)

如您所見，Argo 工作流程管理所有混沌實驗和測試案例。通常，完整的測試週期包含以下步驟：

1. Argo 建立 Cron Workflow，定義待測叢集、待注入故障、測試案例及任務持續時間。必要時，Cron Workflow 還允許即時查看案例日誌。

![Argo 工作流程](/img/blog/argo-workflow.png)

1. 在指定時間，工作流程中啟動獨立的 TiPocket 執行緒觸發 Cron Workflow。TiPocket 向 TiDB-Operator 發送待測叢集定義，由 TiDB-Operator 建立目標 TiDB 叢集。同時，Loki 收集相關日誌。

2. Chaos Mesh 在叢集中注入故障。

3. 使用者透過前述測試案例驗證系統健康狀態。任何測試案例失敗都會導致 Argo 工作流程失敗，觸發 Alertmanager 將結果發送至指定 Slack 頻道。若測試案例正常完成，則清除叢集，Argo 待命至下次測試。

![Slack 警報訊息](/img/blog/alert_message.png)

這就是完整的 TiPocket 工作流程。

## 加入我們

[Chaos Mesh](https://github.com/pingcap/chaos-mesh) 與 [TiPocket](https://github.com/pingcap/tipocket) 都在積極迭代發展中。我們已將 Chaos Mesh 捐贈給 [CNCF](https://github.com/cncf/toc/pull/367)，並期待更多社群成員加入我們，共同構建完整的混沌工程生態系統。若您對此感興趣，請查看我們的[網站](https://chaos-mesh.org/)，或加入 [CNCF Slack](hthttps://slack.cncf.io/) 中的 #project-chaos-mesh 頻道。
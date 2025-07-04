---
slug: /chaos-mesh-remake-one-step-closer-towards-chaos-as-a-service
title: 'Chaos Mesh Remake: One Step Closer toward Chaos as a Service'
authors:
  - xiangwang
  - changyu
image: /img/blog/chaos-engineering-tools-as-a-service.jpeg
tags: [Chaos Mesh, Chaos Engineering]
---

![混沌工程工具](/img/blog/chaos-engineering-tools-as-a-service.jpeg)

[Chaos Mesh](https://chaos-mesh.org/) 是一個雲原生的混沌工程平台，可在 Kubernetes 環境中協調混亂。透過 Chaos Mesh，您可以將各種故障注入 Pod、網路、檔案系統甚至核心，測試系統在 Kubernetes 環境下的韌性和穩健性。

<!--truncate-->

自開源並被雲原生計算基金會（CNCF）接納為沙箱專案以來，Chaos Mesh 吸引了全球貢獻者並協助用戶測試系統。然而仍有許多改進空間：

- 需提升易用性：部分功能操作複雜，例如應用混沌實驗時需手動檢查實驗是否啟動

- 主要適用於 Kubernetes 環境：因無法管理多個 Kubernetes 叢集，需為每個叢集部署 Chaos Mesh。雖然 [chaosd](https://github.com/chaos-mesh/chaosd) 支援實體機混沌實驗，但功能有限且命令列操作不夠友善

- 不支援外掛機制：需修改原始碼才能應用客製化混沌實驗，且僅支援 Golang

Chaos Mesh 無疑是一流的混沌工程平台，但距離提供混沌工程即服務（CaaS）仍有差距。因此在 [TiDB Hackathon 2020](https://pingcap.com/community-activity/tidb-hackathon-2020/) 中，**我們重構了 Chaos Mesh 架構，使其向 CaaS 邁進了一步**。

本文將探討 CaaS 的概念、我們如何透過 Chaos Mesh 實現，以及相關規劃與經驗教訓，希望這些經驗對您建立混沌工程系統有所助益。

## 什麼是混沌工程即服務（CaaS）？

正如 Gremlin 共同創辦人 Matt Fornaciari [所述](https://jaxenter.com/chaos-engineering-service-144113.html)，CaaS 意味著「您將獲得直觀的 UI、客戶支援、開箱即用的整合方案，以及數分鐘內即可開始實驗所需的一切」。

我們認為 CaaS 應提供：

- 統一管理控制台：可編輯配置並創建混沌實驗

- 實驗狀態視覺化指標

- 暫停或歸檔實驗的操作功能

- 簡易互動：透過拖放物件即可協調實驗

部分企業如[網易伏羲實驗室](https://pingcap.com/blog/how-a-top-game-company-uses-chaos-engineering-to-improve-testing)和 FreeWheel 已客製化 Chaos Mesh 滿足需求，使其成為 CaaS 的雛形。

## 推動 Chaos Mesh 邁向 CaaS

基於對 CaaS 的理解，我們在駭客松期間優化了 Chaos Mesh 架構，包括增強多系統支援和可觀測性。程式碼詳見 [wuntun/chaos-mesh](https://github.com/wuntun/chaos-mesh/tree/caas) 和 [wuntun/chaosd](https://github.com/wuntun/chaosd/tree/caas)。

### 重構 Chaos Dashboard

當前 Chaos Mesh 架構適用於單一 Kubernetes 叢集，其網頁 UI Chaos Dashboard 綁定於特定 Kubernetes 環境：

![Chaos Mesh 架構](/img/blog/chaos-mesh-remake-architecture.jpeg)

在此次重構中，**為了讓 Chaos Dashboard 能管理多個 Kubernetes 集群，我們將其從主架構中分離出來**。現在，若將 Chaos Dashboard 部署在 Kubernetes 集群外部，可透過網頁 UI 添加集群；若部署在集群內部，則會透過環境變數自動獲取集群資訊。

您可以將 Chaos Mesh（技術上是 Kubernetes 配置）註冊到 Chaos Dashboard，或要求 `chaos-controller-manager` 透過配置向 Chaos Dashboard 報告。Chaos Dashboard 與 `chaos-controller-manager` 透過 CustomResourceDefinitions (CRDs) 互動。當 `chaos-controller-manager` 偵測到 Chaos Mesh CRD 事件時，會調用 `chaos-daemon` 執行相關混沌實驗。因此，Chaos Dashboard 能透過操作 CRDs 來管理實驗。

### 重構 chaosd

chaosd 是在實體機器上運行混沌實驗的工具包。此前它僅是命令列工具且功能有限。

![chaosd，一個混沌工程命令列工具](/img/blog/chaosd-chaos-engineering-command-line-tool.jpeg)

在重構過程中，**我們讓 chaosd 支援 RESTful API 並強化服務，使其能透過解析 CRD 格式的 JSON 或 YAML 檔案來配置混沌實驗**。

現在，chaosd 可透過配置向 Chaos Dashboard 註冊自身，並定期發送心跳信號。Chaos Dashboard 透過心跳信號管理 chaosd 節點狀態。您也能透過網頁 UI 將 chaosd 節點加入 Chaos Dashboard。

此外，**chaosd 現在能排程指定時間的混沌實驗並管理實驗生命週期，這統一了 Kubernetes 與實體機器上的使用者體驗**。

透過新版 Chaos Dashboard 和 chaosd，Chaos Mesh 的優化架構如下：

![Chaos Mesh 優化後的架構](/img/blog/chaos-mesh-optimized-architecture.jpeg)

### 提升可觀測性

另一項改進是可觀測性，即如何判斷實驗是否成功執行。

改進前，您必須手動檢查實驗指標。若向 Pod 注入 [StressChaos](https://chaos-mesh-website-archived.netlify.app/docs/1.2.4/chaos_experiments/stresschaos)，需進入 Pod 查看是否存在 `stress-ng` 進程，再用 `top` 命令檢查 CPU 和記憶體使用率。這些指標能判斷 StressChaos 實驗是否成功創建。

為簡化流程，我們將 `node_exporter` 整合至 `chaos-daemon` 和 chaosd 以收集節點指標。同時在 Kubernetes 集群部署 `kube-state-metrics`，結合 cadvisor 收集 Kubernetes 指標。收集的指標由 Prometheus 和 Grafana 儲存並視覺化，提供簡易方法檢查實驗狀態。

#### 需要進一步的改進

整體而言，指標旨在協助您：

- 確認混沌注入是否成功。

- 觀察混沌對服務的影響並進行週期性分析。

- 應對異常混沌事件。

為達成這些目標，系統需監控實驗數據指標、常規指標及實驗事件。Chaos Mesh 仍需改進：

- 實驗數據指標，例如注入的網路延遲確切時長、模擬工作負載的具體壓力值。

- 實驗事件，即創建、刪除和運行實驗的 Kubernetes 事件。

[Litmus](https://github.com/litmuschaos/chaos-exporter#example-metrics) 提供了優良的指標範例。

## Chaos Mesh 的其他提案

由於 Hackathon 時間有限，我們未完成所有計畫。以下是供 Chaos Mesh 社群未來參考的部分提案。

### 編排

Chaos Engineering 的閉環包含四個步驟：探索混亂、發現系統缺陷、分析根本原因，以及發送回饋進行改進。

![Chaos Engineering 的閉環](/img/blog/closed-loop-of-chaos-engineering.jpeg)

然而，**目前大多數開源混沌工程工具僅專注於探索階段，未提供實質性回饋機制。** 基於改進後的可觀測性元件，我們能即時監控混沌實驗，並比對分析實驗結果。

透過這些結果，我們可藉由新增關鍵元件「編排」實現閉環。Chaos Mesh 社群已提出 [Workflow](https://github.com/chaos-mesh/rfcs/pull/10/files) 功能，讓您輕鬆編排與回呼混沌實驗，或無縫整合 Chaos Mesh 至其他系統。您可在 CI/CD 階段或金絲雀發布後執行混沌實驗。

**結合可觀測性與編排能力，將形成混沌工程的閉環回饋系統。** 若對 Pod 執行 100 毫秒網路延遲測試，您可透過可觀測性元件監控延遲變化，並使用基於編排的 PromQL 或其他 DSL 驗證 Pod 服務可用性。若服務不可用，即可推論當延遲 >=100 毫秒時服務失效。

但 100 毫秒並非服務的臨界值；您需知曉服務能承受的最大延遲。透過編排混沌實驗數值，您將掌握達成服務級別目標（SLO）的關鍵閾值。同時，您能了解服務在不同網路條件下的效能表現，檢驗是否符合預期。

### 資料格式

Chaos Mesh 使用 CRD 定義混沌物件。若能將 CRD 轉換為 JSON 檔案，即可實現元件間通訊。

在資料格式層面，chaosd 僅需接收並註冊 JSON 格式的 CRD 資料。任何混沌工具若能接收 CRD 資料並自行註冊，即可在多種場景執行混沌實驗。

### 外掛機制

Chaos Mesh 的外掛支援有限。目前僅能透過在 Kubernetes API 註冊 CRD 來[新增混沌實驗](https://chaos-mesh-website-archived.netlify.app/docs/1.2.4/development_guides/develop_a_new_chaos/)，這導致兩大問題：

- 必須使用與 Chaos Mesh 相同的 Golang 語言開發外掛。

- 必須將擴充程式碼合併至 Chaos Mesh 專案。由於缺乏類似 Berkeley Packet Filter (BPF) 的安全機制，合併外掛程式碼可能引入額外風險。

為實現完整外掛支援，需探索新的擴充方法。Chaos Mesh 本質基於 CRD 執行混沌實驗，實驗僅需生成、監聽與刪除 CRD。對此我們有幾個可行方案：

- 開發控制器（controller）或運算子（operator）管理 CRD。

- 統一處理 CRD 事件，透過 HTTP 回呼操作 CRD。此方法僅需 HTTP API，無需 Golang 技能，可參考 [Whitebox Controller](https://github.com/summerwind/whitebox-controller)。

- 採用 WebAssembly (Wasm)。需呼叫混沌實驗邏輯時，直接執行 Wasm 程式。

- 使用 SQL 查詢混沌實驗狀態。基於 CRD 特性，可直接用 SQL 操作 Kubernetes，例如 [Presto connector](https://github.com/xuxinkun/kubesql) 與 [osquery extension](https://github.com/aquasecurity/kube-query)。

- 採用 SDK 擴充方案，例如 [Chaos Toolkit](https://docs.chaostoolkit.org/reference/api/experiment/)。

### 混沌工具整合

在真實系統中，單一混沌工程工具難以覆蓋所有用例。因此整合其他混沌工具，將大幅強化混沌工程生態系能力。

市場上有眾多混沌工程工具。Litmus 的 [Kubernetes 實現](https://github.com/litmuschaos/litmus-go/tree/2.14.1/chaoslib/powerfulseal)基於 [PowerfulSeal](https://github.com/powerfulseal/powerfulseal)，而其[容器實現](https://github.com/litmuschaos/litmus-go/tree/2.14.1/chaoslib/pumba)則基於 [Pumba](https://github.com/alexei-led/pumba)。[Kraken](https://github.com/cloud-bulldozer/kraken) 專注於 Kubernetes，[AWSSSMChaosRunner](https://github.com/amzn/awsssmchaosrunner) 專注於 AWS，而 [Toxiproxy](https://github.com/shopify/toxiproxy) 則針對 TCP。此外還有基於 [Envoy](https://docs.google.com/presentation/d/1gMlmXqH6ufnb8eNO10WqVjqrPRGAO5-1S1zjcGo1Zr4/edit#slide=id.g58453c664c_2_75) 和 Istio 的合併專案。

為管理各種混沌工具，我們可能需要統一模式，例如 [Chaos Hub](https://hub.litmuschaos.io/)。

## 來自社群的呼聲

在此，我們分享中國一家領先網絡安全公司（同時也是 Chaos Mesh 使用者）如何調整 Chaos Mesh 以滿足其需求。他們的調整分為三個層面：實體節點、容器和應用程式。

### 實體節點

- 支援在實體伺服器上執行腳本。您可在 CRD 中配置腳本目錄，並透過 `chaos-daemon` 執行腳本。

- 使用自訂腳本模擬重啟、關機和核心恐慌。

- 使用自訂腳本關閉節點的網路介面卡（NIC）。

- 使用 sysbench 建立頻繁的上下文切換，模擬「嘈雜鄰居」效應。

- 透過傳遞和過濾 PID，使用 BPF 的 `seccomp` 攔截容器系統呼叫。

### 容器

- 隨機變更 Deployment 副本數量，測試應用程式流量是否異常。

- 基於 CRD 物件嵌入：在混沌 CRD 中填入 Ingress 物件以模擬介面速率限制。

- 基於 CRD 物件嵌入：在混沌 CRD 中填入 Cilium 網路政策物件以模擬波動網路狀況。

### 應用程式

- 支援執行自訂任務。目前 Chaos Mesh 透過 `chaos-daemon` 注入混沌，不保證排程公平性與親和性。為解決此問題，可使用 `chaos-controller-manager` 直接為不同 CRD 建立任務。

- 支援在自訂任務中執行 [Newman](https://github.com/postmanlabs/newman)，隨機變更 HTTP 參數。這用於在 HTTP 介面實施混沌實驗，模擬使用者執行異常行為時的情況。

## 總結

傳統故障測試針對系統中預期易受攻擊的特定點，通常是斷言式：特定條件產生特定結果。

**混沌工程更強大之處在於幫助您發現「未知的未知」**。透過在更廣泛領域探索，混沌工程加深您對被測系統的理解，並挖掘出新資訊。

總而言之，這些是我們對混沌工程與 Chaos Mesh 的個人思考與實踐。我們的駭客松專案尚未投入生產，但希望能闡明 CaaS（混沌即服務），並為 Chaos Mesh 繪製前瞻性路線圖。若您有興趣建構混沌即服務，[請加入我們的 Slack](https://slack.cncf.io/) (#project-chaos-mesh)！
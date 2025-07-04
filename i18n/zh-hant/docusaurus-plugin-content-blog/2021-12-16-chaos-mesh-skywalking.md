---
slug: /better-observability-for-chaos-engineering
title: 'Chaos Mesh + SkyWalking: Better Observability for Chaos Engineering'
authors: ningxuanwang
image: /img/blog/chaos-mesh-skywalking-banner.png
tags: [Chaos Mesh, Chaos Engineering, Tutorials]
---

![Chaos Mesh + SkyWalking：強化混沌工程的可觀測性](/img/blog/chaos-mesh-skywalking-banner.png)

[Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh) 是一款開源雲原生[混沌工程](https://en.wikipedia.org/wiki/Chaos_engineering)平台。您可以使用 Chaos Mesh 便捷地注入故障並模擬現實中可能發生的異常狀況，從而識別系統潛在問題。Chaos Mesh 還提供 Chaos Dashboard 用於監控混沌實驗狀態，但該儀表板無法讓您觀察實驗中的故障如何影響應用程式的服務性能，這阻礙了我們進一步測試系統並發現潛在問題。

<!--truncate-->

[Apache SkyWalking](https://github.com/apache/skywalking) 是一款開源應用性能監控（APM）工具，專為監控、追蹤及診斷雲原生容器化分散式系統而設計。它會收集系統發生的事件並在儀表板上顯示，讓您直觀觀察系統中各類事件的數量、類型及其對服務性能的影響。

在混沌實驗中同時使用 SkyWalking 和 Chaos Mesh，您能清晰觀測不同故障對服務性能的影響程度。

本教學將示範如何配置 SkyWalking 與 Chaos Mesh，並教您如何結合兩套系統來監控事件，即時觀察混沌實驗對應用服務性能的影響。

## 準備工作

開始使用 SkyWalking 和 Chaos Mesh 前，您需要：

- 依照 [SkyWalking 配置指南](https://github.com/apache/skywalking-kubernetes#install) 建立 SkyWalking 集群

- 使用 [Helm 部署 Chaos Mesh](https://chaos-mesh.org/docs/production-installation-using-helm/)

- 安裝 [JMeter](https://jmeter.apache.org/index.html) 或其他 Java 測試工具（用於增加服務負載）

- 若僅需運行示範環境，請依 [此指南](https://github.com/chaos-mesh/chaos-mesh-on-skywalking) 配置 SkyWalking 與 Chaos Mesh

完成上述準備後，即可進入實作環節。

## 步驟 1：接入 SkyWalking 集群

安裝 SkyWalking 集群後，您可訪問其使用者介面（UI）。但此時尚未運行任何服務，因此在開始監控前，需先新增服務並設定代理。

本教學以輕量級微服務框架 Spring Boot 為例，建立簡化示範環境。

1. 參考 [此文件](https://github.com/chaos-mesh/chaos-mesh-on-skywalking/blob/master/demo-deployment.yaml) 在 Spring Boot 建立 SkyWalking 示範程式

2. 執行指令 `kubectl apply -f demo-deployment.yaml -n skywalking` 部署示範程式

部署完成後，您可在 SkyWalking UI 觀察即時監控結果。

**注意：** Spring Boot 與 SkyWalking 預設連接埠均為 8080。配置連接埠轉發時需特別留意，否則可能發生連接埠衝突。例如，可將 Spring Boot 連接埠設為 8079，使用指令如 `kubectl port-forward svc/spring-boot-skywalking-demo 8079:8080 -n skywalking` 避免衝突。

## 步驟 2：部署 SkyWalking Kubernetes 事件導出器

[SkyWalking Kubernetes 事件導出器](https://github.com/apache/skywalking-kubernetes-event-exporter) 能監控、過濾 Kubernetes 事件並發送至 SkyWalking 後端。SkyWalking 會將事件與系統指標關聯，顯示事件發生時機及其對指標的影響概況。

若想透過單行指令部署 SkyWalking Kubernetes 事件導出器，請參考[此文件](https://github.com/chaos-mesh/chaos-mesh-on-skywalking/blob/master/exporter-deployment.yaml)建立 YAML 格式設定檔，接著在過濾器與導出器中自訂參數。此時即可使用 `kubectl apply` 指令部署 SkyWalking Kubernetes 事件導出器。

## 步驟三：使用 JMeter 增加服務負載

為更清晰觀察服務效能變化，需增加 Spring Boot 的服務負載。本教程採用廣泛使用的 Java 測試工具 JMeter 來提升負載。

使用 JMeter 對 `localhost:8079` 執行壓力測試，新增五個執行緒持續增加服務負載。

![JMeter 儀表板 1](/img/blog/jmeter-1.png)

![JMeter 儀表板 2](/img/blog/jmeter-2.png)

開啟 SkyWalking 儀表板，可見存取率達 100%，服務負載約每分鐘 5,300 次呼叫 (CPM)。

![SkyWalking 儀表板](/img/blog/skywalking-dashboard.png)

## 步驟四：透過 Chaos Mesh 注入故障並觀察結果

完成上述三步驟後，即可使用 Chaos 儀表板模擬壓力情境，並在混沌實驗期間觀察服務效能變化。

![Chaos 儀表板上的 StressChaos](/img/blog/chaos-dashboard-stresschaos.png)

以下章節說明三種混沌條件下服務效能變化情況：

- CPU 負載：10%；記憶體負載：128 MB

  首次混沌實驗模擬低 CPU 使用率。點擊儀表板右側切換按鈕可顯示實驗起訖時間。將游標懸停於短綠線上可確認實驗狀態為「已應用」或「已恢復」。

  兩道短綠線之間時段，服務負載降至 4,929 CPM，實驗結束後恢復正常。

  ![測試 1](/img/blog/cpuload-1.png)

- CPU 負載：50%；記憶體負載：128 MB

  當應用程式 CPU 負載增至 50%，服務負載降至 4,307 CPM。

  ![測試 2](/img/blog/cpuload-2.png)

- CPU 負載：100%；記憶體負載：128 MB

  CPU 使用率達 100% 時，服務負載驟降至無混沌實驗狀態的 40%。

  ![測試 3](/img/blog/cpuload-3.png)

  由於 Linux 系統的行程排程機制禁止行程持續佔用 CPU，即使處於 CPU 滿載極端狀況，部署的 Spring Boot 示範程式仍能處理 40% 的存取請求。

## 總結

結合 SkyWalking 與 Chaos Mesh 可清晰觀察混沌實驗何時及如何影響應用服務效能。此工具組合讓您能在各種極端條件下監控服務表現，從而增強對服務穩定性的信心。

2021 年 Chaos Mesh 在 PingCAP 工程師與社群貢獻者不懈努力下取得長足進展。為持續提升多元用戶支援並深入理解混沌工程實踐經驗，誠邀您參與[本次問卷調查](https://www.surveymonkey.com/r/X77BCNM)，提供寶貴意見回饋。

若想進一步了解 Chaos Mesh，歡迎加入 [GitHub 上的 Chaos Mesh 社群](https://github.com/chaos-mesh) 或我們的 [Slack 討論區](https://slack.cncf.io/) (#project-chaos-mesh)。若在使用 Chaos Mesh 時發現任何錯誤或功能缺失，可提交 pull requests 或 issues 至我們的 [GitHub 儲存庫](https://github.com/chaos-mesh/chaos-mesh)。
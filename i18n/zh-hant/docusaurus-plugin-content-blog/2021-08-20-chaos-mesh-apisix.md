---
slug: /How-Chaos-Mesh-Helps-Apache-APISIX-Improve-System-Stability
title: 'How Chaos Mesh Helps Apache APISIX Improve System Stability'
authors: shuyangwu
image: /img/blog/chaos-mesh-apisix.jpeg
tags: [Chaos Mesh, Chaos Engineering]
---

![Chaos Mesh 協助 Apache APISIX 提升系統穩定性](/img/blog/chaos-mesh-apisix.jpeg)

[Apache APISIX](https://github.com/apache/apisix) 是一個雲原生、高效能、可擴展的微服務 API 閘道器。作為 Apache 軟體基金會的頂級專案，它為全球數百家企業處理關鍵任務流量，涵蓋金融、互聯網、製造、零售和電信營運商等領域。我們的客戶包括 NASA、歐盟數位工廠、中國移動和騰訊。

<!--truncate-->

隨著社群發展，Apache APISIX 的功能與外部元件的互動日益頻繁，系統複雜度隨之提升，錯誤發生可能性也隨之增加。為識別潛在系統故障並建立對生產環境的信心，我們引入了混沌工程（Chaos Engineering）概念。

![Apache APISIX 架構](/img/blog/apache-apisix-architecture.jpg)

本文將分享我們如何運用 [Chaos Mesh](https://chaos-mesh.org/) 提升系統穩定性。

## 我們的痛點

Apache APISIX 每日處理數百億次請求。在此流量規模下，使用者發現了以下問題：

- **情境一：** 在 Apache APISIX 的配置中心，當 etcd 與 Apache APISIX 之間發生意外高網路延遲時，Apache APISIX 是否仍能正常過濾並轉發流量？

- **情境二：** 當 etcd 集群中某節點故障且集群仍可正常運行時，該節點與 Apache APISIX 管理 API 的互動會報錯。

儘管 Apache APISIX 透過持續整合（CI）中的單元測試、端到端（E2E）測試和模糊測試覆蓋了多數場景，但仍未涵蓋與外部元件的互動情境。若系統出現異常（如網路抖動、硬碟故障或行程被殺），Apache APISIX 能否給出適當錯誤訊息？能否持續運行或自我恢復至正常狀態？

## 為何選擇 Chaos Mesh

為測試這些使用者情境，並在產品上線前發現類似問題，我們的社群決定採用 Chaos Mesh 進行混沌測試。

Chaos Mesh 是雲原生的混沌工程平台，提供全方位故障注入方法，涵蓋 Kubernetes 上複雜系統的 Pod、網路、檔案系統乃至核心層級故障。它能協助使用者發掘系統弱點，確保系統在生產環境中能抵禦失控狀況。

與 Apache APISIX 相同，Chaos Mesh 擁有活躍的開源社群。我們深知活躍社群能確保軟體穩定使用與快速迭代，這使得 Chaos Mesh 更具吸引力。

## 我們如何在 APISIX 運用 Chaos Mesh

混沌工程已超越單純的故障注入，發展為完整的方法論。建立混沌實驗時，我們先定義應用程式的正常運作狀態（「穩定狀態」），隨後注入潛在問題以觀察系統反應。若問題導致應用偏離穩定狀態，我們便進行修復。

現在，我們將透過前述兩個情境，展示如何在 Apache APISIX 中運用 Chaos Mesh。

### 情境一

我們透過以下步驟部署混沌工程實驗：

1. 確立衡量 Apache APISIX 運作是否正常的指標。實驗中最關鍵的方法是使用 Grafana 監控 Apache APISIX 運行指標。我們從 CI 中的 Prometheus 提取數據進行比對，採用路由轉發請求每秒處理量（RPS）和 etcd 連線狀態作為評估指標。同時分析日誌：針對 Apache APISIX，我們檢查 Nginx 錯誤日誌以判斷是否存在錯誤，以及錯誤是否符合預期。

2. 我們在控制組進行測試，發現 `create route` 和 `access route` 都成功執行，且能正常連接 etcd。我們記錄了 RPS 數據。

3. 我們使用網路混沌注入 5 秒網路延遲後重新測試。這次 `set route` 失敗，但 `get route` 成功，etcd 仍可連接，且 RPS 與前次實驗相比無顯著變化。實驗結果符合預期。

![etcd 與 Apache APISIX 之間發生高網路延遲](/img/blog/high-network-latency-between-etcd-and-apache-apisix.jpg)

### 場景 #2

在控制組重複前述實驗後，我們注入 pod-kill 混沌並重現預期錯誤。隨機刪除 etcd 叢集中少量節點時，APISIX 有時能連接 etcd 有時失敗，日誌出現大量連線拒絕錯誤。

當刪除 etcd 端點列表中第一或第三個節點時，`set route` 正常返回結果；但刪除列表中第二個節點時，`set route` 返回 "connection refused" 錯誤。

排查發現 Apache APISIX 使用的 etcd Lua API 採用順序選擇端點而非隨機機制。因此建立 etcd 客戶端時，僅綁定單一 etcd 端點，導致持續失敗。

修復此問題後，我們在 etcd Lua API 加入健康檢查機制，確保大量請求不會發往已斷線的 etcd 節點。同時新增後備機制，避免 etcd 叢集完全斷線時日誌被錯誤訊息淹沒。

![etcd 節點互動回報錯誤](/img/blog/error-reported-from-etcd-node-interaction.jpg)

## 未來規劃

### 在 E2E 模擬場景執行混沌測試

目前 Apache APISIX 採手動識別系統弱點進行測試修復。雖在 CI 環境測試無需擔憂混沌工程影響生產環境，但仍無法覆蓋生產環境中複雜全面的應用場景。

為涵蓋更多場景，社群計劃運用現有 E2E 測試框架模擬更完整場景，執行隨機性更高、覆蓋範圍更廣的混沌測試。

### 將混沌測試擴展至更多 Apache APISIX 專案

除持續為 Apache APISIX 挖掘漏洞外，社群計劃將混沌測試導入更多專案，如 Apache APISIX Dashboard 與 Apache APISIX Ingress Controller。

### 為 Chaos Mesh 新增功能

部署 Chaos Mesh 時，部分功能暫未支援，例如無法選擇服務作為網路延遲目標，或指定容器連接埠注入網路混沌。未來 Apache APISIX 社群將協助 Chaos Mesh 新增相關功能。

歡迎至 GitHub 貢獻 [Apache APISIX 專案](https://github.com/apache/apisix)。若對 Chaos Mesh 感興趣並希望改進，請加入 [Slack 頻道](https://slack.cncf.io/) (#project-chaos-mesh) 或提交 PR/issue 至 [GitHub 儲存庫](https://github.com/chaos-mesh/chaos-mesh)。
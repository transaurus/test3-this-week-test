---
slug: /how-a-top-game-company-uses-chaos-engineering-to-improve-testing
title: 'How a Top Game Company Uses Chaos Engineering to Improve Testing'
authors: huizhang
image: /img/blog/fuxi-case-banner.jpg
tags: [Chaos Mesh, Chaos Engineering]
---

![頂尖遊戲公司如何運用混沌工程提升測試品質](/img/blog/fuxi-case-banner.jpg)

網易伏羲實驗室是中國首個專業遊戲 AI 研究機構。研究人員使用基於 Kubernetes 的丹爐平台進行算法開發、訓練調優和線上發布。得益於與 Kubernetes 的整合，我們的平台效率大幅提升。然而，由於 Kubernetes 和微服務相關問題，我們持續測試並改進平台以提升其穩定性。

<!--truncate-->

本文將介紹我們最寶貴的測試工具之一——[Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh)。Chaos Mesh 是一個開源的混沌工程工具，透過其儀表板提供廣泛的故障注入能力和卓越的故障監控功能。

## 為何選擇 Chaos Mesh

我們自 2018 年開始尋找混沌工程工具，期望找到具備以下特性的方案：

- **雲原生支援**：Kubernetes 已是服務編排與調度的實質標準，應用運行環境已全面標準化。對於完全運行於 K8s 的應用，配套工具必須具備雲原生支援能力。

- **充足的故障注入類型**：對於有狀態服務，網路故障模擬尤為關鍵。平台需能模擬不同層級的故障，例如 Pod、網路和 I/O 層面。

- **良好的可觀測性**：精確掌握故障注入時間與恢復時機，對我們判斷應用異常至關重要。

- **活躍的社群支援**：我們希望採用經過充分測試且持續維護的開源專案，因此重視持久且及時的社群支援。

- **對現有應用零侵入，無需領域知識**。

- **具備實際用例供評估參考**。

2019 年，當專為 Kubernetes 設計的混沌工程平台 Chaos Mesh 開源時，我們終於找到了理想工具。儘管當時仍處早期階段，但其豐富的故障類型支援立即吸引了我們。相較其他混沌工程工具，這項優勢至關重要——它直接決定了系統可發現問題的維度。我們當即意識到 Chaos Mesh 幾乎滿足了所有預期。

![Chaos Mesh 架構](/img/blog/chaos-mesh-architecture.png)

## 我們的 Chaos Mesh 實踐歷程

Chaos Mesh 協助我們發現了多個關鍵錯誤。例如，它檢測出丹爐平台開源消息隊列軟體 [rabbitMQ](https://www.rabbitmq.com/) 的腦裂問題。根據 [維基百科](https://en.wikipedia.org/wiki/Split-brain) 解釋：「腦裂狀態源於兩個獨立數據集在重疊範圍內的維護不一致，導致數據或可用性異常」。當 rabbitMQ 集群發生腦裂錯誤時，將出現數據寫入衝突或錯誤，進而引發消息服務數據不一致等更嚴重問題。如下方架構圖所示，發生腦裂時消費者無法正常運作並持續報告伺服器異常。

![RabbitMQ 集群架構](/img/blog/architecture-of-a-rabbitmq-cluster.png)

透過 Chaos Mesh，我們可穩定復現此問題——只需在容器實例雲中注入 `pod-kill` 故障即可。

Chaos Mesh 還發現了其他多項問題，包括啟動失敗、崩潰代理集群的加入失敗、心跳超時及連接通道關閉等。隨著時間推移，我們的開發團隊修復了這些問題，大幅提升了丹爐平台的穩定性。

## 快速發展的專案

Chaos Mesh 持續更新與改進。當我們最初採用時，它甚至尚未達到穩定版本。缺乏調試或日誌收集工具，且儀表板元件僅適用於 TiDB。當時唯一能使用 Chaos Mesh 測試其他應用程式的方式，是透過 `kubectl apply` 執行 YAML 設定檔。

[Chaos Mesh 1.0](https://chaos-mesh.org/blog/chaos-mesh-1.0-chaos-engineering-on-kubernetes-made-easier) 解決或改善了大部分限制。它提供更細粒度且強大的混沌支援、正式可用的 Chaos Dashboard、增強的觀測能力，以及更精準的混沌範圍控制。這些進展皆由開放、協作且充滿活力的社群驅動。

![Chaos Dashboard 現已正式可用](/img/blog/chaos-dashboard.gif)

## 未來展望

見證 Chaos Mesh 的成長與廣泛採用令人驚嘆，我們也對透過它達成的成果感到欣喜。

然而混沌工程領域仍有廣闊發展空間。未來我們期望看到以下功能：

- 原子故障注入

- 無人值守故障注入：結合自訂故障類型與標準化方法驗證實驗對象

- 通用元件標準測試案例（如 MySQL、Redis、Kafka）

我們已與 Chaos Mesh 維護者討論這些功能，確認它們已列入 Chaos Mesh 2.0 路線圖。

若您有興趣，歡迎透過 [Slack](https://slack.cncf.io/) (#project-chaos-mesh) 或 [GitHub](https://github.com/chaos-mesh/chaos-mesh) 加入 Chaos Mesh 社群。
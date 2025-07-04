---
slug: /chaos-mesh-q&a
title: 'Chaos Mesh Q&A'
authors: chaos-mesh
image: /img/blog/chaos-mesh-q&a.jpeg
tags: [Chaos Mesh, Chaos Engineering]
---

![Chaos Mesh 問答](/img/blog/chaos-mesh-q&a.jpeg)

在 KubeCon EU 2021 大會上，[Chaos Mesh](https://chaos-mesh.org/) 團隊舉辦了兩場「辦公室時間會議」，讓新成員、社群參與者和專案維護者有機會交流互動、相互認識，並深入了解此專案。

<!--truncate-->

衷心感謝超過 200 位參與者！會議期間我們收到大量優質問題，因此決定整理成這篇問答集錦。

## 您的問題解答

**提問：Chaos Mesh 是否相容於 Istio 等服務網格（Service Mesh）？**

**解答：** 是的，您可在服務網格環境中使用 Chaos Mesh。在我們[先前的社群會議](https://www.youtube.com/watch?v=paIgJYOhdGw)中，瓜地馬拉聖卡洛斯大學的 Sergio Méndez 和 Jossie Castrillo 曾分享如何運用 Linkerd 與 Chaos Mesh 為其專案「[COVID-19 即時疫苗接種視覺化系統](https://github.com/sergioarmgpl/operating-systems-usac-course/blob/master/lang/en/projects/project1v3/project1.md)」執行混沌實驗。

![專案架構圖](/img/blog/chaos-mesh-linkerd-architecture.png)

**提問：我能否在本地部署環境使用 Chaos Mesh？還是必須依賴 AWS 或 GCP 雲端服務？**

**解答：** 兩種方式皆可！Chaos Mesh 可部署於任何 Kubernetes 叢集，無論是自行管理的本地環境或 AWS/GCP 託管服務皆適用。但若要在 Kubernetes 環境運行，安裝時需[設定相關參數](https://chaos-mesh-website-archived.netlify.app/docs/1.2.4/user_guides/installation)。

**提問：「混沌操作」（chaos actions）如何運作？**

**解答：** Chaos Mesh 採用 Kubernetes 自定義資源定義（CRD）管理混沌實驗。不同故障注入行為的實作方式各異，但核心原理相同：透過應用程式的執行鏈路注入混沌。例如在網路互動鏈路注入故障時，由於 Linux 使用流量控制機制對特定網路介面卡施加干擾，我們可直接運用流量控制實現網路故障注入。

**提問：是否計劃在 Chaos Mesh 加入探針（probe）支援，用於穩態檢測與實驗驗證？**

**解答：** 目前暫無此計劃。穩態檢測與實驗驗證是應用上生產環境的必要環節。Chaos Mesh 本身不負責監控相關工作，但提供介面可整合現有監控系統或應用程式的狀態介面，藉此監測與偵測應用穩態。

**提問：Chaos Mesh 的 Pod 需要哪些進階權限？**

**A:** By default, the Chaos Daemon components in Chaos Mesh run in the `privileged` mode. If your Kubernetes cluster version is v3.11 or higher, you can replace `privileged` mode by configuring `capabilities`.

**提問：能否在構建流水線中整合 Chaos Mesh 來記錄特定測試結果？**

**解答：** 可以，這很容易實現。您可將 Chaos Mesh 與 Argo、Jenkins、GitHub Action、Spanner 等流水線系統整合。由於 Chaos Mesh 使用 CRD 管理實驗，只需在流水線中建立對應的混沌 CRD 物件即可注入故障。實驗運行狀態可透過其狀態結構和事件取得。

**提問：2.0 版本將帶來哪些更新？能否分享 HTTPChaos 的最新進展？**

**A:** Chaos Mesh 2.0 將提供原生工作流程支援，使用者可在 Chaos Mesh 中編排混沌實驗。此外，針對 Chaos Mesh 2.0，我們已重構現有的混沌控制器，讓使用者能更輕鬆新增故障注入類型。至於 HTTPChaos，我們正在為 HTTP 應用層加入網路故障模擬功能！

## 加入 Chaos Mesh 社群

若您對 Chaos Mesh 感興趣且希望協助改進，歡迎加入 [我們的 Slack 頻道](https://slack.cncf.io/) 或提交 pull request 與 issue 至 [GitHub 儲存庫](https://github.com/chaos-mesh/chaos-mesh)。
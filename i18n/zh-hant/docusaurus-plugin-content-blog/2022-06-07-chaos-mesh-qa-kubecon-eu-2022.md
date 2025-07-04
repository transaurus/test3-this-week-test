---
slug: /chaos-mesh-qa-at-kubecon-eu-2022
title: 'Chaos Mesh Q&A at KUBECON EU 2022'
authors: chaos-mesh
image: /img/blog/chaos-mesh-q&a.jpeg
tags: [Chaos Mesh, Chaos Engineering, KubeCon, CloudNativeCon]
---

![Chaos Mesh 問答集](/img/blog/chaos-mesh-q&a.jpeg)

在 KubeCon EU 2022 期間，[Chaos Mesh](https://chaos-mesh.org/) 團隊舉辦了兩場活動：「讓雲原生混沌工程更簡單 - Chaos Mesh 深度解析」與「辦公室時間會議」。我們衷心感謝所有參與者，並與大家度過了非常愉快的時光。我們彼此分享交流、深入認識，並針對多項議題展開了深度討論。

<!--truncate-->

在專題演講中，我們簡要概述了 Chaos Mesh 的基本架構，深入探討其實作原理與應用實踐，並分享團隊在混沌工程領域的最新探索成果以及 Chaos Mesh 的未來發展規劃。

在辦公室時間環節，我們介紹了 Chaos Mesh 專案及其最新進展，同時回答了線上與會者的提問。

非常感謝每位前來支持我們的朋友！在辦公室時間中，我們收到了許多優質提問，因此決定進行後續問答整理。

## 您的問題解答

**Q: Chaos Mesh 是否適用於 Windows/Linux 混合叢集？**

**A:** Chaos Mesh 目前僅支援 Linux 環境，但已有熱心貢獻者正嘗試將部分功能移植至 Windows：[github.com/chaos-mesh/chaos-mesh/issues/2956](https://github.com/chaos-mesh/chaos-mesh/issues/2956)

**Q: Istio 和 Linkerd 同樣支援故障注入。Chaos Mesh 有何不同？雖然 Chaos Mesh 提供更豐富的混沌注入類型（如 IOChaos、TimeChaos 等），但據我所知 Linkerd 或 Istio 的注入功能主要專注於網路層？**

**A:** 是的！服務網格框架主要針對 RPC/網路層進行故障注入。Chaos Mesh 除了網路層外，還可注入 stresschaos、pod kill、DNSChaos 等更多類型的混沌實驗（如前述）。此外，我們還支援 JVM、GCP、Azure 等更多平台的混沌注入。

**Q: 在 Chaos Mesh 中，我們能否在執行混沌實驗前運行預初始化腳本？**

**A:** 可以！您可透過 Chaos Mesh 的整合工作流引擎，將自訂腳本與各類混沌實驗組合執行。詳見[工作流任務欄位說明](https://chaos-mesh.org/docs/next/create-chaos-mesh-workflow/#task-field-description)文件。

**Q: 這是否類似於 Gremlin 混沌工程工具？**

**A:** 是的，這是專為 Kubernetes 設計的開源專案。您可將其作為 Kubernetes 插件使用，更多資訊請參閱 https://chaos-mesh.org

**Q: Chaos Mesh 如何實現網路延遲注入？若使用不依賴 iptables 的 Cilium CNI，延遲注入功能是否仍可運作？**

**A:** Chaos Mesh 包含 chaos-daemon 元件。當觸發網路混沌時，chaos-daemon 會進入目標 Pod 的網路命名空間，並在網路裝置上設定 TC 規則與 iptables 規則。

即使使用不依賴 iptables 的 Cilium CNI，Chaos Mesh 仍可正常運作。

## 加入 Chaos Mesh 社群

若您對 Chaos Mesh 感興趣並希望協助我們改進，歡迎加入[我們的 Slack 頻道](https://slack.cncf.io/)(#project-chaos-mesh)或提交 PR/issue 至 [GitHub 儲存庫](https://github.com/chaos-mesh/chaos-mesh)。
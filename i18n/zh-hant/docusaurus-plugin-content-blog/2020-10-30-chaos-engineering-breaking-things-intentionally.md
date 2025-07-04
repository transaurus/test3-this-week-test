---
slug: /chaos-engineering-breaking-things-intentionally
title: Chaos Engineering - Breaking things Intentionally
authors: manishdangi
image: /img/blog/chaos-engineering2.png
tags: [Chaos Engineering, Chaos Mesh, Open Source]
---

![混沌工程-故意破壞系統](/img/blog/chaos-engineering2.png)

「需求是發明之母」；同樣地，Netflix 不僅僅是線上影音串流平台。Netflix 因自身需求而催生了混沌工程。

<!--truncate-->

2008 年，Netflix [經歷了嚴重的資料庫損壞事件](https://about.netflix.com/en/news/completing-the-netflix-cloud-migration)。他們連續三天無法寄送 DVD。這促使 Netflix 工程師開始思考將單體式架構遷移至分散式的雲端架構。

Netflix 的新分散式架構由數百個微服務組成。遷移到分散式架構解決了單點故障問題，但也帶來了其他複雜性，需要更可靠且具備容錯能力的系統。此時，Netflix 工程師提出了一個創新想法：在不影響客戶服務的情況下測試系統的容錯能力。

他們開發了 [Chaos Monkey](https://github.com/Netflix/chaosmonkey)：這個工具能在不同位置隨機引發故障，並以不同時間間隔觸發。隨著 Chaos Monkey 的發展，一門新學科應運而生：混沌工程。

「混沌工程是透過在系統上進行實驗的學科，以建立對系統在生產環境中承受動盪條件能力的信心。」—— [混沌原則](https://principlesofchaos.org/)

混沌工程是透過實證探索的學科來了解系統行為的方法。正如科學家透過實驗研究物理和社會現象，混沌工程使用實驗來了解特定系統——系統的可靠性、穩定性以及在意外或不穩定條件下存活的能力。

當我們擁有大型分散式系統時，故障可能由多種因素引起，例如應用程式故障、基礎設施故障、依賴項故障、網路故障等。這些故障無法全部透過傳統方法（如整合測試或單元測試）涵蓋，這使得混沌工程成為必要手段：

- 提升系統韌性

- 揭露系統潛藏的威脅與漏洞

- 在系統弱點導致生產環境故障前發現問題

許多人認為自己的規模無法與 Netflix 等科技巨頭相比，也沒有同等規模的資料庫或系統。

他們可能沒錯，但隨著時間推移，混沌工程已發展到不再局限於 Netflix 等數位公司。為了確保系統的穩定性能和持續可用性，越來越多不同產業的公司正在實施混沌實驗。

## Chaos Mesh

:::note

2022-10-24: Because of https://www.oreilly.com/online-learning/leveraging-katacoda-technology.html, and refer to [#356](https://github.com/chaos-mesh/website/pull/356), the interactive tutorial is temporarily unavailable.

:::

為了測試 [TiDB](https://pingcap.com/products/tidb) 的韌性與可靠性，[PingCAP](https://pingcap.com/) 的工程師開發了出色的混沌測試工具 [Chaos Mesh](https://chaos-mesh.org/)。這是一個雲原生的混沌工程平台，可在 Kubernetes 環境中編排混沌實驗。Chaos Mesh 涵蓋了分散式系統可能出現的故障，包括 Pod、網路、系統 I/O 和核心層面。

Chaos Mesh 提供多種故障注入方法：

- **clock-skew:** 模擬時鐘偏移

- **container-kill:** 模擬容器被終止

- **cpu-burn:** 模擬 CPU 壓力

- **io-attribution-override:** 模擬檔案異常

- **io-fault:** 模擬檔案系統 I/O 錯誤

- **io-latency:** 模擬檔案系統 I/O 延遲

- **kernel-injection:** 模擬核心故障

- **memory-burn:** 模擬記憶體壓力

- **network-corrupt:** 模擬網路封包損壞

- **network-duplication:** 模擬網路封包重複

- **network-latency:** 模擬網路延遲

- **network-loss:** 模擬網路封包遺失

- **network-partition:** 模擬網路分割

- **pod-failure:** 模擬 Kubernetes Pod 持續不可用

- **pod-kill:** 模擬 Kubernetes Pod 被終止

Chaos Mesh 主要著重於簡化所有混沌測試的執行方式，使其能快速完成且易於使用者理解。

近期發佈的 [1.0 版本](https://chaos-mesh.org/blog/chaos-mesh-1.0-chaos-engineering-on-kubernetes-made-easier/) 正式推出 Chaos Dashboard，大幅簡化混沌實驗的複雜性。只需點擊幾下滑鼠，即可在 Chaos Mesh 儀表板中完成：定義實驗範圍、指定混沌注入類型、設定排程規則，並觀察實驗結果。

In case you want to try Chaos Mesh in your browser, checkout `Katakoda interactive tutorial`, where you can get your hands on Chaos Mesh without even deploying it. To understand the design principles and how Chaos Mesh works, read [this blog](https://chaos-mesh.org/blog/chaos_mesh_your_chaos_engineering_solution) by the project's maintainer, [Cwen Yin](https://www.linkedin.com/in/cwen-yin-81985318b/).

## 加入社群

歡迎所有想探索混沌工程或 Chaos Mesh 的開發者加入社群。作為社群成員，我必須說這是個充滿活力的社群：專案維護者樂於交流，並重視您對專案與社群的改進建議。

欲加入社群或了解更多資訊，請至 [CNCF Slack 工作區](https://slack.cncf.io/) 的 #project-chaos-mesh 頻道。
---
slug: /celebrating-one-year-of-chaos-mesh-looking-back-and-ahead
title: 'Celebrating One Year of Chaos Mesh: Looking Back and Ahead'
authors: chaos-mesh
image: /img/blog/celebrating-one-year-of-chaos-mesh-looking-back-and-ahead.jpg
tags: [Chaos Mesh, Chaos Engineering]
---

![慶祝 Chaos Mesh 一週年：回顧與展望](/img/blog/celebrating-one-year-of-chaos-mesh-looking-back-and-ahead.jpg)

距離 Chaos Mesh 首次在 GitHub 上開源已有一年時間。Chaos Mesh 從單純的故障注入工具起步，如今正朝著建立混沌工程生態的目標邁進。與此同時，Chaos Mesh 社群也從零開始建立，並協助 [Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh) 以 Sandbox 專案身份加入 CNCF。

<!--truncate-->

本文將與您分享 Chaos Mesh 過去一年的成長與變化，並探討其未來的目標與規劃。

## 專案：目標明確，蓬勃發展

過去一年在社群共同努力下，Chaos Mesh 以驚人速度成長。從最初版本到近期發布的 [v1.1.0](https://github.com/chaos-mesh/chaos-mesh/releases/tag/v1.1.0)，無論在功能、易用性還是安全性方面，Chaos Mesh 都獲得了顯著提升。

### 功能

開源初期，Chaos Mesh 僅支援三種故障類型：PodChaos、NetworkChaos 和 IOChaos。短短一年內，它已能對網路、系統時鐘、JVM 應用、檔案系統、作業系統等進行全方位故障注入。

![混沌測試](/img/blog/chaos-tests.png)

經過持續優化，Chaos Mesh 現提供靈活的調度機制，讓使用者能更完善地設計混沌實驗，為混沌編排奠定基礎。

我們欣喜看到許多使用者開始在[主要雲端平台測試 Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh/issues/1182)，如 Amazon Web Services (AWS)、Google Kubernetes Engine (GKE)、阿里雲和騰訊雲。我們持續進行相容性測試與適配，以支援[特定雲平台的故障注入](https://github.com/chaos-mesh/chaos-mesh/pull/1330)。

為更好支援 Kubernetes 原生元件及節點層級故障，我們開發了 [Chaosd](https://github.com/chaos-mesh/chaosd)，提供實體節點層級的故障注入功能，目前正進行全面測試與精煉，預計未來數月內發布。

### 易用性

易用性始終是 Chaos Mesh 開發的核心原則。您只需一行指令即可部署 Chaos Mesh。v1.0 版本推出期盼已久的 Chaos Dashboard，提供一站式 Web 介面讓使用者編排混沌實驗。您可在同個介面中定義實驗範圍、指定故障注入類型、設定調度規則並觀察實驗結果，僅需點擊幾下即可完成。

![混沌儀表板](/img/blog/chaos-dashboard1.png)

在 v1.0 之前，許多使用者回報注入 IOChaos 故障時遭遇各種配置問題。經深入調查與討論，我們放棄原有的 SideCar 實現方式，改用 chaos-daemon 動態侵入目標 Pod，大幅簡化邏輯。這項優化使 Chaos Mesh 實現動態 I/O 故障注入，使用者能專注實驗本身，無需擔憂額外配置。

### 安全性

我們已強化 Chaos Mesh 的安全性。它現在提供完整的選擇器套件來控制實驗範圍，並支援設定特定命名空間以保護關鍵應用程式。此外，命名空間權限支援讓使用者能將混沌實驗的「爆炸半徑」限制在特定命名空間內。

更重要的是，Chaos Mesh 直接重用 Kubernetes 原生權限機制，並在混沌儀表板上支援驗證功能。這能保護您免受其他使用者錯誤操作的影響，避免導致混沌實驗失敗或失控。

## 雲原生生態系統：整合與合作

2020年7月，Chaos Mesh成功[獲接納為CNCF沙盒專案](https://chaos-mesh.org/blog/chaos-mesh-join-cncf-sandbox-project)。這表明Chaos Mesh已獲得雲原生社群的初步認可，同時意味著Chaos Mesh肩負明確使命：推動混亂工程在雲原生領域的應用，並與其他雲原生專案協同發展，實現共同成長。

### Grafana

為進一步提升混亂實驗的可觀測性，我們為Chaos Mesh開發了專屬的[Grafana外掛](https://github.com/chaos-mesh/chaos-mesh-datasource)，讓使用者能直接在應用監控面板上顯示即時混亂實驗資訊。如此一來，使用者可同步觀察應用運行狀態與當前混亂實驗動態。

### GitHub Action

為使開發階段即可執行混亂實驗，我們開發了[chaos-mesh-action](https://github.com/chaos-mesh/chaos-mesh-action)專案，使Chaos Mesh能在GitHub Actions工作流中運行。這讓Chaos Mesh能無縫整合至日常系統開發與測試流程。

### TiPocket

[TiPocket](https://github.com/pingcap/tipocket)是整合Chaos Mesh與Kubernetes工作流程引擎Argo的自動化測試平台，專為分散式資料庫TiDB設計的完整混亂工程測試迴圈。執行混亂實驗包含多步驟：部署應用、運行負載、注入異常及業務驗證。為實現全自動化，TiPocket整合Argo提供靈活編排調度，而Chaos Mesh則提供豐富的故障注入能力。

![TiPocket](/img/blog/tipocket.png)

## 社群建設：從零起步

Chaos Mesh是社群驅動專案，離不開活躍、友善且開放的社群環境。開源一年內，它已成為混亂工程領域最引人注目的開源專案之一，累積逾3k GitHub星標與70+貢獻者。採用者包含騰訊雲、小鵬汽車、Dailymotion、網易伏羲實驗室、JuiceFS、APISIX及美團等。回顧這一年，Chaos Mesh社群從零建構，為透明、開放、友善且自治的開源社群奠定基礎。

### 加入CNCF大家庭

雲原生基因深植於Chaos Mesh的發展歷程，加入CNCF是必然選擇，這標誌著專案邁向供應商中立、開放透明的關鍵一步。除生態整合外，加入CNCF為Chaos Mesh帶來：

- 更高社群與專案曝光度：透過與其他專案合作及參與Kubernetes Meetup、KubeCon等活動，我們獲得絕佳社群交流機會。社群產出的高品質內容對推廣Chaos Mesh產生深遠積極影響。

- 更完善開放的社群框架：CNCF提供成熟的開源社群運作框架。在其指導下，我們建立基礎社群架構，包含行為準則、貢獻指南及路線圖，並在CNCF Slack創建專屬頻道#project-chaos-mesh。

### 友善支持的社群環境

:::note

2022-10-24: Because of https://www.oreilly.com/online-learning/leveraging-katacoda-technology.html, and refer to [#356](https://github.com/chaos-mesh/website/pull/356), the interactive tutorial is temporarily unavailable.

:::

開源社群的品質決定了採用者和貢獻者是否願意長期參與社群發展。在這方面，我們持續努力的方向包括：

- 持續豐富文件並優化結構。目前我們已針對不同用戶群體開發完整文件體系，涵蓋[使用指南](https://chaos-mesh-website-archived.netlify.app/docs/1.2.4/user_guides/installation)、[開發者指南](https://chaos-mesh-website-archived.netlify.app/docs/1.2.4/development_guides/development_overview)、[快速入門指引](https://chaos-mesh-website-archived.netlify.app/docs/1.2.4/get_started/get_started_on_kind)、[應用案例](https://chaos-mesh-website-archived.netlify.app/docs/1.2.4/use_cases/multi_data_centers)及[貢獻指南](https://github.com/chaos-mesh/chaos-mesh/blob/master/CONTRIBUTING.md)，所有內容均隨版本持續更新。

- Working with the community to publish blog posts, tutorials, use cases, and chaos engineering practices. So far, we’ve produced 26 Chaos Mesh related articles. Among them is `an interactive tutorial`, published on O’Reilly’s Katakoda site. These materials make a great complement to the documentation.

- 將社群會議、線上研討會及技術交流活動產出的影片與教學資源重新整合運用，重視並回應社群反饋與疑問。

## 未來展望

Google近期全球服務中斷事件再次警示系統可靠性的重要性，同時突顯混沌工程的關鍵價值。CNCF技術監督委員會主席Liz Rice提出[2021年五大值得關注技術](https://twitter.com/CloudNativeFdn/status/1329863326428499971)，混沌工程位居榜首。我們大膽預測混沌工程即將邁入新階段，目前Chaos Mesh 2.0正積極開發中，重點包含社群需求功能：嵌入式工作流引擎以支援更靈活的混沌場景定義與管理、應用狀態檢查機制，以及更詳盡的實驗報告。可透過專案[路線圖](https://github.com/chaos-mesh/chaos-mesh/blob/master/ROADMAP.md)持續關注進展。

## 結語

Chaos Mesh過去一年快速成長，但仍是年輕專案，航行目標才剛啟程。在此誠摯邀請您參與共建混沌工程生態體系！

若對Chaos Mesh感興趣並願意協助改進，歡迎加入[Slack頻道](https://slack.cncf.io/)或至[GitHub代碼庫](https://github.com/chaos-mesh/chaos-mesh)提交拉取請求與議題。
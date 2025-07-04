---
slug: /experience-as-a-chaos-mesh-lfx-mentee
title: 'Experience as an LFX Mentee for Chaos Mesh'
authors: chunxuzhang
image: /img/blog/lfx-mentee-experience-banner.png
tags: [Chaos Mesh, Chaos Engineering, LFX Mentorship, Monitoring Metrics]
---

![Chaos Mesh LFX 學員經驗分享](/img/blog/lfx-mentee-experience-banner.png)

我目前是南京大學軟體工程研究所的研究生，研究領域聚焦於 DevOps，這與混沌工程和可觀測性有著本質上的關聯。為了參與開源社群、深入理解 Kubernetes 並體驗基礎設施領域的日常工作，我申請了 2021 年秋季的 CNCF LFX 導師計畫，參與 [Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh) 專案的開發。

<!--truncate-->

## 申請流程

八月底結束商業性質的實習後，我確認自己對業務導向的工作興趣不大，但始終對基礎設施技術懷抱熱忱。偶然發現 CNCF LFX 導師計畫中的 Chaos Mesh 專案後，我認為這是實現參與開源專案夢想的絕佳機會。由於技術棧吻合，我在截止前遞交了申請。

三天後收到導師的面試邀請，其中包含一項小任務：編寫能暴露 Prometheus 指標的 mini-node-exporter，並在 Grafana 儀表板呈現數據。同時需將該組件與配置好的 Prometheus、Grafana 部署至 Kubernetes 平台。開發過程相當順利，唯一挑戰是將 Grafana 儀表板轉寫為 Kubernetes 部署用的 YAML 配置檔。經過文件查閱與實驗驗證後，最終成功解決。

8 月 30 日幸運獲知面試通過。與導師一對一會議中，我們簡要討論了我對 Kubernetes 等技術的熟悉度、主要任務與關鍵時程。我也提出研究生實驗室專案可能影響進度、指標設計規範等顧慮，導師充分理解並給予支持。

## 專案執行

我申請的專案名為 [Chaos Mesh 監控指標](https://mentorship.lfx.linuxfoundation.org/project/8db683b0-0273-4a83-9ed9-4c33ee2cfcf0)，目標是透過收集指標並提供 Grafana 儀表板來提升 Chaos Mesh 系統的可觀測性。

專案前兩週熟悉 Chaos Mesh 業務流程與程式碼細節，隨後兩週開始撰寫設計文件梳理所有指標及採集方式。期間研讀指標設計規範，並與導師會議釐清提案細節與部分程式邏輯。

多數指標採集相對簡單，僅需查詢資料庫物件、k8s 物件或進行基礎計數。但部分特殊指標採集難度較高，例如：需在對應容器的網路命名空間執行指令查詢資料、透過三種不同容器運行時查詢 daemon 下所有容器，或收集 gRPC 客戶端與伺服器間的通訊數據。

這些任務對我而言相當陌生，因此不時需要導師的技術支援，而導師總是及時回應。導師在該領域的深厚知識與經驗令我印象深刻。在指導下，最終完成設計的 [RFC](https://github.com/chaos-mesh/rfcs/pull/23) 文件。後續為追蹤進度，建立了[專案追蹤 issue](https://github.com/chaos-mesh/chaos-mesh/issues/2397)。

![專案追蹤 issue](/img/blog/lfx-mentee-experience-tracking-issue.png)

然而後續編碼過程中仍遇到諸多問題，回顧發現許多可提前避免。以下總結幾點建議：

**保持批判性思考**。當我接受提案時，憑直覺提出了每個指標的解決方案，卻忽略了一些基本問題：這些指標是否必要？我們是否有更好的解決方案可用？這些基礎問題本應在提案階段解決，卻被延續到後續的設計實施階段。例如提交 RFC 時，導師和審閱者提醒我部分指標已在 controller-runtime 函式庫中實現；處理 BPM 相關指標時，審閱者同樣提出質疑。直到那時我才意識到自己從未關注這點。

![關於 BPM 指標的評論](/img/blog/lfx-mentee-experience-thinking-critically.png)

**持續溝通**。如何有效溝通是本次導師計畫的重要課題。我學到許多溝通經驗，最深刻的體悟是：尋求建議前應先提供選項。當需要求助時，提供參考選項給對方。儘管這些選項可能不成立，但包含你的思考脈絡。因此除非徹底思考後仍無頭緒，否則別讓他人在你的問題中迷失方向。

**理解開源文化**。這是我首次實際參與開源專案，與企業工作模式大相逕庭。以下具體說明：

1. **資訊同步方式**。企業工作常透過面對面會議溝通，開源社群則集中於 Slack 頻道、GitHub 議題與拉取請求。因此必須完整記錄工作進展，讓社群成員隨時掌握狀況。前幾週我沿用習慣維護線上研發文件，後發現改用 GitHub 看板或議題更合適，避免因使用不同平台增加導師溝通成本。

   ![線上研發文件](/img/blog/lfx-mentee-experience-rd-doc.png)

2. **更嚴謹的自動化測試**。商業公司的自動化測試通常僅含靜態程式碼分析、單元測試與基礎冒煙測試，主要倚重人工測試。但開源專案的自動化流程包含更完整的測試案例，如整合測試、端到端測試、授權檢查等，初步把關程式碼品質與安全性。

3. **程式碼審查機制**。多人將參與你的程式碼審查，過程可能持續較久。與企業專職審查者不同，開源社群審查者可能是使用者、維護者或其他自願參與的成員，這種開放性可能是審查週期較長的部分原因。

## 專案結束後

這 12 週是段美妙經歷。我深入理解了 Kubernetes、CRD 與可觀測性，也意識到在程式碼架構優化、Linux 基礎與容器技術方面仍有不足，學習之路漫長！

同時因研究所專案突增壓力，我未能投入充足時間於導師計畫，甚至未能在時限內完成 Grafana 儀表板設計。我必將持續跟進，期望圓滿完成此專案，為這段旅程畫下句點。

衷心感謝導師 [@STRRL](https://github.com/STRRL)。實習期間遭遇諸多技術挑戰：Git 操作、循環依賴解決方案、CRI-O 執行時介面查詢等。若無導師耐心指導，我難以克服這些陌生技術難題。同時感謝 Chaos Mesh 維護者審閱程式碼，以及 CNCF LFX 導師計畫為開源社群參與者提供的卓越平台。

![導師的 LGTM 認可](/img/blog/lfx-mentee-experience-mentors-lgtm.png)

最後，我希望每位想要參與開源社群的學生都能透過 LFX Mentorship 踏出第一步！
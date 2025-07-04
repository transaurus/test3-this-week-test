---
slug: /develop-a-daily-reporting-system
title: 'How to Develop a Daily Reporting System to Track Chaos Testing Results'
authors: leili
image: /img/blog/chaos-mesh-digitalchina-banner.png
tags: [Chaos Mesh, Chaos Engineering, Use Cases]
---

![如何開發日常報告系統追蹤混沌測試結果](/img/blog/chaos-mesh-digitalchina-banner.png)

Chaos Mesh 是一個雲原生的混沌工程平台，可在 Kubernetes 環境中編排混沌實驗。它讓您能透過模擬網路故障、檔案系統故障和 Pod 故障等問題來測試系統韌性。每次混沌實驗後，您可以檢查日誌來審查測試結果，但這種方式既不直觀也不高效。因此，我決定開發一個日常報告系統，自動分析日誌並生成報告，從而輕鬆檢視日誌並識別問題。

<!--truncate-->

本文將分享關於如何建構日常報告系統的見解，以及我在過程中遇到的問題與解決方案。

## 在 Kubernetes 上部署 Chaos Mesh

Chaos Mesh 專為 Kubernetes 設計，這正是它能讓使用者針對特定應用注入檔案系統、Pod 或網路故障的重要原因。

早期文件中，Chaos Mesh 提供兩種在本機快速部署虛擬 Kubernetes 叢集的方式：[kind](https://github.com/kubernetes-sigs/kind) 和 [minikube](https://minikube.sigs.k8s.io/docs/start/)。通常只需一行指令就能部署 Kubernetes 叢集並安裝 Chaos Mesh。但存在以下問題：

- 在本機啟動 Kubernetes 叢集會影響網路相關的故障類型。

- 中國大陸使用者可能會遇到 Docker 映像拉取極慢甚至超時的情況。

若使用提供的腳本透過 kind 部署 Kubernetes 叢集，所有節點都是虛擬機器 (VM)，這會增加離線拉取映像的難度。為解決此問題，您可改在多台實體機器上部署 Kubernetes 叢集，每台實體機器作為工作節點。要加速映像拉取過程，可預先使用 `docker load` 指令載入所需映像。除上述兩個問題外，您可遵循文件安裝 [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) 和 [Helm](https://helm.sh/)。

注意：最新安裝部署說明請參閱 [Chaos Mesh 快速入門](https://chaos-mesh.org/docs/quick-start/)。

## 部署 TiDB

下一步是在 Kubernetes 上部署 TiDB。我使用 TiDB Operator 簡化流程，詳細步驟請參閱 [在 Kubernetes 中開始使用 TiDB Operator](https://docs.pingcap.com/tidb-in-kubernetes/stable/get-started)。

在此過程中，我想強調兩點：

- 首先安裝自訂資源定義 (CRDs) 來實現 TiDB Operator 的各個元件，否則安裝 TiDB Operator 時會出錯。

- 使用 [Longhorn](https://longhorn.io/)（Kubernetes 的分散式區塊儲存系統）為 Kubernetes 叢集建立本地持久卷 (PV)。如此便無需預先建立 PV：每當 Pod 被調度時，PV 會自動建立並掛載。

我遇到的最大問題是部署服務時拉取映像可能極慢。若 Kubernetes 叢集中的節點是虛擬機器，請預先拉取所需映像並載入到每台機器的 Docker：

```bash
## Pull required images on a machine with a good network connection
docker pull pingcap/tikv:latest
docker pull pingcap/tidb:latest
docker pull pingcap/pd:latest

## Export images and save them to each machine in the Kubernetes cluster
docker save -o tikv.tar pingcap/tikv:latest
docker save -o tidb.tar pingcap/tidb:latest
docker save -o pd.tar pingcap/pd:latest

## Load images to each machine
docker load &lt; tikv.tar
docker load &lt; tidb.tar
docker load &lt; pd.tar
```

上述指令讓您能使用本地 Docker 倉庫中的 TiDB 映像來部署最新 TiDB 叢集，省去從遠端倉庫拉取映像的麻煩。此方法同樣適用於前述的 Chaos Mesh 安裝。若不清楚需要拉取哪些映像，可透過 Helm 安裝 Chaos Mesh 來觸發安裝流程，再用 `kubectl describe` 指令驗證：

```bash
## Check pods that are deployed in a specific namespace.
kubectl describe pods -n tidb-test
```

映像拉取過程通常耗時最長。若 Pod 正在被調度到節點，請稍後再檢查。

## 執行混沌實驗

要執行混沌實驗，必須先透過 YAML 檔案定義實驗內容，再使用 `kubectl apply` 指令啟動。在此範例中，我建立了一個 PodChaos 混沌實驗來模擬 Pod 崩潰。詳細操作指引請參閱[執行混沌實驗](https://chaos-mesh.org/docs/run-a-chaos-experiment/)。

## 生成日報系統

### 收集日誌

在 TiDB 叢集上執行混沌實驗時，通常會產生大量錯誤訊息。要收集這些錯誤日誌，可執行 `kubectl logs` 指令：

```bash
kubectl logs &lt;podname> -n tidb-test --since=24h >> tidb.log
```

`tidb-test` 命名空間中特定 Pod 過去 24 小時內產生的所有日誌，將被保存至 `tidb.log` 檔案。

### 過濾錯誤與警告

此步驟需從日誌中過濾出錯誤訊息與警告訊息，有兩種方式可實現：

- 使用文字處理工具（如 awk），需精通 Linux/Unix 指令

- 撰寫腳本程式，若不熟悉 Linux/Unix 指令，此為更佳選擇

### 繪製圖表

繪圖部分採用 [gnuplot](http://www.gnuplot.info/) 命令列繪圖工具。下方範例導入壓力測試結果，繪製折線圖展示特定 Pod 失效時每秒查詢率（QPS）的變化。由於混沌實驗定期執行，QPS 數量呈現規律波動：驟降後迅速恢復正常。

![QPS 折線圖](/img/blog/qps-line-graph.png)

### 生成 PDF 報告

目前尚無現成 API 可用於生成 Chaos Mesh 報告或分析結果。我選擇以 PDF 格式生成報告，確保跨瀏覽器可讀性。實作採用 [gopdf](https://github.com/signintech/gopdf) 函式庫建立 PDF 檔案，支援插入圖片與繪製表格，符合需求。

為自動生成日報，使用 [crond](https://www.linux.org/docs/man8/cron.html) 背景定時任務工具，每日清晨執行指令。如此在工作開始時，即可取得現成的日報。

## 建置日報系統網頁應用

為提升報告可讀性與存取便利性，網頁應用是更理想的查閱方式。起初考慮建立後端 API 追蹤報告生成時間，但考量僅需識別待排查報告（如檔案名 `report-2021-07-09-bad.pdf` 已含關鍵資訊），此方案過於繁複。簡化設計可大幅降低系統負載與複雜度。

後續仍需強化後端介面並豐富報告內容，但現階段以建置可運作的日報系統為優先。

實作採用 [Vue.js](https://github.com/vuejs/vue) 架構網頁應用，搭配 [antd](https://www.antdv.com/docs/vue/introduce/) UI 元件庫。將自動生成的報告儲存至靜態資源目錄 `static`，網頁應用即可讀取靜態報告並渲染至前端頁面。詳細步驟參閱[在 vue-cli 3 使用 antd](https://www.antdv.com/docs/vue/use-with-vue-cli/)。

下圖為日報系統網頁應用範例。紅色卡片標示需檢視測試報告，因執行混沌實驗後出現異常。

![日報系統網頁應用](/img/blog/web-app-for-daily-reporting.png)

點擊紅色卡片將會開啟報告，如下所示。我使用 [pdf.js](https://github.com/mozilla/pdf.js) 來查看 PDF。

![每日報告 PDF](/img/blog/daily-report-pdf.png)

## 總結

Chaos Mesh 讓您能夠模擬大多數雲原生應用程式可能遇到的故障。在本文中，我建立了一個 PodChaos 實驗，並觀察到當 Pod 不可用時，TiDB 叢集中的 QPS 受到了影響。分析日誌後，我可以增強系統的健壯性和高可用性。我建立了一個網頁應用程式來生成每日報告以進行故障排除和除錯。您也可以自訂報告以滿足自己的需求。

我們的團隊也在進行一個專案，旨在[讓 TiDB 相容於 PostgreSQL](https://github.com/DigitalChinaOpenSource/TiDB-for-PostgreSQL)。如果您有興趣並想貢獻，歡迎挑選一個議題並開始參與。

**原文發佈於 _[The New Stack](https://thenewstack.io/develop-a-daily-reporting-system-for-chaos-mesh-to-improve-system-resilience/)_。**
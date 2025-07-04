---
slug: /chaos_mesh_your_chaos_engineering_solution
title: Chaos Mesh - Your Chaos Engineering Solution for System Resiliency on Kubernetes
authors: cwen
image: /img/blog/chaos-engineering.png
tags: [Chaos Mesh, Chaos Engineering, Kubernetes]
---

![混沌工程](/img/blog/chaos-engineering.png)

## 為什麼選擇 Chaos Mesh？

在分散式運算領域，故障可能隨時隨地不可預測地發生在您的集群中。傳統上我們透過單元測試和整合測試確保系統具備生產環境準備度，但隨著集群規模擴大、複雜性增加以及資料量攀升至 PB 等級，這些測試僅涵蓋冰山一角。為了更有效識別系統脆弱性並提升韌性，Netflix 開發了 [Chaos Monkey](https://netflix.github.io/chaosmonkey/)，將各類故障注入基礎設施和業務系統，混沌工程由此誕生。

<!--truncate-->

在 [PingCAP](https://chaos-mesh.org/) 構建開源分散式 NewSQL 資料庫 [TiDB](https://github.com/pingcap/tidb) 時，我們面臨同樣挑戰。對資料庫而言，韌性至關重要——因為用戶最重要的資產「資料」正處於風險中。為確保韌性，我們[在測試框架中實踐混沌工程](https://pingcap.com/blog/chaos-practice-in-tidb/)。隨著 TiDB 發展，測試需求日益增長，我們意識到需要一個通用混沌測試平台，不僅服務 TiDB，也適用於其他分散式系統。

因此我們推出 Chaos Mesh——專為 Kubernetes 環境設計的雲原生混沌工程平台，可於 [https://github.com/chaos-mesh/chaos-mesh](https://github.com/chaos-mesh/chaos-mesh) 開源取得。

以下我將分享 Chaos Mesh 的核心概念、設計實現原理，並展示如何在您的環境中使用。

## Chaos Mesh 能做什麼？

Chaos Mesh 是功能全面的混沌工程平台，提供涵蓋 Kubernetes 上複雜系統的全方位故障注入方法，包括 Pod、網路、檔案系統乃至核心層級的故障模擬。

此案例展示我們如何透過 Chaos Mesh 定位 TiDB 系統錯誤：模擬分散式儲存引擎 ([TiKV](https://docs.pingcap.com/tidb/stable/tidb-architecture#tikv-server)) 的 Pod 停機，並觀察每秒查詢率 (QPS) 變化。正常情況下，當一個 TiKV 節點停機時，QPS 會經歷短暫波動後恢復至故障前水平，此即高可用性的保障機制。

![Chaos Mesh 發現 TiKV 停機恢復異常](/img/blog/chaos-mesh-discovers-downtime-recovery-exceptions-in-tikv.png)

儀表板顯示：

- 前兩次停機後，QPS 約 1 分鐘恢復正常

- 第三次停機後，QPS 恢復耗時約 9 分鐘。此異常延遲將嚴重影響線上服務

經診斷發現，受測 TiDB 集群版本 (V3.0.1) 處理 TiKV 停機時存在棘手問題，後續版本已修復此缺陷。

Chaos Mesh 的功能遠不止模擬停機，還包含以下故障注入方法：

- **pod-kill:** 模擬 Kubernetes Pod 被終止

- **pod-failure:** 模擬 Kubernetes Pod 持續不可用

- **network-delay:** 模擬網路延遲

- **network-loss:** 模擬網路封包遺失

- **network-duplication:** 模擬網路封包重複

- **network-corrupt:** 模擬網路封包損壞

- **network-partition:** 模擬網路分區

- **I/O 延遲:** 模擬檔案系統的 I/O 延遲

- **I/O 錯誤碼:** 模擬檔案系統的 I/O 錯誤

## 設計原則

我們將 Chaos Mesh 設計為易於使用、具備擴展性，並專為 Kubernetes 打造。

### 易於使用

為實現易用性，Chaos Mesh 必須：

- 無需特殊依賴，可直接部署於 Kubernetes 叢集（包括 [Minikube](https://github.com/kubernetes/minikube)）。

- 無需修改受測系統（SUT）的部署邏輯，即可在生產環境執行混沌實驗。

- 輕鬆編排混沌實驗中的故障注入行為，便捷查看實驗狀態與結果，並能快速回滾注入的故障。

- 隱藏底層實作細節，讓使用者專注於混沌實驗編排。

### 擴展性

Chaos Mesh 應具備擴展性，以便靈活"插入"新需求而無需重複造輪子。具體而言：

- 複用現有實作，使故障注入方法可輕鬆擴展。

- 易於與其他測試框架整合。

### 專為 Kubernetes 設計

在容器領域，Kubernetes 已成為絕對領導者，其採用增速遠超預期並贏得容器編排之爭。本質上，Kubernetes 是雲端的作業系統。

TiDB 作為雲原生分散式資料庫，其內部自動化測試平台自始建於 Kubernetes。我們每日有數百個 TiDB 叢集在 Kubernetes 上運行，執行包含大規模混沌測試在內的各類實驗，以模擬生產環境故障。為支援此類實驗，混沌工程與 Kubernetes 的結合成為我們實作的自然選擇與核心原則。

## CustomResourceDefinitions 設計

Chaos Mesh 採用 [CustomResourceDefinitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)（CRD）定義混沌物件。在 Kubernetes 生態中，CRD 是成熟的客製資源實作方案，具備豐富案例與工具鏈，使 Chaos Mesh 能無縫整合至 Kubernetes 生態系。

我們未將所有故障注入類型統一於單個 CRD 物件，而是為不同故障類型設計靈活獨立的 CRD 物件。若新增故障注入方法符合現有 CRD 物件，則基於該物件直接擴展；若為全新方法，則為其創建新 CRD 物件。此設計將混沌物件定義與邏輯實作從頂層解耦，使程式結構更清晰，降低耦合度與錯誤率。此外，Kubernetes 的 [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) 提供優秀的控制器封裝，省去為每個 CRD 專案重複實作控制器的成本。

Chaos Mesh 實作了 PodChaos、NetworkChaos 與 IOChaos 物件，其名稱明確標示對應的故障注入類型。

例如，Pod 崩潰是 Kubernetes 環境的常見問題。許多原生資源物件會透過重建 Pod 等操作自動處理此類錯誤，但我們的應用能否真正應對此類故障？若 Pod 無法啟動又該如何？

透過明確定義如 `pod-kill` 等操作，PodChaos 能協助精準定位此類問題。PodChaos 物件使用以下程式碼：

```yml
spec:
 action: pod-kill
 mode: one
 selector:
   namespaces:
     - tidb-cluster-demo
   labelSelectors:
     "app.kubernetes.io/component": "tikv"
  scheduler:
   cron: "@every 2m"
```

此程式碼執行以下操作：

- `action` 屬性定義需注入的具體錯誤類型。本例中，`pod-kill` 將隨機殺死 Pod。

- `selector` 屬性限制了混沌實驗的範圍至特定區域。此處的範圍是 `tidb-cluster-demo` 命名空間中 TiDB 叢集的 TiKV Pods。

- `scheduler` 屬性定義了每次混沌故障動作的觸發間隔。

有關 NetworkChaos 和 IOChaos 等 CRD 物件的詳細說明，請參閱 [Chaos-mesh 文件](https://github.com/chaos-mesh/chaos-mesh)。

## Chaos Mesh 如何運作？

確立 CRD 設計後，我們來綜觀 Chaos Mesh 的運作機制。主要包含以下核心元件：

- **controller-manager**

  作為平台的「大腦」，管理 CRD 物件的生命週期並調度混沌實驗。其物件控制器負責調度 CRD 物件實例，[admission-webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) 控制器則動態注入邊車容器至 Pods。

- **chaos-daemon**

  以特權 DaemonSet 形式運行，可操作節點的網路設備及 Cgroup。

- **sidecar**

  作為特殊容器類型，由 admission-webhooks 動態注入目標 Pod。例如 `chaosfs` 邊車容器運行 fuse-daemon 來劫持應用容器的 I/O 操作。

![Chaos Mesh 工作流程](/img/blog/chaos-mesh-workflow.png)

這些元件協作執行混沌實驗的流程如下：

1. 使用者透過 YAML 檔案或 Kubernetes 用戶端，建立或更新混沌物件至 Kubernetes API 伺服器。

2. Chaos Mesh 監控 API 伺服器的混沌物件，透過建立/更新/刪除事件管理實驗生命週期。此過程中 controller-manager、chaos-daemon 和邊車容器協同注入故障。

3. 當 admission-webhooks 收到 Pod 建立請求時，動態更新即將建立的 Pod 物件（例如注入邊車容器）。

## 執行混沌實驗

前文已說明 Chaos Mesh 的設計與運作原理，現在我們將實際演示操作方式。請注意，測試時間會因應用複雜度及 CRD 中定義的調度規則而異。

### 準備環境

Chaos Mesh 需運行於 Kubernetes v1.12 以上版本，並透過 Helm（Kubernetes 套件管理工具）部署管理。執行前請確保 Helm 已正確安裝於叢集中。環境設置步驟如下：

1. 確認已擁有 Kubernetes 叢集。若已具備則跳至步驟 2；否則使用 Chaos Mesh 提供的腳本在本機啟動叢集：

   ```bash
   // install kind
   curl -Lo ./kind https://github.com/kubernetes-sigs/kind/releases/download/v0.6.1/kind-$(uname)-amd64
   chmod +x ./kind
   mv ./kind /some-dir-in-your-PATH/kind

   // get script
   git clone https://github.com/chaos-mesh/chaos-mesh
   cd chaos-mesh
   // start cluster
   hack/kind-cluster-build.sh
   ```

   **注意：** 本機啟動 Kubernetes 叢集會影響網路相關故障注入。

2. 若 Kubernetes 集群已就緒，請使用 [Helm](https://helm.sh/) 和 [Kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) 部署 Chaos Mesh：

   ```bash
   git clone https://github.com/chaos-mesh/chaos-mesh.git
   cd chaos-mesh
   // 建立 CRD 資源
   kubectl apply -f manifests/
   // 安裝 chaos-mesh
   helm install helm/chaos-mesh --name=chaos-mesh --namespace=chaos-mesh
   ```

   等待所有元件安裝完成，並使用以下指令檢查安裝狀態：

   ```bash
   // 檢查 chaos-mesh 狀態
   kubectl get pods --namespace chaos-mesh -l app.kubernetes.io/instance=chaos-mesh
   ```

   若安裝成功，您將看到所有 Pod 都在運行中。現在，是時候開始了。

   您可以使用 YAML 定義或 Kubernetes API 來運行 Chaos Mesh。

### 使用 YAML 檔案運行混沌實驗

您可以通過 YAML 檔案方法定義自己的混沌實驗，這提供了一種在部署應用後快速、方便地進行混沌實驗的方式。要使用 YAML 檔案運行混沌實驗，請按照以下步驟操作：

**注意：** 為了說明，我們使用 TiDB 作為測試系統。您可以使用自己選擇的目標系統，並相應地修改 YAML 檔案。

1. 部署一個名為 `chaos-demo-1` 的 TiDB 集群。您可以使用 [TiDB Operator](https://github.com/pingcap/tidb-operator) 來部署 TiDB。

2. 建立一個名為 `kill-tikv.yaml` 的 YAML 檔案，並加入以下內容：

   ```yml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: PodChaos
   metadata:
     name: pod-kill-chaos-demo
     namespace: chaos-mesh
   spec:
     action: pod-kill
     mode: one
     selector:
       namespaces:
         - chaos-demo-1
       labelSelectors:
         'app.kubernetes.io/component': 'tikv'
     scheduler:
       cron: '@every 1m'
   ```

3. 儲存檔案。

4. 要啟動混沌實驗，執行 `kubectl apply -f kill-tikv.yaml`。

以下混沌實驗模擬了在 `chaos-demo-1` 集群中頻繁殺死 TiKV Pods 的情況：

![混沌實驗運行中](/img/blog/chaos-experiment-running.gif)

我們使用 sysbench 程式監控 TiDB 集群的即時 QPS 變化。當錯誤被注入集群時，QPS 會出現劇烈的抖動，這表示某個特定的 TiKV Pod 已被刪除，而 Kubernetes 隨後重新建立了一個新的 TiKV Pod。

For more YAML file examples, see https://github.com/chaos-mesh/chaos-mesh/tree/master/examples.

### 使用 Kubernetes API 運行混沌實驗

Chaos Mesh 使用 CRD 定義混沌物件，因此您可以直接透過 Kubernetes API 操作 CRD 物件。這種方式非常方便將 Chaos Mesh 應用於您自己的應用程式，並進行自訂測試場景和自動化混沌實驗。

在 [test-infra](https://github.com/pingcap/tipocket/tree/35206e8483b66f9728b7b14823a10b3e4114e0e3/test-infra) 專案中，我們模擬 Kubernetes 上 [etcd](https://github.com/pingcap/tipocket/blob/35206e8483b66f9728b7b14823a10b3e4114e0e3/test-infra/tests/etcd/nemesis_test.go) 集群的潛在錯誤，包括節點重啟、網路故障和檔案系統故障。

以下是一個使用 Kubernetes API 的 Chaos Mesh 範例腳本：

```go
import (
  "context"

  "github.com/chaos-mesh/chaos-mesh/api/v1alpha1"
  "sigs.k8s.io/controller-runtime/pkg/client"
)

func main() {
  // ...
  delay := &chaosv1alpha1.NetworkChaos{
    Spec: chaosv1alpha1.NetworkChaosSpec{
      // ...
    },
  }
  k8sClient := client.New(conf, client.Options{ Scheme: scheme.Scheme })
  k8sClient.Create(context.TODO(), delay)
  k8sClient.Delete(context.TODO(), delay)
}
```

## 未來展望？

在本文中，我們向您介紹了 Chaos Mesh，這是一個開源的雲原生混沌工程平台。目前仍有許多功能正在開發中，關於設計、使用案例和開發的更多細節將會陸續揭露。敬請期待。

開源只是一個起點。除了前文介紹的基礎設施層級的混沌實驗之外，我們正在支援更廣泛且更細粒度的故障類型，例如：

- 藉助 eBPF 等工具在系統呼叫和核心層級注入錯誤

- 透過整合 [failpoint](https://github.com/pingcap/failpoint)，在應用程式函數和語句層級注入特定錯誤類型，這將涵蓋傳統注入方法無法實現的場景

未來，我們將持續改進 Chaos Mesh 儀表板，讓使用者能夠輕鬆查看其線上業務是否受到故障注入的影響以及影響方式。此外，我們的發展藍圖還包括一個易於使用的故障編排介面。我們還計劃推出其他酷炫功能，例如 Chaos Mesh Verifier 和 Chaos Mesh Cloud。

如果您對這些內容感興趣，歡迎加入我們，共同打造世界級的混沌工程平台。願我們的應用在 Kubernetes 的混沌中舞動！

如果您發現錯誤或認為缺少某些內容，請隨時提交 [issue](https://github.com/chaos-mesh/chaos-mesh/issues)、開啟 PR，或在 [CNCF Slack](https://slack.cncf.io/) 工作區的 #project-chaos-mesh 頻道中留言給我們。

GitHub: [https://github.com/chaos-mesh/chaos-mesh](https://github.com/chaos-mesh/chaos-mesh)
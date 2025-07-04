---
title: Integrate Chaos Mesh to GitHub Actions
---

本文件描述如何整合 Chaos Mesh 使用 chaos-mesh-action 來自訂持續整合（CI）。這有助於您在產品發布前識別系統開發中引入的問題。

chaos-mesh-action 是已發布於 [GitHub Marketplace](https://github.com/marketplace/actions/chaos-mesh) 的 GitHub Action，其原始碼同樣位於 [GitHub](https://github.com/chaos-mesh/chaos-mesh-action)。

## chaos-mesh-action 的設計

[GitHub Action](https://docs.github.com/en/actions) 是 GitHub 原生支援的持續整合（CI）與持續部署（CD）功能。使用 GitHub Action，您可以直接在儲存庫中透過 GitHub Actions 輕鬆自動化並自訂軟體開發工作流程。

透過 GitHub Action，Chaos Mesh 能輕鬆整合至日常開發與測試，確保所有提交至 GitHub 的程式碼均無缺陷（至少通過測試），且不影響現有邏輯。下圖展示 chaos-mesh-action 在 CI 工作流程中的整合：

![chaos-mesh-action 整合至 CI 工作流程](./img/chaos-mesh-action-integrate-in-the-ci-workflow.png)

## 在 GitHub 工作流程中使用 chaos-mesh-action

chaos-mesh-action 適用於 GitHub 工作流程。GitHub 工作流程是可配置的自動化流程，您可在儲存庫中設定 GitHub 工作流程來建構、測試、封裝、發布或部署任何 GitHub 專案。整合 Chaos Mesh 至 CI 的流程如下：

- 步驟 1：設計工作流程

- 步驟 2：建立工作流程

- 步驟 3：執行工作流程

### 步驟 1：設計工作流程

設計工作流程前，需考量以下問題：

- 在此工作流程中需測試哪些功能？

- 將注入何種類型的故障？

- 如何驗證系統正確性？

例如，我們可設計一個簡單的測試工作流程，包含以下步驟：

1. 在 Kubernetes 叢集中建立兩個 Pod

2. 從一個 Pod 向另一個 Pod 傳送 ping 請求

3. 使用 Chaos Mesh 注入網路延遲故障，測試 ping 指令是否受影響

### 步驟 2：建立工作流程

設計完成工作流程後，請依下列步驟建立工作流程：

1. 進入待測試軟體的 GitHub 儲存庫

2. 點擊 `Actions` 後點擊 `New workflow` 建立工作流程

![建立工作流程](./img/creating-a-workflow.png)

本質上，工作流程是序列化的自動化任務配置。請注意以下任務配置於單一檔案中，為清晰說明，腳本拆分為不同工作組，如下所示：

- 設定工作流程名稱與觸發規則

  將工作流程命名為 "Chaos"。當您提交程式碼或建立拉取請求至 master 分支時，此工作流程將被觸發。

  ```yaml
  name: Chaos

  on:
    push:
      branches:
        - master
    pull_request:
      branches:
        - master
  ```

- 安裝 CI 相關環境。

  此配置指定操作系統 (Ubuntu)，並使用 helm/kind-action 創建 Kind 集群。之後打印集群信息。最後檢出工作流要訪問的 GitHub 倉庫。

  ```yaml
  jobs:
    build:
      runs-on: ubuntu-latest
      steps:
        - name: Creating kind cluster
          uses: helm/kind-action@v1.0.0-rc.1

        - name: Print cluster information
          run: |
            kubectl config view
            kubectl cluster-info
            kubectl get nodes
            kubectl get pods -n kube-system
            helm version
            kubectl version

        - uses: actions/checkout@v2
  ```

- 部署應用程式。

  在以下範例中，此作業部署的應用程式會創建兩個 Kubernetes Pod。

  ```yaml
  - name: Deploy an application
       run: |
         kubectl apply -f https://raw.githubusercontent.com/chaos-mesh/apps/master/ping/busybox-statefulset.yaml
  ```

- 使用 Chaos Mesh 注入故障。

  ```yaml
  - name: Run chaos mesh action
      uses: chaos-mesh/chaos-mesh-action@v0.5
      env:
        CHAOS_MESH_VERSION: v1.0.0
        CFG_BASE64: YXBpVmVyc2lvbjogY2hhb3MtbWVzaC5vcmcvdjFhbHBoYTEKa2luZDogTmV0d29ya0NoYW9zCm1ldGFkYXRhOgogIG5hbWU6IG5ldHdvcmstZGVsYXkKICBuYW1lc3BhY2U6IGJ1c3lib3gKc3BlYzoKICBhY3Rpb246IGRlbGF5ICMgdGhlIHNwZWNpZmljIGNoYW9zIGFjdGlvbiB0byBpbmplY3QKICBtb2RlOiBhbGwKICBzZWxlY3RvcjoKICAgIHBvZHM6CiAgICAgIGJ1c3lib3g6CiAgICAgICAgLSBidXN5Ym94LTAKICBkZWxheToKICAgIGxhdGVuY3k6ICIxMG1zIgogIGR1cmF0aW9uOiAiNXMiCiAgc2NoZWR1bGVyOgogICAgY3JvbjogIkBldmVyeSAxMHMiCiAgZGlyZWN0aW9uOiB0bwogIHRhcmdldDoKICAgIHNlbGVjdG9yOgogICAgICBwb2RzOgogICAgICAgIGJ1c3lib3g6CiAgICAgICAgICAtIGJ1c3lib3gtMQogICAgbW9kZTogYWxsCg==
  ```

  使用 chaos-mesh-action 時，Chaos Mesh 會自動安裝並注入故障。您只需準備混沌實驗的配置並取得其 base64 編碼值。若要對 Pod 注入網絡延遲，可使用以下配置範例：

  ```yaml
  apiVersion: chaos-mesh.org/v1alpha1
  kind: NetworkChaos
  metadata:
    name: network-delay
    namespace: busybox
  spec:
    action: delay # 要注入的具體混沌操作
    mode: all
    selector:
      pods:
        busybox:
          - busybox-0
    delay:
      latency: '10ms'
    duration: '5s'
    scheduler:
      cron: '@every 10s'
    direction: to
    target:
      selector:
        pods:
          busybox:
            - busybox-1
      mode: all
  ```

  使用以下命令取得上述混沌實驗配置檔案的 base64 編碼值：

  ```bash
  base64 chaos.yaml
  ```

- 驗證系統的正確性。

  在此作業中，工作流程會從一個 Pod 向另一個 Pod 傳送 ping 請求，並觀察網路延遲。

  ```yaml
  - name: Verify
       run: |
         echo "do some verification"
         kubectl exec busybox-0 -it -n busybox -- ping -c 30 busybox-1.busybox.busybox.svc
  ```

### 步驟 3：執行工作流程

建立工作流程後，您可以透過建立拉取請求（pull request）到主分支來觸發它。當工作流程完成執行後，作業驗證的輸出類似以下內容：

```log
do some verification
Unable to use a TTY - input is not a terminal or the right kind of file
PING busybox-1.busybox.busybox.svc (10.244.0.6): 56 data bytes
64 bytes from 10.244.0.6: seq=0 ttl=63 time=0.069 ms
64 bytes from 10.244.0.6: seq=1 ttl=63 time=10.136 ms
64 bytes from 10.244.0.6: seq=2 ttl=63 time=10.192 ms
64 bytes from 10.244.0.6: seq=3 ttl=63 time=10.129 ms
64 bytes from 10.244.0.6: seq=4 ttl=63 time=10.120 ms
64 bytes from 10.244.0.6: seq=5 ttl=63 time=0.070 ms
64 bytes from 10.244.0.6: seq=6 ttl=63 time=0.073 ms
64 bytes from 10.244.0.6: seq=7 ttl=63 time=0.111 ms
64 bytes from 10.244.0.6: seq=8 ttl=63 time=0.070 ms
64 bytes from 10.244.0.6: seq=9 ttl=63 time=0.077 ms
……
```

輸出顯示一系列 10 毫秒的延遲，每次延遲持續 5 秒（共 5 次）。這與使用 chaos-mesh-action 注入的混亂實驗配置一致。

## 後續步驟

目前 chaos-mesh-action 已應用於 [TiDB Operator](https://github.com/pingcap/tidb-operator)。透過在工作流程中注入 Pod 故障，您可以驗證 Operator 實例的重啟機制，確保當 TiDB operator 的 Pod 被注入的故障隨機刪除時，TiDB Operator 仍能正常運作。詳情請參閱 [TiDB Operator 工作流程頁面](https://github.com/pingcap/tidb-operator/actions?query=workflow%3Achaos)。

未來 chaos-mesh-action 將應用於更多 TiDB 測試中，以確保 TiDB 及其元件的穩定性。歡迎您使用 chaos-mesh-action 建立自己的工作流程。

If you find any issue in chaos-mesh-action, or find any information is missing, you are welcome to create an [GitHub issue](https://github.com/pingcap/chaos-mesh/issues) or a [pull request (PR)](https://github.com/chaos-mesh/chaos-mesh/pulls) in the Chaos Mesh repository. You can also join our slack channel [#project-chaos-mesh](https://slack.cncf.io/) in the [CNCF](https://www.cncf.io/) workspace.
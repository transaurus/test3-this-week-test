---
slug: /chaos-mesh-action-integrate-chaos-engineering-into-your-ci
title: 'chaos-mesh-action: Integrate Chaos Engineering into Your CI'
authors: xiangwang
image: /img/blog/chaos-mesh-action.png
tags: [Chaos Mesh, Chaos Engineering, GitHub Actions, CI]
---

![chaos-mesh-action - 將混沌工程整合到您的 CI 中](/img/blog/chaos-mesh-action.png)

[Chaos Mesh](https://chaos-mesh.org) 是一個雲原生的混沌測試平台，可在 Kubernetes 環境中編排混沌實驗。儘管它以其豐富的故障注入類型和易於使用的儀表板在社群中廣受好評，但將 Chaos Mesh 與端到端測試或持續整合（CI）流程一起使用卻很困難。因此，在系統開發期間引入的問題無法在發布前被發現。

在本文中，我將分享我們如何使用 chaos-mesh-action（一個 GitHub Action）將 Chaos Mesh 整合到 CI 流程中。

<!--truncate-->

chaos-mesh-action 可在 [GitHub 市集](https://github.com/marketplace/actions/chaos-mesh)上取得，其原始碼位於 [GitHub](https://github.com/chaos-mesh/chaos-mesh-action)。

## chaos-mesh-action 的設計

[GitHub Actions](https://docs.github.com/en/actions) 是 GitHub 原生支援的 CI/CD 功能，透過它我們可以在 GitHub 儲存庫中輕鬆建構自動化和客製化的軟體開發工作流程。

結合 GitHub Actions，Chaos Mesh 可以更輕鬆地整合到系統的日常開發和測試中，從而確保在 GitHub 上提交的每一段程式碼都沒有錯誤，並且不會損壞現有程式碼。下圖顯示了 chaos-mesh-action 在 CI 工作流程中的整合：

![chaos-mesh-action 在 CI 工作流程中的整合](/img/blog/chaos-mesh-action-integrate-in-the-ci-workflow.png)

## 在 GitHub 工作流程中使用 chaos-mesh-action

[chaos-mesh-action](https://github.com/marketplace/actions/chaos-mesh) 在 Github 工作流程中運作。GitHub 工作流程是一個可配置的自動化流程，您可以在儲存庫中設定它來建構、測試、打包、發布或部署任何 GitHub 專案。要將 Chaos Mesh 整合到您的 CI 中，請執行以下操作：

1. 設計工作流程。

2. 建立工作流程。

3. 執行工作流程。

### 設計工作流程

在設計工作流程之前，您必須考慮以下問題：

- 我們要在這個工作流程中測試哪些功能？

- 我們將注入哪些類型的故障？

- 我們如何驗證系統的正確性？

舉例來說，我們設計一個簡單的測試工作流程，包含以下步驟：

1. 在 Kubernetes 叢集中建立兩個 Pod。

2. 從一個 Pod ping 另一個 Pod。

3. 使用 Chaos Mesh 注入網路延遲混沌，並測試 ping 指令是否受到影響。

### 建立工作流程

設計工作流程後，下一步是建立它。

1. 導航至包含您要測試的軟體的 GitHub 儲存庫。

2. 要開始建立工作流程，請點擊 **Actions**，然後點擊 **New workflow** 按鈕：

![建立工作流程](/img/blog/creating-a-workflow.png)

工作流程本質上是按順序自動執行的作業配置。請注意，作業在單一檔案中配置。為了更好地說明，我們將腳本拆分為不同的作業群組，如下所示：

- 設定工作流程名稱和觸發規則。

  此作業將工作流程命名為 "Chaos"。當程式碼被推送到 master 分支或向 master 分支提交拉取請求時，此工作流程將被觸發。

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

  此配置指定了作業系統 (Ubuntu)，並使用 [helm/kind-action](https://github.com/marketplace/actions/kind-cluster) 來建立 Kind 叢集。接著，它會輸出與叢集相關的資訊。最後，它會簽出 GitHub 儲存庫以供工作流程存取。

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

  在我們的範例中，此工作部署了一個會建立兩個 Kubernetes Pod 的應用程式。

  ```yaml
  - name: Deploy an application
       run: |
         kubectl apply -f https://raw.githubusercontent.com/chaos-mesh/apps/master/ping/busybox-statefulset.yaml
  ```

- 使用 chaos-mesh-action 注入混沌。

  ```yaml
  - name: Run chaos mesh action
      uses: chaos-mesh/chaos-mesh-action@v0.5
      env:
        CHAOS_MESH_VERSION: v1.0.0
        CFG_BASE64: YXBpVmVyc2lvbjogY2hhb3MtbWVzaC5vcmcvdjFhbHBoYTEKa2luZDogTmV0d29ya0NoYW9zCm1ldGFkYXRhOgogIG5hbWU6IG5ldHdvcmstZGVsYXkKICBuYW1lc3BhY2U6IGJ1c3lib3gKc3BlYzoKICBhY3Rpb246IGRlbGF5ICMgdGhlIHNwZWNpZmljIGNoYW9zIGFjdGlvbiB0byBpbmplY3QKICBtb2RlOiBhbGwKICBzZWxlY3RvcjoKICAgIHBvZHM6CiAgICAgIGJ1c3lib3g6CiAgICAgICAgLSBidXN5Ym94LTAKICBkZWxheToKICAgIGxhdGVuY3k6ICIxMG1zIgogIGR1cmF0aW9uOiAiNXMiCiAgc2NoZWR1bGVyOgogICAgY3JvbjogIkBldmVyeSAxMHMiCiAgZGlyZWN0aW9uOiB0bwogIHRhcmdldDoKICAgIHNlbGVjdG9yOgogICAgICBwb2RzOgogICAgICAgIGJ1c3lib3g6CiAgICAgICAgICAtIGJ1c3lib3gtMQogICAgbW9kZTogYWxsCg==
  ```

  透過 chaos-mesh-action，Chaos Mesh 的安裝和混沌的注入會自動完成。您只需要準備打算使用的混沌配置，並取得其 Base64 表示。在此，我們想要將網路延遲混沌注入 Pod，因此我們使用原始的混沌配置如下：

  ```yaml
  apiVersion: chaos-mesh.org/v1alpha1
  kind: NetworkChaos
  metadata:
    name: network-delay
    namespace: busybox
  spec:
    action: delay # the specific chaos action to inject
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

  您可以使用以下指令取得上述混沌配置檔案的 Base64 值：

  ```shell
  $ base64 chaos.yaml
  ```

- 驗證系統正確性。

  在此作業中，工作流程從一個 Pod 對另一個 Pod 執行 ping 操作，並觀察網路延遲的變化。

  ```yaml
  - name: Verify
       run: |
         echo "do some verification"
         kubectl exec busybox-0 -it -n busybox -- ping -c 30 busybox-1.busybox.busybox.svc
  ```

### 執行工作流程

現在工作流程已配置完成，我們可以透過向主分支提交拉取請求來觸發它。當工作流程完成時，驗證作業會輸出類似以下的結果：

```shell
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

輸出顯示了一系列規律的 10 毫秒延遲，每次持續約 5 秒。這與我們注入 chaos-mesh-action 的混沌配置一致。

## 目前狀態與後續步驟

目前，我們已將 chaos-mesh-action 應用於 [TiDB Operator](https://github.com/pingcap/tidb-operator) 專案。該工作流程注入了 Pod 混沌以驗證 operator 指定實例的重啟功能。目的是確保當 operator 的 Pod 被注入的故障隨機刪除時，tidb-operator 仍能正常運作。您可以在 [TiDB Operator 頁面](https://github.com/pingcap/tidb-operator/actions?query=workflow%3Achaos) 查看詳情。

未來，我們計劃將 chaos-mesh-action 應用於更多測試中，以確保 TiDB 及相關元件的穩定性。歡迎您使用 chaos-mesh-action 建立自己的工作流程。

If you find a bug or think something is missing, feel free to file an issue, open a pull request (PR), or join us on the [#project-chaos-mesh](https://slack.cncf.io/) channel in the [CNCF](https://www.cncf.io/) slack workspace.
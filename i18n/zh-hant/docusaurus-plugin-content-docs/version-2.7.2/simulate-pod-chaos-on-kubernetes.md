---
title: Simulate Pod Faults
---

本文說明如何使用 Chaos Mesh 向 Kubernetes Pod 注入故障，以模擬 Pod 或容器故障。提供 Chaos Dashboard 和 YAML 檔案來建立 PodChaos 實驗。

## PodChaos 介紹

PodChaos 是 Chaos Mesh 中的故障類型。透過建立 PodChaos 實驗，您可以模擬指定 Pod 或容器的故障情境。目前 PodChaos 支援以下故障類型：

- **Pod Failure**：向指定 Pod 注入故障，使 Pod 在一段時間內不可用。

- **Pod Kill**：終止指定 Pod。為確保 Pod 能成功重啟，需配置 ReplicaSet 或類似機制。

- **Container Kill**：終止目標 Pod 中指定的容器。

## 使用限制

Chaos Mesh 可向任何 Pod 注入 PodChaos，無論該 Pod 是否綁定 Deployment、StatefulSet、DaemonSet 或其他控制器。但當您向獨立 Pod 注入 PodChaos 時，可能出現不同情況。例如當向獨立 Pod 注入 "pod-kill" 混沌時，Chaos Mesh 無法保證應用程式能從故障中恢復。

## 注意事項

建立 PodChaos 實驗前，請確保：

- 目標 Pod 上未運行 Chaos Mesh 的控制管理器。

- 若故障類型為 Pod Kill，需配置 replicaSet 或類似機制以確保 Pod 能自動重啟。

## 使用 Chaos Dashboard 建立實驗

:::note

使用 Chaos Dashboard 建立實驗前，請確保：

- 已安裝 Chaos Dashboard。
- 若已安裝 Chaos Dashboard，可執行 `kubectl port-forward` 存取 Dashboard：`bash kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333`。接著可透過 [`http://localhost:2333`](http://localhost:2333) 存取 Chaos Dashboard。

:::

1. 開啟 Chaos Dashboard，點擊頁面中的 **NEW EXPERIMENT** 建立新實驗。

   ![建立新實驗](./img/create-new-exp.png)

2. 在 **Choose a Target** 區域，選擇 **POD FAULT** 並選取特定行為，例如 **POD FAILURE**。

3. 填寫實驗資訊，並指定實驗範圍與排定的實驗持續時間。

4. 提交實驗資訊。

## 使用 YAML 設定檔建立實驗

### pod-failure 範例

1. 將實驗配置寫入 `pod-failure.yaml` 檔案：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: PodChaos
   metadata:
     name: pod-failure-example
     namespace: chaos-mesh
   spec:
     action: pod-failure
     mode: one
     duration: '30s'
     selector:
       labelSelectors:
         'app.kubernetes.io/component': 'tikv'
   ```

   根據此範例，Chaos Mesh 將 `pod-failure` 注入指定 Pod，使該 Pod 在 30 秒內不可用。

2. 準備好設定檔後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f ./pod-failure.yaml
   ```

### pod-kill 範例

1. 將實驗配置寫入 `pod-kill.yaml` 檔案：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: PodChaos
   metadata:
     name: pod-kill-example
     namespace: chaos-mesh
   spec:
     action: pod-kill
     mode: one
     selector:
       namespaces:
         - tidb-cluster-demo
       labelSelectors:
         'app.kubernetes.io/component': 'tikv'
   ```

   根據此範例，Chaos Mesh 將 `pod-kill` 注入指定 Pod 並終止該 Pod 一次。

2. 準備好配置檔案後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f ./pod-kill.yaml
   ```

### container-kill 範例

1. 將實驗配置寫入 `container-kill.yaml` 檔案：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: PodChaos
   metadata:
     name: container-kill-example
     namespace: chaos-mesh
   spec:
     action: container-kill
     mode: one
     containerNames: ['prometheus']
     selector:
       labelSelectors:
         'app.kubernetes.io/component': 'monitor'
   ```

   根據此範例，Chaos Mesh 將 `container-kill` 注入指定容器並終止該容器一次。

2. 準備好配置檔案後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f ./container-kill.yaml
   ```

### 欄位說明

下表描述 YAML 配置檔案中的欄位。

| Parameter | Type | Description | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| action | string | Specifies the fault type to inject. The supported types include `pod-failure`, `pod-kill`, and `container-kill`. | None | Yes | `pod-kill` |
| mode | string | Specifies the mode of the experiment. The mode options include `one` (selecting a random Pod), `all` (selecting all eligible Pods), `fixed` (selecting a specified number of eligible Pods), `fixed-percent` (selecting a specified percentage of Pods from the eligible Pods), and `random-max-percent` (selecting the maximum percentage of Pods from the eligible Pods). | None | Yes | `one` |
| value | string | Provides parameters for the `mode` configuration, depending on `mode`.For example, when `mode` is set to `fixed-percent`, `value` specifies the percentage of Pods. | None | No | 1 |
| selector | struct | Specifies the target Pod. For details, refer to [Define the experiment scope](./define-chaos-experiment-scope.md). | None | Yes |  |
| containerNames | []string | When you configure `action` to `container-kill`, this configuration is mandatory to specify the target container name for injecting faults. | None | No | ['prometheus'] |
| gracePeriod | int64 | When you configure `action` to `pod-kill`, this configuration is mandatory to specify the duration before deleting Pod. | 0 | No | 0 |
| duration | string | Specifies the duration of the experiment. | None | Yes | 30s |

## "Pod Failure" 混沌實驗注意事項

簡要說明：使用 "Pod Failure" 混沌實驗時有以下建議：

- 若在隔離環境的 Kubernetes 叢集操作，請改用可用的「暫停映像檔」(pause image)。

- 為容器設置 `livenessProbe` 和 `readinessProbe`。

Pod Failure 混沌實驗會將目標 Pod 中每個容器的 `image` 更改為「暫停映像檔」，這是一種不執行任何操作的特殊映像檔。預設使用 `gcr.io/google-containers/pause:latest` 作為暫停映像檔，您可透過 Helm 參數 `controllerManager.podChaos.podFailure.pauseImage` 更換為其他映像檔。

Downloading `pause image` would consume time, and that duration would be counted in the experiment duration. So you might find that the "actual effected duration" might be shorter than the configured duration. That's another reason why recommend to setup available "pause image".

另一項需注意之處是「暫停映像檔」對未配置 `command` 的容器可能顯示「正常運作」。若容器未配置 `command`、`livenessProbe` 和 `readinessProbe`，容器狀態將顯示為 `Running` 和 `Ready`，儘管它已被替換為「暫停映像檔」且實際上無法提供正常功能。因此強烈建議為容器設置 `livenessProbe` 和 `readinessProbe`。
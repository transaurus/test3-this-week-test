---
title: Simulate GCP Faults
---

本文件說明如何使用 Chaos Mesh 向 GCP Pod 注入故障。提供 Chaos Dashboard 和 YAML 文件來創建 GCPChaos 實驗。

## GCPChaos 介紹

GCPChaos 是 Chaos Mesh 中的故障類型。通過創建 GCPChaos 實驗，您可以模擬指定 GCP 實例的故障場景。目前 GCPChaos 支援以下故障類型：

- **節點停止**：停止指定的 GCP 實例。

- **節點重置**：重新啟動指定的 GCP 實例。

- **磁碟丟失**：從指定 GCP 實例卸載存儲卷。

## `Secret` 文件

為方便連接 GCP 集群，可預先創建 Kubernetes `Secret` 文件存儲身份驗證資訊。

以下是 `secret` 文件範例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloud-key-secret
  namespace: chaos-mesh
type: Opaque
stringData:
  service_account: your-gcp-service-account-base64-encode
```

- **name** 定義 Kubernetes secret 的名稱。

- **namespace** 定義 Kubernetes secret 的命名空間。

- **service_account** 存儲 GCP 集群的服務帳戶金鑰。請確保對 GCP 服務帳戶金鑰進行 [Base64](https://zh.wikipedia.org/wiki/Base64) 編碼。有關服務帳戶金鑰詳情，請參閱[創建和管理服務帳戶金鑰](https://cloud.google.com/iam/docs/creating-managing-service-account-keys)。

## 使用 Chaos Dashboard 創建實驗

:::note

使用 Chaos Dashboard 創建實驗前，請確保滿足以下要求：

1. 已安裝 Chaos Dashboard。
2. 可通過 `kubectl port-forward` 命令訪問 Chaos Dashboard：

   ```bash
   kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333
   ```

   然後即可在瀏覽器中通過 [`http://localhost:2333`](http://localhost:2333) 訪問儀表板。

:::

1. 打開 Chaos Dashboard，點擊頁面中的 **NEW EXPERIMENT** 創建新實驗：

   ![img](img/create-pod-chaos-on-dashboard-1.png)

2. 在 **Choose a Target** 區域選擇 **GCP fault**，並選取特定行為如 **STOP NODE**：

   ![img](img/create-gcp-chaos-on-dashboard-2.png)

3. 填寫實驗資訊，指定實驗範圍和計劃持續時間：

   ![img](img/create-gcp-chaos-on-dashboard-3.png)

   ![img](img/create-gcp-chaos-on-dashboard-4.png)

4. 提交實驗資訊。

## 使用 YAML 文件創建實驗

### `node-stop` 配置範例

1. 將實驗配置寫入 `gcpchaos-node-stop.yaml`，如下所示：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: GCPChaos
   metadata:
     name: node-stop-example
     namespace: chaos-mesh
   spec:
     action: node-stop
     secretName: 'cloud-key-secret'
     project: 'your-project-id'
     zone: 'your-zone'
     instance: 'your-instance-name'
     duration: '5m'
   ```

   根據此配置範例，Chaos Mesh 將向指定 GCP 實例注入 `node-stop` 故障，使該實例在 5 分鐘內不可用。

   有關停止 GCP 實例的更多資訊，請參閱[停止 GCP 實例](https://cloud.google.com/compute/docs/instances/stop-start-instance)。

2. 準備好配置文件後，使用 `kubectl` 創建實驗：

   ```bash
   kubectl apply -f gcpchaos-node-stop.yaml
   ```

### `node-reset` 配置範例

1. 將實驗配置寫入 `gcpchaos-node-reset.yaml`，如下所示：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: GCPChaos
   metadata:
     name: node-reset-example
     namespace: chaos-mesh
   spec:
     action: node-reset
     secretName: 'cloud-key-secret'
     project: 'your-project-id'
     zone: 'your-zone'
     instance: 'your-instance-name'
     duration: '5m'
   ```

   根據此配置範例，Chaos Mesh 將對指定的 GCP 實例注入 `node-reset` 故障，使該 GCP 實例被重置。

   有關重置 GCP 實例的更多資訊，請參閱 [重置 GCP 實例](https://cloud.google.com/compute/docs/instances/stop-start-instance#resetting_an_instance)。

2. 配置檔案準備好後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f gcpchaos-node-reset.yaml
   ```

### `disk-loss` 配置範例

1. 將實驗配置寫入 `gcpchaos-disk-loss.yaml`，如下所示：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: GCPChaos
   metadata:
     name: disk-loss-example
     namespace: chaos-mesh
   spec:
     action: disk-loss
     secretName: 'cloud-key-secret'
     project: 'your-project-id'
     zone: 'your-zone'
     instance: 'your-instance-name'
     deviceNames: ['disk-name']
     duration: '5m'
   ```

   根據此配置範例，Chaos Mesh 將對指定的 GCP 實例注入 `disk-loss` 故障，使該 GCP 實例在 5 分鐘內與指定的儲存卷分離。

   有關分離 GCP 儲存的更多資訊，請參閱 [分離 GCP 儲存](https://cloud.google.com/compute/docs/reference/rest/v1/instances/detachDisk)。

2. 配置檔案準備好後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f gcpchaos-disk-loss.yaml
   ```

### 欄位說明

下表顯示了 YAML 配置檔案中的欄位。

| Parameter | Type | Descpription | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| action | string | Indicates the specific type of faults. The available fault types include node-stop, node-reset, and disk-loss. | node-stop | Yes | node-stop |
| mode | string | Indicates the mode of the experiment. The mode options include `one` (selecting a Pod at random), `all` (selecting all eligible Pods), `fixed` (selecting a specified number of eligible Pods), `fixed-percent` (selecting a specified percentage of the eligible Pods), and `random-max-percent` (selecting the maximum percentage of the eligible Pods). | None | Yes | `one` |
| value | string | Provides parameters for the `mode` configuration, depending on `mode`. For example, when `mode` is set to `fixed-percent`, `value` specifies the percentage of pods. | None | No | 1 |
| secretName | string | Indicates the name of the Kubernetes secret that stores the GCP authentication information. | None | No | cloud-key-secret |
| project | string | Indicates the ID of GCP project. | None | Yes | real-testing-project |
| zone | string | Indicates the region of GCP instance. | None | Yes | us-central1-a |
| instance | string | Indicates the name of GCP instance. | None | Yes | gke-xxx-cluster--default-pool-xxx-yyy |
| deviceNames | []string | This is a required field when the `action` is `disk-loss`. This field specifies the machine disk ID. | None | no | ["your-disk-id"] |
| duration | string | Indicates the duration of the experiment. | None | Yes | 30s |
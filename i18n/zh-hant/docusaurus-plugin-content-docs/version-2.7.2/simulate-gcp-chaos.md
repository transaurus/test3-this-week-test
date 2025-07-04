---
title: Simulate GCP Faults
---

本文件說明如何使用 Chaos Mesh 向 GCP Pod 注入故障。提供 Chaos Dashboard 和 YAML 檔案來建立 GCPChaos 實驗。

## GCPChaos 介紹

GCPChaos 是 Chaos Mesh 中的一種故障類型。透過建立 GCPChaos 實驗，您可以模擬指定 GCP 執行個體的故障場景。目前，GCPChaos 支援以下故障類型：

- **節點停止（Node Stop）**：停止指定的 GCP 執行個體。

- **節點重置（Node Reset）**：重新啟動指定的 GCP 執行個體。

- **磁碟遺失（Disk Loss）**：從指定的 GCP 執行個體卸載儲存卷。

## `Secret` 檔案

為了方便連線到 GCP 叢集，您可以事先建立一個 Kubernetes `Secret` 檔案來儲存驗證資訊。

以下是一個 `secret` 檔案的範例：

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

- **name**：定義 Kubernetes secret 的名稱。

- **namespace**：定義 Kubernetes secret 的命名空間。

- **service_account**：儲存 GCP 叢集的服務帳戶金鑰。請記得對您的 GCP 服務帳戶金鑰進行 [Base64](https://zh.wikipedia.org/wiki/Base64) 編碼。要了解更多關於服務帳戶金鑰的資訊，請參閱[建立和管理服務帳戶金鑰](https://cloud.google.com/iam/docs/creating-managing-service-account-keys)。

## 使用 Chaos Dashboard 建立實驗

:::note

在使用 Chaos Dashboard 建立實驗之前，請確保滿足以下要求：

1. Chaos Dashboard 已安裝。
2. 可以使用 `kubectl port-forward` 指令存取 Chaos Dashboard：

   ```bash
   kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333
   ```

   然後，您可以在瀏覽器中透過 [`http://localhost:2333`](http://localhost:2333) 存取儀表板。

:::

1. 開啟 Chaos Dashboard，在頁面上點擊 **NEW EXPERIMENT** 以建立新實驗：

   ![img](img/create-pod-chaos-on-dashboard-1.png)

2. 在 **Choose a Target** 區域中，選擇 **GCP fault** 並選擇一個特定行為，例如 **STOP NODE**：

   ![img](img/create-gcp-chaos-on-dashboard-2.png)

3. 填寫實驗資訊，並指定實驗範圍和排程的實驗持續時間：

   ![img](img/create-gcp-chaos-on-dashboard-3.png)

   ![img](img/create-gcp-chaos-on-dashboard-4.png)

4. 提交實驗資訊。

## 使用 YAML 檔案建立實驗

### `node-stop` 設定範例

1. 將實驗設定寫入 `gcpchaos-node-stop.yaml` 檔案，如下所示：

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

   根據此設定範例，Chaos Mesh 會將 `node-stop` 故障注入指定的 GCP 執行個體，使該 GCP 執行個體在 5 分鐘內不可用。

   有關停止 GCP 執行個體的更多資訊，請參閱[停止 GCP 執行個體](https://cloud.google.com/compute/docs/instances/stop-start-instance)。

2. 準備好設定檔案後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f gcpchaos-node-stop.yaml
   ```

### `node-reset` 設定範例

1. 將實驗配置寫入 `gcpchaos-node-reset.yaml` 檔案，如下所示：

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

   根據此配置範例，Chaos Mesh 將對指定的 GCP 實例注入 `node-reset` 故障，使該 GCP 實例重新啟動。

   有關重置 GCP 實例的更多資訊，請參閱[重置 GCP 實例](https://cloud.google.com/compute/docs/instances/stop-start-instance#resetting_an_instance)。

2. 準備好配置文件後，使用 `kubectl` 創建實驗：

   ```bash
   kubectl apply -f gcpchaos-node-reset.yaml
   ```

### `disk-loss` 配置範例

1. 將實驗配置寫入 `gcpchaos-disk-loss.yaml` 檔案，如下所示：

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

   根據此配置範例，Chaos Mesh 將對指定的 GCP 實例注入 `disk-loss` 故障，使該 GCP 實例在 5 分鐘內與指定的存儲卷分離。

   有關分離 GCP 存儲的更多資訊，請參閱[分離 GCP 存儲](https://cloud.google.com/compute/docs/reference/rest/v1/instances/detachDisk)。

2. 準備好配置文件後，使用 `kubectl` 創建實驗：

   ```bash
   kubectl apply -f gcpchaos-disk-loss.yaml
   ```

### 字段說明

下表顯示了 YAML 配置文件中的字段。

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
---
title: Simulate Azure Faults
---

本文說明如何使用 Chaos Mesh 模擬 Azure 故障。

## AzureChaos 介紹

AzureChaos 可協助您在指定的 Azure 實例上模擬故障情境。目前支援以下故障類型：

- **VM Stop**：停止指定的 VM 實例。

- **VM Restart**：重新啟動指定的 VM 實例。

- **Disk Detach**：從指定的 VM 實例卸載資料磁碟。

## `Secret` 文件

為簡化連接 Azure 叢集的操作，您可預先建立 Kubernetes `Secret` 文件儲存驗證資訊。

`Secret` 文件範例如下：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloud-key-secret
  namespace: chaos-mesh
type: Opaque
stringData:
  client_id: your-client-id
  client_secret: your-client-secret
  tenant_id: your-tenant-id
```

- **name** 表示 Kubernetes Secret 物件。

- **namespace** 表示 Kubernetes Secret 物件的命名空間。

- **client_id** 儲存 Azure 應用程式註冊的應用程式 (用戶端) ID。

- **client_secret** 儲存 Azure 應用程式註冊的應用程式 (用戶端) 密鑰值。

- **tenant_id** 儲存 Azure 應用程式註冊的目錄 (租戶) ID。關於 `client_id` 和 `client_secret` 的詳細資訊，請參閱 [機密用戶端應用程式](https://docs.microsoft.com/en-us/azure/healthcare-apis/azure-api-for-fhir/register-confidential-azure-ad-client-app)。

:::note

請確保 Secret 文件中的應用程式註冊已被添加為 VM 實例存取控制 (IAM) 的參與者或擁有者。

:::

## 使用 Chaos Dashboard 建立實驗

1. 開啟 Chaos Dashboard，點擊頁面上的 **NEW EXPERIMENT** 建立新實驗：

   ![img](./img/create-new-exp.png)

2. 在 **Choose a Target** 區域選擇 **AZURE FAULT** 並選取特定行為，例如 **VM STOP**。

3. 填寫實驗資訊，並指定實驗範圍與預定實驗持續時間。

4. 提交實驗資訊。

## 使用 YAML 文件建立實驗

### `vm-stop` 配置範例

1. 將實驗配置寫入 `azurechaos-vm-stop.yaml` 文件，如下所示：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: AzureChaos
   metadata:
     name: vm-stop-example
     namespace: chaos-mesh
   spec:
     action: vm-stop
     secretName: 'cloud-key-secret'
     subscriptionID: 'your-subscription-id'
     resourceGroupName: 'your-resource-group-name'
     vmName: 'your-vm-name'
     duration: '5m'
   ```

   根據此配置範例，Chaos Mesh 將對指定 VM 實例注入 `vm-stop` 故障，使該 VM 實例在 5 分鐘內不可用。

   有關停止 VM 實例的詳細資訊，請參閱 [Azure 文件 - 啟動或停止 VM](https://docs.microsoft.com/en-us/azure/devtest-labs/use-command-line-start-stop-virtual-machines)。

2. 準備好配置文件後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f azurechaos-vm-stop.yaml
   ```

### `vm-restart` 配置範例

1. 將實驗配置寫入 `azurechaos-vm-restart.yaml` 檔案：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: AzureChaos
   metadata:
     name: vm-restart-example
     namespace: chaos-mesh
   spec:
     action: vm-restart
     secretName: 'cloud-key-secret'
     subscriptionID: 'your-subscription-id'
     resourceGroupName: 'your-resource-group-name'
     vmName: 'your-vm-name'
   ```

   基於此配置範例，Chaos Mesh 將會對指定的 VM 實例注入 `vm-restart` 故障，從而導致該 VM 實例重新啟動。

   有關重新啟動 VM 實例的更多資訊，請參閱 [Azure 文件 - 重新啟動 VM](https://docs.microsoft.com/en-us/azure/devtest-labs/devtest-lab-restart-vm)。

2. 準備好配置檔案後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f azurechaos-vm-restart.yaml
   ```

### `detach-volume` 配置範例

1. 將實驗配置寫入 `azurechaos-disk-detach.yaml` 檔案：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: AzureChaos
   metadata:
     name: disk-detach-example
     namespace: chaos-mesh
   spec:
     action: disk-detach
     secretName: 'cloud-key-secret'
     subscriptionID: 'your-subscription-id'
     resourceGroupName: 'your-resource-group-name'
     vmName: 'your-vm-name'
     diskName: 'your-disk-name'
     lun: 'your-disk-lun'
     duration: '5m'
   ```

   基於此配置範例，Chaos Mesh 將會對指定的 VM 實例注入 `disk-detach` 故障，從而使該 VM 實例在 5 分鐘內與指定的資料磁碟分離。

   有關卸離 Azure 資料磁碟的更多資訊，請參閱 [Azure 文件 - 卸離資料磁碟](https://docs.microsoft.com/en-us/azure/devtest-labs/devtest-lab-attach-detach-data-disk#detach-a-data-disk)。

2. 準備好配置檔案後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f azurechaos-disk-detach.yaml
   ```

### 欄位說明

下表顯示了 YAML 配置檔案中的欄位。

| Parameter | Type | Description | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| action | string | Indicates the specific type of faults. Only `vm-stop`, `vm-restart`, and `disk-detach` are supported. | `vm-stop` | Yes | `vm-stop` |
| mode | string | Specifies the mode of the experiment. The mode options include `one` (selecting a random Pod), `all` (selecting all eligible Pods), `fixed` (selecting a specified number of eligible Pods), `fixed-percent` (selecting a specified percentage of Pods from the eligible Pods), and `random-max-percent` (selecting the maximum percentage of Pods from the eligible Pods). | N/A | Yes | `one` |
| value | string | Provides parameters for the `mode` configuration, depending on `mode`. For example, when `mode` is set to `fixed-percent`, `value` specifies the percentage of Pods. | N/A | No | `1` |
| secretName | string | Specifies the name of the Kubernetes Secret that stores the Azure authentication information. | N/A | No | `cloud-key-secret` |
| subscriptionID | string | Specifies the VM instacnce's subscription ID. | N/A | Yes | `your-subscription-id` |
| resourceGroupName | string | Specifies the Resource group of VM. | N/A | Yes | `your-resource-group-name` |
| vmName | string | VMName defines the name of Virtual Machine. | N/A | Yes | `your-vm-name` |
| diskName | string | This is a required field when the `action` is `disk-detach`, specifies the name of data disk. | N/A | No | `DATADISK_0` |
| lun | string | This is a required field when the `action` is `disk-detach`, specifies the LUN (Logic Unit Number) of data disk. | N/A | No | `0` |
| duration | string | Specifies the duration of the experiment. | N/A | Yes | `30s` |
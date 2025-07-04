---
title: Simulate AWS Faults
---

本文說明如何使用 Chaos Mesh 模擬 AWS 故障。

## AWSChaos 介紹

AWSChaos 可協助您在指定的 AWS 執行個體上模擬故障情境。目前支援以下故障類型：

- **EC2 Stop**：停止指定的 EC2 執行個體。

- **EC2 Restart**：重新啟動指定的 EC2 執行個體。

- **Detach Volume**：從指定的 EC2 執行個體卸載儲存卷。

## `Secret` 文件

為簡化連接 AWS 叢集流程，可預先建立 Kubernetes `Secret` 文件儲存認證資訊。

`Secret` 文件範例如下：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloud-key-secret
  namespace: chaos-mesh
type: Opaque
stringData:
  aws_access_key_id: your-aws-access-key-id
  aws_secret_access_key: your-aws-secret-access-key
  aws_session_token: your-aws-session-token
```

- **name** 表示 Kubernetes Secret 物件。

- **namespace** 表示 Kubernetes Secret 物件的命名空間。

- **aws_access_key_id** 儲存 AWS 叢集的存取金鑰 ID。

- **aws_secret_access_key** 儲存 AWS 叢集的秘密存取金鑰。

- **aws_session_token** 儲存 AWS 工作階段權杖（使用[臨時 AWS 憑證](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html)時需提供）。

## 透過 Chaos Dashboard 建立實驗

:::note

透過 Chaos Dashboard 建立實驗前，請確保滿足以下要求：

1. 已安裝 Chaos Dashboard。
2. 可透過 `kubectl port-forward` 存取 Dashboard：

   ```bash
    kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333
   ```

   隨後即可在瀏覽器中存取 [`http://localhost:2333`](http://localhost:2333)。

:::

1. 開啟 Chaos Dashboard，點擊頁面中的 **NEW EXPERIMENT** 建立新實驗：

   ![img](./img/create-new-exp.png)

2. 在 **Choose a Target** 區域選擇 **AWS FAULT**，並選取特定行為（例如 **STOP EC2**）。

3. 填寫實驗資訊，指定實驗範圍與排程執行時間。

4. 提交實驗資訊。

## 使用 YAML 文件建立實驗

### `ec2-stop` 設定範例

1. 將實驗設定寫入 `awschaos-ec2-stop.yaml` 文件，內容如下：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: AWSChaos
   metadata:
     name: ec2-stop-example
     namespace: chaos-mesh
   spec:
     action: ec2-stop
     secretName: 'cloud-key-secret'
     awsRegion: 'us-east-2'
     ec2Instance: 'your-ec2-instance-id'
     duration: '5m'
   ```

   此設定將對指定 EC2 執行個體注入 `ec2-stop` 故障，使該執行個體在 5 分鐘內不可用。

   有關停止 EC2 執行個體的詳細資訊，請參閱 [AWS 文件 - 停止與啟動執行個體](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Stop_Start.html)。

2. 準備好設定文件後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f awschaos-ec2-stop.yaml
   ```

### `ec2-start` 設定範例

1. 將實驗配置寫入 `awchaos-ec2-restot.yaml` 檔案：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: AWSChaos
   metadata:
     name: ec2-restart-example
     namespace: chaos-mesh
   spec:
     action: ec2-restart
     secretName: 'cloud-key-secret'
     awsRegion: 'us-east-2'
     ec2Instance: 'your-ec2-instance-id'
   ```

   根據此配置範例，Chaos Mesh 會將 `ec2-restart` 故障注入指定的 EC2 執行個體，從而重新啟動該 EC2 執行個體。

   有關重新啟動 EC2 執行個體的更多資訊，請參閱 [AWS 文件 - 重新啟動執行個體](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-reboot.html)。

2. 準備好配置文件後，使用 `kubectl` 創建實驗：

   ```bash
   kubectl apply -f awschaos-ec2-restart.yaml
   ```

### 一個 `detach-volume` 配置範例

1. Write the experiment configuration to the `awschaos-detach-volume.yaml` file:

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: AWSChaos
   metadata:
     name: ec2-detach-volume-example
     namespace: chaos-mesh
   spec:
     action: ec2-stop
     secretName: 'cloud-key-secret'
     awsRegion: 'us-east-2'
     ec2Instance: 'your-ec2-instance-id'
     volumeID: 'your-volume-id'
     deviceName: '/dev/sdf'
     duration: '5m'
   ```

   Based on this configuration example, Chaos Mesh will inject a `detail-volume` fault into the specified EC2 instance so that the EC2 instance is detached from the specified storage volume within 5 minutes.

   For more information about detaching Amazon EBS volumes, refer to the [AWS documentation - Detach an Amazon EBS volume from a Linux instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-detaching-volume.html).

2. 準備好配置文件後，使用 `kubectl` 創建實驗：

   ```bash
   kubectl apply -f awschaos-detach-volume.yaml
   ```

### 欄位描述

下表顯示了 YAML 配置文件中的欄位。

| Parameter | Type | Description | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| action | string | Indicates the specific type of faults. Only ec2-stop, ec2-restore, and detain-volume are supported. | ec2-stop | Yes | ec2-stop |
| mode | string | Specifies the mode of the experiment. The mode options include `one` (selecting a random Pod), `all` (selecting all eligible Pods), `fixed` (selecting a specified number of eligible Pods), `fixed-percent` (selecting a specified percentage of Pods from the eligible Pods), and `random-max-percent` (selecting the maximum percentage of Pods from the eligible Pods). | None | Yes | `one` |
| value | string | Provides parameters for the `mode` configuration, depending on `mode`.For example, when `mode` is set to `fixed-percent`, `value` specifies the percentage of Pods. | None | No | 1 |
| secretName | string | Specifies the name of the Kubernetes Secret that stores the AWS authentication information. | None | No | cloud-key-secret |
| awsRegion | string | Specifies the AWS region. | None | Yes | us-east-2 |
| ec2Instance | string | Specifies the ID of the EC2 instance. | None | Yes | your-ec2-instance-id |
| volumeID | string | This is a required field when the `action` is `detach-volume`. This field specifies the EBS volume ID. | None | No | your-volume-id |
| deviceName | string | This is a required field when the `action` is `detach-volume`. This field specifies the machine name. | None | No | /dev/sdf |
| duration | string | Specifies the duration of the experiment. | None | Yes | 30s |
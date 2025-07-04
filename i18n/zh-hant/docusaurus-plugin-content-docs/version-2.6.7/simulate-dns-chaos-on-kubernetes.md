---
title: Simulate DNS Faults
---

import PickHelmVersion from '@site/src/components/PickHelmVersion'

本文件說明如何在 Chaos Mesh 中建立 DNSChaos 實驗以模擬 DNS 故障。

:::info

要模擬 DNS 故障，您需要部署一個名為 Chaos DNS Server 的特殊 DNS 服務。

在 `v2.6` 之後，Chaos Mesh 預設會部署 Chaos DNS Server。如果您不需要模擬 DNS 故障，可以在安裝 Chaos Mesh 時將 `dnsServer.create` 設定為 `false`：

<PickHelmVersion>{`helm install chaos-mesh chaos-mesh/chaos-mesh --namespace=chaos-mesh --version latest --set dnsServer.create=false`}</PickHelmVersion>

:::

## DNSChaos 介紹

DNSChaos 用於模擬錯誤的 DNS 回應。例如，DNSChaos 可以在收到 DNS 請求時返回錯誤或隨機 IP 位址。

## 檢查 Chaos DNS Server 是否已部署

執行以下指令檢查 Chaos DNS Server 是否已部署：

```bash
kubectl get pods -n chaos-mesh -l app.kubernetes.io/component=chaos-dns-server
```

請確認 Pod 狀態為 `Running`。

## 注意事項

1. 目前，DNSChaos 僅支援記錄類型 `A` 和 `AAAA`。

2. Chaos DNS 服務使用 [k8s_dns_chaos](https://github.com/chaos-mesh/k8s_dns_chaos) 外掛執行 CoreDNS。如果您的 Kubernetes 叢集中的 CoreDNS 服務包含一些特殊設定，您可以使用以下指令編輯 configMap `dns-server-config`，以使 Chaos DNS 服務的設定與 K8s CoreDNS 服務一致：

   ```bash
   kubectl edit configmap dns-server-config -n chaos-mesh
   ```

## 使用 Chaos Dashboard 建立實驗

1. 開啟 Chaos Dashboard，在頁面上點擊 **NEW EXPERIMENT** 以建立新實驗：

   ![建立實驗](./img/create-new-exp.png)

2. In the **Choose a Target** area, choose **DNS FAULT** and select a specific behavior, such as **ERROR**. Then fill out the matching rules.

   ![DNSChaos Experiment](./img/dnschaos-exp.png)

   According to the matching rules configured in the screenshot, the DNS FAULT takes effect for domains including `google.com`, `chaos-mesh.org`, and `github.com`, which means that an error will be returned when a DNS request is sent to these three domains. For details of specific matching rules, refer to the description of the `patterns` field in [Configuration Description](#configuration-description).

3. 填寫實驗資訊，並指定實驗範圍和排程的實驗持續時間：

   ![實驗資訊](./img/exp-info.png)

4. 提交實驗資訊。

## 使用 YAML 檔案建立實驗

1. Write the experiment configuration to the `dnschaos.yaml` file:

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: DNSChaos
   metadata:
     name: dns-chaos-example
     namespace: chaos-mesh
   spec:
     action: random
     mode: all
     patterns:
       - google.com
       - chaos-mesh.*
       - github.?om
     selector:
       namespaces:
         - busybox
   ```

   This configuration can take effect for domains including `google.com`, `chaos-mesh.org`, and `github.com`, which means that an IP address will be returned when a DNS request is sent to these three domains. For specific matching rules, refer to the `patterns` description in [Configuration Description](#configuration-description).

2. 準備好設定檔後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f dnschaos.yaml
   ```

### 設定說明

| Parameter | Type | Description | Default value | Required | Example |
| :-- | :-- | :-- | :-- | :-- | :-- |
| `action` | string | Defines the behavior of DNS fault. Optional values: `random` or `error`. When the value is `random`, DNS service returns a random IP address; when the value is `error`, DNS service returns an error. | None | Yes | `random` or `error` |
| `patterns` | String array | Selects a domain template that matches faults. Placeholder `?` and wildcard are supported. `*` | [] | No | `google.com`, `chaos-mesh.org`, `github.com` |
| `mode` | string | Specifies the mode of the experiment. The mode options include `one` (selecting a random Pod), `all` (selecting all eligible Pods), `fixed` (selecting a specified number of eligible Pods), `fixed-percent` (selecting a specified percentage of Pods from the eligible Pods), and `random-max-percent` (selecting the maximum percentage of Pods from the eligible Pods). | None | Yes | `one` |
| `value` | string | Provides parameters for the `mode` configuration, depending on `mode`. For example, when `mode` is set to `fixed-percent`, `value` specifies the percentage of Pods. | None | No | `1` |
| `selector` | struct | Specifies the target Pod. For details, refer to [Define the Scope of Chaos Experiments](./define-chaos-experiment-scope.md). | None | Yes |  |

:::note

- 在 `patterns` 設定中，萬用字元必須位於字串的結尾。例如，`chaos-mes*.org.` 是無效的設定。

- 當未設定 `patterns` 時，故障會注入所有網域。

:::
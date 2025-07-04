---
title: Remote Cluster Management
---

## 遠端叢集簡介

Chaos Mesh 提供叢集範圍的 `RemoteCluster` 資源，協助您管理遠端 Kubernetes 叢集並注入故障。本文件說明如何建立 `RemoteCluster` 物件並使用它來注入故障。

:::note

`RemoteCluster` 目前處於早期階段。其配置和功能（例如配置遷移、版本管理和驗證）將持續改進。如果您遇到任何問題，請在 [chaos-mesh/chaos-mesh](https://github.com/chaos-mesh/chaos-mesh) 開啟 issue 回報。

:::

## 註冊遠端叢集

若要將遠端叢集註冊到當前已安裝 Chaos Mesh 的叢集中，您需要建立一個 `RemoteCluster` 資源。建立此資源後，必要的元件將會自動安裝在遠端叢集中。以下是一個 `RemoteCluster` 資源的範例：

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: RemoteCluster
metadata:
  name: cluster-xxx
spec:
  namespace: chaos-mesh
  version: 2.6.2
  kubeConfig:
    secretRef:
      name: remote-chaos-mesh.kubeconfig
      namespace: chaos-mesh
      key: kubeconfig
  # configOverride:
  #   dashboard:
  #     create: true
```

It will install the `chaos-mesh` helm chart with the `KUBECONFIG` provided in the `.spec.kubeConfig` field in the specified namespace.

### 欄位說明

| Parameter | Type | Description | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| namespace | string | Represent the namespace to install Chaos Mesh components in the remote cluster | None | Yes | chaos-mesh |
| version | string | The version of Chaos Mesh to install in the remote cluster | None | Yes | 2.6.2 |
| kubeConfig.secretRef.name | string | The name of the secret, which is used to store the kubeconfig of remote cluster. This kubeconfig will be used to install chaos-mesh components and inject errors | None | Yes | `remote-chaos-mesh.kubeconfig` |
| kubeConfig.secretRef.namespace | string | The name of the namespace of the kubeconfig secret. | None | Yes | `default` |
| kubeConfig.secretRef.key | string | The key of the kubeconfig in the secret. | None | Yes | `kubeconfig` |
| configOverride | string | Passing helm values during install or upgrade | None | No | `{"dashboard":{"create":true}}` |

## 在遠端叢集中注入錯誤

To inject the errors to a remote cluster using the registered `RemoteCluster`, you could use the `remoteCluster` field in the `.spec` of every chaos types. For example:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: burn-cpu
spec:
  remoteCluster: cluster-xxx
  mode: one
  selector:
    labelSelectors:
      'app.kubernetes.io/component': 'tikv'
  stressors:
    cpu:
      workers: 1
      load: 100
      options: ['--cpu 2', '--timeout 600', '--hdd 1']
  duration: '30s'
```

The Chaos Mesh will inject the errors to the remote cluster using the kubeconfig registered with the `RemoteCluster` named `cluster-xxx`. The corresponding `StressChaos` will be automatically created in the remote cluster, and the status is synchronized back to the current cluster, so that you can manage the chaos injection for multiple different clusters in a single kubernetes.
---
title: Uninstall Chaos Mesh
---

本文介紹如何解除安裝 Chaos Mesh，包括使用 Helm 解除安裝 Chaos Mesh 和手動解除安裝 Chaos Mesh。當您需要從 Kubernetes 叢集中徹底清除 Chaos Mesh 安裝時，手動清除也非常有幫助。

## 使用 Helm 解除安裝 Chaos Mesh

### 步驟 1：清理混沌實驗

解除安裝 Chaos Mesh 前，請確保所有混沌實驗均已刪除。您可透過執行以下命令列出混沌相關物件：

```shell
for i in $(kubectl api-resources | grep chaos-mesh | awk '{print $1}'); do kubectl get $i -A; done
```

確認所有混沌實驗均已刪除後，即可解除安裝 Chaos Mesh。

### 步驟 2：列出 Helm Release

您可透過執行以下命令查看已安裝的 Helm Release：

```shell
helm list -A
```

輸出結果應類似：

```text
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
chaos-mesh-playground   chaos-mesh      1               2021-12-01 22:58:18.037052401 +0800 CST deployed        chaos-mesh-2.1.0        2.1.0
```

這表示 Chaos Mesh 已以 Helm Release 名稱 `chaos-mesh-playground` 安裝在命名空間 `chaos-mesh` 中。此即為要解除安裝的目標 Release。

### 步驟 3：刪除 Helm Release

確定目標 Helm Release 後，可透過執行以下命令刪除：

```shell
helm uninstall chaos-mesh-playground -n chaos-mesh
```

### 步驟 4：移除 CRDs

`helm uninstall` 不會移除 CRDs，您需手動執行以下命令移除：

```shell
kubectl delete crd $(kubectl get crd | grep 'chaos-mesh.org' | awk '{print $1}')
```

## 手動解除安裝 Chaos Mesh

若您是透過 `install.sh` 腳本安裝 Chaos Mesh，或安裝後修改過配置與元件，或在解除安裝時遇到問題，以下步驟可協助您手動解除安裝 Chaos Mesh。

### 步驟 1：清理混沌實驗

解除安裝 Chaos Mesh 前，請確保所有混沌實驗均已刪除。您可透過執行以下命令列出混沌相關物件：

```shell
for i in $(kubectl api-resources | grep chaos-mesh | awk '{print $1}'); do kubectl get $i -A; done
```

確認所有混沌實驗均已刪除後，即可解除安裝 Chaos Mesh。

### 步驟 2：移除 Chaos Mesh 工作負載

Chaos Mesh 安裝後通常包含以下元件：

- A `Deployment` called `chaos-controller-manager`, it is the controller/reconciler for Chaos Mesh.

- A `DaemonSet` called `chaos-daemon`, it is the agent for Chaos Mesh on each Kubernetes worker node.

- A `Deployment` called `chaos-dashboard`, the WebUI for Chaos Mesh.

- A `Deployment` called `chaos-dns-server`, it is the DNS proxy server, only occurs with you enable the DNSChaos feature.

您應移除這些工作負載物件。

接著刪除其對應的 `Service`：

- chaos-daemon

- chaos-dashboard

- chaos-mesh-controller-manager

- chaos-mesh-dns-server

### 步驟 3：移除 Chaos Mesh RBAC 物件

Chaos Mesh 安裝後包含以下 RBAC 物件：

- ClusterRoleBinding
  - chaos-mesh-playground-chaos-controller-manager-cluster-level
  - chaos-mesh-playground-chaos-controller-manager-target-namespace
  - chaos-mesh-playground-chaos-dns-server-cluster-level
  - chaos-mesh-playground-chaos-dns-server-target-namespace

- ClusterRole
  - chaos-mesh-playground-chaos-controller-manager-cluster-level
  - chaos-mesh-playground-chaos-controller-manager-target-namespace
  - chaos-mesh-playground-chaos-dns-server
  - chaos-mesh-playground-chaos-dns-server-cluster-level

- RoleBinding
  - chaos-mesh-playground-chaos-controller-manager-control-plane
  - chaos-mesh-playground-chaos-dns-server-control-plane

- Role
  - chaos-mesh-playground-chaos-controller-manager-control-plane
  - chaos-mesh-playground-chaos-dns-server-control-plane

- ServiceAccount
  - chaos-controller-manager
  - chaos-daemon
  - chaos-dns-server

您應該移除這些 RBAC 物件。

### 步驟 4：移除 ConfigMap 和 Secret

Chaos Mesh 安裝了以下幾種 ConfigMap 和 Secret：

- ConfigMap
  - chaos-mesh
  - dns-server-config

- Secret
  - chaos-mesh-webhook-certs

您應該移除這些 ConfigMap 和 Secret 物件。

### 步驟 5：移除 Webhook

Chaos Mesh 安裝了以下幾種 Webhook：

- ValidatingWebhookConfigurations
  - chaos-mesh-validation
  - chaos-mesh-validate-auth

- MutatingWebhookConfigurations
  - chaos-mesh-mutation

您應該移除這些 Webhook。

### 步驟 6：移除 CRD

最後，您可以執行以下指令來移除 CRD：

```shell
kubectl delete crd $(kubectl get crd | grep 'chaos-mesh.org' | awk '{print $1}')
```
---
title: FAQs
---

import PickHelmVersion from '@site/src/components/PickHelmVersion'

### 如果尚未部署 Kubernetes 叢集，我是否可以使用 Chaos Mesh 建立混沌實驗？

不行。但您可以使用 [`chaosd`](https://github.com/chaos-mesh/chaosd/) 在沒有 Kubernetes 的情況下注入故障。

### 我已經部署了 Chaos Mesh 並成功建立了 PodChaos 實驗，但在建立 NetworkChaos/TimeChaos 實驗時仍然失敗。日誌顯示如下：

```console
2020-06-18T02:49:15.160Z ERROR controllers.TimeChaos failed to apply chaos on all pods {"reconciler": "timechaos", "error": "rpc error: code = Unavailable desc = connection error: desc = \"transport: Error while dialing dial tcp xx.xx.xx.xx:xxxx: connect: connection refused\""}
```

原因在於 `chaos-controller-manager` 無法連接到 `chaos-daemon`。您需要先檢查 Pod 網路及其[政策](https://kubernetes.io/docs/concepts/services-networking/network-policies/)。

如果一切正常，您可以透過以下方式使用 `hostNetwork` 參數解決此問題：

<PickHelmVersion>{`helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --version latest --set chaosDaemon.hostNetwork=true`}</PickHelmVersion>

參考：https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/#hostport-services-do-not-work

### 預設的 Google Cloud 管理員帳戶被禁止建立混沌實驗。如何修復？

預設的 Google Cloud 管理員帳戶無法通過 `AdmissionReview` 檢查。您需要建立管理員角色並將該角色指派給帳戶，以授予建立混沌實驗的權限。例如：

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: role-cluster-manager-pdmas
rules:
  - apiGroups: ['']
    resources: ['pods', 'namespaces']
    verbs: ['get', 'watch', 'list']
  - apiGroups:
      - chaos-mesh.org
    resources: ['*']
    verbs: ['get', 'list', 'watch', 'create', 'delete', 'patch', 'update']
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-manager-binding
  namespace: chaos-mesh
subjects:
  # Google Cloud user account
  - kind: User
    name: USER_ACCOUNT
roleRef:
  kind: ClusterRole
  name: role-cluster-manager-pdmas
  apiGroup: rbac.authorization.k8s.io
```

上述的 `USER_ACCOUNT` 應為您的 Google Cloud 使用者電子郵件。

### Daemon 拋出類似 `version 1.41 is too new. The maximum supported API version is 1.39` 的錯誤

這表示 Docker daemon 可接受的最大 API 版本為 `1.39`，但 `chaos-daemon` 中的客戶端預設使用 `1.41`。您可以選擇以下解決方案：

1. 將 Docker 升級至較新版本。

2. 在 Helm 安裝/升級時加入 `--set chaosDaemon.env.DOCKER_API_VERSION=1.39`。

## DNSChaos

### 嘗試在 OpenShift 執行 DNSChaos 時，授權問題導致程序受阻

若錯誤訊息類似以下內容：

```bash
Error creating: pods "chaos-dns-server-123aa56123-" is forbidden: unable to validate against any security context constraint: [spec.containers[0].securityContext.capabilities.add: Invalid value: "NET_BIND_SERVICE": capability may not be added]
```

您需要將 privileged Security Context Constraints (SCC) 新增至 `chaos-dns-server`。

```bash
oc adm policy add-scc-to-user privileged -n chaos-mesh -z chaos-dns-server
```

## 安裝

### 嘗試在 OpenShift 安裝 Chaos Mesh 時，授權問題導致安裝程序受阻

若錯誤訊息類似以下內容：

```bash
Error creating: pods "chaos-daemon-" is forbidden: unable
 to validate against any security context constraint: [spec.securityContext.hostNetwork:
 Invalid value: true: Host network is not allowed to be used spec.securityContext.hostPID:
 Invalid value: true: Host PID is not allowed to be used spec.securityContext.hostIPC:
 Invalid value: true: Host IPC is not allowed to be used securityContext.runAsUser:
 Invalid value: "hostPath": hostPath volumes are not allowed to be used spec.containers[0].securityContext.volumes[1]:
 Invalid value: true: Host network is not allowed to be used spec.containers[0].securityContext.containers[0].hostPort:
 Invalid value: 31767: Host ports are not allowed to be used spec.containers[0].securityContext.hostPID:
 Invalid value: true: Host PID is not allowed to be used spec.containers[0].securityContext.hostIPC:
......]
```

您需要將 privileged scc 新增至 default。

```bash
oc adm policy add-scc-to-user privileged -n chaos-mesh -z chaos-daemon
```

### 安裝 Chaos Mesh 失敗並顯示訊息：no matches for kind "CustomResourceDefinition" in version "apiextensions.k8s.io/v1"

此問題發生於 Kubernetes v1.15 或更早版本安裝 Chaos Mesh 時。我們預設使用 `apiextensions.k8s.io/v1`，但該版本於 2019-09-19 在 Kubernetes v1.16 引入。

在低於 v1.16 的 Kubernetes 安裝 Chaos Mesh 時，需遵循以下流程：

1. 透過 `https://mirrors.chaos-mesh.org/<chaos-mesh-version>/crd-v1beta1.yaml` 手動建立 CRD。

2. 新增 `--validate=false`。若未添加此配置，可能因 CRD 的破壞性變更導致相容性問題。例如：`kubectl create -f https://mirrors.chaos-mesh.org/v2.1.0/crd-v1beta1.yaml --validate=false`。

3. Use Helm to finish the rest process of installation, and append `--skip-crds` with `helm install` command.

我們建議參考 Kubernetes 的[版本偏差策略](https://kubernetes.io/releases/version-skew-policy/)來升級您的 Kubernetes 叢集。
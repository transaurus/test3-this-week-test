---
title: Manage User Permissions
---

import PickHelmVersion from '@site/src/components/PickHelmVersion'

本文件說明如何在 Chaos Mesh 中管理使用者權限，包括建立不同角色的使用者帳戶、綁定權限至使用者帳戶、管理令牌，以及啟用或停用權限驗證。

Chaos Mesh uses [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) to manage user permissions. To create, view and manage chaos experiments, users must have the appropriate permissions in the `apiGroups` of `chaos-mesh.org` to refer the resources of chaos experiments.

:::caution

Chaos Mesh 允許您停用權限驗證，請參閱[啟用或停用權限驗證](#enable-or-disable-permission-authentication)了解停用方式。

**請注意，我們不建議在生產環境中停用權限驗證。**

:::

## 建立使用者帳戶並綁定權限

您可以使用 Chaos Dashboard 協助建立使用者帳戶並綁定權限。當您存取儀表板時，會出現登入視窗。點擊 **Click here to generate** 連結：

![Dashboard Token Login 1](img/dashboard_login1.png)

點擊連結後會出現令牌產生器，如下所示：

![Dashboard Token Generator](img/token_helper.png)

建立使用者帳戶並綁定權限的步驟如下：

### 選擇權限範圍

若要讓帳戶具備叢集中所有混沌實驗的對應權限，請勾選 **Cluster scoped** 核取方塊。若在 **Namespace** 下拉選單中指定命名空間，該帳戶將僅具備指定命名空間內的權限。

總結來說有兩種選擇：

- `Cluster scoped`：帳戶具備叢集中所有混沌實驗的權限。

- `Namespace scoped`：帳戶僅具備指定命名空間內所有混沌實驗的權限。

### 選擇使用者角色

目前 Chaos Mesh 提供以下使用者角色：

- `Manager`：具備建立、檢視、更新及刪除混沌實驗的所有權限。

- `Viewer`：僅具備檢視混沌實驗的權限。

### 產生權限設定

定義權限範圍和使用者角色後，儀表板會在令牌產生器中顯示對應的 RBAC 設定。例如，具備 `default` 命名空間管理員權限的設定如下：

```yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  namespace: default
  name: account-default-manager-vfmot

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: role-default-manager-vfmot
rules:
  - apiGroups: ['']
    resources: ['pods', 'namespaces']
    verbs: ['get', 'watch', 'list']
  - apiGroups:
      - chaos-mesh.org
    resources: ['*']
    verbs: ['get', 'list', 'watch', 'create', 'delete', 'patch', 'update']

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-default-manager-vfmot
  namespace: default
subjects:
  - kind: ServiceAccount
    name: account-default-manager-vfmot
    namespace: default
roleRef:
  kind: Role
  name: role-default-manager-vfmot
  apiGroup: rbac.authorization.k8s.io
```

點擊設定區塊右上角的 **COPY** 複製 RBAC 設定，並將內容儲存為本機的 `rbac.yaml` 檔案。

### 建立使用者帳戶並綁定權限

在終端機中執行以下命令：

```bash
kubectl apply -f rbac.yaml
```

:::note

需確保執行 `kubectl` 的本機使用者具備叢集操作權限，才能為其他使用者建立帳戶、綁定權限及產生令牌。

:::

### 取得令牌

:::info

Kubernetes v1.22 之前的版本會自動建立長期憑證以存取 Kubernetes API。在較新版本的 Kubernetes 中，您必須手動建立服務帳戶的令牌 Secret。

詳細資訊請參閱[手動為 ServiceAccount 建立 API 令牌](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-an-api-token-for-a-serviceaccount)。

:::

複製令牌產生器第三步顯示的命令，並在終端機中執行。以下是範例命令：

```bash
kubectl describe -n default secrets account-default-manager-vfmot
```

輸出結果如下：

```log
Name:         account-default-manager-vfmot-token-n4tg8
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: account-default-manager-vfmot
              kubernetes.io/service-account.uid: b71b3bf4-cd5e-4efb-8bf6-ff9a55fd7e07

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1111 bytes
namespace:  7 bytes
token:      eyJhbG...
```

複製底部的令牌，並在下一步登入時使用。

## 使用已建立的用戶帳戶登入 Chaos Dashboard

**關閉** Token 產生器。在 **Token** 欄位中輸入您於前一步驟取得的 token，並在 **Name** 欄位為該 token 輸入有意義的名稱。建議使用權限範圍與使用者角色的組合作為名稱，例如 `default-manager`。完成這兩個欄位後，點擊 **Submit** 登入：

![儀表板 Token 登入畫面 2](img/dashboard_login2.png)

:::info

若您尚未部署 Chaos Dashboard，也可自行產生 RBAC 設定，再使用 `kubectl` 建立用戶帳戶並綁定權限。

:::

## 登出 Chaos Dashboard

若需更換其他 token，請點擊儀表板左側邊欄的 **Settings**：

![儀表板 Token 登出畫面](img/token_logout.png)

在頁面頂端您會看到 **Logout** 按鈕。點擊即可登出目前帳戶。

## 常見問題

### 啟用或停用權限驗證

使用 Helm 安裝 Chaos Mesh 時，預設啟用權限驗證。在生產環境等高安全性情境中，建議保持權限驗證啟用。若您僅試用 Chaos Mesh 並想快速建立混沌實驗，可在 Helm 指令中設定 `--set dashboard.securityMode=false` 停用驗證。指令範例如下：

<PickHelmVersion>
{`helm upgrade chaos-mesh chaos-mesh/chaos-mesh --namespace=chaos-mesh --version latest --set dashboard.securityMode=false`}
</PickHelmVersion>

若需重新啟用權限驗證，則在 Helm 指令中重設 `--set dashboard.securityMode=true`。
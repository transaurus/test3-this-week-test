---
title: Manage User Permissions
---

import PickHelmVersion from '@site/src/components/PickHelmVersion'

本文說明如何在 Chaos Mesh 中管理使用者權限，包括建立不同角色的使用者帳號、綁定權限、管理權杖，以及啟用或停用權限驗證。

Chaos Mesh uses [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) to manage user permissions. To create, view and manage chaos experiments, users must have the appropriate permissions in the `apiGroups` of `chaos-mesh.org` to refer the resources of chaos experiments.

:::caution

Chaos Mesh 允許您停用權限驗證，詳見[啟用或停用權限驗證](#enable-or-disable-permission-authentication)了解停用方法。

**請注意，我們不建議在生產環境中停用權限驗證。**

:::

## 建立使用者帳號並綁定權限

您可使用 Chaos Dashboard 協助建立使用者帳號並綁定權限。存取儀表板時會出現登入視窗，點擊 **Click here to generate** 連結：

![Dashboard Token Login 1](img/dashboard_login1.png)

點擊連結後將顯示權杖產生器，如下所示：

![Dashboard Token Generator](img/token_helper.png)

建立使用者帳號並綁定權限的步驟如下：

### 選擇權限範圍

若要賦予帳號叢集內所有混沌實驗的完整權限，請勾選 **Cluster scoped** 核取方塊。若在 **Namespace** 下拉選單指定命名空間，帳號將僅擁有該命名空間內的權限。

總結有兩種選項：

- `Cluster scoped`：帳號擁有叢集內所有混沌實驗的權限。

- `Namespace scoped`：帳號僅擁有指定命名空間內混沌實驗的權限。

### 選擇使用者角色

目前 Chaos Mesh 提供以下使用者角色：

- `Manager`：擁有建立、檢視、更新及刪除混沌實驗的完整權限。

- `Viewer`：僅擁有檢視混沌實驗的權限。

### 產生權限設定

定義權限範圍和使用者角色後，儀表板將在權杖產生器中顯示對應的 RBAC 設定。例如，針對 `default` 命名空間管理員的權限設定如下：

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

點擊設定區塊右上角的 **COPY** 複製 RBAC 設定，並將內容儲存為本機 `rbac.yaml` 檔案。

### 建立使用者帳號並綁定權限

在終端機執行以下指令：

```bash
kubectl apply -f rbac.yaml
```

:::note

需確保執行 `kubectl` 的本機使用者擁有叢集權限，才能為其他使用者建立帳號、綁定權限及產生權杖。

:::

### 取得權杖

:::info

Kubernetes v1.22 之前的版本會自動建立長期憑證用於存取 Kubernetes API。近期 Kubernetes 版本中，您必須手動建立服務帳號權杖 Secret。

詳細說明請參閱[手動建立 ServiceAccount 的 API 權杖](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-an-api-token-for-a-serviceaccount)。

:::

複製權杖產生器中第三步顯示的指令，在終端機執行。範例指令如下：

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

複製底部的權杖，於後續步驟中用於登入。

## 使用您創建的用戶帳號登入 Chaos Dashboard

**關閉** Token 生成器。在 **Token** 欄位中輸入您在上一步獲取的 token，並在 **Name** 欄位中為該 token 輸入一個有意義的名稱。建議採用權限範圍和用戶角色的組合名稱，例如 `default-manager`。完成這兩個欄位後，點擊 **Submit** 登入：

![Dashboard Token Login 2](img/dashboard_login2.png)

:::info

若尚未部署 Chaos Dashboard，您也可以自行生成 RBAC 配置，然後使用 `kubectl` 創建用戶帳號並綁定權限。

:::

## 登出 Chaos Dashboard

如需更換其他 token，請點擊 Dashboard 左側導覽欄的 **Settings**：

![Dashboard Token Logout](img/token_logout.png)

頁面頂部將顯示 **Logout** 按鈕。點擊該按鈕即可登出當前帳號。

## 常見問題

### 啟用或停用權限驗證

使用 Helm 安裝 Chaos Mesh 時，預設啟用權限驗證。對於生產環境等高安全性場景，建議保持啟用狀態。若僅需快速試用 Chaos Mesh 創建混沌實驗，可在 Helm 命令中設定 `--set dashboard.securityMode=false` 停用驗證。命令範例如下：

<PickHelmVersion>
{`helm upgrade chaos-mesh chaos-mesh/chaos-mesh --namespace=chaos-mesh --version latest --set dashboard.securityMode=false`}
</PickHelmVersion>

如需重新啟用權限驗證，請在 Helm 命令中重置為 `--set dashboard.securityMode=true`。
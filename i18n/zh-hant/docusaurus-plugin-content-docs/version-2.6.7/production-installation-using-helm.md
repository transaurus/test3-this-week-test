---
title: Install Chaos Mesh using Helm
---

import Tabs from '@theme/Tabs'

import TabItem from '@theme/TabItem'

import PickVersion from '@site/src/components/PickVersion'

import PickHelmVersion from '@site/src/components/PickHelmVersion'

import VerifyInstallation from './common/\_verify-installation.md'

import QuickRun from './common/\_quick-run.md'

本文件描述如何在生產環境中安裝 Chaos Mesh。

## 先決條件

安裝 Chaos Mesh 前，請確保您已在環境中安裝 [Helm](https://helm.sh/docs/intro/install/)。

若要檢查 Helm 是否已安裝，請執行以下指令：

```bash
helm version
```

預期輸出如下：

```bash
version.BuildInfo{Version:"v3.5.4", GitCommit:"1b5edb69df3d3a08df77c9902dc17af864ff05d1", GitTreeState:"dirty", GoVersion: "go1.16.3"}
```

若您的實際輸出與預期輸出相似，包含 `Version`、`GitCommit`、`GitTreeState` 和 `GoVersion`，則表示 Helm 已成功安裝。

:::note

本文件中，指令使用 Helm v3 來操作 Chaos Mesh。若您的環境使用 Helm v2，請參考 [Migrating Helm v2 to v3](https://helm.sh/docs/topics/v2_v3_migration/) 或將 Helm 版本修改為 v2 格式。

:::

## 使用 Helm 安裝 Chaos Mesh

### 步驟 1：新增 Chaos Mesh 儲存庫

將 Chaos Mesh 儲存庫新增至 Helm 儲存庫：

```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
```

### 步驟 2：檢視可安裝的 Chaos Mesh 版本

若要查看可安裝的圖表，請執行以下指令：

```bash
helm search repo chaos-mesh
```

:::note

上述指令將輸出圖表的最新版本。若您想安裝歷史版本，請執行以下指令以查看所有已發布版本：

```bash
helm search repo chaos-mesh -l
```

:::

上述指令完成後，您即可開始安裝 Chaos Mesh。

### 步驟 3：建立安裝 Chaos Mesh 的命名空間

建議在 `chaos-mesh` 命名空間下安裝 Chaos Mesh，或者您可以指定任何命名空間來安裝 Chaos Mesh：

```bash
kubectl create ns chaos-mesh
```

### 步驟 4：在不同環境中安裝 Chaos Mesh

:::note

在 Kubernetes v1.15（或更早版本）上安裝 Chaos Mesh 時，您需要手動安裝 CRD。更多資訊，請參閱 [FAQ](./faqs.md#failed-to-install-chaos-mesh-with-the-message-no-matches-for-kind-customresourcedefinition-in-version-apiextensionsk8siov1)。

:::

由於不同容器運行時的守護程式監聽不同的 socket 路徑，您需要在安裝期間設定適當的值。您可以根據不同環境執行以下安裝指令。

<!-- prettier-ignore -->

<Tabs defaultValue="docker" values={[
  {label: 'Docker', value: 'docker'},
  {label: 'Containerd', value: 'containerd'},
  {label: 'K3s', value: 'k3s'},
  {label: 'MicroK8s', value: 'microk8s'},
  {label: 'CRI-O', value: 'cri-o'}
]}>
  <TabItem value="docker">
    <PickHelmVersion>{`\# Default to /var/run/docker.sock\nhelm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --version latest`}</PickHelmVersion>
  </TabItem>
  <TabItem value="containerd">
    <PickHelmVersion>{`helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock --version latest`}</PickHelmVersion>
  </TabItem>
  <TabItem value="k3s">
    <PickHelmVersion>{`helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/k3s/containerd/containerd.sock --version latest`}</PickHelmVersion>
  </TabItem>
  <TabItem value="microk8s">
    <PickHelmVersion>{`helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/var/snap/microk8s/common/run/containerd.sock --version latest`}</PickHelmVersion>
  </TabItem>
  <TabItem value="cri-o">
    <PickHelmVersion>{`helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --set chaosDaemon.runtime=crio --set chaosDaemon.socketPath=/var/run/crio/crio.sock --version latest`}</PickHelmVersion>
  </TabItem>
</Tabs>

:::info

To install a specific version of Chaos Mesh, add the `--version x.y.z` parameter after `helm install`. For example, `helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --version 2.1.0`.

:::

:::tip

為確保高可用性，Chaos Mesh 預設開啟 `leader-election` 功能。若您不需要使用此功能，可以手動使用 `--set controllerManager.leaderElection.enabled=false` 將其停用。

> 若版本 `<2.6.1`，您仍需要設定 `--set controllerManager.replicaCount=1` 以將控制器管理器的副本數減少為一個。

:::

## 驗證安裝

<VerifyInstallation />

## 執行混沌實驗

<QuickRun />

## 升級 Chaos Mesh

若要升級 Chaos Mesh，請執行以下指令：

```bash
helm upgrade chaos-mesh chaos-mesh/chaos-mesh
```

:::info

To upgrade to a specific version of Chaos Mesh, add the `--version x.y.z` parameter after `helm upgrade`. For example, `helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --version 2.1.0`.

:::

:::note

若您在非 Docker 環境中升級 Chaos Mesh，則需按照[步驟 4：在不同環境中安裝 Chaos Mesh](#step-4-install-chaos-mesh-in-different-environments) 所述新增相應參數。

:::

若要修改配置，請根據需求設定不同值。例如執行以下指令以升級並解除安裝 `chaos-dashboard`：

<PickHelmVersion>{`helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --version latest --set dashboard.create=false`}</PickHelmVersion>

:::note

更多可設定值及其用法，請參閱 [all values](https://github.com/chaos-mesh/chaos-mesh/blob/master/helm/chaos-mesh/values.yaml)。

:::

:::caution

目前 Helm 升級過程中不會自動套用最新的 CustomResourceDefinition (CRD)，可能導致錯誤。為避免此情況，請手動套用最新 CRD：

<PickVersion>
{`curl -sSL https://mirrors.chaos-mesh.org/latest/crd.yaml | kubectl create -f -`}
</PickVersion>

:::

## 解除安裝 Chaos Mesh

若要解除安裝 Chaos Mesh，請執行以下指令：

```bash
helm uninstall chaos-mesh -n chaos-mesh
```

## 常見問題

### 如何安裝最新版本的 Chaos Mesh？

`helm/chaos-mesh/values.yaml` 檔案定義了最新版本（master 分支）的映像檔。若要安裝最新版 Chaos Mesh，請執行以下指令：

```bash
# Clone repository
git clone https://github.com/chaos-mesh/chaos-mesh.git
cd chaos-mesh

helm install chaos-mesh helm/chaos-mesh -n=chaos-mesh
```

### 如何停用安全模式？

安全模式允許停用 Chaos Mesh 儀表板的驗證功能，僅應用於非生產部署。安全模式預設**啟用**。若要停用安全模式，請在安裝或升級時將 `dashboard.securityMode` 設定為 `false`：

<PickHelmVersion>
{`helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --set dashboard.securityMode=false --version latest`}
</PickHelmVersion>

### 如何持久化 Chaos Dashboard 資料

Chaos Dashboard 預設使用 `SQLite` 作為資料庫引擎。若 [PV（持久卷）](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) 被停用，重啟後 Chaos Dashboard 的資料將會遺失。為避免資料遺失，請參閱 [持久化 Chaos Dashboard 資料](persistence-dashboard.md) 文件，啟用 PV 或設定 `MySQL` 和 `PostgreSQL` 作為資料庫引擎。
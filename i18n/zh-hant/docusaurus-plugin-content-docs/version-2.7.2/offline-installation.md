---
title: Install Chaos Mesh Offline
---

import PickVersion from '@site/src/components/PickVersion'

import PickHelmVersion from '@site/src/components/PickHelmVersion'

import VerifyInstallation from './common/\_verify-installation.md'

import QuickRun from './common/\_quick-run.md'

本文檔描述如何離線安裝 Chaos Mesh。

## 先決條件

在安裝 Chaos Mesh 之前，請確保離線環境中已安裝 Docker 並部署 Kubernetes 叢集。若環境尚未準備就緒，請參考以下文件安裝 Docker 並部署 Kubernetes 叢集：

- [Docker](https://www.docker.com/get-started)

- [Kubernetes](https://kubernetes.io/docs/setup/)

## 準備安裝檔案

在離線安裝 Chaos Mesh 前，您需要從具備外部網路連線的機器下載所有 Chaos Mesh 映像檔及儲存庫壓縮套件，並將下載的檔案複製到離線環境中。

### 指定版本號碼

在具備外部網路連線的機器上，將 Chaos Mesh 版本號碼設定為環境變數：

<PickVersion>
{`export CHAOS_MESH_VERSION=latest`}
</PickVersion>

### 下載 Chaos Mesh 映像檔

在連接到外部網路的機器上，使用已設定的版本號碼拉取映像檔：

```bash
docker pull ghcr.io/chaos-mesh/chaos-mesh:${CHAOS_MESH_VERSION}
docker pull ghcr.io/chaos-mesh/chaos-daemon:${CHAOS_MESH_VERSION}
docker pull ghcr.io/chaos-mesh/chaos-dashboard:${CHAOS_MESH_VERSION}
```

將映像檔儲存為 tar 套件包：

```bash
docker save ghcr.io/chaos-mesh/chaos-mesh:${CHAOS_MESH_VERSION} > image-chaos-mesh.tar
docker save ghcr.io/chaos-mesh/chaos-daemon:${CHAOS_MESH_VERSION} > image-chaos-daemon.tar
docker save ghcr.io/chaos-mesh/chaos-dashboard:${CHAOS_MESH_VERSION} > image-chaos-dashboard.tar
```

:::note

若要模擬 DNS 故障（例如使 DNS 回應返回隨機錯誤的 IP 位址），您需要拉取額外的 [`pingcap/coredns`](https://hub.docker.com/r/pingcap/coredns) 映像檔。

:::

### 下載 Chaos Mesh 儲存庫壓縮套件

在連接到外部網路的機器上，下載 Chaos Mesh 的 zip 壓縮套件：

<PickVersion isArchive replaced="refs/heads/master">
{`curl -fsSL -o chaos-mesh.zip https://github.com/chaos-mesh/chaos-mesh/archive/refs/heads/master.zip`}
</PickVersion>

### 複製檔案

下載所有安裝所需的檔案後，請將這些檔案複製到離線環境：

- `image-chaos-mesh.tar`

- `image-chaos-daemon.tar`

- `image-chaos-dashboard.tar`

- `chaos-mesh.zip`

## 安裝 Chaos Mesh

將 Chaos Mesh 映像檔的 tar 套件包和儲存庫的 zip 壓縮套件複製到離線環境後，請依照以下步驟安裝 Chaos Mesh。

### 步驟 1. 載入 Chaos Mesh 映像檔

從 tar 套件包載入映像檔：

```bash
docker load < image-chaos-mesh.tar
docker load < image-chaos-daemon.tar
docker load < image-chaos-dashboard.tar
```

### 步驟 2. 將映像檔推送至 Registry

:::note

在將映像檔推送至 Registry 前，請確保離線環境中已部署 Registry。若尚未部署 Registry，請參考 [Docker Registry](https://docs.docker.com/registry/) 的部署方法。

:::

將 Chaos Mesh 版本和 Registry 位址設定為環境變數：

<PickVersion>
{`export CHAOS_MESH_VERSION=latest; export DOCKER_REGISTRY=localhost:5000`}
</PickVersion>

標記映像檔使其指向 Registry：

```bash
export CHAOS_MESH_IMAGE=$DOCKER_REGISTRY/chaos-mesh/chaos-mesh:${CHAOS_MESH_VERSION}
export CHAOS_DAEMON_IMAGE=$DOCKER_REGISTRY/chaos-mesh/chaos-daemon:${CHAOS_MESH_VERSION}
export CHAOS_DASHBOARD_IMAGE=$DOCKER_REGISTRY/chaos-mesh/chaos-dashboard:${CHAOS_MESH_VERSION}
docker image tag ghcr.io/chaos-mesh/chaos-mesh:${CHAOS_MESH_VERSION} $CHAOS_MESH_IMAGE
docker image tag ghcr.io/chaos-mesh/chaos-daemon:${CHAOS_MESH_VERSION} $CHAOS_DAEMON_IMAGE
docker image tag ghcr.io/chaos-mesh/chaos-dashboard:${CHAOS_MESH_VERSION} $CHAOS_DASHBOARD_IMAGE
```

將映像檔推送至 Registry：

```bash
docker push $CHAOS_MESH_IMAGE
docker push $CHAOS_DAEMON_IMAGE
docker push $CHAOS_DASHBOARD_IMAGE
```

### 步驟 3. 使用 Helm 安裝 Chaos Mesh

解壓縮 Chaos Mesh 的 zip 壓縮套件：

```bash
unzip chaos-mesh.zip -d chaos-mesh && cd chaos-mesh
```

:::note

在 Kubernetes v1.15（或更早版本）上安裝 Chaos Mesh 時，需先使用 `kubectl create -f manifests/crd-v1beta1.yaml` 手動安裝 CRD。詳細資訊請參閱 [FAQ](./faqs.md#failed-to-install-chaos-mesh-with-the-message-no-matches-for-kind-customresourcedefinition-in-version-apiextensionsk8siov1)。

:::

建立命名空間：

```bash
kubectl create ns chaos-mesh
```

執行安裝指令。執行安裝指令時，您需要指定 Chaos Mesh 的命名空間以及每個元件的映像檔值：

```bash
helm install chaos-mesh helm/chaos-mesh -n=chaos-mesh --set images.registry=$DOCKER_REGISTRY
```

## 驗證安裝

<VerifyInstallation />

## 執行混沌實驗

<QuickRun />
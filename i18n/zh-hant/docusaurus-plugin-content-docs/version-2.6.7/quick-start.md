---
title: Quick Start
---

import Tabs from '@theme/Tabs'

import TabItem from '@theme/TabItem'

import PickVersion from '@site/src/components/PickVersion'

import VerifyInstallation from './common/\_verify-installation.md'

import QuickRun from './common/\_quick-run.md'

本文件說明如何在測試或本地環境中快速啟動 Chaos Mesh。

:::caution

**本文檔僅適用於透過腳本快速試用 Chaos Mesh 的安裝方式。**

若需在生產環境或其他嚴格非測試場景中安裝 Chaos Mesh，建議使用 [Helm](https://helm.sh/)。詳細資訊請參閱[使用 Helm 安裝](production-installation-using-helm.md)。

:::

## 環境準備

試用前請確保環境中已部署 Kubernetes 叢集。若尚未部署，可參考以下連結完成部署：

- [Kubernetes](https://kubernetes.io/docs/setup/)

- [minikube](https://minikube.sigs.k8s.io/docs/start/)

- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

- [K3s](https://rancher.com/docs/k3s/latest/en/quick-start/)

- [Microk8s](https://microk8s.io/)

## 快速安裝

在測試環境中安裝 Chaos Mesh，請執行以下腳本：

<!-- prettier-ignore -->

<Tabs defaultValue="k8s" values={[
  {label: 'K8s', value: 'k8s'},
  {label: 'kind', value: 'kind'},
  {label: 'K3s', value: 'k3s'},
  {label: 'MicroK8s', value: 'microk8s'}
]}>
  <TabItem value="k8s">
    <PickVersion>
    {`curl -sSL https://mirrors.chaos-mesh.org/latest/install.sh | bash`}
    </PickVersion>
  </TabItem>
  <TabItem value="kind">
    <PickVersion>{`curl -sSL https://mirrors.chaos-mesh.org/latest/install.sh | bash -s -- --local kind`}</PickVersion>

    If you want to specify a `kind` version, add the `--kind-version xx` parameter at the end of the script, for example:

    <PickVersion>
    {`curl -sSL https://mirrors.chaos-mesh.org/latest/install.sh | bash -s -- --local kind --kind-version v0.20.0`}
    </PickVersion>

  </TabItem>
  <TabItem value="k3s">
    <PickVersion>
    {`curl -sSL https://mirrors.chaos-mesh.org/latest/install.sh | bash -s -- --k3s`}
    </PickVersion>
  </TabItem>
  <TabItem value="microk8s">
    <PickVersion>
    {`curl -sSL https://mirrors.chaos-mesh.org/latest/install.sh | bash -s -- --microk8s`}
    </PickVersion>
  </TabItem>
</Tabs>

執行後，Chaos Mesh 將自動安裝相應版本的 CustomResourceDefinitions 及必要元件。

更多安裝細節請參閱 [`install.sh`](https://github.com/chaos-mesh/chaos-mesh/blob/master/install.sh) 原始碼。

## 驗證安裝

<VerifyInstallation />

## 執行混沌實驗

<QuickRun />

## 解除安裝 Chaos Mesh

解除安裝 Chaos Mesh 請執行以下命令：

<PickVersion>
{`curl -sSL https://mirrors.chaos-mesh.org/latest/install.sh | bash -s -- --template | kubectl delete -f -`}
</PickVersion>

亦可直接刪除 `chaos-mesh` 命名空間以解除安裝：

```sh
kubectl delete ns chaos-mesh
```

## 常見問題

### 為何安裝後根目錄會出現 `local` 目錄？

If you don't install `kind` in the existing environment, and you use the `--local kind` parameter when executing the installation command, the `install.sh` script will automatically install the `kind` in the `local` directory under the root directory.
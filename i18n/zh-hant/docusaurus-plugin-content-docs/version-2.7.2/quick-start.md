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

**本文件僅描述透過腳本快速試用 Chaos Mesh 的安裝方式。**

若需在生產環境或其他嚴格的非測試情境下安裝 Chaos Mesh，建議使用 [Helm](https://helm.sh/)。更多詳細資訊，請參閱 [使用 Helm 安裝](production-installation-using-helm.md)。

:::

## 環境準備

在試用前，請確保環境中已部署 Kubernetes 叢集。若尚未部署 Kubernetes 叢集，可參考以下連結完成部署：

- [Kubernetes](https://kubernetes.io/docs/setup/)

- [minikube](https://minikube.sigs.k8s.io/docs/start/)

- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

- [K3s](https://rancher.com/docs/k3s/latest/en/quick-start/)

- [Microk8s](https://microk8s.io/)

## 快速安裝

若要在測試環境中安裝 Chaos Mesh，請執行以下腳本：

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

執行後，Chaos Mesh 會自動安裝適當版本的 CustomResourceDefinitions 和必要元件。

更多安裝細節，請參考 [`install.sh`](https://github.com/chaos-mesh/chaos-mesh/blob/master/install.sh) 的原始碼。

## 驗證安裝

<VerifyInstallation />

## 執行 Chaos 實驗

<QuickRun />

## 解除安裝 Chaos Mesh

若要解除安裝 Chaos Mesh，請執行以下命令：

<PickVersion>
{`curl -sSL https://mirrors.chaos-mesh.org/latest/install.sh | bash -s -- --template | kubectl delete -f -`}
</PickVersion>

您也可以刪除 `chaos-mesh` 命名空間來直接解除安裝 Chaos Mesh：

```sh
kubectl delete ns chaos-mesh
```

## 常見問題

### 為什麼安裝後根目錄會出現 `local` 目錄？

若在現有環境中未安裝 `kind`，並且在執行安裝命令時使用了 `--local kind` 參數，則 `install.sh` 腳本會自動將 `kind` 安裝在根目錄下的 `local` 目錄中。
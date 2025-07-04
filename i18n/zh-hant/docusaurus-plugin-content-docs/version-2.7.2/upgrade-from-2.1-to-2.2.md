---
title: Upgrade from 2.1 to 2.2
---

Helm Charts 2.2.0 版本中有多項變更。本文檔介紹如何從 2.1.x 遷移到 2.2.0。

## 使用 Helm 升級

### 步驟 1：新增/更新 Chaos Mesh Helm 儲存庫

將 Chaos Mesh 儲存庫新增至 Helm 儲存庫並更新：

```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update
```

### 步驟 2：遷移 `values.yaml` 文件

若您先前使用特定 `values.yaml` 安裝 Chaos Mesh，建議將自訂配置套用至 Chaos Mesh 2.2.0 的 `values.yaml`。

可透過以下命令取得預設 `values.yaml`：

```bash
helm show values chaos-mesh/chaos-mesh --version 2.2.0 > values.yaml
```

若您不熟悉變更的配置項，可能表示未依賴該功能，通常可安全忽略這些變更。

以下是 Helm Chart 的變更清單：

- 新增配置：`chaosDaemon.mtls.enabled` 表示在 chaos-controller-manager 與 chaos-daemon 間啟用 mtls

- 新增配置：`webhook.caBundlePEM` 表示用於 webhook 伺服器的 CA 憑證包

- 值變更：`dashboard.serviceAccount` 從 `chaos-controller-manager` 改為 `chaos-dashboard`

- 值變更：`webhook.FailurePolicy` 從 `Ignore` 改為 `Fail`

:::note

詳細說明請參閱 [README](https://github.com/chaos-mesh/chaos-mesh/blob/v2.2.0/helm/chaos-mesh/README.md)。

:::

### 步驟 3：更新 CRD

若 `Kubernetes >= 1.16`，請執行以下命令套用最新 CRD：

```bash
kubectl replace -f https://mirrors.chaos-mesh.org/v2.2.0/crd.yaml
```

若 `Kubernetes <= 1.15`，請執行以下命令套用最新 CRD：

```bash
kubectl replace -f https://mirrors.chaos-mesh.org/v2.2.0/crd-v1beta1.yaml
```

:::caution

Chaos Mesh 2.2.x 將是最後支援 Kubernetes < 1.19 的版本系列。

:::

### 步驟 4：透過 `helm upgrade` 升級 Chaos Mesh

執行以下命令將 Chaos Mesh 升級至 2.2.0：

```bash
helm upgrade <release-name> chaos-mesh/chaos-mesh --namespace=<namespace> --version=2.2.0 <--other-required-flags>
```

## 詢問社群

如有任何升級問題，歡迎透過 [Slack 頻道](https://cloud-native.slack.com/archives/C0193VAV272)、GitHub [議題](https://github.com/chaos-mesh/chaos-mesh/issues/new?assignees=&labels=&template=question.md) 或 [討論區](https://github.com/chaos-mesh/chaos-mesh/discussions/new) 聯繫我們。
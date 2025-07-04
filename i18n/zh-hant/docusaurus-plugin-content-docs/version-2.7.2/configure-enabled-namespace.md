---
title: Configure namespace for Chaos experiments
---

import PickHelmVersion from '@site/src/components/PickHelmVersion'

本章將引導您如何設定 Chaos 實驗，使其僅在指定的命名空間中生效，並保護其他未指定的命名空間免受故障注入影響。

## 控制 Chaos 實驗生效範圍

Chaos Mesh 提供兩種方式控制 Chaos 實驗的生效範圍：

- 若要將 Chaos 實驗限制在指定命名空間生效，需啟用 FilterNamespace 功能（預設關閉）。此功能具有全域作用範圍，啟用後需在允許執行 Chaos 實驗的命名空間添加註釋，未添加註釋的命名空間將受到保護。

- 若要為單一 Chaos 實驗指定生效範圍，請參閱[定義 Chaos 實驗範圍](define-chaos-experiment-scope.md)。

## 啟用 FilterNamespace

若尚未安裝 Chaos Mesh，可在 Helm 安裝時透過添加 `--set controllerManager.enableFilterNamespace=true` 參數啟用此功能。以下為 Docker 容器內的指令範例：

<PickHelmVersion>{`helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --set controllerManager.enableFilterNamespace=true --version latest`}</PickHelmVersion>

:::note

使用 Helm 安裝時，不同容器的指令與參數可能有所差異。詳細資訊請參閱[使用 Helm 安裝 Chaos Mesh](production-installation-using-helm.md)。

:::

若已透過 Helm 安裝 Chaos Mesh，可使用 `helm upgrade` 指令升級配置來啟用此功能。例如：

<PickHelmVersion>{`helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --version latest --set controllerManager.enableFilterNamespace=true`}</PickHelmVersion>

在 `helm upgrade` 指令中，可透過多個 `--set` 參數設定多個值，後續設定將覆蓋先前設定。例如若指令中包含 `--set controllerManager.enableFilterNamespace=false -set controllerManager.enableFilterNamespace=true`，最終仍會啟用此功能。

您也可使用 `-f` 參數指定描述配置的 YAML 檔案。詳見 [Helm 升級說明](https://helm.sh/zh/docs/helm/helm_upgrade/#%E7%AE%80%E4%BB%8B)。

## 為命名空間添加 Chaos 實驗註釋

啟用 FilterNamespace 後，Chaos Mesh 僅會對包含 `chaos-mesh.org/inject=enabled` 註釋的命名空間注入故障。因此執行 Chaos 實驗前，需在允許實驗生效的命名空間添加此註釋，未添加的命名空間將受到保護。

You can add the annotation for a `namespace` using the following `kubectl` command:

```bash
kubectl annotate ns $NAMESPACE chaos-mesh.org/inject=enabled
```

上述指令中的 `$NAMESPACE` 指命名空間名稱（例如 `default`）。若註釋添加成功，輸出如下：

```bash
namespace/$NAMESPACE annotated
```

若要刪除此註釋，可使用以下指令：

```bash
kubectl annotate ns $NAMESPACE chaos-mesh.org/inject-
```

若註釋刪除成功，輸出如下：

```bash
namespace/$NAMESPACE annotated
```

## 檢視所有生效的命名空間

可透過以下指令列出所有允許執行 Chaos 實驗的命名空間：

```bash
kubectl get ns -o jsonpath='{.items[?(@.metadata.annotations.chaos-mesh\.org/inject=="enabled")].metadata.name}'
```

此指令將輸出所有包含 `chaos-mesh.org/inject=enabled` 註釋的命名空間。例如：

```bash
default
```
---
title: Configure namespace for Chaos experiments
---

import PickHelmVersion from '@site/src/components/PickHelmVersion'

本章節將引導您如何設定混沌實驗僅在指定命名空間生效，並保護其他未指定的命名空間免受故障注入影響。

## 控制混沌實驗生效範圍

Chaos Mesh 提供兩種方式控制混沌實驗生效範圍：

- 若要設定混沌實驗僅在指定命名空間生效，需啟用 FilterNamespace 功能（預設關閉）。此功能在全局範圍生效，啟用後需在允許混沌實驗生效的命名空間添加註釋，未添加註釋的命名空間將受到保護，避免故障注入。

- 若要為單個混沌實驗指定生效範圍，請參閱[定義混沌實驗範圍](define-chaos-experiment-scope.md)。

## 啟用 FilterNamespace 功能

若尚未安裝 Chaos Mesh，可在安裝時透過 Helm 指令添加 `--set controllerManager.enableFilterNamespace=true` 啟用此功能。以下是在 Docker 容器中的指令範例：

<PickHelmVersion>{`helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --set controllerManager.enableFilterNamespace=true --version latest`}</PickHelmVersion>

:::note

使用 Helm 安裝時，不同容器的指令與參數可能不同。詳情請參閱[使用 Helm 安裝 Chaos Mesh](production-installation-using-helm.md)。

:::

若已使用 Helm 安裝 Chaos Mesh，可透過 `helm upgrade` 指令升級配置以啟用此功能。例如：

<PickHelmVersion>{`helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --version latest --set controllerManager.enableFilterNamespace=true`}</PickHelmVersion>

For `helm upgrade`, you can set multiple parameters by adding multiple `--set` in the command. Later settings override previous settings. For example, if you add `--set controllerManager.enableFilterNamespace=false -set controllerManager.enableFilterNamespace=true` in the command, it still enables this feature.

也可透過 `-f` 參數指定描述配置的 YAML 文件。詳情請參閱 [Helm 升級說明](https://helm.sh/zh/docs/helm/helm_upgrade/#%E7%AE%80%E4%BB%8B)。

## 為命名空間添加混沌實驗註釋

啟用 FilterNamespace 後，Chaos Mesh 僅會對包含 `chaos-mesh.org/inject=enabled` 註釋的命名空間注入故障。因此啟動混沌實驗前，需在允許實驗生效的命名空間添加此註釋，其餘命名空間將受到保護，避免故障注入。

You can add the annotation for a `namespace` using the following `kubectl` command:

```bash
kubectl annotate ns $NAMESPACE chaos-mesh.org/inject=enabled
```

上述指令中的 `$NAMESPACE` 為命名空間名稱（例如 `default`）。若註釋添加成功，輸出如下：

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

## 檢視所有允許混沌實驗的命名空間

可透過以下指令列出所有允許混沌實驗的命名空間：

```bash
kubectl get ns -o jsonpath='{.items[?(@.metadata.annotations.chaos-mesh\.org/inject=="enabled")].metadata.name}'
```

此指令將輸出所有包含 `chaos-mesh.org/inject=enabled` 註釋的命名空間。例如：

```bash
default
```
---
title: Configure namespace for Chaos experiments
---

import PickHelmVersion from '@site/src/components/PickHelmVersion'

本章將引導您設定混沌實驗，使其僅在指定的命名空間中生效，並保護其他未指定的命名空間免受故障注入影響。

## 控制混沌實驗生效範圍

Chaos Mesh 提供兩種方式控制混沌實驗生效範圍：

- 若要設定混沌實驗僅在指定命名空間生效，需啟用 FilterNamespace 功能（預設關閉）。此功能具全局效力，啟用後需在允許混沌實驗的命名空間添加註解，未添加註解的命名空間將受到保護，免於故障注入。

- 若要為單個混沌實驗指定生效範圍，請參閱[定義混沌實驗範圍](define-chaos-experiment-scope.md)。

## 啟用 FilterNamespace

若尚未安裝 Chaos Mesh，可在 Helm 安裝時透過指令添加 `--set controllerManager.enableFilterNamespace=true` 啟用此功能。以下為 Docker 容器中的指令範例：

<PickHelmVersion>{`helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --set controllerManager.enableFilterNamespace=true --version latest`}</PickHelmVersion>

:::note

使用 Helm 安裝時，指令與參數因容器而異。詳見[使用 Helm 安裝 Chaos Mesh](production-installation-using-helm.md)。

:::

若已透過 Helm 安裝 Chaos Mesh，可透過 `helm upgrade` 指令升級設定啟用此功能。例如：

<PickHelmVersion>{`helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --version latest --set controllerManager.enableFilterNamespace=true`}</PickHelmVersion>

`helm upgrade` 可透過多次添加 `--set` 設定多參數，後續設定會覆蓋先前設定。例如若指令包含 `--set controllerManager.enableFilterNamespace=false -set controllerManager.enableFilterNamespace=true`，最終仍會啟用此功能。

亦可使用 `-f` 參數指定 YAML 檔案描述設定。詳見 [Helm upgrade](https://helm.sh/zh/docs/helm/helm_upgrade/#%E7%AE%80%E4%BB%8B)。

## 為命名空間添加混沌實驗註解

啟用 FilterNamespace 後，Chaos Mesh 僅會將故障注入含註解 `chaos-mesh.org/inject=enabled` 的命名空間。因此開始混沌實驗前，需在允許實驗生效的命名空間添加此註解，其他命名空間將受保護免於故障注入。

You can add the annotation for a `namespace` using the following `kubectl` command:

```bash
kubectl annotate ns $NAMESPACE chaos-mesh.org/inject=enabled
```

上述指令中 `$NAMESPACE` 為命名空間名稱（如 `default`）。若註解添加成功，輸出如下：

```bash
namespace/$NAMESPACE annotated
```

若要刪除此註解，可使用以下指令：

```bash
kubectl annotate ns $NAMESPACE chaos-mesh.org/inject-
```

若註解成功刪除，輸出如下：

```bash
namespace/$NAMESPACE annotated
```

## 檢查混沌實驗生效的所有命名空間

可透過以下指令列出所有允許混沌實驗的命名空間：

```bash
kubectl get ns -o jsonpath='{.items[?(@.metadata.annotations.chaos-mesh\.org/inject=="enabled")].metadata.name}'
```

此指令將輸出所有含 `chaos-mesh.org/inject=enabled` 註解的命名空間。例如：

```bash
default
```
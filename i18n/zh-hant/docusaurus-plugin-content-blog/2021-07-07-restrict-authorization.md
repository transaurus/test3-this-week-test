---
slug: /securing-tenant-namespaces-using-restrict-authorization-feature
title: 'Securing tenant namespaces using restrict authorization feature in Chaos Mesh'
authors: anuragpaliwal
image: /img/blog/chaos-engineering-tools-as-a-service.jpeg
tags: [Chaos Mesh, Chaos Engineering]
---

![混沌工程工具](/img/blog/chaos-mesh-restrict-authorization.jpeg)

[多租戶](https://cloud.google.com/kubernetes-engine/docs/concepts/multitenancy-overview)叢集由多個用戶和/或工作負載（稱為「租戶」）共享。多租戶叢集的運營者必須隔離各租戶，以最小化受損或惡意租戶對叢集和其他租戶造成的損害。

<!--truncate-->

## 叢集多租戶架構

規劃多租戶架構時，應考量 Kubernetes 中的資源隔離層級：叢集、命名空間、節點、Pod 和容器。

儘管 Kubernetes 無法保證租戶間的完美安全隔離，但它提供了可能滿足特定用例的功能。您可以將每個租戶及其 Kubernetes 資源分離至各自的[命名空間](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)。Kubernetes 支援由同一實體叢集託管的多個虛擬叢集，這些虛擬叢集稱為命名空間。[命名空間](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)適用於多團隊或多專案環境中眾多用戶的使用場景。

## 部署 Chaos Mesh 的叢集

您設計的 Kubernetes 叢集包含多租戶服務，並遵循 Kubernetes 最佳安全實踐：每個租戶服務運行在專屬命名空間中，這些租戶服務的用戶僅擁有相應命名空間的適當存取權限等。

<!--truncate-->

您在叢集上啟用了 Chaos Mesh（[Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh) 是雲原生的混沌工程平台，可在 Kubernetes 環境中編排混沌實驗），使租戶服務能執行各類混沌活動以確保其應用/系統的韌性。同時，您已透過 [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) 授予租戶服務用戶管理 Chaos Mesh 資源的特定權限。

<!--truncate-->

假設某租戶用戶欲在其命名空間（例如 chaos-mesh）執行 Pod 終止操作，該用戶創建了以下 Chaos Mesh YAML 檔案：

```yml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill
  namespace: chaos-mesh
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - tidb-cluster-demo
    labelSelectors:
      'app.kubernetes.io/component': 'tikv'
  scheduler:
    cron: '@every 1m'
```

該用戶擁有 chaos-mesh 命名空間的必要權限，但對 tidb-cluster-demo 命名空間無權限。當用戶使用 kubectl 應用此 YAML 檔案時，將在 chaos-mesh 命名空間創建 pod-kill 混沌資源。如選擇器（selector）所示，用戶指定了其他命名空間（tidb-cluster-demo），意味著混沌操作將選取 tidb-cluster-demo 命名空間的 Pod，而非用戶有權存取的 chaos-mesh 命名空間。這表示該用戶能影響其無權存取的命名空間。**問題發生！**

<!--truncate-->

自 Chaos Mesh 1.1.3 版本起，此安全問題已透過限制授權功能修復。現在當用戶應用上述 YAML 檔案時，系統將顯示類似錯誤：

```yml
Error when creating "pod/pod-kill.yaml": admission webhook "vauth.kb.io" denied the request: ... is forbidden on namespace
tidb-cluster-demo
```

**問題已解決！**

請注意，若用戶同時擁有 tidb-cluster-demo 命名空間的必要權限，則不會出現此類錯誤。

## 更多教學資源

若需強制禁止用戶跨命名空間創建混沌實驗，請參閱我的先前的部落格文章：[使用 OPA 在 Chaos Mesh 中保護租戶服務](https://anuragpaliwal-93749.medium.com/securing-tenant-services-while-using-chaos-mesh-using-opa-3ae80c7f4b85)。

## 最後重點提醒

如果您對 Chaos Mesh 感興趣並想了解更多，歡迎加入 [Slack 頻道](https://slack.cncf.io/) 或提交您的 pull requests 及問題到其 [GitHub 儲存庫](https://github.com/chaos-mesh/chaos-mesh)。
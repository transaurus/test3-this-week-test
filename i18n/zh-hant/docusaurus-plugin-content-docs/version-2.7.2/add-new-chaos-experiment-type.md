---
title: Add a New Chaos Experiment Type
---

import PickHelmVersion from '@site/src/components/PickHelmVersion'

本文介紹如何新增一種混沌實驗類型。

以下將引導您完成一個 `HelloWorldChaos` 的範例，這是一種新的混沌實驗類型，它會在日誌中輸出 `Hello world!`。步驟包括：

- [步驟 1：定義 HelloWorldChaos 的結構](#step-1-define-the-schema-of-helloworldchaos)

- [步驟 2：註冊 CRD](#step-2-register-the-crd)

- [步驟 3：註冊 helloworldchaos 物件的事件處理器](#step-3-register-the-event-handler-for-helloworldchaos-objects)

- [步驟 4：建構 Docker 映像檔](#step-4-build-docker-images)

- [步驟 5：執行 HelloWorldChaos](#step-5-run-helloworldchaos)

## 步驟 1：定義 HelloWorldChaos 的結構

1. Add `helloworldchaos_types.go` to the `api/v1alpha1` API directory with the following content:

   ```go
   package v1alpha1

   import (
           metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
   )

   // +kubebuilder:object:root=true
   // +chaos-mesh:experiment
   // +chaos-mesh:oneshot=true

   // HelloWorldChaos is the Schema for the helloworldchaos API
   type HelloWorldChaos struct {
           metav1.TypeMeta   `json:",inline"`
           metav1.ObjectMeta `json:"metadata,omitempty"`

           Spec   HelloWorldChaosSpec   `json:"spec"`
           Status HelloWorldChaosStatus `json:"status,omitempty"`
   }

   // HelloWorldChaosSpec defines the desired state of HelloWorldChaos
   type HelloWorldChaosSpec struct {
           // ContainerSelector specifies the target for injection
           ContainerSelector `json:",inline"`

           // Duration represents the duration of the chaos
           // +optional
           Duration *string `json:"duration,omitempty"`

           // RemoteCluster represents the remote cluster where the chaos will be deployed
           // +optional
           RemoteCluster string `json:"remoteCluster,omitempty"`
   }

   // HelloWorldChaosStatus defines the observed state of HelloWorldChaos
   type HelloWorldChaosStatus struct {
           ChaosStatus `json:",inline"`
   }

   // GetSelectorSpecs is a getter for selectors
   func (obj *HelloWorldChaos) GetSelectorSpecs() map[string]interface{} {
           return map[string]interface{}{
                   ".": &obj.Spec.ContainerSelector,
           }
   }
   ```

   This file defines the schema type of `HelloWorldChaos`, which can be described in a YAML file:

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: HelloWorldChaos
   metadata:
     name: <resource name>
     namespace: <namespace>
   spec:
     duration: <duration>
   #...
   ```

## 步驟 2：註冊 CRD

您需要註冊 `HelloWorldChaos` 的 CRD（自訂資源定義）以透過 Kubernetes API 與其互動。

1. 為了將 CRD 合併到 manifests/crd.yaml 中，請將我們在上一步中產生的 `config/crd/bases/chaos-mesh.org_helloworldchaos.yaml` 附加到 `config/crd/kustomization.yaml`：

   ```yaml
   resources:
     - bases/chaos-mesh.org_podchaos.yaml
     - bases/chaos-mesh.org_networkchaos.yaml
     - bases/chaos-mesh.org_iochaos.yaml
     - bases/chaos-mesh.org_helloworldchaos.yaml # 此行新增
   ```

2. Run `make generate` in the root directory of Chaos Mesh, which generates a boilerplate of `HelloWorldChaos` for Chaos Mesh to compile:

   ```bash
   make generate
   ```

   Then you can see the definition of `HelloWorldChaos` in `manifests/crd.yaml`.

## 步驟 3：註冊 helloworldchaos 物件的事件處理器

1. 建立新檔案 `controllers/chaosimpl/helloworldchaos/types.go`，內容如下：

   ```go
   package helloworldchaos

   import (
           "context"

           "github.com/go-logr/logr"
           "go.uber.org/fx"
           "sigs.k8s.io/controller-runtime/pkg/client"

           "github.com/chaos-mesh/chaos-mesh/api/v1alpha1"
           impltypes "github.com/chaos-mesh/chaos-mesh/controllers/chaosimpl/types"
           "github.com/chaos-mesh/chaos-mesh/controllers/chaosimpl/utils"
   )

   var _ impltypes.ChaosImpl = (*Impl)(nil)

   type Impl struct {
           client.Client
           Log logr.Logger

           decoder *utils.ContainerRecordDecoder
   }

   // 此方法對應 HelloWorldChaos 的 Apply 階段，將觸發 HelloWorldChaos 的執行
   func (impl *Impl) Apply(ctx context.Context, index int, records []*v1alpha1.Record, obj v1alpha1.InnerObject) (v1alpha1.Phase, error) {
           impl.Log.Info("Hello world!")
           return v1alpha1.Injected, nil
   }

   // 此方法對應 HelloWorldChaos 的 Recover 階段，將觸發調解器恢復混沌動作
   func (impl *Impl) Recover(ctx context.Context, index int, records []*v1alpha1.Record, obj v1alpha1.InnerObject) (v1alpha1.Phase, error) {
           impl.Log.Info("Goodbye world!")
           return v1alpha1.NotInjected, nil
   }

   // NewImpl 返回新的 HelloWorldChaos 實作實例
   func NewImpl(c client.Client, log logr.Logger, decoder *utils.ContainerRecordDecoder) *impltypes.ChaosImplPair {
           return &impltypes.ChaosImplPair{
                   Name:   "helloworldchaos",
                   Object: &v1alpha1.HelloWorldChaos{},
                   Impl: &Impl{
                           Client:  c,
                           Log:     log.WithName("helloworldchaos"),
                           decoder: decoder,
                   },
                   ObjectList: &v1alpha1.HelloWorldChaosList{},
        }
   }

   var Module = fx.Provide(
            fx.Annotated{
                    Group:  "impl",
                    Target: NewImpl,
            },
   )
   ```

2. Chaos Mesh 使用 [fx](https://github.com/uber-go/fx) 函式庫進行依賴注入。要將 `HelloWorldChaos` 註冊到控制器管理器，請在 `controllers/chaosimpl/fx.go` 添加一行：

   ```go
   var AllImpl = fx.Options(
           gcpchaos.Module,
           stresschaos.Module,
           jvmchaos.Module,
           timechaos.Module,
           helloworldchaos.Module // 新增此行，請確保已先匯入 helloworldchaos
           //...
   )
   ```

   接著在 `controllers/types/types.go` 中，將以下內容追加至 `ChaosObjects`：

   ```go
   var ChaosObjects = fx.Supply(
          //...
          fx.Annotated{
                  Group: "objs",
                  Target: Object{
                          Name:   "helloworldchaos",
                          Object: &v1alpha1.HelloWorldChaos{},
                  },
          },
   )
   ```

## 步驟 4：建構 Docker 映像檔

1. 建構生產環境映像檔：

   ```bash
   make image
   ```

2. 若使用 minikube 部署 Kubernetes 集群，則需將映像檔載入集群：

   ```bash
   minikube image load ghcr.io/chaos-mesh/chaos-dashboard:latest
   minikube image load ghcr.io/chaos-mesh/chaos-mesh:latest
   minikube image load ghcr.io/chaos-mesh/chaos-daemon:latest
   ```

## 步驟 5：執行 HelloWorldChaos

在此步驟中，您需要部署包含最新變更的 Chaos Mesh 以測試 HelloWorldChaos。

1. 在您的集群中註冊 CRD：

   ```bash
   kubectl create -f manifests/crd.yaml
   ```

   從輸出中可以看到 `HelloWorldChaos` 已創建：

   ```log
   customresourcedefinition.apiextensions.k8s.io/helloworldchaos.chaos-mesh.org created
   ```

   現在您可以使用以下命令獲取 `HelloWorldChaos` 的 CRD：

   ```bash
   kubectl get crd helloworldchaos.chaos-mesh.org
   ```

2. 部署 Chaos Mesh：

   ```bash
   helm install chaos-mesh helm/chaos-mesh -n=chaos-mesh --set controllerManager.leaderElection.enabled=false,dashboard.securityMode=false
   ```

   若要驗證部署是否成功，可檢查 `chaos-mesh` 命名空間中的所有 Pod：

   ```bash
   kubectl get pods --namespace chaos-mesh -l app.kubernetes.io/instance=chaos-mesh
   ```

3. 部署測試用的 Deployment，我們可以使用 minikube 文檔中的範例 echo server：

   ```bash
   kubectl create deployment hello-minikube --image=kicbase/echo-server:1.0
   ```

   等待 Pod 運行：

   ```bash
   kubectl get pods
   ```

   範例輸出：

   ```log
   NAME                              READY   STATUS    RESTARTS   AGE
   hello-minikube-77b6f68484-dg4sw   1/1     Running   0          2m
   ```

4. 創建包含以下內容的 `hello.yaml` 文件：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: HelloWorldChaos
   metadata:
     name: hello-world
     namespace: chaos-mesh
   spec:
     selector:
       labelSelectors:
         app: hello-minikube
     mode: one
     duration: 1h
   ```

5. 執行：

   ```bash
   kubectl apply -f hello.yaml
   # helloworldchaos.chaos-mesh.org/hello-world created
   ```

   現在您可以檢查 `chaos-controller-manager` 的日誌中是否有 `Hello world!`：

   ```bash
   kubectl logs -n chaos-mesh chaos-controller-manager-xxx
   ```

   範例輸出：

   ```txt
   2023-07-16T06:19:40.068Z INFO records records/controller.go:149 apply chaos {"id": "default/hello-minikube-77b6f68484-dg4sw/echo-server"}
   2023-07-16T06:19:40.068Z INFO helloworldchaos helloworldchaos/types.go:26 Hello world!
   ```

## 後續步驟

若過程中遇到任何問題，請在 Chaos Mesh 存儲庫中創建 [issue](https://github.com/chaos-mesh/chaos-mesh/issues)。

在下一節中，我們將深入探討如何擴展 `HelloWorldChaos` 的行為。
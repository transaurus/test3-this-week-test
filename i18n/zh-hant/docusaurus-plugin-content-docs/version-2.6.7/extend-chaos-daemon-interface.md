---
title: Extend Chaos Daemon Interface
---

import PickHelmVersion from '@site/src/components/PickHelmVersion'

在[新增混沌實驗類型](add-new-chaos-experiment-type.md)中，您已新增了 `HelloWorldChaos`，它可以在 Chaos Controller Manager 的日誌中列印 `Hello world!`。

要讓 `HelloWorldChaos` 能夠向目標 Pod 注入一些故障，您需要擴充 Chaos Daemon 的介面。

:::tip

建議在繼續之前閱讀 [Chaos Mesh 的架構](./overview.md#architecture-overview)。

:::

本文件涵蓋：

- [Selector](#selector)

- [實作 gRPC 介面](#implement-the-grpc-interface)

- [驗證 HelloWorldChaos 的輸出](#verify-the-output-of-helloworldchaos)

- [後續步驟](#next-steps)

## Selector

在 `api/v1alpha1/helloworldchaos_type.go` 中，您已定義了 `HelloWorldSpec`，其中包含了 `ContainerSelector`：

```go
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

//...

// GetSelectorSpecs is a getter for selectors
func (obj *HelloWorldChaos) GetSelectorSpecs() map[string]interface{} {
        return map[string]interface{}{
                ".": &obj.Spec.ContainerSelector,
        }
}
```

在 Chaos Mesh 中，Selector 用於定義混沌實驗的範圍、目標命名空間、註解、標籤等。

Selector can also be some more specific values (e.g. `AWSSelector` in `AWSChaos`). Normally each chaos experiment needs only one selector, with exceptions like `NetworkChaos` because it sometimes needs two selectors as two objects for network partitioning.

您可以參考[定義混沌實驗的範圍](./define-chaos-experiment-scope.md)以取得有關 Selector 的更多資訊。

## 實作 gRPC 介面

為了讓 Chaos Daemon 能夠接受來自 Chaos Controller Manager 的請求，您需要實作一個新的 gRPC 介面。

1. 在 `pkg/chaosdaemon/pb/chaosdaemon.proto` 中新增 RPC：

   ```proto
   service ChaosDaemon {
      ...

      rpc ExecHelloWorldChaos(ExecHelloWorldRequest) returns (google.protobuf.Empty) {}
   }

   message ExecHelloWorldRequest {
      string container_id = 1;
   }
   ```

   然後您需要執行以下命令來更新相關的 `chaosdaemon.pb.go` 檔案：

   ```bash
   make proto
   ```

2. 在 Chaos Daemon 中實作 gRPC 服務。

   在 `pkg/chaosdaemon` 目錄中，建立一個名為 `helloworld_server.go` 的檔案，內容如下：

   ```go
   package chaosdaemon

   import (
           "context"

           "github.com/golang/protobuf/ptypes/empty"

           "github.com/chaos-mesh/chaos-mesh/pkg/bpm"
           "github.com/chaos-mesh/chaos-mesh/pkg/chaosdaemon/pb"
   )

   func (s *DaemonServer) ExecHelloWorldChaos(ctx context.Context, req *pb.ExecHelloWorldRequest) (*empty.Empty, error) {
           log := s.getLoggerFromContext(ctx)
           log.Info("ExecHelloWorldChaos", "request", req)

           pid, err := s.crClient.GetPidFromContainerID(ctx, req.ContainerId)
           if err != nil {
                   return nil, err
           }

           cmd := bpm.DefaultProcessBuilder("sh", "-c", "ps aux").
                   SetContext(ctx).
                   SetNS(pid, bpm.MountNS).
                   Build(ctx)
           out, err := cmd.Output()
           if err != nil {
                   return nil, err
           }
           if len(out) != 0 {
                   log.Info("cmd output", "output", string(out))
           }

           return &empty.Empty{}, nil
   }
   ```

   當 `chaos-daemon` 收到 `ExecHelloWorldChaos` 請求後，您可以在當前容器中看到程序列表。

3. Send a gRPC request when applying the chaos experiment.

   Every chaos experiment has a life cycle: `apply` and then `recover`. However, there are some chaos experiments that cannot be recovered by default (for example, PodKill in PodChaos and HelloWorldChaos). These are called OneShot experiments. You can find `+chaos-mesh:oneshot=true`, which we have defined in the `HelloWorldChaos` schema.

   The chaos controller manager needs to send a request to the chaos daemon when `HelloWorldChaos` is in the `apply` phase. This is done by updating `controllers/chaosimpl/helloworldchaos/types.go`:

   ```go
   func (impl *Impl) Apply(ctx context.Context, index int, records []*v1alpha1.Record, obj v1alpha1.InnerObject) (v1alpha1.Phase, error) {
           impl.Log.Info("Apply helloworld chaos")

           decodedContainer, err := impl.decoder.DecodeContainerRecord(ctx, records[index], obj)
           if err != nil {
                   return v1alpha1.NotInjected, err
           }

           pbClient := decodedContainer.PbClient
           containerId := decodedContainer.ContainerId

           _, err = pbClient.ExecHelloWorldChaos(ctx, &pb.ExecHelloWorldRequest{
                   ContainerId: containerId,
           })
           if err != nil {
                   return v1alpha1.NotInjected, err
           }

           return v1alpha1.Injected, nil
   }

   func (impl *Impl) Recover(ctx context.Context, index int, records []*v1alpha1.Record, obj v1alpha1.InnerObject) (v1alpha1.Phase, error) {
           impl.Log.Info("Recover helloworld chaos")
           return v1alpha1.NotInjected, nil
   }
   ```

   :::info

   There is no need to recover `HelloWorldChaos` because `HelloWorldChaos` is a **OneShot** experiment. For the type of chaos experiment you develop, you can implement the logic of the recovery function as needed.

   :::

## 驗證 HelloWorldChaos 輸出

現在可驗證 `HelloWorldChaos` 的輸出：

1. 依照[新增混沌實驗類型](add-new-chaos-experiment-type.md#step-4-build-docker-images)所述建置 Docker 映像檔，並載入至叢集。

   :::note

   若使用 minikube，請注意部分版本無法覆蓋相同標籤的現有映像檔，可先刪除舊映像檔再載入新檔。

   :::

2. 更新 Chaos Mesh：

   <PickHelmVersion>{`helm upgrade chaos-mesh helm/chaos-mesh -n=chaos-mesh --set controllerManager.leaderElection.enabled=false,dashboard.securityMode=false`}</PickHelmVersion>

3. 部署測試用 Pod：

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/chaos-mesh/apps/master/ping/busybox-statefulset.yaml
   ```

4. 建立 `hello-busybox.yaml` 檔案，內容如下：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: HelloWorldChaos
   metadata:
     name: hello-busybox
     namespace: chaos-mesh
   spec:
     selector:
       namespaces:
         - busybox
     mode: all
     duration: 1h
   ```

5. 執行：

   ```bash
   kubectl apply -f hello-busybox.yaml
   # helloworldchaos.chaos-mesh.org/hello-busybox created
   ```

   - 現在可以檢查 `chaos-controller-manager` 日誌中是否有 `Apply helloworld chaos` 記錄：

     ```bash
     kubectl logs -n chaos-mesh chaos-controller-manager-xxx
     ```

     範例輸出：

     ```log
     2023-07-16T08:20:46.823Z INFO records records/controller.go:149 apply chaos {"id": "busybox/busybox-0/busybox"}
     2023-07-16T08:20:46.823Z INFO helloworldchaos helloworldchaos/types.go:27 Apply helloworld chaos
     ```

   - 檢查 Chaos Daemon 的日誌：

     ```bash
     kubectl logs -n chaos-mesh chaos-daemon-xxx
     ```

     範例輸出：

     ```log
     2023-07-16T08:20:46.833Z INFO chaos-daemon.daemon-server chaosdaemon/server.go:187 ExecHelloWorldChaos {"namespacedName": "chaos-mesh/hello-busybox", "request": "container_id:\"docker://5e01e76efdec6aa0934afc15bb80e121d58b43c529a6696a01a242f7ac68f201\""}
     2023-07-16T08:20:46.834Z INFO chaos-daemon.daemon-server.background-process-manager.process-builder pb/chaosdaemon.pb.go:4568 build command {"namespacedName": "chaos-mesh/hello-busybox", "command": "/usr/local/bin/nsexec -m /proc/104710/ns/mnt -- sh -c ps aux"}
     2023-07-16T08:20:46.841Z INFO chaos-daemon.daemon-server chaosdaemon/server.go:187 cmd output {"namespacedName": "chaos-mesh/hello-busybox", "output": "PID   USER     TIME  COMMAND\n    1 root      0:00 sh -c echo Container is Running ; sleep 3600\n"}
     2023-07-16T08:20:46.856Z INFO chaos-daemon.daemon-server chaosdaemon/server.go:187 ExecHelloWorldChaos {"namespacedName": "chaos-mesh/hello-busybox", "request": "container_id:\"docker://bab4f632a0358529f7d72d35e014b8c2ce57438102d99d6174dd9df52d093e99\""}
     2023-07-16T08:20:46.864Z INFO chaos-daemon.daemon-server.background-process-manager.process-builder pb/chaosdaemon.pb.go:4568 build command {"namespacedName": "chaos-mesh/hello-busybox", "command": "/usr/local/bin/nsexec -m /proc/104841/ns/mnt -- sh -c ps aux"}
     2023-07-16T08:20:46.867Z INFO chaos-daemon.daemon-server chaosdaemon/server.go:187 cmd output {"namespacedName": "chaos-mesh/hello-busybox", "output": "PID   USER     TIME  COMMAND\n    1 root      0:00 sh -c echo Container is Running ; sleep 3600\n"}
     ```

   您將會看到兩行獨立的 `ps aux` 輸出，分別對應兩個不同的 Pod。

## 後續步驟

若在此過程中遇到任何問題，請至 Chaos Mesh 儲存庫建立[議題](https://github.com/chaos-mesh/chaos-mesh/issues)。

若想深入了解運作原理，可閱讀[controllers/README.md](https://github.com/chaos-mesh/chaos-mesh/blob/master/controllers/README.md)及各控制器的程式碼。

您現在已準備好成為 Chaos Mesh 開發者！歡迎造訪 [Chaos Mesh 議題](https://github.com/chaos-mesh/chaos-mesh/issues) 尋找[適合新手的議題](https://github.com/chaos-mesh/chaos-mesh/labels/good%20first%20issue)並開始貢獻！
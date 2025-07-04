---
slug: /implement-chaos-engineering-in-k8s
title: 'Implementing Chaos Engineering in K8s: Chaos Mesh Principle Analysis and Control Plane Development'
authors: mayocream
image: /img/blog/implement-chaos-engineering-in-k8s.png
tags: [Chaos Mesh, Chaos Engineering]
---

![在 K8s 中實現混沌工程](/img/blog/implement-chaos-engineering-in-k8s.png)

[Chaos Mesh](https://chaos-mesh.org/docs/) 是一個基於 Kubernetes (K8s) 自訂資源定義 (CRD) 的開源雲原生混沌工程平台。Chaos Mesh 能模擬多種類型的故障，並具備強大的故障場景編排能力。您可以使用 Chaos Mesh 方便地模擬開發、測試和生產環境中可能出現的各種異常狀況，從而發現系統中的潛在問題。

<!--truncate-->

本文將探討在 Kubernetes 叢集中實踐混沌工程的方法，透過源碼分析解讀 Chaos Mesh 的重要特性，並以程式碼範例說明如何開發 Chaos Mesh 的控制平面。

如果您不熟悉 Chaos Mesh，請先參閱 [Chaos Mesh 文件](https://chaos-mesh.org/docs/#architecture-overview) 以掌握其架構基礎知識。

本文的測試程式碼請參閱 GitHub 上的 [mayocream/chaos-mesh-controlpanel-demo](https://github.com/mayocream/chaos-mesh-controlpanel-demo) 儲存庫。

## Chaos Mesh 如何製造混沌

Chaos Mesh 是在 Kubernetes 上實施混沌工程的瑞士軍刀。本節將介紹其運作原理。

### 特權模式

Chaos Mesh 透過在 Kubernetes 中運行特權容器來製造故障。Chaos Daemon 的 Pod 以 `DaemonSet` 形式運行，並透過 Pod 的安全上下文為容器運行時附加額外的 [權能](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#capabilities)。

```yaml
apiVersion: apps/v1
kind: DaemonSet
spec:
 template:
   metadata: ...
   spec:
     containers:
       - name: chaos-daemon
         securityContext:
           {{- if .Values.chaosDaemon.privileged }}
           privileged: true
           capabilities:
             add:
               - SYS_PTRACE
           {{- else }}
           capabilities:
             add:
               - SYS_PTRACE
               - NET_ADMIN
               - MKNOD
               - SYS_CHROOT
               - SYS_ADMIN
               - KILL
               # CAP_IPC_LOCK is used to lock memory
               - IPC_LOCK
           {{- end }}
```

Linux 權能授予容器創建和訪問 `/dev/fuse` 用戶空間檔案系統 (FUSE) 管道的特權。FUSE 是 Linux 用戶空間檔案系統介面，它允許非特權用戶無需修改核心程式碼即可創建自己的檔案系統。

According to [pull request #1109](https://github.com/chaos-mesh/chaos-mesh/pull/1109) on GitHub, the `DaemonSet` program uses cgo to call the Linux `makedev` function to create a FUSE pipe.

```go
// #include <sys/sysmacros.h>
// #include <sys/types.h>
// // makedev is a macro, so a wrapper is needed
// dev_t Makedev(unsigned int maj, unsigned int min) {
//   return makedev(maj, min);
// }
// EnsureFuseDev ensures /dev/fuse exists. If not, it will create one

func EnsureFuseDev() {
   if _, err := os.Open("/dev/fuse"); os.IsNotExist(err) {
       // 10, 229 according to https://www.kernel.org/doc/Documentation/admin-guide/devices.txt
       fuse := C.Makedev(10, 229)
       syscall.Mknod("/dev/fuse", 0o666|syscall.S_IFCHR, int(fuse))
   }
}
```

In [pull request #1453](https://github.com/chaos-mesh/chaos-mesh/pull/1453), Chaos Daemon enables privileged mode by default; that is, it sets `privileged: true` in the container's `SecurityContext`.

### 終止 Pod

`PodKill`、`PodFailure` 和 `ContainerKill` 屬於 `PodChaos` 類別。`PodKill` 會隨機終止 Pod，它透過調用 API 伺服器發送終止指令。

```go
import (
   "context"
   v1 "k8s.io/api/core/v1"
   "sigs.k8s.io/controller-runtime/pkg/client"
)

type Impl struct {
   client.Client
}

func (impl *Impl) Apply(ctx context.Context, index int, records []*v1alpha1.Record, obj v1alpha1.InnerObject) (v1alpha1.Phase, error) {
   ...
   err = impl.Get(ctx, namespacedName, &pod)
   if err != nil {
       // TODO: handle this error
       return v1alpha1.NotInjected, err
   }
   err = impl.Delete(ctx, &pod, &client.DeleteOptions{
       GracePeriodSeconds: &podchaos.Spec.GracePeriod, // PeriodSeconds has to be set specifically
   })
   ...
   return v1alpha1.Injected, nil
}
```

`GracePeriodSeconds` 參數允許 Kubernetes [強制終止 Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced)。例如若需立即刪除 Pod，可使用 `kubectl delete pod --grace-period=0 --force` 指令。

`PodFailure` patches the Pod object resource to replace the image in the Pod with a wrong one. Chaos only modifies the `image` fields of `containers` and `initContainers`. This is because most of the metadata about a Pod is immutable. For more details, see [Pod update and replacement](https://kubernetes.io/docs/concepts/workloads/pods/#pod-update-and-replacement).

```go
func (impl *Impl) Apply(ctx context.Context, index int, records []*v1alpha1.Record, obj v1alpha1.InnerObject) (v1alpha1.Phase, error) {
   ...
   pod := origin.DeepCopy()
   for index := range pod.Spec.Containers {
       originImage := pod.Spec.Containers[index].Image
       name := pod.Spec.Containers[index].Name
       key := annotation.GenKeyForImage(podchaos, name, false)
       if pod.Annotations == nil {
           pod.Annotations = make(map[string]string)
       }
       // If the annotation is already existed, we could skip the reconcile for this container
       if _, ok := pod.Annotations[key]; ok {
           continue
       }
       pod.Annotations[key] = originImage
       pod.Spec.Containers[index].Image = config.ControllerCfg.PodFailurePauseImage
   }
   for index := range pod.Spec.InitContainers {
       originImage := pod.Spec.InitContainers[index].Image
       name := pod.Spec.InitContainers[index].Name
       key := annotation.GenKeyForImage(podchaos, name, true)
       if pod.Annotations == nil {
           pod.Annotations = make(map[string]string)
       }
       // If the annotation is already existed, we could skip the reconcile for this container
       if _, ok := pod.Annotations[key]; ok {
           continue
       }
       pod.Annotations[key] = originImage
       pod.Spec.InitContainers[index].Image = config.ControllerCfg.PodFailurePauseImage
   }
   err = impl.Patch(ctx, pod, client.MergeFrom(&origin))
   if err != nil {
       // TODO: handle this error
       return v1alpha1.NotInjected, err
   }
   return v1alpha1.Injected, nil
}
```

預設導致故障的容器鏡像為 `gcr.io/google-containers/pause:latest`。

`PodKill` 和 `PodFailure` 透過 Kubernetes API 伺服器控制 Pod 生命週期，但 `ContainerKill` 則是透過運行在叢集節點上的 Chaos Daemon 實現。`ContainerKill` 使用 Chaos Controller Manager 運行客戶端，向 Chaos Daemon 發起 gRPC 調用。

```go
func (b *ChaosDaemonClientBuilder) Build(ctx context.Context, pod *v1.Pod) (chaosdaemonclient.ChaosDaemonClientInterface, error) {
   ...
   daemonIP, err := b.FindDaemonIP(ctx, pod)
   if err != nil {
       return nil, err
   }
   builder := grpcUtils.Builder(daemonIP, config.ControllerCfg.ChaosDaemonPort).WithDefaultTimeout()
   if config.ControllerCfg.TLSConfig.ChaosMeshCACert != "" {
       builder.TLSFromFile(config.ControllerCfg.TLSConfig.ChaosMeshCACert, config.ControllerCfg.TLSConfig.ChaosDaemonClientCert, config.ControllerCfg.TLSConfig.ChaosDaemonClientKey)
   } else {
       builder.Insecure()
   }
   cc, err := builder.Build()
   if err != nil {
       return nil, err
   }
   return chaosdaemonclient.New(cc), nil
}
```

當 Chaos Controller Manager 向 Chaos Daemon 發送指令時，它會根據 Pod 的資訊建立對應的客戶端。例如，為了控制節點上的 Pod，會透過獲取該 Pod 所在節點的 `ClusterIP` 來建立客戶端。若存在傳輸層安全性 (TLS) 憑證配置，Controller Manager 會為客戶端新增 TLS 憑證。

當 Chaos Daemon 啟動時，若具備 TLS 憑證，會附加該憑證以啟用 gRPCS。TLS 配置選項 `RequireAndVerifyClientCert` 表示是否啟用雙向 TLS (mTLS) 驗證。

```go
func newGRPCServer(containerRuntime string, reg prometheus.Registerer, tlsConf tlsConfig) (*grpc.Server, error) {
   ...
   if tlsConf != (tlsConfig{}) {
       caCert, err := ioutil.ReadFile(tlsConf.CaCert)
       if err != nil {
           return nil, err
       }
       caCertPool := x509.NewCertPool()
       caCertPool.AppendCertsFromPEM(caCert)
       serverCert, err := tls.LoadX509KeyPair(tlsConf.Cert, tlsConf.Key)
       if err != nil {
           return nil, err
       }
       creds := credentials.NewTLS(&tls.Config{
           Certificates: []tls.Certificate{serverCert},
           ClientCAs:    caCertPool,
           ClientAuth:   tls.RequireAndVerifyClientCert,
       })
       grpcOpts = append(grpcOpts, grpc.Creds(creds))
   }
   s := grpc.NewServer(grpcOpts...)
   grpcMetrics.InitializeMetrics(s)
   pb.RegisterChaosDaemonServer(s, ds)
   reflection.Register(s)
   return s, nil
}
```

Chaos Daemon 提供以下 gRPC 介面供呼叫：

```go
// ChaosDaemonClient is the client API for ChaosDaemon service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://godoc.org/google.golang.org/grpc#ClientConn.NewStream.

type ChaosDaemonClient interface {

   SetTcs(ctx context.Context, in *TcsRequest, opts ...grpc.CallOption) (*empty.Empty, error)
   FlushIPSets(ctx context.Context, in *IPSetsRequest, opts ...grpc.CallOption) (*empty.Empty, error)
   SetIptablesChains(ctx context.Context, in *IptablesChainsRequest, opts ...grpc.CallOption) (*empty.Empty, error)
   SetTimeOffset(ctx context.Context, in *TimeRequest, opts ...grpc.CallOption) (*empty.Empty, error)
   RecoverTimeOffset(ctx context.Context, in *TimeRequest, opts ...grpc.CallOption) (*empty.Empty, error)
   ContainerKill(ctx context.Context, in *ContainerRequest, opts ...grpc.CallOption) (*empty.Empty, error)
   ContainerGetPid(ctx context.Context, in *ContainerRequest, opts ...grpc.CallOption) (*ContainerResponse, error)
   ExecStressors(ctx context.Context, in *ExecStressRequest, opts ...grpc.CallOption) (*ExecStressResponse, error)
   CancelStressors(ctx context.Context, in *CancelStressRequest, opts ...grpc.CallOption) (*empty.Empty, error)
   ApplyIOChaos(ctx context.Context, in *ApplyIOChaosRequest, opts ...grpc.CallOption) (*ApplyIOChaosResponse, error)
   ApplyHttpChaos(ctx context.Context, in *ApplyHttpChaosRequest, opts ...grpc.CallOption) (*ApplyHttpChaosResponse, error)
   SetDNSServer(ctx context.Context, in *SetDNSServerRequest, opts ...grpc.CallOption) (*empty.Empty, error)
}
```

### 網路故障注入

從 [pull request #41](https://github.com/chaos-mesh/chaos-mesh/pull/41) 可知，Chaos Mesh 透過呼叫 `pbClient.SetNetem` 將參數封裝成請求，再發送至節點上的 Chaos Daemon 處理，以此注入網路故障。

以下展示的是 2019 年的網路故障注入程式碼，隨著專案發展，這些功能已分散至多個檔案。

```go
func (r *Reconciler) applyPod(ctx context.Context, pod *v1.Pod, networkchaos *v1alpha1.NetworkChaos) error {
   ...
   pbClient := pb.NewChaosDaemonClient(c)
   containerId := pod.Status.ContainerStatuses[0].ContainerID
   netem, err := spec.ToNetem()
   if err != nil {
       return err
   }
   _, err = pbClient.SetNetem(ctx, &pb.NetemRequest{
       ContainerId: containerId,
       Netem:       netem,
   })
   return err
}
```

在 `pkg/chaosdaemon` 套件中，可觀察 Chaos Daemon 處理請求的機制。

```go
func (s *Server) SetNetem(ctx context.Context, in *pb.NetemRequest) (*empty.Empty, error) {
   log.Info("Set netem", "Request", in)
   pid, err := s.crClient.GetPidFromContainerID(ctx, in.ContainerId)
   if err != nil {
       return nil, status.Errorf(codes.Internal, "get pid from containerID error: %v", err)
   }
   if err := Apply(in.Netem, pid); err != nil {
       return nil, status.Errorf(codes.Internal, "netem apply error: %v", err)
   }
   return &empty.Empty{}, nil
}

// Apply applies a netem on eth0 in pid related namespace

func Apply(netem *pb.Netem, pid uint32) error {
   log.Info("Apply netem on PID", "pid", pid)
   ns, err := netns.GetFromPath(GenNetnsPath(pid))
   if err != nil {
       log.Error(err, "failed to find network namespace", "pid", pid)
       return errors.Trace(err)
   }
   defer ns.Close()
   handle, err := netlink.NewHandleAt(ns)
   if err != nil {
       log.Error(err, "failed to get handle at network namespace", "network namespace", ns)
       return err
   }
   link, err := handle.LinkByName("eth0") // TODO: check whether interface name is eth0
   if err != nil {
       log.Error(err, "failed to find eth0 interface")
       return errors.Trace(err)
   }
   netemQdisc := netlink.NewNetem(netlink.QdiscAttrs{
       LinkIndex: link.Attrs().Index,
       Handle:    netlink.MakeHandle(1, 0),
       Parent:    netlink.HANDLE_ROOT,
   }, ToNetlinkNetemAttrs(netem))
   if err = handle.QdiscAdd(netemQdisc); err != nil {
       if !strings.Contains(err.Error(), "file exists") {
           log.Error(err, "failed to add Qdisc")
           return errors.Trace(err)
       }
   }
   return nil
}
```

最終透過 [`vishvananda/netlink` 函式庫](https://github.com/vishvananda/netlink) 操作 Linux 網路介面來完成工作。

至此，`NetworkChaos` 透過操作 Linux 主機網路製造混亂，包含 iptables 和 ipset 等工具。

在 Chaos Daemon 的 Dockerfile 中，可見其依賴的 Linux 工具鏈：

```dockerfile
RUN apt-get update && \
   apt-get install -y tzdata iptables ipset stress-ng iproute2 fuse util-linux procps curl && \
   rm -rf /var/lib/apt/lists/*
```

### 壓力測試

Chaos Daemon 也實作 `StressChaos`。Controller Manager 計算規則後，將任務發送至特定 `Daemon`。下方展示組裝後的參數，這些參數會合併為命令執行參數，附加至 `stress-ng` 命令執行。

```go
// Normalize the stressors to comply with stress-ng
func (in *Stressors) Normalize() (string, error) {
   stressors := ""
   if in.MemoryStressor != nil && in.MemoryStressor.Workers != 0 {
       stressors += fmt.Sprintf(" --vm %d --vm-keep", in.MemoryStressor.Workers)
       if len(in.MemoryStressor.Size) != 0 {
           if in.MemoryStressor.Size[len(in.MemoryStressor.Size)-1] != '%' {
               size, err := units.FromHumanSize(string(in.MemoryStressor.Size))
               if err != nil {
                   return "", err
               }
               stressors += fmt.Sprintf(" --vm-bytes %d", size)
           } else {
               stressors += fmt.Sprintf(" --vm-bytes %s",
                   in.MemoryStressor.Size)
           }
       }
       if in.MemoryStressor.Options != nil {
           for _, v := range in.MemoryStressor.Options {
               stressors += fmt.Sprintf(" %v ", v)
           }
       }
   }
   if in.CPUStressor != nil && in.CPUStressor.Workers != 0 {
       stressors += fmt.Sprintf(" --cpu %d", in.CPUStressor.Workers)
       if in.CPUStressor.Load != nil {
           stressors += fmt.Sprintf(" --cpu-load %d",
               *in.CPUStressor.Load)
       }
       if in.CPUStressor.Options != nil {
           for _, v := range in.CPUStressor.Options {
               stressors += fmt.Sprintf(" %v ", v)
           }
       }
   }
   return stressors, nil
}
```

Chaos Daemon 伺服器端處理函式執行命令時，呼叫官方的 Go 套件 `os/exec`。詳見 [`pkg/chaosdaemon/stress_server_linux.go`](https://github.com/chaos-mesh/chaos-mesh/blob/98af3a0e7832a4971d6b133a32069539d982ef0a/pkg/chaosdaemon/stress_server_linux.go#L33) 檔案。另有以 darwin 結尾的同名檔案，`*_darwin` 檔案用於避免程式在 macOS 執行時可能產生的錯誤。

程式碼使用 [`shirou/gopsutil`](https://github.com/shirou/gopsutil) 套件獲取 PID 程序狀態，並讀取 stdout 和 stderr 標準輸出。此處理模式可見於 [`hashicorp/go-plugin`](https://github.com/hashicorp/go-plugin)，而 go-plugin 的實作更為完善。

### I/O 故障注入

[Pull request #826](https://github.com/chaos-mesh/chaos-mesh/pull/826) 引入 IOChaos 的新實作方式，無需使用邊車注入。其透過 Chaos Daemon 以 [runc](https://github.com/opencontainers/runc) 容器的底層命令直接操作 Linux 命名空間，並執行由 Rust 開發的 [chaos-mesh/toda](https://github.com/chaos-mesh/toda) FUSE 程式來注入容器 I/O 混亂。toda 與控制平面間使用 [JSON-RPC 2.0](https://pkg.go.dev/github.com/ethereum/go-ethereum/rpc) 協定進行通訊。

新的 IOChaos 實作不會修改 Pod 資源。當您定義 IOChaos 混沌實驗時，對於每個由選擇器欄位過濾出的 Pod，都會建立一個對應的 PodIOChaos 資源。PodIoChaos 的 [owner reference](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/) 是該 Pod。同時，會為 PodIoChaos 加入一組 [finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/)，以便在 PodIoChaos 被刪除前釋放其資源。

```go
// Apply implements the reconciler.InnerReconciler.Apply

func (r *Reconciler) Apply(ctx context.Context, req ctrl.Request, chaos v1alpha1.InnerObject) error {
   iochaos, ok := chaos.(*v1alpha1.IoChaos)
   if !ok {
       err := errors.New("chaos is not IoChaos")
       r.Log.Error(err, "chaos is not IoChaos", "chaos", chaos)
       return err
   }
   source := iochaos.Namespace + "/" + iochaos.Name
   m := podiochaosmanager.New(source, r.Log, r.Client)
   pods, err := utils.SelectAndFilterPods(ctx, r.Client, r.Reader, &iochaos.Spec)
   if err != nil {
       r.Log.Error(err, "failed to select and filter pods")
       return err
   }
   r.Log.Info("applying iochaos", "iochaos", iochaos)
   for _, pod := range pods {
       t := m.WithInit(types.NamespacedName{
           Name:      pod.Name,
           Namespace: pod.Namespace,
       })

       // TODO: support chaos on multiple volume

       t.SetVolumePath(iochaos.Spec.VolumePath)
       t.Append(v1alpha1.IoChaosAction{
           Type: iochaos.Spec.Action,
           Filter: v1alpha1.Filter{
               Path:    iochaos.Spec.Path,
               Percent: iochaos.Spec.Percent,
               Methods: iochaos.Spec.Methods,
           },
           Faults: []v1alpha1.IoFault{
               {
                   Errno:  iochaos.Spec.Errno,
                   Weight: 1,
               },
           },
           Latency:          iochaos.Spec.Delay,
           AttrOverrideSpec: iochaos.Spec.Attr,
           Source:           m.Source,
       })
       key, err := cache.MetaNamespaceKeyFunc(&pod)
       if err != nil {
           return err
       }
       iochaos.Finalizers = utils.InsertFinalizer(iochaos.Finalizers, key)
   }
   r.Log.Info("commiting updates of podiochaos")
   err = m.Commit(ctx)
   if err != nil {
       r.Log.Error(err, "fail to commit")
       return err
   }
   r.Event(iochaos, v1.EventTypeNormal, utils.EventChaosInjected, "")
   return nil
}
```

在 PodIoChaos 資源的控制器中，Controller Manager 將資源封裝成參數，並呼叫 Chaos Daemon 介面來處理這些參數。

```go
// Apply flushes io configuration on pod

func (h *Handler) Apply(ctx context.Context, chaos *v1alpha1.PodIoChaos) error {
   h.Log.Info("updating io chaos", "pod", chaos.Namespace+"/"+chaos.Name, "spec", chaos.Spec)
   ...
   res, err := pbClient.ApplyIoChaos(ctx, &pb.ApplyIoChaosRequest{
       Actions:     input,
       Volume:      chaos.Spec.VolumeMountPath,
       ContainerId: containerID,
       Instance:  chaos.Spec.Pid,
       StartTime: chaos.Spec.StartTime,
   })
   if err != nil {
       return err
   }
   chaos.Spec.Pid = res.Instance
   chaos.Spec.StartTime = res.StartTime
   chaos.OwnerReferences = []metav1.OwnerReference{
       {
           APIVersion: pod.APIVersion,
           Kind:       pod.Kind,
           Name:       pod.Name,
           UID:        pod.UID,
       },
   }
   return nil
}
```

`pkg/chaosdaemon/iochaos_server.go` 檔案處理 IOChaos。在該檔案中，需要將一個 FUSE 程式注入容器中。如 GitHub 上的 issue [#2305](https://github.com/chaos-mesh/chaos-mesh/issues/2305) 所討論，會執行 `/usr/local/bin/nsexec -l- p /proc/119186/ns/pid -m /proc/119186/ns/mnt - /usr/local/bin/toda --path /tmp --verbose info` 命令，以在與 Pod 相同的命名空間下執行 toda 程式。

```go
func (s *DaemonServer) ApplyIOChaos(ctx context.Context, in *pb.ApplyIOChaosRequest) (*pb.ApplyIOChaosResponse, error) {
   ...
   pid, err := s.crClient.GetPidFromContainerID(ctx, in.ContainerId)
   if err != nil {
       log.Error(err, "error while getting PID")
       return nil, err
   }
   args := fmt.Sprintf("--path %s --verbose info", in.Volume)
   log.Info("executing", "cmd", todaBin+" "+args)
   processBuilder := bpm.DefaultProcessBuilder(todaBin, strings.Split(args, " ")...).
       EnableLocalMnt().
       SetIdentifier(in.ContainerId)
   if in.EnterNS {
       processBuilder = processBuilder.SetNS(pid, bpm.MountNS).SetNS(pid, bpm.PidNS)
   }
   ...

   // Calls JSON RPC

   client, err := jrpc.DialIO(ctx, receiver, caller)
   if err != nil {
       return nil, err
   }
   cmd := processBuilder.Build()
   procState, err := s.backgroundProcessManager.StartProcess(cmd)
   if err != nil {
       return nil, err
   }
   ...
}
```

以下程式碼範例建構了執行命令。這些命令是 runc 底層命名空間隔離的實作：

```go
// GetNsPath returns corresponding namespace path

func GetNsPath(pid uint32, typ NsType) string {
   return fmt.Sprintf("%s/%d/ns/%s", DefaultProcPrefix, pid, string(typ))
}

// SetNS sets the namespace of the process

func (b *ProcessBuilder) SetNS(pid uint32, typ NsType) *ProcessBuilder {
   return b.SetNSOpt([]nsOption{{
       Typ:  typ,
       Path: GetNsPath(pid, typ),
   }})
}

// Build builds the process

func (b *ProcessBuilder) Build() *ManagedProcess {
   args := b.args
   cmd := b.cmd
   if len(b.nsOptions) > 0 {
       args = append([]string{"--", cmd}, args...)
       for _, option := range b.nsOptions {
           args = append([]string{"-" + nsArgMap[option.Typ], option.Path}, args...)
       }
       if b.localMnt {
           args = append([]string{"-l"}, args...)
       }
       cmd = nsexecPath
   }
   ...
}
```

## 控制平面

Chaos Mesh 是一個基於 Apache 2.0 協議的開源混沌工程系統。如上所述，它具有豐富的功能和良好的生態系統。維護團隊基於該混沌系統開發了 [`chaos-mesh/toda`](https://github.com/chaos-mesh/toda) FUSE、[`chaos-mesh/k8s_dns_chaos`](https://github.com/chaos-mesh/k8s_dns_chaos) CoreDNS 混沌外掛，以及基於 Berkeley 封包過濾器 (BPF) 的核心錯誤注入 [`chaos-mesh/bpfki`](https://github.com/chaos-mesh/bpfki)。

現在，我將描述建構一個面向終端使用者的混沌工程平台所需的伺服器端程式碼。此實作僅為一個範例——並非最佳範例。如果您想查看真實平台上的開發實踐，可以參考 Chaos Mesh 的 [Dashboard](https://github.com/chaos-mesh/chaos-mesh/tree/master/pkg/dashboard)。它使用了 [`uber-go/fx`](https://github.com/uber-go/fx) 依賴注入框架以及控制器執行期的管理員模式。

### 關鍵的 Chaos Mesh 功能

如下方的 Chaos Mesh 工作流程所示，我們需要實作一個伺服器來發送 YAML 到 Kubernetes API。Chaos Controller Manager 實作了複雜的規則驗證和規則傳遞給 Chaos Daemon。如果您想在自己的平台中使用 Chaos Mesh，您只需要連接到建立 CRD 資源的流程。

![Chaos Mesh 的基本工作流程](/img/blog/chaos-mesh-basic-workflow.png)

讓我們來看一下 Chaos Mesh 網站上的範例：

```go
import (
   "context"
   "github.com/pingcap/chaos-mesh/api/v1alpha1"
   "sigs.k8s.io/controller-runtime/pkg/client"
)

func main() {
   ...
   delay := &chaosv1alpha1.NetworkChaos{
       Spec: chaosv1alpha1.NetworkChaosSpec{...},
   }
   k8sClient := client.New(conf, client.Options{ Scheme: scheme.Scheme })
   k8sClient.Create(context.TODO(), delay)
   k8sClient.Delete(context.TODO(), delay)
}
```

Chaos Mesh provides APIs corresponding to all CRDs. We use the [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) developed by Kubernetes [API Machinery SIG](https://github.com/kubernetes/community/tree/master/sig-api-machinery) to simplify the interaction with the Kubernetes API.

### 注入混沌

假設我們希望透過呼叫程式來建立一個 `PodKill` 資源。當資源傳送至 Kubernetes API 伺服器後，會經過 Chaos Controller Manager 的 [驗證准入控制器](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) 進行資料驗證。建立混沌實驗時，若准入控制器無法驗證輸入資料，會向用戶端返回錯誤。具體參數可參考 [使用 YAML 設定檔案建立實驗](https://chaos-mesh.org/docs/simulate-pod-chaos-on-kubernetes/#create-experiments-using-yaml-configuration-files)。

`NewClient` 會建立一個 Kubernetes API 用戶端。可參考以下範例：

```go
package main

import (
   "context"
   "controlpanel"
   "log"
   "github.com/chaos-mesh/chaos-mesh/api/v1alpha1"
   "github.com/pkg/errors"
   metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func applyPodKill(name, namespace string, labels map[string]string) error {
   cli, err := controlpanel.NewClient()
   if err != nil {
       return errors.Wrap(err, "create client")
   }
   cr := &v1alpha1.PodChaos{
       ObjectMeta: metav1.ObjectMeta{
           GenerateName: name,
           Namespace:    namespace,
       },
       Spec: v1alpha1.PodChaosSpec{
           Action: v1alpha1.PodKillAction,
           ContainerSelector: v1alpha1.ContainerSelector{
               PodSelector: v1alpha1.PodSelector{
                   Mode: v1alpha1.OnePodMode,
                   Selector: v1alpha1.PodSelectorSpec{
                       Namespaces:     []string{namespace},
                       LabelSelectors: labels,
                   },
               },
           },
       },
   }

   if err := cli.Create(context.Background(), cr); err != nil {
       return errors.Wrap(err, "create podkill")
   }
   return nil
}
```

執行程式的日誌輸出為：

```bash
I1021 00:51:55.225502   23781 request.go:665] Waited for 1.033116256s due to client-side throttling, not priority and fairness, request: GET:https://***
2021/10/21 00:51:56 apply podkill
```

使用 kubectl 檢查 `PodKill` 資源的狀態：

```bash
$ k describe podchaos.chaos-mesh.org -n dev podkillvjn77
Name:         podkillvjn77
Namespace:    dev
Labels:       <none>
Annotations:  <none>
API Version:  chaos-mesh.org/v1alpha1
Kind:         PodChaos

Metadata:
 Creation Timestamp:  2021-10-20T16:51:56Z
 Finalizers:
   chaos-mesh/records
 Generate Name:     podkill
 Generation:        7
 Resource Version:  938921488
 Self Link:         /apis/chaos-mesh.org/v1alpha1/namespaces/dev/podchaos/podkillvjn77
 UID:               afbb40b3-ade8-48ba-89db-04918d89fd0b

Spec:
 Action:        pod-kill
 Grace Period:  0
 Mode:          one
 Selector:
   Label Selectors:
     app:  nginx
   Namespaces:
     dev

Status:
 Conditions:
   Reason:
   Status:  False
   Type:    Paused
   Reason:
   Status:  True
   Type:    Selected
   Reason:
   Status:  True
   Type:    AllInjected
   Reason:
   Status:  False
   Type:    AllRecovered

 Experiment:
   Container Records:
     Id:            dev/nginx
     Phase:         Injected
     Selector Key:  .
   Desired Phase:   Run

Events:
 Type    Reason           Age    From          Message
 ----    ------           ----   ----          -------
 Normal  FinalizerInited  6m35s  finalizer     Finalizer has been inited
 Normal  Updated          6m35s  finalizer     Successfully update finalizer of resource
 Normal  Updated          6m35s  records       Successfully update records of resource
 Normal  Updated          6m35s  desiredphase  Successfully update desiredPhase of resource
 Normal  Applied          6m35s  records       Successfully apply chaos for dev/nginx
 Normal  Updated          6m35s  records       Successfully update records of resource
```

控制平面還需查詢並取得混沌資源，讓平台使用者能檢視所有混沌實驗的執行狀態並進行管理。為實現此功能，我們可呼叫 `REST` API 發送 `Get` 或 `List` 請求。但在實務中需注意細節：我們公司觀察到，每當控制器請求完整資源資料時，Kubernetes API 伺服器的負載會明顯升高。

I recommend that you read the [How to use the controller-runtime client](https://zoetrope.github.io/kubebuilder-training/controller-runtime/client.html) (in Japanese) controller runtime tutorial. If you don't understand Japanese, you can still learn a lot from the tutorial by reading the source code. It covers many details. For example, by default, the controller runtime reads kubeconfig, flags, environment variables, and the service account automatically mounted in the Pod from multiple locations. [Pull request #21](https://github.com/armosec/kubescape/pull/21) for [`armosec/kubescape`](https://github.com/armosec/kubescape) uses this feature. This tutorial also includes common operations, such as how to paginate, update, and overwrite objects. I haven't seen any English tutorials that are so detailed.

以下是 `Get` 與 `List` 請求的範例：

```go
package controlpanel

import (
   "context"
   "github.com/chaos-mesh/chaos-mesh/api/v1alpha1"
   "github.com/pkg/errors"
   "sigs.k8s.io/controller-runtime/pkg/client"
)

func GetPodChaos(name, namespace string) (*v1alpha1.PodChaos, error) {
   cli := mgr.GetClient()
   item := new(v1alpha1.PodChaos)
   if err := cli.Get(context.Background(), client.ObjectKey{Name: name, Namespace: namespace}, item); err != nil {
       return nil, errors.Wrap(err, "get cr")
   }
   return item, nil
}

func ListPodChaos(namespace string, labels map[string]string) ([]v1alpha1.PodChaos, error) {
   cli := mgr.GetClient()
   list := new(v1alpha1.PodChaosList)
   if err := cli.List(context.Background(), list, client.InNamespace(namespace), client.MatchingLabels(labels)); err != nil {
       return nil, err
   }
   return list.Items, nil
}
```

此範例採用管理器模式，可避免快取機制重複獲取大量資料。下圖 [figure](https://zoetrope.github.io/kubebuilder-training/controller-runtime/client.html) 展示其工作流程：

1. 取得 Pod。

2. 首次取得 `List` 請求的完整資料。

3. 當監視資料變更時更新快取。

![List request](/img/blog/list-request.png)

### 編排混沌

容器執行時期介面 (CRI) 提供強大的底層隔離能力以維持容器穩定運行，但面對更複雜的可擴展場景時，需進行容器編排。Chaos Mesh 還提供 [`Schedule`](https://chaos-mesh.org/docs/define-scheduling-rules/) 與 [`Workflow`](https://chaos-mesh.org/docs/create-chaos-mesh-workflow/) 功能：基於設定的 `Cron` 時間，`Schedule` 可定期或間隔觸發故障；`Workflow` 則能像 Argo Workflows 般排程多個故障測試。

Chaos Controller Manager 已處理大部分工作，控制平面主要管理這些 YAML 資源。您只需專注於提供給終端使用者的功能設計。

### 平台功能

下圖展示 Chaos Mesh 儀表板，我們需思考平台應提供哪些功能給終端使用者。

![Chaos Mesh Dashboard](/img/blog/chaos-mesh-dashboard-k8s.png)

由儀表板可知，平台可能具備以下功能：

- 混沌注入

- Pod 崩潰

- 網路故障

- 負載測試

- I/O 故障

- 事件追蹤

- 關聯警報

- 時序遙測

若您對 Chaos Mesh 感興趣並希望參與改進，請加入其 [Slack 頻道](https://slack.cncf.io/) (#project-chaos-mesh) 或提交 pull request 與問題至 [GitHub 倉庫](https://github.com/chaos-mesh/chaos-mesh)。
---
slug: /how-to-efficiently-stress-test-pod-memory
title: 'How to efficiently stress test Pod memory'
authors: yinghaowang
image: /img/blog/how-to-efficiently-stress-test-pod-memory-banner.jpg
tags: [Chaos Mesh, Chaos Engineering, StressChaos, Stress Testing]
---

![橫幅圖片](/img/blog/how-to-efficiently-stress-test-pod-memory-banner.jpg)

[Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh) 內含 StressChaos 工具，可將 CPU 與記憶體壓力注入您的 Pod。當您測試或評估對 CPU/記憶體敏感的程式，並想了解其在壓力下的行為時，此工具非常實用。

然而在測試與使用 StressChaos 過程中，我們發現一些可用性與效能問題。例如：為何 StressChaos 消耗的記憶體遠低於設定值？為修正這些問題，我們開發了新測試方案。本文將說明如何排查這些問題並修正，助您充分發揮 StressChaos 效能。

<!--truncate-->

繼續前，請先在叢集中安裝 Chaos Mesh，詳細步驟請參閱我們的[官網文件](https://chaos-mesh.org/docs/quick-start)。

## 對目標注入壓力

I’d like to demonstrate how to inject StressChaos into a target. In this example, I’ll use [`hello-kubernetes`](https://github.com/paulbouwer/hello-kubernetes), which is managed by [helm charts](https://helm.sh/). The first step is to clone the [`hello-kubernetes`](https://github.com/paulbouwer/hello-kubernetes) repo and modify the chart to give it a resource limit.

```bash
git clone https://github.com/paulbouwer/hello-kubernetes.git
code deploy/helm/hello-kubernetes/values.yaml # or whichever editor you prefer
```

找到 resources 設定行，修改為：

```yaml
resources:
  requests:
    memory: '200Mi'
  limits:
    memory: '500Mi'
```

但在注入壓力前，先觀察目標的記憶體消耗量。進入 Pod 並啟動 shell（請替換範例中的 Pod 名稱）：

```bash
kubectl exec -it -n hello-kubernetes hello-kubernetes-hello-world-b55bfcf68-8mln6 -- /bin/sh
```

顯示記憶體使用摘要，輸入：

```sh
/usr/src/app $ free -m
/usr/src/app $ top
```

如下方輸出所示，該 Pod 消耗了 4,269 MB 記憶體。

```sh
/usr/src/app $ free -m
              used
Mem:          4269
Swap:            0

/usr/src/app $ top
Mem: 12742432K used
  PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
    1     0 node     S     285m   2%   0   0% npm start
   18     1 node     S     284m   2%   3   0% node server.js
   29     0 node     S     1636   0%   2   0% /bin/sh
   36    29 node     R     1568   0%   3   0% top
```

這顯然有誤——我們已將記憶體限制設為 500 MiB，但 Pod 卻顯示消耗數 GB。即便加總所有行程記憶體用量，也未達 500 MiB。不過 top 與 free 指令的結果至少相近。

我們將對 Pod 執行 StressChaos 並觀察變化。使用的 yaml 如下：

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: mem-stress
  namespace: chaos-mesh
spec:
  mode: all
  selector:
    namespaces:
      - hello-kubernetes
  stressors:
    memory:
      workers: 4
      size: 50MiB
      options: ['']
  duration: '1h'
```

將 yaml 存檔（我命名為 `memory.yaml`），執行以下指令應用混沌實驗：

```bash
~ kubectl apply -f memory.yaml
stresschaos.chaos-mesh.org/mem-stress created
```

現在再次檢查記憶體使用量：

```sh
              used
Mem:          4332
Swap:            0

Mem: 12805568K used
  PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
   54    50 root     R    53252   0%   1  24% {stress-ng-vm} stress-ng --vm 4 --vm-keep --vm-bytes 50000000
   57    52 root     R    53252   0%   0  22% {stress-ng-vm} stress-ng --vm 4 --vm-keep --vm-bytes 50000000
   55    53 root     R    53252   0%   2  21% {stress-ng-vm} stress-ng --vm 4 --vm-keep --vm-bytes 50000000
   56    51 root     R    53252   0%   3  21% {stress-ng-vm} stress-ng --vm 4 --vm-keep --vm-bytes 50000000
   18     1 node     S     289m   2%   2   0% node server.js
    1     0 node     S     285m   2%   0   0% npm start
   51    49 root     S    41048   0%   0   0% {stress-ng-vm} stress-ng --vm 4 --vm-keep --vm-bytes 50000000
   50    49 root     S    41048   0%   2   0% {stress-ng-vm} stress-ng --vm 4 --vm-keep --vm-bytes 50000000
   52    49 root     S    41048   0%   0   0% {stress-ng-vm} stress-ng --vm 4 --vm-keep --vm-bytes 50000000
   53    49 root     S    41048   0%   3   0% {stress-ng-vm} stress-ng --vm 4 --vm-keep --vm-bytes 50000000
   49     0 root     S    41044   0%   0   0% stress-ng --vm 4 --vm-keep --vm-bytes 50000000
   29     0 node     S     1636   0%   3   0% /bin/sh
   48    29 node     R     1568   0%   1   0% top
```

可見 stress-ng 實例已被注入 Pod，記憶體增加 60 MiB，這與預期不符。[文件](https://manpages.ubuntu.com/manpages/focal/en/man1/stress-ng.1.html)顯示應增加 200 MiB（4 * 50 MiB）。

我們將壓力從 50 MiB 提升至 3,000 MiB，這應會突破 Pod 的記憶體限制。先刪除混沌實驗，修改大小後重新套用。

接著——砰！shell 以代碼 137 退出。稍後重新連接容器時，記憶體用量恢復正常，且找不到任何 stress-ng 實例！發生了什麼？

## 為何 StressChaos 會消失？

Kubernetes 透過 [cgroup](https://man7.org/linux/man-pages/man7/cgroups.7.html) 機制限制容器記憶體用量。要查看 Pod 的 500 MiB 限制，請進入容器並輸入：

```bash
/usr/src/app $ cat /sys/fs/cgroup/memory/memory.limit_in_bytes
524288000
```

輸出以位元組顯示，換算為 `500 * 1024 * 1024`。

Requests 僅用於調度 Pod 的部署位置。該 Pod 沒有記憶體限制或請求，但可視為其所有容器需求的總和。

我們從一開始就犯了錯誤。free 和 top 指令並非「cgroup 感知」工具，它們依賴 `/proc/meminfo` (procfs) 獲取數據。遺憾的是，`/proc/meminfo` 過於陳舊，早於 cgroup 技術出現。它提供的是**主機**記憶體資訊而非容器內的狀況。讓我們從頭開始，重新檢視記憶體使用量。

要獲取 cgroup 記憶體使用量，請輸入：

```sh
/usr/src/app $ cat /sys/fs/cgroup/memory/memory.usage_in_bytes
39821312
```

施加 50 MiB 的 StressChaos 後，結果如下：

```sh
/usr/src/app $ cat /sys/fs/cgroup/memory/memory.usage_in_bytes
93577216
```

這比未施加 StressChaos 時增加了約 51 MiB 記憶體使用量。

接著，為何 shell 會退出？退出碼 137 表示「容器收到 SIGKILL 訊號而失敗」。這引導我們檢查 Pod 狀態，需特別關注 Pod 狀態和事件：

```bash
~ kubectl describe pods -n hello-kubernetes
......
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
......
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
......
  Warning  Unhealthy  10m (x4 over 16m)    kubelet            Readiness probe failed: Get "http://10.244.1.19:8080/": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
  Normal   Killing    10m (x2 over 16m)    kubelet            Container hello-kubernetes failed liveness probe, will be restarted
......
```

事件記錄揭示了 shell 崩潰原因。`hello-kubernetes` 設有存活探針(liveness probe)，當容器記憶體逼近限制值時，應用程式開始故障，Kubernetes 決定終止並重啟容器。Pod 重啟時 StressChaos 即停止運行。此時可說混沌測試成功發揮作用——它暴露了 Pod 的脆弱性。您可修復問題後重新施加混沌測試。看似完美，但尚存疑問：為何四個 50 MiB 的 vm worker 進程總共只佔用 51 MiB？答案需檢視 stress-ng 源碼[此處](https://github.com/ColinIanKing/stress-ng/blob/819f7966666dafea5264cf1a2a0939fd344fcf08/stress-vm.c#L2074)：

```c
vm_bytes /= args->num_instances;
```

原來如此！文件說明有誤。多個 vm worker 會分攤指定的總記憶體量，而非每個 worker 獨立佔用 `mmap` 映射的全額空間。至此所有疑問終獲解答。後續章節將討論其他記憶體壓力情境。

## 若未設置存活探針會如何？

讓我們刪除探針設定後重試。找到 `deploy/helm/hello-kubernetes/templates/deployment.yaml` 中的以下行並刪除：

```yaml
livenessProbe:
  httpGet:
    path: /
    port: http
readinessProbe:
  httpGet:
    path: /
    port: http
```

隨後升級部署：

此情境有趣之處在於：記憶體使用量持續上升後驟降，呈現反覆震盪。究竟發生何事？讓我們檢查核心日誌，請注意最後兩行：

```sh
/usr/src/app $ dmesg
......
[189937.362908] [ pid ]   uid  tgid total_vm      rss nr_ptes swapents oom_score_adj name
[189937.363092] [441060]  1000 441060    63955     3791      80     3030           988 node
[189937.363110] [441688]     0 441688   193367     2136     372   181097          1000 stress-ng-vm
......
[189937.363148] Memory cgroup out of memory: Kill process 443160 (stress-ng-vm) score 1272 or sacrifice child
[189937.363186] Killed process 443160 (stress-ng-vm), UID 0, total-vm:773468kB, anon-rss:152704kB, file-rss:164kB, shmem-rss:0kB
```

輸出明確顯示 `stress-ng-vm` 進程因記憶體不足(OOM)錯誤而被終止。

當進程無法獲取所需記憶體時，情況將變得棘手。進程極可能故障，與其等待崩潰，不如終止部分進程以釋放記憶體。OOM killer 按特定順序終止進程，力求在造成最小干擾下回收最多記憶體。詳細運作機制請參閱[此介紹](https://lwn.net/Articles/391222/)。

觀察上述輸出，您會發現名為 `node` 的應用程序進程（本應永不終止）的 `oom_score_adj` 值為 988。這相當危險，因為它是被殺優先級最高的進程。但有一個簡單方法可阻止 OOM Killer 殺死特定進程：當您創建 Pod 時，它會被分配一個服務質量（QoS）等級。詳細資訊請參閱[配置 Pod 的服務質量](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)。

通常情況下，若創建 Pod 時精確指定了資源請求量，該 Pod 會被歸類為 `Guaranteed` 等級。當存在其他可殺對象時，OOM Killer 不會殺死 `Guaranteed` Pod 中的容器。這些可殺對象包括非 `Guaranteed` Pod 和 stress-ng 工作進程。未指定資源請求量的 Pod 會被標記為 `BestEffort` 等級，OOM Killer 將優先終止此類 Pod。

本次探索至此結束。我們建議不應使用 `free` 和 `top` 評估容器內存狀況。為 Pod 分配資源限制時需謹慎，並選擇合適的 QoS 等級。後續我們將編寫更詳細的 StressChaos 文檔。

## 深入探索 Kubernetes 內存管理機制

Kubernetes tries to evict Pods that use too much memory (but not more memory than their limits). Kubernetes gets your Pod memory usage from `/sys/fs/cgroup/memory/memory.usage_in_bytes` and subtracts it by the `total_inactive_file` line in `memory.stat`.

請注意 Kubernetes **不支援**交換空間（swap）。即使節點啟用了交換空間，Kubernetes 創建容器時也會設置 `swappiness=0`，這意味著交換空間實際上被禁用，此設計主要出於性能考量。

`memory.usage_in_bytes` equals `resident set` plus `cache`, and `total_inactive_file` is memory in cache that the OS can retrieve if the memory is running out. `memory.usage_in_bytes - total_inactive_file` is called `working_set`. You will get this `working_set` value by `kubectl top pod <your pod> --containers`. Kubernetes uses this value to decide whether or not to evict your Pods.

Kubernetes 定期檢查內存使用情況。若容器內存用量增長過快或無法被驅逐，則觸發 OOM Killer。Kubernetes 有保護自身進程的機制，因此總是選擇終止容器。容器被殺後是否重啟取決於您的重啟策略。若容器因內存不足被殺，執行 `kubectl describe pod <your pod>` 時會看到重啟原因標記為 `OOMKilled`。

另一值得提及的是內核內存。自 `v1.9` 起，Kubernetes 默認啟用內核內存支持功能（cgroup 內存子系統特性）。您可以限制容器內核內存用量，但遺憾的是在 `v4.2` 及更早內核版本中會導致 cgroup 洩漏。解決方案是升級內核至 `v4.3` 或禁用此功能。

## StressChaos 的實現原理

StressChaos 是測試容器在低內存環境下行為的簡便方法。其利用名為 `stress-ng` 的強大工具分配內存並持續寫入。由於容器有內存限制且限制綁定於 cgroup，我們需找到在特定 cgroup 中運行 `stress-ng` 的方法。幸運的是，此部分較易實現：通過向 `/sys/fs/cgroup/` 中的文件寫入數據，在擁有足夠權限時可將任意進程分配至任意 cgroup。

若您對 Chaos Mesh 感興趣並願意協助改進，歡迎加入我們的 [Slack 頻道](https://slack.cncf.io/) (#project-chaos-mesh)！或向 [GitHub 代碼庫](https://github.com/chaos-mesh/chaos-mesh)提交 Pull Request 與 Issues。
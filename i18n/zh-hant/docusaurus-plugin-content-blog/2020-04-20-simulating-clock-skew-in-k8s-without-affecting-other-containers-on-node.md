---
slug: /simulating-clock-skew-in-k8s-without-affecting-other-containers-on-node
title: Simulating Clock Skew in K8s Without Affecting Other Containers on the Node
authors: cwen
image: /img/blog/clock-sync-chaos-engineering-k8s.jpg
tags: [Chaos Mesh, Chaos Engineering, Kubernetes, Distributed System]
---

![分散式系統中的時鐘同步](/img/blog/clock-sync-chaos-engineering-k8s.jpg)

[Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh) 是一個易於使用的開源雲原生混沌工程平台，專為 Kubernetes (K8s) 設計，其新功能 TimeChaos 可模擬[時鐘偏移](https://en.wikipedia.org/wiki/Clock_skew#On_a_network)現象。通常當我們修改容器內的時鐘時，希望將[影響範圍最小化](https://learning.oreilly.com/library/view/chaos-engineering/9781491988459/ch07.html)，避免影響節點上的其他容器。然而實際上，實現這個目標可能比想像中困難。Chaos Mesh 如何解決這個問題？

<!--truncate-->

本文將闡述我們如何突破各種時鐘偏移模擬方法的限制，以及 Chaos Mesh 的 TimeChaos 如何實現容器內時間的自由波動。

## 模擬時鐘偏移而不影響節點上的其他容器

時鐘偏移指網路中節點時鐘之間的時間差異，可能導致分散式系統的可靠性問題，是複雜分散式系統設計者和開發者的關注點。例如在分散式 SQL 資料庫中，跨節點維持同步的本地時鐘至關重要，以實現一致的全局快照並確保交易的 ACID 特性。

目前已有公認的[時鐘同步解決方案](https://pingcap.com/blog/Time-in-Distributed-Systems/)，但未經適當測試，永遠無法確保實現的穩固性。

那麼如何測試分散式系統的全局快照一致性？答案顯而易見：可透過模擬時鐘偏移來測試分散式系統能否在異常時鐘條件下維持一致的全局快照。部分測試工具支援在容器內模擬時鐘偏移，但會對物理節點產生影響。

[TimeChaos](https://github.com/chaos-mesh/chaos-mesh/wiki/Time-Chaos) 是一款能**在容器內模擬時鐘偏移以測試應用程式影響，且不波及整個節點**的工具。如此可精準識別時鐘偏移的潛在後果並採取相應措施。

## 我們探索過的各種時鐘偏移模擬方法

檢視現有方案後，我們明確意識到它們無法應用於運行在 Kubernetes 上的 Chaos Mesh。兩種常見的時鐘偏移模擬方法——直接修改節點時鐘和使用 Jepsen 框架——都會改變節點上所有進程的時間。這些方案對我們而言皆不可接受。在 Kubernetes 容器中，若注入影響整個節點的時鐘偏移錯誤，同節點上的其他容器將受干擾，此種粗劣手法無法容忍。

那麼該如何解決此問題？首先想到的是在核心層使用[柏克萊封包過濾器](https://en.wikipedia.org/wiki/Berkeley_Packet_Filter) (BPF) 尋找解決方案。

### `LD_PRELOAD`

`LD_PRELOAD` 是 Linux 環境變數，允許定義程式執行前預先載入的動態連結庫。

此變數具備兩項優勢：

- 無需關注原始碼即可調用自定義函數

- 可將程式碼注入其他程式以達成特定目的

對於透過應用程式調用 glibc 時間函數的語言（如 Rust 和 C），使用 `LD_PRELOAD` 足以模擬時鐘偏移。但 Golang 更為棘手，因其直接解析虛擬動態共享物件 ([vDSO](http://man7.org/linux/man-pages/man7/vdso.7.html))——加速系統調用的機制。為獲取時間函數地址，無法單純透過 `LD_PRELOAD` 攔截 glibc 介面。因此 `LD_PRELOAD` 非解決方案。

### 使用 BPF 修改 `clock_gettime` 系統呼叫的返回值

我們也嘗試使用 BPF 過濾任務的[行程識別碼](http://www.linfo.org/pid.html) (PID)，藉此在指定行程上模擬時鐘偏移，並修改 `clock_gettime` 系統呼叫的返回值。

這似乎是個好主意，但我們遇到一個問題：在多數情況下，vDSO 會加速 `clock_gettime`，但 `clock_gettime` 本身不會觸發系統呼叫。這個方法同樣行不通。糟糕。

所幸我們發現，若系統核心版本為 4.18 或更高，且使用 [HPET](https://www.kernel.org/doc/html/latest/timers/hpet.html) 時鐘時，`clock_gettime()` 會透過標準系統呼叫（而非 vDSO）取得時間。我們據此實作了[時鐘偏移版本](https://github.com/chaos-mesh/bpfki)，在 Rust 和 C 語言環境運作良好。但對於 Golang，程式雖能正確取得時間，若在時鐘偏移注入期間執行 `sleep` 操作，該操作極可能被阻塞。即使取消注入，系統也無法恢復正常，因此我們最終放棄此方案。

## TimeChaos：我們的最終解法

從前文可知，程式通常透過呼叫 `clock_gettime` 取得系統時間。在我們的場景中，`clock_gettime` 利用 vDSO 加速呼叫過程，因此無法使用 `LD_PRELOAD` 來侵入 `clock_gettime` 系統呼叫。

既然找出癥結點，解法為何？從 vDSO 著手。若能將 vDSO 中儲存 `clock_gettime` 返回值的位址重新導向至我們定義的位址，問題即可迎刃而解。

說來容易做時難。為達成此目標，必須解決以下問題：

- 獲知 vDSO 使用的使用者模式位址

- 獲知 vDSO 的核心模式位址（若需透過核心模式位址修改 vDSO 中的 `clock_gettime` 函數）

- 掌握修改 vDSO 資料的方法

首先需要探查 vDSO 內部。透過 `/proc/pid/maps` 可檢視 vDSO 的記憶體位址。

```
$ cat /proc/pid/maps
...
7ffe53143000-7ffe53145000 r-xp 00000000 00:00 0                     [vdso]
```

最後一行即 vDSO 資訊。此記憶體空間權限標示為 `r-xp`：可讀可執行，但不可寫入。這意味著使用者模式無法修改此記憶體。我們可運用 [ptrace](http://man7.org/linux/man-pages/man2/ptrace.2.html) 突破此限制。

接著使用 `gdb dump memory` 導出 vDSO，並透過 `objdump` 檢視內容，結果如下：

```
(gdb) dump memory vdso.so 0x00007ffe53143000 0x00007ffe53145000
$ objdump -T vdso.so
vdso.so:    file format elf64-x86-64
DYNAMIC SYMBOL TABLE:
ffffffffff700600  w  DF .text   0000000000000545  LINUX_2.6  clock_gettime
```

可見 vDSO 整體結構類似 `.so` 檔案，因此可用可執行與可連結格式 (ELF) 檔案進行解析。據此，TimeChaos 的基礎工作流程逐漸成形：

![TimeChaos 工作流程](/img/blog/timechaos-workflow.jpg)

上圖即 **TimeChaos** 的運作流程——Chaos Mesh 中實現時鐘偏移的核心機制。

1. 使用 ptrace 附加至指定 PID 行程，暫停當前行程。

2. 透過 ptrace 在呼叫行程的虛擬位址空間建立新映射，並運用 [`process_vm_writev`](https://linux.die.net/man/2/process_vm_writev) 將自定義的 `fake_clock_gettime` 函數寫入記憶體空間。

3. 使用 `process_vm_writev` 將指定參數寫入 `fake_clock_gettime`。這些參數即欲注入的時間偏移量（例如倒退兩小時或前進兩天）。

4. 使用 ptrace 修改 vDSO 中的 `clock_gettime` 函數，並將其重定向至 `fake_clock_gettime` 函數。

5. 使用 ptrace 分離 PID 程序。

若您對細節感興趣，請參閱 [Chaos Mesh GitHub 儲存庫](https://github.com/chaos-mesh/chaos-mesh/blob/release-1.0/pkg/time/time_linux.go)。

## 在分散式 SQL 資料庫上模擬時鐘偏移

數據勝於雄辯。我們將在 [TiDB](https://docs.pingcap.com/tidb/stable/overview/) 上嘗試 TimeChaos，這是一個開源、[NewSQL](https://en.wikipedia.org/wiki/NewSQL)、分散式的 SQL 資料庫，支援[混合交易/分析處理](https://en.wikipedia.org/wiki/Hybrid_transactional/analytical_processing) (HTAP) 工作負載，以驗證混沌測試是否真的有效。

TiDB 使用集中式服務 Timestamp Oracle (TSO) 來獲取全域一致的版本號，並確保交易版本號單調遞增。TSO 服務由 Placement Driver (PD) 元件管理。因此，我們選擇一個隨機的 PD 節點，並定期注入 TimeChaos，每次注入 10 毫秒的時鐘回撥。讓我們看看 TiDB 能否應對這一挑戰。

為更好地執行測試，我們使用 [bank](https://github.com/cwen0/bank) 作為工作負載，它模擬銀行系統中的金融轉帳，常用於驗證資料庫交易的正確性。

這是我們的測試配置：

```
apiVersion: chaos-mesh.org/v1alpha1
kind: TimeChaos
metadata:
  name: time-skew-example
  namespace: tidb-demo
spec:
  mode: one
  selector:
    labelSelectors:
      "app.kubernetes.io/component": "pd"
  timeOffset:
    sec: -600
  clockIds:
    - CLOCK_REALTIME
  duration: "10s"
  scheduler:
    cron: "@every 1m"
```

在此測試期間，Chaos Mesh 每 1 毫秒向選定的 PD Pod 注入一次 TimeChaos，持續 10 秒。在此期間，PD 獲取的時間將與實際時間有 600 秒的偏移。更多詳細資訊，請參閱 [Chaos Mesh Wiki](https://github.com/chaos-mesh/chaos-mesh/wiki/Time-Chaos)。

讓我們使用 `kubectl apply` 指令建立一個 TimeChaos 實驗：

```
kubectl apply -f pd-time.yaml
```

現在，我們可以透過以下指令擷取 PD 日誌：

```
kubectl logs -n tidb-demo tidb-app-pd-0 | grep "system time jump backward"
```

日誌如下：

```
[2020/03/24 09:06:23.164 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585041383060109693]
[2020/03/24 09:16:32.260 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585041992160476622]
[2020/03/24 09:20:32.059 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585042231960027622]
[2020/03/24 09:23:32.059 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585042411960079655]
[2020/03/24 09:25:32.059 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585042531963640321]
[2020/03/24 09:28:32.060 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585042711960148191]
[2020/03/24 09:33:32.063 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585043011960517655]
[2020/03/24 09:34:32.060 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585043071959942937]
[2020/03/24 09:35:32.059 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585043131978582964]
[2020/03/24 09:36:32.059 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585043191960687755]
[2020/03/24 09:38:32.060 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585043311959970737]
[2020/03/24 09:41:32.060 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585043491959970502]
[2020/03/24 09:45:32.061 +00:00] [ERROR] [systime_mon.go:32] ["system time jump backward"] [last=1585043731961304629]
...
```

從上方的日誌可見，PD 會定期檢測到系統時間回撥。這意味著：

- TimeChaos 成功模擬了時鐘偏移

- PD 能夠處理時鐘偏移的情況

這令人鼓舞。但 TimeChaos 是否會影響 PD 以外的服務？我們可以在 Chaos Dashboard 中查看：

![Chaos Dashboard](/img/blog/chaos-dashboard.jpg)

監控畫面清晰顯示：TimeChaos 每 1 毫秒注入一次，整個持續時間為 10 秒。更重要的是，TiDB 並未受到注入影響——bank 程式正常運行，效能也未受影響。

## 試用 Chaos Mesh

作為雲原生混沌工程平台，Chaos Mesh 具備全方位的 [Kubernetes 複雜系統故障注入方法](https://pingcap.com/blog/chaos-mesh-your-chaos-engineering-solution-for-system-resiliency-on-kubernetes/)，涵蓋 Pod、網路、檔案系統，甚至核心層級的故障。

想要獲得混沌工程的實戰經驗嗎？歡迎造訪 [Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh)。這個 [10 分鐘教學](https://pingcap.com/blog/run-first-chaos-experiment-in-ten-minutes/)將幫助您快速入門混沌工程，並使用 Chaos Mesh 運行您的第一個混沌實驗。
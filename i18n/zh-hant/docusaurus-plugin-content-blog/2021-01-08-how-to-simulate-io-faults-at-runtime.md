---
slug: /how-to-simulate-io-faults-at-runtime
title: 'How to Simulate I/O Faults at Runtime'
authors: keaoyang
image: /img/blog/how-to-simulate-io-faults-at-runtime.jpg
tags: [Chaos Mesh, Chaos Engineering, Fault Injection]
---

![混沌工程 - 如何在運行時模擬 I/O 故障](/img/blog/how-to-simulate-io-faults-at-runtime.jpg)

在生產環境中，檔案系統故障可能因多種事件而發生，例如磁碟故障和管理員錯誤。作為混沌工程平台，Chaos Mesh 從早期版本就支援模擬檔案系統中的 I/O 故障。只需新增 IOChaos 自訂資源定義（CRD），我們就能觀察檔案系統如何失敗並回傳錯誤。

<!--truncate-->

然而在 Chaos Mesh 1.0 之前，此實驗並不容易且可能消耗大量資源。我們需透過突變許可 Webhook 將邊車容器注入 Pod，並重寫 `ENTRYPOINT` 指令。即使未注入故障，注入的邊車容器也會造成顯著系統開銷。

Chaos Mesh 1.0 改變了這一切。現在我們可使用 IOChaos 在運行時將故障注入檔案系統，簡化流程並大幅降低系統開銷。本文將介紹如何在不使用邊車的情況下實作 IOChaos 實驗。

## I/O 故障注入

為在運行時模擬 I/O 故障，需在程式啟動[系統呼叫](https://man7.org/linux/man-pages/man2/syscall.2.html)（如讀寫）後、但請求抵達目標檔案系統前注入故障。我們有兩種實現方式：

- 使用柏克萊封包過濾器（BPF）；但其[無法注入延遲](https://github.com/iovisor/bcc/issues/2336)。

- 在目標檔案系統前新增名為 ChaosFS 的檔案系統層。ChaosFS 以目標檔案系統為後端接收作業系統請求，完整呼叫鏈為**目標程式系統呼叫** -> **Linux 核心** -> **ChaosFS** -> **目標檔案系統**。因 ChaosFS 可客製化，我們能任意注入延遲與錯誤，故選擇此方案。

但 ChaosFS 存在多項問題：

- 若 ChaosFS 需讀寫目標檔案系統檔案，必須將 ChaosFS [掛載](https://man7.org/linux/man-pages/man2/mount.2.html)至與 Pod 設定目標路徑不同的位置。ChaosFS **無法**直接掛載至目標目錄路徑。

- 需**在**目標程式開始運行**前**掛載 ChaosFS。因新掛載的 ChaosFS 僅對程式在目標檔案系統中新開啟的檔案生效。

- 需將 ChaosFS 掛載至目標容器的 `mnt` 命名空間。詳見 [mount_namespaces(7) — Linux 手冊頁](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html)。

Chaos Mesh 1.0 前，我們使用[突變許可 Webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) 實作 IOChaos，此技術解決上述三問題並實現：

- 在目標容器執行指令碼：變更 ChaosFS 後端檔案系統目標目錄（如將 `/mnt/a` 改為 `/mnt/a_bak`），使 ChaosFS 能掛載至目標路徑（`/mnt/a`）。修改 Pod 啟動指令，如將原始指令 `/app` 改為 `/waitfs.sh /app`。

- `waitfs.sh` 指令碼持續檢查檔案系統是否成功掛載，掛載成功後才啟動 `/app`。

- 在 Pod 新增執行 ChaosFS 的容器：此容器需與目標容器共享卷（如 `/mnt`），並將卷掛載至目標目錄（如 `/mnt/a`）。同時啟用[掛載傳播](https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation)，使卷掛載能穿透至主機再傳遞至目標。

這三種方法讓我們能在程式運行時注入 I/O 故障，但注入過程遠非便捷：

- We could only inject faults into a volume subdirectory, not into the entire volume. The workaround was to replace `mv` (rename) with `mount move` to move the mount point of the target volume.

- 必須在 Pod 中明確寫入指令，而非隱式使用鏡像指令。否則 `/waitfs.sh` 腳本無法在檔案系統掛載後正確啟動程式。

- 對應容器需正確配置掛載傳播。基於潛在隱私與安全問題，我們**無法**透過突變准入 Webhook 修改配置。

- 注入配置過程繁瑣，更糟的是必須在配置生效後創建新的 Pod 才能注入故障。

- 無法在程式運行時撤回 ChaosFS。即使未注入任何故障或錯誤，效能仍會大幅下降。

## 無需突變准入 Webhook 注入 I/O 故障

若不使用突變准入 Webhook 該如何解決這些難題？讓我們回溯思考當初為何使用突變准入 Webhook 添加運行 ChaosFS 的容器——是為了將檔案系統掛載至目標容器。

事實上另有解法：無需向 Pod 添加容器，我們可先用 `setns` Linux 系統呼叫修改當前行程的命名空間，再透過 `mount` 呼叫將 ChaosFS 掛載至目標容器。假設要注入的檔案系統是 `/mnt`，新注入流程如下：

1. 使用 `setns` 讓當前行程進入目標容器的 mnt 命名空間。

2. 執行 `mount --move` 將 `/mnt` 移動至 `/mnt_bak`。

3. 將 ChaosFS 掛載到 `/mnt` 並使用 `/mnt_bak` 作為後端。

完成後，目標容器將透過 ChaosFS 開啟、讀取和寫入 `/mnt` 中的檔案，延遲或故障注入因此大幅簡化。但仍有兩個問題待解：

- 如何處理目標行程已開啟的檔案？

- 如何在檔案開啟狀態下（無法卸載檔案系統）恢復行程？

### 動態替換檔案描述符

**ptrace 可同時解決上述兩個問題**。我們能透過 ptrace 在運行時替換已開啟的檔案描述符 (FD)，並替換當前工作目錄 (CWD) 與 mmap。

#### 使用 ptrace 讓追蹤對象執行二進位程式

[ptrace](https://man7.org/linux/man-pages/man2/ptrace.2.html) 是強大工具，可使目標行程（追蹤對象）執行任意系統呼叫或二進位程式。為讓追蹤對象執行程式，ptrace 會修改 RIP 指向的地址至目標行程，並添加 `int3` 指令觸發中斷點。當二進位程式停止時，需恢復暫存器與記憶體狀態。

> **注意：**
>
> 在 [x86_64 架構](https://en.wikipedia.org/wiki/X86_assembly_language)中，RIP 暫存器（亦稱指令指標）始終指向下個待執行指令的記憶體位址。要將程式載入目標行程記憶體空間：

1. 使用 ptrace 呼叫目標程式中的 mmap 來分配所需記憶體。

2. 將二進位程式寫入新分配的記憶體，並使 RIP 暫存器指向該位置。

3. 二進位程式停止後，呼叫 munmap 清理記憶體區段。

最佳實踐是將 ptrace 的 `POKE_TEXT` 寫入替換為 `process_vm_writev`，因為當需寫入大量資料時，`process_vm_writev` 效能更佳。

使用 ptrace，我們能夠讓一個進程替換自身的檔案描述符（FD）。現在我們只需要一種方法來實現這個替換操作，這個方法就是 `dup2` 系統呼叫。

#### 使用 `dup2` 替換檔案描述符

`dup2` 函數的簽名為 `int dup2(int oldfd, int newfd);`。它用於建立舊 FD (`oldfd`) 的副本，該副本的 FD 編號為 `newfd`。如果 `newfd` 已對應某個已開啟檔案的 FD，系統會自動關閉該已開啟檔案的 FD。

例如，當前進程開啟了 `/var/run/__chaosfs__test__/a`，其 FD 為 `1`。要將此開啟的檔案替換為 `/var/run/test/a`，該進程需執行以下操作：

1. Uses the `fcntl` system call to get the `OFlags` (the parameter used by the `open` system call, such as `O_WRONLY`) of `/var/run/__chaosfs__test__/a`.

2. Uses the `Iseek` system call to get the current location of `seek`.

3. Uses the `open` system call to open `/var/run/test/a` using the same `OFlags`. Assume that the FD is `2`.

4. Uses `Iseek` to change the `seek` location of the newly opened FD `2`.

5. Uses `dup2(2, 1)` to replace the FD `1` of `/var/run/__chaosfs__test__/a` with the newly opened FD `2`.

6. 關閉 FD `2`。

完成後，當前進程的 FD `1` 將指向 `/var/run/test/a`。如此便能注入故障，後續對目標檔案的操作都會經過[用戶空間檔案系統](https://en.wikipedia.org/wiki/Filesystem_in_Userspace)（FUSE）。FUSE 是 Unix 及類 Unix 作業系統的軟體介面，允許非特權使用者無需修改核心程式碼即可建立自有檔案系統。

#### 編寫程式使目標進程替換自身檔案描述符

結合 ptrace 與 dup2 的功能，追蹤程式（tracer）能讓被追蹤程式（tracee）自行替換已開啟的 FD。現在我們需要編寫一個二進位程式，並讓目標進程執行它：

> **注意：**
>
> 上述實作基於以下假設：
>
> - 目標進程的執行緒均為 POSIX 執行緒，且共享已開啟檔案。
> - 目標進程使用 `clone` 函數建立執行緒時，傳遞了 `CLONE_FILES` 參數。
>
> 因此，Chaos Mesh 僅替換執行緒群組中首個執行緒的 FD。

1. 根據前兩節內容及 syscall 指令用法編寫組語程式碼。[此處](https://github.com/chaos-mesh/toda/blob/1d73871d8ab72b8d1eace55f5222b01957193531/src/replacer/fd_replacer.rs#L133)提供組語程式碼範例。

2. 使用組譯器將程式碼轉譯為二進位程式。我們採用 [dynasm-rs](https://github.com/CensoredUsername/dynasm-rs) 作為組譯器。

3. 透過 ptrace 使目標進程執行此程式。程式執行時，FD 將在運行時被替換。

### 整體故障注入流程

下圖展示完整的 I/O 故障注入流程：

![Fault injection process](/img/blog/fault-injection-process.jpg)

<div style={{ margin: '1rem 0', fontStyle: 'italic', textAlign: 'center' }}> Fault injection process </div>

在此圖中，每條水平線對應一個沿箭頭方向運行的執行緒。**掛載/卸載檔案系統**和**替換檔案描述符（FD）**的任務被仔細地按順序排列。鑑於上述過程，這種安排非常合理。

## 接下來

我已經討論了我們如何在運行時實作故障注入以模擬 I/O 故障（請參閱 [chaos-mesh/toda](https://github.com/chaos-mesh/toda)）。然而，目前的實作還遠未完美：

- 不支援世代編號（generation numbers）

- 不支援 ioctl 系統呼叫

- Chaos Mesh 不會立即判定檔案系統是否成功掛載，而只會在一秒後才進行判定

如果您對 Chaos Mesh 感興趣並希望協助我們改進，歡迎加入 [我們的 Slack 頻道](https://slack.cncf.io/) 或提交拉取請求（pull requests）及問題至我們的 [GitHub 儲存庫](https://github.com/chaos-mesh/chaos-mesh)

這是 Chaos Mesh 實作系列文章的第一篇。若您想了解其他類型的故障注入如何實作，請持續關注。
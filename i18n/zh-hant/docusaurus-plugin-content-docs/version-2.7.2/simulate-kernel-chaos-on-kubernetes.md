---
title: Simulate Linux Kernel Faults
---

本文檔描述如何使用 KernelChaos 模擬 Linux 核心故障。此功能透過 [BPF](https://lore.kernel.org/lkml/20171213180356.hsuhzoa7s4ngro2r@destiny/T/) 將基於 I/O 或記憶體的故障注入指定核心路徑。

儘管您可以將 KernelChaos 的注入目標設定為單一或多個 Pod，但主機上其他 Pod 的效能仍會受到影響，因為所有 Pod 共享同一個核心。

:::warning

Linux 核心故障模擬功能預設為停用狀態。請勿在生產環境中使用此功能。

:::

## 先決條件

- Linux 核心版本 >= 4.18

- Linux 核心配置 [CONFIG_BPF_KPROBE_OVERRIDE](https://cateee.net/lkddb/web-lkddb/BPF_KPROBE_OVERRIDE.html) 已啟用

- The `bpfki.create` configuration value in [values.yaml](https://github.com/chaos-mesh/chaos-mesh/blob/master/helm/chaos-mesh/values.yaml) is `true`.

## 配置檔案

以下是簡單的 KernelChaos 配置檔案範例：

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: KernelChaos
metadata:
  name: kernel-chaos-example
  namespace: chaos-mesh
spec:
  mode: one
  selector:
    namespaces:
      - chaos-mount
  failKernRequest:
    callchain:
      - funcname: '__x64_sys_mount'
    failtype: 0
```

更多配置範例請參閱 [examples](https://github.com/chaos-mesh/chaos-mesh/tree/master/examples)，您可根據需求修改這些配置範例。

配置說明：

- **mode** 指定實驗執行模式，選項如下：

  - `one`：隨機選擇一個符合條件的 Pod
  - `all`：選擇所有符合條件的 Pod
  - `fixed`：選擇指定數量的符合條件 Pod
  - `fixed-percent`：選擇指定百分比的符合條件 Pod
  - `random-max-percent`：選擇符合條件 Pod 的最大百分比

- **selector** 指定要注入故障的目標 Pod

- **FailedkernRequest** 指定故障模式（如 kmallo 和 bio），同時定義特定調用鏈路徑及可選過濾條件。配置項目如下：

  - **Failtype** 指定故障類型，可選值為：

    - '0': 注入 slab 分配錯誤 should_failslab
    - '1': 注入記憶體分頁分配錯誤 should_fail_alloc_page
    - '2': 注入 bio 錯誤 should_fail_bio

    有關這三種故障類型的詳細資訊，請參閱 [fault-injection](https://www.kernel.org/doc/html/latest/fault-injection/fault-injection.html) 和 [inject_example](http://github.com/iovisor/bcc/blob/master/tools/inject_example.txt)。

  - **Callchain** 指定特定調用鏈，例如：

    ```c
    ext4_mount
    -> mount_subtree
    -> ...
        -> should_failslab
    ```

    您亦可使用函數參數作為過濾規則，以注入更細粒度的故障。詳見 [call chain and predicate examples](https://github.com/chaos-mesh/bpfki/tree/develop/examples)。若未指定調用鏈，請保持 `callchain` 欄位為空，表示故障將注入任何調用 slab alloc 的路徑（例如 kmallo）。

    調用鏈類型為幀陣列，包含以下三部分：

    - **funcname**：可從核心原始碼或 `/proc/kallsyms` 中查找，例如 `ext4_mount`
    - **parameters**：用於過濾。若要在 `d_alloc_parallel(struct dentry *parent, const struct qstr *name)` 路徑上針對特定名稱 `bananas` 注入 slab 錯誤，需將 `parameters` 設為 `struct dentry *parent, const struct qstr *name`，否則省略此配置
    - **predicate**：用於存取幀陣列參數。以 **parameters** 為例，可設定為 `STRNCMP(name->name, "bananas", 8)` 來控制故障注入路徑，或留空表示所有執行 `d_allo_parallel` 的調用路徑均接受 slab 故障注入

  - **headers** 指定所需的核心標頭檔，例如 "linux/mmzone.h" 和 "linux/blkdev.h"
  - **probability** 設定故障發生機率，如設定 '1' 表示 1% 機率
  - **times** 設定故障觸發次數上限

## 使用 kubectl 創建實驗

使用 `kubectl` 創建實驗：

```bash
kubectl apply -f KernelChaos
```

KernelChaos 功能類似於 [inject.py](https://github.com/iovisor/bcc/blob/master/tools/inject.py)，詳細資訊請參閱 [input_example.txt](https://github.com/iovisor/bcc/blob/master/tools/inject_example.txt)

簡單範例如下：

```c
#include <sys/mount.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>

int main(void) {
  int ret;
  while (1) {
    ret = mount("/dev/sdc", "/mnt", "ext4",
          MS_MGC_VAL | MS_RDONLY | MS_NOSUID, "");
    if (ret < 0)
      fprintf(stderr, "%s\n", strerror(errno));
    sleep(1);
    ret = umount("/mnt");
    if (ret < 0)
      fprintf(stderr, "%s\n", strerror(errno));
  }
}
```

故障注入期間輸出如下：

```
> Cannot allocate memory
> Invalid argument
> Cannot allocate memory
> Invalid argument
> Cannot allocate memory
> Invalid argument
> Cannot allocate memory
> Invalid argument
> Cannot allocate memory
> Invalid argument
```

## 使用限制

您可使用 container_id 限制故障注入範圍，但某些路徑會觸發系統級行為，例如：

當 `failtype` 為 `1` 時，表示實體分頁分配失敗。若短時間內頻繁觸發此事件（例如 `while (1) {memset(malloc(1M), '1', 1M)}`），將觸發 oom-killer 系統調用來回收記憶體
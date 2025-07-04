---
title: Simulate Linux Kernel Faults
---

本文件說明如何使用 KernelChaos 模擬 Linux 核心錯誤。此功能透過 [BPF](https://lore.kernel.org/lkml/20171213180356.hsuhzoa7s4ngro2r@destiny/T/) 將基於 I/O 或記憶體的錯誤注入指定核心路徑。

儘管您可以將 KernelChaos 的注入目標設定為單一或多個 Pod，但主機上其他 Pod 的效能仍會受到影響，因為所有 Pod 共享同一個核心。

:::warning

Linux 核心錯誤模擬功能預設為停用狀態。請勿在生產環境中使用此功能。

:::

## 先決條件

- Linux 核心版本 >= 4.18

- 已啟用 Linux 核心配置 [CONFIG_BPF_KPROBE_OVERRIDE](https://cateee.net/lkddb/web-lkddb/BPF_KPROBE_OVERRIDE.html)

- The `bpfki.create` configuration value in [values.yaml](https://github.com/chaos-mesh/chaos-mesh/blob/master/helm/chaos-mesh/values.yaml) is `true`.

## 設定檔

簡單的 KernelChaos 設定檔範例如下：

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

更多設定範例請參閱 [examples](https://github.com/chaos-mesh/chaos-mesh/tree/master/examples)，您可根據需求修改這些設定範例。

設定說明：

- **mode** 指定實驗執行方式，選項如下：

  - `one`: 隨機選擇一個符合條件的 Pod
  - `all`: 選擇所有符合條件的 Pod
  - `fixed`: 選擇指定數量的符合條件 Pod
  - `fixed-percent`: 選擇指定百分比的符合條件 Pod
  - `random-max-percent`: 選擇最大百分比的符合條件 Pod

- **selector** 指定要注入錯誤的目標 Pod

- **FailedkernRequest** specifies the fault mode (such as kmallo and bio). It also specifies a specific call chain path and the optional filtering conditions. The configuration items are as follows:

  - **Failtype** specifies the fault type. The value options are as follows:

    - '0': injects the slab allocation error should_failslab.
    - '1': injects the memory page allocation error should_fail_alloc_page.
    - '2': injects the bio error should_fail_bio.

    For more information on these three fault types, refer to [fault-injection](https://www.kernel.org/doc/html/latest/fault-injection/fault-injection.html) and [inject_example](http://github.com/iovisor/bcc/blob/master/tools/inject_example.txt).

  - **Callchain** specifies a specific call chain. For example:

    ```c
    ext4_mount
    -> mount_subtree
    -> ...
        -> should_failslab
    ```

    You can also use the function parameters as filtering rules to inject more fine-grained faults. Refer to [call chain and predicate examples](https://github.com/chaos-mesh/bpfki/tree/develop/examples) for more information. If no call chain is specified, keep the `callchain` field empty, indicating that faults will be injected to any path on which slab alloc is called (for example, kmallo).

    The call chain type is a frame array, consisting of the following three parts:

    - **funcname**, which can be found from the kernel source code or from `/proc/kallsyms`, such as `ext4_mount`.
    - **parameters**, which is used for filtering. If you want to inject a slab error on the `d_alloc_parallel(struct dentry *parent, const struct qstr *name)` with a special name `bananas` path, you need to set the `parameters` to `struct dentry *parent, const struct qstr *name`. Otherwise, omit this configuration.
    - **predicate**, which is used to access the parameters of the frame array. Taking **parameters** as an example, you can set it to `STRNCMP(name->name, "bananas", 8)` to control the path of fault injection, or you can leave it empty for all call paths that execute `d_allo_parallel` receive the slab fault injection.

  - **headers** specifies the kernel header file you need. For example, "linux/mmzone.h" and "linux/blkdev.h".
  - **probability** specifies the probability of faults. If you want the probability of 1%, set to '1'.
  - **times** specifies the maximum number of times a fault is triggered.

## 使用 kubectl 建立實驗

使用 `kubectl` 建立實驗：

```bash
kubectl apply -f KernelChaos
```

KernelChaos 功能類似 [inject.py](https://github.com/iovisor/bcc/blob/master/tools/inject.py)，詳細資訊請參閱 [input_example.txt](https://github.com/iovisor/bcc/blob/master/tools/inject_example.txt)。

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

故障注入期間的輸出如下：

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

您可透過 container_id 限制故障注入範圍，但部分路徑會觸發系統層級行為，例如：

當 `failtype` 為 `1` 時表示實體頁面分配失敗。若短時間內頻繁觸發此事件（例如 `while (1) {memset(malloc(1M), '1', 1M)}`），將觸發 oom-killer 系統呼叫進行記憶體回收。
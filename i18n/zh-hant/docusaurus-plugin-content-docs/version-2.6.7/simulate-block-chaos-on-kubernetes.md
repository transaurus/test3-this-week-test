---
title: Simulate Block Device Incidents
---

## BlockChaos 簡介

Chaos Mesh 提供 BlockChaos 實驗類型。您可以使用此實驗類型模擬區塊裝置延遲或凍結情境。本文檔描述如何安裝 BlockChaos 實驗的依賴項，以及建立 BlockChaos。

:::note

BlockChaos 目前處於早期階段。其安裝和配置體驗將持續改進。如果您發現任何問題，請在 [chaos-mesh/chaos-mesh](https://github.com/chaos-mesh/chaos-mesh) 開一個 issue 回報。

:::

:::note

BlockChaos 的 `freeze` 操作會影響使用該區塊裝置的所有程序，而不僅是目標容器。

:::

## 安裝核心模組

BlockChaos 的 `delay` 操作依賴於 [chaos-driver](https://github.com/chaos-mesh/chaos-driver) 核心模組。該操作只能在安裝了此模組的機器上注入。目前，您必須手動編譯並安裝此模組。

1. 使用以下指令下載此模組的原始碼：

   ```bash
   curl -fsSL -o chaos-driver-v0.2.1.tar.gz https://github.com/chaos-mesh/chaos-driver/archive/refs/tags/v0.2.1.tar.gz
   ```

2. 解壓縮 `chaos-driver-v0.2.1.tar.gz` 檔案：

   ```bash
   tar xvf chaos-driver-v0.2.1.tar.gz
   ```

3. 準備您當前核心的標頭檔。如果您使用 CentOS/Fedora，可以使用 `yum` 安裝核心標頭檔：

   ```bash
   yum install kernel-devel-$(uname -r)
   ```

   如果您使用 Ubuntu/Debian，可以使用 `apt` 安裝核心標頭檔：

   ```bash
   apt install linux-headers-$(uname -r)
   ```

4. 編譯模組：

   ```bash
   cd chaos-driver-v0.2.1
   make driver/chaos_driver.ko
   ```

5. 安裝核心模組：

   ```bash
   insmod ./driver/chaos_driver.ko
   ```

`chaos_driver` 模組必須在每次重啟後重新安裝。若要自動載入模組，您可以將模組複製到 `/lib/modules/$(uname -r)/kernel/drivers` 的子目錄中，執行 `depmod -a`，然後將 `chaos_driver` 加入 `/etc/modules`。

如果您升級了核心，則應重新編譯此模組。

:::note

建議使用 DKMS 或 akmod 來自動編譯或載入核心模組。如果您想幫助我們改善安裝體驗，非常歡迎建立 DKMS 或 akmod 套件並提交至不同的發行版儲存庫。

:::

## 使用 YAML 檔案建立實驗

1. 將實驗配置寫入 YAML 設定檔。以下以 `block-latency.yaml` 檔案為例：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: BlockChaos
   metadata:
     name: hostpath-example-delay
   spec:
     selector:
       labelSelectors:
         app: hostpath-example
     mode: all
     volumeName: hostpath-example
     action: delay
     delay:
       latency: 1s
   ```

   :::note

   僅支援 hostpath 或 localvolume。

   :::

2. 使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f block-latency.yaml
   ```

您會發現以下神奇的事情發生：

1. 該卷的電梯演算法被更改為 `ioem` 或 `ioem-mq`。您可以透過 `cat /sys/block/<device>/queue/scheduler` 來檢查。

2. `ioem` 或 `ioem-mq` 排程器將接收延遲請求，並將請求延遲指定的時間。

YAML 設定檔中的欄位描述如下表所示：

| Parameter | Type | Note | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| `mode` | string | Specifies the mode of the experiment. The mode options include `one` (selecting a random Pod), `all` (selecting all eligible Pods), `fixed` (selecting a specified number of eligible Pods), `fixed-percent` (selecting a specified percentage of Pods from the eligible Pods), and `random-max-percent` (selecting the maximum percentage of Pods from the eligible Pods). | None | Yes | `one` |
| `value` | string | Provides parameters for the `mode` configuration, depending on `mode`. For example, when `mode` is set to `fixed-percent`, `value` specifies the percentage of Pods. | None | No | `1` |
| `selector` | struct | Specifies the target Pod. For details, refer to [Define the experiment scope](./define-chaos-experiment-scope.md). | None | Yes |  |
| `volumeName` | string | Specifies the volume to inject in the target pods. There should be a corresponding entry in the pods' `.spec.volumes`. | None | Yes | `hostpath-example` |
| `action` | string | Indicates the specific type of faults. The available fault types include `delay` and `freeze`. `delay` will simulate the latency of block devices, and `freeze` will simulate that the block device cannot handle any requests | None | Yes | `delay` |
| `delay.latency` | string | Specifies the latency of the block device. | None | Yes (if `action` is `delay`) | `500ms` |
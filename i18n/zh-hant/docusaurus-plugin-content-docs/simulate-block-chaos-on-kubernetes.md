---
title: Simulate Block Device Incidents
---

## BlockChaos 簡介

Chaos Mesh 提供 BlockChaos 實驗類型，可用於模擬區塊裝置延遲或凍結情境。本文檔說明如何安裝 BlockChaos 實驗的依賴項並建立 BlockChaos 實驗。

:::note

BlockChaos 目前處於早期階段，其安裝與配置體驗將持續改進。若發現任何問題，請至 [chaos-mesh/chaos-mesh](https://github.com/chaos-mesh/chaos-mesh) 提交 issue 回報。

:::

:::note

BlockChaos 的 `freeze` 操作將影響所有使用該區塊裝置的進程，不僅限於目標容器。

:::

## 安裝核心模組

BlockChaos 的 `delay` 操作需依賴 [chaos-driver](https://github.com/chaos-mesh/chaos-driver) 核心模組，僅能在已安裝此模組的機器上注入。目前需手動編譯並安裝該模組。

1. 使用以下指令下載模組原始碼：

   ```bash
   curl -fsSL -o chaos-driver-v0.2.1.tar.gz https://github.com/chaos-mesh/chaos-driver/archive/refs/tags/v0.2.1.tar.gz
   ```

2. 解壓縮 `chaos-driver-v0.2.1.tar.gz` 檔案：

   ```bash
   tar xvf chaos-driver-v0.2.1.tar.gz
   ```

3. 準備當前核心的標頭檔：
   - CentOS/Fedora 系統使用 `yum` 安裝：
   
     ```bash
     yum install kernel-devel-$(uname -r)
     ```

   - Ubuntu/Debian 系統使用 `apt` 安裝：

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

The `chaos_driver` module has to be installed every time after rebooting. To load the module automatically, you can copy the module to a subdirectory in `/lib/modules/$(uname -r)/kernel/drivers`, run `depmod -a`, and then add `chaos_driver` to the `/etc/modules`.

若升級了核心，需重新編譯此模組。

:::note

建議使用 DKMS 或 akmod 實現核心模組的自動編譯與載入。若您願意協助改善安裝體驗，歡迎建立 DKMS 或 akmod 套件並提交至各發行版的套件庫。

:::

## 使用 YAML 檔案建立實驗

1. 將實驗配置寫入 YAML 檔案，以下以 `block-latency.yaml` 為例：

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

   僅支援 hostpath 或 localvolume 儲存類型。

   :::

2. 使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f block-latency.yaml
   ```

您將觀察到以下特殊現象：

1. 儲存卷的調度器變更為 `ioem` 或 `ioem-mq`，可透過 `cat /sys/block/<device>/queue/scheduler` 指令確認。

2. `ioem` 或 `ioem-mq` 排程器會接收延遲請求，並將請求延遲指定的時間。

YAML 設定檔中的欄位說明如下表：

| Parameter | Type | Note | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| `mode` | string | Specifies the mode of the experiment. The mode options include `one` (selecting a random Pod), `all` (selecting all eligible Pods), `fixed` (selecting a specified number of eligible Pods), `fixed-percent` (selecting a specified percentage of Pods from the eligible Pods), and `random-max-percent` (selecting the maximum percentage of Pods from the eligible Pods). | None | Yes | `one` |
| `value` | string | Provides parameters for the `mode` configuration, depending on `mode`. For example, when `mode` is set to `fixed-percent`, `value` specifies the percentage of Pods. | None | No | `1` |
| `selector` | struct | Specifies the target Pod. For details, refer to [Define the experiment scope](./define-chaos-experiment-scope.md). | None | Yes |  |
| `volumeName` | string | Specifies the volume to inject in the target pods. There should be a corresponding entry in the pods' `.spec.volumes`. | None | Yes | `hostpath-example` |
| `action` | string | Indicates the specific type of faults. The available fault types include `delay` and `freeze`. `delay` will simulate the latency of block devices, and `freeze` will simulate that the block device cannot handle any requests | None | Yes | `delay` |
| `delay.latency` | string | Specifies the latency of the block device. | None | Yes (if `action` is `delay`) | `500ms` |
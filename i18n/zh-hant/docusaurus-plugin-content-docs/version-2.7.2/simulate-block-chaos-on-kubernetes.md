---
title: Simulate Block Device Incidents
---

## BlockChaos 簡介

Chaos Mesh 提供了 BlockChaos 實驗類型，可用於模擬區塊裝置的延遲或凍結情境。本文檔說明如何安裝 BlockChaos 實驗的依賴項並建立 BlockChaos 實驗。

:::note

BlockChaos 目前處於早期階段，其安裝和配置體驗將持續改進。若發現任何問題，請在 [chaos-mesh/chaos-mesh](https://github.com/chaos-mesh/chaos-mesh) 提交 issue 回報。

:::

:::note

BlockChaos 的 `freeze` 操作會影響所有使用該區塊裝置的進程，不僅限於目標容器。

:::

## 安裝核心模組

BlockChaos 的 `delay` 操作依賴於 [chaos-driver](https://github.com/chaos-mesh/chaos-driver) 核心模組，此操作僅能在已安裝該模組的機器上注入。目前需手動編譯並安裝此模組。

1. 使用以下指令下載模組原始碼：

   ```bash
   curl -fsSL -o chaos-driver-v0.2.1.tar.gz https://github.com/chaos-mesh/chaos-driver/archive/refs/tags/v0.2.1.tar.gz
   ```

2. 解壓縮 `chaos-driver-v0.2.1.tar.gz` 檔案：

   ```bash
   tar xvf chaos-driver-v0.2.1.tar.gz
   ```

3. 準備當前核心標頭檔：
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

`chaos_driver` 模組需在每次重啟後重新安裝。若要自動載入，可將模組複製至 `/lib/modules/$(uname -r)/kernel/drivers` 子目錄，執行 `depmod -a`，並將 `chaos_driver` 加入 `/etc/modules`。

若升級核心，需重新編譯該模組。

:::note

建議使用 DKMS 或 akmod 實現核心模組自動編譯與載入。歡迎貢獻 DKMS/akmod 套件至各發行版儲存庫以改善安裝體驗。

:::

## 使用 YAML 檔案建立實驗

1. 將實驗配置寫入 YAML 檔案（以 `block-latency.yaml` 為例）：

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

您將觀察到以下現象：

1. 該卷的 I/O 排程器將變更為 `ioem` 或 `ioem-mq`，可透過 `cat /sys/block/<device>/queue/scheduler` 驗證。

2. `ioem` 或 `ioem-mq` 排程器將會接收延遲請求，並將請求延遲指定的時間。

YAML 設定檔中的欄位如下表所述：

| Parameter | Type | Note | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| `mode` | string | Specifies the mode of the experiment. The mode options include `one` (selecting a random Pod), `all` (selecting all eligible Pods), `fixed` (selecting a specified number of eligible Pods), `fixed-percent` (selecting a specified percentage of Pods from the eligible Pods), and `random-max-percent` (selecting the maximum percentage of Pods from the eligible Pods). | None | Yes | `one` |
| `value` | string | Provides parameters for the `mode` configuration, depending on `mode`. For example, when `mode` is set to `fixed-percent`, `value` specifies the percentage of Pods. | None | No | `1` |
| `selector` | struct | Specifies the target Pod. For details, refer to [Define the experiment scope](./define-chaos-experiment-scope.md). | None | Yes |  |
| `volumeName` | string | Specifies the volume to inject in the target pods. There should be a corresponding entry in the pods' `.spec.volumes`. | None | Yes | `hostpath-example` |
| `action` | string | Indicates the specific type of faults. The available fault types include `delay` and `freeze`. `delay` will simulate the latency of block devices, and `freeze` will simulate that the block device cannot handle any requests | None | Yes | `delay` |
| `delay.latency` | string | Specifies the latency of the block device. | None | Yes (if `action` is `delay`) | `500ms` |
---
title: Chaosctl
---

Chaosctl 是一個協助除錯 Chaos Mesh 的工具。透過 Chaosctl，您可以簡化開發和除錯新混沌類型的流程，並在提出問題時為其他開發者提供參考。

## 取得 Chaosctl

Linux 使用者可直接下載 Chaosctl 的可執行檔。

```bash
curl -sSL https://mirrors.chaos-mesh.org/latest/chaosctl -O
```

Windows 或 macOS 使用者可從原始碼編譯，建議使用 Go v1.15 或更高版本進行編譯。請執行以下步驟：

1. 將 Chaos Mesh 儲存庫複製到本機。

   ```bash
   git clone https://github.com/chaos-mesh/chaos-mesh.git
   ```

2. 切換至 Chaos Mesh 目錄。

3. 執行以下指令：

   ```bash
   make chaosctl
   ```

   編譯後的可執行檔位於 `bin/chaosctl`。

## 功能

目前 Chaosctl 支援列印混沌實驗的日誌和除錯資訊。

### 列印日誌

使用 `chaosctl logs` 指令可列印所有 Chaos Mesh 元件的日誌。欲查看此功能的說明資訊與範例，請使用 `chaosctl logs -h` 指令。範例指令如下：

```bash
chaosctl logs -t 100 # Print the last 100 lines of logs from all components
```

### 列印除錯資訊

使用 `chaosctl debug` 指令可列印除錯資訊。欲查看此功能的說明資訊與範例，請使用 `chaosctl debug -h` 指令。除錯時需確保 Chaosctl 已連接對應的 `chaos-daemon`。若在部署 Chaos Mesh 時停用了 TLS（預設啟用），請添加 `-i` 選項告知 Chaosctl 未使用 TLS。範例指令如下：

```bash
./chaosctl debug -i networkchaos web-show-network-delay
```

目前 Chaosctl 僅支援 IOChaos、NetworkChaos 和 StressChaos 的除錯。

### 為 Chaosd 產生 TLS 憑證

當 Chaosd 與 Chaos Mesh 之間發起請求時，為確保 Chaosd 與 Chaos-controller-manager 服務間的通訊安全，Chaos Mesh 建議啟用 mTLS（雙向傳輸層安全）模式。

啟用 mTLS 模式需在 Chaosd 和 Chaos Mesh 中設定 TLS 憑證參數。因此請確保 Chaosd 和 Chaos Mesh 已產生 TLS 憑證，再以 TLS 憑證作為參數啟動服務。

- Chaosd：您可在設定 TLS 憑證參數**前**或**後**啟動 Chaosd。基於叢集安全考量，建議先設定 TLS 憑證參數再啟動 Chaosd。詳情請參閱[部署 Chaosd 伺服器](simulate-physical-machine-chaos.md#deploy-chaosd-server)。

- Chaos Mesh：若使用 Helm 部署 Chaos Mesh，則預設已設定 TLS 憑證參數。

若您的 Chaosd 未產生 TLS 憑證，可使用 Chaosctl 透過指令列輕鬆產生憑證。以下使用案例展示 Chaosctl 透過不同方案執行指令。

**案例 1**：執行 Chaosctl 的節點可存取 Kubernetes 叢集，且能使用 SSH 工具連接實體機器。

執行以下指令以完成操作：

- 指令：使用 `chaosctl pm init` 指令：

  ```bash
  ./chaosctl pm init pm-name --ip=123.123.123.123 -l arch=amd64,anotherkey=value
  ```

- 操作：該指令執行以下操作：
  - 簡化產生 Chaosd 所需憑證的流程，並將憑證儲存至對應的實體機器。
  - 在 Kubernetes 叢集中建立對應的 `PhysicalMachine` 資源。

有關此功能的進一步資訊與範例，請參閱 `chaosctl pm init -h`。

**案例 2**：Chaosctl 所在的節點可存取 Kubernetes 叢集，但無法透過 SSH 工具連線至實體機器。

請執行以下指令完成操作：

1. 執行指令前，需先透過指令手動從 Kubernetes 叢集取得 CA 憑證。例如：

   ```bash
   kubectl get secret chaos-mesh-chaosd-client-certs -n chaos-mesh -o "jsonpath={.data['ca\.crt']}" | base64 -d > ca.crt

   kubectl get secret chaos-mesh-chaosd-client-certs -n chaos-mesh -o "jsonpath={.data['ca\.key']}" | base64 -d> ca.key
   ```

2. 將 `ca.crt` 和 `ca.key` 檔案複製至**對應的實體機器**。例如：將檔案複製至 `/etc/chaosd/pki` 目錄。

3. Use the `chaosctl pm generate` command to generate TLS certificates (save to `/etc/chaosd/pki by default) on **the physical machine**. For example:

   ```bash
   ./chaosctl pm generate --cacert=/etc/chaosd/pki/ca.crt --cakey=/etc/chaosd/pki/ca.key
   ```

   For further information and examples of this feature, refer to `chaosctl pm generate -h`.

4. 在可存取 Kubernetes 叢集的機器上，使用 `chaosctl pm create` 指令於 Kubernetes 叢集建立 `PhysicalMachine` 資源。例如：

   ```bash
   ./chaosctl pm create pm-name --ip=123.123.123.123 -l arch=amd64
   ```

   此功能詳細說明與範例請參閱 `chaosctl pm create -h`。

## 問題與回饋

Chaosctl 原始碼目前託管於 Chaos Mesh 專案中，詳見 [chaos-mesh/pkg/chaosctl](https://github.com/chaos-mesh/chaos-mesh/tree/master/pkg/chaosctl)。

若操作過程遇到問題，或有興趣協助改進此工具，歡迎透過 [CNCF Slack](https://cloud-native.slack.com/archives/C0193VAV272) 聯繫 Chaos Mesh 團隊，或建立 [GitHub issue](https://github.com/chaos-mesh/chaos-mesh/issues)。

描述問題時，請附上相關日誌與 Chaos 資訊。為提供開發者參考依據，建議在問題中附加 `chaosctl logs` 的執行結果。此外，若問題與 iochaos、networkchaos、stresschaos 相關，`chaosctl debug` 的相關資訊亦有助於診斷問題。
---
title: Chaosctl
---

Chaosctl 是一款用於協助除錯 Chaos Mesh 的工具。透過 Chaosctl，您可以簡化開發和除錯新型混沌類型的流程，並在提交問題時為其他開發者提供參考依據。

## 取得 Chaosctl

Linux 使用者可直接下載 Chaosctl 的可執行檔。

```bash
curl -sSL https://mirrors.chaos-mesh.org/latest/chaosctl -O
```

Windows 或 macOS 使用者可從原始碼進行編譯，建議使用 Go v1.15 或更高版本進行編譯。請執行下列步驟：

1. 將 Chaos Mesh 儲存庫複製到您的本機機器。

   ```bash
   git clone https://github.com/chaos-mesh/chaos-mesh.git
   ```

2. 切換至 Chaos Mesh 目錄。

3. 執行下列命令：

   ```bash
   make chaosctl
   ```

   編譯後的可執行檔位於 `bin/chaosctl`。

## 功能

目前 Chaosctl 支援列印混沌實驗的日誌與除錯資訊。

### 列印日誌

要列印所有 Chaos Mesh 元件的日誌，請使用 `chaosctl logs` 命令。查看此功能的說明資訊與範例，請使用 `chaosctl logs -h` 命令。範例命令如下：

```bash
chaosctl logs -t 100 # Print the last 100 lines of logs from all components
```

### 列印除錯資訊

要列印除錯資訊，請使用 `chaosctl debug` 命令。查看此功能的說明資訊與範例，請使用 `chaosctl debug -h` 命令。進行除錯時，需確保 Chaosctl 已連接至對應的 `chaos-daemon`。若您在部署 Chaos Mesh 時停用 TLS（預設啟用），請新增 `-i` 選項告知 Chaosctl 未使用 TLS。範例命令如下：

```bash
./chaosctl debug -i networkchaos web-show-network-delay
```

目前 Chaosctl 僅支援對 IOChaos、NetworkChaos 及 StressChaos 進行除錯。

### 為 Chaosd 產生 TLS 憑證

當 Chaosd 與 Chaos Mesh 之間發起請求時，為確保 Chaosd 與 Chaos-controller-manager 服務間的通信安全，Chaos Mesh 建議啟用 mTLS（雙向傳輸層安全）模式。

啟用 mTLS 模式需在 Chaosd 與 Chaos Mesh 中配置 TLS 憑證參數。因此請確保 Chaosd 與 Chaos Mesh 已產生 TLS 憑證，再以 TLS 憑證作為參數啟動服務。

- Chaosd：您可在配置 TLS 憑證參數**之前或之後**啟動 Chaosd。為確保叢集安全，建議先配置 TLS 憑證參數再啟動 Chaosd。詳情請參見[部署 Chaosd 伺服器](simulate-physical-machine-chaos.md#deploy-chaosd-server)。

- Chaos Mesh：若使用 Helm 部署 Chaos Mesh，則預設已配置 TLS 憑證參數。

若您的 Chaosd 未產生 TLS 憑證，可透過 Chaosctl 的命令列輕鬆產生憑證。下列使用案例中，Chaosctl 透過不同方案執行命令。

**案例 1**：執行 Chaosctl 的節點可存取 Kubernetes 叢集，並能使用 SSH 工具連接實體機器。

執行下列命令完成操作：

- 命令：使用 `chaosctl pm init` 命令：

  ```bash
  ./chaosctl pm init pm-name --ip=123.123.123.123 -l arch=amd64,anotherkey=value
  ```

- 操作：該命令執行下列操作：
  - 簡化產生 Chaosd 所需憑證流程，並將憑證儲存至對應實體機器
  - 在 Kubernetes 叢集中建立對應的 `PhysicalMachine` 資源

此功能的詳細資訊與範例，請參閱 `chaosctl pm init -h`。

**案例 2**：Chaosctl 運行的節點可以存取 Kubernetes 叢集，但無法使用 SSH 工具連接到實體機器。

執行以下命令以完成操作：

1. 在執行命令之前，您需要透過命令手動從 Kubernetes 叢集取得 CA 憑證。例如：

   ```bash
   kubectl get secret chaos-mesh-chaosd-client-certs -n chaos-mesh -o "jsonpath={.data['ca\.crt']}" | base64 -d > ca.crt

   kubectl get secret chaos-mesh-chaosd-client-certs -n chaos-mesh -o "jsonpath={.data['ca\.key']}" | base64 -d> ca.key
   ```

2. 將 `ca.crt` 和 `ca.key` 檔案複製到**對應的實體機器**。例如，將檔案複製到 `/etc/chaosd/pki` 目錄。

3. Use the `chaosctl pm generate` command to generate TLS certificates (save to `/etc/chaosd/pki by default) on **the physical machine**. For example:

   ```bash
   ./chaosctl pm generate --cacert=/etc/chaosd/pki/ca.crt --cakey=/etc/chaosd/pki/ca.key
   ```

   For further information and examples of this feature, refer to `chaosctl pm generate -h`.

4. 在可存取 Kubernetes 叢集的機器上，使用 `chaosctl pm create` 命令在 Kubernetes 叢集中建立 `PhysicalMachine` 資源。例如：

   ```bash
   ./chaosctl pm create pm-name --ip=123.123.123.123 -l arch=amd64
   ```

   有關此功能的詳細資訊和範例，請參閱 `chaosctl pm create -h`。

## 問題與回饋

Chaosctl 的程式碼目前託管在 Chaos Mesh 專案中。詳情請參閱 [chaos-mesh/pkg/chaosctl](https://github.com/chaos-mesh/chaos-mesh/tree/master/pkg/chaosctl)。

如果您在操作過程中遇到問題，或者有興趣協助我們改進此工具，歡迎透過 [CNCF Slack](https://cloud-native.slack.com/archives/C0193VAV272) 聯絡 Chaos Mesh 團隊，或建立 [GitHub issue](https://github.com/chaos-mesh/chaos-mesh/issues)。

在描述問題時，附上相關的日誌和 Chaos 資訊會很有幫助。為提供開發者參考材料，建議在問題中附上 `chaosctl logs` 的結果。此外，如果您的問題與 iochaos、networkchaos、stresschaos 相關，`chaosctl debug` 的相關資訊也有助於診斷問題。
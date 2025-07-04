---
title: Chaosctl
---

Chaosctl 是一款輔助調試 Chaos Mesh 的工具。透過 Chaosctl，您可以簡化開發和調試新混沌類型的流程，並在提交問題時為其他開發者提供參考依據。

## 取得 Chaosctl

Linux 使用者可直接下載 Chaosctl 的可執行檔。

```bash
curl -sSL https://mirrors.chaos-mesh.org/latest/chaosctl -O
```

Windows 或 macOS 使用者需從原始碼編譯，建議使用 Go v1.15 或更高版本進行編譯。請執行以下步驟：

1. 將 Chaos Mesh 儲存庫克隆至本機。

   ```bash
   git clone https://github.com/chaos-mesh/chaos-mesh.git
   ```

2. 切換至 Chaos Mesh 目錄。

3. 執行以下指令：

   ```bash
   make chaosctl
   ```

   編譯後的可執行檔位於 `bin/chaosctl`。

## 功能特性

目前 Chaosctl 支援輸出混沌實驗的日誌與調試資訊。

### 輸出日誌

若要輸出所有 Chaos Mesh 元件的日誌，請使用 `chaosctl logs` 指令。查閱此功能的說明與範例請執行 `chaosctl logs -h`。範例指令如下：

```bash
chaosctl logs -t 100 # Print the last 100 lines of logs from all components
```

### 輸出調試資訊

輸出調試資訊請使用 `chaosctl debug` 指令。查閱此功能的說明與範例請執行 `chaosctl debug -h`。調試時需確保 Chaosctl 已連接對應的 `chaos-daemon`。若部署 Chaos Mesh 時禁用 TLS（預設啟用），需添加 `-i` 參數告知 Chaosctl 未使用 TLS。範例指令如下：

```bash
./chaosctl debug -i networkchaos web-show-network-delay
```

目前 Chaosctl 僅支援調試 IOChaos、NetworkChaos 與 StressChaos。

### 為 Chaosd 生成 TLS 憑證

當 Chaosd 與 Chaos Mesh 之間發起請求時，為確保 Chaosd 與 Chaos-controller-manager 服務間的通信安全，Chaos Mesh 建議啟用 mTLS（雙向傳輸層安全）模式。

啟用 mTLS 模式需在 Chaosd 與 Chaos Mesh 中配置 TLS 憑證參數。因此請確保 Chaosd 與 Chaos Mesh 已生成 TLS 憑證，再以 TLS 憑證作為參數啟動服務。

- Chaosd：您可在配置 TLS 憑證參數**之前或之後**啟動 Chaosd。為確保叢集安全，建議先配置 TLS 憑證參數再啟動 Chaosd。詳情參見[部署 Chaosd 伺服器](simulate-physical-machine-chaos.md#deploy-chaosd-server)。

- Chaos Mesh：若使用 Helm 部署 Chaos Mesh，TLS 憑證參數預設已配置。

若您的 Chaosd 未生成 TLS 憑證，可使用 Chaosctl 透過指令快速生成。以下使用案例展示 Chaosctl 透過不同方案執行指令。

**案例 1**：執行 Chaosctl 的節點可訪問 Kubernetes 叢集，且能透過 SSH 工具連接實體機器。

請執行以下指令完成操作：

- 指令：使用 `chaosctl pm init` 指令：

  ```bash
  ./chaosctl pm init pm-name --ip=123.123.123.123 -l arch=amd64,anotherkey=value
  ```

- 操作：該指令將執行以下操作。
  - 簡化生成 Chaosd 所需憑證，並將憑證儲存至對應實體機器。
  - 在 Kubernetes 叢集中創建對應的 `PhysicalMachine` 資源。

查閱此功能的進階說明與範例請執行 `chaosctl pm init -h`。

**案例二**：Chaosctl 運行的節點可存取 Kubernetes 叢集，但無法透過 SSH 工具連接物理機器。

執行以下指令完成操作：

1. 執行指令前，需手動從 Kubernetes 叢集取得 CA 憑證。例如：

   ```bash
   kubectl get secret chaos-mesh-chaosd-client-certs -n chaos-mesh -o "jsonpath={.data['ca\.crt']}" | base64 -d > ca.crt

   kubectl get secret chaos-mesh-chaosd-client-certs -n chaos-mesh -o "jsonpath={.data['ca\.key']}" | base64 -d> ca.key
   ```

2. 將 `ca.crt` 和 `ca.key` 檔案複製到**對應的物理機器**。例如將檔案複製至 `/etc/chaosd/pki` 目錄。

3. Use the `chaosctl pm generate` command to generate TLS certificates (save to `/etc/chaosd/pki by default) on **the physical machine**. For example:

   ```bash
   ./chaosctl pm generate --cacert=/etc/chaosd/pki/ca.crt --cakey=/etc/chaosd/pki/ca.key
   ```

   For further information and examples of this feature, refer to `chaosctl pm generate -h`.

4. 在可存取 Kubernetes 叢集的機器上，使用 `chaosctl pm create` 指令在 Kubernetes 叢集中建立 `PhysicalMachine` 資源。例如：

   ```bash
   ./chaosctl pm create pm-name --ip=123.123.123.123 -l arch=amd64
   ```

   此功能的詳細說明與範例請參閱 `chaosctl pm create -h`。

## 問題與回饋

Chaosctl 原始碼目前託管於 Chaos Mesh 專案中，詳情參閱 [chaos-mesh/pkg/chaosctl](https://github.com/chaos-mesh/chaos-mesh/tree/master/pkg/chaosctl)。

若操作過程中遇到問題，或有興趣協助改進此工具，歡迎透過 [CNCF Slack](https://cloud-native.slack.com/archives/C0193VAV272) 聯繫 Chaos Mesh 團隊，或建立 [GitHub issue](https://github.com/chaos-mesh/chaos-mesh/issues)。

描述問題時，建議附上相關日誌與混沌工程資訊。為提供開發者參考依據，鼓勵在提問時附上 `chaosctl logs` 的執行結果。此外，若問題與 iochaos、networkchaos 或 stresschaos 相關，`chaosctl debug` 的相關資訊亦有助於診斷問題。
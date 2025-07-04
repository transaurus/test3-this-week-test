---
title: Configure the Development Environment
---

本文件描述如何為 Chaos Mesh 設定本地開發環境。

Chaos Mesh 的大多數元件**僅針對 Linux 設計**，因此建議您將開發環境設定在 Linux 上執行，例如使用虛擬機器、WSL 2 並搭配 VSCode Remote 作為編輯器。

本文件假設您使用 Linux 環境，且不受特定 Linux 發行版限制。若堅持使用 Windows/MacOS，可能需要額外技巧使其運作（例如某些 make 目標可能因環境而失敗）。

## 設定需求

設定前建議安裝以下開發工具，多數工具可能已存在於您的環境中：

- [make](https://www.gnu.org/software/make/)

- [docker](https://docs.docker.com/install/)

- [golang](https://go.dev/doc/install)，`v1.18` 或更新版本

- [gcc](https://gcc.gnu.org/)

- [helm](https://helm.sh/)，`v3.9.0` 或更新版本

- [minikube](https://minikube.sigs.k8s.io/docs/start/)

可選工具：

- [nodejs](https://nodejs.org/en/) 與 [pnpm](https://pnpm.io/)，用於開發 Chaos Dashboard

## 編譯 Chaos Mesh

安裝完成後，請遵循以下步驟編譯 Chaos Mesh：

1. 將 Chaos Mesh 儲存庫複製到本地伺服器：

   ```bash
   git clone https://github.com/chaos-mesh/chaos-mesh.git
   cd chaos-mesh
   ```

2. 確保 [Docker](https://docs.docker.com/install/) 已安裝並執行中：

   :::info

   Chaos Mesh 依賴 Docker 建置容器映像，以確保與生產環境一致性。

   :::

3. 編譯 Chaos Mesh：

   ```bash
   UI=1 make image
   ```

   :::tip

   `UI=1` 表示將編譯 Chaos Dashboard 的使用者介面，若不需要可省略此環境變數。

   :::

   :::tip

   若要指定映像標籤，可使用 `UI=1 make IMAGE_TAG=dev image`。

   :::

   編譯完成後應獲得以下容器映像：

   - `ghcr.io/chaos-mesh/chaos-dashboard:latest`
   - `ghcr.io/chaos-mesh/chaos-mesh:latest`
   - `ghcr.io/chaos-mesh/chaos-daemon:latest`

## 在本地 minikube Kubernetes 叢集中執行 Chaos Mesh

完成編譯後，即可在本地 Kubernetes 叢集中執行 Chaos Mesh。

1. 使用 minikube 啟動本地 Kubernetes 叢集：

   ```bash
   minikube start
   ```

2. 將容器映像載入 minikube：

   ```bash
   minikube image load ghcr.io/chaos-mesh/chaos-dashboard:latest
   minikube image load ghcr.io/chaos-mesh/chaos-mesh:latest
   minikube image load ghcr.io/chaos-mesh/chaos-daemon:latest
   ```

3. 透過 Helm 安裝 Chaos Mesh：

   ```bash
   helm upgrade --install chaos-mesh-debug ./helm/chaos-mesh -n=chaos-mesh-debug --create-namespace
   ```

:::tip

`minikube image load` 會耗費大量時間，因此這裡提供一個技巧以避免在開發過程中反覆載入映像檔：改用 minikube 節點的 Docker 而非主機的 Docker：

```bash
minikube start --mount --mount-string "$(pwd):$(pwd)"
eval $(minikube -p minikube docker-env)
UI=1 make image
```

:::

## 在本機環境中除錯 Chaos Mesh

我們可以使用 [delve](https://github.com/go-delve/delve) 配合遠端除錯功能來在本機環境中除錯 Chaos Mesh。

1. 使用 `DEBUGGER=1` 編譯 Chaos Mesh：

   ```bash
   UI=1 DEBUGGER=1 make image
   ```

2. 將容器映像載入 minikube：

   ```bash
   minikube image load ghcr.io/chaos-mesh/chaos-mesh:latest
   minikube image load ghcr.io/chaos-mesh/chaos-daemon:latest
   minikube image load ghcr.io/chaos-mesh/chaos-dashboard:latest
   ```

3. 安裝 Chaos Mesh 並啟用遠端除錯功能：

   ```bash
   helm upgrade --install chaos-mesh-debug ./helm/chaos-mesh -n=chaos-mesh-debug --create-namespace --set chaosDlv.enable=true --set controllerManager.leaderElection.enabled=false
   ```

   :::note

   為確保高可用性，Chaos Mesh 預設啟用 `leader-election` 功能，會為 `chaos-controller-manager` 建立 3 個副本。我們將使用 `controllerManager.leaderElection.enabled=false` 確保 Chaos Mesh 僅建立 1 個 `chaos-controller-manager` 實例以簡化除錯流程。

   更多細節請參閱[在不同環境中安裝 Chaos Mesh](production-installation-using-helm.md#step-4-install-chaos-mesh-in-different-environments)。

   :::

4. 設定 Port-Forwarding 並設定 IDE 以連接遠端除錯器：

   我們可使用 `kubectl port-forward` 將 delve 除錯伺服器轉發到本機埠。

   例如若要除錯 `chaos-controller-manger`，可執行以下命令：

   ```bash
   kubectl -n chaos-mesh-debug port-forward chaos-controller-manager-766dc8488d-7n5bq 58000:8000
   ```

   接著即可透過 `127.0.0.1:58000` 存取遠端 delve 除錯伺服器。

   :::info

   我們在 Pod 中固定使用 `8000` 埠提供 delve 除錯伺服器服務，這是約定俗成的做法。您可在 Helm 模板檔案中找到此設定。

   :::

   隨後可設定常用 IDE 連接遠端除錯器，以下為參考範例：

   - Goland 使用者請參閱[附加至運行中的 Go 程序#附加至遠端機器程序](https://www.jetbrains.com/help/go/attach-to-running-go-processes-with-debugger.html#attach-to-a-process-on-a-remote-machine)
   - VSCode 使用者請參閱[vscode-go - 除錯#遠端除錯](https://github.com/golang/vscode-go/blob/master/docs/debugging.md#remote-debugging)

更多詳細資訊請參閱[容器映像 chaos-dlv 的 README.md](https://github.com/chaos-mesh/chaos-mesh/blob/master/images/chaos-dlv/README.md)。

## 後續步驟

完成上述準備後，您可以嘗試[新增混沌實驗類型](add-new-chaos-experiment-type.md)。

## 常見問題

### 在 MacOS 上執行 make 失敗，出現 `error obtaining VCS status: exit status 128`

此問題與 https://github.blog/2022-04-12-git-security-vulnerability-announced/ 有關。建議您先閱讀該文章。

Chaos Mesh will start the container (`dev-env` or `build-env`) with the current user (when you call `make`). You can find the appropriate `--user` flag in [get_env_shell.py#L81C10-L81C10](https://github.com/chaos-mesh/chaos-mesh/blob/813b650c02e0b065ae5c4707725c346929ab1847/build/get_env_shell.py#L81C10-L81C10). So when Git is looking for a top-level `.git` directory, it will stop if its directory traversal changes ownership from the current user.

目前暫時的解決方案是將 `--user` 這行註解掉。
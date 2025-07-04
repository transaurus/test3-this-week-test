---
slug: /run_your_first_chaos_experiment
title: Run Your First Chaos Experiment in 10 Minutes
authors: cwen
image: /img/blog/run-first-chaos-experiment-in-ten-minutes.jpg
tags: [Chaos Mesh, Chaos Engineering, Kubernetes]
---

![在 10 分鐘內運行你的第一個混沌實驗](/img/blog/run-first-chaos-experiment-in-ten-minutes.jpg)

混沌工程是一種透過模擬異常或破壞性條件來測試生產軟體系統健壯性的方法。然而對許多人而言，從學習混沌工程到在自身系統實踐的轉變過程令人卻步。這聽起來像是需要完整團隊預先規劃的大型構想，但其實不然。開始混沌實驗，可能只差一個合適的平台。

<!--truncate-->

[Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh) 是一個**易於使用**、開源、雲原生的混沌工程平台，可在 Kubernetes 環境中編排混亂。這篇 10 分鐘教學將協助你快速入門混沌工程，並使用 Chaos Mesh 運行首個混沌實驗。

有關 Chaos Mesh 的更多資訊，請參閱我們的[前文](https://pingcap.com/blog/chaos-mesh-your-chaos-engineering-solution-for-system-resiliency-on-kubernetes/)或 GitHub 上的 [chaos-mesh 專案](https://github.com/chaos-mesh/chaos-mesh)。

## 實驗預覽

混沌實驗類似科學課堂實驗，在受控環境中模擬混亂狀況完全可行。本文將在名為 [web-show](https://github.com/chaos-mesh/web-show) 的小型 Web 應用程式上模擬網路混沌。為可視化混沌效果，web-show 每 10 秒記錄其 Pod 到 `kube-system` 命名空間下 kube-controller Pod 的延遲。

以下片段展示通過幾個指令安裝 Chaos Mesh、部署 web-show 及建立混沌實驗的過程：

![混沌實驗完整流程](/img/blog/whole-process-of-chaos-experiment.gif)

現在輪到你了！是時候動手實作。

## 開始實作

本簡單實驗採用 Docker 中的 Kubernetes ([Kind](https://kind.sigs.k8s.io/)) 進行開發。你可自由使用 [Minikube](https://minikube.sigs.k8s.io/) 或任何現有 Kubernetes 叢集跟隨操作。

### 準備環境

繼續前，請確保本機已安裝並運行 [Git](https://git-scm.com/) 和 [Docker](https://www.docker.com/)。macOS 建議為 Docker 分配至少 6 個 CPU 核心，詳見 [Mac 進階配置](https://docs.docker.com/docker-for-mac/#advanced)。

1. 取得 Chaos Mesh：

   ```bash
   git clone https://github.com/chaos-mesh/chaos-mesh.git
   cd chaos-mesh/
   ```

2. Install Chaos Mesh with the `install.sh` script:

   ```bash
   ./install.sh --local kind
   ```

   `install.sh` is an automated shell script that checks your environment, installs Kind, launches Kubernetes clusters locally, and deploys Chaos Mesh. To see the detailed description of `install.sh`, you can include the `--help` option.

   > **Note:**
   >
   > If your local computer cannot pull images from `docker.io` or `gcr.io`, use the local gcr.io mirror and execute `./install.sh --local kind --docker-mirror` instead.

3. 設定系統環境變數：

   ```bash
   source ~/.bash_profile
   ```

> **注意：**
>
> - 根據您的網路狀況，這些步驟可能需要幾分鐘時間。
> - 若出現類似以下錯誤訊息：
>
>   ```bash
>   ERROR: failed to create cluster: failed to generate kubeadm config content: failed to get kubernetes version from node: failed to get file: command "docker exec --privileged kind-control-plane cat /kind/version" failed with error: exit status 1
>   ```
>
>   請增加本地電腦分配給 Docker 的資源，並執行以下命令：
>
>   ```bash
>   ./install.sh --local kind --force-local-kube
>   ```

當程序完成時，您將看到 Chaos Mesh 已成功安裝的提示訊息。

### 部署應用程式

下一步是部署待測試的應用程式。此處我們選擇 web-show，因為它能直觀呈現網路混沌實驗的效果。您也可以部署自己的應用程式進行測試。

1. 使用 `deploy.sh` 腳本部署 web-show：

   ```bash
   # 確保位於 Chaos Mesh 目錄中
   cd examples/web-show &&
   ./deploy.sh
   ```

   > **注意：**
   >
   > 若本地電腦無法從 `docker.io` 拉取鏡像，請使用 `local gcr.io` 鏡像並執行 `./deploy.sh --docker-mirror`。

2. 存取 web-show 應用程式。在網頁瀏覽器中前往 `http://localhost:8081`。

### 建立混沌實驗

現在萬事俱備，是時候運行您的混沌實驗了！

Chaos Mesh 使用 [CustomResourceDefinitions](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/) (CRD) 定義混沌實驗。CRD 物件根據不同實驗場景獨立設計，大幅簡化了定義流程。目前 Chaos Mesh 已實作的 CRD 物件包括 PodChaos、NetworkChaos、IOChaos、TimeChaos 和 KernelChaos，後續將支援更多故障注入類型。

本實驗中，我們使用 [NetworkChaos](https://github.com/chaos-mesh/chaos-mesh/blob/master/examples/web-show/network-delay.yaml) 進行混沌實驗。以下為以 YAML 撰寫的 NetworkChaos 設定檔：

```
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-example
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - default
    labelSelectors:
      "app": "web-show"
  delay:
    latency: "10ms"
    correlation: "100"
    jitter: "0ms"
  duration: "30s"
  scheduler:
    cron: "@every 60s"
```

NetworkChaos 操作的詳細說明請參閱 [Chaos Mesh wiki](https://github.com/chaos-mesh/chaos-mesh/wiki/Network-Chaos)。此處我們將設定內容簡述為：

- 目標：`web-show`

- mission: inject a `10ms` network delay every `60s`

- 攻擊持續時間：每次 `30s`

啟動 NetworkChaos 的步驟如下：

1. 運行 `network-delay.yaml`：

   ```bash
   # 確保位於 chaos-mesh/examples/web-show 目錄
   kubectl apply -f network-delay.yaml
   ```

2. 存取 web-show 應用程式。在網頁瀏覽器中前往 `http://localhost:8081`。

   從折線圖可觀察到每隔 60 秒會出現 10 毫秒的網路延遲。

![使用 Chaos Mesh 在 web-show 注入延遲](/img/blog/using-chaos-mesh-to-insert-delays-in-web-show.png)

恭喜！您已成功製造了一點混沌。若感興趣並想嘗試更多 Chaos Mesh 實驗，請參閱 [examples/web-show](https://github.com/chaos-mesh/chaos-mesh/tree/master/examples/web-show)。

### 刪除混沌實驗

測試完成後，請終止混沌實驗。

1. 刪除 `network-delay.yaml`：

   ```bash
   # 確保位於 chaos-mesh/examples/web-show 目錄
   kubectl delete -f network-delay.yaml
   ```

2. 存取 web-show 應用程式。請在您的網頁瀏覽器中前往 `http://localhost:8081`。

從折線圖中，您可以看到網路延遲水準已恢復正常。

![網路延遲水準恢復正常](/img/blog/network-latency-level-is-back-to-normal.png)

### 刪除 Kubernetes 叢集

完成混沌實驗後，請執行以下指令刪除 Kubernetes 叢集：

```bash
kind delete cluster --name=kind
```

> **注意：**
>
> 若遇到 `kind: command not found` 錯誤，請先執行 `source ~/.bash_profile` 指令，再刪除 Kubernetes 叢集。

## 太棒了！接下來呢？

恭喜您成功完成首次混沌工程之旅！感覺如何？混沌工程很簡單對吧？不過 Chaos Mesh 可能沒那麼容易使用。命令列操作不夠便利、手動編寫 YAML 檔案有些繁瑣，或是檢查實驗結果稍顯笨拙？別擔心，Chaos Dashboard 即將登場！在網頁上執行混沌實驗聽起來確實令人興奮！如果您願意協助我們建立雲端平台的測試標準或改進 Chaos Mesh，我們非常期待您的參與！

若發現錯誤或認為內容有所遺漏，歡迎提交 issue、發起拉取請求（PR），或加入 [CNCF slack 工作區](https://slack.cncf.io/)的 #project-chaos-mesh 頻道。

GitHub: [https://github.com/chaos-mesh/chaos-mesh](https://github.com/chaos-mesh/chaos-mesh)
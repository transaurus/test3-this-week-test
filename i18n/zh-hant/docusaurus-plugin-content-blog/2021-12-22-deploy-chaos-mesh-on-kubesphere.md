---
slug: /deploy-chaos-mesh-on-kubesphere
title: 'Deploy Chaos Mesh on KubeSphere'
authors: cwen
image: /img/blog/chaos-mesh-kubesphere-banner.png
tags: [Chaos Mesh, Chaos Engineering, Community]
---

![在 KubeSphere 上部署 Chaos Mesh](/img/blog/chaos-mesh-kubesphere-banner.png)

[Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh) 是一個雲原生的混沌工程平台，可在 Kubernetes 環境中編排混亂實驗。透過 Chaos Mesh，您能夠向 Pod、網路、檔案系統甚至核心注入各類故障，藉此測試 Kubernetes 系統的韌性與健壯性。

<!--truncate-->

![Chaos Mesh 架構圖](/img/blog/chaos-mesh-architecture-2.0.png)

## 什麼是 KubeSphere

[KubeSphere](https://kubesphere.io/) 是以 Kubernetes 為核心的分散式雲原生應用管理作業系統，採用即插即用架構，允許第三方應用無縫整合至其生態系統。

KubeSphere 3.2.0 新增了將社群開發的 Helm chart 動態載入至 [KubeSphere 應用商店](https://kubesphere.io/docs/pluggable-components/app-store/)的功能。得益於此新特性，Chaos Mesh 現已登陸 KubeSphere。本教程將指導您在 KubeSphere 上部署 Chaos Mesh 以執行混沌實驗。

## 啟用 KubeSphere 應用商店

1. 確保您已安裝並啟用 [KubeSphere 應用商店](https://kubesphere.io/docs/pluggable-components/app-store/)。

2. 您需要為本教程建立工作區、專案和使用者帳戶 (project-regular)。該帳戶需為平台普通使用者，並需被邀請為具備操作員角色的專案操作員。詳細資訊請參閱[建立工作區、專案、使用者和角色](https://kubesphere.io/docs/quick-start/create-workspace-and-project/)。

## 使用 Chaos Mesh 進行混沌實驗

### 步驟 1：部署 Chaos Mesh

1. 以 `project-regular` 身份登入 KubeSphere，於**應用商店**中搜尋 **chaos-mesh**，點擊搜尋結果進入應用。

   ![Chaos Mesh 應用](/img/blog/chaos-mesh-app.png)

2. 在**應用資訊**頁面中，點擊右上角的**安裝**按鈕。

   ![安裝 Chaos Mesh](/img/blog/install-chaos-mesh.png)

3. 於**應用設定**頁面，設定應用的**名稱**、**位置**（即您的命名空間）與**應用版本**，接著點擊右上角的**下一步**。

   ![Chaos Mesh 基本資訊](/img/blog/chaos-mesh-basic-info.png)

4. 按需配置 `values.yaml` 檔案，或點擊**安裝**使用預設配置。

   ![Chaos Mesh 配置項](/img/blog/chaos-mesh-config.png)

5. 等待部署完成。部署完成後，Chaos Mesh 在 KubeSphere 中將顯示為**運行中**狀態。

   ![已部署 Chaos Mesh](/img/blog/chaos-mesh-deployed.png)

### 步驟 2：訪問 Chaos Dashboard

1. 在**資源狀態**頁面，複製 `chaos-dashboard` 的 **NodePort** 值。

   ![Chaos Mesh NodePort](/img/blog/chaos-mesh-nodeport.png)

2. 在瀏覽器中輸入 `${NodeIP}:${NODEPORT}` 訪問 Chaos Dashboard。請參閱[管理使用者權限](https://chaos-mesh.org/docs/manage-user-permissions/)生成 Token 並登入 Chaos Dashboard。

   ![登入 Chaos Dashboard](/img/blog/login-to-dashboard.png)

### 步驟 3：建立混沌實驗

Before creating a chaos experiment, you should identify and deploy your experiment target, for example, to test how an application works under network latency. Here, we use a demo application `web-show` as the target application to be tested, and the test goal is to observe the system network latency. You can deploy a demo application `web-show` with the following command: `web-show`.

```bash
curl -sSL https://mirrors.chaos-mesh.org/latest/web-show/deploy.sh | bash
```

> 注意：Pod 的網路延遲可直接從 web-show 應用程式面板觀察至 kube-system pod。

1. 在網頁瀏覽器中訪問 `${NodeIP}:8081` 以存取 **Web Show** 應用程式。

   ![Chaos Mesh web show app](/img/blog/web-show-app.png)

2. 登入 Chaos Dashboard 建立混沌實驗。為觀察網路延遲對應用程式的影響，我們將**目標**設為「Network Attack」以模擬網路延遲情境。

   ![Chaos Dashboard](/img/blog/chaos-dashboard-networkchaos.png)

   實驗**範圍**設定為 `app: web-show`。

   ![Chaos Experiment scope](/img/blog/chaos-experiment-scope.png)

3. 透過提交啟動混沌實驗。

   ![Submit Chaos Experiment](/img/blog/start-chaos-experiment.png)

現在您可訪問 **Web Show** 觀察實驗結果：

![Chaos Experiment result](/img/blog/experiment-result.png)

## 總結

KubeSphere 讓雲原生應用程式的部署與維護更加簡易。得益於應用商店，使用者僅需點擊幾下即可在 KubeSphere 上部署 Chaos Mesh，讓您能快速展開混沌實驗。

欲深入了解 Chaos Mesh，請參閱 [Chaos Mesh 文件](https://chaos-mesh.org/docs/) 或加入社群 Slack ([CNCF](https://slack.cncf.io/)/#project-chaos-mesh)。
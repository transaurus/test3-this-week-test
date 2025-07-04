---
title: Send HTTP Requests on Workflow
---

Chaos Mesh Workflow 提供 `Task` 節點以支援任何工作負載，類似於 Kubernetes 的 `Job`。為提升使用者體驗，Chaos Dashboard 提供基於 `Task` 的模板，可在 WebUI 中建立 HTTP 請求。

Chaos Mesh Workflow 提供 `Task` 節點以支援任何工作負載，其功能類似 Kubernetes 中的 `Job`。為提供更佳使用者體驗，Chaos Dashboard 提供基於 `Task` 的模板，透過此模板可輕鬆在 WebUI 中建立 HTTP 請求。本文件說明如何透過 Chaos Dashboard 建立 HTTP 請求。

:::note

Chaos Mesh **並未**提供 `HTTPRequest` 類型的工作流程節點。此功能基於 `Task` 節點實現，透過使用該功能可更便捷地傳送 HTTP 請求。

:::

:::note

此功能目前處於實驗階段，不建議在生產環境中使用。該功能的配置與行為未來可能發生變動。

:::

## 透過 Chaos Dashboard 建立 HTTP 請求

您可依照下列步驟在 Chaos Dashboard 中建立 HTTP 請求，此處以透過 Slack Webhook 傳送訊息為例。

### 步驟 1. 建立類型為 `HTTPRequest` 的工作流程節點

選擇「HTTP Request」作為任務類型：

![create-http-request-workflow-node](img/create-http-request-workflow-node.png)

### 步驟 2. 設定 HTTP 請求

配置以下項目：

- 節點名稱：`send-slack-message`

- 請求 URL：`https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX`

- 請求方法：`POST`

- 請求主體：`{"text": "Hello, world."}`，並勾選「For JSON content」。

![configure-http-request-workflow-node](img/configure-http-request-workflow-node.png)

### 步驟 3. 提交工作流程節點

點擊「Submit」按鈕，在預覽視窗中查看任務：

![http-request-task-node-preview](img/http-request-task-node-preview.png)

## 欄位說明

| Parameter | Type | Description | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| Name | string | Name of the workflow node |  | Yes | `send-slack-message` |
| URL | string | URL of an HTTP request |  | Yes | `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX` |
| Method | string | Method of an HTTP request |  | Yes | `POST` |
| Body | string | Body of an HTTP request |  | No | `{"text": "Hello, world."}` |
| Follow 301/302 Location | boolean | The value of this parameter corresponds to the `-L` parameter of `curl`. | `false` | No | `false` |
| Json Content | boolean | The parameter appends `Content-Type: application/json` to the header of an HTTP request. | `false` | No | `false` |

所產生任務節點的 `name` 欄位值會在「name」結尾處添加後綴 `http-request`。
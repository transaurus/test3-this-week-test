---
title: Status Check in Workflow
---

In Workflow, the status check could execute specified operations on external systems, such as application systems and monitoring systems, to obtain their statuses, and automatically abort the Workflow when it finds the system is unhealthy. The concept is similar to `Container Probes` in Kubernetes. This article describes how to execute status checks in Workflow using YAML files.

:::note

Chaos Mesh 目前尚未支援在 Chaos Dashboard 上建立 `StatusCheck` 節點，因此您暫時只能透過 YAML 建立 `StatusCheck` 節點。

:::

## 狀態檢查類型

Chaos Mesh 目前僅支援透過 `HTTP` 類型執行狀態檢查。

### 定義 `HTTP` 類型的 `StatusCheck` 節點

`StatusCheck` 節點會向特定 URL 發送 `GET` 或 `POST` HTTP 請求，包含自訂標頭和主體，並根據 `criteria` 欄位中的條件判定請求結果。

```yaml
- name: workflow-status-check
  templateType: StatusCheck
  deadline: 20s
  statusCheck:
    mode: Continuous
    type: HTTP
    intervalSeconds: 1
    timeoutSeconds: 1
    http:
      url: http://123.123.123.123
      method: GET
      criteria:
        statusCode: '200'
```

In the configuration, you can see a `StatusCheck` node with `HTTP` type. The `deadline` field specifies that this node could be executed for a maximum of 20 seconds. The `mode` field specifies that this node will execute status checks continuously. The `intervalSeconds` field specifies a repetition interval of 1 second. The `timeoutSeconds` field specifies the timeout for each execution.

當工作流程執行到此 `StatusCheck` 節點時，將每秒執行指定的狀態檢查。該檢查使用 `GET` 方法向 URL `http://123.123.123.123` 發送 HTTP 請求。若回應在 1 秒內返回且狀態碼為 `200`，則該次執行成功，否則視為失敗。

## 狀態檢查結果

Each execution of the status check will get an `execution result`, either `Success` or `Failure`. Because a single `execution result` may not reflect the real situation of the system, due to fluctuations in certain conditions, the final `status check result` is not determined based on a single `execution result`.

`StatusCheck` 節點包含 `failureThreshold` 與 `successThreshold` 兩個欄位：

- When the number of consecutive failed `execution results` exceeds the `failureThreshold`, the `status check result` is considered to be a `Failure`.

- When the number of consecutive successful `execution results` exceeds the `successThreshold`, the `status check result` is considered to be a `Success`.

```yaml
- name: workflow-status-check
  templateType: StatusCheck
  deadline: 20s
  statusCheck:
    mode: Continuous
    type: HTTP
    successThreshold: 1
    failureThreshold: 3
    http:
      url: http://123.123.123.123
      method: GET
      criteria:
        statusCode: '200'
```

在此配置中，`StatusCheck` 節點將持續執行狀態檢查：

- When the `execution result` is `Success` for 1 or more consecutive times, the `status check result` is considered to be a `Success`.

- When the `execution result` is `Failure` for 3 or more consecutive times, the `status check result` is considered to be a `Failure`.

:::note

In the following sections, `status check fails` refers to that `status check result` is `Failure`, rather than a single `execution result` is `Failure`.

:::

### 當狀態檢查未成功時中止工作流程

:::note

`StatusCheck` 節點僅支援在狀態檢查未成功時自動中止工作流程，無法暫停或恢復工作流程。

:::

When executing chaos experiments, the application system might become `unhealthy`, this function can be used to restore the application system by quickly ending chaos experiments. To enable the workflow to abort automatically when the status check fails, you can set the `abortWithStatusCheck` field to `true` on the `StatusCheck` node.

```yaml
- name: workflow-status-check
  templateType: StatusCheck
  deadline: 20s
  abortWithStatusCheck: true
  statusCheck:
    mode: Continuous
    type: HTTP
    http:
      url: http://123.123.123.123
      method: GET
      criteria:
        statusCode: '200'
```

當滿足以下任一條件時，狀態檢查即視為未成功：

- 狀態檢查失敗。

- 當 `StatusCheck` 節點超時，且 `status check result`（狀態檢查結果）不成功時。例如，`successThreshold` 設為 1，`failureThreshold` 設為 3，而當超時發生時，連續失敗次數為 2 次且成功次數為 0。儘管這不符合「狀態檢查失敗」的條件，在此情況下仍被視為不成功。

## 狀態檢查模式

### 連續狀態檢查

當 `mode` 欄位設定為 `Continuous` 時，表示此 `StatusCheck` 節點將持續執行狀態檢查，直到節點超時或狀態檢查失敗。

```yaml
- name: workflow-status-check
  templateType: StatusCheck
  deadline: 20s
  statusCheck:
    mode: Continuous
    type: HTTP
    intervalSeconds: 1
    successThreshold: 1
    failureThreshold: 3
    http:
      url: http://123.123.123.123
      method: GET
      criteria:
        statusCode: '200'
```

在設定中，`StatusCheck` 節點將每秒執行一次狀態檢查，並在滿足以下任一條件時退出：

- 狀態檢查失敗，即連續 3 次或以上 `execution results`（執行結果）失敗

- 在 20 秒後觸發節點超時

### 一次性狀態檢查

當 `mode` 欄位設定為 `Synchronous` 時，表示此 `StatusCheck` 節點將在 `status check result`（狀態檢查結果）明確或節點超時時立即退出。

```yaml
- name: workflow-status-check
  templateType: StatusCheck
  deadline: 20s
  statusCheck:
    mode: Synchronous
    type: HTTP
    intervalSeconds: 1
    successThreshold: 1
    failureThreshold: 3
    http:
      url: http://123.123.123.123
      method: GET
      criteria:
        statusCode: '200'
```

在設定中，`StatusCheck` 節點將每秒執行一次狀態檢查，並在滿足以下任一條件時退出：

- 狀態檢查成功，即連續 1 次或以上 `execution results`（執行結果）成功

- 狀態檢查失敗，即連續 3 次或以上 `execution results`（執行結果）失敗

- 在 20 秒後觸發節點超時

## StatusCheck 與 HTTP Request Task 的比較

相似之處：

- `StatusCheck` 節點和 `HTTP Request Task` 節點（執行 HTTP 請求的 `Task` 節點）都是 Workflow 的節點類型。

- `StatusCheck` 節點和 `HTTP Request Task` 節點都可以透過 HTTP 請求取得外部系統的資訊。

差異：

- `HTTP Request Task` 節點只能發送一次 HTTP 請求，無法連續發送 HTTP 請求。

- `HTTP Request Task` 節點在請求失敗時無法影響工作流程的狀態，例如中止工作流程。

## 欄位說明

有關 Workflow 和 Template 的更多資訊，請參閱[建立 Chaos Mesh Workflow](create-chaos-mesh-workflow.md#field-description)。

### StatusCheck 欄位說明

| Parameter | Type | Description | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| mode | `string` | The execution mode of the status check. Support value: `Synchronous`/`Continuous`. | None | Yes | `Synchronous` |
| type | `string` | The type of the status check. Support value: `HTTP`. | `HTTP` | Yes | `HTTP` |
| duration | `string` | The duration of the whole status check if the number of failed execution does not exceed the `failureThreshold`. It is available in both `Synchronous` and `Continuous` modes. | None | No | `100s` |
| timeoutSeconds | `int` | The timeout seconds when the status check fails. | `1` | No | `1` |
| intervalSeconds | `int` | Defines how often (in seconds) to perform an execution of status check. | `1` | No | `1` |
| failureThreshold | `int` | The minimum consecutive failure for the status check to be considered failed. | `3` | No | `3` |
| successThreshold | `int` | The minimum consecutive successes for the status check to be considered successful. | `1` | No | `1` |
| recordsHistoryLimit | `int` | The number of records to retain. | `100` | No | `100` |
| http | `HTTPStatusCheck` | Configure the detail of the HTTP request to execute. | None | No |  |

### HTTPStatusCheck 欄位說明

| Parameter | Type | Description | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| url | `string` | The URL of the HTTP request. | None | Yes | `http://123.123.123.123` |
| method | `string` | The method of the HTTP request. Support value: `GET`/`POST`. | `GET` | No | `GET` |
| headers | `map[string][]string` | The headers of the HTTP request. | None | No |  |
| body | `string` | The body of the HTTP request. | None | No | `{"a":"b"}` |
| criteria | `HTTPCriteria` | Defines how to determine the result of the HTTP StatusCheck. | None | Yes |  |

### HTTPCriteria 欄位說明

| Parameter | Type | Description | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| statusCode | `string` | The expected http status code for the request. A statusCode string could be a single code (e.g. `200`), or an inclusive range (e.g. `200-400`, both `200` and `400` are included). | None | Yes | `200` |
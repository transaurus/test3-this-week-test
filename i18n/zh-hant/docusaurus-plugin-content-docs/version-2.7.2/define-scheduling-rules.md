---
title: Define Scheduling Rules
---

## 排程功能概覽

本文件說明如何使用 Chaos Mesh 建立排程任務，該任務可在固定時間（或固定時間間隔）自動建立混沌實驗。

在 Kubernetes 環境中，Chaos Mesh 使用 `Schedule` 物件來描述排程任務。

:::note

The name of a `Schedule` object should not exceed 57 characters because the created Chaos experiment will add 6 additional random characters to the end of the name.The name of the `Schedule` object with `Workflow` should not exceed 51 characters because Workflow will add 6 additional random characters to the end of the name.

:::

## 使用 YAML 檔案透過 `kubectl` 建立排程規則

例如，要在每小時第 5 分鐘執行 12 秒的 100 毫秒延遲實驗，可配置以下 YAML 檔案：

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: Schedule
metadata:
  name: schedule-delay-example
spec:
  schedule: '5 * * * *'
  historyLimit: 2
  concurrencyPolicy: 'Allow'
  type: 'NetworkChaos'
  networkChaos:
    action: delay
    mode: one
    selector:
      namespaces:
        - default
      labelSelectors:
        'app': 'web-show'
    delay:
      latency: '10ms'
      correlation: '100'
      jitter: '0ms'
    duration: '12s'
```

將此 YAML 檔案儲存為 `schedule-networkchaos.yaml`，然後執行 `kubectl apply -f ./schedule-networkchaos.yaml`。

Based on this configuration, Chaos Mesh will create the following `NetworkChaos` object in the fifth minute of each hour (such as `0:05`, `1:05`...):

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: schedule-delay-example-xxxxx
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - default
    labelSelectors:
      'app': 'web-show'
  delay:
    latency: '10ms'
    correlation: '100'
    jitter: '0ms'
  duration: '12s'
```

`Schedule` 中的欄位說明如下，多數與 Kubernetes `CronJob` 的欄位相似。更多資訊可參考 [Kubernetes CronJob 文件](https://kubernetes.io/zh/docs/concepts/workloads/controllers/cron-jobs/)。

:::note

`schedule` 欄位的時區取決於 `chaos-controller-manager` 的時區設定。

:::

### `schedule` 欄位

`schedule` 欄位用於指定實驗執行時間，其本質等同於 cron 任務：

```txt
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of the month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday; 7 is also Sunday on some systems)
# │ │ │ │ │
# │ │ │ │ │
# │ │ │ │ │
# * * * * * <command to execute>
```

> This diagram is taken from https://en.wikipedia.org/wiki/Cron.

Chaos Mesh 內部使用 [robfig/cron/v3](https://pkg.go.dev/github.com/robfig/cron/v3) 將 `schedule` 欄位轉換為 cron 表達式。

:::tip

若需生成排程規則，可使用線上工具如 [crontab.guru](https://crontab.guru)。

:::

#### 預定義排程

除標準語法外，還提供多種預定義排程規則，可替代 cron 表達式：

| Entry                  | Description                                | Equivalent To |
| ---------------------- | ------------------------------------------ | ------------- |
| @yearly (or @annually) | Run once a year, midnight, Jan. 1st        | 0 0 1 1 \*    |
| @monthly               | Run once a month, midnight, first of month | 0 0 1 \* \*   |
| @weekly                | Run once a week, midnight between Sat/Sun  | 0 0 \* \* 0   |
| @daily (or @midnight)  | Run once a day, midnight                   | 0 0 \* \* \*  |
| @hourly                | Run once an hour, beginning of hour        | 0 \* \* \* \* |

> This table is taken from https://pkg.go.dev/github.com/robfig/cron/v3#hdr-Predefined_schedules.

#### 間隔設定

亦可設定固定間隔執行任務，從添加任務或 cron 啟動時開始計算：

```txt
@every <duration>
```

例如，`@every 1h30m10s` 表示首次在 1 小時 30 分 10 秒後觸發，後續按此間隔重複執行。

:::info

The content of `Intervals` is taken from https://pkg.go.dev/github.com/robfig/cron/v3#hdr-Intervals. You can refer to the official documentation for more information.

:::

### `historyLimit` 欄位

實驗結束後，相關紀錄不會立即刪除，以便錯誤排查時檢索觀察結果。`historyLimit` 設定保留的任務數量（包含進行中任務），Chaos Mesh 不會刪除運行中的任務。

當任務數量超過 `historyLimit` 時，Chaos Mesh 會依序刪除最早建立的任務。若這些任務仍在運行中，將被跳過而不刪除。

### `concurrencyPolicy` 欄位

此欄位可用值為 `"Forbid"`、`"Allow"` 及 `""`。

此欄位用於指定是否允許此 `Schedule` 物件建立多重並行實驗。例如使用 `schedule: * * * * *` 設定時，每分鐘會建立一個實驗。若實驗的 `duration` 設定為 70 秒，將同時存在多個實驗。

By default, the `concurrencyPolicy` field is set to `Forbid`, which means multiple experiments are not allowed to be created simultaneously. If you set the value of the `concurrencyPolicy` field to `Allow`, multiple experiments are allowed to be created simultaneously.

以下配置仍以延遲實驗為例：

```yaml
spec:
  schedule: '* * * * *'
  type: 'NetworkChaos'
  networkChaos:
    action: delay
    mode: one
    selector:
      namespaces:
        - default
      labelSelectors:
        'app': 'web-show'
    delay:
      latency: '10ms'
    duration: '70s'
```

根據此配置，若設定 `concurrencyPolicy: "Allow"`，則每分鐘會有 10 秒產生 20 毫秒延遲，其餘 50 秒產生 10 毫秒延遲。若設定 `concurrencyPolicy: "Forbid"`，則始終維持 10 毫秒延遲。

:::note

並非所有實驗類型都支援在同個 Pod 上執行多重實驗。詳細資訊請參閱特定實驗類型文件。

:::

### `startingDeadlineSeconds` 欄位

`startingDeadlineSeconds` 的預設值為 `nil`。

當 `startingDeadlineSeconds` 設為 `nil` 時，Chaos Mesh 會檢查從上次排程至今是否有錯過的實驗（可能發生於關閉 Chaos Mesh、長期暫停 Schedule 或將 `concurrencyPolicy` 設為 `Forbid` 時）。

When `startingDeadlineSeconds` is set and larger than `0`, Chaos Mesh will check if any experiments are missed for the past `startingDeadlineSeconds` seconds since the current time. If the value of `startingDeadlineSeconds` is too small, some experiments might be missed. For example:

```yaml
spec:
  schedule: '* * * * *'
  type: 'NetworkChaos'
  networkChaos:
    action: delay
    mode: one
    selector:
      namespaces:
        - default
      labelSelectors:
        'app': 'web-show'
    startingDeadlineSeconds: 5
    delay:
      latency: '10ms'
    duration: '70s'
```

上例中因 `concurrencyPolicy` 設為 `Forbid`，分鐘起始時禁止建立新任務。在該分鐘第 10 秒，上次建立的 Chaos 實驗已結束運行。但受限於 `startingDeadlineSeconds` 及 `concurrencyPolicy` 設定，錯過事件將不被補回，且不會建立 Chaos 實驗。新 Chaos 實驗僅會在下一分鐘起始時建立。

若未設定 `startingDeadlineSeconds`（或設為 `nil`），則始終維持 10 毫秒延遲。這是因為運行任務結束後，Chaos Mesh 發現先前錯過的任務（因 `concurrencyPolicy` 設為 `Forbid`），並立即建立新任務。

更多範例及類似說明請參閱 [Kubernetes CronJob 文件](https://kubernetes.io/zh/docs/concepts/workloads/controllers/cron-jobs/#cron-job-limitations)。

### 定義實驗

要定義實驗具體內容，需在 `Schedule` 中指定兩個欄位：`type` 和 `*Chaos`。`type` 欄位用於指定實驗類型，`*Chaos` 欄位描述實驗內容。通常 `type` 欄位使用大駝峰式命名（如 `NetworkChaos`、`PodChaos`、`IOChaos`），而 `*Chaos` 鍵名使用小駝峰式命名（如 `networkChaos`、`podChaos`、`ioChaos`）。`*Chaos` 鍵名即為對應實驗類型的 `spec`，詳細說明請參閱特定實驗類型文件。

## 使用 Chaos Dashboard 建立排程規則

1. 開啟 Chaos Dashboard，在頁面上點擊 **新建實驗** 以建立新實驗。

   ![建立計劃](./img/create-new-schedule.png)

2. 選擇並填寫實驗的具體細節。

   ![選擇並填寫內容](./img/define-schedule-inner-resource.png)

3. 填寫包括計劃週期和並行策略在內的資訊。

   ![填寫計劃規則](./img/define-schedule-spec.png)

4. 提交實驗資訊。

### 暫停排程任務

與 `CronJob` 不同，暫停 `Schedule` 不僅會阻止其建立新實驗，還會暫停已建立的實驗。

If you do not want to create a Chaos experiment as a scheduled task for now, you need to add the `experiment.chaos-mesh.org/pause=true` annotation to the `Schedule` object. You can add the annotation using the `kubectl` command:

```bash
kubectl annotate -n $NAMESPACE schedule $NAME experiment.chaos-mesh.org/pause=true
```

在指令中，`$NAMESPACE` 是命名空間，`$NAME` 是 `Schedule` 的名稱。成功的結果返回如下：

```bash
schedule/$NAME annotated
```

如果您想取消暫停任務，可以使用以下指令移除註解：

```bash
kubectl annotate -n $NAMESPACE schedule $NAME experiment.chaos-mesh.org/pause-
```

在指令中，`$NAMESPACE` 是命名空間，`$NAME` 是 `Schedule` 的名稱。成功的結果返回如下：

```bash
schedule/$NAME annotated
```
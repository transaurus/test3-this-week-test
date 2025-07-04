---
title: Inspect Results of Chaos Experiments
---

本文說明如何使用 Chaos Mesh 檢查混沌實驗的運行狀態與結果。

## 混沌實驗步驟介紹

在 Chaos Mesh 中，根據混沌實驗的運行過程，其生命週期分為四個步驟：

- 注入中（Injecting）：混沌實驗正處於故障注入過程中。此步驟通常持續時間較短。若「注入中」狀態持續過久，可能是實驗發生異常，此時可檢查 `Events` 尋找異常原因。

- 運行中（Running）：故障成功注入所有目標 Pod 後，混沌實驗進入運行狀態。

- 已暫停（Paused）：對運行中的混沌實驗執行[暫停流程](run-a-chaos-experiment.md#pause-chaos-experiments)時，Chaos Mesh 會從所有目標 Pod 恢復已注入的故障，此時實驗進入暫停狀態。

- 已完成（Finished）：若實驗配置了 `duration` 參數且運行時間達到設定值，Chaos Mesh 會從所有目標 Pod 恢復故障，此時實驗標記為完成。

## 使用 Chaos Dashboard 檢查結果

通過 Chaos Dashboard 可在以下任一頁面查看混沌實驗的運行步驟：

- 混沌實驗列表頁面：

  ![實驗狀態](img/list_chaos_status.png)

- 混沌實驗詳情頁面：

  ![實驗狀態](img/chaos_detail_status.png)

:::note

- If the **"Injecting"** step lasts for a long time, it may be due to some anomalies in the chaos experiment (e.g. the configured selectors have not selected target pods where to inject chaos actions). In this case, you can check **`Events`** to find the cause of the exceptions and check the configuration of the chaos experiment.
- Chaos Dashboard only displays [main steps of a chaos experiment](#introduction-to-steps-of-a-chaos-experiment). For more detailed information about experiment status and results, run the `kubectl` command.

:::

## 使用 `kubectl` 命令檢查結果

確認混沌實驗結果時，可使用以下 `kubectl describe` 命令檢查實驗物件的 `Status` 與 `Events`。

```shell
kubectl describe podchaos pod-failure-tikv -n tidb-cluster
```

預期輸出如下：

```shell
...
Status:
  Conditions:
    Reason:
    Status:  False
    Type:    Paused
    Reason:
    Status:  True
    Type:    Selected
    Reason:
    Status:  True
    Type:    AllInjected
    Reason:
    Status:  False
    Type:    AllRecovered
  Experiment:
    Container Records:
      Id:            tidb-cluster/basic-tikv-0
      Phase:         Injected
      Selector Key:  .
    Desired Phase:   Run
Events:
  Type    Reason           Age   From          Message
  ----    ------           ----  ----          -------
  Normal  FinalizerInited  39s   finalizer     Finalizer has been inited
  Normal  Paused           39s   desiredphase  Experiment has been paused
  Normal  Updated          39s   finalizer     Successfully update finalizer of resource
  Normal  Updated          39s   records       Successfully update records of resource
  Normal  Updated          39s   desiredphase  Successfully update desiredPhase of resource
  Normal  Started          17s   desiredphase  Experiment has started
  Normal  Updated          17s   desiredphase  Successfully update desiredPhase of resource
  Normal  Applied          17s   records       Successfully apply chaos for tidb-cluster/basic-tikv-0
  Normal  Updated          17s   records       Successfully update records of resource
```

上述輸出包含兩個部分：

- `Status`

  根據混沌實驗運行過程，`Status` 提供四類狀態記錄：
  
  - `Paused`: 表示混沌實驗處於「已暫停」步驟
  - `Selected`: 表示實驗已正確選中待注入故障的目標 Pod
  - `AllInjected`: 表示故障已成功注入所有目標 Pod
  - `AllRecoverd`: 表示已注入的故障已從所有目標 Pod 成功恢復

  可從這四類狀態記錄推斷當前混沌實驗的實際運行狀態，例如：
  
  - 當 `Paused`、`Selected`、`AllRecoverd` 為 `True` 且 `AllInjected` 為 `False` 時，表示當前實驗已暫停
  - 當 `Paused` 為 `True` 時表示實驗已暫停，但若同時 `Selected` 為 `False`，則表示當前實驗無法選中目標 Pod

  :::note

  可組合上述狀態記錄獲取更多資訊，例如當 `Paused` 為 `True` 時表示實驗已暫停，但若同時 `Selected` 為 `False`，則表示當前實驗無法選中待注入故障的目標 Pod。

  :::

- `Events`

  包含混沌實驗整個生命週期中的操作記錄，有助於檢查實驗狀態及排查問題。
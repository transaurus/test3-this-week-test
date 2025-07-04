要檢查 Chaos Mesh 的運行狀態，請執行以下命令：

```sh
kubectl get pods -n chaos-mesh -l app.kubernetes.io/instance=chaos-mesh
```

預期輸出如下：

```txt
NAME                                       READY   STATUS    RESTARTS   AGE
chaos-controller-manager-7b8c86cc9-44dzf   1/1     Running   0          17m
chaos-controller-manager-7b8c86cc9-mxw99   1/1     Running   0          17m
chaos-controller-manager-7b8c86cc9-xmc5v   1/1     Running   0          17m
chaos-daemon-sg2k2                         1/1     Running   0          17m
chaos-dashboard-b9dbc6b68-hln25            1/1     Running   0          17m
chaos-dns-server-546675d89d-qkjqq          1/1     Running   0          17m
```

如果您的實際輸出與預期輸出相似，則表示 Chaos Mesh 已成功安裝。

:::note

如果您的實際輸出中 `STATUS` 狀態不是 `Running`，請執行以下命令檢查 Pod 詳細資訊，並根據錯誤訊息進行問題排查。

```sh
# 以 chaos-controller 為例
kubectl describe po -n chaos-mesh chaos-controller-manager-7b8c86cc9-44dzf
```

:::

:::note

如果關閉了 leader election 功能，`chaos-controller-manager` 應該只有 1 個複本。

```txt
NAME                                        READY   STATUS    RESTARTS   AGE
chaos-controller-manager-676d8567c7-ndr5j   1/1     Running   0          24m
chaos-daemon-6l55b                          1/1     Running   0          24m
chaos-dashboard-b9dbc6b68-hln25             1/1     Running   0          44m
chaos-dns-server-546675d89d-qkjqq           1/1     Running   0          44m
```

:::
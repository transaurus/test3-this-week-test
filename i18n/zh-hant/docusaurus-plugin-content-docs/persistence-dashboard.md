---
title: Persistence Chaos Dashboard
---

import PickHelmVersion from '@site/src/components/PickHelmVersion'

本文件說明如何實現 Chaos Dashboard 的持久化設定。

Chaos Dashboard 支援使用 `SQLite`、`MySQL` 和 `PostgreSQL` 作為持久化的資料庫後端。

## SQLite (預設)

Chaos Dashboard 預設採用 `SQLite` 作為資料庫引擎，建議啟用 [PV (持久卷)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)。要啟用 PV，請將 `dashboard.persistentVolume.enabled` 設定為 `true`。您可在 [`value.yaml`](https://github.com/chaos-mesh/chaos-mesh/blob/master/helm/chaos-mesh/values.yaml#L255-L282) 中找到相關配置，如下所示：

```yaml
dashboard:
  ...
  persistentVolume:
    # If you are using SQLite as your DB for Chaos Dashboard, it is recommended to enable persistence.
    # If enable, the chart will create a PersistenceVolumeClaim to store its state in. If you are
    # using a DB other than SQLite, set this to false to avoid allocating unused storage.
    # If set to false, Chaos Mesh will use an emptyDir instead, which is ephemeral.
    enabled: true

    # If you'd like to bring your own PVC for persisting chaos event, pass the name of the
    # created + ready PVC here. If set, this Chart will not create the default PVC.
    # Requires server.persistentVolume.enabled: true
    existingClaim: ""

    # Chaos Dashboard data Persistent Volume size.
    size: 8Gi

    # Chaos Dashboard data Persistent Volume Storage Class.
    # If defined, storageClassName: <storageClass>
    storageClassName: standard

    # Chaos Dashboard data Persistent Volume mount root path
    mountPath: /data

    # Subdirectory of Chaos Dashboard data Persistent Volume to mount
    # Useful if the volume's root directory is not empty
    subPath: ""
```

:::warning

若 Chaos Dashboard 元件在未啟用 PV 的情況下重啟，所有資料將會遺失且無法復原。

:::

## MySQL

Chaos Dashboard 支援 MySQL 5.6 及以上版本作為資料庫引擎。以下範例展示 MySQL 資料庫配置。有關連線字串設定的詳細說明，請參閱 [Go 的 MySQL 驅動程式](https://github.com/go-sql-driver/mysql#dsn-data-source-name)。

<PickHelmVersion>
{`helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --version latest --set dashboard.env.DATABASE_DRIVER=mysql --set dashboard.env.DATABASE_DATASOURCE=root:password@tcp(1.2.3.4:3306)/chaos-mesh?parseTime=true`}
</PickHelmVersion>

## PostgreSQL

Chaos Dashboard 支援 PostgreSQL 9.6 及以上版本作為資料庫引擎。以下範例展示 PostgreSQL 資料庫配置。有關連線字串設定的詳細說明，請參閱 [libpq 連線設定](https://www.postgresql.org/docs/current/static/libpq-connect.html#LIBPQ-CONNSTRING)。

<PickHelmVersion>
{`helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --version latest --set dashboard.env.DATABASE_DRIVER=postgres --set dashboard.env.DATABASE_DATASOURCE=postgres://root:password@1.2.3.4:5432/postgres?sslmode=disable`}
</PickHelmVersion>

## 設定 Chaos Dashboard 資料的 TTL (存活時間)

Chaos Dashboard 支援設定資料的過期時間。預設情況下，`Event` 相關資料於 `168h` 後過期，`Experiment` 相關資料則預設為 `336h`。如需修改，可設定 `dashboard.env.TTL_EVENT` 和 `dashboard.env.TTL_EXPERIMENT` 參數，例如：

<PickHelmVersion>
{`helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --version latest --set dashboard.env.TTL_EVENT=168h --set dashboard.env.TTL_EXPERIMENT=336h`}
</PickHelmVersion>
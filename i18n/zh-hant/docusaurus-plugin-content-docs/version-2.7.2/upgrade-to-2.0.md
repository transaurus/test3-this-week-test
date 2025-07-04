---
title: Upgrade to Chaos Mesh 2.0
---

:::warning

我們已停止隨每個版本上傳遷移檔案，因 Chaos Mesh 2.0 已發佈相當長時間。

最後一次上傳版本為 `v2.7.2`。

:::

本文檔提供從 Chaos Mesh 1.x 升級至 2.0 的詳細指引。Chaos Mesh 2.0 引入了新功能並修復多項問題。由於 2.0 版本重建了部分程式碼，升級時需執行額外操作。

## Upgrade tools

因 Chaos Mesh 2.0 的 CRD 已變更，舊版實驗的 YAML 檔案無法在 2.0 上執行。若需沿用這些 YAML 檔案，您必須先導出並更新檔案，待升級後重新導入。

為簡化升級流程，Chaos Mesh 2.0 提供以下工具：

- `migrate.sh`：用於自動導出、升級 YAML 檔案，更新 CRD 並導入升級後的檔案。

- `schedule-migration`：用於將舊版 YAML 檔案更新至最新格式。

To get the upgrade tools, it is recommended to clone the Chaos Mesh project to your local repository and then execute the command `make schedule-migration.tar.gz`. Or you can download the project from [https://mirrors.chaos-mesh.org/v2.0.0/schedule-migration.tar.gz](https://mirrors.chaos-mesh.org/v2.0.0/schedule-migration.tar.gz). After the `tar.gz` package is downloaded, execute the following command and you can get the above two upgrade tools:

```bash
tar xvf ./schedule-migration.tar.gz
```

此套件中的 `schedule-migration` 工具僅適用於 Linux x86_64 平台。若使用其他作業系統或架構，需自行編譯程式碼。

## 步驟 1：導出並升級實驗

可使用升級工具 `migrate.sh` 自動導出並升級實驗。執行前請確保具備足夠權限存取叢集。

若 `migrate.sh` 位於當前目錄，請將 `schedule-migration` 工具放置於同目錄。執行以下指令以導出並升級實驗：

```bash
bash migrate.sh -e
```

執行後將於當前目錄產生多個 YAML 檔案，此為導出的實驗檔案。導出的實驗檔案已自動完成升級。

此外，亦可使用 `schedule-migration` 工具升級指定的舊版 YAML 檔案：

```bash
./schedule-migration <path-to-old-yaml> <path-to-new-yaml>
```

在指定路徑中可取得升級後的 YAML 檔案。刪除舊資源後，重新套用新 YAML 檔案即可完成更新。

## 步驟 2：升級 CRD

使用 Helm 升級 Chaos Mesh 前，為提高成功率，請先執行以下指令手動升級 CRD：

```bash
bash migrate.sh -c
```

您將看到 CRD 被刪除後重新添加。

## 步驟 3：升級 Chaos Mesh

因 Chaos Mesh 2.0 相較 1.x 有大幅變更，建議先解除安裝 1.x 再安裝 2.0。具體安裝步驟請參閱[使用 Helm 安裝（推薦生產環境）](production-installation-using-helm.md)。

## 步驟 4：導入實驗

Chaos Mesh 2.0 對實驗定義進行了調整。沿用舊版實驗檔案前，需先升級這些檔案。

在導出實驗檔案的相同目錄下，執行以下指令以導入實驗：

```bash
bash migrate.sh -i
```

## 問題回報

如果在升級過程中遇到任何問題，請將命令輸出提交至 [slack](https://cloud-native.slack.com/archives/C0193VAV272) 或在 Github 上建立 [issue](https://github.com/pingcap/chaos-mesh/issues)。感謝您的回饋，Chaos Mesh 團隊樂意為您解決問題。
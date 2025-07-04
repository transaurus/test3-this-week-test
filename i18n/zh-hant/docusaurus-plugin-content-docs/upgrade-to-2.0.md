---
title: Upgrade to Chaos Mesh 2.0
---

:::warning

我們已停止隨每個版本發布遷移檔案，因 Chaos Mesh 2.0 已發布相當長時間。

最後一次上傳版本為 `v2.7.2`。

:::

本文檔提供從 Chaos Mesh 1.x 升級至 2.0 的詳細指引。Chaos Mesh 2.0 引入了若干新功能並修復多項問題。由於 Chaos Mesh 2.0 中部分程式碼已重建，升級需執行額外操作。

## Upgrade tools

因 Chaos Mesh 2.0 的 CRD 已變更，舊版 Chaos Mesh 的實驗 YAML 檔案無法在 2.0 上運行。若需繼續使用這些 YAML 檔案，需先導出並更新 YAML 檔案，升級後重新導入。

為簡化升級流程，Chaos Mesh 2.0 提供以下升級工具：

- `migrate.sh`：用於自動導出並升級 YAML 檔案、更新 CRD、導入升級後的 YAML 檔案。

- `schedule-migration`：用於將舊版 YAML 檔案更新至最新版本。

建議將 Chaos Mesh 專案克隆至本地儲存庫後執行 `make schedule-migration.tar.gz` 指令獲取升級工具。或從 [https://mirrors.chaos-mesh.org/v2.0.0/schedule-migration.tar.gz](https://mirrors.chaos-mesh.org/v2.0.0/schedule-migration.tar.gz) 下載專案。下載 `tar.gz` 壓縮包後執行以下指令，即可取得上述兩項升級工具：

```bash
tar xvf ./schedule-migration.tar.gz
```

此套件中的 `schedule-migration` 工具僅適用於 Linux x86_64 平台。若使用其他作業系統或架構，需自行編譯程式碼。

## 步驟 1：導出並升級實驗

可使用升級工具 `migrate.sh` 自動導出並升級實驗。執行前請確保具備足夠權限存取叢集。

若 `migrate.sh` 位於當前目錄，請將 `schedule-migration` 工具置於同目錄。執行以下指令以導出並升級實驗：

```bash
bash migrate.sh -e
```

隨後當前目錄將生成多個 YAML 檔案，即實驗的導出檔案。導出的實驗檔案已自動升級。

此外，可使用 `schedule-migration` 工具升級指定的舊版 YAML 檔案：

```bash
./schedule-migration <path-to-old-yaml> <path-to-new-yaml>
```

在指定 YAML 檔案路徑中，可取得升級後的 YAML 檔案。刪除舊資源後重新套用新 YAML 檔案即可完成更新。

## 步驟 2：升級 CRD

使用 Helm 升級 Chaos Mesh 前，為提高升級成功率，請執行以下指令手動升級 CRD：

```bash
bash migrate.sh -c
```

可見 CRD 已被刪除並重新添加。

## 步驟 3：升級 Chaos Mesh

因 Chaos Mesh 2.0 相較 1.x 版有大量變更，建議先解除安裝 1.x 版再安裝 2.0 版。具體安裝步驟請參閱[使用 Helm 安裝（推薦生產環境）](production-installation-using-helm.md)。

## 步驟 4：導入實驗

Chaos Mesh 2.0 對實驗定義進行了調整。繼續使用前需先升級舊版實驗檔案。

在導出實驗檔案的相同目錄下，執行以下指令以導入實驗：

```bash
bash migrate.sh -i
```

## 問題回報

如果在升級過程中遇到任何問題，請將命令輸出提交至 [slack](https://cloud-native.slack.com/archives/C0193VAV272) 或在 Github 上建立 [issue](https://github.com/pingcap/chaos-mesh/issues)。感謝您的回饋，Chaos Mesh 團隊樂意為您解決問題。
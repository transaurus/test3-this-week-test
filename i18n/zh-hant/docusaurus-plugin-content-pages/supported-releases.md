# 支援的版本

本頁列出當前支援版本的狀態、時間表與政策。

:::info

## 摘要說明

~~每個版本支援期為 **六個月**，我們 **每三個月** 發布一次新版本。~~

2024-02-01 更新：

因目前缺乏足夠維護者支援發布週期，我們將不定期發布新版本，週期約為 **六個月**。

:::

## 命名方案

我們的命名方案主要遵循[語義化版本 2.0.0](https://semver.org/)，並在 Git 標籤與 Docker 映像檔前綴 `v`：

```plain
v<major>.<minor>.<patch>
```

其中 `<minor>` 隨每次版本發布遞增，`<patch>` 則計算當前 `<minor>` 版本的修補次數。修補版本通常是相對於 `<minor>` 版本的微小變更。

## Chaos Mesh 支援狀態

:::note

Kubernetes `1.22` 的支援始於版本 `2.0.4`。

:::

:::info

`2.6` 版本理論上可在 Kubernetes `1.26`、`1.27` 和 `1.28` 版本正常運作，但我們未同步針對這些版本的端對端測試。

:::

| Version | Currently Supported   | Release Date | End of Life  | Supported Kubernetes versions                        |
| :------ | :-------------------- | :----------- | :----------- | :--------------------------------------------------- |
| master  | No, development only  | -            | -            | 1.30, 1.31, 1.32                                     |
| 2.7     | `Yes`                 | Sep 20, 2024 | -            | 1.26, 1.27, 1.28                                     |
| 2.6     | `Yes`                 | May 31, 2023 | -            | 1.20, 1.21, 1.22, 1.23, 1.24, 1.25, 1.26, 1.27, 1.28 |
| 2.5     | No                    | Nov 22, 2022 | Sep 20, 2024 | 1.20, 1.21, 1.22, 1.23, 1.24, 1.25                   |
| 2.4     | No                    | Sep 23, 2022 | May 31, 2023 | 1.20, 1.21, 1.22, 1.23, 1.24, 1.25                   |
| 2.3     | No                    | Jul 29, 2022 | Nov 22, 2022 | 1.15, 1.16, 1.17, 1.18, 1.19, 1.20, 1.21, 1.22       |
| 2.2     | No                    | Apr 29, 2022 | Sep 23, 2022 | 1.15, 1.16, 1.17, 1.18, 1.19, 1.20, 1.21, 1.22       |
| 2.1     | No                    | Nov 30, 2021 | Jul 29, 2022 | 1.15, 1.16, 1.17, 1.18, 1.19, 1.20, 1.21, 1.22       |
| 2.0     | No                    | Jul 23, 2021 | Apr 29, 2022 | 1.15, 1.16, 1.17, 1.18, 1.19, 1.20, 1.21, 1.22       |
| 1.2     | No                    | Apr 23, 2021 | Nov 30, 2021 | 1.12, 1.13, 1.14, 1.15                               |
| 1.1     | No                    | Jan 08, 2021 | Jul 23, 2021 | 1.12, 1.13, 1.14, 1.15                               |
| 1.0     | No                    | Sep 25, 2020 | Apr 23, 2021 | 1.12, 1.13, 1.14, 1.15                               |

## 即將發布的版本

您可透過 [GitHub 里程碑](https://github.com/chaos-mesh/chaos-mesh/milestones)追蹤即將發布的版本。

## 支援政策

每個發布分支的支援窗口期為 **六個月**。考慮我們 **每三個月** 發布一次正式版本，此支援窗口對應 **最新的兩個版本**。

為此我們提供兩類支援：

- [Community technical support](#community-technical-support)

- [Security and bug fixes](#security-and-bug-fixes)

### 社群技術支援

您可透過 CNCF Slack（在 [#project-chaos-mesh](https://cloud-native.slack.com/archives/C0193VAV272) 頻道）或 [GitHub 討論區](https://github.com/chaos-mesh/chaos-mesh/discussions)向社群請求支援。

### 安全性與錯誤修復

安全性問題會盡快修復。這些修復會回溯至最後兩個版本，並立即為其建立新的修補版本。

對於功能增強或錯誤修復，我們將視需求發布新的修補版本。

## 我們如何決定支援的 Kubernetes 版本

After testing the compatibility of various versions of Kubernetes clusters through E2E testing, the Chaos Mesh team determines [Supported Status of Chaos Mesh](#support-status-of-chaos-mesh) based on its test results.

以下是各版本端對端測試涵蓋的 Kubernetes 版本：

| Version | Tested kubernetes Versions |
| :------ | :------------------------- |
| master  | 1.30.10, 1.31.6, 1.32.2    |
| 2.7     | 1.26.13, 1.27.10, 1.28.6   |
| 2.6     | 1.20.15, 1.23.4, 1.25.1    |
| 2.5     | 1.20, 1.23, 1.25           |
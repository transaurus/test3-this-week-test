---
title: Chaos Mesh Release Cycle
---

本文件主要面向 Chaos Mesh 開發者與貢獻者，這些人員需要建立針對特定發布里程碑的增強功能、議題或拉取請求。

## 角色

### 貢獻者/開發者

貢獻者應以下列形式參與發布週期：

- 參與新功能/增強功能的討論收集

- 貢獻 Chaos Mesh 的程式碼/文件

- 必要時協助精簡功能

身為貢獻者，您僅需注意兩件事：

- 您可能會被要求完成未完成的工作，或將其從發布分支中精簡移除

- 在「程式碼凍結」期間，您的拉取請求可能無法快速合併至主分支

### 發布管理員

「發布管理員」泛指負責維護發布分支、標記版本以及建構/打包 Chaos Mesh 的貢獻者群體。

發布管理員應執行以下事項：

- 收集「新功能/增強功能」並建立 GitHub 議題

- 建立並維護 `vX.Y.0` GitHub 里程碑

- 安排並主持必要會議

- 持續維護下個發布的追蹤文件

- 與貢獻者溝通，請求協助完成/精簡未完成功能或文件；若與貢獻者失聯則自行精簡

- 切出發布分支

- 草擬次要版本的發布說明

- 發布 alpha、beta 及正式次要版本

- 持續完善發布相關文件

### 如何成為發布管理員？

Chaos Mesh 的提交者與維護者將在新發布週期約第 1 週提名新任發布管理員，提名應發布於 Slack 頻道 #chaos-mesh-maintainers。若反對票未超過半數則獲選，最終我們會在文件網站與 Slack 頻道 [#project-chaos-mesh](https://cloud-native.slack.com/archives/C0193VAV272) 公告發布管理員名單。

## 版本規範與時程

- Chaos Mesh 每 8 週發布一次次要版本 `vX.Y.0`

- Chaos Mesh 至少每 2 週發布一次預覽版本 (`vX.Y.0.alpha/beta-W`, W>=0)

- Chaos Mesh 將視需求發布錯誤修正/補丁版本 (`vX.Y.Z`, Z>0)

## 發布階段

每個發布週期包含三個階段：

- 常規開發期 (第 1-5 週)

- 程式碼凍結期 (第 6-7 週)

- 發布週 (第 8 週)

### 常規開發期

此階段發生事項：

- 遴選新任發布管理員

- 收集將納入下個版本的新功能/增強功能

- 若尚未建立則創建 `vX.Y.0` 里程碑

- 編寫程式碼與文件

- 每 2 週發布 alpha 版本

### 程式碼凍結期

此階段發生事項：

- 禁止合併所有不相關的拉取請求

- 審查將納入下個版本的「新功能/增強功能」
  - 完成或修剪未完成的功能
  - 文件準備就緒，至少在 chaos-mesh/website 上有相關的開放議題

- 建立分支 `release-X.Y`

- 發布 beta 版本

- 準備版本發布說明

- 建立 `vX.Y+1.0` 里程碑

- 合併錯誤修復（若需要）

- 撰寫關於新版本的說明文件

「程式碼凍結」階段將於第 6 週開始，並在建立 `release-X.Y` 分支時結束。

當處於「程式碼凍結」期間，與即將發布的次要版本無關的 PR 將禁止合併至 master 分支。僅與該版本相關的 PR 可合併至 master 分支。

版本經理將與貢獻者溝通，要求其完成或修剪未完成功能。若與貢獻者失聯，版本經理有時會自行修剪這些功能。

當所有未完成功能皆完成或標記為「需修剪」後，版本經理將建立 `release-X.Y` 分支。所有 PR 的合併流程隨即恢復正常。

未完成功能修剪後，版本經理將發布首個 beta 版本。此後僅錯誤修復會被揀選至發布分支。若發布分支有更新，版本經理可發布更多 beta 版本。

版本經理應在 beta 版本發布後開始準備版本發布說明。

### 發布週

「發布週」期間發生事項：

- 合併緊急錯誤修復或漏洞修補（若需要）

- 發布次要版本成品（Helm charts、容器映像檔等）

- 發布次要版本文件

版本經理將於本週發布 `vX.Y.0` 版本。
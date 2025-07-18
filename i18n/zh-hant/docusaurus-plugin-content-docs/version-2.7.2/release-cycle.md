---
title: Chaos Mesh Release Cycle
---

本文件主要針對需要建立針對特定發行里程碑的增強功能、問題或拉取請求的 Chaos Mesh 開發人員和貢獻者。

## 角色

### 貢獻者/開發者

貢獻者應以下列形式參與發行週期：

- 參與新功能/增強功能收集的討論

- 為 Chaos Mesh 貢獻程式碼/文件

- 必要時協助修剪功能

作為貢獻者，您只需注意以下兩點：

- 您可能會被要求完成未完成的工作，或從發行分支中將其修剪掉。

- 在「程式碼凍結」期間，您的拉取請求可能不會快速合併至主分支。

### 發行經理

「發行經理」是一個統稱，涵蓋負責維護發行分支、標記發行版本以及建置/打包 Chaos Mesh 的 Chaos Mesh 貢獻者。

發行經理應：

- 將「新功能/增強功能」收集為 GitHub Issue

- 建立並維護 `vX.Y.0` GitHub Milestone

- 安排並舉行必要的會議

- 持續維護下一個發行版本的追蹤文件

- 與貢獻者溝通，請求協助完成/修剪未完成的功能/文件；若與貢獻者失去聯繫，則自行修剪。

- 切割發行分支

- 起草次要版本的發行說明

- 發行 alpha、beta 和次要版本

- 持續豐富發行相關文件

### 如何成為發行經理？

Chaos Mesh 提交者（committers）和維護者（maintainers）會在新發行週期的第 1 週左右提名新的發行經理，提名應在 Slack 頻道 #chaos-mesh-maintainers 中公佈。如果反對票數不超過半數，他們將被選中。最後，我們會在文件網站和 Slack 頻道 [#project-chaos-mesh](https://cloud-native.slack.com/archives/C0193VAV272) 上宣佈發行經理。

## 版本方案與時間表

- Chaos Mesh 每 8 週發行一個次要版本 `vX.Y.0`。

- Chaos Mesh 至少每 2 週發行一個預覽版本 (`vX.Y.0.alpha/beta-W`, W>=0)。

- Chaos Mesh 會根據需要發行錯誤修正/補丁 (`vX.Y.Z`, Z>0) 版本。

## 發行階段

一個發行週期包含以下 3 個階段：

- 正常開發 (第 1-5 週)

- 程式碼凍結 (第 6-7 週)

- 發行週 (第 8 週)

### 正常開發

在「正常開發」期間發生的事情：

- 挑選新的發行經理

- 收集「新功能/增強功能」將在下一版本中進行

- 若不存在則建立 `vX.Y.0` 里程碑

- 編寫程式碼和文件

- 每 2 週發行 alpha 版本

### 程式碼凍結

在「程式碼凍結」期間發生的事情：

- 阻止所有不相關的拉取請求 (PR) 合併

- 審查將納入下一版本的新功能/增強項目
  - 完成或移除未完成的功能
  - 文件準備就緒，至少在 chaos-mesh/website 上有相關的開放議題

- 建立分支 `release-X.Y`

- 發布 beta 版本

- 準備發行說明

- 建立 `vX.Y+1.0` 里程碑

- 必要時合併錯誤修復

- 撰寫新版本相關文件

「程式碼凍結」階段將於第 6 週開始，並在分支 `release-X.Y` 建立時結束

處於「程式碼凍結」期間時，與即將發布的次要版本無關的 PR 將禁止合併至 master 分支，僅限與該版本相關的 PR 可合併至 master

發行經理將與貢獻者溝通，要求完成或移除未完成功能。若與貢獻者失去聯繫，發行經理有時會自行移除

當所有未完成功能均完成或標記為「需移除」後，發行經理將建立 `release-X.Y` 分支，所有 PR 的合併流程恢復正常

未完成功能移除後，發行經理將發布首個 beta 版本。此後僅錯誤修復可挑選至發行分支，若有更新，發行經理可發布更多 beta 版本

發行經理應在 beta 版本發布後開始準備發行說明

### 發布週

「發布週」期間事項：

- 必要時合併緊急錯誤修復或漏洞修補

- 發布次要版本成品（Helm 圖表、容器映像檔等）

- 發布次要版本文件

發行經理將於本週發布 `vX.Y.0` 版本
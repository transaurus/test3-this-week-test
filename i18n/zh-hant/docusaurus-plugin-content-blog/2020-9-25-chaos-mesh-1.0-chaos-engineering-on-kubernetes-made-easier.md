---
slug: /chaos-mesh-1.0-chaos-engineering-on-kubernetes-made-easier
title: 'Chaos Mesh 1.0: Chaos Engineering on Kubernetes Made Easier'
authors: chaos-mesh
image: /img/blog/chaos-mesh-1.0.png
tags: [Chaos Mesh, Chaos Engineering, Announcement]
---

![Chaos Mesh 1.0 - 讓 Kubernetes 混沌工程更簡單](/img/blog/chaos-mesh-1.0.png)

今天，我們自豪地宣布 Chaos Mesh 1.0 正式發布（GA），這是繼 2020 年 7 月成為 [CNCF 沙盒專案](https://pingcap.com/blog/announcing-chaos-mesh-as-a-cncf-sandbox-project)後的重要里程碑。

<!--truncate-->

Chaos Mesh 1.0 是專案發展的重大里程碑。經過開源社群 10 個月的努力，Chaos Mesh 在功能完備性、擴展性和易用性方面已臻成熟。以下是主要亮點：

## 強大的故障注入支援

[Chaos Mesh](https://chaos-mesh.org) 起源於分散式資料庫 [TiDB](https://pingcap.com/products/tidb) 的測試框架，因此充分考量了分散式系統的潛在故障場景。Chaos Mesh 提供全面且細粒度的故障類型，涵蓋 Pod、網路、I/O 和核心層面。透過 YAML 定義混沌實驗，操作簡便高效。

Chaos Mesh 1.0 支援以下故障類型：

- clock-skew: 模擬時鐘偏移

- container-kill: 模擬容器被終止

- cpu-burn: 模擬 CPU 壓力

- io-attribution-override: 模擬檔案異常

- io-fault: 模擬檔案系統 I/O 錯誤

- io-latency: 模擬檔案系統 I/O 延遲

- kernel-injection: 模擬核心故障

- memory-burn: 模擬記憶體壓力

- network-corrupt: 模擬網路封包損壞

- network-duplication: 模擬網路封包重複

- network-latency: 模擬網路延遲

- network-loss: 模擬網路封包遺失

- network-partition: 模擬網路分割

- pod-failure: 模擬 Kubernetes Pod 持續不可用

- pod-kill: 模擬 Kubernetes Pod 被終止

## 可視化的混沌實驗編排

Chaos Dashboard 組件是 Chaos Mesh 的一站式 Web 介面，用於編排混沌實驗。先前僅供 TiDB 測試使用，Chaos Mesh 1.0 已全面開放。該儀表板大幅簡化混沌實驗複雜度，透過點擊操作即可完成實驗範圍定義、故障類型指定、排程規則設定及實驗結果觀察。

![混沌儀表板](/img/blog/chaos-dashboard.gif)

## 強化可觀測性的 Grafana 插件

為提升混沌實驗可觀測性，Chaos Mesh 1.0 內建 Grafana 插件，可直接在應用監控面板顯示即時實驗資訊。目前以註釋形式呈現實驗數據，實現應用運行狀態與混沌實驗狀態的同步觀察。

![Grafana 上的混沌狀態與應用狀態](/img/blog/chaos-status.png)

## 安全可控的混沌實驗

混沌實驗需嚴格控制故障範圍（爆炸半徑）。Chaos Mesh 1.0 不僅提供豐富選擇器精確定位實驗範圍，還支援設定受保護命名空間以保障關鍵應用。同時可透過命名空間權限限制 Chaos Mesh 作用域，多重機制確保混沌實驗安全可控。

## 立即試用

:::note

2022-10-24: Because of https://www.oreilly.com/online-learning/leveraging-katacoda-technology.html, and refer to [#356](https://github.com/chaos-mesh/website/pull/356), the interactive tutorial is temporarily unavailable.

:::

You can quickly deploy Chaos Mesh in your Kubernetes environment through the `install.sh` script or the Helm tool. For specific installation steps, please refer to the [Chaos Mesh Getting Started](https://chaos-mesh-website-archived.netlify.app/docs/1.2.4/user_guides/installation) document. In addition, thanks to the `Katakoda interactive tutorial`, you can also quickly get your hands on Chaos Mesh without having to deploy it.

若您尚未升級至 1.0 GA，請參閱 [1.0 版本發布說明](https://github.com/chaos-mesh/chaos-mesh/releases/tag/v1.0.0)了解變更內容及升級指南。

## 致謝

感謝所有 Chaos Mesh [貢獻者](https://github.com/chaos-mesh/chaos-mesh/graphs/contributors)！

如果您對 Chaos Mesh 感興趣，歡迎透過提交問題、貢獻程式碼、文件或文章加入我們。我們期待您的參與和回饋！
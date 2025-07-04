---
slug: /lfx-mentorship-enriching-awschaos
title: From a Newbie in Software Engineering to a Graduated LFX-Mentee
authors: debabratapanigrahi
image: /img/blog/mentorship_experience.png
tags: [Chaos Mesh, Chaos Engineering, LFX Mentorship, AWS Chaos]
---

![LFX 導師計畫經驗分享](/img/blog/mentorship_blog.jpeg)

[我](https://mentorship.lfx.linuxfoundation.org/mentee/6a0bf7de-9e18-4acb-9a66-f5fecdbeb42e)是印度[國家技術學院魯爾克拉分校](https://nitrkl.ac.in/)生物技術與醫學工程系的生物醫學工程大三學生。作為純粹因熱愛而開始編程的人，這是一段充滿挑戰的自學歷程。但當我踏入開源貢獻領域時，發現它對初學者非常友好，並遇到許多幫助我精進技術的人。

<!--truncate-->

![圖1](/img/blog/mentroship_blog1.png)

## 申請歷程

2021年春季，我得知LFX導師計畫，瀏覽所有[專案清單](https://github.com/cncf/mentoring/blob/master/lfx-mentorship/2021/01-Spring/README.md)後感到忐忑，因不熟悉多數術語而困惑，以為不適合新手。隨後我研讀計畫[文件](https://docs.linuxfoundation.org/lfx/mentorship)、導師計畫[常見問答](https://docs.linuxfoundation.org/lfx/mentorship/mentorship-faqs)，依指示申請了幾個感興趣且技術棧熟悉的專案，如Docker、AWS、Python等。

接著我申請了[Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh)的兩個專案，並立即提交履歷和求職信。幾天後收到導師來信要求提交額外任務。

![圖2](/img/blog/mentorship_blog2.png)

我完成上述任務後，將檔案上傳至GitHub並與導師分享連結。

## 獲選與初期見習

:::note

2022-10-24：因應 https://www.oreilly.com/online-learning/leveraging-katacoda-technology.html 並參見 [#356](https://github.com/chaos-mesh/website/pull/356)，互動式教學目前暫停提供。

:::

我清晰記得收到導師錄取通知郵件的那天，欣喜萬分——這是我首次參與開源計畫。成為計畫見習生讓我倍感榮幸，甚至收到CNCF的正式錄取通知。

![圖3](/img/blog/mentorship_blog4.png)

與導師商定透過Slack溝通後，他詢問我對Kubernetes和GOlang的掌握程度。由於基礎有限，他推薦學習資源並給予兩週準備期，同時規劃實驗任務助我熟悉這些技術。

As I was getting more comfortable with Kubernetes, I started exploring Chaos Mesh and completed the `interactive tutorial`, which gave me a clearer idea about the usage of Chaos Mesh. I then implemented the [hello-world chaos](https://chaos-mesh-website-archived.netlify.app/docs/1.2.4/development_guides/develop_a_new_chaos), which helped me to know more about controllers and CRDs, considered to be the most important part of Chaos Mesh. Also, I got to know about the boilerplate codes, the [kube-builder client](https://github.com/kubernetes-sigs/kubebuilder), and how to use them for scaffolding, followed by writing our own controllers.

在最初的幾天實驗和更了解專案後，我開始解決一些「good first issues」以熟悉對 Chaos Mesh 的上游貢獻。

![img4](/img/blog/mentorship_blog3.png)

在其中一項貢獻中，我嘗試為 stress-chaos 添加多容器支援，這是之前無法實現的。雖然成功實現了，但它破壞了一些其他功能，因此無法合併到即將發布的版本中。更重要的是，在 2.0.0 版本中，這項重構已經完成，因此這項貢獻對我和我的導師來說都是一次學習經驗。之後，我們變得更加謹慎，下次嘗試實現任何新功能時，我們會先提交一份 [RFC](https://github.com/chaos-mesh/rfcs)，並在開始前與其他貢獻者進行討論。

## 我對 AWS Chaos 的貢獻

最初，我被要求實現一種 AWS Chaos 作為該專案的一部分，但當我開始深入探索時，我發現了 [awsssmchaosrunner](https://github.com/amzn/awsssmchaosrunner)，鑑於其功能，我們希望將其整合到 Chaos Mesh 中。

我們計劃分兩部分進行，一部分是「[runner thing](https://github.com/STRRL/awsssmchaosrunner-cli)」專案，它與 awsssmchaosrunner 整合，該部分應使用 kotlin 編寫，並從中構建一個 docker 映像。

另一部分是 AWS Chaos 的定義及其[控制器](https://github.com/chaos-mesh/chaos-mesh/pull/1919)，該部分將使用 go 語言編寫，AWS Chaos 的控制器將使用該「kotlin cli image」建立一個 pod，並向 AWS 發送命令。

## 其他機會

在導師計畫接近尾聲時，我受邀參加了一次 Chaos Mesh [社群會議](https://www.youtube.com/watch?v=ElG0pHRoXwI&t=2s)，並在會上展示我的專案。

之後，我申請了 [Kubernetes Community Days Bangalore](https://community.cncf.io/events/details/cncf-kcd-bengaluru-presents-kubernetes-community-days-bengaluru/) 的 CFP（徵求提案），該活動定於 2021 年 6 月 25 日至 26 日以虛擬方式舉行，我被選為演講者，現在已準備好在會上進行演講。

## 畢業與後續步驟

太棒了！！經過 12 週，我成功從該計畫畢業，感謝我的導師 [Zhou Zhiqiang](https://mentorship.lfx.linuxfoundation.org/mentor/e78b3177-160c-4566-9f3d-8fc9b2ec3cea) 和他的指導，因為沒有他，這一切都不可能實現。

我在 Chaos Mesh 社群度過了一段美好的時光，社群中出色的成員在整個旅程中給予我支持和幫助。我期待為該專案貢獻更多，並在社群中更加活躍。

## 加入 Chaos Mesh 社群

若要加入並了解更多關於 Chaos Mesh 的資訊，請在 [CNCF slack workspace](https://slack.cncf.io/) 中找到 #project-chaos-mesh 頻道，或造訪他們的 [GitHub](https://github.com/chaos-mesh/chaos-mesh)。
# 第五章 监控和报告 Docker 安全事件

在本章中，我们将看看如何及时了解 Docker 发布的安全发现，以便了解您的环境。此外，我们将看看如何安全地报告您发现的任何安全问题，以确保 Docker 有机会在问题变得公开和普遍之前缓解这些问题。在本章中，我们将涵盖以下主题：

+   Docker 安全监控

+   Docker **常见漏洞和暴露** (**CVE**)

+   邮件列表

+   Docker 安全报告

+   负责任的披露

+   安全报告

+   其他 Docker 资源

+   Docker Notary

+   硬件签名

+   阅读材料

# Docker 安全监控

在本节中，我们将看一些监控与您可能使用的任何 Docker 产品相关的安全问题的方法。在使用各种产品时，您需要能够了解是否存在任何安全问题，以便您可以减轻这些风险，保持您的环境和数据安全。

# Docker CVE

要了解 Docker CVE 是什么，您首先需要知道什么是 CVE。CVE 实际上是由 MITRE 公司维护的系统。这些用作以 CVE 编号为基础的公开提供信息的方式，每个漏洞都有一个专用的 CVE 编号以便易于参考。这允许 MITRE 公司为所有获得 CVE 编号的漏洞建立一个国家数据库。要了解更多关于 CVE 的信息，您可以在维基百科文章中找到：

[`en.wikipedia.org/wiki/Common_Vulnerabilities_and_Exposures`](https://en.wikipedia.org/wiki/Common_Vulnerabilities_and_Exposures)

维基百科文章解释了它们如何分配 CVE 编号以及它们遵循的格式。

现在您知道了 CVE 是什么，您可能已经了解了 Docker CVE 是什么。它们是直接与 Docker 安全事件或问题相关的 CVE。要了解更多关于 Docker CVE 的信息或查看当前 Docker CVE 的列表，请访问[`www.docker.com/docker-cve-database`](https://www.docker.com/docker-cve-database)。

此列表将在为 Docker 产品创建 CVE 时随时更新。正如您所见，列表非常小，因此，这可能是一个不会以日常或甚至每月频率增长的列表。

# 邮件列表

在生态系统中跟踪或讨论任何 Docker 产品的安全相关问题的另一种方法是加入他们的邮件列表。目前，他们有两个邮件列表，你可以加入或者跟随。

第一个是开发者列表，你可以加入或者跟随。这个列表是为那些要么帮助贡献 Docker 产品的代码，要么使用 Docker 代码库开发产品的人准备的。链接如下：

[`groups.google.com/forum/#!forum/docker-dev`](https://groups.google.com/forum/#!forum/docker-dev)

第二个列表是用户列表。这个列表是为那些可能有安全相关问题的各种 Docker 产品的用户准备的。你可以搜索已经提交的讨论，加入现有的对话，或者提出新问题，会有其他在邮件列表上的人回答你的问题。论坛链接如下：

[`groups.google.com/forum/#!forum/docker-user`](https://groups.google.com/forum/#!forum/docker-user)

在提出安全相关问题之前，你需要阅读以下部分，以确保你不会暴露任何可能引诱攻击者的现有安全问题。

# Docker 安全报告

报告 Docker 安全问题和监控 Docker 安全问题一样重要。虽然报告这些问题很重要，但是当你发现安全问题并希望报告时，你应该遵循一定的标准。

## 负责披露

在披露与安全相关的问题时，不仅对于 Docker，对于任何产品，都有一个叫做**负责披露**的术语，每个人都应该遵循。负责披露是一项协议，允许开发人员或产品维护者在向公众披露问题之前有充足的时间提供安全问题的修复。

要了解更多关于负责披露的信息，你可以访问[`en.wikipedia.org/wiki/Responsible_disclosure`](https://en.wikipedia.org/wiki/Responsible_disclosure)。

记得要站在负责代码的团队的角度考虑。如果这是你的代码，你不是也希望有人提醒你存在漏洞，这样你就有充足的时间在披露之前修复问题，避免造成普遍恐慌并使收件箱被大量邮件淹没吗？

## 安全报告

目前，报告安全问题的方法是给 Docker 安全团队发送电子邮件，并提供尽可能多的关于安全问题的信息。虽然这些不是 Docker 可能推荐的确切项目，但这些是大多数其他安全专业人员在报告安全问题时喜欢看到的一般准则，例如以下内容：

+   产品和版本，在哪里发现了安全问题

+   重现问题的方法

+   当时使用的操作系统，以及版本

+   您可以提供的任何其他信息

记住，你提供的信息越多，团队就能更快地从他们的角度做出反应，从一开始就更积极地对问题进行攻击。

要报告任何与 Docker 相关产品的安全问题，请确保将任何信息发送到`<security@docker.com>`

# 额外的 Docker 安全资源

如果你正在寻找其他要查看的项目，我们在第一章中涵盖了一些额外的项目，值得进行快速审查。确保回顾第一章，以获取有关接下来的几个项目或每个部分提供的链接的更多详细信息。

## Docker Notary

让我们快速看一下**Docker Notary**，但是要了解有关 Docker Notary 的更多信息，您可以回顾第二章中的信息，*保护 Docker 组件*或以下 URL：

[`github.com/docker/notary`](https://github.com/docker/notary)

Docker Notary 允许您使用推荐的离线保存的私钥对内容进行签名并发布。使用这些密钥对您的内容进行签名有助于确保其他人知道他们正在使用的内容实际上来自于它所说的您，并且可以信任该内容，假设用户信任您。

Docker Notary 有一些关键目标，我认为以下几点很重要：

+   可生存的密钥妥协

+   新鲜度保证

+   可配置的信任阈值

+   签名委托

+   使用现有分发

+   不受信任的镜像和传输

重要的是要知道，Docker Notary 也有服务器和客户端组件。要使用 Notary，您必须熟悉命令行环境。前面的链接将为您解释清楚，并为您提供设置和使用每个组件的演练。

## 硬件签名

与之前的 Docker Notary 部分类似，让我们快速看一下硬件签名，因为这是一个非常重要的功能，必须充分理解。

Docker 还允许硬件签名。这是什么意思？从前面的部分我们看到，您可以使用高度安全的密钥对内容进行签名，使其他人能够验证信息来自于它所说的那个人，这最终为每个人提供了极大的安心。

硬件签名通过允许您添加另一层代码签名，将其提升到一个全新的水平。通过引入一个硬件设备，Yubikey——一个 USB 硬件设备——您可以使用您的私钥（记得将它们安全地离线存放在某个地方），以及一个需要您在签署代码时轻触的硬件设备。这证明了您是一个人类，因为您在签署代码时必须亲自触摸 YubiKey。

有关 Notary 硬件签名部分的更多信息，值得阅读他们发布此功能时的公告，网址如下：

[`blog.docker.com/2015/11/docker-content-trust-yubikey/`](https://blog.docker.com/2015/11/docker-content-trust-yubikey/)

要观看使用 YubiKeys 和 Docker Notary 的视频演示，请访问以下 YouTube 网址：

[`youtu.be/fLfFFtOHRZQ?t=1h21m23s`](https://youtu.be/fLfFFtOHRZQ?t=1h21m23s)

要了解有关 YubiKeys 的更多信息，请访问其网站：

[`www.yubico.com`](https://www.yubico.com)

## 阅读材料

还有一些额外的阅读材料，可以帮助确保您的重点是监控整个 Docker 生态系统的安全方面。

回顾一下第四章，“Docker 安全基准”，我们介绍了 Docker 基准，这是一个用于扫描整个 Docker 环境的应用程序。这对于帮助指出可能存在的任何安全风险非常有用。

我还找到了一本很棒的免费 Docker 安全电子书。这本书将涵盖潜在的安全问题，以及您可以利用的工具和技术来保护您的容器环境。免费的东西不错，对吧？！您可以在以下网址找到这本书：

[`www.openshift.com/promotions/docker-security.html`](https://www.openshift.com/promotions/docker-security.html)

您可以参考以下《容器安全简介》白皮书获取更多信息：

[`d3oypxn00j2a10.cloudfront.net/assets/img/Docker%20Security/WP_Intro_to_container_security_03.20.2015.pdf`](https://d3oypxn00j2a10.cloudfront.net/assets/img/Docker%20Security/WP_Intro_to_container_security_03.20.2015.pdf)

您还可以参考以下《Docker 容器权威指南》白皮书：

[`www.docker.com/sites/default/files/WP-%20Definitive%20Guide%20To%20Containers.pdf`](https://www.docker.com/sites/default/files/WP-%20Definitive%20Guide%20To%20Containers.pdf)

最后两项——《容器安全简介》白皮书和《Docker 容器权威指南》——都是直接由 Docker 创建的，因此它们包含了直接与理解容器结构相关的信息，并且将大量 Docker 信息分解到一个中心位置，您可以随时下载或打印出来并随时使用。它们还可以帮助您了解容器的各个层，并且它们如何帮助保持您的环境和应用程序相互安全。

## Awesome Docker

虽然这不是一个与安全相关的工具，但它是一个非常有用且经常更新的 Docker 工具。Awesome Docker 是一个精选的 Docker 项目列表。它允许其他人通过拉取请求向精选列表贡献。该列表包括了想要开始使用 Docker 的人的主题；有用的文章；深入的文章；网络文章；以及关于使用多服务器 Docker 环境、云基础设施、技巧和新闻通讯的文章，列表还在不断增加。要查看该项目以及其中包含的所有*精彩*内容，请访问以下网址：

[`github.com/veggiemonk/awesome-docker`](https://github.com/veggiemonk/awesome-docker)

# 摘要

在本章中，我们看了一些监视和报告 Docker 安全问题的方法。我们看了一些邮件列表，您可以加入监视 Docker CVE 列表。我们还回顾了如何使用 Docker Notary 对您的图像进行签名，以及如何利用硬件签名来使用硬件设备，比如 YubiKeys。我们还研究了负责任的披露，即在向公众发布之前，给 Docker 修复任何安全相关问题的机会。

在下一章中，我们将研究如何使用一些 Docker 工具。这些工具可以用来保护 Docker 环境。我们将研究命令行工具和图形界面工具，您可以利用它们的优势。我们将研究如何在您的环境中使用 TLS，使用只读容器，利用内核命名空间和控制组，并减轻风险，同时注意 Docker 守护程序的攻击面。

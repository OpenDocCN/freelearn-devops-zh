# 前言

Kubernetes 已经风靡全球，成为 DevOps 团队开发、测试和运行应用程序的标准基础设施。大多数企业要么已经在运行它，要么计划在未来一年内运行它。在任何主要的招聘网站上查看职位发布，几乎每个知名公司都有 Kubernetes 职位空缺。快速的采用速度导致与 Kubernetes 相关的职位在过去 4 年增长了超过 2000%。

公司普遍面临的一个问题是缺乏企业级 Kubernetes 知识。由于这项技术相对较新，甚至对于生产工作负载来说更是新颖，公司在尝试构建可靠运行集群的团队时遇到了问题。寻找具有基本 Kubernetes 技能的人变得更容易，但寻找具有企业集群所需知识的人仍然是一个挑战。

# 这本书是为谁准备的

我们创建这本书是为了帮助 DevOps 团队将他们的技能扩展到超越 Kubernetes 的基础知识。这本书是根据我们在多个企业环境中使用集群的多年经验创建的。

有许多介绍 Kubernetes 和安装集群、创建部署以及使用 Kubernetes 对象基础知识的书籍可供选择。我们的计划是创建一本超越基础集群的书籍，并且为了保持书籍的合理长度，我们不会重复介绍 Kubernetes 的基础知识。在阅读本书之前，您应该具有一些 Kubernetes 的经验。

虽然本书的主要重点是扩展具有企业特性的集群，但书的第一部分将提供对关键 Docker 主题和 Kubernetes 对象的复习。您应该对 Kubernetes 对象有扎实的理解，以便充分利用更高级的章节。

# 这本书涵盖了什么

*第一章*, *理解 Docker 和容器基础知识*，帮助您了解 Docker 和 Kubernetes 为开发人员解决了哪些问题。您将被介绍到 Docker 的不同方面，包括 Docker 守护程序、数据、安装和使用 Docker CLI。

第二章《使用 Docker 数据》讨论了容器需要存储数据，有些用例只需要临时磁盘，而其他需要持久磁盘。在这一章中，你将了解持久数据以及 Docker 如何与卷、绑定挂载和 tmpfs 一起使用来存储数据。

第三章《理解 Docker 网络》向你介绍了 Docker 中的网络。它将涵盖创建不同类型的网络、添加和移除容器网络以及暴露容器服务。

第四章《使用 KinD 部署 Kubernetes》展示了 KinD 是一个强大的工具，允许你创建从单节点集群到完整多节点集群的 Kubernetes 集群。这一章超越了基本的 KinD 集群，解释了如何使用运行 HAproxy 的负载均衡器来负载均衡工作节点。通过这一章的学习，你将了解 KinD 的工作原理，以及如何创建一个自定义的多节点集群，这将用于接下来章节的练习中。

第五章《Kubernetes 训练营》涵盖了集群包含的大部分对象，无论你需要对 Kubernetes 进行复习，还是你是新手。它通过描述每个对象的功能和在集群中的作用来解释这些对象。这一章旨在作为一个复习，或者对象的“口袋指南” - 它不包含每个对象的详尽细节，因为那将需要另一本书。

第六章《服务、负载均衡和外部 DNS》教会你如何使用服务暴露 Kubernetes 部署。每种服务类型都有例子进行解释，你将学会如何使用 Layer-7 和 Layer-4 负载均衡器来暴露它们。在这一章中，你将超越简单的 Ingress 控制器的基础，安装 MetalLB，为服务提供 Layer-4 访问。你还将安装一个名为 external-dns 的孵化器项目，为 MetalLB 暴露的服务提供动态名称解析。

第七章《将身份验证集成到您的集群中》考虑了一旦构建完成，用户将如何访问您的集群的问题。在本章中，我们将详细介绍 OpenID Connect 的工作原理以及为什么应该使用它来访问您的集群。我们还将介绍应该避免的几种反模式以及为什么应该避免它们。

第八章《RBAC 策略和审计》演示了一旦用户访问集群，您需要能够限制他们的访问。无论您是为用户提供整个集群还是只是一个命名空间，您都需要了解 Kubernetes 如何通过其基于角色的访问控制系统（RBAC）授权访问。在本章中，我们将详细介绍如何设计 RBAC 策略，如何调试它们以及多租户的不同策略。

第九章《保护 Kubernetes 仪表板》着眼于 Kubernetes 仪表板，这通常是用户在集群启动后尝试启动的第一件事。关于安全性（或缺乏安全性）有相当多的神话。您的集群还将由其他 Web 应用程序组成，例如网络仪表板、日志系统和监控仪表板。本章将介绍仪表板的架构、如何正确地保护它，以及如何不部署它的示例，并详细说明为什么不应该这样做。

第十章《创建 Pod 安全策略》涉及运行您的 Pod 实例的节点的安全性。我们将讨论如何安全地设计您的容器，使它们更难被滥用，以及如何构建策略来限制您的容器访问它们不需要的资源。我们还将介绍 PodSecurityPolicy API 的弃用以及如何处理它。

第十一章《使用 Open Policy Agent 扩展安全性》为您提供了部署 OpenPolicyAgent 和 GateKeeper 的指导，以实现无法使用 RBAC 或 PodSecurityPolicies 实施的策略。我们将介绍如何部署 GateKeeper，如何在 Rego 中编写策略，以及如何使用 OPA 的内置测试框架测试您的策略。

*第十二章*, *使用 Falco 和 EFK 进行审计,* 讨论了 Kubernetes 包括 API 访问的事件日志记录，但它没有能力记录可能在 Pod 内执行的事件。为了解决这个限制，我们将安装一个捐赠给 CNCF 的项目，名为 Falco。你还将学习如何使用 FalcoSideKick 和**EFK**堆栈（**ElasticSearch，FluentD 和 Kibana**）呈现 Falco 捕获的数据。你将通过在 Kibana 中查找事件并创建包含重要事件的自定义仪表板来获得实际操作经验。

*第十三章*, *备份工作负载,* 教你如何使用 Velero 为灾难恢复或集群迁移创建集群工作负载的备份。你将亲自动手创建示例工作负载的备份，并将备份恢复到全新的集群，以模拟集群迁移。

*第十四章*, *平台配置,* 让你构建一个用于自动化多租户集群的平台，其中包括 GitLab、Tekton、ArgoCD 和 OpenUnison。我们将探讨如何构建流水线以及如何自动化它们的创建。我们将探讨用于驱动流水线的对象如何相互关联，如何在系统之间建立关系，最后，如何创建用于自动化流水线部署的自助式工作流程。

# 为了充分利用本书

你应该对 Linux、基本命令和工具（如 Git 和 vi 文本编辑器）有基本的了解。

这些章节包含理论和动手练习。我们认为练习有助于加强理论，但并非必须理解每个主题。如果你想在书中做练习，你需要满足下表中的要求：

![](img/B15514_Table_01.jpg)

所有练习都使用 Ubuntu，但大多数练习也适用于其他 Linux 安装。Falco 章节中的步骤特定于 Ubuntu，其他 Linux 安装可能无法正确部署练习。

**如果你使用的是本书的数字版本，我们建议你自己输入代码或通过 GitHub 存储库访问代码（链接在下一节中提供）。这样做将帮助你避免与复制和粘贴代码相关的潜在错误。**

# 下载示例代码文件

您可以从[www.packt.com](http://www.packt.com)的帐户中下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问[www.packtpub.com/support](http://www.packtpub.com/support)并注册，以便文件直接发送到您的邮箱。

您可以按照以下步骤下载代码文件：

1.  登录或注册[www.packt.com](http://www.packt.com)。

1.  选择**支持**选项卡。

1.  单击**代码下载**。

1.  在**搜索**框中输入书名，然后按照屏幕上的说明操作。

下载文件后，请确保使用以下最新版本解压或提取文件夹：

+   Windows 上的 WinRAR/7-Zip

+   Mac 上的 Zipeg/iZip/UnRarX

+   Linux 上的 7-Zip/PeaZip

本书的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/Kubernetes-and-Docker-The-Complete-Guide`](https://github.com/PacktPublishing/Kubernetes-and-Docker-The-Complete-Guide)。如果代码有更新，将在现有的 GitHub 存储库上进行更新。

我们还有来自丰富书籍和视频目录的其他代码包，可在[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)上找到！快去看看吧！

# 实际代码

本书的实际代码视频可以在[`bit.ly/2OQfDum`](http://bit.ly/2OQfDum)上观看。

# 下载彩色图像

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。您可以在此处下载：**https://static.packt-cdn.com/downloads/9781839213403_ColorImages.pdf**。

# 使用的约定

本书中使用了许多文本约定。

**文本中的代码**：指示文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。以下是一个示例：“要识别的最后一个组件是**apiGroups**。这是 URL 模型中的另一个不一致的地方。”

代码块设置如下：

apiVersion：rbac.authorization.k8s.io/v1

类型：ClusterRole

元数据：

名称：cluster-pod-and-pod-logs-reader

规则：

- apiGroups：[""]

资源：["pods", "pods/log"]

动词：["get", "list"]

当我们希望引起您对代码块的特定部分的注意时，相关行或项目将以粗体显示：

- hostPath：

路径：/usr/share/ca-certificates

类型：DirectoryOrCreate

名称：usr-share-ca-certificates

**  - hostPath：**

**      路径：/var/log/k8s**

**      类型：DirectoryOrCreate**

   name: var-log-k8s

- hostPath:

      path: /etc/kubernetes/audit

      type: DirectoryOrCreate

   name: etc-kubernetes-audit

任何命令行输入或输出都以以下方式编写：

PS C:\Users\mlb> kubectl create ns not-going-to-work namespace/not-going-to-work created

粗体：表示一个新术语，一个重要单词，或者你在屏幕上看到的单词。例如，菜单或对话框中的单词会以这种方式出现在文本中。这是一个例子：“点击屏幕底部的**完成登录**按钮。”

提示或重要说明

看起来像这样。

# 前言

Kubernetes 可能是我们所知道的最大的项目。它是庞大的，然而许多人认为经过几周或几个月的阅读和实践后，他们就知道了所有关于它的知识。它比这大得多，而且它的增长速度比我们大多数人能够跟上的要快。你在 Kubernetes 采用中走了多远？

根据我的经验，Kubernetes 采用有四个主要阶段。

在第一阶段，我们创建一个集群，并学习 Kube API 的复杂性以及不同类型的资源（例如 Pods，Ingress，Deployments，StatefulSets 等）。一旦我们熟悉了 Kubernetes 的工作方式，我们就开始部署和管理我们的应用程序。在这个阶段结束时，我们可以大喊“**看，我的生产 Kubernetes 集群中有东西在运行，没有出现问题！**”我在《DevOps 2.3 工具包：Kubernetes》中解释了这个阶段的大部分内容（[`amzn.to/2GvzDjy`](https://amzn.to/2GvzDjy)）。

第二阶段通常是自动化。一旦我们熟悉了 Kubernetes 的工作方式，并且我们正在运行生产负载，我们就可以转向自动化。我们经常采用某种形式的持续交付（CD）或持续部署（CDP）。我们使用我们需要的工具创建 Pods，构建我们的软件和容器映像，运行测试，并部署到生产环境。完成后，我们的大部分流程都是自动化的，我们不再手动部署到 Kubernetes。我们可以说**事情正在运行，我甚至没有碰键盘**。我尽力在《DevOps 2.4 工具包：持续部署到 Kubernetes》中提供了一些关于 Kubernetes 的 CD 和 CDP 的见解（[`amzn.to/2NkIiVi`](https://amzn.to/2NkIiVi)）。

第三阶段在许多情况下与监控、警报、日志记录和扩展有关。我们可以在 Kubernetes 中运行（几乎）任何东西，并且它会尽最大努力使其具有容错性和高可用性，但这并不意味着我们的应用程序和集群是防弹的。我们需要监视集群，并且我们需要警报来通知我们可能存在的问题。当我们发现有问题时，我们需要能够查询整个系统的指标和日志。只有当我们知道根本原因是什么时，我们才能解决问题。在像 Kubernetes 这样高度动态的分布式系统中，这并不像看起来那么容易。

此外，我们需要学习如何扩展（和缩减）一切。应用程序的 Pod 数量应随时间变化，以适应流量和需求的波动。节点也应该进行扩展，以满足我们应用程序的需求。

Kubernetes 已经有了提供指标和日志可见性的工具。它允许我们创建自动扩展规则。然而，我们可能会发现单单 Kubernetes 还不够，我们可能需要用额外的流程和工具来扩展我们的系统。这本书的主题就是这个阶段。当你读完它时，你将能够说**你的集群和应用程序真正是动态和有弹性的，并且需要很少的手动干预。我们将努力使我们的系统自适应。**

我提到了第四阶段。亲爱的读者，那就是其他一切。最后阶段主要是关于跟上 Kubernetes 提供的所有其他好东西。这是关于遵循其路线图并调整我们的流程以获得每个新版本的好处。

最终，你可能会遇到困难，需要帮助。或者你可能想对这本书的内容写一篇评论或评论。请加入*DevOps20*（[`slack.devops20toolkit.com/`](http://slack.devops20toolkit.com/)）Slack 工作区，发表你的想法，提出问题，或参与讨论。如果你更喜欢一对一的沟通，你可以使用 Slack 给我发私信，或发送邮件至`viktor@farcic.com`。我写的所有书对我来说都很重要，我希望你在阅读它们时有一个很好的体验。其中一部分体验就是可以联系我。不要害羞。

请注意，这本书和之前的书一样，是我自行出版的。我相信作家和读者之间没有中间人是最好的方式。这样可以让我更快地写作，更频繁地更新书籍，并与你进行更直接的沟通。你的反馈是这个过程的一部分。无论你是在只有少数章节还是所有章节都写完时购买了这本书，我的想法是它永远不会真正完成。随着时间的推移，它将需要更新，以使其与技术或流程的变化保持一致。在可能的情况下，我会尽量保持更新，并在有意义的时候发布更新。最终，事情可能会发生如此大的变化，以至于更新不再是一个好选择，这将是需要一本全新书的迹象。**只要我继续得到你的支持，我就会继续写作。**

# 概述

我们将探讨操作 Kubernetes 集群所需的一些技能和知识。我们将处理一些通常不会在最初阶段学习的主题，而是在我们厌倦了 Kubernetes 的核心功能（如 Pod、ReplicaSets、Deployments、Ingress、PersistentVolumes 等）之后才会涉及。我们将掌握一些我们通常在学会基础知识并自动化所有流程之后才会深入研究的主题。我们将探讨**监控**、**警报**、**日志记录**、**自动扩展**等旨在使我们的集群**具有弹性**、**自给自足**和**自适应**的主题。

# 受众

我假设你对 Kubernetes 很熟悉，不需要解释 Kube API 的工作原理，也不需要解释主节点和工作节点之间的区别，尤其不需要解释像 Pods、Ingress、Deployments、StatefulSets、ServiceAccounts 等资源和构造。如果你不熟悉，这个内容可能太高级了，我建议你先阅读《The DevOps 2.3 Toolkit: Kubernetes》（[`amzn.to/2GvzDjy`](https://amzn.to/2GvzDjy)）。我希望你已经是一个 Kubernetes 忍者学徒，你对如何使你的集群更具弹性、可扩展和自适应感兴趣。如果是这样，这本书就是为你准备的。继续阅读。

# 要求

这本书假设你已经知道如何操作 Kubernetes 集群，因此我们不会详细介绍如何创建一个，也不会探讨 Pods、Deployments、StatefulSets 等常用的 Kubernetes 资源。如果这个假设是不正确的，你可能需要先阅读《The DevOps 2.3 Toolkit: Kubernetes》。

除了基于知识的假设外，还有一些技术要求。如果您是**Windows 用户**，请从**Git Bash**中运行所有示例。这将允许您像 MacOS 和 Linux 用户一样通过他们的终端运行相同的命令。Git Bash 在[Git](https://git-scm.com/download/win)安装过程中设置。如果您还没有它，请重新运行 Git 设置。

由于我们将使用 Kubernetes 集群，我们将需要**kubectl** ([`kubernetes.io/docs/tasks/tools/install-kubectl/`](https://kubernetes.io/docs/tasks/tools/install-kubectl/))。我们将在集群内运行的大多数应用程序都将使用**Helm** ([`helm.sh/`](https://helm.sh/))进行安装，因此请确保您也安装了客户端。最后，也安装**jq** ([`stedolan.github.io/jq/`](https://stedolan.github.io/jq/))。这是一个帮助我们格式化和过滤 JSON 输出的工具。

最后，我们将需要一个 Kubernetes 集群。所有示例都是使用**Docker for Desktop**，**minikube**，**Google Kubernetes Engine (GKE)**，**Amazon Elastic Container Service for Kubernetes (EKS)**和**Azure Kubernetes Service (AKS)**进行测试的。我将为每种 Kubernetes 版本提供要求（例如节点数、CPU、内存、Ingress 等）。

您可以将这些教训应用于任何经过测试的 Kubernetes 平台，或者您可以选择使用其他平台。这本书中的示例不应该在任何 Kubernetes 版本中无法运行。您可能需要在某些地方进行微调，但我相信这不会成为问题。

如果遇到任何问题，请通过*DevOps20* ([`slack.devops20toolkit.com/`](http://slack.devops20toolkit.com/)) slack 工作区与我联系，或者通过发送电子邮件至`viktor@farcic.com`。我会尽力帮助解决。如果您使用的是我未测试过的 Kubernetes 集群，请帮助我扩展列表。

在选择 Kubernetes 版本之前，您应该知道并非所有功能都在所有地方都可用。在基于 Docker for Desktop 或 minikube 的本地集群中，由于两者都是单节点集群，将无法扩展节点。其他集群可能也无法使用更多特定功能。我将利用这个机会比较不同的平台，并为您提供额外的见解，如果您正在评估要使用哪个 Kubernetes 发行版以及在哪里托管它，您可能会想要使用。或者，您可以选择在本地集群中运行一些章节，并仅在本地无法运行的部分切换到多节点集群。这样，您可以通过在云中拥有一个短暂的集群来节省一些开支。

如果您不确定要选择哪种 Kubernetes 版本，请选择 GKE。它目前是市场上最先进和功能丰富的托管 Kubernetes。另一方面，如果您已经习惯了 EKS 或 AKS，它们也差不多可以。本书中的大多数，如果不是全部的功能都会起作用。最后，您可能更喜欢在本地运行集群，或者您正在使用不同的（可能是本地）Kubernetes 平台。在这种情况下，您将了解到您所缺少的内容，以及您需要在“标准提供”的基础上构建哪些内容来实现相同的结果。

# 下载示例代码文件

您可以从您在[www.packt.com](http://www.packt.com)上的账户中下载本书的示例代码文件。如果您在其他地方购买了这本书，您可以访问[www.packtpub.com/support](https://www.packtpub.com/support)注册，将文件直接发送到您的邮箱。

您可以按照以下步骤下载代码文件：

1.  在[www.packt.com](http://www.packt.com)上登录或注册。

1.  选择“支持”选项卡。

1.  点击“代码下载”。

1.  在搜索框中输入书名，按照屏幕上的指示操作。

文件下载后，请确保使用以下最新版本的解压缩或提取文件夹：

+   Windows 的 WinRAR/7-Zip

+   Mac 的 Zipeg/iZip/UnRarX

+   Linux 的 7-Zip/PeaZip

该书的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/The-DevOps-2.5-Toolkit`](https://github.com/PacktPublishing)。如果代码有更新，将会在现有的 GitHub 存储库中更新。

我们还有来自我们丰富书籍和视频目录的其他代码包，可在[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)上找到。去看看吧！

# 下载彩色图片

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。您可以在这里下载：[`static.packt-cdn.com/downloads/9781838647513_ColorImages.pdf`](https://static.packt-cdn.com/downloads/9781838647513_ColorImages.pdf)。

# 使用的约定

本书中使用了许多文本约定。

`CodeInText`：表示文本中的代码词，数据库表名，文件夹名，文件名，文件扩展名，路径名，虚拟 URL，用户输入和 Twitter 句柄。这是一个例子：“该定义使用`HorizontalPodAutoscaler`目标为`api`部署。”

代码块设置如下：

[PRE0]

当我们希望引起您对代码块的特定部分的注意时，相关的行或项目将以粗体显示：

[PRE1]

任何命令行输入或输出都以以下方式编写：

[PRE2]

**粗体**：表示一个新术语，一个重要的词，或者你在屏幕上看到的词。例如，菜单或对话框中的单词会以这种方式出现在文本中。这是一个例子：“选择 Prometheus，并单击导入按钮。”

警告或重要说明会以这种方式出现。提示和技巧会以这种方式出现。

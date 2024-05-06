# 第十一章：更多资源

我们已经结束了我们的 Docker 和 Kubernetes 之旅。阅读完这本书后，您应该已经知道 Kubernetes 如何补充 Docker。您可以将它们视为软件堆栈的不同层；Docker 位于下方，为单个容器提供服务，而 Kubernetes 在集群中编排和管理它们。Docker 变得越来越受欢迎，很多人在开发或生产部署中使用它。举几个例子，PayPal、通用电气、Groupon、Spotify 和 Uber 都在使用它。它已经足够成熟，可以在生产环境中运行，我希望您也能成功地使用它来部署和运行您的 Java 应用程序。

要进一步扩展您对 Docker 和 Kubernetes 的知识，有大量的信息。关键是找到有价值的信息。在本章中，我将介绍最有用的信息，如果您想进一步扩展您的 Docker 和 Kubernetes 知识。

# Docker

我们列表上的第一个将是令人敬畏的 Docker 列表。

# 令人敬畏的 Docker

在 GitHub 上可以找到令人敬畏的 Docker，网址为[`veggiemonk.github.io/awesome-docker/`](http://veggiemonk.github.io/awesome-docker/)。作者经常更新列表，因此您可以在本地克隆 Git 存储库，并定期更新以查看新内容。令人敬畏的 Docker 包含诸如 Docker 简介、工具（包括开发工具、测试或实用工具）等部分。视频部分在学习 Docker 时尤其有用，您可以在这里找到教程和培训。除了这个列表，很难找到更多有用的信息。

# 博客

我建议继续学习有关 Docker 的第一个博客将是 Arun Gupta 的博客，网址为[`blog.arungupta.me`](http://blog.arungupta.me)。Arun 于 2014 年 7 月开始首次撰写有关 Docker 的博客，他是 Couchbase 的开发者倡导副总裁，也是 Java 冠军、JUG 领导者和 Docker 船长。他在博客上写了很多东西；您可以使用`#docker`标签过滤与 Docker 相关的内容，链接为：[`blog.arungupta.me/tag/docker/`](http://blog.arungupta.me/tag/docker/)。

您将在这里找到许多与 Java 开发和 Docker 相关的有用信息。他还编写了一篇很棒的 Docker 教程，可在 GitHub 上找到：[`github.com/arun-gupta/docker-tutorial`](https://github.com/arun-gupta/docker-tutorial)。

接下来是官方的 Docker 博客，位于[`blog.docker.com`](https://blog.docker.com)。您在这里找不到有关如何使用 Docker 的许多教程，但会有关于新版本及其功能的公告，更高级的 Docker 使用技巧，以及 Docker 活动等社区新闻。

红帽开发者计划，位于容器类别下，可在[`developers.redhat.com/blog/category/containers/`](https://developers.redhat.com/blog/category/containers/)找到许多关于 Docker 和容器技术的有用文章。

# 互动教程

网上有许多 Docker 教程，但我发现其中一个特别有趣。这是 Katakoda 的互动式 Docker 学习课程，可在[`www.katacoda.com/courses/docker`](https://www.katacoda.com/courses/docker)找到。您将在这里找到 Docker 的完整功能集，从部署单个容器开始，然后涉及添加标签、检查容器和优化图像构建等主题。它是互动式的；您只需要一个现代浏览器，甚至不需要在本地机器上安装 Docker。它非常完整且学起来很有趣。另一个是[`training.play-with-docker.com`](http://training.play-with-docker.com/)。它包括三个部分：初学者，涵盖运行单个容器等基础知识，中级，涵盖网络等内容，以及高级，涵盖 Docker 安全性。其中一些课程任务是互动式的，您可以直接在浏览器中执行它们。

# Kubernetes

当 Docker 开始变得更受欢迎时，对容器管理平台的需求开始引起关注。因此，关于 Kubernetes 的更多资源开始在互联网上出现。

# 令人敬畏的 Kubernetes

类似于其 Docker 对应物，位于 GitHub [`github.com/ramitsurana/awesome-kubernetes`](https://github.com/ramitsurana/awesome-kubernetes)的令人敬畏的 Kubernetes 列表包含了许多关于 Kubernetes 的有用资源。您将在这里找到很多内容；从 Kubernetes 的介绍开始，通过有用工具和开发平台的列表，直到企业 Kubernetes 产品。甚至还有一个链接指向如何使用树莓派设备安装 Kubernetes 集群的教程！

# 教程

Kubernetes 官方网站包含许多有趣的教程，从基础知识开始，逐步介绍整个 Kubernetes 功能列表。教程列表可在[`kubernetes.io/docs/tutorials/`](https://kubernetes.io/docs/tutorials/)上找到。如果您还没有按照我们的 Minikube 安装指南进行操作，我强烈建议您这样做，使用 Kubernetes 官方的 Bootcamp，这是一个互动式的基于 Web 的教程，其目标是使用 Minikube 部署本地开发 Kubernetes 集群。它可在[`kubernetes.io/docs/tutorials/kubernetes-basics/cluster-interactive/`](https://kubernetes.io/docs/tutorials/kubernetes-basics/cluster-interactive/)上找到。

# 博客

Kubernetes 官方博客可在[`blog.kubernetes.io/`](http://blog.kubernetes.io/)找到。您将在这里找到有关新发布的公告、有用的技术文章和有趣的案例研究。

红帽企业 Linux 博客还包含许多有关 Kubernetes 的有趣文章。它们都标有 Kubernetes 标签，因此您可以通过使用链接[`rhelblog.redhat.com/tag/kubernetes/`](http://rhelblog.redhat.com/tag/kubernetes/)轻松过滤出它们。

# 扩展

正如您所知，Kubernetes 支持扩展。有一个很好的资源跟踪许多 Kubernetes，可在[`github.com/coreos/awesome-kubernetes-extensions`](https://github.com/coreos/awesome-kubernetes-extensions)上找到。例如，如果您需要将某些证书管理器集成到您的架构中，您可能会在那里找到合适的扩展。

# 工具

除了有用的文章和教程之外，还有一些有用的工具或平台，使使用 Kubernetes 更加愉快。现在让我们简要介绍它们。

# Rancher

Rancher，可在[`rancher.com`](http://rancher.com)找到，是一个在我们的书中值得单独介绍的平台。它是一个开源软件，可以轻松在任何基础设施上部署和管理 Docker 容器和 Kubernetes。您可以使用最完整的容器管理平台在任何基础设施上轻松部署和运行容器。

# Helm 和图表

Kubernetes Helm（在 GitHub 上可用：[`github.com/kubernetes/helm`](https://github.com/kubernetes/helm)）引入了图表的概念，这些图表是预配置的 Kubernetes 资源包，是为 Kubernetes 精心策划的应用程序定义。Helm 是一个管理图表的工具；它简化了安装和管理 Kubernetes 应用程序。可以将其视为 Kubernetes 的`apt/yum/homebrew`软件包管理器。您可以使用它来查找和使用打包为 Kubernetes 图表的热门软件，共享您自己的应用程序作为 Kubernetes 图表，并创建可重现的 Kubernetes 应用程序构建。当然，GitHub 上有一个专门的图表存储库：[`github.com/kubernetes/charts`](https://github.com/kubernetes/charts)。目前，图表二进制存储库可在 Google Cloud 上使用：[`console.cloud.google.com/storage/browser/kubernetes-charts/`](https://console.cloud.google.com/storage/browser/kubernetes-charts/)，其中包含许多有用的预打包工具，如 Ghost（`node.js`博客平台）、Jenkins、Joomla、MongoDb、MySQL、Redis、Minecraft 等等。

# Kompose

Kompose（[`github.com/kubernetes/kompose`](https://github.com/kubernetes/kompose)）是一个帮助将 Compose 配置文件移入 Kubernetes 的工具。Kompose 是一个用于定义和运行多容器 Docker 应用程序的工具。如果您是 Kompose 用户，可以使用它将多容器配置直接转换为 Kubernetes 设置，方法是将 Docker Compose 文件转换为 Kubernetes 对象。请注意，将 Docker Compose 格式转换为 Kubernetes 资源清单可能不会完全准确，但在首次在 Kubernetes 上部署应用程序时，它会有很大帮助。

# Kubetop

Kubetop，同样可在 GitHub 上找到：[`github.com/LeastAuthority/kubetop`](https://github.com/LeastAuthority/kubetop)，与 Kubernetes 集群的`top`命令相同。它非常有用；它列出了集群上所有正在运行的节点、它们上的所有 pod 以及这些 pod 中的所有容器。该工具为您提供有关每个节点的 CPU 和内存利用率的信息，类似于 Unix/Linux 的`top`命令。如果您需要快速了解集群上消耗最多资源的内容，这个快速的命令行工具是一个非常方便的选择。

# Kube-applier

在 GitHub 上可以找到`kube-applier`[`github.com/box/kube-applier`](https://github.com/box/kube-applier)，它可以为你的 Kubernetes 集群提供自动化部署和声明性配置。它作为一个 Kubernetes 服务运行，获取托管在 Git 存储库中的一组声明性配置文件，并将它们应用于 Kubernetes 集群。

`kube-applier`作为一个 Pod 在你的集群中运行，并持续监视 Git 存储库，以确保集群对象与存储库中的相关`spec`文件（JSON 或 YAML）保持最新。该工具还包含一个状态页面，并提供用于监控的指标。我发现它在日常开发中非常有用，特别是在你的部署、服务或 Pod 定义经常发生变化的情况下。

正如你所看到的，网络上有很多关于 Docker 和 Kubernetes 的有用资源。阅读完这本书后，你可能会想跳过大部分基础知识，直接进入更高级的话题。所有这些资源最好的一点是它们都是免费的，所以基本上没有什么能阻止你去探索受管容器的美妙世界。尝试学习，如果时机成熟，那就继续使用 Docker 和 Kubernetes 来部署你的生产就绪的 Java 软件，无论是在你自己的基础设施上还是在云上。看到你的 Java 应用如何自我扩展并变得无故障将会是令人惊叹的。Docker 和 Kubernetes 使其成为可能，而你现在已经具备了使用它们的知识。Docker 和 Kubernetes 彻底改变了技术领域的面貌，我希望它也能改变你的开发和发布流程，使之变得更好。

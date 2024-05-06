# 第十七章：企业工具

在本章中，我们将看一些 Docker 公司提供的企业级工具。我们将看到如何安装它们，配置它们，备份它们和恢复它们。

这将是一个相当长的章节，包含大量逐步的技术细节。我会尽量保持有趣，但这可能会很难:-D

其他工具也存在，但我们将集中在 Docker 公司的工具上。

让我们直接开始。

### 企业工具-简而言之

Docker 和容器已经席卷了应用程序开发世界-构建，发布和运行应用程序从未如此简单。因此，企业想要参与其中并不奇怪。但企业有比典型的前沿开发者更严格的要求。

企业需要以他们可以使用的方式打包 Docker。这通常意味着他们拥有和管理自己的本地解决方案。这还意味着角色和安全功能，使其适应其内部结构，并得到安全部门的认可。这也意味着一切都有一个有意义的支持合同支持。

这就是 Docker 企业版（EE）发挥作用的地方！

Docker EE 是*企业版的 Docker*。它是一套产品，包括一个经过加固的引擎，一个操作界面和一个安全的私有注册表。您可以在本地部署它，并且它包含在支持合同中。

高级堆栈如图 16.1 所示。

![图 16.1 Docker EE](img/figure16-1.png)

图 16.1 Docker EE

### 企业工具-深入挖掘

我们将把本章的其余部分分为以下几个部分：

+   Docker EE 引擎

+   Docker Universal Control Plane (UCP)

+   Docker Trusted Registry (DTR)

我们将看到如何安装每个工具，并在适用的情况下配置 HA，并执行备份和恢复作业。

#### Docker EE 引擎

*Docker 引擎*提供了所有核心的 Docker 功能。像镜像和容器管理，网络，卷，集群，秘密等等。在撰写本文时，有两个版本：

+   社区版（CE）

+   企业版（EE）

就我们而言，最大的两个区别是发布周期和支持。

Docker EE 按季度发布，并使用基于时间的版本方案。例如，2018 年 6 月发布的 Docker EE 将是`18.06.x-ee`。Docker 公司保证每个版本提供 1 年的支持和补丁。

##### 安装 Docker EE

安装 Docker EE 很简单。但是，不同平台之间存在细微差异。我们将向您展示如何在 Ubuntu 16.04 上进行操作，但在其他平台上进行操作同样简单。

Docker EE 是基于订阅的服务，因此您需要一个 Docker ID 和一个活跃的订阅。这将为您提供访问一个独特个性化的 Docker EE 仓库，我们将在接下来的步骤中配置和使用。[试用许可证](https://store.docker.com/editions/enterprise/docker-ee-trial)通常是可用的。

> **注意：** Windows Server 上的 Docker 始终安装 Docker EE。有关如何在 Windows Server 2016 上安装 Docker EE 的信息，请参阅第三章。

您可能需要在以下命令前加上`sudo`。

1.  确保您拥有最新的软件包列表。

[PRE0]

* 安装访问 Docker EE 仓库所需的软件包。

[PRE1]

* 登录[Docker Store](https://store.docker.com/)并复制您独特的 Docker EE 仓库 URL。

将您的网络浏览器指向 store.docker.com。点击右上角的用户名，然后选择`My Content`。在您的活动 Docker EE 订阅下选择`Setup`。

![](img/figure16-2.png)

从`Resources`窗格下复制您的仓库 URL。

同时下载您的许可证。

![](img/figure16-3.png)

我们正在演示如何为 Ubuntu 设置仓库。但是，这个 Docker Store 页面包含了如何为其他 Linux 版本进行设置的说明。

+   将您独特的 Docker EE 仓库 URL 添加到环境变量中。

[PRE2]

* 将官方 Docker GPG 密钥添加到所有密钥环中。

[PRE3]

* 设置最新的稳定仓库。您可能需要用最新的稳定版本替换最后一行的值。

[PRE4]

* 运行另一个`apt-get update`，以从您新添加的 Docker EE 仓库中获取最新的软件包列表。

[PRE5]

* 卸载先前版本的 Docker。

[PRE6]

* 安装 Docker EE

[PRE7]

* 检查安装是否成功。

[PRE8][PRE9][PRE10]

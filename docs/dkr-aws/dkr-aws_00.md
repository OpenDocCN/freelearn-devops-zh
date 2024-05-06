# 前言

欢迎阅读《在亚马逊网络服务上使用 Docker》！我非常兴奋能够写这本书，并分享如何利用 Docker 和亚马逊网络服务（AWS）生态系统提供的精彩技术，构建真正世界一流的解决方案，用于部署和运营您的应用程序。

Docker 已成为构建、打包、发布和运营应用程序的现代标准，利用容器的力量来提高应用程序交付速度，增加安全性并降低成本。本书将向您展示如何通过使用持续交付的最佳实践，来加速构建 Docker 应用程序的过程，提供一个完全自动化、一致、可靠和可移植的工作流程，用于测试、构建和发布您的 Docker 应用程序。在我看来，这是在考虑将应用程序部署到云端之前的基本先决条件，本书的前几章将重点介绍建立本地 Docker 环境，并为我们在整本书中将使用的示例应用程序创建一个本地持续交付工作流程。

AWS 是全球领先的公共云服务提供商，提供丰富的解决方案来管理和运营您的 Docker 应用程序。本书将涵盖 AWS 提供的所有主要服务，以支持 Docker 和容器，包括弹性容器服务（ECS）、Fargate、弹性 Beanstalk 和弹性 Kubernetes 服务（EKS），还将讨论您如何利用 Docker Inc 提供的 Docker for AWS 解决方案来部署 Docker Swarm 集群。

在 AWS 中运行完整的应用程序环境远不止您的容器平台，这本书还将描述管理访问 AWS 账户的最佳实践，并利用其他 AWS 服务来支持应用程序的要求。例如，您将学习如何设置 AWS 应用程序负载均衡器，为您的应用程序发布高可用、负载均衡的端点，创建 AWS 关系数据库服务（RDS）实例，提供托管的应用程序数据库，将您的应用程序集成到 AWS Secrets Manager 中，提供安全的秘密管理解决方案，并使用 AWS CodePipeline、CodeBuild 和 CloudFormation 服务创建完整的持续交付管道，该管道将自动测试、构建和发布 Docker 镜像，以适应您应用程序的任何新更改，并自动将其部署到开发和生产环境。

您将使用 AWS CloudFormation 服务构建所有这些支持基础设施，该服务提供了强大的基础设施即代码模板，允许您定义我提到的所有 AWS 服务和资源，并将其部署到 AWS，只需点击一个按钮。

我相信现在你一定和我一样对学习所有这些美妙的技术充满了期待，我相信在读完这本书之后，你将拥有部署和管理 Docker 应用程序所需的专业知识和技能，使用最新的前沿技术和最佳实践。

# 这本书适合谁

《在亚马逊网络服务上使用 Docker》适用于任何希望利用容器、Docker 和 AWS 的强大功能来构建、部署和操作应用程序的人。

读者最好具备对 Docker 和容器的基本理解，并且已经使用过 AWS 或其他云服务提供商，尽管不需要有容器或 AWS 的先前经验，因为这本书采用了一步一步的方法，并在您进展时解释关键概念。了解如何使用 Linux 命令行、Git 和基本的 Python 脚本知识将是有用的，但不是必需的。

请参阅“充分利用本书”部分，了解推荐的先决条件技能的完整列表。

# 本书涵盖了什么

第一章，“容器和 Docker 基础”，将简要介绍 Docker 和容器，并概述 AWS 中可用于运行 Docker 应用程序的各种服务和选项。您将设置您的本地环境，安装 Docker、Docker Compose 和其他各种工具，以完成每章中的示例。最后，您将下载示例应用程序，并学习如何在本地测试、构建和运行应用程序，以便您对应用程序的工作原理和您需要执行的特定任务有一个良好的理解，以使应用程序正常运行。

第二章，“使用 Docker 构建应用程序”，将描述如何构建一个完全自动化的基于 Docker 的工作流程，用于测试、构建、打包和发布您的应用程序作为生产就绪的 Docker 发布映像，使用 Docker、Docker Compose 和其他工具。这将建立一个便携式的持续交付工作流的基础，您可以在多台机器上一致地执行，而无需在每个本地环境中安装特定于应用程序的依赖项。

第三章，“开始使用 AWS”，将描述如何创建一个免费的 AWS 账户，并开始使用各种免费的服务，让您熟悉 AWS 提供的广泛的服务。您将学习如何为您的账户建立最佳实践的管理和用户访问模式，配置增强安全性的多因素身份验证（MFA）并安装 AWS 命令行界面，该界面可用于各种操作和自动化用例。您还将介绍 CloudFormation，这是 AWS 免费提供的管理工具和服务，您将在本书中使用它，它允许您使用强大而富有表现力的基础设施即代码模板格式，通过单击按钮部署复杂的环境。

第四章，ECS 简介，将帮助您快速上手弹性容器服务（ECS），这是在 AWS 中运行 Docker 应用程序的旗舰服务。您将了解 ECS 的架构，创建您的第一个 ECS 集群，使用 ECS 任务定义定义您的容器配置，然后将 Docker 应用程序部署为 ECS 服务。最后，您将简要介绍 ECS 命令行界面（CLI），它允许您与本地 Docker Compose 文件进行交互，并使用 ECS 自动部署 Docker Compose 资源到 AWS。

第五章，使用 ECR 发布 Docker 镜像，将教您如何使用弹性容器注册表（ECR）建立一个私有的 Docker 注册表，使用 IAM 凭证对您的注册表进行身份验证，然后将 Docker 镜像发布到注册表中的私有存储库。您还将学习如何与其他账户和 AWS 服务共享您的 Docker 镜像，以及如何配置生命周期策略以自动清理孤立的镜像，确保您只支付活动和当前的镜像。

第六章，构建自定义 ECS 容器实例，将向您展示如何使用一种流行的开源工具 Packer 来构建和发布自定义的 Amazon Machine Images（AMIs）用于在 ECS 集群中运行您的容器工作负载的 EC2 实例（ECS 容器实例）。您将安装一组辅助脚本，使您的实例能够与 CloudFormation 集成，并在实例创建时下载自定义的配置操作，从而使您能够动态配置 ECS 集群，配置实例应发布日志信息的 CloudWatch 日志组，并最终向 CloudFormation 发出信号，表明配置已成功或失败。

第七章，创建 ECS 集群，将教您如何基于利用上一章中创建的自定义 AMI 的特性来构建基于 EC2 自动扩展组的 ECS 集群。您将使用 CloudFormation 定义您的 EC2 自动扩展组、ECS 集群和其他支持资源，并配置 CloudFormation Init 元数据来执行自定义运行时配置和 ECS 容器实例的配置。

第八章，“使用 ECS 部署应用程序”，将扩展上一章创建的环境，添加支持资源，如关系数据库服务（RDS）实例和 AWS 应用程序负载均衡器（ALB）到您的 CloudFormation 模板中。然后，您将为示例应用程序定义一个 ECS 任务定义和 ECS 服务，并学习 ECS 如何执行应用程序的自动滚动部署和更新。为了编排所需的部署任务，如运行数据库迁移，您将扩展 CloudFormation 并编写自己的 Lambda 函数，创建一个 ECS 任务运行器自定义资源，提供运行任何可以作为 ECS 任务执行的配置操作的强大能力。

第九章，“管理机密”，将介绍 AWS Secrets Manager，这是一个完全托管的服务，可以以加密格式存储机密数据，被授权方（如您的用户、AWS 资源和应用程序）可以轻松安全地访问。您将使用 AWS CLI 与 Secrets Manager 进行交互，为敏感凭据（如数据库密码）创建机密，然后学习如何为容器创建入口脚本，在容器启动时将机密值作为内部环境变量注入，然后交给主应用程序。最后，您将创建一个 CloudFormation 自定义资源，将机密暴露给不支持 AWS Secrets Manager 的其他 AWS 服务，例如为关系数据库服务（RDS）实例提供管理密码。

第十章，“隔离网络访问”，描述了如何在 ECS 任务定义中使用 awsvpc 网络模式，以隔离网络访问，并将 ECS 控制平面通信与容器和应用程序通信分开。这将使您能够采用最佳安全实践模式，例如在私有网络上部署您的容器，并实现提供互联网访问的解决方案，包括 AWS VPC NAT 网关服务。

第十一章，“管理 ECS 基础设施生命周期”，将为您提供在运行 ECS 集群时的操作挑战的理解，其中包括将 ECS 容器实例移出服务，无论是为了缩减自动扩展组还是用新的 Amazon 机器映像替换 ECS 容器实例。您将学习如何利用 EC2 自动扩展生命周期挂钩，在 ECS 容器实例即将被终止时调用 AWS Lambda 函数，这允许您执行优雅的关闭操作，例如将活动容器转移到集群中的其他实例，然后发出 EC2 自动扩展以继续实例终止。

第十二章，“ECS 自动扩展”，将描述 ECS 集群如何管理 CPU、内存和网络端口等资源，以及这如何影响您的集群容量。如果您希望能够动态自动扩展您的集群，您需要动态监视 ECS 集群容量，并在容量阈值处扩展或缩减集群，以确保您将满足组织或用例的服务水平期望。您将实施一个解决方案，该解决方案在通过 AWS CloudWatch Events 服务生成 ECS 容器实例状态更改事件时计算 ECS 集群容量，将容量指标发布到 CloudWatch，并使用 CloudWatch 警报动态地扩展或缩减您的集群。有了动态集群容量解决方案，您将能够配置 AWS 应用程序自动扩展服务，根据适当的指标（如 CPU 利用率或活动连接）动态调整服务实例的数量，而无需担心对底层集群容量的影响。

第十三章，“持续交付 ECS 应用程序”，将使用 AWS CodePipeline 服务建立一个持续交付流水线，该流水线与 GitHub 集成，以侦测应用程序源代码和基础设施部署脚本的更改，使用 AWS CodeBuild 服务运行单元测试，构建应用程序构件并使用示例应用程序 Docker 工作流发布 Docker 镜像，并使用本书中迄今为止使用的 CloudFormation 模板持续部署您的应用程序更改到 AWS。

这将自动部署到一个您测试的 AWS 开发环境中，然后创建一个变更集和手动批准操作，以便将其部署到生产环境，为您的所有应用程序新功能和错误修复提供了一个快速且可重复的生产路径。

第十四章《Fargate 和 ECS 服务发现》将介绍 AWS Fargate，它提供了一个完全管理传统上需要使用常规 ECS 服务来管理的 ECS 服务控制平面和 ECS 集群的解决方案。您将使用 Fargate 部署 AWS X-Ray 守护程序作为 ECS 服务，并配置 ECS 服务发现，动态发布您的服务端点使用 DNS 和 Route 53。这将允许您为您的示例应用程序添加对 X-Ray 跟踪的支持，该跟踪可用于跟踪传入的 HTTP 请求到您的应用程序，并监视 AWS 服务调用、数据库调用和其他类型的调用，这些调用是为了服务每个传入请求。

第十五章《弹性 Beanstalk》将介绍流行的**平台即服务**（**PaaS**）提供的概述，其中包括对 Docker 应用程序的支持。您将学习如何创建一个弹性 Beanstalk 多容器 Docker 应用程序，建立一个由托管的 EC2 实例、一个 RDS 数据库实例和一个**应用负载均衡器**（**ALB**）组成的环境，然后使用各种技术扩展环境，以支持 Docker 应用程序的要求，例如卷挂载和在每个应用程序部署中运行单次任务。

第十六章，*AWS 中的 Docker Swarm*，将重点介绍如何在 AWS 中运行 Docker Swarm 集群，使用为 Docker Swarm 社区版提供的 Docker for AWS 蓝图。该蓝图为您提供了一个 CloudFormation 模板，在几分钟内在 AWS 中建立一个预配置的 Docker Swarm 集群，并与关键的 AWS 服务（如弹性负载均衡（ELB）、弹性文件系统（EFS）和弹性块存储（EBS）服务）进行集成。您将使用 Docker Compose 定义一个堆栈，该堆栈配置了以熟悉的 Docker Compose 规范格式表示的多服务环境，并学习如何配置关键的 Docker Swarm 资源，如服务、卷和 Docker 秘密。您将学习如何创建由 EFS 支持的共享 Docker 卷，由 EBS 支持的可重定位 Docker 卷，Docker Swarm 将在节点故障后自动重新连接到重新部署的新容器，并使用由 Docker Swarm 自动创建和管理的 ELB 为您的应用程序发布外部服务端点。

第十七章，*弹性 Kubernetes 服务*，介绍了 AWS 最新的容器管理平台，该平台基于流行的开源 Kubernetes 平台。您将首先在本地 Docker Desktop 环境中设置 Kubernetes，该环境包括 Docker 18.06 CE 版本对 Kubernetes 的本机支持，并学习如何使用多个 Kubernetes 资源（包括 pod、部署、服务、秘密、持久卷和作业）为您的 Docker 应用创建完整的环境。然后，您将在 AWS 中建立一个 EKS 集群，创建一个 EC2 自动扩展组，将工作节点连接到您的集群，并确保您的本地 Kubernetes 客户端可以对 EKS 控制平面进行身份验证和连接，之后您将部署 Kubernetes 仪表板，为您的集群提供全面的管理界面。最后，您将定义一个使用弹性块存储（EBS）服务的默认存储类，然后将您的 Docker 应用部署到 AWS，利用您之前为本地环境创建的相同 Kubernetes 定义，为您提供一个强大的解决方案，可以快速部署用于开发目的的 Docker 应用程序，然后使用 EKS 直接部署到生产环境。

# 为了充分利用本书

+   Docker 的基本工作知识-如果您以前没有使用过 Docker，您应该了解[`docs.docker.com/engine/docker-overview/`](https://docs.docker.com/engine/docker-overview/)上 Docker 的基本概念，然后按照 Docker 入门教程的第一部分([`docs.docker.com/get-started/`](https://docs.docker.com/get-started/))和第二部分([`docs.docker.com/get-started/part2`](https://docs.docker.com/get-started/part2))进行学习。要更全面地了解 Docker，请查看 Packt Publishing 的[Learn Docker - Fundamentals of Docker 18.x](https://www.packtpub.com/networking-and-servers/learn-docker-fundamentals-docker-18x)书籍。

+   Git 的基本工作知识-如果您以前没有使用过 Git，您应该运行[`www.atlassian.com/git/tutorials`](https://www.atlassian.com/git/tutorials)上的初学者和入门教程。要更全面地了解 Git，请查看 Packt Publishing 的[Git Essentials](https://www.packtpub.com/application-development/git-essentials)书籍。

+   熟悉 AWS-如果您以前没有使用过 AWS，运行[`aws.amazon.com/getting-started/tutorials/launch-a-virtual-machine/`](https://aws.amazon.com/getting-started/tutorials/launch-a-virtual-machine/)上的启动 Linux 虚拟机教程将提供有用的介绍。

+   熟悉 Linux/Unix 命令行-如果您以前没有使用过 Linux/Unix 命令行，我建议您运行一个基本教程，比如[`maker.pro/linux/tutorial/basic-linux-commands-for-beginners`](https://maker.pro/linux/tutorial/basic-linux-commands-for-beginners)，使用您在完成启动 Linux 虚拟机教程时创建的 Linux 虚拟机。

+   Python 的基本理解-本书的示例应用程序是用 Python 编写的，后面章节的一些示例包括基本的 Python 脚本。如果您以前没有使用过 Python，您可能希望阅读[`docs.python.org/3/tutorial/`](https://docs.python.org/3/tutorial)的前几课。

# 下载示例代码文件

您可以从[www.packtpub.com](http://www.packtpub.com)的帐户中下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问[www.packtpub.com/support](http://www.packtpub.com/support)并注册，以便文件直接通过电子邮件发送给您。

您可以按照以下步骤下载代码文件：

1.  登录或注册[www.packtpub.com](http://www.packtpub.com/support)。

1.  选择“支持”选项卡。

1.  单击“代码下载和勘误”。

1.  在搜索框中输入书名，然后按照屏幕上的说明操作。

文件下载后，请确保使用最新版本的解压软件解压文件夹：

+   WinRAR/7-Zip 适用于 Windows

+   Zipeg/iZip/UnRarX 适用于 Mac

+   7-Zip/PeaZip 适用于 Linux

该书的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/Docker-on-Amazon-Web-Services`](https://github.com/PacktPublishing/Docker-on-Amazon-Web-Services)。如果代码有更新，将在现有的 GitHub 存储库中进行更新。

我们还有其他代码包，来自我们丰富的图书和视频目录，可在**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**上找到。去看看吧！

# 下载彩色图像

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。您可以在这里下载：[`www.packtpub.com/sites/default/files/downloads/DockeronAmazonWebServices_ColorImages.pdf`](https://www.packtpub.com/sites/default/files/downloads/DockeronAmazonWebServices_ColorImages.pdf)

# 代码演示

访问以下链接查看代码运行的视频：

[`bit.ly/2Noqdpn`](http://bit.ly/2Noqdpn)

# 使用的约定

本书中使用了许多文本约定。

`CodeInText`：表示文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。例如：“请注意，要点中包含了一个名为`PASTE_ACCOUNT_NUMBER`的占位符，位于策略文档中，因此您需要将其替换为您的实际 AWS 账户 ID。”

代码块设置如下：

```
AWSTemplateFormatVersion: "2010-09-09"

Description: Cloud9 Management Station

Parameters:
  EC2InstanceType:
    Type: String
    Description: EC2 instance type
    Default: t2.micro
  SubnetId:
    Type: AWS::EC2::Subnet::Id
    Description: Target subnet for instance
```

任何命令行输入或输出都以以下方式编写：

```
> aws configure
AWS Access Key ID [None]:
```

**粗体**：表示新术语、重要单词或屏幕上看到的单词。例如，菜单中的单词或对话框中的单词会以这种方式出现在文本中。例如：“要创建管理员角色，请从 AWS 控制台中选择**服务**|**IAM**，从左侧菜单中选择**角色**，然后单击**创建角色**按钮。”

警告或重要说明会出现在这样的样式中。提示和技巧会出现在这样的样式中。

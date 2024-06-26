# 前言

本书将带您学习 DevOps、容器和 Kubernetes 的基本概念和有用技能的旅程。

# 本书涵盖的内容

第一章《DevOps 简介》带您了解了从过去到今天我们所说的 DevOps 的演变以及您应该了解的工具。近几年对具有 DevOps 技能的人的需求一直在迅速增长。它加速了软件开发和交付速度，也帮助了业务的敏捷性。

第二章《使用容器进行 DevOps》帮助您学习基本概念和容器编排。随着微服务的趋势，容器已成为每个 DevOps 的便捷和必要工具，因为它具有语言不可知的隔离性。

第三章《使用 Kubernetes 入门》探讨了 Kubernetes 中的关键组件和 API 对象，以及如何在 Kubernetes 集群中部署和管理容器。Kubernetes 通过许多强大的功能（如容器扩展、挂载存储系统和服务发现）简化了容器编排的痛苦。

第四章《存储和资源管理》描述了卷管理，并解释了 Kubernetes 中的 CPU 和内存管理。在集群中进行容器存储管理可能很困难。

第五章《网络和安全》解释了如何允许入站连接访问 Kubernetes 服务，以及 Kubernetes 中默认网络的工作原理。对我们的服务进行外部访问对业务需求是必要的。

第六章《监控和日志记录》向您展示如何使用 Prometheus 监视应用程序、容器和节点级别的资源使用情况。本章还展示了如何从您的应用程序以及 Kubernetes 中收集日志，以及如何使用 Elasticsearch、Fluentd 和 Kibana 堆栈。确保服务正常运行和健康是 DevOps 的主要责任之一。

*第七章，持续交付*，解释了如何使用 GitHub/DockerHub/TravisCI 构建持续交付管道。它还解释了如何管理更新，消除滚动更新时可能的影响，并防止可能的失败。持续交付是加快上市时间的一种方法。

*第八章，集群管理*，描述了如何使用 Kubernetes 命名空间和 ResourceQuota 解决前述问题，以及如何在 Kubernetes 中进行访问控制。建立管理边界和对 Kubernetes 集群进行访问控制对 DevOps 至关重要。

*第九章，AWS 上的 Kubernetes*，解释了 AWS 组件，并展示了如何在 AWS 上部署 Kubernetes。AWS 是最受欢迎的公共云。它为我们的世界带来了基础设施的灵活性和敏捷性。

*第十章，GCP 上的 Kubernetes*，帮助您了解 GCP 和 AWS 之间的区别，以及从 Kubernetes 的角度来看在托管服务中运行容器化应用的好处。GCP 中的 Google 容器引擎是 Kubernetes 的托管环境。

*第十一章，接下来是什么？*，介绍了其他类似的技术，如 Docker Swarm 模式、Amazon ECS 和 Apache Mesos，您将了解哪种方法对您的业务最为合适。Kubernetes 是开放的。本章将教您如何与 Kubernetes 社区联系，学习他人的想法。

# 本书所需内容

本书将指导您通过 macOS 和公共云（AWS 和 GCP）使用 Docker 容器和 Kubernetes 进行软件开发和交付的方法论。您需要安装 minikube、AWSCLI 和 Cloud SDK 来运行本书中的代码示例。

# 本书的受众

本书适用于具有一定软件开发经验的 DevOps 专业人士，他们愿意将软件交付规模化、自动化并缩短上市时间。

# 惯例

在本书中，您将找到许多区分不同信息类型的文本样式。以下是一些样式的示例及其含义的解释。

文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 用户名显示如下："将下载的`WebStorm-10*.dmg`磁盘映像文件挂载为系统中的另一个磁盘。"

任何命令行输入或输出都是这样写的：

```
$ sudo yum -y -q install nginx
$ sudo /etc/init.d/nginx start
Starting nginx: 
```

**新术语**和**重要单词**以粗体显示。您在屏幕上看到的单词，例如菜单或对话框中的单词，会在文本中出现，就像这样："本书中的快捷键基于 Mac OS X 10.5+方案。"

警告或重要提示会出现在这样的地方。提示和技巧会出现在这样的地方。

# 前言

容器是近年来最受关注的技术之一。随着它们改变了我们开发、部署和运行软件应用程序的方式，它们变得越来越受欢迎。OpenStack 因被全球许多组织使用而获得了巨大的关注，随着容器的普及和复杂性的增加，OpenStack 需要为容器提供各种基础设施资源，如计算、网络和存储。

*OpenStack 中的容器化*旨在回答一个问题，那就是 OpenStack 如何跟上容器技术不断增长的挑战？您将首先熟悉容器和 OpenStack 的基础知识，以便了解容器生态系统和 OpenStack 如何协同工作。为了帮助您更好地掌握计算、网络、管理应用服务和部署工具，本书为不同的 OpenStack 项目专门设置了章节：Magnum、Zun、Kuryr、Murano 和 Kolla。

最后，您将了解一些保护容器和 OpenStack 上的容器编排引擎的最佳实践，并概述了如何使用每个 OpenStack 项目来处理不同的用例。

# 本书内容包括：

第一章，*使用容器*，从讨论虚拟化的历史开始，然后讨论容器的演变。之后，重点解释了容器、它们的类型和不同的容器运行时工具。然后深入介绍了 Docker 及其安装，并展示了如何使用 Docker 对容器执行操作。

第二章，*使用容器编排引擎*，从介绍容器编排引擎开始，然后介绍了当今可用的不同容器编排引擎。它解释了 Kubernetes 的安装以及如何在示例应用程序中使用它来管理容器。

第三章，*OpenStack 架构*，从介绍 OpenStack 及其架构开始。然后简要解释了 OpenStack 的核心组件及其架构。

第四章，*OpenStack 中的容器化*，解释了 OpenStack 中容器化的需求，并讨论了不同的与 OpenStack 相关的容器项目。

《第五章》（part0124.html#3M85O0-08510d04d33546e798ef8c1140114deb），*Magnum – OpenStack 中的 COE 管理*，详细介绍了 OpenStack 的 Magnum 项目。它讨论了 Magnum 的概念、组件和架构。然后，它演示了使用 DevStack 安装 Magnum 并进行实际操作。

《第六章》（part0152.html#4GULG0-08510d04d33546e798ef8c1140114deb），*Zun – OpenStack 中的容器管理*，详细介绍了 OpenStack 的 Zun 项目。它讨论了 Zun 的概念、组件和架构。然后，它演示了使用 DevStack 安装 Zun 并进行实际操作。

《第七章》（part0178.html#59O440-08510d04d33546e798ef8c1140114deb），*Kuryr – OpenStack 网络的容器插件*，详细介绍了 OpenStack 的 Kuryr 项目。它讨论了 Kuryr 的概念、组件和架构。然后，它演示了使用 DevStack 安装 Kuryr 并进行实际操作。

《第八章》（part0188.html#5J99O0-08510d04d33546e798ef8c1140114deb），*Murano – 在 OpenStack 上部署容器化应用*，详细介绍了 OpenStack 的 Murano 项目。它讨论了 Murano 的概念、组件和架构。然后，它演示了使用 DevStack 安装 Murano 并进行实际操作。

《第九章》（part0216.html#6DVPG0-08510d04d33546e798ef8c1140114deb），*Kolla – OpenStack 的容器化部署*，详细介绍了 OpenStack 的 Kolla 项目。它讨论了 Kolla 的子项目、主要特点和架构。然后，它解释了使用 Kolla 项目部署 OpenStack 生态系统的过程。

《第十章》（part0233.html#6U6J20-08510d04d33546e798ef8c1140114deb），*容器和 OpenStack 的最佳实践*，总结了不同与容器相关的 OpenStack 项目及其优势。然后，它还解释了容器的安全问题以及解决这些问题的最佳实践。

# 本书所需的内容

本书假定读者具有基本的云计算、Linux 操作系统和容器的理解。本书将指导您安装所需的任何工具。

您可以使用任何测试环境的工具，如 Vagrant、Oracle 的 VirtualBox 或 VMware 工作站。

在本书中，需要以下软件清单：

+   操作系统：Ubuntu 16.04

+   OpenStack：Pike 版本或更新版本

+   VirtualBox 4.5 或更新版本

+   Vagrant 1.7 或更新版本

要在开发环境中运行 OpenStack 安装，需要以下最低硬件资源：

+   具有 CPU 硬件虚拟化支持的主机

+   8 核 CPU

+   12 GB 的 RAM

+   60 GB 的免费磁盘空间

需要互联网连接来下载 OpenStack 和其他工具的必要软件包。

# 这本书适合谁

这本书的目标读者是云工程师、系统管理员，或者任何在 OpenStack 云上工作的生产团队成员。本书是一本端到端的指南，适用于任何想要在 OpenStack 中开始使用容器化概念的人。

# 约定

在本书中，您会发现一些不同类型信息的文本样式。以下是一些这些样式的例子，以及它们的含义解释。

文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄显示如下：“`zun-compute`服务是 Zun 系统的主要组件。”

任何命令行输入或输出都以以下形式编写：

```
$ sudo mkdir -p /opt/stack
```

新术语和重要词以粗体显示。例如，在屏幕上看到的单词，比如菜单或对话框中的单词，会以这样的形式出现在文本中：“您可以在以下截图中看到，我们有两个选项可供选择作为我们的容器主机：Kubernetes Pod 和 Docker Standalone Host。”

警告或重要提示会出现在这样的框中。提示和技巧会以这样的形式出现。

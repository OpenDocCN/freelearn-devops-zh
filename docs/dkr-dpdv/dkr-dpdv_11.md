# 第十章：使用 Docker Compose 部署应用程序

在本章中，我们将看看如何使用 Docker Compose 部署多容器应用程序。

Docker Compose 和 Docker Stacks 非常相似。在本章中，我们将专注于 Docker Compose，它在**单引擎模式**下部署和管理多容器应用程序在 Docker 节点上运行。在后面的章节中，我们将专注于 Docker Stacks。Stacks 在**Swarm 模式**下部署和管理多容器应用程序。

我们将把这一章分成通常的三个部分：

+   简而言之

+   深入研究

+   命令

### 使用 Compose 部署应用程序-简而言之

大多数现代应用程序由多个较小的服务组成，这些服务相互交互形成一个有用的应用程序。我们称之为微服务。一个简单的例子可能是一个具有以下四个服务的应用程序：

+   Web 前端

+   排序

+   目录

+   后端数据库

把所有这些放在一起，你就有了一个*有用的应用程序*。

部署和管理大量服务可能很困难。这就是*Docker Compose*发挥作用的地方。

与使用脚本和冗长的`docker`命令将所有内容粘合在一起不同，Docker Compose 允许你在一个声明性的配置文件中描述整个应用程序。然后你可以用一个命令部署它。

一旦应用程序被*部署*，你可以用一组简单的命令*管理*它的整个生命周期。你甚至可以将配置文件存储和管理在版本控制系统中！这一切都非常成熟 :-D

这就是基础知识。让我们深入了解一下。

### 使用 Compose 部署应用程序-深入研究

我们将把深入研究部分分为以下几个部分：

+   Compose 背景

+   安装 Compose

+   Compose 文件

+   使用 Compose 部署应用程序

+   使用 Compose 管理应用程序

#### Compose 背景

一开始是*Fig*。Fig 是一个强大的工具，由一个叫做*Orchard*的公司创建，它是管理多容器 Docker 应用程序的最佳方式。它是一个 Python 工具，位于 Docker 之上，允许你在一个单独的 YAML 文件中定义整个多容器应用程序。然后你可以用`fig`命令行工具部署应用程序。Fig 甚至可以管理整个应用程序的生命周期。

在幕后，Fig 会读取 YAML 文件，并通过 Docker API 部署和管理应用程序。这是一件好事！

事实上，2014 年，Docker 公司收购了 Orchard 并将 Fig 重新命名为*Docker Compose*。命令行工具从`fig`改名为`docker-compose`，自收购以来，它一直是一个外部工具，可以附加到 Docker Engine 上。尽管它从未完全集成到 Docker Engine 中，但它一直非常受欢迎并被广泛使用。

就目前而言，Compose 仍然是一个外部的 Python 二进制文件，您必须在运行 Docker Engine 的主机上安装它。您可以在 YAML 文件中定义多容器（多服务）应用程序，将 YAML 文件传递给`docker-compose`二进制文件，然后 Compose 通过 Docker Engine API 部署它。

是时候看它的表现了。

#### 安装 Compose

Docker Compose 在多个平台上都可用。在本节中，我们将演示在 Windows、Mac 和 Linux 上安装它的*一些*方法。还有更多的安装方法，但我们在这里展示的方法可以让您开始。

##### 在 Windows 10 上安装 Compose

在 Windows 10 上运行 Docker 的推荐方式是*Docker for Windows (DfW)*。Docker Compose 包含在标准的 DfW 安装中。因此，如果您安装了 DfW，您就有了 Docker Compose。

使用以下命令检查是否已安装 Compose。您可以从 PowerShell 或 CMD 终端运行此命令。

[PRE0]

如果您需要有关在 Windows 10 上安装*Docker for Windows*的更多信息，请参阅**第三章：安装 Docker**。

##### 在 Mac 上安装 Compose

与 Windows 10 一样，Docker Compose 作为*Docker for Mac (DfM)*的一部分安装。因此，如果您安装了 DfM，您就有了 Docker Compose。

从终端窗口运行以下命令以验证您是否安装了 Docker Compose。

[PRE1]

如果您需要有关在 Mac 上安装*Docker for Mac*的更多信息，请参阅**第三章：安装 Docker**。

##### 在 Windows Server 上安装 Compose

Docker Compose 作为一个独立的二进制文件安装在 Windows Server 上。要使用它，您需要在 Windows Server 上安装最新的 Docker。

在 PowerShell 终端中键入以下命令以安装 Docker Compose。

为了可读性，该命令使用反引号（`）来转义回车并将命令包装在多行上。

以下命令安装了 Docker Compose 的`1.18.0`版本。您可以安装此处列出的任何版本：https://github.com/docker/compose/releases。只需用您想要安装的版本替换 URL 中的`1.18.0`。

[PRE2]

使用`docker-compose --version`命令验证安装。

[PRE3]

Compose 现在已安装。只要您的 Windows Server 机器安装了最新版本的 Docker Engine，您就可以开始了。

##### 在 Linux 上安装 Compose

在 Linux 上安装 Docker Compose 是一个两步过程。首先，您使用`curl`命令下载二进制文件。然后使用`chmod`使其可执行。

要使 Docker Compose 在 Linux 上工作，您需要一个可用的 Docker Engine 版本。

以下命令将下载 Docker Compose 的版本`1.18.0`并将其复制到`/usr/bin/local`。您可以在[GitHub](https://github.com/docker/compose/releases)的发布页面上检查最新版本，并将 URL 中的`1.18.0`替换为您想要安装的版本。

该命令可能会在书中跨越多行。如果您在一行上运行该命令，您需要删除任何反斜杠（`\`）。

[PRE4][PRE5]``-[PRE6][PRE7]

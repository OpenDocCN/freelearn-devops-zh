# 第一章：运行我的第一个 Docker 容器

概述

在本章中，您将学习 Docker 和容器化的基础知识，并探索将传统的多层应用程序迁移到快速可靠的容器化基础设施的好处。通过本章的学习，您将对运行容器化应用程序的好处有深入的了解，以及使用`docker run`命令运行容器的基础知识。本章不仅将向您介绍 Docker 的基础知识，还将为您提供对本次研讨会中将要构建的 Docker 概念的扎实理解。

# 介绍

近年来，各行各业的技术创新迅速增加了软件产品交付的速度。由于技术趋势，如敏捷开发（一种快速编写软件的方法）和持续集成管道，使软件的快速交付成为可能，运营人员最近一直在努力快速构建基础设施，以满足不断增长的需求。为了跟上发展，许多组织选择迁移到云基础设施。

云基础设施提供了托管的虚拟化、网络和存储解决方案，可以按需使用。这些提供商允许任何组织或个人注册并获得传统上需要大量空间和昂贵硬件才能在现场或数据中心实施的基础设施。云提供商，如亚马逊网络服务和谷歌云平台，提供易于使用的 API，允许几乎立即创建大量的虚拟机（或 VMs）。

将基础设施部署到云端为组织面临的许多传统基础设施解决了难题，但也带来了与在规模上运行这些服务相关的管理成本的额外问题。公司如何管理全天候运行昂贵服务器的持续月度和年度支出？

虚拟机通过利用 hypervisors 在较大的硬件之上创建较小的服务器，从而革新了基础设施。虚拟化的缺点在于运行虚拟机的资源密集程度。虚拟机本身看起来、行为和感觉都像真正的裸金属硬件，因为 hypervisors（如 Zen、KVM 和 VMWare）分配资源来引导和管理整个操作系统镜像。与虚拟机相关的专用资源使其变得庞大且难以管理。在本地 hypervisor 和云之间迁移虚拟机可能意味着每个虚拟机移动数百 GB 的数据。

为了提供更高程度的自动化，更好地利用计算密度，并优化他们的云存在，公司发现自己朝着容器化和微服务架构的方向迈进作为解决方案。容器提供了进程级别的隔离，或者在主机操作系统内核的隔离部分内运行软件服务。与运行整个操作系统内核以提供隔离不同，容器可以共享主机操作系统的内核来运行多个软件应用程序。这是通过 Linux 内核中的控制组（或 cgroups）和命名空间隔离等功能实现的。在单个虚拟机或裸金属机器上，用户可能会运行数百个容器，这些容器在单个主机操作系统上运行各自的软件应用程序实例。

这与传统的虚拟机架构形成鲜明对比。通常，当我们部署虚拟机时，我们目的是让该机器运行单个服务器或一小部分服务。这会导致宝贵的 CPU 周期的浪费，这些周期本可以分配给其他任务并提供其他请求。理论上，我们可以通过在单个虚拟机上安装多个服务来解决这个困境。然而，这可能会在关于哪台机器运行哪项服务方面造成极大的混乱。它还将多个软件安装和后端依赖项的托管权放在单个操作系统中。

容器化的微服务方法通过允许容器运行时在主机操作系统上调度和运行容器来解决了这个问题。容器运行时不关心容器内运行的是什么应用程序，而是关心容器是否存在，并且可以在主机操作系统上下载和执行。容器内运行的应用程序是 Go web API、简单的 Python 脚本还是传统的 Cobol 应用程序都无关紧要。由于容器是以标准格式存在的，容器运行时将下载容器镜像并在其中执行软件。在本书中，我们将学习 Docker 容器运行时，并学习在本地和规模化运行容器的基础知识。

Docker 是一个容器运行时，于 2013 年开发，旨在利用 Linux 内核的进程隔离功能。与其他容器运行时实现不同的是，Docker 开发了一个系统，不仅可以运行容器，还可以构建和推送容器到容器存储库。这一创新引领了容器不可变性的概念——只有在软件发生变化时才通过构建和推送容器的新版本来改变容器。

如下图所示（*图 1.1*），我们在两个 Docker 服务器上部署了一系列容器化应用程序。在两个服务器实例之间，部署了七个容器化应用程序。每个容器都托管着自己所需的二进制文件、库和自包含的依赖关系。当 Docker 运行一个容器时，容器本身承载了其正常运行所需的一切。甚至可以部署同一应用程序框架的不同版本，因为每个容器都存在于自己的内核空间中。

![图 1.1：在两个不同的容器服务器上运行的七个容器](img/B15021_01_01.jpg)

图 1.1：在两个不同的容器服务器上运行的七个容器

在本章中，您将通过容器化的帮助了解 Docker 提供的各种优势。您还将学习使用`docker run`命令来运行容器的基础知识。

# 使用 Docker 的优势

在传统的虚拟机方法中，代码更改需要运维人员或配置管理工具访问该机器并安装软件的新版本。不可变容器的原则意味着当代码更改发生时，将构建新版本的容器映像，并创建为新的构件。如果需要回滚此更改，只需下载并重新启动容器映像的旧版本就可以了。

利用容器化方法还使软件开发团队能够在各种场景和多个环境中可预测和可靠地测试应用程序。由于 Docker 运行时环境提供了标准的执行环境，软件开发人员可以快速重现问题并轻松调试问题。由于容器的不可变性，开发人员可以确保相同的代码在所有环境中运行，因为相同的 Docker 映像可以部署在任何环境中。这意味着配置变量，如无效的数据库连接字符串、API 凭据或其他特定于环境的差异，是故障的主要来源。这减轻了运维负担，并提供了无与伦比的效率和可重用性。

使用 Docker 的另一个优势是，与传统基础设施相比，容器化应用程序通常更小、更灵活。容器通常只提供运行应用程序所需的必要库和软件包，而不是提供完整的操作系统内核和执行环境。

在构建 Docker 容器时，开发人员不再受限于主机操作系统上安装的软件包和工具，这些可能在不同环境之间有所不同。他们可以将容器映像中仅包含应用程序运行所需的确切版本的库和实用程序。在部署到生产机器上时，开发人员和运维团队不再关心容器运行在什么硬件或操作系统版本上，只要他们的容器在运行就可以了。

例如，截至 2020 年 1 月 1 日，Python 2 不再受支持。因此，许多软件仓库正在逐步淘汰 Python 2 包和运行时。利用容器化方法，您可以继续以受控、安全和可靠的方式运行传统的 Python 2 应用程序，直到这些传统应用程序可以被重写。这消除了担心安装操作系统级补丁的恐惧，这些补丁可能会移除 Python 2 支持并破坏传统应用程序堆栈。这些 Python 2 容器甚至可以与 Docker 服务器上的 Python 3 应用程序并行运行，以在这些应用程序迁移到新的现代化堆栈时提供精确的测试。

现在我们已经了解了 Docker 是什么以及它是如何工作的，我们可以开始使用 Docker 来了解进程隔离与虚拟化和其他类似技术的区别。

注意

在我们开始运行容器之前，您必须在本地开发工作站上安装 Docker。有关详细信息，请查看本书的*前言*部分。

# Docker 引擎

**Docker 引擎**是提供对 Linux 内核进程隔离功能的接口。由于只有 Linux 暴露了允许容器运行的功能，因此 Windows 和 macOS 主机利用后台的 Linux 虚拟机来实现容器执行。对于 Windows 和 macOS 用户，Docker 提供了“**Docker 桌面**”套件，用于在后台部署和运行这个虚拟机。这允许从 macOS 或 Windows 主机的终端或 PowerShell 控制台本地执行 Docker 命令。Linux 主机有特权直接在本地执行 Docker 引擎，因为现代版本的 Linux 内核支持`cgroups`和命名空间隔离。

注意

由于 Windows、macOS 和 Linux 在网络和进程管理方面具有根本不同的操作系统架构，本书中的一些示例（特别是在网络方面）有时会根据在您的开发工作站上运行的操作系统而有不同的行为。这些差异会在出现时进行说明。

Docker 引擎不仅支持执行容器镜像，还提供了内置机制，可以从源代码文件（称为`Dockerfiles`）构建和测试容器镜像。构建容器镜像后，可以将其推送到容器镜像注册表。**镜像注册表**是容器镜像的存储库，其他 Docker 主机可以从中下载和执行容器镜像。Docker 引擎支持运行容器镜像、构建容器镜像，甚至在配置为这样运行时托管容器镜像注册表。

当容器启动时，Docker 默认会下载容器镜像，将其存储在本地容器镜像缓存中，最后执行容器的`entrypoint`指令。`entrypoint`指令是启动应用程序主要进程的命令。当这个进程停止或关闭时，容器也将停止运行。

根据容器内运行的应用程序，`entrypoint`指令可能是长期运行的服务器守护程序，始终可用，或者可能是一个短暂的脚本，在执行完成后自然停止。另外，许多容器执行`entrypoint`脚本，在启动主要进程之前完成一系列设置步骤，这可能是长期或短期的。

在运行任何容器之前，最好先了解将在容器内运行的应用程序类型，以及它是短暂执行还是长期运行的服务器守护程序。

# 运行 Docker 容器

构建容器和微服务架构的最佳实践规定，一个容器应该只运行一个进程。牢记这一原则，我们可以设计容器，使其易于构建、故障排除、扩展和部署。

容器的生命周期由容器的状态和其中运行的进程定义。根据操作员、容器编排器或容器内部运行的应用程序的状态，容器可以处于运行或停止状态。例如，操作员可以使用`docker stop`或`docker start`命令手动停止或启动容器。如果 Docker 检测到容器进入不健康状态，它甚至可能自动停止或重新启动容器。此外，如果容器内部运行的主要应用程序失败或停止，运行的容器实例也应该停止。许多容器运行时平台，如 Docker，甚至提供自动机制来自动重新启动进入停止状态的容器。许多容器平台利用这一原则构建作业和任务执行功能。

由于容器在容器内部的主要进程完成时终止，容器是执行脚本和其他类型的具有无限寿命的作业的优秀平台。下面的*图 1.2*说明了典型容器的生命周期：

![图 1.2：典型容器的生命周期](img/B15021_01_02.jpg)

图 1.2：典型容器的生命周期

一旦在目标操作系统上下载并安装了 Docker，就可以开始运行容器。Docker CLI 具有一个名为`docker run`的命令，专门用于启动和运行 Docker 容器。正如我们之前学到的，容器提供了与系统上运行的其他应用程序和进程隔离的功能。由于这个事实，Docker 容器的生命周期由容器内部运行的主要进程决定。当容器停止时，Docker 可能会尝试重新启动容器，以确保应用程序的连续性。

为了查看主机系统上正在运行的容器，我们还将利用`docker ps`命令。`docker ps`命令类似于 Unix 风格的`ps`命令，用于显示 Linux 或基于 Unix 的操作系统上运行的进程。

记住，当 Docker 首次运行容器时，如果它在本地缓存中没有存储容器镜像，它将从容器镜像注册表中下载容器镜像。要查看本地存储的容器镜像，使用`docker images`命令。

以下练习将演示如何使用`docker run`、`docker ps`和`docker images`命令来启动和查看简单的`hello-world`容器的状态。

## 练习 1.01：运行 hello-world 容器

一个简单的“Hello World”应用程序通常是开发人员在学习软件开发或开始新的编程语言时编写的第一行代码，容器化也不例外。Docker 发布了一个非常小且简单执行的`hello-world`容器。该容器演示了运行单个具有无限寿命的进程的容器的特性。

在这个练习中，你将使用`docker run`命令启动`hello-world`容器，并使用`docker ps`命令查看容器在执行完成后的状态。这将为你提供一个在本地开发环境中运行容器的基本概述。

1.  在 Bash 终端或 PowerShell 窗口中输入`docker run`命令。这会指示 Docker 运行一个名为`hello-world`的容器：

```
$ docker run hello-world
```

你的 shell 应该返回类似以下的输出：

```
Unable to find image 'hello-world: latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete 
Digest: sha256:
8e3114318a995a1ee497790535e7b88365222a21771ae7e53687ad76563e8e76
Status: Downloaded newer image for hello-world:latest
Hello from Docker!
This message shows that your installation appears to be working 
correctly.
To generate this message, Docker took the following steps:
 1\. The Docker client contacted the Docker daemon.
 2\. The Docker daemon pulled the "hello-world" image from the 
Docker Hub.
    (amd64)
 3\. The Docker daemon created a new container from that image 
which runs the executable that produces the output you are 
currently reading.
4\. The Docker daemon streamed that output to the Docker 
client, which sent it to your terminal.
To try something more ambitious, you can run an Ubuntu 
container with:
 $ docker run -it ubuntu bash
Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/
For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

刚刚发生了什么？你告诉 Docker 运行名为`hello-world`的容器。所以，首先，Docker 会在本地容器缓存中查找具有相同名称的容器。如果找不到，它将尝试在互联网上的容器注册表中查找以满足命令。通过简单地指定容器的名称，Docker 将默认查询 Docker Hub 以获取该名称的已发布容器镜像。

如你所见，它能够找到一个名为`library/hello-world`的容器，并开始逐层拉取容器镜像。在*第二章*《使用 Dockerfiles 入门》中，你将更深入地了解容器镜像和层。一旦镜像完全下载，Docker 运行该镜像，显示`Hello from Docker`输出。由于该镜像的主要进程只是显示该输出，容器在显示输出后停止并停止运行。

1.  使用`docker ps`命令查看系统上正在运行的容器。在 Bash 或 PowerShell 终端中，输入以下命令：

```
$ docker ps
```

这将返回类似以下的输出：

```
CONTAINER ID      IMAGE     COMMAND      CREATED
  STATUS              PORTS                   NAMES
```

`docker ps`命令的输出为空，因为它默认只显示当前正在运行的容器。这类似于 Linux/Unix 的`ps`命令，它只显示正在运行的进程。

1.  使用`docker ps -a`命令显示所有容器，甚至是已停止的容器：

```
$ docker ps -a
```

在返回的输出中，你应该看到`hello-world`容器实例：

```
CONTAINER ID     IMAGE           COMMAND     CREATED
  STATUS                          PORTS         NAMES
24c4ce56c904     hello-world     "/hello"    About a minute ago
  Exited (0) About a minute ago                 inspiring_moser
```

正如你所看到的，Docker 给容器分配了一个唯一的容器 ID。它还显示了运行的`IMAGE`，在该映像中执行的`COMMAND`，创建的`TIME`，以及运行该容器的进程的`STATUS`，以及一个唯一的可读名称。这个特定的容器大约一分钟前创建，执行了程序`/hello`，并成功运行。你可以看出程序运行并成功执行，因为它产生了一个`Exited (0)`的代码。

1.  你可以查询你的系统，看看 Docker 本地缓存了哪些容器映像。执行`docker images`命令来查看本地缓存：

```
$ docker images
```

返回的输出应该显示本地缓存的容器映像：

```
REPOSITORY     TAG        IMAGE ID        CREATED         SIZE
hello-world    latest     bf756fb1ae65    3 months ago    13.3kB
```

到目前为止，唯一缓存的映像是`hello-world`容器映像。这个映像正在运行`latest`版本，创建于 3 个月前，大小为 13.3 千字节。从前面的输出中，你知道这个 Docker 映像非常精简，开发者在 3 个月内没有发布过这个映像的代码更改。这个输出对于排除现实世界中软件版本之间的差异非常有帮助。

由于你只是告诉 Docker 运行`hello-world`容器而没有指定版本，Docker 将默认拉取最新版本。你可以通过在`docker run`命令中指定标签来指定不同的版本。例如，如果`hello-world`容器映像有一个版本`2.0`，你可以使用`docker run hello-world:2.0`命令运行该版本。

想象一下，如果容器比一个简单的`hello-world`应用程序复杂一些。想象一下，你的同事编写了一个软件，需要下载许多第三方库的非常特定的版本。如果你传统地运行这个应用程序，你将不得不下载他们开发语言的运行环境，以及所有的第三方库，以及详细的构建和执行他们的代码的说明。

然而，如果他们将他们的代码的 Docker 镜像发布到内部 Docker 注册表，他们只需要向您提供运行容器的`docker run`语法。由于您拥有 Docker，无论您的基础平台是什么，容器图像都将运行相同。容器图像本身已经包含了库和运行时的详细信息。

1.  如果您再次执行相同的`docker run`命令，那么对于用户输入的每个`docker run`命令，都将创建一个新的容器实例。值得注意的是，容器化的一个好处是能够轻松运行多个软件应用的实例。为了看到 Docker 如何处理多个容器实例，再次运行相同的`docker run`命令，以创建`hello-world`容器的另一个实例：

```
$ docker run hello-world
```

您应该看到以下输出：

```
Hello from Docker!
This message shows that your installation appears to be 
working correctly.
To generate this message, Docker took the following steps:
 1\. The Docker client contacted the Docker daemon.
 2\. The Docker daemon pulled the "hello-world" image from 
    the Docker Hub.
    (amd64)
 3\. The Docker daemon created a new container from that image 
    which runs the executable that produces the output you 
    are currently reading.
 4\. The Docker daemon streamed that output to the Docker client, 
    which sent it to your terminal.
To try something more ambitious, you can run an Ubuntu container 
with:
 $ docker run -it ubuntu bash
Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/
For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

请注意，这一次，Docker 不必再次从 Docker Hub 下载容器图像。这是因为您现在在本地缓存了该容器图像。相反，Docker 能够直接运行容器并将输出显示在屏幕上。让我们看看您的`docker ps -a`输出现在是什么样子。

1.  在您的终端中，再次运行`docker ps -a`命令：

```
docker ps -a
```

在输出中，您应该看到这个容器图像的第二个实例已经完成了执行并进入了停止状态，如输出的`STATUS`列中的`Exit (0)`所示：

```
CONTAINER ID     IMAGE           COMMAND       CREATED
  STATUS                      PORTS               NAMES
e86277ca07f1     hello-world     "/hello"      2 minutes ago
  Exited (0) 2 minutes ago                        awesome_euclid
24c4ce56c904     hello-world     "/hello"      20 minutes ago
  Exited (0) 20 minutes ago                       inspiring_moser
```

您现在在输出中看到了这个容器的第二个实例。每次执行`docker run`命令时，Docker 都会创建该容器的一个新实例，具有其属性和数据。您可以运行尽可能多的容器实例，只要您的系统资源允许。在这个例子中，您 20 分钟前创建了一个实例。您 2 分钟前创建了第二个实例。

1.  再次执行`docker images`命令，检查基本图像：

```
$ docker images
```

返回的输出将显示 Docker 从单个基本图像创建的两个运行实例：

```
REPOSITORY     TAG       IMAGE ID        CREATED         SIZE
hello-world    latest    bf756fb1ae65    3 months ago    13.3kB
```

在这个练习中，您使用`docker run`启动了`hello-world`容器。为了实现这一点，Docker 从 Docker Hub 注册表下载了图像，并在 Docker Engine 中执行了它。一旦基本图像被下载，您可以使用后续的`docker run`命令创建任意数量的该容器的实例。

Docker 容器管理比在开发环境中仅仅启动和查看容器状态更加复杂。Docker 还支持许多其他操作，这些操作有助于了解在 Docker 主机上运行的应用程序的状态。在接下来的部分中，我们将学习如何使用不同的命令来管理 Docker 容器。

# 管理 Docker 容器

在我们的容器之旅中，我们将经常从本地环境中拉取、启动、停止和删除容器。在将容器部署到生产环境之前，我们首先需要在本地运行容器，以了解其功能和正常行为。这包括启动容器、停止容器、获取有关容器运行方式的详细信息，当然还包括访问容器日志以查看容器内运行的应用程序的关键细节。这些基本命令如下所述：

+   `docker pull`：此命令将容器镜像下载到本地缓存

+   `docker stop`：此命令停止运行中的容器实例

+   `docker start`：此命令启动不再处于运行状态的容器实例

+   `docker restart`：此命令重新启动运行中的容器

+   `docker attach`：此命令允许用户访问（或附加）运行中的 Docker 容器实例的主要进程

+   `docker exec`：此命令在运行中的容器内执行命令

+   `docker rm`：此命令删除已停止的容器

+   `docker rmi`：此命令删除容器镜像

+   `docker inspect`：此命令显示有关容器状态的详细信息

容器生命周期管理是生产环境中有效容器管理的关键组成部分。在评估容器化基础设施的健康状况时，了解如何调查运行中的容器是至关重要的。

在接下来的练习中，我们将逐个使用这些命令，深入了解它们的工作原理以及如何利用它们来了解容器化基础设施的健康状况。

## 练习 1.02：管理容器生命周期

在开发和生产环境中管理容器时，了解容器实例的状态至关重要。许多开发人员使用包含特定基线配置的基础容器镜像，他们的应用程序可以在其上部署。Ubuntu 是一个常用的基础镜像，用户用它来打包他们的应用程序。

与完整的操作系统镜像不同，Ubuntu 基础容器镜像非常精简，故意省略了许多完整操作系统安装中的软件包。大多数基础镜像都有软件包系统，可以让您安装任何缺失的软件包。

请记住，在构建容器镜像时，您希望尽可能保持基础镜像的精简，只安装最必要的软件包。这样可以确保 Docker 主机可以快速拉取和启动容器镜像。

在这个练习中，您将使用官方的 Ubuntu 基础容器镜像。这个镜像将用于启动容器实例，用于测试各种容器生命周期管理命令，比如`docker pull`、`docker start`和`docker stop`。这个容器镜像很有用，因为默认的基础镜像允许我们在长时间运行的会话中运行容器实例，以了解容器生命周期管理命令的功能。在这个练习中，您还将拉取`Ubuntu 18.04`容器镜像，并将其与`Ubuntu 19.04`容器镜像进行比较：

1.  在新的终端或 PowerShell 窗口中，执行`docker pull`命令以下载`Ubuntu 18.04`容器镜像：

```
$ docker pull ubuntu:18.04
```

您应该看到以下输出，表明 Docker 正在下载基础镜像的所有层：

```
5bed26d33875: Pull complete 
f11b29a9c730: Pull complete 
930bda195c84: Pull complete 
78bf9a5ad49e: Pull complete 
Digest: sha256:bec5a2727be7fff3d308193cfde3491f8fba1a2ba392
        b7546b43a051853a341d
Status: Downloaded newer image for ubuntu:18.04
docker.io/library/ubuntu:18.04
```

1.  使用`docker pull`命令下载`Ubuntu 19.04`基础镜像：

```
$ docker pull ubuntu:19.04
```

当 Docker 下载`Ubuntu 19.04`基础镜像时，您将看到类似的输出：

```
19.04: Pulling from library/ubuntu
4dc9c2fff018: Pull complete 
0a4ccbb24215: Pull complete 
c0f243bc6706: Pull complete 
5ff1eaecba77: Pull complete 
Digest: sha256:2adeae829bf27a3399a0e7db8ae38d5adb89bcaf1bbef
        378240bc0e6724e8344
Status: Downloaded newer image for ubuntu:19.04
docker.io/library/ubuntu:19.04
```

1.  使用`docker images`命令确认容器镜像已下载到本地容器缓存：

```
$ docker images
```

本地容器缓存的内容将显示`Ubuntu 18.04`和`Ubuntu 19.04`基础镜像，以及我们之前练习中的`hello-world`镜像：

```
REPOSITORY     TAG        IMAGE ID         CREATED         SIZE
ubuntu         18.04      4e5021d210f6     4 weeks ago     64.2MB
ubuntu         19.04      c88ac1f841b7     3 months ago    70MB
hello-world    latest     bf756fb1ae65     3 months ago    13.3kB
```

1.  在运行这些镜像之前，使用`docker inspect`命令获取关于容器镜像的详细输出以及它们之间的差异。在你的终端中，运行`docker inspect`命令，并使用`Ubuntu 18.04`容器镜像的 ID 作为主要参数：

```
$ docker inspect 4e5021d210f6
```

`inspect`输出将包含定义该容器的所有属性的大型列表。例如，你可以看到容器中配置了哪些环境变量，容器在最后一次更新镜像时是否设置了主机名，以及定义该容器的所有层的详细信息。这个输出包含了在规划升级时可能会有价值的关键调试细节。以下是`inspect`命令的截断输出。在`Ubuntu 18.04`镜像中，`"Created"`参数应该提供构建容器镜像的日期和时间：

```
"Id": "4e5021d210f6d4a0717f4b643409eff23a4dc01c4140fa378b1b
       f0a4f8f4",
"Created": "2020-03-20T19:20:22.835345724Z",
"Path": "/bin/bash",
"Args": [],
```

1.  检查`Ubuntu 19.04`容器，你会看到这个参数是不同的。在`Ubuntu 19.04`容器镜像 ID 中运行`docker inspect`命令：

```
$ docker inspect c88ac1f841b7
```

在显示的输出中，你会看到这个容器镜像是在一个不同的日期创建的，与`18.04`容器镜像不同：

```
"Id": "c88ac1f841b74e5021d210f6d4a0717f4b643409eff23a4dc0
       1c4140fa"
"Created": "2020-01-16T01:20:46.938732934Z",
"Path": "/bin/bash",
"Args": []
```

如果你知道 Ubuntu 基础镜像中可能存在安全漏洞，这可能是至关重要的信息。这些信息也可以对帮助你确定要运行哪个版本的容器至关重要。

1.  在检查了两个容器镜像之后，很明显你最好的选择是坚持使用 Ubuntu 长期支持版 18.04 版本。正如你从前面的输出中看到的，18.04 版本比 19.04 版本更加更新。这是可以预期的，因为 Ubuntu 通常会为长期支持版本提供更稳定的更新。

1.  使用`docker run`命令启动 Ubuntu 18.04 容器的一个实例：

```
$ docker run -d ubuntu:18.04
```

请注意，这次我们使用了带有`-d`标志的`docker run`命令。这告诉 Docker 以守护进程模式（或后台模式）运行容器。如果我们省略`-d`标志，容器将占用我们当前的终端，直到容器内的主要进程终止。

注意

成功调用`docker run`命令通常只会返回容器 ID 作为输出。某些版本的 Docker 不会返回任何输出。

1.  使用`docker ps -a`命令检查容器的状态：

```
$ docker ps -a
```

这将显示类似于以下内容的输出：

```
CONTAINER ID     IMAGE           COMMAND        CREATED
  STATUS                     PORTS         NAMES
c139e44193de     ubuntu:18.04    "/bin/bash"    6 seconds ago
  Exited (0) 4 seconds ago                 xenodochial_banzai
```

正如你所看到的，你的容器已经停止并退出。这是因为容器内的主要进程是`/bin/bash`，这是一个 shell。Bash shell 不能在没有以交互模式执行的情况下运行，因为它期望来自用户的文本输入和输出。

1.  再次运行`docker run`命令，传入`-i`标志以使会话交互（期望用户输入），并传入`-t`标志以为容器分配一个**伪 tty**处理程序。`伪 tty`处理程序将基本上将用户的终端链接到容器内运行的交互式 Bash shell。这将允许 Bash 正确运行，因为它将指示容器以交互模式运行，期望用户输入。您还可以通过传入`--name`标志为容器指定一个易读的名称。在您的 Bash 终端中键入以下命令：

```
$ docker run -i -t -d --name ubuntu1 ubuntu:18.04
```

1.  再次执行`docker ps -a`命令以检查容器实例的状态：

```
$ docker ps -a 
```

您现在应该看到新的实例正在运行，以及刚刚无法启动的实例：

```
CONTAINER ID    IMAGE          COMMAND         CREATED
  STATUS            PORTS               NAMES
f087d0d92110    ubuntu:18.04   "/bin/bash"     4 seconds ago
  Up 2 seconds                          ubuntu1
c139e44193de    ubuntu:18.04   "/bin/bash"     5 minutes ago
  Exited (0) 5 minutes ago              xenodochial_banzai
```

1.  您现在有一个正在运行的 Ubuntu 容器。您可以使用`docker exec`命令在此容器内运行命令。运行`exec`命令以访问 Bash shell，这将允许我们在容器内运行命令。类似于`docker run`，传入`-i`和`-t`标志使其成为交互式会话。还传入容器的名称或 ID，以便 Docker 知道您要定位哪个容器。`docker exec`的最后一个参数始终是您希望执行的命令。在这种情况下，它将是`/bin/bash`，以在容器实例内启动 Bash shell：

```
docker exec -it ubuntu1 /bin/bash
```

您应该立即看到您的提示更改为根 shell。这表明您已成功在 Ubuntu 容器内启动了一个 shell。容器的主机名`cfaa37795a7b`取自容器 ID 的前 12 个字符。这使用户可以确定他们正在访问哪个容器，如下例所示：

```
root@cfaa37795a7b:/#
```

1.  在容器内，您所拥有的工具非常有限。与 VM 镜像不同，容器镜像在预安装的软件包方面非常精简。但是`echo`命令应该是可用的。使用`echo`将一个简单的消息写入文本文件：

```
root@cfaa37795a7b:/# echo "Hello world from ubuntu1" > hello-world.txt
```

1.  运行`exit`命令退出`ubuntu1`容器的 Bash shell。您应该返回到正常的终端 shell：

```
root@cfaa37795a7b:/# exit
```

该命令将返回以下输出。请注意，对于运行该命令的每个用户，输出可能会有所不同：

```
user@developmentMachine:~/
```

1.  现在创建一个名为`ubuntu2`的第二个容器，它也将在您的 Docker 环境中使用`Ubuntu 19.04`镜像运行：

```
$ docker run -i -t -d --name ubuntu2 ubuntu:19.04
```

1.  运行`docker exec`来访问第二个容器的 shell。记得使用你创建的新容器的名称或容器 ID。同样地，访问这个容器内部的 Bash shell，所以最后一个参数将是`/bin/bash`：

```
$ docker exec -it ubuntu2 /bin/bash
```

你应该观察到你的提示会变成一个 Bash root shell，类似于`Ubuntu 18.04`容器镜像的情况：

```
root@875cad5c4dd8:/#
```

1.  在`ubuntu2`容器实例内部运行`echo`命令，写入类似的`hello-world`类型的问候语：

```
root@875cad5c4dd8:/# echo "Hello-world from ubuntu2!" > hello-world.txt
```

1.  目前，在你的 Docker 环境中有两个运行中的 Ubuntu 容器实例，根账户的主目录中有两个单独的`hello-world`问候消息。使用`docker ps`来查看这两个运行中的容器镜像：

```
$ docker ps
```

运行容器的列表应该反映出两个 Ubuntu 容器，以及它们创建后经过的时间：

```
CONTAINER ID    IMAGE            COMMAND        CREATED
  STATUS              PORTS               NAMES
875cad5c4dd8    ubuntu:19.04     "/bin/bash"    3 minutes ago
  Up 3 minutes                            ubuntu2
cfaa37795a7b    ubuntu:18.04     "/bin/bash"    15 minutes ago
  Up 15 minutes                           ubuntu1
```

1.  不要使用`docker exec`来访问容器内部的 shell，而是使用它来显示你通过在容器内执行`cat`命令写入的`hello-world.txt`文件的输出：

```
$ docker exec -it ubuntu1 cat hello-world.txt
```

输出将显示你在之前步骤中传递给容器的`hello-world`消息。请注意，一旦`cat`命令完成并显示输出，用户就会被移回到主终端的上下文中。这是因为`docker exec`会话只会存在于用户执行命令的时间内。

在之前的 Bash shell 示例中，只有用户使用`exit`命令终止它时，Bash 才会退出。在这个例子中，只显示了`Hello world`输出，因为`cat`命令显示了输出并退出，结束了`docker exec`会话：

```
Hello world from ubuntu1
```

你会看到`hello-world`文件的内容显示，然后返回到你的主终端会话。

1.  在`ubuntu2`容器实例中运行相同的`cat`命令：

```
$ docker exec -it ubuntu2 cat hello-world.txt
```

与第一个例子类似，`ubuntu2`容器实例将显示之前提供的`hello-world.txt`文件的内容：

```
Hello-world from ubuntu2!
```

正如你所看到的，Docker 能够在两个容器上分配一个交互式会话，执行命令，并直接返回输出到我们正在运行的容器实例中。

1.  与你用来在运行中的容器内执行命令的方式类似，你也可以停止、启动和重新启动它们。使用`docker stop`命令停止其中一个容器实例。在你的终端会话中，执行`docker stop`命令，然后是`ubuntu2`容器的名称或容器 ID：

```
$ docker stop ubuntu2
```

该命令应该不返回任何输出。

1.  使用`docker ps`命令查看所有正在运行的容器实例：

```
$ docker ps
```

输出将显示`ubuntu1`容器正在运行：

```
CONTAINER ID    IMAGE           COMMAND        CREATED
  STATUS              PORTS               NAMES
cfaa37795a7b    ubuntu:18.04    "/bin/bash"    26 minutes ago
  Up 26 minutes                           ubuntu1
```

1.  执行`docker ps -a`命令以查看所有容器实例，无论它们是否正在运行，以查看您的容器是否处于停止状态：

```
$ docker ps -a
```

该命令将返回以下输出：

```
CONTAINER ID     IMAGE            COMMAND         CREATED
  STATUS                      PORTS             NAMES
875cad5c4dd8     ubuntu:19.04     "/bin/bash"     14 minutes ago
  Exited (0) 6 seconds ago                      ubuntu2
```

1.  使用`docker start`或`docker restart`命令重新启动容器实例：

```
$ docker start ubuntu2
```

该命令将不返回任何输出，尽管某些版本的 Docker 可能会显示容器 ID。

1.  使用`docker ps`命令验证容器是否再次运行：

```
$ docker ps
```

注意`STATUS`显示该容器只运行了很短的时间（`1 秒`），尽管容器实例是 29 分钟前创建的：

```
CONTAINER ID    IMAGE           COMMAND         CREATED
  STATUS              PORTS               NAMES
875cad5c4dd8    ubuntu:19.04    "/bin/bash"     17 minutes ago
  Up 1 second                             ubuntu2
cfaa37795a7b    ubuntu:18.04    "/bin/bash"     29 minutes ago
  Up 29 minutes                           ubuntu1
```

从这个状态开始，您可以尝试启动、停止或在这些容器内执行命令。

1.  容器管理生命周期的最后阶段是清理您创建的容器实例。使用`docker stop`命令停止`ubuntu1`容器实例：

```
$ docker stop ubuntu1
```

该命令将不返回任何输出，尽管某些版本的 Docker 可能会返回容器 ID。

1.  执行相同的`docker stop`命令以停止`ubuntu2`容器实例：

```
$ docker stop ubuntu2
```

1.  当容器实例处于停止状态时，使用`docker rm`命令彻底删除容器实例。使用`docker rm`后跟名称或容器 ID 删除`ubuntu1`容器实例：

```
$ docker rm ubuntu1
```

该命令将不返回任何输出，尽管某些版本的 Docker 可能会返回容器 ID。

在`ubuntu2`容器实例上执行相同的步骤：

```
$ docker rm ubuntu2
```

1.  执行`docker ps -a`以查看所有容器，即使它们处于停止状态。您会发现停止的容器由于之前的命令已被删除。您也可以删除`hello-world`容器实例。使用从`docker ps -a`输出中捕获的容器 ID 删除`hello-world`容器：

```
$ docker rm b291785f066c
```

1.  要完全重置我们的 Docker 环境状态，请删除您在此练习中下载的基本图像。使用`docker images`命令查看缓存的基本图像：

```
$ docker images
```

您的本地缓存中将显示 Docker 图像列表和所有关联的元数据：

```
REPOSITORY     TAG        IMAGE ID        CREATED         SIZE
ubuntu         18.04      4e5021d210f6    4 weeks ago     64.2MB
ubuntu         19.04      c88ac1f841b7    3 months ago    70MB
hello-world    latest     bf756fb1ae65    3 months ago    13.3kB
```

1.  执行`docker rmi`命令，后跟图像 ID 以删除第一个图像 ID：

```
$ docker rmi 4e5021d210f6
```

类似于`docker pull`，`rmi`命令将删除每个图像和所有关联的层：

```
Untagged: ubuntu:18.04
Untagged: ubuntu@sha256:bec5a2727be7fff3d308193cfde3491f8fba1a2b
a392b7546b43a051853a341d
Deleted: sha256:4e5021d210f65ebe915670c7089120120bc0a303b9020859
2851708c1b8c04bd
Deleted: sha256:1d9112746e9d86157c23e426ce87cc2d7bced0ba2ec8ddbd
fbcc3093e0769472
Deleted: sha256:efcf4a93c18b5d01aa8e10a2e3b7e2b2eef0378336456d86
53e2d123d6232c1e
Deleted: sha256:1e1aa31289fdca521c403edd6b37317bf0a349a941c7f19b
6d9d311f59347502
Deleted: sha256:c8be1b8f4d60d99c281fc2db75e0f56df42a83ad2f0b0916
21ce19357e19d853
```

对于要删除的每个映像，执行此步骤，替换各种映像 ID。对于删除的每个基本映像，您将看到所有图像层都被取消标记并与其一起删除。

定期清理 Docker 环境很重要，因为频繁构建和运行容器会导致长时间大量的硬盘使用。现在您已经知道如何在本地开发环境中运行和管理 Docker 容器，可以使用更高级的 Docker 命令来了解容器的主要进程功能以及如何解决问题。在下一节中，我们将看看`docker attach`命令，直接访问容器的主要进程。

注意

为了简化清理环境的过程，Docker 提供了一个`prune`命令，将自动删除旧的容器和基本映像：

`$ docker system prune -fa`

执行此命令将删除任何未绑定到现有运行容器的容器映像，以及 Docker 环境中的任何其他资源。

# 使用 attach 命令附加到容器

在上一个练习中，您看到了如何使用`docker exec`命令在运行的容器实例中启动新的 shell 会话以执行命令。`docker exec`命令非常适合快速访问容器化实例以进行调试、故障排除和了解容器运行的上下文。

但是，正如本章前面所述，Docker 容器按照容器内部运行的主要进程的生命周期运行。当此进程退出时，容器将停止。如果要直接访问容器内部的主要进程（而不是次要的 shell 会话），那么 Docker 提供了`docker attach`命令来附加到容器内部正在运行的主要进程。

使用`docker attach`时，您可以访问容器中运行的主要进程。如果此进程是交互式的，例如 Bash 或 Bourne shell 会话，您将能够通过`docker attach`会话直接执行命令（类似于`docker exec`）。但是，如果容器中的主要进程终止，整个容器实例也将终止，因为 Docker 容器的生命周期取决于主要进程的运行状态。

在接下来的练习中，您将使用`docker attach`命令直接访问 Ubuntu 容器的主要进程。默认情况下，此容器的主要进程是`/bin/bash`。

## 练习 1.03：附加到 Ubuntu 容器

`docker attach`命令用于在主要进程的上下文中附加到运行中的容器。在此练习中，您将使用`docker attach`命令附加到运行中的容器并直接调查主容器`entrypoint`进程：

1.  使用`docker run`命令启动一个新的 Ubuntu 容器实例。以交互模式（`-i`）运行此容器，分配一个 TTY 会话（`-t`），并在后台（`-d`）运行。将此容器命名为`attach-example1`：

```
docker run -itd --name attach-example1 ubuntu:latest
```

这将使用 Ubuntu 容器图像的最新版本启动一个名为`attach-example1`的新 Ubuntu 容器实例。

1.  使用`docker ps`命令来检查该容器是否在我们的环境中运行：

```
docker ps 
```

将显示运行中容器实例的详细信息。请注意，此容器的主要进程是 Bash shell（`/bin/bash`）：

```
CONTAINER ID    IMAGE            COMMAND          CREATED
  STATUS              PORTS               NAMES
90722712ae93    ubuntu:latest    "/bin/bash"      18 seconds ago
  Up 16 seconds                           attach-example1
```

1.  运行`docker attach`命令以附加到此容器内部的主要进程（`/bin/bash`）。使用`docker attach`后跟容器实例的名称或 ID：

```
$ docker attach attach-example1
```

这应该将您放入此容器实例的主 Bash shell 会话中。请注意，您的终端会话应更改为根 shell 会话，表示您已成功访问了容器实例：

```
root@90722712ae93:/#
```

在这里需要注意，使用诸如`exit`之类的命令来终止 shell 会话将导致停止容器实例，因为您现在已连接到容器实例的主要进程。默认情况下，Docker 提供了*Ctrl* + *P*然后*Ctrl* + *Q*的快捷键序列，以正常分离`attach`会话。

1.  使用键盘组合*Ctrl* + *P*然后*Ctrl* + *Q*正常分离此会话：

```
root@90722712ae93:/# CTRL-p CTRL-q
```

注意

您不会输入`CTRL-p CTRL-q`这些单词；相反，您将按住*Ctrl*键，按下*P*键，然后释放两个键。然后，再次按住*Ctrl*键，按下*Q*键，然后再次释放两个键。

成功分离容器后，将显示单词`read escape sequence`，然后将您返回到主终端或 PowerShell 会话：

```
root@90722712ae93:/# read escape sequence
```

1.  使用`docker ps`验证 Ubuntu 容器是否仍然按预期运行：

```
$ docker ps
```

`attach-example1`容器将被显示为预期运行：

```
CONTAINER ID    IMAGE            COMMAND          CREATED
  STATUS              PORTS               NAMES
90722712ae93    ubuntu:latest    "/bin/bash"      13 minutes ago
  Up 13 minutes                           attach-example1
```

1.  使用`docker attach`命令再次附加到`attach-example1`容器实例：

```
$ docker attach attach-example1
```

您应该被放回到主进程的 Bash 会话中：

```
root@90722712ae93:/#
```

1.  现在，使用`exit`命令终止这个容器的主进程。在 Bash shell 会话中，输入`exit`命令：

```
root@90722712ae93:/# exit
```

终端会话应该已经退出，再次返回到您的主终端。

1.  使用`docker ps`命令观察`attach-example1`容器不再运行：

```
$ docker ps
```

这应该不会显示任何正在运行的容器实例：

```
CONTAINER ID    IMAGE            COMMAND              CREATED
  STATUS              PORTS               NAMES
```

1.  使用`docker ps -a`命令查看所有容器，即使已停止或已退出的容器也会显示：

```
$ docker ps -a
```

这应该显示`attach-example1`容器处于停止状态：

```
CONTAINER ID      IMAGE                COMMAND 
  CREATED            STATUS    PORTS           NAMES
90722712ae93      ubuntu:latest        "/bin/bash"
  20 minutes ago     Exited (0) 3 minutes ago  attach-example1
```

正如你所看到的，容器已经优雅地终止（`Exited (0)`）大约 3 分钟前。`exit`命令会优雅地终止 Bash shell 会话。

1.  使用`docker system prune -fa`命令清理已停止的容器实例：

```
docker system prune -fa
```

这应该删除所有已停止的容器实例，包括`attach-example1`容器实例，如下面的输出所示：

```
Deleted Containers:
ry6v87v9a545hjn7535jk2kv9x8cv09wnkjnscas98v7a762nvnw7938798vnand
Deleted Images:
untagged: attach-example1
```

在这个练习中，我们使用`docker attach`命令直接访问正在运行的容器的主进程。这与我们在本章中早些时候探讨的`docker exec`命令不同，因为`docker exec`在运行的容器内执行一个新的进程，而`docker attach`直接附加到容器的主进程。然而，在附加到容器时，必须注意不要通过终止主进程来停止容器。

在下一个活动中，我们将整合本章中涵盖的 Docker 管理命令，开始组装成全景徒步旅行微服务应用程序堆栈的构建块容器。

## 活动 1.01：从 Docker Hub 拉取并运行 PostgreSQL 容器镜像

全景徒步旅行是我们将在本书中构建的多层 Web 应用程序。与任何 Web 应用程序类似，它将包括一个 Web 服务器容器（NGINX）、一个 Python Django 后端应用程序和一个 PostgreSQL 数据库。在部署 Web 应用程序或前端 Web 服务器之前，您必须先部署后端数据库。

在这个活动中，您被要求使用默认凭据启动一个 PostgreSQL 版本 12 的数据库容器。

注意

官方的 Postgres 容器映像提供了许多环境变量覆盖，您可以利用这些变量来配置 PostgreSQL 实例。在 Docker Hub 上查看有关容器的文档[`hub.docker.com/_/postgres`](https://hub.docker.com/_/postgres)。

执行以下步骤：

1.  创建一个 Postgres 数据库容器实例，将作为我们应用程序堆栈的数据层。

1.  使用环境变量在运行时配置容器以使用以下数据库凭据：

```
username: panoramic
password: trekking
```

1.  验证容器是否正在运行和健康。

**预期输出：**

运行`docker ps`命令应返回以下输出：

```
CONTAINER ID  IMAGE         COMMAND                 CREATED
  STATUS              PORTS               NAMES
29f115af8cdd  postgres:12   "docker-entrypoint.s…"  4 seconds ago
  Up 2 seconds        5432/tcp            blissful_kapitsa
```

注意

此活动的解决方案可以通过此链接找到。

在下一个活动中，您将访问刚刚在容器实例中设置的数据库。您还将与容器交互，以获取容器中运行的数据库列表。

## 活动 1.02：访问全景徒步应用程序数据库

本活动将涉及使用`PSQL` CLI 实用程序访问在容器实例内运行的数据库。一旦您使用凭据（`panoramic/trekking`）登录，您将查询容器中运行的数据库列表。

执行以下步骤：

1.  使用 PSQL 命令行实用程序登录到 Postgres 数据库容器。

1.  登录到数据库后，默认情况下返回 Postgres 中的数据库列表。

注意

如果您对 PSQL CLI 不熟悉，以下是一些参考命令的列表，以帮助您完成此活动：

登录：`psql --username username --password`

列出数据库：`\l`

退出 PSQL shell：`\q`

**预期输出：**

![图 1.3：活动 1.02 的预期输出](img/B15021_01_03.jpg)

图 1.3：活动 1.02 的预期输出

注意

此活动的解决方案可以通过此链接找到。

# 摘要

在本章中，您学习了容器化的基础知识，以及在容器中运行应用程序的好处，以及管理容器化实例的基本 Docker 生命周期命令。您了解到容器作为一个真正可以构建一次并在任何地方运行的通用软件部署包。因为我们在本地运行 Docker，我们可以确信在我们的本地环境中运行的相同容器映像可以部署到生产环境并且可以放心地运行。

通过诸如`docker run`、`docker start`、`docker exec`、`docker ps`和`docker stop`之类的命令，我们通过 Docker CLI 探索了容器生命周期管理的基础知识。通过各种练习，我们从相同的基础映像启动了容器实例，使用`docker exec`对其进行了配置，并使用其他基本的容器生命周期命令（如`docker rm`和`docker rmi`）清理了部署。

在本章的最后部分，我们毅然决然地迈出了第一步，通过启动一个 PostgreSQL 数据库容器实例，开始运行我们的全景徒步应用程序。我们在`docker run`命令中使用环境变量创建了一个配置了默认用户名和密码的实例。我们通过在容器内部执行 PSQL 命令行工具并查询数据库来测试配置，以查看模式。

虽然这只是触及 Docker 能力表面的一部分，但我们希望它能激发你对即将在后续章节中涵盖的内容的兴趣。在下一章中，我们将讨论使用`Dockerfiles`和`docker build`命令构建真正不可变的容器。编写自定义的`Dockerfiles`来构建和部署独特的容器映像将展示在规模上运行容器化应用程序的强大能力。

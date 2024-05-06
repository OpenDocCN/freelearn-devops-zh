# 第二章：介绍 Docker

我们将讨论现代持续交付过程应该如何看待，引入 Docker，这种改变了 IT 行业和服务器使用方式的技术。

本章涵盖以下内容：

+   介绍虚拟化和容器化的概念

+   在不同的本地和服务器环境中安装 Docker

+   解释 Docker 工具包的架构

+   使用 Dockerfile 构建 Docker 镜像并提交更改

+   将应用程序作为 Docker 容器运行

+   配置 Docker 网络和端口转发

+   介绍 Docker 卷作为共享存储

# 什么是 Docker？

Docker 是一个旨在通过软件容器帮助应用程序部署的开源项目。这句话来自官方 Docker 页面：

“Docker 容器将软件包装在一个完整的文件系统中，其中包含运行所需的一切：代码、运行时、系统工具、系统库 - 任何可以安装在服务器上的东西。这保证软件无论在什么环境下都能始终运行相同。”

因此，Docker 与虚拟化类似，允许将应用程序打包成可以在任何地方运行的镜像。

# 容器化与虚拟化

没有 Docker，可以使用硬件虚拟化来实现隔离和其他好处，通常称为虚拟机。最流行的解决方案是 VirtualBox、VMware 和 Parallels。虚拟机模拟计算机架构，并提供物理计算机的功能。如果每个应用程序都作为单独的虚拟机镜像交付和运行，我们可以实现应用程序的完全隔离。以下图展示了虚拟化的概念：

![](img/020d670e-74d0-41e8-af9d-8598b12046c2.png)

每个应用程序都作为一个单独的镜像启动，具有所有依赖项和一个客户操作系统。镜像由模拟物理计算机架构的 hypervisor 运行。这种部署方法得到许多工具（如 Vagrant）的广泛支持，并专门用于开发和测试环境。然而，虚拟化有三个重大缺点：

+   **性能低下**：虚拟机模拟整个计算机架构来运行客户操作系统，因此每个操作都会带来显着的开销。

+   **高资源消耗**：模拟需要大量资源，并且必须针对每个应用程序单独进行。这就是为什么在标准桌面机器上只能同时运行几个应用程序。

+   **大的镜像大小**：每个应用程序都随着完整的操作系统交付，因此在服务器上部署意味着发送和存储大量数据。

容器化的概念提出了一个不同的解决方案：

![](img/41c0db73-c10e-42f2-8fec-f45e28ef52f4.png)

每个应用程序都与其依赖项一起交付，但没有操作系统。应用程序直接与主机操作系统接口，因此没有额外的客户操作系统层。这导致更好的性能和没有资源浪费。此外，交付的 Docker 镜像大小明显更小。

请注意，在容器化的情况下，隔离发生在主机操作系统进程的级别。然而，这并不意味着容器共享它们的依赖关系。它们每个都有自己的正确版本的库，如果其中任何一个被更新，它对其他容器没有影响。为了实现这一点，Docker 引擎为容器创建了一组 Linux 命名空间和控制组。这就是为什么 Docker 安全性基于 Linux 内核进程隔离。尽管这个解决方案已经足够成熟，但与虚拟机提供的完整操作系统级隔离相比，它可能被认为略微不够安全。

# Docker 的需求

Docker 容器化解决了传统软件交付中出现的许多问题。让我们仔细看看。

# 环境

安装和运行软件是复杂的。您需要决定操作系统、资源、库、服务、权限、其他软件以及您的应用程序所依赖的一切。然后，您需要知道如何安装它。而且，可能会有一些冲突的依赖关系。那么你该怎么办？如果您的软件需要升级一个库，但其他软件不需要呢？在一些公司中，这些问题是通过拥有**应用程序类别**来解决的，每个类别由专用服务器提供服务，例如，一个用于具有 Java 7 的 Web 服务的服务器，另一个用于具有 Java 8 的批处理作业，依此类推。然而，这种解决方案在资源方面不够平衡，并且需要一支 IT 运维团队来照顾所有的生产和测试服务器。

环境复杂性的另一个问题是，通常需要专家来运行应用程序。一个不太懂技术的人可能会很难设置 MySQL、ODBC 或任何其他稍微复杂的工具。对于不作为特定操作系统二进制文件交付但需要源代码编译或任何其他特定环境配置的应用程序来说，这一点尤为真实。

# 隔离

保持工作区整洁。一个应用程序可能会改变另一个应用程序的行为。想象一下会发生什么。应用程序共享一个文件系统，因此如果应用程序 A 将某些内容写入错误的目录，应用程序 B 将读取不正确的数据。它们共享资源，因此如果应用程序 A 存在内存泄漏，它不仅会冻结自身，还会冻结应用程序 B。它们共享网络接口，因此如果应用程序 A 和 B 都使用端口`8080`，其中一个将崩溃。隔离也涉及安全方面。运行有错误的应用程序或恶意软件可能会对其他应用程序造成损害。这就是为什么将每个应用程序保持在单独的沙盒中是一种更安全的方法，它限制了损害范围仅限于应用程序本身。

# 组织应用程序

服务器通常会因为有大量运行的应用程序而变得混乱，而没有人知道这些应用程序是什么。你将如何检查服务器上运行的应用程序以及它们各自使用的依赖关系？它们可能依赖于库、其他应用程序或工具。如果没有详尽的文档，我们所能做的就是查看运行的进程并开始猜测。Docker 通过将每个应用程序作为一个单独的容器来保持组织，这些容器可以列出、搜索和监视。

# 可移植性

“一次编写，到处运行”，这是 Java 最早版本的广告口号。的确，Java 解决了可移植性问题；然而，我仍然可以想到一些它失败的情况，例如不兼容的本地依赖项或较旧版本的 Java 运行时。此外，并非所有软件都是用 Java 编写的。

Docker 将可移植性的概念提升了一个层次；如果 Docker 版本兼容，那么所提供的软件将在编程语言、操作系统或环境配置方面都能正确运行。因此，Docker 可以用“不仅仅是代码，而是整个环境”来表达。

# 小猫和牛

传统软件部署和基于 Docker 的部署之间的区别通常用小猫和牛的类比来表达。每个人都喜欢小猫。小猫是独一无二的。每只小猫都有自己的名字，需要特殊对待。小猫是用情感对待的。它们死了我们会哭。相反，牛只存在来满足我们的需求。即使牛的形式是单数，因为它只是一群一起对待的动物。没有命名，没有独特性。当然，它们是独一无二的（就像每个服务器都是独一无二的），但这是无关紧要的。

这就是为什么对 Docker 背后的理念最直接的解释是<q>把你的服务器当作牛，而不是宠物。</q>

# 替代的容器化技术

Docker 并不是市场上唯一的容器化系统。实际上，Docker 的最初版本是基于开源的**LXC**（**Linux Containers**）系统的，这是一个容器的替代平台。其他已知的解决方案包括 FreeBSD Jails、OpenVZ 和 Solaris Containers。然而，Docker 因其简单性、良好的营销和创业方法而超越了所有其他系统。它适用于大多数操作系统，允许您在不到 15 分钟内做一些有用的事情，具有许多易于使用的功能，良好的教程，一个伟大的社区，可能是 IT 行业中最好的标志。

# Docker 安装

Docker 的安装过程快速简单。目前，它在大多数 Linux 操作系统上得到支持，并提供了专门的二进制文件。Mac 和 Windows 也有很好的本地应用支持。然而，重要的是要理解，Docker 在内部基于 Linux 内核及其特定性，这就是为什么在 Mac 和 Windows 的情况下，它使用虚拟机（Mac 的 xhyve 和 Windows 的 Hyper-V）来运行 Docker 引擎环境。

# Docker 的先决条件

Docker 的要求针对每个操作系统都是特定的。

**Mac**：

+   2010 年或更新型号，具有英特尔对**内存管理单元**（**MMU**）虚拟化的硬件支持

+   macOS 10.10.3 Yosemite 或更新版本

+   至少 4GB 的 RAM

+   未安装早于 4.3.30 版本的 VirtualBox

**Windows**：

+   64 位 Windows 10 专业版

+   启用了 Hyper-V 包

**Linux**：

+   64 位架构

+   Linux 内核 3.10 或更高版本

如果您的机器不符合要求，那么解决方案是使用安装了 Ubuntu 操作系统的 VirtualBox。尽管这种解决方法听起来有些复杂，但并不一定是最糟糕的方法，特别是考虑到在 Mac 和 Windows 的情况下 Docker 引擎环境本身就是虚拟化的。此外，Ubuntu 是使用 Docker 的最受支持的系统之一。

本书中的所有示例都在 Ubuntu 16.04 操作系统上进行了测试。

# 在本地机器上安装

Dockers 的安装过程非常简单，并且在其官方页面上有很好的描述。

# Ubuntu 的 Docker

[`docs.docker.com/engine/installation/linux/ubuntulinux/`](https://docs.docker.com/engine/installation/linux/ubuntulinux/) 包含了在 Ubuntu 机器上安装 Docker 的指南。

在 Ubuntu 16.04 的情况下，我执行了以下命令：

```
$ sudo apt-get update
$ sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 9DC858229FC7DD38854AE2D88D81803C0EBFCD88
$ sudo apt-add-repository 'deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial main stable'
$ sudo apt-get update
$ sudo apt-get install -y docker-ce
```

所有操作完成后，Docker 应该已安装。然而，目前唯一被允许使用 Docker 命令的用户是`root`。这意味着每个 Docker 命令前都必须加上`sudo`关键字。

我们可以通过将他们添加到`docker`组来使其他用户使用 Docker：

```
$ sudo usermod -aG docker <username>
```

成功注销后，一切都设置好了。然而，通过最新的命令，我们需要采取一些预防措施，以免将 Docker 权限赋予不需要的用户，从而在 Docker 引擎中创建漏洞。这在服务器机器上安装时尤为重要。

# Linux 的 Docker

[`docs.docker.com/engine/installation/linux/`](https://docs.docker.com/engine/installation/linux/) 包含了大多数 Linux 发行版的安装指南。

# Mac 的 Docker

[`docs.docker.com/docker-for-mac/`](https://docs.docker.com/docker-for-mac/) 包含了在 Mac 机器上安装 Docker 的逐步指南。它与一系列 Docker 组件一起提供：

+   带有 Docker 引擎的虚拟机

+   Docker Machine（用于在虚拟机上创建 Docker 主机的工具）

+   Docker Compose

+   Docker 客户端和服务器

+   Kitematic：一个 GUI 应用程序

Docker Machine 工具有助于在 Mac、Windows、公司网络、数据中心以及 AWS 或 Digital Ocean 等云提供商上安装和管理 Docker 引擎。

# Windows 的 Docker

[`docs.docker.com/docker-for-windows/`](https://docs.docker.com/docker-for-windows/) 包含了如何在 Windows 机器上安装 Docker 的逐步指南。它与一组类似于 Mac 的 Docker 组件一起提供。

所有支持的操作系统和云平台的安装指南都可以在官方 Docker 页面上找到，[`docs.docker.com/engine/installation/`](https://docs.docker.com/engine/installation/)。

# 测试 Docker 安装

无论您选择了哪种安装方式（Mac、Windows、Ubuntu、Linux 或其他），Docker 都应该已经设置好并准备就绪。测试的最佳方法是运行`docker info`命令。输出消息应该类似于以下内容：

```
$ docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
 Images: 0
...
```

# 在服务器上安装

为了在网络上使用 Docker，可以利用云平台提供商或在专用服务器上手动安装 Docker。

在第一种情况下，Docker 配置因平台而异，但在专门的教程中都有很好的描述。大多数云平台都可以通过用户友好的网络界面创建 Docker 主机，或者描述在其服务器上执行的确切命令。

然而，第二种情况（手动安装 Docker）需要一些评论。

# 专用服务器

在服务器上手动安装 Docker 与本地安装并没有太大区别。

还需要两个额外的步骤，包括设置 Docker 守护程序以侦听网络套接字和设置安全证书。

让我们从第一步开始。出于安全原因，默认情况下，Docker 通过非网络化的 Unix 套接字运行，只允许本地通信。必须添加监听所选网络接口套接字，以便外部客户端可以连接。[`docs.docker.com/engine/admin/`](https://docs.docker.com/engine/admin/)详细描述了每个 Linux 发行版所需的所有配置步骤。

在 Ubuntu 的情况下，Docker 守护程序由 systemd 配置，因此为了更改它的启动配置，我们需要修改`/lib/systemd/system/docker.service`文件中的一行：

```
ExecStart=/usr/bin/dockerd -H <server_ip>:2375
```

通过更改这一行，我们启用了通过指定的 IP 地址访问 Docker 守护程序。有关 systemd 配置的所有细节可以在[`docs.docker.com/engine/admin/systemd/`](https://docs.docker.com/engine/admin/systemd/)找到。

服务器配置的第二步涉及 Docker 安全证书。这使得只有通过证书认证的客户端才能访问服务器。Docker 证书配置的详细描述可以在[`docs.docker.com/engine/security/https/`](https://docs.docker.com/engine/security/https/)找到。这一步并不是严格要求的；然而，除非您的 Docker 守护程序服务器位于防火墙网络内，否则是必不可少的。

如果您的 Docker 守护程序在公司网络内运行，您必须配置 HTTP 代理。详细描述可以在[`docs.docker.com/engine/admin/systemd/`](https://docs.docker.com/engine/admin/systemd/)找到。

# 运行 Docker hello world>

Docker 环境已经设置好，所以我们可以开始第一个示例。

在控制台中输入以下命令：

```
$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
78445dd45222: Pull complete
Digest: sha256:c5515758d4c5e1e838e9cd307f6c6a0d620b5e07e6f927b07d05f6d12a1ac8d7
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

恭喜，您刚刚运行了您的第一个 Docker 容器。我希望您已经感受到 Docker 是多么简单。让我们逐步检查发生了什么：

1.  您使用`run`命令运行了 Docker 客户端。

1.  Docker 客户端联系 Docker 守护程序，要求从名为`hello-world`的镜像创建一个容器。

1.  Docker 守护程序检查是否在本地包含`hello-world`镜像，由于没有，它从远程 Docker Hub 注册表请求了`hello-world`镜像。

1.  Docker Hub 注册表包含了`hello-world`镜像，因此它被拉入了 Docker 守护程序。

1.  Docker 守护程序从`hello-world`镜像创建了一个新的容器，启动了产生输出的可执行文件。

1.  Docker 守护程序将此输出流式传输到 Docker 客户端。

1.  Docker 客户端将其发送到您的终端。

预期的流程可以用以下图表表示：

![](img/991a6408-e39f-4455-80eb-b42d883c3a49.png)

让我们看一下在本节中所示的每个 Docker 组件。

# Docker 组件

官方 Docker 页面上说：

“Docker Engine 是一个创建和管理 Docker 对象（如镜像和容器）的客户端-服务器应用程序。”

让我们搞清楚这意味着什么。

# Docker 客户端和服务器

让我们看一下展示 Docker Engine 架构的图表：

![](img/22168b81-9283-421d-8295-d0af9d675db3.png)

Docker Engine 由三个组件组成：

+   **Docker 守护程序**（服务器）在后台运行

+   **Docker 客户端**作为命令工具运行

+   **REST API**

安装 Docker Engine 意味着安装所有组件，以便 Docker 守护程序作为服务在我们的计算机上运行。在`hello-world`示例中，我们使用 Docker 客户端与 Docker 守护程序交互；但是，我们也可以使用 REST API 来做完全相同的事情。同样，在 hello-world 示例中，我们连接到本地 Docker 守护程序；但是，我们也可以使用相同的客户端与远程机器上运行的 Docker 守护程序交互。

要在远程机器上运行 Docker 容器，可以使用`-H`选项：`docker -H <server_ip>:2375 run hello-world`

# Docker 镜像和容器

在 Docker 世界中，镜像是一个无状态的构建块。您可以将镜像想象为运行应用程序所需的所有文件的集合，以及运行它的方法。镜像是无状态的，因此可以通过网络发送它，将其存储在注册表中，命名它，对其进行版本控制，并将其保存为文件。镜像是分层的，这意味着可以在另一个镜像的基础上构建镜像。

容器是镜像的运行实例。如果我们想要多个相同应用的实例，我们可以从同一个镜像创建多个容器。由于容器是有状态的，我们可以与它们交互并更改它们的状态。

让我们来看一个容器和镜像层结构的例子：

![](img/0099e866-f813-47d3-a4b7-b34fb88f722b.png)

底部始终是基础镜像。在大多数情况下，它代表一个操作系统，我们在现有的基础镜像上构建我们的镜像。从技术上讲，可以创建自己的基础镜像，但这很少需要。

在我们的例子中，`ubuntu`基础镜像提供了 Ubuntu 操作系统的所有功能。`add git`镜像添加了 Git 工具包。然后，有一个添加了 JDK 环境的镜像。最后，在顶部，有一个从`add JDK`镜像创建的容器。这样的容器能够从 GitHub 仓库下载 Java 项目并将其编译为 JAR 文件。因此，我们可以使用这个容器来编译和运行 Java 项目，而无需在我们的操作系统上安装任何工具。

重要的是要注意，分层是一种非常聪明的机制，可以节省带宽和存储空间。想象一下，我们的应用程序也是基于`ubuntu`：

![](img/26c3a1aa-7f15-49e9-9c27-e5a2ed80fbdb.png)

这次我们将使用 Python 解释器。在安装`add python`镜像时，Docker 守护程序将注意到`ubuntu`镜像已经安装，并且它需要做的只是添加一个非常小的`python`层。因此，`ubuntu`镜像是一个被重复使用的依赖项。如果我们想要在网络中部署我们的镜像，情况也是一样的。当我们部署 Git 和 JDK 应用程序时，我们需要发送整个`ubuntu`镜像。然而，随后部署`python`应用程序时，我们只需要发送一个小的`add python`层。

# Docker 应用程序

许多应用程序以 Docker 镜像的形式提供，可以从互联网上下载。如果我们知道镜像名称，那么只需以与 hello world 示例相同的方式运行它就足够了。我们如何在 Docker Hub 上找到所需的应用程序镜像呢？

让我们以 MongoDB 为例。如果我们想在 Docker Hub 上找到它，我们有两个选项：

+   在 Docker Hub 探索页面上搜索（[`hub.docker.com/explore/`](https://hub.docker.com/explore/)）

+   使用`docker search`命令

在第二种情况下，我们可以执行以下操作：

```
$ docker search mongo
NAME DESCRIPTION STARS OFFICIAL AUTOMATED
mongo MongoDB document databases provide high av... 2821 [OK] 
mongo-express Web-based MongoDB admin interface, written... 106 [OK] 
mvertes/alpine-mongo light MongoDB container 39 [OK]
mongoclient/mongoclient Official docker image for Mongoclient, fea... 19 [OK]
...
```

有很多有趣的选项。我们如何选择最佳镜像？通常，最吸引人的是没有任何前缀的镜像，因为这意味着它是一个官方的 Docker Hub 镜像，因此应该是稳定和维护的。带有前缀的镜像是非官方的，通常作为开源项目进行维护。在我们的情况下，最好的选择似乎是`mongo`，因此为了运行 MongoDB 服务器，我们可以运行以下命令：

```
$ docker run mongo
Unable to find image 'mongo:latest' locally
latest: Pulling from library/mongo
5040bd298390: Pull complete
ef697e8d464e: Pull complete
67d7bf010c40: Pull complete
bb0b4f23ca2d: Pull complete
8efff42d23e5: Pull complete
11dec5aa0089: Pull complete
e76feb0ad656: Pull complete
5e1dcc6263a9: Pull complete
2855a823db09: Pull complete
Digest: sha256:aff0c497cff4f116583b99b21775a8844a17bcf5c69f7f3f6028013bf0d6c00c
Status: Downloaded newer image for mongo:latest
2017-01-28T14:33:59.383+0000 I CONTROL [initandlisten] MongoDB starting : pid=1 port=27017 dbpath=/data/db 64-bit host=0f05d9df0dc2
...
```

就这样，MongoDB 已经启动了。作为 Docker 容器运行应用程序是如此简单，因为我们不需要考虑任何依赖项；它们都与镜像一起提供。

在 Docker Hub 服务上，你可以找到很多应用程序；它们存储了超过 100,000 个不同的镜像。

# 构建镜像

Docker 可以被视为一个有用的工具来运行应用程序；然而，真正的力量在于构建自己的 Docker 镜像，将程序与环境一起打包。在本节中，我们将看到如何使用两种不同的方法来做到这一点，即 Docker `commit`命令和 Dockerfile 自动构建。

# Docker commit

让我们从一个例子开始，使用 Git 和 JDK 工具包准备一个镜像。我们将使用 Ubuntu 16.04 作为基础镜像。无需创建它；大多数基础镜像都可以在 Docker Hub 注册表中找到：

1.  从`ubuntu:16.04`运行一个容器，并连接到其命令行：

```
 $ docker run -i -t ubuntu:16.04 /bin/bash
```

我们拉取了`ubuntu:16.04`镜像，并将其作为容器运行，然后以交互方式（-i 标志）调用了`/bin/bash`命令。您应该看到容器的终端。由于容器是有状态的和可写的，我们可以在其终端中做任何我们想做的事情。

1.  安装 Git 工具包：

```
 root@dee2cb192c6c:/# apt-get update
 root@dee2cb192c6c:/# apt-get install -y git
```

1.  检查 Git 工具包是否已安装：

```
 root@dee2cb192c6c:/# which git
 /usr/bin/git
```

1.  退出容器：

```
 root@dee2cb192c6c:/# exit
```

1.  检查容器中的更改，将其与`ubuntu`镜像进行比较：

```
 $ docker diff dee2cb192c6c
```

该命令应打印出容器中所有更改的文件列表。

1.  将容器提交到镜像：

```
 $ docker commit dee2cb192c6c ubuntu_with_git
```

我们刚刚创建了我们的第一个 Docker 镜像。让我们列出 Docker 主机上的所有镜像，看看镜像是否存在：

```
$ docker images
REPOSITORY       TAG      IMAGE ID      CREATED            SIZE
ubuntu_with_git  latest   f3d674114fe2  About a minute ago 259.7 MB
ubuntu           16.04    f49eec89601e  7 days ago         129.5 MB
mongo            latest   0dffc7177b06  10 days ago        402 MB
hello-world      latest   48b5124b2768  2 weeks ago        1.84 kB
```

如预期的那样，我们看到了`hello-world`，`mongo`（之前安装的），`ubuntu`（从 Docker Hub 拉取的基础镜像）和新构建的`ubuntu_with_git`。顺便说一句，我们可以观察到每个镜像的大小，它对应于我们在镜像上安装的内容。

现在，如果我们从镜像创建一个容器，它将安装 Git 工具：

```
$ docker run -i -t ubuntu_with_git /bin/bash
root@3b0d1ff457d4:/# which git
/usr/bin/git
root@3b0d1ff457d4:/# exit
```

使用完全相同的方法，我们可以在`ubuntu_with_git`镜像的基础上构建`ubuntu_with_git_and_jdk`：

```
$ docker run -i -t ubuntu_with_git /bin/bash
root@6ee6401ed8b8:/# apt-get install -y openjdk-8-jdk
root@6ee6401ed8b8:/# exit
$ docker commit 6ee6401ed8b8 ubuntu_with_git_and_jdk
```

# Dockerfile

手动创建每个 Docker 镜像并使用 commit 命令可能会很费力，特别是在构建自动化和持续交付过程中。幸运的是，有一种内置语言可以指定构建 Docker 镜像时应执行的所有指令。

让我们从一个类似于 Git 和 JDK 的例子开始。这次，我们将准备`ubuntu_with_python`镜像。

1.  创建一个新目录和一个名为`Dockerfile`的文件，内容如下：

```
 FROM ubuntu:16.04
 RUN apt-get update && \
 apt-get install -y python
```

1.  运行命令以创建`ubuntu_with_python`镜像：

```
 $ docker build -t ubuntu_with_python .
```

1.  检查镜像是否已创建：

```
$ docker images
REPOSITORY              TAG     IMAGE ID       CREATED            SIZE
ubuntu_with_python      latest  d6e85f39f5b7  About a minute ago 202.6 MB
ubuntu_with_git_and_jdk latest  8464dc10abbb  3 minutes ago      610.9 MB
ubuntu_with_git         latest  f3d674114fe2  9 minutes ago      259.7 MB
ubuntu                  16.04   f49eec89601e  7 days ago         129.5 MB
mongo                   latest  0dffc7177b06   10 days ago        402 MB
hello-world             latest  48b5124b2768   2 weeks ago        1.84 kB
```

现在我们可以从镜像创建一个容器，并检查 Python 解释器是否存在，方式与执行`docker commit`命令后的方式完全相同。请注意，即使`ubuntu`镜像是`ubuntu_with_git`和`ubuntu_with_python`的基础镜像，它也只列出一次。

在这个例子中，我们使用了前两个 Dockerfile 指令：

+   `FROM`定义了新镜像将基于的镜像

+   `RUN`指定在容器内部运行的命令

所有 Dockerfile 指令都可以在官方 Docker 页面[`docs.docker.com/engine/reference/builder/`](https://docs.docker.com/engine/reference/builder/)上找到。最常用的指令如下：

+   `MAINTAINER`定义了关于作者的元信息

+   `COPY`将文件或目录复制到镜像的文件系统中

+   `ENTRYPOINT`定义了可执行容器中应该运行哪个应用程序

您可以在官方 Docker 页面[https://docs.docker.com/engine/reference/builder/]上找到所有 Dockerfile 指令的完整指南。

# 完整的 Docker 应用程序

我们已经拥有构建完全可工作的应用程序作为 Docker 镜像所需的所有信息。例如，我们将逐步准备一个简单的 Python hello world 程序。无论我们使用什么环境或编程语言，这些步骤都是相同的。

# 编写应用程序

创建一个新目录，在这个目录中，创建一个名为`hello.py`的文件，内容如下：

```
print "Hello World from Python!"
```

关闭文件。这是我们应用程序的源代码。

# 准备环境

我们的环境将在 Dockerfile 中表示。我们需要定义以下指令：

+   应该使用哪个基础镜像

+   （可选）维护者是谁

+   如何安装 Python 解释器

+   如何将`hello.py`包含在镜像中

+   如何启动应用程序

在同一目录中，创建 Dockerfile：

```
FROM ubuntu:16.04
MAINTAINER Rafal Leszko
RUN apt-get update && \
    apt-get install -y python
COPY hello.py .
ENTRYPOINT ["python", "hello.py"]
```

# 构建镜像

现在，我们可以以与之前完全相同的方式构建镜像：

```
$ docker build -t hello_world_python .
```

# 运行应用程序

我们通过运行容器来运行应用程序：

```
$ docker run hello_world_python
```

您应该看到友好的 Hello World from Python!消息。这个例子中最有趣的是，我们能够在没有在主机系统中安装 Python 解释器的情况下运行 Python 编写的应用程序。这是因为作为镜像打包的应用程序在内部具有所需的所有环境。

Python 解释器的镜像已经存在于 Docker Hub 服务中，因此在实际情况下，使用它就足够了。

# 环境变量

我们已经运行了我们的第一个自制 Docker 应用程序。但是，如果应用程序的执行应该取决于一些条件呢？

例如，在生产服务器的情况下，我们希望将`Hello`打印到日志中，而不是控制台，或者我们可能希望在测试阶段和生产阶段有不同的依赖服务。一个解决方案是为每种情况准备一个单独的 Dockerfile；然而，还有一个更好的方法，即环境变量。

让我们将我们的 hello world 应用程序更改为打印`Hello World from` `<name_passed_as_environment_variable> !`。为了做到这一点，我们需要按照以下步骤进行：

1.  更改 Python 脚本以使用环境变量：

```
        import os
        print "Hello World from %s !" % os.environ['NAME']
```

1.  构建镜像：

```
 $ docker build -t hello_world_python_name .
```

1.  运行传递环境变量的容器：

```
 $ docker run -e NAME=Rafal hello_world_python_name
 Hello World from Rafal !
```

1.  或者，我们可以在 Dockerfile 中定义环境变量的值，例如：

```
        ENV NAME Rafal
```

1.  然后，我们可以运行容器而不指定`-e`选项。

```
 $ docker build -t hello_world_python_name_default .
 $ docker run hello_world_python_name_default
 Hello World from Rafal !
```

当我们需要根据其用途拥有 Docker 容器的不同版本时，例如，为生产和测试服务器拥有单独的配置文件时，环境变量尤其有用。

如果环境变量在 Dockerfile 和标志中都有定义，那么命令标志优先。

# Docker 容器状态

到目前为止，我们运行的每个应用程序都应该做一些工作然后停止。例如，我们打印了`Hello from Docker!`然后退出。但是，有些应用程序应该持续运行，比如服务。要在后台运行容器，我们可以使用`-d`（`--detach`）选项。让我们尝试一下`ubuntu`镜像：

```
$ docker run -d -t ubuntu:16.04
```

这个命令启动了 Ubuntu 容器，但没有将控制台附加到它上面。我们可以使用以下命令看到它正在运行：

```
$ docker ps
CONTAINER ID IMAGE        COMMAND     STATUS PORTS NAMES
95f29bfbaadc ubuntu:16.04 "/bin/bash" Up 5 seconds kickass_stonebraker
```

这个命令打印出所有处于运行状态的容器。那么我们的旧容器呢，已经退出了？我们可以通过打印所有容器来找到它们：

```
$ docker ps -a
CONTAINER ID IMAGE        COMMAND        STATUS PORTS  NAMES
95f29bfbaadc ubuntu:16.04 "/bin/bash"    Up 33 seconds kickass_stonebraker
34080d914613 hello_world_python_name_default "python hello.py" Exited lonely_newton
7ba49e8ee677 hello_world_python_name "python hello.py" Exited mad_turing
dd5eb1ed81c3 hello_world_python "python hello.py" Exited thirsty_bardeen
6ee6401ed8b8 ubuntu_with_git "/bin/bash" Exited        grave_nobel
3b0d1ff457d4 ubuntu_with_git "/bin/bash" Exited        desperate_williams
dee2cb192c6c ubuntu:16.04 "/bin/bash"    Exited        small_dubinsky
0f05d9df0dc2 mongo        "/entrypoint.sh mongo" Exited trusting_easley
47ba1c0ba90e hello-world  "/hello"       Exited        tender_bell
```

请注意，所有旧容器都处于退出状态。我们还没有观察到的状态有两种：暂停和重新启动。

所有状态及其之间的转换都在以下图表中显示：

![](img/9b56cc40-6571-4e7b-98f0-7617455661b3.png)

暂停 Docker 容器非常罕见，从技术上讲，它是通过使用 SIGSTOP 信号冻结进程来实现的。重新启动是一个临时状态，当容器使用`--restart`选项运行以定义重新启动策略时（Docker 守护程序能够在发生故障时自动重新启动容器）。

该图表还显示了用于将 Docker 容器状态从一个状态更改为另一个状态的 Docker 命令。

例如，我们可以停止正在运行的 Ubuntu 容器：

```
$ docker stop 95f29bfbaadc

$ docker ps
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
```

我们一直使用`docker run`命令来创建和启动容器；但是，也可以只创建容器而不启动它。

# Docker 网络

如今，大多数应用程序不是独立运行的，而是需要通过网络与其他系统进行通信。如果我们想在 Docker 容器内运行网站、网络服务、数据库或缓存服务器，那么我们需要至少了解 Docker 网络的基础知识。

# 运行服务

让我们从一个简单的例子开始，直接从 Docker Hub 运行 Tomcat 服务器：

```
$ docker run -d tomcat
```

Tomcat 是一个 Web 应用程序服务器，其用户界面可以通过端口`8080`访问。因此，如果我们在本机安装了 Tomcat，我们可以在`http://localhost:8080`上浏览它。

然而，在我们的情况下，Tomcat 是在 Docker 容器内运行的。我们以与第一个`Hello World`示例相同的方式启动了它。我们可以看到它正在运行：

```
$ docker ps
CONTAINER ID IMAGE  COMMAND           STATUS            PORTS    NAMES
d51ad8634fac tomcat "catalina.sh run" Up About a minute 8080/tcp jovial_kare
```

由于它是作为守护进程运行的（使用`-d`选项），我们无法立即在控制台中看到日志。然而，我们可以通过执行以下代码来访问它：

```
$ docker logs d51ad8634fac
```

如果没有错误，我们应该会看到很多日志，说明 Tomcat 已经启动，并且可以通过端口`8080`访问。我们可以尝试访问`http://localhost:8080`，但是我们无法连接。原因是 Tomcat 已经在容器内启动，我们试图从外部访问它。换句话说，我们只能在连接到容器中的控制台并在那里检查时才能访问它。如何使正在运行的 Tomcat 可以从外部访问呢？

我们需要启动容器并指定端口映射，使用`-p`（`--publish`）标志：

```
-p, --publish <host_port>:<container_port>
```

因此，让我们首先停止正在运行的容器并启动一个新的容器：

```
$ docker stop d51ad8634fac
$ docker run -d -p 8080:8080 tomcat
```

等待几秒钟后，Tomcat 必须已经启动，我们应该能够打开它的页面，`http://localhost:8080`。

![](img/5a754575-bd73-41d5-9f7c-01590ca4ecb7.png)

在大多数常见的 Docker 使用情况下，这样简单的端口映射命令就足够了。我们能够将（微）服务部署为 Docker 容器，并公开它们的端口以启用通信。然而，让我们深入了解一下发生在幕后的情况。

Docker 允许使用`-p <ip>:<host_port>:<container_port>`将指定的主机网络接口发布出去。

# 容器网络

我们已经连接到容器内运行的应用程序。事实上，这种连接是双向的，因为如果你还记得我们之前的例子，我们是从内部执行`apt-get install`命令，并且包是从互联网下载的。这是如何可能的呢？

如果您检查您的机器上的网络接口，您会看到其中一个接口被称为`docker0`：

```
$ ifconfig docker0
docker0 Link encap:Ethernet HWaddr 02:42:db:d0:47:db 
 inet addr:172.17.0.1 Bcast:0.0.0.0 Mask:255.255.0.0
...
```

`docker0`接口是由 Docker 守护程序创建的，以便与 Docker 容器连接。现在，我们可以使用`docker inspect`命令查看 Docker 容器内创建的接口：

```
$ docker inspect 03d1e6dc4d9e
```

它以 JSON 格式打印有关容器配置的所有信息。其中，我们可以找到与网络设置相关的部分。

```
"NetworkSettings": {
     "Bridge": "",
     "Ports": {
          "8080/tcp": [
               {
                    "HostIp": "0.0.0.0",
                    "HostPort": "8080"
               }
          ]
          },
     "Gateway": "172.17.0.1",
     "IPAddress": "172.17.0.2",
     "IPPrefixLen": 16,
}
```

为了过滤`docker inspect`的响应，我们可以使用`--format`选项，例如，`docker inspect --format '{{ .NetworkSettings.IPAddress }}' <container_id>`。

我们可以观察到 Docker 容器的 IP 地址为`172.17.0.2`，并且它与具有 IP 地址`172.17.0.1`的 Docker 主机进行通信。这意味着在我们之前的示例中，即使没有端口转发，我们也可以访问 Tomcat 服务器，使用地址`http://172.17.0.2:8080`。然而，在大多数情况下，我们在服务器机器上运行 Docker 容器并希望将其暴露到外部，因此我们需要使用`-p`选项。

请注意，默认情况下，容器受主机防火墙系统保护，并且不会从外部系统打开任何路由。我们可以通过使用`--network`标志并将其设置为以下内容来更改此默认行为：

+   `bridge`（默认）：通过默认 Docker 桥接网络

+   `none`：无网络

+   `container`：与其他（指定的）容器连接的网络

+   `host`：主机网络（无防火墙）

不同的网络可以通过`docker network`命令列出和管理：

```
$ docker network ls
NETWORK ID   NAME   DRIVER SCOPE
b3326cb44121 bridge bridge local 
84136027df04 host   host   local 
80c26af0351c none   null   local
```

如果我们将`none`指定为网络，则将无法连接到容器，反之亦然；容器无法访问外部世界。`host`选项使容器网络接口与主机相同。它们共享相同的 IP 地址，因此容器上启动的所有内容在外部可见。最常用的选项是默认选项（`bridge`），因为它允许我们明确定义应发布哪些端口。它既安全又可访问。

# 暴露容器端口

我们多次提到容器暴露端口。实际上，如果我们深入研究 GitHub 上的 Tomcat 镜像（[`github.com/docker-library/tomcat`](https://github.com/docker-library/tomcat)），我们可以注意到 Dockerfile 中的以下行：

```
EXPOSE 8080
```

这个 Dockerfile 指令表示应该从容器中公开端口 8080。然而，正如我们已经看到的，这并不意味着端口会自动发布。EXPOSE 指令只是通知用户应该发布哪些端口。

# 自动端口分配

让我们尝试在不停止第一个 Tomcat 容器的情况下运行第二个 Tomcat 容器：

```
$ docker run -d -p 8080:8080 tomcat
0835c95538aeca79e0305b5f19a5f96cb00c5d1c50bed87584cfca8ec790f241
docker: Error response from daemon: driver failed programming external connectivity on endpoint distracted_heyrovsky (1b1cee9896ed99b9b804e4c944a3d9544adf72f1ef3f9c9f37bc985e9c30f452): Bind for 0.0.0.0:8080 failed: port is already allocated.
```

这种错误可能很常见。在这种情况下，我们要么自己负责端口的唯一性，要么让 Docker 使用`publish`命令的以下版本自动分配端口：

+   -p <container_port>：将容器端口发布到未使用的主机端口

+   `-P`（`--publish-all`）：将容器公开的所有端口发布到未使用的主机端口：

```
$ docker run -d -P tomcat
 078e9d12a1c8724f8aa27510a6390473c1789aa49e7f8b14ddfaaa328c8f737b

$ docker port 078e9d12a1c8
8080/tcp -> 0.0.0.0:32772
```

我们可以看到第二个 Tomcat 已发布到端口`32772`，因此可以在`http://localhost:32772`上浏览。

# 使用 Docker 卷

假设您想将数据库作为容器运行。您可以启动这样一个容器并输入数据。它存储在哪里？当您停止容器或删除它时会发生什么？您可以启动新的容器，但数据库将再次为空。除非这是您的测试环境，您不会期望这样的情况发生。

Docker 卷是 Docker 主机的目录，挂载在容器内部。它允许容器像写入自己的文件系统一样写入主机的文件系统。该机制如下图所示：

![](img/b175c0eb-1e9a-4d07-8f40-8ec867942345.png)

Docker 卷使容器的数据持久化和共享。卷还清楚地将处理与数据分开。

让我们从一个示例开始，并使用`-v <host_path>:<container_path>`选项指定卷并连接到容器：

```
$ docker run -i -t -v ~/docker_ubuntu:/host_directory ubuntu:16.04 /bin/bash
```

现在，我们可以在容器中的`host_directory`中创建一个空文件：

```
root@01bf73826624:/# touch host_directory/file.txt
```

让我们检查一下文件是否在 Docker 主机的文件系统中创建：

```
root@01bf73826624:/# exit
exit

$ ls ~/docker_ubuntu/
file.txt
```

我们可以看到文件系统被共享，数据因此得以永久保存。现在我们可以停止容器并运行一个新的容器，看到我们的文件仍然在那里：

```
$ docker stop 01bf73826624

$ docker run -i -t -v ~/docker_ubuntu:/host_directory ubuntu:16.04 /bin/bash
root@a9e0df194f1f:/# ls host_directory/
file.txt

root@a9e0df194f1f:/# exit
```

不需要使用`-v`标志来指定卷，可以在 Dockerfile 中将卷指定为指令，例如：

```
VOLUME /host_directory
```

在这种情况下，如果我们运行 docker 容器而没有`-v`标志，那么容器的`/host_directory`将被映射到主机的默认卷目录`/var/lib/docker/vfs/`。如果您将应用程序作为镜像交付，并且知道它因某种原因需要永久存储（例如存储应用程序日志），这是一个很好的解决方案。

如果卷在 Dockerfile 中和作为标志定义，那么命令标志优先。

Docker 卷可能会更加复杂，特别是在数据库的情况下。然而，Docker 卷的更复杂的用例超出了本书的范围。

使用 Docker 进行数据管理的一个非常常见的方法是引入一个额外的层，即数据卷容器。数据卷容器是一个唯一目的是声明卷的 Docker 容器。然后，其他容器可以使用它（使用`--volumes-from <container>`选项）而不是直接声明卷。在[`docs.docker.com/engine/tutorials/dockervolumes/#creating-and-mounting-a-data-volume-container`](https://docs.docker.com/engine/tutorials/dockervolumes/#creating-and-mounting-a-data-volume-container)中了解更多。

# 在 Docker 中使用名称

到目前为止，当我们操作容器时，我们总是使用自动生成的名称。这种方法有一些优势，比如名称是唯一的（没有命名冲突）和自动的（不需要做任何事情）。然而，在许多情况下，最好为容器或镜像提供一个真正用户友好的名称。

# 命名容器

命名容器有两个很好的理由：方便和自动化：

+   方便，因为通过名称对容器进行任何操作比检查哈希或自动生成的名称更简单

+   自动化，因为有时我们希望依赖于容器的特定命名

例如，我们希望有一些相互依赖的容器，并且有一个链接到另一个。因此，我们需要知道它们的名称。

要命名容器，我们使用`--name`参数：

```
$ docker run -d --name tomcat tomcat
```

我们可以通过`docker ps`检查容器是否有有意义的名称。此外，作为结果，任何操作都可以使用容器的名称执行，例如：

```
$ docker logs tomcat
```

请注意，当容器被命名时，它不会失去其身份。我们仍然可以像以前一样通过自动生成的哈希 ID 来寻址容器。

容器始终具有 ID 和名称。可以通过任何一个来寻址，它们两个都是唯一的。

# 给图像打标签

图像可以被标记。我们在创建自己的图像时已经做过这个，例如，在构建`hello-world_python`图像的情况下：

```
$ docker build -t hello-world_python .
```

`-t`标志描述了图像的标签。如果我们没有使用它，那么图像将被构建而没有任何标签，结果我们将不得不通过其 ID（哈希）来寻址它以运行容器。

图像可以有多个标签，并且它们应该遵循命名约定：

```
<registry_address>/<image_name>:<version>
```

标签由以下部分组成：

+   `registry_address`：注册表的 IP 和端口或别名

+   `image_name`：构建的图像的名称，例如，`ubuntu`

+   `version`：图像的版本，可以是任何形式，例如，16.04，20170310

我们将在第五章中介绍 Docker 注册表，*自动验收测试*。如果图像保存在官方 Docker Hub 注册表上，那么我们可以跳过注册表地址。这就是为什么我们在没有任何前缀的情况下运行了`tomcat`图像。最后一个版本总是被标记为最新的，也可以被跳过，所以我们在没有任何后缀的情况下运行了`tomcat`图像。

图像通常有多个标签，例如，所有四个标签都是相同的图像：`ubuntu:16.04`，`ubuntu:xenial-20170119`，`ubuntu:xenial`和`ubuntu:latest`。

# Docker 清理

在本章中，我们创建了许多容器和图像。然而，这只是现实场景中的一小部分。即使容器此刻没有运行，它们也需要存储在 Docker 主机上。这很快就会导致存储空间超出并停止机器。我们如何解决这个问题呢？

# 清理容器

首先，让我们看看存储在我们的机器上的容器。要打印所有容器（无论它们的状态如何），我们可以使用`docker ps -a`命令：

```
$ docker ps -a
CONTAINER ID IMAGE  COMMAND           STATUS  PORTS  NAMES
95c2d6c4424e tomcat "catalina.sh run" Up 5 minutes 8080/tcp tomcat
a9e0df194f1f ubuntu:16.04 "/bin/bash" Exited         jolly_archimedes
01bf73826624 ubuntu:16.04 "/bin/bash" Exited         suspicious_feynman
078e9d12a1c8 tomcat "catalina.sh run" Up 14 minutes 0.0.0.0:32772->8080/tcp nauseous_fermi
0835c95538ae tomcat "catalina.sh run" Created        distracted_heyrovsky
03d1e6dc4d9e tomcat "catalina.sh run" Up 50 minutes 0.0.0.0:8080->8080/tcp drunk_ritchie
d51ad8634fac tomcat "catalina.sh run" Exited         jovial_kare
95f29bfbaadc ubuntu:16.04 "/bin/bash" Exited         kickass_stonebraker
34080d914613 hello_world_python_name_default "python hello.py" Exited lonely_newton
7ba49e8ee677 hello_world_python_name "python hello.py" Exited mad_turing
dd5eb1ed81c3 hello_world_python "python hello.py" Exited thirsty_bardeen
6ee6401ed8b8 ubuntu_with_git "/bin/bash" Exited      grave_nobel
3b0d1ff457d4 ubuntu_with_git "/bin/bash" Exited      desperate_williams
dee2cb192c6c ubuntu:16.04 "/bin/bash" Exited         small_dubinsky
0f05d9df0dc2 mongo  "/entrypoint.sh mongo" Exited    trusting_easley
47ba1c0ba90e hello-world "/hello"     Exited         tender_bell
```

为了删除已停止的容器，我们可以使用`docker rm`命令（如果容器正在运行，我们需要先停止它）：

```
$ docker rm 47ba1c0ba90e
```

如果我们想要删除所有已停止的容器，我们可以使用以下命令：

```
$ docker rm $(docker ps --no-trunc -aq)
```

`-aq`选项指定仅传递所有容器的 ID（没有额外数据）。另外，`--no-trunc`要求 Docker 不要截断输出。

我们也可以采用不同的方法，并要求容器在停止时使用`--rm`标志自行删除，例如：

```
$ docker run --rm hello-world
```

在大多数实际场景中，我们不使用已停止的容器，它们只用于调试目的。

# 清理图像

图像和容器一样重要。它们可能占用大量空间，特别是在持续交付过程中，每次构建都会产生一个新的 Docker 图像。这很快就会导致设备上没有空间的错误。要检查 Docker 容器中的所有图像，我们可以使用`docker images`命令：

```
$ docker images
REPOSITORY TAG                         IMAGE ID     CREATED     SIZE
hello_world_python_name_default latest 9a056ca92841 2 hours ago 202.6 MB
hello_world_python_name latest         72c8c50ffa89 2 hours ago 202.6 MB
hello_world_python latest              3e1fa5c29b44 2 hours ago 202.6 MB
ubuntu_with_python latest              d6e85f39f5b7 2 hours ago 202.6 MB
ubuntu_with_git_and_jdk latest         8464dc10abbb 2 hours ago 610.9 MB
ubuntu_with_git latest                 f3d674114fe2 3 hours ago 259.7 MB
tomcat latest                          c822d296d232 2 days ago  355.3 MB
ubuntu 16.04                           f49eec89601e 7 days ago  129.5 MB
mongo latest                           0dffc7177b06 11 days ago 402 MB
hello-world latest                     48b5124b2768 2 weeks ago 1.84 kB
```

要删除图像，我们可以调用以下命令：

```
$ docker rmi 48b5124b2768
```

在图像的情况下，自动清理过程稍微复杂一些。图像没有状态，所以我们不能要求它们在不使用时自行删除。常见的策略是设置 Cron 清理作业，删除所有旧的和未使用的图像。我们可以使用以下命令来做到这一点：

```
$ docker rmi $(docker images -q)
```

为了防止删除带有标签的图像（例如，不删除所有最新的图像），非常常见的是使用`dangling`参数：

```
$ docker rmi $(docker images -f "dangling=true" -q)
```

如果我们有使用卷的容器，那么除了图像和容器之外，还值得考虑清理卷。最简单的方法是使用`docker volume ls -qf dangling=true | xargs -r docker volume rm`命令。

# Docker 命令概述

通过执行以下`help`命令可以找到所有 Docker 命令：

```
$ docker help
```

要查看任何特定 Docker 命令的所有选项，我们可以使用`docker help <command>`，例如：

```
$ docker help run
```

在官方 Docker 页面[`docs.docker.com/engine/reference/commandline/docker/`](https://docs.docker.com/engine/reference/commandline/docker/)上也有对所有 Docker 命令的很好的解释。真的值得阅读，或者至少浏览一下。

在本章中，我们已经介绍了最有用的命令及其选项。作为一个快速提醒，让我们回顾一下：

| **命令** | **解释** |
| --- | --- |
| `docker build` | 从 Dockerfile 构建图像 |
| `docker commit` | 从容器创建图像 |
| `docker diff` | 显示容器中的更改 |
| `docker images` | 列出图像 |
| `docker info` | 显示 Docker 信息 |
| `docker inspect` | 显示 Docker 镜像/容器的配置 |
| `docker logs` | 显示容器的日志 |
| `docker network` | 管理网络 |
| `docker port` | 显示容器暴露的所有端口 |
| `docker ps` | 列出容器 |
| `docker rm` | 删除容器 |
| `docker rmi` | 删除图像 |
| `docker run` | 从图像运行容器 |
| `docker search` | 在 Docker Hub 中搜索 Docker 镜像 |
| `docker start/stop/pause/unpause` | 管理容器的状态 |

# 练习

在本章中，我们涵盖了大量的材料。为了记忆深刻，我们建议进行两个练习。

1.  运行`CouchDB`作为一个 Docker 容器并发布它的端口：

您可以使用`docker search`命令来查找`CouchDB`镜像。

+   +   运行容器

+   发布`CouchDB`端口

+   打开浏览器并检查`CouchDB`是否可用

1.  创建一个 Docker 镜像，其中 REST 服务回复`Hello World!`到`localhost:8080/hello`。使用您喜欢的任何语言和框架：

创建 REST 服务的最简单方法是使用 Python 和 Flask 框架，[`flask.pocoo.org/`](http://flask.pocoo.org/)。请注意，许多 Web 框架默认只在 localhost 接口上启动应用程序。为了发布端口，有必要在所有接口上启动它（在 Flask 框架的情况下，使用`app.run(host='0.0.0.0')`）。

+   +   创建一个 Web 服务应用程序

+   创建一个 Dockerfile 来安装依赖和库

+   构建镜像

+   运行容器并发布端口

+   使用浏览器检查它是否正常运行

# 总结

在本章中，我们已经涵盖了足够构建镜像和运行应用程序作为容器的 Docker 基础知识。本章的关键要点如下：

+   容器化技术利用 Linux 内核特性解决了隔离和环境依赖的问题。这是基于进程分离机制的，因此没有观察到真正的性能下降。

+   Docker 可以安装在大多数系统上，但只有在 Linux 上才能得到原生支持。

+   Docker 允许从互联网上可用的镜像中运行应用程序，并构建自己的镜像。

+   镜像是一个打包了所有依赖关系的应用程序。

+   Docker 提供了两种构建镜像的方法：Dockerfile 或提交容器。在大多数情况下，第一种选项被使用。

+   Docker 容器可以通过发布它们暴露的端口进行网络通信。

+   Docker 容器可以使用卷共享持久存储。

+   为了方便起见，Docker 容器应该被命名，Docker 镜像应该被标记。在 Docker 世界中，有一个特定的约定来标记镜像。

+   Docker 镜像和容器应该定期清理，以节省服务器空间并避免*设备上没有空间*的错误。

在下一章中，我们将介绍 Jenkins 的配置以及 Jenkins 与 Docker 一起使用的方式。

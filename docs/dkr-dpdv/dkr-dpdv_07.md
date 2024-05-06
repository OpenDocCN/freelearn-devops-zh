## 5：Docker 引擎

在本章中，我们将快速查看 Docker 引擎的内部情况。

你可以在不了解本章内容的情况下使用 Docker。所以，随意跳过它。然而，要成为真正的专家，你需要了解底层发生了什么。所以，要成为一个*真正*的 Docker 专家，你需要了解本章的内容。

这将是一个基于理论的章节，没有实际操作练习。

由于本章是书中**技术部分**的一部分，我们将采用三层方法，将章节分为三个部分：

+   **简而言之：** 两三段简短的内容，你可以在排队买咖啡时阅读

+   **深入挖掘：** 我们深入细节的部分

+   **命令：** 我们学到的命令的快速回顾

让我们去学习关于 Docker 引擎的知识！

### Docker 引擎 - 简而言之

*Docker 引擎*是运行和管理容器的核心软件。我们经常简称为*Docker*或*Docker 平台*。如果你对 VMware 有所了解，把它想象成类似于 ESXi 可能会有所帮助。

Docker 引擎的设计是模块化的，具有许多可互换的组件。在可能的情况下，这些组件基于开放标准，由开放容器倡议（OCI）概述。

在许多方面，Docker 引擎就像汽车引擎 - 都是模块化的，通过连接许多小的专门部件创建而成：

+   汽车引擎由许多专门的部件组成，这些部件共同工作使汽车行驶 - 进气歧管、节气门、气缸、火花塞、排气歧管等。

+   Docker 引擎由许多专门的工具组成，这些工具共同工作以创建和运行容器 - API、执行驱动程序、运行时、shims 等。

在撰写本文时，构成 Docker 引擎的主要组件有：*Docker 客户端*、*Docker 守护程序*、*containerd*和*runc*。这些组件共同创建和运行容器。

图 5.1 显示了一个高层次的视图。

![图 5.1](img/figure5-1.png)

图 5.1

在整本书中，我们将用小写字母“r”和“c”来指代`runc`和`containerd`。这意味着以`____r____unc` `____c____ontainerd`开头的句子将不以大写字母开头。这是故意的，不是错误。

### Docker 引擎 - 深入挖掘

当 Docker 首次发布时，Docker 引擎有两个主要组件：

+   Docker 守护程序（以下简称“守护程序”）

+   LXC

Docker 守护程序是一个单一的二进制文件。它包含了 Docker 客户端、Docker API、容器运行时、镜像构建等等的所有代码。

LXC 为守护程序提供了访问 Linux 内核中存在的容器的基本构建模块。诸如*命名空间*和*控制组（cgroups）*之类的东西。

图 5.2 显示了在旧版本的 Docker 中守护程序、LXC 和操作系统是如何交互的。

![图 5.2 以前的 Docker 架构](img/figure5-2.png)

图 5.2 以前的 Docker 架构

#### 摆脱 LXC

从一开始就依赖 LXC 是一个问题。

首先，LXC 是特定于 Linux 的。这对于一个有多平台愿景的项目来说是一个问题。

其次，对于一个如此核心的项目来说，依赖外部工具是一个巨大的风险，可能会阻碍开发。

因此，Docker. Inc.开发了他们自己的工具*libcontainer*来替代 LXC。*libcontainer*的目标是成为一个平台无关的工具，为 Docker 提供访问内核中存在的基本容器构建模块。

Libcontainer 在 Docker 0.9 中取代了 LXC 成为默认的*执行驱动程序*。

#### 摆脱单一的 Docker 守护程序

随着时间的推移，Docker 守护程序的单一性变得越来越成问题：

1.  这很难进行创新。

1.  它变得更慢了。

1.  这并不是生态系统（或 Docker, Inc.）想要的。

Docker, Inc.意识到了这些挑战，并开始了一项巨大的工作，以拆分单一的守护程序并使其模块化。这项工作的目标是尽可能地从守护程序中分离出尽可能多的功能，并在较小的专门工具中重新实现它。这些专门工具可以被替换，也可以被第三方轻松重用以构建其他工具。这个计划遵循了经过验证的 Unix 哲学，即构建小型专门的工具，可以拼接成更大的工具。

拆分和重构 Docker 引擎的工作是一个持续的过程。然而，它已经看到**所有的*容器执行*和容器*运行时*代码完全从守护程序中移除，并重构为小型的专门工具**。

图 5.3 显示了当前 Docker 引擎架构的高层视图，并附有简要描述。

![图 5.3](img/figure5-3.png)

图 5.3

#### 开放容器倡议（OCI）的影响

在 Docker 公司拆分守护程序并重构代码的同时，OCI 正在定义两个与容器相关的规范（也称为标准）：

1.  [镜像规范](https://github.com/opencontainers/image-spec)

1.  [容器运行时规范](https://github.com/opencontainers/runtime-spec/blob/master/RELEASES.md)

这两个规范于 2017 年 7 月发布为 1.0 版本。

Docker 公司在创建这些规范方面发挥了重要作用，并为其贡献了大量代码。

从 Docker 1.11（2016 年初）开始，Docker 引擎尽可能地实现了 OCI 规范。例如，Docker 守护程序不再包含任何容器运行时代码 — 所有容器运行时代码都在一个单独的符合 OCI 规范的层中实现。默认情况下，Docker 使用一个名为*runc*的工具。runc 是 OCI 容器运行时规范的*参考实现*。这是图 5.3 中的`runc`容器运行时层。runc 项目的目标是与 OCI 规范保持一致。然而，现在 OCI 规范都已经达到 1.0 版本，我们不应该期望它们会有太多迭代 — 稳定性才是关键。

此外，Docker 引擎的*containerd*组件确保 Docker 镜像以有效的 OCI 捆绑包形式呈现给*runc*。

> **注意：** Docker 引擎在规范正式发布为 1.0 版本之前就已经实现了 OCI 规范的部分内容。

#### runc

如前所述，*runc*是 OCI 容器运行时规范的参考实现。Docker 公司在定义规范和开发 runc 方面发挥了重要作用。

如果你剥离其他一切，runc 只是一个小巧、轻量级的 CLI 包装器，用于 libcontainer（请记住，libcontainer 最初替代了 Docker 早期架构中的 LXC）。

runc 在生活中只有一个目的 — 创建容器。而且它做得非常好。而且快！但由于它是一个 CLI 包装器，它实际上是一个独立的容器运行时工具。这意味着您可以下载和构建二进制文件，然后就拥有了构建和使用 runc（OCI）容器所需的一切。但它只是基本功能，您将无法获得完整的 Docker 引擎所具有的丰富功能。

我们有时称 runc 操作的层为“OCI 层”。参见图 5.3。

您可以在以下链接查看 runc 的发布信息：

+   https://github.com/opencontainers/runc/releases

#### containerd

作为从 Docker 守护程序中剥离功能的努力的一部分，所有容器执行逻辑都被剥离并重构为一个名为 containerd（发音为 container-dee）的新工具。它的唯一目的是管理容器的生命周期操作-`start | stop | pause | rm...`。

containerd 可用作 Linux 和 Windows 的守护程序，并且自 1.11 版本以来，Docker 一直在 Linux 上使用它。在 Docker 引擎堆栈中，containerd 位于守护程序和 OCI 层的 runc 之间。Kubernetes 也可以通过 cri-containerd 使用 containerd。

正如先前所述，containerd 最初旨在小巧，轻量，并设计用于生命周期操作。然而，随着时间的推移，它已经扩展并承担了更多功能。比如镜像管理。

其中一个原因是为了使它更容易在其他项目中使用。例如，containerd 是 Kubernetes 中流行的容器运行时。然而，在像 Kubernetes 这样的项目中，containerd 能够执行额外的操作，比如推送和拉取镜像，这是有益的。因此，containerd 现在做的不仅仅是简单的容器生命周期管理。然而，所有额外的功能都是模块化和可选的，这意味着你可以选择你想要的部分。因此，可以将 containerd 包含在诸如 Kubernetes 之类的项目中，但只需选择你的项目需要的部分。

containerd 是由 Docker，Inc.开发的，并捐赠给了 Cloud Native Computing Foundation（CNCF）。它于 2017 年 12 月发布了 1.0 版本。您可以在以下链接查看发布信息：

+   https://github.com/containerd/containerd/releases

#### 启动一个新容器（示例）

现在我们已经了解了整体情况和部分历史，让我们来看看创建一个新容器的过程。

启动容器的最常见方式是使用 Docker CLI。以下`docker container run`命令将基于`alpine:latest`镜像启动一个简单的新容器。

```
$ docker container run --name ctr1 -it alpine:latest sh 
```

`当您在 Docker CLI 中输入这样的命令时，Docker 客户端会将它们转换为适当的 API 负载并将其 POST 到正确的 API 端点。`

API 是在守护程序中实现的。这是相同丰富，版本化的 REST API，已成为 Docker 的标志，并在行业中被接受为事实上的容器 API。

一旦守护进程接收到创建新容器的命令，它就会调用 containerd。请记住，守护进程不再包含任何创建容器的代码！

守护进程通过 [gRPC](https://grpc.io/) 与 containerd 进行 CRUD 风格的 API 通信。

尽管 *containerd* 的名字中带有“container”，但它实际上不能创建容器。它使用 *runc* 来完成这个任务。它将所需的 Docker 镜像转换为 OCI bundle，并告诉 runc 使用这个 bundle 来创建一个新的容器。

runc 与操作系统内核进行接口，汇集所有必要的构造来创建一个容器（命名空间、cgroups 等）。容器进程作为 runc 的子进程启动，一旦启动，runc 就会退出。

哇！容器现在已经启动了。

这个过程在图 5.4 中总结了。

![图 5.4](img/figure5-4.png)

图 5.4

#### 这种模型的一个巨大好处

从守护进程中移除了启动和管理容器的所有逻辑和代码意味着整个容器运行时与 Docker 守护进程解耦。我们有时称之为“无守护进程的容器”，这使得可以在不影响正在运行的容器的情况下对 Docker 守护进程进行维护和升级！

在旧模型中，容器运行时逻辑全部实现在守护进程中，关闭守护进程会导致主机上所有运行的容器被杀死。这在生产环境中是一个巨大的问题——特别是考虑到 Docker 的新版本发布频率！每次守护进程升级都会杀死主机上的所有容器——这不好！

幸运的是，这不再是一个问题。

#### 这个 shim 到底是什么？

本章中的一些图表显示了一个 shim 组件。

shim 对于实现无守护进程的容器（就是我们刚刚提到的将运行的容器与守护进程解耦以进行守护进程升级等操作）是至关重要的。

我们之前提到 *containerd* 使用 runc 来创建新的容器。实际上，它为每个创建的容器分叉出一个新的 runc 实例。然而，一旦每个容器被创建，它的父 runc 进程就会退出。这意味着我们可以运行数百个容器而不必运行数百个 runc 实例。

一旦一个容器的父 runc 进程退出，相关的 containerd-shim 进程就成为容器的父进程。shim 作为容器的父进程执行的一些职责包括：

+   保持任何 STDIN 和 STDOUT 流保持打开，这样当守护进程重新启动时，容器不会因为管道关闭而终止等。

+   向守护进程报告容器的退出状态。

#### 在 Linux 上的实现方式

在 Linux 系统上，我们讨论过的组件被实现为以下独立的二进制文件：

+   `dockerd`（Docker 守护进程）

+   `docker-containerd`（containerd）

+   `docker-containerd-shim`（shim）

+   `docker-runc`（runc）

你可以通过在 Docker 主机上运行 `ps` 命令来在 Linux 系统上看到所有这些。显然，当系统有运行的容器时，其中一些将会存在。

#### 那么守护进程的目的是什么

当守护进程中剥离了所有的执行和运行时代码，你可能会问这个问题：“守护进程中还剩下什么？”。

显然，随着越来越多的功能被剥离和模块化，这个问题的答案会随着时间的推移而改变。然而，在撰写本文时，仍然存在于守护进程中的一些主要功能包括：镜像管理、镜像构建、REST API、身份验证、安全性、核心网络和编排。

### 章节总结

Docker 引擎在设计上是模块化的，并且严重依赖于 OCI 的开放标准。

*Docker 守护进程* 实现了 Docker API，这是一个丰富、版本化的 HTTP API，它是随着 Docker 项目的其余部分一起发展的。

容器执行由 *containerd* 处理。containerd 是由 Docker, Inc. 编写并贡献给 CNCF 的。你可以把它看作是一个处理容器生命周期操作的容器监督程序。它小巧轻便，可以被其他项目和第三方工具使用。例如，它被认为将成为 Kubernetes 中默认和最常见的容器运行时。

containerd 需要与符合 OCI 标准的容器运行时进行通信，以实际创建容器。默认情况下，Docker 使用 *runc* 作为其默认的容器运行时。runc 是 OCI 容器运行时规范的事实实现，并且期望从符合 OCI 标准的捆绑包中启动容器。containerd 与 runc 进行通信，并确保 Docker 镜像以符合 OCI 标准的捆绑包的形式呈现给 runc。

runc 可以作为一个独立的 CLI 工具来创建容器。它基于 libcontainer 的代码，并且也可以被其他项目和第三方工具使用。

Docker 守护程序中仍然实现了许多功能。随着时间的推移，这些功能可能会进一步拆分。目前仍然包含在 Docker 守护程序中的功能包括但不限于：API、镜像管理、身份验证、安全功能、核心网络和卷管理。

模块化 Docker 引擎的工作正在进行中。
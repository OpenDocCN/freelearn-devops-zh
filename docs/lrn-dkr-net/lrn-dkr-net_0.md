# 前言

本书帮助读者学习、创建、部署和提供 Docker 网络的管理步骤。Docker 是一个 Linux 容器实现，可以创建轻量级便携的开发和生产环境。这些环境可以进行增量更新。Docker 通过利用 cgroups 和 Linux 命名空间等封装原则以及基于覆盖文件系统的便携式镜像来实现这一点。

Docker 提供了网络原语，允许管理员指定不同容器如何与每个应用程序进行网络连接，连接到它们的各个组件，然后将它们分布在大量服务器上，并确保它们之间的协调，无论它们运行在哪个主机或虚拟机上。本书汇集了所有最新的 Docker 网络技术，并提供了深入的设置详细说明。

# 第一章《Docker 网络入门》解释了 Docker 网络的基本组件，这些组件是从简单的 Docker 抽象和强大的网络组件（如 Linux 桥接、Open vSwitch 等）演变而来。本章还解释了 Docker 容器可以以各种模式创建。在默认模式下，端口映射通过使用 iptables NAT 规则帮助我们，允许到达主机的流量到达容器。本章后面还涵盖了容器的基本链接以及下一代 Docker 网络——libnetwork。

本书内容

第二章《Docker 网络内部》讨论了 Docker 的内部网络架构。我们将了解 Docker 中的 IPv4、IPv6 和 DNS 配置。本章后面还涵盖了 Docker 桥接和单主机和多主机之间容器之间的通信。本章还解释了覆盖隧道和 Docker 网络上实现的不同方法，如 OVS、Flannel 和 Weave。

第三章，“构建您的第一个 Docker 网络”，展示了 Docker 容器如何使用不同的网络选项从多个主机进行通信，例如 Weave、OVS 和 Flannel。Pipework 使用传统的 Linux 桥接，Weave 创建虚拟网络，OVS 使用 GRE 隧道技术，Flannel 为每个主机提供单独的子网，以连接多个主机上的容器。一些实现，如 Pipework，是传统的，并将在一段时间内变得过时，而其他一些则设计用于特定操作系统的上下文中，例如 Flannel 与 CoreOS。本章还涵盖了 Docker 网络选项的基本比较。

第四章，“Docker 集群中的网络”，深入解释了 Docker 网络，使用各种框架，如原生 Docker Swarm，使用 libnetwork 或开箱即用的覆盖网络，Swarm 提供了多主机网络功能。另一方面，Kubernetes 与 Docker 有不同的观点，其中每个 pod 将获得一个唯一的 IP 地址，并且可以借助服务在 pod 之间进行通信。使用 Open vSwitch 或 IP 转发高级路由规则，可以增强 Kubernetes 网络，以在不同子网的主机之间提供连接，并将 pod 暴露给外部世界。在 Mesosphere 的情况下，我们可以看到 Marathon 被用作部署容器的网络后端。在 Mesosphere 的 DCOS 情况下，整个部署的机器堆栈被视为一个机器，以在部署的容器服务之间提供丰富的网络体验。

第五章，“Docker 容器的安全性和 QoS”，通过引用内核和 cgroups 命名空间，深入探讨 Docker 安全性。我们还将讨论文件系统和各种 Linux 功能的一些方面，容器利用这些功能来提供更多功能，例如特权容器，但以暴露更多威胁为代价。我们还将看到如何在 AWS ECS 中部署容器在安全环境中使用代理容器来限制易受攻击的流量。我们还将讨论 AppArmor 提供了丰富的强制访问控制（MAC）系统，提供内核增强功能，以限制应用程序对有限资源的访问。利用它们对 Docker 容器的好处有助于在安全环境中部署它们。在最后一节中，我们将快速深入研究 Docker 安全基准和一些在审核和在生产环境中部署 Docker 时可以遵循的重要建议。

第六章，“Docker 的下一代网络堆栈：libnetwork”，将深入探讨 Docker 网络的一些更深层次和概念性方面。其中之一是 libnetworking——Docker 网络模型的未来，随着 Docker 1.9 版本的发布已经初具规模。在解释 libnetworking 概念的同时，我们还将研究 CNM 模型，以及其各种对象和组件，以及其实现代码片段。接下来，我们将详细研究 CNM 的驱动程序，主要是覆盖驱动程序，并作为 Vagrant 设置的一部分进行部署。我们还将研究容器与覆盖网络的独立集成，以及 Docker Swarm 和 Docker Machine。在接下来的部分中，我们将解释 CNI 接口，其可执行插件，并提供使用 CNI 插件配置 Docker 网络的教程。在最后一节中，将详细解释 Project Calico，该项目提供基于 libnetwork 的可扩展网络解决方案，并与 Docker、Kubernetes、Mesos、裸机和虚拟机集成。

# 本书所需内容

基本上所有的设置都需要 Ubuntu 14.04（安装在物理机器上或作为虚拟机）和 Docker 1.9，这是截止目前的最新版本。如果需要，会在每个设置之前提到特定的操作系统和软件要求（开源 Git 项目）。

# 这本书适合谁

如果您是一名 Linux 管理员，想要通过使用 Docker 来学习网络知识，以确保对核心元素和应用程序进行高效管理，那么这本书适合您。假定您具有 LXC/Docker 的基础知识。

# 约定

您还会发现一些文本样式，用于区分不同类型的信息。以下是一些这些样式的示例和它们的含义解释。

文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 用户名会显示如下：“您可能会注意到，我们使用了 Unix 命令`rm`来删除`Drush`目录，而不是 DOS 的`del`命令。”

代码块设置如下：

```
# * Fine Tuning
#
key_buffer = 16M
key_buffer_size = 32M
max_allowed_packet = 16M
thread_stack = 512K
thread_cache_size = 8
max_connections = 300
```

当我们希望引起您对代码块的特定部分的注意时，相关的行或项目会以粗体显示：

```
# * Fine Tuning
#
key_buffer = 16M
key_buffer_size = 32M
max_allowed_packet = 16M
thread_stack = 512K
thread_cache_size = 8
max_connections = 300
```

任何命令行输入或输出都会以以下形式书写：

```
cd /ProgramData/Propeople
rm -r Drush
git clone --branch master http://git.drupal.org/project/drush.git

```

**新术语**和**重要单词**会以粗体显示。您在屏幕上看到的单词，比如菜单或对话框中的单词，会以这样的形式出现在文本中：“在**选择目标位置**屏幕上，点击**下一步**以接受默认目标。”

### 注意

警告或重要提示会出现在这样的框中。

### 提示

技巧和窍门会以这样的形式出现。

# 第四章：OpenStack 中的容器化

本章首先解释了 OpenStack 中容器的需求。然后，它还解释了 OpenStack 内部支持容器的不同过程。

容器是一个非常热门的话题。用户希望在虚拟机上运行他们的生产工作负载。它们之所以受欢迎，是因为以下原因：

+   容器使用包装概念提供不可变的基础架构模型

+   使用容器轻松开发和运行微服务

+   它们促进了应用程序的更快开发和测试

Linux 内核多年来一直支持容器。微软最近也开始支持 Windows Server 容器和 Hyper-V 容器。随着时间的推移，容器和 OpenStack 对容器的支持也在不断发展。OpenStack 提供 API 来管理数据中心内的容器及其编排引擎。

在本章中，我们将讨论 OpenStack 和容器如何结合在一起。本章涵盖以下主题：

+   OpenStack 中容器的需求

+   OpenStack 社区内支持容器的努力

# OpenStack 中容器的需求

许多组织使用 OpenStack。云基础设施供应商称 OpenStack 为维护私有云但具有公共云可扩展性和灵活性的开源替代品。OpenStack 在基于 Linux 的基础设施即服务（IaaS）方面很受欢迎。随着容器的流行，OpenStack 必须提供各种基础设施资源，如计算、网络和存储，以支持容器。开发人员和运营商可以通过提供跨平台 API 来管理虚拟机、容器和裸金属，而不是在其数据中心中创建新的垂直孤立。

# OpenStack 社区内支持容器的努力

OpenStack 提供以下功能：

+   计算资源

+   多租户安全和隔离

+   管理和监控

+   存储和网络

前面提到的服务对于任何云/数据中心管理工具都是必需的，无论使用哪种容器、虚拟机或裸金属服务器。容器补充了现有技术并带来了一系列新的好处。OpenStack 提供支持在裸金属或虚拟机上运行容器的支持。

在 OpenStack 中，以下项目已经采取了主动行动或提供了对容器和相关技术的支持。

# Nova

Nova 是 OpenStack 的计算服务。Nova 提供 API 来管理虚拟机。Nova 支持使用两个库（即 LXC 和 OpenVZ（Virtuozzo））来提供机器容器的配置。这些与容器相关的库由 libvirt 支持，Nova 使用它来管理虚拟机。

# Heat

Heat 是 OpenStack 的编排服务。自 OpenStack 的 Icehouse 版本以来，Heat 已经支持了对 Docker 容器的编排。用户需要在 Heat 中启用 Docker 编排插件才能使用这个功能。

# Magnum

Magnum 是 OpenStack 的容器基础设施管理服务。Magnum 提供 API 来在 OpenStack 基础设施上部署 Kubernetes、Swarm 和 Mesos 集群。Magnum 使用 Heat 模板在 OpenStack 上部署这些集群。用户可以使用这些集群来运行他们的容器化应用程序。

# Zun

Zun 是 OpenStack 的容器管理服务。Zun 提供 API 来管理 OpenStack 云中容器的生命周期。目前，Zun 支持在裸金属上运行容器，但在将来，它可能会支持在 Nova 创建的虚拟机上运行容器。Zun 使用 Kuryr 为容器提供 neutron 网络。Zun 使用 Cinder 为容器提供持久存储。

# Kuryr

Kuryr 是一个 Docker 网络插件，使用 Neutron 为 Docker 容器提供网络服务。

# Kolla

Kolla 是一个项目，它在 Docker 容器中部署 OpenStack 控制器平面服务。Kolla 通过将每个控制器服务打包为 Docker 容器中的微服务，简化了部署和操作。

# Murano

Murano 是一个为应用开发人员和云管理员提供应用程序目录的 OpenStack 项目，可以在 OpenStack Dashboard（Horizon）中的存储库中发布云就绪应用程序，这些应用程序可以在 Docker 或 Kubernetes 中运行。它为开发人员和运营商提供了控制应用程序完整生命周期的能力。

# Fuxi

Fuxi 是 Docker 容器的存储插件，使容器能够在其中使用 Cinder 卷和 Manila 共享作为持久存储。

# OpenStack-Helm

OpenStack-Helm 是另一个 OpenStack 项目，为运营商和开发人员提供了在 Kubernetes 之上部署 OpenStack 的框架。

# 总结

在本章中，我们了解了为什么 OpenStack 应该支持容器。我们还看了 OpenStack 社区为支持容器所做的努力。

在下一章中，我们将详细了解 Magnum（OpenStack 中的容器基础设施管理服务）。我们还将使用 Magnum 在 OpenStack 中进行一些 COE 管理的实践练习。

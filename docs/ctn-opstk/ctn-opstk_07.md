# 第七章：Kuryr - OpenStack 网络的容器插件

在本章中，我们将学习关于 Kuryr 的内容，这是一个用于容器网络的 OpenStack 项目。本章将涵盖以下主题：

+   介绍 Kuryr

+   Kuryr 架构

+   安装 Kuryr

+   步骤

# 介绍 Kuryr

Kuryr 是捷克语单词，意思是快递员。它是一个 Docker 网络插件，使用 OpenStack Neutron 为 Docker 容器提供网络服务。它将容器网络抽象映射到 OpenStack neutron API。这提供了将虚拟机、容器和裸金属服务器连接到同一虚拟网络的能力，实现无缝的管理体验，并为所有三者提供一致的网络。Kuryr 可以使用 Python 包或 Kolla 容器进行部署。它为使用 neutron 作为提供者的容器提供以下功能：

+   安全组

+   子网池

+   NAT（SNAT/DNAT，浮动 IP）

+   端口安全（ARP 欺骗）

+   服务质量（QoS）

+   配额管理

+   Neutron 可插拔 IPAM

+   通过 neutron 实现良好集成的 COE 负载平衡

+   用于容器的 FWaaS

# Kuryr 架构

在接下来的章节中，我们将看一下 Kuryr 的架构。

# 将 Docker libnetwork 映射到 neutron API

以下图表显示了 Kuryr 的架构，将 Docker libnetwork 网络模型映射到 neutron API。Kuryr 映射**libnetwork** API 并在 neutron 中创建适当的资源，这就解释了为什么**Neutron API**也可以用于容器网络：

![](img/00022.jpeg)

# 提供通用的 VIF 绑定基础设施

Kuryr 为各种端口类型提供了通用的 VIF 绑定机制，这些端口类型将从 Docker 命名空间接收并根据其类型附加到网络解决方案基础设施，例如**Linux 桥接口端口**，**Open vSwitch 端口**，**Midonet 端口**等。以下图表表示了这一点：

![](img/00023.jpeg)

# 提供 neutron 插件的容器化镜像

Kuryr 旨在提供与 Kolla 集成的各种 neutron 插件的容器化镜像。

# 嵌套虚拟机和 Magnum 用例

Kuryr 在容器网络方面解决了 Magnum 项目的用例，并作为 Magnum 或任何其他需要通过 neutron API 利用容器网络的 OpenStack 项目的统一接口。在这方面，Kuryr 利用支持 VM 嵌套容器用例的 neutron 插件，并增强 neutron API 以支持这些用例（例如，OVN）。

# Kuryr 的安装

在本节中，我们将看到如何安装 Kuryr。先决条件如下：

+   KeyStone

+   Neutron

+   DB 管理系统，如 MySQL 或 MariaDB（用于 neutron 和 KeyStone）

+   您选择的供应商的 Neutron 代理

+   如果您选择的 neutron 代理需要，Rabbitmq

+   Docker 1.9+

以下步骤在 Docker 容器内运行 Kuryr：

1.  拉取上游 Kuryr libnetwork Docker 镜像：

[PRE0]

1.  准备 Docker 以找到 Kuryr 驱动程序：

[PRE1]

1.  启动 Kuryr 容器：

[PRE2]

这里：

+   `SERVICE_USER`，`SERVICE_PROJECT_NAME`，`SERVICE_PASSWORD`，`SERVICE_DOMAIN_NAME`和`USER_DOMAIN_NAME`是 OpenStack 凭据

+   `IDENTITY_URL`是指向 OpenStack KeyStone v3 端点的 URL

+   创建卷，以便日志在主机上可用

+   为了在主机命名空间上执行网络操作，例如`ovs-vsctl`，需要给予`NET_ADMIN`权限

# 演练

Kuryr 存在于运行容器并为 libnetwork 远程网络驱动程序提供所需的 API 的每个主机中。

以下是执行创建由 neutron 提供的容器网络的步骤：

1.  用户向 libnetwork 发送请求，以使用 Kuryr 作为网络驱动程序指定符创建 Docker 网络。以下示例创建名为 bar 的 Docker 网络：

[PRE3]

1.  libnetwork 调用 Kuryr 插件创建网络

1.  Kuryr 将调用转发给 Neutron，Neutron 使用 Kuryr 提供的输入数据创建网络

1.  收到来自 neutron 的响应后，它准备输出并将其发送到 libnetwork

1.  libnetwork 将响应存储到其键/值数据存储后端

1.  用户可以使用先前创建的网络启动容器：

[PRE4]

# 总结

在本章中，我们了解了 Kuryr。我们了解了 Kuryr 是什么，它的架构以及安装过程。我们还看了用户使用 Kuryr 作为网络驱动程序创建 Docker 网络时的整体工作流程。

下一章将重点介绍 Murano 项目。我们将了解 Murano 及其架构，并完成实际操作练习。

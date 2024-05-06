# 第十章：Swarm 和云

在本书中，我们使用了 Docker Swarm 在一系列不同的底层技术上，到目前为止，我们还没有深入探讨这一含义：我们在 AWS、DigitalOcean 以及我们的本地工作站上运行了 Swarm。对于测试和分期目的，我们运行 Swarm 的平台可能是次要的（*让我们用 Docker Machine 启动一些 AWS 实例并以这种方式工作*），但对于生产环境来说，了解利弊、原因、评估和跟随趋势是必不可少的。

在本章中，我们将审查几种公共和私有云选项和技术以及它们可能的交集。最后，我们将在第十一章中讨论**CaaS**（容器即服务）和**IaaC**（基础设施即代码）这两个全新的热词，*接下来是什么？*

主要我们将关注：

+   Docker for AWS 和 Azure

+   Docker 数据中心

+   在 OpenStack 上的 Swarm

# Docker for AWS 和 Azure

与 Docker For Mac 和 Windows 一样，Docker 团队开始着手开发*新一代*的运维工具集：Docker for AWS 和 Docker for Windows。这些工具旨在为部署 Docker 基础架构提供自动化支持，特别是适用于 Swarm 的基础架构。

目标是提供一种标准的做事方式，将底层基础设施与 Docker 工具集集成，并让人们毫不费力地在他们喜爱的平台上运行最新的软件版本。最终目标实际上是让开发人员将东西从他们的笔记本电脑上移动到云端，使用 Docker for Mac/Windows 和 Docker for AWS/Azure。

## Docker for AWS

用户体验在 Docker 生态系统中一如既往地出色。要求如下：

+   一个 AWS ID

+   导入到 AWS 密钥环中的 SSH 密钥

+   准备好的安全组

基本上，Docker for AWS 是 CloudForms 的可点击模板。CloudForms 是 AWS 的编排系统，允许创建复杂系统的模板，例如，您可以指定由三个 Web 服务器、一个数据库和一个负载均衡器组成的 Web 基础架构。

Docker for AWS 当然具备创建 Docker Swarm（模式）基础架构的能力，它会根据您的指定创建尽可能多的主节点和工作节点，放置一个负载均衡器，并相应地配置所有网络。

这是欢迎界面：

![Docker for AWS](img/image_10_001.jpg)

然后，您可以指定一些基本和高级选项：

![Docker for AWS](img/image_10_002.jpg)

如您所见，您可以选择管理者和工作节点的数量，以及要启动的实例的类型。到目前为止，支持最多 1000 个工作节点。之后，您只需在下一步中点击创建堆栈，并等待几分钟让 CloudForms 启动基础架构。

模板的作用是：

1.  在您的 AWS 帐户中创建一个新的虚拟私有云，包括网络和子网。

1.  创建两个自动扩展组，一个用于管理者，一个用于工作节点。

1.  启动管理者并确保它们与 Raft quorum 达成健康状态。

1.  逐个启动和注册工作节点到 Swarm。

1.  创建**弹性负载均衡器**（**ELBs**）来路由流量。

1.  完成

一旦 CloudFormation 完成，它将提示一个绿色的确认。

![Docker for AWS](img/image_10_003.jpg)

现在，我们准备好进入我们的新 Docker Swarm 基础架构。只需选择一个管理者的公共 IP 并使用在第一步中指定的 SSH 密钥连接到它：

```
 ssh docker@ec2-52-91-75-252.compute-1.amazonaws.com

```

![Docker for AWS](img/image_10_004.jpg)

## Docker for Azure

由于与微软的协议，Azure 也可以作为一键式体验（或几乎是）自动部署 Swarm。

在 Azure 上部署 Swarm 的先决条件是：

+   拥有有效的 Azure 帐户

+   将此帐户 ID 与 Docker for Azure 关联。

+   活动目录主体应用程序 ID

要生成最后一个，您可以方便地使用一个 docker 镜像，并使用以下命令启动它：

```
 docker run -it docker4x/create-sp-azure docker-swarm

```

在过程中，您将需要通过浏览器登录到指定的 URL。最后，将为您提供一对 ID/密钥，供您在 Azure 向导表单中输入。

![Docker for Azure](img/image_10_005.jpg)

一切就绪后，您只需点击**OK**和**Create**。

![Docker for Azure](img/image_10_006.jpg)

将创建一组经典虚拟机来运行指定数量的管理者（这里是 1）和工作节点（这里是 4），以及适当的内部网络、负载均衡器和路由器。就像在 Docker for AWS 中一样，您可以通过 SSH 连接到一个管理者的公共 IP 来开始使用部署的 Swarm：

```
 ssh docker@52.169.125.191

```

![Docker for Azure](img/image_10_007.jpg)

目前 Azure 模板有一个限制，它只支持一个管理者。然而，很快就会有添加新管理者的可能性。

# Docker 数据中心

Docker 数据中心，以前是 Tutum，被 Docker 收购，是 Docker 提供的单击部署解决方案，用于使用 UCP，Universal Control Panel，Docker 的商业和企业产品。

Docker 数据中心包括：

+   **Universal Control Plane**（**UCP**），UI，请参阅[`docs.docker.com/ucp/overview`](https://docs.docker.com/ucp/overview)

+   **Docker Trusted Registry (DTR)**，私有注册表，请参阅[`docs.docker.com/docker-trusted-registry`](https://docs.docker.com/docker-trusted-registry)

在 Dockercon 16 上，团队发布了对在 AWS 和 Azure 上运行 Docker 数据中心的支持（目前处于 Beta 阶段）。要尝试 Docker 数据中心，你需要将许可证与你的公司/项目的 AWS 或 Azure ID 关联起来。

对于 AWS 的数据中心，就像对于 AWS 的 Docker 一样，有一个 CloudFormation 模板，可以立即启动一个 Docker 数据中心。要求是：

+   至少有一个配置了 Route53 的，AWS 的 DNS 服务，请参阅[`docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html`](http://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)

+   一个 Docker 数据中心许可证

你需要做的就是从你的许可证链接进入创建堆栈页面。在这里，你只需输入**HostedZone** ID 和 Docker 数据中心许可证，然后开始创建堆栈。在内部，Docker 数据中心在私有网络（节点）上放置一些虚拟机，并且一些通过弹性负载均衡器（ELBs，用于控制器）进行负载平衡，上面安装了商业支持的引擎版本。当前版本的 Docker 数据中心虚拟机在内部运行 Swarm 独立和发现机制，以相互连接。我们可以预期数据中心的稳定版本很快就会发布。

Docker 数据中心和 Docker for AWS 之间的主要区别在于前者旨在成为一体化的企业就绪解决方案。而后者是部署 Swarm 集群的最快方式，前者是更完整的解决方案，具有时尚的 UI，Notary 和来自生态系统的可选服务。

# 在 OpenStack 上的 Swarm

说到私有云，最受欢迎的 IaaS 开源解决方案是 OpenStack。OpenStack 是一个由程序（以前称为项目）组成的庞大生态系统，旨在提供所谓的云操作系统。核心 OpenStack 程序包括：

+   **Keystone**：身份和授权系统

+   **Nova**：虚拟机抽象层。Nova 可以与虚拟化模块（如 Libvirt、VMware）进行插接

+   **Neutron**：处理租户网络、实例端口、路由和流量的网络模块

+   **Cinder**：负责处理卷的存储模块

+   **Glance**：镜像存储

一切都由额外的参与者粘合在一起：

+   一个数据库系统，比如 MySQL，保存配置。

+   一个 AMQP 代理，比如 Rabbit，用于排队和传递操作

+   一个代理系统，比如 HAproxy，用来代理 HTTP API 请求

在 OpenStack 中典型的 VM 创建中，发生以下情况：

1.  用户可以从 UI（Horizon）或 CLI 决定生成一个 VM。

1.  他/她点击一个按钮或输入一个命令，比如`nova boot ...`

1.  Keystone 检查用户在他/她的租户中的授权和认证，通过在用户的数据库或 LDAP 中检查（取决于 OpenStack 的配置），并生成一个将在整个会话中使用的令牌：“这是你的令牌：gAAAAABX78ldEiY2”*.*

1.  如果认证成功并且用户被授权生成 VM，Nova 将使用授权令牌调用：“我们正在启动一个 VM，你能找到一个合适的物理主机吗？”

1.  如果这样的主机存在，Nova 从 Glance 获取用户选择的镜像：“Glance，请给我一个 Ubuntu Xenial 可启动的 qcow2 文件”

1.  在物理启动 VM 的计算主机上，一个`nova-compute`进程，它与配置的插件进行通信，例如，对 Libvirt 说：“我们正在这个主机上启动一个 VM”

1.  Neutron 为 VM 分配私有（如果需要还有公共）网络端口：“请在指定的网络上创建这些端口，在这些子网池中”

1.  如果用户愿意，Cinder 会在其调度程序设计的主机上分配卷。也就是说。让我们创建额外的卷，并将它们附加到 VM。

1.  如果使用 KVM，将生成一个包含上述所有信息的适当 XML，并且 Libvirt 在计算主机上启动 VM

1.  当 VM 启动时，通过 cloud-init 注入一些变量，例如，允许无密码 SSH 登录的 SSH 密钥

这（除了 Cinder 上的第 8 步）正是 Docker Machine 的 OpenStack 驱动程序的行为方式：当你使用`-d openstack`在 Machine 上创建一个 Docker 主机时，你必须指定一个现有的 glance 镜像，一个现有的私有（和可选的公共）网络，并且（可选的，否则会自动生成）指定一个存储在 Nova 数据库中的 SSH 镜像。当然，你必须将授权变量传递给你的 OpenStack 环境，或者作为导出的 shell 变量源它们。

在 OpenStack 上创建一个 Docker 主机的 Machine 命令将如下所示：

```
 docker-machine create \
 --driver openstack \
 --openstack-image-id 98011e9a-fc46-45b6-ab2c-cf6c43263a22 \
 --openstack-flavor-id 3 \
 --openstack-floatingip-pool public \
 --openstack-net-id 44ead515-da4b-443b-85cc-a5d13e06ddc85 \
 --openstack-sec-groups machine \
 --openstack-ssh-user ubuntu \
 ubuntu1

```

## OpenStack Nova

因此，在 OpenStack 上进行 Docker Swarm 的经典方式将是开始创建实例，比如从 Ubuntu 16.04 镜像中创建 10 个 VMs，放在一个专用网络中：

+   从 Web UI 中，指定 10 作为实例的数量

+   或者从 CLI 中，使用`nova boot ... --max-count 10 machine-`

+   或者使用 Docker Machine

最后一种方法更有前途，因为 Machine 会自动安装 Docker，而不必在新创建的实例上进行后续的黑客攻击或使用其他工具（例如使用通用驱动程序的 Machine，Belt，Ansible，Salt 或其他脚本）。但在撰写本文时（Machine 0.8.2），Machine 不支持批量主机创建，因此您将不得不使用一些基本的 shell 逻辑循环`docker-machine`命令：

```
 #!/bin/bash
 for i in `seq 0 9`; do
 docker-machine create -d openstack ... openstack-machine-$i
 done

```

这根本不是一个好的用户体验，因为当我们谈论数十个主机时，机器的扩展性仍然非常糟糕。

### （不推荐使用的）nova-docker 驱动程序

曾经，有一个用于 Nova 的驱动程序，可以将 Docker 容器作为 Nova 的最终目的地（而不是创建 KVM 或 VmWare 虚拟机，例如，这些驱动程序允许从 Nova 创建和管理 Docker 容器）。如果对于*旧*的 Swarm 来说使用这样的工具是有意义的（因为一切都被编排为容器），那么对于 Swarm Mode 来说就没有兴趣了，它需要的是 Docker 主机而不是裸露的容器。

## 现实 - OpenStack 友好的方式

幸运的是，OpenStack 是一个非常充满活力的项目，现在它已经发布了**O**（**Ocata**），它通过许多可选模块得到了丰富。从 Docker Swarm 的角度来看，最有趣的是：

+   **Heat:** 这是一个编排系统，可以从模板创建 VMs 配置。

+   **Murano:** 这是一个应用程序目录，可以从由开源社区维护的目录中运行应用程序，包括 Docker 和 Kubernetes 容器。

+   **Magnum:** 这是来自 Rackspace 的容器即服务解决方案。

+   **Kuryr：** 这是网络抽象层。使用 Kuryr，您可以将 Neutron 租户网络和使用 Docker Libnetwork 创建的 Docker 网络（比如 Swarm 网络）连接起来，并将 OpenStack 实例与 Docker 容器连接起来，就好像它们连接到同一个网络一样。

## OpenStack Heat

OpenStack Heat 有点类似于 Docker Compose，允许您通过模板启动系统，但它更加强大：您不仅可以从镜像启动一组实例，比如 Ubuntu 16.04，还可以对它们进行编排，这意味着创建网络，将 VM 接口连接到网络，放置负载均衡器，并在实例上执行后续任务，比如安装 Docker。粗略地说，Heat 相当于 OpenStack 的 Amazon CloudFormation。

在 Heat 中，一切都始于 YAML 模板，借助它，您可以在启动之前对基础架构进行建模，就像您使用 Compose 一样。例如，您可以创建一个这样的模板文件：

```
 ...
 resources:
   dockerhosts_group:
     type: OS::Heat::ResourceGroup
     properties:
       count: 10
       resource_def:
         type: OS::Nova::Server
         properties:
           # create a unique name for each server
           # using its index in the group
           name: docker_host_%index%
           image: Ubuntu 16.04
           flavor: m.large
 ...

```

然后，您可以从中启动一个堆栈（`heat stack-create -f configuration.hot dockerhosts`）。Heat 将调用 Nova、Neutron、Cinder 和所有必要的 OpenStack 服务来编排资源并使其可用。

在这里，我们不会展示如何通过 Heat 启动 Docker Swarm 基础架构，而是会看到 Magnum，它在底层使用 Heat 来操作 OpenStack 对象。

## OpenStack Magnum

Magnum 于 2015 年底宣布，并由 OpenStack 容器团队开发，旨在将**容器编排引擎**（**COEs**）如 Docker Swarm 和**Kubernetes**作为 OpenStack 中的一流资源。在 OpenStack 领域内有许多项目专注于提供容器支持，但 Magnum 走得更远，因为它旨在支持*容器编排*，而不仅仅是裸露的容器管理。

![OpenStack Magnum](img/image_10_008.jpg)

到目前为止，重点特别放在 Kubernetes 上，但我们在这里谈论**Magnum**，因为它是提供在私有云上运行 CaaS 编排的最有前途的开源技术。在撰写本文时，Magnum 尚不支持最新的 Swarm Mode：这个功能必须得到解决。作者已经在 Launchpad 蓝图上开放了一个问题，可能会在书出版后开始着手处理：[`blueprints.launchpad.net/magnum/+spec/swarm-mode-support`](https://blueprints.launchpad.net/magnum/+spec/swarm-mode-support)。

### 架构和核心概念

Magnum 有两个主要组件，运行在控制节点上：

```
 magnum-api
 magnum-conductor

```

第一个进程`magnum-api`是典型的 OpenStack API 提供者，由 magnum Python 客户端或其他进程调用进行操作，例如创建集群。后者`magnum-conductor`由`magnum-api`（或多或少具有`nova-conductor`的相同功能）通过 AMQP 服务器（如 Rabbit）调用，其目标是与 Kubernetes 或 Docker API 进行接口。实际上，这两个二进制文件一起工作，提供一种编排抽象。

![架构和核心概念](img/image_10_009.jpg)

在 OpenStack 集群计算节点上，除了`nova-compute`进程外，不需要运行任何特殊的东西：Magnum conductor 直接利用 Heat 创建堆栈，这些堆栈创建网络并在 Nova 中实例化 VM。

Magnum 术语随着项目的发展而不断演变。但这些是主要概念：

+   **容器**是 Docker 容器。

+   **集群**（以前是 Bay）是一组调度工作的节点对象的集合，例如 Swarm 节点。

+   **ClusterTemplate**（以前是 BayModel）是存储有关集群类型信息的模板。例如，ClusterTemplate 定义了*具有 3 个管理节点和 5 个工作节点的 Swarm 集群*。

+   **Pods**是在同一物理或虚拟机上运行的一组容器。

至于高级选项，如存储、新的 COE 支持和扩展，Magnum 是一个非常活跃的项目，我们建议您在[`docs.openstack.org/developer/magnum/`](http://docs.openstack.org/developer/magnum/)上关注其发展。

### 在 Mirantis OpenStack 上安装 HA Magnum

安装 Magnum 并不那么简单，特别是如果您想要保证 OpenStack HA 部署的一些故障转移。互联网上有许多关于如何在 DevStack（开发者的 1 节点暂存设置）中配置 Magnum 的教程，但没有一个显示如何在具有多个控制器的真实生产系统上工作。在这里，我们展示了如何在真实设置中安装 Magnum。

通常，生产 OpenStack 安装会有一些专门用于不同目标的节点。在最小的 HA 部署中，通常有：

+   三个或更多（为了仲裁原因是奇数）**控制节点**，负责托管 OpenStack 程序的 API 和配置服务，如 Rabbit、MySQL 和 HAproxy

+   任意数量的**计算节点**，在这里工作负载在物理上运行（VM 托管的地方）

还可以选择专用存储、监控、数据库、网络和其他节点。

在我们的设置中，基于运行 Newton 的**Mirantis OpenStack**，安装了 Heat，我们有三个控制器和三个计算加存储节点。使用 Pacemaker 配置了 HA，它将 MySQL、Rabbitmq 和 HAproxy 等资源保持高可用性。API 由 HAproxy 代理。这是一个显示配置到 Pacemaker 中的资源的屏幕截图。它们都已启动并正常工作：

![在 Mirantis OpenStack 上安装 HA Magnum](img/image_10_010.jpg)

集群中的所有节点都运行 Ubuntu 16.04（Xenial），稳定的 Magnum 2.0 软件包存在，因此只需从上游使用它们并使用`apt-get install`进行安装。

然而，在安装 Magnum 之前，有必要准备环境。首先需要一个数据库。只需在任何控制器上输入 MySQL 控制台：

```
 node-1# mysql

```

在 MySQL 中，创建 magnum 数据库和用户，并授予正确的权限：

```
 CREATE DATABASE magnum;
 GRANT ALL PRIVILEGES ON magnum.* TO 'magnum'@'controller' \
   IDENTIFIED BY 'password';
 GRANT ALL PRIVILEGES ON magnum.* TO 'magnum'@'%' \
   IDENTIFIED BY 'password';

```

现在，有必要在 Keystone 中创建服务凭据，首先要定义一个 magnum OpenStack 用户，必须将其添加到服务组中。服务组是一个特殊的组，其中包括在集群中运行的 OpenStack 服务，如 Nova、Neutron 等。

```
 openstack user create --domain default --password-prompt magnum
 openstack role add --project services --user magnum admin

```

之后，必须创建一个新的服务：

```
 openstack service create --name magnum \   --description "OpenStack 
    Container Infrastructure" \   container-infra

```

OpenStack 程序通过其 API 调用并进行通信。API 通过端点访问，这是一个 URL 和端口的配对，在 HA 设置中由 HAproxy 代理。在我们的设置中，HAproxy 在`10.21.22.2`上接收 HTTP 请求，并在控制器 IP 之间进行负载均衡，即`10.21.22.4, 5`和`6`。

![在 Mirantis OpenStack 上安装 HA Magnum](img/image_10_011.jpg)

我们必须为 Magnum 创建这样的端点，默认情况下监听端口 9511，对于每个区域（公共、内部和管理员）：

```
 openstack endpoint create --region RegionOne \
   container-infra public http://10.21.22.2:9511/v1
 openstack endpoint create --region RegionOne \
   container-infra internal http://10.21.22.2:9511/v1
 openstack endpoint create --region RegionOne \
   container-infra admin http://10.21.22.2:9511/v1

```

此外，Magnum 需要额外的配置来在域内部组织其工作负载，因此必须添加一个专用域和域用户：

```
 openstack domain create --description "Magnum" magnum
 openstack user create --domain magnum --password-prompt 
    magnum_domain_admin
 openstack role add --domain magnum --user magnum_domain_admin admin

```

现在一切就绪，最终运行`apt-get`。在所有三个控制器上运行以下命令，并在 ncurses 界面中，始终选择 No，以不更改环境，或保持默认配置：

```
 apt-get install magnum-api magnum-conductor

```

### 配置 HA Magnum 安装

Magnum 的配置非常简单。使其处于运行状态所需的操作是：

1.  通过`magnum.conf`文件进行配置。

1.  重新启动 magnum 二进制文件。

1.  打开端口`tcp/9511`。

1.  配置 HAproxy 以接受和平衡 magnum APIs。

1.  重新加载 HAproxy。

必须在每个控制器上进行的关键配置如下。首先，在每个控制器上，主机参数应该是管理网络上接口的 IP：

```
 [api]
 host = 10.21.22.6

```

如果未安装**Barbican**（专门用于管理密码等秘密的 OpenStack 项目），则必须由`**x509keypair**`插件处理证书：

```
 [certificates]
 cert_manager_type = x509keypair

```

然后，需要一个数据库连接字符串。在这个 HA 设置中，MySQL 在 VIP`10.21.22.2`上响应：

```
 [database]
 connection=mysql://magnum:password@10.21.22.2/magnum

```

Keystone 身份验证配置如下（选项相当不言自明）：

```
 [keystone_authtoken]
 auth_uri=http://10.21.22.2:5000/
 memcached_servers=10.21.22.4:11211,
    10.21.22.5:11211,10.21.22.6:11211
 auth_type=password
 username=magnum
 project_name=services
 auth_url=http://10.21.22.2:35357/
 password=password
 user_domain_id = default
 project_domain_id = default
 auth_host = 127.0.0.1
 auth_protocol = http
 admin_user = admin
 admin_password =
 admin_tenant_name = admin

```

必须配置 Oslo（消息代理）以进行消息传递：

```
 [oslo_messaging_notifications]
 driver = messaging

```

Rabbitmq 的配置是这样的，指定 Rabbit 集群主机（因为 Rabbit 在控制器上运行，所以所有控制器的管理网络的 IP）：

```
 [oslo_messaging_rabbit]
 rabbit_hosts=10.21.22.6:5673, 10.21.22.4:5673, 10.21.22.5:5673
 rabbit_ha_queues=True
 heartbeat_timeout_threshold=60
 heartbeat_rate=2
 rabbit_userid=magnum
 rabbit_password=A3elbTUIqOcqRihB6XE3MWzN

```

最后，受托人的额外配置如下：

```
 [trust]
 trustee_domain_name = magnum
 trustee_domain_admin_name = magnum_domain_admin
 trustee_domain_admin_password = magnum

```

在进行此重新配置后，必须重新启动 Magnum 服务：

```
 service magnum-api restart
 service magnum-conductor restart

```

Magnum 默认使用端口`tcp/9511`，因此必须在 iptables 中接受到该端口的流量：修改 iptables 以添加此规则：

```
 -A INPUT -s 10.21.22.0/24 -p tcp -m multiport --dports 9511 -m 
    comment --comment "117 magnum-api from 10.21.22.0/24" -j ACCEPT

```

就在其他 OpenStack 服务之后，在`116 openvswitch db`之后。

现在，是时候配置 HAproxy 来接受 magnum 了。在所有控制器的`/etc/haproxy/conf.d`中添加一个`180-magnum.cfg`文件，内容如下：

```
 listen magnum-api
 bind 10.21.22.2:9511
 http-request  set-header X-Forwarded-Proto https if { ssl_fc }
 option  httpchk
 option  httplog
 option  httpclose
 option  http-buffer-request
 timeout  server 600s
 timeout  http-request 10s
 server node-1 10.21.22.6:9511  check inter 10s fastinter 2s 
      downinter 3s rise 3 fall 3
 server node-2 10.21.22.5:9511  check inter 10s fastinter 2s 
      downinter 3s rise 3 fall 3
 server node-3 10.21.22.4:9511  check inter 10s fastinter 2s 
      downinter 3s rise 3 fall 3

```

这将配置 magnum-api 侦听 VIP`10.21.22.2:9511`，支持三个控制器。

紧接着，必须从 Pacemaker 重新启动 HAproxy。从任何控制器上运行：

```
 pcs resource disable p_haproxy

```

等待直到所有控制器上没有运行 HAproxy 进程（您可以使用`ps aux`进行检查），但这应该非常快，不到 1 秒，然后：

```
 pcs resource enable p_haproxy

```

之后，Magnum 将可用并且服务已启动：

```
 source openrc
 magnum service-list

```

![配置 HA Magnum 安装](img/image_10_012.jpg)

### 在 Magnum 上创建一个 Swarm 集群

创建一个 Swarm 集群，当 COE 被添加到 Magnum 时，将需要执行以下步骤：

1.  创建一个 Swarm 模板。

1.  从模板启动一个集群。

我们不会深入研究尚不存在的东西，但命令可能是这样的：

```
 magnum cluster-template-create \
 --name swarm-mode-cluster-template \
 --image-id ubuntu_xenial \
 --keypair-id fuel \
 --fixed-network private \
 --external-network-id public \
 --dns-nameserver 8.8.8.8 \
 --flavor-id m1.medium \
 --docker-volume-size 5 \
 --coe swarm-mode

```

在这里，定义了一个基于 Ubuntu Xenial 的`m1.medium` flavors 的 swarm-mode 类型的集群模板：VM 将注入 fuel 密钥对，将具有额外的外部公共 IP。基于这样一个模板创建集群的 UX 可能会是：

```
 magnum cluster-create --name swarm-mode-cluster \
       --cluster-template swarm-mode-cluster-template \
       --manager-count 3 \
       --node-count 8

```

在这里，使用三个管理节点和五个工作节点实例化了一个 Swarm 集群。

Magnum 是一个很棒的项目，在 OpenStack 上以最高级别的抽象层次运行容器编排。它在 Rackspace 云上运行，并且通过 Carina 可以供公众使用，参见[`blog.rackspace.com/carina-by-rackspace-simplifies-containers-with-easy-to-use-instant-on-native-container-environment`](http://blog.rackspace.com/carina-by-rackspace-simplifies-containers-with-easy-to-use-instant-on-native-container-environment)。

# 总结

在本章中，我们探讨了可以运行 Docker Swarm 集群的替代平台。我们使用了最新的 Docker 工具--Docker for AWS 和 Docker for Azure--并用它们来演示如何以新的方式安装 Swarm。在介绍了 Docker Datacenter 之后，我们转向了私有云部分。我们在 OpenStack 上工作，展示了如何在其上运行 Docker 主机，如何安装 OpenStack Magnum，以及如何在其上创建 Swarm 对象。我们的旅程即将结束。

下一章也是最后一章将勾勒出 Docker 编排的未来。

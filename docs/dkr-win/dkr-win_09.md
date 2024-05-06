# 第七章：使用 Docker Swarm 编排分布式解决方案

您可以在单台 PC 上运行 Docker，这是我在本书中迄今为止所做的，也是您在开发和基本测试环境中使用 Docker 的方式。在更高级的测试环境和生产环境中，单个服务器是不合适的。为了实现高可用性并为您提供扩展解决方案的灵活性，您需要多台作为集群运行的服务器。Docker 在平台中内置了集群支持，您可以使用 Docker Swarm 模式将多个 Docker 主机连接在一起。

到目前为止，您学到的所有概念（镜像、容器、注册表、网络、卷和服务）在集群模式下仍然适用。Docker Swarm 是一个编排层。它提供与独立的 Docker 引擎相同的 API，还具有额外的功能来管理分布式计算的各个方面。当您在集群模式下运行服务时，Docker 会确定在哪些主机上运行容器；它管理不同主机上容器之间的安全通信，并监视主机。如果集群中的服务器宕机，Docker 会安排它正在运行的容器在不同的主机上启动，以维持应用程序的服务水平。

自 2015 年发布的 Docker 1.12 版本以来，集群模式一直可用，并提供经过生产硬化的企业级服务编排。集群中的所有通信都使用相互 TLS 进行安全保护，因此节点之间的网络流量始终是加密的。您可以安全地在集群中存储应用程序机密，并且 Docker 只向那些需要访问的容器提供它们。集群是可扩展的，因此您可以轻松添加节点以增加容量，或者移除节点进行维护。Docker 还可以在集群模式下运行自动滚动服务更新，因此您可以在零停机的情况下升级应用程序。

在本章中，我将设置一个 Docker Swarm，并在多个节点上运行 NerdDinner。我将首先创建单个服务，然后转而从 Compose 文件部署整个堆栈。您将学习以下内容：

+   创建集群和管理节点

+   在集群模式下创建和管理服务

+   在 Docker Swarm 中管理应用程序配置

+   将堆栈部署到 Docker Swarm

+   无停机部署更新

# 技术要求

您需要在 Windows 10 更新 18.09 或 Windows Server 2019 上运行 Docker 才能跟随示例。本章的代码可在[`github.com/sixeyed/docker-on-windows/tree/second-edition/ch07`](https://github.com/sixeyed/docker-on-windows/tree/second-edition/ch07)找到。

# 创建一个群集并管理节点

Docker Swarm 模式使用具有管理者和工作者高可用性的管理者-工作者架构。管理者面向管理员，您可以使用活动管理者来管理集群和运行在集群上的资源。工作者面向用户，并且他们运行您的应用程序服务的容器。

群集管理者也可以运行您的应用程序的容器，这在管理者-工作者架构中是不寻常的。管理小型群集的开销相对较低，因此如果您有 10 个节点，其中 3 个是管理者，管理者也可以运行一部分应用程序工作负载（但在生产环境中，您需要意识到在它们上运行大量应用程序工作负载可能会使管理者计算资源不足的风险）。

您可以在同一个群集中拥有 Windows 和 Linux 节点的混合，这是管理混合工作负载的好方法。建议所有节点运行相同版本的 Docker，但可以是 Docker CE 或 Docker Enterprise——Docker Swarm 功能内置于核心 Docker 引擎中。

许多在生产中运行 Docker 的企业都有一个具有 Linux 节点作为管理者的群集，以及 Windows 和 Linux 节点混合作为工作者。这意味着您可以在单个集群中使用节点操作系统的最低成本选项来运行 Windows 和 Linux 应用程序容器。

# 初始化群集

群集可以是任何规模。您可以在笔记本电脑上运行单节点群集来测试功能，也可以扩展到数千个节点。您可以通过使用`docker swarm init`命令来初始化群集：

```
> docker swarm init --listen-addr 192.168.2.214 --advertise-addr 192.168.2.214
Swarm initialized: current node (jea4p57ajjalioqokvmu82q6y) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-37p6ufk5jku6tndotqlcy1w54grx5tvxb3rxphj8xkdn9lbeml-3w7e8hxfzzpt2fbf340d8phia 192.168.2.214:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

这将创建一个具有单个节点的群集——即您运行命令的 Docker 引擎，并且该节点将成为群集管理器。我的机器有多个 IP 地址，因此我已经指定了`listen-addr`和`advertise-addr`选项，告诉 Docker 要使用哪个网络接口进行群集通信。始终指定 IP 地址并为管理节点使用静态地址是一个良好的做法。

您可以使用内部私有网络来保护您的集群，以便通信不在公共网络上。您甚至可以完全将管理节点保持在公共网络之外。只有具有面向公共的工作负载的工作节点需要连接到公共网络，除了内部网络之外-如果您正在使用负载均衡器作为基础架构的公共入口点，甚至可以避免这种情况。

# 将工作节点添加到集群

`docker swarm init`的输出告诉您如何通过加入其他节点来扩展集群。节点只能属于一个集群，并且要加入，它们需要使用加入令牌。该令牌可以防止恶意节点加入您的集群，如果网络受到损害，因此您需要将其视为安全秘密。节点可以作为工作节点或管理节点加入，并且每个节点都有不同的令牌。您可以使用`docker swarm join-token`命令查看和旋转令牌。

在运行相同版本的 Docker 的第二台机器上，我可以运行`swarm join`命令加入集群：

```
> docker swarm join `
   --token SWMTKN-1-37p6ufk5jku6tndotqlcy1w54grx5tvxb3rxphj8xkdn9lbeml-3w7e8hxfzzpt2fbf340d8phia `
   192.168.2.214:2377 
This node joined a swarm as a worker.
```

现在我的 Docker 主机正在运行在集群模式下，当我连接到管理节点时，我可以使用更多的命令。`docker node`命令管理集群中的节点，因此我可以列出集群中的所有节点，并使用`docker node ls`查看它们的当前状态：

```
> docker node ls
ID    HOSTNAME    STATUS   AVAILABILITY  MANAGER STATUS  ENGINE VERSION
h2ripnp8hvtydewpf5h62ja7u  win2019-02      Ready Active         18.09.2
jea4p57ajjalioqokvmu82q6y * win2019-dev-02 Ready Active Leader  18.09.2
```

`状态`值告诉您节点是否在线在集群中，`可用性`值告诉您节点是否能够运行容器。`管理节点状态`字段有三个选项：

+   `领导者`：控制集群的活跃管理节点。

+   `可达`：备用管理节点；如果当前领导者宕机，它可以成为领导者。

+   `无值`：工作节点。

多个管理节点支持高可用性。Docker Swarm 使用 Raft 协议在当前领导者丢失时选举新领导者，因此具有奇数个管理节点，您的集群可以在硬件故障时生存。对于生产环境，您应该有三个管理节点，这就是您所需要的，即使对于具有数百个工作节点的大型集群也是如此。

工作节点不会自动晋升为管理节点，因此如果所有管理节点丢失，那么您将无法管理集群。在这种情况下，工作节点上的容器继续运行，但没有管理节点来监视工作节点或您正在运行的服务。

# 晋升和删除集群节点

您可以使用`docker node promote`将工作节点转换为管理节点，并使用`docker node demote`将管理节点转换为工作节点；这些是您在管理节点上运行的命令。

要离开 Swarm，您需要在节点本身上运行`docker swarm leave`命令：

```
> docker swarm leave
Node left the swarm.
```

如果您有单节点 Swarm，您可以使用相同的命令退出 Swarm 模式，但是您需要使用`--force`标志，因为这实际上将您从 Swarm 模式切换回单个 Docker Engine 模式。

`docker swarm`和`docker node`命令管理着 Swarm。当您在 Swarm 模式下运行时，您将使用特定于 Swarm 的命令来管理容器工作负载。

您将看到关于*Docker Swarm*和*Swarm 模式*的引用。从技术上讲，它们是不同的东西。Docker Swarm 是一个早期的编排器，后来被构建到 Docker Engine 中作为 Swarm 模式。*经典*的 Docker Swarm 只在 Linux 上运行，因此当您谈论带有 Windows 节点的 Swarm 时，它总是 Swarm 模式，但通常被称为 Docker Swarm。

# 在云中运行 Docker Swarm

Docker 具有最小的基础设施要求，因此您可以轻松在任何云中快速启动 Docker 主机或集群 Docker Swarm。要大规模运行 Windows 容器，您只需要能够运行 Windows Server 虚拟机并将它们连接到网络。

云是运行 Docker 的好地方，而 Docker 是迁移到云的好方法。Docker 为您提供了现代应用程序平台的强大功能，而不受**平台即服务**（**PaaS**）产品的限制。PaaS 选项通常具有专有的部署系统、代码中的专有集成，并且开发人员体验不会使用相同的运行时。

Docker 允许您打包应用程序并以便携方式定义解决方案结构，这样可以在任何机器和任何云上以相同的方式运行。您可以使用所有云提供商支持的基本**基础设施即服务**（**IaaS**）服务，并在每个环境中实现一致的部署、管理和运行时体验。

主要的云还提供托管的容器服务，但这些服务已经集中在 Kubernetes 上——Azure 上的 AKS，Amazon Web Services 上的 EKS 和 Google Cloud 上的 GKE。在撰写本文时，它们都是 100%的 Linux 产品。对于 Kubernetes 的 Windows 支持正在积极开发中，一旦支持，云服务将开始提供 Windows 支持，但 Kubernetes 比 Swarm 更复杂，我不会在这里涵盖它。

在云中部署 Docker Swarm 的最简单方法之一是使用 Terraform，这是一种强大的基础设施即代码（IaC）技术，通常比云提供商自己的模板语言或脚本工具更容易使用。通过几十行配置，您可以定义管理节点和工作节点的虚拟机，以及网络设置、负载均衡器和任何其他所需的服务。

# Docker 认证基础设施

Docker 使用 Terraform 来支持 Docker 认证基础设施（DCI），这是一个用于在主要云提供商和主要本地虚拟化工具上部署 Docker 企业的单一工具。它使用每个提供商的相关服务来设置 Docker 企业平台的企业级部署，包括通用控制平面和 Docker 可信注册表。

DCI 在 Docker 的一系列参考架构指南中有详细介绍，可在 Docker 成功中心（[`success.docker.com`](https://success.docker.com)）找到。这个网站值得收藏，你还可以在那里找到关于现代化传统应用程序的指南，以及有关容器中日志记录、监控、存储和网络的最佳实践文档。

# 在 swarm 模式下创建和管理服务

在上一章中，您看到了如何使用 Docker Compose 来组织分布式解决方案。在 Compose 文件中，您可以使用网络将应用程序的各个部分定义为服务并将它们连接在一起。在 swarm 模式中，使用相同的 Docker Compose 文件格式和相同的服务概念。在 swarm 模式中，构成服务的容器被称为副本。您可以使用 Docker 命令行在 swarm 上创建服务，而 swarm 管理器会在 swarm 节点上作为容器运行副本。

我将通过创建服务来部署 NerdDinner 堆栈。所有服务将在我的集群上的同一个 Docker 网络中运行。在 swarm 模式下，Docker 有一种特殊类型的网络称为覆盖网络。覆盖网络是跨多个物理主机的虚拟网络，因此运行在一个 swarm 节点上的容器可以访问在另一个节点上运行的容器。服务发现的工作方式也是一样的：容器通过服务名称相互访问，Docker 的 DNS 服务器将它们指向一个容器。

要创建覆盖网络，您需要指定要使用的驱动程序并给网络命名。Docker CLI 将返回新网络的 ID，就像其他资源一样：

```
> docker network create --driver overlay nd-swarm
206teuqo1v14m3o88p99jklrn
```

您可以列出网络，您会看到新网络使用覆盖驱动程序，并且范围限定为群集，这意味着使用此网络的任何容器都可以相互通信，无论它们在哪个节点上运行：

```
> docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
osuduab0ge73        ingress             overlay             swarm
5176f181eee8        nat                 nat                 local
206teuqo1v14        nd-swarm            overlay             swarm
```

这里的输出还显示了默认的`nat`网络，它具有本地范围，因此容器只能在同一主机上相互访问。在群集模式下创建的另一个网络称为`ingress`，这是具有发布端口的服务的默认网络。

我将使用新网络来部署 NerdDinner 服务，因为这将使我的应用与群集中将使用自己网络的其他应用隔离开来。我将在本章后面使用 Docker Compose 文件来部署整个解决方案，但我将首先通过手动使用`docker service create`命令来创建服务，以便您可以看到服务与容器的不同之处。这是如何在 Docker Swarm 中将 NATS 消息队列部署为服务的方法：

```
docker service create `   --network nd-swarm `
  --name message-queue ` dockeronwindows/ch05-nats:2e 
```

`docker service create`的必需选项除了镜像名称外，但对于分布式应用程序，您需要指定以下内容：

+   `network`：要连接到服务容器的 Docker 网络

+   `name`：用作其他组件 DNS 条目的服务名称

Docker 支持容器的不同类型的 DNS 解析。默认值是虚拟 IP `vip` 模式，您应该使用它，因为它是最高性能的。 `vip` 模式仅支持从 Windows Server 2019 开始，因此对于较早版本，您将看到端点模式设置为`dnsrr`的示例。这是 DNS 轮询模式，效率较低，并且可能会在客户端缓存 DNS 响应时引起问题，因此除非必须与 Windows Server 2016 上的容器一起工作，否则应避免使用它。

您可以从连接到群集管理器的 Docker CLI 中运行`service create`命令。管理器查看群集中的所有节点，并确定哪些节点有能力运行副本，然后安排任务在节点上创建为容器。默认副本级别为*one*，因此此命令只创建一个容器，但它可以在群集中的任何节点上运行。

`docker service ps`显示正在运行服务的副本，包括托管每个容器的节点的名称：

```
> docker service ps message-queue
ID    NAME      IMAGE     NODE  DESIRED  STATE  CURRENT    STATE
xr2vyzhtlpn5 message-queue.1  dockeronwindows/ch05-nats:2e  win2019-02  Running        Running
```

在这种情况下，经理已经安排了一个容器在节点`win2019-02`上运行，这是我集群中唯一的工作节点。看起来如果我直接在该节点上运行 Docker 容器，我会得到相同的结果，但是将其作为 Docker Swarm 服务运行给了我编排的所有额外好处：

+   **应用程序可靠性**：如果此容器停止，经理将安排立即启动替代容器。

+   **基础设施可靠性**：如果工作节点宕机，经理将安排在不同节点上运行新的容器。

+   **可发现性**：该容器连接到一个覆盖网络，因此可以使用服务名称与在其他节点上运行的容器进行通信（Windows 容器甚至可以与同一集群中运行的 Linux 容器进行通信，反之亦然）。

在 Docker Swarm 中运行服务比在单个 Docker 服务器上运行容器有更多的好处，包括安全性、可扩展性和可靠的应用程序更新。我将在本章中涵盖它们。

在源代码存储库中，`ch07-create-services`文件夹中有一个脚本，按正确的顺序启动 NerdDinner 的所有服务。每个`service create`命令的选项相当于第六章中 Compose 文件的服务定义，*使用 Docker Compose 组织分布式解决方案*。前端服务和 Traefik 反向代理中只有一些差异。

Traefik 在 Docker Swarm 中运行得很好——它连接到 Docker API 来构建其前端路由映射，并且以与在单个运行 Docker Engine 的服务器上完全相同的方式代理来自后端容器的内容。要在 swarm 模式下向 Traefik 注册服务，您还需要告诉 Traefik 容器中的应用程序使用的端口，因为它无法自行确定。REST API 服务定义添加了`traefik.port`标签：

```
docker service create `   --network nd-swarm `
  --env-file db-credentials.env `
  --name nerd-dinner-api `
  --label "traefik.frontend.rule=Host:api.nerddinner.swarm"  `
  --label "traefik.port=80"  `
 dockeronwindows/ch05-nerd-dinner-api:2e
```

Traefik 本身是在 swarm 模式下创建的最复杂的服务，具有一些额外的选项：

```
docker service create `
  --detach=true `
  --network nd-swarm ` --constraint=node.role==manager `  --publish 80:80  --publish 8080:8080  `
  --mount type=bind,source=C:\certs\client,target=C:\certs `
  --name reverse-proxy `
 sixeyed/traefik:v1.7.8-windowsservercore-ltsc2019 `
  --docker --docker.swarmMode --docker.watch `
  --docker.endpoint=tcp://win2019-dev-02:2376  ` --docker.tls.ca=/certs/ca.pem `
  --docker.tls.cert=/certs/cert.pem `
  --docker.tls.key=/certs/key.pem `
  --api
```

你只能从运行在管理节点上的 Docker API 获取有关集群服务的信息，这就是为什么你需要将 Docker CLI 连接到管理节点以处理集群资源。服务的`constraint`选项确保 Docker 只会将容器调度到满足约束条件的节点上运行。在这种情况下，服务副本只会在管理节点上运行。这不是唯一的选择 - 如果你已经配置了对 Docker API 的安全远程访问，你可以在工作节点上运行 Traefik。

为了将 Traefik 连接到 Docker API，我以前使用卷来挂载 Windows 命名的*pipe*，但是这个功能在 Docker Swarm 中还不支持。所以，我改用 TCP 连接到 API，指定管理者的 DNS 名称`win2019-dev-02`。我已经用 TLS 保护了我的 Docker 引擎（就像我在第一章中解释的那样，在 Windows 上使用 Docker 入门），所以我还提供了客户端证书来安全地使用连接。证书存储在我的管理节点上的`C:\certs\client`中，我将其挂载为容器内的一个目录。

*服务挂载的命名管道支持*意味着你可以使用挂载管道的方法，这样做更容易，因为你不需要指定管理者的主机名，或者提供 TLS 证书。这个功能计划在 Docker 19.03 中推出，并且可能在你阅读本书时已经可用。Docker 的好处在于它是由开源组件构建的，因此诸如此类的功能都是公开讨论的 - [`github.com/moby/moby/issues/34795`](https://github.com/moby/moby/issues/34795)会告诉你背景和当前状态。

当我在我的集群上运行脚本时，我会得到一个服务 ID 列表作为输出：

```
> .\ch07-run-nerd-dinner.ps1
206teuqo1v14m3o88p99jklrn
vqnncr7c9ni75ntiwajcg05ym
2pzc8c5rahn25l7we3bzqkqfo
44xsmox6d8m480sok0l4b6d80
u0uhwiakbdf6j6yemuy6ey66p
v9ujwac67u49yenxk1albw4bt
s30phoip8ucfja45th5olea48
24ivvj205dti51jsigneq3z8q
beakbbv67shh0jhtolr35vg9r
sc2yzqvf42z4l88d3w31ojp1c
vx3zyxx2rubehee9p0bov4jio
rl5irw1w933tz9b5cmxyyrthn
```

现在我可以用`docker service ls`看到所有正在运行的服务：

```
> docker service ls
ID           NAME          MODE       REPLICAS            IMAGE 
8bme2svun122 message-queue             replicated 1/1      nats:nanoserver
deevh117z4jg nerd-dinner-homepage      replicated 1/1      dockeronwindows/ch03-nerd-dinner-homepage...
lxwfb5s9erq6 nerd-dinner-db            replicated 1/1      dockeronwindows/ch06-nerd-dinner-db:latest
ol7u97cpwdcn nerd-dinner-index-handler replicated 1/1      dockeronwindows/ch05-nerd-dinner-index...
rrgn4n3pecgf elasticsearch             replicated 1/1      sixeyed/elasticsearch:nanoserver
w7d7svtq2k5k nerd-dinner-save-handler  replicated 1/1      dockeronwindows/ch05-nerd-dinner-save...
ydzb1z1af88g nerd-dinner-web           replicated 1/1      dockeronwindows/ch05-nerd-dinner-web:latest
ywrz3ecxvkii kibana                    replicated 1/1      sixeyed/kibana:nanoserver
```

每个服务都列出了一个`1/1`的副本状态，这意味着一个副本正在运行，而请求的服务级别是一个副本。这是用于运行服务的容器数量。Swarm 模式支持两种类型的分布式服务：复制和全局。默认情况下，分布式服务只有一个副本，这意味着在集群上只有一个容器。我的脚本中的`service create`命令没有指定副本计数，所以它们都使用默认值*one*。

# 跨多个容器运行服务

复制的服务是你如何在集群模式下扩展的方式，你可以更新正在运行的服务来添加或删除容器。与 Docker Compose 不同，你不需要一个定义每个服务期望状态的 Compose 文件；这些细节已经存储在集群中，来自`docker service create`命令。要添加更多的消息处理程序，我使用`docker service scale`，传递一个或多个服务的名称和期望的服务级别：

```
> docker service scale nerd-dinner-save-handler=3
nerd-dinner-save-handler scaled to 3
overall progress: 1 out of 3 tasks
1/3: starting  [============================================>      ]
2/3: starting  [============================================>      ]
3/3: running   [==================================================>]
```

消息处理程序服务是使用默认的单个副本创建的，因此这将添加两个容器来共享 SQL Server 处理程序服务的工作。在多节点集群中，管理器可以安排容器在任何具有容量的节点上运行。我不需要知道或关心哪个服务器实际上在运行容器，但我可以通过`docker service ps`深入了解服务列表，看看容器在哪里运行：

```
> docker service ps nerd-dinner-save-handler
ID      NAME    IMAGE  NODE            DESIRED STATE  CURRENT STATE 
sbt4c2jof0h2  nerd-dinner-save-handler.1 dockeronwindows/ch05-nerd-dinner-save-handler:2e    win2019-dev-02      Running             Running 23 minutes ago
bibmh984gdr9  nerd-dinner-save-handler.2 dockeronwindows/ch05-nerd-dinner-save-handler:2e    win2019-dev-02      Running             Running 3 minutes ago
3lkz3if1vf8d  nerd-dinner-save-handler.3 dockeronwindows/ch05-nerd-dinner-save-handler:2e   win2019-02           Running             Running 3 minutes ago
```

在这种情况下，我正在运行一个双节点集群，副本分布在节点`win2019-dev-02`和`win2019-02`之间。集群模式将服务进程称为副本，但它们实际上只是容器。你可以登录到集群的节点，并像往常一样使用`docker ps`、`docker logs`和`docker top`命令管理服务容器。

通常情况下，你不会这样做。运行副本的节点只是由集群为你管理的黑匣子；你通过管理节点与你的服务一起工作。就像 Docker Compose 为服务提供了日志的整合视图一样，你可以通过连接到集群管理器的 Docker CLI 获得相同的视图：

```
PS> docker service logs nerd-dinner-save-handler
nerd-dinner-save-handler.1.sbt4c2jof0h2@win2019-dev-02
    | Connecting to message queue url: nats://message-queue:4222
nerd-dinner-save-handler.1.sbt4c2jof0h2@win2019-dev-02
    | Listening on subject: events.dinner.created, queue: save-dinner-handler
nerd-dinner-save-handler.2.bibmh984gdr9@win2019-dev-02
    | Connecting to message queue url: nats://message-queue:4222
nerd-dinner-save-handler.2.bibmh984gdr9@win2019-dev-02
    | Listening on subject: events.dinner.created, queue: save-dinner-handler
...
```

副本是集群为服务提供容错的方式。当你使用`docker service create`、`docker service update`或`docker service scale`命令为服务指定副本级别时，该值将记录在集群中。管理节点监视服务的所有任务。如果容器停止并且运行服务的数量低于期望的副本级别，新任务将被安排以替换停止的容器。在本章后面，我将演示当我在多节点集群上运行相同的解决方案时，我可以从集群中取出一个节点，而不会造成任何服务的丢失。

# 全局服务

替代复制服务的选择是全局服务。在某些情况下，您可能希望同一个服务在集群的每个节点上作为单个容器运行。为此，您可以以全局模式运行服务——Docker 在每个节点上精确安排一个任务，并且任何加入的新节点也将安排一个任务。

全局服务对于具有许多服务使用的组件的高可用性可能很有用，但是再次强调，并不是通过运行许多实例来获得集群化的应用程序。NATS 消息队列可以在多台服务器上作为集群运行，并且可以作为全局服务运行的一个很好的候选。但是，要将 NATS 作为集群运行，每个实例都需要知道其他实例的地址，这与 Docker Engine 分配的动态虚拟 IP 地址不兼容。

相反，我可以将我的 Elasticsearch 消息处理程序作为全局服务运行，因此每个节点都将运行一个消息处理程序的实例。您无法更改正在运行的服务的模式，因此首先需要删除原始服务。

```
> docker service rm nerd-dinner-index-handler
nerd-dinner-index-handler 
```

然后，我可以创建一个新的全局服务。

```
> docker service create `
>>  --mode=global `
>>  --network nd-swarm `
>>  --name nerd-dinner-index-handler `
>>  dockeronwindows/ch05-nerd-dinner-index-handler:2e;
q0c20sx5y25xxf0xqu5khylh7
overall progress: 2 out of 2 tasks
h2ripnp8hvty: running   [==================================================>]
jea4p57ajjal: running   [==================================================>]
verify: Service converged 
```

现在我在集群中的每个节点上都有一个任务在运行，如果节点被添加到集群中，任务的总数将增加，如果节点被移除，任务的总数将减少。这对于您想要分发以实现容错的服务可能很有用，并且您希望服务的总容量与集群的大小成比例。

全局服务在监控和审计功能中也很有用。如果您有诸如 Splunk 之类的集中式监控系统，或者正在使用 Elasticsearch Beats 进行基础设施数据捕获，您可以将代理作为全局服务在每个节点上运行。

通过全局和复制服务，Docker Swarm 提供了扩展应用程序和维护指定服务水平的基础设施。如果您有固定大小的集群但可变的工作负载，这对于本地部署非常有效。您可以根据需求扩展应用程序组件，只要它们不都需要在同一时间进行峰值处理。在云中，您有更多的灵活性，可以通过向集群添加新节点来增加集群的总容量，从而更广泛地扩展应用程序服务。

在许多实例中扩展运行应用程序通常会增加复杂性 - 您需要一种注册所有活动实例的方式，一种在它们之间共享负载的方式，以及一种监视所有实例的方式，以便如果有任何实例失败，它们不会有任何负载发送到它们。这一切都是 Docker Swarm 中内置的功能，它透明地提供服务发现、负载均衡、容错和自愈应用程序的基础设施。

# Swarm 模式中的负载均衡和扩展

Docker 使用 DNS 进行服务发现，因此容器可以通过标准网络找到彼此。应用程序在其客户端连接配置中使用服务器名称，当应用程序进行 DNS 查询以找到目标时，Docker 会响应容器的 IP 地址。在 Docker Swarm 中也是一样的，当您的目标**服务器**名称实际上可能是在集群中运行着数十个副本的 Docker 服务的名称。

Docker 有两种方式来管理具有多个副本的服务的 DNS 响应。默认情况下是使用**VIP**：虚拟 IP 地址。Docker 为服务使用单个 IP 地址，并依赖于主机操作系统中的网络堆栈将 VIP 上的请求路由到实际的容器。VIP 负责负载均衡和健康。请求被分享给服务中的所有健康容器。这个功能在 Linux 中已经存在很久了，在 Windows Server 2019 中是新功能。

VIP 的替代方案是**DNSRR**：**DNS 轮询**，您可以在服务配置中的`endpoint_mode`设置中指定。DNSRR 返回服务中所有健康容器的 IP 地址列表，并且列表的顺序会轮换以提供负载均衡。在 Windows Server 2019 之前，DNSRR 是 Windows 容器的唯一选项，并且您会在许多示例中看到它，但 VIP 是更好的选择。客户端有缓存 DNS 查询响应的倾向。使用 DNSRR，您可以更新一个服务并发现客户端已经缓存了一个已被替换的旧容器的 IP 地址，因此它们的连接失败。这在 VIP 中不会发生，在 VIP 中，DNS 响应中有一个单一的 IP 地址，客户端可以安全地缓存它，因为它总是会路由到一个健康的容器。

Docker Swarm 负责在服务副本之间负载平衡网络流量，但它也负责负载平衡进入集群的外部流量。在新的 NerdDinner 架构中，只有一个组件是公开访问的——Traefik 反向代理。我们知道一个端口在一台机器上只能被一个进程使用，所以这意味着我们只能将代理服务扩展到集群中每个节点的最大一个容器。但是 Docker Swarm 允许我们过度或不足地提供服务，使用机器上的相同端口来处理零个或多个副本。

附加到覆盖网络的集群服务在发布端口时与标准容器的行为不同。集群中的每个节点都监听发布的端口，当接收到流量时，它会被定向到一个健康的容器。该容器可以在接收请求的节点上运行，也可以在不同的节点上运行。

在这个例子中，客户端在 Docker Swarm 中运行的服务的标准端口`80`上进行了 HTTP GET 请求：

![](img/f379e589-aa28-4882-9d1f-d8aa739e6e2f.png)

1.  客户端请求到达一个没有运行任何服务副本的节点。该节点没有在端口`80`上监听的容器，因此无法直接处理请求。

1.  接收节点将请求转发到集群中另一个具有在端口`80`上监听的容器的节点——这对原始客户端来说是不可见的。

1.  新节点将请求转发到正在运行的容器，该容器处理请求并发送响应。

这被称为**入口网络**，它是一个非常强大的功能。这意味着您可以在大型集群上运行低规模的服务，或者在小型集群上运行高规模的服务，它们将以相同的方式工作。如果服务的副本少于集群中的节点数，这不是问题，因为 Docker 会透明地将请求发送到另一个节点。如果服务的副本多于节点数，这也不是问题，因为每个节点都可以处理请求，Docker 会在节点上的容器之间负载平衡流量。

Docker Swarm 中的网络是一个值得详细了解的主题，因为它将帮助您设计和交付可扩展和具有弹性的系统。我编写了一门名为**在 Docker Swarm 模式集群中管理负载平衡和扩展**的 Pluralsight 课程，涵盖了 Linux 和 Windows 容器的所有关键主题。

负载均衡和服务发现都基于健康的容器，并且这是 Docker Swarm 的一个功能，不需要我进行任何特殊设置。在群集模式下运行的服务默认为 VIP 服务发现和用于发布端口的入口网络。当我在 Docker Swarm 中运行 NerdDinner 时，我不需要对我的部署进行任何更改，就可以在生产环境中获得高可用性和扩展性，并且可以专注于自己应用程序的配置。

# 在 Docker Swarm 中管理应用程序配置

我在第五章中花了一些时间，*采用基于容器的解决方案设计*，为 NerdDinner 堆栈构建了一个灵活的配置系统。其中的核心原则是将开发的默认配置捆绑到每个镜像中，但允许在运行容器时覆盖设置。这意味着我们将在每个环境中使用相同的 Docker 镜像，只是交换配置设置以更改行为。

这适用于单个 Docker 引擎，我可以使用环境变量来覆盖单个设置，并使用卷挂载来替换整个配置文件。在 Docker Swarm 中，您可以使用 Docker 配置对象和 Docker 秘密来存储可以传递给容器的群集中的数据。这比使用环境变量和文件更加整洁地处理配置和敏感数据，但这意味着我在每个环境中仍然使用相同的 Docker 镜像。

# 在 Docker 配置对象中存储配置

在群集模式中有几种新资源 - 除了节点和服务外，还有堆栈、秘密和配置。配置对象只是在群集中创建的文本文件，并在服务容器内部作为文件出现。它们是管理配置设置的绝佳方式，因为它们为您提供了一个存储所有应用程序设置的单一位置。

您可以以两种方式使用配置对象。您可以使用`docker config`命令创建和管理它们，并在 Docker 服务命令和 Docker Compose 文件中使其对服务可用。这种清晰的分离意味着您的应用程序定义与配置分开 - 定义在任何地方都是相同的，而配置是由 Docker 从环境中加载的。

Docker 将配置对象表面化为容器内的文本文件，位于您指定的路径，因此您可以在 swarm 中拥有一个名为`my-app-config`的秘密，显示为`C:\my-app\config\appSettings.config`。Docker 不关心文件内容，因此它可以是 XML、JSON、键值对或其他任何内容。由您的应用程序实际执行文件的操作，这可以是使用完整文件作为配置，或将文件内容与 Docker 镜像中内置的一些默认配置合并。

在我现代化 NerdDinner 时，我已经为我的应用程序设置转移到了.NET Core 配置框架。我在组成 NerdDinner 的所有.NET Framework 和.NET Core 应用程序中都使用相同的`Config`类。`Config`类为配置提供程序添加了自定义文件位置：

```
public  static  IConfigurationBuilder  AddProviders(IConfigurationBuilder  config) {
  return  config.AddJsonFile("config/appsettings.json")
               .AddEnvironmentVariables()
               .AddJsonFile("config/config.json", optional: true)
               .AddJsonFile("config/secrets.json", optional: true); } 
```

配置提供程序按优先顺序倒序列出。首先，它们从应用程序镜像的`config/appsettings.json`文件中加载。然后，合并任何环境变量-添加新键，或替换现有键的值。接下来，如果路径`config/config.json`处存在文件，则其内容将被合并-覆盖任何现有设置。最后，如果路径`config/secrets.json`处存在文件，则其值将被合并。

这种模式让我可以使用一系列配置源。应用程序的默认值都存在于 Docker 镜像中。在运行时，用户可以使用环境变量或环境变量文件指定覆盖值-这对于在单个 Docker 主机上工作的开发人员来说很容易。在集群环境中，部署可以使用 Docker 配置对象和秘密，这将覆盖默认值和任何环境变量。

举个简单的例子，我可以更改新 REST API 的日志级别。在 Docker 镜像的`appsettings.json`文件中，日志级别设置为`Warning`。每次有`GET`请求时，应用程序都会写入信息级别的日志，因此如果我在配置中更改日志级别，我将能够看到这些日志条目。

我想要在名为`nerd-dinner-api-config.json`的文件中使用我想要的设置：

```
{
  "Logging": {
  "LogLevel": {
   "Default": "Information"
  } 
} }
```

首先，我需要将其存储为 swarm 中的配置对象，因此容器不需要访问原始文件。我使用`docker config create`来实现这一点，给对象一个名称和配置源的路径。

```
docker config create nerd-dinner-api-config .\configs\nerd-dinner-api-config.json
```

您只需要在创建配置对象时访问该文件。现在数据存储在 swarm 中。swarm 中的任何节点都可以获取配置数据并将其提供给容器，任何具有对 Docker Engine 访问权限的人都可以查看配置数据，而无需该源文件。`docker config inspect`会显示配置对象的内容。

```
> docker config inspect --pretty nerd-dinner-api-config
ID:                     yongm92k597gxfsn3q0yhnvtb
Name:                   nerd-dinner-api-config
Created at:             2019-02-13 22:09:04.3214402 +0000 utc
Updated at:             2019-02-13 22:09:04.3214402 +0000 utc
Data:
{
 "Logging": {
 "LogLevel": {
 "Default": "Information"
    }
 }
}
```

您可以通过检查来查看配置对象的纯文本值。这对于故障排除应用程序问题非常有用，但对于安全性来说不好——您应该始终使用 Docker secrets 来存储敏感配置值，而不是配置对象。

# 在 swarm 服务中使用 Docker 配置对象

在创建服务时，您可以使用`--config`选项将配置对象提供给容器。然后，您应该能够直接在应用程序中使用它们，但可能会有一个陷阱。当将配置对象作为文件呈现给容器时，它们受到保护，只有管理帐户才能读取它们。如果您的应用程序以最低特权用户身份运行，它可以看到配置文件，但无法读取它。这是一个安全功能，旨在在某人获得对容器中文件系统的访问权限时保护您的配置文件。

在 Linux 容器中情况就不同了，您可以指定在容器内具有文件所有权的用户 ID，因此可以让最低特权帐户访问该文件。Windows 容器不支持该功能，但 Windows 容器正在不断发展，以实现与 Linux 容器功能完备，因此这应该会在未来的版本中实现。在撰写本文时，要使用配置对象，应用程序需要以管理员帐户或具有本地系统访问权限的帐户运行。

以提升的权限运行应用程序在安全角度不是一个好主意，但当您在容器中运行时，这就不那么值得关注了。我在《第九章》中涵盖了这一点，*了解 Docker 的安全风险和好处*。

我已经更新了来自《第五章》*采用基于容器的解决方案设计*的 REST API 的 Dockerfile，以使用容器中的内置管理员帐户：

```
# escape=` FROM microsoft/dotnet:2.1-aspnetcore-runtime-nanoserver-1809 EXPOSE 80 WORKDIR /dinner-api ENTRYPOINT ["dotnet", "NerdDinner.DinnerApi.dll"] USER ContainerAdministrator COPY --from=dockeronwindows/ch05-nerd-dinner-builder:2e C:\dinner-api .
```

改变的只是`USER`指令，它设置了 Dockerfile 的其余部分和容器启动的用户。代码完全相同：我仍然使用来自第五章的构建器镜像，*采用面向容器的解决方案设计*。我已将此新镜像构建为`dockeronwindows/ch07-nerd-dinner-api:2e`，并且可以升级正在运行的 API 服务并应用新配置与`docker service update`：

```
docker service update `
  --config-add src=nerd-dinner-api-config,target=C:\dinner-api\config\config.json `
  --image dockeronwindows/ch07-nerd-dinner-api:2e  `
 nerd-dinner-api;
```

更新服务将正在运行的副本替换为新配置，在本例中，使用新镜像并应用配置对象。现在，当我对 REST API 进行`GET`请求时，它会以信息级别记录日志，并且我可以在服务日志中看到更多细节：

```
> docker service logs nerd-dinner-api
nerd-dinner-api.1.cjurm8tg1lmj@win2019-02    | Hosting environment: Production
nerd-dinner-api.1.cjurm8tg1lmj@win2019-02    | Content root path: C:\dinner-api
nerd-dinner-api.1.cjurm8tg1lmj@win2019-02    | Now listening on: http://[::]:80
nerd-dinner-api.1.cjurm8tg1lmj@win2019-02    | Application started. Press Ctrl+C to shut down.
nerd-dinner-api.1.cjurm8tg1lmj@win2019-02    | info: Microsoft.AspNetCore.Hosting.Internal.WebHost[1]
nerd-dinner-api.1.cjurm8tg1lmj@win2019-02    |       Request starting HTTP/1.1 GET http://api.nerddinner.swarm/api/dinners
nerd-dinner-api.1.cjurm8tg1lmj@win2019-02    | info: Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker[1]
nerd-dinner-api.1.cjurm8tg1lmj@win2019-02    |       Route matched with {action = "Get", controller = "Dinners"}. Executing action NerdDinner.DinnerApi.Controllers.DinnersController.Get (NerdDinner.DinnerApi)
```

您可以使用此方法来处理在不同环境之间更改的功能标志和行为设置。这是一种非常灵活的应用程序配置方法。使用单个 Docker 引擎的开发人员可以使用镜像中的默认设置运行容器，或者使用环境变量覆盖它们，或者通过挂载本地卷替换整个配置文件。在使用 Docker Swarm 的测试和生产环境中，管理员可以使用配置对象集中管理配置，而在每个环境中仍然使用完全相同的 Docker 镜像。

# 在 Docker secrets 中存储敏感数据

Swarm 模式本质上是安全的。所有节点之间的通信都是加密的，并且 swarm 提供了分布在管理节点之间的加密数据存储。您可以将此存储用于应用程序**秘密**。秘密的工作方式与配置对象完全相同-您在 swarm 中创建它们，然后使它们对服务可用。不同之处在于，秘密在 swarm 的数据存储中是加密的，并且在从管理节点到工作节点的传输中也是加密的。它只在运行副本的容器内解密，然后以与配置对象相同的方式作为文件显示。

秘密是通过名称和秘密内容创建的，可以从文件中读取或输入到命令行中。我打算将我的敏感数据移动到 secrets 中，首先是 SQL Server 管理员帐户密码。在`ch07-app-config`文件夹中，我有一个名为`secrets`的文件夹，其中包含数据库密码的秘密文件。我将使用它来安全地存储密码在 swarm 中，但在数据库镜像支持秘密之前，我需要对其进行一些工作。

我将最新的 SQL Server 数据库架构打包到 Docker 镜像`dockeronwindows/ch06-nerd-dinner-db`中。该映像使用环境变量来设置管理员密码，这对开发人员来说很好，但在测试环境中不太好，因为您希望限制访问。我为本章准备了一个更新的版本，其中包括用于数据库的更新的 Dockerfile 和启动脚本，因此我可以从秘密文件中读取密码。

在`ch07-nerd-dinner-db`的`InitializeDatabase.ps1`脚本中，我添加了一个名为`sa_password_path`的新参数，并添加了一些简单的逻辑，以从文件中读取密码，如果该路径中存在文件：

```
if ($sa_password_path  -and (Test-Path  $sa_password_path)) {
  $password  =  Get-Content  -Raw $sa_password_path
  if ($password) {
    $sa_password  =  $password
    Write-Verbose  "Using SA password from secret file: $sa_password_path" }
```

这是一种完全不同的方法，与 REST API 中采用的方法相反。应用程序对配置有自己的期望，您需要将其与 Docker 的方法整合起来，以在文件中显示配置数据。在大多数情况下，您可以在 Dockerfile 中完成所有操作，因此不需要更改代码直接从文件中读取配置。

Dockerfile 使用具有密码文件路径的默认值的环境变量：

```
ENV sa_password_path="C:\secrets\sa-password"
```

这仍然支持以不同方式运行数据库。开发人员可以在不指定任何配置设置的情况下运行它，并且它将使用内置于映像中的默认密码，这与应用程序映像的连接字符串中的相同默认密码相同。在集群环境中，管理员可以单独创建秘密，而无需部署应用程序，并安全地访问数据库容器。

我需要创建秘密，然后更新数据库服务以使用秘密和应用密码的新映像：

```
docker secret create nerd-dinner-db-sa-password .\secrets\nerd-dinner-db-sa-password.txt; docker service update `
  --secret-add src=nerd-dinner-db-sa-password,target=C:\secrets\sa-password `
  --image dockeronwindows/ch07-nerd-dinner-db:2e  `
 nerd-dinner-db;
```

现在数据库正在使用由 Docker Swarm 保护的强密码。可以访问 Docker 引擎的用户无法看到秘密的内容，因为它只在明确使用秘密的服务的容器内解密。我可以检查秘密，但我只能看到元数据：

```
> docker secret inspect --pretty nerd-dinner-db-sa-password
ID:              u2zsrjouhicjnn1fwo5x8jqpk
Name:              nerd-dinner-db-sa-password
Driver:
Created at:        2019-02-14 10:33:04.0575536 +0000 utc
Updated at:        2019-02-14 10:33:04.0575536 +0000 utc
```

现在我的应用程序出现了问题，因为我已更新了数据库密码，但没有更新使用数据库的应用程序中的连接字符串。这是通过向 Docker Swarm 发出命令来管理分布式应用程序的危险。相反，您应该使用 Docker Compose 文件以声明方式管理应用程序，定义所有服务和其他资源，并将它们部署为 Docker 堆栈。

# 将堆栈部署到 Docker Swarm

Docker Swarm 中的堆栈解决了在单个主机上使用 Docker Compose 或在 Docker Swarm 上手动创建服务的限制。您可以从 Compose 文件创建堆栈，并且 Docker 将堆栈服务的所有元数据存储在 Swarm 中。这意味着 Docker 知道这组资源代表一个应用程序，您可以在任何 Docker 客户端上管理服务，而无需 Compose 文件。

*堆栈*是对构成您的应用程序的所有对象的抽象。它包含服务、卷和网络，就像标准的 Docker Compose 文件一样，但它还支持 Docker Swarm 对象——配置和密码——以及用于在规模上运行应用程序的附加部署设置。

堆栈甚至可以抽象出您正在使用的编排器。Docker Enterprise 同时支持 Docker Swarm 和 Kubernetes 在同一集群上，并且您可以使用简单的 Docker Compose 格式和 Docker CLI 将应用程序部署和管理为堆栈到任一编排器。

# 使用 Docker Compose 文件定义堆栈

Docker Compose 文件模式已经从支持单个 Docker 主机上的客户端部署发展到 Docker Swarm 上的堆栈部署。不同的属性集在不同的场景中是相关的，并且工具会强制执行。Docker Compose 将忽略仅适用于堆栈部署的属性，而 Docker Swarm 将忽略仅适用于单节点部署的属性。

我可以利用多个 Compose 文件来实现这一点，在一个文件中定义应用程序的基本设置，在一个覆盖文件中添加本地设置，并在另一个覆盖文件中添加 Swarm 设置。我已经在`ch07-docker-compose`文件夹中的 Compose 文件中这样做了。`docker-compose.yml`中的核心服务定义现在非常简单，它们只包括适用于每种部署模式的属性。甚至 Traefik 的反向代理定义也很简单：

```
reverse-proxy:
  image: sixeyed/traefik:v1.7.8-windowsservercore-ltsc2019
  networks:
 - nd-net 
```

在`docker-compose.local.yml`覆盖文件中，我添加了在我的笔记本电脑上开发应用程序和使用 Docker Compose 部署时相关的属性。对于 Traefik，我需要配置要运行的命令以及要发布的端口，并挂载一个用于 Docker Engine 命名管道的卷：

```
reverse-proxy:
  command: --docker --docker.endpoint=npipe:////./pipe/docker_engine --api
  ports:
  - "80"
  - "8080"
  volumes:
  - type: npipe
     source: \\.\pipe\docker_engine
     target: \\.\pipe\docker_engine 
```

在 `docker-compose.swarm.yml` 覆盖文件中，我有一个属性，当我在集群化的 Docker Swarm 环境中运行时应用——这可能是测试中的两节点 swarm 和生产中的 200 节点 swarm；Compose 文件将是相同的。 我设置了 Traefik 命令以使用 TCP 连接到 swarm 管理器，并且我正在使用 secrets 在 swarm 中存储 TLS 证书：

```
reverse-proxy:
  command: --docker --docker.swarmMode --docker.watch --docker.endpoint=tcp://win2019-dev-02:2376  
           --docker.tls.ca=/certs/ca.pem --docker.tls.cert=/certs/cert.pem ...
  ports:
   - "80:80"
   - "8080:8080"
  secrets:
   - source: docker-client-ca
      target: C:\certs\ca.pem
   - source: docker-client-cert
      target: C:\certs\cert.pem - source: docker-client-key target: C:\certs\key.pem
  deploy:
   placement:
     constraints:
      - node.role == manager
```

这个应用程序清单的唯一不可移植部分是我的 swarm 管理器的 DNS 名称 `win2019-dev-02`。 我在第六章中解释过，*使用 Docker Compose 组织分布式解决方案*，在 swarm 模式下还不能挂载命名管道，但很快就会推出。 当该功能到来时，我可以在 swarm 模式下像在单个 Docker 引擎上一样使用命名管道来使用 Traefik，并且我的 Compose 文件将在任何 Docker 集群上运行。

其余服务的模式相同：`compose.yml` 中有基本定义，本地文件中有开发人员的一组覆盖，以及一组替代覆盖在 swarm 文件中。 核心 Compose 文件不能单独使用，因为它没有指定的所有配置，这与第六章中的不同，*使用 Docker Compose 组织分布式解决方案*，我的 Docker Compose 文件是为开发设置的。 您可以使用最适合您的任何方法，但这种方式的优势在于每个环境的设置都在其自己的覆盖文件中。

有几个值得更详细查看的服务选项。 REST API 在核心 Compose 文件中定义，只需图像和网络设置。 本地覆盖添加了用于向代理注册 API 的标签，并且还捕获了对数据库服务的依赖关系：

```
nerd-dinner-api:
  depends_on:
   - nerd-dinner-db
  labels:
   - "traefik.frontend.rule=Host:api.nerddinner.local"
```

Swarm 模式不支持 `depends_on` 属性。 当您部署堆栈时，无法保证服务将以何种顺序启动。 如果您的应用程序组件具有 `retry` 逻辑以解决任何依赖关系，那么服务启动顺序就无关紧要。 如果您的组件不具有弹性，并且在无法访问依赖项时崩溃，那么 Docker 将重新启动失败的容器，并且经过几次重试后应用程序应该准备就绪。

传统应用程序通常缺乏弹性，它们假设它们的依赖始终可用并能立即响应。如果您转移到云服务，容器也是如此。Docker 将不断替换失败的容器，但即使对传统应用程序，您也可以通过在 Dockerfile 中构建启动检查和健康检查来增加弹性。

Swarm 定义添加了秘密和配置设置，容器标签的应用方式也有所不同。

```
nerd-dinner-api:
  configs:
   - source: nerd-dinner-api-config
      target: C:\dinner-api\config\config.json
  secrets:
   - source: nerd-dinner-api-secrets
      target: C:\dinner-api\config\secrets.json
  deploy:
  replicas: 2  labels:
     - "traefik.frontend.rule=Host:api.nerddinner.swarm"
     - "traefik.port=80" 
```

配置和秘密只适用于 Swarm 模式，但可以在任何 Compose 文件中包含它们——当您在单个 Docker 引擎上运行时，Docker Compose 会忽略它们。`deploy`部分也只适用于 Swarm 模式，它捕获了副本的基础架构设置。在这里，我有一个副本计数为 2，这意味着 Swarm 将为此服务运行两个容器。我还在`deploy`部分下有 Traefik 的标签，这确保了标签被应用到容器上，而不是服务本身。

Docker 使用标签来注释任何类型的对象——卷、节点、服务、秘密、容器和任何其他 Docker 资源都可以添加或删除标签，并且它们以键值对的形式暴露在 Docker Engine API 中。Traefik 只查找容器标签，这些标签在 Compose 文件的`deploy`部分中在 Swarm 模式下应用。如果您直接在服务部分下有标签，那么它们将被添加到服务而不是容器。在这种情况下，这意味着容器上没有标签，因此 Traefik 不会注册任何路由。

# 在 Docker Compose 文件中定义 Swarm 资源

在本章中，核心的`docker-compose.yml`文件只包含一个`services`部分；没有指定其他资源。这是因为我的应用程序的资源在单个 Docker 引擎部署和 Docker Swarm 之间都是不同的。

本地覆盖文件使用现有的`nat`网络，并对 SQL Server 和 Elasticsearch 使用默认规范的卷。

```
networks:
  nd-net:
    external:
      name: nat volumes:
  ch07-es-data: ch07-db-data:
```

Swarm 覆盖将所有服务附加到的相同`nd-net`网络映射为一个名为`nd-swarm`的外部网络，这个网络需要在我部署此堆栈之前存在。

```
networks:
  nd-net:
    external:
      name: nd-swarm
```

在集群覆盖中没有定义卷。在集群模式下，您可以像在单个 Docker 引擎上使用卷一样使用它们，但您可以选择使用不同的驱动程序，并将存储设备连接到数据中心或云存储服务以连接到您的容器卷。

Docker 中的存储本身就是一个完整的主题。我在我的 Pluralsight 课程**在 Docker 中处理数据和有状态应用程序**中详细介绍了它。在那门课程中，我演示了如何在桌面上以及在 Docker Swarm 中以高可用性和规模的方式运行有状态的应用程序和数据库。

在集群覆盖文件中有另外两个部分，涵盖了我的应用程序配置：

```
configs: nerd-dinner-api-config: external: true
  nerd-dinner-config: 
    external: true

secrets:
  nerd-dinner-db-sa-password:
    external: true nerd-dinner-save-handler-secrets: external: true nerd-dinner-api-secrets: external: true nerd-dinner-secrets: external: true
```

如果您看到这些并认为这是很多需要管理的`configs`和`secrets`，请记住，这些是您的应用程序无论在哪个平台上都需要的配置数据。Docker 的优势在于所有这些设置都被集中存储和管理，并且如果它们包含敏感数据，您可以选择安全地存储和传输它们。

我的所有配置和秘密对象都被定义为外部资源，因此它们需要在集群中存在才能部署我的应用程序。在`ch07-docker-stack`目录中有一个名为`apply-configuration.ps1`的脚本，其中包含所有的`docker config create`和`docker secret create`命令：

```
> .\apply-configuration.ps1
ntkafttcxvf5zjea9axuwa6u9
razlhn81s50wrqojwflbak6qx
npg65f4g8aahezqt14et3m31l
ptofylkbxcouag0hqa942dosz
dwtn1q0idjz6apbox1kh512ns
reecdwj5lvkeugm1v5xy8dtvb
nyjx9jd4yzddgrb2nhwvqjgri
b3kk0hkzykiyjnmknea64e3dk
q1l5yp025tqnr6fd97miwii8f
```

输出是新对象 ID 的列表。现在所有资源都存在，我可以将我的应用程序部署为一个堆栈。

# 从 Docker Compose 文件部署集群堆栈

我可以通过在开发笔记本上指定多个 Compose 文件（核心文件和本地覆盖）来使用 Docker Compose 部署应用程序。在集群模式下，您使用标准的`docker`命令，而不是`docker-compose`来部署堆栈。Docker CLI 不支持堆栈部署的多个文件，但我可以使用 Docker Compose 将源文件合并成一个单独的堆栈文件。这个命令从两个 Compose 文件中生成一个名为`docker-stack.yml`的单个 Compose 文件，用于堆栈部署：

```
docker-compose -f docker-compose.yml -f docker-compose.swarm.yml config > docker-stack.yml
```

Docker Compose 合并输入文件并检查输出配置是否有效。我将输出保存在一个名为`docker-stack.yml`的文件中。这是一个额外的步骤，可以轻松地融入到您的部署流程中。现在我可以使用包含核心服务描述、秘密和部署配置的堆栈文件在集群上部署我的堆栈。

您可以使用单个命令`docker stack deploy`从 Compose 文件中部署堆栈。您需要传递 Compose 文件的位置和堆栈的名称，然后 Docker 将创建 Compose 文件中的所有资源：

```
> docker stack deploy --compose-file docker-stack.yml nerd-dinner
Creating service nerd-dinner_message-queue
Creating service nerd-dinner_elasticsearch
Creating service nerd-dinner_nerd-dinner-api
Creating service nerd-dinner_kibana
Creating service nerd-dinner_nerd-dinner-index-handler
Creating service nerd-dinner_nerd-dinner-save-handler
Creating service nerd-dinner_reverse-proxy
Creating service nerd-dinner_nerd-dinner-web
Creating service nerd-dinner_nerd-dinner-homepage
Creating service nerd-dinner_nerd-dinner-db
```

结果是一组资源被逻辑地组合在一起形成堆栈。与 Docker Compose 不同，后者依赖命名约定和标签来识别分组，堆栈在 Docker 中是一等公民。我可以列出所有堆栈，这给我基本的细节——堆栈名称和堆栈中的服务数量：

```
> docker stack ls
NAME                SERVICES            ORCHESTRATOR
nerd-dinner         10                  Swarm
```

我的堆栈中有 10 个服务，从一个包含 137 行 YAML 的单个 Docker Compose 文件部署。对于这样一个复杂的系统来说，这是一个很小的配置量：两个数据库，一个反向代理，多个前端，一个 RESTful API，一个消息队列和多个消息处理程序。这样大小的系统通常需要一个运行数百页的 Word 部署文档，并且需要一个周末的手动工作来运行所有步骤。我只用了一个命令来部署这个系统。

我还可以深入了解运行堆栈的容器的状态和它们所在的节点，使用`docker stack ps`，或者使用`docker stack services`来获得堆栈中服务的更高级视图。

```
> docker stack services nerd-dinner
ID              NAME       MODE        REPLICAS        IMAGE
3qc43h4djaau  nerd-dinner_nerd-dinner-homepage       replicated  2/2       dockeronwindows/ch03...
51xrosstjd79  nerd-dinner_message-queue              replicated  1/1       dockeronwindows/ch05...
820a4quahjlk  nerd-dinner_elasticsearch              replicated  1/1       sixeyed/elasticsearch...
eeuxydk6y8vp  nerd-dinner_nerd-dinner-web            replicated  2/2       dockeronwindows/ch07...
jlr7n6minp1v  nerd-dinner_nerd-dinner-index-handler  replicated  2/2       dockeronwindows/ch05...
lr8u7uoqx3f8  nerd-dinner_nerd-dinner-save-handler   replicated  3/3       dockeronwindows/ch05...
pv0f37xbmz7h  nerd-dinner_reverse-proxy              replicated  1/1       sixeyed/traefik...
qjg0262j8hwl  nerd-dinner_nerd-dinner-db             replicated  1/1       dokeronwindows/ch07...
va4bom13tp71  nerd-dinner_kibana                     replicated  1/1       sixeyed/kibana...
vqdaxm6rag96  nerd-dinner_nerd-dinner-api            replicated  2/2       dockeronwindows/ch07...
```

这里的输出显示我有多个副本运行前端容器和消息处理程序。总共，在我的两节点集群上有 15 个容器在运行，这是两个虚拟机，总共有四个 CPU 核心和 8GB 的 RAM。在空闲时，容器使用的计算资源很少，我有足够的容量来运行额外的堆栈。我甚至可以部署相同堆栈的副本，为代理使用不同的端口，然后我可以在相同的硬件上运行两个完全独立的测试环境。

将服务分组到堆栈中可以更轻松地管理应用程序，特别是当您有多个应用程序在运行，每个应用程序中有多个服务时。堆栈是对一组 Docker 资源的抽象，但您仍然可以直接管理单个资源。如果我运行`docker service rm`，它将删除一个服务，即使该服务是堆栈的一部分。当我再次运行`docker stack deploy`时，Docker 会发现堆栈中缺少一个服务，并重新创建它。

当涉及到使用新的镜像版本或更改服务属性来更新应用程序时，您可以采取命令式方法直接修改服务，或者通过修改堆栈文件并再次部署来保持声明性。Docker 不会强加给您任何流程，但最好保持声明性，并将 Compose 文件用作唯一的真相来源。

我可以通过在堆栈文件的部署部分添加`replicas: 2`并再次部署它，或者通过运行`docker service update --replicas=2 nerd-dinner_nerd-dinner-save-handler`来扩展解决方案中的消息处理程序。如果我更新了服务但没有同时更改堆栈文件，那么下次部署堆栈时，我的处理程序将减少到一个副本。堆栈文件被视为期望的最终状态，如果当前状态偏离了，那么在再次部署时将进行纠正。

使用声明性方法意味着您始终在 Docker Compose 文件中进行这些更改，并通过再次部署堆栈来更新应用程序。Compose 文件与您的 Dockerfiles 和应用程序源代码一起保存在源代码控制中，因此它们可以进行版本控制、比较和标记。这意味着当您拉取应用程序的任何特定版本的源代码时，您将拥有构建和部署所需的一切。

秘密和配置是例外，您应该将它们保存在比中央源代码库更安全的位置，并且只有管理员用户才能访问明文。Compose 文件只是引用外部秘密，因此您可以在源代码控制中获得应用程序清单的唯一真相来源的好处，而敏感数据则保留在外部。

在开发和测试环境中运行单个节点或双节点集群是可以的。我可以将完整的 NerdDinner 套件作为一个堆栈运行，验证堆栈文件是否正确定义，并且可以扩展和缩小以检查应用程序的行为。这并不会给我带来高可用性，因为集群只有一个管理节点，所以如果管理节点下线，那么我就无法管理堆栈。在数据中心，您可以运行一个拥有数百个节点的集群，并且通过三个管理节点获得完整的高可用性。

您可以在云中运行它，构建具有更高可用性和规模弹性的群集。所有主要的云运营商都支持其 IaaS 服务中的 Docker，因此您可以轻松地启动预安装了 Docker 的 Linux 和 Windows VM，并使用本章中所见的简单命令将它们加入到群集中。

Docker Swarm 不仅仅是在集群中规模化运行应用程序。在多个节点上运行使我具有高可用性，因此在发生故障时我的应用程序可以继续运行，并且我可以利用它来支持应用程序生命周期，实现零停机滚动更新和自动回滚。

# 无停机部署更新

Docker Swarm 具有两个功能，可以在不影响应用程序的情况下更新整个堆栈-滚动更新和节点排空。滚动更新在您有一个组件的新版本要发布时，用新图像的新实例替换应用程序容器。更新是分阶段进行的，因此只要您有多个副本，就会始终有任务在运行以提供请求，同时其他任务正在升级。

应用程序更新将频繁发生，但您还需要定期更新主机，无论是升级 Docker 还是应用 Windows 补丁。Docker Swarm 支持排空节点，这意味着在节点上运行的所有容器都将停止，并且不会再安排更多容器。如果在排空节点时服务的副本级别下降，任务将在其他节点上启动。当节点排空时，您可以更新主机，然后将其加入到群集中。

我将通过覆盖这两种情况来完成本章。

# 更新应用程序服务

我在 Docker Swarm 上运行我的堆栈，现在我要部署一个应用程序更新-一个具有重新设计的 UI 的新主页组件，这是一个很好的、容易验证的变化。我已经构建了`dockeronwindows/ch07-nerd-dinner-homepage:2e`。为了进行更新，我有一个新的 Docker Compose 覆盖文件，其中只包含现有服务的新图像名称：

```
version: '3.7' services:
  nerd-dinner-homepage:
    image: dockeronwindows/ch07-nerd-dinner-homepage:2e
```

在正常发布中，您不会使用覆盖文件来更新一个服务。您将更新核心 Docker Compose 文件中的图像标签，并将文件保存在源代码控制中。我使用覆盖文件是为了更容易地跟随本章的示例。

此更新有两个步骤。首先，我需要通过组合 Compose 文件和所有覆盖文件来生成新的应用程序清单：

```
docker-compose `
  -f docker-compose.yml `
  -f docker-compose.swarm.yml `
 -f new-homepage.yml `
 config > docker-stack-2.yml
```

现在我可以部署这个堆栈：

```
> docker stack deploy -c .\docker-stack-2.yml nerd-dinner
Updating service nerd-dinner_nerd-dinner-save-handler (id: 0697sstia35s7mm3wo6q5t8nu)
Updating service nerd-dinner_nerd-dinner-homepage (id: v555zpu00rwu734l2zpi6rwz3)
Updating service nerd-dinner_reverse-proxy (id: kchmkm86wk7d13eoj9t26w1hw)
Updating service nerd-dinner_message-queue (id: jlzt6svohv1bo4og0cbx4y5ac)
Updating service nerd-dinner_nerd-dinner-api (id: xhlzf3kftw49lx9f8uemhv0mo)
Updating service nerd-dinner_elasticsearch (id: 126s2u0j78k1c9tt9htdkup8x)
Updating service nerd-dinner_nerd-dinner-index-handler (id: zd651rohewgr3waud6kfvv7o0)
Updating service nerd-dinner_nerd-dinner-web (id: yq6c51bzrnrfkbwqv02k8shvr)
Updating service nerd-dinner_nerd-dinner-db (id: wilnzl0jp1n7ey7kgjyjak32q)
Updating service nerd-dinner_kibana (id: uugw7yfaza84k958oyg45cznp)
```

命令输出显示所有服务都在 `Updating`，但 Docker Swarm 只会实际更改 Compose 文件中期望状态与运行状态不同的服务。在这个部署中，它将使用 Compose 文件中的新镜像名称更新主页服务。

更新对您要升级的镜像没有任何限制。它不需要是同一存储库名称的新标签；它可以是完全不同的镜像。这是非常灵活的，但这意味着您需要小心，不要意外地用新版本的 Web 应用程序更新您的消息处理程序，反之亦然。

Docker 一次更新一个容器，您可以配置更新之间的延迟间隔以及更新失败时要采取的行为。在更新过程中，我可以运行 `docker service ps` 命令，并看到原始容器处于 `Shutdown` 状态，替换容器处于 `Running` 或 `Starting` 状态：

```
> docker service ps nerd-dinner_nerd-dinner-homepage
ID    NAME   IMAGE   NODE  DESIRED STATE CURRENT STATE ERROR  PORTS
is12l1gz2w72 nerd-dinner_nerd-dinner-homepage.1 win2019-02          Running Running about a minute ago
uu0s3ihzp4lk \_ nerd-dinner_nerd-dinner-homepage.1 win2019-02       Shutdown Shutdown 2 minutes ago
0ruzheqp29z1 nerd-dinner_nerd-dinner-homepage.2 win2019-dev-02      Running Running 2 minutes ago
5ivddeffrkjj \_ nerd-dinner_nerd-dinner-homepage.2 win2019-dev-02   Shutdown  Shutdown 2 minutes ago
```

新的 NerdDinner 主页应用程序的 Dockerfile 具有健康检查，Docker 会等到新容器的健康检查通过后才会继续替换下一个容器。在滚动更新期间，一些用户将看到旧的主页，而一些用户将看到时尚的新主页：

![](img/c490a14f-6719-4859-8377-c5232d8783cd.png)

Traefik 与主页容器之间的通信使用 VIP 网络，因此它只会将流量发送到运行容器的主机 - 用户将从已更新并运行 `ch07` 镜像的容器或者即将更新并运行 `ch03` 镜像的容器中获得响应。如果这是一个高流量的应用程序，我需要确保服务中有足够的容量，这样当一个任务正在更新时，剩余的任务可以处理负载。

滚动更新可以实现零停机时间，但这并不一定意味着您的应用程序在更新期间将正常运行。这个过程只适用于无状态应用程序 - 如果任务存储任何会话状态，那么用户体验将受到影响。当包含状态的容器被替换时，状态将丢失。如果您有有状态的应用程序，您需要计划一个更谨慎的升级过程 - 或者最好是将这些组件现代化，以便在容器中运行的共享组件中存储状态。

# 服务更新回滚

在群集模式下更新服务时，群集会存储先前部署的配置。如果您发现发布存在问题，可以使用单个命令回滚到先前的状态：

```
> docker service update --rollback nerd-dinner_nerd-dinner-homepage
nerd-dinner_nerd-dinner-homepage
```

回滚是服务更新的一种特殊形式。`rollback`标志不是传递任务要更新的镜像名称，而是对服务使用的先前镜像进行滚动更新。同样，回滚是一次只更新一个任务，因此这是一个零停机过程。无论您如何应用更新，都可以使用此命令回滚到之前的状态，无论您是使用`docker stack deploy`还是`docker service update`。

回滚是少数几种情况之一，您可能希望使用命令式命令来管理应用程序，而不是声明式的 Docker Compose 文件。如果您发现服务更新存在问题，只需使用单个命令即可将其回滚到先前状态，这非常棒。

服务更新仅保留一个先前的服务配置用于回滚。如果您从版本 1 更新到版本 2，然后再更新到版本 3，版本 1 的配置将丢失。您可以从版本 3 回滚到版本 2，但如果再次从版本 2 回滚，将回到先前的版本，这将使您在版本 3 之间循环。

# 配置更新行为

对于大规模部署，可以更改默认的更新行为，以便更快地完成滚动更新，或者运行更保守的滚动更新策略。默认行为是一次只更新一个任务，任务更新之间没有延迟，如果任务更新失败，则暂停滚动更新。可以使用三个参数覆盖配置：

+   `update-parallelism`：同时更新的任务数量

+   `update-delay`：任务更新之间等待的时间段；可以指定为小时、分钟和秒

+   `update-failure-action`：如果任务更新失败，要采取的操作，是继续还是停止滚动更新

您可以在 Dockerfile 中指定默认参数，以便将其嵌入到镜像中，或者在 Compose 文件中指定默认参数，以便在部署时或使用服务命令时设置。对于 NerdDinner 的生产部署，我可能有九个 SQL 消息处理程序实例，Compose 文件中的`update_config`设置为以三个为一批进行更新，并设置为 10 秒的延迟：

```
nerd-dinner-save-handler:
  deploy:
  replicas: 9
  update_config:
    parallelism: 3
    delay: 10s
...
```

服务的更新配置也可以通过`docker service update`命令进行更改，因此您可以修改更新参数并通过单个命令启动滚动升级。

健康检查在服务更新中尤为重要。如果服务更新中的新任务健康检查失败，这可能意味着镜像存在问题。完成部署可能导致 100%的不健康任务和一个破损的应用程序。默认的更新配置可以防止这种情况发生，因此如果更新的任务没有进入运行状态，部署将被暂停。更新将不会继续进行，但这比拥有一个破损的更新应用程序要好。

# 更新集群节点

应用程序更新是更新例程的一部分，主机更新是另一部分。您的 Windows Docker 主机应该运行一个最小的操作系统，最好是 Windows Server 2019 Core。这个版本没有用户界面，因此更新的表面积要小得多，但仍然会有一些需要重新启动的 Windows 更新。

重新启动服务器是一个侵入性的过程——它会停止 Docker Engine Windows 服务，杀死所有正在运行的容器。出于同样的原因，升级 Docker 同样具有侵入性：这意味着需要重新启动 Docker Engine。在集群模式中，您可以通过在更新期间将节点从服务中移除来管理此过程，而不会影响服务水平。

我将用我的集群来展示这一点。如果我需要在`win2019-02`上工作，我可以通过`docker node update`优雅地重新安排它正在运行的任务，将其置于排水模式：

```
> docker node update --availability drain win2019-02
win-node02
```

将节点置于排水模式意味着所有容器都将被停止，由于这些是服务任务容器，它们将在其他节点上被新容器替换。当排水完成时，`win-node02`上将没有正在运行的任务：它们都已经被关闭。您可以看到任务已被故意关闭，因为“关闭”被列为期望状态：

```
> docker node ps win2019-02
ID   NAME  NODE         DESIRED STATE         CURRENT                STATE              
kjqr0b0kxoah  nerd-dinner_nerd-dinner-homepage.1      win2019-02     Shutdown Shutdown 48 seconds ago
is12l1gz2w72 \_ nerd-dinner_nerd-dinner-homepage.1    win2019-02     Shutdown Shutdown 8 minutes ago
xdbsme89swha nerd-dinner_nerd-dinner-index-handler.1  win2019-02     Shutdown Shutdown 49 seconds ago
j3ftk04x1e9j  nerd-dinner_nerd-dinner-db.1            win2019-02     Shutdown 
Shutdown 47 seconds ago
luh79mmmtwca   nerd-dinner_nerd-dinner-api.1          win2019-02     Shutdown Shutdown 47 seconds ago
... 
```

我可以检查服务列表，并看到每个服务仍然处于所需的副本级别：

```
> docker service ls
ID              NAME                                 MODE          REPLICAS   
126s2u0j78k1  nerd-dinner_elasticsearch            replicated       1/1 
uugw7yfaza84  nerd-dinner_kibana                   replicated       1/1 
jlzt6svohv1b  nerd-dinner_message-queue            replicated       1/1 
xhlzf3kftw49  nerd-dinner_nerd-dinner-api          replicated       2/2  
wilnzl0jp1n7  nerd-dinner_nerd-dinner-db           replicated       1/1   
v555zpu00rwu nerd-dinner_nerd-dinner-homepage      replicated       2/2
zd651rohewgr nerd-dinner_nerd-dinner-index-handler replicated       2/2  
0697sstia35s nerd-dinner_nerd-dinner-save-handler  replicated       3/3
yq6c51bzrnrf nerd-dinner_nerd-dinner-web           replicated       2/2 
kchmkm86wk7d nerd-dinner_reverse-proxy             replicated       1/1 
```

集群已经创建了新的容器来替换在`win2019-02`上运行的副本。实际上，现在所有的副本都在单个节点上运行，但通过入口网络和 VIP 负载平衡，应用程序仍然以相同的方式工作。Docker Engine 仍然以排水模式运行，因此如果任何外部流量到达排水节点，它们仍然会将其转发到活动节点上的容器。

处于排水模式的节点被视为不可用，因此如果群需要安排新任务，则不会分配任何任务给排水节点。`win-node02`现在有效地停用了，所以我可以登录并使用`sconfig`工具运行 Windows 更新，或者更新 Docker Engine。

更新节点可能意味着重新启动 Docker Engine 或重新启动服务器。完成后，我可以使用另一个`docker node update`命令将服务器重新上线到群中：

```
docker node update --availability active win2019-02
```

这使得节点再次可用。当节点加入群时，Docker 不会自动重新平衡运行的服务，因此所有容器仍然留在`win2019-dev02`上，即使`win-node02`再次可用并且容量更大。

在高吞吐量环境中，服务经常启动、停止和扩展，加入群的任何节点很快就会运行其份额的任务。在更静态的环境中，您可以通过运行 Docker 服务`update --force`来手动重新平衡服务。这不会更改服务的配置，但它会替换所有副本，并在安排新容器运行时使用所有活动节点。

这是一种破坏性的行为，因为它迫使 Docker 停止健康的容器。您需要确信如果强制重新平衡不会影响应用程序的可用性。Docker 无法保证不知道您的应用程序的架构，这就是为什么当节点加入群时服务不会自动重新平衡。

Swarm 模式使您有权更新应用程序的任何组件和运行群的节点，而无需任何停机时间。在更新期间，您可能需要在群中委托额外的节点，以确保您有足够的容量来覆盖被停用的节点，但之后可以将其移除。您无需任何额外的工具即可进行滚动更新、自动回滚和路由到健康容器——这一切都内置在 Docker 中。

# 混合主机在混合群中

Swarm 模式的另一个功能使其非常强大。群中的节点使用 Docker API 进行通信，而 API 是跨平台的，这意味着您可以在单个群中运行混合的 Windows 和 Linux 服务器。Docker 还可以在不同的 CPU 架构上运行，因此您可以将传统的 64 位 Intel 服务器与高效的新 ARM 板混合使用。

Linux 不是本书的重点，但我会简要介绍混合群集，因为它们开启了新的可能性范围。混合群集可以将 Linux 和 Windows 节点作为管理节点和工作节点。您可以使用完全相同的 Docker CLI 以相同的方式管理节点和它们运行的服务。

混合群集的一个用例是在 Linux 上运行您的管理节点，以减少许可成本或如果您的群集在云中运行则减少运行成本。生产群集将需要至少三个管理节点。即使您的所有工作负载都是基于 Windows 的，也可能更具成本效益地运行 Linux 节点作为管理节点 - 如果有这个选项的话 - 并将 Windows 节点保留给用户工作负载。

另一个用例是用于混合工作负载。我的 NerdDinner 解决方案使用的是作为 Linux Docker 镜像可用的开源软件，但我不得不自己为 Windows Server 2019 容器打包。我可以将任何跨平台组件迁移到混合群集中的 Linux 容器中运行。这可能是来自第五章的.NET Core 组件，以及 Traefik、NATS 消息队列、Elasticsearch、Kibana，甚至 SQL Server。Linux 镜像通常比 Windows 镜像小得多，更轻巧，因此您应该能够以更高的密度运行，将更多的容器打包到每个主机上。

混合群集的巨大好处在于，您可以以相同的方式从相同的用户界面管理所有这些组件。您可以将本地的 Docker CLI 连接到群集管理器，并使用完全相同的命令管理 Linux 上的 Traefik 代理和 Windows 上的 ASP.NET 应用程序。

# 总结

本章主要介绍了 Docker Swarm 模式，这是内置在 Docker 中的本地集群选项。您学会了如何创建一个群集，如何添加和删除群集节点，以及如何在连接了覆盖网络的群集上部署服务。我展示了您必须为高可用性创建服务，并讨论了如何使用配置和秘密在群集中安全存储敏感的应用程序数据。

您可以使用 Compose 文件将应用程序部署为群集上的堆栈，这样可以非常容易地对应用程序组件进行分组和管理。我演示了在单节点群集和多节点群集上的堆栈部署 - 对于具有数百个节点的群集，流程是相同的。

Docker Swarm 中的高可用性意味着您可以在没有停机时间的情况下执行应用程序更新和回滚。甚至在需要更新 Windows 或 Docker 时，您也可以将节点停用，仍然可以在剩余节点上以相同的服务水平运行您的应用程序。

在下一章中，我将更仔细地研究 docker 化解决方案的管理选项。我将首先看看如何使用现有的管理工具来管理在 Docker 中运行的应用程序。然后，我将继续使用 Docker Enterprise 在生产环境中管理 swarms。

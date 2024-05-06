# 第十四章：Fargate 和 ECS 服务发现

到目前为止，在本书中，我们已经花了大量时间专注于构建支持您的 ECS 集群的基础架构，详细介绍了如何为 ECS 容器实例构建自定义的 Amazon 机器映像，以及如何创建可以动态添加或删除 ECS 容器实例到 ECS 集群的 EC2 自动扩展组，还有专门用于管理集群的生命周期和容量的章节。

想象一下不必担心 ECS 集群和 ECS 容器实例。想象一下，有人为您管理它们，以至于您甚至真的不知道它们的存在。对于某些用例，对硬件选择、存储配置、安全姿态和其他基础设施相关问题具有强大的控制能力非常重要；到目前为止，您应该对 ECS 如何提供这些功能有相当深入的了解。然而，在许多情况下，不需要那种控制水平，并且能够利用一个管理您的 ECS 集群补丁、安全配置、容量和其他一切的服务将会带来显著的好处，降低您的运营开销，并使您能够专注于实现您的组织正在努力实现的目标。

好消息是，这是完全可能的，这要归功于一个名为**AWS Fargate**的服务，该服务于 2017 年 12 月推出。Fargate 是一个完全托管的服务，您只需定义 ECS 任务定义和 ECS 服务，然后让 Fargate 来处理本书中您已经习惯的 ECS 集群和容器实例管理的其余部分。在本章中，您将学习如何使用 AWS Fargate 部署容器应用程序，使用我们在本书中一直在采用的 CloudFormation 的**基础设施即代码**（**IaC**）方法。为了使本章更加有趣，我们将为名为 X-Ray 的 AWS 服务添加支持，该服务为在 AWS 中运行的应用程序提供分布式跟踪。

当您想要在容器应用程序中使用 X-Ray 时，您需要实现所谓的 X-Ray 守护程序，这是一个从容器应用程序收集跟踪信息并将其发布到 X-Ray 服务的应用程序。我们将扩展 todobackend 应用程序以捕获传入请求的跟踪信息，并通过利用 AWS Fargate 服务向您的 AWS 环境添加 X-Ray 守护程序，该服务将收集跟踪信息并将其转发到 X-Ray 服务。

作为额外的奖励，我们还将实现一个名为 ECS 服务发现的功能，它允许您的容器应用程序自动发布和发现，使用 DNS。这个功能对于 X-Ray 守护程序非常有用，它是一个基于 UDP 的应用程序，不能由各种可用于前端 TCP 和基于 HTTP 的应用程序的负载平衡服务提供服务。ECS 包含对服务发现的内置支持，负责在您的 ECS 任务启动和停止时进行服务注册和注销，使您能够创建其他应用程序可以轻松发现的高可用服务。

本章将涵盖以下主题：

+   何时使用 Fargate

+   为应用程序添加对 AWS X-Ray 的支持

+   创建 X-Ray 守护程序 Docker 镜像

+   配置 ECS 服务发现资源

+   为 Fargate 配置 ECS 任务定义

+   为 Fargate 配置 ECS 服务

+   部署和测试 X-Ray 守护程序

# 技术要求

本章的技术要求如下：

+   对 AWS 帐户的管理员访问权限

+   本地 AWS 配置文件，根据第三章的说明进行配置

+   AWS CLI 版本 1.15.71 或更高版本

+   Docker 18.06 CE 或更高版本

+   Docker Compose 1.22 或更高版本

+   GNU Make 3.82 或更高版本

+   本章继续自第十三章，因此需要您成功完成第十三章中定义的所有配置任务

以下 GitHub URL 包含本章中使用的代码示例：[`github.com/docker-in-aws/docker-in-aws/tree/master/ch14`](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch14)

查看以下视频以查看代码的实际操作：

[`bit.ly/2Lyd9ft`](http://bit.ly/2Lyd9ft)

# 何时使用 Fargate？

正如本章介绍中所讨论的，AWS Fargate 是一项服务，允许您部署基于容器的应用程序，而无需部署任何 ECS 容器实例、自动扩展组，或者与管理 ECS 集群基础设施相关的任何操作要求。这使得 AWS Fargate 成为一个介于使用 AWS Lambda 运行函数作为服务和使用传统 ECS 集群和 ECS 容器实例运行自己基础设施之间的无服务器技术。

尽管 Fargate 是一项很棒的技术，但重要的是要了解，Fargate 目前还很年轻（至少在撰写本书时是这样），它确实存在一些限制，这些限制可能使其不适用于某些用例，如下所述：

+   **无持久存储**：Fargate 目前不支持持久存储，因此如果您的应用程序需要使用持久的 Docker 卷，您应该使用其他服务，例如传统的 ECS 服务。

+   **定价**：定价始终可能会有变化；然而，与 ECS 一起获得的常规 EC2 实例定价相比，Fargate 的初始定价被许多人认为是昂贵的。例如，您可以购买的最小 Fargate 配置为 0.25v CPU 和 512 MB 内存，价格为每月 14.25 美元。相比之下，具有 0.5v CPU 和 512 MB 内存的 t2.nano 的价格要低得多，为 4.75 美元（所有价格均基于“us-east-1”地区）。

+   **部署时间**：就我个人经验而言，在 Fargate 上运行的 ECS 任务通常需要更长的时间来进行配置和部署，这可能会影响您的应用程序部署所需的时间（这也会影响自动扩展操作）。

+   **安全和控制**：使用 Fargate，您无法控制运行容器的底层硬件或实例的任何内容。如果您有严格的安全和/或合规性要求，那么 Fargate 可能无法为您提供满足特定要求的保证或必要的控制。然而，重要的是要注意，AWS 将 Fargate 列为符合 HIPAA 和 PCI Level 1 DSS 标准。

+   网络隔离：在撰写本书时，Fargate 不支持 ECS 代理和 CloudWatch 日志通信使用 HTTP 代理。这要求您将 Fargate 任务放置在具有互联网连接性的公共子网中，或者放置在具有 NAT 网关的私有子网中，类似于您在“隔离网络访问”章节中学到的方法。为了允许访问公共 AWS API 端点，这确实要求您打开出站网络访问，这可能违反您组织的安全要求。

+   服务可用性：在撰写本书时，Fargate 仅在美国东部（弗吉尼亚州）、美国东部（俄亥俄州）、美国西部（俄勒冈州）和欧盟（爱尔兰）地区可用；但是，我希望 Fargate 能够在大多数地区迅速广泛地可用。

如果您可以接受 Fargate 当前的限制，那么 Fargate 将显著减少您的运营开销，并使您的生活更加简单。例如，在自动扩展方面，您可以简单地使用我们在“ECS 自动扩展”章节末尾讨论的应用自动扩展方法来自动扩展您的 ECS 服务，Fargate 将负责确保有足够的集群容量。同样，您无需担心 ECS 集群的打补丁和生命周期管理 - Fargate 会为您处理上述所有事项。

在本章中，我们将部署一个 AWS X-Ray 守护程序服务，以支持 todobackend 应用程序的应用程序跟踪。鉴于这种类型的服务是 Fargate 非常适合的，因为它是一个不需要持久存储、不会影响 todobackend 应用程序的可用性（如果它宕机），也不会处理最终用户数据的后台服务。

# 为应用程序添加对 AWS X-Ray 的支持

在我们可以使用 AWS X-Ray 服务之前，您的应用程序需要支持收集和发布跟踪信息到 X-Ray 服务。X-Ray 软件开发工具包（SDK）包括对各种编程语言和流行的应用程序框架的支持，包括 Python 和 Django，它们都是 todobackend 应用程序的动力源。

您可以在[`aws.amazon.com/documentation/xray/`](https://aws.amazon.com/documentation/xray/)找到适合您选择的语言的适当 SDK 文档，但对于我们的用例，[`docs.aws.amazon.com/xray-sdk-for-python/latest/reference/frameworks.html`](https://docs.aws.amazon.com/xray-sdk-for-python/latest/reference/frameworks.html)提供了有关如何配置 Django 以自动为应用程序的每个传入请求创建跟踪的相关信息。

在 todobackend 存储库中，您首先需要将 X-Ray SDK 包添加到`src/requirements.txt`文件中，这将确保 SDK 与 todobackend 应用程序的其他依赖项一起安装：

[PRE0]

接下来，您需要将 Django X-Ray 中间件组件（包含在 SDK 中）添加到位于`src/todobackend/settings_release.py`中的 Django 项目的`MIDDLEWARE`配置元素中：

[PRE1]

这种配置与[Django 的 X 射线文档](https://docs.aws.amazon.com/xray-sdk-for-python/latest/reference/frameworks.html)有所不同，但通常情况下，您只想在 AWS 环境中运行 X-Ray，并且使用标准方法可能会导致本地开发环境中的 X-Ray 配置问题。因为我们有一个单独的发布设置文件，导入基本设置文件，我们可以简单地使用`insert()`函数将 X-Ray 中间件组件插入到基本的`MIDDLEWARE`列表的开头，如所示。这种方法确保我们将在使用发布设置的 AWS 环境中运行 X-Ray，但不会在本地开发环境中使用 X-Ray。

重要的是要在`MIDDLEWARE`列表中首先指定 X-Ray 中间件组件，因为这样可以确保 X-Ray 可以在任何其他中间件组件之前开始跟踪传入请求。

最后，Python X-Ray SDK 包括对许多流行软件包的跟踪支持，包括`mysql-connector-python`软件包，该软件包被 todobackend 应用程序用于连接其 MySQL 数据库。在 Python 中，X-Ray 使用一种称为 patching 的技术来包装受支持软件包的调用，这允许 X-Ray 拦截软件包发出的调用并捕获跟踪信息。对于我们的用例，对`mysql-connector-python`软件包进行 patching 将使我们能够跟踪应用程序发出的数据库调用，这对于解决性能问题非常有用。要对此软件包进行 patching，您需要向应用程序入口点添加几行代码，对于 Django 来说，该入口点位于文件`src/todobackend.wsgi.py`中：

[PRE2]

`xray_recorder`配置将向每个跟踪段添加服务名称，否则您将观察到 SegmentNameMissingException 错误。在这一点上，您已经在应用程序级别上添加了支持以开始跟踪传入请求，并且在提交并将更改推送到 GitHub 之前，您应该能够成功运行“make workflow”（运行`make test`和`make release`）。因为您现在已经建立了一个持续交付管道，这将触发该管道，该管道确保一旦管道构建阶段完成，您的应用程序更改将被发布到 ECR。如果您尚未完成上一章，或者已删除管道，则需要在运行`make test`和`make release`后使用`make login`和`make publish`命令手动发布新镜像。

# 创建 X-Ray 守护程序 Docker 镜像

在我们的应用程序可以发布 X-Ray 跟踪信息之前，您必须部署一个 X-Ray 守护程序，以便您的应用程序可以将此信息发送到它。我们的目标是使用 AWS Fargate 运行 X-Ray 守护程序，但在此之前，我们需要创建一个将运行守护程序的 Docker 镜像。AWS 提供了如何构建 X-Ray 守护程序镜像的示例，我们将按照 AWS 文档中记录的类似方法创建一个名为`Dockerfile.xray`的文件，该文件位于`todobackend-aws`存储库的根目录中：

[PRE3]

您现在可以使用`docker build`命令在本地构建此镜像，如下所示：

[PRE4]

现在我们的镜像已构建，我们需要将其发布到 ECR。在此之前，您需要为 X-Ray 镜像创建一个新的存储库，然后将其添加到`todobackend-aws`存储库的根目录中的现有`ecr.yml`文件中：

[PRE5]

在前面的示例中，您使用名称`docker-in-aws/xray`创建了一个新的存储库，这将导致一个完全合格的存储库名称为`<account-id>.dkr.ecr.<region>.amazonaws.com/docker-in-aws/xray`（例如，`385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/xray`）。

您现在可以通过运行`aws cloudformation deploy`命令来创建新的存储库：

[PRE6]

部署完成后，您可以登录到 ECR，然后使用新的 ECR 存储库的完全合格名称对之前创建的图像进行标记和发布。

[PRE7]

# 配置 ECS 服务发现资源

ECS 服务发现是一个功能，允许您的客户端应用程序在动态环境中发现 ECS 服务，其中基于容器的端点来来去去。到目前为止，我们已经使用 AWS 应用程序负载均衡器来执行此功能，您可以配置一个稳定的服务端点，您的应用程序可以连接到该端点，然后在 ECS 管理的目标组中进行负载平衡，该目标组包括与 ECS 服务相关联的每个 ECS 任务。尽管这通常是我推荐的最佳实践方法，但对于不支持负载均衡器的应用程序（例如，基于 UDP 的应用程序），或者对于非常庞大的微服务架构，在这种架构中，与给定的 ECS 任务直接通信更有效，ECS 服务发现可能比使用负载均衡器更好。

ECS 服务发现还支持 AWS 负载均衡器，如果负载均衡器与给定的 ECS 服务相关联，ECS 将发布负载均衡器侦听器的 IP 地址。

ECS 服务发现使用 DNS 作为其发现机制，这是有用的，因为在其最基本的形式中，DNS 被任何应用客户端普遍支持。您的 ECS 服务注册的 DNS 命名空间被称为**服务发现命名空间**，它简单地对应于 Route 53 DNS 域或区域，您在命名空间中注册的每个服务被称为**服务发现**。例如，您可以将`services.dockerinaws.org`配置为服务发现命名空间，如果您有一个名为`todobackend`的 ECS 服务，那么您将使用 DNS 名称`todobackend.services.dockerinaws.org`连接到该服务。ECS 将自动管理针对您的服务的 DNS 记录注册的地址（`A`）记录，动态添加与您的 ECS 服务的每个活动和健康的 ECS 任务关联的 IP 地址，并删除任何退出或变得不健康的 ECS 任务。ECS 服务发现支持公共和私有命名空间，对于我们运行 X-Ray 守护程序的示例，私有命名空间是合适的，因为此服务只需要支持来自 todobackend 应用程序的内部应用程序跟踪通信。

ECS 服务发现支持 DNS 服务（SRV）记录的配置，其中包括有关给定服务端点的 IP 地址和 TCP/UDP 端口信息。当使用静态端口映射或**awsvpc**网络模式（例如 Fargate）时，通常使用地址（`A`）记录，当使用动态端口映射时使用 SRV 记录，因为 SRV 记录可以包括为创建的端口映射提供动态端口信息。请注意，应用程序对 SRV 记录的支持有些有限，因此我通常建议在 ECS 服务发现中使用经过验证的`A`记录的方法。

# 配置服务发现命名空间

与大多数 AWS 资源一样，您可以使用 AWS 控制台、AWS CLI、各种 AWS SDK 之一或 CloudFormation 来配置服务发现资源。鉴于本书始终采用基础设施即代码的方法，我们自然会在本章中采用 CloudFormation；因为 X-Ray 守护程序是一个新服务（通常被视为每个应用程序发布跟踪信息的共享服务），我们将在名为`xray.yml`的文件中创建一个新的堆栈，放在`todobackend-aws`存储库的根目录。

以下示例演示了创建初始模板和创建服务发现命名空间资源：

[PRE8]

在前面的示例中，我们创建了一个私有服务发现命名空间，它只需要命名空间的 DNS 名称、可选描述和关联的私有 Route 53 区域的 VPC ID。为了保持简单，我还硬编码了与我的 AWS 账户相关的 VPC ID 的适当值，通常您会通过堆栈参数注入这个值。

鉴于服务发现命名空间的意图是支持多个服务，您通常会在单独的 CloudFormation 堆栈中创建命名空间，比如创建共享网络资源的专用网络堆栈。然而，为了保持简单，我们将在 X-Ray 堆栈中创建命名空间。

现在，您可以使用`aws cloudformation deploy`命令将初始堆栈部署到 CloudFormation，这应该会创建一个服务发现命名空间和相关的 Route 53 私有区域。

[PRE9]

在前面的示例中，一旦您的堆栈成功部署，您将使用`aws servicediscovery list-namespaces`命令来验证是否创建了一个私有命名空间，而`aws route53 list-hosted-zones`命令将显示已创建一个 Route 53 区域，其区域名称为`services.dockerinaws.org`。

# 配置服务发现服务

现在您已经有了一个服务发现命名空间，下一步是创建一个服务发现服务，它与每个 ECS 服务都有一对一的关系，这意味着您需要创建一个代表稍后在本章中创建的 X-Ray ECS 服务的服务发现服务。

[PRE10]

在前面的示例中，您添加了一个名为`ApplicationServiceDiscoveryService`的新资源，并配置了以下属性：

+   `Name`：定义服务的名称。此名称将用于在关联的命名空间中注册服务。

+   `DnsConfig`：指定服务关联的命名空间（由`NamespaceId`属性定义），并定义应创建的 DNS 记录类型和生存时间（TTL）。在这里，您指定了一个地址记录（类型为`A`）和一个 60 秒的 TTL，这意味着客户端只会缓存该记录最多 60 秒。通常情况下，您应该将 TTL 设置为较低的值，以确保您的客户端在新的 ECS 任务注册到服务或现有的 ECS 任务从服务中移除时能够获取 DNS 更改。

+   `HealthCheckCustomConfig`：这配置 ECS 来管理确定是否可以注册 ECS 任务的健康检查。您还可以配置 Route 53 健康检查（参见[`docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html#service-discovery-concepts`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html#service-discovery-concepts)）；然而，对于我们的用例来说，鉴于 X-Ray 是基于 UDP 的应用程序，而 Route 53 健康检查仅支持基于 TCP 的服务，您必须使用前面示例中显示的`HealthCheckCustomConfig`配置。`FailureThreshold`指定服务发现在接收到自定义健康检查更新后等待更改给定服务实例的健康状态的`30`秒间隔数。

您现在可以使用`aws cloudformation deploy`命令将更新后的堆栈部署到 CloudFormation，这应该会创建一个服务发现服务。

[PRE11]

这将为`xray.services.dockerinaws.org`创建一个 DNS 记录集，直到我们在本章后面将要创建的 X-Ray ECS 服务的 ECS 服务发现支持配置之前，它将不会有任何地址（`A`）记录与之关联。

# 为 Fargate 配置 ECS 任务定义

您现在可以开始定义您的 ECS 资源，您将配置为使用 AWS Fargate 服务，并利用您在上一节中创建的服务发现资源。

在配置 ECS 任务定义以支持 Fargate 时，有一些关键考虑因素需要您了解：

+   **启动类型**：ECS 任务定义包括一个名为`RequiresCompatibilities`的参数，该参数定义了定义的兼容启动类型。当前的启动类型包括 EC2，指的是在传统 ECS 集群上启动的 ECS 任务，以及 FARGATE，指的是在 Fargate 上启动的 ECS 任务。默认情况下，`RequiresCompatibilities`参数配置为 EC2，这意味着如果要使用 Fargate，必须显式配置此参数。

+   **网络模式**：Fargate 仅支持`awsvpc`网络模式，我们在第十章“隔离网络访问”中讨论过。

+   **执行角色**：Fargate 要求您配置一个**执行角色**，这是分配给管理 ECS 任务生命周期的 ECS 代理和 Fargate 运行时的 IAM 角色。这是一个独立的角色，不同于您在第九章“管理机密”中配置的任务 IAM 角色功能，该功能用于向在 ECS 任务中运行的应用程序授予 IAM 权限。执行角色通常配置为具有类似权限的权限，这些权限您将为与传统 ECS 容器实例关联的 EC2 IAM 实例角色配置，至少授予 ECS 代理和 Fargate 运行时从 ECR 拉取图像和将日志写入 CloudWatch 日志的权限。

+   **CPU 和内存**：Fargate 要求您在任务定义级别定义 CPU 和内存要求，因为这决定了基于您的任务定义运行的 ECS 任务的基础目标实例。请注意，这与您在第八章“使用 ECS 部署应用程序”中为 todobackend 应用程序的 ECS 任务定义配置的每个容器定义 CPU 和内存设置是分开的；您仍然可以配置每个容器定义的 CPU 和内存设置，但需要确保分配给容器定义的总 CPU/内存不超过分配给 ECS 任务定义的总 CPU/内存。Fargate 目前仅支持有限的 CPU/内存分配，您可以在[`docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html)的“任务 CPU 和内存”部分了解更多信息。

+   **日志记录**：截至撰写本文时，Fargate 仅支持`awslogs`日志记录驱动程序，该驱动程序将您的容器日志转发到 CloudWatch 日志。

考虑到上述情况，现在让我们为我们的 X-Ray 守护程序服务定义一个任务定义：

[PRE12]

在上面的示例中，请注意`RequiresCompatibilities`参数指定`FARGATE`作为支持的启动类型，并且`NetworkMode`参数配置为所需的`awsvpc`模式。`Cpu`和`Memory`设置分别配置为 256 CPU 单位（0.25 vCPU）和 512 MB，这代表了最小可用的 Fargate CPU/内存配置。对于`ExecutionRoleArn`参数，您引用了一个名为`ApplicationTaskExecutionRole`的 IAM 角色，我们将很快单独配置，与为`TaskRoleArn`参数配置的角色分开。

接下来，您定义一个名为`xray`的单个容器定义，该容器定义引用了您在本章前面创建的 ECR 存储库；请注意，您为`Command`参数指定了`-o`标志。这将在您在上一个示例中配置的 X-Ray 守护程序镜像的`ENTRYPOINT`指令中附加`-o`，从而阻止 X-Ray 守护程序尝试查询 EC2 实例元数据，因为在使用 Fargate 时不支持这一操作。

容器定义的日志配置配置为使用`awslogs`驱动程序，这是 Fargate 所需的，它引用了任务定义下配置的`ApplicationLogGroup` CloudWatch 日志组资源。最后，您指定了 X-Ray 守护程序端口（`UDP 端口 2000`）作为容器端口映射，并配置了一个名为`AWS_REGION`的环境变量，该变量引用您部署堆栈的区域，这对于 X-Ray 守护程序确定守护程序应将跟踪数据发布到的区域性 X-Ray 服务端点是必需的。

# 为 Fargate 配置 IAM 角色

在上一个示例中，您的 ECS 任务定义引用了一个任务执行角色（由`ExecutionRoleArn`参数定义）和一个任务角色（由`TaskRoleArn`参数定义）。

如前所述，任务执行角色定义了将分配给 ECS 代理和 Fargate 运行时的 IAM 权限，通常包括拉取任务定义中定义的容器所需的 ECR 镜像的权限，以及写入容器日志配置中引用的 CloudWatch 日志组的权限：

[PRE13]

任务角色定义了从您的 ECS 任务定义中运行的应用程序可能需要的任何 IAM 权限。对于我们的用例，X-Ray 守护程序需要权限将跟踪发布到 X-Ray 服务，如下例所示：

[PRE14]

在前面的例子中，您授予`xray:PutTraceSegments`和`xray:PutTelemetryRecords`权限给 X-Ray 守护程序，这允许守护程序将从您的应用程序捕获的应用程序跟踪发布到 X-Ray 服务。请注意，对于`ApplicationTaskExecutionRole`和`ApplicationTaskRole`资源，`AssumeRolePolicyDocument`部分中的受信任实体必须配置为`ecs-tasks.amazonaws.com`服务。

# 为 Fargate 配置 ECS 服务

现在您已经为 Fargate 定义了一个 ECS 任务定义，您可以创建一个 ECS 服务，该服务将引用您的 ECS 任务定义，并为您的服务部署一个或多个实例（ECS 任务）。

正如您可能期望的那样，在配置 ECS 服务以支持 Fargate 时，有一些关键考虑因素需要您注意：

+   **启动类型**：您必须指定 Fargate 作为任何要使用 Fargate 运行的 ECS 服务的启动类型。

+   **平台版本**：AWS 维护不同版本的 Fargate 运行时或平台，这些版本会随着时间的推移而发展，并且可能在某个时候为您的 ECS 服务引入破坏性更改。您可以选择为您的 ECS 服务针对特定的平台版本，或者简单地省略配置此属性，以使用最新可用的平台版本。

+   **网络配置**：因为 Fargate 需要使用**awsvpc**网络模式，您的 ECS 服务必须定义一个网络配置，定义您的 ECS 服务将在其中运行的子网，分配给您的 ECS 服务的安全组，以及您的服务是否分配了公共 IP 地址。在撰写本书时，当使用 Fargate 时，您必须分配公共 IP 地址或使用 NAT 网关，如章节*隔离网络访问*中所讨论的，以确保管理您的 ECS 服务的 ECS 代理能够与 ECS 通信，从 ECR 拉取镜像，并将日志发布到 CloudWatch 日志服务。

尽管您无法与 ECS 代理进行交互，但重要的是要理解所有 ECS 代理通信都使用与在 Fargate 中运行的容器应用程序相同的网络接口。这意味着您必须考虑 ECS 代理和 Fargate 运行时的通信需求，当附加安全组并确定您的 ECS 服务的网络放置时。

以下示例演示了为 Fargate 和 ECS 服务发现配置 ECS 服务：

[PRE15]

在前面的示例中，首先要注意的是，尽管在使用 Fargate 时您不运行任何 ECS 容器实例或其他基础设施，但在为 Fargate 配置 ECS 服务时仍需要定义一个 ECS 集群，然后在您的 ECS 服务中引用它。

ECS 服务配置类似于在*隔离网络访问*章节中使用 ECS 任务网络运行 todobackend 应用程序时定义的配置，尽管有一些关键的配置属性需要讨论：

+   `LaunchType`：必须指定为`FARGATE`。确保将您的 ECS 服务放置在公共子网中，并在网络配置中将`AssignPublicIp`属性配置为`ENABLED`，或者将您的服务放置在带有 NAT 网关的私有子网中非常重要。在前面的示例中，请注意我已经将`Subnets`属性硬编码为我的 VPC 中的公共子网；您需要将这些值更改为您的环境的适当值，并且通常会通过堆栈参数注入这些值。

+   `ServiceRegistries`：此属性配置您的 ECS 服务以使用我们在本章前面配置的 ECS 服务发现功能，在这里，您引用了您在上一个示例中配置的服务发现服务的 ARN。有了这个配置，ECS 将自动在为链接的服务发现服务创建的 DNS 记录集中注册/注销每个 ECS 服务实例（ECS 任务）的 IP 地址。

在这一点上，还有一个最终需要配置的资源——您需要定义被您的 ECS 服务引用的`ApplicationSecurityGroup`资源：

[PRE16]

在上面的示例中，再次注意，我在这里使用了硬编码的值，而我通常会使用堆栈参数，以保持简单和简洁。安全组允许从 VPC 内的任何主机对 UDP 端口 2000 进行入口访问，而出口安全规则允许访问 DNS、HTTP 和 HTTPS，这是为了确保 ECS 代理可以与 ECS、ECR 和 CloudWatch 日志进行通信，以及 X-Ray 守护程序可以与 X-Ray 服务进行通信。

# 部署和测试 X-Ray 守护程序

此时，我们已经完成了配置 CloudFormation 模板的工作，该模板将使用启用了 ECS 服务发现的 Fargate 服务将 X-Ray 守护程序部署到 AWS；您可以使用`aws cloudformation deploy`命令将更改部署到您的堆栈中，包括`--capabilities`参数，因为我们的堆栈现在正在创建 IAM 资源：

[PRE17]

一旦部署完成，如果您在 AWS 控制台中打开 ECS 仪表板并选择集群，您应该会看到一个名为 xray-daemon-cluster 的新集群，其中包含一个单一服务和两个正在运行的任务，在 FARGATE 部分：

![](img/c46789ab-3cb2-4a26-b6bc-229781c40131.png)X-Ray 守护程序集群

如果您选择集群并单击**xray-daemon-application-service**，您应该在“详细信息”选项卡中看到 ECS 服务发现配置已经就位：

![](img/b005f475-d3a7-4e40-a393-cf610c7882a4.png)X-Ray 守护程序服务详细信息

在服务发现命名空间中，您现在应该找到附加到`xray.services.dockerinaws.org`记录集的两个地址记录，您可以通过导航到 Route 53 仪表板，从左侧菜单中选择托管区域，并选择`services.dockerinaws.org`区域来查看：

![](img/fdcda94e-f9c6-49e2-87b9-b843c2465b05.png)服务发现 DNS 记录

请注意，这里有两个`A`记录，每个支持我们的 ECS 服务的 ECS 任务一个。如果您停止其中一个 ECS 任务，ECS 将自动从 DNS 中删除该记录，然后在 ECS 将 ECS 服务计数恢复到所需计数并启动替换的 ECS 任务后，添加一个新的`A`记录。这确保了您的服务具有高可用性，并且依赖于您的服务的应用程序可以动态解析适当的服务实例。

# 为 X-Ray 支持配置 todobackend 堆栈

有了我们的 X 射线守护程序服务，我们现在可以为`todobackend-aws`堆栈添加对 X 射线的支持。在本章的开头，您配置了 todobackend 应用程序对 X 射线的支持，如果您提交并推送了更改，您在上一章中创建的持续交付流水线应该已经将更新的 Docker 镜像发布到了 ECR（如果不是这种情况，请在 todobackend 存储库中运行`make publish`命令）。您需要执行的唯一其他配置是更新附加到 todobackend 集群实例的安全规则，以允许 X 射线通信，并确保 Docker 环境配置了适当的环境变量，以启用正确的 X 射线操作。

以下示例演示了在`todobackend-aws`堆栈中的`ApplicationAutoscalingSecurityGroup`资源中添加安全规则，该规则允许与 X 射线守护程序进行通信：

[PRE18]

以下示例演示了为`ApplicationTaskDefinition`资源中的 todobackend 容器定义配置环境设置：

[PRE19]

在前面的示例中，您添加了一个名为`AWS_XRAY_DAEMON_ADDRESS`的变量，该变量引用了我们的 X 射线守护程序服务的`xray.services.dockerinaws.org`服务端点，并且必须以`<hostname>:<port>`的格式表示。

您可以通过设置`AWS_XRAY_TRACE_NAME`环境变量来覆盖 X 射线跟踪中使用的服务名称。在我们的场景中，我们在同一帐户中有 todobackend 应用程序的开发和生产实例，并希望确保每个应用程序环境都有自己的跟踪集。

如果您现在提交并推送所有更改到`todobackend-aws`存储库，则上一章的持续交付流水线应该会检测到更改并自动部署您的更新堆栈，或者您可以通过命令行运行`make deploy/dev`命令来部署更改。

# 测试 X 射线服务

在成功部署更改后，浏览到您环境的 todobackend URL，并与应用程序进行一些交互，例如添加一个`todo`项目。

如果您接下来从 AWS 控制台打开 X 射线仪表板（服务|开发人员工具|X 射线）并从左侧菜单中选择服务地图，您应该会看到一个非常简单的地图，其中包括 todobackend 应用程序。

![](img/51bba37a-3dbf-4f06-8d4f-ecc168183a1c.png)X-Ray 服务地图

在上述截图中，我点击了 todobackend 服务，显示了右侧的服务详情窗格，显示了响应时间分布和响应状态响应等信息。另外，请注意，服务地图包括 todobackend RDS 实例，因为我们在本章的前一个示例中配置了应用程序以修补`mysql-connector-python`库。

如果您点击“查看跟踪”按钮，将显示该服务的跟踪；请注意，Django 的 X-Ray 中间件包括 URL 信息，允许根据 URL 对跟踪进行分组：

![](img/d25b61ac-61ec-4621-8b3d-99a298865260.png)X-Ray 跟踪

在上述截图中，请注意 85%的跟踪都命中了一个 IP 地址 URL，这对应于正在进行的应用程序负载均衡器健康检查。如果您点击跟踪列表中的“年龄”列，以从最新到最旧对跟踪进行排序，您应该能够看到您对 todobackend 应用程序所做的请求，对我来说，是一个创建新的`todo`项目的`POST`请求。

您可以通过点击 ID 链接查看以下截图中`POST`跟踪的更多细节：

![](img/9eaf6820-622f-422a-bbc3-6c5e41b1d710.png)X-Ray 跟踪详情

在上述截图中，您可以看到响应总共花费了 218 毫秒，并且进行了两次数据库调用，每次调用都少于 2 毫秒。如果您正在使用 X-Ray SDK 支持的其他库，您还可以看到这些库所做调用的跟踪信息；例如，通过 boto3 库进行的任何 AWS 服务调用，比如将文件复制到 S3 或将消息发布到 Kinesis 流，也会被捕获。显然，这种信息在排除应用程序性能问题时非常有用。

# 摘要

在本章中，您学习了如何使用 AWS Fargate 服务部署 Docker 应用程序。为了使事情更有趣，您还学习了如何利用 ECS 服务发现自动发布应用程序端点的服务可达性信息，这是传统方法的替代方案，传统方法是将应用程序端点发布在负载均衡器后面。最后，为了结束这一定会让您觉得有趣和有趣的章节，您为 todobackend 应用程序添加了对 AWS X-Ray 服务的支持，并部署了一个 X-Ray 守护程序服务，使用 Fargate 来捕获应用程序跟踪。

首先，您学习了如何为 Python Django 应用程序添加对 X-Ray 的支持，这只需要您添加一个拦截传入请求的 X-Ray 中间件组件，并且还需要修补支持包，例如 mysql-connector-python 和 boto3 库，这允许您捕获 MySQL 数据库调用和应用程序可能进行的任何 AWS 服务调用。然后，您为 X-Ray 守护程序创建了一个 Docker 镜像，并将其发布到弹性容器注册表，以便在您的 AWS 环境中部署。

然后，您学习了如何配置 ECS 服务发现所需的支持元素，添加了一个服务发现命名空间，创建了一个公共或私有 DNS 区域，其中维护了服务发现服务端点，然后为 X-Ray 守护程序创建了一个服务发现服务，允许您的 todobackend 应用程序（以及其他应用程序）通过逻辑 DNS 名称发现所有活动和健康的 X-Ray 守护程序实例。

有了这些组件，您继续创建了一个使用 Fargate 的 X-Ray 守护程序服务，创建了一个 ECS 任务定义和一个 ECS 服务。ECS 任务定义对支持 Fargate 有一些特定要求，包括定义一个单独的任务执行角色，该角色授予基础 ECS 代理和 Fargate 运行时的特权，指定 Fargate 作为支持的启动类型，并确保配置了 awsvpc 网络模式。您创建的 ECS 服务要求您配置网络配置以支持 ECS 任务定义的 awsvpc 网络模式。您还通过引用本章早些时候创建的服务发现服务，为 ECS 服务添加了对 ECS 服务发现的支持。

最后，您在 todobackend 堆栈中配置了现有的 ECS 任务定义，以将服务发现服务名称指定为`AWS_XRAY_DAEMON_ADDRESS`变量；在部署更改后，您学会了如何使用 X-Ray 跟踪来分析传入请求到您的应用程序的性能，并能够对 todobackend 应用程序数据库的个别调用进行分析。

在下一章中，您将了解另一个支持 Docker 应用程序的 AWS 服务，称为 Elastic Beanstalk。它提供了一种平台即服务（PaaS）的方法，用于在 AWS 中部署和运行基于容器的应用程序。

# 问题

1.  Fargate 是否需要您创建 ECS 集群？

1.  在配置 Fargate 时，支持哪些网络模式？

1.  真/假：Fargate 将 ECS 代理的控制平面网络通信与 ECS 任务的数据平面网络通信分开。

1.  您使用 Fargate 部署一个新的 ECS 服务，但失败了，出现错误指示无法拉取任务定义中指定的 ECR 镜像。您验证了镜像名称和标签是正确的，并且任务定义的`TaskRoleArn`属性引用的 IAM 角色允许访问 ECR 存储库。这个错误最有可能的原因是什么？

1.  根据这些要求，您正在确定在 AWS 中部署基于容器的应用程序的最佳技术。您的组织部署 Splunk 来收集所有应用程序的日志，并使用 New Relic 来收集性能指标。基于这些要求，Fargate 是否是一种合适的技术？

1.  真/假：ECS 服务发现使用 Consul 发布服务注册信息。

1.  哪种服务发现资源创建了 Route 53 区域？

1.  您配置了一个 ECS 任务定义来使用 Fargate，并指定任务应分配 400 个 CPU 单位和 600 MB 的内存。当您部署使用任务定义的 ECS 服务时，部署失败了。您如何解决这个问题？

1.  默认情况下，AWS X-Ray 通信使用哪种网络协议和端口？

1.  真/假：当您为基于容器的应用程序添加 X-Ray 支持时，它们将发布跟踪到 AWS X-Ray 服务。

# 进一步阅读

您可以查看本章涵盖的主题的更多信息的以下链接：

+   AWS Fargate on Amazon ECS: [`docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html)

+   Amazon ECS 任务执行 IAM 角色: [`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-service.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-service.html)

+   ECS 服务发现: [`docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html)

+   AWS X-Ray 开发人员指南: [`docs.aws.amazon.com/xray/latest/devguide/aws-xray.html`](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html)

+   AWS X-Ray Python SDK: [`docs.aws.amazon.com/xray/latest/devguide/xray-sdk-python.html`](https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-python.html)

+   在 Amazon ECS 上运行 X-Ray 守护程序: [`docs.aws.amazon.com/xray/latest/devguide/xray-daemon-ecs.html`](https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon-ecs.html)

+   CloudFormation 服务发现公共命名空间资源参考: [`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-servicediscovery-publicdnsnamespace.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-servicediscovery-publicdnsnamespace.html)

+   CloudFormation 服务发现私有命名空间资源参考: [`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-servicediscovery-privatednsnamespace.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-servicediscovery-privatednsnamespace.html)

+   CloudFormation 服务发现服务资源参考: [`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-servicediscovery-service.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-servicediscovery-service.html)

+   CloudFormation ECS 任务定义资源参考: [`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-taskdefinition.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-taskdefinition.html)

+   CloudFormation ECS 服务资源参考: [`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-service.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-service.html)

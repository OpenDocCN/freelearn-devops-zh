# 第五章：在容器化应用程序之间移动

在上一章中，我们介绍了使用 Docker 容器部署微服务应用程序架构。在本章中，我们将探讨 Docker 注册表以及如何在公共和私有模式下使用它。我们还将深入探讨在使用公共和私有 Docker 注册表时出现问题时的故障排除。

我们将讨论以下主题：

+   通过 Docker 注册表重新分发

+   公共 Docker 注册表

+   私有 Docker 注册表

+   确保镜像的完整性-签名镜像

+   **Docker Trusted Registry**（**DTR**）

+   Docker 通用控制平面

# 通过 Docker 注册表重新分发

Docker 注册表是服务器端应用程序，允许用户存储和分发 Docker 镜像。默认情况下，公共 Docker 注册表（Docker Hub）可用于托管多个 Docker 镜像，提供免费使用、零维护和自动构建等附加功能。让我们详细看看公共和私有 Docker 注册表。

## Docker 公共存储库（Docker Hub）

正如前面解释的那样，Docker Hub 允许个人和组织与内部团队和客户共享 Docker 镜像，而无需维护基于云的公共存储库。它提供了集中的资源镜像发现和管理。它还为开发流水线提供了团队协作和工作流自动化。除了镜像存储库管理之外，Docker Hub 的一些附加功能如下：

+   **自动构建**：它帮助在 GitHub 或 Bitbucket 存储库中的代码更改时创建新的镜像

+   **WebHooks**：这是一个新功能，允许在成功将镜像推送到存储库后触发操作

+   **用户管理**：它允许创建工作组来管理组织对镜像存储库的用户访问

可以使用 Docker Hub 登录页面创建帐户以托管 Docker 镜像；每个帐户将与唯一的基于用户的 Docker ID 相关联。可以在不创建 Docker Hub 帐户的情况下执行基本功能，例如从 Docker Hub 进行 Docker 镜像搜索和*拉取*。可以使用以下命令浏览 Docker Hub 中存在的镜像：

[PRE0]

它将根据匹配的关键字显示 Docker Hub 中存在的镜像。

也可以使用`docker login`命令创建 Docker ID。以下命令将提示创建一个 Docker ID，该 ID 将成为用户公共存储库的公共命名空间。它将提示输入`用户名`，还将提示输入`密码`和`电子邮件`以完成注册过程：

[PRE1]

为了注销，可以使用以下命令：

[PRE2]

## 私有 Docker 注册表

私有 Docker 注册表可以部署在本地组织内；它是 Apache 许可下的开源软件，并且易于部署。

使用私有 Docker 注册表，您有以下优势：

+   组织可以控制并监视 Docker 图像存储的位置

+   完整的图像分发流程将由组织拥有

+   图像存储和分发对于内部开发工作流程以及与其他 DevOps 组件（如 Jenkins）的集成将非常有用

# 将图像推送到 Docker Hub

我们可以创建一个定制的图像，然后使用标记将其推送到 Docker Hub。让我们创建一个带有小型基于终端的应用程序的简单图像。创建一个包含以下内容的 Dockerfile：

[PRE3]

转到包含 Dockerfile 的目录并执行以下命令来构建图像：

[PRE4]

或者，如下图所示，我们可以首先创建一个容器并对其进行测试，然后创建一个带有标记的**Docker 图像**，可以轻松地推送到**Docker Hub**：

![将图像推送到 Docker Hub](img/image_05_002.jpg)

从 Docker 容器创建 Docker 图像并将其推送到公共 Docker Hub 的步骤

我们可以使用以下命令检查图像是否已创建。如您所见，`test/cowsay-dockerfile`图像已创建：

[PRE5]

为了将图像推送到 Docker Hub 帐户，我们将不得不使用图像 ID 对其进行标记，标记为 Docker 标记/Docker ID，方法如下：

[PRE6]

由于标记的用户名将与 Docker Hub ID 帐户匹配，因此我们可以轻松地推送图像：

[PRE7]

![将图像推送到 Docker Hub](img/image_05_003.jpg)

Docker Hub 的屏幕截图

### 提示

可以预先检查的故障排除问题之一是，自定义 Docker 镜像上标记的用户名应与 Docker Hub 帐户的用户名匹配，以便成功推送镜像。推送到 Docker Hub 的自定义镜像将公开可用。Docker 免费提供一个私有仓库，应该用于推送私有镜像。Docker 客户端版本 1.5 及更早版本将无法将镜像推送到 Docker Hub 帐户，但仍然可以拉取镜像。只支持 1.6 或更高版本。因此，建议始终保持 Docker 版本最新。

如果向 Docker Hub 推送失败并出现**500 内部服务器错误**，则问题与 Docker Hub 基础设施有关，重新推送可能有帮助。如果在推送 Docker 镜像时问题仍然存在，则应参考`/var/log/docker.log`中的 Docker 日志以进行详细调试。

## 安装私有本地 Docker 注册表

可以使用存在于 Docker Hub 上的镜像部署私有 Docker 注册表。映射到访问私有 Docker 注册表的端口将是`5000`：

[PRE8]

现在，我们将在前面教程中创建的相同镜像标记为`localhost:5000/cowsay-dockerfile`，以便可以轻松地将匹配的仓库名称和镜像名称推送到私有 Docker 注册表：

[PRE9]

将镜像推送到私有 Docker 注册表：

[PRE10]

推送是指一个仓库（`localhost:5000/cowsay-dockerfile`）（长度：1）：

[PRE11]

可以通过访问浏览器中的链接或使用`curl`命令来查看镜像 ID，该命令在推送镜像后会出现。

## 在主机之间移动镜像

将一个镜像从一个注册表移动到另一个注册表需要从互联网上推送和拉取镜像。如果需要将镜像从一个主机移动到另一个主机，那么可以简单地通过`docker save`命令来实现，而不需要上传和下载镜像。Docker 提供了两种不同的方法来将容器镜像保存为 tar 包：

+   `docker export`：这将一个容器的运行或暂停状态保存到一个 tar 文件中

+   `docker save`：这将一个非运行的容器镜像保存到一个文件中

让我们通过以下教程来比较`docker export`和`docker save`命令：

使用 export，从 Docker Hub 拉取一个基本的镜像：

[PRE12]

在从上述镜像运行 Docker 容器后，让我们创建一个示例文件：

[PRE13]

在另一个 shell 中，我们可以看到正在运行的 Docker 容器，然后可以使用以下命令将其导出到 tar 文件中：

[PRE14]

然后可以将 tar 文件导出到另一台机器，然后使用以下命令导入：

[PRE15]

在另一台机器上从 Ubuntu-sample 图像运行容器后，我们可以发现示例文件完好无损。

[PRE16]

使用 save 命令，以便在运行 Docker 容器的情况下传输图像，我们可以使用`docker save`命令将图像转换为 tar 文件：

[PRE17]

`ubuntu-bundle.tar.gz`文件现在可以使用`docker load`命令在另一台机器上提取并使用：

[PRE18]

在另一台机器上从`ubuntu-bundle`图像运行容器，我们会发现示例文件不存在，因为`docker load`命令将存储图像而不会有任何投诉：

[PRE19]

前面的例子都展示了导出和保存命令之间的区别，以及它们在不使用 Docker 注册表的情况下在本地主机之间传输图像的用法。

## 确保图像的完整性-签名图像

从 Docker 版本 1.8 开始，包含的功能是 Docker 容器信任，它将**The Update Framework**（**TUF**）集成到 Docker 中，使用开源工具 Notary 提供对任何内容或数据的信任。它允许验证发布者-Docker 引擎使用发布者密钥来验证，并且用户即将运行的图像确实是发布者创建的；它没有被篡改并且是最新的。因此，这是一个允许验证图像发布者的选择性功能。 Docker 中央命令-*push*，*pull*，*build*，*create*和*run-*将对具有内容签名或显式内容哈希的图像进行操作。图像在推送到存储库之前由内容发布者使用私钥进行签名。当用户第一次与图像交互时，与发布者建立了信任，然后所有后续交互只需要来自同一发布者的有效签名。该模型类似于我们熟悉的 SSH 的第一个模型。 Docker 内容信任使用两个密钥-**离线密钥**和**标记密钥**-当发布者推送图像时，它们在第一次生成。每个存储库都有自己的标记密钥。当用户第一次运行`docker pull`命令时，使用离线密钥建立了对存储库的信任：

+   离线密钥：它是您存储库的信任根源；不同的存储库使用相同的离线密钥。由于具有针对某些攻击类别的优势，应将此密钥保持离线。基本上，在创建新存储库时需要此密钥。

+   **标记密钥**：为发布者拥有的每个新存储库生成。它可以导出并与需要为特定存储库签署内容的人共享。

以下是按照信任密钥结构提供的保护列表：

+   **防止图像伪造**：Docker 内容信任可防止中间人攻击。如果注册表遭到破坏，恶意攻击者无法篡改内容并向用户提供，因为每次运行命令都会失败，显示无法验证内容的消息。

+   **防止重放攻击**：在重放攻击的情况下，攻击者使用先前的有效负载来欺骗系统。Docker 内容信任在发布图像时使用时间戳密钥，从而提供对重放攻击的保护，并确保用户接收到最新的内容。

+   **防止密钥被破坏**：由于标记密钥的在线特性，可能会遭到破坏，并且每次将新内容推送到存储库时都需要它。Docker 内容信任允许发布者透明地旋转受损的密钥，以便用户有效地将其从系统中删除。

Docker 内容信任是通过将 Notary 集成到 Docker 引擎中实现的。任何希望数字签名和验证任意内容集合的人都可以下载并实施 Notary。基本上，这是用于在分布式不安全网络上安全发布和验证内容的实用程序。在以下序列图中，我们可以看到 Notary 服务器用于验证元数据文件及其与 Docker 客户端的集成的流程。受信任的集合将存储在 Notary 服务器中，一旦 Docker 客户端具有命名哈希（标记）的受信任列表，它就可以利用客户端到守护程序的 Docker 远程 API。一旦拉取成功，我们就可以信任注册表拉取中的所有内容和层。

![确保图像的完整性-签名图像](img/image_05_004.jpg)

Docker 受信任运行的序列图

在内部，Notary 使用 TUF，这是一个用于软件分发和更新的安全通用设计，通常容易受到攻击。TUF 通过提供一个全面的、灵活的安全框架来解决这个普遍的问题，开发人员可以将其与软件更新系统集成。通常，软件更新系统是在客户端系统上运行的应用程序，用于获取和安装软件。

让我们开始安装 Notary；在 Ubuntu 16.04 上，可以直接使用以下命令安装 Notary：

[PRE20]

否则，该项目可以从 GitHub 下载并手动构建和安装；构建该项目需要安装 Docker Compose：

[PRE21]

在上述步骤之后，将`127.0.0.1` Notary 服务器添加到`/etc/hosts`中，或者如果使用 Docker 机器，则将`$(docker-machine ip)`添加到 Notary 服务器。

现在，我们将推送之前创建的`docker-cowsay`镜像。默认情况下，内容信任是禁用的；可以使用`DOCKER_CONTENT_TRUST`环境变量来启用它，这将在本教程中稍后完成。目前，操作内容信任的命令如下所示：

+   push

+   build

+   create

+   pull

+   run

我们将使用仓库名称标记该镜像：

[PRE22]

现在，让我们检查 notary 是否有这个镜像的数据：

[PRE23]

正如我们在这里所看到的，没有信任数据可以让我们启用`DOCKER_CONTENT_TRUST`标志，然后尝试推送该镜像：

[PRE24]

正如我们在这里所看到的，第一次推送时，它将要求输入密码来签署标记的镜像。

现在，我们将从 Notary 获取先前推送的最新镜像的信任数据：

[PRE25]

借助上面的例子，我们清楚地了解了 Notary 和 Docker 内容信任的工作原理。

# Docker Trusted Registry (DTR)

DTR 提供企业级的 Docker 镜像存储，可以在本地以及虚拟私有云中提供安全性并满足监管合规要求。DTR 在 Docker Universal Control Plane (UCP)之上运行，UCP 可以在本地或虚拟私有云中安装，借助它我们可以将 Docker 镜像安全地存储在防火墙后面。

![Docker Trusted Registry (DTR)](img/image_05_005.jpg)

DTR 在 UCP 节点上运行

DTR 的两个最重要的特性如下：

+   **镜像管理**：它允许用户在防火墙后安全存储 Docker 镜像，并且 DTR 可以轻松地作为持续集成和交付过程的一部分，以构建、运行和交付应用程序。![Docker 受信任的注册表（DTR）](img/image_05_006.jpg)

DTR 的屏幕截图

+   **访问控制和内置安全性**：DTR 提供身份验证机制，以添加用户，并集成了**轻量级目录访问协议**（**LDAP**）和 Active Directory。它还支持**基于角色的身份验证**（**RBAC**），允许您为每个用户分配访问控制策略。![Docker 受信任的注册表（DTR）](img/image_05_007.jpg)

DTR 中的用户身份验证选项

# Docker 通用控制平面

Docker UCP 是企业级集群管理解决方案，允许您从单个平台管理 Docker 容器。它还允许您管理数千个节点，并可以通过图形用户界面进行管理和监控。

UCP 有两个重要组件：

+   **控制器**：管理集群并保留集群配置

+   **节点**：可以添加多个节点到集群中以运行容器

可以使用 Mac OS X 或 Windows 系统上的**Docker Toolbox**进行沙盒安装 UCP。安装包括一个 UCP 控制器和一个或多个主机，这些主机将作为节点添加到 UCP 集群中，使用 Docker Toolbox。

Docker Toolbox 的先决条件是必须在 Mac OS X 和 Windows 系统上安装，使用官方 Docker 网站提供的安装程序。

![Docker 通用控制平面](img/image_05_008.jpg)

Docker Toolbox 安装

让我们开始部署 Docker UCP：

1.  安装完成后，启动 Docker Toolbox 终端：![Docker 通用控制平面](img/image_05_009.jpg)

Docker Quickstart 终端

1.  使用`docker-machine`命令和`virtualbox`创建一个名为`node1`的虚拟机，该虚拟机将充当 UCP 控制器：

[PRE26]

1.  还要创建一个名为`node2`的虚拟机，稍后将其配置为 UCP 节点：

[PRE27]

1.  将`node1`配置为 UCP 控制器，负责提供 UCP 应用程序并运行管理 Docker 对象安装的过程。在此之前，设置环境以将`node1`配置为 UCP 控制器：

[PRE28]

1.  在将`node1`设置为 UCP 控制器时，它将要求输入 UCP 管理员帐户的密码，并且还将要求输入其他别名，可以使用 enter 命令添加或跳过：

[PRE29]

1.  UCP 控制台可以使用安装结束时提供的 URL 访问；使用`admin`作为用户名和之前安装时设置的密码登录。![Docker Universal Control Plane](img/image_05_010.jpg)

Docker UCP 许可证页面

1.  登录后，可以添加或跳过试用许可证。可以通过在 Docker 网站上的 UCP 仪表板上的链接下载试用许可证。UCP 控制台具有多个选项，如列出应用程序、容器和节点：![Docker Universal Control Plane](img/image_05_011.jpg)

Docker UCP 管理仪表板

1.  首先通过设置环境将 UCP 的`node2`加入控制器：

[PRE30]

1.  使用以下命令将节点添加到 UCP 控制器。将要求输入 UCP 控制器 URL、用户名和密码，如图所示：

[PRE31]

1.  UCP 的安装已完成；现在可以通过从 Docker Hub 拉取官方 DTR 镜像来在`node2`上安装 DTR。为了完成 DTR 的安装，还需要 UCP URL、用户名、密码和证书：

[PRE32]

1.  安装成功后，DTR 可以在 UCP UI 中列为一个应用程序：![Docker Universal Control Plane](img/image_05_012.jpg)

Docker UCP 列出所有应用程序

1.  DTR UI 可以使用`http://node2` URL 访问。单击**新存储库**按钮即可创建新存储库：![Docker Universal Control Plane](img/image_05_013.jpg)

在 DTR 中创建新的私有注册表

1.  可以从之前创建的安全 DTR 中推送和拉取镜像，并且还可以将存储库设置为私有，以便保护公司内部的容器。![Docker Universal Control Plane](img/image_05_014.jpg)

在 DTR 中创建新的私有注册表

1.  可以使用**设置**选项从菜单中配置 DTR，该选项允许设置 Docker 镜像的域名、TLS 证书和存储后端。![Docker Universal Control Plane](img/image_05_015.jpg)

DTR 中的设置选项

# 摘要

在本章中，我们深入探讨了 Docker 注册表。我们从使用 Docker Hub 的 Docker 公共存储库的基本概念开始，并讨论了与更大观众共享容器的用例。Docker 还提供了部署私有 Docker 注册表的选项，我们研究了这一点，并可以用于在组织内部推送、拉取和共享 Docker 容器。然后，我们研究了通过使用 Notary 服务器对 Docker 容器进行签名来标记和确保其完整性，该服务器可以与 Docker Engine 集成。DTR 提供了更强大的解决方案，它在本地以及虚拟私有云中提供企业级 Docker 镜像存储，以提供安全性并满足监管合规要求。它在 Docker UCP 之上运行，如前面详细的安装步骤所示。我希望本章能帮助您解决问题并了解 Docker 注册表的最新趋势。在下一章中，我们将探讨如何通过特权容器和资源共享使容器正常工作。

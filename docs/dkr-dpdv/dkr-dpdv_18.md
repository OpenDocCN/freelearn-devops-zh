# 第十七章：企业工具

在本章中，我们将看一些 Docker 公司提供的企业级工具。我们将看到如何安装它们，配置它们，备份它们和恢复它们。

这将是一个相当长的章节，包含大量逐步的技术细节。我会尽量保持有趣，但这可能会很难:-D

其他工具也存在，但我们将集中在 Docker 公司的工具上。

让我们直接开始。

### 企业工具-简而言之

Docker 和容器已经席卷了应用程序开发世界-构建，发布和运行应用程序从未如此简单。因此，企业想要参与其中并不奇怪。但企业有比典型的前沿开发者更严格的要求。

企业需要以他们可以使用的方式打包 Docker。这通常意味着他们拥有和管理自己的本地解决方案。这还意味着角色和安全功能，使其适应其内部结构，并得到安全部门的认可。这也意味着一切都有一个有意义的支持合同支持。

这就是 Docker 企业版（EE）发挥作用的地方！

Docker EE 是*企业版的 Docker*。它是一套产品，包括一个经过加固的引擎，一个操作界面和一个安全的私有注册表。您可以在本地部署它，并且它包含在支持合同中。

高级堆栈如图 16.1 所示。

![图 16.1 Docker EE](img/figure16-1.png)

图 16.1 Docker EE

### 企业工具-深入挖掘

我们将把本章的其余部分分为以下几个部分：

+   Docker EE 引擎

+   Docker Universal Control Plane (UCP)

+   Docker Trusted Registry (DTR)

我们将看到如何安装每个工具，并在适用的情况下配置 HA，并执行备份和恢复作业。

#### Docker EE 引擎

*Docker 引擎*提供了所有核心的 Docker 功能。像镜像和容器管理，网络，卷，集群，秘密等等。在撰写本文时，有两个版本：

+   社区版（CE）

+   企业版（EE）

就我们而言，最大的两个区别是发布周期和支持。

Docker EE 按季度发布，并使用基于时间的版本方案。例如，2018 年 6 月发布的 Docker EE 将是`18.06.x-ee`。Docker 公司保证每个版本提供 1 年的支持和补丁。

##### 安装 Docker EE

安装 Docker EE 很简单。但是，不同平台之间存在细微差异。我们将向您展示如何在 Ubuntu 16.04 上进行操作，但在其他平台上进行操作同样简单。

Docker EE 是基于订阅的服务，因此您需要一个 Docker ID 和一个活跃的订阅。这将为您提供访问一个独特个性化的 Docker EE 仓库，我们将在接下来的步骤中配置和使用。[试用许可证](https://store.docker.com/editions/enterprise/docker-ee-trial)通常是可用的。

> **注意：** Windows Server 上的 Docker 始终安装 Docker EE。有关如何在 Windows Server 2016 上安装 Docker EE 的信息，请参阅第三章。

您可能需要在以下命令前加上`sudo`。

1.  确保您拥有最新的软件包列表。

```
 $ apt-get update 
```

* 安装访问 Docker EE 仓库所需的软件包。

```
 $ apt-get install -y \
 	  apt-transport-https \
 	  curl \
 	  software-properties-common 
```

* 登录[Docker Store](https://store.docker.com/)并复制您独特的 Docker EE 仓库 URL。

将您的网络浏览器指向 store.docker.com。点击右上角的用户名，然后选择`My Content`。在您的活动 Docker EE 订阅下选择`Setup`。

![](img/figure16-2.png)

从`Resources`窗格下复制您的仓库 URL。

同时下载您的许可证。

![](img/figure16-3.png)

我们正在演示如何为 Ubuntu 设置仓库。但是，这个 Docker Store 页面包含了如何为其他 Linux 版本进行设置的说明。

+   将您独特的 Docker EE 仓库 URL 添加到环境变量中。

```
 $ DOCKER_EE_REPO=<paste-in-your-unique-ee-url> 
```

+   将官方 Docker GPG 密钥添加到所有密钥环中。

```
 $ curl -fsSL "`${``DOCKER_EE_REPO``}`/ubuntu/gpg" | sudo apt-key add - 
```

+   设置最新的稳定仓库。您可能需要用最新的稳定版本替换最后一行的值。

```
 $ add-apt-repository \
    "deb [arch=amd64] $DOCKER_EE_REPO/ubuntu \
    $(lsb_release -cs) \
    stable-17.06" 
```

+   运行另一个`apt-get update`，以从您新添加的 Docker EE 仓库中获取最新的软件包列表。

```
 $ apt-get update 
```

+   卸载先前版本的 Docker。

```
 $ apt-get remove docker docker-engine docker-ce docker.io 
```

+   安装 Docker EE

```
 $ apt-get install docker-ee -y 
```

+   检查安装是否成功。

```
$ docker --version
Docker version `17`.06.2-ee-6, build e75fdb8 
```

就这样，您已经安装了 Docker EE 引擎。

现在，您可以安装 Universal Control Plane 了。

#### Docker 通用控制平面（UCP）

在本章的其余部分，我们将把 Docker 通用控制平面简称为**UCP**。

UCP 是一种企业级的容器即服务平台，具有操作 UI。它基于 Docker 引擎，并添加了企业喜爱和需求的所有功能。例如*RBAC、策略、信任、高可用控制平面*和*简单的 UI*。在幕后，它是一个下载和作为一组容器运行的容器化微服务应用程序。

从架构上看，UCP 基于*Swarm mode*的 Docker EE 构建。如图 16.4 所示，UCP 控制平面运行在 Swarm 管理者上，应用程序部署在 Swarm 工作节点上。

![图 16.4 高级 UCP 架构](img/figure16-4.png)

图 16.4 高级 UCP 架构

在撰写本文时，UCP 管理者必须是 Linux。工作节点可以是 Windows 和 Linux 的混合。

##### 计划 UCP 安装

在计划 UCP 安装时，适当地确定集群的规模和规模是很重要的。我们将研究一些您应该考虑的事项。

集群中的所有节点应该同步其时钟（例如 NTP）。如果没有，可能会发生一些令人头痛的问题需要进行故障排除。

所有节点应具有一个静态 IP 地址和一个稳定的 DNS 名称。

默认情况下，UCP 管理者不运行用户工作负载。这是一种推荐的最佳实践，您应该在生产环境中执行它-它允许管理者专注于控制平面任务。这也使故障排除变得更容易。

您应该始终拥有奇数个管理者。这有助于避免管理者发生故障或与集群的其余部分分区的分裂脑条件。理想的数量是 3、5 或 7，其中 3 或 5 通常是最好的选择。拥有超过 7 个管理者可能会引发后端 Raft 和集群协调方面的问题。如果没有足够的节点来运行 3 个管理者，比起 2 个来说，1 个管理者要好！

如果您正在实施备份计划（您应该这样做），并进行定期备份，您可能希望部署 5 个管理者。这是因为 Swarm 和 UCP 备份操作需要停止 Docker 和 UCP 服务。在此类操作期间拥有 5 个管理者可以帮助维护集群的弹性。

管理节点应该分布在数据中心可用区。最不希望看到的情况是一个可用区失败并将所有的 UCP 管理者都带走。然而，通过高速可靠网络连接您的管理者非常重要。因此，如果您的数据中心可用区之间没有通过良好网络连接，最好将所有管理者保留在一个单一的可用区。作为一条一般规则，如果您在公共云基础架构上部署，应该将您的管理者部署在一个单一的*区域*中的可用区内。跨越*区域*通常涉及到不那么可靠的高延迟网络。

你可以拥有任意多的*工作节点* ——它们不参与集群 Raft 操作，因此不会影响控制平面操作。

规划工作节点的数量和大小需要了解你计划在集群上运行的应用程序。例如，了解这一点将帮助你确定你需要多少个 Windows 节点和 Linux 节点。你还需要知道你的应用程序是否有特殊要求，并且需要专门的工作节点 —— 可能是 PCI 工作负载。

此外，尽管 Docker 引擎轻量且小巧，但你在节点上运行的容器化应用可能并非如此。考虑到这一点，根据应用程序的 CPU、RAM、网络和磁盘 I/O 要求来调整节点的大小非常重要。

制定服务器大小要求并不是我喜欢做的事情，因为这完全取决于*你的*工作负载。然而，Docker 网站目前建议 Linux 上的 Docker UCP 2.2.4 的**最低**要求如下：

+   运行 DTR 的 UCP 管理节点：8GB RAM，3GB 磁盘空间

+   UCP 工作节点：4GB RAM，3GB 空闲磁盘空间

推荐的**要求是：

+   运行 DTR 的 UCP 管理节点：8GB RAM，4 个 vCPU 和 100GB 磁盘空间

+   UCP 工作节点：4GB RAM，25-100GB 空闲磁盘空间

对此要持谨慎态度，并确保进行自己的大小调整练习。

有一点是肯定的 —— Windows 镜像比 Linux 镜像**大得多**。所以请务必考虑这一点在你的大小调整中。

关于大小调整要有最后一句话。Docker Swarm 和 Docker UCP 使添加和删除管理节点和工作节点变得非常容易。新的管理节点会自动添加到 HA 控制平面，新的工作节点会立即可用于工作负载调度。类似地，删除管理节点和工作节点也很简单。只要你有多个管理节点，你可以删除一个管理节点而不影响集群操作。对于工作节点，你可以将它们排空并将其从运行中的集群中删除。这一切使得 UCP 在更改管理节点和工作节点时非常容易。

在考虑这些因素之后，我们准备安装 UCP 了。

##### 安装 Docker UCP

在这一部分，我们将会逐步介绍在新集群的第一个管理节点上安装 Docker UCP 的过程。

1.  从你想要成为 UCP 集群中第一个管理节点的基于 Linux 的 Docker EE 节点上运行以下命令。

    关于该命令有几点需要注意。示例中使用`docker/ucp:2.2.5`镜像安装 UCP，你需要替换为你想要的版本。`--host-address`是你将用于访问 Web UI 的地址。例如，如果你在 AWS 上安装，并计划通过互联网从你的公司网络访问，你需要输入 AWS 的公共 IP。

    安装是交互式的，所以你将会被提示输入进一步的信息以完成安装。

    ```
     $ docker container run --rm -it --name ucp \
       -v /var/run/docker.sock:/var/run/docker.sock \
       docker/ucp:2.2.5 install \
       --host-address <node-ip-address> \
       --interactive 
    ```

+   配置凭据。

    您将被提示为 UCP 管理员帐户创建用户名和密码。这是一个本地帐户，您应遵循企业指南选择用户名和密码。确保您不要忘记它 :-D

+   主题替代名称（SAN）。

    安装程序会给您提供输入替代 IP 地址和名称列表的选项，这些 IP 地址和名称可能用于访问 UCP。这些可以是公共和私有 IP 地址以及 DNS 名称，并将添加到证书中。

关于安装的一些注意事项。

UCP 利用 Docker Swarm。这意味着 UCP 管理器必须在 Swarm 管理器上运行。如果您在*单引擎模式*下安装 UCP，则会自动切换到*Swarm 模式*。

安装程序拉取各种 UCP 服务的所有图像，并从中启动容器。以下列表显示了安装程序正在拉取的一些图像。

```
INFO[0008] Pulling required images... (this may take a while)
INFO[0008] Pulling docker/ucp-auth-store:2.2.5
INFO[0013] Pulling docker/ucp-hrm:2.2.5
INFO[0015] Pulling docker/ucp-metrics:2.2.5
INFO[0020] Pulling docker/ucp-swarm:2.2.5
INFO[0023] Pulling docker/ucp-auth:2.2.5
INFO[0026] Pulling docker/ucp-etcd:2.2.5
INFO[0028] Pulling docker/ucp-agent:2.2.5
INFO[0030] Pulling docker/ucp-cfssl:2.2.5
INFO[0032] Pulling docker/ucp-dsinfo:2.2.5
INFO[0080] Pulling docker/ucp-controller:2.2.5
INFO[0084] Pulling docker/ucp-proxy:2.2.5 
```

其中一些有趣的包括：

+   `ucp-agent` 这是主要的 UCP 代理。它部署到集群中的所有节点，并负责确保所需的 UCP 容器正在运行。

+   `ucp-etcd` 集群的持久键值存储。

+   `ucp-auth` 共享认证服务（也用于 DTR 的单点登录）。

+   `ucp-proxy` 控制对本地 Docker 套接字的访问，以防未经身份验证的客户端对集群进行更改。

+   `ucp-swarm` 与底层 Swarm 的兼容性。

最后，安装创建了几个根 CA：一个用于内部集群通信，一个用于外部访问。它们发出自签名证书，这在实验室和测试中是可以的，但在生产中不可以。

要使用来自受信任 CA 的证书安装 UCP，您将需要一个包含以下三个文件的证书包：

+   `ca.pem` 受信任 CA 的证书（通常是您内部公司的一个）。

+   `cert.pem` UCP 的公共证书。这需要包含集群将被访问的所有 IP 地址和 DNS 名称，包括任何负载均衡器。

+   `key.pem` UCP 的私钥。

如果您有这些文件，您需要将它们挂载到名为 `ucp-controller-server-certs` 的 Docker 卷中，并使用 `--external-ca` 标志指定该卷。您还可以在安装后的 Web UI 的 `Admin Settings` 页面上更改证书。

UCP 安装程序输出的最后一件事是您可以从中访问它的 URL。

```
<Snip>
INFO[0049] Login to UCP at https://<IP or DNS>:443 
```

将 Web 浏览器指向该地址并登录。如果您使用自签名证书，则需要接受浏览器警告。您还需要指定您的许可证文件，该文件可以从 Docker Store 的 `My Content` 部分下载。

登录后，您将登陆到 UCP 仪表板。

![](img/figure16-5.png)

此时，您拥有一个单节点的 UCP 集群。

您可以通过仪表板底部的 `Add Nodes` 链接添加更多的管理器和工作节点。

图 16.6 显示了添加节点的屏幕。您可以选择添加`管理者`或`工作节点`，并为要添加的节点提供相应的命令。示例显示了添加 Linux 工作节点的命令。注意，命令是一个`docker swarm`命令。

![](img/figure16-6.png)

添加节点将其加入到 Swarm 并在其上配置所需的 UCP 服务。如果要添加管理者，建议在每次新添加之间等待。这使得 Docker 有机会下载并运行所需的 UCP 容器，以及允许集群注册新管理者并实现法定人数。

新添加的管理者将自动配置为高可用 (HA) Raft 共识组的一部分，并被授予对集群存储的访问权限。此外，虽然外部负载均衡器通常不被视为 UCP HA 的核心部分，但它们提供了一个稳定的 DNS 主机名，掩盖了背后的情况 —— 例如节点故障。

你应该配置外部负载均衡器以在端口 443 上进行*TCP 透传*，并为每个 UCP 管理节点设置自定义的 HTTPS 健康检查，地址为`https://<ucp_manager>/_ping`。

现在您有一个可用的 UCP，您应该查看从`Admin Settings`页面可配置的选项。

![图 16.7 UCP 管理设置](img/figure16-7.png)

图 16.7 UCP 管理设置

此页面上的设置组成了作为 UCP 备份操作的一部分备份的大部分配置数据。

###### 控制对 UCP 的访问

所有对 UCP 的访问都通过身份管理子系统进行控制。这意味着在执行集群上的任何操作之前，您需要使用有效的 UCP 用户名和密码进行身份验证。这包括集群管理以及部署和管理服务。

我们已经在 UI 中见过这个 —— 我们必须使用用户名和密码登录。但同样适用于 Docker CLI —— 你不能从命令行运行未经身份验证的命令来操作 UCP！这是因为 UCP 集群节点上的本地 Docker 套接字受`ucp-proxy`服务保护，它不会接受未经授权的命令。

让我们看一下。

###### 客户端捆绑包

任何运行 Docker CLI 的节点都可以部署和管理 UCP 集群上的工作负载，**只要它提供了有效的 UCP 用户证书！**

在本节中，我们将创建一个新的 UCP 用户，为该用户创建并下载证书包，并配置 Docker 客户端以使用这些证书。完成后，我们将解释其工作原理。

1.  如果你还没有，请以`admin`身份登录 UCP。

1.  点击`User Management` > `Users`，然后创建一个新用户。

    由于我们尚未讨论角色和授权，将用户设置为 Docker EE 管理员。

1.  选择新用户后，点击`Configure`下拉框，选择`Client Bundle`。![](img/figure16-8.png)

1.  点击`New Client Bundle +`链接生成并下载用户的客户端捆绑包。

    此时，重要的是要注意客户端包是特定于用户的。下载的证书将使任何正确配置的 Docker 客户端以属于该包所属用户的身份在 UCP 集群上执行命令。

1.  将包复制到您要配置为管理 UCP 的 Docker 客户端。

1.  登录到客户端节点，并从该节点执行所有以下命令。

1.  解压包的内容。

    此示例使用 Linux `unzip`软件包将包的内容解压缩到当前目录。将包的名称替换为与您的环境中的包匹配的名称。

    ```
     $ unzip ucp-bundle-nigelpoulton.zip
     Archive:  ucp-bundle-nigelpoulton.zip
     extracting: ca.pem
     extracting: cert.pem
     extracting: key.pem
     extracting: cert.pub
     extracting: env.sh
     extracting: env.ps1
     extracting: env.cmd 
    ```

    正如输出所示，包含所需的`ca.pem`、`cert.pem`和`key.pem`文件。它还包括配置 Docker 客户端使用证书的脚本。

1.  使用适当的脚本配置 Docker 客户端。`env.sh`适用于 Linux 和 Mac，`env.ps1`和`env.cmd`适用于 Windows。

    您可能需要管理员/根权限来运行脚本。

    此示例适用于 Linux 和 Mac。

    ```
     $ eval "$(<env.sh)" 
    ```

    此时，客户端节点已完全配置。` `*   测试访问。

    ```
     $ docker version

      <Snip>

     Server:
      Version:      ucp/2.2.5
      API version:  1.30 (minimum version 1.20)
      Go version:   go1.8.3
      Git commit:   42d28d140
      Built:        Wed Jan 17 04:44:14 UTC 2018
      OS/Arch:      linux/amd64
      Experimental: false 
    ```

    请注意，输出的服务器部分显示版本为`ucp/2.2.5`。这证明 Docker 客户端成功与 UCP 节点上的守护程序通信！

在幕后，该脚本配置了三个环境变量：

+   `DOCKER_HOST`

+   `DOCKER_TLS_VERIFY`

+   `DOCKER_CERT_PATH`

DOCKER_HOST 将客户端指向 UCP 控制器上的远程 Docker 守护程序。示例可能如下所示`DOCKER_HOST=tcp://34.242.196.63:443`。正如我们所见，通过端口 443 访问。

`DOCKER_TLS_VERIFY`被设置为 1，告诉客户端在*客户端模式*下使用 TLS 验证。

`DOCKER_CERT_PATH`指示 Docker 客户端在哪里找到证书包。

结果是来自客户端的所有 docker 命令将由用户的证书签名并通过网络发送到远程 UCP 管理器。如图 16.9 所示。

![图 16.9](img/figure16-9.png)

图 16.9

让我们改变策略，看看如何备份和恢复 UCP。

###### 备份 UCP

首先和最重要的是，高可用性（HA）与备份不同！

考虑以下示例。您有一个具有 5 个管理节点的高可用 UCP 集群。所有管理节点都处于健康状态，控制平面正在复制。一个不满意的员工损坏了集群（或删除了所有用户帐户）。这种*损坏*会自动复制到所有 5 个管理节点，使集群破裂。在这种情况下，HA 无法帮助您。您需要的是备份！

UCP 集群由三个需要分别备份的主要组件组成：

+   Swarm

+   UCP

+   Docker Trusted Registry（DTR）

我们将指导您完成备份 Swarm 和 UCP 的过程，并在本章后面向您展示如何备份 DTR。

尽管 UCP 位于 Swarm 之上，但它们是独立的组件。Swarm 拥有所有节点成员、网络、卷和服务定义。UCP 位于其上并维护着自己的数据库和卷，其中包含用户、组、授权、捆绑包、许可证文件、证书等内容。

让我们看看如何**备份 Swarm**。

Swarm 的配置和状态存储在 `/var/lib/docker/swarm` 中。这包括 Raft 日志键，并且它被复制到每个管理节点。Swarm 备份是该目录中所有文件的副本。

因为它被复制到每个管理节点，你可以从任何一个管理节点执行备份。

您需要停止要在其上执行备份的节点上的 Docker。这意味着在领导管理者上执行备份可能不是一个好主意，因为会启动领导者选举。您还应该在业务的安静时段执行备份 — 即使在多管理节点 Swarm 中停止单个管理节点上的 Docker 不是问题，但如果在备份期间另一个管理节点失败，可能会增加群集失去法定人数的风险。

在继续之前，您可能想创建一些 Swarm 对象，以便您可以验证备份和恢复操作是否有效。我们将在这些示例中备份的示例 Swarm 具有名为 `vantage-net` 的覆盖网络和名为 `vantage-svc` 的 Swarm 服务。

1.  停止正在执行备份的 Swarm 管理节点上的 Docker。

    这将停止节点上的所有 UCP 容器。如果 UCP 配置为 HA，则其他管理节点将确保控制平面保持可用。

    ```
     $ service docker stop 
    ```

+   备份 Swarm 配置。

    该示例使用 Linux 的 `tar` 实用工具来执行文件复制。请随意使用其他工具。

    ```
     $ tar -czvf swarm.bkp /var/lib/docker/swarm/
     tar: Removing leading `/' from member names
     /var/lib/docker/swarm/
     /var/lib/docker/swarm/docker-state.json
     /var/lib/docker/swarm/state.json
     <Snip> 
    ```

+   验证备份文件是否存在。

    ```
     $ ls -l
     -rw-r--r-- 1 root   root   450727 Jan 29 14:06 swarm.bkp 
    ```

    按照公司备份政策，您应该轮换并将备份文件存储在外部。

+   重启 Docker。

    ```
     $ service docker restart 
    ```

现在 Swarm 已经备份好了，是时候**备份 UCP**了。

在我们开始之前，关于备份 UCP 的一些注意事项。

UCP 备份作业作为容器运行，因此备份需要 Docker 正在运行。

您可以在集群中的任何 UCP 管理节点上运行备份，并且只需在一个节点上运行操作（UCP 将其配置复制到所有管理节点，因此不需要从多个节点备份）。

备份 UCP 将停止执行操作的管理节点上的所有 UCP 容器。考虑到这一点，您应该运行一个高可用的 UCP 集群，并且应该在业务的安静时段运行操作。

最后，运行在管理节点上的用户工作负载将不会被停止。但是，不建议在 UCP 管理节点上运行用户工作负载。

让我们备份 UCP。

在 UCP 管理节点上执行以下命令。节点上需要运行 Docker。

```
$ docker container run --log-driver none --rm -i --name ucp `\`
  -v /var/run/docker.sock:/var/run/docker.sock `\`
  docker/ucp:2.2.5 backup --interactive `\`
  --passphrase `"Password123"` > ucp.bkp 
```

这是一个很长的命令，所以让我们逐步进行。

第一行是一个标准的`docker container run`命令，告诉 Docker 在操作完成后运行一个不带日志驱动程序的容器，并将其命名为`ucp`。第二行将*Docker 套接字*挂载到容器中，以便容器可以访问 Docker API 以停止容器等。第三行告诉 Docker 在基于`docker/ucp:2.2.5`镜像的容器内运行`backup --interactive`命令。最后一行创建了一个名为`ucp.bkp`的加密文件，并使用密码保护它。

一些值得注意的要点。

具体指定要使用的 UCP 图像版本（标签）是个好主意。这个示例指定了`docker/ucp:2.2.5`。具体指定的原因之一是，建议使用相同版本的镜像运行备份和恢复操作。如果您不明确指定要使用哪个镜像，Docker 将使用标记为`latest`的镜像，这可能会在运行备份命令和运行恢复命令的时间之间不同。

您应该始终使用`--passphrase`标志来保护备份内容，并且绝对应该使用比示例中更好的密码 :-D

根据您公司的备份政策对备份文件进行分类并制作离线副本。您还应该配置备份计划和作业验证。

现在 Swarm 和 UCP 已经备份，您可以在灾难发生时安全地恢复它们。说到这个……

###### 恢复 UCP

在我们深入讨论恢复 UCP 的细节之前，我们需要澄清一件事：从备份中恢复是最后的手段，只有在集群已损坏或所有管理器节点都丢失时才应使用！

如果在高可用（HA）集群中丢失了单个管理器，则**无需从备份中恢复**。在这种情况下，您可以轻松地添加一个新的管理器并加入集群。

我们将展示如何从备份中恢复 Swarm，然后是 UCP。

从您希望恢复的 Swarm/UCP 管理器节点执行以下任务。

1.  停止 Docker。

    ```
     $ service docker stop 
    ```

+   删除任何现有的 Swarm 配置。

    ```
     $ rm -r /var/lib/docker/swarm 
    ```

+   从备份中恢复 Swarm 配置。

    在这个示例中，我们将从名为`swarm.bkp`的压缩`tar`文件中恢复。执行此命令需要恢复到根目录，因为它将原始文件的完整路径作为提取操作的一部分。在您的环境中可能会有所不同。

    ```
     $ tar -zxvf swarm.bkp -C / 
    ```

+   初始化新的 Swarm 集群。

    请记住，您不是在恢复一个管理器并将其添加回一个正常运行的集群。此操作是为了恢复一个没有幸存的管理器的失败的 Swarm。`--force-new-cluster`标志告诉 Docker 使用当前节点上存储在`/var/lib/docker/swarm`中的配置创建一个新集群。

    ```
     $ docker swarm init --force-new-cluster
     Swarm initialized: current node (jhsg...3l9h) is now a manager. 
    ```

+   检查网络和服务是否作为操作的一部分恢复。

    ```
     $ docker network ls
     NETWORK ID        NAME            DRIVER       SCOPE
     snkqjy0chtd5      vantage-net     overlay      swarm

     $ docker service ls
     ID              NAME          MODE         REPLICAS    IMAGE
     w9dimu8jfrze    vantage-svc   replicated   5/5         alpine:latest 
    ```

    恭喜。Swarm 已恢复。

+   向 Swarm 添加新的管理器和工作节点，并进行新的备份。

Swarm 恢复后，现在可以**恢复 UCP**。

在这个示例中，UCP 被备份到当前目录下名为 `ucp.bkp` 的文件中。尽管备份文件的名字叫做备份文件，但它实际上是一个 Linux tarball。

从你想要恢复 UCP 的节点上运行以下命令。这可以是你刚刚恢复 Swarm 的节点。

1.  删除任何现有的、可能损坏的 UCP 安装。

    ```
     $ docker container run --rm -it --name ucp \
       -v /var/run/docker.sock:/var/run/docker.sock \
       docker/ucp:2.2.5 uninstall-ucp --interactive

     INFO[0000] Your engine version 17.06.2-ee-6, build e75fdb8 is compatible
     INFO[0000] We're about to uninstall from this swarm cluster.
     Do you want to proceed with the uninstall? (y/n): y
     INFO[0000] Uninstalling UCP on each node...
     INFO[0009] UCP has been removed from this cluster successfully.
     INFO[0011] Removing UCP Services 
    ```

+   从备份中恢复 UCP。

    ```
     $ docker container run --rm -i --name ucp \
       -v /var/run/docker.sock:/var/run/docker.sock  \
       docker/ucp:2.2.5 restore --passphrase "Password123" < ucp.bkp

     INFO[0000] Your engine version 17.06.2-ee-6, build e75fdb8 is compatible
     <Snip>
     time="2018-01-30T10:16:29Z" level=info msg="Parsing backup file"
     time="2018-01-30T10:16:38Z" level=info msg="Deploying UCP Agent Service"
     time="2018-01-30T10:17:18Z" level=info msg="Cluster successfully restored. 
    ```

+   登录 UCP Web UI，并确保之前创建的用户仍然存在（或之前在你的环境中存在的任何其他 UCP 对象）。

恭喜。你现在知道如何备份和恢复 Docker Swarm 和 Docker UCP 了。

让我们把注意力转向 Docker Trusted Registry。

#### Docker Trusted Registry（DTR）

Docker Trusted Registry，我们将简称为 DTR，是一个安全的、高可用的本地 Docker 注册表。如果你了解 Docker Hub，把 DTR 想象成一个你可以在本地安装并自行管理的私有 Docker Hub。

在本节中，我们将展示如何在高可用配置中安装它，以及如何备份和执行恢复操作。我们将在下一章节展示 DTR 如何实现高级功能。

在开始安装之前，让我们先提到一些重要的事项。

如果可能的话，你应该在专用节点上运行你的 DTR 实例。你绝对不应该在你的生产 DTR 节点上运行用户工作负载。

与 UCP 一样，你应该运行奇数个 DTR 实例。3 或 5 对于容错性最好。对于生产环境的推荐配置可能是：

+   3 个专用的 UCP 管理节点

+   3 个专用的 DTR 实例

+   无论你的应用需求需要多少工作节点

让我们安装并配置一个单独的 DTR 实例。

##### 安装 DTR

接下来的几个步骤将介绍如何在 UCP 集群中配置第一个 DTR 实例的过程。

要跟着进行操作，你需要一个 UCP 节点，在该节点上你将安装 DTR，以及一个配置为在 TCP 透传模式下监听端口 443，并对端口 443 上的 `/health` 配置了健康检查的负载均衡器。图 16.10 显示了我们将要构建的高层次图表。

配置负载均衡器超出了本书的范围，但图表显示了与 DTR 相关的重要配置要求。

![图 16.10 单一实例 DTR 高级配置。](img/figure16-10.png)

图 16.10 单一实例 DTR 高级配置。

1.  登录 UCP Web UI，点击 `Admin` > `Admin Settings` > `Docker Trusted Registry`。

1.  填写 DTR 配置表单。

    +   `DTR EXTERNAL URL:` 将其设置为外部负载均衡器的 URL。

    +   `UCP NODE:` 选择你要在其上安装 DTR 的节点的名称。

    +   `Disable TLS Verification For UCP:` 如果你使用自签名证书，请勾选此框。

1.  复制表单底部的长命令。

1.  将命令粘贴到任何 UCP 管理节点。

    命令包含 `--ucp-node` 标志，告诉 UCP 在哪个节点上执行安装。

    以下是与图 16.10 配置匹配的示例 DTR 安装命令。假设您已经在`dtr.mydns.com`配置了负载均衡器。

    ```
     $ docker run -it --rm docker/dtr install \
       --dtr-external-url dtr.mydns.com \
       --ucp-node dtr1  \
       --ucp-url https://34.252.195.122 \
       --ucp-username admin --ucp-insecure-tls 
    ```

    您需要提供 UCP 管理员密码以完成安装。

+   安装完成后，将您的 Web 浏览器指向负载均衡器。您将自动登录到 DTR。![图 16.11 DTR 主页](img/figure16-11.png)

    图 16.11 DTR 主页`

`DTR 已准备就绪。但尚未配置为 HA。

##### 为高可用性配置 DTR

使用多个副本为 HA 配置 DTR 需要共享存储后端。这可以是 NFS 或对象存储，并且可以在本地或公共云中。我们将逐步介绍使用 Amazon S3 桶作为共享后端配置 DTR 以实现 HA 的过程。

1.  登录到 DTR 控制台并导航到`Settings`。

1.  选择`Storage`选项卡并配置共享存储后端。

    图 16.12 显示了配置为使用名为`deep-dive-dtr`的 AWS S3 存储桶和`eu-west-1` AWS 可用区的 DTR。您将无法使用此示例。

    ![图 16.12 AWS 的 DTR 共享存储配置](img/figure16-12.png)

    图 16.12 AWS 的 DTR 共享存储配置

DTR 现在配置了共享存储后端，并准备好添加额外的副本。

1.  从 UCP 集群中的管理节点运行以下命令。

    ```
     $ docker run -it --rm \
       docker/dtr:2.4.1 join \
       --ucp-node dtr2 \
       --existing-replica-id 47f20fb864cf \
       --ucp-insecure-tls 
    ```

    `--ucp-node`标志告诉命令在哪个节点上添加新的 DTR 副本。如果使用自签名证书，则需要`--insecure-tls`标志。

    您需要替换图像版本和副本 ID。安装初始副本时，副本 ID 将作为输出的一部分显示。

+   当提示时，输入 UCP URL 和端口，以及管理员凭据。

当连接完成时，您将看到类似以下的消息。

```
INFO[0166] Join is complete
INFO[0166] Replica ID is set to: a6a628053157
INFO[0166] There are currently 2 replicas in your DTR cluster
INFO[0166] You have an even number of replicas which can impact availability
INFO[0166] It is recommended that you have 3, 5 or 7 replicas in your cluster 
```

请务必遵循建议并安装额外的副本，以便您运行奇数个。

您可能需要更新负载均衡器配置，以便在新副本之间平衡流量。

DTR 现已配置为 HA。这意味着您可以丢失一个副本而不影响服务的可用性。图 16.13 显示了 HA DTR 配置。

![图 16.13 DTR HA](img/figure16-13.png)

图 16.13 DTR HA

注意，外部负载均衡器正在将流量发送到所有三个 DTR 副本，并在所有三个副本上执行健康检查。所有三个 DTR 副本还共享相同的外部共享存储后端。

在图中，负载均衡器和共享存储后端是第三方产品，并且被描述为单例（非 HA）。为了尽可能保持整个环境的高可用性，您应确保它们具有本地 HA，并且还备份它们的内容和配置（例如，确保负载均衡器和存储系统具有本地 HA，并备份它们）。

##### 备份 DTR

与 UCP 一样，DTR 具有原生的 `backup` 命令，该命令是安装 DTR 使用的 Docker 镜像的一部分。此原生备份命令将备份存储在一组命名卷中的 DTR 配置，并包括：

+   DTR 配置

+   存储库元数据

+   公证数据

+   证书

图像不作为原生 DTR 备份的一部分进行备份**。预期图像存储在具有自己独立备份计划的高可用性存储后端中，该计划使用非 Docker 工具。

从 UCP 管理节点运行以下命令执行 DTR 备份。

```
$ read -sp 'ucp password: ' UCP_PASSWORD; \
    docker run --log-driver none -i --rm \
    --env UCP_PASSWORD=$UCP_PASSWORD \
    docker/dtr:2.4.1 backup \
    --ucp-insecure-tls \
    --ucp-username admin \
    > ucp.bkp 
```

让我们解释一下这个命令在做什么。

`read` 命令会提示您输入 UCP 管理员帐户的密码，并将其存储在名为 `UCP_PASSWORD` 的变量中。第二行告诉 Docker 启动一个新的临时容器进行操作。第三行使 UCP 密码作为环境变量在容器内可用。第四行发出备份命令。第五行使其与自签名证书一起工作。第六行将 UCP 用户名设置为“admin”。最后一行将备份指向当前目录中的名为 `ucp.bkp` 的文件。

您将被提示输入 UCP URL 以及副本 ID。您可以将这些指定为备份命令的一部分，我只是不想解释一个长达 9 行的单个命令！

备份完成后，您的工作目录中将有一个名为 `ucp.bkp` 的文件。您的企业备份工具应该会捡起这个文件，并与您现有的企业备份策略一起管理。

##### 从备份中恢复 DTR

从备份中恢复 DTR 应该是最后的手段，只有在大多数副本都关闭且群集无法以其他方式恢复时才应尝试。如果丢失了单个副本而大多数副本仍然运行，您应该使用 `dtr join` 命令添加一个新副本。

如果您确定必须从备份中恢复，工作流程如下：

1.  停止并删除节点上的 DTR（可能已经停止）

1.  将图像恢复到共享存储后端（可能不需要）

1.  恢复 DTR

从要将 DTR 恢复到的节点运行以下命令。这个节点显然需要是与 DTR 所在的 UCP 集群的成员相同的。您还应该使用创建备份时使用的 `docker/dtr` 镜像的相同版本。

1.  停止并删除 DTR。

    ```
     $ docker run -it --rm \
       docker/dtr:2.4.1 destroy \
       --ucp-insecure-tls

     INFO[0000] Beginning Docker Trusted Registry replica destroy
     ucp-url (The UCP URL including domain and port): https://34.252.195.122:443
     ucp-username (The UCP administrator username): admin
     ucp-password:
     INFO[0020] Validating UCP cert
     INFO[0020] Connecting to UCP
     INFO[0021] Searching containers in UCP for DTR replicas
     INFO[0023] This cluster contains the replicas: 47f20fb864cf a6a628053157
     Choose a replica to destroy [47f20fb864cf]:
     INFO[0030] Force removing replica
     INFO[0030] Stopping containers
     INFO[0035] Removing containers
     INFO[0045] Removing volumes
     INFO[0047] Replica removed. 
    ```

    您将被提示输入要删除的 UCP URL、管理员凭据和副本 ID。

    如果您有多个副本，可以多次运行该命令以删除它们全部。

+   如果从共享后端丢失了图像，您需要恢复它们。由于此步骤可能特定于您的共享存储后端，因此超出了本书的范围。* 使用以下命令还原 DTR。

    您将需要使用您环境中的值替换第 5 和第 6 行上的值。不幸的是 `restore` 命令不能交互式运行，因此一旦 `restore` 开始，您将无法提示值。

    ```
     $ read -sp 'ucp password: ' UCP_PASSWORD; \
     docker run -i --rm \
     --env UCP_PASSWORD=$UCP_PASSWORD \
     docker/dtr:2.4.1 restore \
     --ucp-url <ENTER_YOUR_ucp-url> \
     --ucp-node <ENTER_DTR_NODE_hostname> \
     --ucp-insecure-tls \
     --ucp-username admin \
     < ucp.bkp 
    ```

DTR 现已恢复。

恭喜。您现在知道如何备份和恢复；Swarm、UCP 和 DTR。

在结束本章之前，还有最后一件事——网络端口！

UCP 管理器、工作节点和 DTR 节点需要能够在网络上进行通信。图 16.14 总结了端口要求。

![图 16.14 UCP 集群网络端口要求](img/figure16-14.png)

图 16.14 UCP 集群网络端口要求

### 章节总结

Docker 企业版（EE）是一套产品，形成了一个“*企业友好*”的容器即服务平台。它包括一个经过硬化的 Docker 引擎、一个运维 UI 和一个安全的注册表。所有这些都可以部署在本地并由客户管理。甚至还附带了支持合同。

Docker 通用控制平面（UCP）提供了一个简单易用的 Web UI，专注于传统企业运维团队。它支持本地高可用性（HA），并具有执行备份和恢复操作的工具。一旦启动，它提供了一整套我们将在下一章讨论的企业级功能。

Docker 受信任的注册表（DTR）位于 UCP 之上，提供了一个高可用的安全注册表。与 UCP 一样，这可以在公司*“防火墙”*的安全环境中本地部署，并提供用于备份和恢复的本地工具。

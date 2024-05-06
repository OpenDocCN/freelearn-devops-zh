# 第五章：Docker 的朋友

到目前为止，我们一直忙于学习有关 Docker 的一切。影响开源项目寿命的一个主要因素是其周围的社区。Docker 的创建者 Docker Inc.（**dotCloud**的分支）负责开发和维护 Docker 及其姊妹项目，如 libcontainer、libchan、swarm 等（完整列表可在[github.com/docker](http://github.com/docker)找到）。然而，像任何其他开源项目一样，开发是公开的（在 GitHub 上），他们接受拉取请求。

行业也接受了 Docker。像谷歌、亚马逊、微软、eBay 和 RedHat 这样的大公司积极使用和贡献 Docker。大多数流行的 IaaS 解决方案，如亚马逊网络服务、谷歌计算云等，都支持创建预加载和优化为 Docker 的镜像。许多初创公司也在 Docker 上押注他们的财富。CoreOS、Drone.io 和 Shippable 是一些初创公司，它们提供基于 Docker 的服务。因此，您可以放心，它不会很快消失。

在本章中，我们将讨论围绕 Docker 的一些项目以及如何使用它们。我们还将看看您可能已经熟悉的项目，这些项目可以促进您的 Docker 工作流程（并使您的生活变得更加轻松）。

首先，我们将讨论如何使用 Chef 和 Puppet 配方与 Docker。你们中的许多人可能已经在工作流程中使用这些工具。本节将帮助您将 Docker 与当前工作流程集成，并使您逐步进入 Docker 生态系统。

接下来，我们将尝试设置一个**apt-cacher**，这样我们的 Docker 构建就不会花费大量时间从 Canonical 服务器获取经常使用的软件包。这将大大减少使用 Dockerfile 构建镜像所需的时间。

在早期阶段，给 Docker 带来如此大的关注的一件事是，一些本来被认为很难的事情在 Docker 实现时似乎变得很容易。其中一个项目是**Dokku**，一个 100 行的 bash 脚本，可以设置一个类似于**mini**-**Heroku**的 PaaS。在本章中，我们将使用 Dokku 设置我们自己的 PaaS。本书中我们将讨论的最后一件事是使用 CoreOS 和 Fleet 部署高可用服务。

简而言之，在我们旅程的最后一段，我们将讨论以下主题：

+   使用 Docker 与 Chef 和 Puppet

+   设置一个 apt-cacher

+   设置您自己的 mini-Heroku

+   设置一个高可用的服务

# 使用 Docker 与 Chef 和 Puppet

当企业开始进入云时，扩展变得更加容易，因为可以从一台单机扩展到数百台而不费吹灰之力。但这也意味着需要配置和维护这些机器。配置管理工具，如 Chef 和 Puppet，是为了自动部署公共/私有云中的应用而产生的需求。如今，Chef 和 Puppet 每天都被全球各地的初创公司和企业用来管理他们的云环境。

## 使用 Docker 与 Chef

Chef 的网站上写着：

> *"Chef 将基础设施转化为代码。使用 Chef，您可以自动化构建、部署和管理基础设施。您的基础设施变得像应用代码一样可版本化、可测试和可重复。"*

现在，假设您已经设置好了 Chef，并熟悉了 Chef 的工作流程，让我们看看如何使用 chef-docker 食谱在 Chef 中使用 Docker。

您可以使用任何食谱依赖管理器安装此食谱。有关 Berkshelf、Librarian 和 Knife 的安装说明可在该食谱的 Chef 社区网站上找到（[`supermarket.getchef.com/cookbooks/docker`](https://supermarket.getchef.com/cookbooks/docker)）。

### 安装和配置 Docker

安装 Docker 很简单。只需将`recipe[docker]`命令添加到运行列表（配置设置列表）即可。举个例子，让我们看看如何编写一个 Chef 配方来在 Docker 上运行`code.it`文件（我们的示例项目）。

### 编写一个 Chef 配方，在 Docker 上运行 Code.it

以下 Chef 配方基于`code.it`启动一个容器：

[PRE0]

第一个非注释语句包括 Chef-Docker 配方。`docker_image 'shrikrishna/code.it'`语句相当于在控制台中运行`$ docker pull shrikrishna/code.it`命令。配方末尾的语句块相当于运行`$ docker run --d -p '8000:8000' -e 'NODE_PORT=8000' -v '/var/log/code.it:/var/log/code.it' shrikrishna/code.it`命令。

## 使用 Docker 与 Puppet

PuppetLabs 的网站上写着：

> “Puppet 是一个配置管理系统，允许您定义 IT 基础架构的状态，然后自动强制执行正确的状态。无论您是管理几台服务器还是成千上万台物理和虚拟机，Puppet 都会自动化系统管理员经常手动执行的任务，从而节省时间和精力，使系统管理员可以专注于提供更大商业价值的项目。”

Puppet 的等效于 Chef cookbooks 的模块。有一个为 Docker 提供支持的模块可用。通过运行以下命令来安装它：

[PRE1]

### 编写一个 Puppet 清单来在 Docker 上运行 Code.it

以下 Puppet 清单启动了一个`code.it`容器：

[PRE2]

第一个非注释语句包括`docker`模块。`docker::image {'shrikrishna/code.it':}`语句相当于在控制台中运行`$ docker pull shrikrishna/code.it`命令。在配方末尾的语句块相当于运行`$ docker run --d -p '8000:8000' -e 'NODE_PORT=8000' -v '/var/log/code.it:/var/log/code.it' shrikrishna/code.it node /srv/app.js`命令。

# 设置 apt-cacher

当您有多个 Docker 服务器，或者当您正在构建多个不相关的 Docker 镜像时，您可能会发现每次都必须下载软件包。这可以通过在服务器和客户端之间设置缓存代理来防止。它在您安装软件包时缓存软件包。如果您尝试安装已经缓存的软件包，它将从代理服务器本身提供，从而减少获取软件包的延迟，大大加快构建过程。

让我们编写一个 Dockerfile 来设置一个 apt 缓存服务器作为缓存代理服务器：

[PRE3]

这个 Dockerfile 在镜像中安装了`apt-cacher-ng`软件包，并暴露端口`3142`（供目标容器使用）。

使用此命令构建镜像：

[PRE4]

然后运行它，绑定暴露的端口：

[PRE5]

要查看日志，请运行以下命令：

[PRE6]

## 在构建 Dockerfiles 时使用 apt-cacher

所以我们已经设置了一个 apt-cacher。现在我们必须在我们的 Dockerfiles 中使用它：

[PRE7]

在第二条指令中，用您的 Docker 主机的 IP 地址（在`docker0`接口处）替换`<host's-docker0-ip-here>`命令。在构建这个 Dockerfile 时，如果遇到任何已经安装过的软件包的`apt-get install`安装命令（无论是为了这个镜像还是其他镜像），它将从本地代理服务器获取软件包，从而加快构建过程中的软件包安装速度。如果要安装的软件包不在缓存中，则从 Canonical 仓库获取并保存在缓存中。

### 提示

apt-cacher 只对使用 Apt 软件包管理工具的基于 Debian 的容器（如 Ubuntu）有效。

# 设置您自己的迷你 Heroku

现在让我们做一些酷炫的事情。对于初学者来说，Heroku 是一个云 PaaS，这意味着您在构建应用程序时只需要将其推送到 Heroku，它就会部署在[`www.herokuapp.com`](https://www.herokuapp.com)上。您不需要担心应用程序运行的方式或位置。只要 PaaS 支持您的技术栈，您就可以在本地开发并将应用程序推送到服务上，让其在公共互联网上实时运行。

除了 Heroku 之外，还有许多 PaaS 提供商。一些流行的提供商包括 Google App Engine、Red Hat Cloud 和 Cloud Foundry。Docker 是由一个这样的 PaaS 提供商 dotCloud 开发的。几乎每个 PaaS 都通过在预定义的沙盒环境中运行应用程序来工作，而这正是 Docker 擅长的。如今，Docker 已经使得设置 PaaS 变得更加容易，如果不是简单的话。证明这一点的项目是 Dokku。Dokku 与 Heroku 共享使用模式和术语（如`buildpacks`、`slug` `builder`脚本），这使得它更容易使用。在本节中，我们将使用 Dokku 设置一个迷你 PaaS，并推送我们的`code.it`应用程序。

### 注意

接下来的步骤应该在虚拟专用服务器（VPS）或虚拟机上完成。您正在使用的主机应该已经设置好了 git 和 SSH。

## 使用 bootstrapper 脚本安装 Dokku

有一个`bootstrapper`脚本可以设置 Dokku。在 VPS/虚拟机内运行此命令：

[PRE8]

### 注意

12.04 版本的用户需要在运行上述`bootstrapper`脚本之前运行`$ apt-get install -y python-software-properties`命令。

`bootstrapper`脚本将下载所有依赖项并设置 Dokku。

## 使用 Vagrant 安装 Dokku

步骤 1：克隆 Dokku：

[PRE9]

步骤 2：在您的`/etc/hosts`文件中设置 SSH 主机：

[PRE10]

步骤 3：在`~/.ssh/config`中设置 SSH 配置

[PRE11]

步骤 4：创建虚拟机

以下是一些可选的 ENV 参数设置：

[PRE12]

步骤 5：使用此命令复制您的 SSH 密钥：

[PRE13]

在`http://dokku.app`的 dokku-installer 中粘贴您的 SSH 密钥（指向`/etc/hosts`文件中分配的`10.0.0.2`）。在**Dokku 设置**屏幕上更改**主机名**字段为您的域名，然后选中**使用虚拟主机命名**的复选框。然后，单击**完成设置**以安装您的密钥。您将从这里被引导到应用程序部署说明。

您现在已经准备好部署应用程序或安装插件。

## 配置主机名并添加公钥

我们的 PaaS 将子域路由到使用相同名称部署的应用程序。这意味着设置了 Dokku 的机器必须对您的本地设置以及运行 Dokku 的机器可见。

设置一个通配符域，指向 Dokku 主机。运行`bootstrapper`脚本后，检查 Dokku 主机中的`/home/dokku/VHOST`文件是否设置为此域。只有当 dig 工具可以解析主机名时，它才会被创建。

在此示例中，我已将我的 Dokku 主机名设置为`dokku.app`，方法是将以下配置添加到我的本地主机的`/etc/hosts`文件中：

[PRE14]

我还在本地主机的`~/.ssh/config`文件中设置了 SSH 端口转发规则：

[PRE15]

### 注意

根据维基百科，**域名信息检索器**（**dig**）是一个用于查询 DNS 名称服务器的网络管理命令行工具。这意味着给定一个 URL，dig 将返回 URL 指向的服务器的 IP 地址。

如果`/home/dokku/VHOST`文件没有自动创建，您将需要手动创建它并将其设置为您喜欢的域名。如果在部署应用程序时缺少此文件，Dokku 将使用端口名称而不是子域名发布应用程序。

最后要做的事情是将您的公共`ssh`密钥上传到 Dokku 主机并将其与用户名关联起来。要这样做，请运行此命令：

[PRE16]

在上述命令中，将`dokku.app`名称替换为您的域名，将`shrikrishna`替换为您的名称。

太好了！现在我们已经准备好了，是时候部署我们的应用程序了。

## 部署应用程序

我们现在有了自己的 PaaS，可以在那里部署我们的应用程序。让我们在那里部署`code.it`文件。您也可以尝试在那里部署您自己的应用程序：

[PRE17]

就是这样！我们现在在我们的 PaaS 中有一个可用的应用程序。有关 Dokku 的更多详细信息，您可以查看其 GitHub 存储库页面[`github.com/progrium/dokku`](https://github.com/progrium/dokku)。

如果您想要一个生产就绪的 PaaS，您必须查找 Deis [`deis.io/`](http://deis.io/)，它提供多主机和多租户支持。

# 建立一个高可用的服务

虽然 Dokku 非常适合部署偶尔的副业，但对于较大的项目可能不太合适。大规模部署基本上具有以下要求：

+   **水平可扩展**：单个服务器实例只能做这么多。随着负载的增加，处于快速增长曲线上的组织将发现自己必须在一组服务器之间平衡负载。在早期，这意味着必须设计数据中心。今天，这意味着向云中添加更多实例。

+   **容错**：即使有广泛的交通规则来避免交通事故，事故也可能发生，但即使您采取了广泛的措施来防止事故，一个实例的崩溃也不应该导致服务停机。良好设计的架构将处理故障条件，并使另一个服务器可用以取代崩溃的服务器。

+   **模块化**：虽然这可能看起来不是这样，但模块化是大规模部署的一个定义特征。模块化架构使其灵活且具有未来可塑性（因为模块化架构将随着组织的范围和影响力的增长而容纳新的组件）。

这绝不是一个详尽的清单，但它标志着构建和部署高可用服务所需的努力。然而，正如我们到目前为止所看到的，Docker 仅用于单个主机，并且（直到现在）没有可用于管理运行 Docker 的一组实例的工具。

这就是 CoreOS 的用武之地。它是一个精简的操作系统，旨在成为 Docker 大规模部署服务的构建模块。它配备了一个高可用的键值配置存储，称为`etcd`，用于配置管理和服务发现（发现集群中其他组件的位置）。`etcd`服务在第四章中进行了探讨，*自动化和最佳实践*。它还配备了 fleet，这是一个利用`etcd`提供的一种在整个集群上执行操作的工具，而不是在单个实例上执行操作。

### 注意

您可以将 fleet 视为在集群级别而不是机器级别运行的`systemd`套件的扩展。`systemd`套件是单机初始化系统，而 fleet 是集群初始化系统。您可以在[`coreos.com/using-coreos/clustering/`](https://coreos.com/using-coreos/clustering/)了解更多关于 fleet 的信息。

在本节中，我们将尝试在本地主机上的三节点 CoreOS 集群上部署我们的标准示例`code.it`。这是一个代表性的示例，实际的多主机部署将需要更多的工作，但这是一个很好的起点。这也帮助我们欣赏多年来在硬件和软件方面所做的伟大工作，使得部署高可用服务成为可能，甚至变得容易，而这在几年前只有在大型数据中心才可能。

## 安装依赖项

运行上述示例需要以下依赖项：

1.  **VirtualBox**：VirtualBox 是一种流行的虚拟机管理软件。您可以从[`www.virtualbox.org/wiki/Downloads`](https://www.virtualbox.org/wiki/Downloads)下载适用于您平台的安装可执行文件。

1.  **Vagrant**：Vagrant 是一个开源工具，可以被视为 Docker 的虚拟机等效物。它可以从[`www.vagrantup.com/downloads.html`](https://www.vagrantup.com/downloads.html)下载。

1.  **Fleetctl**：Fleet 简而言之是一个分布式初始化系统，这意味着它将允许我们在集群级别管理服务。Fleetctl 是一个 CLI 客户端，用于运行 fleet 命令。要安装 fleetctl，请运行以下命令：

[PRE18]

## 获取并配置 Vagrantfile

Vagrantfiles 是 Dockerfiles 的 Vagrant 等价物。Vagrantfile 包含诸如获取基本虚拟机、运行设置命令、启动虚拟机镜像实例数量等详细信息。CoreOS 有一个包含 Vagrantfile 的存储库，可用于下载和在虚拟机中使用 CoreOS。这是在开发环境中尝试 CoreOS 功能的理想方式：

[PRE19]

上述命令克隆了包含 Vagrantfile 的`coreos-vagrant`存储库，该文件下载并启动基于 CoreOS 的虚拟机。

### 注意

Vagrant 是一款免费开源软件，用于创建和配置虚拟开发环境。它可以被视为围绕虚拟化软件（如 VirtualBox、KVM 或 VMware）和配置管理软件（如 Chef、Salt 或 Puppet）的包装器。您可以从[`www.vagrantup.com/downloads.html`](https://www.vagrantup.com/downloads.html)下载 Vagrant。

不过，在启动虚拟机之前，我们需要进行一些配置。

### 获取发现令牌

每个 CoreOS 主机都运行`etcd`服务的一个实例，以协调在该机器上运行的服务，并与集群中其他机器上运行的服务进行通信。为了实现这一点，`etcd`实例本身需要相互发现。

CoreOS 团队构建了一个发现服务（[`discovery.etcd.io`](https://discovery.etcd.io)），它提供了一个免费服务，帮助`etcd`实例通过存储对等信息相互通信。它通过提供一个唯一标识集群的令牌来工作。集群中的每个`etcd`实例都使用此令牌通过发现服务识别其他`etcd`实例。生成令牌很容易，只需通过`GET`请求发送到[discovery.etcd.io/new](http://discovery.etcd.io/new)即可。

[PRE20]

现在打开`coreos-vagrant`目录中名为`user-data.sample`的文件，并找到包含`etcd`服务下的`discovery`配置选项的注释行。取消注释并提供先前运行的`curl`命令返回的令牌。完成后，将文件重命名为`user-data`。

### 注意

`user-data`文件用于为 CoreOS 实例中的`cloud-config`程序设置配置参数。Cloud-config 受`cloud-init`项目中的`cloud-config`文件的启发，后者定义自己为处理云实例的早期初始化的事实多发行包（`cloud-init`文档）。简而言之，它有助于配置各种参数，如要打开的端口，在 CoreOS 的情况下，`etcd`配置等。您可以在以下网址找到更多信息：

[`coreos.com/docs/cluster-management/setup/cloudinit-cloud-config/`](https://coreos.com/docs/cluster-management/setup/cloudinit-cloud-config/)和[`cloudinit.readthedocs.org/en/latest/index.html`](http://cloudinit.readthedocs.org/en/latest/index.html)。

以下是 CoreOS 代码的示例：

[PRE21]

### 提示

每次运行集群时，您都需要生成一个新的令牌。简单地重用令牌将不起作用。

### 设置实例数量

在`coreos-vagrant`目录中，还有另一个名为`config.rb.sample`的文件。找到该文件中的注释行，其中写着`$num_instances=1`。取消注释并将值设置为`3`。这将使 Vagrant 生成三个 CoreOS 实例。现在将文件保存为`config.rb`。

### 注意

`cnfig.rb`文件保存了 Vagrant 环境的配置以及集群中的机器数量。

以下是 Vagrant 实例的代码示例：

[PRE22]

### 生成实例并验证健康

现在我们已经准备好配置，是时候在本地机器上看到一个运行的集群了：

[PRE23]

创建完机器后，您可以 SSH 登录到它们，尝试以下命令，但您需要将`ssh`密钥添加到您的 SSH 代理中。这样做将允许您将 SSH 会话转发到集群中的其他节点。要添加密钥，请运行以下命令：

[PRE24]

现在让我们验证一下机器是否正常运行，并要求 fleet 列出集群中正在运行的机器：

[PRE25]

### 启动服务

要在新启动的集群中运行服务，您将需要编写`unit-files`文件。单元文件是列出必须在每台机器上运行的服务以及如何管理这些服务的一些规则的配置文件。

创建三个名为`code.it.1.service`、`code.it.2.service`和`code.it.3.service`的文件。用以下配置填充它们：

`code.it.1.service`

[PRE26]

`code.it.2.service`

[PRE27]

`code.it.3.service`

[PRE28]

您可能已经注意到这些文件中的模式。`ExecStart`参数保存了必须执行的命令，以启动服务。在我们的情况下，这意味着运行`code.it`容器。`ExecStartPost`是在`ExecStart`参数成功后执行的命令。在我们的情况下，服务的可用性被注册在`etcd`服务中。相反，`ExecStop`命令将停止服务，而`ExecStopPost`命令在`ExecStop`命令成功后执行，这在这种情况下意味着从`etcd`服务中删除服务的可用性。

`X-Fleet`是 CoreOS 特有的语法，告诉 fleet 两个服务不能在同一台机器上运行（因为它们在尝试绑定到相同端口时会发生冲突）。现在所有的块都就位了，是时候将作业提交到集群了：

[PRE29]

让我们验证服务是否已提交到集群：

[PRE30]

机器列为空，活动状态未设置。这意味着我们的服务尚未启动。让我们启动它们：

[PRE31]

让我们通过再次执行`$ fleetctl list-units`文件来验证它们是否正在运行：

[PRE32]

恭喜！您刚刚建立了自己的集群！现在在 Web 浏览器中转到`172.17.8.101`、`172.17.8.102`或`172.17.8.103`，看看`code.it`应用程序正在运行！

在这个例子中，我们只是建立了一个运行高可用服务的机器集群。如果我们添加一个负载均衡器，它与`etcd`服务保持连接，将请求路由到可用的机器，我们将在我们的系统中运行一个完整的端到端生产级服务。但这样做会偏离主题，所以留给你作为练习。

通过这个，我们来到了尽头。Docker 仍在积极发展，像 CoreOS、Deis、Flynn 等项目也是如此。因此，尽管我们在过去几个月看到了很棒的东西，但即将到来的将会更好。我们生活在激动人心的时代。因此，让我们充分利用它，构建能让这个世界变得更美好的东西。祝愉快！

# 总结

在本章中，我们学习了如何使用 Docker 与 Chef 和 Puppet。然后我们设置了一个 apt-cacher 来加快软件包的下载速度。接下来，我们用 Dokku 搭建了自己的迷你 PaaS。最后，我们使用 CoreOS 和 Fleet 搭建了一个高可用性的服务。恭喜！我们一起获得了使用 Docker 构建容器、"dockerize"我们的应用甚至运行集群所需的知识。我们的旅程到此结束了。但是对于你，亲爱的读者，一个新的旅程刚刚开始。这本书旨在奠定基础，帮助你使用 Docker 构建下一个大事件。我祝你世界上一切成功。如果你喜欢这本书，在 Twitter 上给我发消息`@srikrishnaholla`。如果你不喜欢，也请告诉我如何改进。

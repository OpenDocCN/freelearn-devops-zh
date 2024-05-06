# 第二章：准备 Kubernetes 和 Helm 环境

Helm 是一个提供各种好处的工具，帮助用户更轻松地部署和管理 Kubernetes 应用程序。然而，在用户可以开始体验这些好处之前，他们必须满足一些先决条件。首先，用户必须能够访问 Kubernetes 集群。其次，用户应该具有 Kubernetes 和 Helm 的命令行工具。最后，用户应该了解 Helm 的基本配置选项，以便尽可能少地产生摩擦地提高生产力。

在本章中，我们将概述开始使用 Helm 所需的工具和概念。本章将涵盖以下主题：

+   使用 Minikube 准备本地 Kubernetes 环境

+   设置`kubectl`

+   设置 Helm

+   配置 Helm

# 技术要求

在本章中，您将在本地工作站上安装以下技术：

+   Minikube

+   VirtualBox

+   Helm

这些工具可以通过软件包管理器安装，也可以通过下载链接直接下载。我们将提供在 Windows 上使用`Chocolatey`软件包管理器，在 macOS 上使用`Homebrew`软件包管理器，在基于 Debian 的 Linux 发行版上使用`apt-get`软件包管理器，在基于 RPM 的 Linux 发行版上使用`dnf`软件包管理器的使用说明。

# 使用 Minikube 准备本地 Kubernetes 环境

没有访问 Kubernetes 集群，Helm 将无法部署应用程序。因此，让我们讨论一个用户可以遵循的选项，在他们的机器上运行自己的集群的选项—Minikube。

Minikube 是一个由社区驱动的工具，允许用户轻松在本地机器上部署一个小型的单节点 Kubernetes 集群。使用 Minikube 创建的集群是在一个虚拟机（VM）内创建的，因此可以在与运行 VM 的主机操作系统隔离的方式下创建和丢弃。Minikube 提供了一个很好的方式来尝试 Kubernetes，并且还可以用来学习如何在本书中提供的示例中使用 Helm。

在接下来的几节中，我们将介绍如何安装和配置 Minikube，以便在学习如何使用 Helm 时拥有一个可用的 Kubernetes 集群。有关更全面的说明，请参考官方 Minikube 网站的*入门*页面[`minikube.sigs.k8s.io/docs/start/`](https://minikube.sigs.k8s.io/docs/start/)。

## 安装 Minikube

与本章中将安装的其他工具一样，Minikube 的二进制文件是为 Windows、macOS 和 Linux 操作系统编译的。在 Windows 和 macOS 上安装最新版本的 Minikube 的最简单方法是通过软件包管理器，例如 Windows 的`Chocolatey`和 macOS 的`Homebrew`。

Linux 用户将发现，通过从 Minikube 的 GitHub 发布页面下载最新的`minikube`二进制文件更容易安装，尽管这种方法也可以在 Windows 和 macOS 上使用。

以下步骤描述了如何根据您的计算机和安装偏好安装 Minikube。请注意，在撰写本书中使用的示例的编写和开发过程中使用了 Minikube 版本 v1.5.2。

要通过软件包管理器安装它（在 Windows 和 macOS 上），请执行以下操作：

+   对于 Windows，请使用以下命令：

[PRE0]

+   对于 macOS，请使用以下命令：

[PRE1]

以下步骤向您展示了如何通过下载链接（在 Windows、macOS 和 Linux 上）安装它。

`Minikube`二进制文件可以直接从其在 Git 上的发布页面下载[Hub at https://github.com/kubernetes/minikube/re](https://github.com/kubernetes/minikube/releases/tag/v1.5.2)leases/：

1.  在发布页面的底部，有一个名为*Assets*的部分，其中包含了各种支持的平台可用的 Minikube 二进制文件：![图 2.1：来自 GitHub 发布页面的 Minikube 二进制文件](img/Figure_2.1.jpg)

图 2.1：来自 GitHub 发布页面的 minikube 二进制文件

1.  在**Assets**部分下，应下载与目标平台对应的二进制文件。下载后，您应将二进制文件重命名为`minikube`。例如，如果您正在下载 Linux 二进制文件，您将运行以下命令：

[PRE2]

1.  为了执行`minikube`，Linux 和 macOS 用户可能需要通过运行`chmod`命令添加可执行位：

[PRE3]

1.  然后，`minikube`应移动到由`PATH`变量管理的位置，以便可以从命令行的任何位置执行它。`PATH`变量包含的位置因操作系统而异。对于 macOS 和 Linux 用户，可以通过在终端中运行以下命令来确定这些位置：

[PRE4]

1.  Windows 用户可以通过在命令提示符或 PowerShell 中运行以下命令来确定`PATH`变量的位置：

[PRE5]

1.  然后，您可以使用 `mv` 命令将 `minikube` 二进制文件移动到新位置。以下示例将 `minikube` 移动到 Linux 上的常见 `PATH` 位置：

[PRE6]

1.  您可以通过运行 `minikube version` 并确保显示的版本与下载的版本相对应来验证 Minikube 的安装：

[PRE7]

尽管您已经下载了 Minikube，但您还需要一个 hypervisor 来运行本地 Kubernetes 集群。这可以通过安装 VirtualBox 来实现，我们将在下一节中描述。

## 安装 VirtualBox

Minikube 依赖于存在的 hypervisors，以便在虚拟机上安装单节点 Kubernetes 集群。对于本书，我们选择讨论 VirtualBox 作为 hypervisor 选项，因为它是最灵活的，并且可用于 Windows、macOS 和 Linux 操作系统。每个操作系统的其他 hypervisor 选项可以在官方 Minikube 文档中找到 [`minikube.sigs.k8s.io/docs/start/`](https://minikube.sigs.k8s.io/docs/start/)。

与 Minikube 一样，VirtualBox 可以通过 Chocolatey 或 Homebrew 轻松安装，但也可以使用 `apt-get`（Debian-based Linux）和 `dnf`（RPM/RHEL-based Linux）轻松安装：

+   在 Windows 上安装 VirtualBox 的代码如下：

[PRE8]

+   在 macOS 上安装 VirtualBox 的代码如下：

[PRE9]

+   在基于 Debian 的 Linux 上安装 VirtualBox 的代码如下：

[PRE10]

+   在 RHEL-based Linux 上安装 VirtualBox 的代码如下：

[PRE11]

可以在其官方下载页面 [`www.virtualbox.org/wiki/Downloads`](https://www.virtualbox.org/wiki/Downloads) 找到安装 VirtualBox 的其他方法。

安装了 VirtualBox 后，必须配置 Minikube 以利用 VirtualBox 作为其默认 hypervisor。此配置将在下一节中进行。

## 将 VirtualBox 配置为指定的 hypervisor

可以通过将 `minikube` 的 `vm-driver` 选项设置为 `virtualbox` 来将 VirtualBox 设置为默认 hypervisor：

[PRE12]

请注意，此命令可能会产生以下警告：

[PRE13]

如果工作站上没有活动的 Minikube 集群，则可以安全地忽略此消息。此命令表示任何现有的 Kubernetes 集群在删除并重新创建集群之前都不会使用 VirtualBox 作为 hypervisor。

可以通过评估 `vm-driver` 配置选项的值来确认切换到 VirtualBox：

[PRE14]

如果一切顺利，输出将如下所示：

[PRE15]

除了配置默认的 hypervisor 之外，您还可以配置分配给 Minikube 集群的资源，这将在下一节中讨论。

## 配置 Minikube 资源分配

默认情况下，Minikube 将为其虚拟机分配两个 CPU 和 2 GB 的 RAM。这些资源对本书中的每个示例都足够，除了*第七章*中更需要资源的示例。如果您的机器有可用资源，应该将默认内存分配增加到 4 GB（CPU 分配可以保持不变）。

运行以下命令将增加新 Minikube 虚拟机的默认内存分配为 4 GB（4000 MB）。

[PRE16]

可以通过运行`minikube config get memory`命令来验证此更改，类似于之前验证`vm-driver`更改的方式。

让我们继续探索 Minikube，讨论其基本用法。

## 探索基本用法

在本书中，了解典型 Minikube 操作中使用的关键命令将非常方便。在本书的示例执行过程中，了解这些命令也是至关重要的。幸运的是，Minikube 是一个很容易上手的工具。

Minikube 有三个关键子命令：

+   `start`

+   `stop`

+   `delete`

`start`子命令用于创建单节点 Kubernetes 集群。它将创建一个虚拟机并在其中引导集群。一旦集群准备就绪，命令将终止：

[PRE17]

`stop`子命令用于关闭集群和虚拟机。集群和虚拟机的状态将保存到磁盘上，允许用户再次运行`start`子命令快速开始工作，而不必从头开始构建新的虚拟机。当您完成对集群的工作并希望以后返回时，应该尝试养成运行`minikube stop`的习惯：

[PRE18]

`delete`子命令用于删除集群和虚拟机。此命令将擦除集群和虚拟机的状态，释放先前分配的磁盘空间。下次执行`minikube start`时，将创建一个全新的集群和虚拟机。当您希望删除所有分配的资源并在下次调用`minikube start`时在一个全新的 Kubernetes 集群上工作时，应该运行`delete`子命令：

[PRE19]

还有更多 Minikube 子命令可用，但这些是您应该知道的主要命令。

安装并配置了 Minikube 后，您现在可以安装`kubectl`，即 Kubernetes 命令行工具，并满足使用 Helm 的其余先决条件。

# 设置 Kubectl

如*第一章*中所述，*了解 Kubernetes 和 Helm*，Kubernetes 是一个公开不同 API 端点的系统。这些 API 端点用于在集群上执行各种操作，例如创建、查看或删除资源。为了提供更简单的用户体验，开发人员需要一种与 Kubernetes 交互的方式，而无需管理底层 API 层。

虽然在本书的过程中，您主要会使用 Helm 命令行工具来安装和管理应用程序，但`kubectl`是常见任务的必备工具。

继续阅读以了解如何在本地工作站上安装`kubectl`。请注意，写作时使用的`kubectl`版本为`v1.16.2`。

## 安装 Kubectl

`kubectl`可以使用 Minikube 安装，也可以通过软件包管理器或直接下载获取。我们首先描述如何使用 Minikube 获取`kubectl`。

### 通过 Minikube 安装 Kubectl

使用 Minikube 安装`kubectl`非常简单。Minikube 提供了一个名为`kubectl`的子命令，它将下载 Kubectl 二进制文件。首先运行`minikube kubectl`：

[PRE20]

此命令将`kubectl`安装到`$HOME/.kube/cache/v1.16.2`目录中。请注意，路径中包含的`kubectl`版本将取决于您使用的 Minikube 版本。要访问`kubectl`，可以使用以下语法：

[PRE21]

以下是一个示例命令：

[PRE22]

使用`minikube kubectl`调用`kubectl`就足够了，但是语法比直接调用`kubectl`更加笨拙。可以通过将`kubectl`可执行文件从本地 Minikube 缓存复制到由`PATH`变量管理的位置来克服这个问题。在每个操作系统上执行此操作类似，但以下是如何在 Linux 机器上实现的示例：

[PRE23]

完成后，`kubectl`可以作为独立的二进制文件调用，如下所示：

[PRE24]

### 在没有 Minikube 的情况下安装 Kubectl

Kubectl 也可以在没有 Minikube 的情况下安装。Kubernetes 官方文档提供了多种不同的机制来为各种目标操作系统进行安装，网址为 https://kubernetes.io/docs/tasks/tools/install-kubectl/。

### 使用软件包管理器

`kubectl`可以在没有 Minikube 的情况下通过本机软件包管理进行安装。以下列表演示了如何在不同的操作系统上完成此操作：

+   使用以下命令在 Windows 上安装`kubectl`：

[PRE25]

+   使用以下命令在 macOS 上安装`kubectl`：

[PRE26]

+   使用以下命令在基于 Debian 的 Linux 上安装`kubectl`：

[PRE27]

+   使用以下命令在基于 RPM 的 Linux 上安装`kubectl`：

[PRE28]

我们将在下一节讨论最终的 Kubectl 安装方法。

### 直接从链接下载

Kubectl 也可以直接从下载链接下载。下载链接将包含要下载的 Kubectl 版本。您可以通过在浏览器中访问[`storage.googleapis.com/kubernetes-release/release/stable.txt`](https://storage.googleapis.com/kubernetes-release/release/stable.txt)来确定 Kubectl 的最新版本。

以下示例说明了如何下载版本 v1.16.2，这是本书中使用的 Kubectl 版本：

+   从 https://storage.googleapis.com/kubernetes-release/release/v1.16.2/bin/windows/amd64/kubectl.exe 下载 Windows 的 Kubectl。

+   从 https://storage.googleapis.com/kubernetes-release/releas](https://storage.googleapis.com/kubernetes-release/release/v1.16.2/bin/darwin/amd64/kubectl)e/v1.16.2/bin/darwin/amd64/kubectl 下载 macOS 的 Kubectl。

+   从[`storage.googleapis.com/kubernetes-release/release/v1.16.2/bin/linux/amd64/kubectl`](https://storage.googleapis.com/kubernetes-release/release/v1.16.2/bin/linux/amd64/kubectl)下载 Linux 的 Kubectl。

Kubectl 二进制文件可以移动到由`PATH`变量管理的位置。在 macOS 和 Linux 操作系统上，确保授予可执行权限：

[PRE29]

可以通过运行以下命令来验证 Kubectl 的安装。

[PRE30]

现在我们已经介绍了如何设置`kubectl`，我们准备进入本书的关键技术——Helm。

设置 Helm

安装 Minikube 和`kubectl`后，下一个逻辑工具是配置 Helm。请注意，写作本书时使用的 Helm 版本是`v3.0.0`，但建议您使用 Helm v3 发布的最新版本，以获得最新的漏洞修复和 bug 修复。

## 安装 Helm

Chocolatey 和 Homebrew 都有 Helm 软件包，可以方便地在 Windows 或 macOS 上安装。在这些系统上，可以运行以下命令来使用软件包管理器安装 Helm：

+   使用以下命令在 Windows 上安装 Helm：

[PRE31]

+   使用以下命令在 macOS 上安装 Helm：

[PRE32]

Linux 用户或者宁愿从直接可下载链接安装 Helm 的用户可以按照以下步骤从 Helm 的 GitHub 发布页面下载存档文件：

1.  在 Helm 的 GitHub 发布页面上找到名为**Installati**[**on**的部分：](https://github.com/helm/helm/releases)：![图 2.2：Helm GitHub 发布页面上的安装部分](img/Figure_2.2.jpg)

图 2.2：Helm GitHub 发布页面上的安装部分

1.  下载与所使用操作系统对应版本的存档文件。

1.  下载后，需要解压文件。可以通过在 PowerShell 上使用`Expand-Archive`命令函数或在 Bash 上使用`tar`实用程序来实现这一点：

+   对于 Windows/PowerShell，请使用以下示例：

[PRE33]

+   对于 Linux 和 Mac，请使用以下示例：

[PRE34]

确保指定与下载版本对应的版本。`helm`二进制文件可以在未解压的文件夹中找到。它应该被移动到由`PATH`变量管理的位置。

以下示例向您展示了如何将`helm`二进制文件移动到 Linux 系统上的`/usr/local/bin`文件夹中：

[PRE35]

无论 Helm 是以何种方式安装的，都可以通过运行`helm version`命令来进行验证。如果结果输出类似于以下输出，则 Helm 已成功安装：

[PRE36]

安装了 Helm 后，继续下一部分，了解基本的 Helm 配置主题。

# 配置 Helm

Helm 是一个具有合理默认值的工具，允许用户在安装后无需执行大量任务即可提高生产力。话虽如此，用户可以更改或启用几种不同的选项来修改 Helm 的行为。我们将在接下来的部分中介绍这些选项，首先是配置上游仓库。

## 添加上游仓库

用户可以开始修改他们的 Helm 安装的一种方式是添加上游图表存储库。在[*第一章*]中，*理解 Kubernetes 和 Helm*，我们描述了图表存储库包含 Helm 图表，用于打包 Kubernetes 资源文件。作为 Kubernetes 包管理器的 Helm，可以连接到各种图表存储库来安装 Kubernetes 应用程序。

Helm 提供了 `repo` 子命令，允许用户管理配置的图表存储库。这个子命令包含其他子命令，可以用来执行针对指定存储库的操作。

以下是五个 `repo` 子命令：

+   `add`：添加图表存储库

+   `list`：列出图表存储库

+   `remove`：删除图表存储库

+   `update`：从图表存储库更新本地可用图表的信息

+   `index`：根据包含打包图表的目录生成索引文件

使用上述列表作为指南，可以使用 `repo add` 子命令来添加图表存储库，如下所示：

[PRE37]

为了安装其中管理的图表，需要添加图表存储库。本书将详细讨论图表安装。

您可以通过利用 `repo list` 子命令来确认存储库是否已成功添加：

[PRE38]

已添加到 Helm 客户端的存储库将显示在此输出中。前面的示例显示，`bitnami` 存储库已添加，因此它出现在 Helm 客户端已知的存储库列表中。如果添加了其他存储库，它们也将出现在此输出中。

随着时间的推移，更新的图表将被发布并发布到这些存储库中。存储库元数据被本地缓存。因此，Helm 不会自动意识到图表已更新。您可以通过运行 `repo update` 子命令来指示 Helm 从每个添加的存储库检查更新。一旦执行了这个命令，您就可以从每个存储库安装最新的图表：

[PRE39]

您可能还需要删除先前添加的存储库。这可以通过使用 `repo remove` 子命令来完成：

[PRE40]

最后剩下的 `repo` 子命令形式是 `index`。这个子命令被存储库和图表维护者用来发布新的或更新的图表。这个任务将在[*第五章*]中更详细地介绍，*构建您的第一个 Helm 图表*。

接下来，我们将讨论 Helm 插件配置。

## 添加插件

插件是可以用来为 Helm 提供额外功能的附加功能。大多数用户不需要担心 Helm 的插件和插件管理。Helm 本身就是一个强大的工具，并且在开箱即用时就具备了它承诺的功能。话虽如此，Helm 社区维护了各种不同的插件，可以用来增强 Helm 的功能。这些插件的列表可以在[`helm.sh/docs/community/related/`](https://helm.sh/docs/community/related/)找到。

Helm 提供了一个`plugin`子命令来管理插件，其中包含进一步的子命令，如下表所述：

![](img/011.jpg)

插件可以提供各种不同的生产力增强。

以下是一些上游插件的示例：

+   `helm diff`: 在部署的发布和建议的 Helm 升级之间执行差异

+   `helm secrets`: 用于帮助隐藏 Helm 图表中的秘密

+   `helm monitor`: 用于监视发布并在发生特定事件时执行回滚

+   `helm unittest`: 用于对 Helm 图表执行单元测试

我们将继续讨论 Helm 配置选项，通过审查可以设置的不同环境变量来改变 Helm 行为的各个方面。

## 环境变量

Helm 依赖于外部化变量的存在来配置低级选项。Helm 文档列出了用于配置 Helm 的六个主要环境变量：

+   **XDG_CACHE_HOME**: 设置存储缓存文件的替代位置

+   **XDG_CONFIG_HOME**: 设置存储 Helm 配置的替代位置

+   **XDG_DATA_HOME**: 设置存储 Helm 数据的替代位置

+   **HELM_DRIVER**: 设置后端存储驱动程序

+   **HELM_NO_PLUGINS**: 禁用插件

+   **KUBECONFIG**: 设置替代的 Kubernetes 配置文件

Helm 遵循**XDG 基本目录规范**，该规范旨在提供一种标准化的方式来定义操作系统文件系统上不同文件的位置。根据 XDG 规范，Helm 会根据需要在每个操作系统上自动创建三个不同的默认目录：

![](img/02.jpg)

Helm 使用缓存路径存储从上游图表存储库下载的图表。安装的图表被缓存到本地机器，以便在下次引用时更快地安装图表。要更新缓存，用户可以运行`helm repo update`命令，这将使用最新可用的信息刷新存储库元数据，并将图表保存到本地缓存中。

配置路径用于保存通过运行`helm repo add`命令添加的存储库信息。当安装尚未缓存的图表时，Helm 使用配置路径查找图表存储库的 URL。Helm 使用该 URL 来了解图表所在的位置以便下载。

数据路径用于存储插件。当使用`helm plugin install`命令安装插件时，插件数据存储在此位置。

关于我们之前详细介绍的其余环境变量，`HELM_DRIVER`用于确定发布状态在 Kubernetes 中的存储方式。默认值为`secret`，这也是推荐的值。`Secret`将在 Kubernetes **Secret**中对状态进行 Base64 编码。其他选项包括`configmap`，它将在明文 Kubernetes ConfigMap 中存储状态，以及`memory`，它将在本地进程的内存中存储状态。本地内存的使用是为了测试目的，不适用于通用或生产环境。

`HELM_NO_PLUGINS`环境变量用于禁用插件。如果未设置，默认值将保持插件启用为`0`。要禁用插件，应将变量设置为`1`。

`KUBECONFIG`环境变量用于设置用于对 Kubernetes 集群进行身份验证的文件。如果未设置，默认值将为`~/.kube/config`。在大多数情况下，用户不需要修改此值。

Helm 的另一个可配置的组件是选项卡完成，接下来讨论。

## 选项卡完成

Bash 和 Z shell 用户可以启用选项卡完成以简化 Helm 的使用。选项卡完成允许在按下*Tab*键时自动完成 Helm 命令，使用户能够更快地执行任务并帮助防止输入错误。

这类似于大多数现代终端仿真器的默认行为。当按下*Tab*键时，终端会通过观察命令和环境的状态来猜测下一个参数应该是什么。例如，在 Bash shell 中，`cd /usr/local/b`输入可以通过 Tab 补全为`cd /usr/local/bin`。类似地，输入`helm upgrade hello-`可以通过 Tab 补全为`helm upgrade hello-world`。

可以通过运行以下命令启用 Tab 补全：

[PRE41]

`$SHELL`变量必须是`bash`或`zsh`。请注意，自动补全只存在于运行前述命令的终端窗口中，因此其他窗口也需要运行此命令才能体验到自动补全功能。

## 身份验证

Helm 需要能够通过`kubeconfig`文件对 Kubernetes 集群进行身份验证，以便部署和管理应用程序。它通过引用`kubeconfig`文件进行身份验证，该文件指定了不同的 Kubernetes 集群以及如何对其进行身份验证。

在阅读本书时使用 Minikube 的人不需要配置身份验证，因为 Minikube 在每次创建新集群时会自动配置`kubeconfig`文件。然而，没有运行 Minikube 的人可能需要创建`kubeconfig`文件或者根据使用的 Kubernetes 发行版提供一个。

`kubeconfig`文件可以通过利用三个不同的`kubectl`命令来创建：

+   第一个命令是`set-cluster`：

[PRE42]

`set-cluster`命令将在`kubeconfig`文件中定义一个`cluster`条目。它确定 Kubernetes 集群的主机名或 IP 地址，以及其证书颁发机构。

+   下一个命令是`set-credentials`：

[PRE43]

`set-credentials`命令将定义用户的名称以及其身份验证方法和详细信息。此命令可以配置用户名和密码对、客户端证书、持有者令牌或身份验证提供程序，以允许用户和管理员指定不同的身份验证方法。

+   然后，我们有`set-context`命令：

[PRE44]

`set-context`命令用于将凭据与集群关联起来。一旦建立了凭据和集群之间的关联，用户将能够使用凭据的身份验证方法对指定的集群进行身份验证。

`kubectl config view`命令可用于查看`kubeconfig`文件。注意`kubeconfig`的`clusters`、`contexts`和`user`部分与先前描述的命令相对应，如下所示：

[PRE45]

一旦存在有效的 kubeconfig 文件，Kubectl 和 Helm 将能够与 Kubernetes 集群进行交互。

在下一节中，我们将讨论授权如何针对 Kubernetes 集群进行处理。

## 授权/RBAC

身份验证是确认身份的一种方式，授权定义了经过身份验证的用户被允许执行的操作。Kubernetes 使用基于角色的访问控制（RBAC）来执行对 Kubernetes 的授权。RBAC 是一种设计角色和特权的系统，可以分配给特定用户或用户组。用户被允许在 Kubernetes 上执行的操作取决于用户被分配的角色。

Kubernetes 在平台上提供了许多不同的角色。这里列出了三种常见的角色：

+   `cluster-admin`：允许用户对整个集群中的任何资源执行任何操作

+   `edit`：允许用户在命名空间或逻辑分组的大多数资源中进行读写

+   `view`：防止用户修改现有资源，只允许用户在命名空间内读取资源

由于 Helm 使用 kubeconfig 文件中定义的凭据对 Kubernetes 进行身份验证，因此 Helm 被赋予与文件中定义的用户相同级别的访问权限。如果启用了`edit`访问权限，Helm 可以假定具有足够的权限来安装应用程序，在大多数情况下。对于仅具有查看权限的情况，Helm 将无法安装应用程序，因为这种访问级别是只读的。

运行 Minikube 的用户在集群创建后默认被赋予`cluster-admin`权限。虽然这在生产环境中不是最佳做法，但对于学习和实验是可以接受的。运行 Minikube 的用户不必担心配置授权以便跟随本书提供的概念和示例。那些使用其他不是 Minikube 的 Kubernetes 集群的用户需要确保他们至少被赋予编辑角色才能够使用 Helm 部署大多数应用程序。可以通过要求管理员运行以下命令来实现这一点：

[PRE46]

在*第九章*中将讨论 RBAC 的最佳实践，*Helm 安全考虑*，我们将更详细地讨论与安全相关的概念，包括如何适当地应用角色以防止集群中的错误或恶意意图。

# 总结

使用 Helm 需要准备各种不同的组件。在本章中，您学习了如何安装 Minikube，以提供可用于本书的本地 Kubernetes 集群。您还学习了如何安装 Kubectl，这是与 Kubernetes API 交互的官方工具。最后，您学习了如何安装 Helm 客户端，并探索了 Helm 的各种配置方式，包括添加存储库和插件，修改环境变量，启用选项卡完成，并配置针对 Kubernetes 集群的身份验证和授权。

现在您已经安装了必备的工具，可以开始学习如何使用 Helm 部署您的第一个应用程序。在下一章中，您将从上游图表存储库安装 Helm 图表，并了解生命周期管理和应用程序配置。完成本章后，您将了解 Helm 如何作为 Kubernetes 的软件包管理器。

# 进一步阅读

查看以下链接，了解 Minikube、Kubectl 和 Helm 的安装选项：

+   Minikube：[`kubernetes.io/docs/tasks/tools/install-minikube/`](https://kubernetes.io/docs/tasks/tools/install-minikube/)

+   Kubectl：[`kubernetes.io/docs/tasks/tools/install-kubectl/`](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

+   Helm：[`helm.sh/docs/intro/install/`](https://helm.sh/docs/intro/install/)

我们介绍了各种不同的 Helm 后安装配置方式。查看以下链接，了解更多有关以下主题的信息：

+   存储库管理：[`helm.sh/docs/intro/quickstart/#initialize-a-helm-chart-repository`](https://helm.sh/docs/intro/quickstart/#initialize-a-helm-chart-repository)

+   插件管理：[`helm.sh/docs/topics/plugins/`](https://helm.sh/docs/topics/plugins/)

+   环境变量和`helm help`输出：[`helm.sh/docs/helm/helm/`](https://helm.sh/docs/helm/helm/)

+   选项卡完成：[`helm.sh/docs/helm/helm_completion/`](https://helm.sh/docs/helm/helm_completion/)

+   通过`kub`[`econfig`文件进行身份验证和授权：https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/

# 问题

1.  您可以列出安装 Helm 客户端的各种方法吗？

1.  Helm 如何对 Kubernetes 集群进行身份验证？

1.  有什么机制可以为 Helm 客户端提供授权？管理员如何管理这些权限？

1.  `helm repo add`命令的目的是什么？

1.  Helm 使用的三个 XDG 环境变量是什么？它们有什么作用？

1.  为什么 Minikube 是学习如何使用 Kubernetes 和 Helm 的好选择？Minikube 自动配置了哪些内容，以使用户能够更快地开始使用？

# 第六章：使容器工作

在本章中，我们将探索使用特权模式和超级特权模式容器创建 Docker 容器的各种选项。我们还将探索这些模式的各种故障排除问题。

我们将深入研究各种部署管理工具，如**Chef**、**Puppet**和**Ansible**，它们与 Docker 集成，以便为生产环境部署数千个容器减轻痛苦。

在本章中，我们将涵盖以下主题：

+   特权容器和超级特权容器

+   解决使用不同设置选项创建容器时遇到的问题

+   使 Docker 容器与 Puppet、Ansible 和 Chef 配合工作

+   使用 Puppet 创建 Docker 容器并部署应用程序

+   使用 Ansible 管理 Docker 容器

+   将 Docker 和 Ansible 一起构建

+   用于 Docker 的 Chef

利用前述管理工具自动化 Docker 容器的部署具有以下优势：

+   **灵活性**：它们为您提供了在云实例或您选择的裸机上复制基于 Docker 的应用程序以及 Docker 应用程序所需的环境的灵活性。这有助于管理和测试，以及根据需要提供开发环境。

+   **可审计性**：这些工具还提供了审计功能，因为它们提供了隔离，并帮助跟踪任何潜在的漏洞以及在哪个环境中部署了什么类型的容器。

+   **普遍性**：它们帮助您管理容器周围的整个环境，即管理容器以及非容器环境，如存储、数据库和容器应用程序周围的网络模型。

# 特权容器

默认情况下，容器以非特权模式运行，也就是说，我们不能在 Docker 容器内运行 Docker 守护程序。但是，特权 Docker 容器被赋予对所有设备的访问权限。Docker 特权模式允许访问主机上的所有设备，并在**App Armor**和**SELinux**中设置系统配置，以允许容器与主机上运行的进程具有相同的访问权限：

![特权容器](img/image_06_001-1.jpg)

突出显示的特权容器

特权容器可以使用以下命令启动：

[PRE0]

正如我们所看到的，启动特权模式容器后，我们可以列出连接到主机机器的所有设备。

## 故障排除提示

Docker 允许您通过支持添加和删除功能来使用非默认配置文件。最好删除容器进程不需要的功能，这样可以使其更安全。

如果您的主机系统上运行的容器面临安全威胁，通常建议检查是否有任何容器以特权模式运行，这可能会通过运行安全威胁应用程序来影响主机系统的安全。

如下例所示，当我们以非特权模式运行容器时，无法更改内核参数，但当我们使用`--privileged`标志以特权模式运行容器时，它可以轻松更改内核参数，这可能会在主机系统上造成安全漏洞：

[PRE1]

因此，在审核时，您应确保主机系统上运行的所有容器的特权模式未设置为`true`，除非某些特定应用程序在 Docker 容器中运行时需要：

[PRE2]

# 超级特权容器

这个概念是在 Redhat 的 Project Atomic 博客中介绍的。它提供了使用特殊/特权容器作为代理来控制底层主机的能力。如果我们只发布应用程序代码，我们就有将容器变成黑匣子的风险。将代理打包为具有正确访问权限的 Docker 容器对主机有许多好处。我们可以通过`-v /dev:/dev`绑定设备，这将帮助在容器内部挂载设备而无需超级特权访问。

使用`nsenter`技巧，允许您在另一个命名空间中运行命令，也就是说，如果 Docker 有自己的私有挂载命名空间，通过`nsenter`和正确的模式，我们可以到达主机并在其命名空间中挂载东西。

我们可以以特权模式运行，将整个主机系统挂载到某个路径（`/media/host`）上：

[PRE3]

然后我们可以在容器内部使用`nsenter`；`--mount`告诉`nsenter`查看`/media/host`，然后选择 proc 编号 1 的挂载命名空间。然后，运行常规挂载命令将设备链接到挂载点。如前所述，此功能允许我们挂载主机套接字和设备，例如文件，因此所有这些都可以绑定到容器中供使用：

![超级特权容器](img/image_06_002-2.jpg)

作为超级特权容器运行的 nsenter 监视主机

基本上，超级特权容器不仅提供安全隔离、资源和进程隔离，还提供了一种容器的运输机制。允许软件以容器镜像的形式进行运输，也允许我们管理主机操作系统和管理其他容器进程，就像之前解释的那样。

让我们考虑一个例子，目前，我们正在加载应用程序所需的内核模块，这些模块是主机操作系统中不包括的 RPM 软件包，并在应用程序启动时运行它们。这个模块可以通过超级特权容器的帮助进行运输，好处是这个自定义内核模块可以与当前内核非常好地配合，而不是将内核模块作为特权容器的一部分进行运输。在这种方法中，不需要将应用程序作为特权容器运行；它们可以分开运行，内核模块可以作为不同镜像的一部分加载，如下所示：

[PRE4]

## 故障排除 - 大规模的 Docker 容器

在生产环境中工作意味着持续部署。当基础设施是分散的和基于云的时，我们经常在相同的系统上管理相同服务的部署。自动化整个配置和管理这个系统的过程将是一个福音。部署管理工具就是为此目的而设计的。它们提供配方、剧本和模板，简化编排和自动化，提供标准和一致的部署。在接下来的章节中，我们将探讨三种常见的配置自动化工具：Chef、Puppet 和 Ansible，以及它们在大规模部署 Docker 容器时提供的便利。

# 木偶

Puppet 是一个自动化引擎，执行自动化的管理任务，如更新配置、添加用户和根据用户规范安装软件包。 Puppet 是一个众所周知的开源配置管理工具，可在各种系统上运行，如 Microsoft Windows、Unix 和 Linux。用户可以使用 Puppet 的声明性语言或特定领域语言（Ruby）描述配置。 Puppet 是模型驱动的，使用时需要有限的编程知识。 Puppet 提供了一个用于管理 Docker 容器的模块。 Puppet 和 Docker 集成可以帮助轻松实现复杂的用例。 Puppet 管理文件、软件包和服务，而 Docker 将二进制文件和配置封装在容器中，以便部署为应用程序。

Puppet 的一个潜在用例是，它可以用于为 Jenkins 构建所需的 Docker 容器进行配置，并且可以根据开发人员的需求进行规模化，即在触发构建时。构建过程完成后，二进制文件可以交付给各自的所有者，并且每次构建后都可以销毁容器。在这种用例中，Puppet 扮演着非常重要的角色，因为代码只需使用 Puppet 模板编写一次，然后可以根据需要触发：

![Puppet](img/image_06_003-1.jpg)

将 Puppet 和 Jenkins 集成以部署构建 Docker 容器

可以根据`garethr-docker` GitHub 项目安装用于管理 Docker 的 Puppet 模块。该模块只需要包含一个类：

[PRE5]

它设置了一个 Docker 托管的存储库，并安装了 Docker 软件包和任何所需的内核扩展。 Docker 守护程序将绑定到`unix socket /var/run/docker.sock`；根据需求，可以更改此配置：

[PRE6]

如前面的代码所示，Docker 的默认配置可以根据此模块提供的配置进行更改。

## 图像

可以使用此处详细说明的配置语法来拉取 Docker 镜像。

`ubuntu:trusty docker`命令的替代方法如下：

[PRE7]

甚至配置允许链接到 Dockerfile 以构建镜像。也可以通过订阅外部事件（如 Dockerfile 中的更改）来触发镜像的重建。我们订阅`vkohli/Dockerfile`文件夹中的更改，如下所示：

[PRE8]

## 容器

创建图像后，可以使用多个可选参数启动容器。我们可以使用基本的`docker run`命令获得类似的功能：

[PRE9]

如下所示，我们还可以传递一些更多的参数，例如以下内容：

+   `pull_on_start`：在启动图像之前，每次都会重新拉取它

+   `before_stop`：在停止容器之前将执行所述命令

+   `extra_parameters`：传递给`docker run`命令所需的附加数组参数，例如`--restart=always`

+   `after`：此选项允许表达需要首先启动的容器

可以设置的其他参数包括`ports`、`expose`、`env_files`和`volumes`。可以传递单个值或值数组。

## 网络

最新的 Docker 版本已经官方支持网络：该模块现在公开了一种类型，Docker 网络，可以用来管理它们：

[PRE10]

正如前面的代码所示，可以创建一个新的覆盖网络`sample-net`，并配置 Docker 守护程序以使用它。

## Docker compose

Compose 是一个用于运行多个 Docker 容器应用程序的工具。使用 compose 文件，我们可以配置应用程序的服务并启动它们。提供了`docker_compose`模块类型，允许 Puppet 轻松运行 compose 应用程序。

还可以添加一个 compose 文件，例如运行四个容器的缩放规则，如下面的代码片段所示。我们还可以提供网络和其他配置所需的附加参数：

[PRE11]

1.  如果 Puppet 程序未安装在您的计算机上，可以按以下方式进行安装：

[PRE12]

1.  在安装 Puppet 模块之后，可以按照所示安装`garethr-docker`模块：

[PRE13]

1.  我们将创建一个示例 hello world 应用程序，将使用 Puppet 进行部署：

[PRE14]

1.  创建文件后，我们应用（运行）它：

[PRE15]

1.  我们可以将其附加到容器并查看输出：

[PRE16]

如前所示，容器可以部署在多个主机上，并且整个集群可以通过单个 Puppet 配置文件创建。

## 故障排除提示

如果即使 Puppet `apply`命令成功运行后，仍无法列出 Docker 镜像，请检查语法和是否在示例文件中放置了正确的镜像名称。

# Ansible

Ansible 是一个工作流编排工具，通过一个易于使用的平台提供配置管理、供应和应用程序部署的帮助。Ansible 的一些强大功能如下：

+   **供应**：应用程序在不同的环境中开发和部署。可以是裸金属服务器、虚拟机或 Docker 容器，在本地或云上。Ansible 可以通过 Ansible tower 和 playbooks 来简化供应步骤。

+   **配置管理**：保持一个通用的配置文件是 Ansible 的主要用例之一，有助于在所需的环境中进行管理和部署。

+   **应用程序部署**：Ansible 有助于管理应用程序的整个生命周期，从部署到生产。

+   **持续交付**：管理持续交付流水线需要来自各个团队的资源。这不能仅靠简单的平台实现，因此，Ansible playbooks 在部署和管理应用程序的整个生命周期中发挥着重要作用。

+   **安全和合规性**：安全性可以作为部署阶段的一个组成部分，通过将各种安全策略作为自动化流程的一部分，而不是作为事后的思考过程或稍后合并。

+   **编排**：如前所述，Ansible 可以定义管理多个配置的方式，与它们交互，并管理部署脚本的各个部分。

## 使用 Ansible 自动化 Docker

Ansible 还提供了一种自动化 Docker 容器的方式；它使我们能够将 Docker 容器构建和自动化流程进行通道化和操作化，这个过程目前大多数情况下是手动处理的。Ansible 为编排 Docker 容器提供了以下模块：

+   **Docker_service**：现有的 Docker compose 文件可以用于通过 Ansible 的 Docker 服务部分在单个 Docker 守护程序或集群上编排容器。Docker compose 文件与 Ansible playbook 具有相同的语法，因为它们都是**Yaml**文件，语法几乎相同。Ansible 也是用 Python 编写的，Docker 模块使用的是 docker compose 在内部使用的确切 docker-py API 客户端。

这是一个简单的 Docker compose 文件：

[PRE17]

前面的 Docker compose 文件的 Ansible playbook 看起来很相似：

[PRE18]

+   **docker_container**：通过提供启动、停止、创建和销毁 Docker 容器的能力，来管理 Docker 容器的生命周期。

+   **docker_image**：这提供了帮助来管理 Docker 容器的镜像，包括构建、推送、标记和删除 Docker 镜像的命令。

+   **docker_login**：这将与 Docker hub 或任何 Docker 注册表进行身份验证，并提供从注册表推送和拉取 Docker 镜像的功能。

## Ansible Container

Ansible Container 是一个工具，仅使用 Ansible playbooks 来编排和构建 Docker 镜像。可以通过创建 `virtualenv` 并使用 pip 安装的方式来安装 Ansible Container：

[PRE19]

## 故障排除提示

如果您在安装 Ansible Container 方面遇到问题，可以通过从 GitHub 下载源代码来进行安装：

[PRE20]

Ansible Container 有以下命令可供开始使用：

+   **ansible_container init**：此命令创建一个用于开始的 Ansible 文件目录。

[PRE21]

+   **ansible-container build**：这将从 Ansible 目录中的 playbooks 创建镜像

+   **ansible-container run**：这将启动 `container.yml` 文件中定义的容器

+   **ansible-container push**：这将根据用户选择将项目的镜像推送到私有或公共仓库

+   **ansible-container shipit**：这将导出必要的 playbooks 和角色以部署容器到支持的云提供商

如在 GitHub 上的示例中所示，可以在 `container.yml` 文件中以以下方式定义 Django 服务：

[PRE22]

# Chef

Chef 有一些重要的组件，如 cookbook 和 recipes。Cookbook 定义了一个场景并包含了一切；其中第一个是 recipes，它是组织中的一个基本配置元素，使用 Ruby 语言编写。它主要是使用模式定义的资源集合。Cookbook 还包含属性值、文件分发和模板。Chef 允许以可版本控制、可测试和可重复的方式管理 Docker 容器。它为基于容器的开发提供了构建高效工作流和管理发布流水线的能力。Chef delivery 允许您自动化并使用可扩展的工作流来测试、开发和发布 Docker 容器。

Docker cookbook 可在 GitHub 上找到（[`github.com/chef-cookbooks/docker`](https://github.com/chef-cookbooks/docker)），并提供自定义资源以在配方中使用。它提供了各种选项，例如以下内容：

+   `docker_service`：这些是用于 `docker_installation` 和 `docker_service` 管理器的复合资源

+   `docker_image`: 这个用于从仓库中拉取 Docker 镜像

+   `docker_container`: 这个处理所有 Docker 容器操作

+   `docker_registry`: 这个处理所有 Docker 注册操作

+   `docker_volume`: 这个管理 Docker 容器所有卷相关的操作

以下是一个样本 Chef Docker 配方，可用作参考以使用 Chef 配方部署容器：

[PRE23]

# 总结

在本章中，我们首先深入研究了特权容器，它可以访问所有主机设备以及超级特权容器，展示了容器管理在后台运行服务的能力，这可以用于在 Docker 容器中运行服务以管理底层主机。然后，我们研究了 Puppet，一个重要的编排工具，以及它如何借助 `garethr-docker` GitHub 项目来处理容器管理。我们还研究了 Ansible 和 Chef，它们提供了类似的能力，可以规模化地管理 Docker 容器。在下一章中，我们将探索 Docker 网络堆栈。

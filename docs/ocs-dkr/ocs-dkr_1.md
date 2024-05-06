# 第一章：打开 Docker

**Docker**是一种轻量级的容器化技术，在近年来广受欢迎。它利用了 Linux 内核的一系列特性，如命名空间、cgroups、AppArmor 配置文件等，将进程隔离到可配置的虚拟环境中。

在本章中，您将学习如何在各种系统上安装 Docker，无论是在开发还是生产环境。对于基于 Linux 的系统，由于内核已经可用，安装就像`apt-get install`或`yum install`命令一样简单。然而，要在 OSX 和 Windows 等非 Linux 操作系统上运行 Docker，您需要安装 Docker Inc.开发的一个辅助应用程序，称为**Boot2Docker**。这将在**VirtualBox**上安装一个轻量级的 Linux 虚拟机，通过**Internet** **Assigned** **Numbers** **Authority** (**IANA**)分配的端口 2375，使 Docker 可用。

在本章结束时，您将在您的系统上安装了 Docker，无论是在开发还是生产环境，并进行了验证。

本章将涵盖以下内容：

+   介绍 Docker

+   安装 Docker

+   Ubuntu（14.04 和 12.04）

+   Mac OSX 和 Windows

+   OpenStack

+   Inception：在 Docker 中构建 Docker

+   验证安装：`Hello` `World` 输出

+   介绍 Docker

Docker 是由 DotCloud Inc.（目前是 Docker Inc.）开发的，作为他们构建的**Platform** **as** **a** **Service** (**PaaS**)的框架。当他们发现开发人员对这项技术越来越感兴趣时，他们将其作为开源发布，并自那时起宣布他们将完全专注于 Docker 技术的发展，这是一个好消息，因为这意味着平台将得到持续的支持和改进。

已经有许多旨在使分布式应用程序成为可能，甚至易于设置的工具和技术，但没有一个像 Docker 一样具有如此广泛的吸引力，这主要是因为它的跨平台性和对系统管理员和开发人员的友好性。在任何操作系统上都可以设置 Docker，无论是 Windows、OSX 还是 Linux，Docker 容器在任何地方都可以以相同的方式工作。这是非常强大的，因为它实现了一次编写，到处运行的工作流程。Docker 容器保证在开发桌面、裸机服务器、虚拟机、数据中心或云上以相同的方式运行。不再出现程序在开发人员的笔记本电脑上运行但在服务器上不运行的情况。

Docker 的工作流程的性质使得开发人员可以完全专注于构建应用程序并在容器内运行它们，而系统管理员可以专注于在部署中运行容器。这种角色的分离和一个单一的基础工具的存在使得代码管理和部署过程变得简单。

但是虚拟机不是已经提供了所有这些功能吗？

**虚拟机**（**VMs**）是完全虚拟化的。这意味着它们在彼此之间共享最少的资源，每个虚拟机都有其自己分配的资源集。虽然这允许对各个虚拟机进行细粒度配置，但最小的共享也意味着更大的资源使用、冗余的运行进程（需要运行整个操作系统！）和因此性能开销。

另一方面，Docker 建立在容器技术之上，它隔离一个进程并使其相信自己在独立的操作系统上运行。该进程仍然在与其主机相同的操作系统中运行，共享其内核。它使用了一个名为**Another** **Unionfs**（**AUFS**）的分层写时复制文件系统，它在容器之间共享操作系统的公共部分。更大的共享当然只能意味着更少的隔离，但 Linux 进程资源管理解决方案的巨大改进，如命名空间和 cgroups，已经使 Docker 实现了类似虚拟机的进程隔离，同时保持了非常小的资源占用。

让我们来看一下以下图片：

![拆箱 Docker](img/4787_01_04.jpg)

这是一个 Docker 与虚拟机的比较。容器与其他容器和进程共享主机的资源，而虚拟机必须为每个实例运行整个操作系统。

# 安装 Docker

Docker 在大多数主要 Linux 发行版的标准存储库中都有。我们将看一下 Ubuntu 14.04 和 12.04（Trusty 和 Precise）、Mac OSX 和 Windows 中 Docker 的安装程序。如果您目前使用的操作系统不在上述列表中，您可以在[`docs.docker.com/installation/#installation`](https://docs.docker.com/installation/#installation)上查找您操作系统的说明。

## 在 Ubuntu 中安装 Docker

Ubuntu 从 Ubuntu 12.04 开始支持 Docker。请记住，您仍然需要 64 位操作系统才能运行 Docker。让我们来看一下 Ubuntu 14.04 的安装说明。

### 在 Ubuntu Trusty 14.04 LTS 中安装 Docker

Docker 作为一个软件包在 Ubuntu Trusty 版本的软件存储库中以`docker.io`的名称可用：

```
$ sudo apt-get update
$ sudo apt-get -y install docker.io

```

就是这样！您现在已经在系统上安装了 Docker。但是，由于命令已更名为`docker.io`，您将不得不使用`docker.io`而不是`docker`来运行所有 Docker 命令。

### 注意

该软件包的名称为`docker.io`，因为它与另一个名为`docker`的 KDE3/GNOME2 软件包冲突。如果您更愿意以`docker`运行命令，可以创建一个符号链接到`/usr/local/bin`目录。第二个命令将自动完成规则添加到 bash：

```
$ sudo ln -s /usr/bin/docker.io /usr/local/bin/docker
$ sudo sed -i '$acomplete -F _docker docker' \> /etc/bash_completion.d/docker.io
```

### 在 Ubuntu Precise 12.04 LTS 中安装 Docker

Ubuntu 12.04 带有较旧的内核（3.2），与 Docker 的一些依赖项不兼容。因此，我们需要升级它：

```
$ sudo apt-get update
$ sudo apt-get -y install linux-image-generic-lts-raring linux-headers-generic-lts-raring
$ sudo reboot

```

我们刚刚安装的内核内置了 AUFS，这也是 Docker 的要求。

现在让我们结束安装：

```
$ curl -s https://get.docker.io/ubuntu/ | sudo sh

```

这是一个用于简单安装的`curl`脚本。查看此脚本的各个部分将帮助我们更好地理解该过程：

1.  首先，脚本检查我们的**高级** **软件包** **工具**（**APT**）系统是否能处理`https` URL，并在无法处理时安装`apt-transport-https`：

```
# Check that HTTPS transport is available to APT
if [ ! -e /usr/lib/apt/methods/https ]; then  apt-get update  apt-get install -y apt-transport-https
fi

```

1.  然后它将 Docker 存储库添加到我们的本地密钥链中：

```
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9

```

### 提示

您可能会收到一个警告，表示软件包不受信任。回答`yes`以继续安装。

1.  最后，它将 Docker 存储库添加到 APT 源列表中，并更新并安装`lxc-docker`软件包：

```
$ sudo sh -c "echo deb https://get.docker.io/ubuntu docker main\
> /etc/apt/sources.list.d/docker.list"
$ sudo apt-get update
$ sudo apt-get install lxc-docker

```

### 注意

0.9 版本之前的 Docker 对 LXC（Linux 容器）有严格依赖，因此无法安装在 OpenVZ 托管的 VM 上。但自 0.9 版本以来，执行驱动程序已从 Docker 核心中解耦，这使我们可以使用众多隔离工具之一，如 LXC、OpenVZ、systemd-nspawn、libvirt-lxc、libvirt-sandbox、qemu/kvm、BSD Jails、Solaris Zones，甚至 chroot！但是，它默认使用 Docker 自己的容器化引擎的执行驱动程序，称为 l**ibcontainer**，这是一个纯 Go 库，可以直接访问内核的容器 API，而无需任何其他依赖关系。

要使用任何其他容器化引擎，比如 LXC，您可以使用-e 标志，如下所示：`$ docker -d -e lxc`。

现在我们已经安装了 Docker，我们可以全速前进了！不过有一个问题：像 APT 这样的软件仓库通常滞后于时代，经常有较旧的版本。Docker 是一个快速发展的项目，在最近的几个版本中发生了很多变化。因此，建议始终安装最新版本。

## 升级 Docker

您可以根据 APT 仓库中的更新来升级 Docker。另一种（更好的）方法是从源代码构建。此方法的教程在标题为*Inception: Docker in Docker*的部分中。建议升级到最新的稳定版本，因为更新的版本可能包含关键的安全更新和错误修复。此外，本书中的示例假定 Docker 版本大于 1.0，而 Ubuntu 的标准仓库中打包了一个更旧的版本。

## Mac OSX 和 Windows

Docker 依赖于 Linux 内核，因此我们需要在虚拟机中运行 Linux，并通过它安装和使用 Docker。Boot2Docker 是由 Docker Inc.构建的辅助应用程序，它安装了一个包含轻量级 Linux 发行版的虚拟机，专门用于运行 Docker 容器。它还带有一个客户端，提供与 Docker 相同的**应用程序** **接口** (**API**)，但与在虚拟机中运行的`docker`守护程序进行交互，允许我们从 OSX/Windows 终端运行命令。要安装 Boot2Docker，请执行以下步骤：

1.  从[`boot2docker.io/`](http://boot2docker.io/)下载适用于您操作系统的最新版本的 Boot2Docker。

1.  安装镜像如下所示：![Mac OSX 和 Windows](img/4787_01_01.jpg)

1.  运行安装程序，它将安装 VirtualBox 和 Boot2Docker 管理工具。

运行 Boot2docker。第一次运行时会要求您输入**安全** **Shell** (**SSH**)密钥密码。脚本的后续运行将连接您到虚拟机中的 shell 会话。如果需要，后续运行将初始化一个新的虚拟机并启动它。

或者，要运行 Boot2Docker，您也可以使用终端命令`boot2docker`。

```
$ boot2docker init # First run
$ boot2docker start
$ export DOCKER_HOST=tcp://$(boot2docker ip 2>/dev/null):2375

```

您只需要运行`boot2docker init`一次。它会要求您输入 SSH 密钥密码。随后，`boot2docker ssh`将使用此密码来验证 SSH 访问。

初始化 Boot2Docker 后，您随后可以使用`boot2docker start`和`boot2docker stop`命令。

`DOCKER_HOST` 是一个环境变量，设置后，指示 Docker 客户端 `docker` 守护程序的位置。端口转发规则设置为 boot2Docker VM 的端口 2375（`docker` 守护程序运行的位置）。您将需要在每个要在其中使用 Docker 的终端 shell 中设置此变量。

### 注意

Bash 允许您通过在 `` ` ``或者`$()`中包含子命令来插入命令。这些将首先被评估，结果将被替换到外部命令中。

如果您是那种喜欢四处探索的人，Boot2Docker 的默认用户是 `docker`，密码是 `tcuser`。

boot2Docker 管理工具提供了几个命令：

``` 

$ boot2docker

Usage: boot2docker [<options>] {help|init|up|ssh|save|down|poweroff|reset|restart|config|status|info|ip|delete|download|version} [<args>]

```

使用 boot2Docker 时，必须在终端会话中使 `DOCKER_HOST` 环境变量可用，以使 Docker 命令起作用。因此，如果您遇到 `Post http:///var/run/docker.sock/v1.12/containers/create: dial unix /var/run/docker.sock: no such file or directory` 错误，意味着环境变量未分配。当您打开新终端时很容易忘记设置此环境变量。对于 OSX 用户，为了简化操作，请将以下行添加到您的 `.bashrc` 或 `.bash_profile` shell 中：

```

alias setdockerhost='export DOCKER_HOST=tcp://$(boot2docker ip 2>/dev/null):2375'

```

现在，每当您打开一个新的终端或出现上述错误时，只需运行以下命令：

```

$ setdockerhost

```

![Mac OSX 和 Windows](img/4787_01_02_revised.jpg)

此图像显示了当您登录到 Boot2Docker VM 后终端屏幕的外观。

### 升级 Boot2Docker

1.  从 [`boot2docker.io/`](http://boot2docker.io/) 下载 OSX 的最新版本 Boot2Docker Installer。

1.  运行安装程序，将更新 VirtualBox 和 Boot2Docker 管理工具。

要升级现有的虚拟机，请打开终端并运行以下命令：

```

$ boot2docker stop

$ boot2docker download

```

# OpenStack

OpenStack** 是一款免费开源软件，可让您建立一个云。它主要用于部署公共和私有 **Infrastructure** **as** **a** **Service** (**IaaS**) 解决方案。它由一组相互关联的项目组成，用于云设置的不同组件，如计算调度程序、密钥链管理器、网络管理器、存储管理器、仪表板等等。

Docker 可以作为 OpenStack Nova Compute 的虚拟化驱动程序。Docker 对 OpenStack 的支持是从 **Havana** 版本开始引入的。

但是... 如何做到呢？

Nova 的 Docker 驱动程序嵌入了一个微型 HTTP 服务器，通过 **UNIX** **TCP** socket 与 Docker 引擎的内部 **Representational** **State** **Transfer** (**REST**) API 通信（稍后您将了解更多）。

Docker 有其自己的镜像仓库系统，称为 Docker-Registry，可以嵌入到 Glance（OpenStack 的镜像仓库）中以推送和拉取 Docker 镜像。Docker-Registry 可以作为 `docker` 容器或独立模式运行。

## 使用 DevStack 安装

如果您只是设置 OpenStack 并采用 DevStack 路线，那么配置设置以使用 Docker 很容易。

在运行 DevStack 路线的 `stack.sh` 脚本之前，请在 `localrc` 文件中配置 **virtual** **driver** 选项以使用 Docker：

```

VIRT_DRIVER=docker

```

然后从 `devstack` 目录运行 Docker 安装脚本。此脚本需要 `socat` 实用程序（通常由 `stack.sh` 脚本安装）。如果您尚未安装 `socat` 实用程序，请运行以下命令：

```

$ apt-get install socat

$ ./tools/docker/install_docker.sh

```

最后，从 `devstack` 目录运行 `stack.sh` 脚本：

```

$ ./stack.sh

```

## 手动为 OpenStack 安装 Docker

如果您已经设置了 OpenStack，或者 DevStack 方法不起作用，则可以手动安装 Docker：

1.  首先，根据 Docker 的一个安装程序安装 Docker。

如果您正在将 `docker` 注册表与 Glance 服务放置在一起，请运行以下命令：

```

$ sudo yum -y install docker-registry

```

在 `/etc/sysconfig/docker-registry` 文件夹中，设置 `REGISTRY_PORT` 和 `SETTINGS_FLAVOR` 注册表如下：

```

$ export SETTINGS_FLAVOR=openstack

$ export REGISTRY_PORT=5042

```

在 `docker` 注册文件中，您还需要指定 OpenStack 认证变量。以下命令完成此操作：

```

$ source /root/keystonerc_admin

$ export OS_GLANCE_URL=http://localhost:9292

```

默认情况下，`/etc/docker-registry.yml` 为 openstack 配置设置了本地或替代 `storage_path` 路径为 `/tmp`。您可能希望将路径更改为更永久的位置：

```

openstack:

storage: glance

storage_alternate: local

storage_path: /var/lib/docker-registry

```

1.  为了使 **Nova** 能够通过其本地套接字与 Docker 通信，请将 `nova` 添加到 `docker` 组，并重新启动 `compute` 服务以接收更改：

```

$ usermod -G docker nova

$ service openstack-nova-compute restart

```

1.  启动 Redis（Docker Registry 使用），如果尚未启动：

```

$ sudo service redis start

$ sudo chkconfig redis on

```

1.  最后，启动注册表：

```

$ sudo service docker-registry start

$ sudo chkconfig docker-registry on

```

## Nova 配置

Nova 需要配置为使用 `virt` Docker 驱动程序。

根据以下选项编辑 `/etc/nova/nova.conf` 配置文件：

```

[DEFAULT]

compute_driver = docker.DockerDriver

```

或者，如果您想使用您自己的 Docker-Registry，并且监听的端口不同于 5042，则可以覆盖以下选项：

```

docker_registry_default_port = 5042

```

## Glance 配置

Glance 需要配置以支持 Docker 容器格式。只需在 Glance 配置文件中将 Docker 添加到容器格式列表中即可：

```

[DEFAULT]

container_formats = ami,ari,aki,bare,ovf,docker

```

### 提示

为了不破坏现有的 glance 安装，请保留默认格式。

## Docker-OpenStack 流程

一旦您配置了 Nova 使用 `docker` 驱动程序，流程与任何其他驱动程序中的流程相同：

```

$ docker search hipache

Found 3 results matching your query ("hipache")

NAME                             DESCRIPTION

samalba/hipache                  https://github.com/dotcloud/hipache

```

然后，使用 Docker-Registry 位置标记图像并推送它：

```

$ docker pull samalba/hipache

$ docker tag samalba/hipache localhost:5042/hipache

$ docker push localhost:5042/hipache

```

推送引用了一个仓库：

```

[localhost:5042/hipache] (len: 1)

Sending image list

Pushing repository localhost:5042/hipache (1 tags)

Push 100% complete

```

在这种情况下，Docker-Registry（在一个端口映射为 5042 的 Docker 容器中运行）将图像推送到 Glance。从那里，Nova 可以访问它们，并且您可以使用 Glance **Command**-**Line** **Interface**（**CLI**）验证图像：

```

$ glance image-list

```

### 注意

只有具有 Docker 容器格式的图像才能启动。图像基本上包含容器文件系统的 tarball。

您可以使用 `nova` `boot` 命令引导实例：

```

$ nova boot --image "docker-busybox:latest" --flavor m1.tiny test

```

### 提示

使用的命令将是在图像中配置的命令。每个容器图像都可以为运行配置一个命令。驱动程序不会覆盖此命令。

一旦实例引导完成，它将在 `nova` `list` 中列出：

```

$ nova list

```

您还可以在 Docker 中查看相应的容器：

```

$ docker ps

```

# Inception：构建 Docker 中的 Docker

虽然从标准仓库安装更容易，但它们通常包含较旧的版本，这意味着您可能会错过关键的更新或功能。保持更新的最佳方法是定期从公共 `GitHub` 仓库获取最新版本。传统上，从源代码构建软件是很痛苦的，只有实际从事项目工作的人才会这样做。但 Docker 不是这样。从 Docker 0.6 开始，就可以在 Docker 中构建 Docker。这意味着升级 Docker 就像在 Docker 中构建新版本并替换二进制文件一样简单。让我们看看如何做到这一点。

## 依赖关系

您需要在 64 位 Linux 机器（虚拟机或裸机）上安装以下工具才能构建 Docker：

+   **Git

+   **Make

Git** 是一个免费且开放源代码的分布式版本控制系统，旨在以速度和效率处理从小型到非常大型的项目。它在此用于克隆 Docker 公共源代码仓库。有关更多详细信息，请查看 [git-scm.org](http://git-scm.org)。

`make` 实用程序是用于管理和维护计算机程序的软件工程工具。**Make** 在程序由许多组件文件组成时提供了最大的帮助。在这里，使用一个 `Makefile` 文件以一种可重复和一致的方式启动 Docker 容器。

## 从源代码构建 Docker

要在 Docker 中构建 Docker，我们首先会获取源代码，然后运行几个 `make` 命令，最终创建一个 `docker` 二进制文件，该文件将替换 Docker 安装路径中的当前二进制文件。

在终端中运行以下命令：

```

$ git clone https://git@github.com/dotcloud/docker

```

此命令将官方 Docker 源代码仓库从 `Github` 仓库克隆到名为 `docker` 的目录中：

```

$ cd docker

$ sudo make build

```

这将准备开发环境并安装创建二进制文件所需的所有依赖项。在第一次运行时可能需要一些时间，所以你可以去喝杯咖啡。

### 提示

如果遇到任何难以调试的错误，您可以随时转到 `#docker` 上的 freenode IRC。开发人员和 Docker 社区都非常乐意帮助。

现在我们已经准备好编译二进制文件了：

```

$ sudo make binary

```

这将编译一个二进制文件，并将其放置在 `./bundles/<version>-dev/binary/` 目录中。然后！您现在有一个准备就绪的 Docker 新版本。

但在替换现有二进制文件之前，请运行测试：

```

$ sudo make test

```

如果测试通过，则可以安全地用您刚刚编译的二进制文件替换当前的二进制文件。停止 `docker` 服务，创建现有二进制文件的备份，然后将新鲜出炉的二进制文件复制到其位置：

```

$ sudo service docker stop

$ alias wd='which docker'

$ sudo cp $(wd) $(wd)_

$ sudo cp $(pwd)/bundles/<version>-dev/binary/docker-<version>-dev $(wd)

$ sudo service docker start

```

恭喜！您现在拥有最新版本的 Docker 运行。

### 提示

OSX 和 Windows 用户可以按照 SSH 进入 boot2Docker VM 的相同步骤进行操作。

# 验证安装

要验证您的安装是否成功，请在终端控制台中运行以下命令：

```

$ docker run -i -t ubuntu echo Hello World!

```

`docker` `run` 命令使用`ubuntu`基础镜像启动容器。由于这是您首次启动`ubuntu`容器，容器的输出将类似于这样：

```

Unable to find image 'ubuntu' locally
Pulling repository ubuntu
e54ca5efa2e9: Download complete
511136ea3c5a: Download complete
d7ac5e4f1812: Download complete
2f4b4d6a4a06: Download complete
83ff768040a0: Download complete
6c37f792ddac: Download complete

Hello World!

```

当您发出`docker` `run` `ubuntu`命令时，Docker 将在本地查找`ubuntu`镜像，如果找不到，它将从公共`docker`注册表下载`ubuntu`镜像。您还将看到它显示**正在拉取依赖层**。

这意味着它正在下载文件系统层。默认情况下，Docker 使用 AUFS，一种分层的写时复制文件系统，这意味着容器镜像的文件系统是多个只读文件系统层的结合体。而这些层是在运行的容器之间共享的。如果你启动了一个会写入此文件系统的操作，它将创建一个新的层，该层将是底层层和新数据的差异。共享常见层意味着只有第一个容器会占用大量内存，而后续容器将占用微不足道的内存，因为它们将共享只读层。这意味着即使在相对性能较低的笔记本电脑上，你也可以运行数百个容器。

![验证安装](img/4787_01_03.jpg)

一旦镜像完全下载完成，它将启动容器并在您的控制台中回显`Hello``World!`。这是 Docker 容器的另一个显著特点。每个容器都与一个命令关联，并且应该运行该命令。请记住，Docker 容器不像虚拟机那样虚拟化整个操作系统。每个`docker`容器只接受一个单一命令，并在一个独立环境中运行它。

# 有用的提示

以下是两个有用的提示，以后可能会为您节省大量麻烦。第一个显示了如何为 Docker 客户端提供非根访问权限，第二个显示了如何配置 Ubuntu 防火墙规则以启用转发网络流量。

### 注意

如果您使用的是 Boot2Docker，则无需遵循这些步骤。

## 给予非根访问权限

创建一个名为`docker`的组，并将您的用户添加到该组，以避免每个`docker`命令都需要添加`sudo`前缀。默认情况下，您需要使用`sudo`前缀运行`docker`命令的原因是`docker`守护程序需要以`root`权限运行，但 docker 客户端（您运行的命令）不需要。因此，通过创建一个`docker`组，您可以在不使用`sudo`前缀的情况下运行所有客户端命令，而守护程序则以`root`权限运行：

```

$ sudo groupadd docker # Adds the docker group
$ sudo gpasswd -a $(whoami) docker # Adds the current user to the group
$ sudo service docker restart

```

你可能需要退出并重新登录以使更改生效。

## UFW 设置

Docker 使用桥接来管理容器中的网络。**简化防火墙**（**UFW**）是 Ubuntu 中的默认防火墙工具。它会拒绝所有转发流量。您需要像这样启用转发：

```

$ sudo vim /etc/default/ufw
# Change:
# DEFAULT_FORWARD_POLICY="DROP"
# to
DEFAULT_FORWARD_POLICY="ACCEPT"

```

运行以下命令重新加载防火墙：

```

$ sudo ufw reload

```

或者，如果你想要能够从其他主机访问你的容器，那么你应该在 Docker 端口（`default` `2375`）上启用入站连接：


```

$ sudo ufw allow 2375/tcp

```

### 提示

下载示例代码

您可以从您在[`www.packtpub.com`](http://www.packtpub.com)的帐户中下载示例代码文件，用于您购买的所有 Packt Publishing 图书。如果您在其他地方购买了这本书，您可以访问[`www.packtpub.com/support`](http://www.packtpub.com/support)并注册，以便文件直接通过电子邮件发送给您

# 总结

我希望这个介绍性的章节让你着迷于 Docker。接下来的章节将带你进入 Docker 的世界，并试图用它的神奇之处来迷住你。

在本章中，您学习了一些关于 Docker 的历史和基础知识，以及它的工作原理。我们看到了它与虚拟机的不同之处以及优势。

然后，我们继续在我们的开发环境中安装 Docker，无论是 Ubuntu、Mac 还是 Windows。然后我们看到如何用 Docker 替换 OpenStack 的 hypervisor。后来，我们在 Docker 中构建了 Docker 源代码！说到吃自己的狗粮！

最后，我们下载了我们的第一个镜像并运行了我们的第一个容器。现在你可以拍拍自己的背，继续下一章，在那里我们将深入介绍主要的 Docker 命令，并看看我们如何创建自己的镜像。

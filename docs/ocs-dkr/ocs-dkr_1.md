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

[PRE0]

就是这样！您现在已经在系统上安装了 Docker。但是，由于命令已更名为`docker.io`，您将不得不使用`docker.io`而不是`docker`来运行所有 Docker 命令。

### 注意

该软件包的名称为`docker.io`，因为它与另一个名为`docker`的 KDE3/GNOME2 软件包冲突。如果您更愿意以`docker`运行命令，可以创建一个符号链接到`/usr/local/bin`目录。第二个命令将自动完成规则添加到 bash：

[PRE1]

### 在 Ubuntu Precise 12.04 LTS 中安装 Docker

Ubuntu 12.04 带有较旧的内核（3.2），与 Docker 的一些依赖项不兼容。因此，我们需要升级它：

[PRE2]

我们刚刚安装的内核内置了 AUFS，这也是 Docker 的要求。

现在让我们结束安装：

[PRE3]

这是一个用于简单安装的`curl`脚本。查看此脚本的各个部分将帮助我们更好地理解该过程：

1.  首先，脚本检查我们的**高级** **软件包** **工具**（**APT**）系统是否能处理`https` URL，并在无法处理时安装`apt-transport-https`：

[PRE4]

1.  然后它将 Docker 存储库添加到我们的本地密钥链中：

[PRE5]

### 提示

您可能会收到一个警告，表示软件包不受信任。回答`yes`以继续安装。

1.  最后，它将 Docker 存储库添加到 APT 源列表中，并更新并安装`lxc-docker`软件包：

[PRE6]

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

[PRE7]

您只需要运行`boot2docker init`一次。它会要求您输入 SSH 密钥密码。随后，`boot2docker ssh`将使用此密码来验证 SSH 访问。

初始化 Boot2Docker 后，您随后可以使用`boot2docker start`和`boot2docker stop`命令。

`DOCKER_HOST` 是一个环境变量，设置后，指示 Docker 客户端 `docker` 守护程序的位置。端口转发规则设置为 boot2Docker VM 的端口 2375（`docker` 守护程序运行的位置）。您将需要在每个要在其中使用 Docker 的终端 shell 中设置此变量。

### 注意

Bash 允许您通过在 [PRE8] 中包含子命令来插入命令

**$ boot2docker**

Usage: boot2docker [<options>] {help|init|up|ssh|save|down|poweroff|reset|restart|config|status|info|ip|delete|download|version} [<args>]

[PRE9]

**alias setdockerhost='export DOCKER_HOST=tcp://$(boot2docker ip 2>/dev/null):2375'**

[PRE10]

**$ setdockerhost**

[PRE11]

**$ boot2docker stop**

**$ boot2docker download**

[PRE12]

**VIRT_DRIVER=docker**

[PRE13]

**$ apt-get install socat**

**$ ./tools/docker/install_docker.sh**

[PRE14]

**$ ./stack.sh**

[PRE15]

**$ sudo yum -y install docker-registry**

[PRE16]

**$ export SETTINGS_FLAVOR=openstack**

**$ export REGISTRY_PORT=5042**

[PRE17]

**$ source /root/keystonerc_admin**

**$ export OS_GLANCE_URL=http://localhost:9292**

[PRE18]

**openstack:**

**storage: glance**

**storage_alternate: local**

**storage_path: /var/lib/docker-registry**

[PRE19]

**$ usermod -G docker nova**

**$ service openstack-nova-compute restart**

[PRE20]

**$ sudo service redis start**

**$ sudo chkconfig redis on**

[PRE21]

**$ sudo service docker-registry start**

**$ sudo chkconfig docker-registry on**

[PRE22]

**[DEFAULT]**

**compute_driver = docker.DockerDriver**

[PRE23]

**docker_registry_default_port = 5042**

[PRE24]

**[DEFAULT]**

**container_formats = ami,ari,aki,bare,ovf,docker**

[PRE25]

**$ docker search hipache**

**Found 3 results matching your query ("hipache")**

**NAME                             DESCRIPTION**

**samalba/hipache                  https://github.com/dotcloud/hipache**

[PRE26]

**$ docker pull samalba/hipache**

**$ docker tag samalba/hipache localhost:5042/hipache**

**$ docker push localhost:5042/hipache**

[PRE27]

**[localhost:5042/hipache] (len: 1)**

**Sending image list**

**Pushing repository localhost:5042/hipache (1 tags)**

**Push 100% complete**

[PRE28]

**$ glance image-list**

[PRE29]

**$ nova boot --image "docker-busybox:latest" --flavor m1.tiny test**

[PRE30]

**$ nova list**

[PRE31]

**$ docker ps**

[PRE32]

**$ git clone https://git@github.com/dotcloud/docker**

[PRE33]

**$ cd docker**

**$ sudo make build**

[PRE34]

**$ sudo make binary**

[PRE35]

**$ sudo make test**

[PRE36]

**$ sudo service docker stop**

**$ alias wd='which docker'**

**$ sudo cp $(wd) $(wd)_**

**$ sudo cp $(pwd)/bundles/<version>-dev/binary/docker-<version>-dev $(wd)**

**$ sudo service docker start**

[PRE37]

**$ docker run -i -t ubuntu echo Hello World!**

[PRE38]

无法找到本地镜像'ubuntu'

拉取存储库 ubuntu

e54ca5efa2e9：下载完成

511136ea3c5a：下载完成

d7ac5e4f1812：下载完成

2f4b4d6a4a06：下载完成

83ff768040a0：下载完成

6c37f792ddac：下载完成

你好，世界！

[PRE39]

$ sudo groupadd docker # 添加 docker 组

$ sudo gpasswd -a $(whoami) docker # 将当前用户添加到组中

$ sudo service docker restart

[PRE40]

$ sudo vim /etc/default/ufw

# 更改：

# DEFAULT_FORWARD_POLICY="DROP"

# 到

DEFAULT_FORWARD_POLICY="ACCEPT"

[PRE41]

$ sudo ufw reload

[PRE42]

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

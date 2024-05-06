# 第一章。打开 Docker

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
**$ sudo apt-get update**
**$ sudo apt-get -y install docker.io**

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
**$ sudo apt-get update**
**$ sudo apt-get -y install linux-image-generic-lts-raring linux-headers-generic-lts-raring**
**$ sudo reboot**

```

我们刚刚安装的内核内置了 AUFS，这也是 Docker 的要求。

现在让我们结束安装：

```
**$ curl -s https://get.docker.io/ubuntu/ | sudo sh**

```

这是一个用于简单安装的`curl`脚本。查看此脚本的各个部分将帮助我们更好地理解该过程：

1.  首先，脚本检查我们的**高级** **软件包** **工具**（**APT**）系统是否能处理`https` URL，并在无法处理时安装`apt-transport-https`：

```
**# Check that HTTPS transport is available to APT**
**if [ ! -e /usr/lib/apt/methods/https ]; then  apt-get update  apt-get install -y apt-transport-https**
**fi**

```

1.  然后它将 Docker 存储库添加到我们的本地密钥链中：

```
**$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9**

```

### 提示

您可能会收到一个警告，表示软件包不受信任。回答`yes`以继续安装。

1.  最后，它将 Docker 存储库添加到 APT 源列表中，并更新并安装`lxc-docker`软件包：

```
**$ sudo sh -c "echo deb https://get.docker.io/ubuntu docker main\**
**> /etc/apt/sources.list.d/docker.list"**
**$ sudo apt-get update**
**$ sudo apt-get install lxc-docker**

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
**$ boot2docker init # First run**
**$ boot2docker start**
**$ export DOCKER_HOST=tcp://$(boot2docker ip 2>/dev/null):2375**

```

您只需要运行`boot2docker init`一次。它会要求您输入 SSH 密钥密码。随后，`boot2docker ssh`将使用此密码来验证 SSH 访问。

初始化 Boot2Docker 后，您随后可以使用`boot2docker start`和`boot2docker stop`命令。

`DOCKER_HOST` 是一个环境变量，设置后，指示 Docker 客户端 `docker` 守护程序的位置。端口转发规则设置为 boot2Docker VM 的端口 2375（`docker` 守护程序运行的位置）。您将需要在每个要在其中使用 Docker 的终端 shell 中设置此变量。

### 注意

Bash 允许您通过在 ```` or `$()`. These will be evaluated first and the result will be substituted in the outer commands.

If you are the kind that loves to poke around, the Boot2Docker default user is `docker` and the password is `tcuser`.

The boot2Docker management tool provides several commands:

``` 中包含子命令来插入命令

**$ boot2docker**

Usage: boot2docker [<options>] {help|init|up|ssh|save|down|poweroff|reset|restart|config|status|info|ip|delete|download|version} [<args>]

```

When using boot2Docker, the `DOCKER_HOST` environment variable has to be available in the terminal session for Docker commands to work. So, if you are getting the `Post http:///var/run/docker.sock/v1.12/containers/create: dial unix /var/run/docker.sock: no such file or directory` error, it means that the environment variable is not assigned. It is easy to forget to set this environment variable when you open a new terminal. For OSX users, to make things easy, add the following line to your `.bashrc` or `.bash_profile` shells:

```

**alias setdockerhost='export DOCKER_HOST=tcp://$(boot2docker ip 2>/dev/null):2375'**

```

Now, whenever you open a new terminal or get the above error, just run the following command:

```

**$ setdockerhost**

```

![Mac OSX and Windows](img/4787_01_02_revised.jpg)

This image shows how the terminal screen will look like when you have logged into the Boot2Docker VM.

### Upgrading Boot2Docker

1.  Download the latest release of the Boot2Docker Installer for OSX from [`boot2docker.io/`](http://boot2docker.io/).
2.  Run the installer, which will update VirtualBox and the Boot2Docker management tool.

To upgrade your existing virtual machine, open a terminal and run the following commands:

```

**$ boot2docker stop**

**$ boot2docker download**

```

# OpenStack

**OpenStack** is a piece of free and open source software that allows you to set up a cloud. It is primarily used to deploy public and private **Infrastructure** **as** **a** **Service** (**IaaS**) solutions. It consists of a pool of interrelated projects for the different components of a cloud setup such as compute schedulers, keychain managers, network managers, storage managers, dashboards, and so on.

Docker can act as a hypervisor driver for OpenStack Nova Compute. Docker support for OpenStack was introduced with the **Havana** release.

But... how?

Nova's Docker driver embeds a tiny HTTP server that talks to the Docker Engine's internal **Representational** **State** **Transfer** (**REST**) API (you will learn more on this later) through a **UNIX** **TCP** socket.

Docker has its own image repository system called Docker-Registry, which can be embedded into Glance (OpenStack's image repository) to push and pull Docker images. Docker-Registry can be run either as a `docker` container or in a standalone mode.

## Installation with DevStack

If you are just setting up OpenStack and taking up the DevStack route, configuring the setup to use Docker is pretty easy.

Before running the DevStack route's `stack.sh` script, configure the **virtual** **driver** option in the `localrc` file to use Docker:

```

**VIRT_DRIVER=docker**

```

Then run the Docker installation script from the `devstack` directory. The `socat` utility is needed for this script (usually installed by the `stack.sh` script). If you don't have the `socat` utility installed, run the following:

```

**$ apt-get install socat**

**$ ./tools/docker/install_docker.sh**

```

Finally, run the `stack.sh` script from the `devstack` directory:

```

**$ ./stack.sh**

```

## Installing Docker for OpenStack manually

Docker can also be installed manually if you already have OpenStack set up or in case the DevStack method doesn't work out:

1.  Firstly, install Docker according to one of the Docker installation procedures.

If you are co-locating the `docker` registry alongside the Glance service, run the following command:

```

**$ sudo yum -y install docker-registry**

```

In the `/etc/sysconfig/docker-registry` folder, set the `REGISTRY_PORT` and `SETTINGS_FLAVOR` registries as follows:

```

**$ export SETTINGS_FLAVOR=openstack**

**$ export REGISTRY_PORT=5042**

```

In the `docker` registry file, you will also need to specify the OpenStack authentication variables. The following commands accomplish this:

```

**$ source /root/keystonerc_admin**

**$ export OS_GLANCE_URL=http://localhost:9292**

```

By default, `/etc/docker-registry.yml` sets the local or alternate `storage_path` path for the openstack configuration under `/tmp`. You may want to alter the path to a more permanent location:

```

**openstack:**

**storage: glance**

**storage_alternate: local**

**storage_path: /var/lib/docker-registry**

```

2.  In order for **Nova** to communicate with Docker over its local socket, add `nova` to the `docker` group and restart the `compute` service to pick up the change:

```

**$ usermod -G docker nova**

**$ service openstack-nova-compute restart**

```

3.  Start Redis (used by the Docker Registry), if it wasn't started already:

```

**$ sudo service redis start**

**$ sudo chkconfig redis on**

```

4.  Finally, start the registry:

```

**$ sudo service docker-registry start**

**$ sudo chkconfig docker-registry on**

```

## Nova configuration

Nova needs to be configured to use the `virt` Docker driver.

Edit the `/etc/nova/nova.conf` configuration file according to the following options:

```

**[DEFAULT]**

**compute_driver = docker.DockerDriver**

```

Alternatively, if you want to use your own Docker-Registry, which listens on a port different than 5042, you can override the following option:

```

**docker_registry_default_port = 5042**

```

## Glance configuration

Glance needs to be configured to support the Docker container format. Just add Docker to the list of container formats in the Glance configuration file:

```

**[DEFAULT]**

**container_formats = ami,ari,aki,bare,ovf,docker**

```

### Tip

Leave the default formats in order to not break an existing glance installation.

## Docker-OpenStack flow

Once you configured Nova to use the `docker` driver, the flow is the same as that in any other driver:

```

**$ docker search hipache**

**Found 3 results matching your query ("hipache")**

**NAME                             DESCRIPTION**

**samalba/hipache                  https://github.com/dotcloud/hipache**

```

Then tag the image with the Docker-Registry location and push it:

```

**$ docker pull samalba/hipache**

**$ docker tag samalba/hipache localhost:5042/hipache**

**$ docker push localhost:5042/hipache**

```

The push refers to a repository:

```

**[localhost:5042/hipache] (len: 1)**

**Sending image list**

**Pushing repository localhost:5042/hipache (1 tags)**

**Push 100% complete**

```

In this case, the Docker-Registry (running in a docker container with a port mapped on 5042) will push the images to Glance. From there, Nova can reach them and you can verify the images with the Glance **Command**-**Line** **Interface** (**CLI**):

```

**$ glance image-list**

```

### Note

Only images with a docker container format will be bootable. The image basically contains a tarball of the container filesystem.

You can boot instances with the `nova` `boot` command:

```

**$ nova boot --image "docker-busybox:latest" --flavor m1.tiny test**

```

### Tip

The command used will be the one configured in the image. Each container image can have a command configured for the run. The driver does not override this command.

Once the instance is booted, it will be listed in `nova` `list`:

```

**$ nova list**

```

You can also see the corresponding container in Docker:

```

**$ docker ps**

```

# Inception: Build Docker in Docker

Though installing from standard repositories is easier, they usually contain older versions, which means that you might miss critical updates or features. The best way to remain updated is to regularly get the latest version from the public `GitHub` repository. Traditionally, building software from a source has been painful and done only by people who actually work on the project. This is not so with Docker. From Docker 0.6, it has been possible to build Docker in Docker. This means that upgrading Docker is as simple as building a new version in Docker itself and replacing the binary. Let's see how this is done.

## Dependencies

You need to have the following tools installed in a 64-bit Linux machine (VM or bare-metal) to build Docker:

*   **Git**
*   **Make**

**Git** is a free and open source distributed version control system designed to handle everything from small to very large projects with speed and efficiency. It is used here to clone the Docker public source code repository. Check out [git-scm.org](http://git-scm.org) for more details.

The `make` utility is a software engineering tool used to manage and maintain computer programs. **Make** provides most help when the program consists of many component files. A `Makefile` file is used here to kick off the Docker containers in a repeatable and consistent way.

## Building Docker from source

To build Docker in Docker, we will first fetch the source code and then run a few `make` commands that will, in the end, create a `docker` binary, which will replace the current binary in the Docker installation path.

Run the following command in your terminal:

```

**$ git clone https://git@github.com/dotcloud/docker**

```

This command clones the official Docker source code repository from the `Github` repository into a directory named `docker`:

```

**$ cd docker**

**$ sudo make build**

```

This will prepare the development environment and install all the dependencies required to create the binary. This might take some time on the first run, so you can go and have a cup of coffee.

### Tip

If you encounter any errors that you find difficult to debug, you can always go to `#docker` on freenode IRC. The developers and the Docker community are very helpful.

Now we are ready to compile that binary:

```

**$ sudo make binary**

```

This will compile a binary and place it in the `./bundles/<version>-dev/binary/` directory. And voila! You have a fresh version of Docker ready.

Before replacing your existing binary though, run the tests:

```

**$ sudo make test**

```

If the tests pass, then it is safe to replace your current binary with the one you've just compiled. Stop the `docker` service, create a backup of the existing binary, and then copy the freshly baked binary in its place:

```

**$ sudo service docker stop**

**$ alias wd='which docker'**

**$ sudo cp $(wd) $(wd)_**

**$ sudo cp $(pwd)/bundles/<version>-dev/binary/docker-<version>-dev $(wd)**

**$ sudo service docker start**

```

Congratulations! You now have the up-to-date version of Docker running.

### Tip

OSX and Windows users can follow the same procedures as SSH in the boot2Docker VM.

# Verifying Installation

To verify that your installation is successful, run the following command in your terminal console:

```

**$ docker run -i -t ubuntu echo Hello World!**

```

The `docker` `run` command starts a container with the `ubuntu` base image. Since this is the first time you are starting an `ubuntu` container, the output of the container will be something like this:

```

无法找到本地镜像'ubuntu'

拉取存储库 ubuntu

e54ca5efa2e9：下载完成

511136ea3c5a：下载完成

d7ac5e4f1812：下载完成

2f4b4d6a4a06：下载完成

83ff768040a0：下载完成

6c37f792ddac：下载完成

你好，世界！

```

When you issue the `docker` `run` `ubuntu` command, Docker looks for the `ubuntu` image locally, and it's not found, it will download the `ubuntu` image from the public `docker` registry. You will also see it say **Pulling** **dependent layers**.

This means that it is downloading filesystem layers. By default, Docker uses AUFS, a layered copy-on-write filesystem, which means that the container image's filesystem is a culmination of multiple read-only filesystem layers. And these layers are shared between running containers. If you initiate an action that will write to this filesystem, it will create a new layer that will be the difference of the underlying layers and the new data. Sharing of common layers means that only the first container will take up a considerable amount of memory and subsequent containers will take up an insignificant amount of memory as they will be sharing the read-only layers. This means that you can run hundreds of containers even on a relatively low-powered laptop.

![Verifying Installation](img/4787_01_03.jpg)

Once the image has been completely downloaded, it will start the container and echo `Hello` `World!` in your console. This is another salient feature of the Docker containers. Every container is associated with a command and it should run that command. Remember that the Docker containers are unlike VMs in that they do not virtualize the entire operating system. Each `docker` container accepts only a single command and runs it in a sandboxed process that lives in an isolated environment.

# Useful tips

The following are two useful tips that might save you a lot of trouble later on. The first shows how to give the docker client non-root access, and the second shows how to configure the Ubuntu firewall rules to enable forwarding network traffic.

### Note

You do not need to follow these if you are using Boot2Docker.

## Giving non-root access

Create a group called `docker` and add your user to that group to avoid having to add the `sudo` prefix to every `docker` command. The reason you need to run a `docker` command with the `sudo` prefix by default is that the `docker` daemon needs to run with `root` privileges, but the docker client (the commands you run) doesn't. So, by creating a `docker` group, you can run all the client commands without using the `sudo` prefix, whereas the daemon runs with the `root` privileges:

```

$ sudo groupadd docker # 添加 docker 组

$ sudo gpasswd -a $(whoami) docker # 将当前用户添加到组中

$ sudo service docker restart

```

You might need to log out and log in again for the changes to take effect.

## UFW settings

Docker uses a bridge to manage network in the container. **Uncomplicated** **Firewall** (**UFW**) is the default firewall tool in Ubuntu. It drops all forwarding traffic. You will need to enable forwarding like this:

```

$ sudo vim /etc/default/ufw

# 更改：

# DEFAULT_FORWARD_POLICY="DROP"

# 到

DEFAULT_FORWARD_POLICY="ACCEPT"

```

Reload the firewall by running the following command:

```

$ sudo ufw reload

```

Alternatively, if you want to be able to reach your containers from other hosts, then you should enable incoming connections on the docker port (`default` `2375`):

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

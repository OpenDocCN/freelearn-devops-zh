# 第二章：使用容器的 DevOps

我们已经熟悉了许多 DevOps 工具，这些工具帮助我们自动化任务并在应用程序交付的不同阶段管理配置，但随着应用程序变得更加微小和多样化，仍然存在挑战。在本章中，我们将向我们的工具箱中添加另一把瑞士军刀，即容器。这样做，我们将寻求获得以下技能：

+   容器概念和基础知识

+   运行 Docker 应用程序

+   使用`Dockerfile`构建 Docker 应用程序

+   使用 Docker Compose 编排多个容器

# 理解容器

容器的关键特性是隔离。在本节中，我们将详细阐述容器是如何实现隔离的，以及为什么在软件开发生命周期中这一点很重要，以帮助建立对这个强大工具的正确理解。

# 资源隔离

当应用程序启动时，它会消耗 CPU 时间，占用内存空间，链接到其依赖库，并可能写入磁盘，传输数据包，并访问其他设备。它使用的一切都是资源，并且被同一主机上的所有程序共享。容器的理念是将资源和程序隔离到单独的盒子中。

您可能听说过诸如 para-virtualization、虚拟机（VMs）、BSD jails 和 Solaris 容器等术语，它们也可以隔离主机的资源。然而，由于它们的设计不同，它们在根本上是不同的，但提供了类似的隔离概念。例如，虚拟机的实现是为了使用 hypervisor 对硬件层进行虚拟化。如果您想在虚拟机上运行应用程序，您必须首先安装完整的操作系统。换句话说，在同一 hypervisor 上的客户操作系统之间的资源是隔离的。相比之下，容器是建立在 Linux 原语之上的，这意味着它只能在具有这些功能的操作系统中运行。BSD jails 和 Solaris 容器在其他操作系统上也以类似的方式工作。容器和虚拟机的隔离关系如下图所示。容器在操作系统层隔离应用程序，而基于 VM 的分离是通过操作系统实现的。

![](img/00026.jpeg)

# Linux 容器概念

容器由几个构建模块组成，其中最重要的两个是**命名空间**和**控制组**（**cgroups**）。它们都是 Linux 内核的特性。命名空间提供了对某些类型的系统资源的逻辑分区，例如挂载点（`mnt`）、进程 ID（`PID`）、网络（net）等。为了解释隔离的概念，让我们看一些关于 `pid` 命名空间的简单示例。以下示例均来自 Ubuntu 16.04.2 和 util-linux 2.27.1。

当我们输入 `ps axf` 时，会看到一个长长的正在运行的进程列表：

```
$ ps axf
 PID TTY      STAT   TIME COMMAND
    2 ?        S      0:00 [kthreadd]
    3 ?        S      0:42  \_ [ksoftirqd/0]
    5 ?        S<     0:00  \_ [kworker/0:0H]
    7 ?        S      8:14  \_ [rcu_sched]
    8 ?        S      0:00  \_ [rcu_bh]
```

`ps` 是一个报告系统上当前进程的实用程序。`ps axf` 是列出所有进程的命令。

现在让我们使用 `unshare` 进入一个新的 `pid` 命名空间，它能够逐部分将进程资源与新的命名空间分离，并再次检查进程：

```
$ sudo unshare --fork --pid --mount-proc=/proc /bin/sh
$ ps axf
 PID TTY      STAT   TIME COMMAND
    1 pts/0    S      0:00 /bin/sh
    2 pts/0    R+     0:00 ps axf
```

您会发现新命名空间中 shell 进程的 `pid` 变为 `1`，而所有其他进程都消失了。也就是说，您已经创建了一个 `pid` 容器。让我们切换到命名空间外的另一个会话，并再次列出进程：

```
$ ps axf // from another terminal
 PID TTY   COMMAND
  ...
  25744 pts/0 \_ unshare --fork --pid --mount-proc=/proc    
  /bin/sh
 25745 pts/0    \_ /bin/sh
  3305  ?     /sbin/rpcbind -f -w
  6894  ?     /usr/sbin/ntpd -p /var/run/ntpd.pid -g -u  
  113:116
    ...
```

在新的命名空间中，您仍然可以看到其他进程和您的 shell 进程。

通过 `pid` 命名空间隔离，不同命名空间中的进程无法看到彼此。然而，如果一个进程占用了大量系统资源，比如内存，它可能会导致系统内存耗尽并变得不稳定。换句话说，一个被隔离的进程仍然可能干扰其他进程，甚至导致整个系统崩溃，如果我们不对其施加资源使用限制。

以下图表说明了 `PID` 命名空间以及一个**内存不足**（**OOM**）事件如何影响子命名空间外的其他进程。气泡代表系统中的进程，数字代表它们的 PID。子命名空间中的进程有自己的 PID。最初，系统中仍然有可用的空闲内存。后来，子命名空间中的进程耗尽了系统中的所有内存。内核随后启动了 OOM killer 来释放内存，受害者可能是子命名空间外的进程：

![](img/00027.jpeg)

鉴于此，`cgroups` 在这里被用来限制资源使用。与命名空间一样，它可以对不同类型的系统资源设置约束。让我们从我们的 `pid` 命名空间继续，用 `yes > /dev/null` 来压力测试 CPU，并用 `top` 进行监控：

```
$ yes > /dev/null & top
$ PID USER  PR  NI    VIRT   RES   SHR S  %CPU %MEM    
TIME+ COMMAND
 3 root  20   0    6012   656   584 R 100.0  0.0  
  0:15.15 yes
 1 root  20   0    4508   708   632 S   0.0  0.0                   
  0:00.00 sh
 4 root  20   0   40388  3664  3204 R   0.0  0.1  
  0:00.00 top
```

我们的 CPU 负载达到了预期的 100%。现在让我们使用 CPU cgroup 来限制它。Cgroups 组织为`/sys/fs/cgroup/`下的目录（首先切换到主机会话）：

```
$ ls /sys/fs/cgroup
blkio        cpuset   memory            perf_event
cpu          devices  net_cls           pids
cpuacct      freezer  net_cls,net_prio  systemd
cpu,cpuacct  hugetlb  net_prio 
```

每个目录代表它们控制的资源。创建一个 cgroup 并控制进程非常容易：只需在资源类型下创建一个任意名称的目录，并将您想要控制的进程 ID 附加到`tasks`中。这里我们想要限制`yes`进程的 CPU 使用率，所以在`cpu`下创建一个新目录，并找出`yes`进程的 PID：

```
$ ps x | grep yes
11809 pts/2    R     12:37 yes

$ mkdir /sys/fs/cgroup/cpu/box && \
 echo 11809 > /sys/fs/cgroup/cpu/box/tasks
```

我们刚刚将`yes`添加到新创建的 CPU 组`box`中，但策略仍未设置，进程仍在没有限制地运行。通过将所需的数字写入相应的文件来设置限制，并再次检查 CPU 使用情况：

```
$ echo 50000 > /sys/fs/cgroup/cpu/box/cpu.cfs_quota_us
$ PID USER  PR  NI    VIRT   RES   SHR S  %CPU %MEM    
 TIME+ COMMAND
    3 root  20   0    6012   656   584 R  50.2  0.0     
    0:32.05 yes
    1 root  20   0    4508  1700  1608 S   0.0  0.0  
    0:00.00 sh
    4 root  20   0   40388  3664  3204 R   0.0  0.1  
    0:00.00 top
```

CPU 使用率显着降低，这意味着我们的 CPU 限制起作用了。

这两个例子阐明了 Linux 容器如何隔离系统资源。通过在应用程序中增加更多的限制，我们绝对可以构建一个完全隔离的盒子，包括文件系统和网络，而无需在其中封装操作系统。

# 容器化交付

为了部署应用程序，通常会使用配置管理工具。它确实可以很好地处理模块化和基于代码的配置设计，直到应用程序堆栈变得复杂和多样化。维护一个庞大的配置清单基础是复杂的。当我们想要更改一个软件包时，我们将不得不处理系统和应用程序软件包之间纠缠不清和脆弱的依赖关系。经常会出现在升级一个无关的软件包后一些应用程序意外中断的情况。此外，升级配置管理工具本身也是一项具有挑战性的任务。

为了克服这样的困境，引入了使用预先烘焙的虚拟机镜像进行不可变部署。也就是说，每当系统或应用程序包有任何更新时，我们将根据更改构建一个完整的虚拟机镜像，并相应地部署它。这解决了一定程度的软件包问题，因为我们现在能够为无法共享相同环境的应用程序定制运行时。然而，使用虚拟机镜像进行不可变部署是昂贵的。从另一个角度来看，为了隔离应用程序而不是资源不足而配置虚拟机会导致资源利用效率低下，更不用说启动、分发和运行臃肿的虚拟机镜像的开销了。如果我们想通过共享虚拟机来消除这种低效，很快就会意识到我们将遇到进一步的麻烦，即资源管理。

容器在这里是一个完美适应部署需求的拼图块。容器的清单可以在版本控制系统中进行管理，并构建成一个 blob 图像；毫无疑问，该图像也可以被不可变地部署。这使开发人员可以抽象出实际资源，基础设施工程师可以摆脱他们的依赖地狱。此外，由于我们只需要打包应用程序本身及其依赖库，其图像大小将明显小于虚拟机的。因此，分发容器图像比虚拟机更经济。此外，我们已经知道，在容器内运行进程基本上与在其 Linux 主机上运行是相同的，因此几乎不会产生额外开销。总之，容器是轻量级的、自包含的和不可变的。这也清晰地划定了应用程序和基础设施之间的责任边界。

# 开始使用容器。

有许多成熟的容器引擎，如 Docker（[`www.docker.com`](https://www.docker.com)）和 rkt（[`coreos.com/rkt`](https://coreos.com/rkt)），它们已经实现了用于生产的功能，因此您无需从头开始构建一个。此外，由容器行业领导者组成的**Open Container Initiative**（[`www.opencontainers.org`](https://www.opencontainers.org)）已经制定了一些容器规范。这些标准的任何实现，无论基础平台如何，都应具有与 OCI 旨在提供的类似属性，以便在各种操作系统上无缝体验容器。在本书中，我们将使用 Docker（社区版）容器引擎来构建我们的容器化应用程序。

# 为 Ubuntu 安装 Docker

Docker 需要 Yakkety 16.10、Xenial 16.04LTS 和 Trusty 14.04LTS 的 64 位版本。您可以使用`apt-get install docker.io`安装 Docker，但它通常更新速度比 Docker 官方存储库慢。以下是来自 Docker 的安装步骤（[`docs.docker.com/engine/installation/linux/docker-ce/ubuntu/#install-docker-ce`](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/#install-docker-ce)）：

1.  确保您拥有允许`apt`存储库的软件包；如果没有，请获取它们：

```
$ sudo apt-get install apt-transport-https ca-certificates curl software-properties-common 
```

1.  添加 Docker 的`gpg`密钥并验证其指纹是否匹配`9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`：

```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo apt-key fingerprint 0EBFCD88 
```

1.  设置`amd64`架构的存储库：

```
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" 
```

1.  更新软件包索引并安装 Docker CE：

```
 $ sudo apt-get update 
 $ sudo apt-get install docker-ce
```

# 在 CentOS 上安装 Docker

需要 CentOS 7 64 位才能运行 Docker。同样，您可以通过`sudo yum install docker`从 CentOS 的存储库获取 Docker 软件包。同样，Docker 官方指南（[`docs.docker.com/engine/installation/linux/docker-ce/centos/#install-using-the-repository`](https://docs.docker.com/engine/installation/linux/docker-ce/centos/#install-using-the-repository)）中的安装步骤如下：

1.  安装实用程序以启用`yum`使用额外的存储库：

```
    $ sudo yum install -y yum-utils  
```

1.  设置 Docker 的存储库：

```
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo 
```

1.  更新存储库并验证指纹是否匹配：

`060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35`：

```
    $ sudo yum makecache fast   
```

1.  安装 Docker CE 并启动它：

```
$ sudo yum install docker-ce
$ sudo systemctl start docker 
```

# 为 macOS 安装 Docker

Docker 使用微型 Linux moby 和 Hypervisor 框架来在 macOS 上构建本机应用程序，这意味着我们不需要第三方虚拟化工具来开发 Mac 上的 Docker。要从 Hypervisor 框架中受益，您必须将您的 macOS 升级到 10.10.3 或更高版本。

下载 Docker 软件包并安装它：

[`download.docker.com/mac/stable/Docker.dmg`](https://download.docker.com/mac/stable/Docker.dmg)

同样，Docker for Windows 不需要第三方工具。请查看此处的安装指南：[`docs.docker.com/docker-for-windows/install`](https://docs.docker.com/docker-for-windows/install)

现在您已经进入了 Docker。尝试创建和运行您的第一个 Docker 容器；如果您在 Linux 上，请使用 `sudo` 运行：

```
$ docker run alpine ls
bin dev etc home lib media mnt proc root run sbin srv sys tmp usr var
```

您会发现您处于 `root` 目录下而不是当前目录。让我们再次检查进程列表：

```
$ docker run alpine ps aux
PID   USER     TIME   COMMAND
1 root       0:00 ps aux
```

它是隔离的，正如预期的那样。您已经准备好使用容器了。

Alpine 是一个 Linux 发行版。由于其体积非常小，许多人使用它作为构建应用程序容器的基础图像。

# 容器生命周期

使用容器并不像我们习惯的工具那样直观。在本节中，我们将从最基本的想法开始介绍 Docker 的用法，直到我们能够从容器中受益为止。

# Docker 基础知识

当执行 `docker run alpine ls` 时，Docker 在幕后所做的是：

1.  在本地找到图像 `alpine`。如果找不到，Docker 将尝试从公共 Docker 注册表中找到并将其拉取到本地图像存储中。

1.  提取图像并相应地创建一个容器。

1.  使用命令执行图像中定义的入口点，这些命令是图像名称后面的参数。在本例中，它是 `ls`。在基于 Linux 的 Docker 中，默认的入口点是 `/bin/sh -c`。

1.  当入口点进程退出时，容器也会退出。

图像是一组不可变的代码、库、配置和运行应用程序所需的一切。容器是图像的一个实例，在运行时实际上会被执行。您可以使用 `docker inspect IMAGE` 和 `docker inspect CONTAINER` 命令来查看区别。

有时，当我们需要进入容器检查镜像或在内部更新某些内容时，我们将使用选项`-i`和`-t`（`--interactive`和`--tty`）。此外，选项`-d`（`--detach`）使您可以以分离模式运行容器。如果您想与分离的容器进行交互，`exec`和`attach`命令可以帮助我们。`exec`命令允许我们在运行的容器中运行进程，而`attach`按照其字面意思工作。以下示例演示了如何使用它们：

```
$ docker run alpine /bin/sh -c "while :;do echo  
  'meow~';sleep 1;done"
meow~
meow~
...
```

您的终端现在应该被“喵喵喵”淹没了。切换到另一个终端并运行`docker ps`命令，以获取容器的状态，找出喵喵叫的容器的名称和 ID。这里的名称和 ID 都是由 Docker 生成的，您可以使用其中任何一个访问容器。为了方便起见，名称可以在`create`或`run`时使用`--name`标志进行分配：

```
$ docker ps
CONTAINER ID    IMAGE    (omitted)     NAMES
d51972e5fc8c    alpine      ...        zen_kalam

$ docker exec -it d51972e5fc8c /bin/sh
/ # ps
PID   USER     TIME   COMMAND
  1 root       0:00 /bin/sh -c while :;do echo  
  'meow~';sleep 1;done
  27 root       0:00 /bin/sh
  34 root       0:00 sleep 1
  35 root       0:00 ps
  / # kill -s 2 1
  $ // container terminated
```

一旦我们进入容器并检查其进程，我们会看到两个 shell：一个是喵喵叫，另一个是我们所在的位置。在容器内部使用`kill -s 2 1`杀死它，我们会看到整个容器停止，因为入口点已经退出。最后，让我们使用`docker ps -a`列出已停止的容器，并使用`docker rm CONTAINER_NAME`或`docker rm CONTAINER_ID`清理它们。自 Docker 1.13 以来，引入了`docker system prune`命令，它可以帮助我们轻松清理已停止的容器和占用的资源。

# 层、镜像、容器和卷

我们知道镜像是不可变的；容器是短暂的，我们知道如何将镜像作为容器运行。然而，在打包镜像时仍然缺少一步。

镜像是一个只读的堆栈，由一个或多个层组成，而层是文件系统中的文件和目录的集合。为了改善磁盘使用情况，层不仅被锁定在一个镜像上，而且在镜像之间共享；这意味着 Docker 只在本地存储基础镜像的一个副本，而不管从它派生了多少镜像。您可以使用`docker history [image]`命令来了解镜像是如何构建的。例如，如果您键入`docker history alpine`，则 Alpine Linux 镜像中只有一个层。

每当创建一个容器时，它会在基础镜像的顶部添加一个可写层。Docker 在该层上采用了**写时复制**（**COW**）策略。也就是说，容器读取存储目标文件的基础镜像的层，并且如果文件被修改，就会将文件复制到自己的可写层。这种方法可以防止从相同镜像创建的容器相互干扰。`docker diff [CONTAINER]` 命令显示容器与其基础镜像在文件系统状态方面的差异。例如，如果基础镜像中的 `/etc/hosts` 被修改，Docker 会将文件复制到可写层，并且在 `docker diff` 的输出中也只会有这一个文件。

以下图示了 Docker 镜像的层次结构：

![](img/00028.jpeg)

需要注意的是，可写层中的数据会随着容器的删除而被删除。为了持久化数据，您可以使用 `docker commit [CONTAINER]` 命令将容器层提交为新镜像，或者将数据卷挂载到容器中。

数据卷允许容器的读写绕过 Docker 的文件系统，它可以位于主机的目录或其他存储中，比如 Ceph 或 GlusterFS。因此，对卷的磁盘 I/O 可以根据底层存储的实际速度进行操作。由于数据在容器外是持久的，因此可以被多个容器重复使用和共享。通过在 `docker run` 或 `docker create` 中指定 `-v`（`--volume`）标志来挂载卷。以下示例在容器中挂载了一个卷到 `/chest`，并在其中留下一个文件。然后，我们使用 `docker inspect` 来定位数据卷：

```
$ docker run --name demo -v /chest alpine touch /chest/coins
$ docker inspect demo
...
"Mounts": [
 {
    "Type": "volume",
     "Name":(hash-digits),
     "Source":"/var/lib/docker/volumes/(hash- 
      digits)/_data",
      "Destination": "/chest",
      "Driver": "local",
      "Mode": "",
       ...
$ ls /var/lib/docker/volumes/(hash-digits)/_data
      coins
```

Docker CE 在 macOS 上提供的 moby Linux 的默认 `tty` 路径位于：

`~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty`.

您可以使用 `screen` 连接到它。

数据卷的一个用例是在容器之间共享数据。为此，我们首先创建一个容器并在其上挂载卷，然后挂载一个或多个容器，并使用 `--volumes-from` 标志引用卷。以下示例创建了一个带有数据卷 `/share-vol` 的容器。容器 A 可以向其中放入一个文件，容器 B 也可以读取它：

```
$ docker create --name box -v /share-vol alpine nop
c53e3e498ab05b19a12d554fad4545310e6de6950240cf7a28f42780f382c649
$ docker run --name A --volumes-from box alpine touch /share-vol/wine
$ docker run --name B --volumes-from box alpine ls /share-vol
wine
```

此外，数据卷可以挂载在给定的主机路径下，当然其中的数据是持久的：

```
$ docker run --name hi -v $(pwd)/host/dir:/data alpine touch /data/hi
$ docker rm hi
$ ls $(pwd)/host/dir
hi
```

# 分发镜像

注册表是一个存储、管理和分发图像的服务。公共服务，如 Docker Hub ([`hub.docker.com`](https://hub.docker.com)) 和 Quay ([`quay.io`](https://quay.io))，汇集了各种流行工具的预构建图像，如 Ubuntu 和 Nginx，以及其他开发人员的自定义图像。我们多次使用的 Alpine Linux 实际上是从 Docker Hub ([`hub.docker.com/_/alpine`](https://hub.docker.com/_/alpine))中拉取的。当然，你也可以将你的工具上传到这样的服务并与所有人分享。

如果你需要一个私有注册表，但出于某种原因不想订阅注册表服务提供商的付费计划，你总是可以使用 registry ([`hub.docker.com/_/registry`](https://hub.docker.com/_/registry))在自己的计算机上设置一个。

在配置容器之前，Docker 将尝试在图像名称中指示的规则中定位指定的图像。图像名称由三个部分`[registry/]name[:tag]`组成，并根据以下规则解析：

+   如果省略了`registry`字段，则在 Docker Hub 上搜索该名称

+   如果`registry`字段是注册表服务器，则在其中搜索该名称

+   名称中可以有多个斜杠

+   如果省略了标记，则默认为`latest`

例如，图像名称`gcr.io/google-containers/guestbook:v3`指示 Docker 从`gcr.io`下载`google-containers/guestbook`的`v3`版本。同样，如果你想将图像推送到注册表，也要以相同的方式标记你的图像并推送它。要列出当前在本地磁盘上拥有的图像，使用`docker images`，并使用`docker rmi [IMAGE]`删除图像。以下示例显示了如何在不同的注册表之间工作：从 Docker Hub 下载`nginx`图像，将其标记为私有注册表路径，并相应地推送它。请注意，尽管默认标记是`latest`，但你必须显式地标记和推送它。

```
$ docker pull nginx
Using default tag: latest
latest: Pulling from library/nginx
ff3d52d8f55f: Pull complete
...
Status: Downloaded newer image for nginx:latest

$ docker tag nginx localhost:5000/comps/prod/nginx:1.14
$ docker push localhost:5000/comps/prod/nginx:1.14
The push refers to a repository [localhost:5000/comps/prod/nginx]
...
8781ec54ba04: Pushed
1.14: digest: sha256:(64-digits-hash) size: 948
$ docker tag nginx localhost:5000/comps/prod/nginx
$ docker push localhost:5000/comps/prod/nginx
The push refers to a repository [localhost:5000/comps/prod/nginx]
...
8781ec54ba04: Layer already exists
latest: digest: sha256:(64-digits-hash) size: 948
```

大多数注册表服务在你要推送图像时都会要求进行身份验证。`docker login`就是为此目的而设计的。有时，当尝试拉取图像时，你可能会收到`image not found error`的错误，即使图像路径是有效的。这很可能是你未经授权访问保存图像的注册表。要解决这个问题，首先要登录：

```
$ docker pull localhost:5000/comps/prod/nginx
Pulling repository localhost:5000/comps/prod/nginx
Error: image comps/prod/nginx:latest not found
$ docker login -u letme -p in localhost:5000
Login Succeeded
$ docker pull localhost:5000/comps/prod/nginx
Pulling repository localhost:5000/comps/prod/nginx
...
latest: digest: sha256:(64-digits-hash) size: 948
```

除了通过注册表服务分发图像外，还有将图像转储为 TAR 存档文件，并将其导入到本地存储库的选项：

+   `docker commit [CONTAINER]`：将容器层的更改提交为新镜像

+   `docker save --output [filename] IMAGE1 IMAGE2 ...`：将一个或多个镜像保存到 TAR 存档中

+   `docker load -i [filename]`：将`tarball`镜像加载到本地存储库

+   `docker export --output [filename] [CONTAINER]`：将容器的文件系统导出为 TAR 存档

+   `docker import --output [filename] IMAGE1 IMAGE2`：导入文件系统`tarball`

`commit`命令与`save`和`export`看起来基本相同。主要区别在于保存的镜像即使最终将被删除，也会保留层之间的文件；另一方面，导出的镜像将所有中间层压缩为一个最终层。另一个区别是保存的镜像保留元数据，例如层历史记录，但这些在导出的镜像中不可用。因此，导出的镜像通常体积较小。

以下图表描述了容器和镜像之间状态的关系。箭头上的标题是 Docker 的相应子命令：

![](img/00029.jpeg)

# 连接容器

Docker 提供了三种网络类型来管理容器内部和主机之间的通信，即`bridge`、`host`和`none`。

```
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
1224183f2080        bridge              bridge              local
801dec6d5e30        host                host                local
f938cd2d644d        none                null                local
```

默认情况下，每个容器在创建时都连接到桥接网络。在这种模式下，每个容器都被分配一个虚拟接口和一个私有 IP 地址，通过该接口传输的流量被桥接到主机的`docker0`接口。此外，同一桥接网络中的其他容器可以通过它们的 IP 地址相互连接。让我们运行一个通过端口`5000`发送短消息的容器，并观察其配置。`--expose`标志将给定端口开放给容器外部的世界：

```
$ docker run --name greeter -d --expose 5000 alpine \
/bin/sh -c "echo Welcome stranger! | nc -lp 5000"
2069cbdf37210461bc42c2c40d96e56bd99e075c7fb92326af1ec47e64d6b344 $ docker exec greeter ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
inet addr:172.17.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
...
```

在这里，容器`greeter`被分配了 IP`172.17.0.2`。现在运行另一个连接到该 IP 地址的容器：

```
$ docker run alpine telnet 172.17.0.2 5000
Welcome stranger!
Connection closed by foreign host
```

`docker network inspect bridge`命令提供配置详细信息，例如子网段和网关信息。

此外，您可以将一些容器分组到一个用户定义的桥接网络中。这也是连接单个主机上多个容器的推荐方式。用户定义的桥接网络与默认的桥接网络略有不同，主要区别在于您可以通过名称而不是 IP 地址访问其他容器。创建网络是通过`docker network create [NW-NAME]`完成的，将容器附加到它是通过创建时的标志`--network [NW-NAME]`完成的。容器的网络名称默认为其名称，但也可以使用`--network-alias`标志给它另一个别名：

```
$ docker network create room
b0cdd64d375b203b24b5142da41701ad9ab168b53ad6559e6705d6f82564baea
$ docker run -d --network room \
--network-alias dad --name sleeper alpine sleep 60
b5290bcca85b830935a1d0252ca1bf05d03438ddd226751eea922c72aba66417
$ docker run --network room alpine ping -c 1 sleeper
PING sleeper (172.18.0.2): 56 data bytes
...
$ docker run --network room alpine ping -c 1 dad
PING dad (172.18.0.2): 56 data bytes
...
```

主机网络按照其名称的字面意思工作；每个连接的容器共享主机的网络，但同时失去了隔离属性。none 网络是一个完全分离的盒子。无论是入口还是出口，流量都在内部隔离，因为容器上没有网络接口。在这里，我们将一个监听端口`5000`的容器连接到主机网络，并在本地与其通信：

```
$ docker run -d --expose 5000 --network host alpine \
/bin/sh -c "echo im a container | nc -lp 5000"
ca73774caba1401b91b4b1ca04d7d5363b6c281a05a32828e293b84795d85b54
$ telnet localhost 5000
im a container
Connection closed by foreign host
```

如果您在 macOS 上使用 Docker CE，主机指的是 hypervisor 框架上的 moby Linux。

主机和三种网络模式之间的交互如下图所示。主机和桥接网络中的容器都连接了适当的网络接口，并与相同网络内的容器以及外部世界进行通信，但 none 网络与主机接口保持分离。

![](img/00030.jpeg)

除了共享主机网络外，在创建容器时，标志`-p(--publish) [host]:[container]`还允许您将主机端口映射到容器。这个标志意味着`-expose`，因为您无论如何都需要打开容器的端口。以下命令在端口`80`启动一个简单的 HTTP 服务器。您也可以用浏览器查看它。

```
$ docker run -p 80:5000 alpine /bin/sh -c \
"while :; do echo -e 'HTTP/1.1 200 OK\n\ngood day'|nc -lp 5000; done"

$ curl localhost
good day
```

# 使用 Dockerfile

在组装镜像时，无论是通过 Docker commit 还是 export，以受控的方式优化结果都是一个挑战，更不用说与 CI/CD 管道集成了。另一方面，`Dockerfile` 以代码的形式表示构建任务，这显著减少了我们构建任务的复杂性。在本节中，我们将描述如何将 Docker 命令映射到 `Dockerfile` 中，并进一步对其进行优化。

# 编写您的第一个 Dockerfile

`Dockerfile`由一系列文本指令组成，指导 Docker 守护程序形成一个 Docker 镜像。通常，`Dockerfile`是以指令`FROM`开头的，后面跟着零个或多个指令。例如，我们可以从以下一行指令构建一个镜像：

```
docker commit $(   \
docker start $(  \
docker create alpine /bin/sh -c    \
"echo My custom build > /etc/motd" \
 ))
```

它大致相当于以下`Dockerfile`：

```
./Dockerfile:
---
FROM alpine
RUN echo "My custom build" > /etc/motd
---
```

显然，使用`Dockerfile`构建更加简洁和清晰。

`docker build [OPTIONS] [CONTEXT]`命令是与构建任务相关的唯一命令。上下文可以是本地路径、URL 或`stdin`；表示`Dockerfile`的位置。一旦触发构建，`Dockerfile`以及上下文中的所有内容将首先被发送到 Docker 守护程序，然后守护程序将开始按顺序执行`Dockerfile`中的指令。每次执行指令都会产生一个新的缓存层，随后的指令会在级联中的新缓存层上执行。由于上下文将被发送到不一定是本地路径的地方，将`Dockerfile`、代码、必要的文件和`.dockerignore`文件放在一个空文件夹中是一个良好的做法，以确保生成的镜像仅包含所需的文件。

`.dockerignore`文件是一个列表，指示在构建时可以忽略同一目录下的哪些文件，它通常看起来像下面的文件：

```
./.dockerignore:
---
# ignore .dockerignore, .git
.dockerignore 
.git
# exclude all *.tmp files and vim swp file recursively
/*.tmp
/[._]*.s[a-w][a-z]
...
---
```

通常，`docker build`将尝试在`context`下找到一个名为`Dockerfile`的文件来开始构建；但有时出于某些原因，我们可能希望给它另一个名称。`-f`（`--file`）标志就是为了这个目的。另外，另一个有用的标志`-t`（`--tag`）在构建完镜像后能够给一个或多个仓库标签。假设我们想要在`./deploy`下构建一个名为`builder.dck`的`Dockerfile`，并用当前日期和最新标签标记它，命令将是：

```
$ docker build -f deploy/builder.dck  \
-t my-reg.com/prod/teabreak:$(date +"%g%m%d") \
-t my-reg.com/prod/teabreak:latest .
```

# Dockerfile 语法

`Dockerfile`的构建块是十几个或更多的指令；其中大多数是`docker run/create`标志的对应物。这里我们列出最基本的几个：

+   `FROM <IMAGE>[:TAG|[@DIGEST]`：这是告诉 Docker 守护程序当前`Dockerfile`基于哪个镜像。这也是唯一必须在`Dockerfile`中的指令，这意味着你可以有一个只包含一行的`Dockerfile`。像所有其他与镜像相关的命令一样，如果未指定标签，则默认为最新的。

+   `RUN`：

```
RUN <commands>
RUN ["executable", "params", "more params"]
```

`RUN`指令在当前缓存层运行一行命令，并提交结果。两种形式之间的主要差异在于命令的执行方式。第一种称为**shell 形式**，实际上以`/bin/sh -c <commands>`的形式执行命令；另一种形式称为**exec 形式**，它直接使用`exec`处理命令。

使用 shell 形式类似于编写 shell 脚本，因此通过 shell 运算符和行继续、条件测试或变量替换来连接多个命令是完全有效的。但请记住，命令不是由`bash`而是由`sh`处理。

exec 形式被解析为 JSON 数组，这意味着您必须用双引号包装文本并转义保留字符。此外，由于命令不会由任何 shell 处理，数组中的 shell 变量将不会被评估。另一方面，如果基本图像中不存在 shell，则仍然可以使用 exec 形式来调用可执行文件。

+   `CMD`：

```
CMD ["executable", "params", "more params"]
CMD ["param1","param2"]
CMD command param1 param2 ...:
```

`CMD`设置了构建图像的默认命令；它不会在构建时运行命令。如果在 Docker run 时提供了参数，则这里的`CMD`配置将被覆盖。`CMD`的语法规则几乎与`RUN`相同；第一种形式是 exec 形式，第三种形式是 shell 形式，也就是在前面加上`/bin/sh -c`。`ENTRYPOINT`与`CMD`交互的另一个指令；实际上，三种`CMD`形式在容器启动时都会被`ENTRYPOINT`所覆盖。在`Dockerfile`中可以有多个`CMD`指令，但只有最后一个会生效。

+   `ENTRYPOINT`：

```
ENTRYPOINT ["executable", "param1", "param2"] ENTRYPOINT command param1 param2
```

这两种形式分别是执行形式和 shell 形式，语法规则与`RUN`相同。入口点是图像的默认可执行文件。也就是说，当容器启动时，它会运行由`ENTRYPOINT`配置的可执行文件。当`ENTRYPOINT`与`CMD`和`docker run`参数结合使用时，以不同形式编写会导致非常不同的行为。以下是它们组合的规则：

+   +   如果`ENTRYPOINT`是 shell 形式，则`CMD`和 Docker `run`参数将被忽略。命令将变成：

```
     /bin/sh -c entry_cmd entry_params ...     
```

+   +   如果`ENTRYPOINT`是 exec 形式，并且指定了 Docker `run`参数，则`CMD`命令将被覆盖。运行时命令将是：

```
      entry_cmd entry_params run_arguments
```

+   +   如果`ENTRYPOINT`以执行形式存在，并且只配置了`CMD`，则三种形式的运行时命令将变为以下形式：

```
  entry_cmd entry_parms CMD_exec CMD_parms
  entry_cmd entry_parms CMD_parms
  entry_cmd entry_parms /bin/sh -c CMD_cmd 
  CMD_parms   
```

+   `ENV`：

```
ENV key value
ENV key1=value1 key2=value2 ... 
```

`ENV`指令为随后的指令和构建的镜像设置环境变量。第一种形式将键设置为第一个空格后面的字符串，包括特殊字符。第二种形式允许我们在一行中设置多个变量，用空格分隔。如果值中有空格，可以用双引号括起来或转义空格字符。此外，使用`ENV`定义的键也会影响同一文档中的变量。查看以下示例以观察`ENV`的行为：

```
    FROM alpine
    ENV key wD # aw
    ENV k2=v2 k3=v\ 3 \
        k4="v 4"
    ENV k_${k2}=$k3 k5=\"K\=da\"

    RUN echo key=$key ;\
       echo k2=$k2 k3=$k3 k4=$k4 ;\
       echo k_\${k2}=k_${k2}=$k3 k5=$k5

```

在 Docker 构建期间的输出将是：

```
    ...
    ---> Running in 738709ef01ad
    key=wD # aw
    k2=v2 k3=v 3 k4=v 4
    k_${k2}=k_v2=v 3 k5="K=da"
    ...
```

+   `LABEL key1=value1 key2=value2 ...`：`LABEL`的用法类似于`ENV`，但标签仅存储在镜像的元数据部分，并由其他主机程序使用，而不是容器中的程序。它取代了以下形式的`maintainer`指令：

```
LABEL maintainer=johndoe@example.com
```

如果命令带有`-f(--filter)`标志，则可以使用标签过滤对象。例如，`docker images --filter label=maintainer=johndoe@example.com`会查询出带有前面维护者标签的镜像。

+   `EXPOSE <port> [<port> ...]`：此指令与`docker run/create`中的`--expose`标志相同，会在由生成的镜像创建的容器中暴露端口。

+   `USER <name|uid>[:<group|gid>]`：`USER`指令切换用户以运行随后的指令。但是，如果用户在镜像中不存在，则无法正常工作。否则，在使用`USER`指令之前，您必须运行`adduser`。

+   `WORKDIR <path>`：此指令将工作目录设置为特定路径。如果路径不存在，路径将被自动创建。它的工作原理类似于`Dockerfile`中的`cd`，因为它既可以接受相对路径也可以接受绝对路径，并且可以多次使用。如果绝对路径后面跟着一个相对路径，结果将相对于前一个路径：

```
    WORKDIR /usr
    WORKDIR src
    WORKDIR app
    RUN pwd
    ---> Running in 73aff3ae46ac
    /usr/src/app
    ---> 4a415e366388

```

此外，使用`ENV`设置的环境变量会影响路径。

+   `COPY：`

```
COPY <src-in-context> ... <dest-in-container> COPY ["<src-in-context>",... "<dest-in-container>"]
```

该指令将源复制到构建容器中的文件或目录。源可以是文件或目录，目的地也可以是文件或目录。源必须在上下文路径内，因为只有上下文路径下的文件才会被发送到 Docker 守护程序。此外，`COPY`利用`.dockerignore`来过滤将被复制到构建容器中的文件。第二种形式适用于路径包含空格的情况。

+   `ADD`：

```
ADD <src > ... <dest >
ADD ["<src>",... "<dest >"]
```

`ADD`在功能上与`COPY`非常类似：将文件移动到镜像中。除了复制文件外，`<src>`也可以是 URL 或压缩文件。如果`<src>`是一个 URL，`ADD`将下载并将其复制到镜像中。如果`<src>`被推断为压缩文件，它将被提取到`<dest>`路径中。

+   `VOLUME`：

```
VOLUME mount_point_1 mount_point_2 VOLUME ["mount point 1", "mount point 2"]
```

`VOLUME`指令在给定的挂载点创建数据卷。一旦在构建时声明了数据卷，后续指令对数据卷的任何更改都不会持久保存。此外，在`Dockerfile`或`docker build`中挂载主机目录是不可行的，因为存在可移植性问题：无法保证指定的路径在主机中存在。两种语法形式的效果是相同的；它们只在语法解析上有所不同；第二种形式是 JSON 数组，因此需要转义字符，如`"\"`。

+   `ONBUILD [其他指令]`：`ONBUILD`允许您将一些指令推迟到派生图像的后续构建中。例如，我们可能有以下两个 Dockerfiles：

```
    --- baseimg ---
    FROM alpine
    RUN apk add --no-update git make
    WORKDIR /usr/src/app
    ONBUILD COPY . /usr/src/app/
    ONBUILD RUN git submodule init && \
              git submodule update && \
              make
    --- appimg ---
    FROM baseimg
    EXPOSE 80
    CMD ["/usr/src/app/entry"]
```

然后，指令将按以下顺序在`docker build`中进行评估：

```
    $ docker build -t baseimg -f baseimg .
    ---
    FROM alpine
    RUN apk add --no-update git make
    WORKDIR /usr/src/app
    ---
    $ docker build -t appimg -f appimg .
    ---
    COPY . /usr/src/app/
    RUN git submodule init   && \
        git submodule update && \
        make
    EXPOSE 80
    CMD ["/usr/src/app/entry"] 
```

# 组织 Dockerfile

即使编写`Dockerfile`与编写构建脚本相同，但我们还应考虑一些因素来构建高效、安全和稳定的镜像。此外，`Dockerfile`本身也是一个文档，保持其可读性可以简化管理工作。

假设我们有一个应用程序堆栈，其中包括应用程序代码、数据库和缓存，我们可能会从一个`Dockerfile`开始，例如以下内容：

```
---
FROM ubuntu
ADD . /app
RUN apt-get update 
RUN apt-get upgrade -y
RUN apt-get install -y redis-server python python-pip mysql-server
ADD db/my.cnf /etc/mysql/my.cnf
ADD db/redis.conf /etc/redis/redis.conf
RUN pip install -r /app/requirements.txt
RUN cd /app ; python setup.py
CMD /app/start-all-service.sh
```

第一个建议是创建一个专门用于一件事情的容器。因此，我们将在这个`Dockerfile`的开头删除`mysql`和`redis`的安装和配置。接下来，代码将被移入容器中，使用`ADD`，这意味着我们很可能将整个代码库移入容器。通常有许多与应用程序直接相关的文件，包括 VCS 文件、CI 服务器配置，甚至构建缓存，我们可能不希望将它们打包到镜像中。因此，建议使用`.dockerignore`来过滤掉这些文件。顺便说一句，由于`ADD`指令，我们可以做的不仅仅是将文件添加到构建容器中。通常情况下，使用`COPY`更为合适，除非确实有不这样做的真正需要。现在我们的`Dockerfile`更简单了，如下面的代码所示：

```
FROM ubuntu
COPY . /app
RUN apt-get update 
RUN apt-get upgrade -y
RUN apt-get install -y python python-pip
RUN pip install -r /app/requirements.txt
RUN cd /app ; python setup.py
CMD python app.py
```

在构建镜像时，Docker 引擎将尽可能地重用缓存层，这显著减少了构建时间。在我们的`Dockerfile`中，只要存储库有任何更新，我们就必须经历整个更新和依赖项安装过程。为了从构建缓存中受益，我们将根据一个经验法则重新排序指令：首先运行不太频繁的指令。

另外，正如我们之前所描述的，对容器文件系统的任何更改都会导致新的镜像层。即使我们在随后的层中删除了某些文件，这些文件仍然占用着镜像大小，因为它们仍然保存在中间层。因此，我们的下一步是通过简单地压缩多个`RUN`指令来最小化镜像层。此外，为了保持`Dockerfile`的可读性，我们倾向于使用行继续字符“`\`”格式化压缩的`RUN`。

除了与 Docker 的构建机制一起工作之外，我们还希望编写一个可维护的`Dockerfile`，使其更清晰、可预测和稳定。以下是一些建议：

+   使用`WORKDIR`而不是内联`cd`，并为`WORKDIR`使用绝对路径。

+   明确公开所需的端口

+   为基础镜像指定标签

+   使用执行形式启动应用程序

前三个建议非常直接，旨在消除歧义。最后一个建议是关于应用程序如何终止。当来自 Docker 守护程序的停止请求发送到正在运行的容器时，主进程（PID 1）将接收到一个停止信号（`SIGTERM`）。如果进程在一定时间后仍未停止，Docker 守护程序将发送另一个信号（`SIGKILL`）来终止容器。在这里，exec 形式和 shell 形式有所不同。在 shell 形式中，PID 1 进程是"`/bin/sh -c`"，而不是应用程序。此外，不同的 shell 处理信号的方式也不同。有些将停止信号转发给子进程，而有些则不会。Alpine Linux 的 shell 不会转发它们。因此，为了正确停止和清理我们的应用程序，建议使用`exec`形式。结合这些原则，我们有以下`Dockerfile`：

```
FROM ubuntu:16.04
RUN apt-get update && apt-get upgrade -y  \
&& apt-get install -y python python-pip
ENTRYPOINT ["python"]
CMD ["entry.py"]
EXPOSE 5000
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . /app 
```

还有其他一些实践可以使`Dockerfile`更好，包括从专用和更小的基础镜像开始，例如基于 Alpine 的镜像，而不是通用目的的发行版，使用除`root`之外的用户以提高安全性，并在`RUN`中删除不必要的文件。

# 多容器编排

随着我们将越来越多的应用程序打包到隔离的容器中，我们很快就会意识到我们需要一种工具，能够帮助我们同时处理多个容器。在这一部分，我们将从仅仅启动单个容器上升一步，开始编排一组容器。

# 堆叠容器

现代系统通常构建为由多个组件组成的堆栈，这些组件分布在网络上，如应用服务器、缓存、数据库、消息队列等。同时，一个组件本身也是一个包含许多子组件的自包含系统。此外，微服务的趋势为系统之间纠缠不清的关系引入了额外的复杂性。由于这个事实，即使容器技术在部署任务方面给了我们一定程度的缓解，启动一个系统仍然很困难。

假设我们有一个名为 kiosk 的简单应用程序，它连接到 Redis 来管理我们当前拥有的门票数量。一旦门票售出，它会通过 Redis 频道发布一个事件。记录器订阅了 Redis 频道，并在接收到任何事件时将时间戳日志写入 MySQL 数据库。

对于**kiosk**和**recorder**，你可以在这里找到代码以及 Dockerfiles：[`github.com/DevOps-with-Kubernetes/examples/tree/master/chapter2`](https://github.com/DevOps-with-Kubernetes/examples/tree/master/chapter2)。架构如下：

![](img/00031.jpeg)

我们知道如何分别启动这些容器，并将它们连接在一起。基于我们之前讨论的内容，我们首先会创建一个桥接网络，并在其中运行容器：

```

$ docker network create kiosk
$ docker run -d -p 5000:5000 \
    -e REDIS_HOST=lcredis --network=kiosk kiosk-example 
$ docker run -d --network-alias lcredis --network=kiosk redis
$ docker run -d -e REDIS_HOST=lcredis -e MYSQL_HOST=lmysql \
-e MYSQL_ROOT_PASSWORD=$MYPS -e MYSQL_USER=root \
--network=kiosk recorder-example
$ docker run -d --network-alias lmysql -e MYSQL_ROOT_PASSWORD=$MYPS \ 
 --network=kiosk mysql:5.7 
```

到目前为止一切都运行良好。然而，如果下次我们想再次启动相同的堆栈，我们的应用很可能会在数据库之前启动，并且如果有任何传入连接请求对数据库进行任何更改，它们可能会失败。换句话说，我们必须在启动脚本中考虑启动顺序。此外，脚本还存在一些问题，比如如何处理随机组件崩溃，如何管理变量，如何扩展某些组件等等。

# Docker Compose 概述

Docker Compose 是一个非常方便地运行多个容器的工具，它是 Docker CE 发行版中的内置工具。它的作用就是读取`docker-compose.yml`（或`.yaml`）来运行定义的容器。`docker-compose`文件是基于 YAML 的模板，通常是这样的：

```
version: '3'
services:
 hello-world:
 image: hello-world
```

启动它非常简单：将模板保存为`docker-compose.yml`，然后使用`docker-compose up`命令启动它。

```
$ docker-compose up
Creating network "cwd_default" with the default driver
Creating cwd_hello-world_1
Attaching to cwd_hello-world_1
hello-world_1  |
hello-world_1  | Hello from Docker!
hello-world_1  | This message shows that your installation appears to be working correctly.
...
cwd_hello-world_1 exited with code 0

```

让我们看看`docker-compose`在`up`命令后面做了什么。

Docker Compose 基本上是 Docker 的多个容器功能的混合体。例如，`docker build`的对应命令是`docker-compose build`；前者构建一个 Docker 镜像，后者构建`docker-compose.yml`中列出的 Docker 镜像。但需要指出的是：`docker-compose run`命令并不是`docker run`的对应命令；它是从`docker-compose.yml`中的配置中运行特定容器。实际上，与`docker run`最接近的命令是`docker-compose up`。

`docker-compose.yml`文件包括卷、网络和服务的配置。此外，应该有一个版本定义来指示使用的`docker-compose`格式的版本。通过对模板结构的理解，前面的`hello-world`示例所做的事情就很清楚了；它创建了一个名为`hello-world`的服务，它是由`hello-world:latest`镜像创建的。

由于没有定义网络，`docker-compose`将使用默认驱动程序创建一个新网络，并将服务连接到与示例输出中的 1 到 3 行相同的网络。

此外，容器的网络名称将是服务的名称。您可能会注意到控制台中显示的名称与`docker-compose.yml`中的原始名称略有不同。这是因为 Docker Compose 尝试避免容器之间的名称冲突。因此，Docker Compose 使用生成的名称运行容器，并使用服务名称创建网络别名。在此示例中，`hello-world`和`cwd_hello-world_1`都可以在同一网络中解析到其他容器。

# 组合容器

由于 Docker Compose 在许多方面与 Docker 相同，因此更有效的方法是了解如何使用示例编写`docker-compose.yml`，而不是从`docker-compose`语法开始。现在让我们回到之前的`kiosk-example`，并从`version`定义和四个`services`开始：

```
version: '3'
services:
 kiosk-example:
 recorder-example:
 lcredis:
 lmysql:
```

`kiosk-example`的`docker run`参数非常简单，包括发布端口和环境变量。在 Docker Compose 方面，我们相应地填写源镜像、发布端口和环境变量。因为 Docker Compose 能够处理`docker build`，如果本地找不到这些镜像，它将构建镜像。我们很可能希望利用它来进一步减少镜像管理的工作量。

```
kiosk-example:
 image: kiosk-example
 build: ./kiosk
 ports:
  - "5000:5000"
  environment:
    REDIS_HOST: lcredis
```

以相同的方式转换`recorder-example`和`redis`的 Docker 运行，我们得到了以下模板：

```
version: '3'
services:
  kiosk-example:
    image: kiosk-example
    build: ./kiosk
    ports:
    - "5000:5000"
    environment:
      REDIS_HOST: lcredis
  recorder-example:
    image: recorder-example
    build: ./recorder
    environment:
      REDIS_HOST: lcredis
      MYSQL_HOST: lmysql
      MYSQL_USER: root
      MYSQL_ROOT_PASSWORD: mysqlpass
  lcredis:
    image: redis
    ports:
    - "6379"
```

对于 MySQL 部分，它需要一个数据卷来保存数据以及配置。因此，除了`lmysql`部分之外，我们在`services`级别添加`volumes`，并添加一个空映射`mysql-vol`来声明一个数据卷：

```
 lmysql:
 image: mysql:5.7
   environment:
     MYSQL_ROOT_PASSWORD: mysqlpass
   volumes:
   - mysql-vol:/var/lib/mysql
   ports:
   - "3306"
  ---
volumes:
  mysql-vol:
```

结合所有前述的配置，我们得到了最终的模板，如下所示：

```
docker-compose.yml
---
version: '3'
services:
 kiosk-example:
    image: kiosk-example
    build: ./kiosk
    ports:
    - "5000:5000"
    environment:
      REDIS_HOST: lcredis
 recorder-example:
    image: recorder-example
    build: ./recorder
    environment:
      REDIS_HOST: lcredis
      MYSQL_HOST: lmysql
      MYSQL_USER: root
      MYSQL_ROOT_PASSWORD: mysqlpass
 lcredis:
 image: redis
    ports:
    - "6379"
 lmysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: mysqlpass
    volumes:
    - mysql-vol:/var/lib/mysql
    ports:
    - "3306"
volumes:
 mysql-vol: 
```

该文件放在项目的根文件夹中。相应的文件树如下所示：

```
├── docker-compose.yml
├── kiosk
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
└── recorder
 ├── Dockerfile
 ├── process.py
 └── requirements.txt  
```

最后，运行`docker-compose up`来检查一切是否正常。我们可以通过发送`GET /tickets`请求来检查我们的售票亭是否正常运行。

编写 Docker Compose 的模板就是这样简单。现在我们可以轻松地在堆栈中运行应用程序。

# 总结

从 Linux 容器的最原始元素到 Docker 工具栈，我们经历了容器化应用的每个方面，包括打包和运行 Docker 容器，为基于代码的不可变部署编写`Dockerfile`，以及使用 Docker Compose 操作多个容器。然而，本章获得的能力只允许我们在同一主机上运行和连接容器，这限制了构建更大应用的可能性。因此，在下一章中，我们将遇到 Kubernetes，释放容器的力量，超越规模的限制。

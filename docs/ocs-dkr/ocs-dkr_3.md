# 第三章：配置 Docker 容器

在上一章中，我们看到了 Docker 中所有可用的不同命令。我们看了一些示例，涵盖了如何拉取镜像、运行容器、将镜像附加到容器、提交并将镜像推送到存储库的过程。我们还学习了如何编写 Dockerfile，使构建镜像成为一个可重复的过程。

在本章中，我们将更仔细地了解如何控制容器的运行方式。尽管 Docker 容器被隔离，但这并不能阻止其中一个容器中的流浪进程占用其他容器（包括主机）可用的资源。例如，要小心这个命令（不要运行它）：

```
**$ docker run ubuntu /bin/bash -c ":(){ :|:& };:"**

```

通过运行前面的命令，您将 fork bomb 容器以及运行它的主机。

*fork bomb*的维基百科定义如下：

> *"在计算机中，fork bomb 是一种拒绝服务攻击，其中一个进程不断复制自身以耗尽可用的系统资源，导致资源匮乏并减慢或崩溃系统。"*

由于预计 Docker 将用于生产，一个容器使所有其他容器停滞的可能性将是致命的。因此，有机制来限制容器可以拥有的资源量，我们将在本章中进行讨论。

在上一章中，当我们谈论`docker` run 时，我们对卷进行了基本介绍。现在我们将更详细地探讨卷，并讨论它们为什么重要以及如何最好地使用它们。我们还将尝试更改`docker`守护程序使用的存储驱动程序。

另一个方面是网络。在检查运行中的容器时，您可能已经注意到 Docker 会随机选择一个子网并分配一个 IP 地址（默认通常是范围 172.17.42.0/16）。我们将尝试通过设置自己的子网来覆盖这一点，并探索其他可用的帮助管理网络方面的选项。在许多情况下，我们需要在容器之间进行通信（想象一个容器运行您的应用程序，另一个容器运行您的数据库）。由于 IP 地址在构建时不可用，我们需要一种机制来动态发现在其他容器中运行的服务。我们将探讨实现这一点的方法，无论容器是在同一主机上运行还是在不同主机上运行。

简而言之，在本章中，我们将涵盖以下主题：

+   限制资源

+   CPU

+   RAM

+   存储

+   使用卷在容器中管理数据

+   配置 Docker 使用不同的存储驱动程序

+   配置网络

+   端口转发

+   自定义 IP 地址范围

+   链接容器

+   使用容器链接在同一主机内进行链接

+   使用大使容器进行跨主机链接

# 约束资源

对于任何承诺提供沙箱功能的工具来说，提供一种约束资源分配的机制是至关重要的。Docker 在容器启动时提供了限制 CPU 内存和 RAM 使用量的机制。

## 设置 CPU 份额

可以使用`docker run`命令中的`-c`选项来控制容器所占用的 CPU 份额：

```
**$ docker run -c 10 -it ubuntu /bin/bash**

```

值`10`是相对于其他容器给予该容器的优先级。默认情况下，所有容器都具有相同的优先级，因此具有相同的 CPU 处理周期比率，您可以通过运行`$ cat /sys/fs/cgroup/cpu/docker/cpu.shares`来检查（如果您使用的是 OS X 或 Windows，请在执行此操作之前将 SSH 添加到 boot2Docker VM）。但是，您可以在运行容器时提供自己的优先级值。

在容器已经运行时设置 CPU 份额是否可能？是的。编辑`/sys/fs/cgroup/cpu/docker/<container-id>/cpu.shares`文件，并输入您想要给它的优先级。

### 注意

如果提到的位置不存在，请通过运行命令`$ grep -w cgroup /proc/mounts | grep -w cpu`找出`cpu` `cgroup`挂载的位置。

然而，这是一个 hack，如果 Docker 决定改变 CPU 共享的实现方式，将来可能会发生变化。有关更多信息，请访问[`groups.google.com/forum/#!topic/docker-user/-pP8-KgJJGg`](https://groups.google.com/forum/#!topic/docker-user/-pP8-KgJJGg)。

## 设置内存限制

类似地，容器被允许消耗的 RAM 量在启动容器时也可以受到限制：

```
**$ docker run -m <value><optional unit>**

```

在这里，`unit`可以是`b`、`k`、`m`或`g`，分别表示字节、千字节、兆字节和千兆字节。

单位的示例可以表示如下：

```
**$ docker run -m 1024m -dit ubuntu /bin/bash**

```

这为容器设置了 1GB 的内存限制。

与限制 CPU 份额一样，您可以通过运行以下代码来检查默认的内存限制：

```
**$ cat /sys/fs/cgroup/memory/docker/memory.limit_in_bytes**
**18446744073709551615**

```

正如文件名所示，前面的代码以字节为单位打印限制。输出中显示的值对应于 1.8 x 1010 千字节，这实际上意味着没有限制。

在容器已经运行时设置内存限制是否可能？

与 CPU 份额一样，内存限制是通过`cgroup`文件强制执行的，这意味着我们可以通过更改容器的`cgroup`内存文件的值来动态更改限制：

```
**$ echo 1073741824 > \ /sys/fs/cgroup/memory/docker/<container_id>/memory.limit_in_bytes**

```

### 注意

如果`cgroup`文件的位置不存在，请通过运行`$ grep -w cgroup /proc/mounts | grep -w memory`找出文件的挂载位置。

这也是一个黑客，如果 Docker 决定改变内部实现内存限制的方式，可能会在将来发生变化。

有关此更多信息，请访问[`groups.google.com/forum/#!topic/docker-user/-pP8-KgJJGg`](https://groups.google.com/forum/#!topic/docker-user/-pP8-KgJJGg)。

## 在虚拟文件系统（Devicemapper）上设置存储限制

限制磁盘使用可能有点棘手。没有直接的方法来限制容器可以使用的磁盘空间的数量。默认存储驱动程序 AUFS 不支持磁盘配额，至少没有没有黑客（困难是因为 AUFS 没有自己的块设备。访问[`aufs.sourceforge.net/aufs.html`](http://aufs.sourceforge.net/aufs.html)以获取有关 AUFS 工作原理的深入信息）。在撰写本书时，需要磁盘配额的 Docker 用户选择`devicemapper`驱动程序，该驱动程序允许每个容器使用一定量的磁盘空间。但是，正在进行跨存储驱动程序的更通用机制，并且可能会在将来的版本中引入。

### 注意

`devicemapper`驱动程序是用于将块设备映射到更高级虚拟块设备的 Linux 内核框架。

`devicemapper`驱动程序基于两个块设备（将它们视为虚拟磁盘），一个用于数据，另一个用于元数据，创建一个存储块的`thin`池。默认情况下，这些块设备是通过将稀疏文件挂载为回环设备来创建的。

### 注意

**稀疏文件**是一个文件，其中大部分是空白空间。因此，100 GB 的稀疏文件实际上可能只包含一点字节在开头和结尾（并且只占据磁盘上的这些字节），但对应用程序来说，它可能是一个 100 GB 的文件。在读取稀疏文件时，文件系统会在运行时将空块透明地转换为实际填充了零字节的实际块。它通过文件的元数据跟踪已写入和空块的位置。在类 UNIX 操作系统中，回环设备是一个伪设备，它使文件作为块设备可访问。

“薄”池之所以被称为“薄”，是因为只有在实际写入块时，它才将存储块标记为已使用（来自池）。每个容器都被配置了一个特定大小的基础薄设备，容器不允许累积超过该大小限制的数据。

默认限制是什么？“薄”池的默认限制为 100 GB。但由于用于此池的回环设备是稀疏文件，因此最初不会占用这么多空间。

为每个容器和镜像创建的基础设备的默认大小限制为 10 GB。同样，由于这是稀疏的，因此最初不会占用物理磁盘上的这么多空间。但是，随着大小限制的增加，它占用的空间也会增加，因为块设备的大小越大，稀疏文件的（虚拟）大小就越大，需要存储的元数据也越多。

如何更改这些默认值？您可以在运行`docker`守护程序时使用`--storage-opts`选项更改这些选项，该选项带有`dm`（用于`devicemapper`）前缀。

### 注意

在运行本节中的任何命令之前，请使用`docker save`备份所有镜像并停止`docker`守护程序。完全删除`/var/lib/docker`（Docker 存储图像数据的路径）可能也是明智的。

### Devicemapper 配置

可用的各种配置如下：

+   `dm.basesize`：这指定了基础设备的大小，容器和镜像将使用它。默认情况下，这被设置为 10 GB。创建的设备是稀疏的，因此最初不会占用 10 GB。相反，它将随着数据写入而填满，直到达到 10 GB 的限制。

```
**$ docker -d -s devicemapper --storage-opt dm.basesize=50G**

```

+   `dm.loopdatasize`：这是“薄”池的大小。默认大小为 100 GB。需要注意的是，这个文件是稀疏的，因此最初不会占用这个空间；相反，随着越来越多的数据被写入，它将逐渐填满：

```
**$ docker -d -s devicemapper --storage-opt dm.loopdatasize=1024G**

```

+   `dm.loopmetadatasize`：如前所述，创建了两个块设备，一个用于数据，另一个用于元数据。此选项指定创建此块设备时要使用的大小限制。默认大小为 2 GB。这个文件也是稀疏的，因此最初不会占用整个大小。建议的最小大小是总池大小的 1％：

```
**$ docker -d -s devicemapper --storage-opt dm.loopmetadatasize=10G**

```

+   `dm.fs`：这是用于基础设备的文件系统类型。支持`ext4`和`xfs`文件系统，尽管默认情况下采用`ext4`：

```
**$ docker -d -s devicemapper --storage-opt dm.fs=xfs**

```

+   `dm.datadev`：这指定要使用的自定义块设备（而不是回环）用于`thin`池。如果您使用此选项，建议同时为数据和元数据指定块设备，以完全避免使用回环设备：

```
**$ docker -d -s devicemapper --storage-opt dm.datadev=/dev/sdb1 \-storage-opt dm.metadatadev=/dev/sdc1**

```

还有更多选项可用，以及关于所有这些工作原理的清晰解释，请参阅[`github.com/docker/docker/tree/master/daemon/graphdriver/devmapper/README.md`](https://github.com/docker/docker/tree/master/daemon/graphdriver/devmapper/README.md)。

另一个很好的资源是 Docker 贡献者 Jérôme Petazzoni 在[`jpetazzo.github.io/2014/01/29/docker-device-mapper-resize/`](http://jpetazzo.github.io/2014/01/29/docker-device-mapper-resize/)上发布的有关调整容器大小的博文。

### 注意

如果切换存储驱动程序，旧容器和图像将不再可见。

在本节的开头提到了有可能通过一个黑客手段实现配额并仍然使用 AUFS。这个黑客手段涉及根据需要创建基于`ext4`文件系统的回环文件系统，并将其作为容器的卷进行绑定挂载：

```
**$ DIR=$(mktemp -d)**
**$ DB_DIR=(mktemp -d)**
**$ dd if=/dev/zero of=$DIR/data count=102400**
**$ yes | mkfs -t ext4 $DIR/data**
**$ mkdir $DB_DIR/db**
**$ sudo mount -o loop=/dev/loop0 $DIR/data $DB_DIR**

```

您现在可以使用`docker run`命令的`-v`选项将`$DB_DIR`目录绑定到容器：

```
**$ docker run -v $DB_DIR:/var/lib/mysql mysql mysqld_safe.**

```

# 使用卷管理容器中的数据

Docker 卷的一些显着特点如下所述：

+   卷是与容器的`root`文件系统分开的目录。

+   它由`docker`守护程序直接管理，并可以在容器之间共享。

+   卷还可以用于在容器内挂载主机系统的目录。

+   对卷进行的更改不会在从运行中的容器更新图像时包含在内。

+   由于卷位于容器文件系统之外，它没有数据层或快照的概念。因此，读取和写入直接在卷上进行。

+   如果多个容器使用相同的卷，则卷将持久存在，直到至少有一个容器使用它。

创建卷很容易。只需使用`-v`选项启动容器：

```
**$ docker run -d -p 80:80 --name apache-1 -v /var/www apache.**

```

现在请注意，卷没有`ID`参数，因此您无法像命名容器或标记图像一样确切地命名卷。但是，可以利用一个容器使用它至少一次的条件来使卷持久存在，这引入了仅包含数据的容器的概念。

### 注意

自 Docker 版本 1.1 以来，如果您愿意，可以使用`-v`选项将主机的整个文件系统绑定到容器，就像这样：

```
**$ docker run -v /:/my_host ubuntu:ro ls /my_host****.**

```

但是，禁止挂载到容器的/，因此出于安全原因，您无法替换容器的`root`文件系统。

## 仅数据容器

数据专用容器是一个除了公开其他数据访问容器可以使用的卷之外什么也不做的容器。数据专用容器用于防止容器访问卷停止或由于意外崩溃而被销毁。

## 使用另一个容器的卷

一旦我们使用`-v`选项启动容器，就创建了一个卷。我们可以使用`--volumes-from`选项与其他容器共享由容器创建的卷。此选项的可能用例包括备份数据库、处理日志、对用户数据执行操作等。

## 用例 - 在 Docker 上生产中使用 MongoDB

作为一个用例，假设您想在生产环境中使用**MongoDB**，您将运行一个 MongoDB 服务器以及一个`cron`作业，定期备份数据库快照。

### 注意

MongoDB 是一个文档数据库，提供高性能、高可用性和易扩展性。您可以在[`www.mongodb.org`](http://www.mongodb.org)获取有关 MongoDB 的更多信息。

让我们看看如何使用`docker`卷设置 MongoDB：

1.  首先，我们需要一个数据专用容器。该容器的任务只是公开 MongoDB 存储数据的卷：

```
 **$ docker run -v /data/db --name data-only mongo \ echo "MongoDB stores all its data in /data/db"**

```

1.  然后，我们需要运行 MongoDB 服务器，该服务器使用数据专用容器创建的卷：

```
**$ docker run -d --volumes-from data-only -p 27017:27017 \ --name mongodb-server mongo mongod**

```

### 注意

`mongod`命令运行 MongoDB 服务器，通常作为守护程序/服务运行。它通过端口`27017`访问。

1.  最后，我们需要运行`backup`实用程序。在这种情况下，我们只是将 MongoDB 数据存储转储到主机上的当前目录：

```
**$ docker run -d --volumes-from data-only --name mongo-backup \ -v $(pwd):/backup mongo $(mkdir -p /backup && cd /backup && mongodump)**

```

### 注意

这绝不是在生产中设置 MongoDB 的详尽示例。您可能需要一个监视 MongoDB 服务器健康状况的过程。您还需要使 MongoDB 服务器容器可以被您的应用程序容器发现（我们将在后面详细学习）。

# 配置 Docker 以使用不同的存储驱动程序

在使用不同的存储驱动程序之前，使用`docker save`备份所有图像，并停止`docker`守护程序。一旦备份了所有重要图像，删除`/var/lib/docker`。更改存储驱动程序后，可以恢复保存的图像。

我们现在将把默认存储驱动程序 AUFS 更改为两种备用存储驱动程序-devicemapper 和 btrfs。

## 使用 devicemapper 作为存储驱动程序

切换到 devicemapper 驱动程序很容易。只需使用-s 选项启动 docker 守护程序：

```
**$ docker -d -s devicemapper**

```

此外，您可以使用--storage-opts 标志提供各种 devicemapper 驱动程序选项。devicemapper 驱动程序的各种可用选项和示例已在本章的*限制资源存储*部分中介绍。

### 注意

如果您在没有 AUFS 的 RedHat/Fedora 上运行，Docker 将使用 devicemapper 驱动程序，该驱动程序可用。

切换存储驱动程序后，您可以通过运行 docker info 来验证更改。

## 使用 btrfs 作为存储驱动程序

要将 btrfs 用作存储驱动程序，您必须首先设置它。本节假定您正在运行 Ubuntu 14.04 操作系统。根据您运行的 Linux 发行版，命令可能会有所不同。以下步骤将设置一个带有 btrfs 文件系统的块设备：

1.  首先，您需要安装 btrfs 及其依赖项：

```
**# apt-get -y btrfs-tools**

```

1.  接下来，您需要创建一个 btrfs 文件系统类型的块设备：

```
**# mkfs btrfs /dev/sdb**

```

1.  现在为 Docker 创建目录（到此时，您应该已经备份了所有重要的镜像并清理了/var/lib/docker）。

```
**# mkdir /var/lib/docker**

```

1.  然后在/var/lib/docker 挂载 btrfs 块设备：

```
**# mount /dev/sdb var/lib/docker**

```

1.  检查挂载是否成功：

```
**$ mount | grep btrfs**
**/dev/sdb on /var/lib/docker type btrfs (rw)**

```

### 注意

来源：[`serverascode.com/2014/06/09/docker-btrfs.html`](http://serverascode.com/2014/06/09/docker-btrfs.html)。

现在，您可以使用-s 选项启动 docker 守护程序：

```
**$ docker -d -s btrfs**

```

切换存储驱动程序后，您可以通过运行 docker info 命令来验证其中的更改。

# 配置 Docker 的网络设置

Docker 为每个容器创建一个单独的网络堆栈和一个虚拟桥（docker0）来管理容器内部、容器与主机之间以及两个容器之间的网络通信。

有一些网络配置可以作为 docker run 命令的参数设置。它们如下：

+   --dns：DNS 服务器是将 URL（例如[`www.docker.io`](http://www.docker.io)）解析为运行网站的服务器的 IP 地址。

+   --dns-search：这允许您设置 DNS 搜索服务器。

### 注意

如果将 DNS 搜索服务器解析`abc`为`abc.example.com`，则`example.com`设置为 DNS 搜索域。如果您有许多子域在您的公司网站中需要经常访问，这将非常有用。反复输入整个 URL 太痛苦了。如果您尝试访问一个不是完全合格的域名的站点（例如`xyz.abc.com`），它会为查找添加搜索域。来源：[`superuser.com/a/184366`](http://superuser.com/a/184366)。

+   `-h`或`--hostname`：这允许您设置主机名。这将被添加为对容器的面向主机的 IP 的`/etc/hosts`路径的条目。

+   `--link`：这是另一个可以在启动容器时指定的选项。它允许容器与其他容器通信，而无需知道它们的实际 IP 地址。

+   `--net`：此选项允许您为容器设置网络模式。它可以有四个值：

+   `桥接`：这为 docker 容器创建了一个网络堆栈。

+   `none`：不会为此容器创建任何网络堆栈。它将完全隔离。

+   `container:<name|id>`：这使用另一个容器的网络堆栈。

+   `host`：这使用主机的网络堆栈。

### 提示

这些值具有副作用，例如本地系统服务可以从容器中访问。此选项被认为是不安全的。

+   `--expose`：这将暴露容器的端口，而不在主机上发布它。

+   `--publish-all`：这将所有暴露的端口发布到主机的接口。

+   `--publish`：这将以以下格式将容器的端口发布到主机：`ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort | containerPort`。

### 提示

如果未给出`--dns`或`--dns-search`，则容器的`/etc/resolv.conf`文件将与守护程序正在运行的主机的`/etc/resolv.conf`文件相同。

然而，当你运行它时，`docker`守护进程也可以给出一些配置。它们如下所述：

### 注意

这些选项只能在启动`docker`守护程序时提供，一旦运行就无法调整。这意味着您必须在`docker -d`命令中提供这些参数。

+   `--ip`：此选项允许我们在面向容器的`docker0`接口上设置主机的 IP 地址。因此，这将是绑定容器端口时使用的默认 IP 地址。例如，此选项可以显示如下：

```
**$ docker -d --ip 172.16.42.1**

```

+   `--ip-forward`：这是一个`布尔`选项。如果设置为`false`，则运行守护程序的主机将不会在容器之间或从外部世界到容器之间转发数据包，从网络角度完全隔离它。

### 注意

可以使用`sysctl`命令来检查此设置：

```
**$ sysctl net.ipv4.ip_forward**
**net.ipv4.ip_forward = 1**
**.**

```

+   `--icc`：这是另一个`布尔`选项，代表`容器间通信`。如果设置为`false`，容器将彼此隔离，但仍然可以向包管理器等发出一般的 HTTP 请求。

### 注意

如何只允许那两个容器之间的通信？通过链接。我们将在*链接容器*部分详细探讨链接。

+   `-b 或--bridge`：您可以让 Docker 使用自定义桥接而不是`docker0`。（创建桥接超出了本讨论的范围。但是，如果您感兴趣，可以在[`docs.docker.com/articles/networking/#building-your-own-bridge`](http://docs.docker.com/articles/networking/#building-your-own-bridge)找到更多信息。）

+   `-H 或--host`：此选项可以接受多个参数。Docker 具有 RESTful API。守护程序充当服务器，当您运行客户端命令（如`run`和`ps`）时，它会向服务器发出`GET`和`POST`请求，服务器执行必要的操作并返回响应。`-H`标志用于告诉`docker`守护程序必须监听哪些通道以接收客户端命令。参数可以如下：

+   以`tcp://<host>:<port>`形式表示的 TCP 套接字

+   `unix:///path/to/socket`形式的 UNIX 套接字

## 在容器和主机之间配置端口转发

容器可以在没有任何特殊配置的情况下连接到外部世界，但外部世界不允许窥视它们。这是一项安全措施，显而易见，因为容器都通过虚拟桥连接到主机，从而有效地将它们放置在虚拟网络中。但是，如果您在容器中运行一个希望暴露给外部世界的服务呢？

端口转发是暴露在容器中运行的服务的最简单的方法。在镜像的 Dockerfile 中提到需要暴露的端口是明智的。在早期版本的 Docker 中，可以在 Dockerfile 本身指定 Dockerfile 应绑定到的主机端口，但这样做是因为有时主机中已经运行的服务会干扰容器。现在，您仍然可以在 Dockerfile 中指定要暴露的端口（使用`EXPOSE`指令），但如果要将其绑定到您选择的端口，需要在启动容器时执行此操作。

有两种方法可以启动容器并将其端口绑定到主机端口。它们的解释如下：

+   `-P 或--publish-all`：使用`docker run`启动容器，并使用`-P`选项将发布在镜像的 Dockerfile 中使用`EXPOSE`指令暴露的所有端口。Docker 将浏览暴露的端口，并将它们绑定到`49000`到`49900`之间的随机端口。

+   `-p 或--publish`：此选项允许您明确告诉 Docker 应将哪个 IP 上的哪个端口绑定到容器上的端口（当然，主机中的一个接口应该具有此 IP）。可以多次使用该选项进行多个绑定：

1.  `docker run -p ip:host_port:container_port`

1.  `docker run -p ip::container_port`

1.  `docker run -p host_port:container_port`

## 自定义 IP 地址范围

我们已经看到了如何将容器的端口绑定到主机的端口，如何配置容器的 DNS 设置，甚至如何设置主机的 IP 地址。但是，如果我们想要自己设置容器和主机之间网络的子网怎么办？Docker 在 RFC 1918 提供的可用私有 IP 地址范围中创建了一个虚拟子网。

设置自己的子网范围非常容易。`docker`守护程序的`--bip`选项可用于设置桥接的 IP 地址以及它将创建容器的子网：

```
**$ docker -d --bip 192.168.0.1/24**

```

在这种情况下，我们已将 IP 地址设置为`192.168.0.1`，并指定它必须将 IP 地址分配给子网范围`192.168.0.0/24`中的容器（即从`192.168.0.2`到`192.168.0.254`，共 252 个可能的 IP 地址）。

就是这样！在[`docs.docker.com/articles/networking/`](https://docs.docker.com/articles/networking/)上有更多高级网络配置和示例。一定要查看它们。

# 链接容器

如果您只有一个普通的 Web 服务器想要暴露给互联网，那么将容器端口绑定到主机端口就可以了。然而，大多数生产系统由许多不断相互通信的单独组件组成。诸如数据库服务器之类的组件不应绑定到公开可见的 IP，但运行前端应用程序的容器仍然需要发现数据库容器并连接到它们。在应用程序中硬编码容器的 IP 地址既不是一个干净的解决方案，也不会起作用，因为 IP 地址是随机分配给容器的。那么我们如何解决这个问题呢？答案如下。

## 在同一主机内链接容器

可以在启动容器时使用`--link`选项指定链接：

```
**$ docker run --link CONTAINER_IDENTIFIER:ALIAS . . .**

```

这是如何工作的？当给出链接选项时，Docker 会向容器的`/etc/hosts`文件添加一个条目，其中`ALIAS`命令作为主机名，容器命名为`CONTAINER_IDENTIFIER`的 IP 地址。

### 注意

`/etc/hosts`文件可用于覆盖 DNS 定义，即将主机名指向特定的 IP 地址。在主机名解析期间，在向 DNS 服务器发出请求之前，将检查`/etc/hosts`。

例如，下面显示了命令行代码：

```
**$ docker run --name pg -d postgres**
**$ docker run --link pg:postgres postgres-app**

```

上面的命令运行了一个 PostgreSQL 服务器（其 Dockerfile 公开了端口 5432，PostgeSQL 的默认端口），第二个容器将使用`postgres`别名链接到它。

### 注意

PostgreSQL 是一个完全符合**ACID**的功能强大的开源对象关系数据库系统。

## 使用 ambassador 容器进行跨主机链接

当所有容器都在同一主机上时，链接容器可以正常工作，但是 Docker 的容器通常可能分布在不同的主机上，在这些情况下链接会失败，因为在当前主机上运行的`docker`守护程序不知道在不同主机上运行的容器的 IP 地址。此外，链接是静态的。这意味着如果容器重新启动，其 IP 地址将更改，并且所有链接到它的容器将失去连接。一个可移植的解决方案是使用 ambassador 容器。

以下图显示了 ambassador 容器：

![使用 ambassador 容器进行跨主机链接](img/4787OS_03_02.jpg)

在这种架构中，一个主机中的数据库服务器暴露给另一个主机。同样，如果数据库容器发生更改，只需要重新启动`host1`阶段的 ambassador 容器。

### 用例 - 多主机 Redis 环境

让我们使用`progrium/ambassadord`命令设置一个多主机 Redis 环境。还有其他可以用作大使容器的镜像。它们可以使用`docker search`命令搜索，或者在[`registry.hub.docker.com`](https://registry.hub.docker.com)上搜索。

### 注意

Redis 是一个开源的、网络化的、内存中的、可选的持久性键值数据存储。它以快速的速度而闻名，无论是读取还是写入。

在这个环境中，有两个主机，`Host` `1`和`Host` `2`。`Host` `1`的 IP 地址是`192.168.0.100`，是私有的（不暴露给公共互联网）。`Host` `2`在 192.168.0.1，并绑定到一个公共 IP。这是运行您的前端 Web 应用程序的主机。

### 注意

要尝试这个例子，启动两个虚拟机。如果您使用 Vagrant，我建议使用安装了 Docker 的 Ubuntu 镜像。如果您有 Vagrant v1.5，可以通过运行`$ vagrant init phusion/ubuntu-14.04-amd64`来使用 Phusion 的 Ubuntu 镜像。

#### 主机 1

在第一个主机上，运行以下命令：

```
**$ docker run -d --name redis --expose 6379 dockerfile/redis**

```

这个命令启动了一个 Redis 服务器，并暴露了端口`6379`（这是 Redis 服务器运行的默认端口），但没有将其绑定到任何主机端口。

以下命令启动一个大使容器，链接到 Redis 服务器，并将端口 6379 绑定到其私有网络 IP 地址的 6379 端口（在这种情况下是 192.168.0.100）。这仍然不是公开的，因为主机是私有的（不暴露给公共互联网）：

```
**$ docker run -d --name redis-ambassador-h1 \**
 **-p 192.168.0.100:6379:6379 --link redis:redis \**
 **progrium/ambassadord --links**

```

#### 主机 2

在另一个主机（如果您在开发中使用 Vagrant，则是另一个虚拟机），运行以下命令：

```
**$ docker run -d --name redis-ambassador-h2 --expose 6379 \**
**progrium/ambassadord 192.168.0.100:6379**

```

这个大使容器监听目标 IP 的端口，这种情况下是主机 1 的 IP 地址。我们已经暴露了端口 6379，这样它现在可以被我们的应用容器连接：

```
**$ docker run -d --name application-container \--link redis-ambassador-h2:redis myimage mycommand**

```

这将是在互联网上公开的容器。由于 Redis 服务器在私有主机上运行，因此无法从私有网络外部受到攻击。

# 总结

在本章中，我们看到了如何在 Docker 容器中配置 CPU、RAM 和存储等资源。我们还讨论了如何使用卷和卷容器来管理容器中应用程序产生的持久数据。我们了解了切换 Docker 使用的存储驱动程序以及各种网络配置及其相关用例。最后，我们看到了如何在主机内部和跨主机之间链接容器。

在下一章中，我们将看看哪些工具和方法可以帮助我们考虑使用 Docker 部署我们的应用程序。我们将关注的一些内容包括多个服务的协调、服务发现以及 Docker 的远程 API。我们还将涵盖安全考虑。

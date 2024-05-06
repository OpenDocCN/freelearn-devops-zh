# 第五章。Docker 容器的安全性和 QoS

在本章中，我们将学习安全性是如何在容器的上下文中实现的，以及如何实施 QoS 策略以确保 CPU 和 IO 等资源按预期共享。大部分讨论将集中在这些主题在 Docker 上下文中的相关性。

在本章中，我们将涵盖以下内容：

+   文件系统限制

+   只读挂载点

+   写时复制

+   Linux 功能和 Docker

+   在 AWS ECS（EC2 容器服务）中保护容器

+   理解 Docker 安全性 I - 内核命名空间

+   理解 Docker 安全性 II - cgroups

+   使用 AppArmour 来保护 Docker 容器

+   Docker 安全基准

# 文件系统限制

在本节中，我们将研究 Docker 容器启动时的文件系统限制。下一节解释了只读挂载点和写时复制文件系统，这些文件系统被用作 Docker 容器的基础和内核对象的表示。

## 只读挂载点

Docker 需要访问文件系统，如 sysfs 和 proc，以使进程正常运行。但它不一定需要修改这些挂载点。

两个主要的只读挂载点是：

+   `/sys`

+   `/proc`

### sysfs

sysfs 文件系统加载到挂载点`/sys`中。sysfs 是一种表示内核对象、它们的属性和它们之间关系的机制。它提供了两个组件：

+   用于通过 sysfs 导出这些项目的内核编程接口

+   一个用户界面，用于查看和操作这些项目，它映射回它们所代表的内核对象

以下代码显示了挂载的挂载点：

```
{
  Source:      "sysfs",
  Destination: "/sys",
  Device:      "sysfs",
  Flags:       defaultMountFlags | syscall.MS_RDONLY,
},
```

上述代码的参考链接在[`github.com/docker/docker/blob/ecc3717cb17313186ee711e624b960b096a9334f/daemon/execdriver/native/template/default_template_linux.go`](https://github.com/docker/docker/blob/ecc3717cb17313186ee711e624b960b096a9334f/daemon/execdriver/native/template/default_template_linux.go)。

### procfs

proc 文件系统（procfs）是 Unix-like 操作系统中的一个特殊文件系统，它以分层文件样式的结构呈现有关进程和其他系统信息的信息。它加载到`/proc`中。它提供了一个更方便和标准化的方法来动态访问内核中保存的进程数据，而不是传统的跟踪方法或直接访问内核内存。它在引导时映射到名为`/proc`的挂载点：

```
{
  Source:      "proc",
  Destination: "/proc",
  Device:      "proc",
  Flags:       defaultMountFlags,
},
```

使用`/proc`的只读路径：

```
ReadonlyPaths: []string{
  "/proc/asound",
  "/proc/bus",
  "/proc/fs",
  "/proc/irq",
  "/proc/sys",
  "/proc/sysrq-trigger",
}
```

### /dev/pts

这是另一个在创建过程中作为读写挂载的挂载点。`/dev/pts`完全存在于内存中，没有任何内容存储在磁盘上，因此可以安全地以读写模式加载它。

`/dev/pts`中的条目是伪终端（简称 pty）。Unix 内核有终端的通用概念。终端提供了应用程序通过终端设备显示输出和接收输入的方式。一个进程可能有一个控制终端。对于文本模式应用程序，这是它与用户交互的方式：

```
{
  Source:      "devpts",
  Destination: "/dev/pts",
  Device:      "devpts",
  Flags:       syscall.MS_NOSUID | syscall.MS_NOEXEC,
  Data:        "newinstance,ptmxmode=0666,mode=0620,gid=5",
},
```

### /sys/fs/cgroup

这是 cgroups 实现的挂载点，并且在容器中加载为`MS_RDONLY`：

```
{
  Source:      "cgroup",
  Destination: "/sys/fs/cgroup",
  Device:      "cgroup",
  Flags:       defaultMountFlags | syscall.MS_RDONLY,
},
```

## 写时复制

Docker 使用联合文件系统，这是写时复制文件系统。这意味着容器可以使用相同的文件系统镜像作为容器的基础。当容器向镜像写入内容时，它会被写入到特定于容器的文件系统中。即使它们是从相同的文件系统镜像创建的，一个容器也不能访问另一个容器的更改。一个容器不能改变镜像内容以影响另一个容器中的进程。以下图解释了这个过程：

![写时复制](img/00041.jpeg)

# Linux 功能

1.2 版本之前的 Docker 容器可以在特权模式下获得完整的功能，或者它们可以遵循允许的功能白名单，同时放弃所有其他功能。如果使用`--privileged`标志，它将授予容器所有功能。这在生产中是不推荐的，因为它真的很不安全；它允许 Docker 作为直接主机下的进程拥有所有特权。

使用 Docker 1.2 引入了两个`docker run`标志：

+   `--cap-add`

+   `--cap-drop`

这两个标志为容器提供了细粒度的控制，例如：

+   更改 Docker 容器接口的状态：

```
**docker run --cap-add=NET_ADMIN busybox sh -c "ip link eth0 down"**

```

+   防止 Docker 容器中的任何 chown：

```
**docker run --cap-drop=CHOWN ...**

```

+   允许除`mknod`之外的所有功能：

```
**docker run --cap-add=ALL --cap-drop=MKNOD ...**

```

Docker 默认以受限的功能集启动容器。功能将根和非根的二进制模式转换为更精细的访问控制。例如，提供 HTTP 请求的 Web 服务器需要绑定到端口 80 进行 HTTP 和端口 443 进行 HTTPs。这些服务器不需要以根模式运行。这些服务器可以被授予`net_bind_service`功能。

在这个情境中，容器和服务器有一些不同。服务器需要以 root 模式运行一些进程。例如，ssh，cron 和网络配置来处理 dhcp 等。另一方面，容器不需要这种访问。

以下任务不需要在容器中发生：

+   ssh 访问由 Docker 主机管理

+   cron 作业应该在用户模式下运行

+   网络配置，如 ipconfig 和路由，不应该在容器内发生

我们可以安全地推断容器可能不需要 root 权限。

可以拒绝的示例如下：

+   不允许挂载操作

+   不允许访问套接字

+   阻止对文件系统操作的访问，如更改文件属性或文件所有权

+   阻止容器加载新模块

Docker 只允许以下功能：

```
Capabilities: []string{
  "CHOWN",
  "DAC_OVERRIDE",
  "FSETID",
  "FOWNER",
  "MKNOD",
  "NET_RAW",
  "SETGID",
  "SETUID",
  "SETFCAP",
  "SETPCAP",
  "NET_BIND_SERVICE",
  "SYS_CHROOT",
  "KILL",
  "AUDIT_WRITE",
},
```

对先前代码的引用在[`github.com/docker/docker/blob/master/daemon/execdriver/native/template/default_template_linux.go`](https://github.com/docker/docker/blob/master/daemon/execdriver/native/template/default_template_linux.go)。

可以在 Linux man-pages 中找到所有可用功能的完整列表([`man7.org/linux/man-pages/man7/capabilities.7.html`](http://man7.org/linux/man-pages/man7/capabilities.7.html))。

运行 Docker 容器的一个主要风险是，容器的默认功能和挂载集可能提供不完整的隔离，无论是独立使用还是与内核漏洞结合使用。

Docker 支持添加和删除功能，允许使用非默认配置文件。这可以通过删除功能或添加功能使 Docker 更安全或更不安全。用户的最佳做法是删除除了明确需要的功能之外的所有功能。

# 在 AWS ECS 中保护容器

亚马逊**EC2 容器服务**(**ECS**)提供了一个高度可扩展、高性能的容器管理服务，支持 Docker 容器。它允许您轻松地在一组托管的亚马逊 EC2 实例上运行应用程序。Amazon ECS 消除了您安装、操作和扩展自己的集群管理基础设施的需要。通过简单的 API 调用，您可以启动和停止启用 Docker 的应用程序，并查询集群的完整状态。

在以下示例中，我们将看到如何使用两个 Docker 容器部署一个安全的 Web 应用程序，一个包含一个简单的 Web 应用程序（应用程序容器），另一个包含启用了限流的反向代理（代理容器），可以用来保护 Web 应用程序。这些容器将在 Amazon EC2 实例上使用 ECS 部署。如下图所示，所有网络流量将通过限流请求的代理容器路由。此外，我们可以在代理容器上使用各种安全软件执行过滤、日志记录和入侵检测等活动。

以下是这样做的步骤：

1.  我们将从 GitHub 项目构建一个基本的 PHP Web 应用程序容器。以下步骤可以在单独的 EC2 实例或本地机器上执行：

```
**$ sudo yum install -y git**
**$ git clone https://github.com/awslabs/ecs-demo-php-simple-app**

```

1.  切换到`ecs-demo-php-simple-app`文件夹：

```
**$ cd ecs-demo-php-simple-app**

```

1.  我们可以检查`Dockerfile`如下，以了解它将部署的 Web 应用程序：

```
**$ cat Dockerfile**

```

1.  使用 Dockerfile 构建容器镜像，然后将其推送到您的 Docker Hub 帐户。 Docker Hub 帐户是必需的，因为它可以通过指定容器名称来在 Amazon ECS 服务上部署容器：

```
**$ docker build -t my-dockerhub-username/amazon-ecs-sample.**

```

此处构建的镜像需要将`dockerhub-username`（无空格）作为第一个参数。

下图描述了黑客无法访问 Web 应用程序，因为请求通过代理容器进行过滤并且被阻止：

![在 AWS ECS 中保护容器](img/00042.jpeg)

1.  将 Docker 镜像上传到 Docker Hub 帐户：

```
**$ docker login**

```

1.  检查以确保您的登录成功：

```
**$ docker info**

```

1.  将您的镜像推送到 Docker Hub 帐户：

```
**$ docker push my-dockerhub-username/amazon-ecs-sample**

```

1.  创建示例 Web 应用程序 Docker 容器后，我们将创建代理容器，如果需要，还可以包含一些与安全相关的软件，以加强安全性。我们将使用定制的 Dockerfile 创建一个新的代理 Docker 容器，然后将镜像推送到您的 Docker Hub 帐户：

```
**$ mkdir proxy-container**
**$ cd proxy-container**
**$ nano Dockerfile**
**FROM ubuntu**
**RUN apt-get update && apt-get install -y nginx**
**COPY nginx.conf /etc/nginx/nginx.conf**
**RUN echo "daemon off;" >> /etc/nginx/nginx.conf**
**EXPOSE 80**
**CMD service nginx start**

```

在上一个 Dockerfile 中，我们使用了一个基本的 Ubuntu 镜像，并安装了 nginx，并将其暴露在 80 端口。

1.  接下来，我们将创建一个定制的`nginx.conf`，它将覆盖默认的`nginx.conf`，以确保反向代理配置正确：

```
**user www-data;**
**worker_processes 4;**
**pid /var/run/nginx.pid;**

**events {**
 **worker_connections 768;**
 **# multi_accept on;**
**}**

**http {**
 **server {**
 **listen           80;**

 **# Proxy pass to servlet container**
 **location / {**
 **proxy_pass      http://application-container:80;**
 **}**
 **}**
**}**

```

1.  构建代理 Docker 镜像并将构建的镜像推送到 Docker Hub 帐户：

```
**$ docker build -t my-dockerhub-username/proxy-image.**
**$ docker push my-dockerhub-username/proxy-image**

```

1.  可以通过转到 AWS 管理控制台（[`aws.amazon.com/console/`](https://aws.amazon.com/console/)）来部署 ECS 容器服务。

1.  在左侧边栏中单击“任务定义”，然后单击“创建新任务定义”。

1.  给你的任务定义起一个名字，比如`SecurityApp`。

1.  接下来，单击“添加容器”，并插入推送到 Docker Hub 帐户的代理 Web 容器的名称，以及应用程序 Web 容器的名称。使用“通过 JSON 配置”选项卡查看 JSON 的内容，以查看您创建的任务定义。它应该是这样的：

```
**Proxy-container:**
**Container Name: proxy-container**
**Image: username/proxy-image**
**Memory: 256**
**Port Mappings**
**Host port: 80**
**Container port: 80**
**Protocol: tcp**
**CPU: 256**
**Links: application-container**
**Application container:**
**Container Name: application-container**
**Image: username/amazon-ecs-sample**
**Memory: 256**
**CPU: 256**

```

单击“创建”按钮以部署应用程序。

1.  在左侧边栏中单击“集群”。如果默认集群不存在，则创建一个。

1.  启动一个 ECS 优化的 Amazon 机器映像（AMI），确保它具有公共 IP 地址和通往互联网的路径。

1.  当您的实例正在运行时，导航到 AWS 管理控制台的 ECS 部分，然后单击“集群”，然后单击“默认”。现在，我们应该能够在“ECS 实例”选项卡下看到我们的实例。

1.  从 AWS 管理控制台选项卡的左侧导航到任务定义，然后单击“运行任务”。

1.  在下一页上，确保集群设置为“默认”，任务数为“1”，然后单击“运行任务”。

1.  进程完成后，我们可以从挂起状态到绿色运行状态看到任务的状态。

1.  单击“ECS”选项卡，我们可以看到先前创建的容器实例。单击它，我们将获得有关其公共 IP 地址的信息。通过浏览器点击此公共 IP 地址，我们将能够看到我们的示例 PHP 应用程序。

# 理解 Docker 安全性 I—内核命名空间

命名空间提供了对内核全局系统资源的包装器，并使资源对于命名空间内的进程看起来像是有一个隔离的实例。全局资源更改对于相同命名空间中的进程是可见的，但对其他进程是不可见的。容器被认为是内核命名空间的一个很好的实现。

Docker 实现了以下命名空间：

+   pid 命名空间：用于进程隔离（PID—进程 ID）

+   net 命名空间：用于管理网络接口（NET—网络）

+   IPC 命名空间：用于管理对 IPC 资源（IPC—进程间通信）的访问

+   mnt 命名空间：用于管理挂载点（MNT—挂载）

+   **uts 命名空间**：用于隔离内核和版本标识（**UTS**—**Unix Time sharing System**）

在 libcontainer 中添加命名空间支持需要在 GoLang 的系统层中添加补丁（[`codereview.appspot.com/126190043/patch/140001/150001`](https://codereview.appspot.com/126190043/patch/140001/150001)<emphsis>src/syscall/exec_linux.go</emphsis>），以便可以维护新的数据结构用于 PID、用户 UID 等。

## pid 命名空间

pid 命名空间隔离了进程 ID 号空间；不同 pid 命名空间中的进程可以拥有相同的 pid。pid 命名空间允许容器提供功能，如暂停/恢复容器中的一组进程，并在容器内部的进程保持相同的 pid 的情况下将容器迁移到新主机。

新命名空间中的 PID 从 1 开始。内核需要配置标志`CONFIG_PID_NS`才能使命名空间工作。

pid 命名空间可以嵌套。每个 pid 命名空间都有一个父命名空间，除了初始（根）pid 命名空间。pid 命名空间的父命名空间是使用 clone 或 unshare 创建命名空间的进程的 pid 命名空间。pid 命名空间形成一棵树，所有命名空间最终都可以追溯到根命名空间，如下图所示：

![pid 命名空间](img/00043.jpeg)

## net 命名空间

net 命名空间提供了与网络相关的系统资源隔离。每个网络命名空间都有自己的网络设备、IP 地址、IP 路由表、`/proc/net`目录、端口号等。

网络命名空间使容器在网络方面变得有用：每个容器可以拥有自己的（虚拟）网络设备和绑定到每个命名空间端口号空间的应用程序；主机系统中适当的路由规则可以将网络数据包定向到与特定容器关联的网络设备。使用网络命名空间需要内核配置`CONFIG_NET_NS`选项（[`lwn.net/Articles/531114/`](https://lwn.net/Articles/531114/)）。

由于每个容器都有自己的网络命名空间，基本上意味着拥有自己的网络接口和路由表，net 命名空间也被 Docker 直接利用来隔离 IP 地址、端口号等。

### 基本网络命名空间管理

通过向`clone()`系统调用传递一个标志`CLONE_NEWNET`来创建网络命名空间。不过，从命令行来看，使用 IP 网络配置工具来设置和处理网络命名空间是很方便的：

```
**# ip netns add netns1**

```

这个命令创建了一个名为`netns1`的新网络命名空间。当 IP 工具创建网络命名空间时，它将在`/var/run/netns`下为其创建一个绑定挂载，这样即使在其中没有运行任何进程时，命名空间也会持续存在，并且便于对命名空间本身进行操作。由于网络命名空间通常需要大量配置才能准备好使用，这个特性将受到系统管理员的赞赏。

`ip netns exec`命令可用于在命名空间内运行网络管理命令：

```
**# ip netns exec netns1 ip link list**
**1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00**

```

这个命令列出了命名空间内可见的接口。可以使用以下命令删除网络命名空间：

```
**# ip netns delete netns1**

```

这个命令移除了指向给定网络命名空间的绑定挂载。然而，命名空间本身将持续存在，只要其中有任何进程在其中运行。

### 网络命名空间配置

新的网络命名空间将拥有一个环回设备，但没有其他网络设备。除了环回设备外，每个网络设备（物理或虚拟接口，桥接等）只能存在于单个网络命名空间中。此外，物理设备（连接到真实硬件的设备）不能被分配到除根之外的命名空间。相反，可以创建虚拟网络设备（例如虚拟以太网或 vEth）并分配给命名空间。这些虚拟设备允许命名空间内的进程通过网络进行通信；决定它们可以与谁通信的是配置、路由等。

创建时，新命名空间中的`lo`环回设备是关闭的，因此即使是环回的`ping`也会失败。

```
**# ip netns exec netns1 ping 127.0.0.1**
**connect: Network is unreachable**

```

在前面的命令中，我们可以看到由于 Docker 容器的网络命名空间存储在单独的位置，因此需要创建到`/var/run/netns`的符号链接，可以通过以下方式完成：

```
**# pid=`docker inspect -f '{{.State.Pid}}' $container_id`**
**# ln -s /proc/$pid/ns/net /var/run/netns/$container_id**

```

在这个例子中，通过启动该接口来实现，这将允许对环回地址进行 ping。

```
**# ip netns exec netns1 ip link set dev lo up**
**# ip netns exec netns1 ping 127.0.0.1**
 **PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.**
**64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.052 ms**
**64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.042 ms**
**64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.044 ms**
**64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.031 ms**
**64 bytes from 127.0.0.1: icmp_seq=5 ttl=64 time=0.042 ms**

```

这仍然不允许`netns1`和根命名空间之间的通信。为了实现这一点，需要创建和配置虚拟以太网设备。

```
**# ip link add veth0 type veth peer name veth1**
**# ip link set veth1 netns netns1**

```

第一条命令设置了一对连接的虚拟以太网设备。发送到`veth0`的数据包将被`veth1`接收，反之亦然。第二条命令将`veth1`分配给`netns1`命名空间。

```
**# ip netns exec netns1 ifconfig veth1 10.0.0.1/24 up**
**# ifconfig veth0 10.0.0.2/24 up**

```

然后，这两条命令为这两个设备设置了 IP 地址。

```
**# ping 10.0.0.1**
**# ip netns exec netns1 ping 10.0.0.2**

```

现在可以进行双向通信，就像之前的`ping`命令所示。

如前所述，命名空间不共享路由表或防火墙规则，运行`route`和`iptables -L`在`netns1`中将证明这一点：

```
**# ip netns exec netns1 route**
**Kernel IP routing table**
**Destination   Gateway    Genmask        Flags    Metric Ref    Use Iface**
**10.0.0.0         *      255.255.255.0     U        0  0  0       veth1**

**# ip netns exec netns1 iptables -L**
**Chain INPUT (policy ACCEPT)**
**target     prot opt source               destination**
**Chain FORWARD (policy ACCEPT)**
**target     prot opt source               destination**
**Chain OUTPUT (policy ACCEPT)**
**target     prot opt source               destination**

```

## 用户命名空间

用户命名空间允许用户和组 ID 在命名空间内进行映射。这意味着命名空间内的进程的用户 ID 和组 ID 可以与其在命名空间外的 ID 不同。一个进程在命名空间外可以具有非零用户 ID，同时在命名空间内可以具有零用户 ID。该进程在用户命名空间外进行操作时没有特权，但在命名空间内具有 root 特权。

### 创建新的用户命名空间

通过在调用`clone()`或`unshare()`时指定`CLONE_NEWUSER`标志来创建用户命名空间：

`clone()` 允许子进程与调用进程共享其执行上下文的部分，例如内存空间、文件描述符表和信号处理程序表。

`unshare()` 允许进程（或线程）取消与其他进程（或线程）共享的执行上下文的部分。当使用`fork()`或`vfork()`创建新进程时，执行上下文的一部分，例如挂载命名空间，会隐式共享。

如前所述，Docker 容器与 LXC 容器非常相似，因为为容器单独创建了一组命名空间和控制组。每个容器都有自己的网络堆栈和命名空间。除非容器没有特权访问权限，否则不允许访问其他主机的套接字或接口。如果将主机网络模式赋予容器，那么它才能访问主机端口和 IP 地址，这可能对主机上运行的其他程序造成潜在威胁。

如下例所示，在容器中使用`host`网络模式，并且能够访问所有主机桥接设备：

```
**docker run -it --net=host ubuntu /bin/bash**
**$ ifconfig**
**docker0   Link encap:Ethernet  HWaddr 02:42:1d:36:0d:0d**
 **inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0**
 **inet6 addr: fe80::42:1dff:fe36:d0d/64 Scope:Link**
 **UP BROADCAST MULTICAST  MTU:1500  Metric:1**
 **RX packets:24 errors:0 dropped:0 overruns:0 frame:0**
 **TX packets:38 errors:0 dropped:0 overruns:0 carrier:0**
 **collisions:0 txqueuelen:0**
 **RX bytes:1608 (1.6 KB)  TX bytes:5800 (5.8 KB)**

**eno16777736 Link encap:Ethernet  HWaddr 00:0c:29:02:b9:13**
 **inet addr:192.168.218.129  Bcast:192.168.218.255  Mask:255.255.255.0**
 **inet6 addr: fe80::20c:29ff:fe02:b913/64 Scope:Link**
 **UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1**
 **RX packets:4934 errors:0 dropped:0 overruns:0 frame:0**
 **TX packets:4544 errors:0 dropped:0 overruns:0 carrier:0**
 **collisions:0 txqueuelen:1000**
 **RX bytes:2909561 (2.9 MB)  TX bytes:577079 (577.0 KB)**

**$ docker ps -q | xargs docker inspect --format '{{ .Id }}: NetworkMode={{ .HostConfig.NetworkMode }}'**
**52afb14d08b9271bd96045bebd508325a2adff98dbef8c10c63294989441954d: NetworkMode=host**

```

在审核过程中，应该检查所有容器，默认情况下网络模式是否设置为`default`而不是`host`：

```
**$ docker ps -q | xargs docker inspect --format '{{ .Id }}: NetworkMode={{ .HostConfig.NetworkMode }}'**
**1aca7fe47882da0952702c383815fc650f24da2c94029b5ad8af165239b78968: NetworkMode=default**

```

每个 Docker 容器都连接到以太网桥，以便在容器之间提供互连性。它们可以相互 ping 以发送/接收 UDP 数据包并建立 TCP 连接，但如果有必要，可以进行限制。命名空间还提供了一种简单的隔离，限制了在其他容器中运行的进程以及主机的访问。

我们将使用以下`nsenter`命令行实用程序进入命名空间。它是 GitHub 上的一个开源项目，可在[`github.com/jpetazzo/nsenter`](https://github.com/jpetazzo/nsenter)上找到。

使用它，我们将尝试进入现有容器的命名空间，或者尝试生成一组新的命名空间。它与 Docker `exec`命令不同，因为`nsenter`不会进入 cgroups，这可以通过使用命名空间来逃避资源限制，从而为调试和外部审计带来潜在好处。

我们可以从 PyPI 安装`nsenter`（它需要 Python 3.4），并使用命令行实用程序连接到正在运行的容器：

```
**$ pip install nsenter**

```

使用以下命令替换 pid 为容器的 pid：

```
**$ sudo nsenter --net --target=PID /bin/ip a**
**1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default**
 **link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00**
 **inet 127.0.0.1/8 scope host lo**
 **valid_lft forever preferred_lft forever**
 **inet6 ::1/128 scope host**
 **valid_lft forever preferred_lft forever**
**14: eth0: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default**
 **link/ether 02:42:ac:11:00:06 brd ff:ff:ff:ff:ff:ff**
 **inet 172.17.0.6/16 scope global eth0**
 **valid_lft forever preferred_lft forever**
 **inet6 fe80::42:acff:fe11:6/64 scope link**
 **valid_lft forever preferred_lft forever**

```

我们可以使用`docker inspect`命令使其更加方便：

1.  首先启动一个新的 nginx 服务器：

```
**$ docker run -d --name=nginx -t nginx**

```

1.  然后获取容器的 pid：

```
**PID=$(docker inspect --format {{.State.Pid}} nginx)**

```

1.  连接到正在运行的 nginx 容器：

```
**$ nsenter --target $PID --uts --ipc --net –pid**

```

`docker-enter`也是可以用来进入容器并指定 shell 命令的包装器，如果没有指定命令，将调用一个 shell。如果需要在不执行其他命令行工具的情况下检查或操作容器，可以使用上下文管理器来实现：

```
**import subprocess**
**from nsenter import Namespace**
**with Namespace(mypid, 'net'):**
**# output network interfaces as seen from within the mypid's net NS:**
 **subprocess.check_output(['ip', 'a'])**

```

# 理解 Docker 安全 II - cgroups

在本节中，我们将看看 cgroups 如何构成容器隔离的基础。

## 定义 cgroups

控制组提供了一种将任务（进程）及其所有未来的子任务聚合/分区到分层组中的机制。

cgroup 将一组任务与子系统的参数关联起来。子系统本身是用于定义 cgroups 边界或为资源提供的资源控制器。

层次结构是一组以树状排列的 cgroups，系统中的每个任务都恰好位于层次结构中的一个 cgroup 中，并且一组子系统。

## 为什么需要 cgroups？

Linux 内核中有多个努力提供进程聚合，主要用于资源跟踪目的。

这些努力包括 cpusets、CKRM/ResGroups、UserBeanCounters 和虚拟服务器命名空间。所有这些都需要基本的进程分组/分区概念，新分叉的进程最终进入与其父进程相同的组（cgroup）。

内核 cgroup 补丁提供了必要的内核机制，以有效地实现这些组。它对系统快速路径的影响很小，并为特定子系统提供了钩子，例如 cpusets，以提供所需的附加行为。

## 手动创建一个 cgroup

在以下步骤中，我们将创建一个`cpuset`控制组：

```
**# mount -t tmpfs cgroup_root /sys/fs/cgroup**

```

`tmpfs`是一种将所有文件保存在虚拟内存中的文件系统。`tmpfs`中的所有内容都是临时的，即不会在硬盘上创建任何文件。如果卸载`tmpfs`实例，则其中存储的所有内容都会丢失：

```
**# mkdir /sys/fs/cgroup/cpuset**
**# mount -t cgroup -ocpuset cpuset /sys/fs/cgroup/cpuset**
**# cd /sys/fs/cgroup/cpuset**
**# mkdir Charlie**
**# cd Charlie**
**# ls**
**cgroup.clone_children  cpuset.cpu_exclusive  cpuset.mem_hardwall     cpuset.memory_spread_page  cpuset.sched_load_balance  tasks**
**cgroup.event_control   cpuset.cpus           cpuset.memory_migrate   cpuset.memory_spread_slab  cpuset.sched_relax_domain_level**
**cgroup.procs           cpuset.mem_exclusive  cpuset.memory_pressure  cpuset.mems                notify_on_release**

```

为此 cgroup 分配 CPU 和内存限制：

```
**# /bin/echo 2-3 > cpuset.cpus**
**# /bin/echo 0 > cpuset.mems**
**# /bin/echo $$ > tasks**

```

以下命令显示`/Charlie`作为 cpuset cgroup： 

```
**# cat /proc/self/cgroup**
**11:name=systemd:/user/1000.user/c2.session**
**10:hugetlb:/user/1000.user/c2.session**
**9:perf_event:/user/1000.user/c2.session**
**8:blkio:/user/1000.user/c2.session**
**7:freezer:/user/1000.user/c2.session**
**6:devices:/user/1000.user/c2.session**
**5:memory:/user/1000.user/c2.session**
**4:cpuacct:/user/1000.user/c2.session**
**3:cpu:/user/1000.user/c2.session**
**2:cpuset:/Charlie**

```

## 将进程附加到 cgroups

将进程 ID`PID{X}`添加到任务文件中，如下所示：

```
**# /bin/echo PID > tasks**

```

请注意，这是`PID`，而不是 PIDs。

您一次只能附加一个任务。如果有多个任务要附加，您必须一个接一个地执行：

```
**# /bin/echo PID1 > tasks**
**# /bin/echo PID2 > tasks**
**...**
**# /bin/echo PIDn > tasks**

```

通过回显`0`将当前 shell 任务附加：

```
**# echo 0 > tasks**

```

## Docker 和 cgroups

cgroups 作为 Docker 的 GitHub 存储库（[`github.com/opencontainers/runc/tree/master/libcontainer/cgroups`](https://github.com/opencontainers/runc/tree/master/libcontainer/cgroups)）下的 libcontainer 项目的一部分进行管理。有一个 cgroup 管理器，负责与内核中的 cgroup API 进行交互。

以下代码显示了管理器管理的生命周期事件：

```
**type Manager interface {**
 **// Apply cgroup configuration to the process with the specified pid**
 **Apply(pid int) error**
 **// Returns the PIDs inside the cgroup set**
 **GetPids() ([]int, error)**
 **// Returns statistics for the cgroup set**
 **GetStats() (*Stats, error)**
 **// Toggles the freezer cgroup according with specified state**
 **Freeze(state configs.FreezerState) error**
 **// Destroys the cgroup set**
 **Destroy() error**
 **// Paths maps cgroup subsystem to path at which it is mounted.**
 **// Cgroups specifies specific cgroup settings for the various subsystems**
 **// Returns cgroup paths to save in a state file and to be able to**
 **// restore the object later.**
 **GetPaths() map[string]string**
 **// Set the cgroup as configured.**
 **Set(container *configs.Config) error**
**}**

```

# 使用 AppArmor 保护 Docker 容器

AppArmor 是一种**强制访问控制**（**MAC**）系统，是内核增强功能，用于将程序限制在有限的资源集合中。AppArmor 的安全模型是将访问控制属性绑定到程序，而不是用户。

AppArmor 约束是通过加载到内核中的配置文件提供的，通常在启动时加载。AppArmor 配置文件可以处于两种模式之一：强制执行或投诉。

以强制执行模式加载的配置文件将导致强制执行配置文件中定义的策略，并报告策略违规尝试（通过 syslog 或 auditd）。

投诉模式下的配置文件不会强制执行策略，而是报告策略违规尝试。

AppArmor 与 Linux 上的其他一些 MAC 系统不同：它是基于路径的，允许混合强制和投诉模式配置文件，使用包含文件来简化开发，并且比其他流行的 MAC 系统具有更低的入门门槛。以下图显示了与应用程序相关联的 AppArmour 应用程序配置文件：

![使用 AppArmor 保护 Docker 容器](img/00044.jpeg)

AppArmor 是一项成熟的技术，最初出现在 Immunix 中，后来集成到 Ubuntu、Novell/SUSE 和 Mandriva 中。核心 AppArmor 功能从 Linux 内核 2.6.36 版本开始就已经在主线内核中；AppArmor、Ubuntu 和其他开发人员正在进行工作，将其他额外的 AppArmor 功能合并到主线内核中。

您可以在[`wiki.ubuntu.com/AppArmor`](https://wiki.ubuntu.com/AppArmor)找到有关 AppArmor 的更多信息。

## AppArmor 和 Docker

在 Docker 内运行的应用程序可以利用 AppArmor 来定义策略。这些配置文件可以手动创建，也可以使用一个名为 bane 的工具加载。

### 注意

在 Ubuntu 14.x 上，确保安装了 systemd 才能使以下命令生效。

以下步骤显示了如何使用这个工具：

1.  从 GitHub 下载 bane 项目：

```
**$ git clone https://github.com/jfrazelle/bane**

```

确保这是在您的 GOPATH 目录中完成的。例如，我们使用了`/home/ubuntu/go`，bane 源代码下载在`/home/Ubuntu/go/src/github.com/jfrazelle/bane`。

1.  安装 bane 编译所需的 toml 解析器：

```
**$ go get github.com/BurntSushi/toml**

```

1.  转到`/home/Ubuntu/go/src/github.com/jfrazelle/bane`目录并运行以下命令：

```
**$ go install**

```

1.  您将在`/home/Ubuntu/go/bin`中找到 bane 二进制文件。

1.  使用`.toml`文件创建配置文件：

```
**Name = "nginx-sample"**
**[Filesystem]**
**# read only paths for the container**
**ReadOnlyPaths = [**
 **"/bin/**",**
 **"/boot/**",**
 **"/dev/**",**
 **"/etc/**",**
 **…**
**]**
**AllowExec = [**
 **"/usr/sbin/nginx"**
**]**
**# denied executable files**
**DenyExec = [**
 **"/bin/dash",**
 **"/bin/sh",**
 **"/usr/bin/top"**
**]**

```

1.  执行 bane 加载配置文件。`sample.toml`是在`/home/Ubuntu/go/src/github.com/jfrazelle/bane`目录中的文件：

```
**$ sudo bane sample.toml**
**# Profile installed successfully you can now run the profile with # `docker run --security-opt="apparmor:docker-nginx-sample"`**

```

这个配置文件将使大量路径变为只读，并且只允许在我们将要创建的容器中执行 nginx。它禁用了 TOP、PING 等。

1.  一旦配置文件加载，您就可以创建一个 nginx 容器：

```
**$ docker run --security-opt="apparmor:docker-nginx-sample" -p 80:80 --rm -it nginx bash**

```

注意，如果 AppArmor 无法找到文件，将文件复制到`/etc/apparmor.d`目录并重新加载 AppArmour 配置文件：

```
**$ sudo invoke-rc.d apparmor reload**

```

使用 AppArmor 配置文件创建 nginx 容器：

```
**ubuntu@ubuntu:~/go/src/github.com$ docker run --security-opt="apparmor:docker-nginx-sample" -p 80:80 --rm -it nginx bash**
**root@84d617972e04:/# ping 8.8.8.8**
**ping: Lacking privilege for raw socket.**

```

以下图显示了容器中运行的 nginx 应用程序如何使用 AppArmour 应用程序配置文件：

![AppArmor and Docker](img/00045.jpeg)

## Docker 安全基准

以下教程展示了一些重要的准则，应遵循以在安全和生产环境中运行 Docker 容器。这是从 CIS Docker 安全基准[`benchmarks.cisecurity.org/tools2/docker/CIS_Docker_1.6_Benchmark_v1.0.0.pdf`](https://benchmarks.cisecurity.org/tools2/docker/CIS_Docker_1.6_Benchmark_v1.0.0.pdf)中引用的。

### 定期审计 Docker 守护程序

除了审计常规的 Linux 文件系统和系统调用外，还要审计 Docker 守护程序。Docker 守护程序以 root 权限运行。因此，有必要审计其活动和使用情况：

```
**$ apt-get install auditd**
**Reading package lists... Done**
**Building dependency tree**
**Reading state information... Done**
**The following extra packages will be installed:**
 **libauparse0**
**Suggested packages:**
 **audispd-plugins**
**The following NEW packages will be installed:**
 **auditd libauparse0**
**0 upgraded, 2 newly installed, 0 to remove and 50 not upgraded.**
**Processing triggers for libc-bin (2.21-0ubuntu4) ...**
**Processing triggers for ureadahead (0.100.0-19) ...**
**Processing triggers for systemd (225-1ubuntu9) ...**

```

如果存在审计日志文件，则删除：

```
**$ cd /etc/audit/**
**$ ls**
**audit.log**
**$ nano audit.log**
**$ rm -rf audit.log**

```

为 Docker 服务添加审计规则并审计 Docker 服务：

```
**$ nano audit.rules**
**-w /usr/bin/docker -k docker**
**$ service auditd restart**
**$ ausearch -k docker**
**<no matches>**
**$ docker ps**
**CONTAINER ID    IMAGE      COMMAND    CREATED    STATUS   PORTS     NAMES**
**$ ausearch -k docker**
**----**
**time->Fri Nov 27 02:29:50 2015**
**type=PROCTITLE msg=audit(1448620190.716:79): proctitle=646F636B6572007073**
**type=PATH msg=audit(1448620190.716:79): item=1 name="/lib64/ld-linux-x86-64.so.2" inode=398512 dev=08:01 mode=0100755 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL**
**type=PATH msg=audit(1448620190.716:79): item=0 name="/usr/bin/docker" inode=941134 dev=08:01 mode=0100755 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL**
**type=CWD msg=audit(1448620190.716:79):  cwd="/etc/audit"**
**type=EXECVE msg=audit(1448620190.716:79): argc=2 a0="docker" a1="ps"**
**type=SYSCALL msg=audit(1448620190.716:79): arch=c000003e syscall=59 success=yes exit=0 a0=ca1208 a1=c958c8 a2=c8**

```

### 为容器创建一个用户

目前，Docker 不支持将容器的 root 用户映射到主机上的非 root 用户。用户命名空间的支持将在未来版本中提供。这会导致严重的用户隔离问题。因此，强烈建议确保为容器创建一个非 root 用户，并使用该用户运行容器。

如我们在以下片段中所见，默认情况下，`centos` Docker 镜像的`user`字段为空，这意味着默认情况下容器在运行时将获得 root 用户，这应该避免：

```
**$ docker inspect centos**
**[**
 **{**
 **"Id": "e9fa5d3a0d0e19519e66af2dd8ad6903a7288de0e995b6eafbcb38aebf2b606d",**
 **"RepoTags": [**
 **"centos:latest"**
 **],**
 **"RepoDigests": [],**
 **"Parent": "c9853740aa059d078b868c4a91a069a0975fb2652e94cc1e237ef9b961afa572",**
 **"Comment": "",**
 **"Created": "2015-10-13T23:29:04.138328589Z",**
 **"Container": "eaa200e2e187340f0707085b9b4eab5658b13fd190af68c71a60f6283578172f",**
 **"ContainerConfig": {**
 **"Hostname": "7aa5783a47d5",**
 **"Domainname": "",**
 **"User": "",**
 **contd**

```

在构建 Docker 镜像时，可以在 Dockerfile 中提供`test`用户，即权限较低的用户，如以下片段所示：

```
**$ cd**
**$ mkdir test-container**
**$ cd test-container/**
**$ cat Dockerfile**
**FROM centos:latest**
**RUN useradd test**
**USER test**
**root@ubuntu:~/test-container# docker build -t vkohli .**
**Sending build context to Docker daemon 2.048 kB**
**Step 1 : FROM centos:latest**
 **---> e9fa5d3a0d0e**
**Step 2 : RUN useradd test**
 **---> Running in 0c726d186658**
 **---> 12041ebdfd3f**
**Removing intermediate container 0c726d186658**
**Step 3 : USER test**
 **---> Running in 86c5e0599c72**
 **---> af4ba8a0fec5**
**Removing intermediate container 86c5e0599c72**
**Successfully built af4ba8a0fec5**
**$ docker images | grep vkohli**
**vkohli    latest     af4ba8a0fec5      9 seconds ago     172.6 MB**

```

当我们启动 Docker 容器时，可以看到它获得了一个`test`用户，而`docker inspect`命令也显示默认用户为`test`：

```
**$ docker run -it vkohli /bin/bash**
**[test@2ff11ee54c5f /]$ whoami**
**test**
**[test@2ff11ee54c5f /]$ exit**
**$ docker inspect vkohli**
**[**
 **{**
 **"Id": "af4ba8a0fec558d68b4873e2a1a6d8a5ca05797e0bfbab0772bcedced15683ea",**
 **"RepoTags": [**
 **"vkohli:latest"**
 **],**
 **"RepoDigests": [],**
 **"Parent": "12041ebdfd3f38df3397a8961f82c225bddc56588e348761d3e252eec868d129",**
 **"Comment": "",**
 **"Created": "2015-11-27T14:10:49.206969614Z",**
 **"Container": "86c5e0599c72285983f3c5511fdec940f70cde171f1bfb53fab08854fe6d7b12",**
 **"ContainerConfig": {**
 **"Hostname": "7aa5783a47d5",**
 **"Domainname": "",**
 **"User": "test",**
 **Contd..**

```

### 不要在容器上挂载敏感主机系统目录

如果敏感目录以读写模式挂载，可能会对这些敏感目录内的文件进行更改。这些更改可能带来安全隐患或不必要的更改，可能使 Docker 主机处于受损状态。

如果在容器中挂载了`/run/systemd`敏感目录，那么我们实际上可以从容器本身关闭主机：

```
**$ docker run -ti -v /run/systemd:/run/systemd centos /bin/bash**
**[root@1aca7fe47882 /]# systemctl status docker**
**docker.service - Docker Application Container Engine**
 **Loaded: loaded (/lib/systemd/system/docker.service; enabled)**
 **Active: active (running) since Sun 2015-11-29 12:22:50 UTC; 21min ago**
 **Docs: https://docs.docker.com**
 **Main PID: 758**
 **CGroup: /system.slice/docker.service**
**[root@1aca7fe47882 /]# shutdown**

```

可以通过使用以下命令进行审计，该命令返回当前映射目录的列表以及每个容器实例是否以读写模式挂载：

```
**$ docker ps -q | xargs docker inspect --format '{{ .Id }}: Volumes={{ .Volumes }} VolumesRW={{ .VolumesRW }}'**

```

### 不要使用特权容器

Docker 支持添加和删除功能，允许使用非默认配置文件。这可能通过删除功能使 Docker 更安全，或者通过添加功能使其不太安全。因此建议除了容器进程明确需要的功能外，删除所有功能。

正如下所示，当我们在不使用特权模式的情况下运行容器时，我们无法更改内核参数，但是当我们使用`--privileged`标志在特权模式下运行容器时，可以轻松更改内核参数，这可能会导致安全漏洞。

```
**$ docker run -it centos /bin/bash**
**[root@7e1b1fa4fb89 /]#  sysctl -w net.ipv4.ip_forward=0**
**sysctl: setting key "net.ipv4.ip_forward": Read-only file system**
**$ docker run --privileged -it centos /bin/bash**
**[root@930aaa93b4e4 /]#  sysctl -a | wc -l**
**sysctl: reading key "net.ipv6.conf.all.stable_secret"**
**sysctl: reading key "net.ipv6.conf.default.stable_secret"**
**sysctl: reading key "net.ipv6.conf.eth0.stable_secret"**
**sysctl: reading key "net.ipv6.conf.lo.stable_secret"**
**638**
**[root@930aaa93b4e4 /]# sysctl -w net.ipv4.ip_forward=0**
**net.ipv4.ip_forward = 0**

```

因此，在审核时，必须确保所有容器的特权模式未设置为`true`。

```
**$ docker ps -q | xargs docker inspect --format '{{ .Id }}: Privileged={{ .HostConfig.Privileged }}'**
**930aaa93b4e44c0f647b53b3e934ce162fbd9ef1fd4ec82b826f55357f6fdf3a: Privileged=true**

```

# 总结

在本章中，我们深入探讨了 Docker 安全性，并概述了 cgroups 和内核命名空间。我们还讨论了文件系统和 Linux 功能的一些方面，容器利用这些功能来提供更多功能，例如特权容器，但代价是在威胁方面更容易暴露。我们还看到了如何在 AWS ECS（EC2 容器服务）中部署容器以在受限流量中使用代理容器来在安全环境中部署容器。AppArmor 还提供了内核增强功能，以将应用程序限制在有限的资源集上。利用它们对 Docker 容器的好处有助于在安全环境中部署它们。最后，我们快速了解了 Docker 安全基准和在生产环境中进行审核和 Docker 部署期间可以遵循的一些重要建议。

在下一章中，我们将学习使用各种工具在 Docker 网络中进行调优和故障排除。

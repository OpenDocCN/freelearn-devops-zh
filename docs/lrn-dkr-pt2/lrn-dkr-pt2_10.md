# 第十章：调试容器

调试一直是软件工程领域的艺术组成部分。各种软件构建模块个别以及集体都需要经过软件开发和测试专业人员深入而决定性的调查流程，以确保最终软件应用程序的安全性和安全性。由于 Docker 容器被认为是下一代关键运行时环境，用于使命关键的软件工作负载，因此对于容器、制作者和作曲家来说，进行容器的系统和明智的验证和验证是相关和至关重要的。

本章专门为技术人员撰写，旨在为他们提供所有正确和相关的信息，以便精心调试在容器内运行的应用程序和容器本身。在本章中，我们将从理论角度探讨作为容器运行的进程的进程隔离方面。Docker 容器在主机上以用户级进程运行，通常具有与操作系统提供的隔离级别相同的隔离级别。随着 Docker 1.5 的发布，许多调试工具可供使用，可以有效地用于调试应用程序。我们还将介绍主要的 Docker 调试工具，如 Docker `exec`、`stats`、`ps`、`top`、`events`和`logs`。最后，我们将介绍`nsenter`工具，以便登录到容器而无需运行**Secure Shell**（**SSH**）守护程序。

本章将涵盖的主题列表如下：

+   Docker 容器的进程级隔离

+   调试容器化应用程序

+   安装和使用`nsenter`

# Docker 容器的进程级隔离

在虚拟化范式中，hypervisor 模拟计算资源并提供一个虚拟化环境，称为 VM，用于在其上安装操作系统和应用程序。而在容器范式的情况下，单个系统（裸机或虚拟机）被有效地分区，以便同时运行多个服务而互不干扰。为了防止它们相互干扰，这些服务必须相互隔离，以防止它们占用对方的资源或产生依赖冲突（也称为依赖地狱）。Docker 容器技术基本上通过利用 Linux 内核构造（如命名空间和 cgroups，特别是命名空间）实现了进程级别的隔离。Linux 内核提供了以下五个强大的命名空间，用于将全局系统资源相互隔离。这些是用于隔离进程间通信资源的**进程间通信**（**IPC**）命名空间：

+   网络命名空间用于隔离网络资源，如网络设备、网络堆栈、端口号等

+   挂载命名空间隔离文件系统挂载点

+   PID 命名空间隔离进程标识号

+   用户命名空间用于隔离用户 ID 和组 ID

+   UTS 命名空间用于隔离主机名和 NIS 域名

当我们必须调试容器内运行的服务时，这些命名空间会增加额外的复杂性，我们将在下一章节中更详细地学习。

在本节中，我们将讨论 Docker 引擎如何通过一系列实际示例利用 Linux 命名空间提供进程隔离，其中之一列在此处：

1.  首先，通过使用`docker run`子命令以交互模式启动一个`ubuntu`容器，如下所示：

```
**$ sudo docker run -it --rm ubuntu /bin/bash**
**root@93f5d72c2f21:/#**

```

1.  继续在不同的终端中使用`docker inspect`子命令查找前面容器`93f5d72c2f21`的进程 ID：

```
**$ sudo docker inspect \**
 **--format "{{ .State.Pid }}" 93f5d72c2f21**
**2543**

```

显然，从前面的输出中，容器`93f5d72c2f21`的进程 ID 是`2543`。

1.  得到容器的进程 ID 后，让我们继续看看与容器关联的进程在 Docker 主机中的情况，使用`ps`命令：

```
**$ ps -fp 2543**
**UID        PID  PPID  C STIME TTY          TIME CMD**
**root      2543  6810  0 13:46 pts/7    00:00:00 /bin/bash**

```

很神奇，不是吗？我们启动了一个带有`/bin/bash`作为其命令的容器，我们在 Docker 主机中也有`/bin/bash`进程。

1.  让我们再进一步，使用`cat`命令在 Docker 主机中显示`/proc/2543/environ`文件：

```
**$ sudo cat -v /proc/2543/environ**
**PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin^@HOSTNAME=93f5d72c2f21^@TERM=xterm^@HOME=/root^@$**

```

在前面的输出中，`HOSTNAME=93f5d72c2f21`从其他环境变量中脱颖而出，因为`93f5d72c2f21`是容器的 ID，也是我们之前启动的容器的主机名。

1.  现在，让我们回到终端，我们正在运行交互式容器`93f5d72c2f21`，并使用`ps`命令列出该容器内运行的所有进程：

```
**root@93f5d72c2f21:/# ps -ef**
**UID    PID PPID C STIME TTY     TIME CMD**
**root     1   0 0 18:46 ?    00:00:00 /bin/bash**
**root    15   1 0 19:30 ?    00:00:00 ps -ef**

```

令人惊讶，不是吗？在容器内，`bin/bash`进程的进程 ID 为`1`，而在容器外，即 Docker 主机中，进程 ID 为`2543`。此外，**父进程 ID**（**PPID**）为`0`（零）。

在 Linux 世界中，每个系统只有一个 PID 为 1 且 PPID 为 0 的根进程，这是该系统完整进程树的根。Docker 框架巧妙地利用 Linux PID 命名空间来生成一个全新的进程树；因此，容器内运行的进程无法访问 Docker 主机的父进程。然而，Docker 主机可以完全查看 Docker 引擎生成的子 PID 命名空间。

网络命名空间确保所有容器在主机上拥有独立的网络接口。此外，每个容器都有自己的回环接口。每个容器使用自己的网络接口与外部世界通信。您会惊讶地知道，该命名空间不仅有自己的路由表，还有自己的 iptables、链和规则。本章的作者在他的主机上运行了三个容器。在这里，自然期望每个容器有三个网络接口。让我们运行`docker ps`命令：

```
**$ sudo docker ps**
**41668be6e513        docker-apache2:latest   "/bin/sh -c 'apachec** 
**069e73d4f63c        nginx:latest            "nginx -g '** 
**871da6a6cf43        ubuntu:14.04            "/bin/bash"** 

```

所以这里有三个接口，每个容器一个。让我们通过运行以下命令来获取它们的详细信息：

```
**$ ifconfig**
**veth2d99bd3 Link encap:Ethernet  HWaddr 42:b2:cc:a5:d8:f3**
 **inet6 addr: fe80::40b2:ccff:fea5:d8f3/64 Scope:Link**
 **UP BROADCAST RUNNING  MTU:9001  Metric:1**
**veth422c684 Link encap:Ethernet  HWaddr 02:84:ab:68:42:bf**
 **inet6 addr: fe80::84:abff:fe68:42bf/64 Scope:Link**
 **UP BROADCAST RUNNING  MTU:9001  Metric:1**
**vethc359aec Link encap:Ethernet  HWaddr 06:be:35:47:0a:c4**
 **inet6 addr: fe80::4be:35ff:fe47:ac4/64 Scope:Link**
 **UP BROADCAST RUNNING  MTU:9001  Metric:1**

```

挂载命名空间确保挂载的文件系统只能被同一命名空间内的进程访问。容器 A 无法看到容器 B 的挂载点。如果您想要检查您的挂载点，您需要首先使用`exec`命令（在下一节中描述），然后转到`/proc/mounts`：

```
**root@871da6a6cf43:/# cat /proc/mounts**
**rootfs / rootfs rw 0 0/dev/mapper/docker-202:1-149807 871da6a6cf4320f625d5c96cc24f657b7b231fe89774e09fc771b3684bf405fb / ext4 rw,relatime,discard,stripe=16,data=ordered 0 0 proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0** 

```

让我们运行一个带有挂载点的容器，作为**存储区域网络**（**SAN**）或**网络附加存储**（**NAS**）设备，并通过登录到容器来访问它。这是给您的一个练习。我在工作中的一个项目中实现了这一点。

这些容器/进程可以被隔离到其他命名空间中，包括用户、IPC 和 UTS。用户命名空间允许您在命名空间内拥有根权限，而不会将该特定访问权限授予命名空间外的进程。使用 IPC 命名空间隔离进程会为其提供自己的进程间通信资源，例如 System V IPC 和 POSIX 消息。UTS 命名空间隔离系统的*主机名*。

Docker 使用`clone`系统调用实现了这个命名空间。在主机上，您可以检查 Docker 为容器创建的命名空间（带有`pid 3728`）：

```
**$ sudo ls /proc/3728/ns/**
**ipc  mnt  net  pid  user  uts** 

```

在大多数 Docker 的工业部署中，人们广泛使用经过修补的 Linux 内核来满足特定需求。此外，一些公司已经修补了他们的内核，以将任意进程附加到现有的命名空间，因为他们认为这是部署、控制和编排容器最方便和可靠的方式。

## 控制组

Linux 容器依赖于控制组（cgroups），它们不仅跟踪进程组，还公开 CPU、内存和块 I/O 使用情况的指标。您还可以访问这些指标，并获取网络使用情况指标。控制组是 Linux 容器的另一个重要组件。控制组已经存在一段时间，并最初是在 Linux 内核代码 2.6.24 中合并的。它们确保每个 Docker 容器都将获得固定数量的内存、CPU 和磁盘 I/O，以便任何容器都无法在任何情况下使主机机器崩溃。控制组不会阻止访问一个容器，但它们对抵御一些**拒绝服务**（**DoS**）攻击至关重要。

在 Ubuntu 14.04 上，`cgroup`实现在`/sys/fs/cgroup`路径中。Docker 的内存信息可在`/sys/fs/cgroup/memory/docker/`路径下找到。

类似地，CPU 详细信息可以在`/sys/fs/cgroup/cpu/docker/`路径中找到。

让我们找出容器（`41668be6e513e845150abd2dd95dd574591912a7fda947f6744a0bfdb5cd9a85`）可以消耗的最大内存限制。

为此，您可以转到`cgroup`内存路径，并检查`memory.max.usage`文件：

```
**/sys/fs/cgroup/memory/docker/41668be6e513e845150abd2dd95dd574591912a7fda947f6744a0bfdb5cd9a85**
**$ cat memory.max_usage_in_bytes**
**13824000**

```

因此，默认情况下，任何容器只能使用最多 13.18 MB 的内存。

类似地，CPU 参数可以在以下路径中找到：

```
**/sys/fs/cgroup/cpu/docker/41668be6e513e845150abd2dd95dd574591912a7fda947f6744a0bfdb5cd9a85**

```

传统上，Docker 在容器内部只运行一个进程。因此，通常情况下，您会看到人们为 PHP、nginx 和 MySQL 分别运行三个容器。然而，这是一个谬论。您可以在单个容器内运行所有三个进程。

Docker 在不具备 root 权限的情况下，隔离了容器中运行的应用程序与底层主机的许多方面。然而，这种分离并不像虚拟机那样强大，虚拟机在 hypervisor 之上独立运行独立的操作系统实例，而不与底层操作系统共享内核。在同一主机上以容器化应用程序的形式运行具有不同安全配置文件的应用程序并不是一个好主意，但将不同的应用程序封装到容器化应用程序中具有安全性的好处，否则这些应用程序将直接在同一主机上运行。

# 调试容器化应用程序

计算机程序（软件）有时无法按预期行为。这是由于错误的代码或由于开发、测试和部署系统之间的环境变化。Docker 容器技术通过将所有应用程序依赖项容器化，尽可能消除开发、测试和部署之间的环境问题。尽管如此，由于错误的代码或内核行为的变化，仍可能出现异常，需要进行调试。调试是软件工程世界中最复杂的过程之一，在容器范式中变得更加复杂，因为涉及到隔离技术。在本节中，我们将学习使用 Docker 本机工具以及外部提供的工具来调试容器化应用程序的一些技巧和窍门。

最初，Docker 社区中的许多人单独开发了自己的调试工具，但后来 Docker 开始支持本机工具，如`exec`、`top`、`logs`、`events`等。在本节中，我们将深入探讨以下 Docker 工具：

+   `exec`

+   `ps`

+   `top`

+   `stats`

+   `events`

+   `logs`

## Docker exec 命令

`docker exec`命令为部署自己的 Web 服务器或在后台运行的其他应用程序的用户提供了非常需要的帮助。现在，不需要登录到容器中运行 SSH 守护程序。

首先，运行`docker ps -a`命令以获取容器 ID：

```
**$ sudo docker ps -a**
**b34019e5b5ee        nsinit:latest             "make local"** 
**a245253db38b        training/webapp:latest    "python app.py"** 

```

然后，运行`docker exec`命令以登录到容器中。

```
**$ sudo docker exec -it a245253db38b bash**
**root@a245253db38b:/opt/webapp#**

```

需要注意的是，`docker exec` 命令只能访问正在运行的容器，因此如果容器停止运行，则需要重新启动已停止的容器才能继续。`docker exec` 命令使用 Docker API 和 CLI 在目标容器中生成一个新进程。因此，如果你在目标容器内运行 `pe -aef` 命令，结果如下：

```
**# ps -aef**
**UID        PID  PPID  C STIME TTY          TIME CMD**
**root         1     0  0 Mar22 ?        00:00:53 python app.py**
**root        45     0  0 18:11 ?        00:00:00 bash**
**root        53    45  0 18:11 ?        00:00:00 ps -aef**

```

这里，`python app.y` 是已在目标容器中运行的应用程序，`docker exec` 命令已在容器内添加了 `bash` 进程。如果你运行 `kill -9 pid(45)`，你将自动退出容器。

如果你是一名热情的开发者，并且想增强 `exec` 功能，你可以参考 [`github.com/chris-rock/docker-exec`](https://github.com/chris-rock/docker-exec)。

建议仅将 `docker exec` 命令用于监视和诊断目的，我个人认为一个容器一个进程的概念是最佳实践之一。

## Docker ps 命令

`docker ps` 命令可在容器内部使用，用于查看进程的状态。这类似于 Linux 环境中的标准 `ps` 命令，*不*是我们在 Docker 主机上运行的 `docker ps` 命令。

此命令在 Docker 容器内运行：

```
**root@5562f2f29417:/# ps –s**
 **UID   PID   PENDING   BLOCKED   IGNORED    CAUGHT STAT TTY        TIME COMMAND**
 **0     1  00000000  00010000  00380004  4b817efb Ss   ?          0:00 /bin/bash**
 **0    33  00000000  00000000  00000000  73d3fef9 R+   ?          0:00 ps -s**
**root@5562f2f29417:/# ps -l**
**F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD**
**4 S     0     1     0  0  80   0 -  4541 wait   ?        00:00:00 bash**
**0 R     0    34     1  0  80   0 -  1783 -      ?        00:00:00 ps**
**root@5562f2f29417:/# ps -t**
 **PID TTY      STAT   TIME COMMAND**
 **1 ?        Ss     0:00 /bin/bash**
 **35 ?        R+     0:00 ps -t**
**root@5562f2f29417:/# ps -m**
 **PID TTY          TIME CMD**
 **1 ?        00:00:00 bash**
 **- -        00:00:00 -**
 **36 ?        00:00:00 ps**
 **- -        00:00:00 -**
**root@5562f2f29417:/# ps -a**
 **PID TTY          TIME CMD**
 **37 ?        00:00:00 ps**

```

使用 `ps --help <simple|list|output|threads|misc|all>` 或 `ps --help <s|l|o|t|m|a>` 获取额外的帮助文本。

## Docker top 命令

你可以使用以下命令从 Docker 主机机器上运行 `top` 命令：

```
**docker top [OPTIONS] CONTAINER [ps OPTIONS]**

```

这将列出容器的运行进程，而无需登录到容器中，如下所示：

```
**$ sudo docker top  a245253db38b**
**UID                 PID                 PPID                C**
**STIME               TTY                 TIME                CMD**
**root                5232                3585                0**
**Mar22               ?                   00:00:53            python app.py** 
**$ sudo docker top  a245253db38b  -aef**
**UID                 PID                 PPID                C**
**STIME               TTY                 TIME                CMD**
**root                5232                3585                0**
**Mar22               ?                   00:00:53            python app.py**

```

Docker `top` 命令提供有关 CPU、内存和交换使用情况的信息，如果你在 Docker 容器内运行它：

```
**root@a245253db38b:/opt/webapp# top**
**top - 19:35:03 up 25 days, 15:50,  0 users,  load average: 0.00, 0.01, 0.05**
**Tasks:   3 total,   1 running,   2 sleeping,   0 stopped,   0 zombie**
**Cpu(s):  0.0%us,  0.0%sy,  0.0%ni, 99.9%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st**
**Mem:   1016292k total,   789812k used,   226480k free,    83280k buffers**
**Swap:        0k total,        0k used,        0k free,   521972k cached**
 **PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND**
 **1 root      20   0 44780  10m 1280 S  0.0  1.1   0:53.69 python**
 **62 root      20   0 18040 1944 1492 S  0.0  0.2   0:00.01 bash**
 **77 root      20   0 17208 1164  948 R  0.0  0.1   0:00.00 top**

```

如果在容器内运行 `top` 命令时出现 `error - TERM environment variable not set` 错误，请执行以下步骤解决：

运行 `echo $TERM` 命令。你会得到 `dumb` 作为结果。然后运行以下命令：

```
**$ export TERM=dumb** 

```

这将解决错误。

## Docker stats 命令

Docker `stats` 命令使你能够从 Docker 主机机器上查看容器的内存、CPU 和网络使用情况，如下所示：

```
**$ sudo docker stats a245253db38b**
**CONTAINER           CPU %               MEM USAGE/LIMIT       MEM %**
 **NET I/O**
**a245253db38b        0.02%               16.37 MiB/992.5 MiB   1.65%**
 **3.818 KiB/2.43 KiB**

```

你也可以运行 `stats` 命令来查看多个容器的使用情况：

```
**$ sudo docker stats a245253db38b f71b26cee2f1** 

```

在最新的 Docker 1.5 版本中，Docker 为您提供了对容器统计信息的*只读*访问权限。这将简化容器的 CPU、内存、网络 IO 和块 IO。这有助于您选择资源限制，以及进行性能分析。Docker stats 实用程序仅为正在运行的容器提供这些资源使用详细信息。您可以使用端点 API 在[`docs.docker.com/reference/api/docker_remote_api_v1.17/#inspect-a-container`](https://docs.docker.com/reference/api/docker_remote_api_v1.17/#inspect-a-container)获取详细信息。

Docker stats 最初是从 Michael Crosby 的代码贡献中获取的，可以在[`github.com/crosbymichael`](https://github.com/crosbymichael)上访问。

## Docker 事件命令

Docker 容器将报告以下实时事件：`create`、`destroy`、`die`、`export`、`kill`、`omm`、`pause`、`restart`、`start`、`stop`和`unpause`。以下是一些示例，说明如何使用这些命令：

```
**$ sudo docker pause  a245253db38b**
**a245253db38b**
**$ sudo docker ps -a**
**a245253db38b        training/webapp:latest    "python app.py"        4 days ago         Up 4 days (Paused)       0.0.0.0:5000->5000/tcp   sad_sammet**
**$ sudo docker unpause  a245253db38b**
**a245253db38b**
**$ sudo docker ps -a**
**a245253db38b        training/webapp:latest    "python app.py"        4 days ago    Up 4 days        0.0.0.0:5000->5000/tcp   sad_sammet**

```

Docker 镜像还将报告取消标记和删除事件。

使用多个过滤器将被视为`AND`操作；例如，`--filter container= a245253db38b --filter event=start`将显示容器`a245253db38b`的事件和事件类型为 start 的事件。

目前，支持的过滤器有 container、event 和 image。

## Docker 日志命令

此命令获取容器的日志，而无需登录到容器中。它批量检索执行时存在的日志。这些日志是`STDOUT`和`STDERR`的输出。通用用法显示在`docker logs [OPTIONS] CONTAINER`中。

`–follow`选项将继续提供输出直到结束，`-t`将提供时间戳，`--tail= <number of lines>`将显示容器日志消息的行数：

```
**$ sudo docker logs a245253db38b**
 *** Running on http://0.0.0.0:5000/**
**172.17.42.1 - - [22/Mar/2015 06:04:23] "GET / HTTP/1.1" 200 -**
**172.17.42.1 - - [24/Mar/2015 13:43:32] "GET / HTTP/1.1" 200 -**
**$**
**$ sudo docker logs -t a245253db38b**
**2015-03-22T05:03:16.866547111Z  * Running on http://0.0.0.0:5000/**
**2015-03-22T06:04:23.349691099Z 172.17.42.1 - - [22/Mar/2015 06:04:23] "GET / HTTP/1.1" 200 -**
**2015-03-24T13:43:32.754295010Z 172.17.42.1 - - [24/Mar/2015 13:43:32] "GET / HTTP/1.1" 200 -**

```

我们还在第二章和第六章中使用了`docker logs`实用程序，以查看我们的容器的日志。

# 安装和使用 nsenter

在任何商业 Docker 部署中，您可能会使用各种容器，如 Web 应用程序、数据库等。但是，您需要访问这些容器以修改配置或调试/排除故障。这个问题的一个简单解决方案是在每个容器中运行一个 SSH 服务器。这不是一个访问机器的好方法，因为会带来意想不到的安全影响。然而，如果您在 IBM、戴尔、惠普等世界一流的 IT 公司工作，您的安全合规人员绝不会允许您使用 SSH 连接到机器。

所以，这就是解决方案。`nsenter`工具为您提供了登录到容器的访问权限。请注意，`nsenter`将首先作为 Docker 容器部署。使用部署的`nsenter`，您可以访问您的容器。按照以下步骤进行：

1.  让我们运行一个简单的 Web 应用程序作为一个容器：

```
**$ sudo docker run -d -p 5000:5000 training/webapp python app.py**
**------------------------**
**a245253db38b626b8ac4a05575aa704374d0a3c25a392e0f4f562df92bb98d74**

```

1.  测试 Web 容器：

```
**$ curl localhost:5000**
**Hello world!**

```

1.  安装`nsenter`并将其作为一个容器运行：

```
**$ sudo docker run -v /usr/local/bin:/target jpetazzo/nsenter**

```

现在，`nsenter`作为一个容器正在运行。

1.  使用 nsenter 容器登录到我们在步骤 1 中创建的容器（`a245253db38b`）。

运行以下命令以获取`PID`值：

```
**$ PID=$(sudo docker inspect --format {{.State.Pid}} a245253db38b)**

```

1.  现在，访问 Web 容器：

```
**$ sudo nsenter --target $PID --mount --uts --ipc --net --pid**
**root@a245253db38b:/#**

```

然后，您可以登录并开始访问您的容器。以这种方式访问您的容器将使您的安全和合规专业人员感到满意，他们会感到放松。

自 Docker 1.3 以来，Docker exec 是一个支持的工具，用于登录到容器中。

`nsenter`工具不进入 cgroups，因此规避了资源限制。这样做的潜在好处是调试和外部审计，但对于远程访问，`docker exec`是当前推荐的方法。

`nsenter`工具仅在 Intel 64 位平台上进行测试。您不能在要访问的容器内运行`nsenter`，因此只能在主机上运行`nsenter`。通过在主机上运行`nsenter`，您可以访问该主机上的所有容器。此外，您不能使用在特定主机 A 上运行的`nsenter`来访问主机 B 上的容器。

# 总结

Docker 利用 Linux 容器技术（如 LXC 和现在的`libcontainer`）为您提供容器的隔离。Libcontainer 是 Docker 在 Go 编程语言中的自己的实现，用于访问内核命名空间和控制组。这个命名空间用于进程级别的隔离，而控制组用于限制运行容器的资源使用。由于容器作为独立进程直接在 Linux 内核上运行，因此**通常可用**（**GA**）的调试工具不足以在容器内部工作以调试容器化的进程。Docker 现在为您提供了丰富的工具集，以有效地调试容器，以及容器内部的进程。Docker `exec` 将允许您登录到容器，而无需在容器中运行 SSH 守护程序。

Docker `stats` 提供有关容器内存和 CPU 使用情况的信息。Docker `events` 报告事件，比如创建、销毁、杀死等。同样，Docker `logs` 从容器中获取日志，而无需登录到容器中。

调试是可以用来制定其他安全漏洞和漏洞的基础。因此，下一章将详细阐述 Docker 容器的可能安全威胁，以及如何通过各种安全方法、自动化工具、最佳实践、关键指南和指标来抑制这些威胁。

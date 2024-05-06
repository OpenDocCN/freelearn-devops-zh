# 第八章：容器

现在我们对镜像有了一些了解，是时候进入容器了。由于这是一本关于 Docker 的书，我们将专门讨论 Docker 容器。然而，Docker 一直在努力实现由开放容器倡议（OCI）在 https://www.opencontainers.org 发布的镜像和容器规范。这意味着你在这里学到的很多东西将适用于其他符合 OCI 标准的容器运行时。

我们将把本章分为通常的三个部分：

+   简而言之

+   深入探讨

+   命令

让我们去学习关于容器的知识吧！

### Docker 容器-简而言之

容器是镜像的运行时实例。就像我们可以从虚拟机模板启动虚拟机（VM）一样，我们可以从单个镜像启动一个或多个容器。容器和虚拟机之间的主要区别在于容器更快速和更轻量级-容器不像虚拟机一样运行完整的操作系统，而是与它们运行的主机共享操作系统/内核。

图 7.1 显示了一个 Docker 镜像被用来启动多个 Docker 容器。

![图 7.1](img/figure7-1.png)

图 7.1

启动容器的最简单方法是使用`docker container run`命令。该命令可以带很多参数，但在其最基本的形式中，你告诉它要使用的镜像和要运行的应用程序：`docker container run <image> <app>`。下一个命令将启动一个运行 Bash shell 的 Ubuntu Linux 容器：`docker container run -it ubuntu /bin/bash`。要启动一个运行 PowerShell 应用程序的 Windows 容器，你可以这样做：`docker container run -it microsoft/powershell:nanoserver pwsh.exe`。

`-it`标志将把你当前的终端窗口连接到容器的 shell。

容器运行直到它们执行的应用退出。在上面的两个例子中，Linux 容器将在 Bash shell 退出时退出，而 Windows 容器将在 PowerShell 进程终止时退出。

一个非常简单的演示方法是启动一个新的容器，并告诉它运行 sleep 命令 10 秒钟。容器将启动，运行 10 秒钟然后退出。如果你从 Linux 主机（或在 Linux 容器模式下运行的 Windows 主机）运行以下命令，你的 shell 将附加到容器的 shell 上 10 秒钟，然后退出：`docker container run alpine:latest sleep 10`。你可以用以下命令在 Windows 容器中做同样的事情：`docker container run microsoft/powershell:nanoserver Start-Sleep -s 10`。

您可以使用`docker container stop`命令手动停止容器，然后使用`docker container start`重新启动它。要永久删除容器，必须使用`docker container rm`显式删除它。

这就是电梯推介！现在让我们深入了解细节...

### Docker 容器-深入探讨

我们将在这里首先介绍容器和虚拟机之间的基本区别。目前主要是理论，但这很重要。在此过程中，我们将指出容器模型相对于虚拟机模型的潜在优势。

> **注意：**作为作者，在我们继续之前，我要说这个。我们很多人对我们所做的事情和我们拥有的技能都很热情。我记得*大型 Unix*的人抵制 Linux 的崛起。你可能也记得同样的事情。你可能还记得人们试图抵制 VMware 和 VM 巨头。在这两种情况下，“抵抗是徒劳的”。在本节中，我将强调容器模型相对于 VM 模型的一些优势。但我猜你们中的很多人都是 VM 专家，对 VM 生态系统投入了很多。我猜你们中的一两个人可能想和我争论我说的一些事情。所以让我明白一点...我是个大个子，我会在肉搏战中打败你 :-D 只是开玩笑。但我不是想摧毁你的帝国或者说你的孩子丑陋！我是想帮助你。我写这本书的整个原因就是为了帮助你开始使用 Docker 和容器！

我们开始吧。

#### 容器 vs 虚拟机

容器和虚拟机都需要主机来运行。这可以是从您的笔记本电脑，到您数据中心的裸机服务器，一直到公共云实例的任何东西。在这个例子中，我们假设需要在单个物理服务器上运行 4 个业务应用程序。

在虚拟机模型中，物理服务器启动并引导虚拟机监视器（我们跳过 BIOS 和引导加载程序代码等）。一旦虚拟机监视器启动，它就会占用系统上的所有物理资源，如 CPU、RAM、存储和 NIC。然后，虚拟机监视器将这些硬件资源划分为看起来、闻起来和感觉起来与真实物品完全相同的虚拟版本。然后将它们打包成一个名为虚拟机（VM）的软件构造。然后我们将这些 VM 安装操作系统和应用程序。我们说我们有一个单独的物理服务器，需要运行 4 个应用程序，所以我们会创建 4 个 VM，安装 4 个操作系统，然后安装 4 个应用程序。完成后，它看起来有点像图 7.2。

![图 7.2](img/figure7-2.png)

图 7.2

容器模型中有些不同。

当服务器启动时，您选择的操作系统引导。在 Docker 世界中，这可以是 Linux，或者具有对其内核中容器基元支持的现代 Windows 版本。与虚拟机模型类似，操作系统占用所有硬件资源。在操作系统之上，我们安装了一个名为 Docker 的容器引擎。容器引擎然后获取**操作系统资源**，如*进程树*、*文件系统*和*网络堆栈*，并将它们划分为安全隔离的构造，称为*容器*。每个容器看起来、闻起来和感觉起来就像一个真正的操作系统。在每个*容器*内部，我们可以运行一个应用程序。与之前一样，我们假设有一个单独的物理服务器，需要运行 4 个应用程序。因此，我们会划分出 4 个容器，并在每个容器内运行一个应用程序。这在图 7.3 中显示。

![图 7.3](img/figure7-3.png)

图 7.3

在高层次上，我们可以说虚拟机监视器执行**硬件虚拟化** - 它们将物理硬件资源划分为虚拟版本。另一方面，容器执行**操作系统虚拟化** - 它们将操作系统资源划分为虚拟版本。

#### 虚拟机的开销

让我们在刚才讨论的基础上深入探讨虚拟机监视器模型的一个主要问题。

我们最初只有一个物理服务器，需要运行 4 个业务应用程序。在两种模型中，我们安装了操作系统或者虚拟机监视器（一种针对虚拟机高度优化的操作系统）。到目前为止，这两种模型几乎是相同的。但这就是相似之处的尽头。

然后，虚拟机模型将低级硬件资源划分为虚拟机。每个虚拟机都是一个包含虚拟 CPU、虚拟 RAM、虚拟磁盘等的软件构造。因此，每个虚拟机都需要自己的操作系统来声明、初始化和管理所有这些虚拟资源。不幸的是，每个操作系统都带有自己的一套负担和开销。例如，每个操作系统都会消耗一部分 CPU、一部分 RAM、一部分存储空间等。大多数还需要自己的许可证，以及人员和基础设施来打补丁和升级它们。每个操作系统还呈现出相当大的攻击面。我们经常将所有这些称为“操作系统税”或“虚拟机税”——你安装的每个操作系统都会消耗资源！

容器模型在主机操作系统中运行一个单一的内核。可以在单个主机上运行数十甚至数百个容器，每个容器共享同一个操作系统/内核。这意味着一个单一的操作系统消耗 CPU、RAM 和存储空间。一个需要许可证的单一操作系统。一个需要升级和打补丁的单一操作系统。以及一个呈现攻击面的单一操作系统内核。总而言之，只有一个操作系统税账单！

在我们的例子中，这可能看起来不是很多，因为一个单独的服务器需要运行 4 个业务应用程序。但是当我们谈论数百或数千个应用程序时，这可能会改变游戏规则。

考虑的另一件事是启动时间。因为容器不是一个完整的操作系统，所以它的启动速度比虚拟机快得多。记住，容器内部没有需要定位、解压和初始化的内核，更不用说与正常内核引导相关的所有硬件枚举和初始化。启动容器时不需要任何这些！在操作系统级别下，单个共享的内核已经启动了！最终结果是，容器可以在不到一秒的时间内启动。唯一影响容器启动时间的是运行的应用程序启动所需的时间。

所有这些都导致容器模型比虚拟机模型更精简和高效。我们可以在更少的资源上运行更多的应用程序，更快地启动它们，并且在许可和管理成本上支付更少，同时对黑暗面呈现出更少的攻击面。这有什么不好的呢！

有了这个理论，让我们来玩一下一些容器。

#### 运行容器

要跟随这些示例，您需要一个可用的 Docker 主机。对于大多数命令，它是 Linux 还是 Windows 都没有关系。

#### 检查 Docker 守护程序

我登录到 Docker 主机时，总是首先检查 Docker 是否正在运行。

```
$ docker version
Client:
 Version:      `17`.05.0-ce
 API version:  `1`.29
 Go version:   go1.7.5
 Git commit:   89658be
 Built:        Thu May  `4` `22`:10:54 `2017`
 OS/Arch:      linux/amd64

Server:
 Version:      `17`.05.0-ce
 API version:  `1`.29 `(`minimum version `1`.12`)`
 Go version:   go1.7.5
 Git commit:   89658be
 Built:        Thu May  `4` `22`:10:54 `2017`
 OS/Arch:      linux/amd64
 Experimental: `false` 
```

只要在 `Client` 和 `Server` 部分得到响应，你就可以继续。如果在 `Server` 部分得到错误代码，很可能是 docker 守护程序（服务器）没有运行，或者你的用户账户没有权限访问它。

如果你正在运行 Linux，并且你的用户账户没有权限访问守护程序，你需要确保它是本地 `docker` Unix 组的成员。如果不是，你可以用 `usermod -aG docker <user>` 添加它，然后你需要注销并重新登录到你的 shell 以使更改生效。

如果你的用户账户已经是本地 `docker` 组的成员，问题可能是 Docker 守护程序没有运行。要检查 Docker 守护程序的状态，请根据 Docker 主机的操作系统运行以下命令之一。

```
//Run this command on Linux systems not using Systemd
$ service docker status
docker start/running, process 29393

//Run this command on Linux systems that are using Systemd
$ systemctl is-active docker
active

//Run this command on Windows Server 2016 systems from a PowerShell window
> Get-Service docker

Status    Name      DisplayName
------    ----      -----------
Running   Docker    docker 
```

如果 Docker 守护程序正在运行，你可以继续。

#### 启动一个简单的容器

启动容器的最简单方法是使用 `docker container run` 命令。

以下命令启动一个简单的容器，将运行 Ubuntu Linux 的容器化版本。

```
`$` `docker` `container` `run` `-``it` `ubuntu``:``latest` `/``bin``/``bash`
`Unable` `to` `find` `image` `'``ubuntu``:``latest``'` `locally`
`latest``:` `Pulling` `from` `library``/``ubuntu`
`952132``ac251a``:` `Pull` `complete`
`82659f8f``1``b76``:` `Pull` `complete`
`c19118ca682d``:` `Pull` `complete`
`8296858250f``e``:` `Pull` `complete`
`24e0251``a0e2c``:` `Pull` `complete`
`Digest``:` `sha256``:``f4691c96e6bbaa99d9``...``e95a60369c506dd6e6f6ab`
`Status``:` `Downloaded` `newer` `image` `for` `ubuntu``:``latest`
`root``@3027``eb644874``:``/``#` 
```

Windows 的一个例子可能是

```
docker container run -it microsoft/powershell:nanoserver pwsh.exe 
```

命令的格式基本上是 `docker container run <options> <image>:<tag> <app>`。

让我们分解一下这个命令。

我们从 `docker container run` 开始，这是启动新容器的标准命令。然后我们使用了 `-it` 标志使容器交互，并将其附加到我们的终端。接下来，我们告诉它使用 `ubuntu:latest` 或 `microsoft/powershell:nanoserver` 镜像。最后，我们告诉它在 Linux 示例中运行 Bash shell，在 Windows 示例中运行 PowerShell 应用程序。

当我们按下 `Return` 键时，Docker 客户端会向 Docker 守护程序发出适当的 API 调用。Docker 守护程序接受了命令，并搜索 Docker 主机的本地缓存，看看它是否已经有所请求的镜像的副本。在所引用的例子中，它没有，所以它去 Docker Hub 看看是否能在那里找到。它找到了，所以它在本地 *拉取* 并将其存储在本地缓存中。

> **注意：** 在标准的 Linux 安装中，Docker 守护程序在本地 IPC/Unix 套接字`/var/run/docker.sock`上实现 Docker 远程 API。在 Windows 上，它在`npipe:////./pipe/docker_engine`上监听一个命名管道。也可以配置 Docker 客户端和守护程序通过网络进行通信。Docker 的默认非 TLS 网络端口是 2375，默认的 TLS 端口是 2376。

一旦镜像被拉取，守护程序就会创建容器并在其中执行指定的应用程序。

如果你仔细看，你会发现你的 shell 提示已经改变，你现在在容器内部。在所引用的例子中，shell 提示已经改变为`root@3027eb644874:/#`。`@`后面的长数字是容器唯一 ID 的前 12 个字符。

尝试在容器内执行一些基本命令。你可能会注意到一些命令无法使用。这是因为我们使用的镜像，就像几乎所有的容器镜像一样，都是针对容器高度优化的。这意味着它们并没有安装所有正常的命令和软件包。下面的例子展示了一些命令 - 一个成功，另一个失败。

```
`root``@3027``eb644874``:``/``#` `ls` `-``l`
`total` `64`
`drwxr``-``xr``-``x`   `2` `root` `root` `4096` `Aug` `19` `00``:``50` `bin`
`drwxr``-``xr``-``x`   `2` `root` `root` `4096` `Apr` `12` `20``:``14` `boot`
`drwxr``-``xr``-``x`   `5` `root` `root`  `380` `Sep` `13` `00``:``47` `dev`
`drwxr``-``xr``-``x`  `45` `root` `root` `4096` `Sep` `13` `00``:``47` `etc`
`drwxr``-``xr``-``x`   `2` `root` `root` `4096` `Apr` `12` `20``:``14` `home`
`drwxr``-``xr``-``x`   `8` `root` `root` `4096` `Sep` `13`  `2015` `lib`
`drwxr``-``xr``-``x`   `2` `root` `root` `4096` `Aug` `19` `00``:``50` `lib64`
`drwxr``-``xr``-``x`   `2` `root` `root` `4096` `Aug` `19` `00``:``50` `media`
`drwxr``-``xr``-``x`   `2` `root` `root` `4096` `Aug` `19` `00``:``50` `mnt`
`drwxr``-``xr``-``x`   `2` `root` `root` `4096` `Aug` `19` `00``:``50` `opt`
`dr``-``xr``-``xr``-``x` `129` `root` `root`    `0` `Sep` `13` `00``:``47` `proc`
`drwx``------`   `2` `root` `root` `4096` `Aug` `19` `00``:``50` `root`
`drwxr``-``xr``-``x`   `6` `root` `root` `4096` `Aug` `26` `18``:``50` `run`
`drwxr``-``xr``-``x`   `2` `root` `root` `4096` `Aug` `26` `18``:``50` `sbin`
`drwxr``-``xr``-``x`   `2` `root` `root` `4096` `Aug` `19` `00``:``50` `srv`
`dr``-``xr``-``xr``-``x`  `13` `root` `root`    `0` `Sep` `13` `00``:``47` `sys`
`drwxrwxrwt`   `2` `root` `root` `4096` `Aug` `19` `00``:``50` `tmp`
`drwxr``-``xr``-``x`  `11` `root` `root` `4096` `Aug` `26` `18``:``50` `usr`
`drwxr``-``xr``-``x`  `13` `root` `root` `4096` `Aug` `26` `18``:``50` `var`

`root``@3027``eb644874``:``/``#` `ping` `www``.``docker``.``com`
`bash``:` `ping``:` `command` `not` `found`
`root``@3027``eb644874``:``/``#` 
```

`如上所示，`ping`实用程序不包含在官方 Ubuntu 镜像中。

#### 容器进程

当我们在上一节中启动 Ubuntu 容器时，我们告诉它运行 Bash shell（`/bin/bash`）。这使得 Bash shell 成为**容器内唯一运行的进程**。你可以通过在容器内运行`ps -elf`来看到这一点。

```
`root``@3027``eb644874``:``/``#` `ps` `-``elf`
`F` `S` `UID`   `PID`  `PPID`   `NI` `ADDR` `SZ` `WCHAN`  `STIME` `TTY`     `TIME`      `CMD`
`4` `S` `root`    `1`     `0`    `0` `-`  `4558` `wait`   `00``:``47` `?`     `00``:``00``:``00`  `/``bin``/``bash`
`0` `R` `root`   `11`     `1`    `0` `-`  `8604` `-`      `00``:``52` `?`     `00``:``00``:``00`  `ps` `-``elf` 
```

`虽然在上面的输出中看起来有两个进程在运行，但实际上并没有。列表中的第一个进程，PID 为 1，是我们告诉容器运行的 Bash shell。第二个进程是我们运行的`ps -elf`命令来生成列表。这是一个短暂的进程，在输出显示时已经退出。长话短说，这个容器正在运行一个单一的进程 - `/bin/bash`。

> **注意：** Windows 容器略有不同，通常会运行相当多的进程。

这意味着如果你输入`exit`来退出 Bash shell，容器也会退出（终止）。原因是容器不能在没有运行的进程的情况下存在 - 终止 Bash shell 会终止容器的唯一进程，导致容器也被终止。这对 Windows 容器也是适用的 - **终止容器中的主进程也会终止容器**。

按下`Ctrl-PQ`退出容器而不终止它。这样做会将你放回 Docker 主机的 shell，并将容器保持在后台运行。你可以使用`docker container ls`命令查看系统上正在运行的容器列表。

```
$ docker container ls
CNTNR ID  IMAGE          COMMAND    CREATED  STATUS    NAMES
`302`...74  ubuntu:latest  /bin/bash  `6` mins   Up 6mins  sick_montalcini 
```

重要的是要理解，这个容器仍在运行，你可以使用`docker container exec`命令重新连接你的终端。

```
`$` `docker` `container` `exec` `-``it` `3027``eb644874` `bash`
`root``@3027``eb644874``:``/``#` 
```

重新连接到 Windows Nano Server PowerShell 容器的命令将是`docker container exec -it <container-name-or-ID> pwsh.exe`。

如您所见，shell 提示已经改回到容器。如果再次运行`ps`命令，您现在将看到**两个** Bash 或 PowerShell 进程。这是因为`docker container exec`命令创建了一个新的 Bash 或 PowerShell 进程并附加到其中。这意味着在这个 shell 中输入`exit`不会终止容器，因为原始的 Bash 或 PowerShell 进程将继续运行。

输入`exit`离开容器，并使用`docker container ps`验证它仍在运行。它仍在运行。

如果您正在自己的 Docker 主机上跟着示例操作，您应该使用以下两个命令停止并删除容器（您需要替换您的容器的 ID）。

```
$ docker container stop 3027eb64487
3027eb64487

$ docker container rm 3027eb64487
3027eb64487 
```

在前面的示例中启动的容器将不再存在于您的系统中。

#### 容器生命周期

普遍的误解是容器无法持久保存数据。它们可以！

人们认为容器不适合持久工作负载或持久数据的一个很大的原因是因为它们在非持久性工作上表现得很好。但擅长一件事并不意味着你不能做其他事情。很多虚拟机管理员会记得微软和甲骨文这样的公司告诉你他们的应用程序不能在虚拟机内运行，或者至少如果你这样做他们不会支持你。我想知道我们是否在容器化的过程中看到了类似的情况——是否有人试图保护他们认为受到容器威胁的持久工作负载帝国？

在本节中，我们将看一下容器的生命周期——从诞生，工作和休假，到最终的死亡。

我们已经看到如何使用`docker container run`命令启动容器。让我们启动另一个，以便我们可以完整地了解其生命周期。以下示例将来自运行 Ubuntu 容器的 Linux Docker 主机。但是，所有示例都将适用于我们在先前示例中使用的 Windows PowerShell 容器 - 尽管您将不得不用其等效的 Windows 命令替换 Linux 命令。

```
`$` `docker` `container` `run` `--``name` `percy` `-``it` `ubuntu``:``latest` `/``bin``/``bash`
`root``@9``cb2d2fd1d65``:``/``#` 
```

这就是我们创建的容器，我们将其命名为“percy”以表示持久:-S

现在让我们通过向其写入一些数据来让其工作。

在新容器的 shell 中，按照以下步骤将一些数据写入`tmp`目录中的新文件，并验证写入操作是否成功。

```
`root``@9``cb2d2fd1d65``:``/``#` `cd` `tmp`

`root``@9``cb2d2fd1d65``:``/``tmp``#` `ls` `-``l`
`total` `0`

`root``@9``cb2d2fd1d65``:``/``tmp``#` `echo` `"DevOps FTW"` `>` `newfile`

`root``@9``cb2d2fd1d65``:``/``tmp``#` `ls` `-``l`
`total` `4`
`-``rw``-``r``--``r``--` `1` `root` `root` `14` `May` `23` `11``:``22` `newfile`

`root``@9``cb2d2fd1d65``:``/``tmp``#` `cat` `newfile`
`DevOps` `FTW` 
```

按`Ctrl-PQ`退出容器而不杀死它。

现在使用`docker container stop`命令停止容器并将其放在*休假*中。

```
$ docker container stop percy
percy 
```

您可以使用`docker container stop`命令的容器名称或 ID。格式为`docker container stop <container-id 或 container-name>`。

现在运行`docker container ls`命令以列出所有正在运行的容器。

```
$ docker container ls
CONTAINER ID   IMAGE   COMMAND   CREATED  STATUS  PORTS   NAMES 
```

容器未在上面的输出中列出，因为您使用`docker container stop`命令将其置于停止状态。再次运行相同的命令，只是这次添加`-a`标志以显示所有容器，包括那些已停止的容器。

```
$ docker container ls -a
CNTNR ID  IMAGE          COMMAND    CREATED  STATUS      NAMES
9cb...65  ubuntu:latest  /bin/bash  `4` mins   Exited `(``0``)`  percy 
```

现在我们可以看到容器显示为`Exited (0)`。停止容器就像停止虚拟机一样。尽管它目前没有运行，但它的整个配置和内容仍然存在于 Docker 主机的文件系统中，并且可以随时重新启动。

让我们使用`docker container start`命令将其从休假中带回来。

```
$ docker container start percy
percy

$ docker container ls
CONTAINER ID  IMAGE          COMMAND      CREATED  STATUS     NAMES
9cb2d2fd1d65  ubuntu:latest  `"/bin/bash"`  `4` mins   Up `3` secs  percy 
```

停止的容器现在已重新启动。是时候验证我们之前创建的文件是否仍然存在了。使用`docker container exec`命令连接到重新启动的容器。

```
`$` `docker` `container` `exec` `-``it` `percy` `bash`
`root``@9``cb2d2fd1d65``:``/``#` 
```

您的 shell 提示符将更改以显示您现在正在容器的命名空间中操作。

验证您之前创建的文件是否仍然存在，并且包含您写入其中的数据。

```
`root``@9``cb2d2fd1d65``:``/``#` `cd` `tmp`
`root``@9``cb2d2fd1d65``:``/``#` `ls` `-``l`
`-``rw``-``r``--``r``--` `1` `root` `root` `14` `Sep` `13` `04``:``22` `newfile`
`root``@9``cb2d2fd1d65``:``/``#`
`root``@9``cb2d2fd1d65``:``/``#` `cat` `newfile`
`DevOps` `FTW` 
```

就像魔术一样，您创建的文件仍然存在，并且其中包含的数据与您离开时完全相同！这证明停止容器不会销毁容器或其中的数据。

虽然这个例子说明了容器的持久性，我应该指出，*卷*是在容器中存储持久数据的首选方式。但在我们的旅程中的这个阶段，我认为这是容器持久性的一个有效例子。

到目前为止，我认为你很难在容器与虚拟机的行为中找到重大差异。

现在让我们杀死容器并从系统中删除它。

可以通过向`docker container rm`传递`-f`标志来删除*运行中*的容器。然而，最佳做法是先停止容器，然后再删除容器。这样做可以给容器正在运行的应用/进程一个干净停止的机会。稍后会详细介绍这一点。

下一个例子将停止`percy`容器，删除它，并验证操作。如果您的终端仍然连接到 percy 容器，您需要通过按`Ctrl-PQ`返回到 Docker 主机的终端。

```
$ docker container stop percy
percy

$ docker container rm percy
percy

$ docker container ls -a
CONTAINER ID    IMAGE      COMMAND    CREATED  STATUS     PORTS      NAMES 
```

“容器现在已被删除 - 在地球上被彻底抹去。如果它是一个好容器，它将成为*无服务器函数*在来世。如果它是一个淘气的容器，它将成为一个愚蠢的终端:-D

总结容器的生命周期...您可以停止、启动、暂停和重新启动容器多次。而且这一切都会发生得非常快。但容器及其数据始终是安全的。直到您明确删除容器，您才有可能丢失其数据。即使在那种情况下，如果您将容器数据存储在*卷*中，那么数据将在容器消失后仍然存在。

让我们快速提一下为什么我们建议在删除容器之前采取两阶段的停止容器的方法。

#### 优雅地停止容器

在 Linux 世界中，大多数容器将运行单个进程。在 Windows 世界中，它们会运行一些进程，但以下规则仍然适用。

在我们之前的例子中，容器正在运行`/bin/bash`应用程序。当您使用`docker container rm <container> -f`杀死正在运行的容器时，容器将在没有警告的情况下被杀死。这个过程非常暴力 - 有点像从容器背后悄悄接近并向其后脑勺开枪。您实际上给了容器和它正在运行的应用程序在被杀死之前没有机会整理自己的事务。

然而，`docker container stop`命令要温和得多（就像拿枪指着容器的头说“你有 10 秒钟说最后的话”）。它会提醒容器内的进程即将被停止，让它有机会在结束之前整理好事情。一旦`docker stop`命令返回，你就可以用`docker container rm`删除容器。

这里背后的魔法可以用 Linux/POSIX *信号*来解释。`docker container stop`向容器内的 PID 1 进程发送**SIGTERM**信号。正如我们刚才说的，这给了进程一个机会来清理事情并优雅地关闭自己。如果它在 10 秒内没有退出，它将收到**SIGKILL**。这实际上就是子弹打在头上。但是嘿，它有 10 秒的时间先整理自己！

`docker container rm <container> -f`不会用**SIGTERM**客气地询问，它直接使用**SIGKILL**。就像我们刚才说的，这就像从背后悄悄接近并猛击头部。顺便说一句，我不是一个暴力的人！

#### 使用重启策略的自我修复容器

通常情况下，使用重启策略来运行容器是个好主意。这是一种自我修复的形式，使 Docker 能够在发生某些事件或故障后自动重新启动它们。

重启策略是针对每个容器应用的，并且可以作为`docker-container run`命令的一部分在命令行上进行配置，也可以在 Compose 文件中声明式地用于 Docker Compose 和 Docker Stacks。

在撰写本文时，存在以下重启策略：

+   `always`

+   `unless-stopped`

+   `on-failed`

**always**策略是最简单的。它会始终重新启动已停止的容器，除非它已被明确停止，比如通过`docker container stop`命令。演示这一点的简单方法是使用`--restart always`策略启动一个新的交互式容器，并告诉它运行一个 shell 进程。当容器启动时，您将附加到其 shell。从 shell 中输入 exit 将杀死容器的 PID 1 进程，从而杀死容器。但是，Docker 会自动重新启动它，因为它是使用`--restart always`策略启动的。如果您发出`docker container ls`命令，您将看到容器的正常运行时间将少于其创建时间。我们在以下示例中展示了这一点。

如果你在 Windows 上进行操作，请用以下命令替换示例中的`docker container run`命令：`docker container run --name neversaydie -it --restart always microsoft/powershell:nanoserver`。

```
$ docker container run --name neversaydie -it --restart always alpine sh

//Wait a few seconds before typing the exit command

/# exit

$ docker container ls
CONTAINER ID    IMAGE     COMMAND    CREATED           STATUS
0901afb84439    alpine    "sh"       35 seconds ago    Up 1 second 
```

注意，容器是 35 秒前创建的，但只运行了 1 秒钟。这是因为我们从容器内部发出`exit`命令时将其杀死了，Docker 不得不重新启动它。

`--restart always`策略的一个有趣的特性是当 Docker 守护进程启动时会重新启动已停止的容器。例如，您使用`--restart always`策略启动一个新容器，然后用`docker container stop`命令停止它。此时，容器处于`Stopped(Exited)`状态。但是，如果您重新启动 Docker 守护程序，则在守护程序重新启动时，容器将自动重新启动。

**always**和**unless-stopped**策略之间的主要区别在于，具有`--restart unless-stopped`策略的容器在`Stopped(Exited)`状态下不会在守护进程重新启动时重新启动。这可能是一个令人困惑的句子，因此让我们通过一个例子来演示。

我们将创建两个新容器。一个名为“always”，使用`--restart always`策略；一个名为“unless-stopped”，使用`--restart unless-stopped`策略。我们使用`docker container stop`命令停止它们，然后重新启动 Docker。 “always”容器将重新启动，但是“unless-stopped”容器将不会重新启动。

1.  创建两个新容器。

    ```
     $ docker container run -d --name always \
       --restart always \
       alpine sleep 1d

     $ docker container run -d --name unless-stopped \
       --restart unless-stopped \
       alpine sleep 1d

     $ docker container ls
     CONTAINER ID   IMAGE     COMMAND       STATUS       NAMES
     3142bd91ecc4   alpine    "sleep 1d"    Up 2 secs    unless-stopped
     4f1b431ac729   alpine    "sleep 1d"    Up 17 secs   always 
    ```

我们现在有两个正在运行的容器。一个名为“always”，一个名为“unless-stopped”。

1.  停止两个容器

    ```
     $ docker container stop always unless-stopped

     $ docker container ls -a
     CONTAINER ID   IMAGE     STATUS                        NAMES
     3142bd91ecc4   alpine    Exited (137) 3 seconds ago    unless-stopped
     4f1b431ac729   alpine    Exited (137) 3 seconds ago    always 
    ```

+   重启 Docker。

在不同的操作系统上，重启 Docker 的过程也不同。这个例子展示了如何停止在运行`systemd`的 Linux 主机上的 Docker。要在 Windows Server 2016 上重新启动 Docker，请使用`restart-service Docker`。

```
 $ systemlctl restart docker 
```

1.  Docker 重启后，您可以检查容器的状态。

    ```
     $ docker container ls -a
     CONTAINER   CREATED             STATUS                       NAMES
     314..cc4    2 minutes ago      Exited (137) 2 minutes ago    unless-stopped
     4f1..729    2 minutes ago      Up 9 seconds                  always 
    ```

注意，使用`--restart always`策略启动的“always”容器已经重新启动，但是使用`--restart unless-stopped`策略启动的“unless-stopped”容器没有重新启动。

**on-failure**策略会在容器以非零的退出代码退出时重新启动。即使 Docker 守护程序重新启动，它也会重新启动处于停止状态的容器。

如果您正在使用 Docker Compose 或 Docker Stacks，则可以将重启策略应用于`service`对象，如下所示：

```
version: "3"
services:
  myservice:
    <Snip>
    restart_policy:
      condition: always | unless-stopped | on-failure 
```

#### Web 服务器示例

到目前为止，我们已经看到了如何启动一个简单的容器并与之交互。我们还看到了如何停止，重启和删除容器。现在让我们看一下 Linux Web 服务器的一个例子。

在此示例中，我们将从我用于一些[Pluralsight 视频课程](https://www.pluralsight.com/search?q=nigel%20poulton%20docker&categories=all)中使用的镜像启动一个新的容器。该镜像在端口 8080 上运行一个极其简单的 Web 服务器。

使用`docker container stop`和`docker container rm`命令来清理您系统上的任何现有容器。然后运行以下`docker container run`命令。

```
$ docker container run -d --name webserver -p 80:8080 \
  nigelpoulton/pluralsight-docker-ci

Unable to find image 'nigelpoulton/pluralsight-docker-ci:latest' locally
latest: Pulling from nigelpoulton/pluralsight-docker-ci
a3ed95caeb02: Pull complete
3b231ed5aa2f: Pull complete
7e4f9cd54d46: Pull complete
929432235e51: Pull complete
6899ef41c594: Pull complete
0b38fccd0dab: Pull complete
Digest: sha256:7a6b0125fe7893e70dc63b2...9b12a28e2c38bd8d3d
Status: Downloaded newer image for nigelpoulton/plur...docker-ci:latest
6efa1838cd51b92a4817e0e7483d103bf72a7ba7ffb5855080128d85043fef21 
```

注意到你的 shell 提示符并没有改变。这是因为我们使用了 `-d` 标志在后台启动了这个容器。在后台启动容器时，不会将其附加到你的终端。

这个例子向 `docker container run` 命令传递了更多的参数，所以让我们快速看一下它们。

我们知道 `docker container run` 会启动一个新容器。但这次我们用 `-d` 标志而不是 `-it`。`-d` 代表**守护进程**模式，告诉容器在后台运行。

然后，我们为容器命名，然后给它 `-p 80:8080`。`-p` 标志将 Docker 主机上的端口映射到容器内部的端口。这次我们将 Docker 主机上的端口 80 映射到容器内部的端口 8080。这意味着命中 Docker 主机上的端口 80 的流量将被定向到容器内部的端口 8080。碰巧，我们用于此容器的镜像定义了一个监听 8080 端口的 web 服务。这意味着我们的容器将启动一个监听 8080 端口的 web 服务器。

最后，我们告诉它使用哪个镜像：`nigelpoulton/pluralsight-docker-ci`。这个镜像没有保持更新，**会**包含漏洞！

运行 `docker container ls` 命令将显示容器正在运行，并显示映射的端口。重要的是要知道端口映射表示为 `主机端口:容器端口`。

```
$ docker container ls
CONTAINER ID  COMMAND        STATUS       PORTS               NAMES
6efa1838cd51  /bin/sh -c...  Up 2 mins  0.0.0.0:80->8080/tcp  webserver 
```

> **注意：** 我们已经从上面的输出中删除了一些列以提高可读性。

现在容器正在运行并且端口已经映射，我们可以通过将 web 浏览器指向 **Docker 主机**的 IP 地址或 DNS 名称的 80 端口来连接到容器。图 7.4 展示了容器提供的网页。

![图 7.4](img/figure7-4.png)

图 7.4

同样的 `docker container stop`、`docker container pause`、`docker container start` 和 `docker container rm` 命令可以用于容器。同样，持久性规则也适用 —— 停止或暂停容器不会销毁容器或其中存储的任何数据。

#### 检查容器

在上一个例子中，你可能已经注意到我们在发出 `docker container run` 命令时没有为容器指定应用程序。但容器运行了一个简单的 web 服务。这是怎么发生的？

在构建 Docker 镜像时，可以嵌入一条指令，列出你希望使用该镜像运行的默认应用程序。如果我们对我们用来运行容器的镜像运行 `docker image inspect`，我们将能够看到容器启动时将运行的应用程序。

```
$ docker image inspect nigelpoulton/pluralsight-docker-ci

[
    {
        "Id": "sha256:07e574331ce3768f30305519...49214bf3020ee69bba1",
        "RepoTags": [
            "nigelpoulton/pluralsight-docker-ci:latest"

            <Snip>

            ],
            "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) CMD [\"/bin/sh\" \"-c\" \"cd /src \u0026\u0026 node \
./app.js\"]"
            ],
<Snip> 
```

`我们已经剪切了输出以便更容易找到我们感兴趣的信息。

在 `Cmd` 后面的条目显示容器将运行的命令/应用，除非你在使用 `docker container run` 启动容器时用不同的命令覆盖它。如果你移除示例中的所有 shell 转义，你将得到以下命令 `/bin/sh -c "cd /src && node ./app.js"`。这是基于该镜像的容器将运行的默认应用。

通常会使用默认命令构建镜像，因为这样可以更容易地启动容器。它还强制了默认行为，并且是镜像的一种自我文档形式 — 即我们可以*检查*镜像并知道它应该运行哪个应用程序。

这就是本章中示例的全部内容。让我们看看整理系统的快速方法。

#### 整理

让我们来看看在 Docker 主机上最简单、最快速地**清除所有正在运行的容器**的方法。不过请注意，该过程将**强制**销毁**所有**容器，而不给它们清理的机会。**绝对不能在生产系统或运行重要容器的系统上执行此操作。

从你的 Docker 主机的 shell 中运行以下命令以删除所有容器。

```
$ docker container rm $(docker container ls -aq) -f
6efa1838cd51 
```

在这个例子中，我们只运行了一个容器，因此只删除了一个 (6efa1838cd51)。然而，这个命令的工作方式与我们在上一章中使用的 `docker image rm $(docker image ls -q)` 命令相同，用于删除单个 Docker 主机上的所有镜像。我们已经知道 `docker container rm` 命令用于删除容器。将 `$(docker container ls -aq)` 传递给它作为参数，实际上传递给它系统上每个容器的 ID。`-f` 标志强制执行操作，使得正在运行的容器也将被销毁。最终结果是……所有容器，无论正在运行还是已停止，都将被销毁并从系统中移除。

上述命令将在 Windows Docker 主机上的 PowerShell 终端中工作。

### 容器 - 命令

+   `docker container run` 是用于启动新容器的命令。在最简单的形式中，它接受一个*镜像*和一个*命令*作为参数。镜像用于创建容器，而命令是你希望容器运行的应用程序。这个例子将在前台启动一个 Ubuntu 容器，并告诉它运行 Bash shell：`docker container run -it ubuntu /bin/bash`。

+   `Ctrl-PQ` 将会将你的 shell 从容器的终端中分离出来，并将容器保持在后台处于运行状态 `(UP)`。

+   `docker container ls` 列出所有处于运行状态 `(UP)` 的容器。如果加上 `-a` 标志，你还会看到处于停止状态 `(Exited)` 的容器。

+   `docker container exec` 允许你在正在运行的容器内运行新进程。它对于将你的 Docker 主机的 shell 附加到正在运行的容器内的终端非常有用。此命令将在正在运行的容器内启动一个新的 Bash shell 并连接到它：`docker container exec -it <container-name or container-id> bash`。为了让这个命令工作，用于创建你的容器的镜像必须包含 Bash shell。

+   `docker container stop`命令将停止运行中的容器并将其置于`Exited (0)`状态。它通过向容器内的 PID 1 进程发送`SIGTERM`来实现。如果该进程在 10 秒内没有清理和停止，将会发出`SIGKILL`强制停止容器。`docker container stop`接受容器 ID 和容器名称作为参数。

+   `docker container start`命令将重新启动已停止的`(Exited)`容器。您可以指定容器的名称或 ID 给`docker container start`。

+   `docker container rm`命令将删除已停止的容器。您可以使用名称或 ID 指定容器。在使用`docker container rm`命令删除容器之前，建议先使用`docker container stop`命令停止容器。

+   `docker container inspect`命令将显示有关容器的详细配置和运行时信息。它接受容器名称和容器 ID 作为其主要参数。

### 章节总结

在这一章节中，我们比较和对比了容器和虚拟机模型。我们探讨了虚拟机模型中固有的“操作系统税”问题，并看到容器模型可以像虚拟机模型对物理模型带来巨大的优势一样带来巨大的优势。

我们看到如何使用`docker container run`命令来启动一些简单的容器，并且我们了解了交互式容器在前台运行与后台运行容器之间的区别。

我们知道，杀死容器内的 PID 1 进程将会杀死容器。而且我们已经看到如何启动、停止和删除容器。

我们使用`docker container inspect`命令来查看详细的容器元数据，完成了这一章节的学习。

到目前为止一切顺利！

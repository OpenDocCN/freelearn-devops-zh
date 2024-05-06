# 第二章：Docker CLI 和 Dockerfile

在上一章中，我们在开发环境中设置了 Docker 并运行了我们的第一个容器。在本章中，我们将探索 Docker 命令行界面。在本章后面，我们将看到如何使用 Dockerfiles 创建自己的 Docker 镜像以及如何自动化这个过程。

在本章中，我们将涵盖以下主题：

+   Docker 术语

+   Docker 命令

+   Dockerfiles

+   Docker 工作流程-拉取-使用-修改-提交-推送工作流程

+   自动化构建

# Docker 术语

在我们开始激动人心的 Docker 领域之前，让我们更好地了解本书中将使用的 Docker 术语。与 VM 镜像类似，Docker 镜像是系统的快照。VM 镜像和 Docker 镜像之间的区别在于 VM 镜像可以运行服务，而 Docker 镜像只是文件系统的快照，这意味着虽然你可以配置镜像来拥有你喜欢的软件包，但你只能在容器中运行一个命令。不过不要担心，由于限制是一个命令，而不是一个进程，所以有方法让 Docker 容器几乎可以执行任何 VM 实例可以执行的任务。

Docker 还实现了类似 Git 的分布式版本管理系统，用于 Docker 镜像。镜像可以存储在本地和远程的仓库中。其功能和术语大量借鉴自 Git-快照被称为提交，你拉取一个镜像仓库，你将本地镜像推送到仓库，等等。

## Docker 容器

一个 Docker 容器可以与虚拟机的实例相关联。它运行沙盒化的进程，这些进程与主机共享相同的内核。术语**容器**来自于集装箱的概念。其想法是你可以从开发环境将容器运送到部署环境，容器中运行的应用程序无论在哪里运行，都会表现出相同的行为。

以下图片显示了 AUFS 的层次结构：

![Docker 容器](img/4787OS_02_01.jpg)

这与集装箱的情境类似，集装箱在交付之前保持密封，但可以在装卸货物、堆叠和运输之间进行操作。

容器中进程的可见文件系统基于 AUFS（尽管您也可以配置容器以使用不同的文件系统）。AUFS 是一种分层文件系统。这些层都是只读的，这些层的合并是进程可见的。但是，如果进程在文件系统中进行更改，将创建一个新层，该层代表原始状态和新状态之间的差异。当您从此容器创建图像时，这些层将被保留。因此，可以基于现有图像构建新图像，创建一个非常方便的图像层次模型。

## Docker 守护程序

`docker`守护程序是管理容器的进程。很容易将其与 Docker 客户端混淆，因为相同的二进制文件用于运行这两个进程。然而，`docker`守护程序需要`root`权限，而客户端不需要。

不幸的是，由于`docker`守护程序以 root 权限运行，它也引入了一个攻击向量。阅读[`docs.Docker.com/articles/security/`](https://docs.Docker.com/articles/security/)获取更多详细信息。

## Docker 客户端

Docker 客户端是与`docker`守护程序交互以启动或管理容器的工具。Docker 使用 RESTful API 在客户端和守护程序之间进行通信。

### 注意

REST 是一种架构风格，由一组协调的架构约束应用于分布式超媒体系统中的组件、连接器和数据元素。简而言之，RESTful 服务使用标准的 HTTP 方法，如`GET`、`POST`、`PUT`和`DELETE`方法。

## Dockerfile

Dockerfile 是一个用**特定领域语言（DSL）**编写的文件，其中包含设置 Docker 镜像的指令。可以将其视为 Docker 的 Makefile 等效文件。

## Docker 注册表

这是 Docker 社区发布的所有 Docker 镜像的公共存储库。您可以自由地从该注册表中拉取镜像，但要推送镜像，您必须在[`hub.docker.com`](http://hub.docker.com)注册。Docker 注册表和 Docker Hub 是由 Docker Inc.运营和维护的服务，并提供无限免费的存储库。您也可以购买私人存储库。

# Docker 命令

现在让我们在 Docker CLI 上动手。我们将看一下最常用的命令及其用法。Docker 命令是模仿 Linux 和 Git 的，所以如果您使用过其中任何一个，您将发现在 Docker 中也能得心应手。

这里只提到了最常用的选项。要获取完整的参考信息，您可以查看官方文档[`docs.docker.com/reference/commandline/cli/`](https://docs.docker.com/reference/commandline/cli)。

## 守护程序命令

如果您通过标准存储库安装了`docker`守护程序，则启动`docker`守护程序的命令将被添加到`init`脚本中，以便在启动时自动启动服务。否则，您将需要自己运行`docker`守护程序，以使客户端命令正常工作。

现在，在启动守护程序时，您可以使用控制**域** **名** **系统**（**DNS**）配置、存储驱动程序和容器的执行驱动程序的参数来运行它：

```
$ export DOCKER_HOST="tcp://0.0.0.0:2375"
$ Docker -d -D -e lxc -s btrfs –-dns 8.8.8.8 –-dns-search example.com

```

### 注意

只有在您想要自己启动守护程序时才需要这些。否则，您可以使用`$ sudo service Docker start`启动`docker`守护程序。对于 OSX 和 Windows，您需要运行第一章中提到的命令，*安装 Docker*。

以下表格描述了各种标志：

| 标志 | 说明 |
| --- | --- |

|

```
-d

```

| 这以守护程序运行 Docker。 |
| --- |

|

```
-D

```

| 这以调试模式运行 Docker。 |
| --- |

|

```
-e [option]

```

| 这是要使用的执行驱动程序。默认的执行驱动程序是本机，它使用`libcontainer`。 |
| --- |

|

```
-s [option]

```

| 这会强制 Docker 使用不同的存储驱动程序。默认值为""，Docker 使用 AUFS。 |
| --- |

|

```
--dns [option(s)]

```

| 这为所有 Docker 容器设置 DNS 服务器（或服务器）。 |
| --- |

|

```
--dns-search [option(s)]

```

| 这为所有 Docker 容器设置 DNS 搜索域（或域）。 |
| --- |

|

```
-H [option(s)]

```

| 这是要绑定的套接字（或套接字）。可以是一个或多个`tcp://host:port, unix:///path/to/socket, fd://* or fd://socketfd`。 |
| --- |

如果同时运行多个`docker`守护程序，则客户端将遵循`DOCKER_HOST`参数设置的值。您还可以使用`-H`标志使其连接到特定的守护程序。

考虑这个命令：

```
$ docker -H tcp://0.0.0.0:2375 run -it ubuntu /bin/bash

```

前面的命令与以下命令相同：

```
$ DOCKER_HOST="tcp://0.0.0.0:2375" docker run -it ubuntu /bin/bash

```

## 版本命令

`version`命令打印版本信息：

```
$ docker -vDocker version 1.1.1, build bd609d2

```

## 信息命令

`info`命令打印`docker`守护程序配置的详细信息，例如执行驱动程序、正在使用的存储驱动程序等：

```
$ docker info # The author is running it in boot2docker on OSX
Containers: 0
Images: 0
Storage Driver: aufs
Root Dir: /mnt/sda1/var/lib/docker/aufs
Dirs: 0
Execution Driver: native-0.2
Kernel Version: 3.15.3-tinycore64
Debug mode (server): true
Debug mode (client): false
Fds: 10
Goroutines: 10
EventsListeners: 0
Init Path: /usr/local/bin/docker
Sockets: [unix:///var/run/docker.sock tcp://0.0.0.0:2375]

```

## run 命令

run 命令是我们将经常使用的命令。它用于运行 Docker 容器：

```
$ docker run [options] IMAGE [command] [args]

```

| 标志 | 解释 |
| --- | --- |

|

```
-a, --attach=[]

```

| 附加到`stdin`，`stdout`或`stderr`文件（标准输入，输出和错误文件）。 |
| --- |

|

```
-d, --detach

```

| 在后台运行容器。 |
| --- |

|

```
-i, --interactive

```

| 以交互模式运行容器（保持`stdin`文件打开）。 |
| --- |

|

```
-t, --tty

```

| 分配伪`tty`标志（如果要附加到容器的终端，则需要）。 |
| --- |

|

```
-p, --publish=[]

```

| 将容器的端口发布到主机（`ip:hostport:containerport`）。 |
| --- |

|

```
--rm

```

| 退出时自动删除容器（不能与`-d`标志一起使用）。 |
| --- |

|

```
--privileged

```

| 这为该容器提供了额外的特权。 |
| --- |

|

```
-v, --volume=[]

```

| 绑定挂载卷（从主机=>`/host:/container`；从 docker=>`/container`）。 |
| --- |

|

```
--volumes-from=[]

```

| 从指定的容器中挂载卷。 |
| --- |

|

```
-w, --workdir=""

```

| 这是容器内的工作目录。 |
| --- |

|

```
--name=""

```

| 为容器分配一个名称。 |
| --- |

|

```
-h, --hostname=""

```

| 为容器分配一个主机名。 |
| --- |

|

```
-u, --user=""

```

| 这是容器应该运行的用户名或 UID。 |
| --- |

|

```
-e, --env=[]

```

| 设置环境变量。 |
| --- |

|

```
--env-file=[]

```

| 从新的行分隔文件中读取环境变量。 |
| --- |

|

```
--dns=[]

```

| 设置自定义 DNS 服务器。 |
| --- |

|

```
--dns-search=[]

```

| 设置自定义 DNS 搜索域。 |
| --- |

|

```
--link=[]

```

| 添加到另一个容器的链接（`name:alias`）。 |
| --- |

|

```
-c, --cpu-shares=0

```

| 这是此容器的相对 CPU 份额。 |
| --- |

|

```
--cpuset=""

```

| 这些是允许执行的 CPU；从 0 开始。（例如，0 到 3）。 |
| --- |

|

```
-m, --memory=""

```

| 这是此容器的内存限制`(<number><b | k | m | g>`)。 |
| --- | --- | --- | --- |

|

```
--restart=""

```

| （v1.2+）指定容器崩溃时的重启策略。 |
| --- |

|

```
--cap-add=""

```

| （v1.2+）这向容器授予一个功能（参考第四章，“安全最佳实践”）。 |
| --- |

|

```
--cap-drop=""

```

| （v1.2+）这将把一个功能限制到一个容器中（参考第四章，“安全最佳实践”）。 |
| --- |

|

```
--device=""

```

| （v1.2+）这在容器上挂载设备。 |
| --- |

在运行容器时，重要的是要记住，容器的生命周期与启动容器时运行的命令的生命周期相关联。现在尝试运行这个：

```
$ docker run -dt ubuntu ps
b1d037dfcff6b076bde360070d3af0d019269e44929df61c93dfcdfaf29492c9
$ docker attach b1d037
2014/07/16 16:01:29 You cannot attach to a stopped container, start it first

```

发生了什么？当我们运行简单命令`ps`时，容器运行了该命令并退出。因此，我们得到了一个错误。

### 注意

`attach`命令将标准输入和输出附加到正在运行的容器上。

这里还有一条重要的信息，您不需要为所有需要容器 ID 的命令使用完整的 64 字符 ID。前面的几个字符就足够了。使用与以下代码中显示的相同示例：

```
$ docker attach b1d03
2014/07/16 16:09:39 You cannot attach to a stopped container, start it first
$ docker attach b1d0
2014/07/16 16:09:40 You cannot attach to a stopped container, start it first
$ docker attach b1d
2014/07/16 16:09:42 You cannot attach to a stopped container, start it first
$ docker attach b1
2014/07/16 16:09:44 You cannot attach to a stopped container, start it first
$ docker attach b
2014/07/16 16:09:45 Error: No such container: b

```

一个更方便的方法是自己为容器命名：

```
$ docker run -dit --name OD-name-example ubuntu /bin/bash
1b21af96c38836df8a809049fb3a040db571cc0cef000a54ebce978c1b5567ea
$ docker attach OD-name-example
root@1b21af96c388:/#

```

`-i`标志是必要的，以便在容器中进行任何交互，`-t`标志是必要的，以创建一个伪终端。

前面的示例还让我们意识到，即使我们退出容器，它仍处于`stopped`状态。也就是说，我们可以重新启动容器，并保留其文件系统层。您可以通过运行以下命令来查看：

```
$ docker ps -a
CONTAINER ID IMAGE         COMMAND CREATED    STATUS    NAMES
eb424f5a9d3f ubuntu:latest ps      1 hour ago Exited OD-name-example

```

虽然这很方便，但很快您的主机磁盘空间可能会耗尽，因为保存了越来越多的容器。因此，如果您要运行一个一次性容器，可以使用`--rm`标志运行它，这将在进程退出时删除容器：

```
$ docker run --rm -it --name OD-rm-example ubuntu /bin/bash
root@0fc99b2e35fb:/# exit
exit
$ docker ps -a
CONTAINER ID    IMAGE    COMMAND    CREATED    STATUS   PORTS   NAMES

```

### 运行服务器

现在，对于我们的下一个示例，我们将尝试运行一个 Web 服务器。选择此示例是因为 Docker 容器最常见的实际用例是运行 Web 应用程序：

```
$ docker run -it –-name OD-pythonserver-1 --rm python:2.7 \
python -m SimpleHTTPServer 8000;
Serving HTTP on 0.0.0.0 port 8000

```

现在我们知道问题所在；我们在一个容器中运行了一个服务器，但由于 Docker 动态分配了容器的 IP，这使事情变得困难。但是，我们可以将容器的端口绑定到主机的端口，Docker 会负责转发网络流量。现在让我们再次尝试这个命令，加上`-p`标志：

```
$ docker run -p 0.0.0.0:8000:8000 -it --rm –-name OD-pythonserver-2 \ python:2.7 python -m SimpleHTTPServer 8000;
Serving HTTP on 0.0.0.0 port 8000 ...
172.17.42.1 - - [18/Jul/2014 14:25:46] "GET / HTTP/1.1" 200 -

```

现在打开浏览器，转到`http://localhost:8000`。大功告成！

如果您是 OS X 用户，并且意识到无法访问`http://localhost:8000`，那是因为 VirtualBox 尚未配置为响应对 boot2Docker VM 的**网络地址转换**（**NAT**）请求。将以下函数添加到您的别名文件（`bash_profile`或`.bashrc`）将节省很多麻烦：

```
natboot2docker () { VBoxManage controlvm boot2docker-vm natpf1 \
   "$1,tcp,127.0.0.1,$2,,$3"; }

removeDockerNat() {
    VBoxManage modifyvm boot2docker-vm \
    --natpf1 delete $1;
}
```

之后，您应该能够使用`$ natboot2docker mypythonserver 8000 8000`命令来访问 Python 服务器。但是请记住，在完成后运行`$ removeDockerDockerNat mypythonserver`命令。否则，当您下次运行 boot2Docker VM 时，您将面临一个错误，它将不允许您获取 IP 地址或`ssh`脚本：

```
$ boot2docker ssh
ssh_exchange_identification: Connection closed by remote host
2014/07/19 11:55:09 exit status 255

```

您的浏览器现在显示容器的`/root`路径。如果您想要提供主机的目录怎么办？让我们尝试挂载设备：

```
root@eb53f7ec79fd:/# mount -t tmpfs /dev/random /mnt
mount: permission denied

```

正如您所见，`mount`命令不起作用。实际上，除非包括`--privileged`标志，否则大多数潜在危险的内核功能都会被禁用。

但是，除非您知道自己在做什么，否则永远不要使用此标志。Docker 提供了一种更简单的方式来绑定挂载主机卷和使用`-v`和`–volumes`选项绑定挂载主机卷。让我们在我们当前所在的目录中再次尝试这个例子：

```
$ docker run -v $(pwd):$(pwd) -p 0.0.0.0:8000:8000 -it –rm \
--name OD-pythonserver-3 python:2.7 python -m SimpleHTTPServer 8000;
Serving HTTP on 0.0.0.0 port 8000 ...
10.0.2.2 - - [18/Jul/2014 14:40:35] "GET / HTTP/1.1" 200 -

```

现在，您已经将您从中运行命令的目录绑定到了容器。但是，当您访问容器时，仍然会得到容器根目录的目录列表。为了提供已绑定到容器的目录，让我们使用`-w`标志将其设置为容器的工作目录（容器化进程运行的目录）：

```
$ docker run -v $(pwd):$(pwd) -w $(pwd) -p 0.0.0.0:8000:8000 -it \ --name OD-pythonserver-4 python:2.7 python -m SimpleHTTPServer 8000;
Serving HTTP on 0.0.0.0 port 8000 ...
10.0.2.2 - - [18/Jul/2014 14:51:35] "GET / HTTP/1.1" 200 -

```

### 注意

Boot2Docker 用户目前还无法利用这一功能，除非您使用了增强功能并设置了共享文件夹，可以在[`medium.com/boot2docker-lightweight-linux-for-docker/boot2docker-together-with-virtualbox-guest-additions-da1e3ab2465c`](https://medium.com/boot2docker-lightweight-linux-for-docker/boot2docker-together-with-virtualbox-guest-additions-da1e3ab2465c)找到相关指南。尽管这种解决方案有效，但这是一种 hack 方法，不建议使用。与此同时，Docker 社区正在积极寻找解决方案（请查看 boot2Docker GitHub 存储库中的问题`#64`和 Docker 存储库中的问题`#4023`）。

现在，`http://localhost:8000`将提供您当前正在运行的目录，但是从 Docker 容器中提供。但要小心，因为您所做的任何更改也会写入主机的文件系统中。

### 提示

自 v1.1.1 版本开始，您可以使用`$ docker run -v /:/my_host:ro ubuntu ls /my_host`将主机的根目录绑定到容器，但是禁止在容器的`/`路径上进行挂载。

卷可以选择地以`：ro`或`：rw`命令作为后缀，以只读或读写模式挂载卷。默认情况下，卷以与主机相同的模式（读写或只读）挂载。

此选项主要用于挂载静态资产和写入日志。

但是，如果我想挂载外部设备呢？

在 v1.2 之前，您必须在主机中挂载设备，并在特权容器中使用`-v`标志进行绑定挂载，但是 v1.2 添加了一个`--device`标志，您可以使用它来挂载设备，而无需使用`--privileged`标志。

例如，要在容器中使用网络摄像头，请运行此命令：

```
$ docker run --device=/dev/video0:/dev/video0

```

Docker v1.2 还添加了一个`--restart`标志，用于为容器指定重新启动策略。目前有三种重新启动策略：

+   `no`：如果容器死掉，则不重新启动（默认）。

+   `on-failure`：如果以非零退出代码退出，则重新启动容器。它还可以接受一个可选的最大重新启动计数（例如，`on-failure:5`）。

+   `always`：无论返回的退出代码是什么，都始终重新启动容器。

以下是一个无限重新启动的示例：

```
$ docker run --restart=always code.it

```

下一行用于在放弃之前尝试五次：

```
$ docker run --restart=on-failure:5 code.it

```

## search 命令

`search`命令允许我们在公共注册表中搜索 Docker 镜像。让我们搜索与 Python 相关的所有镜像：

```
$ docker search python | less

```

## pull 命令

`pull`命令用于从注册表中拉取镜像或仓库。默认情况下，它们从公共 Docker 注册表中拉取，但如果您运行自己的注册表，也可以从中拉取它们：

```
$ docker pull python # pulls repository from Docker Hub
$ docker pull python:2.7 # pulls the image tagged 2.7
$ docker pull <path_to_registry>/<image_or_repository>

```

## start 命令

我们在讨论`docker run`时看到，容器状态在退出时会被保留，除非明确删除。`docker start`命令用于启动已停止的容器：

```
$ docker start [-i] [-a] <container(s)>

```

考虑以下`start`命令的示例：

```
$ docker ps -a
CONTAINER ID IMAGE         COMMAND   CREATED STATUS    NAMES
e3c4b6b39cff ubuntu:latest python -m 1h ago  Exited OD-pythonserver-4
81bb2a92ab0c ubuntu:latest /bin/bash 1h ago  Exited evil_rosalind
d52fef570d6e ubuntu:latest /bin/bash 1h ago  Exited prickly_morse
eb424f5a9d3f ubuntu:latest /bin/bash 20h ago Exited OD-name-example
$ docker start -ai OD-pythonserver-4
Serving HTTP on 0.0.0.0 port 8000

```

选项的含义与`docker run`命令相同。

## stop 命令

stop 命令通过发送`SIGTERM`信号然后在宽限期之后发送`SIGKILL`信号来停止正在运行的容器：

### 注意

`SIGTERM`和`SIGKILL`是 Unix 信号。信号是 Unix、类 Unix 和其他符合 POSIX 的操作系统中使用的一种进程间通信形式。`SIGTERM`信号指示进程终止。`SIGKILL`信号用于强制终止进程。

```
docker run -dit --name OD-stop-example ubuntu /bin/bash
$ docker ps
CONTAINER ID IMAGE         COMMAND   CREATED  STATUS    NAMES
679ece6f2a11 ubuntu:latest /bin/bash 5h ago   Up 3s   OD-stop-example
$ docker stop OD-stop-example
OD-stop-example
$ docker ps
CONTAINER ID IMAGE         COMMAND   CREATED  STATUS    NAMES

```

您还可以指定`-t`标志或`--time`标志，允许您设置等待时间。

## restart 命令

`restart`命令重新启动正在运行的容器：

```
$ docker run -dit --name OD-restart-example ubuntu /bin/bash
$ sleep 15s # Suspends execution for 15 seconds
$ docker ps
CONTAINER ID IMAGE         COMMAND   STATUS    NAMES
cc5d0ae0b599 ubuntu:latest /bin/bash Up 20s    OD-restart-example

$ docker restart OD-restart-example
$ docker ps
CONTAINER ID IMAGE         COMMAND   STATUS    NAMES
cc5d0ae0b599 ubuntu:latest /bin/bash Up 2s    OD-restart-example

```

如果您观察状态，您会注意到容器已经重新启动。

## rm 命令

`rm`命令用于删除 Docker 容器：

```
$ Docker ps -a # Lists containers including stopped ones
CONTAINER ID  IMAGE  COMMAND   CREATED  STATUS NAMES
cc5d0ae0b599  ubuntu /bin/bash 6h ago   Exited OD-restart-example
679ece6f2a11  ubuntu /bin/bash 7h ago   Exited OD-stop-example
e3c4b6b39cff  ubuntu /bin/bash 9h ago   Exited OD-name-example

```

在我们的冒险之后，似乎有很多容器剩下。让我们移除其中一个：

```
$ dockerDocker rm OD-restart-example
cc5d0ae0b599

```

我们还可以组合两个 Docker 命令。让我们将`docker ps -a -q`命令（打印`docker ps -a`中容器的 ID 参数）和`docker rm`命令结合起来，一次性删除所有容器：

```
$ docker rm $(docker ps -a -q)
679ece6f2a11
e3c4b6b39cff
$ docker ps -a
CONTAINER ID    IMAGE    COMMAND     CREATED    STATUS      NAMES

```

首先对`docker ps -a -q`命令进行评估，然后输出由`docker rm`命令使用。

## ps 命令

`ps`命令用于列出容器。它的使用方式如下：

```
$ docker ps [option(s)]

```

| 标志 | 解释 |
| --- | --- |

|

```
-a, --all

```

| 这显示所有容器，包括已停止的容器。 |
| --- |

|

```
-q, --quiet

```

| 这仅显示容器 ID 参数。 |
| --- |

|

```
-s, --size

```

| 这打印出容器的大小。 |
| --- |

|

```
-l, --latest

```

| 这只显示最新的容器（包括已停止的容器）。 |
| --- |

|

```
-n=""

```

| 这显示最后*n*个容器（包括已停止的容器）。其默认值为-1。 |
| --- |

|

```
--before=****""

```

| 这显示了在指定 ID 或名称之前创建的容器。它包括已停止的容器。 |
| --- |

|

```
--after=""

```

| 这显示了在指定 ID 或名称之后创建的容器。它包括已停止的容器。 |
| --- |

`docker ps`命令默认只显示正在运行的容器。要查看所有容器，请运行`docker ps -a`命令。要仅查看容器 ID 参数，请使用`-q`标志运行它。

## logs 命令

`logs`命令显示容器的日志：

```
Let us look at the logs of the python server we have been running
$ docker logs OD-pythonserver-4
Serving HTTP on 0.0.0.0 port 8000 ...
10.0.2.2 - - [18/Jul/2014 15:06:39] "GET / HTTP/1.1" 200 -
^CTraceback (most recent call last):
File ...
...
KeyboardInterrupt

```

你还可以提供一个`--tail`参数来跟踪容器运行时的输出。

## inspect 命令

`inspect`命令允许你获取容器或镜像的详细信息。它将这些详细信息作为 JSON 数组返回：

```
$ Docker inspect ubuntu # Running on an image
[{
"Architecture": "amd64",
"Author": "",
"Comment": "",
.......
.......
.......
"DockerVersion": "0.10.0",
"Id": "e54ca5efa2e962582a223ca9810f7f1b62ea9b5c3975d14a5da79d3bf6020f37",
"Os": "linux",
"Parent": "6c37f792ddacad573016e6aea7fc9fb377127b4767ce6104c9f869314a12041e",
"Size": 178365
}]

```

同样，对于一个容器，我们运行以下命令：

```
$ Docker inspect OD-pythonserver-4 # Running on a container
[{
"Args": [
"-m",
"SimpleHTTPServer",
"8000"
],
......
......
"Name": "/OD-pythonserver-4",
"NetworkSettings": {
"Bridge": "Docker0",
"Gateway": "172.17.42.1",
"IPAddress": "172.17.0.11",
"IPPrefixLen": 16,
"PortMapping": null,
"Ports": {
"8000/tcp": [
{
"HostIp": "0.0.0.0",
"HostPort": "8000"
}
]
}
},
......
......
"Volumes": {
"/home/Docker": "/home/Docker"
},
"VolumesRW": {
"/home/Docker": true
}
}]

```

Docker inspect 提供了关于容器或镜像的所有低级信息。在上面的例子中，找出容器的 IP 地址和暴露的端口，并向`IP:port`发出请求。你会发现你直接访问了在容器中运行的服务器。

然而，手动查看整个 JSON 数组并不是最佳选择。因此，`inspect`命令提供了一个标志`-f`（或`--follow`标志），允许你使用`Go`模板精确地指定你想要的内容。例如，如果你只想获取容器的 IP 地址，运行以下命令：

```
$ docker inspect -f  '{{.NetworkSettings.IPAddress}}' \
OD-pythonserver-4;
172.17.0.11

```

`{{.NetworkSettings.IPAddress}}`是在 JSON 结果上执行的`Go`模板。`Go`模板非常强大，你可以在[`golang.org/pkg/text/template/`](http://golang.org/pkg/text/template/)上列出一些你可以用它们做的事情。

## top 命令

`top`命令显示容器中正在运行的进程及其统计信息，模仿 Unix 的`top`命令。

让我们下载并运行`ghost`博客平台，并查看其中运行的进程：

```
$ docker run -d -p 4000:2368 --name OD-ghost dockerfile/ghost
ece88c79b0793b0a49e3d23e2b0b8e75d89c519e5987172951ea8d30d96a2936

$ docker top OD-ghost-1
PID                 USER                COMMAND
1162                root                bash /ghost-start
1180                root                npm
1186                root                sh -c node index
1187                root                node index

```

是的！我们只需一条命令就设置了我们自己的`ghost`博客。这带来了另一个微妙的优势，并展示了可能是未来趋势的东西。现在，通过 TCP 端口暴露其服务的每个工具都可以被容器化，并在其自己的沙盒世界中运行。你只需要暴露它的端口并将其绑定到你的主机端口。你不需要担心安装、依赖关系、不兼容性等，卸载将是干净的，因为你只需要停止所有的容器并删除镜像。

### 注意

Ghost 是一个开源的发布平台，设计精美，易于使用，对所有人免费。它是用 Node.js 编写的，是一个服务器端 JavaScript 执行引擎。

## 附加命令

`attach`命令用于附加到正在运行的容器。

让我们启动一个带有 Node.js 的容器，将 node 交互式 shell 作为守护进程运行，然后稍后附加到它。

### 注意

Node.js 是一个事件驱动的、异步 I/O 的 Web 框架，它在 Google 的 V8 运行环境上运行用 JavaScript 编写的应用程序。

带有 Node.js 的容器如下：

```
$ docker run -dit --name OD-nodejs shykes/nodejs node
8e0da647200efe33a9dd53d45ea38e3af3892b04aa8b7a6e167b3c093e522754

$ docker attach OD-nodejs
console.log('Docker rocks!');Docker rocks!

```

## 杀死命令

`kill`命令会杀死一个容器，并向容器中运行的进程发送`SIGTERM`信号：

```
Let us kill the container running the ghost blog.
$ docker kill OD-ghost-1
OD-ghost-1

$ docker attach OD-ghost-1 # Verification
2014/07/19 18:12:51 You cannot attach to a stopped container, start it first

```

## cp 命令

`cp`命令将文件或文件夹从容器的文件系统复制到主机路径。路径是相对于文件系统的根目录的。

是时候玩一些游戏了。首先，让我们用`/bin/bash`命令运行一个 Ubuntu 容器：

```
$ docker run -it –name OD-cp-bell ubuntu /bin/bash

```

现在，在容器内部，让我们创建一个带有特殊名称的文件：

```
# touch $(echo -e '\007')

```

`\ 007`字符是 ASCII`BEL`字符，当在终端上打印时会响铃系统。你可能已经猜到我们要做什么了。所以让我们打开一个新的终端，并执行以下命令将这个新创建的文件复制到主机：

```
$ docker cp OD-cp-bell:/$(echo -e '\007') $(pwd)

```

### 提示

要使`docker cp`命令工作，容器路径和主机路径都必须完整，所以不要使用`.`、`,`、`*`等快捷方式。

所以我们在容器中创建了一个文件名为`BEL`字符的空文件。然后我们将文件复制到主机容器中的当前目录。只剩最后一步了。在执行`docker cp`命令的主机标签中，运行以下命令：

```
$ echo *

```

你会听到系统铃声响起！我们本可以从容器中复制任何文件或目录到主机。但玩一些游戏也无妨！

### 注意

如果您觉得这很有趣，您可能会喜欢阅读[`www.dwheeler.com/essays/fixing-unix-linux-filenames.html`](http://www.dwheeler.com/essays/fixing-unix-linux-filenames.html)。这是一篇很棒的文章，讨论了文件名中的边缘情况，这可能会在程序中引起简单到复杂的问题。

## 端口命令

`port`命令查找绑定到容器中公开端口的公共端口：

```
$ docker port CONTAINER PRIVATE_PORT
$ docker port OD-ghost 2368
4000

```

Ghost 在`2368`端口运行一个服务器，允许您编写和发布博客文章。在示例中，我们将主机端口绑定到`OD-ghost`容器的端口`2368`。

# 运行您自己的项目

到目前为止，我们已经相当熟悉基本的 Docker 命令。让我们提高赌注。在接下来的几个命令中，我将使用我的一个副业项目。请随意使用您自己的项目。

让我们首先列出我们的要求，以确定我们必须传递给`docker run`命令的参数。

我们的应用程序将在 Node.js 上运行，因此我们将选择维护良好的`dockerfile/nodejs`镜像来启动我们的基础容器：

+   我们知道我们的应用程序将绑定到端口`8000`，因此我们将将端口暴露给主机的`8000`端口。

+   我们需要为容器指定一个描述性名称，以便我们可以在将来的命令中引用它。在这种情况下，让我们选择应用程序的名称：

```
$ docker run -it --name code.it dockerfile/nodejs /bin/bash
[ root@3b0d5a04cdcd:/data ]$ cd /home
[ root@3b0d5a04cdcd:/home ]$

```

一旦您启动了容器，您需要检查应用程序的依赖项是否已经可用。在我们的情况下，除了 Node.js 之外，我们只需要 Git，它已经安装在`dockerfile/nodejs`镜像中。

既然我们的容器已经准备好运行我们的应用程序，剩下的就是获取源代码并进行必要的设置来运行应用程序：

```
$ git clone https://github.com/shrikrishnaholla/code.it.git
$ cd code.it && git submodule update --init --recursive

```

这将下载应用程序中使用的插件的源代码。

然后运行以下命令：

```
$ npm install

```

现在所有运行应用程序所需的节点模块都已安装。

接下来，运行此命令：

```
$ node app.js

```

现在您可以转到`localhost:8000`来使用该应用程序。

## 差异命令

`diff`命令显示容器与其基于的镜像之间的差异。在这个例子中，我们正在运行一个带有`code.it`的容器。在一个单独的标签中，运行此命令：

```
$ docker diff code.it
C /home
A /home/code.it
...

```

## 提交命令

`commit`命令使用容器的文件系统创建一个新的镜像。就像 Git 的`commit`命令一样，您可以设置描述镜像的提交消息：

```
$ docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]

```

| 标志 | 解释 |
| --- | --- |

|

```
-p, --pause

```

| 这在提交期间暂停容器（从 v1.1.1+开始可用）。 |
| --- |

|

```
-m, --message=""

```

| 这是提交消息。它可以是对图像功能的描述。 |
| --- |

|

```
-a, --author=""

```

| 这显示了作者的详细信息。 |
| --- |

例如，让我们使用这个命令来提交我们设置的容器：

```
$ docker commit -m "Code.it – A browser based text editor and interpreter" -a "Shrikrishna Holla <s**a@gmail.com>" code.it shrikrishna/code.it:v1

```

### 提示

如果您正在复制这些示例，请替换作者详细信息和图像名称的用户名部分。

输出将是一个冗长的图像 ID。如果您仔细查看命令，我们已经命名了图像`shrikrishna/code.it:v1`。这是一个约定。图像/存储库名称的第一部分（斜杠之前）是作者的 Docker Hub 用户名。第二部分是预期的应用程序或图像名称。第三部分是一个标签（通常是版本描述），用冒号与第二部分分隔。

### 注意

`Docker` `Hub`是由 Docker，Inc 维护的公共注册表。它托管公共 Docker 图像，并提供帮助您构建和管理 Docker 环境的服务。有关更多详细信息，请访问[`hub.docker.com`](https://hub.docker.com)。

带有不同版本标签的图像集合是一个存储库。通过运行`docker commit`命令创建的图像将是本地图像，这意味着您将能够从中运行容器，但它不会公开可用。要使其公开或推送到您的私有 Docker 注册表，请使用`docker push`命令。

## images 命令

`images`命令列出系统中的所有图像：

```
$ docker images [OPTIONS] [NAME]

```

| 标志 | 说明 |
| --- | --- |

|

```
-a, --all

```

| 这显示所有图像，包括中间层。 |
| --- |

|

```
-f, --filter=[]

```

| 这提供过滤值。 |
| --- |

|

```
--no-trunc

```

| 这不会截断输出（显示完整的 ID）。 |
| --- |

|

```
-q, --quiet

```

| 这只显示图像 ID。 |
| --- |

现在让我们看一下`image`命令的几个用法示例：

```
$ docker images
REPOSITORY           TAG   IMAGE ID       CREATED    VIRTUAL SIZE
shrikrishna/code.it  v1    a7cb6737a2f6   6m ago     704.4 MB

```

这列出了所有顶层图像，它们的存储库和标签，以及它们的虚拟大小。

Docker 图像只是一堆只读文件系统层。然后，像 AUFS 这样的联合文件系统合并这些层，它们看起来像是一个文件系统。

在 Docker 术语中，只读层就是一个图像。它永远不会改变。当运行一个容器时，进程会认为整个文件系统是可读写的。但是更改只会发生在最顶层的可写层，这是在容器启动时创建的。图像的只读层保持不变。当你提交一个容器时，它会冻结顶层（底层已经冻结）并将其转换为图像。现在，当一个容器启动这个图像时，图像的所有层（包括之前的可写层）都是只读的。所有的更改现在都是在所有底层的顶部创建一个新的可写层。然而，由于联合文件系统（如 AUFS）的工作方式，进程认为文件系统是可读写的。

我们`code.it`示例中涉及的层次的大致示意图如下：

![图像命令](img/4787OS_02_02.jpg)

### 注意

在这一点上，可能明智地考虑联合文件系统需要多大的努力来合并所有这些层，并提供一致的性能。在某个时候，事情不可避免地会出错。例如，AUFS 有一个 42 层的限制。当层数超过这个限制时，它就不允许创建更多的层，构建就会失败。阅读[`github.com/docker/docker/issues/1171`](https://github.com/docker/docker/issues/1171)获取更多关于这个问题的信息。

以下命令列出了最近创建的图像：

```
$ docker images | head

```

`-f`标志可以给出`key=value`类型的参数。它经常用于获取悬空图像的列表：

```
$ docker images -f "dangling=true"

```

这将显示未标记的图像，也就是说，已经提交或构建而没有标记的图像。

## rmi 命令

`rmi`命令删除图像。删除一个图像也会删除它所依赖的所有底层图像，并在拉取时下载的图像：

```
$ docker rmi [OPTION] {IMAGE(s)]

```

| 标志 | 解释 |
| --- | --- |

|

```
-f, --force

```

| 这将强制删除图像（或图像）。 |
| --- |

|

```
--no-prune

```

| 这个命令不会删除未标记的父级。 |
| --- |

这个命令从你的机器中删除一个图像：

```
$ docker rmi test

```

## 保存命令

`save`命令将图像或存储库保存在一个 tarball 中，并将其流到`stdout`文件，保留有关图像的父层和元数据：

```
$ docker save -o codeit.tar code.it

```

`-o`标志允许我们指定一个文件而不是流到`stdout`文件。它用于创建一个备份，然后可以与`docker load`命令一起使用。

## 加载命令

`load`命令从 tarball 中加载图像，恢复文件系统层和与图像相关的元数据：

```
$ docker load -i codeit.tar

```

`-i`标志允许我们指定一个文件，而不是尝试从`stdin`文件获取流。

## 导出命令

`export`命令将容器的文件系统保存为 tarball 并流式传输到`stdout`文件。它会展平文件系统层。换句话说，它会合并所有文件系统层。在此过程中，图像历史的所有元数据都会丢失：

```
$ sudo Docker export red_panda > latest.tar

```

在这里，`red_panda`是我其中一个容器的名称。

## 导入命令

`import`命令创建一个空的文件系统映像，并将 tarball 的内容导入其中。您可以选择为该图像打标签：

```
$ docker import URL|- [REPOSITORY[:TAG]]

```

URL 必须以`http`开头。

```
$ docker import http://example.com/test.tar.gz # Sample url

```

如果您想要从本地目录或存档中导入，可以使用-参数从`stdin`文件中获取数据：

```
$ cat sample.tgz | docker import – testimage:imported

```

## 标签命令

您可以向图像添加`tag`命令。它有助于识别图像的特定版本。

例如，`python`图像名称表示`python:latest`，即可用的最新版本的 Python，这可能会随时更改。但每当它更新时，旧版本都会用相应的 Python 版本标记。因此，`python:2.7`命令将安装 Python 2.7。因此，`tag`命令可用于表示图像的版本，或用于需要识别不同图像版本的任何其他目的：

```
$ docker tag IMAGE [REGISTRYHOST/][USERNAME/]NAME[:TAG]

```

`REGISTRYHOST`命令仅在您使用自己的私有注册表时才需要。同一图像可以有多个标签：

```
$ docker tag shrikrishna/code.it:v1 shrikrishna/code.it:latest

```

### 提示

每当您给图像打标签时，请遵循`username/repository:tag`约定。

现在，再次运行`docker images`命令将显示相同的图像已被标记为`v1`和`latest`命令：

```
$ docker images
REPOSITORY            TAG     IMAGE ID      CREATED     VIRTUAL SIZE
shrikrishna/code.it   v1      a7cb6737a2f6  8 days ago  704.4 MB
shrikrishna/code.it   latest  a7cb6737a2f6  8 days ago  704.4 MB

```

## 登录命令

`login`命令用于注册或登录到 Docker 注册服务器。如果未指定服务器，默认为[`index.docker.io/v1/`](https://index.docker.io/v1/)。

```
$ Docker login [OPTIONS] [SERVER]

```

| 标志 | 解释 |
| --- | --- |

|

```
-e, --email=""

```

| 电子邮件 |
| --- |

|

```
-p, --password=""

```

| 密码 |
| --- |

|

```
-u, --username=""

```

| 用户名 |
| --- |

如果未提供标志，则服务器将提示您提供详细信息。第一次登录后，详细信息将存储在`$HOME/.dockercfg`路径中。

## 推送命令

`push`命令用于将图像推送到公共图像注册表或私有 Docker 注册表：

```
$ docker push NAME[:TAG]

```

## 历史命令

`history`命令显示图像的历史记录：

```
$ docker history shykes/nodejs
IMAGE         CREATED        CREATED BY                      SIZE
6592508b0790  15 months ago  /bin/sh -c wget http://nodejs.  15.07 MB
0a2ff988ae20  15 months ago  /bin/sh -c apt-get install ...  25.49 MB
43c5d81f45de  15 months ago  /bin/sh -c apt-get update       96.48 MB
b750fe79269d  16 months ago  /bin/bash                       77 B
27cf78414709  16 months ago                                  175.3 MB

```

## 事件命令

一旦启动，`events`命令会实时打印`docker`守护程序处理的所有事件：

```
$ docker events [OPTIONS]

```

| 标志 | 解释 |
| --- | --- |

|

```
--since=""

```

| 这显示自 Unix 时间戳以来创建的所有事件。 |
| --- |

|

```
--until=""

```

| 这个流事件直到时间戳。 |
| --- |

例如，`events`命令的使用如下：

```
$ docker events

```

现在，在另一个标签中，运行以下命令：

```
$ docker start code.it

```

然后运行以下命令：

```
$ docker stop code.it

```

现在回到运行 Docker 事件的标签并查看输出。它将沿着这些线路进行：

```
[2014-07-21 21:31:50 +0530 IST] c7f2485863b2c7d0071477e6cb8c8301021ef9036afd4620702a0de08a4b3f7b: (from dockerfile/nodejs:latest) start

[2014-07-21 21:31:57 +0530 IST] c7f2485863b2c7d0071477e6cb8c8301021ef9036afd4620702a0de08a4b3f7b: (from dockerfile/nodejs:latest) stop

[2014-07-21 21:31:57 +0530 IST] c7f2485863b2c7d0071477e6cb8c8301021ef9036afd4620702a0de08a4b3f7b: (from dockerfile/nodejs:latest) die

```

您可以使用`--since`和`--until`等标志来获取特定时间范围内的事件日志。

## 等待命令

`wait`命令会阻塞，直到容器停止，然后打印其退出代码：

```
$ docker wait CONTAINER(s)

```

## 构建命令

构建命令从指定路径的源文件构建镜像：

```
$ Docker build [OPTIONS] PATH | URL | -

```

| 标志 | 解释 |
| --- | --- |

|

```
-t, --tag=""

```

| 这是要应用于成功时生成的图像的存储库名称（和可选标签）。 |
| --- |

|

```
-q, --quiet

```

| 这会抑制默认情况下冗长的输出。 |
| --- |

|

```
--rm=true

```

| 这会在成功构建后删除中间容器。 |
| --- |

|

```
--force-rm

```

| 这总是在构建失败后删除中间容器。 |
| --- |

|

```
--no-cache

```

| 此命令在构建镜像时不使用缓存。 |
| --- |

此命令使用 Dockerfile 和上下文来构建 Docker 镜像。

Dockerfile 就像一个 Makefile。它包含了各种配置和命令的指令，需要运行以创建一个镜像。我们将在下一节中讨论编写 Dockerfiles。

### 提示

最好先阅读关于 Dockerfiles 的部分，然后再回到这里，以更好地理解这个命令以及它是如何工作的。

在`PATH`或`URL`路径下的文件被称为构建的**上下文**。上下文用于指代 Dockerfile 中的文件或文件夹，例如在`ADD`指令中（这就是为什么诸如`ADD ../file.txt`这样的指令不起作用。它不在上下文中！）。

当给出 GitHub URL 或带有`git://`协议的 URL 时，该存储库将被用作上下文。该存储库及其子模块将递归克隆到您的本地机器，然后作为上下文上传到`docker`守护程序。这允许您在私人 Git 存储库中拥有 Dockerfiles，您可以从本地用户凭据或**虚拟****私人****网络**（**VPN**）访问。

## 上传到 Docker 守护程序

请记住，Docker 引擎既有`docker`守护程序又有 Docker 客户端。您作为用户给出的命令是通过 Docker 客户端，然后再与`docker`守护程序（通过 TCP 或 Unix 套接字）进行通信，它会执行必要的工作。`docker`守护程序和 Docker 主机可以在不同的主机上（这是 boot2Docker 的前提），并且`DOCKER_HOST`环境变量设置为远程`docker`守护程序的位置。

当您为`docker build`命令提供上下文时，本地目录中的所有文件都会被打包并发送到`docker`守护程序。`PATH`变量指定了在`docker`守护程序中构建上下文的文件的位置。因此，当您运行`docker build .`时，当前文件夹中的所有文件都会被上传，而不仅仅是 Dockerfile 中列出要添加的文件。

由于这可能会有些问题（因为一些系统如 Git 和一些 IDE 如 Eclipse 会创建隐藏文件夹来存储元数据），Docker 提供了一种机制来忽略某些文件或文件夹，方法是在`PATH`变量中创建一个名为`.dockerignore`的文件，并添加必要的排除模式。例如，查看[`github.com/docker/docker/blob/master/.dockerignore`](https://github.com/docker/docker/blob/master/.dockerignore)。

如果提供了一个普通的 URL，或者 Dockerfile 通过`stdin`文件流传输，那么不会设置上下文。在这些情况下，只有当`ADD`指令引用远程 URL 时才起作用。

现在让我们通过 Dockerfile 构建`code.it`示例图像。如何创建这个 Dockerfile 的说明在*Dockerfile*部分提供。

到目前为止，您已经创建了一个目录，并在其中放置了 Dockerfile。现在，在您的终端上，转到该目录并执行`docker build`命令：

```
$ docker build -t shrikrishna/code.it:docker Dockerfile .
Sending build context to Docker daemon  2.56 kB
Sending build context to Docker daemon
Step 0 : FROM Dockerfile/nodejs
---> 1535da87b710
Step 1 : MAINTAINER Shrikrishna Holla <s**a@gmail.com>
---> Running in e4be61c08592
---> 4c0eabc44a95
Removing intermediate container e4be61c08592
Step 2 : WORKDIR /home
---> Running in 067e8951cb22
---> 81ead6b62246
Removing intermediate container 067e8951cb22
. . . . .
. . . . .
Step 7 : EXPOSE  8000
---> Running in 201e07ec35d3
---> 1db6830431cd
Removing intermediate container 201e07ec35d3
Step 8 : WORKDIR /home
---> Running in cd128a6f090c
---> ba05b89b9cc1
Removing intermediate container cd128a6f090c
Step 9 : CMD     ["/usr/bin/node", "/home/code.it/app.js"]
---> Running in 6da5d364e3e1
---> 031e9ed9352c
Removing intermediate container 6da5d364e3e1
Successfully built 031e9ed9352c

```

现在，您将能够在 Docker 镜像的输出中查看您新构建的图像

```
REPOSITORY          TAG        IMAGE ID     CREATED      VIRTUAL SIZE
shrikrishna/code.it Dockerfile 031e9ed9352c 21 hours ago 1.02 GB

```

要查看缓存的实际效果，请再次运行相同的命令

```
$ docker build -t shrikrishna/code.it:dockerfile .
Sending build context to Docker daemon  2.56 kB
Sending build context to Docker daemon
Step 0 : FROM dockerfile/nodejs
---> 1535da87b710
Step 1 : MAINTAINER Shrikrishna Holla <s**a@gmail.com>
---> Using cache
---> 4c0eabc44a95
Step 2 : WORKDIR /home
---> Using cache
---> 81ead6b62246
Step 3 : RUN     git clone https://github.com/shrikrishnaholla/code.it.git
---> Using cache
---> adb4843236d4
Step 4 : WORKDIR code.it
---> Using cache
---> 755d248840bb
Step 5 : RUN     git submodule update --init --recursive
---> Using cache
---> 2204a519efd3
Step 6 : RUN     npm install
---> Using cache
---> 501e028d7945
Step 7 : EXPOSE  8000
---> Using cache
---> 1db6830431cd
Step 8 : WORKDIR /home
---> Using cache
---> ba05b89b9cc1
Step 9 : CMD     ["/usr/bin/node", "/home/code.it/app.js"]
---> Using cache
---> 031e9ed9352c
Successfully built 031e9ed9352c

```

### 提示

现在尝试使用缓存。更改中间的一行（例如端口号），或者在中间的某个地方添加一个`RUN echo "testing cache"`行，看看会发生什么。

使用存储库 URL 构建图像的示例如下：

```
$ docker build -t shrikrishna/optimus:git_url \ git://github.com/shrikrishnaholla/optimus
Sending build context to Docker daemon 1.305 MB
Sending build context to Docker daemon
Step 0 : FROM        dockerfile/nodejs
---> 1535da87b710
Step 1 : MAINTAINER  Shrikrishna Holla
---> Running in d2aae3dba68c
---> 0e8636eac25b
Removing intermediate container d2aae3dba68c
Step 2 : RUN         git clone https://github.com/pesos/optimus.git /home/optimus
---> Running in 0b46e254e90a
. . . . .
. . . . .
. . . . .
Step 5 : CMD         ["/usr/local/bin/npm", "start"]
---> Running in 0e01c71faa0b
---> 0f0dd3deae65
Removing intermediate container 0e01c71faa0b
Successfully built 0f0dd3deae65

```

# Dockerfile

我们已经看到了通过提交容器来创建镜像。如果您想要使用依赖项的新版本或您自己应用程序的新版本来更新镜像怎么办？反复启动、设置和提交步骤很快就变得不切实际。我们需要一种可重复的方法来构建镜像。这就是 Dockerfile 的作用，它不过是一个包含指令的文本文件，用于自动化构建镜像的步骤。`docker build`将按顺序读取这些指令，途中提交它们，并构建一个镜像。

`docker build`命令使用此 Dockerfile 和上下文来执行指令，并构建 Docker 镜像。上下文是指提供给`docker build`命令的路径或源代码仓库 URL。

Dockerfile 以这种格式包含指令：

```
# Comment
INSTRUCTION arguments

```

任何以`#`开头的行将被视为注释。如果`#`符号出现在其他地方，它将被视为参数的一部分。指令不区分大小写，尽管按照惯例，指令应该大写以便与参数区分开。

让我们看看在 Dockerfile 中可以使用的指令。

## `FROM`指令

`FROM`指令设置了后续指令的基础镜像。有效的 Dockerfile 的第一行非注释行将是一个`FROM`指令：

```
FROM <image>:<tag>

```

镜像可以是任何有效的本地或公共镜像。如果在本地找不到，`Docker build`命令将尝试从公共注册表中拉取。这里`tag`命令是可选的。如果没有给出，将假定为`latest`命令。如果给出了不正确的`tag`命令，将返回错误。

## `MAINTAINER`指令

`MAINTAINER`指令允许您为生成的镜像设置作者：

```
MAINTAINER <name>

```

## `RUN`指令

`RUN`指令将在当前镜像的新层上执行任何命令，并提交此镜像。因此提交的镜像将用于 Dockerfile 中的下一条指令。

`RUN`指令有两种形式：

+   `RUN <command>`形式

+   `RUN ["executable", "arg1", "arg2"...]`形式

在第一种形式中，命令在 shell 中运行，具体来说是`/bin/sh -c <command>` shell。第二种形式在基础镜像没有`/bin/sh` shell 的情况下很有用。Docker 对这些镜像构建使用缓存。因此，如果您的镜像构建在中间某个地方失败，下一次运行将重用先前成功的部分构建，并从失败的地方继续。

在以下情况下，缓存将被使无效：

+   当使用`--no-cache`标志运行`docker build`命令时。

+   如果给出了诸如`apt-get update`之类的不可缓存命令，则所有后续的`RUN`指令将再次运行。

+   当首次遇到`ADD`指令时，如果上下文的内容发生了变化，将使 Dockerfile 中所有后续指令的缓存无效。这也将使`RUN`指令的缓存无效。

## CMD 指令

`CMD`指令提供了容器执行的默认命令。它有以下形式：

+   `CMD ["executable", "arg1", "arg2"...]`形式

+   `CMD ["arg1", "arg2"...]`形式

+   `CMD command arg1 arg2 …`形式

第一种形式类似于一个 exec，这是首选形式，其中第一个值是可执行文件的路径，后面跟着它的参数。

第二种形式省略了可执行文件，但需要`ENTRYPOINT`指令来指定可执行文件。

如果您使用`CMD`指令的 shell 形式，那么`<command>`命令将在`/bin/sh -c` shell 中执行。

### 注意

如果用户在`docker run`中提供了一个命令，它将覆盖`CMD`命令。

`RUN`和`CMD`指令之间的区别在于，`RUN`指令实际上运行命令并提交它，而`CMD`指令在构建时不会被执行。这是一个默认的命令，在用户启动容器时运行，除非用户提供了一个启动命令。

例如，让我们编写一个`Dockerfile`，将`Star Wars`的输出带到您的终端：

```
FROM ubuntu:14.04
MAINTAINER shrikrishna
RUN apt-get -y install telnet
CMD ["/usr/bin/telnet", "towel.blinkenlights.nl"]

```

将其保存在名为`star_wars`的文件夹中，并在此位置打开您的终端。然后运行此命令：

```
$ docker build -t starwars .

```

现在您可以使用以下命令运行它：

```
$ docker run -it starwars

```

以下截图显示了`starwars`的输出：

![CMD 指令](img/4787OS_02_07.jpg)

因此，您可以在终端上观看**星球大战**！

### 注意

这个*星球大战*致敬是由 Simon Jansen，Sten Spans 和 Mike Edwards 创建的。当您已经看够时，按住*Ctrl* + *]*。您将收到一个提示，您可以在其中输入`close`以退出。

## ENTRYPOINT 指令

`ENTRYPOINT`指令允许你将 Docker 镜像变成一个可执行文件。换句话说，当你在`ENTRYPOINT`中指定一个可执行文件时，容器将运行得就像是那个可执行文件一样。

`ENTRYPOINT`指令有两种形式：

1.  `ENTRYPOINT ["executable", "arg1", "arg2"...]`形式。

1.  `ENTRYPOINT command arg1 arg2 …`形式。

这个指令添加了一个入口命令，当参数传递给`docker run`命令时，不会被覆盖，不像`CMD`指令的行为。这允许参数传递给`ENTRYPOINT`指令。`docker run <image> -arg`命令将`-arg`参数传递给`ENTRYPOINT`指令中指定的命令。

如果在`ENTRYPOINT`指令中指定了参数，它们不会被`docker run`的参数覆盖，但是通过`CMD`指令指定的参数会被覆盖。

例如，让我们编写一个带有`cowsay`的`ENTRYPOINT`指令的 Dockerfile：

### 注意

`cowsay`是一个生成带有消息的牛的 ASCII 图片的程序。它还可以使用其他动物的预制图片生成图片，比如 Tux 企鹅，Linux 吉祥物。

```
FROM ubuntu:14.04
RUN apt-get -y install cowsay
ENTRYPOINT ["/usr/games/cowsay"]
CMD ["Docker is so awesomoooooooo!"]

```

将其保存为名为`Dockerfile`的文件，放在名为`cowsay`的文件夹中。然后通过终端，进入该目录，并运行以下命令：

```
$ docker build -t cowsay .

```

构建完镜像后，运行以下命令：

```
$ docker run cowsay

```

以下截图显示了前面命令的输出：

![ENTRYPOINT 指令](img/4787OS_02_08.jpg)

如果你仔细看截图，第一次运行没有参数，并且使用了我们在 Dockerfile 中配置的参数。然而，当我们在第二次运行中给出自己的参数时，它覆盖了默认值，并将所有参数（`-f`标志和句子）传递给了`cowsay`文件夹。

### 注意

如果你是那种喜欢恶作剧的人，这里有一个提示：应用[`superuser.com/a/175802`](http://superuser.com/a/175802)中给出的指令来设置一个预执行脚本（每次执行命令时调用的函数），将每个命令传递给这个 Docker 容器，并将其放在`.bashrc`文件中。现在，cowsay 将打印出它在文本气球中执行的每个命令，由一个 ASCII 牛说出来！

## WORKDIR 指令

`WORKDIR`指令为接下来的`RUN`，`CMD`和`ENTRYPOINT` Dockerfile 命令设置工作目录：

```
WORKDIR /path/to/working/directory

```

此指令可以在同一个 Dockerfile 中多次使用。如果提供了相对路径，则`WORKDIR`指令将相对于先前的`WORKDIR`指令的路径。

## EXPOSE 指令

`EXPOSE`指令通知 Docker 在启动容器时要公开某个端口：

```
EXPOSE port1 port2 …

```

即使在暴露端口之后，在启动容器时，仍然需要使用`-p`标志来提供端口映射给`Docker run`。这个指令在链接容器时很有用，我们将在第三章中看到*链接容器*。

## ENV 指令

ENV 命令用于设置环境变量：

```
ENV <key> <value>

```

这将把`<key>`环境变量设置为`<value>`。这个值将传递给所有未来的`RUN`指令。这相当于在命令前加上`<key>=<value>`。

使用`ENV`命令设置的环境变量将持久存在。这意味着当从生成的镜像运行容器时，环境变量也将对运行的进程可用。`docker inspect`命令显示了在创建镜像过程中分配的值。但是，可以使用`$ docker run –env <key>=<value>`命令覆盖这些值。

## USER 指令

USER 指令设置在运行镜像和任何后续`RUN`指令时要使用的用户名或 UID：

```
USER xyz

```

## VOLUME 指令

`VOLUME`指令将创建一个具有给定名称的挂载点，并将其标记为保存来自主机或其他容器的外部挂载卷：

```
VOLUME [path]

```

以下是`VOLUME`指令的示例：

```
VOLUME ["/data"]

```

以下是此指令的另一个示例：

```
VOLUME /var/log

```

两种格式都可以接受。

## ADD 指令

`ADD`指令用于将文件复制到镜像中：

```
ADD <src> <dest>

```

`ADD`指令将文件从`<src>`复制到`<dest>`的路径中。

`<src>`路径必须是相对于正在构建的源目录（也称为构建上下文）的文件或目录的路径，或者是远程文件 URL。

`<dest>`路径是源将被复制到目标容器内部的绝对路径。

### 注意

如果通过`stdin`文件（`docker build - <` `somefile`）构建 Dockerfile，则没有构建上下文，因此 Dockerfile 只能包含基于 URL 的`ADD`语句。您还可以通过`stdin`文件（`docker build - <` `archive.tar.gz`）传递压缩存档。Docker 将在存档的根目录查找 Dockerfile，并且存档的其余部分将用作构建的上下文。

`ADD`指令遵循以下规则：

+   `<src>`路径必须在构建的上下文中。您不能使用`ADD ../file as ..`语法，因为它超出了上下文。

+   如果`<src>`是一个 URL，并且`<dest>`路径不以斜杠结尾（它是一个文件），则将 URL 处的文件复制到`<dest>`路径。

+   如果`<src>`是一个 URL，并且`<dest>`路径以斜杠结尾（它是一个目录），则会获取 URL 处的内容，并且会从 URL 中推断出一个文件名，并将其保存到`<dest>/filename`路径中。因此，在这种情况下，URL 不能具有简单的路径，例如`example.com`。

+   如果`<src>`是一个目录，则整个目录将被复制，连同文件系统元数据一起。

+   如果`<src>`是本地 tar 存档，则它将被提取到`<dest>`路径中。`<dest>`处的结果是：

+   `<dest>`路径处存在的任何内容。

+   提取的 tar 存档的内容，以文件为基础解决冲突，优先考虑`<src>`路径。

+   如果`<dest>`路径不存在，则将创建该路径以及其路径中的所有缺失目录。

## COPY 指令

COPY 指令将文件复制到镜像中：

```
COPY <src> <dest>

```

`COPY`指令类似于`ADD`指令。不同之处在于`COPY`指令不允许超出上下文的任何文件。因此，如果您通过`stdin`文件或 URL（指向源代码存储库的 URL）流式传输 Dockerfile，则无法使用`COPY`指令。

## ONBUILD 指令

`ONBUILD`指令将触发器添加到镜像中，当镜像用作另一个构建的基础镜像时，将执行该触发器。

```
ONBUILD [INSTRUCTION]

```

当源应用程序涉及需要在使用之前编译的生成器时，这是有用的。除了`FROM`，`MAINTAINER`和`ONBUILD`指令之外的任何构建指令都可以注册。

以下是此指令的工作方式：

1.  在构建过程中，如果遇到`ONBUILD`指令，它会注册一个触发器并将其添加到镜像的元数据中。当前构建不会以任何方式受到影响。

1.  所有这些触发器的列表都被添加到镜像清单中，作为一个名为`OnBuild`的键，在构建结束时可以通过`Docker inspect`命令看到。

1.  当这个镜像后来被用作新构建的基础镜像时，在处理`FROM`指令的过程中，`OnBuild key`触发器按照注册的顺序被读取和执行。如果其中任何一个失败，`FROM`指令将中止，导致构建失败。否则，`FROM`指令完成，构建将继续进行。

1.  触发器在执行后会从最终镜像中清除。换句话说，它们不会被*grand-child builds*继承。

让我们把`cowsay`带回来！这是一个带有`ONBUILD`指令的 Dockerfile：

```
FROM ubuntu:14.04
RUN apt-get -y install cowsay
RUN apt-get -y install fortune
ENTRYPOINT ["/usr/games/cowsay"]
CMD ["Docker is so awesomoooooooo!"]
ONBUILD RUN /usr/games/fortune | /usr/games/cowsay

```

现在将这个文件保存在一个名为`OnBuild`的文件夹中，打开该文件夹中的终端，并运行这个命令：

```
$ Docker build -t shrikrishna/onbuild .

```

我们需要编写另一个基于这个镜像的 Dockerfile。让我们写一个：

```
FROM shrikrishna/onbuild
RUN  apt-get moo
CMD ['/usr/bin/apt-get', 'moo']

```

### 注意

`apt-get moo`命令是许多开源工具中通常找到的彩蛋的一个例子，只是为了好玩！

构建这个镜像现在将执行我们之前给出的`ONBUILD`指令：

```
$ docker build -t shrikrishna/apt-moo apt-moo/
Sending build context to Docker daemon  2.56 kB
Sending build context to Docker daemon
Step 0 : FROM shrikrishna/onbuild
# Executing 1 build triggers
Step onbuild-0 : RUN /usr/games/fortune | /usr/games/cowsay
---> Running in 887592730f3d
________________________________
/ It was all so different before \
\ everything changed.            /
--------------------------------
\   ^__^
\  (oo)\_______
(__)\       )\/\
||----w |
||     ||
---> df01e4ca1dc7
---> df01e4ca1dc7
Removing intermediate container 887592730f3d
Step 1 : RUN  apt-get moo
---> Running in fc596cb91c2a
(__)
(oo)
/------\/
/ |    ||
*  /\---/\
~~   ~~
..."Have you mooed today?"...
---> 623cd16a51a7
Removing intermediate container fc596cb91c2a
Step 2 : CMD ['/usr/bin/apt-get', 'moo']
---> Running in 22aa0b415af4
---> 7e03264fbb76
Removing intermediate container 22aa0b415af4
Successfully built 7e03264fbb76

```

现在让我们利用我们新获得的知识来为我们之前通过手动满足容器中的依赖关系并提交构建的`code.it`应用程序编写一个 Dockerfile。Dockerfile 看起来会像这样：

```
# Version 1.0
FROM dockerfile/nodejs
MAINTAINER Shrikrishna Holla <s**a@gmail.com>

WORKDIR /home
RUN     git clone \ https://github.com/shrikrishnaholla/code.it.git

WORKDIR code.it
RUN     git submodule update --init --recursive
RUN     npm install

EXPOSE  8000

WORKDIR /home
CMD     ["/usr/bin/node", "/home/code.it/app.js"]

```

创建一个名为`code.it`的文件夹，并将这个内容保存为一个名为`Dockerfile`的文件。

### 注意

即使不需要上下文，为每个 Dockerfile 创建一个单独的文件夹是一个很好的做法。这样可以在不同的项目之间分离关注点。当你继续前进时，你可能会注意到许多 Dockerfile 作者会将`RUN`指令合并在一起（例如，查看[dockerfile.github.io](http://dockerfile.github.io)中的 Dockerfile）。原因是 AUFS 将可能的层的数量限制为 42。更多信息，请查看[`github.com/docker/docker/issues/1171`](https://github.com/docker/docker/issues/1171)。

你可以回到*Docker build*部分，看看如何从这个 Dockerfile 构建一个镜像。

# Docker 工作流程 - 拉取-使用-修改-提交-推送

现在，当我们接近本章的结束时，我们可以猜测一个典型的 Docker 工作流程是什么样的：

1.  准备运行应用程序的要求清单。

1.  确定哪个公共图像（或您自己的图像）可以满足大多数要求，同时也要维护良好（这很重要，因为您需要图像在可用时更新为新版本）。

1.  接下来，通过运行容器并执行满足要求的命令（可以是安装依赖项、绑定挂载卷或获取源代码），或者编写 Dockerfile（这更可取，因为您将能够使构建可重复）来满足其余要求。

1.  将新图像推送到公共 Docker 注册表，以便社区也可以使用它（或者根据需要推送到私有注册表或存储库）。

# 自动构建

自动构建自动化了从 GitHub 或 BitBucket 直接在 Docker Hub 上构建和更新图像。它们通过向您选择的 GitHub 或 BitBucket 存储库添加`commit`挂钩来工作，在您推送提交时触发构建和更新。因此，每次更新时都不需要手动构建和推送图像到 Docker Hub。以下步骤将向您展示如何执行此操作：

1.  要设置自动构建，请登录到您的 Docker Hub 帐户。![自动构建](img/4787OS_02_03.jpg)

1.  通过**链接** **帐户**菜单链接您的 GitHub 或 BitBucket 帐户。

1.  在**添加** **存储库**菜单中选择**自动** **构建**。![自动构建](img/4787OS_02_04.jpg)

1.  选择包含您想要构建的 Dockerfile 的 GitHub 或 BitBucket 项目。（您需要授权 Docker Hub 访问您的存储库。）

1.  选择包含源代码和 Dockerfile 的分支（默认为主分支）。

1.  为自动构建命名。这也将是存储库的名称。

1.  为构建分配一个可选的 Docker 标签。默认为`lastest`标签。

1.  指定 Dockerfile 的位置。默认为`/`。![自动构建](img/4787OS_02_05.jpg)

一旦配置完成，自动构建将触发构建，并且您将能够在几分钟内在 Docker Hub 注册表中看到它。它将与您的 GitHub 和 BitBucket 存储库保持同步，直到您自己停用自动构建。

构建状态和历史记录可以在 Docker Hub 中您的个人资料的自动构建页面中查看。

![自动构建](img/4787OS_02_06.jpg)

创建自动构建后，您可以停用或删除它。

### 注意

但是，您不能使用 Docker 的`push`命令推送到自动构建。您只能通过向 GitHub 或 BitBucket 存储库提交代码来管理它。

您可以为每个存储库创建多个自动构建，并将它们配置为指向特定的 Dockerfile 或 Git 分支。

## 构建触发器

也可以通过 Docker Hub 上的 URL 触发自动构建。这允许您根据需要重新构建自动构建的图像。

## Webhooks

Webhooks 是在成功构建事件发生时调用的触发器。通过 webhook，您可以指定目标 URL（例如通知您的服务）和在推送图像时将传递的 JSON 有效负载。如果您有持续集成工作流程，webhooks 非常有用。

要将 webhook 添加到您的 Github 存储库，请按照以下步骤进行：

1.  转到存储库中的**Settings**。![Webhooks](img/4787OS_02_09.jpg)

1.  从左侧菜单栏转到**Webhooks** **and** **Services**。![Webhooks](img/4787OS_02_10.jpg)

1.  单击**Add** **Service**。![Webhooks](img/4787OS_02_11.jpg)

1.  在打开的文本框中，输入**Docker**并选择该服务。![Webhooks](img/4787OS_02_12.jpg)

1.  您已经准备就绪！现在每当您提交到存储库时，Docker Hub 都会触发构建。

# 摘要

在本章中，我们查看了**Docker**命令行工具并尝试了可用的命令。然后，我们找出如何使用 Dockerfile 使构建可重复。此外，我们使用 Docker Hub 的自动构建服务自动化了此构建过程。

在下一章中，我们将尝试通过查看帮助我们配置它们的各种命令来更好地控制容器的运行方式。我们将研究限制容器可消耗的资源（CPU、RAM 和存储）的数量。

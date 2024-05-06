# 数据卷和配置

在上一章中，我们学习了如何构建和共享我们自己的容器镜像。特别关注了如何通过只包含容器化应用程序真正需要的构件来尽可能地减小镜像的大小。

在本章中，我们将学习如何处理有状态的容器，即消耗和产生数据的容器。我们还将学习如何使用环境变量和配置文件在运行时和构建时配置容器。

以下是我们将讨论的主题列表：

+   创建和挂载数据卷

+   在容器之间共享数据

+   使用主机卷

+   在镜像中定义卷

+   配置容器

完成本章后，您将能够做到以下事项：

+   创建、删除和列出数据卷。

+   将现有的数据卷挂载到容器中。

+   在容器内部使用数据卷创建持久化数据。

+   使用数据卷在多个容器之间共享数据。

+   使用数据卷将任何主机文件夹挂载到容器中。

+   定义容器访问数据卷中数据的访问模式（读/写或只读）。

+   为在容器中运行的应用程序配置环境变量。

+   通过使用构建参数对`Dockerfile`进行参数化。

# 技术要求

在本章中，您需要在您的机器上安装 Docker Toolbox 或者访问在您的笔记本电脑或云中运行 Docker 的 Linux 虚拟机（VM）。此外，最好在您的机器上安装 Docker for Desktop。本章没有附带任何代码。

# 创建和挂载数据卷

所有有意义的应用程序都会消耗或产生数据。然而，容器最好是无状态的。我们该如何处理这个问题呢？一种方法是使用 Docker 卷。卷允许容器消耗、产生和修改状态。卷的生命周期超出了容器的生命周期。当使用卷的容器死亡时，卷仍然存在。这对状态的持久性非常有利。

# 修改容器层

在我们深入讨论卷之前，让我们首先讨论一下如果容器中的应用程序更改了容器文件系统中的内容会发生什么。在这种情况下，更改都发生在我们在《精通容器》第三章中介绍的可写容器层中。我们可以通过运行一个容器并在其中执行一个创建新文件的脚本来快速演示这一点，就像这样：

```
$ docker container run --name demo \
 alpine /bin/sh -c 'echo "This is a test" > sample.txt'
```

上述命令创建了一个名为`demo`的容器，并在该容器内创建了一个名为`sample.txt`的文件，内容为`This is a test`。运行`echo`命令后容器退出，但仍保留在内存中，供我们进行调查。让我们使用`diff`命令来查找容器文件系统中与原始镜像文件系统相关的更改，如下所示：

```
$ docker container diff demo
```

输出应该如下所示：

```
A /sample.txt
```

显然，如`A`所示，容器的文件系统中已经添加了一个新文件，这是预期的。由于所有源自基础镜像（在本例中为`alpine`）的层都是不可变的，更改只能发生在可写容器层中。

与原始镜像相比发生了变化的文件将用`C`标记，而已删除的文件将用`D`标记。

如果我们现在从内存中删除容器，它的容器层也将被删除，所有更改将被不可逆转地删除。如果我们需要我们的更改持久存在，甚至超出容器的生命周期，这不是一个解决方案。幸运的是，我们有更好的选择，即 Docker 卷。让我们来了解一下它们。

# 创建卷

由于在这个时候，在 macOS 或 Windows 计算机上使用 Docker for Desktop 时，容器并不是在 macOS 或 Windows 上本地运行，而是在 Docker for Desktop 创建的（隐藏的）VM 中运行，为了说明问题，最好使用`docker-machine`来创建和使用运行 Docker 的显式 VM。在这一点上，我们假设您已经在系统上安装了 Docker Toolbox。如果没有，请返回到第二章《设置工作环境》中，我们提供了如何安装 Toolbox 的详细说明：

1.  使用`docker-machine`列出当前在 VirtualBox 中运行的所有虚拟机，如下所示：

```
$ docker-machine ls 
```

1.  如果您的列表中没有名为`node-1`的 VM，请使用以下命令创建一个：

```
$ docker-machine create --driver virtualbox node-1 
```

如果您在启用了 Hyper-V 的 Windows 上运行，可以参考第二章 *设置工作环境*中的内容，了解如何使用`docker-machine`创建基于 Hyper-V 的 VM。

1.  另一方面，如果您有一个名为`node-1`的 VM，但它没有运行，请按以下方式启动它：

```
$ docker-machine start node-1
```

1.  现在一切准备就绪，使用`docker-machine`以这种方式 SSH 到这个 VM：

```
$ docker-machine ssh node-1
```

1.  您应该会看到这个欢迎图片：

![](img/223cb246-5c36-42aa-8905-22913d6642ba.png)

docker-machine VM 欢迎消息

1.  要创建一个新的数据卷，我们可以使用`docker volume create`命令。这将创建一个命名卷，然后可以将其挂载到容器中，用于持久数据访问或存储。以下命令创建一个名为`sample`的卷，使用默认卷驱动程序：

```
$ docker volume create sample 
```

默认的卷驱动程序是所谓的本地驱动程序，它将数据存储在主机文件系统中。

1.  找出主机上存储数据的最简单方法是使用`docker volume inspect`命令查看我们刚刚创建的卷。实际位置可能因系统而异，因此这是找到目标文件夹的最安全方法。您可以在以下代码块中看到这个命令：

```
$ docker volume inspect sample [ 
    { 
        "CreatedAt": "2019-08-02T06:59:13Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/mnt/sda1/var/lib/docker/volumes/sample/_data",
        "Name": "my-data",
        "Options": {},
        "Scope": "local"
    } 
] 
```

主机文件夹可以在输出中的`Mountpoint`下找到。在我们的情况下，当使用基于 LinuxKit 的 VM 在 VirtualBox 中运行`docker-machine`时，文件夹是`/mnt/sda1/var/lib/docker/volumes/sample/_data`。

目标文件夹通常是受保护的文件夹，因此我们可能需要使用`sudo`来导航到这个文件夹并在其中执行任何操作。

在我们基于 LinuxKit 的 VM 中，Docker Toolbox 中，访问也被拒绝，但我们也没有`sudo`。我们的探索到此为止了吗？

幸运的是，我已经准备了一个`fundamentalsofdocker/nsenter`实用程序容器，允许我们访问我们之前创建的`sample`卷的后备文件夹。

1.  我们需要以`privileged`模式运行此容器，以访问文件系统的受保护部分，就像这样：

```
$ docker run -it --rm --privileged --pid=host \
 fundamentalsofdocker/nsenter / #
```

我们正在使用`--privileged`标志运行容器。这意味着在容器中运行的任何应用程序都可以访问主机的设备。`--pid=host`标志表示容器被允许访问主机的进程树（Docker 守护程序运行的隐藏 VM）。现在，前面的容器运行 Linux `nsenter`工具以进入主机的 Linux 命名空间，然后在其中运行一个 shell。通过这个 shell，我们因此被授予对主机管理的所有资源的访问权限。

在运行容器时，我们基本上在容器内执行以下命令：

`nsenter -t 1 -m -u -n -i sh`

如果这对你来说听起来很复杂，不用担心；随着我们在本书中的学习，你会更多地理解。如果有一件事可以让你受益，那就是意识到正确使用容器可以有多强大。

1.  在容器内部，我们现在可以导航到代表卷挂载点的文件夹，然后列出其内容，如下所示：

```
/ # cd /mnt/sda1/var/lib/docker/volumes/sample/_data
/ # ls -l total 0
```

由于我们尚未在卷中存储任何数据，该文件夹目前为空。

1.  通过按下*Ctrl* + *D*退出工具容器。

还有其他来自第三方的卷驱动程序，以插件的形式提供。我们可以在`create`命令中使用`--driver`参数来选择不同的卷驱动程序。其他卷驱动程序使用不同类型的存储系统来支持卷，例如云存储、网络文件系统（NFS）驱动、软件定义存储等。然而，正确使用其他卷驱动程序的讨论超出了本书的范围。

# 挂载卷

一旦我们创建了一个命名卷，我们可以按照以下步骤将其挂载到容器中：

1.  为此，我们可以在`docker container run`命令中使用`-v`参数，如下所示：

```
$ docker container run --name test -it \
 -v sample:/data \
    alpine /bin/sh Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
050382585609: Pull complete
Digest: sha256:6a92cd1fcdc8d8cdec60f33dda4db2cb1fcdcacf3410a8e05b3741f44a9b5998
Status: Downloaded newer image for alpine:latest
/ #
```

上述命令将`sample`卷挂载到容器内的`/data`文件夹。

1.  在容器内，我们现在可以在`/data`文件夹中创建文件，然后退出，如下所示：

```
/ # cd /data / # echo "Some data" > data.txt 
/ # echo "Some more data" > data2.txt 
/ # exit
```

1.  如果我们导航到包含卷数据的主机文件夹并列出其内容，我们应该看到我们刚刚在容器内创建的两个文件（记住：我们需要使用`fundamentalsofdocker/nsenter`工具容器来这样做），如下所示：

```
$ docker run -it --rm --privileged --pid=host \
 fundamentalsofdocker/nsenter
/ # cd /mnt/sda1/var/lib/docker/volumes/sample/_data
/ # ls -l 
total 8 
-rw-r--r-- 1 root root 10 Jan 28 22:23 data.txt
-rw-r--r-- 1 root root 15 Jan 28 22:23 data2.txt
```

1.  我们甚至可以尝试输出，比如说，第二个文件的内容，如下所示：

```
/ # cat data2.txt
```

1.  让我们尝试从主机在这个文件夹中创建一个文件，然后像这样使用另一个容器的卷：

```
/ # echo "This file we create on the host" > host-data.txt 
```

1.  通过按下*Ctrl* + *D*退出工具容器。

1.  现在，让我们删除`test`容器，并基于 CentOS 运行另一个容器。这次，我们甚至将我们的卷挂载到不同的容器文件夹`/app/data`中，就像这样：

```
$ docker container rm test
$ docker container run --name test2 -it \
 -v my-data:/app/data \
 centos:7 /bin/bash Unable to find image 'centos:7' locally
7: Pulling from library/centos
8ba884070f61: Pull complete
Digest: sha256:a799dd8a2ded4a83484bbae769d97655392b3f86533ceb7dd96bbac929809f3c
Status: Downloaded newer image for centos:7
[root@275c1fe31ec0 /]#
```

1.  一旦进入`centos`容器，我们可以导航到我们已经挂载卷的`/app/data`文件夹，并列出其内容，如下所示：

```
[root@275c1fe31ec0 /]# cd /app/data 
[root@275c1fe31ec0 /]# ls -l 
```

正如预期的那样，我们应该看到这三个文件：

```
-rw-r--r-- 1 root root 10 Aug 2 22:23 data.txt
-rw-r--r-- 1 root root 15 Aug 2 22:23 data2.txt
-rw-r--r-- 1 root root 32 Aug 2 22:31 host-data.txt
```

这是数据在 Docker 卷中持久存在超出容器生命周期的明确证据，也就是说，卷可以被其他甚至不同的容器重复使用，而不仅仅是最初使用它的容器。

重要的是要注意，在容器内部挂载 Docker 卷的文件夹被排除在 Union 文件系统之外。也就是说，该文件夹及其任何子文件夹内的每个更改都不会成为容器层的一部分，而是将持久保存在卷驱动程序提供的后备存储中。这一事实非常重要，因为当相应的容器停止并从系统中删除时，容器层将被删除。

1.  使用*Ctrl* + *D*退出`centos`容器。现在，再次按*Ctrl* + *D*退出`node-1`虚拟机。

# 删除卷

可以使用`docker volume rm`命令删除卷。重要的是要记住，删除卷会不可逆地销毁包含的数据，因此应该被视为危险命令。在这方面，Docker 在一定程度上帮助了我们，因为它不允许我们删除仍然被容器使用的卷。在删除卷之前，一定要确保要么有数据的备份，要么确实不再需要这些数据。让我们看看如何按照以下步骤删除卷：

1.  以下命令删除了我们之前创建的`sample`卷：

```
$ docker volume rm sample 
```

1.  执行上述命令后，仔细检查主机上的文件夹是否已被删除。

1.  为了清理系统，删除所有正在运行的容器，运行以下命令：

```
$ docker container rm -f $(docker container ls -aq)  
```

请注意，在用于删除容器的命令中使用`-v`或`--volume`标志，您可以要求系统同时删除与该特定容器关联的任何卷。当然，这只有在特定卷只被该容器使用时才有效。

在下一节中，我们将展示在使用 Docker for Desktop 时如何访问卷的后备文件夹。

# 访问使用 Docker for Desktop 创建的卷

按照以下步骤：

1.  让我们创建一个`sample`卷并使用我们的 macOS 或 Windows 机器上的 Docker for Desktop 进行检查，就像这样：

```
$ docker volume create sample
$ docker volume inspect sample
[
 {
 "CreatedAt": "2019-08-02T07:44:08Z",
 "Driver": "local",
 "Labels": {},
 "Mountpoint": "/var/lib/docker/volumes/sample/_data",
 "Name": "sample",
 "Options": {},
 "Scope": "local"
 }
]
```

`Mountpoint`显示为`/var/lib/docker/volumes/sample/_data`，但您会发现在您的 macOS 或 Windows 机器上没有这样的文件夹。原因是显示的路径是与 Docker for Windows 用于运行容器的隐藏 VM 相关的。此时，Linux 容器无法在 macOS 或 Windows 上本地运行。

1.  接下来，让我们从`alpine`容器内部生成两个带有卷数据的文件。要运行容器并将示例`volume`挂载到容器的`/data`文件夹，请使用以下代码：

```
$ docker container run --rm -it -v sample:/data alpine /bin/sh
```

1.  在容器内的`/data`文件夹中生成两个文件，就像这样：

```
/ # echo "Hello world" > /data/sample.txt
/ # echo "Other message" > /data/other.txt
```

1.  通过按*Ctrl + D*退出`alpine`容器。

如前所述，我们无法直接从我们的 macOS 或 Windows 访问`sample`卷的支持文件夹。这是因为该卷位于 macOS 或 Windows 上运行的隐藏 VM 中，该 VM 用于在 Docker for Desktop 中运行 Linux 容器。

要从我们的 macOS 访问隐藏的 VM，我们有两个选项。我们可以使用特殊容器并以特权模式运行它，或者我们可以使用`screen`实用程序来筛选 Docker 驱动程序。第一种方法也适用于 Windows 的 Docker。

1.  让我们从运行容器的`fundamentalsofdocker/nsenter`镜像开始尝试提到的第一种方法。我们在上一节中已经在使用这个容器。运行以下代码：

```
$ docker run -it --rm --privileged --pid=host fundamentalsofdocker/nsenter / #
```

1.  现在我们可以导航到支持我们`sample`卷的文件夹，就像这样：

```
/ # cd /var/lib/docker/volumes/sample/_data
```

通过运行此代码来查看此文件夹中有什么：

```
**/ # ls -l** 
total 8
-rw-r--r-- 1 root root 14 Aug 2 08:07 other.txt
-rw-r--r-- 1 root root 12 Aug 2 08:07 sample.txt
```

1.  让我们尝试从这个特殊容器内创建一个文件，然后列出文件夹的内容，如下所示：

```
/ # echo "I love Docker" > docker.txt
/ # ls -l total 12
-rw-r--r-- 1 root root 14 Aug 2 08:08 docker.txt
-rw-r--r-- 1 root root 14 Aug 2 08:07 other.txt
-rw-r--r-- 1 root root 12 Aug 2 08:07 sample.txt
```

现在，我们在`sample`卷的支持文件夹中有了文件。

1.  要退出我们的特权容器，只需按*Ctrl* + *D*。

1.  现在我们已经探索了第一种选项，如果您使用的是 macOS，让我们尝试`screen`工具，如下所示：

```
$ screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
```

1.  这样做，我们将会看到一个空屏幕。按*Enter*，将显示一个`docker-desktop:~#`命令行提示符。现在我们可以导航到卷文件夹，就像这样：

```
docker-desktop:~# cd /var/lib/docker/volumes/sample/_data
```

1.  让我们创建另一个带有一些数据的文件，然后列出文件夹的内容，如下所示：

```
docker-desktop:~# echo "Some other test" > test.txt 
docker-desktop:~# ls -l
total 16 -rw-r--r-- 1 root root 14 Aug 2 08:08 docker.txt -rw-r--r-- 1 root root 14 Aug 2 08:07 other.txt
-rw-r--r-- 1 root root 12 Aug 2 08:07 sample.txt
-rw-r--r-- 1 root root 16 Aug 2 08:10 test.txt
```

1.  要退出 Docker VM 的会话，请按*Ctrl* + *A* + *K*。

我们现在已经使用三种不同的方法创建了数据，如下所示：

+   +   从已挂载`sample`卷的容器内部。

+   使用特权文件夹来访问 Docker for Desktop 使用的隐藏虚拟机，并直接写入`sample`卷的后备文件夹。

+   仅在 macOS 上，使用`screen`实用程序进入隐藏的虚拟机，并直接写入`sample`卷的后备文件夹。

# 在容器之间共享数据

容器就像应用程序在其中运行的沙盒。这在很大程度上是有益的和需要的，以保护运行在不同容器中的应用程序。这也意味着对于在容器内运行的应用程序可见的整个文件系统对于这个应用程序是私有的，其他在不同容器中运行的应用程序不能干扰它。

有时，我们想要在容器之间共享数据。假设在容器 A 中运行的应用程序生成了一些数据，将被在容器 B 中运行的另一个应用程序使用。*我们该如何实现这一点？*好吧，我相信你已经猜到了——我们可以使用 Docker 卷来实现这一目的。我们可以创建一个卷，并将其挂载到容器 A，以及容器 B。这样，应用程序 A 和 B 都可以访问相同的数据。

现在，当多个应用程序或进程同时访问数据时，我们必须非常小心以避免不一致。为了避免并发问题，如竞争条件，理想情况下只有一个应用程序或进程创建或修改数据，而所有其他进程同时访问这些数据只读取它。我们可以通过将卷作为只读挂载来强制在容器中运行的进程只能读取卷中的数据。看一下以下命令：

```
$ docker container run -it --name writer \
 -v shared-data:/data \
 alpine /bin/sh
```

在这里，我们创建了一个名为`writer`的容器，它有一个卷`shared-data`，以默认的读/写模式挂载：

1.  尝试在这个容器内创建一个文件，就像这样：

```
# / echo "I can create a file" > /data/sample.txt 
```

它应该成功。

1.  退出这个容器，然后执行以下命令：

```
$ docker container run -it --name reader \
 -v shared-data:/app/data:ro \
 ubuntu:19.04 /bin/bash
```

我们有一个名为`reader`的容器，它有相同的卷挂载为**只读**(`ro`)。

1.  首先，确保你能看到在第一个容器中创建的文件，就像这样：

```
$ ls -l /app/data 
total 4
-rw-r--r-- 1 root root 20 Jan 28 22:55 sample.txt
```

1.  然后，尝试创建一个文件，就像这样：

```
# / echo "Try to break read/only" > /app/data/data.txt
```

它将失败，并显示以下消息：

```
bash: /app/data/data.txt: Read-only file system
```

1.  通过在命令提示符处输入`exit`来退出容器。回到主机上，让我们清理所有容器和卷，如下所示：

```
$ docker container rm -f $(docker container ls -aq) 
$ docker volume rm $(docker volume ls -q) 
```

1.  完成后，通过在命令提示符处输入 exit 退出 docker-machine VM。您应该回到您的 Docker for Desktop。使用 docker-machine 停止 VM，就像这样：

```
$ docker-machine stop node-1 
```

接下来，我们将展示如何将 Docker 主机中的任意文件夹挂载到容器中。

# 使用主机卷

在某些情况下，比如开发新的容器化应用程序或者容器化应用程序需要从某个文件夹中消耗数据——比如说——由传统应用程序产生，使用挂载特定主机文件夹的卷非常有用。让我们看下面的例子：

```
$ docker container run --rm -it \
 -v $(pwd)/src:/app/src \
 alpine:latest /bin/sh
```

前面的表达式交互式地启动一个带有 shell 的 alpine 容器，并将当前目录的 src 子文件夹挂载到容器的 /app/src。我们需要使用 $(pwd)（或者 ``pwd``，无论哪种方式），即当前目录，因为在使用卷时，我们总是需要使用绝对路径。

开发人员在他们在容器中运行的应用程序上工作时，经常使用这些技术，并希望确保容器始终包含他们对代码所做的最新更改，而无需在每次更改后重新构建镜像和重新运行容器。

让我们做一个示例来演示它是如何工作的。假设我们想要使用 nginx 创建一个简单的静态网站作为我们的 web 服务器，如下所示：

1.  首先，在主机上创建一个新的文件夹，我们将把我们的网页资产—如 HTML、CSS 和 JavaScript 文件—放在其中，并导航到它，就像这样：

```
$ mkdir ~/my-web 
$ cd ~/my-web 
```

1.  然后，我们创建一个简单的网页，就像这样：

```
$ echo "<h1>Personal Website</h1>" > index.html 
```

1.  现在，我们添加一个 `Dockerfile`，其中包含构建包含我们示例网站的镜像的说明。

1.  在文件夹中添加一个名为 `Dockerfile` 的文件，内容如下：

```
FROM nginx:alpine
COPY . /usr/share/nginx/html
```

Dockerfile 以最新的 Alpine 版本的 nginx 开始，然后将当前主机目录中的所有文件复制到 /usr/share/nginx/html 容器文件夹中。这是 nginx 期望网页资产位于的位置。

1.  现在，让我们用以下命令构建镜像：

```
$ docker image build -t my-website:1.0 . 
```

1.  最后，我们从这个镜像中运行一个容器。我们将以分离模式运行容器，就像这样：

```
$ docker container run -d \
 --name my-site \
 -p 8080:80 \
 my-website:1.0
```

注意 `-p 8080:80` 参数。我们还没有讨论这个，但我们将在第十章《单主机网络》中详细讨论。目前，只需知道这将把 nginx 监听传入请求的容器端口 80 映射到您的笔记本电脑的端口 8080，然后您可以访问应用程序。

1.  现在，打开一个浏览器标签，导航到`http://localhost:8080/index.html`，你应该看到你的网站，目前只包括一个标题，`个人网站`。

1.  现在，在你喜欢的编辑器中编辑`index.html`文件，使其看起来像这样：

```
<h1>Personal Website</h1> 
<p>This is some text</p> 
```

1.  现在保存它，然后刷新浏览器。哦！那没用。浏览器仍然显示`index.html`文件的先前版本，只包括标题。所以，让我们停止并删除当前容器，然后重建镜像，并重新运行容器，如下所示：

```
$ docker container rm -f my-site
$ docker image build -t my-website:1.0 .
$ docker container run -d \
 --name my-site \
   -p 8080:80 \
 my-website:1.0
```

这次，当你刷新浏览器时，新内容应该显示出来。好吧，它起作用了，但涉及的摩擦太多了。想象一下，每次对网站进行简单更改时都要这样做。这是不可持续的。

1.  现在是使用主机挂载卷的时候了。再次删除当前容器，并使用卷挂载重新运行它，就像这样：

```
$ docker container rm -f my-site
$ docker container run -d \
 --name my-site \
   -v $(pwd):/usr/share/nginx/html \
 -p 8080:80 \
 my-website:1.0
```

1.  现在，向`index.html`文件追加一些内容，并保存。然后，刷新你的浏览器。你应该看到变化。这正是我们想要实现的；我们也称之为*编辑和继续*体验。你可以对网页文件进行任意更改，并立即在浏览器中看到结果，而无需重建镜像和重新启动包含你的网站的容器。

重要的是要注意，更新现在是双向传播的。如果你在主机上进行更改，它们将传播到容器，反之亦然。同样重要的是，当你将当前文件夹挂载到容器目标文件夹`/usr/share/nginx/html`时，已经存在的内容将被主机文件夹的内容替换。

# 在镜像中定义卷

如果我们回顾一下我们在第三章中学到的关于容器的知识，*掌握容器*，那么我们有这样的情况：每个容器的文件系统在启动时由底层镜像的不可变层和特定于该容器的可写容器层组成。容器内运行的进程对文件系统所做的所有更改都将持久保存在该容器层中。一旦容器停止并从系统中删除，相应的容器层将从系统中删除并且不可逆地丢失。

一些应用程序，比如在容器中运行的数据库，需要将它们的数据持久保存超出容器的生命周期。在这种情况下，它们可以使用卷。为了更加明确，让我们看一个具体的例子。MongoDB 是一个流行的开源文档数据库。许多开发人员使用 MongoDB 作为他们应用程序的存储服务。MongoDB 的维护者已经创建了一个镜像，并将其发布到 Docker Hub，可以用来在容器中运行数据库的实例。这个数据库将产生需要长期持久保存的数据，但 MongoDB 的维护者不知道谁使用这个镜像以及它是如何被使用的。因此，他们对于用户启动这个容器的`docker container run`命令没有影响。*他们现在如何定义卷呢？*

幸运的是，在`Dockerfile`中有一种定义卷的方法。这样做的关键字是`VOLUME`，我们可以添加单个文件夹的绝对路径或逗号分隔的路径列表。这些路径代表容器文件系统的文件夹。让我们看一些这样的卷定义示例，如下：

```
VOLUME /app/data 
VOLUME /app/data, /app/profiles, /app/config 
VOLUME ["/app/data", "/app/profiles", "/app/config"] 
```

前面片段中的第一行定义了一个要挂载到`/app/data`的单个卷。第二行定义了三个卷作为逗号分隔的列表。最后一个与第二行定义相同，但这次值被格式化为 JSON 数组。

当容器启动时，Docker 会自动创建一个卷，并将其挂载到`Dockerfile`中定义的每个路径对应的容器目标文件夹。由于每个卷都是由 Docker 自动创建的，它将有一个 SHA-256 作为其 ID。

在容器运行时，在`Dockerfile`中定义为卷的文件夹被排除在联合文件系统之外，因此这些文件夹中的任何更改都不会改变容器层，而是持久保存到相应的卷中。现在，运维工程师有责任确保卷的后备存储得到适当备份。

我们可以使用`docker image inspect`命令来获取关于`Dockerfile`中定义的卷的信息。让我们按照以下步骤来看看 MongoDB 给我们的信息：

1.  首先，我们使用以下命令拉取镜像：

```
$ docker image pull mongo:3.7
```

1.  然后，我们检查这个镜像，并使用`--format`参数来从大量数据中提取必要的部分，如下：

```
 $ docker image inspect \
    --format='{{json .ContainerConfig.Volumes}}' \
    mongo:3.7 | jq . 
```

请注意命令末尾的`| jq .`。我们正在将`docker image inspect`的输出导入`jq`工具，它会很好地格式化输出。如果您尚未在系统上安装`jq`，您可以在 macOS 上使用`brew install jq`，或在 Windows 上使用`choco install jq`来安装。

上述命令将返回以下结果：

```
{
 "/data/configdb": {},
 "/data/db": {}
}
```

显然，MongoDB 的`Dockerfile`在`/data/configdb`和`/data/db`定义了两个卷。

1.  现在，让我们作为后台守护进程运行一个 MongoDB 实例，如下所示：

```
$ docker run --name my-mongo -d mongo:3.7
```

1.  我们现在可以使用`docker container inspect`命令获取有关已创建的卷等信息。

使用此命令只获取卷信息：

```
$ docker inspect --format '{{json .Mounts}}' my-mongo | jq .
```

前面的命令应该输出类似这样的内容（缩短）：

```
[
  {
    "Type": "volume",
    "Name": "b9ea0158b5...",
    "Source": "/var/lib/docker/volumes/b9ea0158b.../_data",
    "Destination": "/data/configdb",
    "Driver": "local",
    ...
  },
  {
    "Type": "volume",
    "Name": "5becf84b1e...",
    "Source": "/var/lib/docker/volumes/5becf84b1.../_data",
    "Destination": "/data/db",
    ...
  }
]
```

请注意，为了便于阅读，`Name`和`Source`字段的值已被修剪。`Source`字段为我们提供了主机目录的路径，MongoDB 在容器内生成的数据将存储在其中。

目前关于卷的内容就是这些。在下一节中，我们将探讨如何配置在容器中运行的应用程序，以及容器镜像构建过程本身。

# 配置容器

往往我们需要为容器内运行的应用程序提供一些配置。配置通常用于允许同一个容器在非常不同的环境中运行，例如开发、测试、暂存或生产环境。

在 Linux 中，通常通过环境变量提供配置值。

我们已经了解到，在容器内运行的应用程序与其主机环境完全隔离。因此，在主机上看到的环境变量与在容器内看到的环境变量是不同的。

让我们首先看一下在我们的主机上定义了什么：

1.  使用此命令：

```
$ export
```

在我的 macOS 上，我看到类似这样的东西（缩短）：

```
...
COLORFGBG '7;0'
COLORTERM truecolor
HOME /Users/gabriel
ITERM_PROFILE Default
ITERM_SESSION_ID w0t1p0:47EFAEFE-BA29-4CC0-B2E7-8C5C2EA619A8
LC_CTYPE UTF-8
LOGNAME gabriel
...
```

1.  接下来，让我们在`alpine`容器内运行一个 shell，并列出我们在那里看到的环境变量，如下所示：

```
$ docker container run --rm -it alpine /bin/sh
/ # export 
export HOME='/root'
export HOSTNAME='91250b722bc3'
export PATH='/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'
export PWD='/'
export SHLVL='1'
export TERM='xterm'
```

我们从`export`命令看到的前面的输出显然与我们直接在主机上看到的完全不同。

1.  按下*Ctrl* + *D*离开`alpine`容器。

接下来，让我们为容器定义环境变量。

# 为容器定义环境变量

现在，好处是我们实际上可以在启动时将一些配置值传递到容器中。我们可以使用`--env`（或简写形式`-e`）参数以`--env <key>=<value>`的形式这样做，其中`<key>`是环境变量的名称，`<value>`表示与该变量关联的值。假设我们希望在容器中运行的应用程序能够访问名为`LOG_DIR`的环境变量，其值为`/var/log/my-log`。我们可以使用以下命令来实现：

```
$ docker container run --rm -it \
 --env LOG_DIR=/var/log/my-log \
 alpine /bin/sh
/ #
```

上述代码在`alpine`容器中启动了一个 shell，并在运行的容器内定义了所请求的环境。为了证明这是真的，我们可以在`alpine`容器内执行这个命令：

```
/ # export | grep LOG_DIR 
export LOG_DIR='/var/log/my-log'
```

输出看起来如预期的那样。我们现在确实在容器内有了所请求的环境变量和正确的值。

当然，当我们运行容器时，我们可以定义多个环境变量。我们只需要重复`--env`（或`-e`）参数。看一下这个示例：

```
$ docker container run --rm -it \
 --env LOG_DIR=/var/log/my-log \    --env MAX_LOG_FILES=5 \
 --env MAX_LOG_SIZE=1G \
 alpine /bin/sh
/ #
```

如果我们现在列出环境变量，我们会看到以下内容：

```
/ # export | grep LOG 
export LOG_DIR='/var/log/my-log'
export MAX_LOG_FILES='5'
export MAX_LOG_SIZE='1G'
```

让我们现在看一下我们有许多环境变量需要配置的情况。

# 使用配置文件

复杂的应用程序可能有许多环境变量需要配置，因此我们运行相应容器的命令可能会变得难以控制。为此，Docker 允许我们将环境变量定义作为文件传递，并且我们在`docker container run`命令中有`--env-file`参数。

让我们试一下，如下所示：

1.  创建一个`fod/05`文件夹并导航到它，就像这样：

```
$ mkdir -p ~/fod/05 && cd ~/fod/05
```

1.  使用您喜欢的编辑器在此文件夹中创建一个名为`development.config`的文件。将以下内容添加到文件中，并保存如下：

```
LOG_DIR=/var/log/my-log
MAX_LOG_FILES=5
MAX_LOG_SIZE=1G
```

注意我们每行定义一个环境变量的格式是`<key>=<value>`，其中，再次，`<key>`是环境变量的名称，`<value>`表示与该变量关联的值。

1.  现在，从`fod/05`文件夹中，让我们运行一个`alpine`容器，将文件作为环境文件传递，并在容器内运行`export`命令，以验证文件中列出的变量确实已经在容器内部创建为环境变量，就像这样：

```
$ docker container run --rm -it \
 --env-file ./development.config \
 alpine sh -c "export"
```

确实，变量已经被定义，正如我们在生成的输出中所看到的：

```
export HOME='/root'
export HOSTNAME='30ad92415f87'
export LOG_DIR='/var/log/my-log'
export MAX_LOG_FILES='5'
export MAX_LOG_SIZE='1G'
export PATH='/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'
export PWD='/'
export SHLVL='1'
export TERM='xterm'
```

接下来，让我们看看如何为给定 Docker 镜像的所有容器实例定义环境变量的默认值。

# 在容器镜像中定义环境变量

有时，我们希望为必须存在于给定容器镜像的所有容器实例中的环境变量定义一些默认值。我们可以在用于创建该镜像的`Dockerfile`中这样做，按照以下步骤：

1.  使用您喜欢的编辑器在`~/fod/05`文件夹中创建一个名为`Dockerfile`的文件。将以下内容添加到文件中，并保存：

```
FROM alpine:latest
ENV LOG_DIR=/var/log/my-log
ENV  MAX_LOG_FILES=5
ENV MAX_LOG_SIZE=1G
```

1.  使用前述`Dockerfile`创建一个名为`my-alpine`的容器镜像，如下所示：

```
$ docker image build -t my-alpine .
```

从该镜像运行一个容器实例，输出容器内定义的环境变量，就像这样：

```
$ docker container run --rm -it \
    my-alpine sh -c "export | grep LOG" 
export LOG_DIR='/var/log/my-log'
export MAX_LOG_FILES='5'
export MAX_LOG_SIZE='1G'
```

这正是我们所期望的。

不过，好消息是，我们并不完全受困于这些变量值。我们可以使用`docker container run`命令中的`--env`参数覆盖其中一个或多个变量。看一下以下命令及其输出：

```
$ docker container run --rm -it \
    --env MAX_LOG_SIZE=2G \
    --env MAX_LOG_FILES=10 \
    my-alpine sh -c "export | grep LOG" 
export LOG_DIR='/var/log/my-log'
export MAX_LOG_FILES='10'
export MAX_LOG_SIZE='2G'
```

我们还可以使用环境文件和`docker container run`命令中的`--env-file`参数来覆盖默认值。请自行尝试。

# 构建时的环境变量

有时，我们希望在构建容器镜像时定义一些环境变量，这些变量在构建时是有效的。想象一下，你想定义一个`BASE_IMAGE_VERSION`环境变量，然后在你的`Dockerfile`中将其用作参数。想象一下以下的`Dockerfile`：

```
ARG BASE_IMAGE_VERSION=12.7-stretch
FROM node:${BASE_IMAGE_VERSION}
WORKDIR /app
COPY packages.json .
RUN npm install
COPY . .
CMD npm start
```

我们使用`ARG`关键字来定义一个默认值，每次从前述`Dockerfile`构建镜像时都会使用这个默认值。在这种情况下，这意味着我们的镜像使用`node:12.7-stretch`基础镜像。

现在，如果我们想为—比如—测试目的创建一个特殊的镜像，我们可以使用`--build-arg`参数在构建镜像时覆盖这个变量，如下所示：

```
$ docker image build \
 --build-arg BASE_IMAGE_VERSION=12.7-alpine \
 -t my-node-app-test .
```

在这种情况下，生成的`my-node-test:latest`镜像将从`node:12.7-alpine`基础镜像构建，而不是从`node:12.7-stretch`默认镜像构建。

总之，通过`--env`或`--env-file`定义的环境变量在容器运行时有效。在`Dockerfile`中使用`ARG`或在`docker container build`命令中使用`--build-arg`定义的变量在容器镜像构建时有效。前者用于配置容器内运行的应用程序，而后者用于参数化容器镜像构建过程。

# 总结

在本章中，我们介绍了 Docker 卷，可以用来持久保存容器产生的状态并使其持久。我们还可以使用卷来为容器提供来自各种来源的数据。我们已经学会了如何创建、挂载和使用卷。我们已经学会了各种定义卷的技术，例如按名称、通过挂载主机目录或在容器镜像中定义卷。

在这一章中，我们还讨论了如何配置环境变量，这些变量可以被容器内运行的应用程序使用。我们已经展示了如何在`docker container run`命令中定义这些变量，可以明确地一个一个地定义，也可以作为配置文件中的集合。我们还展示了如何通过使用构建参数来参数化容器镜像的构建过程。

在下一章中，我们将介绍常用的技术，允许开发人员在容器中运行代码时进行演变、修改、调试和测试。

# 问题

请尝试回答以下问题，以评估您的学习进度：

1.  如何创建一个名为`my-products`的命名数据卷，使用默认驱动程序？

1.  如何使用`alpine`镜像运行一个容器，并将`my-products`卷以只读模式挂载到`/data`容器文件夹中？

1.  如何找到与`my-products`卷关联的文件夹并导航到它？另外，您将如何创建一个带有一些内容的文件`sample.txt`？

1.  如何在另一个`alpine`容器中运行，并将`my-products`卷挂载到`/app-data`文件夹中，以读/写模式？在此容器内，导航到`/app-data`文件夹并创建一个带有一些内容的`hello.txt`文件。

1.  如何将主机卷（例如`~/my-project`）挂载到容器中？

1.  如何从系统中删除所有未使用的卷？

1.  在容器中运行的应用程序看到的环境变量列表与应用程序直接在主机上运行时看到的相同。

A. 真

B. 假

1.  您的应用程序需要在容器中运行，并为其配置提供大量环境变量。运行一个包含您的应用程序并向其提供所有这些信息的容器的最简单方法是什么？

# 进一步阅读

以下文章提供更深入的信息：

+   使用卷，在[`dockr.ly/2EUjTml`](http://dockr.ly/2EUjTml)

+   在 Docker 中管理数据，在[`dockr.ly/2EhBpzD`](http://dockr.ly/2EhBpzD)

+   **Play with Docker** (**PWD**)上的 Docker 卷，在[`bit.ly/2sjIfDj`](http://bit.ly/2sjIfDj)

+   `nsenter`—Linux man 页面，在[`bit.ly/2MEPG0n`](https://bit.ly/2MEPG0n)

+   设置环境变量，在[`dockr.ly/2HxMCjS`](https://dockr.ly/2HxMCjS)

+   了解`ARG`和`FROM`如何交互，在[`dockr.ly/2OrhZgx`](https://dockr.ly/2OrhZgx)

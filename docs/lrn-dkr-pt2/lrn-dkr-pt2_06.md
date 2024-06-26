# 第六章：在容器中运行服务

我们一步步地走到了这一步，为快速发展的 Docker 技术奠定了坚实而令人振奋的基础。我们谈论了高度可用和可重复使用的 Docker 镜像的重要构建模块。此外，您可以阅读如何通过精心设计的存储框架存储和共享 Docker 镜像的易于使用的技术和提示。通常情况下，镜像必须不断经过一系列验证、验证和不断完善，以使它们更加正确和相关，以满足渴望发展的社区的需求。在本章中，我们将通过描述创建一个小型 Web 服务器的步骤，将其运行在容器内，并从外部世界连接到 Web 服务器，将我们的学习提升到一个新的水平。

在本章中，我们将涵盖以下主题：

+   容器网络

+   **容器即服务**（**CaaS**）-构建、运行、暴露和连接到容器服务

+   发布和检索容器端口

+   将容器绑定到特定 IP 地址

+   自动生成 Docker 主机端口

+   使用`EXPOSE`和`-P`选项进行端口绑定

# 容器网络简要概述

与任何计算节点一样，Docker 容器需要进行网络连接，以便其他容器和客户端可以找到并访问它们。在网络中，通常通过 IP 地址来识别任何节点。此外，IP 地址是任何客户端到达任何服务器节点提供的服务的唯一机制。Docker 内部使用 Linux 功能来为容器提供网络连接。在本节中，我们将学习有关容器 IP 地址分配和检索容器 IP 地址的过程。

当容器启动时，Docker 引擎会无需用户干预地选择并分配 IP 地址给容器。您可能会对 Docker 如何为容器选择 IP 地址感到困惑，这个谜团分为两部分来解答，如下所示：

1.  在安装过程中，Docker 在 Docker 主机上创建一个名为`docker0`的虚拟接口。它还选择一个私有 IP 地址范围，并从所选范围中为`docker0`虚拟接口分配一个地址。所选的 IP 地址始终位于 Docker 主机 IP 地址范围之外，以避免 IP 地址冲突。

1.  稍后，当我们启动一个容器时，Docker 引擎会从为`docker0`虚拟接口选择的 IP 地址范围中选择一个未使用的 IP 地址。然后，引擎将这个 IP 地址分配给新启动的容器。

默认情况下，Docker 会选择 IP 地址`172.17.42.1/16`，或者在`172.17.0.0`到`172.17.255.255`范围内的 IP 地址之一。如果与`172.17.x.x`地址直接冲突，Docker 将选择不同的私有 IP 地址范围。也许，老式的`ifconfig`（显示网络接口详细信息的命令）在这里很有用，可以用来找出分配给虚拟接口的 IP 地址。让我们用`docker0`作为参数运行`ifconfig`，如下所示：

```
$ ifconfig docker0

```

输出的第二行将显示分配的 IP 地址及其子网掩码：

```
inet addr:172.17.42.1  Bcast:0.0.0.0  Mask:255.255.0.0

```

显然，从前面的文本中，`172.17.42.1`是分配给`docker0`虚拟接口的 IP 地址。IP 地址`172.17.42.1`是从`172.17.0.0`到`172.17.255.255`的私有 IP 地址范围中的一个地址。

现在迫切需要我们学习如何找到分配给容器的 IP 地址。容器应该使用`-i`选项以交互模式启动。当然，我们可以通过在容器内运行`ifconfig`命令来轻松找到 IP 地址，如下所示：

```
$ sudo docker run -i -t ubuntu:14.04 /bin/bash
root@4b0b567b6019:/# ifconfig

```

`ifconfig`命令将显示 Docker 容器中所有接口的详细信息，如下所示：

```
eth0      Link encap:Ethernet  HWaddr e6:38:dd:23:aa:3f
 inet addr:172.17.0.12  Bcast:0.0.0.0  Mask:255.255.0.0
 inet6 addr: fe80::e438:ddff:fe23:aa3f/64 Scope:Link
 UP BROADCAST RUNNING  MTU:1500  Metric:1
 RX packets:6 errors:0 dropped:2 overruns:0 frame:0
 TX packets:7 errors:0 dropped:0 overruns:0 carrier:0
 collisions:0 txqueuelen:1000
 RX bytes:488 (488.0 B)  TX bytes:578 (578.0 B)

lo        Link encap:Local Loopback
 inet addr:127.0.0.1  Mask:255.0.0.0
 inet6 addr: ::1/128 Scope:Host
 UP LOOPBACK RUNNING  MTU:65536  Metric:1
 RX packets:0 errors:0 dropped:0 overruns:0 frame:0
 TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
 collisions:0 txqueuelen:0
 RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

```

显然，`ifconfig`命令的前面输出显示 Docker 引擎为容器虚拟化了两个网络接口，如下所示：

+   第一个是`eth0`（以太网）接口，Docker 引擎分配了 IP 地址`172.17.0.12`。显然，这个地址也在`docker0`虚拟接口的相同 IP 地址范围内。此外，分配给`eth0`接口的地址用于容器内部通信和主机到容器的通信。

+   第二个接口是`lo`（环回）接口，Docker 引擎分配了环回地址`127.0.0.1`。环回接口用于容器内部的本地通信。

简单吧？然而，当使用`docker run`子命令中的`-d`选项以分离模式启动容器时，检索 IP 地址变得复杂起来。分离模式中的这种复杂性的主要原因是没有 shell 提示符来运行`ifconfig`命令。幸运的是，Docker 提供了一个`docker inspect`子命令，它像瑞士军刀一样方便，并允许我们深入了解 Docker 容器或镜像的低级细节。`docker inspect`子命令以 JSON 数组格式生成请求的详细信息。

以下是我们之前启动的交互式容器上`docker inspect`子命令的示例运行。`4b0b567b6019`容器 ID 取自容器的提示符：

```
$ sudo docker inspect 4b0b567b6019

```

该命令生成有关容器的大量信息。在这里，我们展示了从`docker inspect`子命令的输出中提取的容器网络配置的一些摘录：

```
"NetworkSettings": {
 "Bridge": "docker0",
 "Gateway": "172.17.42.1",
 "IPAddress": "172.17.0.12",
 "IPPrefixLen": 16,
 "PortMapping": null,
 "Ports": {}
 },

```

在这里，网络配置列出了以下详细信息：

+   **Bridge**：这是容器绑定的桥接口

+   **Gateway**：这是容器的网关地址，也是桥接口的地址

+   **IPAddress**：这是分配给容器的 IP 地址

+   **IPPrefixLen**：这是 IP 前缀长度，表示子网掩码的另一种方式

+   **PortMapping**：这是端口映射字段，现在已经被弃用，其值始终为 null

+   **Ports**：这是端口字段，将列举所有端口绑定，这将在本章后面介绍

毫无疑问，`docker inspect`子命令对于查找容器或镜像的细节非常方便。然而，浏览令人生畏的细节并找到我们渴望寻找的正确信息是一项繁琐的工作。也许，您可以使用`grep`命令将其缩小到正确的信息。或者更好的是，使用`docker inspect`子命令，它可以帮助您使用`docker inspect`子命令的`--format`选项从 JSON 数组中选择正确的字段。

值得注意的是，在以下示例中，我们使用`docker inspect`子命令的`--format`选项仅检索容器的 IP 地址。IP 地址可以通过 JSON 数组的`.NetworkSettings.IPAddress`字段访问：

```
$ sudo docker inspect \
 --format='{{.NetworkSettings.IPAddress}}' 4b0b567b6019
172.17.0.12

```

# 将容器视为服务

我们为 Docker 技术的基础打下了良好的基础。在本节中，我们将专注于使用 HTTP 服务创建镜像，使用创建的镜像在容器内启动 HTTP 服务，然后演示连接到容器内运行的 HTTP 服务。

## 构建 HTTP 服务器镜像

在本节中，我们将创建一个 Docker 镜像，以在`Ubuntu 14.04`基础镜像上安装`Apache2`，并配置`Apache HTTP Server`以作为可执行文件运行，使用`ENTRYPOINT`指令。

在第三章*构建镜像*中，我们演示了使用 Dockerfile 在`Ubuntu 14.04`基础镜像上创建`Apache2`镜像的概念。在这个例子中，我们将通过设置`Apache`日志路径和使用`ENTRYPOINT`指令将`Apache2`设置为默认执行应用程序来扩展这个 Dockerfile。以下是`Dockerfile`内容的详细解释。

我们将使用`FROM`指令以`ubuntu:14.04`作为基础镜像构建镜像，如`Dockerfile`片段所示：

```
###########################################
# Dockerfile to build an apache2 image
###########################################
# Base image is Ubuntu
FROM ubuntu:14.04
```

使用 MAINTAINER 指令设置作者详细信息

```
# Author: Dr. Peter
MAINTAINER Dr. Peter <peterindia@gmail.com>
```

使用一个`RUN`指令，我们将同步`apt`仓库源列表，安装`apache2`包，然后清理检索到的文件：

```
# Install apache2 package
RUN apt-get update && \
     apt-get install -y apache2 && \
     apt-get clean
```

使用`ENV`指令设置 Apache 日志目录路径：

```
# Set the log directory PATH
ENV APACHE_LOG_DIR /var/log/apache2
```

现在，最后一条指令是使用`ENTRYPOINT`指令启动`apache2`服务器：

```
# Launch apache2 server in the foreground
ENTRYPOINT ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

在上一行中，您可能会惊讶地看到`FOREGROUND`参数。这是传统和容器范式之间的关键区别之一。在传统范式中，服务器应用通常在后台启动，作为服务或守护程序，因为主机系统是通用系统。然而，在容器范式中，必须在前台启动应用程序，因为镜像是为唯一目的而创建的。

在`Dockerfile`中规定了构建镜像的指令后，现在让我们通过使用`docker build`子命令来构建镜像，将镜像命名为`apache2`，如下所示：

```
$ sudo docker build -t apache2 .

```

现在让我们使用`docker images`子命令快速验证镜像：

```
$ sudo docker images

```

正如我们在之前的章节中所看到的，`docker images`命令显示了 Docker 主机中所有镜像的详细信息。然而，为了准确说明使用`docker build`子命令创建的镜像，我们从完整的镜像列表中突出显示了`apache2:latest`（目标镜像）和`ubuntu:14.04`（基础镜像）的详细信息，如下面的输出片段所示：

```
apache2             latest              d5526cd1a645        About a minute ago   232.6 MB
ubuntu              14.04               5506de2b643b        5 days ago           197.8 MB

```

构建了 HTTP 服务器镜像后，现在让我们继续下一节，学习如何运行 HTTP 服务。

## 作为服务运行 HTTP 服务器镜像

在这一节中，我们将使用在上一节中制作的 Apache HTTP 服务器镜像来启动一个容器。在这里，我们使用`docker run`子命令的`-d`选项以分离模式（类似于 UNIX 守护进程）启动容器：

```
$ sudo docker run -d apache2
9d4d3566e55c0b8829086e9be2040751017989a47b5411c9c4f170ab865afcef

```

启动了容器后，让我们运行`docker logs`子命令，看看我们的 Docker 容器是否在其`STDIN`（标准输入）或`STDERR`（标准错误）上生成任何输出：

```
$ sudo docker logs \ 9d4d3566e55c0b8829086e9be2040751017989a47b5411c9c4f170ab865afcef

```

由于我们还没有完全配置 Apache HTTP 服务器，您将会发现以下警告，作为`docker logs`子命令的输出：

```
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 172.17.0.13\. Set the 'ServerName' directive globally to suppress this message

```

从前面的警告消息中，很明显可以看出分配给这个容器的 IP 地址是`172.17.0.13`。

## 连接到 HTTP 服务

在前面的部分中，从警告消息中，我们发现容器的 IP 地址是`172.17.0.13`。在一个完全配置好的 HTTP 服务器容器上，是没有这样的警告的，所以让我们仍然运行`docker inspect`子命令来使用容器 ID 检索 IP 地址：

```
$ sudo docker inspect \
--format='{{.NetworkSettings.IPAddress}}' \ 9d4d3566e55c0b8829086e9be2040751017989a47b5411c9c4f170ab865afcef
172.17.0.13

```

在 Docker 主机的 shell 提示符下，找到容器的 IP 地址为`172.17.0.13`，让我们快速在这个 IP 地址上运行一个 web 请求，使用`wget`命令。在这里，我们选择使用`-qO-`参数来以安静模式运行`wget`命令，并在屏幕上显示检索到的 HTML 文件：

```
$ wget -qO - 172.17.0.13

```

在这里，我们展示了检索到的 HTML 文件的前五行：

```
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html >
  <!--
    Modified from the Debian original for Ubuntu
    Last updated: 2014-03-19
```

很棒，不是吗？我们在一个容器中运行了我们的第一个服务，并且我们能够从 Docker 主机访问我们的服务。

此外，在普通的 Docker 安装中，一个容器提供的服务可以被 Docker 主机内的任何其他容器访问。您可以继续，在交互模式下启动一个新的 Ubuntu 容器，使用`apt-get`安装`wget`软件包，并运行与我们在 Docker 主机中所做的相同的`wget -qO - 172.17.0.13`命令。当然，您会看到相同的输出。

# 暴露容器服务

到目前为止，我们已经成功启动了一个 HTTP 服务，并从 Docker 主机以及同一 Docker 主机内的另一个容器访问了该服务。此外，正如在第二章的*从容器构建镜像*部分所演示的，容器能够通过在互联网上连接到公共可用的 apt 仓库来成功安装`wget`软件包。然而，默认情况下，外部世界无法访问容器提供的服务。起初，这可能看起来像是 Docker 技术的一个限制。然而，事实是，容器是根据设计与外部世界隔离的。

Docker 通过 IP 地址分配标准实现容器的网络隔离，具体列举如下：

1.  为容器分配一个私有 IP 地址，该地址无法从外部网络访问。

1.  为容器分配一个在主机 IP 网络之外的 IP 地址。

因此，Docker 容器甚至无法从与 Docker 主机相同的 IP 网络连接的系统访问。这种分配方案还可以防止可能会出现的 IP 地址冲突。

现在，您可能想知道如何使服务在容器内运行，并且可以被外部访问，换句话说，暴露容器服务。嗯，Docker 通过在底层利用 Linux `iptables`功能来弥合这种连接差距。

在前端，Docker 为用户提供了两种不同的构建模块来弥合这种连接差距。其中一个构建模块是使用`docker run`子命令的`-p`（将容器的端口发布到主机接口）选项来绑定容器端口。另一种选择是使用`EXPOSE` Dockerfile 指令和`docker run`子命令的`-P`（将所有公开的端口发布到主机接口）选项的组合。

## 发布容器端口-使用-p 选项

Docker 使您能够通过将容器的端口绑定到主机接口来发布容器内提供的服务。`docker run`子命令的`-p`选项使您能够将容器端口绑定到 Docker 主机的用户指定或自动生成的端口。因此，任何发送到 Docker 主机的 IP 地址和端口的通信都将转发到容器的端口。实际上，`-p`选项支持以下四种格式的参数：

+   `<hostPort>:<containerPort>`

+   `<containerPort>`

+   `<ip>:<hostPort>:<containerPort>`

+   `<ip>::<containerPort>`

在这里，`<ip>`是 Docker 主机的 IP 地址，`<hostPort>`是 Docker 主机的端口号，`<containerPort>`是容器的端口号。在本节中，我们向您介绍了`-p <hostPort>:<containerPort>`格式，并在接下来的部分介绍其他格式。

为了更好地理解端口绑定过程，让我们重用之前创建的`apache2` HTTP 服务器镜像，并使用`docker run`子命令的`-p`选项启动一个容器。端口`80`是 HTTP 服务的发布端口，作为默认行为，我们的`apache2` HTTP 服务器也可以在端口`80`上访问。在这里，为了演示这种能力，我们将使用`docker run`子命令的`-p <hostPort>:<containerPort>`选项，将容器的端口`80`绑定到 Docker 主机的端口`80`，如下命令所示：

```
$ sudo docker run -d -p 80:80 apache2
baddba8afa98725ec85ad953557cd0614b4d0254f45436f9cb440f3f9eeae134

```

现在我们已经成功启动了容器，我们可以使用任何外部系统的 Web 浏览器连接到我们的 HTTP 服务器（只要它具有网络连接），以访问我们的 Docker 主机。到目前为止，我们还没有向我们的`apache2` HTTP 服务器镜像添加任何网页。

因此，当我们从 Web 浏览器连接时，我们将得到以下屏幕，这只是随`Ubuntu Apache2`软件包一起提供的默认页面：

![发布容器端口- -p 选项](img/7937OT_06_01.jpg)

## 容器的网络地址转换

在上一节中，我们看到`-p 80:80`选项是如何起作用的，不是吗？实际上，在幕后，Docker 引擎通过自动配置 Linux `iptables`配置文件中的**网络地址转换**（**NAT**）规则来实现这种无缝连接。

为了说明在 Linux `iptables`中自动配置 NAT 规则，让我们查询 Docker 主机的`iptables`以获取其 NAT 条目，如下所示：

```
$ sudo iptables -t nat -L -n

```

接下来的文本是从 Docker 引擎自动添加的`iptables` NAT 条目中摘录的：

```
Chain DOCKER (2 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.17.0.14:80

```

从上面的摘录中，很明显 Docker 引擎有效地添加了一个`DNAT`规则。以下是`DNAT`规则的详细信息：

+   `tcp`关键字表示这个`DNAT`规则仅适用于 TCP 传输协议。

+   第一个`0.0.0.0/0`地址是源地址的元 IP 地址。这个地址表示连接可以来自任何 IP 地址。

+   第二个`0.0.0.0/0`地址是 Docker 主机上目标地址的元 IP 地址。这个地址表示连接可以与 Docker 主机中的任何有效 IP 地址建立。

+   最后，`dpt:80 to:172.17.0.14:80`是用于将 Docker 主机端口`80`上的任何 TCP 活动转发到 IP 地址`172.17.0.17`，即我们容器的 IP 地址和端口`80`的转发指令。

因此，Docker 主机接收到的任何 TCP 数据包都将转发到容器的端口`80`。

## 检索容器端口

Docker 引擎提供至少三种不同的选项来检索容器的端口绑定详细信息。在这里，让我们首先探索选项，然后继续分析检索到的信息。选项如下：

+   `docker ps`子命令始终显示容器的端口绑定详细信息，如下所示：

```
$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                NAMES
baddba8afa98        apache2:latest      "/usr/sbin/apache2ct   26 seconds ago      Up 25 seconds       0.0.0.0:80->80/tcp   furious_carson

```

+   `docker inspect`子命令是另一种选择；但是，你必须浏览相当多的细节。运行以下命令：

```
$ sudo docker inspect baddba8afa98

```

`docker inspect`子命令以三个 JSON 对象显示与端口绑定相关的信息，如下所示：

+   `ExposedPorts`对象枚举了通过`Dockerfile`中的`EXPOSE`指令暴露的所有端口，以及使用`docker run`子命令中的`-p`选项映射的容器端口。由于我们没有在`Dockerfile`中添加`EXPOSE`指令，我们只有使用`-p80:80`作为`docker run`子命令的参数映射的容器端口：

```
"ExposedPorts": {
 "80/tcp": {}
 },

```

+   `PortBindings`对象是`HostConfig`对象的一部分，该对象列出了通过`docker run`子命令中的`-p`选项进行的所有端口绑定。该对象永远不会列出通过`Dockerfile`中的`EXPOSE`指令暴露的端口：

```
"PortBindings": {
 "80/tcp": [
 {
 "HostIp": "",
 "HostPort": "80"
 }
 ]
 },

```

+   `NetworkSettings`对象的`Ports`对象具有与先前的`PortBindings`对象相同级别的细节。但是，该对象包含通过`Dockerfile`中的`EXPOSE`指令暴露的所有端口，以及使用`docker run`子命令的`-p`选项映射的容器端口：

```
"NetworkSettings": {
 "Bridge": "docker0",
 "Gateway": "172.17.42.1",
 "IPAddress": "172.17.0.14",
 "IPPrefixLen": 16,
 "PortMapping": null,
 "Ports": {
 "80/tcp": [
 {
 "HostIp": "0.0.0.0",
 "HostPort": "80"
 }
 ]
 }
 },

```

当然，可以使用`docker inspect`子命令的`--format`选项来过滤特定的端口字段。

+   `docker port`子命令允许您通过指定容器的端口号来检索 Docker 主机上的端口绑定：

```
$ sudo docker port baddba8afa98 80
0.0.0.0:80

```

显然，在所有先前的输出摘录中，突出显示的信息是 IP 地址`0.0.0.0`和端口号`80`。IP 地址`0.0.0.0`是一个元地址，代表了 Docker 主机上配置的所有 IP 地址。实际上，容器端口`80`绑定到了 Docker 主机上所有有效的 IP 地址。因此，HTTP 服务可以通过 Docker 主机上配置的任何有效 IP 地址访问。

## 将容器绑定到特定的 IP 地址

到目前为止，使用我们学到的方法，容器总是绑定到 Docker 主机上配置的所有 IP 地址。然而，您可能希望在不同的 IP 地址上提供不同的服务。换句话说，特定的 IP 地址和端口将被配置为提供特定的服务。我们可以在 Docker 中使用`docker run`子命令的`-p <ip>:<hostPort>:<containerPort>`选项来实现这一点，如下例所示：

```
$ sudo docker run -d -p 198.51.100.73:80:80 apache2
92f107537bebd48e8917ea4f4788bf3f57064c8c996fc23ea0fd8ea49b4f3335

```

在这里，IP 地址必须是 Docker 主机上的有效 IP 地址。如果指定的 IP 地址不是 Docker 主机上的有效 IP 地址，则容器启动将失败，并显示错误消息，如下所示：

```
2014/11/09 10:22:10 Error response from daemon: Cannot start container 99db8d30b284c0a0826d68044c42c370875d2c3cad0b87001b858ba78e9de53b: Error starting userland proxy: listen tcp 198.51.100.73:80: bind: cannot assign requested address

```

现在，让我们快速回顾一下前面示例的端口映射以及 NAT 条目。

以下文本是`docker ps`子命令的输出摘录，显示了此容器的详细信息：

```
92f107537beb        apache2:latest      "/usr/sbin/apache2ct   About a minute ago   Up About a minute   198.51.100.73:80->80/tcp   boring_ptolemy

```

以下文本是`iptables -n nat -L -n`命令的输出摘录，显示了为此容器创建的`DNAT`条目：

```
DNAT    tcp -- 0.0.0.0/0      198.51.100.73     tcp dpt:80 to:172.17.0.15:80

```

在审查`docker run`子命令的输出和`iptables`的`DNAT`条目之后，您将意识到 Docker 引擎如何优雅地配置了容器在 Docker 主机的 IP 地址`198.51.100.73`和端口`80`上提供的服务。

## 自动生成 Docker 主机端口

Docker 容器天生轻量级，由于其轻量级的特性，您可以在单个 Docker 主机上运行多个相同或不同服务的容器。特别是根据需求在多个容器之间自动扩展相同服务的需求是当今 IT 基础设施的需求。在本节中，您将了解在启动多个具有相同服务的容器时所面临的挑战，以及 Docker 解决这一挑战的方式。

在本章的前面，我们使用`apache2 http server`启动了一个容器，并将其绑定到 Docker 主机的端口`80`。现在，如果我们尝试再启动一个绑定到相同端口`80`的容器，容器将无法启动，并显示错误消息，如下例所示：

```
$ sudo docker run -d -p 80:80 apache2
6f01f485ab3ce81d45dc6369316659aed17eb341e9ad0229f66060a8ba4a2d0e
2014/11/03 23:28:07 Error response from daemon: Cannot start container 6f01f485ab3ce81d45dc6369316659aed17eb341e9ad0229f66060a8ba4a2d0e: Bind for 0.0.0.0:80 failed: port is already allocated

```

显然，在上面的例子中，容器无法启动，因为先前的容器已经映射到`0.0.0.0`（Docker 主机的所有 IP 地址）和端口`80`。在 TCP/IP 通信模型中，IP 地址、端口和传输协议（TCP、UDP 等）的组合必须是唯一的。

我们可以通过手动选择 Docker 主机端口号（例如，`-p 81:80`或`-p 8081:80`）来解决这个问题。虽然这是一个很好的解决方案，但在自动扩展的场景下表现不佳。相反，如果我们把控制权交给 Docker，它会在 Docker 主机上自动生成端口号。通过使用`docker run`子命令的`-p <containerPort>`选项来实现这种端口号生成，如下例所示：

```
$ sudo docker run -d -p 80 apache2
ea3e0d1b18cff40ffcddd2bf077647dc94bceffad967b86c1a343bd33187d7a8

```

成功启动了具有自动生成端口的新容器后，让我们回顾一下上面例子的端口映射以及 NAT 条目：

+   以下文本是`docker ps`子命令的输出摘录，显示了该容器的详细信息：

```
ea3e0d1b18cf        apache2:latest      "/usr/sbin/apache2ct   5 minutes ago       Up 5 minutes        0.0.0.0:49158->80/tcp   nostalgic_morse

```

+   以下文本是`iptables -n nat -L -n`命令的输出摘录，显示了为该容器创建的`DNAT`条目：

```
DNAT    tcp -- 0.0.0.0/0      0.0.0.0/0      tcp dpt:49158 to:172.17.0.18:80

```

在审查了`docker run`子命令的输出和`iptables`的`DNAT`条目之后，引人注目的是端口号`49158`。端口号`49158`是由 Docker 引擎在 Docker 主机上巧妙地自动生成的，借助底层操作系统的帮助。此外，元 IP 地址`0.0.0.0`意味着容器提供的服务可以通过 Docker 主机上配置的任何有效 IP 地址从外部访问。

您可能有一个使用案例，您希望自动生成端口号。但是，如果您仍希望将服务限制在 Docker 主机的特定 IP 地址上，您可以使用`docker run`子命令的`-p <IP>::<containerPort>`选项，如下例所示：

```
$ sudo docker run -d -p 198.51.100.73::80 apache2
6b5de258b3b82da0290f29946436d7ae307c8b72f22239956e453356532ec2a7

```

在前述的两种情况中，Docker 引擎在 Docker 主机上自动生成了端口号并将其暴露给外部世界。网络通信的一般规范是通过预定义的端口号公开任何服务，以便任何人都可以知道 IP 地址，并且端口号可以轻松访问提供的服务。然而，在这里，端口号是自动生成的，因此外部世界无法直接访问提供的服务。因此，容器创建的这种方法的主要目的是实现自动扩展，并且以这种方式创建的容器将与预定义端口上的代理或负载平衡服务进行接口。

## 使用 EXPOSE 和-P 选项进行端口绑定

到目前为止，我们已经讨论了将容器内运行的服务发布到外部世界的四种不同方法。在这四种方法中，端口绑定决策是在容器启动时进行的，并且镜像对于提供服务的端口没有任何信息。到目前为止，这已经运作良好，因为镜像是由我们构建的，我们对提供服务的端口非常清楚。然而，在第三方镜像的情况下，容器内的端口使用必须明确发布。此外，如果我们为第三方使用或甚至为自己使用构建镜像，明确声明容器提供服务的端口是一个良好的做法。也许，镜像构建者可以随镜像一起提供一个自述文件。然而，将端口详细信息嵌入到镜像本身中会更好，这样您可以轻松地从镜像中手动或通过自动化脚本找到端口详细信息。

Docker 技术允许我们使用`Dockerfile`中的`EXPOSE`指令嵌入端口信息，我们在第三章*构建镜像*中介绍过。在这里，让我们编辑之前在本章中使用的构建`apache2` HTTP 服务器镜像的`Dockerfile`，并添加一个`EXPOSE`指令，如下所示。HTTP 服务的默认端口是端口`80`，因此端口`80`被暴露：

```
###########################################
# Dockerfile to build an apache2 image
###########################################
# Base image is Ubuntu
FROM ubuntu:14.04
# Author: Dr. Peter
MAINTAINER Dr. Peter <peterindia@gmail.com>
# Install apache2 package
RUN apt-get update && \
     apt-get install -y apache2 && \
     apt-get clean
# Set the log directory PATH
ENV APACHE_LOG_DIR /var/log/apache2
# Expose port 80
EXPOSE 80
# Launch apache2 server in the foreground
ENTRYPOINT ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

现在我们已经在我们的`Dockerfile`中添加了`EXPOSE`指令，让我们继续使用`docker build`命令构建镜像的下一步。在这里，让我们重用镜像名称`apache2`，如下所示：

```
$ sudo docker build -t apache2 .

```

成功构建了镜像后，让我们检查镜像以验证`EXPOSE`指令对镜像的影响。正如我们之前学到的，我们可以使用`docker inspect`子命令，如下所示：

```
$ sudo docker inspect apache2

```

在仔细审查前面命令生成的输出后，您会意识到 Docker 将暴露的端口信息存储在`Config`对象的`ExposedPorts`字段中。以下是摘录，显示了暴露的端口信息是如何显示的：

```
"ExposedPorts": {
 "80/tcp": {}
 },

```

或者，您可以将格式选项应用于`docker inspect`子命令，以便将输出缩小到非常特定的信息。在这种情况下，`Config`对象的`ExposedPorts`字段在以下示例中显示：

```
$ sudo docker inspect --format='{{.Config.ExposedPorts}}' \
 apache2
map[80/tcp:map[]]

```

继续讨论`EXPOSE`指令，我们现在可以使用我们刚刚创建的`apache2`镜像启动容器。然而，`EXPOSE`指令本身不能在 Docker 主机上创建端口绑定。为了为使用`EXPOSE`指令声明的端口创建端口绑定，Docker 引擎在`docker run`子命令中提供了`-P`选项。

在下面的示例中，从之前重建的`apache2`镜像启动了一个容器。在这里，使用`-d`选项以分离模式启动容器，并使用`-P`选项为 Docker 主机上声明的所有端口创建端口绑定，使用`Dockerfile`中的`EXPOSE`指令：

```
$ sudo docker run -d -P apache2
fdb1c8d68226c384ab4f84882714fec206a73fd8c12ab57981fbd874e3fa9074

```

现在我们已经使用`EXPOSE`指令创建了新容器的镜像，就像之前的容器一样，让我们回顾一下端口映射以及前面示例的 NAT 条目：

+   以下文本摘自`docker ps`子命令的输出，显示了此容器的详细信息：

```
ea3e0d1b18cf        apache2:latest      "/usr/sbin/apache2ct   5 minutes ago       Up 5 minutes        0.0.0.0:49159->80/tcp   nostalgic_morse

```

+   以下文本摘自`iptables -t nat -L -n`命令的输出，显示了为该容器创建的`DNAT`条目：

```
DNAT    tcp -- 0.0.0.0/0      0.0.0.0/0      tcp dpt:49159 to:172.17.0.19:80

```

`docker run`子命令的`-P`选项不接受任何额外的参数，比如 IP 地址或端口号；因此，无法对端口绑定进行精细调整，如`docker run`子命令的`-p`选项。如果对端口绑定的精细调整对您至关重要，您可以随时使用`docker run`子命令的`-p`选项。

# 总结

容器在实质上并不以孤立或独立的方式提供任何东西。它们需要系统地构建，并配备网络接口和端口号。这导致容器在外部世界中的标准化展示，使其他主机或容器能够在任何网络上找到、绑定和利用它们独特的能力。因此，网络可访问性对于容器被注意并以无数种方式被利用至关重要。本章专门展示了容器如何被设计和部署为服务，以及容器网络的方面如何在日益展开的日子里精确而丰富地赋予容器服务的独特世界力量。在接下来的章节中，我们将详细讨论 Docker 容器在软件密集型 IT 环境中的各种能力。

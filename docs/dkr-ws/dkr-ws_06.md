# 6. Docker 网络简介

概述

本章的目标是为您提供容器网络如何工作的简明概述，以及它与 Docker 主机级别的网络有何不同，以及容器如何利用 Docker 网络提供对其他容器化服务的直接网络连接。通过本章的学习，您将了解如何使用`bridge`、`overlay`、`macvlan`和`host`等网络配置部署容器。您将了解不同网络驱动程序的优势，并在何种情况下应选择特定的网络驱动程序。最后，我们将研究在 Docker swarm 集群中部署的主机之间的容器化网络。

# 介绍

在整个研讨会中，我们已经研究了与 Docker 相关的容器化和微服务架构的许多方面。我们已经了解了如何将应用程序封装为执行离散功能的微服务，从而创建了一种非常灵活的架构，可以实现快速部署和强大的水平扩展。也许与容器化相关的更有趣和复杂的话题之一是网络。毕竟，为了开发灵活和敏捷的微服务架构，需要考虑适当的网络，以确保容器实例之间可靠的连接。

在涉及**容器网络**时，始终要记住容器主机上的网络（底层网络）与同一主机上或不同集群内的容器之间的网络（`覆盖`网络）之间的区别。Docker 支持许多不同类型的网络配置，可以根据基础设施和部署策略的需求进行定制。

例如，一个容器可能具有一个 IP 地址，该 IP 地址是该容器实例独有的，在容器主机之间的虚拟子网上存在。这种类型的网络是 Docker swarm 集群配置的典型特征，其中网络流量被加密并通过主机机器的网络接口传输，然后在不同的主机上解密，然后传递给接收微服务。这种类型的网络配置通常涉及 Docker 维护容器和服务名称到容器 IP 地址的映射。这提供了强大的服务发现机制，即使容器在不同的集群主机上终止和重新启动，也可以进行容器网络。

另外，容器也可以以更简单的主机网络模式运行。在这种情况下，运行在集群或独立主机中的容器会在主机机器的网络接口上公开端口，以发送和接收网络流量。容器本身仍然可能有它们的 IP 地址，这些地址通过 Docker 映射到主机上的物理网络接口。当您的微服务需要主要与容器化基础设施之外存在的服务进行通信时，这种类型的网络配置非常有用。

默认情况下，Docker 以**桥接网络模式**运行。`bridge`网络在主机上创建一个单一网络接口，充当与主机上配置的另一个子网进行桥接的桥接器。所有传入（入口）和传出（出口）的网络流量都通过`bridge`网络接口在容器子网和主机之间传输。

在 Linux 环境中安装了 Docker 引擎后，如果运行`ifconfig`命令，Docker 将创建一个名为`docker0`的新虚拟桥接网络接口。该接口将默认创建的 Docker 私有子网（通常为`172.16.0.0/16`）与主机的网络堆栈进行桥接。如果一个容器在默认的 Docker 网络中以 IP 地址`172.17.8.1`运行，并且您尝试联系该 IP 地址，内部路由表将通过`docker0`桥接接口将流量传递到私有子网上容器的 IP 地址。除非通过 Docker 发布端口，否则无法从外部访问该容器的 IP 地址。在本章中，我们将深入探讨 Docker 提供的各种网络驱动程序和配置选项。

在下一个练习中，我们将看看如何在默认的 Docker `bridge`网络中创建 Docker 容器，以及如何将容器端口暴露给外部世界。

## 练习 6.01：Docker 网络实践

默认情况下，在 Docker 中运行容器时，您创建的容器实例将存在于一个 Docker 网络中。Docker 网络是 Docker 用来为在即时 Docker 服务器或 Docker 集群中的服务器上运行的容器分配网络资源的子网集合、规则和元数据。该网络将为容器提供访问同一子网中的其他容器，甚至提供对其他外部网络（包括互联网）的出站（egress）访问。每个 Docker 网络都与一个网络驱动程序相关联，该驱动程序确定了网络在容器所在系统的上下文中的功能。

在这个练习中，您将运行 Docker 容器，并使用基本网络来运行两个简单的 Web 服务器（Apache2 和 NGINX），它们将在几种不同的基本网络场景中暴露端口。然后，您将访问容器的暴露端口，以更多地了解 Docker 网络是如何在最基本的级别上工作的。启动容器并暴露服务端口以使它们可用是最常见的网络场景之一，当您首次开始使用容器化基础设施时：

1.  列出当前在您的 Docker 环境中配置的网络，使用`docker network ls`命令：

```
$ docker network ls
```

显示的输出将显示系统中所有配置的 Docker 网络。它应该类似于以下内容：

```
NETWORK ID      NAME      DRIVER     SCOPE
0774bdf6228d    bridge    bridge     local
f52b4a5440ad    host      host       local
9bed60b88784    none      null       local
```

1.  在 Docker 中创建容器时，如果没有指定网络或网络驱动程序，Docker 将使用`bridge`网络创建容器。这个网络存在于您的主机操作系统中配置的`bridge`网络接口后面。在 Linux 或 macOS 的 Bash shell 中使用`ifconfig`，或在 Windows PowerShell 中使用`ipconfig`，来查看 Docker 桥接口配置为哪个接口。通常被称为`docker0`：

```
$ ifconfig 
```

此命令的输出将列出环境中所有可用的网络接口，如下图所示：

![图 6.1：列出可用的网络接口](img/B15021_06_01.jpg)

图 6.1：列出可用的网络接口

可以观察到在前面的图中，Docker 的`bridge`接口被称为`docker0`，并且具有 IP 地址`172.17.0.1`。

1.  使用`docker run`命令创建一个简单的 NGINX Web 服务器容器，使用`latest`镜像标签。使用`-d`标志将容器设置为在后台启动，并使用`--name`标志为其指定一个可读的名称`webserver1`：

```
$ docker run -d –-name webserver1 nginx:latest 
```

如果命令成功，终端会没有返回任何输出。

1.  执行`docker ps`命令来检查容器是否正在运行：

```
$ docker ps
```

如您所见，`webserver1`容器正在如预期地运行：

```
CONTAINER ID  IMAGE         COMMAND                 CREATED
  STATUS                   PORTS               NAMES
0774bdf6228d  nginx:latest  "nginx -g 'daemon of…"  4 seconds ago
  Up 3 seconds             80/tcp              webserver1
```

1.  执行`docker inspect`命令来检查这个容器默认的网络配置：

```
$ docker inspect webserver1
```

Docker 将以 JSON 格式返回有关正在运行的容器的详细信息。在这个练习中，重点关注`NetworkSettings`块。特别注意`networks`子块下面的`Gateway`、`IPAddress`、`Ports`和`NetworkID`参数：

![图 6.2：docker inspect 命令的输出](img/B15021_06_02.jpg)

图 6.2：docker inspect 命令的输出

从这个输出可以得出结论，这个容器存在于默认的 Docker`bridge`网络中。观察`NetworkID`的前 12 个字符，您会发现它与`docker network ls`命令的输出中使用的标识符相同，该命令是在*步骤 1*中执行的。还应该注意，这个容器配置使用的`Gateway`是`docker0``bridge`接口的 IP 地址。Docker 将使用这个接口作为访问自身以外子网中的网络的出口点，同时将来自我们环境的流量转发到子网中的容器。还可以观察到，这个容器在 Docker 桥接网络中有一个唯一的 IP 地址，在本例中为`172.17.0.2`。由于我们有`docker0``bridge`接口可用来转发流量，我们的本地机器可以路由到这个子网。最后，可以观察到，默认情况下，NGINX 容器正在暴露 TCP 端口`80`用于传入流量。

1.  在 Web 浏览器中，通过 IP 地址和端口`80`访问`webserver1`容器。在您喜欢的 Web 浏览器中输入`webserver1`容器的 IP 地址：![图 6.3：通过 IP 地址访问 NGINX Web 服务器容器默认的 Docker 桥接网络](img/B15021_06_03.jpg)

图 6.3：通过默认的 Docker 桥接网络通过 IP 地址访问 NGINX Web 服务器容器

1.  或者，使用`curl`命令查看类似的输出，尽管是以文本格式：

```
$ curl 172.17.0.2:80
```

以下 HTML 响应表示您已从正在运行的 NGINX 容器收到响应：

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully 
installed and working. Further configuration is required.</p>
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

1.  访问本地`bridge`子网中容器的 IP 地址对于测试本地容器非常有效。要将您的服务暴露给其他用户或服务器的网络，请在`docker run`命令中使用`-p`标志。这将允许您将主机上的端口映射到容器上的公开端口。这类似于路由器或其他网络设备上的端口转发。要通过端口向外部世界暴露容器，请使用`docker run`命令，后跟`-d`标志以在后台启动容器。`-p`标志将使您能够指定主机上的端口，用冒号分隔，并指定要公开的容器上的端口。还要为此容器指定一个唯一的名称，`webserver2`：

```
$ docker run -d -p 8080:80 –-name webserver2 nginx:latest
```

成功启动容器后，您的 shell 将不会返回任何内容。但是，某些版本的 Docker 可能会显示完整的容器 ID。

1.  运行`docker ps`命令，检查是否有两个正在运行的 NGINX 容器：

```
$ docker ps
```

将显示两个正在运行的容器，`webserver1`和`webserver2`：

```
CONTAINER ID IMAGE         COMMAND                 CREATED
  STATUS              PORTS                  NAMES
b945fa75b59a nginx:latest  "nginx -g 'daemon of…"  1 minute ago
  Up About a minute   0.0.0.0:8080->80/tcp   webserver2
3267bf4322ed nginx:latest  "nginx -g 'daemon of…"  2 minutes ago
  Up 2 minutes        80/tcp                 webserver1
```

在`PORTS`列中，您将看到 Docker 现在正在将`webserver`容器上的端口`80`转发到主机上的端口`8080`。这是从输出的`0.0.0.0:8080->80/tcp`部分推断出来的。

注意

重要的是要记住，使用`-p`标志指定端口时，主机机器端口始终位于冒号的左侧，而容器端口位于右侧。

1.  在您的网络浏览器中，导航至`http://localhost:8080`，以查看您刚刚生成的运行容器实例：![图 6.4：NGINX 默认页面，指示您已成功转发将端口映射到您的 Web 服务器容器](img/B15021_06_04.jpg)

图 6.4：NGINX 默认页面，指示您已成功将端口转发到您的 Web 服务器容器

1.  现在，在相同的 Docker 环境中运行两个 NGINX 实例，具有略有不同的网络配置。`webserver1`实例仅在 Docker 网络上运行，没有任何端口暴露。使用`docker inspect`命令检查`webserver2`实例的配置，后跟容器名称或 ID：

```
$ docker inspect webserver2
```

JSON 输出底部的`NetworkSettings`部分将类似于以下内容。请特别注意`networks`子块下面的参数（`Gateway`、`IPAddress`、`Ports`和`NetworkID`）：

![图 6.5：docker inspect 命令的输出](img/B15021_06_05.jpg)

图 6.5：docker inspect 命令的输出

正如`docker inspect`输出显示的那样，`webserver2`容器的 IP 地址为`172.17.0.3`，而您的`webserver1`容器的 IP 地址为`172.17.0.1`。根据 Docker 分配 IP 地址给容器的方式，您本地环境中的 IP 地址可能略有不同。这两个容器都位于同一个 Docker 网络（`bridge`）上，并且具有相同的默认网关，即主机上的`docker0` `bridge`接口。

1.  由于这两个容器都位于同一个子网上，您可以在 Docker`bridge`网络内测试容器之间的通信。运行`docker exec`命令以访问`webserver1`容器上的 shell：

```
docker exec -it webserver1 /bin/bash
```

提示符应明显更改为根提示符，表示您现在在`webserver1`容器的 Bash shell 中：

```
root@3267bf4322ed:/#
```

1.  在根 shell 提示符下，使用`apt`软件包管理器在此容器中安装`ping`实用程序：

```
root@3267bf4322ed:/# apt-get update && apt-get install -y inetutils-ping
```

然后，aptitude 软件包管理器将在`webserver1`容器中安装`ping`实用程序。请注意，`apt`软件包管理器将安装`ping`以及运行`ping`命令所需的其他依赖项：

![图 6.6：在 Docker 容器内安装 ping 命令](img/B15021_06_06.jpg)

图 6.6：在 Docker 容器内安装 ping 命令

1.  安装`ping`实用程序后，使用它来 ping 另一个容器的 IP 地址：

```
root@3267bf4322ed:/# ping 172.17.0.3
```

输出应显示 ICMP 响应数据包，表明容器可以通过 Docker`bridge`网络成功 ping 通彼此：

```
PING 172.17.0.1 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: icmp_seq=0 ttl=64 time=0.221 ms
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.207 ms
```

1.  您还可以使用`curl`命令访问 NGINX 默认的 Web 界面。使用`apt`软件包管理器安装`curl`：

```
root@3267bf4322ed:/# apt-get install -y curl
```

接下来的输出应显示，正在安装`curl`实用程序和所有必需的依赖项：

![图 6.7：安装 curl 实用程序](img/B15021_06_07.jpg)

图 6.7：安装 curl 实用程序

1.  安装`curl`后，使用它来 curl`webserver2`的 IP 地址：

```
root@3267bf4322ed:/# curl 172.17.0.3
```

您应该看到以 HTML 格式显示的“欢迎使用 nginx！”页面，这表明您能够通过 Docker`bridge`网络成功联系到`webserver2`容器的 IP 地址：

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully 
installed and working. Further configuration is required.</p>
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

由于您正在使用`curl`导航到 NGINX 欢迎页面，它将以原始 HTML 格式呈现在您的终端显示器上。

在这一部分，我们已经成功在同一个 Docker 环境中生成了两个 NGINX Web 服务器实例。我们配置了一个实例，不在默认 Docker 网络之外暴露任何端口，而我们配置了第二个 NGINX 实例在同一网络上运行，但将端口`80`暴露给主机系统的端口`8080`。我们看到这些容器可以使用标准的互联网浏览器以及 Linux 中的`curl`实用程序进行访问。

在这个练习中，我们还看到了容器如何使用 Docker 网络直接与其他容器通信。我们使用`webserver1`容器调用`webserver2`容器的 IP 地址，并显示容器托管的网页的输出。

在这个练习中，我们还能够演示容器实例之间使用本机 Docker`bridge`网络进行网络连接。然而，当我们大规模部署容器时，很难知道 Docker 网络中的哪个 IP 地址属于哪个容器。

在接下来的部分，我们将看看本机 Docker DNS，并学习如何使用可靠的人类可读的 DNS 名称将网络流量可靠地发送到其他容器实例。

# 本机 Docker DNS

运行容器化基础架构的最大好处之一是能够快速轻松地横向扩展工作负载。在一个具有共享`overlay`网络的集群中有多台机器意味着你可以在多台服务器上运行许多容器。

正如我们在前面的练习中看到的，Docker 赋予我们的权力允许容器通过 Docker 提供的各种网络驱动程序（如`bridge`、`macvlan`和`overlay`驱动程序）直接与集群中的其他容器通信。在前面的例子中，我们利用 Docker`bridge`网络使容器能够通过各自的 IP 地址相互通信。然而，当您的容器部署在真实服务器上时，通常不能依赖容器具有一致的 IP 地址来相互通信。每当一个新的容器实例终止或重新生成时，Docker 都会给该容器一个新的 IP 地址。

类似于传统基础设施场景，我们可以利用容器网络内的 DNS 来为容器提供可靠的通信方式。通过为 Docker 网络中的容器分配可读的名称，用户不再需要每次想要在 Docker 网络上的容器之间发起通信时查找 IP 地址。Docker 本身将跟踪容器的 IP 地址，因为它们生成和重新生成。

在旧版本的 Docker 中，可以通过在`docker run`命令中使用`--link`标志在容器之间建立链接来实现简单的 DNS 解析。使用链接，Docker 会在链接的容器的`hosts`文件中创建一个条目，从而实现简单的名称解析。然而，正如你将在即将进行的练习中看到的，使用容器之间的链接可能会很慢，不可扩展，并且容易出错。最近的 Docker 版本支持在同一 Docker 网络上运行的容器之间的原生 DNS 服务。这允许容器查找在同一 Docker 网络中运行的其他容器的名称。这种方法的唯一注意事项是，原生 Docker DNS 在默认的 Docker `bridge`网络上不起作用；因此，必须首先创建其他网络来构建您的容器。

为了使原生 Docker DNS 工作，我们必须首先使用`docker network create`命令创建一个新的网络。然后，我们可以使用`docker run`命令和`--network-alias`标志在该网络中创建新的容器。在接下来的练习中，我们将使用这些命令来学习原生 Docker DNS 是如何工作的，以实现容器实例之间的可扩展通信。

## 练习 6.02：使用 Docker DNS

在接下来的练习中，您将学习在运行在同一网络上的 Docker 容器之间的名称解析。您将首先使用传统的链接方法启用简单的名称解析。然后，您将通过使用更新的、更可靠的原生 Docker DNS 服务来对比这种方法：

1.  首先，在默认的 Docker `bridge`网络上创建两个 Alpine Linux 容器，它们将使用`--link`标志相互通信。Alpine 是这个练习的一个很好的基础镜像，因为它默认包含`ping`实用程序。这将使您能够快速测试各种情况下容器之间的连接。要开始，请创建一个名为`containerlink1`的容器，以指示您是使用传统的链接方法创建了这个容器。

```
$ docker run -itd --name containerlink1 alpine:latest
```

这将在名为`containerlink1`的默认 Docker 网络中启动一个容器。

1.  在默认的 Docker 桥接网络中启动另一个名为`containerlink2`的容器，它将创建一个到`containerlink1`的链接以启用基本的 DNS：

```
$ docker run -itd --name containerlink2 --link containerlink1 alpine:latest
```

这将在名为`containerlink2`的默认 Docker 网络中启动一个容器。

1.  运行`docker exec`命令以访问`containerlink2`容器内部的 shell。这将允许您调查链接功能的工作方式。由于此容器正在运行 Alpine Linux，默认情况下您无法访问 Bash shell。而是使用`sh` shell 进行访问：

```
$ docker exec -it containerlink2 /bin/sh
```

这应该将您放入`containerlink2`容器中的 root `sh` shell 中。

1.  从`containerlink2`容器的 shell 中，ping `containerlink1`：

```
/ # ping containerlink1
```

您将收到`ping`请求的回复：

```
PING container1 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.307 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.162 ms
64 bytes from 172.17.0.2: seq=2 ttl=64 time=0.177 ms
```

1.  使用`cat`实用程序查看`containerlink2`容器的`/etc/hosts`文件。`hosts`文件是 Docker 可以维护和覆盖的可路由名称到 IP 地址的列表：

```
/ # cat /etc/hosts
```

`hosts`文件的输出应该显示并类似于以下内容：

```
127.0.0.1  localhost
::1  localhost ip6-localhost ip6-loopback
fe00::0    ip6-localnet
ff00::0    ip6-mcastprefix
ff02::1    ip6-allnodes
ff02::2    ip6-allrouters
172.17.0.2    containerlink1 032f038abfba
172.17.0.3    9b62c4a57ce3
```

从`containerlink2`容器的`hosts`文件输出中，观察到 Docker 正在为`containerlink1`容器名称以及其容器 ID 添加条目。这使得`containerlink2`容器可以知道名称，并且容器 ID 映射到 IP 地址`172.17.0.2`。输入`exit`命令将终止`sh` shell 会话，并将您带回到您环境的主终端。

1.  运行`docker exec`命令以访问`containerlink1`容器内部的`sh` shell：

```
$ docker exec -it containerlink1 /bin/sh
```

这应该将您放入`containerlink1`容器的 shell 中。

1.  使用`ping`实用程序对`containerlink2`容器进行 ping 测试：

```
/ # ping containerlink2
```

您应该看到以下输出：

```
ping: bad address 'containerlink2'
```

由于容器之间的链接只能单向工作，所以无法对`containerlink2`容器进行 ping 测试。`containerlink1`容器不知道`containerlink2`容器的存在，因为在`containerlink1`容器实例中没有创建`hosts`文件条目。

注意

您只能使用容器之间的传统链接方法链接到运行中的容器。这意味着第一个容器不能链接到稍后启动的容器。这是使用容器之间的链接不再是推荐方法的许多原因之一。我们在本章中介绍这个概念，以向您展示功能是如何工作的。

1.  由于使用传统链接方法存在限制，Docker 还支持使用用户创建的 Docker 网络来支持本机 DNS。为了利用这个功能，创建一个名为`dnsnet`的 Docker 网络，并在该网络中部署两个 Alpine 容器。首先，使用`docker network create`命令创建一个新的 Docker 网络，使用`192.168.56.0/24`子网和 IP 地址`192.168.54.1`作为默认网关：

```
$ docker network create dnsnet --subnet 192.168.54.0/24 --gateway 192.168.54.1
```

根据您使用的 Docker 版本，成功执行此命令可能会返回您创建的网络的 ID。

注意

仅使用`docker network create dnsnet`命令将创建一个具有 Docker 分配的子网和网关的网络。此练习演示了如何为 Docker 网络指定子网和网关。还应注意，如果您的计算机连接到`192.168.54.0/24`子网或与该空间重叠的子网，可能会导致网络连接问题。请为此练习使用不同的子网。

1.  使用`docker network ls`命令列出此环境中可用的 Docker 网络：

```
$ docker network ls
```

应返回 Docker 网络列表，包括您刚刚创建的`dnsnet`网络：

```
NETWORK ID      NAME       DRIVER     SCOPE
ec5b91e88a6f    bridge     bridge     local
c804e768413d    dnsnet     bridge     local
f52b4a5440ad    host       host       local
9bed60b88784    none       null       local
```

1.  运行`docker network inspect`命令查看此网络的配置：

```
$ docker network inspect dnsnet
```

应显示`dnsnet`网络的详细信息。特别注意`子网`和`网关`参数。这些是您在*步骤 8*中用来创建 Docker 网络的相同参数：

![图 6.8：来自 docker network inspect 命令的输出](img/B15021_06_08.jpg)

图 6.8：来自 docker network inspect 命令的输出

1.  由于这是一个 Docker“桥接”网络，Docker 还将为此网络创建一个相应的桥接网络接口。桥接网络接口的 IP 地址将与您在创建此网络时指定的默认网关地址相同。使用`ifconfig`命令在 Linux 或 macOS 上查看配置的网络接口。如果您使用 Windows，请使用`ipconfig`命令：

```
$ ifconfig
```

这应该显示所有可用网络接口的输出，包括新创建的`bridge`接口：

![图 6.9：分析新创建的 Docker 网络的桥接网络接口](img/B15021_06_09.jpg)

图 6.9：分析新创建的 Docker 网络的桥接网络接口

1.  现在已经创建了一个新的 Docker 网络，使用`docker run`命令在此网络中启动一个新的容器（`alpinedns1`）。使用`docker run`命令，使用`--network`标志指定刚刚创建的`dnsnet`网络，并使用`--network-alias`标志为容器指定自定义 DNS 名称：

```
$ docker run -itd --network dnsnet --network-alias alpinedns1 --name alpinedns1 alpine:latest
```

成功执行命令后，应显示完整的容器 ID，然后返回到正常的终端提示符。

1.  使用相同的`--network`和`--network-alias`设置启动第二个容器（`alpinedns2`）：

```
$ docker run -itd --network dnsnet --network-alias alpinedns2 --name alpinedns2 alpine:latest
```

注意

重要的是要理解`--network-alias`标志和`--name`标志之间的区别。`--name`标志用于在 Docker API 中为容器指定一个易于阅读的名称。这使得通过名称轻松启动、停止、重新启动和管理容器。然而，`--network-alias`标志用于为容器创建自定义 DNS 条目。

1.  使用`docker ps`命令验证容器是否按预期运行：

```
$ docker ps 
```

输出将显示正在运行的容器实例：

```
CONTAINER ID    IMAGE           COMMAND      CREATED 
  STATUS              PORTS             NAMES
69ecb9ad45e1    alpine:latest   "/bin/sh"    4 seconds ago
  Up 2 seconds                          alpinedns2
9b57038fb9c8    alpine:latest   "/bin/sh"    6 minutes ago
  Up 6 minutes                          alpinedns1
```

1.  使用`docker inspect`命令验证容器实例的 IP 地址是否来自指定的子网（`192.168.54.0/24`）：

```
$ docker inspect alpinedns1
```

以下输出被截断以显示相关细节：

![图：6.10：alpinedns1 容器实例的网络部分输出](img/B15021_06_10.jpg)

图：6.10：alpinedns1 容器实例的网络部分输出

可以从输出中观察到，`alpinedns1`容器部署时具有 IP 地址`192.168.54.2`，这是在创建 Docker 网络时定义的子网的一部分。

1.  以类似的方式执行`docker network inspect`命令，针对`alpinedns2`容器：

```
$ docker inspect alpinedns2
```

输出再次被截断以显示相关的网络细节：

![图 6.11：alpinedns2 容器实例的网络部分输出](img/B15021_06_11.jpg)

图 6.11：alpinedns2 容器实例的网络部分输出

可以观察到在前面的输出中，`alpinedns2`容器具有 IP 地址`192.168.54.3`，这是`dnsnet`子网内的不同 IP 地址。

1.  运行`docker exec`命令以访问`alpinedns1`容器中的 shell：

```
$ docker exec -it alpinedns1 /bin/sh
```

这应该将您放入容器内的 root shell 中。

1.  进入`alpinedns1`容器后，使用`ping`实用程序对`alpinedns2`容器进行 ping 测试：

```
/ # ping alpinedns2
```

`ping`输出应显示与`alpinedns2`容器实例的成功网络连接：

```
PING alpinedns2 (192.168.54.3): 56 data bytes
64 bytes from 192.168.54.3: seq=0 ttl=64 time=0.278 ms
64 bytes from 192.168.54.3: seq=1 ttl=64 time=0.233 ms
```

1.  使用`exit`命令返回到主要终端。使用`docker exec`命令访问`alpinedns2`容器内的 shell：

```
$ docker exec -it alpinedns2 /bin/sh
```

这将使您进入`alpinedns2`容器内的 shell。

1.  使用`ping`实用程序通过名称 ping`alpinedns1`容器：

```
$ ping alpinedns1
```

输出应显示来自`alpinedns1`容器的成功响应：

```
PING alpinedns1 (192.168.54.2): 56 data bytes
64 bytes from 192.168.54.2: seq=0 ttl=64 time=0.115 ms
64 bytes from 192.168.54.2: seq=1 ttl=64 time=0.231 ms
```

注意

与传统链接方法相比，Docker DNS 允许在同一 Docker 网络中的容器之间进行双向通信。

1.  在任何`alpinedns`容器内使用`cat`实用程序来揭示 Docker 正在使用真正的 DNS，而不是容器内的`/etc/hosts`文件条目：

```
# cat /etc/hosts
```

这将显示各自容器内`/etc/hosts`文件的内容：

```
127.0.0.1  localhost
::1  localhost ip6-localhost ip6-loopback
fe00::0    ip6-localnet
ff00::0    ip6-mcastprefix
ff02::1    ip6-allnodes
ff02::2    ip6-allrouters
192.168.54.2    9b57038fb9c8
```

使用`exit`命令终止`alpinedns2`容器内的 shell 会话。

1.  使用`docker stop`命令停止所有正在运行的容器来清理您的环境：

```
$ docker stop  containerlink1
$ docker stop  containerlink2
$ docker stop  alpinedns1
$ docker stop  alpinedns2
```

1.  使用`docker system prune -fa`命令清理剩余的已停止容器和网络：

```
$ docker system prune -fa
```

成功执行此命令应清理`dnsnet`网络以及容器实例和镜像：

```
Deleted Containers:
69ecb9ad45e16ef158539761edc95fc83b54bd2c0d2ef55abfba1a300f141c7c
9b57038fb9c8cf30aaebe6485e9d223041a9db4e94eb1be9392132bdef632067
Deleted Networks:
dnsnet
Deleted Images:
untagged: alpine:latest
untagged: alpine@sha256:9a839e63dad54c3a6d1834e29692c8492d93f90c
    59c978c1ed79109ea4fb9a54
deleted: sha256:f70734b6a266dcb5f44c383274821207885b549b75c8e119
    404917a61335981a
deleted: sha256:3e207b409db364b595ba862cdc12be96dcdad8e36c59a03b
    b3b61c946a5741a
Total reclaimed space: 42.12M
```

系统清理输出的每个部分都将识别并删除不再使用的 Docker 资源。在这种情况下，它将删除`dnsnet`网络，因为当前在该网络中没有部署容器实例。

在这个练习中，您看到了使用名称解析在 Docker 网络上启用容器之间通信的好处。使用名称解析是高效的，因为应用程序不必担心其他正在运行的容器的 IP 地址。相反，通信可以通过简单地按名称调用其他容器来启动。

我们首先探讨了名称解析的传统链接方法，通过该方法，运行的容器可以建立关系，利用容器的`hosts`文件中的条目进行单向关系。使用 DNS 在容器之间进行通信的第二种更现代的方法是创建用户定义的 Docker 网络，允许双向 DNS 解析。这将使网络上的所有容器都能够通过名称或容器 ID 解析所有其他容器，而无需任何额外的配置。

正如我们在本节中所见，Docker 提供了许多独特的方法来为容器实例提供可靠的网络资源，例如在相同的 Docker 网络上启用容器之间的路由和容器之间的本机 DNS 服务。这只是 Docker 提供的网络选项的冰山一角。

在下一节中，我们将学习如何使用其他类型的网络驱动程序部署容器，以便在部署容器化基础架构时提供最大的灵活性。

# 本机 Docker 网络驱动程序

由于 Docker 是近年来最广泛支持的容器平台之一，Docker 平台已在许多生产级网络场景中得到验证。为了支持各种类型的应用程序，Docker 提供了各种网络驱动程序，可以灵活地创建和部署容器。这些网络驱动程序允许容器化应用程序在几乎任何直接在裸机或虚拟化服务器上支持的网络配置中运行。

例如，可以部署共享主机服务器网络堆栈的容器，或者以允许它们从底层网络基础设施分配唯一 IP 地址的配置。在本节中，我们将学习基本的 Docker 网络驱动程序以及如何利用它们为各种类型的网络基础设施提供最大的兼容性：

+   `bridge`：`bridge`是 Docker 将容器运行在其中的默认网络。如果在启动容器实例时没有定义任何内容，Docker 将使用`docker0`接口后面的子网，其中容器将被分配在`172.17.0.0/16`子网中的 IP 地址。在`bridge`网络中，容器可以与`bridge`子网中的其他容器进行网络连接，也可以与互联网进行出站连接。到目前为止，在本章中创建的所有容器都在`bridge`网络中。Docker 的`bridge`网络通常用于仅公开简单端口或需要与同一主机上存在的其他容器进行通信的简单 TCP 服务。

+   `host`：以`host`网络模式运行的容器直接访问主机机器的网络堆栈。这意味着容器暴露的任何端口也会暴露给运行容器的主机机器上的相同端口。容器还可以看到主机上运行的所有物理和虚拟网络接口。通常在运行消耗大量带宽或利用多个协议的容器实例时，会首选`host`网络。

+   `none`：`none`网络不提供网络连接给部署在该网络中的容器。在`none`网络中部署的容器实例只有一个环回接口，根本无法访问其他网络资源。没有驱动程序操作这个网络。使用`none`网络模式部署的容器通常是在存储或磁盘工作负载上运行的应用程序，不需要网络连接。出于安全目的而被隔离在网络连接之外的容器也可以使用这个网络驱动程序进行部署。

+   `macvlan`：在 Docker 中创建的`macvlan`网络用于容器化应用程序需要 MAC 地址和直接网络连接到底层网络的情况。使用`macvlan`网络，Docker 将通过主机机器上的物理接口为容器实例分配一个 MAC 地址。这使得您的容器在部署的网络段上看起来像是一个物理主机。需要注意的是，许多云环境，如 AWS、Azure 和许多虚拟化 hypervisor，不允许在容器实例上配置`macvlan`网络。`macvlan`网络允许 Docker 根据连接到主机机器的物理网络接口，为容器分配 IP 地址和 MAC 地址。如果未正确配置，使用`macvlan`网络可能会很容易导致 IP 地址耗尽或 IP 地址冲突。`macvlan`容器网络通常用于非常特定的网络用例，例如监视网络流量模式或其他网络密集型工作负载的应用程序。

关于 Docker 网络的讨论如果没有对**Docker 叠加网络**进行简要概述，就不完整。`叠加`网络是 Docker 处理与集群的网络的方式。当在节点之间定义 Docker 集群时，Docker 将使用将节点连接在一起的物理网络来定义在节点上运行的容器之间的逻辑网络。这允许容器在集群节点之间直接通信。在*练习 6.03，探索 Docker 网络*中，我们将看看 Docker 默认支持的各种类型的 Docker 网络驱动程序，如`host`、`none`和`macvlan`。在*练习 6.04*中，*定义叠加网络*，我们将定义一个简单的 Docker 集群，以发现在集群模式下配置的 Docker 主机之间的`叠加`网络是如何工作的。

## 练习 6.03：探索 Docker 网络

在这个练习中，我们将研究 Docker 默认支持的各种类型的网络驱动程序，如`host`、`none`和`macvlan`。我们将从`bridge`网络开始，然后再看看`none`、`host`和`macvlan`网络：

1.  首先，您需要了解在您的 Docker 环境中如何设置网络。从 Bash 或 PowerShell 终端，在 Windows 上使用`ifconfig`或`ipconfig`命令。这将显示您的 Docker 环境中的所有网络接口：

```
$ ifconfig
```

这将显示您可用的所有网络接口。您应该看到一个名为`docker0`的`bridge`接口。这是 Docker 的`bridge`接口，用作默认 Docker 网络的入口（或入口点）：

![图 6.12：来自您的 Docker 开发环境的示例 ifconfig 输出](img/B15021_06_12.jpg)

图 6.12：来自您的 Docker 开发环境的示例 ifconfig 输出

1.  使用`docker network ls`命令查看您的 Docker 环境中可用的网络：

```
$ docker network ls
```

这应该列出之前定义的三种基本网络类型，显示网络 ID、Docker 网络的名称和与网络类型相关联的驱动程序：

```
NETWORK ID       NAME      DRIVER     SCOPE
50de4997649a     bridge    bridge     local
f52b4a5440ad     host      host       local
9bed60b88784     none      null       local
```

1.  使用`docker network inspect`命令查看这些网络的详细信息，然后跟上要检查的网络的 ID 或名称。在这一步中，您将查看`bridge`网络的详细信息：

```
$ docker network inspect bridge
```

Docker 将以 JSON 格式显示`bridge`网络的详细输出：

![图 6.13：检查默认的桥接网络](img/B15021_06_13.jpg)

图 6.13：检查默认的桥接网络

在这个输出中需要注意的一些关键参数是`Scope`、`Subnet`和`Gateway`关键字。根据这个输出，可以观察到这个网络的范围只是本地主机（`Scope: Local`）。这表明该网络不在 Docker 集群中的主机之间共享。在`Config`部分下，这个网络的`Subnet`值是`172.17.0.0/16`，子网的`Gateway`地址是定义的子网内的 IP 地址（`172.17.0.1`）。子网的`Gateway`值是子网内的 IP 地址，以便在该子网中部署的容器可以访问该网络范围之外的其他网络。最后，这个网络与主机接口`docker0`绑定，它将作为网络的`bridge`接口。`docker network inspect`命令的输出对于全面了解在该网络中部署的容器预期行为非常有帮助。

1.  使用`docker network inspect`命令查看`host`网络的详细信息：

```
$ docker network inspect host
```

这将以 JSON 格式显示`host`网络的详细信息：

![图 6.14：主机网络的 docker 网络检查输出](img/B15021_06_14.jpg)

图 6.14：主机网络的 docker 网络检查输出

正如你所看到的，`host`网络中没有太多的配置。由于它使用`host`网络驱动程序，所有容器的网络将与主机共享。因此，这个网络配置不需要定义特定的子网、接口或其他元数据，就像我们之前在默认的`bridge`网络中看到的那样。

1.  接下来调查`none`网络。使用`docker network inspect`命令查看`none`网络的详细信息：

```
docker network inspect none
```

详细信息将以 JSON 格式显示：

![图 6.15：none 网络的 docker 网络检查输出](img/B15021_06_15.jpg)

图 6.15：none 网络的 docker 网络检查输出

与`host`网络类似，`none`网络大部分是空的。由于在这个网络中部署的容器将通过`null`驱动程序没有网络连接，因此不需要太多的配置。

注意

请注意，`none` 和 `host` 网络之间的区别在于它们使用的驱动程序，尽管配置几乎相同。在 `none` 网络中启动的容器根本没有网络连接，并且没有网络接口分配给容器实例。然而，在 `host` 网络中启动的容器将与主机系统共享网络堆栈。

1.  现在在 `none` 网络中创建一个容器以观察其操作。在您的终端或 PowerShell 会话中，使用 `docker run` 命令使用 `--network` 标志在 `none` 网络中启动一个 Alpine Linux 容器。将此容器命名为 `nonenet`，以便我们知道它部署在 `none` 网络中：

```
$ docker run -itd --network none --name nonenet alpine:latest 
```

这将在 `none` 网络中拉取并启动一个 Alpine Linux Docker 容器。

1.  使用 `docker ps` 命令验证容器是否按预期运行：

```
$ docker ps 
```

输出应显示 `nonenet` 容器已启动并运行：

```
CONTAINER ID    IMAGE            COMMAND      CREATED 
  STATUS              PORTS              NAMES
972a80984703    alpine:latest    "/bin/sh"    9 seconds ago
  Up 7 seconds                           nonenet
```

1.  执行 `docker inspect` 命令，以及容器名称 `nonenet`，以更深入地了解此容器的配置：

```
$ docker inspect nonenet
```

`docker inspect` 的输出将以 JSON 格式显示完整的容器配置。这里提供了一个突出显示 `NetworkSettings` 部分的缩略版本。请特别注意 `IPAddress` 和 `Gateway` 设置：

![图 6.16：nonenet 容器的 docker inspect 输出](img/B15021_06_16.jpg)

图 6.16：nonenet 容器的 docker inspect 输出

`docker inspect` 输出将显示该容器没有 IP 地址，也没有网关或任何其他网络设置。

1.  使用 `docker exec` 命令访问此容器内部的 `sh` shell：

```
$ docker exec -it nonenet /bin/sh
```

成功执行此命令后，您将进入容器实例中的 root shell：

```
/ #
```

1.  执行 `ip a` 命令查看容器中可用的网络接口：

```
/ $ ip a 
```

这将显示在此容器中配置的所有网络接口：

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state 
UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    valid_lft forever preferred_lft forever
```

此容器可用的唯一网络接口是其 `LOOPBACK` 接口。由于此容器未配置 IP 地址或默认网关，常见的网络命令将无法使用。

1.  使用 Alpine Linux Docker 镜像中默认提供的 `ping` 实用程序测试网络连接的缺失。尝试 ping IP 地址为 `8.8.8.8` 的谷歌 DNS 服务器：

```
/ # ping 8.8.8.8
```

`ping` 命令的输出应显示它没有网络连接：

```
PING 8.8.8.8 (8.8.8.8): 56 data bytes
ping: sendto: Network unreachable
```

使用`exit`命令返回到主终端会话。

现在您已经仔细查看了`none`网络，请考虑`host`网络驱动程序。Docker 中的`host`网络驱动程序是独特的，因为它没有任何中间接口或创建任何额外的子网。相反，`host`网络驱动程序与主机操作系统共享网络堆栈，因此主机可用的任何网络接口也适用于以`host`模式运行的容器。

1.  要开始以`host`模式运行容器，请执行`ifconfig`（如果在 macOS 或 Linux 上运行）或使用`ipconfig`（如果在 Windows 上运行）来列出主机机器上可用的网络接口清单：

```
$ ifconfig
```

这应该输出主机机器上可用的网络接口列表：

![图 6.17：主机机器上配置的网络接口列表](img/B15021_06_12.jpg)

图 6.17：主机机器上配置的网络接口列表

在此示例中，您的主机的主要网络接口是`enp1s0`，IP 地址为`192.168.122.185`。

注意

在 macOS 或 Windows 上的某些版本的 Docker Desktop 可能无法正确启动和运行`host`网络模式或使用`macvlan`网络驱动程序的容器，因为这些功能依赖于 Linux 内核。在 macOS 或 Windows 上运行这些示例时，您可能会看到运行 Docker 的基础 Linux 虚拟机的网络详细信息，而不是 macOS 或 Windows 主机机器上可用的网络接口。

1.  使用`docker run`命令在`host`网络中启动一个 Alpine Linux 容器。将其命名为`hostnet1`以区分其他容器：

```
docker run -itd --network host --name hostnet1 alpine:latest
```

Docker 将使用`host`网络在后台启动此容器。

1.  使用`docker inspect`命令查看刚创建的`hostnet1`容器的网络配置：

```
$ docker inspect hostnet1
```

这将以 JSON 格式显示运行容器的详细配置，包括网络详细信息：

![图 6.18：hostnet1 容器的 docker inspect 输出](img/B15021_06_18.jpg)

图 6.18：hostnet1 容器的 docker inspect 输出

需要注意的是，`NetworkSettings`块的输出看起来很像您在`none`网络中部署的容器。在“主机”网络模式下，Docker 不会为容器实例分配 IP 地址或网关，因为它直接与主机机器共享所有网络接口。

1.  使用`docker exec`访问此容器内的`sh` shell，提供名称`hostnet1`：

```
$ docker exec -it hostnet1 /bin/sh
```

这应该会将您放入`hostnet1`容器内的 root shell 中。

1.  在`hostnet1`容器内，执行`ifconfig`命令列出可用的网络接口：

```
/ # ifconfig
```

应该显示此容器内可用的完整网络接口列表：

![图 6.19：显示 hostnet1 容器内可用的网络接口](img/B15021_06_19.jpg)

图 6.19：显示 hostnet1 容器内可用的网络接口

请注意，这个网络接口列表与直接查询主机机器时遇到的是相同的。这是因为这个容器和主机机器直接共享网络。对主机机器可用的任何东西也将对在“主机”网络模式下运行的容器可用。

1.  使用`exit`命令结束 shell 会话并返回到主机机器的终端。

1.  为了更充分地了解 Docker 中共享网络模型的工作原理，以“主机”网络模式启动一个 NGINX 容器。NGINX 容器会自动暴露端口`80`，以前我们必须将其转发到主机机器上的一个端口。使用`docker run`命令在主机机器上启动一个 NGINX 容器：

```
$ docker run -itd --network host --name hostnet2 nginx:latest
```

这个命令将在“主机”网络模式下启动一个 NGINX 容器。

1.  在主机机器上使用 Web 浏览器导航至`http://localhost:80`：![图 6.20：访问运行在主机网络模式下的容器的 NGINX 默认网页在主机网络模式下运行](img/B15021_06_20.jpg)

图 6.20：访问在主机网络模式下运行的容器的 NGINX 默认网页

您应该能够在 Web 浏览器中看到 NGINX 默认网页。需要注意的是，`docker run`命令没有明确地将任何端口转发或暴露给主机机器。由于容器在“主机”网络模式下运行，容器默认暴露的任何端口将直接在主机机器上可用。

1.  使用`docker run`命令在`host`网络模式下创建另一个 NGINX 实例。将此容器命名为`hostnet3`，以便与其他两个容器实例区分开来：

```
$ docker run -itd --network host --name hostnet3 nginx:latest
```

1.  现在使用`docker ps -a`命令列出所有容器，包括运行和停止状态的容器：

```
$ docker ps -a
```

将显示运行中的容器列表：

```
CONTAINER ID  IMAGE         COMMAND                CREATED
  STATUS                        PORTS           NAMES
da56fcf81d02  nginx:latest  "nginx -g 'daemon of…" 4 minutes ago
  Exited (1) 4 minutes ago                      hostnet3
5786dac6fd27  nginx:latest  "nginx -g 'daemon of…" 37 minutes ago
  Up 37 minutes                                 hostnet2
648b291846e7  alpine:latest "/bin/sh"              38 minutes ago
  Up 38 minutes                                 hostnet
```

1.  根据上述输出，您可以看到`hostnet3`容器已退出并当前处于停止状态。要更充分地了解原因，使用`docker logs`命令查看容器日志：

```
$ docker logs hostnet3
```

日志输出应显示如下：

![图 6.21：hostnet3 容器中的 NGINX 错误](img/B15021_06_21.jpg)

图 6.21：hostnet3 容器中的 NGINX 错误

基本上，这第二个 NGINX 容器实例无法正常启动，因为它无法绑定到主机上的端口`80`。原因是`hostnet2`容器已经在监听该端口。

注意

请注意，以`host`网络模式运行的容器需要谨慎部署和考虑。如果没有适当的规划和架构，容器的泛滥可能会导致在同一台机器上运行的容器实例之间发生各种端口冲突。

1.  您将要调查的下一个类型的本机 Docker 网络是`macvlan`。在`macvlan`网络中，Docker 将为容器实例分配一个 MAC 地址，使其在特定网络段上看起来像是物理主机。它可以在`bridge`模式下运行，该模式使用父`host`网络接口来获得对底层网络的物理访问，或者在`802.1Q trunk`模式下运行，该模式利用 Docker 动态创建的子接口。

1.  首先，使用`docker network create`命令指定主机上的物理接口作为父接口，通过`macvlan` Docker 网络驱动程序创建一个新网络。

1.  在之前的`ifconfig`或`ipconfig`输出中，你看到`enp1s0`接口是机器上的主要网络接口。替换你的机器的主要网络接口的名称。由于你正在使用主机机器的主要网络接口作为父接口，为我们的容器的网络连接指定相同的子网（或者在该空间内更小的子网）。在这里使用`192.168.122.0/24`子网，因为它是主要网络接口的相同子网。同样，你想要指定与父接口相同的默认网关。使用主机机器的相同子网和网关：

```
$ docker network create -d macvlan --subnet=192.168.122.0/24 --gateway=192.168.122.1 -o parent=enp1s0 macvlan-net1
```

这个命令应该创建一个名为`macvlan-net1`的网络。

1.  使用`docker network ls`命令来确认网络已经被创建，并且正在使用`macvlan`网络驱动程序：

```
$ docker network ls
```

这个命令将输出当前在你的环境中定义的所有网络。你应该会看到`macvlan-net1`网络：

```
NETWORK ID       NAME            DRIVER     SCOPE
f4c9408f22e2     bridge          bridge     local
f52b4a5440ad     host            host       local
b895c821b35f     macvlan-net1    macvlan    local
9bed60b88784     none            null       local
```

1.  现在`macvlan`网络已经在 Docker 中定义，创建一个在这个网络中的容器，并从主机的角度调查网络连接。使用`docker run`命令在`macvlan`网络`macvlan-net1`中创建另一个名为`macvlan1`的 Alpine Linux 容器：

```
$ docker run -itd --name macvlan1 --network macvlan-net1 alpine:latest
```

这应该会在后台启动一个名为`macvlan1`的 Alpine Linux 容器实例。

1.  使用`docker ps -a`命令来检查并确保这个容器实例正在运行：

```
$ docker ps -a
```

这应该显示名为`macvlan1`的容器正在按预期运行：

```
CONTAINER ID   IMAGE           COMMAND      CREATED
  STATUS              PORTS              NAMES
cd3c61276759   alpine:latest   "/bin/sh"    3 seconds ago
  Up 1 second                            macvlan1
```

1.  使用`docker inspect`命令来调查这个容器实例的网络配置：

```
$ docker inspect macvlan1
```

容器配置的详细输出应该被显示出来。以下输出已经被截断，只显示了 JSON 格式的网络设置部分：

![图 6.22：macvlan1 网络的 docker 网络检查输出](img/B15021_06_22.jpg)

图 6.22：macvlan1 网络的 docker 网络检查输出

从这个输出中，你可以看到这个容器实例（类似于其他网络模式的容器）既有 IP 地址又有默认网关。还可以得出结论，这个容器也在`192.168.122.0/24`网络中拥有一个 OSI 模型第 2 层的 MAC 地址，根据`Networks`子部分下的`MacAddress`参数。这个网络段内的其他主机会认为这台机器是另一个物理节点，而不是托管在子网中的节点上的容器。

1.  使用`docker run`创建`macvlan-net1`网络内的第二个容器实例命名为`macvlan2`：

```
$ docker run -itd --name macvlan2 --network macvlan-net1 alpine:latest
```

这应该在`macvlan-net1`网络中启动另一个容器实例。

1.  运行`docker inspect`命令以查看`macvlan-net2`容器实例的 MAC 地址：

```
$ docker inspect macvlan2
```

这将以 JSON 格式输出`macvlan2`容器实例的详细配置，此处仅显示相关的网络设置。

![图 6.23：macvlan2 容器的 docker inspect 输出](img/B15021_06_23.jpg)

图 6.23：macvlan2 容器的 docker inspect 输出

可以在此输出中看到`macvlan2`容器具有与`macvlan1`容器实例不同的 IP 地址和 MAC 地址。Docker 分配不同的 MAC 地址以确保在许多容器使用`macvlan`网络时不会出现第 2 层冲突。

1.  运行`docker exec`命令以访问此容器内的`sh` shell：

```
$ docker exec -it macvlan1 /bin/sh
```

这应该将您放入容器内的 root 会话。

1.  在容器内使用`ifconfig`命令观察在`macvlan1`容器的`docker inspect`输出中看到的 MAC 地址是否存在于容器的主要网络接口的 MAC 地址中：

```
/ # ifconfig
```

在`eth0`接口的详细信息中，查看`HWaddr`参数。您还可以注意`inet addr`参数下列出的 IP 地址，以及通过此网络接口传输和接收的字节数-`RX 字节`（接收的字节数）和`TX 字节`（传输的字节数）：

```
eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:7A:02
          inet addr:192.168.122.2  Bcast:192.168.122.255
                                   Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:353 errors:0 dropped:0 overruns:0 frame:0
          TX packets:188 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:1789983 (1.7 MiB)  TX bytes:12688 (12.3 KiB)
```

1.  使用 Alpine Linux 容器中可用的`apk`软件包管理器安装`arping`实用程序。这是一个用于向 MAC 地址发送`arp`消息以检查第 2 层连接的工具：

```
/ # apk add arping
```

`arping`实用程序应该安装在`macvlan1`容器内：

```
fetch http://dl-cdn.alpinelinux.org/alpine/v3.11/main
/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.11/community
/x86_64/APKINDEX.tar.gz
(1/3) Installing libnet (1.1.6-r3)
(2/3) Installing libpcap (1.9.1-r0)
(3/3) Installing arping (2.20-r0)
Executing busybox-1.31.1-r9.trigger
OK: 6 MiB in 17 packages
```

1.  将`macvlan2`容器实例的第 3 层 IP 地址指定为`arping`的主要参数。现在，`arping`将自动查找 MAC 地址并检查与其的第 2 层连接：

```
/ # arping 192.168.122.3
```

`arping`实用程序应该报告`macvlan2`容器实例的正确 MAC 地址，表明成功的第 2 层网络连接：

```
ARPING 192.168.122.3
42 bytes from 02:42:c0:a8:7a:03 (192.168.122.3): index=0 
time=8.563 usec
42 bytes from 02:42:c0:a8:7a:03 (192.168.122.3): index=1 
time=18.889 usec
42 bytes from 02:42:c0:a8:7a:03 (192.168.122.3): index=2 
time=15.917 use
type exit to return to the shell of your primary terminal. 
```

1.  使用`docker ps -a`命令检查容器的状态：

```
$ docker ps -a 
```

此命令的输出应显示环境中所有正在运行和停止的容器实例。

1.  接下来，使用`docker stop`停止所有正在运行的容器，然后是容器名称或 ID：

```
$ docker stop hostnet1
```

对您环境中的所有运行容器重复此步骤。

1.  使用 `docker system prune` 命令清理容器镜像和未使用的网络：

```
$ docker system prune -fa 
```

这个命令将清理您的机器上剩余的所有未使用的容器镜像、网络和卷。

在这个练习中，我们看了一下 Docker 默认提供的四种默认网络驱动程序：`bridge`、`host`、`macvlan` 和 `none`。对于每个示例，我们探讨了网络的功能，使用这些网络驱动程序部署的容器如何与主机机器一起工作，以及它们如何在网络上与其他容器一起工作。

Docker 默认公开的网络功能可以用来部署非常高级的网络配置，正如我们迄今所见。Docker 还提供了管理和协调集群化的容器网络之间的能力。

在下一节中，我们将看看创建网络，这些网络将在 Docker 主机之间创建覆盖网络，以确保容器实例之间的直接连接。

# Docker Overlay 网络

`Overlay` 网络是在特定目的下在物理（底层）网络之上创建的逻辑网络。例如，**虚拟专用网络**（**VPN**）是一种常见的 `overlay` 网络类型，它使用互联网来创建与另一个私有网络的连接。Docker 可以创建和管理容器之间的 `overlay` 网络，这些网络可以用于容器化应用程序直接相互通信。当容器部署到 `overlay` 网络中时，它们部署在集群中的哪个主机上并不重要；它们将直接连接到同一 `overlay` 网络中存在的其他容器化服务，就像它们存在于同一物理主机上一样。

## 练习 6.04：定义 Overlay 网络

Docker `overlay` 网络用于在 Docker 集群中的机器之间创建网格网络。在这个练习中，您将使用两台机器创建一个基本的 Docker 集群。理想情况下，这些机器将存在于同一个网络段，以确保它们之间的直接网络连接和快速网络连接。此外，它们应该在支持的 Linux 发行版（如 RedHat、CentOS 或 Ubuntu）上运行相同版本的 Docker。

您将定义将跨越 Docker 集群中的主机的 `overlay` 网络。然后，您将确保部署在不同主机上的容器可以通过 `overlay` 网络相互通信：

注意

这个练习需要访问一个安装了 Docker 的辅助机器。通常，基于云的虚拟机或部署在另一个虚拟化程序中的机器效果最好。在使用 Docker Desktop 在系统上部署 Docker 集群可能会导致网络问题或严重的性能下降。

1.  在第一台机器 `Machine1` 上运行 `docker --version` 来查找当前正在运行的 Docker 版本。

```
Machine1 ~$ docker --version
```

将显示 `Machine1` 的 Docker 安装的版本细节：

```
Docker version 19.03.6, build 369ce74a3c
```

然后，您可以对 `Machine2` 执行相同的操作：

```
Machine2 ~$ docker --version
```

将显示 `Machine2` 的 Docker 安装的版本细节：

```
Docker version 19.03.6, build 369ce74a3c
```

在继续之前，验证已安装的 Docker 版本是否相同。

注意

Docker 版本可能会因系统而异。

1.  在 `Machine1` 上，运行 `docker swarm init` 命令来初始化 Docker 集群：

```
Machine1 ~$ docker swarm init
```

这应该打印出您可以在其他节点上使用的命令，以加入 Docker 集群，包括 IP 地址和 `join` 令牌：

```
docker swarm join --token SWMTKN-1-57n212qtvfnpu0ab28tewiorf3j9fxzo9vaa7drpare0ic6ohg-5epus8clyzd9xq7e7ze1y0p0n 
192.168.122.185:2377
```

1.  在 `Machine2` 上，运行由 `Machine1` 提供的 `docker swarm join` 命令，以加入 Docker 集群：

```
Machine2 ~$  docker swarm join --token SWMTKN-1-57n212qtvfnpu0ab28tewiorf3j9fxzo9vaa7drpare0ic6ohg-5epus8clyzd9xq7e7ze1y0p0n 192.168.122.185:2377
```

`Machine2` 应成功加入 Docker 集群：

```
This node joined a swarm as a worker.
```

1.  在两个节点上执行 `docker info` 命令，以确保它们已成功加入集群：

`Machine1`：

```
Machine1 ~$ docker info
```

`Machine2`：

```
Machine2 ~$ docker info
```

以下输出是 `docker info` 输出的 `swarm` 部分的截断。从这些细节中，您将看到这些 Docker 节点配置在一个集群中，并且集群中有两个节点，一个是单个管理节点（`Machine1`）。这些参数在两个节点上应该是相同的，除了 `Is Manager` 参数，其中 `Machine1` 将是管理节点。默认情况下，Docker 将为默认的 Docker 集群 `overlay` 网络分配一个默认子网 `10.0.0.0/8`：

```
 swarm: active
  NodeID: oub9g5383ifyg7i52yq4zsu5a
  Is Manager: true
  ClusterID: x7chp0w3two04ltmkqjm32g1f
  Managers: 1
  Nodes: 2
  Default Address Pool: 10.0.0.0/8  
  SubnetSize: 24
  Data Path Port: 4789
  Orchestration:
    Task History Retention Limit: 5
```

1.  从 `Machine1` 中，使用 `docker network create` 命令创建一个 `overlay` 网络。由于这是一个将跨越简单集群中的多个节点的网络，因此需要将 `overlay` 驱动程序指定为网络驱动程序。将此网络命名为 `overlaynet1`。使用尚未被 Docker 主机上的任何网络使用的子网和网关，以避免子网冲突。使用 `172.45.0.0/16` 和 `172.45.0.1` 作为网关：

```
Machine1 ~$ docker network create overlaynet1 --driver overlay --subnet 172.45.0.0/16 --gateway 172.45.0.1
```

将创建 `overlay` 网络。

1.  使用 `docker network ls` 命令来验证网络是否成功创建并且是否使用了正确的 `overlay` 驱动程序：

```
Machine1 ~$ docker network ls
```

将显示 Docker 主机上可用的网络列表：

```
NETWORK ID       NAME              DRIVER     SCOPE
54f2af38e6a8     bridge            bridge     local
df5ebd75303e     docker_gwbridge   bridge     local
f52b4a5440ad     host              host       local
8hm1ouvt4z7t     ingress           overlay    swarm
9bed60b88784     none              null       local
60wqq8ewt8zq     overlaynet1       overlay    swarm
```

1.  使用`docker service create`命令创建一个将跨多个节点的 swarm 集群的服务。将容器部署为服务允许您指定一个容器实例的多个副本，以进行水平扩展或在集群中的节点之间扩展容器实例以实现高可用性。为了保持这个例子简单，创建一个 Alpine Linux 的单个容器服务。将此服务命名为`alpine-overlay1`：

```
Machine1 ~$ docker service create -t --replicas 1 --network overlaynet1 --name alpine-overlay1 alpine:latest
```

一个基于文本的进度条将显示`alpine-overlay1`服务部署的进度：

```
overall progress: 1 out of 1 tasks 
1/1: running   [===========================================>]
verify: Service converged 
```

1.  重复相同的`docker service create`命令，但现在将`alpine-overlay2`指定为服务名称：

```
Machine1 ~$ docker service create -t --replicas 1 --network overlaynet1 --name alpine-overlay2 alpine:latest
```

一个基于文本的进度条将再次显示服务部署的进度：

```
overall progress: 1 out of 1 tasks 
1/1: running   [===========================================>]
verify: Service converged
```

注意

有关在 Docker swarm 中创建服务的更多详细信息，请参阅《第九章，Docker Swarm》。由于本练习的范围是网络，我们现在将专注于网络组件。

1.  从`Machine1`节点，执行`docker ps`命令以查看此节点上正在运行的服务：

```
Machine1 ~$ docker ps 
```

正在运行的容器将被显示。Docker 将在 Docker swarm 集群中的节点之间智能地扩展容器。在本例中，`alpine-overlay1`服务的容器落在了`Machine1`上。根据 Docker 部署服务的方式，您的环境可能会有所不同：

```
CONTAINER ID    IMAGE           COMMAND     CREATED
  STATUS              PORTS             NAMES
4d0f5fa82add    alpine:latest   "/bin/sh"   59 seconds ago
  Up 57 seconds                         alpine-overlay1.1.
r0tlm8w0dtdfbjaqyhobza94p
```

1.  运行`docker inspect`命令以查看正在运行的容器的详细信息：

```
Machine1 ~$ docker inspect alpine-overlay1.1.r0tlm8w0dtdfbjaqyhobza94p
```

将显示正在运行的容器实例的详细信息。以下输出已被截断以显示`docker inspect`输出的`NetworkSettings`部分：

![图 6.24：检查 alpine-overlay1 容器实例](img/B15021_06_24.jpg)

图 6.24：检查 alpine-overlay1 容器实例

注意，此容器的 IP 地址与您在`Machine1`上指定的子网中的预期值相同。

1.  在`Machine2`实例上，执行`docker network ls`命令以查看主机上可用的 Docker 网络：

```
Machine2 ~$ docker network ls
```

将显示 Docker 主机上所有可用的 Docker 网络的列表：

```
NETWORK ID       NAME              DRIVER     SCOPE
8c7755be162f     bridge            bridge     local
28055e8c63a0     docker_gwbridge   bridge     local
c62fb7ac090f     host              host       local
8hm1ouvt4z7t     ingress           overlay    swarm
6182d77a8f62     none              null       local
60wqq8ewt8zq     overlaynet1       overlay    swarm
```

注意，`Machine1`上定义的`overlaynet1`网络也可在`Machine2`上使用。这是因为使用`overlay`驱动程序创建的网络可用于 Docker swarm 集群中的所有主机。这使得可以使用此网络部署容器以在集群中的所有主机上运行。

1.  使用`docker ps`命令列出此 Docker 实例上正在运行的容器：

```
Machine2 ~$ docker ps
```

将所有正在运行的容器列出。在这个例子中，`alpine-overlay2` 服务中的容器落在了 `Machine2` 集群节点上：

```
CONTAINER ID   IMAGE           COMMAND      CREATED
  STATUS              PORTS               NAMES
53747ca9af09   alpine:latest   "/bin/sh"    33 minutes ago
  Up 33 minutes                           alpine-overlay2.1.ui9vh6zn18i48sxjbr8k23t71
```

注意

在您的示例中，服务落在哪个节点可能与此处显示的不同。Docker 根据各种标准（如可用的 CPU 带宽、内存和对部署容器的调度限制）来决定如何部署容器。

1.  使用 `docker inspect` 来调查该容器的网络配置：

```
Machine2 ~$ docker inspect alpine-overlay2.1.ui9vh6zn18i48sxjbr8k23t71
```

将显示详细的容器配置。此输出已被截断，以 JSON 格式显示输出的 `NetworkSettings` 部分：

![图 6.25：alpine-overlay2 容器实例的 docker inspect 输出](img/B15021_06_25.jpg)

图 6.25：alpine-overlay2 容器实例的 docker inspect 输出

请注意，该容器还在 `overlaynet1` `overlay` 网络中拥有一个 IP 地址。

1.  由于两个服务都部署在同一个 `overlay` 网络中，但存在于两个独立的主机中，您可以看到 Docker 正在使用 `underlay` 网络来代理 `overlay` 网络的流量。通过尝试从一个服务到另一个服务的 ping 来检查服务之间的网络连接。需要注意的是，类似于部署在同一网络中的静态容器，部署在同一网络上的服务可以使用 Docker DNS 通过名称解析彼此。在 `Machine2` 主机上使用 `docker exec` 命令访问 `alpine-overlay2` 容器内的 `sh` shell：

```
Machine2 ~$ docker exec -it alpine-overlay2.1.ui9vh6zn18i48sxjbr8k23t71 /bin/sh
```

这应该将您放入 `alpine-overlay2` 容器实例的 root shell。使用 `ping` 命令发起与 `alpine-overlay1` 容器的网络通信：

```
/ # ping alpine-overlay1
PING alpine-overlay1 (172.45.0.10): 56 data bytes
64 bytes from 172.45.0.10: seq=0 ttl=64 time=0.314 ms
64 bytes from 172.45.0.10: seq=1 ttl=64 time=0.274 ms
64 bytes from 172.45.0.10: seq=2 ttl=64 time=0.138 ms
```

请注意，即使这些容器部署在两个独立的主机上，它们也可以通过名称使用共享的 `overlay` 网络进行通信。

1.  从 `Machine1` 主机，您可以尝试与 `alpine-overlay2` 服务容器进行相同的通信。使用 `docker exec` 命令在 `Machine1` 主机上访问 `alpine-overlay2` 容器内的 `sh` shell：

```
Machine1 ~$ docker exec -it alpine-overlay1.1.r0tlm8w0dtdfbjaqyhobza94p /bin/sh
```

这应该将您放入容器内的 root shell。使用 `ping` 命令发起与 `alpine-overlay2` 容器实例的网络通信：

```
/ # ping alpine-overlay2
PING alpine-overlay2 (172.45.0.13): 56 data bytes
64 bytes from 172.45.0.13: seq=0 ttl=64 time=0.441 ms
64 bytes from 172.45.0.13: seq=1 ttl=64 time=0.227 ms
64 bytes from 172.45.0.13: seq=2 ttl=64 time=0.282 ms
```

再次注意，通过使用 Docker DNS，可以使用 `overlay` 网络驱动程序在主机之间解析 `alpine-overlay2` 容器的 IP 地址。

1.  使用 `docker service rm` 命令从 `Machine1` 节点中删除这两个服务：

```
Machine1 ~$ docker service rm alpine-overlay1
Machine1 ~$ docker service rm alpine-overlay2
```

对于这些命令中的每一个，服务名称将会短暂地显示，表明命令执行成功。在两个节点上，`docker ps` 将显示当前没有正在运行的容器。

1.  使用 `docker rm` 命令并指定名称 `overlaynet1` 删除 `overlaynet1` Docker 网络。

```
Machine1 ~$ docker network rm overlaynet1
```

`overlaynet1` 网络将被删除。

在这个练习中，我们研究了 Docker 集群中两个主机之间的 `overlay` 网络。`Overlay` 网络在 Docker 容器集群中非常有益，因为它允许在集群中的节点之间水平扩展容器。从网络的角度来看，这些容器可以通过使用服务网格在主机机器的物理网络接口上直接进行通信。这不仅减少了延迟，还通过利用 Docker 的许多功能（如 DNS）简化了部署。

现在我们已经看过了所有本地 Docker 网络类型以及它们的功能示例，我们可以看看 Docker 网络的另一个方面，这个方面最近变得越来越受欢迎。由于 Docker 网络非常模块化，正如我们所见，Docker 支持插件系统，允许用户部署和管理自定义网络驱动程序。

在下一节中，我们将通过从 Docker Hub 安装第三方网络驱动程序来了解非本地 Docker 网络是如何工作的。

# 非本地 Docker 网络

在本章的最后一节中，我们将讨论非本地 Docker 网络。除了可用的本地 Docker 网络驱动程序之外，Docker 还支持用户编写或通过 Docker Hub 从第三方下载的自定义网络驱动程序。自定义的第三方网络驱动程序在需要非常特定的网络配置或容器网络需要以特定方式运行的情况下非常有用。例如，一些网络驱动程序提供了用户设置自定义策略以控制对互联网资源的访问，或者定义容器化应用之间通信的白名单的能力。从安全、策略和审计的角度来看，这是有帮助的。

在接下来的练习中，我们将下载并安装 Weave Net 驱动程序，并在 Docker 主机上创建一个网络。Weave Net 是一个得到高度支持的第三方网络驱动程序，可以很好地查看容器网格网络，允许用户创建可以跨多云场景的复杂服务网格基础设施。我们将从 Docker Hub 安装 Weave Net 驱动程序，并在前面练习中定义的简单集群中配置一个基本网络。

## 练习 6.05：安装和配置 Weave Net Docker 网络驱动程序

在这个练习中，您将下载并安装 Weave Net Docker 网络驱动程序，并在之前创建的 Docker 集群中部署它。Weave Net 是最常见和灵活的第三方 Docker 网络驱动程序之一。使用 Weave Net，可以定义非常复杂的网络配置，以实现基础设施的最大灵活性：

1.  在`Machine1`节点上使用`docker plugin install`命令从 Docker Hub 安装 Weave Net 驱动程序：

```
Machine1 ~$ docker plugin install store/weaveworks/net-plugin:2.5.2
```

这将提示您在安装它的机器上授予 Weave Net 权限。授予请求的权限是安全的，因为 Weave Net 需要这些权限才能在主机操作系统上正确设置网络驱动程序：

```
Plugin "store/weaveworks/net-plugin:2.5.2" is requesting 
the following privileges:
 - network: [host]
 - mount: [/proc/]
 - mount: [/var/run/docker.sock]
 - mount: [/var/lib/]
 - mount: [/etc/]
 - mount: [/lib/modules/]
 - capabilities: [CAP_SYS_ADMIN CAP_NET_ADMIN CAP_SYS_MODULE]
Do you grant the above permissions? [y/N]
```

通过按下*y*键来回答提示。Weave Net 插件应该安装成功。

1.  在`Machine2`节点上运行相同的`docker plugin install`命令。Docker 集群中的所有节点都应该安装了插件，因为所有节点都将参与到集群网格网络中：

```
Machine2 ~$ docker plugin install store/weaveworks/net-plugin:2.5.2
```

权限提示将被显示。在提示继续安装时回答*y*：

```
Plugin "store/weaveworks/net-plugin:2.5.2" is requesting 
the following privileges:
 - network: [host]
 - mount: [/proc/]
 - mount: [/var/run/docker.sock]
 - mount: [/var/lib/]
 - mount: [/etc/]
 - mount: [/lib/modules/]
 - capabilities: [CAP_SYS_ADMIN CAP_NET_ADMIN CAP_SYS_MODULE]
Do you grant the above permissions? [y/N]
```

1.  在`Machine1`节点上使用`docker network create`命令创建一个网络。将 Weave Net 驱动程序指定为主驱动程序，网络名称为`weavenet1`。对于子网和网关参数，请使用之前练习中尚未使用的唯一子网：

```
Machine1 ~$  docker network create --driver=store/weaveworks/net-plugin:2.5.2 --subnet 10.1.1.0/24 --gateway 10.1.1.1 weavenet1
```

这应该在 Docker 集群中创建一个名为`weavenet1`的网络。

1.  使用`docker network ls`命令列出 Docker 集群中可用的网络：

```
Machine1 ~$ docker network ls 
```

`weavenet1`网络应该显示在列表中：

```
NETWORK ID     NAME             DRIVER
  SCOPE
b3f000eb4699   bridge           bridge
  local
df5ebd75303e   docker_gwbridge  bridge
  local
f52b4a5440ad   host             host
  local
8hm1ouvt4z7t   ingress          overlay
  swarm
9bed60b88784   none             null
  local
q354wyn6yvh4   weavenet1        store/weaveworks/net-plugin:2.5.2
  swarm
```

1.  在`Machine2`节点上执行`docker network ls`命令，以确保`weavenet1`网络也存在于该机器上：

```
Machine2 ~$ docker network ls 
```

`weavenet1`网络应该被列出：

```
NETWORK ID    NAME              DRIVER
  SCOPE
b3f000eb4699  bridge            bridge
  local
df5ebd75303e  docker_gwbridge   bridge
  local
f52b4a5440ad  host              host
  local
8hm1ouvt4z7t  ingress           overlay
  swarm
9bed60b88784  none              null
  local
q354wyn6yvh4  weavenet1         store/weaveworks/net-plugin:2.5.2
  swarm
```

1.  在`Machine1`节点上，使用`docker service create`命令创建一个名为`alpine-weavenet1`的服务，该服务使用`weavenet1`网络：

```
Machine1 ~$ docker service create -t --replicas 1 --network weavenet1 --name alpine-weavenet1 alpine:latest
```

文本进度条将显示服务的部署状态。它应该在没有任何问题的情况下完成：

```
overall progress: 1 out of 1 tasks 
1/1: running   [===========================================>]
verify: Service converged 
```

1.  再次使用`docker service create`命令在`weavenet1`网络中创建另一个名为`alpine-weavenet2`的服务：

```
Machine1 ~$ docker service create -t --replicas 1 --network weavenet1 --name alpine-weavenet2 alpine:latest
```

文本进度条将再次显示，指示服务创建的状态：

```
overall progress: 1 out of 1 tasks 
1/1: running   [===========================================>]
verify: Service converged 
```

1.  运行`docker ps`命令验证集群中每个节点上是否成功运行了 Alpine 容器：

`Machine1`：

```
Machine1 ~$ docker ps
```

`Machine2`：

```
Machine2 ~$ docker ps
```

其中一个服务容器应该在两台机器上都正常运行：

`Machine1`：

```
CONTAINER ID    IMAGE           COMMAND      CREATED
  STATUS              PORTS               NAMES
acc47f58d8b1    alpine:latest   "/bin/sh"    7 minutes ago
  Up 7 minutes                            alpine-weavenet1.1.zo5folr5yvu6v7cwqn23d2h97
```

`Machine2`：

```
CONTAINER ID    IMAGE           COMMAND     CREATED
  STATUS              PORTS        NAMES
da2a45d8c895    alpine:latest   "/bin/sh"   4 minutes ago
  Up 4 minutes                     alpine-weavenet2.1.z8jpiup8yetj
rqca62ub0yz9k
```

1.  使用`docker exec`命令访问`weavenet1.1`容器实例内部的`sh` shell。确保在运行此容器的 swarm 集群中的节点上运行此命令：

```
Machine1 ~$ docker exec -it alpine-weavenet1.1.zo5folr5yvu6v7cwqn23d2h97 /bin/sh
```

这应该将您带入容器内部的 root shell：

```
/ #
```

1.  使用`ifconfig`命令查看此容器内部存在的网络接口：

```
/ # ifconfig
```

这将显示一个名为`ethwe0`的新命名网络接口。Weave Net 核心网络策略的核心部分是在容器内创建自定义命名的接口，以便进行简单识别和故障排除。需要注意的是，此接口被分配了一个 IP 地址，该地址来自我们提供的子网作为配置参数：

```
ethwe0  Link encap:Ethernet  HWaddr AA:11:F2:2B:6D:BA  
        inet addr:10.1.1.3  Bcast:10.1.1.255  Mask:255.255.255.0
        UP BROADCAST RUNNING MULTICAST  MTU:1376  Metric:1
        RX packets:37 errors:0 dropped:0 overruns:0 frame:0
        TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
        collisions:0 txqueuelen:0 
        RX bytes:4067 (3.9 KiB)  TX bytes:0 (0.0 B)
```

1.  从容器内部，使用`ping`实用程序通过名称 ping`alpine-weavenet2`服务：

```
ping alpine-weavenet2
```

您应该看到来自`alpine-weavenet2`服务的解析 IP 地址的响应：

```
64 bytes from 10.1.1.4: seq=0 ttl=64 time=3.430 ms
64 bytes from 10.1.1.4: seq=1 ttl=64 time=1.541 ms
64 bytes from 10.1.1.4: seq=2 ttl=64 time=1.363 ms
64 bytes from 10.1.1.4: seq=3 ttl=64 time=1.850 ms
```

注意

由于 Docker 和 Docker Swarm 最近版本中 Docker libnetwork 堆栈的更新，通过名称 ping 服务：`alpine-weavenet2`可能无法正常工作。为了证明网络按预期工作，请尝试直接 ping 容器的名称而不是服务的名称：`alpine-weavenet2.1.z8jpiup8yetjrqca62ub0yz9k` - 请记住，在您的实验环境中，此容器的名称将不同。

1.  还可以尝试从这些容器中通过开放的互联网 ping Google DNS 服务器（`8.8.8.8`）以确保这些容器具有互联网访问权限：

```
ping 8.8.8.8
```

您应该看到返回的响应，表明这些容器具有互联网访问权限：

```
/ # ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=51 time=13.224 ms
64 bytes from 8.8.8.8: seq=1 ttl=51 time=11.840 ms
type exit to quit the shell session in this container.
```

1.  使用`docker service rm`命令从`Machine1`节点中删除两个服务：

```
Machine1 ~$ docker service rm alpine-weavenet1
Machine1 ~$ docker service rm alpine-weavenet2
```

这将删除两个服务，停止并删除容器实例。

1.  通过运行以下命令删除 Weave Net 网络：

```
Machine1 ~$ docker network rm weavenet1
```

应删除和移除 Weave Net 网络。

在容器化网络概念的强大系统中，Docker 拥有广泛的网络驱动程序，几乎可以满足工作负载需求的任何情况。然而，对于所有超出默认 Docker 网络驱动程序范围的用例，Docker 支持第三方自定义驱动程序，几乎可以满足可能出现的任何网络条件。第三方网络驱动程序使 Docker 能够灵活地与各种平台甚至跨多个云提供商进行集成。在这个练习中，我们看了安装和配置 Weave Net 网络插件，并在 Docker 集群中创建简单服务以利用这个网络。

在接下来的活动中，您将应用本章学到的知识，使用各种 Docker 网络驱动程序部署多容器基础架构解决方案。这些容器将使用不同的 Docker 网络驱动程序在同一台主机甚至跨多台主机在 Docker 集群配置中进行通信。

## 活动 6.01：利用 Docker 网络驱动程序

在本章的前面，我们看了各种类型的 Docker 网络驱动程序以及它们以不同的方式运行，为您的容器环境提供各种程度的网络功能。在这个活动中，您将在 Docker“桥接”网络中部署 Panoramic Trekking 应用程序的示例容器。然后，您将以“主机”网络模式部署一个辅助容器，该容器将作为监控服务器，并能够使用`curl`来验证应用程序是否按预期运行。

执行以下步骤完成此活动：

1.  创建一个自定义的 Docker“桥接”网络，具有自定义子网和网关 IP。

1.  在“桥接”网络中部署一个名为`webserver1`的 NGINX web 服务器，将容器上的转发端口`80`暴露到主机上的端口`8080`。

1.  在“主机”网络模式下部署一个 Alpine Linux 容器，该容器将作为监控容器。

1.  使用 Alpine Linux 容器对 NGINX web 服务器进行`curl`操作并获得响应。

**预期输出：**

在活动完成后连接到转发端口`8080`和`webserver1`容器的 IP 地址上的端口`80`，您应该会得到以下输出：

![图 6.26：从容器实例的 IP 地址访问 NGINX web 服务器](img/B15021_06_26.jpg)

图 6.26：从容器实例的 IP 地址访问 NGINX Web 服务器

注意

本活动的解决方案可以通过此链接找到。

在下一个活动中，我们将看看如何利用 Docker `overlay`网络为我们的全景徒步应用程序提供水平可扩展性。通过在多个主机上部署全景徒步，我们可以确保可靠性和耐久性，并利用环境中多个节点的系统资源。

## 活动 6.02：`overlay`网络实战

在本章中，您已经看到了在集群主机之间部署多个容器时，`overlay`网络是多么强大，它们之间可以直接进行网络连接。在本活动中，您将重新访问双节点 Docker swarm 集群，并从全景徒步应用程序创建服务，这些服务将使用 Docker DNS 在两个主机之间进行连接。在这种情况下，不同的微服务将在不同的 Docker swarm 主机上运行，但仍然能够利用 Docker `overlay`网络直接相互通信。

要成功完成此活动，请执行以下步骤：

1.  使用自定义子网和网关的 Docker `overlay`网络

1.  一个名为`trekking-app`的应用程序 Docker swarm 服务，使用 Alpine Linux 容器

1.  一个名为`database-app`的数据库 Docker swarm 服务，使用 PostgreSQL 12 容器（额外学分提供默认凭据）

1.  证明`trekking-app`服务可以使用`overlay`网络与`database-app`服务通信

**预期输出:**

`trekking-app`服务应能够与`database-app`服务通信，可以通过 ICMP 回复进行验证，例如：

```
PING database-app (10.2.0.5): 56 data bytes
64 bytes from 10.2.0.5: seq=0 ttl=64 time=0.261 ms
64 bytes from 10.2.0.5: seq=1 ttl=64 time=0.352 ms
64 bytes from 10.2.0.5: seq=2 ttl=64 time=0.198 ms
```

注意

本活动的解决方案可以通过此链接找到。

# 摘要

在本章中，我们探讨了与微服务和 Docker 容器相关的许多网络方面。Docker 配备了许多驱动程序和配置选项，用户可以使用这些选项来调整他们的容器网络在几乎任何环境中的工作方式。通过部署正确的网络和正确的驱动程序，可以快速地建立强大的服务网格网络，实现容器之间的访问，而无需从任何物理 Docker 主机出口。甚至可以创建绑定到主机网络结构以利用底层网络基础设施的容器。

在 Docker 中可以启用的最强大的网络功能可能是能够在 Docker 主机集群之间创建网络的能力。这可以让我们快速地在主机之间创建和部署水平扩展的应用程序，以实现高可用性和冗余。通过利用底层网络，在集群中的“覆盖”网络允许容器利用强大的 Docker DNS 系统直接联系其他集群主机上运行的容器。

在下一章中，我们将看看强大的容器化基础设施的下一个支柱：存储。通过了解容器存储如何用于有状态的应用程序，可以设计出非常强大的解决方案，不仅涉及容器化的无状态应用程序，还涉及可以像其他容器一样轻松部署、扩展和优化的容器化数据库服务。

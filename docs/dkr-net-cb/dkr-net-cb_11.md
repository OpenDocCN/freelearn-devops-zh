# 第十一章：故障排除 Docker 网络

在本章中，我们将涵盖以下示例：

+   使用 tcpdump 验证网络路径

+   验证 VETH 对

+   验证发布的端口和出站伪装

+   验证名称解析

+   构建一个测试容器

+   重置本地 Docker 网络数据库

# 介绍

正如我们在前几章中看到的，Docker 利用了一系列相对知名的 Linux 网络构造来提供容器网络。在本书中，我们已经看过许多不同的方式，您可以配置、使用和验证 Docker 网络配置。我们还没有概述当您遇到问题时可以使用的故障排除和验证方法。在故障排除容器网络时，重要的是要理解并能够排除用于提供端到端连接的每个特定网络组件。本章的目标是提供在需要验证或故障排除 Docker 网络问题时可以采取的具体步骤。

# 使用 tcpdump 验证网络路径

尽管我们在之前的章节中简要介绍了它的用法，但任何在基于 Linux 的系统上使用网络的人都应该熟悉`tcpdump`。`tcpdump`允许您在主机上的一个或多个接口上捕获网络流量。在这个示例中，我们将介绍如何使用`tcpdump`来验证几种不同的 Docker 网络场景中的容器网络流量。

## 准备工作

在这个示例中，我们将使用一个单独的 Docker 主机。假设 Docker 已安装并处于默认配置。您还需要 root 级别的访问权限，以便检查和更改主机的网络和防火墙配置。您还需要安装`tcpdump`实用程序。如果您的系统上没有它，您可以使用以下命令安装它：

```
sudo apt-get install tcpdump
```

## 如何做…

`tcpdump`是一个令人惊叹的故障排除工具。当正确使用时，它可以让您详细查看 Linux 主机上接口上的数据包。为了演示，让我们在我们的 Docker 主机上启动一个单个容器：

```
user@docker1:~$ docker run -dP --name web1 jonlangemak/web_server_1
ea32565ece0c0c22eace935113b6697bebe837f0b5ddf31724f371220792fb15
user@docker1:~$
```

由于我们没有指定任何网络参数，这个容器将在`docker0`桥上运行，并且任何暴露的端口都将发布到主机接口上。从容器生成的流量也将隐藏在主机的 IP 接口后，因为流量朝向外部网络。使用`tcpdump`，我们可以在每个阶段看到这个流量。

让我们首先检查进入主机的流量：

```
user@docker1:~$ docker port web1
80/tcp -> 0.0.0.0:32768
user@docker1:~$
```

在我们的案例中，这个容器暴露了端口`80`，现在已经发布到主机接口的端口`32768`上。让我们首先确保流量进入主机的正确端口。为了做到这一点，我们可以在主机的`eth0`接口上捕获到目标端口为`32768`的流量：

```
user@docker1:~$ **sudo tcpdump -qnn -i eth0 dst port 32768**
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
**15:46:07.629747 IP 10.20.30.41.55939 > 10.10.10.101.32768: tcp 0**
**15:46:07.629997 IP 10.20.30.41.55940 > 10.10.10.101.32768: tcp 0**
**15:46:07.630257 IP 10.20.30.41.55939 > 10.10.10.101.32768: tcp 0**

```

要使用`tcpdump`捕获这个入站流量，我们使用了一些不同的参数：

+   `q`：这告诉`tcpdump`保持安静，或者不要生成太多输出。因为我们只想看到第 3 层和第 4 层的信息，这样可以清理输出得很好

+   `nn`：这告诉`tcpdump`不要尝试将 IP 解析为 DNS 名称。同样，我们想在这里看到 IP 地址

+   `i`：这指定了我们要捕获的接口，在这种情况下是`eth0`

+   `src port`：告诉`tcpdump`过滤具有目的端口为`32768`的流量

### 注意

`dst`参数可以从此命令中删除。这样做将过滤任何端口为`32768`的流量，从而显示整个流量，包括返回流量。

如前面的代码所示，我们可以看到主机在其物理接口（`10.10.10.101`）上接收到来自远程源（`10.20.30.41`）的端口`32768`的流量。在这种情况下，`10.20.30.41`是一个测试服务器，它正在向容器的发布端口发出流量。

既然我们已经看到流量到达主机，让我们看看它是如何穿过`docker0`桥的：

```
user@docker1:~$ **sudo tcpdump -qnn -i docker0**
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on docker0, link-type EN10MB (Ethernet), capture size 65535 bytes
**16:34:54.193822 IP 10.20.30.41.53846 > 172.17.0.2.80: tcp 0**
**16:34:54.193848 IP 10.20.30.41.53847 > 172.17.0.2.80: tcp 0**
**16:34:54.193913 IP 172.17.0.2.80 > 10.20.30.41.53846: tcp 0**
**16:34:54.193940 IP 172.17.0.2.80 > 10.20.30.41.53847: tcp 0**

```

在这种情况下，我们可以通过只在`docker0`桥接口上过滤流量来看到流量。正如预期的那样，我们看到相同的流量，具有相同的源，但现在反映了容器中运行的服务的准确目的地 IP 和端口，这要归功于发布端口功能。

虽然这当然是捕获流量的最简单方法，但如果您在`docker0`桥上运行多个容器，这种方法并不是非常有效。当前的过滤器将为您提供桥上所有的流量，而不仅仅是您正在寻找的特定容器。在这种情况下，您还可以在过滤器中指定 IP 地址，就像这样：

```
user@docker1:~$ **sudo tcpdump -qnn -i docker0 dst 172.17.0.2**
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on docker0, link-type EN10MB (Ethernet), capture size 65535 bytes
**16:42:22.332555 IP 10.20.30.41.53878 > 172.17.0.2.80: tcp 0**
**16:42:22.332940 IP 10.20.30.41.53878 > 172.17.0.2.80: tcp 0**

```

### 注意

我们在这里将目的地 IP 指定为过滤器。如果我们希望看到源和目的地都是该 IP 地址的流量，我们可以用`host`替换`dst`。

这种数据包捕获对于验证端口发布等功能是否按预期工作至关重要。捕获可以在大多数接口类型上进行，包括那些没有与其关联的 IP 地址的接口。这种接口的一个很好的例子是用于将容器命名空间连接回默认命名空间的 VETH 对的主机端。在排除容器连接问题时，能够将到达`docker0`桥的流量与特定主机端 VETH 接口相关联可能会很方便。我们可以通过从多个地方相关数据来实现这一点。例如，假设我们执行以下`tcpdump`：

```
user@docker1:~$ **sudo tcpdump -qnne -i docker0 host 172.17.0.2**
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on docker0, link-type EN10MB (Ethernet), capture size 65535 bytes
16:59:33.334941 **02:42:ab:27:0e:3e** > **02:42:ac:11:00:02**, IPv4, length 66: 10.20.30.41.57260 > 172.17.0.2.80: tcp 0
16:59:33.335012 **02:42:ac:11:00:02** > **02:42:ab:27:0e:3e**, IPv4, length 66: 172.17.0.2.80 > 10.20.30.41.57260: tcp 0
```

请注意，在这种情况下，我们向`tcpdump`传递了`e`参数。这告诉`tcpdump`显示每个帧的源和目的 MAC 地址。在这种情况下，我们可以看到我们有两个 MAC 地址。其中一个将是与`docker0`桥相关联的 MAC 地址，另一个将是与容器相关联的 MAC 地址。我们可以查看`docker0`桥信息来确定其 MAC 地址是什么：

```
user@docker1:~$ ip link show dev docker0
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether **02:42:ab:27:0e:3e** brd ff:ff:ff:ff:ff:ff
user@docker1:~$
```

这将留下地址`02:42:ac:11:00:02`。使用作为`iproute2`工具集的一部分的 bridge 命令，我们可以确定这个 MAC 地址存在于哪个接口上：

```
user@docker1:~$ bridge fdb show | grep **02:42:ac:11:00:02**
**02:42:ac:11:00:02 dev vetha431055**
user@docker1:~$
```

在这里，我们可以看到容器的 MAC 地址可以通过名为`vetha431055`的接口访问。在该接口上进行捕获将确认我们是否正在查看正确的接口：

```
user@docker1:~$ **sudo tcpdump -qnn -i vetha431055**
tcpdump: WARNING: vetha431055: no IPv4 address assigned
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on vetha431055, link-type EN10MB (Ethernet), capture size 65535 bytes
21:01:24.503939 IP 10.20.30.41.58035 > **172.17.0.2.80**: tcp 0
21:01:24.503990 IP **172.17.0.2.80** > 10.20.30.41.58035: tcp 0
```

`tcpdump`可以成为验证容器通信的重要工具。花一些时间了解该工具以及使用其不同参数过滤流量的不同方式是明智的。

# 验证 VETH 对

在本书中我们审查过的所有 Linux 网络构造中，VETH 对可能是最重要的。它们是命名空间感知的，允许您将一个唯一命名空间中的容器连接到包括默认命名空间在内的任何其他命名空间。虽然 Docker 会为您处理所有这些，但能够确定 VETH 对的端点位于何处并将它们相关联以确定 VETH 对的用途是很有用的。在本教程中，我们将深入研究如何找到和相关 VETH 对的端点。

## 准备工作

在本教程中，我们将使用单个 Docker 主机。假定 Docker 已安装并处于默认配置。您还需要 root 级别访问权限，以便检查和更改主机的网络和防火墙配置。

## 如何做…

Docker 中 VETH 对的主要用例是将容器的网络命名空间连接回默认网络命名空间。它通过将 VETH 对中的一个放置在`docker0`桥上，另一个放置在容器中来实现这一点。VETH 对的容器端被分配了一个 IP 地址，然后重命名为`eth0`。

当寻找匹配容器的 VETH 对端时，有两种情况。第一种是当你从默认命名空间开始，第二种是当你从容器命名空间开始。让我们逐步讨论每种情况以及如何将它们联系在一起。

让我们首先从了解接口的主机端开始。例如，假设我们正在寻找这个接口的容器端：

```
user@docker1:~$ ip -d link show
…<Additional output removed for brevity>… 
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:42:ab:27:0e:3e brd ff:ff:ff:ff:ff:ff promiscuity 0
    bridge
**6: vetha431055@if5**: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
    link/ether 82:69:cb:b6:9a:db brd ff:ff:ff:ff:ff:ff promiscuity 1
    **veth**
user@docker1:~$
```

这里有几件事情需要指出。首先，将`-d`参数传递给`ip link`子命令会显示有关接口的额外详细信息。在这种情况下，它确认了接口是一个 VETH 对。其次，VETH 对的命名通常遵循`<end1>@<end2>`的命名约定。在这种情况下，我们可以看到`vetha431055`端是本地接口，而`if5`是另一端。`if5`代表接口 5 或主机上第 5 个接口的索引 ID。由于 VETH 接口总是成对创建的，可以合理地假设具有索引 6 的 VETH 对端很可能是索引 5 或 7。在这种情况下，命名表明它是 5，但我们可以使用`ethtool`命令来确认：

```
user@docker1:~$ sudo ethtool -S **vetha431055**
NIC statistics:
 **peer_ifindex: 5**
user@docker1:~$
```

正如你所看到的，这个 VETH 对的另一端具有接口索引 5，正如名称所示。现在找到具有 5 的容器是困难的部分。为了做到这一点，我们需要检查每个容器的特定接口号。如果你运行了很多容器，这可能是一个挑战。你可以使用 Linux 的`xargs`循环遍历它们，而不是手动检查每个容器。例如，看看这个命令：

```
docker ps -q | xargs --verb -I {} docker exec {} ip link | grep ⁵:
```

我们在这里要做的是返回所有正在运行的容器的容器 ID 列表，然后将该列表传递给`xargs`。反过来，`xargs`正在使用这些容器 ID 在容器内部运行`docker exec`命令。该命令恰好是`ip link`命令，它将返回所有接口及其关联的索引号的列表。如果返回的任何信息以`5:`开头，表示接口索引为 5，我们将把它打印到屏幕上。为了查看哪个容器具有相关接口，我们必须以详细模式（`--verb`）运行`xargs`命令，这将显示每个命令的运行情况。输出将如下所示：

```
user@docker1:~$ **docker ps -q | xargs --verb -I {} docker exec {} ip link | grep ⁵:**
docker exec 4b521df22184 ip link
docker exec 772e12b15c92 ip link
docker exec d8f3e7936690 ip link
docker exec a2e3201278e2 ip link
docker exec f9216233ba56 ip link
**docker exec ea32565ece0c ip link**
**5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP**
user@docker1:~$
```

如您所见，此主机上有六个容器正在运行。直到最后一个容器，我们才找到了我们要找的接口 ID。有了容器 ID，我们就可以知道哪个容器具有 VETH 接口的另一端。

### 注意

您可以通过运行`docker exec -it` `ea32565ece0c ip link`命令来确认这一点。

现在，让我们尝试另一个例子，从 VETH 对的容器端开始。这稍微容易一些，因为接口的命名告诉我们主机端匹配接口的索引：

```
user@docker1:~$ docker exec web1 ip -d link show dev eth0
5: **eth0@if6**: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    veth
user@docker1:~$
```

然后，我们可以通过再次使用`ethtool`来验证主机上索引为 6 的接口是否与容器中索引为 5 的接口匹配：

```
user@docker1:~$ ip -d link show | grep ^**6:**
**6: vetha431055**@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
user@docker1:~$ sudo ethtool -S **vetha431055**
[sudo] password for user:
NIC statistics:
 **peer_ifindex: 5**
user@docker1:~$
```

# 验证已发布的端口和出站伪装

Docker 网络中涉及的较困难的部分之一是`iptables`。`iptables`/netfilter 集成在提供端口发布和出站伪装等功能方面发挥着关键作用。然而，如果您对其不熟悉，`iptables`可能很难理解和排除故障。在本教程中，我们将审查如何详细检查`iptables`配置，并验证连接是否按预期工作。

## 准备工作

在本教程中，我们将使用单个 Docker 主机。假设 Docker 已安装并处于默认配置中。您还需要 root 级别的访问权限才能检查`iptables`规则集。

## 如何做…

正如我们在前几章中看到的，Docker 在代表您管理主机防火墙规则方面做得非常出色。您可能很少需要查看或修改与 Docker 相关的`iptables`规则。然而，能够验证配置以排除`iptables`可能是容器网络故障的一个好主意。

为了演示遍历`iptables`规则集，我们将检查一个发布端口的示例容器。我们执行这些步骤很容易转移到检查任何其他 Docker 集成`iptables`用例的规则。为此，我们将运行一个简单的容器，该容器公开端口`80`以进行发布：

```
user@docker1:~$ docker run -dP --name web1 jonlangemak/web_server_1
```

由于我们告诉 Docker 发布任何公开的端口，我们知道该容器应该将其公开的端口`80`发布到主机。为了验证端口是否真的被发布，我们可以检查`iptables`规则集。我们想要做的第一件事是确保端口发布所需的目标 NAT 已经就位。为了检查`iptables`表，我们可以使用`iptables`命令并传递以下参数：

+   `n`：告诉`iptables`在输出中使用数值信息，如地址和端口

+   `L`：告诉`iptables`你想输出一个规则列表

+   `v`：告诉`iptables`提供详细输出，这样我们就可以看到所有的规则信息以及规则计数器

+   `t`：告诉`iptables`仅显示特定表的信息

将所有这些放在一起，我们可以使用命令`sudo iptables -nL -t nat`来查看主机 NAT 表中的规则：

![操作步骤...](img/B05453_11_01.jpg)

### 注意

请注意，我们将在本教程中检查的所有默认表和链策略都是“接受”。如果默认链策略是“接受”，这意味着即使我们没有匹配规则，流量仍将被允许。无论默认策略设置为什么，Docker 都将创建规则。

如果你对`iptables`不太熟悉，解释这个输出可能有点令人生畏。即使我们正在查看 NAT 表，我们也需要知道哪个链正在处理进入主机的通信。在我们的情况下，由于流量进入主机，我们感兴趣的链是`PREROUTING`链。让我们来看一下表是如何处理的：

+   `PREROUTING`链中的第一行寻找目的地为`LOCAL`或主机本身的流量。由于流量的目的地是主机接口之一的 IP 地址，我们匹配了这条规则并执行了引用跳转到一个名为`DOCKER`的新链的动作。

+   在`DOCKER`链中，我们命中了第一条规则，该规则正在寻找进入`docker0`桥的流量。由于这个流量并没有进入`docker0`桥，所以规则被跳过，我们继续移动到链中的下一条规则。

+   `DOCKER`链中的第二条规则正在寻找并非进入`docker0`桥且目的端口为 TCP `32768`的流量。我们匹配了这条规则并执行了将目的 NAT 转换为`172.17.0.2`端口`80`的动作。

表中的处理看起来是这样的：

![操作步骤...](img/B05453_11_02.jpg)

在前面的图像中的箭头表示流量在穿过 NAT 表时的流动。在这个例子中，我们只有一个运行在主机上的容器，所以很容易看出哪些规则正在被处理。

### 注意

您可以将这种输出与`watch`命令配合使用，以获得计数器的实时输出，例如：

```
sudo watch --interval 0 iptables -vnL -t nat
```

现在我们已经穿过了 NAT 表，接下来我们需要担心的是过滤表。我们可以以与查看 NAT 表相同的方式查看过滤表：

![操作步骤...](img/B05453_11_03.jpg)

乍一看，我们可以看到这个表的布局与 NAT 表略有不同。例如，这个表中的链与 NAT 表中的不同。在我们的情况下，我们对入站发布端口通信感兴趣的链是 forward 链。这是因为主机正在转发或路由流量到容器。流量将按以下方式穿过这个表：

+   转发链中的第一行直接将流量发送到`DOCKER-ISOLATION`链。

+   在这种情况下，`DOCKER-ISOLATION`链中唯一的规则是将流量发送回来，所以我们继续审查`FORWARD`表中的规则。

+   转发表中的第二条规则表示，如果流量要离开`docker0`桥，则将流量发送到`DOCKER`链。由于我们的目的地（`172.17.0.20`）位于`docker0`桥外，我们匹配了这条规则并跳转到`DOCKER`链。

+   在`DOCKER`链中，我们检查第一条规则，并确定它正在寻找目的地为容器 IP 地址，端口为 TCP `80`的流量，且是从`docker0`桥接口出去而不是进来的。我们匹配到了这条规则，流量被接受了。

表中的处理如下：

![操作步骤...](img/B05453_11_04.jpg)

通过过滤表是发布端口流量必须经过的最后一步，以便到达容器。然而，我们现在只是到达了容器。我们仍然需要考虑从容器返回到与发布端口通信的主机的返回流量。因此，现在，我们需要讨论容器发起的流量如何由`iptables`处理。

我们将遇到的第一个表是出站流量的过滤表。再次，来自容器的流量将使用过滤表的转发链。流程大致如下：

+   转发链中的第一条规则直接将流量发送到`DOCKER-ISOLATION`链。

+   在这种情况下，`DOCKER-ISOLATION`链中唯一的规则是将流量发送回去，所以我们继续查看转发表中的规则。

+   转发表中的第二条规则表示，如果流量是从`docker0`桥接口出去的，就将流量发送到`DOCKER`链。由于我们的流量是进入`docker0`桥接口而不是出去，所以这条规则被跳过，我们继续查看链中的下一条规则。

+   转发表中的第三条规则表示，如果流量是从`docker0`桥接口出去，并且连接状态是`RELATED`或`ESTABLISHED`，则应该接受该流量。这个流量是进入`docker0`桥接口的，所以我们也不会匹配到这条规则。然而，值得指出的是，这条规则用于允许容器发起的流量的返回流量。它只是作为初始出站连接的一部分而没有被命中，因为那代表了一个新的流量。

+   转发表中的第四条规则表示，如果流量经过`docker0`桥接口，但不是从`docker0`桥接口出去，就接受它。因为我们的流量是进入`docker0`桥接口的，所以我们匹配到了这条规则，流量被接受了。

表中的处理如下：

![操作步骤...](img/B05453_11_05.jpg)

我们将命中的下一个表是 NAT 表。这一次，我们要查看`POSTROUTING`链。在这种情况下，我们匹配链的第一条规则，该规则寻找不是从`docker0`桥出去的流量，并且源自`docker0`桥子网（`172.17.0.0/16`）的流量。

![如何操作…](img/B05453_11_06.jpg)

这条规则的操作是`MASQUERADE`，它将根据主机的路由表隐藏流量在主机接口之一后面。

采用相同的方法，您可以轻松验证与 Docker 相关的其他`iptables`流。当然，随着容器数量的增加，这变得更加困难。然而，由于大多数规则是按照每个容器的基础编写的，命中计数器将对每个容器都是唯一的，这使得缩小范围变得更容易。

### 注意

有关`iptables`表和链的处理顺序的更多信息，请查看这个`iptables`网页和相关的流程图[`www.iptables.info/en/structure-of-iptables.html`](http://www.iptables.info/en/structure-of-iptables.html)。

# 验证名称解析

容器的 DNS 解析一直都很简单。容器接收与主机相同的 DNS 配置。然而，随着用户定义网络和嵌入式 DNS 服务器的出现，这现在变得有点棘手。我见过的许多 DNS 问题中的一个常见问题是不理解嵌入式 DNS 服务器的工作原理以及如何验证它是否正常工作。在这个教程中，我们将逐步介绍容器 DNS 配置，以验证它使用哪个 DNS 服务器来解析特定的命名空间。

## 准备工作

在这个教程中，我们将使用一个单独的 Docker 主机。假设 Docker 已安装并处于默认配置。您还需要 root 级别的访问权限，以便检查和更改主机的网络和防火墙配置。

## 如何操作…

没有用户定义网络的情况下，Docker 的标准 DNS 配置是将主机的 DNS 配置简单地复制到容器中。在这些情况下，DNS 解析很简单：

```
user@docker1:~$ docker run -dP --name web1 jonlangemak/web_server_1
e5735b30ce675d40de8c62fffe28e338a14b03560ce29622f0bb46edf639375f
user@docker1:~$
user@docker1:~$ **docker exec web1 more /etc/resolv.conf**
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver **<your local DNS server>**
**search lab.lab**
user@docker1:~$
user@docker1:~$ more **/etc/resolv.conf**
**nameserver <your local DNS server>**
**search lab.lab**
user@docker1:~$
```

在这些情况下，所有的 DNS 请求都会直接发送到定义的 DNS 服务器。这意味着我们的容器可以解析任何我们的主机可以解析的 DNS 记录：

```
user@docker1:~$ docker exec -it web1 **ping docker2.lab.lab** -c 2
PING docker2.lab.lab (10.10.10.102): 48 data bytes
**56 bytes from 10.10.10.102: icmp_seq=0 ttl=63 time=0.471 ms**
**56 bytes from 10.10.10.102: icmp_seq=1 ttl=63 time=0.453 ms**
--- docker2.lab.lab ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.453/0.462/0.471/0.000 ms
user@docker1:~$
```

再加上 Docker 将这些流量伪装成主机本身的 IP 地址，这就成为了一个简单且易于维护的解决方案。

然而，当我们开始使用用户定义的网络时，情况就会变得有些棘手。这是因为用户定义的网络提供了容器名称解析。也就是说，一个容器可以解析另一个容器的名称，而无需使用静态或手动主机文件条目和链接。这是一个很棒的功能，但如果您不了解容器如何接收其 DNS 配置，可能会导致一些混乱。例如，现在让我们创建一个新的用户定义网络：

```
user@docker1:~$ docker network create -d bridge mybridge1
e8afb0e506298e558baf5408053c64c329b8e605d6ad12efbf10e81f538df7b9
user@docker1:~$
```

现在让我们在这个网络上启动一个名为`web2`的新容器：

```
user@docker1:~$ docker run -dP --name web2 --net \
mybridge1 jonlangemak/web_server_2
1b38ad04c3c1be7b0f1af28550bf402dcde1515899234e4b09e482da0a560a0a
user@docker1:~$
```

现在，如果我们将现有的`web1`容器连接到这个桥接器，我们应该会发现`web1`可以通过名称解析容器`web2`：

```
user@docker1:~$ docker network connect mybridge1 web1
user@docker1:~$ docker exec -it web1 ping web2 -c 2
PING web2 (172.18.0.2): 48 data bytes
**56 bytes from 172.18.0.2: icmp_seq=0 ttl=64 time=0.100 ms**
**56 bytes from 172.18.0.2: icmp_seq=1 ttl=64 time=0.086 ms**
--- web2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.086/0.093/0.100/0.000 ms
user@docker1:~$
```

问题在于，为了实现这一点，Docker 必须更改`web1`容器的 DNS 配置。这样做会在容器的 DNS 请求中间注入嵌入式 DNS 服务器。因此，在此之前，当我们直接与主机的 DNS 服务器通信时，现在我们是在与嵌入式 DNS 服务器通信：

```
user@docker1:~$ docker exec -t  web1 more /etc/resolv.conf
search lab.lab
**nameserver 127.0.0.11**
options ndots:0
user@docker1:~$
```

这对于容器的 DNS 解析是必需的，但它有一个有趣的副作用。嵌入式 DNS 服务器会读取主机的`/etc/resolv.conf`文件，并使用该文件中定义的任何名称服务器作为嵌入式 DNS 服务器的转发器。这样做的净效果是，您不会注意到嵌入式 DNS 服务器，因为它仍然将无法回答的请求转发给主机的 DNS 服务器。但是，它只会在这些转发器被定义时进行编程。如果它们不存在或设置为`127.0.0.1`，那么 Docker 会将转发器设置为 Google 的公共 DNS 服务器（`8.8.8.8`和`8.4.4.4`）。

尽管这是有道理的，但在某些罕见情况下，您的本地 DNS 服务器恰好是`127.0.0.1`。例如，您可能在同一台主机上运行某种类型的本地 DNS 解析器，或者使用 DNS 转发应用程序，比如**DNSMasq**。在这些情况下，Docker 将容器的 DNS 请求转发到前面提到的外部 DNS 服务器，而不是本地定义的 DNS 服务器，可能会引起一些复杂情况。换句话说，内部 DNS 区域将不再可解析：

```
user@docker1:~$ docker exec -it web1 ping docker2.lab.lab
ping: unknown host
user@docker1:~$
```

### 注意

这也可能导致一般的解析问题，因为通常会阻止 DNS 流量到外部 DNS 服务器，而是更倾向于强制内部端点使用内部 DNS 服务器。

在这些情景中，有几种方法可以解决这个问题。您可以在容器运行时通过传递 DNS 标志来指定运行容器的特定 DNS 服务器：

```
user@docker1:~$ docker run -dP --name web2 --net mybridge1 \
--dns <your local DNS server> jonlangemak/web_server_2
```

否则，您可以在 Docker 服务级别设置 DNS 服务器，然后嵌入式 DNS 服务器将使用它作为转发器：

```
ExecStart=/usr/bin/dockerd --dns=<your local DNS server>
```

无论哪种情况，如果您遇到容器解析问题，请始终检查并查看容器在其`/etc/resolv.conf`文件中配置了什么。如果是`127.0.0.11`，那表明您正在使用 Docker 嵌入式 DNS 服务器。如果是这样，并且您仍然遇到问题，请确保验证主机 DNS 配置，以确定嵌入式 DNS 服务器正在使用什么作为转发器。如果没有定义或者是`127.0.0.1`，那么请确保告诉 Docker 服务应该将哪个 DNS 服务器传递给容器，可以使用前面定义的两种方式之一。

# 构建一个测试容器

构建 Docker 容器的原则之一是保持它们小巧精悍。在某些情况下，这可能会限制您的故障排除选项，因为容器的镜像中可能没有许多常见的 Linux 网络工具。虽然不是理想的情况，但有时候有一个安装了这些工具的容器镜像是很好的，这样您就可以从容器的角度来排查网络问题。在本章中，我们将讨论如何专门为此目的构建 Docker 镜像。

## 准备工作

在本示例中，我们将使用单个 Docker 网络主机。假设 Docker 已安装并处于默认配置状态。您还需要 root 级别访问权限，以便检查和更改主机的网络和防火墙配置。

## 如何做…

Docker 镜像是通过定义 Dockerfile 来构建的。Dockerfile 定义了要使用的基础镜像以及容器内部要运行的命令。在我的示例中，我将定义 Dockerfile 如下：

```
FROM ubuntu:16.04
MAINTAINER Jon Langemak jon@interubernet.com
RUN apt-get update && apt-get install -y apache2 net-tools \
inetutils-ping curl dnsutils vim ethtool tcpdump
ADD index.html /var/www/html/index.html
ENV APACHE_RUN_USER www-data
ENV APACHE_RUN_GROUP www-data
ENV APACHE_LOG_DIR /var/log/apache2
ENV APACHE_PID_FILE /var/run/apache2/apache2.pid
ENV APACHE_LOCK_DIR /var/run/apache2
RUN mkdir -p /var/run/apache2
RUN chown www-data:www-data /var/run/apache2
EXPOSE 80
CMD ["/usr/sbin/apache2", "-D", "FOREGROUND"]
```

这个镜像的目标是双重的。首先，我希望能够以分离模式运行容器，并且让它提供一个服务。这将允许我定义容器并验证诸如端口发布之类的功能是否在主机上正常工作。这个容器镜像为我提供了一个已知良好的容器，将在端口`80`上发布一个服务。为此，我们使用 Apache 来托管一个简单的索引页面。

索引文件在构建时被拉入镜像中，并且可以由您自定义。我使用一个简单的 HTML 页面`index.html`，显示大红色的字体，如下所示：

```
<body>
  <html>
    <h1><span style="color:#FF0000;font-size:72px;">Test Web Server - Running on port 80</span>
    </h1>
</body>
  </html>
```

其次，镜像中安装了许多网络工具。您会注意到我正在安装以下软件包：

+   `net-tools`：提供网络实用程序以查看和配置接口

+   `inetutils-ping`：提供 ping 功能

+   `curl`：这是从其他网络端点拉取文件

+   `dnsutils`：这是用于解析 DNS 名称和其他 DNS 跟踪

+   `ethtool`：这是从接口获取信息和统计信息

+   `tcpdump`：这是从容器内进行数据包捕获

如果您定义了这个 Dockerfile，以及它所需的支持文件（一个索引页面），您可以按以下方式构建图像：

```
sudo docker build -t <tag name for image> <path files ('.' If local)>
```

### 注意

在构建图像时，您可以定义很多选项。查看`docker build --help`以获取更多信息。

然后 Docker 将处理 Dockerfile，如果成功，它将生成一个`docker image`文件，然后您可以将其推送到您选择的容器注册表，以便在其他主机上使用`docker pull`进行消费。

构建完成后，您可以运行它并验证工具是否按预期工作。在容器内有`ethtool`意味着我们可以轻松确定 VETH 对的主机端 VETH 端：

```
user@docker1:~$ docker run -dP --name nettest jonlangemak/net_tools
user@docker1:~$ docker exec -it nettest /bin/bash
root@2ef59fcc0f60:/# **ethtool -S eth0**
**NIC statistics:**
 **peer_ifindex: 5**
root@2ef59fcc0f60:/#
```

我们还可以执行本地的`tcpdump`操作来验证到达容器的流量：

```
root@2ef59fcc0f60:/# tcpdump -qnn -i eth0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
**15:17:43.442243 IP 10.20.30.41.54974 > 172.17.0.3.80: tcp 0**
**15:17:43.442286 IP 172.17.0.3.80 > 10.20.30.41.54974: tcp 0**

```

随着您的用例的改变，您可以修改 Dockerfile，使其更符合您自己的用例。在容器内进行故障排除时，能够进行故障排除可能会在诊断连接问题时提供很大帮助。

### 注意

这个图像只是一个例子。有很多方法可以使它更加轻量级。我决定使用 Ubuntu 作为基础图像，只是为了熟悉起见。前面描述的图像因此相当沉重。

# 重置本地 Docker 网络数据库

随着用户定义网络的出现，用户可以为其容器定义自定义网络类型。一旦定义，这些网络将在系统重新启动时持久存在，直到被管理员删除。为了使这种持久性工作，Docker 需要一些地方来存储与您的用户定义网络相关的信息。答案是一个本地主机的数据库文件。在一些罕见的情况下，这个数据库可能与主机上容器的当前状态不同步，或者变得损坏。这可能会导致与删除容器、删除网络和启动 Docker 服务相关的问题。在这个教程中，我们将向您展示如何删除数据库以将 Docker 恢复到其默认网络配置。

## 准备工作

在本教程中，我们将使用单个 Docker 网络主机。假设 Docker 已安装并处于默认配置状态。您还需要 root 级别访问权限，以便检查和更改主机的网络和防火墙配置。

## 如何操作…

Docker 将与用户定义网络相关的信息存储在本地主机上的数据库中。当定义网络时，会将数据写入该数据库，并在服务启动时从中读取。在极少数情况下，如果该数据库不同步或损坏，您可以删除数据库并重新启动 Docker 服务，以重置 Docker 用户定义网络并恢复三种默认网络类型（桥接、主机和无）。

### 注意

警告：删除此数据库会删除主机上的任何 Docker 用户定义网络。最好只在万不得已且有能力重新创建先前定义的网络时才这样做。在尝试此操作之前，应该追求所有其他故障排除选项，并在删除之前创建文件的备份。

该数据库的名称为`local-kv.db`，存储在路径`/var/lib/network/files/`中。访问或删除该文件需要 root 级别访问权限。为了方便浏览这个受保护的目录，我们将切换到 root 用户：

```
user@docker1:~$ sudo su
[sudo] password for user:
root@docker1:/home/user# cd **/var/lib/docker/network/files**
root@docker1:/var/lib/docker/network/files# ls -al
total 72
drwxr-x--- 2 root root 32768 Aug  9 21:27 .
drwxr-x--- 3 root root  4096 Apr  3 21:04 ..
-rw-r--r-- 1 root root 65536 Aug  9 21:27 **local-kv.db**
root@docker1:/var/lib/docker/network/files#
```

为了演示删除此文件时会发生什么，让我们首先创建一个新的用户定义网络并将一个容器连接到它：

```
root@docker1:~# **docker network create -d bridge mybridge**
c765f1d24345e4652b137383839aabdd3b01b1441d1d81ad4b4e17229ddca7ac
root@docker1:~# **docker run -d --name web1 --net mybridge jonlangemak/web_server_1**
24a6497e99de9e114b617b65673a8a50492655e9869dbf7f7930dd7f9f930b5e
root@docker1:~#
```

现在让我们删除文件`local-db.kv`：

```
root@docker1:/var/lib/docker/network/files# rm local-kv.db
```

尽管这对正在运行的容器没有立即影响，但它阻止我们向与此用户定义网络关联的新容器添加、删除或启动：

```
root@docker1:/~# docker run -d --name web2 --net mybridge \
jonlangemak/web_server_2
2ef7e52f44c93412ea7eaa413f523020a65f1a9fa6fd6761ffa6edea157c2623
**docker: Error response from daemon: failed to update store for object type *libnetwork.endpointCnt: Key not found in store.**
root@docker1:~#
```

删除`boltdb`数据库文件`local-kv.db`后，您需要重新启动 Docker 服务，以便 Docker 使用默认设置重新创建它：

```
root@docker1:/var/lib/docker/network/files# cd
root@docker1:~# systemctl restart docker
root@docker1:~# ls /var/lib/docker/network/files
local-kv.db
root@docker1:~# docker network ls
NETWORK ID          NAME                DRIVER
**bfd1ba1175a9        none                null**
**0740840aef37        host                host**
**97cbc0e116d7        bridge              bridge**
root@docker1:/var/lib/docker/network/files#
```

现在文件已重新创建，您将再次能够创建用户定义网络。但是，以前连接到先前配置的用户定义网络的任何容器现在将无法启动：

```
root@docker1:~# docker start web1
**Error response from daemon: network mybridge not found**
Error: failed to start containers: web1
root@docker1:~#
```

这是预期行为，因为 Docker 仍然认为容器应该在该网络上有一个接口：

```
root@docker1:~# docker inspect web1
…<Additional output removed for brevity>…
            "Networks": {
                "**mybridge**": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "c765f1d24345e4652b137383839aabdd3b01b1441d1d81ad4b4e17229ddca7ac",
…<Additional output removed for brevity>…
root@docker1:~#
```

为了解决这个问题，你有两个选择。首先，你可以使用与最初配置时相同的配置选项重新创建名为`mybridge`的用户定义网络。如果这不起作用，你唯一的选择就是删除容器并重新启动一个新实例，引用新创建的或默认网络。

### 注意：…

：GitHub 上已经讨论过 Docker 的新版本是否支持在使用`docker network disconnect`子命令时使用`--force`标志。在 1.10 版本中，存在这个参数，但仍然不喜欢用户定义的网络不存在。如果你正在运行一个更新的版本，这也许值得一试。

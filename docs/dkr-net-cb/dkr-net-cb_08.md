# 第八章：使用 Flannel

在本章中，我们将涵盖以下配方：

+   安装和配置 Flannel

+   将 Flannel 与 Docker 集成

+   使用 VXLAN 后端

+   使用主机网关后端

+   指定 Flannel 选项

# 介绍

Flannel 是由**CoreOS**团队开发的 Docker 的第三方网络解决方案。Flannel 是早期旨在为每个容器提供唯一可路由 IP 地址的项目之一。这消除了跨主机容器到容器通信需要使用发布端口的要求。与我们审查过的其他一些解决方案一样，Flannel 使用键值存储来跟踪分配和各种其他配置设置。但是，与 Weave 不同，Flannel 不提供与 Docker 服务的直接集成，也不提供插件。相反，Flannel 依赖于您告诉 Docker 使用 Flannel 网络来配置容器。在本章中，我们将介绍如何安装 Flannel 以及其各种配置选项。

# 安装和配置 Flannel

在这个教程中，我们将介绍安装 Flannel。Flannel 需要安装一个密钥存储和 Flannel 服务。由于每个服务的依赖关系，它们需要在 Docker 主机上配置为实际服务。为此，我们将利用`systemd`单元文件来定义每个相应的服务。

## 准备工作

在本示例中，我们将使用与第三章中使用的相同的实验拓扑，*用户定义的网络*，在那里我们讨论了用户定义的覆盖网络：

![准备工作](img/B05453_08_01.jpg)

你需要一对主机，最好其中一些位于不同的子网上。假设在这个实验中使用的 Docker 主机处于它们的默认配置中。在某些情况下，我们所做的更改可能需要您具有系统的根级访问权限。

## 如何做…

如前所述，Flannel 依赖于一个键值存储来向参与 Flannel 网络的所有节点提供信息。在其他示例中，我们运行了基于容器的键值存储，如 Consul，以提供此功能。由于 Flannel 是由 CoreOS 构建的，我们将利用他们的键值存储`etcd`。虽然`etcd`以容器格式提供，但由于 Flannel 工作所需的一些先决条件，我们无法轻松使用基于容器的版本。也就是说，我们将下载`etcd`和 Flannel 的二进制文件，并在我们的主机上将它们作为服务运行。

让我们从`etcd`开始，因为它是 Flannel 的先决条件。你需要做的第一件事是下载代码。在这个例子中，我们将利用`etcd`版本 3.0.12，并在主机`docker1`上运行键值存储。要下载二进制文件，我们将运行以下命令：

```
user@docker1:~$ curl -LO \
https://github.com/coreos/etcd/releases/download/v3.0.12/\
etcd-v3.0.12-linux-amd64.tar.gz
```

下载完成后，我们可以使用以下命令从存档中提取二进制文件：

```
user@docker1:~$ tar xzvf etcd-v3.0.12-linux-amd64.tar.gz
```

然后我们可以将需要的二进制文件移动到正确的位置，使它们可以执行。在这种情况下，位置是`/usr/bin`，我们想要的二进制文件是`etcd`服务本身以及其命令行工具`etcdctl`：

```
user@docker1:~$ cd etcd-v3.0.12-linux-amd64
user@docker1:~/etcd-v2.3.7-linux-amd64$ sudo mv etcd /usr/bin/
user@docker1:~/etcd-v2.3.7-linux-amd64$ sudo mv etcdctl /usr/bin/
```

现在我们已经把所有的部件都放在了正确的位置，我们需要做的最后一件事就是在系统上创建一个服务，来负责运行`etcd`。由于我们的 Ubuntu 版本使用`systemd`，我们需要为`etcd`服务创建一个 unit 文件。要创建服务定义，您可以在`/lib/systemd/system/`目录中创建一个服务 unit 文件：

```
user@docker1:~$  sudo vi /lib/systemd/system/etcd.service
```

然后，您可以创建一个运行`etcd`的服务定义。`etcd`服务的一个示例 unit 文件如下所示：

```
[Unit]
Description=etcd key-value store
Documentation=https://github.com/coreos/etcd
After=network.target

[Service]
Environment=DAEMON_ARGS=
Environment=ETCD_NAME=%H
Environment=ETCD_ADVERTISE_CLIENT_URLS=http://0.0.0.0:2379
Environment=ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
Environment=ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2378
Environment=ETCD_DATA_DIR=/var/lib/etcd/default
Type=notify
ExecStart=/usr/bin/etcd $DAEMON_ARGS
Restart=always
RestartSec=10s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### 注意

请记住，`systemd`可以根据您的要求以许多不同的方式进行配置。前面给出的 unit 文件演示了配置`etcd`作为服务的一种方式。

一旦 unit 文件就位，我们可以重新加载`systemd`，然后启用并启动服务：

```
user@docker1:~$ sudo systemctl daemon-reload
user@docker1:~$ sudo systemctl enable etcd
user@docker1:~$ sudo systemctl start etcd
```

如果由于某种原因服务无法启动或保持启动状态，您可以使用`systemctl status etcd`命令来检查服务的状态：

```
user@docker1:~$ systemctl status etcd
  etcd.service - etcd key-value store
   Loaded: loaded (/lib/systemd/system/etcd.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2016-10-11 13:41:01 CDT; 1h 30min ago
     Docs: https://github.com/coreos/etcd
 Main PID: 17486 (etcd)
    Tasks: 8
   Memory: 8.5M
      CPU: 22.095s
   CGroup: /system.slice/etcd.service
           └─17486 /usr/bin/etcd

Oct 11 13:41:01 docker1 etcd[17486]: setting up the initial cluster version to 3.0
Oct 11 13:41:01 docker1 etcd[17486]: published {Name:docker1 **ClientURLs:[http://0.0.0.0:2379]}** to cluster cdf818194e3a8c32
Oct 11 13:41:01 docker1 etcd[17486]: ready to serve client requests
Oct 11 13:41:01 docker1 etcd[17486]: **serving insecure client requests on 0.0.0.0:2379, this is strongly  iscouraged!**
Oct 11 13:41:01 docker1 systemd[1]: Started etcd key-value store.
Oct 11 13:41:01 docker1 etcd[17486]: set the initial cluster version to 3.0
Oct 11 13:41:01 docker1 etcd[17486]: enabled capabilities for version 3.0
Oct 11 15:04:20 docker1 etcd[17486]: start to snapshot (applied: 10001, lastsnap: 0)
Oct 11 15:04:20 docker1 etcd[17486]: saved snapshot at index 10001
Oct 11 15:04:20 docker1 etcd[17486]: compacted raft log at 5001
user@docker1:~$
```

稍后，如果您在使用启用 Flannel 的节点与`etcd`通信时遇到问题，请检查并确保`etcd`允许在所有接口（`0.0.0.0`）上访问，如前面加粗的输出所示。这在示例单元文件中有定义，但如果未定义，`etcd`将默认仅在本地环回接口（`127.0.0.1`）上侦听。这将阻止远程服务器访问该服务。

### 注意

由于键值存储配置是明确为了演示 Flannel 而进行的，我们不会涵盖键值存储的基础知识。这些配置选项足以让您在单个节点上运行，并且不打算在生产环境中使用。在将其用于生产环境之前，请确保您了解`etcd`的工作原理。

一旦启动了`etcd`服务，我们就可以使用`etcdctl`命令行工具来配置 Flannel 的一些基本设置：

```
user@docker1:~$ etcdctl mk /coreos.com/network/config \
'{"Network":"10.100.0.0/16"}'
```

我们将在以后的教程中讨论这些配置选项，但现在只需知道我们定义为`Network`参数的子网定义了 Flannel 的全局范围。

现在我们已经配置了`etcd`，我们可以专注于配置 Flannel 本身。将 Flannel 配置为系统服务与我们刚刚为`etcd`所做的非常相似。主要区别在于我们将在所有四个实验室主机上进行相同的配置，而键值存储只在单个主机上配置。我们将展示在单个主机`docker4`上安装 Flannel，但您需要在实验室环境中的每个主机上重复这些步骤，以便成为 Flannel 网络的成员：

首先，我们将下载 Flannel 二进制文件。在本例中，我们将使用版本 0.5.5：

```
user@docker4:~$ cd /tmp/
user@docker4:/tmp$ curl -LO \
https://github.com/coreos/flannel/releases/download/v0.6.2/\
flannel-v0.6.2-linux-amd64.tar.gz
```

然后，我们需要从存档中提取文件并将`flanneld`二进制文件移动到正确的位置。请注意，与`etcd`一样，没有命令行工具与 Flannel 交互：

```
user@docker4:/tmp$ tar xzvf flannel-v0.6.2-linux-amd64.tar.gz
user@docker4:/tmp$ sudo mv flanneld /usr/bin/
```

与`etcd`一样，我们希望定义一个`systemd`单元文件，以便我们可以在每个主机上将`flanneld`作为服务运行。要创建服务定义，您可以在`/lib/systemd/system/`目录中创建另一个服务单元文件：

```
user@docker4:/tmp$ sudo vi /lib/systemd/system/flanneld.service
```

然后，您可以创建一个运行`etcd`的服务定义。`etcd`服务的示例单元文件如下所示：

```
[Unit]
Description=Flannel Network Fabric
Documentation=https://github.com/coreos/flannel
Before=docker.service
After=etcd.service

[Service]
Environment='DAEMON_ARGS=--etcd-endpoints=http://10.10.10.101:2379'
Type=notify
ExecStart=/usr/bin/flanneld $DAEMON_ARGS
Restart=always
RestartSec=10s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

一旦单元文件就位，我们可以重新加载`systemd`，然后启用并启动服务：

```
user@docker4:/tmp$ sudo systemctl daemon-reload
user@docker4:/tmp$ sudo systemctl enable flanneld
user@docker4:/tmp$ sudo systemctl start flanneld
```

如果由于某种原因服务无法启动或保持启动状态，您可以使用`systemctl status flanneld`命令来检查服务的状态：

```
user@docker4:/tmp$ systemctl status flanneld
  flanneld.service - Flannel Network Fabric
   Loaded: loaded (/lib/systemd/system/flanneld.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2016-10-12 08:50:54 CDT; 6s ago
     Docs: https://github.com/coreos/flannel
 Main PID: 25161 (flanneld)
    Tasks: 6
   Memory: 3.3M
      CPU: 12ms
   CGroup: /system.slice/flanneld.service
           └─25161 /usr/bin/flanneld --etcd-endpoints=http://10.10.10.101:2379

Oct 12 08:50:54 docker4 systemd[1]: Starting Flannel Network Fabric...
Oct 12 08:50:54 docker4 flanneld[25161]: I1012 08:50:54.409928 25161 main.go:126] Installing signal handlers
Oct 12 08:50:54 docker4 flanneld[25161]: I1012 08:50:54.410384 25161 manager.go:133] Determining IP address of default interface
Oct 12 08:50:54 docker4 flanneld[25161]: I1012 08:50:54.410793 25161 manager.go:163] Using 192.168.50.102 as external interface
Oct 12 08:50:54 docker4 flanneld[25161]: I1012 08:50:54.411688 25161 manager.go:164] Using 192.168.50.102 as external endpoint
Oct 12 08:50:54 docker4 flanneld[25161]: I1012 08:50:54.423706 25161 local_manager.go:179] **Picking subnet in range 10.100.1.0 ... 10.100.255.0**
Oct 12 08:50:54 docker4 flanneld[25161]: I1012 08:50:54.429636 25161 manager.go:246] **Lease acquired: 10.100.15.0/24**
Oct 12 08:50:54 docker4 flanneld[25161]: I1012 08:50:54.430507 25161 network.go:98] Watching for new subnet leases
Oct 12 08:50:54 docker4 systemd[1]: **Started Flannel Network Fabric.**
user@docker4:/tmp$
```

您应该在日志中看到类似的输出，表明 Flannel 在您配置的`etcd`全局范围分配中找到了一个租约。这些租约对每个主机都是本地的，我经常将它们称为本地范围或网络。下一步是在其余主机上完成此配置。通过检查每个主机上的 Flannel 日志，我可以知道为每个主机分配了哪些子网。在我的情况下，我得到了以下结果：

+   `docker1`：`10.100.93.0/24`

+   `docker2`：`10.100.58.0/24`

+   `docker3`：`10.100.90.0/24`

+   `docker4`：`10.100.15.0/24`

此时，Flannel 已经完全配置好了。在下一个教程中，我们将讨论如何配置 Docker 来使用 Flannel 网络。

# 将 Flannel 与 Docker 集成

正如我们之前提到的，目前 Flannel 和 Docker 之间没有直接集成。也就是说，我们需要找到一种方法将容器放入 Flannel 网络，而 Docker 并不直接知道正在发生的事情。在这个教程中，我们将展示如何做到这一点，讨论导致我们当前配置的一些先决条件，并了解 Flannel 如何处理主机之间的通信。

## 准备工作

假设您正在构建上一个教程中描述的实验室。在某些情况下，我们所做的更改可能需要您具有系统的根级访问权限。

## 如何做到这一点...

在上一个教程中，我们配置了 Flannel，但我们并没有从网络的角度实际检查 Flannel 配置到底做了什么。让我们快速查看一下我们的一个启用了 Flannel 的主机的配置，看看发生了什么变化：

```
user@docker4:~$ ip addr
…<loopback interface removed for brevity>…
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether d2:fe:5e:b2:f6:43 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.102/24 brd 192.168.50.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::d0fe:5eff:feb2:f643/64 scope link
       valid_lft forever preferred_lft forever
**3: flannel0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN group default qlen 500**
 **link/none**
 **inet 10.100.15.0/16 scope global flannel0**
 **valid_lft forever preferred_lft forever**
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:16:78:74:cf brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever 
user@docker4:~$
```

您会注意到一个名为`flannel0`的新接口的添加。您还会注意到它具有分配给此主机的`/24`本地范围内的 IP 地址。如果我们深入挖掘一下，我们可以使用`ethtool`来确定这个接口是一个虚拟的`tun`接口。

```
user@docker4:~$ ethtool -i flannel0
**driver: tun**
version: 1.6
firmware-version:
bus-info: tun
supports-statistics: no
supports-test: no
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: no
user@docker4:~$
```

Flannel 在运行 Flannel 服务的每个主机上创建了这个接口。请注意，`flannel0`接口的子网掩码是`/16`，它覆盖了我们在`etcd`中定义的整个全局范围分配。尽管为主机分配了`/24`范围，但主机认为整个`/16`都可以通过`flannel0`接口访问：

```
user@docker4:~$ ip route
default via 192.168.50.1 dev eth0
**10.100.0.0/16 dev flannel0  proto kernel  scope link  src 10.100.93.0**
172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1
192.168.50.0/24 dev eth0  proto kernel  scope link  src 192.168.50.102
user@docker4:~$
```

有了接口后，就会创建这条路由，确保前往其他主机上分配的本地范围的流量通过`flannel0`接口。我们可以通过 ping 其他主机上的其他`flannel0`接口来证明这一点：

```
user@docker4:~$ **ping 10.100.93.0 -c 2**
PING 10.100.93.0 (10.100.93.0) 56(84) bytes of data.
**64 bytes from 10.100.93.0: icmp_seq=1 ttl=62 time=0.901 ms**
**64 bytes from 10.100.93.0: icmp_seq=2 ttl=62 time=0.930 ms**
--- 10.100.93.0 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.901/0.915/0.930/0.033 ms
user@docker4:~$
```

由于物理网络对`10.100.0.0/16`网络空间一无所知，Flannel 必须在流经物理网络时封装流量。为了做到这一点，它需要知道哪个物理 Docker 主机分配了给定的范围。回想一下我们在上一篇示例中检查的 Flannel 日志，Flannel 根据主机的默认路由为每个主机选择了一个外部接口：

```
I0707 09:07:01.733912 02195 main.go:130] **Determining IP address of default interface**
I0707 09:07:01.734374 02195 main.go:188] **Using 192.168.50.102 as external interface**

```

这些信息以及分配给每个主机的范围都在键值存储中注册。使用这些信息，Flannel 可以确定哪个主机分配了哪个范围，并可以使用该主机的外部接口作为发送封装流量的目的地。

### 注意

Flannel 支持多个后端或传输机制。默认情况下，它会在端口`8285`上使用 UDP 封装流量。在接下来的示例中，我们将讨论其他后端选项。

既然我们知道了 Flannel 的工作原理，我们需要解决如何将实际的 Docker 容器放入 Flannel 网络中。最简单的方法是让 Docker 使用分配的范围作为`docker0`桥接的子网。Flannel 将范围信息写入一个文件，保存在`/run/flannel/subnet.env`中：

```
user@docker4:~$ more /run/flannel/subnet.env
FLANNEL_NETWORK=10.100.0.0/16
**FLANNEL_SUBNET=10.100.15.1/24**
**FLANNEL_MTU=1472**
FLANNEL_IPMASQ=false
user@docker4:~$
```

利用这些信息，我们可以配置 Docker 使用正确的子网作为其桥接接口。Flannel 提供了两种方法来实现这一点。第一种方法涉及使用随 Flannel 二进制文件一起提供的脚本生成新的 Docker 配置文件。该脚本允许您输出一个使用`subnet.env`文件中信息的新 Docker 配置文件。例如，我们可以使用该脚本生成一个新的配置，如下所示：

```
user@docker4:~$ cd /tmp
user@docker4:/tmp$ ls
flannel-v0.6.2-linux-amd64.tar.gz  **mk-docker-opts.sh**  README.md  
user@docker4:~/flannel-0.5.5$ ./**mk-docker-opts.sh -c -d \**
**example_docker_config**
user@docker4:/tmp$ more example_docker_config
**DOCKER_OPTS=" --bip=10.100.15.1/24 --ip-masq=true --mtu=1472"**
user@docker4:/tmp$
```

在不使用`systemd`的系统中，Docker 在大多数情况下会自动检查`/etc/default/docker`文件以获取服务级选项。这意味着我们可以简单地让 Flannel 将前面提到的配置文件写入`/etc/default/docker`，这样当服务重新加载时，Docker 就可以使用新的设置。然而，由于我们的系统使用`systemd`，这种方法需要更新我们的 Docker drop-in 文件(`/etc/systemd/system/docker.service.d/docker.conf`)，使其如下所示：

```
[Service]
EnvironmentFile=**/etc/default/docker**
ExecStart=
ExecStart=/usr/bin/dockerd **$DOCKER_OPTS**

```

加粗的行表示服务应该检查文件`etc/default/docker`，然后加载变量`$DOCKER_OPTS`以在运行时传递给服务。如果您使用此方法，为了简单起见，定义所有服务级选项都在`etc/default/docker`中可能是明智的。

### 注意：

应该注意的是，这种第一种方法依赖于运行脚本来生成配置文件。如果您手动运行脚本来生成文件，则有可能如果 Flannel 配置更改，配置文件将过时。稍后显示的第二种方法更加动态，因为`/run/flannel/subnet.env`文件由 Flannel 服务更新。

尽管第一种方法当然有效，但我更喜欢使用一个略有不同的方法，我只是从`/run/flannel/subnet.env`文件中加载变量，并在 drop-in 文件中使用它们。为了做到这一点，我们将我们的 Docker drop-in 文件更改为如下所示：

```
[Service]
EnvironmentFile=/run/flannel/subnet.env
ExecStart=
ExecStart=/usr/bin/dockerd --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
```

通过将`/run/flannel/subnet.env`指定为`EnvironmentFile`，我们使文件中定义的变量可供服务定义中使用。然后，我们只需在服务启动时将它们用作选项传递给服务。如果我们在我们的 Docker 主机上进行这些更改，重新加载`systemd`配置，并重新启动 Docker 服务，我们应该看到我们的`docker0`接口现在反映了 Flannel 子网：

```
user@docker4:~$ ip addr show dev docker0
8: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:24:0a:e3:c8 brd ff:ff:ff:ff:ff:ff
    inet **10.100.15.1/24** scope global docker0
       valid_lft forever preferred_lft forever
user@docker4:~$ 
```

您还可以根据 Flannel 配置手动更新 Docker 服务级参数。只需确保您使用`/run/flannel/subnet.env`文件中的信息。无论您选择哪种方法，请确保`docker0`桥在所有四个 Docker 主机上都使用 Flannel 指定的配置。我们的拓扑现在应该是这样的：

![如何做…](img/B05453_08_02.jpg)

由于每个 Docker 主机只使用其子网的 Flannel 分配范围，因此每个主机都认为全局 Flannel 网络中包含的剩余子网仍然可以通过`flannel0`接口访问。只有分配的本地范围的特定`/24`可以通过`docker0`桥在本地访问：

```
user@docker4:~$ ip route
default via 192.168.50.1 dev eth0 onlink
**10.100.0.0/16 dev flannel0**  proto kernel  scope link src 10.100.15.0
**10.100.15.0/24 dev docker0**  proto kernel  scope link src 10.100.15.1 
192.168.50.0/24 dev eth0  proto kernel  scope link src 192.168.50.102
user@docker4:~$
```

我们可以通过在两个不同的主机上运行两个不同的容器来验证 Flannel 的操作：

```
user@**docker1**:~$ docker run -dP **--name=web1** jonlangemak/web_server_1
7e44a55c7ea7704d97a8804bfa211344c66f9fb83b3ac17f697c504b3b193e2d
user@**docker1**:~$
user@**docker4**:~$ docker run -dP **--name=web2** jonlangemak/web_server_2
39a47920588b5e0d77ca9d2838988e2d8de893dee6198759f9ddbd3b38cea80d
user@**docker4**:~$
```

现在，我们可以通过 IP 地址直接访问每个容器上运行的服务。首先，找到一个容器的 IP 地址：

```
user@**docker1**:~$ docker exec -it **web1 ip addr show dev eth0**
12: eth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1472 qdisc noqueue state UP
    link/ether 02:42:0a:64:5d:02 brd ff:ff:ff:ff:ff:ff
    inet **10.100.93.2/24** scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:aff:fe64:5d02/64 scope link
       valid_lft forever preferred_lft forever
user@**docker1**:~$
```

然后，从第二个容器访问服务：

```
user@**docker4**:~$ docker exec -it web2 curl http://**10.100.93.2**
<body>
  <html>
    <h1><span style="color:#FF0000;font-size:72px;">**Web Server #1 - Running on port 80**</span>
    </h1>
</body>
  </html>
user@**docker4**:~$
```

连接正常工作。现在我们已经将整个 Flannel 配置与 Docker 一起工作，重要的是要指出我们做事情的顺序。我们查看的其他解决方案能够将其解决方案的某些部分容器化。例如，Weave 能够以容器格式提供其服务，而不需要像我们使用 Flannel 那样需要本地服务。对于 Flannel，每个组件都有一个先决条件才能工作。

例如，我们需要在 Flannel 注册之前运行`etcd`服务。这本身并不是一个很大的问题，如果`etcd`和 Flannel 都在容器中运行，你可以相当容易地解决这个问题。然而，由于 Docker 需要对其桥接 IP 地址进行的更改是在服务级别完成的，所以 Docker 在启动之前需要知道有关 Flannel 范围的信息。这意味着我们不能在 Docker 容器中运行`etcd`和 Flannel 服务，因为我们无法在没有从`etcd`读取密钥生成的 Flannel 信息的情况下启动 Docker。在这种情况下，了解每个组件的先决条件是很重要的。

### 注意

在 CoreOS 中运行 Flannel 时，他们能够在容器中运行这些组件。解决方案在他们的文档中详细说明了这一点，在*底层*部分的这一行：

[`coreos.com/flannel/docs/latest/flannel-config.html`](https://coreos.com/flannel/docs/latest/flannel-config.html)

# 使用 VXLAN 后端

如前所述，Flannel 支持多种不同的后端配置。后端被认为是 Flannel 在启用 Flannel 的主机之间传递流量的手段。默认情况下，这是通过 UDP 完成的，就像我们在前面的示例中看到的那样。然而，Flannel 也支持 VXLAN。使用 VXLAN 而不是 UDP 的优势在于，较新的主机支持内核中的 VXLAN。在这个示例中，我们将演示如何将 Flannel 后端类型更改为 VXLAN。

## 准备工作

假设您正在构建本章前面示例中描述的实验室。您将需要与 Docker 集成的启用了 Flannel 的主机，就像本章的前两个示例中描述的那样。在某些情况下，我们所做的更改可能需要您具有系统的根级访问权限。

## 如何做…

在你首次在`etcd`中实例化网络时，你希望使用的后端类型是被定义的。由于我们在定义网络`10.100.0.0/16`时没有指定类型，Flannel 默认使用 UDP 后端。这可以通过更新我们最初在`etcd`中设置的配置来改变。回想一下，我们的 Flannel 网络是通过这个命令首次定义的：

```
etcdctl mk /coreos.com/network/config '{"Network":"10.10.0.0/16"}'
```

注意我们如何使用`etcdctl`的`mk`命令来创建键。如果我们想将后端类型更改为 VXLAN，我们可以运行这个命令：

```
etcdctl set /coreos.com/network/config '{"Network":"10.100.0.0/16", "Backend": {"Type": "vxlan"}}'
```

请注意，由于我们正在更新对象，我们现在使用`set`命令代替`mk`。虽然在纯文本形式下有时很难看到，但我们传递给`etcd`的格式正确的 JSON 看起来像这样：

```
{
    "Network": "10.100.0.0/16",
    "Backend": {
        "Type": "vxlan",
    }
}
```

这将定义这个后端的类型为 VXLAN。虽然前面的配置本身足以改变后端类型，但有时我们可以指定作为后端的一部分的额外参数。例如，当将类型定义为 VXLAN 时，我们还可以指定**VXLAN 标识符**（**VNI**）和 UDP 端口。如果未指定，VNI 默认为`1`，端口默认为`8472`。为了演示，我们将默认值作为我们配置的一部分应用：

```
user@docker1:~$ etcdctl set /coreos.com/network/config \
'{"Network":"10.100.0.0/16", "Backend": {"Type": "vxlan","VNI": 1, "Port": 8472}}'
```

这在格式正确的 JSON 中看起来像这样：

```
{
    "Network": "10.100.0.0/16",
    "Backend": {
        "Type": "vxlan",
        "VNI": 1,
        "Port": 8472
    }
}
```

如果我们运行命令，本地`etcd`实例的配置将被更新。我们可以通过`etcdctl`命令行工具查询`etcd`，以验证`etcd`是否具有正确的配置。要读取配置，我们可以使用`etcdctl get`子命令：

```
user@docker1:~$ etcdctl get /coreos.com/network/config
{"Network":"10.100.0.0/16", "Backend": {"Type": "vxlan", "VNI": 1, "Port": 8472}}
user@docker1:~$
```

尽管我们已成功更新了`etcd`，但每个节点上的 Flannel 服务不会根据这个新配置进行操作。这是因为每个主机上的 Flannel 服务只在服务启动时读取这些变量。为了使这个更改生效，我们需要重新启动每个节点上的 Flannel 服务：

```
user@docker4:~$ sudo systemctl restart flanneld
```

确保您重新启动每个主机上的 Flannel 服务。如果有些主机使用 VXLAN 后端，而其他主机使用 UDP 后端，主机将无法通信。重新启动后，我们可以再次检查我们的 Docker 主机的接口：

```
user@docker4:~$ ip addr show
…<Additional output removed for brevity>… 
11: **flannel.1**: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether 2e:28:e7:34:1a:ff brd ff:ff:ff:ff:ff:ff
    inet **10.100.15.0/16** scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::2c28:e7ff:fe34:1aff/64 scope link
       valid_lft forever preferred_lft forever 
```

在这里，我们可以看到主机现在有一个名为`flannel.1`的新接口。如果我们使用`ethtool`检查接口，我们可以看到它正在使用 VXLAN 驱动程序：

```
user@docker4:~$ **ethtool -i flannel.1**
driver: **vxlan**
version: 0.1
firmware-version:
bus-info:
supports-statistics: no
supports-test: no
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: no
user@docker4:~$
```

而且我们应该仍然能够使用 Flannel IP 地址访问服务：

```
user@**docker4**:~$ docker exec -it **web2 curl http://10.100.93.2**
<body>
  <html>
    <h1><span style="color:#FF0000;font-size:72px;">**Web Server #1 - Running on port 80**</span>
    </h1>
</body>
  </html>
user@**docker4**:~$
```

### 注意

如果您指定了不同的 VNI，Flannel 接口将被定义为`flannel.<VNI 编号>`。

重要的是要知道，Flannel 不会清理旧配置的遗留物。例如，如果您更改了`etcd`中的 VXLAN ID 并重新启动 Flannel 服务，您将得到两个接口在同一个网络上。您需要手动删除使用旧 VNI 命名的旧接口。此外，如果更改了分配给 Flannel 的子网，您需要在重新启动 Flannel 服务后重新启动 Docker 服务。请记住，Docker 在加载 Docker 服务时从 Flannel 读取配置变量。如果这些变化，您需要重新加载配置才能生效。

# 使用主机网关后端

正如我们已经看到的，Flannel 支持两种类型的覆盖网络。使用 UDP 或 VXLAN 封装，Flannel 可以在 Docker 主机之间构建覆盖网络。这样做的明显优势是，您可以在不触及物理底层网络的情况下，在不同的 Docker 节点之间提供网络。然而，某些类型的覆盖网络也会引入显著的性能惩罚，特别是对于在用户空间执行封装的进程。主机网关模式旨在通过不使用覆盖网络来解决这个问题。然而，这也带来了自己的限制。在这个示例中，我们将回顾主机网关模式可以提供什么，并展示如何配置它。

## 准备工作

在这个示例中，我们将稍微修改我们一直在使用的实验室。实验室拓扑将如下所示：

![准备工作](img/B05453_08_03.jpg)

在这种情况下，主机`docker3`和`docker4`现在具有与`docker1`和`docker2`相同子网的 IP 地址。也就是说，所有主机现在都是相互的二层邻接，并且可以直接通信，无需通过网关进行路由。一旦您将主机在此拓扑中重新配置，我们将希望清除 Flannel 配置。要做到这一点，请执行以下步骤：

+   在运行`etcd`服务的主机上：

```
sudo systemctl stop etcd
sudo rm -rf /var/lib/etcd/default 
sudo systemctl start etcd
```

+   在所有运行 Flannel 服务的主机上：

```
sudo systemctl stop flanneld
sudo ip link delete flannel.1
sudo systemctl --no-block start flanneld
```

### 注意

您会注意到我们在启动`flanneld`时传递了`systemctl`命令和`--no-block`参数。由于我们从`etcd`中删除了 Flannel 配置，Flannel 服务正在搜索用于初始化的配置。由于服务的定义方式（类型为通知），传递此参数是必需的，以防止命令在 CLI 上挂起。

## 如何做…

此时，您的 Flannel 节点将正在搜索其配置。由于我们删除了`etcd`数据存储，目前缺少告诉 Flannel 节点如何配置服务的密钥，Flannel 服务将继续轮询`etcd`主机，直到我们进行适当的配置。我们可以通过检查其中一个主机的日志来验证这一点：

```
user@docker4:~$ journalctl -f -u flanneld
-- Logs begin at Wed 2016-10-12 12:39:35 CDT. –
Oct 12 12:39:36 docker4 flanneld[873]: I1012 12:39:36.843784 00873 manager.go:163] **Using 10.10.10.104 as external interface**
Oct 12 12:39:36 docker4 flanneld[873]: I1012 12:39:36.844160 00873 manager.go:164] **Using 10.10.10.104 as external endpoint**
Oct 12 12:41:22 docker4 flanneld[873]: E1012 12:41:22.102872 00873 network.go:106] **failed to retrieve network config: 100: Key not found (/coreos.com)** [4]
Oct 12 12:41:23 docker4 flanneld[873]: E1012 12:41:23.104904 00873 network.go:106] **failed to retrieve network config: 100: Key not found (/coreos.com)** [4] 
```

重要的是要注意，此时 Flannel 已经通过查看哪个接口支持主机的默认路由来决定其外部端点 IP 地址：

```
user@docker4:~$ ip route
**default via 10.10.10.1 dev eth0**
10.10.10.0/24 dev eth0  proto kernel  scope link  src 10.10.10.104
user@docker4:~$
```

由于这恰好是`eth0`，Flannel 选择该接口的 IP 地址作为其外部地址。要配置主机网关模式，我们可以将以下配置放入`etcd`：

```
{  
   "Network":"10.100.0.0/16",
   "Backend":{  
      "Type":"host-gw"
   }
}
```

正如我们以前看到的，我们仍然指定一个网络。唯一的区别是我们提供了`type`为`host-gw`。将其插入`etcd`的命令如下：

```
user@docker1:~$ etcdctl set /coreos.com/network/config \
'{"Network":"10.100.0.0/16", "Backend": {"Type": "host-gw"}}'
```

在我们插入此配置后，Flannel 节点应该都会接收到新的配置。让我们检查主机`docker4`上 Flannel 的服务日志以验证这一点：

```
user@docker4:~$ journalctl -r -u flanneld
-- Logs begin at Wed 2016-10-12 12:39:35 CDT, end at Wed 2016-10-12 12:55:38 CDT. --
Oct 12 12:55:06 docker4 flanneld[873]: I1012 12:55:06.797289 00873 network.go:83] **Subnet added: 10.100.23.0/24 via 10.10.10.103**
Oct 12 12:55:06 docker4 flanneld[873]: I1012 12:55:06.796982 00873 network.go:83] **Subnet added: 10.100.20.0/24 via 10.10.10.101**
Oct 12 12:55:06 docker4 flanneld[873]: I1012 12:55:06.796468 00873 network.go:83] **Subnet added: 10.100.43.0/24 via 10.10.10.102**
Oct 12 12:55:06 docker4 flanneld[873]: I1012 12:55:06.785464 00873 network.go:51] **Watching for new subnet leases**
Oct 12 12:55:06 docker4 flanneld[873]: I1012 12:55:06.784436 00873 manager.go:246] **Lease acquired: 10.100.3.0/24**
Oct 12 12:55:06 docker4 flanneld[873]: I1012 12:55:06.779349 00873 local_manager.go:179] **Picking subnet in range 10.100.1.0 ... 10.100.255.0**

```

### 注意

`journalctl`命令对于查看由`systemd`管理的服务的所有日志非常有用。在前面的示例中，我们传递了`-r`参数以倒序显示日志（最新的在顶部）。我们还传递了`-u`参数以指定我们要查看日志的服务。

我们看到的最旧的日志条目是这个主机的 Flannel 服务在`10.100.0.0/16`子网内选择并注册范围。这与 UDP 和 VXLAN 后端的工作方式相同。接下来的三个日志条目显示 Flannel 检测到其他三个 Flannel 节点范围的注册。由于`etcd`正在跟踪每个 Flannel 节点的外部 IP 地址，以及它们注册的范围，所有 Flannel 主机现在都知道可以用什么外部 IP 地址来到达每个注册的 Flannel 范围。在覆盖模式（UDP 或 VXLAN）中，此外部 IP 地址被用作封装流量的目的地。在主机网关模式中，此外部 IP 地址被用作路由目的地。如果我们检查路由表，我们可以看到每个主机的路由条目：

```
user@docker4:~$ ip route
default via 10.10.10.1 dev eth0 onlink
10.10.10.0/24 dev eth0  proto kernel  scope link  src 10.10.10.104
**10.100.20.0/24 via 10.10.10.101 dev eth0**
**10.100.23.0/24 via 10.10.10.103 dev eth0**
**10.100.43.0/24 via 10.10.10.102 dev eth0**
user@docker4:~$
```

在这种配置中，Flannel 只是依赖基本路由来提供对所有 Flannel 注册范围的可达性。在这种情况下，主机`docker4`有路由到所有其他 Docker 主机的路由，以便到达它们的 Flannel 网络范围：

![如何做…](img/B05453_08_04.jpg)

这不仅比处理覆盖网络要简单得多，而且比要求每个主机为覆盖网络进行封装要更高效。这种方法的缺点是每个主机都需要在同一网络上有一个接口才能正常工作。如果主机不在同一网络上，Flannel 无法添加这些路由，因为这将需要上游网络设备（主机的默认网关）也具有有关如何到达远程主机的路由信息。虽然 Flannel 节点可以在其默认网关上指定静态路由，但物理网络对`10.100.0.0/16`网络一无所知，并且无法传递流量。其结果是主机网关模式限制了您可以放置启用 Flannel 的 Docker 主机的位置。

最后，重要的是要指出，Flannel 在 Docker 服务已经运行后可能已经改变状态。如果是这种情况，您需要重新启动 Docker，以确保它从 Flannel 中获取新的变量。如果在重新配置网络接口时重新启动了主机，则可能只需要启动 Docker 服务。系统启动时，服务可能因缺少 Flannel 配置信息而未能加载，现在应该已经存在。

### 注意

Flannel 还为各种云提供商（如 GCE 和 AWS）提供了后端。您可以查看它们的文档，以获取有关这些后端类型的更多具体信息。

# 指定 Flannel 选项

除了配置不同的后端类型，您还可以通过`etcd`和 Flannel 客户端本身指定其他选项。这些选项允许您限制 IP 分配范围，并指定用作 Flannel 节点外部 IP 端点的特定接口。在本教程中，我们将审查您在本地和全局都可以使用的其他配置选项。

## 做好准备

我们将继续构建上一章中的实验，在那里我们配置了主机网关后端。但是，实验拓扑将恢复到以前的配置，其中 Docker 主机`docker3`和`docker4`位于`192.168.50.0/24`子网中：

![做好准备](img/B05453_08_05.jpg)

一旦您在这个拓扑中配置了您的主机，我们将想要清除 Flannel 配置。为此，请执行以下步骤：

+   在运行`etcd`服务的主机上：

```
sudo systemctl stop etcd
sudo rm -rf /var/lib/etcd/default 
sudo systemctl start etcd
```

+   在所有运行 Flannel 服务的主机上：

```
sudo systemctl stop flanneld
sudo ip link delete flannel.1
sudo systemctl --no-block start flanneld
```

在某些情况下，我们所做的更改可能需要您具有系统的根级访问权限。

## 如何做…

之前的示例展示了如何指定整体 Flannel 网络或全局范围，并改变后端网络类型。我们还看到一些后端网络类型允许额外的配置选项。除了我们已经看到的选项之外，我们还可以全局配置其他参数，来决定 Flannel 的整体工作方式。有三个其他主要参数可以影响分配给 Flannel 节点的范围：

+   `SubnetLen`: 此参数以整数形式指定，并规定了分配给每个节点的范围的大小。正如我们所见，这默认为`/24`

+   `SubnetMin`: 此参数以字符串形式指定，并规定了范围分配应该开始的起始 IP 范围

+   `SubnetMax`: 此参数以字符串形式指定，并规定了子网分配应该结束的 IP 范围的末端

将这些选项与`Network`标志结合使用时，我们在分配网络时具有相当大的灵活性。例如，让我们使用这个配置：

```
{  
   "Network":"10.100.0.0/16",
   "SubnetLen":25,
   "SubnetMin":"10.100.0.0",
   "SubnetMax":"10.100.1.0",
   "Backend":{  
      "Type":"host-gw"
   }
}
```

这定义了每个 Flannel 节点应该获得一个`/25`的范围分配，第一个子网应该从`10.100.0.0`开始，最后一个子网应该结束于`10.100.1.0`。您可能已经注意到，在这种情况下，我们只有空间来容纳三个子网：

+   `10.100.0.0/25`

+   `10.100.0.128./25`

+   `10.100.1.0/25`

这是故意为了展示当 Flannel 在全局范围内空间不足时会发生什么。现在让我们使用这个命令将这个配置放入`etcd`中：

```
user@docker1:~$ etcdctl set /coreos.com/network/config \
 '{"Network":"10.100.0.0/16","SubnetLen": 25, "SubnetMin": "10.100.0.0", "SubnetMax": "10.100.1.0", "Backend": {"Type": "host-gw"}}'
```

一旦放置，您应该会看到大多数主机接收到本地范围的分配。但是，如果我们检查我们的主机，我们会发现有一个主机未能接收到分配。在我的情况下，那就是主机`docker4`。我们可以在 Flannel 服务的日志中看到这一点：

```
user@docker4:~$ journalctl -r -u flanneld
-- Logs begin at Wed 2016-10-12 12:39:35 CDT, end at Wed 2016-10-12 13:17:42 CDT. --
Oct 12 13:17:42 docker4 flanneld[1422]: E1012 13:17:42.650086 01422 network.go:106] **failed to register network: failed to acquire lease: out of subnets**
Oct 12 13:17:42 docker4 flanneld[1422]: I1012 13:17:42.649604 01422 local_manager.go:179] Picking subnet in range 10.100.0.0 ... 10.100.1.0
```

由于我们在全局范围内只允许了三个分配空间，第四个主机无法接收本地范围，并将继续请求，直到有一个可用。这可以通过更新`SubnetMax`参数为`10.100.1.128`并重新启动未能接收本地范围分配的主机上的 Flannel 服务来解决。

正如我所提到的，我们还可以将配置参数传递给每个主机上的 Flannel 服务。

### 注意

Flannel 客户端支持各种参数，所有这些参数都可以通过运行`flanneld --help`来查看。这些参数涵盖了新的和即将推出的功能，以及与基于 SSL 的通信相关的配置，在在运行这些类型的服务时，这些配置将是重要的。

从网络的角度来看，也许最有价值的配置选项是`--iface`参数，它允许您指定要用作 Flannel 外部端点的主机接口。为了了解其重要性，让我们看一个我们的多主机实验室拓扑的快速示例：

![如何做…](img/B05453_08_06.jpg)

如果你还记得，在主机网关模式下，Flannel 要求所有 Flannel 节点都是二层相邻的，或者在同一个网络上。在这种情况下，左侧有两个主机在`10.10.10.0/24`网络上，右侧有两个主机在`192.168.50.0/24`网络上。为了彼此通信，它们需要通过多层交换机进行路由。这种情况通常需要一个覆盖后端模式，可以通过多层交换机隧道传输容器流量。然而，如果主机网关模式是性能或其他原因的要求，如果您可以为主机提供额外的接口，您可能仍然可以使用它。例如，想象一下，这些主机实际上是虚拟机，相对容易为我们在每个主机上提供另一个接口，称之为`eth1`：

![如何做…](img/B05453_08_07.jpg)

这个接口可以专门用于 Flannel 流量，允许每个主机仍然在 Flannel 流量的情况下保持二层相邻，同时保持它们通过`eth0`的现有默认路由。然而，仅仅配置接口是不够的。请记住，Flannel 默认通过引用主机的默认路由来选择其外部端点接口。由于在这种模型中默认路由没有改变，Flannel 将无法添加适当的路由：

```
user@docker4:~$ journalctl -ru flanneld
-- Logs begin at Wed 2016-10-12 14:24:51 CDT, end at Wed 2016-10-12 14:31:14 CDT. --
Oct 12 14:31:14 docker4 flanneld[1491]: E1012 14:31:14.463106 01491 network.go:116] **Error adding route to 10.100.1.128/25 via 10.10.10.102: network is unreachable**
Oct 12 14:31:14 docker4 flanneld[1491]: I1012 14:31:14.462801 01491 network.go:83] Subnet added: 10.100.1.128/25 via 10.10.10.102
Oct 12 14:31:14 docker4 flanneld[1491]: E1012 14:31:14.462589 01491 network.go:116] **Error adding route to 10.100.0.128/25 via 10.10.10.101: network is unreachable**
Oct 12 14:31:14 docker4 flanneld[1491]: I1012 14:31:14.462008 01491 network.go:83] Subnet added: 10.100.0.128/25 via 10.10.10.101
```

由于 Flannel 仍然使用`eth0`接口作为其外部端点 IP 地址，它知道另一个子网上的主机是无法直接到达的。我们可以通过向 Flannel 服务传递`--iface`选项来告诉 Flannel 使用`eth1`接口来解决这个问题。

例如，我们可以通过更新 Flannel 服务定义（`/lib/systemd/system/flanneld.service`）来更改 Flannel 配置，使其如下所示：

```
[Unit]
Description=Flannel Network Fabric
Documentation=https://github.com/coreos/flannel
Before=docker.service
After=etcd.service

[Service]
Environment= 'DAEMON_ARGS=--etcd-endpoints=http://10.10.10.101:2379 **--iface=eth1'**
Type=notify
ExecStart=/usr/bin/flanneld $DAEMON_ARGS
Restart=always
RestartSec=10s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

有了这个配置，Flannel 将使用`eth1`接口作为其外部端点，从而使所有主机能够直接在`10.11.12.0/24`网络上进行通信。然后，您可以通过重新加载`systemd`配置并在所有主机上重新启动服务来加载新配置：

```
sudo systemctl daemon-reload
sudo systemctl restart flanneld
```

请记住，Flannel 使用外部端点 IP 地址来跟踪 Flannel 节点。更改这意味着 Flannel 将为每个 Flannel 节点分配一个新的范围。最好在加入 Flannel 节点之前配置这些选项。在我们的情况下，由于`etcd`已经配置好，我们将再次删除现有的`etcd`配置，并重新配置它，以便范围变得可用。

```
user@docker1:~$ sudo systemctl stop etcd
user@docker1:~$ sudo rm -rf /var/lib/etcd/default
user@docker1:~$ sudo systemctl start etcd
user@docker1:~$ etcdctl set /coreos.com/network/config \
 '{"Network":"10.100.0.0/16","SubnetLen": 25, "SubnetMin": "10.100.0.0", "SubnetMax": "10.100.1.128", "Backend": {"Type": "host-gw"}}'
```

如果您检查主机，现在应该看到它有三个 Flannel 路由——每个路由对应其他三个主机的分配范围之一：

```
user@docker1:~$ ip route
default via 10.10.10.1 dev eth0 onlink
10.10.10.0/24 dev eth0  proto kernel  scope link src 10.10.10.101
10.11.12.0/24 dev eth1  proto kernel  scope link src 10.11.12.101
**10.100.0.0/25 via 10.11.12.102 dev eth1**
**10.100.1.0/25 via 10.11.12.104 dev eth1**
**10.100.1.128/25 via 10.11.12.103 dev eth1**
10.100.0.128/25 dev docker0  proto kernel  scope link src 10.100.75.1 
user@docker1:~$
```

此外，如果您将通过 NAT 使用 Flannel，您可能还想查看`--public-ip`选项，该选项允许您定义节点的公共 IP 地址。这在云环境中尤为重要，因为服务器的真实 IP 地址可能被隐藏在 NAT 后面。

# 第六章：使用 Docker 内置安全功能

在本章中，我们将介绍可以用来保护您的环境的 Docker 工具。我们将介绍命令行工具和 GUI 工具，您可以利用这些工具来帮助您。本章将涵盖以下内容：

+   Docker 工具

+   在您的环境中使用 TLS，以确保各个部分之间的安全通信

+   使用只读容器来帮助保护容器中的数据免受某种形式的操纵

+   Docker 安全基础知识

+   内核命名空间

+   控制组

+   Linux 内核功能

# Docker 工具

在本节中，我们将介绍可以帮助您保护 Docker 环境的工具。这些是内置在您已经使用的 Docker 软件中的选项。现在是时候学习如何启用或利用这些功能，以确保通信安全；在这里，我们将介绍启用 TLS，这是一种确保应用程序之间隐私的协议。它确保没有人在监听通信。可以将其视为观看电影时人们在电话中说“这条线路安全吗？”的情景。当涉及网络通信时，这是同样的想法。然后，我们将看看如何利用只读容器来确保您提供的数据不会被任何人操纵。

## 使用 TLS

强烈建议使用 Docker Machine 来创建和管理 Docker 主机。它将自动设置通信以使用 TLS。以下是您可以验证由`docker-machine`创建的*默认*主机是否确实使用 TLS 的方法。

一个重要因素是知道您是否在使用 TLS，然后根据实际情况调整使用 TLS。重要的是要记住，如今几乎所有的 Docker 工具都启用了 TLS，或者如果它们没有启用，它们似乎正在朝着这个目标努力。您可以使用 Docker Machine `inspect`命令来检查您的 Docker 主机是否使用了 TLS。接下来，我们将查看一个主机，并查看它是否启用了 TLS：

```
**docker-machine inspect default**

**{**
 **"ConfigVersion": 3,**
 **"Driver": {**
 **"IPAddress": "192.168.99.100",**
 **"MachineName": "default",**
 **"SSHUser": "docker",**
 **"SSHPort": 50858,**
 **"SSHKeyPath": "/Users/scottgallagher/.docker/machine/machines/default/id_rsa",**
 **"StorePath": "/Users/scottgallagher/.docker/machine",**
 **"SwarmMaster": false,**
 **"SwarmHost": "tcp://0.0.0.0:3376",**
 **"SwarmDiscovery": "",**
 **"VBoxManager": {},**
 **"CPU": 1,**
 **"Memory": 2048,**
 **"DiskSize": 204800,**
 **"Boot2DockerURL": "",**
 **"Boot2DockerImportVM": "",**
 **"HostDNSResolver": false,**
 **"HostOnlyCIDR": "192.168.99.1/24",**
 **"HostOnlyNicType": "82540EM",**
 **"HostOnlyPromiscMode": "deny",**
 **"NoShare": false,**
 **"DNSProxy": false,**
 **"NoVTXCheck": false**
 **},**
 **"DriverName": "virtualbox",**
 **"HostOptions": {**
 **"Driver": "",**
 **"Memory": 0,**
 **"Disk": 0,**
 **"EngineOptions": {**
 **"ArbitraryFlags": [],**
 **"Dns": null,**
 **"GraphDir": "",**
 **"Env": [],**
 **"Ipv6": false,**
 **"InsecureRegistry": [],**
 **"Labels": [],**
 **"LogLevel": "",**
 **"StorageDriver": "",**
 **"SelinuxEnabled": false,**
 **"TlsVerify": true,**
 **"RegistryMirror": [],**
 **"InstallURL": "https://get.docker.com"**
 **},**
 **"SwarmOptions": {**
 **"IsSwarm": false,**
 **"Address": "",**
 **"Discovery": "",**
 **"Master": false,**
 **"Host": "tcp://0.0.0.0:3376",**
 **"Image": "swarm:latest",**
 **"Strategy": "spread",**
 **"Heartbeat": 0,**
 **"Overcommit": 0,**
 **"ArbitraryFlags": [],**
 **"Env": null**
 **},**
 **"AuthOptions": {**
 **"CertDir": "/Users/scottgallagher/.docker/machine/certs",**
 **"CaCertPath": "/Users/scottgallagher/.docker/machine/certs/ca.pem",**
 **"CaPrivateKeyPath": "/Users/scottgallagher/.docker/machine/certs/ca-key.pem",**
 **"CaCertRemotePath": "",**
 **"ServerCertPath": "/Users/scottgallagher/.docker/machine/machines/default/server.pem",**
 **"ServerKeyPath": "/Users/scottgallagher/.docker/machine/machines/default/server-key.pem",**
 **"ClientKeyPath": "/Users/scottgallagher/.docker/machine/certs/key.pem",**
 **"ServerCertRemotePath": "",**
 **"ServerKeyRemotePath": "",**
 **"ClientCertPath": "/Users/scottgallagher/.docker/machine/certs/cert.pem",**
 **"ServerCertSANs": [],**
 **"StorePath": "/Users/scottgallagher/.docker/machine/machines/default"**
 **}**
 **},**
 **"Name": "default"**
**}**

```

从前面的输出中，我们可以关注以下行：

```
 **"SwarmHost": "tcp://0.0.0.0:3376",**

```

这向我们表明，如果我们正在运行**Swarm**，这个主机将利用安全的`3376`端口。现在，如果你没有使用 Docker Swarm，那么你可以忽略这一行。但是，如果你正在使用 Docker Swarm，那么这一行就很重要。

回过头来，让我们先确定一下 Docker Swarm 是什么。Docker Swarm 是 Docker 内部的原生集群。它有助于将多个 Docker 主机转变为易于管理的单个虚拟主机：

```
 **"AuthOptions": {**
 **"CertDir": "/Users/scottgallagher/.docker/machine/certs",**
 **"CaCertPath": "/Users/scottgallagher/.docker/machine/certs/ca.pem",**
 **"CaPrivateKeyPath": "/Users/scottgallagher/.docker/machine/certs/ca-key.pem",**
 **"CaCertRemotePath": "",**
 **"ServerCertPath": "/Users/scottgallagher/.docker/machine/machines/default/server.pem",**
 **"ServerKeyPath": "/Users/scottgallagher/.docker/machine/machines/default/server-key.pem",**
 **"ClientKeyPath": "/Users/scottgallagher/.docker/machine/certs/key.pem",**
 **"ServerCertRemotePath": "",**
 **"ServerKeyRemotePath": "",**
 **"ClientCertPath": "/Users/scottgallagher/.docker/machine/certs/cert.pem",**
 **"ServerCertSANs": [],**
 **"StorePath": "/Users/scottgallagher/.docker/machine/machines/default"**
 **}**

```

这向我们表明这个主机实际上正在使用证书，所以我们知道它正在使用 TLS，但仅凭此如何知道呢？在接下来的部分，我们将看看如何确切地知道它是否正在使用 TLS。

Docker Machine 还有一个选项，可以通过 TLS 运行所有内容。这是使用 Docker Machine 管理 Docker 主机的最安全方式。如果你开始使用自己的证书，这种设置可能会有些棘手。默认情况下，Docker Machine 会将它使用的证书存储在`/Users/<user_id>/.docker/machine/certs/`目录下。你可以从前面的输出中看到证书存储在你的机器上的位置。

让我们看看如何实现查看我们的 Docker 主机是否使用 TLS 的目标：

```
**docker-machine ls**
**NAME      ACTIVE   URL          STATE     URL SWARM   DOCKER   ERRORS**
**default   *        virtualbox   Running   tcp://192.168.99.100:2376  v1.9.1** 

```

这就是我们可以知道它正在使用 TLS 的地方。Docker Machine 主机的不安全端口是`2375`端口，而这个主机使用的是`2376`，这是 Docker Machine 的安全 TLS 端口。因此，这个主机实际上正在使用 TLS 进行通信，这让你放心知道通信是安全的。

## 只读容器

关于`docker run`命令，我们主要关注的是允许我们将容器内的所有内容设置为只读的选项。让我们看一个例子，并分解它到底是做什么的：

```
**$ docker run --name mysql --read-only -v /var/lib/mysql v /tmp --e MYSQL_ROOT_PASSWORD=password -d mysql**

```

在这里，我们正在运行一个`mysql`容器，并将整个容器设置为只读，除了`/var/lib/mysql`目录。这意味着容器内唯一可以写入数据的位置是`/var/lib/mysql`目录。容器内的任何其他位置都不允许你在其中写入任何内容。如果你尝试运行以下命令，它会失败：

```
**$ docker exec mysql touch /opt/filename**

```

如果你想控制容器可以写入或不写入的位置，这将非常有帮助。一定要明智地使用它。进行彻底测试，因为当应用程序无法写入某些位置时可能会产生后果。

还记得我们在之前章节中看到的 Docker 卷吗？我们能够将卷设置为只读。类似于之前的`docker run`命令，我们将所有内容设置为只读，除了指定的卷，现在我们可以做相反的操作，将单个卷（或者如果你使用更多的`-v`开关，可以是多个卷）设置为只读。关于卷需要记住的一点是，当你使用一个卷并将其挂载到容器中时，它将作为空卷挂载到容器内的目录顶部，除非你使用`--volumes-from`开关或在事后以其他方式向容器添加数据：

```
**$ docker run -d -v /opt/uploads:/opt/uploads:/opt/uploads:ro nginx**

```

这将在`/opt/uploads`中挂载一个卷，并将其设置为只读。如果你不希望运行的容器写入卷以保持数据或配置文件的完整性，这可能会很有用。

关于`docker run`命令，我们要看的最后一个选项是`--device=`开关。这个开关允许我们将 Docker 主机上的设备挂载到容器内的指定位置。在这样做时，我们需要意识到一些安全风险。默认情况下，当你这样做时，容器将获得对设备位置的完全访问权限：读、写和`mknod`访问。现在，你可以通过在开关命令的末尾操纵`rwm`来控制这些权限。

让我们来看看其中一些，并了解它们是如何工作的：

```
**$ docker run --device=/dev/sdb:/dev/sdc2 -it ubuntu:latest /bin/bash**

```

之前的命令将运行最新的 Ubuntu 镜像，并将`/dev/sdb`设备挂载到容器内的`/dev/sdc2`位置：

```
**$ docker run --device=/dev/sdb:/dev/sdc2:r -it ubuntu:latest /bin/bash**

```

这个命令将运行最新的 Ubuntu 镜像，并将`/dev/sdb1`设备挂载到容器内的`/dev/sdc2`位置。然而，这个命令的末尾有一个`:r`标签，指定它是只读的，不能被写入。

# Docker 安全基础知识

在前面的章节中，我们研究了一些你可以使用的 Docker 工具，比如用于通信的 TLS，以及使用只读容器来确保数据不被更改或操纵。在本节中，我们将重点介绍 Docker 生态系统中提供的一些更多选项，可以用来帮助加强你的环境安全性。我们将看一下内核命名空间，它提供了另一层抽象，通过为运行的进程提供自己的资源，这些资源只对进程本身可见，而对其他可能正在运行的进程不可见。我们将在本节中更多地了解内核命名空间。然后我们将看一下控制组。控制组，更常被称为 cgroups，让你能够限制特定进程所拥有的资源。然后我们将介绍 Linux 内核功能。通过这个，我们将看一下在使用 Docker 运行时，默认情况下对容器施加的限制。最后，我们将看一下 Docker 守护程序的攻击面，需要注意的 Docker 守护程序存在的风险，以及如何减轻这些风险。

## 内核命名空间

内核命名空间为容器提供了一种隔离形式。可以把它们看作是一个容器包裹在另一个容器中。在一个容器中运行的进程不能干扰另一个容器内运行的进程，更不用说在容器所在的 Docker 主机上运行了。它的工作方式是，每个容器都有自己的网络堆栈来操作。然而，有办法将这些容器链接在一起，以便能够相互交互；然而，默认情况下，它们是相互隔离的。内核命名空间已经存在了相当长的时间，所以它们是一种经过验证的隔离保护方法。它们在 2008 年被引入，而在撰写本书时，已经是 2016 年了。你可以看到，到了今年 7 月，它们将满八岁。因此，当你发出`docker run`命令时，你正在受益于后台进行的大量工作。这些工作正在创建自己的网络堆栈来操作。这也使得容器免受其他容器能够操纵容器的运行进程或数据的影响。

## 控制组

控制组，或更常见的称为 cgroups，是 Linux 内核的一个功能，允许您限制容器可以使用的资源。虽然它们限制资源，但它们也确保每个容器获得它所需的资源，以及没有单个容器可以使整个 Docker 主机崩溃。

使用控制组，您可以限制特定容器获得的 CPU、内存或磁盘 I/O 的数量。如果我们查看`docker run`命令的帮助，让我们突出显示我们可以控制的项目。我们只会突出显示一些对大多数用户特别有用的项目，但请查看它们，看看是否有其他项目适合您的环境，如下所示：

```
**$ docker run --help** 

**Usage: docker run [OPTIONS] IMAGE [COMMAND] [ARG...]**

**Run a command in a new container**

 **-a, --attach=[]                 Attach to STDIN, STDOUT or STDERR**
 **--add-host=[]                   Add a custom host-to-IP mapping (host:ip)**
 **--blkio-weight=0                Block IO (relative weight), between 10 and 1000**
 **--cpu-shares=0                  CPU shares (relative weight)**
 **--cap-add=[]                    Add Linux capabilities**
 **--cap-drop=[]                   Drop Linux capabilities**
 **--cgroup-parent=                Optional parent cgroup for the container**
 **--cidfile=                      Write the container ID to the file**
 **--cpu-period=0                  Limit CPU CFS (Completely Fair Scheduler) period**
 **--cpu-quota=0                   Limit CPU CFS (Completely Fair Scheduler) quota**
 **--cpuset-cpus=                  CPUs in which to allow execution (0-3, 0,1)**
 **--cpuset-mems=                  MEMs in which to allow execution (0-3, 0,1)**
 **-d, --detach=false              Run container in background and print container ID**
 **--device=[]                     Add a host device to the container**
 **--disable-content-trust=true    Skip image verification**
 **--dns=[]                        Set custom DNS servers**
 **--dns-opt=[]                    Set DNS options**
 **--dns-search=[]                 Set custom DNS search domains**
 **-e, --env=[]                    Set environment variables**
 **--entrypoint=                   Overwrite the default ENTRYPOINT of the image**
 **--env-file=[]                   Read in a file of environment variables**
 **--expose=[]                     Expose a port or a range of ports**
 **--group-add=[]                  Add additional groups to join**
 **-h, --hostname=                 Container host name**
 **--help=false                    Print usage**
 **-i, --interactive=false         Keep STDIN open even if not attached**
 **--ipc=                          IPC namespace to use**
 **--kernel-memory=                Kernel memory limit**
 **-l, --label=[]                  Set meta data on a container**
 **--label-file=[]                 Read in a line delimited file of labels**
 **--link=[]                       Add link to another container**
 **--log-driver=                   Logging driver for container**
 **--log-opt=[]                    Log driver options**
 **--lxc-conf=[]                   Add custom lxc options**
 **-m, --memory=                   Memory limit**
 **--mac-address=                  Container MAC address (e.g. 92:d0:c6:0a:29:33)**
 **--memory-reservation=           Memory soft limit**
 **--memory-swap=                  Total memory (memory + swap), '-1' to disable swap**
 **--memory-swappiness=-1          Tuning container memory swappiness (0 to 100)**
 **--name=                         Assign a name to the container**
 **--net=default                   Set the Network for the container**
 **--oom-kill-disable=false        Disable OOM Killer**
 **-P, --publish-all=false         Publish all exposed ports to random ports**
 **-p, --publish=[]                Publish a container's port(s) to the host**
 **--pid=                          PID namespace to use**
 **--privileged=false              Give extended privileges to this container**
 **--read-only=false               Mount the container's root filesystem as read only**
 **--restart=no                    Restart policy to apply when a container exits**
 **--rm=false                      Automatically remove the container when it exits**
 **--security-opt=[]               Security Options**
 **--sig-proxy=true                Proxy received signals to the process**
 **--stop-signal=SIGTERM           Signal to stop a container, SIGTERM by default**
 **-t, --tty=false                 Allocate a pseudo-TTY**
 **-u, --user=                     Username or UID (format: <name|uid>[:<group|gid>])**
 **--ulimit=[]                     Ulimit options**
 **--uts=                          UTS namespace to use**
 **-v, --volume=[]                 Bind mount a volume**
 **--volume-driver=                Optional volume driver for the container**
 **--volumes-from=[]               Mount volumes from the specified container(s)**
 **-w, --workdir=                  Working directory inside the container**

```

正如您可以从前面突出显示的部分看到的，这些只是您可以在每个容器基础上控制的一些项目。

## Linux 内核功能

Docker 使用内核功能来放置 Docker 在启动或启动容器时放置的限制。限制根访问是这些内核功能的最终目标。有一些通常以 root 身份运行的服务，但现在可以在没有这些权限的情况下运行。其中一些包括`SSH`、`cron`和`syslogd`。

总的来说，这意味着您不需要像通常想的那样在服务器上拥有 root 权限。您可以以降低的容量集运行。这意味着您的 root 用户不需要通常需要的特权。

您可能不再需要启用的一些项目如下所示：

+   执行挂载操作

+   使用原始套接字，这将有助于防止数据包欺骗

+   创建新设备

+   更改文件的所有者

+   更改属性

这有助于防止如果有人破坏了一个容器，那么他们无法提升到您提供的更高权限。要从运行的容器提升到运行的 Docker 主机，将会更加困难，甚至不可能。由于这种复杂性，攻击者可能会选择其他地方而不是您的 Docker 环境来尝试攻击。Docker 还支持添加和删除功能，因此建议删除所有功能，除了您打算使用的功能。例如，可以在`docker run`命令上使用`-cap-add net_bind_service`开关。

# 容器与虚拟机

希望您信任您的组织和所有可以访问这些系统的人。您很可能会从头开始设置虚拟机。由于其庞大的体积，很可能无法从他人那里获取虚拟机。因此，您将了解虚拟机内部的情况。也就是说，对于 Docker 容器，您可能不知道您可能在容器中使用的镜像中有什么。

# 总结

在本章中，我们研究了将 TLS 部署到我们 Docker 环境的所有部分，以便确保所有通信都是安全的，流量不能被拦截和解释。我们还了解了如何利用只读容器来确保提供的数据不能被篡改。然后，我们看了如何为进程提供它们自己的抽象，比如网络、挂载、用户等。接着，我们深入了解了控制组，或者更常见的称为 cgroups，作为限制进程或容器资源的一种方式。我们还研究了 Linux 内核的功能，即在启动或启动容器时施加的限制。最后，我们深入了解了如何减轻针对 Docker 守护程序攻击面的风险。

在下一章中，我们将研究使用第三方工具保护 Docker，并了解除 Docker 提供的工具之外，还有哪些第三方工具可以帮助您保护环境，以确保在 Docker 上运行时保持应用程序的安全。

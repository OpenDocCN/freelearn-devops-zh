# 第一章：保护 Docker 主机

欢迎来到*Securing Docker*书籍！我们很高兴您决定阅读这本书，我们希望确保您使用的资源得到适当的保护，以确保系统完整性和数据丢失预防。了解为什么您应该关心安全性也很重要。如果数据丢失预防还没有吓到您，那么考虑最坏的情况——整个系统被攻破，您的机密设计可能被泄露或被他人窃取——可能有助于加强安全性。在本书中，我们将涵盖许多主题，以帮助您安全地设置环境，以便您可以放心地开始部署容器，知道您在开始时采取了正确的步骤来加强您的环境。在本章中，我们将着眼于保护 Docker 主机，并将涵盖以下主题：

+   Docker 主机概述

+   讨论 Docker 主机

+   虚拟化和隔离

+   Docker 守护程序的攻击面

+   保护 Docker 主机

+   Docker Machine

+   SELinux 和 AppArmor

+   自动修补主机

# Docker 主机概述

在我们深入研究之前，让我们先退一步，确切地了解 Docker 主机是什么。在本节中，我们将查看 Docker 主机本身，以了解我们在谈论 Docker 主机时指的是什么。我们还将研究 Docker 使用的虚拟化和隔离技术，以确保安全性。

# 讨论 Docker 主机

当我们想到 Docker 主机时，我们会想到什么？如果用我们几乎都熟悉的虚拟机来说，让我们看看典型的 VM 主机与 Docker 主机有何不同。**VM 主机**是虚拟机实际运行在其上的地方。通常情况下，如果您使用 VMware，则是**VMware ESXi**，如果您使用**Hyper-V**，则是**Windows Server**。让我们来看看它们的比较，以便您可以对两者有一个视觉上的表示，如下图所示：

![讨论 Docker 主机](img/00002.jpeg)

上图描述了 VM 主机和 Docker 主机之间的相似之处。如前所述，任何服务的主机只是底层虚拟机或 Docker 容器运行的系统。因此，主机是包含和操作您安装和设置服务的底层系统的操作系统或服务，例如 Web 服务器、数据库等。

# 虚拟化和隔离

了解如何保护 Docker 主机之前，我们必须首先了解 Docker 主机是如何设置以及 Docker 主机中包含哪些项目。与 VM 主机一样，它们包含底层服务运行的操作系统。对于 VM，您正在在 VM 主机操作系统之上创建一个全新的操作系统。然而，在 Docker 上，您并没有这样做，而是共享 Docker 主机正在使用的 Linux 内核。让我们看一下以下图表，以帮助我们表示这一点：

![虚拟化和隔离](img/00003.jpeg)

从上图中可以看出，VM 主机和 Docker 主机上的项目设置方式有明显的区别。在 VM 主机上，每个虚拟机都有其自己的项目。每个容器化应用程序都带有自己的一套库，无论是 Windows 还是 Linux。现在，在 Docker 主机上，我们看不到这一点。我们看到它们共享 Docker 主机上正在使用的 Linux 内核版本。也就是说，Docker 主机方面需要解决一些安全方面的问题。现在，在 VM 主机方面，如果某个虚拟机受到了损害，操作系统只是隔离在那一个虚拟机中。回到 Docker 主机方面，如果 Docker 主机上的内核受到了损害，那么运行在该主机上的所有容器也会面临很高的风险。

因此，现在您应该明白了，当涉及到 Docker 主机时，我们专注于安全是多么重要。Docker 主机确实使用了一些隔离技术，可以在一定程度上保护免受内核或容器受损。其中两种方式是通过实施命名空间和 cgroups。在讨论它们如何帮助之前，让我们先给出它们的定义。

内核命名空间，通常被称为，为在主机上运行的容器提供一种隔离形式。这意味着什么？这意味着你在 Docker 主机上运行的每个容器都将被赋予自己的网络堆栈，以便它不会特权访问另一个容器的套接字或接口。然而，默认情况下，所有 Docker 容器都位于桥接接口上，以便它们可以轻松地相互通信。将桥接接口视为所有容器连接到的网络交换机。

命名空间还为进程和挂载点提供隔离。在一个容器中运行的进程不能影响或甚至看到在另一个 Docker 容器中运行的进程。挂载点的隔离也是基于每个容器的。这意味着一个容器上的挂载点不能看到或与另一个容器上的挂载点进行交互。

另一方面，控制组是控制和限制将在 Docker 主机上运行的容器资源的工具。这归结为什么，意味着它将如何使你受益？这意味着 cgroups，它们将被称为，帮助每个容器获得其公平份额的内存磁盘 I/O，CPU 等等。因此，一个容器不能通过耗尽其上可用的所有资源来使整个主机崩溃。这将有助于确保即使一个应用程序表现不佳，其他容器也不会受到这个应用程序的影响，你的其他应用程序可以保证正常运行时间。

# Docker 守护程序的攻击面

虽然 Docker 确实简化了虚拟化领域的一些复杂工作，但很容易忘记考虑在 Docker 主机上运行容器的安全影响。您需要注意的最大问题是 Docker 需要 root 权限才能运行。因此，您需要知道谁可以访问您的 Docker 主机和 Docker 守护程序，因为他们将完全管理您 Docker 主机上的所有 Docker 容器和镜像。他们可以启动新容器，停止现有容器，删除镜像，拉取新镜像，甚至通过向其中注入命令来重新配置正在运行的容器。他们还可以从容器中提取密码和证书等敏感信息。因此，如果确实需要对谁可以访问您的 Docker 守护程序进行分开控制，还需要确保分开重要的容器。这适用于需要访问运行容器的 Docker 主机的人。如果用户需要 API 访问，则情况就不同了，可能不需要分开。例如，将敏感的容器保留在一个 Docker 主机上，而将正常运行的容器保留在另一个 Docker 主机上，并授予其他员工对非特权主机上的 Docker 守护程序的访问权限。如果可能的话，还建议取消将在主机上运行的容器的 setuid 和 setgid 功能。如果要运行 Docker，建议只在此服务器上使用 Docker 而不是其他应用程序。Docker 还以非常受限的功能集启动容器，这有利于解决安全问题。

### 注意

要在启动 Docker 容器时取消 setuid 或 setgid 功能，您需要做类似以下操作：

```
$ docker run -d --cap-drop SETGID --cap-drop SETUID nginx

```

这将启动`nginx`容器，并且会为容器取消`SETGID`和`SETUID`的功能。

Docker 的最终目标是将根用户映射到 Docker 主机上存在的非根用户。他们还致力于使 Docker 守护程序能够在不需要 root 权限的情况下运行。这些未来的改进将有助于简化 Docker 在实施其功能集时所需的关注度。

## 保护 Docker 守护程序

为了进一步保护 Docker 守护程序，我们可以保护 Docker 守护程序正在使用的通信。我们可以通过生成证书和密钥来实现这一点。在我们深入创建证书和密钥之前，有一些术语需要理解。**证书颁发机构**（**CA**）是颁发证书的实体。该证书证明了主体对公钥的所有权。通过这样做，我们可以确保您的 Docker 守护程序只接受由相同 CA 签署的证书的其他守护程序的通信。

现在，我们将看一下如何确保您在 Docker 主机上运行的容器将在接下来的几页中是安全的；然而，首先，您需要确保 Docker 守护程序在安全运行。为此，您需要在守护程序启动时启用一些参数。您需要预先准备一些东西，如下所示：

1.  创建 CA。

```
$ openssl genrsa -aes256 -out ca-key.pem 4096
Generating RSA private key, 4096 bit long modulus
......................................................................................................................................................................................................................++
....................................................................++
e is 65537 (0x10001)
Enter pass phrase for ca-key.pem:
Verifying - Enter pass phrase for ca-key.pem:

```

您需要指定两个值，`密码短语`和`密码短语`。这需要在`4`和`1023`个字符之间。少于`4`或多于`1023`的字符将不被接受。

```
$ openssl req -new -x509 -days <number_of_days> -key ca-key.pem -sha256 -out ca.pem
Enter pass phrase for ca-key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:Pennsylvania
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:
Email Address []:

```

您将需要一些项目。您需要之前输入的`密码短语`用于`ca-key.pem`。您还需要`国家`、`州`、`城市`、`组织名称`、`组织单位名称`、**完全限定域名**（**FQDN**）和`电子邮件地址`以完成证书。

1.  创建客户端密钥和签名证书。

```
$ openssl genrsa -out key.pem 4096
$ openssl req -subj '/CN=<client_DNS_name>' -new -key key.pem -out client.csr

```

1.  签署公钥。

```
$ openssl x509 -req -days <number_of_days> -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.em

```

1.  更改权限。

```
$ chmod -v 0400 ca-key.pem key.pem server-key.em
$ chmod -v 0444 ca.pem server-cert.pem cert.em
```

现在，您可以确保您的 Docker 守护程序只接受来自您提供签署证书的其他 Docker 主机的连接：

```
$ docker daemon --tlsverify --tlscacert=ca.pem --tlscert=server-certificate.pem --tlskey=server-key.pem -H=0.0.0.0:2376
```

确保证书文件位于您运行命令的目录中，否则您需要指定证书文件的完整路径。

在每个客户端上，您需要运行以下命令：

```
$ docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem -H=<$DOCKER_HOST>:2376 version

```

再次强调，证书的位置很重要。确保它们位于您计划运行前述命令的目录中，或者指定证书和密钥文件位置的完整路径。

您可以通过访问以下链接了解如何默认情况下在 Docker 守护程序中使用**传输层安全**（**TLS**）：

[`docs.docker.com/engine/articles/https/`](http://docs.docker.com/engine/articles/https/)

有关**Docker 安全部署指南**的更多阅读，以下链接提供了一个表格，可以用来深入了解您还可以利用的其他一些项目：

[`github.com/GDSSecurity/Docker-Secure-Deployment-Guidelines`](https://github.com/GDSSecurity/Docker-Secure-Deployment-Guidelines)

该网站的一些亮点包括：

+   收集安全和审计日志

+   在运行 Docker 容器时使用特权开关

+   设备控制组

+   挂载点

+   安全审计

# 保护 Docker 主机

我们从哪里开始保护我们的主机？我们需要从哪些工具开始？我们将在本节中使用 Docker Machine，以及如何确保我们创建的主机是以安全的方式创建的。Docker 主机就像您房子的前门，如果您没有适当地保护它们，任何人都可以随意进入。我们还将看看**安全增强型 Linux**（**SELinux**）和**AppArmor**，以确保您在创建的主机上有额外的安全层。最后，我们将看看一些支持并在发现安全漏洞时自动修补其操作系统的操作系统。

# Docker Machine

Docker Machine 是一种工具，允许您将 Docker 守护程序安装到您的虚拟主机上。然后，您可以使用 Docker Machine 管理这些 Docker 主机。Docker Machine 可以通过 Windows 和 Mac 上的**Docker 工具箱**安装。如果您使用 Linux，则可以通过简单的 `curl` 命令安装 Docker Machine：

```
$ curl -L https://github.com/docker/machine/releases/download/v0.6.0/docker-machine-`uname -s`-`uname -m` > /usr/local/bin/docker-machine && \
$ chmod +x /usr/local/bin/docker-machine

```

第一条命令将 Docker Machine 安装到 `/usr/local/bin` 目录中，第二条命令更改文件的权限并将其设置为可执行文件。

我们将在以下演练中使用 Docker Machine 来设置新的 Docker 主机。

Docker Machine 是您应该或将要使用来设置您的主机的工具。因此，我们将从它开始，以确保您的主机以安全的方式设置。我们将看看您如何在使用 Docker Machine 工具创建主机时，如何判断您的主机是否安全。让我们看看使用 Docker Machine 创建 Docker 主机时的情况，如下：

```
$ docker-machine create --driver virtualbox host1

Running pre-create checks...
Creating machine...
Waiting for machine to be running, this may take a few minutes...
Machine is running, waiting for SSH to be available...
Detecting operating system of created instance...
Provisioning created instance...
Copying certs to the local machine directory...
Copying certs to the remote machine...

Setting Docker configuration on the remote daemon...

```

从前面的输出中，随着创建的运行，Docker Machine 正在执行诸如创建机器、等待 SSH 可用、执行操作、将证书复制到正确位置以及设置 Docker 配置等操作，我们将看到如何连接 Docker 到这台机器，如下所示：

```
$ docker-machine env host1

export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="/Users/scottpgallagher/.docker/machine/machines/host1"
export DOCKER_MACHINE_NAME="host1"
# Run this command to configure your shell:
# eval "$(docker-machine env host1)"

```

前面的命令输出显示了设置此机器为 Docker 命令将要运行的机器所需运行的命令：

```
eval "$(docker-machine env host1)"

```

现在我们可以运行常规的 Docker 命令，比如`docker info`，它将从`host1`返回信息，现在我们已经将其设置为我们的环境。

我们可以从前面突出显示的输出中看到，主机从两个导出行开始就被安全地设置了。以下是第一个突出显示的行：

```
export DOCKER_TLS_VERIFY="1"

```

从其他突出显示的输出中，`DOCKER_TLS_VERIFY`被设置为`1`或`true`。以下是第二个突出显示的行：

```
export DOCKER_HOST="tcp://192.168.99.100:2376"

```

我们将主机设置为在安全端口`2376`上运行，而不是在不安全端口`2375`上运行。

我们还可以通过运行以下命令来获取这些信息：

```
$ docker-machine ls
NAME      ACTIVE   DRIVER       STATE     URL                         SWARM
host1              *        virtualbox     Running   tcp://192.168.99.100:2376

```

如果您已经使用先前的说明来设置 Docker 主机和 Docker 容器使用 TLS，请确保检查可以与 Docker Machine 一起使用的 TLS 开关选项。如果您有现有的证书要使用，这些开关将非常有用。通过运行以下命令，可以在突出显示的部分找到这些开关：

```
$ docker-machine --help

Options:
--debug, -D      Enable debug mode
-s, --storage-path "/Users/scottpgallagher/.docker/machine"
Configures storage path [$MACHINE_STORAGE_PATH]
--tls-ca-cert      CA to verify remotes against [$MACHINE_TLS_CA_CERT]
--tls-ca-key      Private key to generate certificates [$MACHINE_TLS_CA_KEY]
--tls-client-cert     Client cert to use for TLS [$MACHINE_TLS_CLIENT_CERT]
--tls-client-key       Private key used in client TLS auth [$MACHINE_TLS_CLIENT_KEY]
--github-api-token     Token to use for requests to the Github API [$MACHINE_GITHUB_API_TOKEN]
--native-ssh      Use the native (Go-based) SSH implementation. [$MACHINE_NATIVE_SSH]
--help, -h      show help
--version, -v      print the version

```

如果您想要安心，或者您的密钥确实被 compromise，您也可以使用`regenerate-certs`子命令重新生成机器的 TLS 证书。一个示例命令看起来类似于以下命令：

```
$ docker-machine regenerate-certs host1

Regenerate TLS machine certs?  Warning: this is irreversible. (y/n): y
Regenerating TLS certificates
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...

```

# SELinux 和 AppArmor

大多数 Linux 操作系统都基于它们可以利用 SELinux 或 AppArmor 来实现对操作系统上的文件或位置的更高级访问控制。使用这些组件，您可以限制容器以 root 用户的权限执行程序的能力。

Docker 确实提供了一个安全模型模板，其中包括 AppArmor，Red Hat 也为 Docker 提供了 SELinux 策略。您可以利用这些提供的模板在您的环境中添加额外的安全层。

有关 SELinux 和 Docker 的更多信息，我建议访问以下网站：

[`www.mankier.com/8/docker_selinux`](https://www.mankier.com/8/docker_selinux)

另一方面，如果你正在寻找关于 AppArmor 和 Docker 的更多阅读材料，我建议访问以下网站：

[`github.com/docker/docker/tree/master/contrib/apparmor`](https://github.com/docker/docker/tree/master/contrib/apparmor)

在这里，您将找到一个`template.go`文件，这是 Docker 随其应用程序一起提供的 AppArmor 模板。

# 自动修补主机

如果您真的想深入了解高级 Docker 主机，那么您可以使用 CoreOS 和 Amazon Linux AMI，它们都以不同的方式进行自动修补。CoreOS 将在安全更新发布时对您的操作系统进行修补并重新启动您的操作系统，而 Amazon Linux AMI 将在您重新启动时完成更新。因此，在设置 Docker 主机时选择要使用的操作系统时，请确保考虑到这两种操作系统都以不同的方式实现了某种形式的自动修补。您将希望确保实施某种类型的扩展或故障转移来满足在 CoreOS 上运行时的需求，以便在重新启动以修补操作系统时不会出现停机时间。

# 总结

在本章中，我们看了如何保护我们的 Docker 主机。Docker 主机是第一道防线，因为它们是您的容器将运行和相互通信以及最终用户的起点。如果这些不安全，那么继续进行其他任何事情就没有意义。您学会了如何设置 Docker 守护程序以通过为主机和客户端生成适当的证书来安全运行 TLS。我们还看了使用 Docker 容器的虚拟化和隔离优势，但请记住 Docker 守护程序的攻击面。

其他内容包括如何使用 Docker Machine 在安全操作系统上轻松创建 Docker 主机，并确保在设置容器时使用安全方法。使用 SELinux 和 AppArmor 等项目也有助于改善您的安全性。最后，我们还介绍了一些 Docker 主机操作系统，您也可以使用自动修补，例如 CoreOS 和 Amazon Linux AMI。

在下一章中，我们将研究如何保护 Docker 的组件。我们将重点关注保护 Docker 的组件，比如您可以使用的注册表、运行在您主机上的容器，以及如何对您的镜像进行签名。

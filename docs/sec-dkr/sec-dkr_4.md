# 第四章：安全的 Docker Bench

在本章中，我们将看一下**Docker 安全基准**。这是一个工具，可用于扫描您的 Docker 环境，从主机级别开始检查所有这些主机的各个方面，检查 Docker 守护程序及其配置，检查在 Docker 主机上运行的容器，并审查 Docker 安全操作，并为您提供跨越威胁或关注的建议，您可能希望查看以解决它。在本章中，我们将看以下项目：

+   **Docker 安全** - 最佳实践

+   **Docker** - 最佳实践

+   **互联网安全中心**（**CIS**）指南

+   主机配置

+   Docker 守护程序配置

+   Docker 守护程序配置文件

+   容器镜像/运行时

+   Docker 安全操作

+   Docker Bench 安全应用程序

+   运行工具

+   理解输出

# Docker 安全 - 最佳实践

在本节中，我们将介绍使用 Docker 的最佳实践以及 CIS 指南，以正确保护 Docker 环境的所有方面。当您实际运行扫描（在本章的下一节）并获得需要或应该修复的结果时，您将参考此指南。该指南分为以下几个部分：

+   主机配置

+   Docker 守护程序配置

+   Docker 守护程序配置文件

+   容器镜像/运行时

+   Docker 安全操作

# Docker - 最佳实践

在我们深入研究 CIS 指南之前，让我们回顾一些在使用 Docker 时的最佳实践：

+   **每个容器一个应用程序**：将您的应用程序分布到每个容器中。Docker 就是为此而构建的，这样做会让一切变得更加简单。我们之前谈到的隔离就是关键所在。

+   **审查谁可以访问您的 Docker 主机**：请记住，谁拥有访问您的 Docker 主机的权限，谁就可以操纵主机上的所有镜像和容器。

+   **使用最新版本**：始终使用最新版本的 Docker。这将确保所有安全漏洞都已修补，并且您也拥有最新的功能。

+   **使用资源**：如果需要帮助，请使用可用资源。Docker 社区庞大而乐于助人。充分利用他们的网站、文档和**Internet Relay Chat**（**IRC**）聊天室。

# CIS 指南

CIS 指南是一份文件（[`benchmarks.cisecurity.org/tools2/docker/cis_docker_1.6_benchmark_v1.0.0.pdf`](https://benchmarks.cisecurity.org/tools2/docker/cis_docker_1.6_benchmark_v1.0.0.pdf)），它涵盖了 Docker 部件的各个方面，以帮助您安全地配置 Docker 环境的各个部分。我们将在以下部分进行介绍。

## 主机配置

本指南的这一部分涉及配置您的 Docker 主机。这是 Docker 环境中所有容器运行的部分。因此，保持其安全性至关重要。这是对抗攻击者的第一道防线。

## Docker 守护程序配置

本指南的这一部分建议保护运行中的 Docker 守护程序。您对 Docker 守护程序配置所做的每一项更改都会影响每个容器。这些是您可以附加到 Docker 守护程序的开关，我们之前看到的，以及在我们运行工具时将在以下部分看到的项目。

## Docker 守护程序配置文件

本指南的这一部分涉及 Docker 守护程序使用的文件和目录。这涵盖了从权限到所有权的范围。有时，这些区域可能包含您不希望他人知道的信息，这些信息可能以纯文本格式存在。

## 容器镜像/运行时

本指南的这一部分包含了保护容器镜像以及容器运行时的信息。

第一部分包含镜像，涵盖了基础镜像和使用的构建文件。您需要确保您使用的镜像不仅适用于基础镜像，还适用于 Docker 体验的任何方面。本指南的这一部分涵盖了在创建自己的基础镜像时应遵循的项目，以确保它们是安全的。

第二部分，容器运行时，涵盖了许多与安全相关的项目。您必须注意您提供的运行时变量。在某些情况下，攻击者可以利用它们来获得优势，而您可能认为您正在利用它们来获得自己的优势。在您的容器中暴露太多内容可能会危及不仅该容器的安全性，还会危及 Docker 主机和在该主机上运行的其他容器的安全性。

## Docker 安全操作

本指南的这一部分涵盖了涉及部署的安全领域。这些项目与最佳实践和建议更密切地相关，这些项目应该遵循。

# Docker Bench Security 应用程序

在本节中，我们将介绍 Docker Benchmark Security 应用程序，您可以安装和运行该工具。该工具将检查以下组件：

+   主机配置

+   Docker 守护进程配置

+   Docker 守护进程配置文件

+   容器镜像和构建文件

+   容器运行时

+   Docker 安全操作

看起来很熟悉？应该是的，因为这些是我们在上一节中审查的相同项目，只是内置到一个应用程序中，它将为您做很多繁重的工作。它将向您显示配置中出现的警告，并提供有关其他配置项甚至通过测试的项目的信息。

我们将看看如何运行该工具，一个实时例子，以及进程输出的含义。

## 运行工具

运行该工具很简单。它已经被打包到一个 Docker 容器中。虽然您可以获取源代码并自定义输出或以某种方式操纵它（比如，通过电子邮件输出），但默认情况可能是您所需要的一切。

代码在这里找到：[`github.com/docker/docker-bench-security`](https://github.com/docker/docker-bench-security)

要运行该工具，我们只需将以下内容复制并粘贴到我们的 Docker 主机中：

```
$ docker run -it --net host --pid host --cap-add audit_control \
-v /var/lib:/var/lib \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /usr/lib/systemd:/usr/lib/systemd \
-v /etc:/etc --label docker_bench_security \
docker/docker-bench-security

```

如果您还没有该镜像，它将首先下载该镜像，然后为您启动该进程。现在我们已经看到了安装和运行它有多么简单，让我们看一个 Docker 主机上的例子，看看它实际上做了什么。然后我们将查看输出并深入分析它。

还有一个选项可以克隆 Git 存储库，输入`git clone`命令的目录，并运行提供的 shell 脚本。所以，我们有多个选择！

让我们看一个例子，并分解每个部分，如下命令所示：

```
# ------------------------------------------------------------------------------
# Docker Bench for Security v1.0.0
#
# Docker, Inc. (c) 2015
#
# Checks for dozens of common best-practices around deploying Docker containers in production.
# Inspired by the CIS Docker 1.6 Benchmark:
# https://benchmarks.cisecurity.org/tools2/docker/CIS_Docker_1.6_Benchmark_v1.0.0.pdf
# ------------------------------------------------------------------------------

Initializing Sun Jan 17 19:18:56 UTC 2016

```

### 运行工具 - 主机配置

让我们看看主机配置运行时的输出：

```
[INFO] 1 - Host configuration
[WARN] 1.1  - Create a separate partition for containers
[PASS] 1.2  - Use an updated Linux Kernel
[PASS] 1.5  - Remove all non-essential services from the host - Network
[PASS] 1.6  - Keep Docker up to date
[INFO]       * Using 1.9.1 which is current as of 2015-11-09
[INFO]       * Check with your operating system vendor for support and security maintenance for docker
[INFO] 1.7  - Only allow trusted users to control Docker daemon
[INFO]      * docker:x:100:docker
[WARN] 1.8  - Failed to inspect: auditctl command not found.
[INFO] 1.9  - Audit Docker files and directories - /var/lib/docker
[INFO]      * Directory not found
[WARN] 1.10 - Failed to inspect: auditctl command not found.
[INFO] 1.11 - Audit Docker files and directories - docker-registry.service
[INFO]      * File not found
[INFO] 1.12 - Audit Docker files and directories - docker.service
[INFO]      * File not found
[WARN] 1.13 - Failed to inspect: auditctl command not found.
[INFO] 1.14 - Audit Docker files and directories - /etc/sysconfig/docker
[INFO]      * File not found
[INFO] 1.15 - Audit Docker files and directories - /etc/sysconfig/docker-network
[INFO]      * File not found
[INFO] 1.16 - Audit Docker files and directories - /etc/sysconfig/docker-registry
[INFO]      * File not found
[INFO] 1.17 - Audit Docker files and directories - /etc/sysconfig/docker-storage
[INFO]      * File not found
[INFO] 1.18 - Audit Docker files and directories - /etc/default/docker
[INFO]      * File not found

```

### 运行工具 - Docker 守护进程配置

让我们看看 Docker 守护进程配置运行时的输出，如下命令所示：

```
[INFO] 2 - Docker Daemon Configuration
[PASS] 2.1  - Do not use lxc execution driver
[WARN] 2.2  - Restrict network traffic between containers
[PASS] 2.3  - Set the logging level
[PASS] 2.4  - Allow Docker to make changes to iptables
[PASS] 2.5  - Do not use insecure registries
[INFO] 2.6  - Setup a local registry mirror
[INFO]      * No local registry currently configured
[WARN] 2.7  - Do not use the aufs storage driver
[PASS] 2.8  - Do not bind Docker to another IP/Port or a Unix socket
[INFO] 2.9  - Configure TLS authentication for Docker daemon
[INFO]      * Docker daemon not listening on TCP
[INFO] 2.10 - Set default ulimit as appropriate
[INFO]      * Default ulimit doesn't appear to be set

```

### 运行工具 - Docker 守护进程配置文件

让我们看看 Docker 守护进程配置文件运行时的输出，如下所示：

```
[INFO] 3 - Docker Daemon Configuration Files
[INFO] 3.1  - Verify that docker.service file ownership is set to root:root
[INFO]      * File not found
[INFO] 3.2  - Verify that docker.service file permissions are set to 644
[INFO]      * File not found
[INFO] 3.3  - Verify that docker-registry.service file ownership is set to root:root
[INFO]      * File not found
[INFO] 3.4  - Verify that docker-registry.service file permissions are set to 644
[INFO]      * File not found
[INFO] 3.5  - Verify that docker.socket file ownership is set to root:root
[INFO]      * File not found
[INFO] 3.6  - Verify that docker.socket file permissions are set to 644
[INFO]      * File not found
[INFO] 3.7  - Verify that Docker environment file ownership is set to root:root
[INFO]      * File not found
[INFO] 3.8  - Verify that Docker environment file permissions are set to 644
[INFO]      * File not found
[INFO] 3.9  - Verify that docker-network environment file ownership is set to root:root
[INFO]      * File not found
[INFO] 3.10 - Verify that docker-network environment file permissions are set to 644
[INFO]      * File not found
[INFO] 3.11 - Verify that docker-registry environment file ownership is set to root:root
[INFO]      * File not found
[INFO] 3.12 - Verify that docker-registry environment file permissions are set to 644
[INFO]      * File not found
[INFO] 3.13 - Verify that docker-storage environment file ownership is set to root:root
[INFO]      * File not found
[INFO] 3.14 - Verify that docker-storage environment file permissions are set to 644
[INFO]      * File not found
[PASS] 3.15 - Verify that /etc/docker directory ownership is set to root:root
[PASS] 3.16 - Verify that /etc/docker directory permissions are set to 755
[INFO] 3.17 - Verify that registry certificate file ownership is set to root:root
[INFO]      * Directory not found
[INFO] 3.18 - Verify that registry certificate file permissions are set to 444
[INFO]      * Directory not found
[INFO] 3.19 - Verify that TLS CA certificate file ownership is set to root:root
[INFO]      * No TLS CA certificate found
[INFO] 3.20 - Verify that TLS CA certificate file permissions are set to 444
[INFO]      * No TLS CA certificate found
[INFO] 3.21 - Verify that Docker server certificate file ownership is set to root:root
[INFO]      * No TLS Server certificate found
[INFO] 3.22 - Verify that Docker server certificate file permissions are set to 444
[INFO]      * No TLS Server certificate found
[INFO] 3.23 - Verify that Docker server key file ownership is set to root:root
[INFO]      * No TLS Key found
[INFO] 3.24 - Verify that Docker server key file permissions are set to 400
[INFO]      * No TLS Key found
[PASS] 3.25 - Verify that Docker socket file ownership is set to root:docker
[PASS] 3.26 - Verify that Docker socket file permissions are set to 660

```

### 运行工具 - 容器镜像和构建文件

让我们看看容器镜像和构建文件运行时的输出，如下命令所示：

```
[INFO] 4 - Container Images and Build Files
[INFO] 4.1  - Create a user for the container
[INFO]      * No containers running

```

### 运行工具 - 容器运行时

让我们看看容器运行时的输出，如下所示：

```
[INFO] 5  - Container Runtime
[INFO]      * No containers running, skipping Section 5

```

### 运行工具 - Docker 安全操作

让我们来看一下 Docker 安全操作运行时的输出，如下命令所示：

```
[INFO] 6  - Docker Security Operations
[INFO] 6.5 - Use a centralized and remote log collection service
[INFO]      * No containers running
[INFO] 6.6 - Avoid image sprawl
[INFO]      * There are currently: 23 images
[WARN] 6.7 - Avoid container sprawl
[WARN]      * There are currently a total of 51 containers, with only 1 of them currently running

```

哇！大量的输出和大量的内容需要消化；但这一切意味着什么？让我们来看看并分解每个部分。

## 理解输出

我们将看到三种类型的输出，如下所示：

+   【通过】：这些项目是可靠的，可以直接使用。它们不需要任何关注，但是阅读它们会让您感到温暖。越多越好！

+   【信息】：这些是您应该审查并修复的项目，如果您认为它们与您的设置和安全需求相关。

+   【警告】：这些是需要修复的项目。这些是我们不希望看到的项目。

记住，我们在扫描中涵盖了六个主要主题，如下所示：

+   主机配置

+   Docker 守护程序配置

+   Docker 守护程序配置文件

+   容器镜像和构建文件

+   容器运行时

+   Docker 安全操作

让我们来看看我们在扫描的每个部分中看到的内容。这些扫描结果来自默认的 Ubuntu Docker 主机，在此时没有对系统进行任何调整。我们再次关注每个部分中的`[警告]`项目。当您运行您自己的扫描时，可能会出现其他警告，但这些将是最常见的，如果不是每个人都会首先遇到的。

### 理解输出 - 主机配置

让我们来看一下主机配置运行时输出：

```
[WARN] 1.1 - Create a separate partition for containers

```

对于这一点，您将希望将`/var/lib/docker`映射到一个单独的分区。

```
[WARN] 1.8 - Failed to inspect: auditctl command not found.
[WARN] 1.9 - Failed to inspect: auditctl command not found.
[WARN] 1.10 - Failed to inspect: auditctl command not found.
[WARN] 1.13 - Failed to inspect: auditctl command not found.
[WARN] 1.18 - Failed to inspect: auditctl command not found.

```

### 理解输出 - Docker 守护程序配置

让我们来看一下 Docker 守护程序配置输出：

```
[WARN] 2.2 - Restrict network traffic between containers

```

默认情况下，运行在同一 Docker 主机上的所有容器都可以访问彼此的网络流量。要防止这种情况发生，您需要在 Docker 守护程序的启动过程中添加`--icc=false`标志：

```
[WARN] 2.7 - Do not use the aufs storage driver

```

同样，您可以在 Docker 守护程序启动过程中添加一个标志，以防止 Docker 使用`aufs`存储驱动程序。在 Docker 守护程序启动时使用`-s <storage_driver>`，您可以告诉 Docker 不要使用`aufs`作为存储。建议您使用适合您所使用的 Docker 主机操作系统的最佳存储驱动程序。

### 理解输出 - Docker 守护程序配置文件

如果您使用的是原始的 Docker 守护程序，您不应该看到任何警告。如果您以某种方式定制了代码，可能会在这里收到一些警告。这是您希望永远不会看到任何警告的一个领域。

### 理解输出-容器镜像和构建文件

让我们来看看容器镜像和构建文件运行时输出的输出：

```
[WARN] 4.1 - Create a user for the container
[WARN] * Running as root: suspicious_mccarthy

```

这说明`suspicious_mccarthy`容器正在以 root 用户身份运行，建议创建另一个用户来运行您的容器。

### 理解输出-容器运行时

让我们来看看容器运行时输出的输出，如下所示：

```
[WARN] 5.1: - Verify AppArmor Profile, if applicable
[WARN] * No AppArmorProfile Found: suspicious_mccarthy

```

这说明`suspicious_mccarthy`容器没有`AppArmorProfile`，这是 Ubuntu 中提供的额外安全性。

```
[WARN] 5.3 - Verify that containers are running only a single main process
[WARN] * Too many processes running: suspicious_mccarthy

```

这个错误非常直接。您需要确保每个容器只运行一个进程。如果运行多个进程，您需要将它们分布在多个容器中，并使用容器链接，如下命令所示：

```
[WARN] 5.4 - Restrict Linux Kernel Capabilities within containers
[WARN] * Capabilities added: CapAdd=[audit_control] to suspicious_mccarthy

```

这说明`audit_control`功能已添加到此运行的容器中。您可以使用`--cap-drop={}`从您的`docker run`命令中删除容器的额外功能，如下所示：

```
[WARN] 5.6 - Do not mount sensitive host system directories on containers
[WARN] * Sensitive directory /etc mounted in: suspicious_mccarthy
[WARN] * Sensitive directory /lib mounted in: suspicious_mccarthy
[WARN] 5.7 - Do not run ssh within containers
[WARN] * Container running sshd: suspicious_mccarthy

```

这很直接。不需要在容器内运行 SSH。您可以使用 Docker 提供的工具来对容器进行所有操作。确保任何容器中都没有运行 SSH。您可以使用`docker exec`命令来执行对容器的操作（更多信息请参见：[`docs.docker.com/engine/reference/commandline/exec/`](https://docs.docker.com/engine/reference/commandline/exec/)），如下命令所示：

```
[WARN] 5.10 - Do not use host network mode on container
[WARN] * Container running with networking mode 'host':
suspicious_mccarthy

```

这个问题在于，当容器启动时，传递了`--net=host`开关。不建议使用这个开关，因为它允许容器修改网络配置，并打开低端口号，以及访问 Docker 主机上的网络服务，如下所示：

```
[WARN] 5.11 - Limit memory usage for the container
[WARN] * Container running without memory restrictions:
suspicious_mccarthy

```

默认情况下，容器没有内存限制。如果在 Docker 主机上运行多个容器，这可能很危险。您可以在发出`docker run`命令时使用`-m`开关来限制容器的内存使用量。值以兆字节为单位（即 512 MB 或 1024 MB），如下命令所示：

```
[WARN] 5.12 - Set container CPU priority appropriately
[WARN] * The container running without CPU restrictions:
suspicious_mccarthy

```

与内存选项一样，您还可以在每个容器上设置 CPU 优先级。这可以在发出`docker run`命令时使用`--cpu-shares`开关来完成。CPU 份额基于数字 1,024。因此，一半将是 512，25%将是 256。使用 1,024 作为基本数字来确定 CPU 份额，如下所示：

```
[WARN] 5.13 - Mount container's root filesystem as readonly
[WARN] * Container running with root FS mounted R/W:
suspicious_mccarthy

```

您确实希望将容器用作不可变环境，这意味着它们不会在内部写入任何数据。数据应该写入卷。同样，您可以使用`--read-only`开关，如下所示：

```
[WARN] 5.16 - Do not share the host's process namespace
[WARN] * Host PID namespace being shared with: suspicious_mccarthy

```

当您使用`--pid=host`开关时会出现此错误。不建议使用此开关，因为它会破坏容器和 Docker 主机之间的进程隔离。

### 理解输出- Docker 安全操作

再次，您希望永远不要看到的另一个部分是如果您使用的是标准 Docker，则会看到警告。在这里，您将看到信息，并应该审查以确保一切正常。

# 总结

在本章中，我们看了一下 Docker 的 CIS 指南。这个指南将帮助您设置 Docker 环境的多个方面。最后，我们看了一下 Docker 安全基准。我们看了如何启动它并进行了一个示例，展示了运行后的输出。然后我们看了输出以了解其含义。请记住应用程序涵盖的六个项目：主机配置、Docker 守护程序配置、Docker 守护程序配置文件、容器镜像和构建文件、容器运行时和 Docker 安全操作。

在下一章中，我们将看一下如何监视以及报告您遇到的任何 Docker 安全问题。这将帮助您了解在现有环境中可能与安全有关的任何内容。如果您遇到自己发现的与安全有关的问题，有最佳实践可用于报告这些问题，以便 Docker 有时间修复它们，然后再允许公共社区知道这个问题，这将使黑客能够利用这些漏洞。

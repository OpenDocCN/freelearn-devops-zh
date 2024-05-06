# 前言

使用 Docker^(TM)，容器正在成为主流，企业已准备好在生产中使用它们。这本书专门设计帮助您快速掌握最新的 Docker 版本，并让您有信心在生产中使用它。本书还涵盖了 Docker 的用例、编排、集群、托管平台、安全性和性能，这将帮助您了解生产部署的不同方面。

Docker 及其生态系统正在以非常快的速度发展，因此了解基础知识并逐步采用新概念和工具非常重要。通过逐步的实用和适用的操作指南，“Docker Cookbook”不仅将帮助您使用当前版本的 Docker（1.6），而且通过附带的文本，将为您提供应对新版本 Docker 中的微小变化的概念信息。要了解更多关于本书的信息，请访问[`dockercookbook.github.io/`](http://dockercookbook.github.io/)。

Docker^(TM)是 Docker，Inc.的注册商标。

# 本书涵盖的内容

第一章，“介绍和安装”，将容器与裸机和虚拟机进行比较。它可以帮助您了解启用容器化的 Linux 内核功能；最后，我们将看一下安装操作。

第二章，“使用 Docker 容器”，涵盖了大部分与容器相关的操作，如启动、停止和删除容器。它还可以帮助您获取有关容器的低级信息。

第三章，“使用 Docker 镜像”，解释了与镜像相关的操作，如拉取、推送、导出、导入、基础镜像创建和使用 Dockerfile 创建镜像。我们还建立了一个私有注册表。

第四章，“容器的网络和数据管理”，涵盖了连接容器与另一个容器在外部世界的操作。它还涵盖了如何共享来自其他容器和主机系统的外部存储。

第五章，“Docker 的用例”，解释了大部分 Docker 的用例，如将 Docker 用于测试、CI/CD、设置 PaaS 以及将其用作计算引擎。

第六章，“Docker API 和语言绑定”，涵盖了 Docker 远程 API 和 Python 语言绑定作为示例。

第七章，“Docker 性能”，解释了一个人可以遵循的性能方法，以比较容器与裸金属和虚拟机的性能。它还涵盖了监控工具。

第八章，“Docker 编排和托管平台”，介绍了 Docker compose 和 Swarm。我们将研究 CoreOS 和 Project Atomic 作为容器托管平台，然后介绍 Docker 编排的 Kubernetes。

第九章，“Docker 安全性”，解释了一般安全准则，用于强制访问控制的 SELinux，以及更改功能和共享命名空间等其他安全功能。

第十章，“获取帮助和技巧和窍门”，提供了有关 Docker 管理和开发相关的帮助、技巧和资源。

# 本书需要什么

这本食谱中的食谱肯定会在安装了 Fedora 21 的物理机器或虚拟机上运行，因为我将该配置作为主要环境。由于 Docker 可以在许多平台和发行版上运行，您应该能够毫无问题地运行大多数食谱。对于一些食谱，您还需要 Vagrant ([`www.vagrantup.com/`](https://www.vagrantup.com/)) 和 Oracle Virtual Box ([`www.virtualbox.org/`](https://www.virtualbox.org/))。

# 本书适合谁

*Docker Cookbook*适用于希望在开发、QA 或生产环境中使用 Docker 的开发人员、系统管理员和 DevOps 工程师。

预计读者具有基本的 Linux/Unix 技能，如安装软件包，编辑文件，管理服务等。

任何关于虚拟化技术（如 KVM、XEN 和 VMware）的经验都将帮助读者更好地理解容器技术，但并非必需。

# 章节

在本书中，您会发现一些经常出现的标题（准备工作，如何做，它是如何工作的，还有更多，以及另请参阅）。

为了清晰地说明如何完成一个食谱，我们使用以下章节：

## 准备工作

本节告诉您在食谱中可以期待什么，并描述如何设置食谱所需的任何软件或任何初步设置。

## 如何做…

本节包含遵循食谱所需的步骤。

## 它是如何工作的…

本节通常包括对前一节发生的事情的详细解释。

## 还有更多…

本节包括有关食谱的额外信息，以使读者更加了解食谱。

## 另请参阅

本节提供有关食谱的其他有用信息的链接。

# 约定

在本书中，您将找到许多文本样式，用于区分不同类型的信息。以下是一些示例以及它们的含义解释。

文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄显示如下：“您可以使用`--driver/-d`选项来选择部署所需的多个端点之一。”

代码块设置如下：

```
[Unit] 
Description=MyApp 
After=docker.service 
Requires=docker.service 

[Service] 
TimeoutStartSec=0 
ExecStartPre=-/usr/bin/docker kill busybox1 
ExecStartPre=-/usr/bin/docker rm busybox1 
ExecStartPre=/usr/bin/docker pull busybox 
ExecStart=/usr/bin/docker run --name busybox1 busybox /bin/sh -c "while true; do echo Hello World; sleep 1; done" 
```

当我们希望引起您对代码块的特定部分的注意时，相关行或项目将以粗体显示：

```
[Service] 
Type=notify 
EnvironmentFile=-/etc/sysconfig/docker 
EnvironmentFile=-/etc/sysconfig/docker-storage 
**ExecStart=/usr/bin/docker -d -H fd:// $OPTIONS $DOCKER_STORAGE_OPTIONS** 
LimitNOFILE=1048576 
LimitNPROC=1048576 

[Install] 
WantedBy=multi-user.target 
```

任何命令行输入或输出都以以下方式编写：

```
**$ docker pull fedora**

```

**新术语**和**重要单词**以粗体显示。例如，在屏幕上看到的单词，例如菜单或对话框中的单词，会以这种方式出现在文本中：“转到项目主页，在**APIs & auth**部分下，选择**APIs**，并启用 Google **Compute Engine API**。”

### 注意

警告或重要说明会出现在这样的框中。

### 提示

技巧和窍门会出现在这样的地方。

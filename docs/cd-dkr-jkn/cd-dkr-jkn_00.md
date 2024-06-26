# 前言

多年来，我一直观察软件交付流程。我写了这本书，因为我知道有多少人仍然在发布过程中挣扎，并在日夜奋斗后感到沮丧。尽管多年来已经开发了许多自动化工具和流程，但这一切仍在发生。当我第一次看到持续交付流程是多么简单和有效时，我再也不愿意回到繁琐的传统手动交付周期。这本书是我经验的结果，也是我进行的许多持续交付研讨会的结果。我分享了使用 Jenkins、Docker 和 Ansible 的现代方法；然而，这本书不仅仅是工具。它介绍了持续交付背后的理念和推理，最重要的是，我向所有我遇到的人传达的主要信息：持续交付流程很简单，要使用它！

# 本书内容

第一章《介绍持续交付》介绍了公司传统的软件交付方式，并解释了使用持续交付方法改进的理念。本章还讨论了引入该流程的先决条件，并介绍了本书将构建的系统。

第二章《介绍 Docker》解释了容器化的概念和 Docker 工具的基础知识。本章还展示了如何使用 Docker 命令，将应用程序打包为 Docker 镜像，发布 Docker 容器的端口，并使用 Docker 卷。

第三章《配置 Jenkins》介绍了如何安装、配置和扩展 Jenkins。本章还展示了如何使用 Docker 简化 Jenkins 配置，并实现动态从节点供应。

第四章《持续集成管道》解释了流水线的概念，并介绍了 Jenkinsfile 语法。本章还展示了如何配置完整的持续集成管道。

第五章《自动验收测试》介绍了验收测试的概念和实施。本章还解释了工件存储库的含义，使用 Docker Compose 进行编排，以及编写面向 BDD 的验收测试的框架。

第六章，*使用 Ansible 进行配置管理*，介绍了配置管理的概念及其使用 Ansible 的实现。本章还展示了如何将 Ansible 与 Docker 和 Docker Compose 一起使用。

第七章，*持续交付流水线*，结合了前几章的所有知识，以构建完整的持续交付过程。本章还讨论了各种环境和非功能测试的方面。

第八章，*使用 Docker Swarm 进行集群*，解释了服务器集群的概念及其使用 Docker Swarm 的实现。本章还比较了替代的集群工具（Kubernetes 和 Apache Mesos），并解释了如何将集群用于动态 Jenkins 代理。

第九章，*高级持续交付*，介绍了与持续交付过程相关的不同方面的混合：数据库管理、并行流水线步骤、回滚策略、遗留系统和零停机部署。本章还包括持续交付过程的最佳实践。

# 本书所需内容

Docker 需要 64 位 Linux 操作系统。本书中的所有示例都是使用 Ubuntu 16.04 开发的，但任何其他具有 3.10 或更高内核版本的 Linux 系统都足够。

# 本书适合对象

本书适用于希望改进其交付流程的开发人员和 DevOps。无需先前知识即可理解本书。

# 约定

在本书中，您将找到一些区分不同信息种类的文本样式。以下是一些这些样式的示例，以及它们的含义解释。

文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄显示如下：`docker info`

代码块设置如下：

```
      pipeline {
           agent any
           stages {
                stage("Hello") {
                     steps {
                          echo 'Hello World'
                     }
                }
           }
      }
```

当我们希望引起您对代码块的特定部分的注意时，相关行或项目以粗体设置：

```
 FROM ubuntu:16.04
 RUN apt-get update && \
 apt-get install -y python
```

任何命令行输入或输出都以以下方式编写：

```
$ docker images
REPOSITORY              TAG     IMAGE ID         CREATED            SIZE
ubuntu_with_python      latest  d6e85f39f5b7  About a minute ago 202.6 MB
ubuntu_with_git_and_jdk latest  8464dc10abbb  3 minutes ago      610.9 MB
```

新术语和重要单词以粗体显示。您在屏幕上看到的单词，例如菜单或对话框中的单词，会在文本中显示为这样："点击 新项目"。

警告或重要说明以这样的框出现。

如果您的 Docker 守护程序在公司网络内运行，您必须配置 HTTP 代理。详细说明可以在[`docs.docker.com/engine/admin/systemd/`](https://docs.docker.com/engine/admin/systemd/)找到。

提示和技巧会显示在这样。

所有支持的操作系统和云平台的安装指南都可以在官方 Docker 页面[`docs.docker.com/engine/installation/`](https://docs.docker.com/engine/installation/)上找到。

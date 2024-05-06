# 前言

想象一下，在几分钟内在 Apache Tomcat 或 Wildfly 上创建和测试 Java EE 应用程序，以及迅速部署和管理 Java 应用程序。听起来太好了吧？您有理由欢呼，因为通过利用 Docker 和 Kubernetes，这样的场景是可能的。

本书将首先介绍 Docker，并深入探讨其网络和持久存储概念。然后，您将了解微服务的概念，以及如何将 Java 微服务部署和运行为 Docker 容器。接下来，本书将专注于 Kubernetes 及其特性。您将首先使用 Minikube 运行本地集群。下一步将是在亚马逊 AWS 上运行的 Kubernetes 上部署您的 Java 服务。在本书的最后，您将亲身体验一些更高级的主题，以进一步扩展您对 Docker 和 Kubernetes 的知识。

# 本书涵盖的内容

第一章，*Docker 简介*，介绍了 Docker 背后的原因，并介绍了 Docker 与传统虚拟化之间的区别。该章还解释了基本的 Docker 概念，如镜像、容器和 Dockerfile。

第二章，*网络和持久存储*，解释了 Docker 容器中网络和持久存储的工作原理。

第三章，*使用微服务*，概述了微服务的概念，并解释了它们与单片架构相比的优势。

第四章，*创建 Java 微服务*，探讨了通过使用 Java EE7 或 Spring Boot 快速构建 Java 微服务的方法。

第五章，*使用 Java 应用程序创建镜像*，教授如何将 Java 微服务打包成 Docker 镜像，无论是手动还是从 Maven 构建文件中。

第六章，*运行带有 Java 应用程序的容器*，展示了如何使用 Docker 运行容器化的 Java 应用程序。

第七章，*Kubernetes 简介*，介绍了 Kubernetes 的核心概念，如 Pod、节点、服务和部署。

第八章，*使用 Java 与 Kubernetes*，展示了如何在本地 Kubernetes 集群上部署打包为 Docker 镜像的 Java 微服务。

第九章，*使用 Kubernetes API*，展示了如何使用 Kubernetes API 来自动创建 Kubernetes 对象，如服务或部署。本章提供了如何使用 API 获取有关集群状态的信息的示例。

第十章，*在云中部署 Java 到 Kubernetes*，向读者展示了如何配置 Amazon AWS EC2 实例，使其适合运行 Kubernetes 集群。本章还详细说明了如何在 Amazon AWS 云上创建 Kubernetes 集群的方法。

第十一章，*更多资源*，探讨了 Java 和 Kubernetes 如何指向互联网上其他高质量的可用资源，以进一步扩展有关 Docker 和 Kubernetes 的知识。

# 本书所需内容

对于本书，您将需要任何一台能够运行现代版本的 Linux、Windows 10 64 位或 macOS 的体面 PC 或 Mac。

# 本书适合对象

本书适用于希望进入容器化世界的 Java 开发人员。读者将学习 Docker 和 Kubernetes 如何帮助在集群上部署和管理 Java 应用程序，无论是在自己的基础设施上还是在云中。

# 约定

在本书中，您将找到许多文本样式，用于区分不同类型的信息。以下是一些这些样式的示例及其含义的解释。文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄显示如下：“当您运行`docker build`命令时，Dockerfile 用于创建图像。”代码块设置如下：

```
{

"apiVersion": "v1",

"kind": "Pod",

"metadata":{

"name": ”rest_service”,

"labels": {

"name": "rest_service"

}

},

"spec": {

"containers": [{

"name": "rest_service",

"image": "rest_service",

"ports": [{"containerPort": 8080}],

}]

}

}
```

任何命令行输入或输出都以以下方式编写：

```
docker rm $(docker ps -a -q -f status=exited)

```

**新术语**和**重要单词**以粗体显示。屏幕上看到的单词，例如菜单或对话框中的单词，会在文本中出现，如：“点击“暂时跳过”将使您在不登录 Docker Hub 的情况下转到图像列表。”

警告或重要说明会出现在这样的框中。提示和技巧会出现在这样。

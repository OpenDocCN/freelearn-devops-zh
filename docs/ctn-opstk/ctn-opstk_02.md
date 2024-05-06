# 第二章：与容器编排引擎合作

在本章中，我们将看一下容器编排引擎（COE）。容器编排引擎是帮助管理在多个主机上运行的许多容器的工具。

在本章中，我们将涵盖以下主题：

+   COE 简介

+   Docker Swarm

+   Apache Mesos

+   Kubernetes

+   Kubernetes 安装

+   Kubernetes 实践

# COE 简介

容器为用户提供了一种打包和运行其应用程序的简便方法。打包涉及定义用户应用程序运行所必需的库和工具。一旦转换为图像，这些软件包可以用于创建和运行容器。这些容器可以在任何地方运行，无论是在开发人员的笔记本电脑，QA 系统还是生产机器上，而不需要改变环境。Docker 和其他容器运行时工具提供了管理这些容器的生命周期的功能。

使用这些工具，用户可以构建和管理图像，运行容器，删除容器，并执行其他容器生命周期操作。但是这些工具只能在单个主机上管理一个容器。当我们在多个容器和多个主机上部署我们的应用程序时，我们需要某种自动化工具。这种自动化通常被称为编排。编排工具提供了许多功能，包括：

+   提供和管理容器将运行的主机

+   从存储库中拉取图像并实例化容器

+   管理容器的生命周期

+   根据主机资源的可用性在主机上调度容器

+   当一个容器死掉时启动一个新的容器

+   扩展容器以匹配应用程序的需求

+   在容器之间提供网络，以便它们可以在不同的主机上相互访问

+   将这些容器公开为服务，以便可以从外部访问

+   对容器进行健康监控

+   升级容器

通常，这些类型的编排工具提供 YAML 或 JSON 格式的声明性配置。这些定义携带与容器相关的所有信息，包括图像、网络、存储、扩展和其他内容。编排工具使用这些定义来应用相同的设置，以便每次都提供相同的环境。

有许多容器编排工具可用，例如 Docker Machine，Docker Compose，Kubernetes，Docker Swarm 和 Apache Mesos，但本章仅关注 Docker Swarm，Apache Mesos 和 Kubernetes。

# Docker Swarm

**Docker Swarm**是 Docker 自身的本地编排工具。它管理一组 Docker 主机并将它们转换为单个虚拟 Docker 主机。Docker Swarm 提供了标准的 Docker API 来管理集群上的容器。如果用户已经在使用 Docker 来管理他们的容器，那么他们很容易转移到 Docker Swarm。

Docker Swarm 遵循*swap，plug 和 play*原则。这为集群提供了可插拔的调度算法，广泛的注册表和发现后端支持。用户可以根据自己的需求使用各种调度算法和发现后端。以下图表示 Docker Swarm 架构：

![](img/00010.jpeg)

# Docker Swarm 组件

以下各节解释了 Docker Swarm 中的各种组件。

# 节点

节点是参与 Swarm 集群的 Docker 主机的实例。单个 Swarm 集群部署中可以有一个或多个节点。根据它们在系统中的角色，节点被分类为管理节点和工作节点。

# 管理节点

Swarm 管理节点管理集群中的节点。它提供 API 来管理集群中的节点和容器。管理节点将工作单元（也称为任务）分配给工作节点。如果有多个管理节点，那么它们会选择一个单一的领导者来执行编排任务。

# 工作节点

工作节点接收并执行由管理节点分发的任务。默认情况下，每个管理节点也是工作节点，但它们可以配置为仅运行管理任务。工作节点运行代理并跟踪正在运行的任务，并报告它们。工作节点还通知管理节点有关分配任务的当前状态。

# 任务

任务是具有在容器内运行的命令的单个 Docker 容器。管理节点分配任务给工作节点。任务是集群中调度的最小单位。

# 服务

服务是跨 Swarm 集群运行的一组 Docker 容器或任务的接口。

# 发现服务

发现服务存储集群状态，并提供节点和服务的可发现性。Swarm 支持可插拔的后端架构，支持 etcd、Consul、Zookeeper、静态文件、IP 列表等作为发现服务。

# 调度程序

Swarm 调度程序在系统中的不同节点上调度任务。Docker Swarm 带有许多内置的调度策略，使用户能够指导容器在节点上的放置，以最大化或最小化集群中的任务分布。Swarm 也支持随机策略。它选择一个随机节点来放置任务。

# Swarm 模式

在 1.12 版本中，Docker 引入了内置的 Swarm 模式。要运行一个集群，用户需要在每个 Docker 主机上执行两个命令：

进入 Swarm 模式：

```
$ docker swarm init
```

添加节点到集群：

```
$ docker swarm join  
```

与 Swarm 不同，Swarm 模式内置于 Docker 引擎本身，具有服务发现、负载平衡、安全性、滚动更新和扩展等功能。Swarm 模式使集群管理变得简单，因为它不需要任何编排工具来创建和管理集群。

# Apache Mesos

Apache Mesos 是一个开源的、容错的集群管理器。它管理一组称为从节点的节点，并向框架提供它们的可用计算资源。框架从主节点获取资源可用性，并在从节点上启动任务。Marathon 就是这样一个框架，它在 Mesos 集群上运行容器化应用程序。Mesos 和 Marathon 一起成为一个类似于 Swarm 或 Kubernetes 的容器编排引擎。

以下图表示了整个架构：

![](img/00011.jpeg)

# Apache Mesos 及其组件

以下是 Apache Mesos 组件的列表：

# 主节点

主节点管理系统中的从节点。系统中可能有许多主节点，但只有一个被选举为领导者。

# 从节点

从节点是提供其资源给主节点并运行框架提供的任务的节点。

# 框架

框架是长期运行的应用程序，由调度程序组成，这些调度程序从主节点接受资源提供并在从节点上执行任务。

# 提供

提供只是每个从节点的可用资源的集合。主节点从从节点获取这些提供，并将它们提供给框架，框架反过来在从节点上运行任务。

# 任务

任务是由框架调度在从节点上运行的最小工作单元。例如，一个容器化应用程序可以是一个任务

# Zookeeper

Zookeeper 是集群中的集中式配置管理器。Mesos 使用 Zookeeper 来选举主节点，并让从节点加入集群

此外，Mesos Marathon 框架为长时间运行的应用程序（如容器）提供了服务发现和负载均衡。Marathon 还提供了 REST API 来管理工作负载。

# Kubernetes

Kubernetes 是由谷歌创建的容器编排引擎，旨在自动化容器化应用程序的部署、扩展和运行。它是最快发展的 COE 之一，因为它提供了一个可靠的平台，可以在大规模上构建分布式应用程序。Kubernetes 自动化您的应用程序，管理其生命周期，并在服务器集群中维护和跟踪资源分配。它可以在物理或虚拟机集群上运行应用程序容器。

它提供了一个统一的 API 来部署 Web 应用程序、数据库和批处理作业。它包括一套丰富的复杂功能：

+   自动扩展

+   自愈基础设施

+   批处理作业的配置和更新

+   服务发现和负载均衡

+   应用程序生命周期管理

+   配额管理

# Kubernetes 架构

本节概述了 Kubernetes 架构和各种组件，以提供一个运行中的集群。

从顶层视图来看，Kubernetes 由以下组件组成：

+   外部请求

+   主节点

+   工作节点

以下图显示了 Kubernetes 的架构：

![](img/00012.jpeg)

我们将在下一节详细讨论每个组件。图中描述了一些关键组件。

# 外部请求

用户通过 API 与 Kubernetes 集群进行交互；他们解释他们的需求以及他们的应用程序的样子，Kubernetes 会为他们管理所有的工作。`kubectl`是 Kubernetes 项目中的命令行工具，可以简单地调用 Kubernetes API。

# 主节点

主节点提供了集群的控制平面。它在集群中充当控制器的角色。大部分主要功能，如调度、服务发现、负载均衡、响应集群事件等，都是由运行在主节点上的组件完成的。现在，让我们来看看主要组件及其功能。

# kube-apiserver

它公开了 Kubernetes 的 API。所有内部和外部请求都通过 API 服务器。它验证所有传入请求的真实性和正确的访问级别，然后将请求转发到集群中的目标组件。

# etcd

`etcd`用于存储 Kubernetes 的所有集群状态信息。`etcd`是 Kubernetes 中的关键组件。

# kube-controller-manager

Kubernetes 集群中有多个控制器，如节点控制器、复制控制器、端点控制器、服务账户和令牌控制器。这些控制器作为后台线程运行，处理集群中的常规任务。

# kube-scheduler

它监视所有新创建的 pod，并在它们未分配到任何节点时将它们调度到节点上运行。

请阅读 Kubernetes 文档（[`kubernetes.io/docs/concepts/overview/components/`](https://kubernetes.io/docs/concepts/overview/components/)）了解控制平面中的其他组件，包括：

+   Cloud-controller-manager

+   Web UI

+   容器资源监控

+   集群级别日志记录

# 工作节点

工作节点运行用户的应用程序和服务。集群中可以有一个或多个工作节点。您可以向集群添加或删除节点，以实现集群的可伸缩性。工作节点还运行多个组件来管理应用程序。

# kubelet

`kubelet`是每个工作节点上的主要代理。它监听`kube-apiserver`的命令执行。`kubelet`的一些功能包括挂载 pod 的卷、下载 pod 的秘密、通过 Docker 或指定的容器运行时运行 pod 的容器等。

# kube-proxy

它通过在主机上维护网络规则并执行连接转发，为 Kubernetes 提供了服务抽象。

# 容器运行时

使用 Docker 或 Rocket 创建容器。

# supervisord

`supervisord`是一个轻量级的进程监视和控制系统，可用于保持`kubelet`和 Docker 运行。

# fluentd

`fluentd`是一个守护程序，帮助提供集群级别的日志记录。

# Kubernetes 中的概念

在接下来的章节中，我们将学习 Kubernetes 的概念，这些概念用于表示您的集群。

# Pod

Pod 是 Kubernetes 中最小的可部署计算单元。Pod 是一个或多个具有共享存储或共享网络的容器组，以及如何运行这些容器的规范。容器本身不分配给主机，而密切相关的容器总是作为 Pod 一起共同定位和共同调度，并在共享上下文中运行。

Pod 模型是一个特定于应用程序的逻辑主机；它包含一个或多个应用程序容器，并且它们之间相对紧密地耦合。在没有容器的世界中，它们将在同一台物理或虚拟机上执行。使用 Pod，我们可以更好地共享资源、保证命运共享、进行进程间通信并简化管理。

# 副本集和复制控制器

副本集是复制控制器的下一代。两者之间唯一的区别是副本集支持更高级的基于集合的选择器，而复制控制器只支持基于相等性的选择器，因此副本集比复制控制器更灵活。然而，以下解释适用于两者。

Pod 是短暂的，如果它正在运行的节点宕机，它将不会被重新调度。副本集确保特定数量的 Pod 实例（或副本）在任何给定时间都在运行。

# 部署

部署是一个高级抽象，它创建副本集和 Pod。副本集维护运行状态中所需数量的 Pod。部署提供了一种简单的方式来通过改变部署规范来升级、回滚、扩展或缩减 Pod。

# Secrets

Secrets 用于存储敏感信息，如用户名、密码、OAuth 令牌、证书和 SSH 密钥。将这些敏感信息存储在 Secrets 中比将它们放在 Pod 模板中更安全、更灵活。Pod 可以引用这些 Secrets 并使用其中的信息。

# 标签和选择器

标签是可以附加到对象（如 Pod 甚至节点）的键值对。它们用于指定对象的标识属性，这些属性对用户来说是有意义和相关的。标签可以在创建对象时附加，也可以在以后添加或修改。它们用于组织和选择对象的子集。一些示例包括环境（开发、测试、生产、发布）、稳定、派克等。

标签不提供唯一性。使用标签选择器，客户端或用户可以识别并随后管理一组对象。这是 Kubernetes 的核心分组原语，在许多情况下使用。

Kubernetes 支持两种选择器：基于相等性和基于集合。基于相等性使用键值对进行过滤，而基于集合更强大，允许根据一组值对键进行过滤。

# 服务

由于 pod 在 Kubernetes 中是短暂的对象，分配给它们的 IP 地址不能长时间稳定。这使得 pod 之间的通信变得困难。因此，Kubernetes 引入了服务的概念。服务是对一些 pod 的抽象，以及访问它们的策略，通常需要运行代理来通过虚拟 IP 地址与其他服务进行通信。

# 卷

卷为 pod 或容器提供持久存储。如果数据没有持久存储在外部存储上，那么一旦容器崩溃，所有文件都将丢失。卷还可以使多个容器之间的数据共享变得容易。Kubernetes 支持许多类型的卷，pod 可以同时使用任意数量的卷。

# Kubernetes 安装

Kubernetes 可以在各种平台上运行，从笔记本电脑和云提供商的虚拟机到一排裸机服务器。今天有多种解决方案可以安装和运行 Kubernetes 集群。阅读 Kubernetes 文档，找到适合您特定用例的最佳解决方案。

在本章中，我们将使用`kubeadm`在 Ubuntu 16.04+上创建一个 Kubernetes 集群。`kubeadm`可以用一个命令轻松地在每台机器上创建一个集群。

在这个安装中，我们将使用一个名为`kubeadm`的工具，它是 Kubernetes 的一部分。安装`kubeadm`的先决条件是：

+   一台或多台运行 Ubuntu 16.04+的机器

+   每台机器至少需要 1GB 或更多的 RAM

+   集群中所有机器之间的完整网络连接

集群中的所有机器都需要安装以下组件：

1.  在所有机器上安装 Docker。根据 Kubernetes 文档，建议使用 1.12 版本。有关安装 Docker 的说明，请参阅第一章中的*安装 Docker*部分，*使用容器*。

1.  在每台机器上安装`kubectl`。`kubectl`是来自 Kubernetes 的命令行工具，用于在 Kubernetes 上部署和管理应用程序。您可以使用`kubectl`来检查集群资源，创建、删除和更新组件，并查看您的新集群并启动示例应用程序。再次强调，安装`kubectl`有多种选项。在本章中，我们将使用 curl 进行安装。请参考 Kubernetes 文档以获取更多选项。

1.  使用 curl 下载最新版本的`kubectl`：

```
        $ curl -LO https://storage.googleapis.com/kubernetes-
        release/release/$(curl -s https://storage.googleapis.com/kubernetes
        release/release/stable.txt)/bin/linux/amd64/kubectl
```

1.  使`kubectl`二进制文件可执行：

```
        $ chmod +x ./kubectl  
```

1.  现在，在所有机器上安装`kubelet`和`kubeadm`。`kubelet`是在集群中所有机器上运行的组件，负责启动 pod 和容器等工作。`kubeadm`是引导集群的命令：

1.  以 root 用户登录：

```
        $ sudo -i  
```

1.  更新并安装软件包：

```
        $ apt-get update && apt-get install -y apt-transport-https
```

1.  为软件包添加认证密钥：

```
        $ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg 
        | apt-key add -  
```

1.  将 Kubernetes 源添加到`apt`列表中：

```
        $ cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
        deb http://apt.kubernetes.io/ kubernetes-xenial main
        EOF  
```

1.  更新并安装工具：

```
        $ apt-get update
        $ apt-get install -y kubelet kubeadm  
```

以下步骤演示了如何使用`kubeadm`设置安全的 Kubernetes 集群。我们还将在集群上创建一个 pod 网络，以便应用程序组件可以相互通信。最后，在集群上安装一个示例微服务应用程序以验证安装。

1.  初始化主节点。要初始化主节点，请选择之前安装了`kubeadm`的机器之一，并运行以下命令。我们已指定`pod-network-cidr`以提供用于通信的网络：

```
          $ kubeadm init --pod-network-cidr=10.244.0.0/16  
```

请参考`kubeadm`参考文档，了解更多关于`kubeadm init`提供的标志。

这可能需要几分钟，因为`kubeadm init`将首先运行一系列预检查，以确保机器准备好运行 Kubernetes。它可能会暴露警告并根据预检查结果退出错误。然后，它将下载并安装控制平面组件和集群数据库。

前面命令的输出如下：

```
[kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
[init] Using Kubernetes version: v1.7.4
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks
[preflight] WARNING: docker version is greater than the most recently validated version. Docker version: 17.06.1-ce. Max validated version: 1.12
[preflight] Starting the kubelet service
[kubeadm] WARNING: starting in 1.8, tokens expire after 24 hours by default (if you require a non-expiring token use --token-ttl 0)
[certificates] Generated CA certificate and key.
[certificates] Generated API server certificate and key.
[certificates] API Server serving cert is signed for DNS names [galvin kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.2.15]
[certificates] Generated API server kubelet client certificate and key.
[certificates] Generated service account token signing key and public key.
[certificates] Generated front-proxy CA certificate and key.
[certificates] Generated front-proxy client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[apiclient] Created API client, waiting for the control plane to become ready
[apiclient] All control plane components are healthy after 62.001439 seconds
[token] Using token: 07fb67.033bd701ad81236a
[apiconfig] Created RBAC rules
[addons] Applied essential addon: kube-proxy
[addons] Applied essential addon: kube-dns  
Your Kubernetes master has initialized successfully:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config 
You should now deploy a pod network to the cluster.
Run kubectl apply -f [podnetwork].yaml with one of the options listed at: http://kubernetes.io/docs/admin/addons/. You can now join any number of machines by running the following on each node as the root:
kubeadm join --token 07fb67.033bd701ad81236a 10.0.2.15:6443 
```

保存前面输出的`kubeadm join`命令。您将需要这个命令来将节点加入到您的 Kubernetes 集群中。令牌用于主节点和节点之间的相互认证。

现在，要开始使用您的集群，请以普通用户身份运行以下命令：

```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config  
```

1.  安装 pod 网络。此网络用于集群中 pod 之间的通信：

在运行任何应用程序之前，必须部署网络。此外，诸如`kube-dns`之类的服务在安装网络之前不会启动。`kubeadm`仅支持**容器网络接口**（**CNI**）网络，不支持`kubenet`。

有多个网络附加项目可用于创建安全网络。要查看完整列表，请访问 Kubernetes 文档以供参考。在本例中，我们将使用 flannel 进行网络连接。Flannel 是一种覆盖网络提供程序：

```
 $ sudo kubectl apply -f 
https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
 serviceaccount "flannel" created
 configmap "kube-flannel-cfg" created
 daemonset "kube-flannel-ds" created
 $ sudo kubectl apply -f 
https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
 clusterrole "flannel" created
 clusterrolebinding "flannel" created  
```

您可以通过检查输出中的`kube-dns` pod 是否正在运行来确认它是否正在工作：

```
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                             READY     STATUS    RESTARTS   AGE
kube-system   etcd-galvin                      1/1       Running   0          2m
kube-system   kube-apiserver-galvin            1/1       Running   0          2m
kube-system   kube-controller-manager-galvin   1/1       Running   0          2m
kube-system   kube-dns-2425271678-lz9fp        3/3       Running   0          2m
kube-system   kube-flannel-ds-f9nx8            2/2       Running   2          1m
kube-system   kube-proxy-wcmdg                 1/1       Running   0          2m
kube-system   kube-scheduler-galvin            1/1       Running   0          2m  
```

1.  将节点加入集群。要将节点添加到 Kubernetes 集群，请通过 SSH 连接到节点并运行以下命令：

```
$ sudo kubeadm join --token <token> <master-ip>:<port>
[kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
[preflight] Running pre-flight checks
[discovery] Trying to connect to API Server "10.0.2.15:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://10.0.2.15:6443"
[discovery] Cluster info signature and contents are valid, will use API Server "https://10.0.2.15:6443"
[discovery] Successfully established connection with API Server "10.0.2.15:6443"
[bootstrap] Detected server version: v1.7.4
[bootstrap] The server supports the Certificates API (certificates.k8s.io/v1beta1)
[csr] Created API client to obtain unique certificate for this node, generating keys and certificate signing request
[csr] Received signed certificate from the API server, generating KubeConfig...
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"  
Node join complete:
Certificate signing request sent to master and response
Received
Kubelet informed of new secure connection details
Run kubectl get nodes on the master to see this machine join.
```

现在，运行以下命令来验证节点的加入：

```
$ kubectl get nodes
NAME      STATUS    AGE       VERSION
brunno    Ready     14m       v1.7.4
```

通过创建一个示例 Nginx pod 来验证您的安装：

```
$ kubectl run my-nginx --image=nginx --replicas=2 --port=80
deployment "my-nginx" created

$ kubectl get pods 
NAME                        READY     STATUS    RESTARTS   AGE
my-nginx-4293833666-c4c5p   1/1       Running   0          22s
my-nginx-4293833666-czrnf   1/1       Running   0          22s  
```

# Kubernetes 实践

我们在上一节中学习了如何安装 Kubernetes 集群。现在，让我们使用 Kubernetes 创建一个更复杂的示例。在这个应用程序中，我们将部署一个运行 WordPress 站点和 MySQL 数据库的应用程序，使用官方 Docker 镜像。

1.  创建持久卷。WordPress 和 MySQL 将使用此卷来存储数据。我们将创建两个大小为 5 GB 的本地持久卷。将以下内容复制到`volumes.yaml`文件中：

```
        apiVersion: v1
        kind: PersistentVolume
        metadata:
          name: pv-1
          labels:
            type: local
        spec:
          capacity:
            storage: 5Gi
          accessModes:
            - ReadWriteOnce
          hostPath:
            path: /tmp/data/pv-1
         storageClassName: slow 
        ---
        apiVersion: v1
        kind: PersistentVolume
        metadata:
          name: pv-2
          labels:
            type: local
        spec:
          capacity:
            storage: 5Gi
          accessModes:
            - ReadWriteOnce
          hostPath:
            path: /tmp/data/pv-2
        storageClassName: slow 

```

1.  现在，通过运行以下命令来创建卷：

```
 $ kubectl create -f volumes.yaml 
 persistentvolume "pv-1" created
 persistentvolume "pv-2" created    
```

1.  检查卷是否已创建：

```
          $ kubectl get pv
          NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      
          CLAIM     STORAGECLASS   REASON    AGE
          pv-1      5Gi        RWO           Retain          Available                                     
          8s
          pv-2      5Gi        RWO           Retain          Available                                    
          8s  
```

1.  创建一个用于存储 MySQL 密码的密钥。MySQL 和 WordPress pod 将引用此密钥，以便这些 pod 可以访问它：

```
        $ kubectl create secret generic mysql-pass -from-
        literal=password=admin
        secret "mysql-pass" created
```

1.  验证密钥是否已创建：

```
        $ kubectl get secrets
        NAME                  TYPE                                  DATA    
        AGE
        default-token-1tb58   kubernetes.io/service-account-token   3      
        3m
        mysql-pass            Opaque                                1   
        9s
```

1.  创建 MySQL 部署。现在，我们将创建一个服务，公开一个 MySQL 容器，一个 5 GB 的持久卷索赔，以及运行 MySQL 容器的 pod 的部署。将以下内容复制到`mysql-deployment.yaml`文件中：

```
        apiVersion: v1
        kind: Service
        metadata:
          name: wordpress-mysql
          labels:
            app: wordpress
        spec:
          ports:
            - port: 3306
          selector:
            app: wordpress
            tier: mysql
          clusterIP: None
        ---
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: mysql-pv-claim
          labels:
            app: wordpress
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi
        storageClassName: slow 

        ---
        apiVersion: extensions/v1beta1
        kind: Deployment
        metadata:
          name: wordpress-mysql
          labels:
            app: wordpress
        spec:
          strategy:
            type: Recreate
          template:
            metadata:
              labels:
                app: wordpress
                tier: mysql
            spec:
              containers:
              - image: mysql:5.6
                name: mysql
                env:
                - name: MYSQL_ROOT_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: mysql-pass
                      key: password
                    ports:
                - containerPort: 3306
                  name: mysql
                volumeMounts:
                - name: mysql-persistent-storage
                  mountPath: /var/lib/mysql
              volumes:
              - name: mysql-persistent-storage
                persistentVolumeClaim:
                   claimName: mysql-pv-claim  
```

1.  现在，启动 MySQL pod：

```
        $ kubectl create -f mysql-deployment.yaml 
          service "wordpress-mysql" created
          persistentvolumeclaim "mysql-pv-claim" created
          deployment "wordpress-mysql" created  
```

1.  检查 pod 的状态：

```
          $ kubectl get pods
          NAME                               READY     STATUS    RESTARTS
          AGE
            wordpress-mysql-2222028001-l8x9x   1/1       Running   0  
          6m      
```

1.  或者，您可以通过运行以下命令来检查 pod 的日志：

```
        $ kubectl logs wordpress-mysql-2222028001-l8x9x

        Initializing database
        2017-08-27 15:30:00 0 [Warning] TIMESTAMP with implicit DEFAULT 
        value is deprecated. Please use --explicit_defaults_for_timestamp 
        server 
        option (see documentation for more details).
        2017-08-27 15:30:00 0 [Note] Ignoring --secure-file-priv value as
        server is running with --bootstrap.
        2017-08-27 15:30:00 0 [Note] /usr/sbin/mysqld (mysqld 5.6.37)
        starting as process 36 ...

        2017-08-27 15:30:03 0 [Warning] TIMESTAMP with implicit DEFAULT
        value is deprecated. Please use --explicit_defaults_for_timestamp 
        server 
        option (see documentation for more details).
        2017-08-27 15:30:03 0 [Note] Ignoring --secure-file-priv value as 
        server is running with --bootstrap.
        2017-08-27 15:30:03 0 [Note] /usr/sbin/mysqld (mysqld 5.6.37)
        starting as process 59 ...
        Please remember to set a password for the MySQL root user!
 To do so, start the server, then issue the following 
 commands:
 /usr/bin/mysqladmin -u root password 'new-password' 
        /usr/bin/mysqladmin -u root -h wordpress-mysql-2917821887-dccql 
        password 'new-password' 
```

或者，您可以运行以下命令：

```
/usr/bin/mysql_secure_installation 
```

这还将为您提供删除默认创建的测试数据库和匿名用户的选项。强烈建议用于生产服务器。

查看手册以获取更多说明：

请在[`bugs.mysql.com/`](http://bugs.mysql.com/)报告任何问题。有关 MySQL 的最新信息可在网上获取：[`www.mysql.com`](http://www.mysql.com)。通过在[`shop.mysql.com`](http://shop.mysql.com)购买支持/许可证来支持 MySQL。

请注意，没有创建新的默认`config`文件；请确保您的`config`文件是最新的。

默认的`config`文件`/etc/mysql/my.cnf`存在于系统上。

此文件将被 MySQL 服务器默认读取。如果您不想使用它，要么删除它，要么使用以下命令：

```
--defaults-file argument to mysqld_safe when starting the server

Database initialized
MySQL init process in progress...
2017-08-27 15:30:05 0 [Warning] TIMESTAMP with implicit DEFAULT 
value is deprecated. Please use --explicit_defaults_for_timestamp 
server option (see documentation for more details).
2017-08-27 15:30:05 0 [Note] mysqld (mysqld 5.6.37) starting as 
process 87 ...
Warning: Unable to load '/usr/share/zoneinfo/iso3166.tab' as time 
zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/leap-seconds.list' as
time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone.tab' as time
zone. Skipping it.  
```

MySQL 的`init`过程现在已经完成。我们已经准备好启动：

```
2017-08-27 15:30:11 0 [Warning] TIMESTAMP with implicit DEFAULT 
value is deprecated. Please use --explicit_defaults_for_timestamp
server 
option (see documentation for more details).
2017-08-27 15:30:11 0 [Note] mysqld (mysqld 5.6.37) starting as
process 5 ...  
```

通过运行以下命令检查持久卷索赔的状态：

```
$ kubectl get pvc
NAME             STATUS    VOLUME    CAPACITY   ACCESSMODES   
STORAGECLASS   AGE
mysql-pv-claim   Bound     pv-1      5Gi        RWO         
slow           2h
wp-pv-claim      Bound     pv-2      5Gi        RWO         
slow           2h
```

创建 WordPress 部署。我们现在将创建一个服务，公开一个 WordPress 容器，一个持久卷索赔 5GB，以及运行 WordPress 容器的 pod 的部署。将以下内容复制到`wordpress-deployment.yaml`文件中：

```
apiVersion: v1 
kind: Service 
metadata: 
  name: wordpress 
  labels: 
    app: wordpress 
spec: 
  ports: 
    - port: 80 
  selector: 
    app: wordpress 
    tier: frontend 
  type: NodePort 
--- 
apiVersion: v1 
kind: PersistentVolumeClaim 
metadata: 
  name: wp-pv-claim 
  labels: 
    app: wordpress 
spec: 
  accessModes: 
    - ReadWriteOnce 
  resources: 
    requests: 
      storage: 5Gi 
  storageClassName: slow  

--- 
apiVersion: extensions/v1beta1 
kind: Deployment 
metadata: 
  name: wordpress 
  labels: 
    app: wordpress 
spec: 
  strategy: 
    type: Recreate 
  template: 
    metadata: 
      labels: 
        app: wordpress 
        tier: frontend 
    spec: 
      containers: 
      - image: wordpress:4.7.3-apache 
        name: wordpress 
        env: 
        - name: WORDPRESS_DB_HOST 
          value: wordpress-mysql 
        - name: WORDPRESS_DB_PASSWORD 
          valueFrom: 
            secretKeyRef: 
              name: mysql-pass 
              key: password 
        ports: 
        - containerPort: 80 
          name: wordpress 
        volumeMounts: 
        - name: wordpress-persistent-storage 
          mountPath: /var/www/html 
      volumes: 
      - name: wordpress-persistent-storage 
        persistentVolumeClaim: 
          claimName: wp-pv-claim 
```

1.  现在，启动 WordPress pod：

```
    $ kubectl create -f wordpress-deployment.yaml 
      service "wordpress" created
      persistentvolumeclaim "wp-pv-claim" created
      deployment "wordpress" created

```

1.  检查服务的状态：

```
        $ kubectl get services wordpress
        NAME        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
        wordpress   10.99.124.161   <nodes>      80:31079/TCP   4m
```

应用程序现在正在运行！

以下列出了删除所有创建的资源所需的命令：

+   要删除您的秘密：

```
        $ kubectl delete secret mysql-pass  
```

+   要删除所有部署和服务：

```
        $ kubectl delete deployment -l app=wordpress
        $ kubectl delete service -l app=wordpress
```

+   要删除持久卷索赔和持久卷：

```
        $ kubectl delete pvc -l app=wordpress
        $ kubectl delete pv pv-1 pv-2  
```

# 摘要

在本章中，我们学习了容器编排引擎。我们看了不同的 COE，如 Docker Swarm 和 Apache Mesos。我们详细介绍了 Kubernetes 及其架构、组件和概念。

我们学会了如何使用`kubeadm`工具安装 Kubernetes 集群。然后，在最后，我们进行了一个实际操作，将 MySQL WordPress 应用程序在 Kubernetes 集群上运行。在下一章中，我们将了解 OpenStack 架构及其核心组件。

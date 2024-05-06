# 根据资源使用自动调整部署和有状态集

变化是所有存在的基本过程。

- *斯波克*

到目前为止，你可能已经了解到，基于 Kubernetes 的系统的一个关键方面是高度的动态性。几乎没有什么是静态的。我们定义部署或有状态集，Kubernetes 会在集群中分发 Pods。在大多数情况下，这些 Pods 很少在一个地方停留很长时间。滚动更新会导致 Pods 被重新创建并可能移动到其他节点。任何类型的故障都会引发受影响资源的重新调度。许多其他事件也会导致 Pods 移动。Kubernetes 集群就像一个蜂巢。它充满了生机，而且总是在运动中。

Kubernetes 集群的动态性不仅是由我们（人类）的行为或由故障引起的重新调度所致。自动缩放也应该受到责备。我们应该充分接受 Kubernetes 的动态性，并朝着能够满足我们应用程序需求的自主和自给的集群发展。为了实现这一点，我们需要提供足够的信息，让 Kubernetes 能够调整应用程序以及构成集群的节点。在本章中，我们将重点关注前一种情况。我们将探讨基于内存和 CPU 消耗的自动缩放 Pods 的常用和基本方法。我们将使用 HorizontalPodAutoscaler 来实现这一点。

HorizontalPodAutoscaler 的唯一功能是自动调整部署、有状态集或其他一些类型资源中 Pods 的数量。它通过观察 Pods 的 CPU 和内存消耗，并在达到预定义阈值时采取行动来实现这一点。

HorizontalPodAutoscaler 被实现为 Kubernetes API 资源和控制器。资源决定了控制器的行为。控制器定期调整有状态集或部署中的副本数量，以匹配用户指定的目标平均 CPU 利用率。

我们很快就会看到 HorizontalPodAutoscaler 的实际应用，并通过实际示例评论其特定功能。但在那之前，我们需要一个 Kubernetes 集群以及一个度量源。

# 创建集群

在创建集群之前（或开始使用您已经可用的集群），我们将克隆 `vfarcic/k8s-specs` ([`github.com/vfarcic/k8s-specs`](https://github.com/vfarcic/k8s-specs)) 存储库，其中包含本书中大部分我们将使用的定义。

给 Windows 用户的说明：请从 Git Bash 中执行本书中的所有命令。这样，您就可以直接运行它们，而不需要修改其语法以适应 Windows 终端或 PowerShell。本章中的所有命令都可以在 `01-hpa.sh` ([`gist.github.com/vfarcic/b46ca2eababb98d967e3e25748740d0d`](https://gist.github.com/vfarcic/b46ca2eababb98d967e3e25748740d0d)) Gist 中找到。

```
 1  git clone https://github.com/vfarcic/k8s-specs.git
 2
 3  cd k8s-specs
```

如果您之前克隆过该存储库，请确保通过执行 `git pull` 来获取最新版本。

以下的代码片段和规范用于测试本章中的命令。请在创建自己的测试集群时以此为灵感，或者验证您计划用于练习的集群是否满足最低要求。

+   `docker-scale.sh`: **Docker for Desktop** with 2 CPUs, 2 GB RAM and with **tiller** ([`gist.github.com/vfarcic/ca52ff97fc80565af0c46c37449babac`](https://gist.github.com/vfarcic/ca52ff97fc80565af0c46c37449babac)).

+   `minikube-scale.sh`: **minikube** with 2 CPUs, 2 GB RAM and with **tiller** ([`gist.github.com/vfarcic/5bc07d822f8825263245829715261a68`](https://gist.github.com/vfarcic/5bc07d822f8825263245829715261a68)).

+   `gke-scale.sh`: **GKE** with 3 n1-standard-1 worker nodes and with **tiller** ([`gist.github.com/vfarcic/9c777487f7ebee6c09027d3a1df8663c`](https://gist.github.com/vfarcic/9c777487f7ebee6c09027d3a1df8663c)).

+   `eks-scale.sh`: **EKS** with 3 t2.small worker nodes and with **tiller** ([`gist.github.com/vfarcic/a94dffef7d6dc60f79570d351c92408d`](https://gist.github.com/vfarcic/a94dffef7d6dc60f79570d351c92408d)).

+   `aks-scale.sh`: **AKS** with 3 Standard_B2s worker nodes and with **tiller** ([`gist.github.com/vfarcic/f1b05d33cc8a98e4ceab3d3770c2fe0b`](https://gist.github.com/vfarcic/f1b05d33cc8a98e4ceab3d3770c2fe0b)).

请注意，我们将使用 Helm 来安装必要的应用程序，但我们将切换到“纯粹”的 Kubernetes YAML 来尝试（可能是新的）本章中使用的资源，并部署演示应用程序。换句话说，我们将使用 Helm 进行一次性安装（例如，Metrics Server），并使用 YAML 来更详细地探索我们将要使用的内容（例如，HorizontalPodAutoscaler）。

现在，让我们来谈谈 Metrics Server。

# 观察 Metrics Server 数据

在扩展 Pods 的关键元素是 Kubernetes Metrics Server。你可能认为自己是 Kubernetes 的高手，但从未听说过 Metrics Server。如果是这种情况，不要感到羞愧。你并不是唯一一个。

如果你开始观察 Kubernetes 的指标，你可能已经使用过 Heapster。它已经存在很长时间了，你可能已经在你的集群中运行它，即使你不知道它是什么。两者都有相同的目的，其中一个已经被弃用了一段时间，所以让我们澄清一下事情。

早期，Kubernetes 引入了 Heapster 作为一种工具，用于为 Kubernetes 启用容器集群监控和性能分析。它从 Kubernetes 版本 1.0.6 开始存在。你可以说 Heapster 从 Kubernetes 的幼年时代就开始了。它收集和解释各种指标，如资源使用情况、事件等。Heapster 一直是 Kubernetes 的一个重要组成部分，并使其能够适当地调度 Pods。没有它，Kubernetes 将是盲目的。它不会知道哪个节点有可用内存，哪个 Pod 使用了太多的 CPU 等等。但是，就像大多数其他早期可用的工具一样，它的设计是一个“失败的实验”。

随着 Kubernetes 的持续增长，我们（Kubernetes 周围的社区）开始意识到需要一个新的、更好的、更重要的是更具可扩展性的设计。因此，Metrics Server 诞生了。现在，尽管 Heapster 仍在使用中，但它被视为已弃用，即使在今天（2018 年 9 月），Metrics Server 仍处于测试阶段。

那么，Metrics Server 是什么？一个简单的解释是，它收集有关节点和 Pod 使用的资源（内存和 CPU）的信息。它不存储指标，所以不要认为您可以使用它来检索历史值和预测趋势。有其他工具可以做到这一点，我们稍后会探讨它们。相反，Metrics Server 的目标是提供一个 API，可以用来检索当前的资源使用情况。我们可以通过`kubectl`或通过发送直接请求，比如`curl`来使用该 API。换句话说，Metrics Server 收集集群范围的指标，并允许我们通过其 API 检索这些指标。这本身就非常强大，但这只是故事的一部分。

我已经提到了可扩展性。我们可以扩展 Metrics Server 以从其他来源收集指标。我们会在适当的时候到达那里。现在，我们将探索它提供的开箱即用功能，以及它如何与一些其他 Kubernetes 资源交互，这些资源将帮助我们使我们的 Pods 可伸缩和更具弹性。

如果您读过我的其他书，您就会知道我不会过多涉及理论，而是更喜欢通过实际示例来演示功能和原则。这本书也不例外，我们将直接深入了解 Metrics Server 的实际练习。第一步是安装它。

Helm 使安装几乎任何公开可用的软件变得非常容易，如果有 Chart 可用的话。如果没有，您可能需要考虑另一种选择，因为这清楚地表明供应商或社区不相信 Kubernetes。或者，也许他们没有必要开发 Chart 的技能。无论哪种方式，最好的做法是远离它并采用另一种选择。如果这不是一个选择，那就自己开发一个 Helm Chart。在我们的情况下，不需要这样的措施。Metrics Server 确实有一个 Helm Chart，我们需要做的就是安装它。

GKE 和 AKS 用户请注意，Google 和 Microsoft 已经将 Metrics Server 作为其托管的 Kubernetes 集群（GKE 和 AKS）的一部分进行了打包。无需安装它，请跳过接下来的命令。对于 minikube 用户，请注意，Metrics Server 作为插件之一可用。请执行`minikube addons enable metrics-server`和`kubectl -n kube-system rollout status deployment metrics-server`命令，而不是接下来的命令。对于 Docker for Desktop 用户，请注意，Metrics Server 的最新更新默认情况下不适用于自签名证书。由于 Docker for Desktop 使用这样的证书，您需要允许不安全的 TLS。请在接下来的`helm install`命令中添加`--set args={"--kubelet-insecure-tls=true"}`参数。

```
 1  helm install stable/metrics-server \
 2      --name metrics-server \
 3      --version 2.0.2 \
 4      --namespace metrics
 5
 6  kubectl -n metrics \
 7      rollout status \
 8      deployment metrics-server
```

我们使用 Helm 安装了 Metrics Server，并等待直到它部署完成。

Metrics Server 将定期从运行在节点上的 Kubeletes 中获取指标。目前，这些指标包括 Pod 和节点的内存和 CPU 利用率。其他实体可以通过具有 Master Metrics API 的 API 服务器从 Metrics Server 请求数据。这些实体的一个例子是调度程序，一旦安装了 Metrics Server，就会使用其数据来做出决策。

很快您将会看到，Metrics Server 的使用超出了调度程序，但是目前，这个解释应该提供了一个基本数据流的图像。

![](img/b3d2c07d-2505-4373-ae4c-181a83afb8c0.png)图 1-1：数据流向和从 Metrics Server 获取数据的基本流程（箭头显示数据流向）

现在我们可以探索一种检索指标的方式。我们将从与节点相关的指标开始。

```
 1  kubectl top nodes
```

如果您很快，输出应该会声明“尚未提供指标”。这是正常的。在执行第一次迭代的指标检索之前需要几分钟时间。例外情况是 GKE 和 AKS，它们已经预先安装了 Metrics Server。

在重复命令之前先去冲杯咖啡。

```
 1  kubectl top nodes
```

这次，输出是不同的。

在本章中，我将展示来自 Docker for Desktop 的输出。根据您使用的 Kubernetes 版本不同，您的输出也会有所不同。但是，逻辑是相同的，您不应该有问题跟随操作。

我的输出如下。

```
NAME               CPU(cores) CPU% MEMORY(bytes) MEMORY%
docker-for-desktop 248m       12%  1208Mi        63%
```

我们可以看到我有一个名为`docker-for-desktop`的节点。它正在使用 248 CPU 毫秒。由于节点有两个核心，这占总可用 CPU 的 12%。同样，使用了 1.2GB 的 RAM，这占总可用内存 2GB 的 63%。

节点的资源使用情况很有用，但不是我们要寻找的内容。在本章中，我们专注于 Pod 的自动扩展。但是，在我们开始之前，我们应该观察一下我们的每个 Pod 使用了多少内存。我们将从在`kube-system`命名空间中运行的 Pod 开始。

```
 1  kubectl -n kube-system top pod
```

输出（在 Docker for Desktop 上）如下。

```
NAME                                       CPU(cores) MEMORY(bytes)
etcd-docker-for-desktop                    16m        74Mi
kube-apiserver-docker-for-desktop          33m        427Mi
kube-controller-manager-docker-for-desktop 44m        63Mi
kube-dns-86f4d74b45-c47nh                  1m         39Mi
kube-proxy-r56kd                           2m         22Mi
kube-scheduler-docker-for-desktop          13m        23Mi
tiller-deploy-5c688d5f9b-2pspz             0m         21Mi

```

我们可以看到`kube-system`中当前运行的每个 Pod 的资源使用情况（CPU 和内存）。如果我们找不到更好的工具，我们可以使用该信息来调整这些 Pod 的`requests`以使其更准确。但是，有更好的方法来获取这些信息，所以我们将暂时跳过调整。相反，让我们尝试获取所有 Pod 的当前资源使用情况，无论命名空间如何。

```
 1  kubectl top pods --all-namespaces
```

输出（在 Docker for Desktop 上）如下。

```
NAMESPACE   NAME                                       CPU(cores) MEMORY(bytes) 
docker      compose-7447646cf5-wqbwz                   0m         11Mi 
docker      compose-api-6fbc44c575-gwhxt               0m         14Mi 
kube-system etcd-docker-for-desktop                    16m        74Mi 
kube-system kube-apiserver-docker-for-desktop          33m        427Mi 
kube-system kube-controller-manager-docker-for-desktop 46m        63Mi 
kube-system kube-dns-86f4d74b45-c47nh                  1m         38Mi 
kube-system kube-proxy-r56kd                           3m         22Mi 
kube-system kube-scheduler-docker-for-desktop          14m        23Mi 
kube-system tiller-deploy-5c688d5f9b-2pspz             0m         21Mi 
metrics     metrics-server-5d78586d76-pbqj8            0m         10Mi 
```

该输出显示与上一个输出相同的信息，只是扩展到所有命名空间。不需要对其进行评论。

通常，Pod 的度量不够精细，我们需要观察构成 Pod 的每个容器的资源。要获取容器度量，我们只需要添加`--containers`参数。

```
 1  kubectl top pods \
 2    --all-namespaces \
 3    --containers
```

输出（在 Docker for Desktop 上）如下。

```
NAMESPACE   POD                                        NAME                 CPU(cores) MEMORY(bytes) 
docker      compose-7447646cf5-wqbwz                   compose                 0m         11Mi 
docker      compose-api-6fbc44c575-gwhxt               compose                 0m         14Mi 
kube-system etcd-docker-for-desktop                    etcd                    16m        74Mi 
kube-system kube-apiserver-docker-for-desktop          kube-apiserver          33m        427Mi 
kube-system kube-controller-manager-docker-for-desktop kube-controller-manager 46m        63Mi 
kube-system kube-dns-86f4d74b45-c47nh                  kubedns                 0m         13Mi 
kube-system kube-dns-86f4d74b45-c47nh                  dnsmasq                 0m         10Mi 
kube-system kube-dns-86f4d74b45-c47nh                  sidecar                 1m         14Mi 
kube-system kube-proxy-r56kd                           kube-proxy              3m         22Mi 
kube-system kube-scheduler-docker-for-desktop          kube-scheduler          14m        23Mi 
kube-system tiller-deploy-5c688d5f9b-2pspz             tiller                  0m         21Mi 
metrics     metrics-server-5d78586d76-pbqj8            metrics-server          0m         10Mi 
```

我们可以看到，这次输出显示了每个容器。例如，我们可以观察到`kube-dns-*` Pod 的度量分为三个容器（`kubedns`，`dnsmasq`，`sidecar`）。

当我们通过`kubectl top`请求指标时，数据流几乎与调度程序发出请求时的流程相同。请求被发送到 API 服务器（主度量 API），该服务器从度量服务器获取数据，而度量服务器又从集群节点上运行的 Kubeletes 收集信息。

![](img/e5faabe0-8a82-4b4a-b2b3-7e8298f6dc0a.png)图 1-2：数据流向和从度量服务器流向的方向（箭头显示数据流向）

虽然 `kubectl top` 命令对观察当前指标很有用，但如果我们想从其他工具访问它们，它就没什么用了。毕竟，我们的目标不是坐在终端前用 `watch "kubectl top pods"` 命令。那将是浪费我们（人类）的才能。相反，我们的目标应该是从其他工具中抓取这些指标，并根据实时和历史数据创建警报和（也许）仪表板。为此，我们需要以 JSON 或其他机器可解析的格式输出。幸运的是，`kubectl` 允许我们以原始格式直接调用其 API，并检索与工具查询相同的结果。

```
 1  kubectl get \
 2      --raw "/apis/metrics.k8s.io/v1beta1" \
 3      | jq '.'
```

输出如下。

```
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "nodes",
      "singularName": "",
      "namespaced": false,
      "kind": "NodeMetrics",
      "verbs": [
        "get",
        "list"
      ]
    },
    {
      "name": "pods",
      "singularName": "",
      "namespaced": true,
      "kind": "PodMetrics",
      "verbs": [
        "get",
        "list"
      ]
    }
  ]
}
```

我们可以看到 `/apis/metrics.k8s.io/v1beta1` 端点是一个索引 API，有两个资源（`nodes` 和 `pods`）。

让我们更仔细地看一下度量 API 的 `pods` 资源。

```
 1  kubectl get \
 2      --raw "/apis/metrics.k8s.io/v1beta1/pods" \
 3      | jq '.'
```

输出太大，无法在一本书中呈现，所以我会留给你去探索。你会注意到输出是通过 `kubectl top pods --all-namespaces --containers` 命令观察到的 JSON 等效物。

这是度量服务器的快速概述。有两件重要的事情需要注意。首先，它提供了集群内运行的容器的当前（或短期）内存和 CPU 利用率。第二个更重要的注意事项是我们不会直接使用它。度量服务器不是为人类设计的，而是为机器设计的。我们以后会到那里。现在，记住有一个叫做度量服务器的东西，你不应该直接使用它（一旦你采用了一个会抓取其度量的工具）。

现在我们已经探索了度量服务器，我们将尝试充分利用它，并学习如何根据资源利用率自动扩展我们的 Pods。

# 根据资源利用率自动扩展 Pods

我们的目标是部署一个应用程序，根据其资源使用情况自动扩展（或缩小）。我们将首先部署一个应用程序，然后讨论如何实现自动扩展。

我已经警告过您，我假设您熟悉 Kubernetes，并且在本书中我们将探讨监控，警报，扩展和其他一些特定主题。我们不会讨论 Pods，StatefulSets，Deployments，Services，Ingress 和其他“基本”Kubernetes 资源。这是您承认您不了解 Kubernetes 基础知识的最后机会，退一步，并阅读*The DevOps 2.3 Toolkit: Kubernetes* ([`www.devopstoolkitseries.com/posts/devops-23/`](https://www.devopstoolkitseries.com/posts/devops-23/))和*The DevOps 2.4 Toolkit: Continuous Deployment To Kubernetes* ([`www.devopstoolkitseries.com/posts/devops-24/`](https://www.devopstoolkitseries.com/posts/devops-24/)*)*。

让我们看一下我们示例中将使用的应用程序的定义。

```
 1  cat scaling/go-demo-5-no-sidecar-mem.yml
```

如果您熟悉 Kubernetes，YAML 定义应该是不言自明的。我们只会评论与自动扩展相关的部分。

输出，仅限于相关部分，如下。

```
...
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  namespace: go-demo-5
spec:
  ...
  template:
    ...
    spec:
      ...
      containers:
      - name: db
        ...
        resources:
          limits:
            memory: "150Mi"
            cpu: 0.2
          requests:
            memory: "100Mi"
            cpu: 0.1
        ...
      - name: db-sidecar
    ... 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: go-demo-5
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: api
        ...
        resources:
          limits:
            memory: 15Mi
            cpu: 0.1
          requests:
            memory: 10Mi
            cpu: 0.01
...
```

我们有两个形成应用程序的 Pod。 `api`部署是一个后端 API，使用`db` StatefulSet 来保存其状态。

定义的基本部分是“资源”。 `api`和`db`都为内存和 CPU 定义了“请求”和“限制”。数据库使用一个 sidecar 容器，将 MongoDB 副本加入到副本集中。请注意，与其他容器不同，sidecar 没有“资源”。这背后的重要性将在稍后揭示。现在，只需记住两个容器有定义的“请求”和“限制”，而另一个没有。

现在，让我们创建这些资源。

```
 1  kubectl apply \
 2      -f scaling/go-demo-5-no-sidecar-mem.yml \
 3      --record
```

输出应该显示已创建了相当多的资源，我们的下一步是等待`api`部署推出，从而确认应用程序正在运行。

```
 1  kubectl -n go-demo-5 \
 2      rollout status \
 3      deployment api
```

几分钟后，您应该会看到消息，指出“api”部署成功推出。

为了安全起见，我们将列出`go-demo-5`命名空间中的 Pod，并确认每个 Pod 都在运行一个副本。

```
 1  kubectl -n go-demo-5 get pods
```

输出如下。

```
NAME    READY STATUS  RESTARTS AGE
api-... 1/1   Running 0        1m
db-0    2/2   Running 0        1m
```

到目前为止，我们还没有做任何超出 StatefulSet 和 Deployment 的普通创建。

他们又创建了 ReplicaSets，这导致了 Pod 的创建。

![](img/ed5f3b30-98d3-4735-b3f4-7a4a79755f72.png)图 1-3：StatefulSet 和 Deployment 的创建

希望你知道，我们应该至少有每个 Pod 的两个副本，只要它们是可扩展的。然而，这两者都没有定义`replicas`。这是有意的。我们可以指定部署或 StatefulSet 的副本数量，并不意味着我们应该这样做。至少，不总是。

如果副本数量是静态的，并且你没有打算随时间扩展（或缩减）你的应用程序，那么将`replicas`作为部署或 StatefulSet 定义的一部分。另一方面，如果你计划根据内存、CPU 或其他指标更改副本数量，请改用 HorizontalPodAutoscaler 资源。

让我们来看一个 HorizontalPodAutoscaler 的简单示例。

```
 1  cat scaling/go-demo-5-api-hpa.yml
```

输出如下。

```
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: api
  namespace: go-demo-5
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 80
```

定义使用`HorizontalPodAutoscaler`来定位`api`部署。它的边界是最少两个和最多五个副本。这些限制是基本的。没有这些限制，我们会面临无限扩展或缩减到零副本的风险。`minReplicas`和`maxReplicas`字段是一个安全网。

定义的关键部分是`metrics`。它提供了 Kubernetes 应该使用的公式来决定是否应该扩展（或缩减）资源。在我们的例子中，我们使用`Resource`类型的条目。它们针对内存和 CPU 的平均利用率为 80％。如果两者中的任何一个实际使用情况偏离，Kubernetes 将扩展（或缩减）资源。

请注意，我们使用了 API 的`v2beta1`版本，你可能想知道为什么我们选择了这个版本，而不是稳定且适用于生产的`v1`。毕竟，`beta1`版本仍远未经过充分打磨以供一般使用。原因很简单。HorizontalPodAutoscaler `v1`太基础了。它只允许基于 CPU 进行扩展。即使我们的简单示例也超越了这一点，通过将内存加入其中。以后，我们将进一步扩展它。因此，虽然`v1`被认为是稳定的，但它并没有提供太多价值，我们可以等待`v2`发布，或者立即开始尝试`v2beta`版本。我们选择了后者。当你阅读这篇文章时，更稳定的版本可能已经存在并且在你的 Kubernetes 集群中得到支持。如果是这种情况，请随时在应用定义之前更改`apiVersion`。

现在让我们应用它。

```
 1  kubectl apply \
 2      -f scaling/go-demo-5-api-hpa.yml \
 3      --record
```

我们应用了创建**HorizontalPodAutoscaler**（**HPA**）的定义。接下来，我们将查看检索 HPA 资源时获得的信息。

```
 1  kubectl -n go-demo-5 get hpa
```

如果你很快，输出应该类似于以下内容。

```
NAME REFERENCE      TARGETS                      MINPODS MAXPODS REPLICAS AGE
api  Deployment/api <unknown>/80%, <unknown>/80% 2       5       0        20s

```

我们可以看到，Kubernetes 尚未具有实际的 CPU 和内存利用率，而是输出了`<unknown>`。在从 Metrics Server 收集下一次数据之前，我们需要再给它一些时间。在我们重复相同的查询之前，先喝杯咖啡。

```
 1  kubectl -n go-demo-5 get hpa
```

这次，输出中没有未知项。

```
NAME REFERENCE      TARGETS          MINPODS MAXPODS REPLICAS AGE
api  Deployment/api 38%/80%, 10%/80% 2       5       2        1m

```

我们可以看到，CPU 和内存利用率远低于预期的`80%`利用率。尽管如此，Kubernetes 将副本数从一个增加到两个，因为这是我们定义的最小值。我们签订了合同，规定`api` Deployment 的副本数永远不得少于两个，即使资源利用率远低于预期的平均利用率，Kubernetes 也会遵守这一点进行扩展。我们可以通过 HorizontalPodAutoscaler 的事件来确认这种行为。

```
 1  kubectl -n go-demo-5 describe hpa api
```

输出，仅限于事件消息，如下所示。

```
...
Events:
... Message
... -------
... New size: 2; reason: Current number of replicas below Spec.MinReplicas
```

事件的消息应该是不言自明的。HorizontalPodAutoscaler 将副本数更改为`2`，因为当前数量（1）低于`MinReplicas`值。

最后，我们将列出 Pods，以确认所需数量的副本确实正在运行。

```
 1  kubectl -n go-demo-5 get pods
```

输出如下。

```
NAME    READY STATUS  RESTARTS AGE
api-... 1/1   Running 0        2m
api-... 1/1   Running 0        6m
db-0    2/2   Running 0        6m
```

到目前为止，HPA 尚未根据资源使用情况执行自动缩放。相反，它只增加了 Pod 的数量以满足指定的最小值。它通过操纵 Deployment 来实现这一点。

![](img/c9a6db6e-d3a9-485e-a687-b92b0629e19e.png)图 1-4：根据 HPA 中指定的最小副本数进行部署的扩展

接下来，我们将尝试创建另一个 HorizontalPodAutoscaler，但这次，我们将以运行我们的 MongoDB 的 StatefulSet 为目标。因此，让我们再看一下另一个 YAML 定义。

```
 1  cat scaling/go-demo-5-db-hpa.yml
```

输出如下。

```
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: db
  namespace: go-demo-5
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: db
  minReplicas: 3
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 80
```

该定义几乎与我们之前使用的定义相同。唯一的区别是，这次我们的目标是名为`db`的`StatefulSet`，并且最小副本数应为`3`。

让我们应用它。

```
 1  kubectl apply \
 2      -f scaling/go-demo-5-db-hpa.yml \
 3      --record
```

让我们再看一下 HorizontalPodAutoscaler 资源。

```
 1  kubectl -n go-demo-5 get hpa
```

输出如下。

```
NAME REFERENCE      TARGETS                      MINPODS MAXPODS REPLICAS AGE
api  Deployment/api 41%/80%, 0%/80%              2       5       2        5m
db   StatefulSet/db <unknown>/80%, <unknown>/80% 3       5       0        20s
```

我们可以看到第二个 HPA 已经创建，并且当前利用率为“未知”。这一定是之前的类似情况。我们应该给它一些时间让数据开始流动吗？等待片刻，然后再次检索 HPA。目标仍然是“未知”吗？

资源利用持续未知可能有问题。让我们描述新创建的 HPA，看看是否能找到问题的原因。

```
 1  kubectl -n go-demo-5 describe hpa db
```

输出，仅限于事件消息，如下所示。

```
...
Events:
... Message
... -------
... New size: 3; reason: Current number of replicas below Spec.MinReplicas
... missing request for memory on container db-sidecar in pod go-demo-5/db-0
... failed to get memory utilization: missing request for memory on container db-sidecar in pod go-demo-5/db-0

```

请注意，您的输出可能只有一个事件，甚至没有这些事件。如果是这种情况，请等待几分钟，然后重复上一个命令。

如果我们关注第一条消息，我们可以看到它开始得很好。HPA 检测到当前副本数低于限制，并将它们增加到了三个。这是预期的行为，所以让我们转向其他两条消息。

HPA 无法计算百分比，因为我们没有指定`db-sidecar`容器请求多少内存。没有`requests`，HPA 无法计算实际内存使用的百分比。换句话说，我们忽略了为`db-sidecar`容器指定资源，HPA 无法完成其工作。我们将通过应用`go-demo-5-no-hpa.yml`来解决这个问题。

让我们快速看一下新定义。

```
 1  cat scaling/go-demo-5-no-hpa.yml
```

输出，仅限于相关部分，如下所示。

```
...
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  namespace: go-demo-5
spec:
  ...
  template:
    ...
    spec:
      ...
      - name: db-sidecar
        ...
        resources:
          limits:
            memory: "100Mi"
            cpu: 0.2
          requests:
            memory: "50Mi"
            cpu: 0.1
...
```

与初始定义相比，唯一显着的区别是这次我们为`db-sidecar`容器定义了资源。让我们应用它。

```
 1  kubectl apply \
 2      -f scaling/go-demo-5-no-hpa.yml \
 3      --record
```

接下来，我们将等待片刻以使更改生效，然后再次检索 HPA。

```
 1  kubectl -n go-demo-5 get hpa
```

这一次，输出更有希望。

```
NAME REFERENCE      TARGETS          MINPODS MAXPODS REPLICAS AGE
api  Deployment/api 66%/80%, 10%/80% 2       5       2        16m
db   StatefulSet/db 60%/80%, 4%/80%  3       5       3        10m
```

两个 HPA 都显示了当前和目标资源使用情况。都没有达到目标值，所以 HPA 保持了最小副本数。我们可以通过列出`go-demo-5`命名空间中的所有 Pod 来确认这一点。

```
 1  kubectl -n go-demo-5 get pods
```

输出如下。

```
NAME    READY STATUS  RESTARTS AGE
api-... 1/1   Running 0        42m
api-... 1/1   Running 0        46m
db-0    2/2   Running 0        33m
db-1    2/2   Running 0        33m
db-2    2/2   Running 0        33m
```

我们可以看到`api`部署有两个 Pod，而`db` StatefulSet 有三个副本。这些数字等同于 HPA 定义中的`spec.minReplicas`条目。

让我们看看当实际内存使用量高于目标值时会发生什么。

我们将通过降低其中一个 HPA 的目标来修改其定义，以重现我们的 Pod 消耗资源超出预期的情况。

让我们看一下修改后的 HPA 定义。

```
 1  cat scaling/go-demo-5-api-hpa-low-mem.yml
```

输出，仅限于相关部分，如下所示。

```
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: api
  namespace: go-demo-5
spec:
  ...
  metrics:
  ...
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 10
```

我们将`targetAverageUtilization`减少到`10`。这肯定低于当前的内存利用率，我们将能够见证 HPA 的工作。让我们应用新的定义。

```
 1  kubectl apply \
 2      -f scaling/go-demo-5-api-hpa-low-mem.yml \
 3      --record
```

请等待一段时间，以便进行下一次数据收集迭代，并检索 HPAs。

```
 1  kubectl -n go-demo-5 get hpa
```

输出如下。

```
NAME REFERENCE      TARGETS          MINPODS MAXPODS REPLICAS AGE
api  Deployment/api 49%/10%, 10%/80% 2       5       2        44m
db   StatefulSet/db 64%/80%, 5%/80%  3       5       3        39m
```

我们可以看到`api` HPA 的实际内存（`49%`）远远超过了阈值（`10%`）。然而，副本的数量仍然是相同的（`2`）。我们需要等待几分钟，然后再次检索 HPAs。

```
 1  kubectl -n go-demo-5 get hpa
```

这次，输出略有不同。

```
NAME REFERENCE      TARGETS          MINPODS MAXPODS REPLICAS AGE
api  Deployment/api 49%/10%, 10%/80% 2       5       4        44m
db   StatefulSet/db 64%/80%, 5%/80%  3       5       3        39m
```

我们可以看到副本数量增加到`4`。HPA 改变了部署，导致了级联效应，从而增加了 Pod 的数量。

让我们描述一下`api` HPA。

```
 1  kubectl -n go-demo-5 describe hpa api
```

输出，仅限于事件消息，如下所示。

```
...
Events:
... Message
... -------
... New size: 2; reason: Current number of replicas below Spec.MinReplicas
... New size: 4; reason: memory resource utilization (percentage of request) above target
```

我们可以看到 HPA 将大小更改为`4`，因为`内存资源利用率（请求百分比）`高于目标。

由于在这种情况下，增加副本数量并没有将内存消耗降低到 HPA 目标以下，我们应该期望 HPA 将继续扩展部署，直到达到`5`的限制。我们将通过等待几分钟并再次描述 HPA 来确认这一假设。

```
 1  kubectl -n go-demo-5 describe hpa api
```

输出，仅限于事件消息，如下所示。

```
...
Events:
... Message
... -------
... New size: 2; reason: Current number of replicas below Spec.MinReplicas
... New size: 4; reason: memory resource utilization (percentage of request) above target
... New size: 5; reason: memory resource utilization (percentage of request) above target
```

我们收到了消息，说明新的大小现在是`5`，从而证明 HPA 将继续扩展，直到资源低于目标，或者在我们的情况下，达到最大副本数量。

我们可以通过列出`go-demo-5`命名空间中的所有 Pod 来确认扩展确实起作用。

```
 1  kubectl -n go-demo-5 get pods
```

输出如下。

```
NAME    READY STATUS  RESTARTS AGE
api-... 1/1   Running 0        47m
api-... 1/1   Running 0        51m
api-... 1/1   Running 0        4m
api-... 1/1   Running 0        4m
api-... 1/1   Running 0        24s
db-0    2/2   Running 0        38m
db-1    2/2   Running 0        38m
db-2    2/2   Running 0        38m
```

正如我们所看到的，`api`部署确实有五个副本。

HPA 从 Metrics Server 中检索数据，得出实际资源使用量高于阈值，并使用新的副本数量操纵了部署。

![](img/1176f620-4e31-44e3-8eb0-5c8a4ef9b2e6.png)图 1-5：HPA 通过操纵部署进行扩展

接下来，我们将验证缩减副本数量也能正常工作。我们将重新应用初始定义，其中内存和 CPU 都设置为百分之八十。由于实际内存使用量低于该值，HPA 应该开始缩减，直到达到最小副本数量。

```
 1  kubectl apply \
 2      -f scaling/go-demo-5-api-hpa.yml \
 3      --record
```

与之前一样，我们将等待几分钟，然后再描述 HPA。

```
 1  kubectl -n go-demo-5 describe hpa api
```

输出，仅限于事件消息，如下所示。

```
...
Events:
... Message
... -------
... New size: 2; reason: Current number of replicas below Spec.MinReplicas
... New size: 4; reason: memory resource utilization (percentage of request) above target
... New size: 5; reason: memory resource utilization (percentage of request) above target
... New size: 3; reason: All metrics below target
```

正如我们所看到的，它将大小更改为`3`，因为所有的`metrics`都`below target`。

一段时间后，它会再次缩减到两个副本，并停止，因为这是我们在 HPA 定义中设置的限制。

# 在部署和有状态集中使用副本还是不使用副本？

知道 HorizontalPodAutoscaler（HPA）管理我们应用程序的自动扩展，可能会产生关于副本的问题。我们应该在我们的部署和有状态集中定义它们，还是应该完全依赖 HPA 来管理它们？我们不直接回答这个问题，而是探讨不同的组合，并根据结果定义策略。

首先，让我们看看我们集群中有多少个 Pods。

```
 1  kubectl -n go-demo-5 get pods
```

输出如下。

```
NAME    READY STATUS  RESTARTS AGE
api-... 1/1   Running 0        27m
api-... 1/1   Running 2        31m
db-0    2/2   Running 0        20m
db-1    2/2   Running 0        20m
db-2    2/2   Running 0        21m
```

我们可以看到`api`部署有两个副本，`db`有三个有状态集的副本。

假设我们想要发布一个新版本的`go-demo-5`应用程序。我们将使用的定义如下。

```
 1  cat scaling/go-demo-5-replicas-10.yml
```

输出，仅限于相关部分，如下所示。

```
...
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: go-demo-5
spec:
  replicas: 10
... 
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: api
  namespace: go-demo-5
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 80
```

需要注意的重要事情是我们的`api`部署有`10`个副本，并且我们有 HPA。其他一切都和以前一样。

如果我们应用了那个定义会发生什么？

```
 1  kubectl apply \
 2    -f scaling/go-demo-5-replicas-10.yml
 3
 4  kubectl -n go-demo-5 get pods
```

我们应用了新的定义，并从`go-demo-5`命名空间中检索了所有的 Pods。后一条命令的输出如下。

```
NAME    READY STATUS            RESTARTS AGE
api-... 1/1   Running           0        9s
api-... 0/1   ContainerCreating 0        9s
api-... 0/1   ContainerCreating 0        9s
api-... 1/1   Running           2        41m
api-... 1/1   Running           0        22s
api-... 0/1   ContainerCreating 0        9s
api-... 0/1   ContainerCreating 0        9s
api-... 1/1   Running           0        9s
api-... 1/1   Running           0        9s
api-... 1/1   Running           0        9s
db-0    2/2   Running           0        31m
db-1    2/2   Running           0        31m
db-2    2/2   Running           0        31m
```

Kubernetes 遵循我们希望有十个`api`副本的要求，并创建了八个 Pods（之前我们有两个）。乍一看，HPA 似乎没有任何效果。让我们再次检索 Pods。

```
 1  kubectl -n go-demo-5 get pods
```

输出如下。

```
NAME    READY STATUS  RESTARTS AGE
api-... 1/1   Running 0        30s
api-... 1/1   Running 2        42m
api-... 1/1   Running 0        43s
api-... 1/1   Running 0        30s
api-... 1/1   Running 0        30s
db-0    2/2   Running 0        31m
db-1    2/2   Running 0        32m
db-2    2/2   Running 0        32m
```

我们的部署从十个缩减到了五个副本。HPA 检测到副本超过了最大阈值，并相应地采取了行动。但它做了什么？它只是简单地移除了五个副本吗？那不可能，因为那只会有暂时的效果。如果 HPA 移除或添加 Pods，部署也会移除或添加 Pods，两者将互相对抗。Pods 的数量将无限波动。相反，HPA 修改了部署。

让我们描述一下`api`。

```
 1  kubectl -n go-demo-5 \
 2    describe deployment api
```

输出，仅限于相关部分，如下所示。

```
...
Replicas: 5 desired | 5 updated | 5 total | 5 available | 0 unavailable
...
Events:
... Message
... -------
...
... Scaled up replica set api-5bbfd85577 to 10
... Scaled down replica set api-5bbfd85577 to 5
```

副本的数量设置为`5 desired`。HPA 修改了我们的部署。我们可以通过事件消息更好地观察到这一点。倒数第二条消息表明副本的数量被扩展到`10`，而最后一条消息表明它被缩减到`5`。前者是我们通过应用新的部署来执行滚动更新的结果，而后者是由 HPA 修改部署并改变其副本数量产生的。

到目前为止，我们观察到 HPA 修改了我们的部署。无论我们在部署（或 StatefulSets）中定义了多少副本，HPA 都会更改它以适应自己的阈值和计算。换句话说，当我们更新部署时，副本的数量将暂时更改为我们定义的任何内容，然后在几分钟后再次被 HPA 修改。这种行为是不可接受的。

如果 HPA 更改了副本的数量，通常会有很好的理由。将该数字重置为部署（或 StatetifulSet）中设置的任何数字可能会产生严重的副作用。

假设我们在部署中定义了三个副本，并且 HPA 将其扩展到三十个，因为该应用程序的负载增加了。如果我们`apply`部署，因为我们想要推出一个新的版本，那么在短暂的时间内，将会有三个副本，而不是三十个。

因此，我们的用户可能会在我们的应用程序中经历较慢的响应时间，或者由于太少的副本提供了太多的流量而导致其他影响。我们必须努力避免这种情况。副本的数量应始终由 HPA 控制。这意味着我们需要改变我们的策略。

如果在部署中指定副本的数量没有产生我们想要的效果，我们可能会干脆将它们全部删除。让我们看看在这种情况下会发生什么。

我们将使用`go-demo-5.yml`的定义，让我们看看它与我们之前使用的`go-demo-5-replicas-10.yml`有何不同。

```
 1  diff \
 2    scaling/go-demo-5-replicas-10.yml \
 3    scaling/go-demo-5.yml
```

输出显示的唯一区别是，这一次，我们没有指定副本的数量。

让我们应用这个变化，看看会发生什么。

```
 1  kubectl apply \
 2    -f scaling/go-demo-5.yml
 3
 4  kubectl -n go-demo-5 \
 5    describe deployment api
```

后一条命令的输出，仅限于相关部分，如下所示。

```
...
Replicas: 1 desired | 5 updated | 5 total | 5 available | 0 unavailable
...
Events:
... Message
... -------
...
... Scaled down replica set api-5bbfd85577 to 5
... Scaled down replica set api-5bbfd85577 to 1
```

应用部署而没有`副本`导致`1 desired`。当然，HPA 很快会将其扩展到`2`（其最小值），但我们仍然未能实现我们的使命，即始终保持 HPA 定义的副本数量。

我们还能做什么？无论我们是使用`副本`定义还是不使用`副本`定义我们的部署，结果都是一样的。应用部署总是会取消 HPA 的效果，即使我们没有指定`副本`。

实际上，这个说法是不正确的。如果我们知道整个过程是如何工作的，我们可以实现期望的行为而不需要`副本`。

如果为部署定义了`副本`，那么每次我们`应用`一个定义时都会使用它。如果我们通过删除`副本`来更改定义，部署将认为我们想要一个副本，而不是之前的副本数量。但是，如果我们从未指定`副本`的数量，它们将完全由 HPA 控制。

让我们来测试一下。

```
 1  kubectl delete -f scaling/go-demo-5.yml
```

我们删除了与`go-demo-5`应用程序相关的所有内容。现在，让我们测试一下，如果从一开始就没有定义`副本`，部署会如何行为。

```
 1  kubectl apply \
 2    -f scaling/go-demo-5.yml
 3
 4  kubectl -n go-demo-5 \
 5    describe deployment api
```

后一条命令的输出，仅限于相关部分，如下所示。

```
...
Replicas: 1 desired | 1 updated | 1 total | 0 available | 1 unavailable
...
```

看起来我们失败了。部署确实将副本的数量设置为`1`。但是，您看不到的是副本在内部没有定义。

然而，几分钟后，我们的部署将被 HPA 扩展到两个副本。这是预期的行为，但我们将确认一下。

```
 1  kubectl -n go-demo-5 \
 2    describe deployment api
```

您应该从输出中看到副本的数量已经被（由 HPA）更改为`2`。

现在是最终测试。如果我们发布一个新版本的部署，它会缩减到`1`个副本，还是会保持在`2`个副本？

我们将应用一个新的定义。与当前运行的定义相比，唯一的区别在于镜像的标签。这样我们将确保部署确实被更新。

```
 1  kubectl apply \
 2    -f scaling/go-demo-5-2-5.yml
 3
 4  kubectl -n go-demo-5 \
 5    describe deployment api
```

后一条命令的输出，仅限于相关部分，如下所示。

```
...
Replicas: 2 desired | 1 updated | 3 total | 2 available | 1 unavailable
...
Events:
... Message
... -------
... Scaled up replica set api-5bbfd85577 to 1
... Scaled up replica set api-5bbfd85577 to 2
... Scaled up replica set api-745bc9fc6d to 1
```

我们可以看到，由 HPA 设置的副本数量得到了保留。

如果您在“事件”中看到副本的数量被缩减为`1`，不要惊慌。那是部署启动的第二个 ReplicaSet。您可以通过观察 ReplicaSet 的名称来看到这一点。部署正在通过搅动两个 ReplicaSet 来进行滚动更新，以尝试在没有停机时间的情况下推出新版本。这与自动扩展无关，我假设您已经知道滚动更新是如何工作的。如果您不知道，您知道在哪里学习它。

现在出现了关键问题。在部署和有状态集中，我们应该如何定义副本？

如果您计划在部署或 StatefulSet 中使用 HPA，请不要声明副本。如果这样做，每次滚动更新都会暂时取消 HPA 的效果。仅为不与 HPA 一起使用的资源定义副本。

# 现在呢？

我们探讨了扩展部署和 StatefulSets 的最简单方法。这很简单，因为这个机制已经内置在 Kubernetes 中。我们所要做的就是定义一个具有目标内存和 CPU 的 HorizontalPodAutoscaler。虽然这种自动缩放的方法通常被使用，但通常是不够的。并非所有应用程序在压力下都会增加内存或 CPU 使用率。即使它们这样做了，这两个指标可能还不够。

在接下来的章节中，我们将探讨如何扩展 HorizontalPodAutoscaler 以使用自定义的指标来源。现在，我们将销毁我们创建的内容，并开始下一章。

如果您计划保持集群运行，请执行以下命令以删除我们创建的资源。

```
 1  # If NOT GKE or AKS
 2  helm delete metrics-server --purge
 3
 4  kubectl delete ns go-demo-5
```

否则，请删除整个集群，如果您只是为了本书的目的而创建它，并且不打算立即深入下一章。

在您离开之前，您可能希望复习本章的要点。

+   水平 Pod 自动缩放器的唯一功能是自动调整部署、StatefulSet 或其他一些类型的资源中 Pod 的数量。它通过观察 Pod 的 CPU 和内存消耗，并在它们达到预定义的阈值时采取行动来实现这一点。

+   Metrics Server 收集有关节点和 Pod 使用的资源（内存和 CPU）的信息。

+   Metrics Server 定期从运行在节点上的 Kubeletes 获取指标。

+   如果副本的数量是静态的，并且您没有打算随时间缩放（或反向缩放）您的应用程序，请将`replicas`作为部署或 StatefulSet 定义的一部分。另一方面，如果您计划根据内存、CPU 或其他指标更改副本的数量，请改用 HorizontalPodAutoscaler 资源。

+   如果为部署定义了`replicas`，那么每次我们`apply`一个定义时都会使用它。如果我们通过删除`replicas`来更改定义，部署将认为我们想要一个，而不是我们之前拥有的副本数量。但是，如果我们从未指定`replicas`的数量，它们将完全由 HPA 控制。

+   如果您计划在部署或 StatefulSet 中使用 HPA，请不要声明`replicas`。如果这样做，每次滚动更新都会暂时取消 HPA 的效果。仅为不与 HPA 一起使用的资源定义`replicas`。

# Kubernetes 集群的自动缩放节点

我可以说我并没有完全享受与人类一起工作吗？我发现他们的不合逻辑和愚蠢的情绪是一个不断的刺激。

- *斯波克*

使用**HorizontalPodAutoscaler**（**HPA**）是使系统具有弹性、容错和高可用性的最关键方面之一。然而，如果没有可用资源的节点，它就没有用处。当 Kubernetes 无法调度新的 Pod 时，因为没有足够的可用内存或 CPU，新的 Pod 将无法调度并处于挂起状态。如果我们不增加集群的容量，挂起的 Pod 可能会无限期地保持在那种状态。更复杂的是，Kubernetes 可能会开始删除其他 Pod，以为那些处于挂起状态的 Pod 腾出空间。你可能已经猜到，这可能会导致比我们的应用程序没有足够的副本来满足需求的问题更严重的问题。

Kubernetes 通过 Cluster Autoscaler 解决了节点扩展的问题。

Cluster Autoscaler 只有一个目的，那就是通过添加或删除工作节点来调整集群的大小。当 Pod 由于资源不足而无法调度时，它会添加新节点。同样，当节点在一段时间内未被充分利用，并且在该节点上运行的 Pod 可以在其他地方重新调度时，它会删除节点。

Cluster Autoscaler 背后的逻辑很容易理解。我们还没有看到它是否也很容易使用。

让我们创建一个集群（除非您已经有一个），并为其准备自动缩放。

# 创建一个集群

我们将继续使用`vfarcic/k8s-specs`（[`github.com/vfarcic/k8s-specs`](https://github.com/vfarcic/k8s-specs)）存储库中的定义。为了安全起见，我们将首先拉取最新版本。

本章中的所有命令都可以在`02-ca.sh`（[`gist.github.com/vfarcic/a6b2a5132aad6ca05b8ff5033c61a88f`](https://gist.github.com/vfarcic/a6b2a5132aad6ca05b8ff5033c61a88f)）Gist 中找到。

```
 1  cd k8s-specs
 2
 3  git pull
```

接下来，我们需要一个集群。请使用下面的 Gists 作为灵感来创建一个新的集群，或者验证您已经满足所有要求。

AKS 用户注意：在撰写本文时（2018 年 10 月），Cluster Autoscaler 在**Azure Kubernetes Service**（**AKS**）中并不总是有效。请参阅*在 AKS 中设置 Cluster Autoscaler*部分以获取更多信息和设置说明的链接。

+   `gke-scale.sh`：**GKE**有 3 个 n1-standard-1 工作节点，带有**tiller**，并带有`--enable-autoscaling`参数（[`gist.github.com/vfarcic/9c777487f7ebee6c09027d3a1df8663c`](https://gist.github.com/vfarcic/9c777487f7ebee6c09027d3a1df8663c)）。

+   `eks-ca.sh`：**EKS**有 3 个 t2.small 工作节点，带有**tiller**，并带有**Metrics Server**（[`gist.github.com/vfarcic/3dfc71dc687de3ed98e8f804d7abba0b`](https://gist.github.com/vfarcic/3dfc71dc687de3ed98e8f804d7abba0b)）。

+   `aks-scale.sh`：**AKS**有 3 个 Standard_B2s 工作节点和**tiller**（[`gist.github.com/vfarcic/f1b05d33cc8a98e4ceab3d3770c2fe0b`](https://gist.github.com/vfarcic/f1b05d33cc8a98e4ceab3d3770c2fe0b)）。

当检查 Gists 时，你会注意到一些事情。首先，Docker for Desktop 和 minikube 都不在其中。它们都是无法扩展的单节点集群。我们需要在一个可以根据需求添加和删除节点的地方运行集群。我们将不得不使用云供应商之一（例如 AWS、Azure、GCP）。这并不意味着我们不能在本地集群上扩展。

我们可以，但这取决于我们使用的供应商。有些供应商有解决方案，而其他供应商没有。为简单起见，我们将坚持使用三大云供应商之一。请在**Google Kubernetes Engine**（**GKE**）、亚马逊**弹性容器服务**（**EKS**）或**Azure Kubernetes 服务**（**AKS**）之间进行选择。如果你不确定选择哪一个，我建议选择 GKE，因为它是最稳定和功能丰富的托管 Kubernetes 集群。

你还会注意到，GKE 和 AKS 的 Gists 与上一章相同，而 EKS 发生了变化。正如你已经知道的那样，前者已经内置了 Metrics Server。EKS 没有，所以我复制了我们之前使用的 Gist，并添加了安装 Metrics Server 的说明。也许在这一章中我们不需要它，但以后会经常用到，我希望你习惯随时拥有它。

如果你更喜欢在本地运行示例，你可能会因为我们在本章中不使用本地集群而感到沮丧。不要绝望。成本将被保持在最低水平（总共可能只有几美元），我们将在下一章回到本地集群（除非你选择留在云端）。

现在我们在 GKE、EKS 或 AKS 中有了一个集群，我们的下一步是启用集群自动扩展。

# 设置集群自动扩展

在开始使用之前，我们可能需要安装集群自动缩放器。我说“可能”，而不是说“必须”，因为某些 Kubernetes 版本确实预先配置了集群自动缩放器，而其他版本则没有。我们将逐个讨论“三大”托管 Kubernetes 集群。您可以选择探索它们三个，或者直接跳转到您喜欢的一个。作为学习经验，我认为体验在所有三个提供商中运行 Kubernetes 是有益的。尽管如此，这可能不是您的观点，您可能更喜欢只使用一个。选择权在您手中。

# 在 GKE 中设置集群自动缩放器

这将是有史以来最短的部分。如果在创建集群时指定了`--enable-autoscaling`参数，则在 GKE 中无需进行任何操作。它已经预先配置并准备好了集群自动缩放器。

# 在 EKS 中设置集群自动缩放器

与 GKE 不同，EKS 不带有集群自动缩放器。我们将不得不自己配置它。我们需要向专用于工作节点的 Autoscaling Group 添加一些标签，为我们正在使用的角色添加额外的权限，并安装集群自动缩放器。

让我们开始吧。

我们将向专用于工作节点的 Autoscaling Group 添加一些标签。为此，我们需要发现组的名称。由于我们使用**eksctl**创建了集群，名称遵循一种模式，我们可以使用该模式来过滤结果。另一方面，如果您在没有使用 eksctl 的情况下创建了 EKS 集群，逻辑应该与接下来的逻辑相同，尽管命令可能略有不同。

首先，我们将检索 AWS Autoscaling Groups 的列表，并使用`jq`过滤结果，以便只返回匹配组的名称。

```
 1  export NAME=devops25
 2
 3  ASG_NAME=$(aws autoscaling \
 4      describe-auto-scaling-groups \
 5      | jq -r ".AutoScalingGroups[] \
 6      | select(.AutoScalingGroupName \
 7      | startswith(\"eksctl-$NAME-nodegroup\")) \
 8      .AutoScalingGroupName")
 9
10 echo $ASG_NAME
```

后一个命令的输出应该类似于接下来的输出。

```
eksctl-devops25-nodegroup-0-NodeGroup-1KWSL5SEH9L1Y
```

我们将集群的名称存储在环境变量`NAME`中。然后，我们检索了所有组的列表，并使用`jq`过滤输出，以便只返回名称以`eksctl-$NAME-nodegroup`开头的组。最后，相同的`jq`命令检索了`AutoScalingGroupName`字段，并将其存储在环境变量`ASG_NAME`中。最后一个命令输出了组名，以便我们可以确认（视觉上）它看起来是否正确。

接下来，我们将向组添加一些标记。Kubernetes Cluster Autoscaler 将与具有`k8s.io/cluster-autoscaler/enabled`和`kubernetes.io/cluster/[NAME_OF_THE_CLUSTER]`标记的组一起工作。因此，我们只需添加这些标记，让 Kubernetes 知道要使用哪个组。

```
 1  aws autoscaling \
 2      create-or-update-tags \
 3      --tags \
 4      ResourceId=$ASG_NAME,ResourceType=auto-scaling-group,Key=k8s.io/
    clusterautoscaler/enabled,Value=true,PropagateAtLaunch=true \
 5      ResourceId=$ASG_NAME,ResourceType=auto-scaling-
    group,Key=kubernetes.io/cluster/$NAME,Value=true,PropagateAtLaunch=true
```

我们在 AWS 中需要做的最后一项更改是向通过 eksctl 创建的角色添加一些额外的权限。与自动缩放组一样，我们不知道角色的名称，但我们知道用于创建它的模式。因此，在添加新策略之前，我们将检索角色的名称。

```
 1  IAM_ROLE=$(aws iam list-roles \
 2      | jq -r ".Roles[] \
 3      | select(.RoleName \
 4      | startswith(\"eksctl-$NAME-nodegroup-0-NodeInstanceRole\")) \
 5      .RoleName")
 6  
 7  echo $IAM_ROLE
```

后一条命令的输出应该类似于接下来的输出。

```
eksctl-devops25-nodegroup-0-NodeInstanceRole-UU6CKXYESUES
```

我们列出了所有角色，并使用`jq`过滤输出，以便只返回名称以`eksctl-$NAME-nodegroup-0-NodeInstanceRole`开头的角色。过滤角色后，我们检索了`RoleName`并将其存储在环境变量`IAM_ROLE`中。

接下来，我们需要描述新策略的 JSON。我已经准备好了，让我们快速看一下。

```
 1  cat scaling/eks-autoscaling-policy.json
```

输出如下。

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup"
      ],
      "Resource": "*"
    }
  ]
}
```

如果你熟悉 AWS（我希望你是），那个策略应该很简单。它允许与`autoscaling`相关的一些额外操作。

最后，我们可以将新策略`put`到角色中。

```
 1  aws iam put-role-policy \
 2      --role-name $IAM_ROLE \
 3      --policy-name $NAME-AutoScaling \
 4      --policy-document file://scaling/eks-autoscaling-policy.json
```

现在我们已经向自动缩放组添加了所需的标记，并创建了额外的权限，允许 Kubernetes 与该组进行交互，我们可以安装 Cluster Autoscaler Helm Chart。

```
 1  helm install stable/cluster-autoscaler \
 2      --name aws-cluster-autoscaler \
 3      --namespace kube-system \
 4      --set autoDiscovery.clusterName=$NAME \
 5      --set awsRegion=$AWS_DEFAULT_REGION \
 6      --set sslCertPath=/etc/kubernetes/pki/ca.crt \
 7      --set rbac.create=true
 8
9  kubectl -n kube-system \
10      rollout status \
11      deployment aws-cluster-autoscaler
```

一旦部署完成，自动缩放器应该完全可用。

# 在 AKS 中设置 Cluster Autoscaler

在撰写本文时（2018 年 10 月），Cluster Autoscaler 在 AKS 中无法正常工作。至少，不总是。它仍处于测试阶段，我暂时不能推荐。希望它很快就能完全运行并保持稳定。一旦发生这种情况，我将使用 AKS 特定的说明更新本章。如果你感到有冒险精神，或者你致力于 Azure，请按照*Azure Kubernetes Service（AKS）上的 Cluster Autoscaler - 预览*（[`docs.microsoft.com/en-in/azure/aks/cluster-autoscaler`](https://docs.microsoft.com/en-in/azure/aks/cluster-autoscaler)）文章中的说明。如果它有效，你应该能够按照本章的其余部分进行操作。

# 扩大集群

我们的目标是扩展集群的节点，以满足 Pod 的需求。我们不仅希望在需要额外容量时增加工作节点的数量，而且在它们被闲置时也要删除它们。现在，我们将专注于前者，并在之后探索后者。

让我们首先看一下集群中有多少个节点。

```
 1  kubectl get nodes
```

来自 GKE 的输出如下。

```
NAME             STATUS ROLES  AGE   VERSION
gke-devops25-... Ready  <none> 5m27s v1.9.7-gke.6
gke-devops25-... Ready  <none> 5m28s v1.9.7-gke.6
gke-devops25-... Ready  <none> 5m24s v1.9.7-gke.6
```

在您的情况下，节点的数量可能会有所不同。这并不重要。重要的是要记住您现在有多少个节点，因为这个数字很快就会改变。

在我们推出`go-demo-5`应用程序之前，让我们先看一下它的定义。

```
 1  cat scaling/go-demo-5-many.yml
```

输出内容，仅限于相关部分，如下所示。

```
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
            memory: 1Gi
            cpu: 0.1
          requests:
            memory: 500Mi
            cpu: 0.01
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
  minReplicas: 15
  maxReplicas: 30
  ...
```

在这种情况下，我们即将应用的定义中唯一重要的部分是与`api`部署连接的 HPA。它的最小副本数是`15`。假设每个`api`容器请求 500 MB RAM，那么十五个副本（7.5 GB RAM）应该超出了我们的集群可以承受的范围，假设它是使用其中一个 Gists 创建的。否则，您可能需要增加最小副本数。

让我们应用这个定义并看一下 HPA。

```
 1  kubectl apply \
 2      -f scaling/go-demo-5-many.yml \
 3      --record
 4
 5  kubectl -n go-demo-5 get hpa
```

后一条命令的输出如下。

```
NAME   REFERENCE        TARGETS                        MINPODS   MAXPODS   REPLICAS   AGE
api    Deployment/api   <unknown>/80%, <unknown>/80%   15        30        1          38s
db     StatefulSet/db   <unknown>/80%, <unknown>/80%   3         5         1          40s
```

无论目标是否仍然是`未知`，它们很快就会被计算出来，但我们现在不关心它们。重要的是`api` HPA 将会将部署扩展至至少`15`个副本。

接下来，我们需要等待几秒钟，然后再看一下`go-demo-5`命名空间中的 Pod。

```
 1  kubectl -n go-demo-5 get pods
```

输出如下。

```
NAME    READY STATUS            RESTARTS AGE
api-... 0/1   ContainerCreating 0        2s
api-... 0/1   Pending           0        2s
api-... 0/1   Pending           0        2s
api-... 0/1   ContainerCreating 0        2s
api-... 0/1   ContainerCreating 0        2s
api-... 0/1   ContainerCreating 1        32s
api-... 0/1   Pending           0        2s
api-... 0/1   ContainerCreating 0        2s
api-... 0/1   ContainerCreating 0        2s
api-... 0/1   ContainerCreating 0        2s
api-... 0/1   ContainerCreating 0        2s
api-... 0/1   ContainerCreating 0        2s
api-... 0/1   Pending           0        2s
api-... 0/1   ContainerCreating 0        2s
api-... 0/1   ContainerCreating 0        2s
db-0    2/2   Running           0        34s
db-1    0/2   ContainerCreating 0        34s
```

我们可以看到一些`api` Pod 正在被创建，而其他一些则是挂起的。Pod 进入挂起状态可能有很多原因。

在我们的情况下，没有足够的可用资源来托管所有的 Pod。

![](img/a1c839db-c439-4994-a113-8b0a27c59e84.png)图 2-1：无法调度（挂起）的 Pod 正在等待集群容量增加

让我们看看集群自动缩放器是否有助于解决我们的容量不足问题。我们将探索包含集群自动缩放器状态的 ConfigMap。

```
 1  kubectl -n kube-system get cm \
 2      cluster-autoscaler-status \
 3      -o yaml
```

输出内容太多，无法完整呈现，所以我们将专注于重要的部分。

```
apiVersion: v1
data:
  status: |+
    Cluster-autoscaler status at 2018-10-03 ...
    Cluster-wide:
      ...
      ScaleUp: InProgress (ready=3 registered=3)
    ... 
    NodeGroups:
      Name:    ...gke-devops25-default-pool-ce277413-grp
      ...
      ScaleUp: InProgress (ready=1 cloudProviderTarget=2)
               ...
```

状态分为两个部分：`整个集群`和`节点组`。整个集群状态的`ScaleUp`部分显示缩放正在进行中。目前有`3`个就绪节点。

如果我们移动到`NodeGroups`，我们会注意到每个托管我们节点的组都有一个。在 AWS 中，这些组映射到自动缩放组，在谷歌的情况下映射到实例组，在 Azure 中映射到自动缩放。配置中的一个`NodeGroups`具有`ScaleUp`部分`InProgress`。在该组内，`1`个节点是`ready`。`cloudProviderTarget`值应设置为高于`ready`节点数量的数字，我们可以得出结论，集群自动缩放器已经增加了该组中所需的节点数量。

根据提供商的不同，您可能会看到三个组（GKE）或一个（EKS）节点组。这取决于每个提供商如何在内部组织其节点组。

现在我们知道集群自动缩放器正在进行节点扩展，我们可以探索是什么触发了该操作。

让我们描述`api` Pod 并检索它们的事件。由于我们只想要与`cluster-autoscaler`相关的事件，我们将使用`grep`来限制输出。

```
 1  kubectl -n go-demo-5 \
 2      describe pods \
 3      -l app=api \
 4      | grep cluster-autoscaler
```

在 GKE 上的输出如下。

```
  Normal TriggeredScaleUp 85s cluster-autoscaler pod triggered scale-up: [{... 1->2 (max: 3)}]
  Normal TriggeredScaleUp 86s cluster-autoscaler pod triggered scale-up: [{... 1->2 (max: 3)}]
  Normal TriggeredScaleUp 87s cluster-autoscaler pod triggered scale-up: [{... 1->2 (max: 3)}]
  Normal TriggeredScaleUp 88s cluster-autoscaler pod triggered scale-up: [{... 1->2 (max: 3)}]
```

我们可以看到几个 Pod 触发了`scale-up`事件。这些是处于挂起状态的 Pod。这并不意味着每个触发都创建了一个新节点。集群自动缩放器足够智能，知道不应该为每个触发创建新节点，但在这种情况下，一个或两个节点（取决于缺少的容量）应该足够。如果证明这是错误的，它将在一段时间后再次扩展。

让我们检索构成集群的节点，看看是否有任何变化。

```
 1  kubectl get nodes
```

输出如下。

```
NAME                                     STATUS     ROLES    AGE     VERSION
gke-devops25-default-pool-7d4b99ad-...   Ready      <none>   2m45s   v1.9.7-gke.6
gke-devops25-default-pool-cb207043-...   Ready      <none>   2m45s   v1.9.7-gke.6
gke-devops25-default-pool-ce277413-...   NotReady   <none>   12s     v1.9.7-gke.6
gke-devops25-default-pool-ce277413-...   Ready      <none>   2m48s   v1.9.7-gke.6
```

我们可以看到一个新的工作节点被添加到集群中。它还没有准备好，所以我们需要等待一段时间，直到它完全可操作。

请注意，新节点的数量取决于托管所有 Pod 所需的容量。您可能会看到一个、两个或更多新节点。

![](img/27fff93a-5bb4-452a-bc2f-e480ec2c725a.png)图 2-2：集群自动缩放器扩展节点的过程

现在，让我们看看我们的 Pod 发生了什么。记住，上次我们检查它们时，有相当多的 Pod 处于挂起状态。

```
 1  kubectl -n go-demo-5 get pods
```

输出如下。

```
NAME    READY STATUS  RESTARTS AGE
api-... 1/1   Running 1        75s
api-... 1/1   Running 0        75s
api-... 1/1   Running 0        75s
api-... 1/1   Running 1        75s
api-... 1/1   Running 1        75s
api-... 1/1   Running 3        105s
api-... 1/1   Running 0        75s
api-... 1/1   Running 0        75s
api-... 1/1   Running 1        75s
api-... 1/1   Running 1        75s
api-... 1/1   Running 0        75s
api-... 1/1   Running 1        75s
api-... 1/1   Running 0        75s
api-... 1/1   Running 1        75s
api-... 1/1   Running 0        75s
db-0    2/2   Running 0        107s
db-1    2/2   Running 0        67s
db-2    2/2   Running 0        28s
```

集群自动缩放器增加了节点组（例如，AWS 中的自动缩放组）中所需的节点数量，从而创建了一个新节点。一旦调度程序注意到集群容量的增加，它就会将待定的 Pod 调度到新节点中。在几分钟内，我们的集群扩展了，所有缩放的 Pod 都在运行。

![](img/3126993e-31ae-4184-bee7-068f2752fa4c.png)图 2-3：通过节点组创建新节点和挂起 Pod 的重新调度

那么，集群自动缩放器在何时决定扩大节点的规则是什么？

# 节点规模扩大的规则

集群自动缩放器通过对 Kube API 进行监视来监视 Pod。它每 10 秒检查一次是否有任何无法调度的 Pod（可通过`--scan-interval`标志进行配置）。在这种情况下，当 Kubernetes 调度程序无法找到可以容纳它的节点时，Pod 是无法调度的。例如，一个 Pod 可以请求比任何工作节点上可用的内存更多的内存。

集群自动缩放器假设集群运行在某种节点组之上。例如，在 AWS 的情况下，这些组是**自动缩放组**（**ASGs**）。当需要额外的节点时，集群自动缩放器通过增加节点组的大小来创建一个新节点。

集群自动缩放器假设请求的节点将在 15 分钟内出现（可通过`--max-node-provision-time`标志进行配置）。如果该时间段到期，新节点未注册，它将尝试扩展不同的组，如果 Pod 仍处于挂起状态。它还将在 15 分钟后删除未注册的节点（可通过`--unregistered-node-removal-time`标志进行配置）。

接下来，我们将探讨如何缩小集群。

# 缩小集群

扩大集群以满足需求是必不可少的，因为它允许我们托管我们需要满足（部分）SLA 的所有副本。当需求下降，我们的节点变得未充分利用时，我们应该缩小规模。鉴于我们的用户不会因为集群中有太多硬件而遇到问题，这并非必要。然而，如果我们要减少开支，我们不应该有未充分利用的节点。未使用的节点会导致浪费。这在所有情况下都是正确的，特别是在云中运行并且只支付我们使用的资源的情况下。即使在本地，我们已经购买了硬件，缩小规模并释放资源以便其他集群使用是必不可少的。

我们将通过应用一个新的定义来模拟需求下降，这将重新定义 HPAs 的阈值为`2`（最小）和`5`（最大）。

```
 1  kubectl apply \
 2      -f scaling/go-demo-5.yml \
 3      --record
 4
 5  kubectl -n go-demo-5 get hpa
```

后一条命令的输出如下。

```
NAME REFERENCE      TARGETS          MINPODS MAXPODS REPLICAS AGE
api  Deployment/api 0%/80%, 0%/80%   2       5       15       2m56s
db   StatefulSet/db 56%/80%, 10%/80% 3       5       3        2m57s
```

我们可以看到`api` HPA 的最小和最大值已经改变为`2`和`5`。当前副本的数量仍然是`15`，但很快会降到`5`。HPA 已经改变了部署的副本，所以让我们等待它的部署完成，然后再看一下 Pods。

```
 1  kubectl -n go-demo-5 rollout status \
 2      deployment api
 3
 4  kubectl -n go-demo-5 get pods
```

后一个命令的输出如下。

```
NAME    READY STATUS  RESTARTS AGE
api-... 1/1   Running 0        104s
api-... 1/1   Running 0        104s
api-... 1/1   Running 0        104s
api-... 1/1   Running 0        94s
api-... 1/1   Running 0        104s
db-0    2/2   Running 0        4m37s
db-1    2/2   Running 0        3m57s
db-2    2/2   Running 0        3m18s
```

让我们看看`nodes`发生了什么。

```
 1  kubectl get nodes
```

输出显示我们仍然有四个节点（或者在我们缩减部署之前的数字）。

考虑到我们还没有达到只有三个节点的期望状态，我们可能需要再看一下`cluster-autoscaler-status` ConfigMap。

```
 1  kubectl -n kube-system \
 2      get configmap \
 3      cluster-autoscaler-status \
 4      -o yaml
```

输出，仅限于相关部分，如下所示。

```
apiVersion: v1
data:
  status: |+
    Cluster-autoscaler status at 2018-10-03 ...
    Cluster-wide:
      Health: Healthy (ready=4 ...)
      ...
      ScaleDown: CandidatesPresent (candidates=1)
                 ...
    NodeGroups:
      Name:      ...gke-devops25-default-pool-f4c233dd-grp
      ...
      ScaleDown: CandidatesPresent (candidates=1)
                 LastProbeTime:      2018-10-03 23:06:...
                 LastTransitionTime: 2018-10-03 23:05:...
      ...
```

如果您的输出不包含`ScaleDown: CandidatesPresent`，您可能需要等一会儿并重复上一个命令。

如果我们关注整个集群状态的`Health`部分，所有四个节点仍然是就绪的。

从状态的整个集群部分来看，我们可以看到有一个候选节点进行`ScaleDown`（在您的情况下可能有更多）。如果我们转到`NodeGroups`，我们可以观察到其中一个节点组在`ScaleDown`部分中的`CandidatesPresent`设置为`1`（或者在扩展之前的初始值）。

换句话说，其中一个节点是待删除的候选节点。如果它保持这样十分钟，节点将首先被排空，以允许其中运行的 Pods 优雅关闭。之后，通过操纵扩展组来物理移除它。

![](img/ede07cff-dc03-427e-9205-4facebd4711f.png)图 2-4：集群自动缩放的缩减过程

在继续之前，我们应该等待十分钟，所以这是一个很好的机会去喝杯咖啡（或茶）。

现在已经过了足够的时间，我们将再次查看`cluster-autoscaler-status` ConfigMap。

```
 1  kubectl -n kube-system \
 2      get configmap \
 3      cluster-autoscaler-status \
 4      -o yaml
```

输出，仅限于相关部分，如下所示。

```
apiVersion: v1
data:
  status: |+
    Cluster-autoscaler status at 2018-10-03 23:16:24...
    Cluster-wide:
      Health:    Healthy (ready=3 ... registered=4 ...)
                 ...
      ScaleDown: NoCandidates (candidates=0)
                 ...
    NodeGroups:
      Name:      ...gke-devops25-default-pool-f4c233dd-grp
      Health:    Healthy (ready=1 ... registered=2 ...)
                 ...
      ScaleDown: NoCandidates (candidates=0)
                 ...
```

从整个集群部分，我们可以看到现在有`3`个就绪节点，但仍然有`4`（或更多）已注册。这意味着其中一个节点已经被排空，但仍然没有被销毁。同样，其中一个节点组显示有`1`个就绪节点，尽管已注册`2`个（您的数字可能有所不同）。

从 Kubernetes 的角度来看，我们回到了三个操作节点，尽管第四个节点仍然存在。

现在我们需要再等一会儿，然后检索节点并确认只有三个可用。

```
 1  kubectl get nodes
```

来自 GKE 的输出如下。

```
NAME    STATUS ROLES  AGE VERSION
gke-... Ready  <none> 36m v1.9.7-gke.6
gke-... Ready  <none> 36m v1.9.7-gke.6
gke-... Ready  <none> 36m v1.9.7-gke.6
```

我们可以看到节点已被移除，我们已经从过去的经验中知道，Kube Scheduler 将那个节点中的 Pod 移动到仍在运行的节点中。现在您已经经历了节点的缩减，我们将探讨管理该过程的规则。

# 控制节点缩减的规则

集群自动缩放器每 10 秒迭代一次（可通过`--scan-interval`标志进行配置）。如果不满足扩展的条件，它会检查是否有不需要的节点。

当满足以下所有条件时，它将考虑将节点标记为可移除。

+   在节点上运行的所有 Pod 的 CPU 和内存请求总和小于节点可分配资源的 50%（可通过`--scale-down-utilization-threshold`标志进行配置）。

+   所有在节点上运行的 Pod 都可以移动到其他节点。例外情况是那些在所有节点上运行的 Pod，比如通过 DaemonSets 创建的 Pod。

当满足以下条件之一时，Pod 可能不符合重新调度到不同节点的条件。

+   具有亲和性或反亲和性规则将其与特定节点绑定的 Pod。

+   使用本地存储的 Pod。

+   直接创建的 Pod，而不是通过部署、有状态集、作业或副本集等控制器创建的 Pod。

所有这些规则归结为一个简单的规则。如果一个节点包含一个不能安全驱逐的 Pod，那么它就不符合移除的条件。

接下来，我们应该谈谈集群扩展的边界。

# 我们是否可以扩展得太多或将节点缩减到零？

如果让集群自动缩放器在不定义任何阈值的情况下进行"魔术"，我们的集群或钱包可能会面临风险。

例如，我们可能会错误配置 HPA，导致将部署或有状态集扩展到大量副本。结果，集群自动缩放器可能会向集群添加过多的节点。因此，我们可能会支付数百个节点的费用，尽管我们实际上需要的要少得多。幸运的是，AWS、Azure 和 GCP 限制了我们可以拥有的节点数量，因此我们无法无限扩展。尽管如此，我们也不应允许集群自动缩放器超出一些限制。

同样，集群自动缩放器可能会缩减到太少的节点。拥有零个节点几乎是不可能的，因为这意味着我们在集群中没有 Pod。尽管如此，我们应该保持健康的最小节点数量，即使有时会被低效利用。

节点的合理最小数量是三个。这样，我们在该地区的每个区域（数据中心）都有一个工作节点。正如您已经知道的，Kubernetes 需要三个带有主节点的区域来维持法定人数。在某些情况下，特别是在本地，我们可能只有一个地理上相邻的延迟较低的数据中心。在这种情况下，一个区域（数据中心）总比没有好。但是，在云服务提供商的情况下，三个区域是推荐的分布，并且在每个区域至少有一个工作节点是有意义的。如果我们使用块存储，这一点尤为重要。

根据其性质，块存储（例如 AWS 中的 EBS、GCP 中的持久磁盘和 Azure 中的块 Blob）无法从一个区域移动到另一个区域。这意味着我们必须在每个区域都有一个工作节点，以便（很可能）总是有一个与存储在同一区域的位置。当然，如果我们不使用块存储，那么这个论点就站不住脚了。

那么工作节点的最大数量呢？嗯，这取决于不同的用例。您不必永远坚持相同的最大值。它可以随着时间的推移而改变。

作为一个经验法则，我建议将最大值设为实际节点数量的两倍。但是，不要太认真对待这个规则。这确实取决于您的集群大小。如果您只有三个工作节点，那么最大尺寸可能是九个（三倍）。另一方面，如果您有数百甚至数千个节点，将该数字加倍作为最大值就没有意义。那将太多了。只需确保节点的最大数量反映了需求的潜在增长。

无论如何，我相信您会弄清楚您的工作节点的最小和最大数量应该是多少。如果您犯了错误，可以随后更正。更重要的是如何定义这些阈值。

幸运的是，在 EKS、GKE 和 AKS 中设置最小和最大值很容易。对于 EKS，如果您使用`eksctl`来创建集群，我们只需在`eksctl create cluster`命令中添加`--nodes-min`和`--nodes-max`参数。GKE 遵循类似的逻辑，使用`gcloud container clusters create`命令的`--min-nodes`和`--max-nodes`参数。如果其中一个是您的首选项，那么如果您遵循了 Gists，您已经使用了这些参数。即使您忘记指定它们，您也可以随时修改自动缩放组（AWS）或实例组（GCP），因为实际应用限制的地方就在那里。

Azure 采取了稍微不同的方法。我们直接在`cluster-autoscaler`部署中定义其限制，并且可以通过应用新的定义来更改它们。

# 在 GKE、EKS 和 AKS 中比较的集群自动缩放器

集群自动缩放器是不同托管 Kubernetes 服务提供商之间差异的一个主要例子。我们将使用它来比较三个主要的 Kubernetes 即服务提供商。

我将把供应商之间的比较限制在与集群自动缩放相关的主题上。

对于那些可以使用谷歌来托管他们的集群的人来说，GKE 是一个不言而喻的选择。它是最成熟和功能丰富的平台。他们比其他人早很久就开始了**Google Kubernetes Engine**（**GKE**）。当我们将他们的领先优势与他们是 Kubernetes 的主要贡献者并且因此拥有最丰富经验这一事实结合起来时，他们的产品远远超过其他人并不足为奇。

在使用 GKE 时，一切都包含在集群中。这包括集群自动缩放器。我们不必执行任何额外的命令。只要我们在创建集群时指定`--enable-autoscaling`参数，它就可以直接使用。此外，GKE 比其他提供商更快地启动新节点并将它们加入集群。如果需要扩展集群，新节点将在一分钟内添加。

我会推荐 GKE 的许多其他原因，但现在不是讨论的主题。不过，单单集群自动缩放就足以证明 GKE 是其他人努力追随的解决方案。

亚马逊的**弹性容器服务**（**EKS**）处于中间位置。集群自动缩放器可以工作，但它并不是内置的。就好像亚马逊认为扩展集群并不重要，所以将其作为一个可选的附加组件。

与 GKE 和 AKS 相比，EKS 的安装过于复杂，但多亏了来自 Weaveworks 的 eksctl（[`eksctl.io/`](https://eksctl.io/)），我们解决了这个问题。不过，eksctl 还有很多需要改进的地方。例如，我们无法使用它来升级我们的集群。

我提到 eksctl 是在自动缩放设置的上下文中。

我不能说在 EKS 中设置集群自动缩放器很难。并不是。然而，它并不像应该的那么简单。我们需要给自动缩放组打标签，为角色添加额外的权限，并安装集群自动缩放器。这并不多。然而，这些步骤比应该的复杂得多。我们可以拿 GKE 来比较。谷歌明白自动缩放 Kubernetes 集群是必须的，并提供了一个参数（或者如果你更喜欢 UI，可以选择一个复选框）。而 AWS 则认为自动缩放并不重要，没有给我们那么简单的设置。除了 EKS 中不必要的设置之外，事实上 AWS 最近才添加了扩展所需的内部组件。Metrics Server 只能在 2018 年 9 月之后使用。

我怀疑 AWS 并不急于让 EKS 变得更好，而是把改进留给了 Fargate。如果是这样的话（我们很快就会知道），我会把它称为“隐秘的商业行为”。Kubernetes 拥有所有扩展 Pod 和节点所需的工具，并且它们被设计为可扩展的。选择不将集群自动缩放器作为托管 Kubernetes 服务的一部分是一个很大的缺点。

AKS 有什么好说的呢？我钦佩微软在 Azure 上所做的改进，以及他们对 Kubernetes 的贡献。他们确实意识到了提供一个良好的托管 Kubernetes 的需求。然而，集群自动缩放器仍处于测试阶段。有时它能正常工作，但更多时候却不能。即使它正常工作，速度也很慢。等待新节点加入集群需要耐心等待。

在 AKS 中安装集群自动缩放器所需的步骤有些荒谬。我们需要定义大量参数，而这些参数本应该已经在集群内可用。它应该知道集群的名称，资源组的名称等等。然而，它并不知道。至少在撰写本文时是这样的（2018 年 10 月）。我希望随着时间的推移，这个过程和体验会得到改善。目前来看，就自动缩放的角度来看，AKS 处于队伍的最后。

你可能会说设置的复杂性并不重要。你说得对。重要的是集群自动缩放器的可靠性以及它添加新节点到集群的速度。然而，情况却是一样的。GKE 在可靠性和速度方面处于领先地位。EKS 紧随其后，而 AKS 则落后。

# 现在呢？

关于集群自动缩放器没有太多要说的了。

我们已经探索了自动缩放 Pod 和节点的基本方法。很快我们将深入探讨更复杂的主题，并探索那些没有“内置”到 Kubernetes 集群中的东西。我们将超越核心项目，并介绍一些新的工具和流程。

如果您不打算立即进入下一章，并且您的集群是可丢弃的（例如，不在裸机上），那么这就是您应该销毁集群的时刻。否则，请删除`go-demo-5`命名空间，以删除我们在本章中创建的资源。

```
 1  kubectl delete ns go-demo-5
```

在您离开之前，您可能希望复习本章的要点。

+   集群自动缩放器有一个单一的目的，即通过添加或删除工作节点来调整集群的大小。当 Pod 由于资源不足而无法调度时，它会添加新节点。同样，当节点在一段时间内利用不足，并且运行在该节点上的 Pod 可以在其他地方重新调度时，它会消除节点。

+   集群自动缩放器假设集群正在某种节点组之上运行。例如，在 AWS 的情况下，这些组是自动缩放组（ASG）。当需要额外的节点时，集群自动缩放器通过增加节点组的大小来创建新节点。

+   当运行在节点上的所有 Pod 的 CPU 和内存请求总和小于节点可分配资源的 50％时，集群将被缩减，并且当运行在节点上的所有 Pod 可以移动到其他节点时（DamonSets 是例外情况）。

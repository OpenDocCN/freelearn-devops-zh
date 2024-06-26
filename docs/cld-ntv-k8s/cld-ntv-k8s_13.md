# 第十章：排除故障的 Kubernetes

本章将审查有效排除 Kubernetes 集群和运行在其中的应用程序的最佳实践方法。这包括讨论常见的 Kubernetes 问题，以及如何分别调试主节点和工作节点。常见的 Kubernetes 问题将以案例研究的形式进行讨论和教学，分为集群问题和应用程序问题。

我们将首先讨论一些常见的 Kubernetes 故障模式，然后再讨论如何最好地排除集群和应用程序的故障。

在本章中，我们将涵盖以下主题：

+   理解分布式应用的故障模式

+   排除故障的 Kubernetes 集群

+   在 Kubernetes 上排除故障

# 技术要求

为了运行本章中详细介绍的命令，您需要一台支持`kubectl`命令行工具的计算机，以及一个正常运行的 Kubernetes 集群。请参阅*第一章*，*与 Kubernetes 通信*，了解快速启动和运行 Kubernetes 的几种方法，以及如何安装`kubectl`工具的说明。

本章中使用的代码可以在书籍的 GitHub 存储库中找到，网址为[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter10`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter10)。

# 理解分布式应用的故障模式

默认情况下，Kubernetes 组件（以及在 Kubernetes 上运行的应用程序）是分布式的，如果它们运行多个副本。这可能导致一些有趣的故障模式，这些故障模式可能很难调试。

因此，如果应用程序是无状态的，它们在 Kubernetes 上就不太容易失败-在这种情况下，状态被卸载到在 Kubernetes 之外运行的缓存或数据库中。Kubernetes 的原语，如 StatefulSets 和 PersistentVolumes，可以使在 Kubernetes 上运行有状态的应用程序变得更加容易-并且随着每个版本的发布，在 Kubernetes 上运行有状态的应用程序的体验也在不断改善。然而，决定在 Kubernetes 上运行完全有状态的应用程序会引入复杂性，因此也会增加失败的可能性。

分布式应用程序的故障可能由许多不同的因素引起。诸如网络可靠性和带宽限制等简单事物可能会导致重大问题。这些问题如此多样化，以至于*Sun Microsystems*的*Peter Deutsch*帮助撰写了*分布式计算的谬论*（连同*James Gosling*一起添加了第八点），这些谬论是关于分布式应用程序失败的共识因素。在论文*解释分布式计算的谬论*中，*Arnon Rotem-Gal-Oz*讨论了这些谬论的来源。

这些谬论按照数字顺序如下：

1.  网络是可靠的。

1.  延迟为零。

1.  带宽是无限的。

1.  网络是安全的。

1.  拓扑结构不会改变。

1.  只有一个管理员。

1.  传输成本为零。

1.  网络是同质的。

Kubernetes 在设计和开发时考虑了这些谬论，因此更具有容忍性。它还有助于解决在 Kubernetes 上运行的应用程序的这些问题-但并非完美。因此，当您的应用程序在 Kubernetes 上进行容器化并运行时，很可能会在面对这些问题时出现问题。每个谬论，当假设为不真实并推导到其逻辑结论时，都可能在分布式应用程序中引入故障模式。让我们逐个讨论 Kubernetes 和在 Kubernetes 上运行的应用程序的每个谬论。

## 网络是可靠的。

在多个逻辑机器上运行的应用程序必须通过互联网进行通信-因此，网络中的任何可靠性问题都可能引入问题。特别是在 Kubernetes 上，控制平面本身可以在高度可用的设置中进行分布（这意味着具有多个主节点的设置-请参见*第一章*，*与 Kubernetes 通信*），这意味着故障模式可能会在控制器级别引入。如果网络不可靠，那么 kubelet 可能无法与控制平面进行通信，从而导致 Pod 放置问题。

同样，控制平面的节点可能无法彼此通信-尽管`etcd`当然是使用一致性协议构建的，可以容忍通信故障。

最后，工作节点可能无法彼此通信-在微服务场景中，这可能会根据 Pod 的放置而引起问题。在某些情况下，工作节点可能都能够与控制平面通信，但仍然无法彼此通信，这可能会导致 Kubernetes 叠加网络出现问题。

与一般的不可靠性一样，延迟也可能引起许多相同的问题。

## 延迟是零

如果网络延迟显着，许多与网络不可靠性相同的故障也会适用。例如，kubelet 和控制平面之间的调用可能会失败，导致`etcd`中出现不准确的时期，因为控制平面可能无法联系 kubelet-或者正确更新`etcd`。同样，如果运行在工作节点上的应用程序之间的请求丢失，否则如果这些应用程序在同一节点上共存，则可以完美运行。

## 带宽是无限的

带宽限制可能会暴露与前两个谬论类似的问题。Kubernetes 目前没有完全支持的方法来基于带宽订阅来放置 Pod。这意味着达到网络带宽限制的节点仍然可以安排新的 Pod，导致请求的失败率和延迟问题增加。已经有要求将此作为核心 Kubernetes 调度功能添加的请求（基本上，一种根据节点带宽消耗进行调度的方法，就像 CPU 和内存一样），但目前，解决方案大多受限于容器网络接口（CNI）插件。

重要提示

举例来说，CNI 带宽插件支持在 Pod 级别进行流量整形-请参阅[`kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#support-traffic-shaping`](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#support-traffic-shaping)。

第三方 Kubernetes 网络实现也可能提供围绕带宽的附加功能-并且许多与 CNI 带宽插件兼容。

## 网络是安全的

网络安全的影响远不止于 Kubernetes——因为任何不安全的网络都可能遭受各种攻击。攻击者可能能够获得对 Kubernetes 集群中的主节点或工作节点的 SSH 访问权限，这可能会造成重大破坏。由于 Kubernetes 的许多功能都是通过网络而不是在单台机器上完成的，因此在攻击情况下对网络的访问会变得更加棘手。

## 拓扑结构不会改变

这种谬误在 Kubernetes 的背景下尤为重要，因为不仅可以通过添加和移除新节点来改变元网络拓扑结构，覆盖网络拓扑结构也会直接受到 Kubernetes 控制平面和 CNI 的影响。

因此，一个应用程序在某一时刻在一个逻辑位置运行，可能在网络中的完全不同位置运行。因此，使用 Pod IP 来识别逻辑应用程序是一个不好的主意——这是服务抽象的一个目的（参见*第五章*，*服务和入口*——*与外部世界通信*）。任何不考虑集群内部拓扑结构（至少涉及 IP）的应用程序可能会出现问题。例如，将应用程序路由到特定的 Pod IP 只能在该 Pod 发生变化之前起作用。如果该 Pod 关闭，控制它的部署（例如）将启动一个新的 Pod 来替代它，但 IP 将完全不同。集群 DNS（以及由此衍生的服务）为在集群中的应用程序之间进行请求提供了更好的方式，除非您的应用程序具有动态调整到集群变化（如 Pod 位置）的能力。

## 只有一个管理员

在基础网络中，多个管理员和冲突的规则可能会导致问题，多个 Kubernetes 管理员还可能通过更改资源配置（例如 Pod 资源限制）而引发进一步的问题，导致意外行为。使用 Kubernetes 的**基于角色的访问控制**（**RBAC**）功能可以通过为 Kubernetes 用户提供他们所需的权限（例如只读权限）来解决这个问题。

## 运输成本为零

这种谬误有两种常见的解释方式。首先，传输的延迟成本为零 - 这显然是不真实的，因为数据在电线上传输的速度并不是无限的，而且更低级的网络问题会增加延迟。这与“延迟为零”谬误产生的影响本质上是相同的。

其次，这个声明可以被解释为创建和操作网络传输的成本为零 - 就像零美元和零美分一样。虽然这也是显然不真实的（只需看看您的云服务提供商的数据传输费用就可以证明），但这并不特别对应于 Kubernetes 上的应用程序故障排查，所以我们将专注于第一种解释。

## 网络是同质的

这个最后的谬误与 Kubernetes 的组件关系不大，而与在 Kubernetes 上运行的应用程序有更多关系。然而，事实是，今天的环境中操作的开发人员都清楚地知道，应用程序网络可能在不同的应用程序中有不同的实现 - 从 HTTP 1 和 2 到诸如 *gRPC* 的协议。

现在我们已经回顾了一些 Kubernetes 应用失败的主要原因，我们可以深入研究排查 Kubernetes 和在 Kubernetes 上运行的应用程序的实际过程。

# 排查 Kubernetes 集群

由于 Kubernetes 是一个分布式系统，旨在容忍应用程序运行的故障，大多数（但不是全部）问题往往集中在控制平面和 API 上。在大多数情况下，工作节点的故障只会导致 Pod 被重新调度到另一个节点 - 尽管复合因素可能会引入问题。

为了演示常见的 Kubernetes 集群问题场景，我们将使用案例研究方法。这应该为您提供调查真实世界集群问题所需的所有工具。我们的第一个案例研究集中在 API 服务器本身的故障上。

重要提示

在本教程中，我们将假设一个自管理的集群。托管的 Kubernetes 服务，如 EKS、AKS 和 GKE 通常会消除一些故障域（例如通过自动缩放和管理主节点）。一个好的规则是首先检查您的托管服务文档，因为任何问题可能是特定于实现的。

## 案例研究 - Kubernetes Pod 放置失败

让我们来设定场景。您的集群正在运行，但是您遇到了 Pod 调度的问题。Pods 一直停留在 `Pending` 状态，无限期地。让我们用以下命令确认一下：

```
kubectl get pods
```

命令的输出如下：

```
NAME                              READY     STATUS    RESTARTS   AGE
app-1-pod-2821252345-tj8ks        0/1       Pending   0          2d
app-1-pod-2821252345-9fj2k        0/1       Pending   0          2d
app-1-pod-2821252345-06hdj        0/1       Pending   0          2d
```

正如我们所看到的，我们的 Pod 都没有在运行。此外，我们正在运行应用程序的三个副本，但没有一个被调度。下一个很好的步骤是检查节点状态，看看是否有任何问题。运行以下命令以获取输出：

```
kubectl get nodes
```

我们得到以下输出：

```
  NAME           STATUS     ROLES    AGE    VERSION
  node-01        NotReady   <none>   5m     v1.15.6
```

这个输出给了我们一些很好的信息 - 我们只有一个工作节点，并且它无法用于调度。当 `get` 命令没有给我们足够的信息时，`describe` 通常是一个很好的下一步。

让我们运行 `kubectl describe node node-01` 并检查 `conditions` 键。我们已经删除了一列，以便将所有内容整齐地显示在页面上，但最重要的列都在那里：

![图 10.1 - 描述节点条件输出](img/B14790_10_001.jpg)

图 10.1 - 描述节点条件输出

我们在这里有一个有趣的分裂：`MemoryPressure` 和 `DiskPressure` 都很好，而 `OutOfDisk` 和 `Ready` 条件的状态是未知的，消息是 `kubelet stopped posting node status`。乍一看，这似乎是荒谬的 - `MemoryPressure` 和 `DiskPressure` 怎么可能正常，而 kubelet 却停止工作了呢？

重要的部分在 `LastTransitionTime` 列中。kubelet 最近的内存和磁盘特定通信发送了积极的状态。然后，在稍后的时间，kubelet 停止发布其节点状态，导致 `OutOfDisk` 和 `Ready` 条件的状态为 `Unknown`。

在这一点上，我们可以肯定我们的节点是问题所在 - kubelet 不再将节点状态发送到控制平面。然而，我们不知道为什么会发生这种情况。可能是网络错误，机器本身的问题，或者更具体的问题。我们需要进一步挖掘才能弄清楚。

在这里一个很好的下一步是接近我们的故障节点，因为我们可以合理地假设它遇到了某种问题。如果您可以访问 `node-01` VM 或机器，现在是 SSH 进入的好时机。一旦我们进入机器，让我们进一步进行故障排除。

首先，让我们检查节点是否可以通过网络访问控制平面。如果不能，这显然是 kubelet 无法发布状态的明显原因。假设我们的集群控制平面（例如，本地负载均衡器）位于`10.231.0.1`，为了检查我们的节点是否可以访问 Kubernetes API 服务器，我们可以像下面这样 ping 控制平面：

```
ping 10.231.0.1   
```

重要提示

为了找到控制平面的 IP 或 DNS，请检查您的集群配置。在 AWS Elastic Kubernetes Service 或 Azure AKS 等托管的 Kubernetes 服务中，这可能可以在控制台中查看。例如，如果您使用 kubeadm 自己引导了集群，那么这是您在安装过程中提供的值之一。

让我们来检查结果：

```
Reply from 10.231.0.1: bytes=1500 time=28ms TTL=54
Reply from 10.231.0.1: bytes=1500 time=26ms TTL=54
Reply from 10.231.0.1: bytes=1500 time=27ms TTL=54
```

这证实了 - 我们的节点确实可以与 Kubernetes 控制平面通信。因此，网络不是问题。接下来，让我们检查实际的 kubelet 服务。节点本身似乎是正常运行的，网络也正常，所以逻辑上，kubelet 是下一个要检查的东西。

Kubernetes 组件在 Linux 节点上作为系统服务运行。

重要提示

在 Windows 节点上，故障排除说明会略有不同 - 请参阅 Kubernetes 文档以获取更多信息（[`kubernetes.io/docs/setup/production-environment/windows/intro-windows-in-kubernetes/`](https://kubernetes.io/docs/setup/production-environment/windows/intro-windows-in-kubernetes/)）。

为了找出我们的`kubelet`服务的状态，我们可以运行以下命令：

```
systemctl status kubelet -l 
```

这给我们以下输出：

```
 • kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Fri 2020-05-22 05:44:25 UTC; 3s ago
     Docs: http://kubernetes.io/docs/
  Process: 32315 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=1/FAILURE)
 Main PID: 32315 (code=exited, status=1/FAILURE)
```

看起来我们的 kubelet 目前没有运行 - 它以失败退出。这解释了我们所看到的集群状态和 Pod 问题。

实际上修复问题，我们可以首先尝试使用以下命令重新启动`kubelet`：

```
systemctl start kubelet
```

现在，让我们使用我们的状态命令重新检查`kubelet`的状态：

```
 • kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Fri 2020-05-22 06:13:48 UTC; 10s ago
     Docs: http://kubernetes.io/docs/
  Process: 32007 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=1/FAILURE)
 Main PID: 32007 (code=exited, status=1/FAILURE)
```

看起来`kubelet`又失败了。我们需要获取一些关于失败模式的额外信息，以便找出发生了什么。

让我们使用`journalctl`命令查看是否有相关的日志：

```
sudo journalctl -u kubelet.service | grep "failed"
```

输出应该显示`kubelet`服务的日志，其中发生了故障：

```
May 22 04:19:16 nixos kubelet[1391]: F0522 04:19:16.83719    1287 server.go:262] failed to run Kubelet: Running with swap on is not supported, please disable swap! or set --fail-swap-on flag to false. /proc/swaps contained: [Filename                                Type                Size        Used        Priority /dev/sda1                               partition        6198732        0        -1]
```

看起来我们已经找到了原因-Kubernetes 默认不支持在 Linux 机器上运行时将`swap`设置为`on`。我们在这里的唯一选择要么是禁用`swap`，要么是使用设置为`false`的`--fail-swap-on`标志重新启动`kubelet`。

在我们的情况下，我们将使用以下命令更改`swap`设置：

```
sudo swapoff -a
```

现在，重新启动`kubelet`服务：

```
sudo systemctl restart kubelet
```

最后，让我们检查一下我们的修复是否奏效。使用以下命令检查节点：

```
kubectl get nodes 
```

这应该显示类似于以下内容的输出：

```
  NAME           STATUS     ROLES    AGE    VERSION
  node-01        Ready      <none>   54m    v1.15.6
```

我们的节点最终发布了`Ready`状态！

让我们使用以下命令检查我们的 Pod：

```
kubectl get pods
```

这应该显示如下输出：

```
NAME                              READY     STATUS    RESTARTS   AGE
app-1-pod-2821252345-tj8ks        1/1       Running   0          1m
app-1-pod-2821252345-9fj2k        1/1       Running   0          1m
app-1-pod-2821252345-06hdj        1/1       Running   0          1m
```

成功！我们的集群健康，我们的 Pod 正在运行。

接下来，让我们看看在解决了任何集群问题后如何排除 Kubernetes 上的应用程序故障。

# 在 Kubernetes 上排除应用程序故障

一个完全运行良好的 Kubernetes 集群可能仍然存在需要调试的应用程序问题。这可能是由于应用程序本身的错误，也可能是由于组成应用程序的 Kubernetes 资源的错误配置。与排除集群故障一样，我们将通过使用案例研究来深入了解这些概念。

## 案例研究 1-服务无响应

我们将把这一部分分解为 Kubernetes 堆栈各个级别的故障排除，从更高级别的组件开始，然后深入到 Pod 和容器调试。

假设我们已经配置我们的应用程序`app-1`通过`NodePort`服务响应端口`32688`的请求。该应用程序监听端口`80`。

我们可以尝试通过在我们的节点之一上使用`curl`请求来访问我们的应用程序。命令将如下所示：

```
curl http://10.213.2.1:32688
```

如果 curl 命令失败，输出将如下所示：

```
curl: (7) Failed to connect to 10.231.2.1 port 32688: Connection refused
```

此时，我们的`NodePort`服务没有将请求路由到任何 Pod。按照我们典型的调试路径，让我们首先查看使用以下命令在集群中运行的哪些资源：

```
kubectl get services
```

添加`-o`宽标志以查看更多信息。接下来，运行以下命令：

```
kubectl get services -o wide 
```

这给了我们以下输出：

```
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE SELECTOR 
app-1-svc NodePort 10.101.212.57 <none> 80:32688/TCP 3m01s app=app-1
```

很明显，我们的服务存在一个正确的节点端口-但是我们的请求没有被路由到 Pod，这是从失败的 curl 命令中显而易见的。

要查看我们的服务设置了哪些路由，让我们使用`get endpoints`命令。这将列出服务配置的 Pod IP（如果有的话）。

```
kubectl get endpoints app-1-svc
```

让我们检查命令的结果输出：

```
NAME        ENDPOINTS
app-1-svc   <none>
```

嗯，这里肯定有问题。

我们的服务没有指向任何 Pod。这很可能意味着没有任何与我们的服务选择器匹配的 Pod 可用。这可能是因为根本没有可用的 Pod - 或者因为这些 Pod 不正确地匹配了服务选择器。

要检查我们的服务选择器，让我们沿着调试路径迈出下一步，并使用以下命令：

```
kubectl describe service app-1-svc  
```

这给我们一个类似以下的输出：

```
Name:                   app-1-svc
Namespace:              default
Labels:                 app=app-11
Annotations:            <none>
Selector:               app=app-11
Type:                   NodePort
IP:                     10.57.0.15
Port:                   <unset> 80/TCP
TargetPort:             80/TCP
NodePort:               <unset> 32688/TCP
Endpoints:              <none>
Session Affinity:       None
Events:                 <none>
```

正如您所看到的，我们的服务配置为与我们的应用程序上的正确端口进行通信。但是，选择器正在寻找与标签`app = app-11`匹配的 Pod。由于我们知道我们的应用程序名称为`app-1`，这可能是我们问题的原因。

让我们编辑我们的服务，以寻找正确的 Pod 标签`app-1`，再次运行另一个`describe`命令以确保：

```
kubectl describe service app-1-svc
```

这会产生以下输出：

```
Name:                   app-1-svc
Namespace:              default
Labels:                 app=app-1
Annotations:            <none>
Selector:               app=app-1
Type:                   NodePort
IP:                     10.57.0.15
Port:                   <unset> 80/TCP
TargetPort:             80/TCP
NodePort:               <unset> 32688/TCP
Endpoints:              <none>
Session Affinity:       None
Events:                 <none>
```

现在，您可以在输出中看到我们的服务正在寻找正确的 Pod 选择器，但我们仍然没有任何端点。让我们使用以下命令来查看我们的 Pod 的情况：

```
kubectl get pods
```

这显示了以下输出：

```
NAME                              READY     STATUS    RESTARTS   AGE
app-1-pod-2821252345-tj8ks        0/1       Pending   0          -
app-1-pod-2821252345-9fj2k        0/1       Pending   0          -
app-1-pod-2821252345-06hdj        0/1       Pending   0          -
```

我们的 Pod 仍在等待调度。这解释了为什么即使有正确的选择器，我们的服务也无法正常运行。为了更细致地了解为什么我们的 Pod 没有被调度，让我们使用`describe`命令：

```
kubectl describe pod app-1-pod-2821252345-tj8ks
```

以下是输出。让我们专注于“事件”部分：

![图 10.2 - 描述 Pod 事件输出](img/B14790_10_002.jpg)

图 10.2 - 描述 Pod 事件输出

从“事件”部分来看，似乎我们的 Pod 由于容器镜像拉取失败而无法被调度。这可能有很多原因 - 例如，我们的集群可能没有必要的身份验证机制来从私有仓库拉取，但这会出现不同的错误消息。

从上下文和“事件”输出来看，我们可能可以假设问题在于我们的 Pod 定义正在寻找一个名为`myappimage:lates`的容器，而不是`myappimage:latest`。

让我们使用正确的镜像名称更新我们的部署规范并进行更新。

使用以下命令来确认：

```
kubectl get pods
```

输出看起来像这样：

```
NAME                              READY     STATUS    RESTARTS   AGE
app-1-pod-2821252345-152sf        1/1       Running   0          1m
app-1-pod-2821252345-9gg9s        1/1       Running   0          1m
app-1-pod-2821252345-pfo92        1/1       Running   0          1m
```

我们的 Pod 现在正在运行 - 让我们检查一下我们的服务是否已注册了正确的端点。使用以下命令来执行此操作：

```
kubectl describe services app-1-svc
```

输出应该是这样的：

```
Name:                   app-1-svc
Namespace:              default
Labels:                 app=app-1
Annotations:            <none>
Selector:               app=app-1
Type:                   NodePort
IP:                     10.57.0.15
Port:                   <unset> 80/TCP
TargetPort:             80/TCP
NodePort:               <unset> 32688/TCP
Endpoints:              10.214.1.3:80,10.214.2.3:80,10.214.4.2:80
Session Affinity:       None
Events:                 <none>
```

成功！我们的服务正确地指向了我们的应用程序 Pod。

在下一个案例研究中，我们将通过排除具有不正确启动参数的 Pod 来深入挖掘一些问题。

## 案例研究 2 - 错误的 Pod 启动命令

让我们假设我们的 Service 已经正确配置，我们的 Pods 正在运行并通过健康检查。然而，我们的 Pod 没有按照我们的预期响应请求。我们确信这不是 Kubernetes 的问题，而更多是应用程序或配置的问题。

我们的应用程序容器工作方式如下：它接受一个带有`color`标志的启动命令，并根据容器的`image`标签的`version number`变量组合起来，并将其回显给请求者。我们期望我们的应用程序返回`green 3`。

幸运的是，Kubernetes 为我们提供了一些很好的工具来调试应用程序，我们可以用这些工具来深入研究我们特定的容器。

首先，让我们使用以下命令`curl`应用程序，看看我们得到什么响应：

```
curl http://10.231.2.1:32688  
red 2
```

我们期望得到`green 3`，但得到了`red 2`，所以看起来输入和版本号变量出了问题。让我们先从前者开始。

像往常一样，我们首先用以下命令检查我们的 Pods：

```
kubectl get pods
```

输出应该如下所示：

```
NAME                              READY     STATUS    RESTARTS   AGE
app-1-pod-2821252345-152sf        1/1       Running   0          5m
app-1-pod-2821252345-9gg9s        1/1       Running   0          5m
app-1-pod-2821252345-pfo92        1/1       Running   0          5m
```

这个输出看起来很好。我们的应用程序似乎作为部署的一部分运行（因此也是 ReplicaSet） - 我们可以通过运行以下命令来确保：

```
kubectl get deployments
```

输出应该如下所示：

```
NAME          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
app-1-pod     3         3         3            3           5m
```

让我们更仔细地查看我们的部署，看看我们的 Pods 是如何配置的，使用以下命令：

```
kubectl describe deployment app-1-pod -o yaml
```

输出应该如下所示：

Broken-deployment-output.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-1-pod
spec:
  selector:
    matchLabels:
      app: app-1
  replicas: 3
  template:
    metadata:
      labels:
        app: app-1
    spec:
      containers:
      - name: app-1
        image: mycustomrepository/app-1:2
        command: [ "start", "-color", "red" ]
        ports:
        - containerPort: 80
```

让我们看看是否可以解决我们的问题，这实际上非常简单。我们使用了错误版本的应用程序，而且我们的启动命令也是错误的。在这种情况下，让我们假设我们没有一个包含我们部署规范的文件 - 所以让我们直接在原地编辑它。

让我们使用`kubectl edit deployment app-1-pod`，并编辑 Pod 规范如下：

fixed-deployment-output.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-1-pod
spec:
  selector:
    matchLabels:
      app: app-1
  replicas: 3
  template:
    metadata:
      labels:
        app: app-1
    spec:
      containers:
      - name: app-1
        image: mycustomrepository/app-1:3
        command: [ "start", "-color", "green" ]
        ports:
        - containerPort: 80
```

一旦部署保存，你应该开始看到你的新 Pods 启动。让我们通过以下命令再次检查：

```
 kubectl get pods
```

输出应该如下所示：

```
NAME                              READY     STATUS    RESTARTS   AGE
app-1-pod-2821252345-f928a        1/1       Running   0          1m
app-1-pod-2821252345-jjsa8        1/1       Running   0          1m
app-1-pod-2821252345-92jhd        1/1       Running   0          1m
```

最后 - 让我们发出一个`curl`请求来检查一切是否正常运行：

```
curl http://10.231.2.1:32688  
```

命令的输出如下：

```
green 3
```

成功！

## 案例研究 3 - Pod 应用程序日志故障

在上一章[*第九章*]（B14790_9_Final_PG_ePub.xhtml#_idTextAnchor212），*Kubernetes 上的可观测性*中，我们为我们的应用程序实现了可观测性，让我们看一个案例，这些工具确实非常有用。我们将使用手动的`kubectl`命令来进行这个案例研究 - 但要知道，通过聚合日志（例如，在我们的 EFK 堆栈实现中），我们可以使调试这个应用程序的过程变得更容易。

在这个案例研究中，我们再次部署了 Pod - 为了检查它，让我们运行以下命令：

```
kubectl get pods
```

命令的输出如下：

```
NAME              READY     STATUS    RESTARTS   AGE
app-2-ss-0        1/1       Running   0          10m
app-2-ss-1       1/1       Running   0          10m
app-2-ss-2       1/1       Running   0          10m
```

看起来，在这种情况下，我们使用的是 StatefulSet 而不是 Deployment - 这里的一个关键特征是从 0 开始递增的 Pod ID。

我们可以通过使用以下命令来确认这一点：

```
kubectl get statefulset
```

命令的输出如下：

```
NAME          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
app-2-ss      3         3         3            3           10m
```

让我们使用`kubectl get statefulset -o yaml app-2-ss`来更仔细地查看我们的 StatefulSet。通过使用`get`命令以及`-o yaml`，我们可以以与典型的 Kubernetes 资源 YAML 相同的格式获得我们的`describe`输出。

上述命令的输出如下。我们已经删除了 Pod 规范部分以使其更短：

statefulset-output.yaml

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: app-2-ss
spec:
  selector:
    matchLabels:
      app: app-2
  replicas: 3
  template:
    metadata:
      labels:
        app: app-2
```

我们知道我们的应用程序正在使用一个服务。让我们看看是哪一个！

运行 `kubectl get services -o wide`。输出应该类似于以下内容：

```
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE SELECTOR 
app-2-svc NodePort 10.100.213.13 <none> 80:32714/TCP 3m01s app=app-2
```

很明显我们的服务叫做`app-2-svc`。让我们使用以下命令查看我们的确切服务定义：

```
kubectl describe services app-2-svc 
```

命令的输出如下：

```
Name:                   app-2-svc
Namespace:              default
Labels:                 app=app-2
Annotations:            <none>
Selector:               app=app-2
Type:                   NodePort
IP:                     10.57.0.12
Port:                   <unset> 80/TCP
TargetPort:             80/TCP
NodePort:               <unset> 32714/TCP
Endpoints:              10.214.1.1:80,10.214.2.3:80,10.214.4.4:80
Session Affinity:       None
Events:                 <none>
```

要确切地查看我们的应用程序对于给定输入返回的内容，我们可以在我们的`NodePort`服务上使用`curl`：

```
> curl http://10.231.2.1:32714?equation=1plus1
3
```

根据我们对应用程序的现有知识，我们会假设这个调用应该返回`2`而不是`3`。我们团队的应用程序开发人员已经要求我们调查任何日志输出，以帮助他们找出问题所在。

我们知道从之前的章节中，你可以使用`kubectl logs <pod name>`来调查日志输出。在我们的情况下，我们有三个应用程序的副本，所以我们可能无法在一次迭代中找到我们的日志。让我们随机选择一个 Pod，看看它是否是为我们提供服务的那个：

```
> kubectl logs app-2-ss-1
>
```

看起来这不是为我们提供服务的 Pod，因为我们的应用程序开发人员告诉我们，当向服务器发出`GET`请求时，应用程序肯定会记录到`stdout`。

我们可以使用联合命令从所有三个 Pod 中获取日志，而不是逐个检查另外两个 Pod。命令将如下：

```
> kubectl logs statefulset/app-2-ss
```

输出如下：

```
> Input = 1plus1
> Operator = plus
> First Number = 1
> Second Number = 2
```

这样就解决了问题 - 而且更重要的是，我们可以看到一些关于我们问题的很好的见解。

除了日志行读取`Second Number`之外，一切都如我们所期望的那样。我们的请求明显使用`1plus1`作为查询字符串，这将使第一个数字和第二个数字（由运算符值分隔）都等于一。

这将需要一些额外的挖掘。我们可以通过发送额外的请求并检查输出来对这个问题进行分类，以猜测发生了什么，但在这种情况下，最好只是获取对 Pod 的 bash 访问并弄清楚发生了什么。

首先，让我们检查一下我们的 Pod 规范，这是从前面的 StatefulSet YAML 中删除的。要查看完整的 StatefulSet 规范，请检查 GitHub 存储库：

Statefulset-output.yaml

```
spec:
  containers:
  - name: app-2
    image: mycustomrepository/app-2:latest
    volumeMounts:
    - name: scratch
      mountPath: /scratch
  - name: sidecar
    image: mycustomrepository/tracing-sidecar
  volumes:
  - name: scratch-volume
    emptyDir: {}
```

看起来我们的 Pod 正在挂载一个空卷作为临时磁盘。每个 Pod 中还有两个容器 - 一个用于应用程序跟踪的 sidecar，以及我们的应用程序本身。我们需要这些信息来使用`kubectl exec`命令`ssh`到其中一个 Pod（对于这个练习来说，无论选择哪一个都可以）。

我们可以使用以下命令来完成：

```
kubectl exec -it app-2-ss-1 app2 -- sh.  
```

这个命令应该给你一个 bash 终端作为输出：

```
> kubectl exec -it app-2-ss-1 app2 -- sh
# 
```

现在，使用我们刚创建的终端，我们应该能够调查我们的应用程序代码。在本教程中，我们使用了一个非常简化的 Node.js 应用程序。

让我们检查一下我们的 Pod 文件系统，看看我们使用以下命令在处理什么：

```
# ls
# app.js calculate.js scratch
```

看起来我们有两个 JavaScript 文件，以及我们之前提到的`scratch`文件夹。可以假设`app.js`包含引导和提供应用程序的逻辑，而`calculate.js`包含我们的控制器代码来进行计算。

我们可以通过打印`calculate.js`文件的内容来确认：

Broken-calculate.js

```
# cat calculate.js
export const calculate(first, second, operator)
{
  second++;
  if(operator === "plus")
  {
   return first + second;
  }
}
```

即使对 JavaScript 几乎一无所知，这里的问题也是非常明显的。代码在执行计算之前递增了`second`变量。

由于我们在 Pod 内部，并且正在使用非编译语言，我们实际上可以内联编辑这个文件！让我们使用`vi`（或任何文本编辑器）来纠正这个文件：

```
# vi calculate.js
```

并编辑文件如下所示：

fixed-calculate.js

```
export const calculate(first, second, operator)
{
  if(operator === "plus")
  {
   return first + second;
  }
}
```

现在，我们的代码应该正常运行。重要的是要说明，这个修复只是临时的。一旦我们的 Pod 关闭或被另一个 Pod 替换，它将恢复到最初包含在容器镜像中的代码。然而，这种模式确实允许我们尝试快速修复。

在使用`exit` bash 命令退出`exec`会话后，让我们再次尝试我们的 URL：

```
> curl http://10.231.2.1:32714?equation=1plus1
2
```

正如你所看到的，我们的热修复容器显示了正确的结果！现在，我们可以使用我们的修复以更加永久的方式更新我们的代码和 Docker 镜像。使用`exec`是一个很好的方法来排除故障和调试运行中的容器。

# 总结

在本章中，我们学习了如何在 Kubernetes 上调试应用程序。首先，我们介绍了分布式应用程序的一些常见故障模式。然后，我们学习了如何对 Kubernetes 组件的问题进行分类。最后，我们回顾了几种 Kubernetes 配置和应用程序调试的场景。在本章中学到的 Kubernetes 调试和故障排除技术将帮助你在处理任何 Kubernetes 集群和应用程序的问题时。

在下一章，*第十一章*，*Kubernetes 上的模板代码生成和 CI/CD*，我们将探讨一些用于模板化 Kubernetes 资源清单和与 Kubernetes 一起进行持续集成/持续部署的生态系统扩展。

# 问题

1.  分布式系统谬误“*拓扑结构不会改变*”如何适用于 Kubernetes 上的应用程序？

1.  Kubernetes 控制平面组件（和 kubelet）在操作系统级别是如何实现的？

1.  当 Pod 被卡在`Pending`状态时，你会如何调试问题？你的第一步会是什么？第二步呢？

# 进一步阅读

+   用于流量整形的 CNI 插件：[`kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#support-traffic-shaping`](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#support-traffic-shaping)

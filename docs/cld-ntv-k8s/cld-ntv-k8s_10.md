# 第八章：Pod 放置控制

本章描述了在 Kubernetes 中控制 Pod 放置的各种方式，以及解释为什么首先实施这些控制可能是一个好主意。Pod 放置意味着控制 Pod 在 Kubernetes 中被调度到哪个节点。我们从简单的控制开始，比如节点选择器，然后转向更复杂的工具，比如污点和容忍度，最后介绍两个 beta 功能，节点亲和性和 Pod 间亲和性/反亲和性。

在过去的章节中，我们已经学习了如何在 Kubernetes 上最好地运行应用程序 Pod - 从使用部署协调和扩展它们，使用 ConfigMaps 和 Secrets 注入配置，到使用持久卷添加存储。

然而，尽管如此，我们始终依赖 Kubernetes 调度程序将 Pod 放置在最佳节点上，而没有给调度程序提供有关所讨论的 Pod 的太多信息。到目前为止，我们已经在 Pod 中添加了资源限制和请求（Pod 规范中的`resource.requests`和`resource.limits`）。资源请求指定 Pod 在调度时需要的节点上的最低空闲资源水平，而资源限制指定 Pod 允许使用的最大资源量。然而，我们并没有对 Pod 必须运行在哪些节点或节点集上提出任何具体要求。

对于许多应用程序和集群来说，这是可以的。然而，正如我们将在第一节中看到的，有许多情况下使用更精细的 Pod 放置控制是一种有用的策略。

在本章中，我们将涵盖以下主题：

+   识别 Pod 放置的用例

+   使用节点选择器

+   实施污点和容忍度

+   使用节点亲和性控制 Pod

+   使用 Pod 亲和性和反亲和性

# 技术要求

为了运行本章中详细介绍的命令，您需要一台支持`kubectl`命令行工具的计算机，以及一个可用的 Kubernetes 集群。请参阅*第一章*，*与 Kubernetes 通信*，了解快速启动和运行 Kubernetes 的几种方法，以及如何安装`kubectl`工具的说明。

本章中使用的代码可以在书的 GitHub 存储库中找到[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter8`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter8)。

# 识别 Pod 放置的用例

Pod 放置控制是 Kubernetes 提供给我们的工具，用于决定将 Pod 调度到哪个节点，或者由于缺少我们想要的节点而完全阻止 Pod 的调度。这可以用于几种不同的模式，但我们将回顾一些主要的模式。首先，Kubernetes 本身默认完全实现了 Pod 放置控制-让我们看看如何实现。

## Kubernetes 节点健康放置控制

Kubernetes 使用一些默认的放置控制来指定某种方式不健康的节点。这些通常是使用污点和容忍来定义的，我们将在本章后面详细讨论。

Kubernetes 使用的一些默认污点（我们将在下一节中讨论）如下：

+   `memory-pressure`

+   `disk-pressure`

+   `unreachable`

+   `not-ready`

+   `out-of-disk`

+   `network-unavailable`

+   `unschedulable`

+   `uninitialized`（仅适用于由云提供商创建的节点）

这些条件可以将节点标记为无法接收新的 Pod，尽管调度器在处理这些污点的方式上有一定的灵活性，我们稍后会看到。这些系统创建的放置控制的目的是防止不健康的节点接收可能无法正常运行的工作负载。

除了用于节点健康的系统创建的放置控制之外，还有一些用例，您作为用户可能希望实现精细调度，我们将在下一节中看到。

## 需要不同节点类型的应用程序

在异构的 Kubernetes 集群中，每个节点并不相同。您可能有一些更强大的虚拟机（或裸金属）和一些较弱的，或者有不同的专门的节点集。

例如，在运行数据科学流水线的集群中，您可能有具有 GPU 加速能力的节点来运行深度学习算法，常规计算节点来提供应用程序，具有大量内存的节点来基于已完成的模型进行推理，等等。

使用 Pod 放置控制，您可以确保平台的各个部分在最适合当前任务的硬件上运行。

## 需要特定数据合规性的应用程序

与前面的例子类似，应用程序要求可能决定了对不同类型的计算需求，某些数据合规性需求可能需要特定类型的节点。

例如，像 AWS 和 Azure 这样的云提供商通常允许您购买具有专用租户的 VM - 这意味着没有其他应用程序在底层硬件和虚拟化程序上运行。这与其他典型的云提供商 VM 不同，其他客户可能共享单个物理机。

对于某些数据法规，需要这种专用租户级别来保持合规性。为了满足这种需求，您可以使用 Pod 放置控件来确保相关应用仅在具有专用租户的节点上运行，同时通过在更典型的 VM 上运行控制平面来降低成本。

## 多租户集群

如果您正在运行一个具有多个租户的集群（例如通过命名空间分隔），您可以使用 Pod 放置控件来为租户保留某些节点或节点组，以便将它们与集群中的其他租户物理或以其他方式分开。这类似于 AWS 或 Azure 中的专用硬件的概念。

## 多个故障域

尽管 Kubernetes 已经通过允许您在多个节点上运行工作负载来提供高可用性，但也可以扩展这种模式。我们可以创建自己的 Pod 调度策略，考虑跨多个节点的故障域。处理这个问题的一个很好的方法是通过 Pod 或节点的亲和性或反亲和性特性，我们将在本章后面讨论。

现在，让我们构想一个情况，我们的集群在裸机上，每个物理机架有 20 个节点。如果每个机架都有自己的专用电源连接和备份，它可以被视为一个故障域。当电源连接失败时，机架上的所有机器都会失败。因此，我们可能希望鼓励 Kubernetes 在不同的机架/故障域上运行两个实例或 Pod。以下图显示了应用程序如何跨故障域运行：

![图 8.1 - 故障域](img/B14790_08_001.jpg)

图 8.1 - 故障域

正如您在图中所看到的，由于应用程序 Pod 分布在多个故障域中，而不仅仅是在同一故障域中的多个节点，即使*故障域 1*发生故障，我们也可以保持正常运行。*App A - Pod 1*和*App B - Pod 1*位于同一个（红色）故障域。但是，如果该故障域（*Rack 1*）发生故障，我们仍将在*Rack 2*上有每个应用的副本。

我们在这里使用“鼓励”这个词，因为在 Kubernetes 调度程序中，可以将一些功能配置为硬性要求或尽力而为。

这些示例应该让您对高级放置控件的一些潜在用例有一个扎实的理解。

现在让我们讨论实际的实现，逐个使用每个放置工具集。我们将从最简单的节点选择器开始。

# 使用节点选择器和节点名称

节点选择器是 Kubernetes 中一种非常简单的放置控制类型。每个 Kubernetes 节点都可以在元数据块中带有一个或多个标签，并且 Pod 可以指定一个节点选择器。

要为现有节点打标签，您可以使用`kubectl label`命令：

```
> kubectl label nodes node1 cpu_speed=fast
```

在这个例子中，我们使用标签`cpu_speed`和值`fast`来标记我们的`node1`节点。

现在，让我们假设我们有一个应用程序，它确实需要快速的 CPU 周期才能有效地执行。我们可以为我们的工作负载添加`nodeSelector`，以确保它只被调度到具有我们快速 CPU 速度标签的节点上，如下面的代码片段所示：

pod-with-node-selector.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: speedy-app
spec:
  containers:
  - name: speedy-app
    image: speedy-app:latest
    imagePullPolicy: IfNotPresent
  nodeSelector:
    cpu_speed: fast
```

当部署时，作为部署的一部分或单独部署，我们的`speedy-app` Pod 将只被调度到具有`cpu_speed`标签的节点上。

请记住，与我们即将审查的一些其他更高级的 Pod 放置选项不同，节点选择器中没有任何余地。如果没有具有所需标签的节点，应用程序将根本不会被调度。

对于更简单（但更脆弱）的选择器，您可以使用`nodeName`，它指定 Pod 应该被调度到的确切节点。您可以像这样使用它：

pod-with-node-name.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: speedy-app
spec:
  containers:
  - name: speedy-app
    image: speedy-app:latest
    imagePullPolicy: IfNotPresent
  nodeName: node1
```

正如您所看到的，这个选择器只允许 Pod 被调度到`node1`，所以如果它当前由于任何原因不接受 Pods，Pod 将不会被调度。

对于稍微更加微妙的放置控制，让我们转向污点和容忍。

# 实施污点和容忍

在 Kubernetes 中，污点和容忍的工作方式类似于反向节点选择器。与节点吸引 Pods 因具有适当的标签而被选择器消耗不同，我们对节点进行污点处理，这会排斥所有 Pod 被调度到该节点，然后标记我们的 Pods 具有容忍，这允许它们被调度到被污点处理的节点上。

正如本章开头提到的，Kubernetes 使用系统创建的污点来标记节点为不健康，并阻止新的工作负载被调度到它们上面。例如，`out-of-disk`污点将阻止任何新的 Pod 被调度到具有该污点的节点上。

让我们使用污点和容忍度来应用与节点选择器相同的示例用例。由于这基本上是我们先前设置的反向，让我们首先使用`kubectl taint`命令给我们的节点添加一个污点：

```
> kubectl taint nodes node2 cpu_speed=slow:NoSchedule
```

让我们分解这个命令。我们给`node2`添加了一个名为`cpu_speed`的污点和一个值`slow`。我们还用一个效果标记了这个污点 - 在这种情况下是`NoSchedule`。

一旦我们完成了我们的示例（如果您正在跟随命令进行操作，请不要立即执行此操作），我们可以使用减号运算符删除`taint`：

```
> kubectl taint nodes node2 cpu_speed=slow:NoSchedule-
```

`taint`效果让我们在调度器处理污点时增加了一些细粒度。有三种可能的效果值：

+   `NoSchedule`

+   `NoExecute`

+   `PreferNoSchedule`

前两个效果，`NoSchedule`和`NoExecute`，提供了硬效果 - 也就是说，像节点选择器一样，只有两种可能性，要么 Pod 上存在容忍度（我们马上就会看到），要么 Pod 没有被调度。`NoExecute`通过驱逐所有具有容忍度的节点上的 Pod 来增加这个基本功能，而`NoSchedule`让现有的 Pod 保持原状，同时阻止任何没有容忍度的新 Pod 加入。

`PreferNoSchedule`，另一方面，为 Kubernetes 调度器提供了一些余地。它告诉调度器尝试为没有不可容忍污点的 Pod 找到一个节点，但如果不存在，则继续安排它。它实现了软效果。

在我们的情况下，我们选择了`NoSchedule`，因此不会将新的 Pod 分配给该节点 - 除非当然我们提供了一个容忍度。现在让我们这样做。假设我们有第二个应用程序，它不关心 CPU 时钟速度。它很乐意生活在我们较慢的节点上。这是 Pod 清单：

pod-without-speed-requirement.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: slow-app
spec:
  containers:
  - name: slow-app
    image: slow-app:latest
```

现在，我们的`slow-app` Pod 将不会在任何具有污点的节点上运行。我们需要为这个 Pod 提供一个容忍度，以便它可以被调度到具有污点的节点上 - 我们可以这样做：

pod-with-toleration.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: slow-app
spec:
  containers:
  - name: slow-app
    image: slow-app:latest
tolerations:
- key: "cpu_speed"
  operator: "Equal"
  value: "slow"
  effect: "NoSchedule"
```

让我们分解我们的`tolerations`条目，这是一个值数组。每个值都有一个`key`-与我们的污点名称相同。然后是一个`operator`值。这个`operator`可以是`Equal`或`Exists`。对于`Equal`，您可以使用`value`键，就像前面的代码中那样，配置污点必须等于的值，以便 Pod 容忍。对于`Exists`，污点名称必须在节点上，但不管值是什么都没有关系，就像这个 Pod 规范中一样：

pod-with-toleration2.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: slow-app
spec:
  containers:
  - name: slow-app
    image: slow-app:latest
tolerations:
- key: "cpu_speed"
  operator: "Exists"
  effect: "NoSchedule"
```

如您所见，我们已经使用了`Exists` `operator`值来允许我们的 Pod 容忍任何`cpu_speed`污点。

最后，我们有我们的`effect`，它的工作方式与污点本身的`effect`相同。它可以包含与污点效果完全相同的值- `NoSchedule`，`NoExecute`和`PreferNoSchedule`。

具有`NoExecute`容忍的 Pod 将无限期容忍与其关联的污点。但是，您可以添加一个名为`tolerationSeconds`的字段，以便在经过规定的时间后，Pod 离开受污染的节点。这允许您指定在一段时间后生效的容忍。让我们看一个例子：

pod-with-toleration3.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: slow-app
spec:
  containers:
  - name: slow-app
    image: slow-app:latest
tolerations:
- key: "cpu_speed"
  operator: "Equal"
  Value: "slow"
  effect: "NoExecute"
  tolerationSeconds: 60
```

在这种情况下，当污点和容忍执行时，已经在具有`taint`的节点上运行的 Pod 将在重新调度到不同节点之前在节点上保留`60`秒。

## 多个污点和容忍

当 Pod 和节点上有多个污点或容忍时，调度程序将检查它们所有。这里没有`OR`逻辑运算符-如果节点上的任何污点在 Pod 上没有匹配的容忍，它将不会被调度到节点上（除了`PreferNoSchedule`之外，在这种情况下，与以前一样，调度程序将尽量不在节点上调度）。即使在节点上有六个污点中，Pod 容忍了其中五个，它仍然不会被调度到`NoSchedule`污点，并且仍然会因为`NoExecute`污点而被驱逐。

对于一个可以更微妙地控制放置方式的工具，让我们看一下节点亲和力。

# 使用节点亲和力控制 Pod

正如你可能已经注意到的，污点和容忍性 - 虽然比节点选择器灵活得多 - 仍然留下了一些用例未解决，并且通常只允许*过滤*模式，你可以使用`Exists`或`Equals`来匹配特定的污点。可能有更高级的用例，你想要更灵活的方法来选择节点 - Kubernetes 的*亲和性*就是解决这个问题的功能。

有两种亲和性：

+   **节点亲和性**

+   **跨 Pod 的亲和性**

节点亲和性是节点选择器的类似概念，只是它允许更强大的选择特征集。让我们看一些示例 YAML，然后分解各个部分：

pod-with-node-affinity.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: affinity-test
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: cpu_speed
            operator: In
            values:
            - fast
            - medium_fast
  containers:
  - name: speedy-app
    image: speedy-app:latest
```

正如你所看到的，我们的`Pod` `spec`有一个`affinity`键，并且我们指定了一个`nodeAffinity`设置。有两种可能的节点亲和性类型：

+   `requiredDuringSchedulingIgnoredDuringExecution`

+   `preferredDuringSchedulingIgnoredDuringExecution`

这两种类型的功能直接映射到`NoSchedule`和`PreferNoSchedule`的工作方式。

## 使用 requiredDuringSchedulingIgnoredDuringExecution 节点亲和性

对于`requiredDuringSchedulingIgnoredDuringExecution`，Kubernetes 永远不会调度一个没有与节点匹配的术语的 Pod。

对于`preferredDuringSchedulingIgnoredDuringExecution`，它将尝试满足软性要求，但如果不能，它仍然会调度 Pod。

节点亲和性相对于节点选择器和污点和容忍性的真正能力在于你可以在选择器方面实现的实际表达式和逻辑。

`requiredDuringSchedulingIgnoredDuringExecution`和`preferredDuringSchedulingIgnoredDuringExecution`亲和性的功能是非常不同的，因此我们将分别进行审查。

对于我们的`required`亲和性，我们有能力指定`nodeSelectorTerms` - 可以是一个或多个包含`matchExpressions`的块。对于每个`matchExpressions`块，可以有多个表达式。

在我们在上一节中看到的代码块中，我们有一个单一的节点选择器术语，一个`matchExpressions`块 - 它本身只有一个表达式。这个表达式寻找`key`，就像节点选择器一样，代表一个节点标签。接下来，它有一个`operator`，它给了我们一些灵活性，让我们决定如何识别匹配。以下是操作符的可能值：

+   `In`

+   `NotIn`

+   `Exists`

+   `DoesNotExist`

+   `Gt`（注意：大于）

+   `Lt`（注意：小于）

在我们的情况下，我们使用了`In`运算符，它将检查值是否是我们指定的几个值之一。最后，在我们的`values`部分，我们可以列出一个或多个值，根据运算符，必须匹配才能使表达式为真。

正如你所看到的，这为我们在指定选择器时提供了更大的粒度。让我们看一个使用不同运算符的`cpu_speed`的例子：

pod-with-node-affinity2.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: affinity-test
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: cpu_speed
            operator: Gt
            values:
            - "5"
  containers:
  - name: speedy-app
    image: speedy-app:latest
```

正如你所看到的，我们正在使用非常精细的`matchExpressions`选择器。现在，使用更高级的运算符匹配的能力使我们能够确保我们的`speedy-app`只安排在具有足够高时钟速度（在本例中为 5 GHz）的节点上。我们可以更加精细地规定，而不是将我们的节点分类为“慢”和“快”这样的广泛组别。

接下来，让我们看看另一种节点亲和性类型 - `preferredDuringSchedulingIgnoredDuringExecution`。

## 使用 preferredDuringSchedulingIgnoredDuringExecution 节点亲和性

这种情况的语法略有不同，并且使我们能够更精细地影响这个“软”要求。让我们看一个实现这一点的 Pod spec YAML：

pod-with-node-affinity3.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: slow-app-affinity
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: cpu_speed
            operator: Lt
            values:
            - "3"
  containers:
  - name: slow-app
    image: slow-app:latest
```

这看起来与我们的`required`语法有些不同。

对于`preferredDuringSchedulingIgnoredDuringExecution`，我们有能力为每个条目分配一个“权重”，并附带一个偏好，这可以再次是一个包含多个内部表达式的`matchExpressions`块，这些表达式使用相同的`key-operator-values`语法。

这里的关键区别是“权重”值。由于`preferredDuringSchedulingIgnoredDuringExecution`是一个**软**要求，我们可以列出几个不同的偏好，并附带权重，让调度器尽力满足它们。其工作原理是，调度器将遍历所有偏好，并根据每个偏好的权重和是否满足来计算节点的得分。假设所有硬性要求都得到满足，调度器将选择得分最高的节点。在前面的情况下，我们有一个权重为 1 的单个偏好，但权重可以从 1 到 100 不等 - 所以让我们看一个更复杂的设置，用于我们的`speedy-app`用例：

pod-with-node-affinity4.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: speedy-app-prefers-affinity
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 90
        preference:
          matchExpressions:
          - key: cpu_speed
            operator: Gt
            values:
            - "3"
      - weight: 10
        preference:
          matchExpressions:
          - key: memory_speed
            operator: Gt
            values:
            - "4"
  containers:
  - name: speedy-app
    image: speedy-app:latest
```

在确保我们的`speedy-app`在最佳节点上运行的过程中，我们决定只实现`soft`要求。如果没有快速节点存在，我们仍希望我们的应用程序被调度和运行。为此，我们指定了两个偏好 - 一个`cpu_speed`超过 3（3 GHz）和一个内存速度超过 4（4 GHz）的节点。

由于我们的应用程序更多地受限于 CPU 而不是内存，我们决定适当地权衡我们的偏好。在这种情况下，`cpu_speed`具有`weight`为`90`，而`memory_speed`具有`weight`为`10`。

因此，满足我们的`cpu_speed`要求的任何节点的计算得分都比仅满足`memory_speed`要求的节点高得多 - 但仍然比同时满足两者的节点低。当我们尝试为这个应用程序调度 10 或 100 个新的 Pod 时，您可以看到这种计算是如何有价值的。

## 多个节点亲和性

当我们处理多个节点亲和性时，有一些关键的逻辑要记住。首先，即使只有一个节点亲和性，如果它与同一 Pod 规范下的节点选择器结合使用（这确实是可能的），则节点选择器必须在任何节点亲和性逻辑生效之前满足。这是因为节点选择器只实现硬性要求，并且两者之间没有`OR`逻辑运算符。`OR`逻辑运算符将检查两个要求，并确保它们中至少有一个为真 - 但节点选择器不允许我们这样做。

其次，对于`requiredDuringSchedulingIgnoredDuringExecution`节点亲和性，`nodeSelectorTerms`下的多个条目将在`OR`逻辑运算符中处理。如果满足一个但不是全部，则 Pod 仍将被调度。

最后，对于`matchExpressions`下有多个条目的`nodeSelectorTerm`，所有条目都必须满足 - 这是一个`AND`逻辑运算符。让我们看一个这样的示例 YAML：

pod-with-node-affinity5.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: affinity-test
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: cpu_speed
            operator: Gt
            values:
            - "5"
          - key: memory_speed
            operator: Gt
            values:
            - "4"
  containers:
  - name: speedy-app
    image: speedy-app:latest
```

在这种情况下，如果一个节点的 CPU 速度为`5`，但不满足内存速度要求（或反之亦然），则 Pod 将不会被调度。

关于节点亲和性的最后一件事要注意的是，正如您可能已经注意到的，这两种亲和性类型都不允许我们在我们的污点和容忍设置中可以使用的`NoExecute`功能。

另一种节点亲和性类型 - `requiredDuringSchedulingRequiredDuring execution` - 将在将来的版本中添加此功能。截至 Kubernetes 1.19，这种类型尚不存在。

接下来，我们将看一下 Pod 间亲和性和反亲和性，它提供了 Pod 之间的亲和性定义，而不是为节点定义规则。

# 使用 Pod 间亲和性和反亲和性

Pod 间亲和性和反亲和性让您根据节点上已经存在的其他 Pod 来指定 Pod 应该如何运行。由于集群中的 Pod 数量通常比节点数量要大得多，并且一些 Pod 亲和性和反亲和性规则可能相当复杂，如果您在许多节点上运行许多 Pod，这个功能可能会给您的集群控制平面带来相当大的负载。因此，Kubernetes 文档不建议在集群中有大量节点时使用这些功能。

Pod 亲和性和反亲和性的工作方式有很大不同-让我们先单独看看每个，然后再讨论它们如何结合起来。

## Pod 亲和性

与节点亲和性一样，让我们深入讨论 YAML，以讨论 Pod 亲和性规范的组成部分：

pod-with-pod-affinity.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: not-hungry-app-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: hunger
            operator: In
            values:
            - "1"
            - "2"
        topologyKey: rack
  containers:
  - name: not-hungry-app
    image: not-hungry-app:latest
```

就像节点亲和性一样，Pod 亲和性让我们在两种类型之间进行选择：

+   `preferredDuringSchedulingIgnoredDuringExecution`

+   `requiredDuringSchedulingIgnoredDuringExecution`

与节点亲和性类似，我们可以有一个或多个选择器-因为我们选择的是 Pod 而不是节点，所以它们被称为`labelSelector`。`matchExpressions`功能与节点亲和性相同，但是 Pod 亲和性添加了一个全新的关键字叫做`topologyKey`。

`topologyKey`本质上是一个选择器，限制了调度器应该查看的范围，以查看是否正在运行相同选择器的其他 Pod。这意味着 Pod 亲和性不仅需要意味着同一节点上相同类型（选择器）的其他 Pod；它可以意味着多个节点的组。

让我们回到本章开头的故障域示例。在那个例子中，每个机架都是自己的故障域，每个机架有多个节点。为了将这个概念扩展到`topologyKey`，我们可以使用`rack=1`或`rack=2`为每个机架上的节点打上标签。然后我们可以使用`topologyKey`机架，就像我们在 YAML 中所做的那样，指定调度器应该检查所有运行在具有相同`topologyKey`的节点上的 Pod（在这种情况下，这意味着同一机架上的`Node 1`和`Node 2`上的所有 Pod）以应用 Pod 亲和性或反亲和性规则。

因此，将我们的示例 YAML 全部加起来，告诉调度器的是：

+   这个 Pod *必须*被调度到具有标签`rack`的节点上，其中标签`rack`的值将节点分成组。

+   然后 Pod 将被调度到一个组中，该组中已经存在一个带有标签`hunger`和值为 1 或 2 的 Pod。

基本上，我们将我们的集群分成拓扑域 - 在这种情况下是机架 - 并指示调度器仅在共享相同拓扑域的节点上将相似的 Pod 一起调度。这与我们第一个故障域示例相反，如果可能的话，我们不希望 Pod 共享相同的域 - 但也有理由希望相似的 Pod 在同一域上。例如，在多租户设置中，租户希望在域上拥有专用硬件租用权，您可以确保属于某个租户的每个 Pod 都被调度到完全相同的拓扑域。

您可以以相同的方式使用`preferredDuringSchedulingIgnoredDuringExecution`。在我们讨论反亲和性之前，这里有一个带有 Pod 亲和性和`preferred`类型的示例：

pod-with-pod-affinity2.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: not-hungry-app-affinity
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: hunger
              operator: Lt
              values:
              - "3"
          topologyKey: rack
  containers:
  - name: not-hungry-app
    image: not-hungry-app:latest
```

与之前一样，在这个代码块中，我们有我们的`weight` - 在这种情况下是`50` - 和我们的表达式匹配 - 在这种情况下，使用小于（`Lt`）运算符。这种亲和性将促使调度器尽力将 Pod 调度到一个节点上，该节点上已经运行着一个`hunger`小于 3 的 Pod，或者与另一个在同一机架上运行着`hunger`小于 3 的 Pod。调度器使用`weight`来比较节点 - 正如在节点亲和性部分讨论的那样 - *使用节点亲和性控制 Pod*（参见`pod-with-node-affinity4.yaml`）。在这种特定情况下，`50`的权重并没有任何区别，因为亲和性列表中只有一个条目。

Pod 反亲和性使用相同的选择器和拓扑结构来扩展这种范例-让我们详细看一下它们。

## Pod 反亲和性

Pod 反亲和性允许您阻止 Pod 在与匹配选择器的 Pod 相同的拓扑域上运行。它们实现了与 Pod 亲和性相反的逻辑。让我们深入了解一些 YAML，并解释一下它是如何工作的：

pod-with-pod-anti-affinity.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: hungry-app
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: hunger
              operator: In
              values:
              - "4"
              - "5"
          topologyKey: rack
  containers:
  - name: hungry-app
    image: hungry-app
```

与 Pod 亲和性类似，我们使用`affinity`键来指定`podAntiAffinity`下的反亲和性的位置。与 Pod 亲和性一样，我们可以使用`preferredDuringSchedulingIgnoredDuringExecution`或`requireDuringSchedulingIgnoredDuringExecution`。我们甚至可以使用与 Pod 亲和性相同的选择器语法。

语法上唯一的实际区别是在`affinity`键下使用`podAntiAffinity`。

那么，这个 YAML 文件是做什么的呢？在这种情况下，我们建议调度器（一个`soft`要求）应该尝试将这个 Pod 调度到一个节点上，在这个节点或具有相同值的`rack`标签的任何其他节点上都没有运行带有`hunger`标签值为 4 或 5 的 Pod。我们告诉调度器*尽量不要将这个 Pod 与任何额外饥饿的 Pod 放在一起*。

这个功能为我们提供了一个很好的方法来按故障域分隔 Pod - 我们可以将每个机架指定为一个域，并给它一个与自己相同类型的反亲和性。这将使调度器将 Pod 的克隆（或尝试在首选亲和性中）调度到不在相同故障域的节点上，从而在发生域故障时提供更大的可用性。

我们甚至可以选择结合 Pod 的亲和性和反亲和性。让我们看看这样可以如何工作。

## 结合亲和性和反亲和性

这是一个情况，你可以真正给你的集群控制平面增加不必要的负载。结合 Pod 的亲和性和反亲和性可以允许传递给 Kubernetes 调度器的非常微妙的规则。

让我们看一些结合这两个概念的部署规范的 YAML。请记住，亲和性和反亲和性是应用于 Pod 的概念 - 但我们通常不会指定没有像部署或副本集这样的控制器的 Pod。因此，这些规则是在部署 YAML 中的 Pod 规范级别应用的。出于简洁起见，我们只显示了这个部署的 Pod 规范部分，但你可以在 GitHub 存储库中找到完整的文件。

pod-with-both-antiaffinity-and-affinity.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hungry-app-deployment
# SECTION REMOVED FOR CONCISENESS  
     spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - other-hungry-app
            topologyKey: "rack"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - hungry-app-cache
            topologyKey: "rack"
      containers:
      - name: hungry-app
        image: hungry-app:latest
```

在这个代码块中，我们告诉调度器将我们的部署中的 Pod 视为这样：Pod 必须被调度到具有`rack`标签的节点上，以便它或具有相同值的`rack`标签的任何其他节点都有一个带有`app=hungry-label-cache`的 Pod。

其次，调度器必须尝试将 Pod 调度到具有`rack`标签的节点上，以便它或具有相同值的`rack`标签的任何其他节点都没有运行带有`app=other-hungry-app`标签的 Pod。

简而言之，我们希望我们的`hungry-app`的 Pod 在与`hungry-app-cache`相同的拓扑结构中运行，并且如果可能的话，我们不希望它们与`other-hungry-app`在相同的拓扑结构中。

由于强大的力量伴随着巨大的责任，而我们的 Pod 亲和性和反亲和性工具既强大又降低性能，Kubernetes 确保对您可以使用它们的可能方式设置了一些限制，以防止奇怪的行为或重大性能问题。

## Pod 亲和性和反亲和性限制

亲和性和反亲和性的最大限制是，您不允许使用空的`topologyKey`。如果不限制调度器将作为单个拓扑类型处理的内容，可能会发生一些非常意外的行为。

第二个限制是，默认情况下，如果您使用反亲和性的硬版本-`requiredOnSchedulingIgnoredDuringExecution`，您不能只使用任何标签作为`topologyKey`。

Kubernetes 只允许您使用`kubernetes.io/hostname`标签，这基本上意味着如果您使用`required`反亲和性，您只能在每个节点上有一个拓扑。这个限制对于`prefer`反亲和性或任何亲和性都不存在，甚至是`required`。可以更改此功能，但需要编写自定义准入控制器-我们将在*第十二章*中讨论，*Kubernetes 安全性和合规性*，以及*第十三章*，*使用 CRD 扩展 Kubernetes*。

到目前为止，我们对放置控件的工作尚未讨论命名空间。但是，对于 Pod 亲和性和反亲和性，它们确实具有相关性。

## Pod 亲和性和反亲和性命名空间

由于 Pod 亲和性和反亲和性会根据其他 Pod 的位置而改变行为，命名空间是决定哪些 Pod 计入或反对亲和性或反亲和性的相关因素。

默认情况下，调度器只会查看创建具有亲和性或反亲和性的 Pod 的命名空间。对于我们之前的所有示例，我们没有指定命名空间，因此将使用默认命名空间。

如果要添加一个或多个命名空间，其中 Pod 将影响亲和性或反亲和性，可以使用以下 YAML：

pod-with-anti-affinity-namespace.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: hungry-app
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: hunger
              operator: In
              values:
              - "4"
              - "5"
          topologyKey: rack
          namespaces: ["frontend", "backend", "logging"]
  containers:
  - name: hungry-app
    image: hungry-app
```

在这个代码块中，调度器将在尝试匹配反亲和性时查看前端、后端和日志命名空间（如您在`podAffinityTerm`块中的`namespaces`键中所见）。这允许我们限制调度器在验证其规则时操作的命名空间。

# 总结

在本章中，我们了解了 Kubernetes 提供的一些不同控件，以强制执行调度器通过规则来放置 Pod。我们了解到有“硬”要求和“软”规则，后者是调度器尽最大努力但不一定阻止违反规则的 Pod 被放置。我们还了解了一些实施调度控件的原因，比如现实生活中的故障域和多租户。

我们了解到有一些简单的方法可以影响 Pod 的放置，比如节点选择器和节点名称，还有更高级的方法，比如污点和容忍，Kubernetes 本身也默认使用这些方法。最后，我们发现 Kubernetes 提供了一些高级工具，用于节点和 Pod 的亲和性和反亲和性，这些工具允许我们创建复杂的调度规则。

在下一章中，我们将讨论 Kubernetes 上的可观察性。我们将学习如何查看应用程序日志，还将使用一些很棒的工具实时查看我们集群中正在发生的事情。

# 问题

1.  节点选择器和节点名称字段之间有什么区别？

1.  Kubernetes 如何使用系统提供的污点和容忍？出于什么原因？

1.  在使用多种类型的 Pod 亲和性或反亲和性时，为什么要小心？

1.  如何在多个故障区域之间平衡可用性，并出于性能原因进行合作，为三层 Web 应用程序提供一个例子？使用节点或 Pod 的亲和性和反亲和性。

# 进一步阅读

+   要了解有关默认系统污点和容忍的更深入解释，请访问[`kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions`](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions)。

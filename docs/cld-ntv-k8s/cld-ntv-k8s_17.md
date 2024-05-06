# 第十三章：使用 CRD 扩展 Kubernetes

本章解释了扩展 Kubernetes 功能的许多可能性。它从讨论**自定义资源定义**（**CRD**）开始，这是一种 Kubernetes 本地的方式，用于指定可以通过熟悉的`kubectl`命令（如`get`、`create`、`describe`和`apply`）对其进行操作的自定义资源。接下来是对运算符模式的讨论，这是 CRD 的扩展。然后详细介绍了云提供商附加到其 Kubernetes 实现的一些钩子，并以对更大的云原生生态系统的简要介绍结束。使用本章学到的概念，您将能够设计和开发对 Kubernetes 集群的扩展，解锁高级使用模式。

本章的案例研究将包括创建两个简单的 CRD 来支持一个示例应用程序。我们将从 CRD 开始，这将让您对扩展如何构建在 Kubernetes API 上有一个良好的基础理解。

在本章中，我们将涵盖以下主题：

+   如何使用**自定义资源定义**（**CRD**）扩展 Kubernetes

+   使用 Kubernetes 运算符进行自管理功能

+   使用特定于云的 Kubernetes 扩展

+   与生态系统集成

# 技术要求

为了运行本章中详细介绍的命令，您需要一台支持`kubectl`命令行工具的计算机，以及一个正常运行的 Kubernetes 集群。请参阅*第一章*，*与 Kubernetes 通信*，了解快速启动和运行 Kubernetes 的几种方法，以及如何安装`kubectl`工具。

本章中使用的代码可以在书籍的 GitHub 存储库中找到，网址为[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter13`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter13)。

# 如何使用自定义资源定义扩展 Kubernetes

让我们从基础知识开始。什么是 CRD？我们知道 Kubernetes 有一个 API 模型，我们可以对资源执行操作。一些 Kubernetes 资源的例子（现在你应该对它们非常熟悉）是 Pods、PersistentVolumes、Secrets 等。

现在，如果我们想在集群中实现一些自定义功能，编写我们自己的控制器，并将控制器的状态存储在某个地方，我们可以，当然，将我们自定义功能的状态存储在 Kubernetes 或其他地方运行的 SQL 或 NoSQL 数据库中（这实际上是扩展 Kubernetes 的策略之一）-但是如果我们的自定义功能更像是 Kubernetes 功能的扩展，而不是完全独立的应用程序呢？

在这种情况下，我们有两个选择：

+   自定义资源定义

+   API 聚合

API 聚合允许高级用户在 Kubernetes API 服务器之外构建自己的资源 API，并使用自己的存储，然后在 API 层聚合这些资源，以便可以使用 Kubernetes API 进行查询。这显然是非常可扩展的，实质上只是使用 Kubernetes API 作为代理来使用您自己的自定义功能，这可能实际上与 Kubernetes 集成，也可能不会。

另一个选择是 CRDs，我们可以使用 Kubernetes API 和底层数据存储（`etcd`）而不是构建我们自己的。我们可以使用我们知道的`kubectl`和`kube api`方法与我们自己的自定义功能进行交互。

在这本书中，我们不会讨论 API 聚合。虽然比 CRDs 更灵活，但这是一个高级主题，需要对 Kubernetes API 有深入的了解，并仔细阅读 Kubernetes 文档以正确实施。您可以在 Kubernetes 文档中了解更多关于 API 聚合的信息[`kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/`](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)。

所以，现在我们知道我们正在使用 Kubernetes 控制平面作为我们自己的有状态存储来存储我们的新自定义功能，我们需要一个模式。类似于 Kubernetes 中 Pod 资源规范期望特定字段和配置，我们可以告诉 Kubernetes 我们对新的自定义资源期望什么。现在让我们来看一下 CRD 的规范。

## 编写自定义资源定义

对于 CRDs，Kubernetes 使用 OpenAPI V3 规范。有关 OpenAPI V3 的更多信息，您可以查看官方文档[`github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md`](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md)，但我们很快将看到这如何转化为 Kubernetes CRD 定义。

让我们来看一个 CRD 规范的示例。现在让我们明确一点，这不是任何特定记录的 YAML 的样子。相反，这只是我们在 Kubernetes 内部定义 CRD 的要求的地方。一旦创建，Kubernetes 将接受与规范匹配的资源，我们就可以开始制作我们自己的这种类型的记录。

这里有一个 CRD 规范的示例 YAML，我们称之为`delayedjob`。这个非常简单的 CRD 旨在延迟启动容器镜像作业，这样用户就不必为他们的容器编写延迟启动的脚本。这个 CRD 非常脆弱，我们不建议任何人真正使用它，但它确实很好地突出了构建 CRD 的过程。让我们从一个完整的 CRD 规范 YAML 开始，然后分解它：

自定义资源定义-1.yaml

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: delayedjobs.delayedresources.mydomain.com
spec:
  group: delayedresources.mydomain.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                delaySeconds:
                  type: integer
                image:
                  type: string
  scope: Namespaced
  conversion:
    strategy: None
  names:
    plural: delayedjobs
    singular: delayedjob
    kind: DelayedJob
    shortNames:
    - dj
```

让我们来审视一下这个文件的部分。乍一看，它看起来像是您典型的 Kubernetes YAML 规范 - 因为它就是！在`apiVersion`字段中，我们有`apiextensions.k8s.io/v1`，这是自 Kubernetes `1.16`以来的标准（在那之前是`apiextensions.k8s.io/v1beta1`）。我们的`kind`将始终是`CustomResourceDefinition`。

`metadata`字段是当事情开始变得特定于我们的资源时。我们需要将`name`元数据字段结构化为我们资源的`复数`形式，然后是一个句号，然后是它的组。让我们从我们的 YAML 文件中快速偏离一下，讨论一下 Kubernetes API 中组的工作原理。

### 理解 Kubernetes API 组

组是 Kubernetes 在其 API 中分割资源的一种方式。每个组对应于 Kubernetes API 服务器的不同子路径。

默认情况下，有一个名为核心组的遗留组 - 它对应于在 Kubernetes REST API 的`/api/v1`端点上访问的资源。因此，这些遗留组资源在其 YAML 规范中具有`apiVersion: v1`。核心组中资源的一个例子是 Pod。

接下来，有一组命名的组 - 这些组对应于可以在`REST` URL 上访问的资源，形式为`/apis/<GROUP NAME>/<VERSION>`。这些命名的组构成了 Kubernetes 资源的大部分。然而，最古老和最基本的资源，如 Pod、Service、Secret 和 Volume，都在核心组中。一个在命名组中的资源的例子是`StorageClass`资源，它在`storage.k8s.io`组中。

重要说明

要查看哪个资源属于哪个组，您可以查看您正在使用的 Kubernetes 版本的官方 Kubernetes API 文档。例如，版本`1.18`的文档将位于[`kubernetes.io/docs/reference/generated/kubernetes-api/v1.18`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18)。

CRD 可以指定自己的命名组，这意味着特定的 CRD 将在 Kubernetes API 服务器可以监听的`REST`端点上可用。考虑到这一点，让我们回到我们的 YAML 文件，这样我们就可以讨论 CRD 的主要部分-版本规范。

### 理解自定义资源定义版本

正如您所看到的，我们选择了组`delayedresources.mydomain.com`。该组理论上将包含任何其他延迟类型的 CRD-例如，`DelayedDaemonSet`或`DelayedDeployment`。

接下来，我们有我们的 CRD 的主要部分。在`versions`下，我们可以定义一个或多个 CRD 版本（在`name`字段中），以及该 CRD 版本的 API 规范。然后，当您创建 CRD 的实例时，您可以在 YAML 文件的`apiVersion`键的版本参数中定义您将使用的版本-例如，`apps/v1`，或在这种情况下，`delayedresources.mydomain.com/v1`。

每个版本项还有一个`served`属性，这实质上是一种定义给定版本是否启用或禁用的方式。如果`served`为`false`，则该版本将不会被 Kubernetes API 创建，并且该版本的 API 请求（或`kubectl`命令）将失败。

此外，还可以在特定版本上定义一个`deprecated`键，这将导致 Kubernetes 在使用弃用版本进行 API 请求时返回警告消息。这就是带有弃用版本的 CRD 的`yaml`文件的样子-我们已删除了一些规范，以使 YAML 文件简短：

自定义资源定义-2.yaml

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: delayedjob.delayedresources.mydomain.com
spec:
  group: delayedresources.mydomain.com
  versions:
    - name: v1
      served: true
      storage: false
      deprecated: true
      deprecationWarning: "DelayedJob v1 is deprecated!"
      schema:
        openAPIV3Schema:
		…
    - name: v2
      served: true
      storage: true
      schema:
        openAPIV3Schema:
		...
  scope: Namespaced
  conversion:
    strategy: None
  names:
    plural: delayedjobs
    singular: delayedjob
    kind: DelayedJob
    shortNames:
    - dj
```

正如您所看到的，我们已将`v1`标记为已弃用，并且还包括一个弃用警告，以便 Kubernetes 作为响应发送。如果我们不包括弃用警告，将使用默认消息。

进一步向下移动，我们有`storage`键，它与`served`键交互。这是必要的原因是，虽然 Kubernetes 支持同时拥有多个活动（也就是`served`）版本的资源，但是只能有一个版本存储在控制平面中。然而，`served`属性意味着 API 可以提供多个版本的资源。那么这是如何工作的呢？

答案是，Kubernetes 将把 CRD 对象从存储的版本转换为您要求的版本（或者反过来，在创建资源时）。

这种转换是如何处理的？让我们跳过其余的版本属性，看看`conversion`键是如何工作的。

`conversion`键允许您指定 Kubernetes 将如何在您的服务版本和存储版本之间转换 CRD 对象的策略。如果两个版本相同-例如，如果您请求一个`v1`资源，而存储的版本是`v1`，那么不会发生转换。

截至 Kubernetes 1.13 的默认值是`none`。使用`none`设置，Kubernetes 不会在字段之间进行任何转换。它只会包括应该出现在`served`（或存储，如果创建资源）版本上的字段。

另一个可能的转换策略是`Webhook`，它允许您定义一个自定义 Webhook，该 Webhook 将接收一个版本并对其进行适当的转换为您想要的版本。这里有一个使用`Webhook`转换策略的 CRD 示例-为了简洁起见，我们省略了一些版本模式：

Custom-resource-definition-3.yaml

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: delayedjob.delayedresources.mydomain.com
spec:
  group: delayedresources.mydomain.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
		...
  scope: Namespaced
  conversion:
    strategy: Webhook
    webhook:
      clientConfig:
        url: "https://webhook-conversion.com/delayedjob"
  names:
    plural: delayedjobs
    singular: delayedjob
    kind: DelayedJob
    shortNames:
    - dj
```

正如您所看到的，`Webhook`策略让我们定义一个 URL，请求将发送到该 URL，其中包含有关传入资源对象、其当前版本和需要转换为的版本的信息。

想法是我们的`Webhook`服务器将处理转换并传回修正后的 Kubernetes 资源对象。`Webhook`策略是复杂的，可以有许多可能的配置，我们在本书中不会深入讨论。

重要提示

要了解如何配置转换 Webhooks，请查看官方 Kubernetes 文档[`kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/`](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)。

现在，回到我们在 YAML 中的`version`条目！在`served`和`storage`键下，我们看到`schema`对象，其中包含我们资源的实际规范。如前所述，这遵循 OpenAPI Spec v3 模式。

`schema`对象，由于空间原因已从前面的代码块中删除，如下所示：

自定义资源定义-3.yaml（续）

```
     schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                delaySeconds:
                  type: integer
                image:
                  type: string
```

正如您所看到的，我们支持`delaySeconds`字段，它将是一个整数，以及`image`，它是一个与我们的容器映像相对应的字符串。如果我们真的想要使`DelayedJob`达到生产就绪状态，我们会希望包括各种其他选项，使其更接近原始的 Kubernetes Job 资源 - 但这不是我们的意图。

在原始代码块中进一步向后移动，超出版本列表，我们看到一些其他属性。首先是`scope`属性，可以是`Cluster`或`Namespaced`。这告诉 Kubernetes 是否将 CRD 对象的实例视为特定于命名空间的资源（例如 Pods，Deployments 等），还是作为集群范围的资源 - 就像命名空间本身一样，因为在命名空间中获取命名空间对象是没有意义的！

最后，我们有`names`块，它允许您定义资源名称的复数和单数形式，以在各种情况下使用（例如，`kubectl get pods`和`kubectl get pod`都可以工作）。

`names`块还允许您定义驼峰命名的`kind`值，在资源 YAML 中将使用该值，以及一个或多个`shortNames`，可以用来在 API 或`kubectl`中引用该资源 - 例如，`kubectl get po`。

解释了我们的 CRD 规范 YAML 后，让我们来看一下我们 CRD 的一个实例 - 正如我们刚刚审查的规范所定义的，YAML 将如下所示：

Delayed-job.yaml

```
apiVersion: delayedresources.mydomain.com/v1
kind: DelayedJob
metadata:
  name: my-instance-of-delayed-job
spec:
  delaySeconds: 6000
  image: "busybox"
```

正如您所看到的，这就像我们的 CRD 定义了这个对象。现在，所有的部分都就位了，让我们测试一下我们的 CRD！

### 测试自定义资源定义

让我们继续在 Kubernetes 上测试我们的 CRD 概念：

1.  首先，让我们在 Kubernetes 中创建 CRD 规范 - 就像我们创建任何其他对象一样：

```
kubectl apply -f delayedjob-crd-spec.yaml
```

这将导致以下输出：

```
customresourcedefinition "delayedjob.delayedresources.mydomain.com" has been created
```

1.  现在，Kubernetes 将接受对我们的`DelayedJob`资源的请求。我们可以通过最终使用前面的资源 YAML 创建一个来测试这一点：

```
kubectl apply -f my-delayed-job.yaml
```

如果我们正确定义了我们的 CRD，我们将看到以下输出：

```
delayedjob "my-instance-of-delayed-job" has been created
```

正如您所看到的，Kubernetes API 服务器已成功创建了我们的`DelayedJob`实例！

现在，您可能会问一个非常相关的问题 - 现在怎么办？这是一个很好的问题，因为事实上，到目前为止，我们实际上什么也没有做，只是向 Kubernetes API 数据库添加了一个新的`表`。

仅仅因为我们给我们的`DelayedJob`资源一个应用程序镜像和一个`delaySeconds`字段，并不意味着我们打算的任何功能实际上会发生。通过创建我们的`DelayedJob`实例，我们只是向那个`表`添加了一个条目。我们可以使用 Kubernetes API 或`kubectl`命令获取它，编辑它或删除它，但没有实现任何应用功能。

为了让我们的`DelayedJob`资源真正做些什么，我们需要一个自定义控制器，它将获取我们的`DelayedJob`实例并对其进行操作。最终，我们仍然需要使用官方 Kubernetes 资源（如 Pods 等）来实现实际的容器功能。

这就是我们现在要讨论的。有许多构建 Kubernetes 自定义控制器的方法，但流行的方法是**运算符模式**。让我们继续下一节，看看我们如何让我们的`DelayedJob`资源拥有自己的生命。

# 使用 Kubernetes 运算符自管理功能

在没有首先讨论**Operator Framework**之前，不可能讨论 Kubernetes 运算符。一个常见的误解是，运算符是通过 Operator Framework 专门构建的。Operator Framework 是一个开源框架，最初由 Red Hat 创建，旨在简化编写 Kubernetes 运算符。

实际上，运算符只是一个与 Kubernetes 接口并对资源进行操作的自定义控制器。Operator Framework 是一种使 Kubernetes 运算符的一种偏见方式，但还有许多其他开源框架可以使用 - 或者，您可以从头开始制作一个！

使用框架构建运算符时，最流行的两个选项是前面提到的**Operator Framework**和**Kubebuilder**。

这两个项目有很多共同之处。它们都使用`controller-tools`和`controller-runtime`，这是两个由 Kubernetes 项目官方支持的构建 Kubernetes 控制器的库。如果您从头开始构建运算符，使用这些官方支持的控制器库将使事情变得更容易。

与 Operator Framework 不同，Kubebuilder 是 Kubernetes 项目的官方部分，就像`controller-tools`和`controller-runtime`库一样 - 但这两个项目都有其优缺点。重要的是，这两个选项以及一般的 Operator 模式都是在集群上运行控制器。这似乎是最好的选择，但你也可以在集群外运行控制器，并且它可以正常工作。要开始使用 Operator Framework，请查看官方 GitHub 网站[`github.com/operator-framework`](https://github.com/operator-framework)。对于 Kubebuilder，你可以查看[`github.com/kubernetes-sigs/kubebuilder`](https://github.com/kubernetes-sigs/kubebuilder)。

大多数操作员，无论使用哪种框架，都遵循控制循环范式 - 让我们看看这个想法是如何工作的。

## 映射操作员控制循环

控制循环是系统设计和编程中的控制方案，由一系列逻辑过程组成的永无止境的循环。通常，控制循环实现了一种测量-分析-调整的方法，它测量系统的当前状态，分析需要做出哪些改变使其与预期状态一致，然后调整系统组件使其与预期状态一致（或至少更接近预期状态）。

在 Kubernetes 的操作员或控制器中，这个操作通常是这样工作的：

1.  首先是一个“监视”步骤 - 也就是监视 Kubernetes API 中预期状态的变化，这些状态存储在`etcd`中。

1.  然后是一个“分析”步骤 - 控制器决定如何使集群状态与预期状态一致。

1.  最后是一个“更新”步骤 - 更新集群状态以实现集群变化的意图。

为了帮助理解控制循环，这里有一个图表显示了这些部分是如何组合在一起的：

![图 13.1 - 测量分析更新循环](img/B14790_13_01.jpg)

图 13.1 - 测量分析更新循环

让我们使用 Kubernetes 调度器来说明这一点 - 它本身就是一个控制循环过程：

1.  让我们从一个假设的集群开始，处于稳定状态：所有的 Pod 都已经调度，节点也健康，一切都在正常运行。

1.  然后，用户创建了一个新的 Pod。

我们之前讨论过 kubelet 是基于`pull`的工作方式。这意味着当 kubelet 在其节点上创建一个 Pod 时，该 Pod 已经通过调度器分配给了该节点。然而，当通过`kubectl create`或`kubectl apply`命令首次创建 Pod 时，该 Pod 尚未被调度或分配到任何地方。这就是我们的调度器控制循环开始的地方：

1.  第一步是**测量**，调度器从 Kubernetes API 读取状态。当从 API 列出 Pod 时，它发现其中一个 Pod 未分配给任何节点。现在它转移到下一步。

1.  接下来，调度器对集群状态和 Pod 需求进行分析，以决定将 Pod 分配给哪个节点。正如我们在前几章中讨论的那样，这涉及到 Pod 资源限制和请求、节点状态、放置控制等等，这使得它成为一个相当复杂的过程。一旦处理完成，更新步骤就可以开始了。

1.  最后，**更新** - 调度器通过将 Pod 分配给从*步骤 2*分析中获得的节点来更新集群状态。此时，kubelet 接管自己的控制循环，并为其节点上的 Pod 创建相关的容器。

接下来，让我们将从调度器控制循环中学到的内容应用到我们自己的`DelayedJob`资源上。

## 为自定义资源定义设计运算符

实际上，为我们的`DelayedJob` CRD 编写运算符超出了我们书的范围，因为这需要对编程语言有所了解。如果您选择使用 Go 构建运算符，它提供了与 Kubernetes SDK、**controller-tools**和**controller-runtime**最多的互操作性，但任何可以编写 HTTP 请求的编程语言都可以使用，因为这是所有 SDK 的基础。

然而，我们仍将逐步实现`DelayedJob` CRD 的运算符步骤，使用一些伪代码。让我们一步一步来。

### 步骤 1：测量

首先是**测量**步骤，我们将在我们的伪代码中实现为一个永远运行的`while`循环。在生产实现中，会有去抖动、错误处理和一堆其他问题，但是对于这个说明性的例子，我们会保持简单。

看一下这个循环的伪代码，这实际上是我们应用程序的主要功能：

Main-function.pseudo

```
// The main function of our controller
function main() {
  // While loop which runs forever
  while() {
     // fetch the full list of delayed job objects from the cluster
	var currentDelayedJobs = kubeAPIConnector.list("delayedjobs");
     // Call the Analysis step function on the list
     var jobsToSchedule = analyzeDelayedJobs(currentDelayedJobs);
     // Schedule our Jobs with added delay
     scheduleDelayedJobs(jobsToSchedule);
     wait(5000);
  }
}
```

正如您所看到的，在我们的`main`函数中的循环调用 Kubernetes API 来查找存储在`etcd`中的`delayedjobs` CRD 列表。这是`measure`步骤。然后调用分析步骤，并根据其结果调用更新步骤来安排需要安排的任何`DelayedJobs`。

重要说明

请记住，在这个例子中，Kubernetes 调度程序仍然会执行实际的容器调度 - 但是我们首先需要将我们的`DelayedJob`简化为官方的 Kubernetes 资源。

在更新步骤之后，我们的循环在执行循环之前等待完整的 5 秒。这确定了控制循环的节奏。接下来，让我们继续进行分析步骤。

### 步骤 2：分析

接下来，让我们来审查我们操作员的**Analysis**步骤，这是我们控制器伪代码中的`analyzeDelayedJobs`函数：

分析函数伪代码

```
// The analysis function
function analyzeDelayedJobs(listOfDelayedJobs) {
  var listOfJobsToSchedule = [];
  foreach(dj in listOfDelayedJobs) {
    // Check if dj has been scheduled, if not, add a Job object with
    // added delay command to the to schedule array
    if(dj.annotations["is-scheduled"] != "true") {
      listOfJobsToSchedule.push({
        Image: dj.image,
        Command: "sleep " + dj.delaySeconds + "s",
        originalDjName: dj.name
      });
    }
  }
  return listOfJobsToSchedule;  
}
```

正如您所看到的，前面的函数循环遍历了从**Measure**循环传递的集群中的`DelayedJob`对象列表。然后，它检查`DelayedJob`是否已经通过检查对象的注释之一的值来进行了调度。如果尚未安排，它将向名为`listOfJobsToSchedule`的数组添加一个对象，该数组包含`DelayedJob`对象中指定的图像，一个命令以睡眠指定的秒数，以及`DelayedJob`的原始名称，我们将在**Update**步骤中用来标记为已调度。

最后，在**Analyze**步骤中，`analyzeDelayedJobs`函数将我们新创建的`listOfJobsToSchedule`数组返回给主函数。让我们用最终的更新步骤来结束我们的操作员设计，这是我们主循环中的`scheduleDelayedJobs`函数。

### 步骤 3：更新

最后，我们的控制循环的**Update**部分将从我们的分析中获取输出，并根据需要更新集群以创建预期的状态。以下是伪代码：

更新函数伪代码

```
// The update function
function scheduleDelayedJobs(listOfJobs) {
  foreach(job in listOfDelayedJobs) {
    // First, go ahead and schedule a regular Kubernetes Job
    // which the Kube scheduler can pick up on.
    // The delay seconds have already been added to the job spec
    // in the analysis step
    kubeAPIConnector.create("job", job.image, job.command);
    // Finally, mark our original DelayedJob with a "scheduled"
    // attribute so our controller doesn't try to schedule it again
    kubeAPIConnector.update("delayedjob", job.originalDjName,
    annotations: {
      "is-scheduled": "true"
    });
  } 
}
```

在这种情况下，我们正在使用从我们的`DelayedJob`对象派生的常规 Kubernetes 对象，并在 Kubernetes 中创建它，以便`Kube`调度程序可以找到它，创建相关的 Pod 并管理它。一旦我们使用延迟创建了常规作业对象，我们还会使用注释更新我们的`DelayedJob` CRD 实例，将`is-scheduled`注释设置为`true`，以防止它被重新调度。

这完成了我们的控制循环 - 从这一点开始，`Kube`调度器接管并且我们的 CRD 被赋予生命作为一个 Kubernetes Job 对象，它控制一个 Pod，最终分配给一个 Node，并且一个容器被调度来运行我们的代码！

当然，这个例子是高度简化的，但你会惊讶地发现有多少 Kubernetes 操作员执行一个简单的控制循环来协调 CRD 并将其简化为基本的 Kubernetes 资源。操作员可以变得非常复杂，并执行特定于应用程序的功能，例如备份数据库、清空持久卷等，但这种功能通常与被控制的内容紧密耦合。

现在我们已经讨论了 Kubernetes 控制器中的操作员模式，我们可以谈谈一些特定于云的 Kubernetes 控制器的开源选项。

# 使用特定于云的 Kubernetes 扩展

通常情况下，在托管的 Kubernetes 服务（如 Amazon EKS、Azure AKS 和 Google Cloud 的 GKE）中默认可用，特定于云的 Kubernetes 扩展和控制器可以与相关的云平台紧密集成，并且可以轻松地从 Kubernetes 控制其他云资源。

即使不添加任何额外的第三方组件，许多这些特定于云的功能都可以通过**云控制器管理器**（**CCM**）组件在上游 Kubernetes 中使用，该组件包含许多与主要云提供商集成的选项。这通常是在每个公共云上的托管 Kubernetes 服务中默认启用的功能，但它们可以与在特定云平台上运行的任何集群集成，无论是托管还是非托管。

在本节中，我们将回顾一些常见的云扩展到 Kubernetes 中，包括**云控制器管理器（CCM）**和需要安装其他控制器的功能，例如**external-dns**和**cluster-autoscaler**。让我们从一些常用的 CCM 功能开始。

## 了解云控制器管理器组件

正如在*第一章*中所述，*与 Kubernetes 通信*，CCM 是一个官方支持的 Kubernetes 控制器，提供了对几个公共云服务功能的钩子。为了正常运行，CCM 组件需要以访问特定云服务的权限启动，例如在 AWS 中的 IAM 角色。

对于官方支持的云，如 AWS、Azure 和 Google Cloud，CCM 可以简单地作为集群中的 DaemonSet 运行。我们使用 DaemonSet，因为 CCM 可以执行诸如在云提供商中创建持久存储等任务，并且需要能够将存储附加到特定的节点。如果您使用的是官方不支持的云，您可以为该特定云运行 CCM，并且应该遵循该项目中的具体说明。这些替代类型的 CCM 通常是开源的，可以在 GitHub 上找到。关于安装 CCM 的具体信息，让我们继续下一节。

## 安装 cloud-controller-manager

通常，在创建集群时配置 CCM。如前一节所述，托管服务，如 EKS、AKS 和 GKE，将已经启用此组件，但即使 Kops 和 Kubeadm 也将 CCM 组件作为安装过程中的一个标志暴露出来。

假设您尚未以其他方式安装 CCM 并计划使用上游版本的官方支持的公共云之一，您可以将 CCM 安装为 DaemonSet。

首先，您需要一个`ServiceAccount`：

Service-account.yaml

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloud-controller-manager
  namespace: kube-system
```

这个`ServiceAccount`将被用来给予 CCM 必要的访问权限。

接下来，我们需要一个`ClusterRoleBinding`：

Clusterrolebinding.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:cloud-controller-manager
subjects:
- kind: ServiceAccount
  name: cloud-controller-manager
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

如您所见，我们需要给`cluster-admin`角色访问我们的 CCM 服务账户。CCM 将需要能够编辑节点，以及其他一些操作。

最后，我们可以部署 CCM 的`DaemonSet`本身。您需要使用适合您特定云提供商的正确设置填写此 YAML 文件-查看您云提供商关于 Kubernetes 的文档以获取这些信息。

`DaemonSet`规范非常长，因此我们将分两部分进行审查。首先，我们有`DaemonSet`的模板，其中包含所需的标签和名称：

Daemonset.yaml

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    k8s-app: cloud-controller-manager
  name: cloud-controller-manager
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: cloud-controller-manager
  template:
    metadata:
      labels:
        k8s-app: cloud-controller-manager
```

正如您所看到的，为了匹配我们的`ServiceAccount`，我们在`kube-system`命名空间中运行 CCM。我们还使用`k8s-app`标签对`DaemonSet`进行标记，以将其区分为 Kubernetes 控制平面组件。

接下来，我们有`DaemonSet`的规范：

Daemonset.yaml（续）

```
    spec:
      serviceAccountName: cloud-controller-manager
      containers:
      - name: cloud-controller-manager
        image: k8s.gcr.io/cloud-controller-manager:<current ccm version for your version of k8s>
        command:
        - /usr/local/bin/cloud-controller-manager
        - --cloud-provider=<cloud provider name>
        - --leader-elect=true
        - --use-service-account-credentials
        - --allocate-node-cidrs=true
        - --configure-cloud-routes=true
        - --cluster-cidr=<CIDR of the cluster based on Cloud Provider>
      tolerations:
      - key: node.cloudprovider.kubernetes.io/uninitialized
        value: "true"
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      nodeSelector:
        node-role.kubernetes.io/master: ""
```

正如您所看到的，此规范中有一些地方需要查看您选择的云提供商的文档或集群网络设置，以找到正确的值。特别是在网络标志中，例如`--cluster-cidr`和`--configure-cloud-routes`，这些值可能会根据您如何设置集群而改变，即使在单个云提供商上也是如此。

既然我们在集群中以某种方式运行 CCM，让我们深入了解它提供的一些功能。

## 了解云控制器管理器的功能

默认的 CCM 在一些关键领域提供了功能。首先，CCM 包含了节点、路由和服务的子控制器。让我们依次审查每个，看看它为我们提供了什么，从节点/节点生命周期控制器开始。

### CCM 节点/节点生命周期控制器

CCM 节点控制器确保集群状态，就是集群中的节点与云提供商系统中的节点是等价的。一个简单的例子是 AWS 中的自动扩展组。在使用 AWS EKS（或者只是在 AWS EC2 上使用 Kubernetes，尽管这需要额外的配置）时，可以配置 AWS 自动扩展组中的工作节点组，根据节点的 CPU 或内存使用情况进行扩展或缩减。当这些节点由云提供商添加和初始化时，CCM 节点控制器将确保集群对于云提供商呈现的每个节点都有一个节点资源。

接下来，让我们转向路由控制器。

### CCM 路由控制器

CCM 路由控制器负责以支持 Kubernetes 集群的方式配置云提供商的网络设置。这可能包括分配 IP 和在节点之间设置路由。服务控制器也处理网络 - 但是外部方面。

### CCM 服务控制器

CCM 服务控制器提供了在公共云提供商上运行 Kubernetes 的“魔力”。我们在*第五章*中审查的一个方面是`LoadBalancer`服务，*服务和入口 - 与外部世界通信*，例如，在配置了 AWS CCM 的集群上，类型为`LoadBalancer`的服务将自动配置匹配的 AWS 负载均衡器资源，为您提供了一种在集群中公开服务的简单方法，而无需处理`NodePort`设置甚至 Ingress。

现在我们了解了 CCM 提供的内容，我们可以进一步探讨一下在公共云上运行 Kubernetes 时经常使用的一些其他云提供商扩展。首先，让我们看看`external-dns`。

## 使用 Kubernetes 的 external-dns

`external-dns`库是一个官方支持的 Kubernetes 插件，允许集群配置外部 DNS 提供程序以自动化方式为服务和 Ingress 提供 DNS 解析。`external-dns`插件支持广泛的云提供商，如 AWS 和 Azure，以及其他 DNS 服务，如 Cloudflare。

重要说明

要安装`external-dns`，您可以在[`github.com/kubernetes-sigs/external-dns`](https://github.com/kubernetes-sigs/external-dns)上查看官方 GitHub 存储库。

一旦在您的集群上实施了`external-dns`，就可以简单地以自动化的方式创建新的 DNS 记录。要测试`external-dns`与服务的配合，我们只需要在 Kubernetes 中创建一个带有适当注释的服务。

让我们看看这是什么样子：

service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: my-service-with-dns
  annotations:
    external-dns.alpha.kubernetes.io/hostname: myapp.mydomain.com
spec:
  type: LoadBalancer
  ports:
  - port: 80
    name: http
    targetPort: 80
  selector:
    app: my-app
```

正如您所看到的，我们只需要为`external-dns`控制器添加一个注释，以便检查要在 DNS 中创建的域记录。当然，域和托管区必须可以被您的`external-dns`控制器访问 - 例如，在 AWS Route 53 或 Azure DNS 上。请查看`external-dns` GitHub 存储库上的具体文档。

一旦服务启动运行，`external-dns`将获取注释并创建一个新的 DNS 记录。这种模式非常适合多租户或每个版本部署，因为像 Helm 图表这样的东西可以使用变量来根据应用程序的部署版本或分支来更改域 - 例如，`v1.myapp.mydomain.com`。

对于 Ingress，这甚至更容易 - 您只需要在 Ingress 记录中指定一个主机，就像这样：

ingress.yaml

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: my-domain-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx".
spec:
  rules:
  - host: myapp.mydomain.com
    http:
      paths:
      - backend:
          serviceName: my-app-service
          servicePort: 80
```

此主机值将自动创建一个 DNS 记录，指向 Ingress 正在使用的任何方法 - 例如，在 AWS 上的负载均衡器。

接下来，让我们谈谈**cluster-autoscaler**库的工作原理。

## 使用 cluster-autoscaler 插件

与`external-dns`类似，`cluster-autoscaler`是 Kubernetes 的一个官方支持的附加组件，支持一些主要的云提供商具有特定功能。 `cluster-autoscaler`的目的是触发集群中节点数量的扩展。它通过控制云提供商自己的扩展资源（例如 AWS 自动缩放组）来执行此过程。

集群自动缩放器将在任何单个 Pod 由于节点上的资源限制而无法调度时执行向上缩放操作，但仅当现有节点大小（例如，在 AWS 中为`t3.medium`大小的节点）可以允许 Pod 被调度时才会执行。

类似地，集群自动缩放器将在任何节点可以在不会对其他节点造成内存或 CPU 压力的情况下清空 Pod 时执行向下缩放操作。

要安装`cluster-autoscaler`，只需按照您的云提供商的正确说明，为集群类型和预期的`cluster-autoscaler`版本进行操作。例如，EKS 上的 AWS`cluster-autoscaler`的安装说明可在[`aws.amazon.com/premiumsupport/knowledge-center/eks-cluster-autoscaler-setup/`](https://aws.amazon.com/premiumsupport/knowledge-center/eks-cluster-autoscaler-setup/)找到。

接下来，让我们看看如何通过检查 Kubernetes 生态系统来找到开源和闭源的扩展。

# 与生态系统集成

Kubernetes（以及更一般地说，云原生）生态系统是庞大的，包括数百个流行的开源软件库，以及成千上万个新兴的软件库。这可能很难导航，因为每个月都会有新的技术需要审查，而收购、合并和公司倒闭可能会将您最喜欢的开源库变成一个未维护的混乱。

幸运的是，这个生态系统中有一些结构，了解它是值得的，以帮助导航云原生开源选项的匮乏。这其中的第一个重要结构组件是**云原生计算基金会**或**CNCF**。

## 介绍云原生计算基金会

CNCF 是 Linux 基金会的一个子基金会，它是一个主持开源项目并协调不断变化的公司列表的非营利实体，这些公司为和使用开源软件做出贡献。

CNCF 几乎完全是为了引导 Kubernetes 项目的未来而成立的。它是在 Kubernetes 1.0 发布时宣布的，并且此后已经发展到涵盖了云原生空间中的数百个项目 - 从 Prometheus 到 Envoy 到 Helm，以及更多。

了解 CNCF 组成项目的最佳方法是查看 CNCF Cloud Native Landscape，网址为[`landscape.cncf.io/`](https://landscape.cncf.io/)。

如果你对你在 Kubernetes 或云原生中遇到的问题感兴趣，CNCF Landscape 是一个很好的起点。对于每个类别（监控、日志记录、无服务器、服务网格等），都有几个开源选项供您选择。

当前云原生技术生态系统的优势和劣势。有大量的选择可用，这使得正确的路径通常不明确，但也意味着你可能会找到一个接近你确切需求的解决方案。

CNCF 还经营着一个官方的 Kubernetes 论坛，可以从 Kubernetes 官方网站[kubernetes.io](http://kubernetes.io)加入。Kubernetes 论坛的网址是[`discuss.kubernetes.io/`](https://discuss.kubernetes.io/)。

最后，值得一提的是*KubeCon*/*CloudNativeCon*，这是由 CNCF 主办的一个大型会议，涵盖了 Kubernetes 本身和许多生态项目等主题。*KubeCon*每年都在扩大规模，2019 年*KubeCon* *North America*有近 12,000 名与会者。

# 总结

在本章中，我们学习了如何扩展 Kubernetes。首先，我们讨论了 CRDs - 它们是什么，一些相关的用例，以及如何在集群中实现它们。接下来，我们回顾了 Kubernetes 中操作员的概念，并讨论了如何使用操作员或自定义控制器来赋予 CRD 生命。

然后，我们讨论了针对 Kubernetes 的特定于云供应商的扩展，包括`cloud-controller-manager`、`external-dns`和`cluster-autoscaler`。最后，我们介绍了大型云原生开源生态系统以及发现适合你使用情况的项目的一些好方法。

本章中使用的技能将帮助您扩展 Kubernetes 集群，以便与您的云提供商以及您自己的自定义功能进行接口。

在下一章中，我们将讨论作为应用于 Kubernetes 的两种新兴架构模式 - 无服务器和服务网格。

# 问题

1.  什么是 CRD 的服务版本和存储版本之间的区别？

1.  自定义控制器或操作员控制循环的三个典型部分是什么？

1.  `cluster-autoscaler`如何与现有的云提供商扩展解决方案（如 AWS 自动扩展组）交互？

# 进一步阅读

+   CNCF 景观：[`landscape.cncf.io/`](https://landscape.cncf.io/)

+   官方 Kubernetes 论坛：[`discuss.kubernetes.io/`](https://discuss.kubernetes.io/)

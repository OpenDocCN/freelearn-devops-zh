# 第三章：在 Kubernetes 上运行应用程序容器

本章包含了 Kubernetes 提供的最小的乐高积木块——Pod 的全面概述。其中包括 PodSpec YAML 格式和可能的配置的解释，以及 Kubernetes 如何处理和调度 Pod 的简要讨论。Pod 是在 Kubernetes 上运行应用程序的最基本方式，并且在所有高阶应用程序控制器中使用。

在本章中，我们将涵盖以下主题：

+   什么是 Pod？

+   命名空间

+   Pod 的生命周期

+   Pod 资源规范

+   Pod 调度

# 技术要求

为了运行本章详细介绍的命令，您需要一台支持`kubectl`命令行工具的计算机，以及一个可用的 Kubernetes 集群。请参见*第一章*，*与 Kubernetes 通信*，了解快速启动和运行 Kubernetes 的几种方法，以及如何安装`kubectl`工具的说明。

本章中使用的代码可以在书的 GitHub 存储库中找到以下链接：

[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter3`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter3)

# 什么是 Pod？

Pod 是 Kubernetes 中最简单的计算资源。它指定一个或多个容器由 Kubernetes 调度程序在节点上启动和运行。Pod 有许多潜在的配置和扩展，但仍然是在 Kubernetes 上运行应用程序的最基本方式。

重要说明

单独一个 Pod 并不是在 Kubernetes 上运行应用程序的很好的方式。Pod 应该被视为一次性的东西，以便充分利用像 Kubernetes 这样的容器编排器的真正能力。这意味着将容器（因此也是 Pod）视为牲畜，而不是宠物。为了真正利用容器和 Kubernetes，应用程序应该在自愈、可扩展的组中运行。Pod 是这些组的构建块，我们将在后面的章节中讨论如何以这种方式配置应用程序。

# 实现 Pod

Pod 是使用 Linux 隔离原则（如组和命名空间）实现的，并且通常可以被视为逻辑主机。Pod 运行一个或多个容器（可以基于 Docker、CRI-O 或其他运行时），这些容器可以以与 VM 上的不同进程通信的方式相互通信。

为了使两个不同 Pod 中的容器进行通信，它们需要通过 IP 访问另一个 Pod（和容器）。默认情况下，只有运行在同一个 Pod 上的容器才能使用更低级别的通信方法，尽管可以配置不同的 Pod 以使它们能够通过主机 IPC 相互通信。

## Pod 范例

在最基本的层面上，有两种类型的 Pods：

+   单容器 Pods

+   多容器 Pods

通常最好的做法是每个 Pod 包含一个单独的容器。这种方法允许您分别扩展应用程序的不同部分，并且在创建一个可以启动和运行而不出现问题的 Pod 时通常会保持简单。

另一方面，多容器 Pods 更复杂，但在各种情况下都可能很有用：

+   如果您的应用程序有多个部分运行在不同的容器中，但彼此之间紧密耦合，您可以将它们都运行在同一个 Pod 中，以使通信和文件系统访问无缝。

+   在实施*侧车*模式时，实用程序容器被注入到主应用程序旁边，用于处理日志记录、度量、网络或高级功能，比如服务网格（更多信息请参阅*第十四章*，*服务网格和无服务器*）。

下图显示了一个常见的侧车实现：

![图 3.1 - 常见的侧边栏实现](img/B14790_03_001.jpg)

图 3.1 - 常见的侧边栏实现

在这个例子中，我们有一个只有两个容器的 Pod：我们的应用容器运行一个 Web 服务器，一个日志应用程序从我们的服务器 Pod 中拉取日志并将其转发到我们的日志基础设施。这是侧车模式非常适用的一个例子，尽管许多日志收集器在节点级别工作，而不是在 Pod 级别，所以这并不是在 Kubernetes 中从我们的应用容器收集日志的通用方式。

## Pod 网络

正如我们刚才提到的，Pods 有自己的 IP 地址，可以用于 Pod 间通信。每个 Pod 都有一个 IP 地址和端口，如果有多个容器运行在一个 Pod 中，这些端口是共享的。

在 Pod 内部，正如我们之前提到的，容器可以在不调用封装 Pod 的 IP 的情况下进行通信 - 相反，它们可以简单地使用 localhost。这是因为 Pod 内的容器共享网络命名空间 - 本质上，它们通过相同的*bridge*进行通信，这是使用虚拟网络接口实现的。

## Pod 存储

Kubernetes 中的存储是一个独立的大主题，我们将在*第七章*中深入讨论它 - 但现在，您可以将 Pod 存储视为附加到 Pod 的持久或非持久卷。非持久卷可以被 Pod 用于存储数据或文件，具体取决于类型，但它们在 Pod 关闭时会被删除。持久类型的卷将在 Pod 关闭后保留，并且甚至可以用于在多个 Pod 或应用程序之间共享数据。

在我们继续讨论 Pod 之前，我们将花一点时间讨论命名空间。由于我们在处理 Pod 时将使用`kubectl`命令，了解命名空间如何与 Kubernetes 和`kubectl`相关联非常重要，因为这可能是一个重要的“坑”。

## 命名空间

在*第一章*的*与 Kubernetes 通信*部分，我们简要讨论了命名空间，但在这里我们将重申并扩展它们的目的。命名空间是一种在集群中逻辑上分隔不同区域的方式。一个常见的用例是每个环境一个命名空间 - 一个用于开发，一个用于暂存，一个用于生产 - 所有这些都存在于同一个集群中。

正如我们在*授权*部分中提到的，可以按命名空间指定用户权限 - 例如，允许用户向`dev`命名空间部署新应用程序和资源，但不允许向生产环境部署。

在运行的集群中，您可以通过运行`kubectl get namespaces`或`kubectl get ns`来查看存在哪些命名空间，这应该会产生以下输出：

```
NAME          STATUS    AGE
default       Active    1d
kube-system   Active    1d
kube-public   Active    1d
```

通过以下命令可以创建一个命名空间：`kubectl create namespace staging`，或者使用以下 YAML 资源规范运行`kubectl apply -f /path/to/file.yaml`：

Staging-ns.yaml

```
apiVersion: v1
kind: Namespace
metadata:
  name: staging
```

如您所见，`Namespace`规范非常简单。让我们继续讨论更复杂的内容 - PodSpec 本身。

## Pod 生命周期

要快速查看集群中正在运行的 Pods，您可以运行`kubectl get pods`或`kubectl get pods --all-namespaces`来分别获取当前命名空间中的 Pods（由您的`kubectl`上下文定义，如果未指定，则为默认命名空间）或所有命名空间中的 Pods。

`kubectl get pods`的输出如下：

```
NAME     READY   STATUS    RESTARTS   AGE
my-pod   1/1     Running   0          9s
```

正如您所看到的，Pod 具有一个`STATUS`值，告诉我们 Pod 当前处于哪种状态。

Pod 状态的值如下：

+   **运行**：在`运行`状态下，Pod 已成功启动其容器，没有任何问题。如果 Pod 只有一个容器，并且处于`运行`状态，那么容器尚未完成或退出其进程。它也可能正在重新启动，您可以通过检查`READY`列来判断。例如，如果`READY`值为`0/1`，这意味着 Pod 中的容器当前未通过健康检查。这可能是由于各种原因：容器可能仍在启动，数据库连接可能无法正常工作，或者一些重要配置可能会阻止应用程序进程启动。

+   **成功**：如果您的 Pod 容器设置为运行可以完成或退出的命令（不是长时间运行的命令，例如启动 Web 服务器），则如果这些容器已完成其进程命令，Pod 将显示`成功`状态。

+   **挂起**：`挂起`状态表示 Pod 中至少有一个容器正在等待其镜像。这可能是因为容器镜像仍在从外部存储库获取，或者因为 Pod 本身正在等待被`kube-scheduler`调度。

+   未知：`未知`状态表示 Kubernetes 无法确定 Pod 实际处于什么状态。这通常意味着 Pod 所在的节点遇到某种错误。可能是磁盘空间不足，与集群的其余部分断开连接，或者遇到其他问题。

+   **失败**：在`失败`状态下，Pod 中的一个或多个容器以失败状态终止。此外，Pod 中的其他容器必须以成功或失败的方式终止。这可能是由于集群删除 Pods 或容器应用程序内部的某些东西破坏了进程而发生的各种原因。

## 理解 Pod 资源规范

由于 Pod 资源规范是我们真正深入研究的第一个资源规范，我们将花时间详细介绍 YAML 文件的各个部分以及它们如何配合。

让我们从一个完全规范的 Pod 文件开始，然后我们可以分解和审查它：

Simple-pod.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: myApp
  namespace: dev
  labels:
    environment: dev
  annotations:
    customid1: 998123hjhsad 
spec:
  containers:
  - name: my-app-container
    image: busybox
```

这个 Pod YAML 文件比我们在第一章中看到的要复杂一些。它公开了一些新的 Pod 功能，我们将很快进行审查。

### API 版本

让我们从第 1 行开始：`apiVersion`。正如我们在*第一章*中提到的，*与 Kubernetes 通信*，`apiVersion` 告诉 Kubernetes 在创建和配置资源时应查看哪个 API 版本。Pod 在 Kubernetes 中已经存在很长时间，因此 PodSpec 已经固定为 API 版本`v1`。其他资源类型可能除了版本名称外还包含组名 - 例如，在 Kubernetes 中，CronJob 资源使用`batch/v1beta1` `apiVersion`，而 Job 资源使用`batch/v1` `apiVersion`。在这两种情况下，`batch` 对应于 API 组名。

### Kind

`kind` 值对应于 Kubernetes 中资源类型的实际名称。在这种情况下，我们正在尝试规范一个 Pod，所以这就是我们放置的内容。`kind` 值始终采用驼峰命名法，例如 `Pod`、`ConfigMap`、`CronJob` 等。

重要说明

要获取完整的`kind`值列表，请查看官方 Kubernetes 文档[`kubernetes.io/docs/home/`](https://kubernetes.io/docs/home/)。新的 Kubernetes `kind` 值会在新版本中添加，因此本书中审查的内容可能不是详尽的列表。

### 元数据

元数据是一个顶级键，可以在其下具有几个不同的值。首先，`name` 是资源名称，这是资源通过`kubectl`显示的名称，也是在`etcd`中存储的名称。`namespace` 对应于资源应该被创建在的命名空间。如果在 YAML 规范中未指定命名空间，则资源将被创建在`default`命名空间中 - 除非在`apply`或`create`命令中指定了命名空间。

接下来，`labels` 是用于向资源添加元数据的键值对。`labels` 与其他元数据相比是特殊的，因为它们默认用于 Kubernetes 本机`selectors`中，以过滤和选择资源 - 但它们也可以用于自定义功能。

最后，`metadata`块可以承载多个`annotations`，就像`labels`一样，可以被控制器和自定义 Kubernetes 功能用来提供额外的配置和特定功能的数据。在这个 PodSpec 中，我们在元数据中指定了几个注释：

pod-with-annotations.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: myApp
  namespace: dev
  labels:
    environment: dev
  annotations:
    customid1: 998123hjhsad
    customid2: 1239808908sd 
spec:
  containers:
  - name: my-app-container
    image: busybox
```

通常，最好使用`labels`来进行 Kubernetes 特定功能和选择器的配置，同时使用`annotations`来添加数据或扩展功能 - 这只是一种惯例。

### 规范

`spec`是包含特定于资源的配置的顶级键。在这种情况下，由于我们的`kind`值是`Pod`，我们将添加一些特定于我们的 Pod 的配置。所有进一步的键将缩进在这个`spec`键下，并将代表我们的 Pod 配置。

### 容器

`containers`键期望一个或多个容器的列表，这些容器将在一个 Pod 中运行。每个容器规范将公开其自己的配置值，这些配置值缩进在资源 YAML 中的容器列表项下。我们将在这里审查一些这些配置，但是要获取完整列表，请查看 Kubernetes 文档（[`kubernetes.io/docs/home/`](https://kubernetes.io/docs/home/)）。

### 名称

在容器规范中，`name`指的是容器在 Pod 中的名称。容器名称可以用于使用`kubectl logs`命令特别访问特定容器的日志，但这部分我们以后再说。现在，请确保为 Pod 中的每个容器选择一个清晰的名称，以便在调试时更容易处理事情。

### 图像

对于每个容器，`image`用于指定应在 Pod 中启动的 Docker（或其他运行时）镜像的名称。默认情况下，图像将从配置的存储库中拉取，这是公共 Docker Hub，但也可以是私有存储库。

就是这样 - 这就是你需要指定一个 Pod 并在 Kubernetes 中运行它的全部内容。从`Pod`部分开始的一切都属于*额外配置*的范畴。

### Pod 资源规范

Pod 可以配置为具有分配给它们的特定内存和计算量。这可以防止特别耗费资源的应用程序影响集群性能，也可以帮助防止内存泄漏。可以指定两种可能的资源 - `cpu`和`memory`。对于每个资源，有两种不同类型的规范，`Requests`和`Limits`，总共有四个可能的资源规范键。

内存请求和限制可以使用任何典型的内存数字后缀进行配置，或者其二的幂等价 - 例如，50 Mi（mebibytes），50 MB（megabytes）或 1 Gi（gibibytes）。

CPU 请求和限制可以通过使用`m`来配置，它对应于 1 毫 CPU，或者只是使用一个小数。因此，`200m`等同于`0.2`，相当于 20%或五分之一的逻辑 CPU。无论核心数量如何，这个数量都将是相同的计算能力。1 CPU 等于 AWS 中的虚拟核心或 GCP 中的核心。让我们看看这些资源请求和限制在我们的 YAML 文件中是什么样子的：

pod-with-resource-limits.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: myApp
spec:
  containers:
  - name: my-app-container
    image: mydockername
    resources:
      requests:
        memory: "50Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"
        cpu: "500m"
```

在这个`Pod`中，我们有一个运行 Docker 镜像的容器，该容器在`cpu`和`memory`上都指定了请求和限制。在这种情况下，我们的容器镜像名称`mydockername`是一个占位符 - 但是如果您想在此示例中测试 Pod 资源限制，可以使用 busybox 镜像。

### 容器启动命令

当容器在 Kubernetes Pod 中启动时，它将运行容器的默认启动脚本 - 例如，在 Docker 容器规范中指定的脚本。为了使用不同的命令或附加参数覆盖此功能，您可以提供`command`和`args`键。让我们看一个配置了`start`命令和一些参数的容器：

pod-with-start-command.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: myApp
spec:
  containers:
  - name: my-app-container
    image: mydockername
    command: ["run"]
    args: ["--flag", "T", "--run-type", "static"]
```

正如您所看到的，我们指定了一个命令以及作为字符串数组的参数列表，用逗号分隔空格。

### 初始化容器

`init`容器是 Pod 中特殊的容器，在正常 Pod 容器启动之前启动、运行和关闭。

`init`容器可用于许多不同的用例，例如在应用程序启动之前初始化文件，或者确保其他应用程序或服务在启动 Pod 之前正在运行。

如果指定了多个`init`容器，它们将按顺序运行，直到所有`init`容器都关闭。因此，`init`容器必须运行一个完成并具有端点的脚本。如果您的`init`容器脚本或应用程序继续运行，Pod 中的正常容器将不会启动。

在下面的 Pod 中，`init`容器正在运行一个循环，通过`nslookup`检查我们的`config-service`是否存在。一旦它看到`config-service`已经启动，脚本就会结束，从而触发我们的`my-app`应用容器启动：

pod-with-init-container.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: myApp
spec:
  containers:
  - name: my-app
    image: mydockername
    command: ["run"]
  initContainers:
  - name: init-before
    image: busybox
    command: ['sh', '-c', 'until nslookup config-service; do echo config-service not up; sleep 2; done;']
```

重要提示

当`init`容器失败时，Kubernetes 将自动重新启动 Pod，类似于通常的 Pod 启动功能。可以通过在 Pod 级别更改`restartPolicy`来更改此功能。

这是一个显示 Kubernetes 中典型 Pod 启动流程的图表：

![图 3.2-初始化容器流程图](img/B14790_03_002.jpg)

图 3.2-初始化容器流程图

如果一个 Pod 有多个`initContainer`，它们将按顺序被调用。这对于那些设置了必须按顺序执行的模块化步骤的`initContainers`非常有价值。以下 YAML 显示了这一点：

pod-with-multiple-init-containers.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: myApp
spec:
  containers:
  - name: my-app
    image: mydockername
    command: ["run"]
  initContainers:
  - name: init-step-1
    image: step1-image
    command: ['start-command']
  - name: init-step-2
    image: step2-image
    command: ['start-command']
```

例如，在这个`Pod` YAML 文件中，`step-1 init`容器需要在调用`init-step-2`之前成功，两者都需要在启动`my-app`容器之前显示成功。

### 在 Kubernetes 中引入不同类型的探针

为了知道容器（因此也是 Pod）何时失败，Kubernetes 需要知道如何测试容器是否正常工作。我们通过定义`probes`来实现这一点，Kubernetes 可以在指定的间隔运行这些`probes`，以确定容器是否正常工作。

Kubernetes 允许我们配置三种类型的探针-就绪、存活和启动。

### 就绪探针

首先，就绪探针可用于确定容器是否准备好执行诸如通过 HTTP 接受流量之类的功能。这些探针在应用程序运行的初始阶段非常有帮助，例如，当应用程序可能仍在获取配置，尚未准备好接受连接时。

让我们看一下配置了就绪探针的 Pod 是什么样子。接下来是一个附有就绪探针的 PodSpec：

pod-with-readiness-probe.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: myApp
spec:
  containers:
  - name: my-app
    image: mydockername
    command: ["run"]
    ports:
    - containerPort: 8080
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/thisfileshouldexist.txt
      initialDelaySeconds: 5
      periodSeconds: 5
```

首先，正如您所看到的，探针是针对每个容器而不是每个 Pod 定义的。Kubernetes 将对每个容器运行所有探针，并使用它来确定 Pod 的总体健康状况。

### 存活探针

存活探针可用于确定应用程序是否因某种原因（例如，由于内存错误）而失败。对于长时间运行的应用程序容器，存活探针可以作为一种方法，帮助 Kubernetes 回收旧的和损坏的 Pod，以便创建新的 Pod。虽然探针本身不会导致容器重新启动，但其他 Kubernetes 资源和控制器将检查探针状态，并在必要时使用它来重新启动 Pod。以下是附有存活探针定义的 PodSpec：

pod-with-liveness-probe.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: myApp
spec:
  containers:
  - name: my-app
    image: mydockername
    command: ["run"]
    ports:
    - containerPort: 8080
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/thisfileshouldexist.txt
      initialDelaySeconds: 5
      failureThreshold: 3
      periodSeconds: 5
```

正如您所看到的，我们的活跃性探针与就绪性探针以相同的方式指定，只是增加了`failureThreshold`。

`failureThreshold`值将决定 Kubernetes 在采取行动之前尝试探测的次数。对于活跃性探针，一旦超过`failureThreshold`，Kubernetes 将重新启动 Pod。对于就绪性探针，Kubernetes 将简单地标记 Pod 为`Not Ready`。此阈值的默认值为`3`，但可以更改为大于或等于`1`的任何值。

在这种情况下，我们使用了`exec`机制进行探测。我们将很快审查可用的各种探测机制。

### 启动探针

最后，启动探针是一种特殊类型的探针，它只会在容器启动时运行一次。一些（通常是较旧的）应用程序在容器中启动需要很长时间，因此在容器第一次启动时提供一些额外的余地，可以防止活跃性或就绪性探针失败并导致重新启动。以下是配置了启动探针的 Pod 示例：

pod-with-startup-probe.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: myApp
spec:
  containers:
  - name: my-app
    image: mydockername
    command: ["run"]
    ports:
    - containerPort: 8080
    startupProbe:
      exec:
        command:
        - cat
        - /tmp/thisfileshouldexist.txt
      initialDelaySeconds: 5
      successThreshold: 2
      periodSeconds: 5
```

启动探针提供的好处不仅仅是延长活跃性或就绪性探针之间的时间 - 它们允许 Kubernetes 在启动后处理问题时保持快速反应，并且（更重要的是）防止启动缓慢的应用程序不断重新启动。如果您的应用程序需要多秒甚至一两分钟才能启动，您将更容易实现启动探针。

`successThreshold`就像它的名字一样，是`failureThreshold`的对立面。它指定在容器标记为`Ready`之前需要连续多少次成功。对于在启动时可能会上下波动然后稳定下来的应用程序（如一些自我集群应用程序），更改此值可能很有用。默认值为`1`，对于活跃性探针，唯一可能的值是`1`，但我们可以更改就绪性和启动探针的值。

### 探测机制配置

有多种机制可以指定这三种探针中的任何一种：`exec`、`httpGet`和`tcpSocket`。

`exec`方法允许您指定在容器内运行的命令。成功执行的命令将导致探测通过，而失败的命令将导致探测失败。到目前为止，我们配置的所有探针都使用了`exec`方法，因此配置应该是不言自明的。如果所选命令（以逗号分隔的列表形式指定的任何参数）失败，探测将失败。

`httpGet`方法允许您为探针指定容器上的 URL，该 URL 将受到 HTTP `GET`请求的访问。如果 HTTP 请求返回的代码在`200`到`400`之间，它将导致探测成功。任何其他 HTTP 代码将导致失败。

`httpGet`的配置如下：

pod-with-get-probe.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: myApp
spec:
  containers:
  - name: my-app
    image: mydockername
    command: ["run"]
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthcheck
        port: 8001
        httpHeaders:
        - name: My-Header
          value: My-Header-Value
        initialDelaySeconds: 3
        periodSeconds: 3
```

最后，`tcpSocket`方法将尝试在容器上打开指定的套接字，并使用结果来决定成功或失败。`tcpSocket`配置如下：

pod-with-tcp-probe.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: myApp
spec:
  containers:
  - name: my-app
    image: mydockername
    command: ["run"]
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
```

正如您所看到的，这种类型的探针接收一个端口，每次检查发生时都会对其进行 ping 测试。

### 常见的 Pod 转换

Kubernetes 中的失败 Pod 往往在不同状态之间转换。对于初次使用者来说，这可能会令人生畏，因此将我们之前列出的 Pod 状态与探针功能相互作用进行分解是很有价值的。再次强调一下，这是我们的状态：

+   `Running`

+   `Succeeded`

+   `Pending`

+   `Unknown`

+   `Failed`

一个常见的流程是运行`kubectl get pods -w`（`-w`标志会在命令中添加一个监视器），然后查看有问题的 Pod 在`Pending`和`Failed`之间的转换。通常情况下，发生的是 Pod（及其容器）正在启动和拉取镜像 - 这是`Pending`状态，因为健康检查尚未开始。

一旦初始探测超时（正如我们在前一节中看到的那样，这是可配置的），第一个探测失败。这可能会持续几秒甚至几分钟，具体取决于失败阈值的高低，状态仍然固定在`Pending`。

最后，我们的失败阈值达到，我们的 Pod 状态转换为`Failed`。在这一点上，有两种情况可能发生，决定纯粹基于 PodSpec 上的`RestartPolicy`，它可以是`Always`、`Never`或`OnFailure`。如果一个 Pod 失败并且`restartPolicy`是`Never`，那么 Pod 将保持在失败状态。如果是其他两个选项之一，Pod 将自动重新启动，并返回到`Pending`，这是我们永无止境的转换循环的根本原因。

举个不同的例子，您可能会看到 Pod 永远停留在`Pending`状态。这可能是由于 Pod 无法被调度到任何节点。这可能是由于资源请求约束（我们将在本书的后面深入讨论，*第八章*，*Pod 放置控件*），或其他问题，比如节点无法访问。

最后，对于`Unknown`，通常 Pod 被调度的节点由于某种原因无法访问 - 例如，节点可能已关闭，或者通过网络无法访问。

### Pod 调度

Pod 调度的复杂性以及 Kubernetes 让您影响和控制它的方式将保存在我们的*第八章*中，*Pod 放置控件* - 但现在我们将回顾基础知识。

在决定在哪里调度一个 Pod 时，Kubernetes 考虑了许多因素，但最重要的是考虑（当不深入研究 Kubernetes 让我们使用的更复杂的控件时）Pod 优先级、节点可用性和资源可用性。

Kubernetes 调度程序操作一个不断的控制循环，监视集群中未绑定（未调度）的 Pod。如果找到一个或多个未绑定的 Pod，调度程序将使用 Pod 优先级来决定首先调度哪一个。

一旦调度程序决定要调度一个 Pod，它将执行几轮和类型的检查，以找到调度 Pod 的节点的局部最优解。后面的检查由细粒度的调度控件决定，我们将在*第八章*中详细介绍*Pod 放置控件*。现在我们只关心前几轮的检查。

首先，Kubernetes 检查当前时刻哪些节点可以被调度。节点可能无法正常工作，或者遇到其他问题，这将阻止新的 Pod 被调度。

其次，Kubernetes 通过检查哪些节点与 PodSpec 中规定的最小资源需求匹配来过滤可调度的节点。

在没有其他放置控制的情况下，调度器将做出决定并将新的 Pod 分配给一个节点。当该节点上的 `kubelet` 看到有一个新的 Pod 分配给它时，该 Pod 将被启动。

# 摘要

在本章中，我们了解到 Pod 是我们在 Kubernetes 中使用的最基本的构建块。对 Pod 及其所有微妙之处有深入的理解非常重要，因为在 Kubernetes 上的所有计算都使用 Pod 作为构建块。现在可能很明显了，但 Pod 是非常小的、独立的东西，不太牢固。在 Kubernetes 上以单个 Pod 运行应用程序而没有控制器是一个糟糕的决定，你的 Pod 出现任何问题都会导致停机时间。

在下一章中，我们将看到如何通过使用 Pod 控制器同时运行应用程序的多个副本来防止这种情况发生。

# 问题

1.  你如何使用命名空间来分隔应用程序环境？

1.  Pod 状态被列为 `Unknown` 的可能原因是什么？

1.  限制 Pod 内存资源的原因是什么？

1.  如果在 Kubernetes 上运行的应用程序经常在失败的探测重新启动 Pod 之前无法及时启动，你应该调整哪种探测类型？就绪性、存活性还是启动？

# 进一步阅读

+   官方 Kubernetes 文档：[`kubernetes.io/docs/home/`](https://kubernetes.io/docs/home/)

+   《Kubernetes The Hard Way》：[`github.com/kelseyhightower/kubernetes-the-hard-way`](https://github.com/kelseyhightower/kubernetes-the-hard-way)

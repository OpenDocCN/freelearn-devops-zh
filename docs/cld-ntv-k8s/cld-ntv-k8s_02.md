# *第一章*：与 Kubernetes 通信

本章包含容器编排的解释，包括其优势、用例和流行的实现。我们还将简要回顾 Kubernetes，包括架构组件的布局，以及对授权、身份验证和与 Kubernetes 的一般通信的入门。到本章结束时，您将知道如何对 Kubernetes API 进行身份验证和通信。

在本章中，我们将涵盖以下主题：

+   容器编排入门

+   Kubernetes 的架构

+   在 Kubernetes 上的身份验证和授权

+   使用 kubectl 和 YAML 文件

# 技术要求

为了运行本章详细介绍的命令，您需要一台运行 Linux、macOS 或 Windows 的计算机。本章将教您如何安装`kubectl`命令行工具，您将在以后的所有章节中使用它。

本章中使用的代码可以在书的 GitHub 存储库中找到，链接如下：

[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter1`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter1)

# 介绍容器编排

谈论 Kubernetes 时，不能不介绍其目的。Kubernetes 是一个容器编排框架，让我们在本书的背景下回顾一下这意味着什么。

## 什么是容器编排？

容器编排是在云端和数据中心运行现代应用程序的流行模式。通过使用容器-预配置的应用程序单元和捆绑的依赖项-作为基础，开发人员可以并行运行许多应用程序实例。

## 容器编排的好处

容器编排提供了许多好处，但我们将重点介绍主要的好处。首先，它允许开发人员轻松构建**高可用性**应用程序。通过运行多个应用程序实例，容器编排系统可以配置成自动替换任何失败的应用程序实例为新的实例。

这可以通过在物理数据中心中分散应用程序的多个实例来扩展到云端，因此如果一个数据中心崩溃，应用程序的其他实例将保持运行，并防止停机。

其次，容器编排允许高度**可扩展**的应用程序。由于可以轻松创建和销毁应用程序的新实例，编排工具可以自动扩展以满足需求。在云环境或数据中心环境中，可以向编排工具添加新的**虚拟机**（**VMs**）或物理机，以提供更大的计算资源池。在云环境中，这个过程可以完全自动化，实现完全无需人工干预的扩展，无论是在微观还是宏观层面。

## 流行的编排工具

生态系统中有几种非常流行的容器编排工具：

+   **Docker Swarm**：Docker Swarm 是由 Docker 容器引擎团队创建的。与 Kubernetes 相比，它更容易设置和运行，但相对灵活性较差。

+   **Apache Mesos**：Apache Mesos 是一个较低级别的编排工具，可以管理数据中心和云环境中的计算、内存和存储。默认情况下，Mesos 不管理容器，但是 Marathon - 一个在 Mesos 之上运行的框架 - 是一个完全成熟的容器编排工具。甚至可以在 Mesos 之上运行 Kubernetes。

+   **Kubernetes**：截至 2020 年，容器编排工作大部分集中在 Kubernetes（koo-bur-net-ees）周围，通常缩写为 k8s。Kubernetes 是一个开源容器编排工具，最初由谷歌创建，借鉴了谷歌多年来在内部编排工具 Borg 和 Omega 的经验。自 Kubernetes 成为开源项目以来，它已经成为企业环境中运行和编排容器的事实标准。其中一些原因包括 Kubernetes 是一个成熟的产品，拥有一个非常庞大的开源社区。它比 Mesos 更容易操作，比 Docker Swarm 更灵活。

从这个比较中最重要的一点是，尽管容器编排有多个相关选项，而且在某些方面确实更好，但 Kubernetes 已经成为事实标准。有了这个认识，让我们来看看 Kubernetes 是如何工作的。

# Kubernetes 的架构

Kubernetes 是一个可以在云虚拟机上运行的编排工具，也可以在数据中心的虚拟机或裸机服务器上运行。一般来说，Kubernetes 在一组节点上运行，每个节点可以是虚拟机或物理机。

## Kubernetes 节点类型

Kubernetes 节点可以是许多不同的东西-从虚拟机到裸金属主机再到树莓派。Kubernetes 节点分为两个不同的类别：首先是主节点，运行 Kubernetes 控制平面应用程序；其次是工作节点，运行您部署到 Kubernetes 上的应用程序。

一般来说，为了实现高可用性，Kubernetes 的生产部署应该至少有三个主节点和三个工作节点，尽管大多数大型部署的工作节点比主节点多得多。

## Kubernetes 控制平面

Kubernetes 控制平面是一套运行在主节点上的应用程序和服务。有几个高度专业化的服务在发挥作用，构成了 Kubernetes 功能的核心。它们如下：

+   kube-apiserver：这是 Kubernetes API 服务器。该应用程序处理发送到 Kubernetes 的指令。

+   kube-scheduler：这是 Kubernetes 调度程序。该组件处理决定将工作负载放置在哪些节点上的工作，这可能变得非常复杂。

+   kube-controller-manager：这是 Kubernetes 控制器管理器。该组件提供了一个高级控制循环，确保集群的期望配置和运行在其上的应用程序得到实施。

+   etcd：这是一个包含集群配置的分布式键值存储。

一般来说，所有这些组件都采用系统服务的形式，在每个主节点上运行。如果您想完全手动引导集群，可以手动启动它们，但是通过使用集群创建库或云提供商管理的服务，例如**弹性 Kubernetes 服务（EKS）**，在生产环境中通常会自动完成这些操作。

## Kubernetes API 服务器

Kubernetes API 服务器是一个接受 HTTPS 请求的组件，通常在端口`443`上。它提供证书，可以是自签名的，以及身份验证和授权机制，我们将在本章后面介绍。

当对 Kubernetes API 服务器进行配置请求时，它将检查`etcd`中的当前集群配置，并在必要时进行更改。

Kubernetes API 通常是一个 RESTful API，每个 Kubernetes 资源类型都有端点，以及在查询路径中传递的 API 版本；例如，`/api/v1`。

为了扩展 Kubernetes（参见[*第十三章*]（B14790_13_Final_PG_ePub.xhtml#_idTextAnchor289），*使用 CRD 扩展 Kubernetes*），API 还具有一组基于 API 组的动态端点，可以向自定义资源公开相同的 RESTful API 功能。

## Kubernetes 调度程序

Kubernetes 调度程序决定工作负载的实例应该在哪里运行。默认情况下，此决定受工作负载资源要求和节点状态的影响。您还可以通过 Kubernetes 中可配置的放置控件来影响调度程序（参见[*第八章*]（B14790_08_Final_PG_ePub.xhtml#_idTextAnchor186），*Pod 放置控件*）。这些控件可以作用于节点标签，其他 Pod 已经在节点上运行的情况，以及许多其他可能性。

## Kubernetes 控制器管理器

Kubernetes 控制器管理器是运行多个控制器的组件。控制器运行控制循环，确保集群的实际状态与配置中存储的状态匹配。默认情况下，这些包括以下内容：

+   节点控制器，确保节点正常运行

+   复制控制器，确保每个工作负载被适当地扩展

+   端点控制器，处理每个工作负载的通信和路由配置（参见[*第五章*]（B14790_05_Final_PG_ePub.xhtml#_idTextAnchor127）*，服务和入口 - 与外部世界通信*）

+   服务帐户和令牌控制器，处理 API 访问令牌和默认帐户的创建

## etcd

etcd 是一个分布式键值存储，以高可用的方式存储集群的配置。每个主节点上都运行一个`etcd`副本，并使用 Raft 一致性算法，确保在允许对键或值进行任何更改之前保持法定人数。

## Kubernetes 工作节点

每个 Kubernetes 工作节点都包含允许其与控制平面通信和处理网络的组件。

首先是**kubelet**，它确保容器根据集群配置在节点上运行。其次，**kube-proxy**为在每个节点上运行的工作负载提供网络代理层。最后，**容器运行时**用于在每个节点上运行工作负载。

## kubelet

kubelet 是在每个节点上运行的代理程序（包括主节点，尽管在该上下文中它具有不同的配置）。它的主要目的是接收 PodSpecs 的列表（稍后会详细介绍），并确保它们所规定的容器在节点上运行。kubelet 通过几种不同的可能机制获取这些 PodSpecs，但主要方式是通过查询 Kubernetes API 服务器。另外，kubelet 可以通过文件路径启动，它将监视 PodSpecs 的列表，监视 HTTP 端点，或者在其自己的 HTTP 端点上接收请求。

## kube-proxy

kube-proxy 是在每个节点上运行的网络代理。它的主要目的是对其节点上运行的工作负载进行 TCP、UDP 和 SCTP 转发（通过流或轮询）。kube-proxy 支持 Kubernetes 的`Service`构造，我们将在*第五章**，服务和入口 - 与外部世界通信*中讨论。

## 容器运行时

容器运行时在每个节点上运行，它实际上运行您的工作负载。Kubernetes 支持 CRI-O、Docker、containerd、rktlet 和任何有效的**容器运行时接口**（**CRI**）运行时。从 Kubernetes v1.14 开始，RuntimeClass 功能已从 alpha 版移至 beta 版，并允许特定于工作负载的运行时选择。

## 插件

除了核心集群组件外，典型的 Kubernetes 安装包括插件，这些是提供集群功能的附加组件。

例如，**容器网络接口**（**CNI**）插件，如`Calico`、`Flannel`或`Weave`，提供符合 Kubernetes 网络要求的覆盖网络功能。

另一方面，CoreDNS 是一个流行的插件，用于集群内的 DNS 和服务发现。还有一些工具，比如 Kubernetes Dashboard，它提供了一个 GUI，用于查看和与您的集群进行交互。

到目前为止，您应该对 Kubernetes 的主要组件有一个高层次的了解。接下来，我们将回顾用户如何与 Kubernetes 交互以控制这些组件。

# Kubernetes 上的身份验证和授权

命名空间是 Kubernetes 中一个非常重要的概念，因为它们可以影响 API 访问以及授权，我们现在将介绍它们。

## 命名空间

Kubernetes 中的命名空间是一种构造，允许您在集群中对 Kubernetes 资源进行分组。它们是一种分离的方法，有许多可能的用途。例如，您可以在集群中为每个环境（开发、暂存和生产）创建一个命名空间。

默认情况下，Kubernetes 将创建默认命名空间、`kube-system`命名空间和`kube-public`命名空间。在未指定命名空间的情况下创建的资源将在默认命名空间中创建。`kube-system`包含集群服务，如`etcd`、调度程序以及 Kubernetes 本身创建的任何资源，而不是用户创建的资源。`kube-public`默认情况下可被所有用户读取，并且可用于公共资源。

## 用户

Kubernetes 中有两种类型的用户 - 常规用户和服务帐户。

通常由集群外的服务管理常规用户，无论是私钥、用户名和密码，还是某种用户存储形式。但是，服务帐户由 Kubernetes 管理，并且受限于特定的命名空间。要创建服务帐户，Kubernetes API 可能会自动创建一个，或者可以通过调用 Kubernetes API 手动创建。

Kubernetes API 有三种可能的请求类型 - 与常规用户关联的请求，与服务帐户关联的请求和匿名请求。

## 认证方法

为了对请求进行身份验证，Kubernetes 提供了几种不同的选项：HTTP 基本身份验证、客户端证书、bearer 令牌和基于代理的身份验证。

要使用 HTTP 身份验证，请求者发送带有`Authorization`头的请求，其值为 bearer `"token value"`。

为了指定哪些令牌是有效的，可以在 API 服务器应用程序启动时使用`--token-auth-file=filename`参数提供一个 CSV 文件。一个新的测试功能（截至本书撰写时），称为*引导令牌*，允许在 API 服务器运行时动态交换和更改令牌，而无需重新启动它。

还可以通过`Authorization`令牌进行基本的用户名/密码身份验证，方法是使用头部值`Basic base64encoded(username:password)`。

## Kubernetes 的 TLS 和安全证书基础设施

为了使用客户端证书（X.509 证书），API 服务器必须使用`--client-ca-file=filename`参数启动。该文件需要包含一个或多个用于验证通过 API 请求传递的证书的**证书颁发机构**（**CAs**）。

除了**CA**之外，必须为每个用户创建一个**证书签名请求**（**CSR**）。在这一点上，可以包括用户`groups`，我们将在*授权*选项部分讨论。

例如，您可以使用以下内容：

```
openssl req -new -key myuser.pem -out myusercsr.pem -subj "/CN=myuser/0=dev/0=staging"
```

这将为名为`myuser`的用户创建一个 CSR，该用户属于名为`dev`和`staging`的组。

创建 CA 和 CSR 后，可以使用`openssl`、`easyrsa`、`cfssl`或任何证书生成工具创建实际的客户端和服务器证书。此时还可以创建用于 Kubernetes API 的 TLS 证书。

由于我们的目标是尽快让您开始在 Kubernetes 上运行工作负载，我们将不在本书中涉及各种可能的证书配置 - 但 Kubernetes 文档和文章* Kubernetes The Hard Way*都有一些关于从头开始设置集群的很棒的教程。在大多数生产环境中，您不会手动执行这些步骤。

## 授权选项

Kubernetes 提供了几种授权方法：节点、webhooks、RBAC 和 ABAC。在本书中，我们将重点关注 RBAC 和 ABAC，因为它们是用户授权中最常用的方法。如果您通过其他服务和/或自定义功能扩展了集群，则其他授权模式可能变得更加重要。

## RBAC

**RBAC**代表**基于角色的访问控制**，是一种常见的授权模式。在 Kubernetes 中，RBAC 的角色和用户使用四个 Kubernetes 资源来实现：`Role`、`ClusterRole`、`RoleBinding`和`ClusterRoleBinding`。要启用 RBAC 模式，API 服务器可以使用`--authorization-mode=RBAC`参数启动。

`Role`和`ClusterRole`资源指定了一组权限，但不会将这些权限分配给任何特定的用户。权限使用`resources`和`verbs`来指定。以下是一个指定`Role`的示例 YAML 文件。不要太担心 YAML 文件的前几行 - 我们很快就会涉及到这些内容。专注于`resources`和`verbs`行，以了解如何将操作应用于资源：

只读角色.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: read-only-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

`Role` 和 `ClusterRole` 之间唯一的区别是，`Role` 限定于特定的命名空间（在本例中是默认命名空间），而 `ClusterRole` 可以影响集群中该类型的所有资源的访问，以及集群范围的资源，如节点。

`RoleBinding` 和 `ClusterRoleBinding` 是将 `Role` 或 `ClusterRole` 与用户或用户列表关联的资源。以下文件表示一个 `RoleBinding` 资源，将我们的 `read-only-role` 与用户 `readonlyuser` 连接起来：

只读-rb.yaml

```
apiVersion: rbac.authorization.k8s.io/v1namespace.
kind: RoleBinding
metadata:
  name: read-only
  namespace: default
subjects:
- kind: User
  name: readonlyuser
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: read-only-role
  apiGroup: rbac.authorization.k8s.io
```

`subjects` 键包含要将角色与的所有实体的列表；在本例中是用户 `alex`。`roleRef` 包含要关联的角色的名称和类型（`Role` 或 `ClusterRole`）。

## ABAC

**ABAC** 代表 **基于属性的访问控制**。ABAC 使用 *策略* 而不是角色。API 服务器在 ABAC 模式下启动，使用一个称为授权策略文件的文件，其中包含一个名为策略对象的 JSON 对象列表。要启用 ABAC 模式，API 服务器可以使用 `--authorization-mode=ABAC` 和 `--authorization-policy-file=filename` 参数启动。

在策略文件中，每个策略对象包含有关单个策略的信息：首先，它对应的主体，可以是用户或组，其次，可以通过策略访问哪些资源。此外，可以包括一个布尔值 `readonly`，以限制策略仅限于 `list`、`get` 和 `watch` 操作。

与资源关联的第二种类型的策略与非资源请求类型相关联，例如对 `/version` 端点的调用。

当在 ABAC 模式下对 API 发出请求时，API 服务器将检查用户及其所属的任何组是否与策略文件中的列表匹配，并查看是否有任何策略与用户正在尝试访问的资源或端点匹配。匹配时，API 服务器将授权请求。

现在您应该对 Kubernetes API 如何处理身份验证和授权有了很好的理解。好消息是，虽然您可以直接访问 API，但 Kubernetes 提供了一个出色的命令行工具，可以简单地进行身份验证并发出 Kubernetes API 请求。

# 使用 kubectl 和 YAML

kubectl 是官方支持的命令行工具，用于访问 Kubernetes API。它可以安装在 Linux、macOS 或 Windows 上。

## 设置 kubectl 和 kubeconfig

要安装最新版本的 kubectl，可以使用[`kubernetes.io/docs/tasks/tools/install-kubectl/`](https://kubernetes.io/docs/tasks/tools/install-kubectl/)上的安装说明。

安装了 kubectl 之后，需要设置身份验证以与一个或多个集群进行身份验证。这是使用`kubeconfig`文件完成的，其外观如下：

示例-kubeconfig

```
apiVersion: v1
kind: Config
preferences: {}
clusters:
- cluster:
    certificate-authority: fake-ca-file
    server: https://1.2.3.4
  name: development
users:
- name: alex
  user:
    password: mypass
    username: alex
contexts:
- context:
    cluster: development
    namespace: frontend
    user: developer
  name: development
```

该文件以 YAML 编写，与我们即将介绍的其他 Kubernetes 资源规范非常相似 - 只是该文件仅驻留在您的本地计算机上。

`Kubeconfig` YAML 文件有三个部分：`clusters`，`users`和`contexts`：

+   `clusters`部分是您可以通过 kubectl 访问的集群列表，包括 CA 文件名和服务器 API 端点。

+   `users`部分列出了您可以授权的用户，包括用于身份验证的任何用户证书或用户名/密码组合。

+   最后，`contexts`部分列出了集群、命名空间和用户的组合，这些组合形成一个上下文。使用`kubectl config use-context`命令，您可以轻松地在上下文之间切换，从而实现集群、用户和命名空间组合的轻松切换。

## 命令式与声明式命令

与 Kubernetes API 交互有两种范式：命令式和声明式。命令式命令允许您向 Kubernetes“指示要做什么” - 也就是说，“启动两个 Ubuntu 副本”，“将此应用程序扩展到五个副本”等。

另一方面，声明式命令允许您编写一个文件，其中包含应在集群上运行的规范，并且 Kubernetes API 确保配置与集群配置匹配，并在必要时进行更新。

尽管命令式命令允许您快速开始使用 Kubernetes，但最好在运行生产工作负载或任何复杂工作负载时编写一些 YAML 并使用声明性配置。原因是这样做可以更容易地跟踪更改，例如通过 GitHub 存储库，或者向您的集群引入基于 Git 的持续集成/持续交付（CI/CD）。

一些基本的 kubectl 命令

kubectl 提供了许多方便的命令来检查集群的当前状态，查询资源并创建新资源。 kubectl 的结构使大多数命令可以以相同的方式访问资源。

首先，让我们学习如何查看集群中的 Kubernetes 资源。您可以使用`kubectl get resource_type`来执行此操作，其中`resource_type`是 Kubernetes 资源的完整名称，或者是一个更短的别名。别名（和`kubectl`命令）的完整列表可以在 kubectl 文档中找到：[`kubernetes.io/docs/reference/kubectl/overview`](https://kubernetes.io/docs/reference/kubectl/overview)。

我们已经了解了节点，所以让我们从那里开始。要查找集群中存在哪些节点，我们可以使用`kubectl get nodes`或别名`kubectl get no`。

kubectl 的`get`命令返回当前集群中的 Kubernetes 资源列表。我们可以使用任何 Kubernetes 资源类型运行此命令。要向列表添加附加信息，可以添加`wide`输出标志：`kubectl get nodes -o wide`。

列出资源是不够的，当然 - 我们需要能够查看特定资源的详细信息。为此，我们使用`describe`命令，它的工作方式类似于`get`，只是我们可以选择传递特定资源的名称。如果省略了最后一个参数，Kubernetes 将返回该类型所有资源的详细信息，这可能会导致终端中大量的滚动。

例如，`kubectl describe nodes`将返回集群中所有节点的详细信息，而`kubectl describe nodes node1`将返回名为`node1`的节点的描述。

您可能已经注意到，这些命令都是命令式风格的，这是有道理的，因为我们只是获取有关现有资源的信息，而不是创建新资源。要创建 Kubernetes 资源，我们可以使用以下命令：

+   `kubectl create -f /path/to/file.yaml`，这是一个命令式命令

+   `kubectl apply -f /path/to/file.yaml`，这是声明式的

这两个命令都需要一个文件路径，可以是 YAML 或 JSON 格式，或者您也可以使用 `stdin`。您还可以传递文件夹的路径，而不是文件的路径，这将创建或应用该文件夹中的所有 YAML 或 JSON 文件。`create` 是命令式的，因此它将创建一个新的资源，但如果您再次运行它并使用相同的文件，命令将失败，因为资源已经存在。`apply` 是声明性的，因此如果您第一次运行它，它将创建资源，而后续运行将使用任何更改更新 Kubernetes 中正在运行的资源。您可以使用 `--dry-run` 标志来查看 `create` 或 `apply` 命令的输出（即将创建的资源，或者如果存在错误的话）。

要以命令式方式更新现有资源，可以使用 `edit` 命令，如：`kubectl edit resource_type resource_name` – 就像我们的 `describe` 命令一样。这将打开默认的终端编辑器，并显示现有资源的 YAML，无论您是以命令式还是声明式方式创建的。您可以编辑并保存，这将触发 Kubernetes 中资源的自动更新。

要以声明性方式更新现有资源，可以编辑您用于首次创建资源的本地 YAML 资源文件，然后运行 `kubectl apply -f /path/to/file.yaml`。最好通过命令式命令 `kubectl delete resource_type resource_name` 来删除资源。

我们将在本节讨论的最后一个命令是 `kubectl cluster-info`，它将显示主要 Kubernetes 集群服务运行的 IP 地址。

## 编写 Kubernetes 资源 YAML 文件

用于与 Kubernetes API 声明性通信的格式包括 YAML 和 JSON。为了本书的目的，我们将坚持使用 YAML，因为它更清晰，占用页面空间更少。典型的 Kubernetes 资源 YAML 文件如下：

resource.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: ubuntu
    image: ubuntu:trusty
    command: ["echo"]
    args: ["Hello Readers"]
```

有效的 Kubernetes YAML 文件至少有四个顶级键。它们是 `apiVersion`、`kind`、`metadata` 和 `spec`。

`apiVersion`决定将使用哪个版本的 Kubernetes API 来创建资源。`kind`指定 YAML 文件引用的资源类型。`metadata`提供了一个位置来命名资源，以及添加注释和命名空间信息（稍后会详细介绍）。最后，`spec`键将包含 Kubernetes 创建资源所需的所有特定于资源的信息。

不要担心`kind`和`spec`，我们将在*第三章*中介绍`Pod`是什么，*在 Kubernetes 上运行应用容器*。

# 总结

在本章中，我们学习了容器编排背后的背景，Kubernetes 集群的架构概述，集群如何对 API 调用进行身份验证和授权，以及如何使用 kubectl 以命令和声明模式与 API 进行通信，kubectl 是 Kubernetes 的官方支持的命令行工具。

在下一章中，我们将学习几种启动测试集群的方法，并掌握到目前为止学到的 kubectl 命令。

# 问题

1.  什么是容器编排？

1.  Kubernetes 控制平面的组成部分是什么，它们的作用是什么？

1.  如何启动处于 ABAC 授权模式的 Kubernetes API 服务器？

1.  为什么对于生产 Kubernetes 集群来说拥有多个主节点很重要？

1.  `kubectl apply`和`kubectl create`之间有什么区别？

1.  如何使用`kubectl`在上下文之间切换？

1.  以声明方式创建 Kubernetes 资源然后以命令方式进行编辑的缺点是什么？

# 进一步阅读

+   官方 Kubernetes 文档：[`kubernetes.io/docs/home/`](https://kubernetes.io/docs/home/)

+   *Kubernetes The Hard Way*：[`github.com/kelseyhightower/kubernetes-the-hard-way`](https://github.com/kelseyhightower/kubernetes-the-hard-way)

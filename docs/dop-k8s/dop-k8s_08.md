# 第八章：集群管理

在之前的章节中，我们学习了 Kubernetes 中大部分基本的 DevOps 技能，从如何将应用程序容器化到通过持续部署将我们的容器化软件无缝部署到 Kubernetes。现在，是时候更深入地了解如何管理 Kubernetes 集群了。

在本章中，我们将学习：

+   如何利用命名空间设置管理边界

+   使用 kubeconfig 在多个集群之间切换

+   Kubernetes 身份验证

+   Kubernetes 授权

虽然 minikube 是一个相当简单的环境，但在本章中，我们将以**Google 容器引擎**（**GKE**）和 AWS 中的自托管集群作为示例，而不是 minikube。有关详细设置，请参阅第九章，*AWS 上的 Kubernetes*，以及第十章，*GCP 上的 Kubernetes*。

# Kubernetes 命名空间

Kubernetes 具有命名空间概念，将物理集群中的资源划分为多个虚拟集群。这样，不同的组可以共享同一个物理集群并实现隔离。每个命名空间提供：

+   一组名称范围；每个命名空间中的对象名称是唯一的

+   确保受信任身份验证的策略

+   设置资源配额以进行资源管理

命名空间非常适合同一公司中的不同团队或项目，因此不同的组可以拥有自己的虚拟集群，这些集群具有资源隔离但共享同一个物理集群。一个命名空间中的资源对其他命名空间是不可见的。可以为不同的命名空间设置不同的资源配额，并提供不同级别的 QoS。请注意，并非所有对象都在命名空间中，例如节点和持久卷，它们属于整个集群。

# 默认命名空间

默认情况下，Kubernetes 有三个命名空间：`default`，`kube-system`和`kube-public`。`default`命名空间包含未指定任何命名空间创建的对象，而`kube-system`包含由 Kubernetes 系统创建的对象，通常由系统组件使用，例如 Kubernetes 仪表板或 Kubernetes DNS。`kube-public`是在 1.6 中新引入的，旨在定位每个人都可以访问的资源。它现在主要关注公共 ConfigMap，如集群信息。

# 创建新的命名空间

让我们看看如何创建一个命名空间。命名空间也是 Kubernetes 对象。我们可以像其他对象一样指定种类为命名空间。下面是创建一个命名空间`project1`的示例：

[PRE0]

然后让我们尝试通过`project1`命名空间中的部署启动两个 nginx 容器：

[PRE1]

当我们通过`kubectl get pods`列出 pod 时，我们会在我们的集群中看不到任何内容。为什么？因为 Kubernetes 使用当前上下文来决定哪个命名空间是当前的。如果我们在上下文或`kubectl`命令行中不明确指定命名空间，则将使用`default`命名空间：

[PRE2]

您可以使用`--namespace <namespace_name>`，`--namespace=<namespace_name>`，`-n <namespace_name>`或`-n=<namespace_name>`来指定命令的命名空间。要列出跨命名空间的资源，请使用`--all-namespaces`参数。

另一种方法是将当前上下文更改为指向所需命名空间，而不是默认命名空间。

# 上下文

**上下文**是集群信息、用于身份验证的用户和命名空间的组合概念。例如，以下是我们在 GKE 中一个集群的上下文信息：

[PRE3]

我们可以使用`kubectl config current-context`命令查看当前上下文：

[PRE4]

要列出所有配置信息，包括上下文，您可以使用`kubectl config view`命令；要检查当前正在使用的上下文，使用`kubectl config get-contexts`命令。

# 创建上下文

下一步是创建上下文。与前面的示例一样，我们需要为上下文设置用户和集群名称。如果我们不指定这些，将设置为空值。创建上下文的命令是：

[PRE5]

在同一集群中可以创建多个上下文。以下是如何在我的 GKE 集群`gke_devops-with-kubernetes_us-central1-b_cluster`中为`project1`创建上下文的示例：

[PRE6]

# 切换当前上下文

然后我们可以通过`use-context`子命令切换上下文：

[PRE7]

上下文切换后，我们通过`kubectl`调用的每个命令都在`project1`上下文下。我们不需要明确指定命名空间来查看我们的 pod：

[PRE8]

# 资源配额

在 Kubernetes 中，默认情况下，pod 是无限制的资源。然后运行的 pod 可能会使用集群中的所有计算或存储资源。ResourceQuota 是一个资源对象，允许我们限制命名空间可以使用的资源消耗。通过设置资源限制，我们可以减少嘈杂的邻居症状。为`project1`工作的团队不会耗尽物理集群中的所有资源。

然后我们可以确保其他项目中工作的团队在共享同一物理集群时的服务质量。Kubernetes 1.7 支持三种资源配额。每种类型包括不同的资源名称（[`kubernetes.io/docs/concepts/policy/resource-quotas`](https://kubernetes.io/docs/concepts/policy/resource-quotas)）。

+   计算资源配额（CPU，内存）

+   存储资源配额（请求的存储、持久卷索赔）

+   对象计数配额（pod、RCs、ConfigMaps、services、LoadBalancers）

创建的资源不会受到新创建的资源配额的影响。如果资源创建请求超过指定的 ResourceQuota，资源将无法启动。

# 为命名空间创建资源配额

现在，让我们学习`ResourceQuota`的语法。以下是一个例子：

[PRE9]

模板与其他对象一样，只是这种类型变成了`ResourceQuota`。我们指定的配额适用于处于成功或失败状态的 pod（即非终端状态）。支持几种资源约束。在前面的例子中，我们演示了如何设置计算 ResourceQuota、存储 ResourceQuota 和对象 CountQuota。随时，我们仍然可以使用`kubectl`命令来检查我们设置的配额：`kubectl describe resourcequota <resource_quota_name>`。

现在让我们通过命令`kubectl edit deployment nginx`修改我们现有的 nginx 部署，将副本从`2`更改为`4`并保存。现在让我们列出状态。

[PRE10]

它指示一些 pod 在创建时失败。如果我们检查相应的 ReplicaSet，我们可以找出原因：

[PRE11]

由于我们已经在内存和 CPU 上指定了请求限制，Kubernetes 不知道新期望的三个 pod 的默认请求限制。我们可以看到原来的两个 pod 仍在运行，因为资源配额不适用于现有资源。然后我们使用`kubectl edit deployment nginx`来修改容器规范如下：

![](img/00116.jpeg)

在这里，我们在 pod 规范中指定了 CPU 和内存的请求和限制。这表明 pod 不能超过指定的配额，否则将无法启动：

[PRE12]

可用的 pod 变成了四个，而不是两个，但仍然不等于我们期望的四个。出了什么问题？如果我们退一步检查我们的资源配额，我们会发现我们已经使用了所有的 pod 配额。由于部署默认使用滚动更新部署机制，它将需要大于四的 pod 数量，这正是我们之前设置的对象限制：

[PRE13]

通过`kubectl edit resourcequota project1-resource-quota`命令将 pod 配额从`4`修改为`8`后，部署有足够的资源来启动 pod。一旦`Used`配额超过`Hard`配额，请求将被资源配额准入控制器拒绝，否则，资源配额使用将被更新以确保足够的资源分配。

由于资源配额不会影响已创建的资源，有时我们可能需要调整失败的资源，比如删除一个 RS 的空更改集或者扩展和缩小部署，以便让 Kubernetes 创建新的 pod 或 RS，这将吸收最新的配额限制。

# 请求具有默认计算资源限制的 pod

我们还可以为命名空间指定默认的资源请求和限制。如果在创建 pod 时不指定请求和限制，将使用默认设置。关键是使用`LimitRange`资源对象。`LimitRange`对象包含一组`defaultRequest`（请求）和`default`（限制）。

LimitRange 由 LimitRanger 准入控制器插件控制。如果启动自托管解决方案，请确保启用它。有关更多信息，请查看本章的准入控制器部分。

下面是一个示例，我们将`cpu.request`设置为`250m`，`limits`设置为`500m`，`memory.request`设置为`256Mi`，`limits`设置为`512Mi`：

[PRE14]

当我们在此命名空间内启动 pod 时，即使在 ResourceQuota 中设置了总限制，我们也不需要随时指定`cpu`和`memory`请求和`limits`。

CPU 的单位是核心，这是一个绝对数量。它可以是 AWS vCPU，GCP 核心或者装备了超线程处理器的机器上的超线程。内存的单位是字节。Kubernetes 使用字母或二的幂的等价物。例如，256M 可以写成 256,000,000，256 M 或 244 Mi。

此外，我们可以在 LimitRange 中为 pod 设置最小和最大的 CPU 和内存值。它与默认值的作用不同。默认值仅在 pod 规范不包含任何请求和限制时使用。最小和最大约束用于验证 pod 是否请求了太多的资源。语法是`spec.limits[].min`和`spec.limits[].max`。如果请求超过了最小和最大值，服务器将抛出 forbidden 错误。

[PRE15]

Pod 的服务质量：Kubernetes 中的 pod 有三个 QoS 类别：Guaranteed、Burstable 和 BestEffort。它与我们上面学到的命名空间和资源管理概念密切相关。我们还在第四章中学习了 QoS，*使用存储和资源*。请参考第四章中的最后一节*使用存储和资源*进行复习。

# 删除一个命名空间

与其他资源一样，删除一个命名空间是`kubectl delete namespace <namespace_name>`。请注意，如果删除一个命名空间，与该命名空间关联的所有资源都将被清除。

# Kubeconfig

Kubeconfig 是一个文件，您可以使用它来通过切换上下文来切换多个集群。我们可以使用`kubectl config view`来查看设置。以下是`kubeconfig`文件中 minikube 集群的示例。

[PRE16]

就像我们之前学到的一样。我们可以使用`kubectl config use-context`来切换要操作的集群。我们还可以使用`kubectl config --kubeconfig=<config file name>`来指定要使用的`kubeconfig`文件。只有指定的文件将被使用。我们还可以通过环境变量`$KUBECONFIG`指定`kubeconfig`文件。这样，配置文件可以被合并。例如，以下命令将合并`kubeconfig-file1`和`kubeconfig-file2`：

[PRE17]

您可能会发现我们之前没有进行任何特定的设置。那么`kubectl config view`的输出来自哪里呢？默认情况下，它存在于`$HOME/.kube/config`下。如果没有设置前面的任何一个，将加载此文件。

# 服务账户

与普通用户不同，**服务账户**是由 pod 内的进程用来联系 Kubernetes API 服务器的。默认情况下，Kubernetes 集群为不同的目的创建不同的服务账户。在 GKE 中，已经创建了大量的服务账户：

[PRE18]

Kubernetes 将在每个命名空间中创建一个默认的服务账户，如果在创建 pod 时未指定服务账户，则将使用该默认服务账户。让我们看看默认服务账户在我们的`project1`命名空间中是如何工作的：

[PRE19]

我们可以看到，服务账户基本上是使用可挂载的密钥作为令牌。让我们深入了解令牌中包含的内容：

[PRE20]

密钥将自动挂载到目录`/var/run/secrets/kubernetes.io/serviceaccount`。当 pod 访问 API 服务器时，API 服务器将检查证书和令牌进行认证。服务账户的概念将在接下来的部分中与我们同在。

# 认证和授权

从 DevOps 的角度来看，认证和授权非常重要。认证验证用户并检查用户是否真的是他们所代表的身份。另一方面，授权检查用户拥有哪些权限级别。Kubernetes 支持不同的认证和授权模块。

以下是一个示例，展示了当 Kubernetes API 服务器收到请求时如何处理访问控制。

![](img/00117.jpeg)API 服务器中的访问控制

当请求发送到 API 服务器时，首先，它通过使用 API 服务器中的**证书颁发机构**（**CA**）验证客户端的证书来建立 TLS 连接。API 服务器中的 CA 通常位于`/etc/kubernetes/`，客户端的证书通常位于`$HOME/.kube/config`。握手完成后，进入身份验证阶段。在 Kubernetes 中，身份验证模块是基于链的。我们可以使用多个身份验证和授权模块。当请求到来时，Kubernetes 将依次尝试所有的身份验证器，直到成功。如果请求在所有身份验证模块上失败，将被拒绝为 HTTP 401 未经授权。否则，其中一个身份验证器验证用户的身份并对请求进行身份验证。然后 Kubernetes 授权模块将发挥作用。它将验证用户是否有权限执行他们请求的操作，通过一组策略。授权模块也是基于链的。它将不断尝试每个模块，直到成功。如果请求在所有模块上失败，将得到 HTTP 403 禁止的响应。准入控制是 API 服务器中一组可配置的插件，用于确定请求是否被允许或拒绝。在这个阶段，如果请求没有通过其中一个插件，那么请求将立即被拒绝。

# 身份验证

默认情况下，服务账户是基于令牌的。当您创建一个服务账户或一个带有默认服务账户的命名空间时，Kubernetes 会创建令牌并将其存储为一个由 base64 编码的秘密，并将该秘密作为卷挂载到 pod 中。然后 pod 内的进程有能力与集群通信。另一方面，用户账户代表一个普通用户，可能使用`kubectl`直接操作资源。

# 服务账户身份验证

当我们创建一个服务账户时，Kubernetes 服务账户准入控制器插件会自动创建一个签名的令牌。

在第七章，*持续交付*中，在我们演示了如何部署`my-app`的示例中，我们创建了一个名为`cd`的命名空间，并且我们使用了脚本`get-sa-token.sh`（[`github.com/DevOps-with-Kubernetes/examples/blob/master/chapter7/get-sa-token.sh`](https://github.com/DevOps-with-Kubernetes/examples/blob/master/chapter7/get-sa-token.sh)）来为我们导出令牌。然后我们通过`kubectl config set-credentials <user> --token=$TOKEN`命令创建了一个名为`mysa`的用户：

[PRE21]

接下来，我们将上下文设置为与用户和命名空间绑定：

[PRE22]

最后，我们将把我们的上下文`myctxt`设置为默认上下文：

[PRE23]

当服务账户发送请求时，API 服务器将验证令牌，以检查请求者是否有资格以及它所声称的身份是否属实。

# 用户账户认证

有几种用户账户认证的实现方式。从客户端证书、持有者令牌、静态文件到 OpenID 连接令牌。您可以选择多种身份验证链。在这里，我们将演示客户端证书的工作原理。

在第七章，*持续交付*中，我们学习了如何为服务账户导出证书和令牌。现在，让我们学习如何为用户做这件事。假设我们仍然在`project1`命名空间中，并且我们想为我们的新 DevOps 成员琳达创建一个用户，她将帮助我们为`my-app`进行部署。

首先，我们将通过 OpenSSL（[`www.openssl.org`](https://www.openssl.org)）生成一个私钥：

[PRE24]

接下来，我们将为琳达创建一个证书签名请求（`.csr`）：

[PRE25]

现在，`linda.key`和`linda.csr`应该位于当前文件夹中。为了批准签名请求，我们需要找到我们 Kubernetes 集群的 CA。

在 minikube 中，它位于`~/.minikube/`。对于其他自托管解决方案，通常位于`/etc/kubernetes/`下。如果您使用 kops 部署集群，则位置位于`/srv/kubernetes`下，您可以在`/etc/kubernetes/manifests/kube-apiserver.manifest`文件中找到路径。

假设我们在当前文件夹下有`ca.crt`和`ca.key`，我们可以通过我们的签名请求生成证书。使用`-days`参数，我们可以定义过期日期：

[PRE26]

在我们的集群中有证书签名后，我们可以在集群中设置一个用户。

[PRE27]

记住上下文的概念：它是集群信息、用于认证的用户和命名空间的组合。现在，我们将在`kubeconfig`中设置一个上下文条目。请记住从以下示例中替换您的集群名称、命名空间和用户：

[PRE28]

现在，琳达应该没有任何权限：

[PRE29]

琳达现在通过了认证阶段，而 Kubernetes 知道她是琳达。但是，为了让琳达有权限进行部署，我们需要在授权模块中设置策略。

# 授权

Kubernetes 支持多个授权模块。在撰写本文时，它支持：

+   ABAC

+   RBAC

+   节点授权

+   Webhook

+   自定义模块

**基于属性的访问控制**（**ABAC**）是在**基于角色的访问控制**（**RBAC**）引入之前的主要授权模式。节点授权被 kubelet 用于向 API 服务器发出请求。Kubernetes 支持 webhook 授权模式，以与外部 RESTful 服务建立 HTTP 回调。每当面临授权决定时，它都会进行 POST。另一种常见的方式是按照预定义的授权接口实现自己的内部模块。有关更多实现信息，请参阅[`kubernetes.io/docs/admin/authorization/#custom-modules`](https://kubernetes.io/docs/admin/authorization/#custom-modules)。在本节中，我们将更详细地描述 ABAC 和 RBAC。

# 基于属性的访问控制（ABAC）

ABAC 允许管理员将一组用户授权策略定义为每行一个 JSON 格式的文件。ABAC 模式的主要缺点是策略文件在启动 API 服务器时必须存在。文件中的任何更改都需要使用`--authorization-policy-file=<policy_file_name>`命令重新启动 API 服务器。自 Kubernetes 1.6 以来引入了另一种授权方法 RBAC，它更灵活，不需要重新启动 API 服务器。RBAC 现在已成为最常见的授权模式。

以下是 ABAC 工作原理的示例。策略文件的格式是每行一个 JSON 对象。策略的配置文件类似于我们的其他配置文件。只是在规范中有不同的语法。ABAC 有四个主要属性：

| **属性类型** | **支持的值** |
| --- | --- |
| 主题匹配 | 用户，组 |
| 资源匹配 | `apiGroup`，命名空间和资源 |
| 非资源匹配 | 用于非资源类型请求，如`/version`，`/apis`，`/cluster` |
| 只读 | true 或 false |

以下是一些示例：

[PRE30]

在前面的例子中，我们有一个名为 admin 的用户，可以访问所有内容。另一个名为`linda`的用户只能在命名空间`project1`中读取部署和副本集。

# 基于角色的访问控制（RBAC）

RBAC 在 Kubernetes 1.6 中处于 beta 阶段，默认情况下是启用的。在 RBAC 中，管理员创建了几个`Roles`或`ClusterRoles`，这些角色定义了细粒度的权限，指定了一组资源和操作（动词），角色可以访问和操作这些资源。之后，管理员通过`RoleBinding`或`ClusterRoleBindings`向用户授予`Role`权限。

如果你正在运行 minikube，在执行`minikube start`时添加`--extra-config=apiserver.Authorization.Mode=RBAC`。如果你通过 kops 在 AWS 上运行自托管集群，则在启动集群时添加`--authorization=rbac`。Kops 会将 API 服务器作为一个 pod 启动；使用`kops edit cluster`命令可以修改容器的规范。

# 角色和集群角色

在 Kubernetes 中，`Role`绑定在命名空间内，而`ClusterRole`是全局的。以下是一个`Role`的示例，可以对部署、副本集和 pod 资源执行所有操作，包括`get`、`watch`、`list`、`create`、`update`、`delete`、`patch`。

[PRE31]

在我们写这本书的时候，`apiVersion`仍然是`v1beta1`。如果 API 版本发生变化，Kubernetes 会抛出错误并提醒您进行更改。在`apiGroups`中，空字符串表示核心 API 组。API 组是 RESTful API 调用的一部分。核心表示原始 API 调用路径，例如`/api/v1`。新的 REST 路径中包含组名和 API 版本，例如`/apis/$GROUP_NAME/$VERSION`；要查找您想要使用的 API 组，请查看[`kubernetes.io/docs/reference`](https://kubernetes.io/docs/reference)中的 API 参考。在资源下，您可以添加您想要授予访问权限的资源，在动词下列出了此角色可以执行的操作数组。让我们来看一个更高级的`ClusterRoles`示例，我们在上一章中使用了持续交付角色：

[PRE32]

`ClusterRole`是集群范围的。一些资源不属于任何命名空间，比如节点，只能由`ClusterRole`控制。它可以访问的命名空间取决于它关联的`ClusterRoleBinding`中的`namespaces`字段。我们可以看到，我们授予了该角色读取和写入 Deployments、ReplicaSets 和 ingresses 的权限，它们分别属于 extensions 和 apps 组。在核心 API 组中，我们只授予了对命名空间和事件的访问权限，以及对其他资源（如 pods 和 services）的所有权限。

# RoleBinding 和 ClusterRoleBinding

`RoleBinding`用于将`Role`或`ClusterRole`绑定到一组用户或服务账户。如果`ClusterRole`与`RoleBinding`绑定而不是`ClusterRoleBinding`，它将只被授予`RoleBinding`指定的命名空间内的权限。以下是`RoleBinding`规范的示例：

[PRE33]

在这个例子中，我们通过`roleRef`将`Role`与用户绑定。Kubernetes 支持不同类型的`roleRef`；我们可以在这里将`Role`的类型替换为`ClusterRole`：

[PRE34]

然后`cd-role`只能访问`project1`命名空间中的资源。

另一方面，`ClusterRoleBinding`用于在所有命名空间中授予权限。让我们回顾一下我们在第七章中所做的事情，*持续交付*。我们首先创建了一个名为`cd-agent`的服务账户，然后创建了一个名为`cd-role`的`ClusterRole`。最后，我们为`cd-agent`和`cd-role`创建了一个`ClusterRoleBinding`。然后我们使用`cd-agent`代表我们进行部署：

[PRE35]

`cd-agent`通过`ClusterRoleBinding`与`ClusterRole`绑定，因此它可以跨命名空间拥有`cd-role`中指定的权限。由于服务账户是在命名空间中创建的，我们需要指定其完整名称，包括命名空间：

[PRE36]

让我们通过`8-5-2_role.yml`和`8-5-2_rolebinding_user.yml`启动`Role`和`RoleBinding`：

[PRE37]

现在，我们不再被禁止了：

[PRE38]

如果 Linda 想要列出命名空间，允许吗？：

[PRE39]

答案是否定的，因为 Linda 没有被授予列出命名空间的权限。

# 准入控制

准入控制发生在 Kubernetes 处理请求之前，经过身份验证和授权之后。在启动 API 服务器时，通过添加`--admission-control`参数来启用它。如果集群版本>=1.6.0，Kubernetes 建议在集群中使用以下插件。

[PRE40]

以下介绍了这些插件的用法，以及为什么我们需要它们。有关支持的准入控制插件的更多最新信息，请访问官方文档[`kubernetes.io/docs/admin/admission-controllers`](https://kubernetes.io/docs/admin/admission-controllers)。

# 命名空间生命周期

正如我们之前所了解的，当命名空间被删除时，该命名空间中的所有对象也将被驱逐。此插件确保在终止或不存在的命名空间中无法发出新的对象创建请求。它还防止了 Kubernetes 本机命名空间的删除。

# 限制范围

此插件确保`LimitRange`可以正常工作。使用`LimitRange`，我们可以在命名空间中设置默认请求和限制，在启动未指定请求和限制的 Pod 时将使用这些设置。

# 服务帐户

如果使用服务帐户对象，则必须添加服务帐户插件。有关服务帐户的更多信息，请再次查看本章中的服务帐户部分。

# PersistentVolumeLabel

`PersistentVolumeLabel`根据底层云提供商提供的标签，为新创建的 PV 添加标签。此准入控制器已从 1.8 版本中弃用。

# 默认存储类

如果在持久卷索赔中未设置`StorageClass`，此插件确保默认存储类可以按预期工作。不同的云提供商使用不同的供应工具来利用`DefaultStorageClass`（例如 GKE 使用 Google Cloud 持久磁盘）。请确保您已启用此功能。

# 资源配额

就像`LimitRange`一样，如果您正在使用`ResourceQuota`对象来管理不同级别的 QoS，则必须启用此插件。资源配额应始终放在准入控制插件列表的末尾。正如我们在资源配额部分提到的，如果使用的配额少于硬配额，资源配额使用将被更新，以确保集群具有足够的资源来接受请求。将其放在准入控制器列表的末尾可以防止请求在被以下控制器拒绝之前过早增加配额使用。

# 默认容忍秒

在介绍此插件之前，我们必须了解**污点**和**容忍**是什么。

# 污点和容忍

污点和忍受度用于阻止一组 Pod 在某些节点上调度运行。污点应用于节点，而忍受度则指定给 Pod。污点的值可以是`NoSchedule`或`NoExecute`。如果在运行一个带有污点的节点上的 Pod 时没有匹配的忍受度，那么这些 Pod 将被驱逐。

假设我们有两个节点：

[PRE41]

现在通过`kubectl run nginx --image=nginx:1.12.0 --replicas=1 --port=80`命令运行一个 nginx Pod。

该 Pod 正在第一个节点`ip-172-20-56-91.ec2.internal`上运行：

[PRE42]

通过 Pod 描述，我们可以看到有两个默认的忍受度附加到 Pod 上。这意味着如果节点尚未准备好或不可达，那么在 Pod 从节点中被驱逐之前等待 300 秒。这两个忍受度由 DefaultTolerationSeconds 准入控制器插件应用。我们稍后会谈论这个。接下来，我们将在第一个节点上设置一个 taint：

[PRE43]

由于我们将操作设置为`NoExecute`，并且`experimental=true`与我们的 Pod 上的任何忍受度不匹配，因此 Pod 将立即从节点中删除并重新调度。可以将多个 taints 应用于一个节点。Pod 必须匹配所有忍受度才能在该节点上运行。以下是一个可以通过的带污染节点的示例：

[PRE44]

除了`Equal`运算符，我们也可以使用`Exists`。在这种情况下，我们不需要指定值。只要键存在并且效果匹配，那么 Pod 就有资格在带污染的节点上运行。

`DefaultTolerationSeconds`插件用于设置那些没有设置任何忍受度的 Pod。然后将应用于`taints`的默认忍受度`notready:NoExecute`和`unreachable:NoExecute`，持续 300 秒。如果您不希望在集群中发生此行为，禁用此插件可能有效。

# PodNodeSelector

此插件用于将`node-selector`注释设置为命名空间。当启用插件时，使用以下格式通过`--admission-control-config-file`命令传递配置文件：

[PRE45]

然后`node-selector`注释将应用于命名空间。然后该命名空间上的 Pod 将在这些匹配的节点上运行。

# AlwaysAdmit

这总是允许所有请求，可能仅用于测试。

# AlwaysPullImages

拉取策略定义了 kubelet 拉取镜像时的行为。默认的拉取策略是`IfNotPresent`，也就是说，如果本地不存在镜像，它将拉取镜像。如果启用了这个插件，那么默认的拉取策略将变为`Always`，也就是说，总是拉取最新的镜像。这个插件还带来了另一个好处，如果你的集群被不同的团队共享。每当一个 pod 被调度，它都会拉取最新的镜像，无论本地是否存在该镜像。这样我们就可以确保 pod 创建请求始终通过对镜像的授权检查。

# AlwaysDeny

这总是拒绝所有请求。它只能用于测试。

# DenyEscalatingExec

这个插件拒绝任何`kubectl exec`和`kubectl attach`命令升级特权模式。具有特权模式的 pod 具有主机命名空间的访问权限，这可能会带来安全风险。

# 其他准入控制器插件

还有许多其他的准入控制器插件可以使用，比如 NodeRestriciton 来限制 kubelet 的权限，ImagePolicyWebhook 来建立一个控制对镜像访问的 webhook，SecurityContextDeny 来控制 pod 或容器的权限。请参考官方文档([`kubernetes.io/docs/admin/admission-controllers)`](https://kubernetes.io/docs/admin/admission-controllers/))以了解其他插件。

# 总结

在本章中，我们学习了命名空间和上下文是什么以及它们是如何工作的，如何通过设置上下文在物理集群和虚拟集群之间切换。然后我们了解了重要的对象——服务账户，它用于识别在 pod 内运行的进程。然后我们了解了如何在 Kubernetes 中控制访问流程。我们了解了认证和授权之间的区别，以及它们在 Kubernetes 中的工作方式。我们还学习了如何利用 RBAC 为用户提供细粒度的权限。最后，我们学习了一些准入控制器插件，它们是访问控制流程中的最后一道防线。

AWS 是公共 IaaS 提供商中最重要的参与者。在本章中，我们在自托管集群示例中经常使用它。在下一章第九章，*在 AWS 上使用 Kubernetes*，我们将最终学习如何在 AWS 上部署集群以及在使用 AWS 时的基本概念。

# *第八章*：RBAC 策略和审计

认证只是集群访问管理的第一步。一旦集群访问权限被授予，限制账户的操作是很重要的，这取决于账户是用于自动化系统还是用户。授权访问资源是保护集群免受意外问题和恶意行为者滥用的重要部分。

在本章中，我们将详细介绍 Kubernetes 如何通过其基于角色的访问控制（RBAC）模型授权访问。本章的第一部分将深入探讨 Kubernetes RBAC 的配置方式，可用的选项以及将理论映射到实际示例中。调试和故障排除 RBAC 策略将是第二部分的重点。

在本章中，我们将涵盖以下主题：

+   RBAC 简介

+   将企业身份映射到 Kubernetes 以授权访问资源

+   命名空间多租户

+   Kubernetes 审计

+   使用**audit2rbac**调试策略

# 技术要求

本章具有以下技术要求：

+   使用*第七章*的配置运行的 KinD 集群，*将身份验证集成到您的集群*

+   从*第六章*的 SAML2 实验室访问，*服务、负载均衡和外部 DNS*

您可以在以下 GitHub 存储库中访问本章的代码：[`github.com/PacktPublishing/Kubernetes-and-Docker-The-Complete-Guide`](https://github.com/PacktPublishing/Kubernetes-and-Docker-The-Complete-Guide)。

# RBAC 简介

在我们深入研究 RBAC 之前，让我们快速了解一下 Kubernetes 和访问控制的历史。

在 Kubernetes 1.6 之前，访问控制是基于基于属性的访问控制（ABAC）的。顾名思义，ABAC 通过将规则与属性进行比较来提供访问权限，而不是角色。分配的属性可以分配任何类型的数据，包括用户属性、对象、环境、位置等。

过去，要为 ABAC 配置 Kubernetes 集群，您必须在 API 服务器上设置两个值：

+   **--authorization-policy-file**

+   **--authorization-mode=ABAC**

**authorization-policy-file** 是 API 服务器上的本地文件。由于它是每个 API 服务器上的本地文件，对文件的任何更改都需要对主机进行特权访问，并且需要重启 API 服务器。可以想象，更新 ABAC 策略的过程变得困难，任何即时更改都将需要短暂的停机，因为 API 服务器正在重新启动。

从 Kubernetes 1.6 开始，**RBAC** 成为授权访问资源的首选方法。与**ABAC** 不同，**RBAC** 使用 Kubernetes 本机对象，更新可以在不重启 API 服务器的情况下反映出来。**RBAC** 也与不同的身份验证方法兼容。从这里开始，我们的重点将放在如何开发 RBAC 策略并将其应用到您的集群上。

# 什么是角色？

在 Kubernetes 中，角色是将权限绑定到可以描述和配置的对象的一种方式。角色有规则，这些规则是资源和动词的集合。往回推，我们有以下内容：

+   **动词**：可以在 API 上执行的操作，例如读取（**get**），写入（**create**，**update**，**patch**和**delete**），或列出和监视。

+   **资源**：要对其应用动词的 API 名称，例如**services**，**endpoints**等。也可以列出特定的子资源。可以命名特定资源以在对象上提供非常具体的权限。

角色并不说明谁可以在资源上执行动词，这由**RoleBindings**和**ClusterRoleBindings**处理。我们将在*RoleBindings 和 ClusterRoleBindings*部分了解更多信息。

重要提示

术语“角色”可能有多重含义，并且 RBAC 经常在其他上下文中使用。在企业世界中，“角色”一词通常与业务角色相关联，并用于传达该角色的权限，而不是特定的个人。例如，企业可能会为所有应付账款人员分配发放支票的权限，而不是为应付账款部门的每个成员创建特定的分配以发放支票的特定权限。当某人在不同角色之间移动时，他们会失去旧角色的权限，并获得新角色的权限。例如，从应付账款到应收账款的转移中，用户将失去支付的能力并获得接受付款的能力。通过将权限与角色而不是个人绑定，权限的更改会随着角色更改而自动发生，而不必为每个用户手动切换权限。这是术语 RBAC 的更“经典”用法。

每个规则将构建的资源由以下内容标识：

+   **apiGroups**：资源所属的组列表

+   **resources**：资源的对象类型的名称（可能还包括子资源）

+   **resourceNames**：要应用此规则的特定对象的可选列表

每个规则*必须*有一个**apiGroups**和**resources**的列表。**resourceNames**是可选的。

重要提示

如果您发现自己从命名空间内部授权对特定对象的访问权限，那么是时候重新思考您的授权策略了。Kubernetes 的租户边界是命名空间。除非有非常特定的原因，否则在 RBAC 角色中命名特定的 Kubernetes 对象是一种反模式，应该避免。当 RBAC 角色命名特定对象时，请考虑分割它们所在的命名空间以创建单独的命名空间。

一旦在规则中标识了资源，就可以指定动词。动词是可以在资源上执行的操作，从而在 Kubernetes 中提供对对象的访问权限。

如果对对象的期望访问应为**all**，则无需添加每个动词；相反，可以使用通配符字符来标识所有**动词**、**资源**或**apiGroups**。

## 识别角色

Kubernetes 授权页面（[`kubernetes.io/docs/reference/access-authn-authz/rbac/`](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)）使用以下角色作为示例，允许某人获取 pod 及其日志的详细信息：

apiVersion: rbac.authorization.k8s.io/v1

种类：角色

元数据：

命名空间：默认

名称：pod-and-pod-logs-reader

规则：

- apiGroups: [""]

资源：["pods", "pods/log"]

动词：["get", "list"]

逆向确定此角色是如何定义的，我们将从**资源**开始，因为这是最容易找到的方面。Kubernetes 中的所有对象都由 URL 表示。如果要获取默认命名空间中有关 pod 的所有信息，您将调用**/api/v1/namespaces/default/pods** URL，如果要获取特定 pod 的日志，您将调用**/api/v1/namespaces/default/pods/mypod/log** URL。

此 URL 模式适用于所有命名空间范围的对象。**pods**与**资源**对齐，**pods/log**也是如此。在尝试确定要授权的资源时，请使用 Kubernetes API 文档中的**api-reference**文档[`kubernetes.io/docs/reference/#api-reference`](https://kubernetes.io/docs/reference/#api-reference)。

如果您尝试在对象名称之后访问其他路径组件（例如在 pod 上的状态和日志），则需要明确授权。授权 pod 并不立即授权日志或状态。

基于使用 URL 映射到**资源**，你可能会认为**动词**将是 HTTP 动词。但事实并非如此。在 Kubernetes 中没有**GET**动词。动词是由 API 服务器中对象的模式定义的。好消息是，HTTP 动词和 RBAC 动词之间有静态映射（https://kubernetes.io/docs/reference/access-authn-authz/authorization/#determine-the-request-verb）。查看此 URL 时，请注意**PodSecurityPolicies**和模拟的 HTTP 动词之上有动词。这是因为**RBAC**模型不仅用于授权特定 API，还用于授权谁可以模拟用户以及如何分配**PodSecurityPolicy**对象。本章重点将放在标准 HTTP 动词映射上。

最后一个要识别的组件是 **apiGroups**。这是来自 URL 模型的另一个不一致的地方。**pods** 是“core”组的一部分，但 **apiGroups** 列表只是一个空字符串（**""**）。这些是最初 Kubernetes 的一部分的传统 API。大多数其他 API 将在 API 组中，并且该组将成为它们的 URL 的一部分。您可以通过查看要授权的对象的 API 文档来找到该组。

RBAC 模型中的不一致性可能会使调试变得困难，至少可以这么说。本章的最后一个实验将介绍调试过程，并消除定义规则时的大部分猜测。

现在我们已经定义了 Role 的内容以及如何定义特定权限，重要的是要注意，Role 可以应用于命名空间和集群级别。

## 角色与 ClusterRoles

RBAC 规则可以针对特定命名空间或整个集群进行范围限定。以前面的示例为例，如果我们将其定义为 ClusterRole 而不是 Role，并移除命名空间，我们将得到一个授权某人获取整个集群中所有 pod 的详细信息和日志的 Role。这个新角色也可以用于单独的命名空间，以将权限分配给特定命名空间中的 pod：

apiVersion: rbac.authorization.k8s.io/v1

种类：ClusterRole

元数据：

名称：cluster-pod-and-pod-logs-reader

规则：

- apiGroups: [""]

资源：["pods", "pods/log"]

动词：["get", "list"]

这个权限是全局应用于集群还是在特定命名空间范围内取决于它绑定到的主体。这将在 *RoleBindings 和 ClusterRoleBindings* 部分进行介绍。

除了在集群中应用一组规则外，ClusterRoles 也用于将规则应用于未映射到命名空间的资源，例如 PersistentVolume 和 StorageClass 对象。

在了解了如何定义 Role 之后，让我们了解一下为特定目的设计 Role 的不同方式。在接下来的部分中，我们将看看定义 Role 和它们在集群中的应用的不同模式。

## 负面角色

授权的最常见请求之一是“*我能否编写一个让我做除 xyz 之外的所有事情的 Role*？”在 RBAC 中，答案是*不行*。RBAC 要求要么允许每个资源，要么枚举特定资源和动词。这在 RBAC 中有两个原因：

+   **通过简单实现更好的安全性**：能够执行一条规则，说*每个秘密除了这一个*，需要比 RBAC 提供的更复杂的评估引擎。引擎越复杂，测试和验证就越困难，破坏的可能性就越大。一个更简单的引擎只是更容易编码和保持安全。

+   意想不到的后果：允许某人做任何事情，*除了* xyz，会在集群不断增长并添加新功能时以意想不到的方式留下问题的可能性。

在第一点上，构建具有这种功能的引擎很难构建和维护。这也使规则更难以跟踪。要表达这种类型的规则，你不仅需要授权规则，还需要对这些规则进行排序。例如，要说*我想允许一切，除了这个秘密*，你首先需要一个规则，说*允许一切*，然后一个规则，说*拒绝这个秘密*。如果你把规则改成*拒绝这个秘密*然后*允许一切*，第一个规则将被覆盖。你可以为不同的规则分配优先级，但这会使事情变得更加复杂。

有多种方法可以实现这种模式，可以使用自定义授权 webhook 或使用控制器动态生成 RBAC **Role**对象。这两种方法都应被视为安全反模式，因此本章不涉及这些内容。

第二点涉及意想不到的后果。支持使用操作员模式支持不是 Kubernetes 的基础设施的供应变得越来越流行，其中自定义控制器寻找**CustomResourceDefinition**（**CRD**）的新实例来供应基础设施，如数据库。亚马逊网络服务为此目的发布了一个操作员（[`github.com/aws/aws-controllers-k8s`](https://github.com/aws/aws-controllers-k8s)）。这些操作员在其自己的命名空间中以其云的管理凭据运行，寻找其对象的新实例来供应资源。如果你有一个允许一切“除了…”的安全模型，那么一旦部署，你集群中的任何人都可以供应具有实际成本并可能造成安全漏洞的云资源。从安全的角度来枚举你的资源是了解正在运行的内容和谁有访问权限的重要部分。

Kubernetes 集群的趋势是通过自定义资源 API 在集群外提供对基础设施的更多控制。您可以为 VM、额外节点或任何类型的 API 驱动云基础设施提供任何内容。除了 RBAC 之外，您还可以使用其他工具来减轻某人创建不应该创建的资源的风险，但这些应该是次要措施。

## 聚合的 ClusterRoles

ClusterRoles 可能会很快变得令人困惑并且难以维护。最好将它们分解为较小的 ClusterRoles，以根据需要进行组合。以管理员 ClusterRole 为例，它旨在让某人在特定命名空间内做任何事情。当我们查看管理员 ClusterRole 时，它列举了几乎所有资源。您可能会认为有人编写了这个 ClusterRole，以便它包含所有这些资源，但那将非常低效，而且随着新的资源类型被添加到 Kubernetes，会发生什么？管理员 ClusterRole 是一个聚合的 ClusterRole。看一下**ClusterRole**：

种类：ClusterRole

apiVersion：rbac.authorization.k8s.io/v1

元数据：

名称：admin

标签：

kubernetes.io/bootstrapping：rbac-defaults

注释：

rbac.authorization.kubernetes.io/autoupdate：'true'

规则：

。

。

。

aggregationRule：

clusterRoleSelectors：

- 匹配标签：

rbac.authorization.k8s.io/aggregate-to-admin：'true'

关键是**aggregationRule**部分。该部分告诉 Kubernetes 将所有具有**rbac.authorization.k8s.io/aggregate-to-admin**标签为 true 的 ClusterRoles 的规则合并起来。当创建新的 CRD 时，管理员无法创建该 CRD 的实例，而不添加包含此标签的新 ClusterRole。为了允许命名空间管理员用户创建新的**myapi**/**superwidget**对象的实例，创建一个新的**ClusterRole**：

apiVersion：rbac.authorization.k8s.io/v1

种类：ClusterRole

元数据：

名称：aggregate-superwidget-admin

标签：

# 将这些权限添加到“admin”默认角色。

rbac.authorization.k8s.io/aggregate-to-admin："true"

规则：

- apiGroups：["myapi"]

资源：["superwidgets"]

动词：["get"，"list"，"watch"，"create"，"update"，"patch"，"delete"]

下次您查看管理员 ClusterRole 时，它将包括**myapi**/**superwidgets**。您还可以直接引用此 ClusterRole 以获取更具体的权限。

## RoleBindings 和 ClusterRoleBindings

一旦权限被定义，就需要将其分配给某个东西以启用它。这个“东西”可以是用户、组或服务账户。这些选项被称为主题。与角色和 ClusterRoles 一样，RoleBinding 将一个角色或 ClusterRole 绑定到特定的命名空间，而 ClusterRoleBinding 将在整个集群中应用一个 ClusterRole。一个绑定可以有多个主题，但只能引用一个单一的角色或 ClusterRole。为了将本章前面创建的**pod-and-pod-logs-reader**角色分配给默认命名空间中名为**mysa**的服务账户、名为**podreader**的用户，或者拥有**podreaders**组的任何人，创建一个**RoleBinding**：

api 版本：rbac.authorization.k8s.io/v1

类型：RoleBinding

元数据：

名称：pod-and-pod-logs-reader

命名空间：默认

主题：

- 类型：ServiceAccount

名称：mysa

命名空间：默认

api 组：rbac.authorization.k8s.io

- 类型：用户

名称：podreader

- 类型：组

名称：podreaders

roleRef：

类型：角色

名称：pod-and-pod-logs-reader

api 组：rbac.authorization.k8s.io

前面的**RoleBinding**列出了三个不同的主题：

+   **ServiceAccount**：集群中的任何服务账户都可以被授权为 RoleBinding。必须包含命名空间，因为 RoleBinding 可以授权任何命名空间中的服务账户，而不仅仅是定义 RoleBinding 的命名空间。

+   **用户**：用户是由认证过程断言的。请记住来自*第七章*，*将认证集成到您的集群中*，在 Kubernetes 中没有代表用户的对象。

+   **组**：与用户一样，组也是认证过程的一部分，并与一个对象相关联。

最后，我们引用了之前创建的角色。类似地，为了将相同的主题赋予在整个集群中读取 pod 及其日志的能力，可以创建一个 ClusterRoleBinding 来引用本章前面创建的**cluster-pod-and-pod-logs-reader** ClusterRole：

api 版本：rbac.authorization.k8s.io/v1

类型：ClusterRoleBinding

元数据：

名称：cluster-pod-and-pod-logs-reader

主题：

- 类型：ServiceAccount

名称：mysa

命名空间：默认

api 组：rbac.authorization.k8s.io

- 类型：用户

名称：podreader

- 类型：组

名称：podreaders

roleRef：

类型：ClusterRole

名称：cluster-pod-and-pod-logs-reader

api 组：rbac.authorization.k8s.io

**ClusterRoleBinding**绑定到相同的主体，但是绑定到一个 ClusterRole 而不是命名空间绑定的 Role。现在，这些用户可以读取所有命名空间中的所有 pod 详情和 pod/logs，而不是只能读取默认命名空间中的 pod 详情和 pod/logs。

### 结合 ClusterRoles 和 RoleBindings

我们有一个使用案例，日志聚合器希望从多个命名空间中的 pod 中拉取日志，但不是所有命名空间。ClusterRoleBinding 太宽泛了。虽然 Role 可以在每个命名空间中重新创建，但这样做效率低下且维护困难。相反，定义一个 ClusterRole，但在适用的命名空间中从 RoleBinding 中引用它。这允许重用权限定义，同时仍将这些权限应用于特定的命名空间。一般来说，请注意以下内容：

+   ClusterRole + ClusterRoleBinding = 集群范围的权限

+   ClusterRole + RoleBinding = 特定于命名空间的权限

要在特定命名空间中应用我们的 ClusterRoleBinding，创建一个 Role，引用**ClusterRole**而不是命名空间的**Role**对象：

apiVersion：rbac.authorization.k8s.io/v1

种类：RoleBinding

元数据：

名称：pod-and-pod-logs-reader

命名空间：默认

主体：

- 种类：ServiceAccount

名称：mysa

命名空间：默认

apiGroup：rbac.authorization.k8s.io

- 种类：用户

名称：podreader

- 种类：组

名称：podreaders

角色引用：

种类：ClusterRole

名称：cluster-pod-and-pod-logs-reader

apiGroup：rbac.authorization.k8s.io

前面的**RoleBinding**让我们重用现有的**ClusterRole**。这减少了需要在集群中跟踪的对象数量，并且使得在 ClusterRole 权限需要更改时更容易更新权限。

在构建了我们的权限并定义了如何分配它们之后，接下来我们将看看如何将企业身份映射到集群策略中。

# 将企业身份映射到 Kubernetes 以授权对资源的访问

集中身份验证的好处之一是利用企业现有的身份，而不是必须创建用户需要记住的新凭据。重要的是要知道如何将您的策略映射到这些集中的用户。在*第七章*中，*将身份验证集成到您的集群*，您创建了一个集群，并将其与**Active Directory 联合服务**（**ADFS**）或 Tremolo Security 的测试身份提供者集成。为了完成集成，创建了以下**ClusterRoleBinding**：

apiVersion：rbac.authorization.k8s.io/v1

类型：ClusterRoleBinding

元数据：

名称：ou-cluster-admins

主题：

- 类型：组

名称：k8s-cluster-admins

apiGroup：rbac.authorization.k8s.io

roleRef：

类型：ClusterRole

名称：cluster-admin

apiGroup：rbac.authorization.k8s.io

这个绑定允许所有属于**k8s-cluster-admins**组的用户拥有完整的集群访问权限。当时，重点是身份验证，所以并没有提供太多关于为什么创建这个绑定的细节。

如果我们想直接授权我们的用户会怎样？这样，我们就可以控制谁可以访问我们的集群。我们的 RBAC **ClusterRoleBinding**会有所不同：

apiVersion：rbac.authorization.k8s.io/v1

类型：ClusterRoleBinding

元数据：

名称：ou-cluster-admins

主题：

- 类型：用户

名称：https://k8sou.apps.192-168-2-131.nip.io/auth/idp/k8sIdp#mlbiamext

apiGroup：rbac.authorization.k8s.io

roleRef：

类型：ClusterRole

名称：cluster-admin

apiGroup：rbac.authorization.k8s.io

使用与之前相同的 ClusterRole，这个 ClusterRoleBinding 将仅将**cluster-admin**权限分配给我的测试用户。

首先要指出的问题是用户在用户名前面有我们的 OpenID Connect 发行者的 URL。当 OpenID Connect 首次引入时，人们认为 Kubernetes 将与多个身份提供者和不同类型的身份提供者集成，因此开发人员希望您能够轻松区分来自不同身份来源的用户。例如，域 1 中的**mlbiamext**与域 2 中的**mlbiamext**是不同的用户。为确保用户的身份不会与来自身份提供者的另一个用户发生冲突，Kubernetes 要求在用户之前添加身份提供者的发行者。如果在 API 服务器标志中定义的用户名声明是邮件，则不适用此规则。如果您使用证书或模拟，则也不适用此规则。

除了不一致的实施要求，这种方法还可能在几个方面引起问题：

+   **更改您的身份提供者 URL**：今天，您在一个 URL 上使用一个身份提供者，但明天您决定将其移动。现在，您需要查看每个 ClusterRoleBinding 并对其进行更新。

+   **审计**：您无法查询与用户关联的所有 RoleBindings。您需要枚举每个绑定。

+   **大型绑定**：根据您拥有的用户数量，您的绑定可能会变得非常庞大且难以跟踪。

虽然有一些工具可以帮助您管理这些问题，但将绑定与组关联起来要比将其与个人用户关联起来容易得多。您可以使用**mail**属性来避免 URL 前缀，但这被认为是一种反模式，如果出于任何原因更改了电子邮件地址，将导致对集群的同样困难的更改。

在本章中，我们已经学会了如何定义访问策略并将这些策略映射到企业用户。接下来，我们需要确定如何将集群划分为租户。

# 实施命名空间多租户

为多个利益相关者或租户部署的集群应该按命名空间划分。这是 Kubernetes 从一开始就设计的边界。在部署命名空间时，通常会为命名空间中的用户分配两个 ClusterRoles：

+   **管理员**：这个聚合的 ClusterRole 提供了对 Kubernetes 提供的几乎每个资源的每个动词的访问权限，使管理员用户成为其命名空间的统治者。唯一的例外是可能影响整个集群的命名空间范围对象，例如**ResourceQuotas**。

+   **编辑**：类似于**admin**，但没有创建 RBAC 角色或 RoleBindings 的能力。

需要注意的是，**admin** ClusterRole 本身不能对命名空间对象进行更改。命名空间是集群范围的资源，因此只能通过 ClusterRoleBinding 分配权限。

根据您的多租户策略，**admin** ClusterRole 可能不合适。生成 RBAC Role 和 RoleBinding 对象的能力意味着命名空间管理员可以授予自己更改资源配额或运行提升的 PodSecurityPolicy 权限的能力。这就是 RBAC 倾向于崩溃并需要一些额外选项的地方：

+   **不要授予对 Kubernetes 的访问权限**：许多集群所有者希望让他们的用户远离 Kubernetes，并将其互动限制在外部 CI/CD 工具上。这对于微服务来说效果很好，但在多条线上开始出现问题。首先，将更多的传统应用程序移入 Kubernetes 意味着需要更多的传统管理员直接访问其命名空间。其次，如果 Kubernetes 团队让用户远离集群，他们现在就要负责了。拥有 Kubernetes 的人可能不想成为应用程序所有者希望的事情没有发生的原因，而且通常，应用程序所有者希望能够控制自己的基础设施，以确保他们能够处理任何影响其性能的情况。

+   将访问视为特权：大多数企业都需要特权用户才能访问基础设施。这通常是使用特权访问模型来完成的，其中管理员有一个单独的帐户，需要“签出”才能使用，并且只在“变更委员会”或流程批准的特定时间内获得授权。对这些帐户的使用受到严格监控。如果您已经有一个系统，特别是一个与企业的中央身份验证系统集成的系统，这是一个很好的方法。

+   **为每个租户提供一个集群**：这种模式将多租户从集群移动到基础设施层。您并没有消除问题，只是移动了解决问题的地方。这可能导致无法管理的蔓延，并且根据您如何实施 Kubernetes，成本可能会飙升。

+   **准入控制器**：这些通过限制可以创建哪些对象来增强 RBAC。例如，准入控制器可以决定阻止创建 RBAC 策略，即使 RBAC 明确允许。这个主题将在 *第十一章*，*使用 Open Policy Agent 扩展安全性* 中介绍。

除了授权访问命名空间和资源外，多租户解决方案还需要知道如何提供租户。这个主题将在最后一章，*第十四章*，*提供平台* 中介绍。

现在我们已经有了实施授权策略的策略，我们需要一种方法来调试这些策略，以及在创建它们时知道何时违反这些策略。Kubernetes 提供了审计功能，这将是下一节的重点，我们将在其中将审计日志添加到我们的 KinD 集群并调试 RBAC 策略的实施。

# Kubernetes 审计

Kubernetes 审计日志是您从 API 视角跟踪集群中发生的事情的地方。它以 JSON 格式呈现，这使得直接阅读变得更加困难，但使用诸如 Elasticsearch 等工具解析变得更加容易。在 *第十二章*，*使用 Falco 和 EFK 进行 Pod 审计* 中，我们将介绍如何使用 **Elasticsearch、Fluentd 和 Kibana (EFK)** 堆栈创建完整的日志系统。

## 创建审计策略

策略文件用于控制记录哪些事件以及在哪里存储日志，可以是标准日志文件或 Webhook。我们在 GitHub 存储库的 **chapter8** 目录中包含了一个示例审计策略，并将其应用于我们在整本书中一直在使用的 KinD 集群。

审计策略是一组规则，告诉 API 服务器要记录哪些 API 调用以及如何记录。当 Kubernetes 解析策略文件时，所有规则都按顺序应用，只有初始匹配的策略事件才会应用。如果对某个事件有多个规则，可能无法在日志文件中收到预期的数据。因此，您需要小心确保事件被正确创建。

策略使用 **audit.k8s.io** API 和 **Policy** 的清单类型。以下示例显示了策略文件的开头：

apiVersion: audit.k8s.io/v1beta1

kind: Policy

规则：

- level: Request

userGroups: ["system:nodes"]

动词：["update","patch"]

资源：

- group: "" # core

resources: ["nodes/status", "pods/status"]

omitStages:

- "RequestReceived"

重要提示

虽然策略文件看起来像标准的 Kubernetes 清单，但您不使用 **kubectl** 应用它。策略文件与 API 服务器上的 **--audit-policy-file** API 标志一起使用。这将在 *在集群上启用审计* 部分进行解释。

为了理解规则及其将记录的内容，我们将详细介绍每个部分。

规则的第一部分是 **level**，它确定将为事件记录的信息类型。可以为事件分配四个级别：

![表 8.1 – Kubernetes 审计级别](img/B15514_table_8.1.jpg)

表 8.1 – Kubernetes 审计级别

**userGroups**、**verbs** 和 **resources** 值告诉 API 服务器将触发审计事件的对象和操作。在这个例子中，只有来自 **system:nodes** 的请求，尝试在 **core** API 上的 **node/status** 或 **pod/status** 上执行 **update** 或 **patch** 操作才会创建事件。

**omitStages** 告诉 API 服务器在 *stage* 期间跳过任何日志记录事件，这有助于限制记录的数据量。API 请求经历四个阶段：

![表 8.2 – 审计阶段](img/B15514_table_8.2.jpg)

表 8.2 – 审计阶段

在我们的例子中，我们设置了事件忽略 **RequestReceived** 事件，这告诉 API 服务器不要记录任何传入 API 请求的数据。

每个组织都有自己的审计政策，政策文件可能会变得又长又复杂。不要害怕设置一个记录所有内容的策略，直到您掌握了可以创建的事件类型。记录所有内容并不是一个好的做法，因为日志文件会变得非常庞大。调整审计策略是一个随着时间学习的技能，随着您对 API 服务器的了解越来越多，您将开始了解哪些事件对审计最有价值。

策略文件只是启用集群审计的开始，现在我们已经了解了策略文件，让我们解释如何在集群上启用审计。

## 在集群上启用审计

启用审计对于每个 Kubernetes 发行版都是特定的。在本节中，我们将在 KinD 中启用审计日志以了解低级步骤。作为一个快速提醒，上一章的最终产品是一个启用模拟的 KinD 集群（而不是直接集成 OpenID Connect）。本章中的其余步骤和示例假定正在使用此集群。

您可以手动按照本节中的步骤操作，也可以在 GitHub 存储库的**chapter8**目录中执行包含的脚本**enable-auditing.sh**：

1.  首先，将示例审计策略从**chapter8**目录复制到 API 服务器：

k8s@book:~/kind-oidc-ldap-master$ docker cp k8s-audit-policy.yaml cluster01-control-plane:/etc/kubernetes/audit/

1.  接下来，在 API 服务器上创建存储审计日志和策略配置的目录。我们将进入容器，因为我们需要在下一步中修改 API 服务器文件：

k8s@book:~/kind-oidc-ldap-master$ docker exec -ti cluster01-control-plane bash

root@cluster01-control-plane:/# mkdir /var/log/k8s

root@cluster01-control-plane:/# mkdir /etc/kubernetes/audit

root@cluster01-control-plane:/# exit

此时，您已经在 API 服务器上有了审计策略，并且可以启用 API 选项以使用该文件。

1.  在 API 服务器上，编辑**kubeadm**配置文件**/etc/kubernetes/manifests/kube-apiserver.yaml**，这是我们更新以启用 OpenID Connect 的相同文件。要启用审计，我们需要添加三个值。需要注意的是，许多 Kubernetes 集群可能只需要文件和 API 选项。由于我们正在使用 KinD 集群进行测试，我们需要第二和第三步。

1.  首先，为启用审计日志的 API 服务器添加命令行标志。除了策略文件，我们还可以添加选项来控制日志文件的轮换、保留和最大大小：

- --tls-private-key-**file=/etc/kubernetes/pki/apiserver.key**

**    - --audit-log-path=/var/log/k8s/audit.log**

**    - --audit-log-maxage=1**

**    - --audit-log-maxbackup=10**

**    - --audit-log-maxsize=10**

**    - --audit-policy-file=/etc/kubernetes/audit/k8s-audit-policy.yaml**

请注意，该选项指向您在上一步中复制的策略文件。

1.  接下来，在**volumeMounts**部分添加存储策略配置和生成日志的目录：

- mountPath: /usr/share/ca-certificates

name: usr-share-ca-certificates

readOnly: true

- mountPath：/var/log/k8s

名称：var-log-k8s

只读：false

- mountPath：/etc/kubernetes/audit

名称：etc-kubernetes-audit

只读：true

1.  最后，将 **hostPath** 配置添加到 **volumes** 部分，以便 Kubernetes 知道在哪里挂载本地路径：

- hostPath：

路径：/usr/share/ca-certificates

类型：目录或创建

名称：usr-share-ca-certificates

- hostPath：

路径：/var/log/k8s

类型：目录或创建

名称：var-log-k8s

- hostPath：

路径：/etc/kubernetes/audit

类型：目录或创建

名称：etc-kubernetes-audit

1.  保存并退出文件。

1.  与所有 API 选项更改一样，您需要重新启动 API 服务器才能使更改生效；但是，KinD 将检测到文件已更改并自动重新启动 API 服务器的 pod。

退出附加的 shell 并检查 **kube-system** 命名空间中的 pods：

k8s@book:~/kind-oidc-ldap-master$ kubectl get pods -n kube-system

名称：READY STATUS RESTARTS AGE

calico-kube-controllers-5b644bc49c-q68q7 1/1 Running 0 28m

calico-node-2cvm9 1/1 Running 0 28m

calico-node-n29tl 1/1 Running 0 28m

coredns-6955765f44-gzvjd 1/1 Running 0 28m

coredns-6955765f44-r567x 1/1 Running 0 28m

etcd-cluster01-control-plane 1/1 Running 0 28m

kube-apiserver-cluster01-control-plane 1/1 Running 0 14s

kube-controller-manager-cluster01-control-plane 1/1 Running 0 28m

kube-proxy-h62mj 1/1 Running 0 28m

kube-proxy-pl4z4 1/1 Running 0 28m

kube-scheduler-cluster01-control-plane 1/1 Running 0 28m

API 服务器被强调仅运行了 14 秒，显示其成功重新启动。

1.  已验证 API 服务器正在运行，让我们查看审计日志以验证其是否正常工作。要检查日志，您可以使用 **docker exec** 来查看 **audit.log**：

$ docker exec cluster01-control-plane tail /var/log/k8s/audit.log

此命令生成以下日志数据：

**{"kind":"Event","apiVersion":"audit.k8s.io/v1","level":"Metadata","auditID":"473e8161-e243-4c5d-889c-42f478025cc2","stage":"ResponseComplete","requestURI":"/apis/crd.projectcalico.org/v1/clusterinformations/default","verb":"get","user":{"usernam**

**e":"system:serviceaccount:kube-system:calico-kube-controllers","uid":"38b96474-2457-4ec9-a146-9a63c2b8182e","groups":["system:serviceaccounts","system:serviceaccounts:kube-system","system:authenticated"]},"sourceIPs":["172.17.0.2"],"userAgent":"**

**Go-http-client/2.0","objectRef":{"resource":"clusterinformations","name":"default","apiGroup":"crd.projectcalico.org","apiVersion":"v1"},"responseStatus":{"metadata":{},"code":200},"requestReceivedTimestamp":"2020-05-20T00:27:07.378345Z","stageT**

**imestamp":"2020-05-20T00:27:07.381227Z","annotations":{"authorization.k8s.io/decision":"allow","authorization.k8s.io/reason":"RBAC: allowed by ClusterRoleBinding \"calico-kube-controllers\" of ClusterRole \"calico-kube-controllers\" to ServiceAc**

**计数\"calico-kube-controllers/kube-system\""}}**

这个 JSON 中包含了大量信息，直接查看日志文件可能会很具有挑战性地找到特定事件。幸运的是，现在您已经启用了审计，可以将事件转发到中央日志服务器。我们将在*第十二章**，使用 Falco 和 EFK 进行审计*中进行此操作，我们将部署一个 EFK 堆栈。

现在我们已经启用了审计，下一步是练习调试 RBAC 策略。

# 使用 audit2rbac 调试策略

有一个名为**audit2rbac**的工具，可以将审计日志中的错误反向工程成 RBAC 策略对象。在本节中，我们将使用这个工具在发现我们的一个用户无法执行他们需要执行的操作后生成一个 RBAC 策略。这是一个典型的 RBAC 调试过程，学会使用这个工具可以节省您花费在隔离 RBAC 问题上的时间：

1.  在上一章中，创建了一个通用的 RBAC 策略，允许**k8s-cluster-admins**组的所有成员成为我们集群中的管理员。如果您已登录 OpenUnison，请注销。

1.  现在，再次登录，但在屏幕底部点击**完成登录**按钮之前，删除**k8s-cluster-admins**组，并添加**cn=k8s-create-ns,cn=users,dc=domain,dc=com**：![图 8.1 – 更新的登录属性](img/Fig_8.1_B15514.jpg)

图 8.1 – 更新的登录属性

1.  接下来，点击**完成登录**。登录后，转到仪表板。就像当 OpenUnison 首次部署时一样，因为集群管理员的 RBAC 策略不再适用，所以不会有任何命名空间或其他信息。

重要提示

**memberOf**属性的格式已从简单名称更改为 LDAP 专有名称，因为这是 ADFS 或 Active Directory 最常呈现的格式。**专有名称**或**DN**从左到右读取，最左边的组件是对象的名称，其右边的每个组件是其在 LDAP 树中的位置。例如，**name cn=k8s-create-ns,cn=users,dc=domain,dc=com**组被读作"在**domain.com**域（**dc**）的**users**容器（**cn**）中的**k8s-create-ns**组。"虽然 ADFS 可以生成更用户友好的名称，但这需要特定的配置或脚本编写，因此大多数实现只是添加**memberOf**属性，列出用户所属的所有组。

1.  接下来，从令牌屏幕上复制您的**kubectl**配置，确保将其粘贴到不是您的主 KinD 终端的窗口中，以免覆盖您的主配置。

1.  一旦您的令牌设置好，尝试创建一个名为**not-going-to-work**的命名空间：

**PS C:\Users\mlb> kubectl create ns not-going-to-work**

**服务器错误（禁止）：命名空间被禁止：用户"mlbiamext"无法在集群范围的 API 组""中创建资源"namespaces"**

这里有足够的信息来反向工程一个 RBAC 策略。

1.  为了消除此错误消息，创建一个带有**"namespaces"**资源的**ClusterRole**，**apiGroups**设置为**""**，动词为**"create"**：

apiVersion: rbac.authorization.k8s.io/v1

kind: ClusterRole

metadata:

name: cluster-create-ns

rules:

- apiGroups: [""]

resources: ["namespaces"]

verbs: ["create"]

1.  接下来，为用户和这个 ClusterRole 创建一个**ClusterRoleBinding**：

apiVersion: rbac.authorization.k8s.io/v1

kind: ClusterRoleBinding

metadata:

name: cluster-create-ns

subjects:

- kind: User

name: mlbiamext

apiGroup: rbac.authorization.k8s.io

roleRef:

kind: ClusterRole

name: cluster-create-ns

apiGroup: rbac.authorization.k8s.io

1.  创建了 ClusterRole 和 ClusterRoleBinding 后，尝试再次运行命令，它将起作用：

**PS C:\Users\mlb> kubectl create ns not-going-to-work namespace/not-going-to-work created**

不幸的是，这不太可能是大多数 RBAC 调试的情况。大多数情况下，调试 RBAC 不会如此清晰或简单。通常，调试 RBAC 意味着在系统之间收到意外的错误消息。例如，如果您正在部署 **kube-Prometheus** 项目进行监控，通常希望通过 **Service** 对象进行监控，而不是通过显式命名的 pods。为了做到这一点，Prometheus ServiceAccount 需要能够列出要监控的服务所在命名空间中的 **Service** 对象。Prometheus 不会告诉您需要发生这种情况；您只会看不到列出的服务。更好的调试方法是使用一个知道如何读取审计日志并且可以根据日志中的失败来逆向工程一组角色和绑定的工具。

**audit2rbac** 工具是这样做的最佳方式。它将读取审计日志并为您提供一组可行的策略。它可能不是确切所需的策略，但它将提供一个良好的起点。让我们试一试：

1.  首先，将 shell 附加到集群的 **control-plane** 容器上，并从 GitHub 下载工具（[`github.com/liggitt/audit2rbac/releases`](https://github.com/liggitt/audit2rbac/releases)）：

**root@cluster01-control-plane:/# curl -L https://github.com/liggitt/audit2rbac/releases/download/v0.8.0/audit2rbac-linux-amd64.tar.gz 2>/dev/null > audit2rbac-linux-amd64.tar.gz**

**root@cluster01-control-plane:/# tar -xvzf audit2rbac-linux-amd64.tar.gz**

1.  在使用该工具之前，请确保关闭包含 Kubernetes 仪表板的浏览器，以免污染日志。此外，删除之前创建的 **cluster-create-ns** ClusterRole 和 ClusterRoleBinding。最后，尝试创建 **still-not-going-to-work** 命名空间：

**PS C:\Users\mlb> kubectl create ns still-not-going-to-work**

**服务器错误（禁止）：用户“mlbiamext”无法在集群范围的 API 组中创建“namespaces”资源**

1.  接下来，使用 **audit2rbac** 工具查找测试用户的任何失败： 

**root@cluster01-control-plane:/# ./audit2rbac --filename=/var/log/k8s/audit.log  --user=mlbiamext**

**打开审计源...**

**加载事件...**

**评估 API 调用...**

**生成角色...**

**apiVersion: rbac.authorization.k8s.io/v1**

**kind: ClusterRole**

**metadata:**

**  annotations:**

**    audit2rbac.liggitt.net/version: v0.8.0**

**  labels:**

**    audit2rbac.liggitt.net/generated: "true"**

**    audit2rbac.liggitt.net/user: mlbiamext**

**  名称：audit2rbac:mlbiamext**

**规则：**

**- apiGroups:**

**  - ""**

**  资源：**

**  - 命名空间**

**  动词：**

**  - 创建**

**---**

**apiVersion: rbac.authorization.k8s.io/v1**

**种类：ClusterRoleBinding**

**元数据：**

**  注释：**

**    audit2rbac.liggitt.net/version: v0.8.0**

**  标签：**

**    audit2rbac.liggitt.net/generated: "true"**

**    audit2rbac.liggitt.net/user: mlbiamext**

**  名称：audit2rbac:mlbiamext**

**roleRef:**

**  apiGroup: rbac.authorization.k8s.io**

**  种类：ClusterRole**

**  名称：audit2rbac:mlbiamext**

主题：

**- apiGroup: rbac.authorization.k8s.io**

**  种类：用户**

**  名称：mlbiamext**

**完成！**

这个命令生成了一个策略，确切地允许测试用户创建命名空间。然而，这成为了一个反模式，明确授权用户访问。

1.  为了更好地利用这个策略，最好使用我们的组：

apiVersion: rbac.authorization.k8s.io/v1

种类：ClusterRole

元数据：

名称：create-ns-audit2rbac

规则：

- apiGroups:

- ""

资源：

- 命名空间

动词：

- 创建

---

apiVersion: rbac.authorization.k8s.io/v1

种类：ClusterRoleBinding

元数据：

名称：create-ns-audit2rbac

roleRef:

apiGroup: rbac.authorization.k8s.io

种类：ClusterRole

名称：create-ns-audit2rbac

主题：

- apiGroup: rbac.authorization.k8s.io

**种类：组**

**  名称：cn=k8s-create-ns,cn=users,dc=domain,dc=com**

主要变化已经突出显示。现在，**ClusterRoleBinding**不再直接引用用户，而是引用**cn=k8s-create-ns,cn=users,dc=domain,dc=com**组，以便该组的任何成员现在都可以创建命名空间。

# 摘要

本章重点是 RBAC 策略的创建和调试。我们探讨了 Kubernetes 如何定义授权策略以及如何将这些策略应用于企业用户。我们还看了这些策略如何用于在集群中启用多租户。最后，我们在 KinD 集群中启用了审计日志，并学习了如何使用**audit2rbac**工具来调试 RBAC 问题。

使用 Kubernetes 内置的 RBAC 策略管理对象可以让您在集群中启用操作和开发任务所需的访问权限。了解如何设计策略可以帮助限制问题的影响，从而让用户更有信心自行处理更多事务。

在下一章中，我们将学习如何保护 Kubernetes 仪表板，以及如何处理组成您集群的其他基础设施应用程序的安全性。您将学习如何将我们对认证和授权的学习应用到组成您集群的应用程序中，为您的开发人员和基础设施团队提供更好、更安全的体验。

# 问题

1.  真或假 - ABAC 是授权访问 Kubernetes 集群的首选方法。

A. 正确

B. 错误

1.  角色的三个组成部分是什么？

A. 主题，名词和动词

B. 资源，动作和组

C. **apiGroups**，资源和动词

D. 组，资源和子资源

1.  你可以去哪里查找资源信息？

A. Kubernetes API 参考

B. 这个库

C. 教程和博客文章

1.  如何在命名空间之间重用角色？

A. 你不能；你需要重新创建它们。

B. 定义一个 ClusterRole，并在每个命名空间中引用它作为 RoleBinding。

C. 在一个命名空间中引用角色，使用其他命名空间的 RoleBindings。

D. 以上都不是

1.  绑定应该如何引用用户？

A. 直接，列出每个用户。

B. RoleBindings 应该只引用服务账户。

C. 只有 ClusterRoleBindings 应该引用用户。

D. 在可能的情况下，RoleBindings 和 ClusterRoleBindings 应该引用组。

1.  真或假 - RBAC 可以用于授权访问除一个资源之外的所有内容。

A. 正确

B. 错误

1.  真或假 - RBAC 是 Kubernetes 中唯一的授权方法。

A. 正确

B. 错误

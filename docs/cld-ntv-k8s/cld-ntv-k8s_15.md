# *第十二章*：Kubernetes 安全性和合规性

在本章中，您将了解一些关键的 Kubernetes 安全性要点。我们将讨论一些最近的 Kubernetes 安全问题，以及对 Kubernetes 进行的最近审计的发现。然后，我们将从我们集群的每个级别开始实施安全性，从 Kubernetes 资源及其配置的安全性开始，然后是容器安全，最后是入侵检测的运行时安全。首先，我们将讨论一些与 Kubernetes 相关的关键安全概念。

在本章中，我们将涵盖以下主题：

+   了解 Kubernetes 上的安全性

+   审查 Kubernetes 的 CVE 和安全审计

+   实施集群配置和容器安全的工具

+   处理 Kubernetes 上的入侵检测、运行时安全性和合规性

# 技术要求

为了运行本章详细介绍的命令，您需要一台支持`kubectl`命令行工具的计算机，以及一个正常运行的 Kubernetes 集群。请参阅*第一章*，*与 Kubernetes 通信*，了解快速启动 Kubernetes 的几种方法，以及如何安装`kubectl`工具的说明。

此外，您还需要一台支持 Helm CLI 工具的机器，通常具有与`kubectl`相同的先决条件-有关详细信息，请查看 Helm 文档[`helm.sh/docs/intro/install/`](https://helm.sh/docs/intro/install/)。

本章中使用的代码可以在书籍的 GitHub 存储库中找到[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter12`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter12)。

# 了解 Kubernetes 上的安全性

在讨论 Kubernetes 上的安全性时，非常重要的是要注意安全边界和共享责任。*共享责任模型*是一个常用术语，用于描述公共云服务中的安全处理方式。它指出客户对其应用程序的安全性以及公共云组件和服务的配置的安全性负责。另一方面，公共云提供商负责服务本身的安全性以及其运行的基础设施，一直到数据中心和物理层。

同样，Kubernetes 的安全性是共享的。尽管上游 Kubernetes 不是商业产品，但成千上万的 Kubernetes 贡献者和来自大型科技公司的重要组织力量确保了 Kubernetes 组件的安全性得到维护。此外，大量的个人贡献者和使用该技术的公司构成了庞大的生态系统，确保了在 CVE 报告和处理时的改进。不幸的是，正如我们将在下一节讨论的那样，Kubernetes 的复杂性意味着存在许多可能的攻击向量。

因此，作为开发人员，根据共享责任模型，你需要负责配置 Kubernetes 组件的安全性，你在 Kubernetes 上运行的应用程序的安全性，以及集群配置中的访问级别安全性。虽然你的应用程序和容器本身的安全性不在本书的范围内，但它们对 Kubernetes 的安全性绝对重要。我们将花大部分时间讨论配置级别的安全性，访问安全性和运行时安全性。

Kubernetes 本身或 Kubernetes 生态系统提供了工具、库和完整的产品来处理这些级别的安全性 - 我们将在本章中审查其中一些选项。

现在，在我们讨论这些解决方案之前，最好先从为什么可能需要它们的基本理解开始。让我们继续下一节，我们将详细介绍 Kubernetes 在安全领域遇到的一些问题。

# 审查 Kubernetes 的 CVE 和安全审计

Kubernetes 在其悠久历史中遇到了几个**通用漏洞和暴露**（**CVEs**）。在撰写本文时，MITRE CVE 数据库在搜索`kubernetes`时列出了 2015 年至 2020 年间的 73 个 CVE 公告。其中每一个要么直接与 Kubernetes 相关，要么与在 Kubernetes 上运行的常见开源解决方案相关（例如 NGINX 入口控制器）。

其中一些攻击向量足够严重，需要对 Kubernetes 源代码进行热修复，因此它们在 CVE 描述中列出了受影响的版本。关于 Kubernetes 相关的所有 CVE 的完整列表可以在[`cve.mitre.org/cgi-bin/cvekey.cgi?keyword=kubernetes`](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=kubernetes)找到。为了让你了解一些已经发现的问题，让我们按时间顺序回顾一些这些 CVE。

## 了解 CVE-2016-1905 – 不正确的准入控制

这个 CVE 是生产 Kubernetes 中的第一个重大安全问题。国家漏洞数据库（NIST 网站）给出了这个问题的基础评分为 7.7，将其归类为高影响类别。

通过这个问题，Kubernetes 准入控制器不会确保`kubectl patch`命令遵循准入规则，允许用户完全绕过准入控制器 - 在多租户场景中是一场噩梦。

## 了解 CVE-2018-1002105 – 连接升级到后端

这个 CVE 很可能是迄今为止 Kubernetes 项目中最关键的。事实上，NVD 给出了它 9.8 的严重性评分！在这个 CVE 中，发现在某些版本的 Kubernetes 中，可以利用 Kubernetes API 服务器的错误响应进行连接升级。一旦连接升级，就可以向集群中的任何后端服务器发送经过身份验证的请求。这允许恶意用户在没有适当凭据的情况下模拟完全经过身份验证的 TLS 请求。

除了这些 CVE（很可能部分受它们驱动），CNCF 在 2019 年赞助了 Kubernetes 的第三方安全审计。审计的结果是开源的，公开可用，值得一看。

## 了解 2019 年安全审计结果

正如我们在前一节中提到的，2019 年 Kubernetes 安全审计是由第三方进行的，审计结果完全是开源的。所有部分的完整审计报告可以在[`www.cncf.io/blog/2019/08/06/open-sourcing-the-kubernetes-security-audit/`](https://www.cncf.io/blog/2019/08/06/open-sourcing-the-kubernetes-security-audit/)找到。

总的来说，这次审计关注了以下 Kubernetes 功能的部分：

+   `kube-apiserver`

+   `etcd`

+   `kube-scheduler`

+   `kube-controller-manager`

+   `cloud-controller-manager`

+   `kubelet`

+   `kube-proxy`

+   容器运行时

意图是在涉及安全性时专注于 Kubernetes 最重要和相关的部分。审计的结果不仅包括完整的安全报告，还包括威胁模型和渗透测试，以及白皮书。

深入了解审计结果不在本书的范围内，但有一些重要的收获是对许多最大的 Kubernetes 安全问题的核心有很好的了解。

简而言之，审计发现，由于 Kubernetes 是一个复杂的、高度网络化的系统，具有许多不同的设置，因此有许多可能的配置，经验不足的工程师可能会执行，并在这样做的过程中，打开他们的集群给外部攻击者。

Kubernetes 的这个想法足够复杂，以至于不安全的配置很容易发生，这一点很重要，需要注意和牢记。

整个审计值得一读-对于那些具有重要的网络安全和容器知识的人来说，这是对 Kubernetes 作为平台开发过程中所做的一些安全决策的极好的视角。

现在我们已经讨论了 Kubernetes 安全问题的发现位置，我们可以开始研究如何增加集群的安全姿态。让我们从一些默认的 Kubernetes 安全功能开始。

# 实施集群配置和容器安全的工具

Kubernetes 为我们提供了许多内置选项，用于集群配置和容器权限的安全性。由于我们已经讨论了 RBAC、TLS Ingress 和加密的 Kubernetes Secrets，让我们讨论一些我们还没有时间审查的概念：准入控制器、Pod 安全策略和网络策略。

## 使用准入控制器

准入控制器经常被忽视，但它是一个极其重要的 Kubernetes 功能。许多 Kubernetes 的高级功能都在幕后使用准入控制器。此外，您可以创建新的准入控制器规则，以添加自定义功能到您的集群中。

有两种一般类型的准入控制器：

+   变异准入控制器

+   验证准入控制器

变异准入控制器接受 Kubernetes 资源规范并返回更新后的资源规范。它们还执行副作用计算或进行外部调用（在自定义准入控制器的情况下）。

另一方面，验证准入控制器只是接受或拒绝 Kubernetes 资源 API 请求。重要的是要知道，这两种类型的控制器只对创建、更新、删除或代理请求进行操作。这些控制器不能改变或更改列出资源的请求。

当这些类型的请求进入 Kubernetes API 服务器时，它将首先通过所有相关的变异准入控制器运行请求。然后，输出（可能已经变异）将通过验证准入控制器，最后在 API 服务器中被执行（或者如果被准入控制器拒绝，则不会被执行）。

在结构上，Kubernetes 提供的准入控制器是作为 Kubernetes API 服务器的一部分运行的函数或“插件”。它们依赖于两个 webhook 控制器（它们本身就是准入控制器，只是特殊的准入控制器）：**MutatingAdmissionWebhook** 和 **ValidatingAdmissionWebhook**。所有其他准入控制器在底层都使用这两个 webhook 中的一个，具体取决于它们的类型。此外，您编写的任何自定义准入控制器都可以附加到这两个 webhook 中的任一个。

在我们看创建自定义准入控制器的过程之前，让我们回顾一下 Kubernetes 提供的一些默认准入控制器。有关完整列表，请查看 Kubernetes 官方文档 [`kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do)。

### 理解默认准入控制器

在典型的 Kubernetes 设置中有许多默认的准入控制器，其中许多对一些非常重要的基本功能是必需的。以下是一些默认准入控制器的示例。

#### **NamespaceExists** 准入控制器

**NamespaceExists** 准入控制器检查任何传入的 Kubernetes 资源（除了命名空间本身）。这是为了检查资源所附加的命名空间是否存在。如果不存在，它将在准入控制器级别拒绝资源请求。

#### PodSecurityPolicy 准入控制器

**PodSecurityPolicy** 准入控制器支持 Kubernetes Pod 安全策略，我们马上就会了解到。该控制器阻止不符合 Pod 安全策略的资源被创建。

除了默认准入控制器之外，我们还可以创建自定义准入控制器。

### 创建自定义准入控制器

可以使用两个 webhook 控制器之一动态地创建自定义准入控制器。其工作方式如下：

1.  您必须编写自己的服务器或脚本，以独立于 Kubernetes API 服务器运行。

1.  然后，您可以配置前面提到的两个 webhook 触发器之一，向您的自定义服务器控制器发送带有资源数据的请求。

1.  基于结果，webhook 控制器将告诉 API 服务器是否继续。

让我们从第一步开始：编写一个快速的准入服务器。

### 编写自定义准入控制器的服务器

为了创建我们的自定义准入控制器服务器（它将接受来自 Kubernetes 控制平面的 webhook），我们可以使用任何编程语言。与大多数对 Kubernetes 的扩展一样，Go 语言具有最好的支持和库，使编写自定义准入控制器更容易。现在，我们将使用一些伪代码。

我们的服务器的控制流将看起来像这样：

Admission-controller-server.pseudo

```
// This function is called when a request hits the
// "/mutate" endpoint
function acceptAdmissionWebhookRequest(req)
{
  // First, we need to validate the incoming req
  // This function will check if the request is formatted properly
  // and will add a "valid" attribute If so
  // The webhook will be a POST request from Kubernetes in the
  // "AdmissionReviewRequest" schema
  req = validateRequest(req);
  // If the request isn't valid, return an Error
  if(!req.valid) return Error; 
  // Next, we need to decide whether to accept or deny the Admission
  // Request. This function will add the "accepted" attribute
  req = decideAcceptOrDeny(req);
  if(!req.accepted) return Error;
  // Now that we know we want to allow this resource, we need to
  // decide if any "patches" or changes are necessary
  patch = patchResourceFromWebhook(req);
  // Finally, we create an AdmissionReviewResponse and pass it back
  // to Kubernetes in the response
  // This AdmissionReviewResponse includes the patches and
  // whether the resource is accepted.
  admitReviewResp = createAdmitReviewResp(req, patch);
  return admitReviewResp;
}
```

现在我们有了一个简单的服务器用于我们的自定义准入控制器，我们可以配置一个 Kubernetes 准入 webhook 来调用它。

### 配置 Kubernetes 调用自定义准入控制器服务器

为了告诉 Kubernetes 调用我们的自定义准入服务器，它需要一个地方来调用。我们可以在任何地方运行我们的自定义准入控制器 - 它不需要在 Kubernetes 上。

也就是说，出于本章的目的，在 Kubernetes 上运行它很容易。我们不会详细介绍清单，但让我们假设我们有一个 Service 和一个 Deployment 指向它，运行着我们的服务器的容器。Service 看起来会像这样：

Service-webhook.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: my-custom-webhook-server
spec:
  selector:
    app: my-custom-webhook-server
  ports:
    - port: 443
      targetPort: 8443
```

重要的是要注意，我们的服务器需要使用 HTTPS，以便 Kubernetes 接受 webhook 响应。有许多配置的方法，我们不会在本书中详细介绍。证书可以是自签名的，但证书的通用名称和 CA 需要与设置 Kubernetes 集群时使用的名称匹配。

现在我们的服务器正在运行并接受 HTTPS 请求，让我们告诉 Kubernetes 在哪里找到它。为此，我们使用`MutatingWebhookConfiguration`。

下面的代码块显示了`MutatingWebhookConfiguration`的一个示例：

Mutating-webhook-config-service.yaml

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  name: my-service-webhook
webhooks:
  - name: my-custom-webhook-server.default.svc
    rules:
      - operations: [ "CREATE" ]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods", "deployments", "configmaps"]
    clientConfig:
      service:
        name: my-custom-webhook-server
        namespace: default
        path: "/mutate"
      caBundle: ${CA_PEM_B64}
```

让我们分解一下我们的`MutatingWebhookConfiguration`的 YAML。正如你所看到的，我们可以在这个配置中配置多个 webhook - 尽管在这个示例中我们只做了一个。

对于每个 webhook，我们设置`name`，`rules`和`configuration`。`name`只是 webhook 的标识符。`rules`允许我们精确配置 Kubernetes 应该在哪些情况下向我们的准入控制器发出请求。在这种情况下，我们已经配置了我们的 webhook，每当发生`pods`、`deployments`和`configmaps`类型资源的`CREATE`事件时触发。

最后，我们有`clientConfig`，在其中我们指定 Kubernetes 应该如何在哪里进行 webhook 请求。由于我们在 Kubernetes 上运行我们的自定义服务器，我们指定了服务名称，以及在我们的服务器上要命中的路径（`"/mutate"`在这里是最佳实践），以及要与 HTTPS 终止证书进行比较的集群 CA。如果您的自定义准入服务器在其他地方运行，还有其他可能的配置字段-如果需要，可以查看文档（[`kubernetes.io/docs/reference/access-authn-authz/admission-controllers/`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)）。

一旦我们在 Kubernetes 中创建了`MutatingWebhookConfiguration`，就很容易测试验证。我们所需要做的就是像平常一样创建一个 Pod、Deployment 或 ConfigMap，并检查我们的请求是否根据服务器中的逻辑被拒绝或修补。

假设我们的服务器目前设置为拒绝任何包含字符串`deny-me`的 Pod。它还设置了在`AdmissionReviewResponse`中添加错误响应。

让我们使用以下的 Pod 规范：

To-deny-pod.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-to-deny
spec:
  containers:
  - name: nginx
    image: nginx
```

现在，我们可以创建我们的 Pod 来检查准入控制器。我们可以使用以下命令：

```
kubectl create -f to-deny-pod.yaml
```

这导致以下输出：

```
Error from server (InternalError): error when creating "to-deny-pod.yaml": Internal error occurred: admission webhook "my-custom-webhook-server.default.svc" denied the request: Pod name contains "to-deny"!
```

就是这样！我们的自定义准入控制器成功拒绝了一个不符合我们在服务器中指定条件的 Pod。对于被修补（而不是被拒绝但被更改）的资源，`kubectl`不会显示任何特殊响应。您需要获取相关资源以查看修补的效果。

现在我们已经探讨了自定义准入控制器，让我们看看另一种实施集群安全实践的方法- Pod 安全策略。

## 启用 Pod 安全策略

Pod 安全策略的基本原则是允许集群管理员创建规则，Pod 必须遵循这些规则才能被调度到节点上。从技术上讲，Pod 安全策略只是另一种准入控制器。然而，这个功能得到了 Kubernetes 的官方支持，并值得深入讨论，因为有许多选项可用。

Pod 安全策略可用于防止 Pod 以 root 身份运行，限制端口和卷的使用，限制特权升级等等。我们现在将回顾一部分 Pod 安全策略的功能，但要查看完整的 Pod 安全策略配置类型列表，请查阅官方 PSP 文档[https://kubernetes.io/docs/concepts/policy/pod-security-policy/]。

最后，Kubernetes 还支持用于控制容器权限的低级原语 - 即*AppArmor*，*SELinux*和*Seccomp*。这些配置超出了本书的范围，但对于高度安全的环境可能会有用。

### 创建 Pod 安全策略的步骤

实施 Pod 安全策略有几个步骤：

1.  首先，必须启用 Pod 安全策略准入控制器。

1.  这将阻止在您的集群中创建所有 Pod，因为它需要匹配的 Pod 安全策略和角色才能创建 Pod。出于这个原因，您可能希望在启用准入控制器之前创建您的 Pod 安全策略和角色。

1.  启用准入控制器后，必须创建策略本身。

1.  然后，必须创建具有对 Pod 安全策略访问权限的`Role`或`ClusterRole`对象。

1.  最后，该角色可以与**ClusterRoleBinding**或**RoleBinding**绑定到用户或服务`accountService`帐户，允许使用该服务帐户创建的 Pod 使用 Pod 安全策略可用的权限。

在某些情况下，您的集群可能默认未启用 Pod 安全策略准入控制器。让我们看看如何启用它。

### 启用 Pod 安全策略准入控制器

为了启用 PSP 准入控制器，`kube-apiserver`必须使用指定准入控制器的标志启动。在托管的 Kubernetes（EKS、AKS 等）上，PSP 准入控制器可能会默认启用，并且为初始管理员用户创建一个特权 Pod 安全策略。这可以防止 PSP 在新集群中创建 Pod 时出现任何问题。

如果您正在自行管理 Kubernetes，并且尚未启用 PSP 准入控制器，您可以通过使用以下标志重新启动`kube-apiserver`组件来启用它：

```
kube-apiserver --enable-admission-plugins=PodSecurityPolicy,ServiceAccount…<all other desired admission controllers>
```

如果您的 Kubernetes API 服务器是使用`systemd`文件运行的（如果遵循*Kubernetes：困难的方式*，它将是这样），则应该在那里更新标志。通常，`systemd`文件放置在`/etc/systemd/system/`文件夹中。

为了找出已经启用了哪些准入插件，您可以运行以下命令：

```
kube-apiserver -h | grep enable-admission-plugins
```

此命令将显示已启用的准入插件的长列表。例如，您将在输出中看到以下准入插件：

```
NamespaceLifecycle, LimitRanger, ServiceAccount…
```

现在我们确定了 PSP 准入控制器已启用，我们实际上可以创建 PSP 了。

### 创建 PSP 资源

Pod 安全策略本身可以使用典型的 Kubernetes 资源 YAML 创建。以下是一个特权 Pod 安全策略的 YAML 文件：

Privileged-psp.yaml

```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged-psp
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  privileged: true
  allowedCapabilities:
  - '*'
  volumes:
  - '*'
  hostNetwork: true
  hostPorts:
  - min: 2000
    max: 65535
  hostIPC: true
  hostPID: true
  allowPrivilegeEscalation: true
  runAsUser:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

此 Pod 安全策略允许用户或服务账户（通过**RoleBinding**或**ClusterRoleBinding**）创建具有特权功能的 Pod。例如，使用此`PodSecurityPolicy`的 Pod 将能够绑定到主机网络的端口`2000`-`65535`，以任何用户身份运行，并绑定到任何卷类型。此外，我们还有一个关于`allowedProfileNames`的`seccomp`限制的注释-这可以让您了解`Seccomp`和`AppArmor`注释与`PodSecurityPolicies`的工作原理。

正如我们之前提到的，仅仅创建 PSP 是没有任何作用的。对于将创建特权 Pod 的任何服务账户或用户，我们需要通过**Role**和**RoleBinding**（或`ClusterRole`和`ClusterRoleBinding`）为他们提供对 Pod 安全策略的访问权限。

为了创建具有对此 PSP 访问权限的`ClusterRole`，我们可以使用以下 YAML：

Privileged-clusterrole.yaml

```
apiVersion: rbac.authorization.k8s.io
kind: ClusterRole
metadata:
  name: privileged-role
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - privileged-psp
```

现在，我们可以将新创建的`ClusterRole`绑定到我们打算创建特权 Pod 的用户或服务账户上。让我们使用`ClusterRoleBinding`来做到这一点：

Privileged-clusterrolebinding.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: privileged-crb
roleRef:
  kind: ClusterRole
  name: privileged-role
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:authenticated
```

在我们的情况下，我们希望让集群上的每个经过身份验证的用户都能创建特权 Pod，因此我们绑定到`system:authenticated`组。

现在，我们可能不希望所有用户或 Pod 都具有特权。一个更现实的 Pod 安全策略会限制 Pod 的能力。

让我们看一下具有这些限制的 PSP 的一些示例 YAML：

unprivileged-psp.yaml

```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: unprivileged-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false
```

正如您所看到的，这个 Pod 安全策略在其对创建的 Pod 施加的限制方面大不相同。在此策略下，不允许任何 Pod 以 root 身份运行或升级为 root。它们还对它们可以绑定的卷的类型有限制（在前面的代码片段中已经突出显示了这一部分）-它们不能使用主机网络或直接绑定到主机端口。

在这个 YAML 中，`runAsUser`和`supplementalGroups`部分都控制可以运行或由容器添加的 Linux 用户 ID 和组 ID，而`fsGroup`键控制容器可以使用的文件系统组。

除了使用诸如`MustRunAsNonRoot`之类的规则，还可以直接指定容器可以使用的用户 ID - 任何未在其规范中明确使用该 ID 运行的 Pod 将无法调度到节点上。

要查看限制用户特定 ID 的示例 PSP，请查看以下 YAML：

Specific-user-id-psp.yaml

```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: specific-user-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 3000
  readOnlyRootFilesystem: false
```

应用此 Pod 安全策略后，将阻止任何以用户 ID`0`或`3001`或更高的身份运行的 Pod。为了创建一个满足这个条件的 Pod，我们在 Pod 规范的`securityContext`中使用`runAs`选项。

这是一个满足这一约束的示例 Pod，即使有了这个 Pod 安全策略，它也可以成功调度：

Specific-user-pod.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: specific-user-pod
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: test
    image: busybox
    securityContext:
      allowPrivilegeEscalation: false
```

正如您在这个 YAML 中看到的，我们为我们的 Pod 指定了一个特定的用户 ID`1000`来运行。我们还禁止我们的 Pod 升级为 root。即使`specific-user-psp`已经生效，这个 Pod 规范也可以成功调度。

现在我们已经讨论了 Pod 安全策略如何通过对 Pod 运行方式施加限制来保护 Kubernetes，我们可以转向网络策略，我们可以限制 Pod 的网络。

## 使用网络策略

Kubernetes 中的网络策略类似于防火墙规则或路由表。它们允许用户通过选择器指定一组 Pod，然后确定这些 Pod 可以如何以及在哪里进行通信。

为了使网络策略工作，您选择的 Kubernetes 网络插件（如*Weave*、*Flannel*或*Calico*）必须支持网络策略规范。网络策略可以像其他 Kubernetes 资源一样通过一个 YAML 文件创建。让我们从一个非常简单的网络策略开始。

这是一个限制访问具有标签`app=server`的 Pod 的网络策略规范。

Label-restriction-policy.yaml

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-network-policy
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

现在，让我们逐步解析这个网络策略的 YAML，因为这将帮助我们解释随着我们的进展一些更复杂的网络策略。

首先，在我们的规范中，我们有一个`podSelector`，它在功能上类似于节点选择器。在这里，我们使用`matchLabels`来指定这个网络策略只会影响具有标签`app=server`的 Pod。

接下来，我们为我们的网络策略指定一个策略类型。有两种策略类型：`ingress`和`egress`。一个网络策略可以指定一个或两种类型。`ingress`指的是制定适用于连接到匹配的 Pod 的网络规则，而`egress`指的是制定适用于离开匹配的 Pod 的连接的网络规则。

在这个特定的网络策略中，我们只是规定了一个单一的`ingress`规则：只有来自具有标签`app=server`的 Pod 的流量才会被接受，这些流量是源自具有标签`app:frontend`的 Pod。此外，唯一接受具有标签`app=server`的 Pod 上的流量的端口是`80`。

在`ingress`策略集中可以有多个`from`块对应多个流量规则。同样，在`egress`中也可以有多个`to`块。

重要的是要注意，网络策略是按命名空间工作的。默认情况下，如果在命名空间中没有单个网络策略，那么在该命名空间中的 Pod 之间的通信就没有任何限制。然而，一旦一个特定的 Pod 被单个网络策略选中，所有到该 Pod 的流量和从该 Pod 出去的流量都必须明确匹配一个网络策略规则。如果不匹配规则，它将被阻止。

有了这个想法，我们可以轻松地创建强制执行广泛限制的 Pod 网络策略。让我们来看看以下网络策略：

Full-restriction-policy.yaml

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: full-restriction-policy
  namespace: development
spec:
  policyTypes:
  - Ingress
  - Egress
  podSelector: {}
```

在这个`NetworkPolicy`中，我们指定我们将包括`Ingress`和`Egress`策略，但我们没有为它们写一个块。这样做的效果是自动拒绝任何`Egress`和`Ingress`的流量，因为没有规则可以匹配流量。

另外，我们的`{}` Pod 选择器值对应于选择命名空间中的每个 Pod。这条规则的最终结果是，`development`命名空间中的每个 Pod 将无法接受入口流量或发送出口流量。

重要提示

还需要注意的是，网络策略是通过结合影响 Pod 的所有单独的网络策略，然后将所有这些规则的组合应用于 Pod 流量来解释的。

这意味着，即使在我们先前的示例中限制了`development`命名空间中的所有入口和出口流量，我们仍然可以通过添加另一个网络策略来为特定的 Pod 启用它。

假设现在我们的`development`命名空间对 Pod 有完全的流量限制，我们希望允许一部分 Pod 在端口`443`上接收网络流量，并在端口`6379`上向数据库 Pod 发送流量。为了做到这一点，我们只需要创建一个新的网络策略，通过策略的叠加性质，允许这种流量。

这就是网络策略的样子：

覆盖限制网络策略.yaml

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: override-restriction-policy
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 443
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 6379
```

在这个网络策略中，我们允许`development`命名空间中的服务器 Pod 在端口`443`上接收来自前端 Pod 的流量，并在端口`6379`上向数据库 Pod 发送流量。

如果我们想要打开所有 Pod 之间的通信而没有任何限制，同时实际上还要制定网络策略，我们可以使用以下 YAML 来实现：

全开放网络策略.yaml

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
spec:
  podSelector: {}
  egress:
  - {}
  ingress:
  - {}
  policyTypes:
  - Egress
  - Ingress
```

现在我们已经讨论了如何使用网络策略来设置 Pod 之间的流量规则。然而，也可以将网络策略用作外部防火墙。为了做到这一点，我们创建基于外部 IP 而不是 Pod 作为源或目的地的网络策略规则。

让我们看一个限制与特定 IP 范围作为目标的 Pod 之间通信的网络策略的示例：

外部 IP 网络策略.yaml

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: specific-ip-policy
spec:
  podSelector:
    matchLabels:
      app: worker
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 157.10.0.0/16
        except:
        - 157.10.1.0/24
  egress:
  - to:
    - ipBlock:
        cidr: 157.10.0.0/16
        except:
        - 157.10.1.0/24
```

在这个网络策略中，我们指定了一个`Ingress`规则和一个`Egress`规则。每个规则根据网络请求的源 IP 而不是来自哪个 Pod 来接受或拒绝流量。

在我们的情况下，我们已经为我们的`Ingress`和`Egress`规则选择了一个`/16`子网掩码范围（带有指定的`/24` CIDR 异常）。这会产生一个副作用，即阻止集群内部的任何流量到达这些 Pod，因为我们的 Pod IP 都不会匹配默认集群网络设置中的规则。

然而，在指定的子网掩码中来自集群外部的流量（并且不在异常范围内）将能够向`worker`Pod 发送流量，并且还能够接受来自`worker`Pod 的流量。

随着我们讨论网络策略的结束，我们可以转向安全堆栈的一个完全不同的层面 - 运行时安全和入侵检测。

# 处理 Kubernetes 上的入侵检测、运行时安全和合规性

一旦您设置了 Pod 安全策略和网络策略，并且通常确保您的配置尽可能牢固 - Kubernetes 仍然存在许多可能的攻击向量。在本节中，我们将重点关注来自 Kubernetes 集群内部的攻击。即使在具有高度特定的 Pod 安全策略的情况下（这确实有所帮助，需要明确），您的集群中运行的容器和应用程序仍可能执行意外或恶意操作。

为了解决这个问题，许多专业人士寻求运行时安全工具，这些工具允许对应用程序进程进行持续监控和警报。对于 Kubernetes 来说，一个流行的开源工具就是*Falco*。

## 安装 Falco

Falco 自称为 Kubernetes 上进程的*行为活动监视器*。它可以监视在 Kubernetes 上运行的容器化应用程序以及 Kubernetes 组件本身。

Falco 是如何工作的？在实时中，Falco 解析来自 Linux 内核的系统调用。然后，它通过规则过滤这些系统调用 - 这些规则是可以应用于 Falco 引擎的一组配置。每当系统调用违反规则时，Falco 就会触发警报。就是这么简单！

Falco 附带了一套广泛的默认规则，可以在内核级别增加显著的可观察性。当然，Falco 支持自定义规则 - 我们将向您展示如何编写这些规则。

但首先，我们需要在我们的集群上安装 Falco！幸运的是，Falco 可以使用 Helm 进行安装。但是，非常重要的是要注意，有几种不同的安装 Falco 的方式，在事件发生时它们在有效性上有很大的不同。

我们将使用 Helm 图表安装 Falco，这对于托管的 Kubernetes 集群或者您可能无法直接访问工作节点的任何情况都非常简单且有效。

然而，为了获得最佳的安全姿态，Falco 应该直接安装到 Kubernetes 节点的 Linux 级别。使用 DaemonSet 的 Helm 图表非常易于使用，但本质上不如直接安装 Falco 安全。要直接将 Falco 安装到您的节点上，请查看[`falco.org/docs/installation/`](https://falco.org/docs/installation)上的安装说明。

有了这个警告，我们可以使用 Helm 安装 Falco：

1.  首先，我们需要将`falcosecurity`存储库添加到我们本地的 Helm 中：

```
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

接下来，我们可以继续使用 Helm 实际安装 Falco。

重要提示

Falco Helm 图表有许多可能可以在 values 文件中更改的变量-要全面审查这些变量，您可以在官方 Helm 图表存储库[`github.com/falcosecurity/charts/tree/master/falco`](https://github.com/falcosecurity/charts/tree/master/falco)上查看。

1.  要安装 Falco，请运行以下命令：

```
helm install falco falcosecurity/falco
```

此命令将使用默认值安装 Falco，您可以在[`github.com/falcosecurity/charts/blob/master/falco/values.yaml`](https://github.com/falcosecurity/charts/blob/master/falco/values.yaml)上查看默认值。

接下来，让我们深入了解 Falco 为安全意识的 Kubernetes 管理员提供了什么。

## 了解 Falco 的功能

如前所述，Falco 附带一组默认规则，但我们可以使用新的 YAML 文件轻松添加更多规则。由于我们使用的是 Helm 版本的 Falco，因此将自定义规则传递给 Falco 就像创建一个新的 values 文件或编辑具有自定义规则的默认文件一样简单。

添加自定义规则看起来像这样：

Custom-falco.yaml

```
customRules:
  my-rules.yaml: |-
    Rule1
    Rule2
    etc...
```

现在是讨论 Falco 规则结构的好时机。为了说明，让我们借用一些来自随 Falco Helm 图表一起提供的`Default` Falco 规则集的规则。

在 YAML 中指定 Falco 配置时，我们可以使用三种不同类型的键来帮助组成我们的规则。这些是宏、列表和规则本身。

在这个例子中，我们正在查看的具体规则称为`启动特权容器`。这个规则将检测特权容器何时被启动，并记录一些关于容器的信息到`STDOUT`。规则在处理警报时可以做各种事情，但记录到`STDOUT`是在发生高风险事件时增加可观察性的好方法。

首先，让我们看一下规则条目本身。这个规则使用了一些辅助条目，几个宏和列表 - 但我们将在稍后讨论这些：

```
- rule: Launch Privileged Container
  desc: Detect the initial process started in a privileged container. Exceptions are made for known trusted images.
  condition: >
    container_started and container
    and container.privileged=true
    and not falco_privileged_containers
    and not user_privileged_containers
  output: Privileged container started (user=%user.name command=%proc.cmdline %container.info image=%container.image.repository:%container.image.tag)
  priority: INFO
  tags: [container, cis, mitre_privilege_escalation, mitre_lateral_movement]
```

正如您所看到的，Falco 规则有几个部分。首先，我们有规则名称和描述。然后，我们指定规则的触发条件 - 这充当 Linux 系统调用的过滤器。如果系统调用匹配`condition`块中的所有逻辑过滤器，规则就会被触发。

当触发规则时，`output`键允许我们设置输出文本的格式。`priority`键让我们分配一个优先级，可以是`emergency`、`alert`、`critical`、`error`、`warning`、`notice`、`informational`和`debug`中的一个。

最后，`tags`键将标签应用于相关的规则，使得更容易对规则进行分类。当使用不仅仅是简单文本`STDOUT`条目的警报时，这一点尤为重要。

这里`condition`的语法特别重要，我们将重点关注过滤系统的工作原理。

首先，由于过滤器本质上是逻辑语句，您将看到一些熟悉的语法（如果您曾经编程或编写伪代码） - `and`、`and not`、`and so on`。这种语法很容易学习，可以在[`github.com/draios/sysdig/wiki/sysdig-user-guide#filtering`](https://github.com/draios/sysdig/wiki/sysdig-user-guide#filtering)找到关于它的全面讨论 - *Sysdig*过滤器语法。

需要注意的是，Falco 开源项目最初是由*Sysdig*创建的，这就是为什么它使用常见的*Sysdig*过滤器语法。

接下来，您将看到对`container_started`和`container`的引用，以及`falco_privileged_containers`和`user_privileged_containers`的引用。这些不是简单的字符串，而是宏的使用 - 引用 YAML 中其他块的引用，指定了额外的功能，并且通常使得编写规则变得更加容易，而不需要重复大量的配置。

为了了解这个规则是如何真正工作的，让我们看一下在前面规则中引用的所有宏的完整参考：

```
- macro: container
  condition: (container.id != host)
- macro: container_started
  condition: >
    ((evt.type = container or
     (evt.type=execve and evt.dir=< and proc.vpid=1)) and
     container.image.repository != incomplete)
- macro: user_sensitive_mount_containers
  condition: (container.image.repository = docker.io/sysdig/agent)
- macro: falco_privileged_containers
  condition: (openshift_image or
              user_trusted_containers or
              container.image.repository in (trusted_images) or
              container.image.repository in (falco_privileged_images) or
              container.image.repository startswith istio/proxy_ or
              container.image.repository startswith quay.io/sysdig)
- macro: user_privileged_containers
  condition: (container.image.repository endswith sysdig/agent)
```

您将在前面的 YAML 中看到，每个宏实际上只是一块可重用的`Sysdig`过滤器语法块，通常使用其他宏来完成规则功能。列表在这里没有显示，它们类似于宏，但不描述过滤逻辑。相反，它们包括一个字符串值列表，可以作为使用过滤器语法的比较的一部分。

例如，在`falco_privileged_containers`宏中的`(``trusted_images)`引用了一个名为`trusted_images`的列表。以下是该列表的来源：

```
- list: trusted_images
  items: []
```

正如您所看到的，在默认规则中，这个特定列表是空的，但自定义规则集可以在这个列表中使用一组受信任的镜像，然后这些受信任的镜像将自动被所有使用`trusted_image`列表作为其过滤规则一部分的其他宏和规则所使用。

正如之前提到的，除了跟踪 Linux 系统调用之外，Falco 在版本 v0.13.0 中还可以跟踪 Kubernetes 控制平面事件。

### 了解 Falco 中的 Kubernetes 审计事件规则

在结构上，这些 Kubernetes 审计事件规则的工作方式与 Falco 的 Linux 系统调用规则相同。以下是 Falco 中默认 Kubernetes 规则的示例：

```
- rule: Create Disallowed Pod
  desc: >
    Detect an attempt to start a pod with a container image outside of a list of allowed images.
  condition: kevt and pod and kcreate and not allowed_k8s_containers
  output: Pod started with container not in allowed list (user=%ka.user.name pod=%ka.resp.name ns=%ka.target.namespace images=%ka.req.pod.containers.image)
  priority: WARNING
  source: k8s_audit
  tags: [k8s]
```

这个规则在 Falco 中针对 Kubernetes 审计事件（基本上是控制平面事件），在创建不在`allowed_k8s_containers`列表中的 Pod 时发出警报。默认的`k8s`审计规则包含许多类似的规则，大多数在触发时输出格式化日志。

现在，我们在本章的前面谈到了一些 Pod 安全策略，你可能会发现 PSPs 和 Falco Kubernetes 审计事件规则之间有一些相似之处。例如，看看默认的 Kubernetes Falco 规则中的这个条目：

```
- rule: Create HostNetwork Pod
  desc: Detect an attempt to start a pod using the host network.
  condition: kevt and pod and kcreate and ka.req.pod.host_network intersects (true) and not ka.req.pod.containers.image.repository in (falco_hostnetwork_images)
  output: Pod started using host network (user=%ka.user.name pod=%ka.resp.name ns=%ka.target.namespace images=%ka.req.pod.containers.image)
  priority: WARNING
  source: k8s_audit
  tags: [k8s]
```

这个规则在尝试使用主机网络启动 Pod 时触发，直接映射到主机网络 PSP 设置。

Falco 利用这种相似性，让我们可以使用 Falco 作为一种`试验`新的 Pod 安全策略的方式，而不会在整个集群中应用它们并导致运行中的 Pod 出现问题。

为此，`falcoctl`（Falco 命令行工具）带有`convert psp`命令。该命令接受一个 Pod 安全策略定义，并将其转换为一组 Falco 规则。这些 Falco 规则在触发时只会将日志输出到`STDOUT`（而不会像 PSP 不匹配那样导致 Pod 调度失败），这样就可以更轻松地在现有集群中测试新的 Pod 安全策略。

要了解如何使用`falcoctl`转换工具，请查看官方 Falco 文档[`falco.org/docs/psp-support/`](https://falco.org/docs/psp-support/)。

现在我们对 Falco 工具有了很好的基础，让我们讨论一下它如何用于实施合规性控制和运行时安全。

## 将 Falco 映射到合规性和运行时安全用例

由于其可扩展性和审计低级别的 Linux 系统调用的能力，Falco 是持续合规性和运行时安全的绝佳工具。

在合规性方面，可以利用 Falco 规则集，这些规则集专门映射到合规性标准的要求-例如 PCI 或 HIPAA。这使用户能够快速检测并采取行动，处理不符合相关标准的任何进程。有几个标准的开源和闭源 Falco 规则集。

同样地，对于运行时安全，Falco 公开了一个警报/事件系统，这意味着任何触发警报的运行时事件也可以触发自动干预和补救过程。这对安全性和合规性都适用。例如，如果一个 Pod 触发了 Falco 的不合规警报，一个进程可以立即处理该警报并删除有问题的 Pod。

# 总结

在本章中，我们了解了 Kubernetes 上下文中的安全性。首先，我们回顾了 Kubernetes 上的安全性基础知识-安全堆栈的哪些层对我们的集群相关，以及如何管理这种复杂性的一些基本概念。接下来，我们了解了 Kubernetes 遇到的一些主要安全问题，以及讨论了 2019 年安全审计的结果。

然后，我们在 Kubernetes 的两个不同级别实施了安全性-首先是使用 Pod 安全策略和网络策略进行配置，最后是使用 Falco 进行运行时安全。

在下一章中，我们将学习如何通过构建自定义资源使 Kubernetes 成为您自己的。这将允许您为集群添加重要的新功能。

# 问题

1.  自定义准入控制器可以使用哪两个 Webhook 控制器的名称？

1.  空的`NetworkPolicy`对入口有什么影响？

1.  为了防止攻击者更改 Pod 功能，哪种类型的 Kubernetes 控制平面事件对于跟踪是有价值的？

# 进一步阅读

+   Kubernetes CVE 数据库：[`cve.mitre.org/cgi-bin/cvekey.cgi?keyword=kubernetes`](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=kubernetes)

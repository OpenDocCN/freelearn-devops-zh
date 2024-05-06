# *第五章*：服务和 Ingress-与外部世界通信

本章包含了 Kubernetes 提供的方法的全面讨论，允许应用程序相互通信，以及与集群外部的资源通信。您将了解 Kubernetes 服务资源及其所有可能的类型-ClusterIP、NodePort、LoadBalancer 和 ExternalName-以及如何实现它们。最后，您将学习如何使用 Kubernetes Ingress。

在本章中，我们将涵盖以下主题：

+   理解服务和集群 DNS

+   实现 ClusterIP

+   使用 NodePort

+   设置 LoadBalancer 服务

+   创建 ExternalName 服务

+   配置 Ingress

# 技术要求

为了运行本章中详细介绍的命令，您需要一台支持`kubectl`命令行工具的计算机，以及一个可用的 Kubernetes 集群。请查看*第一章*，*与 Kubernetes 通信*，了解快速启动和运行 Kubernetes 的几种方法，以及如何安装`kubectl`工具的说明。

本章中使用的代码可以在书籍的 GitHub 存储库中找到，网址为[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter5`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter5)。

# 理解服务和集群 DNS

在过去的几章中，我们已经讨论了如何有效地在 Kubernetes 上运行应用程序，使用包括 Pods、Deployments 和 StatefulSets 在内的资源。然而，许多应用程序，如 Web 服务器，需要能够接受来自其容器外部的网络请求。这些请求可能来自其他应用程序，也可能来自访问公共互联网的设备。

Kubernetes 提供了几种资源类型，用于处理允许集群外部和内部资源访问运行在 Pods、Deployments 等应用程序的各种情况。

这些属于两种主要资源类型，服务和 Ingress：

+   **服务**有几种子类型-ClusterIP、NodePort 和 LoadBalancer-通常用于提供从集群内部或外部简单访问单个应用程序。

+   Ingress 是一个更高级的资源，它创建一个控制器，负责基于路径名和主机名的路由到集群内运行的各种资源。Ingress 通过使用规则将流量转发到服务来工作。您需要使用服务来使用 Ingress。

在我们开始第一种类型的服务资源之前，让我们回顾一下 Kubernetes 如何处理集群内部的 DNS。

## 集群 DNS

让我们首先讨论在 Kubernetes 中哪些资源默认拥有自己的 DNS 名称。Kubernetes 中的 DNS 名称仅限于 Pod 和服务。Pod DNS 名称包含几个部分，结构化为子域。

在 Kubernetes 中运行的 Pod 的典型完全限定域名（FQDN）如下所示：

```
my-hostname.my-subdomain.my-namespace.svc.my-cluster-domain.example
```

让我们从最右边开始分解：

+   `my-cluster-domain.example`对应于 Cluster API 本身的配置 DNS 名称。根据用于设置集群的工具以及其运行的环境，这可以是外部域名或内部 DNS 名称。

+   `svc`是一个部分，即使在 Pod DNS 名称中也会出现 - 因此我们可以假设它会在那里。但是，正如您很快会看到的，您通常不会通过它们的 FQDN 访问 Pod 或服务。

+   `my-namespace`相当容易理解。DNS 名称的这一部分将是您的 Pod 所在的命名空间。

+   `my-subdomain`对应于 Pod 规范中的`subdomain`字段。这个字段是完全可选的。

+   最后，`my-hostname`将设置为 Pod 在 Pod 元数据中的名称。

总的来说，这个 DNS 名称允许集群中的其他资源访问特定的 Pod。这通常本身并不是很有用，特别是如果您正在使用通常有多个 Pod 的部署和有状态集。这就是服务的用武之地。

让我们来看看服务的 A 记录 DNS 名称：

```
my-svc.my-namespace.svc.cluster-domain.example
```

正如您所看到的，这与 Pod DNS 名称非常相似，不同之处在于我们在命名空间左侧只有一个值 - 就是服务名称（与 Pod 一样，这是基于元数据名称生成的）。

这些 DNS 名称的处理方式的一个结果是，在命名空间内，您可以仅通过其服务（或 Pod）名称和子域访问服务或 Pod。

例如，以前的服务 DNS 名称。在`my-namespace`命名空间内，可以通过 DNS 名称`my-svc`简单地访问服务。在`my-namespace`之外，可以通过`my-svc.my-namespace`访问服务。

现在我们已经了解了集群内 DNS 的工作原理，我们可以讨论这如何转化为服务代理。

## 服务代理类型

服务，尽可能简单地解释，提供了一个将请求转发到一个或多个运行应用程序的 Pod 的抽象。

创建服务时，我们定义了一个选择器，告诉服务将请求转发到哪些 Pod。通过`kube-proxy`组件的功能，当请求到达服务时，它们将被转发到与服务选择器匹配的各个 Pod。

在 Kubernetes 中，有三种可能的代理模式：

+   **用户空间代理模式**：最古老的代理模式，自 Kubernetes 版本 1.0 以来可用。这种代理模式将以轮询方式将请求转发到匹配的 Pod。

+   **Iptables 代理模式**：自 1.1 版本以来可用，并且自 1.2 版本以来是默认选项。这比用户空间模式的开销要低，并且可以使用轮询或随机选择。

+   **IPVS 代理模式**：自 1.8 版本以来提供的最新选项。此代理模式允许其他负载平衡选项（不仅仅是轮询）：

a. 轮询

b. 最少连接（最少数量的打开连接）

c. 源哈希

d. 目标哈希

e. 最短预期延迟

f. 从不排队

与此列表相关的是对轮询负载均衡的讨论，对于那些不熟悉的人。

轮询负载均衡涉及循环遍历潜在的服务端点列表，每个网络请求一次。以下图表显示了这个过程的简化视图，它与 Kubernetes 服务后面的 Pod 相关：

![图 5.1 - 服务负载均衡到 Pods](img/B14790_05_001.jpg)

图 5.1 - 服务负载均衡到 Pods

正如您所看到的，服务会交替将请求发送到不同的 Pod。第一个请求发送到 Pod A，第二个发送到 Pod B，第三个发送到 Pod C，然后循环。现在我们知道服务实际上如何处理请求了，让我们来回顾一下主要类型的服务，从 ClusterIP 开始。

# 实现 ClusterIP

ClusterIP 是在集群内部公开的一种简单类型的服务。这种类型的服务无法从集群外部访问。让我们来看看我们服务的 YAML 文件：

clusterip-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: my-svc
Spec:
  type: ClusterIP
  selector:
    app: web-application
    environment: staging
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```

与其他 Kubernetes 资源一样，我们有我们的元数据块和我们的`name`值。正如您可以从我们关于 DNS 的讨论中回忆起来，这个`name`值是您如何可以从集群中的其他地方访问您的服务的。因此，ClusterIP 是一个很好的选择，适用于只需要被集群内其他 Pod 访问的服务。

接下来，我们有我们的`Spec`，它由三个主要部分组成：

+   首先，我们有我们的`type`，它对应于我们服务的类型。由于默认类型是`ClusterIP`，如果您想要一个 ClusterIP 服务，实际上不需要指定类型。

+   接下来，我们有我们的`selector`。我们的`selector`由键值对组成，必须与相关 Pod 的元数据中的标签匹配。在这种情况下，我们的服务将寻找具有`app=web-application`和`environment=staging`标签的 Pod 来转发流量。

+   最后，我们有我们的`ports`块，我们可以将服务上的端口映射到我们 Pod 上的`targetPort`号码。在这种情况下，我们服务上的端口`80`（HTTP 端口）将映射到我们应用程序 Pod 上的端口`8080`。我们的服务可以打开多个端口，但在打开多个端口时，`name`字段是必需的。

接下来，让我们深入审查`protocol`选项，因为这些对我们讨论服务端口很重要。

## 协议

在我们之前的 ClusterIP 服务的情况下，我们选择了`TCP`作为我们的协议。截至目前（截至版本 1.19），Kubernetes 支持多种协议：

+   **TCP**

+   **UDP**

+   **HTTP**

+   **PROXY**

+   **SCTP**

这是一个新功能可能会出现的领域，特别是涉及 HTTP（L7）服务的地方。目前，在不同环境或云提供商中，并不完全支持所有这些协议。

重要提示

有关更多信息，您可以查看主要的 Kubernetes 文档（[`kubernetes.io/docs/concepts/services-networking/service/`](https://kubernetes.io/docs/concepts/services-networking/service/)）了解当前服务协议的状态。

现在我们已经讨论了 Cluster IP 的服务 YAML 的具体内容，我们可以继续下一个类型的服务 - NodePort。

# 使用 NodePort

NodePort 是一种面向外部的服务类型，这意味着它实际上可以从集群外部访问。创建 NodePort 服务时，将自动创建同名的 ClusterIP 服务，并由 NodePort 路由到，因此您仍然可以从集群内部访问服务。这使 NodePort 成为在无法或不可能使用 LoadBalancer 服务时外部访问应用程序的良好选择。

NodePort 听起来像它的名字 - 这种类型的服务在集群中的每个节点上打开一个可以访问服务的端口。这个端口默认在`30000`-`32767`之间，并且在服务创建时会自动链接。

以下是我们的 NodePort 服务 YAML 的样子：

NodePort 服务.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: my-svc
Spec:
  type: NodePort
  selector:
    app: web-application
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```

正如您所看到的，与 ClusterIP 服务唯一的区别是服务类型 - 然而，重要的是要注意，我们在“端口”部分中的预期端口`80`只有在访问自动创建的 ClusterIP 版本的服务时才会被使用。从集群外部，我们需要查看生成的端口链接以访问我们的节点 IP 上的服务。

为了做到这一点，我们可以使用以下命令创建我们的服务：

```
kubectl apply -f svc.yaml 
```

然后运行这个命令：

```
kubectl describe service my-svc
```

上述命令的结果将是以下输出：

```
Name:                   my-svc
Namespace:              default
Labels:                 app=web-application
Annotations:            <none>
Selector:               app=web-application
Type:                   NodePort
IP:                     10.32.0.8
Port:                   <unset> 8080/TCP
TargetPort:             8080/TCP
NodePort:               <unset> 31598/TCP
Endpoints:              10.200.1.3:8080,10.200.1.5:8080
Session Affinity:       None
Events:                 <none>
```

从这个输出中，我们看`NodePort`行，看到我们为这个服务分配的端口是`31598`。因此，这个服务可以在任何节点上通过`[NODE_IP]:[ASSIGNED_PORT]`访问。

或者，我们可以手动为服务分配一个 NodePort IP。手动分配 NodePort 的 YAML 如下：

手动 NodePort 服务.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: my-svc
Spec:
  type: NodePort
  selector:
    app: web-application
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 31233
```

正如您所看到的，我们选择了一个在`30000`-`32767`范围内的`nodePort`，在这种情况下是`31233`。要确切地了解这个 NodePort 服务在节点之间是如何工作的，请看下面的图表：

![图 5.2 - NodePort 服务](img/B14790_05_002.jpg)

图 5.2 - NodePort 服务

正如您所看到的，虽然服务可以在集群中的每个节点（节点 A、节点 B 和节点 C）访问，但网络请求仍然在所有节点的 Pod（Pod A、Pod B 和 Pod C）之间进行负载均衡，而不仅仅是访问的节点。这是确保应用程序可以从任何节点访问的有效方式。然而，在使用云服务时，您已经有了一系列工具来在服务器之间分发请求。下一个类型的服务，LoadBalancer，让我们在 Kubernetes 的上下文中使用这些工具。

# 设置 LoadBalancer 服务

LoadBalancer 是 Kubernetes 中的特殊服务类型，根据集群运行的位置提供负载均衡器。例如，在 AWS 中，Kubernetes 将提供弹性负载均衡器。

重要提示

有关 LoadBalancer 服务和配置的完整列表，请查阅 Kubernetes 服务文档，网址为[`kubernetes.io/docs/concepts/services-networking/service/#loadbalancer`](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)。

与`ClusterIP`或 NodePort 不同，我们可以以特定于云的方式修改 LoadBalancer 服务的功能。通常，这是通过服务 YAML 文件中的注释块完成的-正如我们之前讨论的那样，它只是一组键和值。要了解如何在 AWS 中完成此操作，让我们回顾一下 LoadBalancer 服务的规范：

loadbalancer-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: my-svc
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws.. 
spec:
  type: LoadBalancer
  selector:
    app: web-application
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```

虽然我们可以创建没有任何注释的 LoadBalancer，但是支持的 AWS 特定注释使我们能够（如前面的 YAML 代码所示）指定要附加到我们的负载均衡器的 TLS 证书（通过其在 Amazon 证书管理器中的 ARN）。AWS 注释还允许配置负载均衡器的日志等。

以下是 AWS 云提供商支持的一些关键注释，截至本书编写时：

+   `service.beta.kubernetes.io/aws-load-balancer-ssl-cert`

+   `service.beta.kubernetes.io/aws-load-balancer-proxy-protocol`

+   `service.beta.kubernetes.io/aws-load-balancer-ssl-ports`

重要提示

有关所有提供商的注释和解释的完整列表可以在官方 Kubernetes 文档的**云提供商**页面上找到，网址为[`kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/`](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/)。

最后，通过 LoadBalancer 服务，我们已经涵盖了您可能最常使用的服务类型。但是，对于服务本身在 Kubernetes 之外运行的特殊情况，我们可以使用另一种服务类型：ExternalName。

# 创建 ExternalName 服务

类型为 ExternalName 的服务可用于代理实际未在集群上运行的应用程序，同时仍保持服务作为可以随时更新的抽象层。

让我们来设定场景：你有一个在 Azure 上运行的传统生产应用程序，你希望从集群内部访问它。你可以在`myoldapp.mydomain.com`上访问这个传统应用程序。然而，你的团队目前正在将这个应用程序容器化，并在 Kubernetes 上运行它，而这个新版本目前正在你的`dev`命名空间环境中在你的集群上运行。

与其要求你的其他应用程序根据环境对不同的地方进行通信，你可以始终在你的生产（`prod`）和开发（`dev`）命名空间中都指向一个名为`my-svc`的 Service。

在`dev`中，这个 Service 可以是一个指向你的新容器化应用程序的 Pods 的`ClusterIP` Service。以下 YAML 显示了开发中的容器化 Service 应该如何工作：

clusterip-for-external-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: my-svc
  namespace: dev
Spec:
  type: ClusterIP
  selector:
    app: newly-containerized-app
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```

在`prod`命名空间中，这个 Service 将会是一个`ExternalName` Service：

externalname-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: my-svc
  namespace: prod
spec:
  type: ExternalName
  externalName: myoldapp.mydomain.com
```

由于我们的`ExternalName` Service 实际上并不转发请求到 Pods，所以我们不需要一个选择器。相反，我们指定一个`ExternalName`，这是我们希望 Service 指向的 DNS 名称。

以下图表显示了如何在这种模式中使用`ExternalName` Service：

![图 5.3 - ExternalName Service 配置](img/B14790_05_003.jpg)

图 5.3 - ExternalName Service 配置

在上图中，我们的**EC2 Running Legacy Application**是一个 AWS VM，不属于集群。我们的类型为**ExternalName**的**Service B**将请求路由到 VM。这样，我们的**Pod C**（或集群中的任何其他 Pod）可以通过 ExternalName 服务的 Kubernetes DNS 名称简单地访问我们的外部传统应用程序。

通过`ExternalName`，我们已经完成了对所有 Kubernetes Service 类型的审查。让我们继续讨论一种更复杂的暴露应用程序的方法 - Kubernetes Ingress 资源。

# 配置 Ingress

正如本章开头提到的，Ingress 提供了一个将请求路由到集群中的细粒度机制。Ingress 并不取代 Services，而是通过诸如基于路径的路由等功能来增强它们。为什么这是必要的？有很多原因，包括成本。一个具有 10 个路径到`ClusterIP` Services 的 Ingress 比为每个路径创建一个新的 LoadBalancer Service 要便宜得多 - 而且它保持了事情简单和易于理解。

Ingress 与 Kubernetes 中的其他服务不同。仅仅创建 Ingress 本身是不会有任何作用的。您需要两个额外的组件：

+   Ingress 控制器：您可以选择许多实现，构建在诸如 Nginx 或 HAProxy 等工具上。

+   用于预期路由的 ClusterIP 或 NodePort 服务。

首先，让我们讨论如何配置 Ingress 控制器。

## Ingress 控制器

一般来说，集群不会预先配置任何现有的 Ingress 控制器。您需要选择并部署一个到您的集群中。`ingress-nginx` 可能是最受欢迎的选择，但还有其他几种选择 - 请参阅[`kubernetes.io/docs/concepts/services-networking/ingress-controllers/`](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)获取完整列表。

让我们学习如何部署 Ingress 控制器 - 为了本书的目的，我们将坚持使用由 Kubernetes 社区创建的 Nginx Ingress 控制器 `ingress-nginx`。

安装可能因控制器而异，但对于 `ingress-nginx`，有两个主要部分。首先，要部署主控制器本身，请运行以下命令，具体取决于目标环境和最新的 Nginx Ingress 版本：

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.41.2/deploy/static/provider/cloud/deploy.yaml
```

其次，我们可能需要根据我们运行的环境来配置我们的 Ingress。对于在 AWS 上运行的集群，我们可以配置 Ingress 入口点以使用我们在 AWS 中创建的弹性负载均衡器。

重要提示

要查看所有特定于环境的设置说明，请参阅 `ingress-nginx` 文档[`kubernetes.github.io/ingress-nginx/deploy/`](https://kubernetes.github.io/ingress-nginx/deploy/)。

Nginx Ingress 控制器是一组 Pod，它将在创建新的 Ingress 资源（自定义的 Kubernetes 资源）时自动更新 Nginx 配置。除了 Ingress 控制器，我们还需要一种方式将请求路由到 Ingress 控制器 - 称为入口点。

### Ingress 入口点

默认的 `nginx-ingress` 安装还将创建一个服务，用于为 Nginx 层提供请求，此时 Ingress 规则接管。根据您配置 Ingress 的方式，这可以是一个负载均衡器或节点端口服务。在云环境中，您可能会使用云负载均衡器服务作为集群 Ingress 的入口点。

### Ingress 规则和 YAML

既然我们的 Ingress 控制器已经启动并运行，我们可以开始配置我们的 Ingress 规则了。

让我们从一个简单的例子开始。我们有两个服务，`service-a`和`service-b`，我们希望通过我们的 Ingress 在不同的路径上公开它们。一旦您的 Ingress 控制器和任何相关的弹性负载均衡器被创建（假设我们在 AWS 上运行），让我们首先通过以下步骤来创建我们的服务：

1.  首先，让我们看看如何在 YAML 中创建服务 A。让我们将文件命名为`service-a.yaml`：

service-a.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: service-a
Spec:
  type: ClusterIP
  selector:
    app: application-a
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```

1.  您可以通过运行以下命令来创建我们的服务 A：

```
kubectl apply -f service-a.yaml
```

1.  接下来，让我们创建我们的服务 B，其 YAML 代码看起来非常相似：

```
apiVersion: v1
kind: Service
metadata:
  name: service-b
Spec:
  type: ClusterIP
  selector:
    app: application-b
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8000
```

1.  通过运行以下命令来创建我们的服务 B：

```
kubectl apply -f service-b.yaml
```

1.  最后，我们可以为每个路径创建 Ingress 规则。以下是我们的 Ingress 的 YAML 代码，根据基于路径的路由规则，将根据需要拆分请求：

ingress.yaml

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-first-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: my.application.com
    http:
      paths:
      - path: /a
        backend:
          serviceName: service-a
          servicePort: 80
      - path: /b
        backend:
          serviceName: service-b
          servicePort: 80
```

在我们之前的 YAML 中，ingress 有一个单一的`host`值，这对应于通过 Ingress 传入的流量的主机请求头。然后，我们有两个路径，`/a`和`/b`，它们分别指向我们之前创建的两个`ClusterIP`服务。为了将这个配置以图形的形式呈现出来，让我们看一下下面的图表：

![图 5.4 - Kubernetes Ingress 示例](img/B14790_05_004.jpg)

图 5.4 - Kubernetes Ingress 示例

正如您所看到的，我们简单的基于路径的规则导致网络请求直接路由到正确的 Pod。这是因为`nginx-ingress`使用服务选择器来获取 Pod IP 列表，但不直接使用服务与 Pod 通信。相反，Nginx（在这种情况下）配置会在新的 Pod IP 上线时自动更新。

`host`值实际上并不是必需的。如果您将其省略，那么通过 Ingress 传入的任何流量，无论主机头如何（除非它匹配指定主机的不同规则），都将根据规则进行路由。以下的 YAML 显示了这一点：

ingress-no-host.yaml

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-first-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
   - http:
      paths:
      - path: /a
        backend:
          serviceName: service-a
          servicePort: 80
      - path: /b
        backend:
          serviceName: service-b
          servicePort: 80
```

这个先前的 Ingress 定义将流量流向基于路径的路由规则，即使没有主机头值。

同样，也可以根据主机头将流量分成多个独立的分支路径，就像这样：

ingress-branching.yaml

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multiple-branches-ingress
spec:
  rules:
  - host: my.application.com
    http:
      paths:
      - backend:
          serviceName: service-a
          servicePort: 80
  - host: my.otherapplication.com
    http:
      paths:
      - backend:
          serviceName: service-b
          servicePort: 80
```

最后，在许多情况下，您还可以使用 TLS 来保护您的 Ingress，尽管这个功能在每个 Ingress 控制器的基础上有所不同。对于 Nginx，可以使用 Kubernetes Secret 来实现这一点。我们将在下一章介绍这个功能，但现在，请查看 Ingress 端的配置：

ingress-secure.yaml

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secured-ingress
spec:
  tls:
  - hosts:
    - my.application.com
    secretName: my-tls-secret
  rules:
    - host: my.application.com
      http:
        paths:
        - path: /
          backend:
            serviceName: service-a
            servicePort: 8080
```

此配置将查找名为`my-tls-secret`的 Kubernetes Secret，以附加到 Ingress 以进行 TLS。

这结束了我们对 Ingress 的讨论。Ingress 的许多功能可能取决于您决定使用的 Ingress 控制器，因此请查看您选择的实现的文档。

# 摘要

在本章中，我们回顾了 Kubernetes 提供的各种方法，以便将在集群上运行的应用程序暴露给外部世界。主要方法是服务和 Ingress。在服务中，您可以使用 ClusterIP 服务进行集群内路由，使用 NodePort 直接通过节点上的端口访问服务。LoadBalancer 服务允许您使用现有的云负载均衡系统，而 ExternalName 服务允许您将请求路由到集群外部的资源。

最后，Ingress 提供了一个强大的工具，可以通过路径在集群中路由请求。要实现 Ingress，您需要在集群上安装第三方或开源 Ingress 控制器。

在下一章中，我们将讨论如何使用 ConfigMap 和 Secret 两种资源类型将配置信息注入到在 Kubernetes 上运行的应用程序中。

# 问题

1.  对于仅在集群内部访问的应用程序，您会使用哪种类型的服务？

1.  您如何确定 NodePort 服务正在使用哪个端口？

1.  为什么 Ingress 比纯粹的服务更具成本效益？

1.  除了支持传统应用程序外，在云平台上 ExternalName 服务可能有什么用处？

# 进一步阅读

+   有关云提供商的信息，请参阅 Kubernetes 文档：[`kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/`](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/)

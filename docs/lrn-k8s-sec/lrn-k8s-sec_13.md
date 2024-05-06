# *第十一章*：深度防御

深度防御是一种在网络安全中应用多层安全控制来保护有价值资产的方法。在传统或单片式 IT 环境中，我们可以列举出许多：认证、加密、授权、日志记录、入侵检测、防病毒、**虚拟私人网络**（**VPN**）、防火墙等等。您可能会发现这些安全控制也存在于 Kubernetes 集群中（而且应该存在）。

在之前的章节中，我们已经讨论了认证、授权、准入控制器、保护 Kubernetes 组件、保护配置、加固镜像和 Kubernetes 工作负载等主题。所有这些都构建了不同的安全控制层，以保护您的 Kubernetes 集群。在本章中，我们将讨论构建额外安全控制层的主题，这些主题与 Kubernetes 集群中的运行时防御最相关。以下是本章将要解决的问题：您的集群是否暴露了任何敏感数据？如果 Kubernetes 集群发生攻击，您能否检测到攻击？您的 Kubernetes 集群能够承受攻击吗？您如何应对攻击？

在本章中，我们将讨论 Kubernetes 审计，然后介绍高可用性的概念，并讨论如何在 Kubernetes 集群中应用高可用性。接下来，我们将介绍 Vault，这是一个方便的秘密管理产品，适用于 Kubernetes 集群。然后，我们将讨论如何使用 Falco 来检测 Kubernetes 集群中的异常活动。最后但同样重要的是，我们将介绍 Sysdig Inspect 和**用户空间的检查点和资源**（也称为**CRIU**）用于取证。

本章将涵盖以下主题：

+   介绍 Kubernetes 审计

+   在 Kubernetes 集群中启用高可用性

+   使用 Vault 管理秘密

+   使用 Falco 检测异常

+   使用 Sysdig Inspect 和 CRIU 进行取证

# 介绍 Kubernetes 审计

Kubernetes 审计是在 1.11 版本中引入的。Kubernetes 审计记录事件，例如创建部署，修补 pod，删除命名空间等，按照时间顺序进行记录。通过审计，Kubernetes 集群管理员能够回答以下问题：

+   发生了什么？（创建了一个 pod，是什么类型的 pod）

+   谁做的？（来自用户/管理员）

+   发生在什么时候？（事件的时间戳）

+   它发生在哪里？（Pod 是在哪个命名空间中创建的？）

从安全的角度来看，审计使 DevOps 团队和安全团队能够通过跟踪 Kubernetes 集群内发生的事件来更好地检测和预防异常。

在 Kubernetes 集群中，是`kube-apiserver`进行审计。当请求（例如，创建一个命名空间）发送到`kube-apiserver`时，请求可能会经过多个阶段。每个阶段将生成一个事件。已知的阶段如下：

+   `RequestReceived`：在审计处理程序接收请求而不处理它时生成事件。

+   `RequestStarted`：在发送响应头并发送响应正文之间生成事件，仅适用于长时间运行的请求，如`watch`。

+   `RequestComplete`：在发送响应正文时生成事件。

+   `Panic`：当发生紧急情况时生成事件。

在本节中，我们将首先介绍 Kubernetes 审计策略，然后向您展示如何启用 Kubernetes 审计以及持久化审计记录的几种方法。

## Kubernetes 审计策略

由于记录 Kubernetes 集群内发生的一切事情并不现实，审计策略允许用户定义关于应记录何种事件以及应记录事件的多少细节的规则。当`kube-apiserver`处理事件时，它将按顺序比较审计策略中的规则列表。第一个匹配的规则还决定了事件的审计级别。让我们看看审计策略是什么样子。以下是一个示例：

```
apiVersion: audit.k8s.io/v1 # This is required.
kind: Policy
# Skip generating audit events for all requests in RequestReceived stage. This can be either set at the policy level or rule level.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    verbs: ["create", "update"]
    namespace: ["ns1", "ns2", "ns3"]
    resources:
    - group: ""
# Only check access to resource "pods", not the sub-resource of pods which is consistent with the RBAC policy.
      resources: ["pods"]
# Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]
# Don't log authenticated requests to certain non-resource URL paths.
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs: ["/api*", "/version"]
# Log configmap and secret changes in all other namespaces at the Metadata level.
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]
```

您可以在审计策略中配置多个审计规则。每个审计规则将由以下字段配置：

+   `level`：定义审计事件详细程度的审计级别。

+   `resources`：审计的 Kubernetes 对象。资源可以通过**应用程序编程接口**（**API**）组和对象类型来指定。

+   `nonResourcesURL`：与审计的任何资源不相关的非资源**统一资源定位符**（**URL**）路径。

+   `namespace`：决定哪个命名空间中的 Kubernetes 对象将接受审计。空字符串将用于选择非命名空间对象，空列表意味着每个命名空间。

+   `verb`：决定将接受审计的 Kubernetes 对象的具体操作，例如`create`，`update`或`delete`。

+   `users`：决定审计规则适用于的经过身份验证的用户

+   `userGroups`：决定认证用户组适用于的审计规则。

+   `omitStages`：跳过在给定阶段生成事件。这也可以在策略级别设置。

审计策略允许您通过指定`verb`、`namespace`、`resources`等在细粒度级别上配置策略。规则的审计级别定义了应记录事件的详细程度。有四个审计级别，如下所述：

+   `None`：不记录与审计规则匹配的事件。

+   `Metadata`：当事件匹配审计规则时，记录请求到`kube-apiserver`的元数据（如`user`、`timestamp`、`resource`、`verb`等）。

+   `Request`：当事件匹配审计规则时，记录元数据以及请求正文。这不适用于非资源 URL。

+   `RequestResponse`：当事件匹配审计规则时，记录元数据、请求和响应正文。这不适用于非资源请求。

请求级别的事件比元数据级别的事件更详细，而`RequestResponse`级别的事件比请求级别的事件更详细。高详细度需要更多的输入/输出（I/O）吞吐量和存储。了解审计级别之间的差异非常必要，这样您就可以正确定义审计规则，既可以节约资源又可以保障安全。成功配置审计策略后，让我们看看审计事件是什么样子的。以下是一个元数据级别的审计事件：

```
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "05698e93-6ad7-4f4e-8ae9-046694bee469",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/ns1/pods",
  "verb": "create",
  "user": {
    "username": "admin",
    "uid": "admin",
    "groups": [
      "system:masters",
      "system:authenticated"
    ]
  },
  "sourceIPs": [
    "98.207.36.92"
  ],
  "userAgent": "kubectl/v1.17.4 (darwin/amd64) kubernetes/8d8aa39",
  "objectRef": {
    "resource": "pods",
    "namespace": "ns1",
    "name": "pod-1",
    "apiVersion": "v1"
  },
  "responseStatus": {
    "metadata": {},
    "code": 201
  },
  "requestReceivedTimestamp": "2020-04-09T07:10:52.471720Z",
  "stageTimestamp": "2020-04-09T07:10:52.485551Z",
  "annotations": {
    "authorization.k8s.io/decision": "allow",
    "authorization.k8s.io/reason": ""
  }
}
```

前面的审计事件显示了`user`、`timestamp`、被访问的对象、授权决定等。请求级别的审计事件在审计事件中的`requestObject`字段中提供了额外的信息。您将在`requestObject`字段中找到工作负载的规范，如下所示：

```
  "requestObject": {
    "kind": "Pod",
    "apiVersion": "v1",
    "metadata": {
      "name": "pod-2",
      "namespace": "ns2",
      "creationTimestamp": null,
      ...
    },
    "spec": {
      "containers": [
        {
          "name": "echo",
          "image": "busybox",
          "command": [
            "sh",
            "-c",
            "echo 'this is echo' && sleep 1h"
          ],
          ...
          "imagePullPolicy": "Always"
        }
      ],
      ...
      "securityContext": {},
    },
```

`RequestResponse`级别的审计事件是最详细的。事件中的`responseObject`实例几乎与`requestObject`相同，但包含了额外的信息，如资源版本和创建时间戳，如下面的代码块所示：

```
{
  "responseObject": {
      ...
      "selfLink": "/api/v1/namespaces/ns3/pods/pod-3",
      "uid": "3fd18de1-7a31-11ea-9e8d-0a39f00d8287",
      "resourceVersion": "217243",
      "creationTimestamp": "2020-04-09T07:10:53Z",
      "tolerations": [
        {
          "key": "node.kubernetes.io/not-ready",
          "operator": "Exists",
          "effect": "NoExecute",
          "tolerationSeconds": 300
        },
        {
          "key": "node.kubernetes.io/unreachable",
          "operator": "Exists",
          "effect": "NoExecute",
          "tolerationSeconds": 300
        }
      ],
      ...
    },
 }
```

请务必正确选择审计级别。更详细的日志提供了对正在进行的活动更深入的洞察。然而，存储和处理审计事件的时间成本更高。值得一提的是，如果在 Kubernetes 秘密对象上设置了请求或`RequestResponse`审计级别，秘密内容将被记录在审计事件中。如果将审计级别设置为比包含敏感数据的 Kubernetes 对象的元数据更详细，您应该使用敏感数据遮蔽机制，以避免秘密被记录在审计事件中。

Kubernetes 审计功能通过对象类型、命名空间、操作、用户等提供了对 Kubernetes 对象的审计灵活性。由于 Kubernetes 审计默认情况下未启用，接下来，让我们看看如何启用 Kubernetes 审计并存储审计记录。

## 配置审计后端

为了启用 Kubernetes 审计，您需要在启动`kube-apiserver`时传递`--audit-policy-file`标志和您的审计策略文件。可以配置两种类型的审计后端来处理审计事件：日志后端和 webhook 后端。让我们来看看它们。

### 日志后端

日志后端将审计事件写入主节点上的文件。以下标志用于在`kube-apiserver`中配置日志后端：

+   `--log-audit-path`：指定主节点上的日志路径。这是打开或关闭日志后端的标志。

+   `--audit-log-maxage`：指定保留审计记录的最大天数。

+   `--audit-log-maxbackup`：指定主节点上要保留的审计文件的最大数量。

+   `--audit-log-maxsize`：指定在日志文件被轮换之前的最大兆字节大小。

让我们来看看 webhook 后端。

### webhook 后端

webhook 后端将审计事件写入注册到`kube-apiserver`的远程 webhook。要启用 webhook 后端，您需要使用 webhook 配置文件设置`--audit-webhook-config-file`标志。此标志也在启动`kube-apiserver`时指定。以下是一个用于为稍后将更详细介绍的 Falco 服务注册 webhook 后端的 webhook 配置的示例：

```
apiVersion: v1
kind: Config
clusters:
- name: falco
  cluster:
    server: http://$FALCO_SERVICE_CLUSTERIP:8765/k8s_audit
contexts:
- context:
    cluster: falco
    user: ""
  name: default-context
current-context: default-context
preferences: {}
users: []
```

`server`字段中指定的 URL（`http://$FALCO_SERVICE_CLUSTERIP:8765/k8s_audit`）是审计事件将要发送到的远程端点。自 Kubernetes 1.13 版本以来，可以通过`AuditSink`对象动态配置 webhook 后端，该对象仍处于 alpha 阶段。

在本节中，我们介绍了 Kubernetes 审计，介绍了审计策略和审计后端。在下一节中，我们将讨论 Kubernetes 集群中的高可用性。

# 在 Kubernetes 集群中启用高可用性

可用性指的是用户访问服务或系统的能力。系统的高可用性确保了系统的约定的正常运行时间。例如，如果只有一个实例来提供服务，而该实例宕机，用户将无法再访问该服务。具有高可用性的服务由多个实例提供。当一个实例宕机时，备用实例仍然可以提供服务。以下图表描述了具有和不具有高可用性的服务：

![图 11.1 - 具有和不具有高可用性的服务](img/B15566_11_001.jpg)

图 11.1 - 具有和不具有高可用性的服务

在 Kubernetes 集群中，通常会有多个工作节点。集群的高可用性得到了保证，即使一个工作节点宕机，仍然有其他工作节点来承载工作负载。然而，高可用性不仅仅是在集群中运行多个节点。在本节中，我们将从三个层面来看 Kubernetes 集群中的高可用性：工作负载、Kubernetes 组件和云基础设施。

## 启用 Kubernetes 工作负载的高可用性

对于 Kubernetes 工作负载，比如部署和 StatefulSet，您可以在规范中指定`replicas`字段，用于指定微服务运行多少个复制的 pod，并且控制器将确保在集群中的不同工作节点上有`x`个 pod 运行，如`replicas`字段中指定的那样。DaemonSet 是一种特殊的工作负载；控制器将确保在集群中的每个节点上都有一个 pod 运行，假设您的 Kubernetes 集群有多个节点。因此，在部署或 StatefulSet 中指定多个副本，或者使用 DaemonSet，将确保您的工作负载具有高可用性。为了确保工作负载的高可用性，还需要确保 Kubernetes 组件的高可用性。

## 启用 Kubernetes 组件的高可用性

高可用性也适用于 Kubernetes 组件。让我们来回顾一下几个关键的 Kubernetes 组件，如下所示：

+   `kube-apiserver`：Kubernetes API 服务器（`kube-apiserver`）是一个控制平面组件，用于验证和配置诸如 pod、服务和控制器之类的对象的数据。它使用**REepresentational State Transfer**（**REST**）请求与对象进行交互。

+   `etcd`：`etcd`是一个高可用性的键值存储，用于存储配置、状态和元数据等数据。其`watch`功能使 Kubernetes 能够监听配置的更新并相应地进行更改。

+   `kube-scheduler`：`kube-scheduler`是 Kubernetes 的默认调度程序。它会观察新创建的 pod 并将 pod 分配给节点。

+   `kube-controller-manager`：Kubernetes 控制器管理器是观察状态更新并相应地对集群进行更改的核心控制器的组合。

如果`kube-apiserver`宕机，那么基本上您的集群也会宕机，因为用户或其他 Kubernetes 组件依赖于与`kube-apiserver`通信来执行其任务。如果`etcd`宕机，那么集群和对象的状态将无法被消费。`kube-scheduler`和`kube-controller-manager`也很重要，以确保工作负载在集群中正常运行。所有这些组件都在主节点上运行，以确保组件的高可用性。一个简单的方法是为您的 Kubernetes 集群启动多个主节点，可以通过`kops`或`kubeadm`来实现。您会发现类似以下的内容：

```
$ kubectl get pods -n kube-system
...
etcd-manager-events-ip-172-20-109-109.ec2.internal       1/1     Running   0          4h15m
etcd-manager-events-ip-172-20-43-65.ec2.internal         1/1     Running   0          4h16m
etcd-manager-events-ip-172-20-67-151.ec2.internal        1/1     Running   0          4h16m
etcd-manager-main-ip-172-20-109-109.ec2.internal         1/1     Running   0          4h15m
etcd-manager-main-ip-172-20-43-65.ec2.internal           1/1     Running   0          4h15m
etcd-manager-main-ip-172-20-67-151.ec2.internal          1/1     Running   0          4h16m
kube-apiserver-ip-172-20-109-109.ec2.internal            1/1     Running   3          4h15m
kube-apiserver-ip-172-20-43-65.ec2.internal              1/1     Running   4          4h16m
kube-apiserver-ip-172-20-67-151.ec2.internal             1/1     Running   4          4h15m
kube-controller-manager-ip-172-20-109-109.ec2.internal   1/1     Running   0          4h15m
kube-controller-manager-ip-172-20-43-65.ec2.internal     1/1     Running   0          4h16m
kube-controller-manager-ip-172-20-67-151.ec2.internal    1/1     Running   0          4h15m
kube-scheduler-ip-172-20-109-109.ec2.internal            1/1     Running   0          4h15m
kube-scheduler-ip-172-20-43-65.ec2.internal              1/1     Running   0          4h15m
kube-scheduler-ip-172-20-67-151.ec2.internal             1/1     Running   0          4h16m
```

现在您有多个`kube-apiserver` pod、`etcd` pod、`kube-controller-manager` pod 和`kube-scheduler` pod 在`kube-system`命名空间中运行，并且它们在不同的主节点上运行。还有一些其他组件，如`kubelet`和`kube-proxy`，它们在每个节点上运行，因此它们的可用性由节点的可用性保证，并且`kube-dns`默认情况下会启动多个 pod，因此它们的高可用性是得到保证的。无论您的 Kubernetes 集群是在公共云上运行还是在私有数据中心中运行——基础设施都是支持 Kubernetes 集群可用性的支柱。接下来，我们将讨论云基础设施的高可用性，并以云提供商为例。

## 启用云基础设施的高可用性

云提供商通过位于不同地区的多个数据中心提供全球范围的云服务。云用户可以选择在哪个地区和区域（实际数据中心）托管他们的服务。区域和区域提供了对大多数类型的物理基础设施和基础设施软件服务故障的隔离。请注意，云基础设施的可用性也会影响托管在云中的 Kubernetes 集群上运行的服务。您应该利用云的高可用性，并最终确保在 Kubernetes 集群上运行的服务的高可用性。以下代码块提供了使用`kops`指定区域的示例，以利用云基础设施的高可用性：

```
export NODE_SIZE=${NODE_SIZE:-t2.large}
export MASTER_SIZE=${MASTER_SIZE:-t2.medium}
export ZONES=${ZONES:-"us-east-1a,us-east-1b,us-east-1c"}
export KOPS_STATE_STORE="s3://my-k8s-state-store2/"
kops create cluster k8s-clusters.k8s-demo-zone.com \
  --cloud aws \
  --node-count 3 \
  --zones $ZONES \
  --node-size $NODE_SIZE \
  --master-size $MASTER_SIZE \
  --master-zones $ZONES \
  --networking calico \
  --kubernetes-version 1.14.3 \
  --yes \
```

Kubernetes 集群的节点如下所示：

```
$ kops validate cluster
...
INSTANCE GROUPS
NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-us-east-1a	Master	t2.medium	1	1	us-east-1a
master-us-east-1b	Master	t2.medium	1	1	us-east-1b
master-us-east-1c	Master	t2.medium	1	1	us-east-1c
nodes			Node	t2.large	3	3	us-east-1a,us-east-1b,us-east-1c
```

前面的代码块显示了分别在`us-east-1a`、`us-east-1b`和`us-east-1c`可用区运行的三个主节点。因此，作为工作节点，即使其中一个数据中心宕机或正在维护，主节点和工作节点仍然可以在其他数据中心中运行。

在本节中，我们已经讨论了 Kubernetes 工作负载、Kubernetes 组件和云基础设施的高可用性。让我们使用以下图表来总结 Kubernetes 集群的高可用性：

![图 11.2-云中 Kubernetes 集群的高可用性](img/B15566_11_002.jpg)

图 11.2-云中 Kubernetes 集群的高可用性

现在，让我们转到下一个关于在 Kubernetes 集群中管理秘密的主题。

# 使用 Vault 管理秘密

秘密管理是一个重要的话题，许多开源和专有解决方案已经被开发出来，以帮助解决不同平台上的秘密管理问题。因此，在 Kubernetes 中，它的内置`Secret`对象用于存储秘密数据，并且实际数据与其他 Kubernetes 对象一起存储在`etcd`中。默认情况下，秘密数据以明文（编码格式）存储在`etcd`中。`etcd`可以配置为在静止状态下加密秘密。同样，如果`etcd`未配置为使用**传输层安全性**（**TLS**）加密通信，则秘密数据也以明文传输。除非安全要求非常低，否则建议在 Kubernetes 集群中使用第三方解决方案来管理秘密。

在本节中，我们将介绍 Vault，这是一个**Cloud Native Computing Foundation**（**CNCF**）秘密管理项目。Vault 支持安全存储秘密、动态秘密生成、数据加密、密钥吊销等。在本节中，我们将重点介绍如何在 Kubernetes 集群中为应用程序存储和提供秘密。现在，让我们看看如何为 Kubernetes 集群设置 Vault。

## 设置 Vault

您可以使用`helm`在 Kubernetes 集群中部署 Vault，如下所示：

```
helm install vault --set='server.dev.enabled=true' https://github.com/hashicorp/vault-helm/archive/v0.4.0.tar.gz
```

请注意，设置了`server.dev.enabled=true`。这对开发环境很好，但不建议在生产环境中设置。您应该看到有两个正在运行的 pod，如下所示：

```
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          80s
vault-agent-injector-7fd6b9588b-fgsnj   1/1     Running   0          80s
```

`vault-0` pod 是用于管理和存储秘密的 pod，而`vault-agent-injector-7fd6b9588b-fgsnj` pod 负责将秘密注入带有特殊 vault 注释的 pod 中，我们将在*提供和轮换秘密*部分中更详细地展示。接下来，让我们为`postgres`数据库连接创建一个示例秘密，如下所示：

```
vault kv put secret/postgres username=alice password=pass
```

请注意，前面的命令需要在`vault-0` pod 内执行。由于您希望限制 Kubernetes 集群中仅有相关应用程序可以访问秘钥，您可能希望定义一个策略来实现，如下所示：

```
cat <<EOF > /home/vault/app-policy.hcl
path "secret*" {
  capabilities = ["read"]
}
EOF
vault policy write app /home/vault/app-policy.hcl
```

现在，您有一个定义了在`secret`路径下读取秘密权限的策略，比如`secret`/`postgres`。接下来，您希望将策略与允许的实体关联，比如 Kubernetes 中的服务账户。这可以通过执行以下命令来完成：

```
vault auth enable kubernetes
vault write auth/kubernetes/config \
   token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
   kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
vault write auth/kubernetes/role/myapp \
   bound_service_account_names=app \
   bound_service_account_namespaces=demo \
   policies=app \
   ttl=24h
```

Vault 可以利用 Kubernetes 的天真认证，然后将秘密访问策略绑定到服务账户。现在，命名空间 demo 中的服务账户 app 可以访问`postgres`秘密。现在，让我们在`vault-app.yaml`文件中部署一个演示应用程序，如下所示：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app: vault-agent-demo
spec:
  selector:
    matchLabels:
      app: vault-agent-demo
  replicas: 1
  template:
    metadata:
      annotations:
      labels:
        app: vault-agent-demo
    spec:
      serviceAccountName: app
      containers:
      - name: app
        image: jweissig/app:0.0.1
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  labels:
    app: vault-agent-demo
```

请注意，在上述的`.yaml`文件中，尚未添加注释，因此在创建应用程序时，秘密不会被注入，也不会添加 sidecar 容器。代码可以在以下片段中看到：

```
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
app-668b8bcdb9-js9mm                    1/1     Running   0          3m23s
```

接下来，我们将展示秘密注入的工作原理。

## 提供和轮换秘密

我们在部署应用程序时不展示秘密注入的原因是，我们想向您展示在注入到演示应用程序 pod 之前和之后的详细差异。现在，让我们使用以下 Vault 注释来补丁部署：

```
$ cat patch-template-annotation.yaml
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-status: "update"
        vault.hashicorp.com/agent-inject-secret-postgres: "secret/postgres"
        vault.hashicorp.com/agent-inject-template-postgres: |
          {{- with secret "secret/postgres" -}}
          postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
          {{- end }}
        vault.hashicorp.com/role: "myapp"
```

上述注释规定了将注入哪个秘密，以及以什么格式和使用哪个角色。一旦我们更新了演示应用程序的部署，我们将发现秘密已经被注入，如下所示：

```
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
app-68d47bb844-2hlrb                    2/2     Running   0          13s
$ kubectl -n demo exec -it app-68d47bb844-2hlrb -c app -- cat /vault/secrets/postgres
postgresql://alice:pass@postgres:5432/wizard
```

让我们来看一下 pod 的规范（而不是补丁后的部署）-与补丁后的部署规范相比，您会发现以下内容（用粗体标记）已经添加：

```
  containers:
  - image: jweissig/app:0.0.1
    ...
    volumeMounts:
    - mountPath: /vault/secrets
      name: vault-secrets
  - args:
    - echo ${VAULT_CONFIG?} | base64 -d > /tmp/config.json && vault agent -config=/tmp/config.json
    command:
    - /bin/sh
    - -ec
    image: vault:1.3.2
    name: vault-agent
    volumeMounts:
    - mountPath: /vault/secrets
      name: vault-secrets
 initContainers:
  - args:
    - echo ${VAULT_CONFIG?} | base64 -d > /tmp/config.json && vault agent -config=/tmp/config.json
    command:
    - /bin/sh
    - -ec
    image: vault:1.3.2
    name: vault-agent-init
    volumeMounts:
    - mountPath: /vault/secrets
      name: vault-secrets
  volumes:
   - emptyDir:
      medium: Memory
    name: vault-secrets
```

在上述列出的变化中值得一提的几件事情：注入了一个名为`vault-agent-init`的`init`容器和一个名为`vault-agent`的 sidecar 容器，以及一个名为`vault-secrets`的`emptyDir`类型卷。这就是为什么在补丁之后，你会看到演示应用程序 pod 中运行了两个容器。此外，`vault-secrets`卷被挂载在`init`容器、`sidecar`容器和`app`容器的`/vault/secrets/`目录中。秘密存储在`vault-secrets`卷中。通过预定义的变异 webhook 配置（通过`helm`安装）来完成 pod 规范的修改，如下所示：

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  ...
  name: vault-agent-injector-cfg
webhooks:
- admissionReviewVersions:
  - v1beta1
  clientConfig:
    caBundle: <CA_BUNDLE>
    service:
      name: vault-agent-injector-svc
      namespace: demo
      path: /mutate
  failurePolicy: Ignore
  name: vault.hashicorp.com
  namespaceSelector: {}
  rules:
  - apiGroups:
    - ""
    apiVersions:
    - v1
    operations:
    - CREATE
    - UPDATE
    resources:
    - pods
    scope: '*'
```

注册到`kube-apiserver`的变异 webhook 配置基本上告诉`kube-apiserver`将任何 pod 的创建或更新请求重定向到`demo`命名空间中的`vault-agent-injector-svc`服务。服务的后面是`vault-agent-injector` pod。然后，`vault-agent-injector` pod 将查找相关的注释，并根据请求将`init`容器和`sidecar`容器以及存储秘密的卷注入到 pod 的规范中。为什么我们需要一个`init`容器和一个`sidecar`容器？`init`容器是为了预先填充我们的秘密，而`sidecar`容器是为了在整个应用程序生命周期中保持秘密数据同步。

现在，让我们运行以下代码来更新秘密，并看看会发生什么：

```
vault kv put secret/postgres username=alice password=changeme
```

现在，密码已从`pass`更新为`changeme`在`vault` pod 中。并且，在`demo`应用程序方面，我们可以看到在等待几秒钟后，它也已经更新了：

```
$ kubectl -n demo exec -it app-68d47bb844-2hlrb -c app -- cat /vault/secrets/postgres
postgresql://alice:changeme@postgres:5432/wizard
```

Vault 是一个强大的秘密管理解决方案，它的许多功能无法在单个部分中涵盖。我鼓励你阅读文档并尝试使用它来更好地了解 Vault。接下来，让我们谈谈在 Kubernetes 中使用 Falco 进行运行时威胁检测。

# 使用 Falco 检测异常

Falco 是一个 CNCF 开源项目，用于检测云原生环境中的异常行为或运行时威胁，比如 Kubernetes 集群。它是一个基于规则的运行时检测引擎，具有约 100 个现成的检测规则。在本节中，我们将首先概述 Falco，然后向您展示如何编写 Falco 规则，以便您可以构建自己的 Falco 规则来保护您的 Kubernetes 集群。

## Falco 概述

Falco 被广泛用于检测云原生环境中的异常行为，特别是在 Kubernetes 集群中。那么，什么是异常检测？基本上，它使用行为信号来检测安全异常，比如泄露的凭据或异常活动，行为信号可以从你对实体的了解中得出正常行为是什么。

### 面临的挑战

要确定 Kubernetes 集群中的正常行为并不容易。从运行应用程序的角度来看，我们可以将它们分为三类，如下所示：

+   **Kubernetes 组件**：`kube-apiserver`、`kube-proxy`、`kubelet`、**容器运行时接口**（**CRI**）插件、**容器网络接口**（**CNI**）插件等

+   **自托管应用程序**：Java、Node.js、Golang、Python 等

+   **供应商服务**：Cassandra、Redis、MySQL、NGINX、Tomcat 等

或者，从系统的角度来看，我们有以下类型的活动：

+   文件活动，如打开、读取和写入

+   进程活动，如`execve`和`clone`系统调用

+   网络活动，如接受、连接和发送

或者，从 Kubernetes 对象的角度来看：`pod`、`secret`、`deployment`、`namespace`、`serviceaccount`、`configmap`等

为了覆盖 Kubernetes 集群中发生的所有这些活动或行为，我们将需要丰富的信息来源。接下来，让我们谈谈 Falco 依赖的事件来源，以进行异常检测，以及这些来源如何涵盖前述的活动和行为。

### 异常检测的事件来源

Falco 依赖两个事件来源进行异常检测。一个是系统调用，另一个是 Kubernetes 审计事件。对于系统调用事件，Falco 使用内核模块来监听机器上的系统调用流，并将这些系统调用传递到用户空间（最近也支持了`ebpf`）。在用户空间，Falco 还会丰富原始系统调用事件的上下文，如进程名称、容器 ID、容器名称、镜像名称等。对于 Kubernetes 审计事件，用户需要启用 Kubernetes 审计策略，并将 Kubernetes 审计 webhook 后端注册到 Falco 服务端点。然后，Falco 引擎检查引擎中加载的任何 Falco 规则匹配的任何系统调用事件或 Kubernetes 审计事件。

讨论使用系统调用和 Kubernetes 审计事件作为事件源进行异常检测的原因也很重要。系统调用是应用程序与操作系统交互以访问文件、设备、网络等资源的编程方式。考虑到容器是一组具有自己专用命名空间的进程，并且它们共享节点上相同的操作系统，系统调用是可以用来监视容器活动的统一事件源。应用程序使用什么编程语言并不重要；最终，所有函数都将被转换为系统调用以与操作系统交互。看一下下面的图表：

![图 11.3 - 容器和系统调用](img/B15566_11_003.jpg)

图 11.3 - 容器和系统调用

在上图中，有四个运行不同应用程序的容器。这些应用程序可能使用不同的编程语言编写，并且它们都调用一个函数来以不同的函数名打开文件（例如，`fopen`、`open`和`os.Open`）。然而，从操作系统的角度来看，所有这些应用程序都调用相同的系统调用`open`，但可能使用不同的参数。Falco 能够从系统调用中检索事件，因此无论应用程序是什么类型或使用什么编程语言都不重要。

另一方面，借助 Kubernetes 审计事件，Falco 可以完全了解 Kubernetes 对象的生命周期。这对于异常检测也很重要。例如，在生产环境中，以特权方式启动一个带有`busybox`镜像的 pod 可能是异常的。

总的来说，两个事件源——系统调用和 Kubernetes 审计事件——足以覆盖 Kubernetes 集群中发生的所有重要活动。现在，通过对 Falco 事件源的理解，让我们用一个高级架构图总结一下 Falco 的概述。

### 高级架构

Falco 主要由几个组件组成，如下：

+   **Falco 规则**：定义用于检测事件是否异常的规则。

+   **Falco 引擎**：使用 Falco 规则评估传入事件，并在事件匹配任何规则时产生输出。

+   **内核模块/Sysdig 库**：在发送到 Falco 引擎进行评估之前，标记系统调用事件并丰富它们。

+   Web 服务器：监听 Kubernetes 审计事件并传递给 Falco 引擎进行评估。

以下图表显示了 Falco 的内部架构：

图 11.4 - Falco 的内部架构

](image/B15566_11_004.jpg)

图 11.4 - Falco 的内部架构

现在，我们已经总结了 Falco 的概述。接下来，让我们尝试创建一些 Falco 规则并检测任何异常行为。

## 创建 Falco 规则以检测异常

在我们深入研究 Falco 规则之前，请确保已通过以下命令安装了 Falco：

```
helm install --name falco stable/falco
```

Falco DaemonSet 应该在您的 Kubernetes 集群中运行，如下面的代码块所示：

```
$ kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
falco-9h8tg   1/1     Running   10         62m
falco-cnt47   1/1     Running   5          3m45s
falco-mz6jg   1/1     Running   0          6s
falco-t4cpw   1/1     Running   0          10s
```

要启用 Kubernetes 审计并将 Falco 注册为 webhook 后端，请按照 Falco 存储库中的说明进行操作（[`github.com/falcosecurity/evolution/tree/master/examples/k8s_audit_config`](https://github.com/falcosecurity/evolution/tree/master/examples/k8s_audit_config)）。

Falco 规则中有三种类型的元素，如下所示：

+   规则：触发警报的条件。规则具有以下属性：规则名称、描述、条件、优先级、来源、标签和输出。当事件匹配任何规则的条件时，根据规则的输出定义生成警报。

+   宏：可以被其他规则或宏重复使用的规则条件片段。

+   列表：可以被宏和规则使用的项目集合。

为了方便 Falco 用户构建自己的规则，Falco 提供了一些默认列表和宏。

### 创建系统调用规则

Falco 系统调用规则评估系统调用事件 - 更准确地说是增强的系统调用。系统调用事件字段由内核模块提供，并且与 Sysdig（Sysdig 公司构建的开源工具）过滤字段相同。策略引擎使用 Sysdig 的过滤器从系统调用事件中提取信息，如进程名称、容器映像和文件路径，并使用 Falco 规则进行评估。

以下是可以用于构建 Falco 规则的最常见的 Sysdig 过滤字段：

+   proc.name：进程名称

+   fd.name：写入或读取的文件名

+   container.id：容器 ID

+   container.image.repository：不带标签的容器映像名称

+   fd.sip 和 fd.sport：服务器**Internet Protocol**（**IP**）地址和服务器端口

+   fd.cip 和 fd.cport：客户端 IP 和客户端端口

+   **evt.type**: 系统调用事件（`open`、`connect`、`accept`、`execve`等）

让我们尝试构建一个简单的 Falco 规则。假设您有一个`nginx` pod，仅从`/usr/share/nginx/html/`目录提供静态文件。因此，您可以创建一个 Falco 规则来检测任何异常的文件读取活动，如下所示：

```
    - rule: Anomalous read in nginx pod
      desc: Detect any anomalous file read activities in Nginx pod.
      condition: (open_read and container and container.image.repository="kaizheh/insecure-nginx" and fd.directory != "/usr/share/nginx/html")
      output: Anomalous file read activity in Nginx pod (user=%user.name process=%proc.name file=%fd.name container_id=%container.id image=%container.image.repository)
      priority: WARNING
```

前面的规则使用了两个默认宏：`open_read`和`container`。`open_read`宏检查系统调用事件是否仅以读模式打开，而`container`宏检查系统调用事件是否发生在容器内。然后，该规则仅适用于运行`kaizheh/insecure-nginx`镜像的容器，并且`fd.directory`过滤器从系统调用事件中检索文件目录信息。在此规则中，它检查是否有任何文件读取超出`/usr/share/nginx/html/`目录。那么，如果`nginx`的配置错误导致文件路径遍历（在任意目录下读取文件）会怎么样？以下代码块显示了一个示例：

```
# curl insecure-nginx.insecure-nginx.svc.cluster.local/files../etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/bin/false
```

与此同时，Falco 检测到超出指定目录的文件访问，输出如下：

```
08:22:19.484698397: Warning Anomalous file read activity in Nginx pod (user=<NA> process=nginx file=/etc/passwd container_id=439e2e739868 image=kaizheh/insecure-nginx) k8s.ns=insecure-nginx k8s.pod=insecure-nginx-7c99fdf44b-gffp4 container=439e2e739868 k8s.ns=insecure-nginx k8s.pod=insecure-nginx-7c99fdf44b-gffp4 container=439e2e739868
```

接下来，让我们看看如何使用 K8s 审计规则。

### 创建 K8s 审计规则

K8s 审计规则评估 Kubernetes 审计事件。在本章的前面部分，我们已经展示了 Kubernetes 审计事件记录的样子。与 Sysdig 过滤器类似，有两种方法可以从 Kubernetes 审计事件中检索信息。一种是使用**JavaScript 对象表示法**（**JSON**）指针；另一种是使用 Falco 内置过滤器。以下是用于检索 Kubernetes 审计事件信息的一些常用 Falco 内置过滤器：

+   `ka.verb`: Kubernetes 审计事件的动词字段。`jevt.value[/verb]`是其对应的 JSON 指针。

+   `ka.target.resource`: Kubernetes 审计事件的资源字段。`jevt.value[/objectRef/resource]`是其对应的 JSON 指针。

+   `ka.user.name`: Kubernetes 审计事件的用户名字段。`jevt.value[/user/username]`是其对应的 JSON 指针。

+   `ka.uri`: Kubernetes 审计事件的`requestURI`字段。`jet.value[/requestURI]`是其对应的 JSON 指针。

让我们尝试构建一个简单的 K8s 审计规则。假设您不希望在`kube-system`命名空间中部署除了一些受信任的服务镜像（如`kube-apiserver`、`etcd-manager`等）之外的镜像。因此，您可以创建一个 Falco 规则，如下所示：

```
- list: trusted_images
  items: [calico/node, kopeio/etcd-manager, k8s.gcr.io/kube-apiserver, k8s.gcr.io/kube-controller-manager, k8s.gcr.io/kube-proxy, k8s.gcr.io/kube-scheduler]
- rule: Untrusted Image Deployed in kube-system Namespace
  desc: >
    Detect an untrusted image deployed in kube-system namespace
  condition: >
    kevt and pod
    and kcreate
    and ka.target.namespace=kube-system
    and not ka.req.pod.containers.image.repository in (trusted_images)
  output: Untrusted image deployed in kube-system namespace (user=%ka.user.name image=%ka.req.pod.containers.image.repository resource=%ka.target.name)
  priority: WARNING
  source: k8s_audit
  tags: [k8s]
```

首先，我们定义了一个受信任的镜像列表，这些镜像将被允许部署到`kube-system`命名空间中。在规则中，我们使用了两个默认宏：`pod`和`kcreate`。 `pod`宏检查目标资源是否为 Pod，而`kcreate`检查动词是否为`create`。我们还检查目标命名空间是否为`kube-system`，并且部署的镜像不在`trusted_images`列表中。规则的`source`字段中的`k8s_audit`值表示此规则评估 Kubernetes 审计事件。然后，如果我们尝试在`kube-system`命名空间中部署`busybox`镜像的 Pod，我们将从 Falco 看到以下警报：

```
21:47:15.063915008: Warning Untrusted image deployed in kube-system namespace (user=admin image=busybox resource=pod-1)
```

请注意，为了使此规则起作用，需要将 Pod 创建的审计级别至少设置为“请求”级别，其中审计事件包括 Pod 的规范信息，例如镜像。

在本节中，我们介绍了 Falco，并向您展示了如何从系统调用和 Kubernetes 审计事件两个事件源创建 Falco 规则。这两个规则都用于基于工作负载或集群已知良性活动来检测异常活动。接下来，让我们谈谈如何在 Kubernetes 集群中进行取证工作。

# 使用 Sysdig Inspect 和 CRIU 进行取证。

在网络安全中，取证意味着收集、处理和分析信息，以支持漏洞缓解和/或欺诈、反情报或执法调查。您可以保存的数据越多，对收集的数据进行的分析越快，您就越能追踪攻击并更好地应对事件。在本节中，我们将向您展示如何使用 CRIU 和 Sysdig 开源工具来收集数据，然后介绍 Sysdig Inspect，这是一个用于分析 Sysdig 收集的数据的开源工具。

## 使用 CRIU 收集数据

**CRIU**是**Checkpoint and Restore In Userspace**的缩写。它是一个可以冻结运行中的容器并在磁盘上捕获容器状态的工具。稍后，可以将磁盘上保存的容器和应用程序数据恢复到冻结时的状态。它对于容器快照、迁移和远程调试非常有用。从安全的角度来看，它特别有用于捕获容器中正在进行的恶意活动（以便您可以在检查点后立即终止容器），然后在沙盒环境中恢复状态以进行进一步分析。

CRIU 作为 Docker 插件工作，仍处于实验阶段，已知问题是 CRIU 在最近的几个版本中无法正常工作（[`github.com/moby/moby/issues/37344`](https://github.com/moby/moby/issues/37344)）。出于演示目的，我使用了较旧的 Docker 版本（Docker CE 17.03），并将展示如何使用 CRIU 对运行中的容器进行检查点，并将状态恢复为新容器。

要启用 CRIU，您需要在 Docker 守护程序中启用`experimental`模式，如下所示：

```
echo "{\"experimental\":true}" >> /etc/docker/daemon.json
```

然后，在重新启动 Docker 守护程序后，您应该能够成功执行`docker checkpoint`命令，就像这样：

```
# docker checkpoint
Usage:	docker checkpoint COMMAND
Manage checkpoints
Options:
      --help   Print usage
Commands:
  create      Create a checkpoint from a running container
  ls          List checkpoints for a container
  rm          Remove a checkpoint
```

然后，按照说明安装 CRIU（[`criu.org/Installation`](https://criu.org/Installation)）。接下来，让我们看一个简单的示例，展示 CRIU 的强大之处。我有一个简单的`busybox`容器在运行，每秒增加`1`，如下面的代码片段所示：

```
# docker run -d --name looper --security-opt seccomp:unconfined busybox /bin/sh -c 'i=0; while true; do echo $i; i=$(expr $i + 1); sleep 1; done'
91d68fafec8fcf11e7699539dec0b037220b1fcc856fb7050c58ab90ae8cbd13
```

睡了几秒钟后，我看到计数器的输出在增加，如下所示：

```
# sleep 5
# docker logs looper
0
1
2
3
4
5
```

接下来，我想对容器进行检查点，并将状态存储到本地文件系统，就像这样：

```
# docker checkpoint create --checkpoint-dir=/tmp looper checkpoint
checkpoint
```

现在，`checkpoint`状态已保存在`/tmp`目录下。请注意，除非在创建检查点时指定了`--leave-running`标志，否则容器 looper 将在检查点后被杀死。

然后，创建一个镜像容器，但不运行它，就像这样：

```
# docker create --name looper-clone --security-opt seccomp:unconfined busybox /bin/sh -c 'i=0; while true; do echo $i; i=$(expr $i + 1); sleep 1; done'
49b9ade200e7da6bbb07057da02570347ad6fefbfc1499652ed286b874b59f2b
```

现在，我们可以启动具有存储状态的新`looper-clone`容器。让我们再等几秒钟，看看会发生什么。结果可以在下面的代码片段中看到：

```
# docker start --checkpoint-dir=/tmp --checkpoint=checkpoint looper-clone
# sleep 5
# docker logs looper-clone
6
7
8
9
10
```

新的`looper-clone`容器从`6`开始计数，这意味着状态（计数器为`5`）已成功恢复并使用。

CRIU 对容器取证非常有用，特别是当容器中发生可疑活动时。您可以对容器进行检查点（假设在集群中有多个副本运行），让 CRIU 杀死可疑容器，然后在沙盒环境中恢复容器的可疑状态以进行进一步分析。接下来，让我们谈谈另一种获取取证数据的方法。

## 使用 Sysdig 和 Sysdig Inspect

Sysdig 是一个用于 Linux 系统探索和故障排除的开源工具，支持容器。Sysdig 还可以用于通过在 Linux 内核中进行仪器化和捕获系统调用和其他操作系统事件来创建系统活动的跟踪文件。捕获功能使其成为容器化环境中的一种出色的取证工具。为了支持在 Kubernetes 集群中捕获系统调用，Sysdig 提供了一个`kubectl`插件，`kubectl-capture`，它使您可以像使用其他`kubectl`命令一样简单地捕获目标 pod 的系统调用。捕获完成后，可以使用强大的开源工具 Sysdig Inspect 进行故障排除和安全调查。

让我们继续以`insecure-nginx`为例，因为我们收到了 Falco 警报，如下面的代码片段所示：

```
08:22:19.484698397: Warning Anomalous file read activity in Nginx pod (user=<NA> process=nginx file=/etc/passwd container_id=439e2e739868 image=kaizheh/insecure-nginx) k8s.ns=insecure-nginx k8s.pod=insecure-nginx-7c99fdf44b-gffp4 container=439e2e739868 k8s.ns=insecure-nginx k8s.pod=insecure-nginx-7c99fdf44b-gffp4 container=439e2e739868
```

在触发警报时，`nginx` pod 仍然可能正在遭受攻击。您可以采取一些措施来应对。启动捕获，然后分析 Falco 警报的更多上下文是其中之一。

要触发捕获，请从[`github.com/sysdiglabs/kubectl-capture`](https://github.com/sysdiglabs/kubectl-capture)下载`kubectl-capture`并将其放置在其他`kubectl`插件中，就像这样：

```
$ kubectl plugin list
The following compatible plugins are available:
/Users/kaizhehuang/.krew/bin/kubectl-advise_psp
/Users/kaizhehuang/.krew/bin/kubectl-capture
/Users/kaizhehuang/.krew/bin/kubectl-ctx
/Users/kaizhehuang/.krew/bin/kubectl-krew
/Users/kaizhehuang/.krew/bin/kubectl-ns
/Users/kaizhehuang/.krew/bin/kubectl-sniff
```

然后，像这样在`nginx` pod 上启动捕获：

```
$ kubectl capture insecure-nginx-7c99fdf44b-4fl5s -ns insecure-nginx
Sysdig is starting to capture system calls:
Node: ip-172-20-42-49.ec2.internal
Pod: insecure-nginx-7c99fdf44b-4fl5s
Duration: 120 seconds
Parameters for Sysdig: -S -M 120 -pk -z -w /capture-insecure-nginx-7c99fdf44b-4fl5s-1587337260.scap.gz
The capture has been downloaded to your hard disk at:
/Users/kaizhehuang/demo/chapter11/sysdig/capture-insecure-nginx-7c99fdf44b-4fl5s-1587337260.scap.gz
```

在幕后，`kubectl-capture`在运行疑似受害者 pod 的主机上启动一个新的 pod 进行捕获，持续时间为`120`秒，这样我们就可以看到主机上正在发生的一切以及接下来`120`秒内的情况。捕获完成后，压缩的捕获文件将在当前工作目录中创建。您可以将 Sysdig Inspect 作为 Docker 容器引入，以开始安全调查，就像这样：

```
$ docker run -d -v /Users/kaizhehuang/demo/chapter11/sysdig:/captures -p3000:3000 sysdig/sysdig-inspect:latest
17533f98a947668814ac6189908ff003475b10f340d8f3239cd3627fa9747769
```

现在，登录到`http://localhost:3000`，您应该看到登录**用户界面**（**UI**）。记得解压`scap`文件，这样您就可以看到捕获文件的概述页面，如下所示：

![图 11.5 - Sysdig Inspect 概述](img/B15566_11_005.jpg)

图 11.5 - Sysdig Inspect 概述

Sysdig Inspect 从以下角度提供了对容器内发生活动的全面洞察：

+   执行的命令

+   文件访问

+   网络连接

+   系统调用

让我们不仅仅限于 Falco 警报进行更深入的挖掘。根据警报，我们可能怀疑这是一个文件路径遍历问题，因为是`nginx`进程访问`/etc/passwd`文件，我们知道这个 pod 只提供静态文件服务，所以`nginx`进程不应该访问`/usr/share/nginx/html/`目录之外的任何文件。现在，让我们看一下以下截图，看看发送给`nginx` pod 的网络请求是什么：

![图 11.6 – Sysdig Inspect 调查连接到 nginx 的网络连接](img/B15566_11_006.jpg)

图 11.6 – Sysdig Inspect 调查连接到 nginx 的网络连接

在查看连接后，我们发现请求来自单个 IP，`100.123.226.66`，看起来像是一个 pod IP。它可能来自同一个集群吗？在左侧面板上点击**Containers**视图，并在过滤器中指定`fd.cip=100.123.226.66`。然后，你会发现它来自`anchore-cli`容器，如下截图所示：

![图 11.7 – Sysdig Inspect 调查一个容器向 nginx 发送请求](img/B15566_11_007.jpg)

图 11.7 – Sysdig Inspect 调查一个容器向 nginx 发送请求

事实上，`anchore-cli` pod 碰巧运行在与`nginx` pod 相同的节点上，如下面的代码块所示：

```
$ kubectl get pods -o wide
NAME          READY   STATUS    RESTARTS   AGE   IP               NODE                           NOMINATED NODE   READINESS GATES
anchore-cli   1/1     Running   1          77m   100.123.226.66   ip-172-20-42-49.ec2.internal   <none>           <none>
$ kubectl get pods -n insecure-nginx -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP               NODE                           NOMINATED NODE   READINESS GATES
insecure-nginx-7c99fdf44b-4fl5s   1/1     Running   0          78m   100.123.226.65   ip-172-20-42-49.ec2.internal   <none>           <none>
```

现在我们知道可能有一些文件路径遍历攻击是从`anchore-cli` pod 发起的，让我们看看这是什么（只需在前面的**Sysdig Inspect**页面中双击条目），如下所示：

![图 11.8 – Sysdig Inspect 调查路径遍历攻击命令](img/B15566_11_008.jpg)

图 11.8 – Sysdig Inspect 调查路径遍历攻击命令

我们发现在`anchore-cli` pod 中执行了一系列文件路径遍历命令，详细如下：

+   使用 curl 命令访问 100.71.138.95 上的文件../etc/

+   使用 curl 命令访问 100.71.138.95 上的文件../

+   使用 curl 命令访问 100.71.138.95 上的文件../etc/passwd

+   使用 curl 命令访问 100.71.138.95 上的文件../etc/shadow

我们现在能够更接近攻击者了，下一步是尝试更深入地调查攻击者是如何进入`anchore-cli` pod 的。

CRIU 和 Sysdig 都是在容器化环境中进行取证的强大工具。希望 CRIU 问题能够很快得到解决。请注意，CRIU 还需要 Docker 守护程序以`experimental`模式运行，而 Sysdig 和 Sysdig Inspect 更多地在 Kubernetes 级别工作。Sysdig Inspect 提供了一个漂亮的用户界面，帮助浏览发生在 Pod 和容器中的不同活动。

# 总结

在这一长章中，我们涵盖了 Kubernetes 审计、Kubernetes 集群的高可用性、使用 Vault 管理秘密、使用 Falco 检测异常活动以及使用 CRIU 和 Sysdig 进行取证。虽然您可能会发现需要花费相当长的时间来熟悉所有的实践和工具，但深度防御是一个庞大的主题，值得深入研究安全性，这样您就可以为 Kubernetes 集群建立更强大的防护。

我们谈到的大多数工具都很容易安装和部署。我鼓励您尝试它们：添加自己的 Kubernetes 审计规则，使用 Vault 在 Kubernetes 集群中管理秘密，编写自己的 Falco 规则来检测异常行为，因为您比任何其他人都更了解您的集群，并使用 Sysdig 收集所有取证数据。一旦您熟悉了所有这些工具，您应该会对自己的 Kubernetes 集群更有信心。

在下一章中，我们将讨论一些已知的攻击，比如针对 Kubernetes 集群的加密挖矿攻击，看看我们如何利用本书中学到的技术来减轻这些攻击。

# 问题

1.  为什么我们不应该将审计级别设置为`Request`或`RequestResponse`用于秘密对象？

1.  在`kops`中用什么标志设置多个主节点？

1.  当 Vault 中的秘密更新时，侧车容器会做什么？

1.  Falco 使用哪些事件源？

1.  Falco 使用哪个过滤器从系统调用事件中检索进程名称？

1.  CRIU 对正在运行的容器有什么作用？

1.  您可以用 Sysdig Inspect 做什么？

# 更多参考资料

+   Kubernetes 审计：[`kubernetes.io/docs/tasks/debug-application-cluster/audit/`](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/)

+   使用`kubeadm`实现高可用性：[`kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/`](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)

+   Vault：[`www.vaultproject.io/docs/internals/architecture`](https://www.vaultproject.io/docs/internals/architecture)

+   Falco：https://falco.org/docs/

+   Sysdig 过滤：[`github.com/draios/sysdig/wiki/Sysdig-User-Guide#user-content-filtering`](https://github.com/draios/sysdig/wiki/Sysdig-User-Guide#user-content-filtering)

+   CRIU：[`criu.org/Docker`](https://criu.org/Docker)

+   Sysdig `kubectl-capture`：[`sysdig.com/blog/tracing-in-kubernetes-kubectl-capture-plugin/`](https://sysdig.com/blog/tracing-in-kubernetes-kubectl-capture-plugin/)

+   Sysdig Inspect：[`github.com/draios/sysdig-inspect`](https://github.com/draios/sysdig-inspect)

+   Sysdig：[`github.com/draios/sysdig`](https://github.com/draios/sysdig)

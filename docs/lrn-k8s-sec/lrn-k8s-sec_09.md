# 第七章：身份验证、授权和准入控制

身份验证和授权在保护应用程序中起着非常重要的作用。这两个术语经常被交替使用，但它们是非常不同的。身份验证验证用户的身份。一旦身份得到验证，授权就用来检查用户是否有执行所需操作的特权。身份验证使用用户知道的东西来验证他们的身份；在最简单的形式中，这是用户名和密码。一旦应用程序验证了用户的身份，它会检查用户可以访问哪些资源。在大多数情况下，这是访问控制列表的一个变体。用户的访问控制列表与请求属性进行比较，以允许或拒绝操作。

在本章中，我们将讨论请求在被`kube-apiserver`处理之前如何经过身份验证、授权模块和准入控制器的处理。我们将详细介绍不同模块和准入控制器的细节，并强调推荐的安全配置。

最后，我们将介绍**Open Policy Agent**（**OPA**），这是一个开源工具，可用于在微服务中实现授权。在 Kubernetes 中，我们将看看它如何作为一个验证准入控制器。许多集群需要比 Kubernetes 已提供的更细粒度的授权。使用 OPA，开发人员可以定义可以在运行时更新的自定义授权策略。有几个利用 OPA 的开源工具，比如 Istio。

在本章中，我们将讨论以下主题：

+   在 Kubernetes 中请求工作流

+   Kubernetes 身份验证

+   Kubernetes 授权

+   准入控制器

+   介绍 OPA

# 在 Kubernetes 中请求工作流

在 Kubernetes 中，`kube-apiserver`处理所有修改集群状态的请求。`kube-apiserver`首先验证请求的来源。它可以使用一个或多个身份验证模块，包括客户端证书、密码或令牌。请求依次从一个模块传递到另一个模块。如果请求没有被所有模块拒绝，它将被标记为匿名请求。API 服务器可以配置为允许匿名请求。

一旦请求的来源得到验证，它将通过授权模块，检查请求的来源是否被允许执行操作。授权模块允许请求，如果策略允许用户执行操作。Kubernetes 支持多个授权模块，如基于属性的访问控制（ABAC）、基于角色的访问控制（RBAC）和 webhooks。与认证模块类似，集群可以使用多个授权：

![图 7.1 - 在 kube-apiserver 处理之前进行请求解析](img/B15566_07_001.jpg)

图 7.1 - 在 kube-apiserver 处理之前进行请求解析

经过授权和认证模块后，准入控制器修改或拒绝请求。准入控制器拦截创建、更新或删除对象的请求。准入控制器分为两类：变异或验证。变异准入控制器首先运行；它们修改它们承认的请求。接下来运行验证准入控制器。这些控制器不能修改对象。如果任何准入控制器拒绝请求，将向用户返回错误，并且请求将不会被 API 服务器处理。

# Kubernetes 认证

Kubernetes 中的所有请求都来自外部用户、服务账户或 Kubernetes 组件。如果请求的来源未知，则被视为匿名请求。根据组件的配置，认证模块可以允许或拒绝匿名请求。在 v1.6+中，匿名访问被允许以支持匿名和未经认证的用户，用于 RBAC 和 ABAC 授权模式。可以通过向 API 服务器配置传递`--anonymous-auth=false`标志来明确禁用匿名访问：

```
$ps aux | grep api
root      3701  6.1  8.7 497408 346244 ?       Ssl  21:06   0:16 kube-apiserver --advertise-address=192.168.99.111 --allow-privileged=true --anonymous-auth=false
```

Kubernetes 使用一个或多个这些认证策略。让我们逐一讨论它们。

## 客户端证书

在 Kubernetes 中，使用 X509 证书颁发机构（CA）证书是最常见的认证策略。可以通过向服务器传递`--client-ca-file=file_path`来启用它。传递给 API 服务器的文件包含 CA 的列表，用于在集群中创建和验证客户端证书。证书中的“通用名称”属性通常用作请求的用户名，“组织”属性用于标识用户的组：

```
kube-apiserver --advertise-address=192.168.99.104 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/var/lib/minikube/certs/ca.crt
```

要创建新证书，需要执行以下步骤：

1.  生成私钥。可以使用`openssl`、`easyrsa`或`cfssl`生成私钥：

```
openssl genrsa -out priv.key 4096
```

1.  生成**证书签名请求**（**CSR**）。使用私钥和类似以下的配置文件生成 CSR。此 CSR 是为`test`用户生成的，该用户将成为`dev`组的一部分：

```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
[ dn ]
CN = test
O = dev
[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment extendedKeyUsage=serverAuth,clientAuth
```

您可以使用`openssl`生成 CSR：

```
openssl req -config ./csr.cnf -new -key priv.key -nodes -out new.csr
```

1.  签署 CSR。使用以下 YAML 文件创建一个 Kubernetes`CertificateSigningRequest`请求：

```
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
 name: mycsr
spec:
 groups:
 - system:authenticated
 request: ${BASE64_CSR}
 usages:
 - digital signature
 - key encipherment
 - server auth
 - client auth
```

之前生成的证书签名请求与前面的 YAML 规范一起使用，生成一个新的 Kubernetes 证书签名请求：

```
$ export BASE64_CSR=$(cat ./new.csr | base64 | tr -d '\n')
$ cat csr.yaml | envsubst | kubectl apply -f -
```

创建此请求后，需要由集群管理员批准以生成证书：

```
kubectl certificate approve mycsr
```

1.  导出 CRT。可以使用`kubectl`导出证书：

```
kubectl get csr mycsr -o jsonpath='{.status.certificate}' \
 | base64 --decode > new.crt
```

接下来，我们将看一下静态令牌，这是开发和调试环境中常用的身份验证模式，但不应在生产集群中使用。

## 静态令牌

API 服务器使用静态文件来读取令牌。将此静态文件传递给 API 服务器使用`--token-auth-file=<path>`。令牌文件是一个逗号分隔的文件，包括`secret`、`user`、`uid`、`group1`和`group2`。

令牌作为 HTTP 标头传递在请求中：

```
Authorization: Bearer 66e6a781-09cb-4e7e-8e13-34d78cb0dab6
```

令牌会持久存在，API 服务器需要重新启动以更新令牌。这*不*是一种推荐的身份验证策略。如果攻击者能够在集群中生成恶意 Pod，这些令牌很容易被破坏。一旦被破坏，生成新令牌的唯一方法是重新启动 API 服务器。

接下来，我们将看一下基本身份验证，这是静态令牌的一种变体，多年来一直作为 Web 服务的身份验证方法。

## 基本身份验证

与静态令牌类似，Kubernetes 还支持基本身份验证。可以通过使用`basic-auth-file=<path>`来启用。认证凭据存储在 CSV 文件中，包括`password`、`user`、`uid`、`group1`和`group2`。

用户名和密码作为认证标头传递在请求中：

```
Authentication: Basic base64(user:password)
```

与静态令牌类似，基本身份验证密码无法在不重新启动 API 服务器的情况下更改。不应在生产集群中使用基本身份验证。

## 引导令牌

引导令牌是静态令牌的一种改进。引导令牌是 Kubernetes 中默认使用的身份验证方法。它们是动态管理的，并存储为`kube-system`中的秘密。要启用引导令牌，请执行以下操作：

1.  在 API 服务器中使用`--enable-bootstrap-token-auth`来启用引导令牌验证器：

```
$ps aux | grep api
root      3701  3.8  8.8 497920 347140 ?       Ssl  21:06   4:58 kube-apiserver --advertise-address=192.168.99.111 --allow-privileged=true --anonymous-auth=true --authorization-mode=Node,RBAC --client-ca-file=/var/lib/minikube/certs/ca.crt --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota --enable-bootstrap-token-auth=true
```

1.  使用`controller`标志在控制器管理器中启用`tokencleaner`：

```
$ ps aux | grep controller
root      3693  1.4  2.3 211196 94396 ?        Ssl  21:06   1:55 kube-controller-manager --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf --bind-address=127.0.0.1 --client-ca-file=/var/lib/minikube/certs/ca.crt --cluster-name=mk --cluster-signing-cert-file=/var/lib/minikube/certs/ca.crt --cluster-signing-key-file=/var/lib/minikube/certs/ca.key --controllers=*,bootstrapsigner,tokencleaner
```

1.  与令牌身份验证类似，引导令牌作为请求中的 HTTP 头传递：

```
Authorization: Bearer 123456.aa1234fdeffeeedf
```

令牌的第一部分是`TokenId`值，第二部分是`TokenSecret`值。`TokenController`确保从系统秘密中删除过期的令牌。

## 服务账户令牌

服务账户验证器会自动启用。它验证签名的持有者令牌。签名密钥是使用`--service-account-key-file`指定的。如果未指定此值，则将使用 Kube API 服务器的私钥：

```
$ps aux | grep api
root      3711 27.1 14.9 426728 296552 ?       Ssl  04:22   0:04 kube-apiserver --advertise-address=192.168.99.104 ... --secure-port=8443 --service-account-key-file=/var/lib/minikube/certs/sa.pub --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/var/lib/minikube/certs/apiserver.crt --tls-private-key-file=/var/lib/minikube/certs/apiserver.key
docker    4496  0.0  0.0  11408   544 pts/0    S+   04:22   0:00 grep api
```

服务账户由`kube-apiserver`创建，并与 pod 关联。这类似于 AWS 中的实例配置文件。如果未指定服务账户，则默认服务账户将与 pod 关联。

要创建一个名为 test 的服务账户，您可以使用以下命令：

```
kubectl create serviceaccount test 
```

服务账户有关联的秘密，其中包括 API 服务器的 CA 和签名令牌：

```
$ kubectl get serviceaccounts test -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2020-03-29T04:35:58Z"
  name: test
  namespace: default
  resourceVersion: "954754"
  selfLink: /api/v1/namespaces/default/serviceaccounts/test
  uid: 026466f3-e2e8-4b26-994d-ee473b2f36cd
secrets:
- name: test-token-sdq2d
```

如果我们列举细节，我们可以看到证书和令牌：

```
$ kubectl get secret test-token-sdq2d -o yaml
apiVersion: v1
data:
  ca.crt: base64(crt)
  namespace: ZGVmYXVsdA==
  token: base64(token)
kind: Secret
```

接下来，我们将讨论 webhook 令牌。一些企业拥有远程身份验证和授权服务器，通常在所有服务中使用。在 Kubernetes 中，开发人员可以使用 webhook 令牌来利用远程服务进行身份验证。

## Webhook 令牌

在 webhook 模式下，Kubernetes 会调用集群外的 REST API 来确定用户的身份。可以通过向 API 服务器传递`--authorization-webhook-config-file=<path>`来启用身份验证的 webhook 模式。

以下是 webhook 配置的示例。在此示例中，[authn.example.com/authenticate](http://authn.example.com/authenticate)用作 Kubernetes 集群的身份验证端点：

```
clusters:
  - name: name-of-remote-authn-service
    cluster:
      certificate-authority: /path/to/ca.pem
      server: https://authn.example.com/authenticate
```

让我们看看另一种远程服务可以用于身份验证的方式。

## 身份验证代理

`kube-apiserver`可以配置为使用`X-Remote`请求头标识用户。您可以通过向 API 服务器添加以下参数来启用此方法：

```
--requestheader-username-headers=X-Remote-User
--requestheader-group-headers=X-Remote-Group
--requestheader-extra-headers-prefix=X-Remote-Extra-
```

每个请求都有以下标头来识别它们：

```
GET / HTTP/1.1
X-Remote-User: foo
X-Remote-Group: bar
X-Remote-Extra-Scopes: profile
```

API 代理使用 CA 验证请求。

## 用户冒充

集群管理员和开发人员可以使用用户冒充来调试新用户的身份验证和授权策略。要使用用户冒充，用户必须被授予冒充特权。API 服务器使用以下标头来冒充用户：

+   `冒充-用户`

+   `冒充-组`

+   `冒充-额外-*`

一旦 API 服务器接收到冒充标头，API 服务器会验证用户是否经过身份验证并具有冒充特权。如果是，则请求将以冒充用户的身份执行。`kubectl`可以使用`--as`和`--as-group`标志来冒充用户：

```
kubectl apply -f pod.yaml --as=dev-user --as-group=system:dev
```

一旦身份验证模块验证了用户的身份，它们会解析请求以检查用户是否被允许访问或修改请求。

# Kubernetes 授权

授权确定请求是否允许或拒绝。一旦确定请求的来源，活动授权模块会评估请求的属性与用户的授权策略，以允许或拒绝请求。每个请求依次通过授权模块，如果任何模块提供允许或拒绝的决定，它将自动被接受或拒绝。

## 请求属性

授权模块解析请求中的一组属性，以确定请求是否应该被解析、允许或拒绝：

+   **用户**：请求的发起者。这在身份验证期间进行验证。

+   **组**：用户所属的组。这是在身份验证层中提供的。

+   **API**：请求的目的地。

+   **请求动词**：请求的类型，可以是`GET`、`CREATE`、`PATCH`、`DELETE`等。

+   **资源**：正在访问的资源的 ID 或名称。

+   **命名空间**：正在访问的资源的命名空间。

+   **请求路径**：如果请求是针对非资源端点的，则使用路径来检查用户是否被允许访问端点。这对于`api`和`healthz`端点是正确的。

现在，让我们看看使用这些请求属性来确定请求发起者是否被允许发起请求的不同授权模式。

## 授权模式

现在，让我们看看 Kubernetes 中可用的不同授权模式。

## 节点

节点授权模式授予 kubelet 访问节点的服务、端点、节点、Pod、秘密和持久卷的权限。kubelet 被识别为`system:nodes`组的一部分，用户名为`system:node:<name>`，由节点授权者授权。这种模式在 Kubernetes 中默认启用。

`NodeRestriction`准入控制器与节点授权者一起使用，我们将在本章后面学习，以确保 kubelet 只能修改其正在运行的节点上的对象。API 服务器使用`--authorization-mode=Node`标志来使用节点授权模块：

```
$ps aux | grep api
root      3701  6.1  8.7 497408 346244 ?       Ssl  21:06   0:16 kube-apiserver --advertise-address=192.168.99.111 --allow-privileged=true --anonymous-auth=true --authorization-mode=Node,RBAC --client-ca-file=/var/lib/minikube/certs/ca.crt --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota
```

节点授权与 ABAC 或 RBAC 一起使用，接下来我们将看一下。

## ABAC

使用 ABAC，通过验证请求的属性来允许请求。可以通过在 API 服务器中使用`--authorization-policy-file=<path>`和`--authorization-mode=ABAC`来启用 ABAC 授权模式。

策略包括每行一个 JSON 对象。每个策略包括以下内容：

+   **版本**：策略格式的 API 版本。

+   **种类**：`Policy`字符串用于策略。

+   **规范**：包括用户、组和资源属性，如`apiGroup`、`namespace`和`nonResourcePath`（如`/version`、`/apis`、`readonly`），以允许不修改资源的请求。

一个示例策略如下：

```
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "kubelet", "namespace": "*", "resource": "pods", "readonly": true}} 
```

此策略允许 kubelet 读取任何 Pod。ABAC 难以配置和维护。不建议在生产环境中使用 ABAC。

## RBAC

使用 RBAC，通过分配给用户的角色来规范对资源的访问。自 v1.8 以来，RBAC 在许多集群中默认启用。要启用 RBAC，请使用`--authorization-mode=RBAC`启动 API 服务器：

```
$ ps aux | grep api
root     14632  9.2 17.0 495148 338780 ?       Ssl  06:11   0:09 kube-apiserver --advertise-address=192.168.99.104 --allow-privileged=true --authorization-mode=Node,RBAC ...
```

RBAC 使用 Role，这是一组权限，以及 RoleBinding，它向用户授予权限。Role 和 RoleBinding 受到命名空间的限制。如果角色需要跨命名空间，则可以使用 ClusterRole 和 ClusterRoleBinding 来向用户授予权限。

以下是允许用户在默认命名空间中创建和修改 Pod 的`Role`属性示例：

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: default
  name: deployment-manager
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

相应的`RoleBinding`可以与`Role`一起使用，向用户授予权限：

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: binding
  namespace: default
subjects:
- kind: User
  name: employee
  apiGroup: ""
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: ""
```

一旦应用了`RoleBinding`，您可以切换上下文查看是否工作正常：

```
$ kubectl --context=employee-context get pods
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-677b9cfc6b-xks5f   1/1     Running   0          12m
```

但是，如果尝试查看部署，将导致错误：

```
$ kubectl --context=employee-context get deployments
Error from server (Forbidden): deployments.apps is forbidden: User "employee" cannot list resource "deployments" in API group "apps" in the namespace "default"
```

由于角色和角色绑定受限于默认命名空间，访问不同命名空间中的 Pod 将导致错误：

```
$ kubectl --context=employee-context get pods -n test
Error from server (Forbidden): pods is forbidden: User "test" cannot list resource "pods" in API group "" in the namespace "test"
$ kubectl --context=employee-context get pods -n kube-system
Error from server (Forbidden): pods is forbidden: User "test" cannot list resource "pods" in API group "" in the namespace "kube-system"
```

接下来，我们将讨论 webhooks，它为企业提供了使用远程服务器进行授权的能力。

## Webhooks

类似于用于身份验证的 webhook 模式，用于授权的 webhook 模式使用远程 API 服务器来检查用户权限。可以通过使用`--authorization-webhook-config-file=<path>`来启用 webhook 模式。

让我们看一个示例 webhook 配置文件，将[`authz.remote`](https://authz.remote)设置为 Kubernetes 集群的远程授权端点：

```
clusters:
  - name: authz_service
    cluster:
      certificate-authority: ca.pem
      server: https://authz.remote/
```

一旦请求通过了认证和授权模块，准入控制器就会处理请求。让我们详细讨论准入控制器。

# 准入控制器

准入控制器是在请求经过认证和授权后拦截 API 服务器的模块。控制器在修改集群中对象的状态之前验证和改变请求。控制器可以是改变和验证的。如果任何控制器拒绝请求，请求将立即被丢弃，并向用户返回错误，以便请求不会被处理。

可以通过使用`--enable-admission-plugins`标志来启用准入控制器：

```
$ps aux | grep api
root      3460 17.0  8.6 496896 339432 ?       Ssl  06:53   0:09 kube-apiserver --advertise-address=192.168.99.106 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/var/lib/minikube/certs/ca.crt --enable-admission-plugins=PodSecurityPolicy,NamespaceLifecycle,LimitRanger --enable-bootstrap-token-auth=true
```

可以使用`--disable-admission-plugins`标志来禁用默认的准入控制器。

在接下来的章节中，我们将看一些重要的准入控制器。

### AlwaysAdmit

此准入控制器允许所有的 Pod 存在于集群中。自 1.13 版本以来，该控制器已被弃用，不应在任何集群中使用。使用此控制器，集群的行为就好像集群中不存在任何控制器一样。

## AlwaysPullImages

该控制器确保新的 Pod 始终强制拉取镜像。这有助于确保 Pod 使用更新的镜像。它还确保只有有权限访问的用户才能使用私有镜像，因为没有访问权限的用户在启动新的 Pod 时无法拉取镜像。应该在您的集群中启用此控制器。

## 事件速率限制

拒绝服务攻击在基础设施中很常见。行为不端的对象也可能导致资源的高消耗，如 CPU 或网络，从而导致成本增加或可用性降低。`EventRateLimit`用于防止这些情况发生。

限制是使用配置文件指定的，可以通过向 API 服务器添加 `--admission-control-config-file` 标志来指定。

集群可以有四种类型的限制：`Namespace`、`Server`、`User` 和 `SourceAndObject`。对于每个限制，用户可以拥有 **每秒查询** (**QPS**)、突发和缓存大小的最大限制。

让我们看一个配置文件的例子：

```
limits:
- type: Namespace
  qps: 50
  burst: 100
  cacheSize: 200
- type: Server
  qps: 10
  burst: 50
  cacheSize: 200
```

这将为所有 API 服务器和命名空间添加 `qps`、`burst` 和 `cacheSize` 限制。

接下来，我们将讨论 LimitRanger，它可以防止集群中可用资源的过度利用。

## LimitRanger

这个准入控制器观察传入的请求，并确保它不违反 `LimitRange` 对象中指定的任何限制。

一个 `LimitRange` 对象的例子如下：

```
apiVersion: "v1"
kind: "LimitRange"
metadata:
  name: "pod-example" 
spec:
  limits:
    - type: "Pod"
      max:
        memory: "128Mi"
```

有了这个限制范围对象，任何请求内存超过 128 Mi 的 pod 都将失败：

```
pods "range-demo" is forbidden maximum memory usage per Pod is 128Mi, but limit is 1073741824
```

在使用 LimitRanger 时，恶意的 pod 无法消耗过多的资源。

## NodeRestriction

这个准入控制器限制了 kubelet 可以修改的 pod 和节点。有了这个准入控制器，kubelet 以 `system:node:<name>` 格式获得一个用户名，并且只能修改自己节点上运行的节点对象和 pod。

## PersistentVolumeClaimResize

这个准入控制器为 `PersistentVolumeClaimResize` 请求添加了验证。

## PodSecurityPolicy

这个准入控制器在创建或修改 pod 时运行，以确定是否应该基于 pod 的安全敏感配置来运行 pod。策略中的一组条件将与工作负载配置进行检查，以验证是否应该允许工作负载创建请求。PodSecurityPolicy 可以检查诸如 `privileged`、`allowHostPaths`、`defaultAddCapabilities` 等字段。您将在下一章中了解更多关于 PodSecurityPolicy 的内容。

## SecurityContextDeny

如果未启用 PodSecurityPolicy，则建议使用此准入控制器。它限制了安全敏感字段的设置，这可能会导致特权升级，例如运行特权 pod 或向容器添加 Linux 功能：

```
$ ps aux | grep api
root      3763  6.7  8.7 497344 345404 ?       Ssl  23:28   0:14 kube-apiserver --advertise-address=192.168.99.112 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/var/lib/minikube/certs/ca.crt --enable-admission-plugins=SecurityContextDeny
```

建议默认情况下在集群中启用 PodSecurityPolicy。但是，由于管理开销，可以在为集群配置 PodSecurityPolicy 之前使用 `SecurityContextDeny`。

## ServiceAccount

`ServiceAccount`是 pod 的身份。这个准入控制器实现了`ServiceAccount`；如果集群使用服务账户，应该使用它。

## MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook

类似于用于身份验证和授权的 webhook 配置，webhook 可以用作准入控制器。MutatingAdmissionWebhook 修改工作负载的规范。这些钩子按顺序执行。ValidatingAdmissionWebhook 解析传入的请求以验证其是否正确。验证钩子同时执行。

现在，我们已经了解了 Kubernetes 中资源的身份验证、授权和准入控制。让我们看看开发人员如何在他们的集群中实现细粒度的访问控制。在下一节中，我们将讨论 OPA，这是一个在生产集群中广泛使用的开源工具。

# OPA 简介

**OPA**是一个开源的策略引擎，允许在 Kubernetes 中执行策略。许多开源项目，如 Istio，利用 OPA 提供更精细的控制。OPA 是由**Cloud Native Computing Foundation** (**CNCF**)托管的孵化项目。

OPA 部署为与其他服务一起的服务。为了做出授权决策，微服务调用 OPA 来决定请求是否应该被允许或拒绝。授权决策被卸载到 OPA，但这种执行需要由服务本身实现。在 Kubernetes 环境中，它经常被用作验证 webhook：

![图 7.2 - 开放策略代理](img/B15566_07_002.jpg)

图 7.2 - 开放策略代理

为了做出策略决策，OPA 需要以下内容：

+   **集群信息**：集群的状态。集群中可用的对象和资源对于 OPA 来说是重要的，以便决定是否应该允许请求。

+   **输入查询**：策略代理分析请求的参数，以允许或拒绝请求。

+   **策略**：策略定义了解析集群信息和输入查询以返回决策的逻辑。OPA 的策略是用一种称为 Rego 的自定义语言定义的。

让我们看一个例子，说明如何利用 OPA 来拒绝创建带有`busybox`镜像的 pod。您可以使用官方的 OPA 文档([`www.openpolicyagent.org/docs/latest/kubernetes-tutorial/`](https://www.openpolicyagent.org/docs/latest/kubernetes-tutorial/))在您的集群上安装 OPA。

以下是限制使用`busybox`镜像创建和更新 pod 的策略：

```
$ cat pod-blacklist.rego
package kubernetes.admission
import data.kubernetes.namespaces
operations = {"CREATE", "UPDATE"}
deny[msg] {
	input.request.kind.kind == "Pod"
	operations[input.request.operation]
	image := input.request.object.spec.containers[_].image
	image == "busybox"
	msg := sprintf("image not allowed %q", [image])
}
```

要应用此策略，您可以使用以下内容：

```
kubectl create configmap pod —from-file=pod-blacklist.rego
```

一旦创建了`configmap`，`kube-mgmt`会从`configmap`中加载这些策略，在`opa`容器中，`kube-mgmt`和`opa`容器都在`opa` pod 中。现在，如果您尝试使用`busybox`镜像创建一个 pod，您将得到以下结果：

```
$ cat busybox.yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: sec-ctx-demo
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
```

该策略检查对`busybox`镜像名称的请求，并拒绝使用`busybox`镜像创建带有`image not allowed`错误的 pod：

```
admission webhook "validating-webhook.openpolicyagent.org" denied the request: image not allowed "busybox"
```

类似于我们之前讨论过的准入控制器，可以使用 OPA 在 Kubernetes 集群中创建进一步细粒度的准入控制器。

# 总结

在本章中，我们讨论了在 Kubernetes 中进行身份验证和授权的重要性。我们讨论了可用于身份验证和授权的不同模块，并详细讨论了这些模块，以及详细介绍了每个模块的使用示例。在讨论身份验证时，我们讨论了用户模拟，这可以由集群管理员或开发人员用来测试权限。接下来，我们谈到了准入控制器，它可以用于在身份验证和授权之后验证或改变请求。我们还详细讨论了一些准入控制器。最后，我们看了一下 OPA，它可以在 Kubernetes 集群中执行更细粒度的授权。

现在，您应该能够为您的集群制定适当的身份验证和授权策略。您应该能够确定哪些准入控制器适用于您的环境。在许多情况下，您将需要更细粒度的授权控制，这可以通过使用 OPA 来实现。

在下一章中，我们将深入探讨保护 pod。本章将更详细地涵盖我们在本章中涵盖的一些主题，如 PodSecurityPolicy。保护 pod 对于保护 Kubernetes 中的应用部署至关重要。

# 问题

1.  哪些授权模块不应该在集群中使用？

1.  集群管理员如何测试对新用户授予的权限？

1.  哪些授权模式适合生产集群？

1.  `EventRateLimit`和`LimitRange`准入控制器之间有什么区别？

1.  您能否编写一个 Rego 策略来拒绝创建带有`test.example`端点的 ingress？

# 进一步阅读

您可以参考以下链接获取更多信息：

+   准入控制器: [`kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do)

+   OPA: [`www.openpolicyagent.org/docs/latest/`](https://www.openpolicyagent.org/docs/latest/)

+   Kubernetes RBAC: [`rbac.dev/`](https://rbac.dev/)

+   audit2RBAC: [`github.com/liggitt/audit2rbac`](https://github.com/liggitt/audit2rbac)

+   KubiScan: [`github.com/cyberark/KubiScan`](https://github.com/cyberark/KubiScan)

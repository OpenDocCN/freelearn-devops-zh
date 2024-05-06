# *第六章*：保护集群组件

在之前的章节中，我们看过了 Kubernetes 集群的架构。一个 Kubernetes 集群包括主要组件，包括`kube-apiserver`、`etcd`、`kube-scheduler`、CoreDNS、`kube-controller-manager`和`cloud-controller-manager`，以及节点组件，包括`kubelet`、`kube-proxy`和`container-runtime`。主要组件负责集群管理，它们构成了集群的控制平面。另一方面，节点组件负责节点上 pod 和容器的运行。

在*第三章*中，*威胁建模*，我们简要讨论了 Kubernetes 集群中的组件需要进行配置以确保集群的安全。任何集群组件的妥协都可能导致数据泄露。环境的错误配置是传统或微服务环境中数据泄露的主要原因之一。了解每个组件的配置以及每个设置如何打开新的攻击面是很重要的。因此，集群管理员了解不同的配置是很重要的。

在本章中，我们将详细讨论如何保护集群中的每个组件。在许多情况下，可能无法遵循所有安全最佳实践，但重要的是要强调风险，并制定一套缓解策略，以防攻击者试图利用易受攻击的配置。

对于每个主要和节点组件，我们简要讨论了 Kubernetes 集群中具有安全相关配置的组件的功能，并详细查看了每个配置。我们查看了这些配置的可能设置，并强调了推荐的最佳实践。最后，我们介绍了`kube-bench`，并演示了如何使用它来评估您集群的安全姿势。

在本章中，我们将涵盖以下主题：

+   保护 kube-apiserver

+   保护 kubelet

+   保护 etcd

+   保护 kube-scheduler

+   保护 kube-controller-manager

+   保护 CoreDNS

+   对集群的安全配置进行基准测试

# 保护 kube-apiserver

`kube-apiserver`是您集群的网关。它实现了**表述状态转移**（**REST**）**应用程序编程接口**（**API**）来授权和验证对象的请求。它是与 Kubernetes 集群内的其他组件进行通信和管理的中央网关。它执行三个主要功能：

+   **API 管理**：`kube-apiserver`公开用于集群管理的 API。开发人员和集群管理员使用这些 API 来修改集群的状态。

+   **请求处理**：对于对象管理和集群管理的请求进行验证和处理。

+   **内部消息传递**：API 服务器与集群中的其他组件进行交互，以确保集群正常运行。

API 服务器的请求在处理之前经过以下步骤：

1.  **身份验证**：`kube-apiserver`首先验证请求的来源。`kube-apiserver`支持多种身份验证模式，包括客户端证书、持有者令牌和**超文本传输协议**（**HTTP**）身份验证。

1.  **授权**：一旦验证了请求来源的身份，API 服务器会验证该来源是否被允许执行请求。`kube-apiserver`默认支持**基于属性的访问控制**（**ABAC**）、**基于角色的访问控制**（**RBAC**）、节点授权和用于授权的 Webhooks。RBAC 是推荐的授权模式。

1.  **准入控制器**：一旦`kube-apiserver`验证并授权请求，准入控制器会解析请求，以检查其是否在集群内允许。如果任何准入控制器拒绝请求，则该请求将被丢弃。

`kube-apiserver`是集群的大脑。API 服务器的妥协会导致集群的妥协，因此确保 API 服务器安全至关重要。Kubernetes 提供了大量设置来配置 API 服务器。让我们接下来看一些与安全相关的配置。

为了保护 API 服务器，您应该执行以下操作：

+   **禁用匿名身份验证**：使用`anonymous-auth=false`标志将匿名身份验证设置为`false`。这可以确保被所有身份验证模块拒绝的请求不被视为匿名并被丢弃。

+   **禁用基本身份验证**：基本身份验证在`kube-apiserver`中为方便起见而受支持，不应使用。基本身份验证密码会持续存在。`kube-apiserver`使用`--basic-auth-file`参数来启用基本身份验证。确保不使用此参数。

+   **禁用令牌认证**：`--token-auth-file`启用集群的基于令牌的认证。不建议使用基于令牌的认证。静态令牌会永久存在，并且需要重新启动 API 服务器才能更新。应该使用客户端证书进行认证。

+   **确保与 kubelet 的连接使用 HTTPS**：默认情况下，`--kubelet-https`设置为`true`。确保对于`kube-apiserver`，不要将此参数设置为`false`。

+   **禁用分析**：使用`--profiling`启用分析会暴露不必要的系统和程序细节。除非遇到性能问题，否则通过设置`--profiling=false`来禁用分析。

+   **禁用 AlwaysAdmit**：`--enable-admission-plugins`可用于启用默认未启用的准入控制插件。`AlwaysAdmit`接受请求。确保该插件不在`--enabled-admission-plugins`列表中。

+   **使用 AlwaysPullImages**：`AlwaysPullImages`准入控制确保节点上的镜像在没有正确凭据的情况下无法使用。这可以防止恶意 Pod 为节点上已存在的镜像创建容器。

+   **使用 SecurityContextDeny**：如果未启用`PodSecurityPolicy`，应使用此准入控制器。`SecurityContextDeny`确保 Pod 无法修改`SecurityContext`以提升特权。

+   **启用审计**：审计在`kube-apiserver`中默认启用。确保`--audit-log-path`设置为安全位置的文件。此外，确保审计的`maxage`、`maxsize`和`maxbackup`参数设置满足合规性要求。

+   **禁用 AlwaysAllow 授权**：授权模式确保具有正确权限的用户的请求由 API 服务器解析。不要在`--authorization-mode`中使用`AlwaysAllow`。

+   **启用 RBAC 授权**：RBAC 是 API 服务器的推荐授权模式。ABAC 难以使用和管理。RBAC 角色和角色绑定的易用性和易于更新使其适用于经常扩展的环境。

+   **确保对 kubelet 的请求使用有效证书**：默认情况下，`kube-apiserver`对`kubelet`的请求使用 HTTPS。启用`--kubelet-certificate-authority`、`--kubelet-client-key`和`--kubelet-client-key`确保通信使用有效的 HTTPS 证书。

+   **启用 service-account-lookup**：除了确保服务账户令牌有效外，`kube-apiserver`还应验证令牌是否存在于`etcd`中。确保`--service-account-lookup`未设置为`false`。

+   **启用 PodSecurityPolicy**：`--enable-admission-plugins`可用于启用`PodSecurityPolicy`。正如我们在[*第五章*]（B15566_05_Final_ASB_ePub.xhtml#_idTextAnchor144）中所看到的，*配置 Kubernetes 安全边界*，`PodSecurityPolicy`用于定义 pod 的安全敏感标准。我们将在[*第八章*]（B15566_08_Final_ASB_ePub.xhtml#_idTextAnchor249）中深入探讨创建 pod 安全策略。

+   **使用服务账户密钥文件**：使用`--service-account-key-file`可以启用对服务账户密钥的轮换。如果未指定此选项，`kube-apiserver`将使用**传输层安全性**（**TLS**）证书的私钥来签署服务账户令牌。

+   **启用对 etcd 的授权请求**：`--etcd-certfile`和`--etcd-keyfile`可用于标识对`etcd`的请求。这可以确保`etcd`可以拒绝任何未经识别的请求。

+   **不要禁用 ServiceAccount 准入控制器**：此准入控制自动化服务账户。启用`ServiceAccount`可确保可以将具有受限权限的自定义`ServiceAccount`与不同的 Kubernetes 对象一起使用。

+   **不要为请求使用自签名证书**：如果为`kube-apiserver`启用了 HTTPS，则应提供`--tls-cert-file`和`--tls-private-key-file`，以确保不使用自签名证书。

+   **连接到 etcd 的安全连接**：设置`--etcd-cafile`允许`kube-apiserver`使用证书文件通过**安全套接字层**（**SSL**）向`etcd`验证自身。

+   **使用安全的 TLS 连接**：将`--tls-cipher-suites`设置为仅使用强密码。`--tls-min-version`用于设置最低支持的 TLS 版本。TLS 1.2 是推荐的最低版本。

+   **启用高级审计**：通过将`--feature-gates`设置为`AdvancedAuditing=false`可以禁用高级审计。确保此字段存在并设置为`true`。高级审计有助于调查是否发生违规行为。

在 Minikube 上，`kube-apiserver`的配置如下：

```
$ps aux | grep kube-api
root      4016  6.1 17.2 495148 342896 ?       Ssl  01:03   0:16 kube-apiserver --advertise-address=192.168.99.100 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/var/lib/minikube/certs/ca.crt --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota --enable-bootstrap-token-auth=true --etcd-cafile=/var/lib/minikube/certs/etcd/ca.crt --etcd-certfile=/var/lib/minikube/certs/apiserver-etcd-client.crt --etcd-keyfile=/var/lib/minikube/certs/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --insecure-port=0 --kubelet-client-certificate=/var/lib/minikube/certs/apiserver-kubelet-client.crt --kubelet-client-key=/var/lib/minikube/certs/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/var/lib/minikube/certs/front-proxy-client.crt --proxy-client-key-file=/var/lib/minikube/certs/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/var/lib/minikube/certs/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=8443 --service-account-key-file=/var/lib/minikube/certs/sa.pub --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/var/lib/minikube/certs/apiserver.crt --tls-private-key-file=/var/lib/minikube/certs/apiserver.key
```

正如您所看到的，默认情况下，在 Minikube 上，`kube-apiserver`并未遵循所有安全最佳实践。例如，默认情况下未启用`PodSecurityPolicy`，也未设置强密码套件和`tls`最低版本。集群管理员有责任确保 API 服务器的安全配置。

# 保护 kubelet

`kubelet`是 Kubernetes 的节点代理。它管理 Kubernetes 集群中对象的生命周期，并确保节点上的对象处于健康状态。

要保护`kubelet`，您应该执行以下操作：

+   **禁用匿名身份验证**：如果启用了匿名身份验证，则被其他身份验证方法拒绝的请求将被视为匿名。确保为每个`kubelet`实例设置`--anonymous-auth=false`。

+   **设置授权模式**：使用配置文件设置`kubelet`的授权模式。可以使用`--config`参数指定配置文件。确保授权模式列表中没有`AlwaysAllow`。

+   **轮换 kubelet 证书**：可以使用`kubelet`配置文件中的`RotateCertificates`配置来轮换`kubelet`证书。这应与`RotateKubeletServerCertificate`一起使用，以自动请求轮换服务器证书。

+   **提供证书颁发机构（CA）包**：`kubelet`使用 CA 包来验证客户端证书。可以使用配置文件中的`ClientCAFile`参数进行设置。

+   **禁用只读端口**：默认情况下，`kubelet`启用了只读端口，应该禁用。只读端口没有身份验证或授权。

+   **启用 NodeRestriction 准入控制器**：`NodeRestriction`准入控制器仅允许`kubelet`修改其绑定的节点上的节点和 Pod 对象。

+   **限制对 Kubelet API 的访问**：只有`kube-apiserver`组件与`kubelet` API 交互。如果尝试在节点上与`kubelet` API 通信，将被禁止。这是通过为`kubelet`使用 RBAC 来确保的。

在 Minikube 上，`kubelet`配置如下：

```
root      4286  2.6  4.6 1345544 92420 ?       Ssl  01:03   0:18 /var/lib/minikube/binaries/v1.17.3/kubelet --authorization-mode=Webhook --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --cgroup-driver=cgroupfs --client-ca-file=/var/lib/minikube/certs/ca.crt --cluster-domain=cluster.local --config=/var/lib/kubelet/config.yaml --container-runtime=docker --fail-swap-on=false --hostname-override=minikube --kubeconfig=/etc/kubernetes/kubelet.conf --node-ip=192.168.99.100 --pod-manifest-path=/etc/kubernetes/manifests
```

与 API 服务器类似，默认情况下，`kubelet`上并非所有安全配置都被使用，例如禁用只读端口。接下来，我们将讨论集群管理员如何保护`etcd`。

# 保护 etcd

`etcd`是 Kubernetes 用于数据存储的键值存储。它存储了 Kubernetes 集群的状态、配置和秘密。只有`kube-apiserver`应该可以访问`etcd`。`etcd`的泄露可能导致集群泄露。

为了保护`etcd`，您应该执行以下操作：

+   **限制节点访问**：使用 Linux 防火墙确保只允许需要访问`etcd`的节点访问。

+   **确保 API 服务器使用 TLS**：`--cert-file`和`--key-file`确保对`etcd`的请求是安全的。

+   **使用有效证书**：`--client-cert-auth`确保客户端通信使用有效证书，并将`--auto-tls`设置为`false`确保不使用自签名证书。

+   **加密静态数据**：将`--encryption-provider-config`传递给 API 服务器，以确保在`etcd`中对静态数据进行加密。

在 Minikube 上，`etcd`配置如下：

```
$ ps aux | grep etcd
root      3992  1.9  2.4 10612080 48680 ?      Ssl  01:03   0:18 etcd --advertise-client-urls=https://192.168.99.100:2379 --cert-file=/var/lib/minikube/certs/etcd/server.crt --client-cert-auth=true --data-dir=/var/lib/minikube/etcd --initial-advertise-peer-urls=https://192.168.99.100:2380 --initial-cluster=minikube=https://192.168.99.100:2380 --key-file=/var/lib/minikube/certs/etcd/server.key --listen-client-urls=https://127.0.0.1:2379,https://192.168.99.100:2379 --listen-metrics-urls=http://127.0.0.1:2381 --listen-peer-urls=https://192.168.99.100:2380 --name=minikube --peer-cert-file=/var/lib/minikube/certs/etcd/peer.crt --peer-client-cert-auth=true --peer-key-file=/var/lib/minikube/certs/etcd/peer.key --peer-trusted-ca-file=/var/lib/minikube/certs/etcd/ca.crt --snapshot-count=10000 --trusted-ca-file=/var/lib/minikube/certs/etcd/ca.crt
```

`etcd`存储着 Kubernetes 集群的敏感数据，如私钥和秘密。`etcd`的泄露就意味着`api-server`组件的泄露。集群管理员在设置`etcd`时应特别注意。

# 保护 kube-scheduler

接下来，我们来看看`kube-scheduler`。正如我们在*第一章*中已经讨论过的，*Kubernetes 架构*，`kube-scheduler`负责为 pod 分配节点。一旦 pod 分配给节点，`kubelet`就会执行该 pod。`kube-scheduler`首先过滤可以运行 pod 的节点集，然后根据每个节点的评分，将 pod 分配给评分最高的过滤节点。`kube-scheduler`组件的泄露会影响集群中 pod 的性能和可用性。

为了保护`kube-scheduler`，您应该执行以下操作：

+   **禁用分析**：对`kube-scheduler`的分析会暴露系统细节。将`--profiling`设置为`false`可以减少攻击面。

+   **禁用 kube-scheduler 的外部连接**：应禁用`kube-scheduler`的外部连接。将`AllowExtTrafficLocalEndpoints`设置为`true`会启用`kube-scheduler`的外部连接。确保使用`--feature-gates`禁用此功能。

+   **启用 AppArmor**：默认情况下，`kube-scheduler`启用了`AppArmor`。确保不要禁用`kube-scheduler`的`AppArmor`。

在 Minikube 上，`kube-scheduler`配置如下：

```
$ps aux | grep kube-scheduler
root      3939  0.5  2.0 144308 41640 ?        Ssl  01:03   0:02 kube-scheduler --authentication-kubeconfig=/etc/kubernetes/scheduler.conf --authorization-kubeconfig=/etc/kubernetes/scheduler.conf --bind-address=0.0.0.0 --kubeconfig=/etc/kubernetes/scheduler.conf --leader-elect=true
```

与`kube-apiserver`类似，调度程序也没有遵循所有的安全最佳实践，比如禁用分析。

# 保护 kube-controller-manager

`kube-controller-manager`管理集群的控制循环。它通过 API 服务器监视集群的更改，并旨在将集群从当前状态移动到期望的状态。`kube-controller-manager`默认提供多个控制器管理器，如复制控制器和命名空间控制器。对`kube-controller-manager`的妥协可能导致对集群的更新被拒绝。

要保护`kube-controller-manager`，您应该使用`--use-service-account-credentials`，与 RBAC 一起使用可以确保控制循环以最低特权运行。

在 Minikube 上，`kube-controller-manager`的配置如下：

```
$ps aux | grep kube-controller-manager
root      3927  1.8  4.5 209520 90072 ?        Ssl  01:03   0:11 kube-controller-manager --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf --bind-address=0.0.0.0 --client-ca-file=/var/lib/minikube/certs/ca.crt --cluster-signing-cert-file=/var/lib/minikube/certs/ca.crt --cluster-signing-key-file=/var/lib/minikube/certs/ca.key --controllers=*,bootstrapsigner,tokencleaner --kubeconfig=/etc/kubernetes/controller-manager.conf --leader-elect=true --requestheader-client-ca-file=/var/lib/minikube/certs/front-proxy-ca.crt --root-ca-file=/var/lib/minikube/certs/ca.crt --service-account-private-key-file=/var/lib/minikube/certs/sa.key --use-service-account-credentials=true
```

接下来，让我们谈谈如何保护 CoreDNS。

# 保护 CoreDNS

`kube-dns`是 Kubernetes 集群的默认**域名系统**（**DNS**）服务器。DNS 服务器帮助内部对象（如服务、pod 和容器）相互定位。`kube-dns`由三个容器组成，详细如下：

+   `kube-dns`：此容器使用 SkyDNS 执行 DNS 解析服务。

+   `dnsmasq`：轻量级 DNS 解析器。它从 SkyDNS 缓存响应。

+   `sidecar`：这个监视健康并处理 DNS 的度量报告。

自 1.11 版本以来，`kube-dns`已被 CoreDNS 取代，因为 dnsmasq 存在安全漏洞，SkyDNS 存在性能问题。CoreDNS 是一个单一容器，提供了`kube-dns`的所有功能。

要编辑 CoreDNS 的配置文件，您可以使用`kubectl`，就像这样：

```
$ kubectl -n kube-system edit configmap coredns
```

在 Minikube 上，默认的 CoreDNS 配置文件如下：

```
# Please edit the object below. Lines beginning with a '#' 
# will be ignored, and an empty file will abort the edit. 
# If an error occurs while saving this file will be
# reopened with the relevant failures.
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

要保护 CoreDNS，请执行以下操作：

+   **确保健康插件未被禁用**：`health`插件监视 CoreDNS 的状态。它用于确认 CoreDNS 是否正在运行。通过在`Corefile`中添加`health`来启用它。

+   **为 CoreDNS 启用 istio**：`istio`是 Kubernetes 使用的服务网格，用于提供服务发现、负载平衡和认证。它在 Kubernetes 中默认不可用，需要作为外部依赖项添加。您可以通过启动`istio`服务并将`istio`服务的代理添加到配置文件中来向集群添加`istio`，就像这样：

```
global:53 {
         errors
         proxy . {cluster IP of this istio-core-dns service}
    }
```

现在我们已经查看了集群组件的不同配置，重要的是要意识到随着组件变得更加复杂，将会添加更多的配置参数。集群管理员不可能记住这些配置。因此，接下来，我们将讨论一种帮助集群管理员监视集群组件安全状况的工具。

# 对集群安全配置进行基准测试

**互联网安全中心**（**CIS**）发布了一份 Kubernetes 基准，可以供集群管理员使用，以确保集群遵循推荐的安全配置。发布的 Kubernetes 基准超过 200 页。

`kube-bench`是一个用 Go 编写并由 Aqua Security 发布的自动化工具，运行 CIS 基准中记录的测试。这些测试是用**YAML Ain't Markup Language**（**YAML**）编写的，使其易于演变。

`kube-bench`可以直接在节点上使用`kube-bench`二进制文件运行，如下所示：

```
$kube-bench node --benchmark cis-1.4
```

对于托管在`gke`、`eks`和`aks`上的集群，`kube-bench`作为一个 pod 运行。一旦 pod 运行完成，您可以查看日志以查看结果，如下面的代码块所示：

```
$ kubectl apply -f job-gke.yaml
$ kubectl get pods
NAME               READY   STATUS      RESTARTS   AGE
kube-bench-2plpm   0/1     Completed   0          5m20s
$ kubectl logs kube-bench-2plpm
[INFO] 4 Worker Node Security Configuration
[INFO] 4.1 Worker Node Configuration Files
[WARN] 4.1.1 Ensure that the kubelet service file permissions are set to 644 or more restrictive (Not Scored)
[WARN] 4.1.2 Ensure that the kubelet service file ownership is set to root:root (Not Scored)
[PASS] 4.1.3 Ensure that the proxy kubeconfig file permissions are set to 644 or more restrictive (Scored)
[PASS] 4.1.4 Ensure that the proxy kubeconfig file ownership is set to root:root (Scored)
[WARN] 4.1.5 Ensure that the kubelet.conf file permissions are set to 644 or more restrictive (Not Scored)
[WARN] 4.1.6 Ensure that the kubelet.conf file ownership is set to root:root (Not Scored)
[WARN] 4.1.7 Ensure that the certificate authorities file permissions are set to 644 or more restrictive (Not Scored)
......
== Summary ==
0 checks PASS
0 checks FAIL
37 checks WARN
0 checks INFO
```

重要的是要调查具有`FAIL`状态的检查。您应该力求没有失败的检查。如果由于任何原因这是不可能的，您应该制定一个针对失败检查的风险缓解计划。

`kube-bench`是一个有用的工具，用于监视遵循安全最佳实践的集群组件。建议根据自己的环境添加/修改`kube-bench`规则。大多数开发人员在启动新集群时运行`kube-bench`，但定期运行它以监视集群组件是否安全很重要。

# 总结

在本章中，我们查看了每个主节点和节点组件的不同安全敏感配置：`kube-apiserver`、`kube-scheduler`、`kube-controller-manager`、`kubelet`、CoreDNS 和`etcd`。我们了解了如何保护每个组件。默认情况下，组件可能不遵循所有安全最佳实践，因此集群管理员有责任确保组件是安全的。最后，我们看了一下`kube-bench`，它可以用来了解正在运行的集群的安全基线。

重要的是要了解这些配置，并确保组件遵循这些检查表，以减少受到威胁的机会。

在下一章中，我们将介绍 Kubernetes 中的身份验证和授权机制。在本章中，我们简要讨论了一些准入控制器。我们将深入探讨不同的准入控制器，并最终讨论它们如何被利用以提供更精细的访问控制。

# 问题

1.  什么是基于令牌的身份验证？

1.  什么是`NodeRestriction`准入控制器？

1.  如何确保数据在`etcd`中处于加密状态？

1.  为什么 CoreDNS 取代了`kube-dns`？

1.  如何在**弹性 Kubernetes 服务**（**EKS**）集群上使用`kube-bench`？

# 进一步阅读

您可以参考以下链接，了解本章涵盖的主题的更多信息：

+   CIS 基准：[`www.cisecurity.org/benchmark/kubernetes/`](https://www.cisecurity.org/benchmark/kubernetes/)

+   GitHub（`kube-bench`）：[`github.com/aquasecurity/kube-bench`](https://github.com/aquasecurity/kube-bench)

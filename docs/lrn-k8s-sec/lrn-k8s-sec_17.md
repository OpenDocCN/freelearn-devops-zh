# 第十四章：评估

# 第一章

1.  扩展、运营成本和更长的发布周期。

1.  主要组件运行在主节点上。这些组件负责管理工作节点。主要组件包括`kube-apiserver`、`etcd`、`kube-scheduler`、`kube-controller-manager`、`cloud-controller-manager`和`dns-server`。

1.  Kubernetes 部署帮助根据标签和选择器扩展/缩小 Pod。部署封装了副本集和 Pod。部署的 YAML 规范包括 Pod 的实例数量和`template`，它与 Pod 规范相同。

1.  OpenShift、K3S 和 Minikube。

1.  Kubernetes 环境具有高度可配置性，并由众多组件组成。可配置性和复杂性与不安全的默认设置是一个令人担忧的原因。此外，集群中主要组件的 compromis 是引起违规的最简单方式。

# 第二章

1.  Pod。

1.  网络命名空间和 IPC 命名空间。

1.  用于保存其他容器的网络命名空间的占位符。

1.  ClusterIP、NodePort、LoadBalancer 和 ExternalName。

1.  Ingress 支持第 7 层路由，并且不需要来自云提供商的额外负载均衡器，而 LoadBalancer 服务需要每个服务一个负载均衡器。

# 第三章

1.  威胁建模是一个迭代的过程，从设计阶段开始。

1.  最终用户、内部攻击者和特权攻击者。

1.  存储在`etcd`中的未加密数据。

1.  Kubernetes 环境的复杂性增加了在 Kubernetes 环境中使用威胁建模应用程序的难度。

1.  Kubernetes 引入了与应用程序的额外资产和交互。这增加了 Kubernetes 中应用程序的复杂性，增加了攻击面。

# 第四章

1.  `Role`对象包含由动词和资源组成的规则，指示命名空间中资源的操作特权。

1.  `RoleBinding`对象将命名空间中的`Role`对象与一组主体（例如`User`和`ServiceAccount`）链接起来。它用于将 Role 对象中定义的特权授予主体。

1.  `RoleBinding`表示主体拥有的特权在`RoleBinding`对象的命名空间中有效。`ClusterRoleBinding`表示主体拥有的特权在整个集群中有效。

1.  `hostPID`、`hostNetwork`和`hostIPC`。

1.  为具有出口规则的 Pod 创建网络策略。

# 第五章

1.  主要组件、工作组件和 Kubernetes 对象。

1.  Pod、service/Ingress、`api-server`、节点和命名空间。

1.  RBAC 和网络策略。

1.  Pod 中的进程可以访问主机 PID 命名空间，查看工作节点上运行的所有进程。

[PRE0]

# 第六章

1.  基于令牌的身份验证使静态令牌能够用于识别集群中请求的来源。静态令牌无法在不重新启动 API 服务器的情况下进行更新，因此不应使用。

1.  `NodeRestriction`准入控制器确保 kubelet 只能修改其正在运行的节点的节点和 Pod 对象。

1.  将`--encryption-provider-config`传递给 API 服务器，以确保在`etcd`中对数据进行加密。

1.  `dnsmasq`中的安全漏洞，SkyDNS 中的性能问题，以及使用单个容器而不是三个容器来提供相同功能的`kube-dns`。

1.  您可以在 EKS 集群上使用`kube-bench`如下：

**$ git clone :** https://github.com/aquasecurity/kube-bench** $ kubectl apply -f job-eks.yaml**

# 第七章

1.  在生产集群中不应使用静态令牌和基本身份验证。这些模块使用静态凭据，需要重新启动 API 服务器才能更新。

1.  集群管理员可以使用用户模拟特权来测试授予新用户的权限。使用`kubectl`，集群管理员可以使用`--as --as-group`标志以不同的用户身份运行请求。

1.  Kubernetes 中默认启用了 Node 和 RBAC。应该使用这些。如果集群使用远程 API 进行授权，则应改用 Webhook 模式。

1.  `EventRateLimit`准入控制器指定 API 服务器可以处理的请求的最大限制。另一方面，LimitRanger 确保 Kubernetes 对象遵守`LimitRange`对象指定的资源限制。

1.  拒绝使用`rego`策略创建具有`test.example`端点的 Ingress 如下：

[PRE1]

# 第八章

1.  定义一个命令，要求 Docker 引擎定期检查容器的健康状态。

1.  `COPY`指令只能将文件从构建机器复制到镜像的文件系统，而`ADD`指令不仅可以从本地主机复制文件，还可以从远程 URL 检索文件到镜像的文件系统。使用`ADD`可能会引入从互联网添加恶意文件的风险。

1.  `CAP_NET_BIND_SERVICE`。

1.  将`runAsNonRoot`设置为`true`，kubelet 将阻止以 root 用户身份运行容器。

1.  创建具有特权的角色，使用`PodSecurityPolicy`对象，并创建`rolebinding`对象将角色分配给工作负载使用的服务账户。

# 第九章

1.  `Docker history <image name>`.

1.  7-8.9.

1.  `anchore-cli image add <image name>`.

1.  `anchore-cli image vuln <image name> all`.

1.  `anchore-cli evaluate check <image digets> --tag <image full tag>`.

1.  它有助于识别具有最新已知漏洞的图像。

# 第十章

1.  资源请求指定 Kubernetes 对象保证获得的资源，而限制指定 Kubernetes 对象可以使用的最大资源。

1.  限制内存为 500 mi 的资源配额如下：

[PRE2]

1.  LimitRanger 是一个实施 LimitRanges 的准入控制器。LimitRange 定义了 Kubernetes 资源的约束。限制范围可以应用于 Pod、容器或`persistantvolumeclaim`。命名空间资源配额类似于`LimitRange`，但对整个命名空间进行强制执行。

1.  服务账户令牌。

1.  Prometheus 和 Grafana。

# 第十一章

1.  秘密数据将记录在 Kubernetes 审计日志中。

1.  `--master-zones`.

1.  将更新的秘密同步到 Pod 的挂载卷。

1.  系统调用和 Kubernetes 审计事件。

1.  `proc.name`.

1.  检查运行中的容器，稍后可以在隔离环境中恢复。

1.  故障排除和安全调查。

# 第十二章

1.  仪表板在未启用身份验证的情况下使用。

1.  不要运行仪表板，或者为仪表板启用身份验证。

1.  不。这可能是加密挖矿攻击，但也可能是由其他原因引起的，比如应用程序错误。

1.  加密挖矿二进制文件使用 HTTP 或 HTTPS 协议连接到挖矿池服务器，而不是 stratum。

1.  Kubernetes 集群的配置、构建、部署和运行时。

# 第十三章

1.  集群管理员跟踪 CVE ID，以确保 Kubernetes 集群不容易受到已知的公开问题的影响。安全研究人员研究参考部分，以了解问题的技术细节，以开发 CVE 的缓解措施。最后，攻击者研究参考部分，以找到未修补的变体或使用类似技术来发现代码其他部分的问题。

1.  客户端问题经常导致数据外泄或客户端上的代码执行。构建机器或集群管理员的机器通常包含敏感数据，对这些机器的攻击可能会对组织产生重大经济影响。

1.  `api-server`上的 DoS 问题可能导致整个集群的可用性中断。

1.  未经身份验证的 DoS 问题比经过身份验证的 DoS 问题更严重。理想情况下，未经身份验证的用户不应该能够与`api-server`通信。如果未经身份验证的用户能够发送请求并导致`api-server`的 DoS 问题，那比经过身份验证的用户更糟糕。经过身份验证的 DoS 请求也非常严重，因为集群中的配置错误可能允许未经身份验证的用户提升权限并成为经过身份验证的用户。

1.  Kubernetes 的安全公告和通知是了解任何新公开已知漏洞的好方法。这些公告和通知相当嘈杂，管理员很容易忽略重要问题。定期运行`kube-hunter`有助于集群管理员识别管理员可能忽略的任何已知问题。

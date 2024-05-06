# *第十四章*：服务网格和无服务器

本章讨论了高级 Kubernetes 模式。首先，它详细介绍了时髦的服务网格模式，其中通过 sidecar 代理处理可观察性和服务到服务的发现，以及设置流行的服务网格 Istio 的指南。最后，它描述了无服务器模式以及如何在 Kubernetes 中应用它。本章的主要案例研究将包括为示例应用程序和服务发现设置 Istio，以及 Istio 入口网关。

让我们从讨论 sidecar 代理开始，它为服务网格的服务到服务连接性奠定了基础。

在本章中，我们将涵盖以下主题：

+   使用 sidecar 代理

+   向 Kubernetes 添加服务网格

+   在 Kubernetes 上实现无服务器

# 技术要求

为了运行本章中详细介绍的命令，您需要一台支持`kubectl`命令行工具的计算机，以及一个可用的 Kubernetes 集群。请参阅*第一章*，*与 Kubernetes 通信*，了解快速启动和运行 Kubernetes 的几种方法，以及如何安装`kubectl`工具的说明。

本章中使用的代码可以在书的 GitHub 存储库中找到，网址为[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter14`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter14)。

# 使用 sidecar 代理

正如我们在本书中早些时候提到的，sidecar 是一种模式，其中一个 Pod 包含另一个容器，除了要运行的实际应用程序容器。这个额外的“额外”容器就是 sidecar。Sidecar 可以用于许多不同的原因。一些最常用的 sidecar 用途是监控、日志记录和代理。

对于日志记录，一个 sidecar 容器可以从应用容器中获取应用程序日志（因为它们可以共享卷并在本地通信），然后将日志发送到集中式日志堆栈，或者解析它们以进行警报。监控也是类似的情况，sidecar Pod 可以跟踪并发送有关应用程序 Pod 的指标。

使用侧车代理时，当请求进入 Pod 时，它们首先进入代理容器，然后路由请求（在记录或执行其他过滤之后）到应用程序容器。同样，当请求离开应用程序容器时，它们首先进入代理，代理可以提供 Pod 的路由。

通常，诸如 NGINX 之类的代理侧车只为进入 Pod 的请求提供代理。然而，在服务网格模式中，进入和离开 Pod 的请求都通过代理，这为服务网格模式本身提供了基础。

请参考以下图表，了解侧车代理如何与应用程序容器交互：

![图 14.1 - 代理侧车](img/B14790_14_001.jpg)

图 14.1 - 代理侧车

正如您所看到的，侧车代理负责将请求路由到 Pod 中的应用程序容器，并允许功能，如服务路由、记录和过滤。

侧车代理模式是一种替代基于 DaemonSet 的代理，其中每个节点上的代理 Pod 处理对该节点上其他 Pod 的代理。Kubernetes 代理本身类似于 DaemonSet 模式。使用侧车代理可以提供比使用 DaemonSet 代理更灵活的灵活性，但性能效率会有所降低，因为需要运行许多额外的容器。

一些用于 Kubernetes 的流行代理选项包括以下内容：

+   *NGINX*

+   *HAProxy*

+   *Envoy*

虽然 NGINX 和 HAProxy 是更传统的代理，但 Envoy 是专门为分布式、云原生环境构建的。因此，Envoy 构成了流行的服务网格和为 Kubernetes 构建的 API 网关的核心。

在我们讨论 Envoy 之前，让我们讨论安装其他代理作为侧车的方法。

## 使用 NGINX 作为侧车反向代理

在我们指定 NGINX 如何作为侧车代理之前，值得注意的是，在即将发布的 Kubernetes 版本中，侧车将成为一个 Kubernetes 资源类型，它将允许轻松地向大量 Pod 注入侧车容器。然而，目前侧车容器必须在 Pod 或控制器（ReplicaSet、Deployment 等）级别指定。

让我们看看如何使用以下部署 YAML 配置 NGINX 作为侧车，我们暂时不会创建。这个过程比使用 NGINX Ingress Controller 要手动一些。

出于空间原因，我们将 YAML 分成两部分，并删除了一些冗余内容，但您可以在代码存储库中完整地看到它。让我们从部署的容器规范开始：

Nginx-sidecar.yaml：

```
   spec:
     containers:
     - name: myapp
       image: ravirdv/http-responder:latest
       imagePullPolicy: IfNotPresent
     - name: nginx-sidecar
       image: nginx
       imagePullPolicy: IfNotPresent
       volumeMounts:
         - name: secrets
           mountPath: /app/cert
         - name: config
           mountPath: /etc/nginx/nginx.conf
           subPath: nginx.conf
```

正如您所看到的，我们指定了两个容器，即我们的主应用程序容器`myapp`和`nginx` sidecar，我们通过卷挂载注入了一些配置，以及一些 TLS 证书。

接下来，让我们看看同一文件中的`volumes`规范，我们在其中注入了一些证书（来自一个密钥）和`config`（来自`ConfigMap`）：

```
    volumes:
     - name: secrets
       secret:
         secretName: nginx-certificates
         items:
           - key: server-cert
             path: server.pem
           - key: server-key
             path: server-key.pem
     - name: config
       configMap:
         name: nginx-configuration
```

正如您所看到的，我们需要一个证书和一个密钥。

接下来，我们需要使用`ConfigMap`创建 NGINX 配置。NGINX 配置如下：

nginx.conf：

```
http {
    sendfile        on;
    include       mime.types;
    default_type  application/octet-stream;
    keepalive_timeout  80;
    server {
       ssl_certificate      /app/cert/server.pem;
      ssl_certificate_key  /app/cert/server-key.pem;
      ssl_protocols TLSv1.2;
      ssl_ciphers EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:!EECDH+3DES:!RSA+3DES:!MD5;
      ssl_prefer_server_ciphers on;
      listen       443 ssl;
      server_name  localhost;
      location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_pass http://127.0.0.1:5000/;
      }
    }
}
worker_processes  1;
events {
    worker_connections  1024;
}
```

正如您所看到的，我们有一些基本的 NGINX 配置。重要的是，我们有`proxy_pass`字段，它将请求代理到`127.0.0.1`上的端口，或者本地主机。由于 Pod 中的容器可以共享本地主机端口，这充当了我们的 sidecar 代理。出于本书的目的，我们不会审查所有其他行，但是请查看 NGINX 文档，了解每行的更多信息（[`nginx.org/en/docs/`](https://nginx.org/en/docs/)）。

现在，让我们从这个文件创建`ConfigMap`。使用以下命令来命令式地创建`ConfigMap`：

```
kubectl create cm nginx-configuration --from-file=nginx.conf=./nginx.conf
```

这将导致以下输出：

```
Configmap "nginx-configuration" created
```

接下来，让我们为 NGINX 创建 TLS 证书，并将它们嵌入到 Kubernetes 密钥中。您需要安装 CFSSL（CloudFlare 的 PKI/TLS 开源工具包）库才能按照这些说明进行操作，但您也可以使用任何其他方法来创建您的证书。

首先，我们需要创建**证书颁发机构**（**CA**）。从 CA 的 JSON 配置开始：

nginxca.json：

```
{
   "CN": "mydomain.com",
   "hosts": [
       "mydomain.com",
       "www.mydomain.com"
   ],
   "key": {
       "algo": "rsa",
       "size": 2048
   },
   "names": [
       {
           "C": "US",
           "ST": "MD",
           "L": "United States"
       }
   ]
}
```

现在，使用 CFSSL 创建 CA 证书：

```
cfssl gencert -initca nginxca.json | cfssljson -bare nginxca
```

接下来，我们需要 CA 配置：

Nginxca-config.json：

```
{
  "signing": {
      "default": {
          "expiry": "20000h"
      },
      "profiles": {
          "client": {
              "expiry": "43800h",
              "usages": [
                  "signing",
                  "key encipherment",
                  "client auth"
              ]
          },
          "server": {
              "expiry": "20000h",
              "usages": [
                  "signing",
                  "key encipherment",
                  "server auth",
                  "client auth"
              ]
          }
      }
  }
}
```

我们还需要一个证书请求配置：

Nginxcarequest.json：

```
{
  "CN": "server",
  "hosts": [
    ""
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
```

现在，我们实际上可以创建我们的证书了！使用以下命令：

```
cfssl gencert -ca=nginxca.pem -ca-key=nginxca-key.pem -config=nginxca-config.json -profile=server -hostname="127.0.0.1" nginxcarequest.json | cfssljson -bare server
```

作为证书密钥的最后一步，通过最后一个`cfssl`命令从证书文件的输出创建 Kubernetes 密钥：

```
kubectl create secret generic nginx-certs --from-file=server-cert=./server.pem --from-file=server-key=./server-key.pem
```

现在，我们终于可以创建我们的部署了：

```
kubectl apply -f nginx-sidecar.yaml 
```

这将产生以下输出：

```
deployment "myapp" created
```

为了检查 NGINX 代理功能，让我们创建一个服务来指向我们的部署：

Nginx-sidecar-service.yaml：

```
apiVersion: v1
kind: Service
metadata:
 name:myapp
 labels:
   app: myapp
spec:
 selector:
   app: myapp
 type: NodePort
 ports:
 - port: 443
   targetPort: 443
   protocol: TCP
   name: https
```

现在，使用`https`访问集群中的任何节点应该会导致一个正常工作的 HTTPS 连接！但是，由于我们的证书是自签名的，浏览器将显示一个*不安全*的消息。

现在您已经看到了 NGINX 如何与 Kubernetes 一起作为边车代理使用，让我们转向更现代的、云原生的代理边车 - Envoy。

## 使用 Envoy 作为边车代理

Envoy 是为云原生环境构建的现代代理。在我们稍后将审查的 Istio 服务网格中，Envoy 充当反向和正向代理。然而，在我们进入 Istio 之前，让我们尝试部署 Envoy 作为代理。

我们将告诉 Envoy 在哪里路由各种请求，使用路由、监听器、集群和端点。这个功能是 Istio 的核心，我们将在本章后面进行审查。

让我们逐个查看 Envoy 配置的每个部分，看看它是如何工作的。

### Envoy 监听器

Envoy 允许配置一个或多个监听器。对于每个监听器，我们指定 Envoy 要监听的端口，以及我们想要应用到监听器的任何过滤器。

过滤器可以提供复杂的功能，包括缓存、授权、**跨源资源共享**（**CORS**）配置等。Envoy 支持将多个过滤器链接在一起。

### Envoy 路由

某些过滤器具有路由配置，指定应接受请求的域、路由匹配和转发规则。

### Envoy 集群

Envoy 中的集群表示可以根据监听器中的路由将请求路由到的逻辑服务。在云原生环境中，集群可能包含多个可能的 IP 地址，因此它支持负载均衡配置，如*轮询*。

### Envoy 端点

最后，在集群中指定端点作为服务的一个逻辑实例。Envoy 支持从 API 获取端点列表（这基本上是 Istio 服务网格中发生的事情），并在它们之间进行负载均衡。

在 Kubernetes 上的生产 Envoy 部署中，很可能会使用某种形式的动态、API 驱动的 Envoy 配置。Envoy 的这个特性称为 xDS，并被 Istio 使用。此外，还有其他开源产品和解决方案使用 Envoy 与 xDS，包括 Ambassador API 网关。

在本书中，我们将查看一些静态（非动态）的 Envoy 配置；这样，我们可以分解配置的每个部分，当我们审查 Istio 时，您将对一切是如何工作有一个很好的理解。

现在让我们深入研究一个 Envoy 配置，用于设置一个单个 Pod 需要能够将请求路由到两个服务，*Service 1*和*Service 2*。设置如下：

![图 14.2-出站 envoy 代理](img/B14790_14_002.jpg)

图 14.2-出站 envoy 代理

如您所见，我们应用 Pod 中的 Envoy sidecar 将配置为路由到两个上游服务，*Service 1*和*Service 2*。两个服务都有两个可能的端点。

在 Envoy xDS 的动态设置中，端点的 Pod IPs 将从 API 中加载，但是为了我们的审查目的，我们将在端点中显示静态的 Pod IPs。我们将完全忽略 Kubernetes 服务，而是直接访问 Pod IPs 以进行轮询配置。在服务网格场景中，Envoy 也将部署在所有目标 Pod 上，但现在我们将保持简单。

现在，让我们看看如何在 Envoy 配置 YAML 中配置这个网络映射（您可以在代码存储库中找到完整的配置）。这当然与 Kubernetes 资源 YAML 非常不同-我们将在稍后讨论这一部分。整个配置涉及大量的 YAML，所以让我们一点一点地来。

### 理解 Envoy 配置文件

首先，让我们看看我们配置的前几行-关于我们的 Envoy 设置的一些基本信息。

Envoy-configuration.yaml:

```
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8001
```

如您所见，我们为 Envoy 的`admin`指定了一个端口和地址。与以下配置一样，我们将 Envoy 作为一个 sidecar 运行，因此地址将始终是本地的- `0.0.0.0`。接下来，我们用一个 HTTPS 监听器开始我们的监听器列表：

```
static_resources:
  listeners:
   - address:
      socket_address:
        address: 0.0.0.0
        port_value: 8443
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          stat_prefix: ingress_https
          codec_type: auto
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/service/1"
                route:
                  cluster: service1
              - match:
                  prefix: "/service/2"
                route:
                  cluster: service2
          http_filters:
          - name: envoy.filters.http.router
            typed_config: {}
```

如您所见，对于每个 Envoy 监听器，我们有一个本地地址和端口（此监听器是一个 HTTPS 监听器）。然后，我们有一个过滤器列表-尽管在这种情况下，我们只有一个。每个 envoy 过滤器类型的配置略有不同，我们不会逐行审查它（请查看 Envoy 文档以获取更多信息[`www.envoyproxy.io/docs`](https://www.envoyproxy.io/docs)），但这个特定的过滤器匹配两个路由，`/service/1`和`/service/2`，并将它们路由到两个 envoy 集群。在我们的 YAML 的第一个 HTTPS 监听器部分下，我们有 TLS 配置，包括证书：

```
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          common_tls_context:
            tls_certificates:
              certificate_chain:
                inline_string: |
                   <INLINE CERT FILE>
              private_key:
                inline_string: |
                  <INLINE PRIVATE KEY FILE>
```

如您所见，此配置传递了`private_key`和`certificate_chain`。接下来，我们有第二个也是最后一个监听器，一个 HTTP 监听器：

```
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 8080
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          codec_type: auto
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/service1"
                route:
                  cluster: service1
              - match:
                  prefix: "/service2"
                route:
                  cluster: service2
          http_filters:
          - name: envoy.filters.http.router
            typed_config: {}
```

如您所见，这个配置与我们的 HTTPS 监听器的配置非常相似，只是它监听不同的端口，并且不包括证书信息。接下来，我们进入我们的集群配置。在我们的情况下，我们有两个集群，一个用于`service1`，一个用于`service2`。首先是`service1`：

```
  clusters:
  - name: service1
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: round_robin
    http2_protocol_options: {}
    load_assignment:
      cluster_name: service1
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: service1
                port_value: 5000
```

接下来，`Service 2`：

```
  - name: service2
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: round_robin
    http2_protocol_options: {}
    load_assignment:
      cluster_name: service2
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: service2
                port_value: 5000
```

对于这些集群中的每一个，我们指定请求应该路由到哪里，以及到哪个端口。例如，对于我们的第一个集群，请求被路由到`http://service1:5000`。我们还指定了负载均衡策略（在这种情况下是轮询）和连接的超时时间。现在我们有了我们的 Envoy 配置，我们可以继续创建我们的 Kubernetes Pod，并注入我们的 sidecar 以及 envoy 配置。我们还将把这个文件分成两部分，因为它有点太大了，以至于难以理解：

Envoy-sidecar-deployment.yaml：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: my-service
    spec:
      containers:
      - name: envoy
        image: envoyproxy/envoy:latest
        ports:
          - containerPort: 9901
            protocol: TCP
            name: envoy-admin
          - containerPort: 8786
            protocol: TCP
            name: envoy-web
```

如您所见，这是一个典型的部署 YAML。在这种情况下，我们实际上有两个容器。首先是 Envoy 代理容器（或边车）。它监听两个端口。接下来，继续向下移动 YAML，我们为第一个容器进行了卷挂载（用于保存 Envoy 配置），以及一个启动命令和参数：

```
        volumeMounts:
          - name: envoy-config-volume
            mountPath: /etc/envoy-config/
        command: ["/usr/local/bin/envoy"]
        args: ["-c", "/etc/envoy-config/config.yaml", "--v2-config-only", "-l", "info","--service-cluster","myservice","--service-node","myservice", "--log-format", "[METADATA][%Y-%m-%d %T.%e][%t][%l][%n] %v"]
```

最后，我们有我们 Pod 中的第二个容器，这是一个应用容器：

```
- name: my-service
        image: ravirdv/http-responder:latest
        ports:
        - containerPort: 5000
          name: svc-port
          protocol: TCP
      volumes:
        - name: envoy-config-volume
          configMap:
            name: envoy-config
            items:
              - key: envoy-config
                path: config.yaml
```

如您所见，这个应用在端口`5000`上响应。最后，我们还有我们的 Pod 级别卷定义，以匹配 Envoy 容器中挂载的 Envoy 配置卷。在创建部署之前，我们需要创建一个带有我们的 Envoy 配置的`ConfigMap`。我们可以使用以下命令来做到这一点：

```
kubectl create cm envoy-config 
--from-file=config.yaml=./envoy-config.yaml
```

这将导致以下输出：

```
Configmap "envoy-config" created
```

现在我们可以使用以下命令创建我们的部署：

```
kubectl apply -f deployment.yaml
```

这将导致以下输出：

```
Deployment "my-service" created
```

最后，我们需要我们的下游服务，`service1`和`service2`。为此，我们将继续使用`http-responder`开源容器映像，在端口`5000`上进行响应。部署和服务规范可以在代码存储库中找到，并且我们可以使用以下命令创建它们：

```
kubectl create -f service1-deployment.yaml
kubectl create -f service1-service.yaml
kubectl create -f service2-deployment.yaml
kubectl create -f service2-service.yaml
```

现在，我们可以测试我们的 Envoy 配置！从我们的`my-service`容器中，我们可以向端口`8080`的本地主机发出请求，路径为`/service1`。这应该会指向我们的`service1` Pod IP 之一。为了发出这个请求，我们使用以下命令：

```
Kubectl exec <my-service-pod-name> -it -- curl localhost:8080/service1
```

我们已经设置了我们的服务来在`curl`请求上回显它们的名称。看一下我们`curl`命令的以下输出：

```
Service 1 Reached!
```

现在我们已经看过了 Envoy 如何与静态配置一起工作，让我们转向基于 Envoy 的动态服务网格 - Istio。

# 在 Kubernetes 中添加服务网格

*服务网格*模式是侧车代理的逻辑扩展。通过将侧车代理附加到每个 Pod，服务网格可以控制服务之间的功能，如高级路由规则、重试和超时。此外，通过让每个请求通过代理，服务网格可以实现服务之间的相互 TLS 加密，以增加安全性，并且可以让管理员对集群中的请求有非常好的可观察性。

有几个支持 Kubernetes 的服务网格项目。最流行的如下：

+   Istio

+   Linkerd

+   *Kuma*

+   Consul

这些服务网格中的每一个对服务网格模式有不同的看法。*Istio*可能是最流行和最全面的解决方案，但也非常复杂。*Linkerd*也是一个成熟的项目，但更容易配置（尽管它使用自己的代理而不是 Envoy）。*Consul*是一个支持 Envoy 以及其他提供者的选项，不仅仅在 Kubernetes 上。最后，*Kuma*是一个基于 Envoy 的选项，也在不断增长。

探索所有选项超出了本书的范围，因此我们将坚持使用 Istio，因为它通常被认为是默认解决方案。也就是说，所有这些网格都有优势和劣势，在计划采用服务网格时值得看看每一个。

## 在 Kubernetes 上设置 Istio

虽然 Istio 可以使用 Helm 安装，但 Helm 安装选项不再是官方支持的安装方法。

相反，我们使用`Istioctl` CLI 工具将 Istio 与配置安装到我们的集群上。这个配置可以完全定制，但是为了本书的目的，我们将只使用"demo"配置：

1.  在集群上安装 Istio 的第一步是安装 Istio CLI 工具。我们可以使用以下命令来完成这个操作，这将安装最新版本的 CLI 工具：

```
curl -L https://istio.io/downloadIstio | sh -
```

1.  接下来，我们将希望将 CLI 工具添加到我们的路径中，以便使用：

```
cd istio-<VERSION>
export PATH=$PWD/bin:$PATH
```

1.  现在，让我们安装 Istio！Istio 的配置被称为*配置文件*，如前所述，它们可以使用 YAML 文件进行完全定制。

对于这个演示，我们将使用内置的`demo`配置文件与 Istio 一起使用，这提供了一些基本设置。使用以下命令安装配置文件：

```
istioctl install --set profile=demo
```

这将导致以下输出：

![图 14.3 - Istioctl 配置文件安装输出](img/B14790_14_003.jpg)

图 14.3 - Istioctl 配置文件安装输出

1.  由于截至 Kubernetes 1.19，sidecar 资源尚未发布，因此 Istio 本身将在任何打上`istio-injection=enabled`标签的命名空间中注入 Envoy 代理。

要为任何命名空间打上标签，请运行以下命令：

```
kubectl label namespace my-namespace istio-injection=enabled
```

1.  为了方便测试，使用前面的`label`命令为`default`命名空间打上标签。一旦 Istio 组件启动，该命名空间中的任何 Pod 将自动注入 Envoy sidecar，就像我们在上一节中手动创建的那样。

要从集群中删除 Istio，请运行以下命令：

```
istioctl x uninstall --purge
```

这应该会出现一个确认消息，告诉您 Istio 已被移除。

1.  现在，让我们部署一些东西来测试我们的新网格！我们将部署三种不同的应用服务，每个都有一个部署和一个服务资源：

a. 服务前端

b. 服务后端 A

c. 服务后端 B

这是*服务前端*的部署：

Istio-service-deployment.yaml：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-frontend
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: service-frontend
        version: v2
    spec:
      containers:
      - name: service-frontend
        image: ravirdv/http-responder:latest
        ports:
        - containerPort: 5000
          name: svc-port
          protocol: TCP
```

这是*服务前端*的服务：

Istio-service-service.yaml：

```
apiVersion: v1
kind: Service
metadata:
  name: service-frontend
spec:
  selector:
    name: service-frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```

服务后端 A 和 B 的 YAML 与*服务前端*相同，除了交换名称、镜像名称和选择器标签。

1.  现在我们有了一些要路由到（和之间）的服务，让我们开始设置一些 Istio 资源！

首先，我们需要一个`Gateway`资源。在这种情况下，我们不使用 NGINX Ingress Controller，但这没关系，因为 Istio 提供了一个可以用于入口和出口的`Gateway`资源。以下是 Istio`Gateway`定义的样子：

Istio-gateway.yaml：

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: myapplication-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

这些`Gateway`定义看起来与入口记录非常相似。我们有`name`和`selector`，Istio 用它们来决定使用哪个 Istio Ingress Controller。接下来，我们有一个或多个服务器，它们实质上是我们网关上的入口点。在这种情况下，我们不限制主机，并且接受端口`80`上的请求。

1.  现在我们有了一个用于将请求发送到我们的集群的网关，我们可以开始设置一些路由。我们在 Istio 中使用`VirtualService`来做到这一点。Istio 中的`VirtualService`是一组应该遵循的路由，当对特定主机名的请求时。此外，我们可以使用通配符主机来为网格中的任何地方的请求制定全局规则。让我们看一个示例`VirtualService`配置：

Istio-virtual-service-1.yaml：

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapplication
spec:
  hosts:
  - "*"
  gateways:
  - myapplication-gateway
  http:
  - match:
    - uri:
        prefix: /app
    - uri:
        prefix: /frontend
    route:
    - destination:
        host: service-frontend
        subset: v1
```

在这个`VirtualService`中，如果匹配我们的`uri`前缀，我们将请求路由到任何主机到我们的入口点*Service Frontend*。在这种情况下，我们匹配前缀，但你也可以在 URI 匹配器中使用精确匹配，将`prefix`替换为`exact`。

1.  所以，现在我们有一个设置，与我们预期的 NGINX Ingress 非常相似，入口进入集群由路由匹配决定。

然而，在我们的路由中，`v1`是什么？这实际上代表了我们*Frontend Service*的一个版本。让我们继续使用一个新的资源类型 - Istio `DestinationRule`来指定这个版本。这是一个`DestinationRule`配置的样子：

Istio-destination-rule-1.yaml:

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: service-frontend
spec:
  host: service-frontend
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

正如你所看到的，我们在 Istio 中指定了我们前端服务的两个不同版本，每个版本都查看一个标签选择器。从我们之前的部署和服务中，你可以看到我们当前的前端服务版本是`v2`，但我们也可以并行运行两者！通过在入口虚拟服务中指定我们的`v2`版本，我们告诉 Istio 将所有请求路由到服务的`v2`。此外，我们还配置了我们的`v1`版本，它在之前的`VirtualService`中被引用。这个硬规则只是在 Istio 中将请求路由到不同子集的一种可能的方式。

现在，我们已经成功通过网关将流量路由到我们的集群，并基于目标规则路由到虚拟服务子集。在这一点上，我们实际上已经“在”我们的服务网格中！

1.  现在，从我们的*Service Frontend*，我们希望能够路由到*Service Backend A*和*Service Backend B*。我们该怎么做？更多的虚拟服务就是答案！让我们来看看*Backend Service A*的虚拟服务：

Istio-virtual-service-2.yaml:

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapplication-a
spec:
  hosts:
  - service-a
  http:
    route:
    - destination:
        host: service-backend-a
        subset: v1
```

正如你所看到的，这个`VirtualService`路由到我们服务的`v1`子集，`service-backend-a`。我们还需要另一个`VirtualService`用于`service-backend-b`，我们不会完全包含（但看起来几乎相同）。要查看完整的 YAML，请检查`istio-virtual-service-3.yaml`的代码存储库。

1.  一旦我们的虚拟服务准备好了，我们需要一些目标规则！*Backend Service A*的`DestinationRule`如下：

Istio-destination-rule-2.yaml:

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: service-backend-a
spec:
  host: service-backend-a
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1
```

*Backend Service B*的`DestinationRule`类似，只是有不同的子集。我们不会包含代码，但是在代码存储库中检查`istio-destination-rule-3.yaml`以获取确切的规格。

这些目标规则和虚拟服务相加，形成了以下路由图：

![图 14.4 - Istio 路由图](img/B14790_14_004.jpg)

图 14.4 - Istio 路由图

正如您所看到的，来自“前端服务”Pod 的请求可以路由到“后端服务 A 版本 1”或“后端服务 B 版本 3”，每个后端服务也可以相互路由。对后端服务 A 或 B 的这些请求还额外利用了 Istio 的最有价值的功能之一 - 双向 TLS。在这种设置中，网格中的任何两点之间都会自动保持 TLS 安全。

接下来，让我们看看如何在 Kubernetes 上使用无服务器模式。

# 在 Kubernetes 上实现无服务器

云提供商上的无服务器模式迅速变得越来越受欢迎。无服务器架构由可以自动扩展的计算组成，甚至可以扩展到零（即没有使用计算容量来提供函数或其他应用）。函数即服务（FaaS）是无服务器模式的扩展，其中函数代码是唯一的输入，无服务器系统负责根据需要路由请求到计算资源并进行扩展。AWS Lambda、Azure Functions 和 Google Cloud Run 是一些更受欢迎的 FaaS/无服务器选项，它们得到了云提供商的官方支持。Kubernetes 还有许多不同的无服务器框架和库，可以用于在 Kubernetes 上运行无服务器、扩展到零的工作负载以及 FaaS。其中一些最受欢迎的如下：

+   Knative

+   Kubeless

+   OpenFaaS

+   Fission

关于 Kubernetes 上所有无服务器选项的全面讨论超出了本书的范围，因此我们将专注于两种不同的选项，它们旨在满足两种完全不同的用例：OpenFaaS 和 Knative。

虽然 Knative 非常可扩展和可定制，但它使用了多个耦合的组件，增加了复杂性。这意味着需要一些额外的配置才能开始使用 FaaS 解决方案，因为函数只是 Knative 支持的许多其他模式之一。另一方面，OpenFaaS 使得在 Kubernetes 上轻松启动和运行无服务器和 FaaS 变得非常容易。这两种技术出于不同的原因都是有价值的。

在本章的教程中，我们将看看 Knative，这是最流行的无服务器框架之一，也支持通过其事件功能的 FaaS。

## 在 Kubernetes 上使用 Knative 进行 FaaS

如前所述，Knative 是用于 Kubernetes 上无服务器模式的模块化构建块。因此，在我们实际使用函数之前，它需要一些配置。Knative 也可以与 Istio 一起安装，它用作路由和扩展无服务器应用程序的基础。还有其他非 Istio 路由选项可用。

使用 Knative 进行 FaaS，我们需要安装*Knative Serving*和*Knative Eventing*。Knative Serving 将允许我们运行无服务器工作负载，而 Knative Eventing 将提供通道来向这些规模为零的工作负载发出 FaaS 请求。让我们按照以下步骤来完成这个过程：

1.  首先，让我们安装 Knative Serving 组件。我们将从安装 CRDs 开始：

```
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.18.0/serving-crds.yaml
```

1.  接下来，我们可以安装服务组件本身：

```
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.18.0/serving-core.yaml
```

1.  此时，我们需要安装一个网络/路由层供 Knative 使用。让我们使用 Istio：

```
kubectl apply --filename https://github.com/knative/net-istio/releases/download/v0.18.0/release.yaml
```

1.  我们需要从 Istio 获取网关 IP 地址。根据您运行的位置（换句话说，是在 AWS 还是本地），此值可能会有所不同。使用以下命令获取它：

```
Kubectl get service -n istio-system istio-ingressgateway
```

1.  Knative 需要特定的 DNS 设置来启用服务组件。在云设置中最简单的方法是使用`xip.io`的“Magic DNS”，尽管这对基于 Minikube 的集群不起作用。如果您正在运行其中之一（或者只是想查看所有可用选项），请查看[Knative 文档](https://knative.dev/docs/install/any-kubernetes-cluster/)。

要设置 Magic DNS，请使用以下命令：

```
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.18.0/serving-default-domain.yaml
```

1.  现在我们已经安装了 Knative Serving，让我们安装 Knative Eventing 来处理我们的 FaaS 请求。首先，我们需要更多的 CRDs。使用以下命令安装它们：

```
kubectl apply --filename https://github.com/knative/eventing/releases/download/v0.18.0/eventing-crds.yaml
```

1.  现在，安装事件组件，就像我们安装服务一样：

```
kubectl apply --filename https://github.com/knative/eventing/releases/download/v0.18.0/eventing-core.yaml
```

在这一点上，我们需要为我们的事件系统添加一个队列/消息层来使用。我们是否提到 Knative 支持许多模块化组件？

重要提示

为了简化事情，让我们只使用基本的内存消息层，但了解所有可用选项对您也是有好处的。关于消息通道的模块化选项，请查看[`knative.dev/docs/eventing/channels/channels-crds/`](https://knative.dev/docs/eventing/channels/channels-crds/)上的文档。对于事件源选项，您可以查看[`knative.dev/docs/eventing/sources/`](https://knative.dev/docs/eventing/sources/)。

1.  安装`in-memory`消息层，请使用以下命令：

```
kubectl apply --filename https://github.com/knative/eventing/releases/download/v0.18.0/in-memory-channel.yaml
```

1.  以为我们已经完成了？不！还有最后一件事。我们需要安装一个 broker，它将从消息层获取事件并将它们处理到正确的位置。让我们使用默认的 broker 层，MT-Channel broker 层。您可以使用以下命令安装它：

```
kubectl apply --filename https://github.com/knative/eventing/releases/download/v0.18.0/mt-channel-broker.yaml
```

到此为止，我们终于完成了。我们通过 Knative 安装了一个端到端的 FaaS 实现。正如你所看到的，这并不是一项容易的任务。Knative 令人惊奇的地方与令人头疼的地方是一样的——它提供了许多不同的模块选项和配置，即使在每个步骤选择了最基本的选项，我们仍然花了很多时间来解释安装过程。还有其他可用的选项，比如 OpenFaaS，它们更容易上手，我们将在下一节中进行探讨！然而，在 Knative 方面，现在我们的设置终于准备好了，我们可以添加我们的 FaaS。

### 在 Knative 中实现 FaaS 模式

现在我们已经设置好了 Knative，我们可以使用它来实现一个 FaaS 模式，其中事件将通过触发器触发在 Knative 中运行的一些代码。要设置一个简单的 FaaS，我们将需要三样东西：

+   一个从入口点路由我们的事件的 broker

+   一个消费者服务来实际处理我们的事件

+   一个指定何时将事件路由到消费者进行处理的触发器定义

首先，我们需要创建我们的 broker。这很简单，类似于创建入口记录或网关。我们的`broker` YAML 如下所示：

Knative-broker.yaml：

```
apiVersion: eventing.knative.dev/v1
kind: broker
metadata:
 name: my-broker
 namespace: default
```

接下来，我们可以创建一个消费者服务。这个组件实际上就是我们的应用程序，它将处理事件——我们的函数本身！我们不打算向你展示比你已经看到的更多的 YAML，让我们假设我们的消费者服务只是一个名为`service-consumer`的普通的 Kubernetes 服务，它路由到一个运行我们应用程序的四个副本的 Pod 部署。

最后，我们需要一个触发器。这决定了如何以及哪些事件将从 broker 路由。触发器的 YAML 如下所示：

Knative-trigger.yaml：

```
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: my-trigger
spec:
  broker: my-broker
  filter:
    attributes:
      type: myeventtype
  subscriber:
    ref:
     apiVersion: v1
     kind: Service
     name: service-consumer
```

在这个 YAML 中，我们创建了一个 `Trigger` 规则，任何通过我们的经纪人 `my-broker` 并且类型为 `myeventtype` 的事件将自动路由到我们的消费者 `service-consumer`。有关 Knative 中触发器过滤器的完整文档，请查看 [`knative.dev/development/eventing/triggers/`](https://knative.dev/development/eventing/triggers/) 上的文档。

那么，我们如何创建一些事件呢？首先，使用以下命令检查经纪人 URL：

```
kubectl get broker
```

这应该会产生以下输出：

```
NAME      READY   REASON   URL                                                                                 AGE
my-broker   True             http://broker-ingress.knative-eventing.svc.cluster.local/default/my-broker     1m
```

现在我们终于可以测试我们的 FaaS 解决方案了。让我们快速启动一个 Pod，从中我们可以向我们的触发器发出请求：

```
kubectl run -i --tty --rm debug --image=radial/busyboxplus:curl --restart=Never -- sh
```

现在，从这个 Pod 内部，我们可以继续测试我们的触发器，使用 `curl`。我们需要发出的请求需要有一个等于 `myeventtype` 的 `Ce-Type` 标头，因为这是我们触发器所需的。Knative 使用形式为 `Ce-Id`、`Ce-Type` 的标头，如下面的代码块所示，来进行路由。

`curl` 请求将如下所示：

```
curl -v "http://broker-ingress.knative-eventing.svc.cluster.local/default/my-broker" \
  -X POST \
  -H "Ce-Id: anyid" \
  -H "Ce-Specversion: 1.0" \
  -H "Ce-Type: myeventtype" \
  -H "Ce-Source: any" \
  -H "Content-Type: application/json" \
  -d '{"payload":"Does this work?"}'
```

正如您所看到的，我们正在向经纪人 URL 发送 `curl` `http` 请求。此外，我们还在 HTTP 请求中传递了一些特殊的标头。重要的是，我们传递了 `type=myeventtype`，这是我们触发器上的过滤器所需的，以便发送请求进行处理。

在这个例子中，我们的消费者服务会回显请求的 JSON 主体的 payload 键，以及一个 `200` 的 HTTP 响应，因此运行这个 `curl` 请求会给我们以下结果：

```
> HTTP/1.1 200 OK
> Content-Type: application/json
{
  "Output": "Does this work?"
}
```

成功！我们已经测试了我们的 FaaS，并且它返回了我们期望的结果。从这里开始，我们的解决方案将根据事件的数量进行零扩展和缩减，与 Knative 的所有内容一样，还有许多自定义和配置选项，可以精确地调整我们的解决方案以满足我们的需求。

接下来，我们将使用 OpenFaaS 而不是 Knative 来查看相同的模式，以突出两种方法之间的区别。

## 在 Kubernetes 上使用 OpenFaaS 进行 FaaS

现在我们已经讨论了如何开始使用 Knative，让我们用 OpenFaaS 做同样的事情。首先，要安装 OpenFaaS 本身，我们将使用来自 `faas-netes` 仓库的 Helm 图表，该仓库位于 [`github.com/openfaas/faas-netes`](https://github.com/openfaas/faas-netes)。

### 使用 Helm 安装 OpenFaaS 组件

首先，我们将创建两个命名空间来保存我们的 OpenFaaS 组件：

+   `openfaas` 用于保存 OpenFaas 的实际服务组件

+   `openfaas-fn` 用于保存我们部署的函数

我们可以使用以下命令使用`faas-netes`存储库中的一个巧妙的 YAML 文件来添加这两个命名空间：

```
kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
```

接下来，我们需要使用以下 Helm 命令添加`faas-netes` `Helm` `存储库`：

```
helm repo add openfaas https://openfaas.github.io/faas-netes/
helm repo update
```

最后，我们实际部署 OpenFaaS！

在前面的`faas-netes`存储库中的 OpenFaaS 的 Helm 图表有几个可能的变量，但我们将使用以下配置来确保创建一组初始的身份验证凭据，并部署入口记录：

```
helm install openfaas openfaas/openfaas \
    --namespace openfaas  \
    --set functionNamespace=openfaas-fn \
    --set ingress.enabled=true \
    --set generateBasicAuth=true 
```

现在，我们的 OpenFaaS 基础设施已经部署到我们的集群中，我们将希望获取作为 Helm 安装的一部分生成的凭据。Helm 图表将作为钩子的一部分创建这些凭据，并将它们存储在一个秘密中，因此我们可以通过运行以下命令来获取它们：

```
OPENFAASPWD=$(kubectl get secret basic-auth -n openfaas -o jsonpath="{.data.basic-auth-password}" | base64 --decode)
```

这就是我们需要的所有 Kubernetes 设置！

接下来，让我们安装 OpenFaas CLI，这将使管理 OpenFaas 函数变得非常容易。

### 安装 OpenFaaS CLI 和部署函数

要安装 OpenFaaS CLI，我们可以使用以下命令（对于 Windows，请查看前面的 OpenFaaS 文档）：

```
curl -sL https://cli.openfaas.com | sudo sh
```

现在，我们可以开始构建和部署一些函数。这最容易通过 CLI 来完成。

在构建和部署 OpenFaaS 的函数时，OpenFaaS CLI 提供了一种简单的方法来生成样板，并为特定语言构建和部署函数。它通过“模板”来实现这一点，并支持各种类型的 Node、Python 等。有关模板类型的完整列表，请查看[`github.com/openfaas/templates`](https://github.com/openfaas/templates)上的模板存储库。

使用 OpenFaaS CLI 创建的模板类似于您从 AWS Lambda 等托管无服务器平台期望的内容。让我们使用以下命令创建一个全新的 Node.js 函数：

```
faas-cli new my-function –lang node
```

这将产生以下输出：

```
Folder: my-function created.
Function created in folder: my-function
Stack file written: my-function.yml
```

正如你所看到的，`new`命令生成一个文件夹，在其中有一些函数代码本身的样板，以及一个 OpenFaaS YAML 文件。

OpenFaaS YAML 文件将如下所示：

My-function.yml:

```
provider:
  name: openfaas
  gateway: http://localhost:8080
functions:
  my-function:
    lang: node
    handler: ./my-function
    image: my-function
```

实际的函数代码（在`my-function`文件夹中）包括一个函数文件`handler.js`和一个依赖清单`package.json`。对于其他语言，这些文件将是不同的，我们不会深入讨论 Node 中的具体依赖。但是，我们将编辑`handler.js`文件以返回一些文本。编辑后的文件如下所示：

Handler.js:

```
"use strict"
module.exports = (context, callback) => {
    callback(undefined, {output: "my function succeeded!"});
}
```

这段 JavaScript 代码将返回一个包含我们文本的 JSON 响应。

现在我们有了我们的函数和处理程序，我们可以继续构建和部署我们的函数。OpenFaaS CLI 使构建函数变得简单，我们可以使用以下命令来完成：

```
faas-cli build -f /path/to/my-function.yml 
```

该命令的输出很长，但当完成时，我们将在本地构建一个新的容器映像，其中包含我们的函数处理程序和依赖项！

接下来，我们将像对待任何其他容器一样，将我们的容器映像推送到容器存储库。OpenFaaS CLI 具有一个很好的包装命令，可以将映像推送到 Docker Hub 或其他容器映像存储库：

```
faas-cli push -f my-function.yml 
```

现在，我们可以将我们的函数部署到 OpenFaaS。再次，这由 CLI 轻松完成。使用以下命令进行部署：

```
faas-cli deploy -f my-function.yml
```

现在，一切都已准备好让我们测试在 OpenFaaS 上部署的函数了！我们在部署 OpenFaaS 时使用了一个入口设置，以便请求可以通过该入口。但是，我们新函数生成的 YAML 文件设置为在开发目的地对`localhost:8080`进行请求。我们可以编辑该文件以将请求发送到我们入口网关的正确`URL`（请参阅[`docs.openfaas.com/deployment/kubernetes/`](https://docs.openfaas.com/deployment/kubernetes/)中的文档），但相反，让我们通过快捷方式在本地主机上打开 OpenFaaS。

让我们使用`kubectl port-forward`命令在本地主机端口`8080`上打开我们的 OpenFaaS 服务。我们可以按照以下方式进行：

```
export OPENFAAS_URL=http://127.0.0.1:8080
kubectl port-forward -n openfaas svc/gateway 8080:8080
```

现在，让我们按照以下方式将先前生成的 auth 凭据添加到 OpenFaaS CLI 中：

```
echo -n $OPENFAASPWD | faas-cli login -g $OPENFAAS_URL -u admin --password-stdin
```

最后，为了测试我们的函数，我们只需运行以下命令：

```
faas-cli invoke -f my-function.yml my-function
```

这将产生以下输出：

```
Reading from STDIN - hit (Control + D) to stop.
This is my message
{ output: "my function succeeded!"});}
```

如您所见，我们成功收到了我们预期的响应！

最后，如果我们想要删除这个特定的函数，我们可以使用以下命令，类似于我们使用`kubectl delete -f`的方式：

```
faas-cli rm -f my-function.yml 
```

就是这样！我们的函数已被删除。

# 总结

在本章中，我们学习了关于 Kubernetes 上的服务网格和无服务器模式。为了为这些做好准备，我们首先讨论了在 Kubernetes 上运行边车代理，特别是使用 Envoy 代理。

然后，我们转向服务网格，并学习了如何安装和配置 Istio 服务网格，以实现服务到服务的互相 TLS 路由。

最后，我们转向了在 Kubernetes 上的无服务器模式，您将学习如何配置和安装 Knative，以及另一种选择 OpenFaaS，用于 Kubernetes 上的无服务器事件和 FaaS。

本章中使用的技能将帮助您在 Kubernetes 上构建服务网格和无服务器模式，为您提供完全自动化的服务发现和 FaaS 事件。

在下一章（也是最后一章）中，我们将讨论在 Kubernetes 上运行有状态应用程序。

# 问题

1.  静态和动态 Envoy 配置有什么区别？

1.  Envoy 配置的四个主要部分是什么？

1.  Knative 的一些缺点是什么，OpenFaaS 又如何比较？

# 进一步阅读

+   CNCF 景观：[`landscape.cncf.io/`](https://landscape.cncf.io/)

+   官方 Kubernetes 论坛：[`discuss.kubernetes.io/`](https://discuss.kubernetes.io/)

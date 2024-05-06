# *第六章*：Kubernetes 应用程序配置

本章描述了 Kubernetes 提供的主要配置工具。我们将首先讨论一些将配置注入到容器化应用程序中的最佳实践。接下来，我们将讨论 ConfigMaps，这是 Kubernetes 旨在为应用程序提供配置数据的资源。最后，我们将介绍 Secrets，这是一种安全的方式，用于存储和提供敏感数据给在 Kubernetes 上运行的应用程序。总的来说，本章应该为您提供一个很好的工具集，用于在 Kubernetes 上配置生产应用程序。

在本章中，我们将涵盖以下主题：

+   使用最佳实践配置容器化应用程序

+   实施 ConfigMaps

+   使用 Secrets

# 技术要求

为了运行本章详细介绍的命令，您需要一台支持`kubectl`命令行工具的计算机，以及一个正常运行的 Kubernetes 集群。请查看*第一章*，*与 Kubernetes 通信*，以找到快速启动和运行 Kubernetes 的几种方法，并获取有关如何安装`kubectl`工具的说明。

本章中使用的代码可以在书籍的 GitHub 存储库中找到，网址为[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter6`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter6)。

# 使用最佳实践配置容器化应用程序

到目前为止，我们知道如何有效地部署（如*第四章*中所述，*扩展和部署您的应用程序*）和暴露（如*第五章*中所述，*服务和入口* - *与外部世界通信*）Kubernetes 上的容器化应用程序。这已足以在 Kubernetes 上运行非平凡的无状态容器化应用程序。然而，Kubernetes 还提供了用于应用程序配置和 Secrets 管理的额外工具。

由于 Kubernetes 运行容器，您始终可以配置应用程序以使用嵌入到 Dockerfile 中的环境变量。但这有些绕过了像 Kubernetes 这样的编排器的一些真正价值。我们希望能够在不重建 Docker 镜像的情况下更改我们的应用程序容器。为此，Kubernetes 为我们提供了两个以配置为重点的资源：ConfigMaps 和 Secrets。让我们首先看一下 ConfigMaps。

## 理解 ConfigMaps

在生产环境中运行应用程序时，开发人员希望能够快速、轻松地注入应用程序配置信息。有许多模式可以做到这一点 - 从使用查询的单独配置服务器，到使用环境变量或环境文件。这些策略在提供的安全性和可用性上有所不同。

对于容器化应用程序来说，环境变量通常是最简单的方法 - 但以安全的方式注入这些变量可能需要额外的工具或脚本。在 Kubernetes 中，ConfigMap 资源让我们以灵活、简单的方式做到这一点。ConfigMaps 允许 Kubernetes 管理员指定和注入配置信息，可以是文件或环境变量。

对于诸如秘密密钥之类的高度敏感信息，Kubernetes 为我们提供了另一个类似的资源 - Secrets。

## 理解 Secrets

Secrets 指的是需要以稍微更安全的方式存储的额外应用程序配置项 - 例如，受限 API 的主密钥、数据库密码等。Kubernetes 提供了一个称为 Secret 的资源，以编码方式存储应用程序配置信息。这并不会本质上使 Secret 更安全，但 Kubernetes 通过不自动在`kubectl get`或`kubectl describe`命令中打印秘密信息来尊重秘密的概念。这可以防止秘密意外打印到日志中。

为了确保 Secrets 实际上是秘密的，必须在集群上启用对秘密数据的静态加密 - 我们将在本章后面讨论如何做到这一点。从 Kubernetes 1.13 开始，这个功能让 Kubernetes 管理员可以防止 Secrets 未加密地存储在`etcd`中，并限制对`etcd`管理员的访问。

在我们深入讨论 Secrets 之前，让我们先讨论一下 ConfigMaps，它们更适合非敏感信息。

# 实施 ConfigMaps

ConfigMaps 为在 Kubernetes 上运行的容器存储和注入应用程序配置数据提供了一种简单的方式。

创建 ConfigMap 很简单 - 它们可以实现两种注入应用程序配置数据的可能性：

+   作为环境变量注入

+   作为文件注入

虽然第一种选项仅仅是在内存中使用容器环境变量，但后一种选项涉及到一些卷的方面 - 一种 Kubernetes 存储介质，将在下一章中介绍。我们现在将简要回顾一下，并将其用作卷的介绍，这将在下一章*第七章*中进行扩展，*Kubernetes 上的存储*。

在处理 ConfigMaps 时，使用命令式的`Kubectl`命令创建它们可能更容易。创建 ConfigMaps 的方法有几种，这也导致了从 ConfigMap 本身存储和访问数据的方式上的差异。第一种方法是简单地从文本值创建它，接下来我们将看到。

## 从文本值

通过命令从文本值创建 ConfigMap 的方法如下：

```
kubectl create configmap myapp-config --from-literal=mycategory.mykey=myvalue 
```

上一个命令创建了一个名为`myapp-config`的`configmap`，其中包含一个名为`mycategory.mykey`的键，其值为`myvalue`。您也可以创建一个具有多个键和值的 ConfigMap，如下所示：

```
kubectl create configmap myapp-config2 --from-literal=mycategory.mykey=myvalue
--from-literal=mycategory.mykey2=myvalue2 
```

上述命令会在`data`部分中生成一个具有两个值的 ConfigMap。

要查看您的 ConfigMap 的样子，请运行以下命令：

```
kubectl get configmap myapp-config2
```

您将看到以下输出：

configmap-output.yaml

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config2
  namespace: default
data:
  mycategory.mykey: myvalue
  mycategory.mykey2: myvalue2
```

当您的 ConfigMap 数据很长时，直接从文本值创建它就没有太多意义。对于更长的配置，我们可以从文件创建我们的 ConfigMap。

## 从文件

为了更容易创建一个具有许多不同值的 ConfigMap，或者重用您已经拥有的环境文件，您可以按照以下步骤从文件创建一个 ConfigMap：

1.  让我们从创建我们的文件开始，我们将把它命名为`env.properties`：

```
myconfigid=1125
publicapikey=i38ahsjh2
```

1.  然后，我们可以通过运行以下命令来创建我们的 ConfigMap：

```
kubectl create configmap my-config-map --from-file=env.properties
```

1.  要检查我们的`kubectl create`命令是否正确创建了 ConfigMap，让我们使用`kubectl describe`来描述它：

```
kubectl describe configmaps my-config-map
```

这应该会产生以下输出：

```
Name:           my-config-map
Namespace:      default
Labels:         <none>
Annotations:    <none>
Data
====
env.properties:        39 bytes
```

正如你所看到的，这个 ConfigMap 包含了我们的文本文件（以及字节数）。在这种情况下，我们的文件可以是任何文本文件 - 但是如果你知道你的文件被格式化为环境文件，你可以让 Kubernetes 知道这一点，以便让你的 ConfigMap 更容易阅读。让我们学习如何做到这一点。

## 从环境文件

如果我们知道我们的文件格式化为普通的环境文件与键值对，我们可以使用稍微不同的方法来创建我们的 ConfigMap-环境文件方法。这种方法将使我们的数据在 ConfigMap 对象中更加明显，而不是隐藏在文件中。

让我们使用与之前相同的文件进行环境特定的创建：

```
kubectl create configmap my-env-config-map --from-env-file=env.properties
```

现在，让我们使用以下命令描述我们的 ConfigMap：

```
> kubectl describe configmaps my-env-config-map
```

我们得到以下输出：

```
Name:         my-env-config-map
Namespace:    default
Labels:       <none>
Annotations:  <none>
Data
====
myconfigid:
----
1125
publicapikey:
----
i38ahsjh2
Events:  <none>
```

如您所见，通过使用`-from-env-file`方法，当您运行`kubectl describe`时，`env`文件中的数据很容易查看。这也意味着我们可以直接将我们的 ConfigMap 挂载为环境变量-稍后会详细介绍。

## 将 ConfigMap 挂载为卷

要在 Pod 中使用 ConfigMap 中的数据，您需要在规范中将其挂载到 Pod 中。这与在 Kubernetes 中挂载卷的方式非常相似（出于很好的原因，我们将会发现），卷是提供存储的资源。但是现在，不要担心卷。

让我们来看看我们的 Pod 规范，它将我们的`my-config-map` ConfigMap 作为卷挂载到我们的 Pod 上：

pod-mounting-cm.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-mount-cm
spec:
  containers:
    - name: busybox
      image: busybox
      command:
      - sleep
      - "3600"
      volumeMounts:
      - name: my-config-volume
        mountPath: /app/config
  volumes:
    - name: my-config-volume
      configMap:
        name: my-config-map
  restartPolicy: Never
```

如您所见，我们的`my-config-map` ConfigMap 被挂载为卷（`my-config-volume`）在`/app/config`路径上，以便我们的容器访问。我们将在下一章关于存储中更多了解这是如何工作的。

在某些情况下，您可能希望将 ConfigMap 挂载为容器中的环境变量-我们将在下面学习如何做到这一点。

## 将 ConfigMap 挂载为环境变量

您还可以将 ConfigMap 挂载为环境变量。这个过程与将 ConfigMap 挂载为卷非常相似。

让我们来看看我们的 Pod 规范：

pod-mounting-cm-as-env.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-mount-env
spec:
  containers:
    - name: busybox
      image: busybox
      command:
      - sleep
      - "3600"
      env:
        - name: MY_ENV_VAR
          valueFrom:
            configMapKeyRef:
              name: my-env-config-map
              key: myconfigid
  restartPolicy: Never
```

正如您所看到的，我们不是将 ConfigMap 作为卷挂载，而是在容器环境变量`MY_ENV_VAR`中引用它。为了做到这一点，我们需要在`valueFrom`键中使用`configMapRef`，并引用我们的 ConfigMap 的名称以及 ConfigMap 本身内部要查看的键。

正如我们在*使用最佳实践配置容器化应用程序*部分的章节开头提到的，ConfigMaps 默认情况下不安全，它们的数据以明文存储。为了增加一层安全性，我们可以使用 Secrets 而不是 ConfigMaps。

# 使用 Secrets

Secrets 与 ConfigMaps 非常相似，不同之处在于它们以编码文本（具体来说是 Base64）而不是明文存储。

因此，创建秘密与创建 ConfigMap 非常相似，但有一些关键区别。首先，通过命令方式创建秘密将自动对秘密中的数据进行 Base64 编码。首先，让我们看看如何从一对文件中命令方式创建秘密。

## 从文件

首先，让我们尝试从文件创建一个秘密（这也适用于多个文件）。我们可以使用`kubectl create`命令来做到这一点：

```
> echo -n 'mysecretpassword' > ./pass.txt
> kubectl create secret generic my-secret --from-file=./pass.txt
```

这应该会产生以下输出：

```
secret "my-secret" created
```

现在，让我们使用`kubectl describe`来查看我们的秘密是什么样子的：

```
> kubectl describe secrets/db-user-pass
```

这个命令应该会产生以下输出：

```
Name:            my-secret
Namespace:       default
Labels:          <none>
Annotations:     <none>
Type:            Opaque
Data
====
pass.txt:    16 bytes
```

正如您所看到的，`describe`命令显示了秘密中包含的字节数，以及它的类型`Opaque`。

创建秘密的另一种方法是使用声明性方法手动创建它。让我们看看如何做到这一点。

## 手动声明性方法

当从 YAML 文件声明性地创建秘密时，您需要使用编码实用程序预先对要存储的数据进行编码，例如 Linux 上的`base64`管道。

让我们在这里使用 Linux 的`base64`命令对我们的密码进行编码：

```
> echo -n 'myverybadpassword' | base64
bXl2ZXJ5YmFkcGFzc3dvcmQ=
```

现在，我们可以使用 Kubernetes YAML 规范声明性地创建我们的秘密，我们可以将其命名为`secret.yaml`：

```
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  dbpass: bXl2ZXJ5YmFkcGFzc3dvcmQ=
```

我们的`secret.yaml`规范包含我们创建的 Base64 编码字符串。

要创建秘密，请运行以下命令：

```
kubectl create -f secret.yaml
```

现在您知道如何创建秘密了。接下来，让我们学习如何挂载一个秘密供 Pod 使用。

## 将秘密挂载为卷

挂载秘密与挂载 ConfigMaps 非常相似。首先，让我们看看如何将秘密挂载到 Pod 作为卷（文件）。

让我们来看看我们的 Pod 规范。在这种情况下，我们正在运行一个示例应用程序，以便测试我们的秘密。以下是 YAML：

pod-mounting-secret.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-mount-cm
spec:
  containers:
    - name: busybox
      image: busybox
      command:
      - sleep
      - "3600"
      volumeMounts:
      - name: my-config-volume
        mountPath: /app/config
        readOnly: true
  volumes:
    - name: foo
      secret:
      secretName: my-secret
  restartPolicy: Never
```

与 ConfigMap 的一个区别是，我们在卷上指定了`readOnly`，以防止在 Pod 运行时对秘密进行任何更改。在其他方面，我们挂载秘密的方式与 ConfigMap 相同。

接下来，我们将在下一章[*第七章*]（B14790_07_Final_PG_ePub.xhtml#_idTextAnchor166）*Kubernetes 上的存储*中深入讨论卷。但简单解释一下，卷是一种向 Pod 添加存储的方式。在这个例子中，我们挂载了我们的卷，你可以把它看作是一个文件系统，到我们的 Pod 上。然后我们的秘密被创建为文件系统中的一个文件。

## 将秘密挂载为环境变量

类似于文件挂载，我们可以以与 ConfigMap 挂载方式相同的方式将我们的秘密作为环境变量挂载。

让我们看一下另一个 Pod YAML。在这种情况下，我们将我们的 Secret 作为环境变量挂载：

pod-mounting-secret-env.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-mount-env
spec:
  containers:
    - name: busybox
      image: busybox
      command:
      - sleep
      - "3600"
      env:
        - name: MY_PASSWORD_VARIABLE
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: dbpass
  restartPolicy: Never
```

在使用`kubectl apply`创建前面的 Pod 后，让我们运行一个命令来查看我们的 Pod，看看变量是否被正确初始化。这与`docker exec`的方式完全相同：

```
> kubectl exec -it my-pod-mount-env -- /bin/bash
> printenv MY_PASSWORD_VARIABLE
myverybadpassword
```

它奏效了！现在您应该对如何创建，挂载和使用 ConfigMaps 和 Secrets 有了很好的理解。

作为关于 Secrets 的最后一个主题，我们将学习如何使用 Kubernetes `EncryptionConfig`创建安全的加密 Secrets。

## 实施加密的 Secrets

一些托管的 Kubernetes 服务（包括亚马逊的**弹性 Kubernetes 服务**（**EKS**））会自动加密`etcd`数据在静止状态下-因此您无需执行任何操作即可实现加密的 Secrets。像 Kops 这样的集群提供者有一个简单的标志（例如`encryptionConfig: true`）。但是，如果您是*以困难的方式*创建集群，您需要使用一个标志`--encryption-provider-config`和一个`EncryptionConfig`文件启动 Kubernetes API 服务器。

重要提示

从头开始创建一个完整的集群超出了本书的范围（请参阅*Kubernetes The Hard Way*，了解更多信息，网址为[`github.com/kelseyhightower/kubernetes-the-hard-way`](https://github.com/kelseyhightower/kubernetes-the-hard-way)）。

要快速了解加密是如何处理的，请查看以下`EncryptionConfiguration` YAML，它在启动时传递给`kube-apiserver`：

encryption-config.yaml

```
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aesgcm:
        keys:
        - name: key1
          secret: c2VjcmV0IGlzIHNlY3VyZQ==
        - name: key2
          secret: dGhpcyBpcyBwYXNzd29yZA==
```

前面的`EncryptionConfiguration` YAML 列出了应在`etcd`中加密的资源列表，以及可用于加密数据的一个或多个提供程序。截至 Kubernetes `1.17`，允许以下提供程序：

+   身份：无加密。

+   Aescbc：推荐的加密提供程序。

+   秘密盒：比 Aescbc 更快，更新。

+   Aesgcm：请注意，您需要自己实现 Aesgcm 的密钥轮换。

+   Kms：与第三方 Secrets 存储一起使用，例如 Vault 或 AWS KMS。

要查看完整列表，请参阅 https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#providers。当列表中添加多个提供程序时，Kubernetes 将使用第一个配置的提供程序来加密对象。在解密时，Kubernetes 将按列表顺序进行解密尝试-如果没有一个有效，它将返回错误。

一旦我们创建了一个秘密（查看我们以前的任何示例如何做到这一点），并且我们的`EncryptionConfig`是活动的，我们可以检查我们的秘密是否实际上是加密的。

## 检查您的秘密是否已加密

检查您的秘密是否实际上在`etcd`中被加密的最简单方法是直接从`etcd`中获取值并检查加密前缀：

1.  首先，让我们使用`base64`创建一个秘密密钥：

```
> echo -n 'secrettotest' | base64
c2VjcmV0dG90ZXN0
```

1.  创建一个名为`secret_to_test.yaml`的文件，其中包含以下内容：

```
apiVersion: v1
kind: Secret
metadata:
 name: secret-to-test
type: Opaque
data:
  myencsecret: c2VjcmV0dG90ZXN0
```

1.  创建秘密：

```
kubectl apply -f secret_to_test.yaml
```

1.  创建了我们的秘密后，让我们检查它是否在`etcd`中被加密，通过直接查询它。您通常不需要经常直接查询`etcd`，但如果您可以访问用于引导集群的证书，这是一个简单的过程：

```
> export ETCDCTL_API=3 
> etcdctl --cacert=/etc/kubernetes/certs/ca.crt 
--cert=/etc/kubernetes/certs/etcdclient.crt 
--key=/etc/kubernetes/certs/etcdclient.key 
get /registry/secrets/default/secret-to-test
```

根据您配置的加密提供程序，您的秘密数据将以提供程序标记开头。例如，使用 Azure KMS 提供程序加密的秘密将以`k8s:enc:kms:v1:azurekmsprovider`开头。

1.  现在，通过`kubectl`检查秘密是否被正确解密（它仍然会被编码）：

```
> kubectl get secrets secret-to-test -o yaml
```

输出应该是`myencsecret: c2VjcmV0dG90ZXN0`，这是我们未加密的编码的秘密值：

```
> echo 'c2VjcmV0dG90ZXN0' | base64 --decode
> secrettotest
```

成功！

我们现在在我们的集群上运行加密。让我们找出如何删除它。

## 禁用集群加密

我们也可以相当容易地从我们的 Kubernetes 资源中删除加密。

首先，我们需要使用空白的加密配置 YAML 重新启动 Kubernetes API 服务器。如果您自行配置了集群，这应该很容易，但在 EKS 或 AKS 上，这是不可能手动完成的。您需要按照云提供商的具体文档来了解如何禁用加密。

如果您自行配置了集群或使用了诸如 Kops 或 Kubeadm 之类的工具，那么您可以使用以下`EncryptionConfiguration`在所有主节点上重新启动您的`kube-apiserver`进程：

encryption-reset.yaml

```
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - identity: {}
```

重要提示

请注意，身份提供者不需要是唯一列出的提供者，但它需要是第一个，因为正如我们之前提到的，Kubernetes 使用第一个提供者来加密`etcd`中的新/更新对象。

现在，我们将手动重新创建所有我们的秘密，此时它们将自动使用身份提供者（未加密）：

```
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

此时，我们所有的秘密都是未加密的！

# 摘要

在本章中，我们看了 Kubernetes 提供的注入应用程序配置的方法。首先，我们看了一些配置容器化应用程序的最佳实践。然后，我们回顾了 Kubernetes 提供的第一种方法，ConfigMaps，以及创建和挂载它们到 Pod 的几个选项。最后，我们看了一下 Secrets，当它们被加密时，是处理敏感配置的更安全的方式。到目前为止，您应该已经掌握了为应用程序提供安全和不安全配置值所需的所有工具。

在下一章中，我们将深入探讨一个我们已经涉及到的主题，即挂载我们的 Secrets 和 ConfigMaps - Kubernetes 卷资源，以及更一般地说，Kubernetes 上的存储。

# 问题

1.  Secrets 和 ConfigMaps 之间有什么区别？

1.  Secrets 是如何编码的？

1.  从常规文件创建 ConfigMap 和从环境文件创建 ConfigMap 之间的主要区别是什么？

1.  如何在 Kubernetes 上确保 Secrets 的安全？为什么它们不是默认安全的？

# 进一步阅读

+   有关 Kubernetes 数据加密配置的信息可以在官方文档中找到[`kubernetes.io/docs/tasks/administer-cluster/encrypt-data/`](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)。

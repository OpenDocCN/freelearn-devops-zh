# *第七章*：Kubernetes 上的存储

在本章中，我们将学习如何在 Kubernetes 上提供应用程序存储。我们将回顾 Kubernetes 上的两种存储资源，即卷和持久卷。卷非常适合临时数据需求，但持久卷对于在 Kubernetes 上运行任何严肃的有状态工作负载是必不可少的。通过本章学到的技能，您将能够在多种不同的方式和环境中为在 Kubernetes 上运行的应用程序配置存储。

在本章中，我们将涵盖以下主题：

+   理解卷和持久卷之间的区别

+   使用卷

+   创建持久卷

+   持久卷索赔

# 技术要求

为了运行本章中详细介绍的命令，您需要一台支持`kubectl`命令行工具的计算机，以及一个可用的 Kubernetes 集群。请参阅*第一章*，*与 Kubernetes 通信*，了解快速启动和运行 Kubernetes 的几种方法，以及如何安装`kubectl`工具的说明。

本章中使用的代码可以在书籍的 GitHub 存储库中找到：[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter7`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter7)。

# 理解卷和持久卷之间的区别

一个完全无状态的容器化应用可能只需要磁盘空间来存储容器文件本身。在运行这种类型的应用程序时，Kubernetes 不需要额外的配置。

然而，在现实世界中，这并不总是正确的。正在转移到容器中的传统应用程序可能出于许多可能的原因需要磁盘空间卷。为了保存容器使用的文件，您需要 Kubernetes 卷资源。

Kubernetes 中可以创建两种主要存储资源：

+   卷

+   持久卷

两者之间的区别在于名称：虽然卷与特定 Pod 的生命周期相关联，但持久卷会一直保持活动状态，直到被删除，并且可以在不同的 Pod 之间共享。卷可以在 Pod 内部的容器之间共享数据，而持久卷可以用于许多可能的高级目的。

让我们先看看如何实现卷。

# 卷

Kubernetes 支持许多不同类型的卷。大多数可以用于卷或持久卷，但有些是特定于资源的。我们将从最简单的开始，然后回顾一些类型。

重要提示

您可以在 https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes 上查看完整的当前卷类型列表。

以下是卷子类型的简短列表：

+   `awsElasticBlockStore`

+   `cephfs`

+   `ConfigMap`

+   `emptyDir`

+   `hostPath`

+   `local`

+   `nfs`

+   `persistentVolumeClaim`

+   `rbd`

+   `Secret`

正如您所看到的，ConfigMaps 和 Secrets 实际上是卷的*类型*。此外，列表包括云提供商卷类型，如`awsElasticBlockStore`。

与持久卷不同，持久卷是单独从任何一个 Pod 创建的，创建卷通常是在 Pod 的上下文中完成的。

要创建一个简单的卷，可以使用以下 Pod YAML：

pod-with-vol.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-vol
spec:
  containers:
  - name: busybox
    image: busybox
    volumeMounts:
    - name: my-storage-volume
      mountPath: /data
  volumes:
  - name: my-storage-volume
    emptyDir: {}
```

这个 YAML 将创建一个带有`emptyDir`类型卷的 Pod。`emptyDir`类型的卷是使用分配给 Pod 的节点上已经存在的存储来配置的。如前所述，卷与 Pod 的生命周期相关，而不是与其容器相关。

这意味着在具有多个容器的 Pod 中，所有容器都将能够访问卷数据。让我们看一个 Pod 的以下示例 YAML 文件：

pod-with-multiple-containers.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: busybox
    image: busybox
    volumeMounts:
    - name: config-volume
      mountPath: /shared-config
  - name: busybox2
    image: busybox
    volumeMounts:
    - name: config-volume
      mountPath: /myconfig
  volumes:
  - name: config-volume
    emptyDir: {}
```

在这个例子中，Pod 中的两个容器都可以访问卷数据，尽管路径不同。容器甚至可以通过共享卷中的文件进行通信。

规范的重要部分是`volume spec`本身（`volumes`下的列表项）和卷的`mount`（`volumeMounts`下的列表项）。

每个挂载项都包含一个名称，对应于`volumes`部分中卷的名称，以及一个`mountPath`，它将决定卷被挂载到容器上的哪个文件路径。例如，在前面的 YAML 中，卷`config-volume`将在`busybox` Pod 中的`/shared-config`处访问，在`busybox2` Pod 中的`/myconfig`处访问。

卷规范本身需要一个名称 - 在本例中是`my-storage`，以及特定于卷类型的其他键/值，本例中是`emptyDir`，只需要空括号。

现在，让我们来看一个云配置卷挂载到 Pod 的例子。例如，要挂载 AWS **弹性块存储**（**EBS**）卷，可以使用以下 YAML：

pod-with-ebs.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - image: busybox
    name: busybox
    volumeMounts:
    - mountPath: /data
      name: my-ebs-volume
  volumes:
  - name: my-ebs-volume
    awsElasticBlockStore:
      volumeID: [INSERT VOLUME ID HERE]
```

只要您的集群正确设置了与 AWS 的身份验证，此 YAML 将把现有的 EBS 卷附加到 Pod 上。正如您所看到的，我们使用`awsElasticBlockStore`键来专门配置要使用的确切卷 ID。在这种情况下，EBS 卷必须已经存在于您的 AWS 帐户和区域中。使用 AWS **弹性 Kubernetes 服务**（**EKS**）会更容易，因为它允许我们从 Kubernetes 内部自动提供 EBS 卷。

Kubernetes 还包括 Kubernetes AWS 云提供程序中的功能，用于自动提供卷-但这些是用于持久卷。我们将在*持久卷*部分看看如何获得这些自动提供的卷。

# 持久卷

持久卷相对于常规的 Kubernetes 卷具有一些关键优势。如前所述，它们（持久卷）的生命周期与集群的生命周期相关，而不是与单个 Pod 的生命周期相关。这意味着持久卷可以在集群运行时在 Pod 之间共享和重复使用。因此，这种模式更适合外部存储，比如 EBS（AWS 上的块存储服务），因为存储本身可以超过单个 Pod 的寿命。

实际上，使用持久卷需要两个资源：`PersistentVolume`本身和`PersistentVolumeClaim`，用于将`PersistentVolume`挂载到 Pod 上。

让我们从`PersistentVolume`本身开始-看一下创建`PersistentVolume`的基本 YAML：

pv.yaml

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/mydata"
```

现在让我们来分析一下。从规范中的第一行开始-`storageClassName`。

这个第一个配置，`storageClassName`，代表了我们想要使用的存储类型。对于`hostPath`卷类型，我们只需指定`manual`，但是对于 AWS EBS，例如，您可以创建并使用一个名为`gp2Encrypted`的存储类，以匹配 AWS 中的`gp2`存储类型，并启用 EBS 加密。因此，存储类是特定卷类型的可用配置的组合-可以在卷规范中引用。

继续使用我们的 AWS `StorageClass`示例，让我们为`gp2Encrypted`提供一个新的`StorageClass`：

gp2-storageclass.yaml

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp2Encrypted
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  encrypted: "true"
  fsType: ext4
```

现在，我们可以使用`gp2Encrypted`存储类创建我们的`PersistentVolume`。但是，使用动态配置的 EBS（或其他云）卷创建`PersistentVolumes`有一个快捷方式。当使用动态配置的卷时，我们首先创建`PersistentVolumeClaim`，然后自动生成`PersistentVolume`。

## 持久卷声明

现在我们知道您可以在 Kubernetes 中轻松创建持久卷，但是这并不允许您将存储绑定到 Pod。您需要创建一个`PersistentVolumeClaim`，它声明一个`PersistentVolume`并允许您将该声明绑定到一个或多个 Pod。

在上一节的新`StorageClass`的基础上，让我们创建一个声明，这将自动导致创建一个新的`PersistentVolume`，因为没有其他具有我们期望的`StorageClass`的持久卷：

pvc.yaml

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-pv-claim
spec:
  storageClassName: gp2Encrypted
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

在这个文件上运行`kubectl apply -f`应该会导致创建一个新的自动生成的**持久卷**（**PV**）。如果您的 AWS 云服务提供商设置正确，这将导致创建一个新的类型为 GP2 且启用加密的 EBS 卷。

在将我们的基于 EBS 的持久卷附加到我们的 Pod 之前，让我们确认 EBS 卷在 AWS 中是否正确创建。

为此，我们可以转到 AWS 控制台，并确保我们在运行 EKS 集群的相同区域。然后转到**服务** > **EC2**，在**弹性块存储**下的左侧菜单中单击**卷**。在这一部分，我们应该看到一个与我们的 PVC 状态相同大小（**1 GiB**）的自动生成卷的项目。它应该具有 GP2 的类，并且应该启用加密。让我们看看这在 AWS 控制台中会是什么样子：

![图 7.1 - AWS 控制台自动生成的 EBS 卷](img/B14790_07_001.jpg)

图 7.1 - AWS 控制台自动生成的 EBS 卷

正如您所看到的，我们在 AWS 中正确地创建了我们动态生成的启用加密和分配**gp2**卷类型的 EBS 卷。现在我们已经创建了我们的卷，并且确认它已经在 AWS 中创建，我们可以将它附加到我们的 Pod 上。

## 将持久卷声明（PVC）附加到 Pods

现在我们既有了`PersistentVolume`又有了`PersistentVolumeClaim`，我们可以将它们附加到一个 Pod 以供使用。这个过程与附加 ConfigMap 或 Secret 非常相似 - 这是有道理的，因为 ConfigMaps 和 Secrets 本质上是卷的类型！

查看允许我们将加密的 EBS 卷附加到 Pod 并命名为`pod-with-attachment.yaml`的 YAML：

Pod-with-attachment.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  volumes:
    - name: my-pv
      persistentVolumeClaim:
        claimName: my-pv-claim
  containers:
    - name: my-container
      image: busybox
      volumeMounts:
        - mountPath: "/usr/data"
          name: my-pv
```

运行`kubectl apply -f pod-with-attachment.yaml`将创建一个 Pod，该 Pod 通过我们的声明将我们的`PersistentVolume`挂载到`/usr/data`。

为了确认卷已成功创建，让我们`exec`到我们的 Pod 中，并在我们的卷被挂载的位置创建一个文件：

```
> kubectl exec -it shell-demo -- /bin/bash
> cd /usr/data
> touch myfile.txt
```

现在，让我们使用以下命令删除 Pod：

```
> kubectl delete pod my-pod
```

然后使用以下命令再次重新创建它：

```
> kubectl apply -f my-pod.yaml
```

如果我们做得对，当再次运行`kubectl exec`进入 Pod 时，我们应该能够看到我们的文件：

```
> kubectl exec -it my-pod -- /bin/bash
> ls /usr/data
> myfile.txt
```

成功！

我们现在知道如何为 Kubernetes 创建由云存储提供的持久卷。但是，您可能正在本地环境或使用 minikube 在笔记本电脑上运行 Kubernetes。让我们看看您可以使用的一些替代持久卷子类型。

# 没有云存储的持久卷

我们之前的示例假设您正在云环境中运行 Kubernetes，并且可以使用云平台提供的存储服务（如 AWS EBS 和其他服务）。然而，这并非总是可能的。您可能正在数据中心环境中运行 Kubernetes，或者在专用硬件上运行。

在这种情况下，有许多潜在的解决方案可以为 Kubernetes 提供存储。一个简单的解决方案是将卷类型更改为`hostPath`，它可以在节点现有的存储设备中创建持久卷。例如，在 minikube 上运行时非常适用，但是不像 AWS EBS 那样提供强大的抽象。对于具有类似云存储工具 EBS 的本地功能的工具，让我们看看如何使用 Rook 的 Ceph。有关完整的文档，请查看 Rook 文档（它也会教你 Ceph）[`rook.io/docs/rook/v1.3/ceph-quickstart.html`](https://rook.io/docs/rook/v1.3/ceph-quickstart.html)。

Rook 是一个流行的开源 Kubernetes 存储抽象层。它可以通过各种提供者（如 EdgeFS 和 NFS）提供持久卷。在这种情况下，我们将使用 Ceph，这是一个提供对象、块和文件存储的开源存储项目。为简单起见，我们将使用块模式。

在 Kubernetes 上安装 Rook 实际上非常简单。我们将带您从安装 Rook 到设置 Ceph 集群，最终在我们的集群上提供持久卷。

## 安装 Rook

我们将使用 Rook GitHub 存储库提供的典型 Rook 安装默认设置。这可能会根据用例进行高度定制，但将允许我们快速为我们的工作负载设置块存储。请参考以下步骤来完成这个过程：

1.  首先，让我们克隆 Rook 存储库：

```
> git clone --single-branch --branch master https://github.com/rook/rook.git
> cd cluster/examples/kubernetes/ceph
```

1.  我们的下一步是创建所有相关的 Kubernetes 资源，包括几个**自定义资源定义**（**CRDs**）。我们将在后面的章节中讨论这些，但现在，请将它们视为特定于 Rook 的新 Kubernetes 资源，而不是典型的 Pods、Services 等。要创建常见资源，请运行以下命令：

```
> kubectl apply -f ./common.yaml
```

1.  接下来，让我们启动我们的 Rook 操作员，它将处理为特定的 Rook 提供程序（在本例中将是 Ceph）提供所有必要资源的规划：

```
> kubectl apply -f ./operator.yaml
```

1.  在下一步之前，请确保 Rook 操作员 Pod 实际上正在运行，使用以下命令：

```
> kubectl -n rook-ceph get pod
```

1.  一旦 Rook Pod 处于“运行”状态，我们就可以设置我们的 Ceph 集群！此 YAML 也在我们从 Git 克隆的文件夹中。使用以下命令创建它：

```
> kubectl create -f cluster.yaml
```

这个过程可能需要几分钟。Ceph 集群由几种不同的 Pod 类型组成，包括操作员、**对象存储设备**（**OSDs**）和管理器。

为了确保我们的 Ceph 集群正常工作，Rook 提供了一个工具箱容器映像，允许您使用 Rook 和 Ceph 命令行工具。要启动工具箱，您可以使用 Rook 项目提供的工具箱 Pod 规范，网址为[`rook.io/docs/rook/v0.7/toolbox.html`](https://rook.io/docs/rook/v0.7/toolbox.html)。

这是工具箱 Pod 的规范示例：

rook-toolbox-pod.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: rook-tools
  namespace: rook
spec:
  dnsPolicy: ClusterFirstWithHostNet
  containers:
  - name: rook-tools
    image: rook/toolbox:v0.7.1
    imagePullPolicy: IfNotPresent
```

正如您所看到的，这个 Pod 使用了 Rook 提供的特殊容器映像。该映像预装了您需要调查 Rook 和 Ceph 的所有工具。

一旦您的工具箱 Pod 运行起来，您可以使用`rookctl`和`ceph`命令来检查集群状态（查看 Rook 文档以获取具体信息）。

## rook-ceph-block 存储类

现在我们的集群正在运行，我们可以创建将被 PVs 使用的存储类。我们将称这个存储类为`rook-ceph-block`。这是我们的 YAML 文件（`ceph-rook-combined.yaml`），其中将包括我们的`CephBlockPool`（它将处理 Ceph 中的块存储 - 有关更多信息，请参阅[`rook.io/docs/rook/v0.9/ceph-pool-crd.html`](https://rook.io/docs/rook/v0.9/ceph-pool-crd.html)）以及存储类本身：

ceph-rook-combined.yaml

```
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
    clusterID: rook-ceph
    pool: replicapool
    imageFormat: "2"
currently supports only `layering` feature.
    imageFeatures: layering
    csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
    csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
    csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
csi-provisioner
    csi.storage.k8s.io/fstype: xfs
reclaimPolicy: Delete
```

正如你所看到的，YAML 规范定义了我们的`StorageClass`和`CephBlockPool`资源。正如我们在本章前面提到的，`StorageClass`是我们告诉 Kubernetes 如何满足`PersistentVolumeClaim`的方式。另一方面，`CephBlockPool`资源告诉 Ceph 如何以及在哪里创建分布式存储资源-在这种情况下，要复制多少存储。

现在我们可以给我们的 Pod 一些存储了！让我们使用我们的新存储类创建一个新的 PVC：

rook-ceph-pvc.yaml

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: rook-pvc
spec:
  storageClassName: rook-ceph-block
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

我们的 PVC 是存储类`rook-ceph-block`，因此它将使用我们刚刚创建的新存储类。现在，让我们在 YAML 文件中将 PVC 分配给我们的 Pod：

rook-ceph-pod.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: my-rook-test-pod
spec:
  volumes:
    - name: my-rook-pv
      persistentVolumeClaim:
        claimName: rook-pvc
  containers:
    - name: my-container
      image: busybox
      volumeMounts:
        - mountPath: "/usr/rooktest"
          name: my-rook-pv
```

当 Pod 被创建时，Rook 应该会启动一个新的持久卷并将其附加到 Pod 上。让我们查看一下 Pod，看看它是否正常工作：

```
> kubectl exec -it my-rook-test-pod -- /bin/bash
> cd /usr/rooktest
> touch myfile.txt
> ls
```

我们得到了以下输出：

```
> myfile.txt
```

成功！

尽管我们刚刚使用了 Ceph 的块存储功能，但它也有文件系统模式，这有一些好处-让我们讨论一下为什么你可能想要使用它。

## Rook Ceph 文件系统

Rook 的 Ceph 块提供程序的缺点是一次只能由一个 Pod 进行写入。为了使用 Rook/Ceph 创建一个`ReadWriteMany`持久卷，我们需要使用支持 RWX 模式的文件系统提供程序。有关更多信息，请查看 Rook/Ceph 文档[`rook.io/docs/rook/v1.3/ceph-quickstart.html`](https://rook.io/docs/rook/v1.3/ceph-quickstart.html)。

在创建 Ceph 集群之前，所有先前的步骤都适用。在这一点上，我们需要创建我们的文件系统。让我们使用以下的 YAML 文件来创建它：

rook-ceph-fs.yaml

```
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: ceph-fs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 2
  dataPools:
    - replicated:
        size: 2
  preservePoolsOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true
```

在这种情况下，我们正在复制元数据和数据到至少两个池，以确保可靠性，如在`metadataPool`和`dataPool`块中配置的那样。我们还使用`preservePoolsOnDelete`键在删除时保留池。

接下来，让我们为 Rook/Ceph 文件系统存储专门创建一个新的存储类。以下的 YAML 文件就是这样做的：

rook-ceph-fs-storageclass.yaml

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: ceph-fs
  pool: ceph-fs-data0
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
```

这个`rook-cephfs`存储类指定了我们之前创建的池，并描述了我们存储类的回收策略。最后，它使用了一些在 Rook/Ceph 文档中解释的注释。现在，我们可以通过 PVC 将其附加到一个部署中，而不仅仅是一个 Pod！看一下我们的 PV：

rook-cephfs-pvc.yaml

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: rook-ceph-pvc
spec:
  storageClassName: rook-cephfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

这个持久卷引用了我们的新的`rook-cephfs`存储类，使用`ReadWriteMany`模式 - 我们要求`1 Gi`的数据。接下来，我们可以创建我们的`Deployment`：

rook-cephfs-deployment.yaml

```
apiVersion: v1
kind: Deployment
metadata:
  name: my-rook-fs-test
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25% 
  selector:
    matchLabels:
      app: myapp
  template:
      spec:
  	  volumes:
    	  - name: my-rook-ceph-pv
        persistentVolumeClaim:
          claimName: rook-ceph-pvc
  	  containers:
    	  - name: my-container
         image: busybox
         volumeMounts:
         - mountPath: "/usr/rooktest"
           name: my-rook-ceph-pv
```

这个`Deployment`引用了我们的`ReadWriteMany`持久卷声明，使用`volumes`下的`persistentVolumeClaim`块。部署后，我们所有的 Pod 现在都可以读写同一个持久卷。

之后，您应该对如何创建持久卷并将它们附加到 Pod 有很好的理解。

# 总结

在本章中，我们回顾了在 Kubernetes 上提供存储的两种方法 - 卷和持久卷。首先，我们讨论了这两种方法之间的区别：虽然卷与 Pod 的生命周期相关，但持久卷会持续到它们或集群被删除。然后，我们看了如何实现卷并将它们附加到我们的 Pod。最后，我们将我们对卷的学习扩展到持久卷，并发现了如何使用几种不同类型的持久卷。这些技能将帮助您在许多可能的环境中为您的应用分配持久和非持久的存储 - 从本地到云端。

在下一章中，我们将从应用程序关注点中脱离出来，讨论如何在 Kubernetes 上控制 Pod 的放置。

# 问题

1.  卷和持久卷之间有什么区别？

1.  什么是`StorageClass`，它与卷有什么关系？

1.  在创建 Kubernetes 资源（如持久卷）时，如何自动配置云资源？

1.  在哪些情况下，您认为使用卷而不是持久卷会是禁止的？

# 进一步阅读

请参考以下链接获取更多信息：

+   Rook 的 Ceph 存储快速入门：[`github.com/rook/rook/blob/master/Documentation/ceph-quickstart.md`](https://github.com/rook/rook/blob/master/Documentation/ceph-quickstart.md)

+   Rook 工具箱：[`rook.io/docs/rook/v0.7/toolbox.html`](https://rook.io/docs/rook/v0.7/toolbox.html)

+   云提供商：https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/

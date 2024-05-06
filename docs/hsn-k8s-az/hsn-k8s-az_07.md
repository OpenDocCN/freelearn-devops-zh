# 第五章：5. 在 AKS 中处理常见故障

Kubernetes 是一个具有许多工作部分的分布式系统。AKS 为您抽象了大部分内容，但您仍有责任知道在发生不良情况时应该去哪里寻找以及如何做出响应。Kubernetes 会自动处理大部分故障；然而，您会遇到需要手动干预的情况。

在部署在 AKS 之上的应用程序中，有两个可能出现问题的领域。要么是集群本身出现问题，要么是部署在集群之上的应用程序出现问题。本章专注于集群问题。集群可能出现多种问题。

第一种可能出现的问题是集群中的节点可能变得不可用。这可能是由于 Azure 基础设施故障或虚拟机本身出现问题，例如操作系统崩溃。无论哪种情况，Kubernetes 都会监视集群中的节点故障，并将自动恢复。您将在本章中看到这个过程。

Kubernetes 集群中的第二个常见问题是资源不足的故障。这意味着您尝试部署的工作负载需要的资源超过了集群上可用的资源。您将学习如何监视这些信号以及如何解决这些问题。

另一个常见问题是挂载存储出现问题，这发生在节点变得不可用时。当 Kubernetes 中的节点变得不可用时，Kubernetes 不会分离附加到此失败节点的磁盘。这意味着这些磁盘不能被其他节点上的工作负载使用。您将看到一个实际的例子，并学习如何从这种故障中恢复。

在本章中，我们将深入研究以下故障模式：

+   处理节点故障

+   解决资源不足的故障

+   处理存储挂载问题

在这一章中，您将了解常见的故障场景，以及针对这些场景的解决方案。首先，我们将介绍节点故障。

#### 注意：

参考*Kubernetes the Hard Way*（[`github.com/kelseyhightower/kubernetes-the-hard-way`](https://github.com/kelseyhightower/kubernetes-the-hard-way)），一个优秀的教程，了解 Kubernetes 构建的基础。对于 Azure 版本，请参考*Kubernetes the Hard Way – Azure Translation*（[`github.com/ivanfioravanti/kubernetes-the-hard-way-on-azure`](https://github.com/ivanfioravanti/kubernetes-the-hard-way-on-azure)）。

## 处理节点故障

有意（为了节省成本）或无意中，节点可能会宕机。当这种情况发生时，您不希望在凌晨 3 点接到系统宕机的电话。Kubernetes 可以自动处理节点故障时的工作负载迁移。在这个练习中，我们将部署 guestbook 应用程序，并将在我们的集群中关闭一个节点，看看 Kubernetes 的响应是什么：

1.  确保您的集群至少有两个节点：

```
kubectl get nodes
```

这应该生成一个如*图 5.1*所示的输出：

![执行 kubectl get nodes 命令会显示一个带有两个节点的输出。这两个节点的状态为 Ready。](img/Figure_5.1.jpg)

###### 图 5.1：确保您的集群中有两个正在运行的节点

如果您的集群中没有两个节点，请在 Azure 门户中查找您的集群，导航到**节点池**，然后单击**节点计数**。您可以将其扩展到**2**个节点，如*图 5.2*所示：

![单击 Azure 门户左侧的导航窗格中的节点池选项卡。这将显示给您几个选项。转到节点计数选项。单击它以将此计数扩展到两个节点。](img/Figure_5.2.jpg)

###### 图 5.2：扩展集群

1.  作为本章的示例应用程序，我们将使用 guestbook 应用程序。部署此应用程序的 YAML 文件已在本章的源代码中提供（`guestbook-all-in-one.yaml`）。要部署 guestbook 应用程序，请使用以下命令：

```
kubectl create -f guestbook-all-in-one.yaml
```

1.  我们将再次为服务提供公共 IP，就像在之前的章节中一样。要开始编辑，请执行以下命令：

```
kubectl edit service frontend
```

1.  这将打开一个`vi`环境。导航到现在显示为`type: ClusterIP`（第 27 行）的行，并将其更改为`type: LoadBalancer`。要进行更改，请按*I*按钮进入插入模式，输入更改，然后按*Esc*按钮，输入`:wq!`，然后按*Enter*保存更改。

1.  更改保存后，您可以观察`service`对象，直到公共 IP 变为可用。要做到这一点，请输入以下内容：

```
kubectl get svc -w
```

1.  这将花费几分钟的时间来显示更新后的 IP。*图 5.3*代表了服务的公共 IP。一旦您看到正确的公共 IP，您可以通过按*Ctrl* + *C*（Mac 上为*command* + *C*）退出 watch 命令：![输出显示前端服务将其外部 IP 从<pending>更改为实际 IP。](img/Figure_5.3.jpg)

###### 图 5.3：获取服务的公共 IP

1.  转到`http://<EXTERNAL-IP>`，如*图 5.4*所示：![一旦在浏览器的地址栏中输入公共 IP，它将打开一个带有粗体字“Guestbook”的白屏。这表明您的应用程序现在正在运行。](img/Figure_5.4.jpg)

###### 图 5.4：确保应用程序正在运行

1.  让我们看看当前正在运行的 Pods 使用以下代码：

```
kubectl get pods -o wide
```

这将生成如*图 5.5*所示的输出：

![当您执行 kubectl get pods -o wide 命令时，显示的输出将显示 Pods 分布在节点 0 和 1 之间。](img/Figure_5.5.jpg)

###### 图 5.5：我们的 Pods 分布在节点 0 和节点 1 之间

这显示我们的工作负载分布在节点 0 和节点 1 之间。

1.  在这个例子中，我们想演示 Kubernetes 如何处理节点故障。为了演示这一点，我们将关闭集群中的一个节点。在这种情况下，我们要造成最大的破坏，所以让我们关闭节点 1（您可以选择任何节点 - 出于说明目的，这并不重要）：

要关闭此节点，请在 Azure 搜索栏中查找我们集群中的`虚拟机规模集`，如*图 5.6*所示：

请在 Azure 搜索栏中键入 vmss 以关闭节点。

###### 图 5.6：查找托管集群的 VMSS

导航到规模集的刀片后，转到**实例**视图，选择要关闭的实例，然后点击**分配**按钮，如*图 5.7*所示：

![单击 Azure 门户中导航窗格中的“实例”选项卡。这将显示实例的数量。选择要关闭的实例。单击出现在地址栏中的取消分配图标。](img/Figure_5.7.jpg)

###### 图 5.7：关闭节点 1

这将关闭我们的节点。要查看 Kubernetes 如何与您的 Pods 交互，可以通过以下命令观察集群中的 Pods：

```
kubectl get pods -o wide -w
```

1.  要验证您的应用程序是否可以继续运行，可以选择运行以下命令，每 5 秒钟点击一次 guestbook 前端并获取 HTML。建议在新的 Cloud Shell 窗口中打开此命令：

```
while true; do curl -m 1 http://<EXTERNAl-IP>/ ; sleep 5; done
```

#### 注意

上述命令将一直调用您的应用程序，直到您按下*Ctrl* + *C*（Mac 上的*command* + *C*）。可能会有间歇时间您收不到回复，这是可以预期的，因为 Kubernetes 需要几分钟来重新平衡系统。

添加一些留言板条目，看看当你导致节点关闭时它们会发生什么。这将显示一个如*图 5.8*所示的输出：

![使用留言板应用程序写下几条消息。](img/Figure_5.8.jpg)

###### 图 5.8：在留言板中写下几条消息

你所看到的是你所有珍贵的消息都消失了！这显示了在节点故障的情况下拥有**PersistentVolumeClaims**（**PVCs**）对于任何你希望存活的数据的重要性，在我们的应用程序中并非如此。你将在本章的最后一节中看到一个例子。

1.  过一会儿，观察 Pods 的输出应该显示额外的输出，告诉你 Pods 已经在健康的主机上重新调度，如*图 5.9*所示：![这将显示一个输出，显示了从失败节点重新创建的 Pods。](img/Figure_5.9.jpg)

###### 图 5.9：从失败节点重新创建的 Pods

你在这里看到的是：

+   一个前端 Pod 在主机 1 上运行时被终止，因为主机变得不健康。

+   一个新的前端 Pod 被创建在主机 0 上。这经历了**Pending**、**ContainerCreating**，然后是**Running**这些阶段。

#### 注意

Kubernetes 在重新调度 Pods 之前察觉到主机不健康。如果你运行`kubectl get nodes`，你会看到节点 1 处于`NotReady`状态。Kubernetes 中有一个叫做 pod-eviction-timeout 的配置，它定义了系统等待在健康主机上重新调度 Pods 的时间。默认值是 5 分钟。

在本节中，你学会了 Kubernetes 如何自动处理节点故障，通过在健康节点上重新创建 Pods。在下一节中，你将学习如何诊断和解决资源耗尽的问题。

## 解决资源耗尽的问题

Kubernetes 集群可能出现的另一个常见问题是集群资源耗尽。当集群没有足够的 CPU 或内存来调度额外的 Pods 时，Pods 将被卡在`Pending`状态。

Kubernetes 使用`requests`来计算某个 Pod 需要多少 CPU 或内存。我们的留言板应用程序为所有部署定义了请求。如果你打开`guestbook-all-in-one.yaml`文件，你会看到`redis-slave`部署的以下内容：

```
...
63  kind: Deployment
64  metadata:
65    name: redis-slave
...
83          resources:
84            requests:
85              cpu: 100m
86              memory: 100Mi
...
```

本节解释了`redis-slave`部署的每个 Pod 都需要`100m`的 CPU 核心（100 毫或 10%）和 100MiB（Mebibyte）的内存。在我们的 1 个 CPU 集群（关闭节点 1），将其扩展到 10 个 Pods 将导致可用资源出现问题。让我们来看看这个：

#### 注意

在 Kubernetes 中，您可以使用二进制前缀表示法或基数 10 表示法来指定内存和存储。二进制前缀表示法意味着使用 KiB（kibibyte）表示 1024 字节，MiB（mebibyte）表示 1024 KiB，Gib（gibibyte）表示 1024 MiB。基数 10 表示法意味着使用 kB（kilobyte）表示 1000 字节，MB（megabyte）表示 1000 kB，GB（gigabyte）表示 1000 MB。

1.  让我们首先将`redis-slave`部署扩展到 10 个 Pods：

```
kubectl scale deployment/redis-slave --replicas=10
```

1.  这将导致创建一对新的 Pod。我们可以使用以下命令来检查我们的 Pods：

```
kubectl get pods
```

这将生成以下输出：

![使用 kubectl get pods 命令，您可以检查您的 Pods。输出将生成一些状态为 Pending 的 Pods。](img/Figure_5.10.jpg)

###### 图 5.10：如果集群资源不足，Pods 将进入 Pending 状态

这里突出显示了一个处于`Pending`状态的 Pod。

1.  我们可以使用以下命令获取有关这些待处理 Pods 的更多信息：

```
kubectl describe pod redis-slave-<pod-id>
```

这将显示更多细节。在`describe`命令的底部，您应该看到类似于*图 5.11*中显示的内容：

![输出显示了来自 default-scheduler 的 FailedSchedulingmessage 事件。详细消息显示“0/2 个节点可用：1 个 CPU 不足，1 个节点有 Pod 无法容忍的污点。”](img/Figure_5.11.jpg)

###### 图 5.11：Kubernetes 无法安排此 Pod

它向我们解释了两件事：

+   其中一个节点的 CPU 资源已经用完。

+   其中一个节点有一个 Pod 无法容忍的污点。这意味着`NotReady`的节点无法接受 Pods。

1.  我们可以通过像*图 5.12*中所示的方式启动节点 1 来解决这个容量问题。这可以通过类似于关闭过程的方式完成：![您可以使用与关闭相同的过程来启动节点。单击导航窗格中的实例选项卡。选择要启动的节点。最后，单击工具栏中的“启动”按钮。](img/Figure_5.12.jpg)

###### 图 5.12：重新启动节点 1

1.  其他节点再次在 Kubernetes 中变得可用需要几分钟的时间。如果我们重新执行先前 Pod 上的`describe`命令，我们将看到类似于*图 5.13*所示的输出：![描述 Pod 的事件输出显示，经过一段时间，调度程序将 Pod 分配给新节点，并显示拉取图像的过程，以及创建和启动容器的过程。](img/Figure_5.13.jpg)

###### 图 5.13：当节点再次可用时，其他 Pod 将在新节点上启动。

这表明在节点 1 再次变得可用后，Kubernetes 将我们的 Pod 调度到该节点，然后启动容器。

在本节中，我们学习了如何诊断资源不足的错误。我们通过向集群添加另一个节点来解决了这个错误。在我们继续进行最终故障模式之前，我们将清理一下 guestbook 部署。

#### 注意

在*第四章*中，*扩展您的应用程序*，我们讨论了集群自动缩放器。集群自动缩放器将监视资源不足的错误，并自动向集群添加新节点。

让我们通过运行以下`delete`命令来清理一下：

```
kubectl delete -f guestbook-all-in-one.yaml
```

到目前为止，我们已经讨论了 Kubernetes 集群中节点的两种故障模式。首先，我们讨论了 Kubernetes 如何处理节点离线以及系统如何将 Pod 重新调度到工作节点上。之后，我们看到了 Kubernetes 如何使用请求来调度节点上的 Pod，以及当集群资源不足时会发生什么。在接下来的部分，我们将讨论 Kubernetes 中的另一种故障模式，即当 Kubernetes 移动带有 PVC 的 Pod 时会发生什么。

## 修复存储挂载问题

在本章的前面，您注意到当 Redis 主节点移动到另一个节点时，guestbook 应用程序丢失了数据。这是因为该示例应用程序没有使用任何持久存储。在本节中，我们将介绍当 Kubernetes 将 Pod 移动到另一个节点时，如何使用 PVC 来防止数据丢失的示例。我们将向您展示 Kubernetes 移动带有 PVC 的 Pod 时会发生的常见错误，并向您展示如何修复此错误。

为此，我们将重用上一章中的 WordPress 示例。在开始之前，让我们确保集群处于干净的状态：

```
kubectl get all
```

这向我们展示了一个 Kubernetes 服务，如*图 5.14*所示：

![执行 kubectl get all 命令会生成一个输出，显示目前只有一个 Kubernetes 服务正在运行。](img/Figure_5.14.jpg)

###### 图 5.14：目前您应该只运行 Kubernetes 服务

让我们还确保两个节点都在运行并且处于“就绪”状态：

```
kubectl get nodes
```

这应该显示我们两个节点都处于“就绪”状态，如*图 5.15*所示：

![现在您应该看到两个节点的状态都是 Ready。](img/Figure_5.15.jpg)

###### 图 5.15：您的集群中应该有两个可用节点

在前面的示例中，在*处理节点故障*部分，我们看到如果 Pod 重新启动，存储在`redis-master`中的消息将丢失。原因是`redis-master`将所有数据存储在其容器中，每当重新启动时，它都会使用不带数据的干净镜像。为了在重新启动后保留数据，数据必须存储在外部。Kubernetes 使用 PVC 来抽象底层存储提供程序，以提供这种外部存储。

要开始此示例，我们将设置 WordPress 安装。

### 开始 WordPress 安装

让我们从安装 WordPress 开始。我们将演示其工作原理，然后验证在重新启动后存储是否仍然存在：

使用以下命令开始重新安装：

```
helm install wp stable/wordpress 
```

这将花费几分钟的时间来处理。您可以通过执行以下命令来跟踪此安装的状态：

```
kubectl get pods -w
```

几分钟后，这应该显示我们的 Pod 状态为`Running`，并且两个 Pod 的就绪状态为`1/1`，如*图 5.16*所示：

![使用 kubectl get pods -w 命令，您将看到 Pod 从 ContainerCreating 转换为 Running 状态，并且您将看到 Ready pods 的数量从 0/1 变为 1/1。](img/Figure_5.16.jpg)

###### 图 5.16：几分钟后，所有 Pod 都将显示为运行状态

在本节中，我们看到了如何安装 WordPress。现在，我们将看到如何使用持久卷来避免数据丢失。

### 使用持久卷来避免数据丢失

**持久卷**（**PV**）是在 Kubernetes 集群中存储持久数据的方法。我们在*第三章*的*在 AKS 上部署应用程序*中更详细地解释了 PV。让我们探索为 WordPress 部署创建的 PV：

1.  在我们的情况下，运行以下`describe nodes`命令：

```
kubectl describe nodes
```

滚动查看输出，直到看到类似于*图 5.17*的部分。在我们的情况下，两个 WordPress Pod 都在节点 0 上运行：

![当您执行 kubectl describe nodes 命令时，您将看到指示两个 Pod 正在节点 0 上运行的信息。](img/Figure_5.17.jpg)

###### 图 5.17：在我们的情况下，两个 WordPress Pod 都在节点 0 上运行

您的 Pod 放置可能会有所不同。

1.  我们可以检查的下一件事是我们的 PVC 的状态：

```
kubectl get pvc
```

这将生成一个如*图 5.18*所示的输出：

![输出显示了两个 PVC。除了它们的名称，您还可以看到这些 PVC 的状态、卷、容量和访问模式。](img/Figure_5.18.jpg)

###### 图 5.18：WordPress 部署创建了两个 PVC

以下命令显示了绑定到 Pod 的实际 PV：

```
kubectl describe pv
```

这将向您显示两个卷的详细信息。我们将在*图 5.19*中向您展示其中一个：

使用 kubectl describe pvc 命令，您可以详细查看两个 PVC。图片突出显示了默认/data-wp-mariadb-0 的声明，并突出显示了 Azure 中的 diskURI。![使用 kubectl describe pvc 命令，您可以详细查看两个 PVC。图片突出显示了默认/data-wp-mariadb-0 的声明，并突出显示了 Azure 中的 diskURI。](img/Figure_5.19.jpg)

###### 图 5.19：一个 PVC 的详细信息

在这里，我们可以看到哪个 Pod 声明了这个卷，以及 Azure 中的**DiskURI**是什么。

1.  验证您的网站是否实际在运行：

```
kubectl get service
```

这将显示我们的 WordPress 网站的公共 IP，如*图 5.20*所示：

![输出屏幕仅显示了 wp-WordPress 服务的 External-IP。](img/Figure_5.20.jpg)

###### 图 5.20：获取服务的公共 IP

1.  如果您还记得*第三章*，*AKS 的应用部署*，Helm 向我们显示了获取 WordPress 网站管理员凭据所需的命令。让我们获取这些命令并执行它们以登录到网站，如下所示：

```
helm status wp
echo Username: user
echo Password: $(kubectl get secret --namespace default wp-wordpress -o jsonpath="{.data.wordpress-password}" | base64 -d)
```

这将向您显示`用户名`和`密码`，如*图 5.21*所示：

![输出显示如何通过 Helm 获取用户名和密码。截图中的用户名是 user，密码是 lcsUSJTk8e。在您的情况下，密码将不同。](img/Figure_5.21.jpg)

###### 图 5.21：获取 WordPress 应用程序的用户名和密码

#### 注意

您可能会注意到，我们在书中使用的命令与`helm`返回的命令略有不同。`helm`为密码返回的命令对我们不起作用，我们为您提供了有效的命令。

我们可以通过以下地址登录到我们的网站：`http://<external-ip>/admin`。在这里使用前一步骤中的凭据登录。然后，您可以继续添加一篇文章到您的网站。点击**撰写您的第一篇博客文章**按钮，然后创建一篇简短的文章，如*图 5.22*所示：

![您将看到一个欢迎您来到 WordPress 的仪表板。在这里，您将看到一个按钮，上面写着-写下您的第一篇博客文章。单击它开始写作。](img/Figure_5.22.jpg)

###### 图 5.22：撰写您的第一篇博客文章

现在输入一些文本，然后单击**发布**按钮，就像*图 5.23*中所示。文本本身并不重要；我们写这个来验证数据确实被保留到磁盘上：

![假设您随机输入了单词“测试”。在写完文字后，单击屏幕右上角的“发布”按钮。](img/Figure_5.23.jpg)

###### 图 5.23：发布包含随机文本的文章

如果您现在转到您网站的主页`http://<external-ip>`，您将会看到您的测试文章，就像*图 5.23*中所示。我们将在下一节验证此文章是否经得起重启。

**处理涉及 PVC 的 Pod 故障**

我们将对我们的 PVC 进行的第一个测试是杀死 Pod 并验证数据是否确实被保留。为此，让我们做两件事：

1.  **观察我们应用程序中的 Pod**。为此，我们将使用当前的 Cloud Shell 并执行以下命令：

```
kubectl get pods -w
```

1.  杀死已挂载 PVC 的两个 Pod。为此，我们将通过单击工具栏上显示的图标创建一个新的 Cloud Shell 窗口，如*图 5.24*所示：![单击工具栏左侧花括号图标旁边的图标，单击以打开新的 Cloud Shell。](img/Figure_5.24.jpg)

###### 图 24：打开新的 Cloud Shell 实例

一旦您打开一个新的 Cloud Shell，执行以下命令：

```
kubectl delete pod --all
```

如果您使用`watch`命令，您应该会看到类似于*图 5.25*所示的输出：

![运行 kubectl get pods -w 会显示旧的 Pod 被终止并创建新的 Pod。新的 Pod 会从 Pending 状态过渡到 ContainerCreating 再到 Running。](img/Figure_5.25.jpg)

###### 图 5.25：删除 Pod 后，Kubernetes 将自动重新创建两个 Pod

正如您所看到的，Kubernetes 迅速开始创建新的 Pod 来从 Pod 故障中恢复。这些 Pod 经历了与原始 Pod 相似的生命周期，从**Pending**到**ContainerCreating**再到**Running**。

1.  如果您转到您的网站，您应该会看到您的演示文章已经被保留。这就是 PVC 如何帮助您防止数据丢失的方式，因为它们保留了容器本身无法保留的数据。

*图 5.26*显示，即使 Pod 被重新创建，博客文章仍然被保留：

![您可以看到您的数据被持久保存，带有“test”字样的博客帖子仍然可用。](img/Figure_5.26.jpg)

###### 图 5.26：您的数据被持久保存，您的博客帖子仍然存在

在这里观察的最后一个有趣的数据点是 Kubernetes 事件流。如果运行以下命令，您可以看到与卷相关的事件：

```
kubectl get events | grep -i volume
```

这将生成如*图 5.27*所示的输出。

![输出屏幕将显示一个 FailedAttachVolume 警告。在其下方，它将显示状态现在正常，并带有 SuccessfulAttachVolume 消息。这表明 Kubernetes 最初无法挂载卷，但在下一次尝试时成功挂载了它。](img/Figure_5.27.jpg)

###### 图 5.27：Kubernetes 处理了 FailedAttachVolume 错误

这显示了与卷相关的事件。有两条有趣的消息需要详细解释：`FailedAttachVolume`和`SuccessfulAttachVolume`。这向我们展示了 Kubernetes 如何处理具有 read-write-once 配置的卷。由于特性是只能从单个 Pod 中读取和写入，Kubernetes 只会在成功从当前 Pod 卸载卷后，将卷挂载到新的 Pod 上。因此，最初，当新的 Pod 被调度时，它显示了`FailedAttachVolume`消息，因为卷仍然附加到正在删除的 Pod 上。之后，Pod 成功挂载了卷，并通过`SuccessfulAttachVolume`消息显示了这一点。

在本节中，我们已经学习了 PVC 在 Pod 在同一节点上重新创建时可以起到的作用。在下一节中，我们将看到当节点发生故障时 PVC 的使用情况。

**使用 PVC 处理节点故障**

在前面的示例中，我们看到了 Kubernetes 如何处理具有 PV 附加的 Pod 故障。在这个示例中，我们将看看 Kubernetes 在卷附加时如何处理节点故障：

1.  让我们首先检查哪个节点托管了我们的应用程序，使用以下命令：

```
kubectl get pods -o wide
```

我们发现，在我们的集群中，节点 1 托管了 MariaDB，节点 0 托管了 WordPress 网站，如*图 5.28*所示：

![运行 kubectl get pods -o wide 会显示哪个 pod 在哪个节点上运行。在截图中，一个 pod 在每个主机上运行。](img/Figure_5.28.jpg)

###### 图 5.28：我们的部署中有两个正在运行的 Pod - 一个在节点 1 上，一个在节点 0 上

1.  我们将引入一个故障，并停止可能会造成最严重损害的节点，即关闭 Azure 门户上的节点 0。我们将以与先前示例相同的方式进行。首先，查找支持我们集群的规模集，如*图 5.29*所示：![在 Azure 门户的搜索栏中键入 vmss 将显示托管您 AKS 集群的 VMSS 的完整名称。](img/Figure_5.29.jpg)

###### 图 5.29：查找支持我们集群的规模集

1.  然后按照*图 5.30*中所示关闭节点：![要关闭节点，请在 Azure 门户中导航窗格中的“实例”选项卡上单击。您将看到两个节点。选择第一个节点，然后单击工具栏上的“停用”按钮。](img/Figure_5.30.jpg)

###### 图 5.30：关闭节点

1.  完成此操作后，我们将再次观察我们的 Pod，以了解集群中正在发生的情况：

```
kubectl get pods -o wide -w
```

与先前的示例一样，Kubernetes 将在 5 分钟后开始采取行动来应对我们失败的节点。我们可以在*图 5.31*中看到这一情况：

![当您执行 kubectl get pods -o wide -w 命令时，您将看到状态为 Pending 的 Pod 未分配到节点。](img/Figure_5.31.jpg)

###### 图 5.31：处于挂起状态的 Pod

1.  我们在这里遇到了一个新问题。我们的新 Pod 处于“挂起”状态，尚未分配到新节点。让我们弄清楚这里发生了什么。首先，我们将“描述”我们的 Pod：

```
kubectl describe pods/wp-wordpress-<pod-id>
```

您将得到一个如*图 5.32*所示的输出：

![您将看到此 Pod 处于挂起状态的原因的详细信息，这是由于 CPU 不足造成的。](img/Figure_5.32.jpg)

###### 图 5.32：显示处于挂起状态的 Pod 的输出

1.  这表明我们的集群中没有足够的 CPU 资源来托管新的 Pod。我们可以使用`kubectl edit deploy/...`命令来修复任何不足的 CPU/内存错误。我们将将 CPU 请求从 300 更改为 3，以便我们的示例继续进行：

```
kubectl edit deploy wp-wordpress
```

这将带我们进入一个`vi`环境。我们可以通过输入以下内容快速找到与 CPU 相关的部分：

```
/cpu <enter>
```

一旦到达那里，将光标移动到两个零上，然后按两次`x`键删除零。最后，键入`:wq!`以保存我们的更改，以便我们可以继续我们的示例。

1.  这将导致创建一个新的 ReplicaSet 和一个新的 Pod。我们可以通过输入以下命令来获取新 Pod 的名称：

```
kubectl get pods 
```

查找状态为`ContainerCreating`的 Pod，如下所示：

![输出屏幕显示了四个具有不同状态的 Pod。查找具有 ContainerCreating 状态的 Pod。](img/Figure_5.33.jpg)

###### 图 5.33：新的 Pod 被卡在 ContainerCreating 状态

1.  让我们用`describe`命令查看该 Pod 的详细信息：

```
kubectl describe pod wp-wordpress-<pod-id>
```

在这个`describe`输出的“事件”部分，您可以看到以下错误消息：

![在具有 ContainerCreating 状态的 Pod 上执行 kubectl describe 命令会显示一个详细的错误消息，其中包含 FailedMount 的原因。消息表示 Kubernetes 无法挂载卷。](img/Figure_5.34.jpg)

###### 图 5.34：新的 Pod 有一个新的错误消息，描述了卷挂载问题

1.  这告诉我们，我们的新 Pod 想要挂载的卷仍然挂载到了被卡在“终止”状态的 Pod 上。我们可以通过手动从我们关闭的节点上分离磁盘并强制删除被卡在“终止”状态的 Pod 来解决这个问题。

#### 注意

处于“终止”状态的 Pod 的行为不是一个错误。这是默认的 Kubernetes 行为。Kubernetes 文档中指出：“Kubernetes（1.5 版本或更新版本）不会仅仅因为节点不可达而删除 Pods。在不可达节点上运行的 Pods 在超时后进入“终止”或“未知”状态。当用户尝试在不可达节点上优雅地删除 Pod 时，Pods 也可能进入这些状态。”您可以在[`kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/`](https://kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/)阅读更多内容。

1.  为此，我们需要规模集的名称和资源组的名称。要找到这些信息，请在门户中查找规模集，如*图 5.35*所示：![在 Azure 搜索栏中键入 vmss 以查找支持您集群的 ScaleSet](img/Figure_5.35.jpg)

###### 图 5.35：查找支持您集群的规模集

1.  在规模集视图中，复制并粘贴规模集名称和资源组。编辑以下命令以从失败的节点分离磁盘，然后在 Cloud Shell 中运行此命令：

```
az vmss disk detach --lun 0 --vmss-name <vmss-name> -g <rgname> --instance-id 0
```

1.  这将从节点 0 中分离磁盘。这里需要的第二步是在 Pod 被卡在终止状态时，强制将其从集群中移除：

```
kubectl delete pod wordpress-wp-<pod-id> --grace-period=0 --force
```

1.  这将使我们的新 Pod 恢复到健康状态。系统需要几分钟来接受更改，然后挂载和调度新的 Pod。让我们再次使用以下命令获取 Pod 的详细信息：

```
kubectl describe pod wp-wordpress-<pod-id>
```

这将生成以下输出：

![输出将显示两个 Pod 的事件类型为 Normal。原因是 SuccessfulAttachVolume 和 Pulled 的第二个 Pod。](img/Figure_5.36.jpg)

###### 图 5.36：我们的新 Pod 现在正在挂载卷并拉取容器镜像

1.  这表明新的 Pod 成功挂载了卷，并且容器镜像已被拉取。让我们验证一下 Pod 是否真的在运行：

```
kubectl get pods
```

这将显示 Pod 正在运行，如*图 5.37*所示：

![两个 Pod 的状态为 Running。](img/Figure_5.37.jpg)

###### 图 5.37：两个 Pod 都成功运行

这将使您的 WordPress 网站再次可用。

在继续之前，让我们使用以下命令清理我们的部署：

```
helm delete wp
kubectl delete pvc --all
kubectl delete pv --all
```

让我们还重新启动关闭的节点，如*图 5.38*所示：

![要重新启动关闭的节点，请单击导航窗格中的实例选项卡。选择已分配/已停止的节点。单击搜索栏旁边的工具栏中的启动按钮。您的节点现在应该已重新启动。](img/Figure_5.38.jpg)

###### 图 5.38：重新启动节点 1

在本节中，我们介绍了当 PVC 未挂载到新的 Pod 时，您如何从节点故障中恢复。我们需要手动卸载磁盘，然后强制删除处于`Terminating`状态的 Pod。

## 总结

在本章中，您了解了常见的 Kubernetes 故障模式以及如何从中恢复。我们从一个示例开始，介绍了 Kubernetes 如何自动检测节点故障并启动新的 Pod 来恢复工作负载。之后，您扩展了工作负载，导致集群资源耗尽。您通过重新启动故障节点来为集群添加新资源，从这种情况中恢复了过来。

接下来，您看到了 PV 是如何有用地将数据存储在 Pod 之外的。您关闭了集群上的所有 Pod，并看到 PV 确保应用程序中没有数据丢失。在本章的最后一个示例中，您看到了当 PV 被附加时，您如何从节点故障中恢复。您通过从节点卸载磁盘并强制删除终止的 Pod 来恢复工作负载。这将使您的工作负载恢复到健康状态。

本章已经解释了 Kubernetes 中常见的故障模式。在下一章中，我们将为我们的服务引入 HTTPS 支持，并介绍与 Azure 活动目录的身份验证。

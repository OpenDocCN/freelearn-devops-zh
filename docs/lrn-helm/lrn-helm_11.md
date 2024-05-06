# 第八章：使用 Helm 与操作员框架

使用 Helm 的一个优势是能够同步本地和实时状态。使用 Helm，本地状态是通过值文件进行管理的，当使用`install`或`upgrade`命令提供这些值时，将这些值应用于 Kubernetes 集群中的实时状态以进行同步。在之前的章节中，当希望对应用程序进行更改时，通过调用这些命令来执行此操作。

另一种同步这些更改的方法是在集群内创建一个应用程序，定期检查期望状态是否与环境中的当前配置匹配。如果状态不匹配，应用程序可以自动修改环境以匹配期望的状态。这种应用程序被称为 Kubernetes 操作员。在本章中，我们将创建一个基于 Helm 的操作员，以确保本地定义的状态始终与集群的实时状态匹配。如果不匹配，操作员将执行适当的 Helm 命令来更新环境。

本章将涵盖以下主题：

+   理解 Kubernetes 操作员

+   创建一个 Helm 操作员

+   使用 Helm 来管理操作员和自定义资源（CRs）

+   清理您的 Kubernetes 环境

# 技术要求

对于本章，您需要在本地机器上安装以下技术：

+   `minikube`

+   `helm`

+   `kubectl`

除了这些工具之外，您还应该在 GitHub 上找到 Packt 存储库，其中包含与示例相关的资源，网址为[`github.com/PacktPublishing/-Learn-Helm`](https://github.com/PacktPublishing/-Learn-Helm)。本存储库将在本章中被引用。

# 理解 Kubernetes 操作员

自动化是 Kubernetes 平台的核心。正如在*第一章*中所介绍的，*了解 Kubernetes 和 Helm*，Kubernetes 资源可以通过运行`kubectl`命令隐式管理，也可以通过应用 YAML 格式的表示来声明性地管理。一旦使用 Kubernetes 命令行界面（CLI）应用了资源，Kubernetes 的基本原则之一是将集群中资源的当前状态与期望状态匹配，这个过程称为**控制循环**。这种持续的、非终止的监视集群状态的模式是通过控制器实现的。Kubernetes 包括许多本地于平台的控制器，例如拦截对 Kubernetes 应用程序编程接口（API）的请求的准入控制器，以及管理运行的 Pod 副本数量的复制控制器。

随着对 Kubernetes 的兴趣开始增长，提供用户扩展基础平台功能的能力，以及提供更多关于管理应用程序生命周期的智能的组合，导致了几个重要概念的产生，这些概念定义了 Kubernetes 开发的第二波。首先，引入了自定义资源定义（CRD），使用户能够扩展默认的 Kubernetes API，这是与 Kubernetes 平台交互的机制，以创建和注册新类型的资源。注册新的 CRD 会在 Kubernetes API 服务器上创建一个新的 RESTful 资源路径。因此，类似于您可以使用 Kubernetes CLI 执行`kubectl get pods`来检索所有 Pod 对象，例如，为名为**Guestbook**的对象类型注册一个新的 CRD，允许调用`kubectl get guestbook`来查看先前创建的所有 Guestbook 对象。有了这种新的能力，开发人员现在可以创建自己的控制器来监视这些类型的 CR，以管理可以通过 CRD 描述的应用程序的生命周期。

第二个主要趋势是 Kubernetes 部署的应用程序类型的进展。与小型简单的应用程序不同，更复杂和有状态的应用程序被部署得更频繁。这些高级应用程序通常需要更高级的管理和维护水平，例如处理多个组件的部署，以及围绕“第二天”活动的考虑，如备份和恢复。这些任务超出了 Kubernetes 中典型控制器的范围，因为必须嵌入与其管理的应用程序相关的深层知识。使用 CR 来管理应用程序及其组件的这种模式被称为**Operator**模式。由软件公司 CoreOS 在 2016 年首次提出，Operators 旨在捕获人类操作员在管理应用程序生命周期方面的知识。Operators 被打包为普通的容器化应用程序——部署在 pod 中——对 CR 的 API 更改做出反应。

Operators 通常使用称为 Operator Framework 的工具包编写，并基于以下三种不同的技术之一：

+   Go

+   Ansible

+   Helm

基于 Go 的 Operators 利用 Go 编程语言实现控制循环逻辑。基于 Ansible 的 Operators 利用 Ansible CLI 工具和 Ansible playbooks。Ansible 是一种自动化工具，其逻辑是在称为 playbooks 的 YAML 文件中编写的。

在本章中，我们将专注于基于 Helm 的 Operators。Helm Operators 将其控制循环逻辑基于 Helm 图表和 Helm CLI 提供的一部分功能。因此，它们代表了 Helm 用户实现其 Operators 的一种简单方式。

了解了 Operators，让我们使用 Helm 创建自己的 operator。

# 创建一个 Helm operator

在本节中，我们将编写一个基于 Helm 的 operator，用于安装*第五章*中创建的 Guestbook Helm 图表，*构建您的第一个 Helm 图表*。该图表可以在 Pack[t 存储库的`guestbook/`文件夹下找到（https://github.com/PacktPublishing/-Learn-Helm/tree/master/helm-charts/ch](https://github.com/PacktPublishing/-Learn-Helm/tree/master/helm-charts/charts/guestbook)arts/guestbook）。

操作员是作为一个包含控制循环逻辑以维护应用程序的容器镜像构建的。下图演示了访客留言簿操作员部署后的功能：

![图 8.1 - 访客留言簿操作员工作流](img/Figure_8.1.jpg)

图 8.1 - 访客留言簿操作员工作流

访客留言簿操作员将不断监视访客留言簿 CR 的更改。当创建访客留言簿 CR 时，访客留言簿操作员将安装您在*第五章*中创建的访客留言簿图表，*构建您的第一个 Helm 图表*。相反，如果删除了访客留言簿 CR，访客留言簿操作员将删除访客留言簿 Helm 图表。

了解访客留言簿操作员的功能后，让我们设置一个可以构建和部署操作员的环境。

## 设置环境

首先，由于操作员将部署到 Kubernetes，您应该通过运行以下命令来启动 Minikube 环境：

```
$ minikube start
```

启动 Minikube 后，创建一个名为`chapter8`的命名空间，如下所示：

```
$ kubectl create ns chapter8
```

由于访客留言簿操作员是作为一个容器镜像构建的，您需要创建一个可以存储它以便以后引用的镜像存储库。为了存储这个镜像，我们将在 Quay（quay.io）中创建一个新的存储库，这是一个公共容器注册表（如果您在其他地方有帐户，那也可以）。我们还将准备一个本地开发环境，其中包含构建操作员镜像所需的必要工具。

让我们从在 Quay 中创建一个新的镜像存储库开始。

### 创建 Quay 存储库

在 Quay 中创建一个新的存储库需要您拥有一个 Quay 帐户。按照以下步骤创建一个 Quay 帐户：[nt:](https://quay.io/signin/)

1.  [在浏览器中导航到 https:/](https://quay.io/signin/)/quay.io/signin/。屏幕会提示您输入 Quay 凭据，如下截图所示：![图 8.2 - 红帽 Quay 登录页面](img/Figure_8.2.jpg)

图 8.2 - 红帽 Quay 登录页面

1.  在页面底部，单击**创建帐户**链接。屏幕会提示您使用一组对话框来创建一个新的 Quay 帐户，如下截图所示：![图 8.3 - 红帽 Quay 创建新帐户页面](img/Figure_8.3.jpg)

图 8.3 - 红帽 Quay**创建新帐户**页面

1.  输入您想要的凭据，然后选择**创建免费帐户**。

1.  您很快将收到一封电子邮件确认。单击确认电子邮件中的链接以验证您的帐户并继续使用新帐户的 Quay。

创建了新的 Quay 帐户后，您可以继续为 operator 图像创建新的图像存储库。

要创建新的图像存储库，请在 Quay 页面右上角选择**+**加号图标，然后选择**新存储库**，如下截图所示：

![图 8.4 - 选择“新存储库”以创建新的图像存储库](img/Figure_8.4.jpg)

图 8.4 - 选择“新存储库”以创建新的图像存储库

1.  然后，您将被带到**创建新存储库**页面，在那里您应该输入以下细节：

对于**存储库名称**，输入`guestbook-operator`。

选择**Public**单选按钮，表示对存储库的无身份验证访问。此更改将简化 Kubernetes 访问图像的方式。

其余选项可以保持默认值。完成后，**创建新存储库**页面应该会出现，如下截图所示：

![图 8.5 - Quay 中的“创建新存储库”页面](img/Figure_8.5.jpg)

图 8.5 - Quay 中的“创建新存储库”页面

1.  选择**创建公共存储库**按钮以创建 Quay 存储库。

现在已经创建了一个存储库来存储 Guestbook Operator 图像，让我们准备一个环境，其中包含构建 Helm operator 所需的工具。

### 准备本地开发环境

要创建 Helm operator，您至少需要以下 CLI 工具：

+   `operator-sdk`

+   `docker`，`podman`或`buildah`

`operator-sdk` CLI 是用于帮助开发 Kubernetes Operators 的工具包。它包含简化 operator 开发过程的内在逻辑。在幕后，`operator-sdk`需要一个容器管理工具，它可以用来构建 operator 图像。`operator-sdk` CLI 支持`docker`，`podman`和`buildah`作为底层容器管理工具。

要安装`operator-sdk` CLI，您可以从它们的[GitHub 存储库 https://github.com/operator-framework/](https://github.com/operator-framework/operator-sdk/releases)operator-sdk/releases 下载一个版本。但是，安装`docker`，`podman`或`buildah`的过程可能会因操作系统而异；更不用说，Windows 用户将无法原生地使用`operator-sdk`工具包。

幸运的是，Minikube 虚拟机（VM）可以作为开发人员的工作环境，因为它是一个 Linux VM，并且还包含 Docker CLI，适用于许多不同操作系统。在本节中，我们将在 Minikube VM 上安装`operator-sdk`，并将使用此环境来创建 operator。请注意，虽然提供的步骤旨在在 VM 中运行，但大多数步骤也适用于所有 Linux 和 Mac 机器。

按照以下步骤在 Minikube VM 上安装`operator-sdk`：

1.  通过运行`minikube ssh`命令来访问 VM，如下所示：

```
$ minikube ssh
```

1.  一旦进入 VM，您需要下载`operator-sdk` CLI。这可以通过使用`curl`命令来完成。请注意，写作时使用的`operator-sdk`版本是`0.15.2`版本。

要下载此版本的`operator-sdk` CLI，请运行以下命令：

```
$ cu**rl -o operator-sdk -L https://github.com/operator-framework/operator-sdk/releases/download/v0.15.2/operator-sdk-v0**.15.2-x86_64-linux-gnu 
```

1.  下载后，您需要更改`operator-sdk`二进制文件的权限为用户可执行。运行`chmod`命令进行此修改，如下所示：

```
$ chmod u+x operator-sdk
```

1.  接下来，将`operator-sdk`二进制文件移动到 VM 的`PATH`变量管理的位置，例如`/usr/bin`。因为此操作需要 root 权限，您需要使用`sudo`运行`mv`命令，如下所示：

```
$ sudo mv operator-sdk /usr/bin
```

1.  最后，通过运行`operator-sdk version`命令来验证您的`operator-sdk`安装，如下所示：

```
$ operator-sdk version
operator-sdk version: 'v0.15.2', commit: 'ffaf278993c8fcb00c6f527c9f20091eb8dd3352', go version: 'go1.13.3 linux/amd64'
```

如果此命令执行没有错误，那么您已成功安装了`operator-sdk` CLI。

1.  作为一个额外的步骤，您还应该在 Minikube VM 中克隆 Packt 存储库，因为我们将稍后利用`guestbook` Helm 图表来构建 Helm operator。在 VM 中运行以下命令来克隆存储库：](https://github.com/PacktPublishing/-Learn-Helm.git)

```
$ git clone https://github.com/PacktPub**lishing/-Learn-Helm.git Learn-Helm
```

现在您已经有了 Quay 镜像存储库和从 Minikube VM 创建的本地开发环境，让我们开始编写 Guestbook Operator。请注意，operator 代码的示例位于 Packt 存储库的 https://github.com/PacktPublishing/-Learn-Helm/tree/master/guestbook-operator 位置。

## 搭建 operator 文件结构

与 Helm 图表本身类似，由`operator-sdk` CLI 构建的 Helm Operators 具有必须遵守的特定文件结构。文件结构在下表中进行了解释：

![图 8.6 - 文件结构解释](img/Figure_8.6.jpg)

图 8.6 - 文件结构解释

使用`operator-sdk new`命令可以轻松创建操作员文件结构。在您的 Minikube VM 中，执行以下命令来创建 Guestbook Operator 的脚手架：

```
$ operator-sdk new guestbook-operator --type helm --kind Guestbook --helm-chart Learn-Helm/helm-charts/charts/guestbook
INFO[0000] Creating new Helm operator 'guestbook-operator'. 
INFO[0003] Created helm-charts/guestbook       
WARN[0003] Using default RBAC rules: failed to get Kubernetes config: could not locate a kubeconfig 
INFO[0003] Created build/Dockerfile                     
INFO[0003] Created watches.yaml                         
INFO[0003] Created deploy/service_account.yaml          
INFO[0003] Created deploy/role.yaml                     
INFO[0003] Created deploy/role_binding.yaml             
INFO[0003] Created deploy/operator.yaml                 
INFO[0003] Created deploy/crds/charts.helm.k8s.io_v1alpha1_guestbook_cr.yaml 
INFO[0003] Generated CustomResourceDefinition manifests. 
INFO[0003] Project creation complete.
```

`operator-sdk new`命令创建了一个名为`guestbook-operator`的本地目录，其中包含操作员内容。指定应使用`--type`标志创建 Helm 操作员，以及`Guestbook`作为 CR 的名称。

最后，`--helm-chart`标志指示`operator-sdk` CLI 将源 Guestbook 图表复制到操作员目录。

成功创建了 Guestbook 操作员的脚手架，让我们构建操作员并将其推送到您的 Quay 注册表。

## 构建操作员并将其推送到 Quay

`operator-sdk` CLI 提供了一个`operator-sdk build`命令，可以轻松构建操作员图像。此命令旨在针对操作员的顶级目录运行，并将通过引用位于操作员`build/`文件夹下的 Dockerfile 来构建图像。

在您的 Minikube VM 中，运行`operator-sdk build`命令，将您的 Quay 用户名替换为指定位置，如下所示：

```
$ cd guestbook-operator
$ operator-sdk build quay.io/$QUAY_USERNAME/guestbook-operator
```

如果构建成功，您将收到以下消息：

```
INFO[0092] Operator build complete.
```

由于 Minikube VM 安装了 Docker，`operator-sdk` CLI 在后台使用 Docker 构建图像。您可以运行`docker images`命令来验证图像是否已构建，如下所示：

```
$ docker images
```

操作员图像在本地构建后，必须将其推送到图像注册表，以便可以从 Kubernetes 中拉取。为了使用 Docker 将图像推送到注册表，您必须首先对目标注册表进行身份验证。使用`docker login`命令登录到 Quay，如下面的代码片段所示：

```
$ docker login quay.io --username $QUAY_USERNAME --password $QUAY_PASSWORD
```

登录到 Quay 后，使用`docker push`命令将操作员图像推送到 Quay 注册表，就像这样：

```
$ docker push quay.io/$QUAY_USERNAME/guestbook-operator
```

推送完成后，返回到您在*创建 Quay 存储库*部分创建的`guestbook-operator`存储库。您应该能够在**存储库标签**部分看到一个新的标签发布，如下面的屏幕截图所示：

![图 8.7 – 应将新标签推送到您的 Quay 注册表](img/Figure_8.7.jpg)

图 8.7 – 应将新标签推送到您的 Quay 注册表

现在您的操作员已经推送到容器注册表，让我们继续通过将操作员部署到您的 Kubernetes 环境。

## 部署 Guestbook 操作员

在搭建 Guestbook Operator 时，`operator-sdk` CLI 还创建了一个名为`deploy`的文件夹，并生成了部署操作员所需的文件。

以下是`deploy`文件夹中的内容所示的文件结构：

```
deploy/
  crds/
    charts.helm.k8s.io_guestbooks_crd.yaml
    charts.helm.k8s.io_v1alpha1_guestbook_cr.yaml
  operator.yaml
  role_binding.yaml
  role.yaml
  service_account.yaml
```

`crds/`文件夹包含创建 Guestbook CRD 所需的 YAML 资源（`charts.helm.k8s.io_guestbooks_crd.yaml`）。此文件用于在 Kubernetes 中注册新的 Guestbook API 端点。此外，`crds/`文件夹包含一个示例 Guestbook CR 应用程序（`charts.helm.k8s.io_v1alpha1_guestbook_cr.yaml`）。创建此文件将触发操作员安装 Guestbook Helm 图表。

请查看 CR 的内容，以熟悉所定义属性的类型，如下所示：

```
$ cat guestbook-operator/deploy/crds/charts.helm.k8s.io_v1alpha1_guestbook_cr.yaml
```

以下代码块中提供了输出的片段：

![图 8.8 - Guestbook CR 的片段](img/Figure_8.8.jpg)

图 8.8 - Guestbook CR 的片段

`spec`部分中的每个条目都指向 Guestbook 图表的`values.yaml`文件。`operator-sdk`工具自动使用此文件中包含的每个默认值创建了此示例 CR。在应用此 CR 之前，可以添加或修改其他条目，以覆盖 Guestbook 图表的其他值。这些值在运行时由操作员使用，以相应地部署 Guestbook 应用程序。

`deploy/operator.yaml`文件定义了实际的操作员本身，并包含一个简单的部署资源。我们将很快返回到这个文件的内容。

`role_binding.yaml`、`role.yaml`和`service_account.yaml`文件是为了为操作员提供必要的权限，以便监视 Guestbook CR 并将 Guestbook Helm 图表安装到 Kubernetes 中。它通过在`service_account.yaml`文件中定义的服务帐户进行身份验证，然后执行这些操作。一旦经过身份验证，操作员将根据`role.yaml`和`role_binding.yaml`资源获得授权。`role.yaml`文件列出了描述操作员被允许执行的确切资源和操作的精细权限。`role_binding.yaml`文件将角色绑定到操作员的服务帐户。

了解操作员`deploy/`文件夹下创建的每个资源后，请按照以下步骤部署您的 Guestbook 操作员：

1.  不幸的是，Minikube VM 不包含`Kubectl`，所以如果您仍然通过命令行连接到 VM，您必须首先退出到您的本地系统，通过运行以下命令：

```
$ exit
```

1.  早些时候使用`operator-sdk`创建的资源也位于 Packt 存储库的`guestbook-operator/`文件夹下。如果您之前没有克隆过这个存储库，请使用以下命令现在克隆它：

```
$ git clone https://github.com/PacktPublishing/-Learn-Helm.git Learn-Helm
```

作为一个快速的旁注，需要注意的是，Packt 存储库中唯一修改自 Minikube VM 中创建的资源的资源是`role.yaml`文件。`operator-sdk` CLI 基于包含在 guestbook Helm 图表中的模板文件生成了一个简单的`role.yaml`文件。但是，如果您能回忆起来，guestbook 图表包含了一些资源，只有在条件值基础上才会包含这些资源。这些资源是`Job`和`PersistentVolumeClaim`挂钩资源，只有在启用持久存储时才会包含。其中一个示例显示在`PersistentVolumeClaim`模板中，如下面的代码片段所示：

```
{{- if .Values.redis.master.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
```

`operator-sdk` CLI 没有自动为`Jobs`和`PersistentVolumeClaims`创建**基于角色的访问控制**（**RBAC**）规则，因为它不知道是否应该包含此模板。

因此，作者已将这些规则添加到位于 https://github.com/PacktPublishing/-Learn-Helm/blob/master/guestbook-operator/deploy/role.yaml#L81-L104 的`role.yaml`文件中。

1.  Guestbook 操作员将依赖于一个新的 API 端点。通过在`guestbook-operator/deploy/crds`文件夹下应用 CRD 来创建此端点，如下所示：

```
$ kubectl apply -f guestbook-operator/deploy/crds/charts.helm.k8s.io_guestbooks_crd.yaml
```

我们将在稍后使用该文件夹下的第二个文件（CR）来部署 Guestbook 应用程序。

1.  接下来，您需要修改`guestbook-operator/deploy/operator.yaml`文件，以指定您之前构建的操作员图像。您会注意到在这个文件中有以下代码行：

```
# Replace this with the built image name
image: REPLACE_IMAGE
```

将`REPLACE_IMAGE`文本替换为您的操作员图像的位置。此值应类似于`quay.io/$QUAY_USERNAME/guestbook-operator`。

1.  一旦您应用了 CRD 并更新了您的`operator.yaml`文件，您可以通过运行以下命令来继续应用`guestbook-operator/deploy/`文件夹中的每个资源：

```
$ kubectl apply -f guestbook-operator/deploy -n chapter8
```

1.  通过对`chapter8`命名空间中的 Pods 运行观察，等待操作员报告`1/1`就绪状态，就像这样：

```
$ kubectl get pods -n chapter8 -w
```

现在 Guestbook operator 已部署，让我们使用它来安装 Guestbook Helm chart。

## 部署 Guestbook 应用程序

当使用 Helm 作为独立的 CLI 工具时，您可以通过运行`helm install`命令来安装 Helm chart。使用 Helm operator，您可以通过创建 CR 来安装 Helm chart。通过创建位于`guestbook-operator/deploy/crds/`文件夹下的提供的 CR 来安装 Guestbook Helm chart，如下面的代码片段所示：

```
$ kubectl apply -f guestbook-operator/deploy/crds/charts.helm.k8s.io_v1alpha1_guestbook_cr.yaml -n chapter8
```

对`chapter8`命名空间中的 Pod 运行另一个`watch`命令，如下面的代码片段所示，您应该能够看到 Guestbook 和 Redis Pods 因 Helm chart 安装而启动：

```
$ kubectl get pods -n chapter8 -w
```

以下代码块描述了每个 Pod 处于`READY`状态：

```
NAME                                  READY   STATUS    RESTARTS
example-guestbook-65bc5fdc55-jvkdz    1/1     Running   0
guestbook-operator-6fddc8d7cb-94mzp   1/1     Running   0
redis-master-0                        1/1     Running   0
redis-slave-0                         1/1     Running   0
redis-slave-1                         1/1     Running   0
```

当您创建 Guestbook CR 时，操作员会执行`helm install`命令来安装 Guestbook chart。您可以通过运行`helm list`来确认已创建的发布，就像这样：

```
$ helm list -n chapter8
NAME             	NAMESPACE	REVISION	UPDATED       
example-guestbook	chapter8 	1       	2020-02-24
```

通过修改`example-guestbook` CR 来执行发布的升级。修改您的`guestbook-operator/deploy/crds/charts.helm.k8s.io_v1alpha1_guestbook_cr.yaml`文件，将副本数从`1`更改为`2`，就像这样：

```
replicaCount: 2
```

在更新了`replicaCount`值之后应用更改，如下所示：

```
$ kubectl apply -f guestbook-operator/deploy/crds/charts.helm.k8s.io_v1alpha1_guestbook_cr.yaml -n chapter8
```

修改 Guestbook CR 将触发针对`example-guestbook`发布的`helm upgrade`命令。正如您可能还记得*第五章*中所述，*构建您的第一个 Helm Chart*，Guestbook Helm chart 的升级钩子将启动对 Redis 数据库的备份。如果您在修改 CR 后对`chapter8`命名空间中的 Pod 运行`watch`，您将注意到一个备份`Job`开始，并且一旦备份完成，您将看到两个 Guestbook Pods 中的一个终止。您还将从以下代码片段中的`helm list`命令中注意到`example-guestbook`发布的修订号已增加到`2`：

```
$ helm list -n chapter8
NAME             	NAMESPACE	REVISION	UPDATED       
example-guestbook	chapter8 	2       	2020-02-24
```

尽管修订号已增加到`2`，但截至撰写本文时，基于 Helm 的 Operators 的一个限制是您无法像使用 CLI 那样发起回滚到先前的修订。如果您尝试对`example-guestbook`发布运行`helm history`，您还将注意到只有第二个修订在发布历史中，如下面的代码片段所示：

```
$ helm history example-guestbook -n chapter8
REVISION	UPDATED                 	STATUS        
2       	Tue Feb 25 04:36:10 2020	deployed
```

这是使用 Helm CLI 和使用基于 Helm 的 operator 之间的重要区别。由于不保留发布历史记录，基于 Helm 的 operator 不允许执行显式回滚。但是，如果升级失败，将运行`helm rollback`命令。在这种情况下，将执行回滚钩子，试图回滚到尝试的升级。

尽管基于 Helm 的 operator 不保留发布历史记录，但它在同步应用程序的期望状态和实际状态方面表现出色。这是因为 operator 不断监视 Kubernetes 环境的状态，并确保应用程序始终配置为与 CR 上指定的配置匹配。换句话说，如果修改了 Guestbook 应用程序的资源之一，operator 将立即恢复更改，使其与 CR 上定义的规范匹配。您可以通过修改 Guestbook 资源之一上的字段来看到这一点。

例如，我们将直接将 Guestbook 部署的副本计数从`2`更改为`3`，并观察 operator 自动将其恢复为`2`个副本，以重新同步 CR 中定义的期望状态。

执行以下`kubectl patch`命令，将 Guestbook 部署的副本计数从`2`更改为`3`：

```
$ kubectl patch deployment example-guestbook -p '{'spec':{'replicas':3}}' -n chapter8
```

通常，这只会添加一个额外的 Guestbook 应用程序副本。但是，因为 Guestbook CR 当前仅定义了`2`个副本，所以 operator 会快速将副本计数更改回`2`，并终止创建的额外 Pod。如果您实际上想将副本计数增加到`3`，则必须更新 Guestbook CR 上的`replicaCount`值。该过程的优势在于确保期望状态与集群的实际状态匹配。

使用基于 Helm 的 operator 卸载 Guestbook 应用程序就像删除 CR 一样简单。删除`example-guestbook` CR 以卸载发布，就像这样：

```
$ kubectl delete -f guestbook-operator/deploy/crds/charts.helm.k8s.io_v1alpha1_guestbook_cr.yaml -n chapter8
```

这将删除`example-guestbook`发布以及所有相关资源。

您还可以删除 Guestbook Operator 及其资源，因为我们在下一节中将不再需要它们。您可以通过运行以下命令来执行此操作：

```
$ kubectl delete -f guestbook-operator/deploy/ -n chapter8
```

一般来说，您应该始终确保在删除运算符之前先删除 CR。当您删除 CR 时，运算符会执行`helm uninstall`命令来删除您的发布。如果您意外地先删除了运算符，您将不得不在命令行上手动运行`helm uninstall`。

在本节中，您创建了一个 Helm 运算符，并学习了如何使用基于运算符的方法部署应用程序。在下一节中，我们将继续讨论运算符，探讨如何使用 Helm 来管理它们。

# 使用 Helm 管理运算符和 CRs

在前一节中，您首先通过创建位于`guestbook-operator/deploy/crds/`文件夹下的 CRD 来安装了 Guestbook 运算符。接下来，您创建了位于`guestbook-operator/deploy/`文件夹下的运算符资源。最后，您创建了 CR 来部署 Guestbook 应用程序。这些任务都是使用 Kubectl CLI 执行的，但也可以使用 Helm 图表来提供更灵活和可重复的解决方案来安装和管理运算符。

Helm 允许您在 Helm 图表中提供一个名为`crds/`的特殊目录，用于在安装图表时创建 CRDs。Helm 会在`templates/`文件夹下定义的任何其他资源之前创建 CRDs，使得安装依赖于 CRDs 存在的应用程序（如运算符）更加简单。

以下文件结构描述了一个 Helm 图表，可用于安装 Guestbook 运算符：

```
guestbook-operator/
  Chart.yaml
  crds/
    charts.helm.k8s.io_guestbooks_crd.yaml
  templates/
    operator.yaml
    role_binding.yaml
    role.yaml
    Service_account.yaml
  values.yaml
```

安装此 Helm 图表时，首先会安装 Guestbook CRD。如果 CRD 已经存在于集群中，它将跳过 CRD 的创建，而只会创建模板资源。请注意，虽然 CRDs 可以方便地包含在 Helm 图表中，但存在一些限制。首先，Helm 图表中的 CRDs 不能包含任何 Go 模板，因此 CRDs 无法像典型资源那样受益于参数化。CRDs 也永远无法升级、回滚或删除。因此，如果需要执行这些操作，用户必须小心地手动修改或删除 CRDs。最后，如前所述安装此类图表将需要集群管理员权限，这是 Kubernetes 中允许的最高权限，因为图表至少包含一个 CRD 资源。

前面描述的 Helm chart 可以被集群管理员使用，以便轻松安装 Guestbook operator。然而，这只是方程的一半，因为最终用户仍然必须创建 CRs 来部署 Guestbook 应用程序。幸运的是，operator 的最终用户也可以利用 Helm，创建一个包装 Guestbook CR 的 Helm chart。

这样的 Helm chart 的示例布局显示在以下文件结构中：

```
guestbook-cr
  Chart.yaml
  templates/
    guestbook.yaml
  values.yaml
```

前面的示例包括一个名为`guestbook.yaml`的模板。这个模板可以包含最初由`operator-sdk` CLI 生成的 Guestbook CR，名称为`charts.helm.k8s.io_v1alpha1_guestbook_cr.yaml`。与 CRDs 不同，`templates/`文件夹下的 CRs 受益于 Go 模板和生命周期管理，就像所有其他资源一样。当 CR 包含基于用户提供的值有条件地包含的复杂字段，或者当同一个发布中必须包含多个不同的 CRs 时，这种方法提供了最大的价值。通过这种方法，您还可以管理 CRs 的生命周期并保持修订历史。

现在您已经了解了如何创建 Helm operator 以及如何使用 Helm 来帮助管理 Operators，可以在下一节中自由地清理您的 Kubernetes 环境。

# 清理您的 Kubernetes 环境

首先，运行以下命令来删除您的 Guestbook CRD：

```
$ kubectl delete crd guestbooks.charts.helm.k8s.io
```

在继续下一个清理步骤之前，请注意，在*问题*部分后面提出的一个问题将挑战您编写自己的 Helm charts 来实现*使用 Helm 管理 Operators 和 CRs*部分讨论的图表设计。您可能希望推迟这些步骤来测试您的实现。

要继续清理工作，请运行以下命令来删除您的`chapter8`命名空间：

```
$ kubectl delete ns chapter8
```

最后，运行`minikube stop`命令来停止您的 Minikube 虚拟机。

# 摘要

operator 对于确保期望状态始终与实际状态匹配非常重要。这样的功能允许用户更轻松地维护资源配置的真实来源。用户可以利用基于 Helm 的 operator 来提供这种类型的资源协调，并且很容易上手，因为它使用 Helm 图表作为部署机制。当创建 CR 时，Helm operator 将安装相关的 Helm 图表以创建新的发布。当修改 CR 时，将执行后续升级，并且在删除 CR 时将卸载发布。

为了管理 operator，集群管理员可以创建一个单独的 Helm 图表，用于创建 operator 的资源和 CRDs。最终用户也可以创建一个单独的 Helm 图表，用于创建 operator 的 CRs，以及其他可能相关的任何资源。

在下一章中，我们将讨论 Helm 生态系统中安全性的最佳实践和主题。

# 进一步阅读

有关 Kubernetes 资源的更多信息，您可以查看以下链接：

+   要发现更多由社区开发的 Operators，请查阅此存储库：[`github.com/operator-framework/awesome-operators`](https://github.com/operator-framework/awesome-operators)。

+   您可以从 Kubernetes 文档中了解有关 Operators 及其起源的更多信息：[`kubernetes.io/docs/concepts/extend-kubernetes/operator/.`](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/. )

# 问题

1.  Kubernetes operator 是如何工作的？

1.  使用 Helm CLI 和使用基于 Helm 的 operator 之间有什么区别？

1.  假设你被要求将现有的 Helm 图表创建为 Helm operator。你会采取哪些步骤来完成这个任务？

1.  在 Helm operator 中，安装、升级、回滚和卸载的生命周期钩子函数是如何工作的？

1.  在 Helm 图表中，`crds/`文件夹的目的是什么？

1.  在“使用 Helm 管理 Operators 和 CRs”部分中，我们介绍了两种不同的 Helm 图表，可以用来帮助管理 Operators 和 CRs。使用该部分提供的图表布局来实现 Helm 图表。这些图表应该用于安装 Guestbook operator 和安装 Guestbook CR。有关创建 Helm 图表的帮助，请参考“第五章”，*构建您的第一个 Helm 图表*。

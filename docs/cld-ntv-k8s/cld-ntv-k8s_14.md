# 第十一章：Kubernetes 上的模板代码生成和 CI/CD

本章讨论了一些更容易的方法，用于模板化和配置具有许多资源的大型 Kubernetes 部署。它还详细介绍了在 Kubernetes 上实施**持续集成**/**持续部署**（**CI**/**CD**）的多种方法，以及与每种可能方法相关的利弊。具体来说，我们谈论了集群内 CI/CD，其中一些或所有的 CI/CD 步骤在我们的 Kubernetes 集群中执行，以及集群外 CI/CD，其中所有步骤都在我们的集群之外进行。

本章的案例研究将包括从头开始创建 Helm 图表，以及对 Helm 图表的每个部分及其工作原理的解释。

首先，我们将介绍 Kubernetes 资源模板生成的概况，以及为什么应该使用模板生成工具。然后，我们将首先使用 AWS CodeBuild，然后使用 FluxCD 来实施 CI/CD 到 Kubernetes。

在本章中，我们将涵盖以下主题：

+   了解在 Kubernetes 上进行模板代码生成的选项

+   使用 Helm 和 Kustomize 在 Kubernetes 上实施模板

+   了解 Kubernetes 上的 CI/CD 范式-集群内和集群外

+   在 Kubernetes 上实施集群内和集群外的 CI/CD

# 技术要求

为了运行本章中详细介绍的命令，您需要一台支持`kubectl`命令行工具的计算机，以及一个可用的 Kubernetes 集群。请参考*第一章*，*与 Kubernetes 通信*，了解快速启动和运行 Kubernetes 的几种方法，以及如何安装 kubectl 工具的说明。此外，您还需要一台支持 Helm CLI 工具的机器，通常具有与 kubectl 相同的先决条件-有关详细信息，请查看 Helm 文档[`helm.sh/docs/intro/install/`](https://helm.sh/docs/intro/install/)。

本章中使用的代码可以在书籍的 GitHub 存储库中找到

[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter11`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter11)。

# 了解在 Kubernetes 上进行模板代码生成的选项

正如在*第一章*中讨论的那样，*与 Kubernetes 通信*，Kubernetes 最大的优势之一是其 API 可以通过声明性资源文件进行通信。这使我们能够运行诸如`kubectl apply`之类的命令，并确保控制平面确保集群中运行的任何资源与我们的 YAML 或 JSON 文件匹配。

然而，这种能力引入了一些难以控制的因素。由于我们希望将所有工作负载声明在配置文件中，任何大型或复杂的应用程序，特别是如果它们包含许多微服务，可能会导致大量的配置文件编写和维护。

这个问题在多个环境下会更加复杂。假设我们需要开发、暂存、UAT 和生产环境，这将需要每个 Kubernetes 资源四个单独的 YAML 文件，假设我们想要保持每个文件一个资源的清晰度。

解决这些问题的一种方法是使用支持变量的模板系统，允许单个模板文件适用于多个应用程序或多个环境，通过注入不同的变量集。

有几种受社区支持的流行开源选项可用于此目的。在本书中，我们将重点关注其中两种最受欢迎的选项。

+   Helm

+   Kustomize

有许多其他选项可供选择，包括 Kapitan、Ksonnet、Jsonnet 等，但本书不在讨论范围之内。让我们先来回顾一下 Helm，它在很多方面都是最受欢迎的模板工具。

## Helm

Helm 实际上扮演了模板/代码生成工具和 CI/CD 工具的双重角色。它允许您创建基于 YAML 的模板，可以使用变量进行填充，从而实现跨应用程序和环境的代码和模板重用。它还配备了一个 Helm CLI 工具，可以根据模板本身来推出应用程序的更改。

因此，你可能会在 Kubernetes 生态系统中到处看到 Helm 作为安装工具或应用程序的默认方式。在本章中，我们将使用 Helm 来完成它的两个目的。

现在，让我们转向 Kustomize，它与 Helm 有很大不同。

## Kustomize

与 Helm 不同，Kustomize 得到了 Kubernetes 项目的官方支持，并且支持直接集成到`kubectl`中。与 Helm 不同，Kustomize 使用原始的 YAML 而不是变量，并建议使用*fork and patch*工作流，在这个工作流中，YAML 的部分根据所选择的补丁被替换为新的 YAML。

既然我们对这些工具的区别有了基本的了解，我们可以在实践中使用它们。

# 使用 Helm 和 Kustomize 在 Kubernetes 上实现模板

既然我们知道了我们的选择，我们可以用一个示例应用程序来实现它们中的每一个。这将使我们能够了解每个工具处理变量和模板化过程的具体细节。让我们从 Helm 开始。

## 使用 Helm 与 Kubernetes

如前所述，Helm 是一个开源项目，它使得在 Kubernetes 上模板化和部署应用程序变得容易。在本书的目的上，我们将专注于最新版本（写作时），即 Helm V3。之前的版本 Helm V2 有更多的移动部分，包括一个称为*Tiller*的控制器，它会在集群上运行。Helm V3 被简化了，只包含 Helm CLI 工具。然而，它在集群上使用自定义资源定义来跟踪发布，我们很快就会看到。

让我们从安装 Helm 开始。

### 安装 Helm

如果你想使用特定版本的 Helm，你可以按照[`helm.sh/docs/intro/install/`](https://helm.sh/docs/intro/install/)中的特定版本文档来安装它。对于我们的用例，我们将简单地使用`get helm`脚本，它将安装最新版本。

您可以按照以下步骤获取并运行脚本：

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

现在，我们应该能够运行`helm`命令了。默认情况下，Helm 将自动使用您现有的`kubeconfig`集群和上下文，因此为了在 Helm 中切换集群，您只需要使用`kubectl`来更改您的`kubeconfig`文件，就像您通常做的那样。

要使用 Helm 安装应用程序，请运行`helm install`命令。但是 Helm 是如何决定安装什么和如何安装的呢？我们需要讨论 Helm 图表、Helm 仓库和 Helm 发布的概念。

### Helm 图表、仓库和发布

Helm 提供了一种使用变量在 Kubernetes 上模板化和部署应用程序的方法。为了做到这一点，我们通过一组模板来指定工作负载，这被称为*Helm 图表*。

Helm 图表由一个或多个模板、一些图表元数据和一个`values`文件组成，该文件用最终值填充模板变量。在实践中，您将为每个环境（或应用程序，如果您正在为多个应用程序重用模板）拥有一个`values`文件，该文件将使用新配置填充共享模板。然后，模板和值的组合将用于在集群中安装或部署应用程序。

那么，您可以将 Helm 图表存储在哪里？您可以像对待任何其他 Kubernetes YAML 一样将它们放在 Git 存储库中（这对大多数用例都适用），但 Helm 还支持存储库的概念。Helm 存储库由 URL 表示，可以包含多个 Helm 图表。例如，Helm 在[`hub.helm.sh/charts`](https://hub.helm.sh/charts)上有自己的官方存储库。同样，每个 Helm 图表由一个包含元数据文件的文件夹、一个`Chart.yaml`文件、一个或多个模板文件以及一个可选的 values 文件组成。

为了安装具有本地 values 文件的本地 Helm 图表，您可以为每个传递路径到`helm install`，如以下命令所示：

```
helm install -f values.yaml /path/to/chart/root
```

然而，对于常用的安装图表，您也可以直接从图表存储库安装图表，并且您还可以选择将自定义存储库添加到本地 Helm 中，以便能够轻松地从非官方来源安装图表。

例如，为了通过官方 Helm 图表安装 Drupal，您可以运行以下命令：

```
helm install -f values.yaml stable/drupal
```

此代码从官方 Helm 图表存储库安装图表。要使用自定义存储库，您只需要首先将其添加到 Helm 中。例如，要安装托管在`jetstack` Helm 存储库上的`cert-manager`，我们可以执行以下操作：

```
helm repo add jetstack https://charts.jetstack.io
helm install certmanager --namespace cert-manager jetstack/cert-manager
```

此代码将`jetstack` Helm 存储库添加到本地 Helm CLI 工具中，然后通过其中托管的图表安装`cert-manager`。我们还将发布命名为`cert-manager`。Helm 发布是 Helm V3 中使用 Kubernetes secrets 实现的概念。当我们在 Helm 中创建一个发布时，它将作为同一命名空间中的一个 secret 存储。

为了说明这一点，我们可以使用前面的`install`命令创建一个 Helm 发布。现在让我们来做吧：

```
helm install certmanager --namespace cert-manager jetstack/cert-manager
```

该命令应该产生以下输出，具体内容可能会有所不同，取决于当前的 Cert Manager 版本。为了便于阅读，我们将输出分为两个部分。

首先，命令的输出给出了 Helm 发布的状态：

```
NAME: certmanager
LAST DEPLOYED: Sun May 23 19:07:04 2020
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

正如您所看到的，此部分包含部署的时间戳、命名空间信息、修订版本和状态。接下来，我们将看到输出的注释部分：

```
NOTES:
cert-manager has been deployed successfully!
In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).
More information on the different types of issuers and how to configure them
can be found in our documentation:
https://cert-manager.io/docs/configuration/
For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:
https://cert-manager.io/docs/usage/ingress/
```

正如您所看到的，我们的 Helm `install`命令已经成功，这也给了我们一些来自`cert-manager`的信息，告诉我们如何使用它。这个输出在安装 Helm 软件包时可能会很有帮助，因为它们有时包括先前片段中的文档。现在，为了查看我们的 Kubernetes 中的发布对象是什么样子，我们可以运行以下命令：

```
Kubectl get secret -n cert-manager
```

这将产生以下输出：

![图 11.1 – 来自 kubectl 的 Secrets 列表输出](img/B14790_11_001.jpg)

图 11.1 – 来自 kubectl 的 Secrets 列表输出

正如您所看到的，其中一个密钥的类型为`helm.sh/release.v1`。这是 Helm 用来跟踪 Cert Manager 发布的密钥。

最后，要在 Helm CLI 中查看发布列表，我们可以运行以下命令：

```
helm ls -A
```

此命令将列出所有命名空间中的 Helm 发布（就像`kubectl get pods -A`会列出所有命名空间中的 pod 一样）。输出将如下所示：

![图 11.2 – Helm 发布列表输出](img/B14790_11_002.jpg)

图 11.2 – Helm 发布列表输出

现在，Helm 有更多的组件，包括`升级`、`回滚`等，我们将在下一节中进行审查。为了展示 Helm 的功能，我们将从头开始创建和安装一个图表。

### 创建 Helm 图表

因此，我们希望为我们的应用程序创建一个 Helm 图表。让我们开始吧。我们的目标是轻松地将一个简单的 Node.js 应用程序部署到多个环境中。为此，我们将创建一个包含应用程序组件的图表，然后将其与三个单独的值文件（`dev`、`staging`和`production`）结合起来，以便将我们的应用程序部署到三个环境中。

让我们从 Helm 图表的文件夹结构开始。正如我们之前提到的，Helm 图表由模板、元数据文件和可选值组成。我们将在实际安装图表时注入这些值，但我们可以将我们的文件夹结构设计成这样：

```
Chart.yaml
charts/
templates/
dev-values.yaml
staging-values.yaml
production-values.yaml
```

我们还没有提到的一件事是，您实际上可以在现有图表中拥有一个 Helm 图表的文件夹！这些子图表可以将复杂的应用程序分解为组件，使其易于管理。对于本书的目的，我们将不使用子图表，但是如果您的应用程序变得过于复杂或模块化，这是一个有价值的功能。

此外，您可以看到我们为每个环境都有一个不同的环境文件，在安装命令期间我们将使用它们。

那么，`Chart.yaml`文件是什么样子的呢？该文件将包含有关图表的一些基本元数据，并且通常看起来至少是这样的：

```
apiVersion: v2
name: mynodeapp
version: 1.0.0
```

`Chart.yaml`文件支持许多可选字段，您可以在[`helm.sh/docs/topics/charts/`](https://helm.sh/docs/topics/charts/)中查看，但是对于本教程的目的，我们将保持简单。强制字段是`apiVersion`，`name`和`version`。

在我们的`Chart.yaml`文件中，`apiVersion`对应于图表对应的 Helm 版本。有点令人困惑的是，当前版本的 Helm，Helm V3，使用`apiVersion` `v2`，而包括 Helm V2 在内的旧版本的 Helm 也使用`apiVersion` `v2`。

接下来，`name`字段对应于我们图表的名称。这相当容易理解，尽管请记住，我们有能力为图表的特定版本命名 - 这对于多个环境非常方便。

最后，我们有`version`字段，它对应于图表的版本。该字段支持**SemVer**（语义化版本）。

那么，我们的模板实际上是什么样子的呢？Helm 图表在底层使用 Go 模板库（有关更多信息，请参见[`golang.org/pkg/text/template/`](https://golang.org/pkg/text/template/)），并支持各种强大的操作、辅助函数等等。现在，我们将保持极其简单，以便让您了解基础知识。有关 Helm 图表创建的全面讨论可能需要一本专门的书！

首先，我们可以使用 Helm CLI 命令自动生成我们的`Chart`文件夹，其中包括所有先前的文件和文件夹，减去为您生成的子图和值文件。让我们试试吧 - 首先使用以下命令创建一个新的 Helm 图表：

```
helm create myfakenodeapp
```

这个命令将在名为`myfakenodeapp`的文件夹中创建一个自动生成的图表。让我们使用以下命令检查我们`templates`文件夹的内容：

```
Ls myfakenodeapp/templates
```

这个命令将产生以下输出：

```
helpers.tpl
deployment.yaml
NOTES.txt
service.yaml
```

这个自动生成的图表可以作为起点帮助很多，但是对于本教程的目的，我们将从头开始制作这些。

创建一个名为`mynodeapp`的新文件夹，并将我们之前向您展示的`Chart.yaml`文件放入其中。然后，在里面创建一个名为`templates`的文件夹。

要记住的一件事是：一个 Kubernetes 资源 YAML 本身就是一个有效的 Helm 模板。在模板中使用任何变量并不是必需的。你可以只编写普通的 YAML，Helm 安装仍然可以工作。

为了展示这一点，让我们从我们的模板文件夹中添加一个单个模板文件开始。将其命名为`deployment.yaml`，并包含以下非变量 YAML：

deployment.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-myapp
  labels:
    app: frontend-myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend-myapp
  template:
    metadata:
      labels:
        app: frontend-myapp
    spec:
      containers:
      - name: frontend-myapp
        image: myrepo/myapp:1.0.0
        ports:
        - containerPort: 80
```

正如你所看到的，这个 YAML 只是一个普通的 Kubernetes 资源 YAML。我们在我们的模板中没有使用任何变量。

现在，我们有足够的内容来实际安装我们的图表。让我们接下来做这件事。

### 安装和卸载 Helm 图表

要使用 Helm V3 安装图表，你需要从图表的`root`目录运行`helm install`命令：

```
helm install myapp .
```

这个安装命令创建了一个名为`frontend-app`的 Helm 发布，并安装了我们的图表。现在，我们的图表只包括一个具有两个 pod 的单个部署，我们应该能够通过以下命令在我们的集群中看到它正在运行：

```
kubectl get deployment
```

这应该会产生以下输出：

```
NAMESPACE  NAME            READY   UP-TO-DATE   AVAILABLE   AGE
default    frontend-myapp  2/2     2            2           2m
```

从输出中可以看出，我们的 Helm `install`命令已经成功在 Kubernetes 中创建了一个部署对象。

卸载我们的图表同样简单。我们可以通过运行以下命令来安装通过我们的图表安装的所有 Kubernetes 资源：

```
helm uninstall myapp
```

这个`uninstall`命令（在 Helm V2 中是`delete`）只需要我们 Helm 发布的名称。

到目前为止，我们还没有使用 Helm 的真正功能 - 我们一直把它当作`kubectl`的替代品，没有添加任何功能。让我们通过在我们的图表中实现一些变量来改变这一点。

### 使用模板变量

向我们的 Helm 图表模板添加变量就像使用双括号 - `{{ }}` - 语法一样简单。我们在双括号中放入的内容将直接从我们在安装图表时使用的值中取出，使用点符号表示法。

让我们看一个快速的例子。到目前为止，我们的应用名称（和容器镜像名称/版本）都是硬编码到我们的 YAML 文件中的。如果我们想要使用我们的 Helm 图表部署不同的应用程序或不同的应用程序版本，这将极大地限制我们。

为了解决这个问题，我们将在我们的图表中添加模板变量。看一下这个结果模板：

Templated-deployment.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-{{ .Release.Name }}
  labels:
    app: frontend-{{ .Release.Name }}
    chartVersion: {{ .Chart.version }}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend-{{ .Release.Name }}
  template:
    metadata:
      labels:
        app: frontend-{{ .Release.Name }}
    spec:
      containers:
      - name: frontend-{{ .Release.Name }}
        image: myrepo/{{ .Values.image.name }}
:{{ .Values.image.tag }}
        ports:
        - containerPort: 80
```

让我们浏览一下这个 YAML 文件，审查一下我们的变量。在这个文件中，我们使用了几种不同类型的变量，但它们都使用相同的点符号表示法。

Helm 实际上支持几种不同的顶级对象。这些是您可以在模板中引用的主要对象：

+   `.Chart`：用于引用`Chart.yaml`文件中的元数据值

+   `.Values`：用于引用在安装时从`values`文件传递到图表中的值

+   `.Template`：用于引用当前模板文件的一些信息

+   `.Release`：用于引用有关 Helm 发布的信息

+   `.Files`：用于引用图表中不是 YAML 模板的文件（例如`config`文件）

+   `.Capabilities`：用于引用目标 Kubernetes 集群的信息（换句话说，版本）

在我们的 YAML 文件中，我们正在使用其中的几个。首先，我们在几个地方引用我们发布的`name`（包含在`.Release`对象中）。接下来，我们正在利用`Chart`对象将元数据注入`chartVersion`键中。最后，我们使用`Values`对象引用容器镜像的`name`和`tag`。

现在，我们缺少的最后一件事是我们将通过`values.yaml`注入的实际值，或者通过 CLI 命令。其他所有内容将使用`Chart.yaml`创建，或者我们将通过`helm`命令本身在运行时注入的值。

考虑到这一点，让我们从我们的模板创建我们的值文件，我们将在其中传递我们的图像`name`和`tag`。因此，让我们以正确的格式包含它们：

```
image:
  name: myapp
  tag: 2.0.1
```

现在我们可以通过我们的 Helm 图表安装我们的应用程序！使用以下命令：

```
helm install myrelease -f values.yaml .
```

正如您所看到的，我们正在使用`-f`键传递我们的值（您也可以使用`--values`）。此命令将安装我们应用程序的发布。

一旦我们有了一个发布，我们就可以使用 Helm CLI 升级到新版本或回滚到旧版本-我们将在下一节中介绍这一点。

### 升级和回滚

现在我们有了一个活动的 Helm 发布，我们可以升级它。让我们对我们的`values.yaml`进行一些小改动：

```
image:
  name: myapp
  tag: 2.0.2
```

要使这成为我们发布的新版本，我们还需要更改我们的图表 YAML：

```
apiVersion: v2
name: mynodeapp
version: 1.0.1
```

现在，我们可以使用以下命令升级我们的发布：

```
helm upgrade myrelease -f values.yaml .
```

如果出于任何原因，我们想回滚到早期版本，我们可以使用以下命令：

```
helm rollback myrelease 1.0.0
```

正如您所看到的，Helm 允许无缝地对应用程序进行模板化、发布、升级和回滚。正如我们之前提到的，Kustomize 达到了许多相同的点，但它的方式大不相同-让我们看看。 

## 使用 Kustomize 与 Kubernetes

虽然 Helm 图表可能会变得非常复杂，但 Kustomize 使用 YAML 而不使用任何变量，而是使用基于补丁和覆盖的方法将不同的配置应用于一组基本的 Kubernetes 资源。

使用 Kustomize 非常简单，正如我们在本章前面提到的，不需要先决条件 CLI 工具。一切都可以通过使用`kubectl apply -k /path/kustomize.yaml`命令来完成，而无需安装任何新内容。但是，我们还将演示使用 Kustomize CLI 工具的流程。

重要说明

要安装 Kustomize CLI 工具，您可以在[`kubernetes-sigs.github.io/kustomize/installation`](https://kubernetes-sigs.github.io/kustomize/installation)上查看安装说明。

目前，安装使用以下命令：

```
curl -s "https://raw.githubusercontent.com/\
kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
```

现在我们已经安装了 Kustomize，让我们将 Kustomize 应用于我们现有的用例。我们将从我们的普通 Kubernetes YAML 开始（在我们开始添加 Helm 变量之前）：

plain-deployment.yaml：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-myapp
  labels:
    app: frontend-myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend-myapp
  template:
    metadata:
      labels:
        app: frontend-myapp
    spec:
      containers:
      - name: frontend-myapp
        image: myrepo/myapp:1.0.0
        ports:
        - containerPort: 80
```

创建了初始的`deployment.yaml`后，我们现在可以创建一个 Kustomization 文件，我们称之为`kustomize.yaml`。

当我们稍后使用`-k`参数调用`kubectl`命令时，`kubectl`将查找此`kustomize` YAML 文件，并使用它来确定要应用到传递给`kubectl`命令的所有其他 YAML 文件的补丁。

Kustomize 让我们可以修补单个值或设置自动设置的常见值。一般来说，Kustomize 会创建新行，或者如果 YAML 中的键已经存在，则更新旧行。有三种方法可以应用这些更改：

+   在 Kustomization 文件中直接指定更改。

+   使用`PatchStrategicMerge`策略和`patch.yaml`文件以及 Kustomization 文件。

+   使用`JSONPatch`策略和`patch.yaml`文件以及 Kustomization 文件。

让我们从专门用于修补 YAML 的 Kustomization 文件开始。

### 直接在 Kustomization 文件中指定更改

如果我们想在 Kustomization 文件中直接指定更改，我们可以这样做，但我们的选择有些有限。我们可以在 Kustomization 文件中使用的键的类型如下：

+   `resources`-指定应在应用补丁时自定义的文件

+   `transformers`-直接从 Kustomization 文件中应用补丁的方法

+   `generators`-从 Kustomization 文件创建新资源的方法

+   `meta`-设置可以影响生成器、转换器和资源的元数据字段

如果我们想在 Kustomization 文件中指定直接补丁，我们需要使用转换器。前面提到的`PatchStrategicMerge`和`JSONPatch`合并策略是两种转换器。然而，为了直接应用更改到 Kustomization 文件，我们可以使用几种转换器之一，其中包括`commonLabels`、`images`、`namePrefix`和`nameSuffix`。

在下面的 Kustomization 文件中，我们正在使用`commonLabels`和`images`转换器对我们的初始部署`YAML`进行更改。

Deployment-kustomization-1.yaml：

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
namespace: default
commonLabels:
  app: frontend-app
images:
  - name: frontend-myapp
    newTag: 2.0.0
    newName: frontend-app-1
```

这个特定的`Kustomization.yaml`文件将图像标签从`1.0.0`更新为`2.0.0`，将应用程序的名称从`frontend-myapp`更新为`frontend-app`，并将容器的名称从`frontend-myapp`更新为`frontend-app-1`。

要全面了解每个转换器的具体细节，您可以查看[Kustomize 文档](https://kubernetes-sigs.github.io/kustomize/)。Kustomize 文件假定`deployment.yaml`与其自身在同一个文件夹中。

要查看当我们的 Kustomize 文件应用到我们的部署时的结果，我们可以使用 Kustomize CLI 工具。我们将使用以下命令生成经过自定义处理的输出：

```
kustomize build deployment-kustomization1.yaml
```

该命令将给出以下输出：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-myapp
  labels:
    app: frontend-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend-app
  template:
    metadata:
      labels:
        app: frontend-app
    spec:
      containers:
      - name: frontend-app-1
        image: myrepo/myapp:2.0.0
        ports:
        - containerPort: 80
```

如您所见，我们的 Kustomization 文件中的自定义已经应用。因为`kustomize build`命令输出 Kubernetes YAML，我们可以轻松地将输出部署到 Kubernetes，如下所示：

```
kustomize build deployment-kustomization.yaml | kubectl apply -f -
```

接下来，让我们看看如何使用带有`PatchStrategicMerge`的 YAML 文件来修补我们的部署。

### 使用 PatchStrategicMerge 指定更改

为了说明`PatchStrategicMerge`策略，我们再次从相同的`deployment.yaml`文件开始。这次，我们将通过`kustomization.yaml`文件和`patch.yaml`文件的组合来发布我们的更改。

首先，让我们创建我们的`kustomization.yaml`文件，它看起来像这样：

Deployment-kustomization-2.yaml：

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
namespace: default
patchesStrategicMerge:
  - deployment-patch-1.yaml
```

正如您所见，我们的 Kustomization 文件在`patchesStrategicMerge`部分引用了一个新文件`deployment-patch-1.yaml`。这里可以添加任意数量的补丁 YAML 文件。

然后，我们的`deployment-patch-1.yaml`文件是一个简单的文件，镜像了我们的部署并包含我们打算进行的更改。它看起来像这样：

Deployment-patch-1.yaml：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-myapp
  labels:
    app: frontend-myapp
spec:
  replicas: 4
```

这个补丁文件是原始部署中字段的一个子集。在这种情况下，它只是将 `replicas` 从 `2` 更新为 `4`。再次应用更改，我们可以使用以下命令：

```
 kustomize build deployment-kustomization2.yaml
```

但是，我们也可以在 `kubectl` 命令中使用 `-k` 标志！它看起来是这样的：

```
Kubectl apply -k deployment-kustomization2.yaml
```

这个命令相当于以下内容：

```
kustomize build deployment-kustomization2.yaml | kubectl apply -f -
```

与 `PatchStrategicMerge` 类似，我们还可以在我们的 Kustomization 中指定基于 JSON 的补丁 - 现在让我们来看看。

### 使用 JSONPatch 指定更改

要使用 JSON 补丁文件指定更改，该过程与涉及 YAML 补丁的过程非常相似。

首先，我们需要我们的 Kustomization 文件。它看起来像这样：

Deployment-kustomization-3.yaml:

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
namespace: default
patches:
- path: deployment-patch-2.json
  target:
    group: apps
    version: v1
    kind: Deployment
    name: frontend-myapp
```

正如您所看到的，我们的 Kustomize 文件有一个 `patches` 部分，其中引用了一个 JSON 补丁文件以及一个目标。您可以在此部分引用尽可能多的 JSON 补丁。`target` 用于确定在资源部分中指定的哪个 Kubernetes 资源将接收补丁。

最后，我们需要我们的补丁 JSON 本身，它看起来像这样：

Deployment-patch-2.json:

```
[
  {
   "op": "replace",
   "path": "/spec/template/spec/containers/0/name",
   "value": "frontend-myreplacedapp"
  }
]
```

应用此补丁时，将对我们第一个容器的名称执行 `replace` 操作。您可以沿着我们原始的 `deployment.yaml` 文件路径查看，以查看它引用了第一个容器的名称。它将用新值 `frontend-myreplacedapp` 替换此名称。

现在我们已经在 Kubernetes 资源模板化和使用 Kustomize 和 Helm 进行发布方面有了坚实的基础，我们可以继续自动化部署到 Kubernetes。在下一节中，我们将看到两种实现 CI/CD 的模式。

# 了解 Kubernetes 上的 CI/CD 范式 - 集群内和集群外

对 Kubernetes 进行持续集成和部署可以采用多种形式。

大多数 DevOps 工程师将熟悉 Jenkins、TravisCI 等工具。这些工具非常相似，它们提供了一个执行环境来构建应用程序，执行测试，并在受控环境中调用任意的 Bash 脚本。其中一些工具在容器内运行命令，而其他工具则不会。

在涉及 Kubernetes 时，有多种思路和使用这些工具的方式。还有一种较新的 CI/CD 平台，它们与 Kubernetes 原语更紧密地耦合，并且许多平台都是设计在集群本身上运行的。

为了彻底讨论工具如何与 Kubernetes 相关，我们将把我们的流水线分为两个逻辑步骤：

1.  **构建**：编译、测试应用程序、构建容器映像，并发送到映像仓库

1.  **部署**：通过 kubectl、Helm 或其他工具更新 Kubernetes 资源

为了本书的目的，我们将主要关注第二个部署为重点的步骤。虽然许多可用的选项都处理构建和部署步骤，但构建步骤几乎可以发生在任何地方，并且不值得我们在涉及 Kubernetes 具体细节的书中关注。

考虑到这一点，为了讨论我们的工具选项，我们将把我们的工具集分为两个类别，就我们流水线的部署部分而言：

+   集群外 CI/CD

+   集群内 CI/CD

## 集群外 CI/CD

在第一种模式中，我们的 CI/CD 工具运行在目标 Kubernetes 集群之外。我们称之为集群外 CI/CD。存在一个灰色地带，即工具可能在专注于 CI/CD 的单独 Kubernetes 集群中运行，但我们暂时忽略该选项，因为这两个类别之间的差异仍然基本有效。

您经常会发现行业标准的工具，如 Jenkins 与这种模式一起使用，但任何具有运行脚本和以安全方式保留秘钥的能力的 CI 工具都可以在这里工作。一些例子是**GitLab CI**、**CircleCI**、**TravisCI**、**GitHub Actions**和**AWS CodeBuild**。Helm 也是这种模式的重要组成部分，因为集群外 CI 脚本可以调用 Helm 命令来代替 kubectl。

这种模式的一些优点在于其简单性和可扩展性。这是一种“推送”模式，代码的更改会同步触发 Kubernetes 工作负载的更改。

在推送到多个集群时，集群外 CI/CD 的一些弱点是可伸缩性，以及需要在 CI/CD 管道中保留集群凭据，以便它能够调用 kubectl 或 Helm 命令。

## 集群内 CI/CD

在第二种模式中，我们的工具在与我们的应用程序相同的集群上运行，这意味着 CI/CD 发生在与我们的应用程序相同的 Kubernetes 上下文中，作为 pod。我们称之为集群内 CI/CD。这种集群内模式仍然可以使“构建”步骤发生在集群外，但部署步骤发生在集群内。

自从 Kubernetes 发布以来，这些类型的工具已经变得越来越受欢迎，许多使用自定义资源定义和自定义控制器来完成它们的工作。一些例子是 FluxCD、Argo CD、JenkinsX 和 Tekton Pipelines。在这些工具中，GitOps 模式很受欢迎，其中 Git 存储库被用作集群上应该运行什么应用程序的真相来源。

内部 CI/CD 模式的一些优点是可伸缩性和安全性。通过使用 GitOps 操作模型，使集群从 GitHub“拉取”更改，解决方案可以扩展到许多集群。此外，它消除了在 CI/CD 系统中保留强大的集群凭据的需要，而是在集群本身上具有 GitHub 凭据，从安全性的角度来看可能更好。

内部 CI/CD 模式的弱点包括复杂性，因为这种拉取操作略微异步（因为`git pull`通常在循环中发生，不总是在推送更改时发生）。

# 使用 Kubernetes 实现内部和外部 CI/CD

由于在 Kubernetes 中有很多 CI/CD 的选择，我们将选择两个选项并逐一实施它们，这样您可以比较它们的功能集。首先，我们将在 AWS CodeBuild 上实施 CI/CD 到 Kubernetes，这是一个很好的示例实现，可以在任何可以运行 Bash 脚本的外部 CI 系统中重复使用，包括 Bitbucket Pipelines、Jenkins 等。然后，我们将转向 FluxCD，这是一种基于 GitOps 的内部 CI 选项，它是 Kubernetes 原生的。让我们从外部选项开始。

## 使用 AWS CodeBuild 实现 Kubernetes CI

正如前面提到的，我们的 AWS CodeBuild CI 实现将很容易在任何基于脚本的 CI 系统中复制。在许多情况下，我们将使用的流水线 YAML 定义几乎相同。此外，正如我们之前讨论的，我们将跳过容器镜像的实际构建。我们将专注于实际的部署部分。

快速介绍一下 AWS CodeBuild，它是一个基于脚本的 CI 工具，可以运行 Bash 脚本，就像许多其他类似的工具一样。在 AWS CodePipeline 的上下文中，可以将多个独立的 AWS CodeBuild 步骤组合成更大的流水线。

在我们的示例中，我们将同时使用 AWS CodeBuild 和 AWS CodePipeline。我们不会深入讨论如何使用这两个工具，而是将我们的讨论专门与如何将它们用于部署到 Kubernetes 联系起来。

重要提示

我们强烈建议您阅读和审阅 CodePipeline 和 CodeBuild 的文档，因为我们在本章中不会涵盖所有基础知识。您可以在[`docs.aws.amazon.com/codebuild/latest/userguide/welcome.html`](https://docs.aws.amazon.com/codebuild/latest/userguide/welcome.html)找到 CodeBuild 的文档，以及[`docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html`](https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html)找到 CodePipeline 的文档。

在实践中，您将拥有两个 CodePipeline，每个都有一个或多个 CodeBuild 步骤。第一个 CodePipeline 在 AWS CodeCommit 或其他 Git 仓库（如 GitHub）中的代码更改时触发。

这个流水线的第一个 CodeBuild 步骤运行测试并构建容器镜像，将镜像推送到 AWS **弹性容器仓库**（**ECR**）。第一个流水线的第二个 CodeBuild 步骤部署新的镜像到 Kubernetes。

第二个 CodePipeline 在我们提交对 Kubernetes 资源文件（基础设施仓库）的次要 Git 仓库的更改时触发。它将使用相同的流程更新 Kubernetes 资源。

让我们从第一个 CodePipeline 开始。如前所述，它包含两个 CodeBuild 步骤：

1.  首先，测试和构建容器镜像，并将其推送到 ECR

1.  其次，部署更新后的容器到 Kubernetes。

正如我们在本节前面提到的，我们不会在代码到容器镜像的流水线上花费太多时间，但这里有一个示例（不适用于生产）的`codebuild` YAML，用于实现这一步骤：

Pipeline-1-codebuild-1.yaml:

```
version: 0.2
phases:
  build:
    commands:
      - npm run build
  test:
    commands:
      - npm test
  containerbuild:
    commands:
      - docker build -t $ECR_REPOSITORY/$IMAGE_NAME:$IMAGE_TAG .
  push:
    commands:
      - docker push_$ECR_REPOSITORY/$IMAGE_NAME:$IMAGE_TAG
```

这个 CodeBuild 流水线包括四个阶段。CodeBuild 流水线规范是用 YAML 编写的，并包含一个与 CodeBuild 规范版本对应的`version`标签。然后，我们有一个`phases`部分，按顺序执行。这个 CodeBuild 首先运行`build`命令，然后在测试阶段运行`test`命令。最后，`containerbuild`阶段创建容器镜像，`push`阶段将镜像推送到我们的容器仓库。

需要记住的一件事是，CodeBuild 中每个以`$`开头的值都是环境变量。这些可以通过 AWS 控制台或 AWS CLI 进行自定义，并且有些可以直接来自 Git 仓库。

现在让我们看一下我们第一个 CodePipeline 的第二个 CodeBuild 步骤的 YAML：

Pipeline-1-codebuild-2.yaml:

```
version: 0.2
phases:
  install:
    commands:
      - curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.16.8/2020-04-16/bin/darwin/amd64/kubectl  
      - chmod +x ./kubectl
      - mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
      - echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
      - source ~/.bashrc
  pre_deploy:
    commands:
      - aws eks --region $AWS_DEFAULT_REGION update-kubeconfig --name $K8S_CLUSTER
  deploy:
    commands:
      - cd $CODEBUILD_SRC_DIR
      - kubectl set image deployment/$KUBERNETES-DEPLOY-NAME myrepo:"$IMAGE_TAG"
```

让我们来分解这个文件。我们的 CodeBuild 设置分为三个阶段：`install`、`pre_deploy`和`deploy`。在`install`阶段，我们安装 kubectl CLI 工具。

然后，在`pre_deploy`阶段，我们使用 AWS CLI 命令和一些环境变量来更新我们的`kubeconfig`文件，以便与我们的 EKS 集群通信。在任何其他 CI 工具（或者不使用 EKS 时），您可以使用不同的方法为您的 CI 工具提供集群凭据。在这里使用安全选项很重要，因为直接在 Git 仓库中包含`kubeconfig`文件是不安全的。通常，一些环境变量的组合在这里会很好。Jenkins、CodeBuild、CircleCI 等都有它们自己的系统来处理这个问题。

最后，在`deploy`阶段，我们使用`kubectl`来使用第一个 CodeBuild 步骤中指定的新镜像标签更新我们的部署（也包含在一个环境变量中）。这个`kubectl rollout restart`命令将确保为我们的部署启动新的 pod。结合使用`imagePullPolicy`的`Always`，这将导致我们的新应用程序版本被部署。

在这种情况下，我们正在使用 ECR 中特定的镜像标签名称来修补我们的部署。`$IMAGE_TAG`环境变量将自动填充为 GitHub 中最新的标签，因此我们可以使用它来自动将新的容器镜像滚动到我们的部署中。

接下来，让我们来看看我们的第二个 CodePipeline。这个 Pipeline 只包含一个步骤 - 它监听来自一个单独的 GitHub 仓库的更改，我们的“基础设施仓库”。这个仓库不包含应用程序本身的代码，而是 Kubernetes 资源的 YAML 文件。因此，我们可以更改一个 Kubernetes 资源的 YAML 值 - 例如，在部署中的副本数量，并在 CodePipeline 运行后在 Kubernetes 中看到它更新。这种模式可以很容易地扩展到使用 Helm 或 Kustomize。

让我们来看看我们第二个 CodePipeline 的第一个，也是唯一的步骤。

Pipeline-2-codebuild-1.yaml:

```
version: 0.2
phases:
  install:
    commands:
      - curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.16.8/2020-04-16/bin/darwin/amd64/kubectl  
      - chmod +x ./kubectl
      - mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
      - echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
      - source ~/.bashrc
  pre_deploy:
    commands:
      - aws eks --region $AWS_DEFAULT_REGION update-kubeconfig --name $K8S_CLUSTER
  deploy:
    commands:
      - cd $CODEBUILD_SRC_DIR
      - kubectl apply -f .
```

正如您所看到的，这个 CodeBuild 规范与我们之前的规范非常相似。与以前一样，我们安装 kubectl 并准备好与我们的 Kubernetes 集群一起使用。由于我们在 AWS 上运行，我们使用 AWS CLI 来完成，但这可以通过许多方式来完成，包括只需将`Kubeconfig`文件添加到我们的 CodeBuild 环境中。

不同之处在于，我们不是用新版本的应用程序来修补特定部署，而是在管道中运行全面的`kubectl apply`命令，同时将整个基础设施文件夹传输进来。这样一来，Git 中进行的任何更改都会应用到我们集群中的资源上。例如，如果我们通过更改`deployment.yaml`文件中的值，将我们的部署从 2 个副本扩展到 20 个副本，它将在这个 CodePipeline 步骤中部署到 Kubernetes，并且部署将会扩展。

现在我们已经介绍了使用集群外 CI/CD 环境对 Kubernetes 资源进行更改的基础知识，让我们来看看一个完全不同的 CI 范式，其中流水线在我们的集群上运行。

## 使用 FluxCD 实施 Kubernetes CI

对于我们的集群内 CI 工具，我们将使用**FluxCD**。集群内 CI 有几个选项，包括**ArgoCD**和**JenkinsX**，但我们喜欢**FluxCD**相对简单的特点，以及它可以自动更新 Pod 的新容器版本而无需任何额外配置。作为一个额外的变化，我们将使用 FluxCD 的 Helm 集成来管理部署。让我们从安装 FluxCD 开始（我们假设您已经从本章的前几部分安装了 Helm）。这些安装遵循了书写本书时的官方 FluxCD Helm 兼容性安装说明。

官方的 FluxCD 文档可以在[`docs.fluxcd.io/`](https://docs.fluxcd.io/)找到，我们强烈建议您去看一看！FluxCD 是一个非常复杂的工具，我们在本书中只是浅尝辄止。全面的审查不在范围内 - 我们只是试图向您介绍集群内 CI/CD 模式和相关工具。

让我们从在我们的集群上安装 FluxCD 开始我们的审查。

### 安装 FluxCD（H3）

FluxCD 可以在几个步骤中使用 Helm 轻松安装：

1.  首先，我们需要添加 Flux Helm 图表存储库：

```
helm repo add fluxcd https://charts.fluxcd.io
```

1.  接下来，我们需要添加一个自定义资源定义，FluxCD 需要这样做才能与 Helm 发布一起工作：

```
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml
```

1.  在我们安装 FluxCD Operator（这是 FluxCD 在 Kubernetes 上的核心功能）和 FluxCD Helm Operator 之前，我们需要为 FluxCD 创建一个命名空间。

```
kubectl create namespace flux
```

现在我们可以安装 FluxCD 的主要组件，但我们需要为 FluxCD 提供有关我们的 Git 存储库的一些额外信息。

为什么？因为 FluxCD 使用 GitOps 模式进行更新和部署。这意味着 FluxCD 将每隔几分钟主动访问我们的 Git 仓库，而不是响应 Git 钩子，比如 CodeBuild。

FluxCD 还将通过拉取策略响应新的 ECR 镜像，但我们稍后再讨论这一点。

1.  要安装 FluxCD 的主要组件，请运行以下两个命令，并将`GITHUB_USERNAME`和`REPOSITORY_NAME`替换为您将在其中存储工作负载规范（Kubernetes YAML 或 Helm 图表）的 GitHub 用户和仓库。

这组指令假设 Git 仓库是公开的，但实际上它可能不是。由于大多数组织使用私有仓库，FluxCD 有特定的配置来处理这种情况-只需查看文档[`docs.fluxcd.io/en/latest/tutorials/get-started-helm/`](https://docs.fluxcd.io/en/latest/tutorials/get-started-helm/)。事实上，为了看到 FluxCD 的真正力量，无论如何你都需要给它对 Git 仓库的高级访问权限，因为 FluxCD 可以写入你的 Git 仓库，并在创建新的容器镜像时自动更新清单。但是，在本书中我们不会涉及这个功能。FluxCD 的文档绝对值得仔细阅读，因为这是一个具有许多功能的复杂技术。要告诉 FluxCD 要查看哪个 GitHub 仓库，你可以在安装时使用 Helm 设置变量，就像下面的命令一样：

```
helm upgrade -i flux fluxcd/flux \
--set git.url=git@github.com:GITHUB_USERNAME/REPOSITORY_NAME \
--namespace flux
helm upgrade -i helm-operator fluxcd/helm-operator \
--set git.ssh.secretName=flux-git-deploy \
--namespace flux
```

正如你所看到的，我们需要传递我们的 GitHub 用户名，仓库的名称，以及在 Kubernetes 中用于 GitHub 秘钥的名称。

此时，FluxCD 已完全安装在我们的集群中，并指向我们在 Git 上的基础设施仓库！如前所述，这个 GitHub 仓库将包含 Kubernetes YAML 或 Helm 图表，基于这些内容，FluxCD 将更新在集群中运行的工作负载。

1.  为了让 Flux 有实际操作的内容，我们需要创建 Flux 的实际清单。我们使用`HelmRelease` YAML 文件来实现，其格式如下：

helmrelease-1.yaml:

```
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: myapp
  annotations:
    fluxcd.io/automated: "true"
    fluxcd.io/tag.chart-image: glob:myapp-v*
spec:
  releaseName: myapp
  chart:
    git: ssh://git@github.com/<myuser>/<myinfrastructurerepository>/myhelmchart
    ref: master
    path: charts/myapp
  values:
    image:
      repository: myrepo/myapp
      tag: myapp-v2
```

让我们分析一下这个文件。我们正在指定 Flux 将在哪里找到我们应用程序的 Helm 图表的 Git 仓库。我们还使用`automated`注释标记`HelmRelease`，这告诉 Flux 每隔几分钟去轮询容器镜像仓库，看看是否有新版本需要部署。为了帮助这一点，我们包括了一个`chart-image`过滤模式，标记的容器镜像必须匹配才能触发重新部署。最后，在值部分，我们有 Helm 值，将用于 Helm 图表的初始安装。

为了向 FluxCD 提供这些信息，我们只需要将此文件添加到我们的 GitHub 仓库的根目录并推送更改。

一旦我们将这个发布文件`helmrelease-1.yaml`添加到我们的 Git 仓库中，Flux 将在几分钟内捕捉到它，然后查找`chart`值中指定的 Helm 图表。只有一个问题 - 我们还没有制作它！

目前，我们在 GitHub 上的基础设施仓库只包含我们的单个 Helm 发布文件。文件夹内容如下：

```
helmrelease1.yaml
```

为了闭环并允许 Flux 实际部署我们的 Helm 图表，我们需要将其添加到这个基础设施仓库中。让我们这样做，使我们 GitHub 仓库中的最终文件夹内容如下：

```
helmrelease1.yaml
myhelmchart/
  Chart.yaml
  Values.yaml
  Templates/
    … chart templates
```

现在，当 FluxCD 下次检查 GitHub 上的基础设施仓库时，它将首先找到 Helm 发布 YAML 文件，然后将其指向我们的新 Helm 图表。

有了新版本和 Helm 图表的 FluxCD，然后将我们的 Helm 图表部署到 Kubernetes！

然后，每当对 Helm 发布 YAML 或 Helm 图表中的任何文件进行更改时，FluxCD 将捕捉到，并在几分钟内（在其下一个循环中）部署更改。

此外，每当推送一个具有与过滤模式匹配的标签的新容器镜像到镜像仓库时，应用程序的新版本将自动部署 - 就是这么简单。这意味着 FluxCD 正在监听两个位置 - 基础设施 GitHub 仓库和容器仓库，并将部署对任一位置的任何更改。

您可以看到这如何映射到我们的集群外 CI/CD 实现，我们有一个 CodePipeline 来部署我们应用程序容器的新版本，另一个 CodePipeline 来部署对基础设施仓库的任何更改。FluxCD 以一种拉取方式做同样的事情。

# 总结

在本章中，我们学习了关于 Kubernetes 上的模板代码生成。我们回顾了如何使用 Helm 和 Kustomize 创建灵活的资源模板。有了这些知识，您将能够使用任一解决方案模板化您的复杂应用程序，创建或部署发布。然后，我们回顾了 Kubernetes 上的两种 CI/CD 类型；首先是通过 kubectl 将外部 CI/CD 部署到 Kubernetes，然后是使用 FluxCD 的集群内 CI 范例。有了这些工具和技术，您将能够为生产应用程序在 Kubernetes 上设置 CI/CD。

在下一章中，我们将回顾 Kubernetes 上的安全性和合规性，这是当今软件环境中的一个重要主题。

# 问题

1.  Helm 和 Kustomize 模板之间有哪两个区别？

1.  在使用外部 CI/CD 设置时，应如何处理 Kubernetes API 凭据？

1.  为什么在集群内设置 CI 可能比集群外设置更可取？反之呢？

# 进一步阅读

+   Kustomize 文档：https:[`kubernetes-sigs.github.io/kustomize/`](https://kubernetes-sigs.github.io/kustomize/)

+   Helm 文档[`docs.fluxcd.io/en/latest/tutorials/get-started-helm/`](https://docs.fluxcd.io/en/latest/tutorials/get-started-helm/)

# 第九章：介绍 Helm 用于 Kubernetes

在上一章中，我们学习了如何安装和使用 Kustomize。在本章中，让我们了解 Helm（[`helm.sh`](https://helm.sh)）。

Helm 是事实上的 Kubernetes 包管理器，也是在 Kubernetes 上安装任何复杂应用程序的最佳和最简单的方法之一。

Helm 不是`kubectl`的一部分，也没有`kubectl`插件，但它在 Kubernetes 领域中扮演着重要角色，是一个必须了解的工具。

在本章中，我们将学习 Helm v3，特别是如何安装应用程序、升级和回滚应用程序发布、创建和检查 Helm 图表，并使用插件扩展 Helm。

注意

我们将使用 Helm v3，因为它是写作时的最新版本。

在本章中，我们将涵盖以下主要主题：

+   Helm 简介

+   使用 Helm 图表安装应用程序

+   升级 Helm 发布

+   回滚到先前的 Helm 发布

+   使用 Helm 的模板命令

+   创建 Helm 图表

+   使用 Helm 的检查功能

+   使用插件扩展 Helm

# Helm 简介

Helm 是一个 Kubernetes 包管理器，允许开发人员和用户轻松打包、配置、共享和部署 Kubernetes 应用程序到 Kubernetes 集群上。

您可以将 Helm 视为 Homebrew/APT/Yum 包管理器，但用于 Kubernetes。

Helm v3 基于仅客户端架构。它与 Kubernetes API 的连接方式与`kubectl`相同，使用包含 Kubernetes 集群连接设置的`kubeconfig`文件。因此，在`kubectl`可用的地方，Helm CLI 也可以使用，使用相同的`kubectl`功能和权限。

为了更好地理解 Helm，您应该熟悉以下概念：

+   Helm CLI：与 Kubernetes API 交互并执行各种功能的命令行工具，如安装、升级和删除 Helm 发布。

+   图表：这是描述 Kubernetes 资源的模板文件集合。

+   图表模板化：图表中使用的 Helm 图表模板语言。

+   存储库：Helm 存储库是存储和共享打包图表的位置。

+   发布：部署到 Kubernetes 集群的图表的特定实例。

让我们在接下来的章节中详细了解每一个。

## Helm CLI

Helm CLI 可以使用以下命令在不同操作系统上安装：

+   在 macOS 上的安装如下进行：

[PRE0]

+   在 Windows 上安装使用以下命令进行：

[PRE1]

+   在 Linux 上安装如下进行：

[PRE2]

您可以使用`helm –h`获取所有可用的 Helm CLI 命令。让我们列出最常用的命令以及它们的描述：

+   `helm repo add`: 将 Helm 图表仓库添加到本地缓存列表，之后我们可以引用它从仓库中拉取图表。

+   `helm repo update`: 获取有关图表仓库的最新信息；该信息存储在本地。

+   `helm search repo`: 在给定的仓库中搜索图表。

+   `helm pull`: 从图表仓库下载给定的图表。

+   `helm upgrade -i`: 如果没有发布，则安装它，否则升级发布。

+   `helm ls`: 列出当前命名空间中的发布。如果提供了`-A`标志，它将列出所有命名空间。

+   `helm history`: 打印给定发布的历史修订版本。

+   `helm rollback`: 将发布回滚到先前的修订版本。

+   `helm template`: 在本地渲染图表模板并显示输出。

+   `helm create`: 创建一个图表。

+   `helm lint`: 对图表进行检查。

+   `helm plugin`: 安装、列出、更新和卸载 Helm 插件。

让我们在接下来的章节中更详细地学习每一个。

## Helm 图表

图表是 Helm 的一个包。它是一组描述 Kubernetes 资源的模板文件。它使用模板创建 Kubernetes 清单。

Helm 图表结构示例如下：

![图 9.1 – 图表文件夹布局](img/B16411_09_001.jpg)

图 9.1 – 图表文件夹布局

让我们详细讨论一些前述内容：

+   `Chart.yaml`: 包含有关图表元数据的文件。

+   `charts`: 子图表存储的文件夹。

+   `templates`: 存储模板文件的文件夹。

+   `values.yaml`: 一个 YAML 格式的文件，其中包含图表模板使用的配置数值。这些数值可以是资源、副本计数，或者是镜像仓库和标签等。

提示

要更改数值，建议使用`override-values.yaml`文件，您只需输入要更改的数值。不建议更改随图表提供的默认`values.yaml`文件，因为您可能会丢失对文件新版本中更改的跟踪。

现在我们已经学习了 Helm 图表结构的一些基础知识，让我们深入了解图表模板。

## 图表模板

Helm 最强大的功能是图表模板化。Helm 模板语言基于 Go 语言包`text/template`的语法。使用模板语法的值可以用来定制 Kubernetes 资源清单。在安装图表之前，Helm 通过注入指定的值来渲染图表的模板，然后进行图表安装。

值是从默认的`values.yaml`文件中读取的，该文件与图表一起提供，或者是用户提供的文件，例如命名为`override-values.yaml`。这两个文件的值将被合并，然后应用于图表。

让我们看一下以下图表模板示例：

![图 9.2 – 图表模板示例](img/B16411_09_002.jpg)

图 9.2 – 图表模板示例

Helm 模板的上述代码片段是一个 Kubernetes 服务资源，允许我们设置服务类型和端口。如果默认值不符合您的要求，您可以通过提供新的值使用自定义的`override-values.yaml`文件来更改默认值。

其他值，如`name`、`labels`和`selector`，都是从`_helpers.tpl`文件中注入的，这是模板部分的默认位置：

![图 9.3 – _helpers.tpl 的部分示例](img/B16411_09_003.jpg)

图 9.3 – _helpers.tpl 的部分示例

上述代码片段是一个`_helpers.tpl`文件的一部分，定义了要注入到图表模板中的标签和选择器。

## 仓库

仓库是存储和共享打包图表的位置。它可以是任何能够提供文件的 Web 服务器。仓库中的图表以压缩的`.tgz`格式存储。

## 发布

发布是部署到 Kubernetes 集群的图表的特定实例。可以使用相同的发布名称多次安装一个 Helm 图表，每次都会创建一个新的发布版本。

特定发布的发布信息存储在与发布本身相同的命名空间中。

您可以使用相同的发布名称但不同的命名空间无限次安装相同的 Helm 图表。

现在我们已经学习了 Helm 的一些基础知识，让我们深入了解使用图表安装应用程序。

# 使用 Helm 图表安装应用程序

有许多 Helm 图表仓库，逐个设置它们太麻烦了。

相反，我们将使用[`chartcenter.io`](https://chartcenter.io)作为我们的中央 Helm 图表存储库，该存储库拥有 300 多个 Helm 存储库，并且可以成为我们安装所有图表的单一真相来源。它还有一个很好的 UI，您可以在其中搜索图表并获取非常详细的信息：

![图 9.4 – ChartCenter UI](img/B16411_09_004.jpg)

图 9.4 – ChartCenter UI

上述截图显示了 ChartCenter 的 UI。

将 ChartCenter 设置为中央 Helm 存储库也非常容易，如下所示：

[PRE3]

上述命令添加了`center`图表存储库，并使用其内容更新了 Helm 本地缓存。

现在我们可以尝试通过运行`$ helm search repo center/bitnami/postgresql -l | head -n 5`命令来搜索`postgresql`图表：

![图 9.5 – 搜索 PostgreSQL 图表](img/B16411_09_005.jpg)

图 9.5 – 搜索 PostgreSQL 图表

在上述截图中，我们可以看到 Bitnami PostgreSQL 图表的最新五个版本。

在安装 PostgreSQL 图表之前，我们应该设置一个密码，因为设置自己的密码而不是使用 Helm 图表生成的密码是一个好习惯。

通过阅读[`chartcenter.io/bitnami/postgresql`](https://chartcenter.io/bitnami/postgresql)上的图表`README`，我们可以找到需要使用的值名称：

![图 9.6 – PostgreSQL 图表密码](img/B16411_09_006.jpg)

图 9.6 – PostgreSQL 图表密码

上述截图向我们展示了`values.yaml`文件中`postgresqlPassword`变量需要设置 PostgreSQL 图表的密码。

首先，让我们创建一个`password-values.yaml`文件来存储 PostgreSQL 密码：

[PRE4]

然后使用以下命令进行安装：

[PRE5]

上述命令的输出显示在以下截图中：

![图 9.7 – Helm 安装 PostgreSQL 图表](img/B16411_09_007.jpg)

图 9.7 – Helm 安装 PostgreSQL 图表

上述命令将 PostgreSQL 图表安装到当前命名空间，并命名为`postgresql`。

提示

上述`helm upgrade`命令具有一个`–i`标志（长名称为`--install`），允许我们在第一次安装和随后的升级中使用相同的命令。

使用以下命令检查使用该图表安装了什么：

[PRE6]

上述命令的输出显示在以下截图中：

![图 9.8 – 列出所有已安装的资源](img/B16411_09_008.jpg)

图 9.8 - 列出所有已安装的资源

在前面的截图中，我们可以看到`postgresql` pod，两个与`postgresql`相关的服务，以及`statefulset`。查看`service/postgresql`，我们可以看到`postgresql`可以被其他 Kubernetes 应用访问，端口为`postgresql:5432`。

让我们通过运行以下命令检查所有秘钥是否正确创建：

[PRE7]

上述命令的输出如下截图所示：

![图 9.9 - 列出所有已安装的秘钥](img/B16411_09_009.jpg)

图 9.9 - 列出所有已安装的秘钥

在前面的截图中，我们看到了`postgresql`秘钥，其中存储了 PostgreSQL 密码，以及`sh.helm.release.v1.postgresql.v1`，其中存储了 Helm 发布信息。

现在，让我们通过运行以下命令检查当前命名空间中的 Helm 发布：

[PRE8]

上述命令的输出如下截图所示：

![图 9.10 - 列出 Helm 发布](img/B16411_09_010.jpg)

图 9.10 - 列出 Helm 发布

在前面的截图中，我们看到了一个成功部署的`postgresql` Helm 发布，其中我们列出了以下内容：

+   `STATUS`：显示发布状态为`deployed`

+   `CHART`：显示图表名称和版本为`postgresql-9.2.1`

+   `APP VERSION`：显示 PostgreSQL 版本；在这种情况下为`11.9.0`

这很容易安装 - 我们只需要提供密码，然后，我们就有了一个完全安装好的 PostgreSQL 实例，甚至它的密码也存储在秘钥中。

# 升级 Helm 发布

在前一节中，我们安装了 PostgreSQL，现在让我们尝试升级它。我们需要知道如何做这个，因为它将不时地需要升级。

对于升级，我们将使用最新可用的 PostgreSQL 图表版本，即`9.3.2`。

让我们通过以下命令获取并运行升级：

[PRE9]

上述命令的输出如下截图所示：

![图 9.11 - 列出 Helm 发布](img/B16411_09_011.jpg)

图 9.11 - 列出 Helm 发布

我们运行了上述的`helm upgrade`命令，将`postgresql`图表版本更改为`9.3.2`，但我们看到 PostgreSQL 版本仍然与之前相同，即`11.9.0`，这意味着图表本身接收了一些更改，但应用程序版本保持不变。

运行`helm ls`显示`REVISION 2`，这意味着 PostgreSQL 图表的第二次发布。

让我们通过运行以下命令再次检查秘钥：

[PRE10]

上述命令的输出显示在以下截图中：

![图 9.12 - 列出 Helm 发布](img/B16411_09_012.jpg)

图 9.12 - 列出 Helm 发布

从前面的截图中，我们可以看到一个新的秘钥，`sh.helm.release.v1.postgresql.v2`，这是存储了 PostgreSQL 升级发布的地方。

看到 Helm 如何跟踪所有发布并允许使用单个`helm upgrade`命令轻松进行应用程序升级是件好事。

注意

Helm 发布包含图表中的所有 Kubernetes 模板，这使得跟踪它们（从发布的角度）作为一个单一单元变得更加容易。

让我们学习如何进行发布回滚。我们这样做是因为，有时发布可能出现问题，需要回滚。

# 回滚到先前的 Helm 发布

在本节中，让我们看看如何使用`helm rollback`命令回滚到先前的版本。

`helm rollback`命令是 Helm 独有的，它允许我们回滚整个应用程序，因此您不必担心需要特别回滚哪些 Kubernetes 资源。

当处理真实应用程序的发布 ID 时，数据库模式也会发生变化，因此要回滚前端应用程序，您还必须回滚数据库模式更改。这意味着事情并不总是像在这里看起来的那样简单，但使用 Helm 仍然简化了应用程序发布回滚过程的某些部分。

要运行`helm rollback`命令，我们首先需要知道要回滚到的发布修订版本，我们可以使用以下命令找到它：

[PRE11]

上述命令的输出显示在以下截图中：

![图 9.13 - 列出 Helm 发布修订版本](img/B16411_09_013.jpg)

图 9.13 - 列出 Helm 发布修订版本

在前面的`helm history postgresql`命令中，我们得到了一个发布修订版本的列表。

因此，我们要将`postgresql`回滚到修订版本`1`：

[PRE12]

上述命令的输出显示在以下截图中：

![图 9.14 - Helm 回滚发布](img/B16411_09_014.jpg)

图 9.14 - Helm 回滚发布

在前面的截图中，我们看到使用`helm rollback postgresql 1`命令进行了回滚，现在我们看到了三个修订版本，即使进行回滚，也会创建一个新的发布。

正如您所看到的，回滚到先前的发布非常容易。

# 使用 Helm 的模板命令

使用 Helm 的`helm template`命令，您可以检查图表的完全渲染的 Kubernetes 资源模板的输出。这是一个非常方便的命令，特别是在开发新图表、对图表进行更改、调试等情况下，用于检查模板的输出。

因此，让我们通过运行以下命令来检查它：

[PRE13]

前面的命令将在屏幕上打印所有模板。当然，您也可以将其输出到文件中。

由于输出非常长，我们不打印所有内容，而只打印部分 Kubernetes 清单：

[PRE14]

前面的输出显示了所有属于`postgresql`图表的资源。资源使用`---`分隔。

`helm template`是一个强大的命令，用于检查图表的模板并打印输出，以便您阅读。`helm template`不连接到 Kubernetes 集群，它只填充模板的值并打印输出。

您也可以通过向`helm upgrade`命令添加`--dry-run --debug`标志来实现相同的效果。使用这种方式，Helm 将根据 Kubernetes 集群验证模板。

完整命令的示例如下：

[PRE15]

我们已经学会了一些在安装或升级 Helm 发布之前使用的方便的 Helm 命令。

使用`helm template`的另一个强大用例是将模板渲染到文件中，然后进行比较。这对比较图表版本或自定义参数对最终输出的影响非常有用。

# 创建 Helm 图表

我们已经学会了许多有关 Helm 的技巧！现在让我们学习如何创建 Helm 图表。

`helm create`命令为您创建了一个示例图表，因此您可以将其用作基础，并使用所需的 Kubernetes 资源、值等进行更新。它创建了一个完全可用的`nginx`图表，因此我们将以该名称命名图表。

现在让我们通过运行以下命令来检查创建图表有多容易：

[PRE16]

前面命令的输出显示在以下屏幕截图中：

![图 9.15 – helm create 命令](img/B16411_09_015.jpg)

图 9.15 – helm create 命令

在前面的屏幕截图中，我们运行了`helm create nginx`命令，其中`nginx`是我们的图表名称。该名称也用于创建一个新的文件夹，其中将存储图表内容。文件夹结构使用`tree nginx`命令显示。

正如您在截图中所看到的，`deployment.yaml`文件、**水平 Pod 自动缩放器**（**HPA**）、`ingress`、`service`和`serviceaccount`资源模板都已创建，这些资源提供了一个良好的起点。

上述命令还创建了`test-connection.yaml`文件，因此我们可以对安装的`nginx`图表运行`helm test`进行测试。

现在让我们通过运行以下命令来安装图表：

[PRE17]

上述命令的输出显示在以下截图中：

![图 9.16 - 安装 nginx 图表](img/B16411_09_016.jpg)

图 9.16 - 安装 nginx 图表

在上述截图中，我们运行了`helm install nginx nginx`。该命令使用以下基本语法：

[PRE18]

在这里，`<CHART NAME>`是本地文件夹，因此请注意您可以使用相同的命令从远程 Helm 存储库和本地文件夹安装图表。

我们使用的下一个命令如下：

[PRE19]

该命令帮助我们展示了图表默认部署的资源。

正如我们之前提到的`helm test`命令，让我们来看看该命令的功能：

[PRE20]

上述命令的输出显示在以下截图中：

![图 9.17 - 测试 nginx 图表](img/B16411_09_017.jpg)

图 9.17 - 测试 nginx 图表

上述的`helm test nginx`命令针对名为`nginx`的 Helm 发布运行测试。`kubectl get pods`命令的输出显示了用于运行图表测试的`nginx-test-connection` pod，然后被停止。

接下来，让我们检查`test-connection.yaml`文件的内容：

[PRE21]

上述命令的输出显示在以下截图中：

![图 9.18 - test-connection.yaml 内容](img/B16411_09_018.jpg)

图 9.18 - test-connection.yaml 内容

在上面的截图中，您可以看到一个简单的 pod 模板，该模板针对`nginx`服务资源运行`curl`命令。

当实际的 Kubernetes 资源被创建时，模板代码的`args: ['{{ include "nginx.fullname" . }}:{{ .Values.service.port }}']`行会被转换为`nginx:80`。

简单易行，对吧？正如我们所看到的，`helm create`命令创建了一个带有示例资源模板的工作图表，甚至包括测试模板。

# 使用 Helm 的 linting 功能

到目前为止，我们已经学会了如何创建 Helm 图表。然而，我们还需要知道如何检查图表是否存在可能的问题和错误。为此，我们可以使用`helm lint <CHART NAME>`命令，它将通过运行一系列测试来验证图表的完整性。

让我们`lint`我们创建的`nginx`图表：

[PRE22]

上一个命令的输出如下截图所示：

![图 9.19 - 对 nginx 图表进行 lint](img/B16411_09_019.jpg)

图 9.19 - 对 nginx 图表进行 lint

正如你在上面的截图中所看到的，我们的图表没有问题，可以安全地安装。`[INFO]`消息只是警告说图表的图标丢失了，可以安全地忽略。

如果你想在[`chartcenter.io`](https://chartcenter.io)中托管你的图表并在其 UI 中显示，那么强烈建议你这样做。

# 使用插件扩展 Helm

Helm 也可以通过插件进行扩展。插件对于扩展 Helm CLI 中没有的功能非常有用，因为 Helm 可能没有你需要的一切。

目前还没有中央 Helm 插件存储库，你可以在那里看到所有可用插件的列表，也没有 Helm 插件管理器。

由于大多数插件都存储在 GitHub 存储库中，并且建议使用 GitHub 主题`helm-plugin`来标记插件，你可以在那里轻松搜索可用的插件：

![图 9.20 - 在 GitHub 上搜索 Helm 插件](img/B16411_09_020.jpg)

图 9.20 - 在 GitHub 上搜索 Helm 插件

在上面的截图中使用了[`github.com/search?q=helm-plugin`](https://github.com/search?q=helm-plugin)来在 GitHub 上搜索 Helm 插件。

让我们看看安装 Helm 插件有多容易：

[PRE23]

上一个命令的输出如下截图所示：

![图 9.21 - 安装 Helm 插件 helm-diff](img/B16411_09_021.jpg)

图 9.21 - 安装 Helm 插件 helm-diff

在上一个命令`helm plugin list`中，我们检查了已安装的插件，然后我们使用`helm plugin` install [`github.com/databus23/helm-diff`](https://github.com/databus23/helm-diff)来安装`helm-diff`插件。之前的插件安装输出被截断了，因为安装的插件打印了大量信息。

让我们来检查插件列表：

[PRE24]

上一个命令的输出如下截图所示：

![图 9.22 - Helm 插件列表](img/B16411_09_022.jpg)

图 9.22 - Helm 插件列表

我们看到`diff`插件已安装，这基本上是一个新的 Helm 命令：`helm diff`。

我们不打算检查`helm diff`的工作原理，但它非常方便，因为您可以检查已安装和新图表版本之间的差异。

让我们再安装一个：

[PRE25]

上述命令的输出如下截图所示：

![图 9.23 - Helm 插件安装 helm-kubeval](img/B16411_09_023.jpg)

图 9.23 - helm 插件安装 helm-kubeval

上述命令`helm plugin install` [`github.com/instrumenta/helm-kubeval`](https://github.com/instrumenta/helm-kubeval) 安装了`kubeval`插件，该插件验证 Helm 图表与 Kubernetes 模式的匹配情况。

让我们验证之前使用`helm create`创建的`nginx`图表：

[PRE26]

上述命令的输出如下截图所示：

![图 9.24 - 使用 kubeval 插件验证 nginx 图表](img/B16411_09_024.jpg)

图 9.24 - 使用 kubeval 插件验证 nginx 图表

上述`helm kubeval nginx`命令验证了`nginx`图表 - 正如我们所看到的，一切都是绿色的，所以没有问题。该插件是`helm lint`命令的很好补充，两者结合起来可以为您提供良好的工具来检查图表。

现在，我们知道如何使用 Helm 来扩展额外的功能，因为一个工具不能包含所有功能。编写插件也很容易，当然您可以在自己的时间里学习。

# 总结

在本章中，我们学习了如何使用 Helm 来安装、升级、回滚发布、检查图表模板的输出、创建图表、对图表进行 lint 检查，并使用插件扩展 Helm。

Helm 是一个强大的工具，您可以使用它部署简单和复杂的 Kubernetes 应用程序。它将帮助您部署真实世界的应用程序，特别是因为有许多不同的图表可以从许多 Helm 仓库中使用。

在本书的最后一章中，我们将学习`kubectl`的最佳实践和 Docker 用户的`kubectl`命令。

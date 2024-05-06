# 第六章：测试 Helm 图表

测试是工程师在软件开发过程中必须执行的常见任务。测试是为了验证产品的功能性，以及在产品随着时间的推移而发展时防止回归。经过充分测试的软件更容易随着时间的推移进行维护，并允许开发人员更有信心地向最终用户提供新版本。

为了确保 Helm 图表能够按预期的质量水平提供其功能，应该对其进行适当的测试。在本章中，我们将讨论如何实现强大的 Helm 图表测试，包括以下主题：

+   设置您的环境

+   验证 Helm 模板

+   在一个实时集群中进行测试

+   通过图表测试项目改进图表测试

+   清理

# 技术要求

本章将使用以下技术：

+   `minikube`

+   `kubectl`

+   `helm`

+   `git`

+   `yamllint`

+   `yamale`

+   `chart-testing` (`ct`)

除了这些工具，您还可以在 Packt GitHub 存储库中跟随示例，该存储库位于[`github.com/PacktPublishing/-Learn-Helm`](https://github.com/PacktPublishing/-Learn-Helm)，本章将引用该存储库。在本章中使用的许多示例命令中，我们将引用 Packt 存储库，因此您可能会发现通过运行`git clone`命令克隆此存储库会很有帮助：

[PRE0]

现在，让我们继续设置您的本地`minikube`环境。

# 设置您的环境

在本章中，我们将为上一章创建的`Guestbook`图表创建并运行一系列测试。运行以下步骤来设置您的`minikube`环境，在这里我们将测试 Guestbook 图表：

1.  通过运行`minikube start`命令启动`minikube`：

[PRE1]

1.  然后，创建一个名为`chapter6`的新命名空间：

[PRE2]

准备好您的`minikube`环境后，让我们开始讨论如何测试 Helm 图表。我们将首先讨论您可以使用的方法来验证您的 Helm 模板。

# 验证 Helm 模板

在上一章中，我们从头开始构建了一个 Helm 图表。最终产品非常复杂，包含参数化、条件模板和生命周期钩子。由于 Helm 的主要目的之一是创建 Kubernetes 资源，因此在将资源模板应用到 Kubernetes 集群之前，您应该确保这些资源模板被正确生成。这可以通过多种方式来完成，我们将在下一节中讨论。

## 使用 helm template 在本地验证模板生成

验证图表模板的第一种方法是使用`helm template`命令，该命令可用于在本地呈现图表模板并在标准输出中显示其完全呈现的内容。

`helm template`命令具有以下语法：

[PRE3]

此命令在本地呈现模板，使用`NAME`参数满足`.Release`内置对象，使用`CHART`参数表示包含 Kubernetes 模板的图表。Packt 存储库中的`helm-charts/charts/guestbook`文件夹可用于演示`helm template`命令的功能。该文件夹包含在上一节中开发的图表，以及稍后在本章中将使用的其他资源。

通过运行以下命令在本地呈现`guestbook`图表：

[PRE4]

此命令的结果将显示每个 Kubernetes 资源，如果将其应用于集群，将会创建这些资源，如下所示：

![图 6.1 - 用于 guestbook 图表的 ConfigMap](img/Figure_6.1.jpg)

图 6.1 - "helm template"输出

前面的屏幕截图显示了针对上一章中创建的 Guestbook 图表执行的`helm template`命令的输出的开始部分。正如您所看到的，显示了一个完全呈现的`ConfigMap`，以及另一个`ConfigMap`的开始，该`ConfigMap`是使用该版本创建的。在本地呈现这些资源可以让您了解如果将该版本安装到 Kubernetes 集群中，将会创建哪些确切的资源和规范。

在图表开发过程中，您可能希望定期使用`helm template`命令来验证您的 Kubernetes 资源是否被正确生成。

您可能想要验证图表开发的一些常见方面，包括以下内容：

+   参数化字段成功地被默认值或覆盖值替换

+   控制操作，如`if`、`range`和`with`，根据提供的值成功生成 YAML 文件

+   资源包含适当的间距和缩进

+   函数和管道被正确使用以正确格式化和操作 YAML 文件

+   诸如`required`和`fail`之类的函数根据用户输入正确验证值

了解图表模板如何在本地呈现后，现在让我们深入一些特定方面，您可以通过利用`helm template`命令进行测试和验证。

### 测试模板参数化

重要的是要检查模板的参数是否成功填充了值。这很重要，因为您的图表可能由多个不同的值组成。您可以通过确保每个值具有合理的默认值或具有验证来确保您的图表被正确参数化，如果未提供值，则验证失败图表呈现。

想象以下部署：

[PRE5]

在图表的`values.yaml`文件中应定义`replicas`和`port`值的合理默认值，如下所示：

[PRE6]

运行`helm template`命令针对此模板资源呈现以下部署，将`replicas`和`port`值替换为它们的默认值：

[PRE7]

`helm template`的输出允许您验证参数是否被其默认值正确替换。您还可以通过向`helm template`命令传递`--values`或`--set`参数来验证提供的值是否成功覆盖：

[PRE8]

生成的模板反映了您提供的值：

[PRE9]

虽然具有默认设置的值通常很容易通过`helm template`进行测试，但更重要的是测试需要验证的值，因为无效的值可能会阻止图表正确安装。

您应该使用`helm template`来确保具有限制的值，例如仅允许特定输入的值，通过`required`和`fail`函数成功验证。

想象以下部署模板：

[PRE10]

如果此部署属于具有相同`values`文件的图表，并且您期望用户提供`imageRegistry`和`imageName`值来安装图表，如果您然后使用`helm template`命令而不提供这些值，则结果不尽如人意，如下输出所示：

[PRE11]

由于没有设置门控，呈现的结果是一个具有无效图像的部署，`/`。因为我们使用了`helm template`进行测试，所以我们知道需要处理这些值未定义的情况。可以通过使用`required`函数来提供验证，以确保这些值被指定：

[PRE12]

当对具有更新的部署模板的图表应用`helm template`命令时，结果会显示一条消息，指示用户提供模板引擎遇到的第一个缺失的值：

[PRE13]

您还可以通过在`helm template`命令旁边提供有效的值文件来进一步测试此验证。例如，我们假设以下值是在用户管理的`values`文件中提供的：

[PRE14]

然后在执行以下命令时提供此文件：

[PRE15]

作为参数化的一般准则，请确保跟踪您的值，并确保每个值在您的图表中都有用到。在`values.yaml`文件中设置合理的默认值，并在无法设置默认值的情况下使用`required`函数。使用`helm template`函数确保值被正确渲染并产生期望的 Kubernetes 资源配置。

另外，您可能还希望考虑在您的`values.yaml`文件中包含必需的值，将其作为空字段，并在注释中指出它们是必需的。这样用户就可以查看您的`values.yaml`文件，并查看您的图表支持的所有值，包括他们必须自己提供的值。在添加了`imageRegistry`和`imageName`值后，考虑以下`values`文件：

[PRE16]

尽管这些值写在您的图表的`values.yaml`文件中，但当`helm template`命令运行时，这些值仍然会评估为 null，提供与之前执行时相同的行为。不同之处在于现在您可以明确地看到这些值是必需的，因此当您首次尝试安装图表时，您不会感到惊讶。

接下来，我们将讨论如何在本地生成您的图表模板可以帮助您测试图表的控制操作。

### 测试控制操作

除了基本的参数化，您还应该考虑使用`helm template`命令来验证控制操作（特别是`if`和`range`）是否被正确处理以产生期望的结果。

考虑以下部署模板：

[PRE17]

如果`env`和`enableLiveness`的值都是`null`，您可以通过运行`helm template`命令来测试此渲染是否仍然成功：

[PRE18]

您会注意到`range`和`if`操作均未生成。对于`range`子句，空值或空值不会有任何条目对其进行操作，并且当提供给`if`操作时，这些值也被评估为`false`。通过向`helm template`提供`env`和`enableLiveness`值，您可以验证您已经正确编写了模板以使用这些操作生成 YAML。

您可以将这些值添加到一个`values`文件中，如下所示：

[PRE19]

进行这些更改后，验证`helm template`命令的期望结果，以证明模板已正确编写以使用这些值：

[PRE20]

您应该确保在向图表添加额外的控制结构时，定期使用`helm template`渲染您的模板，因为这些控制结构可能会使图表开发过程变得更加困难，特别是如果控制结构数量众多或复杂。

除了检查控制结构是否正确生成外，您还应检查您的函数和流水线是否按预期工作，接下来我们将讨论这一点。

### 测试函数和流水线

`helm template`命令还可以用于验证函数和流水线生成的渲染结果，这些函数和流水线通常用于生成格式化的 YAML。

以以下模板为例：

[PRE21]

此模板包含一个流水线，该流水线对`resources`值进行参数化和格式化，以指定容器的资源需求。在您的图表的`values.yaml`文件中包含一个明智的默认值，以确保应用程序具有适当的限制，以防止集群资源的过度利用。

此模板的`resources`值示例如下：

[PRE22]

您需要运行`helm template`命令，以确保该值被正确转换为有效的`YAML`格式，并且输出被正确缩进以生成有效的部署资源。

对此模板运行`helm template`命令的结果如下：

[PRE23]

接下来，我们将讨论如何在使用`helm template`渲染资源时启用服务器端验证。

向图表渲染添加服务器端验证

虽然`helm template`命令对图表开发过程很重要，并且应该经常用于验证图表渲染，但它确实有一个关键的限制。`helm template`命令的主要目的是提供客户端渲染，这意味着它不会与 Kubernetes API 服务器通信以提供资源验证。如果您希望在生成资源后确保资源有效，可以使用`--validate`标志指示`helm template`在生成资源后与 Kubernetes API 服务器通信：

[PRE24]

任何生成的模板如果未生成有效的 Kubernetes 资源，则会提供错误消息。例如，假设使用了一个部署模板，其中`apiVersion`值设置为`apiVersion: v1`。为了生成有效的部署，必须将`apiVersion`值设置为`apps/v1`，因为这是提供部署资源的 API 的正确名称。仅将其设置为`v1`将通过`helm template`的客户端渲染生成看似有效的资源，但是使用`--validation`标志，您期望看到以下错误：

[PRE25]

`--validate`标志旨在捕获生成的资源中的错误。如果您可以访问 Kubernetes 集群，并且想要确定您的图表是否生成有效的 Kubernetes 资源，则应使用此标志。或者，您可以针对`install`、`upgrade`、`rollback`和`uninstall`命令使用`--dry-run`标志来执行验证。

以下是使用此标志与`install`命令的示例：

[PRE26]

此标志将生成图表的模板并执行验证，类似于使用`--validate`标志运行`helm template`命令。使用`--dry-run`将在命令行打印每个生成的资源，并且不会在 Kubernetes 环境中创建资源。它主要由最终用户使用，在运行安装之前执行健全性检查，以确保他们提供了正确的值，并且安装将产生期望的结果。图表开发人员可以选择以这种方式使用`--dry-run`标志来测试图表渲染和验证，或者他们可以选择使用`helm template`在本地生成图表的资源，并提供`--validate`以添加额外的服务器端验证。

虽然有必要验证您的模板是否按照您的意图生成，但也有必要确保您的模板是按照最佳实践生成的，以简化开发和维护。Helm 提供了一个名为`helm lint`的命令，可以用于此目的，我们将在下面更多地了解它。

## Linting Helm charts and templates

对您的图表进行 lint 是很重要的，可以防止图表格式或图表定义文件中的错误，并在使用 Helm 图表时提供最佳实践的指导。`helm lint`命令具有以下语法：

[PRE27]

`helm lint`命令旨在针对图表目录运行，以确保图表是有效的和正确格式化的。

重要提示：

`helm lint`命令不验证渲染的 API 模式，也不对您的 YAML 样式进行 linting，而只是检查图表是否包含应有的文件和设置，这是一个有效的 Helm 图表应该具有的。

您可以对您在*第五章*中创建的 Guestbook 图表，或者对 Packt GitHub 存储库中`helm-charts/charts/guestbook`文件夹下的图表运行`helm lint`命令，网址为[`github.com/PacktPublishing/-Learn-Helm/tree/master/helm-charts/charts/guestbook`](https://github.com/PacktPublishing/-Learn-Helm/tree/master/helm-charts/charts/guestbook)：

[PRE28]

这个输出声明了图表是有效的，这是由`1 chart(s) linted, 0 chart(s) failed`消息所指出的。`[INFO]`消息建议图表在`Chart.yaml`文件中包含一个`icon`字段，但这并非必需。其他类型的消息包括`[WARNING]`，它表示图表违反了图表约定，以及`[ERROR]`，它表示图表将在安装时失败。

让我们通过一些例子来运行。考虑一个具有以下文件结构的图表：

[PRE29]

请注意，这个图表结构存在问题。这个图表缺少定义图表元数据的`Chart.yaml`文件。对具有这种结构的图表运行 linter 会产生以下错误：

[PRE30]

这个错误表明 Helm 找不到`Chart.yaml`文件。如果向图表中添加一个空的`Chart.yaml`文件以提供正确的文件结构，错误仍会发生，因为`Chart.yaml`文件包含无效的内容：

[PRE31]

对这个图表运行 linter 会产生以下错误：

[PRE32]

此输出列出了在`Chart.yaml`文件中缺少的必需字段。它指示该文件必须包含`name`、`apiVersion`和`version`字段，因此应将这些字段添加到`Chart.yaml`文件中以生成有效的 Helm 图表。检查器还对`apiVersion`和`version`设置提供了额外的反馈，检查`apiVersion`值是否设置为`v1`或`v2`，以及`version`设置是否为正确的`SemVer`版本。

该检查器还将检查其他必需或建议的文件的存在，例如`values.yaml`文件和`templates`目录。它还将确保`templates`目录下的文件具有`.yaml`、`.yml`、`.tpl`或`.txt`文件扩展名。`helm lint`命令非常适合检查图表是否包含适当的内容，但它不会对图表的 YAML 样式进行广泛的 linting。

要执行此 linting，您可以使用另一个名为`yamllint`的工具，该工具可以在[`github.com/adrienverge/yamllint`](https://github.com/adrienverge/yamllint)找到。可以使用以下命令在一系列操作系统上使用`pip`软件包管理器安装此工具：

[PRE33]

也可以按照`yamllint`快速入门说明中描述的方式，使用操作系统的软件包管理器进行安装，该说明位于[`yamllint.readthedocs.io/en/stable/quickstart.html`](https://yamllint.readthedocs.io/en/stable/quickstart.html)。

为了在图表的 YAML 资源上使用`yamllint`，您必须将其与`helm template`命令结合使用，以去除 Go 模板化并生成您的 YAML 资源。

以下是针对 Packt GitHub 存储库中的 guestbook 图表运行此命令的示例：

[PRE34]

此命令将在`templates/`文件夹下生成资源，并将输出传输到`yamllint`。

结果如下所示：

![图 6.2 - 一个示例 yamllint 输出](img/Figure_6.2.jpg)

图 6.2 - 一个示例`yamllint`输出

提供的行号反映了整个`helm template`输出，这可能会使确定`yamllint`输出中的哪一行对应于您的 YAML 资源中的哪一行变得困难。

您可以通过将`helm template`输出重定向到以下命令来确定其行号，针对`guestbook`图表：

[PRE35]

`yamllint`将针对许多不同的规则进行 lint，包括以下内容：

+   缩进

+   行长度

+   训练空间

+   空行

+   注释格式

您可以通过创建以下文件之一来覆盖默认规则：

+   `.yamllint`、`.yamllint.yaml`和`.yamllint.yml`在当前工作目录中

+   `$XDB_CONFIG_HOME/yamllint/config`

+   `~/.config/yamllint/config`

要覆盖针对 guestbook 图表报告的缩进规则，您可以在当前工作目录中创建一个`.yamllint.yaml`文件，其中包含以下内容：

[PRE36]

此配置覆盖了`yamllint`，使其在添加列表条目时不强制执行一种特定的缩进方法。它由`indent-sequences: whatever`行配置。创建此文件并再次针对 guestbook 运行 linter 将消除先前看到的缩进错误：

[PRE37]

在本节中，我们讨论了如何使用`helm template`和`helm lint`命令验证 Helm 图表的本地渲染。然而，这实际上并没有测试您的图表功能或应用程序使用您的图表创建的资源的能力。

在下一节中，我们将学习如何在实时 Kubernetes 环境中创建测试来测试您的 Helm 图表。

# 在实时集群中进行测试

创建图表测试是开发和维护 Helm 图表的重要部分。图表测试有助于验证您的图表是否按预期运行，并且它们可以帮助防止在添加功能和修复图表时出现回归。

测试包括两个不同的步骤。首先，您需要在图表的`templates/`目录下创建包含`helm.sh/hook`: test`注释的`pod`模板。这些`pod`将运行测试您的图表和应用程序功能的命令。接下来，您需要运行`helm test`命令，该命令会启动`test`钩子并创建具有上述注释的资源。

在本节中，我们将学习如何通过向 Guestbook 图表添加测试来在实时集群中进行测试，继续开发您在上一章中创建的图表。作为参考，您将创建的测试可以在 Packt 存储库中的 Guestbook 图表中查看，位于[`github.com/PacktPublishing/-Learn-Helm/tree/master/helm-charts/charts/guestbook`](https://github.com/PacktPublishing/-Learn-Helm/tree/master/helm-charts/charts/guestbook)。

从您的 Guestbook 图表的`templates/`目录下添加`test/frontend-connection.yaml`和`test/redis-connection.yaml`文件开始。请注意，图表测试不一定要位于`test`子目录下，但将它们放在那里是一种很好的方式，可以使您的测试组织和主要图表模板分开：

[PRE38]

在本节中，我们将填充这些文件以验证它们关联的应用程序组件的逻辑。

现在我们已经添加了占位符，让我们开始编写测试。

## 创建图表测试

您可能还记得，Guestbook 图表由 Redis 后端和 PHP 前端组成。用户在前端的对话框中输入消息，并且这些消息将持久保存到后端。让我们编写一些测试，以确保安装后前端和后端资源都可用。我们将从检查 Redis 后端的可用性开始。将以下内容添加到图表的`templates/test/backend-connection.yaml`文件中（此文件也可以在 Packt 存储库中查看：https://github.com/PacktPublishing/-Learn-Helm/blob/master/helm-charts/charts/guestbook/templates/test/backend-connection.yaml）：

![图 6.3 - 对 Guestbook 服务的 HTTP 请求](img/Figure_6.3.jpg)

图 6.3 - Guestbook Helm 图表的后端连接测试

此模板定义了在测试生命周期钩子期间将创建的 Pod。此模板中还定义了一个钩子删除策略，指示何时应删除先前的测试 Pod。如果我们创建的测试需要按顺序运行，还可以添加钩子权重。

容器对象下的 args 字段显示了测试将基于的命令。它将使用 redis-cli 工具连接到 Redis 主服务器并运行命令 MGET messages。Guestbook 前端设计为将用户输入的消息添加到名为 messages 的数据库键中。这个简单的测试旨在检查是否可以连接到 Redis 数据库，并且它将通过查询 messages 键返回用户输入的消息。

PHP 前端也应该进行可用性测试，因为它是应用程序的用户界面组件。将以下内容添加到 templates/test/frontend-connection.yaml 文件中（这些内容也可以在 Packt 存储库 https://github.com/PacktPublishing/-Learn-Helm/blob/master/helm-charts/charts/guestbook/templates/test/frontend-connection.yaml 中找到）。

![图 6.4 - Guestbook Helm 图表的前端连接测试](img/Figure_6.4-1.jpg)

图 6.4 - Guestbook Helm 图表的前端连接测试

这是一个非常简单的测试，它会向 Guestbook 服务发送 HTTP 请求。发送到服务的流量将在 Guestbook 前端实例之间进行负载平衡。此测试将检查负载平衡是否成功执行以及前端是否可用。

现在，我们已经完成了图表测试所需的模板。请注意，这些模板也可以通过 helm 模板命令在本地呈现，并使用 helm lint 和 yamllint 进行检查，如本章前面部分所述。在开发自己的 Helm 图表时，您可能会发现这对于更高级的测试用例很有用。

现在测试已经编写完成，我们将继续在 Minikube 环境中运行它们。

## 运行图表测试

为了运行图表的测试，必须首先使用`helm install`命令在 Kubernetes 环境中安装图表。因为编写的测试是设计在安装完成后运行的，所以可以在安装图表时使用`--wait`标志，以便更容易确定何时 pod 准备就绪。运行以下命令安装 Guestbook 图表：

[PRE39]

安装图表后，可以使用`helm test`命令执行`test`生命周期钩子并创建测试资源。`helm test`命令的语法如下所示：

[PRE40]

针对`my-guestbook`发布运行`helm test`命令：

[PRE41]

如果您的测试成功，您将在输出中看到以下结果：

[PRE42]

在运行测试时，还可以使用`--logs`标志将日志打印到命令行，从而执行测试。

使用此标志再次运行测试：

[PRE43]

您将看到与之前相同的测试摘要，以及每个测试相关的容器日志。以下是前端连接测试日志输出的第一部分：

[PRE44]

以下是后端连接`test`日志输出：

[PRE45]

这次测试的日志将为空，因为您尚未在 Guestbook 前端输入任何消息。您可以在从前端添加消息后再次运行测试，以确保消息持久。在运行安装和`test`套件时，会打印确定 Guestbook 前端 URL 的说明。

这些说明再次显示在这里：

[PRE46]

从浏览器访问前端后，向 Guestbook 应用程序添加一条消息。

以下是一个示例截图：

![图 6.4 - Guestbook 应用程序的前端](img/Figure_6.4.jpg)

图 6.4-1 - Guestbook 应用程序的前端

一旦添加了消息，再次运行`test`套件，使用`--logs`标志显示测试日志。您应该能够通过观察后端连接`test`日志输出来验证是否已添加此消息：

[PRE47]

以下是显示后端连接`test`日志输出的片段。您可以验证消息是否已持久到 Redis 数据库中：

[PRE48]

在本节中，我们编写了简单的测试，作为一个整体，对图表的安装进行了烟雾测试。有了这些测试，我们将更有信心对图表进行更改和添加功能，前提是在每次修改后运行图表测试以确保功能保持不变。

在下一节中，我们将讨论如何通过利用一个名为`ct`的工具来改进测试过程。

# 使用图表测试项目改进图表测试

前一节中编写的测试已足够测试 Guestbook 应用程序是否可以成功安装。然而，标准 Helm 测试过程中存在一些关键限制，需要指出。

要考虑的第一个限制是测试图表值中可能发生的不同排列的困难。因为`helm test`命令无法修改您发布的值，除了在安装或升级时设置的值，所以在针对不同的值设置运行`helm test`时，必须遵循以下工作流程：

1.  使用初始值安装您的图表。

1.  针对您的发布运行`helm test`。

1.  删除您的发布。

1.  使用不同的值集安装您的图表。

1.  重复*步骤 2*到*4*，直到测试了大量的值可能性。

除了测试不同值的排列组合外，您还应确保在修改图表时不会出现回归。防止回归并测试图表的新版本的最佳方法是使用以下工作流程：

1.  安装先前的图表版本。

1.  将您的发布升级到更新的图表版本。

1.  删除发布。

1.  安装更新的图表版本。

对每组值的排列组合重复此工作流程，以确保没有回归或意外的破坏性更改发生。

这些流程听起来很繁琐，但想象一下当维护多个不同的 Helm 图表时，图表开发人员需要进行仔细的测试，会增加额外的压力和维护工作。在维护多个 Helm 图表时，图表开发人员倾向于采用`git` monorepo 设计。当同一个存储库中包含多个不同的构件或模块时，该存储库被认为是 monorepo。

在 Helm 图表的情况下，monorepo 可能具有以下文件结构：

[PRE49]

在修改 Helm 图表时，应对其进行测试，以确保没有意外的破坏性更改发生。当修改图表时，其`Chart.yaml`文件中的`version`字段也应根据正确的`SemVer`版本进行增加，以表示所做更改的类型。`SemVer`版本遵循`MAJOR.MINOR.PATCH`版本编号格式。

使用以下列表作为如何增加`SemVer`版本的指南：

+   如果您对图表进行了破坏性更改，请增加`MAJOR`版本。破坏性更改是指与先前的图表版本不兼容的更改。

+   如果您正在添加一个功能但没有进行破坏性更改，请增加`MINOR`版本。如果您所做的更改与先前的图表版本兼容，应该增加此版本。

+   如果您正在修复错误或安全漏洞，而不会导致破坏性更改，请增加`PATCH`版本。如果更改与先前的图表版本兼容，应该增加此版本。

没有良好编写的自动化，当修改图表并增加它们的版本时，确保测试图表会变得越来越困难，特别是在维护多个 Helm 图表的 monorepo 时。这一挑战促使 Helm 社区创建了一个名为`ct`的工具，以提供图表测试和维护的结构和自动化。我们接下来将讨论这个工具。

## 介绍图表测试项目

图表测试项目可以在[`github.com/helm/chart-testing`](https://github.com/helm/chart-testing)找到，并设计用于针对 git monorepo 中的图表执行自动化的 linting、验证和测试。通过使用 git 检测已更改的图表来实现自动化测试。已更改的图表应该经历测试过程，而未更改的图表则无需进行测试。

该项目的 CLI`ct`提供了四个主要命令：

+   `lint`：对已修改的图表进行 lint 和验证

+   `install`：安装和测试已修改的图表

+   `lint-and-install`：对已修改的图表进行 lint、安装和测试

+   `list-changed`：列出已修改的图表

`list-changed`命令不执行任何验证或测试，而`lint-and-install`命令将`lint`和`install`命令结合起来，对已修改的图表进行`lint`、`install`和`test`。它还会检查您是否已增加了每个图表的`Chart.yaml`文件中修改的图表的`version`字段，并对未增加版本但内容已修改的图表进行测试失败。这种验证有助于维护者根据所做更改的类型保持严格，以增加其图表版本。

除了检查图表版本外，图表测试还提供了为测试目的指定多个值文件的能力。在调用`lint`、`install`和`lint-and-install`命令时，图表测试会循环遍历每个测试`values`文件，以覆盖图表的默认值，并根据提供的不同值排列进行验证和测试。测试`values`文件写在一个名为`ci/`的文件夹下，以将这些值与图表的默认`values.yaml`文件分开，如下例文件结构所示：

[PRE50]

图表测试适用于`ci/`文件夹下的每个`values`文件，无论文件使用的名称如何。您可能会发现，根据被覆盖的值为每个`values`文件命名，以便维护者和贡献者可以理解文件内容，这是有帮助的。

您可能会经常使用的最常见的`ct`命令是`lint-and-install`命令。以下列出了该命令用于 lint、安装和测试在`git` monorepo 中修改的图表的步骤：

1.  检测已修改的图表。

1.  使用`helm repo update`命令更新本地 Helm 缓存。

1.  使用`helm dependency build`命令下载每个修改后的图表的依赖项。

1.  检查每个修改后的图表版本是否已递增。

1.  对于在*步骤 4*中评估为`true`的每个图表，对图表和`ci/`文件夹下的每个`values`文件进行 lint。

1.  对于在*步骤 4*中评估为`true`的每个图表，执行以下附加步骤：

在自动创建的命名空间中安装图表。

通过执行`helm test`来运行测试。

删除命名空间。

在`ci/`文件夹下的每个`values`文件上重复。

正如您所看到的，该命令执行各种不同的步骤，以确保您的图表通过在单独的命名空间中安装和测试每个修改后的图表来正确进行 lint 和测试，重复该过程对`ci/`文件夹下定义的每个`values`文件。然而，默认情况下，`lint-and-install`命令不会通过从图表的旧版本升级来检查向后兼容性。可以通过添加`--upgrade`标志来启用此功能：

如果没有指示有破坏性变化，则`--upgrade`标志会修改*上一组步骤*中的*步骤 6*，通过运行以下步骤：

1.  在自动创建的命名空间中安装图表的旧版本。

1.  通过执行`helm test`来运行测试。

1.  升级发布到修改后的图表版本并再次运行测试。

1.  删除命名空间。

1.  在新的自动创建的命名空间中安装修改后的图表版本。

1.  通过执行`helm test`来运行测试。

1.  再次使用相同的图表版本升级发布并重新运行测试。

1.  删除命名空间。

1.  在`ci/`文件夹下的每个`values`文件上重复。

建议您添加`--upgrade`标志，以便对 Helm 升级进行额外测试，并防止可能的回归。

重要提示：

`--upgrade`标志将不会生效，如果您已经增加了 Helm 图表的`MAJOR`版本，因为这表示您进行了破坏性更改，并且在此版本上进行就地升级将不会成功。

让我们在本地安装图表测试 CLI 及其依赖项，以便稍后可以看到此过程的实际操作。

## 安装图表测试工具

为了使用图表测试 CLI，您必须在本地机器上安装以下工具：

+   `helm`

+   `git`（版本`2.17.0`或更高）

+   `yamllint`

+   `yamale`

+   `kubectl`

图表测试在测试过程中使用这些工具。`helm`和`kubectl`在*第二章*中安装，*准备 Kubernetes 和 Helm 环境*，Git 在*第五章*中安装，*构建您的第一个 Helm 图表*，yamllint 在本章开头安装。如果您迄今为止一直在跟随本书，现在您应该需要安装的唯一先决条件工具是 Yamale，这是图表测试用来验证您的图表的`Chart.yaml`文件与`Chart.yaml`模式文件相匹配的工具。

Yamale 可以使用`pip`软件包管理器安装，如下所示：

[PRE51]

您也可以通过从[`github.com/23andMe/Yamale/archive/master.zip`](https://github.com/23andMe/Yamale/archive/master.zip)手动下载存档来安装 Yamale。

下载后，解压缩存档并运行安装脚本：

[PRE52]

请注意，如果您使用下载的存档安装工具，您可能需要以提升的权限运行`setup.py`脚本，例如在 macOS 和 Linux 上作为管理员或 root 用户。

安装所需的工具后，您应该从项目的 GitHub 发布页面[`github.com/helm/chart-testing/releases`](https://github.com/helm/chart-testing/releases)下载图表测试工具。每个发布版本都包含一个*Assets*部分，其中列出了存档文件。

下载与本地机器平台类型对应的存档。本书使用的版本是`v3.0.0-beta.1`：

![图 6.5 - GitHub 上的图表测试发布页面](img/Figure_6.5.jpg)

图 6.5 - GitHub 上的图表测试发布页面

从 GitHub 发布页面下载适当的文件后，解压缩图表测试版本。解压缩后，您将看到以下内容：

[PRE53]

您可以删除`LICENSE`和`README.md`文件，因为它们是不需要的。

`etc/chart_schema.yaml`和`etc/lintconf.yaml`文件应移动到本地计算机上的`$HOME/.ct/`或`/etc/ct/`位置。`ct`文件应移动到由系统的`PATH`变量管理的位置：

[PRE54]

现在，所有必需的工具都已安装。在本示例中，我们将在本地对 Packt 存储库进行更改，并使用图表测试来对修改后的图表进行 lint 和安装。

如果您尚未将存储库克隆到本地计算机，请立即执行此操作：

[PRE55]

克隆后，您可能会注意到该存储库在顶层有一个名为`ct.yaml`的文件，其中包含以下内容：

[PRE56]

该文件的`chart-dirs`字段指示`ct`，相对于`ct.yaml`文件，`helm-charts/charts`目录是图表 monorepo 的根目录。`chart-repos`字段提供了应该运行`helm repo add`的存储库列表，以确保它能够下载依赖项。

还有许多其他配置可以添加到此文件中，这些将在此时不予讨论，但可以在图表测试文档中查看。每次调用`ct`命令都会引用`ct.yaml`文件。

现在，工具已安装，并且 Packt 存储库已克隆，让我们通过执行`lint-and-install`命令来测试`ct`工具。

运行图表测试 lint-and-install 命令

`lint-and-install`命令针对`Learn-Helm/helm-charts/charts`下包含的三个 Helm 图表使用：

+   `guestbook`：这是您在上一章中编写的 Guestbook 图表。

+   `nginx`：这是我们为演示目的包含的另一个 Helm 图表。通过运行`helm create`命令创建的此图表用于部署`nginx`反向代理。

要运行测试，首先导航到`Learn-Helm`存储库的顶层：

[PRE57]

`ct.yaml`文件通过`chart-dirs`字段显示了图表 monorepo 的位置，因此您可以直接从顶层运行`ct lint-and-install`命令：

[PRE58]

运行此命令后，您将在输出的末尾看到以下消息显示：

![图 6.6 - 当图表没有被修改时的图表测试 lint-and-install 输出](img/Figure_6.6.jpg)

图 6.6 - 当图表没有被修改时的图表测试`lint-and-install`输出

由于这个存储库中的图表都没有被修改，`ct`没有对您的图表执行任何操作。我们应该至少修改其中一个图表，以便看到`lint-and-install`过程发生。修改应该发生在`master`之外的分支上，因此应该通过执行以下命令创建一个名为`chart-testing-example`的新分支：

[PRE59]

修改可以是大的或小的；对于这个例子，我们将简单地修改每个图表的`Chart.yaml`文件。修改`Learn-Helm/helm-charts/charts/guestbook/Chart.yaml`文件的`description`字段如下所示：

[PRE60]

先前，这个值是`A Helm chart for Kubernetes`。

修改`Learn-Helm/helm-charts/charts/nginx/Chart.yaml`文件的`description`字段如下所示：

[PRE61]

先前，这个值是`A Helm chart for Kubernetes`。通过运行`git status`命令验证上次`git`提交后两个图表是否已被修改：

![图 6.7 - 在修改了两个图表后的 git status 输出](img/Figure_6.7.jpg)

图 6.7 - 在修改了两个图表后的`git status`输出

您应该看到`guestbook`和`nginx`图表的变化。修改了这些图表后，尝试再次运行`lint-and-install`命令：

[PRE62]

这次，`ct`确定了这个 monorepo 中两个图表是否发生了更改，如下所示的输出：

![图 6.8 - 指示对 guestbook 和 nginx 图表的更改的消息](img/Figure_6.8.jpg)

图 6.8 - 指示对`guestbook`和`nginx`图表的更改的消息

然而，这个过程后来会失败，因为这两个图表的版本都没有被修改：

![图 6.9 - 当没有图表更改时的输出](img/Figure_6.9.jpg)

图 6.9 - 当没有图表更改时的输出

这可以通过增加`guestbook`和`nginx`图表的版本来解决。由于这个更改没有引入新功能，我们将增加`PATCH`版本。在各自的`Chart.yaml`文件中将两个图表的版本都修改为`version 1.0.1`：

[PRE63]

通过运行`git diff`命令确保每个图表都已进行了此更改。如果在输出中看到每个版本的修改，请继续再次运行`lint-and-install`命令：

[PRE64]

现在图表版本已经增加，`lint-and-install`命令将遵循完整的图表测试工作流程。您将看到每个修改的图表都会被 linted 并部署到自动创建的命名空间中。一旦部署的应用程序的 pod 被报告为就绪状态，`ct`将自动运行每个图表的测试用例，这些测试用例由带有`helm.sh/hook: test`注释的资源表示。图表测试还将打印每个测试 pod 的日志，以及命名空间事件。

您可能会注意到，在`lint-and-install`输出中，`nginx`图表部署了两次，而`guestbook`图表只部署和测试了一次。这是因为`nginx`图表有一个位于`Learn-Helm/helm-charts/charts/nginx/ci/`的`ci/`文件夹，其中包含两个不同的`values`文件。`ci/`文件夹中的`values`文件将被图表测试迭代，该测试将安装与`values`文件数量相同的图表，以确保每个值组合都能成功安装。`guestbook`图表不包括`ci/`文件夹，因此此图表只安装了一次。

这可以在`lint-and-install`输出的以下行中观察到：

[PRE65]

虽然该命令对于测试两个图表的功能很有用，但它并未验证对新版本的升级是否成功。

为此，我们需要向`lint-and-install`命令提供`--upgrade`标志。再次尝试运行此命令，但这次使用`--upgrade`标志：

[PRE66]

这次，每个`ci/`下的`values`文件将进行原地升级。这可以在输出中看到如下：

[PRE67]

请记住，只有在版本之间的`MAJOR`版本相同时，原地升级才会被测试。如果您使用`--upgrade`标志，但未更改`MAJOR`版本，您将看到类似以下的消息：

[PRE68]

现在，通过了解如何使用图表测试对 Helm 图表进行强大的测试，我们将通过清理您的`minikube`环境来结束。

# 清理

如果您已经完成了本章中描述的示例，可以从您的`minikube`集群中删除`chapter6`命名空间：

[PRE69]

最后，通过运行`minikube stop`关闭您的`minikube`集群。

# 摘要

在本章中，您了解了可以应用于测试 Helm 图表的不同方法。测试图表的最基本方法是针对本地图表目录运行`helm template`命令，以确定其资源是否正确生成。您还可以使用`helm lint`命令来确保您的图表遵循正确的格式，并且可以使用`yamllint`命令来检查图表中使用的 YAML 样式。

除了本地模板化和检查外，您还可以使用`helm test`命令和`ct`工具在 Kubernetes 环境中执行实时测试。除了执行图表测试外，图表测试还提供了使图表开发人员更容易在 monorepo 中维护 Helm 图表的功能。

在下一章中，您将了解 Helm 如何在**持续集成/持续交付**（**CI/CD**）和 GitOps 设置中使用，从图表开发人员构建和测试 Helm 图表的角度，以及从使用 Helm 将应用程序部署到 Kubernetes 的最终用户的角度。

# 进一步阅读

有关`helm template`和`helm lint`命令的更多信息，请参阅以下资源：[`helm.sh/docs/helm/helm_template/`](https://helm.sh/docs/helm/helm_template/)

+   `helm template`：[`helm.sh/docs/helm/helm_template/`](https://helm.sh/docs/helm/helm_template/)

+   `helm lint`：[`helm.sh/docs/helm/helm_lint/`](https://helm.sh/docs/helm/helm_lint/)

Helm 文档中的以下页面讨论了图表测试和`helm test`命令：[`helm.sh/docs/topics/chart_tests/`](https://helm.sh/docs/topics/chart_tests/)

+   图表测试：[`helm.sh/docs/topics/chart_tests/`](https://helm.sh/docs/topics/chart_tests/)

+   `helm test`命令：[`helm.sh/docs/helm/helm_test/`](https://helm.sh/docs/helm/helm_test/)

+   最后，请查看有关`ct` CLI 的图表测试 GitHub 存储库的更多信息：[`github.com/helm/chart-testing`](https://github.com/helm/chart-testing)。

# 问题

1.  `helm template`命令的目的是什么？它与`helm lint`命令有何不同？

1.  在将图表模板安装到 Kubernetes 之前，您可以做什么来验证它们？

1.  可以利用哪个工具来检查您的 YAML 资源的样式？

1.  如何创建图表测试？如何执行图表测试？

1.  `ct`工具为 Helm 内置的测试功能带来了什么附加价值？

1.  在使用`ct`工具时，`ci/`文件夹的目的是什么？

1.  `--upgrade` 标志如何改变 `ct lint-and-install` 命令的行为？

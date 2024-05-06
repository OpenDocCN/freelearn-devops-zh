# 第十章：评估

# 第一章：理解 Kubernetes 和 Helm

以下是本章中提出的一些问题的答案：

1.  如果一个应用程序在一个单一应用程序中包含了所有必要的逻辑和功能，那么它就是`单体`的。单体应用程序可以分解成多个不同的应用程序，称为**微服务**。

1.  Kubernetes 是一个容器编排工具。举几个例子，它解决了关于工作负载调度、可用性和可伸缩性的问题。

1.  `创建`、`描述`、`编辑`、`删除`和`应用`

1.  用户必须了解许多不同类型的资源才能部署应用程序。同时，保持本地和实时状态同步、管理应用程序生命周期以及维护样板 YAML 资源文件也是具有挑战性的。

1.  Helm 包括四个生命周期命令，可以让用户轻松管理 Kubernetes 应用程序。用户可以应用这些命令与 Helm 图表进行交互，Helm 图表是部署应用程序所需的 Kubernetes 资源的打包。Helm 抽象了 Kubernetes 资源的复杂性，并为给定应用程序提供了修订历史，允许将应用程序回滚到先前的快照。它还允许动态生成 YAML 资源，并简化了本地和实时状态之间的同步。最后，Helm 按照预定的顺序应用 Kubernetes 资源，并允许自动化的生命周期钩子，可用于执行各种自动化任务。

1.  您可以使用`helm rollback`命令。Helm 为每个应用程序快照分配一个修订版本。当应用程序的一个或多个区域从其先前应用的状态进行修改时，将分配一个新的修订版本。

1.  `安装`、`升级`、`回滚`和`卸载`。

# 第二章：准备 Kubernetes 和 Helm 环境

以下是本章中提出的一些问题的答案：

1.  Windows 和 Mac 用户可以使用 Chocolatey 或 Homebrew 软件包管理器安装 Helm。所有用户（Windows、Mac 和 Linux）也可以从 Helm 的 GitHub 发布页面[`github.com/helm/helm/releases`](https://github.com/helm/helm/releases)安装 Helm。

1.  Helm 使用本地的`kubeconfig`文件进行身份验证。

1.  Kubernetes 角色提供授权。管理员可以通过创建`RoleBinding`来管理这些特权，将角色绑定到用户或组。

1.  helm repo add 命令用于本地配置 Helm 图表存储库。这是安装存储库中包含的图表的要求。

1.  Helm 使用的三个 XDG 环境变量是 XDG_CACHE_HOME、XDG_CONFIG_HOME 和 XDG_DATA_HOME。XDG_CACHE_HOME 用于指定缓存文件的位置（包括从上游图表存储库下载的图表）。XDG_CONFIG_HOME 用于设置 Helm 配置的位置（包括 helm repo add 保存的存储库信息）。XDG_DATA_HOME 用于保存使用 helm plugin install 命令添加的插件信息。

1.  Minikube 允许用户在他们的本地机器上轻松创建单节点 Kubernetes 集群。Minikube 会自动为认证配置 Kubeconfig，并分配给用户 cluster-admin 权限来执行任何所需的操作。

# 第三章：安装您的第一个 Helm 图表

以下是本章提出的问题的一些答案：

1.  Helm Hub 是上游图表存储库的集中位置。用户可以使用 helm search hub 命令与其交互，或者访问 Helm Hub 网站[`hub.helm.sh/`](https://hub.helm.sh/)。

1.  helm get 命令用于获取已安装的 Helm 发布的详细信息，例如应用的值和生成的 Kubernetes 资源。helm show 命令用于显示 Helm 图表的一般信息，例如支持的值列表和图表 README。

1.  --set 标志用于提供内联值，对于提供简单值或包含不应保存到文件的机密的值很有用。--values 标志用于通过使用值文件提供值，对于一次提供大量值并将应用的值保存到源代码控制存储库很有用。

1.  helm history 命令可用于列出发布的修订版本。

1.  如果升级发布而不提供任何值，则默认应用--reuse-values 标志，该标志将重用先前发布中应用的每个值。如果提供了至少一个值，则将应用--reset-values 标志，该标志将将每个值重置为其默认值，然后合并提供的值。

1.  helm history 命令将显示六个发布版本，第六个发布版本表示应用程序已回滚到第 3 个修订版本。

1.  helm list 命令可用于查看部署到命名空间的所有发布。

1.  `helm search repo`命令可用于列出存储库的每个图表。

# 第四章：理解 Helm 图表

以下是本章中提出的一些问题的答案：

1.  YAML 是最常用的格式，尽管也可以使用 JSON。

1.  三个必填字段是`apiVersion`，`name`和`version`。

1.  可以通过在名称等于依赖图表的名称的映射中放置所需的依赖值来引用或覆盖图表依赖的值。还可以使用`import-values`设置导入值，该设置可用于允许使用不同名称引用依赖值。

1.  您可以创建升级钩子以确保在运行`helm upgrade`命令之前进行数据快照。

1.  您可以提供`README.md`文件来为您的图表提供文档。您还可以创建`templates/NOTES.txt`文件，该文件可以在安装时动态生成发布说明。最后，`LICENSE`文件可用于提供法律信息。

1.  `range`操作允许图表开发人员生成重复的 YAML 部分。

1.  `Chart.yaml`文件用于定义有关 Helm 图表的元数据。此文件也称为图表定义。`Chart.lock`文件用于保存图表依赖状态，提供有关所使用的确切依赖版本的元数据，以便可以重新创建`charts/`文件夹。

1.  `helm.sh/hook`注释用于定义钩子资源。

1.  函数和管道允许图表开发人员在模板中执行复杂的处理和数据格式化。常见函数包括`date`，`include`，`indent`，`quote`和`toYaml`。

# 第五章：创建您的第一个 Helm 图表

以下是本章中提出的一些问题的答案：

1.  `helm create`命令可用于创建新的 Helm 图表。

1.  声明 Redis 依赖性使您无需在 Helm 图表中创建 Redis 模板。它允许您部署 Redis 而无需知道所需的正确 Kubernetes 资源配置。

1.  `helm.sh/hook-weight`注释可用于设置执行顺序。按权重升序执行钩子。

1.  `fail`函数用于立即失败渲染，并可用于限制用户输入以符合一组有效设置。`required`函数用于声明必需值，如果未提供该值，则图表模板将失败。

1.  要将 Helm 图表发布到 GitHub Pages 图表存储库，必须首先使用`helm package`命令将 Helm 图表打包为 TGZ 格式。接下来，应使用`helm repo index`命令生成存储库的`index.yaml`文件。最后，存储库内容应推送到 GitHub。

1.  `index.yaml`文件包含有关图表存储库中包含的每个图表的元数据。

# 第六章：测试 Helm 图表

以下是本章提出的一些问题的答案：

1.  `helm template`命令用于在本地生成 Helm 模板。`helm lint`命令用于检查图表结构和图表定义文件中的错误。它还尝试查找会导致安装失败的错误。

1.  在安装之前验证图表模板，可以运行`helm template`命令在本地生成您的 YAML 资源，以确保它们被正确生成。您还可以使用`--verify`标志在不安装资源的情况下与 API 服务器检查您的 YAML 模式是否正确。`helm install --dry-run`命令也可以在安装之前与 API 服务器执行此检查。

1.  可用于检查 YAML 资源样式的工具之一是`yamllint`工具。它可以与`helm template`一起使用来检查生成的资源（例如，`helm template my-test test-chart | yamllint -`）。

1.  创建图表测试是通过创建一个带有`helm.sh/hook: test`注释的图表模板来实现的。图表测试通常是执行脚本或简短命令的 Pod。可以通过运行`helm test`命令来执行它们。

1.  Chart Testing（**ct**）工具允许 Helm 图表维护者更轻松地在 git monorepo 中测试 Helm 图表。它进行彻底的测试，并确保已修改的图表已增加其版本。

1.  `ci/`文件夹用于测试多种不同的 Helm 值组合。

1.  添加`--upgrade`标志将有助于确保对未增加主要版本的图表未发生回归。它将首先安装图表的旧版本，然后升级到新版本。然后，它将删除发布，安装新版本，并尝试对自身进行升级。测试将在每次安装/升级之间进行。

# 第七章：使用 CI/CD 和 GitOps 自动化 Helm 流程

以下是本章提出的一些问题的答案：

1.  CI 是一种自动化的软件开发过程，可以在软件发生变化时重复进行。CD 是一组定义的步骤，用于将软件推进到发布过程中（通常称为管道）。

1.  虽然 CI/CD 描述了软件开发和发布过程，但 GitOps 描述了在 Git 中存储配置的行为。一个例子是将值文件存储在 Git 中，然后应用于将应用程序部署到 Kubernetes。

1.  用于创建和发布 Helm 图表的 CI 管道可以对 Helm 图表进行 lint、安装和测试。Chart 测试工具可以帮助更轻松地执行这些步骤，特别是在维护图表 monorepo 时。管道还应该打包每个 Helm 图表并将图表部署到图表存储库。对于 GitHub Pages 图表存储库，必须生成`index.yaml`文件，并将内容推送到存储库。

1.  CI 允许轻松快速地测试和发布图表。它还可以帮助防止在添加新功能时出现回归。

1.  CD 管道将 Helm 图表部署到每个所需的环境，每个环境都是不同的管道阶段。每次部署后都可以使用`helm test`命令进行烟雾测试。

1.  CD 管道允许用户轻松部署其应用程序，而无需手动调用 Helm CLI 命令。这可以帮助防止在使用 Helm 部署应用程序时出现人为错误的可能性。

1.  为了维护多个环境的配置，可以使用单独的文件夹来按环境分隔值文件。为了减少样板文件，可以保存一个包含每个环境中使用的通用值的文件，并将其应用于每个 Helm 部署。

# 第八章：使用 Operator Framework 的 Helm

以下是本章提出的问题的一些答案：

1.  操作员通过利用自定义控制器和自定义资源来工作。当创建新的自定义资源时，操作员将执行自定义控制器实现的逻辑。对自定义资源的更改也会触发控制器逻辑。操作员通常用于安装和管理应用程序的生命周期。

1.  当使用 Helm CLI 时，您必须从命令行执行`install`、`upgrade`、`rollback`和`uninstall`命令。但是，当使用基于 Helm 的 operator 时，当您`create`、`modify`或`delete`自定义资源时，这些命令将自动执行。当使用基于 Helm 的 operator 时，您不必在本地运行任何 Helm CLI 命令。

关于应用程序生命周期，Helm CLI 允许用户回滚到先前的修订版本，而 Helm operator 不允许这样做，因为它不保留修订版本的历史记录。

1.  您可以首先使用`operator-sdk new`命令来创建一个新的 Helm operator，将该命令指向现有的 Helm 图表，并使用`--helm-chart`标志。接下来，您可以使用`operator-sdk build`命令构建 operator。最后，您可以将 operator 镜像推送到容器注册表。

1.  安装是通过创建新的自定义资源来执行的。升级是通过修改自定义资源来执行的。如果升级失败，回滚将自动执行，但不能显式执行。卸载是通过删除自定义资源来执行的。

1.  `crds/`文件夹允许在创建`templates/`中的内容之前创建**自定义资源定义（CRD）**。它提供了一种轻松的方式来部署依赖于 CRD 的 operator。

1.  答案会有所不同，但已在[`github.com/PacktPublishing/-Learn-Helm/tree/master/ch8-q6-answer`](https://github.com/PacktPublishing/-Learn-Helm/tree/master/ch8-q6-answer)提供了这些图表的示例。该示例创建了一个名为**guestbook-operator**的图表，用于部署 operator 资源（包括 CRD），而另一个图表名为**guestbook-cr**，用于部署自定义资源。

# 第九章：Helm 安全考虑

以下是本章中提出的一些问题的示例答案：

1.  数据溯源是关于确定数据的来源。数据完整性确定您收到的数据是否是您期望的数据。

1.  用户需要下载附带的`.asc`文件，其中包含数字签名。

1.  `helm verify`命令可用于验证本地下载的图表，而`helm install --verify`命令可用于针对存储在上游图表存储库中的图表。

1.  您可以整合常规漏洞扫描。您还可以尝试避免部署需要以 root 或 root 权限子集运行的映像。最后，您可以使用 `sha256` 值引用映像，而不是标签，以确保始终部署预期的映像。

1.  资源限制有助于防止应用程序耗尽底层节点资源。您还可以利用 `LimitRanges` 来设置每个 Pod 或 PVC 的最大资源量，并且可以利用 `ResourceQuotas` 来设置每个命名空间的最大资源量。

1.  最小权限是指仅授予用户或应用程序所需的最小权限集以正常运行。要实现最小权限访问，您可以使用 Kubernetes 的 `Roles` 和 `RoleBindings` 来创建最小权限角色，并将这些角色绑定到用户或组。

1.  `helm repo add` 命令提供了 `--username` 和 `--password` 标志，用于基本身份验证，以及 `--ca-file`、`--cert-file` 和 `--key-file` 标志，用于基于证书的身份验证。`--ca-file` 标志还用于验证图表存储库的证书颁发机构。

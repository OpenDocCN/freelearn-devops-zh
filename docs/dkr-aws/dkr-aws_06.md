# 第六章：构建自定义 ECS 容器实例

在早期的章节中，您学习了如何使用 Amazon ECS-Optimized Amazon Machine Image (AMI)来在几个简单的步骤中创建 ECS 容器实例并将它们加入到 ECS 集群中。尽管 ECS-Optimized AMI 非常适合快速启动和运行，但您可能希望为生产环境的 ECS 容器实例添加其他功能，例如添加日志代理或包括对 HTTP 代理的支持，以便将 ECS 集群放置在私有子网中。

在本章中，您将学习如何使用 ECS-Optimized AMI 作为基础机器映像来构建自定义 ECS 容器实例，并使用一种名为 Packer 的流行开源工具应用自定义。您将扩展基础映像以包括 AWS CloudWatch 日志代理，该代理可使用 CloudWatch 日志服务从您的 ECS 容器实例进行集中日志记录，并安装一组有用的 CloudFormation 辅助脚本，称为 cfn-bootstrap，它将允许您在实例创建时运行强大的初始化脚本，并提供与 CloudFormation 的强大集成功能。

最后，您将创建一个首次运行脚本，该脚本将允许您使您的实例适应目标环境的特定要求，而无需为每个应用程序和环境构建新的 AMI。该脚本将使您有条件地启用 HTTP 代理支持，从而可以在更安全的私有子网中安装您的 ECS 容器实例，并且还将包括一个健康检查，该检查将等待您的 ECS 容器实例已注册到其配置的 ECS 集群，然后向 CloudFormation 发出信号，表明您的实例已成功初始化。

将涵盖以下主题：

+   设计自定义 AMI

+   使用 Packer 构建自定义 AMI

+   创建自定义存储配置

+   安装 CloudFormation 辅助脚本

+   安装 CloudWatch 日志代理

+   创建首次运行脚本

+   测试您的自定义 ECS 容器实例

# 技术要求

以下列出了完成本章所需的技术要求：

+   Packer 1.0 或更高（将提供安装 Packer 的说明）

+   对 AWS 账户的管理员访问权限

+   根据第三章的说明配置本地 AWS 配置文件

+   GNU Make 版本 3.82 或更高（请注意，macOS 默认不包含此版本）

+   AWS CLI 1.15.71 或更高

此 GitHub URL 包含本章中使用的代码示例：[`github.com/docker-in-aws/docker-in-aws/tree/master/ch6`](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch6)[.](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch4)

查看以下视频以查看代码的实际操作：

[`bit.ly/2LzoxaO`](http://bit.ly/2LzoxaO)

# 设计自定义 Amazon Machine Image

在学习如何构建自定义 Amazon Machine Image 之前，了解为什么您想要或需要构建自己的自定义镜像是很重要的。

这取决于您的用例或组织要求，但通常有许多原因您可能想要构建自定义镜像：

+   自定义存储配置：默认的 ECS 优化 AMI 附带一个 30GB 的卷，其中包括 8GB 用于操作系统分区，以及一个 22GB 的卷用于存储 Docker 镜像和容器文件系统。我通常建议更改配置的一个方面是，默认情况下，不使用分层文件系统的 Docker 卷存储在 8GB 的操作系统分区上。这种方法通常不适合生产用例，而应该为存储 Docker 卷挂载一个专用卷。

+   **安装额外的软件包和工具**：与 Docker 的极简主义理念一致，ECS 优化 AMI 附带了一个最小安装的 Amazon Linux，只包括运行 Docker Engine 和支持的 ECS 代理所需的核心组件。对于实际用例，您通常至少需要添加 CloudWatch 日志代理，它支持在系统级别（例如操作系统、Docker Engine 和 ECS 代理日志）记录到 AWS CloudWatch 日志服务。另一个重要的工具集是 cfn-bootstrap 工具，它提供一组 CloudFormation 辅助脚本，您可以在 CloudFormation 模板中定义自定义的配置操作，并允许您的 EC2 实例在配置和实例初始化完成后向 CloudFormation 发出信号。

+   **添加首次运行脚本**：在部署 ECS 容器实例到 AWS 时，您可能会在各种用例中使用它们，这些用例根据应用程序的性质需要不同的配置。例如，一个常见的安全最佳实践是将 ECS 容器实例部署到没有默认路由附加的私有子网中。这意味着您的 ECS 容器实例必须配置 HTTP 代理，以便与 AWS 服务（如 ECS 和 CloudWatch 日志）或 ECS 容器实例可能依赖的任何其他互联网服务进行通信。然而，在某些情况下，使用 HTTP 代理可能不可行（例如，考虑运行为您的环境提供 HTTP 代理服务的 ECS 容器实例），而不是构建单独的机器映像（一个启用了 HTTP 代理和一个未启用 HTTP 代理），您可以创建一次性运行的配置脚本，根据目标用例有条件地启用/禁用所需的配置，例如 HTTP 代理设置。

当然，还有许多其他用例可能会驱使您构建自己的自定义映像，但在本章中，我们将专注于这里定义的用例示例，这将为您提供坚实的基础和理解如何应用您可能想要使用的任何其他自定义的额外定制。

# 使用 Packer 构建自定义 AMI

现在您了解了构建自定义 ECS 容器实例映像的原因，让我们介绍一个名为 Packer 的工具，它允许您为各种平台构建机器映像，包括 AWS。

**Packer**是 HashiCorp 创建的开源工具，您可以在[`www.packer.io/`](https://www.packer.io/)了解更多信息。Packer 可以为各种目标平台构建机器映像，但在本章中，我们将只关注构建 Amazon Machine Images。

# 安装 Packer

在您开始使用 Packer 之前，您需要在本地环境中安装它。Packer 支持 Linux、macOS 和 Windows 平台，要安装 Packer 到您的目标平台，请按照位于[`www.packer.io/intro/getting-started/install.html`](https://www.packer.io/intro/getting-started/install.html)的说明进行操作。

请注意，Packer 在操作系统和第三方软件包管理工具中得到了广泛支持，例如，在 mac OS 上，您可以通过运行`brew install packer`来使用 Brew 软件包管理器安装 Packer。

# 创建 Packer 模板

安装了 Packer 后，您现在可以开始创建一个 Packer 模板，该模板将定义如何构建您的自定义机器镜像。不过，在这之前，我建议为您的 Packer 模板创建一个单独的存储库，该存储库应该始终放置在版本控制下，就像应用程序源代码和其他基础设施一样。

在本章中，我假设您已经创建了一个名为`packer-ecs`的存储库，并且您可以参考[`github.com/docker-in-aws/docker-in-aws`](https://github.com/docker-in-aws/docker-in-aws)的`ch6`文件夹，该文件夹提供了一个基于本章内容的示例存储库。

# Packer 模板结构

Packer 模板是提供了一个声明性描述的 JSON 文档，告诉 Packer 如何构建机器镜像。

Packer 模板围绕着四个常见的顶级参数进行组织，如下例所示，并在这里描述：

+   **variables**：提供构建的输入变量的对象。

+   **builders**：Packer 构建器的列表，定义了目标机器镜像平台。在本章中，您将针对一个名为[EBS-backed AMI builder](https://www.packer.io/docs/builders/amazon-ebs.html)的构建器进行定位，这是用于创建自定义 Amazon Machine Images 的最简单和最流行的构建器。构建器负责确保正确的图像格式，并以适合部署到目标机器平台的格式发布最终图像。

+   **provisioners**：Packer 配置器的列表或数组，作为图像构建过程的一部分执行各种配置任务。最简单的配置器包括文件和 shell 配置器，它们将文件复制到图像中并执行 shell 任务，例如安装软件包。

+   **post-processors**：Packer 后处理器的列表或数组，一旦机器镜像构建和发布完成，将执行后处理任务。

[PRE0]

Packer 模板结构

# 配置构建器

让我们开始配置我们的 Packer 模板，首先在 packer-ecs 存储库的根目录下创建一个名为`packer.json`的文件，然后定义构建器部分，如下例所示：

[PRE1]

定义一个 EBS-backed AMI 构建器

在上述示例中，将表示我们构建器的单个对象添加到构建器数组中。`type`参数将构建器定义为基于 EBS 的 AMI 构建器，并且随后的设置特定于此类型的构建器：

+   `access_key`：定义用于验证对 AWS 的访问权限的 AWS 访问密钥 ID，用于构建和发布 AMI。

+   `secret_key`：定义用于验证对 AWS 的访问权限的 AWS 秘密访问密钥，用于构建和发布 AMI。

+   `token`：可选地定义用于验证临时会话凭据的 AWS 会话令牌。

+   `region`：目标 AWS 地区。

+   `source_ami`：要构建的源 AMI。在此示例中，指定了撰写时 us-east-1 地区最新的 ECS-Optimized AMI 的源 AMI，您可以从[`docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI.html`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI.html)获取最新列表。

+   `instance_type`：用于构建 AMI 的实例类型。

+   `ssh_username`：Packer 在尝试连接到 Packer 构建过程中创建的临时 EC2 实例时应使用的 SSH 用户名。对于基于 Amazon Linux 的 AMI（例如 ECS-Optimized AMI），必须将其指定为`ec2-user`用户。

+   `associate_public_ip_address`：当设置为 true 时，将公共 IP 地址与实例关联。如果您在互联网上使用 Packer 并且没有对 Packer 构建过程中创建的临时 EC2 实例的私有网络访问权限，则需要此选项。

+   `ami_name`：将要创建的 AMI 的名称。此名称必须是唯一的，确保唯一性的常见方法是使用`{{timestamp}}` Go 模板函数，Packer 将自动将其替换为当前时间戳。

+   `tags`：要添加到创建的 AMI 中的标记列表。这允许您附加元数据，例如图像的源 AMI、ECS 代理版本、Docker 版本或任何其他您可能发现有用的信息。请注意，您可以引用一个名为`SourceAMI`的特殊模板变量，该变量由 Amazon EBS 构建器添加，并基于`source_ami`变量的值。

需要注意的一点是，与其在模板中硬编码您的 AWS 凭据，不如引用一个名为`{{user `<variable-name>`}}`的 Go 模板函数，这将注入在我们即将配置的顶级变量参数中定义的用户变量。

Packer 模板使用 Go 的模板语言进行处理，您可以在[`golang.org/pkg/text/template/`](https://golang.org/pkg/text/template/)上了解更多信息。Go 模板允许您定义自己的模板函数，Packer 包含了一些有用的函数，这些函数在[`www.packer.io/docs/templates/engine.html`](https://www.packer.io/docs/templates/engine.html)中定义。模板函数通过模板表达式调用，表达式采用句柄样式格式：`{{<function> <parameters>}}`。

# 配置变量

变量用于在构建时将用户特定或环境特定的设置注入到模板中，这对于使您的机器映像模板更通用并避免在模板中硬编码凭据非常有用。

在前面的示例中，您在定义 AWS 凭据设置时引用了用户变量，这些变量必须在 Packer 模板的变量部分中定义，就像在前面的示例中演示的那样：

[PRE2]

定义变量

在上面的示例中，请注意您在构建器部分的用户函数中定义了 AWS 凭据设置的每个变量。例如，构建器部分将`access_key`设置定义为`{{user `aws_access_key_id`}}`，它依次引用了变量部分中定义的`aws_access_key_id`变量。

每个变量依次引用`env`模板函数，该函数查找传递给此函数的环境变量的值。这意味着您可以控制每个变量的值如下：

+   `aws_access_key_id`：使用`AWS_ACCESS_KEY_ID`环境变量进行配置

+   `aws_secret_access_key`：使用`AWS_SECRET_ACCESS_KEY`环境变量进行配置

+   `aws_session_token`：使用`AWS_SESSION_TOKEN`环境变量进行配置

+   `timezone`：使用默认值**US/Eastern**进行配置。在运行`packer build`命令时，您可以通过设置`-var '<variable>=<value>'`标志（例如，`-var 'timezone=US/Pacific'`）来覆盖默认变量。

请注意，我们还没有在 Packer 模板中定义`timezone`变量，因为您将在本章后面使用这个变量。

# 配置配置程序

配置程序是 Packer 模板的核心，形成在自定义和构建机器镜像时执行的各种内部配置操作。

Packer 支持许多不同类型的配置程序，包括流行的配置管理工具，如 Ansible 和 Puppet，您可以在[`www.packer.io/docs/provisioners/index.html`](https://www.packer.io/docs/provisioners/index.html)上阅读更多关于不同类型的配置程序的信息。

对于我们的机器镜像，我们只会使用两种最基本和基本的配置程序：

+   [Shell 配置程序](https://www.packer.io/docs/provisioners/shell.html)：使用 shell 命令和脚本执行机器镜像的配置

+   [文件配置程序](https://www.packer.io/docs/provisioners/file.html)：将文件复制到机器镜像中

作为配置程序的介绍，让我们定义一个简单的 shell 配置程序，更新已安装的操作系统软件包，如下例所示：

[PRE3]

定义内联 shell 配置程序

在上面的例子中定义的配置程序使用`inline`参数来定义在配置阶段将执行的命令列表。在这种情况下，您正在运行`yum update`命令，这是 Amazon Linux 系统上的默认软件包管理器，并更新所有安装的操作系统软件包。为了确保您使用基本 ECS-Optimized AMI 中包含的 Docker 和 ECS 代理软件包的推荐和经过测试的版本，您使用`-x`标志来排除以`docker`和`ecs`开头的软件包。

在上面的例子中，yum 命令将被执行为`sudo yum -y -x docker\* -x ecs\* update`。因为反斜杠字符（`\`）在 JSON 中被用作转义字符，在上面的例子中，双反斜杠（例如，`\\*`）用于生成一个字面上的反斜杠。

最后，请注意，您必须使用`sudo`命令运行所有 shell 配置命令，因为 Packer 正在以构建器部分中定义的`ec2_user`用户身份配置 EC2 实例。

# 配置后处理器

我们将介绍 Packer 模板的最终结构组件是[后处理器](https://www.packer.io/docs/post-processors/index.html)，它允许您在机器镜像被配置和构建后执行操作。

后处理器可以用于各种不同的用例，超出了本书的范围，但我喜欢使用的一个简单的后处理器示例是[清单后处理器](https://www.packer.io/docs/post-processors/manifest.html)，它输出一个列出 Packer 生成的所有构件的 JSON 文件。当您创建首先构建 Packer 镜像，然后需要测试和部署镜像的持续交付流水线时，此输出非常有用。

在这种情况下，清单文件可以作为 Packer 构建的输出构件，描述与新机器映像相关的区域和 AMI 标识符，并且可以作为一个示例用作 CloudFormation 模板的输入，该模板将您的新机器映像部署到测试环境中。

以下示例演示了如何向您的 Packer 模板添加清单后处理器：

[PRE4]

定义清单后处理器

正如您在前面的示例中所看到的，清单后处理器非常简单 - `output`参数指定清单将被写入本地的文件名，而`strip_path`参数会剥离任何构建构件的本地文件系统路径信息。

# 构建机器映像

在这一点上，您已经创建了一个简单的 Packer 镜像，它在定制方面并不太多，但仍然是一个完整的模板，可以立即构建。

在实际运行构建之前，您需要确保本地环境已正确配置以成功完成构建。回想一下，在上一个示例中，您为模板定义了引用环境变量的变量，这些环境变量配置了您的 AWS 凭据，这里的一个常见方法是将本地 AWS 访问密钥 ID 和秘密访问密钥设置为环境变量。

然而，在我们的用例中，我假设您正在使用早期章节介绍的最佳实践方法，因此您的模板配置为使用临时会话凭据，这可以通过`aws_session_token`输入变量来证明，需要在运行 Packer 构建之前动态生成并注入到您的本地环境中。

# 生成动态会话凭据

要生成临时会话凭据，假设您已经使用`AWS_PROFILE`环境变量配置了适当的配置文件，您可以运行`aws sts assume-role`命令来生成凭据：

[PRE5]

生成临时会话凭据

在上面的示例中，请注意您可以使用 bash 替换动态获取`role_arn`和`role_session_name`参数，使用`aws configure get <parameter>`命令从 AWS CLI 配置文件中获取，这些参数在生成临时会话凭据时是必需的输入。

上面示例的输出包括一个包含以下值的凭据对象，这些值与 Packer 模板中引用的环境变量相对应：

+   **AccessKeyId**：此值作为`AWS_ACCESS_KEY_ID`环境变量导出

+   **SecretAccessKey**：此值作为`AWS_SECRET_ACCESS_KEY`环境变量导出

+   **SessionToken**：此值作为`AWS_SESSION_TOKEN`环境变量导出

# 自动生成动态会话凭据

虽然您可以使用上面示例中演示的方法根据需要生成临时会话凭据，但这种方法会很快变得繁琐。有许多方法可以自动将生成的临时会话凭据注入到您的环境中，但考虑到本书使用 Make 作为自动化工具，以下示例演示了如何使用一个相当简单的 Makefile 来实现这一点：

[PRE6]

使用 Make 自动生成临时会话凭据确保您的 Makefile 中的所有缩进都是使用制表符而不是空格。

在上面的示例中，请注意引入了一个名为`.ONESHELL`的指令。此指令配置 Make 在给定的 Make 配方中为所有定义的命令生成单个 shell，这意味着 bash 变量赋值和环境设置可以在多行中重复使用。

如果当前环境配置了`AWS_PROFILE`，`build`任务有条件地调用名为`assume_role`的函数，这种方法很有用，因为这意味着如果您在配置为以不同方式获取 AWS 凭据的构建代理上运行此 Makefile，临时会话凭据的动态生成将不会发生。

在 Makefile 中，如果命令以`@`符号为前缀，则执行的命令将不会输出到 stdout，而只会显示命令的输出。

`assume_role`函数使用高级的 JMESPath 查询表达式（如`--query`标志所指定的）来生成一组`export`语句，这些语句引用了在前面示例中运行的命令的**Credentials**字典输出上的各种属性，并使用 JMESPath join 函数将值分配给相关的环境变量。这些语句被包裹在一个命令替换中，使用`eval`命令来执行每个输出的`export`语句。如果你不太理解这个查询，不要担心，但要认识到 AWS CLI 确实包含一个强大的查询语法，可以创建一些相当复杂的一行命令。

在上面的示例中，注意你可以使用反引号（[PRE7]

> export AWS_PROFILE=docker-in-aws
> 
> 进行构建

输入 arn:aws:iam::385605022855:mfa/justin.menga 的 MFA 代码：******

packer build packer.json

亚马逊-ebs 输出将以这种颜色显示。

==> 亚马逊-ebs：预验证 AMI 名称：docker-in-aws-ecs 1518934269

亚马逊-ebs：找到镜像 ID：ami-5e414e24

==> 亚马逊-ebs：创建临时密钥对：packer_5a8918fd-018d-964f-4ab3-58bff320ead5

==> 亚马逊-ebs：为该实例创建临时安全组：packer_5a891904-2c84-aca1-d368-8309f215597d

==> 亚马逊-ebs：授权临时安全组从 0.0.0.0/0 访问端口 22...

==> 亚马逊-ebs：启动源 AWS 实例...

==> 亚马逊-ebs：向源实例添加标签

亚马逊-ebs：添加标签："Name": "Packer Builder"

亚马逊-ebs：实例 ID：i-04c150456ac0748aa

==> 亚马逊-ebs：等待实例(i-04c150456ac0748aa)就绪...

==> 亚马逊-ebs：等待 SSH 可用...

==> 亚马逊-ebs：连接到 SSH！

==> 亚马逊-ebs：使用 shell 脚本进行配置：/var/folders/s4/1mblw7cd29s8xc74vr3jdmfr0000gn/T/packer-shell190211980

亚马逊-ebs：已加载插件：priorities, update-motd, upgrade-helper

亚马逊-ebs：解决依赖关系

亚马逊-ebs：--> 运行事务检查

亚马逊-ebs：---> 包 elfutils-libelf.x86_64 0:0.163-3.18.amzn1 将会被更新

亚马逊-ebs：---> 包 elfutils-libelf.x86_64 0:0.168-8.19.amzn1 将会被更新

亚马逊-ebs：---> 包 python27.x86_64 0:2.7.12-2.121.amzn1 将会被更新

亚马逊-ebs：---> 包 python27.x86_64 0:2.7.13-2.122.amzn1 将会被更新

亚马逊-ebs：---> 包 python27-libs.x86_64 0:2.7.12-2.121.amzn1 将会被更新

亚马逊 EBS：---> 软件包 python27-libs.x86_64 0:2.7.13-2.122.amzn1 将被更新

亚马逊 EBS：--> 完成依赖关系解析

亚马逊 EBS：

亚马逊 EBS：依赖关系已解决

亚马逊 EBS：

亚马逊 EBS：===============================================================================

亚马逊 EBS：软件包 架构 版本 资料库 大小

亚马逊 EBS：===============================================================================

亚马逊 EBS：正在更新：

亚马逊 EBS：elfutils-libelf x86_64 0.168-8.19.amzn1 amzn-updates 313 k

亚马逊 EBS：python27 x86_64 2.7.13-2.122.amzn1 amzn-updates 103 k

亚马逊 EBS：python27-libs x86_64 2.7.13-2.122.amzn1 amzn-updates 6.8 M

亚马逊 EBS：

亚马逊 EBS：事务摘要

亚马逊 EBS：===============================================================================

亚马逊 EBS：升级 3 个软件包

亚马逊 EBS：

亚马逊 EBS：总下载大小：7.2 M

亚马逊 EBS：下载软件包：

亚马逊 EBS：--------------------------------------------------------------------------------

亚马逊 EBS：总共 5.3 MB/s | 7.2 MB 00:01

亚马逊 EBS：运行事务检查

亚马逊 EBS：运行事务测试

亚马逊 EBS：事务测试成功

亚马逊 EBS：运行事务

亚马逊 EBS：正在更新：python27-2.7.13-2.122.amzn1.x86_64 1/6

亚马逊 EBS：正在更新：python27-libs-2.7.13-2.122.amzn1.x86_64 2/6

亚马逊 EBS：正在更新：elfutils-libelf-0.168-8.19.amzn1.x86_64 3/6

亚马逊 EBS：清理：python27-2.7.12-2.121.amzn1.x86_64 4/6

亚马逊 EBS：清理：python27-libs-2.7.12-2.121.amzn1.x86_64 5/6

亚马逊 EBS：清理：elfutils-libelf-0.163-3.18.amzn1.x86_64 6/6

亚马逊 EBS：正在验证：python27-libs-2.7.13-2.122.amzn1.x86_64 1/6

亚马逊 EBS：正在验证：elfutils-libelf-0.168-8.19.amzn1.x86_64 2/6

亚马逊 EBS：正在验证：python27-2.7.13-2.122.amzn1.x86_64 3/6

亚马逊 EBS：正在验证：python27-libs-2.7.12-2.121.amzn1.x86_64 4/6

亚马逊 EBS：正在验证：elfutils-libelf-0.163-3.18.amzn1.x86_64 5/6

亚马逊 EBS：正在验证：python27-2.7.12-2.121.amzn1.x86_64 6/6

亚马逊 EBS：

亚马逊 EBS：已更新：

亚马逊 EBS：elfutils-libelf.x86_64 0:0.168-8.19.amzn1

亚马逊 EBS：python27.x86_64 0:2.7.13-2.122.amzn1

亚马逊 EBS：python27-libs.x86_64 0:2.7.13-2.122.amzn1

亚马逊 EBS：

亚马逊 EBS：完成！

==> 亚马逊 EBS：停止源实例...

亚马逊 EBS：停止实例，尝试 1

==> 亚马逊 EBS：等待实例停止...

==> 亚马逊 EBS：创建 AMI：docker-in-aws-ecs 1518934269

亚马逊 EBS：AMI：ami-57415b2d

==> 亚马逊 EBS：等待 AMI 准备就绪...

==> 亚马逊-ebs：向 AMI（ami-57415b2d）添加标记...

==> 亚马逊-ebs：给快照打标记：snap-0bc767fd982333bf8

==> 亚马逊-ebs：给快照打标记：snap-0104c1a352695c1e9

==> 亚马逊-ebs：创建 AMI 标记

亚马逊-ebs：添加标记："SourceAMI": "ami-5e414e24"

亚马逊-ebs：添加标记："DockerVersion": "17.09.1-ce"

亚马逊-ebs：添加标记："ECSAgentVersion": "1.17.0-2"

亚马逊-ebs：添加标记："Name": "Docker in AWS ECS Base Image 2017.09.h"

==> 亚马逊-ebs：创建快照标记

==> 亚马逊-ebs：终止源 AWS 实例...

==> 亚马逊-ebs：清理任何额外的卷...

==> 亚马逊-ebs：没有要清理的卷，跳过

==> 亚马逊-ebs：删除临时安全组...

==> 亚马逊-ebs：删除临时密钥对...

==> 亚马逊-ebs：运行后处理器：manifest

构建'亚马逊-ebs'完成。

==> 构建完成。成功构建的工件是：

--> 亚马逊-ebs：AMI 已创建：

us-east-1: ami-57415b2d

[PRE8]

> cat manifest.json

{

"builds": [

{

"name": "amazon-ebs",

"builder_type": "amazon-ebs",

"build_time": 1518934504,

"files": null,

"artifact_id": "us-east-1:ami-57415b2d",

"packer_run_uuid": "db07ccb3-4100-1cc8-f0be-354b9f9b021d"

}

],

"last_run_uuid": "db07ccb3-4100-1cc8-f0be-354b9f9b021d"

}

> echo manifest.json >> .gitignore

[PRE9]

{

"variables": {...},

"builders": [

{

"type": "amazon-ebs",

"access_key": "{{user `aws_access_key_id`}}",

"secret_key": "{{user `aws_secret_access_key`}}",

"token": "{{user `aws_session_token`}}",

"region": "us-east-1",

"source_ami": "ami-5e414e24",

"instance_type": "t2.micro",

"ssh_username": "ec2-user",

"associate_public_ip_address": "true",

"ami_name": "docker-in-aws-ecs {{timestamp}}",

"launch_block_device_mappings": [

{

"device_name": "/dev/xvdcy",

"volume_size": 20,

"volume_type": "gp2",

"delete_on_termination": true

}

],

"tags": {

"Name": "Docker in AWS ECS Base Image 2017.09.h",

"SourceAMI": "ami-5e414e24",

"DockerVersion": "17.09.1-ce",

"ECSAgentVersion": "1.17.0-2"

}

}

],

"provisioners": [...],

"post-processors": [...]

}

[PRE10]

{

"variables": {...},

"builders": [...],

"provisioners": [

{

"type": "shell",

"script": "scripts/storage.sh"

},

{

"type": "shell",

"inline": [

"sudo yum -y -x docker\\* -x ecs\\* update"

]

}

],

"post-processors": [...]

}

[PRE11]

> mkdir -p scripts
> 
> touch scripts/storage.sh
> 
> tree

.

├── Makefile

├── manifest.json

├── packer.json

└── 脚本

└── storage.sh

1 个目录，4 个文件

[PRE12]

#!/usr/bin/env bash

set -e

echo "### 配置 Docker 卷存储 ###"

sudo mkdir -p /data

sudo mkfs.ext4 -L docker /dev/xvdcy

echo -e "LABEL=docker\t/data\t\text4\tdefaults,noatime\t0\t0" | sudo tee -a /etc/fstab

sudo mount -a

[PRE13]

{

"variables": {...},

"builders": [...],

"provisioners": [

{

"type": "shell",

"script": "scripts/storage.sh"

},

{

"type": "shell",

"inline": [

"sudo yum -y -x docker\\* -x ecs\\* update",

"sudo yum -y install aws-cfn-bootstrap awslogs jq"

]

}

],

"post-processors": [...]

}

[PRE14]

{

"variables": {...},

"builders": [...],

"provisioners": [

{

"type": "shell",

"script": "scripts/storage.sh"

},

{

"type": "shell",

"script": "scripts/time.sh",

"environment_vars": [

"TIMEZONE={{user `timezone`}}"

]

},

{

"type": "shell",

"inline": [

"sudo yum -y -x docker\\* -x ecs\\* update",

"sudo yum -y install aws-cfn-bootstrap awslogs jq"

]

}

],

"post-processors": [...]

}

[PRE15]

#!/usr/bin/env bash

set -e

# Configure host to use timezone

# http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-time.html

echo "### 设置时区为 $TIMEZONE ###"

sudo tee /etc/sysconfig/clock << EOF > /dev/null

ZONE="$TIMEZONE"

UTC=true

EOF

sudo ln -sf /usr/share/zoneinfo/"$TIMEZONE" /etc/localtime

# 使用 AWS NTP 同步服务

echo "server 169.254.169.123 prefer iburst" | sudo tee -a /etc/ntp.conf

# 启用 NTP

sudo chkconfig ntpd on

[PRE16]

{

"variables": {...},

"builders": [...],

"provisioners": [

{

"type": "shell",

"script": "scripts/storage.sh"

},

{

"type": "shell",

"script": "scripts/time.sh",

"environment_vars": [

"TIMEZONE={{user `timezone`}}"

],

},

{

"type": "shell",

"script": "scripts/cloudinit.sh"

},

{

"type": "shell",

"inline": [

"sudo yum -y -x docker\\* -x ecs\\* update",

"sudo yum -y install aws-cfn-bootstrap awslogs jq"

]

}

],

"post-processors": [...]

}

[PRE17]

#!/usr/bin/env bash

set -e

# 禁用 cloud-init 仓库更新或升级

sudo sed -i -e '/^repo_update: /{h;s/: .*/: false/};${x;/^$/{s//repo_update: false/;H};x}' /etc/cloud/cloud.cfg

sudo sed -i -e '/^repo_upgrade: /{h;s/: .*/: none/};${x;/^$/{s//repo_upgrade: none/;H};x}' /etc/cloud/cloud.cfg

[PRE18]

{

"variables": {...},

"builders": [...],

"provisioners": [

{

"type": "shell",

"script": "scripts/storage.sh"

},

{

"type": "shell",

"script": "scripts/time.sh",

"environment_vars": [

"TIMEZONE={{user `timezone`}}"

]

},

{

"type": "shell",

"script": "scripts/cloudinit.sh"

},

{

"type": "shell",

"inline": [

"sudo yum -y -x docker\\* -x ecs\\* update",

"sudo yum -y install aws-cfn-bootstrap awslogs jq"

]

},

{ "type": "shell",

"script": "scripts/cleanup.sh"

}

],

"post-processors": [...]

}

[PRE19]

#!/usr/bin/env bash

echo "### 执行最终清理任务 ###"

sudo stop ecs

sudo docker system prune -f -a

sudo service docker stop

sudo chkconfig docker off

sudo rm -rf /var/log/docker /var/log/ecs/*

[PRE20]

{

"variables": {...},

"builders": [...],

"provisioners": [

{

"type": "shell",

"script": "scripts/storage.sh"

},

{

"type": "shell",

"script": "scripts/time.sh",

"environment_vars": [

"TIMEZONE={{user `timezone`}}"

]

},

{

"type": "shell",

"script": "scripts/cloudinit.sh"

},

{

"type": "shell",

"inline": [

"sudo yum -y -x docker\\* -x ecs\\* update",

"sudo yum -y install aws-cfn-bootstrap awslogs jq"

]

},

{

"type": "shell",

"script": "scripts/cleanup.sh"

},

{

"type": "file",

"source": "files/firstrun.sh",

"destination": "/home/ec2-user/firstrun.sh"

}

],

"post-processors": [...]

}

[PRE21]

#!/usr/bin/env bash

set -e

# 配置 ECS 代理

echo "ECS_CLUSTER=${ECS_CLUSTER}" > /etc/ecs/ecs.config

[PRE22]

#!/usr/bin/env bash

set -e

# 配置 ECS 代理

echo "ECS_CLUSTER=${ECS_CLUSTER}" > /etc/ecs/ecs.config

# 如果提供了 HTTP 代理 URL，则设置

if [ -n $PROXY_URL ]

then

echo export HTTPS_PROXY=$PROXY_URL >> /etc/sysconfig/docker

echo HTTPS_PROXY=$PROXY_URL >> /etc/ecs/ecs.config

echo NO_PROXY=169.254.169.254,169.254.170.2,/var/run/docker.sock >> /etc/ecs/ecs.config

echo HTTP_PROXY=$PROXY_URL >> /etc/awslogs/proxy.conf

echo HTTPS_PROXY=$PROXY_URL >> /etc/awslogs/proxy.conf

echo NO_PROXY=169.254.169.254 >> /etc/awslogs/proxy.conf

fi

[PRE23]

#!/usr/bin/env bash

set -e

# 配置 ECS 代理

echo "ECS_CLUSTER=${ECS_CLUSTER}" > /etc/ecs/ecs.config

# 如果提供了 HTTP 代理 URL

if [ -n $PROXY_URL ]

then

echo export HTTPS_PROXY=$PROXY_URL >> /etc/sysconfig/docker

echo HTTPS_PROXY=$PROXY_URL >> /etc/ecs/ecs.config

echo NO_PROXY=169.254.169.254,169.254.170.2,/var/run/docker.sock >> /etc/ecs/ecs.config

echo HTTP_PROXY=$PROXY_URL >> /etc/awslogs/proxy.conf

echo HTTPS_PROXY=$PROXY_URL >> /etc/awslogs/proxy.conf

echo NO_PROXY=169.254.169.254 >> /etc/awslogs/proxy.conf

fi

# 写入 AWS 日志区域

使用 sudo tee /etc/awslogs/awscli.conf << EOF > /dev/null 命令

[plugins]

cwlogs = cwlogs

[default]

region = ${AWS_DEFAULT_REGION}

EOF

# 写入 AWS 日志配置

使用 sudo tee /etc/awslogs/awslogs.conf << EOF > /dev/null 命令

[general]

state_file = /var/lib/awslogs/agent-state

[/var/log/dmesg]

file = /var/log/dmesg

log_group_name = /${STACK_NAME}/ec2/${AUTOSCALING_GROUP}/var/log/dmesg

log_stream_name = {instance_id}

[/var/log/messages]

file = /var/log/messages

log_group_name = /${STACK_NAME}/ec2/${AUTOSCALING_GROUP}/var/log/messages

log_stream_name = {instance_id}

datetime_format = %b %d %H:%M:%S

[/var/log/docker]

文件= /var/log/docker

log_group_name = /${STACK_NAME}/ec2/${AUTOSCALING_GROUP}/var/log/docker

log_stream_name = {instance_id}

datetime_format = %Y-%m-%dT%H:%M:%S.%f

[/var/log/ecs/ecs-init.log]

文件= /var/log/ecs/ecs-init.log*

log_group_name = /${STACK_NAME}/ec2/${AUTOSCALING_GROUP}/var/log/ecs/ecs-init

log_stream_name = {instance_id}

datetime_format = %Y-%m-%dT%H:%M:%SZ

time_zone = UTC

[/var/log/ecs/ecs-agent.log]

文件= /var/log/ecs/ecs-agent.log*

log_group_name = /${STACK_NAME}/ec2/${AUTOSCALING_GROUP}/var/log/ecs/ecs-agent

log_stream_name = {instance_id}

datetime_format = %Y-%m-%dT%H:%M:%SZ

time_zone = UTC

[/var/log/ecs/audit.log]

文件= /var/log/ecs/audit.log*

log_group_name = /${STACK_NAME}/ec2/${AUTOSCALING_GROUP}/var/log/ecs/audit.log

log_stream_name = {instance_id}

datetime_format = %Y-%m-%dT%H:%M:%SZ

time_zone = UTC

EOF

[PRE24]

#!/usr/bin/env bash

set -e

# 配置 ECS 代理

echo "ECS_CLUSTER=${ECS_CLUSTER}" > /etc/ecs/ecs.config

# 如果提供了 HTTP 代理 URL，则设置

...

...

# 编写 AWS 日志区域

...

...

# 编写 AWS 日志配置

...

...

# 启动服务

sudo service awslogs start

sudo chkconfig docker on

sudo service docker start

sudo start ecs

[PRE25]

#!/usr/bin/env bash

set -e

# 配置 ECS 代理

echo "ECS_CLUSTER=${ECS_CLUSTER}" > /etc/ecs/ecs.config

# 如果提供了 HTTP 代理 URL，则设置

...

...

# 编写 AWS 日志区域

...

...

# 编写 AWS 日志配置

...

...

# 启动服务

...

...

# 健康检查

# 循环直到 ECS 代理注册到 ECS 集群

echo "检查 ECS 代理是否加入到${ECS_CLUSTER}"

直到[[ "$(curl --fail --silent http://localhost:51678/v1/metadata | jq '.Cluster // empty' -r -e)" == ${ECS_CLUSTER} ]]

do printf '.'

睡眠 5

完成

echo "ECS 代理成功加入到${ECS_CLUSTER}"

[PRE26]

> 树

。

├── Makefile

├── 文件

│   └── firstrun.sh

├── manifest.json

├── packer.json

└── 脚本

├── cleanup.sh

├── cloudinit.sh

├── storage.sh

└── time.sh

2 个目录，8 个文件

[PRE27]

> sudo mount

proc 在/proc 上的类型为 proc（rw，relatime）

sysfs 在/sys 上的类型为 sysfs（rw，relatime）

/dev/xvda1 在/上的类型为 ext4（rw，noatime，data=ordered）

devtmpfs 在/dev 上的类型为 devtmpfs（rw，relatime，size=500292k，nr_inodes=125073，mode=755）

devpts 在/dev/pts 上的类型为 devpts（rw，relatime，gid=5，mode=620，ptmxmode=000）

tmpfs 在/dev/shm 上的类型为 tmpfs（rw，relatime）

/dev/xvdcy 在/data 上的类型为 ext4（rw，noatime，data=ordered）

none 在/proc/sys/fs/binfmt_misc 上的类型为 binfmt_misc（rw，relatime）

[PRE28]

> 日期

Wed Feb 21 06:45:40 EST 2018

> sudo service ntpd status

ntpd 正在运行

[PRE29]

> cat /etc/cloud/cloud.cfg

# 警告：此文件的修改可能会被文件中的内容覆盖

# /etc/cloud/cloud.cfg.d

# 如果设置了这个，'root'将无法通过 ssh 登录，他们

# 将收到一条消息，以默认用户（ec2-user）身份登录

disable_root: true

# 这将导致 set+update 主机名模块不起作用（如果为 true）

preserve_hostname: true

datasource_list: [ Ec2, None ]

repo_upgrade: none

repo_upgrade_exclude:

- 内核

- nvidia*

- cudatoolkit

mounts:

- [ 临时 0，/media/ephemeral0 ]

- [ 交换，无，交换，sw，"0"，"0" ]

# vim:syntax=yaml

repo_update: false

[PRE30]

> sudo service docker status

docker 已停止

> sudo chkconfig --list docker

docker 0:off 1:off 2:off 3:off 4:off 5:off 6:off

[PRE31]

> pwd

/home/ec2-user

> ls

firstrun.sh

```

验证首次运行脚本

此时，您已成功验证了您的 ECS 容器实例已根据您的定制构建，并且现在应该从 EC2 控制台终止该实例。您会注意到它处于未配置状态，实际上您的 ECS 容器实例无法做太多事情，因为 Docker 服务已被禁用，在下一章中，您将学习如何使用 CloudFormation 利用您安装到自定义机器镜像中的 CloudFormation 辅助脚本来配置您的 ECS 容器实例在实例创建时，并利用您创建的定制。

# 摘要

在本章中，您学习了如何使用流行的开源工具 Packer 构建自定义的 ECS 容器实例机器镜像。您学习了如何创建 Packer 模板，并了解了组成模板的各个部分，包括变量、构建器、供应商和后处理器。您能够在图像构建过程中注入临时会话凭据，以便验证访问 AWS，使用 Packer 变量、环境变量和一些 Make 自动化的组合。

您已成功将一些构建时定制引入到您的 ECS 容器实例镜像中，包括安装 CloudFormation 辅助脚本和 CloudWatch 日志代理，并确保系统配置为在启动时以正确的时区运行 NTP 服务。您在 cloud-init 配置中禁用了自动安全更新，这可能会在使用 HTTP 代理时造成问题。

最后，您创建了一个首次运行脚本，旨在在实例创建和首次启动时配置您的 ECS 容器实例。该脚本配置 ECS 集群成员资格，启用可选的 HTTP 代理支持，为 Docker 和 ECS 代理系统日志配置 CloudWatch 日志代理，并执行健康检查，以确保您的实例已成功初始化。

在下一章中，您将学习如何使用自定义 AMI 来构建 ECS 集群和相关的底层 EC2 自动扩展组，这将帮助您理解对自定义机器映像执行的各种自定义的原因。

# 问题

1.  Packer 模板的哪个部分定义了 Packer 构建过程中使用的临时实例的 EC2 实例类型？

1.  True/False: Packer 在构建过程中需要对临时实例进行 SSH 访问。

1.  您使用什么配置文件格式来定义 Packer 模板？

1.  True/False: 您必须将 AWS 凭据硬编码到 Packer 模板中。

1.  True/False: 要捕获 Packer 创建的 AMI ID，您必须解析 Packer 构建过程的日志输出。

1.  ECS-Optimized AMI 的默认存储配置是什么？

1.  您会使用什么类型的 Packer provisioner 来将文件写入/etc 目录？

1.  您从一个需要很长时间才能启动的自定义 AMI 创建了一个 EC2 实例。该 AMI 安装在一个没有额外基础架构配置的私有子网中。导致启动时间缓慢的可能原因是什么？

# 进一步阅读

您可以查看以下链接以获取本章涵盖的主题的更多信息：

+   Packer Amazon EBS Builder documentation: [`www.packer.io/docs/builders/amazon-ebs.html`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html), [`docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html)

+   Amazon ECS-Optimized AMI: [`docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI.html`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI.html)

+   使用 CloudWatch 日志入门：[`docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_GettingStarted.html`](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_GettingStarted.html)

+   CloudFormation 辅助脚本参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-helper-scripts-reference.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-helper-scripts-reference.html)

+   使用 ECS CLI：[`docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI.html`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI.html)

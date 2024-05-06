# 构建自定义 ECS 容器实例

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

```
{
    "variables": {},
    "builders": [],
    "provisioners": [],
    "post-processors": []
}
```

Packer 模板结构

# 配置构建器

让我们开始配置我们的 Packer 模板，首先在 packer-ecs 存储库的根目录下创建一个名为`packer.json`的文件，然后定义构建器部分，如下例所示：

```
{
  "variables": {},
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
 "tags": {
 "Name": "Docker in AWS ECS Base Image 2017.09.h",
 "SourceAMI": "{{ .SourceAMI }}",
 "DockerVersion": "17.09.1-ce",
 "ECSAgentVersion": "1.17.0-2"
 }
 }
 ],
  "provisioners": [],
  "post-processors": []
}
```

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

```
{
  "variables": {
 "aws_access_key_id": "{{env `AWS_ACCESS_KEY_ID`}}",
 "aws_secret_access_key": "{{env `AWS_SECRET_ACCESS_KEY`}}",
 "aws_session_token": "{{env `AWS_SESSION_TOKEN`}}",
 "timezone": "US/Eastern"
 },
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
      "tags": {
        "Name": "Docker in AWS ECS Base Image 2017.09.h",
        "SourceAMI": "{{ .SourceAMI }}",
        "DockerVersion": "17.09.1-ce",
        "ECSAgentVersion": "1.17.0-2"
      }
    }
  ],
  "provisioners": [],
  "post-processors": []
}
```

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

```
{
  "variables": {
    "aws_access_key_id": "{{env `AWS_ACCESS_KEY_ID`}}",
    "aws_secret_access_key": "{{env `AWS_SECRET_ACCESS_KEY`}}",
    "aws_session_token": "{{env `AWS_SESSION_TOKEN`}}",
    "timezone": "US/Eastern"
  },
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
      "tags": {
        "Name": "Docker in AWS ECS Base Image 2017.09.h",
        "SourceAMI": "ami-5e414e24",
        "DockerVersion": "17.09.1-ce",
        "ECSAgentVersion": "1.17.0-2"
      }
    }
  ],
  "provisioners": [
 {
 "type": "shell",
 "inline": [
 "sudo yum -y -x docker\\* -x ecs\\* update"
 ] 
 }
 ],
  "post-processors": []
}
```

定义内联 shell 配置程序

在上面的例子中定义的配置程序使用`inline`参数来定义在配置阶段将执行的命令列表。在这种情况下，您正在运行`yum update`命令，这是 Amazon Linux 系统上的默认软件包管理器，并更新所有安装的操作系统软件包。为了确保您使用基本 ECS-Optimized AMI 中包含的 Docker 和 ECS 代理软件包的推荐和经过测试的版本，您使用`-x`标志来排除以`docker`和`ecs`开头的软件包。

在上面的例子中，yum 命令将被执行为`sudo yum -y -x docker\* -x ecs\* update`。因为反斜杠字符（`\`）在 JSON 中被用作转义字符，在上面的例子中，双反斜杠（例如，`\\*`）用于生成一个字面上的反斜杠。

最后，请注意，您必须使用`sudo`命令运行所有 shell 配置命令，因为 Packer 正在以构建器部分中定义的`ec2_user`用户身份配置 EC2 实例。

# 配置后处理器

我们将介绍 Packer 模板的最终结构组件是[后处理器](https://www.packer.io/docs/post-processors/index.html)，它允许您在机器镜像被配置和构建后执行操作。

后处理器可以用于各种不同的用例，超出了本书的范围，但我喜欢使用的一个简单的后处理器示例是[清单后处理器](https://www.packer.io/docs/post-processors/manifest.html)，它输出一个列出 Packer 生成的所有构件的 JSON 文件。当您创建首先构建 Packer 镜像，然后需要测试和部署镜像的持续交付流水线时，此输出非常有用。

在这种情况下，清单文件可以作为 Packer 构建的输出构件，描述与新机器映像相关的区域和 AMI 标识符，并且可以作为一个示例用作 CloudFormation 模板的输入，该模板将您的新机器映像部署到测试环境中。

以下示例演示了如何向您的 Packer 模板添加清单后处理器：

```
{
  "variables": {
    "aws_access_key_id": "{{env `AWS_ACCESS_KEY_ID`}}",
    "aws_secret_access_key": "{{env `AWS_SECRET_ACCESS_KEY`}}",
    "aws_session_token": "{{env `AWS_SESSION_TOKEN`}}",
    "timezone": "US/Eastern"
  },
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
      "tags": {
        "Name": "Docker in AWS ECS Base Image 2017.09.h",
        "SourceAMI": "ami-5e414e24",
        "DockerVersion": "17.09.1-ce",
        "ECSAgentVersion": "1.17.0-2"
      }
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "inline": [
        "sudo yum -y -x docker\\* -x ecs\\* update"
      ] 
    }
  ],
  "post-processors": [
 {
 "type": "manifest",
 "output": "manifest.json",
 "strip_path": true
 }
 ]
}
```

定义清单后处理器

正如您在前面的示例中所看到的，清单后处理器非常简单 - `output`参数指定清单将被写入本地的文件名，而`strip_path`参数会剥离任何构建构件的本地文件系统路径信息。

# 构建机器映像

在这一点上，您已经创建了一个简单的 Packer 镜像，它在定制方面并不太多，但仍然是一个完整的模板，可以立即构建。

在实际运行构建之前，您需要确保本地环境已正确配置以成功完成构建。回想一下，在上一个示例中，您为模板定义了引用环境变量的变量，这些环境变量配置了您的 AWS 凭据，这里的一个常见方法是将本地 AWS 访问密钥 ID 和秘密访问密钥设置为环境变量。

然而，在我们的用例中，我假设您正在使用早期章节介绍的最佳实践方法，因此您的模板配置为使用临时会话凭据，这可以通过`aws_session_token`输入变量来证明，需要在运行 Packer 构建之前动态生成并注入到您的本地环境中。

# 生成动态会话凭据

要生成临时会话凭据，假设您已经使用`AWS_PROFILE`环境变量配置了适当的配置文件，您可以运行`aws sts assume-role`命令来生成凭据：

```
> export AWS_PROFILE=docker-in-aws
> aws sts assume-role --role-arn=$(aws configure get role_arn) --role-session-name=$(aws configure get role_session_name)
Enter MFA code for arn:aws:iam::385605022855:mfa/justin.menga: ******
{
    "Credentials": {
        "AccessKeyId": "ASIAIIEUKCAR3NMIYM5Q",
        "SecretAccessKey": "JY7HmPMf/tPDXsgQXHt5zFZObgrQJRvNz7kb4KDM",
        "SessionToken": "FQoDYXdzEM7//////////wEaDP0PBiSeZvJ9GjTP5yLwAVjkJ9ZCMbSY5w1EClNDK2lS3nkhRg34/9xVgf9RmKiZnYVywrI9/tpMP8LaU/xH6nQvCsZaVTxGXNFyPz1BcsEGM6Z2ebIFX5rArT9FWu3v7WVs3QQvXeDTasgdvq71eFs2+qX7zbjK0YHXaWuu7GA/LGtNj4i+yi6EZ3OIq3hnz3+QY2dXL7O1pieMLjfZRf98KHucUhiokaq61cXSo+RJa3yuixaJMSxJVD1myx/XNritkawUfI8Xwp6g6KWYQAzDYz3MIWbA5LyX9Q0jk3yXTRAQOjLwvL8ZK/InJCDoPBFWFJwrz+Wxgep+I8iYoijOhqTUBQ==",
        "Expiration": "2018-02-18T05:38:38Z"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAJASB32NFHLLQHZ54S:justin.menga",
        "Arn": "arn:aws:sts::385605022855:assumed-role/admin/justin.menga"
    }
}
> export AWS_ACCESS_KEY_ID="ASIAIIEUKCAR3NMIYM5Q"
> export AWS_SECRET_ACCESS_KEY="JY7HmPMf/tPDXsgQXHt5zFZObgrQJRvNz7kb4KDM"
> export AWS_SESSION_TOKEN="FQoDYXdzEM7//////////wEaDP0PBiSeZvJ9GjTP5yLwAVjkJ9ZCMbSY5w1EClNDK2lS3nkhRg34/9xVgf9RmKiZnYVywrI9/tpMP8LaU/xH6nQvCsZaVTxGXNFyPz1BcsEGM6Z2ebIFX5rArT9FWu3v7WVs3QQvXeDTasgdvq71eFs2+qX7zbjK0YHXaWuu7GA/LGtNj4i+yi6EZ3OIq3hnz3+QY2dXL7O1pieMLjfZRf98KHucUhiokaq61cXSo+RJa3yuixaJMSxJVD1myx/XNritkawUfI8Xwp6g6KWYQAzDYz3MIWbA5LyX9Q0jk3yXTRAQOjLwvL8ZK/InJCDoPBFWFJwrz+Wxgep+I8iYoijOhqTUBQ=="
```

生成临时会话凭据

在上面的示例中，请注意您可以使用 bash 替换动态获取`role_arn`和`role_session_name`参数，使用`aws configure get <parameter>`命令从 AWS CLI 配置文件中获取，这些参数在生成临时会话凭据时是必需的输入。

上面示例的输出包括一个包含以下值的凭据对象，这些值与 Packer 模板中引用的环境变量相对应：

+   **AccessKeyId**：此值作为`AWS_ACCESS_KEY_ID`环境变量导出

+   **SecretAccessKey**：此值作为`AWS_SECRET_ACCESS_KEY`环境变量导出

+   **SessionToken**：此值作为`AWS_SESSION_TOKEN`环境变量导出

# 自动生成动态会话凭据

虽然您可以使用上面示例中演示的方法根据需要生成临时会话凭据，但这种方法会很快变得繁琐。有许多方法可以自动将生成的临时会话凭据注入到您的环境中，但考虑到本书使用 Make 作为自动化工具，以下示例演示了如何使用一个相当简单的 Makefile 来实现这一点：

```
.PHONY: build
.ONESHELL:

build:
  @ $(if $(AWS_PROFILE),$(call assume_role))
  packer build packer.json

# Dynamically assumes role and injects credentials into environment
define assume_role
  export AWS_DEFAULT_REGION=$$(aws configure get region)
  eval $$(aws sts assume-role --role-arn=$$(aws configure get role_arn) \
    --role-session-name=$$(aws configure get role_session_name) \
    --query "Credentials.[ \
        [join('=',['export AWS_ACCESS_KEY_ID',AccessKeyId])], \
        [join('=',['export AWS_SECRET_ACCESS_KEY',SecretAccessKey])], \
        [join('=',['export AWS_SESSION_TOKEN',SessionToken])] \
      ]" \
    --output text)
endef
```

使用 Make 自动生成临时会话凭据确保您的 Makefile 中的所有缩进都是使用制表符而不是空格。

在上面的示例中，请注意引入了一个名为`.ONESHELL`的指令。此指令配置 Make 在给定的 Make 配方中为所有定义的命令生成单个 shell，这意味着 bash 变量赋值和环境设置可以在多行中重复使用。

如果当前环境配置了`AWS_PROFILE`，`build`任务有条件地调用名为`assume_role`的函数，这种方法很有用，因为这意味着如果您在配置为以不同方式获取 AWS 凭据的构建代理上运行此 Makefile，临时会话凭据的动态生成将不会发生。

在 Makefile 中，如果命令以`@`符号为前缀，则执行的命令将不会输出到 stdout，而只会显示命令的输出。

`assume_role`函数使用高级的 JMESPath 查询表达式（如`--query`标志所指定的）来生成一组`export`语句，这些语句引用了在前面示例中运行的命令的**Credentials**字典输出上的各种属性，并使用 JMESPath join 函数将值分配给相关的环境变量。这些语句被包裹在一个命令替换中，使用`eval`命令来执行每个输出的`export`语句。如果你不太理解这个查询，不要担心，但要认识到 AWS CLI 确实包含一个强大的查询语法，可以创建一些相当复杂的一行命令。

在上面的示例中，注意你可以使用反引号（```) as an alternative syntax for bash command substitutions. In other words, `$(command)` and ``command`` both represent command substitutions that will execute the command and return the output.

# Building the image

Now that we have a mechanism of automating the generation of temporary session credentials, assuming that your `packer.json` file and Makefile are in the root of your packer-ecs repository, let's test out building your Packer image by running `make build`:

```

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

```

Running a Packer build

Referring back to the previous example and the output of the preceding one, notice in the `build` task that the command to build a Packer image is simply `packer build <template-file>`, which in this case is `packer build packer.json`.

If you review the output of the preceding example, you can see the following steps are performed by Packer:

*   Packer initially validates the source AMI and then generates a temporary SSH key pair and security group so that it is able to access the temporary EC2 instance.
*   Packer launches a temporary EC2 instance from the source AMI and then waits until it is able to establish SSH access.
*   Packer executes the provisioning actions as defined in the provisioners section of the template. In this case, you can see the output of the yum `update` command, which is our current single provisioning action.
*   Once complete, Packer stops the instance and creates a snapshot of the EBS volume instance, which produces an AMI with an appropriate name and ID.
*   With the AMI created, Packer terminates the instance, deletes the temporary SSH key pair and security group, and outputs the new AMI ID.

Recall in the earlier example, that you added a manifest post-processor to your template, and you should find a file called `manifest.json` has been output at the root of your repository, which you typically would not want to commit to your **packer-ecs** repository:

```

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

```

Viewing the Packer build manifest

# Building custom ECS container instance images using Packer

In the previous section, you established a base template for building a custom AMI using Packer, and proceed to build and publish your first custom AMI. At this point, you have not performed any customization that is specific to the use case of provisioning ECS container instances, so this section will focus on enhancing your Packer template to include such customizations.

The customizations you will learn about now include the following:

*   Defining a custom storage configuration
*   Installing additional packages and configuring operating system settings
*   Configuring a cleanup script
*   Creating a first-run script

With these customizations in place, we will complete the chapter by building your final custom ECS Container Instance AMI and launching an instance to verify the various customizations.

# Defining a custom storage configuration

The AWS ECS-Optimized AMI includes a default storage configuration that uses a 30 GB EBS volume, which is partitioned as follows:

*   `/dev/xvda`: An 8 GB volume that is mounted as the root filesystem and serves as the operating system partition.
*   `dev/xvdcz`: A 22 GB volume that is configured as a logical volume management (LVM) device and is used for Docker image and metadata storage.

The ECS-Optimized AMI uses the devicemapper storage driver for Docker image and metadata storage, which you can learn more about at [`docs.docker.com/v17.09/engine/userguide/storagedriver/device-mapper-driver/`](https://docs.docker.com/storage/storagedriver/device-mapper-driver/).

For most use cases, this storage configuration should be sufficient, however there are a couple of scenarios in which you may want to modify the default configuration:

*   **You need more Docker image and metadata storage**: This is easily addressed by simply configuring your ECS container instances with a larger volume size. The default storage configuration will always reserve 8GB for the operating system and root filesystem, with the remainder of the storage allocated for Docker image and metadata storage.
*   **You need to support Docker volumes that have large storage requirements**: By default, the ECS-Optimized AMI stores Docker volumes at `/var/lib/docker/volumes`, which is part of the root filesystem on the 8GB `/dev/xvda` partition. If you have larger volume requirements, this can cause your operating system partition to quickly become full, so in this scenario you should separate out the volume storage to a separate EBS volume.

Let's now see how you can modify your Packer template to add a new dedicated volume for Docker volume storage and ensure this volume is mounted correctly on instance creation.

# Adding EBS volumes

To add an EBS volume to your custom AMIs, you can configure the `launch_block_device_mappings` parameter within the Amazon EBS builder:

```

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

```

Adding a launch block device mapping

In the preceding example, I have truncated other portions of the Packer template for brevity, and you can see that we have added a single 20 GB volume called `/dev/xvdcy`, which is configured to be destroyed on instance termination. Notice that the `volume_type` parameter is set to `gp2`, which is the general-purpose SSD storage type that typically offers the best overall price/performance in AWS.

# Formatting and mounting volumes

With the configuration of the preceding example in place, we next need to format and mount that new volume. Because we used the `launch_block_device_mappings` parameter (as opposed to the `ami_block_device_mappings` parameter), the block device is actually attached at image build time (the latter parameter is attached upon image creation only) and we can perform all formatting and mount configuration settings at build time.

To perform this configuration, we will add a shell provisioner that references a file called `scripts/storage.sh` to your Packer template:

```

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

```

Adding a shell provisioner for configuring storage

The referenced script is expressed as a path relative to the the Packer template, so you now need to create this script:

```

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

```

Creating a scripts folder

With the script file in place, you can now define the various shell provisioning actions in this script as demonstrated in the following example:

```

#!/usr/bin/env bash

set -e

echo "### 配置 Docker 卷存储 ###"

sudo mkdir -p /data

sudo mkfs.ext4 -L docker /dev/xvdcy

echo -e "LABEL=docker\t/data\t\text4\tdefaults,noatime\t0\t0" | sudo tee -a /etc/fstab

sudo mount -a

```

Storage provisioning script

As you can see in the preceding example, the script is a regular bash script, and it's important to always set the error flag for all of your Packer shell scripts (`set -e`), which ensures the script will exit with an error code should any command fail within the script.

You first create a folder called `/data`, which you will use to store Docker volumes, and then format the `/dev/xvdcy` device you attached earlier in the earlier example with the `.ext4` filesystem, and attach a label called `docker`, which makes mount operations simpler to perform. The next `echo` command adds an entry to the `/etc/fstab` file, which defines all filesystem mounts that will be applied at boot, and notice that you must pipe the `echo` command through to `sudo tee -a /etc/fstab`, which appends the `echo` output to the `/etc/fstab` file with the correct sudo privileges.

Finally, you auto-mount the new entry in the `/etc/fstab` file by running the `mount -a` command, which although not required at image build time, is a simple way to verify that the mount is actually configured correctly (if not, this command will fail and the resulting build will fail).

# Installing additional packages and configuring system settings

The next customizations you will perform are to install additional packages and configuring system settings.

# Installing additional packages

There are a few additional packages that we need to install into our custom ECS container instance, which include the following:

*   **CloudFormation helper scripts**: When you use CloudFormation to deploy your infrastructure, AWS provide a set of CloudFormation helper scripts, collectively referred to as **cfn-bootstrap**, that work with CloudFormation to obtain initialization metadata that allows you to perform custom initialization tasks at instance-creation time, and also signal CloudFormation when the instance has successfully completed initialization. We will explore the benefits of this approach in later chapters, however, for now you need to ensure these helper scripts are present in your custom ECS container-instance image.
*   **CloudWatch logs agent**: The AWS CloudWatch logs service offers central storage of logs from various sources, including EC2 instances, ECS containers, and other AWS services. To ship your ECS container instance (EC2 instance) logs to CloudWatch logs, you must install the CloudWatch logs agent locally, which you can then use to forward various system logs, including operating system, Docker, and ECS agent logs.
*   **`jq` utility**: The `jq` utility ([`stedolan.github.io/jq/manual/`](https://stedolan.github.io/jq/manual/)) is handy for parsing JSON output, and you will need this utility later on in this chapter when you define a simple health check that verifies the ECS container instance has joined to the configured ECS cluster.

Installing these additional packages is very straightforward, and can be achieved by modifying the inline shell provisioner you created earlier:

```

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

```

Installing additional operating system packages

As you can see in the preceding example, each of the required packages can be easily installed using the `yum` package manager.

# Configuring system settings

There are a few minor system settings that you need to make to your custom ECS container instance:

*   Configure time-zone settings
*   Modify default cloud-init behavior

# Configuring timezone settings

Earlier, you defined a variable called `timezone`, which so far you have not referenced in your template. You can use this variable to configure the timezone of your custom ECS container instance image.

To do this, you first need to add a new shell provisioner into your Packer template:

```

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

```

Adding a provisioner to configure time settings

In the preceding example, we reference a script called `scripts/time.sh`, which you will create shortly, but notice that we also include a parameter called `environment_vars`, which allows you to inject your Packer variables (`timezone` in this example) as environment variables into your shell provisioning scripts.

The following example shows the required `scripts/time.sh` script that is referenced in the new Packer template provisioning task:

```

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

```

Time settings provisioning script

In the preceding example, you first configure the [AWS recommended settings for configuring time](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-time.html), configuring the `/etc/sysconfig/clock` file with the configured `TIMEZONE` environment variable, creating the symbolic `/etc/localtime` link, and finally ensuring the `ntpd` service is configured to use the [AWS NTP sync](https://aws.amazon.com/blogs/aws/keeping-time-with-amazon-time-sync-service/) service and start at instance boot.

The AWS NTP sync service is a free AWS service that provides an NTP server endpoint at the `169.254.169.123` local address, ensuring your EC2 instances can obtain accurate time without having to traverse the network or internet.

# Modifying default cloud-init behavior

cloud-init is a standard set of utilities for performing initialization of cloud images and associated instances. The most popular feature of cloud-init is the user-data mechanism, which is a simple means of running your own custom initialization commands at instance creation.

cloud-init is also used in the ECS-Optimized AMI to perform automatic security patching at instance creation, and although this sounds like a useful feature, it can cause problems, particularly in environments where your instances are located in private subnets and require an HTTP proxy to communicate with the internet.

The issue with the cloud-init security mechanism is that although it can be configured to work with an HTTP proxy by setting proxy environment variables, it is invoked prior to when userdata is executed, leading to a chicken-and-egg scenario where you have no option but to disable automated security patching if you are using a proxy.

To disable this mechanism, you first need to configure a new shell provisioner in your Packer template:

```

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

```

Adding a provisioner to configure cloud-init settingsThe referenced `scripts/cloudinit.sh` script can now be created as follows:

```

#!/usr/bin/env bash

set -e

# 禁用 cloud-init 仓库更新或升级

sudo sed -i -e '/^repo_update: /{h;s/: .*/: false/};${x;/^$/{s//repo_update: false/;H};x}' /etc/cloud/cloud.cfg

sudo sed -i -e '/^repo_upgrade: /{h;s/: .*/: none/};${x;/^$/{s//repo_upgrade: none/;H};x}' /etc/cloud/cloud.cfg

```

Disabling security updates for cloud-init

In the following example, the rather scary-looking `sed` expressions will either add or replace lines beginning with `repo_update` and `repo_upgrade` in the `/etc/cloud/cloud.cfg` cloud-init configuration file and ensure they are set to `false` and `none`, respectively.

# Configuring a cleanup script

At this point, we have performed all required installation and configuration shell provisioning tasks. We will create one final shell provisioner, a cleanup script, which will remove any log files created while the instance used to build the custom image was running and to ensure the machine image is in a state ready to be launched.

You first need to add a shell provisioner to your Packer template that references the `scripts/cleanup.sh` script:

```

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

```

Adding a provisioner to clean up the Image

With the provisioner defined in the Packer template, you next need to create the cleanup script, as defined here:

```

#!/usr/bin/env bash

echo "### 执行最终清理任务 ###"

sudo stop ecs

sudo docker system prune -f -a

sudo service docker stop

sudo chkconfig docker off

sudo rm -rf /var/log/docker /var/log/ecs/*

```

Cleanup script

In the following example, notice you don't execute the command `set -e`, given this is a cleanup script that you are not too worried about if there is an error and you don't want your build to fail should a service already be stopped. The ECS agent is first stopped, with the `docker system prune` command used to clear any ECS container state that may be present, and next the Docker service is stopped and then disabled using the `chkconfig` command. The reason for this is that on instance creation, we will always invoke a first-run script that will perform initial configuration of the instance and requires the Docker service to be stopped. Of course this means that once the first-run script has completed its initial configuration, it will be responsible for ensuring the Docker service is both started and enabled to start on boot.

Finally, the cleanup script removes any Docker and ECS agent log files that may have been created during the short period the instance was up during the custom machine-image build process.

# Creating a first-run script

The final set of customizations we will apply to your custom ECS container instance image is to create a first-run script, which will be responsible for performing runtime configuration of your ECS container instance at instance creation, by performing the following tasks:

*   Configuring ECS cluster membership
*   Configuring HTTP proxy support
*   Configuring the CloudWatch logs agent
*   Starting required services
*   Performing health checks

To provision the first-run script, you need to define a file provisioner task in your Packer template, as demonstrated here:

```

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

```

Adding a file provisioner

Notice that the provisioner type is configured as `file`, and specifies a local source file that needs to be located in `files/firstrun.sh`. The `destination` parameter defines the location within the AMI where the first-run script will be located. Note that the file provisioner task copies files as the `ec2-user` user, hence it has limited permissions as to where this script can be copied.

# Configuring ECS cluster membership

You can now create the first-run script at the files/firstrun.sh location referenced by your Packer template. Before you get started configuring this file, it is important to bear in mind that the first-run script is designed to be run at initial boot of an instance created from your custom machine image, so you must consider this when configuring the various commands that will be executed.

We will first create configure the ECS agent to join the ECS cluster that the ECS container instance is intended to join, as demonstrated in the following example:

```

#!/usr/bin/env bash

set -e

# 配置 ECS 代理

echo "ECS_CLUSTER=${ECS_CLUSTER}" > /etc/ecs/ecs.config

```

Configuring ECS cluster membership

Back in Chapter 5, *Publishing Docker Images using ECR*, you saw how the ECS cluster wizard configured ECS container instances using this same approach, although one difference is that the script is expecting an environment variable called `ECS_CLUSTER` to be configured in the environment, as designated by the `${ECS_CLUSTER}` expression. Rather than hardcode the ECS cluster name, which would make the first-run script very inflexible, the idea here is that the configuration being applied to a given instance defines the `ECS_CLUSTER` environment variable with the correct cluster name, meaning the script is reusable and can be configured with any given ECS cluster.

# Configuring HTTP proxy support

A common security best practice is to place your ECS container instances in private subnets, meaning they are located in subnets that possess no default route to the internet. This approach makes it more difficult for attackers to compromise your systems, and even if they do, provides a means to restrict what information they can transmit back to the internet.

Depending on the nature of your application, you typically will require your ECS container instances to be able to connect to the internet, and using an HTTP proxy provides an effective mechanism to provide such access in a controlled manner with Layer 7 application-layer inspection capabilities.

Regardless of the nature of your application, it is important to understand that ECS container instances require internet connectivity for the following purposes:

*   ECS agent control-plane and management-plane communications with ECS
*   Docker Engine communication with ECR and other repositories for downloading Docker images
*   CloudWatch logs agent communication with the CloudWatch logs service
*   CloudFormation helper-script communication with the CloudFormation service

Although configuring a complete end-to-end proxy solution is outside the scope of this book, it is useful to understand how you can customize your ECS container instances to use an HTTP proxy, as demonstrated in the following example:

```

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

```

Configuring HTTP proxy support

In the preceding example, the script checks for the existence of a non-empty environment variable called `PROXY_URL`, and if present proceeds to configure proxy settings for various components of the ECS container instance:

*   Docker Engine: Configured via `/etc/sysconfig/docker`
*   ECS agent: Configured via `/etc/ecs/ecs.config`
*   CloudWatch logs agent: Configured via `/etc/awslogs/proxy.conf`

Notice that in some cases you need to configure the `NO_PROXY` setting, which disables proxy communications for the following IP addresses:

*   `169.254.169.254`: This is a special local address that is used to communicate with the EC2 metadata service to obtain instance metadata, such as EC2 instance role credentials.
*   `169.254.170.2`: This is a special local address that is used to obtained ECS task credentials.

# Configuring the CloudWatch logs agent

The next configuration task that you will perform in the first-run script is to configure the CloudWatch logs agent. On an ECS container instance, the CloudWatch logs agent is responsible for collecting system logs, such as operating system, Docker, and ECS agent logs.

Note that this agent is NOT required to implement CloudWatch logs support for your Docker containers - this is already implemented within the Docker Engine via the `awslogs` logging driver.

Configuring the CloudWatch logs agent requires you to perform the following configuration tasks:

*   **Configure the correct AWS region**: For this task, you will inject the value of an environment variable called `AWS_DEFAULT_REGION` and write this to the `/etc/awslogs/awscli.conf` file.
*   **Define the various log group and log stream settings that the CloudWatch logs agent will log to**: For this task, you will define the recommended set of log groups for ECS container instances, which is described at [`docs.aws.amazon.com/AmazonECS/latest/developerguide/using_cloudwatch_logs.html#configure_cwl_agent`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_cloudwatch_logs.html#configure_cwl_agent)

The following example demonstrates the required configuration:

```

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

```

Configuring the CloudWatch logs agent

You can see that the first-run script includes references to environment variables in the `log_group_name` parameter for each defined log group, which helps ensure unique log group naming in your AWS account:

*   `STACK_NAME`: The name of the CloudFormation stack
*   `AUTOSCALING_GROUP`: The name of the Autoscaling Group

Again, these environment variables must be injected at instance creation to the first-run script, so bear this in mind for future chapters when we will learn how to do this.

One other point to note in the preceding example is the value of each `log_stream_name` parameter - this is set to a special variable called `{instance_id}`, which the CloudWatch logs agent will automatically configure with the EC2 instance ID of the instance.

The end result is that you will get several log groups for each type of log, which are scoped to the context of a given CloudFormation stack and EC2 auto scaling group, and within each log group, a log stream for each ECS container instance will be created, as illustrated in the following diagram:

![](img/a199eec7-ad83-4e86-bd90-ac0febe0df2f.png)CloudWatch logs group configuration for ECS container instances

# Starting required services

Recall in the previous examples, that you added a cleanup script as part of the image-build process, which disables the Docker Engine service from starting on boot. This approach allows you to perform required initialization tasks prior to starting the Docker Engine, and at this point in the first-run script we are ready to start the Docker Engine and other important system services:

```

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

```

Starting services

In the preceding example, note that I have omitted the earlier parts of the first-run script for brevity. Notice that you first start the awslogs service, which ensures the CloudWatch logs agent will pick up all Docker Engine logs, and then proceed to enable Docker to start on boot, start Docker, and finally start the ECS agent.

# Performing required health checks

The final task we need to perform in the first-run script is a health check, which ensures the ECS container instance has initialized and successfully registered to the configured ECS cluster. This is a reasonable health check for your ECS container instances, given the ECS agent can only run if the Docker Engine is operational, and the ECS agent must be registered with the ECS cluster in order to deploy your applications.

Recall in the previous chapter, when you examined the internals of an ECS container instance that the ECS agent exposes a local HTTP endpoint that can be queried for current ECS agent status. You can use this endpoint to create a very simple health check, as demonstrated here:

```

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

```

Performing a health check

In the preceding example, a bash `until` loop is configured, which uses curl to query the `http://localhost:51678/v1/metadata` endpoint every five seconds. The output of this command is piped through to `jq`, which will either return the Cluster property or an empty value if this property is not present. Once the ECS agent registers to the correct ECS cluster and returns this property in the JSON response, the loop will complete and the first-run script will complete.

# Testing your custom ECS container instance image

You have now completed all customizations and it is time to rebuild your image using the `packer build` command. Before you do this, now is a good time to verify you have the correct Packer template in place, and also have created the associated supporting files. The following example shows the folder and file structure you should now have in your packer-ecs repository:

```

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

```

Verifying the Packer repository

Assuming everything is in place, you can now run your Packer build once again by running the `make build` command.

Once everything is complete and your AMI has been successfully created, you can now view your AMI in the AWS console by navigating to **Services** | **EC2** and selecting AMIs from the menu on the left:

![](img/286a5d2f-7e98-467b-a671-5f3ad55eaf24.png)EC2 dashboard AMIs

In the preceding screenshot, you can see the two AMIs you built earlier in this chapter and just now. Notice that the most recent AMI now includes three block devices, with `/dev/xvdcy` representing the additional 20 GB gp2 volume you added earlier in this chapter.

At this point, you can actually test out your AMI by clicking on the **Launch** button, which will start the EC2 Instance Wizard. After clicking the **Review and Launch** button, click on the **Edit security groups** link to grant your IP address access via SSH to the instance, as shown in the following screenshot:

![](img/c0fd0ffb-bc0b-4026-b9c0-9d8232b143cd.png)Launching a new EC2 instance

Once complete, click on **Review and Launch**, then click the **Launch** button, and finally configure an appropriate SSH key pair that you have access to.

On the launching instance screen, you can now click the link to your new EC2 instance, and copy the public IP address so that you can SSH to the instance, as shown in the following screenshot:

![](img/99e70e6b-1183-437c-bf3b-7d2c04fde547.png)Connecting to a new EC2 instance

Once you have connected to the instance, you can verify that the additional 20 GB volume you configured for Docker volume storage has been successfully mounted:

```

> sudo mount

proc 在/proc 上的类型为 proc（rw，relatime）

sysfs 在/sys 上的类型为 sysfs（rw，relatime）

/dev/xvda1 在/上的类型为 ext4（rw，noatime，data=ordered）

devtmpfs 在/dev 上的类型为 devtmpfs（rw，relatime，size=500292k，nr_inodes=125073，mode=755）

devpts 在/dev/pts 上的类型为 devpts（rw，relatime，gid=5，mode=620，ptmxmode=000）

tmpfs 在/dev/shm 上的类型为 tmpfs（rw，relatime）

/dev/xvdcy 在/data 上的类型为 ext4（rw，noatime，data=ordered）

none 在/proc/sys/fs/binfmt_misc 上的类型为 binfmt_misc（rw，relatime）

```

Verifying storage mounts

You can check the timezone is configured correctly by running the `date` command, which should display the correct timezone (US/Eastern), and also verify the `ntpd` service is running:

```

> 日期

Wed Feb 21 06:45:40 EST 2018

> sudo service ntpd status

ntpd 正在运行

```

Verifying time settings

Next, you can verify the cloud-init configuration has been configured to disable security updates, by viewing the `/etc/cloud/cloud.cfg` file:

```

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

```

Verifying cloud-init settings

You should also verify that the Docker service is stopped and was disabled on boot, as per the cleanup script you configured:

```

> sudo service docker status

docker 已停止

> sudo chkconfig --list docker

docker 0:off 1:off 2:off 3:off 4:off 5:off 6:off

```

Verifying disabled services

Finally, you can verify that the first-run script is present in the `ec2-user` home directory:

```

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

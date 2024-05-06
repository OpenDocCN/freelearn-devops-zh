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
Enter MFA code for arn:aws:iam::385605022855:mfa/justin.menga: ****
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

在上面的示例中，注意你可以使用反引号（`` ` ``) 作为bash命令替换的替代语法。换句话说，`$(command)`和` `` `command` ``都表示将执行命令并返回输出的命令替换。

# 构建镜像

现在我们有了自动生成临时会话凭据的机制，假设您的`packer.json`文件和 Makefile 位于您的 packer-ecs 存储库的根目录中，让我们通过运行`make build`来测试构建您的 Packer 镜像：

```
> export AWS_PROFILE=docker-in-aws
> make build
Enter MFA code for arn:aws:iam::385605022855:mfa/justin.menga: ******
packer build packer.json
amazon-ebs output will be in this color.

==> amazon-ebs: Prevalidating AMI Name: docker-in-aws-ecs 1518934269
    amazon-ebs: Found Image ID: ami-5e414e24
==> amazon-ebs: Creating temporary keypair: packer_5a8918fd-018d-964f-4ab3-58bff320ead5
==> amazon-ebs: Creating temporary security group for this instance: packer_5a891904-2c84-aca1-d368-8309f215597d
==> amazon-ebs: Authorizing access to port 22 from 0.0.0.0/0 in the temporary security group...
==> amazon-ebs: Launching a source AWS instance...
==> amazon-ebs: Adding tags to source instance
    amazon-ebs: Adding tag: "Name": "Packer Builder"
    amazon-ebs: Instance ID: i-04c150456ac0748aa
==> amazon-ebs: Waiting for instance (i-04c150456ac0748aa) to become ready...
==> amazon-ebs: Waiting for SSH to become available...
==> amazon-ebs: Connected to SSH!
==> amazon-ebs: Provisioning with shell script: /var/folders/s4/1mblw7cd29s8xc74vr3jdmfr0000gn/T/packer-shell190211980
    amazon-ebs: Loaded plugins: priorities, update-motd, upgrade-helper
    amazon-ebs: Resolving Dependencies
    amazon-ebs: --> Running transaction check
    amazon-ebs: ---> Package elfutils-libelf.x86_64 0:0.163-3.18.amzn1 will be updated
    amazon-ebs: ---> Package elfutils-libelf.x86_64 0:0.168-8.19.amzn1 will be an update
    amazon-ebs: ---> Package python27.x86_64 0:2.7.12-2.121.amzn1 will be updated
    amazon-ebs: ---> Package python27.x86_64 0:2.7.13-2.122.amzn1 will be an update
    amazon-ebs: ---> Package python27-libs.x86_64 0:2.7.12-2.121.amzn1 will be updated
    amazon-ebs: ---> Package python27-libs.x86_64 0:2.7.13-2.122.amzn1 will be an update
    amazon-ebs: --> Finished Dependency Resolution
    amazon-ebs:
    amazon-ebs: Dependencies Resolved
    amazon-ebs:
    amazon-ebs: ================================================================================
    amazon-ebs: Package Arch Version Repository Size
    amazon-ebs: ================================================================================
    amazon-ebs: Updating:
    amazon-ebs: elfutils-libelf x86_64 0.168-8.19.amzn1 amzn-updates 313 k
    amazon-ebs: python27 x86_64 2.7.13-2.122.amzn1 amzn-updates 103 k
    amazon-ebs: python27-libs x86_64 2.7.13-2.122.amzn1 amzn-updates 6.8 M
    amazon-ebs:
    amazon-ebs: Transaction Summary
    amazon-ebs: ================================================================================
    amazon-ebs: Upgrade 3 Packages
    amazon-ebs:
    amazon-ebs: Total download size: 7.2 M
    amazon-ebs: Downloading packages:
    amazon-ebs: --------------------------------------------------------------------------------
    amazon-ebs: Total 5.3 MB/s | 7.2 MB 00:01
    amazon-ebs: Running transaction check
    amazon-ebs: Running transaction test
    amazon-ebs: Transaction test succeeded
    amazon-ebs: Running transaction
    amazon-ebs: Updating : python27-2.7.13-2.122.amzn1.x86_64 1/6
    amazon-ebs: Updating : python27-libs-2.7.13-2.122.amzn1.x86_64 2/6
    amazon-ebs: Updating : elfutils-libelf-0.168-8.19.amzn1.x86_64 3/6
    amazon-ebs: Cleanup : python27-2.7.12-2.121.amzn1.x86_64 4/6
    amazon-ebs: Cleanup : python27-libs-2.7.12-2.121.amzn1.x86_64 5/6
    amazon-ebs: Cleanup : elfutils-libelf-0.163-3.18.amzn1.x86_64 6/6
    amazon-ebs: Verifying : python27-libs-2.7.13-2.122.amzn1.x86_64 1/6
    amazon-ebs: Verifying : elfutils-libelf-0.168-8.19.amzn1.x86_64 2/6
    amazon-ebs: Verifying : python27-2.7.13-2.122.amzn1.x86_64 3/6
    amazon-ebs: Verifying : python27-libs-2.7.12-2.121.amzn1.x86_64 4/6
    amazon-ebs: Verifying : elfutils-libelf-0.163-3.18.amzn1.x86_64 5/6
    amazon-ebs: Verifying : python27-2.7.12-2.121.amzn1.x86_64 6/6
    amazon-ebs:
    amazon-ebs: Updated:
    amazon-ebs: elfutils-libelf.x86_64 0:0.168-8.19.amzn1
    amazon-ebs: python27.x86_64 0:2.7.13-2.122.amzn1
    amazon-ebs: python27-libs.x86_64 0:2.7.13-2.122.amzn1
    amazon-ebs:
    amazon-ebs: Complete!
==> amazon-ebs: Stopping the source instance...
    amazon-ebs: Stopping instance, attempt 1
==> amazon-ebs: Waiting for the instance to stop...
==> amazon-ebs: Creating the AMI: docker-in-aws-ecs 1518934269
    amazon-ebs: AMI: ami-57415b2d
==> amazon-ebs: Waiting for AMI to become ready...
==> amazon-ebs: Adding tags to AMI (ami-57415b2d)...
==> amazon-ebs: Tagging snapshot: snap-0bc767fd982333bf8
==> amazon-ebs: Tagging snapshot: snap-0104c1a352695c1e9
==> amazon-ebs: Creating AMI tags
    amazon-ebs: Adding tag: "SourceAMI": "ami-5e414e24"
    amazon-ebs: Adding tag: "DockerVersion": "17.09.1-ce"
    amazon-ebs: Adding tag: "ECSAgentVersion": "1.17.0-2"
    amazon-ebs: Adding tag: "Name": "Docker in AWS ECS Base Image 2017.09.h"
==> amazon-ebs: Creating snapshot tags
==> amazon-ebs: Terminating the source AWS instance...
==> amazon-ebs: Cleaning up any extra volumes...
==> amazon-ebs: No volumes to clean up, skipping
==> amazon-ebs: Deleting temporary security group...
==> amazon-ebs: Deleting temporary keypair...
==> amazon-ebs: Running post-processor: manifest
Build 'amazon-ebs' finished.

==> Builds finished. The artifacts of successful builds are:
--> amazon-ebs: AMIs were created:
us-east-1: ami-57415b2d
```

运行 Packer 构建

回顾前面的示例和上一个示例的输出，在`build`任务中注意到构建 Packer 镜像的命令只是`packer build <template-file>`，在这种情况下是`packer build packer.json`。

如果您回顾上一个示例的输出，您会看到以下步骤由 Packer 执行：

+   Packer 首先验证源 AMI，然后生成临时 SSH 密钥对和安全组，以便能够访问临时 EC2 实例。

+   Packer 从源 AMI 启动临时 EC2 实例，然后等待能够建立 SSH 访问。

+   Packer 根据模板的 provisioners 部分中定义的配置执行配置操作。在这种情况下，您可以看到 yum `update` 命令的输出，这是我们当前的单个配置操作。

+   完成后，Packer 停止实例并创建 EBS 卷实例的快照，从而生成具有适当名称和 ID 的 AMI。

+   创建完成后，Packer 终止实例，删除临时 SSH 密钥对和安全组，并输出新的 AMI ID。

回顾前面的示例，您向模板添加了一个 manifest 后处理器，并且您应该在存储库的根目录中找到一个名为`manifest.json`的文件，通常您不会想要提交到您的 **packer-ecs** 存储库中：

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

查看 Packer 构建清单

# 使用 Packer 构建自定义 ECS 容器实例镜像

在前一节中，您已经建立了一个用于使用 Packer 构建自定义 AMI 的基本模板，并且继续构建和发布了您的第一个自定义 AMI。在这一点上，您尚未执行任何特定于 ECS 容器实例配置的自定义操作，因此本节将重点介绍如何改进您的 Packer 模板以包括这些自定义操作。

您现在将了解以下自定义内容：

+   定义自定义存储配置

+   安装额外的软件包并配置操作系统设置

+   配置清理脚本

+   创建第一次运行脚本

有了这些自定义设置，我们将通过构建最终的自定义 ECS 容器实例 AMI 并启动实例来完成本章，并验证各种自定义设置。

# 定义自定义存储配置

AWS ECS 优化的 AMI 包括一个使用 30GB EBS 卷的默认存储配置，分区如下：

+   `/dev/xvda`：作为根文件系统挂载的 8GB 卷，用作操作系统分区。

+   `dev/xvdcz`：一个 22GB 卷，配置为逻辑卷管理（LVM）设备，用于 Docker 镜像和元数据存储。

ECS 优化的 AMI 使用 devicemapper 存储驱动程序进行 Docker 镜像和元数据存储，您可以在[`docs.docker.com/v17.09/engine/userguide/storagedriver/device-mapper-driver/`](https://docs.docker.com/storage/storagedriver/device-mapper-driver/)上了解更多信息。

对于大多数用例，这种存储配置应该足够了，但是有一些情况下您可能希望修改默认配置：

+   **你需要更多的 Docker 镜像和元数据存储**：这可以通过简单地配置您的 ECS 容器实例以使用更大的卷大小来轻松解决。默认存储配置将始终保留 8GB 用于操作系统和根文件系统，其余存储用于 Docker 镜像和元数据存储。

+   **你需要支持具有大容量存储需求的 Docker 卷**：默认情况下，ECS 优化的 AMI 将 Docker 卷存储在 `/var/lib/docker/volumes`，这是根文件系统中 8GB `/dev/xvda` 分区的一部分。如果您有更大的卷需求，这可能会导致您的操作系统分区很快变满，所以在这种情况下，您应该将卷存储分离到单独的 EBS 卷中。

现在让我们看看您如何修改您的 Packer 模板，以为 Docker 卷存储添加一个新的专用卷，并确保在实例创建时正确挂载此卷。

# 添加 EBS 卷

要向您的自定义 AMI 添加 EBS 卷，您可以在 Amazon EBS 构建器中配置 `launch_block_device_mappings` 参数：

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

添加一个启动块设备映射

在上述示例中，为了简洁起见，我已经截断了 Packer 模板的其他部分，您可以看到我们添加了一个名为 `/dev/xvdcy` 的 20GB 单一卷，该卷配置为在实例终止时销毁。请注意，`volume_type` 参数设置为 `gp2`，这是通常在 AWS 中提供最佳整体价格/性能的通用 SSD 存储类型。

# 格式化和挂载卷

有了上述示例的配置，我们接下来需要格式化和挂载新卷。因为我们使用了 `launch_block_device_mappings` 参数（而不是 `ami_block_device_mappings` 参数），所以块设备实际上是在构建镜像时附加的（后者仅在创建镜像时附加），我们可以在构建时执行所有格式化和挂载配置设置。

要执行此配置，我们将向您的 Packer 模板添加一个 shell provisioner，该 provisioner 引用名为 `scripts/storage.sh` 的文件：

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

添加一个用于配置存储的 shell provisioner

引用的脚本表示为相对于 Packer 模板的路径，因此您现在需要创建此脚本：

```

> mkdir -p scripts
> touch scripts/storage.sh
> tree
.
├── Makefile
├── manifest.json
├── packer.json
└── scripts
 └── storage.sh

1 directory, 4 files

```

创建一个 scripts 文件夹

通过使用以下示例中所示的脚本文件，你可以定义各种 shell 配置操作：

```

#!/usr/bin/env bash
set -e

echo "### Configuring Docker Volume Storage ###"
sudo mkdir -p /data
sudo mkfs.ext4 -L docker /dev/xvdcy
echo -e "LABEL=docker\t/data\t\text4\tdefaults,noatime\t0\t0" | sudo tee -a /etc/fstab
sudo mount -a
```

存储配置脚本

如你在前面的示例中所见，这个脚本是一个普通的 bash 脚本，重要的是要为所有的 Packer shell 脚本设置错误标志 (`set -e`)，这样可以确保脚本在任何命令失败时都会以错误代码退出。

首先创建一个名为 `/data` 的文件夹，用于存储 Docker 卷，然后使用 `.ext4` 文件系统格式化之前示例中附加的 `/dev/xvdcy` 设备，并附加一个名为 `docker` 的标签，这使得挂载操作更加简单。下一个 `echo` 命令将添加一个条目到 `/etc/fstab` 文件中，该文件定义了在启动时将应用的所有文件系统挂载，注意你必须通过 `sudo tee -a /etc/fstab` 将 `echo` 命令传递给 `sudo`，以使用正确的 sudo 权限将 `echo` 的输出追加到 `/etc/fstab` 文件中。

最后，通过运行 `mount -a` 命令，你可以自动挂载 `/etc/fstab` 文件中的新条目，尽管在构建镜像时不是必需的，但这是一种简单的方法来验证该挂载是否正确配置（如果不正确，此命令将失败并导致构建失败）。

# 安装额外的软件包和配置系统设置

接下来，你将执行其他自定义操作，例如安装额外的软件包和配置系统设置。

# 安装额外的软件包

我们需要安装一些额外的软件包到我们的自定义 ECS 容器实例中，包括以下几个：

+   **CloudFormation 帮助脚本**：当你使用 CloudFormation 部署基础设施时，AWS 提供了一组称为 **cfn-bootstrap** 的 CloudFormation 帮助脚本，它们与 CloudFormation 一起工作，以获取初始化元数据，允许你在实例创建时执行自定义初始化任务，并在实例成功完成初始化后向 CloudFormation 发出信号。我们将在后面的章节中探讨这种方法的好处，但现在你需要确保这些帮助脚本存在于你的自定义 ECS 容器实例镜像中。

+   **CloudWatch 日志代理**：AWS CloudWatch 日志服务提供了从各种来源（包括 EC2 实例、ECS 容器和其他 AWS 服务）集中存储日志的功能。要将 ECS 容器实例（EC2 实例）的日志发送到 CloudWatch 日志，你必须在本地安装 CloudWatch 日志代理，并将其用于转发各种系统日志，包括操作系统、Docker 和 ECS 代理的日志。

+   **`jq` 实用程序**：`jq` 实用程序（[`stedolan.github.io/jq/manual/`](https://stedolan.github.io/jq/manual/)）对于解析 JSON 输出很方便，在本章后面当您定义一个简单的健康检查来验证 ECS 容器实例是否已加入到配置的 ECS 集群时，您将需要此实用程序。

安装这些额外的软件包非常简单，可以通过修改您之前创建的内联 shell provisioner 来实现：

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

安装其他操作系统软件包

如您在上述示例中所见，每个所需的软件包都可以通过 `yum` 软件包管理器轻松安装。

# 配置系统设置

您需要对自定义 ECS 容器实例进行一些小的系统设置：

+   配置时区设置

+   修改默认的 cloud-init 行为

# 配置时区设置

之前，您定义了一个名为 `timezone` 的变量，到目前为止您还没有在模板中引用过。您可以使用此变量来配置自定义 ECS 容器实例镜像的时区。

要做到这一点，您首先需要在您的 Packer 模板中添加一个新的 shell provisioner：

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

添加一个 provisioner 来配置时间设置

在上述示例中，我们引用了一个名为 `scripts/time.sh` 的脚本，您将很快创建它，但请注意，我们还包含了一个名为 `environment_vars` 的参数，它允许您将您的 Packer 变量（例如此示例中的 `timezone`）作为环境变量注入到您的 shell provisioning 脚本中。

下面的示例展示了新的 Packer 模板配置任务中引用的必需 `scripts/time.sh` 脚本：

```

#!/usr/bin/env bash
set -e

# Configure host to use timezone
# http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-time.html
echo "### Setting timezone to end-inline-katex-->TIMEZONE ###"
sudo tee /etc/sysconfig/clock << EOF > /dev/null
ZONE="c194a9eg<!-- begin-inline-katexTIMEZONE"
UTC=true
EOF

sudo ln -sf /usr/share/zoneinfo/"end-inline-katex-->TIMEZONE" /etc/localtime

# Use AWS NTP Sync service
echo "server 169.254.169.123 prefer iburst" | sudo tee -a /etc/ntp.conf

# Enable NTP
sudo chkconfig ntpd on
```

时间设置配置脚本

在上面的示例中，首先配置了[配置时间的 AWS 推荐设置](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-time.html)，通过配置 `/etc/sysconfig/clock` 文件使用配置的 `TIMEZONE` 环境变量，创建了符号链接 `/etc/localtime`，最后确保 `ntpd` 服务配置为使用[AWS NTP 同步服务](https://aws.amazon.com/blogs/aws/keeping-time-with-amazon-time-sync-service/)并在实例启动时启动。

AWS NTP 同步服务是一个免费的 AWS 服务，提供了一个位于本地地址 `169.254.169.123` 的 NTP 服务器端点，确保您的 EC2 实例可以获得准确的时间，而无需穿越网络或互联网。

# 修改默认的 cloud-init 行为

cloud-init 是一组标准的工具，用于执行云映像和相关实例的初始化。cloud-init 最流行的功能是 user-data 机制，它是在实例创建时运行您自己的自定义初始化命令的一种简单方法。

云初始化还用于 ECS 优化的 AMI，以在实例创建时执行自动安全补丁，尽管这听起来像一个有用的功能，但它可能会导致问题，特别是在您的实例位于私有子网并且需要使用 HTTP 代理与互联网通信的环境中。

云初始化安全机制的问题在于，虽然可以通过设置代理环境变量来配置它与 HTTP 代理一起工作，但它在执行 userdata 之前被调用，导致鸡和蛋的情况，即如果你使用代理，你别无选择，只能禁用自动安全补丁。

要禁用此机制，您首先需要在 Packer 模板中配置一个新的外壳供应商：

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
    }
  ],
  "post-processors": [...]
}
```

添加一个供应商以配置云初始化设置引用的`scripts/cloudinit.sh`脚本现在可以按以下方式创建：

```

#!/usr/bin/env bash
set -e

# Disable cloud-init repo updates or upgrades
sudo sed -i -e '/^repo_update: /{h;s/: .*/: false/};c194a9eg<!-- begin-inline-katex{x;/^end-inline-katex-->/{s//repo_update: false/;H};x}' /etc/cloud/cloud.cfg
sudo sed -i -e '/^repo_upgrade: /{h;s/: .*/: none/};c194a9eg<!-- begin-inline-katex{x;/^end-inline-katex-->/{s//repo_upgrade: none/;H};x}' /etc/cloud/cloud.cfg
复制 ErrorOK!

```

禁用云初始化的安全更新

在下面的示例中，看起来相当可怕的`sed`表达式将在`/etc/cloud/cloud.cfg`云初始化配置文件中添加或替换以`repo_update`和`repo_upgrade`开头的行，并确保它们分别设置为`false`和`none`。

# 配置清理脚本

到目前为止，我们已经执行了所有必需的安装和配置外壳供应任务。我们将创建一个最终的外壳供应商，一个清理脚本，它将删除构建自定义镜像的实例运行时创建的任何日志文件，并确保机器镜像处于准备启动的状态。

您首先需要向 Packer 模板添加一个引用`scripts/cleanup.sh`脚本的外壳供应商：

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

添加一个清理镜像的供应商

定义了 Packer 模板中的供应商之后，您需要创建清理脚本，如下所示：

```

#!/usr/bin/env bash
echo "### Performing final clean-up tasks ###"
sudo stop ecs
sudo docker system prune -f -a
sudo service docker stop
sudo chkconfig docker off
sudo rm -rf /var/log/docker /var/log/ecs/*
```

清理脚本

在下面的示例中，请注意您不执行`set -e`命令，因为这是一个清理脚本，如果出现错误，您不太担心，也不希望如果服务已经停止，构建失败。首先停止 ECS 代理，使用`docker system prune`命令清除可能存在的任何 ECS 容器状态，然后停止 Docker 服务，并使用`chkconfig`命令停用它。原因是在实例创建时，我们总是会调用一个首次运行脚本，该脚本将执行实例的初始配置，并要求停止 Docker 服务。当然，这意味着一旦首次运行脚本完成其初始配置，它将负责确保 Docker 服务已启动并且已启用以在启动时启动。

最后，清理脚本将删除在自定义机器镜像构建过程中实例运行的短时间内可能创建的任何 Docker 和 ECS 代理日志文件。

# 创建首次运行脚本

我们将对自定义 ECS 容器实例镜像应用的最终一组自定义是创建一个首次运行脚本，该脚本将负责在实例创建时执行 ECS 容器实例的运行时配置，执行以下任务：

+   配置 ECS 集群成员身份

+   配置 HTTP 代理支持

+   配置 CloudWatch 日志代理

+   启动所需服务

+   执行健康检查

要提供首次运行脚本，您需要在您的 Packer 模板中定义一个文件提供者任务，如下所示：

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

添加文件提供者

注意，配置了提供者类型为 `file`，并指定了需要位于 `files/firstrun.sh` 中的本地源文件。 `destination` 参数定义了首次运行脚本将位于 AMI 中的位置。请注意，文件提供者任务将文件复制为 `ec2-user` 用户，因此它对可以复制该脚本的位置具有有限的权限。

# 配置 ECS 集群成员身份

您现在可以在您的 Packer 模板引用的 files/firstrun.sh 位置创建首次运行脚本。在开始配置此文件之前，重要的是要记住，首次运行脚本被设计为在从您的自定义机器映像创建的实例的初始引导时运行，因此在配置将执行的各种命令时，必须考虑到这一点。

我们首先将配置 ECS 代理加入 ECS 集群，以加入 ECS 容器实例打算加入的 ECS 集群，如下面的示例所示：

```

#!/usr/bin/env bash
set -e

# Configure ECS Agent
echo "ECS_CLUSTER=c194a9eg<!-- begin-inline-katex{ECS_CLUSTER}" > /etc/ecs/ecs.config

```

配置 ECS 集群成员身份

回到第五章，*使用 ECR 发布 Docker 镜像*，您看到了 ECS 集群向导使用了这种相同的方法配置 ECS 容器实例，尽管有一个区别是脚本期望在环境中配置了一个名为 `ECS_CLUSTER` 的环境变量，如 `${ECS_CLUSTER}` 表达式所指定的那样。与硬编码 ECS 集群名称不同，这会使首次运行脚本非常不灵活，这里的想法是，应用于给定实例的配置定义了具有正确集群名称的 `ECS_CLUSTER` 环境变量，这意味着该脚本是可重用的，并且可以配置为任何给定的 ECS 集群。

# 配置 HTTP 代理支持

一个常见的安全最佳实践是将您的 ECS 容器实例放置在私有子网中，这意味着它们位于没有默认路由到互联网的子网中。这种方法使得攻击者更难以破坏您的系统，即使他们这样做了，也提供了一种限制他们可以向互联网传输的信息的方法。

根据您的应用程序性质，您通常需要您的 ECS 容器实例能够连接到互联网，使用 HTTP 代理提供了一种有效的机制，以控制方式提供具有第 7 层应用层检查功能的访问。

无论您的应用程序的性质如何，都重要的是要了解，ECS 容器实例需要互联网连接，用于以下目的：

+   ECS 代理控制平面和管理平面与 ECS 的通信

+   Docker 引擎与 ECR 和其他存储库的通信，以下载 Docker 镜像

+   CloudWatch 日志代理与 CloudWatch 日志服务的通信

+   CloudFormation 辅助脚本与 CloudFormation 服务的通信

尽管配置完整的端到端代理解决方案超出了本书的范围，但了解如何自定义 ECS 容器实例以使用 HTTP 代理是有用的，如下例所示：

```

#!/usr/bin/env bash
set -e

# Configure ECS Agent
echo "ECS_CLUSTER=c194a9eg<!-- begin-inline-katex{ECS_CLUSTER}" > /etc/ecs/ecs.config

# Set HTTP Proxy URL if provided
if [ -n end-inline-katex-->PROXY_URL ]
then
 echo export HTTPS_PROXY=c194a9eg<!-- begin-inline-katexPROXY_URL >> /etc/sysconfig/docker
 echo HTTPS_PROXY=end-inline-katex-->PROXY_URL >> /etc/ecs/ecs.config
 echo NO_PROXY=169.254.169.254,169.254.170.2,/var/run/docker.sock >> /etc/ecs/ecs.config
 echo HTTP_PROXY=c194a9eg<!-- begin-inline-katexPROXY_URL >> /etc/awslogs/proxy.conf
 echo HTTPS_PROXY=end-inline-katex-->PROXY_URL >> /etc/awslogs/proxy.conf
 echo NO_PROXY=169.254.169.254 >> /etc/awslogs/proxy.conf
fi
```

配置 HTTP 代理支持

在上面的示例中，脚本检查名为`PROXY_URL`的非空环境变量是否存在，如果存在，则继续为 ECS 容器实例的各个组件配置代理设置：

+   Docker 引擎：通过`/etc/sysconfig/docker`配置

+   ECS 代理：通过`/etc/ecs/ecs.config`配置

+   CloudWatch 日志代理：通过`/etc/awslogs/proxy.conf`配置

请注意，在某些情况下，您需要配置`NO_PROXY`设置，该设置禁用以下 IP 地址的代理通信：

+   `169.254.169.254`：这是一个特殊的本地地址，用于与 EC2 元数据服务通信，以获取实例元数据，如 EC2 实例角色凭证。

+   `169.254.170.2`：这是一个特殊的本地地址，用于获取 ECS 任务凭证。

# 配置 CloudWatch 日志代理

您将在首次运行脚本中执行的下一个配置任务是配置 CloudWatch 日志代理。在 ECS 容器实例上，CloudWatch 日志代理负责收集系统日志，例如操作系统、Docker 和 ECS 代理日志。

请注意，此代理不需要为您的 Docker 容器实现 CloudWatch 日志支持 - 这已经在 Docker 引擎中通过`awslogs`日志驱动程序实现了。

配置 CloudWatch 日志代理需要执行以下配置任务：

+   **配置正确的 AWS 区域**：对于这个任务，你将注入一个名为`AWS_DEFAULT_REGION`的环境变量的值，并将其写入`/etc/awslogs/awscli.conf`文件中。

+   **定义 CloudWatch 日志代理将记录到的各种日志组和日志流设置**：对于这个任务，您将定义 ECS 容器实例的建议日志组集，该集在[`docs.aws.amazon.com/AmazonECS/latest/developerguide/using_cloudwatch_logs.html#configure_cwl_agent`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_cloudwatch_logs.html#configure_cwl_agent)中描述。

下面的示例演示了所需的配置：

```

#!/usr/bin/env bash
set -e

# Configure ECS Agent
echo "ECS_CLUSTER=c194a9eg<!-- begin-inline-katex{ECS_CLUSTER}" > /etc/ecs/ecs.config

# Set HTTP Proxy URL if provided
if [ -n end-inline-katex-->PROXY_URL ]
then
  echo export HTTPS_PROXY=c194a9eg<!-- begin-inline-katexPROXY_URL >> /etc/sysconfig/docker
  echo HTTPS_PROXY=end-inline-katex-->PROXY_URL >> /etc/ecs/ecs.config
  echo NO_PROXY=169.254.169.254,169.254.170.2,/var/run/docker.sock >> /etc/ecs/ecs.config
  echo HTTP_PROXY=c194a9eg<!-- begin-inline-katexPROXY_URL >> /etc/awslogs/proxy.conf
  echo HTTPS_PROXY=end-inline-katex-->PROXY_URL >> /etc/awslogs/proxy.conf
  echo NO_PROXY=169.254.169.254 >> /etc/awslogs/proxy.conf
fi

# Write AWS Logs region
sudo tee /etc/awslogs/awscli.conf << EOF > /dev/null
[plugins]
cwlogs = cwlogs
[default]
region = c194a9eg<!-- begin-inline-katex{AWS_DEFAULT_REGION}
EOF

# Write AWS Logs config
sudo tee /etc/awslogs/awslogs.conf << EOF > /dev/null
[general]
state_file = /var/lib/awslogs/agent-state 

[/var/log/dmesg]
file = /var/log/dmesg
log_group_name = /end-inline-katex-->{STACK_NAME}/ec2/c194a9eg<!-- begin-inline-katex{AUTOSCALING_GROUP}/var/log/dmesg
log_stream_name = {instance_id} 
[/var/log/messages]
file = /var/log/messages
log_group_name = /end-inline-katex-->{STACK_NAME}/ec2/c194a9eg<!-- begin-inline-katex{AUTOSCALING_GROUP}/var/log/messages
log_stream_name = {instance_id}
datetime_format = %b %d %H:%M:%S 
[/var/log/docker]
file = /var/log/docker
log_group_name = /end-inline-katex-->{STACK_NAME}/ec2/c194a9eg<!-- begin-inline-katex{AUTOSCALING_GROUP}/var/log/docker
log_stream_name = {instance_id}
datetime_format = %Y-%m-%dT%H:%M:%S.%f 
[/var/log/ecs/ecs-init.log]
file = /var/log/ecs/ecs-init.log*
log_group_name = /end-inline-katex-->{STACK_NAME}/ec2/c194a9eg<!-- begin-inline-katex{AUTOSCALING_GROUP}/var/log/ecs/ecs-init
log_stream_name = {instance_id}
datetime_format = %Y-%m-%dT%H:%M:%SZ
time_zone = UTC 
[/var/log/ecs/ecs-agent.log]
file = /var/log/ecs/ecs-agent.log*
log_group_name = /end-inline-katex-->{STACK_NAME}/ec2/c194a9eg<!-- begin-inline-katex{AUTOSCALING_GROUP}/var/log/ecs/ecs-agent
log_stream_name = {instance_id}
datetime_format = %Y-%m-%dT%H:%M:%SZ
time_zone = UTC

[/var/log/ecs/audit.log]
file = /var/log/ecs/audit.log*
log_group_name = /end-inline-katex-->{STACK_NAME}/ec2/c194a9eg<!-- begin-inline-katex{AUTOSCALING_GROUP}/var/log/ecs/audit.log
log_stream_name = {instance_id}
datetime_format = %Y-%m-%dT%H:%M:%SZ
time_zone = UTC
EOF
```

配置 CloudWatch 日志代理

您可以看到第一次运行脚本中包含对每个定义的日志组的`log_group_name`参数中环境变量的引用，这有助于确保在您的 AWS 账户中具有唯一的日志组命名：

+   `STACK_NAME`：CloudFormation 堆栈的名称

+   `AUTOSCALING_GROUP`：自动缩放组的名称

再次强调，这些环境变量必须在实例创建时注入到第一次运行的脚本中，请记住这一点，因为在未来的章节中，我们将学习如何执行此操作。

在前面的示例中需要注意的另一点是每个`log_stream_name`参数的值 - 这设置为一个称为`{instance_id}`的特殊变量，CloudWatch 日志代理将自动配置为实例的 EC2 实例 ID。

结果是，对于每种类型的日志，您将获得几个日志组，这些日志组的范围限定为特定的 CloudFormation 堆栈和 EC2 自动缩放组的上下文，并且在每个日志组中，将为每个 ECS 容器实例创建一个日志流，如下图所示：

![图](img/a199eec7-ad83-4e86-bd90-ac0febe0df2f.png)ECS 容器实例的 CloudWatch 日志组配置

# 启动所需服务

在前面的示例中，您添加了一个清理脚本作为镜像构建过程的一部分，该脚本禁用了 Docker 引擎服务在启动时的启动。这种方法允许您在启动 Docker 引擎之前执行所需的初始化任务，在第一次运行脚本的这一点上，我们准备好启动 Docker 引擎和其他重要的系统服务：

```

#!/usr/bin/env bash
set -e

# Configure ECS Agent
echo "ECS_CLUSTER=end-inline-katex-->{ECS_CLUSTER}" > /etc/ecs/ecs.config

# Set HTTP Proxy URL if provided
...
...

# Write AWS Logs region
...
...

# Write AWS Logs config
...
...

# Start services
sudo service awslogs start
sudo chkconfig docker on
sudo service docker start
sudo start ecs
```

启动服务

在前面的示例中，请注意，出于简洁起见，我省略了第一次运行脚本的早期部分。请注意，您首先启动了 awslogs 服务，这确保了 CloudWatch 日志代理将捕获所有 Docker 引擎日志，然后继续启用 Docker 以在启动时启动，启动 Docker，最后启动 ECS 代理。

# 执行所需的健康检查

在第一次运行脚本中我们需要执行的最终任务是健康检查，以确保 ECS 容器实例已初始化并成功注册到配置的 ECS 集群。鉴于 ECS 代理只能在 Docker 引擎可用时运行，并且必须将 ECS 代理注册到 ECS 集群中以部署您的应用程序，因此这是对您的 ECS 容器实例的合理健康检查。

在上一章中回顾，当您检查 ECS 容器实例的内部时，ECS 代理公开了一个本地 HTTP 端点，可以查询当前 ECS 代理状态。您可以使用此端点创建一个非常简单的健康检查，如下所示：

```

#!/usr/bin/env bash
set -e

# Configure ECS Agent
echo "ECS_CLUSTER=c194a9eg<!-- begin-inline-katex{ECS_CLUSTER}" > /etc/ecs/ecs.config

# Set HTTP Proxy URL if provided
...
...

# Write AWS Logs region
...
...

# Write AWS Logs config
...
...

# Start services
...
...

# Health check
# Loop until ECS agent has registered to ECS cluster
echo "Checking ECS agent is joined to end-inline-katex-->{ECS_CLUSTER}"
until [[ "c194a9eg<!-- begin-inline-katex(curl --fail --silent http://localhost:51678/v1/metadata | jq '.Cluster // empty' -r -e)" == end-inline-katex-->{ECS_CLUSTER} ]]
 do printf '.'
 sleep 5
done
echo "ECS agent successfully joined to ${ECS_CLUSTER}"
```

执行健康检查

在上面的示例中，配置了一个 bash `until` 循环，该循环使用 curl 查询`http://localhost:51678/v1/metadata`端点，每五秒钟一次。此命令的输出通过管道传输到`jq`，它将返回 Cluster 属性或如果不存在此属性，则返回空值。一旦 ECS 代理注册到正确的 ECS 集群并在 JSON 响应中返回此属性，循环将完成，并且第一次运行脚本将完成。

# 测试你的自定义 ECS 容器实例映像

你现在已经完成了所有的定制工作，现在是使用`packer build`命令重建你的映像的时候了。在此之前，现在是验证你已经放置了正确的 Packer 模板，并且也创建了相关的支持文件的好时机。下面的示例显示了你的 packer-ecs 仓库现在应该具有的文件夹和文件结构：

```

> tree
.
├── Makefile
├── files
│   └── firstrun.sh
├── manifest.json
├── packer.json
└── scripts
    ├── cleanup.sh
    ├── cloudinit.sh
    ├── storage.sh
    └── time.sh

2 directories, 8 files
```

验证 Packer 仓库

假设一切就绪，你现在可以通过运行`make build`命令再次运行你的 Packer 构建。

一旦一切都完成并且你的 AMI 已成功创建，你现在可以通过导航到**服务** | **EC2**并从左侧菜单中选择 AMIs 来在 AWS 控制台中查看你的 AMI：

![](img/286a5d2f-7e98-467b-a671-5f3ad55eaf24.png)EC2 仪表板 AMIs

在上面的截图中，你可以看到你在本章和刚刚构建的两个 AMI。请注意，最近的 AMI 现在包括三个块设备，其中`/dev/xvdcy`代表你在本章前面添加的额外 20 GB gp2 卷。

此时，你可以通过点击**启动**按钮来测试你的 AMI，这将启动 EC2 实例向导。点击**审核并启动**按钮后，点击**编辑安全组**链接以通过 SSH 将你的 IP 地址授权给实例，如下截图所示：

![](img/c0fd0ffb-bc0b-4026-b9c0-9d8232b143cd.png)启动新的 EC2 实例

完成后，点击**审核并启动**，然后点击**启动**按钮，最后配置你有权限访问的适当的 SSH 密钥对。

在启动实例屏幕上，你现在可以点击链接到你的新 EC2 实例，并复制公共 IP 地址，以便你可以通过 SSH 连接到实例，如下截图所示：

![](img/99e70e6b-1183-437c-bf3b-7d2c04fde547.png)连接到新的 EC2 实例

连接到实例后，你可以验证你为 Docker 卷存储配置的额外 20 GB 卷已成功挂载：

```

> sudo mount
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
/dev/xvda1 on / type ext4 (rw,noatime,data=ordered)
devtmpfs on /dev type devtmpfs (rw,relatime,size=500292k,nr_inodes=125073,mode=755)
devpts on /dev/pts type devpts (rw,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /dev/shm type tmpfs (rw,relatime)
/dev/xvdcy on /data type ext4 (rw,noatime,data=ordered)
none on /proc/sys/fs/binfmt_misc type binfmt_misc (rw,relatime)
```

验证存储挂载

你可以通过运行`date`命令来检查时区是否正确配置，该命令应显示正确的时区（美国/东部），并验证`ntpd`服务是否正在运行：

```

> date
Wed Feb 21 06:45:40 EST 2018
> sudo service ntpd status
ntpd is runningntpd 正在运行

```

验证时间设置

接下来，你可以通过查看`/etc/cloud/cloud.cfg`文件来验证 cloud-init 配置已经配置为禁用安全更新：

```

> cat /etc/cloud/cloud.cfg
# WARNING: Modifications to this file may be overridden by files in
# /etc/cloud/cloud.cfg.d

# If this is set, 'root' will not be able to ssh in and they
# will get a message to login instead as the default user (ec2-user)
disable_root: true

# This will cause the set+update hostname module to not operate (if true)
preserve_hostname: true

datasource_list: [ Ec2, None ]

repo_upgrade: none
repo_upgrade_exclude:
 - kernel
 - nvidia*
 - cudatoolkit

mounts:
 - [ ephemeral0, /media/ephemeral0 ]
 - [ swap, none, swap, sw, "0", "0" ]
# vim:syntax=yaml
repo_update: false
```

验证 cloud-init 设置

你还应该验证 Docker 服务是否已停止，并根据你配置的清理脚本在启动时被禁用：

```

> sudo service docker status
docker is stopped
> sudo chkconfig --list docker
docker 0:off 1:off 2:off 3:off 4:off 5:off 6:off
```

验证已禁用的服务

最后，你可以验证 `ec2-user` 用户的家目录中是否存在首次运行脚本：


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

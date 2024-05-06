# 使用 ECR 发布 Docker 镜像

Docker 注册表是 Docker 和容器生态系统的关键组件，提供了一种通用机制来公开和分发您的容器应用程序，无论是公开还是私有。

ECR 提供了一个完全托管的私有 Docker 注册表，具有与上一章介绍的 ECS 组件和其他 AWS 服务紧密集成的特性。ECR 具有高度可扩展性，安全性，并提供工具来与用于构建和发布 Docker 镜像的本机 Docker 客户端集成。

在本章中，您将学习如何创建 ECR 存储库来存储您的 Docker 镜像，使用各种机制，包括 AWS 控制台，AWS CLI 和 CloudFormation。一旦您建立了第一个 ECR 存储库，您将学习如何使用 ECR 进行身份验证，拉取存储在您的存储库中的 Docker 镜像，并使用 Docker 客户端构建和发布 Docker 镜像到 ECR。最后，您将学习如何处理更高级的 ECR 使用和管理场景，包括配置跨帐户访问以允许在其他 AWS 帐户中运行的 Docker 客户端访问您的 ECR 存储库，并配置生命周期策略，以确保孤立的 Docker 镜像定期清理，减少管理工作量和成本。

将涵盖以下主题：

+   了解 ECR

+   创建 ECR 存储库

+   登录到 ECR

+   将 Docker 镜像发布到 ECR

+   从 ECR 拉取 Docker 镜像

+   配置生命周期策略

# 技术要求

以下列出了完成本章所需的技术要求：

+   Docker 18.06 或更高版本

+   Docker Compose 1.22 或更高版本

+   GNU Make 3.82 或更高版本

+   jq

+   AWS CLI 1.15.71 或更高版本

+   对 AWS 帐户的管理员访问权限

+   本地 AWS 配置文件按第三章中的说明配置

+   在第二章中配置的示例应用程序的工作 Docker 工作流程（请参阅[`github.com/docker-in-aws/docker-in-aws/tree/master/ch2`](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch2)）。

此 GitHub URL 包含本章中使用的代码示例：[`github.com/docker-in-aws/docker-in-aws/tree/master/ch5`](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch5)。

查看以下视频以查看代码的实际操作：

[`bit.ly/2PKMLSP`](http://bit.ly/2PKMLSP)

# 了解 ECR

在开始创建和配置 ECR 存储库之前，重要的是要简要介绍 ECR 的核心概念。

ECR 是由 AWS 提供的完全托管的私有 Docker 注册表，并与 ECS 和其他 AWS 服务紧密集成。ECR 包括许多组件，如下图所示：

![](img/4472aae0-f99e-482d-a642-fad5cc753bfa.png)ECR 架构

ECR 的核心组件包括：

+   **仓库**: 仓库存储给定 Docker 镜像的所有版本。每个仓库都配置有名称和 URI，该 URI 对于您的 AWS 帐户和区域是唯一的。

+   **权限**: 每个仓库都包括权限，允许您授予各种 ECR 操作的访问权限，例如推送或拉取 Docker 镜像。

+   **生命周期策略**: 每个仓库都可以配置一个可选的生命周期策略，用于清理已被新版本取代的孤立的 Docker 镜像，或者删除您可能不再使用的旧 Docker 镜像。

+   **认证服务**: ECR 包括一个认证服务，其中包括一个令牌服务，可用于以临时认证令牌交换您的 IAM 凭据以进行身份验证，与 Docker 客户端身份验证过程兼容。

考虑 ECR 的消费者也很重要。如前图所示，这些包括：

+   **与您的仓库在同一本地 AWS 帐户中的 Docker 客户端**: 这通常包括在 ECS 集群中运行的 ECS 容器实例。

+   **不同 AWS 帐户中的 Docker 客户端**: 这是较大组织的常见情况，通常包括在远程帐户中运行的 ECS 集群中的 ECS 容器实例。

+   **AWS 服务使用的 Docker 客户端**: 一些 AWS 服务可以利用您在 ECR 中发布的自己的 Docker 镜像，例如 AWS CodeBuild 服务。

在撰写本书时，ECR 仅作为私有注册表提供 - 这意味着如果您想公开发布您的 Docker 镜像，那么至少在发布您的公共 Docker 镜像方面，ECR 不是正确的解决方案。

# 创建 ECR 仓库

现在您已经对 ECR 有了基本概述，让我们开始创建您的第一个 ECR 存储库。回想一下，在早期的章节中，您已经介绍了本书的示例**todobackend**应用程序，并在本地环境中构建了一个 Docker 镜像。为了能够在基于此镜像的 ECS 集群上运行容器，您需要将此镜像发布到 ECS 容器实例可以访问的 Docker 注册表中，而 ECR 正是这个问题的完美解决方案。

为**todobackend**应用程序创建 ECR 存储库，我们将专注于三种流行的方法来创建和配置您的存储库：

+   使用 AWS 控制台创建 ECR 存储库

+   使用 AWS CLI 创建 ECR 存储库

+   使用 AWS CloudFormation 创建 ECR 存储库

# 使用 AWS 控制台创建 ECR 存储库

通过执行以下步骤，可以在 AWS 控制台上创建 ECR 存储库：

1.  从主 AWS 控制台中，选择**服务** | **弹性容器服务**，在计算部分中选择**存储库**，然后单击“开始”按钮。

1.  您将被提示配置存储库的名称。一个标准的约定是以`<organization>/<application>`格式命名您的存储库，这将导致一个完全合格的存储库 URI 为`<registry>/<organization>/<application>`。在下面的示例中，我将存储库命名为`docker-in-aws/todobackend`，但您可以根据自己的喜好命名您的镜像。完成后，点击“下一步”继续：

![](img/b8a39b1e-b38f-4f4b-91d7-162b33f6f0ea.png)配置存储库名称

1.  您的 ECR 存储库现在将被创建，并提供如何登录到 ECR 并发布您的 Docker 镜像的说明。

# 使用 AWS CLI 创建 ECR 存储库

通过运行`aws ecr create-repository`命令可以创建 ECR 存储库，但是考虑到您已经通过 AWS 控制台创建了存储库，让我们看看如何检查 ECR 存储库是否已经存在以及如何使用 AWS CLI 删除存储库。

查看您的 AWS 帐户和本地区域中的 ECR 存储库列表，您可以使用`aws ecr list-repositories`命令，而要删除 ECR 存储库，您可以使用`aws ecr delete-repository`命令，如下所示：

```
> aws ecr list-repositories
{
    "repositories": [
        {
            "repositoryArn": "arn:aws:ecr:us-east-1:385605022855:repository/docker-in-aws/todobackend",
            "registryId": "385605022855",
            "repositoryName": "docker-in-aws/todobackend",
            "repositoryUri": "385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend",
            "createdAt": 1517692382.0
        }
    ]
}
> aws ecr delete-repository --repository-name docker-in-aws/todobackend
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:us-east-1:385605022855:repository/docker-in-aws/todobackend",
        "registryId": "385605022855",
        "repositoryName": "docker-in-aws/todobackend",
        "repositoryUri": "385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend",
        "createdAt": 1517692382.0
    }
}
```

使用 AWS CLI 描述和删除 ECR 存储库

现在，您已经使用 AWS 控制台删除了之前创建的仓库，您可以按照这里演示的方法重新创建它：

```
> aws ecr create-repository --repository-name docker-in-aws/todobackend
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:us-east-1:385605022855:repository/docker-in-aws/todobackend",
        "registryId": "385605022855",
        "repositoryName": "docker-in-aws/todobackend",
        "repositoryUri": "385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend",
        "createdAt": 1517693074.0
    }
}
```

使用 AWS CLI 创建 ECR 仓库

# 使用 AWS CloudFormation 创建 ECR 仓库

AWS CloudFormation 支持通过`AWS::ECR::Repository`资源类型创建 ECR 仓库，在撰写本文时，这允许您管理 ECR 资源策略和生命周期策略，我们将在本章后面介绍。

作为一个经验法则，鉴于 ECR 仓库作为 Docker 镜像分发机制的关键性质，我通常建议将您的帐户和区域中的各种 ECR 仓库定义在一个单独的共享 CloudFormation 堆栈中，专门用于创建和管理 ECR 仓库。

遵循这个建议，并为将来的章节，让我们创建一个名为**todobackend-aws**的仓库，您可以用来存储您将在本书中创建和管理的各种基础架构配置。我会让您在 GitHub 上创建相应的仓库，之后您可以将您的 GitHub 仓库配置为远程仓库：

```
> mkdir todobackend-aws
> touch todobackend-aws/ecr.yml > cd todobackend-aws
> git init Initialized empty Git repository in /Users/jmenga/Source/docker-in-aws/todobackend-aws/.git/
> git remote add origin https://github.com/jmenga/todobackend-aws.git
> tree .
.
└── ecr.yml
```

现在，您可以配置一个名为`ecr.yml`的 CloudFormation 模板文件，该文件定义了一个名为`todobackend`的单个 ECR 仓库：

```
AWSTemplateFormatVersion: "2010-09-09"

Description: ECR Repositories

Resources:
  TodobackendRepository:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: docker-in-aws/todobackend
```

使用 AWS CloudFormation 定义 ECR 仓库

正如您在前面的示例中所看到的，使用 CloudFormation 定义 ECR 仓库非常简单，只需要定义`RepositoryName`属性，这个属性定义了仓库的名称，正如您所期望的那样。

假设您已经删除了之前的 todobackend ECR 仓库，就像之前演示的那样，现在您可以使用`aws cloudformation deploy`命令使用 CloudFormation 创建 todobackend 仓库：

```
> aws cloudformation deploy --template-file ecr.yml --stack-name ecr-repositories
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - ecr-repositories
```

使用 AWS CloudFormation 创建 ECR 仓库

一旦堆栈成功部署，您可以在 CloudFormation 控制台中查看堆栈，如下截图所示：

![](img/355c6511-887a-4c48-a0d6-d563459b8ed9.png)ECR 仓库 CloudFormation 堆栈

如果您现在返回 ECS 控制台，并从左侧菜单中选择**资源**，您应该会看到一个名为`docker-in-aws/todobackend`的单个 ECR 仓库，就像在您的 CloudFormation 堆栈中定义的那样。如果您点击该仓库，您将进入仓库详细页面，该页面为您提供了仓库 URI、仓库中发布的镜像列表、ECR 权限和生命周期策略设置。

# 登录到 ECR

创建 Docker 镜像的存储库后，下一步是构建并将您的镜像发布到 ECR。在此之前，您必须对 ECR 进行身份验证，因为在撰写本文时，ECR 是一个不支持公共访问的私有服务。

登录到 ECR 的说明和命令显示在 ECR 存储库向导的一部分中，但是您可以随时通过选择适当的存储库并单击**查看推送命令**按钮来查看这些说明，该按钮将显示登录、构建和发布 Docker 镜像到存储库所需的各种命令。

显示的第一个命令是`aws ecr get-login`命令，它将生成一个包含临时身份验证令牌的`docker login`表达式，有效期为 12 小时（请注意，出于节省空间的考虑，命令输出已被截断）：

```
> aws ecr get-login --no-include-email
docker login -u AWS -p eyJwYXl2ovSUVQUkJkbGJ5cjQ1YXJkcnNLV29ubVV6TTIxNTk3N1RYNklKdllvanZ1SFJaeUNBYk84NTJ2V2RaVzJUYlk9Iiw
idmVyc2lvbiI6IjIiLCJ0eXBlIjoiREFUQV9LRVkiLCJleHBpcmF0aW9uIjoxNTE4MTIyNTI5fQ== https://385605022855.dkr.ecr.us-east-1.amazonaws.com
```

为 ECR 生成登录命令

对于 Docker 版本 17.06 及更高版本，`--no-include-email`标志是必需的，因为从此版本开始，`-e` Docker CLI 电子邮件标志已被弃用。

尽管您可以复制并粘贴前面示例中生成的命令输出，但更快的方法是使用 bash 命令替换自动执行`aws ecr get-login`命令的输出，方法是用`$(...)`将命令括起来：

```
> $(aws ecr get-login --no-include-email)
Login Succeeded
```

登录到 ECR

# 将 Docker 镜像发布到 ECR

在早期的章节中，您学习了如何使用 todobackend 示例应用程序在本地构建和标记 Docker 镜像。

现在，您可以将此工作流程扩展到将 Docker 镜像发布到 ECR，这需要您执行以下任务：

+   确保您已登录到 ECR

+   使用您的 ECR 存储库的 URI 构建和标记您的 Docker 镜像

+   将您的 Docker 镜像推送到 ECR

# 使用 Docker CLI 发布 Docker 镜像

您已经看到如何登录 ECR，并且构建和标记您的 Docker 镜像与本地使用情况大致相同，只是在标记图像时需要指定 ECR 存储库的 URI。

以下示例演示了构建`todobackend`镜像，使用您的新 ECR 存储库的 URI 标记图像（用于您的存储库的实际 URI），并使用`docker images`命令验证图像名称：

```
> cd ../todobackend
> docker build -t 385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend .
Sending build context to Docker daemon 129.5kB
Step 1/25 : FROM alpine AS build
 ---> 3fd9065eaf02
Step 2/25 : LABEL application=todobackend
 ---> Using cache
 ---> f955808a07fd
...
...
...
Step 25/25 : USER app
 ---> Running in 4cf3fcab97c9
Removing intermediate container 4cf3fcab97c9
---> 2b2d8d17367c
Successfully built 2b2d8d17367c
Successfully tagged 385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend:latest
> docker images
REPOSITORY                                                             TAG    IMAGE ID     SIZE 
385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend latest 2b2d8d17367c 99.4MB
```

为 ECR 标记图像

构建并标记了您的镜像后，您可以将您的镜像推送到 ECR。

请注意，要将图像发布到 ECR，您需要各种 ECR 权限。因为您在您的帐户中使用管理员角色，所以您自动拥有所有所需的权限。我们将在本章后面更详细地讨论 ECR 权限。

因为您已经登录到 ECR，所以只需使用`docker push`命令并引用您的 Docker 图像的名称即可：

```
> docker push 385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend
The push refers to repository [385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend]
1cdf73b07ed7: Pushed
0dfffc4aa16e: Pushed
baaced0ec8f8: Pushed
e3b27097ac3f: Pushed
3a29354c4bcc: Pushed
a031167f960b: Pushed
cd7100a72410: Pushed
latest: digest: sha256:322c8b378dd90b3a1a6dc8553baf03b4eb13ebafcc926d9d87c010f08e0339fa size: 1787
```

将图像推送到 ECR

如果您现在在 ECS 控制台中导航到 todobackend 存储库，您应该会看到您新发布的图像以默认的`latest`标签出现，如下图所示。请注意，当您比较图像的构建大小（在我的示例中为 99 MB）与存储在 ECR 中的图像大小（在我的示例中为 34 MB）时，您会发现 ECR 以压缩格式存储图像，从而降低了存储成本。

在使用 ECR 时，AWS 会对数据存储和数据传输（即拉取 Docker 图像）收费。有关更多详细信息，请参见[`aws.amazon.com/ecr/pricing/`](https://aws.amazon.com/ecr/pricing/)！[](assets/9cb36e30-9f49-412d-833c-93abf0e56183.png)查看 ECR 图像

# 使用 Docker Compose 发布 Docker 图像

在之前的章节中，您已经学会了如何使用 Docker Compose 来帮助简化测试和构建 Docker 图像所需的 CLI 命令数量。目前，Docker Compose 只能在本地构建 Docker 图像，但当然您现在希望能够发布您的 Docker 图像并利用您的 Docker Compose 工作流程。

Docker Compose 包括一个名为`image`的服务配置属性，通常用于指定要运行的容器的图像：

```
version: '2.4'

services:
  web:
    image: nginx
```

示例 Docker Compose 文件

尽管这是 Docker Compose 的一个非常常见的使用模式，但如果您结合`build`和`image`属性，还存在另一种配置和行为集，如在 todobackend 存储库的`docker-compose.yml`文件中所示：

```
version: '2.4'

volumes:
  public:
    driver: local

services:
  test:
    build:
      context: .
      dockerfile: Dockerfile
      target: test
  release:
 image: 385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend:latest
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      DJANGO_SETTINGS_MODULE: todobackend.settings_release
      MYSQL_HOST: db
      MYSQL_USER: todo
      MYSQL_PASSWORD: password
  app:
    image: 385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend:${APP_VERSION}
    extends:
  ...
  ...
```

Todobackend Docker Compose 文件

在上面的示例中，为`release`和`app`服务同时指定了`image`和`build`属性。当这两个属性一起使用时，Docker 仍将从引用的 Dockerfile 构建图像，但会使用`image`属性指定的值对图像进行标记。

您可以通过创建新服务并定义包含附加标签的图像属性来应用多个标签。

请注意，对于`app`服务，我们引用环境变量`APP_VERSION`，这意味着要使用在 todobackend 存储库根目录的 Makefile 中定义的当前应用程序版本标记图像：

```
.PHONY: test release clean version

export APP_VERSION ?= $(shell git rev-parse --short HEAD)

version:
  @ echo '{"Version": "$(APP_VERSION)"}'
```

在上面的示例中，用您自己 AWS 账户生成的适当 URI 替换存储库 URI。

为了演示当您结合`image`和`build`属性时的标记行为，首先删除本章前面创建的 Docker 图像，如下所示：

```
> docker rmi 385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend
Untagged: 385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend:latest
Untagged: 385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend@sha256:322c8b378dd90b3a1a6dc8553baf03b4eb13ebafcc926d9d87c010f08e0339fa
Deleted: sha256:2b2d8d17367c32993b0aa68f407e89bf4a3496a1da9aeb7c00a8e49f89bf5134
Deleted: sha256:523126379df325e1bcdccdf633aa10bc45e43bdb5ce4412aec282e98dbe076fb
Deleted: sha256:54521ab8917e466fbf9e12a5e15ac5e8715da5332f3655e8cc51f5ad3987a034
Deleted: sha256:03d95618180182e7ae08c16b4687a7d191f3f56d909b868db9e889f0653add46
Deleted: sha256:eb56d3747a17d5b7d738c879412e39ac2739403bbf992267385f86fce2f5ed0d
Deleted: sha256:9908bfa1f773905e0540d70e65d6a0991fa1f89a5729fa83e92c2a8b45f7bd29
Deleted: sha256:d9268f192cb01d0e05a1f78ad6c41bc702b11559d547c0865b4293908d99a311
Deleted: sha256:c6e4f60120cdf713253b24bba97a0c2a80d41a0126eb18f4ea5269034dbdc7e1
Deleted: sha256:0b780adf8501c8a0dbf33f49425385506885f9e8d4295f9bc63c3f895faed6d1
```

删除 Docker 图像

如果您现在运行`docker-compose build release`命令，一旦命令完成，Docker Compose 将构建一个新的图像，并标记为您的 ECR 存储库 URI：

```
> docker-compose build release WARNING: The APP_VERSION variable is not set. Defaulting to a blank string.
Building release
Step 1/25 : FROM alpine AS build
 ---> 3fd9065eaf02
Step 2/25 : LABEL application=todobackend
 ---> Using cache
 ---> f955808a07fd
...
...
Step 25/25 : USER app
 ---> Using cache
 ---> f507b981227f

Successfully built f507b981227f
Successfully tagged 385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend:latest
> docker images
```

```
REPOSITORY                                                               TAG                 IMAGE ID            CREATED             SIZE
385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend   latest              f507b981227f        4 days ago          99.4MB
```

使用 Docker Compose 构建带标签的图像

当您的图像构建并正确标记后，您现在可以执行`docker-compose push`命令，该命令可用于推送在 Docker Compose 文件中定义了`build`和`image`属性的服务：

```
> docker-compose push release
Pushing release (385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend:latest)...
The push refers to repository [385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend]
9ae8d6169643: Layer already exists
cdbc5d8be7d1: Pushed
08a1fb32c580: Layer already exists
2e3946df4029: Pushed
3a29354c4bcc: Layer already exists
a031167f960b: Layer already exists
cd7100a72410: Layer already exists
latest: digest: sha256:a1b029d347a2fabd3f58d177dcbbcd88066dc54ccdc15adad46c12ceac450378 size: 1787
```

使用 Docker Compose 发布图像

在上面的示例中，与名为`release`的服务关联的图像被推送，因为这是您使用 Docker 图像 URI 配置的服务。

# 自动化发布工作流程

在之前的章节中，您学习了如何使用 Docker、Docker Compose 和 Make 自动化测试和构建 todobackend 应用程序的 Docker 图像。

现在，您可以增强此工作流程以执行以下附加操作：

+   登录和注销 ECR

+   发布到 ECR

为了实现这一点，您将在 todobackend 存储库的 Makefile 中创建新的任务。

# 自动化登录和注销

以下示例演示了添加名为`login`和`logout`的两个新任务，这些任务将使用 Docker 客户端执行这些操作：

```
.PHONY: test release clean version login logout

export APP_VERSION ?= $(shell git rev-parse --short HEAD)

version:
  @ echo '{"Version": "$(APP_VERSION)"}'

login:
 $$(aws ecr get-login --no-include-email)

logout:
 docker logout https://385605022855.dkr.ecr.us-east-1.amazonaws.com test:
    docker-compose build --pull release
    docker-compose build
    docker-compose run test

release:
    docker-compose up --abort-on-container-exit migrate
    docker-compose run app python3 manage.py collectstatic --no-input
    docker-compose up --abort-on-container-exit acceptance
    @ echo App running at http://$$(docker-compose port app 8000 | sed s/0.0.0.0/localhost/g)

clean:
    docker-compose down -v
    docker images -q -f dangling=true -f label=application=todobackend | xargs -I ARGS docker rmi -f ARGS
```

登录和注销 ECR

请注意，`login`任务使用双美元符号($$)，这是必需的，因为 Make 使用单美元符号来定义 Make 变量。当您指定双美元符号时，Make 将向 shell 传递单美元符号，这将确保执行 bash 命令替换。

在使用`logout`任务注销时，请注意您需要指定 Docker 注册表，否则 Docker 客户端会假定默认的公共 Docker Hub 注册表。

有了这些任务，您现在可以轻松地使用`make logout`和`make login`命令注销和登录 ECR：

```
> make logout docker logout https://385605022855.dkr.ecr.us-east-1.amazonaws.com
Removing login credentials for 385605022855.dkr.ecr.us-east-1.amazonaws.com
 > make login
$(aws ecr get-login --no-include-email)
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
Login Succeeded
```

运行 make logout 和 make login

# 自动化发布 Docker 图像

要自动化发布工作流，您可以在 Makefile 中添加一个名为`publish`的新任务，该任务简单地调用标记为`release`和`app`服务的`docker-compose push`命令：

```
.PHONY: test release clean login logout publish

export APP_VERSION ?= $(shell git rev-parse --short HEAD)

version:
  @ echo '{"Version": "$(APP_VERSION)"}'

...
...

release:
    docker-compose up --abort-on-container-exit migrate
    docker-compose run app python3 manage.py collectstatic --no-input
    docker-compose up --abort-on-container-exit acceptance
    @ echo App running at http://$$(docker-compose port app 8000 | sed s/0.0.0.0/localhost/g)

publish:
 docker-compose push release app
clean:
    docker-compose down -v
    docker images -q -f dangling=true -f label=application=todobackend | xargs -I ARGS docker rmi -f ARGS
```

自动发布到 ECR

有了这个配置，您的 Docker 镜像现在将被标记为提交哈希和最新标记，然后您只需运行`make publish`命令即可将其发布到 ECR。

现在让我们提交您的更改并运行完整的 Make 工作流来测试、构建和发布您的 Docker 镜像，如下例所示。请注意，一个带有提交哈希`97e4abf`标记的镜像被发布到了 ECR：

```
> git commit -a -m "Add publish tasks"
[master 97e4abf] Add publish tasks
 2 files changed, 12 insertions(+), 1 deletion(-)

> make login
$(aws ecr get-login --no-include-email)
Login Succeeded

> make test && make release
docker-compose build --pull release
Building release
...
...
todobackend_db_1 is up-to-date
Creating todobackend_app_1 ... done
App running at http://localhost:32774
$ make publish
docker-compose push release app
Pushing release (385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend:latest)...
The push refers to repository [385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend]
53ca7006d9e4: Layer already exists
ca208f4ebc53: Layer already exists
1702a4329d94: Layer already exists
e2aca0d7f367: Layer already exists
c3e0af9081a5: Layer already exists
20ae2e176794: Layer already exists
cd7100a72410: Layer already exists
latest: digest: sha256:d64e1771440208bde0cabe454f213d682a6ad31e38f14f9ad792fabc51008888 size: 1787
Pushing app (385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend:97e4abf)...
The push refers to repository [385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend]
53ca7006d9e4: Layer already exists
ca208f4ebc53: Layer already exists
1702a4329d94: Layer already exists
e2aca0d7f367: Layer already exists
c3e0af9081a5: Layer already exists
20ae2e176794: Layer already exists
cd7100a72410: Layer already exists
97e4abf: digest: sha256:d64e1771440208bde0cabe454f213d682a6ad31e38f14f9ad792fabc51008888 size: 1787

> make clean
docker-compose down -v
Stopping todobackend_app_1 ... done
Stopping todobackend_db_1 ... done
...
...

> make logout
docker logout https://385605022855.dkr.ecr.us-east-1.amazonaws.com
Removing login credentials for 385605022855.dkr.ecr.us-east-1.amazonaws.com

```

运行更新后的 Make 工作流

# 从 ECR 中拉取 Docker 镜像

现在您已经学会了如何将 Docker 镜像发布到 ECR，让我们专注于在各种场景下运行的 Docker 客户端如何从 ECR 拉取您的 Docker 镜像。回想一下本章开头对 ECR 的介绍，客户端访问 ECR 存在各种场景，我们现在将重点关注这些场景，以 ECS 容器实例作为您的 Docker 客户端：

+   在与您的 ECR 存储库相同的账户中运行的 ECS 容器实例

+   运行在不同账户中的 ECS 容器实例访问您的 ECR 存储库

+   需要访问您的 ECR 存储库的 AWS 服务

# 来自相同账户的 ECS 容器实例对 ECR 的访问

当您的 ECS 容器实例在与您的 ECR 存储库相同的账户中运行时，推荐的方法是使用与运行为 ECS 容器实例的 EC2 实例应用的 IAM 实例角色相关联的 IAM 策略，以使在 ECS 容器实例内运行的 ECS 代理能够从 ECR 中拉取 Docker 镜像。您已经在上一章中看到了这种方法的实际操作，AWS 提供的 ECS 集群向导附加了一个名为`AmazonEC2ContainerServiceforEC2Role`的托管策略到集群中 ECS 容器实例的 IAM 实例角色，并注意到此策略中包含的以下 ECR 权限：

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:CreateCluster",
        "ecs:DeregisterContainerInstance",
        "ecs:DiscoverPollEndpoint",
        "ecs:Poll",
        "ecs:RegisterContainerInstance",
        "ecs:StartTelemetrySession",
        "ecs:Submit*",
        "ecr:GetAuthorizationToken",
 "ecr:BatchCheckLayerAvailability",
 "ecr:GetDownloadUrlForLayer",
 "ecr:BatchGetImage",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

AmazonEC2ContainerServiceforEC2Role 策略

在上面的例子中，您可以看到授予了四个 ECR 权限，这些权限共同允许 ECS 代理登录到 ECR 并拉取 Docker 镜像：

+   `ecr:GetAuthorizationToken`：允许检索有效期为 12 小时的身份验证令牌，可用于使用 Docker CLI 登录到 ECR。

+   `ecr:BatchCheckLayerAvailability`: 检查给定存储库中多个镜像层的可用性。

+   `ecr:GetDownloadUrlForLayer`: 为 Docker 镜像中的给定层检索预签名的 S3 下载 URL。

+   `ecr:BatchGetImage`: 重新获取给定存储库中 Docker 镜像的详细信息。

这些权限足以登录到 ECR 并拉取镜像，但请注意前面示例中的`Resource`属性允许访问您帐户中的所有存储库。

根据您组织的安全要求，对所有存储库的广泛访问可能是可以接受的，也可能不可以接受 - 如果不可以接受，则需要创建自定义 IAM 策略，限制对特定存储库的访问，就像这里演示的那样：

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": [
        "arn:aws:ecr:us-east-1:385605022855:repository/docker-in-aws/todobackend"
      ]
    }
  ]
}
```

授予特定存储库的 ECR 登录和拉取权限

在前面的示例中，请注意`ecr:GetAuthorizationToken`权限仍然适用于所有资源，因为您没有登录到特定的 ECR 存储库，而是登录到给定区域中您帐户的 ECR 注册表。然而，用于拉取 Docker 镜像的其他权限可以应用于单个存储库，您可以看到这些权限仅允许对您的 ECR 存储库的 ARN 进行操作。

请注意，如果您还想要在前面的示例中授予对 ECR 存储库的推送访问权限，则需要额外的 ECR 权限：

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",         
        "ecr:InitiateLayerUpload",         
        "ecr:UploadLayerPart",         
        "ecr:CompleteLayerUpload"
      ],
      "Resource": [
        "arn:aws:ecr:us-east-1:385605022855:repository/docker-in-aws/todobackend"
      ]
    }
  ]
}
```

授予特定存储库的 ECR 推送权限

# 来自不同帐户的 ECS 容器实例访问 ECR

在较大的组织中，资源和用户通常分布在多个帐户中，一个常见的模式是拥有一个中央构建帐户，应用程序构件（如 Docker 镜像）在其中进行集中存储。

下图说明了这种情况，您可能有几个帐户运行 ECS 容器实例，这些实例需要拉取存储在您中央存储库中的 Docker 镜像：

![](img/5cf82ce0-1580-4267-aa6d-c6bf66054e75.png)需要访问中央 ECR 存储库的多个帐户

当您需要授予其他帐户对您的 ECR 存储库的访问权限时，需要执行两项配置任务：

1.  在托管存储库的帐户中配置 ECR *资源策略*，允许您定义适用于单个 ECR 存储库（这是*资源*）的策略，并定义*谁*可以访问存储库（例如，AWS 帐户）以及*他们*可以执行的*操作*（例如，登录，推送和/或拉取映像）。定义*谁*可以访问给定存储库的能力是允许通过资源策略启用和控制跨帐户访问的关键。例如，在前面的图中，存储库配置为允许来自帐户`333333444444`和`555555666666`的访问。

1.  远程帐户中的管理员需要以 IAM 策略的形式分配权限，以从您的 ECR 存储库中提取映像。这是一种委托访问的形式，即托管 ECR 存储库的帐户信任远程帐户的访问，只要通过 IAM 策略明确授予了访问权限。例如，在前面的图中，ECS 容器实例分配了一个 IAM 策略，允许它们访问帐户`111111222222`中的 myorg/app-a 存储库。

# 使用 AWS 控制台配置 ECR 资源策略

您可以通过打开适当的 ECR 存储库，在**权限**选项卡中选择**添加**来配置 ECS 控制台中的 ECR 资源策略，并单击**添加**以添加新的权限集：

![](img/4dc40247-49ee-463d-ab22-2800808bfd0d.png)配置 ECR 资源策略

在上图中，请注意您可以通过主体设置将 AWS 帐户 ID 配置为主体，然后通过选择**仅拉取操作**选项轻松允许拉取访问。通过此配置，您允许与远程帐户关联的任何实体从此存储库中拉取 Docker 映像。

![](img/c0f530e0-0975-491a-8e31-390eedb7a484.png)配置 ECR 资源策略

请注意，如果您尝试保存前图和上图中显示的配置，您将收到错误，因为我使用了无效的帐户。假设您使用了有效的帐户 ID 并保存了策略，则将为配置生成以下策略文档：

```
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Sid": "RemoteAccountAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::*<remote-account-id>*:root"
            },
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability"
            ]
        }
    ]
}
```

示例 ECR 存储库策略文档

# 使用 AWS CLI 配置 ECR 资源策略

您可以使用`aws ecr set-repository-policy`命令通过 AWS CLI 配置 ECR 资源策略，如下所示：

```
> aws ecr set-repository-policy --repository-name docker-in-aws/todobackend --policy-text '{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Sid": "RemoteAccountAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::*<remote-account-id>*:root"
            },
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability"
            ]
        }
    ]
}'
```

通过 AWS CLI 配置 ECR 资源策略

如前面的示例所示，您必须使用`--repository-name`标志指定存储库名称，并使用`--policy-text`标志配置存储库策略为 JSON 格式的文档。

# 使用 AWS CloudFormation 配置 ECR 资源策略

在使用 AWS CloudFormation 定义 ECR 存储库时，您可以配置`AWS::ECR::Repository`资源的`RepositoryPolicyText`属性，以定义 ECR 资源策略：

```
AWSTemplateFormatVersion: "2010-09-09"

Description: ECR Repositories

Resources:
  TodobackendRepository:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: docker-in-aws/todobackend
      RepositoryPolicyText:
 Version: "2008-10-17"
 Statement:
 - Sid: RemoteAccountAccess
 Effect: Allow
 Principal:
 AWS: arn:aws:iam::*<remote-account-id>*:root
 Action:
 - ecr:GetDownloadUrlForLayer
 - ecr:BatchGetImage
 - ecr:BatchCheckLayerAvailability
```

使用 AWS CloudFormation 配置 ECR 资源策略

在前面的示例中，策略文本以 YAML 格式表达了您在之前示例中配置的 JSON 策略，并且您可以通过运行`aws cloudformation deploy`命令将更改部署到您的堆栈。

# 配置远程帐户中的 IAM 策略

通过控制台、CLI 或 CloudFormation 配置好 ECR 资源策略后，您可以继续在您的 ECR 资源策略中指定的远程帐户中创建 IAM 策略。这些策略的配置方式与您在本地帐户中配置 IAM 策略的方式完全相同，如果需要，您可以引用远程 ECR 存储库的 ARN，以便仅授予对该存储库的访问权限。

# AWS 服务访问 ECR

我们将讨论的最后一个场景是 AWS 服务访问您的 ECR 镜像的能力。一个例子是 AWS CodeBuild 服务，它使用基于容器的构建代理执行自动化持续集成任务。CodeBuild 允许您定义自己的自定义构建代理，一个常见的做法是将这些构建代理的镜像发布到 ECR 中。这意味着 AWS CodeBuild 服务现在需要访问 ECR，您可以使用 ECR 资源策略来实现这一点。

以下示例扩展了前面的示例，将 AWS CodeBuild 服务添加到资源策略中：

```
AWSTemplateFormatVersion: "2010-09-09"

Description: ECR Repositories

Resources:
  TodobackendRepository:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: docker-in-aws/todobackend
      RepositoryPolicyText:
        Version: "2008-10-17"
        Statement:
          - Sid: RemoteAccountAccess
            Effect: Allow
            Principal:
              AWS: arn:aws:iam::*<remote-account-id>*:root              Service: codebuild.amazonaws.com
            Action:
              - ecr:GetDownloadUrlForLayer
              - ecr:BatchGetImage
              - ecr:BatchCheckLayerAvailability
```

配置 AWS 服务访问 ECR 存储库

在前面的示例中，请注意您可以在`Principal`属性中使用`Service`属性来标识将应用该策略语句的 AWS 服务。在后面的章节中，当您创建自己的自定义 CodeBuild 镜像并发布到 ECR 时，您将看到这一示例的实际操作。

# 配置生命周期策略

如果您在本章中跟随操作，您将已经多次将 todobackend 图像发布到您的 ECR 存储库，并且很可能已经在您的 ECR 存储库中创建了所谓的*孤立图像*。在早期的章节中，我们讨论了在本地 Docker 引擎中创建的孤立图像，并将其定义为其标记已被新图像取代的图像，从而使旧图像无名，并因此“孤立”。

如果您浏览到您的 ECR 存储库并在 ECS 控制台中选择图像选项卡，您可能会注意到您有一些不再具有标记的图像，这是因为您推送了几个带有`latest`标记的图像，这些图像已经取代了现在孤立的图像：

![](img/6396f20d-d78c-4511-ab1e-c3a676b92950.png)孤立的 ECR 图像

在前面的图中，请注意您的 ECR 中的存储使用量现在已经增加了三倍，即使您只有一个当前的`latest`图像，这意味着您可能也要支付三倍的存储成本。当然，您可以手动删除这些图像，但这很容易出错，而且通常会成为一个被遗忘和忽视的任务。

幸运的是，ECR 支持一种称为*生命周期策略*的功能，允许您定义包含在策略中的一组规则，管理您的 Docker 图像的生命周期。您应该始终应用于您创建的每个存储库的生命周期策略的标准用例是定期删除孤立的图像，因此现在让我们看看如何创建和应用这样的策略。

# 使用 AWS 控制台配置生命周期策略

在配置生命周期策略时，因为这些策略可能实际删除您的 Docker 图像，最好始终使用 AWS 控制台来测试您的策略，因为 ECS 控制台包括一个功能，允许您模拟如果应用生命周期策略会发生什么。

使用 AWS 控制台配置生命周期策略，选择 ECR 存储库中的**生命周期规则的干运行**选项卡，然后单击**添加**按钮以创建新的干运行规则。这允许您在不实际删除 ECR 存储库中的任何图像的情况下测试生命周期策略规则。一旦您满意您的规则安全地行为并符合预期，您可以将它们转换为实际的生命周期策略，这些策略将应用于您的存储库：

![](img/b3d8f9fb-8e73-4b0a-9fca-b5e329a297df.png)ECR 干运行规则

您现在可以在“添加规则”屏幕中使用以下参数定义规则：

+   **规则优先级**：确定在策略中定义多个规则时的规则评估顺序。

+   **规则描述**：规则的可读描述。

+   **图像状态**：定义规则适用于哪种类型的图像。请注意，您只能有一个指定**未标记**图像的规则。

+   **匹配条件**：定义规则应何时应用的条件。例如，您可以配置条件以匹配自上次推送到 ECR 存储库以来超过七天的未标记图像。

+   **规则操作**：定义应对匹配规则的图像执行的操作。目前，仅支持**过期**操作，将删除匹配的图像。

单击保存按钮后，新规则将添加到**生命周期规则的模拟运行**选项卡。如果您现在单击**保存并执行模拟运行**按钮，将显示符合规则条件的任何图像，其中应包括先前显示的孤立图像。

现在，取决于您是否有未标记的图像以及它们与您最后推送到存储库的时间相比有多久，您可能会或可能不会看到与您的模拟运行规则匹配的图像。无论实际结果如何，关键在于确保与规则匹配的任何图像都是您期望的，并且您确信模拟运行规则不会意外删除您期望发布和可用的有效图像。

如果您对模拟运行规则满意，接下来可以单击**应用为生命周期策略**按钮，首先会显示对新规则的确认对话框，一旦应用，如果您导航到**生命周期策略**选项卡，您应该会看到您的生命周期策略：

![](img/5b82b40f-e3f7-4792-aece-cc70583abb0a.png)ECR 生命周期策略

要确认您的生命周期策略是否起作用，您可以单击任何策略规则，然后从“操作”下拉菜单中选择**查看历史记录**，这将显示 ECR 执行的与策略规则相关的任何操作。

# 使用 AWS CLI 配置生命周期策略

AWS CLI 支持与通过 AWS 控制台配置 ECR 生命周期策略类似的工作流程，概述如下：

+   `aws ecr start-lifecycle-policy-preview --repository-name <*name*> --lifecycle-policy-text <*json*>`：对存储库启动生命周期策略的模拟运行

+   `aws ecr get-lifecycle-policy-preview --repository-name <*name*>`：获取试运行的状态

+   `aws ecr put-lifecycle-policy --repository-name <*name*> --lifecycle-policy-text <*json*>`：将生命周期策略应用于存储库

+   `aws ecr get-lifecycle-policy --repository-name <*name*>`：显示应用于存储库的当前生命周期策略

+   `aws ecr delete-lifecycle-policy --repository-name <*name*>`：删除应用于存储库的当前生命周期策略

在使用 CLI 时，您需要以 JSON 格式指定生命周期策略，您可以通过单击前面截图中的“查看 JSON”操作来查看示例。

# 使用 AWS CloudFormation 配置生命周期策略

在使用 AWS CloudFormation 定义 ECR 存储库时，您可以配置之前创建的`AWS::ECR::Repository`资源的`LifecyclePolicy`属性，以定义 ECR 生命周期策略：

```
AWSTemplateFormatVersion: "2010-09-09"

Description: ECR Repositories

Resources:
  TodobackendRepository:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: docker-in-aws/todobackend
      LifecyclePolicy:
 LifecyclePolicyText: |
 {
 "rules": [
 {
 "rulePriority": 10,
 "description": "Untagged images",
 "selection": {
 "tagStatus": "untagged",
 "countType": "sinceImagePushed",
 "countUnit": "days",
 "countNumber": 7
 },
 "action": {
```

```
 "type": "expire"
 }
 }
 ]
 }
```

使用 AWS CloudFormation 配置 ECR 生命周期策略

前面示例中的策略文本表示您在之前示例中配置的 JSON 策略作为 JSON 字符串 - 请注意使用管道（`|`）YAML 运算符，它允许您输入多行文本以提高可读性。

有了这个配置，您可以通过运行`aws cloudformation deploy`命令将更改应用到您的堆栈。

# 总结

在本章中，您学习了如何创建和管理 ECR 存储库，您可以使用它来安全和私密地存储您的 Docker 镜像。创建了第一个 ECR 存储库后，您学会了如何使用 AWS CLI 和 Docker 客户端进行 ECR 身份验证，然后成功地给 ECR 打上标签并发布了您的 Docker 镜像。

发布了您的 Docker 镜像后，您还了解了 Docker 客户端可能需要访问存储库的各种情况，包括来自与您的 ECR 存储库相同账户的 ECS 容器实例访问、来自与您的 ECR 存储库不同账户的 ECS 容器实例访问（即跨账户访问），以及最后授予对 AWS 服务（如 CodeBuild）的访问权限。您创建了 ECR 资源策略，这在配置跨账户访问和授予对 AWS 服务的访问权限时是必需的，并且您了解到，尽管在定义远程账户为受信任的中央账户中创建了 ECR 资源策略，但您仍然需要在每个远程账户中创建明确授予对中央账户存储库访问权限的 IAM 策略。

最后，您创建了 ECR 生命周期策略规则，允许您自动定期删除未标记（孤立）的 Docker 镜像，从而有助于减少存储成本。在下一章中，您将学习如何使用一种流行的开源工具 Packer 构建和发布自己的自定义 ECS 容器实例 Amazon Machine Images（AMIs）。

# 问题

1.  您执行哪个命令以获取 ECR 的身份验证令牌？

1.  真/假：ECR 允许您公开发布和分发 Docker 镜像

1.  如果您注意到存储库中有很多未标记的图像，您应该配置哪个 ECR 功能？

1.  真/假：ECR 以压缩格式存储 Docker 镜像

1.  真/假：配置从相同帐户的 ECS 容器实例访问 ECR 需要 ECR 资源策略

1.  真/假：配置从远程帐户的 ECS 容器实例访问 ECR 需要 ECR 资源策略

1.  真/假：配置从 AWS CodeBuild 访问 ECR 需要 ECR 资源策略

1.  真/假：配置从相同帐户的 ECS 容器实例访问 ECR 需要 IAM 策略

1.  真/假：配置从远程帐户的 ECS 容器实例访问 ECR 需要 IAM 策略

# 进一步阅读

您可以查看以下链接以获取有关本章涵盖的主题的更多信息：

+   ECR 用户指南：[`docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html`](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html)

+   ECR 存储库 CloudFormation 资源：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecr-repository.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecr-repository.html)

+   基于身份和基于资源的策略：[`docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_identity-vs-resource.html`](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_identity-vs-resource.html)

+   ECR 存储库的资源级权限：[`docs.aws.amazon.com/AmazonECR/latest/userguide/ecr-supported-iam-actions-resources.html`](https://docs.aws.amazon.com/AmazonECR/latest/userguide/ecr-supported-iam-actions-resources.html)

+   ECR 的生命周期策略：[`docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html`](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html)

+   AWS ECR CLI 参考：[`docs.aws.amazon.com/cli/latest/reference/ecr/index.html#cli-aws-ecr`](https://docs.aws.amazon.com/cli/latest/reference/ecr/index.html#cli-aws-ecr)

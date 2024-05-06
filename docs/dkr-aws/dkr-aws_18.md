# 第十八章：评估

# 第一章，容器和 Docker 基础知识

1.  错误 - Docker 客户端通过 Docker API 进行通信。

1.  错误 - Docker Engine 在 Linux 上本地运行。

1.  错误 - Docker 镜像会发布到 Docker 注册表以供下载。

1.  您需要在**常规**设置下启用**在 tcp://localhost:2375 上公开守护程序而不使用 TLS**设置，并确保在运行 Docker 客户端的任何位置将 DOCKER_HOST 环境变量设置为**localhost:2375**。

1.  正确。

1.  您需要将`USER_BASE/bin`路径添加到您的`PATH`环境变量中。您可以通过运行`python -m site --user-base`命令来确定`USER_BASE`部分。

# 第二章，使用 Docker 构建应用程序

1.  错误 - 您可以使用`FROM`和`AS`指令来定义多阶段 Dockerfiles - 例如，`FROM nginx AS build`。

1.  正确。

1.  正确。

1.  正确。

1.  错误 - 默认情况下，`docker-compose up`命令不会因命令启动的任何服务失败而失败。您可以使用`--exit-code-from`标志指示特定服务失败是否应导致`docker-compose up`命令失败。

1.  正确。

1.  如果要 Docker Compose 等待直到满足 service_healthy 条件，则必须使用`docker-compose up`命令。

1.  您应该使用端口映射只是`8000`。这将创建一个动态端口映射，其中 Docker Engine 将从 Docker Engine 操作系统的临时端口范围中选择一个可用端口。

1.  Makefile 需要使用单个制表符缩进的配方命令。

1.  `$(shell <command>)`函数。

1.  您应该将测试配方添加到`.PHONY`目标，例如`.PHONY: test`。

1.  `build`和`image`属性。

# 第三章，开始使用 AWS

1.  正确。

1.  错误 - 您应该设置一个管理 IAM 用户来执行账户上的管理操作。根帐户只应用于计费或紧急访问。

1.  错误 - AWS 最佳实践是创建定义一组 IAM 权限并适用于一个或多个资源的 IAM 角色。然后，根据您的用例，应授予 IAM 用户/组承担特定角色或一组角色的能力。

1.  AdministratorAccess。

1.  `pip install awscli --user`

1.  错误 - 您必须存储访问密钥 ID 和秘密访问密钥。

1.  在`~/.aws/credentials`文件中。

1.  您需要向配置文件添加`mfa_serial`参数，并为用户指定 MFA 设备的 ARN。

1.  正确。

1.  正确。

1.  不 - CloudFormation 始终尝试成功创建任何新资源，然后删除旧资源。在这种情况下，因为您定义了一个固定的名称值，CloudFormation 将无法创建具有相同名称的新资源。

# 第四章，ECS 简介

1.  ECS 集群，ECS 任务定义和 ECS 服务。

1.  正确。

1.  YAML。

1.  错误 - 当使用静态端口映射时，每个 ECS 容器实例只能有一个给定静态端口映射的实例（假设单个网络接口）。

1.  错误 - ECS CLI 仅建议用于沙箱/测试环境。

1.  您将创建一个 ECS 任务。

1.  错误 - ECS 任务定义是不可变的，给定任务定义的修订版本不能被修改。但是，您可以创建一个给定 ECS 任务定义的新修订版本，该版本基于先前的修订版本但包括更改。

1.  错误 - 您需要运行 `curl localhost:51678/v1/metadata`。

# 第五章，使用 ECR 发布 Docker 镜像

1.  `aws ecr get-login`

1.  错误 - 在撰写本文时，ECR 仅支持私有注册表

1.  ECR 生命周期策略 - 请参阅[`docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html`](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html)

1.  正确

1.  错误 - 您可以同时使用 ECR 资源策略和/或 IAM 策略来配置对来自同一帐户的 ECR 的访问

1.  正确

1.  正确

1.  错误 - 可能（虽然不是最佳实践）使用 ECR 资源策略来授予 IAM 主体访问权限，例如同一帐户中的 IAM 角色

1.  正确 - 您必须在源帐户中配置 ECR 资源策略，并在远程帐户中配置 IAM 策略

# 第六章，构建自定义 ECS 容器实例

1.  `variables`部分。

1.  正确。

1.  JSON。

1.  错误 - 您可以（而且应该）引用环境变量值作为您的 AWS 凭据。

1.  错误 - 您可以使用清单后处理器（[`www.packer.io/docs/post-processors/manifest.html`](https://www.packer.io/docs/post-processors/manifest.html)）来捕获 AMI ID。

1.  默认情况下，将创建一个 8 GB 的操作系统分区和一个 22 GB 的设备映射逻辑卷。

1.  文件提供程序。

1.  云初始化启动脚本可能正在尝试在 EC2 实例上运行软件包更新。如果没有公共互联网连接，这将在长时间超时后失败。

# 第七章，创建 ECS 集群

1.  错误 - EC2 自动缩放组仅支持动态 IP 寻址。

1.  Base64 编码。

1.  使用`AWS::Region`伪参数。

1.  错误 - `Ref`内部函数可以引用 CloudFormation 模板中的资源和参数。

1.  您需要首先运行`cfn-init`来下载 CloudFormation Init 元数据，然后运行`cfn-signal`来通知 CloudFormation 运行`cfn-init`的结果。

1.  您需要确保在 UserData 脚本中将每个实例应该加入的 ECS 集群的名称写入`/etc/ecs/ecs.config` - 例如，`echo "ECS_CLUSTER=<cluster-name>" > /etc/ecs/ecs.config`。

1.  错误 - 此命令仅用于创建堆栈。您应该使用`aws cloudformation deploy`命令根据需要创建和更新堆栈。

1.  每个实例上的 ECS 代理无法与 ECS 服务 API 通信，目前只能作为公共端点使用。

# 第八章，使用 ECS 部署应用程序

1.  正确。

1.  一个监听器。

1.  错误 - 一个目标组只有在关联的应用程序负载均衡监听器创建后才能接受注册。

1.  `AWS::EC2::SecurityGroupIngress`和`AWS::EC2::SecurityGroupEgress`资源。

1.  您应该将短暂容器定义上的`essential`属性标记为`false`。

1.  `DependsOn`参数。

1.  正确。

1.  `CREATE`，`UPDATE`和`DELETE`。

1.  与 Lambda 函数关联的 IAM 角色没有权限为 Lambda 函数日志组创建日志流。

# 第九章，管理秘密

1.  错误 - KMS 服务允许您使用 AWS 创建的密钥以及您自己的私有密钥。

1.  一个 KMS 别名

1.  CloudFormation 导出

1.  错误 - 您可以在可配置的时间内恢复秘密，最长可达 30 天。

1.  AWS CLI 和`jq`实用程序

1.  您必须为用于加密秘密值的 KMS 密钥授予`kms:Decrypt`权限。

1.  `NoEcho`属性

1.  `AWS_DEFAULT_REGION`环境变量

# 第十章，隔离网络访问

1.  正确。

1.  您可以使用`awsvpc`（推荐）或`host`网络模式，确保您的容器将从附加的 EC2 实例弹性网络接口（ENI）获取 IP 地址。

1.  错误 - `awsvpc`网络模式是 ECS 任务网络所必需的。

1.  您需要确保为 ECS 服务配置的安全组允许从负载均衡器访问。

1.  您为 ECS 任务定义启用了 ECS 任务网络，但是您的容器在启动时失败，并显示无法访问位于互联网上的位置的错误。您如何解决这个问题？

1.  两 - 请参阅[`docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI`](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI)。

1.  一 - t2.micro 支持最多两个 ENI，但是一个 ENI 必须保留用于操作系统和 ECS 代理通信。任务网络只允许每个 ENI 一个任务定义。

1.  10 - 鉴于您最多可以在任务网络模式下运行 1 个 ECS 任务定义（请参阅上一个答案），并且您可以在单个 ECS 任务定义中运行多达 10 个容器（请参阅[`docs.aws.amazon.com/AmazonECS/latest/developerguide/service_limits.html`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service_limits.html)）。

1.  在使用 awsvpc 网络模式时，必须使用 IP 目标类型。

1.  您应该从 ECS 服务定义中删除 loadBalancers 属性。

# 第十一章，管理 ECS 基础设施生命周期

1.  错误 - 您负责调用和管理 ECS 容器实例的排水。

1.  `EC2_INSTANCE_LAUNCHING`和`EC2_INSTANCE_TERMINATING`。

1.  `ABANDON`或`CONTINUE`。

1.  错误 - 您可以将生命周期挂钩发布到 SNS、SQS 或 CloudWatch Events。

1.  很可能是您的 Lambda 函数由于达到最大函数执行超时时间（5 分钟）而失败，这意味着生命周期挂钩永远不会完成，并最终超时。您应该确保您的 Lambda 函数在即将达到函数执行超时时间时重新发布生命周期挂钩，这将自动重新调用您的函数。

1.  您应该配置`UpdatePolicy`属性。

1.  将`MinSuccessfulInstancesPercent`属性设置为 100。

1.  一个 Lambda 权限。

# 第十二章，ECS 自动缩放

1.  错误 - 您负责自动扩展您的 ECS 容器实例。

1.  EC2 自动缩放。

1.  应用自动缩放。

1.  使用值 300 配置`memoryReservation`参数，并使用值 1,024 配置`memory`参数。

1.  将 ECS 容器实例的 CPU 单位分配均匀分配到每个 ECS 任务中（即，将每个任务配置为分配 333 个单位的 CPU）。

1.  正确。

1.  三。

1.  在滚动更新期间，您应该禁用自动缩放。您可以通过配置 CloudFormation `UpdatePolicy`属性的`AutoScalingRollingUpdate.SuspendProcesses`属性来实现这一点。

1.  零任务 - 根据集群的当前状态，每个实例上都运行一个 ECS 任务。鉴于每个任务都有一个静态端口映射到 TCP 端口`80`，您无法安排另一个任务，因为所有端口都在使用中。

1.  四 - 您应该使用每个容器 500 MB 内存的最坏情况。

# 第十三章，持续交付 ECS 应用程序

1.  `buildspec.yml`

1.  错误 - CodeBuild 使用容器，并包含自己的代理来运行构建脚本

1.  Docker 中的 Docker

1.  CloudFormation 变更集

1.  cloudformation.amazonaws.com

1.  在尝试推送映像之前，请确保您的构建脚本登录 ECR

1.  允许`codebuild.amazonaws.com`服务主体对存储库进行拉取访问

1.  确保容器正在以特权标志运行

# 第十四章，Fargate 和 ECS 服务发现

1.  正确。

1.  仅支持`awsvpc`网络模式。

1.  错误 - 您必须确保 ECS 代理可以通过分配给 Fargate ECS 任务的 ENI 进行通信。

1.  您需要确保任务定义的 ExecutionRoleArn 属性引用的 IAM 角色允许访问 ECR 存储库。

1.  不 - Fargate 仅支持 CloudWatch 日志。

1.  错误 - ECS 服务发现使用 Route53 区域发布服务注册信息。

1.  服务发现命名空间。

1.  在配置 Fargate ECS 任务定义时，必须配置受支持的 CPU/内存配置。有关受支持的配置，请参阅[`docs.aws.amazon.com/AmazonECS/latest/developerguide/task-cpu-memory-error.html`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-cpu-memory-error.html)。

1.  UDP 端口`2000`。

1.  错误 - 跟踪必须发布到在您的环境中运行的 X-Ray 守护程序。

# 第十五章，弹性 Beanstalk

1.  错误 - Elastic Beanstalk 支持单容器和多容器 Docker 应用程序

1.  `Dockerrun.aws.json`文件。

1.  正确。

1.  向用于 Elastic Beanstalk EC2 实例的虚拟机实例角色添加 IAM 权限以拉取 ECR 映像。

1.  错误 - Elastic Beanstalk 使用绑定挂载卷，这会分配 root:root 权限，导致非 root 容器在写入卷时失败。

1.  错误 - 您可以将`leader_only`属性设置为`true`，以便在`container_commands`键中仅在一个 Elastic Beanstalk 实例上运行命令。

1.  错误 - `eb ssh`命令用于建立对 Elastic Beanstalk EC2 实例的 SSH 访问。

1.  正确。

# 第十六章，Docker Swarm 在 AWS 中

1.  正确。

1.  `docker service create`

1.  错误 - Docker Swarm 包括两种节点类型：主节点和从节点。

1.  错误 - Docker for AWS 与经典的 AWS 弹性负载均衡器集成。

1.  错误 - 当后端设置为可重定位时，Cloudstore AWS 卷插件会创建一个基于 EBS 的卷。

1.  错误 - 因为 EBS 卷位于不同的可用区，将首先创建原始卷的快照，然后从快照创建新卷，然后将其附加到新的数据库服务容器。

1.  `--with-registry-auth`

1.  您需要安装一个系统组件，该组件将定期自动刷新 Docker 凭据，例如[`github.com/mRoca/docker-swarm-aws-ecr-auth`](https://github.com/mRoca/docker-swarm-aws-ecr-auth)。

1.  版本 3。

1.  错误 - 您应该将重启策略配置为`never`或`on-failure`。

# 第十七章，弹性 Kubernetes 服务

1.  正确 - 适用于 Docker CE 18.06 及更高版本

1.  在`args`属性中定义自定义命令字符串（这相当于 Dockerfile 中的 CMD 指令）

1.  错误 - Kubernetes 包括两种节点类型：管理器和工作节点

1.  错误 - 在撰写时，Kubernetes 支持与经典弹性负载均衡器的集成

1.  错误

1.  kube-proxy

1.  正确

1.  一个服务

1.  一个作业

1.  错误 - EKS 管理 Kubernetes 管理节点

1.  无 - EKS 中没有默认存储类，您必须创建自己的存储类

1.  在 pod 中像 initContainer 一样定义任务

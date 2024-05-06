# 第八章：使用 ECS 部署应用程序

在上一章中，您学习了如何使用 EC2 自动扩展组在 AWS 中配置和部署 ECS 集群，本章的目标是使用 CloudFormation 将 ECS 应用程序部署到您新建的 ECS 集群。

您将首先开始学习如何定义和部署通常在生产环境中 ECS 应用程序中所需的各种支持资源。这些资源包括创建应用程序数据库以存储应用程序的数据，部署应用程序负载均衡器以服务和负载均衡对应用程序的请求，以及配置其他资源，例如 IAM 角色和安全组，以控制对应用程序的访问和从应用程序的访问。

有了这些支持资源，您将继续创建 ECS 任务定义，定义容器的运行时配置，然后配置 ECS 服务，将 ECS 任务定义部署到 ECS 集群，并与应用程序负载均衡器集成，以管理滚动部署等功能。最后，您将学习如何创建 CloudFormation 自定义资源，执行自定义的配置任务，例如运行数据库迁移，为您提供基于 AWS CloudFormation 的完整应用程序部署框架。

将涵盖以下主题：

+   使用 RDS 创建应用程序数据库

+   配置应用程序负载均衡器

+   创建 ECS 任务定义

+   部署 ECS 服务

+   ECS 滚动部署

+   创建 CloudFormation 自定义资源

# 技术要求

以下列出了完成本章所需的技术要求：

+   AWS 账户的管理员访问权限

+   本地 AWS 配置文件按第三章的说明配置

+   AWS CLI

+   本章将继续自第七章开始，因此需要您成功完成那里定义的所有配置任务

以下 GitHub URL 包含本章中使用的代码示例：[`github.com/docker-in-aws/docker-in-aws/tree/master/ch8`](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch8)[.](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch4)

查看以下视频以查看代码的实际操作：

[`bit.ly/2Mx8wHX`](http://bit.ly/2Mx8wHX)

# 使用 RDS 创建应用程序数据库

示例 todobackend 应用程序包括一个 MySQL 数据库，用于持久化通过应用程序 API 创建的待办事项。当您在第一章首次设置和运行示例应用程序时，您使用 Docker 容器提供应用程序数据库，但是在生产级环境中，通常认为最佳做法是在专门为数据库和数据访问操作进行了优化的专用机器上运行数据库和其他提供持久性存储的服务。AWS 中的一个这样的服务是关系数据库服务（RDS），它提供了专用的托管实例，针对提供流行的关系数据库引擎进行了优化，包括 MySQL、Postgres、SQL Server 和 Oracle。RDS 是一个非常成熟和强大的服务，非常常用于支持在 AWS 中运行的 ECS 和其他应用程序的数据库需求。

可以使用 CloudFormation 配置 RDS 实例。要开始，让我们在您的 todobackend CloudFormation 模板中定义一个名为`ApplicationDatabase`的新资源，其资源类型为`AWS::RDS::DBInstance`，如下例所示：

```
AWSTemplateFormatVersion: "2010-09-09"

Description: Todobackend Application

Parameters:
  ApplicationDesiredCount:
    Type: Number
    Description: Desired EC2 instance count
  ApplicationImageId:
    Type: String
    Description: ECS Amazon Machine Image (AMI) ID
  ApplicationSubnets:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Target subnets for EC2 instances
  DatabasePassword:
 Type: String
 Description: Database password
 NoEcho: "true"
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: Target VPC

Resources:
  ApplicationDatabase:
 Type: AWS::RDS::DBInstance
 Properties:
 Engine: MySQL
 EngineVersion: 5.7
 DBInstanceClass: db.t2.micro
 AllocatedStorage: 10
 StorageType: gp2
 MasterUsername: todobackend
 MasterUserPassword: !Ref DatabasePassword
 DBName: todobackend
 VPCSecurityGroups:
 - !Ref ApplicationDatabaseSecurityGroup
 DBSubnetGroupName: !Ref ApplicationDatabaseSubnetGroup
 MultiAZ: "false"
 AvailabilityZone: !Sub ${AWS::Region}a
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName}-db  ApplicationAutoscalingSecurityGroup:
    Type: AWS::EC2::SecurityGroup
...
...
```

创建 RDS 资源

前面示例中的配置被认为是定义 RDS 实例的最小配置，如下所述：

+   `Engine`和`EngineVersion`：数据库引擎，在本例中是 MySQL，以及要部署的主要或次要版本。

+   `DBInstanceClass`：用于运行数据库的 RDS 实例类型。为了确保您有资格获得免费使用，您可以将其硬编码为`db.t2.micro`，尽管在生产环境中，您通常会将此属性参数化为更大的实例大小。

+   `AllocatedStorage`和`StorageType`：定义以 GB 为单位的存储量和存储类型。在第一个示例中，存储类型设置为 10GB 的基于 SSD 的 gp2（通用用途 2）存储。

+   `MasterUsername`和`MasterUserPassword`：指定为 RDS 实例配置的主用户名和密码。`MasterUserPassword`属性引用了一个名为`DatabasePassword`的输入参数，其中包括一个名为`NoEcho`的属性，确保 CloudFormation 不会在任何日志中打印出此参数的值。

+   `DBName`：指定数据库的名称。

+   `VPCSecurityGroups`：要应用于 RDS 实例的网络通信入口和出口的安全组列表。

+   `DBSubnetGroupName`：引用`AWS::RDS::DBSubnetGroup`类型的资源，该资源定义 RDS 实例可以部署到的子网。请注意，即使您只配置了单可用区 RDS 实例，您仍然需要引用您创建的数据库子网组资源中的至少两个子网。在前面的例子中，您引用了一个名为`ApplicationDatabaseSubnetGroup`的资源，稍后将创建该资源。

+   `MultiAZ`：定义是否在高可用的多可用区配置中部署 RDS 实例。对于演示应用程序，可以将此设置配置为`false`，但在实际应用程序中，您通常会将此设置配置为`true`，至少对于生产环境是这样。

+   `AvailabilityZone`：定义 RDS 实例将部署到的可用区。此设置仅适用于单可用区实例（即`MultiAZ`设置为 false 的实例）。在前面的例子中，您使用`AWS::Region`伪参数来引用本地区域中可用区`a`。

# 配置支持的 RDS 资源

回顾前面的例子，很明显您需要配置至少两个额外的支持资源用于 RDS 实例：

+   `ApplicationDatabaseSecurityGroup`：定义应用于 RDS 实例的入站和出站安全规则的安全组资源。

+   `ApplicationDatabaseSubnetGroup`：RDS 实例可以部署到的子网列表。

除了这些资源，以下示例还演示了我们还需要添加一些资源：

```
...

Resources:
  ApplicationDatabase:
    Type: AWS::RDS::DBInstance
    Properties:
      Engine: MySQL
      EngineVersion: 5.7
      DBInstanceClass: db.t2.micro
      AllocatedStorage: 10
      StorageType: gp2
      MasterUsername: todobackend
      MasterUserPassword:
        Ref: DatabasePassword
      DBName: todobackend
      VPCSecurityGroups:
        - !Ref ApplicationDatabaseSecurityGroup
      DBSubnetGroupName: !Ref ApplicationDatabaseSubnetGroup
      MultiAZ: "false"
      AvailabilityZone: !Sub ${AWS::Region}a
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName}-db
 ApplicationDatabaseSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Application Database Subnet Group
      SubnetIds: !Ref ApplicationSubnets
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName}-db-subnet-group
  ApplicationDatabaseSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: !Sub ${AWS::StackName} Application Database Security Group
      VpcId: !Ref VpcId
      SecurityGroupEgress:
        - IpProtocol: icmp
          FromPort: -1
          ToPort: -1
          CidrIp: 192.0.2.0/32
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName}-db-sg
  ApplicationToApplicationDatabaseIngress:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      IpProtocol: tcp
      FromPort: 3306
      ToPort: 3306
      GroupId: !Ref ApplicationDatabaseSecurityGroup
      SourceSecurityGroupId: !Ref ApplicationAutoscalingSecurityGroup
  ApplicationToApplicationDatabaseEgress:
    Type: AWS::EC2::SecurityGroupEgress
    Properties:
      IpProtocol: tcp
      FromPort: 3306
      ToPort: 3306
      GroupId: !Ref ApplicationAutoscalingSecurityGroup
      DestinationSecurityGroupId: !Ref ApplicationDatabaseSecurityGroup
...
...
```

创建支持的 RDS 资源

在前面的例子中，您首先创建了数据库子网组资源，其中 SubnetIds 属性引用了您在第七章中创建的相同的`ApplicationSubnets`列表参数，这意味着您的数据库实例将安装在与应用程序 ECS 集群和 EC2 自动扩展组实例相同的子网中。在生产应用程序中，您通常会在单独的专用子网上运行 RDS 实例，理想情况下，出于安全目的，该子网不会连接到互联网，但出于简化示例的目的，我们将利用与应用程序 ECS 集群相同的子网。

接下来，您创建了一个名为`ApplicationDatabaseSecurityGroup`的安全组资源，并注意到它只包含一个出站规则，有点奇怪的是允许对 IP 地址`192.0.2.0/32`进行 ICMP 访问。这个 IP 地址是"TEST-NET" IP 地址范围的一部分，是互联网上的无效 IP 地址，用于示例代码和文档。包含这个作为出站规则的原因是，AWS 默认情况下会自动应用一个允许任何规则的出站规则，除非您明确覆盖这些规则，因此通过添加一个允许访问无法路由的 IP 地址的规则，您实际上阻止了 RDS 实例发起的任何出站通信。

最后，请注意，您创建了两个与安全组相关的资源，`ApplicationToApplicationDatabaseIngress`和`ApplicationToApplicationDatabaseEgress`，它们分别具有`AWS::EC2::SecurityGroupIngress`和`AWS::EC2::SecurityGroupEgress`的资源类型。这些特殊资源避免了在 CloudFormation 中出现的一个问题，即创建了两个需要相互引用的资源之间的循环依赖。在我们的具体场景中，我们希望允许`ApplicationAutoscalingSecurityGroup`的成员访问`ApplicationDatabaseSecurityGroup`的成员，并应用适当的安全规则，从应用程序数据库中进行入站访问，并从应用程序实例中进行出站访问。如果您尝试按照以下图表所示的规则进行配置，CloudFormation 将抛出错误并检测到循环依赖。

![](img/0139e646-f450-4086-99aa-09fc2e454a4a.png)CloudFormation 循环依赖

为了解决这个问题，以下图表演示了一种替代方法，使用了您在上一个示例中创建的资源。

`ApplicationToApplicationDatabaseIngress`资源将动态创建`ApplicationDatabaseSecurityGroup`中的入口规则（由`GroupId`属性指定），允许从`ApplicationAutoscalingSecurityGroup`（由`SourceSecurityGroupId`属性指定）访问 MySQL 端口（TCP/3306）。同样，`ApplicationToApplicationDatabaseEgress`资源将动态创建`ApplicationAutoscalingSecurityGroup`中的出口规则（由`GroupId`属性指定），允许访问属于`ApplicationDatabaseSecurityGroup`的实例的 MySQL 端口（TCP/3306）（由`DestinationSecurityGroupId`属性指定）。这最终实现了前面图表中所示配置的意图，但不会在 CloudFormation 中引起任何循环依赖错误。

![](img/4abce996-823e-4813-a977-7975e8894666.png)解决 CloudFormation 循环依赖

# 使用 CloudFormation 部署 RDS 资源

在上述示例的配置完成后，您现在可以实际更新 CloudFormation 堆栈，其中将添加 RDS 实例和其他支持资源。在执行此操作之前，您需要更新第七章中创建的`dev.cfg`文件，该文件为您的 CloudFormation 堆栈提供了环境特定的输入参数值。具体来说，您需要为`MasterPassword`参数指定一个值，如下例所示：

```
ApplicationDesiredCount=1
ApplicationImageId=ami-ec957491
ApplicationSubnets=subnet-a5d3ecee,subnet-324e246f
DatabasePassword=my-super-secret-password
VpcId=vpc-f8233a80
```

向 dev.cfg 文件添加数据库密码

此时，如果您对于以明文提供最终将提交到源代码中的密码感到担忧，那么恭喜您，您对于这种方法感到非常担忧是完全正确的。在接下来的章节中，我们将专门讨论如何安全地管理凭据，但目前我们不会解决这个问题，因此请记住，上述示例中演示的方法并不被认为是最佳实践，我们只会暂时保留这个方法来使您的应用数据库实例正常运行。

在上述示例的配置完成后，您现在可以使用在第七章中使用过的`aws cloudformation deploy`命令来部署更新后的堆栈。

```
> export AWS_PROFILE=docker-in-aws
> aws cloudformation deploy --template-file stack.yml \
 --stack-name todobackend --parameter-overrides $(cat dev.cfg) \
 --capabilities CAPABILITY_NAMED_IAM
Enter MFA code for arn:aws:iam::385605022855:mfa/justin.menga:
```

```
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - todobackend
> aws cloudformation describe-stack-resource --stack-name todobackend \
    --logical-resource-id ApplicationDatabase
{
    "StackResourceDetail": {
        "StackName": "todobackend",
        "StackId": "arn:aws:cloudformation:us-east-1:385605022855:stack/todobackend/297933f0-37fe-11e8-82e0-503f23fb55fe",
        "LogicalResourceId": "ApplicationDatabase",
 "PhysicalResourceId": "ta10udhxgd7s4gf",
        "ResourceType": "AWS::RDS::DBInstance",
        "LastUpdatedTimestamp": "2018-04-04T12:12:13.265Z",
        "ResourceStatus": "CREATE_COMPLETE",
        "Metadata": "{}"
    }
}
> aws rds describe-db-instances --db-instance-identifier ta10udhxgd7s4gf
{
    "DBInstances": [
        {
            "DBInstanceIdentifier": "ta10udhxgd7s4gf",
            "DBInstanceClass": "db.t2.micro",
            "Engine": "mysql",
            "DBInstanceStatus": "available",
            "MasterUsername": "todobackend",
            "DBName": "todobackend",
            "Endpoint": {
                "Address": "ta10udhxgd7s4gf.cz8cu8hmqtu1.us-east-1.rds.amazonaws.com",
                "Port": 3306,
                "HostedZoneId": "Z2R2ITUGPM61AM"
            }
...
...
```

使用 RDS 资源更新 CloudFormation 堆栈

部署将需要一些时间（通常为 15-20 分钟）才能完成，一旦部署完成，请注意您可以使用`aws cloudformation describe-stack-resource`命令获取有关`ApplicationDatabase`资源的更多信息，包括`PhysicalResourceId`属性，该属性指定了 RDS 实例标识符。

# 配置应用负载均衡器

我们已经建立了一个 ECS 集群并创建了一个应用程序数据库来存储应用程序数据，接下来我们需要创建前端基础设施，以服务于外部世界对我们的 Docker 应用程序的连接。

在 AWS 中提供这种基础设施的一种流行方法是利用弹性负载均衡服务，该服务提供了多种不同的选项，用于负载均衡连接到您的应用程序：

+   **经典弹性负载均衡器**：原始的 AWS 负载均衡器，支持第 4 层（TCP）负载均衡。一般来说，您应该使用较新的应用负载均衡器或网络负载均衡器，它们共同提供了经典负载均衡器的所有现有功能以及更多功能。

+   **应用负载均衡器**：一种特别针对基于 Web 的应用程序和 API 的 HTTP 感知负载均衡器。

+   **网络负载均衡器**：高性能的第 4 层（TCP）负载均衡服务，通常用于非 HTTP 基于 TCP 的应用程序，或者需要非常高性能的应用程序。

对于我们的目的，我们将利用应用负载均衡器（ALB），这是一个现代的第 7 层负载均衡器，可以根据 HTTP 协议信息执行高级操作，例如基于主机头和基于路径的路由。例如，ALB 可以将针对特定 HTTP 主机头的请求路由到一组特定的目标，并且还可以将针对 some.domain/foo 路径的请求路由到一组目标，将针对 some.domain/bar 路径的请求路由到另一组目标。

AWS ALB 与弹性容器服务集成，支持许多关键的集成功能：

+   **滚动更新**：ECS 服务可以以滚动方式部署，ECS 利用负载均衡器连接排空来优雅地将旧版本的应用程序停止服务，终止并替换每个应用程序容器为新版本，然后将新容器添加到负载均衡器，确保更新在没有最终用户中断或影响的情况下进行。

+   **动态端口映射**：此功能允许您将容器端口映射到 ECS 容器实例上的动态端口，ECS 负责确保动态端口映射正确地注册到应用负载均衡器。动态端口映射的主要好处是它允许同一应用程序容器的多个实例在单个 ECS 容器实例上运行，从而在维度和扩展 ECS 集群方面提供了更大的灵活性。

+   **健康检查**：ECS 使用应用负载均衡器的健康检查来确定您的 Docker 应用程序的健康状况，自动终止和替换任何可能变得不健康并且无法通过负载均衡器健康检查的容器。

# 应用负载均衡器架构

如果您熟悉旧版经典弹性负载均衡器，您会发现新版应用负载均衡器的架构更加复杂，因为 ALB 支持高级的第 7 层/HTTP 功能。

以下图显示了组成应用负载均衡器的各种组件：

应用负载均衡器组件

以下描述了上图中所示的每个组件：

+   **应用负载均衡器**：应用负载均衡器是定义负载均衡器的物理资源，例如负载均衡器应该运行在哪些子网以及允许或拒绝网络流量到负载均衡器或从负载均衡器流出的安全组。

+   **监听器**：监听器定义了终端用户和设备连接的网络端口。您可以将监听器视为负载均衡器的前端组件，为传入连接提供服务，最终将被路由到托管应用程序的目标组。每个应用负载均衡器可以包括多个监听器——一个常见的例子是监听器配置，可以为端口`80`和端口`443`的网络流量提供服务。

+   **监听规则**：监听规则可选择性地根据接收到的主机标头和/或请求路径的值将由监听器接收的 HTTP 流量路由到不同的目标组。例如，如前图所示，您可以将发送到`/foo/*`请求路径的所有流量路由到一个目标组，而将发送到`/bar/*`的所有流量路由到另一个目标组。请注意，每个监听器必须定义一个默认目标组，所有未路由到监听规则的流量将被路由到该目标组。

+   **目标组**：目标组定义了应该路由到的一个或多个目标的传入连接。您可以将目标组视为负载均衡器的后端组件，负责将接收到的连接负载均衡到目标组中的成员。在将应用程序负载均衡器与 ECS 集成时，目标组链接到 ECS 服务，每个 ECS 服务实例（即容器）被视为单个目标。

# 配置应用程序负载均衡器

现在您已经了解了应用程序负载均衡器的基本架构，让我们在您的 CloudFormation 模板中定义各种应用程序负载均衡器组件，并继续将新资源部署到您的 CloudFormation 堆栈中。

# 创建应用程序负载均衡器

以下示例演示了如何添加一个名为`ApplicationLoadBalancer`的资源，正如其名称所示，它配置了基本的应用程序负载均衡器资源：

```
...
...
Resources:
 ApplicationLoadBalancer:
 Type: AWS::ElasticLoadBalancingV2::LoadBalancer
 Properties:
 Scheme: internet-facing
 Subnets: !Ref ApplicationSubnets
 SecurityGroups:
 - !Ref ApplicationLoadBalancerSecurityGroup
 LoadBalancerAttributes:
 - Key: idle_timeout.timeout_seconds
 Value : 30
 Tags:
 - Key: Name
 Value: !Sub ${AWS::StackName}-alb
  ApplicationDatabase:
    Type: AWS::RDS::DBInstance
...
...
```

创建应用程序负载均衡器

在上述示例中，为应用程序负载均衡器资源配置了以下属性：

+   `方案`：定义负载均衡器是否具有公共 IP 地址（由值`internet-facing`指定）或仅具有私有 IP 地址（由值`internal`指定）

+   `子网`：定义了应用程序负载均衡器端点将部署到的子网。在上述示例中，您引用了`ApplicationSubnets`输入参数，该参数之前已用于 EC2 自动扩展组和 RDS 数据库实例资源。

+   `安全组`：指定要应用于负载均衡器的安全组列表，限制入站和出站网络流量。您引用了一个名为`ApplicationLoadBalancerSecurityGroup`的安全组，稍后将创建该安全组。

+   `LoadBalancerAttributes`：以键/值格式配置应用程序负载均衡器的各种属性。您可以在[`docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html#load-balancer-attributes`](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html#load-balancer-attributes)找到支持的属性列表，在前面的示例中，您配置了一个属性，将空闲连接超时从默认值`60`秒减少到`30`秒。

CloudFormation 的一个特性是能够定义自己的*输出*，这些输出可用于提供有关堆栈中资源的信息。您可以为堆栈配置一个有用的输出，即应用程序负载均衡器端点的公共 DNS 名称的值，因为这是负载均衡器提供的任何应用程序发布的地方：

```
...
...
Resources:
  ...
  ...
Outputs:
 PublicURL:
 Description: Public DNS name of Application Load Balancer
 Value: !Sub ${ApplicationLoadBalancer.DNSName}

```

配置 CloudFormation 输出

在前面的例子中，请注意`ApplicationLoadBalancer`资源输出一个名为`DNSName`的属性，该属性返回`ApplicationLoadBalancer`资源的公共 DNS 名称。

# 配置应用程序负载均衡器安全组

在前面的例子中，您引用了一个名为`ApplicationLoadBalancerSecurityGroup`的资源，该资源定义了对应用程序负载均衡器的入站和出站网络访问。

除了这个资源，您还需要以类似的方式创建`AWS::EC2::SecurityGroupIngress`和`AWS::EC2::SecurityGroupEgress`资源，这些资源确保应用程序负载均衡器可以与您的 ECS 服务应用程序实例通信：

```
...
...
Resources:
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internet-facing
      Subnets: !Ref ApplicationSubnets
      SecurityGroups:
        - !Ref ApplicationLoadBalancerSecurityGroup
      LoadBalancerAttributes:
        - Key: idle_timeout.timeout_seconds
          Value : 30
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName}-alb
  ApplicationLoadBalancerSecurityGroup:
 Type: AWS::EC2::SecurityGroup
 Properties:
 GroupDescription: Application Load Balancer Security Group
 VpcId: !Ref VpcId
 SecurityGroupIngress:
 - IpProtocol: tcp
 FromPort: 80
 ToPort: 80
 CidrIp: 0.0.0.0/0
 Tags:
 - Key: Name
 Value: 
 Fn::Sub: ${AWS::StackName}-alb-sg  ApplicationLoadBalancerToApplicationIngress:
 Type: AWS::EC2::SecurityGroupIngress
 Properties:
 IpProtocol: tcp
 FromPort: 32768
 ToPort: 60999
 GroupId: !Ref ApplicationAutoscalingSecurityGroup
 SourceSecurityGroupId: !Ref ApplicationLoadBalancerSecurityGroup
 ApplicationLoadBalancerToApplicationEgress:
 Type: AWS::EC2::SecurityGroupEgress
 Properties:
 IpProtocol: tcp
 FromPort: 32768
 ToPort: 60999
 GroupId: !Ref ApplicationLoadBalancerSecurityGroup
 DestinationSecurityGroupId: !Ref ApplicationAutoscalingSecurityGroup
  ApplicationDatabase:
    Type: AWS::RDS::DBInstance
...
...
```

配置应用程序负载均衡器安全组资源

在前面的例子中，您首先创建了`ApplicationLoadBalancerSecurityGroup`资源，允许从互联网访问端口 80。`ApplicationLoadBalancerToApplicationIngress`和`ApplicationLoadBalancerToApplicationEgress`资源向`ApplicationLoadBalancerSecurityGroup`和`ApplicationAutoscalingSecurityGroup`资源添加安全规则，而不会创建循环依赖（请参阅前面的图表和相关描述），请注意这些规则引用了应用程序自动缩放组的短暂端口范围`32768`到`60999`，因为我们将为您的 ECS 服务配置动态端口映射。

# 创建一个监听器

现在，您已经建立了基本的应用程序负载均衡器和相关的安全组资源，可以为应用程序负载均衡器配置一个监听器。对于本书的目的，您只需要配置一个支持 HTTP 连接的单个监听器，但在任何真实的生产用例中，您通常会为任何面向互联网的服务配置 HTTPS 监听器以及相关证书。

以下示例演示了配置一个支持通过端口`80`（HTTP）访问应用程序负载均衡器的单个监听器：

```
...
...
Resources:
  ApplicationLoadBalancerHttpListener:
 Type: AWS::ElasticLoadBalancingV2::Listener
 Properties:
 LoadBalancerArn: !Ref ApplicationLoadBalancer
 Protocol: HTTP
 Port: 80
 DefaultActions:
 - TargetGroupArn: !Ref ApplicationServiceTargetGroup
 Type: forward
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internet-facing
      Subnets: !Ref ApplicationSubnets
      SecurityGroups:
        - !Ref ApplicationLoadBalancerSecurityGroup
      LoadBalancerAttributes:
        - Key: idle_timeout.timeout_seconds
          Value : 30
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName}-alb
...
...
```

创建应用程序负载均衡器监听器

在上面的示例中，通过`LoadBalancerArn`属性将监听器绑定到`ApplicationLoadBalancer`资源，`Protocol`和`Port`属性配置监听器以期望在端口`80`上接收传入的 HTTP 连接。请注意，您必须定义`DefaultActions`属性，该属性定义了传入连接将被转发到的默认目标组。

# 创建目标组

与配置应用程序负载均衡器相关的最终配置任务是配置目标组，该目标组将用于将监听器资源接收的传入请求转发到应用程序实例。

以下示例演示了配置目标组资源：

```
...
...
Resources:
  ApplicationServiceTargetGroup:
 Type: AWS::ElasticLoadBalancingV2::TargetGroup
 Properties:
 Protocol: HTTP
 Port: 8000
 VpcId: !Ref VpcId
 TargetGroupAttributes:
 - Key: deregistration_delay.timeout_seconds
 Value: 30
  ApplicationLoadBalancerHttpListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Protocol: HTTP
      Port: 80
      DefaultActions:
        - TargetGroupArn: !Ref ApplicationServiceTargetGroup
          Type: forward
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
...
...
```

创建目标组

在上面的示例中，为目标组定义了以下配置：

+   `Protocol`：定义将转发到目标组的连接的协议。

+   `Port`：指定应用程序将运行的容器端口。默认情况下，todobackend 示例应用程序在端口`8000`上运行，因此您可以为端口配置此值。请注意，当配置动态端口映射时，ECS 将动态重新配置此端口。

+   `VpcId`：配置目标所在的 VPC ID。

+   `TargetGroupAttributes`：定义了目标组的配置属性（[`docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-group-attributes`](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-group-attributes)），在上面的示例中，`deregistration_delay.timeout_seconds`属性配置了在滚动部署应用程序期间排空连接时等待取消注册目标的时间。

# 使用 CloudFormation 部署应用负载均衡器

现在，您的 CloudFormation 模板中已经定义了所有应用负载均衡器组件，您可以使用`aws cloudformation deploy`命令将这些组件部署到 AWS。

一旦您的堆栈部署完成，如果您打开 AWS 控制台并导航到 EC2 仪表板，在**负载均衡**部分，您应该能够看到您的新应用负载均衡器资源。

以下截图演示了查看作为部署的一部分创建的应用负载均衡器资源：

查看应用负载均衡器

在前面的截图中，您可以看到应用负载均衡器资源有一个 DNS 名称，这是您的最终用户和设备在访问负载均衡器后面的应用时需要连接的端点名称。一旦您完全部署了堆栈中的所有资源，您将在稍后使用这个名称，但是现在因为您的目标组是空的，这个 URL 将返回一个 503 错误，如下例所示。请注意，您可以通过单击前面截图中的**监听器**选项卡来查看您的监听器资源，您可以通过单击左侧菜单上的**目标组**链接来查看您的关联目标组资源。

您会注意到应用负载均衡器的 DNS 名称并不是您的最终用户能够识别或记住的友好名称。在实际应用中，您通常会创建一个 CNAME 或 ALIAS DNS 记录，配置一个友好的规范名称，比如 example.com，指向您的负载均衡器 DNS 名称。有关如何执行此操作的更多详细信息，请参阅[`docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html`](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html)，并注意您可以并且应该使用 CloudFormation 创建 CNAME 和 ALIAS 记录([`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/quickref-route53.html#scenario-recordsetgroup-zoneapex`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/quickref-route53.html#scenario-recordsetgroup-zoneapex))。

```
> aws cloudformation describe-stacks --stack-name todobackend --query Stacks[].Outputs[]
[
    {
        "OutputKey": "PublicURL",
        "OutputValue": "todob-Appli-5SV5J3NC6AAI-2078461159.us-east-1.elb.amazonaws.com",
        "Description": "Public DNS name of Application Load Balancer"
    }
]
> curl todob-Appli-5SV5J3NC6AAI-2078461159.us-east-1.elb.amazonaws.com
<html>
<head><title>503 Service Temporarily Unavailable</title></head>
<body bgcolor="white">
<center><h1>503 Service Temporarily Unavailable</h1></center>
</body>
</html>
```

测试应用负载均衡器端点

请注意，在上面的示例中，您可以使用 AWS CLI 来查询 CloudFormation 堆栈的输出，并获取应用程序负载均衡器的公共 DNS 名称。您还可以在 CloudFormation 仪表板中选择堆栈后，单击“输出”选项卡来查看堆栈的输出。

# 创建 ECS 任务定义

您现在已经达到了定义使用 CloudFormation 的 ECS 集群并创建了许多支持资源的阶段，包括用于应用程序数据库的 RDS 实例和用于服务应用程序连接的应用程序负载均衡器。

在这个阶段，您已经准备好创建将代表您的应用程序的 ECS 资源，包括 ECS 任务定义和 ECS 服务。

我们将从在 CloudFormation 模板中定义 ECS 任务定义开始，如下例所示：

```
Parameters:
  ...
  ...
  ApplicationImageId:
    Type: String
    Description: ECS Amazon Machine Image (AMI) ID
 ApplicationImageTag:
 Type: String
 Description: Application Docker Image Tag
 Default: latest  ApplicationSubnets:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Target subnets for EC2 instances
 ...
  ... 
Resources:
  ApplicationTaskDefinition:
 Type: AWS::ECS::TaskDefinition
 Properties:
 Family: todobackend      Volumes:
 - Name: public          Host:
 SourcePath: /data/public
 ContainerDefinitions:        - Name: todobackend
 Image: !Sub ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/docker-in-aws/todobackend:${ApplicationImageTag}
 MemoryReservation: 395
 Cpu: 245
 MountPoints:
 - SourceVolume: public
 ContainerPath: /public
 Environment:
            - Name: DJANGO_SETTINGS_MODULE
 Value: todobackend.settings_release
 - Name: MYSQL_HOST
 Value: !Sub ${ApplicationDatabase.Endpoint.Address}
 - Name: MYSQL_USER
 Value: todobackend
 - Name: MYSQL_PASSWORD
 Value: !Ref DatabasePassword
 - Name: MYSQL_DATABASE
 Value: todobackend            - Name: SECRET_KEY
 Value: some-random-secret-should-be-here
 Command: 
 - uwsgi
 - --http=0.0.0.0:8000
 - --module=todobackend.wsgi
 - --master
 - --die-on-term
 - --processes=4
 - --threads=2
 - --check-static=/public
 PortMappings:
 - ContainerPort: 8000
              HostPort: 0
 LogConfiguration:
 LogDriver: awslogs
 Options:
 awslogs-group: !Sub /${AWS::StackName}/ecs/todobackend
 awslogs-region: !Ref AWS::Region
 awslogs-stream-prefix: docker
 - Name: collectstatic
          Essential: false
 Image: !Sub ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/docker-in-aws/todobackend:${ApplicationImageTag}
 MemoryReservation: 5
 Cpu: 5          MountPoints:
 - SourceVolume: public
              ContainerPath: /public
 Environment:
 - Name: DJANGO_SETTINGS_MODULE
              Value: todobackend.settings_release
 Command:
 - python3
            - manage.py
            - collectstatic
            - --no-input
 LogConfiguration:
 LogDriver: awslogs
 Options:
 awslogs-group: !Sub /${AWS::StackName}/ecs/todobackend
 awslogs-region: !Ref AWS::Region
 awslogs-stream-prefix: docker  ApplicationLogGroup:
 Type: AWS::Logs::LogGroup
 Properties:
 LogGroupName: !Sub /${AWS::StackName}/ecs/todobackend
 RetentionInDays: 7
  ApplicationServiceTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
...
...
```

使用 CloudFormation 定义 ECS 任务定义

正如您在上面的示例中所看到的，配置任务定义需要合理的配置量，并需要对任务定义所代表的容器应用程序的运行时配置有详细的了解。

在第一章中，当您创建了示例应用并在本地运行时，您必须使用 Docker Compose 执行类似的操作。以下示例显示了 todobackend 存储库中 Docker Compose 文件中的相关片段：

```
version: '2.3'

volumes:
  public:
    driver: local

services:
  ...
  ...
  app:
    image: 385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend:${APP_VERSION}
    extends:
      service: release
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - public:/public
    healthcheck:
      test: curl -fs localhost:8000
    ports:
      - 8000
    command:
      - uwsgi
      - --http=0.0.0.0:8000
      - --module=todobackend.wsgi
      - --master
      - --die-on-term
      - --processes=4
      - --threads=2
      - --check-static=/public
  acceptance:
    extends:
      service: release
    depends_on:
      app:
        condition: service_healthy
    environment:
      APP_URL: http://app:8000
    command:
      - bats 
      - acceptance.bats
  migrate:
    extends:
      service: release
    depends_on:
      db:
        condition: service_healthy
    command:
      - python3
      - manage.py
      - migrate
      - --no-input
  ...
  ...
```

Todobackend 应用程序 Docker Compose 配置

如果您比较前面两个示例的配置，您会发现您可以使用本地 Docker Compose 配置来确定 ECS 任务定义所需的配置。

现在让我们更详细地检查各种 ECS 任务定义配置属性。

# 配置 ECS 任务定义家族

您在任务定义中定义的第一个属性是**Family**属性，它建立了 ECS 任务定义家族名称，并影响 CloudFormation 在您对任务定义进行更改时创建新实例的方式。

回想一下第四章中，ECS 任务定义支持修订的概念，您可以将其视为 ECS 任务定义的特定版本或配置，每当您需要修改任务定义（例如修改镜像标签）时，您可以创建 ECS 任务定义的新修订版本。

因此，如果您的 ECS 任务定义族名称为**todobackend**，则任务定义的第一个修订版将为**todobackend:1**，对任务定义的任何后续更改都将导致创建一个新的修订版，例如**todobackend:2**，**todobackend:3**等。配置 ECS 任务定义资源中的**Family**属性可确保 CloudFormation 在修改 ECS 任务定义资源时采用创建新修订版的行为。

请注意，如果您未按照之前的示例配置**Family**属性，CloudFormation 将为族生成一个随机名称，修订版为 1，对任务定义的任何后续更改都将导致创建一个*新*的族，其名称随机，修订版仍为 1。

# 配置 ECS 任务定义卷

回顾之前示例中的`ApplicationTaskDefinition`资源，`Volumes`属性定义了每当 ECS 任务定义的实例部署到 ECS 容器实例时将创建的本地 Docker 卷。参考之前示例中的本地 Docker Compose 配置，您可以看到配置了一个名为**public**的卷，然后在**app**服务定义中引用为挂载点。

该卷用于存储静态网页文件，这些文件是通过在本地 Makefile 工作流中运行`python3 manage.py collectstatic --no-input`命令生成的，并且必须对主应用程序容器可用，因此需要一个卷来确保通过运行此命令生成的文件对应用程序容器可用：

```
...
...
release:
  docker-compose up --abort-on-container-exit migrate
 docker-compose run app python3 manage.py collectstatic --no-input
  docker-compose up --abort-on-container-exit acceptance
  @ echo App running at http://$$(docker-compose port app 8000 | sed s/0.0.0.0/localhost/g)
...
...
```

Todobackend Makefile

请注意，在我们的 ECS 任务定义中，我们还需要指定一个主机源路径`/data/public`，这是我们在上一章中作为 ECS 集群自动扩展组 CloudFormation init 配置的一部分创建的。该文件夹在底层 ECS 容器实例上具有正确的权限，这确保我们的应用程序能够读取和写入公共卷。

# 配置 ECS 任务定义容器

之前配置的 ECS 任务定义包括`ContainerDefinitions`属性，该属性定义了与任务定义关联的一个或多个容器的列表。您可以看到有两个容器被定义：

+   `todobackend`容器：这是主应用程序容器定义。

+   `collectstatic`容器：这个容器是一个短暂的容器，运行`python3 manage.py collectstatic`命令来生成本地静态网页文件。与这个容器相关的一个重要配置参数是`Essential`属性，它定义了 ECS 是否应该尝试重新启动容器，如果它失败或退出（事实上，ECS 将尝试重新启动任务定义中的所有容器，导致主应用容器不必要地停止和重新启动）。鉴于`collectstatic`容器只打算作为短暂的任务运行，您必须将此属性设置为 false，以确保 ECS 不会尝试重新启动您的 ECS 任务定义容器。

有许多方法可以满足运行收集静态过程以生成静态网页文件的要求。例如，您可以定义一个启动脚本，首先运行收集静态，然后启动应用程序容器，或者您可能希望将静态文件发布到 S3 存储桶，这意味着您将以完全不同的方式运行收集静态过程。

除了 Essential 属性之外，`todobackend`和`collectstatic`容器定义的配置属性非常相似，因此我们将在这里仅讨论主`todobackend`容器定义的属性，并在适当的地方讨论与`collectstatic`容器定义的任何差异。

+   `Image`：此属性定义了容器基于的 Docker 镜像的 URI。请注意，我们发布了您在第五章创建的 ECR 存储库的 URI，用于 todobackend 应用程序，并引用了一个名为`ApplicationImageTag`的堆栈参数，这允许您在部署堆栈时提供适当版本的 Docker 镜像。

+   `Cpu` 和 `MemoryReservation`：这些属性为您的容器分配 CPU 和内存资源。我们将在接下来的章节中更详细地讨论这些资源，但现在要明白，这些值保留了配置的 CPU 分配和内存，但允许您的容器在可用时使用更多的 CPU 和内存（即“burst”）。请注意，您为 `collectstatic` 容器分配了最少量的 CPU 和内存，因为它只需要运行很短的时间，而且很可能 ECS 容器实例将有多余的 CPU 和内存容量可用来满足容器的实际资源需求。这避免了为只在一小部分时间内活动的容器保留大量的 CPU 和内存。

+   `MountPoints`：定义将挂载到容器的 Docker 卷。每个容器都有一个单独的挂载点，将 **public** 卷挂载到 `/public` 容器路径，用于托管静态网页文件。

+   `Environment`：定义将可用于容器的环境变量。参考前面示例中的本地 Docker Compose 配置，您可以看到 release 服务，这是应用服务继承的基本服务定义，指示容器需要将 `DJANGO_SETTINGS_MODULE` 变量设置为 `todobackend.settings_release`，并需要定义一些与数据库相关的环境变量，以定义与应用程序数据库的连接。另一个需要的环境变量是 `SECRET_KEY` 变量，它用于 Django 框架中的各种加密功能，用于驱动 todobackend 应用程序，应该配置为一个秘密的随机值。正如您所看到的，现在我们设置了一个相当非随机的明文值，下一章中，您将学习如何将此值作为加密的秘密注入。

+   `Command`：定义启动容器时应执行的命令。您可以看到 `todobackend` 容器定义使用了与本地 Docker Compose 工作流使用的相同的 `uwsgi` 命令来启动 `uwsgi` 应用服务器，而 `collectstatic` 容器使用 `python3 manage.py collectstatic` 命令来生成要从主应用程序提供的静态网页文件。

+   `PortMappings`：指定应从容器公开的端口映射。todobackend 容器定义有一个单一的端口映射，指定了容器端口的默认应用程序端口`8000`，并指定了主机端口值为`0`，这意味着将使用动态端口映射（请注意，当使用动态端口映射时，您也可以省略 HostPort 参数）。

+   `LogConfiguration`：配置容器的日志记录配置。在前面的示例中，您使用 awslogs 驱动程序配置 CloudWatch 日志作为日志驱动程序，然后配置特定于此驱动程序的选项。awslogs-group 选项指定日志将输出到的日志组，这引用了在`ApplicationLogGroup`资源下方定义的日志组的名称。awslogs-stream-prefix 非常重要，因为它修改了容器 ID 的默认日志流命名约定为`<prefix-name>/<container-name>/<ecs-task-id>`格式，这里的关键信息是 ECS 任务 ID，这是您在使用 ECS 时处理的主要任务标识，而不是容器 ID。

在第七章中，您授予了 ECS 容器实例发布到任何以您的 CloudFormation 堆栈名称为前缀的日志组的能力。只要您的 ECS 任务定义和相关的日志组遵循这个命名约定，Docker 引擎就能够将您的 ECS 任务和容器的日志发布到 CloudWatch 日志中。

# 使用 CloudFormation 部署 ECS 任务定义

现在您已经定义了 ECS 任务定义，您可以使用现在熟悉的`aws cloudformation deploy`命令部署它。一旦您的堆栈已经更新，一个名为 todobackend 的新任务定义应该被创建，您可以使用 AWS CLI 查看，如下例所示：

```
> aws ecs describe-task-definition --task-definition todobackend
{
    "taskDefinition": {
        "taskDefinitionArn": "arn:aws:ecs:us-east-1:385605022855:task-definition/todobackend:1",
        "family": "todobackend",
        "revision": 1,
        "volumes": [
            {
                "name": "public",
                "host": {
                    "sourcePath": "/data/public"
                }
            }
        ],
        "containerDefinitions": [
            {
                "name": "todobackend",
                "image": "385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend:latest",
                "cpu": 245,
                "memoryReservation": 395,
...
...
```

验证 todobackend 任务定义

# 部署 ECS 服务

有了您的 ECS 集群、ECS 任务定义和各种支持资源，现在您可以定义一个 ECS 服务，将您在 ECS 任务定义中定义的容器应用程序部署到您的 ECS 集群中。

以下示例演示了向您的 CloudFormation 模板添加一个`AWS::ECS::Service`资源的 ECS 服务资源：

```
...
...
Resources:
  ApplicationService:
 Type: AWS::ECS::Service
 DependsOn:
      - ApplicationAutoscaling
      - ApplicationLogGroup
      - ApplicationLoadBalancerHttpListener
    Properties:
      TaskDefinition: !Ref ApplicationTaskDefinition
      Cluster: !Ref ApplicationCluster
      DesiredCount: !Ref ApplicationDesiredCount
      LoadBalancers:
        - ContainerName: todobackend
          ContainerPort: 8000
          TargetGroupArn: !Ref ApplicationServiceTargetGroup
      Role: !Sub arn:aws:iam::${AWS::AccountId}:role/aws-service-role/ecs.amazonaws.com/AWSServiceRoleForECS 
```

```
 DeploymentConfiguration:
 MaximumPercent: 200
 MinimumHealthyPercent: 100
  ApplicationTaskDefinition:
    Type: AWS::ECS::TaskDefinition
...
...
```

创建 ECS 服务

在前面的例子中，配置的一个有趣方面是`DependsOn`参数，它定义了堆栈中必须在创建或更新 ECS 服务资源之前创建或更新的其他资源。虽然 CloudFormation 在资源直接引用另一个资源时会自动创建依赖关系，但是一个资源可能对其他资源有依赖，而这些资源与该资源没有直接关系。ECS 服务资源就是一个很好的例子——服务在没有功能的 ECS 集群和相关的 ECS 容器实例（这由`ApplicationAutoscaling`资源表示）的情况下无法运行，并且在没有`ApplicationLogGroup`资源的情况下无法写入日志。一个更微妙的依赖关系是`ApplicationLoadBalancerHttpListener`资源，在与 ECS 服务关联的目标组注册目标之前必须是功能性的。

这里描述了为 ECS 服务配置的各种属性：

+   `TaskDefinition`、`DesiredCount`和`Cluster`：定义了 ECS 任务定义、ECS 任务数量和服务将部署到的目标 ECS 集群。

+   `LoadBalancers`：配置了 ECS 服务应该集成的负载均衡器资源。您必须指定容器名称、容器端口和 ECS 服务将注册的目标组 ARN。请注意，您引用了在本章前面创建的`ApplicationServiceTargetGroup`资源。

+   `Role`：如果要将 ECS 服务与负载均衡器集成，则只有在这种情况下才需要此属性，并且指定了授予 ECS 服务管理配置的负载均衡器权限的 IAM 角色。在前面的例子中，您引用了一个特殊的 IAM 角色的 ARN，这个角色被称为服务角色（[`docs.aws.amazon.com/IAM/latest/UserGuide/using-service-linked-roles.html`](https://docs.aws.amazon.com/IAM/latest/UserGuide/using-service-linked-roles.html)），它在创建 ECS 资源时由 AWS 自动创建。`AWSServiceRoleForECS`服务角色授予了通常需要的 ECS 权限，包括管理和集成应用程序负载均衡器。

+   `DeploymentConfiguration`：配置与 ECS 任务定义的新版本滚动部署相关的设置。在部署过程中，ECS 将停止现有容器，并根据 ECS 任务定义的新版本部署新容器，`MinimumHealthyPercent`设置定义了在部署过程中必须处于服务状态的容器的最低允许百分比，与`DesiredCount`属性相关。同样，`MaximumPercent`设置定义了在部署过程中可以部署的容器的最大允许百分比，与`DesiredCount`属性相关。

# 使用 CloudFormation 部署 ECS 服务

在设置好 ECS 服务配置后，现在是时候使用`aws cloudformation deploy`命令将更改部署到您的堆栈了。部署完成后，您的 ECS 服务应该注册到您在本章前面创建的目标组中，如果您浏览到您的应用程序负载均衡器的 URL，您应该看到示例应用程序的根 URL 正在正确加载：

![](img/68a5cac1-2c9c-4711-bb88-70eb0ee1e14e.png)测试 todobackend 应用程序

然而，如果您点击前面截图中显示的**todos**链接，您将收到一个错误，如下截图所示：

![](img/b1ba0115-82ab-40ff-9473-2f5887ed15b7.png)todobackend 应用程序错误

在前面的截图中的问题是，应用程序数据库中预期的数据库表尚未创建，因为我们尚未对应用程序数据库运行数据库迁移。我们将很快学习如何解决这个问题，但在我们这样做之前，我们还有一个与部署 ECS 服务相关的主题要讨论：滚动部署。

# ECS 滚动部署

ECS 的一个关键特性是滚动部署，ECS 将自动以滚动方式部署应用程序的新版本，与您配置的负载均衡器一起协调各种操作，以确保您的应用程序成功部署，没有停机时间，也不会影响最终用户。ECS 如何管理滚动部署的过程实际上是非常详细的，以下图表试图以一个图表高层次地描述这个过程：

![](img/9486bf16-31d9-4726-a3fb-d06d0fd92aeb.png)ECS 滚动部署

在前面的图表中，滚动部署期间发生了以下事件：

1.  对与 ECS 服务关联的`ApplicationTaskDefinition` ECS 任务定义进行配置更改，通常是应用程序新版本的镜像标签的更改，但也可能是对任务定义的任何更改。这将导致任务定义的新修订版被创建（在这个例子中是修订版 2）。

1.  ECS 服务配置为使用新的任务定义修订版，当使用 CloudFormation 来管理 ECS 资源时，这是自动发生的。ECS 服务的部署配置决定了 ECS 如何管理滚动部署-在前面的图表中，ECS 必须确保在部署过程中维持配置的期望任务数量的最低 100%，并且可以在部署过程中暂时增加任务数量达到最高 200%。假设期望的任务数量为 1，这意味着 ECS 可以部署基于新任务定义修订版的新 ECS 任务并满足部署配置。请注意，您的 ECS 集群必须有足够的资源来容纳这些部署，并且您负责管理 ECS 集群的容量（即 ECS 不会暂时增加 ECS 集群的容量来容纳部署）。您将在后面的章节中学习如何动态管理 ECS 集群的容量。

1.  一旦新的 ECS 任务成功启动，ECS 会将新任务注册到配置的负载均衡器（在应用负载均衡器的情况下，任务将被注册到目标组资源）。负载均衡器将执行健康检查来确定新任务的健康状况，一旦确认健康，新的 ECS 任务将被注册到负载均衡器并能够接受传入连接。

1.  ECS 现在指示负载均衡器排水现有的 ECS 任务。负载均衡器将使现有的 ECS 任务停止服务（即不会将任何新连接转发到任务），但会等待一段可配置的时间来使现有连接“排水”或关闭。在此期间，任何对负载均衡器的新连接将被转发到在第 3 步中向负载均衡器注册的新 ECS 任务。

1.  一旦排水过程完成，负载均衡器将完全从目标组中删除旧的 ECS 任务，ECS 现在可以终止现有的 ECS 任务。一旦这个过程完成，新应用任务定义的部署就完成了。

从这个描述中可以看出，部署过程非常复杂。好消息是，所有这些都可以通过 ECS 开箱即用——您需要理解的是，对任务定义的任何更改都将触发新的部署，并且您的部署配置，由 DeploymentConfiguration 属性确定，可以在滚动部署中对其进行一些控制。

# 执行滚动部署

现在您了解了滚动部署的工作原理，让我们通过对 ECS 任务定义进行更改并通过 CloudFormation 部署更改的过程来看看它的实际操作，这将触发 ECS 服务的滚动部署。

目前，您的 CloudFormation 配置未指定 ApplicationImageTag 参数，这意味着您的 ECS 任务定义正在使用 latest 的默认值。回到第五章，当您将 Docker 镜像发布到 ECR 时，实际上推送了两个标签——latest 标签和 todobackend 存储库的提交哈希。这为我们提供了一个很好的机会来进一步改进我们的 CloudFormation 模板——通过引用提交哈希，而不是 latest 标签，我们将始终能够在您有新版本的应用程序要部署时触发对 ECS 任务定义的配置更改。

以下示例演示了在 todobackend-aws 存储库中的 dev.cfg 文件中添加 ApplicationImageTag 参数，引用当前发布的 ECR 镜像的提交哈希：

```
ApplicationDesiredCount=1
ApplicationImageId=ami-ec957491
ApplicationImageTag=97e4abf
ApplicationSubnets=subnet-a5d3ecee,subnet-324e246f
VpcId=vpc-f8233a80
```

将 ApplicationImageTag 添加到 dev.cfg 文件

如果您现在使用 aws cloudformation deploy 命令部署更改，尽管您现在引用的镜像与当前 latest 标记的镜像相同，CloudFormation 将检测到这是一个配置更改，创建 ECS 任务定义的新修订版本，并更新 ApplicationService ECS 服务资源，触发滚动部署。

在部署运行时，如果您浏览 ECS 仪表板中的 ECS 服务并选择部署选项卡，如下截图所示，您将看到两个部署——ACTIVE 部署指的是现有的 ECS 任务，而 PRIMARY 部署指的是正在部署的新的 ECS 任务。

![](img/44aab8c6-3a36-47ec-b224-74861afc4181.png)ECS 服务滚动部署

最终，一旦滚动部署过程完成，ACTIVE 部署将消失，如果您点击“事件”选项卡，您将看到部署过程中发生的各种事件，这些事件对应了先前的描述：

![](img/adc2feb0-c12b-4822-b81d-0acf52cefc78.png)ECS 服务滚动部署事件

# 创建 CloudFormation 自定义资源

尽管我们的应用已经部署并运行，但很明显我们有一个问题，即我们尚未运行数据库迁移，这是一个必需的部署任务。我们已经处理了运行另一个部署任务，即收集静态文件，但是数据库迁移应该只作为*单个*部署任务运行。例如，如果您正在部署服务的多个实例，您不希望为每个部署的实例运行迁移，您只想在每个部署中运行一次迁移，而不管服务中有多少实例。

一个明显的解决方案是在每次部署后手动运行迁移，但是理想情况下，您希望完全自动化您的部署，并确保您有一种机制可以自动运行迁移。CloudFormation 不提供允许您运行一次性 ECS 任务的资源，但是 CloudFormation 的一个非常强大的功能是能够创建自己的自定义资源，这使您能够执行自定义的配置任务。创建自定义资源的好处是您可以将自定义的配置任务纳入部署各种 AWS 服务和资源的工作流程中，使用 CloudFormation 框架来为您管理这一切。

现在让我们学习如何创建一个简单的 ECS 任务运行器自定义资源，该资源将作为创建和更新应用程序环境的一部分来运行迁移任务。

# 理解 CloudFormation 自定义资源

在开始配置 CloudFormation 自定义资源之前，值得讨论它们实际上是如何工作的，并描述组成自定义资源的关键组件。

以下图表说明了 CloudFormation 自定义资源的工作原理：

![](img/a8738f96-3ab5-46a3-95dc-d5d9216a7f06.png)CloudFormation 自定义资源

在上图中，当您在 CloudFormation 模板中使用自定义资源时，将发生以下步骤：

1.  您需要在 CloudFormation 模板中定义自定义资源。自定义资源具有`AWS::CloudFormation::CustomResource`资源类型，或者是`Custom::<resource-name>`。当 CloudFormation 遇到自定义资源时，它会查找一个名为`ServiceToken`的特定属性，该属性提供应该配置自定义资源的 Lambda 函数的 ARN。

1.  CloudFormation 调用 Lambda 函数，并以 JSON 对象的形式将自定义资源请求传递给函数。事件具有请求类型，该类型定义了请求是创建、更新还是删除资源，并包括请求属性，这些属性是您可以在自定义资源定义中定义的自定义属性，将传递给 Lambda 函数。请求的另一个重要属性是响应 URL，它提供了一个预签名的 S3 URL，Lambda 函数应在配置完成后向其发布响应。

1.  Lambda 函数处理自定义资源请求，并根据请求类型和请求属性执行资源的适当配置。配置完成后，函数向自定义资源请求中收到的响应 URL 发布成功或失败的响应，并在创建或更新资源时包含资源标识符。假设响应信号成功，响应可能包括`Data`属性，该属性可以包含有关已配置的自定义资源的有用信息，可以在 CloudFormation 堆栈的其他位置使用标准的`!Sub ${<resource-name>.<data-property>}`语法引用，其中`<data-property>`是响应的`Data`属性中包含的属性。

1.  云形成服务轮询响应 URL 以获取响应。一旦收到响应，CloudFormation 解析响应并继续堆栈配置（或在响应指示失败的情况下回滚堆栈）。

# 创建自定义资源 Lambda 函数

如前一节所讨论的，自定义资源需要您创建一个 Lambda 函数，该函数处理 CloudFormation 发送的传入事件，执行自定义配置操作，然后使用预签名的 S3 URL 响应 CloudFormation。

这听起来相当复杂，但是有许多可用的工具可以使这在相对简单的用例中成为可能，如以下示例所示。

```
...
...
Resources:
 EcsTaskRunner:
 Type: AWS::Lambda::Function
    DependsOn:
 - EcsTaskRunnerLogGroup
 Properties:
 FunctionName: !Sub ${AWS::StackName}-ecsTasks
 Description: !Sub ${AWS::StackName} ECS Task Runner
 Handler: index.handler
 MemorySize: 128
 Runtime: python3.6
 Timeout: 300
      Role: !Sub ${EcsTaskRunnerRole.Arn}
 Code:
 ZipFile: |
 import cfnresponse
 import boto3

 client = boto3.client('ecs')

 def handler(event, context):
 try:
              print("Received event %s" % event)
              if event['RequestType'] == 'Delete':
                cfnresponse.send(event, context, cfnresponse.SUCCESS, {}, event['PhysicalResourceId'])
                return
              tasks = client.run_task(
                cluster=event['ResourceProperties']['Cluster'],
                taskDefinition=event['ResourceProperties']['TaskDefinition'],
                overrides=event['ResourceProperties'].get('Overrides',{}),
                count=1,
                startedBy=event['RequestId']
              )
              task = tasks['tasks'][0]['taskArn']
              print("Started ECS task %s" % task)
              waiter = client.get_waiter('tasks_stopped')
              waiter.wait(
                cluster=event['ResourceProperties']['Cluster'],
                tasks=[task],
              )
              result = client.describe_tasks(
                cluster=event['ResourceProperties']['Cluster'],
                tasks=[task]
              )
              exitCode = result['tasks'][0]['containers'][0]['exitCode']
              if exitCode > 0:
                print("ECS task %s failed with exit code %s" % (task, exitCode))
                cfnresponse.send(event, context, cfnresponse.FAILED, {}, task)
              else:
                print("ECS task %s completed successfully" % task)
                cfnresponse.send(event, context, cfnresponse.SUCCESS, {}, task)
            except Exception as e:
              print("A failure occurred with exception %s" % e)
              cfnresponse.send(event, context, cfnresponse.FAILED, {})
 EcsTaskRunnerRole:
 Type: AWS::IAM::Role
 Properties:
 AssumeRolePolicyDocument:
 Version: "2012-10-17"
 Statement:
 - Effect: Allow
 Principal:
 Service: lambda.amazonaws.com
 Action:
 - sts:AssumeRole
 Policies:
 - PolicyName: EcsTaskRunnerPermissions
 PolicyDocument:
 Version: "2012-10-17"
 Statement:
 - Sid: EcsTasks
 Effect: Allow
 Action:
 - ecs:DescribeTasks
 - ecs:ListTasks
 - ecs:RunTask
 Resource: "*"
 Condition:
 ArnEquals:
 ecs:cluster: !Sub ${ApplicationCluster.Arn}
 - Sid: ManageLambdaLogs
 Effect: Allow
 Action:
 - logs:CreateLogStream
 - logs:PutLogEvents
 Resource: !Sub ${EcsTaskRunnerLogGroup.Arn}
 EcsTaskRunnerLogGroup:
 Type: AWS::Logs::LogGroup
 Properties:
 LogGroupName: !Sub /aws/lambda/${AWS::StackName}-ecsTasks
 RetentionInDays: 7
  ApplicationService:
    Type: AWS::ECS::Service
...
...
```

使用 CloudFormation 创建内联 Lambda 函数

前面示例中最重要的方面是`EcsTaskRunner`资源中的`Code.ZipFile`属性，它定义了一个内联 Python 脚本，执行自定义资源的自定义配置操作。请注意，这种内联定义代码的方法通常不推荐用于实际用例，稍后我们将创建一个更复杂的自定义资源，其中包括自己的 Lambda 函数代码的源代码库，但为了保持这个示例简单并介绍自定义资源的核心概念，我现在使用了内联方法。

# 理解自定义资源函数代码

让我们专注于讨论自定义资源函数代码，我已经在之前的示例中将其隔离，并添加了注释来描述各种语句的作用。

```
# Generates an appropriate CloudFormation response and posts to the pre-signed S3 URL
import cfnresponse
# Imports the AWS Python SDK (boto3) for interacting with the ECS service
import boto3

# Create a client for interacting with the ECS service
client = boto3.client('ecs')

# Lambda functions require a handler function that is passed an event and context object
# The event object contains the CloudFormation custom resource event
# The context object contains runtime information about the Lambda function
def handler(event, context):
  # Wrap the code in a try/catch block to ensure any exceptions generate a failure
  try:
    print("Received event %s" % event)
    # If the request is to Delete the resource, simply return success
    if event['RequestType'] == 'Delete':
      cfnresponse.send(event, context, cfnresponse.SUCCESS, {}, event.get('PhysicalResourceId'))
      return
    # Run the ECS task
    # http://boto3.readthedocs.io/en/latest/reference/services/ecs.html#ECS.Client.run_task
    # Requires 'Cluster', 'TaskDefinition' and optional 'Overrides' custom resource properties
    tasks = client.run_task(
      cluster=event['ResourceProperties']['Cluster'],
      taskDefinition=event['ResourceProperties']['TaskDefinition'],
      overrides=event['ResourceProperties'].get('Overrides',{}),
      count=1,
      startedBy=event['RequestId']
    )
    # Extract the ECS task ARN from the return value from the run_task call
    task = tasks['tasks'][0]['taskArn']
    print("Started ECS task %s" % task)

    # Creates a waiter object that polls and waits for ECS tasks to reached a stopped state
    # http://boto3.readthedocs.io/en/latest/reference/services/ecs.html#waiters
    waiter = client.get_waiter('tasks_stopped')
    # Wait for the task ARN that was run earlier to stop
    waiter.wait(
      cluster=event['ResourceProperties']['Cluster'],
      tasks=[task],
    )
    # After the task has stopped, get the status of the task
    # http://boto3.readthedocs.io/en/latest/reference/services/ecs.html#ECS.Client.describe_tasks
    result = client.describe_tasks(
      cluster=event['ResourceProperties']['Cluster'],
      tasks=[task]
    )
    # Get the exit code of the container that ran
    exitCode = result['tasks'][0]['containers'][0]['exitCode']
    # Return failure for non-zero exit code, otherwise return success
    # See https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-lambda-function-code.html for more details on cfnresponse module
    if exitCode > 0:
      print("ECS task %s failed with exit code %s" % (task, exitCode))
      cfnresponse.send(event, context, cfnresponse.FAILED, {}, task)
```

```
else:
      print("ECS task %s completed successfully" % task)
      cfnresponse.send(event, context, cfnresponse.SUCCESS, {}, task)
  except Exception as e:
    print("A failure occurred with exception %s" % e)
    cfnresponse.send(event, context, cfnresponse.FAILED, {})
```

使用 CloudFormation 创建内联 Lambda 函数

在高层次上，自定义资源函数接收 CloudFormation 自定义资源事件，并调用 AWS Python SDK 中 ECS 服务的`run_task`方法，传入 ECS 集群、ECS 任务定义和可选的覆盖以执行。然后函数等待任务完成，检查 ECS 任务的结果，以确定相关容器是否成功完成，然后向 CloudFormation 响应成功或失败。

注意，函数导入了一个名为`cfnresponse`的模块，这是 AWS Lambda Python 运行时环境中包含的一个模块，提供了一个简单的高级机制来响应 CloudFormation 自定义资源请求。函数还导入了一个名为`boto3`的模块，它提供了 AWS Python SDK，并用于创建一个与 ECS 服务专门交互的`client`对象。然后 Lambda 函数定义了一个名为`handler`的函数，这是传递给 Lambda 函数的新事件的入口点，并注意`handler`函数必须接受包含 CloudFormation 自定义资源事件的`event`对象和提供有关 Lambda 环境的运行时信息的`context`对象。请注意，函数应该只尝试运行 CloudFormation 创建和更新请求的任务，并且当接收到删除自定义资源的请求时，可以简单地返回成功，因为任务是短暂的资源。

前面示例中的代码绝不是生产级代码，并且已经简化为仅处理与成功和失败相关的两个主要场景以进行演示。

# 了解自定义资源 Lambda 函数资源

现在您了解了 Lambda 函数代码的实际工作原理，让我们专注于您在之前示例中添加的配置的其余部分。

`EcsTaskRunner`资源定义了 Lambda 函数，其中描述了关键配置属性：

+   `FunctionName`：函数的名称。要理解的一个重要方面是，用于存储函数日志的关联 CloudWatch 日志组必须遵循`/aws/lambda/<function-name>`的命名约定，您会看到`FunctionName`属性与`EcsTaskRunnerLogGroup`资源的`LogGroupName`属性匹配。请注意，`EcsTaskRunner`还必须声明对`EcsTaskRunnerLogGroup`资源的依赖性，根据`DependsOn`设置的配置。

+   `处理程序`：指定 Lambda 函数的入口点，格式为`<module>.<function>`。请注意，当使用模块创建的内联代码机制时，用于 Lambda 函数的模块始终被称为`index`。

+   `超时`：重要的是要理解，目前 Lambda 的最长超时时间为五分钟（300 秒），这意味着您的函数必须在五分钟内完成，否则它们将被终止。Lambda 函数的默认超时时间为 3 秒，因为部署新的 ECS 任务，运行 ECS 任务并等待任务完成需要时间，因此将此超时时间增加到最大超时时间为 300 秒。

+   `角色`：定义要分配给 Lambda 函数的 IAM 角色。请注意，引用的`EcsTaskRunnerRole`资源必须信任 lambda.amazonaws.com，而且至少每个 Lambda 函数必须具有权限写入关联的 CloudWatch 日志组，如果您想要捕获任何日志。ECS 任务运行器函数需要权限来运行和描述 ECS 任务，并且使用条件配置为仅向堆栈中定义的 ECS 集群授予这些权限。

# 创建自定义资源

现在你的自定义资源 Lambda 函数和相关的支持资源都已经就位，你可以定义实际的自定义资源对象。对于我们的用例，我们需要定义一个自定义资源，它将在我们的应用容器中运行`python3 manage.py migrate`命令，并且由于迁移任务与应用数据库交互，任务必须配置各种数据库环境变量，以定义与应用数据库资源的连接。

一种方法是利用之前创建的`ApplicationTaskDefinition`资源，并指定一个命令覆盖，但一个问题是`ApplicationTaskDefinition`包括`collectstatic`容器，我们并不真的想在运行迁移时运行它。为了克服这个问题，你需要创建一个名为`MigrateTaskDefinition`的单独任务定义，它只包括一个特定运行数据库迁移的容器定义：

```
...
...
Resources:
 MigrateTaskDefinition:
    Type: AWS::ECS::TaskDefinition
 Properties:
 Family: todobackend-migrate
 ContainerDefinitions:
 - Name: migrate
 Image: !Sub ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/docker-in-aws/todobackend:${ApplicationImageTag}
 MemoryReservation: 5
 Cpu: 5
 Environment:
 - Name: DJANGO_SETTINGS_MODULE
 Value: todobackend.settings_release
 - Name: MYSQL_HOST
 Value: !Sub ${ApplicationDatabase.Endpoint.Address}
 - Name: MYSQL_USER
 Value: todobackend
 - Name: MYSQL_PASSWORD
 Value: !Ref DatabasePassword
 - Name: MYSQL_DATABASE
 Value: todobackend
```

```
Command: 
 - python3
 - manage.py
 - migrate
 - --no-input
 LogConfiguration:
 LogDriver: awslogs
 Options:
 awslogs-group: !Sub /${AWS::StackName}/ecs/todobackend
 awslogs-region: !Ref AWS::Region
 awslogs-stream-prefix: docker
  EcsTaskRunner:
    Type: AWS::Lambda::Function
...
...

```

创建迁移任务定义

在上面的例子中，注意到`MigrateTaskDefinition`资源需要配置与数据库相关的环境变量，但不需要你之前在`ApplicationTaskDefinition`资源中配置的卷映射或端口映射。

有了这个任务定义，你现在可以创建你的自定义资源，就像下面的例子所示：

```
...
...
Resources:
 MigrateTask:
 Type: AWS::CloudFormation::CustomResource
 DependsOn:
 - ApplicationAutoscaling
 - ApplicationDatabase
 Properties:
 ServiceToken: !Sub ${EcsTaskRunner.Arn}
 Cluster: !Ref ApplicationCluster
 TaskDefinition: !Ref MigrateTaskDefinition MigrateTaskDefinition:
     Type: AWS::ECS::TaskDefinition
   ...
   ...
   ApplicationService:
    Type: AWS::ECS::Service
    DependsOn:
      - ApplicationAutoscaling
      - ApplicationLogGroup
      - ApplicationLoadBalancerHttpListener
 - MigrateTask
```

```
Properties:
...
...
```

创建迁移任务自定义资源

在上面的例子中，注意到你的自定义资源是用`AWS::CloudFormation::CustomResource`类型创建的，你创建的每个自定义资源都必须包括`ServiceToken`属性，它引用了相关自定义资源 Lambda 函数的 ARN。其余的属性是特定于你的自定义资源函数的，对于我们的情况，至少必须指定要执行的任务的目标 ECS 集群和 ECS 任务定义。注意，自定义资源包括依赖关系，以确保它只在`ApplicationAutoscaling`和`ApplicationDatabase`资源创建后运行，你还需要在本章前面创建的`ApplicationService`资源上添加一个依赖关系，以便在`MigrateTask`自定义资源成功完成之前不会创建或更新此资源。

# 部署自定义资源

现在，您可以使用`aws cloudformation deploy`命令部署您的更改。在 CloudFormation 堆栈更改部署时，一旦 CloudFormation 启动创建自定义资源并调用您的 Lambda 函数，您可以导航到 AWS Lambda 控制台查看您的 Lambda 函数，并检查函数日志。

CloudFormation 自定义资源在最初工作时可能会耗费大量时间，特别是如果您的代码抛出异常并且没有适当的代码来捕获这些异常并发送失败响应。您可能需要等待几个小时才能超时，因为您的自定义资源抛出了异常并且没有返回适当的失败响应给 CloudFormation。

以下屏幕截图演示了在 AWS Lambda 控制台中查看从 CloudFormation 堆栈创建的`todobackend-ecsTasks` Lambda 函数：

![](img/eb137671-2741-4ba2-8634-1079e75d3526.png)在 AWS 控制台中查看 Lambda 函数

在上面的屏幕截图中，**配置**选项卡提供了有关函数的配置详细信息，甚至包括内联代码编辑器，您可以在其中查看、测试和调试您的代码。**监控**选项卡提供了对函数的各种指标的访问权限，并包括一个有用的**跳转到日志**链接，该链接可以直接带您到 CloudWatch 日志中函数的日志：

![](img/6adc9b11-80cc-4199-b42f-94ef997e1f74.png)在 AWS 控制台中查看 Lambda 函数日志

在上面的屏幕截图中，START 消息指示函数何时被调用，并且您可以看到生成了一个状态为 SUCCESS 的响应体，该响应体被发布到 CloudFormation 自定义资源响应 URL。

现在是审查 ECS 任务的 CloudWatch 日志的好时机——显示了**/todobackend/ecs/todobackend**日志组，这是在您的 CloudFormation 堆栈中配置的日志组，用于收集应用程序的所有 ECS 任务日志。请注意，有几个日志流 - 一个用于生成静态任务的**collectstatic**容器，一个用于运行迁移的**migrate**容器，以及一个用于主要 todobackend 应用程序的日志流。请注意，每个日志流的末尾都包括 ECS 任务 ID - 这些直接对应于您使用 ECS 控制台或 AWS CLI 与之交互的 ECS 任务 ID：

ECS CloudWatch 日志组

# 验证应用程序

作为最后的检查，示例应用程序现在应该是完全功能的 - 例如，之前失败的待办事项链接现在应该可以工作，如下面的截图所示。

您可以与 API 交互以添加或删除待办事项，并且所有待办事项现在将持久保存在应用程序数据库中，该数据库在您的堆栈中定义：

![](img/98dcd51c-619f-4fde-bab1-95f5526d88bb.png)Working todobackend application

# 总结

在本章中，您成功地将示例 Docker 应用程序部署到 AWS 使用 ECS。您学会了如何定义关键的支持应用程序和基础设施资源，包括如何使用 AWS RDS 服务创建应用程序数据库，以及如何将您的 ECS 应用程序与 AWS 弹性负载均衡服务提供的应用程序负载均衡器集成。

有了这些支持资源，您学会了如何创建控制容器运行时配置的 ECS 任务定义，然后通过为示例应用程序创建 ECS 服务来部署您的 ECS 任务定义的实例。您学会了 ECS 任务定义如何定义卷和多个容器定义，并且您使用了这个功能来创建一个单独的非必要容器定义，每当部署您的 ECS 任务定义时，它总是运行并为示例应用程序生成静态网页文件。您还将示例应用程序的 ECS 服务与堆栈中的各种应用程序负载均衡器资源集成，确保可以跨多个 ECS 服务实例进行负载均衡连接到您的应用程序。

尽管您能够成功将应用程序部署为 ECS 服务，但您发现您的应用程序并不完全功能，因为尚未运行为应用程序数据库建立架构和表的数据库迁移。您通过创建 ECS 任务运行器 CloudFormation 自定义资源来解决了这个问题，这使您能够在每次应用程序部署时运行迁移作为单次任务。自定义资源被定义为一个简单的用 Python 编写的 Lambda 函数，它首先在给定的 ECS 集群上为给定的 ECS 任务定义运行任务，等待任务完成，然后根据与任务相关联的容器的退出代码报告任务的成功或失败。

有了这个自定义资源，您的示例应用现在已经完全可用，尽管它仍然存在一些不足之处。在下一章中，我们将解决其中一个不足之处——保密管理和确保密码保持机密——这在安全的、生产级别的 Docker 应用中至关重要。

# 问题

1.  真/假：RDS 实例需要您创建至少两个子网的 DB 子网组。

1.  在配置应用负载均衡器时，哪个组件服务于来自最终用户的前端连接？

1.  真/假：在创建应用负载均衡器监听器之前，目标组可以接受来自目标的注册。

1.  在配置允许应用数据库和 ECS 容器实例之间访问的安全组规则时，您收到了关于循环依赖的 CloudFormation 错误。您可以使用哪种类型的资源来克服这个问题？

1.  您配置了一个包括两个容器定义的 ECS 任务定义。其中一个容器定义执行一个短暂的配置任务然后退出。您发现 ECS 不断地基于这个任务定义重新启动 ECS 服务。您如何解决这个问题？

1.  您可以配置哪个 CloudFormation 参数来定义对其他资源的显式依赖关系？

1.  真/假：CloudFormation 自定义资源使用 AWS Lambda 函数执行自定义的配置任务。

1.  在接收 CloudFormation 自定义资源事件时，您需要处理哪三种类型的事件？

1.  您创建了一个带有内联 Python 函数的 Lambda 函数，用于执行自定义的配置任务，但是当尝试查看该函数的日志时，没有任何内容被写入 CloudWatch 日志。您确认日志组名称已正确配置给该函数。出现这个问题最可能的原因是什么？

# 进一步阅读

您可以查看以下链接，了解本章涵盖的主题的更多信息：

+   CloudFormation RDS 实例资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html)

+   CloudFormation 应用负载均衡器资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-elasticloadbalancingv2-loadbalancer.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-elasticloadbalancingv2-loadbalancer.html)

+   CloudFormation 应用负载均衡监听器资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-elasticloadbalancingv2-listener.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-elasticloadbalancingv2-listener.html)

+   CloudFormation 应用负载均衡目标组资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-elasticloadbalancingv2-targetgroup.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-elasticloadbalancingv2-targetgroup.html)

+   CloudFormation ECS 任务定义资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-taskdefinition.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-taskdefinition.html)

+   CloudFormation ECS 服务资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-service.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-service.html)

+   CloudFormation Lambda 函数资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-function.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-function.html)

+   CloudFormation Lambda 函数代码：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-lambda-function-code.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-lambda-function-code.html)

+   CloudFormation 自定义资源文档：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)

+   CloudFormation 自定义资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref.html)

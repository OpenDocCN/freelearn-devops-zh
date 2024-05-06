# 第十一章：管理 ECS 基础设施生命周期

与操作 ECS 基础设施相关的一个基本持续活动是管理 ECS 容器实例的生命周期。在任何生产级别的场景中，您都需要对 ECS 容器实例进行打补丁，并确保 ECS 容器实例的核心组件（如 Docker 引擎和 ECS 代理）经常更新，以确保您可以访问最新功能和安全性和性能增强。在一个不可变基础设施的世界中，您的 ECS 容器实例被视为“牲畜”，标准方法是通过滚动新的 Amazon 机器映像（AMIs）销毁和替换 ECS 容器实例，而不是采取传统的打补丁“宠物”方法，并将 ECS 容器实例保留很长时间。另一个常见的用例是需要管理生命周期的与自动扩展相关，例如，如果您在高需求期后扩展 ECS 集群，您需要能够从集群中移除 ECS 容器实例。

将 ECS 容器实例从服务中移除听起来可能是一个很简单的任务，然而请考虑一下如果您的实例上有正在运行的容器会发生什么。如果立即将实例移出服务，连接到运行在这些容器上的应用程序的用户将会受到干扰，这可能会导致数据丢失，至少会让用户感到不满。所需的是一种机制，使您的 ECS 容器实例能够优雅地退出服务，保持当前用户连接，直到可以在不影响最终用户的情况下关闭它们，然后在确保实例完全退出服务后终止实例。

在本章中，您将学习如何通过利用两个关键的 AWS 功能来实现这样的能力——EC2 自动缩放生命周期钩子和 ECS 容器实例排空。EC2 自动缩放生命周期钩子让您了解与启动或停止 EC2 实例相关的待处理生命周期事件，并为您提供机会在发出生命周期事件之前执行任何适当的初始化或清理操作。这就是您可以利用 ECS 容器实例排空的地方，它将受影响的 ECS 容器实例上的 ECS 任务标记为排空或停用，并开始优雅地将任务从服务中取出，方法是在集群中的其他 ECS 容器实例上启动新的替代 ECS 任务，然后排空到受影响的 ECS 任务的连接，直到任务可以停止并且 ECS 容器实例被排空。

将涵盖以下主题：

+   理解 ECS 基础设施的生命周期管理

+   构建新的 ECS 容器实例 AMI

+   配置 EC2 自动缩放滚动更新

+   创建 EC2 自动缩放生命周期钩子

+   创建用于消耗生命周期钩子的 Lambda 函数

+   部署和测试自动缩放生命周期钩子

# 技术要求

以下列出了完成本章所需的技术要求：

+   AWS 账户的管理员访问

+   根据第三章的说明配置本地 AWS 配置文件

+   AWS CLI 版本 1.15.71 或更高版本

+   本章继续自第九章（而不是第十章），因此需要您成功完成第九章中定义的所有配置任务，并确保您已将**todobackend-aws**存储库重置为主分支（应基于第九章的完成）

以下 GitHub URL 包含本章中使用的代码示例 - [`github.com/docker-in-aws/docker-in-aws/tree/master/ch11`](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch11)[.](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch4)

查看以下视频以查看代码的实际操作：

[`bit.ly/2BT7DVh`](http://bit.ly/2BT7DVh)

# 理解 ECS 生命周期管理

如本章介绍中所述，ECS 生命周期管理是指将现有的 ECS 容器实例从服务中取出的过程，而不会影响连接到在您受影响的实例上运行的应用程序的最终用户。

这需要您利用 AWS 提供的两个关键功能：

+   EC2 自动扩展生命周期挂钩

+   ECS 容器实例排水

# EC2 自动扩展生命周期挂钩

EC2 自动扩展生命周期挂钩允许您在挂起的生命周期事件发生之前收到通知并在事件发生之前执行某些操作。目前，您可以收到以下生命周期挂钩事件的通知：

+   `EC2_INSTANCE_LAUNCHING`：当 EC2 实例即将启动时引发

+   `EC2_INSTANCE_TERMINATING`：当 EC2 实例即将终止时引发

一般情况下，您不需要担心`EC2_INSTANCE_LAUNCHING`事件，但是任何运行生产级 ECS 集群的人都应该对`EC2_INSTANCE_TERMINATING`事件感兴趣，因为即将终止的实例可能正在运行具有活动最终用户连接的容器。一旦您订阅了生命周期挂钩事件，EC2 自动扩展服务将等待您发出信号，表明生命周期操作可以继续进行。这为您提供了一种机制，允许您在`EC2_INSTANCE_TERMINATING`事件发生时执行优雅的拆除操作，这就是您可以利用 ECS 容器实例排水的地方。

# ECS 容器实例排水

ECS 容器实例排水是一个功能，允许您优雅地排水您的 ECS 容器实例中正在运行的 ECS 任务，最终结果是您的 ECS 容器实例没有正在运行的 ECS 任务或容器，这意味着可以安全地终止实例而不影响您的容器应用程序。ECS 容器实例排水首先将您的 ECS 容器实例标记为 DRAINING 状态，这将导致在实例上运行的所有 ECS 任务被优雅地关闭并在集群中的其他容器实例上启动。这种排水活动使用了您已经在 ECS 服务中看到的标准*滚动*行为，例如，如果您有一个与具有应用程序负载均衡器集成的 ECS 服务相关联的 ECS 任务，ECS 将首先尝试在另一个 ECS 容器实例上注册一个新的 ECS 任务作为应用程序负载均衡器目标组中的新目标，然后将与正在排水的 ECS 容器实例相关联的目标放置到连接排水状态。

请注意，重要的是您的 ECS 集群具有足够的资源和 ECS 容器实例来迁移每个受影响的 ECS 任务，这可能具有挑战性，因为您还通过一个实例减少了 ECS 集群的容量。这意味着，例如，如果您正在计划替换集群中的 ECS 容器实例（例如，您正在更新到新的 AMI），那么您需要临时向集群添加额外的容量，以便以滚动方式交换实例，而不会减少整体集群容量。如果您正在使用 CloudFormation 部署您的 EC2 自动扩展组，一个非常有用的功能是能够指定更新策略，在滚动更新期间临时向您的自动扩展组添加额外的容量，您将学习如何利用此功能始终确保在执行滚动更新时始终保持 ECS 集群容量。

# ECS 生命周期管理解决方案

现在您已经了解了 ECS 生命周期管理的一些背景知识，让我们讨论一下您将在本章中实施的解决方案，该解决方案将利用 EC2 生命周期挂钩来触发 ECS 容器实例的排空，并在安全终止 ECS 容器实例时向 EC2 自动扩展服务发出信号。

以下图表说明了一个简单的 EC2 自动扩展组和一个具有两个 ECS 容器实例的 ECS 集群，支持 ECS **Service A**和 ECS **Service B**，它们都有两个 ECS 任务或 ECS 服务的实例正在运行：

在服务中的 EC2 自动扩展组/ECS 集群

假设您现在希望使用新的 Amazon Machine Image 更新 EC2 自动扩展组中的 ECS 容器实例，这需要终止并替换每个实例。以下图表说明了我们的生命周期挂钩解决方案将如何处理这一要求，并确保自动扩展组中的每个实例都可以以不干扰连接到每个 ECS 服务的应用程序的最终用户的方式进行替换：

执行滚动更新的在服务中的 EC2 自动扩展组/ECS 集群

在上图中，发生以下步骤：

1.  CloudFormation 滚动更新已配置为 EC2 自动扩展组，这会导致 CloudFormation 服务临时增加 EC2 自动扩展组的大小。

1.  EC2 自动扩展组根据 CloudFormation 中组大小的增加，向自动扩展组添加一个新的 EC2 实例（ECS 容器实例 C）。

1.  一旦新的 EC2 实例启动并向 CloudFormation 发出成功信号，CloudFormation 服务将指示 EC2 自动扩展服务终止 ECS 容器实例 A，因为 ECS 容器实例 C 现在已加入 EC2 自动扩展组和 ECS 集群。

1.  在终止实例之前，EC2 自动扩展服务触发一个生命周期挂钩事件，将此事件发布到配置的简单通知服务（SNS）主题。SNS 是一种发布/订阅样式的通知服务，可用于各种用例，在我们的解决方案中，我们将订阅一个 Lambda 函数到 SNS 主题。

1.  Lambda 函数是由 SNS 主题调用的，以响应生命周期挂钩事件被发布到主题。

1.  Lambda 函数指示 ECS 排空即将被终止的 ECS 容器实例。然后，该函数轮询 ECS 容器实例上正在运行的任务数量，等待任务数量为零后才认为排空过程完成。

1.  ECS 将正在运行在 ECS 容器实例 A 上的当前任务转移到具有空闲容量的其他容器实例。在上图中，由于 ECS 容器实例 C 最近被添加到集群中，因此正在运行在 ECS 容器实例 A 上的 ECS 任务可以被转移到容器实例 C。请注意，如果容器实例 C 尚未添加到集群中，集群中将没有足够的容量来转移容器实例 A，因此确保集群具有足够的容量来处理这些类型的事件非常重要。

1.  在许多情况下，ECS 容器实例的排空可能会超过 Lambda 的当前五分钟执行超时限制。在这种情况下，您可以简单地重新发布生命周期挂钩事件通知到 SNS 主题，这将自动重新调用 Lambda 函数。

1.  Lambda 函数再次指示 ECS 排空容器实例 A（已在进行中），并继续轮询运行任务数量，等待运行任务数量为零。

1.  假设容器实例完成排空并且运行任务数量减少为零，Lambda 函数会向 EC2 自动扩展服务发出生命周期挂钩已完成的信号。

1.  EC2 自动缩放服务现在终止 ECS 容器实例，因为生命周期挂钩已经完成。

此时，由 CloudFormation 在步骤 1 中发起的滚动更新已经完成了 50%，因为旧的 ECS 容器实例 A 已被 ECS 容器实例 C 替换。在前面的图表中描述的过程再次重复，引入了一个新的 ECS 容器实例到集群中，并将 ECS 容器实例 B 标记为终止。一旦 ECS 容器实例 B 的排空完成，自动缩放组/集群中的所有实例都已被替换，滚动更新完成。

# 构建一个新的 ECS 容器实例 AMI

为了测试我们的生命周期管理解决方案，我们需要有一种机制来强制终止您的 ECS 容器实例。虽然您可以简单地调整自动缩放组的期望计数（实际上这是自动缩放组缩减时的常见情况），但另一种常见情况是当您需要通过引入一个新构建的 Amazon Machine Image（AMI）来更新您的 ECS 容器实例，其中包括最新的操作系统和安全补丁，以及最新版本的 Docker Engine 和 ECS 代理。至少，如果您正在使用类似于第六章中学到的方法构建自定义 ECS 容器实例 AMI，那么每当 Amazon 发布基本 ECS 优化 AMI 的新版本时，您都应该重新构建您的 AMI，并且每周或每月更新您的 AMI 是常见做法。

要模拟将新的 AMI 引入 ECS 集群，您可以简单地执行第六章中执行的相同步骤，这将输出一个新的 AMI，然后您可以将其作为输入用于您的堆栈，并强制您的 ECS 集群升级每个 ECS 容器实例。

以下示例演示了从**packer-ecs**存储库的根目录运行`make build`命令，这将输出一个新的 AMI ID，用于新创建和发布的镜像。确保您记下这个 AMI ID，因为您稍后在本章中会需要它：

```
> export AWS_PROFILE=docker-in-aws
> make build
packer build packer.json
amazon-ebs output will be in this color.

==> amazon-ebs: Prevalidating AMI Name: docker-in-aws-ecs 1518934269
...
...
Build 'amazon-ebs' finished.

==> Builds finished. The artifacts of successful builds are:
--> amazon-ebs: AMIs were created:
us-east-1: ami-77893508
```

运行 Packer 构建

# 配置 EC2 自动缩放滚动更新

当您使用 CloudFormation 创建和管理您的 EC2 自动扩展组时，一个有用的功能是能够管理滚动更新。滚动更新是指以受控的方式将新的 EC2 实例*滚入*您的自动扩展组，以确保您的更新过程可以在不引起中断的情况下完成。在第八章，当您通过 CloudFormation 创建 EC2 自动扩展组时，您了解了 CloudFormation 支持创建策略，可以帮助您确保 EC2 自动扩展中的所有实例都已成功初始化。CloudFormation 还支持更新策略，正如您在前面的图表中看到的那样，它可以帮助您管理和控制对 EC2 自动扩展组的更新。

如果您打开 todobackend-aws 存储库并浏览到`stack.yml`文件中的 CloudFormation 模板，您可以向`ApplicationAutoscaling`资源添加更新策略，如以下示例所示：

```
...
...
Resources:
  ...
  ...
  ApplicationAutoscaling:
    Type: AWS::AutoScaling::AutoScalingGroup
    CreationPolicy:
      ResourceSignal:
        Count: !Ref ApplicationDesiredCount
        Timeout: PT15M
    UpdatePolicy:
 AutoScalingRollingUpdate:
 MinInstancesInService: !Ref ApplicationDesiredCount
 MinSuccessfulInstancesPercent: 100
 WaitOnResourceSignals: "true"
 PauseTime: PT15M
  ...
  ...
```

配置 CloudFormation 自动扩展组更新策略

在上面的示例中，`UpdatePolicy`设置应用于`ApplicationAutoscaling`资源，该资源配置 CloudFormation 根据以下`AutoScalingRollingUpdate`配置参数来编排滚动更新，每当自动扩展组中的实例需要被替换（*更新*）时：

+   `MinInstancesInService`：在滚动更新期间必须处于服务状态的最小实例数。这里的标准方法是指定自动扩展组的期望计数，这意味着自动扩展将临时增加大小，以便在添加新实例时保持所需实例的最小数量。

+   `MinSuccessfulInstancesPercent`：必须成功部署的新实例的最低百分比，以便将滚动更新视为成功。如果未达到此百分比，则 CloudFormation 将回滚堆栈更改。

+   `WaitOnResourceSignals`：当设置为 true 时，指定 CloudFormation 在考虑实例成功部署之前等待每个实例发出的成功信号。这需要您的 EC2 实例在第六章安装并在第七章配置的`cfn-bootstrap`脚本向 CloudFormation 发出信号，表示实例初始化已完成。

+   `PauseTime`：当配置了`WaitOnResourceSignals`时，指定等待每个实例发出 SUCCESS 信号的最长时间。此值以 ISO8601 格式表示，在下面的示例中配置为等待最多 15 分钟。

然后，使用`aws cloudformation deploy`命令部署您的更改，如下例所示，您的自动扩展组现在将应用更新策略：

```
> export AWS_PROFILE=docker-in-aws
> aws cloudformation deploy --template-file stack.yml \
 --stack-name todobackend --parameter-overrides $(cat dev.cfg) \
 --capabilities CAPABILITY_NAMED_IAM
Enter MFA code for arn:aws:iam::385605022855:mfa/justin.menga:

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - todobackend
  ...
  ...
```

配置 CloudFormation 自动扩展组更新策略

此时，您现在可以更新堆栈以使用您在第一个示例中创建的新 AMI。这需要您首先更新 todobackend-aws 存储库根目录下的`dev.cfg`文件：

```
ApplicationDesiredCount=1
ApplicationImageId=ami-77893508
ApplicationImageTag=5fdbe62
ApplicationSubnets=subnet-a5d3ecee,subnet-324e246f
VpcId=vpc-f8233a80
```

更新 ECS AMI

然后，使用相同的`aws cloudformation deploy`命令部署更改。

在部署运行时，如果您打开 AWS 控制台，浏览到 CloudFormation 仪表板，并选择 todobackend 堆栈**事件**选项卡，您应该能够看到 CloudFormation 如何执行滚动更新：

CloudFormation 滚动更新

在前面的屏幕截图中，您可以看到 CloudFormation 首先临时增加了自动扩展组的大小，因为它需要始终保持至少一个实例在服务中。一旦新实例向 CloudFormation 发出 SUCCESS 信号，自动扩展组中的旧实例将被终止，滚动更新就完成了。

此时，您可能会感到非常高兴——只需对 CloudFormation 配置进行小小的更改，您就能够为堆栈添加滚动更新。不过，有一个问题，就是旧的 EC2 实例被立即终止。这实际上会导致服务中断，如果您导航到 CloudWatch 控制台，选择指标，在所有指标选项卡中选择 ECS **|** ClusterName，然后选择名为 todobackend-cluster 的集群的 MemoryReservation 指标，您可以看到这种迹象。

在您单击图形化指标选项卡并将统计列更改为最小值，周期更改为 1 分钟后，将显示以下屏幕截图：

ECS 内存预留

如果您回顾之前的屏幕截图中的时间线，您会看到在 21:17:33 旧的 ECS 容器实例被终止，在之前的屏幕截图中，您可以看到集群内存预留在 21:18（09:18）降至 0%。这表明在这个时间点上，没有实际的容器在运行，因为集群内存保留的百分比为 0，这表明在旧实例突然终止后，ECS 尝试将 todobackend 服务恢复到新的 ECS 容器实例时出现了短暂的中断。

因为最小的 CloudWatch 指标分辨率是 1 分钟，如果 ECS 能够在一分钟内恢复 ECS 服务，您可能无法观察到在前一个图表中降至 0%的情况，但请放心，您的应用程序确实会中断。

显然，这并不理想，正如我们之前讨论的那样，我们现在需要引入 EC2 自动扩展生命周期挂钩来解决这种情况。

# 创建 EC2 自动扩展生命周期挂钩

为了解决 EC2 实例终止影响我们的 ECS 服务的问题，我们现在需要创建一个 EC2 自动扩展生命周期挂钩，它将通知我们 EC2 实例即将被终止。回顾第一个图表，这需要几个资源：

+   实际的生命周期挂钩

+   授予 EC2 自动扩展组权限向 SNS 主题发布生命周期挂钩通知的生命周期挂钩角色

+   SNS 主题，生命周期挂钩可以发布和订阅

以下示例演示了创建生命周期挂钩、生命周期挂钩角色和 SNS 主题：

```
...
...
Resources:
  ...
  ...
 LifecycleHook:
 Type: AWS::AutoScaling::LifecycleHook
 Properties:
 RoleARN: !Sub ${LifecycleHookRole.Arn}
 AutoScalingGroupName: !Ref ApplicationAutoscaling
 DefaultResult: CONTINUE
 HeartbeatTimeout: 900
 LifecycleTransition: autoscaling:EC2_INSTANCE_TERMINATING
 NotificationTargetARN: !Ref LifecycleHookTopic
 LifecycleHookRole:
 Type: AWS::IAM::Role
 Properties:
 AssumeRolePolicyDocument:
 Version: "2012-10-17"
 Statement:
 - Action:
 - sts:AssumeRole
 Effect: Allow
 Principal:
 Service: autoscaling.amazonaws.com
 Policies:
- PolicyName: LifecycleHookPermissions
 PolicyDocument:
 Version: "2012-10-17"
 Statement:
 - Sid: PublishNotifications
 Action: 
 - sns:Publish
 Effect: Allow
 Resource: !Ref LifecycleHookTopic
 LifecycleHookTopic:
 Type: AWS::SNS::Topic
 Properties: {}
  LifecycleHookSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Endpoint: !Sub ${LifecycleHookFunction.Arn}
      Protocol: lambda
      TopicArn: !Ref LifecycleHookTopic    ...
    ...

```

在 CloudFormation 中创建生命周期挂钩资源

在前面的示例中，`LifecycleHook`资源创建了一个新的钩子，该钩子与`ApplicationAutoscaling`资源相关联，使用`AutoScalingGroupName`属性，并由 EC2 实例触发，这些实例即将被终止，如`LifecycleTransition`属性配置的`autoscaling:EC2_INSTANCE_TERMINATING`值所指定的那样。该钩子配置为向名为`LifecycleHookTopic`的新 SNS 主题资源发送通知，链接的`LifecycleHookRole` IAM 角色授予`autoscaling.amazonaws.com`服务（如角色的`AssumeRolePolicyDocument`部分中所指定的）权限，以将生命周期钩子事件发布到此主题。`DefaultResult`属性指定了在`HeartbeatTimeout`期间到达并且没有收到钩子响应时应创建的默认结果，例如，在本示例中，发送一个`CONTINUE`消息，指示 Auto Scaling 服务继续处理可能已注册的任何其他生命周期钩子。`DefaultResult`属性的另一个选项是发送一个`ABANDON`消息，这仍然指示 Auto Scaling 服务继续进行实例终止，但放弃处理可能配置的任何其他生命周期钩子。

最终的`LifecycleHookSubscription`资源创建了对`LifecycleHookTopic` SNS 主题资源的订阅，订阅了一个名为`LifecycleHookFunction`的 Lambda 函数资源，我们将很快创建，这意味着每当消息发布到 SNS 主题时，将调用此函数。

# 创建用于消耗生命周期钩子的 Lambda 函数

有了各种生命周期钩子资源，谜题的最后一块是创建一个 Lambda 函数和相关资源，该函数将订阅您在上一节中定义的生命周期钩子 SNS 主题，并最终在发出信号表明生命周期钩子操作可以继续之前执行 ECS 容器实例排空。

让我们首先关注 Lambda 函数本身以及它将需要执行的相关源代码：

```
...
...
Resources: LifecycleHookFunction:
    Type: AWS::Lambda::Function
    DependsOn:
      - LifecycleHookFunctionLogGroup
    Properties:
      Role: !Sub ${LifecycleFunctionRole.Arn}
      FunctionName: !Sub ${AWS::StackName}-lifecycleHooks
      Description: !Sub ${AWS::StackName} Autoscaling Lifecycle Hook
      Environment:
        Variables:
          ECS_CLUSTER: !Ref ApplicationCluster
      Code:
        ZipFile: |
          import os, time
          import json
          import boto3
          cluster = os.environ['ECS_CLUSTER']
          # AWS clients
          ecs = boto3.client('ecs')
          sns = boto3.client('sns')
          autoscaling = boto3.client('autoscaling')

          def handler(event, context):
            print("Received event %s" % event)
            for r in event.get('Records'):
              # Parse SNS message
              message = json.loads(r['Sns']['Message'])
              transition, hook = message['LifecycleTransition'], message['LifecycleHookName']
              group, ec2_instance = message['AutoScalingGroupName'], message['EC2InstanceId']
              if transition != 'autoscaling:EC2_INSTANCE_TERMINATING':
                print("Ignoring lifecycle transition %s" % transition)
                return
              try:
                # Get ECS container instance ARN
                ecs_instance_arns = ecs.list_container_instances(
                  cluster=cluster
                )['containerInstanceArns']
                ecs_instances = ecs.describe_container_instances(
                  cluster=cluster,
                  containerInstances=ecs_instance_arns
                )['containerInstances']
                # Find ECS container instance with same EC2 instance ID in lifecycle hook message
                ecs_instance_arn = next((
                  instance['containerInstanceArn'] for instance in ecs_instances
                  if instance['ec2InstanceId'] == ec2_instance
                ), None)
                if ecs_instance_arn is None:
                  raise ValueError('Could not locate ECS instance')
                # Drain instance
                ecs.update_container_instances_state(
                  cluster=cluster,
                  containerInstances=[ecs_instance_arn],
                  status='DRAINING'
                )
                # Check task count on instance every 5 seconds
                count = 1
                while count > 0 and context.get_remaining_time_in_millis() > 10000:
                  status = ecs.describe_container_instances(
                    cluster=cluster,
                    containerInstances=[ecs_instance_arn],
                  )['containerInstances'][0]
                  count = status['runningTasksCount']
                  print("Sleeping...")
                  time.sleep(5)
                if count == 0:
                  print("All tasks drained - sending CONTINUE signal")
                  autoscaling.complete_lifecycle_action(
                    LifecycleHookName=hook,
                    AutoScalingGroupName=group,
                    InstanceId=ec2_instance,
                    LifecycleActionResult='CONTINUE'
                  )
                else:
                  print("Function timed out - republishing SNS message")
                  sns.publish(TopicArn=r['Sns']['TopicArn'], Message=r['Sns']['Message'])
              except Exception as e:
                print("A failure occurred with exception %s" % e)
                autoscaling.complete_lifecycle_action(
                  LifecycleHookName=hook,
                  AutoScalingGroupName=group,
                  InstanceId=ec2_instance,
                  LifecycleActionResult='ABANDON'
                )
      Runtime: python3.6
      MemorySize: 128
      Timeout: 300
      Handler: index.handler
  LifecycleHookFunctionLogGroup:
    Type: AWS::Logs::LogGroup
    DeletionPolicy: Delete
    Properties:
      LogGroupName: !Sub /aws/lambda/${AWS::StackName}-lifecycleHooks
      RetentionInDays: 7    ...
    ...

```

创建用于处理生命周期钩子的 Lambda 函数

Lambda 函数比我们迄今为止处理的要复杂一些，但如果您有 Python 经验，它仍然是一个相对简单的函数，应该相对容易理解。

该函数首先定义所需的库，并查找名为`ECS_CLUSTER`的环境变量，这是必需的，以便函数知道生命周期挂钩与哪个 ECS 集群相关，并且通过 Lambda 函数资源的`Environment`属性传递此环境变量值。

接下来，函数声明了三个 AWS 客户端：

+   `ecs`：与 ECS 通信，以审查 ECS 容器实例信息并根据生命周期挂钩中接收的 EC2 实例 ID 排空正确的实例。

+   `autoscaling`：在生命周期挂钩可以继续时，向 EC2 自动缩放服务发出信号。

+   `sns`：如果 Lambda 函数即将达到最长五分钟的执行超时，并且 ECS 容器实例尚未排空，则重新发布生命周期挂钩事件。这将再次调用 Lambda 函数，直到 ECS 容器实例完全排空。

`handler`方法定义了 Lambda 函数的入口点，并首先提取出许多变量，这些变量从接收到的 SNS 消息中捕获信息，包括生命周期挂钩事件类型（`transition`变量）、挂钩名称（`hook`变量）、Auto Scaling 组名称（`group`变量）和 EC2 实例 ID（`ec2_instance`变量）。然后立即进行检查，以验证生命周期挂钩事件类型是否与 EC2 实例终止事件相关，如果事件类型（在 transition 变量中捕获）不等于值`autoscaling:EC2_INSTANCE_TERMINATING`，则函数立即返回，有效地忽略该事件。

假设事件确实与 EC2 实例的终止有关，处理程序接下来通过`ecs`客户端查询 ECS 服务，首先描述配置集群中的所有实例，然后尝试定位与生命周期挂钩事件捕获的 EC2 实例 ID 匹配的 ECS 容器实例。如果找不到实例，则会引发`ValueError`异常，该异常将被 catch 语句捕获，导致记录错误并使用`ABANDON`的结果完成生命周期挂钩。如果找到实例，处理程序将继续通过在`ecs`客户端上调用`update_container_instances_state()`方法来排水实例，该方法将实例的状态设置为`DRAINING`，这意味着 ECS 将不再将任何新任务调度到该实例，并尝试将现有任务迁移到集群中的其他实例。在这一点上，处理程序需要等待在实例上运行的所有当前 ECS 任务被排水，这可以通过每五秒轮询一次 ECS 任务计数的`while`循环来实现，直到任务计数减少到零。您可以无限期地尝试这样做，但是在撰写本文时，Lambda 具有最长五分钟的执行时间限制，因此`while`循环使用`context.get_remaining_time_in_millis()`方法来检查 Lambda 执行超时是否即将到达。

`context`对象是由 Lambda 运行时环境传递给处理程序方法的对象，其中包括有关 Lambda 环境的信息，包括内存、CPU 和剩余执行时间。

如果任务计数减少到零，您可以安全地终止 ECS 容器实例，自动缩放客户端将使用`CONTINUE`的结果完成生命周期挂钩，这意味着 EC2 自动缩放服务将继续处理任何其他注册的挂钩并终止实例。如果任务计数在函数即将退出之前没有减少到零，则函数只是重新发布原始的生命周期挂钩通知，这将重新启动函数。由于函数中的所有操作都是幂等的，即更新已经处于排水状态的 ECS 容器实例的状态为 DRAINING 会导致相同的排水状态，因此这种方法是安全的，也是克服 Lambda 执行超时限制的一种非常简单而优雅的方法。

# 为生命周期挂钩 Lambda 函数配置权限

Lambda 函数现在已经就位，最后的配置任务是为 Lambda 函数执行的各种 API 调用和操作添加所需的权限：

```
...
...
Resources: LifecycleHookPermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref LifecycleHookFunction
      Principal: sns.amazonaws.com
      SourceArn: !Ref LifecycleHookTopic
  LifecycleFunctionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Action:
              - sts:AssumeRole
            Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
      Policies:
        - PolicyName: LifecycleHookPermissions
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: ListContainerInstances
                Effect: Allow
                Action:
                  - ecs:ListContainerInstances
                Resource: !Sub ${ApplicationCluster.Arn}
              - Sid: ManageContainerInstances
                Effect: Allow
                Action:
                  - ecs:DescribeContainerInstances
                  - ecs:UpdateContainerInstancesState
                Resource: "*"
                Condition:
                  ArnEquals:
                    ecs:cluster: !Sub ${ApplicationCluster.Arn}
              - Sid: Publish
                Effect: Allow
                Action:
                  - sns:Publish
                Resource: !Ref LifecycleHookTopic
              - Sid: CompleteLifecycleAction
                Effect: Allow
                Action:
                  - autoscaling:CompleteLifecycleAction
                Resource: !Sub arn:aws:autoscaling:${AWS::Region}:${AWS::AccountId}:autoScalingGroup:*:autoScalingGroupName/${ApplicationAutoscaling}
              - Sid: ManageLambdaLogs
                Effect: Allow
                Action:
                - logs:CreateLogStream
                - logs:PutLogEvents
                Resource: !Sub ${LifecycleHookFunctionLogGroup.Arn}    LifecycleHookFunction:
      Type: AWS::Lambda::Function
    ...
    ...

```

为生命周期挂钩 Lambda 函数配置权限

在前面的示例中，需要一个名为`LifecycleHookPermission`的资源，类型为`AWS::Lambda::Permission`，它授予 SNS 服务（由`Principal`属性引用）调用 Lambda 函数（由`LambdaFunction`属性引用）的权限，用于 SNS 主题发布的通知（由`SourceArn`属性引用）。每当您需要授予另一个 AWS 服务代表您调用 Lambda 函数的能力时，通常需要采用这种配置权限的方法，尽管也有例外情况（例如 CloudFormation 自定义资源用例，其中 CloudFormation 隐含具有这样的权限）。

您还需要为 Lambda 函数创建一个名为`LambdaFunctionRole`的 IAM 角色，该角色授予函数执行各种任务和操作的能力，包括：

+   列出、描述和更新应用程序集群中的 ECS 容器实例

+   如果 Lambda 函数即将超时，则重新发布生命周期挂钩事件到 SNS

+   在 ECS 容器实例排空后完成生命周期操作

+   将日志写入 CloudWatch 日志

# 部署和测试自动扩展生命周期挂钩

您现在可以使用`aws cloudformation deploy`命令部署完整的自动扩展生命周期挂钩解决方案，就像本章前面演示的那样。

部署完成后，为了测试生命周期管理是否按预期工作，您可以执行一个简单的更改，强制替换 ECS 集群中当前的 ECS 容器实例，即恢复您在本章前面所做的 AMI 更改：

```
ApplicationDesiredCount=1
ApplicationImageId=ami-ec957491
ApplicationImageTag=5fdbe62
ApplicationSubnets=subnet-a5d3ecee,subnet-324e246f
VpcId=vpc-f8233a80
```

恢复 ECS AMI

现在，一旦您部署了这个更改，再次使用`aws cloudformation deploy`命令，就像之前的示例演示的那样，接下来切换到 CloudFormation 控制台，当事件引发终止现有的 EC2 实例时，快速导航到 ECS 仪表板并选择您的 ECS 集群。在容器实例选项卡上，您应该看到您的 ECS 容器实例中的一个状态正在排空，如下面的屏幕截图所示，一旦所有任务从这个实例中排空，生命周期挂钩函数将向 EC2 自动扩展服务发出信号，以继续终止实例：

![](img/48e994f8-8700-4055-85be-dce072cd5887.png)ECS 容器实例排空

如果您重复执行前面屏幕截图中的步骤，以查看 ECS 容器实例在排空和终止期间的集群内存保留量，您应该会看到一个类似下面示例中的图表：

![](img/6ce7187e-5fa4-4457-bea3-ce1220c04bb1.png)ECS 容器实例排空期间的集群内存保留

在前面的屏幕截图中，请注意在滚动更新期间，集群内存保留量从未降至 0％。由于在滚动升级期间集群中有两个实例，内存利用率百分比确实会发生变化，但我们排空 ECS 容器实例的能力确保了在集群上运行的应用程序的不间断服务。

作为最后的检查，您还可以导航到生命周期挂钩函数的 CloudWatch 日志组，如下面的屏幕截图所示：

![](img/5a4701c4-7ce4-4900-9301-222be446dc52.png)生命周期挂钩函数日志

在前面的屏幕截图中，您可以看到该函数在容器实例排空时定期休眠，大约两分钟后，在这种情况下，所有任务排空并且函数向自动扩展服务发送`CONTINUE`信号以继续挂钩。

# 摘要

在本章中，您创建了一个解决方案，用于管理 ECS 容器实例的生命周期，并确保在需要终止和替换 ECS 集群中的 ECS 容器实例时，运行在 ECS 集群上的应用程序和服务不会受到影响。

您学习了如何通过利用 CloudFormation 更新策略来配置 EC2 自动扩展组的滚动更新，从而控制新实例如何以滚动方式添加到您的自动扩展组。您发现这个功能在自动扩展和 EC2 实例级别上运行良好，但是您发现在集群中突然终止现有 ECS 容器实例会导致应用程序中断。

为了解决这个挑战，您创建了一个注册为`EC2_INSTANCE_TERMINATING`事件的 EC2 生命周期挂钩，并配置此挂钩以将通知发布到 SNS 主题，然后触发一个 Lambda 函数。该函数负责定位与即将终止的 EC2 实例相关联的 ECS 容器实例，排空容器实例，然后等待直到 ECS 任务计数达到 0，表示实例上的所有 ECS 任务都已终止并替换。如果 ECS 容器实例的执行时间超过 Lambda 函数的五分钟最大执行时间，您学会了可以简单地重新发布包含生命周期挂钩信息的 SNS 事件，这将触发函数的新调用，这个过程可以无限期地继续，直到实例上的 ECS 任务计数达到 0。

在下一章中，您将学习如何动态管理 ECS 集群的容量，这对支持应用程序的自动扩展要求至关重要。这涉及不断向您的 ECS 集群添加和删除 ECS 容器实例，因此您可以看到，本章介绍的 ECS 容器实例生命周期机制对确保您的应用程序不受任何自动扩展操作影响至关重要。

# 问题

1.  真/假：当您终止 ECS 容器实例时，该实例将自动将运行的 ECS 任务排空到集群中的另一个实例。

1.  您可以接收哪些类型的 EC2 自动扩展生命周期挂钩？

1.  一旦完成处理 EC2 自动扩展生命周期挂钩，您可以发送哪些类型的响应？

1.  真/假：EC2 自动扩展生命周期挂钩可以向 AWS Kinesis 发布事件。

1.  您创建了一个处理生命周期挂钩并排空 ECS 容器实例的 Lambda 函数。您注意到有时这需要大约 4-5 分钟，但通常需要 15 分钟。您可以采取什么措施来解决这个问题？

1.  您可以配置哪个 CloudFormation 功能以启用自动扩展组的滚动更新？

1.  您想要执行滚动更新，并确保在更新期间始终至少有当前所需数量的实例在服务中。您将如何实现这一点？

1.  在使用 CloudFormation 订阅 Lambda 函数到 SNS 主题时，您需要创建什么类型的资源以确保 SNS 服务具有适当的权限来调用函数？

# 进一步阅读

您可以查看以下链接以获取有关本章涵盖的主题的更多信息：

+   CloudFormation UpdatePolicy 属性：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatepolicy.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatepolicy.html)

+   Amazon EC2 自动扩展生命周期挂钩：[`docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks.html`](https://docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks.html)

+   CloudFormation 生命周期挂钩资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-as-lifecyclehook.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-as-lifecyclehook.html)

+   CloudFormation SNS 主题资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-sns-topic.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-sns-topic.html)

+   CloudFormation SNS 订阅资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-sns-subscription.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-sns-subscription.html)

+   CloudFormation Lambda 权限资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-permission.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-permission.html)

+   CloudFormation ECS 任务定义资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-taskdefinition.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-taskdefinition.html)

+   CloudFormation ECS 服务资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-service.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-service.html)

+   CloudFormation Lambda 函数资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-function.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-function.html)

+   CloudFormation Lambda 函数代码：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-lambda-function-code.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-lambda-function-code.html)

+   CloudFormation 自定义资源文档：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)

+   CloudFormation 自定义资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref.html)

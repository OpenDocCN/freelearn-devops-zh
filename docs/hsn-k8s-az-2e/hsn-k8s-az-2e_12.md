# 第九章：连接到 Azure 事件中心

基于事件的集成是实现微服务的关键模式。微服务架构的理念是将单体应用程序分解为一组较小的服务。事件通常用于协调这些不同的服务之间。当您考虑一个事件时，它可以是许多事物之一。金融交易可以是一个事件，同样的，IoT 传感器数据、网页点击和浏览等也可以是事件。

处理这些类型事件的常用软件是 Apache Kafka（简称 Kafka）。Kafka 最初由 LinkedIn 开发，后来捐赠给 Apache 软件基金会。它是一个流行的开源流平台。流平台具有三个核心能力：发布和订阅消息流（类似于队列），以持久方式存储这些流，并在发生时处理这些流。

Azure 有一个类似于 Apache Kafka 的服务，称为 Azure 事件中心。事件中心是一个提供实时数据摄入的托管服务。它易于设置和使用，并且可以动态扩展。事件中心还与其他 Azure 服务集成，如流分析、函数和 databricks。这些预构建的集成使您更容易构建从事件中心消费事件的应用程序。

事件中心还提供 Kafka 端点。这意味着您可以配置现有的基于 Kafka 的应用程序，并将其指向事件中心。使用事件中心来处理 Kafka 应用程序的好处是您不再需要管理 Kafka 集群，因为您可以将其作为托管服务来使用。

在本章中，您将学习如何在 AKS 上实现微服务，并使用事件中心在应用程序之间实现松耦合集成。您将部署一个使用 Kafka 发送事件的应用程序，并用 Azure 事件中心替换您自己的 Kafka 集群。正如您将在本章中学到的，基于事件的集成是单体和基于微服务的应用程序之间的关键区别之一。

+   部署一组微服务

+   使用 Azure 事件中心

我们将从部署一组构建社交网络的微服务开始本章。

## 部署一组微服务

在本节中，我们将部署一个来自名为社交网络的演示应用程序的一组微服务。该应用程序由两个主要的微服务组成：**用户**和**好友**。用户服务将所有用户存储在自己的数据存储中。用户由 ID、名和姓表示。好友服务存储用户的好友。好友关系链接了两个好友的用户 ID，并且还有自己的 ID。

添加用户/添加好友的事件被发送到消息队列。该应用程序使用 Kafka 作为消息队列，用于存储与用户、好友和推荐相关的事件。

这个队列被一个推荐服务所消费。这个服务由一个**Neo4j**数据库支持，可以用来查询用户之间的关系。Neo4j 是一个流行的图形数据库平台。图形数据库不同于典型的关系数据库，如 MySQL。图形数据库专注于存储不同元素之间的关系。您可以用问题查询图形数据库，比如*给我用户 X 和用户 Y 的共同好友*。

在数据流方面，您可以创建用户和友谊关系。创建用户或友谊关系将在消息队列上生成一条消息，这将导致数据在 Neo4j 数据库中填充。该应用程序没有 Web 界面。您主要将使用命令行与应用程序进行交互，尽管我们可以连接到 Neo4j 数据库以验证数据是否已填充到数据库中。

在接下来的部分，您将学习以下内容：

+   使用 Helm 部署一个示例基于微服务的应用程序。

+   通过发送事件并观察对象的创建和更新来测试服务。

让我们从部署应用程序开始。

### 使用 Helm 部署应用程序

在本节中，我们将使用 Helm 部署演示应用程序。这将使用本地 Kafka 实例部署完整的应用程序。应用程序部署后，我们将生成一个小型社交网络，并验证我们能够创建该社交网络。

1.  这个示例有很多资源需求。为了满足这些需求，将您的集群扩展到四个节点：

```
az aks nodepool scale --node-count 4 -g rg-handsonaks \
  --cluster-name handsonaks --name agentpool
```

1.  这个示例的代码已经包含在本书的 GitHub 存储库中。您可以在`Chapter09`文件夹下的`social-network`文件夹中找到代码。导航到这个文件夹：

```
cd Chapter09/social-network
```

1.  要运行 Kafka，我们还需要运行**ZooKeeper**。ZooKeeper 是 Apache 基金会的另一个开源软件项目。它提供命名、配置管理、同步和分组服务的能力。我们将使用`bitnami`的 Kafka 和 ZooKeeper Helm 图表，因此让我们添加所需的 Helm 存储库：

```
helm repo add bitnami https://charts.bitnami.com
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com
```

这将生成如*图 9.1*所示的输出：

![输出屏幕显示两个 Helm 存储库，bitnami 和 incubator，被添加到您的存储库中。](img/Figure_9.1.jpg)

###### 图 9.1：添加 Helm 存储库

1.  让我们更新依赖项以使依赖图表可用：

```
helm dep update deployment/helm/social-network
helm dep update deployment/helm/friend-service
helm dep update deployment/helm/user-service
helm dep update deployment/helm/recommendation-service
```

这将显示类似于*图 9.2*的内容四次：

![为了使依赖图表可用，输出屏幕将显示成功接收来自 svc-cat、incubator、azure、jetstack、bitnami 和 stable 等四个依赖项更新的单独消息。](img/Figure_9.2.jpg)

###### 图 9.2：更新依赖项

#### 注意

在此示例中，您可能会看到类似于以下内容的警告：`walk.go:74: found symbolic link in path:`。这是一个可以安全忽略的警告。

1.  接下来，为此应用程序创建一个新的`namespace`：

```
kubectl create namespace social-network
```

这将生成如下输出：

![使用 kubectl create namespace social-network 命令，创建一个新的命名空间。](img/Figure_9.3.jpg)

###### 图 9.3：创建一个新的命名空间

1.  现在，继续部署应用程序：

```
helm install social-network --namespace social-network \
  --set fullNameOverride=social-network \
  --set edge-service.service.type=LoadBalancer \
  deployment/helm/social-network
```

1.  使用以下命令检查部署中 Pod 的状态：

```
kubectl get pods -w -n social-network
```

正如您在*图 9.4*中所看到的，大约需要 5 分钟才能使所有的 Pod 都正常运行起来：

![输出显示共有 20 个状态为 Running 的 Pod。](img/Figure_9.4.jpg)

###### 图 9.4：显示所有状态为 Running 的 Pod 的输出

1.  应用程序成功部署后，您可以连接到边缘服务。要获取其 IP，请使用以下命令：

```
kubectl get service -n social-network
```

此命令将生成如下输出：

![您可以使用 kubectl get service -n social-network 命令获取 edge-service 的外部 IP。](img/Figure_9.5.jpg)

###### 图 9.5：获取边缘服务的外部 IP

1.  您可以进行两个测试来验证应用程序是否正常工作。测试 1 是在浏览器中连接端口`9000`上的边缘服务。*图 9.6*显示了 Whitelabel 错误页面，显示应用程序正在运行：![Whitelabel 错误页面将显示应用程序没有配置错误视图，因此您看到这个作为后备。这表明应用程序正在运行。](img/Figure_9.6.jpg)

###### 图 9.6：Whitelabel 错误页面，显示应用程序正在运行

1.  验证应用程序是否运行的第二个测试是实际生成一个小型社交网络。这将验证所有服务是否正常工作。您可以使用以下命令创建这个网络：

```
bash ./deployment/sbin/generate-serial.sh <external-ip>:9000
```

这个命令将生成大量输出。输出将以*图 9.7*中显示的元素开头：

![在创建新的社交网络时，初始输出屏幕将显示名字、姓氏、创建日期和时间、上次修改日期和时间以及一个 ID。](img/Figure_9.7.jpg)

###### 图 9.7：创建新的社交网络时的初始输出

1.  生成一个 15 人的网络大约需要一分钟的时间。要验证网络是否已成功创建，请在您的网络浏览器中浏览到[`http://<external-ip>:9000/user/v1/users/1`](http://<external-ip>:9000/user/v1/users/1)。这应该会显示一个代表社交网络中用户的小 JSON 对象，如*图 9.8*所示：![在用户服务中成功创建用户后，访问<external-ip>:9000/user/v1/users/1 的输出屏幕将显示用户的名字和姓氏、创建日期和时间以及上次修改日期和时间。](img/Figure_9.8.jpg)

###### 图 9.8：用户服务中成功创建用户

1.  最后，您可以连接到 Neo4j 数据库并可视化您创建的社交网络。要能够连接到 Neo4j，您需要首先将其公开为一个服务。使用社交网络文件夹中的`neo4j-service.yaml`文件来公开它：

```
kubectl create -f neo4j-service.yaml -n social-network
```

然后，获取服务的公共 IP 地址。这可能需要大约一分钟才能使用：

```
kubectl get service neo4j-service -n social-network 
```

上述命令将生成以下输出：

![输出显示了 Neo4j 服务的外部 IP 地址。](img/Figure_9.9.jpg)

###### 图 9.9：显示 Neo4j 服务的外部 IP 地址的输出

#### 注意

请注意，Neo4j 服务的外部 IP 可能与边缘服务的外部 IP 不同。

1.  使用浏览器连接到[`http://<external-ip>:7474`](http://<external-ip>:7474)。这将打开一个登录屏幕。使用以下信息登录：

+   **连接 URL**：`bolt://<external-ip>:7678`

+   **用户名**：`neo4j`

+   **密码**：`neo4j`

您的连接信息应该类似于*图 9.10*：

![要登录 Neo4j 服务，用户必须在连接 URL 字段中输入 bolt://<external-ip>:7687，将用户名设置为 neo4j，密码设置为 neo4j。然后他们需要点击密码字段下面的连接选项卡。这将帮助他们登录浏览器。](img/Figure_9.10.jpg)

###### 图 9.10：登录到 Neo4j 浏览器

1.  一旦连接到 Neo4j 浏览器，您可以看到实际的社交网络。点击**数据库信息**图标，然后点击**用户**。这将生成一个查询，显示您刚刚创建的社交网络。这将类似于*图 9.11*：![社交网络的输出屏幕分为两部分。屏幕的左侧显示数据库信息，如节点标签、关系类型、属性键和数据库，而屏幕的其余部分显示用户(14)和朋友(149)的名称。输出还包含所创建的社交网络的图形表示。](img/Figure_9.11.jpg)

###### 图 9.11：您刚刚创建的社交网络的视图

在当前示例中，我们已经设置了端到端的应用程序，使用在我们的 Kubernetes 集群上运行的 Kafka 作为消息队列。在我们进入下一节之前，让我们删除该示例。要删除本地部署，请使用以下命令：

```
helm delete social-network -n social-network
kubectl delete pvc -n social-network --all
kubectl delete pv --all
```

在下一节中，我们将摆脱在集群中存储事件，并将它们存储在 Azure 事件中心。通过利用 Azure 事件中心上的本机 Kafka 支持，并切换到使用更适合生产的事件存储，我们将看到这个过程是简单的。

## 使用 Azure 事件中心

自己在集群上运行 Kafka 是可能的，但对于生产使用可能很难运行。在本节中，我们将把维护 Kafka 集群的责任转移到 Azure 事件中心。事件中心是一个完全托管的实时数据摄入服务。它原生支持 Kafka 协议，因此，通过轻微修改，我们可以将我们的应用程序从使用本地 Kafka 实例更新为可扩展的 Azure 事件中心实例。

在接下来的几节中，我们将执行以下操作：

+   通过门户创建事件中心并收集连接我们基于微服务的应用程序所需的详细信息

+   修改 Helm 图表以使用新创建的事件中心

让我们开始创建事件中心。

### 创建事件中心

在本节中，我们将创建 Azure 事件中心。稍后我们将使用此事件中心来流式传输新消息。执行以下步骤创建事件中心：

1.  要在 Azure 门户上创建事件中心，请搜索`事件中心`，如*图 9.12*所示：![在搜索栏选项卡中，用户需要输入“事件中心”以在 Azure 门户中创建事件中心。](img/Figure_9.12.jpg)

###### 图 9.12：在搜索栏中查找事件中心

1.  点击**事件中心**。

1.  在**事件中心**选项卡上，点击**添加**，如*图 9.13*所示：![要添加新的事件中心，用户需要点击屏幕最左侧的+添加选项卡。](img/Figure_9.13.jpg)

###### 图 9.13：添加新的事件中心

1.  填写以下细节：

+   **名称**：此名称应是全局唯一的。考虑在名称中添加您的缩写。

+   **定价层**：选择标准定价层。基本定价层不支持 Kafka。

+   **使此命名空间区域多余**：已禁用。

+   **Azure 订阅**：选择与托管 Kubernetes 集群的订阅相同的订阅。

+   **资源组**：选择我们为集群创建的资源组，在我们的情况下是`rg-handsonaks`。

+   **位置**：选择与您的集群相同的位置。在我们的情况下，这是`West US 2`。

+   **吞吐量单位**：1 个单位足以进行此测试。

+   **自动膨胀**：已禁用。

这应该给您一个类似于*图 9.14*的创建视图：

![有各种需要填写的字段。定价层应设置为标准，使此命名空间区域冗余选项应禁用，吞吐量单位应设置为 1，自动膨胀选项应禁用，并且名称、订阅、资源组和位置字段也需要填写。在这些字段的底部，您将看到创建选项卡。这就是您的事件中心创建的样子。](img/Figure_9.14.jpg)

###### 图 9.14：您的事件中心创建应该是这样的

1.  点击向导底部的**创建**按钮来创建您的事件中心。

1.  一旦事件中心创建完成，选择它，如*图 9.15*所示：![一旦事件中心创建完成，您将被引导到一个窗口，在那里您可以在门户上查看事件中心。您需要通过点击事件中心的名称来选择它。](img/Figure_9.15.jpg)

###### 图 9.15：一旦创建，点击事件中心名称

1.  点击**共享访问策略**，选择**RootManageSharedAccessKey**，并复制**连接字符串-主密钥**，如*图 9.16*所示：![这个屏幕显示了获取事件中心连接字符串的过程。第一步是在左侧菜单中点击共享访问策略。第二步是选择 RootManageSharedAccessKey 策略，第三步是点击主连接字符串旁边的复制图标。](img/Figure_9.16.jpg)

###### 图 9.16：复制主连接字符串

使用 Azure 门户，我们已经创建了一个事件中心，可以存储和处理生成的事件。我们需要收集连接字符串，以便连接我们的基于微服务的应用程序。在下一节中，我们将重新部署我们的社交网络，并配置它连接到事件中心。为了能够部署我们的社交网络，我们将不得不对 Helm 图表进行一些更改，以便将我们的应用程序指向事件中心而不是 Kafka。

### 修改 Helm 文件

我们将把微服务部署从使用本地 Kafka 实例切换到使用 Azure 托管的、与 Kafka 兼容的事件中心实例。为了做出这个改变，我们将修改 Helm 图表，以使用事件中心而不是 Kafka：

1.  修改`values.yaml`文件，以禁用集群中的 Kafka，并包括连接细节到您的事件中心：

```
code deployment/helm/social-network/values.yaml
```

确保更改以下值：

**第 5、18、26 和 34 行**：将其更改为`enabled: false`。

**第 20、28 和 36 行**：将其更改为您的事件中心名称。

**第 21、29 和 37 行**：将其更改为您的事件中心连接字符串：

```
1   nameOverride: social-network
2   fullNameOverride: social-network
3
4   kafka:
5     enabled: true
6     nameOverride: kafka
7     fullnameOverride: kafka
8     persistence:
9       enabled: false
10    resources:
11      requests:
12        memory: 325Mi
13   
14   
15  friend-service:
16    fullnameOverride: friend-service
17    kafka:
18      enabled: true
19    eventhub:
20      name: <event hub name>
21      connection: "<event hub connection string>"
22   
23  recommendation-service:
24    fullnameOverride: recommendation-service
25    kafka:
26      enabled: true
27    eventhub:
28      name: <event hub name>
29      connection: "<event hub connection string>"
30
31  user-service:
32    fullnameOverride: user-service
33    kafka:
34      enabled: true
35    eventhub:
36      name: <event hub name>
37      connection: "<event hub connection string>"
```

#### 注意

对于我们的演示，我们将连接字符串存储在 Helm 值文件中。这不是最佳实践。对于生产用例，您应该将这些值存储为秘密，并在部署中引用它们。我们将在*第十章*，*保护您的 AKS 集群*中探讨这一点。

1.  按照以下方式运行部署：

```
helm install social-network deployment/helm/social-network/ -n social-network --set edge-service.service.type=LoadBalancer
```

1.  等待所有的 Pod 启动。您可以使用以下命令验证所有的 Pod 是否已经启动并正在运行：

```
kubectl get pods -n social-network
```

这将生成以下输出：

![屏幕上显示的 14 个 Pod 显示它们的运行状态为 Running。](img/Figure_9.17.jpg)

###### 图 9.17：输出显示所有 Pod 的运行状态

1.  要验证您是否连接到事件中心，而不是本地 Kafka，您可以在门户中检查事件中心，并检查不同的主题。您应该会看到一个 friend 和一个 user 主题，如*图 9.18*所示：![当您在屏幕左侧的菜单中滚动时，您会看到实体选项卡。点击其中的事件中心。您将看到在您的事件中心中创建的两个主题。这些主题的名称应该是 friend 和 user。](img/Figure_9.18.jpg)

###### 图 9.18：显示在您的事件中心创建的两个主题

1.  继续观察 Pod。当所有的 Pod 都已启动并正在运行时，获取边缘服务的外部 IP。您可以使用以下命令获取该 IP：

```
kubectl get svc -n social-network
```

1.  然后，运行以下命令验证实际社交网络的创建：

```
bash ./deployment/sbin/generate-serial.sh <external-ip>:9000
```

这将再次创建一个包含 15 个用户的社交网络，但现在将使用事件中心来发送所有与用户、好友和推荐相关的事件。

1.  您可以在 Azure 门户上看到这些活动。Azure 门户为事件中心创建了详细的监控图表。要访问这些图表，请点击**friend**事件中心，如*图 9.19*所示：![在屏幕左侧的导航窗格中，向下滚动到实体部分。点击事件中心。您会看到两个主题：friend 和 user。点击 friend 主题以获取更多指标。](img/Figure_9.19.jpg)

###### 图 9.19：点击 friend 主题以获取更多指标

在*图 9.20*中，您可以看到 Azure 门户为您提供了三个图表：请求的数量、消息的数量和总吞吐量：

![Azure 门户显示了主题的三个图表。这些高级图表提供了请求的数量，消息的数量和总吞吐量。这些图表的底部都有一个蓝色图标，表示传入请求，传入消息和传入字节。您还将看到一个橙色图标，表示成功的请求，传出消息和传出字节。图片中的图表显示了上升的尖峰。](img/Figure_9.20.jpg)

###### 图 9.20：默认情况下显示高级图表

您可以进一步深入研究各个图表。例如，单击消息图表。这将带您进入 Azure 监视器中的交互式图表编辑器。您可以按分钟查看事件中心的进出消息数量，如*图 9.21*所示：

![单击第二个图表，表示消息数量，您将看到更多详细信息。除了蓝色和橙色图标外，您还将看到靛蓝色和水绿色图标，分别表示捕获的消息和捕获积压（总和）。](img/Figure_9.21.jpg)

###### 图 9.21：单击图表以获取更多详细信息

让我们确保清理我们刚刚创建的部署，并将我们的集群缩减回去：

```
helm delete social-network -n social-network
kubectl delete pvc -n social-network --all
kubectl delete pv --all
kubectl delete service neo4j-service -n social-network
az aks nodepool scale --node-count 2 -g rg-handsonaks \
  --cluster-name handsonaks --name agentpool
```

您还可以在 Azure 门户中删除事件中心。要删除事件中心，请转到事件中心的**概述**页面，并选择**删除**按钮，如*图 9.22*所示。系统会要求您重复事件中心的名称，以确保您不会意外删除它：

![单击左侧屏幕上的导航窗格中的概述。您将看到您创建的事件中心的详细信息。要删除此事件中心，请单击工具栏中的删除按钮。此按钮位于刷新按钮的左侧。](img/Figure_9.22.jpg)

###### 图 9.22：单击删除按钮以删除您的事件中心

本章结束了我们使用 Azure Kubernetes 服务与事件中心的示例。在这个示例中，我们重新配置了一个 Kafka 应用程序，以使用 Azure 事件中心。

## 摘要

在本章中，我们部署了一个基于微服务的应用程序，连接到 Kafka。我们使用 Helm 部署了这个示例应用程序。我们能够通过向本地创建的 Kafka 集群发送事件并观察创建和更新的对象来测试应用程序。

最后，我们介绍了使用 Kafka 支持在 Azure 事件中心存储事件，并且我们能够收集所需的细节来连接我们基于微服务的应用程序并修改 Helm 图表。

下一章将涵盖集群安全性。我们将涵盖 RBAC 安全性、秘钥管理以及使用 Istio 的网络安全。

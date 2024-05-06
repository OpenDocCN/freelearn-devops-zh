## 第十六章：14：使用 Docker 堆栈部署应用程序

在规模上部署和管理多服务应用程序是困难的。

幸运的是，Docker 堆栈在这里帮忙！它们通过提供*期望状态、滚动更新、简单的扩展操作、健康检查*等等来简化应用程序管理，所有这些都包含在一个很好的声明模型中。太棒了！

现在，如果这些术语对您来说是新的或听起来很复杂，不要担心！在本章结束时，您将理解它们！

我们将把本章分为通常的三个部分：

+   简而言之

+   深入了解

+   命令

### 使用 Docker 堆栈部署应用程序-简而言之

在您的笔记本电脑上测试和部署简单的应用程序很容易。但那是给业余爱好者的。在现实世界的生产环境中部署和管理多服务应用程序……那是给专业人士的！

幸运的是，堆栈在这里帮忙！它们让您在单个声明文件中定义复杂的多服务应用程序。它们还提供了一种简单的方式来部署应用程序并管理其整个生命周期-初始部署>健康检查>扩展>更新>回滚等等！

这个过程很简单。在*Compose 文件*中定义您的应用程序，然后使用`docker stack deploy`命令部署和管理它。就是这样！

Compose 文件包括组成应用程序的整个服务堆栈。它还包括应用程序需要的所有卷、网络、秘密和其他基础设施。然后，您可以使用`docker stack deploy`命令从文件部署应用程序。简单。

为了完成所有这些，堆栈建立在 Docker Swarm 之上，这意味着您可以获得与 Swarm 一起使用的所有安全性和高级功能。

简而言之，Docker 非常适合开发和测试。Docker 堆栈非常适合规模和生产！

### 使用 Docker 堆栈部署应用程序-深入了解

如果您了解 Docker Compose，您会发现 Docker 堆栈非常容易。实际上，在许多方面，堆栈是我们一直希望 Compose 是-完全集成到 Docker 中，并能够管理应用程序的整个生命周期。

从架构上讲，堆栈位于 Docker 应用程序层次结构的顶部。它们建立在*服务*之上，而服务又建立在容器之上。见图 14.1。

![图 14.1 AtSea Shop 高级架构](img/figure14-1.png)

图 14.1 AtSea Shop 高级架构

我们将把本章的这一部分分为以下几个部分：

+   示例应用程序概述

+   更仔细地查看堆栈文件

+   部署应用程序

+   管理应用程序

#### 示例应用程序概述

在本章的其余部分，我们将使用流行的**AtSea Shop**演示应用程序。它位于[GitHub](https://github.com/dockersamples/atsea-sample-shop-app)，并在[Apache 2.0 许可证](https://github.com/dockersamples/atsea-sample-shop-app/blob/master/LICENSE)下开源。

我们使用这个应用程序，因为它在不太大到无法在书中列出和描述的情况下，具有适度的复杂性。在底层，它是一个利用证书和密钥的多技术微服务应用程序。高级应用程序架构如图 14.2 所示。

![图 14.2 AtSea Shop 高级架构](img/figure14-2.png)

图 14.2 AtSea Shop 高级架构

正如我们所看到的，它包括 5 个*服务*，3 个网络，4 个秘密和 3 个端口映射。当我们检查堆栈文件时，我们将详细了解每一个。

> **注意：**在本章中提到*服务*时，我们指的是 Docker 服务（作为单个对象管理的容器集合和存在于 Docker API 中的服务对象）。

克隆应用程序的 GitHub 存储库，以便在本地机器上拥有所有应用程序源文件。

```
$ git clone https://github.com/dockersamples/atsea-sample-shop-app.git
Cloning into `'atsea-sample-shop-app'`...
remote: Counting objects: `636`, `done`.
remote: Total `636` `(`delta `0``)`, reused `0` `(`delta `0``)`, pack-reused `636`
Receiving objects: `100`% `(``636`/636`)`, `7`.23 MiB `|` `28`.25 MiB/s, `done`.
Resolving deltas: `100`% `(``197`/197`)`, `done`. 
```

`该应用程序由多个目录和源文件组成。请随意探索它们。但是，我们将专注于`docker-stack.yml`文件。我们将把它称为*堆栈文件*，因为它定义了应用程序及其要求。

在最高级别，它定义了 4 个顶级键。

```
`version``:`
`services``:`
`networks``:`
`secrets``:` 
```

`**版本**表示 Compose 文件格式的版本。这必须是 3.0 或更高才能与堆栈一起使用。**服务**是我们定义组成应用程序的服务堆栈的地方。**网络**列出了所需的网络，**秘密**定义了应用程序使用的秘密。

如果我们展开每个顶级键，我们将看到如何将事物映射到图 14.1。堆栈文件有五个名为“reverse_proxy”、“database”、“appserver”、“visualizer”和“payment_gateway”的服务。图 14.1 也是如此。堆栈文件有三个名为“front-tier”、“back-tier”和“payment”的网络。图 14.1 也是如此。最后，堆栈文件有四个名为“postgres_password”、“staging_token”、“revprox_key”和“revprox_cert”的秘密。图 14.1 也是如此。

```
version: "3.2"
services:
    reverse_proxy:
    database:
    appserver:
    visualizer:
    payment_gateway:
networks:
    front-tier:
    back-tier:
    payment:
secrets:
    postgres_password:
    staging_token:
    revprox_key:
    revprox_cert: 
```

`重要的是要理解，堆栈文件捕获并定义了整个应用程序的许多要求。因此，它是一种应用程序自我文档化的形式，也是弥合开发和运维之间差距的重要工具。

让我们更仔细地看看堆栈文件的每个部分。

#### 仔细查看堆栈文件

堆栈文件是一个 Docker Compose 文件。唯一的要求是`version:`键指定一个值为“3.0”或更高。有关 Compose 文件版本的最新信息，请参阅[Docker 文档](https://docs.docker.com/compose/compose-file/)。

从堆栈文件部署应用程序时，Docker 要做的第一件事是检查并创建`networks:`键下列出的网络。如果网络尚不存在，Docker 将创建它们。

让我们看看堆栈文件中定义的网络。

##### 网络

```
`networks``:`
  `front``-``tier``:`
  `back``-``tier``:`
  `payment``:`
    `driver``:` `overlay`
    `driver_opts``:`
      `encrypted``:` `'yes'` 
```

`定义了三个网络; `front-tier`，`back-tier`和`payment`。默认情况下，它们都将由`overlay`驱动程序创建为覆盖网络。但是`payment`网络很特别 - 它需要加密的数据平面。

默认情况下，所有覆盖网络的控制平面都是加密的。要加密数据平面，您有两个选择：

+   将`-o encrypted`标志传递给`docker network create`命令。

+   在堆栈文件中的`driver_opts`下指定`encrypted: 'yes'`。

加密数据平面所产生的开销取决于各种因素，如流量类型和流量流向。但是，预计它将在 10%左右。

如前所述，在创建秘密和服务之前，将创建所有三个网络。

现在让我们来看看秘密。

##### 秘密

秘密被定义为顶级对象，我们使用的堆栈文件定义了四个：

```
`secrets``:`
  `postgres_password``:`
    `external``:` `true`
  `staging_token``:`
    `external``:` `true`
  `revprox_key``:`
    `external``:` `true`
  `revprox_cert``:`
    `external``:` `true` 
```

`请注意，所有四个都被定义为`external`。这意味着它们必须在堆栈部署之前已经存在。

秘密可以在部署应用程序时按需创建 - 只需用`file: <filename>`替换`external: true`。但是，为了使其工作，主机文件系统上必须已经存在包含秘密未加密值的明文文件。这显然具有安全影响。

当我们开始部署应用程序时，我们将看到如何创建这些秘密。现在，知道应用程序定义了需要预先创建的四个秘密就足够了。

让我们来看看每个服务。

##### 服务

服务是大部分操作发生的地方。

每个服务都是一个包含一堆键的 JSON 集合（字典）。我们将逐个解释每个选项的作用。

###### 反向代理服务

正如我们所看到的，`reverse_proxy`服务定义了一个镜像、端口、秘密和网络。

```
`reverse_proxy``:`
  `image``:` `dockersamples``/``atseasampleshopapp_reverse_proxy`
  `ports``:`
    `-` `"80:80"`
    `-` `"443:443"`
  `secrets``:`
    `-` `source``:` `revprox_cert`
      `target``:` `revprox_cert`
    `-` `source``:` `revprox_key`
      `target``:` `revprox_key`
  `networks``:`
    `-` `front``-``tier` 
```

`image`键是服务对象中唯一强制的键。顾名思义，它定义了将用于构建服务副本的 Docker 镜像。

Docker 有自己的见解，因此除非另有说明，**image**将从 Docker Hub 中拉取。您可以通过在镜像名称前加上注册表的 DNS 名称来指定来自第三方注册表的镜像，例如`gcr.io`用于 Google 的容器注册表。

Docker Stacks 和 Docker Compose 之间的一个区别是，堆栈不支持**构建**。这意味着在部署堆栈之前，所有镜像都必须构建。

**ports**键定义了两个映射：

+   `80:80`将 Swarm 上的端口 80 映射到每个服务副本的端口 80。

+   `443:443`将 Swarm 上的端口 443 映射到每个服务副本的端口 443。

默认情况下，所有端口都使用*入口模式*进行映射。这意味着它们将被映射并且可以从 Swarm 中的每个节点访问 - 即使节点没有运行副本。另一种选择是*主机模式*，在这种模式下，端口仅在运行服务副本的 Swarm 节点上映射。但是，*主机模式*要求您使用长格式语法。例如，使用长格式语法在*主机模式*下映射端口 80 将是这样的：

```
ports:
  - target: 80
    published: 80
    mode: host 
```

`长格式语法是推荐的，因为它更容易阅读和更强大（支持入口模式**和**主机模式）。但是，它至少需要版本 3.2 的 Compose 文件格式。

**secrets**键定义了两个秘密 - `revprox_cert`和`revprox_key`。这些必须在顶级`secrets`键中定义，并且必须存在于系统中。

秘密作为常规文件挂载到服务副本中。文件的名称将是您在堆栈文件中指定为`target`值的内容，并且该文件将出现在 Linux 上的副本中的`/run/secrets`下，在 Windows 上为`C:\ProgramData\Docker\secrets`。Linux 将`/run/secrets`挂载为内存文件系统，但 Windows 不会。

在此服务中定义的秘密将被挂载到每个服务副本中，如`/run/secrets/revprox_cert`和`/run/secrets/revprox_key`。要将其中一个挂载为`/run/secrets/uber_secret`，您可以在堆栈文件中定义如下：

```
secrets:
  - source: revprox_cert
    target: uber_secret 
```

**networks**键确保服务的所有副本都将连接到`front-tier`网络。此处指定的网络必须在`networks`顶级键中定义，如果尚不存在，Docker 将将其创建为覆盖。

###### 数据库服务

数据库服务还定义了一个镜像、一个网络和一个秘密。除此之外，它还引入了环境变量和放置约束。

```
`database``:`
  `image``:` `dockersamples``/``atsea_db`
  `environment``:`
    `POSTGRES_USER``:` `gordonuser`
    `POSTGRES_DB_PASSWORD_FILE``:` `/run/secrets/``postgres_password`
    `POSTGRES_DB``:` `atsea`
  `networks``:`
    `-` `back``-``tier`
  `secrets``:`
    `-` `postgres_password`
  `deploy``:`
    `placement``:`
      `constraints``:`
        `-` `'node.role == worker'` 
```

**环境** 键允许您将环境变量注入服务副本。此服务使用三个环境变量来定义数据库用户、数据库密码的位置（挂载到每个服务副本中的秘密）和数据库的名称。

```
`environment``:`
  `POSTGRES_USER``:` `gordonuser`
  `POSTGRES_DB_PASSWORD_FILE``:` `/run/secrets/``postgres_password`
  `POSTGRES_DB``:` `atsea` 
```

`> **注意：** 将所有三个值作为秘密传递更安全，因为这样可以避免在明文变量中记录数据库名称和数据库用户。

该服务还在 `deploy` 键下定义了 *放置约束*。这确保了该服务的副本始终在 Swarm *worker* 节点上运行。

```
`deploy``:`
  `placement``:`
    `constraints``:`
      `-` `'node.role == worker'` 
```

放置约束是一种拓扑感知调度的形式，可以是影响调度决策的好方法。Swarm 目前允许您针对以下所有进行调度：

+   节点 ID。`node.id == o2p4kw2uuw2a`

+   节点名称。`node.hostname == wrk-12`

+   角色。`node.role != manager`

+   引擎标签。`engine.labels.operatingsystem==ubuntu 16.04`

+   自定义节点标签。`node.labels.zone == prod1`

请注意，`==` 和 `!=` 都受支持。

###### appserver 服务

`appserver` 服务使用一个镜像，连接到三个网络，并挂载一个秘密。它还在 `deploy` 键下引入了几个其他功能。

```
`appserver``:`
  `image``:` `dockersamples``/``atsea_app`
  `networks``:`
    `-` `front``-``tier`
    `-` `back``-``tier`
    `-` `payment`
  `deploy``:`
    `replicas``:` `2`
    `update_config``:`
      `parallelism``:` `2`
      `failure_action``:` `rollback`
    `placement``:`
      `constraints``:`
        `-` `'node.role == worker'`
    `restart_policy``:`
      `condition``:` `on``-``failure`
      `delay``:` `5``s`
      `max_attempts``:` `3`
      `window``:` `120``s`
  `secrets``:`
    `-` `postgres_password` 
```

让我们更仔细地看看 `deploy` 键下的新内容。

首先，`services.appserver.deploy.replicas = 2` 将设置服务的期望副本数为 2。如果省略，则默认值为 1。如果服务正在运行，并且您需要更改副本的数量，您应该以声明方式进行。这意味着在堆栈文件中更新 `services.appserver.deploy.replicas` 为新值，然后重新部署堆栈。我们稍后会看到，但重新部署堆栈不会影响您没有进行更改的服务。

`services.appserver.deploy.update_config` 告诉 Docker 在对服务进行更新时如何操作。对于此服务，Docker 将一次更新两个副本（`parallelism`），如果检测到更新失败，将执行“回滚”。回滚将基于服务的先前定义启动新的副本。`failure_action` 的默认值是 `pause`，这将停止进一步更新副本。另一个选项是 `continue`。

```
`update_config``:`
  `parallelism``:` `2`
  `failure_action``:` `rollback` 
```

`services.appserver.deploy.restart-policy`对象告诉 Swarm 如何重新启动副本（容器），如果它们失败的话。此服务的策略将在副本以非零退出代码停止时重新启动（`condition: on-failure`）。它将尝试重新启动失败的副本 3 次，并等待最多 120 秒来决定重新启动是否成功。在三次重新启动尝试之间将等待 5 秒。

```
`restart_policy``:`
  `condition``:` `on``-``failure`
  `delay``:` `5``s`
  `max_attempts``:` `3`
  `window``:` `120``s` 
```

`###### visualizer

visualizer 服务引用了一个镜像，映射了一个端口，定义了一个更新配置，并定义了一个放置约束。它还挂载了一个卷，并为容器停止操作定义了一个自定义宽限期。

```
`visualizer``:`
  `image``:` `dockersamples``/``visualizer``:``stable`
  `ports``:`
    `-` `"8001:8080"`
  `stop_grace_period``:` `1``m30s`
  `volumes``:`
    `-` `"/var/run/docker.sock:/var/run/docker.sock"`
  `deploy``:`
    `update_config``:`
      `failure_action``:` `rollback`
    `placement``:`
      `constraints``:`
        `-` `'node.role == manager'` 
```

当 Docker 停止一个容器时，它向容器内部的 PID 1 进程发出`SIGTERM`。容器（其 PID 1 进程）然后有 10 秒的宽限期来执行任何清理操作。如果它没有处理信号，它将在 10 秒后被强制终止，使用`SIGKILL`。`stop_grace_period`属性覆盖了这个 10 秒的宽限期。

`volumes`键用于将预先创建的卷和主机目录挂载到服务副本中。在这种情况下，它将`/var/run/docker.sock`从 Docker 主机挂载到每个服务副本中的`/var/run/docker.sock`。这意味着对副本中`/var/run/docker.sock`的任何读写都将通过传递到主机中的相同目录。

`/var/run/docker.sock`碰巧是 Docker 守护程序在其上公开所有 API 端点的 IPC 套接字。这意味着让容器访问它允许容器消耗所有 API 端点 - 从本质上讲，这使得容器能够查询和管理 Docker 守护程序。在大多数情况下，这是一个巨大的“不行”。但是，在实验环境中，这是一个演示应用程序。

这个服务需要访问 Docker 套接字的原因是因为它提供了 Swarm 上服务的图形表示。为了做到这一点，它需要能够查询管理节点上的 Docker 守护程序。为了实现这一点，一个放置约束强制所有服务副本进入管理节点，并且 Docker 套接字被绑定挂载到每个服务副本中。*绑定挂载*如图 14.3 所示。

![图 14.3](img/figure14-3.png)

图 14.3

###### payment_gateway

`payment_gateway`服务指定了一个镜像，挂载了一个秘密，连接到一个网络，定义了一个部分部署策略，然后施加了一些放置约束。

```
`payment_gateway``:`
  `image``:` `dockersamples``/``atseasampleshopapp_payment_gateway`
  `secrets``:`
    `-` `source``:` `staging_token`
      `target``:` `payment_token`
  `networks``:`
    `-` `payment`
  `deploy``:`
    `update_config``:`
      `failure_action``:` `rollback`
    `placement``:`
      `constraints``:`
        `-` `'node.role == worker'`
        `-` `'node.labels.pcidss == yes'` 
```

`我们之前见过所有这些选项，除了在放置约束中的`node.label`。节点标签是使用`docker node update`命令添加到 Swarm 节点的自定义定义标签。因此，它们只适用于 Swarm 中节点的角色（您不能在独立容器或 Swarm 之外利用它们）。

在这个例子中，`payment_gateway`服务执行需要在符合 PCI DSS 标准的 Swarm 节点上运行的操作。为了实现这一点，您可以将自定义*节点标签*应用到满足这些要求的任何 Swarm 节点上。在构建实验室以部署应用程序时，我们将这样做。

由于此服务定义了两个放置约束，副本将只部署到符合两者的节点。即具有`pcidss=yes`节点标签的**工作**节点。

现在我们已经完成了检查堆栈文件，我们应该对应用程序的要求有一个很好的理解。如前所述，堆栈文件是应用程序文档的重要部分。我们知道应用程序有 5 个服务，3 个网络和 4 个秘密。我们知道哪些服务连接到哪些网络，哪些端口需要发布，需要哪些镜像，甚至知道一些服务需要在特定节点上运行。

让我们部署它。

#### 部署应用程序

在我们部署应用程序之前，有一些先决条件需要处理：

+   **Swarm 模式：**我们将应用程序部署为 Docker Stack，堆栈需要 Swarm 模式。

+   **标签：**Swarm 工作节点中的一个需要一个自定义节点标签。

+   **秘密：**应用程序使用需要在部署之前预先创建的秘密。

##### 为示例应用程序构建实验室

在这一部分，我们将构建一个满足应用程序所有先决条件的三节点基于 Linux 的 Swarm 集群。一旦完成，实验室将如下所示。

![图 14.4 示例实验室](img/figure14-4.png)

图 14.4 示例实验室

完成以下三个步骤：

+   创建一个新的 Swarm

+   添加一个节点标签

+   创建秘密

让我们创建一个新的三节点 Swarm 集群。

1.  初始化一个新的 Swarm。

在您想要成为 Swarm 管理节点的节点上运行以下命令。

```
 $ docker swarm init
 Swarm initialized: current node (lhma...w4nn) is now a manager.
 <Snip> 
```

`* 添加工作节点。

复制在上一个命令的输出中显示的`docker swarm join`命令。将其粘贴到要加入为工作节点的两个节点中。

```
 //Worker 1 (wrk-1)
 wrk-1$ docker swarm join --token SWMTKN-1-2hl6...-...3lqg 172.31.40.192:2377
 This node joined a swarm as a worker.

 //Worker 2 (wrk-2)
 wrk-2$ docker swarm join --token SWMTKN-1-2hl6...-...3lqg 172.31.40.192:2377
 This node joined a swarm as a worker. 
```

`* 验证 Swarm 是否配置为一个管理节点和两个工作节点。

从管理节点运行此命令。

```
 $ docker node ls
 ID            HOSTNAME   STATUS     AVAILABILITY    MANAGER STATUS
 lhm...4nn *   mgr-1      Ready      Active          Leader
 b74...gz3     wrk-1      Ready      Active
 o9x...um8     wrk-2      Ready      Active 
`````

```The Swarm is now ready.

The `payment_gateway` service has set of placement constraints forcing it to only run on **worker nodes** with the `pcidss=yes` node label. In this step we’ll add that node label to `wrk-1`.

In the real world you would harden at least one of your Docker nodes to PCI standards before labelling it. However, this is just a lab, so we’ll skip the hardening step and just add the label to `wrk-1`.

Run the following commands from the Swarm manager.

1.  Add the node label to `wrk-1`.

```

$ docker node update --label-add pcidss=yes wrk-1

```

 `Node labels only apply within the Swarm.` 
`*   Verify the node label.

```

$ docker node inspect wrk-1

[

{

“ID”：“b74rzajmrimfv7hood6l4lgz3”，

“版本”：{

“索引”：27

},

“创建时间”：“2018-01-25T10:35:18.146831621Z”，

“更新时间”：“2018-01-25T10:47:57.189021202Z”，

“规格”：{

“标签”：{

“pcidss”：“是”

},

<Snip>

```` 

 ``The `wrk-1` worker node is now configured so that it can run replicas for the `payment_gateway` service.

The application defines four secrets, all of which need creating before the app can be deployed:

*   `postgress_password`
*   `staging_token`
*   `revprox_cert`
*   `revprox_key`

Run the following commands from the manager node to create them.

1.  Create a new key pair.

Three of the secrets will be populated with cryptographic keys. We’ll create the keys in this step and then place them inside of Docker secrets in the next steps.

```
 $ openssl req -newkey rsa:4096 -nodes -sha256 \
   -keyout domain.key -x509 -days 365 -out domain.crt 
```

 `You’ll have two new files in your current directory. We’ll use them in the next step.` 
`*   Create the `revprox_cert`, `revprox_key`, and `postgress_password` secrets.

```
 $ docker secret create revprox_cert domain.crt
 cqblzfpyv5cxb5wbvtrbpvrrj

 $ docker secret create revprox_key domain.key
 jqd1ramk2x7g0s2e9ynhdyl4p

 $ docker secret create postgres_password domain.key
 njpdklhjcg8noy64aileyod6l 
```

`*   Create the `staging_token` secret.

```
 $ echo staging | docker secret create staging_token -
 sqy21qep9w17h04k3600o6qsj 
```

`*   List the secrets.

```
 $ docker secret ls
 ID          NAME                CREATED              UPDATED
 njp...d6l   postgres_password   47 seconds ago       47 seconds ago
 cqb...rrj   revprox_cert        About a minute ago   About a minute ago
 jqd...l4p   revprox_key         About a minute ago   About a minute ago
 sqy...qsj   staging_token       23 seconds ago       23 seconds ago 
``````

```That’s all of the pre-requisites taken care of. Time to deploy the app!

##### Deploying the sample app

If you haven’t already done so, clone the app’s GitHub repo to your Swarm manager.

```

$ git clone https://github.com/dockersamples/atsea-sample-shop-app.git

克隆到`'atsea-sample-shop-app'`...

远程：计算对象：`636`，`完成`。

接收对象：`100`% `(``636`/636`)`, `7`.23 MiB `|` `3`.30 MiB/s，`完成`。

远程：总共`636` `（增量`0``）`，重用`0` `（增量`0``）`，包重用`636`

解决增量：`100`% `(``197`/197`)`, `完成`。

检查连通性... `完成`。

$ `cd` atsea-sample-shop-app

```

 `Now that you have the code, you are ready to deploy the app.

Stacks are deployed using the `docker stack deploy` command. In its basic form, it accepts two arguments:

*   name of the stack file
*   name of the stack

The application’s GitHub repository contains a stack file called `docker-stack.yml`, so we’ll use this as stack file. We’ll call the stack `seastack`, though you can choose a different name if you don’t like that.

Run the following commands from within the `atsea-sample-shop-app` directory on the Swarm manager.

Deploy the stack (app).

```

使用 docker 堆栈部署-docker-stack.yml seastack

创建网络 seastack_default

创建网络 seastack_back-tier

创建网络 seastack_front-tier

创建网络 seastack_payment

创建服务 seastack_database

创建服务 seastack_appserver

创建服务 seastack_visualizer

创建服务 seastack_payment_gateway

创建服务 seastack_reverse_proxy

```

 `You can run `docker network ls` and `docker service ls` commands to see the networks and services that were deployed as part of the app.

A few things to note from the output of the command.

The networks were created before the services. This is because the services attach to the networks, so need the networks to be created before they can start.

Docker prepends the name of the stack to every resource it creates. In our example, the stack is called `seastack`, so all resources are named `seastack_<resource>`. For example, the `payment` network is called `seastack_payment`. Resources that were created prior to the deployment, such as secrets, do not get renamed.

Another thing to note is the presence of a network called `seastack_default`. This isn’t defined in the stack file, so why was it created? Every service needs to attach to a network, but the `visualizer` service didn’t specify one. Therefore, Docker created one called `seastack_default` and attached it to that.

You can verify the status of a stack with a couple of commands. `docker stack ls` lists all stacks on the system, including how many services they have. `docker stack ps <stack-name>` gives more detailed information about a particular stack, such as *desired state* and *current state*. Let’s see them both.

```

$ docker stack ls

名称 服务

seastack `5`

$ docker stack ps seastack

名称 节点 期望状态 当前状态

seastack_reverse_proxy.1 wrk-2 运行 运行`7`分钟前

seastack_payment_gateway.1 wrk-1 运行 运行`7`分钟前

seastack_visualizer.1 mgr-1 运行 运行`7`分钟前

seastack_appserver.1 wrk-2 运行 运行`7`分钟前

seastack_database.1 wrk-2 运行 运行`7`分钟前

seastack_appserver.2 wrk-1 运行 运行`7`分钟前

```

 `The `docker stack ps` command is a good place to start when troubleshooting services that fail to start. It gives an overview of every service in the stack, including which node each replica is scheduled on, current state, desired state, and error message. The following output shows two failed attempts to start a replica for the `reverse_proxy` service on the `wrk-2` node.

```

$ docker stack ps seastack

名称 节点 期望的 当前 错误

状态 状态

reverse_proxy.1 wrk-2 关机 失败 `"任务：非零退出（1）"`

`\_`reverse_proxy.1 wrk-2 关机 失败 `"任务：非零退出（1）"`

```

 `For more detailed logs of a particular service you can use the `docker service logs` command. You pass it either the service name/ID, or replica ID. If you pass it the service name or ID, you’ll get the logs for all service replicas. If you pass it a particular replica ID, you’ll only get the logs for that replica.

The following `docker service logs` command shows the logs for all replicas in the `seastack_reverse_proxy` service that had the two failed replicas in the previous output.

```

$ docker service logs seastack_reverse_proxy

seastack_reverse_proxy.1.zhc3cjeti9d4@wrk-2 `|` `[`emerg`]` `1``#1: 主机未找到...`

seastack_reverse_proxy.1.6m1nmbzmwh2d@wrk-2 `|` `[`emerg`]` `1``#1: 主机未找到...`

seastack_reverse_proxy.1.6m1nmbzmwh2d@wrk-2 `|` nginx：`[`emerg`]`主机未找到..

seastack_reverse_proxy.1.zhc3cjeti9d4@wrk-2 `|` nginx：`[`emerg`]`主机未找到..

seastack_reverse_proxy.1.1tmya243m5um@mgr-1 `|` `10`.255.0.2 `"GET / HTTP/1.1"` `302`

```

 `The output is trimmed to fit the page, but you can see that logs from all three service replicas are shown (the two that failed and the one that’s running). Each line starts with the name of the replica, which includes the service name, replica number, replica ID, and name of host that it’s scheduled on. Following that is the log output.

> **Note:** You might have noticed that all of the replicas in the previous output showed as replica number 1\. This is because Docker created one at a time and only started a new one when the previous one had failed.

It’s hard to tell because the output is trimmed to fit the book, but it looks like the first two replicas failed because they were relying on something in another service that was still starting (a sort of race condition when dependent services are starting).

You can follow the logs (`--follow`), tail them (`--tail`), and get extra details (`--details`).

Now that the stack is up and running, let’s see how to manage it.

#### Managing the app

We know that a *stack* is set of related services and infrastructure that gets deployed and managed as a unit. And while that’s a fancy sentence full of buzzwords, it reminds us that the stack is built from normal Docker resources — networks, volumes, secrets, services etc. This means we can inspect and reconfigure these with their normal docker commands: `docker network`, `docker volume`, `docker secret`, `docker service`…

With this in mind, it’s possible to use the `docker service` command to manage services that are part of the stack. A simple example would be using the `docker service scale` command to increase the number of replicas in the `appserver` service. However, **this is not the recommended method!**

The recommended method is the declarative method, which uses the stack file as the ultimate source of truth. As such, all changes to the stack should be made to the stack file, and the updated stack file used to redeploy the app.

Here’s a quick example of why the imperative method (making changes via the CLI) is bad:

> *Imagine that we have a stack deployed from the `docker-stack.yml` file that we cloned from GitHub earlier in the chapter. This means we have two replicas of the `appserver` service. If we use the `docker service scale` command to change that to 4 replicas, the current state of the cluster will be 4 replicas, but the stack file will still define 2\. Admittedly, that doesn’t sound like the end of the world. However, imagine we then make a different change to the stack, this time via the stack file, and we roll it out with the `docker stack deploy` command. As part of this rollout, the number of `appserver` replicas in the cluster will be rolled back to 2, because this is what the stack file defines. For this kind of reason, it is recommended to make all changes to the application via the stack file, and to manage the file in a proper version control system.*

Let’s walk through the process of making a couple of declarative changes to the stack.

We’ll make the following changes:

*   Increase the number of `appserver` replicas from 2 to 10
*   Increase the stop grace period for the visualizer service to 2 minutes

Edit the `docker-stack.yml` file and update the following two values:

*   `.services.appserver.deploy.replicas=10`
*   `.services.visualizer.stop_grace_period=2m`

The relevant sections of the stack file will now look like this:

```

<Snip>

appserver：

图像：dockersamples/atsea_app

网络：

- front-tier

- back-tier

- payment

部署：

副本: 2             <<更新值

<Snip>

visualizer:

镜像: dockersamples/visualizer:stable

端口：

- "8001:8080"

stop_grace_period: 2m     <<更新值

<Snip

```

 `Save the file and redeploy the app.

```

$ docker stack deploy -c docker-stack.yml seastack

更新服务 seastack_reverse_proxy `(`id: z4crmmrz7zi83o0721heohsku`)`

更新服务 seastack_database `(`id: 3vvpkgunetxaatbvyqxfic115`)`

更新服务 seastack_appserver `(`id: ljht639w33dhv0dmht1q6mueh`)`

更新服务 seastack_visualizer `(`id: rbwoyuciglre01hsm5fviabjf`)`

更新服务 seastack_payment_gateway `(`id: w4gsdxfnb5gofwtvmdiooqvxs`)`

```

 `Re-deploying the app like this will only update the changed components.

Run a `docker stack ps` to see the number of `appserver` replicas increasing.

```

$ docker stack ps seastack

名称                    节点     期望状态   当前状态

seastack_visualizer.1   mgr-1    运行中         运行中 `1` 秒前

seastack_visualizer.1   mgr-1    关闭        关闭 `3` 秒前

seastack_appserver.1    wrk-2    运行中         运行中 `24` 分钟前

seastack_appserver.2    wrk-1    运行中         运行中 `24` 分钟前

seastack_appserver.3    wrk-2    运行中         运行中 `1` 秒前

seastack_appserver.4    wrk-1    运行中         运行中 `1` 秒前

seastack_appserver.5    wrk-2    运行中         运行中 `1` 秒前

seastack_appserver.6    wrk-1    运行中         启动 `7` 秒前

seastack_appserver.7    wrk-2    运行中         运行中 `1` 秒前

seastack_appserver.8    wrk-1    运行中         启动 `7` 秒前

seastack_appserver.9    wrk-2    运行中         运行中 `1` 秒前

seastack_appserver.10   wrk-1    运行中         启动 `7` 秒前

```

 `The output has been trimmed so that it fits on the page, and so that only the affected services are shown.

Notice that there are two lines for the `visualizer` service. One line shows a replica that was shutdown 3 seconds ago, and the other line shows a replica that has been running for 1 second. This is because we pushed a change to the `visualizer` service, so Swarm terminated the existing replica and started a new one with the new `stop_grace_period` value.

Also note that we now have 10 replicas for the `appserver` service, and that they are in various states in the “CURRENT STATE” column — some are *running* whereas others are still *starting*.

After enough time, the cluster will converge so that *desired state* and *current state* match. At that point, what is deployed and observed on the cluster will exactly match what is defined in the stack file. This is a happy place to be :-D

This update pattern should be used for all updates to the app/stack. I.e. **all changes should be made declaratively via the stack file, and rolled out using `docker stack deploy`**.

The correct way to delete a stack is with the `docker stack rm` command. Be warned though! It deletes the stack without asking for confirmation.

```

$ docker stack rm seastack

删除服务 seastack_appserver

删除服务 seastack_database

删除服务 seastack_payment_gateway

删除服务 seastack_reverse_proxy

删除服务 seastack_visualizer

删除网络 seastack_front-tier

删除网络 seastack_payment

删除网络 seastack_default

删除网络 seastack_back-tier

```

 `Notice that the networks and services were deleted, but the secrets were not. This is because the secrets were pre-created and existed before the stack was deployed. If your stack defines volumes at the top-level, these will not be deleted by `docker stack rm` either. This is because volumes are intended as long-term persistent data stores and exist independent of the lifecycle of containers, services, and stacks.

Congratulations! You know how to deploy and manage a multi-service app using Docker Stacks.

### Deploying apps with Docker Stacks - The Commands

*   `docker stack deploy` is the command we use to deploy **and** update stacks of services defined in a stack file (usually `docker-stack.yml`).
*   `docker stack ls` will list all stacks on the Swarm, including how many services they have.
*   `docker stack ps` gives detailed information about a deployed stack. It accepts the name of the stack as its main argument, lists which node each replica is running on, and shows *desired state* and *current state*.
*   `docker stack rm` is the command to delete a stack from the Swarm. It does not ask for confirmation before deleting the stack.

### Chapter Summary

Stacks are the native Docker solution for deploying and managing multi-service applications. They’re baked into the Docker engine, and offer a simple declarative interface for deploying and managing the entire lifecycle of an application.

We start with application code and a set of infrastructure requirements — things like networks, ports, volumes and secrets. We containerize the application and group together all of the app services and infrastructure requirements into a single declarative stack file. We set the number of replicas, as well as rolling update and restart policies. Then we take the file and deploy the application from it using the `docker stack deploy` command.

Future updates to the deployed app should be done declaratively by checking the stack file out of source control, updating it, re-deploying the app, and checking the stack file back in to source control.

Because the stack file defines things like number of service replicas, you should maintain separate stack files for each of your environments, such as dev, test and prod.`````````````````````````````````

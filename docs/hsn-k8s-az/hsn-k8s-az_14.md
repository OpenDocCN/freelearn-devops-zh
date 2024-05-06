# 11\. 无服务器函数

在过去几年中，无服务器和无服务器函数已经获得了巨大的发展。Azure Functions、AWS Lambda 和 GCP Cloud Run 等云服务使开发人员非常容易地将其代码作为无服务器函数运行。

“无服务器”一词指的是您无需管理服务器的任何解决方案。无服务器函数是指无服务器计算的子集，您可以按需将代码作为函数运行。这意味着函数中的代码只有在有“需求”时才会运行和执行。这种架构风格称为事件驱动架构。在事件驱动架构中，事件消费者在有事件发生时被触发。在无服务器函数的情况下，事件消费者将是这些无服务器函数。事件可以是从队列上的消息到上传到存储的新对象，甚至是 HTTP 调用的任何内容。

无服务器函数经常用于后端处理。无服务器函数的一个常见示例是创建上传到存储的图片的缩略图。由于无法预测将上传多少图片以及它们何时上传，很难规划传统基础设施以及为此过程应该准备多少服务器。如果将缩略图的创建实现为无服务器函数，该函数将在上传的每张图片上被调用。您无需规划函数的数量，因为每张新图片都将触发执行一个新函数。

这种自动扩展只是使用无服务器函数的一个好处。正如您在前面的示例中看到的，函数将自动扩展以满足增加或减少的需求。此外，每个函数可以独立于其他函数进行扩展。无服务器函数的另一个好处是对开发人员的易用性。无服务器函数允许代码部署而无需担心管理服务器和中间件。最后，在公共云无服务器函数中，您按照函数的执行付费。这意味着每次运行函数时都要付费，并且在函数不运行时不收取任何费用。

公共云无服务器函数平台的流行导致了多个开源框架的创建，使用户能够在 Kubernetes 之上创建无服务器函数。在本章中，您将学习如何使用 Azure Functions 的开源版本直接在**Azure Kubernetes Services** (**AKS**)上部署无服务器函数。您将首先运行一个简单的函数，该函数是基于 HTTP 消息触发的。之后，您将在集群上安装函数的自动缩放功能。您还将把 AKS 部署的应用程序与 Azure 存储队列集成。我们将涵盖以下主题：

+   不同函数平台的概述

+   部署基于 HTTP 触发的函数

+   部署队列触发的函数

让我们从探索适用于 Kubernetes 的多个函数平台开始这一章。

## 多个函数平台

诸如 Azure Functions、AWS Lambda 和 Google Cloud Functions 之类的函数平台在流行度上获得了巨大的增长。无需考虑服务器即可运行代码并且具有几乎无限的扩展性非常受欢迎。使用云提供商的函数实现的缺点是您被锁定在他们的基础设施和编程模型中。此外，您只能在公共云中运行函数，而不能在自己的数据中心中运行。

已经推出了许多开源函数框架来解决这些缺点。有许多流行的框架：

+   **Serverless** ([`serverless.com/`](https://serverless.com/)): 基于 Node.js 的无服务器应用程序框架，可以在多个云提供商上部署和管理函数，包括 Azure。通过 Kubeless 提供 Kubernetes 支持。

+   **OpenFaaS** ([`www.openfaas.com/`](https://www.openfaas.com/)): OpenFaaS 是一个 Kubernetes 原生的无服务器框架。它可以在托管的 Kubernetes 环境（如 AKS）或自托管集群上运行。OpenFaaS 也作为 OpenFaaSCloud 的托管云服务提供。该平台是用 Go 语言编写的。

+   **Fission.io** ([`fission.io/`](https://fission.io/)): Fission 是用 Go 语言编写的，是 Kubernetes 原生的。这是一个由 Platform9 公司支持的无服务器框架。

+   Apache OpenWhisk（[`openwhisk.apache.org/`](https://openwhisk.apache.org/)）：OpenWhisk 是一个由 Apache 组织维护的开源分布式无服务器平台。它可以在 Kubernetes、Mesos 和 Docker Compose 上运行。它主要用 Scala 语言编写。

+   **Knative**（[`cloud.google.com/knative/`](https://cloud.google.com/knative/)）：Knative 是由 Google 开发的用 Go 语言编写的无服务器函数平台。您可以在 Google Cloud 上完全托管 Knative 函数，也可以在自己的 Kubernetes 集群上运行。

微软采取了一种有趣的策略来处理其函数平台。微软在 Azure 上作为托管服务运行 Azure Functions，并已开源了完整解决方案，并可在任何系统上运行（[`github.com/Azure/azure-functions-host`](https://github.com/Azure/azure-functions-host)）。这也使得 Azure Functions 编程模型可以在 Kubernetes 上运行。

微软还与红帽合作发布了一个名为**Kubernetes 事件驱动自动缩放**（**KEDA**）的额外开源项目，以使在 Kubernetes 上扩展函数更加容易。**KEDA**是一个自定义自动缩放器，可以允许部署从 0 个 Pod 缩放到 1 个 Pod。从 0 扩展到 1 个 Pod 很重要，这样您的应用程序就可以开始处理事件。缩减到 0 个实例对于保留集群中的资源很有用。使用 Kubernetes 中的默认**水平 Pod 自动缩放器（HPA）**无法实现从 0 到 1 个 Pod 的扩展。

KEDA 还可以提供额外的指标给 Kubernetes HPA，以便根据集群外部的指标（例如队列中的消息数量）做出扩展决策。

#### 注意

我们在*第四章*中介绍和解释了 HPA，*扩展您的应用程序*。

在本章中，我们将在两个示例中将 Azure Functions 部署到 Kubernetes：

+   一个 HTTP 触发的函数

+   一个队列触发的函数

在开始之前，我们需要设置一个**Azure 容器注册表**（**ACR**）和一个开发机器。ACR 将用于存储包含我们将开发的函数的自定义 Docker 映像。我们将使用开发机器构建函数并创建 Docker 映像。

## 设置先决条件

在本节中，我们将设置我们构建和运行函数所需的先决条件。我们需要一个容器注册表和一个开发机器。

在*第一章*，*Docker 和 Kubernetes 简介*中的*Docker 镜像*部分，我们介绍了容器镜像和容器注册表。容器镜像包含启动实际运行容器所需的所有软件。在本章中，我们将构建包含我们函数的自定义 Docker 镜像。我们需要一个地方来存储这些镜像，以便 Kubernetes 可以拉取这些镜像并以规模运行容器。我们将使用 Azure 容器注册表来实现这一点。Azure 容器注册表是由 Azure 完全管理的私有容器注册表。

到目前为止，在本书中，我们已经在 Azure Cloud Shell 上运行了所有的示例。对于本章的示例，我们需要一个单独的开发机器，因为 Azure Cloud Shell 不允许您构建 Docker 镜像。我们将在 Azure 上创建一个新的开发机器来执行这些任务。

让我们开始创建一个 ACR。

### Azure 容器注册表

Kubernetes 上的 Azure 函数需要一个镜像注册表来存储其容器镜像。在本节中，我们将创建一个 ACR 并配置我们的 Kubernetes 集群以访问此注册表：

1.  在 Azure 搜索栏中，搜索`容器注册表`，然后单击**容器注册表**：![在 Azure 搜索栏中输入关键词“容器注册表”以查找并选择容器注册表。](img/Figure_11.1.jpg)

###### 图 11.1：在搜索栏中查找容器注册表

1.  单击顶部的**添加**按钮以创建一个新的注册表。提供创建注册表的详细信息。注册表名称需要全局唯一，因此考虑在注册表名称中添加您的缩写。建议在与您的集群相同的位置创建注册表。选择**创建**按钮以创建注册表：![在创建容器注册表窗口中输入注册表名称、订阅、资源组、位置和 SKU 等详细信息。](img/Figure_11.2.jpg)

###### 图 11.2：提供创建注册表的详细信息

1.  当您的注册表创建好后，打开 Cloud Shell，这样我们就可以配置我们的 AKS 集群以访问我们的容器注册表。使用以下命令为您的注册表授予 AKS 权限：

```
az aks update -n handsonaks -g rg-handsonaks --attach-acr <acrName>
```

我们现在有了一个与 AKS 集成的 ACR。在下一节中，我们将创建一个开发机器，用于构建 Azure 函数。

### 创建开发机器

在本节中，我们将创建一个开发机器并安装在该机器上运行 Azure 函数所需的工具：

+   Docker 运行时

+   Azure CLI

+   Azure Functions

+   Kubectl

#### 注意

为了确保一致的体验，我们将在 Azure 上创建一个将用于开发的虚拟机（VM）。如果您希望在本地机器上运行示例，可以在本地安装所有所需的工具。

让我们开始创建这台机器：

1.  首先，我们将生成一组用于连接到 VM 的 SSH 密钥：

```
ssh-keygen
```

系统会提示您输入位置和密码。保持默认位置并输入空密码。

#### 注意

如果您按照*第十章*中的示例*保护您的 AKS 集群*，在那里我们创建了一个 Azure AD 集成的集群，您可以跳过第 1 步，因为您已经有了一组 SSH 密钥。

如果您希望重用已有的 SSH 密钥，也可以这样做。

1.  现在我们将创建我们的开发机器。我们将使用以下命令创建一个 Ubuntu VM：

```
az vm create -g rg-handsonaks -n devMachine \
  --image UbuntuLTS --ssh-key-value ~/.ssh/id_rsa.pub \
  --admin-username handsonaks --size Standard_D1_v2
```

1.  这将需要几分钟的时间才能完成。创建 VM 后，Cloud Shell 应该会显示其公共 IP，如*图 11.3*中所示：![显示位置、MAC 地址、电源状态、私有 IP 和 Ubuntu VM 的公共 IP 等详细信息的输出。输出显示 VM 的公共 IP 地址。](img/Figure_11.3.jpg)

###### 图 11.3：连接到机器的公共 IP

使用以下命令连接到 VM：

```
ssh handsonaks@<public IP>
```

系统会询问您是否信任该机器的身份。输入`yes`以确认。

1.  您现在已连接到 Azure 上的一台新机器。在这台机器上，我们将开始安装 Docker：

```
sudo apt-get update
sudo apt-get install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
```

1.  要验证 Docker 是否已安装并运行，可以运行以下命令：

```
sudo docker run hello-world
```

这应该向您显示来自 Docker 的`hello-world`消息：

![sudodocker run hello-world 命令的输出显示消息“Hello! from Docker”。](img/Figure_11.4.jpg)

###### 图 11.4：使用 Docker 运行 hello-world

1.  为了使操作更加顺畅，我们将把我们的用户添加到 Docker 组，这样在 Docker 命令前将不再需要`sudo`：

```
sudo usermod -aG docker handsonaks
newgrp docker
```

现在您应该能够在不使用`sudo`的情况下运行`hello-world`命令：

```
docker run hello-world
```

1.  接下来，我们将在开发机器上安装 Azure CLI。您可以使用以下命令安装 CLI：

```
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

1.  通过登录验证安装了 CLI：

```
az login
```

这将显示一个登录代码，您需要在[`microsoft.com/devicelogin`](https://microsoft.com/devicelogin)输入：

![az CLI 命令的输出指示用户使用 Web 浏览器输入代码。](img/Figure_11.5.jpg)

###### 图 11.5：登录到 az CLI

浏览到该网站，并粘贴提供给您的登录代码，以使您能够登录到 Cloud Shell。请确保在您已登录的浏览器中进行此操作，该浏览器具有访问您的 Azure 订阅的用户权限。

现在，我们可以使用 CLI 将我们的机器认证到 ACR。可以使用以下命令完成：

```
az acr login -n <registryname>
```

这将显示一个警告，指出密码将以未加密的形式存储。您可以忽略这一点，以便进行演示。

ACR 的凭据在一定时间后会过期。如果在此演示过程中遇到以下错误，您可以使用前面的命令重新登录 ACR：

![错误消息指出推送 Docker 镜像失败，并且需要进行身份验证。](img/Figure_11.6.jpg)

###### 图 11.6：如果遇到此错误，您可以通过重新登录 ACR 来解决此问题

1.  接下来，我们将在我们的机器上安装`kubectl`。`az` CLI 有一个安装 CLI 的快捷方式，我们将使用它：

```
sudo az aks install-cli
```

让我们验证`kubectl`是否可以连接到我们的集群。为此，我们首先获取凭据，然后执行一个`kubectl`命令：

```
az aks get-credentials -n handsonaks -g rg-handsonaks
kubectl get nodes
```

1.  现在，我们可以在这台机器上安装 Azure Functions 工具。要做到这一点，请运行以下命令：

```
wget -q https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt-get update
sudo apt-get install azure-functions-core-tools -y
```

#### 注意

如果您使用的 Ubuntu 版本比 18.04 更新，请确保通过更改第一步中的 URL 来下载正确的`dpkg`软件包以反映您的 Ubuntu 版本。

现在，我们已经具备了在 Kubernetes 上使用函数的先决条件。我们创建了一个 ACR 来存储我们的自定义 Docker 镜像，并且有一台开发机器，我们将用它来创建和构建 Azure 函数。在下一节中，我们将构建一个首个 HTTP 触发的函数。

## 创建一个 HTTP 触发的 Azure 函数

在这个第一个示例中，我们将创建一个 HTTP 触发的 Azure 函数。这意味着您可以浏览到托管实际函数的页面：

1.  首先，我们将创建一个新目录并导航到该目录：

```
mkdir http
cd http
```

1.  现在，我们将使用以下命令初始化一个函数。`--docker`参数指定我们将构建我们的函数作为 Docker 容器。这将导致为我们创建一个 Dockerfile。我们将在以下截图中选择 Python 语言，选项`3`：

```
func init --docker
```

这将创建我们的函数所需的文件：

![运行 funcinit --docker 命令后，提供了四个选项：dotnet、node、python 和 powershell。我们选择选项 3。](img/Figure_11.7.jpg)

###### 图 11.7：创建一个 Python 函数

1.  接下来，我们将创建实际的函数。输入以下代码，并选择第五个选项，`HTTP 触发器`，并将函数命名为`python-http`：

```
func new
```

这应该会产生一个类似*图 11.8*的输出：

![func new 命令的输出返回九个选项。我们选择选项 5：HTTP 触发器。](img/Figure_11.8.jpg)

###### 图 11.8：创建一个 HTTP 触发函数

1.  函数的代码存储在名为`python-http`的目录中。我们不打算对函数进行代码更改。如果您想查看函数的源代码，可以运行以下命令：

```
cat python-http/__init__.py
```

1.  我们需要对函数的配置文件进行一次更改。默认情况下，函数需要经过身份验证的请求。我们将在演示中将其更改为匿名。我们将通过执行以下命令在`vi`中进行更改：

```
vi python-http/function.json
```

我们将在第 5 行将`authLevel`替换为`anonymous`。要进行此更改，请执行以下步骤：

按*I*进入插入模式。

删除`function`并替换为`anonymous`：

![在 python-http/function.json 文件中，我们将 authlevel 更改为 anonymous。](img/Figure_11.9.jpg)

###### 图 11.9：将函数更改为匿名

+   按*Esc*，输入`:wq!`，然后按*Enter*保存并退出`vi`。

#### 注意

我们将更改函数的身份验证要求为`anonymous`。这将使我们的演示更容易执行。如果您计划将函数发布到生产环境，您需要仔细考虑此设置，因为这控制着谁可以访问您的函数。

1.  现在我们准备将函数部署到 AKS。我们可以使用以下命令部署函数：

```
func kubernetes deploy --name python-http \
--registry <registry name>.azurecr.io
```

这将导致函数的运行时执行一些步骤。首先，它将构建一个容器映像，然后将该映像推送到我们的注册表，最后将函数部署到 Kubernetes：

![funckubernetes deploy 命令的输出显示 docker build，docker push 以及创建一个 secret，一个 service 和一个 deployment。](img/Figure_11.10.jpg)

###### 图 11.10：将函数部署到 AKS

1.  这将在 Kubernetes 之上创建一个常规 Pod。要检查 Pods，您可以运行以下命令：

```
kubectl get pods
```

1.  一旦该 Pod 处于运行状态，您可以获取部署的 Service 的公共 IP 并连接到它：

```
kubectl get service
```

打开一个网络浏览器，浏览到`http://<external-ip>/api/python-http?name=handsonaks`。您应该会看到一个网页，上面显示着*Hello handsonaks!*，这就是我们的函数应该显示的内容。

![浏览器打印消息“Hello handsonaks！”](img/Figure_11.11.jpg)

###### 图 11.11：我们的函数正在正常工作

我们现在已经创建了一个带有 HTTP 触发器的函数。在转到下一部分之前，让我们清理一下这个部署：

```
Kubectl delete deploy python-http-http
kubectl delete service python-http-http
kubectl delete secret python-http
```

在这一部分，我们使用 HTTP 触发器创建了一个示例函数。让我们进一步将一个新函数与存储队列集成并设置自动缩放。

## 创建队列触发的函数

在上一部分，我们创建了一个示例 HTTP 函数。在实际应用中，队列通常用于在应用程序的不同组件之间传递消息。可以根据队列中的消息触发函数，然后对这些消息进行额外处理。

在这一部分，我们将创建一个与存储队列集成以消耗事件的函数。我们还将配置 KEDA 以允许在低流量情况下从 0 个 Pod 进行扩展/缩减。

我们仍然首先在 Azure 中创建一个队列。

### 创建队列

在这一部分，我们将创建一个新的存储帐户和一个新的队列。我们将在下一部分将函数连接到该队列。

1.  首先，我们将创建一个存储帐户。在 Azure 搜索栏中搜索“存储”并选择**存储帐户**：![通过在 Azure 搜索栏中输入“存储”来搜索存储帐户。](img/Figure_11.12.jpg)

###### 图 11.12：在 Azure 搜索栏中寻找存储

1.  点击顶部的**添加**按钮创建新帐户。提供详细信息以创建存储帐户。存储帐户名称必须是全局唯一的，因此考虑添加您的缩写。建议在与您的 AKS 集群相同的区域创建存储帐户。最后，为了节约成本，建议将复制设置降级为**本地冗余存储（LRS）**：![在基本选项卡中输入存储帐户详细信息，例如订阅、资源组、存储帐户名称、位置、性能、帐户类型、复制和访问层。](img/Figure_11.13.jpg)

###### 图 11.13：提供创建存储帐户的详细信息

如果您准备好了，请点击底部的**审阅和创建**按钮。在审阅屏幕上，选择**创建**开始创建过程。

1.  创建存储账户大约需要一分钟。创建完成后，点击**转到资源**按钮打开账户。在存储账户刀片中，转到**访问密钥**，并复制主连接字符串。暂时记下这个字符串：![在左侧面板中导航到访问密钥选项卡并复制主连接字符串。](img/Figure_11.14.jpg)

###### 图 11.14：复制主连接字符串

#### 注意

对于生产用例，不建议使用访问密钥连接到 Azure 存储。拥有该访问密钥的任何用户都可以完全访问存储账户，并且可以读取和删除其中的所有文件。建议要么生成一个**共享访问签名**（**SAS**）令牌来连接到存储，要么使用 Azure AD 集成安全性。要了解有关使用 SAS 令牌对存储进行身份验证的更多信息，请参阅[`docs.microsoft.com/rest/api/storageservices/delegate-access-with-shared-access-signature`](https://docs.microsoft.com/rest/api/storageservices/delegate-access-with-shared-access-signature)。要了解有关 Azure AD 对 Azure 存储进行身份验证的更多信息，请参阅[`docs.microsoft.com/rest/api/storageservices/authorize-with-azure-active-directory`](https://docs.microsoft.com/rest/api/storageservices/authorize-with-azure-active-directory)。

1.  最后一步是在存储账户中创建我们的队列。在左侧导航中查找`queue`，点击**+Queue**按钮添加一个队列，并为其提供一个名称。要跟随这个演示，请将队列命名为`function`：![在存储账户中创建一个新队列。](img/Figure_11.15.jpg)

###### 图 11.15：创建一个新队列

我们现在在 Azure 中创建了一个存储账户并获得了它的连接字符串。我们在这个存储账户中创建了一个队列。在下一节中，我们将创建一个将从队列中消耗消息的函数。

### 创建一个队列触发的函数

在上一节中，我们在 Azure 上创建了一个队列。在本节中，我们将创建一个新的函数，用于监视这个队列。我们需要使用这个队列的连接字符串来配置这个函数：

1.  我们将从创建一个新目录并导航到它开始：

```
mkdir ~/js-queue
cd ~/js-queue
```

1.  现在我们可以创建函数。我们将从初始化开始：

```
func init --docker
<select node, option 2>
<select javascript, option 1>
```

这应该导致*图 11.16*中显示的输出：

![funcinit --docker 命令的输出要求设置两个配置设置。我们用 2（node）回答第一个，用 1（JavaScript）回答第二个。](img/Figure_11.16.jpg)

###### 图 11.16：初始化新函数

初始化后，我们可以创建实际的函数：

```
func new
<select Azure queue storage trigger, option 10>
<provide a name, suggested name: js-queue>
```

这应该会产生*图 11.17*中显示的输出：

![使用 func new 命令创建实际函数，然后选择选项 10，Azure 队列存储触发器作为模板。](img/Figure_11.17.jpg)

###### 图 11.17：创建新函数

1.  现在我们需要进行一些配置更改。我们需要为函数提供连接到 Azure 存储的连接字符串，并提供队列名称。首先，打开`local.settings.json`文件以配置存储的连接字符串：

```
vi local.settings.json
```

要进行更改，请按照以下说明操作：

+   按*I*进入插入模式。

+   在`AzureWebJobsStorage`（第 6 行）的行上，用之前复制的连接字符串替换该值。在该行的末尾添加逗号。

+   添加一行，然后在该行上添加以下文本：

```
"QueueConnString": "<your connection string>"
```

![通过编辑 local.settings.json 文件更改 AzureWebJobsStorage 参数的值并添加 QueueConnString。](img/Figure_11.18.jpg)

###### 图 11.18：编辑 local.settings.json 文件

+   保存并关闭文件，按下*Esc*键，输入`:wq!`，然后按*Enter*键。

1.  我们需要编辑的下一个文件是函数配置本身。在这里，我们将引用之前的连接字符串，并提供队列名称。为此，请使用以下命令：

```
vi js-queue/function.json
```

要进行更改，请按照以下说明操作：

+   按*I*进入插入模式。

+   在第 7 行，将队列名称更改为我们创建的队列的名称（`function`）。

+   在第 8 行，将`QueueConnString`添加到连接字段中：![通过编辑 js-queue/function.json 文件输入 queueName 并将连接指向 QueueConnString。](img/Figure_11.19.jpg)

###### 图 11.19：编辑 js-queue/function.json 文件

+   保存并关闭文件，按下*Esc*键，输入`:wq!`，然后按*Enter*键。

1.  我们现在准备将我们的函数发布到 Kubernetes。我们将通过在 Kubernetes 集群上设置 KEDA 来开始发布：

```
kubectl create ns keda
func kubernetes install --keda --namespace keda
```

这将在我们的集群上设置 KEDA。安装过程不会花费很长时间。要验证安装是否成功，请确保 KEDA Pod 正在运行：

```
kubectl get pod -n keda
```

1.  现在我们可以将我们的函数部署到 Kubernetes。我们将配置 KEDA 每 5 秒查看一次队列消息数量（`polling-interval=5`），最多有 15 个副本（`max-replicas=15`），并在删除 Pod 之前等待 15 秒（`cooldown-period=15`）。要部署和配置 KEDA，请使用以下命令：

```
func kubernetes deploy --name js-queue \
--registry <registry name>.azurecr.io \
--polling-interval=5 --max-replicas=15 --cooldown-period=15
```

要验证部署，可以运行以下命令：

```
kubectl get all
```

这将显示部署的所有资源。正如您在*图 11.20*中所看到的，此部署创建了一个部署、ReplicaSet 和 HPA。在 HPA 中，您应该看到当前没有副本在运行：

![kubectl get all 命令的输出显示创建了三个对象，并突出显示水平 Pod 自动缩放器当前没有运行副本。](img/Figure_11.20.jpg)

###### 图 11.20：部署创建了三个对象，现在我们没有运行副本

1.  我们现在将在队列中创建一条消息，以唤醒部署并创建一个 Pod。要查看扩展事件，请运行以下命令：

```
kubectl get hpa -w
```

1.  在队列中创建一条消息，我们将打开一个新的云 shell 会话。要打开新会话，请在云 shell 中选择*打开新会话*按钮：![在 Bash 窗口中选择打开新会话按钮。](img/Figure_11.21.jpg)

###### 图 11.21：打开一个新的云 shell 实例

1.  在这个新的 shell 中，运行以下命令在队列中创建一条消息。

```
az storage message put --queue-name function --connection-string <your connection string> --content "test"
```

创建完这条消息后，切换回到之前的 shell。可能需要几秒钟，但很快，您的 HPA 应该会扩展到 1 个副本。之后，它还应该缩减到 0 个副本：

![执行 kubectl get hpa -w 命令并验证 KEDA 在我们在队列中创建消息时将我们的 Pod 从 0 扩展到 1，并在没有消息时缩减到 0。](img/Figure_11.22.jpg)

###### 图 11.22：在队列中有一条消息时，KEDA 从 0 扩展到 1，然后再次缩减到 0 个副本

我们现在已经创建了一个根据队列中消息数量触发的函数。我们能够验证 KEDA 在我们在队列中创建消息时将我们的 Pod 从 0 扩展到 1，并在没有消息时缩减到 0。在下一节中，我们将执行一个规模测试，并创建多条消息在队列中，看看函数的反应。

### 规模测试函数

在前面的部分中，我们看到了当队列中有单个消息时函数的反应。在这个例子中，我们将向队列中发送 1,000 条消息，看看 KEDA 如何首先扩展我们的函数，然后缩小，最终缩减到零：

1.  在当前的云 shell 中，使用以下命令观察 HPA：

```
kubectl get hpa -w
```

1.  要开始推送消息，我们将打开一个新的云 shell 会话。要打开新会话，请在云 shell 中选择“打开新会话”按钮：

###### 图 11.23：打开一个新的云 shell 实例

1.  要将 1,000 条消息发送到队列中，我们提供了一个名为`sendMessages.py`的 Python 脚本，在代码包中。Cloud Shell 已经安装了 Python 和 pip（Python 包管理器）。要运行此脚本，您首先需要安装两个依赖项：

```
pip3 install azure
pip3 install azure-storage-blob==12.0.0
```

安装完毕后，打开`sendMessages.py`文件：

```
code sendMessages.py
```

编辑第 4 行的存储连接字符串为您的连接字符串：

在 sendMessages.py 文件中，将第 4 行的存储连接字符串编辑为我们的连接字符串。

###### 图 11.24：在第 4 行粘贴您的存储账户的连接字符串

1.  一旦您粘贴了连接字符串，您就可以执行 Python 脚本，向您的队列发送 1,000 条消息：

```
python3 sendMessages.py
```

在消息发送的同时，切换回到之前的云 shell 实例，并观察 KEDA 从 0 扩展到 1，然后观察 HPA 扩展到最大的 15 个副本。HPA 使用 KEDA 提供的指标来做出扩展决策。默认情况下，Kubernetes 不知道 KEDA 提供给 HPA 的 Azure 存储队列中的消息数量。

一旦队列为空，KEDA 将缩减到 0 个副本：

执行 kubectl get hpa -w 命令显示 HPA 将从 0 扩展副本到 1，然后到 4，8 和 15，最后回到 0。

###### 图 11.25：KEDA 将从 0 扩展到 1，HPA 将扩展到 15 个 Pod

这结束了我们在 Kubernetes 上运行无服务器函数的示例。让我们确保清理我们的部署。从我们创建的开发机器中运行以下命令（最后一步将删除此虚拟机。如果您想保留虚拟机，请不要运行最后一步）：

```
kubectl delete secret js-queue
kubectl delete scaled object js-queue
kubectl delete deployment js-queue
func kubernetes remove --namespace keda
az vm delete -g rg-handsonaks -n devMachine
```

#### 注意

删除 KEDA 将显示一些错误。这是因为我们只在我们的集群上安装了 KEDA 的一个子集，而删除过程尝试删除所有组件。

在本节中，我们运行了一个在 Kubernetes 上由存储队列上的消息触发的函数。我们使用了一个名为 KEDA 的组件来实现我们集群中的扩展。我们看到了 KEDA 如何从 0 扩展到 1，然后再缩减到 0。我们还看到了 HPA 如何使用 KEDA 提供的指标来扩展部署。

## 总结

在本章中，我们在我们的 Kubernetes 集群上部署了无服务器函数。为了实现这一点，我们首先创建了一个开发机器和一个 Azure 容器注册表。

我们通过部署使用 HTTP 触发器的函数来开始我们的函数部署。使用 Azure 函数核心工具创建该函数并将其部署到 Kubernetes。

之后，我们在我们的 Kubernetes 集群上安装了一个名为 KEDA 的附加组件。KEDA 允许 Kubernetes 中的无服务器扩展：它允许部署到 0 个 Pod，并提供额外的指标给**水平 Pod 自动缩放器**（**HPA**）。我们使用了一个在 Azure 存储队列中的消息触发的函数。

本章也总结了本书。在整本书中，我们通过多个实际操作的例子介绍了 AKS。书的第一部分侧重于启动应用程序。我们创建了一个 AKS 集群，部署了多个应用程序，并学习了如何扩展这些应用程序。

书的第二部分侧重于运行 AKS 的操作方面。我们讨论了常见的故障以及如何解决它们，将应用程序集成到 Azure AD，并研究了集群的监控。

在书的最后部分，我们深入研究了 AKS 与其他 Azure 服务的高级集成。我们将我们的 AKS 集群与 Azure 数据库和 Azure 事件中心集成，保护了我们的集群，最后，在我们的 AKS 集群上开发了 Azure 函数。

通过完成这本书，您现在应该准备好在 AKS 上构建和运行您的应用程序。

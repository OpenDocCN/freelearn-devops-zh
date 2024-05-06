# 第十四章：提供平台

直到目前为止，这本书的每一章都集中在集群的基础设施上。我们已经探讨了如何部署 Kubernetes，如何保护它以及如何监视它。我们还没有讨论如何部署应用程序。

在这本书的最后一章中，我们将致力于利用我们对 Kubernetes 的了解来构建一个应用部署平台。我们将根据一些常见的企业需求来构建我们的平台。在我们无法直接实现某项需求时，因为在 Kubernetes 上构建平台可以填满一本书，我们会指出并提供一些见解。

在本章中，我们将涵盖以下主题：

+   设计管道

+   准备我们的集群

+   部署 GitLab

+   部署 Tekton

+   部署 ArgoCD

+   使用 OpenUnison 自动化项目入职

# 技术要求

要执行本章的练习，您需要一个干净的 KinD 集群，至少有 8GB 内存，75GB 存储空间和 4 个 CPU。我们将构建的系统是最简化的，但仍需要相当大的计算能力来运行。

您可以在以下 GitHub 存储库中访问本章的代码：[`github.com/PacktPublishing/Kubernetes-and-Docker-The-Complete-Guide`](https://github.com/PacktPublishing/Kubernetes-and-Docker-The-Complete-Guide)。

# 设计管道

“管道”这个术语在 Kubernetes 和 DevOps 世界中被广泛使用。非常简单地说，管道是一个过程，通常是自动化的，它接受代码并使其运行。这通常涉及以下内容：

![图 14.1 - 一个简单的管道](img/Fig_14.1_B15514.jpg)

图 14.1 - 一个简单的管道

让我们快速浏览一下这个过程中涉及的步骤：

1.  将源代码存储在一个中央仓库中，通常是 Git

1.  当代码提交时，构建它并生成构件，通常是一个容器

1.  告诉平台 - 在这种情况下是 Kubernetes - 推出新的容器并关闭旧的容器。

这是一个管道可以变得多么基本，并且在大多数部署中并不太有用。除了构建我们的代码并部署它之外，我们还希望确保扫描容器中已知的漏洞。在进入生产之前，我们可能还希望对我们的容器进行一些自动化测试。在企业部署中，通常存在一个合规性要求，即有人对进入生产负责。考虑到这一点，管道开始变得更加复杂：

![图 14.2 - 具有常见企业要求的管道](img/Fig_14.2_B15514.jpg)

图 14.2 - 具有常见企业要求的管道

管道已经添加了一些额外的步骤，但它仍然是线性的，有一个起点，一个提交。这也非常简单和不切实际。您的应用程序构建在其上的基本容器和库不断更新，因为发现和修补了新的**通用漏洞和暴露**（**CVEs**）（一种常见的目录和识别安全漏洞的方式）。除了有开发人员更新应用程序代码以满足新需求之外，您还希望建立一个系统，用于扫描代码和基本容器是否有可用的更新。这些扫描程序监视您的基本容器，并且可以在新的基本容器就绪时触发构建。虽然扫描程序可以调用 API 来触发管道，但您的管道已经在等待 Git 存储库执行某些操作，因此最好只需向 Git 存储库添加提交或拉取请求以触发管道：

![图 14.3 - 集成了扫描程序的管道](img/Fig_14.3_B15514.jpg)

图 14.3 - 集成了扫描程序的管道

这意味着您的应用程序代码和操作更新都在 Git 中进行跟踪。Git 现在不仅是您的应用程序代码的真相，还是操作更新的真相。当需要进行审计时，您就有了一个现成的变更日志！如果您的政策要求您将更改输入到变更管理系统中，只需从 Git 中导出更改即可。

到目前为止，我们已经专注于我们的应用程序代码，并且在管道的最后放置了**Rollout**。最终的部署步骤通常意味着使用我们新构建的容器来修补**Deployment**或**StatefulSet**，让 Kubernetes 来完成启动新的**Pods**和缩减旧的 Pods 的工作。这可以通过简单的 API 调用来完成，但我们如何跟踪和审计这个变化？真相是什么？

我们在 Kubernetes 中的应用程序被定义为存储在**etcd**中的一系列对象，通常使用 YAML 文件表示为代码。为什么不也将这些文件存储在 Git 存储库中呢？这给我们带来了与将应用程序代码存储在 Git 中相同的好处。我们对应用程序源代码和应用程序操作都有一个统一的真相！现在，我们的管道涉及一些更多的步骤：

![图 14.4 - GitOps 管道](img/Fig_14.4_B15514.jpg)

图 14.4 - GitOps 流水线

在这个图表中，我们的部署更新了一个包含我们应用的 Kubernetes YAML 的 Git 仓库。我们集群内的控制器会监视 Git 的更新，当它发现更新时，将集群与 Git 中的内容同步。它还可以检测集群中的漂移，并将其重新调整到与我们的真相源一致。

这种对 Git 的关注被称为 GitOps。其理念是所有应用的工作都是通过代码完成，而不是直接通过 API。你对这个理念的严格程度可以决定你的平台是什么样子的。接下来，我们将探讨观点如何塑造你的平台。

## 主张性平台

Kelsey Hightower，谷歌的开发者倡导者和 Kubernetes 世界的领导者，曾经说过：“Kubernetes 是一个构建平台的平台。这是一个更好的开始，而不是终点。”当你看看构建基于 Kubernetes 的产品的供应商和项目的格局时，它们都对系统应该如何构建有自己的看法。举个例子，红帽的 OpenShift 容器平台（OCP）希望成为多租户企业部署的一站式服务。它内置了我们讨论过的大量流水线。你定义一个由提交触发的流水线，它构建一个容器并将其推送到自己的内部注册表，然后触发新容器的部署。命名空间是租户的边界。Canonical 是一个不包括任何流水线组件的极简主义发行版。亚马逊、Azure 和谷歌等托管供应商提供了集群的构建模块和托管构建工具，但让你自己构建平台。

没有正确的答案来确定使用哪个平台。每个平台都有自己的观点，对于你的部署来说，选择合适的平台将取决于你自己的需求。根据你的企业规模，部署多个平台也不足为奇！

在看了主张性平台的概念之后，让我们探讨构建流水线的安全影响。

## 保护你的流水线

根据您的起点，这可能会很快变得复杂起来。您的管道有多少是一个集成系统，或者可以用涉及胶带的美国俚语来描述？即使在所有组件都在的平台上，将它们联系在一起通常意味着构建一个复杂的系统。您的管道中的大多数系统都将具有视觉组件。通常，视觉组件是一个仪表板。用户和开发人员可能需要访问该仪表板。您不想为所有这些系统维护单独的帐户，对吧？您会希望为管道的所有组件设置一个登录点和门户。

确定如何对使用这些系统的用户进行身份验证后，下一个问题是如何自动化推出。您的管道的每个组件都需要配置。它可以是通过 API 调用创建的对象，也可以是将 Git 存储库和构建过程与 SSH 密钥联系在一起以自动化安全的复杂过程。在这样一个复杂的环境中，手动创建管道基础设施将导致安全漏洞。这也将导致无法管理的系统。自动化流程并提供一致性将帮助您确保基础设施的安全性并使其易于维护。

最后，从安全的角度来看，了解 GitOps 对我们的集群的影响是很重要的。我们在《第七章》，*将认证集成到您的集群*和《第八章》，*使用 Active Directory 用户的 RBAC 策略*中讨论了对管理员和开发人员进行身份验证以使用 Kubernetes API，并授权访问不同 API 的访问。如果有人可以提交一个分配给他们命名空间的**admin ClusterRole**的**RoleBinding**，并且 GitOps 控制器自动将其推送到集群，那会有什么影响？在设计平台时，考虑开发人员和管理员将如何与其互动。诱人的做法是说“让每个人与他们的应用程序的 Git 注册表互动”，但这意味着您作为集群所有者需要处理许多请求的负担。正如我们在《第八章》，*使用 Active Directory 的 RBAC 策略*中讨论的那样，这可能使您的团队成为企业的瓶颈。了解您的客户在这种情况下是重要的，因为这样可以知道他们希望如何与他们的操作进行交互，即使这不是您的本意。

在涉及 GitOps 和流水线的一些安全方面后，让我们探讨一下典型流水线的要求以及我们将如何构建它。

## 满足我们平台的要求。

Kubernetes 部署，特别是在企业环境中，通常会有以下基本要求：

+   开发和测试环境：至少有两个集群，以测试对应用程序的集群级别的更改的影响

+   开发人员沙盒：开发人员可以构建容器并测试它们，而不必担心对共享命名空间的影响

+   源代码控制和问题跟踪：存储代码并跟踪未完成的任务的地方

除了这些基本要求之外，企业通常还会有额外的要求，例如定期访问审查、基于策略限制访问以及分配对可能影响共享环境的操作负责的工作流程。

对于我们的平台，我们希望尽可能包含这些要求。为了更好地自动化部署到我们的平台，我们将定义每个应用程序具有以下内容：

+   开发命名空间：开发人员是管理员。

+   生产命名空间：开发人员是查看者。

+   **源代码控制项目**：开发人员可以 fork。

+   **构建流程**：由对 Git 的更新触发。

+   **部署流程**：由对 Git 的更新触发。

此外，我们希望我们的开发人员有他们自己的沙盒，这样每个用户都将获得自己的开发命名空间。

重要提示

在实际部署中，您会希望将开发和生产环境分开到不同的集群中。这样可以更容易地测试集群范围的操作，例如升级，而不会影响正在运行的应用程序。我们在一个集群中做所有操作是为了让您更容易自行设置。

为了提供对每个应用程序的访问权限，我们将定义三个角色：

+   **所有者**：应用程序所有者可以批准其他角色在其应用程序内的访问权限。这个角色由应用程序请求者分配，并可以由应用程序所有者分配。所有者还负责推送更改到开发和生产环境中。

+   **开发人员**：这些用户将可以访问应用程序的源代码控制，并可以管理应用程序的开发命名空间。他们可以查看生产命名空间中的对象，但不能编辑任何内容。任何用户都可以请求此角色，并由应用程序所有者批准。

+   **运维人员**：这些用户具有开发人员的能力，但也可以根据需要对生产命名空间进行更改。任何用户都可以请求此角色，并由应用程序所有者批准。

我们还将创建一些环境范围的角色：

+   **系统审批者**：拥有此角色的用户可以批准对任何系统范围角色的访问权限。

+   **集群管理员**：这个角色专门用于管理我们的集群和构成我们流水线的应用程序。任何人都可以请求此角色，并必须得到系统审批者角色的批准。

+   **开发人员**：任何登录的用户都会获得自己的开发命名空间。这些命名空间不能被其他用户请求访问。这些命名空间与任何 CI/CD 基础设施或 Git 存储库没有直接连接。

即使我们的平台非常简单，我们有六个角色需要映射到构成我们流水线的应用程序上。每个应用程序都有自己的身份验证和授权流程，这些角色需要映射到这些流程上。这就是为什么自动化对于集群安全如此重要的一个例子。根据电子邮件请求手动提供访问权限可能会迅速变得难以管理。

开发人员预期通过应用程序进行的工作流程将与我们之前设计的 GitOps 流程一致：

+   应用程序所有者将请求创建一个应用程序。一旦获得批准，将为应用程序代码、流水线构建清单和 Kubernetes 清单创建一个 Git 存储库。还将创建开发和生产命名空间，并创建适当的**RoleBinding**对象。将创建反映每个应用程序角色的组，并将访问这些组的批准委托给应用程序所有者。

+   开发人员和运维人员可以通过请求或直接由应用程序所有者提供来获得对应用程序的访问权限。一旦获得访问权限，预期在开发人员的沙盒和开发命名空间中进行更新。更新是在用户的 Git 存储库分支中进行的，使用拉取请求将代码合并到驱动自动化的主存储库中。

+   所有构建都通过应用程序的源代码中的“脚本”进行控制。

+   所有工件都发布到集中式容器注册表中。

+   所有生产更新必须得到应用程序所有者的批准。

这个基本的工作流程不包括工作流的典型组件，比如代码和容器扫描，定期访问重新认证，或者特权访问的要求。本章的主题很容易成为一本完整的书。目标不是构建一个完整的企业平台，而是为您构建和设计自己的系统提供一个起点。

## 选择我们的技术栈

在本节的前几部分中，我们以一种通用的方式讨论了流水线。现在，让我们进入我们的流水线所需的技术的具体细节。我们之前确定，每个应用程序都有应用程序源代码和 Kubernetes 清单定义。它还必须构建容器。需要一种方式来监视 Git 的更改并更新我们的集群。最后，我们需要一个自动化平台，使所有这些组件能够协同工作。

根据我们对平台的要求，我们希望拥有以下功能的技术：

+   **开源**：我们不希望您为了这本书而购买任何东西！

+   基于 API 的：我们需要能够以自动化的方式配置组件并访问。

+   **具有支持外部身份验证的视觉组件**：这本书侧重于企业，企业中的每个人都喜欢他们的图形用户界面。只是不想为每个应用程序使用不同的凭据。

+   **在 Kubernetes 上支持**：这是一本关于 Kubernetes 的书。

为了满足这些要求，我们将在我们的集群中部署以下组件：

+   **Git 注册表 - GitLab**：GitLab 是一个强大的系统，提供了一个很好的 UI 和体验，用于与支持外部身份验证的 Git 一起工作（即**单点登录（SSO）**）。它集成了问题管理和广泛的 API。它还有一个 Helm 图表，我们已经为本书定制了一个最小的安装。

+   **自动化构建 - Tekton**：最初是 Kubernetes 函数即服务部署的 Knative 项目的构建部分，Tekton 被分离出来成为自己的项目，为通用应用程序提供构建服务。它在 Kubernetes 中运行，所有交互都通过 Kubernetes API 进行。它也有一个早期阶段的仪表板！

+   **容器注册表 - 简单的 Docker 注册表**：有许多功能强大的开源注册表。由于这个部署很快就会变得复杂，我们决定只使用 Docker 提供的注册表。它不会有任何安全性，所以不要在生产中使用！

+   **GitOps - ArgoCD**：ArgoCD 是 Intuit 和 Weaveworks 之间合作构建功能丰富的 GitOps 平台。它是 Kubernetes 本地的，有自己的 API，并将其对象存储为 Kubernetes 自定义资源，使自动化变得更容易。它的 UI 和 CLI 工具都使用 OpenID Connect 与 SSO 集成。

+   **访问、身份验证和自动化 - OpenUnison**：我们将继续使用 OpenUnison 进行对集群的身份验证。我们还将整合我们技术堆栈的 UI 组件，以提供一个平台的单一门户。最后，我们将使用 OpenUnison 的工作流根据我们的角色结构管理对每个系统的访问，并提供一切工作所需的对象。访问将通过 OpenUnison 的自助门户提供。

阅读这个技术栈时，你可能会问：“为什么你没有选择*XYZ*？”Kubernetes 生态系统多样且拥有众多出色的项目和产品供您的集群使用。这绝不是一个确定的技术栈，甚至不是一个“推荐”的技术栈。这是一个满足我们需求并让我们专注于正在实施的流程的应用程序集合，而不是学习特定技术。

您可能还会发现，即使在这个技术栈中，工具之间也存在相当大的重叠。例如，GitLab 具有 GitOps 功能和自己的构建系统，但我们选择不在本章中使用它们。我们这样做是为了让您看到如何将不同的系统连接在一起构建平台。您的平台可能会使用 GitHub 的 SaaS 解决方案进行源代码控制，但在内部运行构建，并与亚马逊的容器注册表结合。我们希望您看到这些系统如何连接在一起构建平台，而不是专注于特定工具。

这一部分是对管道设计理论的深入探讨，以及对构建基于 Kubernetes 的平台的常见要求的审视。我们确定了可以实现这些要求的技术组件以及我们为什么选择它们。有了这些知识，现在是时候开始构建了！

# 准备我们的集群

在我们开始部署技术栈之前，我们需要做一些事情。我建议从一个新的集群开始。如果您正在使用本书中的 KinD 集群，请从一个新的集群开始。我们正在部署几个需要集成的组件，最好从头开始，而不是可能与先前的配置斗争。在我们开始部署构成我们技术栈的应用程序之前，我们将部署 JetStack 的 cert-manager 来自动化证书签发，一个简单的容器注册表，以及用于身份验证和自动化的 OpenUnison。

## 部署 cert-manager

JetStack 是一家专注于 Kubernetes 的咨询公司，他们创建了一个名为**cert-manager**的项目，以便更容易地自动创建和更新证书。该项目通过让您使用 Kubernetes 自定义资源定义发行者，然后在**Ingress**对象上使用注释来使用这些发行者生成证书来工作。最终结果是集群运行正常管理和轮换证书，而不需要生成一个**证书签名请求**（**CSR**）或担心过期！

**cert-manager**项目通常与*Let's Encrypt*（https://letsencrypt.org/）一起提到，以自动发布由商业认可的证书颁发机构免费签名的证书（就像啤酒一样）。这是可能的，因为*Let's Encrypt*自动化了这个过程。证书只有 90 天的有效期，整个过程都是 API 驱动的。为了驱动这种自动化，您必须有一种让*Let's Encrypt*验证您正在尝试获取证书的域的所有权的方法。在本书中，我们使用**nip.io**来模拟 DNS。如果您有一个可以使用并且受**cert-manager**支持的 DNS 服务，比如亚马逊的 Route 53，那么这是一个很好的解决方案。

由于我们使用**nip.io**，我们将部署**cert-manager**与自签名证书颁发机构。这使我们能够拥有一个证书颁发机构，可以快速生成证书，而不必担心域验证的问题。然后，我们将指示我们的工作站信任此证书以及我们部署的应用程序，以便一切都使用正确构建的证书进行保护。

重要提示

对于大多数企业来说，使用自签名证书颁发机构是内部部署的常见做法。这避免了处理商业签名证书无法提供太多价值的潜在验证问题。大多数企业能够通过其 Active Directory 基础架构分发内部证书颁发机构的证书。很可能您的企业也有一种方式可以请求内部证书或可用的通配符。

部署**cert-manager**的步骤如下：

1.  从您的集群中，部署**cert-manager**清单：

**$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager.yaml**

1.  一旦 Pod 在**cert-manager**命名空间中运行，创建一个自签名证书，我们将用作我们的证书颁发机构。在本书的 Git 存储库的**chapter14/shell**目录中有一个名为**makeca.sh**的脚本，它将为您生成此证书：

**$ cd Kubernetes-and-Docker-The-Complete-Guide/chapter14/shell/**

**$ sh ./makeca.sh**

**生成 RSA 私钥，2048 位长模数（2 个质数）**

**.............................................................................................................................................+++++**

....................+++++

e 为 65537（0x010001）

1.  现在有一个带有证书和密钥的 SSL 目录。下一步是从这些文件创建一个 secret，这将成为我们的证书颁发机构：

$ cd ssl/

$ kubectl create secret tls ca-key-pair --key=./tls.key --cert=./tls.crt -n cert-manager

secret/ca-key-pair 已创建

1.  接下来，创建 ClusterIssuer 对象，以便所有的 Ingress 对象都可以拥有正确颁发的证书：

$ cd ../../yaml/

$ kubectl create -f ./certmanager-ca.yaml

clusterissuer.cert-manager.io/ca-issuer 已创建

1.  创建了 ClusterIssuer 后，任何带有 cert-manager.io/cluster-issuer: "ca-issuer"注释的 Ingress 对象都将获得我们为它们创建的由我们授权签名的证书。我们将用于此的一个组件是我们的容器注册表。Kubernetes 使用 Docker 的底层机制来拉取容器，并且 KinD 不会从没有 TLS 或使用不受信任证书的注册表中拉取镜像。为了解决这个问题，我们需要将我们的证书导入到我们的 worker 和节点中：

$ cd ~/

$ kubectl get secret ca-key-pair -n cert-manager -o json | jq -r '.data["tls.crt"]' | base64 -d > internal-ca.crt

$ docker cp internal-ca.crt cluster01-worker:/usr/local/share/ca-certificates/internal-ca.crt

$ docker exec -ti cluster01-worker update-ca-certificates

在/etc/ssl/certs 中更新证书...

1 个添加，0 个移除；完成。

在/etc/ca-certificates/update.d 中运行钩子...

完成。

$ docker restart cluster01-worker

$ docker cp internal-ca.crt cluster01-control-plane:/usr/local/share/ca-certificates/internal-ca.crt

$ docker exec -ti cluster01-control-plane update-ca-certificates

在/etc/ssl/certs 中更新证书...

1 个添加，0 个移除；完成。

在/etc/ca-certificates/update.d 中运行钩子...

完成。

$ docker restart cluster01-control-plane

第一个命令从我们创建的 secret 中提取证书。接下来的一系列命令将证书复制到每个容器中，指示容器信任它，并最后重新启动容器。一旦容器重新启动，等待所有 Pod 重新启动；可能需要几分钟。

重要提示

现在是下载**internal-ca.crt**的好时机；将其安装到您的本地工作站，可能还要安装到您选择的浏览器中。不同的操作系统和浏览器的操作方式不同，因此请查阅相应的文档了解如何操作。信任此证书将使与应用程序交互、推送容器和使用命令行工具变得更加容易。

有了**cert-manager**准备好颁发证书，以及您的集群和您的工作站信任这些证书，下一步是部署容器注册表。

## 部署 Docker 容器注册表

Docker, Inc.提供了一个简单的注册表。这个注册表没有安全性，因此绝对不是生产使用的好选择。**chapter14/yaml/docker-registry.yaml**文件将为我们部署注册表并创建一个**Ingress**对象。在部署之前，编辑此文件，将所有的**192-168-2-140**实例更改为集群 IP 地址的破折号表示法。例如，我的集群正在运行**192.168.2.114**，所以我将**192-168-2-140**替换为**192-168-2-114**。然后，运行**kubectl create**在清单上创建注册表：

$ kubectl create -f ./docker-registry.yaml

namespace/docker-registry created

statefulset.apps/docker-registry created

service/docker-registry created

ingress.extensions/docker-registry created

注册表运行后，您可以尝试从浏览器访问它：

![图 14.5 - 在浏览器中访问容器注册表](img/Fig_14.5_B15514.jpg)

图 14.5 - 在浏览器中访问容器注册表

您不会看到太多，因为注册表没有 Web UI，但您也不应该收到证书错误。这是因为我们部署了**cert-manager**并颁发了签名证书！当我们的注册表运行时，部署的最后一个组件是 OpenUnison。

## 部署 OpenUnison

在*第七章*，*将身份验证集成到您的集群中*，我们介绍了 OpenUnison 来验证对我们 KinD 部署的访问。OpenUnison 有两种版本。第一种是我们已经部署的登录门户，它允许我们使用中央来源进行身份验证并将组信息传递给我们的 RBAC 策略。第二种是一个自动化门户，我们将以此为基础集成将管理我们的流水线的系统。该门户还将为我们提供一个中央 UI，用于请求创建项目并管理对我们项目系统的访问。

我们定义了我们部署的每个项目将具有跨多个系统的三个"角色"。您的企业是否允许您为我们创建的每个项目创建和管理组？有些可能会，但 Active Directory 在大多数企业中是一个关键组件，写访问权限可能难以获得。你运行 Active Directory 的人可能不是你在管理集群时向之报告的人，这会使你难以获得你具有管理权限的 Active Directory 区域。OpenUnison 自动化门户允许您使用可以轻松查询的本地组来管理访问，就像使用 Active Directory 一样，但您有权利管理它们。不过，我们仍将对我们的中央 SAML 提供程序进行身份验证。

为了促进 OpenUnison 的自动化能力，我们需要部署一个数据库来存储持久数据和一个 SMTP 服务器，以通知用户何时有未处理的请求或请求何时已完成。对于数据库，我们将部署开源的 MariaDB。对于**简单邮件传输协议**（**SMTP**）（电子邮件）服务器，大多数企业对发送电子邮件有非常严格的规定。我们不想为通知设置电子邮件，因此我们将运行一个只会忽略所有 SMTP 请求的"黑洞"电子邮件服务：

1.  首先，从书的 GitHub 存储库运行**chapter14/yaml/mariadb.yaml**清单。不需要进行任何更改。

1.  接下来，部署 SMTP 黑洞：

**$ kubectl create ns blackhole**

**命名空间/blackhole 已创建**

**$ kubectl create deployment blackhole --image=tremolosecurity/smtp-blackhole -n blackhole**

部署.apps/blackhole 已创建

**$ kubectl expose deployment/blackhole --type=ClusterIP --port 1025 --target-port=1025 -n blackhole**

**服务/blackhole 已公开**

1.  有了 MariaDB 和我们的 SMTP 服务部署，我们就能够部署 OpenUnison。按照*第七章*的*Deploying OpenUnison*部分中的*步骤 1-5*，部署 OpenUnison 操作员和 Kubernetes 仪表板。

1.  接下来，在集群中创建一个**Secret**，用于存储访问 MariaDB 和 SMTP 服务的凭据。我们为 MariaDB 硬编码了密码，以简化部署，因此请确保为生产数据库帐户生成长且随机的密码！在集群中创建以下**Secret**：

apiVersion: v1

type: Opaque

metadata:

name: orchestra-secrets-source

namespace: openunison

数据：

K8S_DB_SECRET: aW0gYSBzZWNyZXQ=

SMTP_PASSWORD: ""

OU_JDBC_PASSWORD: c3RhcnR0MTIz

unisonKeystorePassword: aW0gYSBzZWNyZXQ=

kind: Secret

1.  我们将在*第七章*的*Configuring your cluster for impersonation*部分中重用我们在*步骤 2*中使用的 Helm 值，有三个更改。

1.  首先，将图像从**docker.io/tremolosecurity/openunison-k8s-login-saml2:latest**更改为**docker.io/tremolosecurity/openunison-k8s-saml2:latest**。

1.  接下来，将**internal-ca.crt**文件进行 Base64 编码为单行，并将其添加到**values.yaml**的**trusted_certs**部分：

**$ base64 -w 0 < internal-ca.crt**

**LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0 tCk1JSUREVENDQWZXZ0F3SUJ…**

1.  添加 SMTP 和数据库部分。**values.yaml**的更新如下。我删除了大部分未更改的部分以节省空间：

trusted_certs:

- name: internal-ca

pem_b64: LS0tLS1CRUdJTiB…

saml:

idp_url: https://portal.apps.tremolo.io/idp-test/metadata/dfbe4040-cd32-470e-a9b6-809c8f857c40

metadata_xml_b64: ""

database:

hibernate_dialect: org.hibernate.dialect.MySQL5InnoDBDialect

quartz_dialect: org.quartz.impl.jdbcjobstore.StdJDBCDelegate

driver: com.mysql.jdbc.Driver

url: jdbc:mysql://mariadb.mariadb.svc.cluster.local:3306/unison

user: unison

validation: SELECT 1

smtp:

host: blackhole.blackhole.svc.cluster.local

port: 1025

user: none

from: donotreply@domain.com

tls: false

1.  使用 Helm 图表部署 OpenUnison：

**$ helm install orchestra tremolo/openunison-k8s-saml2 --namespace openunison -f ./openunison-values.yaml**

1.  一旦部署了 OpenUnison，编辑**orchestra** OpenUnison 对象，删除**unison-ca**密钥。删除以下类似的块：

- create_data:

ca_cert: true

key_size: 2048

server_name: k8sou.apps.192-168-2-114.nip.io

sign_by_k8s_ca: false

subject_alternative_names:

- k8sdb.apps.192-168-2-114.nip.io

- k8sapi.apps.192-168-2-114.nip.io

import_into_ks: certificate

name: unison-ca

tls_secret_name: ou-tls-certificate

1.  删除**ou-tls-certificate** **Secret**：

**$ kubectl delete secret ou-tls-certificate -n openunison**

**secret "ou-tls-certificate" deleted**

1.  编辑**openunison** **Ingress**对象，将**cert-manager.io/cluster-issuer: ca-issuer**添加到**annotations**列表中。

1.  使用*第七章*的*配置集群进行模拟*部分的*步骤 4-6*完成与测试身份提供者的 SSO 集成。

1.  登录 OpenUnison，然后退出。

1.  OpenUnison 自动化门户不会处理来自测试身份提供者的组。为了成为集群管理员，您必须被“引导”到环境的组中：

**$ kubectl exec -ti mariadb-0 -n mariadb -- mysql -u \**

**  unison --password='startt123' \**

**  -e "insert into userGroups (userId,groupId) values (2,1);" \**

**  unison**

**$ kubectl exec -ti mariadb-0 -n mariadb -- mysql -u \**

**  unison --password='startt123' \**

**  -e "insert into userGroups (userId,groupId) values (2,2);" \  unison**

1.  最后，重新登录。您将成为集群的全局管理员和集群管理员。

部署了 OpenUnison 后，您现在可以远程管理您的集群。根据您访问集群的方式，可能更容易使用您的工作站直接管理本章其余步骤中的集群。

您会注意到 OpenUnison 现在有不同的“徽章”。除了获取令牌或访问仪表板外，您还可以请求创建新的命名空间或访问 ActiveMQ 仪表板。您还会看到标题栏有额外的选项，比如**请求访问**。OpenUnison 将成为我们的自助门户，用于部署我们的流水线，而无需手动在应用程序或集群中创建对象。在谈论使用 OpenUnison 自动部署流水线之前，我们不会详细讨论这些内容。

准备好您的集群后，下一步是部署我们流水线的组件。

# 部署 GitLab

在构建 GitOps 流水线时，最重要的组件之一是 Git 存储库。除了 Git 之外，GitLab 还有许多组件，包括用于浏览代码的 UI，用于编辑代码的基于 Web 的**集成开发环境**（**IDE**），以及用于在多租户环境中管理对项目的访问的强大身份实现。这使得它成为我们平台的一个很好的解决方案，因为我们可以将我们的“角色”映射到 GitLab 组。

在本节中，我们将在我们的集群中部署 GitLab，并创建两个简单的存储库，以便在我们重新部署 Tekton 和 ArgoCD 时使用。当我们重新访问 OpenUnison 以自动化我们的流水线部署时，我们将专注于自动化步骤。

GitLab 使用 Helm 图表部署。对于本书，我们构建了一个自定义**values**文件来运行最小的安装。虽然 GitLab 具有类似于 ArgoCD 和 Tekton 的功能，但我们不会使用它们。我们也不想担心高可用性。让我们开始：

1.  创建一个名为**gitlab**的新命名空间：

**$ kubectl create ns gitlab**

命名空间/gitlab 已创建

1.  我们需要将我们的证书颁发机构作为一个秘密添加到 GitLab，以便 GitLab 信任与 OpenUnison 对话以及我们最终为 Tekton 创建的 webhooks：

**$ kubectl get secret ca-key-pair \**

**  -n cert-manager -o json | jq -r '.data["tls.crt"]' \**

**  | base64 -d > tls.crt**

**$ kubectl create secret generic \**

**  internal-ca --from-file=. -n gitlab**

1.  在您喜欢的文本编辑器中打开**chapter14/gitlab/secret/provider**。用您集群的完整域后缀替换**local.tremolo.dev**。例如，我的集群运行在**192.168.2.114**上，所以我使用**apps.192-168-2-114.nip.io**后缀。这是我的更新后的**Secret**：

名称：openid_connect

标签：OpenUnison

参数：

名称：openid_connect

范围：

- openid

- 档案

响应类型：代码

发行人：**https://k8sou.apps.192-168-2-114.nip.io/auth/idp/k8sIdp**

发现：true

客户端认证方法：查询

uid 字段：sub

将范围发送到令牌端点：false

客户端选项：

标识符：gitlab

秘密：secret

重定向 URI：**https://gitlab.apps.192-168-2-114.nip.io/users/auth/openid_connect/callback**

重要提示

我们使用**secret**作为客户端秘密。这不应该在生产集群中执行。如果您使用我们的模板作为起点将 GitLab 部署到生产环境，请确保更改此设置。

1.  为 GitLab 创建与 OpenUnison 集成以进行 SSO 的**secret**。当我们重新访问 OpenUnison 时，我们将完成这个过程：

**$ kubectl create secret generic gitlab-oidc --from-file=. -n gitlab**

**secret/gitlab-oidc 已创建**

1.  编辑**chapter14/yaml/gitlab-values.yaml**。就像*步骤 3*中一样，用您的集群的完整域后缀替换**local.tremolo.dev**。例如，我的集群运行在**192.168.2.114**上，所以我使用**apps.192-168-2-114.nip.io**后缀。

1.  如果您的集群在单个虚拟机上运行，现在是创建快照的好时机。如果在 GitLab 部署过程中出现问题，可以更容易地恢复到快照，因为 Helm 图表在删除后没有很好地清理自己。

1.  将图表添加到您的本地存储库并部署 GitLab：

**$ helm repo add gitlab https://charts.gitlab.io**

**$ "gitlab"已添加到您的存储库**

**$ helm install gitlab gitlab/gitlab -n gitlab -f chapter14/yaml/gitlab-values.yaml**

**NAME: gitlab**

**LAST DEPLOYED: Sat Aug  8 14:50:13 2020**

**NAMESPACE: gitlab**

**STATUS: deployed**

**REVISION: 1**

**警告：已禁用使用 cert-manager 进行自动 TLS 证书生成，并且未提供 TLS 证书。已生成自签名证书。**

1.  运行需要一些时间。即使 Helm 图表已安装，所有 Pod 完成部署可能需要 15-20 分钟。在 GitLab 初始化时，我们需要更新 web 前端的**Ingress**对象以使用由我们的证书颁发机构签名的证书。编辑**gitlab**命名空间中的**gitlab-webservice** **Ingress**对象。将**kubernetes.io/ingress.class: gitlab-nginx**注释更改为**kubernetes.io/ingress.class: nginx**。还要将**secretName**从**gitlab-wildcard-tls**更改为**gitlab-web-tls**：

apiVersion: extensions/v1beta1

类型：Ingress

元数据：

注释：

cert-manager.io/cluster-issuer: ca-issuer

**kubernetes.io/ingress.class: nginx**

kubernetes.io/ingress.provider: nginx

。

。

。

tls:

- 主机：

- gitlab.apps.192-168-2-114.nip.io

**secretName: gitlab-web-tls**

状态：

负载均衡器：{}

1.  接下来，我们需要更新我们的 GitLab shell 以接受端口**2222**上的 SSH 连接。这样，我们就可以提交代码，而不必担心阻止对 KinD 服务器的 SSH 访问。编辑**gitlab**命名空间中的**gitlab-gitlab-shell** **Deployment**。找到**containerPort: 2222**并在其下插入**hostPort: 2222**，确保保持间距。一旦 Pod 重新启动，您就可以在端口**2222**上通过 SSH 连接到您的 GitLab 主机名。

1.  要获取登录 GitLab 的根密码，请从生成的密钥中获取：

**$ kubectl get secret gitlab-gitlab-initial-root-password -o json -n gitlab | jq -r '.data.password' | base64 -d**

**10xtSWXfbvH5umAbCk9NoN0wAeYsUo9jRVbXrfLn KbzBoPLrCGZ6kYRe8wdREcDl**

现在，您可以通过转到**https://gitlab.apps.x-x-x-x.nip.io**登录到您的 GitLab 实例，其中**x-x-x-x**是您服务器的 IP。由于我的服务器运行在**192.168.2.114**上，我的 GitLab 实例运行在**https://gitlab.apps.192-168-2-114.nip.io/**上。

## 创建示例项目

为了探索 Tekton 和 ArgoCD，我们将创建两个项目。一个用于存储简单的 Python Web 服务，另一个用于存储运行服务的清单。让我们开始：

1.  GitLab 屏幕顶部将要求您添加 SSH 密钥。现在就这样做，以便我们可以提交代码。由于我们将通过 SAML 集中进行身份验证，GitLab 不会有用于身份验证的密码。

1.  创建一个名为**hello-python**的项目。保持可见性为**私有**。

1.  使用 SSH 克隆项目。因为我们在端口**2222**上运行，所以需要更改 GitLab 提供的 URL 为正确的 SSH URL。例如，我的 GitLab 实例给我提供的 URL 是 git@gitlab.apps.192-168-2-114.nip.io:root/hello-python.git。这需要更改为 ssh://git@gitlab.apps.192-168-2-114.nip.io:2222/root/hello-python.git。

1.  克隆后，将**章节 14/Python-你好**的内容复制到您的存储库中，并推送到 GitLab：

**$ cd 章节 14/Python-你好**

**$ git archive --format=tar HEAD > /path/to/hello-python/data.tar**

**$ cd /path/to/hello-python**

**$ tar -xvf data.tar**

**README.md**

**source/**

**source/Dockerfile**

**source/helloworld.py**

**source/requirements.txt**

**$ git add ***

**$ git commit -m 'initial commit'**

**$ git push**

1.  在 GitLab 中，创建另一个名为**hello-python-operations**的项目，可见性设置为私有。克隆此项目，并将**章节 14/Python-你好-operations**的内容复制到存储库中，然后推送它。

现在 GitLab 部署了一些示例代码，我们可以继续下一步，构建一个实际的流水线！

# 部署 Tekton

Tekton 是我们平台使用的流水线系统。最初是 Knative 项目的一部分，用于在 Kubernetes 上构建函数即服务，后来 Tekton 被拆分为自己的项目。Tekton 与您可能运行的其他流水线技术之间最大的区别在于，Tekton 是 Kubernetes 本地的。从其执行系统、定义和用于自动化的 webhooks，都可以在几乎任何您能找到的 Kubernetes 发行版上运行。例如，我们将在 KinD 中运行它，而 Red Hat 已经开始在 OpenShift 4.1 中将 Tekton 作为主要的流水线技术。

部署 Tekton 的过程非常简单。Tekton 是一系列操作员，用于查找定义构建流水线的自定义资源的创建。部署本身只需要几个**kubectl**命令：

$ kubectl apply --filename \  https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

$ kubectl apply --filename \ https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml

第一条命令部署了运行 Tekton 流水线所需的基本系统。第二条命令部署了构建 webhooks 所需的组件，以便在代码推送后立即启动流水线。一旦两条命令都完成并且**tekton-pipelines**命名空间中的 Pod 正在运行，您就可以开始构建流水线了！我们将以 Python Hello World web 服务作为示例。

## 构建 Hello World

我们的 Hello World 应用非常简单。这是一个简单的服务，回显必要的“hello”和运行服务的主机，以便我们觉得我们的服务正在做一些有趣的事情。由于服务是用 Python 编写的，我们不需要“构建”二进制文件，但我们确实想要构建一个容器。构建容器后，我们希望更新我们运行命名空间的 Git 存储库，并让我们的 GitOps 系统协调更改以重新部署我们的应用程序。我们的构建步骤如下：

1.  检出我们的最新代码。

1.  基于时间戳创建一个标签。

1.  构建我们的镜像。

1.  推送到我们的注册表。

1.  在**operations**命名空间中修补一个部署的 YAML 文件。

我们将逐个构建我们的流水线对象。第一组任务是创建一个 SSH 密钥，Tekton 将使用它来拉取我们的源代码：

1.  创建一个 SSH 密钥对，我们将在流水线中使用它来检出我们的代码。在提示输入密码时，只需按*Enter*跳过添加密码：

**$ ssh-keygen -f ./gitlab-hello-python**

1.  登录到 GitLab 并导航到我们创建的**hello-python**项目。 点击**设置** | **存储库** | **部署密钥**，然后点击**展开**。 使用**tekton**作为标题，并将刚刚创建的**github-hello-python.pub**文件的内容粘贴到**密钥**部分。 保持**不允许写访问** *未选中*，然后点击**添加密钥**。

1.  接下来，创建**python-hello-build**命名空间和以下秘密。 用我们在*步骤 1*中创建的**gitlab-hello-python**文件的 Base64 编码内容替换**ssh-privatekey**属性。 注释是告诉 Tekton 要使用此密钥的服务器。 服务器名称是 GitLab 命名空间中的**Service**：

api 版本：v1

数据：

ssh-privatekey：...

种类：秘密

元数据：

注释：

tekton.dev/git-0：gitlab-gitlab-shell.gitlab.svc.cluster.local

名称：git-pull

命名空间：python-hello-build

类型：kubernetes.io/ssh-auth

1.  创建一个我们将用于我们的管道推送到**operations**存储库的 SSH 密钥对。 在提示输入密码时，只需按*Enter*跳过添加密码：

**$ ssh-keygen -f ./gitlab-hello-python-operations**

1.  登录到 GitLab 并导航到我们创建的**hello-python-operations**项目。 点击**设置** | **存储库** | **部署密钥**，然后点击**展开**。 使用**tekton**作为标题，并将刚刚创建的**github-hello-python-operations.pub**文件的内容粘贴到**密钥**部分。 确保**允许写访问**已*选中*，然后点击**添加密钥**。

1.  接下来，创建以下秘密。 用我们在*步骤 4*中创建的**gitlab-hello-python-operations**文件的 Base64 编码内容替换**ssh-privatekey**属性。 注释是告诉 Tekton 要使用此密钥的服务器。 服务器名称是我们在 GitLab 命名空间中*步骤 6*中创建的**Service**：

api 版本：v1

数据：

ssh-privatekey：...

种类：秘密

元数据：

名称：git-write

命名空间：python-hello-build

类型：kubernetes.io/ssh-auth

1.  为任务运行创建一个服务帐户，就像我们的秘密一样：

**$ kubectl create -f chapter14/tekton-serviceaccount.yaml**

1.  我们需要一个包含**git**和**kubectl**的容器。 我们将构建**chapter14/docker/PatchRepoDockerfile**并将其推送到我们的内部注册表。 确保用您服务器的 IP 地址的主机名替换**192-168-2-114**：

**$ docker build -f ./PatchRepoDockerfile -t \**

**  docker.apps.192-168-2-114.nip.io/gitcommit/gitcommit .**

**$ docker push \**

**  docker.apps.192-168-2-114.nip.io/gitcommit/gitcommit**

每个**Task**对象都可以接受输入并生成可以与其他**Task**对象共享的结果。Tekton 可以为运行（无论是**TaskRun**还是**PipelineRun**）提供一个工作区，其中可以存储和检索状态。写入工作区允许我们在**Task**对象之间共享数据。

在部署任务和流水线之前，让我们逐个步骤地了解每个任务所做的工作。第一个任务生成图像标记并获取最新提交的 SHA 哈希。完整的源代码在**chapter14/yaml/tekton-task1.yaml**中：

- name: create-image-tag

image: docker.apps.192-168-2-114.nip.io/gitcommit/gitcommit

脚本：|-

#!/usr/bin/env bash

导出 IMAGE_TAG=$(date +"%m%d%Y%H%M%S")

echo -n "$(resources.outputs.result-image.url):$IMAGE_TAG" > /tekton/results/image-url

echo "'$(cat /tekton/results/image-url)'"

cd $(resources.inputs.git-resource.path)

RESULT_SHA="$(git rev-parse HEAD | tr -d '\n')"

echo "最后提交：$RESULT_SHA"

echo -n "$RESULT_SHA" > /tekton/results/commit-tag

任务中的每个步骤都是一个容器。在这种情况下，我们使用了之前构建的容器，其中包含**kubectl**和**git**。我们不需要**kubectl**来执行此任务，但我们需要**git**。代码的第一个块从**result-image** URL 和时间戳生成图像名称。我们可以使用最新的提交，但我喜欢使用时间戳，这样我就可以快速了解容器的年龄。我们将完整的图像 URL 保存到**/text/results/image-url**，这对应于我们在任务中定义的一个结果，名为**image-url**。这可以通过我们的流水线或其他任务引用，引用为**$(tasks.generate-image-tag.results.image-url)**，其中**generate-image-tag**是我们**Task**的名称，**image-url**是我们结果的名称。

我们的下一个任务，在**chapter14/yaml/tekton-task2.yaml**中，使用 Google 的 Kaniko 项目（[`github.com/GoogleContainerTools/kaniko`](https://github.com/GoogleContainerTools/kaniko)）从我们应用程序的源代码生成一个容器。Kaniko 允许您生成一个容器，而无需访问 Docker 守护程序。这很棒，因为您不需要特权容器来构建您的镜像：

steps:

- args:

- --dockerfile=$(params.pathToDockerFile)

- --destination=$(params.imageURL)

- --context=$(params.pathToContext)

- --verbosity=debug

- --skip-tls-verify

命令：

- /kaniko/executor

环境：

- 名称：DOCKER_CONFIG

值：/tekton/home/.docker/

镜像：gcr.io/kaniko-project/executor:v0.16.0

名称：build-and-push

资源：{}

Kaniko 容器是所谓的“无发行版”容器。它没有构建在底层 shell 上，也没有许多您可能习惯的命令行工具。它只是一个单一的二进制文件。这意味着任何变量操作，比如为镜像生成标签，都需要在此步骤之前完成。请注意，正在创建的镜像并不引用我们在第一个任务中创建的结果。它代替引用了一个名为**imageURL**的参数。虽然我们可以直接引用结果，但这样做会使得测试这个任务变得更加困难，因为它现在与第一个任务紧密绑定。通过使用由我们的流水线设置的参数，我们可以单独测试这个任务。运行一次后，这个任务将生成并推送我们的容器。

我们在**chapter14/yaml/tekton-task-3.yaml**中的最后一个任务是触发 ArgoCD 来部署新的容器：

- 镜像：docker.apps.192-168-2-114.nip.io/gitcommit/gitcommit

名称：patch-and-push

资源：{}

脚本：|-

#!/bin/bash

export GIT_URL="$(params.gitURL)"

export GIT_HOST=$(sed 's/.*[@]\(.*\)[:].*/\1/' <<< "$GIT_URL")

mkdir /usr/local/gituser/.ssh

cp /pushsecret/ssh-privatekey /usr/local/gituser/.ssh/id_rsa

chmod go-rwx /usr/local/gituser/.ssh/id_rsa

ssh-keyscan -H $GIT_HOST > /usr/local/gituser/.ssh/known_hosts

cd $(workspaces.output.path)

git clone $(params.gitURL) .

kubectl patch --local -f src/deployments/hello-python.yaml -p '{"spec":{"template":{"spec":{"containers":[{"name":"python-hello","image":"$(params.imageURL)"}]}}}}' -o yaml > /tmp/hello-python.yaml

cp /tmp/hello-python.yaml src/deployments/hello-python.yaml

git add src/deployments/hello-python.yaml

git commit -m 'commit $(params.sourceGitHash)'

git push

第一段代码将 SSH 密钥复制到我们的主目录中，生成**known_hosts**，并将我们的存储库克隆到我们在**Task**中定义的工作空间中。我们不依赖 Tekton 从我们的**operations**存储库中拉取代码，因为 Tekton 假设我们不会推送代码，所以它会断开源代码与我们的存储库的连接。如果我们尝试运行提交，它将失败。由于这一步是一个容器，我们不希望尝试向其写入，因此我们创建了一个带有**emptyDir**的工作空间，就像我们可能运行的**Pod**中的**emptyDir**一样。我们还可以基于持久卷定义工作空间。这对于加快下载依赖项的构建可能会有所帮助。

我们正在从**/pushsecret**复制 SSH 密钥，该密钥在任务的卷中定义。我们的容器以用户**431**身份运行，但 SSH 密钥由 Tekton 作为 root 挂载。我们不想运行特权容器来复制来自**Secret**的密钥，因此我们将其挂载为一个普通的**Pod**。

一旦我们克隆了我们的存储库，我们就会使用最新的镜像对部署进行修补，最后，使用我们应用程序存储库中源提交的哈希进行提交更改。现在我们可以追踪图像到生成它的提交！与我们的第二个任务一样，我们不直接引用任务的结果，以便更容易进行测试。

我们将这些任务汇总到一个流水线中-具体来说，**chapter14/yaml/tekton-pipeline.yaml**。这个 YAML 文件有好几页长，但关键部分定义了我们的任务并将它们链接在一起。您不应该在流水线中硬编码值。看一下我们流水线中第三个任务的定义：

- 名称：更新操作-git

任务引用：

名称：patch-deployment

参数：

- 名称：imageURL

值：$(tasks.generate-image-tag.results.image-url)

- 名称：gitURL

值：$(params.gitPushUrl)

- 名称：sourceGitHash

值：$(tasks.generate-image-tag.results.commit-tag)

工作空间：

- 名称：输出

工作空间：输出

我们引用参数和任务结果，但没有硬编码。这使得我们的**Pipeline**可重用。我们还在第二和第三个任务中包含了**runAfter**指令，以确保我们的任务按顺序运行。否则，任务将并行运行。鉴于每个任务都依赖于其前面的任务，我们不希望它们同时运行。接下来，让我们部署我们的流水线并运行它：

1.  将**chapter14/yaml/tekton-source-git.yaml**文件添加到您的集群中； 这告诉 Tekton 从何处拉取您的应用程序代码。

1.  编辑**chapter14/yaml/tekton-image-result.yaml**，将**192-168-2-114**替换为服务器 IP 地址的哈希表示，并将其添加到您的集群中。

1.  编辑**chapter14/yaml/tekton-task1.yaml**，将图像主机替换为您的 Docker 注册表的主机，并将文件添加到您的集群中。

1.  将**chapter14/yaml/tekton-task2.yaml**添加到您的集群。

1.  编辑**chapter14/yaml/tekton-task3.yaml**，将图像主机替换为您的 Docker 注册表的主机，并将文件添加到您的集群中。

1.  将**chapter14/yaml/tekton-pipeline.yaml**添加到您的集群。

1.  将**chapter14/yaml/tekton-pipeline-run.yaml**添加到您的集群。

您可以使用**kubectl**检查管道的进展，或者您可以使用 Tekton 的 CLI 工具**tkn**（[`github.com/tektoncd/cli`](https://github.com/tektoncd/cli)）。 运行**tkn pipelinerun describe build-hello-pipeline-run -n python-hello-build**将列出构建的进度。 您可以通过重新创建您的**run**对象来重新运行构建，但这并不是非常有效。 此外，我们真正想要的是我们的管道在提交时运行！

## 自动构建

我们不想手动运行构建。 我们希望构建自动化。 Tekton 提供了触发器项目来提供 webhooks，因此每当 GitLab 收到提交时，它可以告诉 Tekton 为我们构建一个**PipelineRun**对象。 设置触发器涉及创建一个 Pod，它有自己的服务帐户，可以创建**PipelineRun**对象，一个用于该 Pod 的服务，以及一个**Ingress**对象来托管对 Pod 的 HTTPS 访问。 您还希望使用秘密保护 webhook，以免意外触发。 让我们将这些对象部署到我们的集群中：

1.  将**chapter14/yaml/tekton-webhook-cr.yaml**添加到您的集群。 此**ClusterRole**将被任何想要为构建提供 webhook 的命名空间使用。

1.  编辑**chapter14/yaml/tekton-webhook.yaml**。 文件底部是一个**Ingress**对象。 将**192-168-2-114**更改为代表您集群的 IP，用破折号代替点。 然后，将文件添加到您的集群中：

apiVersion：extensions/v1beta1

kind：Ingress

元数据：

名称：gitlab-webhook

namespace：python-hello-build

注释：

cert-manager.io/cluster-issuer：ca-issuer

规格：

规则：

- 主机："python-hello-application.build.**192-168-2-114**.nip.io"

http：

paths：

- 后端：

serviceName：el-gitlab-listener

servicePort: 8080

pathType: ImplementationSpecific

tls:

- hosts:

- "python-hello-application.build.**192-168-2-114**.nip.io"

secretName: ingresssecret

1.  登录 GitLab。转到**管理区域** | **网络**。单击**出站请求**旁边的**展开**。勾选**允许来自 Web 钩子和服务的本地网络请求**，然后单击**保存更改**。

1.  转到我们创建的**hello-python**项目，单击**设置** | **Webhooks**。对于 URL，请使用您的**Ingress**主机和 HTTPS - 例如，**https://python-hello-application.build.192-168-2-114.nip.io/**。对于**Secret Token**，使用**notagoodsecret**，对于**Push events**，将分支名称设置为**master**。最后，单击**添加 Webhook**。

1.  添加后，单击**测试**，选择**推送事件**。如果一切配置正确，应该已创建一个新的**PipelineRun**对象。您可以运行**tkn pipelinerun list -n python-hello-build**来查看运行列表；应该有一个新的正在运行。几分钟后，您将在**python-hello-operations**项目中获得一个新的容器和一个已修补的部署！

在本节中，我们涵盖了相当多的内容，以构建我们的应用程序并使用 GitOps 进行部署。好消息是一切都是自动化的；推送将创建我们应用程序的新实例！坏消息是我们不得不创建超过十几个 Kubernetes 对象，并手动更新我们在 GitLab 中的项目。在最后一节中，我们将自动化这个过程。首先，让我们部署 ArgoCD，以便我们可以运行我们的应用程序！

# 部署 ArgoCD

到目前为止，我们有一种进入我们集群的方法，一种存储代码的方法，以及一个用于构建我们的代码和生成镜像的系统。我们平台的最后一个组件是我们的 GitOps 控制器。这是让我们将清单提交到我们的 Git 存储库并对我们的集群进行更改的部分。ArgoCD 是 Intuit 和 Weaveworks 之间的合作。它提供了一个很好的 UI，并由自定义资源和 Kubernetes 本机**ConfigMap**和**Secret**对象的组合驱动。它有一个 CLI 工具，Web 和 CLI 工具都与 OpenID Connect 集成，因此很容易使用我们的 OpenUnison 添加 SSO。让我们部署 ArgoCD 并使用它来启动我们的**hello-python**网络服务：

1.  使用标准的 YAML 部署[`argoproj.github.io/argo-cd/getting_started/`](https://argoproj.github.io/argo-cd/getting_started/)：

**$ kubectl create namespace argocd**

**$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml**

1.  通过编辑 **chapter14/yaml/argocd-ingress.yaml** 为 ArgoCD 创建 **Ingress** 对象。用你的 IP 地址替换所有的 **192-168-2-140**，用破折号替换点。我的服务器 IP 是 **192.168.2.114**，所以我使用 **192-168-2-114**。完成后，将文件添加到你的集群中。

1.  通过运行 **kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2** 获取根密码。保存这个密码。

1.  编辑 **argocd** 命名空间中的 **argocd-server** **Deployment**。在命令中添加 **--insecure**：

规范：

容器：

- command:

- argocd-server

- --staticassets

- /shared/app

**- --repo-server**

**        - argocd-repo-server:8081**

**        - --insecure**

1.  现在你可以通过访问你在*步骤 2*中定义的 **Ingress** 主机来登录到 ArgoCD。你还需要从 https://github.com/argoproj/argo-cd/releases/latest 下载 ArgoCD CLI 实用程序。下载完成后，运行 **./argocd login grpc-argocd.apps.192-168-2-114.nip.io** 来登录，用你服务器的 IP 代替 **192-168-2-114**，用破折号代替点。

1.  创建 **python-hello** 命名空间。

1.  在我们添加 GitLab 仓库之前，我们需要告诉 ArgoCD 信任我们的 GitLab 实例的 SSH 主机。因为我们将让 ArgoCD 直接与 GitLab shell 服务通信，我们需要为该服务生成 **known_host**。为了更容易做到这一点，我们包含了一个脚本，该脚本将从集群外部运行 **known_host**，但重写内容，就好像它是从集群内部运行的一样。运行 **chapter14/shell/getSshKnownHosts.sh** 脚本，并将输出导入 **known_host** 的 **argocd** 命令中。记得将主机名更改为反映你自己集群的 IP 地址：

**$ ./chapter14/shell/getSshKnownHosts.sh gitlab.apps.192-168-2-114.nip.io | argocd cert add-ssh --batch**

**输入 SSH 已知主机条目，每行一个。完成后按 CTRL-D。**

**成功创建了 3 个 SSH 已知主机条目**

1.  接下来，我们需要生成一个 SSH 密钥来访问 **python-hello-operations** 仓库：

**$ ssh-keygen -f ./argocd-python-hello**

1.  通过转到项目并单击**设置** | **存储库**，将公钥添加到**python-hello-operations**存储库。在**部署密钥**旁边，单击**展开**。对于**标题**，使用**argocd**。使用**argocd-python-hello.pub**的内容，然后单击**添加密钥**。然后，使用 CLI 将密钥添加到 ArgoCD，并将公共 GitLab 主机替换为**gitlab-gitlab-shell** **Service**主机名：

**$ argocd repo add git@gitlab-gitlab-shell.gitlab.svc.cluster.local:root/hello-python-operations.git --ssh-private-key-path ./argocd-python-hello**

**存储库'git@gitlab-gitlab-shell.gitlab.svc.cluster.local:root/hello-python-operations.git'已添加**

1.  我们的最后一步是创建一个**Application**对象。您可以通过 Web UI 或 CLI 创建它。您还可以通过在**argocd**命名空间中创建**Application**对象来创建它，这就是我们要做的。在您的集群中创建以下对象（**chapter14/yaml/argocd-python-hello.yaml**）：

apiVersion：argoproj.io/v1alpha1

种类：应用

元数据：

名称：python-hello

命名空间：argocd

规范：

目的地：

命名空间：python-hello

服务器：https://kubernetes.default.svc

项目：默认

来源：

目录:

jsonnet: {}

recurse: true

路径：src

repoURL：git@gitlab-gitlab-shell.gitlab.svc.cluster.local:root/hello-python-operations.git

targetRevision: HEAD

同步策略：

automated: {}

这是尽可能基本的配置。我们正在使用简单的清单。ArgoCD 也可以使用 jsonet 和 Helm。创建此应用程序后，查看**python-hello**命名空间中的 Pod。你应该有一个正在运行的！对代码进行更新将导致命名空间的更新。

现在我们有一个可以通过提交自动部署的代码库。我们花了几十页，运行了几十个命令，并创建了 20 多个对象才达到这一点。与其手动创建这些对象，最好是自动化这个过程。现在我们已经有了需要创建的对象，我们可以自动化入职流程。在下一节中，我们将手动构建 GitLab、Tekton 和 ArgoCD 之间的链接，以符合我们的业务流程。

# 使用 OpenUnison 自动化项目入职

在本章的前面，我们部署了 OpenUnison 自动化门户。该门户允许用户请求创建新的命名空间，并允许开发人员通过自助服务界面请求访问这些命名空间。该门户内置的工作流程非常基本，但会创建命名空间和适当的**RoleBinding**对象。我们想要做的是构建一个集成我们平台并创建本章前面手动创建的所有对象的工作流程。我们的目标是能够在不运行**kubectl**命令（或至少最小化使用）的情况下将新应用程序部署到我们的环境中。这将需要仔细的规划。以下是我们的开发人员工作流程的运行方式：

![图 14.6 - 平台开发人员工作流程](img/Fig_14.6_B15514.jpg)

图 14.6 - 平台开发人员工作流程

让我们快速浏览一下前面图中看到的工作流程：

1.  应用程序所有者将请求创建一个应用程序。

1.  基础设施管理员批准创建。

1.  在这一点上，OpenUnison 将部署我们手动创建的对象。我们将很快详细介绍这些对象。

1.  创建后，开发人员可以请求访问该应用程序

1.  应用程序所有者批准对应用程序的访问。

1.  一旦获得批准，开发人员将分叉应用程序源代码库并进行他们的工作。他们可以在他们的开发工作区启动应用程序。他们还可以分叉构建项目以创建流水线，并分叉开发环境操作项目以为应用程序创建清单。一旦工作完成并在本地测试通过，开发人员将代码推送到他们自己的分叉中，然后请求合并请求。

1.  应用程序所有者将批准请求并从 GitLab 合并代码。

代码合并后，ArgoCD 将同步构建和操作项目。应用程序项目中的 webhook 将启动一个 Tekton 流水线，该流水线将构建我们的容器并使用最新容器的标记更新开发操作项目。ArgoCD 将同步更新的清单到我们应用程序的开发命名空间。一旦测试完成，应用程序所有者将从开发操作工作区提交合并请求到生产操作工作区，触发 ArgoCD 进入生产环境。

在这个流程中，没有一步叫做“运维人员使用**kubectl**创建命名空间”。这是一个简单的流程，并不能完全避免你的运维人员使用**kubectl**，但这应该是一个很好的起点。所有这些自动化都需要创建一系列广泛的对象：

![图 14.7 - 应用入职对象映射](img/Fig_14.7_B15514.jpg)

图 14.7 - 应用入职对象映射

在 GitLab 中，我们为我们的应用程序代码、运维和构建流水线创建一个项目。我们还将运维项目作为开发运维项目进行分支。对于每个项目，我们生成部署密钥并注册 webhook。我们还创建了与本章前面定义的角色相匹配的组。

对于 Kubernetes，我们为开发和生产环境创建命名空间。我们还为 Tekton 流水线创建了一个命名空间。我们根据需要向**Secrets**添加密钥。在构建命名空间中，我们创建了支持触发自动构建的 webhook 的所有脚手架。这样，我们的开发人员只需要担心创建他们的流水线对象。

在我们的最后一个应用程序 ArgoCD 中，我们将创建一个**AppProject**，其中包含我们的构建和两个运维命名空间。我们还将添加在创建 GitLab 项目时生成的 SSH 密钥。每个项目还在我们的**AppProject**中得到一个**Application**对象，指示 ArgoCD 如何从 GitLab 同步。最后，我们向 ArgoCD 添加 RBAC 规则，以便我们的开发人员可以查看他们的应用程序同步状态，但所有者和运维人员可以进行更新和更改。

你不需要自己构建这个！**chapter14/openunison**是实现这一流程的 OpenUnison 的源代码。如果你想看到我们创建的每个对象，请参考**chapter14/openunison/src/main/webapp/WEB-INF/workflows/30-NewK8sNamespace.xml**。这个工作流做了我们刚才描述的一切。我们还包括了**chapter14/python-hello**作为我们的示例应用程序，**chapter14/python-hello-operations**作为我们的清单，以及**chapter14/python-hello-build**作为我们的流水线。你需要调整这三个文件夹中的一些对象以匹配你的环境，主要是更新主机名。

设计好了我们的开发人员工作流程，并准备好示例项目，接下来我们将更新 OpenUnison、GitLab 和 ArgoCD，让所有这些自动化工作起来！

## 集成 GitLab

当我们首次部署 Helm 图表时，我们为 SSO 配置了 GitLab。我们部署的**gitlab-oidc** **秘密**包含 GitLab 访问 OpenUnison 的所有信息。尽管我们仍需要配置 OpenUnison。我们可以将 SSO 配置硬编码到我们的 OpenUnison 源代码中，也可以动态添加它作为自定义资源。在这种情况下，我们将通过自定义资源添加 SSO 连接：

1.  编辑**chapter14/yaml/gitlab-trust.yaml**，将**192-168-2-140**替换为集群正在运行的服务器 IP。我的集群在**192.168.2.114**上，所以我将其替换为**192-168-2-114**。将**chapter14/yaml/gitlab-trust.yaml**添加到您的集群。此文件告诉 OpenUnison 与 GitLab 建立 SSO 信任。

1.  编辑**chapter14/yaml/gitlab-url.yaml**，将**192-168-2-140**替换为集群正在运行的服务器 IP。我的集群在**192.168.2.114**上，所以我将其替换为**192-168-2-114**。将**chapter14/yaml/gitlab-url.yaml**添加到您的集群。此文件告诉 OpenUnison 为 GitLab 门户添加徽标。

1.  以 root 身份登录 GitLab。转到用户的个人资料区域，然后单击**访问令牌**。对于**名称**，使用**openunison**。将**过期**留空并检查 API 范围。单击**创建个人访问令牌**。将令牌复制并粘贴到记事本或其他地方。一旦离开此屏幕，就无法再检索此令牌。

1.  编辑**openunison**命名空间中的**orchestra-secrets-source**秘密。添加两个键：

apiVersion：v1

数据：

K8S_DB_SECRET：aW0gYSBzZWNyZXQ=

OU_JDBC_PASSWORD：c3RhcnR0MTIz

SMTP_PASSWORD：""

unisonKeystorePassword：aW0gYSBzZWNyZXQ=

gitlab：c2VjcmV0 GITLAB_TOKEN：S7CCuqHfpw3a6GmAqEYg

种类：秘密

记得对值进行 Base64 编码。**gitlab**键与我们的**oidc-provider**秘密中的秘密匹配。**GITLAB_TOKEN**将由 OpenUnison 用于与 GitLab 交互，以提供我们在入职工作流程中定义的项目和组。配置了 GitLab，接下来是 ArgoCD。

## 集成 ArgoCD

ArgoCD 内置支持 OpenID Connect。尽管在部署中没有为我们配置：

1.  编辑**argocd-cm** **ConfigMap**在**argocd**命名空间中，添加**url**和**oidc.config**键，如下所示的 cde 块。确保更新**192-168-2-140**以匹配您集群的 IP 地址。我的是**192.168.2.114**，所以我将使用**192-168-2-114**：

apiVersion：v1

数据：

url：https://argocd.apps.192-168-2-140.nip.io

oidc.config：|-

name: OpenUnison

issuer: https://k8sou.apps.192-168-2-140.nip.io/auth/idp/k8sIdp

clientID: argocd

requestedScopes: ["openid", "profile", "email", "groups"]

重要提示

我们没有在 ArgoCD 中指定客户端密钥，因为它既有 CLI 又有 Web 组件。就像 API 服务器一样，在这种情况下担心需要驻留在每台工作站上并为用户所知的客户端密钥是没有意义的。在这种情况下，它不会增加任何安全性，所以我们将跳过它。

1.  编辑**chapter14/yaml/argocd-trust.yaml**，将**192-168-2-140**替换为集群正在运行的服务器 IP。我的集群在**192.168.2.114**上，所以我将其替换为**192-168-2-114**。将**chapter14/yaml/argocd-trust.yaml**添加到您的集群。此文件告诉 OpenUnison 与 ArgoCD 建立 SSO 信任。

1.  编辑**chapter14/yaml/argocd-url.yaml**，将**192-168-2-140**替换为集群正在运行的服务器 IP。我的集群在**192.168.2.114**上，所以我将其替换为**192-168-2-114**。将**chapter14/yaml/argocd-url.yaml**添加到您的集群。此文件告诉 OpenUnison 为 ArgoCD 门户添加徽章。

1.  虽然大部分 ArgoCD 是由 Kubernetes 自定义资源控制的，但也有一些特定于 ArgoCD 的 API。要使用这些 API，我们需要创建一个服务账户。我们需要创建此账户并为其生成一个密钥：

**$ kubectl patch configmap argocd-cm -n argocd -p '{"data":{"accounts.openunison":"apiKey","accounts.openunison.enabled":"true"}}'**

**$ argocd account generate-token --account openunison**

1.  获取**generate-token**命令的输出，并将其添加为**orchestra-secrets-source** **Secret**中的**ARGOCD_TOKEN**键，位于**openunison**命名空间中。不要忘记对其进行 Base64 编码。

1.  最后，我们想创建 ArgoCD RBAC 规则，以便我们可以控制谁可以访问 Web UI 和 CLI。编辑**argocd-rbac-cm** **ConfigMap**并添加以下键。第一个键将允许我们的系统管理员和我们的 API 密钥在 ArgoCD 中执行任何操作。第二个键将所有未在**policy.csv**中映射的用户映射到一个不存在的角色中，以便他们无法访问任何内容：

数据：

policy.csv: |-

g，k8s-cluster-administrators，role:admin

g，openunison，role:admin

policy.default: role:none

集成了 ArgoCD，实现自动化的最后一步是更新我们的 OpenUnison 自定义资源！

## 更新 OpenUnison

OpenUnison 已经部署。启动自动化门户与我们构建的开发者工作流程的最后一步是更新 orchestra OpenUnison 自定义资源。按照以下代码块中的方式更新图像。添加 non_secret_data，将 hosts 替换为与您的集群 IP 匹配。最后，将我们创建的新密钥添加到操作员需要导入的密钥列表中：

.

.

.

   图像：docker.io/tremolosecurity/openunison-k8s-definitive-guide:latest

.

.

.

non_secret_data:

.

.

.

- 名称：GITLAB_URL

   值：https://gitlab.apps.192-168-2-140.nip.io

   - 名称：GITLAB_SSH_HOST

   值：gitlab-gitlab-shell.gitlab.svc.cluster.local

   - 名称：GITLAB_WEBHOOK_SUFFIX

   值：gitlab.192-168-2-140.nip.io

   - 名称：ARGOCD_URL

   值：https://argocd.apps.192-168-2-140.nip.io

   - 名称：GITLAB_WRITE_SSH_HOST

   值：gitlab-write-shell.gitlab.svc.cluster.local

.

.

.

secret_data:

- K8S_DB_SECRET

- unisonKeystorePassword

- SMTP_PASSWORD

- OU_JDBC_PASSWORD

- GITLAB_TOKEN

   - ARGOCD_TOKEN

只需几分钟，自动化门户就会运行。当您登录时，您将看到 GitLab 和 ArgoCD 的徽章。您还可以单击“新应用程序”开始根据我们的工作流程部署应用程序！您可以将其用作设计自己的自动化平台的起点，或者将其用作创建所需对象以集成平台工具的地图。

# 摘要

进入本章之前，我们并没有花太多时间部署应用程序。我们希望通过简要介绍应用程序部署和自动化来结束这些内容。我们了解了流水线，它们是如何构建的，以及它们在 Kubernetes 集群上运行的方式。我们通过部署 GitLab 进行源代码控制，构建了一个 Tekton 流水线以适用 GitOps 模型，并使用 ArgoCD 使 GitOps 模型成为现实。最后，我们使用 OpenUnison 自动化了整个流程。

使用本章中的信息应该为您提供了构建自己平台的方向。使用本章中的实际示例将帮助您将组织中的要求映射到自动化基础设施所需的技术。我们在本章中构建的平台远非完整。它应该为您规划自己的平台提供了一个地图，以满足您的需求。

最后，谢谢。感谢您加入我们一起构建 Kubernetes 集群的冒险。我们希望您阅读本书和构建示例时和我们创作时一样开心！

# 问题

1.  正确还是错误：必须实施流水线才能使 Kubernetes 工作。

A. 正确

B. 错误

1.  流水线的最低步骤是什么？

A. 构建、扫描、测试和部署

B. 构建和部署

C. 扫描、测试、部署和构建

D. 以上都不是

1.  什么是 GitOps？

A. 在 Kubernetes 上运行 GitLab

B. 使用 Git 作为操作配置的权威来源

C. 一个愚蠢的营销术语

D. 一家新创企业的产品

1.  编写流水线的标准是什么？

A. 所有流水线应该用 YAML 编写。

B. 没有标准；每个项目和供应商都有自己的实现。

C. JSON 结合 Go。

D. Rust.

1.  在 GitOps 模型中如何部署一个新的容器实例？

A. 使用**kubectl**在命名空间中更新**Deployment**或**StatefulSet**。

B. 在 Git 中更新**Deployment**或**StatefulSet**清单，让 GitOps 控制器更新 Kubernetes 中的对象。

C. 提交一个需要运维人员处理的工单。

D. 以上都不是。

1.  正确还是错误：GitOps 中的所有对象都需要存储在您的 Git 存储库中。

A. 正确

B. 错误

1.  正确还是错误：你的方式是自动化流程的正确方式。

A. 正确

B. 错误

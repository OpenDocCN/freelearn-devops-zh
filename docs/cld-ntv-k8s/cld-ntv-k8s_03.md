# *第二章*：设置您的 Kubernetes 集群

本章包含了创建 Kubernetes 集群的一些可能性的审查，这将使我们能够学习本书中其余概念所需的知识。我们将从 minikube 开始，这是一个创建简单本地集群的工具，然后涉及一些其他更高级（且适用于生产）的工具，并审查来自公共云提供商的主要托管 Kubernetes 服务，最后介绍从头开始创建集群的策略。

在本章中，我们将涵盖以下主题：

+   创建您的第一个集群的选项

+   minikube – 一个简单的开始方式

+   托管服务 – EKS、GKE、AKS 等

+   Kubeadm – 简单的一致性

+   Kops – 基础设施引导

+   Kubespray – 基于 Ansible 的集群创建

+   完全从头开始创建集群

# 技术要求

为了在本章中运行命令，您需要安装 kubectl 工具。安装说明可在*第一章*，*与 Kubernetes 通信*中找到。

如果您确实要使用本章中的任何方法创建集群，您需要查看相关项目文档中每种方法的具体技术要求。对于 minikube，大多数运行 Linux、macOS 或 Windows 的计算机都可以工作。对于大型集群，请查阅您计划使用的工具的具体文档。

本章中使用的代码可以在书籍的 GitHub 存储库中找到，链接如下：

[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter2`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes/tree/master/Chapter2)

# 创建集群的选项

有许多方法可以创建 Kubernetes 集群，从简单的本地工具到完全从头开始创建集群。

如果您刚开始学习 Kubernetes，可能希望使用 minikube 等工具快速启动一个简单的本地集群。

如果您希望为应用程序构建生产集群，您有几个选项：

+   您可以使用 Kops、Kubespray 或 Kubeadm 等工具以编程方式创建集群。

+   您可以使用托管的 Kubernetes 服务。

+   您可以在虚拟机或物理硬件上完全从头开始创建集群。

除非您在集群配置方面有极其特定的需求（即使是这样），通常不建议完全不使用引导工具从头开始创建您的集群。

对于大多数用例，决策将在使用云提供商上的托管 Kubernetes 服务和使用引导工具之间进行。

在空气隔离系统中，使用引导工具是唯一的选择，但对于特定的用例，有些引导工具比其他引导工具更好。特别是，Kops 旨在使在云提供商（如 AWS）上创建和管理集群变得更容易。

重要提示

本节未包括讨论替代的第三方托管服务或集群创建和管理工具，如 Rancher 或 OpenShift。在选择在生产环境中运行集群时，重要的是要考虑包括当前基础设施、业务需求等在内的各种因素。为了简化问题，在本书中，我们将专注于生产集群，假设没有其他基础设施或超特定的业务需求——可以说是一个“白板”。

# minikube-开始的简单方法

minikube 是开始使用简单本地集群的最简单方法。这个集群不会设置为高可用性，并且不针对生产使用，但这是一个在几分钟内开始在 Kubernetes 上运行工作负载的好方法。

## 安装 minikube

minikube 可以安装在 Windows、macOS 和 Linux 上。接下来是三个平台的安装说明，您也可以通过导航到[`minikube.sigs.k8s.io/docs/start`](https://minikube.sigs.k8s.io/docs/start)找到。

### 在 Windows 上安装

在 Windows 上最简单的安装方法是从[`storage.googleapis.com/minikube/releases/latest/minikube-installer.exe`](https://storage.googleapis.com/minikube/releases/latest/minikube-installer.exe)下载并运行 minikube 安装程序。

### 在 macOS 上安装

使用以下命令下载和安装二进制文件。您也可以在代码存储库中找到它：

Minikube-install-mac.sh

```
     curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64 \
&& sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

### 在 Linux 上安装

使用以下命令下载和安装二进制文件：

Minikube-install-linux.sh

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
&& sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

## 在 minikube 上创建一个集群

使用 minikube 创建一个集群，只需运行`minikube start`，这将使用默认的 VirtualBox VM 驱动程序创建一个简单的本地集群。minikube 还有一些额外的配置选项，可以在文档站点上查看。

运行`minikube` `start`命令将自动配置您的`kubeconfig`文件，这样您就可以在新创建的集群上运行`kubectl`命令，而无需进行进一步的配置。

# 托管 Kubernetes 服务

提供托管 Kubernetes 服务的云提供商数量不断增加。然而，对于本书的目的，我们将专注于主要的公共云及其特定的 Kubernetes 服务。这包括以下内容：

+   亚马逊网络服务（AWS） - 弹性 Kubernetes 服务（EKS）

+   谷歌云 - 谷歌 Kubernetes 引擎（GKE）

+   微软 Azure - Azure Kubernetes 服务（AKS）

重要提示

托管 Kubernetes 服务的数量和实施方式总是在变化。AWS、谷歌云和 Azure 被选为本书的这一部分，因为它们很可能会继续以相同的方式运行。无论您使用哪种托管服务，请确保查看服务提供的官方文档，以确保集群创建过程与本书中所呈现的相同。

## 托管 Kubernetes 服务的好处

一般来说，主要的托管 Kubernetes 服务提供了一些好处。首先，我们正在审查的这三个托管服务提供了完全托管的 Kubernetes 控制平面。

这意味着当您使用这些托管 Kubernetes 服务之一时，您不需要担心主节点。它们被抽象化了，可能根本不存在。这三个托管集群都允许您在创建集群时选择工作节点的数量。

托管集群的另一个好处是从一个 Kubernetes 版本无缝升级到另一个版本。一般来说，一旦验证了托管服务的新版本 Kubernetes（不一定是最新版本），您应该能够使用一个按钮或一个相当简单的过程进行升级。

## 托管 Kubernetes 服务的缺点

尽管托管 Kubernetes 集群在许多方面可以简化操作，但也存在一些缺点。

对于许多可用的托管 Kubernetes 服务，托管集群的最低成本远远超过手动创建或使用诸如 Kops 之类的工具创建的最小集群的成本。对于生产用例，这通常不是一个问题，因为生产集群应该包含最少数量的节点，但对于开发环境或测试集群，根据预算，额外的成本可能不值得操作的便利。

此外，虽然抽象化主节点使操作更容易，但它也阻止了对已定义主节点的集群可能可用的精细调整或高级主节点功能。

# AWS - 弹性 Kubernetes 服务

AWS 的托管 Kubernetes 服务称为 EKS，或弹性 Kubernetes 服务。有几种不同的方式可以开始使用 EKS，但我们将介绍最简单的方式。

## 入门

要创建一个 EKS 集群，您必须配置适当的**虚拟私有云（VPC）**和**身份和访问管理（IAM）**角色设置 - 在这一点上，您可以通过控制台创建一个集群。这些设置可以通过控制台手动创建，也可以通过基础设施配置工具如 CloudFormation 和 Terraform 创建。有关通过控制台创建集群的完整说明，请参阅[`docs.aws.amazon.com/en_pv/eks/latest/userguide/getting-started-console.html`](https://docs.aws.amazon.com/en_pv/eks/latest/userguide/getting-started-console.html)。

假设您是从头开始创建集群和 VPC，您可以使用一个名为`eksctl`的工具来配置您的集群。

要安装`eksctl`，您可以在[`docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html`](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)找到 macOS、Linux 和 Windows 的安装说明。

一旦安装了`eksctl`，创建一个集群就像使用`eksctl create cluster`命令一样简单：

Eks-create-cluster.sh

```
eksctl create cluster \
--name prod \
--version 1.17 \
--nodegroup-name standard-workers \
--node-type t2.small \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--node-ami auto
```

这将创建一个由三个`t2.small`实例组成的集群，这些实例被设置为一个具有一个节点最小和四个节点最大的自动缩放组。使用的 Kubernetes 版本将是`1.17`。重要的是，`eksctl`从一个默认区域开始，并根据选择的节点数量，在该区域的多个可用区中分布它们。

`eksctl`还将自动更新您的`kubeconfig`文件，因此在集群创建过程完成后，您应该能够立即运行`kubectl`命令。

使用以下代码测试配置：

```
kubectl get nodes
```

您应该看到您的节点及其关联的 IP 列表。您的集群已准备就绪！接下来，让我们看看 Google 的 GKE 设置过程。

# Google Cloud – Google Kubernetes Engine

GKE 是 Google Cloud 的托管 Kubernetes 服务。使用 gcloud 命令行工具，可以很容易地快速启动 GKE 集群。

## 入门

要使用 gcloud 在 GKE 上创建集群，可以使用 Google Cloud 的 Cloud Shell 服务，也可以在本地运行命令。如果要在本地运行命令，必须通过 Google Cloud SDK 安装 gcloud CLI。有关安装说明，请参阅[`cloud.google.com/sdk/docs/quickstarts`](https://cloud.google.com/sdk/docs/quickstarts)。

安装了 gcloud 后，您需要确保已在 Google Cloud 帐户中激活了 GKE API。

要轻松实现这一点，请转到[`console.cloud.google.com/apis/library`](https://console.cloud.google.com/apis/library)，然后在搜索栏中搜索`kubernetes`。单击**Kubernetes Engine API**，然后单击**启用**。

现在 API 已激活，请使用以下命令在 Google Cloud 中设置您的项目和计算区域：

```
gcloud config set project proj_id
gcloud config set compute/zone compute_zone
```

在命令中，`proj_id`对应于您想要在 Google Cloud 中创建集群的项目 ID，`compute_zone`对应于您在 Google Cloud 中期望的计算区域。

实际上，GKE 上有三种类型的集群，每种类型具有不同（增加）的可靠性和容错能力：

+   单区集群

+   多区集群

+   区域集群

GKE 中的**单区**集群意味着具有单个控制平面副本和一个或多个在同一 Google Cloud 区域运行的工作节点的集群。如果区域发生故障，控制平面和工作节点（因此工作负载）都将宕机。

GKE 中的**多区**集群意味着具有单个控制平面副本和两个或多个在不同的 Google Cloud 区域运行的工作节点的集群。这意味着如果单个区域（甚至包含控制平面的区域）发生故障，集群中运行的工作负载仍将持续存在，但是直到控制平面区域恢复之前，Kubernetes API 将不可用。

最后，在 GKE 中，**区域集群** 意味着具有多区域控制平面和多区域工作节点的集群。如果任何区域出现故障，控制平面和工作节点上的工作负载将持续存在。这是最昂贵和可靠的选项。

现在，要实际创建您的集群，您可以运行以下命令以使用默认设置创建名为 `dev` 的集群：

```
gcloud container clusters create dev \
    --zone [compute_zone]
```

此命令将在您选择的计算区域创建一个单区域集群。

为了创建一个多区域集群，您可以运行以下命令：

```
gcloud container clusters create dev \
    --zone [compute_zone_1]
    --node-locations [compute_zone_1],[compute_zone_2],[etc]
```

在这里，`compute_zone_1` 和 `compute_zone_2` 是不同的 Google Cloud 区域。此外，可以通过 `node-locations` 标志添加更多区域。

最后，要创建一个区域集群，您可以运行以下命令：

```
gcloud container clusters create dev \
    --region [region] \
    --node-locations [compute_zone_1],[compute_zone_2],[etc]
```

在这种情况下，`node-locations` 标志实际上是可选的。如果省略，集群将在该区域内的所有区域中创建工作节点。如果您想更改此默认行为，可以使用 `node-locations` 标志进行覆盖。

现在您已经运行了一个集群，需要配置您的 `kubeconfig` 文件以与集群通信。为此，只需将集群名称传递给以下命令：

```
gcloud container clusters get-credentials [cluster_name]
```

最后，使用以下命令测试配置：

```
kubectl get nodes
```

与 EKS 一样，您应该看到所有已配置节点的列表。成功！最后，让我们来看看 Azure 的托管服务。

# Microsoft Azure – Azure Kubernetes 服务

Microsoft Azure 的托管 Kubernetes 服务称为 AKS。可以通过 Azure CLI 在 AKS 上创建集群。

## 入门

要在 AKS 上创建集群，可以使用 Azure CLI 工具，并运行以下命令以创建服务主体（集群将使用该服务主体访问 Azure 资源的角色）：

```
az ad sp create-for-rbac --skip-assignment --name myClusterPrincipal
```

此命令的结果将是一个包含有关服务主体信息的 JSON 对象，我们将在下一步中使用。此 JSON 对象如下所示：

```
{
  "appId": "559513bd-0d99-4c1a-87cd-851a26afgf88",
  "displayName": "myClusterPrincipal",
  "name": "http://myClusterPrincipal",
  "password": "e763725a-5eee-892o-a466-dc88d980f415",
  "tenant": "72f988bf-90jj-41af-91ab-2d7cd011db48"
}
```

现在，您可以使用上一个 JSON 命令中的值来实际创建您的 AKS 集群：

Aks-create-cluster.sh

```
az aks create \
    --resource-group devResourceGroup \
    --name myCluster \
    --node-count 2 \
    --service-principal <appId> \
    --client-secret <password> \
    --generate-ssh-keys
```

此命令假定存在名为 `devResourceGroup` 的资源组和名为 `devCluster` 的集群。对于 `appId` 和 `password`，请使用服务主体创建步骤中的值。

最后，要在您的计算机上生成正确的 `kubectl` 配置，您可以运行以下命令：

```
az aks get-credentials --resource-group devResourceGroup --name myCluster
```

到这一步，您应该能够正确运行 `kubectl` 命令。使用 `kubectl get nodes` 命令测试配置。

# 程序化集群创建工具

有几种可用的工具可以在各种非托管环境中引导 Kubernetes 集群。我们将重点关注三种最流行的工具：Kubeadm、Kops 和 Kubespray。每种工具都针对不同的用例，并且通常通过不同的方法工作。

## Kubeadm

Kubeadm 是由 Kubernetes 社区创建的工具，旨在简化已经配置好的基础架构上的集群创建。与 Kops 不同，Kubeadm 无法在云服务上提供基础架构。它只是创建一个符合 Kubernetes 一致性测试的最佳实践集群。Kubeadm 对基础架构是不可知的-它应该可以在任何可以运行 Linux VM 的地方工作。

## Kops

Kops 是一种流行的集群配置工具。它为您的集群提供基础架构，安装所有集群组件，并验证您的集群功能。它还可以用于执行各种集群操作，如升级、节点旋转等。Kops 目前支持 AWS，在撰写本书时，还支持 Google Compute Engine 和 OpenStack 的 beta 版本，以及 VMware vSphere 和 DigitalOcean 的 alpha 版本。

## Kubespray

Kubespray 与 Kops 和 Kubeadm 都不同。与 Kops 不同，Kubespray 并不固有地提供集群资源。相反，Kubespray 允许您在 Ansible 和 Vagrant 之间进行选择，以执行配置、编排和节点设置。

与 Kubeadm 相比，Kubespray 集成了更少的集群创建和生命周期流程。Kubespray 的新版本允许您在节点设置后专门使用 Kubeadm 进行集群创建。

重要说明

由于使用 Kubespray 创建集群需要一些特定于 Ansible 的领域知识，我们将不在本书中讨论这个问题-但可以在[`github.com/kubernetes-sigs/kubespray/blob/master/docs/getting-started.md`](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/getting-started.md)找到有关 Kubespray 的所有信息的指南。

# 使用 Kubeadm 创建集群

要使用 Kubeadm 创建集群，您需要提前配置好节点。与任何其他 Kubernetes 集群一样，我们需要运行 Linux 的 VM 或裸金属服务器。

为了本书的目的，我们将展示如何使用单个主节点引导 Kubeadm 集群。对于高可用设置，您需要在其他主节点上运行额外的加入命令，您可以在[`kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/`](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)找到。

## 安装 Kubeadm

首先，您需要在所有节点上安装 Kubeadm。每个支持的操作系统的安装说明可以在[`kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm`](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm)找到。

对于每个节点，还要确保所有必需的端口都是开放的，并且已安装您打算使用的容器运行时。

## 启动主节点

要快速启动使用 Kubeadm 的主节点，您只需要运行一个命令：

```
kubeadm init
```

此初始化命令可以接受几个可选参数 - 根据您的首选集群设置、网络等，您可能需要使用它们。

在`init`命令的输出中，您将看到一个`kubeadm join`命令。确保保存此命令。

## 启动工作节点

为了引导工作节点，您需要运行保存的`join`命令。命令的形式如下：

```
kubeadm join --token [TOKEN] [IP ON MASTER]:[PORT ON MASTER] --discovery-token-ca-cert-hash sha256:[HASH VALUE]
```

此命令中的令牌是引导令牌。它用于验证节点之间的身份，并将新节点加入集群。拥有此令牌的访问权限即可加入新节点到集群中，因此请谨慎对待。

## 设置 kubectl

使用 Kubeadm，kubectl 已经在主节点上正确设置。但是，要从任何其他机器或集群外部使用 kubectl，您可以将主节点上的配置复制到本地机器：

```
scp root@[IP OF MASTER]:/etc/kubernetes/admin.conf .
kubectl --kubeconfig ./admin.conf get nodes 
```

这个`kubeconfig`将是集群管理员配置 - 为了指定其他用户（和权限），您需要添加新的服务账户并为他们生成`kubeconfig`文件。

# 使用 Kops 创建集群

由于 Kops 将为您提供基础设施，因此无需预先创建任何节点。您只需要安装 Kops，确保您的云平台凭据有效，并立即创建您的集群。Kops 可以安装在 Linux、macOS 和 Windows 上。

在本教程中，我们将介绍如何在 AWS 上创建一个集群，但您可以在 Kops 文档中找到其他支持的 Kops 平台的说明，网址为[`github.com/kubernetes/kops/tree/master/docs`](https://github.com/kubernetes/kops/tree/master/docs)。

## 在 macOS 上安装

在 OS X 上，安装 Kops 的最简单方法是使用 Homebrew：

```
brew update && brew install kops
```

或者，您可以从 Kops GitHub 页面上获取最新的稳定 Kops 二进制文件，网址为[`github.com/kubernetes/kops/releases/tag/1.12.3`](https://github.com/kubernetes/kops/releases/tag/1.12.3)。

## 在 Linux 上安装

在 Linux 上，您可以通过以下命令安装 Kops：

Kops-linux-install.sh

```
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
```

## 在 Windows 上安装

要在 Windows 上安装 Kops，您需要从[`github.com/kubernetes/kops/releases/latest`](https://github.com/kubernetes/kops/releases/latest)下载最新的 Windows 版本，将其重命名为`kops.exe`，并将其添加到您的`path`变量中。

## 设置 Kops 的凭据

为了让 Kops 工作，您需要在您的机器上具有一些必需的 IAM 权限的 AWS 凭据。为了安全地执行此操作，您需要为 Kops 专门创建一个 IAM 用户。

首先，为`kops`用户创建一个 IAM 组：

```
aws iam create-group --group-name kops_users
```

然后，为`kops_users`组附加所需的角色。为了正常运行，Kops 将需要`AmazonEC2FullAccess`，`AmazonRoute53FullAccess`，`AmazonS3FullAccess`，`IAMFullAccess`和`AmazonVPCFullAccess`。我们可以通过运行以下命令来实现这一点：

提供-aws-policies-to-kops.sh

```
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
```

最后，创建`kops`用户，将其添加到`kops_users`组，并创建程序访问密钥，然后保存：

```
aws iam create-user --user-name kops
aws iam add-user-to-group --user-name kops --group-name kops_users
aws iam create-access-key --user-name kops
```

为了让 Kops 访问您的新 IAM 凭据，您可以使用以下命令配置 AWS CLI，使用前一个命令（`create-access-key`）中的访问密钥和秘钥：

```
aws configure
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```

## 设置状态存储

凭据设置好后，我们可以开始创建我们的集群。在这种情况下，我们将构建一个简单的基于 gossip 的集群，因此我们不需要处理 DNS。要查看可能的 DNS 设置，您可以查看 Kops 文档（[`github.com/kubernetes/kops/tree/master/docs`](https://github.com/kubernetes/kops/tree/master/docs)）。

首先，我们需要一个位置来存储我们的集群规范。由于我们在 AWS 上，S3 非常适合这个任务。

像往常一样，使用 S3 时，存储桶名称需要是唯一的。您可以使用 AWS SDK 轻松创建一个存储桶（确保将`my-domain-dev-state-store`替换为您想要的 S3 存储桶名称）：

```
aws s3api create-bucket \
    --bucket my-domain-dev-state-store \
    --region us-east-1
```

启用存储桶加密和版本控制是最佳实践：

```
aws s3api put-bucket-versioning --bucket prefix-example-com-state-store  --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket prefix-example-com-state-store --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

最后，要设置 Kops 的变量，请使用以下命令：

```
export NAME=devcluster.k8s.local
export KOPS_STATE_STORE=s3://my-domain-dev-cluster-state-store
```

重要提示

Kops 支持多种状态存储位置，如 AWS S3，Google Cloud Storage，Kubernetes，DigitalOcean，OpenStack Swift，阿里云和 memfs。但是，您可以将 Kops 状态仅保存到本地文件并使用该文件。云端状态存储的好处是多个基础架构开发人员可以访问并使用版本控制进行更新。

## 创建集群

使用 Kops，我们可以部署任何规模的集群。在本指南的目的是，我们将通过在三个可用区域跨越工作节点和主节点来部署一个生产就绪的集群。我们将使用 US-East-1 地区，主节点和工作节点都将是`t2.medium`实例。

要为此集群创建配置，可以运行以下`kops create`命令：

Kops-create-cluster.sh

```
kops create cluster \
    --node-count 3 \
    --zones us-east-1a,us-east-1b,us-east-1c \
    --master-zones us-east-1a,us-east-1b,us-east-1c \
    --node-size t2.medium \
    --master-size t2.medium \
    ${NAME}
```

要查看已创建的配置，请使用以下命令：

```
kops edit cluster ${NAME}
```

最后，要创建我们的集群，请运行以下命令：

```
kops update cluster ${NAME} --yes
```

集群创建过程可能需要一些时间，但一旦完成，您的`kubeconfig`应该已经正确配置，可以使用 kubectl 与您的新集群进行交互。

# 完全从头开始创建集群

完全从头开始创建一个 Kubernetes 集群是一个多步骤的工作，可能需要跨越本书的多个章节。然而，由于我们的目的是尽快让您开始使用 Kubernetes，我们将避免描述整个过程。

如果您有兴趣从头开始创建集群，无论是出于教育目的还是需要精细定制您的集群，都可以参考*Kubernetes The Hard Way*，这是由*Kelsey Hightower*编写的完整集群创建教程。它可以在[`github.com/kelseyhightower/kubernetes-the-hard-way`](https://github.com/kelseyhightower/kubernetes-the-hard-way)找到。

既然我们已经解决了这个问题，我们可以继续概述手动创建集群的过程。

## 配置您的节点

首先，您需要一些基础设施来运行 Kubernetes。通常，虚拟机是一个很好的选择，尽管 Kubernetes 也可以在裸机上运行。如果您在一个不能轻松添加节点的环境中工作（这会消除云的许多扩展优势，但在企业环境中绝对可行），您需要足够的节点来满足应用程序的需求。这在空隔离环境中更有可能成为一个问题。

一些节点将用于主控制平面，而其他节点将仅用作工作节点。没有必要从内存或 CPU 的角度使主节点和工作节点相同 - 甚至可以有一些较弱的和一些更强大的工作节点。这种模式会导致一个非同质的集群，其中某些节点更适合特定的工作负载。

## 为 TLS 创建 Kubernetes 证书颁发机构

为了正常运行，所有主要控制平面组件都需要 TLS 证书。为了创建这些证书，需要创建一个证书颁发机构（CA），它将进一步创建 TLS 证书。

要创建 CA，需要引导公钥基础设施（PKI）。对于这个任务，可以使用任何 PKI 工具，但 Kubernetes 文档中使用的是 cfssl。

一旦为所有组件创建了 PKI、CA 和 TLS 证书，下一步是为控制平面和工作节点组件创建配置文件。

## 创建配置文件

需要为 kubelet、kube-proxy、kube-controller-manager 和 kube-scheduler 组件创建配置文件。它们将使用这些配置文件中的证书与 kube-apiserver 进行身份验证。

## 创建 etcd 集群并配置加密

通过一个带有数据加密密钥的 YAML 文件来处理数据加密配置。此时，需要启动 etcd 集群。

为此，在每个节点上创建带有 etcd 进程配置的 systemd 文件。然后在每个节点上使用 systemctl 启动 etcd 服务器。

这是一个 etcd 的 systemd 文件示例。其他控制平面组件的 systemd 文件将类似于这个：

示例-systemd-control-plane

```
[Unit]
Description=etcd
Documentation=https://github.com/coreos
[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
```

该服务文件为我们的 etcd 组件提供了运行时定义，它将在每个主节点上启动。要在我们的节点上实际启动 etcd，我们运行以下命令：

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

这使得`etcd`服务能够在节点重新启动时自动重新启动。

## 引导控制平面组件

在主节点上引导控制平面组件的过程类似于创建`etcd`集群所使用的过程。为每个组件创建`systemd`文件 - API 服务器、控制器管理器和调度器 - 然后使用`systemctl`命令启动每个组件。

先前创建的配置文件和证书也需要包含在每个主节点上。

让我们来看看我们的`kube-apiserver`组件的服务文件定义，按照以下各节进行拆分。`Unit`部分只是我们`systemd`文件的一个快速描述：

```
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
```

Api-server-systemd-example

第二部分是服务的实际启动命令，以及要传递给服务的任何变量：

```
[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.10.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
```

最后，`Install`部分允许我们指定一个`WantedBy`目标：

```
Restart=on-failure
RestartSec=5
 [Install]
WantedBy=multi-user.target
```

`kube-scheduler`和`kube-controller-manager`的服务文件将与`kube-apiserver`的定义非常相似，一旦我们准备在节点上启动组件，这个过程就很容易：

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

与`etcd`类似，我们希望确保服务在节点关闭时重新启动。

## 引导工作节点

工作节点上也是类似的情况。需要创建并使用`systemctl`运行`kubelet`、容器运行时、`cni`和`kube-proxy`的服务规范。`kubelet`配置将指定上述 TLS 证书，以便它可以通过 API 服务器与控制平面通信。

让我们看看我们的`kubelet`服务定义是什么样子的：

Kubelet-systemd-example

```
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service
[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
```

正如你所看到的，这个服务定义引用了`cni`、容器运行时和`kubelet-config`文件。`kubelet-config`文件包含我们工作节点所需的 TLS 信息。

在引导工作节点和主节点之后，集群应该可以通过作为 TLS 设置的一部分创建的管理员`kubeconfig`文件来使用。

# 总结

在本章中，我们回顾了创建 Kubernetes 集群的几种方法。我们研究了使用 minikube 在本地创建最小的集群，设置在 Azure、AWS 和 Google Cloud 上管理的 Kubernetes 服务的集群，使用 Kops 配置工具创建集群，最后，从头开始手动创建集群。

现在我们有了在几种不同环境中创建 Kubernetes 集群的技能，我们可以继续使用 Kubernetes 来运行应用程序。

在下一章中，我们将学习如何在 Kubernetes 上开始运行应用程序。您对 Kubernetes 在架构层面的工作原理的了解应该会让您更容易理解接下来几章中的概念。

# 问题

1.  minikube 有什么作用？

1.  使用托管 Kubernetes 服务有哪些缺点？

1.  Kops 与 Kubeadm 有何不同？主要区别是什么？

1.  Kops 支持哪些平台？

1.  在手动创建集群时，如何指定主要集群组件？它们如何在每个节点上运行？

# 进一步阅读

+   官方 Kubernetes 文档：[`kubernetes.io/docs/home/`](https://kubernetes.io/docs/home/)

+   *Kubernetes The Hard Way*：[`github.com/kelseyhightower/kubernetes-the-hard-way`](https://github.com/kelseyhightower/kubernetes-the-hard-way)

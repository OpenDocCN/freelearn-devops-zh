# 第十一章：10. 保护您的 AKS 集群

“泄露机密会导致失败”是一个描述在 Kubernetes 管理的集群中很容易危及安全性的短语（顺便说一句，*Kubernetes*在希腊语中是*舵手*的意思，就像*船*的舵手）。如果您的集群开放了错误的端口或服务，或者在应用程序定义中使用了明文作为秘密，不良行为者可以利用这种疏忽的安全性做几乎任何他们想做的事情。

在本章中，我们将更深入地探讨 Kubernetes 安全性。您将了解 Kubernetes 中的**基于角色的访问控制（RBAC）**的概念。之后，您将学习有关秘密以及如何使用它们的内容。您将首先在 Kubernetes 中创建秘密，然后创建一个 Key Vault 来更安全地存储秘密。最后，您将简要介绍服务网格概念，并且将给出一个实际示例供您参考。

本章将简要介绍以下主题：

+   基于角色的访问控制

+   设置秘密管理

+   在 Key Vault 中使用存储的秘密

+   Istio 服务网格为您服务

#### 注意

要完成有关 RBAC 的示例，您需要访问具有全局管理员权限的 Azure AD 实例。

让我们从 RBAC 开始这一章。

## 基于角色的访问控制

在生产系统中，您需要允许不同用户对某些资源有不同级别的访问权限；这被称为**基于角色的访问控制**（**RBAC**）。本节将带您了解如何在 AKS 中配置 RBAC，以及如何分配不同权限的不同角色。建立 RBAC 的好处在于，它不仅可以防止意外删除关键资源，还是一项重要的安全功能，限制了对集群的完全访问权限。在启用 RBAC 的集群上，用户将能够观察到他们只能修改他们有权限访问的资源。

到目前为止，使用 Cloud Shell，我们一直在扮演*root*，这使我们可以在集群中做任何事情。对于生产用例来说，root 访问是危险的，应尽可能受到限制。通常公认的最佳实践是使用**最小权限原则**（PoLP）登录任何计算机系统。这可以防止对安全数据的访问和通过删除关键资源而造成意外停机。据统计，22%至 29%（[`blog.storagecraft.com/data-loss-statistics-infographic/`](https://blog.storagecraft.com/data-loss-statistics-infographic/)）的数据丢失归因于人为错误。您不希望成为这一统计数字的一部分。

Kubernetes 开发人员意识到这是一个问题，并添加了 RBAC 以及服务角色的概念来控制对集群的访问。Kubernetes RBAC 有三个重要的概念：

+   **角色**：角色包含一组权限。角色默认没有权限，每个权限都需要明确声明。权限的示例包括*get*、*watch*和*list*。角色还包含这些权限所赋予的资源。资源可以是所有 Pod、部署等，也可以是特定对象（如*pod/mypod*）。

+   **主体**：主体可以是分配了角色的 Azure AD 用户或组。

+   **RoleBinding**：RoleBinding 将一个主体与特定命名空间中的角色或者 ClusterRoleBinding 中的整个集群中的角色进行了关联。

一个重要的概念要理解的是，在与 AKS 进行交互时，有两个 RBAC 层次：Azure RBAC 和 Kubernetes RBAC。Azure RBAC 处理分配给人们在 Azure 中进行更改的角色，比如创建、修改和删除集群。Kubernetes RBAC 处理集群中资源的访问权限。两者都是独立的控制平面，但可以使用源自 Azure AD 的相同用户和组。

![一个架构图，显示了 RBAC 的两个层次：Azure RBAC 和 Kubernetes RBAC。在 Azure 中，Azure AD 用户或组被分配 Azure 角色，以允许访问订阅和资源组中的节点。同样，Azure AD 用户或组被分配 Kubernetes 角色，以访问 Kubernetes 中的 Pod、部署和命名空间。](img/Figure_10.1.jpg)

###### 图 10.1：两个不同的 RBAC 平面，Azure 和 Kubernetes

Kubernetes 中的 RBAC 是一个可选功能。在 AKS 中，默认情况下是创建启用了 RBAC 的集群。但是，默认情况下，集群未集成到 Azure AD 中。这意味着默认情况下，您无法授予 Kubernetes 权限给 Azure AD 用户。在这个例子中，我们将创建一个与 Azure AD 集成的新集群。让我们通过在 Azure AD 中创建一个新用户和一个新组来开始我们对 RBAC 的探索。

### 创建一个集成了 Azure AD 的新集群

在本节中，我们将创建一个与 Azure AD 集成的新集群。这是必需的，这样我们就可以在接下来的步骤中引用 Azure AD 中的用户。所有这些步骤将在 Cloud Shell 中执行。我们还提供了一个名为`cluster-aad.sh`的文件中的步骤。如果您希望执行该脚本，请更改前四行中的变量以反映您的偏好。让我们继续执行这些步骤：

1.  我们将从缩减当前集群到一个节点开始：

```
az aks nodepool scale --cluster-name handsonaks \
  -g rg-handsonaks --name agentpool--node-count 1
```

1.  然后，我们将设置一些在脚本中将使用的变量：

```
EXISTINGAKSNAME="handsonaks"
NEWAKSNAME="handsonaks-aad"
RGNAME="rg-handsonaks"
LOCATION="westus2"
TENANTID=$(az account show --query tenantId -o tsv)
```

1.  现在，我们将从我们的 AKS 集群中获取现有的服务主体。我们将重用此服务主体，以授予新集群访问我们的 Azure 订阅的权限：

```
# Get SP from existing cluster and create new password
RBACSP=$(azaks show -n $EXISTINGAKSNAME -g $RGNAME \
  --query servicePrincipalProfile.clientId -o tsv)
RBACSPPASSWD=$(openssl rand -base64 32)
az ad sp credential reset --name $RBACSP \
  --password $RBACSPPASSWD --append
```

1.  接下来，我们将创建一个新的 Azure AD 应用程序。这个 Azure AD 应用程序将用于获取用户的 Azure AD 组成员资格：

```
serverApplicationId=$(az ad app create \
    --display-name "${NEWAKSNAME}Server" \
    --identifier-uris "https://${NEWAKSNAME}Server" \
    --query appId -o tsv)
```

1.  在下一步中，我们将更新应用程序，创建服务主体，并从服务主体获取密钥：

```
az ad app update --id $serverApplicationId --set groupMembershipClaims=All
az ad sp create --id $serverApplicationId
serverApplicationSecret=$(az ad sp credential reset \
    --name $serverApplicationId \
    --credential-description "AKSPassword" \
    --query password -o tsv)
```

1.  然后，我们将授予此服务主体访问 Azure AD 中的目录数据的权限：

```
az ad app permission add \
--id $serverApplicationId \
    --api 00000003-0000-0000-c000-000000000000 \
    --api-permissions e1fe6dd8-ba31-4d61-89e7-88639da4683d=Scope \
    06da0dbc-49e2-44d2-8312-53f166ab848a=Scope \
    7ab1d382-f21e-4acd-a863-ba3e13f7da61=Role
az ad app permission grant --id $serverApplicationId\
    --api 00000003-0000-0000-c000-000000000000
```

1.  这里有一个手动步骤，需要我们转到 Azure 门户。我们需要授予应用程序管理员同意。为了实现这一点，在 Azure 搜索栏中查找*Azure Active Directory*：![在 Azure 搜索栏中搜索 Azure Active Directory。](img/Figure_10.2.jpg)

###### 图 10.2：在搜索栏中查找 Azure Active Directory

1.  然后，在左侧菜单中选择**应用程序注册**：![在左侧菜单中选择应用程序注册选项卡。](img/Figure_10.3.jpg)

###### 图 10.3：选择应用程序注册

1.  在**应用程序注册**中，转到**所有应用程序**，查找*<clustername>Server*，并选择该应用程序：![查找我们之前使用脚本创建的名为 handsonaksaadserver 的应用程序注册。](img/Figure_10.4.jpg)

###### 图 10.4：查找我们之前使用脚本创建的应用程序注册

1.  在该应用的视图中，点击**API 权限**，然后点击**为默认目录授予管理员同意**（此名称可能取决于您的 Azure AD 名称）：![转到左侧菜单中的 API 权限，并点击按钮授予管理员特权。](img/Figure_10.5.jpg)

###### 图 10.5：授予管理员同意

在接下来的提示中，选择**是**以授予这些权限。

#### 注意

**授予管理员同意**按钮可能需要大约一分钟才能激活。如果还没有激活，请等待一分钟然后重试。您需要在 Azure AD 中拥有管理员权限才能授予此同意。

1.  接下来，我们将创建另一个服务主体并授予其权限。这个服务主体将接受用户的认证请求，并验证他们的凭据和权限：

```
clientApplicationId=$(az ad app create \
    --display-name "${NEWAKSNAME}Client" \
    --native-app \
    --reply-urls "https://${NEWAKSNAME}Client" \
    --query appId -o tsv)
az ad sp create --id $clientApplicationId
oAuthPermissionId=$(az ad app show --id $serverApplicationId\
--query "oauth2Permissions[0].id" -o tsv)
az ad app permission add --id $clientApplicationId \
--api$serverApplicationId --api-permissions \
$oAuthPermissionId=Scope
az ad app permission grant --id $clientApplicationId\
--api $serverApplicationId
```

1.  然后，作为最后一步，我们可以创建新的集群：

```
azaks create \
    --resource-group $RGNAME \
    --name $NEWAKSNAME \
    --location $LOCATION
    --node-count 2 \
    --node-vm-size Standard_D1_v2 \
    --generate-ssh-keys \
    --aad-server-app-id $serverApplicationId \
    --aad-server-app-secret $serverApplicationSecret \
    --aad-client-app-id $clientApplicationId \
    --aad-tenant-id $TENANTID \
    --service-principal $RBACSP \
    --client-secret $RBACSPPASSWD
```

在本节中，我们已经创建了一个集成了 Azure AD 的新 AKS 集群，用于 RBAC。创建一个新集群大约需要 5 到 10 分钟。在新集群正在创建时，您可以继续下一节并在 Azure AD 中创建新用户和组。

### 在 Azure AD 中创建用户和组

在本节中，我们将在 Azure AD 中创建一个新用户和一个新组。我们将在本章后面使用它们来分配权限给我们的 AKS 集群。

#### 注意

您需要在 Azure AD 中拥有用户管理员角色才能创建用户和组。

1.  首先，在 Azure 搜索栏中查找*Azure 活动目录*：![在 Azure 搜索栏中搜索 Azure 活动目录。](img/Figure_6.17.jpg)

###### 图 10.6：在搜索栏中查找 Azure 活动目录

1.  点击左侧的**用户**。然后选择**新用户**来创建一个新用户：![导航到左侧菜单中的所有用户选项，然后点击新用户按钮。](img/Figure_10.7.jpg)

###### 图 10.7：点击新用户以创建新用户

1.  提供有关用户的信息，包括用户名。确保记下密码，因为这将需要用于登录：![在新用户窗口中，添加所有用户详细信息并确保记下密码。](img/Figure_10.8.jpg)

###### 图 10.8：提供用户详细信息（确保记下密码）

1.  创建用户后，返回 Azure AD 刀片并选择**组**。然后点击**新建组**按钮创建一个新组：![在左侧菜单中选择所有组选项卡，然后点击新建组按钮创建一个新组。](img/Figure_10.9.jpg)

###### 图 10.9：点击新建组创建新组

1.  创建一个新的安全组。将组命名为`kubernetes-admins`，并将`Tim`添加为组的成员。然后点击底部的**创建**按钮：![在新组窗口中，将组类型设置为安全，添加组名称和组描述，并将我们在上一步中创建的用户添加到该组中。](img/Figure_10.10.jpg)

###### 图 10.10：添加组类型、组名称和组描述

1.  我们现在已经创建了一个新用户和一个新组。作为最后一步，我们将使该用户成为 AKS 中的集群所有者，以便他们可以使用 Azure CLI 访问集群。为此，在 Azure 搜索栏中搜索您的集群：![在 Azure 搜索栏中输入集群名称并选择该集群。](img/Figure_10.11.jpg)

###### 图 10.11：在 Azure 搜索栏中查找您的集群

1.  在集群刀片中，点击**访问控制（IAM）**，然后点击**添加**按钮添加新的角色分配。选择**Azure Kubernetes Service Cluster User Role**并分配给您刚创建的新用户：![在集群刀片中，选择访问控制（IAM），点击屏幕顶部的添加按钮，选择 Azure Kubernetes Service Cluster User Role，并查找我们之前创建的用户。](img/Figure_10.12.jpg)

###### 图 10.12：为新创建的用户分配集群用户角色

1.  由于我们还将为新用户使用 Cloud Shell，因此我们将为他们提供对 Cloud Shell 存储账户的贡献者访问权限。首先，在 Azure 搜索栏中搜索*存储*：![在 Azure 搜索栏中搜索存储账户。](img/Figure_10.13.jpg)

###### 图 10.13：在 Azure 搜索栏中搜索存储账户

1.  选择此存储账户所在的资源组：![在存储账户窗口中，选择由 Cloud Shell 创建的存储账户所在的资源组。](img/Figure_10.14.jpg)

###### 图 10.14：选择资源组

1.  转到**访问控制（IAM）**，然后单击**添加**按钮。将**Storage Account Contributor**角色授予您新创建的用户：![导航到访问控制（IAM）窗口，然后单击添加按钮。然后，将 Contributor 角色授予我们新创建的用户。](img/Figure_10.15.jpg)

###### 图 10.15：给予新创建的用户 Storage Account Contributor 访问权限

这已经完成了创建新用户和组，并给予该用户对 AKS 的访问权限。在下一节中，我们将为该用户和组配置 RBAC。

### 在 AKS 中配置 RBAC

为了演示 AKS 中的 RBAC，我们将创建两个命名空间，并在每个命名空间中部署 Azure Vote 应用程序。我们将给予我们的组对 Pod 的全局只读访问权限，并且我们将给予用户仅在一个命名空间中删除 Pod 的能力。实际上，我们需要在 Kubernetes 中创建以下对象：

+   `ClusterRole`来给予只读访问权限

+   `ClusterRoleBinding`来授予我们的组对该角色的访问权限

+   `Role`来在`delete-access`命名空间中给予删除权限

+   `RoleBinding`来授予我们的用户对该角色的访问权限![我们将要构建的演示的图形表示。我们创建的组将获得 ReadOnlyClusterRole，我们创建的用户在 delete-access 命名空间中获得一个角色，以授予删除 pod 的权限。](img/Figure_10.16.jpg)

###### 图 10.16：组获得对整个集群的只读访问权限，用户获得对 delete-access 命名空间的删除权限

让我们在我们的集群上设置不同的角色：

1.  要开始我们的示例，我们需要检索组的 ID。以下命令将检索组 ID：

```
az ad group show -g 'kubernetes-admins' --query objectId -o tsv
```

这将显示您的组 ID。记下来，因为我们在下一步中会需要它：

![az ad group show 命令的输出，显示组 ID。](img/Figure_10.17.jpg)

###### 图 10.17：获取组 ID

1.  由于我们为这个示例创建了一个新的集群，我们将获取凭据以登录到这个集群。我们将使用管理员凭据进行初始设置：

```
az aks get-credentials -n handsonaksad -g rg-handsonaks --admin
```

1.  在 Kubernetes 中，我们将为这个示例创建两个命名空间：

```
kubectl create ns no-access
kubectl create ns delete-access
```

1.  我们将在两个命名空间中部署`azure-vote`应用程序：

```
kubectl create -f azure-vote.yaml -n no-access
kubectl create -f azure-vote.yaml -n delete-access
```

1.  接下来，我们将创建`ClusterRole`文件。这在`clusterRole.yaml`文件中提供：

```
1   apiVersion: rbac.authorization.k8s.io/v1
2   kind: ClusterRole
3   metadata:
4     name: readOnly
5   rules:
6   - apiGroups: [""]
7     resources: ["pods"]
8     verbs: ["get", "watch", "list"]
```

让我们仔细看看这个文件：

**第 2 行**：定义了`ClusterRole`的创建

**第 4 行**：为我们的`ClusterRole`命名

**第 6 行**：给予所有 API 组的访问权限

**第 7 行**：给予所有 Pod 的访问权限

第 8 行：允许执行`get`、`watch`和`list`操作

我们将使用以下命令创建这个`ClusterRole`：

```
kubectl create -f clusterRole.yaml
```

1.  下一步是创建一个 ClusterRoleBinding。该绑定将角色链接到用户。这在`clusterRoleBinding.yaml`文件中提供：

```
1   apiVersion: rbac.authorization.k8s.io/v1
2   kind: ClusterRoleBinding
3   metadata:
4     name: readOnlyBinding
5   roleRef:
6     kind: ClusterRole
7     name: readOnly
8     apiGroup: rbac.authorization.k8s.io
9   subjects:
10  - kind: Group
11   apiGroup: rbac.authorization.k8s.io
12   name: "<group-id>"
```

让我们仔细看看这个文件：

第 2 行：定义我们正在创建一个`ClusterRoleBinding`

第 4 行：为我们的`ClusterRoleBinding`命名

第 5-8 行：指的是我们在上一步中创建的`ClusterRole`

第 9-12 行：在 Azure AD 中引用我们的组

我们可以使用以下命令创建这个`ClusterRoleBinding`：

```
kubectl create -f clusterRoleBinding.yaml
```

1.  接下来，我们将创建一个限制在`delete-access`命名空间的`Role`。这在`role.yaml`文件中提供：

```
1   apiVersion: rbac.authorization.k8s.io/v1
2   kind: Role
3   metadata:
4     name: deleteRole
5     namespace: delete-access
6   rules:
7   - apiGroups: [""]
8     resources: ["pods"]
9     verbs: ["delete"]
```

这个文件类似于之前的`ClusterRole`文件。有两个有意义的区别：

第 2 行：定义我们正在创建一个`Role`，而不是`ClusterRole`

第 5 行：定义了在哪个命名空间中创建这个`Role`

我们可以使用以下命令创建这个`Role`：

```
kubectl create -f role.yaml
```

1.  最后，我们将创建将我们的用户链接到命名空间角色的`RoleBinding`。这在`roleBinding.yaml`文件中提供：

```
1   apiVersion: rbac.authorization.k8s.io/v1
2   kind: RoleBinding
3   metadata:
4     name: deleteBinding
5     namespace: delete-access
6   roleRef:
7     kind: Role
8     name: deleteRole
9     apiGroup: rbac.authorization.k8s.io
10  subjects:
11  - kind: User
12    apiGroup: rbac.authorization.k8s.io
13    name: "<user e-mail address>"
```

这个文件类似于之前的 ClusterRoleBinding 文件。有一些有意义的区别：

第 2 行：定义了创建一个`RoleBinding`而不是`ClusterRoleBinding`

第 5 行：定义了在哪个命名空间中创建这个`RoleBinding`

第 7 行：指的是一个普通的`Role`而不是`ClusterRole`

第 11-13 行：定义了我们的用户而不是一个组

我们可以使用以下命令创建这个`RoleBinding`：

```
kubectl create -f roleBinding.yaml
```

这已经满足了 RBAC 的要求。我们已经创建了两个角色并设置了两个 RoleBindings。在下一节中，我们将通过以我们的用户身份登录到集群来探索 RBAC 的影响。

### 验证 RBAC

为了验证 RBAC 是否按预期工作，我们将使用新创建的用户登录到 Azure 门户。在新的浏览器或 InPrivate 窗口中转到[`portal.azure.com`](https://portal.azure.com)，并使用新创建的用户登录。您将立即收到更改密码的提示。这是 Azure AD 中的安全功能，以确保只有该用户知道他们的密码：

![提示更改密码的浏览器窗口。](img/Figure_10.18.jpg)

###### 图 10.18：您将被要求更改密码

一旦我们更改了您的密码，我们就可以开始测试不同的 RBAC 角色：

1.  我们将通过为新用户设置 Cloud Shell 来开始我们的实验。启动 Cloud Shell 并选择 Bash：![选择 Bash 选项作为 Cloud Shell。](img/Figure_10.19.jpg)

###### 图 10.19：选择 Bash 作为 Cloud Shell

1.  在下一个视图中，选择**显示高级设置**：![在导航到 bash 选项后，选择显示高级设置按钮。](img/Figure_10.20.jpg)

###### 图 10.20：选择显示高级设置

1.  然后，将 Cloud Shell 指向现有的存储账户并创建一个新的文件共享：![通过单击创建存储按钮，将 Cloud Shell 指向现有的存储账户以创建新的文件共享。](img/Figure_10.21.jpg)

###### 图 10.21：指向现有的存储账户并创建一个新的文件共享

1.  一旦 Cloud Shell 可用，让我们获取连接到我们的 AKS 集群的凭据：

```
az aks get-credentials -n handsonaksaad -g rg-handsonaks
```

1.  然后，我们将尝试在 kubectl 中执行一个命令。让我们尝试获取集群中的节点：

```
kubectl get nodes
```

由于这是针对启用 RBAC 的集群执行的第一个命令，您将被要求重新登录。浏览至[`microsoft.com/devicelogin`](https://microsoft.com/devicelogin)并提供 Cloud Shell 显示给您的代码。确保您在此处使用新用户的凭据登录：

![在提示窗口中输入 Cloud Shell 提供的代码，然后单击下一步按钮。](img/Figure_10.22.jpg)

###### 图 10.22：在提示中复制粘贴 Cloud Shell 显示给您的代码

登录后，您应该从 kubectl 收到一个`Forbidden`错误消息，通知您您没有权限查看集群中的节点。这是预期的，因为用户只被配置为可以访问 Pods：

![kubectl 给出错误消息，并声明我们没有权限查看集群中的节点。](img/Figure_10.23.jpg)

###### 图 10.23：提示您登录和被禁止的消息

1.  现在我们可以验证我们的用户可以查看所有命名空间中的 Pods，并且用户有权限在`delete-access`命名空间中删除 Pods：

```
kubectl get pods -n no-access
kubectl get pods -n delete-access
```

这应该对两个命名空间都成功。这是由于为用户组配置的`ClusterRole`：

![输出显示我们的用户可以查看两个命名空间中的 Pods。](img/Figure_10.24.jpg)

###### 图 10.24：我们的用户可以查看两个命名空间中的 Pods

1.  让我们也验证一下“删除”权限：

```
kubectl delete pod --all -n no-access
kubectl delete pod --all -n delete-access
```

正如预期的那样，在`no-access`命名空间中被拒绝，在`delete-access`命名空间中被允许，如*图 10.25*所示：

![验证删除权限显示在无访问命名空间中被拒绝，在删除访问命名空间中被允许。](img/Figure_10.25.jpg)

###### 图 10.25: 在无访问命名空间中被拒绝，在删除访问命名空间中被允许

在本节中，我们已经设置了一个集成了 Azure AD 的新集群，并验证了与 Azure AD 身份的 RBAC 的正确配置。让我们清理本节中创建的资源，获取现有集群的凭据，并将我们的常规集群缩减到两个节点：

```
az aks delete -n handsonaksaad -g rg-handsonaks
az aks get-credentials -n handsonaks -g rg-handsonaks
az aks nodepool scale --cluster-name handsonaks \
  -g rg-handsonaks --name agentpool --node-count 2
```

在下一节中，我们将继续探讨 Kubernetes 安全性的路径，这次是调查 Kubernetes 密码。

## 设置密码管理

所有生产应用程序都需要一些秘密信息才能运行。Kubernetes 具有可插拔的密码后端来管理这些密码。Kubernetes 还提供了多种在部署中使用密码的方式。管理密码并正确使用密码后端的能力将使您的服务能够抵抗攻击。

在之前的章节中，我们在一些部署中使用了密码。大多数情况下，我们将密码作为某种变量的字符串传递，或者 Helm 负责为我们创建密码。在 Kubernetes 中，密码是一种资源，就像 Pods 和 ReplicaSets 一样。密码始终与特定的命名空间相关联。必须在要使用它们的所有命名空间中创建密码。在本节中，我们将学习如何创建、解码和使用我们自己的密码。我们将首先使用 Kubernetes 中的内置密码，最后利用 Azure Key Vault 来存储密码。

### 创建您自己的密码

Kubernetes 提供了三种创建密码的方式，如下所示：

+   从文件创建密码

+   从 YAML 或 JSON 定义创建密码

+   从命令行创建密码

使用上述任何方法，您可以创建三种类型的密码：

+   **通用密码**: 这些可以使用文字值创建。

+   **Docker-registry 凭据**: 用于从私有注册表中拉取镜像。

+   **TLS 证书**: 用于存储 SSL 证书。

我们将从使用文件方法创建密码开始。

**从文件创建密码**

假设您需要存储用于访问 API 的 URL 和秘密令牌。为了实现这一点，您需要按照以下步骤进行操作：

1.  将 URL 存储在`apiurl.txt`中，如下所示：

```
echo https://my-secret-url-location.topsecret.com \
> secreturl.txt
```

1.  将令牌存储在另一个文件中，如下所示：

```
echo 'superSecretToken' > secrettoken.txt
```

1.  让 Kubernetes 从文件中创建密码，如下所示：

```
kubectl create secret generic myapi-url-token \
--from-file=./secreturl.txt --from-file=./secrettoken.txt
```

在这个命令中，我们将秘密类型指定为`generic`。

该命令应返回以下输出：

```
secret/myapi-url-token created
```

1.  我们可以使用`get`命令检查秘密是否与任何其他 Kubernetes 资源以相同的方式创建：

```
kubectl get secrets
```

此命令将返回类似于*图 10.26*的输出：

![kubectl get secrets 命令的输出显示了秘密的名称、类型、数据和年龄。输出中有一个高亮显示，突出显示了名为 myapi-url-token 的秘密，类型为 Opaque。](img/Figure_10.26.jpg)

###### 图 10.26：kubectl get secrets 的输出

在这里，您将看到我们刚刚创建的秘密，以及仍然存在于`default`命名空间中的任何其他秘密。我们的秘密是`Opaque`类型，这意味着从 Kubernetes 的角度来看，内容的模式是未知的。它是一个任意的键值对，没有约束，与 Docker 注册表或 TLS 秘密相反，后者具有将被验证为具有所需详细信息的模式。

1.  要了解有关秘密的更多详细信息，您还可以运行`describe`命令：

```
kubectl describe secrets/myapi-url-token
```

您将获得类似于*图 10.27*的输出：

![描述命令的输出显示了有关秘密的其他详细信息。](img/Figure_10.27.jpg)

###### 图 10.27：描述秘密的输出

正如您所看到的，前面的命令都没有显示实际的秘密值。

1.  要获取秘密，请运行以下命令：

```
kubectl get -o yaml secrets/myapi-url-token
```

您将获得类似于*图 10.28*的输出：

![通过 kubectl get secret 命令中的-o yaml 开关显示秘密的编码值的输出。](img/Figure_10.28.jpg)

###### 图 10.28：在 kubectl get secret 中使用-o yaml 开关可以获取秘密的编码值

数据以键值对的形式存储，文件名作为键，文件的 base64 编码内容作为值。

1.  前面的值是 base64 编码的。Base64 编码并不安全。它会使秘密变得难以被操作员轻松阅读，但任何坏人都可以轻松解码 base64 编码的秘密。要获取实际值，请运行以下命令：

```
echo 'c3VwZXJTZWNyZXRUb2tlbgo=' | base64 -d
```

您将获得最初输入的值：

![获取最初使用 echo <encoded secret> | base64 -d 命令输入的秘密的实际值。](img/Figure_10.29.jpg)

###### 图 10.29：Base64 编码的秘密可以轻松解码

1.  同样，对于`url`值，您可以运行以下命令：

```
echo 'aHR0cHM6Ly9teS1zZWNyZXQtdXJsLWxvY2F0aW9uLnRvcHNlY3JldC5jb20K'| base64 -d
```

您将获得最初输入的`url`值，如*图 10.30*所示：

![显示编码的 URL 的输出，这是最初输入的 URL。](img/Figure_10.30.jpg)

###### 图 10.30：编码的 URL 也可以很容易地解码

在本节中，我们能够使用文件对 URL 进行编码，并获取实际的秘密值。让我们继续探索第二种方法——从 YAML 或 JSON 定义创建秘密。

**使用 YAML 文件手动创建秘密**

我们将按照以下步骤使用 YAML 文件创建相同的秘密：

1.  首先，我们需要将秘密编码为`base64`，如下所示：

```
echo 'superSecretToken' | base64
```

您将获得以下价值：

```
c3VwZXJTZWNyZXRUb2tlbgo=
```

您可能会注意到，这与我们在上一节中获取秘密的`yaml`定义时存在的值相同。

1.  同样，对于`url`值，我们可以获取 base64 编码的值，如下面的代码块所示：

```
echo 'https://my-secret-url-location.topsecret.com' | base64
```

这将给您`base64`编码的 URL：

```
aHR0cHM6Ly9teS1zZWNyZXQtdXJsLWxvY2F0aW9uLnRvcHNlY3JldC5jb20K
```

1.  现在我们可以手动创建秘密定义；然后保存文件。该文件已在代码包中提供，名称为`myfirstsecret.yaml`：

```
1   apiVersion: v1
2   kind: Secret
3   metadata:
4     name: myapiurltoken-yaml
5   type: Opaque
6   data:
7     url: aHR0cHM6Ly9teS1zZWNyZXQtdXJsLWxvY2F0aW9uLnRvcHNlY3JldC5jb20K
8     token: c3VwZXJTZWNyZXRUb2tlbgo=
```

让我们调查一下这个文件：

**第 2 行**：这指定了我们正在创建一个秘密。

**第 5 行**：这指定了我们正在创建一个`Opaque`秘密，这意味着从 Kubernetes 的角度来看，值是无约束的键值对。

**第 7-8 行**：这些是我们秘密的 base64 编码值。

1.  现在，我们可以像任何其他 Kubernetes 资源一样使用`create`命令创建秘密：

```
kubectl create -f myfirstsecret.yaml
```

1.  我们可以通过以下方式验证我们的秘密是否已成功创建：

```
kubectl get secrets
```

这将显示一个类似于*图 10.31*的输出：

![验证我们的秘密是否已成功使用 kubectl get secrets 命令创建的输出。](img/Figure_10.31.jpg)

###### 图 10.31：我们的秘密已成功从 YAML 文件创建

1.  您可以通过在上一节中描述的方式使用`kubectl get -o yaml secrets/myapiurltoken-yaml`来双重检查秘密是否相同。

**使用文字创建通用秘密**

创建秘密的第三种方法是使用`literal`方法，这意味着您可以在命令行上传递值。要做到这一点，请运行以下命令：

```
kubectl create secret generic myapiurltoken-literal \
--from-literal=token='superSecretToken' \
--from-literal=url=https://my-secret-url-location.topsecret.com
```

我们可以通过运行以下命令来验证秘密是否已创建：

```
kubectl get secrets
```

这将给我们一个类似于*图 10.32*的输出：

![运行 kubectl get secrets 命令的输出，验证我们的秘密是否已成功从 CLI 创建。](img/Figure_10.32.jpg)

###### 图 10.32：我们的秘密已成功从 CLI 创建

因此，我们已经使用文字值创建了秘密，除了前面两种方法。

### 创建 Docker 注册表密钥

在生产环境中连接到私有 Docker 注册表是必要的。由于这种用例非常常见，Kubernetes 提供了创建连接的机制：

```
kubectl create secret docker-registry <secret-name> \
--docker-server=<your- registry-server> \
--docker-username=<your-name> \
--docker-password=<your-pword> --docker-email=<your-email>
```

第一个参数是秘密类型，即`docker-registry`。然后，您给秘密一个名称；例如，`regcred`。其他参数是 Docker 服务器（[`index.docker.io/v1/`](https://index.docker.io/v1/)用于 Docker Hub），您的用户名，您的密码和您的电子邮件。

您可以使用`kubectl`访问秘密以相同的方式检索秘密。

在 Azure 中，**Azure 容器注册表**（**ACR**）最常用于存储容器映像。集群可以连接到 ACR 的方式有两种。第一种方式是在集群中使用像我们刚刚描述的秘密。第二种方式 - 这是推荐的方式 - 是使用服务主体。我们将在*第十一章，无服务器函数*中介绍集成 AKS 和 ACR。

Kubernetes 中的最终秘密类型是 TLS 秘密。

### 创建 TLS 秘密

TLS 秘密用于存储 TLS 证书。要创建可用于 Ingress 定义的 TLS 秘密，我们使用以下命令：

```
kubectl create secret tls <secret-name> --key <ssl.key> --cert <ssl.crt>
```

第一个参数是`tls`，用于设置秘密类型，然后是`key`值和实际的证书值。这些文件通常来自您的证书注册商。

#### 注意

我们在*第六章*，*管理您的 AKS*，集群中创建了 TLS 秘密，在那里我们使用`cert-manager`代表我们创建了这些秘密。

如果您想生成自己的秘密，可以运行以下命令生成自签名 SSL 证书：

`openssl req -x509 -nodes -days 365 -newkey rsa:2048 - keyout /tmp/ssl.key -out /tmp/ssl.crt -subj "/CN=foo.bar.com"`

在本节中，我们介绍了 Kubernetes 中不同的秘密类型，并看到了如何创建秘密。在下一节中，我们将在我们的应用程序中使用这些秘密。

### 使用您的秘密

秘密一旦创建，就需要与应用程序进行关联。这意味着 Kubernetes 需要以某种方式将秘密的值传递给正在运行的容器。Kubernetes 提供了两种将秘密链接到应用程序的方式：

+   将秘密用作环境变量

+   将秘密挂载为文件

将秘密挂载为文件是在应用程序中使用秘密的最佳方法。在本节中，我们将解释两种方法，并展示为什么最好使用第二种方法。

### 作为环境变量的秘密

在 Pod 定义中引用秘密在`containers`和`env`部分下。我们将使用之前在 Pod 中定义的秘密，并学习如何在应用程序中使用它们：

1.  我们可以配置一个具有环境变量秘密的 Pod，就像在`pod-with-env-secrets.yaml`中提供的定义：

```
1   apiVersion: v1
2   kind: Pod
3   metadata:
4     name: secret-using-env
5   spec:
6     containers:
7     - name: nginx
8       image: nginx
9       env:
10        - name: SECRET_URL
11          valueFrom:
12            secretKeyRef:
13              name: myapi-url-token
14              key: secreturl.txt
15        - name: SECRET_TOKEN
16          valueFrom:
17            secretKeyRef:
18              name: myapi-url-token
19              key: secrettoken.txt
20    restartPolicy: Never
```

让我们检查一下这个文件：

**第 9 行**：在这里，我们设置环境变量。

**第 11-14 行**：在这里，我们引用`myapi-url-token`秘密中的`secreturl.txt`文件。

**第 16-19 行**：在这里，我们引用`myapi-url-token`秘密中的`secrettoken.txt`文件。

1.  现在让我们创建 Pod 并看看它是否真的起作用：

```
kubectl create -f pod-with-env-secrets.yaml
```

1.  检查环境变量是否设置正确：

```
kubectl exec -it secret-using-env sh
echo $SECRET_URL
echo $SECRET_TOKEN
```

这应该显示与*图 10.33*类似的结果：

![通过使用 kubectl exec 在容器中打开 shell 并运行 echo 来验证环境变量是否设置正确，以获取秘密值。](img/Figure_10.33.jpg)

###### 图 10.33：我们可以在 Pod 内部获取秘密

任何应用程序都可以通过引用适当的`env`变量来使用秘密值。请注意，应用程序和 Pod 定义都没有硬编码的秘密。

### 将秘密作为文件

让我们看看如何将相同的秘密挂载为文件。我们将使用以下 Pod

定义以演示如何完成此操作。它在`pod-with-env-secrets.yaml`文件中提供：

```
1   apiVersion: v1
2   kind: Pod
3   metadata:
4     name: secret-using-volume
5   spec:
6     containers:
7     - name: nginx
8       image: nginx
9       volumeMounts:
10      - name: secretvolume
11        mountPath: "/etc/secrets"
12        readOnly: true
13    volumes:
14    - name: secretvolume
15      secret:
16        secretName: myapi-url-token
```

让我们仔细看看这个文件：

+   **第 9-12 行**：在这里，我们提供挂载详细信息。我们将`/etc/secrets`目录挂载为只读。

+   **第 13-16 行**：在这里，我们引用秘密。请注意，秘密中的两个值都将挂载到容器中。

请注意，这比`env`定义更简洁，因为您不必为每个秘密定义名称。但是，应用程序需要有特殊的代码来读取文件的内容，以便正确加载它。

让我们看看秘密是否传递了：

1.  使用以下命令创建 Pod：

```
kubectl create -f pod-with-vol-secret.yaml
```

1.  回显挂载卷中文件的内容：

```
kubectl exec -it secret-using-volume bash
cd /etc/secrets/ 
cat secreturl.txt
cat /etc/secrets/secrettoken.txt 
```

如您在*图 10.34*中所见，我们的 Pod 中存在秘密：

![输出显示我们的 Pod 中的秘密可用作文件。](img/Figure_10.34.jpg)

###### 图 10.34：我们的 Pod 中的秘密可用作文件

我们已经讨论了将秘密传递给运行中容器的两种方法。在下一节中，我们将解释为什么最好使用文件方法。

### 为什么将秘密作为文件是最佳方法

尽管将秘密作为环境变量是一种常见做法，但将秘密作为文件挂载更安全。Kubernetes 安全地处理秘密作为环境变量，但 Docker 运行时不安全地处理它们。要验证这一点，您可以运行以下命令以在 Docker 运行时看到明文的秘密：

1.  首先使用以下命令获取秘密 Pod 正在运行的实例：

```
kubectl describe pod secret-using-env | grep Node
```

这应该向您显示实例 ID，如*图 10.35*所示：

使用 kubectl describe pod secret-using-env 命令执行以获取实例 ID。

###### 图 10.35：获取实例 ID

1.  接下来，获取正在运行的 Pod 的 Docker ID：

```
kubectl describe pod secret-using-env | grep 'docker://'
```

这应该向您显示 Docker ID：

使用 kubectl describe pod secret-using-env 命令获取 Docker ID。

###### 图 10.36：获取 Docker ID

1.  最后，我们将在运行我们的容器的节点中执行一个命令，以显示我们作为环境变量传递的秘密：

```
INSTANCE=<provide instance number>
DOCKERID=<provide Docker ID>
VMSS=$(az vmss list --query '[].name' -o tsv)
RGNAME=$(az vmss list --query '[].resourceGroup' -o tsv)
az vmss run-command invoke -g $RGNAME -n $VMSS --command-id \
RunShellScript --instance-id $INSTANCE --scripts \
"docker inspect -f '{{ .Config.Env }}' $DOCKERID" \
-o yaml| grep SECRET
```

这将向您显示明文的两个秘密：

输出显示秘密在 Docker 运行时被解码

###### 图 10.37：秘密在 Docker 运行时被解码

如您所见，秘密在 Docker 运行时被解码。这意味着任何有权访问机器的操作者都将能够访问这些秘密。这也意味着大多数日志系统将记录敏感秘密。

#### 注意

RBAC 对于控制秘密也非常重要。拥有对集群的访问权限并具有正确角色的人可以访问存储的秘密。由于秘密只是 base64 编码的，任何具有对秘密的 RBAC 权限的人都可以解码它们。建议谨慎对待对秘密的访问，并非常小心地授予人们使用`kubectl exec`命令获取容器 shell 的访问权限。

让我们确保清理掉我们在这个示例中创建的资源：

```
kubectl delete pod --all
kubectl delete secret myapi-url-token \
myapiurltoken-literal myapiurltoken-yaml
```

现在我们已经使用默认的秘密机制在 Kubernetes 中探索了秘密，让我们继续使用一个更安全的选项，即 Key Vault。

## 使用存储在 Key Vault 中的秘密

在上一节中，我们探讨了存储在 Kubernetes 中的本地秘密。这意味着它们以 base64 编码存储在 Kubernetes API 服务器上（在后台，它们将存储在 etcd 数据库中，但这是微软提供的托管服务的一部分）。我们在上一节中看到，base64 编码的秘密根本不安全。对于高度安全的环境，您将希望使用更好的秘密存储。

Azure 提供了符合行业标准的秘密存储解决方案，称为 Azure 密钥保管库。这是一个托管服务，可以轻松创建、存储和检索秘密，并提供对秘密访问的良好监控。微软维护了一个开源项目，允许您将密钥保管库中的秘密挂载到您的应用程序中。这个解决方案称为密钥保管库 FlexVolume，可以在这里找到：[`github.com/Azure/kubernetes-keyvault-flexvol`](https://github.com/Azure/kubernetes-keyvault-flexvol)。

在本节中，我们将创建一个密钥保管库，并安装密钥保管库 FlexVolume 以挂载存储在密钥保管库中的秘密到 Pod 中。

### 创建密钥保管库

我们将使用 Azure 门户创建密钥保管库：

1.  要开始创建过程，请在 Azure 搜索栏中查找*密钥保管库*：![在 Azure 搜索栏中搜索密钥保管库。](img/Figure_10.38.jpg)

###### 图 10.38：在 Azure 搜索栏中查找密钥保管库

1.  点击“添加”按钮开始创建过程：![导航到左上角并点击“添加”按钮开始创建密钥保管库。](img/Figure_10.39.jpg)

###### 图 10.39：单击“添加”按钮开始创建密钥保管库

1.  提供创建密钥保管库的详细信息。密钥保管库的名称必须是全局唯一的，因此考虑在名称中添加您的缩写。建议在与您的集群相同的区域创建密钥保管库：![输入订阅、密钥保管库名称和区域等详细信息以创建密钥保管库。](img/Figure_10.40.jpg)

###### 图 10.40：提供创建密钥保管库的详细信息

1.  提供详细信息后，点击“审核+创建”按钮以审核并创建您的密钥保管库。点击“创建”按钮完成创建过程。

1.  创建您的密钥保管库需要几秒钟的时间。一旦保管库创建完成，打开它，转到秘密，然后点击**生成/导入**按钮创建一个新的秘密：![导航到左侧导航中的秘密选项卡，并点击生成/导入按钮创建新秘密。](img/Figure_10.41.jpg)

###### 图 10.41：创建新秘密

1.  在秘密创建向导中，提供有关您的秘密的详细信息。为了使演示更容易跟进，建议使用名称`k8s-secret-demo`。点击屏幕底部的**创建**按钮来创建秘密：![输入创建新秘密的详细信息。](img/Figure_10.42.jpg)

###### 图 10.42：提供新秘密的详细信息

现在我们在密钥保管库中有一个秘密，我们可以继续配置 Key Vault FlexVolume 以在 Kubernetes 中访问此秘密。

### 设置 Key Vault FlexVolume

在本节中，我们将在我们的集群中设置 Key Vault FlexVolume。这将允许我们从 Key Vault 中检索秘密：

1.  使用以下命令创建 Key Vault FlexVolume。`kv-flexvol-installer.yaml`文件已在本章的源代码中提供：

```
kubectl create -f kv-flexvol-installer.yaml
```

#### 注意

我们已经提供了`kv-flexvol-installer.yaml`文件，以确保与本书中的示例一致。对于生产用例，我们建议安装最新版本，可在[`github.com/Azure/kubernetes-keyvault-flexvol`](https://github.com/Azure/kubernetes-keyvault-flexvol)上找到。

1.  FlexVolume 需要凭据才能连接到 Key Vault。在这一步中，我们将创建一个新的服务主体：

```
APPID=$(az ad app create \
    --display-name "flex" \
    --identifier-uris "https://flex" \
    --query appId -o tsv)
az ad sp create --id $APPID
APPPASSWD=$(az ad sp credential reset \
    --name $APPID \
    --credential-description "KeyVault" \
    --query password -o tsv)
```

1.  现在我们将在 Kubernetes 中创建两个秘密来存储服务主体连接：

```
kubectl create secret generic kvcreds \
--from-literal=clientid=$APPID \
--from-literal=clientsecret=$APPPASSWD --type=azure/kv
```

1.  现在我们将为此服务主体授予对密钥保管库中秘密的访问权限：

```
KVNAME=handsonaks-kv
az keyvault set-policy -n $KVNAME --key-permissions \
  get --spn $APPID
az keyvault set-policy -n $KVNAME --secret-permissions \
  get --spn $APPID
az keyvault set-policy -n $KVNAME --certificate-permissions \
  get --spn $APPID
```

您可以在门户中验证这些权限是否已成功设置。在您的密钥保管库中，在**访问策略**部分，您应该看到 flex 应用程序对密钥、秘密和证书具有`获取`权限：

![在访问策略部分，我们可以看到 flex 应用程序对密钥、秘密和证书具有获取权限。](img/Figure_10.43.jpg)

###### 图 10.43：flex 应用程序对密钥、秘密和证书具有获取权限

#### 注意

Key Vault FlexVolume 支持多种身份验证选项。我们现在正在使用预创建的服务主体。FlexVolume 还支持使用托管标识，可以使用 Pod 标识或使用托管集群的**虚拟机规模集**（**VMSS**）的标识。本书不会探讨这些其他身份验证选项，但鼓励您在 https://github.com/Azure/kubernetes-keyvault-flexvol 上阅读更多。

这就结束了 Key Vault FlexVolume 的设置。在下一节中，我们将使用 FlexVolume 来访问秘密并将其挂载在 Pod 中。

### 使用 Key Vault FlexVolume 在 Pod 中挂载秘密

在这一部分，我们将把一个来自 Key Vault 的秘密挂载到一个新的 Pod 中。

1.  我们提供了一个名为`pod_secret_flex.yaml`的文件，它将帮助创建一个挂载 Key Vault 秘密的 Pod。您需要对这个文件进行两处更改：

```
1   apiVersion: v1
2   kind: Pod
3   metadata:
4     name: nginx-secret-flex
5   spec:
6     containers:
7     - name: nginx
8       image: nginx
9       volumeMounts:
10      - name: test
11        mountPath: /etc/secret/
12        readOnly: true
13    volumes:
14    - name: test
15      flexVolume:
16        driver: "azure/kv"
17        secretRef:
18          name: kvcreds
19        options:
20          keyvaultname: <keyvault name>
21          keyvaultobjectnames: k8s-secret-demo
22          keyvaultobjecttypes: secret
23          tenantid: "<tenant ID>"
```

让我们调查一下这个文件：

**第 9-12 行**：类似于将秘密作为文件挂载的示例，我们还提供了一个`volumeMount`来将我们的秘密作为文件挂载。

**第 13-23 行**：这是指向 FlexVolume 的卷配置。

**第 17-18 行**：在这里，我们提到了在上一个示例中创建的服务主体凭据。

**第 20-23 行**：您需要更改这些值以代表您的环境。

1.  我们可以使用以下命令创建此 Pod：

```
kubectl create -f pod_secret_flex.yaml
```

1.  一旦 Pod 被创建并运行，我们可以在 Pod 中执行并验证秘密是否存在：

```
kubectl exec -it nginx-secret-flex bash
cd /etc/secret
cat k8s-secret-demo
```

这应该输出我们在 Key Vault 中创建的秘密，如图 10.43 所示：

![显示我们在 Key Vault 中配置的秘密被挂载在 Pod 中作为文件的输出。](img/Figure_10.44.jpg)

###### 图 10.44：我们在 Key Vault 中配置的秘密被挂载在 Pod 中作为一个文件

我们已成功使用 Key Vault 存储秘密。秘密不再以 base64 编码的方式存储在我们的集群中，而是安全地存储在集群外的 Key Vault 中。我们仍然可以使用 Key Vault FlexVolume 在集群中访问秘密。

让我们确保清理我们的部署：

```
kubectl delete -f pod_secret_flex.yaml
kubectl delete -f kv-flexvol-installer.yaml
kubectl delete secret kvcreds
```

在这一部分，我们已经看到了创建和挂载秘密的多种方式。我们已经探讨了使用文件、YAML 文件和直接从命令行创建秘密的方法。我们还探讨了秘密可以如何被使用，可以作为环境变量或作为挂载文件。然后，我们研究了一种更安全的使用秘密的方式，即通过 Azure Key Vault。

在下一节中，我们将探讨使用 Istio 服务网格在我们的集群中配置额外的网络安全。

## Istio 服务网格为您服务

我们已经找到了一些保护我们的 Pod 的方法，但我们的网络连接仍然是开放的。集群中的任何 Pod 都可以与同一集群中的任何其他 Pod 通信。作为一个站点可靠性工程师，你会希望强制执行入口和出口规则。此外，你还希望引入流量监控，并希望有更好的流量控制。作为开发人员，你不想被所有这些要求所困扰，因为你不知道你的应用将被部署在哪里，或者什么是允许的。最好的解决方案是一个工具，让我们可以按原样运行应用程序，同时指定网络策略、高级监控和流量控制。

进入服务网格。这被定义为控制服务之间通信的层。服务网格是微服务之间的网络。服务网格被实现为控制和监视不同微服务之间流量的软件。通常，服务网格利用边车来透明地实现功能。如果你记得，Kubernetes 中的 Pod 可以由一个或多个容器组成。边车是添加到现有 Pod 中以实现附加功能的容器；在服务网格的情况下，这个功能就是服务网格的功能。

#### 注意

服务网格控制的远不止网络安全。如果你的集群只需要网络安全，请考虑采用网络策略。

与微服务一样，服务网格的实现并不是免费的午餐。如果你没有数百个微服务在运行，你可能不需要服务网格。如果你决定你真的需要一个，你首先需要选择一个。有四个流行的选项，每个都有自己的优势：

+   Linkerd (https://linkerd.io/)，包括 Conduit (https://conduit.io/)

+   Istio (https://istio.io/)

+   Consul (https://www.consul.io/mesh.html)

你应该根据自己的需求选择一个服务网格，并且放心任何一个解决方案都会适合你。在本节中，你将介绍 Istio 服务网格。我们选择了 Istio 服务网格，因为它很受欢迎。我们使用 GitHub 上的星星和提交作为受欢迎程度的衡量标准。

### 描述 Istio 服务网格

Istio 是由 IBM、Google 和 Lyft 创建的服务网格。该项目于 2017 年 5 月宣布，并于 2018 年 7 月达到稳定的 v1 版本。Istio 是服务网格的控制平面部分。Istio 默认使用`Envoy` sidecar。在本节中，我们将尝试解释什么是服务网格，以及 Istio 服务网格的核心功能是什么。

#### 注意

在本书中，我们只是简要地涉及了服务网格和 Istio。Istio 不仅是一个非常强大的工具，可以保护，还可以管理云原生应用程序的流量。在本书中，我们没有涵盖很多细节和功能。

在功能方面，服务网格（特别是 Istio）具有许多核心功能。其中之一是流量管理。在这种情况下，流量管理一词意味着流量路由的控制。通过实施服务网格，您可以控制流量以实施 A/B 测试或金丝雀发布。没有服务网格，您需要在应用程序的核心代码中实施该逻辑，而有了服务网格，该逻辑是在应用程序之外实施的。

流量管理还带来了 Istio 提供的额外安全性。Istio 可以管理身份验证、授权和服务之间通信的加密。这确保只有经过授权的服务之间进行通信。在加密方面，Istio 可以实施**双向 TLS**（**mTLS**）来加密服务之间的通信。

Istio 还具有实施策略的能力。策略可用于限制某些流量的速率（例如，每分钟只允许 x 个事务），处理对服务的访问的白名单和黑名单，或实施标头重写和重定向。

最后，Istio 为应用程序中不同服务之间的流量提供了巨大的可见性。Istio 可以执行流量跟踪、监控和日志记录服务之间的流量。您可以使用这些信息创建仪表板来展示应用程序性能，并使用这些信息更有效地调试应用程序故障。

在接下来的部分，我们将安装 Istio 并配置 Pod 之间流量的 mTLS 加密。这只是平台的许多功能之一。我们鼓励您走出书本，了解更多关于 Istio 的信息。

### 安装 Istio

安装 Istio 很容易；要这样做，请按照以下步骤操作：

1.  转到您的主目录并下载`istio`软件包，如下所示：

```
cd ~
curl -L https://istio.io/downloadIstio | sh -
```

1.  将`istio`二进制文件添加到您的路径。首先，获取您正在运行的 Istio 版本号：

```
ls | grep istio 
```

这应该显示类似于*图 10.45*的输出：

![运行 ls | grepistio 命令显示正在运行的 Istio 版本号的输出。](img/Figure_10.45.jpg)

###### 图 10.45：获取您的 Istio 版本号

记下 Istio 版本，并将其用作以下方式将二进制文件添加到您的路径：

```
export PATH="$PATH:~/istio-<release-number>/bin"
```

1.  使用以下命令检查您的集群是否可以用于运行 Istio：

```
istioctl verify-install
```

1.  使用演示配置文件安装`istio`：

```
istioctl manifest apply --set profile=demo
```

#### 注意

演示配置文件非常适合用于演示 Istio，但不建议用于生产安装。

1.  确保一切都已经启动运行，如下所示：

```
kubectl get svc -n istio-system
```

您应该在`istio-system`命名空间中看到许多服务：

![显示 istio-system 命名空间中所有服务的输出。](img/Figure_10.46.jpg)

###### 图 10.46：所有 Istio 服务都已经启动运行

您现在已经安装并运行了 Istio。

### 自动注入 Envoy 作为边车

如本节介绍中所述，服务网格使用边车来实现功能。 Istio 具有使用命名空间中的标签自动安装其使用的`Envoy`边车的能力。我们可以通过以下步骤使其以这种方式运行：

1.  让我们使用适当的标签（即`istio-injection=enabled`）为默认命名空间打标签：

```
kubectl label namespace default istio-injection=enabled
```

1.  让我们启动一个应用程序，看看边车是否确实自动部署（`bookinfo.yaml`文件在本章的源代码中提供）：

```
kubectl create -f bookinfo.yaml
```

获取在默认命名空间上运行的 Pods。Pods 可能需要几秒钟才能显示出来，并且所有 Pods 变为`Running`可能需要几分钟：

```
kubectl get pods
```

1.  在任何一个 Pod 上运行`describe`命令：

```
kubectl describe pods/productpage-v1-<pod-ID>
```

您可以看到边车确实已经应用：

![显示 Istio 自动注入边车代理列出产品页面和 istio-proxy 的详细信息的输出。](img/Figure_10.47.jpg)

###### 图 10.47：Istio 自动注入边车代理

请注意，即使没有对基础应用程序进行任何修改，我们也能够部署并附加 Istio 服务网格到容器。

### 强制使用双向 TLS

为了加密所有的服务对服务流量，我们将启用 mTLS。默认情况下，不强制使用双向 TLS。在本节中，我们将逐步强制执行 mTLS 身份验证。

#### 注意

如果您想了解 Istio 中端到端安全框架的更多信息，请阅读 https://istio.io/docs/concepts/security/#authentication-policies。有关双向 TLS 的更多细节，请阅读 https://istio.io/docs/concepts/security/#mutual-tls-authentication。

**部署示例服务**

在这个例子中，您将部署两个服务，`httpbin`和`sleep`，在不同的命名空间下。其中两个命名空间，`foo`和`bar`，将成为服务网格的一部分。这意味着它们将拥有`istio` sidecar 代理。您将在这里学习一种不同的注入 sidecar 的方式。第三个命名空间，`legacy`，将在没有 sidecar 代理的情况下运行相同的服务：

![我们将要构建的演示应用程序的图形表示。图像显示网格中的两个命名空间和网格外的一个命名空间。每个命名空间都有两个 pod，sleep 和 httpbin。](img/Figure_10.48.jpg)

###### 图 10.48：三个命名空间中的两个在服务网格中

我们将使用以下命令查看命名空间的服务：

1.  首先，我们创建命名空间（`foo`、`bar`和`legacy`），并在这些命名空间中创建`httpbin`和`sleep`服务：

```
kubectl create ns foo
kubectl apply -f <(istioctl kube-inject \
-f httpbin.yaml) -n foo 
kubectl apply -f <(istioctl kube-inject \
-f sleep.yaml) -n foo
kubectl create ns bar
kubectl apply -f <(istioctl kube-inject \
-f httpbin.yaml) -n bar
kubectl apply -f <(istioctl kube-inject \
-f sleep.yaml) -n bar
kubectl create ns legacy
kubectl apply -f httpbin.yaml -n legacy 
kubectl apply -f sleep.yaml -n legacy
```

正如您所看到的，我们现在正在使用`istioctl`工具来注入 sidecar。它会读取我们的 YAML 文件，并在部署中注入 sidecar。现在我们在`foo`和`bar`命名空间中有一个带有 sidecar 的服务。但是在`legacy`命名空间中没有注入。

1.  让我们检查一下是否一切都部署成功了。为了检查这一点，我们提供了一个脚本，它会从每个命名空间到所有其他命名空间建立连接。该脚本将输出每个连接的 HTTP 状态码。![显示脚本如何使用休眠 pod 测试所有连接，并在每个命名空间中与 httpbin 建立连接的图表。](img/Figure_10.49.jpg)

###### 图 10.49：脚本将测试所有连接

1.  使用以下命令运行脚本：

```
bash test_mtls.sh
```

上述命令会迭代所有可达的组合。您应该看到类似以下输出。HTTP 状态码为`200`表示成功：

![bash test_mtls.sh 命令的输出显示 HTTP 状态码为 200，表示所有 pod 都成功，并且证明每个 Pod 都可以与所有其他 Pod 通信。](img/Figure_10.50.jpg)

###### 图 10.50：没有任何策略，我们可以成功地从每个命名空间连接到其他命名空间

这向我们表明，在当前配置中，所有 Pod 都可以与所有其他 Pod 通信。

1.  确保除默认策略外没有现有策略，如下所示：

```
kubectl get policies.authentication.istio.io \
--all-namespaces
kubectl get meshpolicies.authentication.istio.io
```

这应该向您展示：

![验证只有两个策略存在。](img/Figure_10.51.jpg)

###### 图 10.51：只应存在两个策略

1.  另外，确保没有适用的目标规则：

```
kubectl get destinationrules.networking.istio.io \
--all-namespaces -o yaml | grep "host:"
```

在结果中，不应该有带有`foo`、`bar`、`legacy`或通配符（表示为`*`）的主机。

![确保没有适用的目标规则，并且没有带有 foo、bar、legacy 或*通配符的主机。](img/Figure_10.52.jpg)

###### 图 10.52：不应该有带有 foo、bar、legacy 或*通配符的主机

我们已成功部署了示例服务，并且能够确认在默认情况下，所有服务都能够彼此通信。

### 全局启用 mTLS

使用 mTLS 策略，您可以声明所有服务在与其他服务通信时必须使用 mTLS。如果不使用 mTLS，即使恶意用户可以访问集群，即使他们无法访问命名空间，也可以与任何 Pod 通信。如果拥有足够的权限，他们还可以在服务之间充当中间人。在服务之间实施 mTLS 减少了中间人攻击的可能性：

1.  要全局启用双向 TLS，我们将创建以下`MeshPolicy`（在`mtls_policy.yaml`中提供）：

```
1   apiVersion: authentication.istio.io/v1alpha1
2   kind: MeshPolicy
3   metadata:
4     name: default
5   spec:
6     peers:
7     - mtls: {}
```

由于此`MeshPolicy`没有选择器，它将适用于所有工作负载。网格中的所有工作负载只接受使用 TLS 加密的请求。这意味着`MeshPolicy`处理连接的传入部分：

![图形表示显示 MeshPolicy 如何应用于传入连接。图片显示 foo 和 bar 命名空间中的 httpbin 是 MeshPolicy 的一部分。](img/Figure_10.53.jpg)

###### 图 10.53：MeshPolicy 适用于传入连接

您可以使用以下命令创建`MeshPolicy`：

```
kubectl apply -f mtls_policy.yaml
```

#### 注意

我们正在粗暴和激进地应用 mTLS。通常，在生产系统中，您会慢慢引入 mTLS。Istio 有一个特殊的 mTLS 强制模式称为`permissive`，可以帮助实现这一点。在`permissive`模式下使用 mTLS，Istio 将尝试在可能的情况下实现 mTLS，并在不可能的情况下记录警告。但是，流量将继续流动。

1.  现在再次运行脚本以测试网络连接：

```
bash test_mtls.sh
```

1.  带有边车的系统在运行此命令时将失败，并将收到`503`状态代码，因为客户端仍在使用纯文本。可能需要几秒钟才能使`MeshPolicy`生效。*图 10.54*显示输出：![bash test_mtls.sh 命令的输出，显示具有边车的 httpbin.foo 和 httpbin.barPods 的流量失败，状态代码为 503。](img/Figure_10.54.jpg)

###### 图 10.54：带有边车的 Pod 的流量失败，状态代码为 503

如果在先前的输出中更频繁地看到`200`状态代码，请考虑等待几秒钟，然后重新运行测试。

1.  现在，我们将通过将目标规则设置为使用类似于整个网格的身份验证策略的`*`通配符来允许某些流量。这是配置客户端端的必需操作：

```
1   apiVersion: networking.istio.io/v1alpha3
2   kind: DestinationRule
3   metadata:
4     name: default
5     namespace: istio-system
6   spec:
7     host: "*.local"
8     trafficPolicy:
9       tls:
10        mode: ISTIO_MUTUAL
```

让我们看看这个文件：

+   **第 2 行**：在这里，我们正在创建一个`DestinationRule`，它定义了在路由发生后适用于服务流量的策略。

+   **第 7 行**：任何`.local`中的主机的流量（在我们的情况下是集群中的所有流量）应使用此策略。

+   **第 8-10 行**：在这里，我们定义所有流量都需要 mTLS。

`DestinationRule`适用于连接的传出部分，如*图 10.55*所示：

![图形表示显示 DestinationRule 适用于连接的传出部分。图像显示 foo 和 bar 名称空间中的 sleep 是 DestinationRule 的一部分。](img/Figure_10.55.jpg)

###### 图 10.55：DestinationRule 适用于传出流量

我们可以使用以下命令创建这个：

```
kubectl create -f destinationRule.yaml
```

我们可以通过再次运行相同的命令来检查其影响：

```
bash test_mtls.sh
```

这次，返回的代码将如*图 10.56*所示：

![重新运行 bash test_mtls.sh 命令，显示 foo 和 bar 现在可以相互连接，但它们无法再连接到 legacy。](img/Figure_10.56.jpg)

###### 图 10.56：foo 和 bar 现在可以相互连接，但它们无法再连接到 legacy

让我们考虑一下我们在这种情况下实施了什么。我们实施了：

一个需要 Pod 中传入的 TLS 流量的集群范围策略。

一个需要对外流量进行 mTLS 的集群范围 DestinationRule。

这样做的影响是网格内的服务（也称为具有边车）现在可以使用 mTLS 相互通信，并且完全不在网格内的服务（也称为没有边车）也可以相互通信，只是没有 mTLS。由于我们当前的配置，当流的一部分在网格中时，服务之间的通信会中断。

让我们确保清理我们部署的任何资源：

```
istioctl manifest generate --set profile=demo | kubectl delete -f -
for NS in "foo" "bar" "legacy"
do
kubectl delete -f sleep.yaml -n $NS
kubectl delete -f httpbin.yaml -n $NS
done
kubectl delete -f bookinfo.yaml
```

这结束了 Istio 的演示。

## 总结

在本章中，我们专注于 Kubernetes 中的安全性。我们从使用 Azure AD 中的身份查看了集群 RBAC。之后，我们继续讨论在 Kubernetes 中存储秘密。我们详细介绍了创建、解码和使用秘密。最后，我们安装并注入了 Istio，实现了能够设置系统范围策略而无需开发人员干预或监督的目标。由于黑客喜欢攻击易受攻击的系统，您在本章中学到的技能将有助于使您的设置不太可能成为目标。

在接下来的最后一章中，您将学习如何在 Azure Kubernetes Service（AKS）上部署无服务器函数。

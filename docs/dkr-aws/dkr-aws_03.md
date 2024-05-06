# 第三章：开始使用 AWS

在上一章中，我们讨论了部署容器应用程序到 AWS 的各种选项，现在是时候开始使用弹性容器服务（ECS）、Fargate、弹性 Kubernetes 服务（EKS）、弹性 Beanstalk 和 Docker Swarm 来实施实际解决方案了。在我们能够涵盖所有这些令人兴奋的材料之前，您需要建立一个 AWS 账户，了解如何为您的账户设置访问权限，并确保您对我们将在本书中使用的各种工具有牢固的掌握，以与 AWS 进行交互。

开始使用 AWS 非常容易——AWS 提供了一套免费的服务套件，使您能够在 12 个月内免费测试和尝试许多 AWS 服务，或者在某些情况下，无限期地免费使用。当然，会有一些限制，以确保您不能免费设置自己的比特币挖矿服务，但在大多数情况下，您可以利用这些免费套餐服务来测试大量的场景，包括我们将在本书中进行的几乎所有材料。因此，本章将从建立一个新的 AWS 账户开始，这将需要您拥有一张有效的信用卡，以防您真的跟进了那个伟大的新比特币挖矿企业。

一旦您建立了一个账户，下一步是为您的账户设置管理访问权限。默认情况下，所有 AWS 账户都是使用具有最高级别账户特权的根用户创建的，但 AWS 不建议将根账户用于日常管理。因此，我们将配置 AWS 身份访问和管理（IAM）服务，创建 IAM 用户和组，并学习如何使用多因素身份验证（MFA）实施增强安全性。

建立了对 AWS 账户的访问权限后，我们将专注于您可以用来与 AWS 进行交互的各种工具，包括提供基于 Web 的管理界面的 AWS 控制台，以及用于通过命令行与 AWS 进行交互的 AWS CLI 工具。

最后，我们将介绍一种名为 AWS CloudFormation 的管理服务和工具集，它提供了一种基础设施即代码的方法来定义您的 AWS 基础设施和服务。CloudFormation 允许您定义模板，使您能够通过单击按钮构建完整的环境，并且以可重复和一致的方式进行操作。在本书中，我们将广泛使用 CloudFormation，因为在实践中，大多数部署基于 Docker 的应用程序的组织都采用基础设施即代码工具，如 CloudFormation、Ansible 或 Terraform 来自动化其 Docker 应用程序和支持基础设施的部署。您将学习如何创建一个简单的 CloudFormation 模板，然后使用 AWS 控制台和 AWS CLI 部署该模板。

本章将涵盖以下主题：

+   设置 AWS 账户

+   以根账户登录

+   创建 IAM 用户、组和角色

+   创建一个 EC2 密钥对

+   安装 AWS CLI

+   在 AWS CLI 中配置凭据和配置文件

+   使用 AWS CLI 与 AWS 进行交互

+   介绍 AWS CloudFormation

+   定义一个简单的 AWS CloudFormation 模板

+   部署 AWS CloudFormation 堆栈

+   删除 AWS CloudFormation 堆栈

# 技术要求

本章的技术要求如下：

+   根据第一章《容器和 Docker 基础知识》中的说明安装先决条件软件

+   在本章中，需要一个有效的信用卡来创建免费的 AWS 账户

以下 GitHub URL 包含本章中使用的代码示例：[`github.com/docker-in-aws/docker-in-aws/tree/master/ch3`](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch14)[.](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch3)

查看以下视频，了解代码的实际运行情况：

[`bit.ly/2N1nzJc`](http://bit.ly/2N1nzJc)

# 设置 AWS 账户

您 AWS 之旅的第一步是建立一个 AWS 账户，这是 AWS 的基础构建块，为您管理 AWS 服务和资源提供了安全和管理上下文。为了鼓励采用 AWS，并确保首次用户有机会免费尝试 AWS，AWS 提供了一个免费套餐，允许您免费访问一些 AWS 服务（在使用方面有一些限制）。您可以在[`aws.amazon.com/free/`](https://aws.amazon.com/free/)了解更多关于免费套餐和提供的服务。确保您对可以免费使用和不能免费使用有很好的理解，以避免不必要的账单冲击。

在本书中，我们将使用一些免费套餐服务，以下是每月使用限制：

| **服务** | **限制** |
| --- | --- |
| EC2 | 750 小时的 Linux t2.micro（单个 vCPU，1 GB 内存）实例 |
| 弹性块存储 | 30 GB 的块级存储（SSD 或传统旋转磁盘） |
| RDS | 750 小时的 db.t2.micro（单个 vCPU，1 GB 内存）MySQL 实例 |
| 弹性容器注册表 | 500 MB 的存储空间 |
| 弹性负载均衡 | 750 小时的经典或应用负载均衡器 |
| S3 | 5 GB 的 S3 存储空间 |
| Lambda | 1,000,000 次请求 |
| CloudWatch | 10 个自定义指标 |
| SNS | 1,000,000 次发布 |
| CodeBuild | 100 分钟的构建时间 |
| CodePipeline | 1 个活动管道 |
| X-Ray | 100,000 个跟踪 |
| 密钥管理服务 | 20,000 个请求 |
| Secrets Manager | 30 天免费试用期，然后每个秘密/月$0.40 |

正如您所看到的，我们将在本书中涵盖许多 AWS 服务，几乎所有这些服务都是免费的，假设您遵守前表中描述的使用限制。实际上，在本书中我们将使用的唯一一个不免费的服务是 AWS Fargate 服务，所以当您阅读 Fargate 章节时请记住这一点，并尽量减少使用，如果您担心成本。 

要注册免费套餐访问，请点击[`aws.amazon.com/free/`](https://aws.amazon.com/free/)上的**创建免费账户**按钮：

![](img/de4a4e51-dad1-4756-ad13-ce2d6089757a.png)创建免费账户

您将被提示输入电子邮件地址、密码和 AWS 账户名称。重要的是要理解，您在这里输入的电子邮件地址和密码被称为您的 AWS 账户的根账户，这是对您的账户具有最高访问级别的账户。对于 AWS 账户名称，您可以输入任何您喜欢的名称，但它必须在所有其他 AWS 账户中是唯一的，所以至少您将无法使用我选择的账户名称，即`docker-in-aws`。这个账户名称在您登录时使用，比您的 AWS 账户号码更容易记住，后者是一个 12 位数字。

注册过程的其余部分是不言自明的，所以我不会在这里详细说明，但请理解，您将需要提供信用卡详细信息，并将对超出免费使用限制的任何费用负责。您还需要验证注册期间指定的电话号码，这涉及自动电话呼叫到您的号码，因此请确保您在注册期间输入一个有效的电话号码。

# 安装谷歌身份验证器

本节描述的步骤是完全可选的，但是作为安全最佳实践，您应该始终在根账户上启用多因素身份验证（MFA）。事实上，无论所需访问级别如何，您都应该为所有基于用户的 AWS 账户访问启用 MFA。在许多使用 AWS 的组织中，启用 MFA 越来越成为强制性要求，因此在涉及 MFA 时习惯于使用 AWS 是很重要的。因此，我们实际上将在本书中始终使用 MFA。

在您使用 MFA 之前，您需要有一个 MFA 设备，可以是硬件或虚拟 MFA 设备。虚拟 MFA 设备通常安装在您的智能手机上，作为应用程序的形式，完成了您所知道的东西（密码）和您所拥有的东西（您的手机）的多因素范式。

一个流行的 MFA 应用程序可用于 Android 和 iOS 的是谷歌身份验证器应用程序，您可以从谷歌 Play 或苹果应用商店下载。安装应用程序后，您可以继续登录到根账户并设置 MFA 访问。

# 以根账户登录

设置和激活您的账户后，您应该能够登录到 AWS 控制台，您可以在[`console.aws.amazon.com/console/home`](https://console.aws.amazon.com/console/home)访问。

使用根凭据登录后，您应立即启用 MFA 访问。这提供了额外的安全级别，确保如果您的用户名和密码被泄露，攻击者不能在没有您的 MFA 设备（在我们的示例中，这意味着您智能手机上的 Google Authenticator 应用程序）的情况下访问您的帐户。

要为您的根帐户启用 MFA，请选择指定您帐户名称的下拉菜单（在我的情况下，这是“docker-in-aws”），然后选择“我的安全凭据”：

！[](assets/bd93b230-35b0-4d24-b1dd-a148d744fe77.png)访问我的安全凭据

在下一个提示中，点击“继续到安全凭据”按钮，在“您的安全凭据”页面上展开“多因素身份验证（MFA）”选项，然后点击“激活 MFA”按钮：

！[](assets/b554b605-ddd9-4bce-a62c-1c6a6a7be323.png)您的安全凭据屏幕

在“管理 MFA 设备”屏幕上，点击“虚拟 MFA 设备”选项，然后连续点击两次“下一步”，此时您将看到一个 QR 码：

！[](assets/56ca8281-9403-47b8-82c7-5c2b96e94fe3.png)获取 QR 码

您可以使用智能手机上的 Google Authenticator 应用程序扫描此代码，方法是点击添加按钮，选择“扫描条形码”，然后在 AWS 控制台中扫描 QR 码：

！[](assets/6ab37fc0-9b86-4ee4-9c57-b186a02ddb6b.jpg)  ！[](assets/0a255dd8-9ba8-4e15-b55d-2d7e10bfc436.png)！[](assets/5c785b9d-3b72-40db-9f65-dcbb2a5b3343.png)注册 MFA 设备

一旦扫描完成，您需要在“管理 MFA 设备”屏幕上的“身份验证代码 1”输入中输入显示的六位代码。

代码旋转后，将代码的下一个值输入到“身份验证代码 2”输入中，然后点击“激活虚拟 MFA”按钮，以完成 MFA 设备的注册：

！[](assets/e399192a-aa4e-43d0-a977-2e8ba4107479.png)带有 MFA 设备的您的安全凭据

# 创建 IAM 用户、组和角色

在使用 MFA 保护根帐户后，您应立即在您的帐户中创建身份访问和管理（IAM）用户、组和角色以进行日常访问。 IAM 是日常管理和访问 AWS 帐户的推荐方法，您应仅限制根帐户访问计费或紧急情况。在继续之前，您需要知道您的 AWS 帐户 ID，您可以在上一个屏幕截图中看到，在您的 MFA 设备的序列号中（请注意，这将与显示的序列号不同）。记下这个帐户号，因为在配置各种 IAM 资源时将需要它。

# 创建 IAM 角色

创建 IAM 资源的标准做法是创建用户可以承担的*角色*，这将为用户在有限的时间内（通常最多 1 小时）授予提升的特权。最低限度，您需要默认创建一个 IAM 角色：

+   管理员：此角色授予对帐户的完全管理控制，但不包括计费信息

要创建管理员角色，请从 AWS 控制台中选择“服务”|“IAM”，从左侧菜单中选择“角色”，然后单击“创建角色”按钮。在“选择受信任的实体类型”屏幕中，选择“另一个 AWS 帐户”选项，并在“帐户 ID”字段中配置您的帐户 ID：

选择受信任的实体作为管理员角色

单击“下一步：权限”按钮后，选择“AdministratorAccess”策略，该策略授予角色管理访问权限：

将策略附加到 IAM 角色

最后，指定一个名为“admin”的角色名称，然后单击“创建角色”以完成管理员角色的创建：

创建 IAM 角色

这将创建管理员 IAM 角色。如果单击新创建的角色，请注意角色的角色 ARN（Amazon 资源名称），因为您以后会需要这个值：

管理员角色

# 创建管理员组

有了管理角色之后，下一步是将您的角色分配给用户或组。与其直接为用户分配权限，强烈建议改为将其分配给组，因为这提供了一种更可扩展的权限管理方式。鉴于我们已经创建了具有管理权限的角色，现在创建一个名为管理员的组是有意义的，该组将被授予*假定*您刚刚创建的 admin 角色的权限。请注意，我指的是假定一个角色，这类似于 Linux 和 Unix 系统，在那里您以普通用户身份登录，然后使用`sudo`命令临时假定根权限。

您将在本章后面学习如何假定一个角色，但现在您需要通过在 IAM 控制台的左侧菜单中选择**组**并单击**创建新组**按钮来创建管理员组。

![](img/02d4df30-8fb3-4a62-af2f-7770c0acddb4.png)创建 IAM 组

您首先需要指定一个名为管理员的**组名称**，然后单击**下一步**两次以跳过**附加策略**屏幕，最后单击**创建组**以完成组的创建：

![](img/bd6acf11-5675-4b42-aeec-0a51e1c66515.png)管理员组

这创建了一个没有附加权限的组，但是如果您单击该组并选择**权限**，现在您有创建内联策略的选项：

![](img/b3727916-33e9-467c-9529-620aba3b7bbe.png)创建内联策略

在上述截图中选择点击此处链接后，选择**自定义策略**选项并单击选择，这将允许您配置一个 IAM 策略文档，以授予假定您之前创建的`admin`角色的能力：

管理员组内联策略

该策略包括一个允许执行`sts:AssumeRole`操作的声明 - 这里的`sts`指的是安全令牌服务，这是您在假定角色时与之交互的服务（假定角色的操作会授予您与所假定角色相关联的临时会话凭证）。请注意，资源是您创建的 IAM 角色的 ARN，因此该策略允许任何属于**管理员**组的成员假定**admin**角色。单击**应用策略**按钮后，您将成功创建和配置**管理员**组。

# 创建一个用户组

我通常建议创建的另一个组是用户组，每个访问您的 AWS 账户的人类用户都应该属于该组，包括您的管理员（他们也将成为管理员组的成员）。用户组的核心功能是确保除了一小部分权限外，用户组的任何成员执行的所有操作都必须经过 MFA 身份验证，而不管通过其他组可能授予该用户的权限。这本质上是一个强制 MFA 策略，您可以在[`www.trek10.com/blog/improving-the-aws-force-mfa-policy-for-IAM-users/`](https://www.trek10.com/blog/improving-the-aws-force-mfa-policy-for-IAM-users/)上阅读更多相关信息，并且实施这种方法可以增加您为访问 AWS 账户设置的整体安全保护。请注意，该策略允许用户执行一小部分操作而无需 MFA，包括登录、更改用户密码，以及最重要的是允许用户注册 MFA 设备。这允许新用户使用临时密码登录，更改密码，并自行注册 MFA 设备，一旦用户注销并使用 MFA 重新登录，策略允许用户创建用于 API 和 CLI 访问的 AWS 访问密钥。

要实施用户组，我们首先需要创建一个托管 IAM 策略，与我们在前面的截图中采用的内联方法相比，这是一种更可扩展和可重用的机制，用于将策略分配给组和角色。要创建新的托管策略，请从右侧菜单中选择**策略**，然后单击**创建策略**按钮，这将打开**创建策略**屏幕。您需要创建的策略非常广泛，并且在 GitHub 的要点中发布，网址为[`bit.ly/2KfNfAz`](https://bit.ly/2KfNfAz)，该策略基于先前引用的博客文章中讨论的策略，添加了一些额外的安全增强功能。

请注意，要点包括在策略文件中包含一个名为`PASTE_ACCOUNT_NUMBER`的占位符，因此您需要将其替换为您的实际 AWS 账户 ID：

![](img/f65c5220-257c-44e3-be1b-0aa1c9b60e26.png)创建一个 IAM 托管策略

点击**Review policy**按钮后，您需要为策略配置一个名称，我们将其称为`RequireMFAPolicy`，然后点击**Create policy**创建策略，您需要按照本章前面创建 Administrators 组时的相同说明创建一个 Users 组。

当您在创建 Users 组时到达**Attach Policy**屏幕时，您可以输入刚刚创建的 RequireMFAPolicy 托管策略的前几个字母，然后将其附加到组中。

![](img/a9c05282-0d04-4a82-8e86-63463f523563.png)将 RequireMFAPolicy 附加到 Users 组

完成创建**Users**组的向导后，您现在应该在 IAM 控制台中拥有一个**Administrators**组和**Users**组。

# 创建 IAM 用户

您需要执行的最后一个 IAM 设置任务是创建一个 IAM 用户来管理您的帐户。正如本章前面讨论的那样，您不应该使用根凭证进行日常管理任务，而是创建一个管理 IAM 用户。

要创建用户，请从 IAM 控制台的右侧菜单中选择**Users**，然后点击**Add user**按钮。在**Add user**屏幕上，指定一个**User name**，并且只选择**AWS Management Console access**作为**Access type**，确保**Console password**设置为**Autogenerated password**，并且**Require password reset**选项已设置：

![](img/6e4cf11a-4fb8-4226-adcc-a0f4d9cb667b.png)创建新用户

点击**Next: Permissions**按钮后，将用户添加到您之前创建的**Administrators**和**Users**组中：

![](img/f9dab5ca-1eb6-4942-a07a-d8acd12828aa.png)将用户添加到组中

现在，您可以点击**Next: review**和**Create user**按钮来创建用户。用户将被创建，因为您选择创建了自动生成的密码，您可以点击**Password**字段中的**Show**链接来显示用户的初始密码。请注意这个值，因为您将需要它来测试作为刚刚创建的 IAM 用户登录：

![](img/8b50427a-f35e-4772-a692-8e0664aebdf8.png)新创建的用户临时密码

# 作为 IAM 用户登录

现在您已经创建了 IAM 用户，您可以通过单击菜单中的帐户别名/ID 并选择**注销**来测试用户的首次登录体验。如果您现在单击**登录到控制台**按钮或浏览到[`console.aws.amazon.com/console/home`](https://console.aws.amazon.com/console/home)，选择**登录到其他帐户**选项，输入您的帐户别名或帐户 ID，然后单击**下一步**，然后输入刚刚创建的 IAM 用户的用户名和临时密码：

![](img/75e79bbf-c290-4a2d-a106-6f13f32561bb.png)首次以 IAM 用户身份登录

然后会提示您输入新密码：

![](img/7deaf251-e404-41d0-961d-de7d00b8e18e.png)输入新密码

确认密码更改后，您将成功以新用户身份登录。

# 为 IAM 用户启用 MFA

在这一点上，您已经首次使用 IAM 用户登录，接下来需要执行的步骤是为新用户注册 MFA 设备。要做到这一点，选择**服务** | **IAM** 打开 IAM 控制台，从左侧菜单中选择**用户**，然后点击您的 IAM 用户。

在**安全凭证**选项卡中，单击**分配的 MFA 设备**字段旁边的铅笔图标：

![](img/aebab6f6-7c8a-4cd4-9671-dccd26b7dc3e.png)IAM 用户安全凭证

管理 MFA 设备对话框将弹出，允许您注册新的 MFA 设备。这个过程与本章前面为根帐户设置 MFA 的过程相同，因此我不会重复说明这个过程，但是一旦您注册了 MFA 设备，重要的是您登出并重新登录到控制台以强制进行 MFA 身份验证。

如果您已经正确配置了一切，当您再次登录到控制台时，应该会提示您输入 MFA 代码：

![](img/48de9696-0af0-40f0-bded-e298bd27a327.png)MFA 提示

# 假设 IAM 角色

一旦您完成了注册 MFA 设备并使用 MFA 登出并重新登录到 AWS 控制台，您现在满足了导致您之前创建的`RequireMFAPolicy`中的以下语句不被应用的要求：

```
{
    "Sid": "DenyEverythingExceptForBelowUnlessMFAd",
    "Effect": "Deny",
    "NotAction": [
        "iam:ListVirtualMFADevices",
        "iam:ListMFADevices",
        "iam:ListUsers",
        "iam:ListAccountAliases",
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:ResyncMFADevice",
        "iam:ChangePassword",
        "iam:CreateLoginProfile",
        "iam:DeleteLoginProfile",
        "iam:GetAccountPasswordPolicy",
        "iam:GetAccountSummary",
        "iam:GetLoginProfile",
        "iam:UpdateLoginProfile"
    ],
    "Resource": "*",
    "Condition": {
        "Null": {
            "aws:MultiFactorAuthAge": "true"
        }
    }
}
```

在上述代码中，重要的是要注意`Deny`的 IAM 效果是绝对的——一旦 IAM 遇到给定权限或一组权限的`Deny`，那么该权限就无法被允许。然而，`Condition`属性使这个广泛的`Deny`有条件——只有在特殊条件`aws:MultiFactorAuthAge`为 false 的情况下才会应用，这种情况发生在您没有使用 MFA 登录时。

假设 IAM 用户已通过 MFA 登录，并附加到具有承担**管理员**角色权限的**Administrators**组，那么`RequireMFAPolicy`中没有任何内容会拒绝此操作，因此您现在应该能够承担**管理员**角色。

要使用 AWS 控制台承担管理员角色，请点击下拉菜单，选择**切换角色**：

![](img/7579bba8-fdb1-4462-a458-03ea1392ffe7.png)切换角色

点击**切换角色**按钮后，您将被提示输入帐户 ID 或名称，以及您想要在配置帐户中承担的角色：

![](img/b934817f-04fb-44ab-8a4b-ba8ad803e9a3.png)切换角色

您现在应该注意到 AWS 控制台的标题指示您必须承担管理员角色，现在您已经完全具有对 AWS 帐户的管理访问权限。

![](img/582a79e0-cc9d-45da-ab8f-8e273dcb2a0a.png)承担管理员角色在本书的其余部分中，每当您需要在您的帐户中执行管理任务时，我将假定您已经承担了管理员角色，就像在之前的屏幕截图中演示的那样。

# 创建 EC2 密钥对

如果您打算在 AWS 帐户中运行任何 EC2 实例，那么需要完成的一个关键设置任务就是建立一个或多个 EC2 密钥对，对于 Linux EC2 实例，可以用来定义一个 SSH 密钥对，以授予对 EC2 实例的 SSH 访问。

当您创建 EC2 密钥对时，将自动生成一个 SSH 公钥/私钥对，其中 SSH 公钥将作为命名的 EC2 密钥对存储在 AWS 中，并相应的 SSH 私钥下载到您的本地客户端。如果随后创建任何 EC2 实例并在实例创建时引用命名的 EC2 密钥对，您将能够自动使用相关的 SSH 私钥访问您的 EC2 实例。

访问 Linux EC2 实例的 SSH 需要您使用与配置的 EC2 密钥对关联的 SSH 私钥，并且还需要适当的网络配置和安全组，以允许从您的 SSH 客户端所在的任何位置访问 EC2 实例的 SSH 端口。

要创建 EC2 密钥对，首先在 AWS 控制台中导航到**服务| EC2**，从左侧菜单中的**网络和安全**部分中选择**密钥对**，然后单击**创建密钥对**按钮：

![](img/52541551-506e-42ce-9fdf-02cfa8b5a6f9.png)

在这里，您已配置了一个名为 admin 的 EC2 密钥对名称，并在单击“创建”按钮后，将创建一个新的 EC2 密钥对，并将 SSH 私钥下载到您的计算机：

![](img/b793d0f0-fcb8-4f02-9283-228dcf9a540b.png)

此时，您需要将 SSH 私钥移动到计算机上的适当位置，并按下面的示例修改私钥文件的默认权限：

```
> mv ~/Downloads/admin.pem ~/.ssh/admin.pem
> chmod 600 ~/.ssh/admin.pem
```

请注意，如果您不使用 chmod 命令修改权限，当您尝试使用 SSH 密钥时，将会出现以下错误：

```
> ssh -i ~/.ssh/admin.pem 192.0.2.1
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@ WARNING: UNPROTECTED PRIVATE KEY FILE! @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for '/Users/jmenga/.ssh/admin.pem' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "/Users/jmenga/.ssh/admin.pem": bad permissions
```

# 使用 AWS CLI

到目前为止，在本章中，您只与 AWS 控制台进行了交互，该控制台可以从您的 Web 浏览器访问。虽然拥有 AWS 控制台访问权限非常有用，但在许多情况下，您可能更喜欢使用命令行工具，特别是在需要自动化关键操作和部署任务的情况下。

# 安装 AWS CLI

AWS CLI 是用 Python 编写的，因此您必须安装 Python 2 或 Python 3，以及 PIP Python 软件包管理器。

本书中使用的说明和示例假定您使用的是 MacOS 或 Linux 环境。

有关如何在 Windows 上设置 AWS CLI 的说明，请参阅[`docs.aws.amazon.com/cli/latest/userguide/awscli-install-windows.html`](https://docs.aws.amazon.com/cli/latest/userguide/awscli-install-windows.html)。

假设您已满足这些先决条件，您可以在终端中使用`pip`命令安装 AWS CLI，并使用`--upgrade`标志升级到最新的 AWS CLI 版本（如果已安装），并使用`--user`标志避免修改系统库：

```
> pip install awscli --upgrade --user
Collecting awscli
  Downloading https://files.pythonhosted.org/packages/69/18/d0c904221d14c45098da04de5e5b74a6effffb90c2b002bc2051fd59222e/awscli-1.15.45-py2.py3-none-any.whl (1.3MB)
    100% |████████████████████████████████| 1.3MB 1.2MB/s
...
...
Successfully installed awscli-1.15.45 botocore-1.10.45 colorama-0.3.9 pyasn1-0.4.3 python-dateutil-2.7.3
```

根据您的环境，如果您使用的是 Python 3，您可能需要用`pip3 install`命令替换`pip install`。

如果您现在尝试运行 AWS CLI 命令，该命令将失败，并指示您必须配置您的环境：

```
> aws ec2 describe-vpcs
You must specify a region. You can also configure your region by running "aws configure".
```

# 创建 AWS 访问密钥

如果您按照前面的代码建议运行`aws configure`命令，将提示您输入 AWS 访问密钥 ID：

```
> aws configure
AWS Access Key ID [None]:
```

要使用 AWS CLI 和 AWS SDK，您必须创建 AWS 访问密钥，这是由访问密钥 ID 和秘密访问密钥值组成的凭据。要创建访问密钥，请在 AWS 控制台中打开 IAM 仪表板，从左侧菜单中选择**用户**，然后单击您的用户名。在**安全凭据**选项卡下的**访问密钥**部分，单击**创建访问密钥**按钮，这将打开一个对话框，允许您查看访问密钥 ID 和秘密访问密钥值：

![](img/fcc8616b-3a14-43ac-8b78-b7449ca7fc29.png)访问密钥凭证

记下访问密钥 ID 和秘密访问密钥值，因为您将需要这些值来配置您的本地环境。

# 配置 AWS CLI

回到您的终端，现在您可以完成`aws configure`设置过程：

```
> aws configure
AWS Access Key ID [None]: AKIAJXNI5XLCSBRQAZCA
AWS Secret Access Key [None]: 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
Default region name [None]: us-east-1
Default output format [None]:
```

如果您现在尝试运行之前尝试过的`aws ec2 describe-vpcs`命令，该命令仍然失败；但是，错误是不同的：

```
> aws ec2 describe-vpcs

An error occurred (UnauthorizedOperation) when calling the DescribeVpcs operation: You are not authorized to perform this operation.
```

现在的问题是，您未被授权执行此命令，因为您刚刚创建的访问密钥与您的用户帐户相关联，您必须假定管理员角色以获得管理特权。

# 配置 AWS CLI 以假定角色

此时，AWS CLI 正在以您的用户帐户的上下文中运行，您需要配置 CLI 以假定管理员角色以能够执行任何有用的操作。

当您运行`aws configure`命令时，AWS CLI 在名为`.aws`的文件夹中创建了两个重要文件，该文件夹位于您的主目录中：

```
> ls -l ~/.aws

total 16
-rw------- 1 jmenga staff 29  23 Jun 19:31 config
-rw------- 1 jmenga staff 116 23 Jun 19:31 credentials
```

`credentials`文件保存了一个或多个命名配置文件中的 AWS 凭据：

```
> cat ~/.aws/credentials
[default]
aws_access_key_id = AKIAJXNI5XLCSBRQAZCA
aws_secret_access_key = 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
```

在上述代码中，请注意`aws configure`命令创建了一个名为`default`的配置文件，并将访问密钥 ID 和秘密访问密钥值存储在该文件中。作为最佳实践，特别是如果您正在使用多个 AWS 账户，我建议避免使用默认配置文件，因为如果输入 AWS CLI 命令，AWS CLI 将默认使用此配置文件。您很快将学会如何使用命名配置文件来处理多个 AWS 账户，如果您有一个默认配置文件，很容易忘记指定要使用的配置文件，并在默认配置文件引用的账户中意外执行操作。我更喜欢根据您正在使用的账户的名称命名每个配置文件，例如，在这里，我已将凭据文件中的默认配置文件重命名为`docker-in-aws`，因为我将我的 AWS 账户命名为`docker-in-aws`：

```
[docker-in-aws]
aws_access_key_id = AKIAJXNI5XLCSBRQAZCA
aws_secret_access_key = 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
```

AWS CLI 创建的另一个文件是`~/.aws/config`文件，如下所示：

```
[default]
region = us-east-1
```

该文件包括命名的配置文件，并且因为您在运行`aws configure`命令时指定了默认区域，所以`default`配置文件中已经添加了`region`变量。配置文件支持许多变量，允许您执行更高级的任务，比如自动假定角色，因此这就是我们需要配置 CLI 以假定我们在本章前面创建的`admin`角色的地方。鉴于我们已经在`credentials`文件中重命名了`default`配置文件，以下代码演示了将`default`配置文件重命名为`docker-in-aws`并添加支持假定`admin`角色的操作：

```
[profile docker-in-aws]
source_profile = docker-in-aws
role_arn = arn:aws:iam::385605022855:role/admin
role_session_name=justin.menga
mfa_serial = arn:aws:iam::385605022855:mfa/justin.menga
region = us-east-1
```

请注意，在配置命名配置文件时，我们在配置文件名前面添加了`profile`关键字，这是必需的。我们还在配置文件中配置了许多变量：

+   `source_profile`：这是应该用于获取凭据的凭据配置文件。我们指定`docker-in-aws`，因为我们之前已将凭据文件中的配置文件重命名为`docker-in-aws`。

+   `role_arn`：这是要假定的 IAM 角色的 ARN。在这里，您指定了您在上一个截图中创建的`admin`角色的 ARN。

+   `role_session_name`：这是在承担配置的角色时创建的临时会话的名称。作为最佳实践，您应该指定您的 IAM 用户名，因为这有助于审计您使用角色执行的任何操作。当您使用承担的角色在 AWS 中执行操作时，您的身份实际上是`arn:aws:sts::<account-id>:assumed-role/<role-name>/<role-session-name>`，因此将用户名设置为角色会话名称可以确保可以轻松确定执行操作的用户。

+   `mfa_serial`：这是应该用于承担角色的 MFA 设备的 ARN。鉴于您的 IAM 用户属于用户组，对于所有操作，包括通过 AWS CLI 或 SDK 进行的任何 API 调用，都需要 MFA。通过配置此变量，AWS CLI 将在尝试承担配置的角色之前自动提示您输入 MFA 代码。您可以在 IAM 用户帐户的安全凭据选项卡中获取 MFA 设备的 ARN（请参阅分配的 MFA 设备字段，但它将始终遵循`arn:aws:iam::<account-id>:mfa/<user-id>`的命名约定）。

有关支持凭据和配置文件中的所有变量的完整描述，请参阅[`docs.aws.amazon.com/cli/latest/topic/config-vars.html`](https://docs.aws.amazon.com/cli/latest/topic/config-vars.html)。

# 配置 AWS CLI 以使用命名配置文件

有了配置，您就不再有默认配置文件，因此运行 AWS CLI 将返回相同的输出。要使用命名配置文件，您有两个选项：

+   在 AWS CLI 命令中使用`--profile`标志指定配置文件名称。

+   在名为`AWS_PROFILE`的环境变量中指定配置文件名称。这是我首选的机制，我将假设您在本书中一直采用这种方法。

上面的代码演示了使用这两种方法：

```
> aws ec2 describe-vpcs --profile docker-in-aws
Enter MFA code for arn:aws:iam::385605022855:mfa/justin.menga: ****
{
    "Vpcs": [
        {
            "VpcId": "vpc-f8233a80",
            "InstanceTenancy": "default",
            "CidrBlockAssociationSet": [
                {
                    "AssociationId": "vpc-cidr-assoc-32524958",
                    "CidrBlock": "172.31.0.0/16",
                    "CidrBlockState": {
                        "State": "associated"
                    }
                }
            ],
            "State": "available",
            "DhcpOptionsId": "dopt-a037f9d8",
            "CidrBlock": "172.31.0.0/16",
            "IsDefault": true
        }
    ]
}
> export AWS_PROFILE=docker-in-aws
> aws ec2 describe-vpcs --query Vpcs[].VpcId
[
    "vpc-f8233a80"
]
```

在上面的示例中，请注意当您首次运行`aws`命令时，会提示您输入 MFA 令牌，但是当您下次运行该命令时，将不会提示您。这是因为默认情况下，从承担角色获取的临时会话凭据在一个小时内有效，并且 AWS CLI 会缓存凭据，以便您在不必在每次执行命令时刷新凭据的情况下重用它们。当然，在一个小时后，由于临时会话凭据将会过期，您将再次被提示输入 MFA 令牌。

在前面的代码中，还有一个有趣的地方需要注意，就是在最后一个命令示例中使用了`--query`标志。这允许您指定一个 JMESPath 查询，这是一种用于查询 JSON 数据结构的查询语言。AWS CLI 默认输出 JSON，因此您可以使用查询从 AWS CLI 输出中提取特定信息。在本书中，我将经常使用这些查询的示例，您可以在[`jmespath.org/tutorial.html`](http://jmespath.org/tutorial.html)上阅读更多关于 JMESPath 查询语言的信息。

# AWS CloudFormation 简介

**AWS CloudFormation**是一项托管的 AWS 服务，允许您使用基础架构即代码来定义 AWS 服务和资源，并且是使用 AWS 控制台、CLI 或各种 SDK 部署 AWS 基础架构的替代方案。虽然需要一些学习曲线来掌握 CloudFormation，但一旦掌握了使用 CloudFormation 的基础知识，它就代表了一种非常强大的部署 AWS 基础架构的方法，特别是一旦开始部署复杂的环境。

在使用 CloudFormation 时，您可以在 CloudFormation 模板中定义一个或多个资源，这是一种将相关资源组合在一个地方的便捷机制。当您部署模板时，CloudFormation 将创建一个包含在模板中定义的物理资源的*堆栈*。CloudFormation 将部署每个资源，自动确定每个资源之间的任何依赖关系，并优化部署，以便在适用的情况下可以并行部署资源，或者在资源之间存在依赖关系时按正确的顺序部署资源。最好的消息是，所有这些强大的功能都是免费的 - 您只需要在通过 CloudFormation 部署堆栈时支付您消耗的资源。

需要注意的是，有许多第三方替代方案可以替代 CloudFormation - 例如，Terraform 非常受欢迎，传统的配置管理工具如 Ansible 和 Puppet 也包括部署 AWS 资源的支持。我个人最喜欢的是 CloudFormation，因为它得到了 AWS 的原生支持，对各种 AWS 服务和资源有很好的支持，并且与 AWS CLI 和 CodePipeline 等服务进行了原生集成（我们将在本书的第十三章“持续交付 ECS 应用程序”中利用这种集成）。

# 定义 CloudFormation 模板

使用 CloudFormation 的最简单方法是创建一个 CloudFormation 模板。该模板以 JSON 或 YAML 格式定义，我建议使用 YAML 格式，因为相比 JSON，YAML 更容易让人类操作。

[CloudFormation 用户指南](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)详细描述了[模板结构](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-anatomy.html)，但是出于本书的目的，我们只需要关注一个基本的模板结构，最好通过一个真实的例子来演示，您可以将其保存在计算机上一个方便的位置的名为`stack.yml`的文件中。

```
AWSTemplateFormatVersion: "2010-09-09"

Description: Cloud9 Management Station

Parameters:
 EC2InstanceType:
   Type: String
   Description: EC2 instance type
   Default: t2.micro
 SubnetId:
   Type: AWS::EC2::Subnet::Id
   Description: Target subnet for instance

Resources:
  ManagementStation:
    Type: AWS::Cloud9::EnvironmentEC2
    Properties:
      Name: !Sub ${AWS::StackName}-station
      Description:
        Fn::Sub: ${AWS::StackName} Station
      AutomaticStopTimeMinutes: 15
```

```
      InstanceType: !Ref EC2InstanceType
      SubnetId:
        Ref: SubnetId
```

在上述代码中，CloudFormation 定义了一个 Cloud9 管理站 - Cloud9 提供基于云的 IDE 和终端，在 EC2 实例上运行。让我们通过这个例子来讨论模板的结构和特性。

`AWSTemplateFormatVersion`属性是必需的，它指定了 CloudFormation 模板的格式版本，通常以日期形式表示。`Parameters`属性定义了一组输入参数，您可以将这些参数提供给您的模板，这是处理多个环境的好方法，因为您可能在每个环境之间有不同的输入值。例如，`EC2InstanceType`参数指定了管理站的 EC2 实例类型，而`SubnetId`参数指定了 EC2 实例应连接到的子网。这两个值在非生产环境和生产环境之间可能不同，因此将它们作为输入参数使得根据目标环境更容易更改。请注意，`SubnetId`参数指定了`AWS::EC2::Subnet::Id`类型，这意味着 CloudFormation 可以使用它来查找或验证输入值。有关支持的参数类型列表，请参见[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html)。您还可以看到`EC2InstanceType`参数为参数定义了默认值，如果没有为此参数提供输入，则将使用该默认值。

`Resources`属性定义了堆栈中的所有资源 - 这实际上是模板的主体部分，可能包含多达两百个资源。在上面的代码中，我们只定义了一个名为`ManagementStation`的资源，这将创建 Cloud9 EC2 环境，其`Type`值为`AWS::Cloud9::EnvironmentEC2`。所有资源必须指定一个`Type`属性，该属性定义了资源的类型，并确定了每种类型可用的各种配置属性。CloudFormation 用户指南包括一个定义了所有支持的资源类型的部分，截至最后一次统计，有 300 种不同类型的资源。

每个资源还包括一个 Properties 属性，其中包含资源可用的所有各种配置属性。在上面的代码中，您可以看到我们定义了五个不同的属性 - 可用的属性将根据资源类型而变化，并在 CloudFormation 用户指南中得到充分的文档记录。

+   `名称`：这指定了 Cloud9 EC2 环境的名称。属性的值可以是简单的标量值，比如字符串或数字，但是值也可以引用模板中的其他参数或资源。请注意，`Name`属性的值包括所谓的内置函数`Sub`，可以通过前面的感叹号(`!Sub`)来识别。`!Sub`语法实际上是`Fn::Sub`的简写，你可以在`Description`属性中看到一个例子。`Fn::Sub`内置函数允许您定义一个表达式，其中包括对堆栈中其他资源或参数的插值引用。例如，`Name`属性的值是`${AWS::StackName}-station`，其中`${AWS::StackName}`是一个称为[伪参数](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/pseudo-parameter-reference.html)的插值引用，它将被模板部署时的 CloudFormation 堆栈的名称替换。如果您的堆栈名称是`cloud9-management`，那么`${AWS::StackName}-station`的值在部署堆栈时将扩展为`cloud9-management-station`。

+   `Description`: 这为 Cloud9 EC2 环境提供了描述。这包括`Fn::Sub`内部函数的长格式示例，该函数要求您缩进一个新行，而简写的`!Sub`格式允许您在同一行上指定值。

+   `AutomaticStopTime`: 这定义了在停止 Cloud9 EC2 实例之前等待的空闲时间，单位为分钟。这可以节省成本，但只有在您使用 EC2 实例时才会运行（Cloud9 将自动启动您的实例，并从您之前停止的地方恢复会话）。在上面的代码中，该值是一个简单的标量值为 15。

+   `InstanceType`: 这是 EC2 实例的类型。这引用了`EC2InstanceType`参数，使用了 Ref 内部函数（`!Ref`是简写形式），允许您引用堆栈中的其他参数或资源。这意味着在部署堆栈时为此参数提供的任何值都将应用于`InstanceType`属性。

+   `SubnetId`: 这是 EC2 实例将部署的目标子网 ID。此属性引用了 SubnetID 参数，使用了`Ref`内部函数的长格式，这要求您在缩进的新行上表达此引用。

# 部署 CloudFormation 堆栈

现在您已经定义了一个 CloudFormation 模板，可以以 CloudFormation 堆栈的形式部署模板中的资源。

您可以通过选择**服务** | **CloudFormation**在 AWS 控制台上部署堆栈，这将打开 CloudFormation 仪表板。在继续之前，请确保您已经在您的帐户中扮演了管理员角色，并且还选择了美国东部北弗吉尼亚（us-east-1）作为地区：

在本书的所有示例中，我们将使用美国东部北弗吉尼亚（us-east-1）地区。![](img/6aec5bab-afe5-4795-ac18-1c6681e28289.png)CloudFormation 仪表板

如果单击**创建新堆栈**按钮，将提示您选择模板，您可以选择示例模板、上传模板或指定 S3 模板 URL。因为我们在名为`stack.yml`的文件中定义了我们的堆栈，所以选择上传模板的选项，并单击**选择文件**按钮选择计算机上的文件：

![](img/38367206-84db-4a54-8043-fe8aa6613cef.png)选择 CloudFormation 模板

上传模板后，CloudFormation 服务将解析模板并要求您为堆栈指定名称，并为堆栈中的任何参数提供值：

![](img/65e6ca0f-052d-4840-96b2-503e43d49863.png)指定模板详细信息

在上述截图中，默认情况下为`EC2InstanceType`参数设置了值`t2.micro`，因为您在模板中将其设置为默认值。由于您将`AWS::EC2::Subnet::Id`指定为`SubnetId`参数的类型，**创建堆栈**向导会自动查找您帐户和区域中的所有子网，并在下拉菜单中呈现它们。在这里，我选择了位于**us-east-1a**可用区的每个新 AWS 帐户中创建的默认 VPC 中的子网。

您可以通过在 AWS 控制台中选择**服务**|**VPC**|**子网**，或者通过运行带有 JMESPath 查询的`aws ec2 describe-subnets` AWS CLI 命令来确定每个子网属于哪个可用区：

```
> aws ec2 describe-subnets --query 'Subnets[].[SubnetId,AvailabilityZone,CidrBlock]' \
    --output table
-----------------------------------------------------
| DescribeSubnets                                   |
+-----------------+--------------+------------------+
| subnet-a5d3ecee | us-east-1a   | 172.31.16.0/20   |
| subnet-c2abdded | us-east-1d   | 172.31.80.0/20   |
| subnet-aae11aa5 | us-east-1f   | 172.31.48.0/20   |
| subnet-fd3a43c2 | us-east-1e   | 172.31.64.0/20   |
| subnet-324e246f | us-east-1b   | 172.31.32.0/20   |
| subnet-d281a2b6 | us-east-1c   | 172.31.0.0/20    |
+-----------------+--------------+------------------+
```

此时，您可以单击**下一步**，然后在**创建堆栈**向导中单击**创建**，以开始部署新堆栈。在 CloudFormation 仪表板中，您将看到创建了一个名为**cloud9-management**的新堆栈，最初状态为`CREATE_IN_PROGRESS`。通过 CloudFormation 部署 Cloud9 环境的一个有趣行为是，通过`AWS::Cloud9::Environment`资源会自动创建一个单独的子 CloudFormation 堆栈，这在部署其他类型的 CloudFormation 资源时是不太常见的。部署完成后，堆栈的状态将变为`CREATE_COMPLETE`：

![](img/de0f7e1f-35bc-47aa-8df0-7bebfa06bb1f.png)部署 CloudFormation 堆栈

在上述截图中，您可以单击**事件**选项卡以显示与堆栈部署相关的事件。这将显示每个资源部署的进度，并指示是否存在任何失败。

现在您已成功部署了第一个 CloudFormation 堆栈，您应该可以使用全新的 Cloud9 IDE 环境。如果您在 AWS 控制台菜单栏中选择**服务**|**Cloud9**，您应该会看到一个名为`cloud9-management-station`的单个环境：

![](img/bf94b2a5-6146-44ae-a0ed-3de38549d0ea.png)Cloud9 环境

如果单击**打开 IDE**按钮，这将打开一个包含安装了 AWS CLI 的集成终端的新 IDE 会话。请注意，会话具有创建 Cloud9 环境的用户关联的所有权限 - 在本例中，这是假定的**admin**角色，因此您可以从终端执行任何管理任务。Cloud9 环境也在您的 VPC 中运行，因此，如果您部署其他资源（如 EC2 实例），即使您的其他资源部署在没有互联网连接的私有子网中，您也可以从此环境本地管理它们。

确保您了解创建具有完全管理特权的 Cloud9 环境的影响。尽管这非常方便，但它确实代表了一个潜在的安全后门，可能被用来破坏您的环境和帐户。Cloud9 还允许您与其他用户共享您的 IDE，这可能允许其他用户冒充您并执行您被允许执行的任何操作。 ![](img/47bf21c4-dc4c-45a7-800a-25ed9098cff3.png)Cloud9 IDE

# 更新 CloudFormation 堆栈

创建 CloudFormation 堆栈后，您可能希望对堆栈进行更改，例如添加其他资源或更改现有资源的配置。 CloudFormation 定义了与堆栈相关的三个关键生命周期事件 - CREATE，UPDATE 和 DELETE - 这些事件可以应用于堆栈中的单个资源，也可以应用于整个堆栈。

要更新堆栈，只需对 CloudFormation 模板进行任何必要的更改，并提交修改后的模板 - CloudFormation 服务将计算每个资源所需的更改，这可能导致创建新资源，更新或替换现有资源，或删除现有资源。 CloudFormation 还将首先进行任何新更改，仅当这些更改成功时，它才会清理应该被移除的任何资源。这提供了在 CloudFormation 堆栈更新失败的情况下恢复的更高机会，在这种情况下，CloudFormation 将尝试回滚更改以将堆栈恢复到其原始状态。

要测试更新您的 CloudFormation 堆栈，让我们对`stack.yml`模板进行小的更改：

```
AWSTemplateFormatVersion: "2010-09-09"

Description: Cloud9 Management Station

Parameters:
  EC2InstanceType:
    Type: String
    Description: EC2 instance type
    Default: t2.micro
  SubnetId:
    Type: AWS::EC2::Subnet::Id
    Description: Target subnet for instance

```

```
Resources:
  ManagementStation:
    Type: AWS::Cloud9::EnvironmentEC2
    Properties:
      Name: !Sub ${AWS::StackName}-station
      Description:
        Fn::Sub: ${AWS::StackName} Station
 AutomaticStopTimeMinutes: 20
      InstanceType: !Ref EC2InstanceType
      SubnetId:
        Ref: SubnetId
```

应用此更改，我们将使用 AWS CLI 而不是使用 AWS 控制台，AWS CLI 支持通过`aws cloudformation deploy`命令部署 CloudFormation 模板。我们将在本书的其余部分大量使用此命令，现在是介绍该命令的好时机：

```
> export AWS_PROFILE=docker-in-aws
> aws cloudformation deploy --stack-name cloud9-management --template-file stack.yml \
--parameter-overrides SubnetId=subnet-a5d3ecee
Enter MFA code for arn:aws:iam::385605022855:mfa/justin.menga: ****

Waiting for changeset to be created..
Waiting for stack create/update to complete

Failed to create/update the stack. Run the following command
to fetch the list of events leading up to the failure
aws cloudformation describe-stack-events --stack-name cloud9-management
```

在上述代码中，我们首先确保配置了正确的配置文件，然后运行`aws cloudformation deploy`命令，使用`--stack-name`标志指定堆栈名称和`--template-file`标志指定模板文件。`--parameter-overrides`标志允许您以`<parameter>=<value>`格式提供输入参数值-请注意，在像这样的更新场景中，如果您不指定任何参数覆盖，将使用先前提供的参数值（在本例中创建堆栈时）。

请注意，更新实际上失败了，如果您通过 CloudFormation 控制台查看堆栈事件，您可以找出堆栈更新失败的原因。

![](img/74b208ac-b2b6-4b7d-8dd4-d59eaaa6da1c.png)CloudFormation 堆栈更新失败

在上述屏幕截图中，您可以看到堆栈更新失败，因为更改需要 CloudFormation 创建并替换现有资源（在本例中为 Cloud9 环境）为新资源。由于 CloudFormation 始终在销毁任何已被替换的旧资源之前尝试创建新资源，因为资源配置了名称，CloudFormation 无法使用相同名称创建新资源，导致失败。这突显了 CloudFormation 的一个重要注意事项-在定义资源的静态名称时要非常小心-如果 CloudFormation 需要在像这样的更新场景中替换资源，更新将失败，因为通常资源名称必须是唯一的。

有关 CloudFormation 何时选择替换资源（如果正在更新资源），请参考[Amazon Web Services 资源类型参考](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)文档中为每种资源类型定义的资源属性。

您可以看到，CloudFormation 在失败后会自动回滚更改，撤消导致失败的任何更改。堆栈的状态最终会更改为`UPDATE_ROLLBACK_COMPLETE`，表示发生了失败和回滚。

解决堆栈失败的一个修复方法是在堆栈中的`ManagementStation`资源上删除`Name`属性 - 在这种情况下，CloudFormation 将确保生成一个唯一的名称（通常基于 CloudFormation 堆栈名称并附加一些随机的字母数字字符），这意味着每次更新资源以便需要替换时，CloudFormation 将简单地生成一个新的唯一名称并避免我们遇到的失败场景。

# 删除 CloudFormation 堆栈

现在您了解了如何创建和更新堆栈，让我们讨论如何删除堆栈。您可以通过 CloudFormation 仪表板非常轻松地删除堆栈，只需选择堆栈，选择**操作**，然后点击**删除堆栈**：

删除 CloudFormation 堆栈

点击**是，删除**以确认删除堆栈后，CloudFormation 将继续删除堆栈中定义的每个资源。完成后，堆栈将从 CloudFormation 仪表板中消失，尽管您可以更改位于**创建堆栈**按钮下方的**筛选器**下拉菜单，以点击**已删除**以查看以前删除的堆栈。

有人可能会认为删除堆栈太容易了。如果您担心意外删除堆栈，您可以在前面的截图中选择**更改终止保护**选项以启用终止保护，这将防止堆栈被意外删除。

# 摘要

在本章中，您学习了如何通过创建免费账户和建立账户的根用户来开始使用 AWS。您学会了如何使用多因素身份验证来保护根访问权限，然后创建了一些 IAM 资源，这些资源是管理您的账户所必需的。您首先创建了一个名为**admin**的管理 IAM 角色，然后创建了一个管理员组，将其分配为允许假定您的管理 IAM 角色的单一权限。假定角色的这种方法是管理 AWS 的推荐和最佳实践方法，并支持更复杂的多账户拓扑结构，在这种结构中，您可以将所有 IAM 用户托管在一个账户中，并在其他账户中假定管理角色。

然后，您创建了一个用户组，并分配了一个托管策略，该策略强制要求属于该组的任何用户进行多因素身份验证（MFA）。MFA 现在应被视为任何使用 AWS 的组织的强制性安全要求，简单地将用户分配到强制执行 MFA 要求的用户组是一种非常简单和可扩展的机制来实现这一点。创建用户并将其分配到管理员和用户组后，您学会了首次用户设置其访问所需的步骤，其中包括使用一次性密码登录，建立新密码，然后设置 MFA 设备。一旦用户使用 MFA 登录，用户就能执行分配给他们的任何权限 - 例如，您在本章中创建的用户被分配到了管理员组，因此能够承担管理员 IAM 角色，您可以在 AWS 控制台中使用内置的 Switch Role 功能执行此操作。

随着您的 IAM 设置完成并能够通过控制台承担管理员角色，我们接下来将注意力转向命令行，安装 AWS CLI，通过控制台生成访问密钥，然后在本地`~/.aws`文件夹中配置您的访问密钥凭据，该文件夹由 AWS CLI 用于存储凭据和配置文件。您学会了如何在`~/.aws/configuration`文件中配置命名配置文件，该文件会自动承担管理员角色，并在 CLI 检测到需要新的临时会话凭据时提示输入 MFA 代码。您还创建了一个 EC2 密钥对，以便您可以使用 SSH 访问 EC2 实例。

最后，您了解了 AWS CloudFormation，并学会了如何定义 CloudFormation 模板并部署 CloudFormation 堆栈，这是基于您的 CloudFormation 模板定义的资源集合。您学会了 CloudFormation 模板的基本结构，如何使用 AWS 控制台创建堆栈，以及如何使用 AWS CLI 部署堆栈。

在下一章中，您将介绍弹性容器服务，您将充分利用您的新 AWS 账户，并学习如何创建 ECS 集群并将 Docker 应用程序部署到 ECS。

# 问题

1.  真/假：建立免费的 AWS 账户需要一个有效的信用卡。

1.  正确/错误：您应该始终使用根帐户执行管理操作。

1.  正确/错误：您应该直接为 IAM 用户和/或组分配 IAM 权限。

1.  您将使用哪个 IAM 托管策略来分配管理权限？

1.  您运行什么命令来安装 AWS CLI？

1.  正确/错误：当您配置 AWS CLI 时，您必须在本地存储您的 IAM 用户名和密码。

1.  您在哪里存储 AWS CLI 的凭据？

1.  您设置了一个需要 MFA 才能执行管理操作的 IAM 用户。IAM 用户设置了他们的 AWS CLI，但在尝试运行 AWS CLI 命令时抱怨未经授权的错误。命名配置文件包括`source_profile`，`role_arn`和`role_session_name`参数，并且您确认这些已正确配置。您将如何解决这个问题？

1.  正确/错误：CloudFormation 模板可以使用 JSON 或 YAML 编写。

1.  正确/错误：您可以使用`!Ref`关键字来引用 CloudFormation 模板中的另一个资源或参数。

1.  您在 CloudFormation 模板中定义了一个资源，其中包括一个可选的`Name`属性，您将其配置为`my-resource`。您成功从模板创建了一个新堆栈，然后对文档中规定将需要替换整个资源的资源进行了更改。您能成功部署这个更改吗？

# 进一步阅读

您可以查看以下链接，了解本章涵盖的主题的更多信息：

+   设置免费层帐户：[`aws.amazon.com/free`](https://aws.amazon.com/free)

+   IAM 最佳实践：[`docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html`](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

+   您的 AWS 帐户 ID 和别名：[`docs.aws.amazon.com/IAM/latest/UserGuide/console_account-alias.html`](https://docs.aws.amazon.com/IAM/latest/UserGuide/console_account-alias.html)

+   改进 AWS Force MFA 策略：[`www.trek10.com/blog/improving-the-aws-force-mfa-policy-for-IAM-users/`](https://www.trek10.com/blog/improving-the-aws-force-mfa-policy-for-IAM-users/)

+   安装 AWS CLI：[`docs.aws.amazon.com/cli/latest/userguide/installing.html`](https://docs.aws.amazon.com/cli/latest/userguide/installing.html)

+   AWS CLI 参考：[`docs.aws.amazon.com/cli/latest/reference/`](https://docs.aws.amazon.com/cli/latest/reference/)

+   AWS CLI 配置变量：[`docs.aws.amazon.com/cli/latest/topic/config-vars.html`](https://docs.aws.amazon.com/cli/latest/topic/config-vars.html)

+   AWS shell：[`github.com/awslabs/aws-shell`](https://github.com/awslabs/aws-shell)

+   AWS CloudFormation 用户指南：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)

+   AWS CloudFormation 模板解剖：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-anatomy.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-anatomy.html)

+   AWS CloudFormation 资源类型参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)

+   AWS CloudFormation 内部函数：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html)

+   AWS CloudFormation 伪参数：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/pseudo-parameter-reference.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/pseudo-parameter-reference.html)

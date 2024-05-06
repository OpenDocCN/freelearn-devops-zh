## 17：企业级功能

本章是上一章的延续，涵盖了 Docker Universal Control Plane（UCP）和 Docker Trusted Registry（DTR）提供的一些企业级功能。

我们将假设您已经阅读了上一章，因此知道如何安装和配置它们，以及执行备份和恢复操作。

我们将把本章分为两部分：

+   简而言之

+   深入挖掘

### 企业级功能-简而言之

企业希望使用 Docker 和容器，但他们需要像真正的企业应用程序一样打包和支持。他们还需要像基于角色的访问控制和与 Active Directory 等企业目录服务的集成。这就是*Docker 企业版*发挥作用的地方。

Docker 企业版是 Docker 引擎的强化版本，具有运维 UI，安全注册表和一堆企业专注的功能。您可以在本地或云端部署它，自己管理它，并且可以获得支持合同。

总之，它是一个容器即服务平台，您可以在自己公司的数据中心安全运行。

### 企业级功能-深入挖掘

我们将把本章的主要部分分为以下几个部分：

+   基于角色的访问控制（RBAC）

+   Active Directory 集成

+   Docker 内容信任（DCT）

+   配置 Docker Trusted Registry（DTR）

+   使用 Docker Trusted Registry

+   镜像推广

+   HTTP 路由网格（HRM）

#### 基于角色的访问控制（RBAC）

在过去的 10 年中，我大部分时间都在金融服务行业从事 IT 工作。在我工作的大多数地方，角色基础访问控制（RBAC）和 Active Directory（AD）集成是强制性的两个复选框。如果你试图向我们销售一个产品，而它没有这两个功能，我们是不会购买的！

幸运的是，Docker EE 都有。在本节中，我们将讨论 RBAC。

UCP 通过一种称为*授予*的东西来实现 RBAC。在高层次上，授予由三个部分组成：

+   **主题**

+   **角色**

+   **集合**

*主题*是一个或多个用户或团队。*角色*是权限集，*集合*是这些权限适用的资源。见图 17.1。

![图 17.1 授予](img/figure17-1.png)

图 17.1 授予

图 17.2 显示了一个示例，其中`SRT`团队对`/zones/dev/srt`集合中的所有资源具有`container-full-control`访问权限。

![图 17.2](img/figure17-2.png)

图 17.2

让我们完成以下步骤来创建一个授予：

+   创建用户和团队

+   创建一个自定义角色

+   创建一个集合

+   创建一个授权

只有 UCP 管理员才能创建和管理用户、团队、角色、集合和授权。因此，为了跟随操作，你需要以 UCP 管理员身份登录。 

##### 创建用户和团队

将用户分组到团队，并将团队分配到授权是最佳实践。你*可以*将单个用户分配给*授权*，但这并不推荐。

让我们创建一些用户和团队。

1.  登录到 UCP。

1.  展开“用户管理”并点击“用户”。

从这里你可以创建用户。

1.  点击“组织和团队”。

从这里你可以创建组织。在接下来的几个步骤中，我们将使用一个名为“制造业”的组织作为示例。

1.  点击“制造业”组织并创建一个团队。

团队存在于组织中。不可能创建一个不属于任何组织的团队，一个团队只能是一个组织的成员。

1.  将用户添加到一个团队。

要将用户添加到团队，你需要点击进入团队，并从“操作”菜单中选择“添加用户”。

图 17.3 显示了如何将用户添加到“制造业”组织中的`SRT`团队。

![图 17.3 将用户添加到团队](img/figure17-3.png)

图 17.3 将用户添加到团队

现在你有了一些用户和团队。UCP 与 DTR 共享其用户数据库，这意味着你在 UCP 中创建的任何用户和团队也可以在 DTR 中使用。

##### 创建一个自定义角色

自定义角色非常强大，它们可以让你对分配的权限进行极其精细的控制。在这一步中，我们将创建一个名为`secret-ops`的新自定义角色，允许主体创建、删除、更新、使用和查看 Docker secrets。

1.  展开左侧导航窗格的“用户管理”选项卡，选择“角色”。

1.  创建一个新角色。

1.  给角色命名。

在这个例子中，我们将创建一个名为“secret-ops”的新自定义角色，具有执行所有与 secret 相关的操作的权限。

1.  选择“操作”并探索可以分配给角色的操作列表。

列表很长，允许你指定单个 API 操作。

1.  选择你想要分配给角色的单个 API 操作。

在这个例子中，我们将分配所有与 secret 相关的 API 操作。

![图 17.4 分配 API 操作给自定义角色](img/figure17-4.png)

图 17.4 分配 API 操作给自定义角色

1.  点击“创建”。

该角色现在已经在系统中，并可以分配给多个授权。

让我们创建一个集合。

##### 创建一个集合

在上一章中，我们了解到网络、卷、秘密、服务和节点都是 Swarm 资源——它们被存储在 Swarm 配置中的`/var/lib/docker/swarm`中。*集合*让你以符合组织结构和 IT 要求的方式对它们进行分组。例如，您的 IT 基础设施可能分为三个区域；`prod`、`test`和`dev`。如果是这种情况，您可以创建三个集合，并分配资源给每个集合，如图 17.5 所示。

![图 17.5 高级集合](img/figure17-5.png)

图 17.5 高级集合

每个资源只能属于一个集合。

在接下来的步骤中，我们将创建一个名为`zones/dev/srt`的新集合，并将一个秘密分配给它。集合本质上是分层的，因此您需要创建三个嵌套的集合，如下所示：`zones` > `dev` > `srt`。

从 Docker UCP web UI 执行以下所有步骤。

1.  从左侧导航窗格中选择`集合`，然后选择`创建集合`。

1.  创建名为`zones`的根集合。

1.  点击“查看子项”以查看`/zones`集合。

1.  创建一个名为`dev`的嵌套子集合。

1.  点击“查看子项”以查看`/zones/dev`集合。

1.  创建名为`srt`的最终嵌套子集合。

现在您有一个名为`/zones/dev/srt`的集合。但是，它目前是空的。在接下来的步骤中，我们将向其中添加一个*秘密*。

1.  创建一个新的秘密。

您可以从命令行或 UCP web UI 中创建它。我们将解释 web UI 方法。

从 UCP web UI 中点击：`秘密` > `创建秘密`。给它一个名称，一些数据，然后点击`保存`。

在创建秘密的同时也可以配置*集合*。但我们不是这样做的。

1.  在 UCP web UI 中找到并选择秘密。

1.  从“配置”下拉菜单中点击“集合”。

1.  通过“查看子项”层次结构导航，直到选择`/zones/dev/srt`集合，然后点击“保存”。

秘密现在是`/zones/dev/srt`集合的一部分。它不能是任何其他集合的成员。

在创建*授权*之前，还有一件关于*集合*的事情。集合具有继承模型，其中对任何集合的访问自动意味着对嵌套子集合的访问。在图 17.6 中，`dev`团队可以访问`/zones/dev`集合，因此它自动获得对`srt`、`hellcat`和`daemon`子集合中资源的访问权限。

![图 17.6 集合继承](img/figure17-6.png)

图 17.6 集合继承

###### 创建授予

现在您已经有了用户和团队、自定义角色和集合，您可以创建一个授予。在这个例子中，我们将为`srt-dev`团队创建一个授予，使其对`/zones/dev/srt`集合中的所有资源具有自定义`secret-ops`角色。

授予涉及*谁*，获得*什么访问权限*，对*哪些资源*。

![](img/figure17-7.png)

1.  展开左侧导航窗格上的“用户管理”选项卡，然后单击“授予”。

1.  创建一个新的授予。

1.  点击`Subject`，从`manufacturing`组织中选择`SRT`团队。

可以选择整个组织。如果这样做，组织内的所有团队都将包括在授予中。

1.  单击“角色”，然后选择自定义的`secret-ops`角色。

1.  单击“集合”，然后选择`/zones/dev/srt`集合。

在看到`/zones`之前，您可能需要查看顶级`Swarm`集合的子项。

1.  单击“保存”以创建授予。

现在已经创建了授予，并且可以在系统上的所有授予列表中查看。`manufacturing/SRT`团队的成员现在可以在`/zones/dev/srt`集合中执行所有与秘密相关的操作。

![](img/figure17-8.png)

您可以在授予处于活动状态时修改授予的组件。例如，您可以将用户添加到团队，将资源添加到集合。但是您不能更改分配给角色的 API 操作。如果您想要更改角色的权限，您需要创建一个具有所需权限的新角色。

###### 节点的 RBAC

最后关于 RBAC 的一件事。您可以将集群中的工作节点分组以进行调度。例如，您可能会为开发、测试和 QA 工作负载运行一个单一集群——一个单一集群可能会减少管理开销，并使将节点分配给三个不同环境变得更容易。但是您可能还希望将工作节点分成几部分，以便只有`dev`团队的成员可以将工作安排到`dev`集合中的节点等。

正如您所期望的那样，您可以通过*授予*来实现这一点。首先，您会将 UCP Worker 节点分配给一个自定义*集合*。然后，您会创建一个包括集合、内置的`Scheduler`*角色*和您想要分配授予的团队的授予。这样可以让您控制哪些用户可以将工作安排到集群中的哪些节点。

作为一个简单的例子，图 17.9 中显示的授权将允许`dev`团队的成员能够将服务和容器调度到`/zones/dev`集合中的工作节点上。

![图 17.9 节点的 RBAC](img/figure17-9.png)

图 17.9 节点的 RBAC

就是这样！您知道如何在 Docker UCP 中实现 RBAC 了！

#### Active Directory 集成

像所有优秀的企业工具一样，UCP 集成了 Active Directory 和其他 LDAP 目录服务。这使其能够利用来自您组织已建立的单一登录系统的现有用户和组。

在本节中进一步进行之前，非常重要的是与负责组织目录服务的团队讨论任何 AD/DLAP 集成计划。让他们从一开始就参与进来，这样您的规划和实施才能尽可能顺利！

开箱即用，UCP 用户和组数据存储在本地数据库中，DTR 利用该数据库实现单一登录（SSO）体验。这会在本地验证所有访问请求，并允许您登录到 DTR 而无需再次输入 UCP 凭据。但是，**UCP 管理员**可以配置 UCP 以利用存储在 AD 或其他 LDAP 目录服务中的现有企业用户帐户，将认证和帐户管理外包给现有团队和流程。

以下步骤将向您展示如何配置 UCP 以利用 AD 进行用户帐户。在高层次上，该过程告诉 UCP 在特定目录中搜索用户帐户，并将其复制到 UCP 中。如前所述，与您的目录服务团队协调此工作。

让我们开始吧。

1.  展开左侧导航窗格中的`Admin`下拉菜单，然后选择`Admin Settings`。

1.  选择`Authentication & Authorization`，并在**LDAP Enabled**标题下单击`Yes`。

1.  配置 LDAP 服务器设置。

在高层次上，您可以将`LDAP 服务器设置`视为*搜索位置*。例如，要在哪些目录中查找用户帐户。

在此处输入的值将特定于您的环境。

**LDAP 服务器 URL**是您将在其中搜索帐户的域中 LDAP 服务器的名称。例如，`ad.mycompany.internal`。

**Reader DN**和**Reader Password**是具有搜索权限的目录中帐户的凭据。该帐户必须存在于您正在搜索的目录中，或者受到该目录的信任。最佳做法是让它在目录中具有*只读*权限。

您可以使用“添加 LDAP 域+”按钮添加要搜索的其他域。每个域都需要自己的 LDAP 服务器 URL 和读取器帐户。

1.  配置 LDAP 用户搜索配置。

如果“LDAP 服务器设置”是*搜索位置*，那么“LDAP 用户搜索配置”就是*搜索对象*。

**基本 DN**指定从哪个 LDAP 节点开始搜索。

**用户名属性**是用作 UCP 用户名的 LDAP 属性。

**全名属性**是用作 UCP 帐户全名的 LDAP 属性。

请参阅其他更高级的设置的文档。在配置 LDAP 集成时，您还应该与目录服务团队进行咨询。

1.  一旦您配置了 LDAP 设置，UCP 将搜索匹配的用户并在 UCP 用户数据库中创建它们。然后，它将根据“同步间隔（小时）”设置执行定期同步操作。

如果您勾选了“即时用户配置”框，UCP 将推迟创建用户帐户，直到每个帐户的第一次登录事件。

1.  在点击“保存”之前，您应该始终在“LDAP 测试登录”部分执行测试登录。

测试登录需要使用 LDAP 系统中有效的用户帐户。测试将应用上面各节中定义的所有配置值（您即将保存的 LDAP 配置）。

只有在测试登录成功时才保存配置。

1.  保存配置。

此时，UCP 将搜索 LDAP 系统并创建与基本 DN 和其他提供的条件匹配的用户帐户。

在配置 LDAP 之前创建的本地用户帐户仍将存在于系统中，并且仍然可以使用。

#### Docker 内容信任（DCT）

在现代 IT 世界中，*信任*是一件大事！并且未来它将变得更加重要。幸运的是，Docker 通过一个名为 Docker 内容信任（DCT）的功能来实现信任。

在非常高的层次上，Docker 镜像的发布者可以在将其推送到存储库时对其进行签名。消费者随后可以在拉取它们时验证它们，或执行构建和运行操作。长话短说，DCT 使消费者能够保证他们得到了他们所要求的东西！

图 17.10 显示了高级架构。

![图 17.10 高级 DCT 架构](img/figure17-10.png)

图 17.10 高级 DCT 架构

DCT 实现*客户端*签名和验证操作，这意味着 Docker 客户端执行这些操作。

- 尽管在互联网上传输和推送软件时，加密保证非常重要，但在整个软件交付流程的每个层面和每个步骤中，它变得越来越重要。希望不久的将来，交付链的所有方面都将充满加密信任保证。

- 让我们快速示例配置 DCT 并看到它的运行情况。

- 您将需要一个单独的 Docker 客户端和一个可以将镜像推送到的仓库。Docker Hub 上的仓库将起作用。

- DCT 是通过 DOCKER_CONTENT_TRUST 环境变量打开和关闭的。将其设置为“1”将在当前会话中打开 DCT。将其设置为任何其他值将关闭它。以下示例将在基于 Linux 的 Docker 主机上打开它。

```
$ `export` `DOCKER_CONTENT_TRUST``=``1` 
```

- 未来的 docker push 命令将自动在推送操作中签署镜像。同样，只有签署的镜像才能使用 pull、build 和 run 命令。

- 让我们将一个带有新标记的镜像推送到一个仓库。

- 被推送的镜像可以是任何镜像。实际上，我正在使用的是我刚刚拉取的当前 alpine:latest。目前，它还没有由我签名！

1.  - 给镜像打标记，这样它就可以推送到您想要的仓库。我将把它推送到我个人 Docker Hub 帐户命名空间内的一个新仓库。

```
 $ docker image tag alpine:latest nigelpoulton/dockerbook:v1 
```

- 登录到 Docker Hub（或其他注册表），这样您就可以在下一步中推送镜像。

```
 $ docker login
 Login with your Docker ID to push and pull images from Docker Hub.
 Username: nigelpoulton
 Password:
 Login Succeeded 
```

- 推送新标记的镜像。

- ```
     $ docker image push nigelpoulton/dockerbook:v1
     The push refers to a repository [docker.io/nigelpoulton/dockerbook]
     cd7100a72410: Mounted from library/alpine
     v1: digest: sha256:8c03...acbc size: 528
     Signing and pushing trust metadata
     <Snip>
     Enter passphrase for new root key with ID 865e4ec:
     Repeat passphrase for new root key with ID 865e4ec:
     Enter passphrase for new repository key with ID bd0d97d:
     Repeat passphrase for new repository key with ID bd0d97d:
     Finished initializing "docker.io/nigelpoulton/sign"
     Successfully signed "docker.io/nigelpoulton/sign":v1 
    `````

```With DCT enabled, the image was automatically signed as part of the push operation.

Two sets of keys were created as part of the signing operation:

*   Root key
*   Repository key

By default, both are stored below a hidden folder in your home directory called `docker`. On Linux this is `~/.docker/trust`.

The **root key** is the master key (of sorts). It’s used to create and sign new repository keys, and should be kept safe. This means you should protect it with a strong passphrase, and you should store it offline in a secure place when not in use. If it gets compromised, you’ll be in world of pain! You would normally only have one per person, or may be even one per team or organization, and you’ll normally only use it to create new repository keys.

The **repository key**, also known as the *tagging key* is a per-repository key that is used to sign tagged images pushed to a particular repository. As such, you’ll have one per repository. It’s quite a bit easier to recover from a loss of this key, but you should still protect it with a strong passphrase and keep it safe.

Each time you push an image to a **new repository**, you’ll create a new repository tagging key. You need your **root key** to do this, so you’ll need to enter the root key’s passphrase. Subsequent pushes to the same repository will only require you to enter the passphrase for the repository tagging key.

There’s another key called the `timestamp key`. This gets stored in the remote repository and is used in more advanced use-cases to ensure things like *freshness*.

Let’s have a look at pulling images with DCT enabled.

Perform the following commands from the same Docker host that has DCT enabled.

Pull an unsigned image.

```

- docker image pull nigelpoulton/dockerbook:unsigned

- 错误：docker.io/nigelpoulton/dockerbook 的信任数据不存在：

- notary.docker.io 没有 docker.io/nigelpoulton/dockerbook 的信任数据

```

 `> **Note:** Sometimes the error message will be `No trust data for unsigned`.

See how Docker has refused to download the image because it is not signed.

You’ll get similar errors if you try to build new images or run new containers from unsigned images. Let’s test it.

Pull the unsigned image by using the `--disable-content-trust` flag to override DCT.

```

- docker image pull --disable-content-trust nigelpoulton/dockerbook:unsigned

```

 `The `--disable-content-trust` flag overrides DCT on a per-command basis. Use it wisely.

Now try and run a container from the unsigned image.

```

- docker 容器运行-d --rm nigelpoulton/dockerbook:unsigned

- docker：未签名的没有信任数据。

```

 `This proves that Docker Content Trust enforces policy on `push`, `pull` and `run` operations. Try a `build` to see it work there as well.

Docker UCP also supports DCT, allowing you to set a UCP-wide signing policy.

To enable DCT across an entire UCP, expand the `Admin` drop-down and click `Admin Settings`. Select the `Docker Content Trust` option and tick the `Run Only Signed Images` tickbox. This will enforce a signing policy across the entire cluster that will only allow you to deploy services using signed images.

The default configuration will allow any image signed by a valid UCP user. You can optionally configure a list of teams that are authorized to sign images.

That’s the basics of Docker Content Trust. Let’s move on to configuring and using Docker Trusted Registry (DTR).

#### Configuring Docker Trusted Registry (DTR)

In the previous chapter we installed DTR, plugged it in to a shared storage backend, and configured HA. We also learned that UCP and DTR share a common single-sign-on sub-system. But there’s a few other important things you should configure. Let’s take a look.

Most of the DTR configuration settings are located on the `Settings` page of the DTR web UI.

From the `General` tab you can configure:

*   Automatic update settings
*   Licensing
*   Load balancer address
*   Certificates
*   Single-sign-on

The `TLS Settings` under `Domains & proxies` allows you to change the certificates used by UCP. By default, DTR uses self-signed certificates, but you can use this page to configure the use of custom certificates.

The `Storage` tab lets you configure the backend used for **image storage**. We saw this in the previous chapter when we configured a shared Amazon S3 backend so that we could configure DTR HA. Other storage options include object storage services from other cloud providers, as well as volumes and NFS shares.

The `Security` tab is where you enable and disable *Image Scanning* — binary-level scans that identify known vulnerabilities in images. When you enable *image scanning*, you have the option of updating the vulnerability database *online* or *offline*. Online will automatically sync the database over the internet, whereas the offline method is for DTR instances that do not have internet access and need to update the database manually.

See the *Security in Docker* chapter for more information on Image Scanning.

Last but not least, the `Garbage Collection` tab lets you configure when DTR will perform garbage collection on image layers that are no longer referenced in the Registry. By default, unreferenced layers are not garbage collected, resulting in large amounts of wasted disk space. If you enable garbage collection, layers that are no longer referenced by an image will be deleted, but layers that are referenced by at least one image manifest will not.

See the chapter on Images for more information about how image manifests reference image layers.

Now that we know how to configure DTR, let’s use it!

#### Using Docker Trusted Registry

Docker Trusted Registry is a secure on-premises registry that you configure and manage yourself. It’s integrated into UCP for smooth out-of-the-box experience.

In this section, we’ll look at how to push and pull images from it, and we’ll learn how to inspect and manage repositories using the DTR web UI.

##### Log in to the DTR UI and create a repo and permissions

Let’s log in to DTR and create a new repository that members of the `technology/devs` team can push and pull images from.

Log on to DTR. The DTR URL can be found in the UCP web UI under `Admin` > `Admin Settings` > `Docker Trusted Registry`. Remember that the DTR web UI is accessible over HTTPS on TCP port 443.

Create a new organization and team, and add a user to it. The example will create an organization called `technology`, a team called `devs`, and add the `nigelpoulton` user to it. You can substitute these values in your environment.

1.  Click `Organizations` in the left navigation pane.
2.  Click `New organization` and call it `technology`.
3.  Select the new `technology` organization and click the `+` button next to `TEAMS` as shown in Figure 17.11.![Figure 17.11](img/figure17-11.png)

    Figure 17.11

4.  With the `devs` team selected, add an existing user.

    The example will add the `nigelpoulton` user. Your user will be different in your environment.

The organization and team changes you have made in DTR will be reflected in UCP. This is because they share the same accounts database.

Let’s create a new repository and add the `technology/devs` team with read/write permission.

Perform all of the following in the DTR web UI.

1.  If you aren’t already, navigate to `Organizations` > `technology` > `devs`.
2.  Select the `Repositories` tab and create a new repository.
3.  Configure the repository as follows.

    Make it a **New** repository called **test** under the **technology** organization. Make it **public**, enable **scan on push** and assign **read/write** permissions. Figure 17.12 shows a screenshot of how it should look.

    ![Figure 17.12 Creating a new DTR image repo](img/figure17-12.png)

    Figure 17.12 Creating a new DTR image repo

4.  Save changes.

Congratulations! You have an image repo on DTR called `<dtr-url>/technology`, and members of the `technology/devs` team have read/write access, meaning they can `push` and `pull` from it.

##### Push an image to the DTR repo

In this step we’ll push a new image to the repo you just created. To do this, we’ll complete the following steps:

1.  Pull an image and re-tag it.
2.  Configure a client to use a certificate bundle.
3.  Push the re-tagged image to the DTR repo.
4.  Verify the operation in the DTR web UI.

Let’s pull an image and tag it so that it can be pushed to the DTR repo.

It doesn’t matter what image you pull. The example uses the `alpine:latest` image because it’s small.

```

- docker pull alpine:latest

- 最新：正在从库/alpine 拉取

- ff3a5c916c92：拉取完成

- 摘要：sha256:7df6...b1c0

- 状态：已下载更新的镜像 alpine:latest

```

 `In order to push an image to a specific repo, you need to tag the image with the name of the repo. The example DTR repo has a fully qualified name of `dtr.mydns.com/technology/test`. This is made by combining the DNS name of the DTR and the name of the repo. Yours will be different.

Tag the image so it can be pushed to the DTR repo.

```

- docker image tag alpine:latest dtr.mydns.com/technology/test:v1

```

 `The next job is to configure a Docker client to authenticate as a user in the group that has read/write permission to the repository. The high-level process is to create a certificate bundle for the user and configure a Docker client to use those certificates.

1.  Login to UCP as admin, or a user that has read/write permission to the DTR repo.
2.  Navigate to the desired user account and create a `client bundle`.
3.  Copy the bundle file to the Docker client you want to configure.
4.  Login to the Docker client and perform the following commands from the client.
5.  Unzip the bundle and run the appropriate shell script to configure your environment.

The following will work on Mac and Linux.

```

- 执行“$（<env.sh）”。

```

`*   Run a `docker version` command to verify the environment has been configured and the certificates are being used.

    As long as the `Server` section of the output shows the `Version` as `ucp/x.x.x` it is working. This is because the shell script configured the Docker client to talk to a remote daemon on a UCP manager. It also configured the Docker client to sign all commands with the certificates.` 

 `The next job is to log in to DTR. The DTR URL and username will be different in your environment.

```

- docker login dtr.mydns.com

- 用户名：nigelpoulton

- 密码：

- 登录成功

```

 `You are now ready to push the re-tagged image to DTR.

```

- docker image push dtr.mydns.com/technology/test:v1

- 推送是指一个仓库`[dtr.mydns.com/technology/test]`

- cd7100a72410：已推送

- v1：摘要：sha256:8c03...acbc 大小：528

- ```

 `The push looks successful, but let’s verify the operation in the DTR web UI.

1.  If you aren’t already, login to the DTR web UI.
2.  Click `Repositories` in the left navigation pane.
3.  Click `View Details` for the `technology/test` repository.
4.  Click the `IMAGES` tab.

Figure 17.13 shows what the image looks like in the DTR repo. We can see that the image is a Linux-based image and that it has 3 major vulnerabilities. We know about the vulnerabilities because we configured the repository to scan all newly-pushed images.

![Figure 17.13](img/figure17-13.png)

Figure 17.13

Congratulations. You’ve successfully pushed an image to a new repository on DTR.

You can select the checkbox to the left of the image and delete it. Be certain before doing this, as the operation cannot be undone.

#### Image promotions

DTR has a couple other interesting features:

*   Image promotions
*   Immutable repos

Image promotions let you build policy-based automated pipelines that promote images through a set of repositories in the same DTR.

As an example, you might have developers pushing images to a repository called `base`. But you don’t want them to be able to push images straight to production in case they contain vulnerabilities. To help with situations like this, DTR allows you to assign a policy to the `base` repo, that will scan all pushed images, and promote them to another repo based on the results of the scan. If the scan highlights issues, the policy can *promote* the image to a quarantined repo, whereas if the scan results are clean, it can promote it to a QA or prod repo. You can even re-tag the image as it passes through the pipeline.

Let’s see it in action.

The example that we’ll walk through has a single DTR with 3 image repos:

*   `base`
*   `good`
*   `bad`

The `good` and `bad` repos are empty, but the `base` repo has two images in it, shown in Figure 17.14.

![Figure 17.14](img/figure17-14.png)

Figure 17.14

As we can see, both images have been scanned, `v1` is clean and has no known vulnerabilities, but `v2` has 3 majors.

We’ll create two policies on the `base` repo so that images with a clean bill-of-health are promoted to the `good` repo, and images with known vulnerabilities are promoted to the `bad` repo.

Perform all of the following actions on the `base` repo.

1.  Click the `Policies` tab and make sure that `Is source` is selected.
2.  Click `New promotion policy`.
3.  Under “PROMOTE TO TARGET IF…” select `All Vulnerabilities` and create a policy for `equals 0`.![](img/figure17-15.png)

    This will create a policy that acts on all images with zero vulnerabilities.

    Don’t forget to click `Add` before moving to the next step.

4.  Select the `TARGET REPOSITORY` as `technology/good` and hit `Save & Apply`.

    Clicking `Save` will apply the policy to the repo and enforce it for all new images pushed the repo, but it will not affect images already in the repo. `Save & Apply` will do the same, **but also for images already in the repo**.

    If you click `Save & Apply`, the policy will immediately evaluate all images in the repo and promote those that are clean. This means the `v1` image will be promoted to the `technology/good` repo.

5.  Inspect the `technology/good` repo.

    As you can see in Figure 17.16, the `v1` image has been promoted and is showing in the UI as `PROMOTED`.

    ![Figure 17.16](img/figure17-16.png)

    Figure 17.16

The promotion policy is working. Let’s create another one to *promote* images that do have vulnerabilities to the `technology/bad` repo.

Perform all of the following from the `technology/base` repo.

1.  Create another new promotion policy.
2.  Create a policy criteria for All Vulnerabilities > 0 and click `Add`.![Figure 17.17](img/figure17-17.png)

    Figure 17.17

3.  Add the target repo as `technology/bad`, and add “-dirty” to the `TAG NAME IN TARGET` box so that it is now “%n-dirty”. This last bit will re-tag the image as part of the promotion.
4.  Click `Save & Apply`.
5.  Check the `technology/bad` repo to confirm that the policy is enforcing and the `v2` image has been promoted and re-tagged.![Figure 17.18](img/figure17-18.png)

    Figure 17.18

Now that images are being promoted to the `technology/good` repo if they have no vulnerabilities, it might be a good idea to make the repo immutable. This will prevent images from being overwritten and deleted.

1.  Navigate to the `technology/good` repo and click the `Settings` tab.
2.  Set `IMMUTABILITY` to `On` and click `Save`.
3.  Try and delete the image.

    You’ll get the following error.

    ![](img/figure17-19.png)

Time for one last feature!

#### HTTP Routing Mesh (HRM)

Docker Swarm features a layer-4 routing mesh called the Swarm Routing Mesh. This exposes Swarm services on all nodes in the cluster and balances incoming traffic across service replicas. The end results is a moderately even balance of traffic to all service replicas. However, it has no application intelligence. For example, it cannot route based on data at layer 7 in the HTTP headers. To overcome this, UCP implements a layer-7 routing mesh called the HTTP Routing Mesh, or HRM for short. This builds on top of the Swarm Routing Mesh.

The HRM allows multiple Swarm services to be published on the same Swarm-wide port, with ingress traffic being routed to the right service based on hostname data stored in the HTTP headers of incoming requests.

Figure 17.20 shows a simple two-service example.

![Figure 17.20](img/figure17-20.png)

Figure 17.20

In the picture, the laptop client is making an HTTP request to `mustang.internal` on TCP port 80\. The UCP cluster has two services that are both listening on port 80\. The `mustang` service is published on port 80 and configured to receive traffic intended for the `mustang.internal` hostname. The `camero` service is also published on port 80, but is configured to receive traffic coming in to `camero.internal`.

There is a third service called HRM that maintains the mapping between hostnames and UCP services. It is the HRM that receives incoming traffic on port 80, inspects the HTTP headers and makes the decision of which service to route it to.

Let’s walk through an example, then explain a bit more detail when we’re done.

We’ll build the example shown in Figure 17.20\. The process will be as follows: Enable the HRM on port 80\. Deploy a *service* called “mustang” using the `nigelpoulton/dockerbook:mustang` image and create a hostname route for the mustang service so that requests to “mustang.internal” get routed to it. Deploy a second service called “camero” based on the `nigelpoulton/dockerbook:camero` image and create a hostname route for this one that maps it to requests for “camero.internal”.

You can use publicly resolvable DNS names such as mustang.mycompany.com, all that is required is that you have name resolution configured so that requests to those addresses resolve to the load balancer in front of your UCP cluster. IF you don’t have a load balancer, you can point traffic to the IP of any node in the cluster.

Let’s see it.

1.  If you aren’t already, log on to the UCP web UI.
2.  Navigate to `Admin` > `Admin Settings` > `Routing Mesh`.
3.  Tick the `Enable Routing Mesh` tickbox and make sure that the `HTTP Port` is configured to `80`.
4.  Click `Save`.

That’s the UCP cluster configured to use the HRM. Behind the scenes this has deployed a new *system service* called `ucp-hrm`, and a new overlay network called `ucp-hrm`.

If you inspect the `ucp-hrm` system service, you’ll see that it’s publishing port `80` in *ingress mode*. This means the `ucp-hrm` is deployed on the cluster and bound to port `80` on all nodes in the cluster. This means **all traffic** coming into the cluster on port 80 will be handled by this service. When the `mustang` and `camero` services are deployed, the `ucp-hrm` service will be updated with hostname mappings so that it knows how to route traffic to those services.

Now that the HRM is deployed, it’s time to deploy our services.

1.  Select `Services` in the left navigation pane and click `Create Service`.
2.  Deploy the “mustang” service as follows:
    *   **Details/Name:** mustang
    *   **Details/Image:** nigelpoulton/dockerbook:mustang
    *   **Network/Ports/Publish Port:** Click the option to `Publish Port +`
    *   **Network/Ports/Internal Port:** 8080
    *   **Network/Ports/Add Hostname Based Routes:** Click on the option to add a hostname based route
    *   **Network/Ports/External Scheme:** Http://
    *   **Network/Ports/Routing Mesh Host:** mustang.internal
    *   **Network/Ports/Networks:** Make sure that the service is attached to the `ucp-hrm` network
3.  Click `Create` to deploy the service.
4.  Deploy the “camero” service.

    Deploy this service with the same settings as the “mustang” service, but with the following differences:

    *   **Details/Name:** camero
    *   **Details/Image:** nigelpoulton/dockerbook:camero
    *   **Network/Ports/Routing Mesh Host:** camero.internal
5.  Click `Create`.

It’ll take a few seconds for each service to deploy, but when they’re done, you’ll be able to point a web browser at `mustang.internal` and reach the mustang service, and `camero.internal` and reach the camero service.

> **Note:** You will obviously need name resolution configured so that `mustang.internal` and `camero.internal` resolve to your UCP cluster. This can be to a load balancer sitting in front of your cluster forwarding traffic to the cluster on port 80, or you’re in a lab without a load balancer, it can be a simple local `hosts` file resolving the DNS names to the IP address of a cluster node.

Figure 17.21 shows the mustang service being reached via `mustang.internal`.

![Figure 17.21](img/figure17-21.png)

Figure 17.21

Let’s remind ourselves of how this works.

The HTTP Routing Mesh is a Docker UCP feature that builds on top of the transport layer Swarm Routing Mesh. Specifically, the HRM adds application layer intelligence in the form of hostname rules.

Enabling the HRM deploys a new UCP *system service* called `ucp-hrm`. This service is published *swarm-wide* on port 80 and 8443\. This means that all traffic arriving at the cluster on either of those ports will be sent to the `ucp-hrm` service. This puts the `ucp-hrm` service in a position to receive, inspect, and route all traffic entering the cluster on those ports.

We then deployed two *user services*. As part of deploying each service, we created a hostname mapping that was added to the `ucp-hrm` service. The “mustang” service created a mapping so that it would receive all traffic arriving on the cluster on port 80 with “mustang.internal” in the HTTP header. The “camero” service did the same thing for traffic arriving on port 80 with “camero.internal” in the HTTP header. This resulted in the `ucp-hrm` service having two entries effectively saying the following:

*   All traffic arriving on port 80 for “mustang.internal” gets sent to the “mustang” service.
*   All traffic arriving on port 80 for “camero.internal” gets sent to the “camero” service.

Let’s show Figure 17.20 again.

![Figure 17.20](img/figure17-20.png)

Figure 17.20

Hopefully this should be clear now!

### Chapter Summary

UCP and DTR join forces to provide a great suit of features that are valuable to most enterprise organizations.

Strong role-based access control is a fundamental part of UCP, with the ability be extremely granular with permissions – down to individual API operations. Integration with Active Directory and other corporate LDAP solutions is also supported.

Docker Content Trust (DCT) brings cryptographic guarantees to image-based operations. These include `push`, `pull`, `build`, and `run`. When DCT is enabled, all images pushed to remote repos are signed, and all images pulled are verified. This gives you cryptographic certainty that the image you get is the one you asked for. UCP can be configured to enforce a cluster-wide policy requiring all images to be signed.

DTR can be configured to use self-signed certificates, or certificates from trusted 3rd-party CAs. You can configure it to perform binary-level image scans that identify known vulnerabilities. And you can configure policies to automate the promotion of images through your build pipelines.

Finally, we looked at the HTTP Routing mesh that performs application layer routing based on hostnames in HTTP headers.````````````

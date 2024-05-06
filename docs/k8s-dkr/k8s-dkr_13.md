# 第十章：创建 PodSecurityPolicies

到目前为止，大部分讨论的安全重点都集中在保护 Kubernetes API 上。身份验证意味着对 API 调用进行身份验证。授权意味着授权访问某些 API。即使在关于仪表板的讨论中，也主要集中在如何通过仪表板安全地对 API 服务器进行身份验证。

这一章将会有所不同，因为我们现在将把重点转移到保护我们的节点上。我们将学习 PodSecurityPolicies（PSPs）如何保护 Kubernetes 集群的节点。我们的重点将放在容器在集群节点上的运行方式，以及如何防止这些容器获得比它们应该拥有的更多访问权限。在本章中，我们将深入了解影响的细节，看看在节点没有受到保护时，如何利用漏洞来获取对集群的访问权限。我们还将探讨即使在不需要节点访问权限的代码中，这些情景如何被利用。

在本章中，我们将涵盖以下主题：

+   什么是 PSP？

+   它们不会消失吗？

+   启用 pod 安全策略

+   PSP 的替代方案

# 技术要求

要跟随本章的示例，请确保您有一个使用*第八章*中的配置运行的 KinD 集群，*RBAC Policies and Auditing*。

您可以在以下 GitHub 存储库中访问本章的代码：[`github.com/PacktPublishing/Kubernetes-and-Docker-The-Complete-Guide/tree/master/chapter10`](https://github.com/PacktPublishing/Kubernetes-and-Docker-The-Complete-Guide/tree/master/chapter10)。

# 什么是 PodSecurityPolicy？

PSP 是一个 Kubernetes 资源，允许您为您的工作负载设置安全控制，允许您对 pod 的操作进行限制。PSP 在 pod 被允许启动之前进行评估，如果 pod 尝试执行 PSP 禁止的操作，它将不被允许启动。

许多人都有使用物理和虚拟服务器的经验，大多数人知道如何保护运行在它们上面的工作负载。当谈到保护每个工作负载时，容器需要被考虑得与众不同。要理解为什么存在 PSPs 和其他 Kubernetes 安全工具，如 Open Policy Agent（OPA），您需要了解容器与虚拟机（VM）之间的区别。

## 理解容器和虚拟机之间的区别

"*容器是轻量级虚拟机*"经常是对于新接触容器和 Kubernetes 的人描述容器的方式。虽然这样做可以形成一个简单的类比，但从安全的角度来看，这是一个危险的比较。运行时的容器是在节点上运行的进程。在 Linux 系统上，这些进程通过一系列限制它们对底层系统的可见性的 Linux 技术进行隔离。

在 Kubernetes 集群中的任何节点上运行**top**命令，所有来自容器的进程都会被列出。例如，即使 Kubernetes 在 KinD 中运行，运行**ps -A -elf | grep java**将显示 OpenUnison 和 operator 容器进程：

![图 10.1 - 从系统控制台的 Pod 进程](img/Fig_10.1_B15514.jpg)

图 10.1 - 从系统控制台的 Pod 进程

相比之下，虚拟机就像其名称所示，是一个完整的虚拟系统。它模拟自己的硬件，有独立的内核等。虚拟机监视器为虚拟机提供了从硅层到上层的隔离，而与此相比，在节点上的每个容器之间几乎没有隔离。

注意

有一些容器技术可以在自己的虚拟机上运行容器。但容器仍然只是一个进程。

当容器没有运行时，它们只是一个"tarball of tarballs"，其中文件系统的每一层都存储在一个文件中。镜像仍然存储在主机系统上，或者之前容器曾经运行或被拉取的多个主机系统上。

注意

"tarball"是由**tar** Unix 命令创建的文件。它也可以被压缩。

另一方面，虚拟机有自己的虚拟磁盘，存储整个操作系统。虽然有一些非常轻量级的虚拟机技术，但虚拟机和容器之间的大小差异通常是数量级的。

虽然有些人将容器称为轻量级虚拟机，但事实并非如此。它们的隔离方式不同，并且需要更多关注它们在节点上的运行细节。

从这一部分，你可能会认为我们在试图说容器不安全。事实恰恰相反。保护 Kubernetes 集群和其中运行的容器需要注意细节，并且需要理解容器与虚拟机的不同之处。由于很多人都了解虚拟机，因此很容易尝试将其与容器进行比较，但这样做会让你处于不利地位，因为它们是非常不同的技术。

一旦您了解了默认配置的限制和由此带来的潜在危险，您就可以纠正这些“问题”。

## 容器越狱

容器越狱是指您的容器进程获得对底层节点的访问权限。一旦在节点上，攻击者现在可以访问所有其他的 pod 和环境中节点的任何功能。越狱也可能是将本地文件系统挂载到您的容器中。来自[`securekubernetes.com`](https://securekubernetes.com)的一个例子，最初由 VMware 的 Duffie Cooley 指出，使用一个容器来挂载本地文件系统。在 KinD 集群上运行这个命令会打开对节点文件系统的读写权限：

kubectl run r00t --restart=Never -ti --rm --image lol --overrides '{"spec":{"hostPID": true, "containers":[{"name":"1","image":"alpine","command":["nsenter","--mount=/proc/1/ns/mnt","--","/bin/bash"],"stdin": true,"tty":true,"imagePullPolicy":"IfNotPresent","securityContext":{"privileged":true}}]}}'

如果你看不到命令提示符，请尝试按 Enter 键。

上面代码中的**run**命令启动了一个容器，并添加了一个关键选项**hostPID: true**，允许容器共享主机的进程命名空间。您可能会看到一些其他选项，比如**–mount**和一个将**privileged**设置为**true**的安全上下文设置。所有这些选项的组合将允许我们写入主机的文件系统。

现在你在容器中，执行**ls**命令查看文件系统。注意提示符是**root@r00t:/#**，确认你在容器中而不是在主机上：

root@r00t:/# ls

**bin  boot  build  dev  etc  home  kind  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var**

为了证明我们已经将主机的文件系统映射到我们的容器中，创建一个名为**this is from a container**的文件，然后退出容器：

root@r00t:/# touch this_is_from_a_container

root@r00t:/# 退出

最后，让我们查看主机的文件系统，看看容器是否创建了文件。由于我们正在运行 KinD，只有一个工作节点，我们需要使用 Docker **exec**进入工作节点。如果您正在使用本书中的 KinD 集群，工作节点被称为**cluster01-worker**：

docker exec -ti cluster01-worker ls /

**bin  boot  build  dev  etc  home  kind  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  this_is_from_a_container  tmp  usr  var**

在这个例子中，运行了一个容器，挂载了本地文件系统。在 pod 内部，创建了**this_is_from_a_container**文件。退出 pod 并进入节点容器后，文件就在那里。一旦攻击者可以访问节点的文件系统，他们也可以访问 kubelet 的凭据，这可能会打开整个集群。

很容易想象一系列事件会导致比特币矿工（或更糟）在集群上运行。钓鱼攻击获取了开发人员用于他们集群的凭据。尽管这些凭据只能访问一个命名空间，但还是创建了一个容器来获取 kubelet 的凭据，然后启动容器在环境中秘密部署矿工。当然，有多种缓解措施可以用来防止这种攻击，包括以下措施：

+   多因素身份验证可以防止被钓鱼凭据被使用

+   只预授权特定容器

+   PSP 可以阻止容器以**privileged**身份运行，从而阻止这种攻击

+   一个经过适当保护的基本镜像

安全的核心是一个经过适当设计的镜像。对于物理机和虚拟机来说，这是通过保护基本操作系统来实现的。当你安装操作系统时，你不会在安装过程中选择每一个可能的选项。在服务器上运行任何不需要的东西被认为是不良实践。这种做法需要延续到将在集群上运行的镜像，它们应该只包含应用程序所需的必要二进制文件。

考虑到在集群上适当保护镜像的重要性，下一节将从安全的角度探讨容器设计。构建一个安全的容器可以更容易地管理节点的安全。

## 适当设计容器

在探讨如何构建**PodSecurityPolicy**之前，重要的是要解决容器的设计方式。通常，使用**PodSecurityPolicy**来减轻对节点的攻击最困难的部分在于，许多容器都是以 root 用户构建和运行的。一旦应用了受限策略，容器就会停止运行。这在多个层面上都是有问题的。系统管理员在几十年的网络计算中学到，不要以 root 用户身份运行进程，特别是那些通过不受信任的网络匿名访问的 web 服务器等服务。

注意

所有网络都应被视为“不受信任的”。假设所有网络都是敌对的，会导致更安全的实施方法。这也意味着需要安全性的服务需要进行身份验证。这个概念被称为零信任。身份专家多年来一直在使用和倡导这个概念，但是在 DevOps 和云原生领域，谷歌的 BeyondCorp 白皮书（[`cloud.google.com/beyondcorp`](https://cloud.google.com/beyondcorp)）使其更为流行。零信任的概念也应该适用于您的集群内部！

代码中的漏洞可能导致对底层计算资源的访问，然后可能从容器中突破出去。如果通过代码漏洞利用，不需要时以特权容器的 root 身份运行可能会导致突破。

2017 年的 Equifax 泄露事件利用了 Apache Struts web 应用程序框架中的一个漏洞，在服务器上运行代码，然后用于渗透和提取数据。如果这个有漏洞的 web 应用程序在 Kubernetes 上以特权容器运行，这个漏洞可能会导致攻击者获取对集群的访问权限。

构建容器时，至少应遵守以下规定：

+   **以非 root 用户身份运行**：绝大多数应用程序，特别是微服务，不需要 root 权限。不要以 root 用户身份运行。

+   **只写入卷**：如果不向容器写入，就不需要写入访问权限。卷可以由 Kubernetes 控制。如果需要写入临时数据，可以使用**emptyVolume**对象，而不是写入容器的文件系统。

+   **最小化容器中的二进制文件**：这可能有些棘手。有人主张使用“无发行版”的容器，只包含应用程序的二进制文件，静态编译。没有 shell，没有工具。当尝试调试应用程序为何不按预期运行时，这可能会有问题。这是一个微妙的平衡。

+   **扫描已知的常见漏洞暴露（CVE）的容器；经常重建**：容器的一个好处是可以轻松地扫描已知的 CVE。有几种工具和注册表可以为您执行此操作。一旦 CVE 已修补，就进行重建。几个月甚至几年没有重建的容器与未打补丁的服务器一样危险。

重要提示

扫描 CVE 是报告安全问题的标准方法。应用程序和操作系统供应商将使用修补程序更新 CVE，以修复问题。然后，安全扫描工具将使用此信息来处理已修补的已知问题的容器。

在撰写本文时，市场上任何 Kubernetes 发行版中最严格的默认设置属于红帽的 OpenShift。除了合理的默认策略外，OpenShift 会以随机用户 ID 运行 pod，除非 pod 定义指定了 ID。

在 OpenShift 上测试您的容器是个好主意，即使它不是您用于生产的发行版。如果一个容器能在 OpenShift 上运行，它很可能能够适用于集群可以应用的几乎任何安全策略。最简单的方法是使用红帽的 CodeReady Containers（[`developers.redhat.com/products/codeready-containers`](https://developers.redhat.com/products/codeready-containers)）。这个工具可以在您的本地笔记本电脑上运行，并启动一个可以用于测试容器的最小 OpenShift 环境。

注意

虽然 OpenShift 在出厂时具有非常严格的安全控制，但它不使用 PSP。它有自己的策略系统，早于 PSP，称为**安全上下文约束**（**SCCs**）。SCCs 类似于 PSP，但不使用 RBAC 与 pod 关联。

### PSP 详细信息

PSP 与 Linux 进程运行方式紧密相关。策略本身是任何 Linux 进程可能具有的潜在选项列表。

PSP 具有几个特权类别：

+   **特权**：pod 是否需要作为特权 pod 运行？pod 是否需要执行会更改底层操作系统或环境的操作？

+   **主机交互**：pod 是否需要直接与主机交互？例如，它是否需要主机文件系统访问？

+   **卷类型**：这个 pod 可以挂载什么类型的卷？您是否希望将其限制为特定卷，如密钥，而不是磁盘？

+   **用户上下文：**进程允许以哪个用户身份运行？除了确定允许的用户 ID 和组 ID 范围外，还可以设置 SELinux 和 AppArmor 上下文。

一个简单的非特权策略可能如下所示：

api 版本：policy/v1beta1

种类：PodSecurityPolicy

元数据：

名称：pod-security-policy-default

规范：

fsGroup：

规则：'必须以此身份运行'

范围：

# 禁止添加根组。

- 最小值：1

最大值：65535

runAsUser：

规则：'必须以此身份运行'

范围：

# 禁止添加根组。

- 最小值：1

最大值：65535

seLinux：

规则：RunAsAny

辅助组：

规则：'必须以此身份运行'

范围：

# 禁止添加根组。

- 最小值：1

最大值：65535

卷：

- 空目录

- 机密

- 配置映射

- 持久卷索赔

规范中没有提到容器是否可以具有特权，也没有提到可以访问主机的任何资源。这意味着如果 pod 定义尝试直接挂载主机的文件系统或以 root 身份启动，pod 将失败。必须显式启用任何权限，以便 pod 使用它们。

该策略限制了 pod 可以以任何用户身份运行，除了 root 之外，还指定了**MustRunAs**选项，该选项设置为**1**和**65535**之间；不包括用户 0（root）。

最后，该策略允许挂载大多数 pod 可能需要的标准卷类型。很少有 pod 需要能够挂载节点的文件系统。

如果有这样的策略，我们之前用来访问节点文件系统的突破将被阻止。以下是我们之前尝试运行的 pod 的 YAML：

---

规范：

**hostPID：true**

容器：

- 名称：'1'

镜像：alpine

命令：

- nsenter

- "--mount=/proc/1/ns/mnt"

- "--"

- "/bin/bash"

标准输入：true

tty：true

镜像拉取策略：IfNotPresent

**安全上下文：**

**      特权：true**

有两个突出显示的设置。第一个是**hostPID**，它允许 pod 与节点共享进程 ID 空间。Linux 内核用于启用容器的技术之一是 cgroups，它隔离容器中的进程。在 Linux 中，cgroups 将为容器中的进程提供与在节点上简单运行时不同的进程 ID。如所示，可以从节点查看所有容器的进程。从 pod 内部运行**ps -A -elf | grep java**将得到与来自节点的不同 ID。由于我们的策略上没有将**hostPID**选项设置为**true**，**PodSecurityPolicy**执行 webhook 将拒绝此 pod：

![图 10.2-来自主机和容器内的进程 ID](img/Fig_10.2_B15514.jpg)

图 10.2-来自主机和容器内的进程 ID

下一个突出显示的部分是将安全上下文设置为 true 的特权。这两个设置将允许容器运行，就好像它是作为根用户登录到节点一样。再次强调，默认的 PSP 会阻止这一点，因为特权未启用。PSP 控制器会阻止它。

接下来，查看 NGINX Ingress 控制器从[`raw.githubusercontent.com/kubernetes/ingress-nginx/master/docs/examples/psp/psp.yaml`](https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/docs/examples/psp/psp.yaml)推荐的 PSP：

apiVersion: policy/v1beta1

kind: PodSecurityPolicy

元数据：

。

。

.spec:

**allowedCapabilities:**

**  - NET_BIND_SERVICE**

**  allowPrivilegeEscalation: true**

。

。

。

hostPID: false

**hostPorts:**

**  - min: 80**

**    max: 65535**

在运行在主机上的典型 Web 服务器中，进程将以 root（或至少是特权用户）启动，然后降级为非特权用户，以便它可以打开端口 80 和 443 用于 HTTP 和 HTTPS。这些端口在 Linux 中保留给 root 进程，因此位于 1024 以下。

如果你想知道在 Kubernetes 中是否需要能够在端口 80 或 443 上运行 Web 服务器，实际上并不需要。正如本书前面讨论的那样，绝大多数部署都有一个负载均衡器在它们前面，可以将 80 和 443 端口映射到任何端口。这应该真的是一个例外，而不是规则。NGINX Ingress 控制器是在安全性在 Kubernetes 中并不像今天这样重要的时候发布的。此外，部署模型并不是那么成熟。

为了允许类似于 NGINX Web 服务器直接在主机上运行的行为，NGINX 希望能够从 80 端口开始打开端口，并升级为特权用户，具体使用 NET_BIND_SERVICE 特权，以便 Web 服务器可以在不以 root 身份运行整个进程的情况下打开端口 80 和 443。

正如之前讨论的，绝大多数容器不需要特殊权限。获得这些特殊权限的情况应该很少，并且只应该为特定用例保留。在评估可能在集群上运行的系统时，重要的是要看供应商或项目是否提供了经过测试的 PSP。如果没有，就假设它是无特权的，并使用本章后面讨论的工具来调试特定策略。

### 分配 PSP

设计策略后，需要进行分配。这通常是部署 PSP 最困难的部分。确定 PSP 是否应用于 Pod 的机制是两组权限的并集：

+   **提交 Pod 的用户**：这可能会变得棘手，因为用户很少直接提交 Pod。最佳做法是创建一个**Deployment**或**StatefulSet**。控制器然后创建 Pod（虽然不是直接创建）。"创建"Pod 的用户是正确的控制器服务账户，而不是提交**Deployment**或**StatefulSet**的用户。这意味着通常只有一个或两个服务账户实际上创建 Pod。

+   **Pod 运行的服务账户**：每个 Pod 可以定义一个服务账户来运行。这个服务账户的范围是在 Pod 级别上，而不是在单个容器上。

通过"并集"，Kubernetes 将结合这些权限来确定允许哪些功能。例如，如果提交 Pod 的控制器服务账户没有特权，但 Pod 的服务账户可以以 root 身份运行，那么将选择应用于 Pod 的*最佳*策略，允许 Pod 以 root 身份运行。这个过程可能令人困惑和难以调试，并且经常会产生意想不到的结果。Pod 不能直接请求策略；它必须被分配。将策略限制在一定范围内是很重要的，这样更有可能应用正确的策略。

策略是使用特殊的 RBAC 对象进行评估和应用的。就像创建用于授权对 API 的访问的策略对象一样，需要创建**Role**/**ClusterRole**和**RoleBinding**/**ClusterRoleBinding**。RBAC 对象不适用于特定的 API，而是适用于**PodSecurityPolicy**对象的**apiGroups**、PSPs 的资源和**use**动词。**use**动词没有任何对应的 HTTP 动作。绑定对象通常与授权 API 使用时相同，但主体通常是服务账户，而不是用户。

先前创建的第一个策略是一个很好的通用最低访问策略。要在整个集群中应用它，首先创建一个**ClusterRole**：

api 版本：rbac.authorization.k8s.io/v1

类型：ClusterRole

元数据：

名称：default-psp

规则：

- api 组：

- 策略

资源名称：

- **pod-security-policy-default**

资源：

- podsecuritypolicies

动词：

- use

**resourceNames**部分是特定于所引用的 PSP 的策略的唯一部分。策略中的其他所有内容都是样板文件。**ClusterRoleBinding**将在整个集群中应用这一点：

api 版本：rbac.authorization.k8s.io/v1

类型：ClusterRoleBinding

元数据：

名称：default-psp

roleRef：

api 组：rbac.authorization.k8s.io

类型：ClusterRole

名称：default-psp

主体：

- api 组：rbac.authorization.k8s.io

类型：组

名称：system:authenticated

当创建新的 pod 时，如果没有其他策略适用，则将使用受限策略。

注意

如果您来自 OpenShift 生态系统并且习惯使用 SCCs，则授权过程是不同的。SCCs 包含了直接授权对象的信息，而**PodSecurityPolicy**对象依赖于 RBAC。

# 它们不会消失吗？

2018 年发布 Kubernetes 1.11 时，人们发现 PSPs 可能永远不会成为**通用可用性**（**GA**）。这一发现是基于 PSPs 难以使用以及设计上的系统性问题的反馈。这一发现引发的讨论集中在三个潜在的解决方案上：

+   **修复 PSPs/重新实施新标准**：这两个选项被捆绑在一起，因为人们认为“修复”PSPs 将导致一个打破向后兼容性的标准，从而导致一个新的策略系统。另一个被提出的选项是将 OpenShift 的 SCC 实现移植到上游。

+   **移除 PSPs**：有人认为这应该是特定于实施的，因此由实施者决定。由于 PSP 是使用准入控制器实施的，有人认为这可以留给第三方。

+   **提供“基本”实现**：这是一种混合方法，其中上游 Kubernetes 构建支持 PSP 的子集，并依赖于自定义准入控制器来支持更高级的实现。

目前还没有明确的偏爱方向。已经明确的是，直到有替代方案普遍可用之后，PSPs 才不会被弃用和移除。随着 Kubernetes 1.19 的推出，不允许 API 在 alpha 或 beta 模式下超过三个版本的新政策迫使**PodSecurityPolicy** API 被弃用。该 API 直到 1.22 版本才会被移除，而这至少要等到 2023 年 1 月发布（假设每次发布之间至少有 6 个月的时间）。

有多种方法可以保护免受 PSP 最终被弃用的影响：

+   **完全不使用它们**：这不是一个好方法。这会让集群的节点处于开放状态。

+   **避免临时政策**：自动化政策应用过程将使迁移到 PSP 替代方案更容易。

+   **使用其他技术**：有其他 PSP 实现选项，将在*替代 PSPs*部分进行介绍。

根据您的实施需求对 PSP 做出决定。要了解 PSP 的进展，请关注 GitHub 上的问题：[`github.com/kubernetes/enhancements/issues/5`](https://github.com/kubernetes/enhancements/issues/5)。

# 启用 PSPs

启用 PSPs 非常简单。将**PodSecurityPolicy**添加到 API 服务器的准入控制器列表中，将所有新创建的 Pod 对象发送到**PodSecurityPolicy**准入控制器。该控制器有两个作用：

+   **确定最佳策略**：所请求的功能决定了要使用的最佳策略。一个 Pod 不能明确说明它想要强制执行哪个策略，只能说明它想要什么功能。

+   **确定 Pod 的策略是否被授权**：一旦确定了策略，准入控制器需要确定 Pod 的创建者或 Pod 的**serviceAccount**是否被授权使用该策略。

这两个标准的结合可能导致意想不到的结果。创建 pod 的人不是提交**Deployment**或**StatefulSet**定义的用户。有一个控制器监视**Deployment**更新并创建**ReplicaSet**。有一个控制器监视**ReplicaSet**对象并创建（**Pod**）对象。因此，需要授权的不是创建**Deployment**的用户，而是**ReplicaSet**控制器的**serviceAccount**。通常，博客文章和许多默认配置会将特权策略分配给**kube-system**命名空间中的所有**ServiceAccount**对象。这包括**ReplicaSet**控制器运行的**ServiceAccount**，这意味着它可以创建一个具有特权 PSP 的 pod，而不需要**Deployment**的创建者或 pod 的**serviceAccount**被授权这样做。向您的供应商施压，要求他们提供经过测试的经认证的 PSP 定义非常重要。

在启用准入控制器之前，首先创建初始策略非常重要。从[`raw.githubusercontent.com/PacktPublishing/Kubernetes-and-Docker-The-Complete-Guide/master/chapter10/podsecuritypolicies.yaml`](https://raw.githubusercontent.com/PacktPublishing/Kubernetes-and-Docker-The-Complete-Guide/master/chapter10/podsecuritypolicies.yaml)获取的策略集包括两个策略和相关的 RBAC 绑定。第一个策略是在本章前面描述的非特权策略。第二个策略是一个特权策略，分配给**kube-system**命名空间中的大多数**ServiceAccount**对象。**ReplicaSet**控制器的**ServiceAccount**没有被分配访问特权策略。如果一个**Deployment**需要创建一个特权 pod，pod 的**serviceAccount**将需要通过 RBAC 授权来使用特权策略。第一步是应用这些策略；策略文件位于您克隆的存储库的**chapter10**文件夹中：

1.  进入**chapter10**文件夹，并使用**kubectl**创建 PSP 对象：

kubectl create -f podsecuritypolicies.yaml

**podsecuritypolicy.policy/pod-security-policy-default created**

**clusterrole.rbac.authorization.k8s.io/default-psp created**

**clusterrolebinding.rbac.authorization.k8s.io/default-psp created**

**podsecuritypolicy.policy/privileged created**

**clusterrole.rbac.authorization.k8s.io/privileged-psp created**

创建了 **rolebinding.rbac.authorization.k8s.io/kube-system-psp**

1.  一旦策略被创建，**docker exec** 进入控制平面容器并编辑 **/etc/kubernetes/manifests/kube-apiserver.yaml**。查找 **- --enable-admission-plugins=NodeRestriction** 并将其更改为 **- --enable-admission plugins=PodSecurityPolicy,NodeRestriction**。一旦 API 服务器 pod 重新启动，所有新的和更新的 pod 对象将通过 **PodSecurityPolicy** 准入控制器。

注意

托管的 Kubernetes 提供通常预先配置了 **PodSecurityPolicy** 准入控制器。所有 pod 都被授予特权访问，所以一切都 "正常工作"。启用 PSPs 只是创建策略和 RBAC 规则，但不显式启用它们。

1.  由于策略是通过准入控制器强制执行的，任何启动的 pod 如果没有访问特权策略，将继续运行。例如，NGINX Ingress 控制器仍在运行。检查任何使用 **kubectl describe** 的 pod 的注释将显示没有注释指定使用的策略。为了将策略应用到所有正在运行的 pod，它们必须全部被删除：

kubectl delete pods --all-namespaces --all

删除了 "nginx-ingress-controller-7d6bf88c86-q9f2j" pod

删除了 "calico-kube-controllers-5b644bc49c-8lkvs" pod

删除了 "calico-node-r6vwk" pod

删除了 "calico-node-r9ck9" pod

删除了 "coredns-6955765f44-9vw6t" pod

删除了 "coredns-6955765f44-qrcss" pod

删除了 "etcd-cluster01-control-plane" pod

删除了 "kube-apiserver-cluster01-control-plane" pod

删除了 "kube-controller-manager-cluster01-control-plane" pod

删除了 "kube-proxy-n2xf6" pod

删除了 "kube-proxy-tkxh6" pod

删除了 "kube-scheduler-cluster01-control-plane" pod

删除了 "dashboard-metrics-scraper-c79c65bb7-vd2k8" pod

删除了 "kubernetes-dashboard-6f89967466-p7rv5" pod

删除了 "local-path-provisioner-7745554f7f-lklmf" pod

删除了 "openunison-operator-858d496-zxnmj" pod

删除了 "openunison-orchestra-57489869d4-btkvf" pod

这将需要一些时间来运行，因为集群需要重建自身。从 etcd 到网络，所有都在重建它们的 pod。命令完成后，观察所有的 pod 确保它们恢复。

1.  一旦所有 **Pod** 对象恢复，查看 OpenUnison pod 的注释：

kubectl describe pod -l application=openunison-orchestra -n openunison

名称：openunison-orchestra-57489869d4-jmbk2

命名空间：openunison

优先级：0

节点：cluster01-worker/172.17.0.3

开始时间：Thu, 11 Jun 2020 22:57:24 -0400

标签：application=openunison-orchestra

operated-by=openunison-operator

pod-template-hash=57489869d4

注释：cni.projectcalico.org/podIP: 10.240.189.169/32

cni.projectcalico.org/podIPs: 10.240.189.169/32

kubernetes.io/psp: pod-security-policy-default

突出显示的注释显示 OpenUnison 正在使用默认的受限策略运行。

1.  当 OpenUnison 正在运行时，尝试登录将失败。NGINX Ingress 的 pod 没有运行。正如我们在本章前面讨论的那样，NGINX 需要能够打开端口 443 和 80，但使用默认策略不允许这种情况发生。通过检查 ingress-nginx 命名空间中的事件来确认 NGINX 为什么没有运行：

$ kubectl get events -n ingress-nginx

2m4s 警告 FailedCreate replicaset/nginx-ingress-controller-7d6bf88c86 创建错误：pods "nginx-ingress-controller-7d6bf88c86-" 被禁止：无法根据任何 pod 安全策略进行验证：[spec.containers[0].securityContext.capabilities.add: Invalid value: "NET_BIND_SERVICE": capability may not be added spec.containers[0].hostPort: Invalid value: 80: Host port 80 is not allowed to be used. Allowed ports: [] spec.containers[0].hostPort: Invalid value: 443: Host port 443 is not allowed to be used. Allowed ports: []]

1.  即使 NGINX Ingress 项目提供了策略和 RBAC 绑定，让我们假设没有这些来调试。检查 Deployment 对象，规范中的关键块如下：

端口：

- containerPort: 80

主机端口：80

名称：http

协议：TCP

- containerPort: 443

hostPort: 443

名称：https

协议：TCP

。

。

。

securityContext:

allowPrivilegeEscalation: true

功能：

添加：

- NET_BIND_SERVICE

drop：

- ALL

runAsUser: 101

首先，pod 声明要打开端口 80 和 443。接下来，它的 securityContext 声明要进行特权升级，并且要求 NET_BIND_SERVICE 功能以在不作为 root 的情况下打开这些端口。

1.  类似于调试 RBAC 策略时使用的 audit2rbac 工具，Sysdig 发布了一个工具，将检查命名空间中的 pod 并生成推荐的策略和 RBAC 集。从[`github.com/sysdiglabs/kube-psp-advisor/releases`](https://github.com/sysdiglabs/kube-psp-advisor/releases)下载最新版本：

./kubectl-advise-psp inspect --namespace=ingress-nginx

**apiVersion: policy/v1beta1**

**kind: PodSecurityPolicy**

**metadata:**

** creationTimestamp: null**

** name: pod-security-policy-ingress-nginx-20200611232031**

**spec:**

** defaultAddCapabilities:**

** - NET_BIND_SERVICE**

** fsGroup:**

** rule: RunAsAny**

** hostPorts:**

** - max: 80**

** min: 80**

** - max: 443**

** min: 443**

** requiredDropCapabilities:**

** - ALL**

** runAsUser:**

** ranges:**

** - max: 101**

** min: 101**

** rule: MustRunAs**

** seLinux:**

** rule: RunAsAny**

** supplementalGroups:**

** rule: RunAsAny**

** volumes:**

** - secret**

将此策略与本章前面检查过的 NGINX Ingress 项目提供的策略进行比较；您会发现它在端口和用户上更加严格，但在组上不那么严格。**Deployment**声明了用户但没有声明组，因此**kube-psp-advisor**不知道要对其进行限制。与**audit2rbac**不同，**kube-psp-advisor**不是在扫描日志以查看被拒绝的内容；它是积极地检查 pod 定义以创建策略。如果一个 pod 没有声明需要以 root 身份运行，而只是启动一个以 root 身份运行的容器，那么**kube-psp-advisor**将不会生成适当的策略。

1.  从**kube-psp-advisor**创建名为**psp-ingress.yaml**的策略文件：

**$ ./kubectl-advise-psp inspect --namespace=ingress-nginx > psp-ingress.yaml**

1.  使用**kubectl**部署 PSP：

**$ kubectl create -f ./psp-ingress.yaml -n ingress-nginx**

1.  接下来，为**nginx-ingress-serviceaccount ServiceAccount**（在部署中引用）创建 RBAC 绑定，以便访问此策略：

apiVersion: rbac.authorization.k8s.io/v1

kind: Role

metadata:

name: nginx-ingress-psp

namespace: ingress-nginx

rules:

- apiGroups:

- policy

resourceNames:

- pod-security-policy-ingress-nginx-20200611232826

resources:

- podsecuritypolicies

verbs:

- use

---

apiVersion: rbac.authorization.k8s.io/v1

kind: RoleBinding

metadata:

name: nginx-ingress-psp

namespace: ingress-nginx

roleRef:

apiGroup: rbac.authorization.k8s.io

kind: Role

name: nginx-ingress-psp

subjects:

- kind: ServiceAccount

name: nginx-ingress-serviceaccount

namespace: ingress-nginx

1.  一旦 RBAC 对象创建完成，需要更新部署以强制 Kubernetes 尝试重新创建 pod，因为 API 服务器在一定时间后将停止尝试：

$ kubectl scale deployment.v1.apps/nginx-ingress-controller --replicas=0 -n ingress-nginx

deployment.apps/nginx-ingress-controller scaled

$ kubectl scale deployment.v1.apps/nginx-ingress-controller --replicas=1 -n ingress-nginx

deployment.apps/nginx-ingress-controller scaled

$ kubectl get pods -n ingress-nginx

名称                                      准备就绪    状态      重启次数    年龄

nginx-ingress-controller-7d6bf88c86-h4449   0/1     Running   0          21s

如果您检查 Pod 上的注释，将会有**PodSecurityPolicy**注释，并且 OpenUnison 将再次可访问。

注意

使用 RBAC 来控制 PSP 授权的一个副作用是，命名空间中的管理员可以创建可以运行特权容器的**ServiceAccount**对象。在允许命名空间管理员在其命名空间中创建 RBAC 策略的同时停止这种能力将在下一章中讨论。

恭喜，您已成功在集群上实施了 PSPs！尝试运行我们在本章早些时候运行的突破代码，您会发现它不起作用。**Pod**甚至不会启动！看到 NGINX Ingress 控制器无法启动并对其进行调试，使您能够理解如何在启用策略执行后解决问题。

# PSP 的替代方案

如果不是 PSPs，那又是什么呢？这实际上取决于集群的用例。已经有人尝试在 OPA 中实现完整的**PodSecurityPolicy**执行规范，这将在下一章中更详细地讨论。其他几个项目尝试实现 PSPs，即使不是**PodSecurityPolicy**对象的确切规范。鉴于这个领域的变化如此迅速，本章不会列举所有试图做到这一点的项目。

2020 年 5 月，认证特别兴趣小组（sig-auth）发布了*pod 安全标准*文档，以便不同的安全策略实现能够统一词汇和命名。这些标准已经发布在 Kubernetes 网站上（[`kubernetes.io/docs/concepts/security/pod-security-standards/`](https://kubernetes.io/docs/concepts/security/pod-security-standards/)）。

在自己的准入控制器作为验证 webhook 中实现这个逻辑时要小心。就像任何安全实现一样，需要非常小心，不仅要验证预期的结果，还要确保意外情况得到预期的处理。例如，如果使用**Deployment**来创建**Pod**与直接创建**Pod**有什么不同？当有人试图向定义中注入无效数据时会发生什么？或者有人试图创建一个 side car 或一个**init**容器时会发生什么？在选择方法时，重要的是要确保任何实现都有一个彻底的测试环境。

# 总结

在本章中，我们首先探讨了保护节点的重要性，从安全角度讨论了容器和 VM 之间的区别，以及在节点没有受到保护时容易利用集群的情况。我们还研究了安全的容器设计，最后实施和调试了 PSP 实现。

锁定集群节点提供了一个更少的攻击向量。封装策略使得更容易向开发人员解释如何设计他们的容器，并更容易构建安全的解决方案。

到目前为止，我们所有的安全性都是基于 Kubernetes 的标准技术构建的，几乎在所有 Kubernetes 发行版中都是通用的。在下一章中，我们将通过动态准入控制器和 OPA 来应用超出 Kubernetes 范围的策略。

# 问题

1.  容器是“轻量级 VM”——真还是假？

A. 真

B. 假

1.  容器能否访问其主机的资源？

A. 不，它是隔离的。

B. 如果标记为特权，是的。

C. 只有在策略明确授予的情况下。

D. 有时候。

1.  攻击者如何通过容器获得对集群的访问权限？

A. 容器应用程序中的错误可能导致远程代码执行，这可能被用来打破容器的漏洞，然后用来获取 kubelet 的凭证。

B. 具有创建一个命名空间中容器的能力的受损凭证可以用来创建一个挂载节点文件系统以获取 kubelet 凭证的容器。

C. 以上两者都是。

1.  **PodSecurityPolicy**准入控制器如何确定要应用于 pod 的策略？

A. 通过读取 pod 定义中的注释

B. 通过比较 pod 的请求能力和通过 pod 的创建者和其自己的**ServiceAccount**授权的策略的并集。

C. 通过比较 Pod 请求的功能和为其自己的**ServiceAccount**授权的策略

D. 通过比较 Pod 请求的功能和为 Pod 创建者授权的策略

1.  是什么机制执行了 PSPs？

A. 一个审批控制器，在创建和更新时检查所有的 Pods

B. **PodSecurityPolicy** API

C. OPA

D. Gatekeeper

1.  真或假 - **PodSecurityPolicy** API 将很快被移除。

A. 真

B. 假

1.  真或假 - 容器通常应该以 root 用户身份运行。

A. 真

B. 假

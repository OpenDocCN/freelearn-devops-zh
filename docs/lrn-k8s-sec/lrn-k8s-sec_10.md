# 第八章：保护 Kubernetes Pods

尽管 pod 是作为运行微服务的最细粒度单位，保护 Kubernetes pods 是一个广泛的主题，因为它应该涵盖整个 DevOps 流程：构建、部署和运行。

在本章中，我们选择将焦点缩小到构建和运行阶段。为了在构建阶段保护 Kubernetes pods，我们将讨论如何加固容器镜像并配置 pod（或 pod 模板）的安全属性，以减少攻击面。虽然一些工作负载的安全属性，如 AppArmor 和 SELinux 标签，会在运行阶段生效，但安全控制已经为工作负载定义好了。为了进一步澄清问题，我们试图通过在构建阶段配置运行效果的安全属性来保护 Kubernetes 工作负载。为了在运行阶段保护 Kubernetes pods，我们将介绍一个带有示例的 PodSecurityPolicy 以及辅助工具`kube-psp-advisor`。

后续章节将更详细地讨论运行时安全和响应。还要注意，应用程序的利用可能导致 pod 被 compromise。但是，我们不打算在本章中涵盖应用程序。

在本章中，我们将涵盖以下主题：

+   容器镜像的加固

+   配置 pod 的安全属性

+   PodSecurityPolicy 的威力

# 容器镜像的加固

容器镜像的加固意味着遵循安全最佳实践或基线，以配置容器镜像，以减少攻击面。镜像扫描工具只关注在镜像内捆绑的应用程序中找到的公开披露的问题。但是，在构建镜像时遵循最佳实践以及安全配置，可以确保应用程序具有最小的攻击面。

在我们开始讨论安全配置基线之前，让我们看看容器镜像是什么，以及 Dockerfile 是什么，以及它是如何用来构建镜像的。

## 容器镜像和 Dockerfile

一个**容器镜像**是一个文件，它捆绑了微服务二进制文件、它的依赖项和微服务的配置等。一个容器是镜像的运行实例。如今，应用程序开发人员不仅编写代码来构建微服务；他们还需要构建 Dockerfile 来将微服务容器化。为了帮助构建容器镜像，Docker 提供了一种标准化的方法，称为 Dockerfile。一个**Dockerfile**包含一系列的指令，比如复制文件、配置环境变量、配置开放端口和容器入口点，这些指令可以被 Docker 守护进程理解以构建镜像文件。然后，镜像文件将被推送到镜像注册表，然后从那里部署到 Kubernetes 集群中。每个 Dockerfile 指令都会在镜像中创建一个文件层。

在我们看一个 Dockerfile 的例子之前，让我们先了解一些基本的 Dockerfile 指令：

+   **FROM**：从基础镜像或父镜像初始化一个新的构建阶段。两者都意味着你正在捆绑自己的镜像的基础或文件层。

+   **RUN**：执行命令并将结果提交到上一个文件层之上。

+   **ENV**：为运行的容器设置环境变量。

+   **CMD**：指定容器将运行的默认命令。

+   **COPY/ADD**：这两个命令都是将文件或目录从本地（或远程）URL 复制到镜像的文件系统中。

+   **EXPOSE**：指定微服务在容器运行时将监听的端口。

+   **ENTRYPOINT**：类似于`CMD`，唯一的区别是`ENTRYPOINT`会使容器作为可执行文件运行。

+   **WORKDIR**：为接下来的指令设置工作目录。

+   **USER**：为容器的`CMD`/`ENTRYPOINT`设置用户和组 ID。

现在，让我们来看一个 Dockerfile 的例子：

[PRE0]

从上面的 Dockerfile 中，我们可以看出这个镜像是基于`ubuntu`构建的。然后，它运行了一系列的`apt-get`命令来安装依赖，并创建了一个名为`/var/www`的目录。接下来，将`app.js`文件从当前目录复制到镜像文件系统中的`/var/www/app.js`。最后，配置默认命令来运行这个`Node.js`应用程序。我相信当你开始构建镜像时，你会看到 Dockerfile 是多么简单和强大。

下一个问题是安全问题，因为看起来您可以构建任何类型的图像。接下来，让我们谈谈 CIS Docker 基准。

## CIS Docker 基准

互联网安全中心（CIS）制定了有关 Docker 容器管理和管理的指南。现在，让我们来看看 CIS Docker 基准关于容器图像的安全建议：

+   为容器图像创建一个用户来运行微服务：以非 root 用户运行容器是一个好的做法。虽然用户命名空间映射是可用的，但默认情况下未启用。以 root 身份运行意味着如果攻击者成功逃离容器，他们将获得对主机的 root 访问权限。在 Dockerfile 中使用`USER`指令创建一个用户。

+   使用受信任的基础图像构建您自己的图像：从公共存储库下载的图像不能完全信任。众所周知，来自公共存储库的图像可能包含恶意软件或加密货币挖矿程序。因此，建议您从头开始构建图像或使用最小的受信任图像，如 Alpine。此外，在构建图像后执行图像扫描。图像扫描将在下一章节中介绍。

+   不要在图像中安装不必要的软件包：安装不必要的软件包会增加攻击面。建议保持图像的精简。在构建图像的过程中，您可能需要安装一些工具。请记住在 Dockerfile 的末尾将它们删除。

+   扫描并重建图像以应用安全补丁：很可能会在基础图像或图像中安装的软件包中发现新的漏洞。经常扫描图像是一个好的做法。一旦发现任何漏洞，尝试通过重建图像来修补安全漏洞。图像扫描是在构建阶段识别漏洞的关键机制。我们将在下一章节详细介绍图像扫描。

+   为 Docker 启用内容信任：内容信任使用数字签名确保客户端和 Docker 注册表之间的数据完整性。它确保容器图像的来源。但默认情况下未启用。您可以通过将环境变量`DOCKER_CONTENT_TRUST`设置为`1`来启用它。

+   **向容器图像添加 HEALTHCHECK 指令**：`HEALTHCHECK`指令定义了一个命令，要求 Docker 引擎定期检查容器的健康状态。根据健康状态检查结果，Docker 引擎然后退出不健康的容器并启动一个新的容器。

+   **确保 Dockerfile 中的更新不被缓存**：根据您选择的基础镜像，您可能需要在安装新软件包之前更新软件包存储库。但是，如果您在 Dockerfile 中的单行中指定`RUN apt-get update``(Debian)`，Docker 引擎将缓存此文件层，因此，当您再次构建图像时，它仍将使用缓存的旧软件包存储库信息。这将阻止您在图像中使用最新的软件包。因此，要么在单个 Dockerfile 指令中同时使用`update`和`install`，要么在 Docker`build`命令中使用`--no-cache`标志。

+   **从图像中删除 setuid 和 setgid 权限**：`setuid`和`setgid`权限可用于特权升级，因为具有这些权限的文件允许以所有者特权而不是启动器特权执行。您应该仔细审查具有`setuid`和`setgid`权限的文件，并删除不需要此类权限的文件。

+   **在 Dockerfile 中使用 COPY 而不是 ADD**：`COPY`指令只能将文件从本地计算机复制到图像的文件系统，而`ADD`指令不仅可以从本地计算机复制文件，还可以从远程 URL 检索文件到图像的文件系统。使用`ADD`可能会引入从互联网添加恶意文件的风险。

+   **不要在 Dockerfile 中存储机密信息**：有许多工具可以提取图像文件层。如果图像中存储了任何机密信息，那么这些机密信息就不再是机密信息。在 Dockerfile 中存储机密信息会使容器有潜在的可利用性。一个常见的错误是使用`ENV`指令将机密信息存储在环境变量中。

+   **仅安装经过验证的软件包**：这类似于仅使用受信任的基础镜像。在安装图像内的软件包时要小心，确保它们来自受信任的软件包存储库。

如果您遵循前述 CIS Docker 基准的安全建议，您将成功地加固容器镜像。这是在构建阶段保护 pod 的第一步。现在，让我们看看我们需要注意的安全属性，以确保 pod 的安全。

# 配置 pod 的安全属性

正如我们在前一章中提到的，应用程序开发人员应该知道微服务必须具有哪些特权才能执行任务。理想情况下，应用程序开发人员和安全工程师应该共同努力，通过配置 Kubernetes 提供的安全上下文来加固 pod 和容器级别的微服务。

我们将主要的安全属性分为四类：

+   为 pod 设置主机命名空间

+   容器级别的安全上下文

+   Pod 级别的安全上下文

+   AppArmor 配置文件

通过采用这种分类方式，您会发现它们易于管理。

## 为 pod 设置主机级别的命名空间

在 pod 规范中使用以下属性来配置主机命名空间的使用：

+   **hostPID**：默认情况下为`false`。将其设置为`true`允许 pod 在工作节点上看到所有进程。

+   **hostNetwork**：默认情况下为`false`。将其设置为`true`允许 pod 在工作节点上看到所有网络堆栈。

+   **hostIPC**：默认情况下为`false`。将其设置为`true`允许 pod 在工作节点上看到所有 IPC 资源。

以下是如何在`ubuntu-1` pod 的`YAML`文件中配置在 pod 级别使用主机命名空间的示例：

[PRE1]

前面的工作负载 YAML 配置了`ubuntu-1` pod 以使用主机级 PID 命名空间、网络命名空间和 IPC 命名空间。请记住，除非必要，否则不应将这些属性设置为`true`。将这些属性设置为`true`还会解除同一工作节点上其他工作负载的安全边界，正如在*第五章*中已经提到的，*配置 Kubernetes 安全边界*。

## 容器的安全上下文

多个容器可以被分组放置在同一个 Pod 中。每个容器可以拥有自己的安全上下文，定义特权和访问控制。在容器级别设计安全上下文为 Kubernetes 工作负载提供了更精细的安全控制。例如，您可能有三个容器在同一个 Pod 中运行，其中一个必须以特权模式运行，而其他的以非特权模式运行。这可以通过为各个容器配置安全上下文来实现。

以下是容器安全上下文的主要属性：

+   **privileged**: 默认情况下为`false`。将其设置为`true`实质上使容器内的进程等同于工作节点上的 root 用户。

+   **能力**: 容器运行时默认授予容器的一组能力。默认授予的能力包括：`CAP_SETPCAP`、`CAP_MKNOD`、`CAP_AUDIT_WRITE`、`CAP_CHOWN`、`CAP_NET_RAW`、`CAP_DAC_OVERRIDE`、`CAP_FOWNER`、`CAP_FSETID`、`CAP_KILL`、`CAP_SETGID`、`CAP_SETUID`、`CAP_NET_BIND_SERVICE`、`CAP_SYS_CHROOT`和`CAP_SETFCAP`。

您可以通过配置此属性添加额外的能力或删除一些默认的能力。诸如`CAP_SYS_ADMIN`和`CAP_NETWORK_ADMIN`之类的能力应谨慎添加。对于默认的能力，您还应该删除那些不必要的能力。

+   **allowPrivilegeEscalation**: 默认情况下为`true`。直接设置此属性可以控制`no_new_privs`标志，该标志将设置为容器中的进程。基本上，此属性控制进程是否可以获得比其父进程更多的特权。请注意，如果容器以特权模式运行，或者添加了`CAP_SYS_ADMN`能力，此属性将自动设置为`true`。最好将其设置为`false`。

+   **readOnlyRootFilesystem**: 默认情况下为`false`。将其设置为`true`会使容器的根文件系统变为只读，这意味着库文件、配置文件等都是只读的，不能被篡改。将其设置为`true`是一个良好的安全实践。

+   runAsNonRoot：默认情况下为`false`。将其设置为`true`可以启用验证，以确保容器中的进程不能以 root 用户（UID=0）身份运行。验证由`kubelet`执行。将`runAsNonRoot`设置为`true`后，如果以 root 用户身份运行，`kubelet`将阻止容器启动。将其设置为`true`是一个良好的安全实践。这个属性也可以在`PodSecurityContext`中使用，在 Pod 级别生效。如果在`SecurityContext`和`PodSecurityContext`中都设置了这个属性，那么在容器级别指定的值优先。

+   runAsUser：这是用来指定容器镜像入口进程运行的 UID。默认设置是镜像元数据中指定的用户（例如，Dockerfile 中的`USER`指令）。这个属性也可以在`PodSecurityContext`中使用，在 Pod 级别生效。如果在`SecurityContext`和`PodSecurityContext`中都设置了这个属性，那么在容器级别指定的值优先。

+   runAsGroup：类似于`runAsUser`，这是用来指定容器入口进程运行的**Group ID**或**GID**。这个属性也可以在`PodSecurityContext`中使用，在 Pod 级别生效。如果在`SecurityContext`和`PodSecurityContext`中都设置了这个属性，那么在容器级别指定的值优先。

+   seLinuxOptions：这是用来指定容器的 SELinux 上下文的。默认情况下，如果未指定，容器运行时将为容器分配一个随机的 SELinux 上下文。这个属性也可以在`PodSecurityContex`中使用，在 Pod 级别生效。如果在`SecurityContext`和`PodSecurityContext`中都设置了这个属性，那么在容器级别指定的值优先。

既然你现在了解了这些安全属性是什么，你可以根据自己的业务需求提出自己的加固策略。一般来说，安全最佳实践如下：

+   除非必要，不要以特权模式运行。

+   除非必要，不要添加额外的能力。

+   丢弃未使用的默认能力。

+   以非 root 用户身份运行容器。

+   启用`runAsNonRoot`检查。

+   将容器根文件系统设置为只读。

现在，让我们看一个为容器配置`SecurityContext`的示例：

[PRE2]

nginx-pod 内的`nginx`容器以 UID 为`100`和 GID 为`1000`的用户身份运行。除此之外，`nginx`容器还获得了额外的`NETWORK_ADMIN`权限，并且根文件系统被设置为只读。这里的 YAML 文件只是展示了如何配置安全上下文的示例。请注意，在生产环境中运行的容器中不建议添加`NETWORK_ADMIN`。

## Pod 的安全上下文

安全上下文是在 pod 级别使用的，这意味着安全属性将应用于 pod 内的所有容器。

以下是 pod 级别的主要安全属性列表：

+   **fsGroup**：这是一个应用于所有容器的特殊辅助组。这个属性的有效性取决于卷类型。基本上，它允许`kubelet`将挂载卷的所有权设置为具有辅助 GID 的 pod。

+   **sysctls**：`sysctls`用于在运行时配置内核参数。在这样的上下文中，`sysctls`和内核参数是可以互换使用的。这些`sysctls`命令是命名空间内的内核参数，适用于 pod。以下`sysctls`命令已知是命名空间内的：`kernel.shm*`、`kernel.msg*`、`kernel.sem`和`kernel.mqueue.*`。不安全的`sysctls`默认情况下是禁用的，不应在生产环境中启用。

+   **runAsUser**：这是用来指定容器镜像的入口进程运行的 UID 的。默认设置是镜像元数据中指定的用户（例如，Dockerfile 中的`USER`指令）。这个属性也可以在`SecurityContext`中使用，它在容器级别生效。如果在`SecurityContext`和`PodSecurityContext`中都设置了这个属性，那么在容器级别指定的值会优先生效。

+   **runAsGroup**：类似于`runAsUser`，这是用来指定容器的入口进程运行的 GID 的。这个属性也可以在`SecurityContext`中使用，它在容器级别生效。如果在`SecurityContext`和`PodSecurityContext`中都设置了这个属性，那么在容器级别指定的值会优先生效。

+   **runAsNonRoot**：默认情况下设置为`false`，将其设置为`true`可以启用验证，即容器中的进程不能以 root 用户（UID=0）身份运行。验证由`kubelet`执行。将其设置为`true`，`kubelet`将阻止以 root 用户身份运行的容器启动。将其设置为`true`是一个很好的安全实践。此属性也可在`SecurityContext`中使用，其在容器级别生效。如果在`SecurityContext`和`PodSecurityContext`中都设置了此属性，则以容器级别指定的值优先。

+   **seLinuxOptions**：这是用来指定容器的 SELinux 上下文的。如果未指定，默认情况下，容器运行时会为容器分配一个随机的 SELinux 上下文。此属性也可在`SecurityContext`中使用，其在容器级别生效。如果在`SecurityContext`和`PodSecurityContext`中都设置了此属性，则以容器级别指定的值优先。

请注意，`runAsUser`、`runAsGroup`、`runAsNonRoot`和`seLinuxOptions`属性在容器级别的`SecurityContext`和 pod 级别的`PodSecurityContext`中都可用。这为用户提供了灵活性和极其重要的安全控制。`fsGroup`和`sysctls`不像其他属性那样常用，所以只有在必要时才使用它们。

## AppArmor 配置文件

AppArmor 配置文件通常定义了进程拥有的 Linux 功能，容器可以访问的网络资源和文件等。为了使用 AppArmor 配置文件保护 pod 或容器，您需要更新 pod 的注释。让我们看一个例子，假设您有一个 AppArmor 配置文件来阻止任何文件写入活动。

[PRE3]

请注意，AppArmor 不是 Kubernetes 对象，如 pod、部署等。它不能通过`kubectl`操作。您需要 SSH 到每个节点，并将 AppArmor 配置文件加载到内核中，以便 pod 可以使用它。

以下是加载 AppArmor 配置文件的命令：

[PRE4]

然后，将配置文件放入`enforce`模式：

[PRE5]

一旦加载了 AppArmor 配置文件，您可以更新 pod 的注释，以使用 AppArmor 配置文件保护您的容器。以下是将 AppArmor 配置文件应用于容器的示例：

[PRE6]

`hello-apparmor`内的容器除了在回显“Hello AppArmor！”消息后进入睡眠状态外，什么也不做。当它运行时，如果您从容器中启动一个 shell 并写入任何文件，AppArmor 将会阻止。尽管编写健壮的 AppArmor 配置文件并不容易，但您仍然可以创建一些基本的限制，比如拒绝写入到某些目录，拒绝接受原始数据包，并使某些文件只读。此外，在将配置应用到生产集群之前，先测试配置文件。开源工具如 bane 可以帮助为容器创建 AppArmor 配置文件。

我们不打算在本书中深入讨论 seccomp 配置文件，因为为微服务编写 seccomp 配置文件并不容易。即使是应用程序开发人员也不知道他们开发的微服务有哪些系统调用是合法的。尽管您可以打开审计模式以避免破坏微服务的功能，但构建健壮的 seccomp 配置文件仍然任重道远。另一个原因是，这个功能在版本 1.17 之前仍处于 alpha 阶段。根据 Kubernetes 的官方文档，alpha 表示默认情况下禁用，可能存在错误，并且只建议在短期测试集群中运行。当 seccomp 有任何新的更新时，我们可能会在以后更详细地介绍 seccomp。

我们已经介绍了如何在构建时保护 Kubernetes Pod。接下来，让我们看看如何在运行时保护 Kubernetes Pod。

# PodSecurityPolicy 的力量

Kubernetes PodSecurityPolicy 是一个集群级资源，通过它可以控制 pod 规范的安全敏感方面，从而限制 Kubernetes pod 的访问权限。作为一名 DevOps 工程师，您可能希望使用 PodSecurityPolicy 来限制大部分工作负载以受限访问权限运行，同时只允许少数工作负载以额外权限运行。

在本节中，我们将首先仔细研究 PodSecurityPolicy，然后介绍一个名为 `kube-psp-advisor` 的开源工具，它可以帮助为运行中的 Kubernetes 集群构建一个自适应的 PodSecurityPolicy。

## 理解 PodSecurityPolicy

您可以将 PodSecurityPolicy 视为评估 Pod 规范中定义的安全属性的策略。只有那些安全属性符合 PodSecurityPolicy 要求的 Pod 才会被允许进入集群。例如，PodSecurityPolicy 可以用于阻止启动大多数特权 Pod，同时只允许那些必要或受限制的 Pod 访问主机文件系统。

以下是由 PodSecurityPolicy 控制的主要安全属性：

+   **privileged**: 确定 Pod 是否可以以特权模式运行。

+   **hostPID**: 确定 Pod 是否可以使用主机 PID 命名空间。

+   **hostNetwork**: 确定 Pod 是否可以使用主机网络命名空间。

+   **hostIPC**: 确定 Pod 是否可以使用主机 IPC 命名空间。默认设置为`true`。

+   **allowedCapabilities**: 指定可以添加到容器中的功能列表。默认设置为空。

+   **defaultAddCapabilities**: 指定默认情况下将添加到容器中的功能列表。默认设置为空。

+   **requiredDropCapabilities**: 指定将从容器中删除的功能列表。请注意，功能不能同时在`allowedCapabilities`和`requiredDropCapabilities`字段中指定。默认设置为空。

+   **readOnlyRootFilesystem**: 当设置为`true`时，PodSecurityPolicy 将强制容器以只读根文件系统运行。如果容器的安全上下文中明确将属性设置为`false`，则将拒绝 Pod 运行。默认设置为`false`。

+   **runAsUser**: 指定可以在 Pod 和容器的安全上下文中设置的允许用户 ID 列表。默认设置允许所有。

+   **runAsGroup**: 指定可以在 Pod 和容器的安全上下文中设置的允许组 ID 列表。默认设置允许所有。

+   **allowPrivilegeEscalation**: 确定 Pod 是否可以提交请求以允许特权升级。默认设置为`true`。

+   **allowedHostPaths**: 指定 Pod 可以挂载的主机路径列表。默认设置允许所有。

+   **卷**: 指定可以由 Pod 挂载的卷类型列表。例如，`secret`、`configmap`和`hostpath`是有效的卷类型。默认设置允许所有。

+   **seLinux**: 指定可以在 Pod 和容器的安全上下文中设置的允许`seLinux`标签列表。默认设置允许所有。

+   **allowedUnsafeSysctl**：允许运行不安全的`sysctls`。默认设置不允许任何。

现在，让我们来看一个 PodSecurityPolicy 的例子：

[PRE7]

这个 PodSecurityPolicy 允许`NET_ADMIN`和`IPC_LOCK`的权限，从主机和 Kubernetes 的秘密卷挂载`/`，`/dev`和`/run`。它不强制执行任何文件系统组 ID 或辅助组，也允许容器以任何用户身份运行，访问主机网络命名空间，并以特权容器运行。策略中没有强制执行 SELinux 策略。

要启用此 Pod 安全策略，您可以运行以下命令：

[PRE8]

现在，让我们验证 Pod 安全策略是否已成功创建：

[PRE9]

输出将如下所示：

[PRE10]

创建了 Pod 安全策略后，还需要另一步来强制执行它。您将需要授予用户、组或服务帐户使用`PodSecurityPolicy`对象的特权。通过这样做，Pod 安全策略有权根据关联的服务帐户评估工作负载。以下是如何强制执行 PodSecurityPolicy 的示例。首先，您需要创建一个使用 PodSecurityPolicy 的集群角色：

[PRE11]

然后，创建一个`RoleBinding`或`ClusterRoleBinding`对象，将之前创建的`ClusterRole`对象与服务帐户、用户或组关联起来：

[PRE12]

之前创建的`use-example-pspbinding.yaml`文件创建了一个`RoleBinding`对象，将`use-example-psp`集群角色与`psp-test`命名空间中的`test-sa`服务帐户关联起来。通过所有这些设置，`psp-test`命名空间中其服务帐户为`test-sa`的任何工作负载将通过 PodSecurityPolicy 示例的评估。只有符合要求的工作负载才能被允许进入集群。

从上面的例子中，想象一下在您的 Kubernetes 集群中运行不同类型的工作负载，每个工作负载可能需要不同的特权来访问不同类型的资源。为不同的工作负载创建和管理 Pod 安全策略将是一个挑战。现在，让我们来看看`kube-psp-advisor`，看看它如何帮助您创建 Pod 安全策略。

## Kubernetes PodSecurityPolicy Advisor

Kubernetes PodSecurityPolicy Advisor（也称为`kube-psp-advisor`）是来自 Sysdig 的开源工具。它扫描集群中运行的工作负载的安全属性，然后基于此推荐您的集群或工作负载的 Pod 安全策略。

首先，让我们将`kube-psp-advisor`作为`kubectl`插件进行安装。如果您还没有安装`krew`，请按照说明（https://github.com/kubernetes-sigs/krew#installation）安装它。然后，使用`krew`安装`kube-psp-advisor`如下：

[PRE13]

然后，您应该能够运行以下命令来验证安装：

[PRE14]

要为命名空间中的工作负载生成 Pod 安全策略，可以运行以下命令：

[PRE15]

上述命令为在`psp-test`命名空间内运行的工作负载生成了 Pod 安全策略。如果工作负载使用默认服务账户，则不会为其生成 PodSecurityPolicy。这是因为默认服务账户将被分配给没有专用服务账户关联的工作负载。当然，您肯定不希望默认服务账户能够使用特权工作负载的 PodSecurityPolicy。

以下是`kube-psp-advisor`为`psp-test`命名空间中的工作负载生成的输出示例，包括 Role、RoleBinding 和 PodSecurityPolicy 在一个单独的 YAML 文件中，其中包含多个 Pod 安全策略。让我们来看一个推荐的 PodSecurityPolicy：

[PRE16]

以下是由`kube-psp-advisor`生成的 Role：

[PRE17]

以下是由`kube-psp-advisor`生成的 RoleBinding：

[PRE18]

前面的部分是推荐的 PodSecurityPolicy，`psp-for-psp-test-sa-1`，适用于`busy-rs`和`busy-pod`工作负载，因为这两个工作负载共享相同的服务账户`sa-1`。因此，分别创建了`Role`和`RoleBinding`来使用 Pod 安全策略`psp-for-psp-test-sa-1`。PodSecurityPolicy 是基于使用`sa-1`服务账户的工作负载的安全属性的聚合生成的：

[PRE19]

前面的部分提到`busy-rc`工作负载使用`default`服务账户，因此不会为其创建 Pod 安全策略。这是一个提醒，如果要为工作负载生成 Pod 安全策略，请不要使用默认服务账户。

构建 Kubernetes PodSecurityPolicy 并不是一件简单的事情，尽管如果一个受限的 PodSecurityPolicy 适用于整个集群，并且所有工作负载都符合它将是理想的。DevOps 工程师需要有创造力，以便构建受限的 Pod 安全策略，同时不破坏工作负载的功能。`kube-psp-advisor`使得实施 Kubernetes Pod 安全策略变得简单，适应您的应用程序要求，并且特别为每个应用程序提供了细粒度的权限，只允许最少访问权限的特权。

# 总结

在本章中，我们介绍了如何使用 CIS Docker 基准来加固容器镜像，然后详细介绍了 Kubernetes 工作负载的安全属性。接下来，我们详细介绍了 PodSecurityPolicy，并介绍了`kube-psp-advisor`开源工具，该工具有助于建立 Pod 安全策略。

保护 Kubernetes 工作负载不是一蹴而就的事情。安全控制需要从构建、部署和运行阶段应用。它始于加固容器镜像，然后以安全的方式配置 Kubernetes 工作负载的安全属性。这发生在构建阶段。为不同的 Kubernetes 工作负载构建自适应的 Pod 安全策略也很重要。目标是限制大多数工作负载以受限权限运行，同时只允许少数工作负载以额外权限运行，而不会破坏工作负载的可用性。这发生在运行时阶段。`kube-psp-advisor`能够帮助构建自适应的 Pod 安全策略。

在下一章中，我们将讨论图像扫描。在 DevOps 工作流程中，这对于帮助保护 Kubernetes 工作负载至关重要。

# 问题

1.  在 Dockerfile 中，`HEALTHCHECK`是做什么的？

1.  为什么在 Dockerfile 中使用`COPY`而不是`ADD`？

1.  如果您的应用程序不监听任何端口，可以丢弃哪些默认功能？

1.  `runAsNonRoot`属性控制什么？

1.  当您创建一个`PodSecurityPolicy`对象时，为了强制执行该 Pod 安全策略，您还需要做什么？

# 进一步阅读

您可以参考以下链接，了解本章涵盖的主题的更多信息：

+   要了解有关`kube-psp-advisor`的更多信息，请访问以下链接：[`github.com/sysdiglabs/kube-psp-advisor`](https://github.com/sysdiglabs/kube-psp-advisor)

+   要了解有关 AppArmor 的更多信息，请访问以下链接：[`gitlab.com/apparmor/apparmor/-/wikis/Documentation`](https://gitlab.com/apparmor/apparmor/-/wikis/Documentation)

+   要了解有关 bane 的更多信息，请访问以下链接：[`github.com/genuinetools/bane`](https://github.com/genuinetools/bane)

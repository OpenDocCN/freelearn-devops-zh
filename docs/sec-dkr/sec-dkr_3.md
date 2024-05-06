# 第三章：保护和加固 Linux 内核

在本章中，我们将把注意力转向保护和加固每个在您的 Docker 主机上运行的容器所依赖的关键组件：Linux 内核。我们将专注于两个主题：您可以遵循的加固 Linux 内核的指南和您可以添加到您的工具库中以帮助加固 Linux 内核的工具。在深入讨论之前，让我们简要地看一下本章将涵盖的内容：

+   Linux 内核加固指南

+   Linux 内核加固工具

+   **Grsecurity**

+   **Lynis**

# Linux 内核加固指南

在本节中，我们将看一下 SANS 研究所关于 Linux 内核的加固指南。虽然很多信息已经过时，但我认为了解 Linux 内核如何发展并成为一个安全实体是很重要的。如果你能够进入时光机，回到 2003 年并尝试做今天想做的事情，这就是你必须做的一切。

首先，关于 SANS 研究所的一些背景信息。这是一家总部位于美国的私人公司，专门从事网络安全和信息技术相关的培训和教育。这些培训使专业人员能够防御其环境免受攻击者的侵害。SANS 还通过其 SANS 技术学院领导实验室提供各种免费的安全相关内容。有关更多信息，请访问[`www.sans.edu/research/leadership-laboratory`](http://www.sans.edu/research/leadership-laboratory)。

为了帮助减轻这种广泛的攻击基础，需要在 IT 基础设施和软件的每个方面都有安全关注。基于此，开始的第一步应该是在 Linux 内核上。

## SANS 加固指南深入研究

由于我们已经介绍了 SANS 研究所的背景，让我们继续并开始遵循我们将用来保护我们的 Linux 内核的指南。

作为参考，我们将使用以下 URL，并重点介绍您应该关注和在您的环境中实施以保护 Linux 内核的关键点：

[`www.sans.org/reading-room/whitepapers/linux/linux-kernel-hardening-1294`](https://www.sans.org/reading-room/whitepapers/linux/linux-kernel-hardening-1294)

Linux 内核是 Linux 生态系统中不断发展和成熟的一部分，因此，重要的是要对当前的 Linux 内核有一个牢固的掌握，这将有助于在未来的发布中锁定新的功能集。

Linux 内核允许加载模块，而无需重新编译或重新启动，这在您希望消除停机时间时非常有用。一些不同的操作系统在尝试将更新应用到特定的操作系统/应用程序标准时需要重新启动。这也可能是 Linux 内核的一个坏处，因为攻击者可以向内核注入有害材料，并且不需要重新启动机器，这可能会被注意到系统的重新启动。因此，建议禁用带有加载选项的静态编译内核，以帮助防止攻击向量。

缓冲区溢出是攻击者入侵内核并获取权限的另一种方式。应用程序在内存中存储用户数据的限制或缓冲区。攻击者使用特制的代码溢出这个缓冲区，这可能让攻击者控制系统，从而赋予他们在那一点上做任何他们想做的事情的权力。他们可以向系统添加后门，将日志发送到一个邪恶的地方，向系统添加额外的用户，甚至将您锁在系统外。为了防止这些类型的攻击，指南专注于三个重点领域。

第一个是**Openwall** Linux 内核补丁，这是一个用来解决这个问题的补丁。这个补丁还包括一些其他安全增强功能，可能归因于您的运行环境。其中一些项目包括在`/tmp`文件夹位置限制链接和文件读取/写入，以及对文件系统上`/proc`位置的访问限制。它还包括对一些用户进程的增强执行力，您也可以控制，以及能够销毁未使用的共享内存段，最后，对于那些运行内核版本旧于 2.4 版本的用户，还有一些其他增强功能。

如果您正在运行较旧版本的 Linux 内核，您将希望查看[`www.openwall.com/Owl/`](http://www.openwall.com/Owl/)上的 Openwall 强化 Linux 和[`www.openwall.com/linux/`](http://www.openwall.com/linux/)上的 Openwall Linux。

下一个软件叫做**Exec** **Shield**，它采用了类似于 Openwall Linux 内核补丁的方法，实现了一个不可执行的堆栈，但是 Exec Shield 通过尝试保护任何和所有的虚拟内存段来扩展了这一点。这个补丁仅限于防止针对 Linux 内核地址空间的攻击。这些地址空间包括堆栈、缓冲区或函数指针溢出空间。

关于这个补丁的更多信息可以在[`en.wikipedia.org/wiki/Exec_Shield`](https://en.wikipedia.org/wiki/Exec_Shield)找到。

最后一个是**PaX**，它是一个为 Linux 内核创建补丁以防止各种软件漏洞的团队。由于这是我们将在下一节深入讨论的内容，我们将只讨论一些它的特点。这个补丁关注以下三个地址空间：

+   **PAGEEXEC**：这些是基于分页的、不可执行的页面

+   **SEGMEXEC**：这些是基于分段的、不可执行的页面

+   **MPROTECT**：这些是`mmap()`和`mprotect()`的限制

要了解有关 PaX 的更多信息，请访问[`pax.grsecurity.net`](https://pax.grsecurity.net)。

现在你已经看到了你需要付出多少努力，你应该高兴安全现在对每个人都是至关重要的，特别是 Linux 内核。在后面的一些章节中，我们将看一些正在用于帮助保护环境的新技术：

+   命名空间

+   cgroups

+   sVirt

+   Summon

通过`docker run`命令上的`--cap-ad`和`--cap-drop`开关也可以实现许多功能。

即使像以前一样，你仍然需要意识到内核在主机上的所有容器中是共享的，因此，你需要保护这个内核，并在必要时注意漏洞。以下链接允许您查看 Linux 内核中的**常见** **漏洞和** **曝光**（**CVE**）：

[`www.cvedetails.com/vulnerability-list/vendor_id-33/product_id-47/cvssscoremin-7/cvssscoremax-7.99/Linux-Linux-Kernel.html`](https://www.cvedetails.com/vulnerability-list/vendor_id-33/product_id-47/cvssscoremin-7/cvssscoremax-7.99/Linux-Linux-Kernel.html)

## 访问控制

您可以在 Linux 上叠加各种级别的访问控制，以及应该遵循的特定用户的建议，这些用户将是您系统上的超级用户。只是为了给超级用户一些定义，他们是系统上具有无限制访问权限的帐户。在叠加这些访问控制时，应包括 root 用户。

这些访问控制建议将是以下内容：

+   限制 root 用户的使用

+   限制其 SSH 的能力

默认情况下，在某些系统上，如果启用了 SSH，root 用户有能力 SSH 到机器上，我们可以从一些 Linux 系统的`/etc/ssh/sshd_config`文件的部分中看到如下内容：

```
# Authentication:

#LoginGraceTime 2m
#PermitRootLogin no
#StrictModes yes
#MaxAuthTries 6
#MaxSessions 10
```

从这里你可以看到，`PermitRootLogin no`部分被用`#`符号注释掉，这意味着这行不会被解释。要更改这个，只需删除`#`符号，保存文件并重新启动服务。该文件的部分现在应该类似于以下代码：

```
# Authentication:

#LoginGraceTime 2m
PermitRootLogin no
#StrictModes yes
#MaxAuthTries 6
#MaxSessions 10
```

现在，您可能希望重新启动 SSH 服务以使这些更改生效，如下所示：

```
**$ sudo service sshd restart**

```

+   限制其在控制台之外的登录能力。在大多数 Linux 系统上，有一个文件在`/etc/default/login`中，在该文件中，有一行类似以下的内容：

```
#CONSOLE=/dev/console
```

类似于前面的例子，我们需要通过删除`#`来取消注释此行，以使其生效。这将只允许 root 用户在`console`上登录，而不是通过 SSH 或其他方法。

+   限制`su`命令

`su`命令允许您以 root 用户身份登录并能够发出 root 级别的命令，从而完全访问整个系统。为了限制谁可以使用此命令，有一个文件位于`/etc/pam.d/su`，在这个文件中，您会看到类似以下的一行：

```
auth required /lib/security/pam_wheel.so use_uid
```

您还可以选择这里的以下代码行，具体取决于您的 Linux 版本：

```
auth required pam_wheel.so use_uid
```

检查 wheel 成员资格将根据当前用户 ID 来执行对`su`命令的使用能力。

+   要求使用`sudo`来运行命令

+   其他一些推荐使用的访问控制包括以下控制：

+   强制访问控制（MAC）：限制用户在系统上的操作

+   基于角色的访问控制：使用组分配这些组可以执行的角色

+   基于规则集的访问控制（RSBAC）：按请求类型分组的规则集，并根据设置的规则执行操作

+   **领域和类型强制执行**（DTE）：允许或限制某些领域执行特定操作，或防止领域相互交互

您还可以利用以下内容：

+   SELinux（基于 RPM 的系统（如 Red Hat、CentOS 和 Fedora）

+   AppArmor（基于 apt-get 的系统（如 Ubuntu 和 Debian）

RSBAC，正如我们之前讨论的那样，允许您选择适合系统运行的控制方法。您还可以创建自己的访问控制模块，以帮助执行。在大多数 Linux 系统上，默认情况下，这些类型的环境是启用或强制执行模式的。大多数人在创建新系统时会关闭这些功能，但这会带来安全缺陷，因此，重要的是要学习这些系统的工作原理，并在启用或强制执行模式下使用它们以帮助减轻进一步的风险。

有关每个的更多信息可以在以下找到：

+   SELinux：[`en.wikipedia.org/wiki/Security-Enhanced_Linux`](https://en.wikipedia.org/wiki/Security-Enhanced_Linux)

+   **AppArmor**：[`en.wikipedia.org/wiki/AppArmor`](https://en.wikipedia.org/wiki/AppArmor)

## 面向发行版

在 Linux 社区中有许多 Linux 发行版，或者他们称之为“风味”，已经被*预先烘烤*以进行硬化。我们之前提到过一个，即 Linux 的**Owlwall**风味，但还有其他一些。在其他两个中，一个已经不复存在，即**Adamantix**，另一个是**Gentoo Linux**。这些 Linux 风味作为其操作系统构建的标准，具有一些内置的 Linux 内核加固功能。

# Linux 内核加固工具

有一些 Linux 内核加固工具，但在本节中我们将只关注其中两个。第一个是 Grsecurity，第二个是 Lynis。这些工具可以增加您的武器库，帮助增强您将在其中运行 Docker 容器的环境的安全性。

## Grsecurity

那么，Grsecurity 到底是什么？根据他们的网站，Grsecurity 是 Linux 内核的广泛安全增强。这个增强包含了一系列项目，有助于抵御各种威胁。这些威胁可能包括以下组件：

+   零日漏洞利用：这可以减轻并保护您的环境，直到供应商提供长期解决方案。

+   **共享主机或容器的弱点**：这可以保护您免受各种技术以及容器使用的针对主机上每个容器的内核妥协。

+   **它超越了基本的访问控制**：Grsecurity 与 PaX 团队合作，引入复杂性和不可预测性，以防止攻击者，并拒绝攻击者再有机会。

+   **与现有的 Linux 发行版集成**：由于 Grsecurity 是基于内核的，因此可以与 Red Hat、Ubuntu、Debian 和 Gentoo 等任何 Linux 发行版一起使用。无论您使用的是哪种 Linux 发行版，都没有关系，因为重点是底层的 Linux 内核。

更多信息请访问[`grsecurity.net/`](https://grsecurity.net/)。

要直接了解有关使用类似 Grsecurity 的工具提供的功能集，请访问以下链接：

[`grsecurity.net/features.php`](http://grsecurity.net/features.php)

在此页面，项目将被分为以下五个类别：

+   内存损坏防御

+   文件系统加固

+   其他保护

+   RBAC

+   GCC 插件

## Lynis

Lynis 是一个用于审核系统安全性的开源工具。它直接在主机上运行，因此可以访问 Linux 内核本身以及其他各种项目。Lynis 可以在几乎所有 Unix 操作系统上运行，包括以下操作系统：

+   AIS

+   FreeBSD

+   Mac OS

+   Linux

+   Solaris

Lynis 是一个 shell 脚本编写的工具，因此，它与在系统上复制和粘贴以及运行简单命令一样容易：

```
**./lynis audit system**

```

在运行时，将执行以下操作：

+   确定操作系统

+   执行搜索以获取可用工具和实用程序

+   检查是否有 Lynis 更新

+   从启用的插件运行测试

+   按类别运行安全性测试

+   报告安全扫描状态

更多信息请访问[`rootkit.nl/projects/lynis.html`](https://rootkit.nl/projects/lynis.html)和[`cisofy.com/lynis/`](https://cisofy.com/lynis/)。

# 总结

在本章中，我们研究了加固和保护 Linux 内核。我们首先查看了一些加固指南，然后深入了解了 SANS 研究所加固指南的概述。我们还研究了如何通过各种补丁来防止内核和应用程序中的缓冲区溢出。我们还研究了各种访问控制、SELinux 和 AppArmor。最后，我们还研究了两种加固工具，可以将它们添加到我们的软件工具箱中，即 Grsecurity 和 Lynis。

在下一章中，我们将看一下 Docker Bench 安全应用程序。这是一个可以查看各种 Docker 项目的应用程序，比如主机配置、Docker 守护程序配置、守护程序配置文件、容器镜像和构建文件、容器运行时，最后是 Docker 安全操作。它将包含大量代码输出的实际示例。

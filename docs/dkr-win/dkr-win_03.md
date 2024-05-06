# 第二章：打包和运行应用程序作为 Docker 容器

Docker 将基础设施的逻辑视图简化为三个核心组件：主机、容器和图像。主机是运行容器的服务器，每个容器都是应用程序的隔离实例。容器是从图像创建的，图像是打包的应用程序。Docker 容器图像在概念上非常简单：它是一个包含完整、自包含应用程序的单个单元。图像格式非常高效，图像和容器运行时之间的集成非常智能，因此掌握图像是有效使用 Docker 的第一步。

在第一章中，您已经通过运行一些基本容器来检查 Docker 安装是否正常工作，但我没有仔细检查图像或 Docker 如何使用它。在本章中，您将彻底了解 Docker 图像，了解它们的结构，了解 Docker 如何使用它们，并了解如何将自己的应用程序打包为 Docker 图像。

首先要理解的是图像和容器之间的区别，通过从相同的图像运行不同类型的容器，您可以非常清楚地看到这一点。

在本章中，您将更多地了解 Docker 的基础知识，包括：

+   从图像运行容器

+   从 Dockerfile 构建图像

+   将自己的应用程序打包为 Docker 图像

+   在图像和容器中处理数据

+   将传统的 ASP.NET Web 应用程序打包为 Docker 图像

# 技术要求

要跟随示例，您需要在 Windows 10 上运行 Docker，并更新到 18.09 版，或者在 Windows Server 2019 上运行。本章的代码可在[`github.com/sixeyed/docker-on-windows/tree/second-edition/ch02`](https://github.com/sixeyed/docker-on-windows/tree/second-edition/ch02)上找到。

# 从图像运行容器

`docker container run`命令从图像创建一个容器，并在容器内启动应用程序。实际上，这相当于运行两个单独的命令，`docker container create`和`docker container start`，这表明容器可以具有不同的状态。您可以创建一个容器而不启动它，并且可以暂停、停止和重新启动运行中的容器。容器可以处于不同的状态，并且可以以不同的方式使用它们。

# 使用任务容器执行一个任务

`dockeronwindows/ch02-powershell-env:2e`镜像是一个打包的应用程序的示例，旨在在容器中运行并执行单个任务。该镜像基于 Microsoft Windows Server Core，并设置为在启动时运行一个简单的 PowerShell 脚本，打印有关当前环境的详细信息。让我们看看当我直接从镜像运行容器时会发生什么：

[PRE0]

没有任何选项，容器将运行内置于镜像中的 PowerShell 脚本，并且脚本将打印有关操作系统环境的一些基本信息。我将其称为**任务容器**，因为容器执行一个任务然后退出。

如果运行`docker container ls`，列出所有活动容器，您将看不到此容器。但如果运行`docker container ls --all`，显示所有状态的容器，您将在`Exited`状态中看到它：

[PRE1]

任务容器在自动化重复任务方面非常有用，比如运行脚本来设置环境、备份数据或收集日志文件。您的容器镜像打包了要运行的脚本，以及脚本所需的所有要求的确切版本，因此安装了 Docker 的任何人都可以运行脚本，而无需安装先决条件。

这对于 PowerShell 特别有用，因为脚本可能依赖于几个 PowerShell 模块。这些模块可能是公开可用的，但您的脚本可能依赖于特定版本。您可以构建一个已安装了模块的镜像，而不是共享一个需要用户安装许多不同模块的正确版本的脚本。然后，您只需要 Docker 来运行脚本任务。

镜像是自包含的单位，但您也可以将其用作模板。一个镜像可能配置为执行一项任务，但您可以以不同的方式从镜像运行容器以执行不同的任务。

# 连接到交互式容器

**交互式容器**是指与 Docker 命令行保持开放连接的容器，因此您可以像连接到远程机器一样使用容器。您可以通过指定交互式选项和容器启动时要运行的命令来从相同的 Windows Server Core 镜像运行交互式容器：

[PRE2]

`--interactive`选项运行交互式容器，`--tty`标志将终端连接附加到容器。在容器映像名称后的`powershell`语句是容器启动时要运行的命令。通过指定命令，您可以替换映像中设置的启动命令。在这种情况下，我启动了一个 PowerShell 会话，它代替了配置的命令，因此环境打印脚本不会运行。

交互式容器会持续运行，只要其中的命令在运行。当您连接到 PowerShell 时，在主机的另一个窗口中运行`docker container ls`，会显示容器仍在运行。当您在容器中键入`exit`时，PowerShell 会话结束，因此没有进程在运行，容器也会退出。

交互式容器在构建自己的容器映像时非常有用，它们可以让您首先以交互方式进行步骤，并验证一切是否按您的预期工作。它们也是很好的探索工具。您可以从 Docker 注册表中拉取别人的映像，并在运行应用程序之前探索其内容。

阅读本书时，您会发现 Docker 可以在虚拟网络中托管复杂的分布式系统，每个组件都在自己的容器中运行。如果您想检查系统的某些部分，可以在网络内部运行交互式容器，并检查各个组件，而无需使部分公开可访问。

# 在后台容器中保持进程运行

最后一种类型的容器是您在生产中最常使用的，即后台容器，它在后台保持长时间运行的进程。它是一个行为类似于 Windows 服务的容器。在 Docker 术语中，它被称为**分离容器**，Docker 引擎会在后台保持其运行。在容器内部，进程在前台运行。该进程可能是一个 Web 服务器或一个轮询消息队列以获取工作的控制台应用程序，但只要进程保持运行，Docker 就会保持容器保持活动状态。

我可以再次从相同的映像运行后台容器，指定`detach`选项和运行一些分钟的命令：

[PRE3]

在这种情况下，当容器启动后，控制返回到终端；长随机字符串是新容器的 ID。您可以运行`docker container ls`并查看正在运行的容器，`docker container logs`命令会显示容器的控制台输出。对于操作特定容器的命令，您可以通过容器名称或容器 ID 的一部分来引用它们 - ID 是随机的，在我的情况下，这个容器 ID 以`bb3`开头：

[PRE4]

`--detach`标志将容器分离，使其进入后台，而在这种情况下，命令只是重复一百次对`localhost`的 ping。几分钟后，PowerShell 命令完成，因此没有正在运行的进程，容器退出。

这是一个需要记住的关键事情：如果您想要在后台保持容器运行，那么 Docker 在运行容器时启动的进程必须保持运行。

现在您已经看到容器是从镜像创建的，但它可以以不同的方式运行。因此，您可以完全按照准备好的镜像使用，或者将镜像视为内置默认启动模式的模板。接下来，我将向您展示如何构建该镜像。

# 构建 Docker 镜像

Docker 镜像是分层的。底层是操作系统，可以是完整的操作系统，如 Windows Server Core，也可以是微软 Nano Server 等最小的操作系统。在此之上是每次构建镜像时对基本操作系统所做更改的层，通过安装软件、复制文件和运行命令。从逻辑上讲，Docker 将镜像视为单个单位，但从物理上讲，每个层都存储为 Docker 缓存中的单独文件，因此具有许多共同特征的镜像可以共享缓存中的层。

镜像是使用 Dockerfile 语言的文本文件构建的 - 指定要从哪个基本操作系统镜像开始以及添加的所有步骤。这种语言非常简单，您只需要掌握几个命令就可以构建生产级别的镜像。我将从查看到目前为止在本章中一直在使用的基本 PowerShell 镜像开始。

# 理解 Dockerfile

Dockerfile 只是一个将软件打包到 Docker 镜像中的部署脚本。PowerShell 镜像的完整代码只有三行：

[PRE5]

即使你以前从未见过 Dockerfile，也很容易猜到发生了什么。按照惯例，指令（`FROM`、`COPY`和`CMD`）是大写的，参数是小写的，但这不是强制的。同样按照惯例，你保存文本在一个名为`Dockerfile`的文件中，但这也不是强制的（在 Windows 中，没有扩展名的文件看起来很奇怪，但请记住 Docker 的传统是在 Linux 中）。

让我们逐行查看 Dockerfile 中的指令：

+   `FROM mcr.microsoft.com/windows/servercore:ltsc2019`使用名为`windows/servercore`的镜像作为此镜像的起点，指定了镜像的`ltsc2019`版本和其托管的注册表。

+   `COPY scripts/print-env-details.ps1 C:\\print-env.ps1`将 PowerShell 脚本从本地计算机复制到镜像中的特定位置。

+   `CMD ["powershell.exe", "C:\\print-env.ps1"]`指定了容器运行时的启动命令，在这种情况下是运行 PowerShell 脚本。

这里有一些明显的问题。基础镜像是从哪里来的？Docker 内置了镜像注册表的概念，这是一个容器镜像的存储库。默认注册表是一个名为**Docker Hub**的免费公共服务。微软在 Docker Hub 上发布了一些镜像，但 Windows 基础镜像托管在**Microsoft Container Registry**（**MCR**）上。

Windows Server Core 镜像的 2019 版本被称为`windows/servercore:ltsc2019`。第一次使用该镜像时，Docker 会从 MCR 下载到本地计算机，然后缓存以供进一步使用。

Docker Hub 是 Microsoft 所有镜像的发现列表，因为 MCR 没有 Web UI。即使镜像托管在 MCR 上，它们也会在 Docker Hub 上列出，所以当你在寻找镜像时，那就是去的地方。

PowerShell 脚本是从哪里复制过来的？构建镜像时，包含 Dockerfile 的目录被用作构建的上下文。从这个 Dockerfile 构建镜像时，Docker 会期望在上下文目录中找到一个名为`scripts`的文件夹，其中包含一个名为`print-env-details.ps1`的文件。如果找不到该文件，构建将失败。

Dockerfile 使用反斜杠作为转义字符，以便将指令继续到新的一行。这与 Windows 文件路径冲突，所以你必须将`C:\print.ps1`写成`C:\\print.ps1`或`C:/print.ps1`。有一个很好的方法来解决这个问题，在 Dockerfile 开头使用处理器指令，我将在本章后面进行演示。

你如何知道 PowerShell 可以使用？它是 Windows Server Core 基础镜像的一部分，所以你可以依赖它。你可以使用额外的 Dockerfile 指令安装任何不在基础镜像中的软件。你可以添加 Windows 功能，设置注册表值，将文件复制或下载到镜像中，解压 ZIP 文件，部署 MSI 文件，以及其他任何你需要的操作。

这是一个非常简单的 Dockerfile，但即使如此，其中两条指令是可选的。只有`FROM`指令是必需的，所以如果你想构建一个微软的 Windows Server Core 镜像的精确克隆，你可以在 Dockerfile 中只使用一个`FROM`语句，并且随意命名克隆的镜像。

# 从 Dockerfile 构建镜像

现在你有了一个 Dockerfile，你可以使用`docker`命令行将其构建成一个镜像。像大多数 Docker 命令一样，`image build`命令很简单，只有很少的必需选项，更倾向于使用约定而不是命令。

要构建一个镜像，打开命令行并导航到 Dockerfile 所在的目录。然后运行`docker image build`并给你的镜像打上一个标签，这个标签就是将来用来识别镜像的名称。

[PRE6]

每个镜像都需要一个标签，使用`--tag`选项指定，这是本地镜像缓存和镜像注册表中镜像的唯一标识符。标签是你在运行容器时将引用镜像的方式。完整的标签指定要使用的注册表：存储库名称，这是应用程序的标识符，以及后缀，这是镜像的版本标识符。

当你为自己构建一个镜像时，你可以随意命名，但约定是将你的存储库命名为你的注册表用户名，后面跟上应用程序名称：`{user}/{app}`。你还可以使用标签来标识应用程序的版本或变体，比如`sixeyed/git`和`sixeyed/git:2.17.1-windowsservercore-ltsc2019`，这是 Docker Hub 上我的两个镜像。

`image build`命令末尾的句点告诉 Docker 要使用的上下文的位置。`.`是当前目录。Docker 将目录树的内容复制到一个临时文件夹进行构建，因此上下文需要包含 Dockerfile 中引用的任何文件。复制上下文后，Docker 开始执行 Dockerfile 中的指令。

# 检查 Docker 构建镜像的过程

理解 Docker 镜像是如何构建的将有助于您构建高效的镜像。`image build`命令会产生大量输出，告诉您 Docker 在构建的每个步骤中做了什么。Dockerfile 中的每个指令都会作为一个单独的步骤执行，产生一个新的镜像层，最终镜像将是所有层的组合堆栈。以下代码片段是构建我的镜像的输出：

[PRE7]

这就是 Docker 构建镜像时发生的事情：

1.  `FROM`镜像已经存在于我的本地缓存中，因此 Docker 不需要下载它。输出是 Microsoft 的 Windows Server Core 镜像的 ID（以`8b79`开头）。

1.  Docker 将脚本文件从构建上下文复制到一个新的镜像层（ID `5e9e`）。

1.  Docker 配置了当从镜像运行容器时要执行的命令。它从*步骤 2*镜像创建一个临时容器，配置启动命令，将容器保存为一个新的镜像层（ID `5f27`），并删除中间容器（ID `c14c`）。

最终层被标记为镜像名称，但所有中间层也被添加到本地缓存中。这种分层的方法意味着 Docker 在构建镜像和运行容器时可以非常高效。最新的 Windows Server Core 镜像未经压缩超过 4GB，但当您运行基于 Windows Server Core 的多个容器时，它们将都使用相同的基础镜像层，因此您不会得到多个 4GB 镜像的副本。

您将在本章后面更多地了解镜像层和存储，但首先我将看一些更复杂的 Dockerfile，其中打包了.NET 和.NET Core 应用程序。

# 打包您自己的应用程序

构建镜像的目标是将您的应用程序打包成一个便携、自包含的单元。镜像应尽可能小，这样在运行应用程序时移动起来更容易，并且应尽可能少地包含操作系统功能，这样启动时间快，攻击面小。

Docker 不对图像大小施加限制。你的长期目标可能是构建在 Linux 或 Nano Server 上运行轻量级.NET Core 应用程序的最小图像。但你可以先将现有的 ASP.NET 应用程序作为 Docker 图像的全部内容打包，以在 Windows Server Core 上运行。Docker 也不对如何打包应用程序施加限制，因此你可以选择不同的方法。

# 在构建过程中编译应用程序

在 Docker 图像中打包自己的应用程序有两种常见的方法。第一种是使用包含应用程序平台和构建工具的基础图像。因此，在你的 Dockerfile 中，你将源代码复制到图像中，并在图像构建过程中编译应用程序。

这是一个受欢迎的公共图像的方法，因为这意味着任何人都可以构建图像，而无需在本地安装应用程序平台。这也意味着应用程序的工具与图像捆绑在一起，因此可以使在容器中运行的应用程序的调试和故障排除成为可能。

这是一个简单的.NET Core 应用程序的示例。这个 Dockerfile 是为`dockeronwindows/ch02-dotnet-helloworld:2e`图像而设计的：

[PRE8]

Dockerfile 使用了来自 Docker Hub 的 Microsoft 的.NET Core 图像作为基础图像。这是图像的一个特定变体，它基于 Nano Server 1809 版本，并安装了.NET Core 2.2 SDK。构建将应用程序源代码从上下文中复制进来，并在容器构建过程中编译应用程序。

这个 Dockerfile 中有三个你以前没有见过的新指令：

1.  `WORKDIR`指定当前工作目录。如果目录在中间容器中不存在，Docker 会创建该目录，并将其设置为当前目录。它将保持为 Dockerfile 中的后续指令以及从图像运行的容器的工作目录。

1.  `USER`更改构建中的当前用户。Nano Server 默认使用最低特权用户。这将切换到容器图像中的内置帐户，该帐户具有管理权限。

1.  `RUN`在中间容器中执行命令，并在命令完成后保存容器的状态，创建一个新的图像层。

当我构建这个图像时，你会看到`dotnet`命令的输出，这是应用程序从图像构建中的`RUN`指令中编译出来的：

[PRE9]

你会在 Docker Hub 上经常看到这种方法，用于使用.NET Core、Go 和 Node.js 等语言构建的应用程序，其中工具很容易添加到基础镜像中。这意味着你可以在 Docker Hub 上设置自动构建，这样当你将代码更改推送到 GitHub 时，Docker 的服务器就会根据 Dockerfile 构建你的镜像。服务器可以在没有安装.NET Core、Go 或 Node.js 的情况下执行此操作，因为所有构建依赖项都在基础镜像中。

这种选项意味着最终镜像将比生产应用程序所需的要大得多。语言 SDK 和工具可能会占用比应用程序本身更多的磁盘空间，但你的最终结果应该是应用程序；当容器在生产环境运行时，镜像中占用空间的所有构建工具都不会被使用。另一种选择是首先构建应用程序，然后将编译的二进制文件打包到你的容器镜像中。

# 在构建之前编译应用程序

首先构建应用程序与现有的构建流水线完美契合。你的构建服务器需要安装所有的应用程序平台和构建工具来编译应用程序，但你的最终容器镜像只包含运行应用程序所需的最小内容。采用这种方法，我的.NET Core 应用程序的 Dockerfile 变得更加简单：

[PRE10]

这个 Dockerfile 使用了一个不同的`FROM`镜像，其中只包含.NET Core 2.2 运行时，而不包含工具（因此它可以运行已编译的应用程序，但无法从源代码编译）。你不能在构建应用程序之前构建这个镜像，所以你需要在构建脚本中包装`docker image build`命令，该脚本还运行`dotnet publish`命令来编译二进制文件。

一个简单的构建脚本，用于编译应用程序并构建 Docker 镜像，看起来像这样：

[PRE11]

如果你把 Dockerfile 指令放在一个名为**Dockerfile**之外的文件中，你需要使用`--file`选项指定文件名：`docker image build --file Dockerfile.slim`。

我把平台工具的要求从镜像移到了构建服务器上，这导致最终镜像变得更小：与之前版本相比，这个版本的大小为 410 MB，而之前的版本为 1.75 GB。你可以通过列出镜像并按照镜像仓库名称进行过滤来看到大小的差异：

[PRE12]

这个新版本也是一个更受限制的镜像。源代码和.NET Core SDK 没有打包在镜像中，所以你不能连接到正在运行的容器并检查应用程序代码，或者对代码进行更改并重新编译应用程序。

对于企业环境或商业应用程序，你可能已经有一个设备齐全的构建服务器，并且打包构建的应用程序可以成为更全面工作流的一部分：

![](img/d09a1ae3-e97a-4318-8118-f2d2a39c5828.png)

在这个流水线中，开发人员将他们的更改推送到中央源代码仓库（**1**）。构建服务器编译应用程序并运行单元测试；如果测试通过，那么容器镜像将在暂存环境中构建和部署（2）。集成测试和端到端测试在暂存环境中运行，如果测试通过，那么你的容器镜像版本是一个好的发布候选，供测试人员验证（3）。

通过在生产环境中从镜像运行容器来部署新版本，并且你知道你的整个应用程序堆栈是通过了所有测试的相同的一组二进制文件。

这种方法的缺点是你需要在所有构建代理上安装应用程序 SDK，并且 SDK 及其所有依赖项的版本需要与开发人员使用的相匹配。通常在 Windows 项目中，你会发现安装了 Visual Studio 的 CI 服务器，以确保服务器具有与开发人员相同的工具。这使得构建服务器非常庞大，需要大量的努力来委托和维护。

这也意味着，除非你在你的机器上安装了.NET Core 2.2 SDK，否则你无法从本章的源代码构建这个 Docker 镜像。

通过使用多阶段构建，你可以兼顾两种选择，其中你的 Dockerfile 定义了一个步骤来编译你的应用程序，另一个步骤将其打包到最终镜像中。多阶段 Dockerfile 是可移植的，因此任何人都可以在没有先决条件的情况下构建镜像，但最终镜像只包含了应用程序所需的最小内容。

# 使用多阶段构建编译

在多阶段构建中，你的 Dockerfile 中有多个`FROM`指令，每个`FROM`指令在构建中启动一个新阶段。Docker 在构建镜像时执行所有指令，后续阶段可以访问前期阶段的输出，但只有最终阶段用于完成的镜像。

我可以通过将前两个 Dockerfile 合并成一个，为.NET Core 控制台应用程序编写一个多阶段的 Dockerfile：

[PRE13]

这里有一些新的东西。第一阶段使用了一个大的基础镜像，安装了.NET Core SDK。我使用`FROM`指令中的`AS`选项将这个阶段命名为`builder`。阶段的其余部分继续复制源代码并发布应用程序。当构建器阶段完成时，发布的应用程序将存储在一个中间容器中。

第二阶段使用了运行时.NET Core 镜像，其中没有安装 SDK。在这个阶段，我将从上一个阶段复制已发布的输出，在`COPY`指令中指定`--from=builder`。任何人都可以使用 Docker 从源代码编译这个应用程序，而不需要在他们的机器上安装.NET Core。

用于 Windows 应用程序的多阶段 Dockerfile 是完全可移植的。要编译应用程序并构建镜像，唯一的前提是要有一个安装了 Docker 的 Windows 机器和代码的副本。构建器阶段包含了 SDK 和所有编译器工具，但最终镜像只包含运行应用程序所需的最小内容。

这种方法不仅适用于.NET Core。你可以为.NET Framework 应用程序编写一个多阶段的 Dockerfile，其中第一阶段使用安装了 MSBuild 的镜像，用于编译你的应用程序。书中后面有很多这样的例子。

无论你采取哪种方法，都只需要理解几个 Dockerfile 指令，就可以构建更复杂的应用程序镜像，用于与其他系统集成的软件。

# 使用主要的 Dockerfile 指令

Dockerfile 语法非常简单。你已经看到了`FROM`、`COPY`、`USER`、`RUN`和`CMD`，这已经足够打包一个基本的应用程序以在容器中运行。对于真实世界的镜像，你需要做更多的工作，还有三个关键指令需要理解。

这是一个简单静态网站的 Dockerfile；它使用**Internet Information Services**（**IIS**）并在默认网站上提供一个 HTML 页面，显示一些基本细节：

[PRE14]

这个 Dockerfile 的开始方式不同，使用了`escape`指令。这告诉 Docker 使用反引号[PRE15]

docker image build --build-arg ENV_NAME=TEST --tag dockeronwindows/ch02-static-website:2e .

[PRE16]

> docker container run --detach --publish 8081:80 dockeronwindows/ch02-static-website:2e

6e3df776cb0c644d0a8965eaef86e377f8ebe036e99961a0621dcb7912d96980

[PRE17]

> docker container port 6e

80/tcp -> 0.0.0.0:8081

[PRE18]

# escape=` FROM mcr.microsoft.com/windows/nanoserver:1809 RUN md c:\data & `echo 'from image 1' > c:\data\file1.txt

[PRE19]

FROM dockeronwindows/ch02-fs-1:2e RUN echo 'from image 2' > c:\data\file2.txt

[PRE20]

docker container run -it --name c1 dockeronwindows/ch02-fs-2:2e

[PRE21]

C:\>dir C:\data

C 驱动器中的卷没有标签。

卷序列号为 BC8F-B36C

目录：C:\data

02/06/2019  11:00 AM    <DIR>          .

02/06/2019  11:00 AM    <DIR>          ..

02/06/2019  11:00 AM                17 file1.txt

02/06/2019  11:00 AM                17 file2.txt

[PRE22]

C:\>echo ' * ADDITIONAL * ' >> c:\data\file2.txt

C:\>echo 'New!' > c:\data\file3.txt

C:\>dir C:\data

C 驱动器中的卷没有标签。

卷序列号为 BC8F-B36C

目录：C:\data

02/06/2019  01:10 PM    <DIR>          .

02/06/2019  01:10 PM    <DIR>          ..

02/06/2019  11:00 AM                17 file1.txt

02/06/2019  01:10 PM                38 file2.txt

02/06/2019  01:10 PM                 9 file3.txt

[PRE23]

C:\> 退出

PS> docker container run -it --name c2 dockeronwindows/ch02-fs-2:2e

[PRE24]

C:\>dir C:\data

C 驱动器中的卷没有标签。

卷序列号为 BC8F-B36C

目录：C:\data

02/06/2019  11:00 AM    <DIR>          .

02/06/2019  11:00 AM    <DIR>          ..

02/06/2019  11:00 AM                17 file1.txt

02/06/2019  11:00 AM                17 file2.txt

[PRE25]

# escape=`

FROM mcr.microsoft.com/windows/nanoserver:1809 VOLUME C:\app\config VOLUME C:\app\logs USER ContainerAdministrator ENTRYPOINT cmd /S /C

[PRE26]

docker container run --name source dockeronwindows/ch02-volumes:2e "echo 'start' > c:\app\logs\log-1.txt"

[PRE27]

docker container run -it --volumes-from source dockeronwindows/ch02-volumes:2e cmd

[PRE28]

> ls C:\app

目录：C:\app

模式     最后写入时间      长度  名称

----     -------------      ------  ----

d----l   6/22/2017 8:11 AM          config

d----l   6/22/2017 8:11 AM          logs

[PRE29]

C:\>type C:\app\logs\log-1.txt

开始

C:\>echo 'more' >> C:\app\logs\log-1.txt

C:\>type C:\app\logs\log-1.txt

开始

更多

[PRE30]

> docker container run -it --volumes-from source:ro dockeronwindows/ch02-volumes:2e cmd

C:\>type C:\app\logs\log-1.txt

开始

更多

C:\>echo 'more' >> C:\app\logs\log-1.txt

拒绝访问。

C:\>echo 'new' >> C:\app\logs\log-2.txt

拒绝访问。

[PRE31]

> docker container inspect --format '{{ json .Mounts }}' source | ConvertFrom-Json

类型        : 卷

名称：65ab1b420a27bfd79d31d0d325622d0868e6b3f353c74ce3133888fafce972d9

来源：C：\ ProgramData \ docker \ volumes \ 65ab1b42 ... \ _data

目的地：c：\ app \ config

驱动程序：本地

RW：TruePropagation：

类型：卷

名称：b1451fde3e222adbe7f0f058a461459e243ac15af8770a2f7a4aefa7516e0761

来源：C：\ ProgramData \ docker \ volumes \ b1451fde ... \ _data

目的地：c：\ app \ logs

驱动程序：本地

RW：True

[PRE32]

> ls C：\ ProgramData \ docker \ volumes \ b1451fde ... \ _data

目录：C：\ ProgramData \ docker \ volumes \ b1451fde3e222adbe7f0f058a461459e243ac15af8770a2f7a4aefa7516e0761 \ _data

模式 LastWriteTime 长度名称

---- ------------- ------

-a---- 06/02/2019 13:33 19 log-1.txt

[PRE33]

PS> mkdir C：\ app-config | Out-Null

PS> echo 'VERSION = 18.09' > C：\ app-config \ version.txt

[PRE34]

> docker 容器运行`

--volume C：\ app-config：C：\ app \ config `

dockeronwindows / ch02-volumes：2e `

类型 C：\ app \ config \ version.txt

VERSION = 18.09

[PRE35]

# escape = `从 microsoft / dotnet：2.2-sdk-nanoserver-1809 AS 构建者的工作目录 C：\ src 复制 src。用户 ContainerAdministrator 运行 dotnet restore && dotnet publish # app image FROM microsoft / dotnet：2.2-aspnetcore-runtime-nanoserver-1809

EXPOSE 80 WORKDIR C：\ dotnetapp RUN mkdir app-state CMD ["dotnet", "HitCountWebApp.dll"] COPY --from=builder C：\ src \ bin \ Debug \ netcoreapp2.2 \ publish。

[PRE36]

docker image build --tag dockeronwindows / ch02-hitcount-website：2e-v1。

[PRE37]

mkdir C：\ app-state

docker 容器运行-d --publish-all`

-v C：\ app-state：C：\ dotnetapp \ app-state `

--name appv1 `

dockeronwindows / ch02-hitcount-website：2e-v1

[PRE38]

> docker 容器端口 appv1

80 / tcp-> 0.0.0.0：51377

[PRE39]

PS> docker 容器停止 appv1

appv1

PS> docker 容器运行-d --publish-all `

-v C：\ app-state：C：\ dotnetapp \ app-state `

--name appv2 `

dockeronwindows / ch02-hitcount-website：2e-v2

db8a39ba7af43be04b02d4ea5d9e646c87902594c26a62168c9f8bf912188b62

[PRE40]

# escape = `从 microsoft / dotnet-framework：4.7.2-sdk-windowsservercore-ltsc2019 AS 构建者的工作目录 C：\ src \ NerdDinner 复制 src \ NerdDinner \ packages.config。运行 nuget restore packages.config -PackagesDirectory .. \ packages COPY src C：\ src RUN msbuild NerdDinner.csproj / p：OutputPath = c：\ out / p：Configuration = Release

[PRE41]

FROM mcr.microsoft.com / dotnet / framework / aspnet：4.7.2-windowsservercore-ltsc2019 SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'] ENV BING_MAPS_KEY bing_maps_key WORKDIR C：\ nerd-dinner RUN Remove-Website -Name 'Default Web Site'; `

New-Website -Name 'nerd-dinner' ` -Port 80 -PhysicalPath 'c:\nerd-dinner' `-ApplicationPool '.NET v4.5' RUN & c:\windows\system32\inetsrv\appcmd.exe ` unlock config /section:system.webServer/handlers COPY --from=builder C:\out\_PublishedWebsites\NerdDinner C:\nerd-dinner

[PRE42]

docker container run -d -P dockeronwindows/ch02-nerd-dinner:2e

```

我可以使用`docker container port`来获取容器的发布端口，并浏览到 NerdDinner 的主页：

![](img/2e704bc2-8b9b-4e32-a63d-4207c87a2d38.png)

这是一个六年前的应用程序，在 Docker 容器中运行，没有代码更改。Docker 是一个很好的平台，可以用来构建新的应用程序和现代化旧的应用程序，但它也是一个很好的方式，可以将现有的应用程序从数据中心移到云端，或者将它们从不再支持的旧版本的 Windows 中移出，比如 Windows Server 2003 和（很快）Windows Server 2008。

在这一点上，这个应用程序还没有完全功能，我只是运行了一个基本版本。Bing Maps 对象没有显示真实的地图，因为我还没有提供 API 密钥。API 密钥是每个环境（每个开发人员、测试环境和生产环境）都会改变的东西。

在 Docker 中，你可以使用环境变量和配置对象来管理环境配置，我将在第三章中使用这些内容来进行 Dockerfile 的下一个迭代，*开发 Docker 化的.NET Framework 和.NET Core 应用程序*。

如果你在这个版本的 NerdDinner 中浏览并尝试注册一个新用户或搜索一个晚餐，你会看到一个黄色的崩溃页面告诉你数据库不可用。在其原始形式中，NerdDinner 使用 SQL Server LocalDB 作为轻量级数据库，并将数据库文件存储在应用程序目录中。我可以将 LocalDB 运行时安装到容器映像中，但这与 Docker 的哲学不符，即一个容器只运行一个应用程序。相反，我将为数据库构建一个单独的映像，这样我就可以在它自己的容器中运行它。

在下一章中，我将对 NerdDinner 示例进行迭代，添加配置管理，将 SQL Server 作为一个独立组件在自己的容器中运行，并演示如何通过使用 Docker 平台来开始现代化传统的 ASP.NET 应用程序。

# 总结

在本章中，我更仔细地看了 Docker 镜像和容器。镜像是应用程序的打包版本，容器是从镜像运行的应用程序的实例。您可以使用容器来执行简单的一次性任务，与它们进行交互，或者让它们在后台运行。随着您对 Docker 的使用越来越多，您会发现自己会做这三种事情。

Dockerfile 是构建镜像的源脚本。它是一个简单的文本文件，包含少量指令来指定基础镜像，复制文件和运行命令。您可以使用 Docker 命令行来构建镜像，这非常容易添加到您的 CI 构建步骤中。当开发人员推送通过所有测试的代码时，构建的输出将是一个有版本的 Docker 镜像，您可以将其部署到任何主机，知道它将始终以相同的方式运行。

在本章中，我看了一些简单的 Dockerfile，并以一个真实的应用程序结束了。NerdDinner 是一个传统的 ASP.NET MVC 应用程序，它是为在 Windows Server 和 IIS 上运行而构建的。使用多阶段构建，我将这个传统的应用程序打包成一个 Docker 镜像，并在容器中运行它。这表明 Docker 提供的新的计算模型不仅适用于使用.NET Core 和 Nano Server 的新项目，您还可以将现有的应用程序迁移到 Docker，并使自己处于一个良好的现代化起步位置。

在下一章中，我将使用 Docker 来现代化 NerdDinner 的架构，将功能分解为单独的组件，并使用 Docker 将它们全部连接在一起。

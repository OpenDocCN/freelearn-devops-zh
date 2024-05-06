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

```
> docker container run dockeronwindows/ch02-powershell-env:2e

Name                           Value
----                           -----
ALLUSERSPROFILE                C:\ProgramData
APPDATA                        C:\Users\ContainerAdministrator\AppData\Roaming
CommonProgramFiles             C:\Program Files\Common Files
CommonProgramFiles(x86)        C:\Program Files (x86)\Common Files
CommonProgramW6432             C:\Program Files\Common Files
COMPUTERNAME                   8A7D5B9A4021
...
```

没有任何选项，容器将运行内置于镜像中的 PowerShell 脚本，并且脚本将打印有关操作系统环境的一些基本信息。我将其称为**任务容器**，因为容器执行一个任务然后退出。

如果运行`docker container ls`，列出所有活动容器，您将看不到此容器。但如果运行`docker container ls --all`，显示所有状态的容器，您将在`Exited`状态中看到它：

```
> docker container ls --all
CONTAINER ID  IMAGE       COMMAND    CREATED          STATUS
8a7d5b9a4021 dockeronwindows/ch02-powershell-env:2e "powershell.exe C:..."  30 seconds ago   Exited
```

任务容器在自动化重复任务方面非常有用，比如运行脚本来设置环境、备份数据或收集日志文件。您的容器镜像打包了要运行的脚本，以及脚本所需的所有要求的确切版本，因此安装了 Docker 的任何人都可以运行脚本，而无需安装先决条件。

这对于 PowerShell 特别有用，因为脚本可能依赖于几个 PowerShell 模块。这些模块可能是公开可用的，但您的脚本可能依赖于特定版本。您可以构建一个已安装了模块的镜像，而不是共享一个需要用户安装许多不同模块的正确版本的脚本。然后，您只需要 Docker 来运行脚本任务。

镜像是自包含的单位，但您也可以将其用作模板。一个镜像可能配置为执行一项任务，但您可以以不同的方式从镜像运行容器以执行不同的任务。

# 连接到交互式容器

**交互式容器**是指与 Docker 命令行保持开放连接的容器，因此您可以像连接到远程机器一样使用容器。您可以通过指定交互式选项和容器启动时要运行的命令来从相同的 Windows Server Core 镜像运行交互式容器：

```
> docker container run --interactive --tty dockeronwindows/ch02-powershell-env:2e `
 powershell

Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\> Write-Output 'This is an interactive container'
This is an interactive container
PS C:\> exit
```

`--interactive`选项运行交互式容器，`--tty`标志将终端连接附加到容器。在容器映像名称后的`powershell`语句是容器启动时要运行的命令。通过指定命令，您可以替换映像中设置的启动命令。在这种情况下，我启动了一个 PowerShell 会话，它代替了配置的命令，因此环境打印脚本不会运行。

交互式容器会持续运行，只要其中的命令在运行。当您连接到 PowerShell 时，在主机的另一个窗口中运行`docker container ls`，会显示容器仍在运行。当您在容器中键入`exit`时，PowerShell 会话结束，因此没有进程在运行，容器也会退出。

交互式容器在构建自己的容器映像时非常有用，它们可以让您首先以交互方式进行步骤，并验证一切是否按您的预期工作。它们也是很好的探索工具。您可以从 Docker 注册表中拉取别人的映像，并在运行应用程序之前探索其内容。

阅读本书时，您会发现 Docker 可以在虚拟网络中托管复杂的分布式系统，每个组件都在自己的容器中运行。如果您想检查系统的某些部分，可以在网络内部运行交互式容器，并检查各个组件，而无需使部分公开可访问。

# 在后台容器中保持进程运行

最后一种类型的容器是您在生产中最常使用的，即后台容器，它在后台保持长时间运行的进程。它是一个行为类似于 Windows 服务的容器。在 Docker 术语中，它被称为**分离容器**，Docker 引擎会在后台保持其运行。在容器内部，进程在前台运行。该进程可能是一个 Web 服务器或一个轮询消息队列以获取工作的控制台应用程序，但只要进程保持运行，Docker 就会保持容器保持活动状态。

我可以再次从相同的映像运行后台容器，指定`detach`选项和运行一些分钟的命令：

```
> docker container run --detach dockeronwindows/ch02-powershell-env:2e `
 powershell Test-Connection 'localhost' -Count 100

bb326e5796bf48199a9a6c4569140e9ca989d7d8f77988de7a96ce0a616c88e9
```

在这种情况下，当容器启动后，控制返回到终端；长随机字符串是新容器的 ID。您可以运行`docker container ls`并查看正在运行的容器，`docker container logs`命令会显示容器的控制台输出。对于操作特定容器的命令，您可以通过容器名称或容器 ID 的一部分来引用它们 - ID 是随机的，在我的情况下，这个容器 ID 以`bb3`开头：

```
> docker container logs bb3

Source        Destination     IPV4Address      IPV6Address
------        -----------     -----------      -----------
BB326E5796BF  localhost       127.0.0.1        ::1
BB326E5796BF  localhost       127.0.0.1        ::1
```

`--detach`标志将容器分离，使其进入后台，而在这种情况下，命令只是重复一百次对`localhost`的 ping。几分钟后，PowerShell 命令完成，因此没有正在运行的进程，容器退出。

这是一个需要记住的关键事情：如果您想要在后台保持容器运行，那么 Docker 在运行容器时启动的进程必须保持运行。

现在您已经看到容器是从镜像创建的，但它可以以不同的方式运行。因此，您可以完全按照准备好的镜像使用，或者将镜像视为内置默认启动模式的模板。接下来，我将向您展示如何构建该镜像。

# 构建 Docker 镜像

Docker 镜像是分层的。底层是操作系统，可以是完整的操作系统，如 Windows Server Core，也可以是微软 Nano Server 等最小的操作系统。在此之上是每次构建镜像时对基本操作系统所做更改的层，通过安装软件、复制文件和运行命令。从逻辑上讲，Docker 将镜像视为单个单位，但从物理上讲，每个层都存储为 Docker 缓存中的单独文件，因此具有许多共同特征的镜像可以共享缓存中的层。

镜像是使用 Dockerfile 语言的文本文件构建的 - 指定要从哪个基本操作系统镜像开始以及添加的所有步骤。这种语言非常简单，您只需要掌握几个命令就可以构建生产级别的镜像。我将从查看到目前为止在本章中一直在使用的基本 PowerShell 镜像开始。

# 理解 Dockerfile

Dockerfile 只是一个将软件打包到 Docker 镜像中的部署脚本。PowerShell 镜像的完整代码只有三行：

```
FROM mcr.microsoft.com/windows/servercore:ltsc2019  COPY scripts/print-env-details.ps1 C:\\print-env.ps1 CMD ["powershell.exe", "C:\\print-env.ps1"]
```

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

```
docker image build --tag dockeronwindows/ch02-powershell-env:2e .
```

每个镜像都需要一个标签，使用`--tag`选项指定，这是本地镜像缓存和镜像注册表中镜像的唯一标识符。标签是你在运行容器时将引用镜像的方式。完整的标签指定要使用的注册表：存储库名称，这是应用程序的标识符，以及后缀，这是镜像的版本标识符。

当你为自己构建一个镜像时，你可以随意命名，但约定是将你的存储库命名为你的注册表用户名，后面跟上应用程序名称：`{user}/{app}`。你还可以使用标签来标识应用程序的版本或变体，比如`sixeyed/git`和`sixeyed/git:2.17.1-windowsservercore-ltsc2019`，这是 Docker Hub 上我的两个镜像。

`image build`命令末尾的句点告诉 Docker 要使用的上下文的位置。`.`是当前目录。Docker 将目录树的内容复制到一个临时文件夹进行构建，因此上下文需要包含 Dockerfile 中引用的任何文件。复制上下文后，Docker 开始执行 Dockerfile 中的指令。

# 检查 Docker 构建镜像的过程

理解 Docker 镜像是如何构建的将有助于您构建高效的镜像。`image build`命令会产生大量输出，告诉您 Docker 在构建的每个步骤中做了什么。Dockerfile 中的每个指令都会作为一个单独的步骤执行，产生一个新的镜像层，最终镜像将是所有层的组合堆栈。以下代码片段是构建我的镜像的输出：

```
> docker image build --tag dockeronwindows/ch02-powershell-env:2e .

Sending build context to Docker daemon  4.608kB
Step 1/3 : FROM mcr.microsoft.com/windows/servercore:ltsc2019
 ---> 8b79386f6e3b
Step 2/3 : COPY scripts/print-env-details.ps1 C:\\print-env.ps1
 ---> 5e9ed4527b3f
Step 3/3 : CMD ["powershell.exe", "C:\\print-env.ps1"]
 ---> Running in c14c8aef5dc5
Removing intermediate container c14c8aef5dc5
 ---> 5f272fb2c190
Successfully built 5f272fb2c190
Successfully tagged dockeronwindows/ch02-powershell-env:2e
```

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

```
FROM microsoft/dotnet:2.2-sdk-nanoserver-1809 WORKDIR /src COPY src/ . USER ContainerAdministrator RUN dotnet restore && dotnet build CMD ["dotnet", "run"]
```

Dockerfile 使用了来自 Docker Hub 的 Microsoft 的.NET Core 图像作为基础图像。这是图像的一个特定变体，它基于 Nano Server 1809 版本，并安装了.NET Core 2.2 SDK。构建将应用程序源代码从上下文中复制进来，并在容器构建过程中编译应用程序。

这个 Dockerfile 中有三个你以前没有见过的新指令：

1.  `WORKDIR`指定当前工作目录。如果目录在中间容器中不存在，Docker 会创建该目录，并将其设置为当前目录。它将保持为 Dockerfile 中的后续指令以及从图像运行的容器的工作目录。

1.  `USER`更改构建中的当前用户。Nano Server 默认使用最低特权用户。这将切换到容器图像中的内置帐户，该帐户具有管理权限。

1.  `RUN`在中间容器中执行命令，并在命令完成后保存容器的状态，创建一个新的图像层。

当我构建这个图像时，你会看到`dotnet`命令的输出，这是应用程序从图像构建中的`RUN`指令中编译出来的：

```
> docker image build --tag dockeronwindows/ch02-dotnet-helloworld:2e . 
Sending build context to Docker daemon  192.5kB
Step 1/6 : FROM microsoft/dotnet:2.2-sdk-nanoserver-1809
 ---> 90724d8d2438
Step 2/6 : WORKDIR /src
 ---> Running in f911e313b262
Removing intermediate container f911e313b262
 ---> 2e2f7deb64ac
Step 3/6 : COPY src/ .
 ---> 391c7d8f4bcc
Step 4/6 : USER ContainerAdministrator
 ---> Running in f08f860dd299
Removing intermediate container f08f860dd299
 ---> 6840a2a2f23b
Step 5/6 : RUN dotnet restore && dotnet build
 ---> Running in d7d61372a57b

Welcome to .NET Core!
...
```

你会在 Docker Hub 上经常看到这种方法，用于使用.NET Core、Go 和 Node.js 等语言构建的应用程序，其中工具很容易添加到基础镜像中。这意味着你可以在 Docker Hub 上设置自动构建，这样当你将代码更改推送到 GitHub 时，Docker 的服务器就会根据 Dockerfile 构建你的镜像。服务器可以在没有安装.NET Core、Go 或 Node.js 的情况下执行此操作，因为所有构建依赖项都在基础镜像中。

这种选项意味着最终镜像将比生产应用程序所需的要大得多。语言 SDK 和工具可能会占用比应用程序本身更多的磁盘空间，但你的最终结果应该是应用程序；当容器在生产环境运行时，镜像中占用空间的所有构建工具都不会被使用。另一种选择是首先构建应用程序，然后将编译的二进制文件打包到你的容器镜像中。

# 在构建之前编译应用程序

首先构建应用程序与现有的构建流水线完美契合。你的构建服务器需要安装所有的应用程序平台和构建工具来编译应用程序，但你的最终容器镜像只包含运行应用程序所需的最小内容。采用这种方法，我的.NET Core 应用程序的 Dockerfile 变得更加简单：

```
FROM  microsoft/dotnet:2.2-runtime-nanoserver-1809

WORKDIR /dotnetapp
COPY ./src/bin/Debug/netcoreapp2.2/publish .

CMD ["dotnet", "HelloWorld.NetCore.dll"]
```

这个 Dockerfile 使用了一个不同的`FROM`镜像，其中只包含.NET Core 2.2 运行时，而不包含工具（因此它可以运行已编译的应用程序，但无法从源代码编译）。你不能在构建应用程序之前构建这个镜像，所以你需要在构建脚本中包装`docker image build`命令，该脚本还运行`dotnet publish`命令来编译二进制文件。

一个简单的构建脚本，用于编译应用程序并构建 Docker 镜像，看起来像这样：

```
dotnet restore src; dotnet publish src

docker image build --file Dockerfile.slim --tag dockeronwindows/ch02-dotnet-helloworld:2e-slim .
```

如果你把 Dockerfile 指令放在一个名为**Dockerfile**之外的文件中，你需要使用`--file`选项指定文件名：`docker image build --file Dockerfile.slim`。

我把平台工具的要求从镜像移到了构建服务器上，这导致最终镜像变得更小：与之前版本相比，这个版本的大小为 410 MB，而之前的版本为 1.75 GB。你可以通过列出镜像并按照镜像仓库名称进行过滤来看到大小的差异：

```
> docker image ls --filter reference=dockeronwindows/ch02-dotnet-helloworld

REPOSITORY                               TAG     IMAGE ID       CREATED              SIZE
dockeronwindows/ch02-dotnet-helloworld   2e-slim b6e7dca114a4   About a minute ago   410MB
dockeronwindows/ch02-dotnet-helloworld   2e      bf895a7452a2   7 minutes ago        1.75GB
```

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

```
# build stage
FROM microsoft/dotnet:2.2-sdk-nanoserver-1809 AS builder

WORKDIR /src
COPY src/ .

USER ContainerAdministrator
RUN dotnet restore && dotnet publish

# final image stage
FROM microsoft/dotnet:2.2-runtime-nanoserver-1809

WORKDIR /dotnetapp
COPY --from=builder /src/bin/Debug/netcoreapp2.2/publish .

CMD ["dotnet", "HelloWorld.NetCore.dll"]
```

这里有一些新的东西。第一阶段使用了一个大的基础镜像，安装了.NET Core SDK。我使用`FROM`指令中的`AS`选项将这个阶段命名为`builder`。阶段的其余部分继续复制源代码并发布应用程序。当构建器阶段完成时，发布的应用程序将存储在一个中间容器中。

第二阶段使用了运行时.NET Core 镜像，其中没有安装 SDK。在这个阶段，我将从上一个阶段复制已发布的输出，在`COPY`指令中指定`--from=builder`。任何人都可以使用 Docker 从源代码编译这个应用程序，而不需要在他们的机器上安装.NET Core。

用于 Windows 应用程序的多阶段 Dockerfile 是完全可移植的。要编译应用程序并构建镜像，唯一的前提是要有一个安装了 Docker 的 Windows 机器和代码的副本。构建器阶段包含了 SDK 和所有编译器工具，但最终镜像只包含运行应用程序所需的最小内容。

这种方法不仅适用于.NET Core。你可以为.NET Framework 应用程序编写一个多阶段的 Dockerfile，其中第一阶段使用安装了 MSBuild 的镜像，用于编译你的应用程序。书中后面有很多这样的例子。

无论你采取哪种方法，都只需要理解几个 Dockerfile 指令，就可以构建更复杂的应用程序镜像，用于与其他系统集成的软件。

# 使用主要的 Dockerfile 指令

Dockerfile 语法非常简单。你已经看到了`FROM`、`COPY`、`USER`、`RUN`和`CMD`，这已经足够打包一个基本的应用程序以在容器中运行。对于真实世界的镜像，你需要做更多的工作，还有三个关键指令需要理解。

这是一个简单静态网站的 Dockerfile；它使用**Internet Information Services**（**IIS**）并在默认网站上提供一个 HTML 页面，显示一些基本细节：

```
# escape=` FROM mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2019
SHELL ["powershell"]

ARG ENV_NAME=DEV

EXPOSE 80

COPY template.html C:\template.html
RUN (Get-Content -Raw -Path C:\template.html) `
 -replace '{hostname}', [Environment]::MachineName `
 -replace '{environment}', [Environment]::GetEnvironmentVariable('ENV_NAME') `
 | Set-Content -Path C:\inetpub\wwwroot\index.html
```

这个 Dockerfile 的开始方式不同，使用了`escape`指令。这告诉 Docker 使用反引号`` ` ``用于转义字符的选项，将命令拆分为多行，而不是默认的反斜杠`\`选项。通过这个转义指令，我可以在文件路径中使用反斜杠和反斜杠来拆分较长的 PowerShell 命令，这对于 Windows 用户来说更加自然。

基本图像是`microsoft/iis`，这是一个已经设置好 IIS 的 Microsoft Windows Server Core 图像。我将一个 HTML 模板文件从 Docker 构建上下文复制到根文件夹中。然后，我运行一个 PowerShell 命令来更新模板文件的内容，并将其保存在 IIS 的默认网站位置。

在此 Dockerfile 中，我使用了三条新指令：

+   `SHELL`指定在`RUN`命令中使用的命令行。默认值是`cmd`，这会切换到`powershell`。

+   `ARG`指定要在带有默认值的图像中使用的构建参数。

+   `EXPOSE`将使图像中的端口可用，以便从主机中向图像的容器发送流量。

此静态网站有一个单一的主页，其中告诉您发送响应的服务器的名称，并在页面标题中显示环境的名称。HTML 模板文件具有主机名和环境名称的占位符。`RUN`命令执行一个 PowerShell 脚本来读取文件内容，用实际主机名和环境值替换占位符，然后写出内容。

容器在一个隔离的空间中运行，只有在图像明确地使端口可用的情况下，主机才能将网络流量发送到容器中。那就是`EXPOSE`指令，它类似于一个非常简单的防火墙；您可以使用它来暴露应用程序正在侦听的端口。当您从此图像运行容器时，端口`80`可供发布，因此 Docker 可以从容器中提供 Web 流量。

我可以按照通常的方式构建此图像，并利用 Dockerfile 中指定的`ARG`命令，在构建时使用`--build-arg`选项来覆盖默认值：

```

docker image build --build-arg ENV_NAME=TEST --tag dockeronwindows/ch02-static-website:2e .

```

Docker 处理新的指令的方式与您已经看到的指令相同：它从堆栈中的前一个图像创建一个新的中间容器，执行指令，并从容器中提取新的图像层。构建后，我有一个新的图像，可以运行以启动静态 Web 服务器：

```

> docker container run --detach --publish 8081:80 dockeronwindows/ch02-static-website:2e

6e3df776cb0c644d0a8965eaef86e377f8ebe036e99961a0621dcb7912d96980

```

这是一个分离的容器，因此它在后台运行，而`--publish`选项使容器中的端口`80`对主机可用。发布的端口意味着主机进入的流量可以通过 Docker 定向到容器中。我指定主机上的端口`8081`应映射到容器中的端口`80`。

您还可以让 Docker 在主机上选择一个随机端口，并使用`port`命令列出容器公开的端口以及它们在主机上的发布位置：

```

> docker container port 6e

80/tcp -> 0.0.0.0:8081

```

现在，我可以在我的计算机上浏览到端口`8081`，并查看来自容器内运行的 IIS 的响应，显示给我容器的主机名，实际上是容器 ID，以及标题栏中的环境名称：

![](img/76a58414-9209-4783-a22d-ef44c904ef3b.png)

环境名称只是描述文本，但是传递给`docker image build`命令的参数值会覆盖 Dockerfile 中`ARG`指令的默认值。主机名应该显示容器 ID，但是当前实现存在问题。

在网页上，主机名以`bf37`开头，但实际上我的容器 ID 以`6e3d`开头。为了理解为什么显示的 ID 不是正在运行的容器的实际 ID，我将再次查看构建映像期间使用的临时容器。

# 理解临时容器和映像状态

我的网站容器具有以`6e3d`开头的 ID，这是容器内应用程序应看到的主机名，但网站声称不是。那么，出了什么问题？请记住，Docker 在临时中间容器中执行每个构建指令。

生成 HTML 的`RUN`指令在临时容器中运行，因此 PowerShell 脚本将该容器的 ID 写入 HTML 文件作为主机名。这就是起始于`bf37`的容器 ID 的来源。Docker 会删除中间容器，但是它创建的 HTML 文件将保留在映像中。

这是一个重要的概念：当您构建 Docker 映像时，指令在临时容器内执行。容器会被移除，但是它们写入的状态将保留在最终映像中，并将在从该映像运行的任何容器中存在。如果我从我的网站映像运行多个容器，它们将在 HTML 文件中显示相同的主机名，因为它保存在映像中，该映像由所有容器共享。

当然，您也可以将状态存储在单独的容器中，这不是映像的一部分，因此不会在容器之间共享。我将现在介绍如何使用 Docker 的数据，并以真实的 Dockerfile 示例结束本章。

# 使用 Docker 映像和容器中的数据

在 Docker 容器中运行的应用程序看到的是单个文件系统，可以按照操作系统的惯例读取和写入。容器看到的是单个文件系统驱动器，但实际上是一个虚拟文件系统，底层数据可以位于许多不同的物理位置。

容器可以访问其 C 驱动器上的文件，这些文件实际上可以存储在镜像层中，在容器自己的存储层中或者映射到主机上某个位置的卷中。 Docker 将所有这些位置合并为单个虚拟文件系统。

# 层和虚拟 C 驱动器中的数据

虚拟文件系统是 Docker 如何将一组物理镜像层视为一个逻辑容器映像的方式。镜像层被挂载为容器文件系统的只读部分，因此它们无法被更改，这就是它们可以被许多容器安全共享的方式。

每个容器在所有只读层的顶部都有自己的可写层，因此每个容器都可以修改自己的数据而不影响其他容器：

![](img/e5203dc0-e4fa-4e26-a79f-6665788e9d60.png)

该图显示了从相同镜像运行的两个容器。镜像 (1) 由许多层物理组成：每个 Dockerfile 中的指令构建一个层。两个容器 (2 和 3) 在运行时使用相同的镜像层，但它们各自具有其自己的隔离、可写层。

Docker 向容器呈现单个文件系统。层和只读基础层的概念被隐藏起来，你的容器只是读取和写入数据，就好像它拥有一个完整的本机文件系统，带有一个单独的驱动器。如果在构建 Docker 镜像时创建一个文件，然后在容器内编辑该文件，Docker 实际上会在容器的可写层中创建一个已更改文件的副本，并隐藏原始的只读文件。因此，容器中有一个已编辑文件的副本，但镜像中的原始文件保持不变。

你可以通过创建一些具有不同层中的数据的简单镜像来看到这一点。用于`dockeronwindows/ch02-fs-1:2e`镜像的 Dockerfile 使用 Nano Server 作为基础镜像，创建一个目录，并在其中写入一个文件：

```

# escape=` FROM mcr.microsoft.com/windows/nanoserver:1809 RUN md c:\data & `echo 'from image 1' > c:\data\file1.txt

```

用于`dockeronwindows/ch02-fs-2:2e`镜像的 Dockerfile 创建了一个基于该镜像的镜像，并向数据目录添加了第二个文件：

```

FROM dockeronwindows/ch02-fs-1:2e RUN echo 'from image 2' > c:\data\file2.txt

```

*基础*镜像没有任何特殊之处；任何镜像都可以在新镜像的`FROM`指令中使用。它可以是 Docker Hub 上的官方镜像，来自私有注册表的商业镜像，从头开始构建的本地镜像，或者是在层次结构中多层深的镜像。

我将构建两个镜像，并从`dockeronwindows/ch02-fs-2:2e`运行一个交互式容器，以便我可以查看`C`驱动器上的文件。此命令启动一个容器，并为其指定了一个显式名称`c1`，这样我就可以在不使用随机容器 ID 的情况下使用它：

```

docker container run -it --name c1 dockeronwindows/ch02-fs-2:2e

```

Docker 命令中的许多选项都有短格式和长格式。长格式以两个短横线开头，如`--interactive`。短格式是一个单独的字母，并以单个短横线开头，如`-i`。短标记可以组合使用，因此`-it`等同于`-i -t`，等同于`--interactive --tty`。运行`docker --help`来浏览命令及其选项。

Nano Server 是一个精简的操作系统，专为在容器中运行应用程序而构建。它不是 Windows 的完整版本，你不能在虚拟机或物理机上运行 Nano Server，也不能在 Nano Server 容器中运行所有 Windows 应用程序。基础镜像特意设计得很小，甚至连 PowerShell 也没有包含在内，以减少表面积，意味着你需要更少的更新，攻击向量也更少。

你需要重新熟悉旧的 DOS 命令以使用 Nano Server 容器。`dir`列出容器内的目录内容：

```

C:\>dir C:\data
 Volume in drive C has no label.
 Volume Serial Number is BC8F-B36C

 Directory of C:\data

02/06/2019  11:00 AM    <DIR>          .
02/06/2019  11:00 AM    <DIR>          ..
02/06/2019  11:00 AM                17 file1.txt
02/06/2019  11:00 AM                17 file2.txt 
```

这两个文件都在容器中用于在`C:\data`目录中使用；第一个文件在来自`ch02-fs-1:2e`镜像的一个层中，而第二个文件在来自`ch02-fs-2:2e`镜像的一个层中。`dir`可执行文件来自基础 Nano Server 镜像的另一个层，并且容器以相同的方式看待它们。

我将向现有文件中追加一些文本，并在`c1`容器中创建一个新文件：

```

C:\>echo ' * ADDITIONAL * ' >> c:\data\file2.txt

C:\>echo 'New!' > c:\data\file3.txt

C:\>dir C:\data
 Volume in drive C has no label.
 Volume Serial Number is BC8F-B36C

 Directory of C:\data

02/06/2019  01:10 PM    <DIR>          .
02/06/2019  01:10 PM    <DIR>          ..
02/06/2019  11:00 AM                17 file1.txt
02/06/2019  01:10 PM                38 file2.txt
02/06/2019  01:10 PM                 9 file3.txt 
```

从文件列表中你可以看到来自镜像层的`file2.txt`已经被修改，并且有一个新文件`file3.txt`。现在我会退出这个容器，并使用相同的镜像创建一个新容器：

```

C:\> exit
PS> docker container run -it --name c2 dockeronwindows/ch02-fs-2:2e 


```

在这个新容器的`C:\data`目录中，你期望看到什么？让我们来看看：

```

C:\>dir C:\data
 Volume in drive C has no label.
 Volume Serial Number is BC8F-B36C

 Directory of C:\data

02/06/2019  11:00 AM    <DIR>          .
02/06/2019  11:00 AM    <DIR>          ..
02/06/2019  11:00 AM                17 file1.txt
02/06/2019  11:00 AM                17 file2.txt 
```

你知道镜像层是只读的，每个容器都有自己的可写层，所以结果应该是有意义的。新容器`c2`具有来自镜像的原始文件，没有来自第一个容器`c1`的更改，这些更改存储在`c1`的可写层中。每个容器的文件系统是隔离的，所以一个容器看不到另一个容器所做的任何更改。

如果你想在容器之间或在容器和主机之间共享数据，你可以使用 Docker 卷。

# 使用卷在容器之间共享数据

卷是存储单元。它们有一个与容器不同的生命周期，因此它们可以独立创建，然后在一个或多个容器中挂载。你可以使用 Dockerfile 中的`VOLUME`指令确保容器始终使用卷存储。

你可以用目标目录指定卷，这是容器内的位置，卷在其中显示。当你运行一个在镜像中定义了卷的容器时，卷被映射到主机上的一个物理位置，该位置特定于该容器。从相同镜像运行的更多容器将使它们的卷映射到不同的主机位置。

在 Windows 中，卷目录需要为空。在你的 Dockerfile 中，你不能在一个目录中创建文件，然后将其公开为卷。卷还需要在镜像中存在的磁盘上定义。在 Windows 基础镜像中，只有一个`C`盘可用，所以卷需要在`C`盘上创建。

`dockeronwindows/ch02-volumes:2e`的 Dockerfile 创建了一个具有两个卷的镜像，并且在从镜像运行容器时明确指定了`cmd` shell 作为`ENTRYPOINT`：

```

# escape=`

FROM mcr.microsoft.com/windows/nanoserver:1809 VOLUME C:\app\config VOLUME C:\app\logs USER ContainerAdministrator ENTRYPOINT cmd /S /C

```

请记住，Nano Server 镜像默认使用最低权限用户。这个用户无法访问卷，所以这个 Dockerfile 切换到管理帐户，当你从镜像运行一个容器时，你可以访问卷目录。

当我从该镜像运行一个容器时，Docker 会从三个源创建一个虚拟文件系统。镜像层是只读的，容器的层是可写的，卷可以设置为只读或可写：

![](img/f630b9fe-5ce9-44cb-b8ac-9f76da4fb8ab.png)

因为卷是与容器分开的，所以即使源容器未运行，它们也可以与其他容器共享。我可以从该映像运行一个任务容器，并使用命令在卷中创建新文件：

```

docker container run --name source dockeronwindows/ch02-volumes:2e "echo 'start' > c:\app\logs\log-1.txt"

```

Docker 启动容器，该容器写入文件，然后退出。容器及其卷尚未被删除，因此我可以使用`--volumes-from`选项连接到另一个容器中的卷，并指定我的第一个容器的名称：

```

docker container run -it --volumes-from source dockeronwindows/ch02-volumes:2e cmd

```

这是一个交互式容器，当我列出`C:\app`目录的内容时，我将看到两个目录，`logs`和`config`，它们是第一个容器中的卷：

```

> ls C:\app

 Directory: C:\app

Mode     LastWriteTime      Length  Name
----     -------------      ------  ----
d----l   6/22/2017 8:11 AM          config
d----l   6/22/2017 8:11 AM          logs 
```

共享卷具有读写访问权限，因此我可以看到在第一个容器中创建的文件并向其追加：

```

C:\>type C:\app\logs\log-1.txt
'start'

C:\>echo 'more' >> C:\app\logs\log-1.txt

C:\>type C:\app\logs\log-1.txt
'start'
'more'
```

这样在容器之间共享数据非常有用；您可以运行一个任务容器，从长时间运行的后台容器中备份数据或日志文件。默认访问权限是卷可写的，但这是需要注意的一点，因为您可能会编辑数据并破坏运行在源容器中的应用程序。

Docker 允许您以只读模式从另一个容器中挂载卷，方法是在`--volumes-from`选项中将`：ro`标志添加到容器名称。如果您只想阅读而不做更改，这是访问数据的更安全方式。我将运行一个新的容器，在只读模式下共享与原始容器相同的卷：

```

> docker container run -it --volumes-from source:ro dockeronwindows/ch02-volumes:2e cmd

C:\>type C:\app\logs\log-1.txt
'start'
'more'

C:\>echo 'more' >> C:\app\logs\log-1.txt
Access is denied.

C:\>echo 'new' >> C:\app\logs\log-2.txt
Access is denied.
```

在新容器中，我无法创建新文件或向现有日志文件中写入内容，但我可以看到原始容器中的日志文件中的内容，以及第二个容器追加的行。

# 使用卷在容器和主机之间共享数据

容器卷存储在主机上，因此您可以直接从运行 Docker 的机器访问它们，但它们将位于 Docker 的程序数据目录中的某个嵌套目录中。`docker container inspect`命令告诉您容器卷的物理位置，以及更多信息，包括容器的 ID、名称以及 Docker 网络中容器的虚拟 IP 地址。

我可以在`container inspect`命令中使用 JSON 格式化，传递一个查询以提取`Mounts`字段中的卷信息。此命令将 Docker 输出传递到一个 PowerShell cmdlet 中，以友好的格式显示 JSON：

```

> docker container inspect --format '{{ json .Mounts }}' source | ConvertFrom-Json

Type        : volume
Name        : 65ab1b420a27bfd79d31d0d325622d0868e6b3f353c74ce3133888fafce972d9
Source      : C:\ProgramData\docker\volumes\65ab1b42...\_data
Destination : c:\app\config
Driver      : local
RW          : TruePropagation :

Type        : volume
Name        : b1451fde3e222adbe7f0f058a461459e243ac15af8770a2f7a4aefa7516e0761
Source      : C:\ProgramData\docker\volumes\b1451fde...\_data
Destination : c:\app\logs
Driver      : local
RW          : True
```

我已经缩写了输出，但在`Source`字段中，您可以看到卷数据存储在主机上的完整路径。我可以直接从主机访问容器的文件，使用该源目录。当我在我的 Windows 机器上运行此命令时，我将看到在容器卷内创建的文件：

```

> ls C:\ProgramData\docker\volumes\b1451fde...\_data
   Directory: C:\ProgramData\docker\volumes\b1451fde3e222adbe7f0f058a461459e243ac15af8770a2f7a4aefa7516e0761\_data

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----       06/02/2019     13:33             19 log-1.txt
```

通过这种方式访问主机上的文件是可能的，但使用带有卷 ID 的嵌套目录位置非常麻烦。相反，您可以在创建容器时从主机的特定位置挂载卷。

# 从主机目录挂载卷

你可以使用 `--volume` 选项将容器中的一个目录映射到主机上的一个已知位置。容器中的目标位置可以是使用 `VOLUME` 命令创建的目录，或者容器文件系统中的任何目录。如果 Docker 镜像中的目标位置已经存在，它将被卷挂载隐藏，因此你将看不到任何镜像文件。

我将在我的 Windows 机器的 `C` 盘上的一个目录中创建一个虚拟配置文件：

```

PS> mkdir C：\ app-config | Out-Null

PS> echo 'VERSION = 18.09' > C：\ app-config \ version.txt

```

现在我将运行一个容器，该容器从主机映射一个卷，并读取实际存储在主机上的配置文件：

```

> docker container run `
 --volume C:\app-config:C:\app\config `
 dockeronwindows/ch02-volumes:2e `
 type C:\app\config\version.txt
VERSION=18.09
```

`--volume` 选项指定挂载的格式为 `{source}:{target}`。源是主机位置，需要存在。目标是容器位置，不需要存在，但如果存在，现有内容将被隐藏。

Windows 和 Linux 容器中的卷挂载不同。在 Linux 容器中，Docker 将源的内容合并到目标中，因此如果镜像中存在文件，则你会看到它们以及卷源的内容。Linux 上的 Docker 也允许你挂载单个文件位置，但在 Windows 上，你只能挂载整个目录。

卷挂载对于在容器中运行有状态的应用程序（如数据库）非常有用。你可以在容器中运行 SQL Server，并将数据库文件存储在主机上的某个位置，这可以是服务器上的 RAID 阵列。当你有模式更新时，你删除旧的容器，并从更新的 Docker 镜像启动新的容器。你使用相同的卷挂载为新容器，以便从旧容器中保留数据。

# 使用卷来进行配置和状态管理

当你在容器中运行应用程序时，应用程序状态是一个重要考虑因素。容器可以长时间运行，但并不打算是永久的。与传统计算模型相比，容器的最大优势之一是你可以轻松替换它们，而且只需几秒钟的时间。当你有一个新功能要部署，或者要修补的安全漏洞时，你只需构建和测试一个升级后的镜像，停止旧的容器，然后从新镜像启动一个替代容器。

通过将数据与应用程序容器分开，卷让你管理升级过程。我将用一个简单的 Web 应用程序演示这一点，该应用程序将页面的点击计数存储在文本文件中；每当你浏览到页面时，网站都会增加计数。

`dockeronwindows/ch02-hitcount-website` 镜像的 Dockerfile 使用多阶段构建，使用 `microsoft/dotnet` 镜像编译应用程序，并使用 `microsoft/aspnetcore` 作为基础打包最终应用程序：

```

# escape=`
FROM microsoft/dotnet:2.2-sdk-nanoserver-1809 AS builder

WORKDIR C:\src
COPY src .

USER ContainerAdministrator
RUN dotnet restore && dotnet publish

# app image
FROM microsoft/dotnet:2.2-aspnetcore-runtime-nanoserver-1809

EXPOSE 80
WORKDIR C:\dotnetapp
RUN mkdir app-state

CMD ["dotnet", "HitCountWebApp.dll"]
COPY --from=builder C:\src\bin\Debug\netcoreapp2.2\publish .
```

在 Dockerfile 中，我在 `C:\dotnetapp\app-state` 创建了一个空目录，这是应用程序将在其中的文本文件中存储点击计数的位置。我已经使用 `2e-v1` 标签将应用程序的第一个版本构建成一个镜像：

```

docker image build --tag dockeronwindows / ch02-hitcount-website：2e-v1。

```

我会在主机上创建一个目录用于容器的状态，并运行一个容器，将应用程序状态目录从主机上的一个目录挂载到容器中：

```

mkdir C:\app-state

docker container run -d --publish-all `
 -v C:\app-state:C:\dotnetapp\app-state `
 --name appv1 `
 dockeronwindows/ch02-hitcount-website:2e-v1 
 ```

`publish-all` 选项告诉 Docker 将容器镜像的所有暴露端口发布到主机上的随机端口。这是在本地环境中测试容器的快速选项，因为 Docker 将从主机分配一个空闲端口，你不需要担心其他容器已经使用了哪些端口。你可以使用 `container port` 命令查看容器发布的端口：

```

> docker container port appv1
80/tcp -> 0.0.0.0:51377

```

我可以在 `http://localhost:51377` 上浏览到该站点。当我刷新页面几次时，会看到点击数增加：

![](img/47bcf624-7bdc-478d-a735-0edc36d20e14.png)

现在，当我有一个要部署的升级版本的应用程序时，我可以将其打包成一个带有 `2e-v2` 标签的新镜像。当镜像准备好时，我可以停止旧容器并启动一个新容器，使用相同的卷映射：

```

PS> docker container stop appv1
appv1

PS> docker container run -d --publish-all `
 -v C:\app-state:C:\dotnetapp\app-state `
 --name appv2 `
 dockeronwindows/ch02-hitcount-website:2e-v2

db8a39ba7af43be04b02d4ea5d9e646c87902594c26a62168c9f8bf912188b62
```

包含应用程序状态的卷被重用，因此新版本将继续使用旧版本的保存状态。我有一个新的容器，带有一个新发布的端口。当我获取端口并首次浏览时，我会看到更新的 UI 和一个吸引人的图标，但点击数从版本 1 中继承下来：

![](img/6d464a9f-b169-4970-9b8b-cc01198853c4.png)

应用程序状态在版本之间可能会有结构性变化，这是你需要自行管理的。开源 Git 服务器 GitLab 的 Docker 镜像就是一个很好的例子。状态存储在卷上的数据库中，当你升级到新版本时，应用程序会检查数据库并在需要时运行升级脚本。

应用程序配置是利用卷挂载的另一种方式。你可以在镜像中嵌入默认的配置集合，但用户可以使用挂载来覆盖基本配置，使用他们自己的文件。

你将在下一章中看到这些技术被很好地运用。

# 将传统的 ASP.NET Web 应用程序打包为 Docker 镜像

微软已经在 MCR 上提供了 Windows Server Core 基础镜像，这是 Windows Server 2019 的一个版本，具有大部分完整服务器版的功能，但没有用户界面。就基础镜像而言，它非常庞大：在 Docker Hub 上压缩后为 2 GB，而 Nano Server 仅为 100 MB，微小的 Alpine Linux 镜像仅为 2 MB。但这意味着你几乎可以将任何现有的 Windows 应用程序 Docker 化，这是迁移系统到 Docker 的一个很好的方式。

还记得 NerdDinner 吗？它是一个开源的 ASP.NET MVC 展示应用程序，最初由微软的 Scott Hanselman 和 Scott Guthrie 等人编写。你仍然可以在 CodePlex 上获取到代码，但自 2013 年以来就没有进行过任何更改，因此它是证明旧的 .NET Framework 应用程序可以迁移到 Docker Windows 容器的理想候选项，这可以是现代化的第一步。

# 为 NerdDinner 编写 Dockerfile

我将遵循 NerdDinner 的多阶段构建方法，因此`dockeronwindows/ch-02-nerd-dinner:2e`镜像的 Dockerfile 从一个构建器阶段开始：

```

# escape=`
FROM microsoft/dotnet-framework:4.7.2-sdk-windowsservercore-ltsc2019 AS builder

WORKDIR C:\src\NerdDinner
COPY src\NerdDinner\packages.config .
RUN nuget restore packages.config -PackagesDirectory ..\packages

COPY src C:\src
RUN msbuild NerdDinner.csproj /p:OutputPath=c:\out /p:Configuration=Release 
```

该阶段使用`microsoft/dotnet-framework`作为编译应用程序的基础镜像。这是微软在 Docker Hub 上维护的一个镜像。它是构建在 Windows Server Core 镜像之上的，并且拥有编译 .NET Framework 应用程序所需的一切，包括 NuGet 和 MSBuild。构建阶段分为两个部分：

1.  将 NuGet `packages.config` 文件复制到镜像中，然后运行`nuget restore`。

1.  复制其余的源代码树并运行`msbuild`。

将这些部分分开意味着 Docker 将使用多个镜像层：第一层将包含所有已还原的 NuGet 包，第二层将包含已编译的 Web 应用程序。这意味着我可以利用 Docker 的层缓存。除非我更改了 NuGet 引用，否则包将从缓存层加载，Docker 将不会运行还原部分，这是一个昂贵的操作。只要任何源文件发生更改，MSBuild 步骤就会运行。 

如果我有一个 NerdDinner 的部署指南，在迁移到 Docker 之前，它会是这样的：

1.  在一个干净的服务器上安装 Windows。

1.  运行所有 Windows 更新。

1.  安装 IIS。

1.  安装 .NET。

1.  设置 ASP.NET。

1.  将 Web 应用程序复制到`C`驱动器中。

1.  在 IIS 中创建一个应用程序池。

1.  在 IIS 中创建网站并使用应用程序池。

1.  删除默认网站。

这将成为 Dockerfile 的第二阶段的基础，但我将能够简化所有步骤。我可以使用微软的 ASP.NET Docker 镜像作为`FROM`镜像，这将为我提供一个带有 IIS 和 ASP.NET 的干净安装的 Windows。这样一条指令就完成了前五个步骤。这是`dockeronwindows/ch-02-nerd-dinner:2e`的其余 Dockerfile 部分：

```

FROM mcr.microsoft.com/dotnet/framework/aspnet:4.7.2-windowsservercore-ltsc2019
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop']

ENV BING_MAPS_KEY bing_maps_key
WORKDIR C:\nerd-dinner

RUN Remove-Website -Name 'Default Web Site'; `
    New-Website -Name 'nerd-dinner' `
                -Port 80 -PhysicalPath 'c:\nerd-dinner' `
                -ApplicationPool '.NET v4.5'

RUN & c:\windows\system32\inetsrv\appcmd.exe `
      unlock config /section:system.webServer/handlers

COPY --from=builder C:\out\_PublishedWebsites\NerdDinner C:\nerd-dinner 
```

微软同时使用 Docker Hub 和 MCR 存储他们的 Docker 镜像。.NET Framework SDK 在 Docker Hub 上，但 ASP.NET 运行时镜像在 MCR 上。您可以通过在 Docker Hub 上检查来找到镜像的托管位置。

使用`escape`指令和`SHELL`指令让我可以在普通的 Windows 文件路径中使用，而无需双反斜杠，并使用 PowerShell 风格的反引号在多行中分隔命令。使用 PowerShell 在 IIS 中删除默认网站并创建新网站非常简单，Dockerfile 清楚地显示了应用程序使用的端口和内容的路径。

我正在使用内置的 .NET 4.5 应用程序池，这是与原始部署过程相比的简化。在虚拟机上的 IIS 中，通常会为每个网站拥有一个专用的应用程序池，以便将进程与其他进程隔离。但在容器化的应用程序中，只会运行一个网站。任何其他网站都将在其他容器中运行，因此我们已经进行了隔离，并且每个容器可以使用默认的应用程序池，而不必担心干扰。

最后的 `COPY` 指令将发布的 Web 应用程序从构建器阶段复制到应用程序镜像中。这是 Dockerfile 中利用 Docker 缓存的最后一行。当我在应用程序上工作时，源代码将是我最频繁更改的东西。Dockerfile 结构化得当，以便当我更改代码并运行 `docker image build` 时，只会运行第一阶段的 MSBuild 和第二阶段的复制，因此构建非常快速。

这可能是一个完全功能的 Docker 化的 ASP.NET 网站所需的一切，但在 NerdDinner 的情况下，还有一条额外的指令，证明了当您将应用程序容器化时，您可以处理棘手的，意想不到的细节。NerdDinner 应用程序在其 `Web.config` 文件的 `system.webServer` 部分中有一些自定义配置设置，并且默认情况下，该部分由 IIS 锁定。我需要解锁该部分，我在第二个 `RUN` 指令中使用 `appcmd` 完成。

现在我可以构建图像并在 Windows 容器中运行遗留的 ASP.NET 应用程序：

```

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

# 开发 Docker 化的.NET Framework 和.NET Core 应用程序

Docker 是一个用于打包、分发、运行和管理应用程序的平台。当您将应用程序打包为 Docker 镜像时，它们都具有相同的形状。您可以以相同的方式部署、管理、保护和升级它们。所有 Docker 化的应用程序在运行时都具有相同的要求：在兼容的操作系统上运行 Docker 引擎。应用程序在隔离的环境中运行，因此您可以在同一台机器上托管不同的应用程序平台和不同的平台版本而不会发生干扰。

在.NET 世界中，这意味着您可以在单个 Windows 机器上运行多个工作负载。它们可以是 ASP.NET 网站，也可以是作为.NET 控制台应用程序或.NET Windows 服务运行的**Windows Communication Foundation**（**WCF**）应用程序。在上一章中，我们讨论了如何在不进行任何代码更改的情况下将传统的.NET 应用程序 Docker 化，但是 Docker 对容器内运行的应用程序应该如何行为有一些简单的期望，以便它们可以充分利用该平台的全部优势。

在本章中，我们将探讨如何构建应用程序，以便它们可以充分利用 Docker 平台，包括：

+   Docker 与您的应用程序之间的集成点

+   使用配置文件和环境变量配置您的应用程序

+   使用健康检查监视应用程序

+   在不同容器中运行分布式解决方案的组件

这将帮助您开发符合 Docker 期望的.NET 和.NET Core 应用程序，以便您可以完全使用 Docker 进行管理。

我们将在本章中涵盖以下主题：

+   为 Docker 构建良好的应用程序

+   分离依赖项

+   拆分单片应用程序

# 为 Docker 构建良好的应用程序

Docker 平台对使用它的应用程序几乎没有要求。您不受限于特定的语言或框架，您不需要使用特殊的库来在应用程序和容器之间进行通信，也不需要以特定的方式构建您的应用程序。

为了支持尽可能广泛的应用程序范围，Docker 使用控制台在应用程序和容器运行时之间进行通信。应用程序日志和错误消息预期出现在控制台输出和错误流中。由 Docker 管理的存储被呈现为操作系统的普通磁盘，Docker 的网络堆栈是透明的。应用程序将看起来像是在自己的机器上运行，通过普通的 TCP/IP 网络连接到其他机器。

Docker 中的一个良好应用是一个对其运行的系统几乎没有假设，并且使用所有操作系统支持的基本机制：文件系统、环境变量、网络和控制台。最重要的是，应用程序应该只做一件事。正如你所看到的，当 Docker 运行一个容器时，它启动 Dockerfile 或命令行中指定的进程，并监视该进程。当进程结束时，容器退出。因此，理想情况下，你应该构建你的应用程序只有一个进程，这样可以确保 Docker 监视重要的进程。

这些只是建议，而不是要求。当容器启动时，你可以在引导脚本中启动多个进程，Docker 会愉快地运行它，但它只会监视最后启动的进程。你的应用程序可以将日志条目写入本地文件，而不是控制台，Docker 仍然会运行它们，但如果你使用 Docker 来检查容器日志，你将看不到任何输出。

在.NET 中，你可以通过运行控制台应用程序轻松满足建议，这提供了应用程序和主机之间的简化集成，这也是为什么所有.NET Core 应用程序（包括网站和 Web API）都作为控制台应用程序运行的一个原因。对于传统的.NET 应用程序，你可能无法使它们成为完美的应用程序，但你可以注意打包它们，以便它们充分利用 Docker 平台。

# 在 Docker 中托管 Internet 信息服务（IIS）应用程序

完整的.NET Framework 应用程序可以轻松打包成 Docker 镜像，但你需要注意一些限制。微软为 Docker 提供了 Nano Server 和 Windows Server Core 基础镜像。完整的.NET Framework 无法在 Nano Server 上运行，因此要在 Docker 中托管现有的.NET 应用程序，你需要使用 Windows Server Core 基础镜像。

从 Windows Server Core 运行意味着您的应用程序镜像大小约为 4 GB，其中大部分在基础镜像中。您拥有完整的 Windows Server 操作系统，所有软件包都可用于启用 Windows Server 功能，如域名系统（DNS）和动态主机配置协议（DHCP），即使您只想将其用于单个应用程序角色。从 Windows Server Core 运行容器是完全合理的，但您需要了解其影响：

+   基础镜像具有大量安装的软件，这意味着它可能会有更频繁的安全和功能补丁。

+   操作系统除了您的应用程序进程外，还运行了许多自己的进程，因为 Windows 的几个核心部分作为后台 Windows 服务运行。

+   Windows 拥有自己的应用程序平台，具有高价值的特性集，用于托管和管理，这些特性与 Docker 方法不会自然集成。

您可以将 ASP.NET Web 应用程序 Docker 化几个小时。它将构建为一个大型 Docker 镜像，比基于轻量级现代应用程序堆栈构建的应用程序需要更长的时间来分发和启动。但您仍将拥有一个部署、配置和准备运行的整个应用程序的单一软件包。这是提高质量和减少部署时间的重要一步，也可以是现代化传统应用程序计划的第一部分。

将 ASP.NET 应用程序与 Docker 更紧密地集成，可以修改 IIS 日志的编写方式，指定 Docker 如何检查容器是否健康，并向容器注入配置，而无需对应用程序代码进行任何更改。如果更改代码是现代化计划的一部分，那么只需进行最小的更改，就可以使用容器的环境变量和文件系统进行应用程序配置。

# 为 Docker 友好的日志记录配置 IIS

IIS 将日志条目写入文本文件，记录 HTTP 请求和响应。您可以精确配置要写入的字段，但默认安装记录了诸如 HTTP 请求的路由、响应状态代码和 IIS 响应所需的时间等有用信息。将这些日志条目呈现给 Docker 是很好的，但 IIS 管理自己的日志文件，将条目缓冲到磁盘之前，并旋转日志文件以管理磁盘空间。

日志管理是应用程序平台的基本组成部分，这就是为什么 IIS 为 Web 应用程序负责，但 Docker 有自己的日志记录系统。Docker 日志记录比 IIS 使用的文本文件系统更强大和可插拔，但它只从容器的控制台输出流中读取日志条目。您不能让 IIS 将日志写入控制台，因为它在后台作为 Windows 服务运行，没有连接到控制台，所以您需要另一种方法。

有两种选择。第一种是构建一个 HTTP 模块，它插入到 IIS 平台中，具有一个事件处理程序，从 IIS 接收日志。此处理程序可以将所有消息发布到队列或 Windows 管道，因此您不会改变 IIS 日志的方式；您只是添加了另一个日志接收端。然后，您会将您的 Web 应用程序与一个监听已发布的日志条目并将其中继到控制台的控制台应用程序打包在一起。控制台应用程序将是容器启动时的入口点，因此每个 IIS 日志条目都会被路由到控制台供 Docker 读取。

HTTP 模块方法是强大且可扩展的，但在我们刚开始时，它增加了比我们需要的更多复杂性。第二个选项更简单 - 配置 IIS 将所有日志条目写入单个文本文件，并在容器的启动命令中运行一个 PowerShell 脚本来监视该文件，并将新的日志条目回显到控制台。当容器运行时，所有 IIS 日志条目都会回显到控制台，从而将它们呈现给 Docker。

在 Docker 镜像中设置这一点，首先需要配置 IIS，使其将任何站点的所有日志条目写入单个文件，并允许文件增长而不进行旋转。您可以在 Dockerfile 中使用 PowerShell 来完成这一点，使用`Set-WebConfigurationProperty` cmdlet 来修改应用程序主机级别的中央日志属性。我在`dockeronwindows/ch03-iis-log-watcher`镜像的 Dockerfile 中使用了这个 cmdlet：

```
RUN Set-WebConfigurationProperty -p 'MACHINE/WEBROOT/APPHOST' -fi 'system.applicationHost/log' -n 'centralLogFileMode' -v 'CentralW3C'; `
    Set-WebConfigurationProperty -p 'MACHINE/WEBROOT/APPHOST' -fi 'system.applicationHost/log/centralW3CLogFile' -n 'truncateSize' -v 4294967295; `
    Set-WebConfigurationProperty -p 'MACHINE/WEBROOT/APPHOST' -fi 'system.applicationHost/log/centralW3CLogFile' -n 'period' -v 'MaxSize'; `
    Set-WebConfigurationProperty -p 'MACHINE/WEBROOT/APPHOST' -fi 'system.applicationHost/log/centralW3CLogFile' -n 'directory' -v 'C:\iislog'
```

这是丑陋的代码，但它表明你可以在 Dockerfile 中编写任何你需要设置应用程序的内容。它配置 IIS 将所有条目记录到`C:\iislog`中的文件，并设置日志轮换的最大文件大小，让日志文件增长到 4GB。这足够的空间来使用 - 记住，容器不应该长时间存在，所以我们不应该在单个容器中有几 GB 的日志条目。IIS 仍然使用子目录格式来记录日志文件，所以实际的日志文件路径将是`C:\iislog\W3SVC\u_extend1.log`。现在我有了一个已知的日志文件位置，我可以使用 PowerShell 来回显日志条目到控制台。

我在`CMD`指令中执行这个操作，所以 Docker 运行和监控的最终命令是 PowerShell 的 cmdlet 来回显日志条目。当新条目被写入控制台时，它们会被 Docker 捕捉到。PowerShell 可以很容易地监视文件，但是有一个复杂的地方，因为文件需要在 PowerShell 监视之前存在。在 Dockerfile 中，我在启动时按顺序运行多个命令：

```
 CMD Start-Service W3SVC; `
     Invoke-WebRequest http://localhost -UseBasicParsing | Out-Null; `
     netsh http flush logbuffer | Out-Null; `
     Get-Content -path 'c:\iislog\W3SVC\u_extend1.log' -Tail 1 -Wait
```

容器启动时会发生四件事情：

1.  启动 IIS Windows 服务（W3SVC）。

1.  发出 HTTP `GET`请求到本地主机，启动 IIS 工作进程并写入第一个日志条目。

1.  刷新 HTTP 日志缓冲区，这样日志文件就会被写入磁盘并存在于 PowerShell 监视之中。

1.  以尾部模式读取日志文件的内容，这样文件中写入的任何新行都会显示在控制台上。

我可以以通常的方式从这个镜像中运行一个容器：

```
 docker container run -d -P --name log-watcher dockeronwindows/ch03-iis-log-watcher:2e
```

当我通过浏览到容器的 IP 地址（或在 PowerShell 中使用`Invoke-WebRequest`）发送一些流量到站点时，我可以看到从`Get-Content` cmdlet 使用`docker container logs`中中继到 Docker 的 IIS 日志条目：

```
> docker container logs log-watcher
2019-02-06 20:21:30 W3SVC1 172.27.97.43 GET / - 80 - 192.168.2.214 Mozilla/5.0+(Windows+NT+10.0;+Win64;+x64;+rv:64.0)+Gecko/20100101+Firefox/64.0 - 200 0 0 7
2019-02-06 20:21:30 W3SVC1 172.27.97.43 GET /iisstart.png - 80 - 192.168.2.214 Mozilla/5.0+(Windows+NT+10.0;+Win64;+x64;+rv:64.0)+Gecko/20100101+Firefox/64.0 http://localhost:51959/ 200 0 0 17
2019-02-06 20:21:30 W3SVC1 172.27.97.43 GET /favicon.ico - 80 - 192.168.2.214 Mozilla/5.0+(Windows+NT+10.0;+Win64;+x64;+rv:64.0)+Gecko/20100101+Firefox/64.0 - 404 0 2 23
```

IIS 始终在将日志条目写入磁盘之前在内存中缓冲日志条目，以提高性能进行微批量写入。刷新每 60 秒进行一次，或者当缓冲区大小为 64KB 时。如果你想强制容器中的 IIS 日志刷新，可以使用与我在 Dockerfile 中使用的相同的`netsh`命令：`docker container exec log-watcher netsh http flush logbuffer`。你会看到一个`Ok`输出，并且新的条目将在`docker container logs`中。

我已将配置添加到映像中的 IIS 和一个新命令，这意味着所有 IIS 日志条目都会被回显到控制台。这将适用于托管在 IIS 中的任何应用程序，因此我可以在不更改应用程序或站点内容的情况下回显 ASP.NET 应用程序和静态网站的 HTTP 日志。控制台输出是 Docker 查找日志条目的地方，因此这个简单的扩展将现有应用程序的日志集成到新平台中。

# 管理应用程序配置

在 Docker 映像中打包应用程序的目标是在每个环境中使用相同的映像。您不会为测试和生产构建单独的映像，因为这将使它们成为单独的应用程序，并且可能存在不一致性。您应该从用户测试的完全相同的 Docker 映像部署生产应用程序，这是生成过程生成的完全相同的映像，并用于所有自动集成测试的映像。

当然，一些东西需要在环境之间进行更改 - 数据库的连接字符串，日志级别和功能开关。这是应用程序配置，在 Docker 世界中，您使用默认配置构建应用程序映像，通常用于开发环境。在运行时，您将当前环境的正确配置注入到容器中，并覆盖默认配置。

有不同的方法来注入此配置。在本章中，我将向您展示如何使用卷挂载和环境变量。在生产中，您将运行运行 Docker 的机器集群，并且可以将配置数据存储在集群的安全数据库中，作为 Docker 配置对象或 Docker 秘密。我将在第七章中介绍这一点，*使用 Docker Swarm 编排分布式解决方案*。

# 在 Docker 卷中挂载配置文件

传统的应用程序平台使用配置文件在环境之间更改行为。 .NET Framework 应用程序具有丰富的基于 XML 的配置框架，而 Java 应用程序通常在属性文件中使用键值对。您可以在 Dockerfile 中向应用程序映像添加这些配置文件，并且当您从映像运行容器时，它将使用此默认配置。

您的应用程序设置应该使用一个特定的目录来存储配置文件，这样可以通过挂载 Docker 卷在运行时覆盖它们。我已经在`dockeronwindows/ch03-aspnet-config:2e`中使用了一个简单的 ASP.NET WebForms 应用程序。Dockerfile 只使用了您已经看到的命令：

```
# escape=` FROM mcr.microsoft.com/dotnet/framework/aspnet COPY Web.config C:\inetpub\wwwroot COPY config\*.config C:\inetpub\wwwroot\config\ COPY default.aspx C:\inetpub\wwwroot
```

这使用了微软的 ASP.NET 镜像作为基础，并复制了我的应用程序文件 - 一个 ASPX 页面和一些配置文件。在这个例子中，我正在使用默认的 IIS 网站，它从`C:\inetpub\wwwroot`加载内容，所以我只需要在 Dockerfile 中使用`COPY`指令，而不需要运行任何 PowerShell 脚本。

ASP.NET 期望在应用程序目录中找到`Web.config`文件，但您可以将配置的部分拆分成单独的文件。我已经在一个子目录中的文件中做到了这一点，这些文件是从`appSettings`和`connectionStrings`部分加载的：

```
<?xml version="1.0" encoding="utf-8"?> <configuration>
  <appSettings  configSource="config\appSettings.config"  />
  <connectionStrings  configSource="config\connectionStrings.config"  /> </configuration>
```

`config`目录填充了默认配置文件，所以我可以从镜像中运行容器，而不需要指定任何额外的设置：

```
docker container run -d -P dockeronwindows/ch03-aspnet-config:2e
```

当我获取容器的端口并浏览到它时，我看到网页显示来自默认配置文件的值：

![](img/c2ba482d-3bc5-451f-af92-3f10af0aebb4.png)

我可以通过从主机上的目录加载配置文件，将本地目录挂载为一个卷，以`C:\inetpub\wwwroot\config`为目标，来为不同的环境运行应用程序。当容器运行时，该目录的内容将从主机上的目录加载：

```
docker container run -d -P `
 -v $pwd\prod-config:C:\inetpub\wwwroot\config `
 dockeronwindows/ch03-aspnet-config:2e
```

我正在使用 PowerShell 来运行这个命令，它会将`$pwd`扩展到当前目录的完整值，所以我在说当前路径中的`prod-config`目录应该被挂载为容器中的`C:\inetpub\wwwroot\config`。您也可以使用完全限定的路径。

当我浏览到这个容器的端口时，我看到不同的配置值显示出来：

![](img/a42dd810-c2c0-4f38-9f3b-455b9164f98d.png)

这里重要的是，我在每个环境中使用完全相同的 Docker 镜像，具有相同的设置和相同的二进制文件。只有配置文件会改变，Docker 提供了一种优雅的方法来做到这一点。

# 推广环境变量

现代应用程序越来越多地使用环境变量作为配置设置，因为它们几乎被每个平台支持，从物理机器到 PaaS，再到无服务器函数。所有平台都以相同的方式使用环境变量 - 作为键值对的存储，因此通过使用环境变量进行配置，可以使您的应用程序具有高度的可移植性。

ASP.NET 应用程序已经在`Web.config`中具有丰富的配置框架，但通过一些小的代码更改，您可以将关键设置移动到环境变量中。这样，您可以为应用程序构建一个 Docker 镜像，在不同的平台上运行，并在容器中设置环境变量以更改配置。

Docker 允许您在 Dockerfile 中指定环境变量并给出初始默认值。`ENV`指令设置环境变量，您可以在每个`ENV`指令中设置一个或多个变量。以下示例来自于`dockeronwindows/ch03-iis-environment-variables:2e`的 Dockerfile。

```
 ENV A01_KEY A01 value
 ENV A02_KEY="A02 value" `
     A03_KEY="A03 value"
```

使用`ENV`在 Dockerfile 中添加的设置将成为镜像的一部分，因此您从该镜像运行的每个容器都将具有这些值。运行容器时，您可以使用`--env`或`-e`选项添加新的环境变量或替换现有镜像变量的值。您可以通过一个简单的 Nano Server 容器看到环境变量是如何工作的。

```
> docker container run `
  --env ENV_01='Hello' --env ENV_02='World' `
  mcr.microsoft.com/windows/nanoserver:1809 `
  cmd /s /c echo %ENV_01% %ENV_02%

Hello World
```

在 IIS 中托管的应用程序使用 Docker 中的环境变量存在一个复杂性。当 IIS 启动时，它会从系统中读取所有环境变量并对其进行缓存。当 Docker 运行具有设置的环境变量的容器时，它会将它们写入进程级别，但这发生在 IIS 缓存了原始值之后，因此它们不会被更新，IIS 应用程序将无法看到新值。然而，IIS 并不以相同的方式缓存机器级别的环境变量，因此我们可以将 Docker 设置的值提升为机器级别的环境变量，这样 IIS 应用程序就能够读取它们。

推广环境变量可以通过将它们从进程级别复制到机器级别来实现。您可以在容器启动命令中使用 PowerShell 脚本，通过循环遍历所有进程级别变量并将它们复制到机器级别，除非机器级别键已经存在。

```
 foreach($key in [System.Environment]::GetEnvironmentVariables('Process').Keys) {
     if ([System.Environment]::GetEnvironmentVariable($key, 'Machine') -eq $null) {
         $value = [System.Environment]::GetEnvironmentVariable($key, 'Process')
         [System.Environment]::SetEnvironmentVariable($key, $value, 'Machine')
     }
 }
```

如果您使用的是基于 Microsoft 的 IIS 镜像的图像，则无需执行此操作，因为它会为您使用一个名为`ServiceMonitor.exe`的实用程序，该实用程序已打包在 IIS 镜像中。ServiceMonitor 执行三件事——它使进程级环境变量可用，启动后台 Windows 服务，然后监视服务以确保其保持运行。这意味着您可以使用 ServiceMonitor 作为容器的启动进程，如果 IIS Windows 服务失败，ServiceMonitor 将退出，Docker 将看到您的应用程序已停止。

`ServiceMonitor.exe`可以在 GitHub 上作为二进制文件使用，但它不是开源的，并且并非所有行为都有文档记录（它似乎只适用于默认的 IIS 应用程序池）。它被复制到 Microsoft 的 IIS 镜像中，并设置为容器的`ENTRYPOINT`。ASP.NET 镜像是基于 IIS 镜像构建的，因此它也配置了 ServiceMonitor。

如果您想要在自己的逻辑中使用 ServiceMonitor 来回显 IIS 日志，您需要在后台启动 ServiceMonitor，并在 Dockerfile 中的启动命令中完成日志读取。我在`dockeronwindows/ch03-iis-environment-variables:2e`中使用 PowerShell 的`Start-Process`命令运行 ServiceMonitor：

```
ENTRYPOINT ["powershell"] CMD Start-Process -NoNewWindow -FilePath C:\ServiceMonitor.exe -ArgumentList w3svc; ` Invoke-WebRequest http://localhost -UseBasicParsing | Out-Null; `
    netsh http flush logbuffer | Out-Null; `
   Get-Content -path 'C:\iislog\W3SVC\u_extend1.log' -Tail 1 -Wait 
```

`ENTRYPOINT`和`CMD`指令都告诉 Docker 如何运行您的应用程序。您可以将它们组合在一起，以指定默认的入口点，并允许您的镜像用户在启动容器时覆盖命令。

图像中的应用程序是一个简单的 ASP.NET Web Forms 页面，列出了环境变量。我可以以通常的方式在容器中运行这个应用程序：

```
docker container run -d -P --name iis-env dockeronwindows/ch03-iis-environment-variables:2e
```

![](img/2e618d29-6ac9-4bbd-9146-05ec35667a31.png)

```
$port = $(docker container port iis-env).Split(':')[1]
start "http://localhost:$port"
```

网站显示了来自 Docker 镜像的默认环境变量值，这些值被列为进程级变量：

当容器启动时，我可以获取容器的端口，并在 ASP.NET Web Forms 页面上打开浏览器，使用一些简单的 PowerShell 脚本：

您可以使用不同的环境变量运行相同的镜像，覆盖其中一个镜像变量并添加一个新变量：

```
docker container run -d -P --name iis-env2 ` 
 -e A01_KEY='NEW VALUE!' ` 
 -e B01_KEY='NEW KEY!' `
 dockeronwindows/ch03-iis-environment-variables:2e
```

浏览新容器的端口，您将看到 ASP.NET 页面写出的新值：

![](img/ef181092-bc3c-4348-8c38-689692850087.png)

我现在已经将对 Docker 环境变量管理的支持添加到了 IIS 镜像中，因此 ASP.NET 应用程序可以使用`System.Environment`类来读取配置设置。我在这个新镜像中保留了 IIS 日志回显，因此这是一个良好的 Docker 公民，现在您可以通过 Docker 配置应用程序并检查日志。

我可以做的最后一个改进是告诉 Docker 如何监视容器内运行的应用程序，以便 Docker 可以确定应用程序是否健康，并在其变得不健康时采取行动。

# 构建监视应用程序的 Docker 镜像

当我将这些新功能添加到 NerdDinner Dockerfile 并从镜像运行容器时，我将能够使用`docker container logs`命令查看 Web 请求和响应日志，该命令中继了 Docker 捕获的所有 IIS 日志条目，并且我可以使用环境变量和配置文件来指定 API 密钥和数据库用户凭据。这使得运行和管理传统的 ASP.NET 应用程序与我在 Docker 上运行的任何其他容器化应用程序的方式一致。我还可以配置 Docker 来监视容器，以便我可以管理任何意外故障。

Docker 提供了监视应用程序健康状况的能力，而不仅仅是检查应用程序进程是否仍在运行，使用 Dockerfile 中的`HEALTHCHECK`指令。使用`HEALTHCHECK`告诉 Docker 如何测试应用程序是否仍然健康。语法类似于`RUN`和`CMD`指令。您传递一个要执行的 shell 命令，如果应用程序健康，则应该返回`0`，如果不健康，则返回`1`。Docker 在容器运行时定期运行健康检查，并在容器的健康状况发生变化时发出状态事件。

Web 应用程序的*健康*的简单定义是能够正常响应 HTTP 请求。您进行的请求取决于您希望检查的彻底程度。理想情况下，请求应该执行应用程序的关键部分，以便您确信它全部正常工作。但同样，请求应该快速完成并且对计算影响最小，因此处理大量的健康检查不会影响消费者请求。

对于任何 Web 应用程序的简单健康检查只需使用`Invoke-WebRequest` PowerShell 命令来获取主页并检查 HTTP 响应代码是否为`200`，这意味着成功接收到响应：

```
try { 
    $response = iwr http://localhost/ -UseBasicParsing
    if ($response.StatusCode -eq 200) { 
        return 0
    } else {
        return 1
    } 
catch { return 1 }
```

对于更复杂的 Web 应用程序，添加一个专门用于健康检查的新端点可能很有用。您可以向 API 和网站添加一个诊断端点，该端点执行应用程序的一些核心逻辑并返回一个布尔结果，指示应用程序是否健康。您可以在 Docker 健康检查中调用此端点，并检查响应内容以及状态码，以便更有信心地确认应用程序是否正常工作。

Dockerfile 中的`HEALTHCHECK`指令非常简单。您可以配置检查之间的间隔和容器被视为不健康之前可以失败的检查次数，但是要使用默认值，只需在`HEALTHCHECK CMD`中指定测试脚本。以下是来自`dockeronwindows/ch03-iis-healthcheck:2e`镜像的 Dockerfile 的示例，它使用 PowerShell 向诊断 URL 发出`GET`请求并检查响应状态码：

```
HEALTHCHECK --interval=5s `
 CMD powershell -command `
    try { `
     $response = iwr http://localhost/diagnostics -UseBasicParsing; `
     if ($response.StatusCode -eq 200) { return 0} `
     else {return 1}; `
    } catch { return 1 }
```

我已经为健康检查指定了一个间隔，因此 Docker 将每 5 秒在容器内执行此命令（如果您不指定间隔，则默认间隔为 30 秒）。健康检查非常便宜，因为它是本地容器的，所以您可以设置这样的短间隔，并快速捕捉任何问题。

此 Docker 镜像中的应用程序是一个 ASP.NET Web API 应用程序，其中有一个诊断端点和一个控制器，您可以使用该控制器来切换应用程序的健康状态。Dockerfile 包含一个健康检查，当您从该镜像运行容器时，您可以看到 Docker 如何使用它：

```
docker container run -d -P --name healthcheck dockeronwindows/ch03-iis-healthcheck:2e
```

如果您在启动该容器后运行`docker container ls`，您会看到状态字段中稍有不同的输出，类似于`Up 3 seconds (health: starting)`。Docker 每 5 秒运行一次此容器的健康检查，所以在这一点上，检查尚未运行。稍等一会儿，然后状态将变为类似于`Up 46 seconds (healthy)`。

您可以通过查询“诊断”端点来检查 API 的当前健康状况：

```
$port = $(docker container port healthcheck).Split(':')[1]
iwr "http://localhost:$port/diagnostics"
```

在返回的内容中，您会看到`"Status":"GREEN"`，这意味着 API 是健康的。直到我调用控制器来切换健康状态之前，这个容器将保持健康。我可以通过一个`POST`请求来做到这一点，该请求将 API 设置为对所有后续请求返回 HTTP 状态`500`：

```
iwr "http://localhost:$port/toggle/unhealthy" -Method Post
```

现在，应用程序将对 Docker 平台发出的所有`GET`请求响应 500，这将导致健康检查失败。Docker 会继续尝试健康检查，如果连续三次失败，则认为容器不健康。此时，容器列表中的状态字段显示`Up 3 minutes (unhealthy)`。Docker 不会对不健康的单个容器采取自动操作，因此此容器仍在运行，您仍然可以访问 API。

在集群化的 Docker 环境中运行容器时，健康检查非常重要（我在第七章中介绍了*使用 Docker Swarm 编排分布式解决方案*），并且在所有 Dockerfile 中包含它们是一个良好的实践。能够打包一个平台可以测试健康状况的应用程序是一个非常有用的功能 - 这意味着无论在哪里运行应用程序，Docker 都可以对其进行检查。

现在，您拥有了所有工具，可以将 ASP.NET 应用程序容器化，并使其成为 Docker 的良好组成部分，与平台集成，以便可以像其他容器一样进行监视和管理。在 Windows Server Core 上运行的完整.NET Framework 应用程序无法满足运行单个进程的期望，因为所有必要的后台 Windows 服务，但您仍应构建容器映像，以便它们仅运行一个逻辑功能并分离任何依赖项。

# 分离依赖项

在上一章中，我将传统的 NerdDinner 应用程序 Docker 化并使其运行起来，但没有数据库。原始应用程序期望在与应用程序运行的同一主机上使用 SQL Server LocalDB。LocalDB 是基于 MSI 的安装，我可以通过下载 MSI 并在 Dockerfile 中使用`RUN`命令安装它来将其添加到 Docker 镜像中。但这意味着当我从镜像启动容器时，它具有两个功能：托管 Web 应用程序和运行数据库。

在一个容器中具有两个功能并不是一个好主意。如果您想要升级网站而不更改数据库会发生什么？或者如果您需要对数据库进行一些维护，而这不会影响网站会发生什么？如果您需要扩展网站呢？通过将这两个功能耦合在一起，您增加了部署风险、测试工作量和管理复杂性，并减少了操作灵活性。

相反，我将把数据库打包到一个新的 Docker 镜像中，在一个单独的容器中运行它，并使用 Docker 的网络层从网站容器访问数据库容器。SQL Server 是一个有许可的产品，但免费的变体是 SQL Server Express，它可以从 Docker Hub 上的微软镜像中获得，并带有生产许可证。我可以将其用作我的镜像的基础，构建它以准备一个预配置的数据库实例，其中架构已部署并准备连接到 Web 应用程序。

# 为 SQL Server 数据库创建 Docker 镜像

设置数据库镜像就像设置任何其他 Docker 镜像一样。我将把设置任务封装在一个 Dockerfile 中。总的来说，对于一个新的数据库，步骤将是：

1.  安装 SQL Server

1.  配置 SQL Server

1.  运行 DDL 脚本来创建数据库架构

1.  运行 DML 脚本来填充静态数据

这非常适合使用 Visual Studio 的 SQL 数据库项目类型和 Dacpac 部署模型的典型构建过程。从发布项目的输出是一个包含数据库架构和任何自定义 SQL 脚本的`.dacpac`文件。使用`SqlPackage`工具，您可以将 Dacpac 文件部署到 SQL Server 实例，它将创建一个新的数据库（如果不存在），或者升级现有的数据库，使架构与 Dacpac 匹配。

这种方法非常适合自定义 SQL Server Docker 镜像。我可以再次使用多阶段构建来为 Dockerfile 构建，这样其他用户就不需要安装 Visual Studio 来从源代码打包数据库。这是`dockeronwindows/ch03-nerd-dinner-db:2e`镜像的 Dockerfile 的第一阶段：

```
# escape=` FROM microsoft/dotnet-framework:4.7.2-sdk-windowsservercore-ltsc2019 AS builder SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop';"] # add SSDT build tools RUN nuget install Microsoft.Data.Tools.Msbuild -Version 10.0.61804.210 # add SqlPackage tool ENV download_url="https://download.microsoft.com/download/6/E/4/6E406.../EN/x64/DacFramework.msi" RUN Invoke-WebRequest -Uri $env:download_url -OutFile DacFramework.msi ; `Start-Process msiexec.exe -ArgumentList '/i', 'DacFramework.msi', '/quiet', '/norestart' -NoNewWindow -Wait; `Remove-Item -Force DacFramework.msi WORKDIR C:\src\NerdDinner.Database COPY src\NerdDinner.Database . RUN msbuild NerdDinner.Database.sqlproj ` /p:SQLDBExtensionsRefPath="C:\Microsoft.Data.Tools.Msbuild.10.0.61804.210\lib\net46" ` /p:SqlServerRedistPath="C:\Microsoft.Data.Tools.Msbuild.10.0.61804.210\lib\net46" 
```

这里有很多内容，但都很简单。`builder`阶段从微软的.NET Framework SDK 镜像开始。这给了我`NuGet`和`MSBuild`，但没有我构建 SQL Server Dacpac 所需的依赖项。前两个`RUN`指令安装了 SQL Server 数据工具和`SqlPackage`工具。如果我有很多数据库项目要容器化，我可以将其打包为一个单独的 SQL Server SDK 镜像。

阶段的其余部分只是复制 SQL 项目源代码并运行`MSBuild`来生成 Dacpac。

这是 Dockerfile 的第二阶段，它打包了 NerdDinner Dacpac 以在 SQL Server Express 中运行：

```
FROM dockeronwindows/ch03-sql-server:2e ENV DATA_PATH="C:\data" ` sa_password="N3rdD!Nne720⁶" VOLUME ${DATA_PATH} WORKDIR C:\init COPY Initialize-Database.ps1 . CMD powershell ./Initialize-Database.ps1 -sa_password $env:sa_password -data_path $env:data_path -Verbose COPY --from=builder ["C:\\Program Files...\\DAC", "C:\\Program Files...\\DAC"] COPY --from=builder C:\docker\NerdDinner.Database.dacpac . 
```

我正在使用我自己的 Docker 镜像，其中安装了 SQL Server Express 2017。微软在 Docker Hub 上发布了用于 Windows 和 Linux 的 SQL Server 镜像，但 Windows 版本并没有定期维护。SQL Server Express 是免费分发的，所以你可以将其打包到自己的 Docker 镜像中（`dockeronwindows/ch03-sql-server`的 Dockerfile 在 GitHub 的`sixeyed/docker-on-windows`存储库中）。

除了您迄今为止看到的内容之外，这里没有新的说明。为 SQL Server 数据文件设置了一个卷，并设置了一个环境变量来将默认数据文件路径设置为`C:\data`。您会看到没有`RUN`命令，所以当我构建镜像时，我实际上并没有设置数据库架构；我只是将 Dacpac 文件打包到镜像中，这样我就有了创建或升级数据库所需的一切。

在`CMD`指令中，我运行一个设置数据库的 PowerShell 脚本。有时将所有启动细节隐藏在一个单独的脚本中并不是一个好主意，因为这意味着仅凭 Dockerfile 就无法看到容器运行时会发生什么。但在这种情况下，启动过程有很多功能，如果我们把它们都放在那里，Dockerfile 会变得非常庞大。

基本的 SQL Server Express 镜像定义了一个名为`sa_password`的环境变量来设置管理员密码。我扩展了这个镜像并为该变量设置了默认值。我将以相同的方式使用该变量，以便允许用户在运行容器时指定管理员密码。启动脚本的其余部分处理了在 Docker 卷中存储数据库状态的问题。

# 管理 SQL Server 容器的数据库文件

数据库容器与任何其他 Docker 容器一样，但侧重于状态。您需要确保数据库文件存储在容器之外，这样您就可以替换数据库容器而不会丢失任何数据。您可以像我们在上一章中看到的那样轻松地使用卷来实现这一点，但有一个问题。

如果您构建了一个带有部署的数据库架构的自定义 SQL Server 镜像，那么您的数据库文件将位于已知位置的镜像中。您可以从该镜像运行一个容器，而无需挂载卷，它将正常工作，但数据将存储在容器的可写层中。如果您在需要执行数据库升级时替换容器，那么您将丢失所有数据。

相反，您可以使用从主机挂载的卷来运行容器，将预期的 SQL Server 数据目录从主机目录映射到一个已知位置的主机上，这样，您的文件就可以存放在容器之外的主机上。这样，您可以确保您的数据文件存储在可靠的地方，比如在服务器上的 RAID 阵列中。但这意味着您不能在 Dockerfile 中部署数据库，因为数据目录将在镜像中存储数据文件，如果您在目录上挂载卷，这些文件将被隐藏。

微软的 SQL Server 镜像通过在运行时附加数据库和日志文件来处理这个问题，因此它的工作原理是您已经在主机上拥有数据库文件。在这种情况下，您可以直接使用该镜像，挂载您的数据文件夹，并使用参数运行 SQL Server 容器，告诉它要附加哪个数据库。这是一个非常有限的方法 - 这意味着您需要首先在不同的 SQL Server 实例上创建数据库，然后在运行容器时附加它。这与自动化发布流程不符。

对于我的自定义镜像，我想做一些不同的事情。镜像包含了 Dacpac，因此它具有部署数据库所需的一切。当容器启动时，我希望它检查数据目录，如果它是空的，那么我通过部署 Dacpac 模型来创建一个新的数据库。如果在容器启动时数据库文件已经存在，则首先附加数据库文件，然后使用 Dacpac 模型升级数据库。

这种方法意味着您可以使用相同的镜像在新环境中运行一个新的数据库容器，或者升级现有的数据库容器而不丢失任何数据。无论您是否从主机挂载数据库目录，这都能很好地工作，因此您可以让用户选择如何管理容器存储，因此该镜像支持许多不同的场景。

执行此操作的逻辑都在`Initialize-Database.ps1` PowerShell 脚本中，Dockerfile 将其设置为容器的入口点。在 Dockerfile 中，我将数据目录传递给 PowerShell 脚本中的`data_path`变量，并且脚本检查该目录中是否存在 NerdDinner 数据（`mdf`）和日志（`ldf`）文件：

```
$mdfPath  =  "$data_path\NerdDinner_Primary.mdf" $ldfPath  =  "$data_path\NerdDinner_Primary.ldf" # attach data files if they exist: if  ((Test-Path  $mdfPath)  -eq  $true) {  $sqlcmd  =  "IF DB_ID('NerdDinner') IS NULL BEGIN CREATE DATABASE NerdDinner ON (FILENAME = N'$mdfPath')"    if  ((Test-Path  $ldfPath)  -eq  $true) {   $sqlcmd  =  "$sqlcmd, (FILENAME = N'$ldfPath')"
 }  $sqlcmd  =  "$sqlcmd FOR ATTACH; END"  Invoke-Sqlcmd  -Query $sqlcmd  -ServerInstance ".\SQLEXPRESS" }
```

这个脚本看起来很复杂，但实际上，它只是构建了一个`CREATE DATABASE...FOR ATTACH`语句，如果 MDF 数据文件和 LDF 日志文件存在，则填写路径。然后它调用 SQL 语句，将外部卷中的数据库文件作为 SQL Server 容器中的新数据库附加。

这涵盖了用户使用卷挂载运行容器的情况，主机目录已经包含来自先前容器的数据文件。这些文件被附加，数据库在新容器中可用。接下来，脚本使用`SqlPackage`工具从 Dacpac 生成部署脚本。我知道`SqlPackage`工具存在，也知道它的路径，因为它是从构建阶段打包到我的镜像中的：

```
$SqlPackagePath  =  'C:\Program Files\Microsoft SQL Server\140\DAC\bin\SqlPackage.exe' &  $SqlPackagePath  `
  /sf:NerdDinner.Database.dacpac `
  /a:Script /op:deploy.sql /p:CommentOutSetVarDeclarations=true `
  /tsn:.\SQLEXPRESS /tdn:NerdDinner /tu:sa /tp:$sa_password  
```

如果容器启动时数据库目录为空，则容器中没有`NerdDinner`数据库，并且`SqlPackage`将生成一个包含一组`CREATE`语句的脚本来部署新数据库。如果数据库目录包含文件，则现有数据库将被附加。在这种情况下，`SqlPackage`将生成一个包含一组`ALTER`和`CREATE`语句的脚本，以使数据库与 Dacpac 保持一致。

在这一步生成的`deploy.sql`脚本将创建新模式，或者对旧模式进行更改以升级它。最终数据库模式在两种情况下都将是相同的。

最后，PowerShell 脚本执行 SQL 脚本，传入数据库名称、文件前缀和数据路径的变量：

```
$SqlCmdVars  =  "DatabaseName=NerdDinner",  "DefaultFilePrefix=NerdDinner"...  Invoke-Sqlcmd  -InputFile deploy.sql -Variable $SqlCmdVars  -Verbose
```

SQL 脚本运行后，数据库在容器中存在，并且其模式与 Dacpac 中建模的模式相同，Dacpac 是从 Dockerfile 的构建阶段中的 SQL 项目构建的。数据库文件位于预期位置，并具有预期名称，因此如果用相同镜像的另一个容器替换此容器，新容器将找到现有数据库并附加它。

# 在容器中运行数据库

现在我有一个数据库镜像，可以用于新部署和升级。开发人员可以使用该镜像，在他们开发功能时运行它而不挂载卷，这样他们每次运行容器时都可以从一个新的数据库开始。同样的镜像也可以在需要保留现有数据库的环境中使用，通过使用包含数据库文件的卷来运行容器。

这就是您在 Docker 中运行 NerdDinner 数据库的方式，使用默认管理员密码，带有数据库文件的主机目录，并命名容器，以便我可以从其他容器中访问它：

```
mkdir -p C:\databases\nd

docker container run -d -p 1433:1433 ` --name nerd-dinner-db ` -v C:\databases\nd:C:\data ` dockeronwindows/ch03-nerd-dinner-db:2e
```

第一次运行此容器时，Dacpac 将运行以创建数据库，并将数据和日志文件保存在主机上的挂载目录中。您可以使用`ls`检查主机上是否存在文件，并且`docker container logs`的输出显示生成的 SQL 脚本正在运行，并创建资源：

```
> docker container logs nerd-dinner-db
VERBOSE: Starting SQL Server
VERBOSE: Changing SA login credentials
VERBOSE: No data files - will create new database
Generating publish script for database 'NerdDinner' on server '.\SQLEXPRESS'.
Successfully generated script to file C:\init\deploy.sql.
VERBOSE: Changed database context to 'master'.
VERBOSE: Creating NerdDinner...
VERBOSE: Changed database context to 'NerdDinner'.
VERBOSE: Creating [dbo].[Dinners]...
...
VERBOSE: Deployed NerdDinner database, data files at: C:\data
```

我使用的`docker container run`命令还发布了标准的 SQL Server 端口`1433`，因此您可以通过.NET 连接或**SQL Server Management Studio**（**SSMS**）远程连接到容器内运行的数据库。如果您的主机上已经运行了 SQL Server 实例，您可以将容器的端口`1433`映射到主机上的不同端口。

要使用 SSMS、Visual Studio 或 Visual Studio Code 连接到运行在容器中的 SQL Server 实例，请使用`localhost`作为服务器名称，选择 SQL Server 身份验证，并使用`sa`凭据。我使用的是**SqlElectron**，这是一个非常轻量级的 SQL 数据库客户端：

![](img/6fb1e102-ad53-4aca-a138-a1de00d35260.png)

然后，您可以像处理任何其他 SQL Server 数据库一样处理 Docker 化的数据库，查询表并插入数据。从 Docker 主机机器上，您可以使用`localhost`作为数据库服务器名称。通过发布端口，您可以在主机之外访问容器化的数据库，使用主机机器名称作为服务器名称。Docker 将端口`1433`上的任何流量路由到运行在容器上的 SQL Server。

# 从应用程序容器连接到数据库容器

Docker 平台内置了一个 DNS 服务器，容器用它来进行服务发现。我使用了一个显式名称启动了 NerdDinner 数据库容器，同一 Docker 网络中运行的任何其他容器都可以通过名称访问该容器，就像 Web 服务器通过其 DNS 主机名访问远程数据库服务器一样：

![](img/57115c85-6752-43a3-9e3e-9f7c07780995.png)

这使得应用程序配置比传统的分布式解决方案简单得多。每个环境看起来都是一样的。在开发、集成测试、QA 和生产中，Web 容器将始终使用`nerd-dinner-db`主机名连接到实际运行在容器内的数据库。容器可以在同一台 Docker 主机上，也可以在 Docker Swarm 集群中的另一台独立机器上，对应用程序来说是透明的。

Docker 中的服务发现不仅适用于容器。容器可以使用其主机名访问网络上的另一台服务器。您可以在容器中运行 Web 应用程序，但仍然让它连接到物理机上运行的 SQL Server，而不是使用数据库容器。

每个环境可能有一个不同的配置，那就是 SQL Server 的登录凭据。在 NerdDinner 数据库镜像中，我使用了与本章前面的`dockeronwindows/ch03-aspnet-config`相同的配置方法。我已经将`Web.config`中的`appSettings`和`connectionStrings`部分拆分成单独的文件，并且 Docker 镜像将这些配置文件与默认值捆绑在一起。

开发人员可以直接从镜像中运行容器，并且它将使用默认的数据库凭据，这些凭据与 NerdDinner 数据库 Docker 镜像中内置的默认凭据相匹配。在其他环境中，可以通过在主机服务器上使用配置文件进行卷挂载来运行容器，这些配置文件指定了不同的应用程序设置和数据库连接字符串。

这是一个简化的安全凭据方法，我用它来展示如何使我们的应用更加适合 Docker，而不改变代码。将凭据保存在服务器上的纯文本文件中并不是管理机密信息的好方法，我将在第九章*了解 Docker 的安全风险和好处*中再次讨论这个问题，当时我会介绍 Docker 中的安全性。

本章对 NerdDinner 的 Dockerfile 进行了一些更新。我添加了健康检查和从 IIS 中输出日志的设置。我仍然没有对 NerdDinner 代码库进行任何功能性更改，只是将`Web.config`文件拆分，并将默认数据库连接字符串设置为使用运行在 Docker 中的 SQL Server 数据库容器。现在运行 Web 应用程序容器时，它将能够通过名称连接到数据库容器，并使用在 Docker 中运行的 SQL Server Express 数据库：

```
docker container run -d -P dockeronwindows/ch03-nerd-dinner-web:2e
```

您可以在创建容器时明确指定 Docker 网络应加入的容器，但在 Windows 上，所有容器默认加入名为`nat`的系统创建的 Docker 网络。数据库容器和 Web 容器都连接到`nat`网络，因此它们可以通过容器名称相互访问。

当容器启动时，我现在可以使用容器的端口打开网站，点击注册链接并创建一个账户：

![](img/3a32ee44-2cc1-4d04-9244-6d2dd9139dad.png)

注册页面查询运行在 SQL Server 容器中的 ASP.NET 成员数据库。如果注册页面正常运行，则 Web 应用程序与数据库之间存在有效的连接。我可以在 Sqlectron 中验证这一点，查询`UserProfile`表并查看新用户行：

![](img/d293f55f-0ad4-4806-b9a6-f5e771799b2e.png)

我现在已将 SQL Server 数据库与 Web 应用程序分离，每个组件都在一个轻量级的 Docker 容器中运行。在我的开发笔记本上，每个容器在空闲时使用的主机 CPU 不到 1%，数据库使用 250MB 内存，Web 服务器使用 70MB。

`docker container top`可以显示容器内运行的进程信息，包括内存和 CPU。

容器资源占用较少，因此将功能单元拆分为不同的容器没有任何惩罚，然后可以单独扩展、部署和升级这些组件。

# 拆分单片应用程序

传统的依赖于 SQL Server 数据库的.NET Web 应用程序可以以最小的工作量迁移到 Docker，而无需重写任何应用程序代码。在我的 NerdDinner 迁移的这个阶段，我有一个应用程序 Docker 镜像和一个数据库 Docker 镜像，我可以可靠地和重复地部署和维护。我还有一些有益的副作用。

在 Visual Studio 项目中封装数据库定义可能是一种新的方法，但它可以为数据库脚本添加质量保证，并将模式引入代码库，因此可以与系统的其余部分一起进行源代码控制和管理。Dacpacs、PowerShell 脚本和 Dockerfiles 为不同的 IT 功能提供了一个新的共同基础。开发、运维和数据库管理团队可以共同使用相同的语言在相同的工件上进行工作。

Docker 是 DevOps 转型的推动者，但无论您的路线图上是否有 DevOps，Docker 都为快速、可靠的发布提供了基础。为了最大限度地利用这一点，您需要考虑将单片应用程序分解为更小的部分，这样您就可以频繁发布高价值组件，而无需对整个大型应用程序进行回归测试。

从现有应用程序中提取核心组件可以在不进行大规模、复杂的重写的情况下将现代、轻量级技术引入您的系统。您可以将微服务架构原则应用于现有解决方案，其中您已经了解了值得提取到自己服务中的领域。

# 从单体中提取高价值组件

Docker 平台为现代化传统应用程序提供了巨大的机会，使您可以将特性从单体中取出并在单独的容器中运行。如果您可以隔离特性中的逻辑，这也是将其迁移到.NET Core 的机会，这样您可以将其打包成更小的 Docker 镜像。

微软的.NET Core 路线图已经看到它采用了更多的完整.NET Framework 功能，但将传统.NET 应用程序的部分移植到.NET Core 仍然可能是一项艰巨的任务。这是一个值得评估的选项，但它不必成为您现代化方法的一部分。分解单体的价值在于拥有可以独立开发、部署和维护的功能。如果这些组件正在使用完整的.NET Framework，您仍然可以获得这些好处。

当您现代化传统应用程序时的优势在于您已经了解了功能集。您可以识别系统中的高价值功能，并从中提取这些功能到它们自己的组件中。优秀的候选对象将是那些如果频繁更改就能为业务提供价值的功能，因此新的功能请求可以快速构建和部署，而无需修改和测试整个应用程序。

同样优秀的候选特性是那些如果保持不变就能为 IT 提供价值的特性-具有许多依赖关系的复杂组件，业务很少改变。将这样的特性提取到一个单独的组件中意味着您可以部署主应用程序的升级，而无需测试复杂组件，因为它保持不变。像这样分解单体应用程序会给您一组具有自己交付节奏的组件。

在 NerdDinner 中，有一些很适合分离成自己的服务的候选项。在本章的其余部分，我将专注于其中之一：主页。主页是渲染应用程序第一页的 HTML 的功能。在生产环境中快速而安全地部署主页更改的过程将让业务能够尝试新的外观和感觉，评估新版本的影响，并决定是否继续使用它。

当前应用程序分布在两个容器之间。在本章的这一部分，我将把主页分离成自己的组件，这样整个 NerdDinner 应用程序将在三个容器中运行：

![](img/d7df03c2-ff8d-42c1-ae41-cda4c2ad0df0.png)

我不会改变应用程序的路由。用户仍然会首先进入 NerdDinner 应用程序，然后应用程序容器将调用新的主页服务容器以获取内容显示。这样我就不需要公开新的容器。更改只有一个技术要求：主应用程序需要能够与新的主页服务组件通信。

您可以自由选择容器中应用程序的通信方式。Docker 网络为 TCP/IP 和 UDP 提供了完整的协议支持。您可以使整个过程异步运行，将消息队列放在另一个容器中，并在其他容器中监听消息处理程序。但是在本章中，我将从更简单的方式开始。

# 在 ASP.NET Core 应用程序中托管 UI 组件

ASP.NET Core 是一个现代的应用程序堆栈，它在快速而轻量的运行时中提供了 ASP.NET MVC 和 Web API 的最佳功能。ASP.NET Core 网站作为控制台应用程序运行，它们将日志写入控制台输出流，并且它们可以使用环境变量和文件进行配置。这种架构使它们成为优秀的 Docker 公民。

将 NerdDinner 主页提取为一个新的服务的最简单方法是将其编写为一个 ASP.NET Core 网站，具有单个页面，并从现有应用程序中中继新应用程序的输出。以下屏幕截图显示了我在 Docker 中使用 ASP.NET Core Razor Pages 运行的时尚、现代化的主页重新设计：

![](img/76fda9ae-bd4e-4b22-894c-26fef2521d7a.png)

为了将主页应用程序打包为 Docker 镜像，我正在使用与主应用程序和数据库镜像相同的多阶段构建方法。在第十章中，*使用 Docker 支持持续部署流水线*，您将看到如何使用 Docker 来支持 CI/CD 构建流水线，并将整个自动化部署过程联系在一起。

`dockeronwindows/ch03-nerd-dinner-homepage:2e`镜像的 Dockerfile 使用了与完整 ASP.NET 应用程序相同的模式。构建器阶段使用 SDK 镜像并分离包恢复和编译步骤：

```
# escape=` FROM microsoft/dotnet:2.2-sdk-nanoserver-1809 AS builder WORKDIR C:\src\NerdDinner.Homepage COPY src\NerdDinner.Homepage\NerdDinner.Homepage.csproj . RUN dotnet restore COPY src\NerdDinner.Homepage . RUN dotnet publish  
```

Dockerfile 的最后阶段为`NERD_DINNER_URL`环境变量提供了默认值。应用程序将其用作主页上链接的目标。 Dockerfile 的其余指令只是复制已发布的应用程序并设置入口点：

```
FROM microsoft/dotnet:2.2-aspnetcore-runtime-nanoserver-1809 WORKDIR C:\dotnetapp ENV NERD_DINNER_URL="/home/find" EXPOSE 80 CMD ["dotnet", "NerdDinner.Homepage.dll"] COPY --from=builder C:\src\NerdDinner.Homepage\bin\Debug\netcoreapp2.2\publish .
```

我可以在单独的容器中运行主页组件，但它尚未连接到主 NerdDinner 应用程序。使用本章中采用的方法，我需要对原始应用程序进行代码更改，以便集成新的主页服务。

# 从其他应用程序容器连接到应用程序容器

从主应用程序容器调用新主页服务基本上与连接到数据库相同：我将使用已知名称运行主页容器，并且可以使用其名称和 Docker 内置服务发现在其他容器中访问服务。

在主 NerdDinner 应用程序的`HomeController`类中进行简单更改，将从新主页服务中继承响应，而不是从主应用程序呈现页面：

```
static  HomeController() {
  var  homepageUrl  =  Environment.GetEnvironmentVariable("HOMEPAGE_URL", EnvironmentVariableTarget.Machine); if (!string.IsNullOrEmpty(homepageUrl))
  {
    var  request  =  WebRequest.Create(homepageUrl); using (var  response  =  request.GetResponse())
    using (var  responseStream  =  new  StreamReader(response.GetResponseStream()))
    {
      _NewHomePageHtml  =  responseStream.ReadToEnd();
    }
 } } public  ActionResult  Index() { if (!string.IsNullOrEmpty(_NewHomePageHtml)) { return  Content(_NewHomePageHtml);
  }
  else
  {
    return  Find();
 } }
```

在新代码中，我从环境变量中获取主页服务的 URL。与数据库连接一样，我可以在 Dockerfile 中为其设置默认值。在分布式应用程序中，这将是不好的做法，因为我们无法保证组件在何处运行，但是在 Docker 化应用程序中，我可以安全地这样做，因为我将控制容器的名称，因此在部署它们时，我可以确保服务名称是正确的。

我已将此更新的镜像标记为`dockeronwindows/ch03-nerd-dinner-web:2e-v2`。现在，要启动整个解决方案，我需要运行三个容器：

```
docker container run -d -p 1433:1433 `
 --name nerd-dinner-db ` 
 -v C:\databases\nd:C:\data `
 dockeronwindows/ch03-nerd-dinner-db:2e

docker container run -d -P `
 --name nerd-dinner-homepage `
 dockeronwindows/ch03-nerd-dinner-homepage:2e

docker container run -d -P dockeronwindows/ch03-nerd-dinner-web:2e-v2
```

当容器正在运行时，我浏览到 NerdDinner 容器的发布端口，我可以看到来自新组件的主页：

![](img/f9efcc3f-8948-422d-84bb-56c6411792bb.png)

“找晚餐”链接将我带回原始的 Web 应用程序，现在我可以在主页上迭代并通过替换该容器发布新的用户界面 - 而无需发布或测试应用程序的其余部分。

新的用户界面发生了什么？在这个简单的例子中，集成的主页没有新的 ASP.NET Core 版本的样式，因为主应用程序只读取页面的 HTML，而不是 CSS 文件或其他资产。更好的方法是在容器中运行反向代理，并将其用作其他容器的入口点，这样每个容器都可以提供所有资产。我会在书中稍后做到这一点。

现在，我的解决方案分布在三个容器中，我大大提高了灵活性。在构建时，我可以专注于提供最高价值的功能，而不必费力测试未更改的组件。在部署时，我可以快速而自信地发布，知道我们推送到生产环境的新镜像将与测试的内容完全相同。然后在运行时，我可以根据其要求独立地扩展组件。

我确实有一个新的非功能性要求，那就是确保所有容器都具有预期的名称，按正确的顺序启动，并且在同一个 Docker 网络中，以便整个解决方案正常工作。Docker 对此提供了支持，重点是使用 Docker Compose 组织分布式系统。我会在第六章中向您展示这一点，*使用 Docker Compose 组织分布式解决方案*。

# 总结

在本章中，我们涵盖了三个主要主题。首先，我们介绍了将传统的.NET Framework 应用程序容器化，使其成为良好的 Docker 公民，并与平台集成以进行配置、日志记录和监视。

然后，我们介绍了如何使用 SQL Server Express 和 Dacpac 部署模型将数据库工作负载容器化，构建一个版本化的 Docker 镜像，可以将容器作为新数据库运行，或升级现有数据库。

最后，我们展示了如何将单片应用程序的功能提取到单独的容器中，使用 ASP.NET Core 和 Windows Nano Server 打包一个快速、轻量级的服务，主应用程序可以使用。

您已经学会了如何在 Docker Hub 上使用来自 Microsoft 的更多图像，以及如何为完整的.NET 应用程序使用 Windows Server Core，为数据库使用 SQL Server Express，以及.NET Core 图像的 Nano Server 版本。

在后面的章节中，我会回到 NerdDinner，并继续通过将功能提取到专用服务中来使其现代化。在那之前，在下一章中，我会更仔细地研究如何使用 Docker Hub 和其他注册表来存储镜像。

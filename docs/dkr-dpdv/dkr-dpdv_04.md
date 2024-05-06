# 第四章：安装 Docker

有很多种方式和地方可以安装 Docker。有 Windows，有 Mac，显然还有 Linux。但也有云端，本地，笔记本电脑等等...除此之外，我们还有手动安装，脚本安装，基于向导的安装...实际上有很多种方式和地方可以安装 Docker！

但不要让这吓到你！它们都很容易。

在本章中，我们将涵盖一些最重要的安装：

+   桌面安装

+   Docker for Windows

+   Docker for Mac

+   服务器安装

+   Linux

+   Windows Server 2016

+   升级 Docker

+   存储驱动器考虑

我们还将看看如何升级 Docker 引擎并选择适当的存储驱动程序。

### Windows 的 Docker（DfW）

首先要注意的是*Docker for Windows*是 Docker，Inc.的“打包”产品。这意味着它很容易下载，并且有一个漂亮的安装程序。它在 64 位 Windows 10 台式机或笔记本上启动一个单引擎 Docker 环境。

第二件要注意的事情是它是一个社区版（CE）应用。因此不适用于生产。

第三件值得注意的事情是它可能会遇到一些功能滞后。这是因为 Docker，Inc.正在采取“稳定性第一，功能第二”的方法来处理产品。

这三点加起来就是一个快速简单的安装，但不适用于生产。

废话够多了。让我们看看如何安装*Docker for Windows*。

首先，先决条件。*Docker for Windows*需要：

+   Windows 10 专业版|企业版|教育版（1607 周年更新，版本 14393 或更新版本）

+   必须是 64 位 Windows 10

+   *Hyper-V*和*容器*功能必须在 Windows 中启用

+   必须在系统的 BIOS 中启用硬件虚拟化支持

以下将假定硬件虚拟化支持已在系统的 BIOS 中启用。如果没有，您应该仔细遵循您特定机器的程序。

在 Windows 10 中要做的第一件事是确保**Hyper-V**和**容器**功能已安装并启用。

1.  右键单击 Windows 开始按钮，然后选择“应用和功能”。

1.  点击“程序和功能”链接（右侧的一个小链接）。

1.  点击“打开或关闭 Windows 功能”。

1.  勾选`Hyper-V`和“容器”复选框，然后点击“确定”。

这将安装并启用 Hyper-V 和容器功能。您的系统可能需要重新启动。

![图 3.1](img/Figure3-1.png)

图 3.1

*Containers*功能仅在运行 2016 年夏季的 Windows 10 周年更新（版本 14393）或更高版本时可用。

安装了`Hyper-V`和`Containers`功能并重新启动计算机后，现在是安装*Docker for Windows*的时候了。

1.  前往 https://www.docker.com/get-docker 并单击`GET DOCKER COMMUNITY EDITION`链接。

1.  单击`DOCKER CE FOR WINDOWS`部分下方的`Download from Docker Store`链接。这将带您到 Docker Store，您可能需要使用 Docker ID 登录。

1.  单击一个`Get Docker`下载链接。

Docker for Windows 有一个*stable*和*edge*通道。Edge 通道包含更新的功能，但可能不太稳定。

一个名为`Docker for Windows Installer.exe`的安装程序包将被下载到默认的下载目录。

1.  找到并启动在上一步中下载的安装程序包。

按照安装向导的步骤，并提供本地管理员凭据以完成安装。Docker 将自动启动为系统服务，并且 Moby Dock 鲸鱼图标将出现在 Windows 通知区域中。

恭喜！您已经安装了*Docker for Windows*。

打开命令提示符或 PowerShell 终端，尝试以下命令：

[PRE0]

客户端：

版本：18.01.0-ce

API 版本：1.35

Go 版本：go1.9.2

Git 提交：03596f5

构建时间：2018 年 1 月 10 日星期三 20:05:55

OS/Arch：windows/amd64

实验性：false

编排器：swarm

服务器：

引擎：

版本：18.01.0-ce

API 版本：1.35（最低版本 1.12）

Go 版本：go1.9.2

Git 提交：03596f5

构建时间：2018 年 1 月 10 日星期三 20:13:12

OS/Arch：linux/amd64

实验性：false

[PRE1]

注意，输出显示`OS/Arch: linux/amd64`是因为默认安装目前会在轻量级的 Linux Hyper-V 虚拟机中安装 Docker 守护程序。在这种情况下，您只能在*Docker for Windows*上运行 Linux 容器。

如果您想运行*本机 Windows 容器*，可以右键单击 Windows 通知区域中的 Docker 鲸鱼图标，然后选择`Switch to Windows containers...`。您也可以在命令行中使用以下命令（位于`\Program Files\Docker\Docker`目录中）来实现相同的功能：

[PRE2]

如果您没有启用`Windows Containers`功能，您将收到以下警报。

![图 3.2](img/figure3-2.png)

图 3.2

如果您已经启用了 Windows 容器功能，切换只需要几秒钟。切换完成后，`docker version`命令的输出将如下所示。

[PRE3]

`请注意，服务器版本现在显示为`windows/amd64`。这意味着守护程序在 Windows 内核上本地运行，并且只能运行 Windows 容器。

还要注意，系统现在正在运行 Docker 的*实验*版本（`实验性：true`）。如前所述，*Docker for Windows*有一个稳定的通道和一个边缘通道。在撰写本文时，Windows 容器是边缘通道的实验功能。

您可以使用`dockercli -Version`命令检查您正在运行的通道。 `dockercli`命令位于`C:\Program Files\Docker\Docker`中。

[PRE4]

以下列表显示了常规 Docker 命令的正常工作。

[PRE5]

Windows 的 Docker 包括 Docker Engine（客户端和守护程序）、Docker Compose、Docker Machine 和 Docker Notary 命令行。使用以下命令验证每个是否成功安装：

[PRE6]

[PRE7]

C:\> docker-machine --version

docker-machine.exe 版本 0.13.0，构建 9ba6da9

[PRE8]`

C:\> notary version

公证

版本：0.4.3

Git 提交：9211198

[PRE9]

$ docker version

客户端：

版本：17.05.0-ce

API 版本：1.29

Go 版本：go1.7.5

Git 提交：89658be

构建日期：星期四，2017 年 5 月 4 日 21:43:09

OS/Arch：darwin/amd64

服务器：

版本：17.05.0-ce

API 版本：1.29（最低版本 1.12）

Go 版本：go1.7.5

Git 提交：89658be

构建日期：星期四，2017 年 5 月 4 日 21:43:09

OS/Arch：linux/amd64

实验性：true

[PRE10]

$ docker --version

Docker 版本 17.05.0-ce，构建 89658be

$ docker image ls

存储库    标签    映像 ID    创建时间    大小

$ docker container ls

容器 ID   映像   命令   创建时间   状态   端口   名称

[PRE11]

$ docker --version

Docker 版本 17.05.0-ce，构建 89658be

[PRE12]`

$ docker-compose --version

docker-compose 版本 1.13.0，构建 1719ceb

[PRE13]`

$ docker-machine --version

docker-machine 版本 0.11.0，构建 5b27455

[PRE14]`

$ notary version

公证

版本：0.4.3

Git 提交：9211198

[PRE15]

$ wget -qO- https://get.docker.com/ `|` sh

modprobe：致命：未找到模块 aufs/lib/modules/4.4.0-36-generic

+ sh -c 'sleep 3; yum -y -q install docker-engine'

<剪辑>

如果您想以非 root 用户身份使用 Docker，您应该

现在考虑将您的用户添加到`"docker"`组中

类似于：

sudo usermod -aG docker your-user

请记住，您将不得不注销并重新登录...

[PRE16]

$ sudo usermod -aG docker npoulton

$ cat /etc/group `|` grep docker

docker:x:999:npoulton

[PRE17]

$ docker --version

Docker 版本`18`.01.0-ce，构建 03596f5

$ docker system info

容器：`0`

运行中：`0`

已暂停：`0`

已停止：`0`

镜像：`0`

服务器版本：`18`.01.0-ce

存储驱动程序：overlay2

后备文件系统：extfs

<Snip>

[PRE18]

> Install-Module DockerProvider -Force

[PRE19]

> Install-Package Docker -ProviderName DockerProvider -Force

[PRE20]

名称 版本 来源 摘要

---- ------- ------ -------

Docker 17.06.2-ee-6 Docker Docker for Windows Server 2016

[PRE21]

[PRE22]

> docker --version

Docker 版本 17.06.2-ee-6，构建 e75fdb8

> docker system info

容器：0

运行中：0

已暂停：0

已停止：0

镜像：0

服务器版本：17.06.2-ee-6

存储驱动程序：windowsfilter

<Snip>

[PRE23]

$ apt-get update

[PRE24]

$ apt-get remove docker docker-engine docker-ce docker.io -y

[PRE25]

$ wget -qO- https://get.docker.com/ | sh

[PRE26]

$ systemctl enable docker

同步 docker.service 的状态...

执行/lib/systemd/systemd-sysv-install enable docker

$ systemctl is-enabled docker

已启用

[PRE27]

$ docker container ls

容器 ID 图像 命令 创建状态 \

名称

97e599aca9f5 alpine "sleep 1d" 14 分钟前 上线 1 分钟

$ docker service ls

ID 名称 模式 副本 图像

ibyotlt1ehjy prod-equus1 复制 1/1 alpine:latest

[PRE28][PRE29]`

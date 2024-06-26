# 第三章：构建基础和分层图像

在本章中，我们将学习如何为生产就绪的容器构建基础和分层图像。正如我们所见，Docker 容器为我们提供了理想的环境，我们可以在其中构建、测试、自动化和部署。这些确切环境的再现性为我们提供了更高效和更有信心的效果，目前可用的基于脚本的部署系统无法轻易复制。开发人员在本地构建、测试和调试的图像可以直接推送到分段和生产环境中，因为测试环境几乎是应用程序代码运行的镜像。

图像是容器的字面基础组件，定义了部署的 Linux 版本和要包含和提供给容器内部运行的代码的默认工具。因此，图像构建是应用程序容器化生命周期中最关键的任务之一；正确构建图像对于容器化应用程序的有效、可重复和安全功能至关重要。

容器镜像由一组应用程序容器的运行时变量组成。理想情况下，容器镜像应尽可能精简，仅提供所需的功能，这有助于高效处理容器镜像，显著减少了从注册表上传和下载镜像的时间，并在主机上占用空间最小。

我们的重点、意图和方向是为您的 Docker 容器构建、调试和自动化图像。

本章我们将涵盖以下主题：

+   构建容器镜像

+   从头开始构建基础镜像

+   来自 Docker 注册表的官方基础镜像

+   从 Dockerfile 构建分层图像

+   通过测试调试图像

+   带有测试的自动化图像构建

# 构建容器镜像

由于本书试图*解决 Docker*的问题，减少我们需要解决的错误的机会不是很有益吗？幸运的是，Docker 社区（以及开源社区）提供了一个健康的基础（或*根*）镜像注册表，大大减少了错误并提供了更可重复的过程。在**Docker Registry**中搜索，我们可以找到广泛且不断增长的容器镜像的官方和自动化构建状态。 Docker 官方仓库（[`docs.docker.com/docker-hub/official_repos/)`](https://docs.docker.com/docker-hub/official_repos/)）是由 Docker Inc.支持的仔细组织的镜像集合-自动化仓库，允许您验证特定镜像的源和内容也存在。

本章的一个主要重点和主题将是基本的 Docker 基础知识；虽然对于经验丰富的容器用户来说可能看起来微不足道，但遵循一些最佳实践和标准化水平将有助于我们避免麻烦，并增强我们解决问题的能力。

## Docker Registry 的官方图像

标准化是可重复过程的重要组成部分。因此，无论何时何地，都应选择**Docker Hub**中提供的标准基础镜像，用于不同的 Linux 发行版（例如 CentOS、Debian、Fedora、RHEL、Ubuntu 等）或特定用例（例如 WordPress 应用程序）。这些基础镜像源自各自的 Linux 平台镜像，并专门用于容器中使用。此外，标准化的基础镜像经过良好维护，并经常更新以解决安全公告和关键错误修复。

这些基础镜像由 Docker Inc.构建、验证和支持，并通过它们的单词名称（例如`centos`）轻松识别。此外，Docker 社区的用户成员还提供和维护预构建的镜像以解决特定用例。这些用户镜像以创建它们的 Docker Hub 用户名为前缀，后缀为镜像名称（例如`tutum/centos`）。

![Docker Registry 的官方图像](img/image_03_001.jpg)

对我们来说，这些标准基础镜像仍然是准备就绪的，并且可以在 Docker Registry 上公开获取；可以使用`docker search`和`docker pull`终端命令简单地搜索和检索镜像。这将下载任何尚未位于 Docker 主机上的镜像。Docker Registry 在提供官方基础镜像方面变得越来越强大，可以直接使用，或者至少作为解决容器构建需求的一个可用的起点。

### 注意

虽然本书假设您熟悉 Docker Hub/Registry 和 GitHub/Bitbucket，但我们将首先介绍这些内容，作为您构建容器的高效镜像的首要参考线。您可以访问 Docker 镜像的官方注册表[`registry.hub.docker.com/`](https://registry.hub.docker.com/)。

![来自 Docker Registry 的官方镜像](img/image_03_002.jpg)

Docker Registry 可以从您的 Docker Hub 帐户或直接从终端进行搜索，如下所示：

```
$ sudo docker search centos

```

可以对搜索条件应用标志来过滤星级评分、自动构建等图像。要使用来自注册表的官方`centos`镜像，请从终端执行：

+   `$ sudo docker pull centos`：这将把`centos`镜像下载到您的主机上。

+   `$ sudo docker run centos`：这将首先在主机上查找此镜像，如果找不到，将会将镜像下载到主机上。镜像的运行参数将在其 Dockerfile 中定义。

### 用户存储库

此外，正如我们所见，我们不仅仅局限于官方 Docker 镜像的存储库。事实上，社区用户（无论是个人还是来自公司企业）已经准备好了满足某些需求的镜像。例如，创建了一个`ubuntu`镜像，用于在运行在 Apache、MySql 和 PHP 上的容器中运行`joomla`内容管理系统。

在这里，我们有一个用户存储库，其中有这样的镜像（`命名空间/存储库名称`）：

![用户存储库](img/image_03_004.jpg)

### 注意

**试一下：** 从终端练习从 Docker Registry 拉取和运行图像。

`$ sudo docker pull cloudconsulted/joomla`

从容器中拉取我们的基础镜像，并且`$ sudo docker run -d -p 80:80 cloudconsulted/joomla` 运行我们的容器镜像，并将主机的端口`80`映射到容器的端口`80`。

将你的浏览器指向`http://localhost`，你将会看到一个新的 Joomla 网站的构建页面！

## 构建我们自己的基础镜像

然而，可能会有情况需要创建定制的镜像以适应我们自己的开发和部署环境。如果你的使用情况要求使用非标准化的基础镜像，你将需要自己创建镜像。与任何方法一样，事先适当的规划是必要的。在构建镜像之前，你应该花足够的时间充分了解你的容器所要解决的使用情况。没有必要运行不符合预期应用程序的容器。其他考虑因素可能包括你在镜像中包含的库或二进制文件是否可重用，等等。一旦你觉得完成了，再次审查你的需求和要求，并过滤掉不必要的部分；我们不希望毫无理由地膨胀我们的容器。

使用 Docker Registry，你可以找到自动构建。这些构建是从 GitHub/Bitbucket 的仓库中拉取的，因此可以被 fork 并根据你自己的规格进行修改。然后，你新 fork 的仓库可以同步到 Docker Registry，生成你的新镜像，然后可以根据需要被拉取和运行到你的容器中。

### 注意

**试一下**：从以下仓库中拉取 ubuntu minimal 镜像，并将其放入你的 Dockerfile 目录中，以创建你自己的镜像：

`$ sudo docker pull cloudconsulted/ubuntu-dockerbase` `$ mkdir dockerbuilder` `$ cd dockerbuilder`

打开一个编辑器（vi/vim 或 nano）并创建一个新的 Dockerfile：

`$ sudo nano Dockerfile`

稍后我们将深入讨论如何创建良好的 Dockerfile，以及分层和自动化的镜像构建。现在，我们只想创建我们自己的新基础镜像，只是象征性地通过创建 Dockerfile 的过程和位置。为了简单起见，我们只是从我们想要构建新镜像的基础镜像中调用：

```
FROM cloudconsulted/ubuntu-dockerbase:latest 

```

保存并关闭这个 Dockerfile。现在我们在本地构建我们的新镜像：

```
$ sudo docker build -t mynew-ubuntu

```

让我们检查一下确保我们的新镜像已列出：

```
$ sudo docker images

```

注意我们的**IMAGE ID**为**mynew-ubuntu**，因为我们很快会需要它：

在 Docker Hub 用户名下创建一个新的公共/私有仓库。我在这里添加了新的仓库，命名为`<namespace><reponame>`，如`cloudconsulted/mynew-ubuntu`：

![构建我们自己的基础镜像](img/image_03_006.jpg)

接下来，返回到终端，这样我们就可以标记我们的新镜像以推送到我们的`<namespace>`下的新 Docker Hub 仓库：

```
$ sudo docker tag 1d4bf9f2c9c0 cloudconsulted/mynew-ubuntu:latest

```

确保我们的新镜像在我们的镜像列表中正确标记为`<namespace><repository>`：

```
$ sudo docker images

```

此外，我们将找到我们新创建的标记为推送到我们的 Docker Hub 仓库的镜像。

现在，让我们将镜像推送到我们的 Docker Hub 仓库：

```
$ sudo docker push cloudconsulted/mynew-ubuntu

```

然后，检查我们的新镜像是否在 Hub 上：

![构建我们自己的基础镜像](img/image_03_008.jpg)

构建自己的 Docker 镜像基本上有两种方法：

+   通过 bash shell 手动交互式构建层来安装必要的应用程序

+   通过 Dockerfile 自动化构建带有所有必要应用程序的镜像

### 使用 scratch 仓库构建镜像

构建自己的 Docker 容器镜像高度依赖于您打算打包的 Linux 发行版。由于这种差异性，以及通过 Docker Registry 已经可用的镜像的盛行和不断增长的注册表，我们不会花太多时间在这样的手动方法上。

在这里，我们可以再次查看 Docker Registry，以提供我们使用的最小镜像。一个`scratch`仓库已经从一个空的 TAR 文件中创建，可以通过`docker pull`简单地使用。与以前一样，根据您的参数制作 Dockerfile，然后您就有了新的镜像，从头开始。

通过使用可用工具（例如**supermin**（Fedora 系统）或**debootstrap**（Debian 系统）），这个过程甚至可以进一步简化。例如，使用这些工具，构建 Ubuntu 基础镜像的过程可以简单如下：

```
$ sudo debootstrap raring raring > /dev/null 
$ sudo tar -c raring -c . |  docker import - raring a29c15f1bf7a 
$ sudo docker run raring cat /etc/lsb-release 
DISTRIB_ID=Ubuntu 
DISTRIB_RELEASE=14.04 
DISTRIB_CODENAME=raring 
DISTRIB_DESCRIPTION="Ubuntu 14.04" 

```

## 构建分层镜像

Docker 的一个核心概念和特性是分层镜像。Docker 的最重要的特性之一是**镜像分层**和镜像内容的管理。容器镜像的分层方法非常高效，因为您可以引用镜像中的内容，识别分层镜像中的层。在构建多个镜像时，使用 Docker Registry 来推送和拉取镜像非常强大。

![构建分层镜像](img/image_03_009.jpg)

[镜像版权 © Docker, Inc.]

### 使用 Dockerfile 构建分层镜像

分层图像主要是使用**Dockerfile**构建的。实质上，Dockerfile 是一个脚本，它可以按照您需要的顺序从源（*基础*或*根*）镜像自动构建我们的容器，逐步、一层叠一层地由 Docker 守护程序执行。这些是在文件中列出的连续命令（指令）和参数，它们在基础镜像上执行一组规定的操作，每个命令构成一个新层，以构建一个新的镜像。这不仅有助于组织我们的镜像构建，而且通过简化大大增强了从头到尾的部署。Dockerfile 中的脚本可以以各种方式呈现给 Docker 守护程序，以为我们的容器构建新的镜像。

#### Dockerfile 构建

Dockerfile 的第一个命令通常是`FROM`命令。`FROM`指定要拉取的基础镜像。这个基础镜像可以位于公共 Docker 注册表（[`www.docker.com/`](https://www.docker.com/)）中，在私有注册表中，甚至可以是主机上的本地化 Docker 镜像。

Docker 镜像中的附加层根据 Dockerfile 中定义的指令进行填充。Dockerfile 具有非常方便的指令。在 Dockerfile 中定义的每个新指令都构成了分层图像中的一个**层**。通过`RUN`指令，我们可以指定要运行的命令，并将命令的结果作为图像中的附加层。

### 建议

强烈建议将图像中执行的操作逻辑分组，并将层的数量保持在最低限度。例如，在尝试为应用程序安装依赖项时，可以在一个`RUN`指令中安装所有依赖项，而不是使用每个依赖项的*N*个指令。

我们将在后面的章节“自动化镜像构建”中更仔细地检查 Dockerfile 的方面。现在，我们需要确保我们理解 Dockerfile 本身的概念和构造。让我们特别看一下可以使用的一系列简单命令。正如我们之前所看到的，我们的 Dockerfile 应该在包含我们现有代码（和/或其他依赖项、脚本和其他内容）的工作目录中创建。

### 建议

**注意：**避免使用根目录 [`/`] 作为源代码库的根目录。`docker build` 命令使用包含您的 Dockerfile 的目录作为构建上下文（包括其所有子目录）。构建上下文将在构建镜像之前发送到 Docker 守护程序，这意味着如果您使用 `/` 作为源代码库，您硬盘的整个内容将被发送到守护程序（因此发送到运行守护程序的机器）。在大多数情况下，最好将每个 Dockerfile 放在一个空目录中。然后，只向目录添加构建 Dockerfile 所需的文件。为了提高构建的性能，可以向上下文目录添加一个 `.dockerignore` 文件，以正确排除文件和目录。

#### Dockerfile 命令和语法

虽然简单，但我们的 Dockerfile 命令的顺序和语法非常重要。在这里正确关注细节和最佳实践不仅有助于确保成功的自动部署，还有助于任何故障排除工作。

让我们勾画一些基本命令，并直接用一个工作的 Dockerfile 来说明它们；我们之前的`joomla`镜像是一个基本的分层镜像构建的好例子。

### 注意

我们的示例 joomla 基本镜像位于公共 Docker 索引中

`cloudconsulted/joomla`。

**来自**

一个正确的 Dockerfile 从定义一个 `FROM` 镜像开始，构建过程从这里开始。这个指令指定要使用的基本镜像。它应该是 Dockerfile 中的第一个指令，对于通过 Dockerfile 构建镜像是必须的。您可以指定本地镜像、Docker 公共注册表中的镜像，或者私有注册表中的镜像。

**常见结构**

```
FROM <image> 
FROM <image>:<tag> 
FROM <image>@<digest> 

```

`<tag>` 和 `<digest>` 是可选的；如果您不指定它们，它默认为 `latest`。

**我们的 Joomla 镜像的示例 Dockerfile**

在这里，我们定义要用于容器的基本镜像：

```
# Image for container base 
FROM ubuntu 

```

**维护者**

这一行指定了构建镜像的*作者*。这是 Dockerfile 中的一个可选指令；然而，应该指定此指令与作者的姓名和/或电子邮件地址。`维护者`的详细信息可以放在您的 Dockerfile 中任何您喜欢的地方，只要它总是在您的 `FROM` 命令之后，因为它们不构成任何执行，而是一个定义的值（也就是一些额外的信息）。

**常见结构**

```
MAINTAINER <name><email> 

```

**我们的 Joomla 镜像的示例 Dockerfile**

在这里，我们为此容器和镜像定义了作者：

```
# Add name of image author 
MAINTAINER John Wooten <jwooten@cloudconsulted.com> 

```

**ENV**

此指令在 Dockerfile 中设置环境变量。设置的环境变量可以在后续指令中使用。

**常见结构**

```
ENV <key> <value> 

```

上述代码设置了一个环境变量`<key>`为`<value>`。

```
ENV <key1>=<value1> <key2>=<value2> 

```

上述指令设置了两个环境变量。使用`=`符号在环境变量的键和值之间，并用空格分隔两个环境键值来定义多个环境变量：

```
ENV key1="env value with space" 

```

对于具有空格值的环境变量，请使用引号。

以下是关于`ENV`指令的要点：

+   使用单个指令定义多个环境变量

+   创建容器时环境变量可用

+   可以使用`docker inspect <image>`从镜像中查看环境变量

+   环境变量的值可以通过向`docker run`命令传递`--env <key>=<value>`选项在运行时进行更改

**我们的 Joomla 镜像的示例 Dockerfile**

在这里，我们为 Joomla 和 Docker 镜像设置环境变量，而不使用交互式终端：

```
# Set the environment variables 
ENV DEBIAN_FRONTEND noninteractive 
ENV JOOMLA_VERSION 3.4.1 

```

**RUN**

此指令允许您运行命令并生成一个层。`RUN`指令的输出将是在进程中为镜像构建的一个层。传递给`RUN`指令的命令在此指令之前构建的层上运行；需要注意顺序。

**常见结构**

```
RUN <command> 

```

`<command>`在 shell 中执行-`/bin/sh -c` shell 形式。

```
RUN ["executable", "parameter1", "parameter2"] 

```

在这种特殊形式中，您可以在可执行形式中指定`可执行文件`和`参数`。确保在命令中传递可执行文件的绝对路径。这对于基础镜像没有`/bin/sh`的情况很有用。您可以指定一个可执行文件，它可以是基础镜像中的唯一可执行文件，并在其上构建层。

如果您不想使用`/bin/sh` shell，这也很有用。考虑一下：

```
RUN ["/bin/bash", "-c", "echo True!"] 
RUN <command1>;<command2> 

```

实际上，这是一个特殊形式的示例，您可以在其中指定多个由`;`分隔的命令。`RUN`指令一起执行这些命令，并为所有指定的命令构建一个单独的层。

**我们的 Joomla 镜像的示例 Dockerfile**

在这里，我们更新软件包管理器并安装所需的依赖项：

```
# Update package manager and install required dependencies 
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \ 
    mysql-server \ 
    apache2 \ 
    php5 \ 
    php5-imap \ 
    php5-mcrypt \ 
    php5-gd \ 
    php5-curl \ 
    php5-apcu \ 
    php5-mysqlnd \ 
    supervisor 

```

请注意，我们特意这样写，以便将新软件包作为它们自己的 apt-get install 行添加，遵循初始安装命令。

这样做是为了，如果我们需要添加或删除一个软件包，我们可以在 Dockerfile 中不需要重新安装所有其他软件包。显然，如果有这样的需要，这将节省大量的构建时间。

### 注意

**Docker 缓存：** Docker 首先会检查主机的镜像缓存，查找以前构建的任何匹配层。如果找到，Dockerfile 中的给定构建步骤将被跳过，以利用缓存中的上一层。因此，最佳实践是将 Dockerfile 的每个`apt-get -y install`命令单独列出。

正如我们所讨论的，Dockerfile 中的`RUN`命令将在 Docker 容器的上下文和文件系统下执行任何给定的命令，并生成具有任何文件系统更改的新镜像层。我们首先运行`apt-get update`以确保软件包的存储库和 PPA 已更新。然后，在单独的调用中，我们指示软件包管理器安装 MySQL、Apache、PHP 和 Supervisor。`-y`标志跳过交互式确认。

安装了我们所有必要的依赖项来运行我们的服务后，我们应该整理一下，以获得一个更干净的 Docker 镜像：

```
# Clean up any files used by apt-get 
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* 

```

**ADD**

这些信息用于将文件和目录从本地文件系统或远程 URL 中的文件复制到镜像中。源和目标必须在`ADD`指令中指定。

**常见结构**

```
ADD  <source_file>  <destination_directory> 

```

这里的`<source_file>`的路径是相对于构建上下文的。另外，`<destination_directory>`的路径可以是绝对的，也可以是相对于`WORKDIR`的：

```
ADD  <file1> <file2> <file3> <destination_directory> 

```

多个文件，例如`<file1>`，`<file2>`和`<file3>`，被复制到`<destination_directory>`中。请注意，这些源文件的路径应该相对于构建上下文，如下所示：

```
ADD <source_directory> <destination_directory> 

```

`<source_directory>`的内容与文件系统元数据一起复制到`<destination_directory>`中；目录本身不会被复制：

```
ADD text_* /text_files
```

在构建上下文目录中以`text_`开头的所有文件都会被复制到容器镜像中的`/text_files`目录中：

```
ADD ["filename with space",...,  "<dest>"] 

```

文件名中带有空格的情况可以在引号中指定；在这种情况下，需要使用 JSON 数组来指定 ADD 指令。

以下是关于`ADD`指令的要点：

+   所有复制到容器镜像中的新文件和目录的 UID 和 GID 都为`0`

+   在源文件是远程 URL 的情况下，目标文件将具有`600`的权限

+   在`ADD`指令的源中引用的所有本地文件应位于构建上下文目录或其子目录中

+   如果本地源文件是受支持的 tar 存档，则它将被解压缩为目录

+   如果指定了多个源文件，则目标必须是一个目录，并以斜杠`/`结尾

+   如果目标不存在，它将与路径中的所有父目录一起创建，如果需要的话

我们的 Joomla 镜像的示例 Dockerfile

在这里，我们将`joomla`下载到 Apache web 根目录：

```
# Download joomla and put it default apache web root 
ADD https://github.com/joomla/joomla-cms/releases/download/$JOOMLA_VERSION/Joomla_$JOOMLA_VERSION-Stable-Full_Package.tar.gz /tmp/joomla/ 
RUN tar -zxvf /tmp/joomla/Joomla_$JOOMLA_VERSION-Stable-Full_Package.tar.gz -C /tmp/joomla/ 
RUN rm -rf /var/www/html/* 
RUN cp -r /tmp/joomla/* /var/www/html/ 

# Put default htaccess in place 
RUN mv /var/www/html/htaccess.txt /var/www/html/.htaccess 

RUN chown -R www-data:www-data /var/www 

# Expose HTTP and MySQL 
EXPOSE 80 3306 

```

**COPY**

`COPY`命令指定应将位于输入路径的文件从与 Dockerfile 相同的目录复制到容器内部的输出路径。

**CMD**

`CMD`指令有三种形式-作为`ENTRYPOINT`的默认参数的 shell 形式和首选可执行形式。`CMD`的主要目的是为执行容器提供默认值。这些默认值可以包括或省略可执行文件，后者必须指定`ENTRYPOINT`指令。如果用户在 Docker `run`中指定参数，则它们将覆盖`CMD`中指定的默认值。如果您希望容器每次运行相同的可执行文件，则应考虑结合使用`ENTRYPOINT`和`CMD`。

以下是要记住的要点：

+   不要将`CMD`与`RUN`混淆-`RUN`实际上会执行命令并提交结果，而`CMD`不会在构建过程中执行命令，而是指定图像的预期命令

+   Dockerfile 只能执行一个`CMD`；如果列出多个，则只会执行最后一个`CMD`

我们的 Joomla 镜像的示例 Dockerfile

在这里，我们设置 Apache 以启动：

```
# Use supervisord to start apache / mysql 
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf 
CMD ["/usr/bin/supervisord", "-n"] 

```

以下是我们完成的 Joomla Dockerfile 的内容：

```
FROM ubuntu 
MAINTAINER John Wooten <jwooten@cloudconsulted.com> 

ENV DEBIAN_FRONTEND noninteractive 
ENV JOOMLA_VERSION 3.4.1 

RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \ 
    mysql-server \ 
    apache2 \ 
    php5 \ 
    php5-imap \ 
    php5-mcrypt \ 
    php5-gd \ 
    php5-curl \ 
    php5-apcu \ 
    php5-mysqlnd \ 
    supervisor 

# Clean up any files used by apt-get 
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* 

# Download joomla and put it default apache web root 
ADD https://github.com/joomla/joomla-cms/releases/download/$JOOMLA_VERSION/Joomla_$JOOMLA_VERSION-Stable-Full_Package.tar.gz /tmp/joomla/ 
RUN tar -zxvf /tmp/joomla/Joomla_$JOOMLA_VERSION-Stable-Full_Package.tar.gz -C /tmp/joomla/ 
RUN rm -rf /var/www/html/* 
RUN cp -r /tmp/joomla/* /var/www/html/ 

# Put default htaccess in place 
RUN mv /var/www/html/htaccess.txt /var/www/html/.htaccess 

RUN chown -R www-data:www-data /var/www 

# Expose HTTP and MySQL 
EXPOSE 80 3306 

# Use supervisord to start apache / mysql 
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf 
CMD ["/usr/bin/supervisord", "-n"] 

```

其他常见的 Dockerfile 命令如下：**ENTRYPOINT**

`ENTRYPOINT`允许您配置将作为可执行文件运行的容器。根据 Docker 的文档，我们将使用提供的示例；以下将启动`nginx`，并使用其默认内容，在端口`80`上进行侦听：

```
docker run -i -t --rm -p 80:80 nginx 

```

`docker run <image>`的命令行参数将在可执行形式的`ENTRYPOINT`中的所有元素之后追加，并将覆盖使用`CMD`指定的所有元素。这允许将参数传递给入口点，即`docker run <image> -d`将向入口点传递`-d`参数。您可以使用`docker run --entrypoint`标志覆盖`ENTRYPOINT`指令。

**LABEL**

该指令指定了图像的元数据。稍后可以使用`docker inspect <image>`命令来检查这些图像元数据。这里的想法是在图像元数据中添加关于图像的信息，以便轻松检索。为了从图像中获取元数据，不需要从图像创建容器（或将图像挂载到本地文件系统），Docker 将元数据与每个 Docker 图像关联，并为其定义了预定义的结构；使用`LABEL`，可以添加描述图像的附加关联元数据。

图像的标签是键值对。以下是在 Dockerfile 中使用`LABEL`的示例：

```
LABEL <key>=<value>  <key>=<value>  <key>=<value> 

```

此指令将向图像添加三个标签。还要注意，它将创建一个新层，因为所有标签都是在单个`LABEL`指令中添加的：

```
LABEL  "key"="value with spaces" 

```

如果标签值中有空格，请在标签中使用引号：

```
LABEL LongDescription="This label value extends over new \ 
line." 

```

如果标签的值很长，请使用反斜杠将标签值扩展到新行。

```
LABEL key1=value1 
LABEL key2=value2 

```

可以通过**行尾**（**EOL**）分隔它们来定义图像的多个标签。请注意，在这种情况下，将为两个不同的`LABEL`指令创建两个图像层。

关于`LABEL`指令的注意事项：

+   标签按照 Dockerfile 中描述的方式汇总在一起，并与`FROM`指令中指定的基本图像中的标签一起使用

+   如果标签中的`key`重复，后面的值将覆盖先前定义的键的值。

+   尝试在单个`LABEL`指令中指定所有标签，以生成高效的图像，从而避免不必要的图像层计数

+   要查看构建图像的标签，请使用`docker inspect <image>`命令

**WORKDIR**

此指令用于为 Dockerfile 中的后续`RUN`、`ADD`、`COPY`、`CMD`和`ENTRYPOINT`指令设置工作目录。

在 Dockerfile 中定义工作目录，容器中引用的所有后续相对路径将相对于指定的工作目录。

以下是使用`WORKDIR`指令的示例：

```
WORKDIR /opt/myapp 

```

前面的指令将`/opt/myapp`指定为后续指令的工作目录，如下所示：

```
WORKDIR /opt/ 
WORKDIR myapp 
RUN pwd 

```

前面的指令两次定义了工作目录。请注意，第二个`WORKDIR`将相对于第一个`WORKDIR`。`pwd`命令的结果将是`/opt/myapp`：

```
ENV SOURCEDIR /opt/src 
WORKDIR $SOURCEDIR/myapp 

```

工作目录可以解析之前定义的环境变量。在这个例子中，`WORKDIR`指令可以评估`SOURCEDIR`环境变量，结果的工作目录将是`/opt/src/myapp`。

**USER**

这将为后续的`RUN`、`CMD`和`ENTRYPOINT`指令设置用户。当从镜像创建和运行容器时，也会设置用户。

以下指令为镜像和容器设置了用户`myappuser`：

```
USER myappuser 

```

关于`USER`指令的注意事项：

+   可以使用`docker run`命令中的`--user=name|uid[:<group|gid>]`来覆盖用户容器的用户

# 镜像测试和调试

虽然我们可以赞赏容器的好处，但目前对其进行故障排除和有效监控会带来一些复杂性。由于容器设计上的隔离性，它们的环境可能会变得模糊不清。有效的故障排除通常需要进入容器本身的 shell，并且需要安装额外的 Linux 工具来查看信息，这使得调查变得更加困难。

通常，对我们的容器和镜像进行有意义的故障排除所需的工具、方法和途径需要在每个容器中安装额外的软件包。这导致以下结果：

+   连接或直接附加到容器的要求并非总是微不足道的

+   一次只能检查一个容器的限制

增加这些困难的是，使用这些工具给我们的容器增加了不必要的臃肿，这是我们最初在规划中试图避免的；极简主义是我们在使用容器时寻求的优势之一。让我们看看如何可以合理地利用一些基本命令获取有用的容器镜像信息，以及调查新出现的应用程序，使我们能够从外部监视和排除容器。

## 用于故障排除的 Docker 详细信息

现在您已经有了您的镜像（无论构建方法如何）并且 Docker 正在运行，让我们进行一些测试，以确保我们的构建一切正常。虽然这些可能看起来很常规和乏味，但作为故障排除的*自上而下*方法来运行以下任何或所有内容是一个很好的做法。

这里的前两个命令非常简单，看起来似乎太通用了，但它们将提供基本级别的细节，以便开始任何下游的故障排除工作--`$ docker version`和`$ docker info`。

## Docker 版本

首先确保我们知道我们正在运行的 Docker、Go 和 Git 的版本：

```
$ sudo docker version

```

## Docker 信息

此外，我们还应该了解我们的主机操作系统和内核版本，以及存储、执行和日志记录驱动程序。了解这些东西可以帮助我们从*自上而下*的角度进行故障排除：

```
$ sudo docker info

```

## Debian / Ubuntu 的故障排除说明

通过`$ sudo docker info`命令，您可能会收到以下警告中的一个或两个：

```
WARNING: No memory limit support 
WARNING: No swap limit support

```

您需要添加以下命令行参数到内核中，以启用内存和交换空间记账：

```
cgroup_enable=memory swapaccount=1

```

对于这些 Debian 或 Ubuntu 系统，如果使用默认的 GRUB 引导加载程序，则可以通过编辑`/etc/default/grub`并扩展`GRUB_CMDLINE_LINUX`来添加这些参数。找到以下行：

```
GRUB_CMDLINE_LINUX="" 

```

然后，用以下内容替换它：

```
GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1" 

```

然后，运行`update-grub`并重新启动主机。

## 列出已安装的 Docker 镜像

我们还需要确保容器实例实际上已经在本地安装了您的镜像。SSH 进入 docker 主机并执行`docker images`命令。您应该看到您的 docker 镜像列在其中，如下所示：

```
$ sudo docker images

```

*如果我的镜像没有出现怎么办？*检查代理日志，并确保您的容器实例能够通过 curl 访问您的 docker 注册表并打印出可用的标签：

```
curl [need to add in path to registry!]

```

### 注意

**$ sudo docker images 告诉我们什么：**我们的容器镜像已成功安装在主机上。

## 手动启动您的 Docker 镜像

既然我们知道我们的镜像已安装在主机上，我们需要知道它是否对 Docker 守护程序可访问。测试确保您的镜像可以在容器实例上运行的简单方法是尝试从命令行运行您的镜像。这里还有一个额外的好处：我们现在有机会进一步检查应用程序日志以进行进一步的故障排除。

让我们看一下以下示例：

```
$ sudo docker run -it [need to add in path to registry/latest bin!]

```

### 注意

**`$ sudo docker run <imagename>`告诉我们什么：**我们可以从 docker 守护程序访问容器镜像，并且还提供可访问的输出日志以进行进一步的故障排除。

*如果我的镜像无法运行？*检查是否有任何正在运行的容器。如果预期的容器没有在主机上运行，可能会有阻止它启动的问题：

```
$ sudo docker ps

```

当容器启动失败时，它不会记录任何内容。容器启动过程的日志输出位于主机上的`/var/log/containers`中。在这里，您会找到遵循`<service>_start_errors.log`命名约定的文件。在这些日志中，您会找到我们的`RUN`命令生成的任何输出，并且这是故障排除的推荐起点，以了解为什么您的容器启动失败。

### 提示

**提示：** Logspout ([`github.com/gliderlabs/logspout`](https://github.com/gliderlabs/logspout)) 是 Docker 容器的日志路由器，运行在 Docker 内部。Logspout 附加到主机上的所有容器，然后将它们的日志路由到您想要的位置。

虽然我们也可以查看`/var/log/messages`中的输出来尝试故障排除，但我们还有一些其他途径可以追求，尽管可能需要更多的工作量。

## 从缓存中检查文件系统状态

正如我们讨论过的，每次成功的`RUN`命令在我们的 Dockerfile 中，Docker 都会缓存整个文件系统状态。我们可以利用这个缓存来检查失败的`RUN`命令之前的最新状态。

完成任务的方法：

+   访问 Dockerfile 并注释掉失败的`RUN`命令，以及任何后续的`RUN`命令

+   重新保存 Dockerfile

+   重新执行`$ sudo docker build`和`$ sudo docker run`

## 图像层 ID 作为调试容器

每次 Docker 成功执行 Dockerfile 中的`RUN`命令时，图像文件系统中都会提交一个新的层。方便起见，您可以使用这些层 ID 作为图像来启动一个新的容器。

考虑以下 Dockerfile 作为示例：

```
FROM centos 
RUN echo 'trouble' > /tmp/trouble.txt 
RUN echo 'shoot' >> /tmp/shoot.txt 

```

如果我们从这个 Dockerfile 构建：

```
$ docker build -force-rm -t so26220957 .

```

我们将获得类似以下的输出：

```
Sending build context to Docker daemon 3.584 kB 
Sending build context to Docker daemon 
Step 0 : FROM ubuntu 
   ---> b750fe79269d 
Step 1 : RUN echo 'trouble' > /tmp/trouble.txt 
   ---> Running in d37d756f6e55 
   ---> de1d48805de2 
Removing intermediate container d37d756f6e55 
Step 2 : RUN echo 'bar' >> /tmp/shoot.txt 
Removing intermediate container a180fdacd268 
Successfully built 40fd00ee38e1

```

然后，我们可以使用前面的图像层 ID 从`b750fe79269d`、`de1d48805de2`和`40fd00ee38e1`开始新的容器：

```
$ docker run -rm b750fe79269d cat /tmp/trouble.txt 
cat: /tmp/trouble.txt No such file or directory 
$ docker run -rm de1d48805de2 cat /tmp/trouble.txt 
trouble 
$ docker run -rm 40fd00ee38e1 cat /tmp/trouble.txt 
trouble 
shoot

```

### 注意

我们使用`--rm`来删除所有调试容器，因为没有理由让它们在运行后继续存在。

*如果我的容器构建失败会发生什么？*由于构建失败时不会创建任何映像，我们将无法获得容器的哈希 ID。相反，我们可以记录前一层的 ID，并使用该 ID 运行一个带有该 ID 的 shell 的容器：

```
$ sudo docker run --rm -it <id_last_working_layer> bash -il

```

进入容器后，执行失败的命令以重现问题，修复命令并进行测试，最后使用修复后的命令更新 Dockerfile。

您可能还想启动一个 shell 并浏览文件系统，尝试命令等等：

```
$ docker run -rm -it de1d48805de2 bash -il 
root@ecd3ab97cad4:/# ls -l /tmp 
total 4 
-rw-r-r-- 1 root root 4 Jul 3 12:14 trouble.txt 
root@ecd3ab97cad4:/# cat /tmp/trouble.txt 
trouble 
root@ecd3ab97cad4:/#

```

## 其他示例

最后一个示例是注释掉以下 Dockerfile 中的内容，包括有问题的行。然后我们可以手动运行容器和 docker 命令，并以正常方式查看日志。在这个 Dockerfile 示例中：

```
RUN trouble 
RUN shoot 
RUN debug 

```

此外，如果失败是在射击，那么注释如下：

```
RUN trouble 
# RUN shoot 
# RUN debug 

```

然后，构建和运行：

```
$ docker build -t trouble . 
$ docker run -it trouble bash 
container# shoot 
...grep logs...

```

## 检查失败的容器进程

即使您的容器成功从命令行运行，检查任何失败的容器进程，不再运行的容器，并检查我们的容器配置也是有益的。

运行以下命令来检查失败或不再运行的容器，并注意`CONTAINER ID`以检查特定容器的配置：

```
$ sudo docker ps -a

```

注意容器的**状态**。如果您的任何容器的**状态**显示除`0`之外的退出代码，可能存在容器配置的问题。举个例子，一个错误的命令会导致退出代码为`127`。有了这些信息，您可以调试任务定义`CMD`字段。

虽然有些有限，但我们可以进一步检查容器以获取额外的故障排除细节：

```
$ **sudo docker inspect <containerId>

```

最后，让我们也分析一下容器的应用程序日志。容器启动失败的错误消息将在这里输出：

```
$ sudo docker logs <containerId>

```

## 其他潜在有用的资源

`$ sudo docker` top 给出了容器内运行的进程列表。

当您需要比`top`提供的更多细节时，可以使用`$ sudo docker htop`，它提供了一个方便的、光标控制的界面。`htop`比`top`启动更快，您可以垂直和水平滚动列表以查看所有进程和完整的命令行，您不需要输入进程号来终止进程或优先级值来接收进程。

当本书付印时，排除容器和镜像的机制可能已经得到了显著改善。Docker 社区正在致力于*内置*报告和监控解决方案，市场力量也必将带来更多的选择。

## 使用 sysdig 进行调试

与任何新技术一样，一些最初固有的复杂性会随着时间的推移而被排除，新的工具和应用程序也会被开发出来以增强它们的使用。正如我们所讨论的，容器目前确实属于这一类别。虽然我们已经看到 Docker Registry 中官方标准化镜像的可用性有所改善，但我们现在也看到了新出现的工具，这些工具可以帮助我们有效地管理、监视和排除我们的容器。

![使用 sysdig 进行调试](img/image_03_016.jpg)

Sysdig 为容器提供应用程序监控[图片版权© 2014 Draios, Inc.]

**Sysdig** ([`www.sysdig.org/`](http://www.sysdig.org/) [)](http://www.sysdig.org/) 就是这样的一个工具。作为一个用于系统级探索和排除容器化环境可见性的*au courant*应用程序，`sysdig`的美妙之处在于我们能够从外部访问容器数据（尽管`sysdig`实际上也可以安装在容器内部）。从高层来看，`sysdig`为我们的容器管理带来了以下功能：

+   能够访问和审查每个容器中的进程（包括内部和外部 PID）

+   能够深入到特定容器中

+   能够轻松过滤一组容器以进行进程审查和分析

Sysdig 提供有关 CPU 使用、I/O、日志、网络、性能、安全和系统状态的数据。重申一遍，这一切都可以从外部完成，而无需在我们的容器中安装任何东西。

我们将在本书中继续有效地使用`sysdig`来监视和排除与我们的容器相关的特定进程，但现在我们将提供一些示例来排除我们基本的容器进程和日志问题。

让我们安装`sysdig`到我们的主机上，以展示它对我们和我们的容器可以做什么！

### 单步安装

通过以 root 或`sudo`执行以下命令，可以在一步中完成`sysdig`的安装：

```
curl -s https://s3.amazonaws.com/download.draios.com/stable/install-sysdig | sudo bash

```

### 注意

**注意：**`sysdig`目前已在最新的 Debian 和 Ubuntu 版本中本地包含；但建议更新/运行安装以获取最新的软件包。

### 高级安装

根据`sysdig`维基百科，高级安装方法可能对脚本化部署或容器化环境有用。它也很容易；高级安装方法已列入 RHEL 和 Debian 系统。

### 什么是凿子？

要开始使用`sysdig`，我们应该了解一些专业术语，特别是**凿子**。在`sysdig`中，凿子是一些小脚本（用 Lua 编写），用于分析`sysdig`事件流以执行有用的操作。事件被有效地带到用户级别，附加上下文，然后可以应用脚本。凿子在活动系统上运行良好，但也可以与跟踪文件一起用于离线分析。您可以同时运行尽可能多的凿子。例如：

`topcontainers_error` chisel 将按错误数量显示顶部容器。

有关 sysdig 凿子的列表：

`$ sysdig -cl`（使用`-i`标志获取有关特定凿子的详细信息）

**单容器进程分析**

使用`topprocs_cpu`凿子的示例，我们可以应用过滤器：

```
$ sudo sysdig -pc -c topprocs_cpu container.name=zany_torvalds

```

这些是示例结果：

```
CPU%          Process       container.name   
------------------------------------------ 
02.49%        bash          zany_torvalds 
37.06%        curl          zany_torvalds 
0.82%         sleep         zany_torvalds

```

与使用`$ sudo docker top`（以及类似）不同，我们可以确定我们想要查看进程的确切容器；例如，以下示例仅显示来自`wordpress`容器的进程：

```
$ sudo sysdig -pc -c topprocs_cpu container.name contains wordpress 

CPU%           Process         container.name   
-------------------------------------------------- 
5.38%          apache2         wordpress3 
4.37%          apache2         wordpress2 
6.89%          apache2         wordpress4 
7.96%          apache2         wordpress1

```

**其他有用的 Sysdig 凿子和语法**

+   `topprocs_cpu`按 CPU 使用率显示顶部进程

+   `topcontainers_file`按 R+W 磁盘字节显示顶部容器

+   `topcontainers_net`按网络 I/O 显示顶部容器

+   `lscontainers`将列出正在运行的容器

+   `$ sudo sysdig -pc -cspy_logs`分析每个屏幕的所有日志

+   `$ sudo sysdig -pc -cspy_logs container.name=zany_torvalds`打印容器`zany_torvalds`的日志

## 故障排除-一个开放的社区等待您

一般来说，你可能遇到的大多数问题在其他地方和其他时间已经有人经历过。Docker 和开源社区、IRC 频道和各种搜索引擎都可以提供高度可访问的信息，并可能为你提供解决困扰的情况和条件的答案。充分利用开源社区（特别是 Docker 社区）来获取你所寻找的答案。就像任何新兴技术一样，在开始阶段，我们都在一起学习！

# 自动化镜像构建

有许多方法可以自动化构建容器镜像的过程；在一本书中无法合理地提供所有方法。在本书的后面章节中，我们将更深入地探讨一系列自动化选项和工具。在这种特定情况下，我们只讨论使用我们的 Dockerfile 进行自动化。我们已经讨论过 Dockerfile 可以用于自动化镜像构建，所以让我们更专门地研究 Dockerfile 自动化。

## 单元测试部署

在构建过程中，Docker 允许我们运行任何命令。让我们利用这一点，在构建镜像的同时启用单元测试。这些单元测试可以帮助我们在将镜像推送到分阶段或部署之前识别生产镜像中的问题，并且至少部分验证镜像的功能是否符合我们的意图和期望。如果单元测试成功运行，我们就有了一定程度的信心，我们有一个有效的服务运行环境。这也意味着，如果测试失败，我们的构建也会失败，有效地阻止了一个不工作的镜像进入生产环境。

使用我们之前的`cloudconsulted/joomla`仓库镜像，我们将建立一个自动构建的示例工作流程，并进行测试。我们将使用**PHPUnit**，因为它是 Joomla 项目开发团队正式使用的工具，它可以方便地针对整个堆栈（Joomla 代码、Apache、MySQL 和 PHP）运行单元测试。

进入`cloudconsulted/joomla`的 Dockerfile 目录（在我们的例子中是`dockerbuilder`），并进行以下更新。

执行以下命令安装 PHPUnit：

```
[# install composer to a specific directory 
curl -sS https://getcomposer.org/installer | php -- --install-dir=bin 
# use composer to install phpunit 
composer global require "phpunit/phpunit=4.1.*"]

```

PHPUnit 也可以通过执行以下命令进行安装：

```
[# install phpunit 
wget https://phar.phpunit.de/phpunit.phar 
chmod +x phpunit.phar 
mv phpunit.phar /usr/local/bin/phpunit 
# might also need to put the phpunit executable placed here? test this: 
cp /usr/local/bin/phpunit /usr/bin/phpunit]

```

现在，让我们用`phpunit`运行我们的单元测试：

```
# discover and run any tests within the source code 
RUN phpunit 

```

我们还需要确保将我们的单元测试`COPY`到镜像内的资产中：

```
# copy unit tests to assets 
COPY test /root/test 

```

最后，让我们做一些清理工作。为了确保我们的生产代码不能依赖（无论是意外还是其他原因）测试代码，一旦单元测试完成，我们应该删除那些测试文件：

```
# clean up test files 
RUN rm -rf test 

```

我们对 Dockerfile 的总更新包括：

```
wget https://phar.phpunit.de/phpunit.phar 
chmod +x phpunit.phar 
mv phpunit.phar /usr/local/bin/phpunit 

RUN phpunit   
COPY test /root/test 
RUN rm -rf test 

```

现在，我们有一个脚本化的 Dockerfile，每次构建此镜像时，都将完全测试我们的 Joomla 代码、Apache、MySQL 和 PHP 依赖项，作为构建过程的一个文字部分。结果是一个经过测试的、可重现的生产环境！

## 自动化测试部署

在我们对部署可行图像的信心增强之后，这个构建过程仍然需要开发人员或 DevOps 工程师在每次生产推送之前重新构建镜像。相反，我们将依赖于来自我们的 Docker 和 GitHub 存储库的自动构建。

我们的 GitHub 和 Docker Hub 存储库将用于自动化我们的构建。通过在 GitHub 上维护我们的 Dockerfile、依赖项、相关脚本等，对存储库进行任何推送或提交将自动强制将更新的推送到同步的 Docker Hub 存储库。我们在 Docker Hub 上拉取的生产图像会自动更新任何新的构建信息。

Docker Cloud 是最新的应用程序生命周期的一部分，它提供了一个托管的注册服务，具有构建和测试设施。Docker Cloud 扩展了 Tutum 的功能，并与 Docker Hub 更紧密地集成。借助 Docker Cloud 系统，管理员可以仅需点击几下即可在云中部署和扩展应用程序。持续交付代码集成和自动化构建、测试和部署工作流程。它还提供了对整个基础架构容器的可见性，并访问面向开发人员友好的 CLI 工具的程序化 RESTful API。因此，Docker Cloud 可用于自动化构建过程和测试部署。

以下是 Docker Cloud 的重要特性：

+   允许构建 Docker 镜像，并将云存储库链接到源代码，以便简化镜像构建过程

+   它允许将您的基础架构和云服务链接起来，以自动提供新节点

+   一旦镜像构建完成，它可以用于部署服务，并可以与 Docker Cloud 的服务和微服务集合进行链接

+   在 Docker Cloud 中，beta 模式下的 Swarm 管理可用于在 Docker Cloud 中创建 swarm 或将现有的 swarm 注册到 Docker Cloud 中使用 Docker ID

# 总结

Docker 和 Dockerfiles 为应用程序开发周期提供了可重复的流程，为开发人员和 DevOps 工程师提供了独特的便利-生产就绪的部署，注入了经过测试的镜像的信心和自动化的便利。这为最需要的人提供了高度的赋权，并导致了经过测试和生产就绪的图像构建的持续交付，我们可以完全自动化，延伸到我们的云端。

在本章中，我们了解到生产就绪的应用程序容器化中的一项关键任务是图像构建。构建基本和分层镜像以及避免故障排除的领域是我们涵盖的主要主题。在构建我们的基本镜像时，我们看到 Docker Registry 提供了丰富和经过验证的图像，我们可以自由地用于可重复的流程。我们还讨论了手动构建图像，从头开始。前进的时候，我们探讨了使用 Dockerfile 构建分层图像，并详细列出了 Dockerfile 命令。最后，一个示例工作流程说明了自动化图像构建以及镜像和容器的测试。在整个过程中，我们强调了故障排除领域和选项的方法和手段。

为您的应用程序容器构建简洁的 Docker 镜像对于应用程序的功能和可维护性至关重要。现在我们已经了解了构建基本和分层镜像以及基本的故障排除方法，我们将期待构建真实的应用程序镜像。在下一章中，我们将学习使用一组合适的镜像规划和构建多层应用程序。

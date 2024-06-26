# 第三章：构建镜像

在上一章中，我们详细向您解释了镜像和容器处理以及其维护技巧和提示。除此之外，我们还解释了在 Docker 容器上安装任何软件包的标准过程，然后将容器转换为镜像以供将来使用和操作。本章与之前的章节非常不同，它清楚地描述了如何使用`Dockerfile`构建 Docker 镜像的标准方式，这是为软件开发社区提供高度可用的 Docker 镜像的最有力的方式。利用`Dockerfile`是构建强大镜像的最有竞争力的方式。

本章将涵盖以下主题：

+   Docker 集成的镜像构建系统

+   Dockerfile 的语法快速概述

+   `Dockerfile`构建指令

+   Docker 如何存储镜像

# Docker 集成的镜像构建系统

Docker 镜像是容器的基本构建模块。这些镜像可以是非常基本的操作环境，比如我们在前几章中使用 Docker 进行实验时发现的`busybox`或`ubuntu`。另外，这些镜像也可以构建用于企业和云 IT 环境的高级应用程序堆栈。正如我们在上一章中讨论的，我们可以通过从基础镜像启动容器，安装所有所需的应用程序，进行必要的配置文件更改，然后将容器提交为镜像来手动制作镜像。

作为更好的选择，我们可以采用使用`Dockerfile`自动化方法来制作镜像。`Dockerfile`是一个基于文本的构建脚本，其中包含了一系列特殊指令，用于从基础镜像构建正确和相关的镜像。`Dockerfile`中的顺序指令可以包括基础镜像选择、安装所需应用程序、添加配置和数据文件，以及自动运行服务并将这些服务暴露给外部世界。因此，基于 Dockerfile 的自动化构建系统简化了镜像构建过程。它还在构建指令的组织方式和可视化完整构建过程的方式上提供了很大的灵活性。

Docker 引擎通过`docker build`子命令紧密集成了这个构建过程。在 Docker 的客户端-服务器范式中，Docker 服务器（或守护程序）负责完整的构建过程，Docker 命令行界面负责传输构建上下文，包括将`Dockerfile`传输到守护程序。

为了窥探本节中`Dockerfile`集成构建系统，我们将向您介绍一个基本的`Dockerfile`。然后我们将解释将该`Dockerfile`转换为图像，然后从该图像启动容器的步骤。我们的`Dockerfile`由两条指令组成，如下所示：

```
$ cat Dockerfile
FROM busybox:latest
CMD echo Hello World!!

```

接下来，我们将讨论前面提到的两条指令：

+   第一条指令是选择基础图像。在这个例子中，我们将选择`busybox:latest`图像

+   第二条指令是执行`CMD`命令，指示容器`echo Hello World!!`。

现在，让我们通过调用`docker build`以及`Dockerfile`的路径来生成一个 Docker 图像。在我们的例子中，我们将从存储`Dockerfile`的目录中调用`docker build`子命令，并且路径将由以下命令指定：

```
$ sudo docker build .

```

发出上述命令后，`build`过程将通过将`build context`发送到`daemon`并显示以下文本开始：

```
Sending build context to Docker daemon 3.072 kB
Sending build context to Docker daemon
Step 0 : from busybox:latest

```

构建过程将继续，并在完成后显示以下内容：

```
Successfully built 0a2abe57c325

```

在前面的例子中，图像是由`IMAGE ID 0a2abe57c325`构建的。让我们使用这个图像通过使用`docker run`子命令来启动一个容器，如下所示：

```
$ sudo docker run 0a2abe57c325
Hello World!!

```

很酷，不是吗？凭借极少的努力，我们已经能够制作一个以`busybox`为基础图像，并且能够扩展该图像以生成`Hello World!!`。这是一个简单的应用程序，但是使用相同的技术也可以实现企业规模的图像。

现在让我们使用`docker images`子命令来查看图像的详细信息，如下所示：

```
$ sudo docker images
REPOSITORY     TAG         IMAGE ID      CREATED       VIRTUAL SIZE
<none>       <none>       0a2abe57c325    2 hours ago    2.433 MB

```

在这里，你可能会惊讶地看到`IMAGE`（`REPOSITORY`）和`TAG`名称被列为`<none>`。这是因为当我们构建这个图像时，我们没有指定任何图像或任何`TAG`名称。你可以使用`docker tag`子命令指定一个`IMAGE`名称和可选的`TAG`名称，如下所示：

```
$ sudo docker tag 0a2abe57c325 busyboxplus

```

另一种方法是在`build`时使用`-t`选项为`docker build`子命令构建镜像名称，如下所示：

```
$ sudo docker build -t busyboxplus .

```

由于`Dockerfile`中的指令没有变化，Docker 引擎将高效地重用具有`ID 0a2abe57c325`的旧镜像，并将镜像名称更新为`busyboxplus`。默认情况下，构建系统会将`latest`作为`TAG`名称。可以通过在`IMAGE`名称之后指定`TAG`名称并在它们之间放置`:`分隔符来修改此行为。也就是说，`<image name>:<tag name>`是修改行为的正确语法，其中`<image name>`是镜像的名称，`<tag name>`是标签的名称。

再次使用`docker images`子命令查看镜像详细信息，您会注意到镜像（存储库）名称为`busyboxplus`，标签名称为`latest`：

```
$ sudo docker images
REPOSITORY     TAG         IMAGE ID      CREATED       VIRTUAL SIZE
busyboxplus     latest       0a2abe57c325    2 hours ago    2.433 MB

```

始终建议使用镜像名称构建镜像是最佳实践。

在体验了`Dockerfile`的魔力之后，我们将在随后的章节中向您介绍`Dockerfile`的语法或格式，并解释一打`Dockerfile`指令。

### 注意

最新的 Docker 发布版（1.5）在`docker build`子命令中增加了一个额外选项（`-f`），用于指定具有替代名称的`Dockerfile`。

# Dockerfile 的语法快速概述

在本节中，我们将解释`Dockerfile`的语法或格式。`Dockerfile`由指令、注释和空行组成，如下所示：

```
# Comment

INSTRUCTION arguments
```

`Dockerfile`的指令行由两个组件组成，指令行以指令本身开头，后面跟着指令的参数。指令可以以任何大小写形式编写，换句话说，它是不区分大小写的。然而，标准做法或约定是使用*大写*以便与参数区分开来。让我们再次看一下我们之前示例中的`Dockerfile`的内容：

```
FROM busybox:latest
CMD echo Hello World!!
```

这里，`FROM`是一个指令，它以`busybox:latest`作为参数，`CMD`是一个指令，它以`echo Hello World!!`作为参数。

`Dockerfile` 中的注释行必须以 `#` 符号开头。指令后的 `#` 符号被视为参数。如果 `#` 符号前面有空格，则 `docker build` 系统将视其为未知指令并跳过该行。现在，让我们通过一个示例更好地理解这些情况，以更好地理解注释行：

+   有效的 `Dockerfile` 注释行始终以 `#` 符号作为行的第一个字符：

```
# This is my first Dockerfile comment
```

+   `#` 符号可以作为参数的一部分：

```
CMD echo ### Welcome to Docker ###
```

+   如果 `#` 符号前面有空格，则构建系统将其视为未知指令：

```
    # this is an invalid comment line
```

`docker build` 系统会忽略 `Dockerfile` 中的空行，因此鼓励 `Dockerfile` 的作者添加注释和空行，以大大提高 `Dockerfile` 的可读性。

# Dockerfile 构建指令

到目前为止，我们已经看过集成构建系统、`Dockerfile` 语法和一个示例生命周期，包括如何利用示例 `Dockerfile` 生成镜像以及如何从该镜像中生成容器。在本节中，我们将介绍 `Dockerfile` 指令、它们的语法以及一些合适的示例。

## FROM 指令

`FROM` 指令是最重要的指令，也是 `Dockerfile` 的第一个有效指令。它设置了构建过程的基础镜像。随后的指令将使用这个基础镜像并在其上构建。`docker build` 系统允许您灵活地使用任何人构建的镜像。您还可以通过添加更精确和实用的功能来扩展它们。默认情况下，`docker build` 系统在 Docker 主机中查找镜像。但是，如果在 Docker 主机中找不到镜像，则 `docker build` 系统将从公开可用的 Docker Hub Registry 拉取镜像。如果 `docker build` 系统在 Docker 主机和 Docker Hub Registry 中找不到指定的镜像，则会返回错误。

`FROM` 指令具有以下语法：

```
FROM <image>[:<tag>]
```

在上述代码语句中，请注意以下内容：

+   `<image>`：这是将用作基础镜像的镜像的名称。

+   `<tag>`：这是该镜像的可选标签限定符。如果未指定任何标签限定符，则假定为标签 `latest`。

以下是使用镜像名称 `centos` 的 `FROM` 指令的示例：

```
FROM centos
```

以下是带有镜像名称`ubuntu`和标签限定符`14.04`的`FROM`指令的另一个示例：

```
FROM ubuntu:14.04
```

Docker 允许在单个`Dockerfile`中使用多个`FROM`指令以创建多个镜像。Docker 构建系统将拉取`FROM`指令中指定的所有镜像。Docker 不提供对使用多个`FROM`指令生成的各个镜像进行命名的任何机制。我们强烈不建议在单个`Dockerfile`中使用多个`FROM`指令，因为可能会产生破坏性的冲突。

## MAINTAINER 指令

`MAINTAINER`指令是`Dockerfile`的信息指令。此指令能力使作者能够在镜像中设置详细信息。Docker 不对在`Dockerfile`中放置`MAINTAINER`指令施加任何限制。但强烈建议您在`FROM`指令之后放置它。

以下是`MAINTAINER`指令的语法，其中`<author's detail>`可以是任何文本。但强烈建议您使用镜像作者的姓名和电子邮件地址，如此代码语法所示：

```
MAINTAINER <author's detail>
```

以下是带有作者姓名和电子邮件地址的`MAINTAINER`指令的示例：

```
MAINTAINER Dr. Peter <peterindia@gmail.com>
```

## `COPY`指令

`COPY`指令使您能够将文件从 Docker 主机复制到新镜像的文件系统中。以下是`COPY`指令的语法：

```
COPY <src> ... <dst>
```

前面的代码术语包含了这里显示的解释：

+   `<src>`：这是源目录，构建上下文中的文件，或者是执行`docker build`子命令的目录。

+   `...`：这表示可以直接指定多个源文件，也可以通过通配符指定多个源文件。

+   `<dst>`：这是新镜像的目标路径，源文件或目录将被复制到其中。如果指定了多个文件，则目标路径必须是目录，并且必须以斜杠`/`结尾。

推荐为目标目录或文件使用绝对路径。如果没有绝对路径，`COPY`指令将假定目标路径将从根目录`/`开始。`COPY`指令足够强大，可以用于创建新目录，并覆盖新创建的镜像中的文件系统。

在下面的示例中，我们将使用`COPY`指令将源构建上下文中的`html`目录复制到镜像文件系统中的`/var/www/html`，如下所示：

```
COPY html /var/www/html
```

这是另一个示例，多个文件（`httpd.conf`和`magic`）将从源构建上下文复制到镜像文件系统中的`/etc/httpd/conf/`：

```
COPY httpd.conf magic /etc/httpd/conf/
```

## ADD 指令

`ADD`指令类似于`COPY`指令。但是，除了`COPY`指令支持的功能之外，`ADD`指令还可以处理 TAR 文件和远程 URL。我们可以将`ADD`指令注释为“功能更强大的 COPY”。

以下是`ADD`指令的语法：

```
ADD <src> ... <dst>
```

`ADD`指令的参数与`COPY`指令的参数非常相似，如下所示：

+   `<src>`：这既可以是构建上下文中的源目录或文件，也可以是`docker build`子命令将被调用的目录中的文件。然而，值得注意的区别是，源可以是存储在构建上下文中的 TAR 文件，也可以是远程 URL。

+   `...`：这表示多个源文件可以直接指定，也可以使用通配符指定。

+   `<dst>`：这是新镜像的目标路径，源文件或目录将被复制到其中。

这是一个示例，演示了将多个源文件复制到目标镜像文件系统中的各个目标目录的过程。在此示例中，我们在源构建上下文中使用了一个 TAR 文件（`web-page-config.tar`），其中包含`http`守护程序配置文件和网页文件的目录结构，如下所示：

```
$ tar tf web-page-config.tar
etc/httpd/conf/httpd.conf
var/www/html/index.html
var/www/html/aboutus.html
var/www/html/images/welcome.gif
var/www/html/images/banner.gif

```

`Dockerfile`内容中的下一行包含一个`ADD`指令，用于将 TAR 文件（`web-page-config.tar`）复制到目标镜像，并从目标镜像的根目录（`/`）中提取 TAR 文件，如下所示：

```
ADD web-page-config.tar /

```

因此，`ADD`指令的 TAR 选项可用于将多个文件复制到目标镜像。

## ENV 指令

`ENV`指令在新镜像中设置环境变量。环境变量是键值对，可以被任何脚本或应用程序访问。Linux 应用程序在启动配置中经常使用环境变量。

以下一行形成了`ENV`指令的语法：

```
ENV <key> <value>
```

在这里，代码术语表示以下内容：

+   `<key>`：这是环境变量

+   `<value>`：这是要设置为环境变量的值

以下几行给出了`ENV`指令的两个示例，在第一行中，`DEBUG_LVL`已设置为`3`，在第二行中，`APACHE_LOG_DIR`已设置为`/var/log/apache`：

```
ENV DEBUG_LVL 3
ENV APACHE_LOG_DIR /var/log/apache
```

## USER 指令

`USER`指令设置新镜像中的启动用户 ID 或用户名。默认情况下，容器将以`root`作为用户 ID 或`UID`启动。实质上，`USER`指令将把默认用户 ID 从`root`修改为此指令中指定的用户 ID。

`USER`指令的语法如下：

```
USER <UID>|<UName>
```

`USER`指令接受`<UID>`或`<UName>`作为其参数：

+   `<UID>`：这是一个数字用户 ID

+   `<UName>`：这是一个有效的用户名

以下是一个示例，用于在启动时将默认用户 ID 设置为`73`。这里`73`是用户的数字 ID：

```
USER 73
```

但是，建议您拥有一个与`/etc/passwd`文件匹配的有效用户 ID，用户 ID 可以包含任意随机数值。但是，用户名必须与`/etc/passwd`文件中的有效用户名匹配，否则`docker run`子命令将失败，并显示以下错误消息：

```
finalize namespace setup user get supplementary groups Unable to find user

```

## WORKDIR 指令

`WORKDIR`指令将当前工作目录从`/`更改为此指令指定的路径。随后的指令，如`RUN`、`CMD`和`ENTRYPOINT`也将在`WORKDIR`指令设置的目录上工作。

以下一行提供了`WORKDIR`指令的适当语法：

```
WORKDIR <dirpath>
```

在这里，`<dirpath>`是要设置的工作目录的路径。路径可以是绝对路径或相对路径。在相对路径的情况下，它将相对于`WORKDIR`指令设置的上一个路径。如果在目标镜像文件系统中找不到指定的目录，则将创建该目录。

以下一行是`Dockerfile`中`WORKDIR`指令的一个明确示例：

```
WORKDIR /var/log
```

## VOLUME 指令

`VOLUME`指令在镜像文件系统中创建一个目录，以后可以用于从 Docker 主机或其他容器挂载卷。

`VOLUME`指令有两种语法，如下所示：

+   第一种类型是 exec 或 JSON 数组（所有值必须在双引号（`"`）内）：

```
VOLUME ["<mountpoint>"]
```

+   第二种类型是 shell，如下所示：

```
VOLUME <mountpoint>
```

在前一行中，`<mountpoint>`是必须在新镜像中创建的挂载点。

## EXPOSE 指令

`EXPOSE`指令打开容器网络端口，用于容器与外部世界之间的通信。

`EXPOSE`指令的语法如下：

```
EXPOSE <port>[/<proto>] [<port>[/<proto>]...]
```

在这里，代码术语的含义如下：

+   `<port>`：这是要向外部世界暴露的网络端口。

+   `<proto>`：这是一个可选字段，用于指定特定的传输协议，如 TCP 和 UDP。如果未指定传输协议，则假定 TCP 为传输协议。

`EXPOSE`指令允许您在一行中指定多个端口。

以下是`Dockerfile`中`EXPOSE`指令的示例，将端口号`7373`暴露为`UDP`端口，端口号`8080`暴露为`TCP`端口。如前所述，如果未指定传输协议，则假定`TCP`传输协议为传输协议：

```
EXPOSE 7373/udp 8080
```

## RUN 指令

`RUN`指令是构建时的真正工作马，它可以运行任何命令。一般建议使用一个`RUN`指令执行多个命令。这样可以减少生成的 Docker 镜像中的层，因为 Docker 系统固有地为`Dockerfile`中每次调用指令创建一个层。

`RUN`指令有两种语法类型：

+   第一种是 shell 类型，如下所示：

```
RUN <command>
```

在这里，`<command>`是在构建时必须执行的 shell 命令。如果要使用这种类型的语法，那么命令总是使用`/bin/sh -c`来执行。

+   第二种语法类型要么是 exec，要么是 JSON 数组，如下所示：

```
RUN ["<exec>", "<arg-1>", ..., "<arg-n>"]
```

在其中，代码术语的含义如下：

+   `<exec>`：这是在构建时要运行的可执行文件。

+   `<arg-1>, ..., <arg-n>`：这些是可执行文件的参数（零个或多个）。

与第一种语法不同，这种类型不会调用`/bin/sh -c`。因此，这种类型不会发生 shell 处理，如变量替换（`$USER`）和通配符替换（`*`，`?`）。如果 shell 处理对您很重要，那么建议您使用 shell 类型。但是，如果您仍然更喜欢 exec（JSON 数组类型），那么请使用您喜欢的 shell 作为可执行文件，并将命令作为参数提供。

例如，`RUN ["bash", "-c", "rm", "-rf", "/tmp/abc"]`。

现在让我们看一下`RUN`指令的一些示例。在第一个示例中，我们将使用`RUN`指令将问候语添加到目标图像文件系统的`.bashrc`文件中，如下所示：

```
RUN echo "echo Welcome to Docker!" >> /root/.bashrc
```

第二个示例是一个`Dockerfile`，其中包含在`Ubuntu 14.04`基础镜像上构建`Apache2`应用程序镜像的指令。接下来的步骤将逐行解释`Dockerfile`指令：

1.  我们将使用`FROM`指令构建一个以`ubuntu:14.04`为基础镜像的镜像，如下所示：

```
###########################################
# Dockerfile to build an Apache2 image
###########################################
# Base image is Ubuntu
FROM ubuntu:14.04
```

1.  通过使用`MAINTAINER`指令设置作者的详细信息，如下所示：

```
# Author: Dr. Peter
MAINTAINER Dr. Peter <peterindia@gmail.com>
```

1.  通过一个`RUN`指令，我们将同步`apt`存储库源列表，安装`apache2`软件包，然后清理检索到的文件，如下所示：

```
# Install apache2 package
RUN apt-get update && \
   apt-get install -y apache2 && \
   apt-get clean
```

## CMD 指令

`CMD`指令可以运行任何命令（或应用程序），类似于`RUN`指令。但是，这两者之间的主要区别在于执行时间。通过`RUN`指令提供的命令在构建时执行，而通过`CMD`指令指定的命令在从新创建的镜像启动容器时执行。因此，`CMD`指令为此容器提供了默认执行。但是，可以通过`docker run`子命令参数进行覆盖。应用程序终止时，容器也将终止，并且应用程序与之相反。

`CMD`指令有三种语法类型，如下所示：

+   第一种语法类型是 shell 类型，如下所示：

```
CMD <command>
```

在其中，`<command>`是 shell 命令，在容器启动时必须执行。如果使用此类型的语法，则始终使用`/bin/sh -c`执行命令。

+   第二种语法类型是 exec 或 JSON 数组，如下所示：

```
CMD ["<exec>", "<arg-1>", ..., "<arg-n>"]
```

在其中，代码术语的含义如下：

+   `<exec>`：这是要在容器启动时运行的可执行文件。

+   `<arg-1>, ..., <arg-n>`：这些是可执行文件的参数的变量（零个或多个）数字。

+   第三种语法类型也是 exec 或 JSON 数组，类似于前一种类型。但是，此类型用于将默认参数设置为`ENTRYPOINT`指令，如下所示：

```
CMD ["<arg-1>", ..., "<arg-n>"]
```

在其中，代码术语的含义如下：

+   `<arg-1>, ..., <arg-n>`：这些是`ENTRYPOINT`指令的变量（零个或多个）数量的参数，将在下一节中解释。

从语法上讲，你可以在`Dockerfile`中添加多个`CMD`指令。然而，构建系统会忽略除最后一个之外的所有`CMD`指令。换句话说，在多个`CMD`指令的情况下，只有最后一个`CMD`指令会生效。

在这个例子中，让我们使用`Dockerfile`和`CMD`指令来制作一个镜像，以提供默认执行，然后使用制作的镜像启动一个容器。以下是带有`CMD`指令的`Dockerfile`，用于`echo`一段文本：

```
########################################################
# Dockerfile to demonstrate the behaviour of CMD
########################################################
# Build from base image busybox:latest
FROM busybox:latest
# Author: Dr. Peter
MAINTAINER Dr. Peter <peterindia@gmail.com>
# Set command for CMD
CMD ["echo", "Dockerfile CMD demo"]
```

现在，让我们使用`docker build`子命令和`cmd-demo`作为镜像名称来构建一个 Docker 镜像。`docker build`系统将从当前目录（`.`）中读取`Dockerfile`中的指令，并相应地制作镜像，就像这里所示的那样：

```
$ sudo docker build -t cmd-demo .

```

构建了镜像之后，我们可以使用`docker run`子命令来启动容器，就像这里所示的那样：

```
$ sudo docker run cmd-demo
Dockerfile CMD demo

```

很酷，不是吗？我们为容器提供了默认执行，并且我们的容器忠实地回显了`Dockerfile CMD demo`。然而，这个默认执行可以很容易地被通过将另一个命令作为参数传递给`docker run`子命令来覆盖，就像下面的例子中所示：

```
$ sudo docker run cmd-demo echo Override CMD demo
Override CMD demo

```

## ENTRYPOINT 指令

`ENTRYPOINT`指令将帮助制作一个镜像，用于在容器的整个生命周期中运行一个应用程序（入口点），该应用程序将从镜像中衍生出来。当入口点应用程序终止时，容器也将随之终止，应用程序与容器相互关联。因此，`ENTRYPOINT`指令会使容器的功能类似于可执行文件。从功能上讲，`ENTRYPOINT`类似于`CMD`指令，但两者之间的主要区别在于入口点应用程序是通过`ENTRYPOINT`指令启动的，无法通过`docker run`子命令参数来覆盖。然而，这些`docker run`子命令参数将作为额外的参数传递给入口点应用程序。话虽如此，Docker 提供了通过`docker run`子命令中的`--entrypoint`选项来覆盖入口点应用程序的机制。`--entrypoint`选项只能接受一个单词作为其参数，因此其功能有限。

从语法上讲，`ENTRYPOINT`指令与`RUN`和`CMD`指令非常相似，它有两种语法，如下所示：

+   第一种语法是 shell 类型，如下所示：

```
ENTRYPOINT <command>
```

在这里，`<command>`是在容器启动时执行的 shell 命令。如果使用这种类型的语法，则始终使用`/bin/sh -c`执行命令。

+   第二种语法是 exec 或 JSON 数组，如下所示：

```
ENTRYPOINT ["<exec>", "<arg-1>", ..., "<arg-n>"]
```

在这里，代码术语的含义如下：

+   `<exec>`：这是在容器启动时必须运行的可执行文件。

+   `<arg-1>, ..., <arg-n>`：这些是可执行文件的变量（零个或多个）参数。

从语法上讲，你可以在`Dockerfile`中有多个`ENTRYPOINT`指令。然而，构建系统将忽略除最后一个之外的所有`ENTRYPOINT`指令。换句话说，在多个`ENTRYPOINT`指令的情况下，只有最后一个`ENTRYPOINT`指令会生效。

为了更好地理解`ENTRYPOINT`指令，让我们使用带有`ENTRYPOINT`指令的`Dockerfile`来创建一个镜像，然后使用这个镜像启动一个容器。以下是带有`ENTRYPOINT`指令的`Dockerfile`，用于回显文本：

```
########################################################
# Dockerfile to demonstrate the behaviour of ENTRYPOINT
########################################################
# Build from base image busybox:latest
FROM busybox:latest
# Author: Dr. Peter
MAINTAINER Dr. Peter <peterindia@gmail.com>
# Set entrypoint command
ENTRYPOINT ["echo", "Dockerfile ENTRYPOINT demo"]
```

现在，让我们使用`docker build`作为子命令和`entrypoint-demo`作为镜像名称来构建一个 Docker 镜像。`docker build`系统将从当前目录（`.`）中存储的`Dockerfile`中读取指令，并创建镜像，如下所示：

```
$ sudo docker build -t entrypoint-demo .

```

构建完镜像后，我们可以使用`docker run`子命令启动容器：

```
$ sudo docker run entrypoint-demo
Dockerfile ENTRYPOINT demo

```

在这里，容器将像可执行文件一样运行，回显`Dockerfile ENTRYPOINT demo`字符串，然后立即退出。如果我们向`docker run`子命令传递任何额外的参数，那么额外的参数将传递给入口点命令。以下是使用`docker run`子命令给出额外参数启动相同镜像的演示：

```
$ sudo docker run entrypoint-demo with additional arguments
Dockerfile ENTRYPOINT demo with additional arguments

```

现在，让我们看一个例子，我们可以使用`--entrypoint`选项覆盖构建时的入口应用程序，然后在`docker run`子命令中启动一个 shell（`/bin/sh`），如下所示：

```
$ sudo docker run --entrypoint="/bin/sh" entrypoint-demo
/ #

```

## ONBUILD 指令

`ONBUILD`指令将构建指令注册到镜像中，并在使用此镜像作为其基本镜像构建另一个镜像时触发。任何构建指令都可以注册为触发器，并且这些指令将在下游`Dockerfile`中的`FROM`指令之后立即触发。因此，`ONBUILD`指令可用于将构建指令的执行从基本镜像延迟到目标镜像。

`ONBUILD`指令的语法如下：

```
ONBUILD <INSTRUCTION>
```

在其中，`<INSTRUCTION>`是另一个`Dockerfile`构建指令，稍后将被触发。`ONBUILD`指令不允许链接另一个`ONBUILD`指令。此外，它不允许`FROM`和`MAINTAINER`指令作为`ONBUILD`触发器。

以下是`ONBUILD`指令的示例：

```
ONBUILD ADD config /etc/appconfig
```

## .dockerignore 文件

在*Docker 集成的镜像构建系统*部分，我们了解到`docker build`过程将完整的构建上下文发送到守护程序。在实际环境中，`docker build`上下文将包含许多其他工作文件和目录，这些文件和目录永远不会构建到镜像中。然而，`docker build`系统仍然会将这些文件发送到守护程序。因此，您可能想知道如何通过不将这些工作文件发送到守护程序来优化构建过程。嗯，Docker 背后的人也考虑过这个问题，并提供了一个非常简单的解决方案：使用`.dockerignore`文件。

`.dockerignore`是一个以换行分隔的文本文件，在其中您可以提供要从构建过程中排除的文件和目录。文件中的排除列表可以包含完全指定的文件或目录名称和通配符。

以下片段是一个示例`.dockerignore`文件，通过它，构建系统已被指示排除`.git`目录和所有具有`.tmp`扩展名的文件：

```
.git
*.tmp
```

# Docker 镜像管理的简要概述

正如我们在前一章和本章中所看到的，有许多方法可以控制 Docker 镜像。您可以使用`docker pull`子命令从公共存储库下载完全设置好的应用程序堆栈。否则，您可以通过使用`docker commit`子命令手动或使用`Dockerfile`和`docker build`子命令组合自动创建自己的应用程序堆栈。

Docker 镜像被定位为容器化应用程序的关键构建模块，从而实现了部署在云服务器上的分布式应用程序。Docker 镜像是分层构建的，也就是说，可以在其他镜像的基础上构建镜像。原始镜像称为父镜像，生成的镜像称为子镜像。基础镜像是一个捆绑包，包括应用程序的常见依赖项。对原始镜像所做的每个更改都将作为单独的层存储。每次提交到 Docker 镜像时，都会在 Docker 镜像上创建一个新的层，对原始镜像所做的每个更改都将作为单独的层存储。由于层的可重用性得到了便利，制作新的 Docker 镜像变得简单而快速。您可以通过更改`Dockerfile`中的一行来创建新的 Docker 镜像，而无需重新构建整个堆栈。

现在我们已经了解了 Docker 镜像中的层次结构，您可能想知道如何在 Docker 镜像中可视化这些层。好吧，`docker history`子命令是可视化图像层的一个非常好用的工具。

让我们看一个实际的例子，以更好地理解 Docker 镜像中的分层。为此，让我们按照以下三个步骤进行：

1.  在这里，我们有一个`Dockerfile`，其中包含自动构建 Apache2 应用程序镜像的指令，该镜像是基于 Ubuntu 14.04 基础镜像构建的。本章之前制作和使用的`Dockerfile`中的`RUN`部分将在本节中被重用，如下所示：

```
###########################################
# Dockerfile to build an Apache2 image
###########################################
# Base image is Ubuntu
FROM ubuntu:14.04
# Author: Dr. Peter
MAINTAINER Dr. Peter <peterindia@gmail.com>
# Install apache2 package
RUN apt-get update && \
   apt-get install -y apache2 && \
   apt-get clean
```

1.  现在，通过使用`docker build`子命令从上述`Dockerfile`中制作一个镜像，如下所示：

```
$ sudo docker build -t apache2 .

```

1.  最后，让我们使用`docker history`子命令来可视化 Docker 镜像中的层次结构：

```
$ sudo docker history apache2

```

1.  这将生成关于`apache2` Docker 镜像的每个层的详细报告，如下所示：

```
IMAGE          CREATED       CREATED BY                   SIZE
aa83b67feeba    2 minutes ago    /bin/sh -c apt-get update &&   apt-get inst  35.19 MB
c7877665c770    3 minutes ago    /bin/sh -c #(nop) MAINTAINER Dr. Peter <peter  0 B
9cbaf023786c    6 days ago     /bin/sh -c #(nop) CMD [/bin/bash]        0 B
03db2b23cf03    6 days ago     /bin/sh -c apt-get update && apt-get dist-upg  0 B
8f321fc43180    6 days ago     /bin/sh -c sed -i 's/^#\s*\(deb.*universe\)$/  1.895 kB
6a459d727ebb    6 days ago     /bin/sh -c rm -rf /var/lib/apt/lists/*     0 B
2dcbbf65536c    6 days ago     /bin/sh -c echo '#!/bin/sh' > /usr/sbin/polic 194.5 kB
97fd97495e49    6 days ago     /bin/sh -c #(nop) ADD file:84c5e0e741a0235ef8  192.6 MB
511136ea3c5a    16 months ago                            0 B

```

在这里，`apache2`镜像由十个镜像层组成。顶部两层，具有图像 ID`aa83b67feeba`和`c7877665c770`的层，是我们`Dockerfile`中`RUN`和`MAINTAINER`指令的结果。图像的其余八层将通过我们`Dockerfile`中的`FROM`指令从存储库中提取。

# 编写 Dockerfile 的最佳实践

毫无疑问，一套最佳实践总是在提升任何新技术中起着不可或缺的作用。有一份详细列出所有最佳实践的文件，用于编写`Dockerfile`。我们发现它令人难以置信，因此，我们希望分享给您以供您受益。您可以在[`docs.docker.com/articles/dockerfile_best-practices/`](https://docs.docker.com/articles/dockerfile_best-practices/)找到它。

# 摘要

构建 Docker 镜像是 Docker 技术的关键方面，用于简化容器化的繁琐任务。正如之前所指出的，Docker 倡议已经成为颠覆性和变革性的容器化范式。Dockerfile 是生成高效 Docker 镜像的最主要方式，可以被精心使用。我们已经阐明了所有命令、它们的语法和使用技巧，以赋予您所有易于理解的细节，这将简化您的镜像构建过程。我们提供了大量示例，以证实每个命令的内在含义。在下一章中，我们将讨论 Docker Hub，这是一个专门用于存储和共享 Docker 镜像的存储库，并且我们还将讨论它对容器化概念在 IT 企业中的深远贡献。

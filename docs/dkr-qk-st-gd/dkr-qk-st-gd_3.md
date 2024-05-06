# 创建 Docker 镜像

在本章中，我们将学习如何创建企业级的 Docker 镜像。我们将首先学习 Docker 镜像的主要构建块，具体来说是 Dockerfile。然后，我们将探索 Dockerfile 中可用的所有指令。有一些指令在表面上看起来非常相似。我们将揭示`COPY`和`ADD`指令之间的区别，`ENV`和`ARG`指令之间的区别，以及最重要的是`CMD`和`ENTRYPOINT`指令之间的区别。接下来，我们将了解构建上下文是什么以及为什么它很重要。最后，我们将介绍实际的镜像构建命令。

如果得到良好维护，普通的集装箱的平均寿命约为 20 年，而 Docker 容器的平均寿命为 2.5 天。- [`www.tintri.com/blog/2017/03/tintri-supports-containers-advanced-storage-features`](https://www.tintri.com/blog/2017/03/tintri-supports-containers-advanced-storage-features)

在本章中，我们将涵盖以下主题：

+   什么是 Dockerfile？

+   Dockerfile 中可以使用的所有指令

+   何时使用`COPY`或`ADD`指令

+   `ENV`和`ARG`变量之间的区别

+   为什么要使用`CMD`和`ENTRYPOINT`指令

+   构建上下文的重要性

+   使用 Dockerfile 构建 Docker 镜像

# 技术要求

您将从 Docker 的公共存储库中拉取 Docker 镜像，因此需要基本的互联网访问权限来执行本章中的示例。

本章的代码文件可以在 GitHub 上找到：

[`github.com/PacktPublishing/Docker-Quick-Start-Guide/tree/master/Chapter03`](https://github.com/PacktPublishing/Docker-Quick-Start-Guide/tree/master/Chapter03)

查看以下视频以查看代码的实际操作：[`bit.ly/2rbHvwC`](http://bit.ly/2rbHvwC)

# 什么是 Dockerfile？

您在第二章中学到，您可以运行 Docker 容器，对正在运行的容器进行修改，然后使用`docker commit`命令保存这些更改，从而有效地创建一个新的 Docker 镜像。尽管这种方法有效，但不是创建 Docker 容器的首选方式。创建 Docker 镜像的最佳方式是使用具有描述所需镜像的 Dockerfile 的 Docker 镜像构建命令。

Dockerfile（是的，正确的拼写是一个词，首字母大写*D*）是一个文本文件，其中包含 Docker 守护程序用来创建 Docker 镜像的指令。指令使用一种键值对语法进行定义。每个指令都在 Dockerfile 中占据一行。虽然 Dockerfile 指令不区分大小写，但有一个常用的约定，即指令单词始终大写。

Dockerfile 中指令的顺序很重要。指令按顺序评估，从 Dockerfile 的顶部开始，直到文件的底部结束。如果您还记得第一章中的内容，Docker 镜像由层组成。Dockerfile 中的所有指令都会导致生成一个新的层，因此在构建 Docker 镜像时，但是，某些指令只会向创建的镜像添加一个大小为零的元数据层。由于最佳实践是尽可能保持 Docker 镜像尽可能小，因此您将希望尽可能高效地使用创建非零字节大小层的指令。在接下来的部分中，我们将注意到使用指令创建非零字节大小层的地方，以及如何最好地使用该指令来最小化层数量和大小。另一个重要的考虑因素是指令的顺序。某些指令必须在其他指令之前使用，但除了这些例外情况，您可以按任何顺序放置其他指令。最佳实践是在 Dockerfile 的早期使用变化最小的指令，在 Dockerfile 的后期使用变化更频繁的指令。原因是当您需要重新构建镜像时，只有在 Dockerfile 中第一行更改的位置或之后的层才会被重新构建。如果您还不理解这一点，不用担心，一旦我们看到一些例子，它就会更有意义。

我们将在本节末尾回顾构建命令，但我们将从 Dockerfile 可用的指令开始，首先是必须是 Dockerfile 中的第一个指令的指令：`FROM`指令。

# FROM 指令

每个 Dockerfile 必须有一个`FROM`指令，并且它必须是文件中的第一个指令。（实际上，`FROM`指令之前可以使用 ARG 指令，但这不是必需的指令。我们将在 ARG 指令部分更多地讨论这个。）

`FROM`指令设置正在创建的镜像的基础，并指示 Docker 守护程序新镜像的基础应该是指定为参数的现有 Docker 镜像。指定的镜像可以使用与我们在第二章中看到的 Docker `container run`命令相同的语法来描述。在这里，它是一个`FROM`指令，指定使用官方的`nginx`镜像，版本为 1.15.2：

```
# Dockerfile
FROM nginx:1.15.2
```

请注意，在这个例子中，没有指定指示指定的镜像是官方 nginx 镜像的存储库。如果没有指定标签，将假定为`latest`标签。

`FROM`指令将创建我们新镜像中的第一层。该层将是指令参数中指定的镜像大小，因此最好指定满足新镜像所需条件的最小镜像。一个特定于应用程序的镜像，比如`nginx`，会比一个操作系统镜像，比如 ubuntu，要小。而`alpine`的操作系统镜像会比其他操作系统的镜像，比如 Ubuntu、CentOS 或 RHEL，要小得多。`FROM`指令可以使用一个特殊的关键字作为参数。它是`scratch`。Scratch 不是一个可以拉取或运行的镜像，它只是向 Docker 守护程序发出信号，表明你想要构建一个带有空基础镜像层的镜像。`FROM scratch`指令被用作许多其他基础镜像的基础层，或者用于专门的应用程序特定镜像。你已经看到了这样一个专门的应用程序镜像的例子：hello-world。hello-world 镜像的完整 Dockerfile 如下：

```
# hello-world Dockerfile
FROM scratch
COPY hello /
CMD ["/hello"]
```

我们将很快讨论`COPY`和`CMD`指令，但是你应该根据它的 Dockerfile 来感受一下 hello-world 镜像有多小。在 Docker 镜像的世界中，越小越好。参考一下一些镜像的大小：

![](img/3e91115f-6c8d-4634-8638-b0c8e051a85b.png)

# 标签指令

LABEL 指令是向 Docker 镜像添加元数据的一种方法。当创建镜像时，此指令会向镜像添加嵌入式键值对。一个镜像可以有多个 LABEL，并且每个 LABEL 指令可以提供一个或多个标签。LABEL 指令最常见的用途是提供有关镜像维护者的信息。这些数据以前有自己的指令。请参阅有关现在已弃用的 MAINTAINER 指令的下面提示框。以下是一些有效的 LABEL 指令示例：

```
# LABEL instruction syntax
# LABEL <key>=<value> <key>=<value> <key>=<value> ...
LABEL maintainer="Earl Waud <earlwaud@mycompany.com>"
LABEL "description"="My development Ubuntu image"
LABEL version="1.0"
LABEL label1="value1" \
 label2="value2" \
 lable3="value3"
LABEL my-multi-line-label="Labels can span \
more than one line in a Dockerfile."
LABEL support-email="support@mycompany.com" support-phone="(123) 456-7890"
```

LABEL 指令是 Dockerfile 中可以多次使用的指令之一。你将会在后面学到，一些可以多次使用的指令只会保留最后一次使用的内容，忽略之前的所有使用。但是 LABEL 指令不同。每次使用 LABEL 指令都会向生成的镜像添加一个额外的标签。然而，如果两次或更多次使用 LABEL 具有相同的键，标签将获得最后一个匹配的 LABEL 指令中提供的值。就像这样：

```
# earlier in the Dockerfile
LABEL version="1.0"
# later in the Dockerfile...
LABEL version="2.0"
# The Docker image metadata will show version="2.0"
```

重要的是要知道，在你的 FROM 指令中指定的基础镜像可能包含使用 LABEL 指令创建的标签，并且它们将自动包含在你正在构建的镜像的元数据中。如果你的 Dockerfile 中的 LABEL 指令使用与 FROM 镜像的 Dockerfile 中使用的 LABEL 指令相同的键，你（后来的）值将覆盖 FROM 镜像中的值。你可以使用 inspect 命令查看镜像的所有标签：

![](img/c01d1aab-1110-46c5-9623-cad509812cc6.png)

MAINTAINER 指令有一个专门用于提供有关镜像维护者信息的 Dockerfile 指令，但是这个指令已经被弃用。不过，你可能会在某个时候看到它在 Dockerfile 中被使用。语法如下：`"maintainer": "Earl Waud <earlwaud@mycompany.com>"`。

# COPY 指令

你已经在“FROM 指令”部分的 hello-world Dockerfile 中看到了使用 COPY 指令的示例。COPY 指令用于将文件和文件夹复制到正在构建的 Docker 镜像中。COPY 指令的语法如下：

```
# COPY instruction syntax
COPY [--chown=<user>:<group>] <src>... <dest>
# Use double quotes for paths containing whitespace)
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

请注意，--chown 参数仅适用于基于 Linux 的容器。如果没有--chown 参数，所有者 ID 和组 ID 都将设置为 0。

`<src>`或源是文件名或文件夹路径，并且被解释为相对于构建的上下文。我们稍后会在本章中更多地讨论构建上下文，但现在，将其视为构建命令运行的位置。源可能包括通配符。

`<dest>`或目标是正在创建的图像中的文件名或路径。目标是相对于图像文件系统的根目录，除非有一个前置的`WORKDIR`指令。我们稍后会讨论`WORKDIR`指令，但现在，只需将其视为设置当前工作目录的一种方式。当`COPY`命令在 Dockerfile 中的`WORKDIR`指令之后出现时，复制到图像中的文件或文件夹将被放置在相对于当前工作目录的目标中。如果目标包括一个或多个文件夹的路径，如果它们不存在，所有文件夹都将被创建。

在我们之前的 hello-world Dockerfile 示例中，您看到了一个`COPY`指令，它将一个名为`hello`的可执行文件复制到图像的文件系统根位置。它看起来像这样：`COPY hello /`。这是一个基本的`COPY`指令。以下是一些其他示例：

```
# COPY instruction Dockerfile for Docker Quick Start
FROM alpine:latest
LABEL maintainer="Earl Waud <earlwaud@mycompany.com>"
LABEL version=1.0
# copy multiple files, creating the path "/theqsg/files" in the process
COPY file* theqsg/files/
# copy all of the contents of folder "folder1" to "/theqsg/" 
# (but not the folder "folder1" itself)
COPY folder1 theqsg/
# change the current working directory in the image to "/theqsg"
WORKDIR theqsg
# copy the file special1 into "/theqsg/special-files/"
COPY --chown=35:35 special1 special-files/
# return the current working directory to "/"
WORKDIR /
CMD ["sh"]
```

通过从图像运行容器并执行`ls`命令，我们可以看到使用前面的 Dockerfile 得到的图像文件系统会是什么样子：

![](img/442d213e-eea8-4ca6-bee4-253d64e84aa5.png)

您可以看到在目标路径中指定的文件夹在复制期间被创建。您还会注意到提供`--chown`参数会设置目标文件的所有者和组。一个重要的区别是当源是一个文件夹时，文件夹的内容会被复制，但文件夹本身不会被复制。请注意，使用`WORKDIR`指令会更改图像文件系统中的路径，并且随后的`COPY`指令现在将相对于新的当前工作目录。在这个例子中，我们将当前工作目录返回到`/`，以便在容器中执行的命令将相对于`/`运行。

# ADD 指令

`ADD`指令用于将文件和文件夹复制到正在构建的 Docker 图像中。`ADD`指令的语法如下：

```
# ADD instruction syntax
ADD [--chown=<user>:<group>] <src>... <dest>
# Use double quotes for paths containing whitespace)
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

现在，您可能会认为`ADD`指令似乎就像我们刚刚审查的`COPY`指令一样。嗯，你没错。基本上，我们看到`COPY`指令所做的所有事情，`ADD`指令也可以做。它使用与`COPY`指令相同的语法，两者之间的`WORKDIR`指令的效果也是相同的。那么，为什么我们有两个执行相同操作的命令呢？

# COPY 和 ADD 之间的区别

答案是`ADD`指令实际上可以比`COPY`指令做更多。更多取决于用于源输入的值。使用`COPY`指令时，源可以是文件或文件夹。然而，使用`ADD`指令时，源可以是文件、文件夹、本地`.tar`文件或 URL。

当`ADD`指令的源值是`.tar`文件时，该 TAR 文件的内容将被提取到镜像中的相应文件夹中。

当您在`ADD`指令中使用`.tar`文件作为源并包括`--chown`参数时，您可能期望在从存档中提取的文件上设置图像中的所有者和组。目前情况并非如此。不幸的是，尽管使用了`--chown`参数，提取内容的所有者、组和权限将与存档中包含的内容相匹配。当您使用`.tar`文件时，您可能希望在 ADD 之后包含`RUN chown -R X:X`。

如前所述，`ADD`指令可以使用 URL 作为源值。以下是一个包含使用 URL 的`ADD`指令的示例 Dockerfile：

```
# ADD instruction Dockerfile for Docker Quick Start
FROM alpine
LABEL maintainer="Earl Waud <earlwaud@mycompany.com>"
LABEL version=3.0
ADD https://github.com/docker-library/hello-world/raw/master/amd64/hello-world/hello /
RUN chmod +x /hello
CMD ["/hello"]
```

在`ADD`指令中使用 URL 是有效的，将文件下载到镜像中，但是这个功能并不被 Docker 推荐。以下是 Docker 文档对使用`ADD`的建议：

![](img/f457493e-9d04-4f9f-8892-4eb6466b159d.png)

因此，一般来说，每当您可以使用`COPY`指令将所需内容放入镜像时，您应该选择使用`COPY`而不是`ADD`。

# ENV 指令

正如您可能猜到的那样，`ENV`指令用于定义将在从正在构建的镜像创建的运行容器中设置的环境变量。使用典型的键值对定义变量。Dockerfile 可以有一个或多个`ENV`指令。以下是`ENV`指令的语法：

```
# ENV instruction syntax
# This is the form to create a single environment variable per instruction
# Everything after the space following the <key> becomes the value
ENV <key> <value>
# This is the form to use when you want to create more than one variable per instruction
ENV <key>=<value> ...
```

每个`ENV`指令将创建一个或多个环境变量（除非键名重复）。让我们看一下 Dockerfile 中的一些`ENV`指令：

```
# ENV instruction Dockerfile for Docker Quick Start
FROM alpine
LABEL maintainer="Earl Waud <earlwaud@mycompany.com>"
ENV appDescription This app is a sample of using ENV instructions
ENV appName=env-demo
ENV note1="The First Note First" note2=The\ Second\ Note\ Second \
note3="The Third Note Third"
ENV changeMe="Old Value"
CMD ["sh"]
```

使用此 Dockerfile 构建镜像后，您可以检查镜像元数据，并查看已创建的环境变量：

![](img/e98f6d2e-906f-4c99-8a98-7fd5aceb10fc.png)

环境变量可以在运行容器时使用`--env`参数进行设置（或覆盖）。在这里，我们看到了这个功能的实际应用：

![](img/572af022-9c53-496a-96e1-7e1c23a14b24.png)

重要的是要知道，使用`ENV`指令会在生成的镜像中创建一个大小为零字节的额外层。如果要向镜像添加多个环境变量，并且可以使用支持一次设置多个变量的指令形式，那么只会创建一个额外的镜像层，因此这是一个好方法。

# ARG 指令

有时在构建 Docker 镜像时，您可能需要使用变量数据来自定义构建。`ARG`指令是处理这种情况的工具。要使用它，您需要将`ARG`指令添加到 Dockerfile 中，然后在执行构建命令时，通过`--build-arg`参数传入变量数据。`--build-arg`参数使用现在熟悉的键值对格式：

```
# The ARG instruction syntax
ARG <varname>[=<default value>]

# The build-arg parameter syntax
docker image build --build-arg <varname>[=<value>] ...
```

您可以在 Dockerfile 中使用多个`ARG`指令，并在 docker image build 命令上使用相应的`--build-arg`参数。对于每个`--build-arg`参数的使用，都必须包括一个`ARG`指令。如果没有`ARG`指令，则在构建过程中`--build-arg`参数将不会被设置，并且您将收到警告消息。如果您没有提供`--build-arg`参数，或者没有为现有的`ARG`指令提供`--build-arg`参数的值部分，并且该`ARG`指令包括默认值，那么变量将被分配默认值。

请注意，在镜像构建过程中，即使`--build-arg`被包括为 docker image build 命令的参数，相应的变量也不会在 Dockerfile 中的`ARG`指令到达之前设置。换句话说，`--build-arg`参数的键值对的值在其对应的`ARG`行之后才会被设置。

在 `ARG` 指令中定义的参数不会持续到从创建的镜像运行的容器中，但是 ARG 指令会在生成的镜像中创建新的零字节大小的层。以下是使用 `ARG` 指令的教育示例：

```
# ARG instruction Dockerfile for Docker Quick Start
FROM alpine
LABEL maintainer="Earl Waud <earlwaud@mycompany.com>"

ENV key1="ENV is stronger than an ARG"
RUN echo ${key1}
ARG key1="not going to matter"
RUN echo ${key1}

RUN echo ${key2}
ARG key2="defaultValue"
RUN echo ${key2}
ENV key2="ENV value takes over"
RUN echo ${key2}
CMD ["sh"]
```

创建一个包含上述代码块中显示的内容的 Dockerfile，并运行以下构建命令，以查看 `ENV` 和 `ARG` 指令的范围如何发挥作用：

```
# Build the image and look at the output from the echo commands
 docker image build --rm \
 --build-arg key1="buildTimeValue" \
 --build-arg key2="good till env instruction" \
 --tag arg-demo:2.0 .
```

第一个 `echo ${key1}` 会让你看到，即使有一个 `--build-arg` 参数用于 `key1`，它也不会被存储为 `key1`，因为有一个相同键名的 `ENV` 指令。这对于第二个 `echo ${key1}` 仍然成立，这是在 ARG `key1` 指令之后。当 `ARG` 和 `EVN` 指令具有相同的键名时，ENV 变量值总是获胜。

然后，你会看到第一个 `echo ${key2}` 是空的，即使有一个 `--build-arg` 参数。这是因为我们还没有达到 `ARG key2` 指令。第二个 `echo ${key2}` 将包含相应 `--build-arg` 参数的值，即使在 `ARG key2` 指令中提供了默认值。最终的 `echo ${key2}` 将显示在 `ENV key2` 指令中提供的值，尽管在 `ARG` 中有默认值，并且通过 `--build-arg` 参数传递了一个值。同样，这是因为 `ENV` 总是胜过 ARG。

# ENV 和 ARG 之间的区别

这是一对具有类似功能的指令。它们都可以在构建镜像时使用，设置参数以便在其他 Dockerfile 指令中使用。可以使用这些参数的其他 Dockerfile 指令包括 `FROM`、`LABEL`、`COPY`、`ADD`、`ENV`、`USER`、`WORKDIR`、`RUN`、`VOLUME`、`EXPOSE`、`STOPSIGNAL` 和 `ONBUILD`。以下是在其他 Docker 命令中使用 `ARG` 和 `ENV` 变量的示例：

```
# ENV vs ARG instruction Dockerfile for Docker Quick Start
FROM alpine
LABEL maintainer="Earl Waud <earlwaud@mycompany.com>"
ENV lifecycle="production"
RUN echo ${lifecycle}
ARG username="35"
RUN echo ${username}
ARG appdir
RUN echo ${appdir}
ADD hello /${appdir}/
RUN chown -R ${username}:${username} ${appdir}
WORKDIR ${appdir}
USER ${username}
CMD ["./hello"]
```

使用这个 Dockerfile，你会想为 `appdir` `ARG` 指令提供 `--build-arg` 参数，并且在构建命令中提供用户名（如果你想要覆盖默认值）。你也可以在运行时提供一个 `--env` 参数来覆盖生命周期变量。以下是可能使用的构建和运行命令：

```
# Build the arg3 demo image
docker image build --rm \
 --build-arg appdir="/opt/hello" \
 --tag arg-demo:3.0 .

# Run the arg3 demo container
docker container run --rm --env lifecycle="test" arg-demo:3.0
```

虽然 `ENV` 和 `ARG` 指令可能看起来相似，但它们实际上是非常不同的。以下是记住 `ENV` 和 `ARG` 指令创建的参数之间的关键区别：

+   ENV 持续存在于运行中的容器中，ARG 不会。

+   ARG 使用相应的构建参数，ENV 不使用。

+   `ENV`指令必须包括键和值，`ARG`指令有一个键，但（默认）值是可选的。

+   ENV 比 ARG 更重要。

永远不要使用`ENV`或`ARG`指令向构建命令或生成的容器提供秘密数据，因为这些值对于运行 docker history 命令的任何用户都是明文可见的。

# USER 指令

USER 指令允许您为 Dockerfile 中接下来的所有指令和从构建图像运行的容器设置当前用户（和组）。`USER`指令的语法如下：

```
# User instruction syntax
USER <user>[:<group>] or
USER <UID>[:<GID>]
```

如果将命名用户（或组）作为`USER`指令的参数提供，则该用户（和组）必须已经存在于系统的 passwd 文件（或组文件）中，否则将发生构建错误。如果将`UID`（或`GID`）作为`USER`命令的参数提供，则不会执行检查用户（或组）是否存在。考虑以下 Dockerfile：

```
# USER instruction Dockerfile for Docker Quick Start 
FROM alpine
LABEL maintainer="Earl Waud <earl@mycompany.com>"
RUN id
USER games:games
run id
CMD ["sh"]
```

当图像构建开始时，当前用户是 root 或`UID=0` `GID=0`。然后，执行`USER`指令将当前用户和组设置为`games:games`。由于这是 Dockerfile 中`USER`指令的最后一次使用，所有使用构建图像运行的容器将具有当前用户（和组）设置为 games。构建和运行如下所示：

![](img/0aca2a20-ee07-4b3b-ba94-bc58b78b8deb.png)

请注意，步骤 3/6 的 RUN id 的输出显示当前用户为 root，然后在步骤 5/6（在`USER`指令之后）中显示当前用户为 games。最后，请注意，从图像运行的容器具有当前用户 games。`USER`指令在图像中创建了一个大小为零字节的层。

# WORKDIR 指令

我们已经在一些示例中看到了`WORKDIR`指令的使用，用于演示其他指令。它有点像 Linux 的`cd`和`mkdir`命令的组合。`WORKDIR`指令将把图像中的当前工作目录更改为指令中提供的值。如果参数中路径的任何部分尚不存在，则将作为执行指令的一部分创建它。`WORKDIR`指令的语法如下：

```
# WORKDIR instruction syntax
WORKDIR instruction syntax
WORKDIR /path/to/workdir
```

`WORKDIR`指令可以使用`ENV`或`ARG`参数值作为其参数的全部或部分。Dockerfile 可以有多个`WORKDIR`指令，每个后续的`WORKDIR`指令将相对于前一个（如果使用相对路径）。以下是演示此可能性的示例：

```
# WORKDIR instruction Dockerfile for Docker Quick Start
FROM alpine
# Absolute path...
WORKDIR /
# relative path, relative to previous WORKDIR instruction
# creates new folder
WORKDIR sub-folder-level-1
RUN touch file1.txt
# relative path, relative to previous WORKDIR instruction
# creates new folder
WORKDIR sub-folder-level-2
RUN touch file2.txt
# relative path, relative to previous WORKDIR instruction
# creates new folder
WORKDIR sub-folder-level-3
RUN touch file3.txt
# Absolute path, creates three sub folders...
WORKDIR /l1/l2/l3
CMD ["sh"]
```

从这个 Dockerfile 构建镜像将导致镜像具有三层嵌套的文件夹。从镜像运行容器并列出文件和文件夹将如下所示：

![](img/c1c550ed-5793-45d4-a4d7-a9f8c4c60f81.png)

`WORKDIR`指令将在生成的镜像中创建一个大小为零字节的层。

# VOLUME 指令

您应该记住，Docker 镜像由一系列相互叠加的只读层组成，当您从 Docker 镜像运行容器时，它会创建一个新的读写层，您可以将其视为位于只读层之上。所有对容器的更改都应用于读写层。如果对只读层中的文件进行更改，将会创建该文件的副本并将其添加到读写层。然后，所有更改都将应用于该副本。该副本隐藏了只读层中找到的版本，因此从运行的容器的角度来看，文件只有一个版本，即已更改的版本。这大致是统一文件系统的工作原理。

这实际上是一件好事。但是，它也带来了一个挑战，即当运行的容器退出并被删除时，所有更改也将被删除。这通常是可以接受的，直到您希望在容器的生命周期之后保留一些数据，或者希望在容器之间共享数据时。Docker 有一条指令可以帮助您解决这个问题，那就是`VOLUME`指令。

`VOLUME`指令将创建一个存储位置，该位置位于美国文件系统之外，并且通过这样做，允许存储在容器的生命周期之外持久存在。以下是`VOLUME`指令的语法：

```
# VOLUME instruction syntax
VOLUME ["/data"]
# or for creating multiple volumes with a single instruction
VOLUME /var/log /var/db /moreData
```

创建卷的其他方法是向 docker `container run`命令添加卷参数，或者使用 docker volume create 命令。我们将在第四章 *Docker Volumes*中详细介绍这些方法。

这是一个简单的示例 Dockerfile。它在`/myvol`创建了一个卷，其中将有一个名为`greeting`的文件：

```
# VOLUME instruction Dockerfile for Docker Quick Start
FROM alpine
RUN mkdir /myvol
RUN echo "hello world" > /myvol/greeting
VOLUME /myvol
CMD ["sh"]
```

基于从此 Dockerfile 创建的镜像运行容器将在主机系统上创建一个挂载点，最初包含`greeting`文件。当容器退出时，挂载点将保留。在运行具有要持久保存的挂载点的容器时，使用`--rm`参数要小心。使用`--rm`，没有其他卷参数，将导致容器退出时清理挂载点。看起来是这样的：

![](img/b5f7a39c-bbc2-4ed6-96a3-96327e0839af.png)

我们开始时没有卷。然后，我们以分离模式运行了一个基于前面的 Dockerfile 创建的镜像的容器。我们再次检查卷，看到了通过运行容器创建的卷。然后，我们停止容器并再次检查卷，现在卷已经消失了。通常，使用`VOLUME`指令的目的是在容器消失后保留挂载点中的数据。因此，如果您要在运行容器时使用`--rm`，您应该包括`--mount`运行参数，我们将在第四章 *Docker Volumes*中详细介绍。

您可以使用卷的挂载点与主机上的数据进行交互。以下是一个演示这一点的示例：

![](img/bd92fd61-ea21-4375-850c-4e759713d89e.png)

在这个演示中，我们运行了一个基于前面的 Dockerfile 创建的镜像的容器。然后，我们列出了卷，并查看了 myvolsrc 卷（我们已经知道了名称，因为我们在运行命令中提供了它，但您可以使用`ls`命令来查找您可能不知道的卷名称）。使用卷的名称，我们检查卷以找到它在主机上的挂载点。为了验证容器中卷的内容，我们使用 exec 命令来列出文件夹。接下来，使用挂载点路径，我们使用 touch 命令创建一个新文件。最后，我们使用相同的 exec 命令，并看到容器内的卷已经改变（来自容器外的操作）。同样，如果容器更改卷的内容，这些更改将立即反映在主机挂载点上。

前面的示例在 OS X 上直接显示不起作用。它需要一些额外的工作。不过不要惊慌！我们将向您展示如何处理 OS X 所需的额外工作，在第四章 *Docker Volumes*中。

使用`VOLUME`指令既强大又危险。它之所以强大，是因为它让您拥有超出容器生命周期的数据。它之所以危险，是因为数据会立即从容器传递到主机，如果容器被攻击，可能会带来麻烦。出于安全考虑，最佳实践是*不*在 Dockerfile 中包含基于主机的 VOLUME 挂载。我们将在第四章中介绍一些更安全的替代方法，*Docker Volumes*。

`VOLUME`指令将在生成的 Docker 镜像中添加一个大小为零字节的层。

# EXPOSE 指令

`EXPOSE`指令是记录镜像期望在使用 Dockerfile 构建的镜像运行容器时打开的网络端口的一种方式。`EXPOSE`指令的语法如下：

```
# EXPOSE instruction syntax
EXPOSE <port> [<port>/<protocol>...]
```

重要的是要理解，在 Dockerfile 中包含`EXPOSE`指令实际上并不会在容器中打开网络端口。当从具有`EXPOSE`指令的 Dockerfile 中的镜像运行容器时，仍然需要包括`-p`或`-P`参数来实际打开网络端口到容器。

根据需要在 Dockerfile 中包含多个`EXPOSE`指令。在运行时包括`-P`参数是一种快捷方式，可以自动打开 Dockerfile 中包含的所有`EXPOSE`指令的端口。在运行命令时使用`-P`参数时，相应的主机端口将被随机分配。

将`EXPOSE`指令视为镜像开发者向您传达的信息，告诉您在运行容器时，镜像中的应用程序期望您打开指定的端口。`EXPOSE`指令在生成的镜像中创建一个大小为零字节的层。

# RUN 指令

`RUN`指令是 Dockerfile 的真正工作马。这是您对生成的 Docker 镜像产生最大变化的工具。基本上，它允许您在镜像中执行任何命令。`RUN`指令有两种形式。以下是语法：

```
# RUN instruction syntax
# Shell form to run the command in a shell
# For Linux the default is "/bin/sh -c"
# For Windows the default is "cmd /S /C"
RUN <command>

# Exec form
RUN ["executable", "param1", "param2"]
```

每个`RUN`指令在镜像中创建一个新的层，随后的每个指令的层都将建立在`RUN`指令的层的结果之上。除非使用`SHELL`指令覆盖，默认情况下，shell 形式的指令将使用默认 shell。如果您正在构建一个不包含 shell 的容器，您将需要使用`RUN`指令的 exec 形式。您还可以使用 exec 形式的指令来使用不同的 shell。例如，要使用 bash shell 运行命令，您可以添加一个`RUN`指令，如下所示：

```
# Exec form of RUN instruction using bash
RUN ["/bin/bash", "-c", "echo hello world > /myvol/greeting"]
```

`RUN`命令的用途仅受想象力的限制，因此提供`RUN`指令示例的详尽列表是不可能的，但以下是一些使用两种形式的指令的示例，只是为了给您一些想法：

```
# RUN instruction Dockerfile for Docker Quick Start
FROM ubuntu
RUN useradd --create-home -m -s /bin/bash dev
RUN mkdir /myvol
RUN echo "hello DQS Guide" > /myvol/greeting
RUN ["chmod", "664", "/myvol/greeting"]
RUN ["chown", "dev:dev", "/myvol/greeting"]
VOLUME /myvol
USER dev
CMD ["/bin/bash"]
```

当您知道您的镜像将包含 bash 时，可以添加一个有趣且有用的`RUN`指令。这个想法是我在 Dockercon 16 上得知的，由我的同事*Marcello de Sales*与我分享。您可以使用以下代码在 shell 进入容器时创建自定义提示。如果您不喜欢鲸鱼图形，可以更改并使用任何您喜欢的东西。我包括了一些我喜欢的选项。以下是代码：

```
# RUN instruction Dockerfile for Docker Quick Start
FROM ubuntu
RUN useradd --create-home -m -s /bin/bash dev
# Add a fun prompt for dev user of my-app
# whale: "\xF0\x9F\x90\xB3"
# alien:"\xF0\x9F\x91\xBD"
# fish:"\xF0\x9F\x90\xA0"
# elephant:"\xF0\x9F\x91\xBD"
# moneybag:"\xF0\x9F\x92\xB0"
RUN echo 'PS1="\[$(tput bold)$(tput setaf 4)\]my-app $(echo -e "\xF0\x9F\x90\xB3") \[$(tput sgr0)\] [\\u@\\h]:\\W \\$ "' >> /home/dev/.bashrc && \
 echo 'alias ls="ls --color=auto"' >> /home/dev/.bashrc
USER dev
CMD ["/bin/bash"]
```

生成的提示如下：

![](img/2d0c1502-1539-4012-9e25-d472c87bea18.png)

# `CMD`指令

`CMD`指令用于定义从使用其 Dockerfile 构建的镜像运行容器时采取的默认操作。虽然在 Dockerfile 中可以包含多个`CMD`指令，但只有最后一个才会有意义。基本上，最后一个`CMD`指令为镜像提供了默认操作。这允许您在 Dockerfile 的`FROM`指令中覆盖或使用镜像中的`CMD`。以下是一个示例，其中一个微不足道的 Dockerfile 不包含`CMD`指令，并依赖于在`FROM`指令中使用的 ubuntu 镜像中找到的指令：

![](img/2c1f6b2e-a069-432c-a891-fb4eb69f7bab.png)

您可以从 history 命令的输出中看到，ubuntu 镜像包括`CMD ["/bin/bash"]`指令。您还会看到我们的 Dockerfile 没有自己的`CMD`指令。当我们运行容器时，默认操作是运行`"/bin/bash"`。

`CMD`指令有三种形式。第一种是 shell 形式。第二种是 exec 形式，这是最佳实践形式。第三种是特殊的 exec 形式，它有两个参数，并且与`ENTRYPOINT`指令一起使用，我们将在*ENTRYPOINT 指令*部分讨论它。以下是`CMD`指令的语法。

```
# CMD instruction syntax
CMD command param1 param2 (shell form)
CMD ["executable","param1","param2"] (exec form)
CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
```

以下是一些`CMD`指令的示例供您参考：

```
# CMD instruction examples
CMD ["/bin/bash"]
CMD while true; do echo 'DQS Expose Demo' | nc -l -p 80; done
CMD echo "How many words are in this echo command" | wc -
CMD tail -f /dev/null
CMD ["-latr", "/var/opt"]
```

与`RUN`指令一样，`CMD`指令的 shell 形式默认使用`["/bin/sh", "-c"]` shell 命令（或`["cmd", "/S", "/C"]`用于 Windows），除非它被`SHELL`指令覆盖。然而，与`RUN`指令不同，`CMD`指令在构建镜像时不执行任何操作，而是在从镜像构建的容器运行时执行。如果正在构建的容器镜像没有 shell，则可以使用指令的 exec 形式，因为它不会调用 shell。`CMD`指令向镜像添加了一个大小为零字节的层。

# ENTRYPOINT 指令

`ENTRYPOINT`指令用于配置 docker 镜像以像应用程序或命令一样运行。例如，我们可以使用`ENTRYPOINT`指令制作一个显示`curl`命令帮助信息的镜像。考虑这个 Dockerfile：

```
# ENTRYPOINT instruction Dockerfile for Docker Quick Start
FROM alpine
RUN apk add curl
ENTRYPOINT ["curl"]
CMD ["--help"]
```

我们可以运行容器镜像，不覆盖`CMD`参数，它将显示`curl`命令的帮助信息。然而，当我们用`CMD`覆盖参数运行容器时，在这种情况下是一个 URL，响应将是`curl`该 URL。看一下：

![](img/71810ef5-9fe0-439d-963f-7a03c6b758df.png)

当为具有`ENTRYPOINT`指令的 exec 形式的容器提供运行参数时，这些参数将附加到`ENTRYPOINT`指令，覆盖`CMD`指令中提供的任何内容。在这个例子中，`--help`被`google.com`运行参数覆盖，所以结果指令是`curl google.com`。以下是`ENTRYPOINT`指令的实际语法：

```
# ENTRYPOINT instruction syntax
ENTRYPOINT command param1 param2 (shell form)
ENTRYPOINT ["executable", "param1", "param2"] (exec form, best practice)
```

与`CMD`指令一样，只有最后一个`ENTRYPOINT`指令是重要的。同样，这允许您在使用`FROM`镜像时使用或覆盖`ENTRYPOINT`指令。与`RUN`和`CMD`指令一样，使用 shell 形式将调用`["/bin/sh", "-c"]`（或在 Windows 上为`["cmd", "/S", "/C"]`）。当使用指令的 exec 形式时，情况并非如此。这对于没有 shell 或 shell 不可用于活动用户上下文的镜像非常重要。但是，您将不会获得 shell 处理，因此在使用指令的 exec 形式时，任何 shell 环境变量都不会被替换。通常最好尽可能使用`ENTRYPOINT`指令的 exec 形式。

# CMD 和 ENTRYPOINT 之间的区别

在这里，我们再次有两个表面上看起来非常相似的指令。事实上，它们之间确实有一些功能重叠。这两个指令都提供了一种定义在运行容器时执行的默认应用程序的方法。然而，它们各自有其独特的目的，并且在某些情况下共同工作，以提供比任何一条指令单独提供的更大的功能。

最佳实践是在希望容器作为应用程序执行、提供特定（开发者）定义的功能时使用`ENTRYPOINT`指令，并在希望为用户提供更多灵活性以确定容器将提供的功能时使用`CMD`。

这两个指令都有两种形式：shell 形式和 exec 形式。最佳实践是尽可能使用任何一种的 exec 形式。原因是，shell 形式将运行`["/bin/sh", "-c"]`（或在 Windows 上为`["cmd", "/S", "/C"]`）来启动指令参数中的应用程序。由于这个原因，运行在容器中的主要进程不是应用程序，而是 shell。这会影响容器的退出方式，影响信号的处理方式，并且可能会对不包括`"/bin/sh"`的镜像造成问题。您可能需要使用 shell 形式的一个用例是如果您需要 shell 环境变量替换。

在 Dockerfile 中还有一个使用两个指令的用例。当您同时使用两者时，可以定义在运行容器时执行的特定应用程序，并允许用户轻松提供与定义的应用程序一起使用的参数。在这种情况下，您将使用`ENTRYPOINT`指令设置要执行的应用程序，并使用`CMD`指令为应用程序提供一组默认参数。通过这种配置，容器的用户可以从`CMD`指令中提供的默认参数中受益，或者他们可以通过在`container run`命令中提供参数作为参数轻松覆盖应用程序中使用的这些参数。强烈建议在同时使用两个指令时使用它们的 exec 形式。

# `HEALTHCHECK`指令

`HEALTHCHECK`指令是 Dockerfile 中相对较新的添加，用于定义在容器内运行的命令，以测试容器的应用程序健康状况。当容器具有`HEALTHCHECK`时，它会获得一个特殊的状态变量。最初，该变量将被设置为`starting`。每当成功执行`HEALTHCHECK`时，状态将被设置为`healthy`。当执行`HEALTHCHECK`并失败时，失败计数值将被递增，然后与重试值进行比较。如果失败计数等于或超过重试值，则状态将被设置为`unhealthy`。`HEALTHCHECK`指令的语法如下：

```
# HEALTHCHECK instruction syntax
HEALTHCHECK [OPTIONS] CMD command (check container health by running a command inside the container)
HEALTHCHECK NONE (disable any HEALTHCHECK inherited from the base image)
```

在设置`HEALTHCHECK`时有四个选项可用，这些选项如下：

```
# HEALTHCHECK CMD options
--interval=DURATION (default: 30s)
--timeout=DURATION (default: 30s)
--start-period=DURATION (default: 0s)
--retries=N (default: 3)
```

`--interval`选项允许您定义`HEALTHCHECK`测试之间的时间间隔。`--timeout`选项允许您定义被视为`HEALTHCHECK`测试时间过长的时间量。如果超过超时时间，测试将自动视为失败。`--start-period`选项允许在容器启动期间定义一个无失败时间段。最后，`--retries`选项允许您定义多少连续失败才能将`HEALTHCHECK`状态更新为`unhealthy`。

`HEALTHCHECK`指令的`CMD`部分遵循与`CMD`指令相同的规则。有关`CMD`指令的完整详情，请参阅前面的部分。使用的`CMD`在退出时将提供一个状态，该状态要么是成功的 0，要么是失败的 1。以下是使用`HEALTHCHECK`指令的 Dockerfile 示例：

```
# HEALTHCHECK instruction Dockerfile for Docker Quick Start
FROM alpine
RUN apk add curl
EXPOSE 80/tcp
HEALTHCHECK --interval=30s --timeout=3s \
 CMD curl -f http://localhost/ || exit 1
CMD while true; do echo 'DQS Expose Demo' | nc -l -p 80; done
```

使用上述 Dockerfile 构建的镜像运行容器如下：

![](img/5a4d6561-c20e-4f21-94d0-0de027616a41.png)

您可以看到`HEALTHCHECK`最初报告状态为`starting`，但一旦`HEALTHCHECK` `CMD`报告成功，状态就会更新为`healthy`。

# ONBUILD 指令

`ONBUILD`指令是在创建将成为另一个 Dockerfile 中`FROM`指令参数的镜像时使用的工具。`ONBUILD`指令只是向您的镜像添加元数据，具体来说是存储在镜像中而不被其他方式使用的触发器。然而，当您的镜像作为另一个 Dockerfile 中`FROM`命令的参数提供时，该元数据触发器会被使用。以下是`ONBUILD`指令的语法：

```
# ONBUILD instruction syntax
ONBUILD [INSTRUCTION]
```

`ONBUILD`指令有点像 Docker 时间机器，用于将指令发送到未来。（如果您知道我刚刚输入*Doctor time machine*多少次，您可能会笑！）让我们用一个简单的例子来演示`ONBUILD`指令的使用。首先，我们将使用以下 Dockerfile 构建一个名为`my-base`的镜像：

```
# my-base Dockerfile
FROM alpine
LABEL maintainer="Earl Waud <earlwaud@mycompany.com>"
ONBUILD LABEL version="1.0"
ONBUILD LABEL support-email="support@mycompany.com" support-phone="(123) 456-7890"
CMD ["sh"]
```

接下来，让我们构建一个名为`my-app`的镜像，该镜像是从`my-base`镜像构建的，如下所示：

```
# my-app Dockerfile
FROM my-base:1.0
CMD ["sh"]
```

检查生成的`my-app`镜像，我们可以看到`ONBUILD`指令中提供的 LABEL 命令被发送到未来，到达`my-app`镜像：

![](img/a1594a9a-5813-48bd-ae3e-0c40d9ed65b3.png)

如果您对`my-base`镜像进行类似的检查，您会发现它*不*包含版本和支持标签。还要注意，`ONBUILD`指令是一次性使用的时间机器。如果您使用`FROM`指令中的`my-app`构建一个新的镜像，新的镜像将*不*获得`my-base`镜像的 ONBUILD 指令中提供的标签。

# STOPSIGNAL 指令

`STOPSIGNAL`指令用于设置系统调用信号，该信号将被发送到容器，告诉它退出。指令中使用的参数可以是无符号数字，等于内核系统调用表中的位置，也可以是大写的实际信号名称。以下是该指令的语法：

```
# STOPSIGNAL instruction syntax
STOPSIGNAL signal
```

`STOPSIGNAL`指令的示例包括以下内容：

```
# Sample STOPSIGNAL instruction using a position number in the syscall table
STOPSIGNAL 9
# or using a signal name
STOPSIGNAL SIGQUIT
```

`STOPSIGNAL`指令提供的参数在发出`docker container stop`命令时使用。请记住，使用`ENTRYPOINT`和/或`CMD`指令的执行形式非常重要，以便应用程序成为 PID 1，并直接接收信号。以下是有关在 Docker 中使用信号的出色博客文章链接：[`medium.com/@gchudnov/trapping-signals-in-docker-containers-7a57fdda7d86`](https://medium.com/@gchudnov/trapping-signals-in-docker-containers-7a57fdda7d86)。该文章提供了使用 node.js 应用程序处理信号的出色示例，包括代码和 Dockerfile。

# `SHELL`指令

正如您在本章的许多部分中所阅读的，有几个指令有两种形式，即执行形式或 shell 形式。如前所述，所有 shell 形式默认使用`["/bin/sh", "-c"]`用于 Linux 容器，以及`["cmd", "/S", "/C"]`用于 Windows 容器。`SHELL`指令允许您更改该默认设置。以下是`SHELL`指令的语法：

```
# SHELL instruction syntax
SHELL ["executable", "parameters"]
```

`SHELL`指令可以在 Dockerfile 中使用多次。所有使用 shell 的指令，并且在`SHELL`指令之后，将使用新的 shell。因此，根据需要可以在单个 Dockerfile 中多次更改 shell。在创建 Windows 容器时，这可能特别有用，因为它允许您在`cmd.exe`和`powershell.exe`之间来回切换。

# Docker 镜像构建命令

好的，镜像构建命令不是 Dockerfile 指令。相反，它是用于将 Dockerfile 转换为 docker 镜像的 docker 命令。Docker 镜像构建命令将 docker 构建上下文，包括 Dockerfile，发送到 docker 守护程序，它解析 Dockerfile 并逐层构建镜像。我们将很快讨论构建上下文，但现在可以将其视为根据 Dockerfile 中的内容构建 Docker 镜像所需的一切。构建命令的语法如下：

```
# Docker image build command syntax
Usage: docker image build [OPTIONS] PATH | URL | -
```

图像构建命令有许多选项。我们现在不会涵盖所有选项，但让我们看一下一些最常见的选项：

```
# Common options used with the image build command
--rm         Remove intermediate containers after a successful build
--build-arg  Set build-time variables
--tag        Name and optionally a tag in the 'name:tag' format
--file       Name of the Dockerfile (Default is 'PATH/Dockerfile')
```

Docker 守护程序通过从 Dockerfile 中的每个命令创建新的图像来构建图像。每个新图像都是在前一个图像的基础上构建的。使用可选的`--rm`参数将指示守护程序在构建成功完成时删除所有中间图像。当重新构建成功构建的图像时，使用此选项将减慢构建过程，但会保持本地图像缓存的清洁。

当我们讨论`ARG`指令时，我们已经谈到了构建参数。请记住，`--build-arg`选项是您如何为 Dockerfile 中的`ARG`指令提供值。

`--tag`选项允许您为图像指定一个更易读的名称和版本。我们在之前的几个示例中也看到了这个选项的使用。

`--file`选项允许您使用文件名而不是 Dockerfile，并将 Dockerfile 保留在构建上下文文件夹之外的路径中。

以下是一些图像构建命令供参考：

```
# build command samples
docker image build --rm --build-arg username=35 --tag arg-demo:2.0 .
docker image build --rm --tag user-demo:1.0 .
docker image build --rm --tag workdir-demo:1.0 .
```

您会注意到前面每个示例中都有一个尾随的`。`。这个句号表示当前工作目录是图像构建的构建上下文的根目录。

# 解析指令

解析指令是 Dockerfile 中可选注释行的一个特殊子集。任何解析指令必须出现在第一个正常注释行之前。它们还必须出现在任何空行或其他构建指令之前，包括`FROM`指令。基本上，所有解析指令必须位于 Dockerfile 的顶部。顺便说一句，如果你还没有弄清楚，你可以通过以`#`字符开头来创建一个普通的注释行。解析指令的语法如下：

```
# directive=value
# The line above shows the syntax for a parser directive
```

那么，您可以使用解析器指令做什么呢？目前，唯一支持的是`escape`。`escape`解析器指令用于更改用于指示下一个字符在指令中被视为字符而不是表示的特殊字符的字符。如果不使用解析器指令，则默认值为`\`。在本章的几个示例中，您已经看到了它用于转义换行符，允许在 Dockerfile 中将指令继续到下一行。如果需要使用不同的`escape`字符，可以使用`escape`解析器指令来处理。您可以将`escape`字符设置为两种选择之一：

```
# escape=\ (backslash)
Or
# escape=` (backtick)
```

一个例子是当您在 Windows 系统上创建 Dockerfile 时可能需要更改用作`escape`字符的字符。如您所知，`\`用于区分路径字符串中的文件夹级别，例如`c:\windows\system32`。

\drivers`。切换到使用`escape`字符的反引号将避免需要转义此类字符串，例如：`c:\\windows\\system32\\drivers`。

# 构建上下文

构建上下文是在使用构建镜像命令时发送到 Docker 守护程序的所有内容。这包括 Dockerfile 和发出构建命令时当前工作目录的内容，包括当前工作目录可能包含的所有子目录。可以使用`-f`或`--file`选项将 Dockerfile 放在当前工作目录以外的目录中，但 Dockerfile 仍然会随构建上下文一起发送。使用`.dockerignore`文件，可以在发送到 Docker 守护程序的构建上下文中排除文件和文件夹。

构建 Docker 镜像时，非常重要的是尽可能保持构建上下文的大小。这是因为整个构建上下文都会发送到 Docker 守护程序以构建镜像。如果构建上下文中有不必要的文件和文件夹，那么它将减慢构建过程，并且根据 Dockerfile 的内容，可能会导致膨胀的镜像。这是一个如此重要的考虑因素，以至于每个镜像构建命令都会在命令输出的第一行显示构建上下文的大小。它看起来像这样：

![](img/b0300473-f75d-45ad-8bb7-c37fd77987bf.png)

构建上下文成为 Dockerfile 中命令的文件系统根。例如，考虑使用以下`COPY`指令：

```
# build context Dockerfile for Docker Quick Start guide
FROM scratch
COPY hello /
CMD ["/hello"]
```

这告诉 Docker 守护程序将`hello`文件从构建上下文的根目录复制到容器镜像的根目录。

如果命令成功完成，将显示镜像 ID，如果提供了`--tag`选项，则还将显示新的标签和版本：

![](img/a22a41a4-e7bb-47ec-8112-2e22153de989.png)

保持构建上下文小的关键之一是使用`.dockerignore`文件。

# .dockerignore 文件

如果您熟悉使用`.gitignore`文件，那么您已经基本了解了`.dockerignore`文件的目的。`.dockerignore`文件用于排除在 docker 镜像构建过程中不想包含在构建上下文中的文件。使用它有助于防止敏感和其他不需要的文件被包含在构建上下文中，可能最终出现在 docker 镜像中。这是一个帮助保持 Docker 镜像小的绝佳工具。

`.dockerignore`文件需要位于构建上下文的根文件夹中。与`.gitignore`文件类似，它使用一个以换行符分隔的模式列表。`.dockerignore`文件中的注释以`#`作为行的第一个字符。您可以通过包含一个例外行来覆盖模式。例外行以`!`作为行的第一个字符。所有其他行都被视为用于排除文件和/或文件夹的模式。

`.dockerignore`文件中的行顺序很重要。文件后面的匹配模式将覆盖文件前面的匹配模式。如果您添加一个与`.dockerignore`文件或 Dockerfile 文件匹配的模式，它们仍将与构建上下文一起发送到 docker 守护程序，但它们将不可用于任何`ADD`或`COPY`指令，因此不能出现在生成的镜像中。这是一个例子：

```
# Example of a .dockerignore file
# Exclude unwanted files
**/*~
**/*.log
**/.DS_Store
```

# 总结

好了！那是一次冒险。现在您应该能够构建任何类型的 Docker 镜像。您知道何时使用`COPY`而不是`ADD`，何时使用`ENV`而不是`ARG`，也许最重要的是何时使用`CMD`而不是`ENTERYPOINT`。您甚至学会了如何穿越时间！这些信息对于开始使用 Docker 来说真的是一个很好的基础，并且在您开发更复杂的 Docker 镜像时将作为一个很好的参考。

希望你从这一章学到了很多，但我们还有更多要学习，所以让我们把注意力转向下一个主题。在第四章 *Docker Volumes*中，我们将学习更多关于 Docker 卷的知识。翻页，让我们继续我们的快速入门之旅。

# 参考资料

查看以下链接，获取本章讨论的主题信息：

+   hello-world GitHub 存储库：[`github.com/docker-library/hello-world`](https://github.com/docker-library/hello-world)

+   Docker 卷：[`docs.docker.com/storage/volumes/`](https://docs.docker.com/storage/volumes/)

+   使用 Docker 的信号：[`medium.com/@gchudnov/trapping-signals-in-docker-containers-7a57fdda7d86`](https://medium.com/@gchudnov/trapping-signals-in-docker-containers-7a57fdda7d86)

+   `.dockerignore`参考文档：[`docs.docker.com/engine/reference/builder/#dockerignore-file`](https://docs.docker.com/engine/reference/builder/#dockerignore-file)

+   Dockerfile 的最佳实践：[`docs.docker.com/v17.09/engine/userguide/eng-image/dockerfile_best-practices/`](https://docs.docker.com/v17.09/engine/userguide/eng-image/dockerfile_best-practices/)

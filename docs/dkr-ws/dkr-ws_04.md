# 第四章：多阶段 Dockerfiles

概述

在本章中，我们将讨论普通的 Docker 构建。您将审查和实践`Dockerfile`的最佳实践，并学习使用构建模式和多阶段`Dockerfile`来创建和优化 Docker 镜像的大小。

# 介绍

在上一章中，我们学习了 Docker 注册表，包括私有和公共注册表。我们创建了自己的私有 Docker 注册表来存储 Docker 镜像。我们还学习了如何设置访问权限并将我们的 Docker 镜像存储在 Docker Hub 中。在本章中，我们将讨论多阶段`Dockerfiles`的概念。

多阶段`Dockerfiles`是在 Docker 版本 17.05 中引入的一个功能。当我们想要在生产环境中运行 Docker 镜像时，这个功能是可取的。为了实现这一点，多阶段`Dockerfile`将在构建过程中创建多个中间 Docker 镜像，并有选择地从一个阶段复制只有必要的构件到另一个阶段。

在引入多阶段 Docker 构建之前，构建模式被用来优化 Docker 镜像的大小。与多阶段构建不同，构建模式需要两个`Dockerfiles`和一个 shell 脚本来创建高效的 Docker 镜像。

在本章中，我们将首先检查普通的 Docker 构建以及与之相关的问题。接下来，我们将学习如何使用构建模式来优化 Docker 镜像的大小，并讨论与构建模式相关的问题。最后，我们将学习如何使用多阶段`Dockerfiles`来克服构建模式的问题。

# 普通的 Docker 构建

使用 Docker，我们可以使用`Dockerfiles`来创建自定义的 Docker 镜像。正如我们在*第二章，使用 Dockerfiles 入门*中讨论的那样，`Dockerfile`是一个包含如何创建 Docker 镜像的指令的文本文件。然而，在生产环境中运行它们时，拥有最小尺寸的 Docker 镜像是至关重要的。这使开发人员能够加快他们的 Docker 容器的构建和部署时间。在本节中，我们将构建一个自定义的 Docker 镜像，以观察与普通的 Docker 构建过程相关的问题。

考虑一个例子，我们构建一个简单的 Golang 应用程序。我们将使用以下`Dockerfile`部署一个用 Golang 编写的`hello world`应用程序：

```
# Start from latest golang parent image
FROM golang:latest
# Set the working directory
WORKDIR /myapp
# Copy source file from current directory to container
COPY helloworld.go .
# Build the application
RUN go build -o helloworld .
# Run the application
ENTRYPOINT ["./helloworld"]
```

这个`Dockerfile`以最新的 Golang 镜像作为父镜像开始。这个父镜像包含构建 Golang 应用程序所需的所有构建工具。接下来，我们将把`/myapp`目录设置为当前工作目录，并将`helloworld.go`源文件从主机文件系统复制到容器文件系统。然后，我们将使用`RUN`指令执行`go build`命令来构建应用程序。最后，使用`ENTRYPOINT`指令来运行在上一步中创建的`helloworld`可执行文件。

以下是`helloworld.go`文件的内容。这是一个简单的文件，当执行时将打印文本`"Hello World"`：

```
package main
import "fmt"
func main() {
    fmt.Println("Hello World")
}
```

一旦`Dockerfile`准备好，我们可以使用`docker image build`命令构建 Docker 镜像。这个镜像将被标记为`helloworld:v1`：

```
$ docker image build -t helloworld:v1 .
```

现在，使用`docker image ls`命令观察构建的镜像。您将获得类似以下的输出：

```
REPOSITORY   TAG   IMAGE ID       CREATED          SIZE
helloworld   v1    23874f841e3e   10 seconds ago   805MB
```

注意镜像大小。这个构建导致了一个大小为 805 MB 的巨大 Docker 镜像。在生产环境中拥有这些大型 Docker 镜像是低效的，因为它们将花费大量时间和带宽在网络上传输。小型 Docker 镜像更加高效，可以快速推送、拉取和部署。

除了镜像的大小，这些 Docker 镜像可能容易受到攻击，因为它们包含可能存在潜在安全漏洞的构建工具。

注意

潜在的安全漏洞可能会因给定的 Docker 镜像中包含哪些软件包而有所不同。例如，Java JDK 有许多漏洞。您可以在以下链接中详细了解与 Java JDK 相关的漏洞：

[`www.cvedetails.com/vulnerability-list/vendor_id-93/product_id-19116/Oracle-JDK.html`](https://www.cvedetails.com/vulnerability-list/vendor_id-93/product_id-19116/Oracle-JDK.html)。

为了减少攻击面，建议在生产环境中运行 Docker 镜像时只包含必要的构件（例如编译代码）和运行时。例如，使用 Golang 时，需要 Go 编译器来构建应用程序，但不需要运行应用程序。

理想情况下，您希望有一个最小尺寸的 Docker 镜像，其中只包含运行时工具，而排除了用于构建应用程序的所有构建工具。

现在，我们将使用以下练习中的常规构建过程构建这样一个 Docker 镜像。

## 练习 4.01：使用正常构建过程构建 Docker 镜像

您的经理要求您将一个简单的 Golang 应用程序 docker 化。您已经提供了 Golang 源代码文件，您的任务是编译和运行此文件。在这个练习中，您将使用正常的构建过程构建一个 Docker 镜像。然后，您将观察最终 Docker 镜像的大小：

1.  为此练习创建一个名为`normal-build`的新目录：

```
$ mkdir normal-build
```

1.  转到新创建的`normal-build`目录：

```
$ cd normal-build
```

1.  在`normal-build`目录中，创建一个名为`welcome.go`的文件。此文件将在构建时复制到 Docker 镜像中：

```
$ touch welcome.go
```

1.  现在，使用您喜欢的文本编辑器打开`welcome.go`文件：

```
$ vim welcome.go
```

1.  将以下内容添加到`welcome.go`文件中，保存并退出`welcome.go`文件：

```
package main
import "fmt"
func main() {
    fmt.Println("Welcome to multi-stage Docker builds")
}
```

这是一个用 Golang 编写的简单的`hello world`应用程序。这将在执行时输出`"Welcome to multi-stage Docker builds"`。

1.  在`normal-build`目录中，创建一个名为`Dockerfile`的文件：

```
$ touch Dockerfile
```

1.  现在，使用您喜欢的文本编辑器打开`Dockerfile`：

```
$ vim Dockerfile 
```

1.  将以下内容添加到`Dockerfile`中并保存文件：

```
FROM golang:latest
WORKDIR /myapp
COPY welcome.go .
RUN go build -o welcome .
ENTRYPOINT ["./welcome"]
```

`Dockerfile`以`FROM`指令开头，指定最新的 Golang 镜像作为父镜像。这将把`/myapp`目录设置为 Docker 镜像的当前工作目录。然后，`COPY`指令将`welcome.go`源文件复制到 Docker 文件系统中。接下来是`go build`命令，用于构建您创建的 Golang 代码。最后，将执行 welcome 代码。

1.  现在，构建 Docker 镜像：

```
$ docker build -t welcome:v1 .
```

您将看到该镜像成功构建，镜像 ID 为`b938bc11abf1`，标记为`welcome:v1`：

![图 4.1：构建 Docker 镜像](img/B15021_04_01.jpg)

图 4.1：构建 Docker 镜像

1.  使用`docker image ls`命令列出计算机上所有可用的 Docker 镜像：

```
$ docker image ls
```

该命令应返回以下输出：

![图 4.2：列出所有 Docker 镜像](img/B15021_04_02.jpg)

图 4.2：列出所有 Docker 镜像

可以观察到`welcome:v1`镜像的镜像大小为`805MB`。

在本节中，我们讨论了如何使用普通的 Docker 构建过程来构建 Docker 镜像，并观察了其大小。结果是一个巨大的 Docker 镜像，大小超过 800 MB。这些大型 Docker 镜像的主要缺点是它们将花费大量时间来构建、部署、推送和拉取网络。因此，建议尽可能创建最小尺寸的 Docker 镜像。在下一节中，我们将讨论如何使用构建模式来优化镜像大小。

# 什么是构建模式？

**构建模式**是一种用于创建最优尺寸的 Docker 镜像的方法。它使用两个 Docker 镜像，并从一个镜像选择性地复制必要的构件到另一个镜像。第一个 Docker 镜像称为`构建镜像`，用作构建环境，用于从源代码构建可执行文件。这个 Docker 镜像包含在构建过程中所需的编译器、构建工具和开发依赖项。

第二个 Docker 镜像称为`运行时镜像`，用作运行由第一个 Docker 容器创建的可执行文件的运行时环境。这个 Docker 镜像只包含可执行文件、依赖项和运行时工具。使用一个 shell 脚本来使用`docker container cp`命令复制构件。

使用构建模式构建镜像的整个过程包括以下步骤：

1.  创建`Build` Docker 镜像。

1.  从`Build` Docker 镜像创建一个容器。

1.  将构件从`Build` Docker 镜像复制到本地文件系统。

1.  使用复制的构件构建`Runtime` Docker 镜像：

![图 4.3：使用构建模式构建镜像](img/B15021_04_03.jpg)

图 4.3：使用构建模式构建镜像

如前面的图所示，`Build` `Dockerfile`用于创建构建容器，其中包含构建源代码所需的所有工具，包括编译器和构建工具，如 Maven、Gradle 和开发依赖项。创建构建容器后，shell 脚本将从构建容器复制可执行文件到 Docker 主机。最后，将使用从`Build`容器复制的可执行文件创建`Runtime`容器。

现在，观察如何使用构建模式来创建最小的 Docker 镜像。以下是用于创建`Build` Docker 容器的第一个`Dockerfile`。这个`Dockerfile`被命名为 Dockerfile.build，以区别于`Runtime` `Dockerfile`：

```
# Start from latest golang parent image
FROM golang:latest
# Set the working directory
WORKDIR /myapp
# Copy source file from current directory to container
COPY helloworld.go .
# Build the application
RUN go build -o helloworld .
# Run the application
ENTRYPOINT ["./helloworld"]
```

这是我们观察到的与普通 Docker 构建相同的`Dockerfile`。这是用来从`helloworld.go`源文件创建`helloworld`可执行文件的。

以下是用于构建`Runtime` Docker 容器的第二个`Dockerfile`：

```
# Start from latest alpine parent image
FROM alpine:latest
# Set the working directory
WORKDIR /myapp
# Copy helloworld app from current directory to container
COPY helloworld .
# Run the application
ENTRYPOINT ["./helloworld"]
```

与第一个`Dockerfile`相反，它是从`golang`父镜像创建的，这个第二个`Dockerfile`使用`alpine`镜像作为其父镜像，因为它是一个仅有 5MB 的最小尺寸的 Docker 镜像。这个镜像使用 Alpine Linux，一个轻量级的 Linux 发行版。接下来，`/myapp`目录被配置为工作目录。最后，`helloworld`构件被复制到 Docker 镜像中，并且使用`ENTRYPOINT`指令来运行应用程序。

这个`helloworld`构件是在第一个`Dockerfile`中执行的`go build -o helloworld .`命令的结果。我们将使用一个 shell 脚本将这个构件从`build` Docker 容器复制到本地文件系统，然后从那里将这个构件复制到运行时 Docker 镜像。

考虑以下用于在 Docker 容器之间复制构建构件的 shell 脚本：

```
#!/bin/sh
# Build the builder Docker image 
docker image build -t helloworld-build -f Dockerfile.build .
# Create container from the build Docker image
docker container create --name helloworld-build-container   helloworld-build
# Copy build artifacts from build container to the local filesystem
docker container cp helloworld-build-container:/myapp/helloworld .
# Build the runtime Docker image
docker image build -t helloworld .
# Remove the build Docker container
docker container rm -f helloworld-build-container
# Remove the copied artifact
rm helloworld
```

这个 shell 脚本将首先使用`Dockerfile.build`文件构建`helloworld-build` Docker 镜像。下一步是从`helloworld-build`镜像创建一个 Docker 容器，以便我们可以将`helloworld`构件复制到 Docker 主机。容器创建后，我们需要执行命令将`helloworld`构件从`helloworld-build-container`复制到 Docker 主机的当前目录。现在，我们可以使用`docker image build`命令构建运行时容器。最后，我们将执行必要的清理任务，如删除中间构件，比如`helloworld-build-container`容器和`helloworld`可执行文件。

一旦我们执行了 shell 脚本，我们应该能够看到两个 Docker 镜像：

```
REPOSITORY         TAG      IMAGE ID       CREATED       SIZE
helloworld         latest   faff247e2b35   3 hours ago   7.6MB
helloworld-build   latest   f8c10c5bd28d   3 hours ago   805MB
```

注意两个 Docker 镜像之间的大小差异。`helloworld` Docker 镜像的大小仅为 7.6MB，这是从 805MB 的`helloworld-build`镜像中大幅减少的。

正如我们所看到的，构建模式可以通过仅复制必要的构件到最终镜像来大大减小 Docker 镜像的大小。然而，构建模式的缺点是我们需要维护两个`Dockerfiles`和一个 shell 脚本。

在下一个练习中，我们将亲自体验使用构建模式创建优化的 Docker 镜像。

## 练习 4.02：使用构建模式构建 Docker 镜像

在*练习 4.01*中，*使用常规构建过程构建 Docker 镜像*，您创建了一个 Docker 镜像来编译和运行 Golang 应用程序。现在应用程序已经准备就绪，但经理对 Docker 镜像的大小不满意。您被要求创建一个最小尺寸的 Docker 镜像来运行应用程序。在这个练习中，您将使用构建模式优化 Docker 镜像：

1.  为这个练习创建一个名为`builder-pattern`的新目录：

```
$ mkdir builder-pattern
```

1.  导航到新创建的`builder-pattern`目录：

```
$ cd builder-pattern
```

1.  在`builder-pattern`目录中，创建一个名为`welcome.go`的文件。这个文件将在构建时复制到 Docker 镜像中：

```
$ touch welcome.go
```

1.  现在，使用您喜欢的文本编辑器打开`welcome.go`文件：

```
$ vim welcome.go
```

1.  将以下内容添加到`welcome.go`文件中，然后保存并退出该文件：

```
package main
import "fmt"
func main() {
    fmt.Println("Welcome to multi-stage Docker builds")
}
```

这是一个用 Golang 编写的简单的`hello world`应用程序。一旦执行，它将输出“欢迎来到多阶段 Docker 构建”。

1.  在`builder-pattern`目录中，创建一个名为`Dockerfile.build`的文件。这个文件将包含您将用来创建`build` Docker 镜像的所有指令：

```
$ touch Dockerfile.build
```

1.  现在，使用您喜欢的文本编辑器打开`Dockerfile.build`：

```
$ vim Dockerfile.build
```

1.  将以下内容添加到`Dockerfile.build`文件中并保存该文件：

```
FROM golang:latest
WORKDIR /myapp
COPY welcome.go .
RUN go build -o welcome .
ENTRYPOINT ["./welcome"]
```

这与您在*练习 4.01*中为`Dockerfile`创建的内容相同，*使用常规构建过程构建 Docker 镜像*。

1.  接下来，为运行时容器创建`Dockerfile`。在`builder-pattern`目录中，创建一个名为`Dockerfile`的文件。这个文件将包含您将用来创建运行时 Docker 镜像的所有指令：

```
$ touch Dockerfile
```

1.  现在，使用您喜欢的文本编辑器打开`Dockerfile`：

```
$ vim Dockerfile
```

1.  将以下内容添加到`Dockerfile`并保存该文件：

```
FROM scratch
WORKDIR /myapp
COPY welcome .
ENTRYPOINT ["./welcome"]
```

这个`Dockerfile`使用了 scratch 镜像，这是 Docker 中最小的镜像，作为父镜像。然后，它将`/myapp`目录配置为工作目录。接下来，欢迎可执行文件从 Docker 主机复制到运行时 Docker 镜像。最后，使用`ENTRYPOINT`指令来执行欢迎可执行文件。

1.  创建一个 shell 脚本，在两个 Docker 容器之间复制可执行文件。在`builder-pattern`目录中，创建一个名为`build.sh`的文件。这个文件将包含协调两个 Docker 容器之间构建过程的步骤：

```
$ touch build.sh
```

1.  现在，使用您喜欢的文本编辑器打开`build.sh`文件：

```
$ vim build.sh
```

1.  将以下内容添加到 shell 脚本中并保存文件：

```
#!/bin/sh
echo "Creating welcome builder image"
docker image build -t welcome-builder:v1 -f Dockerfile.build .
docker container create --name welcome-builder-container   welcome-builder:v1
docker container cp welcome-builder-container:/myapp/welcome .
docker container rm -f welcome-builder-container
echo "Creating welcome runtime image"
docker image build -t welcome-runtime:v1 .
rm welcome
```

这个 shell 脚本将首先构建`welcome-builder` Docker 镜像并从中创建一个容器。然后它将从容器中将编译的 Golang 可执行文件复制到本地文件系统。接下来，`welcome-builder-container`容器将被删除，因为它是一个中间容器。最后，将构建`welcome-runtime`镜像。

1.  为`build.sh` shell 脚本添加执行权限：

```
$ chmod +x build.sh
```

1.  现在您已经有了两个`Dockerfiles`和 shell 脚本，通过执行`build.sh` shell 脚本构建 Docker 镜像：

```
$ ./build.sh
```

镜像将成功构建并标记为`welcome-runtime:v1`：

![图 4.4：构建 Docker 镜像](img/B15021_04_04.jpg)

图 4.4：构建 Docker 镜像

1.  使用`docker image` ls 命令列出计算机上所有可用的 Docker 镜像：

```
docker image ls
```

您应该得到所有可用 Docker 镜像的列表，如下图所示：

![图 4.5：列出所有 Docker 镜像](img/B15021_04_05.jpg)

图 4.5：列出所有 Docker 镜像

从前面的输出中可以看到，有两个 Docker 镜像可用。welcome-builder 具有所有构建工具，大小为 805 MB，而 welcome-runtime 的镜像大小显著较小，为 2.01 MB。`golang:latest`是我们用作`welcome-builder`父镜像的 Docker 镜像。

在这个练习中，您学会了如何使用构建模式来减小 Docker 镜像的大小。然而，使用构建模式来优化 Docker 镜像的大小意味着我们必须维护两个`Dockerfiles`和一个 shell 脚本。在下一节中，让我们观察如何通过使用多阶段`Dockerfile`来消除它们。

# 多阶段 Dockerfile 简介

**多阶段 Dockerfile**是一个功能，允许单个`Dockerfile`包含多个阶段，可以生成优化的 Docker 镜像。正如我们在上一节中观察到的构建模式一样，这些阶段通常包括一个构建状态，用于从源代码构建可执行文件，以及一个运行时阶段，用于运行可执行文件。多阶段`Dockerfiles`将在`Dockerfile`中为每个阶段使用多个`FROM`指令，并且每个阶段将以不同的基础镜像开始。只有必要的文件将从一个阶段有选择地复制到另一个阶段。在多阶段`Dockerfiles`之前，这是通过构建模式实现的，正如我们在上一节中讨论的那样。

多阶段 Docker 构建允许我们创建与构建模式类似但消除了与之相关的问题的最小尺寸 Docker 镜像。正如我们在前面的示例中看到的，构建模式需要维护两个`Dockerfile`和一个 shell 脚本。相比之下，多阶段 Docker 构建只需要一个`Dockerfile`，并且不需要任何 shell 脚本来在 Docker 容器之间复制可执行文件。此外，构建模式要求您在将可执行文件复制到最终 Docker 镜像之前将其复制到 Docker 主机。而多阶段 Docker 构建不需要这样做，因为我们可以使用`--from`标志在 Docker 镜像之间复制可执行文件，而无需将其复制到 Docker 主机。

现在，让我们观察一下多阶段`Dockerfile`的结构：

```
# Start from latest golang parent image
FROM golang:latest
# Set the working directory
WORKDIR /myapp
# Copy source file from current directory to container
COPY helloworld.go .
# Build the application
RUN go build -o helloworld .
# Start from latest alpine parent image
FROM alpine:latest
# Set the working directory
WORKDIR /myapp
# Copy helloworld app from current directory to container
COPY --from=0 /myapp/helloworld .
# Run the application
ENTRYPOINT ["./helloworld"]
```

普通的`Dockerfile`和多阶段`Dockerfile`之间的主要区别在于，多阶段`Dockerfile`将使用多个`FROM`指令来构建每个阶段。每个新阶段将从一个新的父镜像开始，并且不包含来自先前镜像的任何内容，除了有选择地复制的可执行文件。使用`COPY --from=0`将可执行文件从第一阶段复制到第二阶段。

构建 Docker 镜像并将镜像标记为`multi-stage:v1`：

```
docker image build -t multi-stage:v1 .
```

现在，您可以列出可用的 Docker 镜像：

```
REPOSITORY    TAG      IMAGE ID       CREATED         SIZE
multi-stage   latest   75e1f4bcabd0   7 seconds ago   7.6MB
```

您可以看到，这导致了与构建模式中观察到的相同大小的 Docker 镜像。

注意

多阶段`Dockerfile`减少了所需的`Dockerfile`数量，并消除了 shell 脚本，而不会对镜像的大小产生任何影响。

默认情况下，多阶段`Dockerfile`中的阶段由整数编号引用，从第一阶段开始为`0`。可以通过在`FROM`指令中添加`AS <NAME>`来为这些阶段命名，以增加可读性和可维护性。以下是您在前面的代码块中观察到的多阶段`Dockerfile`的改进版本：

```
# Start from latest golang parent image
FROM golang:latest AS builder 
# Set the working directory
WORKDIR /myapp
# Copy source file from current directory to container
COPY helloworld.go .
# Build the application
RUN go build -o helloworld .
# Start from latest alpine parent image
FROM alpine:latest AS runtime
# Set the working directory
WORKDIR /myapp
# Copy helloworld app from current directory to container
COPY --from=builder /myapp/helloworld .
# Run the application
ENTRYPOINT ["./helloworld"]
```

在前面的示例中，我们将第一阶段命名为`builder`，第二阶段命名为`runtime`，如下所示：

```
FROM golang:latest AS builder
FROM alpine:latest AS runtime
```

然后，在第二阶段复制工件时，您使用了`--from`标志的名称`builder`：

```
COPY --from=builder /myapp/helloworld .
```

在构建多阶段`Dockerfile`时，可能会有一些情况，您只想构建到特定的构建阶段。考虑到您的`Dockerfile`有两个阶段。第一个是构建开发阶段，包含所有构建和调试工具，第二个是构建仅包含运行时工具的生产镜像。在项目的代码开发阶段，您可能只需要构建到开发阶段，以便在必要时测试和调试代码。在这种情况下，您可以使用`docker build`命令的`--target`标志来指定一个中间阶段作为最终镜像的阶段：

```
docker image build --target builder -t multi-stage-dev:v1 .
```

在前面的例子中，您使用了`--target builder`来停止在构建阶段停止构建。

在下一个练习中，您将学习如何使用多阶段`Dockerfile`来创建一个大小优化的 Docker 镜像。

## 练习 4.03：使用多阶段 Docker 构建构建 Docker 镜像

在*Exercise 4.02*，*使用构建模式构建 Docker 镜像*中，您使用了构建模式来优化 Docker 镜像的大小。然而，这会带来操作负担，因为您需要在 Docker 镜像构建过程中管理两个`Dockerfiles`和一个 shell 脚本。在这个练习中，您将使用多阶段`Dockerfile`来消除这种操作负担。

1.  为这个练习创建一个名为`multi-stage`的新目录：

```
mkdir multi-stage
```

1.  导航到新创建的`multi-stage`目录：

```
cd multi-stage
```

1.  在`multi-stage`目录中，创建一个名为`welcome.go`的文件。这个文件将在构建时复制到 Docker 镜像中：

```
$ touch welcome.go
```

1.  现在，使用您喜欢的文本编辑器打开`welcome.go`文件：

```
$ vim welcome.go
```

1.  将以下内容添加到`welcome.go`文件中，然后保存并退出此文件：

```
package main
import "fmt"
func main() {
    fmt.Println("Welcome to multi-stage Docker builds")
}
```

这是一个用 Golang 编写的简单的`hello world`应用程序。一旦执行，它将输出`"Welcome to multi-stage Docker builds"`。

在 multi-stage 目录中，创建一个名为`Dockerfile`的文件。这个文件将是多阶段`Dockerfile`：

```
touch Dockerfile
```

1.  现在，使用您喜欢的文本编辑器打开`Dockerfile`：

```
vim Dockerfile
```

1.  将以下内容添加到`Dockerfile`中并保存文件：

```
FROM golang:latest AS builder
WORKDIR /myapp
COPY welcome.go .
RUN go build -o welcome .
FROM scratch
WORKDIR /myapp
COPY --from=builder /myapp/welcome .
ENTRYPOINT ["./welcome"]
```

这个多阶段`Dockerfile`使用最新的`golang`镜像作为父镜像，这个阶段被命名为`builder`。接下来，指定`/myapp`目录作为当前工作目录。然后，使用`COPY`指令复制`welcome.go`源文件，并使用`RUN`指令构建 Golang 文件。

`Dockerfile`的下一阶段使用`scratch`镜像作为父镜像。这将把`/myapp`目录设置为 Docker 镜像的当前工作目录。然后，使用`COPY`指令将`welcome`可执行文件从构建阶段复制到此阶段。最后，使用`ENTRYPOINT`运行`welcome`可执行文件。

1.  使用以下命令构建 Docker 镜像：

```
docker build -t welcome-optimized:v1 .
```

镜像将成功构建并标记为`welcome-optimized:v1`：

![图 4.6：构建 Docker 镜像](img/B15021_04_06.jpg)

图 4.6：构建 Docker 镜像

1.  使用`docker image ls`命令列出计算机上所有可用的 Docker 镜像。这些镜像可以在您的计算机上使用，无论是从 Docker Registry 拉取还是在您的计算机上构建：

```
docker images
```

从以下输出中可以看出，`welcome-optimized`镜像与您在*练习 4.02 中构建的`welcome-runtime`镜像大小相同，构建 Docker 镜像使用以下命令：

![图 4.7：列出所有 Docker 镜像](img/B15021_04_07.jpg)

图 4.7：列出所有 Docker 镜像

在本练习中，您学习了如何使用多阶段`Dockerfiles`构建优化的 Docker 镜像。以下表格总结了构建器模式和多阶段`Docker Builds`之间的关键差异：

![图 4.8：构建器模式和多阶段 Docker 构建之间的差异](img/B15021_04_08.jpg)

图 4.8：构建器模式和多阶段 Docker 构建之间的差异

在下一节中，我们将回顾编写`Dockerfile`时应遵循的最佳实践。

# Dockerfile 最佳实践

在前一节中，我们讨论了如何使用多阶段`Dockerfiles`构建高效的 Docker 镜像。在本节中，我们将介绍编写`Dockerfiles`的其他推荐最佳实践。这些最佳实践将确保减少构建时间、减少镜像大小、增加安全性和增加 Docker 镜像的可维护性。

## 使用适当的父镜像

在构建高效的 Docker 镜像时，使用适当的基础镜像是其中的关键建议之一。

在构建自定义 Docker 镜像时，建议始终使用**Docker Hub**的官方镜像作为父镜像。这些官方镜像将确保遵循所有最佳实践，提供文档，并应用安全补丁。例如，如果您的应用程序需要**JDK**（Java 开发工具包），您可以使用`openjdk`官方 Docker 镜像，而不是使用通用的`ubuntu`镜像并在`ubuntu`镜像上安装 JDK：

![图 4.9：使用适当的父镜像](img/B15021_04_09.jpg)

图 4.9：使用适当的父镜像

其次，在为生产环境构建 Docker 镜像时，避免使用父镜像的`latest`标签。`latest`标签可能会指向 Docker Hub 发布新版本时的新版本镜像，而新版本可能与您的应用程序不兼容，导致生产环境中的故障。相反，最佳实践是始终使用特定版本的标签作为父镜像：

![图 4.10：避免使用父镜像的最新标签](img/B15021_04_10.jpg)

图 4.10：避免使用父镜像的最新标签

最后，使用父镜像的最小版本对于获得最小尺寸的 Docker 镜像至关重要。Docker Hub 中的大多数官方 Docker 镜像都是围绕 Alpine Linux 镜像构建的最小尺寸镜像。此外，在我们的示例中，我们可以使用**JRE**（Java 运行环境）来运行应用程序，而不是 JDK，后者包含构建工具：

![图 4.11：使用最小尺寸的镜像](img/B15021_04_11.jpg)

图 4.11：使用最小尺寸的镜像

`openjdk:8-jre-alpine`镜像的大小仅为 84.9 MB，而`openjdk:8`的大小为 488 MB。

## 为了更好的安全性，使用非根用户

默认情况下，Docker 容器以 root（`id = 0`）用户运行。这允许用户执行所有必要的管理活动，如更改系统配置、安装软件包和绑定特权端口。然而，在生产环境中运行 Docker 容器时，这是高风险的，被认为是一种不良的安全实践，因为黑客可以通过攻击 Docker 容器内运行的应用程序来获得对 Docker 主机的 root 访问权限。

以非 root 用户身份运行容器是改善 Docker 容器安全性的推荐最佳实践。这将遵循最小特权原则，确保应用程序只具有执行其任务所需的最低权限。我们可以使用两种方法来以非 root 用户身份运行容器：使用`--user`（或`-u`）标志，以及使用`USER`指令。

使用`--user`（或`-u`）标志与`docker run`命令是在运行 Docker 容器时更改默认用户的一种方法。`--user`（或`-u`）标志可以指定用户名或用户 ID：

```
$ docker run --user=9999 ubuntu:focal
```

在前面的命令中，我们指定了用户 ID 为`9999`。如果我们将用户指定为 ID，则相应的用户不必在 Docker 容器中可用。

此外，我们可以在`Dockerfile`中使用`USER`指令来定义默认用户。但是，可以在启动 Docker 容器时使用`--user`标志覆盖此值：

```
FROM ubuntu:focal
RUN apt-get update 
RUN useradd demo-user
USER demo-user
CMD whoami
```

在前面的例子中，我们使用了`USER`指令将默认用户设置为`demo-user`。这意味着在`USER`指令之后的任何命令都将以`demo-user`身份执行。

## 使用 dockerignore

`.dockerignore`文件是 Docker 上下文中的一个特殊文本文件，用于指定在构建 Docker 镜像时要排除的文件列表。一旦执行`docker build`命令，Docker 客户端将整个构建上下文打包为一个 TAR 归档文件，并将其上传到 Docker 守护程序。当我们执行`docker build`命令时，输出的第一行是`Sending build context to Docker daemon`，这表示 Docker 客户端正在将构建上下文上传到 Docker 守护程序。

```
Sending build context to Docker daemon  18.6MB
Step 1/5 : FROM ubuntu:focal
```

每次构建 Docker 镜像时，构建上下文都将被发送到 Docker 守护程序。由于这将在 Docker 镜像构建过程中占用时间和带宽，建议排除所有在最终 Docker 镜像中不需要的文件。`.dockerignore`文件可用于实现此目的。除了节省时间和带宽外，`.dockerignore`文件还用于排除机密文件，例如密码文件和密钥文件，以防止其出现在构建上下文中。

`.dockerignore`文件应该创建在构建上下文的根目录中。在将构建上下文发送到 Docker 守护程序之前，Docker 客户端将在构建上下文的根目录中查找`.dockerignore`文件。如果`.dockerignore`文件存在，Docker 客户端将从构建上下文中排除`.dockerignore`文件中提到的所有文件。

以下是一个示例`.dockerignore`文件的内容：

```
PASSWORDS.txt
tmp/
*.md
!README.md
```

在上面的示例中，我们特别排除了`PASSWORDS.txt`文件和`tmp`目录，以及除`README.md`文件之外的所有扩展名为`.md`的文件。

## 最小化层

`Dockerfile`中的每一行都将创建一个新的层，这将占用 Docker 镜像中的空间。因此，在构建 Docker 镜像时，建议尽可能少地创建层。为了实现这一点，尽可能合并`RUN`指令。

例如，考虑以下`Dockerfile`，它将首先更新软件包存储库，然后安装`redis-server`和`nginx`软件包：

```
FROM ubuntu:focal
RUN apt-get update
RUN apt-get install -y nginx
RUN apt-get install -y redis-server
```

这个`Dockerfile`可以通过合并三个`RUN`指令来优化：

```
FROM ubuntu:focal
RUN apt-get update \
  && apt-get install -y nginx redis-server
```

## 不安装不必要的工具

不安装不必要的调试工具（如`vim`，`curl`和`telnet`）并删除不必要的依赖项可以帮助创建尺寸小的高效 Docker 镜像。一些软件包管理器（如`apt`）将自动安装推荐和建议的软件包，以及所需的软件包。我们可以通过在`apt-get install`命令中指定`no-install-recommends`标志来避免这种情况：

```
FROM ubuntu:focal
RUN apt-get update \
  && apt-get install --no-install-recommends -y nginx 
```

在上面的示例中，我们正在使用`no-install-recommends`标志安装`nginx`软件包，这将帮助将最终镜像大小减少约 10MB。

除了使用`no-install-recommends`标志之外，我们还可以删除`apt`软件包管理器的缓存，以进一步减少最终的 Docker 镜像大小。这可以通过在`apt-get install`命令的末尾运行`rm -rf /var/lib/apt/lists/*`来实现：

```
FROM ubuntu:focal
RUN apt-get update \
    && apt-get install --no-install-recommends -y nginx \
    && rm -rf /var/lib/apt/lists/*
```

在本节中，我们讨论了编写`Dockerfile`时的最佳实践。遵循这些最佳实践将有助于减少构建时间，减小镜像大小，增加安全性，并增加 Docker 镜像的可维护性。

现在，让我们通过在下一个活动中使用多阶段 Docker 构建部署一个 Golang HTTP 服务器来测试我们的知识。

## 活动 4.01：使用多阶段 Docker 构建部署 Golang HTTP 服务器

假设你被要求将一个 Golang HTTP 服务器部署到一个 Docker 容器中。你的经理要求你构建一个最小尺寸的 Docker 镜像，并在构建`Dockerfile`时遵循最佳实践。

这个 Golang HTTP 服务器将根据调用 URL 返回不同的响应：

![图 4.12：基于调用 URL 的响应](img/B15021_04_12.jpg)

图 4.12：基于调用 URL 的响应

你的任务是使用多阶段`Dockerfile`来将下面代码块中给出的 Golang 应用程序 docker 化：

```
package main
import (
    "net/http"
    "fmt"
    "log"
    "os"
)
func main() {
    http.HandleFunc("/", defaultHandler)
    http.HandleFunc("/contact", contactHandler)
    http.HandleFunc("/login", loginHandler)
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }
    log.Println("Service started on port " + port)
    err := http.ListenAndServe(":"+port, nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
        return
    }
}
func defaultHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "<h1>Home Page</h1>")
}
func contactHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "<h1>Contact Us</h1>")
}
func loginHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "<h1>Login Page</h1>")
}
```

执行以下步骤完成这个活动：

1.  创建一个文件夹来存储活动文件。

1.  创建一个`main.go`文件，其中包含前面代码块中提供的代码。

1.  使用两个阶段创建一个多阶段`Dockerfile`。第一阶段将使用`golang`镜像。这个阶段将使用`go build`命令构建 Golang 应用程序。第二阶段将使用`alpine`镜像。这个阶段将从第一阶段复制可执行文件并执行它。

1.  构建并运行 Docker 镜像。

1.  完成后，停止并移除 Docker 容器。

当你访问 URL `http://127.0.0.1:8080/`时，你应该得到以下输出：

![图 4.13：活动 4.01 的预期输出](img/B15021_04_13.jpg)

图 4.13：活动 4.01 的预期输出

注意

这个活动的解决方案可以通过此链接找到。

# 总结

我们从定义普通的 Docker 构建开始，使用普通的 Docker 构建过程创建了一个简单的 Golang Docker 镜像。然后我们观察了生成的 Docker 镜像的大小，并讨论了最小尺寸的 Docker 镜像如何加快 Docker 容器的构建和部署时间，并通过减少攻击面来增强安全性。

然后，我们使用构建器模式创建了最小尺寸的 Docker 镜像，在这个过程中利用了两个`Dockerfiles`和一个 shell 脚本来创建镜像。我们探讨了多阶段 Docker 构建——这是 Docker 17.05 版本引入的新功能，可以帮助消除维护两个`Dockerfiles`和一个 shell 脚本的操作负担。最后，我们讨论了编写`Dockerfiles`的最佳实践以及这些最佳实践如何确保减少构建时间、减小镜像大小、增强安全性，同时提高 Docker 镜像的可维护性。

在下一章中，我们将介绍`docker-compose`以及如何使用它来定义和运行多容器 Docker 应用程序。

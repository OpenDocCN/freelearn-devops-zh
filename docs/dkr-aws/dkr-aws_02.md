# 第二章：使用 Docker 构建应用程序

在上一章中，您已经介绍了示例应用程序，并且能够下载并在本地运行该应用程序。目前，您的开发环境已经设置好用于本地开发；但是，在将应用程序部署到生产环境之前，您需要能够打包应用程序及其所有依赖项，确保目标生产环境具有正确的操作系统支持库和配置，选择适当的 Web 服务器来托管您的应用程序，并且有一种机制能够将所有这些内容打包在一起，最好是一个自包含的构件，需要最少的外部配置。传统上，要可靠和一致地实现所有这些内容非常困难，但是 Docker 已经极大地改变了这一局面。通过 Docker 和支持工具，您现在有能力以比以往更快、更可靠、更一致和更可移植的方式实现所有这些内容以及更多。

在本章中，您将学习如何使用 Docker 创建一个全面的工作流程，使您能够以可移植、可重复和一致的方式测试、构建和发布应用程序。您将学习的方法有许多好处，例如，您将能够通过运行几个简单、易于记忆的命令来执行所有任务，并且无需在本地开发或构建环境中安装任何特定于应用程序或操作系统的依赖项。这使得在另一台机器上移动或配置连续交付服务来执行相同的工作流程非常容易——只要您在上一章中设置的核心基于 Docker 的环境，您就能够在任何机器上运行工作流程，而不受应用程序或编程语言的具体细节的影响。

您将学习如何使用 Dockerfile 为应用程序定义测试和运行时环境，配置支持多阶段构建，允许您在具有所有开发工具和库的镜像中构建应用程序构件，然后将这些构件复制到 Dockerfile 的其他阶段。您将利用 Docker Compose 作为一个工具来编排具有多个容器的复杂 Docker 环境，这使您能够测试集成场景，例如您的应用程序与数据库的交互，并模拟您在生产环境中运行应用程序的方式。一个重要的概念是引入构建发布镜像的概念，这是一个可以被部署到生产环境的生产就绪镜像，假设任何新的应用程序特性和功能都能正常工作。您将在本地 Docker 环境中构建和运行此发布镜像，将您的应用程序连接到数据库，然后创建验收测试，验证应用程序从外部客户端连接到您的应用程序的角度来看是否正常工作。

最后，您将使用 GNU Make 将学到的所有知识整合起来，自动化您的工作流程。完成后，您只需运行`make test`即可运行单元测试和构建应用程序构件，然后构建您的发布镜像，启动类似生产环境的环境，并通过运行`make release`运行验收测试。这将使测试和发布新的应用程序更改变得非常简单，并且可以放心地使用便携和一致的工作流在本地开发环境和任何支持 Docker 和 Docker Compose 的持续交付环境中运行。

将涵盖以下主题：

+   使用 Docker 测试和构建应用程序

+   创建多阶段构建

+   创建一个测试阶段来构建和测试应用程序构件

+   创建一个发布阶段来构建和测试发布镜像

+   使用 Docker Compose 测试和构建应用程序

+   创建验收测试

+   自动化工作流程

# 技术要求

以下列出了完成本章所需的技术要求：

+   根据第一章的说明安装先决软件

+   根据第一章的说明创建 GitHub 帐户

以下 GitHub 网址包含本章中使用的代码示例：[`github.com/docker-in-aws/docker-in-aws/tree/master/ch2`](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch2)[.](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch3)

查看以下视频，了解代码的运行情况：

[`bit.ly/2PJG2Zm`](http://bit.ly/2PJG2Zm)

# 使用 Docker 测试和构建应用程序

在上一章中，您对示例应用程序是什么以及如何在本地开发环境中测试和运行应用程序有了很好的理解。现在，您已经准备好开始创建一个 Docker 工作流程，用于测试、构建和打包应用程序成为一个 Docker 镜像。

重要的是要理解，每当您将一个应用程序打包成一个 Docker 镜像时，最佳实践是减少或消除所有开发和测试依赖项，使其成为最终打包的应用程序。按照我的约定，我将这个打包的应用程序——不包含测试和开发依赖项——称为*发布镜像*，支持持续交付的范式，即每次成功构建都应该是一个发布候选，可以在需要时发布到生产环境。

为了实现创建发布镜像的目标，一个行之有效的方法是将 Docker 构建过程分为两个阶段：

+   **测试阶段**：该阶段具有所有测试和开发依赖项，可用于编译和构建应用程序源代码成应用程序构件，并运行单元测试和集成测试。

+   **发布阶段**：该阶段将经过测试和构建的应用程序构件从测试阶段复制到一个最小化的运行时环境中，该环境已适当配置以在生产环境中运行应用程序。

Docker 原生支持这种方法，使用一个名为多阶段构建的功能，这是我们将在本书中采用的方法。现在，我们将专注于测试阶段，并在下一节转移到发布阶段。

# 创建一个测试阶段

我们将从在`todobackend`存储库的根目录创建一个`Dockerfile`开始，这意味着您的存储库结构应该看起来像这样：

```
todobackend> tree -L 2
.
├── Dockerfile
├── README.md
└── src
    ├── coverage.xml
    ├── db.sqlite3
    ├── manage.py
    ├── requirements.txt
    ├── requirements_test.txt
    ├── todo
    ├── todobackend
    └── unittests.xml

3 directories, 8 files
```

现在让我们在新创建的 Dockerfile 中定义一些指令：

```
# Test stage
FROM alpine AS test
LABEL application=todobackend
```

`FROM`指令是您在 Dockerfile 中定义的第一个指令，注意我们使用 Alpine Linux 发行版作为基础镜像。Alpine Linux 是一个极简的发行版，比传统的 Linux 发行版（如 Ubuntu 和 CentOS）的占用空间要小得多，并且自从 Docker 采用 Alpine 作为官方 Docker 镜像的首选发行版以来，在容器世界中变得非常流行。

一个你可能不熟悉的关键字是`AS`关键字，它附加到`FROM`指令，将 Dockerfile 配置为[多阶段构建](https://docs.docker.com/develop/develop-images/multistage-build/)，并将当前阶段命名为`test`。当你有一个多阶段构建时，你可以包含多个`FROM`指令，每个阶段都包括当前的`FROM`指令和后续的指令，直到下一个`FROM`指令。

接下来，我们使用`LABEL`指令附加一个名为`application`的标签，其值为`todobackend`，这对于能够识别支持 todobackend 应用程序的 Docker 镜像非常有用。

# 安装系统和构建依赖

现在我们需要安装各种系统和构建操作系统依赖项，以支持测试和构建应用程序：

```
# Test stage
FROM alpine AS test
LABEL application=todobackend

# Install basic utilities
RUN apk add --no-cache bash git
# Install build dependencies RUN apk add --no-cache gcc python3-dev libffi-dev musl-dev linux-headers mariadb-dev
RUN pip3 install wheel
```

在上面的示例中，我们安装了以下依赖项：

+   基本实用程序：在 Alpine Linux 中，软件包管理器称为`apk`，在 Docker 镜像中常用的模式是`apk add --no-cache`，它安装了引用的软件包，并确保下载的软件包不被缓存。我们安装了`bash`，这对故障排除很有用，还有`git`，因为我们将在以后使用 Git 元数据来为 Docker 发布镜像生成应用程序版本标签。

+   构建依赖：在这里，我们安装了构建应用程序所需的各种开发库。这包括`gcc`，`python3-dev`，`libffi-dev`，`musl-dev`和`linux-headers`，用于编译任何 Python C 扩展及其支持的标准库，以及`mariadb-dev`软件包，这是构建 todobackend 应用程序中 MySQL 客户端所需的。您还安装了一个名为`wheel`的 Python 软件包，它允许您构建 Python“wheels”，这是一种预编译和预构建的打包格式，我们以后会用到。

# 安装应用程序依赖

下一步是安装应用程序的依赖项，就像你在上一章中学到的那样，这意味着安装在`src/requirements.txt`和`src/requirements_test.txt`文件中定义的软件包：

```
# Test stage
FROM alpine AS test
LABEL application=todobackend

# Install basic utilities
RUN apk add --no-cache bash git

# Install build dependencies
RUN apk add --no-cache gcc python3-dev libffi-dev musl-dev linux-headers mariadb-dev
RUN pip3 install wheel

# Copy requirements
COPY /src/requirements* /build/
WORKDIR /build

# Build and install requirements
RUN pip3 wheel -r requirements_test.txt --no-cache-dir --no-input
RUN pip3 install -r requirements_test.txt -f /build --no-index --no-cache-dir
```

首先使用`COPY`指令将`src/requirements.txt`和`src/requirements_test.txt`文件复制到`/build`容器中的一个文件夹中，然后通过`WORKDIR`指令将其指定为工作目录。请注意，`/src/requirements.txt`不是您的 Docker 客户端上的物理路径 - 它是 Docker *构建上下文*中的路径，这是您在执行构建时指定的 Docker 客户端文件系统上的可配置位置。为了确保 Docker 构建过程中所有相关的应用程序源代码文件都可用，一个常见的做法是将应用程序存储库的根目录设置为构建上下文，因此在上面的示例中，`/src/requirements.txt`指的是您的 Docker 客户端上的`<path-to-repository>/src/requirements.txt`。

接下来，您使用`pip3` wheel 命令将 Python wheels 构建到`/build`工作目录中，用于所有基本应用程序和测试依赖项，使用`--no-cache-dir`标志来避免膨胀我们的镜像，使用`--no-input`标志来禁用提示用户确认。最后，您使用`pip3 install`命令将先前构建的 wheels 安装到容器中，使用`--no-index`标志指示 pip 不要尝试从互联网下载任何软件包，而是从`/build`文件夹中安装所有软件包，如`-f`标志所指定的那样。

这种方法可能看起来有点奇怪，但它基于一个原则，即您应该只构建一次您的应用程序依赖项作为可安装的软件包，然后根据需要安装构建的依赖项。稍后，我们将在发布镜像中安装相同的依赖项，确保您的发布镜像准确反映了您的应用程序经过测试和构建的确切依赖项集。

# 复制应用程序源代码并运行测试

测试阶段的最后步骤是将应用程序源代码复制到容器中，并添加支持运行测试的功能：

```
# Test stage
FROM alpine AS test
LABEL application=todobackend

# Install basic utilities
RUN apk add --no-cache bash git

# Install build dependencies
RUN apk add --no-cache gcc python3-dev libffi-dev musl-dev linux-headers mariadb-dev
RUN pip3 install wheel

# Copy requirements
COPY /src/requirements* /build/
WORKDIR /build

# Build and install requirements
RUN pip3 wheel -r requirements_test.txt --no-cache-dir --no-input
RUN pip3 install -r requirements_test.txt -f /build --no-index --no-cache-dir

# Copy source code COPY /src /app
WORKDIR /app # Test entrypoint CMD ["python3", "manage.py", "test", "--noinput", "--settings=todobackend.settings_test"]
```

在前面的例子中，您首先将整个`/src`文件夹复制到一个名为`/app`的文件夹中，然后将工作目录更改为`/app`。您可能会想为什么我们在复制需求文件时没有直接复制所有应用程序源代码。答案是，我们正在实施缓存优化，因为您的需求文件需要构建应用程序依赖项，并且通过在一个单独的较早的层中构建它们，如果需求文件保持不变（它们往往会这样做），Docker 可以利用最近构建的层的缓存版本，而不必每次构建图像时都构建和安装应用程序依赖项。

最后，我们添加了`CMD`指令，它定义了应该在基于此镜像创建和执行的容器中执行的默认命令。请注意，我们指定了与上一章中用于在本地运行应用程序测试的`python3 manage.py test`命令相同的命令。

您可能会想为什么我们不直接使用`RUN`指令在图像中运行测试。答案是，您可能希望在构建过程中收集构件，例如测试报告，这些构件更容易从您从 Docker 镜像生成的容器中复制，而不是在实际的图像构建过程中。

到目前为止，我们已经定义了 Docker 构建过程的第一个阶段，它将创建一个准备好进行测试的自包含环境，其中包括所需的操作系统依赖项、应用程序依赖项和应用程序源代码。要构建图像，您可以运行`docker build`命令，并使用名称`todobackend-test`对图像进行标记。

```
> docker build --target test -t todobackend-test . Sending build context to Docker daemon 311.8kB
Step 1/12 : FROM alpine AS test
 ---> 3fd9065eaf02
Step 2/12 : LABEL application=todobackend
 ---> Using cache
 ---> afdd1dee07d7
Step 3/12 : RUN apk add --no-cache bash git
 ---> Using cache
 ---> d9cd912ffa68
Step 4/12 : RUN apk add --no-cache gcc python3-dev libffi-dev musl-dev linux-headers mariadb-dev
 ---> Using cache
 ---> 89113207b0b8
Step 5/12 : RUN pip3 install wheel
 ---> Using cache
 ---> a866d3b1f3e0
Step 6/12 : COPY /src/requirements* /build/
 ---> Using cache
 ---> efc869447227
Step 7/12 : WORKDIR /build
 ---> Using cache
 ---> 53ced29de259
Step 8/12 : RUN pip3 wheel -r requirements_test.txt --no-cache-dir --no-input
 ---> Using cache
 ---> ba6d114360b9
Step 9/12 : RUN pip3 install -r requirements_test.txt -f /build --no-index --no-cache-dir
 ---> Using cache
 ---> ba0ebdace940
Step 10/12 : COPY /src /app
 ---> Using cache
 ---> 9ae5c85bc7cb
Step 11/12 : WORKDIR /app
 ---> Using cache
 ---> aedd8073c9e6
Step 12/12 : CMD ["python3", "manage.py", "test", "--noinput", "--settings=todobackend.settings_test"]
 ---> Using cache
 ---> 3ed637e47056
Successfully built 3ed637e47056
Successfully tagged todobackend-test:latest
```

在前面的例子中，`--target`标志允许您针对多阶段 Dockerfile 中的特定阶段进行构建。尽管我们目前只有一个阶段，但该标志允许我们仅在 Dockerfile 中有多个阶段的情况下构建测试阶段。按照惯例，`docker build`命令会在运行命令的目录中查找`Dockerfile`文件，并且命令末尾的句点指定了当前目录（例如，在本例中是应用程序存储库根目录）作为构建上下文，在构建图像时应将其复制到 Docker 引擎。

使用构建并在本地 Docker Engine 中标记为`todobackend`的映像名称构建的映像，您现在可以从映像启动一个容器，默认情况下将运行`python3 manage.py test`命令，如`CMD`指令所指定的那样：

```
todobackend>  docker run -it --rm todobackend-test
Creating test database for alias 'default'...

Ensure we can create a new todo item
- item has correct title
- item was created
- received 201 created status code
- received location header hyperlink

Ensure we can delete all todo items
- all items were deleted
- received 204 no content status code

Ensure we can delete a todo item
- received 204 no content status code
- the item was deleted

Ensure we can update an existing todo item using PATCH
- item was updated
- received 200 ok status code

Ensure we can update an existing todo item using PUT
- item was updated
- received 200 created status code
----------------------------------------------------------------------
XML: /app/unittests.xml
Name                              Stmts   Miss  Cover
-----------------------------------------------------
todo/__init__.py                      0      0   100%
todo/admin.py                         1      1     0%
todo/migrations/0001_initial.py       5      0   100%
todo/migrations/__init__.py           0      0   100%
todo/models.py                        6      6     0%
todo/serializers.py                   7      0   100%
todo/urls.py                          6      0   100%
todo/views.py                        17      0   100%
-----------------------------------------------------
TOTAL                                42      7    83%
----------------------------------------------------------------------
Ran 12 tests in 0.433s

OK

Destroying test database for alias 'default'...
```

`-it`标志指定以交互式终端运行容器，`--rm`标志将在容器退出时自动删除容器。请注意，所有测试都成功通过，因此我们知道映像中构建的应用程序在至少在当前为应用程序定义的测试方面是良好的状态。

# 配置发布阶段

有了测试阶段，我们现在有了一个映像，其中包含了所有应用程序依赖项，以一种可以在不需要编译或开发依赖项的情况下安装的格式打包，以及我们的应用程序源代码处于一个我们可以轻松验证通过所有测试的状态。

我们需要配置的下一个阶段是发布阶段，它将应用程序源代码和在测试阶段构建的各种应用程序依赖项复制到一个新的生产就绪的发布映像中。由于应用程序依赖项现在以预编译格式可用，因此发布映像不需要开发依赖项或源代码编译工具，这使我们能够创建一个更小、更精简的发布映像，减少了攻击面。

# 安装系统依赖项

要开始创建发布阶段，我们可以在 Dockerfile 的底部添加一个新的`FROM`指令，Docker 将把它视为新阶段的开始：

```
# Test stage
FROM alpine AS test
LABEL application=todobackend
.........
...# Test entrypointCMD ["python3", "manage.py", "test", "--noinput", "--settings=todobackend.settings_test"]

# Release stage
FROM alpine
LABEL application=todobackend

# Install operating system dependencies
RUN apk add --no-cache python3 mariadb-client bash
```

在上面的示例中，您可以看到发布映像再次基于 Alpine Linux 映像，这是一个非常好的选择，因为它的占用空间非常小。您可以看到我们安装了更少的操作系统依赖项，其中包括以下内容：

+   `python3`：由于示例应用程序是一个 Python 应用程序，因此需要 Python 3 解释器和运行时

+   `mariadb-client`：包括与 MySQL 应用程序数据库通信所需的系统库

+   `bash`：用于故障排除和执行入口脚本，我们将在后面的章节中讨论。

请注意，我们只需要安装这些软件包的非开发版本，而不是安装`python3-dev`和`mariadb-dev`软件包，因为我们在测试阶段编译和构建了所有应用程序依赖项的预编译轮。

# 创建应用程序用户

下一步是创建一个应用程序用户，我们的应用程序将作为该用户运行。默认情况下，Docker 容器以 root 用户身份运行，这对于测试和开发目的来说是可以的，但是在生产环境中，即使容器提供了隔离机制，作为非 root 用户运行容器仍被认为是最佳实践：

```
# Test stage
...
...
# Release stage
FROM alpine
LABEL application=todobackend

# Install operating system dependencies
RUN apk add --no-cache python3 mariadb-client bash

# Create app user
RUN addgroup -g 1000 app && \
 adduser -u 1000 -G app -D app
```

在上面的示例中，我们首先创建了一个名为`app`的组，组 ID 为`1000`，然后创建了一个名为`app`的用户，用户 ID 为`1000`，属于`app`组。

# 复制和安装应用程序源代码和依赖项

最后一步是复制先前在测试阶段构建的应用程序源代码和依赖项，将依赖项安装到发布镜像中，然后删除在此过程中使用的任何临时文件。我们还需要将工作目录设置为`/app`，并配置容器以作为前一节中创建的`app`用户运行：

```
# Test stage
...
...
# Release stage
FROM alpine
LABEL application=todobackend

# Install operating system dependencies
RUN apk add --no-cache python3 mariadb-client bash

# Create app user
RUN addgroup -g 1000 app && \
    adduser -u 1000 -G app -D app

# Copy and install application source and pre-built dependencies
COPY --from=test --chown=app:app /build /build
COPY --from=test --chown=app:app /app /app
RUN pip3 install -r /build/requirements.txt -f /build --no-index --no-cache-dir
RUN rm -rf /build

# Set working directory and application user
WORKDIR /app
USER app
```

您首先使用`COPY`指令和`--from`标志，告诉 Docker 在`--from`标志指定的阶段查找要复制的文件。在这里，我们将测试阶段镜像中的`/build`和`/app`文件夹复制到发布阶段中同名的文件夹，并配置`--chown`标志以将这些复制的文件夹的所有权更改为应用程序用户。然后我们使用`pip3`命令仅安装`requirements.txt`文件中指定的核心要求（您不需要`requirements_test.txt`中指定的依赖项来运行应用程序），使用`--no-index`标志禁用 PIP 连接到互联网下载软件包，而是使用`-f`标志引用的`/build`文件夹来查找先前在测试阶段构建并复制到此文件夹的依赖项。我们还指定`--no-cache-dir`标志以避免在本地文件系统中不必要地缓存软件包，并在安装完成后删除`/build`文件夹。

最后，您将工作目录设置为`/app`，并通过指定`USER`指令配置容器以`app`用户身份运行。

# 构建和运行发布镜像

现在我们已经完成了 Dockerfile 发布阶段的配置，是时候构建我们的新发布镜像，并验证我们是否能成功运行我们的应用程序。

要构建镜像，我们可以使用`docker build`命令，因为发布阶段是 Dockerfile 的最后阶段，所以你不需要针对特定阶段进行目标设置，就像我们之前为测试阶段所做的那样：

```
> docker build -t todobackend-release . Sending build context to Docker daemon 312.8kB
Step 1/22 : FROM alpine AS test
 ---> 3fd9065eaf02
...
...
Step 13/22 : FROM alpine
 ---> 3fd9065eaf02
Step 14/22 : LABEL application=todobackend
 ---> Using cache
 ---> afdd1dee07d7
Step 15/22 : RUN apk add --no-cache python3 mariadb-client bash
 ---> Using cache
 ---> dfe0b6487459
Step 16/22 : RUN addgroup -g 1000 app && adduser -u 1000 -G app -D app
 ---> Running in d75df9cadb1c
Removing intermediate container d75df9cadb1c
 ---> ac26efcbfea0
Step 17/22 : COPY --from=test --chown=app:app /build /build
 ---> 1f177a92e2c9
Step 18/22 : COPY --from=test --chown=app:app /app /app
 ---> ba8998a31f1d
Step 19/22 : RUN pip3 install -r /build/requirements.txt -f /build --no-index --no-cache-dir
 ---> Running in afc44357fae2
Looking in links: /build
Collecting Django==2.0 (from -r /build/requirements.txt (line 1))
Collecting django-cors-headers==2.1.0 (from -r /build/requirements.txt (line 2))
Collecting djangorestframework==3.7.3 (from -r /build/requirements.txt (line 3))
Collecting mysql-connector-python==8.0.11 (from -r /build/requirements.txt (line 4))
Collecting pytz==2017.3 (from -r /build/requirements.txt (line 5))
Collecting uwsgi (from -r /build/requirements.txt (line 6))
Collecting protobuf>=3.0.0 (from mysql-connector-python==8.0.11->-r /build/requirements.txt (line 4))
Requirement already satisfied: setuptools in /usr/lib/python3.6/site-packages (from protobuf>=3.0.0->mysql-connector-python==8.0.11->-r /build/requirements.txt (line 4)) (28.8.0)
Collecting six>=1.9 (from protobuf>=3.0.0->mysql-connector-python==8.0.11->-r /build/requirements.txt (line 4))
Installing collected packages: pytz, Django, django-cors-headers, djangorestframework, six, protobuf, mysql-connector-python, uwsgi
Successfully installed Django-2.0 django-cors-headers-2.1.0 djangorestframework-3.7.3 mysql-connector-python-8.0.11 protobuf-3.6.0 pytz-2017.3 six-1.11.0 uwsgi-2.0.17
Removing intermediate container afc44357fae2
 ---> ab2bcf89fe13
Step 20/22 : RUN rm -rf /build
 ---> Running in 8b8006ea8636
Removing intermediate container 8b8006ea8636
 ---> ae7f157d29d1
Step 21/22 : WORKDIR /app
Removing intermediate container fbd49835ca49
 ---> 55856af393f0
Step 22/22 : USER app
 ---> Running in d57b2cb9bb69
Removing intermediate container d57b2cb9bb69
 ---> 8170e923b09a
Successfully built 8170e923b09a
Successfully tagged todobackend-release:latest
```

在这一点上，我们可以运行位于发布镜像中的 Django 应用程序，但是你可能想知道它是如何工作的。当我们之前运行`python3 manage.py runserver`命令时，它启动了一个本地开发 Web 服务器，这在生产用户案例中是不推荐的，所以我们需要一个替代的 Web 服务器来在生产环境中运行我们的应用程序。

你可能已经在`requirements.txt`文件中注意到了一个名为`uwsgi`的包——这是一个非常流行的 Web 服务器，可以在生产中使用，并且对于我们的用例非常方便，可以通过 PIP 安装。这意味着`uwsgi`已经作为 Web 服务器在我们的发布容器中可用，并且可以用来提供示例应用程序。

```
> docker run -it --rm -p 8000:8000 todobackend-release uwsgi \
    --http=0.0.0.0:8000 --module=todobackend.wsgi --master *** Starting uWSGI 2.0.17 (64bit) on [Tue Jul 3 11:44:44 2018] *
compiled with version: 6.4.0 on 02 July 2018 14:34:31
os: Linux-4.9.93-linuxkit-aufs #1 SMP Wed Jun 6 16:55:56 UTC 2018
nodename: 5be4dd1ddab0
machine: x86_64
clock source: unix
detected number of CPU cores: 1
current working directory: /app
detected binary path: /usr/bin/uwsgi
!!! no internal routing support, rebuild with pcre support !!!
your memory page size is 4096 bytes
detected max file descriptor number: 1048576
lock engine: pthread robust mutexes
thunder lock: disabled (you can enable it with --thunder-lock)
uWSGI http bound on 0.0.0.0:8000 fd 4
uwsgi socket 0 bound to TCP address 127.0.0.1:35765 (port auto-assigned) fd 3
Python version: 3.6.3 (default, Nov 21 2017, 14:55:19) [GCC 6.4.0]
* Python threads support is disabled. You can enable it with --enable-threads *
Python main interpreter initialized at 0x55e9f66ebc80
your server socket listen backlog is limited to 100 connections
your mercy for graceful operations on workers is 60 seconds
mapped 145840 bytes (142 KB) for 1 cores
* Operational MODE: single process *
WSGI app 0 (mountpoint='') ready in 0 seconds on interpreter 0x55e9f66ebc80 pid: 1 (default app)
* uWSGI is running in multiple interpreter mode *
spawned uWSGI master process (pid: 1)
spawned uWSGI worker 1 (pid: 7, cores: 1)
spawned uWSGI http 1 (pid: 8)
```

我们使用`-p`标志将容器上的端口`8000`映射到主机上的端口`8000`，并执行`uwsgi`命令，传入各种配置标志，以在端口`8000`上运行应用程序，并指定`todobackend.wsgi`模块作为`uwsgi`提供的应用程序。

Web 服务器网关接口（WSGI）是 Python 应用程序用来与 Web 服务器交互的标准接口。每个 Django 应用程序都包括一个用于与 Web 服务器通信的 WSGI 模块，可以通过`<application-name>.wsgi`访问。

在这一点上，你可以浏览`http://localhost:8000`，虽然应用程序确实返回了一个响应，但你会发现 Web 服务器和应用程序缺少一堆静态内容：

![](img/1666a16d-f8ad-4509-974d-aca192694abf.png)

问题在于，当你运行 Django 开发 Web 服务器时，Django 会自动生成静态内容，但是当你在生产环境中与外部 Web 服务器一起运行应用程序时，你需要自己生成静态内容。我们将在本章后面学习如何做到这一点，但是现在，你可以使用`curl`来验证 API 是否可用：

```
> curl -s localhost:8000/todos | jq
[
 {
 "url": "http://localhost:8000/todos/1",
 "title": "Walk the dog",
 "completed": false,
 "order": 1
 },
 {
 "url": "http://localhost:8000/todos/2",
 "title": "Wash the car",
 "completed": true,
 "order": 2
 }
]
```

这里需要注意的一点是，尽管我们是从头开始构建 Docker 镜像，但是 todobackend 数据与我们在第一章加载的数据相同。问题在于，第一章中创建的 SQLite 数据库位于`src`文件夹中，名为`db.sqlite3`。显然，在构建过程中我们不希望将此文件复制到我们的 Docker 镜像中，而要实现这一点的一种方法是在存储库的根目录创建一个`.dockerignore`文件：

```
# Ignore SQLite database files
/***.sqlite3

# Ignore test output and private code coverage files
/*.xml
/.coverage

# Ignore compiled Python source files
/*.pyc
/pycache# Ignore macOS directory metadata files
/.DS_Store

```

`.dockerignore`文件的工作方式类似于 Git 存储库中的`.gitignore`，用于从 Docker 构建上下文中排除文件。因为`db.sqlite3`文件位于子文件夹中，我们使用通配符 globing 模式`**`（请注意，这与`.gitignore`的行为不同，默认情况下进行 globing），这意味着我们递归地排除与通配符模式匹配的任何文件。我们还排除任何具有`.xml`扩展名的测试输出文件，代码覆盖文件，`__pycache__`文件夹以及任何具有`.pyc`扩展名的编译 Python 文件，这些文件是打算在运行时动态生成的。

如果您现在重新构建 Docker 镜像，并在本地端口`8000`上启动`uwsgi`Web 服务器，当您浏览应用程序（`http://localhost:8000`）时，您将会得到一个不同的错误：

![](img/c7ab5b6d-1d1b-47d8-8242-5e7c51cc22c4.png)

现在的问题是 todobackend 应用程序没有数据库存在，因此应用程序失败，因为它无法找到存储待办事项的表。为了解决这个问题，我们现在需要集成一个外部数据库引擎，这意味着我们需要一个解决方案来在本地使用多个容器。

# 使用 Docker Compose 测试和构建应用程序

在上一节中，您使用 Docker 命令执行了以下任务：

+   构建一个测试镜像

+   运行测试

+   构建一个发布镜像

+   运行应用程序

每次我们运行 Docker 命令时，都需要提供相当多的配置，并且试图记住需要运行的各种命令已经开始变得困难。除此之外，我们还发现，要启动应用程序的发布镜像，我们需要有一个操作的外部数据库。对于本地测试用例，运行另一个容器中的外部数据库是一个很好的方法，但是通过运行一系列带有许多不同输入参数的 Docker 命令来协调这一点很快变得难以管理。

**Docker Compose**是一个工具，允许您使用声明性方法编排多容器环境，使得编排可能需要多个容器的复杂工作流程变得更加容易。按照惯例，Docker Compose 会在当前目录中寻找一个名为`docker-compose.yml`的文件，所以让我们在`todobackend`存储库的根目录下创建这个文件，与我们的`Dockerfile`放在一起。

```
version: '2.4'

services:
  test:
    build:
      context: .
      dockerfile: Dockerfile
      target: test
  release:
    build:
      context: .
      dockerfile: Dockerfile
```

Docker Compose 文件是用 YAML 格式定义的，需要正确的缩进来推断父对象、同级对象和子对象或属性之间的正确关系。如果您以前没有使用过 YAML，可以查看[Ansible YAML Syntax guide](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)，这是一个对 YAML 格式的简要介绍。您也可以使用在线的 YAML linting 工具，比如 http://www.yamllint.com/来检查您的 YAML，或者在您喜欢的文本编辑器中安装 YAML 支持。

首先，我们指定了`version`属性，这是必需的，引用了我们正在使用的 Compose 文件格式语法的版本。如果您正在使用 Docker 进行本地开发和构建任务，我建议使用 Compose 文件格式的 2.x 版本，因为它包括一些有用的功能，比如对依赖服务进行健康检查，我们很快将学习如何使用。如果您正在使用 Docker Swarm 来运行您的容器，那么您应该使用 Compose 文件格式的 3.x 版本，因为这个版本支持一些与管理和编排 Docker Swarm 相关的功能。

如果您选择使用 3.x 版本，您的应用程序需要更加健壮，以处理诸如数据库在应用程序启动时不可用的情况（参见[`docs.docker.com/compose/startup-order/`](https://docs.docker.com/compose/startup-order/)），这是我们在本章后面将遇到的一个问题。

接下来，我们指定`services`属性，它定义了在我们的 Docker Compose 环境中运行的一个或多个服务。在前面的示例中，我们创建了两个服务，对应于工作流程的测试和发布阶段，然后为每个服务添加了一个`build`属性，它定义了我们希望如何为每个服务构建 Docker 镜像。请注意，`build`属性基于我们传递给`docker build`命令的各种标志，例如，当我们构建测试阶段镜像时，我们将构建上下文设置为本地文件夹，使用本地 Dockerfile 作为构建规范的图像，并仅针对测试阶段构建图像。我们不是在每次运行 Docker 命令时命令式地指定这些设置，而是声明性地定义了构建过程的期望配置，这是一个重要的区别。

当然，我们需要运行一个命令来实际构建这些服务，您可以在`todobackend`存储库的根目录运行`docker-compose build`命令。

```
> docker-compose build test
Building test
Step 1/12 : FROM alpine AS test
 ---> 3fd9065eaf02
Step 2/12 : LABEL application=todobackend
 ---> Using cache
 ---> 23e0c2657711
...
...
Step 12/12 : CMD ["python3", "manage.py", "test", "--noinput", "--settings=todobackend.settings_test"]
 ---> Running in 1ac9bded79bf
Removing intermediate container 1ac9bded79bf
 ---> f42d0d774c23

Successfully built f42d0d774c23
Successfully tagged todobackend_test:latest
```

你可以看到运行`docker-compose build test`命令实现了我们之前运行的`docker build`命令的等效效果，然而，我们不需要向`docker-compose`命令传递任何构建选项或配置，因为我们所有的特定设置都包含在`docker-compose.yml`文件中。

如果现在要从新构建的镜像运行测试，可以执行`docker-compose run`命令：

```
> docker-compose run test
Creating network "todobackend_default" with the default driver
nosetests --verbosity=2 --nologcapture --with-coverage --cover-package=todo --with-spec --spec-color --with-xunit --xunit-file=./unittests.xml --cover-xml --cover-xml-file=./coverage.xml
Creating test database for alias 'default'...

Ensure we can create a new todo item
- item has correct title
- item was created
- received 201 created status code
- received location header hyperlink
...
...
...
...
Ran 12 tests in 0.316s

OK

Destroying test database for alias 'default'...
```

您还可以扩展 Docker Compose 文件，以向服务添加端口映射和命令配置，如下例所示：

```
version: '2.4'

services:
  test:
    build:
      context: .
      dockerfile: Dockerfile
      target: test
  release:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
 - 8000:8000
 command:
 - uwsgi
 - --http=0.0.0.0:8000
 - --module=todobackend.wsgi
 - --master
```

在这里，我们指定当运行发布服务时，它应该在主机的 8000 端口和容器的 8000 端口之间创建静态端口映射，并将我们之前使用的`uwsgi`命令传递给发布容器。如果现在使用`docker-compose up`命令运行发布阶段，请注意 Docker Compose 将自动为服务构建镜像（如果尚不存在），然后启动服务：

```
> docker-compose up release
Building release
Step 1/22 : FROM alpine AS test
 ---> 3fd9065eaf02
Step 2/22 : LABEL application=todobackend
 ---> Using cache
 ---> 23e0c2657711
...
...

Successfully built 5b20207e3e9c
Successfully tagged todobackend_release:latest
WARNING: Image for service release was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Creating todobackend_release_1 ... done
Attaching to todobackend_release_1
...
...
release_1 | *** uWSGI is running in multiple interpreter mode *
release_1 | spawned uWSGI master process (pid: 1)
release_1 | spawned uWSGI worker 1 (pid: 6, cores: 1)
release_1 | spawned uWSGI http 1 (pid: 7)
```

通常，您使用`docker-compose up`命令来运行长时间运行的服务，使用`docker-compose run`命令来运行短暂的任务。您还不能覆盖传递给`docker-compose up`的命令参数，而可以将命令覆盖传递给`docker-compose run`命令。

# 使用 Docker Compose 添加数据库服务

为了解决运行发布图像时出现的应用程序错误，我们需要运行一个应用程序可以连接到的数据库，并确保应用程序配置为使用该数据库。

我们可以通过使用 Docker Compose 添加一个名为`db`的新服务来实现这一点，该服务基于官方的 MySQL 服务器容器：

```
version: '2.4'

services:
  test:
    build:
      context: .
      dockerfile: Dockerfile
      target: test
  release:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - 8000:8000
    command:
      - uwsgi
      - --http=0.0.0.0:8000
      - --module=todobackend.wsgi
      - --master
  db:
 image: mysql:5.7
 environment:
 MYSQL_DATABASE: todobackend
 MYSQL_USER: todo
 MYSQL_PASSWORD: password
 MYSQL_ROOT_PASSWORD: password
```

请注意，您可以使用`image`属性指定外部图像，并且环境设置将使用数据库名为 todobackend、用户名、密码和根密码配置 MySQL 容器。

现在，您可能想知道如何配置我们的应用程序以使用 MySQL 和新的`db`服务。todobackend 应用程序包括一个名为`src/todobackend/settings_release.py`的设置文件，该文件配置了 MySQL 作为数据库后端的支持：

```
# Import base settings
from .settings import *
import os

# Disable debug
DEBUG = True

# Set secret key
SECRET_KEY = os.environ.get('SECRET_KEY', SECRET_KEY)

# Must be explicitly specified when Debug is disabled
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '*').split(',')

# Database settings
DATABASES = {
    'default': {
        'ENGINE': 'mysql.connector.django',
        'NAME': os.environ.get('MYSQL_DATABASE','todobackend'),
        'USER': os.environ.get('MYSQL_USER','todo'),
        'PASSWORD': os.environ.get('MYSQL_PASSWORD','password'),
        'HOST': os.environ.get('MYSQL_HOST','localhost'),
        'PORT': os.environ.get('MYSQL_PORT','3306'),
    },
    'OPTIONS': {
      'init_command': "SET sql_mode='STRICT_TRANS_TABLES'"
    }
}

STATIC_ROOT = os.environ.get('STATIC_ROOT', '/public/static')
MEDIA_ROOT = os.environ.get('MEDIA_ROOT', '/public/media')
```

`DATABASES`设置包括一个配置，指定了`mysql.connector.django`引擎，该引擎提供了对 MySQL 的支持，覆盖了默认的 SQLite 驱动程序，并且您可以看到数据库名称、用户名和密码可以通过`os.environ.get`调用从环境中获取。还要注意`STATIC_ROOT`设置-这是 Django 查找静态内容（如 HTML、CSS、JavaScript 和图像）的位置-默认情况下，如果未定义此环境变量，Django 将在`/public/static`中查找。正如我们之前看到的，目前我们的 Web 应用程序缺少这些内容，因此在以后修复缺少内容问题时，请记住这个设置。

现在您了解了如何配置 todobackend 应用程序以支持 MySQL 数据库，让我们修改 Docker Compose 文件以使用`db`服务：

```
version: '2.4'

services:
  test:
    build:
      context: .
      dockerfile: Dockerfile
      target: test
  release:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - 8000:8000
 depends_on:
 db:
 condition: service_healthy
    environment:
 DJANGO_SETTINGS_MODULE: todobackend.settings_release
 MYSQL_HOST: db
 MYSQL_USER: todo
 MYSQL_PASSWORD: password
    command:
      - uwsgi
      - --http=0.0.0.0:8000
      - --module=todobackend.wsgi
      - --master
  db:
    image: mysql:5.7
 healthcheck:
 test: mysqlshow -u $$MYSQL_USER -p$$MYSQL_PASSWORD
      interval: 3s
      retries: 10
    environment:
      MYSQL_DATABASE: todobackend
      MYSQL_USER: todo
      MYSQL_PASSWORD: password
      MYSQL_ROOT_PASSWORD: password
```

我们首先配置`release`服务上的`environment`属性，该属性配置了将传递给容器的环境变量。请注意，对于 Django 应用程序，您可以配置`DJANGO_SETTINGS_MODULE`环境变量以指定应该使用哪些设置，这使您可以使用添加了 MySQL 支持的`settings_release`配置。此配置还允许您使用环境变量来指定 MySQL 数据库设置，这些设置必须与`db`服务的配置相匹配。

接下来，我们为`release`服务配置`depends_on`属性，该属性描述了服务可能具有的任何依赖关系。因为应用程序在启动之前必须与数据库建立有效连接，所以我们指定了`service_healthy`的条件，这意味着在 Docker Compose 尝试启动`release`服务之前，`db`服务必须通过 Docker 健康检查。为了配置`db`服务上的 Docker 健康检查，我们配置了`healthcheck`属性，它将配置 Docker 运行`db`服务容器内由`test`参数指定的命令来验证服务健康，并重试此命令，每 3 秒一次，最多重试 10 次，直到`db`服务健康为止。对于这种情况，我们使用`mysqlshow`命令，它只有在 MySQL 进程接受连接时才会返回成功的零退出代码。由于 Docker Compose 将单个美元符号解释为应该在 Docker Compose 文件中评估和替换的环境变量，我们使用双美元符号转义`test`命令中引用的环境变量，以确保该命令会直接执行`mysqlshow -u $MYSQL_USER -p$MYSQL_PASSWORD`。

在这一点上，我们可以通过在运行`release`服务的终端中按下*Ctrl* + *C*并输入`docker-compose down -v`命令（`-v`标志还将删除 Docker Compose 创建的任何卷）来拆除当前环境，然后执行`docker-compose up release`命令来测试更改：

```
> docker-compose down -v
Removing todobackend_release_1 ... done
Removing todobackend_test_run_1 ... done
Removing network todobackend_default
> docker-compose up release Creating network "todobackend_default" with the default driver
Pulling db (mysql:5.7)...
5.7: Pulling from library/mysql
683abbb4ea60: Pull complete
0550d17aeefa: Pull complete
7e26605ddd77: Pull complete
9882737bd15f: Pull complete
999c06ab75f6: Pull complete
c71d695f9937: Pull complete
c38f847c1491: Pull complete
74f9c61f40bf: Pull complete
30b252a90a12: Pull complete
9f92ebb7da55: Pull complete
90303981d276: Pull complete
Digest: sha256:1203dfba2600f140b74e375a354b1b801fa1b32d6f80fdee5f155d1e9f38c841
Status: Downloaded newer image for mysql:5.7
Creating todobackend_db_1 ... done
Creating todobackend_release_1 ... done
Attaching to todobackend_release_1
release_1 | *** Starting uWSGI 2.0.17 (64bit) on [Thu Jul 5 07:45:38 2018] *
release_1 | compiled with version: 6.4.0 on 04 July 2018 11:33:09
release_1 | os: Linux-4.9.93-linuxkit-aufs #1 SMP Wed Jun 6 16:55:56 UTC 2018
...
... *** uWSGI is running in multiple interpreter mode *
release_1 | spawned uWSGI master process (pid: 1)
release_1 | spawned uWSGI worker 1 (pid: 7, cores: 1)
release_1 | spawned uWSGI http 1 (pid: 8)
```

在上面的示例中，请注意，Docker Compose 会根据`image`属性自动拉取 MySQL 5.7 镜像，然后启动`db`服务。这将需要 15-30 秒，在此期间，Docker Compose 正在等待 Docker 报告`db`服务的健康状况。每 3 秒，Docker 运行在健康检查中配置的`mysqlshow`命令，不断重复此过程，直到命令返回成功的退出代码（即退出代码为`0`），此时 Docker 将标记容器为健康。只有在这一点上，Docker Compose 才会启动`release`服务，假设`db`服务完全可操作，`release`服务应该会成功启动。

如果您再次浏览`http://localhost:8000/todos`，您会发现即使我们添加了一个`db`服务并配置了发布服务以使用这个数据库，您仍然会收到之前在上一个截图中看到的`no such table`错误。

# 运行数据库迁移

我们仍然收到有关缺少表的错误，原因是因为我们尚未运行数据库迁移以建立应用程序期望存在的所需数据库架构。请记住，我们在本地使用`python3 manage.py migrate`命令来运行这些迁移，因此我们需要在我们的 Docker 环境中执行相同的操作。

如果您再次拆除环境，按下*Ctrl* + *C*并运行`docker-compose down -v`，一个方法是使用`docker-compose run`命令：

```
> docker-compose down -v ...
...
> docker-compose run release python3 manage.py migrate
Creating network "todobackend_default" with the default driver
Creating todobackend_db_1 ... done
Traceback (most recent call last):
  File "/usr/lib/python3.6/site-packages/mysql/connector/network.py", line 515, in open_connection
    self.sock.connect(sockaddr)
ConnectionRefusedError: [Errno 111] Connection refused
...
...
```

在上面的示例中，请注意，当您使用`docker-compose run`命令时，Docker Compose 不支持我们之前在运行`docker-compose up`时观察到的健康检查行为。这意味着您可以采取以下两种方法：

+   确保您首先运行`docker-compose up release`，然后运行`docker-compose run python3 manage.py migrate` - 这将使您的应用程序处于一种状态，直到迁移完成之前都会引发错误。

+   将迁移定义为一个单独的服务，称为`migrate`，依赖于`db`服务，启动`migrate`服务，该服务将执行迁移并退出，然后启动应用程序。

尽管很快您会看到，选项 1 更简单，但选项 2 更健壮，因为它确保在启动应用程序之前数据库处于正确的状态。选项 2 也符合我们稍后在本书中在 AWS 中编排运行数据库迁移时将采取的方法，因此我们现在将实施选项 2。

以下示例演示了我们需要进行的更改，以将迁移作为一个单独的服务运行：

```
version: '2.4'

services:
  test:
    build:
      context: .
      dockerfile: Dockerfile
      target: test
  release:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      DJANGO_SETTINGS_MODULE: todobackend.settings_release
      MYSQL_HOST: db
      MYSQL_USER: todo
      MYSQL_PASSWORD: password
  app:
 extends:
 service: release
 depends_on:
 db:
 condition: service_healthy
 ports:
 - 8000:8000
 command:
 - uwsgi
 - --http=0.0.0.0:8000
 - --module=todobackend.wsgi
 - --master
  migrate:
 extends:
 service: release
 depends_on:
 db:
 condition: service_healthy
 command:
 - python3
 - manage.py
 - migrate
 - --no-input
  db:
    image: mysql:5.7
    healthcheck:
      test: mysqlshow -u $$MYSQL_USER -p$$MYSQL_PASSWORD
      interval: 3s
      retries: 10
    environment:
      MYSQL_DATABASE: todobackend
      MYSQL_USER: todo
      MYSQL_PASSWORD: password
      MYSQL_ROOT_PASSWORD: password
```

在上面的示例中，请注意，除了`migrate`服务，我们还添加了一个名为`app`的新服务。原因是我们希望从`release`服务扩展`migrate`（如`extends`参数所定义），以便它将继承发布映像和发布服务设置，但是扩展另一个服务的一个限制是您不能扩展具有`depends_on`语句的服务。这要求我们将`release`服务更多地用作其他服务继承的基本配置，并将`depends_on`、`ports`和`command`参数从发布服务转移到新的`app`服务。

有了这个配置，我们可以拆除环境并建立我们的新环境，就像以下示例中演示的那样：

```
> docker-compose down -v ...
...
> docker-compose up migrate
Creating network "todobackend_default" with the default driver
Building migrate
Step 1/24 : FROM alpine AS test
 ---> 3fd9065eaf02
...
...
Successfully built 5b20207e3e9c
Successfully tagged todobackend_migrate:latest
WARNING: Image for service migrate was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Creating todobackend_db_1 ... done
Creating todobackend_migrate_1 ... done
Attaching to todobackend_migrate_1
migrate_1 | Operations to perform:
migrate_1 | Apply all migrations: admin, auth, contenttypes, sessions, todo
migrate_1 | Running migrations:
migrate_1 | Applying contenttypes.0001_initial... OK
migrate_1 | Applying auth.0001_initial... OK
migrate_1 | Applying admin.0001_initial... OK
migrate_1 | Applying admin.0002_logentry_remove_auto_add... OK
migrate_1 | Applying contenttypes.0002_remove_content_type_name... OK
migrate_1 | Applying auth.0002_alter_permission_name_max_length... OK
migrate_1 | Applying auth.0003_alter_user_email_max_length... OK
migrate_1 | Applying auth.0004_alter_user_username_opts... OK
migrate_1 | Applying auth.0005_alter_user_last_login_null... OK
migrate_1 | Applying auth.0006_require_contenttypes_0002... OK
migrate_1 | Applying auth.0007_alter_validators_add_error_messages... OK
migrate_1 | Applying auth.0008_alter_user_username_max_length... OK
migrate_1 | Applying auth.0009_alter_user_last_name_max_length... OK
migrate_1 | Applying sessions.0001_initial... OK
migrate_1 | Applying todo.0001_initial... OK
todobackend_migrate_1 exited with code 0
> docker-compose up app
Building app
Step 1/24 : FROM alpine AS test
 ---> 3fd9065eaf02
...
...
Successfully built 5b20207e3e9c
Successfully tagged todobackend_app:latest
WARNING: Image for service app was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
todobackend_db_1 is up-to-date
Creating todobackend_app_1 ... done
Attaching to todobackend_app_1
app_1 | *** Starting uWSGI 2.0.17 (64bit) on [Thu Jul 5 11:21:00 2018] *
app_1 | compiled with version: 6.4.0 on 04 July 2018 11:33:09
app_1 | os: Linux-4.9.93-linuxkit-aufs #1 SMP Wed Jun 6 16:55:56 UTC 2018
...
...
```

在上面的示例中，请注意 Docker Compose 为每个服务构建新的映像，但是由于每个服务都扩展了`release`服务，因此这些构建非常快速。当您启动`migrate`服务等待`db`服务的健康检查通过时，您将观察到 15-30 秒的延迟，之后将运行迁移，创建 todobackend 应用程序期望的适当模式和表。启动`app`服务后，您应该能够与 todobackend API 交互而不会收到任何错误：

```
> curl -s localhost:8000/todos | jq
[]
```

# 生成静态网页内容

如果您浏览`http://localhost:8000/todos`，尽管应用程序不再返回错误，但网页的格式仍然是错误的。问题在于 Django 要求您运行一个名为`collectstatic`的单独的`manage.py`管理任务，它会生成静态内容并将其放置在`STATIC_ROOT`设置定义的位置。我们应用程序的发布设置将文件位置定义为`/public/static`，因此我们需要在应用程序启动之前运行`collectstatic`任务。请注意，Django 从`/static` URL 路径提供所有静态内容，例如`http://localhost:8000/static`。

有几种方法可以解决这个问题：

+   创建一个在启动时运行并在启动应用程序之前执行`collectstatic`任务的入口脚本。

+   创建一个外部卷并运行一个容器，执行`collectstatic`任务，在卷中生成静态文件。然后启动应用程序，挂载外部卷，确保它可以访问静态内容。

这两种方法都是有效的，但是为了介绍 Docker 卷的概念以及你如何在 Docker Compose 中使用它们，我们将采用第二种方法。

要在 Docker Compose 中定义卷，你可以使用顶层的`volumes`参数，它允许你定义一个或多个命名卷。

```
version: '2.4'

volumes:
 public:
 driver: local

services:
  test:
    ...
    ...
  release:
    ...
    ...
  app:
    extends:
      service: release
    depends_on:
      db:
        condition: service_healthy
    volumes:
 - public:/public
    ports:
      - 8000:8000
    command:
      - uwsgi
      - --http=0.0.0.0:8000
      - --module=todobackend.wsgi
      - --master
 - --check-static=/public
  migrate:
    ...
    ...
  db:
    ...
    ...
```

在上面的例子中，你添加了一个名为`public`的卷，并将驱动程序指定为本地，这意味着它是一个标准的 Docker 卷。然后你在 app 服务中使用`volumes`参数将 public 卷挂载到容器中的`/public`路径，最后你配置`uwsgi`来从`/public`路径为静态内容提供服务，这避免了昂贵的应用程序调用 Python 解释器来提供静态内容。

在销毁当前的 Docker Compose 环境后，生成静态内容只需要使用`docker-compose run`命令。

```
> docker-compose down -v ...
...
> docker-compose up migrate
...
...
> docker-compose run app python3 manage.py collectstatic --no-input
Starting todobackend_db_1 ... done
Copying '/usr/lib/python3.6/site-packages/django/contrib/admin/static/admin/js/prepopulate.js'
Traceback (most recent call last):
  File "manage.py", line 15, in <module>
    execute_from_command_line(sys.argv)
  File "/usr/lib/python3.6/site-packages/django/core/management/__init__.py", line 371, in execute_from_command_line
    utility.execute()
...
...
PermissionError: [Errno 13] Permission denied: '/public/static'
```

在上面的例子中，`collectstatic`任务失败，因为默认情况下卷是以 root 创建的，而容器是以 app 用户运行的。为了解决这个问题，我们需要在`Dockerfile`中预先创建`/public`文件夹，并将 app 用户设置为该文件夹的所有者。

```
# Test stage
...
...
# Release stage
FROM alpine
LABEL application=todobackend
...
...
# Copy and install application source and pre-built dependencies
COPY --from=test --chown=app:app /build /build
COPY --from=test --chown=app:app /app /app
RUN pip3 install -r /build/requirements.txt -f /build --no-index --no-cache-dir
RUN rm -rf /build

# Create public volume
RUN mkdir /public
RUN chown app:app /public
VOLUME /public

# Set working directory and application user
WORKDIR /app
USER app
```

请注意，上面显示的方法仅适用于使用 Docker 卷挂载创建的卷，这是 Docker Compose 在你没有在 Docker Engine 上指定主机路径时使用的方法。如果你指定了主机路径，卷将被绑定挂载，这会导致卷默认具有 root 所有权，除非你在主机上预先创建具有正确权限的路径。当我们使用弹性容器服务时，我们将在以后遇到这个问题，所以请记住这一点。

因为你修改了 Dockerfile，你需要告诉 Docker Compose 重新构建所有镜像，你可以使用`docker-compose build`命令来实现。

```
> docker-compose down -v
...
...
> docker-compose build Building test
Step 1/13 : FROM alpine AS test
...
...
Building release
...
...
Building app
...
...
Building migrate
...
...
> docker-compose up migrate
...
...
> docker-compose run app python3 manage.py collectstatic --no-input
Copying '/usr/lib/python3.6/site-packages/django/contrib/admin/static/admin/js/prepopulate.js'
Copying '/usr/lib/python3.6/site-packages/django/contrib/admin/static/admin/js/SelectFilter2.js'
Copying '/usr/lib/python3.6/site-packages/django/contrib/admin/static/admin/js/change_form.js'
Copying '/usr/lib/python3.6/site-packages/django/contrib/admin/static/admin/js/inlines.min.js'
...
...
> docker-compose up app
```

如果你现在浏览`http://localhost:8000`，正确的静态内容应该被显示出来。

在 Docker Compose 中定义本地卷时，当你运行`docker-compose down -v`命令时，卷将自动销毁。如果你希望独立于 Docker Compose 持久存储，你可以定义一个外部卷，然后你需要负责创建和销毁它。更多详情请参阅[`docs.docker.com/compose/compose-file/compose-file-v2/#external`](https://docs.docker.com/compose/compose-file/compose-file-v2/#external)。

# 创建验收测试

现在应用程序已正确配置，为发布阶段配置的最后一个任务是定义验收测试，以验证应用程序是否按预期工作。验收测试的目的是确保您构建的发布镜像在尽可能接近生产环境的环境中工作，在本地 Docker 环境的约束条件下。至少，如果您的应用程序是 Web 应用程序或 API 服务，比如 todobackend 应用程序，您可能只需验证应用程序返回有效的 HTTP 响应，或者您可能运行关键功能，比如创建项目、更新项目和删除项目。

对于 todobackend 应用程序，我们将创建一些基本测试来演示这种方法，使用一个名为 BATS（Bash 自动化测试系统）的工具。BATS 非常适合更喜欢使用 bash 的系统管理员，并利用开箱即用的工具来执行测试。

要开始使用 BATS，我们需要在**todobackend**存储库的`src`文件夹中使用 BATS 语法创建一个名为`acceptance.bats`的测试脚本，您可以在[`github.com/sstephenson/bats`](https://github.com/sstephenson/bats)上了解更多信息：

```
setup() {
  url=${APP_URL:-localhost:8000}
  item='{"title": "Wash the car", "order": 1}'
  location='Location: ([^[:space:]]*)'
  curl -X DELETE $url/todos
}

@test "todobackend root" {
  run curl -oI -s -w "%{http_code}" $APP_URL
  [ $status = 0 ]
  [ $output = 200 ]
}

@test "todo items returns empty list" {
  run jq '. | length' <(curl -s $url/todos)
  [ $output = 0 ]
}

@test "create todo item" {
  run curl -i -X POST -H "Content-Type: application/json" $url/todos -d "$item"
  [ $status = 0 ]
  [[ $output =~ "201 Created" ]] || false
  [[ $output =~ $location ]] || false
  [ $(curl ${BASH_REMATCH[1]} | jq '.title') = $(echo "$item" | jq '.title') ]
}

@test "delete todo item" {
  run curl -i -X POST -H "Content-Type: application/json" $url/todos -d "$item"
  [ $status = 0 ]
  [[ $output =~ $location ]] || false
  run curl -i -X DELETE ${BASH_REMATCH[1]}
  [ $status = 0 ]
  [[ $output =~ "204 No Content" ]] || false
  run jq '. | length' <(curl -s $APP_URL/todos)
  [ $output = 0 ]
}
```

BATS 文件包括一个`setup()`函数和一些测试用例，每个测试用例都以`@test`标记为前缀。`setup()`函数是一个特殊的函数，在每个测试用例运行之前都会运行，用于定义公共变量并确保应用程序状态在每个测试之前保持一致。您可以看到我们设置了一些在各种测试用例中使用的变量：

+   `url`：定义了要测试的应用程序的 URL。这由`APP_URL`环境变量定义，默认为`localhost:8000`，如果未定义`APP_URL`。

+   `item`：以 JSON 格式定义了一个测试 Todo 项，该项在测试期间通过 Todos API 创建。

+   `location`：定义了一个正则表达式，用于定位和捕获在创建 Todo 项时返回的 HTTP 响应中的 Location 标头的值。正则表达式的`([^[:space:]]*)`部分捕获零个或多个字符，直到遇到空格（由`[:space:]`指示）为止。例如，如果位置标头是`Location: http://localhost:8000/todos/53`，正则表达式将捕获`http://localhost:8000/todos/53`。

+   `curl`命令：最后的设置任务是删除数据库中的所有待办事项，您可以通过向`/todos`URL 发送 DELETE 请求来实现。这确保了每次测试运行时 todobackend 数据库都是干净的，减少了不同测试引入破坏其他测试的副作用的可能性。

BATS 文件接下来定义了几个测试用例：

+   `todobackend root`：这包括`run`函数，该函数运行指定的命令并将命令的退出代码捕获在一个名为 status 的变量中，将命令的输出捕获在一个名为`output`的变量中。对于这种情况，测试运行`curl`命令的特殊配置，该配置仅捕获返回的 HTTP 状态代码，然后通过调用`[ $status = 0 ]`来验证`curl`命令成功完成，并通过调用`[ $output = 200 ]`来验证返回的 HTTP 状态代码是 200 代码。这些测试是常规的 shell *测试表达式*，相当于许多编程语言中找到的规范`assert`语句。

+   `todo items returns empty list`：这个测试用例使用`jq`命令传递调用`/todos`路径的输出。请注意，由于不能在特殊的`run`函数中使用管道，我已经使用了 bash 进程替换语法`<(...)`，使`curl`命令的输出看起来像是被`jq`命令读取的文件。

+   创建待办事项：首先创建一个待办事项，检查返回的退出代码是否为零，然后使用* bash 条件表达式*（如`[[...]]`语法所示）来验证`curl`命令的输出是否包含 HTTP 响应中的`201 Created`，这是创建事项时的标准响应。在使用 bash 条件表达式时，重要的是要注意，如果条件表达式失败，BATS 不会检测到错误，因此我们使用`|| false`特殊语法，该语法仅在条件表达式失败并返回非零响应`false`时才会被评估，如果测试表达式失败，测试用例将失败。条件表达式使用`=~`正则表达式运算符（此运算符在条件表达式中不可用，因此我们使用 bash 测试表达式），第二个条件表达式评估了设置函数中定义的`location`正则表达式。最后一个命令使用特殊的`BASH_REMATCH`变量，其中包含最近一次条件表达式评估的结果，本例中是在 Location 标头中匹配的 URL。这允许我们在创建待办事项时捕获返回的位置，并验证创建的事项是否与我们发布的事项匹配。

+   删除待办事项：这将创建一个待办事项，捕获返回的位置，删除该事项，然后验证事项是否被删除，验证数据库中的待办事项数量在删除后是否为零。请记住，设置函数在每个测试用例运行之前运行，它会清除所有待办事项，因此在这个测试用例开始时，待办事项数量始终为零，创建和删除事项的操作应该总是将数量返回为零。此测试用例中使用的各种命令基于“创建待办事项”测试用例中介绍的概念，因此我不会详细描述每个命令。

现在我们已经定义了一套验收测试，是时候修改 Docker 环境，以支持在应用程序成功启动后执行这些测试。

我们首先需要将`curl`，`bats`和`jq`软件包添加到 todobackend 存储库根目录下的`Dockerfile`中。

```
# Test stage
FROM alpine AS test
LABEL application=todobackend
...
...
# Release stage
FROM alpine
LABEL application=todobackend

# Install dependencies
RUN apk add --no-cache python3 mariadb-client bash curl bats jq
...
...
```

接下来，我们需要向`docker-compose.yml`文件添加一个名为`acceptance`的新服务，该服务将等待`app`服务健康，然后运行验收测试。

```
version: '2.4'

volumes:
  public:
    driver: local

services:
  test:
    ...
    ...
  release:
    ...
    ...
  app:
    extends:
      service: release
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - public:/public
 healthcheck:
 test: curl -fs localhost:8000
      interval: 3s
 retries: 10
    ports:
      - 8000:8000
    command:
      - uwsgi
      - --http=0.0.0.0:8000
      - --module=todobackend.wsgi
      - --master
      - --check-static=/public
  acceptance:
 extends:
 service: release
 depends_on:
 app:
 condition: service_healthy
 environment:
 APP_URL: http://app:8000
 command:
 - bats 
 - acceptance.bats
  migrate:
    ...
    ...
  db:
    ...
    ...
```

首先，我们为`app`服务添加了一个`healthcheck`属性，它使用`curl`实用程序来检查与本地 Web 服务器端点的连接。然后，我们定义了接受服务，我们从`release`镜像扩展，并配置了`APP_URL`环境变量，该变量配置了应该针对执行接受测试的正确 URL，而`command`和`depends_on`属性用于在`app`服务健康时运行接受测试。

有了这个配置，现在您需要拆除当前的环境，重建所有镜像，并执行各种步骤来启动应用程序，除非您到了即将运行`docker-compose up app`命令的时候，您现在应该运行`docker-compose up acceptance`命令，因为这将自动在后台启动`app`服务：

```
> docker-compose down -v
...
...
> docker-compose build
...
...
> docker-compose up migrate
...
...
> docker-compose run app python3 manage.py collectstatic --no-input
...
...
> docker-compose up acceptance todobackend_db_1 is up-to-date
Creating todobackend_app_1 ... done
Creating todobackend_acceptance_1 ... done
Attaching to todobackend_acceptance_1
acceptance_1 | Processing secrets []...
acceptance_1 | 1..4
acceptance_1 | ok 1 todobackend root
acceptance_1 | ok 2 todo items returns empty list
acceptance_1 | ok 3 create todo item
acceptance_1 | ok 4 delete todo item
todobackend_acceptance_1 exited with code 0
```

正如您所看到的，所有测试都通过了，每个测试都显示了`ok`状态。

# 自动化工作流程

到目前为止，您已成功配置了 Docker Compose 来构建、测试和创建样本应用程序的工作本地环境，包括 MySQL 数据库集成和接受测试。现在，您可以用少数命令来启动这个环境，但即使使用 Docker Compose 大大简化了您需要运行的命令，仍然很难记住要使用哪些命令以及以什么顺序。理想情况下，我们希望有一个单一的命令来运行完整的工作流程，这就是 GNU Make 这样的工具非常有用的地方。

Make 已经存在很长时间了，仍然被认为是许多 C 和 C++应用程序的首选构建工具。任务自动化是 Make 的一个关键特性，能够以简单的格式定义任务或目标，并且可以通过单个命令调用，这使得 Make 成为一个流行的自动化工具，特别是在处理 Docker 容器时。

按照惯例，make 会在当前工作目录中寻找一个名为 Makefile 的文件，您可以创建一个非常简单的 Makefile，就像这里演示的那样：

```
hello:
    @ echo "Hello World"
    echo "How are you?"
```

在前面的示例中，您创建了一个名为`hello`的*目标*，其中包含两个 shell 命令，您可以通过运行`make <target>`或在这个例子中运行`make hello`来执行这些命令。每个目标可以包括一个或多个命令，这些命令按照提供的顺序执行。

需要注意的一点是，make 期望在为给定目标定义各种命令时使用制表符（而不是空格），因此如果你收到缺少分隔符的错误，比如`Makefile:2: *** missing separator. Stop.`，请检查你是否使用了制表符来缩进每个命令。

```
> make hello
Hello World
echo "How are you?"
How are you?
```

在上面的例子中，你可以看到每个命令的输出都显示在屏幕上。请注意，第一个命令上的特殊字符`@`会抑制每个命令的回显。

任何像 Sublime Text 或 Visual Studio Code 这样的体面的现代文本编辑器都应该自动处理 Makefiles 中的制表符。

在使用 Makefiles 进行任务自动化时，你应该执行一个重要的清理工作，即配置一个名为`.PHONY`的特殊目标，并列出你将要执行的每个目标的名称：

```
.PHONY: hello

hello:
    @ echo "Hello World"
    echo "How are you?"
```

因为`make`实际上是一个用于编译源代码文件的构建工具，所以`.PHONY`目标告诉 make，如果它看到一个名为`hello`的文件，它仍然应该运行该目标。如果你没有指定`.PHONY`，并且本地目录中有一个名为`hello`的文件，make 将退出并声明`hello`文件已经构建完成。当你使用 make 来自动化任务时，这显然没有多大意义，所以你应该始终使用`.PHONY`目标来避免任何奇怪的意外。

# 自动化测试阶段

既然你已经了解了如何制作，让我们修改我们的 Makefile，以执行实际有用的操作，并执行测试阶段执行的各种操作。回想一下，测试阶段涉及构建 Dockerfile 的第一个阶段作为一个名为`test`的服务，然后运行`test`服务，默认情况下将运行`python3 manage.py test`命令，执行应用程序单元测试：

```
.PHONY: test

test:
    docker-compose build --pull release
    docker-compose build
    docker-compose run test
```

请注意，我们实际上并没有在 Docker Compose 文件中构建`test`服务，而是构建了发布服务并指定了`--pull`标志，这确保 Docker 始终检查 Docker 镜像中的任何更新版本。我们以这种方式构建`release`服务，因为我们只想构建整个`Dockerfile`一次，而不是在每个阶段执行时重新构建`Dockerfile`。

这可以防止一个不太可能但仍然可能发生的情况，即在发布阶段重新构建时，您可能会拉取一个更新的基础镜像，这可能导致与您在测试阶段测试的不同的运行时环境。我们还立即运行 docker-compose build 命令，这可以确保在运行测试之前构建所有服务。因为我们在前一个命令中构建了整个`Dockerfile`，这将确保其他服务的缓存镜像都更新为最新的镜像构建。

# 自动化发布阶段

完成测试阶段后，我们接下来运行发布阶段，这需要我们执行以下操作：

+   运行数据库迁移

+   收集静态文件

+   启动应用程序

+   运行验收测试

以下演示了在 Makefile 中创建一个名为`release`的目标：

```
.PHONY: test release

test:
    docker-compose build --pull release
    docker-compose build
    docker-compose run test

release:
 docker-compose up --abort-on-container-exit migrate
 docker-compose run app python3 manage.py collectstatic --no-input
 docker-compose up --abort-on-container-exit acceptance
```

请注意，我们执行所需命令的每一个时，都会有一个小变化，即在每个`docker-compose up`命令中添加`--abort-on-container-exit`命令。默认情况下，`docker-compose up`命令不会返回非零退出代码，如果命令启动的任何容器失败。这个标志允许您覆盖这一点，并指定任何由`docker-compose up`命令启动的服务失败，那么 Docker Compose 应该以错误退出。如果您希望在出现错误时使您的 make 命令失败，设置此标志是很重要的。

# 完善工作流程

有一些更小的增强可以应用到工作流程中，这将确保我们有一个强大、一致和可移植的测试和构建应用程序的机制。

# 清理 Docker 环境

在本章中，我们一直通过运行`docker-compose down`命令来清理我们的环境，该命令停止并销毁与 todobackend Docker Compose 环境相关的任何容器。

在构建 Docker 镜像时，您需要注意的另一个方面是孤立或悬空的镜像的概念，这些镜像已经被新版本取代。您可以通过运行`docker images`命令来了解这一点，我已经用粗体标出了哪些镜像是悬空的：

```
> docker images REPOSITORY            TAG        IMAGE ID        CREATED            SIZEtodobackend_app       latest     ca3e62e168f2    13 minutes ago     137MBtodobackend_migrate   latest     ca3e62e168f2    13 minutes ago     137MB
todobackend_release   latest     ca3e62e168f2    13 minutes ago     137MB
<none>                <none>     03cc5d44bd7d    14 minutes ago     253MB
<none>                <none>     e88666a35577    22 minutes ago     137MB
<none>                <none>     8909f9001297    23 minutes ago     253MB
<none>                <none>     3d6f9a5c9322    2 hours ago        137MB todobackend_test      latest     60b3a71946cc    2 hours ago        253MB
<none>                <none>     53d19a2de60d    9 hours ago        136MB
<none>                <none>     54f0fb70b9d0    15 hours ago       135MB alpine                latest     11cd0b38bc3c    23 hours ago       4.41MB
```

请注意，每个突出显示的图像都没有存储库和标签，因此它们被称为孤立或悬空。这些悬空图像没有用处，占用资源和存储空间，因此最好定期清理这些图像，以确保 Docker 环境的性能。回到我们的 Dockerfile，我们在每个阶段添加了`LABEL`指令，这允许轻松识别与我们的 todobackend 应用相关的图像。

我们可以利用这些标签来定位为 todobackend 应用构建的悬空图像，因此让我们在 Makefile 中添加一个名为`clean`的新目标，该目标关闭 Docker Compose 环境并删除悬空图像。

```
.PHONY: test release clean

test:
    docker-compose build --pull release
    docker-compose build
    docker-compose run test

release:
    docker-compose up --abort-on-container-exit migrate
    docker-compose run app python3 manage.py collectstatic --no-input
    docker-compose up --abort-on-container-exit acceptance

clean:
 docker-compose down -v
 docker images -q -f dangling=true -f label=application=todobackend | xargs -I ARGS docker rmi -f --no-prune ARGS
```

使用`-q`标志仅打印出图像 ID，然后使用`-f`标志添加过滤器，指定仅显示具有`application=todobackend`标签的悬空图像。然后将此命令的输出导入到`xargs`命令中，`xargs`捕获过滤图像列表并将其传递给`docker rmi -f --no-prune`命令，根据`-f`标志强制删除图像，并使用`--no-prune`标志确保不删除包含当前标记图像层的未标记图像。我们在这里使用`xargs`是因为它能智能地处理图像列表-例如，如果没有要删除的图像，那么`xargs`会在没有错误的情况下静默退出。

以下演示了运行`make clean`命令的输出：

```
> make test
...
...
> make release
...
...
> make clean
docker-compose down -v
Stopping todobackend_app_1 ... done
Stopping todobackend_db_1 ... done
Removing todobackend_app_run_2 ... done
Removing todobackend_app_1 ... done
Removing todobackend_app_run_1 ... done
Removing todobackend_migrate_1 ... done
Removing todobackend_db_1 ... done
Removing todobackend_test_run_1 ... done
Removing network todobackend_default
Removing volume todobackend_public
docker images -q -f dangling=true -f label=application=todobackend | xargs -I ARGS docker rmi -f --no-prune ARGS
Deleted: sha256:03cc5d44bd7dec8d535c083dd5a8e4c177f113bc49f6a97d09f7a1deb64b7728
Deleted: sha256:6448ea330f415f773fc4cd5fe35862678ac0e35a1bf24f3780393eb73637f765
Deleted: sha256:baefcaca3929d6fc419eab06237abfb6d9ba9a1ba8d5623040ea4f49b2cc22d4
Deleted: sha256:b1dca5a87173bfa6a2c0c339cdeea6287e4207f34869a2da080dcef28cabcf6f
...
...
```

当运行`make clean`命令时，您可能会注意到一件事，即停止 todobackend 应用服务需要一些时间，实际上，需要大约 10 秒才能停止。这是因为在停止容器时，Docker 首先向容器发送 SIGTERM 信号，这会向容器发出即将被终止的信号。默认情况下，如果容器在 10 秒内没有退出，Docker 会发送 SIGKILL 信号，强制终止容器。

问题在于我们应用容器中运行的`uwsgi`进程默认情况下会忽略 SIGTERM 信号，因此我们需要在配置`uwsgi`的 Docker Compose 文件中添加`--die-on-term`标志，以确保它能够优雅地和及时地关闭，如果收到 SIGTERM 信号。

```
version: '2.4'

volumes:
  public:
    driver: local

services:
  test:
    ...
    ...
  release:
    ...
    ...
  app:
    extends:
      service: release
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - public:/public
    healthcheck:
      test: curl -fs localhost:8000
      interval: 3s
      retries: 10
    ports:
      - 8000:8000
    command:
      - uwsgi
      - --http=0.0.0.0:8000
      - --module=todobackend.wsgi
      - --master
      - --check-static=/public
 - --die-on-term
 - --processes=4
 - --threads=2
  acceptance:
    ...
    ...
  migrate:
    ...
    ...
  db:
    ...
    ...
```

在上面的例子中，我还添加了`--processes`和`--threads`标志，这些标志启用并发处理。您可以在[`uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html#adding-concurrency-and-monitoring`](https://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html#adding-concurrency-and-monitoring)中了解更多配置选项。

# 使用动态端口映射

当前，发布阶段工作流程使用静态端口映射运行应用程序，其中 app 服务容器上的端口 8000 映射到 Docker Engine 上的端口`8000`。尽管在本地运行时通常可以正常工作（除非有其他使用端口 8000 的应用程序），但是在远程持续交付构建服务上运行发布阶段工作流程时可能会导致问题，该服务可能正在为许多不同的应用程序运行多个构建。

更好的方法是使用动态端口映射，将`app`服务容器端口映射到 Docker Engine 上当前未使用的动态端口。端口是从所谓的*临时端口范围*中选择的，这是一个为应用程序动态使用保留的端口范围。

要配置动态端口映射，您需要在`docker-compose.yml`文件中的`app`服务中更改端口映射：

```
version: '2.4'

volumes:
  public:
    driver: local

services:
  test:
    ...
    ...
  release:
    ...
    ...
  app:
    extends:
      service: release
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - public:/public
    healthcheck:
      test: curl -fs localhost:8000
      interval: 3s
      retries: 10
    ports:
 - 8000
    command:
      - uwsgi
      - --http=0.0.0.0:8000
      - --module=todobackend.wsgi
      - --master
      - --check-static=/public
      - --die-on-term
      - --processes=4
      - --threads=2
  acceptance:
    ...
    ...
  migrate:
    ...
    ...
  db:
    ...
    ...
```

在上面的例子中，我们只是将端口映射从`8000:8000`的静态映射更改为`8000`，这样就可以启用动态端口映射。有了这个配置，一个问题是您事先不知道将分配什么端口，但是您可以使用`docker-compose port <service> <container-port>`命令来确定给定服务在给定容器端口上的当前动态端口映射：

```
> docker-compose port app 8000
0.0.0.0:32768
```

当然，与其每次手动输入此命令，我们可以将其纳入自动化工作流程中：

```
.PHONY: test release clean

test:
    docker-compose build --pull release
    docker-compose build
    docker-compose run test

release:
    docker-compose up --exit-code-from migrate migrate
    docker-compose run app python3 manage.py collectstatic --no-input
    docker-compose up --exit-code-from acceptance acceptance
 @ echo App running at http://$$(docker-compose port app 8000 | sed s/0.0.0.0/localhost/g) clean:
    docker-compose down -v
    docker images -q -f dangling=true -f label=application=todobackend | xargs -I ARGS docker rmi -f --no-prune ARGS
```

在上面的例子中，我们使用命令替换来获取当前的端口映射，并将输出传输到一个`sed`表达式，将`0.0.0.0`替换为`localhost`。请注意，因为 GNU Make 将美元符号解释为 Make 变量引用，如果您希望 shell 命令执行时评估单个美元符号，则需要双重转义美元符号（`$$`）。

有了这个配置，`make release`命令的输出现在将完成如下：

```
> make release
...
...
docker-compose run app bats acceptance.bats
Starting todobackend_db_1 ... done
1..4
ok 1 todobackend root
ok 2 todo items returns empty list
ok 3 create todo item
ok 4 delete todo item
App running at http://localhost:32771
```

# 添加版本目标

对应用程序进行版本控制非常重要，特别是在构建 Docker 镜像时，您希望区分各种镜像。稍后，当我们发布我们的 Docker 镜像时，我们将需要在每个发布的镜像上包含一个版本标签，版本控制的一个简单约定是在应用程序存储库中使用当前提交的 Git 提交哈希。

以下演示了如何在一个 Make 变量中捕获这个，并显示当前版本：

```
.PHONY: test release clean version

export APP_VERSION ?= $(shell git rev-parse --short HEAD)

version:
 @ echo '{"Version": "$(APP_VERSION)"}'

test:
    docker-compose build --pull release
    docker-compose build
    docker-compose run test

release:
    docker-compose up --abort-on-container-exit migrate
    docker-compose run app python3 manage.py collectstatic --no-input
    docker-compose up --abort-on-container-exit acceptance
    @ echo App running at http://$$(docker-compose port app 8000 | sed s/0.0.0.0/localhost/g)clean:
    docker-compose down -v
    docker images -q -f dangling=true -f label=application=todobackend | xargs -I ARGS docker rmi -f --no-prune ARGS
```

我们首先声明一个名为`APP_VERSION`的变量，并在前面加上`export`关键字，这意味着该变量将在每个目标的环境中可用。然后，我们使用一个名为`shell`的 Make 函数来执行`git rev-parse --short HEAD`命令，该命令返回当前提交的七个字符的短哈希。最后，我们添加一个名为`version`的新目标，它简单地以 JSON 格式打印版本到终端，这在本书后面当我们自动化应用程序的持续交付时将会很有用。请注意，`make`使用美元符号来引用变量，也用来执行 Make 函数，您可以在[`www.gnu.org/software/make/manual/html_node/Functions.html`](https://www.gnu.org/software/make/manual/html_node/Functions.html)了解更多信息。

如果只运行`make`命令而没有指定目标，make 将执行 Makefile 中的第一个目标。这意味着，对于我们的情况，只运行`make`将输出当前版本。

以下演示了运行`make version`命令：

```
> make version
{"Version": "5cd83c0"}
```

# 测试端到端工作流

此时，我们本地 Docker 工作流的所有部分都已就位，现在是审查工作流并验证一切是否正常运行的好时机。

核心工作流现在包括以下任务：

+   运行测试阶段 - `make test`

+   运行发布阶段 - `make release`

+   清理 - `make clean`

我会把这个测试留给你，但我鼓励你熟悉这个工作流程，并确保一切都能顺利完成。运行`make release`后，验证您是否可以导航到应用程序，应用程序是否正确显示 HTML 内容，以及您是否可以执行创建、读取、更新和删除操作。

一旦您确信一切都按预期工作，请确保已提交并推送您在上一章中分叉的 GitHub 存储库的更改。

# 总结

在本章中，您实现了一个 Docker 工作流，用于测试、构建和打包应用程序成一个 Docker 镜像，准备发布和部署到生产环境。您学会了如何使用 Docker 多阶段构建来构建应用程序的两个阶段——测试阶段使用开发环境，包括开发库和源代码编译工具，允许您构建和测试应用程序及其依赖关系的预编译包；而发布阶段则将这些构建好的包安装到一个生产就绪的操作环境中，不包含开发库和其他工具，显著减少了应用程序的攻击面。

您学会了如何使用 Docker Compose 来简化测试和发布阶段需要执行的各种命令和操作，创建了一个`docker-compose.yml`文件，其中包含了一些服务，每个服务都以一种声明性、易于理解的格式进行定义。您学会了如何复制一些部署任务，例如运行数据库迁移、收集静态文件，并确保应用程序数据库在尝试运行应用程序之前是健康的。在本地环境中执行每个任务使您能够对这些任务在实际生产环境中的工作方式有信心和了解，并在本地出现任何应用程序或配置更改破坏这些过程时提前警告。在将应用程序处于正确状态并连接到应用程序数据库后，您学会了如何从外部客户端的角度运行验收测试，这让您对镜像是否按预期工作有了很大的信心，并在这些验收测试在应用程序持续开发过程中失败时提前警告。

最后，你学会了如何使用 GNU Make 将所有这些内容整合到一个完全自动化的工作流程中，它为你提供了简单的高级命令，可以用来执行工作流程。现在你可以通过简单地运行`make test`来执行测试阶段，通过运行`make release`来运行发布阶段，并使用`make clean`清理你的环境。这使得运行工作流程变得非常容易，并且在本书的后面，将简化我们将使用的自动测试、构建和发布 Docker 应用程序的持续交付构建系统的配置。

在接下来的章节中，你将学习如何实际发布你在本章中创建的 Docker 发布镜像，但在你这样做之前，你需要建立一个 AWS 账户，配置对你的账户的访问，并安装支持与 AWS 交互的工具，这将是下一章的重点。

# 问题

1.  真/假：你使用`FROM`和`TO`指令来定义多阶段 Dockerfile。

1.  真/假：`docker`命令的`--rm`标志在容器退出后自动删除容器。

1.  真/假：当运行你的工作流程时，你应该只构建应用程序构件一次。

1.  真/假：当运行`docker-compose run`命令时，如果目标服务启动失败并出现错误，docker-compose 将以非零代码退出。

1.  真/假：当运行`docker-compose up`命令时，如果其中一个服务启动失败并出现错误，docker-compose 将以非零代码退出。

1.  真/假：如果你想使用 Docker Swarm，你应该配置一个 Docker Compose 版本为 3.x。

1.  你在 Docker 文件中为一个服务的依赖项配置了 service_healthy 条件。然后你使用`docker-compose run`命令运行服务；依赖项已启动，然而 Docker Compose 并不等待依赖项健康，而是立即启动服务，导致失败。你如何解决这个问题？

1.  你在 Docker Compose 中创建了一个服务，端口映射为`8000:8000`。当你尝试启动这个服务时，会出现一个错误，指示端口已被使用。你如何解决这个问题，并确保它不会再次发生呢？

1.  创建一个 Makefile 后，当尝试运行一个目标时，收到一个关于缺少分隔符的错误。这个错误最有可能的原因是什么？

1.  哪个 GNU Make 函数允许你捕获 shell 命令的输出？

1.  在 Makefile 中定义了一个名为 test 的目标，但是当你运行`make test`时，你会得到一个回应说没有什么可做的。你该如何解决这个问题呢？

1.  在 Docker Compose 服务定义中必须配置哪些属性才能使用`docker-compose push`命令？

# 进一步阅读

您可以查看以下链接，了解本章涵盖的主题的更多信息：

+   Docker 命令行参考：[`docs.docker.com/engine/reference/commandline/cli/`](https://docs.docker.com/engine/reference/commandline/cli/)

+   Docker 多阶段构建：[`docs.docker.com/develop/develop-images/multistage-build/`](https://docs.docker.com/develop/develop-images/multistage-build/)

+   Docker Compose 版本 2 规范：[`docs.docker.com/compose/compose-file/compose-file-v2/`](https://docs.docker.com/compose/compose-file/compose-file-v2/)

+   Docker Compose 命令行参考：[`docs.docker.com/compose/reference/`](https://docs.docker.com/compose/reference/)

+   Docker Compose 启动顺序：[`docs.docker.com/compose/startup-order/`](https://docs.docker.com/compose/startup-order/)

+   uWSGI Python 应用程序快速入门：[`uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html`](http://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html)

+   Bash 自动化测试系统：[`github.com/sstephenson/bats`](https://github.com/sstephenson/bats)

+   GNU Make 虚假目标：[`www.gnu.org/software/make/manual/html_node/Phony-Targets.html`](https://www.gnu.org/software/make/manual/html_node/Phony-Targets.html)

+   GNU Make 函数：[`www.gnu.org/software/make/manual/html_node/Functions.html#Functions`](https://www.gnu.org/software/make/manual/html_node/Functions.html#Functions)

## 8: 容器化应用程序

Docker 的全部内容都是关于将应用程序放入容器中运行。

将应用程序配置为容器运行的过程称为“容器化”。有时我们称之为“Docker 化”。

在本章中，我们将介绍将一个简单的 Linux Web 应用程序容器化的过程。如果您没有 Linux Docker 环境可以跟随操作，可以免费使用*Play With Docker*。只需将您的网络浏览器指向 https://play-with-docker.com 并启动一些 Linux Docker 节点。这是我启动 Docker 并进行测试的最喜欢的方式！

我们将把这一章分为通常的三个部分：

+   简而言之

+   深入探讨

+   命令

让我们将应用程序容器化！

### 容器化应用程序-简而言之

容器都是关于应用程序！特别是，它们是关于使应用程序简单**构建**、**交付**和**运行**。

容器化应用程序的过程如下：

1.  从您的应用程序代码开始。

1.  创建一个描述您的应用程序、其依赖关系以及如何运行它的*Dockerfile*。

1.  将此*Dockerfile*输入`docker image build`命令。

1.  坐下来，让 Docker 将您的应用程序构建成一个 Docker 镜像。

一旦您的应用程序被容器化（制作成 Docker 镜像），您就可以准备将其交付并作为容器运行。

图 8.1 以图片形式展示了这个过程。

![图 8.1-容器化应用程序的基本流程](img/figure8-1.png)

图 8.1-容器化应用程序的基本流程

### 容器化应用程序-深入探讨

我们将把本章的深入探讨部分分为以下几个部分：

+   容器化单容器应用程序

+   使用多阶段构建进行生产

+   一些最佳实践

#### 容器化单容器应用程序

本章的其余部分将引导您完成容器化一个简单的单容器 Node.js Web 应用程序的过程。这个过程对于 Windows 是一样的，未来版本的书籍将包括一个 Windows 示例。

我们将完成以下高层次的步骤：

+   获取应用程序代码

+   检查 Dockerfile

+   容器化应用程序

+   运行应用程序

+   测试应用程序

+   仔细观察一下

+   使用**多阶段构建**进行生产

+   一些最佳实践

尽管在本章中我们将使用单容器应用程序，但在下一章关于 Docker Compose 中，我们将转向多容器应用程序。之后，我们将在关于 Docker Stacks 的章节中转向更复杂的应用程序。

##### 获取应用程序代码

本示例中使用的应用程序可以从 GitHub 克隆：

https://github.com/nigelpoulton/psweb.git

从 GitHub 克隆示例应用程序。

```
$ git clone https://github.com/nigelpoulton/psweb.git

Cloning into `'psweb'`...
remote: Counting objects: `15`, `done`.
remote: Compressing objects: `100`% `(``11`/11`)`, `done`.
remote: Total `15` `(`delta `2``)`, reused `15` `(`delta `2``)`, pack-reused `0`
Unpacking objects: `100`% `(``15`/15`)`, `done`.
Checking connectivity... `done`. 
```

`克隆操作会创建一个名为`psweb`的新目录。切换到`psweb`目录并列出其内容。

```
$ `cd` psweb

$ ls -l
total `28`
-rw-r--r-- `1` root root  `341` Sep `29` `16`:26 app.js
-rw-r--r-- `1` root root  `216` Sep `29` `16`:26 circle.yml
-rw-r--r-- `1` root root  `338` Sep `29` `16`:26 Dockerfile
-rw-r--r-- `1` root root  `421` Sep `29` `16`:26 package.json
-rw-r--r-- `1` root root  `370` Sep `29` `16`:26 README.md
drwxr-xr-x `2` root root `4096` Sep `29` `16`:26 `test`
drwxr-xr-x `2` root root `4096` Sep `29` `16`:26 views 
```

“这个目录包含了所有的应用程序源代码，以及用于视图和单元测试的子目录。随意查看这些文件 - 应用程序非常简单。在本章中，我们不会使用单元测试。

现在我们有了应用程序代码，让我们来看看它的 Dockerfile。

##### 检查 Dockerfile

请注意，存储库有一个名为**Dockerfile**的文件。这个文件描述了应用程序，并告诉 Docker 如何将其构建成一个镜像。

包含应用程序的目录被称为*构建上下文*。将 Dockerfile 放在*构建上下文*的根目录是一种常见做法。同时，**Dockerfile**以大写的“**D**”开头，并且是一个单词。 “dockerfile”和“Docker file”都是无效的。

让我们来看看 Dockerfile 的内容。

```
$ cat Dockerfile

FROM alpine
LABEL `maintainer``=``"nigelpoulton@hotmail.com"`
RUN apk add --update nodejs nodejs-npm
COPY . /src
WORKDIR /src
RUN npm install
EXPOSE `8080`
ENTRYPOINT `[``"node"`, `"./app.js"``]` 
```

`Dockerfile 有两个主要目的：

1.  描述应用程序

1.  告诉 Docker 如何将应用程序容器化（创建一个包含应用程序的镜像）

不要低估 Dockerfile 作为文档的影响！它有助于弥合开发和运维之间的差距！它还有助于加快新开发人员等的入职速度。这是因为该文件准确描述了应用程序及其依赖关系，格式易于阅读。因此，它应被视为代码，并检入源代码控制系统。

在高层次上，示例 Dockerfile 表示：从`alpine`镜像开始，将“nigelpoulton@hotmail.com”添加为维护者，安装 Node.js 和 NPM，复制应用程序代码，设置工作目录，安装依赖项，记录应用程序的网络端口，并将`app.js`设置为默认要运行的应用程序。

让我们更详细地看一下。

所有 Dockerfile 都以`FROM`指令开头。这将是镜像的基础层，应用程序的其余部分将作为附加层添加在顶部。这个特定的应用程序是一个 Linux 应用程序，所以很重要的是`FROM`指令引用一个基于 Linux 的镜像。如果您要容器化一个 Windows 应用程序，您需要指定适当的 Windows 基础镜像 - 比如`microsoft/aspnetcore-build`。

此时，镜像看起来像图 8.2。

![图 8.2](img/figure8-2.png)

图 8.2

接下来，Dockerfile 创建了一个 LABEL，指定“nigelpoulton@hotmail.com”作为镜像的维护者。标签是简单的键值对，是向镜像添加自定义元数据的绝佳方式。将镜像的维护者列出来被认为是最佳实践，这样其他潜在的用户在使用时有一个联系点。

> **注意：** 我将不会维护这个镜像。我包含这个标签是为了向您展示如何使用标签，同时向您展示最佳实践。

`RUN apk add --update nodejs nodejs-npm` 指令使用 Alpine `apk` 包管理器将 `nodejs` 和 `nodejs-npm` 安装到镜像中。RUN 指令将这些软件包安装为新的镜像层，放在由 `FROM alpine` 指令创建的 `alpine` 基础镜像之上。镜像现在看起来像图 8.3。

![图 8.3](img/Figure8-3.png)

图 8.3

`COPY . /src` 指令从*构建上下文*中复制应用程序文件。它将这些文件作为新层复制到镜像中。镜像现在有三个层，如图 8.4 所示。

![图 8.4](img/figure8-4.png)

图 8.4

接下来，Dockerfile 使用 `WORKDIR` 指令为文件中的其余指令设置工作目录。此目录是相对于镜像的，并且该信息被添加为镜像配置的元数据，而不是作为新层。

然后，`RUN npm install` 指令使用 `npm` 在构建上下文中列出的 `package.json` 文件中安装应用程序依赖项。它在前一条指令中设置的 `WORKDIR` 上下文中运行，并将依赖项安装为镜像中的新层。镜像现在有四个层，如图 8.5 所示。

![图 8.5](img/figure8-5.png)

图 8.5

该应用程序在 TCP 端口 8080 上公开了一个 Web 服务，因此 Dockerfile 使用 `EXPOSE 8080` 指令记录了这一点。这被添加为镜像元数据，而不是镜像层。

最后，`ENTRYPOINT` 指令用于设置镜像（容器）应该运行的主要应用程序。这也被添加为元数据，而不是镜像层。

##### 容器化应用/构建镜像

现在我们了解了它是如何工作的，让我们来构建它吧！

以下命令将构建一个名为 `web:latest` 的新镜像。命令末尾的句点 (`.`) 告诉 Docker 使用当前 shell 的工作目录作为*构建上下文*。

确保在命令的末尾包括句点（.），并确保从包含 Dockerfile 和应用程序代码的`psweb`目录运行命令。

```
$ docker image build -t web:latest .

Sending build context to Docker daemon  `76`.29kB
Step `1`/8 : FROM alpine
latest: Pulling from library/alpine
ff3a5c916c92: Pull `complete`
Digest: sha256:7df6db5aa6...0bedab9b8df6b1c0
Status: Downloaded newer image `for` alpine:latest
 ---> 76da55c8019d
<Snip>
Step `8`/8 : ENTRYPOINT node ./app.js
 ---> Running in 13977a4f3b21
 ---> fc69fdc4c18e
Removing intermediate container 13977a4f3b21
Successfully built fc69fdc4c18e
Successfully tagged web:latest 
```

`检查镜像是否存在于您的 Docker 主机的本地存储库中。

```
$ docker image ls
REPO    TAG       IMAGE ID          CREATED              SIZE
web     latest    fc69fdc4c18e      `10` seconds ago       `64`.4MB 
```

`恭喜，应用已经容器化！

您可以使用`docker image inspect web:latest`命令来验证镜像的配置。它将列出从 Dockerfile 配置的所有设置。

##### 推送镜像

创建了一个镜像后，将其存储在镜像注册表中是一个好主意，以确保其安全并使其对他人可用。Docker Hub 是最常见的公共镜像注册表，也是`docker image push`命令的默认推送位置。

为了将镜像推送到 Docker Hub，您需要使用您的 Docker ID 登录。您还需要适当地标记镜像。

让我们登录 Docker Hub 并推送新创建的镜像。

在以下示例中，您需要用您自己的 Docker ID 替换我的 Docker ID。因此，每当您看到“nigelpoulton”时，请将其替换为您的 Docker ID。

```
$ docker login
Login with **your** Docker ID to push and pull images from Docker Hub...
Username: nigelpoulton
Password:
Login Succeeded 
```

`在您可以推送镜像之前，您需要以特殊方式标记它。这是因为 Docker 在推送镜像时需要以下所有信息：

+   `注册表`

+   `存储库`

+   `标签`

Docker 有自己的观点，因此您不需要为`Registry`和`Tag`指定值。如果您不指定值，Docker 将假定`Registry=docker.io`和`Tag=latest`。但是，Docker 没有默认值用于存储库值，它从正在推送的镜像的“REPOSITORY”值中获取。这可能会让人感到困惑，因此让我们仔细看一下我们示例中的值。

先前的`docker image ls`输出显示我们的镜像的存储库名称为`web`。这意味着`docker image push`将尝试将镜像推送到`docker.io/web:latest`。但是，我无法访问`web`存储库，我的所有镜像都必须位于`nigelpoulton`的二级命名空间中。这意味着我们需要重新标记镜像以包含我的 Docker ID。

```
$ docker image tag web:latest nigelpoulton/web:latest 
```

`命令的格式是`docker image tag <current-tag> <new-tag>`，它会添加一个额外的标签，而不是覆盖原始标签。

另一个镜像列表显示，该镜像现在有两个标签，其中一个包含我的 Docker ID。

```
$ docker image ls
REPO                TAG       IMAGE ID         CREATED         SIZE
web                 latest    fc69fdc4c18e     `10` secs ago     `64`.4MB
nigelpoulton/web    latest    fc69fdc4c18e     `10` secs ago     `64`.4MB 
```

`现在我们可以将其推送到 Docker Hub。

```
$ docker image push nigelpoulton/web:latest
The push refers to repository `[`docker.io/nigelpoulton/web`]`
2444b4ec39ad: Pushed
ed8142d2affb: Pushed
d77e2754766d: Pushed
cd7100a72410: Mounted from library/alpine
latest: digest: sha256:68c2dea730...f8cf7478 size: `1160` 
```

`图 8.6 显示了 Docker 如何确定推送位置。

![图 8.6](img/figure8-6.png)

图 8.6

您将无法将镜像推送到我的 Docker Hub 命名空间中的存储库，您将需要使用您自己的存储库。

本章其余部分的所有示例都将使用两个标签中较短的一个（`web:latest`）。

##### 运行应用程序

我们容器化的应用程序是一个简单的 Web 服务器，监听 TCP 端口 8080。您可以在`app.js`文件中验证这一点。

以下命令将基于我们刚刚创建的`web:latest`镜像启动一个名为`c1`的新容器。它将 Docker 主机上的端口`80`映射到容器内部的端口`8080`。这意味着您将能够将网络浏览器指向 Docker 主机的 DNS 名称或 IP 地址，并访问该应用程序。

> **注意：**如果您的主机已经在端口 80 上运行服务，您可以在`docker container run`命令中指定不同的端口。例如，要将应用程序映射到 Docker 主机上的端口 5000，请使用`-p 5000:8080`标志。

```
$ docker container run -d --name c1 `\`
  -p `80`:8080 `\`
  web:latest 
```

`-d`标志在后台运行容器，`-p 80:8080`标志将主机上的端口 80 映射到运行容器内部的端口 8080。

检查容器是否正在运行并验证端口映射。

```
$ docker container ls

ID    IMAGE       COMMAND           STATUS      PORTS
`49`..  web:latest  `"node ./app.js"`   UP `6` secs   `0`.0.0.0:80->8080/tcp 
```

上面的输出被剪辑以提高可读性，但显示应用程序容器正在运行。请注意，端口 80 被映射到容器中的端口 8080，映射到所有主机接口（`0.0.0.0:80`）。

##### 测试应用程序

打开一个网络浏览器，并将其指向容器正在运行的主机的 DNS 名称或 IP 地址。您将看到图中显示的网页。

![图 8.7](img/figure8-7.png)

图 8.7

如果测试不起作用，请尝试以下操作：

1.  确保容器正在运行并且使用`docker container ls`命令。容器名称为`c1`，您应该看到端口映射为`0.0.0.0:80->8080/tcp`。

1.  检查防火墙和其他网络安全设置是否阻止 Docker 主机上的 80 端口的流量。

恭喜，应用程序已经容器化并正在运行！

##### 更仔细地观察

现在应用程序已经容器化，让我们更仔细地看看一些机制是如何工作的。

Dockerfile 中的注释行以`#`字符开头。

所有非注释行都是**指令**。指令采用`INSTRUCTION argument`的格式。指令名称不区分大小写，但通常习惯将它们写成大写。这样可以更容易地阅读 Dockerfile。

`docker image build`命令逐行解析 Dockerfile，从顶部开始。

一些指令创建新的层，而其他一些只是向镜像添加元数据。

创建新层的指令示例包括`FROM`、`RUN`和`COPY`。创建元数据的指令示例包括`EXPOSE`、`WORKDIR`、`ENV`和`ENTRYPOINT`。基本原则是 - 如果一条指令正在向镜像添加*内容*，比如文件和程序，它将创建一个新的层。如果它正在添加有关如何构建镜像和运行应用程序的指令，它将创建元数据。

您可以使用`docker image history`命令查看用于构建镜像的指令。

```
$ docker image `history` web:latest

IMAGE     CREATED BY                                      SIZE
fc6..18e  /bin/sh -c `#(nop)  ENTRYPOINT ["node" "./a...   0B`
`334`..bf0  /bin/sh -c `#(nop)  EXPOSE 8080/tcp              0B`
b27..eae  /bin/sh -c npm install                          `14`.1MB
`932`..749  /bin/sh -c `#(nop) WORKDIR /src                  0B`
`052`..2dc  /bin/sh -c `#(nop) COPY dir:2a6ed1703749e80...   22.5kB`
c1d..81f  /bin/sh -c apk add --update nodejs nodejs-npm   `46`.1MB
`336`..b92  /bin/sh -c `#(nop)  LABEL maintainer=nigelp...   0B`
3fd..f02  /bin/sh -c `#(nop)  CMD ["/bin/sh"]              0B`
<missing> /bin/sh -c `#(nop) ADD file:093f0723fa46f6c...   4.15MB` 
```

上面输出中的两点值得注意。

首先。每行对应 Dockerfile 中的一条指令（从底部开始向上工作）。`CREATED BY`列甚至列出了执行的确切 Dockerfile 指令。

其次。在输出中显示的行中，只有 4 行创建了新的层（`SIZE`列中的非零值）。这些对应于 Dockerfile 中的`FROM`、`RUN`和`COPY`指令。尽管其他指令看起来像是创建了层，但实际上它们创建的是元数据而不是层。`docker image history`输出使所有指令看起来都创建了层的原因是 Docker 构建和镜像分层的方式的产物。

使用`docker image inspect`命令确认只创建了 4 个层。

```
$ docker image inspect web:latest

<Snip>
`}`,
`"RootFS"`: `{`
    `"Type"`: `"layers"`,
    `"Layers"`: `[`
        `"sha256:cd7100...1882bd56d263e02b6215"`,
        `"sha256:b3f88e...cae0e290980576e24885"`,
        `"sha256:3cfa21...cc819ef5e3246ec4fe16"`,
        `"sha256:4408b4...d52c731ba0b205392567"`
    `]`
`}`, 
```

使用`FROM`指令从官方仓库中使用镜像被认为是一个良好的做法。这是因为它们往往遵循最佳实践，并且相对免受已知漏洞的影响。从小型镜像（`FROM`）开始也是一个好主意，因为这样可以减少潜在的漏洞。

您可以查看`docker image build`命令的输出，以了解构建镜像的一般过程。如下摘录所示，基本过程是：`启动临时容器` > `在该容器内运行 Dockerfile 指令` > `将结果保存为新的镜像层` > `删除临时容器`。

```
Step 3/8 : RUN apk add --update nodejs nodejs-npm
 ---> Running in e690ddca785f    << Run inside of temp container
fetch http://dl-cdn...APKINDEX.tar.gz
fetch http://dl-cdn...APKINDEX.tar.gz
(1/10) Installing ca-certificates (20171114-r0)
<Snip>
OK: 61 MiB in 21 packages
 ---> c1d31d36b81f               << Create new layer
Removing intermediate container  << Remove temp container
Step 4/8 : COPY . /src 
```

#### 使用**多阶段构建**进行生产部署

在 Docker 镜像中，大尺寸是不好的！

大尺寸意味着慢。大尺寸意味着难以处理。大尺寸意味着更多的潜在漏洞，可能意味着更大的攻击面！

因此，Docker 镜像应该尽量保持小尺寸。游戏的目标是只在生产镜像中包含**必要**的内容来运行您的应用程序。

问题是...保持镜像尺寸小曾经是一项艰苦的工作。

例如，您编写 Dockerfile 的方式对映像的大小有很大影响。一个常见的例子是每个`RUN`指令都会添加一个新的层。因此，通常被认为是最佳实践的是将多个命令作为单个 RUN 指令的一部分 - 所有这些命令都用双和号（&&）和反斜杠（`\`）换行符粘合在一起。虽然这并不是什么高深的学问，但它需要时间和纪律。

另一个问题是我们没有清理干净。我们将针对一个映像运行一个命令，拉取一些构建时工具，并在将其发送到生产环境时将所有这些工具留在映像中。这并不理想！

有解决方法 - 最明显的是*构建者模式*。但其中大多数都需要纪律和增加复杂性。

构建者模式要求您至少有两个 Dockerfile - 一个用于开发，一个用于生产。您将编写 Dockerfile.dev 以从大型基础映像开始，拉入所需的任何其他构建工具，并构建您的应用程序。然后，您将从 Dockerfile.dev 构建一个映像，并从中创建一个容器。然后，您将使用 Dockerfile.prod 从较小的基础映像构建一个新的映像，并从刚刚从构建映像创建的容器中复制应用程序内容。所有内容都需要用脚本粘合在一起。

这种方法是可行的，但代价是复杂性。

多阶段构建来拯救！

多阶段构建的目标是优化构建而不增加复杂性。它们实现了承诺！

这是一个高层次的…

使用多阶段构建，我们有一个包含多个 FROM 指令的单个 Dockerfile。每个 FROM 指令都是一个新的**构建阶段**，可以轻松地从以前的**阶段**复制构件。

让我们看一个例子！

此示例应用程序可在 https://github.com/nigelpoulton/atsea-sample-shop-app.git 上找到，Dockerfile 位于`app`目录中。这是一个基于 Linux 的应用程序，因此只能在 Linux Docker 主机上运行。

该存储库是`dockersamples/atsea-sample-shop-app`的一个分支，我已经分叉了它，以防上游存储库被删除或删除。

Dockerfile 如下所示：

```
FROM node:latest AS storefront
WORKDIR /usr/src/atsea/app/react-app
COPY react-app .
RUN npm install
RUN npm run build

FROM maven:latest AS appserver
WORKDIR /usr/src/atsea
COPY pom.xml .
RUN mvn -B -f pom.xml -s /usr/share/maven/ref/settings-docker.xml dependency\
:resolve
COPY . .
RUN mvn -B -s /usr/share/maven/ref/settings-docker.xml package -DskipTests

FROM java:8-jdk-alpine AS production
RUN adduser -Dh /home/gordon gordon
WORKDIR /static
COPY --from=storefront /usr/src/atsea/app/react-app/build/ .
WORKDIR /app
COPY --from=appserver /usr/src/atsea/target/AtSea-0.0.1-SNAPSHOT.jar .
ENTRYPOINT ["java", "-jar", "/app/AtSea-0.0.1-SNAPSHOT.jar"]
CMD ["--spring.profiles.active=postgres"] 
```

“首先要注意的是 Dockerfile 有三个`FROM`指令。每个都构成一个独立的**构建阶段**。在内部，它们从顶部开始编号为 0。但是，我们还给每个阶段起了一个友好的名字。

+   第 0 阶段称为“店面”

+   第 1 阶段称为“应用服务器”

+   第 2 阶段称为“生产”

`storefront`阶段拉取了大小超过 600MB 的`node:latest`镜像。它设置了工作目录，复制了一些应用代码，并使用了两个 RUN 指令来执行一些`npm`魔法。这增加了三层和相当大的大小。结果是一个更大的镜像，其中包含了大量的构建工具和非常少的应用代码。

`appserver`阶段拉取了大小超过 700MB 的`maven:latest`镜像。它通过两个 COPY 指令和两个 RUN 指令添加了四层内容。这产生了另一个非常大的镜像，其中包含了大量的构建工具和非常少的实际生产代码。

生产阶段从拉取`java:8-jdk-alpine`镜像开始。这个镜像大约 150MB，比之前构建阶段使用的 node 和 maven 镜像要小得多。它添加了一个用户，设置了工作目录，并从`storefront`阶段生成的镜像中复制了一些应用代码。之后，它设置了一个不同的工作目录，并从`appserver`阶段生成的镜像中复制了应用程序代码。最后，它设置了启动容器时要运行的主应用程序镜像。

需要注意的一件重要的事情是，`COPY --from`指令用于**仅从前几个阶段构建的镜像中复制与生产相关的应用代码**。它们不会复制不需要用于生产的构建产物。

还需要注意的是，我们只需要一个 Dockerfile，并且`docker image build`命令不需要额外的参数！

说到这个……让我们来构建它。

克隆存储库。

```
$ git clone https://github.com/nigelpoulton/atsea-sample-shop-app.git

Cloning into `'atsea-sample-shop-app'`...
remote: Counting objects: `632`, `done`.
remote: Total `632` `(`delta `0``)`, reused `0` `(`delta `0``)`, pack-reused `632`
Receiving objects: `100`% `(``632`/632`)`, `7`.23 MiB `|` `1`.88 MiB/s, `done`.
Resolving deltas: `100`% `(``195`/195`)`, `done`.
Checking connectivity... `done`. 
```

切换到克隆存储库的`app`文件夹，并验证 Dockerfile 是否存在。

```
$ `cd` atsea-sample-shop-app/app

$ ls -l
total `24`
-rw-r--r-- `1` root root  `682` Oct  `1` `22`:03 Dockerfile
-rw-r--r-- `1` root root `4365` Oct  `1` `22`:03 pom.xml
drwxr-xr-x `4` root root `4096` Oct  `1` `22`:03 react-app
drwxr-xr-x `4` root root `4096` Oct  `1` `22`:03 src 
```

进行构建（这可能需要几分钟才能完成）。

```
$ docker image build -t multi:stage .

Sending build context to Docker daemon  `3`.658MB
Step `1`/19 : FROM node:latest AS storefront
latest: Pulling from library/node
aa18ad1a0d33: Pull `complete`
15a33158a136: Pull `complete`
<Snip>
Step `19`/19 : CMD --spring.profiles.active`=`postgres
 ---> Running in b4df9850f7ed
 ---> 3dc0d5e6223e
Removing intermediate container b4df9850f7ed
Successfully built 3dc0d5e6223e
Successfully tagged multi:stage 
```

> **注意：**上面示例中使用的`multi:stage`标签是任意的。您可以根据自己的要求和标准为镜像打标签 - 没有必要像我们在这个示例中那样为多阶段构建打标签。

运行`docker image ls`以查看构建操作拉取和创建的镜像列表。

```
$ docker image ls

REPO    TAG             IMAGE ID        CREATED        SIZE
node    latest          9ea1c3e33a0b    `4` days ago     673MB
<none>  <none>          6598db3cefaf    `3` mins ago     816MB
maven   latest          cbf114925530    `2` weeks ago    750MB
<none>  <none>          d5b619b83d9e    `1` min ago      891MB
java    `8`-jdk-alpine    3fd9dd82815c    `7` months ago   145MB
multi   stage           3dc0d5e6223e    `1` min ago      210MB 
```

上面输出的第一行显示了`storefront`阶段拉取的`node:latest`镜像。下面的镜像是该阶段生成的镜像（通过添加代码并运行 npm install 和构建操作创建）。两者都是非常大的镜像，包含了大量的构建工具。

第三和第四行是由`appserver`阶段拉取和生成的镜像。这两个镜像都很大，包含了很多构建工具。

最后一行是由 Dockerfile 中最终构建阶段（stage2/production）构建的`multi:stage`镜像。你可以看到，这个镜像比之前阶段拉取和生成的镜像要小得多。这是因为它是基于更小的`java:8-jdk-alpine`镜像，并且只添加了前几个阶段的与生产相关的应用文件。

最终结果是通过一个普通的`docker image build`命令和零额外脚本创建的小型生产镜像！

多阶段构建是 Docker 17.05 中的新功能，非常适合构建小型的生产级镜像。

#### 一些最佳实践。

在结束本章之前，让我们列举一些最佳实践。这个列表并不打算是详尽无遗的。

##### 利用构建缓存

Docker 使用的构建过程有一个缓存的概念。看到缓存的影响最好的方法是在一个干净的 Docker 主机上构建一个新的镜像，然后立即重复相同的构建。第一次构建将拉取镜像并花费时间构建层。第二次构建将几乎立即完成。这是因为第一次构建的产物，比如层，被缓存并被后续构建所利用。

正如我们所知，`docker image build` 过程是逐行迭代 Dockerfile，从顶部开始。对于每个指令，Docker 会查看它的缓存中是否已经有了该指令的镜像层。如果有，这就是*缓存命中*，它会使用该层。如果没有，这就是*缓存未命中*，它会根据该指令构建一个新的层。获得*缓存命中*可以极大地加快构建过程。

让我们再仔细看一下。

我们将使用这个示例 Dockerfile 进行快速演示：

```
FROM alpine
RUN apk add --update nodejs nodejs-npm
COPY . /src
WORKDIR /src
RUN npm install
EXPOSE 8080
ENTRYPOINT ["node", "./app.js"] 
```

`第一条指令告诉 Docker 使用`alpine:latest`镜像作为其*基础镜像*。如果该镜像已经存在于主机上，构建将继续进行到下一条指令。如果该镜像不存在，它将从 Docker Hub（docker.io）上拉取。

下一条指令（`RUN apk...`）针对镜像运行一个命令。此时，Docker 会检查构建缓存，查找是否有一个层是从相同的基础镜像构建的，并且使用了当前要执行的相同指令。在这种情况下，它正在寻找一个直接在`alpine:latest`之上构建的层，通过执行`RUN apk add --update nodejs nodejs-npm`指令。

如果它找到了一个层，它会跳过该指令，链接到该现有层，并继续使用缓存进行构建。如果它**没有**找到一个层，它会使缓存失效并构建该层。使缓存失效的操作会使其在剩余的构建过程中失效。这意味着所有后续的 Dockerfile 指令都将完全完成，而不会尝试引用构建缓存。

假设 Docker 已经在缓存中为此指令创建了一个层（缓存命中）。假设该层的 ID 是`AAA`。

下一条指令将一些代码复制到镜像中（`COPY . /src`）。由于前一条指令导致了缓存命中，Docker 现在会检查是否有一个缓存的层是从`AAA`层使用`COPY . /src`命令构建的。如果有，它会链接到该层并继续执行下一条指令。如果没有，它会构建该层并使得剩余的构建过程缓存失效。

假设 Docker 已经在缓存中为此指令创建了一个层（缓存命中）。假设该层的 ID 是`BBB`。

该过程将继续进行，直到 Dockerfile 的其余部分。

重要的是要理解一些事情。

首先，一旦任何指令导致缓存未命中（没有找到该指令的层），缓存将不再用于整个构建的剩余部分。这对您如何编写 Dockerfile 有重要影响。尝试以一种方式构建它们，将可能更改的任何指令放在文件末尾。这意味着直到构建的后期阶段才会发生缓存未命中，从而使构建尽可能多地受益于缓存。

您可以通过向`docker image build`命令传递`--no-cache=true`标志来强制构建过程忽略整个缓存。

还重要的是要理解`COPY`和`ADD`指令包括确保复制到镜像中的内容自上次构建以来没有更改的步骤。例如，Dockerfile 中的`COPY . /src`指令可能自上次构建以来没有更改，**但是...**被复制到镜像中的目录的内容**已经**发生了变化！

为了防止这种情况发生，Docker 对每个被复制的文件执行校验和，并将其与缓存层中相同文件的校验和进行比较。如果校验和不匹配，则缓存将被作废，并构建一个新的层。

##### 压缩镜像

压缩镜像并不是一个最佳实践，因为它有利有弊。

在高层次上，Docker 遵循构建镜像的正常流程，但然后添加了一个额外的步骤，将所有内容压缩成一个单一层。

在镜像开始具有大量层并且这并不理想的情况下，压缩可能是有益的。例如，当创建一个新的基础镜像，您希望将来从中构建其他镜像时 - 这作为单层镜像要好得多。

消极的一面是，压缩的镜像不共享镜像层。这可能导致存储效率低下和更大的推送和拉取操作。

如果要创建压缩的镜像，请在`docker image build`命令中添加`--squash`标志。

图 8.8 显示了压缩镜像带来的一些效率低下。两个镜像除了一个是压缩的，另一个不是，其他都完全相同。压缩的镜像与主机上的其他镜像共享层（节省磁盘空间），但压缩的镜像不共享。压缩的镜像还需要在`docker image push`命令中发送每个字节到 Docker Hub，而非压缩的镜像只需要发送唯一的层。

![图 8.8 - 压缩镜像与非压缩镜像](img/figure8-8.png)

图 8.8 - 压缩镜像与非压缩镜像

##### 不安装推荐软件

如果您正在构建 Linux 镜像，并使用 apt 软件包管理器，您应该在`apt-get install`命令中使用`no-install-recommends`标志。这可以确保`apt`只安装主要依赖项（`Depends`字段中的软件包），而不是推荐或建议的软件包。这可以大大减少下载到镜像中的不需要的软件包的数量。

##### 不要从 MSI 软件包（Windows）安装

如果您正在构建 Windows 映像，应尽量避免使用 MSI 软件包管理器。它不够节省空间，导致映像比所需的要大得多。

### 容器化应用程序 - 命令

+   `docker image build`是一个读取 Dockerfile 并将应用程序容器化的命令。`-t`标志为映像打标签，`-f`标志允许您指定 Dockerfile 的名称和位置。使用`-f`标志，可以使用任意名称和位置的 Dockerfile。*构建上下文*是您的应用程序文件所在的位置，可以是本地 Docker 主机上的目录，也可以是远程 Git 存储库。

+   Dockerfile 中的`FROM`指令指定了要构建的新映像的基础映像。通常是 Dockerfile 中的第一条指令。

+   Dockerfile 中的`RUN`指令允许您在映像内部运行命令，从而创建新的层。每个`RUN`指令都会创建一个新的层。

+   Dockerfile 中的`COPY`指令将文件添加到映像中作为一个新层。通常使用`COPY`指令将应用程序代码复制到映像中。

+   Dockerfile 中的`EXPOSE`指令记录了应用程序使用的网络端口。

+   Dockerfile 中的`ENTRYPOINT`指令设置了在将映像启动为容器时要运行的默认应用程序。

+   其他 Dockerfile 指令包括`LABEL`、`ENV`、`ONBUILD`、`HEALTHCHECK`、`CMD`等等…

### 章节总结

在本章中，我们学习了如何将应用程序容器化（Docker 化）。

我们从远程 Git 存储库中提取了一些应用程序代码。存储库包括应用程序代码以及一个包含如何将应用程序构建成映像的 Dockerfile。我们学习了 Dockerfile 的基础知识，并将其输入到`docker image build`命令中以创建一个新的映像。

创建映像后，我们启动了一个容器并使用 Web 浏览器测试了它的工作情况。

之后，我们了解了多阶段构建如何为我们提供了一种简单的方式来构建和部署更小的映像到生产环境。

我们还了解到 Dockerfile 是一个很好的工具，可以用来记录应用程序。因此，它可以加快新开发人员的入职速度，并弥合开发人员和运维人员之间的鸿沟！考虑到这一点，将其视为代码，并将其检入和检出源代码控制系统。

尽管引用的例子是一个基于 Linux 的例子，但容器化 Windows 应用程序的过程是相同的：从您的应用程序代码开始，创建描述应用程序的 Dockerfile，使用`docker image build`构建镜像。工作完成！

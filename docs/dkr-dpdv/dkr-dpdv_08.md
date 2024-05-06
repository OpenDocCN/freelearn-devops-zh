## 6：镜像

在本章中，我们将深入探讨 Docker 镜像。游戏的目标是让您对 Docker 镜像有一个**扎实的理解**，以及如何执行基本操作。在后面的章节中，我们将看到如何在其中构建包含我们自己应用程序的新镜像（容器化应用程序）。

我们将把本章分为通常的三个部分：

+   TLDR

+   深入了解

+   命令

让我们去学习关于镜像的知识吧！

### Docker 镜像- TLDR

如果您曾经是虚拟机管理员，您可以将 Docker 镜像视为类似于 VM 模板。VM 模板就像是一个停止的 VM- Docker 镜像就像是一个停止的容器。如果您是开发人员，您可以将它们视为类似于*类*。

您可以从镜像注册表中*拉取*镜像开始。最受欢迎的注册表是[Docker Hub](https://hub.docker.com)，但也存在其他注册表。*拉取*操作会将镜像下载到本地 Docker 主机，您可以在其中使用它来启动一个或多个 Docker 容器。

镜像由多个层组成，这些层堆叠在一起并表示为单个对象。镜像中包含一个精简的操作系统（OS）以及运行应用程序所需的所有文件和依赖项。由于容器旨在快速和轻量级，因此镜像往往很小。

恭喜！您现在对 Docker 镜像有了一点概念 :-D 现在是时候让您大开眼界了！

### Docker 镜像-深入了解

我们已经多次提到**镜像**就像是停止的容器（或者如果您是开发人员，就像是**类**）。事实上，您可以停止一个容器并从中创建一个新的镜像。考虑到这一点，镜像被认为是*构建时*构造，而容器是*运行时*构造。

![图 6.1](img/figure6-1.png)

图 6.1

#### 镜像和容器

图 6.1 显示了镜像和容器之间关系的高层视图。我们使用`docker container run`和`docker service create`命令从单个镜像启动一个或多个容器。但是，一旦您从镜像启动了一个容器，这两个构造就会相互依赖，直到最后一个使用它的容器停止和销毁之前，您不能删除该镜像。尝试删除一个正在使用的镜像而不停止和销毁所有使用它的容器将导致以下错误：

```
$ docker image rm <image-name>
Error response from daemon: conflict: unable to remove repository reference `\`
`"<image-name>"` `(`must force`)` - container <container-id> is using its referenc`\`
ed image <image-id> 
```

`#### 镜像通常很小

容器的整个目的是运行应用程序或服务。这意味着容器创建的镜像必须包含运行应用程序/服务所需的所有操作系统和应用程序文件。但是，容器的全部意义在于快速和轻量级。这意味着它们构建的镜像通常很小，并且剥离了所有非必要的部分。

例如，Docker 镜像不会为您提供 6 种不同的 shell 供您选择 - 它们通常只提供一个最小化的 shell，或者根本不提供 shell。它们也不包含内核 - 在 Docker 主机上运行的所有容器共享对主机内核的访问。因此，我们有时会说镜像包含*足够的操作系统*（通常只有与操作系统相关的文件和文件系统对象）。

> **注意：** Hyper-V 容器在专用的轻量级虚拟机内运行，并利用虚拟机内运行的操作系统的内核。

官方的*Alpine Linux* Docker 镜像大约为 4MB，是 Docker 镜像可以有多小的一个极端示例。这不是打字错误！它确实大约为 4 兆字节！然而，一个更典型的例子可能是官方的 Ubuntu Docker 镜像，目前大约为 110MB。这些显然剥离了大多数非必要的部分！

基于 Windows 的镜像往往比基于 Linux 的镜像大，这是因为 Windows 操作系统的工作方式。例如，最新的 Microsoft .NET 镜像（`microsoft/dotnet:latest`）在拉取和解压缩时超过 1.7GB。Windows Server 2016 Nano Server 镜像（`microsoft/nanoserver:latest`）在拉取和解压缩时略大于 1GB。

#### 拉取镜像

干净安装的 Docker 主机在其本地存储库中没有任何镜像。

基于 Linux 的 Docker 主机上的本地镜像存储库通常位于`/var/lib/docker/<storage-driver>`。在基于 Windows 的 Docker 主机上，这是`C:\ ProgramData\docker\windowsfilter`。

您可以使用以下命令检查您的 Docker 主机是否在其本地存储库中有任何镜像。

```
$ docker image ls
REPOSITORY  TAG      IMAGE ID       CREATED         SIZE 
```

将镜像放入 Docker 主机的过程称为*拉取*。因此，如果您想要在 Docker 主机上获取最新的 Ubuntu 镜像，您需要*拉取*它。使用以下命令*拉取*一些镜像，然后检查它们的大小。

> 如果您在 Linux 上进行操作，并且尚未将您的用户帐户添加到本地`docker` Unix 组中，则可能需要在所有以下命令的开头添加`sudo`。

Linux 示例：

```
$ docker image pull ubuntu:latest

latest: Pulling from library/ubuntu
b6f892c0043b: Pull `complete`
55010f332b04: Pull `complete`
2955fb827c94: Pull `complete`
3deef3fcbd30: Pull `complete`
cf9722e506aa: Pull `complete`
Digest: sha256:38245....44463c62a9848133ecb1aa8
Status: Downloaded newer image `for` ubuntu:latest

$ docker image pull alpine:latest

latest: Pulling from library/alpine
cfc728c1c558: Pull `complete`
Digest: sha256:c0537...497c0a7726c88e2bb7584dc96
Status: Downloaded newer image `for` alpine:latest

$ docker image ls

REPOSITORY   TAG     IMAGE ID        CREATED       SIZE
ubuntu       latest  ebcd9d4fca80    `3` days ago    118MB
alpine       latest  02674b9cb179    `8` days ago    `3`.99MB 
```

Windows 示例：

```
> docker image pull microsoft/powershell:nanoserver

nanoserver: Pulling from microsoft/powershell
bce2fbc256ea: Pull complete
58f68fa0ceda: Pull complete
04083aac0446: Pull complete
e42e2e34b3c8: Pull complete
0c10d79c24d4: Pull complete
715cb214dca4: Pull complete
a4837c9c9af3: Pull complete
2c79a32d92ed: Pull complete
11a9edd5694f: Pull complete
d223b37dbed9: Pull complete
aee0b4393afb: Pull complete
0288d4577536: Pull complete
8055826c4f25: Pull complete
Digest: sha256:090fe875...fdd9a8779592ea50c9d4524842
Status: Downloaded newer image for microsoft/powershell:nanoserver
>
> docker image pull microsoft/dotnet:latest

latest: Pulling from microsoft/dotnet
bce2fbc256ea: Already exists
4a8c367fd46d: Pull complete
9f49060f1112: Pull complete
0334ad7e5880: Pull complete
ea8546db77c6: Pull complete
710880d5cbd5: Pull complete
d665d26d9a25: Pull complete
caa8d44fb0b1: Pull complete
cfd178ff221e: Pull complete
Digest: sha256:530343cd483dc3e1...6f0378e24310bd67d2a
Status: Downloaded newer image for microsoft/dotnet:latest
>
> docker image ls
REPOSITORY            TAG         IMAGE ID    CREATED     SIZE
microsoft/dotnet      latest      831..686d   7 hrs ago   1.65 GB
microsoft/powershell  nanoserver  d06..5427   8 days ago  1.21 GB 
```

正如你所看到的，刚刚拉取的镜像现在存在于 Docker 主机的本地存储库中。你还可以看到 Windows 镜像要大得多，并且包含了更多的层。

#### 镜像命名

在每个命令的一部分，我们必须指定要拉取的镜像。所以让我们花点时间来看看镜像命名。为此，我们需要了解一些关于如何存储镜像的背景知识。

#### 镜像注册表

Docker 镜像存储在*镜像注册表*中。最常见的注册表是 Docker Hub（https://hub.docker.com）。还有其他注册表，包括第三方注册表和安全的本地注册表。然而，Docker 客户端有自己的偏好，并默认使用 Docker Hub。我们将在本书的其余部分使用 Docker Hub。

镜像注册表包含多个*镜像存储库*。反过来，镜像存储库可以包含多个镜像。这可能有点令人困惑，因此图 6.2 显示了一个包含 3 个存储库的镜像注册表的图片，每个存储库包含一个或多个镜像。

![图 6.2](img/figure6-2.png)

图 6.2

##### 官方和非官方存储库

Docker Hub 还有*官方存储库*和*非官方存储库*的概念。

正如名称所示，*官方存储库*包含了经过 Docker, Inc.审核的镜像。这意味着它们应该包含最新的、高质量的代码，安全、有良好的文档，并符合最佳实践（请问我能否因在一个句子中使用了五个连字符而获得奖励）。

*非官方存储库*可能像是荒野——你不应该*期望*它们是安全的、有良好的文档或者按照最佳实践构建的。这并不是说*非官方存储库*中的所有东西都是坏的！*非官方存储库*中有一些**精彩**的东西。你只需要在信任它们的代码之前非常小心。老实说，当从互联网获取软件时，你应该始终小心——甚至是从*官方存储库*获取的镜像！

大多数流行的操作系统和应用程序在 Docker Hub 上都有自己的*官方存储库*。它们很容易识别，因为它们位于 Docker Hub 命名空间的顶层。以下列表包含了一些*官方存储库*，并显示了它们在 Docker Hub 命名空间顶层存在的 URL：

+   **nginx:** https://hub.docker.com/_/nginx/

+   **busybox:** https://hub.docker.com/_/busybox/

+   **redis:** https://hub.docker.com/_/redis/

+   **mongo:** https://hub.docker.com/_/mongo/

另一方面，我的个人图像存储在*非官方存储库*的荒野中，不应该被信任！以下是我存储库中图像的一些示例：

+   nigelpoulton/tu-demo

https://hub.docker.com/r/nigelpoulton/tu-demo/

+   nigelpoulton/pluralsight-docker-ci

https://hub.docker.com/r/nigelpoulton/pluralsight-docker-ci/

我的存储库中的图像不仅没有经过审查，也没有及时更新，不安全，文档也不完善... 你还应该注意到它们并不位于 Docker Hub 命名空间的顶层。我的存储库都位于一个名为`nigelpoulton`的二级命名空间中。

你可能会注意到我们使用的 Microsoft 图像并不位于 Docker Hub 命名空间的顶层。在撰写本文时，它们存在于`microsoft`的二级命名空间下。

经过所有这些之后，我们终于可以看一下如何在 Docker 命令行中处理图像。

#### 图像命名和标记

从官方存储库中寻址图像就像简单地给出存储库名称和标签，用冒号（`:`）分隔。当使用来自官方存储库的图像时，`docker image pull`的格式为：

`docker image pull <repository>:<tag>`

在之前的 Linux 示例中，我们使用以下两个命令拉取了 Alpine 和 Ubuntu 图像：

`docker image pull alpine:latest` 和 `docker image pull ubuntu:latest`

这两个命令从“alpine”和“ubuntu”存储库中拉取标记为“latest”的图像。

以下示例展示了如何从*官方存储库*中拉取不同的图像：

```
$ docker image pull mongo:3.3.11
//This will pull the image tagged as ````3`.3.11```
//from the official ```mongo``` repository.

$ docker image pull redis:latest
//This will pull the image tagged as ```latest```
//from the official ```redis``` repository.

$ docker image pull alpine
//This will pull the image tagged as ```latest```
//from the official ```alpine``` repository. 
```

`关于这些命令的一些要点。

首先，如果在存储库名称后**没有**指定图像标签，Docker 将假定你指的是标记为`latest`的图像。

其次，`latest`标签并没有任何神奇的功能！仅仅因为一个图像被标记为`latest`并不意味着它是存储库中最新的图像！例如，`alpine`存储库中最新的图像通常被标记为`edge`。故事的寓意是——在使用`latest`标签时要小心！

从*非官方仓库*中拉取图像本质上是一样的——你只需要在仓库名称前加上一个 Docker Hub 用户名或组织名称。下面的例子展示了如何从一个不可信任的人拥有的 Docker Hub 帐户名为`nigelpoulton`的`tu-demo`仓库中拉取`v2`图像。

```
$ docker image pull nigelpoulton/tu-demo:v2
//This will pull the image tagged as ```v2```
//from the ```tu-demo``` repository within the namespace
//of my personal Docker Hub account. 
```

`在我们之前的 Windows 示例中，我们用以下两个命令拉取了一个 PowerShell 和一个.NET 图像：

`> docker image pull microsoft/powershell:nanoserver`

`> docker image pull microsoft/dotnet:latest`

第一个命令从`microsoft/powershell`仓库中拉取标记为`nanoserver`的图像。第二个命令从`microsoft/dotnet`仓库中拉取标记为`latest`的图像。

如果你想从第三方注册表（而不是 Docker Hub）中拉取图像，你需要在仓库名称前加上注册表的 DNS 名称。例如，如果上面的例子中的图像在 Google 容器注册表（GCR）中，你需要在仓库名称前添加`gcr.io`，如下所示——`docker pull gcr.io/nigelpoulton/tu-demo:v2`（没有这样的仓库和图像存在）。

你可能需要在从第三方注册表中拉取图像之前在其上拥有一个帐户并登录。

#### 具有多个标签的图像

关于图像标签的最后一句话…… 一个单独的图像可以有任意多个标签。这是因为标签是存储在图像旁边的元数据的任意字母数字值。让我们来看一个例子。

通过在`docker image pull`命令中添加`-a`标志来拉取仓库中的所有图像。然后运行`docker image ls`来查看拉取的图像。如果你正在使用 Windows，你可以从`microsoft/nanoserver`仓库中拉取，而不是`nigelpoulton/tu-demo`。

> **注意：**如果你从中拉取的仓库包含多个架构和平台的图像，比如 Linux **和** Windows，该命令可能会失败。

```
$ docker image pull -a nigelpoulton/tu-demo

latest: Pulling from nigelpoulton/tu-demo
237d5fcd25cf: Pull `complete`
a3ed95caeb02: Pull `complete`
<Snip>
Digest: sha256:42e34e546cee61adb1...3a0c5b53f324a9e1c1aae451e9
v1: Pulling from nigelpoulton/tu-demo
237d5fcd25cf: Already exists
a3ed95caeb02: Already exists
<Snip>
Digest: sha256:9ccc0c67e5c5eaae4b...624c1d5c80f2c9623cbcc9b59a
v2: Pulling from nigelpoulton/tu-demo
237d5fcd25cf: Already exists
a3ed95caeb02: Already exists
<Snip>
Digest: sha256:d3c0d8c9d5719d31b7...9fef58a7e038cf0ef2ba5eb74c
Status: Downloaded newer image `for` nigelpoulton/tu-demo

$ docker image ls
REPOSITORY            TAG       IMAGE ID       CREATED    SIZE
nigelpoulton/tu-demo   v2       6ac21e..bead   `1` yr ago   `211`.6 MB
nigelpoulton/tu-demo   latest   9b915a..1e29   `1` yr ago   `211`.6 MB
nigelpoulton/tu-demo   v1       9b915a..1e29   `1` yr ago   `211`.6 MB 
```

关于刚刚发生的一些事情：

首先，该命令从`nigelpoulton/tu-demo`仓库中拉取了三个图像：`latest`、`v1`和`v2`。

其次，请仔细查看`docker image ls`命令的输出中的`IMAGE ID`列。您会发现只有两个唯一的图像 ID。这是因为实际上只有两个图像被下载。这是因为两个标签指向相同的图像。换句话说...其中一个图像有两个标签。如果您仔细观察，您会发现`v1`和`latest`标签具有相同的`IMAGE ID`。这意味着它们是**同一图像**的两个标签。

这是一个关于`latest`标签的警告的完美例子。在这个例子中，`latest`标签指的是与`v1`标签相同的图像。这意味着它指向两个图像中较旧的一个，而不是最新的一个！`latest`是一个任意的标签，并不能保证指向存储库中最新的图像！

#### 过滤`docker image ls`的输出

Docker 提供了`--filter`标志来过滤由`docker image ls`返回的图像列表。

以下示例将仅返回悬空图像。

```
$ docker image ls --filter `dangling``=``true`
REPOSITORY    TAG       IMAGE ID       CREATED       SIZE
<none>        <none>    4fd34165afe0   `7` days ago    `14`.5MB 
```

悬空图像是不再标记的图像，并在列表中显示为`<none>:<none>`。它们出现的常见方式是在构建新图像并使用现有标签对其进行标记时。当这种情况发生时，Docker 将构建新图像，注意到现有图像具有匹配的标签，从现有图像中删除标签，并将标签赋予新图像。例如，您基于`alpine:3.4`构建了一个新图像，并将其标记为`dodge:challenger`。然后，您更新 Dockerfile 以将`alpine:3.4`替换为`alpine:3.5`，并运行完全相同的`docker image build`命令。构建将创建一个新图像，标记为`dodge:challenger`，并从旧图像中删除标签。旧图像将变成悬空图像。

您可以使用`docker image prune`命令删除系统上的所有悬空图像。如果添加`-a`标志，Docker 还将删除所有未使用的图像（即任何容器未使用的图像）。

Docker 目前支持以下过滤器：

+   `dangling:`接受`true`或`false`，并仅返回悬空图像（true）或非悬空图像（false）。

+   `before:`需要一个图像名称或 ID 作为参数，并返回在其之前创建的所有图像。

+   `since:`与上述相同，但返回在指定图像之后创建的图像。

+   `label:`根据标签或标签和值的存在来过滤图像。`docker image ls`命令不会在其输出中显示标签。

对于所有其他过滤，您可以使用`reference`。

以下是一个使用`reference`来仅显示标记为“latest”的图像的例子。

```
$ docker image ls --filter`=``reference``=``"*:latest"`
REPOSITORY   TAG      IMAGE ID        CREATED        SIZE
alpine       latest   3fd9065eaf02    `8` days ago     `4`.15MB
`test`         latest   8426e7efb777    `3` days ago     122MB 
```

`您还可以使用`--format`标志使用 Go 模板格式化输出。例如，以下命令将仅返回 Docker 主机上图像的大小属性。

```
$ docker image ls --format `"{{.Size}}"`
`99`.3MB
111MB
`82`.6MB
`88`.8MB
`4`.15MB
108MB 
```

`使用以下命令返回所有图像，但仅显示存储库、标签和大小。

```
$ docker image ls --format `"{{.Repository}}: {{.Tag}}: {{.Size}}"`
dodge:  challenger: `99`.3MB
ubuntu: latest:     111MB
python: `3`.4-alpine: `82`.6MB
python: `3`.5-alpine: `88`.8MB
alpine: latest:     `4`.15MB
nginx:  latest:     108MB 
```

`如果您需要更强大的过滤功能，您可以随时使用操作系统和 shell 提供的工具，如`grep`和`awk`。

#### 从 CLI 搜索 Docker Hub

`docker search`命令允许您从 CLI 搜索 Docker Hub。您可以针对“NAME”字段中的字符串进行模式匹配，并根据返回的任何列来过滤输出。

在其最简单的形式中，它搜索包含在“NAME”字段中的特定字符串的所有存储库。例如，以下命令搜索所有在“NAME”字段中包含“nigelpoulton”的存储库。

```
$ docker search nigelpoulton
NAME                         DESCRIPTION               STARS   AUTOMATED
nigelpoulton/pluralsight..   Web app used in...        `8`       `[`OK`]`
nigelpoulton/tu-demo                                   `7`
nigelpoulton/k8sbook         Kubernetes Book web app   `1`
nigelpoulton/web-fe1         Web front end example     `0`
nigelpoulton/hello-cloud     Quick hello-world image   `0` 
```

“NAME”字段是存储库名称，并包括非官方存储库的 Docker ID 或组织名称。例如，以下命令将列出所有包含名称中包含“alpine”的存储库。

```
$ docker search alpine
NAME                   DESCRIPTION          STARS    OFFICIAL    AUTOMATED
alpine                 A minimal Docker..   `2988`     `[`OK`]`
mhart/alpine-node      Minimal Node.js..    `332`
anapsix/alpine-java    Oracle Java `8`...     `270`                  `[`OK`]`
<Snip> 
```

`注意一下，返回的一些存储库是官方的，一些是非官方的。您可以使用`--filter "is-official=true"`，这样只有官方存储库才会显示。

```
$ docker search alpine --filter `"is-official=true"`
NAME                   DESCRIPTION          STARS    OFFICIAL    AUTOMATED
alpine                 A minimal Docker..   `2988`     `[`OK`]` 
```

`您可以再次执行相同的操作，但这次只显示具有自动构建的存储库。

```
$ docker search alpine --filter `"is-automated=true"`
NAME                       DESCRIPTION               OFFICIAL     AUTOMATED
anapsix/alpine-java        Oracle Java `8` `(`and `7``)`..                `[`OK`]`
frolvlad/alpine-glibc      Alpine Docker image..                  `[`OK`]`
kiasaki/alpine-postgres    PostgreSQL docker..                    `[`OK`]`
zzrot/alpine-caddy         Caddy Server Docker..                  `[`OK`]`
<Snip> 
```

`关于`docker search`的最后一件事。默认情况下，Docker 只会显示 25 行结果。但是，您可以使用`--limit`标志将其增加到最多 100 行。

#### 图像和层

Docker 图像只是一堆松散连接的只读层。这在图 6.3 中显示出来。

![图 6.3](img/figure6-3.png)

图 6.3

Docker 负责堆叠这些层并将它们表示为单个统一的对象。

有几种方法可以查看和检查构成图像的层，我们已经看到其中一种。让我们再次看一下之前`docker image pull ubuntu:latest`命令的输出：

```
$ docker image pull ubuntu:latest
latest: Pulling from library/ubuntu
952132ac251a: Pull `complete`
82659f8f1b76: Pull `complete`
c19118ca682d: Pull `complete`
8296858250fe: Pull `complete`
24e0251a0e2c: Pull `complete`
Digest: sha256:f4691c96e6bbaa99d...28ae95a60369c506dd6e6f6ab
Status: Downloaded newer image `for` ubuntu:latest 
```

`上面输出中以“Pull complete”结尾的每一行代表了被拉取的图像中的一个层。正如我们所看到的，这个图像有 5 个层。图 6.4 以图片形式显示了这一点，显示了层 ID。

![图 6.4](img/figure6-4.png)

图 6.4

查看图像的层的另一种方法是使用`docker image inspect`命令检查图像。以下示例检查了相同的`ubuntu:latest`图像。

```
$ docker image inspect ubuntu:latest
`[`
    `{`
        `"Id"`: `"sha256:bd3d4369ae.......fa2645f5699037d7d8c6b415a10"`,
        `"RepoTags"`: `[`
            `"ubuntu:latest"`

        <Snip>

        `"RootFS"`: `{`
            `"Type"`: `"layers"`,
            `"Layers"`: `[`
                `"sha256:c8a75145fc...894129005e461a43875a094b93412"`,
                `"sha256:c6f2b330b6...7214ed6aac305dd03f70b95cdc610"`,
                `"sha256:055757a193...3a9565d78962c7f368d5ac5984998"`,
                `"sha256:4837348061...12695f548406ea77feb5074e195e3"`,
                `"sha256:0cad5e07ba...4bae4cfc66b376265e16c32a0aae9"`
            `]`
        `}`
    `}`
`]` 
```

修剪后的输出再次显示了 5 个图层。只是这一次它们使用它们的 SHA256 哈希值显示。然而，两个命令都显示该图像有 5 个图层。

> **注意：**`docker history`命令显示图像的构建历史，**不是**图像中图层的严格列表。例如，用于构建图像的一些 Dockerfile 指令不会创建图层。这些指令包括：“ENV”、“EXPOSE”、“CMD”和“ENTRYPOINT”。这些指令不会创建新的图层，而是向图像添加元数据。

所有的 Docker 图像都以一个基本图层开始，随着更改和新内容的添加，新的图层会被添加到顶部。

作为一个过度简化的例子，你可能会创建一个基于 Ubuntu Linux 16.04 的新图像。这将是你的图像的第一层。如果你稍后添加 Python 包，这将作为第二层添加到基本图层之上。如果你随后添加了一个安全补丁，这将作为第三层添加到顶部。你的图像现在有三个图层，如图 6.5 所示（请记住，这只是一个为了演示目的而过度简化的例子）。

![图 6.5](img/figure6-5.png)

图 6.5

重要的是要理解，随着添加额外的图层，*图像*始终是所有图层的组合。以图 6.6 中显示的两个图层为例。每个*图层*有 3 个文件，但整体*图像*有 6 个文件，因为它是两个图层的组合。

![图 6.6](img/figure6-6.png)

图 6.6

> **注意：**我们在图 6.6 中以略有不同的方式显示了图像图层，这只是为了更容易地显示文件。

在图 6.7 中更复杂的三层图像的例子中，统一视图中的整体图像只呈现了 6 个文件。这是因为顶层的文件 7 是直接下方文件 5 的更新版本（内联）。在这种情况下，更高层的文件遮盖了直接下方的文件。这允许将文件的更新版本作为图像的新图层添加。

![图 6.7](img/figure6-7.png)

图 6.7

Docker 使用存储驱动程序（较新版本中的快照程序）负责堆叠图层并将它们呈现为单一的统一文件系统。Linux 上的存储驱动程序示例包括`AUFS`、`overlay2`、`devicemapper`、`btrfs`和`zfs`。正如它们的名称所暗示的那样，每个驱动程序都基于 Linux 文件系统或块设备技术，并且每个驱动程序都具有其独特的性能特征。Windows 上 Docker 支持的唯一驱动程序是`windowsfilter`，它在 NTFS 之上实现了分层和写时复制（CoW）。

图 6.8 显示了与系统中将显示的相同的 3 层图像。即所有三个图层堆叠和合并，形成单一的统一视图。

![图 6.8](img/figure6-8.png)

图 6.8

#### 共享图像层

多个图像可以共享图层，这导致了空间和性能的效率。

让我们再次看看我们之前使用`docker image pull`命令和`-a`标志来拉取`nigelpoulton/tu-demo`存储库中的所有标记图像。

```
$ docker image pull -a nigelpoulton/tu-demo

latest: Pulling from nigelpoulton/tu-demo
237d5fcd25cf: Pull `complete`
a3ed95caeb02: Pull `complete`
<Snip>
Digest: sha256:42e34e546cee61adb100...a0c5b53f324a9e1c1aae451e9

v1: Pulling from nigelpoulton/tu-demo
237d5fcd25cf: Already exists
a3ed95caeb02: Already exists
<Snip>
Digest: sha256:9ccc0c67e5c5eaae4beb...24c1d5c80f2c9623cbcc9b59a

v2: Pulling from nigelpoulton/tu-demo
237d5fcd25cf: Already exists
a3ed95caeb02: Already exists
<Snip>
eab5aaac65de: Pull `complete`
Digest: sha256:d3c0d8c9d5719d31b79c...fef58a7e038cf0ef2ba5eb74c
Status: Downloaded newer image `for` nigelpoulton/tu-demo

$ docker image ls
REPOSITORY             TAG      IMAGE ID       CREATED        SIZE
nigelpoulton/tu-demo   v2       6ac...ead   `4` months ago   `211`.6 MB
nigelpoulton/tu-demo   latest   9b9...e29   `4` months ago   `211`.6 MB
nigelpoulton/tu-demo   v1       9b9...e29   `4` months ago   `211`.6 MB 
```

注意以`已存在`结尾的行。

这些行告诉我们，Docker 足够聪明，能够识别当被要求拉取已经存在副本的图像层时。在这个例子中，Docker 首先拉取了标记为`latest`的图像。然后，当它拉取`v1`和`v2`图像时，它注意到它已经有了组成这些图像的一些图层。这是因为该存储库中的三个图像几乎是相同的，因此共享许多图层。

如前所述，Linux 上的 Docker 支持许多存储驱动程序（快照程序）。每个都可以自由地以自己的方式实现图像分层、图层共享和写时复制（CoW）行为。然而，总体结果和用户体验基本相同。尽管 Windows 只支持单个存储驱动程序，但该驱动程序提供与 Linux 相同的体验。

#### 通过摘要拉取图像

到目前为止，我们已经向您展示了如何按标记拉取图像，这绝对是最常见的方式。但是它有一个问题——标记是可变的！这意味着有可能意外地使用错误的标记标记图像。有时甚至可能使用与现有但不同的图像相同的标记标记图像。这可能会引起问题！

举个例子，假设你有一个名为`golftrack:1.5`的图像，并且它有一个已知的错误。您拉取图像，应用修复，并使用**相同的标记**将更新后的图像推送回其存储库。

花点时间理解刚刚发生的事情...您有一个名为`golftrack:1.5`的镜像存在漏洞。该镜像正在您的生产环境中使用。您创建了一个包含修复的新版本的镜像。然后出现了错误...您构建并将修复后的镜像推送回其存储库，**与易受攻击的镜像使用相同的标签！**这将覆盖原始镜像，并且无法很好地知道哪些生产容器是从易受攻击的镜像运行的，哪些是从修复的镜像运行的？两个镜像都具有相同的标签！

这就是*镜像摘要*发挥作用的地方。

Docker 1.10 引入了一种新的内容可寻址存储模型。作为这种新模型的一部分，现在所有镜像都会获得一个加密的内容哈希。在本讨论中，我们将把这个哈希称为*摘要*。因为摘要是镜像内容的哈希，所以不可能更改镜像的内容而不更改摘要。这意味着摘要是不可变的。这有助于避免我们刚刚谈到的问题。

每次拉取镜像时，`docker image pull`命令将包括镜像的摘要作为返回代码的一部分。您还可以通过在`docker image ls`命令中添加`--digests`标志来查看 Docker 主机本地存储库中镜像的摘要。这两者都在以下示例中显示。

```
$ docker image pull alpine
Using default tag: latest
latest: Pulling from library/alpine
e110a4a17941: Pull `complete`
Digest: sha256:3dcdb92d7432d56604d...6d99b889d0626de158f73a
Status: Downloaded newer image `for` alpine:latest

$ docker image ls --digests alpine
REPOSITORY  TAG     DIGEST              IMAGE ID      CREATED       SIZE
alpine      latest  sha256:3dcd...f73a  4e38e38c8ce0  `10` weeks ago  `4`.8 MB 
```

`上面的输出显示了`alpine`镜像的摘要为 -

`sha256:3dcdb92d7432d56604d...6d99b889d0626de158f73a`

现在我们知道镜像的摘要，我们可以在再次拉取镜像时使用它。这将确保我们得到**完全符合我们期望的镜像！**

在撰写本文时，没有原生的 Docker 命令可以从 Docker Hub 等远程注册表中检索镜像的摘要。这意味着确定镜像的摘要的唯一方法是按标签拉取它，然后记下其摘要。这无疑将在未来发生变化。

以下示例从 Docker 主机中删除`alpine:latest`镜像，然后演示如何使用其摘要而不是标签再次拉取它。

```
$ docker image rm alpine:latest
Untagged: alpine:latest
Untagged: alpine@sha256:c0537...7c0a7726c88e2bb7584dc96
Deleted: sha256:02674b9cb179d...abff0c2bf5ceca5bad72cd9
Deleted: sha256:e154057080f40...3823bab1be5b86926c6f860

$ docker image pull alpine@sha256:c0537...7c0a7726c88e2bb7584dc96
sha256:c0537...7726c88e2bb7584dc96: Pulling from library/alpine
cfc728c1c558: Pull `complete`
Digest: sha256:c0537ff6a5218...7c0a7726c88e2bb7584dc96
Status: Downloaded newer image `for` alpine@sha256:c0537...bb7584dc96 
```

#### 关于镜像哈希（摘要）的更多信息

自 Docker 版本 1.10 以来，镜像是一个非常松散的独立层集合。

*镜像*本身实际上只是一个列出层和一些元数据的配置对象。

*层*是数据所在的地方（文件等）。每个层都是完全独立的，没有成为集体镜像的概念。

每个图像由一个加密 ID 标识，这是配置对象的哈希值。每个图层由一个加密 ID 标识，这是其包含内容的哈希值。

这意味着更改图像的内容或任何图层都将导致相关的加密哈希值发生变化。因此，图像和图层是不可变的，我们可以轻松地识别对它们所做的任何更改。

我们称这些哈希值为**内容哈希值**。

到目前为止，事情还相当简单。但它们即将变得更加复杂。

当我们推送和拉取图像时，我们会压缩它们的图层以节省带宽，以及注册表的 blob 存储空间。

很酷，但压缩图层会改变其内容！这意味着在推送或拉取操作后，其内容哈希将不再匹配！这显然是一个问题。

例如，当你将图像图层推送到 Docker Hub 时，Docker Hub 将尝试验证图像是否在传输过程中未被篡改。为了做到这一点，它会对图层运行一个哈希，并检查是否与发送的哈希匹配。因为图层被压缩（改变）了，哈希验证将失败。

为了解决这个问题，每个图层还会得到一个叫做*分发哈希*的东西。这是对图层压缩版本的哈希。当图层从注册表中推送和拉取时，它的分发哈希被包括在内，这就是用来验证图层是否在传输过程中被篡改的方法。

这种内容寻址存储模型通过在推送和拉取操作后提供一种验证图像和图层数据的方式，大大提高了安全性。它还避免了如果图像和图层 ID 是随机生成的可能发生的 ID 冲突。

#### 多架构图像

关于 Docker 最好的一点是它的简单易用。例如，运行一个应用程序就像拉取图像并运行一个容器一样简单。不需要担心设置、依赖项或配置。它就能运行。

然而，随着 Docker 的发展，事情开始变得复杂 - 尤其是当新的平台和架构，如 Windows、ARM 和 s390x 被添加进来时。突然间，我们不得不考虑我们正在拉取的图像是否是为我们正在运行的架构构建的。这破坏了流畅的体验。

多架构图像来拯救！

Docker（镜像和注册表规范）现在支持多架构镜像。这意味着单个镜像（`repository:tag`）*可以*在 x64 架构的 Linux 上，PowerPC 架构的 Linux 上，Windows x64 上，ARM 等上都有镜像。让我明确一点，我们说的是一个单一镜像标签支持多个平台和架构。我们马上就会看到它的实际应用。

为了实现这一点，注册表 API 支持两个重要的构造：

+   清单列表（新）

+   **清单**

“清单列表”就是它听起来的样子：一个特定镜像标签支持的架构列表。然后，每个支持的架构都有自己的**清单**，详细说明了它由哪些层组成。

图 6.9 以官方的`golang`镜像为例。左边是**清单列表**，列出了镜像支持的每种架构。箭头显示，**清单列表**中的每个条目指向一个包含镜像配置和层数据的**清单**。

![图 6.9](img/figure6-9.png)

图 6.9

让我们在实际操作之前先看看理论。

假设你正在树莓派上运行 Docker（在 ARM 架构上运行的 Linux）。当你拉取一个镜像时，你的 Docker 客户端会对运行在 Docker Hub 上的 Docker Registry API 进行相关调用。如果镜像存在**清单列表**，则会解析它以查看是否存在 ARM 架构的 Linux 的条目。如果存在 ARM 条目，则会检索该镜像的**清单**并解析出构成镜像的层的加密 ID。然后，每个层都会从 Docker Hub 的 blob 存储中拉取。

以下示例展示了拉取官方的`golang`镜像（支持多种架构）并运行一个简单命令来显示 Go 的版本以及主机的 CPU 架构。需要注意的是，这两个示例使用了完全相同的`docker container run`命令。我们不需要告诉 Docker 我们需要 Linux x64 或 Windows x64 版本的镜像。我们只需运行普通命令，让 Docker 负责获取适合我们正在运行的平台和架构的正确镜像！

Linux x64 示例：

```
$ docker container run --rm golang go version

Unable to find image `'golang:latest'` locally
latest: Pulling from library/golang
723254a2c089: Pull `complete`
<Snip>
39cd5f38ffb8: Pull `complete`
Digest: sha256:947826b5b6bc4...
Status: Downloaded newer image `for` golang:latest
go version go1.9.2 linux/amd64 
```

`Windows x64 示例：

```
PS> docker container run --rm golang go version

Using default tag: latest
latest: Pulling from library/golang
3889bb8d808b: Pull complete
8df8e568af76: Pull complete
9604659e3e8d: Pull complete
9f4a4a55f0a7: Pull complete
6d6da81fc3fd: Pull complete
72f53bd57f2f: Pull complete
6464e79d41fe: Pull complete
dca61726a3b4: Pull complete
9150276e2b90: Pull complete
cd47365a14fb: Pull complete
1783777af4bb: Pull complete
3b8d1834f1d7: Pull complete
7258d77b22dd: Pull complete
Digest: sha256:e2be086d86eeb789...e1b2195d6f40edc4
Status: Downloaded newer image for golang:latest
go version go1.9.2 windows/amd64 
```

`前面的操作从 Docker Hub 拉取`golang`图像，从中启动一个容器，执行`go version`命令，并输出主机系统的 Go 和 OS/CPU 架构的版本。每个示例的最后一行显示了每个`go version`命令的输出。请注意，这两个示例使用完全相同的命令，但 Linux 示例拉取了`linux/amd64`图像，而 Windows 示例拉取了`windows/amd64`图像。

在撰写本文时，所有*官方图像*都有清单列表。但是，支持所有架构是一个持续的过程。

创建在多个架构上运行的图像需要图像发布者额外的工作。此外，一些软件不是跨平台的。考虑到这一点，**清单列表**是可选的 - 如果图像不存在清单列表，注册表将返回正常的**清单**。

#### 删除图像

当您不再需要图像时，可以使用`docker image rm`命令从 Docker 主机中删除它。`rm`是删除的缩写。

删除图像将从 Docker 主机中删除图像及其所有层。这意味着它将不再显示在`docker image ls`命令中，并且包含层数据的 Docker 主机上的所有目录都将被删除。但是，如果一个图像层被多个图像共享，直到引用它的所有图像都被删除之前，该层将不会被删除。

使用`docker image rm`命令删除在上一步中拉取的图像。以下示例通过其 ID 删除图像，这可能与您的系统不同。

```
$ docker image rm 02674b9cb179
Untagged: alpine@sha256:c0537ff6a5218...c0a7726c88e2bb7584dc96
Deleted: sha256:02674b9cb179d57...31ba0abff0c2bf5ceca5bad72cd9
Deleted: sha256:e154057080f4063...2a0d13823bab1be5b86926c6f860 
```

`如果您要删除的图像正在运行的容器中使用，则无法删除它。在尝试再次删除操作之前，请停止并删除任何容器。

在 Docker 主机上**删除所有图像**的一个方便的快捷方式是运行`docker image rm`命令，并通过调用带有`-q`标志的`docker image ls`传递系统上所有图像 ID 的列表。如下所示。

如果您在 Windows 系统上执行以下命令，它只能在 PowerShell 终端中工作。它在 CMD 提示符上不起作用。

```
$ docker image rm `$(`docker image ls -q`)` -f 
```

`要了解这是如何工作的，请下载一些图像，然后运行`docker image ls -q`。

```
$ docker image pull alpine
Using default tag: latest
latest: Pulling from library/alpine
e110a4a17941: Pull `complete`
Digest: sha256:3dcdb92d7432d5...3626d99b889d0626de158f73a
Status: Downloaded newer image `for` alpine:latest

$ docker image pull ubuntu
Using default tag: latest
latest: Pulling from library/ubuntu
952132ac251a: Pull `complete`
82659f8f1b76: Pull `complete`
c19118ca682d: Pull `complete`
8296858250fe: Pull `complete`
24e0251a0e2c: Pull `complete`
Digest: sha256:f4691c96e6bba...128ae95a60369c506dd6e6f6ab
Status: Downloaded newer image `for` ubuntu:latest

$ docker image ls -q
bd3d4369aebc
4e38e38c8ce0 
```

`看看`docker image ls -q`如何返回一个只包含系统上本地拉取的所有图像的图像 ID 的列表。将此列表传递给`docker image rm`将删除系统上的所有图像，如下所示。

```
$ docker image rm `$(`docker image ls -q`)` -f
Untagged: ubuntu:latest
Untagged: ubuntu@sha256:f4691c9...2128ae95a60369c506dd6e6f6ab
Deleted: sha256:bd3d4369aebc494...fa2645f5699037d7d8c6b415a10
Deleted: sha256:cd10a3b73e247dd...c3a71fcf5b6c2bb28d4f2e5360b
Deleted: sha256:4d4de39110cd250...28bfe816393d0f2e0dae82c363a
Deleted: sha256:6a89826eba8d895...cb0d7dba1ef62409f037c6e608b
Deleted: sha256:33efada9158c32d...195aa12859239d35e7fe9566056
Deleted: sha256:c8a75145fcc4e1a...4129005e461a43875a094b93412
Untagged: alpine:latest
Untagged: alpine@sha256:3dcdb92...313626d99b889d0626de158f73a
Deleted: sha256:4e38e38c8ce0b8d...6225e13b0bfe8cfa2321aec4bba
Deleted: sha256:4fe15f8d0ae69e1...eeeeebb265cd2e328e15c6a869f

$ docker image ls
REPOSITORY     TAG    IMAGE ID    CREATED     SIZE 
```

`让我们提醒自己我们用来处理 Docker 图像的主要命令。`

### 镜像 - 命令

+   `docker image pull` 是下载镜像的命令。我们从远程仓库中的存储库中拉取镜像。默认情况下，镜像将从 Docker Hub 上的存储库中拉取。这个命令将从 Docker Hub 上的 `alpine` 存储库中拉取标记为 `latest` 的镜像 `docker image pull alpine:latest`。

+   `docker image ls` 列出了存储在 Docker 主机本地缓存中的所有镜像。要查看镜像的 SHA256 摘要，请添加 `--digests` 标志。

+   `docker image inspect` 是一件美妙的事情！它为你提供了镜像的所有细节 — 层数据和元数据。

+   `docker image rm` 是删除镜像的命令。这个命令展示了如何删除 `alpine:latest` 镜像 — `docker image rm alpine:latest`。你不能删除与正在运行（Up）或停止（Exited）状态的容器相关联的镜像。

### 章节总结

在本章中，我们学习了关于 Docker 镜像。我们了解到它们就像虚拟机模板，用于启动容器。在底层，它们由一个或多个只读层组成，当堆叠在一起时，构成了整个镜像。

我们使用了 `docker image pull` 命令将一些镜像拉取到我们的 Docker 主机本地注册表中。

我们涵盖了镜像命名、官方和非官方仓库、分层、共享和加密 ID。

我们看了一下 Docker 如何支持多架构和多平台镜像，最后看了一些用于处理镜像的最常用命令。

在下一章中，我们将对容器进行类似的介绍 — 镜像的运行时表亲。

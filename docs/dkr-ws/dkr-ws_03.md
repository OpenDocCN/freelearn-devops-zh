# 第三章：3. 管理您的 Docker 镜像

概述

在本章中，我们将深入研究 Docker 层，并分析缓存如何帮助加快镜像构建。我们还将深入研究 Docker 镜像，并设置 Docker 注册表，以增加镜像的可重用性。

通过本章的学习，您将能够演示 Docker 如何使用层构建镜像以及如何通过缓存加快镜像构建。您将使用镜像标签，并为 Docker 镜像设置标记策略。本章将使您能够为您的项目利用 Docker Hub，并区分公共和私有注册表。在处理项目时，它还将帮助您设置自己的 Docker 注册表。

# 介绍

我们之前的章节已经在 Docker 镜像上做了很多工作。正如您所看到的，我们已经能够获取 Docker Hub 中提供给公众的现有镜像，并在其基础上构建后运行或重用它们以满足我们的目的。镜像本身帮助我们简化流程，并减少我们需要做的工作。

在本章中，我们将更深入地了解镜像以及如何在系统上使用它们。我们将学习如何更好地组织和标记镜像，了解不同层的镜像如何工作，并设置公共和私有的注册表，以进一步重用我们创建的镜像。

Docker 镜像也非常适合应用程序开发。镜像本身是应用程序的自包含版本，其中包括运行所需的一切。这使开发人员能够在本地机器上构建镜像，并将其部署到开发或测试环境，以确保它与应用程序的其余部分良好配合。如果一切顺利，他们可以将相同的镜像作为发布推送到生产环境，供用户消费。当我们开始在更大的开发人员群体中工作时，我们需要在使用我们的镜像时保持一致。

本章还将帮助您制定一致的服务标记策略，以帮助限制问题，并确保在问题出现时能够追踪或回滚。了解如何分发镜像以供消费和协作也是我们将在本章进一步讨论的内容。因此，让我们立即开始本章的学习，了解 Docker 中的层和缓存是什么。

# Docker 层和缓存

注册表是存储和分发 Docker 镜像的一种方式。当您从注册表拉取 Docker 镜像时，您可能已经注意到镜像是分成多个部分而不是作为单个镜像拉取的。当您在系统上构建镜像时，同样的事情也会发生。

这是因为 Docker 镜像由多层组成。当您使用`Dockerfile`创建新镜像时，它会在您已构建的现有镜像之上创建更多的层。您在`Dockerfile`中指定的每个命令都将创建一个新的层，每个层都包含在执行命令之前和之后发生的所有文件系统更改。当您从`Dockerfile`运行镜像作为容器时，您正在在只读层的顶部创建可读写的层。这个可写层被称为**容器层**。

正如您将在接下来的练习中看到的那样，当您从`Dockerfile`构建容器时，所呈现的输出显示了在`Dockerfile`中运行的每个命令。它还显示了通过运行每个命令创建的层，这些层由随机生成的 ID 表示。一旦镜像构建完成，您就可以使用`docker history`命令查看在构建过程中创建的层，包括镜像名称或 ID。

注意

在设置构建环境并在开发过程中进一步进行时，请记住，层数越多，镜像就会越大。因此，这额外的存储空间在构建时间和开发和生产环境中使用的磁盘空间方面可能会很昂贵。

从`Dockerfile`构建镜像时，当使用`RUN`、`ADD`和`COPY`命令时会创建层。`Dockerfile`中的所有其他命令都会创建中间层。这些中间层的大小为 0 B；因此，它们不会增加 Docker 镜像的大小。

在构建我们的 Docker 镜像时，我们可以使用`docker history`命令和镜像名称或 ID 来查看用于创建镜像的层。输出将提供有关用于生成层的命令以及层的大小的详细信息：

```
docker history <image_name|image_id>
```

`docker image inspect`命令在提供有关我们镜像的层位于何处的进一步详细信息方面非常有用：

```
docker image inspect <image_id>
```

在本章的后面部分，当我们看创建基本图像时，我们将使用`docker image`命令，该命令与我们正在创建的图像的 TAR 文件版本一起使用。如果我们能够访问正在运行的容器或虚拟机，我们将能够将运行系统的副本放入 TAR 存档中。然后将存档的输出传输到`docker import`命令中，如此处所示：

```
cat <image_tar_file_name> | docker import - <new_image_name>
```

下一个练习将让您亲身体验到我们迄今为止学到的知识以及如何使用 Docker 镜像层进行工作。

注意

请使用`touch`命令创建文件，使用`vim`命令使用 vim 编辑器处理文件。

## 练习 3.01：使用 Docker 镜像层

在这个练习中，您将使用一些基本的`Dockerfiles`来看看 Docker 如何使用层来构建图像。您将首先创建一个`Dockerfile`并构建一个新的图像。然后重新构建图像以查看使用缓存的优势以及由于使用缓存而减少的构建时间：

1.  使用您喜欢的文本编辑器创建一个名为`Dockerfile`的新文件，并添加以下细节：

```
FROM alpine
RUN apk update
RUN apk add wget
```

1.  保存`Dockerfile`，然后从命令行确保您在与您创建的`Dockerfile`相同的目录中。使用`docker build`命令使用`-t`选项为其命名为`basic-app`来创建新的镜像：

```
docker build -t basic-app .
```

如果图像构建成功，您应该会看到类似以下的输出。我们已经用粗体突出显示了每个构建步骤。每个步骤都作为中间层构建，如果成功完成，然后将其转移到只读层：

```
Sending build context to Docker daemon 4.096kB
Step 1/3 : FROM alpine
latest: Pulling from library/alpine
9d48c3bd43c5: Pull complete 
Digest: sha256:72c42ed48c3a2db31b7dafe17d275b634664a
        708d901ec9fd57b1529280f01fb
Status: Downloaded newer image for alpine:latest
  ---> 961769676411
Step 2/3 : RUN apk update
  ---> Running in 4bf85f0c3676
fetch http://dl-cdn.alpinelinux.org/alpine/v3.10/main/
  x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.10/community/
  x86_64/APKINDEX.tar.gz
v3.10.2-64-g631934be3a [http://dl-cdn.alpinelinux.org/alpine
  /v3.10/main]
v3.10.2-65-ge877e766a2 [http://dl-cdn.alpinelinux.org/alpine
  /v3.10/community]
OK: 10336 distinct packages available
Removing intermediate container 4bf85f0c3676
  ---> bcecd2429ac0
Step 3/3 : RUN apk add wget
  ---> Running in ce2a61d90f77
(1/1) Installing wget (1.20.3-r0)
Executing busybox-1.30.1-r2.trigger
OK: 6 MiB in 15 packages
Removing intermediate container ce2a61d90f77
  ---> a6d7e99283d9
Successfully built 0e86ae52098d
Successfully tagged basic-app:latest
```

1.  使用`docker history`命令以及`basic-app`的图像名称来查看图像的不同层：

```
docker history basic-app
```

历史记录提供了创建细节，包括每个层的大小：

```
IMAGE         CREATED            CREATED BY 
                      SIZE
a6d7e99283d9  About a minute ago /bin/sh -c apk add wget
                      476kB
bcecd2429ac0  About a minute ago /bin/sh -c apk update
                      1.4MB
961769676411  5 weeks ago        /bin/sh -c #(nop)
CMD ["/bin/sh"]       0B
<missing>     5 weeks ago        /bin/sh -c #(nop) 
ADD file:fe6407fb…    5.6MB
```

注意

`docker history`命令显示了作为`Dockerfile` `FROM`命令的一部分使用的原始图像的层为`<missing>`。在我们的输出中显示为`missing`，因为它是在不同的系统上创建的，然后被拉到您的系统上。

1.  不做任何更改再次运行构建：

```
docker build -t basic-app .
```

这将显示构建是使用 Docker 镜像缓存中存储的层完成的，从而加快了我们的构建速度。尽管这只是一个小图像，但更大的图像将显示显着的增加：

```
Sending build context to Docker daemon  4.096kB
Step 1/3 : FROM alpine
  ---> 961769676411
Step 2/3 : RUN apk update
  ---> Using cache
  ---> bcecd2429ac0
Step 3/3 : RUN apk add wget
  ---> Using cache
  ---> a6d7e99283d9
Successfully built a6d7e99283d9
Successfully tagged basic-app:latest
```

1.  假设您忘记在图像创建的过程中安装`curl`包。在*步骤 1*中的`Dockerfile`中添加以下行：

```
FROM alpine
RUN apk update
RUN apk add wget curl
```

1.  再次构建图像，现在您将看到图像由缓存层和需要创建的新层混合而成：

```
docker build -t basic-app .
```

突出显示了输出的第三步，显示了我们在`Dockerfile`中所做的更改：

```
Sending build context to Docker daemon 4.096kB
Step 1/3 : FROM alpine
  ---> 961769676411
Step 2/3 : RUN apk update
  ---> Using cache
  ---> cb8098d0c33d
Step 3/3 : RUN apk add wget curl
  ---> Running in b041735ff408
(1/5) Installing ca-certificates (20190108-r0)
(2/5) Installing nghttp2-libs (1.39.2-r0)
(3/5) Installing libcurl (7.66.0-r0)
(4/5) Installing curl (7.66.0-r0)
(5/5) Installing wget (1.20.3-r0)
Executing busybox-1.30.1-r2.trigger
Executing ca-certificates-20190108-r0.trigger
OK: 8 MiB in 19 packages
Removing intermediate container b041735ff408
  ---> c7918f4f95b9
Successfully built c7918f4f95b9
Successfully tagged basic-app:latest
```

1.  再次运行`docker images`命令：

```
docker images
```

您现在会注意到图像被命名和标记为`<none>`，以显示我们现在创建了一个悬空图像：

```
REPOSITORY   TAG      IMAGE ID        CREATED           SIZE
basic-app    latest   c7918f4f95b9    25 seconds ago    8.8MB
<none>       <none>   0e86ae52098d    2 minutes ago     7.48MB
Alpine       latest   961769676411    5 weeks ago       5.58MB
```

注意

悬空图像，在我们的图像列表中表示为`<none>`，是由于一个层与我们系统上的任何图像都没有关联而引起的。这些悬空图像不再起作用，并将占用您系统上的磁盘空间。我们的示例悬空图像只有 7.48 MB，这很小，但随着时间的推移，这可能会累积起来。

1.  使用图像 ID 运行`docker image inspect`命令，查看悬空图像在我们系统上的位置：

```
docker image inspect 0e86ae52098d
```

以下输出已从实际输出减少，仅显示图像的目录：

```
... 
  "Data": {
    "LowerDir": "/var/lib/docker/overlay2/
      41230f31bb6e89b6c3d619cafc309ff3d4ca169f9576fb003cd60fd4ff
      4c2f1f/diff:/var/lib/docker/overlay2/
      b8b90262d0a039db8d63c003d96347efcfcf57117081730b17585e163f
      04518a/diff",
    "MergedDir": "/var/lib/docker/overlay2/
      c7ea9cb56c5bf515a1b329ca9fcb2614f4b7f1caff30624e9f6a219049
      32f585/
      merged",
    "UpperDir": "/var/lib/docker/overlay2/
      c7ea9cb56c5bf515a1b329ca9fcb2614f4b7f1caff30624e9f6a21904
      932f585/diff",
    "WorkDir": "/var/lib/docker/overlay2/
      c7ea9cb56c5bf515a1b329ca9fcb2614f4b7f1caff30624e9f6a21904
      932f585/work"
  },
...
```

我们所有的图像都位于与悬空图像相同的位置。由于它们共享相同的目录，任何悬空图像都会浪费我们系统上的空间。

1.  从命令行运行`du`命令，查看我们的图像使用的总磁盘空间：

```
du -sh /var/lib/docker/overlay2/
```

该命令将返回您的图像使用的总磁盘空间

```
11M    /var/lib/docker/overlay2/
```

注意

如果您正在使用 Docker Desktop，可能是在 Mac 上，您会注意到您无法看到图像，因为 Docker 在您的系统上以虚拟图像运行，即使`docker image inspect`命令显示的位置与上面相同。

1.  再次使用`docker images`命令，并使用`-a`选项：

```
docker images -a
```

它还会显示在构建我们的图像时使用的中间层：

```
REPOSITORY   TAG      IMAGE ID      CREATED          SIZE
basic-app    latest   c7918f4f95b9  25 seconds ago   8.8MB
<none>       <none>   0e86ae52098d  2 minutes ago    7.48MB
<none>       <none>   112a4b041305  11 minutes ago   7MB
Alpine       latest   961769676411  5 weeks ago      5.58MB
```

1.  运行`docker image prune`命令以删除所有悬空图像。您可以使用`docker rmi`命令逐个删除所有悬空图像，使用图像 ID，但`docker image prune`命令是更简单的方法：

```
docker image prune
```

您应该会得到以下输出：

```
WARNING! This will remove all dangling images.
Are you sure you want to continue? [y/N] y
Deleted Images:
deleted: sha256:0dae3460f751d16f41954e0672b0c41295d46ee99d71
         d63e7c0c8521bd9e6493
deleted: sha256:d74fa92b37b74820ccccea601de61d45ccb3770255b9
         c7dd22edf16caabafc1c
Total reclaimed space: 476.4kB
```

1.  再次运行`docker images`命令：

```
docker images
```

您会看到我们的图像列表中不再有悬空图像：

```
REPOSITORY   TAG      IMAGE ID        CREATED           SIZE
basic-app    latest   c7918f4f95b9    25 seconds ago    8.8MB
Alpine       latest   961769676411    5 weeks ago       5.58MB
```

1.  再次在图像目录上运行`du`命令：

```
du -sh /var/lib/docker/overlay2/
```

您还应该观察到尺寸的小幅减小：

```
10M    /var/lib/docker/overlay2/
```

这个练习只显示了较小的图像尺寸，但在运行生产和开发环境时，这绝对是需要牢记的事情。本章的这一部分为您提供了 Docker 如何在其构建过程中使用层和缓存的基础。

对于我们的下一个练习，我们将进一步研究我们的层和缓存，以查看它们如何用于加快图像构建过程。

## 练习 3.02：增加构建速度和减少层

到目前为止，您一直在处理较小的构建。但是，随着您的应用程序在大小和功能上的增加，您将开始考虑您正在创建的 Docker 图像的大小和层数以及创建它们的速度。本练习的目标是加快构建时间并减小图像的大小，并在构建 Docker 图像时使用`--cache-from`选项：

1.  创建一个新的`Dockerfile`来演示您将要进行的更改，但首先，请清理系统上的所有图像。使用`docker rmi`命令并带有`-f`选项来强制进行任何需要的删除，括号中的命令将提供系统上所有图像 ID 的列表。使用`-a`选项来显示所有正在运行和停止的容器，使用`-q`选项仅显示容器图像哈希值，而不显示其他内容。

```
docker rmi -f $(docker images -a -q)
```

该命令应返回以下输出：

```
Untagged: hello-world:latest
...
deleted: sha256:d74fa92b37b74820ccccea601de61d45ccb3770255
         b9c7dd22edf16caabafc1c
```

可以观察到`hello-world:latest`镜像已被取消标记，并且具有 ID`sha256:d74fa92b37b74820ccccea601 de61d45ccb3770255b9c7dd22edf16caabafc1c`的镜像已被删除。

注意

请注意，我们可以使用`rmi`和`prune`命令来删除图像。在这里，我们使用了`rmi`命令，因为`prune`直到最近才可用。

1.  将以下代码添加到您的`Dockerfile`（您在*练习 3.01*中创建的）。它将模拟一个简单的 Web 服务器，并在构建过程中打印我们的`Dockerfile`的输出：

```
1 FROM alpine
2 
3 RUN apk update
4 RUN apk add wget curl
5
6 RUN wget -O test.txt https://github.com/PacktWorkshops/   The-Docker-Workshop/blob/master/Chapter03/Exercise3.02/100MB.bin
7
8 CMD mkdir /var/www/
9 CMD mkdir /var/www/html/
10
11 WORKDIR /var/www/html/
12
13 COPY Dockerfile.tar.gz /tmp/
14 RUN tar -zxvf /tmp/Dockerfile.tar.gz -C /var/www/html/
15 RUN rm /tmp/Dockerfile.tar.gz
16
17 RUN cat Dockerfile
```

您会注意到`Dockerfile`的*第 6 行*正在执行一个相当琐碎的任务（下载一个名为`100MB.bin`的 100MB 文件），这在`Dockerfile`中通常不会执行。我们已经添加了它来代表一个构建任务或类似的东西，例如，下载内容或从文件构建软件。

1.  使用`docker pull`命令下载基本图像，以便您可以从每次测试开始使用相同的图像：

```
docker pull alpine
```

1.  创建一个 TAR 文件，以便按照我们在`Dockerfile`的*第 13 行*中指示的方式添加到我们的图像中：

```
tar zcvf Dockerfile.tar.gz Dockerfile
```

1.  使用与`basic-app`相同的名称构建一个新图像。您将在代码开头使用`time`命令，以便我们可以衡量构建图像所花费的时间：

```
time docker build -t basic-app .
```

输出将返回构建图像所花费的时间：

```
...
real 4m36.810s
user 0m0.354s
sys 0m0.286s
```

1.  对新的`basic-app`镜像运行`docker history`命令：

```
docker history basic-app
```

与上一个练习相比，我们的`Dockerfile`中有一些额外的命令。因此，我们将在新镜像中看到 12 层，这并不奇怪：

```
IMAGE         CREATED      CREATED BY                           SIZE
5b2e3b253899 2 minutes ago /bin/sh -c cat Dockerfile            0B
c4895671a177 2 minutes ago /bin/sh -c rm /tmp/Dockerfile.tar.gz 0B
aaf18a11ba25 2 minutes ago /bin/sh -c tar -zxvf /tmp/Dockfil…   283B
507161de132c 2 minutes ago /bin/sh -c #(nop) COPY file:e39f2a0… 283B
856689ad2bb6 2 minutes ago /bin/sh -c #(nop) WORKDIR /var/…     0B
206675d145d4 2 minutes ago /bin/sh -c #(nop)  CMD ["/bin/sh"…   0B
c947946a36b2 2 minutes ago /bin/sh -c #(nop)  CMD ["/bin/sh"…   0B
32b0abdaa0a9 2 minutes ago /bin/sh -c curl https://github.com…  105MB
e261358addb2 2 minutes ago /bin/sh -c apk add wget curl         1.8MB
b6f77a768f90 2 minutes ago /bin/sh -c apk update                1.4MB
961769676411 6 weeks ago   /bin/sh -c #(nop)  CMD ["/bin/sh"]   0B
<missing>    6 weeks ago   /bin/sh -c #(nop) ADD file:fe3dc…    5.6MB
```

我们可以看到`Dockerfile`中的`RUN`、`COPY`和`ADD`命令正在创建特定大小的层，与运行的命令或添加的文件相关，并且`Dockerfile`中的所有其他命令的大小都为 0 B。

1.  通过合并`Dockerfile`中*第 3 行和第 4 行*的`RUN`命令以及合并*第 8 行和第 9 行*的`CMD`命令，减少镜像中的层数。通过这些更改，我们的`Dockerfile`现在应该如下所示：

```
1 FROM alpine
2 
3 RUN apk update && apk add wget curl
4 
5 RUN wget -O test.txt https://github.com/PacktWorkshops/    The-Docker-Workshop/blob/master/Chapter03/Exercise3.02/100MB.bin
6 
7 CMD mkdir -p /var/www/html/
8 
9 WORKDIR /var/www/html/
10 
11 COPY Dockerfile.tar.gz /tmp/
12 RUN tar -zxvf /tmp/Dockerfile.tar.gz -C /var/www/html/
13 RUN rm /tmp/Dockerfile.tar.gz
14 
15 RUN cat Dockerfile
```

再次运行`docker build`将会减少新镜像的层数，从 12 层减少到 9 层，因为即使运行的命令数量相同，它们在*第 3 行*和*第 7 行*中被链接在一起。

1.  *第 11 行*、*第 12 行*和*第 13 行*的`Dockerfile`正在使用`COPY`和`RUN`命令来`copy`和`unzip`我们的归档文件，然后删除原始的解压文件。用`ADD`命令替换这些行，而无需运行解压和删除`.tar`文件的行：

```
1 FROM alpine
2 
3 RUN apk update && apk add wget curl
4
5 RUN wget -O test.txt https://github.com/PacktWorkshops/    The-Docker-Workshop/blob/master/Chapter03/Exercise3.02/100MB.bin
6 
7 CMD mkdir -p /var/www/html/
8 
9 WORKDIR /var/www/html/
10 
11 ADD Dockerfile.tar.gz /var/www/html/
12 RUN cat Dockerfile
```

1.  再次构建镜像，将新镜像的层数从 9 层减少到 8 层。如果您一直在观察构建过程，您可能会注意到大部分时间是在`Dockerfile`的*第 3 行*和*第 5 行*中运行，我们在那里运行`apk update`，然后安装`wget`和`curl`，然后从网站获取内容。这样做一两次不会有问题，但如果我们创建了基础镜像，`Dockerfile`可以在其上运行，您将能够完全从`Dockerfile`中删除这些行。

1.  进入一个新目录，并创建一个新的`Dockerfile`，它将只拉取基础镜像并运行`apk`命令，如下所示：

```
1 FROM alpine
2
3 RUN apk update && apk add wget curl
4
5 RUN wget -O test.txt https://github.com/PacktWorkshops/    The-Docker-Workshop/blob/master/Chapter03/Exercise3.02/100MB.bin
```

1.  从上一个`Dockerfile`中构建新的基础镜像，并将其命名为`basic-base`：

```
docker build -t basic-base .
```

1.  从原始`Dockerfile`中删除*第 3 行*，因为它将不再需要。进入项目目录，并将`FROM`命令中使用的镜像更新为`basic-base`，并删除*第 3 行*中的`apk`命令。我们的`Dockerfile`现在应该如下所示：

```
1 FROM basic-base
2
3 CMD mkdir -p /var/www/html/
4
5 WORKDIR /var/www/html/
6
7 ADD Dockerfile.tar.gz /var/www/html/
8 RUN cat Dockerfile
```

1.  再次运行新的`Dockerfile`进行构建。再次使用`time`命令进行构建，我们现在可以看到构建在 1 秒多钟内完成：

```
time docker build -t basic-app .
```

如果您一直在观看构建过程，您会注意到与我们以前的构建相比，它运行得更快：

```
...
real 0m1.810s
user 0m0.117s
sys  0m0.070s
```

注意

您将观察到镜像的层将保持不变，因为我们正在在我们的系统上构建基础镜像，该系统执行`apk`命令。即使我们没有减少层数，这仍然是一个很好的结果，可以加快构建速度。

1.  我们可以使用之前使用的`basic-base`镜像的不同方式。使用`docker build`命令和`--cache-from`选项指定构建镜像时将使用的缓存层。设置`FROM`命令仍然使用`alpine`镜像，并使用后面的`--cache-from`选项，以确保用于构建`basic-base`的层被用于我们当前的镜像：

```
docker build --cache-from basic-base -t basic-app .
```

在完成此练习之前，我们还有一些任务要完成。在接下来的步骤中，我们将查看提交对镜像的更改，以查看它如何影响我们的层。这不是我们经常使用的东西，但有时我们需要将生产数据复制到开发或测试环境中，其中一种方法是使用带有`commit`命令的 Docker 镜像，该命令将更改我们运行容器的顶部可写层。

1.  以交互式 shell 模式运行`basic-app`以创建一些生产数据。为此，请使用`-it`选项运行以下`docker run`命令以交互模式运行，并使用`sh` shell 访问运行的容器：

```
docker run -it basic-app sh
/var/www/html #
```

1.  使用 vi 文本编辑器创建一个名为`prod_test_data.txt`的新文本文件：

```
vi prod_test_data.txt
```

1.  添加以下文本行作为一些测试数据。文本中的数据并不重要；这只是一个示例，表明我们可以将这些更改复制到另一个镜像中：

1.  这是一个示例生产数据。退出运行的容器，然后使用带有`-a`选项的`docker ps`命令检查容器 ID：

```
docker ps -a
```

您将获得以下输出：

```
CONTAINER ID    IMAGE        COMMAND    CREATED
ede3d51bba9e    basic-app    "sh"       4 minutes ago
```

1.  运行`docker commit`命令，使用容器 ID 创建一个包含所有这些更改的新镜像。确保添加新镜像的名称。在本例中，使用`basic-app-test`：

```
docker commit ede3d51bba9e basic-app-test
```

您将获得以下输出：

```
sha256:0717c29d29f877a7dafd6cb0555ff6131179b457
       e8b8c25d9d13c2a08aa1e3f4
```

1.  在新创建的镜像上运行`docker history`命令：

```
docker history basic-app-test
```

现在，这应该显示出我们添加了示例生产数据的额外层，显示在我们的输出中，大小为 72B：

```
IMAGE        CREATED       CREATED BY                         SIZE
0717c29d29f8 2 minutes ago sh                                 72B
302e01f9ba6a 2 minutes ago /bin/sh -c cat Dockerfile          0B
10b405ceda34 2 minutes ago /bin/sh -c #(nop) ADD file:e39f…   283B
397f533f4019 2 minutes ago /bin/sh -c #(nop) WORKDIR /var/…   0B
c8782986b276 2 minutes ago /bin/sh -c #(nop)  CMD ["/bin/sh"… 0B
6dee05f36f95 2 minutes ago /bin/sh -c apk update && apk ad    3.2MB
961769676411 6 weeks ago   /bin/sh -c #(nop)  CMD ["/bin/sh"] 0B
<missing>    6 weeks ago   /bin/sh -c #(nop) ADD file:fe3dc…  5.6MB
```

1.  现在，运行新创建的`basic-app-test`镜像和`cat`，我们添加的新文件：

```
docker run basic-app-test cat prod_test_data.txt
```

这应该显示我们添加的输出，表明我们可以在需要时重用现有镜像：

```
This is a sample production piece of data
```

注意

在撰写本文时，`docker build`命令还允许使用`--squash`选项的新实验性功能。该选项尝试在构建时将所有层合并为一层。我们还没有涵盖这个功能，因为它仍处于实验阶段。

这个练习演示了构建缓存和镜像层是如何改善构建时间的。到目前为止，我们所有的构建都是使用从 Docker Hub 下载的镜像开始的，但如果你希望进一步控制，也可以使用自己创建的镜像。下一节将帮助你创建自己的基础 Docker 镜像。

# 创建基础 Docker 镜像

创建自己的基础 Docker 镜像实际上很简单。就像我们之前使用`docker commit`命令从运行的容器创建镜像一样，我们也可以从最初运行我们应用程序的系统或服务器创建镜像。我们需要记住，创建基础镜像仍然需要保持小巧和轻量级。这不仅仅是将现有应用程序从现有服务器迁移到 Docker 的问题。

我们可以使用我们正在专门工作的系统，但如果你正在使用生产服务器，镜像实际上可能会很大。如果你有一个小型虚拟机，认为它非常适合作为基础镜像，可以使用以下步骤创建基础镜像。类似于`docker commit`命令，这可以用于任何你可以访问的系统。

## 练习 3.03：创建自己的基础 Docker 镜像

以下练习将使用我们当前正在运行的`basic-app`镜像，并展示创建基础镜像有多么简单。对于更大、更复杂的环境，也可以使用相同的步骤：

1.  执行`docker run`命令以同时运行容器并登录：

```
docker run -it basic-app sh
```

1.  运行`tar`命令在运行的容器上创建系统的备份。为了限制新镜像中的信息，排除`.proc`、`.tmp`、`.mnt`、`.dev`和`.sys`目录，并将所有内容创建在`basebackup.tar.gz`文件下：

```
tar -czf basebackup.tar.gz --exclude=backup.tar.gz --exclude=proc --exclude=tmp --exclude=mnt --exclude=dev --exclude=sys /
```

1.  为确保`basebackup.tar.gz`文件中有数据，请运行`du`命令，确保其大小足够大：

```
du -sh basebackup.tar.gz 
```

输出返回`basebackup.tar.gz`文件的大小：

```
4.8M	basebackup.tar.gz
```

1.  运行`docker ps`命令找到当前保存新备份文件的容器 ID，`.tar`文件：

```
docker ps
```

该命令将返回镜像的容器 ID：

```
CONTAINER ID        IMAGE        COMMAND      CREATED
6da7a8c1371a        basic-app    "sh"         About a minute ago
```

1.  将`.tar`文件复制到您的开发系统上，使用`docker cp`命令，使用正在运行的容器的容器 ID 以及要复制的位置和文件。以下命令将使用您的容器 ID 执行此操作，并将其移动到您的`/tmp`目录中：

```
docker cp 6da7a8c1371a:/var/www/html/basebackup.tar.gz /tmp/
```

1.  使用`docker import`命令创建一个新的镜像。只需将`basebackup.tar.gz`文件的输出导入`docker import`命令中，并在此过程中命名新镜像。在我们的示例中，将其命名为`mynew-base`：

```
cat /tmp/basebackup.tar.gz | docker import - mynew-base
```

1.  使用您的新镜像的名称运行`docker images`命令，以验证它是否已在上一步中创建：

```
docker images mynew-base
```

您应该得到以下类似的输出：

```
REPOSITORY    TAG     IMAGE ID      CREATED         SIZE
mynew-base    latest  487e14fca064  11 seconds ago  8.79MB
```

1.  运行`docker history`命令：

```
docker history mynew-base
```

您将看到我们的新镜像中只有一个层：

```
IMAGE         CREATED         CREATED BY   SIZE   COMMENT
487e14fca064  37 seconds ago               .79MB  Imported from –
```

1.  要测试新镜像，请在新镜像上运行`docker run`命令，并列出您的`/var/www/html/`目录中的文件：

```
docker run mynew-base ls -l /var/www/html/
```

该命令应返回类似的输出：

```
total 4
-rw-r--r--    1 501      dialout      283 Oct  3 04:07 Dockerfile
```

可以看到镜像已成功创建，并且`/var/www/html/`目录中有 24 个文件。

这个练习向您展示了如何从运行的系统或环境中创建一个基本镜像，但如果您想要创建一个小的基本镜像，那么下一节将向您展示如何使用**scratch**镜像。

## 空白镜像

空白镜像是 Docker 专门为构建最小化镜像而创建的镜像。如果您有一个二进制应用程序，比如 Java、C++等编写并编译的应用程序，可以独立运行而无需任何支持应用程序，那么空白镜像将帮助您使用您可以创建的最小镜像之一来运行该镜像。

当我们在我们的`Dockerfile`中使用`FROM scratch`命令时，我们指定将使用 Docker 保留的最小镜像，该镜像命名为`scratch`来构建我们的新容器镜像。

## 练习 3.04：使用空白镜像

在这个练习中，您将创建一个小的 C 应用程序在镜像上运行。您实际上不需要了解 C 语言的任何内容来完成这个练习。您创建的应用程序将安装在您的空白基本镜像上，以确保该镜像尽可能小。您创建的应用程序将向您展示如何创建一个最小的基本镜像之一：

1.  使用`docker pull`命令拉取空白镜像：

```
docker pull scratch
```

您会注意到无法拉取该镜像，并将收到错误：

```
Using default tag: latest
Error response from daemon: 'scratch' is a reserved name
```

1.  创建一个 C 程序，将其构建到我们的`Dockerfile`中使用的镜像中。创建一个名为`test.c`的程序文件：

```
touch test.c
```

1.  打开文件并添加以下代码，它将在控制台上简单地从 1 数到 10：

```
#include <stdio.h>
int main()
{
    int i;
    for (i=1; i<=10; i++)
    {
        printf("%d\n", i);
    }
    return 0;
}
```

1.  通过运行以下命令构建 C 程序来从命令行构建镜像：

```
g++ -o test -static test.c
```

注意

如果你想在构建镜像之前测试它，可以在命令行上运行`./test`来进行测试。

1.  创建`Dockerfile`。`Dockerfile`将非常简洁，但需要以`FROM scratch`开头。文件的其余部分将把 C 程序添加到你的镜像，然后在*第 4 行*运行它：

```
1 FROM scratch
2
3 ADD test /
4 CMD ["/test"]
```

1.  构建一个新的镜像。在这种情况下，使用以下命令将镜像命名为`scratchtest`：

```
docker build -t scratchtest .
```

1.  从命令行运行镜像：

```
docker run scratchtest
```

你将看到你在这个练习中创建和编译的测试 C 文件的输出：

```
1
2
3
4
5
6
7
8
9
10
```

1.  运行`docker images`命令查看你的新镜像：

```
docker images scratchtest
```

这将向你展示一些令人印象深刻的结果，因为你的镜像只有`913 kB`大小。

```
REPOSITORY   TAG     IMAGE ID         CREATED          SIZE
scratch      latest  221adbe23c26     20 minutes ago   913kB
```

1.  使用`docker history`命令查看镜像的层：

```
docker history scratchtest
```

你将看到类似以下的输出，它只有两层，一层是从头开始的原始层，另一层是我们`ADD`测试 C 程序的层：

```
IMAGE        CREATED        CREATED BY                        SIZE
221adbe23c26 23 minutes ago /bin/sh -c #(nop)  CMD ["/test"]  0B
09b61a3a1043 23 minutes ago /bin/sh -c #(nop) ADD file:80933… 913kB
```

在这个练习中创建的 scratch 镜像在一定程度上创建了一个既功能齐全又最小化的镜像，并且还表明，如果你考虑一下你想要实现什么，就可以轻松加快构建速度并减小镜像的大小。

我们现在将暂停构建镜像的工作，更仔细地研究如何命名和标记我们的 Docker 镜像。

# Docker 镜像命名和标记

我们已经提到了标签，但随着我们更密切地与 Docker 镜像一起工作，现在可能是深入了解镜像标签的好时机。简单来说，标签是 Docker 镜像上的标签，应该为使用该镜像的用户提供一些有用的信息，关于镜像或镜像版本。

到目前为止，我们一直在像独立开发者一样处理我们的镜像，但当我们开始与更大的开发团队合作时，就需要更加努力地考虑如何命名和标记我们的镜像。本章的下一部分将为你的先前工作增添内容，并让你开始为你的项目和工作制定命名和标记策略。

有两种主要方法来命名和标记您的 Docker 图像。您可以使用`docker tag`命令，也可以在从`Dockerfile`构建图像时使用`-t`选项。要使用`docker tag`命令，您需要指定要使用的源存储库名称作为基础和要创建的目标名称和标记：

```
docker tag <source_repository_name>:<tag> <target_repository_name>:tag
```

当您使用`docker build`命令命名图像时，使用的`Dockerfile`将创建您的源，然后使用`-t`选项来命名和标记您的图像如下：

```
docker build -t <target_repository_name>:tag Dockerfile
```

存储库名称有时可以以主机名为前缀，但这是可选的，并且将用于让 Docker 知道存储库的位置。我们将在本章后面演示这一点，当我们创建自己的 Docker 注册表时。如果您要将图像推送到 Docker Hub，还需要使用您的 Docker Hub 用户名作为存储库名称的前缀，就像这样：

```
docker build -t <dockerhub_user>/<target_repository_name>:tag Dockerfile
```

在图像名称中使用两个以上的前缀仅在本地图像注册表中受支持，并且通常不使用。下一个练习将指导您完成标记 Docker 图像的过程。

## 练习 3.05：给 Docker 图像打标签

在接下来的练习中，您将使用不同的图像，使用轻量级的`busybox`图像来演示标记的过程，并开始在项目中实施标记。BusyBox 用于将许多常见的 UNIX 实用程序的微小版本组合成一个小的可执行文件：

1.  运行`docker rmi`命令来清理您当前系统上的图像，这样您就不会因为大量的图像而感到困惑：

```
docker rmi -f $(docker images -a -q)
```

1.  在命令行上，运行`docker pull`命令以下载最新的`busybox`容器：

```
docker pull busybox
```

1.  运行`docker images`命令：

```
docker images
```

这将为我们提供开始组合一些标签命令所需的信息：

```
REPOSITORY    TAG       IMAGE ID        CREATED      SIZE
Busybox       latest    19485c79a9bb    2 weeks ago  1.22MB
```

1.  使用`tag`命令对图像进行命名和标记。您可以使用图像 ID 或存储库名称来标记图像。首先使用图像 ID，但请注意在您的系统上，您将有一个不同的图像 ID。将存储库命名为`new_busybox`，并包括标签`ver_1`：

```
docker tag 19485c79a9bb new_busybox:ver_1
```

1.  使用存储库名称和图像标签。使用您的名称创建一个新的存储库，并使用`ver_1.1`的新版本如下：

```
docker tag new_busybox:ver_1 vince/busybox:ver_1.1
```

注意

在这个例子中，我们使用了作者的名字（`vince`）。

1.  运行`docker images`命令：

```
docker images
```

您应该看到类似于以下内容的输出。当然，您的图像 ID 将是不同的，但存储库名称和标签应该是相似的：

```
REPOSITORY     TAG      ID             CREATED        SIZE
Busybox        latest   19485c79a9bb   2 weeks ago    1.22MB
new_busybox    ver_1    19485c79a9bb   2 weeks ago    1.22MB
vince/busybox  ver_1.1  19485c79a9bb   2 weeks ago    1.22MB
```

1.  使用`Dockerfile`和`docker build`命令的`-t`选项来创建一个基本图像，并为其命名和打上标签。在本章中，你已经做过几次了，所以从命令行中运行以下命令来创建一个基本的`Dockerfile`，使用你之前命名的`new_busybox`图像。还要包括图像名称的标签，因为 Docker 将尝试使用`latest`标签，但由于它不存在，所以会失败。

```
echo "FROM new_busybox:ver_1" > Dockerfile
```

1.  运行`docker build`命令来创建图像，并同时为其命名和打上标签：

```
docker build -t built_image:ver_1.1.1 .
```

1.  运行`docker images`命令：

```
docker images
```

你现在应该在你的系统上有四个可用的图像。它们都有相同的容器 ID，但会有不同的仓库名称和标记版本。

```
REPOSITORY     TAG        ID            CREATED      SIZE
built_image    ver_1.1.1  19485c79a9bb  2 weeks ago  1.22MB
Busybox        latest     19485c79a9bb  2 weeks ago  1.22MB
new_busybox    ver_1      19485c79a9bb  2 weeks ago  1.22MB
vince/busybox  ver_1.1    19485c79a9bb  2 weeks ago  1.22MB
```

给图像打上一个与你的组织或团队相关的适当版本的标签并不需要太多时间，尤其是经过一点练习。本章的这一部分向你展示了如何给你的图像打上标签，这样它们就不再带有`latest`的默认标签了。你将在下一节中看到，使用`latest`标签并希望它能正常工作实际上可能会给你带来一些额外的问题。

# 在 Docker 中使用 latest 标签

在我们使用标签的过程中，我们已经多次提到不要使用`latest`标签，这是 Docker 提供的默认标签。正如你很快就会看到的，使用`latest`标签可能会导致很多问题，特别是在部署图像到生产环境时。

我们首先需要意识到的是，`latest`只是一个标签，就像我们在之前的例子中使用`ver_1`一样。它绝对不意味着我们的代码的最新版本。它只是表示我们的图像的最新构建，没有包括标签。

在大型团队中使用`latest`也会导致很多问题，每天多次部署到环境中。这也意味着你将没有历史记录，这会使得回滚错误更加困难。因此，请记住，每次构建或拉取图像时，如果你没有指定标签，Docker 将使用`latest`标签，并不会做任何事情来确保图像是最新版本。在下一个练习中，我们将检查使用`latest`标签可能会导致什么问题。

## 练习 3.06：使用 latest 时出现的问题

您可能仍然是使用 Docker 和标签的新手，因此您可能尚未遇到使用`latest`标签时出现任何问题。这个练习将为您提供一些明确的想法，说明使用`latest`标签可能会导致您的开发过程出现问题，并为您提供应避免使用它的原因。在上一个练习中，您使用了`new_busybox:ver_1`映像创建了一个简单的`Dockerfile`。在这个练习中，您将进一步扩展此文件：

1.  打开`Dockerfile`并修改文件，使其看起来像以下文件。这是一个简单的脚本，将创建带有简单代码的`version.sh`脚本，以输出我们服务的最新版本。新文件将被命名为`Dockerfile_ver1`。

```
1 FROM new_busybox:ver_1
2
3 RUN echo "#!/bin/sh\n" > /version.sh
4 RUN echo "echo \"This is Version 1 of our service\""   >> /version.sh
5
6 ENTRYPOINT ["sh", "/version.sh"]
```

1.  构建映像并以您的姓名命名，并显示该映像只是一个测试：

```
docker build -t vince/test .
```

注意

我们在这里使用了`vince`作为名称，但您可以使用任何理想的名称。

1.  使用`docker run`命令运行映像：

```
docker run vince/test
```

现在应该看到`versions.sh`脚本的输出：

```
This is Version 1 of our service
```

1.  使用`docker tag`命令将此映像标记为`version1`：

```
docker tag vince/test vince/test:version1
```

1.  打开`Dockerfile`并对*第 4 行*进行以下更改：

```
1 FROM new_busybox:ver_1
2
3 RUN echo "#!/bin/sh\n" > /version.sh
4 RUN echo "echo \"This is Version 2 of our service\""   >> /version.sh
5
6 ENTRYPOINT ["sh", "/version.sh"]
```

1.  构建您修改后的`Dockerfile`并使用`version2`标记：

```
docker build -t vince/test:version2 .
```

1.  使用`docker run`命令运行修改后的映像：

```
docker run vince/test
```

您应该看到您最新的代码更改。

```
This is Version 1 of our service
```

这不是我们要找的版本，是吗？如果不使用正确的标签，Docker 将运行带有`latest`标签的最新版本的映像。此映像是在*步骤 3*中创建的。

1.  现在，使用`latest`和`version2`标签运行两个映像：

```
docker run vince/test:latest
This is Version 1 of our service
```

现在我们可以看到输出的差异：

```
docker run vince/test:version2
This is Version 2 of our service
```

正如您可能已经想到的，您需要指定`version2`标签来运行修改后的代码版本。您可能已经预料到了，但请记住，如果您有多个开发人员将映像推送到共享注册表，这将使跟踪变得更加困难。如果您的团队正在使用编排并使用`latest`版本，您可能会在生产环境中运行混合版本的服务。

这些练习为您提供了如何使用标签的示例，同时向您展示了如果决定仅使用`latest`标签可能会导致的后果。接下来的部分将介绍标记策略以及如何实施自动化流程。

# Docker 映像标记策略

随着开发团队规模的增加和他们所工作的项目复杂性的增加，团队的标记策略变得更加重要。如果您的团队没有正确使用标记，就像我们在之前的部分中所演示的那样，这可能会导致很多混乱，实际上会导致更多问题。早期制定标记策略是一个好习惯，以确保您不会遇到这些问题。

在本章的这一部分，我们将涵盖团队内可以使用的不同标记策略，并举例说明它们如何实施。在设置标记策略时很少有对错之分，但需要及早做出决定，并确保团队中的每个人都同意。

**语义化版本控制**是一个版本控制系统，也可以作为标记策略的一部分使用。如果您不熟悉语义化版本控制，它是一个可信赖的版本系统，使用`major_version.minor_version.patch`格式的三部分数字。例如，如果您看到一个应用程序的语义版本是 2.1.0，它将显示版本 2 为主要发布版本，1 为次要发布版本，0 为没有补丁。语义化版本控制可以很容易地自动化，特别是在自动化构建环境中。另一个选择是使用哈希值，比如您的代码的`git commit`哈希。这意味着您可以将标记与您的存储库匹配，这样任何人都可以具体看到自代码实施以来所做的代码更改。您还可以使用日期值，这也可以很容易地自动化。

这里的共同主题是我们的标记策略应该是自动化的，以确保它被使用、理解和遵守。在接下来的练习中，我们将研究使用哈希值作为标记策略的一部分，然后创建一个脚本来构建我们的 Docker 图像，并为我们的标记添加语义版本控制。

## 练习 3.07：自动化您的图像标记

在这个练习中，您将研究如何自动化图像标记，以减少标记 Docker 图像所需的个人干预量。这个练习再次使用`basic-base`图像：

1.  通过创建以下`Dockerfile`再次创建`basic-base`图像：

```
1 FROM alpine
2
3 RUN apk update && apk add wget curl
```

1.  从前面的`Dockerfile`构建新的基础图像，并将其命名为`basic-base`：

```
docker build -t basic-base .
```

1.  创建`basic-base`图像后，设置名为`Dockerfile_ver1`的`Dockerfile`以再次构建`basic-app`。在这种情况下，返回到此处列出的先前的`Dockerfile`：

```
1 FROM basic-base
2
3 CMD mkdir -p /var/www/html/
4
5 WORKDIR /var/www/html/
6
7 ADD Dockerfile.tar.gz /var/www/html/
8 RUN cat Dockerfile
```

1.  如果您一直在使用 Git 跟踪和提交代码更改，您可以使用`git log`命令将图像标记为来自 Git 的提交哈希。因此，像往常一样使用`docker build`命令构建新图像，但在这种情况下，添加标签以提供来自`git`的短提交哈希：

```
docker build -t basic-app:$(git log -1 --format=%h) .
...
Successfully tagged basic-app:503a2eb
```

注意

如果您是 Git 的新手，它是一个源代码控制应用程序，允许您跟踪更改并与其他用户在不同的编码项目上进行协作。如果您以前从未使用过 Git，则以下命令将初始化您的存储库，将`Dockerfile`添加到存储库，并提交这些更改，以便我们有一个 Git 日志：

`git init; git add Dockerfile; git commit –m "initial commit"`

1.  使用您的`Dockerfile`在构建图像时添加参数。打开您一直在为`basic-app`使用的`Dockerfile`，并添加以下两行以将变量设置为未知，然后在构建时将`LABEL`设置为使用`git-commit`构建参数提供的值。您的`Dockerfile`现在应如下所示：

```
1 FROM basic-base
2
3 ARG GIT_COMMIT=unknown
4 LABEL git-commit=$GIT_COMMIT
5
6 CMD mkdir -p /var/www/html/
7
8 WORKDIR /var/www/html/
9
10 ADD Dockerfile.tar.gz /var/www/html/
11 RUN cat Dockerfile
```

1.  再次使用`--build-arg`选项构建图像，并使用`GIT_COMMIT`参数，该参数现在等于您的`git commit`哈希值：

```
docker build -t basic-app --build-arg GIT_COMMIT=$(git log -1 --format=%h) .
```

1.  运行`docker inspect`命令，搜索`"git-commit"`标签：

```
docker inspect -f '{{index .ContainerConfig.Labels "git-commit"}}' basic-app
```

您可以在构建时看到您添加的 Git 哈希标签：

```
503a2eb
```

这开始朝着您需要的方向发展，但是如果您需要使用语义版本控制，因为您的团队已经决定这是开发的最佳选项，该怎么办？本练习的其余部分将设置一个构建脚本，用于构建和设置标签为语义版本号。

1.  在您的`Dockerfile`旁边，创建一个名为`VERSION`的版本文件。将`basic-app`的此构建的新版本设置为`1.0.0`：

```
echo "1.0.0" > VERSION
```

1.  对`Dockerfile`进行更改，以删除先前添加的`GIT_COMMIT`详细信息，并将`VERSION`文件添加为构建的一部分。将其添加到图像本身意味着用户可以随时参考`VERSION`文件，以验证图像版本号：

```
1 FROM basic-base
2
3 CMD mkdir -p /var/www/html/
4
5 WORKDIR /var/www/html/
6
7 ADD VERSION /var/www/html/
8 ADD Dockerfile.tar.gz /var/www/html/
9 RUN cat Dockerfile
```

1.  创建一个构建脚本来构建和标记您的图像。将其命名为`build.sh`，并且它将驻留在与您的`Dockerfile`和`VERSION`文件相同的目录中：

```
touch build.sh
```

1.  将以下详细信息添加到`build.sh`。*第 3 行*将是您的 Docker Hub 用户名，*第 4 行*是您正在构建的图像或服务的名称（在以下示例中为`basic-app`）。然后，脚本从您的`VERSION`文件中获取版本号，并将所有变量汇集在一起，以使用与您的新语义版本相关的漂亮名称和标记构建您的图像：

```
1 set -ex
2
3 USER=<your_user_name>
4 SERVICENAME=basic-app
5
6 version=`cat VERSION`
7 echo "version: $version"
8
9 docker build -t $USER/$SERVICENAME:$version .
```

1.  确保构建脚本已设置为可执行脚本，使用命令行上的`chmod`命令：

```
chmod +x build.sh 
```

1.  从命令行运行构建脚本。`set -xe`在脚本的*第 1 行*将确保所有命令都输出到控制台，并确保如果任何命令导致错误，脚本将停止。现在运行构建脚本，如下所示：

```
./build.sh 
```

这里只显示了构建脚本的输出，其余的构建过程都是正常进行的：

```
++ USERNAME=vincesestodocker
++ IMAGE=basic-app
+++ cat VERSION
++ version=1.0.0
++ echo 'version: 1.0.0'
version: 1.0.0
++ docker build -t vincesestodocker/basic-app:1.0.0 .
```

1.  使用`docker images`命令查看图像：

```
docker images vincesestodocker/basic-app
```

它应该反映在构建脚本中创建的名称和标记：

```
REPOSITORY                   TAG    IMAGE ID
  CREATED            SIZE
vincesestodocker/basic-app   1.0.0  94d0d337a28c
  29 minutes ago     8.8MB
```

这项练习在自动化我们的标记过程中起到了很大作用，并且允许将`build`脚本添加到源代码控制中，并作为构建流水线的一部分轻松运行。不过，这只是一个开始，你将在本章末尾的活动中看到，我们将进一步扩展这个构建脚本。目前，我们已经完成了关于图像的标记和命名的部分，并且它与下一部分很好地契合，该部分涵盖了存储和发布您的 Docker 图像。

# 存储和发布您的 Docker 图像

自 Docker 历史的早期以来，它的主要吸引力之一就是一个中央网站，用户可以在那里下载图像，重用和改进这些图像以满足他们的目的，并重新上传它们以授予其他用户访问权限。Docker Hub 已经发展壮大，尽管它曾经存在一些安全问题，但通常仍然是人们需要新图像或项目资源时首先寻找的地方。

作为一个公共存储库，Docker Hub 仍然是人们研究和使用图像所需的第一个地方，以简化或改进他们的新开发项目。对于公司和开发人员来说，它也是一个重要的地方，用于托管他们的开源图像，供公众利用。然而，Docker Hub 并不是您存储和分发 Docker 图像的唯一解决方案。

对于开发团队来说，Docker Hub 上的公共存储库虽然易于访问且高度可用，但可能不是最佳选择。如今，您的团队可能会考虑将生产图像存储在基于云的注册表解决方案中，例如 Amazon Elastic Container Registry、Google Container Registry，或者正如本章后面将看到的，另一个选择是设置本地注册表。

在本章的这一部分，我们将首先看看如何实际将图像从一台机器移动到另一台机器，然后更仔细地了解如何使用 Docker Hub。我们将看到如何开始将我们的图像移动到 Docker Hub 作为公开存储的图像。然后，我们将看看如何在开发系统上设置一个本地托管的 Docker 注册表。

`docker save`命令将用于从命令行保存图像。在这里，我们使用`-o`选项来指定输出文件和目录，我们将保存图像到该目录中：

```
docker save -o <output_file_and_Directory> <image_repo_name/image_name:tag>
```

然后，我们将能够使用`load`命令，类似于本章前面创建新基础图像时使用的`import`命令，指定我们之前创建的文件。

```
docker load -i <output_file_and_Directory>
```

请记住，并非所有 Docker Hub 上的图像都应该以相同的方式对待，因为它包含了由 Docker Inc.创建的官方图像和由 Docker 用户创建的社区图像的混合物。官方图像仍然是开源图像和解决方案，可供您添加到您的项目中。社区图像通常由公司或个人提供，希望您利用他们的技术。

注意

即使是从 Docker Hub 获取图像时也要小心。尽量限制从不可靠来源拉取图像，并且尽量避免那些没有经过大量用户审核或下载的来源，因为它们可能构成潜在的安全风险。

## 练习 3.08：手动传输 Docker 图像

有时，无论是网络上的防火墙问题还是其他安全措施，您可能需要直接从一台系统复制图像到另一台系统。幸运的是，Docker 有一种实现这一点的方法，在这个练习中，您将在不使用注册表的情况下将图像从一台系统移动到另一台系统：

1.  使用`docker save`命令和`-o`选项来保存本章最后部分创建的图像。该命令需要用户指定文件名和目录。在下面的示例中，它是`/tmp/basic-app.tar`。还要指定图像的用户、图像名称和标签。

```
docker save -o /tmp/basic-app.tar vincesestodocker/basic-app:1.0.0
```

现在，您应该在`/tmp`目录中看到打包的镜像。您正在使用`.tar`作为文件名的扩展名，因为`save`命令会创建镜像的 TAR 文件。实际上，您可以为文件的扩展名使用任何名称。

1.  使用`du`命令验证`basic-app.tar`文件中是否有数据：

```
du -sh /tmp/basic-app.tar 
8.9M    /tmp/basic-app.tar
```

1.  现在，您可以根据需要移动镜像，无论是通过`rsync`、`scp`还是`cp`。由于它是一个 TAR 文件，如果需要在传输过程中节省一些空间，您还可以将文件压缩为 ZIP 文件。在这个例子中，您将简单地从当前系统中删除镜像。运行`docker rmi`命令，后面跟着您刚保存的镜像的 ID：

```
docker rmi -f 94d0d337a28c
```

1.  使用`docker load`命令将新镜像作为 Docker 镜像加载回来，使用`-i`选项，指向打包镜像的位置。在这种情况下，它是`/tmp`目录：

```
docker load -i /tmp/basic-app.tar 
```

您应该会得到以下输出：

```
Loaded image: vincesestodocker/basic-app:1.0.0
```

1.  使用`docker image`命令将您刚刚加载到本地环境中的镜像启动：

```
docker images vincesestodocker/basic-app
```

您应该会得到以下输出：

```
REPOSITORY                    TAG      IMAGE ID
  CREATED             SIZE
vincesestodocker/basic-app    1.0.0    2056b6e48b1a
  29 minutes ago      8.8MB
```

这只是一个简单的练习，但希望它能向您展示，如果有任何情况导致您无法连接到注册表，您仍然可以传输您的 Docker 镜像。接下来的练习更侧重于存储、发布和分发 Docker 镜像的常规方法。

## 在 Docker Hub 中存储和删除 Docker 镜像

尽管您可以在 Docker Hub 上免费使用，但您需要知道，您的帐户只能免费获得一个私有存储库。如果您想要更多，您需要在 Docker 上支付月度计划。如果 Docker Hub 是您的团队选择使用的解决方案，您很少需要只有一个私有存储库。如果您决定免费帐户适合您，那么您将获得无限数量的免费存储库。

## 练习 3.09：在 Docker Hub 中存储 Docker 镜像并删除存储库

在这个练习中，您将为您正在工作的`basic-app`创建一个新的存储库，并将镜像存储在 Docker Hub 中。一旦您将镜像推送到 Docker Hub，您还将看到如何删除存储库：

注意

以下练习将需要您在 Docker Hub 上拥有帐户。我们将只使用免费存储库，因此您不需要付费月度计划，但如果您还没有在 Docker Hub 上注册免费帐户，请转到[`hub.docker.com/signup`](https://hub.docker.com/signup)。

1.  登录到您的 Docker Hub 帐户，在`存储库`部分下，您将看到右侧的蓝色按钮`创建存储库`选项。单击此按钮，以便为您正在工作的`basic-app`设置存储库：![图 3.1：在 Docker Hub 中创建存储库](img/B15021_03_01.jpg)

图 3.1：在 Docker Hub 中创建存储库

1.  创建新存储库时，您将看到一个类似下面的页面。填写存储库的`名称`，通常是您要存储的图像或服务的名称（在本例中为`basic-app`）。您还可以选择将存储库设置为`公共`或`私有`，在这种情况下，选择`公共`：![图 3.2：Docker Hub 的存储库创建屏幕](img/B15021_03_02.jpg)

图 3.2：Docker Hub 的存储库创建屏幕

1.  在屏幕底部，还有构建图像的选项。单击屏幕底部的`创建`按钮：![图 3.3：Docker Hub 的存储库创建屏幕](img/B15021_03_03.jpg)

图 3.3：Docker Hub 的存储库创建屏幕

1.  创建新存储库后，它将提供有关如何开始将图像推送到新存储库的详细信息。使用`<account_name>/<image_name>:tag`标记您的图像，以便 Docker 知道它将推送图像的位置以及 Docker 将要将其推送到哪个存储库：

```
docker tag basic-app vincesestodocker/basic-app:ver1
```

1.  现在，您的系统上的 Docker 知道在哪里推送图像。使用`docker push <account_name>/<image_name>:tag`命令推送图像：

```
docker push vincesestodocker/basic-app:ver1
denied: requested access to the resource is denied
```

您需要确保您已经从命令行和 Web 界面登录到 Docker Hub。

1.  使用`docker login`命令，并输入创建新存储库时使用的相同凭据：

```
docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: vincesestodocker
Password: 
Login Succeeded
```

1.  现在，将您的图像推送到新的存储库，就像在本练习的*步骤 5*中所做的那样，之前失败了。它应该给您一个成功的结果：

```
docker push basic-app vincesestodocker/basic-app:ver1
```

1.  返回 Docker Hub Web 界面，现在您应该看到您推送的图像版本，位于您新创建的存储库中：![图 3.4：您新创建的 Docker Hub 存储库中的图像](img/B15021_03_04.jpg)

图 3.4：您新创建的 Docker Hub 存储库中的图像

现在您有一个公共存储库可供任何想要拉取您的镜像并为其目的重用的人使用。如果有人需要使用您的镜像，他们只需使用镜像的完整名称，包括`docker pull`命令或`Dockerfile`中的`FROM`命令。

1.  您会注意到在前面的图像中，屏幕右侧有一个“公共视图”按钮。这给了您一个选项，可以看到公众在搜索您的镜像时会看到什么。点击按钮，您应该会看到类似以下的屏幕：![图 3.5：您的 Docker Hub 存储库的公共视图](img/B15021_03_05.jpg)

图 3.5：您的 Docker Hub 存储库的公共视图

这正是公众将看到的您的存储库。现在轮到您确保您的概述是最新的，并确保您的镜像得到支持，以确保任何想要使用您的镜像的人没有问题。

1.  最后，在这个练习中，清理您刚刚创建的存储库。如果您还没有在存储库的网络界面中，请返回到 Docker Hub 网页，然后点击屏幕顶部的“设置”选项卡：![图 3.6：Docker Hub 存储库的设置屏幕](img/B15021_03_06.jpg)

图 3.6：Docker Hub 存储库的设置屏幕

1.  在这里，您将有选择使您的存储库私有，但在这个练习中，您将删除存储库。点击“删除存储库”选项，并确认您现在要删除它。

在这个练习中，Docker Hub 为您提供了一个简单的方式来分发镜像，以便其他用户可以合作或利用您已经完成的工作。公共存储库并不总是企业的最佳选择，但就像 GitHub 允许开发人员分发他们的代码并与其他开发人员合作一样，Docker Hub 也可以为您的 Docker 镜像做同样的事情。

# Docker Registry

Docker Registry 是托管您的镜像的服务。大多数情况下，存储库是私有的，只对团队有权限访问。有很多很好的选择，其中之一是 Docker 提供和维护的注册表镜像。

有几个不同的原因会让您想要运行自己的 Docker 注册表。这可能是由于安全问题，或者您不希望您的最新工作公开可用。甚至可能只是为了方便，在您正在工作的系统上运行您的注册表。在本章的这一部分，我们将在您的工作环境上设置一个注册表，并开始将您的镜像存储在注册表上。

注意

Docker 为我们简化了事情，因为他们在 Docker Hub 上提供了一个注册表镜像供您下载并用于您的项目。有关我们将要使用的镜像的更多信息，您可以在以下位置找到：

[`hub.docker.com/_/registry`](https://hub.docker.com/_/registry)。

## 练习 3.10：创建本地 Docker 注册表

在这个练习中，您将为您的镜像设置一个 Docker 注册表，并在您的系统上运行它们。您不打算为您的团队或外部世界设置一个可用的注册表。您将设置一个漂亮的域名在您的系统上使用，以反映您正在进行的工作。这将帮助您决定是否将此注册表提供给您的团队或其他用户：

1.  要设置您的域名，请将本地注册表的域名添加到您的系统主机文件中。在 Windows 系统上，您需要访问`C:\Windows\System32\drivers\etc\hosts`中的主机文件，而在 Linux 或 Max 上，它将是`/etc/hosts`。打开`hosts`文件并将以下行添加到文件中：

```
127.0.0.1       dev.docker.local
```

这将允许您使用`dev.docker.local`域，而不是在本地注册表中使用 localhost。

1.  从 Docker Hub 拉取最新的`registry`镜像：

```
docker pull registry
```

1.  使用以下命令来运行注册表容器。提供您可以访问注册表的端口；在这种情况下，使用端口`5000`。您还需要使用`--restart=always`选项，这将确保容器在 Docker 或系统需要重新启动时重新启动：

```
docker run -d -p 5000:5000 --restart=always --name registry registry
```

注意

在接下来的章节中，您将学习如何通过挂载来自主机系统的目录来扩展 Docker 容器的文件容量，然后作为运行容器的一部分运行。为此，您将使用`-v`或`--volume`选项作为您的`docker run`命令的一部分，提供文件和容器上的挂载点。例如，您可以运行上述命令来挂载主机系统上的目录如下：

`docker run -d -p 5000:5000 --restart=always --volume <directory_name>:/var/lib/registry:rw --name registry`

1.  运行`docker ps`命令，显示在您的系统上运行的`registry`容器，该容器已准备好接受并存储新的映像：

```
docker ps
```

该命令将返回以下输出：

```
CONTAINER ID  IMAGE     COMMAND                 CREATED
41664c379bec  registry  "/entrypoint.sh /etc…"  58 seconds ago
```

1.  运行`docker tag`命令，使用注册表主机名和端口`dev.docker.local:5000`对现有映像进行标记。

```
docker tag vincesestodocker/basic-app:ver1 dev.docker.local:5000/basic-app:ver1
```

这将确保您的`basic-app`映像将自动推送到本地注册表：

```
docker push dev.docker.local:5000/basic-app:ver1
```

1.  使用`docker image remove`命令从您当前正在使用的系统中删除原始映像：

```
docker image remove dev.docker.local:5000/basic-app:ver1
```

1.  现在，通过在`pull`命令中包含注册表主机名和端口`dev.docker.local:5000`，从本地注册表中拉取映像：

```
docker pull dev.docker.local:5000/basic-app:ver1
```

这将带我们到本节的结束，我们已经在本地系统上创建了我们的注册表来存储我们的 Docker 映像。注册表本身很简单，实际上并不受支持，但它确实有助于帮助您了解注册表将如何工作以及如何与您的团队合作。如果您正在寻找更强大和受支持的映像，Docker 还提供了 Docker Trusted Registry，这是 Docker 提供的商业产品。

现在是时候测试到目前为止所学到的知识了。在下一个活动中，我们将修改`PostgreSQL`容器映像的构建脚本，以使用 Git 提交哈希而不是语义版本。

## 活动 3.01：使用 Git 哈希版本化的构建脚本

在本章的前面，您创建了一个构建脚本，自动化了正在构建的映像的标记和版本化过程。在这个活动中，您将进一步使用全景徒步应用程序，并被要求为`PostgreSQL`容器映像设置一个构建脚本。您可以使用之前创建的构建脚本，但是您需要修改脚本，不再使用语义版本，而是使用当前的 Git 提交哈希。另外，请确保您的构建脚本将构建的映像推送到您的 Docker 注册表。

完成所需的步骤如下：

1.  确保您已经为您的`PostgreSQL`容器映像创建了一个运行的`Dockerfile`。

1.  创建您的构建脚本，执行以下操作：

a) 设置您的 Docker 注册表的变量，正在构建的服务名称和 Git 哈希版本

b) 将 Git 哈希版本打印到屏幕上

c) 构建您的 PostgreSQL Docker 映像

d) 将您的 Docker 映像推送到您的注册表

1.  确保构建脚本运行并成功完成。

预期输出：

```
./BuildScript.sh 
++ REGISTRY=dev.docker.local:5000
++ SERVICENAME=basic-app
+++ git log -1 --format=%h
++ GIT_VERSION=49d3a10
++ echo 'version: 49d3a10 '
version: 49d3a10 
++ docker build -t dev.docker.local:5000/basic-app:49d3a10 .
Sending build context to Docker daemon  3.072kB
Step 1/1 : FROM postgres
 ---> 873ed24f782e
Successfully built 873ed24f782e
Successfully tagged dev.docker.local:5000/basic-app:49d3a10
++ docker push dev.docker.local:5000/basic-app:49d3a10
The push refers to repository [dev.docker.local:5000/basic-app]
```

注意

此活动的解决方案可以通过此链接找到。

在下一个活动中，您将通过更改`docker run`命令将本地 Docker 注册表存储在您的主目录中的一个目录中。

## 活动 3.02：配置本地 Docker 注册表存储

在本章中，您设置了您的注册表并开始使用基本选项来使其运行。注册表本身正在主机文件系统上存储镜像。在这个活动中，您希望更改`docker run`命令以将其存储在您的主目录中的一个目录中。您将创建一个名为`test_registry`的目录，并运行 Docker 命令将镜像存储在您的主目录中的`test_registry`目录中。

完成所需的步骤如下：

1.  在您的主目录中创建一个目录来挂载您的本地注册表。

1.  运行本地注册表。这次将新创建的卷挂载为注册表的一部分。

1.  通过将新镜像推送到本地注册表来测试您的更改。

提示

在运行注册表容器时使用`-v`或`–volume`选项。

**预期输出：**

在列出本地目录中的所有文件时，您将能够看到推送的图像：

```
ls  ~/test_registry/registry/docker/registry/v2/repositories/
basic-app
```

注意

此活动的解决方案可以通过此链接找到。

# 总结

本章演示了 Docker 如何允许用户使用镜像将其应用程序与工作环境一起打包，以便在不同的工作环境之间移动。您已经了解到 Docker 如何使用层和缓存来提高构建速度，并确保您也可以使用这些层来保留资源或磁盘空间。

我们还花了一些时间创建了一个只有一个图像层的基本图像。我们探讨了标记和标记实践，您可以采用这些实践来解决部署和发布图像相关的问题。我们还看了一下我们可以发布图像和与其他用户和开发人员共享图像的不同方法。我们才刚刚开始，还有很长的路要走。

在下一章中，我们将进一步使用我们的`Dockerfiles`来学习多阶段`Dockerfiles`的工作原理。我们还将找到更多优化我们的 Docker 镜像的方法，以便在发布到生产环境时获得更好的性能。

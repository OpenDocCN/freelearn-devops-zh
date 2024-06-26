# 第二章：学习 Docker 命令

在本章中，我们将学习一些基本的 Docker 命令。虽然我们将重点关注最重要的命令之一，即`container run`命令，但我们也将涵盖许多其他您每天都会使用的命令。这些命令包括列出容器命令、停止容器命令和删除容器命令。在学习过程中，我们还将了解其他容器命令，如日志、检查、统计、附加、执行和提交。我认为您会发现本章对 Docker 教育是一个很好的基础。

BIC：国际集装箱局成立于 1933 年，是一个中立的、非营利性的国际组织，其使命是促进集装箱化和联运交通的安全、安全和可持续发展。

在本章结束时，您将了解以下内容：

+   当前和以前的命令行语法

+   使用版本命令的两种方式

+   如何使用`container run`命令及其许多可选参数

+   如何启动和停止容器、查看容器信息、与运行中的容器交互，以及如何保存和重用对容器所做的更改

# 技术要求

您将从 Docker 的公共仓库中拉取 Docker 镜像，并安装 jq 软件包，因此需要基本的互联网访问权限来执行本章中的示例。

本章的代码文件可以在 GitHub 上找到：

[`github.com/PacktPublishing/Docker-Quick-Start-Guide/tree/master/Chapter02`](https://github.com/PacktPublishing/Docker-Quick-Start-Guide/tree/master/Chapter02)

查看以下视频以查看代码的实际操作：[`bit.ly/2P43WNT`](http://bit.ly/2P43WNT)

# 命令语法信息

在我们深入学习 Docker 命令及其众多选项之前，我想通知您的是 Docker CLI 在 2017 年 1 月发生了变化。

随着每个新版本的发布，Docker 命令和相关选项的数量都在增加。Docker 决定需要解决这种复杂性。因此，随着 Docker 版本 1.13 的发布（Docker 还在 2017 年更改了版本编号方案），CLI 命令已被划分为管理功能组。例如，现在有一个容器管理组的命令，以及一个镜像管理组的命令。这改变了您运行 Docker 命令的方式。以下是旧和新`run`命令的使用示例：

```
# the new command syntax...
docker container run hello-world
# the old command syntax...
docker run hello-world
```

这个变化提供了更好的命令组织，但也增加了命令行的冗长。这是一个权衡。就我所知，目前为止，旧的命令语法仍然适用于所有 Docker 命令，但在本书的其余示例中，我打算使用新的语法。至少我会尝试，因为旧习惯难改。

我想在这里提一点，大多数命令选项都有短格式和长格式。我会尝试在我的示例中至少分享一次长格式，这样你就会知道短版本代表什么。如果你安装了 Docker 命令行完成，它将是一个有用的资源，可以记住新的基于 Docker 管理的命令和可以与之一起使用的参数。这是容器命令的顶级命令完成帮助的样子：

![](img/f5fa4bbe-0891-4883-8c95-ddb910ca7e9a.png)

该命令列表让我们提前了解了一些我们将在本章中审查的命令，所以让我们开始学习 Docker 命令。在第一章中，*设置 Docker 开发环境*，我们使用了两个非常常见的 Docker 命令：`version`命令和`run`命令。虽然你认为你已经了解了`version`命令的所有内容，但你可能会惊讶地发现它还有另一个技巧。Docker 的 version 命令还有另一个版本。

# version 命令

你已经使用了`docker --version`命令作为一个快速测试来确认 Docker 是否已安装。现在尝试一下没有破折号的命令：

```
docker version
```

这个版本的命令可以更详细地了解安装在系统上的 Docker 的版本。值得注意的是，docker-compose 命令，我们稍后会谈到，也有两个版本的 version 命令——一个带有破折号提供单行响应，另一个没有破折号，提供更多细节。

请记住，所有 Docker 命令都有一个丰富的帮助系统。尝试输入 Docker 命令的任何部分并使用`--help`参数来查看。例如，`docker container run --help`。

# Docker run 命令

由于我们将经常使用`run`命令，我们现在应该看一下。你已经以其最基本的形式使用了`run`命令：

```
# new syntax
# Usage: docker container run [OPTIONS] IMAGE [COMMAND] [ARG...]
docker container run hello-world

# old syntax
docker run hello-world
```

这个命令告诉 Docker，您想要基于描述为 hello-world 的镜像运行一个容器。您可能会问自己，当我安装 Docker 时，hello-world 容器镜像是否已安装？答案是否定的。`docker run`命令将查看本地容器镜像缓存，以查看是否有与所请求容器描述匹配的容器镜像。如果有，Docker 将从缓存的镜像中运行容器。如果在缓存中找不到所需的容器镜像，Docker 将访问 Docker 注册表，尝试下载容器镜像，并在此过程中将其存储在本地缓存中。然后 Docker 将从缓存中运行新下载的容器。

Docker 注册表只是一个集中存储和检索 Docker 镜像的地方。我们稍后会更多地讨论注册表和 Docker 注册表。现在，只需了解有本地镜像缓存和远程镜像存储这一点。当我们在第一章中运行 hello-world 容器时，您看到了本地未找到容器的过程，*设置 Docker 开发环境。*当 Docker 在本地缓存中找不到容器镜像并且必须从注册表中下载时，情况是这样的：

![](img/8d336a4c-94d9-4a62-832f-c6a370eb84de.png)

您可以使用 docker `pull`命令预先填充本地 docker 缓存，以便运行您计划运行的容器镜像；例如：

```
# new syntax
# Usage: docker image pull [OPTIONS] NAME[:TAG|@DIGEST]
docker image pull hello-world

# old syntax
docker pull hello-world
```

如果您使用`pull`命令预取容器镜像，当您执行 docker `run`命令时，它将在本地缓存中找到镜像，而无需再次下载。

您可能已经注意到在前面的屏幕截图中，您请求了 hello-world 容器镜像，Docker 未能在本地缓存中找到，然后从存储库中下载了`hello-world:latest`容器镜像。每个容器镜像描述由三个部分组成：

+   Docker 注册表主机名

+   斜杠分隔的名称

+   标签名称

第一部分，注册表主机名，我们还没有看到或使用过，但它是通过公共 Docker 注册表的默认值包含的。每当您不指定注册表主机名时，Docker 将隐式使用公共 Docker 注册表。此注册表主机名是`docker.io`。Docker 注册表的内容可以在[`hub.docker.com/explore`](https://hub.docker.com/explore)上浏览。这是 Docker 镜像的主要公共存储库。可以设置和使用其他公共或私有镜像注册表，并且许多公司将这样做，建立自己的私有 Docker 镜像注册表。我们将在第八章“Docker 和 Jenkins”中再谈一些相关内容。现在，只需了解 Docker 镜像描述的第一部分是托管容器镜像的注册表主机名。值得注意的是，注册表主机名可以包括端口号。这可以用于配置为在非默认端口值上提供数据的注册表。

容器镜像描述的第二部分是斜杠分隔的名称。这部分就像是容器镜像的路径和名称。有一些官方容器镜像不需要指定路径。对于这些镜像，您可以简单地指定斜杠分隔名称的名称部分。在我们的示例中，这是描述的 hello-world 部分。

容器镜像描述的第三部分是标签名称。这部分被认为是镜像的版本标签，但它不需要仅由数字组成。标签名称可以是任何一组 ASCII 字符，包括大写和小写字母，数字，破折号，下划线或句点。关于标签名称的唯一限制是它们不能以句点或破折号开头，并且必须少于 128 个字符。标签名称与斜杠分隔的名称之间用冒号分隔。这让我们回到之前看到的`hello-world:latest`镜像描述。与注册表主机名一样，标签名称有一个默认值。默认值是`latest`。在我们的示例中，使用的标签名称是默认值，并且在搜索和下载中显示为`hello-world:latest`。您可以在以下示例中看到所有这些内容：

![](img/8e50f5e2-4ba1-4a13-962d-84665c6b3369.png)

我们确认了我们的本地镜像缓存是空的，使用`docker images`命令，然后拉取了完全限定的 hello-world 镜像以预取到我们的本地缓存中。然后我们使用了与之前所有的 hello-world 示例中相同的简短描述，Docker 运行容器而不再次下载，显示使用了默认值并且它们与完全限定的值匹配。

好的，现在我们已经了解了 Docker `run`命令的所有基础知识，让我们深入一点，检查一些你可以与`run`命令一起使用的可选参数。如果你查看完整的`run`命令语法，你会看到这样的内容：

```
# Usage:  docker container run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

请注意命令的最后部分是`[COMMAND] [ARG...]`。这告诉我们`container run`命令有一个可选的命令参数，也可以包括自己的可选参数。Docker 容器镜像是使用默认命令构建的，当你基于该镜像运行容器时，会执行该默认命令。对于 hello-world 容器，默认命令是`/hello`。对于完整的 Ubuntu OS 容器，默认命令是`bash`。每当你运行一个 Ubuntu 容器并且没有指定在容器中运行的命令时，将使用默认命令。如果现在这些还不太清楚，不要担心——我们将在本章的*回到 Docker 运行命令*部分稍后讨论默认命令和在运行时覆盖它。现在，知道当你运行一个容器时，它将执行一个命令，要么是默认命令，要么是提供给`container run`命令的覆盖命令来在运行的容器中执行。最后一点注意：当运行容器的命令（默认或覆盖）终止时，容器将退出。在我们使用 hello-world 容器的示例中，一旦容器内的`/hello`命令终止，hello-world 容器就会退出。一会儿，你将了解更多关于运行中容器和已退出容器之间的区别。

现在，我们将继续讨论`run`命令的一个我最喜欢的可选参数，即`--rm`参数。这里需要一些背景信息。您可能还记得来自第一章的*设置 Docker 开发环境*，Docker 镜像由多个层组成。每当您运行一个 Docker 容器时，实际上只是使用本地缓存的 Docker 镜像（这是一堆层），并在其顶部创建一个新的读/写层。在容器运行期间发生的所有执行和更改都存储在其自己的读/写层中。

# 列出容器的命令

可以使用以下命令显示运行中的容器：

```
# Usage: docker container ls [OPTIONS]
docker container ls
```

这是列出容器的命令，如果没有任何额外的参数，它将列出当前正在运行的容器。我所说的当前运行是什么意思？容器是在系统上运行的特殊进程，就像系统上的其他进程一样，容器可以停止或退出。然而，与系统上其他类型的进程不同，容器的默认行为是在停止时保留其读/写层。这是因为如果需要，您可以重新启动容器，保持其退出时的状态数据。举个例子，假设您运行一个作为操作系统的容器，比如 Ubuntu，在该容器中安装了`wget`。容器退出后，您可以重新启动它，它仍然安装了`wget`。请记住，每个运行的容器都有自己的读/写层，因此，如果您运行一个 Ubuntu 容器并安装了`wget`，然后运行另一个 Ubuntu 容器，它将不会有`wget`。读/写层在容器之间不共享。但是，如果重新启动安装了`wget`的容器，它仍然会安装。

因此，运行中的容器和停止的容器之间的区别在于进程是正在运行还是已退出，留下了自己的读/写层。有一个参数可以让您列出所有容器，包括正在运行和已退出的容器。您可能已经猜到了，它是`--all`参数，它看起来像这样：

```
# short form of the parameter is -a
docker container ls -a
# long form is --all
docker container ls --all

# old syntax
docker ps -a
```

现在，让我们回到我最喜欢的可选运行命令参数之一，即`--rm`参数：

```
# there is no short form of the --rm parameter
docker container run --rm hello-world
```

此参数指示 Docker 在容器退出时自动删除容器的读/写层。当您运行一个 docker 容器而没有`--rm`参数时，容器数据在容器退出时会被留下，以便稍后可以重新启动容器。然而，如果在运行容器时包括`--rm`参数，那么在容器退出时所有容器的读/写数据都会被删除。这个参数提供了一个在`exit`时进行简单清理的功能，这在很多情况下都会非常有用。让我们通过一个快速示例来看一下，使用我们刚刚讨论过的 run 和`container ls`命令：

![](img/537bbccb-96a6-49ac-8ed4-f80ef15988f7.png)

首先，我们确认我们的本地缓存中有 hello-world 镜像。接下来，我们列出了系统上所有的容器，包括正在运行和已退出的。请注意镜像和容器之间的区别。如果您熟悉 VMware，类似于模板和虚拟机的类比。接下来，我们使用`--rm`参数运行了 hello-world 容器。hello-world 容器打印其消息，然后立即退出（我们将输出重定向到`/dev/null`，以使示例输出变短）。接下来，我们再次列出了容器，因为我们看到 hello-world 容器的读/写数据在容器退出时被自动删除了。之后，我们再次运行了 hello-world 容器，但这次没有使用`--rm`参数。当我们这次列出容器时，我们看到了（已退出）容器的指示。通常，您会运行一个容器，知道您以后永远不需要重新启动它，并使用`--rm`参数自动清理它非常方便。但如果您不使用`--rm`参数会怎么样呢？您会被困在一个不断增长的容器列表中吗？当然不会。Docker 有一个命令来处理这个问题。那就是`container rm`命令。

# 删除容器命令

删除容器命令看起来像这样：

```
# the new syntax
# Usage: docker container rm [OPTIONS] CONTAINER [CONTAINER...]
docker container rm cd828234194a

# the old syntax
docker rm cd828234194a
```

该命令需要一个唯一标识容器的值；在本例中，我使用了刚刚运行的 hello-world 容器的完整容器 ID。您可以使用容器 ID 的前几个字符，只要它在系统上提供了唯一标识符。另一种唯一标识容器的方法是通过分配给它的`name`。当您运行容器时，Docker 将为其提供一个唯一的随机生成的名称。在上面的示例中，分配的随机名称是`competent_payne`。因此，我们可以像这样使用删除命令： 

```
# using the randomly generated name docker container rm competent_payne
```

虽然 Docker 提供的随机生成的名称比其分配的容器 ID 更易读，但它们可能仍然不如您希望的那样相关。这就是为什么 Docker 为`run`命令提供了一个可选参数来为您的容器命名。以下是使用`--name`参数的示例：

```
# using our own name docker container run --name hi-earl hello-world
```

现在，当我们列出所有容器时，我们可以看到我们的容器名称为`hi-earl`。当然，您可能希望使用更好的容器名称，也许是描述容器执行功能的名称，例如`db-for-earls-app`。

注意：与容器 ID 一样，容器名称在主机上必须是唯一的。您不能有两个具有相同名称的容器（即使其中一个已退出）。如果将有多个运行相同镜像的容器，例如 Web 服务器镜像，请为它们分配唯一的名称，例如 web01 和 web02。![](img/9d9bae7d-2569-4d62-b104-0f77cef7e52c.png)

您可以通过在命令行上提供每个容器的唯一标识符来同时删除多个容器：

```
# removing more than one docker container rm hi-earl hi-earl2
```

通常，您只会在容器退出后删除容器，例如我们一直在使用的 hello-world 容器。但是，有时您可能希望删除当前正在运行的容器。您可以使用`--force`参数来处理这种情况。以下是使用 force 参数删除运行中容器的示例：

```
# removing even if it is running docker container rm --force web-server
```

以下是它的样子：

![](img/e6d64ba5-07f6-4495-a2d1-b8e9b6964384.png)

请注意，在第一个`container ls`命令中，我们没有使用`--all`参数。这提醒我们 Web 服务器容器正在运行。当我们尝试删除它时，我们被告知容器仍在运行，不会被删除。这是一个很好的保障，有助于防止删除运行中的容器。接下来，我们使用了强制命令，运行中的容器被删除而没有任何警告。最后，我们进行了另一个`container ls`命令，包括`--all`参数，以显示这次实际上删除了我们容器的读/写数据。

如果您已经设置了 Docker 命令完成，您可以输入命令，直到需要输入容器的唯一标识符，然后使用*Tab*键获取容器列表，切换到您想要删除的容器。一旦您突出显示要删除的容器，使用空格键或*Enter*键进行选择。您可以再次按*Tab*键选择另一个要一次删除多个容器。选择所有容器后，按*Enter*执行命令。请记住，除非包括强制参数`rm -f`，否则在为`rm`命令切换时，您只会看到已停止的容器。

有时，您可能希望删除系统上的所有容器，无论是否正在运行。有一种有用的方法来处理这种情况。您可以结合`container ls`命令和容器删除命令来完成任务。您将使用`container ls`命令的新参数来完成这个任务——`--quiet`参数。此命令指示 Docker 仅返回容器 ID，而不是带有标题的完整列表。以下是命令：

```
# list just the container IDs docker container ls --all --quiet
```

现在我们可以将`container ls`命令返回的值作为输入参数提供给容器删除命令。它看起来像这样：

```
# using full parameter names
docker container rm --force $(docker container ls --all --quiet)
# using short parameter names
docker container rm -f $(docker container ls -aq)

# using the old syntax
docker rm -f $(docker ps -aq)
```

这将从您的系统中删除*所有*容器*运行和退出*，所以要小心！

您可能经常使用这个快捷方式，所以为它创建一个系统别名非常方便。

您可以将以下内容添加到您的`~/.bash_profile`或`~/zshrc`文件中：`alias RMAC='docker container rm --force $(docker container ls --all --quiet)'。

许多容器被设计为运行并立即退出，例如我们已经多次使用的 hello-world 示例。其他容器的镜像被创建为，当您使用它运行容器时，容器将继续运行，提供一些持续有用的功能，例如提供网页服务。当您运行一个持久的容器时，它将保持前台进程直到退出，并附加到进程：标准输入、标准输出和标准错误。这对于一些测试和开发用例来说是可以的，但通常情况下，这不适用于生产容器。相反，最好将`container run`作为后台进程运行，一旦启动就将控制权交还给您的终端会话。当然，有一个参数可以实现这一点。那就是`--detach`参数。使用该参数的效果如下：

```
# using the full form of the parameter
docker container run --detach --name web-server --rm nginx
# using the short form of the parameter
docker container run -d --name web-server --rm nginx
```

使用此参数将进程从前台会话中分离，并在容器启动后立即将控制权返回给您。您可能的下一个问题是，如何停止一个分离的容器？好吧，我很高兴您问了。您可以使用`container stop`命令。

# 停止容器命令

停止命令很容易使用。以下是命令的语法和示例：

```
# Usage: docker container stop [OPTIONS] CONTAINER [CONTAINER...]
docker container stop web-server
```

在我们的情况下，运行容器时我们使用了`--rm`参数，因此一旦容器停止，读/写层将被自动删除。与许多 Docker 命令一样，您可以提供多个唯一的容器标识符作为参数，以一条命令停止多个容器。

现在您可能会想知道，如果我使用`--detach`参数，我如何查看容器的运行情况？有几种方法可以从容器中获取信息。让我们在继续运行参数探索之前先看看其中一些。

# 容器日志命令

当您在前台运行容器时，容器发送到标准输出和标准错误的所有输出都会显示在运行容器的会话控制台中。然而，当您使用`--detach`参数时，容器一旦启动，会立即返回会话控制，因此您看不到发送到`stdout`和`stderr`的数据。如果您想查看这些数据，可以使用`container logs`命令。该命令如下：

```
# the long form of the command
# Usage: docker container logs [OPTIONS] CONTAINER
docker container logs --follow --timestamps web-server
# the short form of the command
docker container logs -f -t web-server

# get just the last 5 lines (there is no short form for the "--tail" parameter)
docker container logs --tail 5 web-server

# the old syntax
docker logs web-server
```

`--details`、`--follow`、`--timestamps`和`--tail`参数都是可选的，但我在这里包括了它们以供参考。当您使用`container logs`命令而没有可选参数时，它将只是将容器日志的所有内容转储到控制台。您可以使用`--tail`参数加上一个数字来仅转储最后几行。您可以组合这些参数（除了`--tail`和`--follow`）以获得您想要的结果。`--follow`参数就像在查看不断写入的日志时使用`tail -f`命令，并将每行写入日志时显示出来。您可以使用*Ctrl *+ *C* 退出正在跟踪的日志。`--timestamps`参数非常适合评估写入容器日志的频率。

# 容器顶部命令

您可能并不总是只想查看容器的日志；有时您想知道容器内运行着哪些进程。这就是`container top`命令的用处。理想情况下，每个容器都运行一个进程，但世界并不总是理想的，因此您可以使用这样的命令来查看目标容器中运行的所有进程：

```
# using the new syntax
# Usage: docker container top CONTAINER [ps OPTIONS]
docker container top web-server

# using the old syntax
docker top web-server
```

正如您可能期望的那样，`container top`命令只用于一次查看单个容器的进程。

# 容器检查命令

当您运行容器时，会有大量与容器关联的元数据。有许多时候您会想要查看那些元数据。用于执行此操作的命令是：

```
# using the new syntax
# Usage: docker container inspect [OPTIONS] CONTAINER [CONTAINER...]
docker container inspect web-server

# using the old syntax
docker inspect web-server
```

如前所述，此命令返回大量数据。您可能只对元数据的子集感兴趣。您可以使用`--format`参数来缩小返回的数据。查看这些示例：

+   获取一些状态数据：

```
# if you want to see the state of a container you can use this command
docker container inspect --format '{{json .State}}' web-server1 | jq

# if you want to narrow the state data to just when the container started, use this command
docker container inspect --format '{{json .State}}' web-server1 | jq '.StartedAt'
```

+   获取一些`NetworkSettings`数据：

```
# if you are interested in the container's network settings, use this command
docker container inspect --format '{{json .NetworkSettings}}' web-server1 | jq

# or maybe you just want to see the ports used by the container, here is a command for that
docker container inspect --format '{{json .NetworkSettings}}' web-server1 | jq '.Ports'

# maybe you just want the IP address used by the container, this is the command you could use.
docker container inspect -f '{{json .NetworkSettings}}' web-server1 | jq '.IPAddress'
```

+   使用单个命令获取多个容器的数据：

```
# maybe you want the IP Addresses for a couple containers
docker container inspect -f '{{json .NetworkSettings}}' web-server1 web-server2 | jq '.IPAddress'

# since the output for each container is a single line, this one can be done without using jq
docker container inspect -f '{{ .NetworkSettings.IPAddress }}' web-server1 web-server2 web-server3
```

这些示例大多使用 json 处理器`jq`。如果您尚未在系统上安装它，现在是一个很好的时机。以下是在本书中使用的每个操作系统上安装`jq`的命令：

```
# install jq on Mac OS
brew install jq

# install jq on ubuntu
sudo apt-get install jq

# install jq on RHEL/CentOS
yum install -y epel-release
yum install -y jq

# install jq on Windows using Chocolatey NuGet package manager
chocolatey install jq
```

inspect 命令的`--format`参数使用 go 模板。您可以在 Docker 文档页面上找到有关它们的更多信息，用于格式化输出：[`docs.docker.com/config/formatting`](https://docs.docker.com/config/formatting)。

# 容器统计命令

另一个非常有用的 Docker 命令是 stats 命令。它为一个或多个正在运行的容器提供实时、持续更新的使用统计信息。这有点像使用 Linux 的`top`命令。您可以不带参数运行该命令，以查看所有正在运行的容器的统计信息，或者您可以提供一个或多个唯一的容器标识符，以查看一个或多个容器的特定容器的统计信息。以下是使用该命令的一些示例：

```
# using the new syntax, view the stats for all running containers
# Usage: docker container stats [OPTIONS] [CONTAINER...]
docker container stats

# view the stats for just two web server containers
docker container stats web-server1 web-server2

# using the old syntax, view stats for all running containers
docker stats
```

当您查看了足够的统计信息后，您可以使用 C*trl* + *C*退出视图。

回到`run`命令参数，接下来，我们将讨论通常一起使用的`run`命令的两个参数。有时候你运行一个容器，你想与它进行交互式会话。例如，您可能运行一个在更多或更少完整的操作系统（如 Ubuntu）内执行某些应用程序的容器，并且您希望在该容器内部进行访问以更改配置或调试一些问题，类似于使用 SSH 连接到服务器。与大多数 Docker 相关的事情一样，有多种方法可以实现这一点。一种常见的方法是使用`run`命令的两个可选参数：`--interactive`和`--tty`。现在让我们看看它是如何工作的。您已经看到我们如何使用`--detach`参数启动与我们正在运行的容器断开连接：

```
# running detached docker container run --detach --name web-server1 nginx
```

当我们运行此命令启动我们的 nginx web 服务器并浏览`http://localhost`时，我们发现它没有提供我们期望的欢迎页面。因此，我们决定进行一些调试，而不是从容器中分离出来，我们决定使用两个`--interactive`和`--tty`参数进行交互式运行。现在，由于这是一个 nginx 容器，它在容器启动时执行一个默认命令。该命令是`nginx -g 'daemon off;'`。由于这是默认命令，与容器进行交互对我们没有任何好处。因此，我们将通过在运行命令中提供一个参数来覆盖默认命令。它看起来会像这样：

```
# using the long form of the parameters
docker container run --interactive --tty --name web-server2 nginx bash

# using the short form of the parameters (joined as one), which is much more common usage
docker container run -it --name web-server2 nginx bash
```

这个命令将像以前一样运行容器，但是不会执行默认命令，而是执行`bash`命令。它还会打开一个与容器交互的终端会话。根据需要，我们可以以`root`用户的身份在容器内执行命令。我们可以查看文件夹和文件，编辑配置设置，安装软件包等等。我们甚至可以运行镜像的默认命令，以查看是否解决了任何问题。这里有一个有点牵强的例子：

![](img/5355ddb7-ef52-4406-9397-c7839cf8de8f.png)

您可能已经注意到了`-p 80:80`参数。这是发布参数的简写形式，我们将在*回到 Docker 运行命令*部分讨论。使用`container ls`命令，您可以看到使用默认命令运行容器与使用覆盖命令运行容器之间的区别：

![](img/339ffd7a-c93b-466f-b9f1-e389a763598b.png)

Web 服务器运行使用了默认的 CMD，而 web-server2 使用了覆盖的 CMD `bash`。这是一个牵强的例子，帮助您理解这些概念。一个真实的例子可能是当您想要与基于操作系统的容器进行交互连接时，比如 Ubuntu。您可能还记得在第一章的开头，*设置 Docker 开发环境*中提到，默认在 Ubuntu 容器中运行的命令是`bash`。既然如此，您就不必提供一个命令来覆盖默认值。您可以使用这样的运行命令：

```
# running interactively with default CMD docker container run -it --name earls-dev ubuntu
```

使用这个`container run`命令，您可以连接到正在运行的 Ubuntu 容器的交互式终端会话。您可以做几乎任何您通常在连接到 Ubuntu 服务器时会做的事情。您可以使用`apt-get`安装软件，查看运行中的进程，执行`top`命令等等。可能会像这样：

![](img/9fafd0d2-f070-4432-bea9-b6d2a7a141ba.png)

还有一些其他容器命令可以帮助您与已经运行并分离的容器进行交互。现在让我们快速看一下这些命令。

# 容器附加命令

假设您有一个正在运行的容器。它当前与您的终端会话分离。您可以使用`container attach`命令将该容器的执行进程带到您的终端会话的前台进程。让我们使用之前使用过的 web 服务器示例：

```
# run a container detached
docker container run --detach -it --name web-server1 -p 80:80 nginx

# show that the container is running
docker container ps

# attach to the container
# Usage: docker container attach [OPTIONS] CONTAINER
docker container attach web-server1

# issue a *Ctrl* + *PQ* keystroke to detach (except for Docker on Mac, see below for special Mac instructions)

# again, show that the container is running detached.
docker container ps
```

当你附加到运行的容器时，它的执行命令将成为你的终端会话的前台进程。要从容器中分离，你需要发出*Ctrl* + *PQ*按键。如果你发出*Ctrl* + *C*按键，容器的执行进程将接收到 sig-term 信号并终止，这将导致容器退出。这通常是不希望的。所以记住要使用*Ctrl* + *PQ*按键来分离。

然而，在 macOS 上存在一个已知问题：对于 Mac 上的 Docker，*Ctrl* + *PQ*按键组合不起作用，除非你在`attach`命令上使用另一个参数，`--sig-proxy=false`参数，否则你将无法在不使用*Ctrl *+ *C*按键的情况下从容器中分离出来：

```
# when you are using Docker for Mac, remember to always add the "--sig-proxy=false" parameter
docker attach --sig-proxy=false web-server1
```

当你向`attach`命令提供`--sig-proxy=false`参数时，你可以向附加的容器发出*Ctrl *+ *C*按键，它将分离而不向容器进程发送 sig-term 信号，从而使容器再次以分离状态运行，脱离你的终端会话：

![](img/fcf8b70a-0e46-4ad1-a260-d783fadf9ca6.png)

# 容器 exec 命令

有时，当你有一个以分离状态运行的容器时，你可能想要访问它，但不想附加到执行命令。你可以通过使用容器 exec 命令来实现这一点。这个命令允许你在运行的容器中执行另一个命令，而不附加或干扰已经运行的命令。这个命令经常用于创建与已经运行的容器的交互会话，或者在容器内执行单个命令。命令看起来像这样：

```
# start an nginx container detached
docker container run --detach --name web-server1 -p 80:80 nginx

# see that the container is currently running
docker container ls

# execute other commands in the running container
# Usage: docker container exec [OPTIONS] CONTAINER COMMAND [ARG...] docker container exec -it web-server1 bash
docker container exec web-server1 cat /etc/debian_version

# confirm that the container is still running 
docker container ls
```

当`exec`命令完成时，你退出 bash shell，或者文件内容已经被替换，然后它会退出到终端会话，让容器以分离状态运行：

![](img/fb344e4d-5ba7-440f-af71-9b42c8c4ccce.png)

让我们在继续讨论许多可选的`container run`参数之前，先看看另一个 Docker 命令。

# 容器 commit 命令

重要的是要知道，当您连接到正在运行的容器并对其进行更改，比如安装新的软件包或更改配置文件时，这些更改只适用于该正在运行的容器。例如，如果您使用 Ubuntu 镜像运行一个容器，然后在该容器中安装`curl`，那么这个更改不会应用到您从中运行容器的镜像，例如 Ubuntu。如果您要从相同的 Ubuntu 镜像启动另一个容器，您需要再次安装`curl`。但是，如果您希望在运行新容器时保留并使用在运行容器内进行的更改，您可以使用`container commit`命令。`container commit`命令允许您保存容器的当前读/写层以及原始镜像的层，从而创建一个全新的镜像。当您使用新镜像运行容器时，它将包括您使用`container commit`命令保存的更改。`container commit`命令的样子如下：

```
# Usage: docker container commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
docker container commit ubuntu new-ubuntu
```

这里有一个使用`container commit`命令将`curl`安装到正在运行的容器中，并创建一个包含安装的`curl`命令的新容器的示例：

![](img/47f18f01-68e7-42e5-80de-4dacdf2aad16.png)

有了这个例子，我现在可以从`ubuntu-curl`镜像运行新的容器，它们都将已经安装了`curl`命令。

# 回到 Docker 运行命令

现在，让我们回到讨论`container run`命令。之前，您看到了使用`run`命令和`--publish`参数的示例。使用可选的发布参数允许您指定与运行容器相关的将要打开的端口。`--publish`参数包括用冒号分隔的端口号对。例如：

```
# create an nginx web-server that redirects host traffic from port 8080 to port 80 in the container
docker container run --detach --name web-server1 --publish 8080:80 nginx
```

第一个端口号与运行容器的主机相关联。在 nginx 示例中，`8080`在主机上暴露；在我们的情况下，那将是`http://localhost:8080`。第二个端口号是运行容器上打开的端口。在这种情况下，它将是`80`。描述`--publish 8080:80`参数时，您可以说类似于，发送到主机上端口`8080`的流量被重定向到运行容器上的端口`80`：

![](img/3309bf24-df9f-47bf-b97c-9bc3112928fb.png)

重要的区别在于主机端口和容器端口。我可以在同一系统上运行多个暴露端口`80`的容器，但是每个端口在主机上只能有一个容器的流量。看下面的例子更好地理解：

```
# all of these can be running at the same time
docker container run --detach --name web-server1 --publish 80:80 nginx
docker container run --detach --name web-server2 --publish 8000:80 nginx
docker container run --detach --name web-server3 --publish 8080:80 nginx
docker container run --detach --name web-server4 --publish 8888:80 nginx # however if you tried to run this one too, it would fail to run 
# because the host already has port 80 assigned to web-server1
docker container run --detach --name web-server5 --publish 80:80 nginx
```

要知道这是网络的一般限制，而不是 Docker 或容器的限制。在这里我们可以看到这些命令及其输出。注意端口和名称，以及已经使用的端口作为端点的使用失败：

![](img/60594bf1-705f-48cf-9ecd-60d4691320be.png)

这是关于`container run`命令的各种选项参数的大量数据。这并不是所有的选项参数，但应该足够让你有一个很好的开始。如果你想了解更多我们探讨的可选参数，或者找出我们没有涵盖的内容，一定要访问 docker 文档页面上的`container run`命令，网址是[`docs.docker.com/engine/reference/run/`](https://docs.docker.com/engine/reference/run/)。

# 总结

在本章中，我们学习了关于 Docker 镜像描述和 Docker 注册表的知识。然后我们看到了版本命令的另一种形式。之后，我们探索了许多 Docker 容器命令，包括`run`、`stop`、`ls`、`logs`、`top`、`stats`、`attach`、`exec`和`commit`命令。最后，我们了解了如何通过从主机到容器打开端口来暴露您的容器。你应该对 Docker 已经能做的事情感到很满意，但是请稍等，在第三章 *创建 Docker 镜像*中，我们将向您展示如何使用`Dockerfile`和镜像构建命令创建自己的 Docker 镜像。如果你准备好了，翻页吧。

# 参考

+   Docker 注册表：[`hub.docker.com/explore/`](https://hub.docker.com/explore/)

+   所有`container run`命令的参数：[`docs.docker.com/engine/reference/run/`](https://docs.docker.com/engine/reference/run/)

+   使用`--format`参数与容器检查命令：[`docs.docker.com/config/formatting`](https://docs.docker.com/config/formatting)

+   json jq 解析器：[`stedolan.github.io/jq/`](https://stedolan.github.io/jq/)

+   Chocolatey Windows 软件包管理器：[`chocolatey.org/`](https://chocolatey.org/)

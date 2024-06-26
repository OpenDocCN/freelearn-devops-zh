# 第十二章：最佳实践

概述

在本章中，您将学习一些在使用 Docker 和容器镜像时的最佳实践，这将使您能够监视和管理容器使用的资源，并限制其对主机系统的影响。您将分析 Docker 的最佳实践，并了解为什么重要的是只在一个容器中运行一个服务，确保您的容器是可扩展的和不可变的，并确保您的基础应用程序在短时间内启动。本章将通过使用 `hadolint` 的 `FROM:latest` 命令和 `dcvalidator` 在应用程序和容器运行之前对您的 `Dockerfiles` 和 `docker-compose.yml` 文件进行检查，以帮助您强制执行这些最佳实践。

# 介绍

安全的前一章涵盖了一些 Docker 镜像和服务的最佳实践，这些实践已经遵循了这些最佳实践。我们确保我们的镜像和服务是安全的，并且限制了如果攻击者能够访问镜像时可以实现的内容。本章不仅将带您了解创建和运行 Docker 镜像的最佳实践，还将关注容器性能、配置我们的服务，并确保运行在其中的服务尽可能高效地运行。

我们将从深入了解如何监视和配置服务使用的资源开始，比如内存和 CPU 使用情况。然后，我们将带您了解一些您可以在项目中实施的重要实践，看看您如何创建 Docker 镜像以及在其上运行的应用程序。最后，本章将为您提供一些实用工具，用于测试您的 `Dockerfiles` 和 `docker-compose.yml` 文件，这将作为一种确保您遵循所述实践的方式。

本章展示了如何确保尽可能优化您的服务和容器，以确保它们从开发环境到生产环境都能无故障地运行。本章的目标是确保您的服务尽快启动，并尽可能高效地处理。本章提到的实践还确保了可重用性（也就是说，他们确保任何想要重用您的镜像或代码的人都可以这样做，并且可以随时了解具体发生了什么）。首先，以下部分讨论了如何使用容器资源。

# 使用容器资源

从传统服务器环境迁移到 Docker 的主要好处之一是，即使在转移到生产环境时，它使我们能够大大减少服务和应用程序的占用空间。然而，这并不意味着我们可以简单地在容器上运行任何东西，期望所有进程都能顺利完成执行。就像在独立服务器上运行服务时一样，我们需要确保我们的容器使用的资源（如 CPU、内存和磁盘输入输出）不会导致我们的生产环境或任何其他容器崩溃。通过监控开发系统中使用的资源，我们可以帮助优化流程，并确保最终用户在将其移入生产环境时体验到无缝操作。

通过测试我们的服务并监控资源使用情况，我们将能够了解运行应用程序所需的资源，并确保运行我们 Docker 镜像的主机具有足够的资源来运行我们的服务。最后，正如您将在接下来的章节中看到的，我们还可以限制容器可以访问的 CPU 和内存资源的数量。在开发运行在 Docker 上的服务时，我们需要在开发系统上测试这些服务，以确切了解它们在移入测试和生产环境时会发生什么。

当我们将多种不同的服务（如数据库、Web 服务器和 API 网关）组合在一起创建一个应用程序时，有些服务比其他服务更重要，在某些情况下，这些服务可能需要分配更多资源。然而，在 Docker 中，运行的容器默认情况下并没有真正的资源限制。

在之前的章节中，我们学习了使用 Swarm 和 Kubernetes 进行编排，这有助于在系统中分配资源，但本章的这一部分将教您一些基本工具来测试和监视您的资源。我们还将看看您可以如何配置您的容器，以不再使用默认可用的资源。

为了帮助我们在本章的这一部分，我们将创建一个新的镜像，该镜像将仅用于演示我们系统中的资源使用情况。在本节的第一部分中，我们将创建一个将添加一个名为 stress 的应用程序的镜像。stress 应用程序的主要功能是对我们的系统施加重负载。该镜像将允许我们查看在我们的主机系统上使用的资源，然后允许我们在运行 Docker 镜像时使用不同的选项来限制使用的资源。

注意

本章的这一部分将为您提供有关监视我们正在运行的 Docker 容器资源的简要指南。本章将仅涵盖一些简单的概念，因为我们将在本书的另一章节中专门提供有关监视容器指标的深入细节。

为了帮助我们查看正在运行的容器消耗的资源，Docker 提供了`stats`命令，作为我们正在运行的容器消耗资源的实时流。如果您希望限制流所呈现的数据，特别是如果您有大量正在运行的容器，您可以通过指定容器的名称或其 ID 来指定只提供某些容器：

```
docker stats <container_name|container_id>
```

`docker` `stats`命令的默认输出将为您提供容器的名称和 ID，容器正在使用的主机 CPU 和内存的百分比，容器正在发送和接收的数据，以及从主机存储中读取和写入的数据量：

```
NAME                CONTAINER           CPU %
docker-stress       c8cf5ad9b6eb        400.43%
```

以下部分将重点介绍如何使用`docker stats`命令来监视我们的资源。我们还将向`stats`命令提供格式控制，以提供我们需要的信息。

# 管理容器 CPU 资源

本章的这一部分将向您展示如何设置容器使用的 CPU 数量限制，因为没有限制的容器可能会占用主机服务器上所有可用的 CPU 资源。我们将着眼于优化我们正在运行的 Docker 容器，但实际上大量使用 CPU 的问题通常出现在基础设施或容器中运行的应用程序上。

当我们讨论 CPU 资源时，通常是指单个物理计算机芯片。如今，CPU 很可能有多个核心，更多的核心意味着更多的进程。但这并不意味着我们拥有无限的资源。当我们显示正在使用的 CPU 百分比时，除非您的系统只有一个 CPU 和一个核心，否则您很可能会看到超过 100%的 CPU 使用率。例如，如果您的系统的 CPU 中有四个核心，而您的容器正在利用所有的 CPU，您将看到 400%的值。

我们可以修改在我们的系统上运行的`docker stats`命令，通过提供`--format`选项来仅提供 CPU 使用情况的详细信息。这个选项允许我们指定我们需要的输出格式，因为我们可能只需要`stats`命令提供的一两个指标。以下示例配置了`stats`命令的输出以以`table`格式显示，只呈现容器的名称、ID 和正在使用的 CPU 百分比：

```
docker stats --format "table {{.Name}}\t{{.Container}}\t{{.CPUPerc}}"
```

如果我们没有运行 Docker 镜像，这个命令将提供一个包含以下三列的表格：

```
NAME                CONTAINER           CPU %
```

为了控制我们正在运行的容器使用的 CPU 核心数量，我们可以在`docker run`命令中使用`--cpus`选项。以下语法向我们展示了运行镜像，但通过使用`--cpus`选项限制了镜像可以访问的核心数量：

```
docker run --cpus 2 <docker-image>
```

更好的选择不是设置容器可以使用的核心数量，而是设置它可以共享的总量。Docker 提供了`--cpushares`或`-c`选项来设置容器可以使用的处理能力的优先级。通过使用这个选项，这意味着在运行容器之前我们不需要知道主机机器有多少个核心。这也意味着我们可以将正在运行的容器转移到不同的主机系统，而不需要更改运行镜像的命令。

默认情况下，Docker 将为每个运行的容器分配 1,024 份份额。如果您将`--cpushares`值设置为`256`，它将拥有其他运行容器的四分之一的处理份额：

```
docker run --cpushares 256 <docker-image>
```

注意

如果系统上没有运行其他容器，即使您已将`--cpushares`值设置为`256`，容器也将被允许使用剩余的处理能力。

即使您的应用程序可能正在正常运行，查看减少其可用 CPU 量以及在正常运行时消耗多少的做法总是一个好习惯。

在下一个练习中，我们将使用`stress`应用程序来监视系统上的资源使用情况。

注意

请使用`touch`命令创建文件，并使用`vim`命令使用 vim 编辑器处理文件。

## 练习 12.01：了解 Docker 镜像上的 CPU 资源

在这个练习中，您将首先创建一个新的 Docker 镜像，这将帮助您在系统上生成一些资源。我们将演示如何在镜像上使用已安装的`stress`应用程序。该应用程序将允许您开始监视系统上的资源使用情况，以及允许您更改镜像使用的 CPU 资源数量：

1.  创建一个新的`Dockerfile`并打开您喜欢的文本编辑器输入以下细节。您将使用 Ubuntu 作为基础来创建镜像，因为`stress`应用程序尚未作为易于在 Alpine 基础镜像上安装的软件包提供：

```
FROM ubuntu
RUN apt-get update && apt-get install stress
CMD stress $var
```

1.  使用`docker build`命令的`-t`选项构建新镜像并将其标记为`docker-stress`：

```
docker build -t docker-stress .
```

1.  在运行新的`docker-stress`镜像之前，请先停止并删除所有其他容器，以确保结果不会被系统上运行的其他容器混淆：

```
docker rm -f $(docker -a -q)
```

1.  在`Dockerfile`的*第 3 行*上，您会注意到`CMD`指令正在运行 stress 应用程序，后面跟着`$var`变量。这将允许您通过环境变量直接向容器上运行的 stress 应用程序添加命令行选项，而无需每次想要更改功能时都构建新镜像。通过运行您的镜像并使用`-e`选项添加环境变量来测试这一点。将`var="--cpu 4 --timeout 20"`作为`stress`命令的命令行选项添加：

```
docker run --rm -it -e var="--cpu 4 --timeout 20" docker-stress
```

`docker run`命令已添加了`var="--cpu 4 --timeout 20"`变量，这将特别使用这些命令行选项运行`stress`命令。`--cpu`选项表示将使用系统的四个 CPU 或核心，`--timeout`选项将允许压力测试运行指定的秒数 - 在本例中为`20`：

```
stress: info: [6] dispatching hogs: 4 cpu, 0 io, 0 vm, 0 hdd
stress: info: [6] successful run completed in 20s
```

注意

如果我们需要连续运行`stress`命令而不停止，我们将简单地不包括`--timeout`选项。我们的示例都包括`timeout`选项，因为我们不想忘记并持续使用运行主机系统的资源。

1.  运行`docker stats`命令，查看这对主机系统的影响。使用`--format`选项限制所提供的输出，只提供 CPU 使用情况：

```
docker stats --format "table {{.Name}}\t{{.Container}}\t{{.CPUPerc}}"
```

除非您的系统上运行着一个容器，否则您应该只看到表头，类似于此处提供的输出：

```
NAME                CONTAINER           CPU %
```

1.  在运行`stats`命令的同时，进入一个新的终端窗口，并再次运行`docker-stress`容器，就像本练习的*步骤 4*中一样。使用`--name`选项确保在使用`docker stress`命令时查看正确的镜像：

```
docker run --rm -it -e var="--cpu 4 --timeout 20" --name docker-stress docker-stress
```

1.  返回到运行`docker stats`的终端。现在您应该看到一些输出呈现在您的表上。您的输出将与以下内容不同，因为您的系统上可能运行着不同数量的核心。以下输出显示我们的 CPU 百分比使用了 400%。运行该命令的系统有六个核心。它显示 stress 应用程序正在使用四个可用核心中的 100%：

```
NAME                CONTAINER           CPU %
docker-stress       c8cf5ad9b6eb        400.43%
```

1.  再次运行`docker-stress`容器，这次将`--cpu`选项设置为`8`：

```
docker run --rm -it -e var="--cpu 8 --timeout 20" --name docker-stress docker-stress
```

如您在以下统计输出中所见，我们已经达到了 Docker 容器几乎使用系统上所有六个核心的极限，为我们的系统上的次要进程留下了一小部分处理能力：

```
NAME                CONTAINER           CPU %
docker-stress       8946da6ffa90        599.44%
```

1.  通过使用`--cpus`选项并指定要允许镜像使用的核心数量，来管理您的`docker-stress`镜像可以访问的核心数量。在以下命令中，将`2`设置为我们的容器被允许使用的核心数量：

```
docker run --rm -it -e var="--cpu 8 --timeout 20" --cpus 2 --name docker-stress docker-stress
```

1.  返回到运行`docker stats`的终端。您将看到正在使用的 CPU 百分比不会超过 200%，显示 Docker 将资源使用限制在我们系统上仅有的两个核心：

```
NAME                CONTAINER           CPU %
docker-stress       79b32c67cbe3        208.91%
```

到目前为止，您只能一次在我们的系统上运行一个容器。这个练习的下一部分将允许您以分离模式运行两个容器。在这里，您将测试在运行的一个容器上使用`--cpu-shares`选项来限制它可以使用的核心数量。

1.  如果您没有在终端窗口中运行`docker stats`，请像之前一样启动它，以便我们监视正在运行的进程：

```
docker stats --format "table {{.Name}}\t{{.Container}}\t{{.CPUPerc}}"
```

1.  访问另一个终端窗口，并启动两个`docker-stress`容器 - `docker-stress1`和`docker-stress2`。第一个将使用`--timeout`值为`60`，让压力应用程序运行 60 秒，但在这里，将`--cpu-shares`值限制为`512`：

```
docker run --rm -dit -e var="--cpu 8 --timeout 60" --cpu-shares 512 --name docker-stress1 docker-stress
```

容器的 ID 将返回如下：

```
5f617e5abebabcbc4250380b2591c692a30b3daf481b6c8d7ab8a0d1840d395f
```

第二个容器将不受限制，但`--timeout`值只有`30`，所以它应该先完成：

```
docker run --rm -dit -e var="--cpu 8 --timeout 30" --name docker-stress2 docker-stress2
```

容器的 ID 将返回如下：

```
83712c28866dd289937a9c5fe4ea6c48a6863a7930ff663f3c251145e2fbb97a
```

1.  回到运行`docker stats`的终端。您会看到两个容器正在运行。在以下输出中，我们可以看到名为`docker-stress1`和`docker-stress2`的容器。`docker-stress1`容器被设置为只有`512` CPU 份额，而其他容器正在运行。还可以观察到它只使用了第二个名为`docker-stress2`的容器的一半 CPU 资源：

```
NAME                CONTAINER           CPU %
docker-stress1      5f617e5abeba        190.25%
docker-stress2      83712c28866d        401.49%
```

1.  当第二个容器完成后，`docker-stress1`容器的 CPU 百分比将被允许使用运行系统上几乎所有六个可用的核心：

```
NAME                CONTAINER           CPU %
stoic_keldysh       5f617e5abeba        598.66%
```

CPU 资源在确保应用程序以最佳状态运行方面起着重要作用。这个练习向您展示了在将容器部署到生产环境之前，监视和配置容器的处理能力有多么容易。接下来的部分将继续对容器的内存执行类似的监视和配置更改。

# 管理容器内存资源

就像我们可以监视和控制容器在系统上使用的 CPU 资源一样，我们也可以对内存的使用情况进行相同的操作。与 CPU 一样，默认情况下，运行的容器可以使用主机的所有内存，并且在某些情况下，如果没有限制，可能会导致系统变得不稳定。如果主机系统内核检测到没有足够的内存可用，它将显示**内存不足异常**并开始终止系统上的进程以释放内存。

好消息是，Docker 守护程序在您的系统上具有高优先级，因此内核将首先终止运行的容器，然后才会停止 Docker 守护程序的运行。这意味着如果高内存使用是由容器应用程序引起的，您的系统应该能够恢复。

注意

如果您的运行容器正在被关闭，您还需要确保已经测试了您的应用程序，以确保它对正在运行的进程的影响是有限的。

再次强调，`docker stats`命令为我们提供了关于内存使用情况的大量信息。它将输出容器正在使用的内存百分比，以及当前内存使用量与其能够使用的总内存量的比较。与之前一样，我们可以通过`--format`选项限制所呈现的输出。在以下命令中，我们通过`.Name`、`.Container`、`.MemPerc`和`.MemUsage`属性，仅显示容器名称和 ID，以及内存百分比和内存使用量：

```
docker stats --format "table {{.Name}}\t{{.Container}}\t{{.MemPerc}}\t{{.MemUsage}}"
```

没有运行的容器，上述命令将显示以下输出：

```
NAME         CONTAINER          MEM %         MEM USAGE / LIMIT
```

如果我们想要限制或控制运行容器使用的内存量，我们有一些选项可供选择。其中一个可用的选项是`--memory`或`-m`选项，它将设置运行容器可以使用的内存量的限制。在以下示例中，我们使用了`--memory 512MB`的语法来限制可用于镜像的内存量为`512MB`：

```
docker run --memory 512MB <docker-image>
```

如果容器正在运行的主机系统也在使用交换空间作为可用内存的一部分，您还可以将内存从该容器分配为交换空间。这只需使用`--memory-swap`选项即可。这只能与`--memory`选项一起使用，正如我们在以下示例中所演示的。我们已将`--memory-swap`选项设置为`1024MB`，这是容器可用内存的总量，包括内存和交换内存。因此，在我们的示例中，交换空间中将有额外的`512MB`可用：

```
docker run --memory 512MB --memory-swap 1024MB <docker-image>
```

但需要记住，交换内存将被分配到磁盘，因此会比 RAM 更慢、响应更慢。

注意

`--memory-swap`选项需要设置为高于`--memory`选项的数字。如果设置为相同的数字，您将无法为运行的容器分配任何内存到交换空间。

另一个可用的选项，只有在需要确保运行容器始终可用时才能使用的是`--oom-kill-disable`选项。此选项会阻止内核在主机系统内存过低时杀死运行的容器。这应该只与`--memory`选项一起使用，以确保您设置了容器可用内存的限制。如果没有限制，`--oom-kill-disable`选项很容易使用主机系统上的所有内存：

```
docker run --memory 512MB --oom-kill-disable <docker-image>
```

尽管您的应用程序设计良好，但前面的配置为您提供了一些选项来控制运行容器使用的内存量。

下一节将为您提供在分析 Docker 镜像上的内存资源方面的实践经验。

## 练习 12.02：分析 Docker 镜像上的内存资源

这项练习将帮助您分析在主机系统上运行时活动容器如何使用内存。再次使用之前创建的`docker-stress`镜像，但这次使用选项仅在运行容器上使用内存。这个命令将允许我们实现一些可用的内存限制选项，以确保我们运行的容器不会使主机系统崩溃：

1.  运行`docker stats`命令以显示所需的百分比内存和内存使用值的相关信息：

```
docker stats --format "table {{.Name}}\t{{.Container}}\t{{.MemPerc}}\t{{.MemUsage}}"
```

这个命令将提供以下类似的输出：

```
NAME        CONTAINER       MEM %         MEM USAGE / LIMIT
```

1.  打开一个新的终端窗口再次运行`stress`命令。你的`docker-stress`镜像只有在使用`--cpu`选项时才会利用 CPU。使用以下命令中的`--vm`选项来启动你希望产生的工作进程数量以消耗内存。默认情况下，每个工作进程将消耗`256MB`：

```
docker run --rm -it -e var="--vm 2 --timeout 20" --name docker-stress docker-stress
```

当你返回监视正在运行的容器时，内存使用量只达到了限制的 20%左右。这可能因不同系统而异。由于只有两个工作进程在运行，每个消耗 256MB，你应该只会看到内存使用量达到大约 500MB：

```
NAME            CONTAINER      MEM %      MEM USAGE / LIMIT
docker-stress   b8af08e4d79d   20.89%     415.4MiB / 1.943GiB
```

1.  压力应用程序还有`--vm-bytes`选项来控制每个被产生的工作进程将消耗的字节数。输入以下命令，将每个工作进程设置为`128MB`。当你监视它时，它应该显示较低的使用量：

```
docker run --rm -it -e var="--vm 2 --vm-bytes 128MB --timeout 20" --name stocker-stress docker-stress
```

正如你所看到的，压力应用程序在推动内存使用量时并没有取得很大的进展。如果你想要使用系统上可用的全部 8GB RAM，你可以使用`--vm 8 --vm-bytes` 1,024 MB：

```
NAME            CONTAINER      MEM %    MEM USAGE / LIMIT
docker-stress   ad7630ed97b0   0.04%    904KiB / 1.943GiB
```

1.  使用`--memory`选项减少`docker-stress`镜像可用的内存。在以下命令中，你会看到我们将正在运行的容器的可用内存限制为`512MB`：

```
docker run --rm -it -e var="--vm 2 --timeout 20" --memory 512MB --name docker-stress docker-stress
```

1.  返回到运行`docker stats`的终端，你会看到内存使用率飙升到了接近 100%。这并不是一件坏事，因为它只是你正在运行的容器分配的一小部分内存。在这种情况下，它是 512MB，仅为之前的四分之一：

```
NAME            CONTAINER      MEM %     MEM USAGE / LIMIT
docker-stress   bd84cf27e480   88.11%    451.1MiB / 512MiB
```

1.  同时运行多个容器，看看我们的`stats`命令如何响应。在`docker run`命令中使用`-d`选项将容器作为守护进程在主机系统的后台运行。现在，两个`docker-stress`容器都将使用六个工作进程，但我们的第一个镜像，我们将其命名为`docker-stress1`，被限制在`512MB`的内存上，而我们的第二个镜像，名为`docker-stress2`，只运行 20 秒，将拥有无限的内存：

```
docker run --rm -dit -e var="--vm 6 --timeout 60" --memory 512MB --name docker-stress1 docker-stress
ca05e244d03009531a6a67045a5b1edbef09778737cab2aec7fa92eeaaa0c487
docker run --rm -dit -e var="--vm 6 --timeout 20" --name docker-stress2 docker-stress
6d9cbb966b776bb162a47f5e5ff3d88daee9b0304daa668fca5ff7ae1ee887ea
```

1.  返回到运行`docker stats`的终端。你会看到只有一个容器，即`docker-stress1`容器，被限制在 512MB，而`docker-stress2`镜像被允许在更多的内存上运行：

```
NAME             CONTAINER       MEM %    MEM USAGE / LIMIT
docker-stress1   ca05e244d030    37.10%   190MiB / 512MiB
docker-stress2   6d9cbb966b77    31.03%   617.3MiB / 1.943GiB
```

如果你等待一会儿，`docker-stress1`镜像将被留下来独自运行：

```
NAME             CONTAINER      MEM %    MEM USAGE / LIMIT
docker-stress1   ca05e244d030   16.17%   82.77MiB / 512MiB
```

注意

我们在这里没有涵盖的一个选项是`--memory-reservation`选项。这也与`--memory`选项一起使用，并且需要设置为低于内存选项。这是一个软限制，当主机系统的内存不足时激活，但不能保证限制将被执行。

本章的这一部分帮助我们确定如何运行容器并监视使用情况，以便在将它们投入生产时，它们不会通过使用所有可用内存来停止主机系统。现在，您应该能够确定您的镜像正在使用多少内存，并在长时间运行或内存密集型进程出现问题时限制可用内存量。在下一节中，我们将看看我们的容器如何在主机系统磁盘上消耗设备的读写资源。

# 管理容器磁盘的读写资源

运行容器消耗的 CPU 和内存通常是环境运行不佳的最大罪魁祸首，但您的运行容器也可能存在问题，尝试读取或写入主机的磁盘驱动器过多。这很可能对 CPU 或内存问题影响较小，但如果大量数据被传输到主机系统的驱动器上，仍可能引起争用并减慢服务速度。

幸运的是，Docker 还为我们提供了一种控制运行容器执行读取和写入操作的方法。就像我们之前看到的那样，我们可以在`docker run`命令中使用多个选项来限制我们要读取或写入设备磁盘的数据量。

`docker stats`命令还允许我们查看传输到和从我们的运行容器的数据。它有一个专用列，可以使用`docker stats`命令中的`BlockIO`值将其添加到我们的表中，该值代表对我们的主机磁盘驱动器或目录的读写操作：

```
docker stats --format "table {{.Name}}\t{{.Container}}\t{{.BlockIO}}"
```

如果我们的系统上没有任何运行的容器，上述命令应该为我们提供以下输出：

```
NAME                CONTAINER           BLOCK I/O
```

如果我们需要限制正在运行的容器可以移动到主机系统磁盘存储的数据量，我们可以从使用`--blkio-weight`选项开始，该选项与我们的`docker run`命令一起使用。此选项代表**块输入输出权重**，允许我们为容器设置一个相对权重，介于`10`和`1000`之间，并且相对于系统上运行的所有其他容器。所有容器将被设置为相同比例的带宽，即 500。如果为任何容器提供值 0，则此选项将被关闭。

```
docker run --blkio-weight <value> <docker-image>
```

我们可以使用的下一个选项是`--device-write-bps`，它将限制指定的设备可用的特定写入带宽，以字节每秒的值为单位。特定设备是相对于容器在主机系统上使用的设备。此选项还有一个“每秒输入/输出（IOPS）”选项，也可以使用。以下语法提供了该选项的基本用法，其中限制值设置为 MB 的数值：

```
docker run --device-write-bps <device>:<limit> <docker-image>
```

就像有一种方法可以限制写入进程到主机系统的磁盘一样，也有一种选项可以限制可用的读取吞吐量。同样，它还有一个“每秒输入/输出（IOPS）”选项，可以用来限制可以从正在运行的容器中读取的数据量。以下示例使用`--device-read-bps`选项作为`docker run`命令的一部分：

```
docker run --device-read-bps <device>:<limit> <docker-image>
```

如果您遵守容器最佳实践，磁盘输入或输出的过度消耗不应该是太大的问题。尽管如此，没有理由认为这不会给您造成任何问题。就像您已经处理过 CPU 和内存一样，您的磁盘输入和输出应该在将服务实施到生产环境之前在运行的容器上进行测试。

## 练习 12.03：理解磁盘读写

这个练习将使您熟悉查看正在运行的容器的磁盘读写。它将允许您通过在运行时使用可用的选项来配置磁盘使用速度的限制来开始运行您的容器：

1.  打开一个新的终端窗口并运行以下命令：

```
docker stats --format "table {{.Name}}\t{{.Container}}\t{{.BlockIO}}" 
```

`docker stats`命令与`BlockIO`选项帮助我们监视从我们的容器到主机系统磁盘的输入和输出级别。

1.  启动容器以从 bash 命令行访问它。在运行的`docker-stress`镜像上直接执行一些测试。stress 应用程序确实为您提供了一些选项，以操纵容器和主机系统上的磁盘利用率，但它仅限于磁盘写入：

```
docker run -it --rm --name docker-stress docker-stress /bin/bash
```

1.  与 CPU 和内存使用情况不同，块输入和输出显示容器使用的总量，因此它不会随着运行容器执行更多更改而动态变化。回到运行`docker stats`的终端。您应该看到输入和输出都为`0B`：

```
NAME                CONTAINER           BLOCK I/O
docker-stress       0b52a034f814        0B / 0B
```

1.  在这种情况下，您将使用 bash shell，因为它可以访问`time`命令以查看每个进程需要多长时间。使用`dd`命令，这是一个用于复制文件系统和备份的 Unix 命令。在以下选项中，使用`if`（输入文件）选项创建我们的`/dev/zero`目录的副本，并使用`of`（输出文件）选项将其输出到`disk.out`文件。`bs`选项是块大小或应该一次读取的数据量，`count`是要读取的总块数。最后，将`oflag`值设置为`direct`，这意味着复制将避免缓冲区缓存，因此您将看到磁盘读取和写入的真实值：

```
time dd if=/dev/zero of=disk.out bs=1M count=10 oflag=direct
10+0 records in
10+0 records out
10485760 bytes (10 MB, 10 MiB) copied, 0.0087094 s, 1.2 GB/s
real    0m0.010s
user    0m0.000s
sys     0m0.007s
```

1.  回到运行`docker stats`命令的终端。您将看到超过 10MB 的数据发送到主机系统的磁盘。与 CPU 和内存不同，传输完成后，您不会看到此数据值下降：

```
NAME                CONTAINER           BLOCK I/O
docker-stress       0b52a034f814        0B / 10.5MB
```

您还会注意到*步骤 4*中的命令几乎立即完成，`time`命令显示实际只需`0.01s`即可完成。您将看到如果限制可以写入磁盘的数据量会发生什么，但首先退出运行的容器，以便它不再存在于我们的系统中。

1.  要再次启动我们的`docker-stress`容器，请将`--device-write-bps`选项设置为每秒`1MB`在`/dev/sda`设备驱动器上：

```
docker run -it --rm --device-write-bps /dev/sda:1mb --name docker-stress docker-stress /bin/bash
```

1.  再次运行`dd`命令，之前加上`time`命令，以测试需要多长时间。您会看到该命令花费的时间比*步骤 4*中的时间长得多。`dd`命令再次设置为复制`1MB`块，`10`次：

```
time dd if=/dev/zero of=test.out bs=1M count=10 oflag=direct
```

因为容器限制为每秒只能写入 1MB，所以该命令需要 10 秒，如下面的输出所示：

```
10+0 records in
10+0 records out
10485760 bytes (10 MB, 10 MiB) copied, 10.0043 s, 1.0 MB/s
real    0m10.006s
user    0m0.000s
sys     0m0.004s
```

我们已经能够很容易地看到我们的运行容器如何影响底层主机系统，特别是在使用磁盘读写时。我们还能够看到我们如何轻松地限制可以写入设备的数据量，以便在运行容器之间减少争用。在下一节中，我们将快速回答一个问题，即如果您正在使用`docker-compose`，您需要做什么，并且限制容器使用的资源数量。

# 容器资源和 Docker Compose

诸如 Kubernetes 和 Swarm 之类的编排器在控制和运行资源以及在需要额外资源时启动新主机方面发挥了重要作用。但是，如果您在系统或测试环境中运行`docker-compose`，您该怎么办呢？幸运的是，前面提到的资源配置也适用于`docker-compose`。

在我们的`docker-compose.yml`文件中，在我们的服务下，我们可以在`deploy`配置下使用`resources`选项，并为我们的服务指定资源限制。就像我们一直在使用`--cpus`、`--cpu_shares`和`--memory`等选项一样，我们在我们的`docker-compose.yml`文件中也会使用相同的选项，如`cpus`、`cpu_shares`和`memory`。

以下代码块中的示例`compose`文件部署了我们在本章中一直在使用的`docker-stress`镜像。如果我们看*第 8 行*，我们可以看到`deploy`语句，后面是`resources`语句。这是我们可以为我们的容器设置限制的地方。就像我们在前面的部分中所做的那样，我们在*第 11 行*上将`cpus`设置为`2`，在*第 12 行*上将`memory`设置为`256MB`。 

```
1 version: '3'
2 services:
3   app:
4     container_name: docker-stress
5     build: .
6     environment:
7       var: "--cpu 2 --vm 6 --timeout 20"
8     deploy:
9       resources:
10         limits:
11           cpus: '2'
12           memory: 256M
```

尽管我们只是简单地涉及了这个主题，但前面涵盖资源使用的部分应该指导您如何在`docker-compose.yml`文件中分配资源。这就是我们关于 Docker 容器资源使用的部分的结束。从这里开始，我们将继续研究创建我们的`Dockerfiles`的最佳实践，以及如何开始使用不同的应用程序来确保我们遵守这些最佳实践。

# Docker 最佳实践

随着我们的容器和服务规模和复杂性的增长，重要的是要确保在创建 Docker 镜像时我们遵循最佳实践。对于我们在 Docker 镜像上运行的应用程序也是如此。在本章的后面，我们将查看我们的`Dockerfiles`和`docker-compose.yml`文件，这将分析我们的文件中的错误和最佳实践，从而让您更清楚地了解。与此同时，让我们来看看在创建 Docker 镜像和应用程序与之配合工作时需要牢记的一些更重要的最佳实践。

注意

本章可能涵盖了一些之前章节的内容，但我们将能够为您提供更多信息和清晰解释为什么我们要使用这些实践。

在接下来的部分，我们将介绍一些在创建服务和容器时应该遵循的常见最佳实践。

## 每个容器只运行一个服务

在现代微服务架构中，我们需要记住每个容器只应安装一个服务。容器的主要进程由`Dockerfile`末尾的`ENTRYPOINT`或`CMD`指令设置。

您在容器中安装的服务很容易运行多个进程，但为了充分利用 Docker 和微服务的优势，您应该每个容器只运行一个服务。更进一步地说，您的容器应该只负责一个单一的功能，如果它负责的事情超过一个，那么它应该拆分成不同的服务。

通过限制每个容器的功能，我们有效地减少了镜像使用的资源，并可能减小了镜像的大小。正如我们在上一章中看到的，这也将减少攻击者在获得运行中的容器访问权限时能够执行任何不应该执行的操作的机会。这也意味着，如果容器因某种原因停止工作，对环境中运行的其他应用程序的影响有限，服务将更容易恢复。

## 基础镜像

当我们为我们的容器选择基础镜像时，我们需要做的第一件事之一是确保我们使用的是最新的镜像。还要做一些研究，确保您不使用安装了许多不需要的额外应用程序的镜像。您可能会发现，受特定语言支持的基础镜像或特定焦点将限制所需的镜像大小，从而限制您在创建镜像时需要安装的内容。

这就是为什么我们使用受 PostgreSQL 支持的 Docker 镜像，而不是在构建时在镜像上安装应用程序。受 PostgreSQL 支持的镜像确保它是安全的，并且运行在最新版本，并确保我们不在镜像上运行不需要的应用程序。

在为我们的`Dockerfile`指定基础镜像时，我们需要确保还指定了特定版本，而不是让 Docker 简单地使用`latest`镜像。另外，确保您不是从不是来自值得信赖的提供者的存储库或注册表中拉取镜像。

如果您已经使用 Docker 一段时间，可能已经遇到了`MAINTAINER`指令，您可以在其中指定生成图像的作者。现在这已经被弃用，但您仍然可以使用`LABEL`指令来提供这些细节，就像我们在以下语法中所做的那样：

```
LABEL maintainer="myemailaddress@emaildomain.com"
```

## 安装应用程序和语言

当您在镜像上安装应用程序时，永远记住不需要执行`apt-get update`或`dist-upgrade`。如果您需要以这种方式升级镜像版本，应该考虑使用不同的镜像。如果您使用`apt-get`或`apk`安装应用程序，请确保指定您需要的特定版本，因为您不希望安装新的或未经测试的版本。

在安装软件包时，确保使用`-y`开关，以确保构建不会停止并要求用户提示。另外，您还应该使用`--no-install-recommends`，因为您不希望安装大量您的软件包管理器建议的不需要的应用程序。此外，如果您使用基于 Debian 的容器，请确保使用`apt-get`或`apt-cache`，因为`apt`命令专门用于用户交互，而不是用于脚本化安装。

如果您正在从其他形式安装应用程序，比如从代码构建应用程序，请确保清理安装文件，以再次减小您创建的镜像的大小。同样，如果您正在使用`apt-get`，您还应该删除`/var/lib/apt/lists/`中的列表，以清理安装文件并减小容器镜像的大小。

## 运行命令和执行任务

当我们的镜像正在创建时，通常需要在我们的`Dockerfile`中执行一些任务，以准备好我们的服务运行的环境。始终确保您不使用`sudo`命令，因为这可能会导致一些意外的结果。如果需要以 root 身份运行命令，您的基础镜像很可能正在以 root 用户身份运行；只需确保您创建一个单独的用户来运行您的应用程序和服务，并且在构建完成之前容器已经切换到所需的用户。

确保您使用`WORKDIR`切换到不同的目录，而不是运行指定长路径的指令，因为这可能会让用户难以阅读。对于`CMD`和`ENTRYPOINT`参数，请使用`JSON`表示法，并始终确保只有一个`CMD`或`ENTRYPOINT`指令。

## 容器需要是不可变的和无状态的

我们需要确保我们的容器和运行在其中的服务是不可变的。我们不能像传统服务器那样对待容器，特别是在运行容器上更新应用程序的服务器。您应该能够从代码更新容器并部署它，而无需访问它。

当我们说不可变时，我们指的是容器在其生命周期内不会被修改，不会进行更新、补丁或配置更改。您的代码或更新的任何更改都应该通过构建新镜像然后部署到您的环境中来实现。这样做可以使部署更安全，如果升级出现任何问题，您只需重新部署旧版本的镜像。这也意味着您在所有环境中运行相同的镜像，确保您的环境尽可能相同。

当我们谈论容器需要是无状态的时候，这意味着运行容器所需的任何数据都应该在容器外部运行。文件存储也应该在容器外部，可能在云存储上或者使用挂载卷。将数据从容器中移除意味着容器可以在任何时候被干净地关闭和销毁，而不必担心数据丢失。当创建一个新的容器来替换旧的容器时，它只需连接到原始数据存储。

## 设计应用程序以实现高可用性和可扩展性

在微服务架构中使用容器旨在使您的应用程序能够扩展到多个实例。因此，在开发您的应用程序时，您应该预期可能会出现许多实例同时部署的情况，需要在需要时进行上下扩展。当容器负载较重时，您的服务运行和完成也不应该有问题。

当您的服务需要因为增加的请求而扩展时，应用程序需要启动的时间就成为一个重要问题。在将您的服务部署到生产环境之前，您需要确保启动时间很快，以确保系统能够更有效地扩展而不会给用户的服务造成任何延迟。为了确保您的服务符合行业最佳实践，您的服务应该在不到 10 秒内启动，但不到 20 秒也是可以接受的。

正如我们在前一节中所看到的，改善应用程序的启动时间不仅仅是提供更多的 CPU 和内存资源的问题。我们需要确保我们容器中的应用程序能够高效运行，如果它们启动和运行特定进程的时间太长，可能是因为一个应用程序执行了太多的任务。

## 图像和容器需要适当地打标签

我们在《第三章》《管理您的 Docker 镜像》中详细介绍了这个主题，并明确指出，我们需要考虑如何命名和标记我们的图像，特别是当我们开始与更大的开发团队合作时。为了让所有用户能够理解图像的功能，并了解部署到环境中的版本，需要在团队开始大部分工作之前决定并达成一致的相关标记和命名策略。

图像和容器名称需要与它们运行的应用程序相关，因为模糊的名称可能会引起混淆。还必须制定一个版本的约定标准，以确保任何用户都可以确定在特定环境中运行的版本以及最新稳定版本是什么版本。正如我们在*第三章*中提到的*管理您的 Docker 镜像*中所提到的，尽量不要使用`latest`，而是选择语义版本控制系统或 Git 存储库`commit`哈希，用户可以参考文档或构建环境，以确保他们拥有最新版本的镜像。

## 配置和秘密

环境变量和秘密不应该内置到您的 Docker 镜像中。通过这样做，您违反了可重用图像的规则。使用您的秘密凭据构建图像也是一种安全风险，因为它们将存储在图像层中，因此任何能够拉取图像的人都将能够看到凭据。

在为应用程序设置配置时，可能需要根据环境的不同进行更改，因此重要的是要记住，当需要时，您需要能够动态更改这些配置。这可能包括应用程序所编写的语言的特定配置，甚至是应用程序需要连接到的数据库。我们之前提到过，如果您正在配置应用程序作为您的`Dockerfile`的一部分，这将使其难以更改，您可能需要为您希望部署图像的每个环境创建一个特定的`Dockerfile`。

配置图像的一种方法，就像我们在`docker-stress`图像中看到的那样，是使用在运行图像时在命令行上设置的环境变量。如果未提供变量，则入口点或命令应包含默认值。这意味着即使未提供额外的变量，容器仍将启动和运行：

```
docker run -e var="<variable_name>" <image_name>
```

通过这样做，我们使我们的配置更加动态，但是当您有一个更大或更复杂的配置时，这可能会限制您的配置。环境变量可以很容易地从您的`docker run`命令转移到`docker-compose`，然后在 Swarm 或 Kubernetes 中使用。

对于较大的配置，您可能希望通过 Docker 卷挂载配置文件。这意味着您可以设置一个配置文件并在系统上轻松测试运行，然后如果需要转移到诸如 Kubernetes 或 Swarm 之类的编排系统，或者外部配置管理解决方案，您可以轻松将其转换为配置映射。

如果我们想要在本章中使用的`docker-stress`镜像中实现这一点，可以修改为使用配置文件来挂载我们想要运行的值。在以下示例中，我们修改了`Dockerfile`以设置*第 3 行*运行一个脚本，该脚本将代替我们运行`stress`命令：

```
1 FROM ubuntu
2 RUN apt-get update && apt-get install stress
3 CMD ["sh","/tmp/stress_test.sh"]
```

这意味着我们可以构建 Docker 镜像，并使其随时准备好供我们使用。我们只需要一个脚本，我们会挂载在`/tmp`目录中运行。我们可以使用以下示例：

```
1 #!/bin/bash
2 
3 /usr/bin/stress --cpu 8 --timeout 20 --vm 6 --timeout 60
```

这说明了将我们的值从环境变量移动到文件的想法。然后，我们将执行以下操作来运行容器和 stress 应用程序，知道如果我们想要更改`stress`命令使用的变量，我们只需要对我们挂载的文件进行微小的更改：

```
docker run --rm -it -v ${PWD}/stress_test.sh:/tmp/stress_test.sh docker-stress
```

注意

阅读完这些最佳实践清单时，你可能会认为我们违背了很多内容，但请记住，我们在很多情况下都这样做是为了演示一个流程或想法。

## 使您的镜像尽可能精简和小

*第三章*，*管理您的 Docker 镜像*，还让我们尽可能地减小了镜像的大小。我们发现通过减小镜像的大小，可以更快地构建镜像。它们也可以更快地被拉取并在我们的系统上运行。在我们的容器上安装的任何不必要的软件或应用程序都会占用额外的空间和资源，并可能因此减慢我们的服务速度。

正如我们在*第十一章*，*Docker 安全*中所做的那样，使用 Anchore Engine 这样的应用程序显示了我们可以审计我们的镜像以查看其内容，以及安装在其中的应用程序。这是一种简单的方法，可以确保我们减小镜像的大小，使其尽可能精简。

您现在已经了解了您应该在容器镜像和服务中使用的最佳实践。本章的以下部分将帮助您通过使用应用程序来验证您的`Dockerfiles`和`docker-compose.yml`是否按照应有的方式创建来强制执行其中的一些最佳实践。

# 在您的代码中强制执行 Docker 最佳实践

就像我们在开发应用程序时寻求使我们的编码更加简单一样，我们可以使用外部服务和测试来确保我们的 Docker 镜像遵守最佳实践。在本章的以下部分，我们将使用三种工具来确保我们的`Dockerfiles`和`docker-compose.yml`文件遵守最佳实践，并确保我们在构建 Docker 镜像时不会引入潜在问题。

这些工具将使用起来非常简单，并提供强大的功能。我们将首先使用`hadolint`在我们的系统上直接对我们的`Dockerfiles`进行代码检查，它将作为一个独立的 Docker 镜像运行，我们将把我们的`Dockerfiles`输入到其中。然后我们将看一下`FROM:latest`，这是一个在线服务，提供一些基本功能来帮助我们找出`Dockerfiles`中的问题。最后，我们将看一下**Docker Compose Validator**（**DCValidator**），它将执行类似的功能，但在这种情况下，我们将对我们的`docker-compose.yml`文件进行代码检查，以帮助找出潜在问题。

通过在构建和部署我们的镜像之前使用这些工具，我们希望减少我们的 Docker 镜像的构建时间，减少我们引入的错误数量，可能减少我们的 Docker 镜像的大小，并帮助我们更多地了解和执行 Docker 最佳实践。

## 使用 Docker Linter 检查您的镜像

包含本书所有代码的 GitHub 存储库还包括将与构建的 Docker 镜像进行比较的测试。另一方面，代码检查器将分析您的代码，并在构建镜像之前寻找潜在错误。在本章的这一部分，我们正在寻找我们的`Dockerfiles`中的潜在问题，特别是使用一个名为`hadolint`的应用程序。

名称`hadolint`是**Haskell Dockerfile Linter**的缩写，并带有自己的 Docker 镜像，允许您拉取该镜像，然后将您的`Dockerfile`发送到正在运行的镜像以进行测试。即使您的`Dockerfile`相对较小，并且构建和运行没有任何问题，`hadolint`通常会提供许多建议，并指出`Dockerfile`中的缺陷，以及可能在将来出现问题的潜在问题。

要在您的`Dockerfiles`上运行`hadolint`，您需要在您的系统上有`hadolint` Docker 镜像。正如您现在所知，这只是运行`docker pull`命令并使用所需镜像的名称和存储库的问题。在这种情况下，存储库和镜像都称为`hadolint`：

```
docker pull hadolint/hadolint
```

然后，您可以简单地运行`hadolint`镜像，并使用小于(`<`)符号将您的`Dockerfile`指向它，就像我们在以下示例中所做的那样：

```
docker run hadolint/hadolint < Dockerfile
```

如果您足够幸运，没有任何问题与您的`Dockerfile`，您不应该看到前面命令的任何输出。如果有需要忽略特定警告的情况，您可以使用`--ignore`选项，后跟触发警告的特定规则 ID：

```
docker run hadolint/hadolint hadolint --ignore <hadolint_rule_id> - < Dockerfile
```

如果您需要忽略一些警告，尝试在命令行中实现可能会有点复杂，因此`hadolint`还有设置配置文件的选项。`hadolint`配置文件仅限于忽略警告并提供受信任存储库的列表。您还可以使用 YAML 格式设置包含您忽略警告列表的配置文件。然后，`hadolint`将需要在运行的镜像上挂载此文件，以便应用程序使用它，因为它将在应用程序的主目录中查找`.hadolint.yml`配置文件位置：

```
docker run --rm -i -v ${PWD}/.hadolint.yml:/.hadolint.yaml hadolint/hadolint < Dockerfile
```

`hadolint`是用于清理您的`Dockerfiles`的更好的应用程序之一，并且可以轻松地作为构建和部署流水线的一部分进行自动化。作为替代方案，我们还将看一下名为`FROM:latest`的在线应用程序。这个应用程序是一个基于 Web 的服务，不提供与`hadolint`相同的功能，但允许您轻松地将您的`Dockerfile`代码复制粘贴到在线编辑器中，并获得有关`Dockerfile`是否符合最佳实践的反馈。

## 练习 12.04：清理您的 Dockerfile

此练习将帮助您了解如何在系统上访问和运行`hadolint`，以帮助您强制执行`Dockerfiles`的最佳实践。我们还将使用一个名为`FROM:latest`的在线`Dockerfile` linter 来比较我们收到的警告：

1.  使用以下`docker pull`命令从`hadolint`存储库中拉取镜像：

```
docker pull hadolint/hadolint
```

1.  您已经准备好一个`Dockerfile`，其中包含您在本章早些时候用来测试和管理资源的`docker-stress`镜像。运行`hadolint`镜像以对此`Dockerfile`进行检查，或者对任何其他`Dockerfile`进行检查，并使用小于（`<`）符号发送到`Dockerfile`，如以下命令所示：

```
docker run --rm -i hadolint/hadolint < Dockerfile
```

从以下输出中可以看出，即使我们的`docker-stress`镜像相对较小，`hadolint`也提供了许多不同的方式，可以改善性能并帮助我们的镜像遵守最佳实践：

```
/dev/stdin:1 DL3006 Always tag the version of an image explicitly
/dev/stdin:2 DL3008 Pin versions in apt get install. Instead of 
'apt-get install <package>' use 'apt-get install 
<package>=<version>'
/dev/stdin:2 DL3009 Delete the apt-get lists after installing 
something
/dev/stdin:2 DL3015 Avoid additional packages by specifying 
'--no-install-recommends'
/dev/stdin:2 DL3014 Use the '-y' switch to avoid manual input 
'apt-get -y install <package>'
/dev/stdin:3 DL3025 Use arguments JSON notation for CMD 
and ENTRYPOINT arguments
```

注意

如果您的`Dockerfile`通过`hadolint`成功运行，并且没有发现任何问题，则在命令行上不会向用户呈现任何输出。

1.  `hadolint`还为您提供了使用`--ignore`选项来抑制不同检查的选项。在以下命令中，我们选择忽略`DL3008`警告，该警告建议您将安装的应用程序固定到特定版本号。执行`docker run`命令以抑制`DL3008`警告。请注意，在指定运行的镜像名称之后，您需要提供完整的`hadolint`命令，以及在提供`Dockerfile`之前提供额外的破折号（`-`）：

```
docker run --rm -i hadolint/hadolint hadolint --ignore DL3008 - < Dockerfile
```

您应该获得以下类似的输出：

```
/dev/stdin:1 DL3006 Always tag the version of an image explicitly
/dev/stdin:2 DL3009 Delete the apt-get lists after installing 
something
/dev/stdin:2 DL3015 Avoid additional packages by specifying 
'--no-install-recommends'
/dev/stdin:2 DL3014 Use the '-y' switch to avoid manual input 
'apt-get -y install <package>'
/dev/stdin:3 DL3025 Use arguments JSON notation for CMD and 
ENTRYPOINT arguments
```

1.  `hadolint`还允许您创建一个配置文件，以添加要忽略的任何警告，并在命令行上指定它们。使用`touch`命令创建一个名为`.hadolint.yml`的文件：

```
touch .hadolint.yml
```

1.  使用文本编辑器打开配置文件，并在`ignored`字段下输入您希望忽略的任何警告。如您所见，您还可以添加一个`trustedRegistries`字段，在其中列出您将从中拉取镜像的所有注册表。请注意，如果您的镜像不来自配置文件中列出的注册表之一，`hadolint`将提供额外的警告：

```
ignored:
  - DL3006
  - DL3008
  - DL3009
  - DL3015
  - DL3014
trustedRegistries:
  - docker.io
```

1.  `hadolint`将在用户的主目录中查找您的配置文件。由于您正在作为 Docker 镜像运行`hadolint`，因此在执行`docker run`命令时，使用`-v`选项将文件从当前位置挂载到运行镜像的主目录上：

```
docker run --rm -i -v ${PWD}/.hadolint.yml:/.hadolint.yaml hadolint/hadolint < Dockerfile
```

该命令将输出如下：

```
/dev/stdin:3 DL3025 Use arguments JSON notation for CMD and ENTRYPOINT arguments
```

注意

`hadolint`的源代码存储库提供了所有警告的列表，以及如何在您的`Dockerfile`中解决这些问题的详细信息。如果您还没有这样做，可以随意查看 Hadolint 维基页面[`github.com/hadolint/hadolint/wiki`](https://github.com/hadolint/hadolint/wiki)。

1.  最后，`hadolint`还允许您选择以 JSON 格式输出检查结果。再次，我们需要在命令行中添加一些额外的值。在命令行中，在将您的`Dockerfile`添加和解析到`hadolint`之前，添加额外的命令行选项`hadolint -f json`。在以下命令中，您还需要安装`jq`软件包：

```
docker run --rm -i -v ${PWD}/.hadolint.yml:/.hadolint.yaml hadolint/hadolint hadolint -f json - < Dockerfile | jq
```

您应该得到以下输出：

```
[
  {
    "line": 3,
    "code": "DL3025",
    "message": "Use arguments JSON notation for CMD and ENTRYPOINT arguments",
    "column": 1,
    "file": "/dev/stdin",
    "level": "warning"
  }
]
```

注意

`hadolint`可以轻松集成到您的构建流水线中，在构建之前对您的`Dockerfiles`进行检查。如果您有兴趣直接将`hadolint`应用程序安装到您的系统上，而不是使用 Docker 镜像，您可以通过克隆以下 GitHub 存储库来实现[`github.com/hadolint/hadolint`](https://github.com/hadolint/hadolint)。

`hadolint`并不是您可以用来确保您的`Dockerfiles`遵守最佳实践的唯一应用程序。这个练习的下一步将介绍一个名为`FROM:latest`的在线服务，也可以帮助强制执行`Dockerfiles`的最佳实践。

1.  要使用`FROM:latest`，打开您喜欢的网络浏览器，输入以下 URL：

```
https://www.fromlatest.io
```

当网页加载时，您应该看到类似以下截图的页面。在网页的左侧，您应该看到输入了一个示例`Dockerfile`，在网页的右侧，您应该看到一个潜在问题或优化`Dockerfile`的方法列表。右侧列出的每个项目都有一个下拉菜单，以向用户提供更多详细信息：

![图 12.1：FROM:latest 网站的截图，显示输入了一个示例 Dockerfile](img/B15021_12_01.jpg)

图 12.1：FROM:latest 网站的截图，显示输入了一个示例 Dockerfile

1.  在这个练习的前一部分中，我们将使用`docker-stress`镜像的`Dockerfile`。要将其与`FROM:latest`一起使用，请将以下代码行复制到网页左侧，覆盖网站提供的示例`Dockerfile`：

```
FROM ubuntu
RUN apt-get update && apt-get install stress
CMD stress $var
```

一旦您将`Dockerfile`代码发布到网页上，页面将开始分析命令。正如您从以下截图中所看到的，它将提供有关如何解决潜在问题并优化`Dockerfile`以使镜像构建更快的详细信息：

![图 12.2：我们的 docker-stress 镜像输入的 Dockerfile](img/B15021_12_02.jpg)

图 12.2：我们的 docker-stress 镜像输入的 Dockerfile

`hadolint`和`FROM latest`都提供了易于使用的选项，以帮助您确保您的`Dockerfiles`遵守最佳实践。下一个练习将介绍一种类似的方法，用于检查您的`docker-compose.yml`文件，以确保它们也可以无故障运行，并且不会引入任何不良实践。

## 练习 12.05：验证您的 docker-compose.yml 文件

Docker 已经有一个工具来验证您的`docker-compose.yml`文件，但是内置的验证器无法捕捉到`docker-compose`文件中的所有问题，包括拼写错误、相同的端口分配给不同的服务或重复的键。我们可以使用`dcvalidator`来查找诸如拼写错误、重复的键和分配给数字服务的端口等问题。

要执行以下练习，您需要在系统上安装 Git 和最新版本的 Python 3。在开始之前，您不会被引导如何执行安装，但在开始之前需要这些项目。

1.  要开始使用`dcvalidator`，请克隆该项目的 GitHub 存储库。如果您还没有这样做，您需要运行以下命令来克隆存储库：

```
git clone https://github.com/serviceprototypinglab/dcvalidator.git
```

1.  命令行应用程序只需要 Python 3 来运行，但是您需要确保首先安装了所有的依赖项，因此请切换到您刚刚克隆的存储库的`dcvalidator`目录：

```
cd dcvalidator
```

1.  安装`dcvalidator`的依赖项很容易，您的系统很可能已经安装了大部分依赖项。要安装依赖项，请在`dcvalidator`目录中使用`pip3 install`命令，并使用`-r`选项来使用服务器目录中的`requirments.txt`文件：

```
pip3 install -r server/requirments.txt
```

1.  从头开始创建一个`docker-compose`文件，该文件将使用本章中已经创建的一些镜像。使用`touch`命令创建一个`docker-compose.yml`文件：

```
touch docker-compose.yml
```

1.  打开您喜欢的文本编辑器来编辑`docker-compose`文件。确保您还包括我们故意添加到文件中的错误，以确保`dcvalidator`能够发现这些错误，并且我们将使用本章前面创建的`docker-stress`镜像。确保您逐字复制此文件，因为我们正在努力确保在我们的`docker-compose.yml`文件中强制出现一些错误：

```
version: '3'
services:
  app:
    container_name: docker-stress-20
    build: .
    environment:
      var: "--cpu 2 --vm 6 --timeout 20"
    ports:
      - 80:8080
      - 80:8080
    dns: 8.8.8
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 50M
  app2:
    container_name: docker-stress-30
    build: .
    environment:
      var: "--cpu 2 --vm 6 --timeout 30"
    dxeploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 50M
```

1.  使用`-f`选项运行`validator-cli.py`脚本来解析我们想要验证的特定文件——在以下命令行中，即`docker-compose.yml`文件。然后，`-fi`选项允许您指定可用于验证我们的`compose`文件的过滤器。在以下代码中，我们正在使用`validator-cli`目前可用的所有过滤器：

```
python3 validator-cli.py -f docker-compose.yml -fi 'Duplicate Keys,Duplicate ports,Typing mistakes,DNS,Duplicate expose'
```

您应该获得以下类似的输出：

```
Warning: no kafka support
loading compose files....
checking consistency...
syntax is ok
= type: docker-compose
- service:app
Duplicate ports in service app port 80
=================== ERROR ===================
Under service: app
The DNS is not appropriate!
=============================================
- service:app2
=================== ERROR ===================
I can not find 'dxeploy' tag under 'app2' service. 
Maybe you can use: 
deploy
=============================================
services: 2
labels:
time: 0.0s
```

正如预期的那样，`validator-cli.py`已经能够找到相当多的错误。它显示您在应用服务中分配了重复的端口，并且您设置的 DNS 也是不正确的。`App2`显示了一些拼写错误，并建议我们可以使用不同的值。

注意

在这一点上，您需要指定您希望您的`docker-compose.yml`文件针对哪些过滤器进行验证，但这将随着即将发布的版本而改变。

1.  您会记得，我们使用了一个`docker-compose`文件来安装 Anchore 镜像扫描程序。当您有`compose`文件的 URL 位置时，使用`-u`选项传递文件的 URL 以进行验证。在这种情况下，它位于 Packt GitHub 账户上：

```
python3 validator-cli.py -u https://github.com/PacktWorkshops/The-Docker-Workshop/blob/master/Chapter11/Exercise11.03/docker-compose.yaml -fi 'Duplicate Keys,Duplicate ports,Typing mistakes,DNS,Duplicate expose'
```

如您在以下代码块中所见，`dcvalidator`没有在`docker-compose.yml`文件中发现任何错误：

```
Warning: no kafka support
discard cache...
loading compose files....
checking consistency...
syntax is ok
= type: docker-compose=
- service:engine-api
- service:engine-catalog
- service:engine-simpleq
- service:engine-policy-engine
- service:engine-analyzer
- service:anchore-db
services: 6
labels:
time: 0.6s
```

如您所见，Docker Compose 验证器相当基本，但它可以发现我们可能错过的`docker-compose.yml`文件中的一些错误。特别是在我们有一个较大的文件时，如果我们在尝试部署环境之前可能错过了一些较小的错误，这可能是可能的。这已经将我们带到了本章的这一部分的结束，我们一直在使用一些自动化流程和应用程序来验证和清理我们的`Dockerfiles`和`docker-compose.yml`文件。

现在，让我们继续进行活动，这将帮助您测试对本章的理解。在接下来的活动中，您将查看 Panoramic Trekking App 上运行的一个服务使用的资源。

## 活动 12.01：查看 Panoramic Trekking App 使用的资源

在本章的前面，我们看了一下我们正在运行的容器在我们的主机系统上消耗了多少资源。在这个活动中，您将选择全景徒步应用程序上运行的服务之一，使用其默认配置运行容器，并查看它使用了什么 CPU 和内存资源。然后，再次运行容器，更改 CPU 和内存配置，以查看这如何影响资源使用情况：

您需要完成此活动的一般步骤如下：

1.  决定在全景徒步应用程序中选择一个您想要测试的服务。

1.  创建一组测试，然后用它们来测量服务的资源使用情况。

1.  启动您的服务，并使用您在上一步中创建的测试来监视资源使用情况。

1.  停止您的服务运行，并再次运行它，这次更改 CPU 和内存配置。

1.  再次使用您在*步骤 2*中创建的测试监视资源使用情况，并比较资源使用情况的变化。

注意

此活动的解决方案可以通过此链接找到。

下一个活动将帮助您在您的`Dockerfiles`上使用`hadolint`来改进最佳实践。

## 活动 12.02：使用 hadolint 改进 Dockerfiles 上的最佳实践

`hadolint`提供了一个很好的方式来强制执行最佳实践，当您创建您的 Docker 镜像时。在这个活动中，您将再次使用`docker-stress`镜像的`Dockerfile`，以查看您是否可以使用`hadolint`的建议来改进`Dockerfile`，使其尽可能地符合最佳实践。

您需要完成此活动的步骤如下：

1.  确保您的系统上有`hadolint`镜像可用并正在运行。

1.  对`docker-stress`镜像的`Dockerfile`运行`hadolint`镜像，并记录结果。

1.  对上一步中的`Dockerfile`进行推荐的更改。

1.  再次测试`Dockerfile`。

完成活动后，您应该获得以下输出：

![图 12.3：活动 12.02 的预期输出](img/B15021_12_03.jpg)

图 12.3：活动 12.02 的预期输出

注意

此活动的解决方案可以通过此链接找到。

# 总结

本章我们深入研究了许多理论知识，以及对练习进行了深入的工作。我们从查看我们的运行 Docker 容器如何利用主机系统的 CPU、内存和磁盘资源开始了本章。我们研究了监视这些资源如何被我们的容器消耗，并配置我们的运行容器以减少使用的资源数量。

然后，我们研究了 Docker 的最佳实践，涉及了许多不同的主题，包括利用基础镜像、安装程序和清理、为可扩展性开发底层应用程序，以及配置应用程序和镜像。然后，我们介绍了一些工具，帮助您执行这些最佳实践，包括`hadolint`和`FROM:latest`，帮助您对`Dockerfiles`进行代码检查，以及`dcvalidator`来检查您的`docker-compose.yml`文件。

下一章将进一步提升我们的监控技能，介绍使用 Prometheus 来监控我们的容器指标和资源。

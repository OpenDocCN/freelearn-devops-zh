# 第七章。与容器共享数据

一次只做一件事，并且做好，是信息技术（IT）部门长期以来的成功口头禅之一。这个广泛使用的原则也很好地适用于构建和暴露 Docker 容器，并被规定为实现最初设想的 Docker 启发式容器化范式的最佳实践之一。也就是说，将单个应用程序以及其直接依赖项和库放在 Docker 容器中，以确保容器的独立性、自给自足性和可操纵性。让我们看看为什么容器如此重要：

+   容器的临时性质：容器通常存在的时间与应用程序存在的时间一样长。然而，这对应用程序数据有一些负面影响。应用程序自然会经历各种变化，以适应业务和技术变化，甚至在其生产环境中也是如此。还有其他原因，比如应用程序故障、版本更改、应用程序维护等，导致应用程序需要不断更新和升级。在通用计算模型的情况下，即使应用程序因任何原因而死亡，与该应用程序关联的持久数据也会保存在文件系统中。然而，在容器范式的情况下，应用程序升级通常是通过创建一个具有较新版本应用程序的新容器来完成的，然后丢弃旧容器。同样，当应用程序发生故障时，需要启动一个新容器，并丢弃旧容器。总之，容器具有临时性质。

+   企业连续性的需求：在容器环境中，完整的执行环境，包括其数据文件通常被捆绑和封装在容器内。无论出于何种原因，当一个容器被丢弃时，应用程序数据文件也会随着容器一起消失。然而，为了提供无缝的服务，这些应用程序数据文件必须在容器外部保留，并传递给将继续提供服务的容器。一些应用程序数据文件，如日志文件，需要在容器外部进行各种后续分析。Docker 技术通过一个称为数据卷的新构建块非常创新地解决了这个文件持久性问题。

在本章中，我们将涵盖以下主题：

+   数据卷

+   共享主机数据

+   在容器之间共享数据

+   可避免的常见陷阱

# 数据卷

数据卷是 Docker 环境中数据共享的基本构建块。在深入了解数据共享的细节之前，必须对数据卷概念有很好的理解。到目前为止，我们在镜像或容器中创建的所有文件都是联合文件系统的一部分。然而，数据卷是 Docker 主机文件系统的一部分，它只是在容器内部挂载。

数据卷可以使用`Dockerfile`的`VOLUME`指令在 Docker 镜像中进行刻录。此外，可以在启动容器时使用`docker run`子命令的`-v`选项进行指定。在下面的示例中，将详细说明在`Dockerfile`中使用`VOLUME`指令的含义，具体步骤如下：

1.  创建一个非常简单的`Dockerfile`，其中包含基础镜像（`ubuntu:14.04`）和数据卷（`/MountPointDemo`）的指令：

```
FROM ubuntu:14.04
VOLUME /MountPointDemo
```

1.  使用`docker build`子命令构建名称为`mount-point-demo`的镜像：

```
**$ sudo docker build -t mount-point-demo .**

```

1.  构建完镜像后，让我们使用`docker inspect`子命令快速检查我们的数据卷：

```
**$ sudo docker inspect mount-point-demo**
**[{**
 **"Architecture": "amd64",**
**... TRUNCATED OUTPUT ...** 
 **"Volumes": {**
 **"/MountPointDemo": {}**
 **},**
**... TRUNCATED OUTPUT ...**

```

显然，在前面的输出中，数据卷是直接刻录在镜像中的。

1.  现在，让我们使用先前创建的镜像启动一个交互式容器，如下命令所示：

```
**$ sudo docker run --rm -it mount-point-demo**

```

从容器的提示符中，使用`ls -ld`命令检查数据卷的存在：

```
**root@8d22f73b5b46:/# ls -ld /MountPointDemo**
**drwxr-xr-x 2 root root 4096 Nov 18 19:22 /MountPointDemo**

```

如前所述，数据卷是 Docker 主机文件系统的一部分，并且会被挂载，如下命令所示：

```
**root@8d22f73b5b46:/# mount**
**... TRUNCATED OUTPUT ...**
**/dev/disk/by-uuid/721cedbd-57b1-4bbd-9488-ec3930862cf5 on /MountPointDemo type ext3 (rw,noatime,nobarrier,errors=remount-ro,data=ordered)**
**... TRUNCATED OUTPUT ...**

```

1.  在本节中，我们检查了镜像，以了解镜像中的数据卷声明。现在我们已经启动了容器，让我们在另一个终端中使用`docker inspect`子命令和容器 ID 作为参数来检查容器的数据卷。我们之前创建了一些容器，为此，让我们直接从容器的提示符中获取容器 ID`8d22f73b5b46`：

```
**$ sudo docker inspect 8d22f73b5b46**
**... TRUNCATED OUTPUT ...**
 **"Volumes": {**
 **"/MountPointDemo": "/var/lib/docker/vfs/dir/737e0355c5d81c96a99d41d1b9f540c2a212000661633ceea46f2c298a45f128"**
 **},**
 **"VolumesRW": {**
 **"/MountPointDemo": true**
 **}**
**}**

```

显然，在这里，数据卷被映射到 Docker 主机中的一个目录，并且该目录以读写模式挂载。这个目录是由 Docker 引擎在容器启动时自动创建的。

到目前为止，我们已经看到了`Dockerfile`中`VOLUME`指令的含义，以及 Docker 如何管理数据卷。像`Dockerfile`中的`VOLUME`指令一样，我们可以使用`docker run`子命令的`-v <容器挂载点路径>`选项，如下面的命令所示：

```
**$ sudo docker run –v /MountPointDemo -it ubuntu:14.04**

```

启动容器后，我们鼓励您尝试在新启动的容器中使用`ls -ld /MountPointDemo`和`mount`命令，然后也像前面的步骤 5 中所示那样检查容器。

在这里描述的两种情况中，Docker 引擎会自动在`/var/lib/docker/vfs/`目录下创建目录，并将其挂载到容器中。当使用`docker rm`子命令删除容器时，Docker 引擎不会删除在容器启动时自动创建的目录。这种行为本质上是为了保留存储在目录中的容器应用程序的状态。如果您想删除 Docker 引擎自动创建的目录，可以在删除容器时使用`docker rm`子命令提供`-v`选项来执行，前提是容器已经停止：

```
**$ sudo docker rm -v 8d22f73b5b46**

```

如果容器仍在运行，则可以通过在上一个命令中添加`-f`选项来删除容器以及自动生成的目录：

```
**$ sudo docker rm -fv 8d22f73b5b46**

```

我们已经介绍了在 Docker 主机中自动生成目录并将其挂载到容器数据卷的技术和提示。然而，使用`docker run`子命令的`-v`选项可以将用户定义的目录挂载到数据卷。在这种情况下，Docker 引擎不会自动生成任何目录。

### 注意

系统生成的目录存在目录泄漏的问题。换句话说，如果您忘记删除系统生成的目录，可能会遇到一些不必要的问题。有关更多信息，您可以阅读本章节中的*避免常见陷阱*部分。

# 共享主机数据

之前，我们描述了在 Docker 镜像中使用`Dockerfile`中的`VOLUME`指令创建数据卷的步骤。然而，Docker 没有提供任何机制在构建时挂载主机目录或文件，以确保 Docker 镜像的可移植性。Docker 提供的唯一规定是在容器启动时将主机目录或文件挂载到容器的数据卷上。Docker 通过`docker run`子命令的`-v`选项公开主机目录或文件挂载功能。`-v`选项有三种不同的格式，如下所列：

1.  -v <容器挂载路径>

1.  `-v <host path>/<container mount path>`

1.  `-v <host path>/<container mount path>:<read write mode>`

`<host path>`是 Docker 主机上的绝对路径，`<container mount path>`是容器文件系统中的绝对路径，`<read write mode>`可以是只读（`ro`）或读写（`rw`）模式。第一个`-v <container mount path>`格式已经在本章的*数据卷*部分中解释过，作为在容器启动时创建挂载点的方法。第二和第三个选项使我们能够将 Docker 主机上的文件或目录挂载到容器的挂载点。

我们希望通过几个例子深入了解主机数据共享。在第一个例子中，我们将演示如何在 Docker 主机和容器之间共享一个目录，在第二个例子中，我们将演示文件共享。

在第一个例子中，我们将一个目录从 Docker 主机挂载到一个容器中，在容器上执行一些基本的文件操作，并从 Docker 主机验证这些操作，详细步骤如下：

1.  首先，让我们使用`docker run`子命令的`-v`选项启动一个交互式容器，将 Docker 主机目录`/tmp/hostdir`挂载到容器的`/MountPoint`：

```
**$ sudo docker run -v /tmp/hostdir:/MountPoint \**
 **-it ubuntu:14.04**

```

### 注意

如果在 Docker 主机上找不到`/tmp/hostdir`，Docker 引擎将自行创建该目录。然而，问题在于系统生成的目录无法使用`docker rm`子命令的`-v`选项删除。

1.  成功启动容器后，我们可以使用`ls`命令检查`/MountPoint`的存在：

```
**root@4a018d99c133:/# ls -ld /MountPoint**
**drwxr-xr-x 2 root root 4096 Nov 23 18:28 /MountPoint**

```

1.  现在，我们可以继续使用`mount`命令检查挂载细节：

```
**root@4a018d99c133:/# mount**
**... TRUNCATED OUTPUT ...**
**/dev/disk/by-uuid/721cedbd-57b1-4bbd-9488-ec3930862cf5 on /MountPoint type ext3 (rw,noatime,nobarrier,errors=remount-ro,data=ordered)**
**... TRUNCATED OUTPUT ...**

```

1.  在这里，我们将验证`/MountPoint`，使用`cd`命令切换到`/MountPoint`目录，使用`touch`命令创建一些文件，并使用`ls`命令列出文件，如下脚本所示：

```
**root@4a018d99c133:/# cd /MountPoint/**
**root@4a018d99c133:/MountPoint# touch {a,b,c}**
**root@4a018d99c133:/MountPoint# ls -l**
**total 0**
**-rw-r--r-- 1 root root 0 Nov 23 18:39 a**
**-rw-r--r-- 1 root root 0 Nov 23 18:39 b**
**-rw-r--r-- 1 root root 0 Nov 23 18:39 c**

```

1.  可能值得努力使用新终端上的`ls`命令验证`/tmp/hostdir` Docker 主机目录中的文件，因为我们的容器正在现有终端上以交互模式运行：

```
**$ sudo  ls -l /tmp/hostdir/**
**total 0**
**-rw-r--r-- 1 root root 0 Nov 23 12:39 a**
**-rw-r--r-- 1 root root 0 Nov 23 12:39 b**
**-rw-r--r-- 1 root root 0 Nov 23 12:39 c**

```

在这里，我们可以看到与第 4 步中相同的一组文件。但是，您可能已经注意到文件的时间戳有所不同。这种时间差异是由于 Docker 主机和容器之间的时区差异造成的。

1.  最后，让我们运行`docker inspect`子命令，以容器 ID`4a018d99c133`作为参数，查看 Docker 主机和容器挂载点之间是否设置了目录映射，如下命令所示：

```
**$ sudo docker inspect \**
 **--format={{.Volumes}} 4a018d99c133**
**map[/MountPoint:/tmp/hostdir]**

```

显然，在`docker inspect`子命令的先前输出中，Docker 主机的`/tmp/hostdir`目录被挂载到容器的`/MountPoint`挂载点上。

对于第二个示例，我们可以将文件从 Docker 主机挂载到容器中，从容器中更新文件，并从 Docker 主机验证这些操作，详细步骤如下：

1.  为了将文件从 Docker 主机挂载到容器中，文件必须在 Docker 主机上预先存在。否则，Docker 引擎将创建一个具有指定名称的新目录，并将其挂载为目录。我们可以通过使用`touch`命令在 Docker 主机上创建一个文件来开始：

```
**$ touch /tmp/hostfile.txt**

```

1.  使用`docker run`子命令的`-v`选项启动交互式容器，将`/tmp/hostfile.txt` Docker 主机文件挂载到容器上，作为`/tmp/mntfile.txt`：

```
**$ sudo docker run -v /tmp/hostfile.txt:/mountedfile.txt \**
 **-it ubuntu:14.04**

```

1.  成功启动容器后，现在让我们使用`ls`命令检查`/mountedfile.txt`的存在：

```
**root@d23a15527eeb:/# ls -l /mountedfile.txt**
**-rw-rw-r-- 1 1000 1000 0 Nov 23 19:33 /mountedfile.txt**

```

1.  然后，继续使用`mount`命令检查挂载细节：

```
**root@d23a15527eeb:/# mount**
**... TRUNCATED OUTPUT ...**
**/dev/disk/by-uuid/721cedbd-57b1-4bbd-9488-ec3930862cf5 on /mountedfile.txt type ext3 (rw,noatime,nobarrier,errors=remount-ro,data=ordered)**
**... TRUNCATED OUTPUT ...**

```

1.  然后，使用`echo`命令更新`/mountedfile.txt`中的一些文本：

```
**root@d23a15527eeb:/# echo "Writing from Container" \**
 **> mountedfile.txt**

```

1.  同时，在 Docker 主机中切换到另一个终端，并使用`cat`命令打印`/tmp/hostfile.txt` Docker 主机文件：

```
**$ cat /tmp/hostfile.txt**
**Writing from Container**

```

1.  最后，运行`docker inspect`子命令，以容器 ID`d23a15527eeb`作为参数，查看 Docker 主机和容器挂载点之间的文件映射：

```
**$ sudo docker inspect \**
 **--format={{.Volumes}} d23a15527eeb**
**map[/mountedfile.txt:/tmp/hostfile.txt]**

```

从前面的输出可以看出，来自 Docker 主机的`/tmp/hostfile.txt`文件被挂载为容器内的`/mountedfile.txt`。

### 注意

在 Docker 主机和容器之间共享文件的情况下，文件必须在启动容器之前存在。然而，在目录共享的情况下，如果 Docker 主机中不存在该目录，则 Docker 引擎会在 Docker 主机中创建一个新目录，如前面所述。

## 主机数据共享的实用性

在上一章中，我们在 Docker 容器中启动了一个`HTTP`服务。然而，如果你记得正确的话，`HTTP`服务的日志文件仍然在容器内，无法直接从 Docker 主机访问。在这里，在本节中，我们逐步阐述了从 Docker 主机访问日志文件的过程：

1.  让我们开始启动一个 Apache2 HTTP 服务容器，将 Docker 主机的`/var/log/myhttpd`目录挂载到容器的`/var/log/apache2`目录，使用`docker run`子命令的`-v`选项。在这个例子中，我们正在利用我们在上一章中构建的`apache2`镜像，通过调用以下命令：

```
**$ sudo docker run -d -p 80:80 \**
 **-v /var/log/myhttpd:/var/log/apache2 apache2**
**9c2f0c0b126f21887efaa35a1432ba7092b69e0c6d523ffd50684e27eeab37ac**

```

如果你还记得第六章中的`Dockerfile`，*在容器中运行服务*，`APACHE_LOG_DIR`环境变量被设置为`/var/log/apache2`目录，使用`ENV`指令。这将使 Apache2 HTTP 服务将所有日志消息路由到`/var/log/apache2`数据卷。

1.  容器启动后，我们可以在 Docker 主机上切换到`/var/log/myhttpd`目录：

```
**$ cd /var/log/myhttpd**

```

1.  也许，在这里适当地快速检查`/var/log/myhttpd`目录中存在的文件：

```
**$ ls -1**
**access.log**
**error.log**
**other_vhosts_access.log**

```

在这里，`access.log`包含了 Apache2 HTTP 服务器处理的所有访问请求。`error.log`是一个非常重要的日志文件，我们的 HTTP 服务器在处理任何 HTTP 请求时记录遇到的错误。`other_vhosts_access.log`文件是虚拟主机日志，在我们的情况下始终为空。

1.  我们可以使用`tail`命令和`-f`选项显示`/var/log/myhttpd`目录中所有日志文件的内容：

```
**$ tail -f *.log**
**==> access.log <==**

**==> error.log <==**
**AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 172.17.0.17\. Set the 'ServerName' directive globally to suppress this message**
**[Thu Nov 20 17:45:35.619648 2014] [mpm_event:notice] [pid 16:tid 140572055459712] AH00489: Apache/2.4.7 (Ubuntu) configured -- resuming normal operations**
**[Thu Nov 20 17:45:35.619877 2014] [core:notice] [pid 16:tid 140572055459712] AH00094: Command line: '/usr/sbin/apache2 -D FOREGROUND'**
**==> other_vhosts_access.log <==** 

```

`tail -f` 命令将持续运行并显示文件的内容，一旦它们被更新。在这里，`access.log` 和 `other_vhosts_access.log` 都是空的，并且 `error.log` 文件上有一些错误消息。显然，这些错误日志是由容器内运行的 HTTP 服务生成的。然后，这些日志被储存在 Docker 主机目录中，在容器启动时被挂载。

1.  当我们继续运行 `tail –f *` 时，让我们从容器内运行的 Web 浏览器连接到 HTTP 服务，并观察日志文件：

```
**==> access.log <==**
**111.111.172.18 - - [20/Nov/2014:17:53:38 +0000] "GET / HTTP/1.1" 200 3594 "-" "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/39.0.2171.65 Safari/537.36"**
**111.111.172.18 - - [20/Nov/2014:17:53:39 +0000] "GET /icons/ubuntu-logo.png HTTP/1.1" 200 3688 "http://111.71.123.110/" "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/39.0.2171.65 Safari/537.36"**
**111.111.172.18 - - [20/Nov/2014:17:54:21 +0000] "GET /favicon.ico HTTP/1.1" 404 504 "-" "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/39.0.2171.65 Safari/537.36"**

```

HTTP 服务更新 `access.log` 文件，我们可以通过 `docker run` 子命令的 `–v` 选项挂载的主机目录进行操作。

# 在容器之间共享数据

在前面的部分中，我们了解了 Docker 引擎如何在 Docker 主机和容器之间无缝地实现数据共享。尽管这是一个非常有效的解决方案，但它将容器紧密耦合到主机文件系统。这些目录可能会留下不好的印记，因为用户必须在它们的目的达到后手动删除它们。因此，Docker 解决这个问题的建议是创建数据专用容器作为基础容器，然后使用 `docker run` 子命令的 `--volume-from` 选项将该容器的数据卷挂载到其他容器。

## 数据专用容器

数据专用容器的主要责任是保存数据。创建数据专用容器与数据卷部分所示的方法非常相似。此外，容器被明确命名，以便其他容器使用容器的名称挂载数据卷。即使数据专用容器处于停止状态，其他容器也可以访问数据专用容器的数据卷。数据专用容器可以通过以下两种方式创建：

+   在容器启动时通过配置数据卷和容器名称。

+   数据卷也可以在构建镜像时通过 `Dockerfile` 进行编写，然后在容器启动时命名容器。

在以下示例中，我们通过配置 `docker run` 子命令的 `–v` 和 `--name` 选项来启动一个数据专用容器，如下所示：

```
**$ sudo docker run --name datavol \**
 **-v /DataMount \**
 **busybox:latest /bin/true**

```

在这里，容器是从`busybox`镜像启动的，该镜像因其较小的占用空间而被广泛使用。在这里，我们选择执行`/bin/true`命令，因为我们不打算对容器进行任何操作。因此，我们使用`--name`选项命名了容器`datavol`，并使用`docker run`子命令的`-v`选项创建了一个新的`/DataMount`数据卷。`/bin/true`命令立即以退出状态`0`退出，这将停止容器并继续停留在停止状态。

## 从其他容器挂载数据卷

Docker 引擎提供了一个巧妙的接口，可以将一个容器的数据卷挂载（共享）到另一个容器。Docker 通过`docker run`子命令的`--volumes-from`选项提供了这个接口。`--volumes-from`选项以容器名称或容器 ID 作为输入，并自动挂载指定容器上的所有数据卷。Docker 允许您多次使用`--volumes-from`选项来挂载多个容器的数据卷。

这是一个实际的示例，演示了如何从另一个容器挂载数据卷，并逐步展示数据卷挂载过程。

1.  我们首先启动一个交互式 Ubuntu 容器，通过挂载数据专用容器（`datavol`）中的数据卷来进行操作，如前述所述：

```
**$ sudo docker run –it \**
 **--volumes-from datavol \**
 **ubuntu:latest /bin/bash**

```

1.  现在从容器的提示符中，让我们使用`mount`命令验证数据卷挂载：

```
**root@e09979cacec8:/# mount**
**. . . TRUNCATED OUTPUT . . .** 
**/dev/disk/by-uuid/32a56fe0-7053-4901-ae7e-24afe5942e91 on /DataMount type ext3 (rw,noatime,nobarrier,errors=remount-ro,data=ordered)**
**. . . TRUNCATED OUTPUT . . .** 

```

在这里，我们成功地从`datavol`数据专用容器中挂载了数据卷。

1.  接下来，我们需要使用`docker inspect`子命令从另一个终端检查该容器的数据卷：

```
**$ sudo docker inspect  e09979cacec8**
**. . . TRUNCATED OUTPUT . . .**
 **"Volumes": {**
 **"/DataMount": "/var/lib/docker/vfs/dir/62f5a3314999e5aaf485fc692ae07b3cbfacbca9815d8071f519c1a836c0f01e"**
**},**
 **"VolumesRW": {**
 **"/DataMount": true**
 **}**
**}**

```

显然，来自`datavol`数据专用容器的数据卷被挂载，就好像它们直接挂载到了这个容器上一样。

我们可以从另一个容器挂载数据卷，并展示挂载点。我们可以通过使用数据卷在容器之间共享数据来使挂载的数据卷工作，如下所示：

1.  让我们重用在上一个示例中启动的容器，并通过向数据卷`/DataMount`写入一些文本来创建一个`/DataMount/testfile`文件，如下所示：

```
**root@e09979cacec8:/# echo \**
 **"Data Sharing between Container" > \**
 **/DataMount/testfile** 

```

1.  只需将一个容器分离出来，以显示我们在上一步中编写的文本，使用`cat`命令：

```
**$ sudo docker run --rm \**
 **--volumes-from datavol \**
 **busybox:latest cat /DataMount/testfile**

```

以下是前述命令的典型输出：

```
**Data Sharing between Container**

```

显然，我们新容器化的`cat`命令的前面输出`容器之间的数据共享`是我们在步骤 1 中写入`/DataMount/testfile`的`datavol`容器中的文本。

很酷，不是吗？您可以通过共享数据卷在容器之间无缝共享数据。在这个例子中，我们使用数据专用容器作为数据共享的基础容器。然而，Docker 允许我们共享任何类型的数据卷，并且可以依次挂载数据卷，如下所示：

```
**$ sudo docker run --name vol1 --volumes-from datavol \**
 **busybox:latest /bin/true**
**$ sudo docker run --name vol2 --volumes-from vol1 \**
 **busybox:latest /bin/true**

```

在这里，在`vol1`容器中，我们可以挂载来自`datavol`容器的数据卷。然后，在`vol2`容器中，我们挂载了来自`vol1`容器的数据卷，这些数据卷最初来自`datavol`容器。

## 容器之间数据共享的实用性

在本章的前面，我们学习了从 Docker 主机访问 Apache2 HTTP 服务的日志文件的机制。虽然通过将 Docker 主机目录挂载到容器中方便地共享数据，但后来我们意识到可以通过仅使用数据卷在容器之间共享数据。因此，在这里，我们通过在容器之间共享数据来改变 Apache2 HTTP 服务日志处理的方法。为了在容器之间共享日志文件，我们将按照以下步骤启动以下容器：

1.  首先，一个仅用于数据的容器，将向其他容器公开数据卷。

1.  然后，一个利用数据专用容器的数据卷的 Apache2 HTTP 服务容器。

1.  一个用于查看我们 Apache2 HTTP 服务生成的日志文件的容器。

### 注意

注意：如果您在 Docker 主机机器的端口号`80`上运行任何 HTTP 服务，请为以下示例选择任何其他未使用的端口号。如果没有，请先停止 HTTP 服务，然后按照示例进行操作，以避免任何端口冲突。

现在，我们将逐步为您介绍如何制作相应的镜像并启动容器以查看日志文件，如下所示：

1.  在这里，我们首先使用`VOLUME`指令使用`/var/log/apache2`数据卷来制作`Dockerfile`。`/var/log/apache2`数据卷是对`Dockerfile`中第六章中设置的环境变量`APACHE_LOG_DIR`的直接映射，使用`ENV`指令：

```
#######################################################
# Dockerfile to build a LOG Volume for Apache2 Service
#######################################################
# Base image is BusyBox
FROM busybox:latest
# Author: Dr. Peter
MAINTAINER Dr. Peter <peterindia@gmail.com>
# Create a data volume at /var/log/apache2, which is
# same as the log directory PATH set for the apache image
VOLUME /var/log/apache2
# Execute command true
CMD ["/bin/true"]
```

由于这个`Dockerfile`是用来启动数据仅容器的，所以默认的执行命令被设置为`/bin/true`。

1.  我们将继续使用`docker build`从上述`Dockerfile`构建一个名为`apache2log`的 Docker 镜像，如下所示：

```
**$ sudo docker build -t apache2log .**
**Sending build context to Docker daemon  2.56 kB**
**Sending build context to Docker daemon**
**Step 0 : FROM busybox:latest**
**... TRUNCATED OUTPUT ...**

```

1.  使用`docker run`子命令从`apache2log`镜像启动一个仅数据的容器，并将生成的容器命名为`log_vol`，使用`--name`选项：

```
**$ sudo docker run --name log_vol apache2log**

```

根据上述命令，容器将在`/var/log/apache2`中创建一个数据卷并将其移至停止状态。

1.  与此同时，您可以使用`-a`选项运行`docker ps`子命令来验证容器的状态：

```
**$ sudo docker ps -a**
**CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS                      PORTS                NAMES**
**40332e5fa0ae        apache2log:latest   "/bin/true"            2 minutes ago      Exited (0) 2 minutes ago                        log_vol**

```

根据输出，容器以退出值`0`退出。

1.  使用`docker run`子命令启动 Apache2 HTTP 服务。在这里，我们重用了我们在第六章中制作的`apache2`镜像，*在容器中运行服务*。在这个容器中，我们将使用`--volumes-from`选项从我们在第 3 步中启动的数据仅容器`log_vol`挂载`/var/log/apache2`数据卷：

```
**$ sudo docker run -d -p 80:80 \**
 **--volumes-from log_vol \**
 **apache2**
**7dfbf87e341c320a12c1baae14bff2840e64afcd082dda3094e7cb0a0023cf42**

```

成功启动了从`log_vol`挂载的`/var/log/apache2`数据卷的 Apache2 HTTP 服务后，我们可以使用临时容器访问日志文件。

1.  在这里，我们使用临时容器列出了 Apache2 HTTP 服务存储的文件。这个临时容器是通过从`log_vol`挂载`/var/log/apache2`数据卷而产生的，并且使用`ls`命令列出了`/var/log/apache2`中的文件。此外，`docker run`子命令的`--rm`选项用于在执行完`ls`命令后删除容器：

```
**$  sudo docker run --rm \**
 **--volumes-from log_vol**
 **busybox:latest ls -l /var/log/apache2**
**total 4**
**-rw-r--r--    1 root     root             0 Dec  5 15:27 access.log**
**-rw-r--r--    1 root     root           461 Dec  5 15:27 error.log**
**-rw-r--r--    1 root     root             0 Dec  5 15:27 other_vhosts_access.log**

```

1.  最后，通过使用`tail`命令访问 Apache2 HTTP 服务生成的错误日志，如下命令所示：

```
**$ sudo docker run  --rm  \**
 **--volumes-from log_vol \**
 **ubuntu:14.04 \**
 **tail /var/log/apache2/error.log**
**AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 172.17.0.24\. Set the 'ServerName' directive globally to suppress this message**
**[Fri Dec 05 17:28:12.358034 2014] [mpm_event:notice] [pid 18:tid 140689145714560] AH00489: Apache/2.4.7 (Ubuntu) configured -- resuming normal operations**
**[Fri Dec 05 17:28:12.358306 2014] [core:notice] [pid 18:tid 140689145714560] AH00094: Command line: '/usr/sbin/apache2 -D FOREGROUND'**

```

# 避免常见陷阱

到目前为止，我们讨论了如何有效地使用数据卷在 Docker 主机和容器之间以及容器之间共享数据。使用数据卷进行数据共享正在成为 Docker 范式中非常强大和必不可少的工具。然而，它确实存在一些需要仔细识别和消除的缺陷。在本节中，我们尝试列出与数据共享相关的一些常见问题以及克服这些问题的方法和手段。

## 目录泄漏

在数据卷部分，我们了解到 Docker 引擎会根据`Dockerfile`中的`VOLUME`指令以及`docker run`子命令的`-v`选项自动创建目录。我们也明白 Docker 引擎不会自动删除这些自动生成的目录，以保留容器内运行的应用程序的状态。我们可以使用`docker rm`子命令的`-v`选项强制 Docker 删除这些目录。手动删除的过程会带来以下两个主要挑战：

1.  **未删除的目录：** 可能会出现这样的情况，您可能有意或无意地选择不删除生成的目录，而删除容器。

1.  **第三方镜像：** 我们经常利用第三方 Docker 镜像，这些镜像可能已经使用了`VOLUME`指令进行构建。同样，我们可能也有自己的 Docker 镜像，其中包含了`VOLUME`。当我们使用这些 Docker 镜像启动容器时，Docker 引擎将自动生成指定的目录。由于我们不知道数据卷的创建，我们可能不会使用`-v`选项调用`docker rm`子命令来删除自动生成的目录。

在前面提到的情况中，一旦相关的容器被移除，就没有直接的方法来识别那些容器被移除的目录。以下是一些建议，可以避免这种问题：

+   始终使用`docker inspect`子命令检查 Docker 镜像，查看镜像中是否有数据卷。

+   始终使用`docker rm`子命令的`-v`选项来删除为容器创建的任何数据卷（目录）。即使数据卷被多个容器共享，仍然可以安全地使用`docker rm`子命令的`-v`选项，因为只有当共享该数据卷的最后一个容器被移除时，与数据卷关联的目录才会被删除。

+   无论出于何种原因，如果您选择保留自动生成的目录，您必须保留清晰的记录，以便以后可以删除它们。

+   实施一个审计框架，用于审计并找出没有任何容器关联的目录。

## 数据卷的不良影响

如前所述，Docker 允许我们在构建时使用`VOLUME`指令将数据卷刻录到 Docker 镜像中。然而，在构建过程中不应该使用数据卷来存储任何数据，否则会产生不良影响。

在本节中，我们将通过制作一个`Dockerfile`来演示在构建过程中使用数据卷的不良影响，然后通过构建这个`Dockerfile`来展示其影响：

以下是`Dockerfile`的详细信息：

1.  使用`Ubuntu 14.04`作为基础镜像构建镜像：

```
# Use Ubuntu as the base image
FROM ubuntu:14.04
```

1.  使用`VOLUME`指令创建一个`/MountPointDemo`数据卷：

```
VOLUME /MountPointDemo
```

1.  使用`RUN`指令在`/MountPointDemo`数据卷中创建一个文件：

```
RUN date > /MountPointDemo/date.txt
```

1.  使用`RUN`指令显示`/MountPointDemo`数据卷中的文件：

```
RUN cat /MountPointDemo/date.txt
```

继续使用`docker build`子命令从这个`Dockerfile`构建一个镜像，如下所示：

```
**$ sudo docker build -t testvol .**
**Sending build context to Docker daemon  2.56 kB**
**Sending build context to Docker daemon**
**Step 0 : FROM ubuntu:14.04**
 **---> 9bd07e480c5b**
**Step 1 : VOLUME /MountPointDemo**
 **---> Using cache**
 **---> e8b1799d4969**
**Step 2 : RUN date > /MountPointDemo/date.txt**
 **---> Using cache**
 **---> 8267e251a984**
**Step 3 : RUN cat /MountPointDemo/date.txt**
 **---> Running in a3e40444de2e**
**cat: /MountPointDemo/date.txt: No such file or directory**
**2014/12/07 11:32:36 The command [/bin/sh -c cat /MountPointDemo/date.txt] returned a non-zero code: 1**

```

在`docker build`子命令的先前输出中，您会注意到构建在第 3 步失败，因为它找不到在第 2 步创建的文件。显然，在第 3 步时创建的文件在第 2 步时消失了。这种不良影响是由 Docker 构建其镜像的方法造成的。了解 Docker 镜像构建过程将揭开这个谜团。

在构建过程中，对于`Dockerfile`中的每个指令，按照以下步骤进行：

1.  通过将`Dockerfile`指令转换为等效的`docker run`子命令来创建一个新的容器

1.  将新创建的容器提交为镜像

1.  通过将新创建的镜像视为第 1 步的基础镜像，重复执行第 1 步和第 2 步。

当容器被提交时，它保存容器的文件系统，并故意不保存数据卷的文件系统。因此，在此过程中存储在数据卷中的任何数据都将丢失。因此，在构建过程中永远不要使用数据卷作为存储。

# 总结

对于企业规模的分布式应用来说，数据是其运营和产出中最重要的工具和成分。通过 IT 容器化，这个旅程以迅速和明亮的方式开始。通过巧妙利用 Docker 引擎，IT 和业务软件解决方案都被智能地容器化。然而，最初的动机是更快速、无缺陷地实现应用感知的 Docker 容器，因此，数据与容器内的应用紧密耦合。然而，这种紧密性带来了一些真正的风险。如果应用程序崩溃，那么数据也会丢失。此外，多个应用程序可能依赖于相同的数据，因此数据必须进行共享。

在本章中，我们讨论了 Docker 引擎在促进 Docker 主机和容器之间以及容器之间无缝数据共享方面的能力。数据卷被规定为实现不断增长的 Docker 生态系统中各组成部分之间数据共享的基础构件。在下一章中，我们将解释容器编排背后的概念，并看看如何通过一些自动化工具简化这个复杂的方面。编排对于实现复合容器至关重要。

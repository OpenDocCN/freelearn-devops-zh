# 附录

# 1\. 运行我的第一个 Docker 容器

## 活动 1.01：从 Docker Hub 拉取并运行 PostgreSQL 容器映像

**解决方案**：

1.  要启动 Postgres Docker 容器，首先确定需要设置数据库的默认用户名和密码凭据的环境变量。通过阅读官方 Docker Hub 页面，您可以看到`POSTGRES_USER`和`POSTGRES_PASSWORD`环境变量的配置选项。使用`-e`标志传递环境变量。启动我们的 Postgres Docker 容器的最终命令如下：

[PRE0]

运行此命令将启动容器。

1.  执行`docker ps`命令以验证其正在运行并且健康：

[PRE1]

该命令应返回如下输出：

[PRE2]

从前面的输出可以看出，具有 ID`29f115af8cdd`的容器正在运行。

在这个活动中，您已成功启动了一个 PostgreSQL 版本 12 容器，该容器是 Panoramic Trekking App 的一部分，该应用程序将在本书的过程中构建。

## 活动 1.02：访问 Panoramic Trekking App 数据库

**解决方案**：

1.  使用`docker exec`登录到数据库实例，启动容器内的 PSQL shell，传递`--username`标志并将`--password`标志留空：

[PRE3]

这应提示您输入密码并启动 PSQL shell。

1.  使用`\l`命令列出所有数据库：

[PRE4]

将返回在容器中运行的数据库列表：

![图 1.4：数据库列表](img/B15021_01_04.jpg)

图 1.4：数据库列表

1.  最后，使用`\q`快捷方式退出 shell。

1.  使用`docker stop`和`docker rm`命令停止和清理容器实例。

在这个活动中，您通过使用在*Activity 1.01*中设置的凭据登录到容器中运行的数据库。您还列出了在容器中运行的数据库。该活动让您亲身体验了如何使用 PSQL shell 访问在任何容器中运行的数据库。

# 2\. 使用 Dockerfiles 入门

## 活动 2.01：在 Docker 容器上运行 PHP 应用程序

**解决方案**：

1.  为此活动创建一个名为`activity-02-01`的新目录：

[PRE5]

1.  导航到新创建的`activity-02-01`目录：

[PRE6]

1.  在`activity-02-01`目录中，创建一个名为`welcome.php`的文件：

[PRE7]

1.  现在，使用您喜欢的文本编辑器打开`welcome.php`：

[PRE8]

1.  创建 `welcome.php` 文件，并使用活动开始时提供的内容，然后保存并退出 `welcome.php` 文件：

[PRE9]

1.  在 `activity-02-01` 目录中，创建一个名为 `Dockerfile` 的文件：

[PRE10]

1.  现在，使用您喜爱的文本编辑器打开 `Dockerfile`：

[PRE11]

1.  将以下内容添加到 `Dockerfile` 中，然后保存并退出 `Dockerfile`：

[PRE12]

我们将使用 `ubuntu` 基础镜像开始这个 `Dockerfile`，然后设置一些标签。接下来，将 `DEBIAN_FRONTEND` 环境变量设置为 `noninteractive`，以使包安装变为非交互式。然后安装 `apache2`、`php` 和 `curl` 包，并将 PHP 文件复制到 `/var/www/html` 目录。接下来，配置健康检查并暴露端口 `80`。最后，使用 `apache2ctl` 命令启动 Apache web 服务器。

1.  现在，构建 Docker 镜像：

[PRE13]

运行 `build` 命令后，您应该会得到以下输出：

![图 2.22：构建活动-02-01 Docker 镜像](img/B15021_02_22.jpg)

图 2.22：构建活动-02-01 Docker 镜像

1.  执行 `docker container run` 命令以从您在上一步中构建的 Docker 镜像启动新容器：

[PRE14]

由于您是以分离模式（使用 `-d` 标志）启动 Docker 容器，上述命令将输出生成的 Docker 容器的 ID。

1.  现在，您应该能够查看 Apache 主页。在您喜爱的网络浏览器中转到 `http://127.0.0.1/welcome.php` 终端节点：![图 2.23：PHP 应用程序页面](img/B15021_02_23.jpg)

图 2.23：PHP 应用程序页面

请注意，默认的 Apache 主页是可见的。在前面的输出中，您收到了 `Good Morning` 的输出。此输出可能会有所不同，根据您运行此容器的时间，可能会显示为 `Good Afternoon` 或 `Good Evening`。

1.  现在，清理容器。首先，使用 `docker container stop` 命令停止 Docker 容器：

[PRE15]

1.  最后，使用 `docker container rm` 命令移除 Docker 容器：

[PRE16]

在这个活动中，我们学习了如何使用本章中迄今为止学到的 `Dockerfile` 指令来将示例 PHP 应用程序 docker 化。我们使用了多个 `Dockerfile` 指令，包括 `FROM`、`LABEL`、`ENV`、`RUN`、`COPY`、`WORKDIR`、`HEALTHCHECK`、`EXPOSE` 和 `ENTRYPOINT`。

# 3\. 管理您的 Docker 镜像

## 活动 3.01：使用 Git 哈希版本控制构建脚本

**解决方案**：

有多种方法可以完成这个活动。以下是一个例子：

1.  创建一个新的构建脚本。第一行显示设置`–ex`命令，将每个步骤打印到屏幕上，并且如果任何步骤失败，脚本将失败。*第 3 行*和*第 4 行*设置了你的注册表和服务名称的变量：

[PRE17]

1.  在*第 6 行*，将`GIT_VERSION`变量设置为指向你的短 Git 提交哈希。构建脚本然后在*第 7 行*将这个值打印到屏幕上：

[PRE18]

1.  在*第 9 行*使用`docker build`命令创建你的新镜像，并在*第 11 行*添加`docker push`命令将镜像推送到你的本地 Docker 注册表：

[PRE19]

脚本文件将如下所示：

[PRE20]

1.  运行以下命令来确保脚本已构建并成功运行：

[PRE21]

你应该会得到以下输出：

[PRE22]

## 活动 3.02：配置本地 Docker 注册表存储

**解决方案**：

以下步骤描述了实现活动目标的一种方式：

1.  在你的主目录中创建`test_registry`目录：

[PRE23]

1.  运行本地注册表，但在这种情况下，包括`-v`选项，将你在前一步创建的目录连接到`/var/lib/registry`容器目录。还要使用`:rw`选项确保你可以读写该目录：

[PRE24]

1.  现在，像往常一样将镜像推送到新挂载的注册表中：

[PRE25]

1.  为了验证镜像现在是否存储在你新挂载的目录中，列出`registry/docker/registry/v2/repositories/`目录中的文件。

[PRE26]

你应该会看到你刚刚在上一步推送的新镜像：

[PRE27]

这个活动让我们开始使用一些更高级的 Docker 选项。别担心，将会有更多章节专门帮助你理解在运行容器时的卷挂载和存储。

# 4\. 多阶段 Docker 文件

## 活动 4.01：使用多阶段 Docker 构建部署 Golang HTTP 服务器

解决方案：

1.  为这个活动创建一个名为`activity-04-01`的新目录：

[PRE28]

1.  导航到新创建的`activity-04-01`目录：

[PRE29]

1.  在`activity-04-01`目录中，创建一个名为`main.go`的文件：

[PRE30]

1.  现在，使用你喜欢的文本编辑器打开`main.go`文件：

[PRE31]

1.  将以下内容添加到`main.go`文件中，然后保存并退出该文件：

[PRE32]

1.  在`activity-04-01`目录中，创建一个名为`Dockerfile`的文件。这个文件将是多阶段`Dockerfile`：

[PRE33]

1.  现在，使用你喜欢的文本编辑器打开`Dockerfile`：

[PRE34]

1.  将以下内容添加到`Dockerfile`并保存文件：

[PRE35]

这个`Dockerfile`有两个阶段，名为`builder`和`runtime`。构建阶段使用 Golang Docker 镜像作为父镜像，负责从 Golang 源文件创建可执行文件。运行时阶段使用`alpine` Docker 镜像作为父镜像，并执行从`builder`阶段复制的可执行文件。

1.  现在，使用`docker build`命令构建 Docker 镜像：

[PRE36]

您应该会得到以下输出：

![图 4.14：构建 Docker 镜像](img/B15021_04_14.jpg)

图 4.14：构建 Docker 镜像

1.  使用`docker image` ls 命令列出计算机上所有可用的 Docker 镜像。验证镜像的大小：

[PRE37]

该命令将返回所有可用的 Docker 镜像列表：

![图 4.15：列出所有 Docker 镜像](img/B15021_04_15.jpg)

图 4.15：列出所有 Docker 镜像

在前面的输出中，您可以看到名为`activity-04-01`的优化 Docker 镜像的大小为 13.1 MB，而在构建阶段使用的父镜像（Golang 镜像）的大小为 370 MB。

1.  执行`docker container run`命令，从您在上一步中构建的 Docker 镜像启动一个新的容器：

[PRE38]

您应该会得到类似以下的输出：

[PRE39]

1.  在您喜欢的网络浏览器中查看以下 URL 的应用程序：

[PRE40]

当我们导航到 URL `http://127.0.0.1:8080/`时，以下图片显示了主页：

![图 4.16：Golang 应用程序-主页](img/B15021_04_16.jpg)

图 4.16：Golang 应用程序-主页

1.  现在，在您喜欢的网络浏览器中浏览以下 URL：

[PRE41]

当我们导航到 URL `http://127.0.0.1:8080/contact`时，以下图片显示了联系页面：

![图 4.17：Golang 应用程序-联系我们页面](img/B15021_04_17.jpg)

图 4.17：Golang 应用程序-联系我们页面

1.  现在，在您喜欢的网络浏览器中输入以下 URL：

[PRE42]

当我们导航到 URL `http://127.0.0.1:8080/login`时，以下图片显示了登录页面：

![图 4.18：Golang 应用程序-登录页面](img/B15021_04_18.jpg)

图 4.18：Golang 应用程序-登录页面

在这个活动中，我们学习了如何部署一个 Golang HTTP 服务器，它可以根据调用 URL 返回不同的响应。在这个活动中，我们使用了多阶段的 Docker 构建来创建一个最小尺寸的 Docker 镜像。

# 5. 使用 Docker Compose 组合环境

## 活动 5.01：使用 Docker Compose 安装 WordPress

**解决方案**：

可以通过以下步骤创建数据库并安装 WordPress：

1.  创建所需的目录并使用`cd`命令进入其中：

[PRE43]

1.  创建一个名为`docker-compose.yaml`的文件，内容如下：

[PRE44]

1.  使用`docker-compose up --detach`命令启动应用程序：![图 5.22：应用程序的启动](img/B15021_05_22.jpg)

图 5.22：应用程序的启动

1.  使用`docker-compose ps`命令检查正在运行的容器。您应该会得到以下输出：![图 5.23：WordPress 和数据库容器](img/B15021_05_23.jpg)

图 5.23：WordPress 和数据库容器

1.  在浏览器中打开`http://localhost:8080`以检查 WordPress 设置屏幕：![图 5.24：WordPress 设置屏幕](img/B15021_05_24.jpg)

图 5.24：WordPress 设置屏幕

在这个活动中，您使用 Docker Compose 创建了一个真实应用程序的部署。该应用程序包括一个数据库容器和一个 WordPress 容器。这两个容器服务都使用环境变量进行配置，通过 Docker Compose 网络和卷进行连接。

## 活动 5.02：使用 Docker Compose 安装全景徒步应用程序

**解决方案**：

可以通过以下步骤创建数据库和全景徒步应用程序：

1.  创建所需的目录并切换到其中：

[PRE45]

1.  创建一个名为`docker-compose.yaml`的文件，内容如下：

[PRE46]

1.  使用`docker-compose up --detach`命令启动应用程序。您应该会得到类似以下的输出：![图 5.25：应用程序的启动](img/B15021_05_25.jpg)

图 5.25：应用程序的启动

注意

您也可以使用`docker-compose up -d`命令来启动应用程序。

1.  使用`docker-compose ps`命令检查正在运行的容器。您应该会得到类似以下的输出：![图 5.26 应用程序、数据库和 nginx 容器](img/B15021_05_26.jpg)

图 5.26 应用程序、数据库和 nginx 容器

1.  使用地址`http://0.0.0.0:8000/admin`在浏览器中打开全景徒步应用程序的管理部分：![图 5.27：管理员设置登录](img/B15021_05_27.jpg)

图 5.27：管理员设置登录

注意

您也可以运行`firefox http://0.0.0.0:8000/admin`命令来打开全景徒步应用程序的管理部分。

使用用户名`admin`和密码`changeme`登录，并添加新的照片和国家。将出现以下屏幕：

![图 5.28：管理员设置视图](img/B15021_05_28.jpg)

图 5.28：管理员设置视图

1.  在浏览器中打开全景徒步应用程序，地址为`http://0.0.0.0:8000/photo_viewer`：![图 5.29：应用程序视图](img/B15021_05_29.jpg)

图 5.29：应用程序视图

在此活动中，您使用 Docker Compose 创建了一个三层应用程序，其中包括用于 PostgreSQL 数据库、后端和代理服务的层。所有服务都使用 Docker Compose 进行配置和连接，具有其网络和存储功能。

# 6. Docker 网络简介

## 活动 6.01：利用 Docker 网络驱动程序

**解决方案**：

以下是根据最佳实践完成此活动的最常见方法：

1.  使用`docker network create`命令为 NGINX Web 服务器创建一个网络。将其命名为`webservernet`，并为其分配子网`192.168.1.0/24`和网关`192.168.1.1`：

[PRE47]

这应该创建`bridge`网络`webservernet`。

1.  使用`docker run`命令创建一个 NGINX Web 服务器。使用`-p`标志将主机上的端口`8080`转发到容器实例上的端口`80`：

[PRE48]

这将在`webservernet`网络中启动`webserver1`容器。

1.  使用`docker run`命令以`host`网络模式启动名为`monitor`的 Alpine Linux 容器。这样，您将知道容器可以访问主系统的主机端口以及`bridge`网络的 IP 地址：

[PRE49]

这将在`host`网络模式下启动一个 Alpine Linux 容器实例。

1.  使用`docker inspect`查找`webserver1`容器的 IP 地址：

[PRE50]

容器的详细信息将以 JSON 格式显示；从`IPAddress`参数中获取 IP 地址：

![图 6.27：检查 webserver1 容器实例](img/B15021_06_27.jpg)

图 6.27：检查 webserver1 容器实例

1.  使用`docker exec`命令在监控容器内部启动`sh` shell：

[PRE51]

这应该将您放入根 shell。

1.  使用`apk install`命令在此容器内安装`curl`命令：

[PRE52]

这应该安装`curl`实用程序：

[PRE53]

1.  使用`curl`命令验证主机级别的连接是否正常工作，调用主机机器上的端口`8080`：

[PRE54]

您应该收到来自 NGINX 的`200 OK`响应，表示在主机级别成功连接：

![图 6.28：从主机上的暴露端口访问 webserver1 容器](img/B15021_06_28.jpg)

图 6.28：从主机上的暴露端口访问 webserver1 容器

1.  同样，使用`curl`命令直接通过端口`80`访问 Docker`bridge`网络中容器的 IP 地址：

[PRE55]

您应该同样收到另一个`200 OK`响应，表明连接成功：

![图 6.29：从 IP 访问 NGINX Web 服务器容器实例的地址](img/B15021_06_29.jpg)

图 6.29：从容器实例的 IP 地址访问 NGINX Web 服务器

在这个活动中，我们能够说明使用不同的 Docker 网络驱动程序在容器之间建立连接。这种情况适用于真实的生产基础设施，因为在部署容器化解决方案时，工程师们将努力部署尽可能不可变的基础设施。通过在 Docker 中部署容器，确切地模拟主机级别的网络，可以设计出需要在主机操作系统上进行非常少量配置的基础设施。这使得在部署和扩展 Docker 部署的主机时非常容易。诸如`curl`和其他监控工具的软件包可以部署到在 Docker 主机上运行的容器中，而不是安装在主机上。这保证了部署和维护的便利性，同时提高了满足不断增长的需求所需的主机部署速度。

## 活动 6.02：叠加网络实践

**解决方案**：

1.  在 Docker swarm 集群中的`Machine1`上使用`docker network create`命令创建一个名为`panoramic-net`的 Docker`overlay`网络，通过传递自定义的`subnet`、`gateway`和`overlay`网络驱动程序：

[PRE56]

1.  在`Machine1`上使用`docker service create`命令创建一个名为`trekking-app`的服务，加入`panoramic-net`网络：

[PRE57]

这将在`panoramic-net` `overlay`网络中启动一个名为`trekking-app`的服务。

1.  在`Machine1`上使用`docker service create`命令创建一个名为`database-app`的服务，加入`panoramic-net`网络。设置默认凭据并指定`postgres:12`版本的 Docker 镜像：

[PRE58]

1.  使用`docker exec`访问`trekking-app`服务容器内的`sh` shell：

[PRE59]

这应该将您放入`trekking-app`容器实例内的根 shell 中。

1.  使用`ping`命令验证对`database-app`服务的网络连接：

[PRE60]

ICMP 回复应指示连接成功：

[PRE61]

在这个活动中，我们利用了 Docker 集群中的自定义 Docker`overlay`网络，以说明两个 Docker 集群服务之间的连接性，使用 Docker DNS。在真实的多层应用程序中，许多微服务可以部署在使用`overlay`网络网格直接相互通信的大型 Docker 集群中。了解`overlay`网络如何与 Docker DNS 协同工作对于实现容器化基础设施的高效可扩展性至关重要。

# 7\. Docker 存储

## 活动 7.01：将容器事件（状态）数据存储在 PostgreSQL 数据库中

**解决方案**：

1.  运行以下命令以删除主机中的所有对象：

[PRE62]

1.  获取卷名称，然后使用以下命令删除所有卷：

[PRE63]

1.  获取网络名称，然后使用以下命令删除所有网络：

[PRE64]

1.  打开两个终端，一个专门用于查看`docker events --format '{{json .}}'`的效果。另一个应该打开以执行先前提到的高级步骤。

1.  在第一个终端中，运行以下命令：

[PRE65]

您应该得到以下输出：

![图 7.11：docker events 命令的输出](img/B15021_07_11.jpg)

图 7.11：docker events 命令的输出

1.  运行以下命令在第二个终端中启动`ubuntu`容器：

[PRE66]

您应该得到以下输出：

![图 7.12：docker run 命令的输出](img/B15021_07_12.jpg)

图 7.12：docker run 命令的输出

1.  使用第二个终端中的以下命令创建名为`vol1`的卷：

[PRE67]

1.  使用第二个终端中的以下命令创建名为`net1`的网络：

[PRE68]

1.  使用以下命令删除容器：

[PRE69]

1.  使用以下命令删除卷和网络：

[PRE70]

1.  在`docker events`终端中单击*Ctrl* + *C*以终止它。

1.  检查以下两个示例以了解 JSON 输出：

**示例 1**：

[PRE71]

**示例 2**：

[PRE72]

您会发现根据对象的不同属性和结构而有所不同。

1.  运行带有卷的 PostgreSQL 容器。将容器命名为`db1`：

[PRE73]

1.  运行`exec`命令，以便 bash 被要执行的命令替换。shell 将更改为`posgres=#`，表示您在容器内部：

[PRE74]

1.  创建一个具有两列的表：`ID`为`serial`类型，`info`为`json`类型：

[PRE75]

1.  将第一个示例的`JSON`输出的第一行插入表中：

[PRE76]

1.  通过输入以下 SQL 语句验证数据库中是否保存了行：

[PRE77]

您应该得到以下输出：

![图 7.13：验证数据库中是否保存了行](img/B15021_07_13.jpg)

图 7.13：验证数据库中是否保存了行

1.  使用 SQL 的`insert`命令将 Docker 事件插入`events`表中。

注意

请参考[`packt.live/2ZKfGgB`](https://packt.live/2ZKfGgB)上的`events.txt`文件，使用`insert`命令插入 Docker 事件。

您应该得到以下输出：

![图 7.14：在数据库中插入多行](img/B15021_07_14.jpg)

图 7.14：在数据库中插入多行

从这个输出中，可以清楚地看到已成功将 11 个事件插入到 PostgreSQL 数据库中。

1.  依次运行以下三个查询。

**查询 1**：

[PRE78]

输出将如下所示：

![图 7.15：查询 1 的输出](img/B15021_07_08.jpg)

图 7.15：查询 1 的输出

**查询 2**：

[PRE79]

输出将如下所示：

![图 7.16：查询 2 的输出](img/B15021_07_16.jpg)

图 7.16：查询 2 的输出

**查询 3**：

[PRE80]

输出将如下所示：

![图 7.17：查询 3 的输出](img/B15021_07_17.jpg)

图 7.17：查询 3 的输出

在这个活动中，您学习了如何记录和监视容器，并使用 SQL 语句查询容器的事件，以及如何获得事件的 JSON 输出并保存在 PostgreSQL 数据库中。您还学习了 JSON 输出结构以及如何查询它。

## 活动 7.02：与主机共享 NGINX 日志文件

**解决方案**：

1.  通过运行以下命令验证您的主机上是否没有`/var/mylogs`文件夹：

[PRE81]

您应该得到以下输出：

[PRE82]

1.  基于 NGINX 镜像运行一个容器。在`run`命令中指定主机和容器内共享卷的路径。在容器内，NGINX 使用`/var/log/nginx`路径存储日志文件。在主机上指定路径为`/var/mylogs`：

[PRE83]

如果您本地没有该镜像，Docker 引擎将自动拉取该镜像：

![图 7.18：运行 docker run 命令的输出](img/B15021_07_18.jpg)

图 7.18：运行 docker run 命令的输出

1.  进入`/var/mylogs`路径。列出该目录中的所有文件：

[PRE84]

您应该在那里找到两个文件：

[PRE85]

1.  （可选）如果没有生成错误，这两个文件将是空的。你可以使用`cat` Linux 命令或者使用`tail` Linux 命令来检查内容。因为我们之前使用了`cat`命令，所以让我们在这个例子中使用`tail`命令：

[PRE86]

你应该得到以下的输出：

[PRE87]

由于这个 NGINX 服务器没有生成任何错误或者没有被访问，这些文件目前是空的。然而，如果 NGINX 在任何时刻崩溃，生成的错误将会保存在`error.log`中。

在这个活动中，你学会了如何将容器的日志文件共享到主机上。你使用了 NGINX 服务器，所以如果它崩溃了，你可以从它的日志文件中追溯发生了什么。

# 8\. 服务发现

## 活动 8.01：利用 Jenkins 和 SonarQube

**解决方案**：

1.  安装 SonarQube 并使用以下命令作为容器运行：

[PRE88]

你应该得到容器 ID 作为输出：

[PRE89]

1.  使用`admin/admin`凭据登录到 SonarQube：![图 8.38：登录到 SonarQube](img/B15021_08_38.jpg)

图 8.38：登录到 SonarQube

成功登录后，应该出现类似以下的屏幕：

![图 8.39：SonarQube 仪表板](img/B15021_08_39.jpg)

图 8.39：SonarQube 仪表板

1.  在右上角，点击用户。会出现一个下拉菜单。点击`我的账户`：

1.  向下滚动并点击`安全`下的`生成`来生成一个令牌。你现在必须复制它，因为以后将无法访问它：![图 8.40：生成令牌](img/B15021_08_40.jpg)

图 8.40：生成令牌

1.  在 Jenkins 中，点击`管理 Jenkins` > `插件管理器`。在`可用`列表中搜索`Sonar`。安装`SonarQube Scanner`插件。![图 8.41：安装 SonarQube Scanner 插件](img/B15021_08_41.jpg)

图 8.41：安装 SonarQube Scanner 插件

1.  通过点击`hit_count`项目，然后点击`配置`选项来验证安装是否正确。在`构建`选项卡上点击`添加构建步骤`，然后点击`执行 SonarQube Scanner`，就像*图 8.43*中那样：![图 8.42：选择执行 SonarQube Scanner](img/B15021_08_42.jpg)

图 8.42：选择执行 SonarQube Scanner

1.  然而，新的框将会生成错误，就像下面截图中显示的那样。为了纠正这个问题，通过`系统配置`和`全局工具配置`选项将 SonarQube 和 Jenkins 集成起来：![图 8.43：由于 SonarQube 尚未配置而生成的错误](img/B15021_08_43.jpg)

图 8.43：由于 SonarQube 尚未配置而生成的错误

1.  在 Jenkins 中，点击“管理 Jenkins”。点击“全局工具配置”选项，然后点击“添加 SonarQube 扫描仪”：![图 8.44：在全局工具配置页面上添加 SonarQube 扫描仪](img/B15021_08_44.jpg)

图 8.44：在全局工具配置页面上添加 SonarQube 扫描仪

1.  输入名称“SonarQube 扫描仪”。勾选“自动安装”。在“从 Maven 中央安装”下，在“版本”中选择`SonarQube Scanner 3.2.0.1227`。点击“添加安装程序”。在“标签”字段中，输入`SonarQube`。在“二进制存档的下载 URL”字段中，输入链接`https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-3.2.0.1227-linux.zip`。

点击“保存”。

![图 8.45：为 SonarQube 扫描仪添加详细信息](img/B15021_08_45.jpg)

图 8.45：为 SonarQube 扫描仪添加详细信息

您现在已完成“全局工具配置”选项，现在是时候转到“配置系统”选项了。

1.  在“管理 Jenkins”中，点击“配置系统”：![图 8.46：在管理 Jenkins 页面上点击配置系统](img/B15021_08_46.jpg)

图 8.46：在管理 Jenkins 页面上点击配置系统

1.  您现在无法进入系统配置，因为它要求“服务器身份验证令牌”。当您点击“添加”按钮时，它将不起作用。在以下步骤中将令牌输入为秘密文本，然后返回到“管理 Jenkins”：![图 8.47：在 Jenkins 配置中插入 SonarQube 令牌](img/B15021_08_47.jpg)

图 8.47：在 Jenkins 配置中插入 SonarQube 令牌

1.  点击“管理凭据”：![图 8.48：管理 Jenkins 页面](img/B15021_08_48.jpg)

图 8.48：管理 Jenkins 页面

1.  点击`Jenkins`：![图 8.49：Jenkins 凭据页面](img/B15021_08_49.jpg)

图 8.49：Jenkins 凭据页面

1.  点击“全局凭据（不受限制）”：![图 8.50：全局凭据（不受限制）域](img/B15021_08_50.jpg)

图 8.50：全局凭据（不受限制）域

1.  点击“添加一些凭据”：![图 8.51：添加一些凭据](img/B15021_08_51.jpg)

图 8.51：添加一些凭据

1.  在“类型”下拉菜单中，点击“秘密文本”：![图 8.52：选择类型为秘密文本](img/B15021_08_52.jpg)

图 8.52：选择类型为秘密文本

1.  在“秘密”文本框中，粘贴您在本活动的*步骤 5*中复制的令牌。在`ID`字段中，输入`SonarQubeToken`。点击“确定”：![图 8.53：将令牌添加到秘密文本框](img/B15021_08_53.jpg)

图 8.53：将令牌添加到秘密文本框

`SonarQubeToken`将保存在“全局凭据”选项中。您将看到类似以下内容的屏幕：

![图 8.54：SonarQubeToken 保存在全局凭据中](img/B15021_08_54.jpg)

图 8.54：SonarQubeToken 保存在全局凭据中

1.  返回到“管理 Jenkins”。点击“配置系统”，然后点击“刷新”。现在，在“服务器身份验证令牌”下拉菜单中，您将找到`SonarQubeToken`。勾选“启用将 SonarQube 服务器配置注入构建环境变量”。在“名称”字段中输入`SonarQube`。在“服务器 URL”字段中输入“http://<您的 IP>:9000”。然后点击“保存”：

您可以运行`ifconfig`命令来获取您的 IP。您将在输出的`en0`部分找到 IP：

[PRE90]

这是将 Jenkins 与 SonarQube 集成的最后一步。让我们返回到项目中。

1.  在“构建环境”中，勾选“准备 SonarQube 扫描器环境”。将“服务器身份验证令牌”设置为`SonarQubeToken`：

1.  现在，点击项目名称，然后点击“配置”。在“构建”步骤中，在“分析属性”字段中输入以下代码：

[PRE91]

点击“保存”。

1.  保存后，您将在项目页面上看到 SonarQube 标志，如*图 8.55*所示。点击“立即构建”：![图 8.55：我们项目仪表板上显示 SonarQube 选项](img/B15021_08_55.jpg)

图 8.55：我们项目仪表板上显示 SonarQube 选项

1.  在“构建历史”中，点击“控制台输出”。您应该会看到类似以下内容的屏幕：![图 8.56：控制台输出](img/B15021_08_56.jpg)

图 8.56：控制台输出

1.  在浏览器中检查`SonarQube`的报告。输入`http://<ip>:9000`或`http://localhost:9000`。您将发现 Jenkins 自动将您的`hit_count`项目添加到 SonarQube 中：

1.  点击`hit_count`。您将找到一个详细的报告。每当 Jenkins 构建项目时，SonarQube 将自动分析代码。

在本活动中，您学习了如何将 Jenkins 与 SonarQube 集成并安装所需的插件，通过在浏览器中检查 SonarQube 进行验证。您还将 SonarQube 应用于您的简单 Web 应用程序`hit_counter`。

## 活动 8.02：在全景徒步应用程序中利用 Jenkins 和 SonarQube

**解决方案**：

1.  在 Jenkins 中创建一个名为`trekking`的新项目。将其选择为`FREESTYLE`项目。点击“确定”。

1.  在“常规”选项卡中，选择“丢弃旧构建”。

1.  在“源代码管理”选项卡中，选择`GIT`。然后输入 URL`http://github.com/efoda/trekking_app`：![图 8.57：插入 GitHub URL](img/B15021_08_57.jpg)

图 8.57：插入 GitHub URL

1.  在“构建触发器”中，选择“轮询 SCM”，并输入`H/15 * * * *`：![图 8.58：插入调度代码](img/B15021_08_58.jpg)

图 8.58：插入调度代码

1.  在“构建环境”选项卡中，选择“准备 SonarQube 扫描器环境”。从下拉菜单中选择“服务器身份验证令牌”：![图 8.59：选择 SonarQubeToken 作为服务器身份验证令牌](img/B15021_08_59.jpg)

图 8.59：选择 SonarQubeToken 作为服务器身份验证令牌

1.  在“构建”选项卡中，在“分析属性”中输入以下代码：

[PRE92]

点击“保存”。

1.  选择“立即构建”。当构建成功完成时，选择“控制台输出”。以下输出将指示它已成功完成：![图 8.60：验证 Jenkins 已成功构建镜像](img/B15021_08_60.jpg)

图 8.60：验证 Jenkins 已成功构建镜像

1.  切换到浏览器中的`SonarQube`选项卡，并检查输出。以下报告表明徒步应用程序有两个错误和零个安全漏洞：![图 8.61：在 SonarQube 浏览器选项卡中显示的报告](img/B15021_08_61.jpg)

图 8.61：在 SonarQube 浏览器选项卡中显示的报告

如果单击“新代码”，它将为空，因为您只构建了项目一次。当 Jenkins 再次构建它时，您将找到两次构建之间的比较。

1.  如果您想编辑项目的代码，请将 GitHub 代码 fork 到您的帐户，并编辑代码以修复错误和漏洞。编辑项目的配置，使其使用您的 GitHub 代码，而不是“源代码”选项卡中提供的代码。

在这个活动中，您将 Jenkins 与 SonarQube 集成，并将其应用于全景徒步应用程序。在活动结束时，您将检查 SonarQube 生成的报告，显示代码中的错误和漏洞。

# 9. Docker Swarm

## 活动 9.01：将全景徒步应用部署到单节点 Docker Swarm

**解决方案**：

有许多方法可以执行此活动。以下步骤是其中一种方法：

1.  为应用程序创建一个目录。在这种情况下，您将创建一个名为`Activity1`的目录，并使用`cd`命令进入新目录：

[PRE93]

1.  从其 GitHub 存储库克隆应用程序，以确保您拥有部署到 Swarm 的 Panoramic Trekking App 服务所需的所有相关信息和应用程序：

[PRE94]

1.  您不需要 NGINX 的任何支持目录，但请确保您的 Web 服务和运行的数据库在此处列出，包括`panoramic_trekking_app`和`photo_viewer`目录以及`Dockerfile`、`entrypoint.sh`、`manage.py`和`requirements.txt`脚本和文件：

[PRE95]

该命令应返回类似以下的输出：

[PRE96]

1.  在目录中创建`.env.dev`文件，并添加以下详细信息，供`panoramic_trekking_app`在其`settings.py`文件中使用。这些环境变量将设置数据库名称、用户、密码和其他数据库设置：

[PRE97]

1.  创建一个新的`docker-compose.yml`文件，并用文本编辑器打开它，并添加以下详细信息：

[PRE98]

如您从`docker-compose.yml`文件中的突出显示的行中所见，`web`服务依赖于`activity_web:latest` Docker 镜像。

1.  运行以下`docker build`命令来构建镜像并适当地标记它：

[PRE99]

1.  现在是时候将堆栈部署到 Swarm 了。使用您创建的`docker-compose.yml`文件运行以下`stack deploy`命令：

[PRE100]

创建网络后，您应该看到`activity_swarm_web`和`activity_swarm_db`服务可用：

[PRE101]

1.  运行`service ls`命令：

[PRE102]

验证所有服务是否已成功启动，并显示`1/1`副本，就像我们这里一样：

[PRE103]

1.  最后，打开您的网络浏览器，并验证您能够从`http://localhost:8000/admin/`和`http://localhost:8000/photo_viewer/`访问该网站。

Panoramic Trekking App 的创建和设置方式与本章中已经完成的一些其他服务类似。

## 活动 9.02：在 Swarm 运行时执行应用程序更新

解决方案：

有许多种方法可以执行此活动。以下步骤详细说明了一种方法：

1.  如果您没有运行 Swarm，请部署您在*活动 9.01*中创建的`docker-compose.yml`文件，*将 Panoramic Trekking App 部署到单节点 Docker Swarm*：

[PRE104]

如您所见，现在所有三个服务都在运行：

[PRE105]

1.  在执行`stack deploy`命令的同一目录中，使用文本编辑器打开`photo_viewer/templates/photo_index.html`文件，并将第四行更改为与以下详细信息匹配，基本上是在主标题中添加单词`Patch`：

photo_index.html

[PRE106]

您可以在此处找到完整的代码[`packt.live/3ceYnta`](https://packt.live/3ceYnta)。

1.  构建一个新图像，这次使用以下命令将图像标记为`patch_1`：

[PRE107]

1.  使用`service update`命令将补丁部署到您的 Swarm Web 服务。还提供要应用更新的图像名称和服务：

[PRE108]

输出应如下所示：

[PRE109]

1.  列出正在运行的服务，并验证新图像是否作为`activity_swarm_web`服务的一部分正在运行：

[PRE110]

从输出中可以看出，Web 服务不再使用`latest`标记。它现在显示`patch_1`图像标记：

[PRE111]

1.  通过访问`http://localhost:8000/photo_viewer/`并查看标题现在显示为`Patch Panoramic Trekking App`来验证更改是否已应用于图像：![图 9.10：全景徒步应用程序的 Patch 版本](img/B15021_09_10.jpg)

图 9.10：全景徒步应用程序的 Patch 版本

在此活动中，您对全景徒步应用程序进行了微小更改，以便可以对服务进行滚动更新。然后，您将图像部署到运行环境中，并执行滚动更新以验证更改是否成功。标题中的更改表明滚动更新已成功执行。

# 10. Kubernetes

## 活动 10.01：在 Kubernetes 上安装全景徒步应用程序

**解决方案**：

可以通过以下步骤创建数据库和全景徒步应用程序：

1.  使用以下`helm`命令安装数据库：

[PRE112]

这将为 PostgreSQL 安装多个 Kubernetes 资源，并显示摘要如下：

![图 10.23：数据库安装](img/B15021_10_23.jpg)

图 10.23：数据库安装

此输出首先列出与 Helm 图表相关的信息，例如名称、部署时间、状态和修订版本，然后是与 PostgreSQL 实例相关的信息以及如何访问它。这是 Helm 图表中广泛接受的方法，在安装图表后提供此类信息。否则，将很难学习如何连接到 Helm 安装的应用程序。

1.  创建一个`statefulset.yaml`文件，其中包含以下内容：

[PRE113]

此文件创建了一个名为`panoramic-trekking-app`的 Statefulset。在`spec`部分定义了两个名为`nginx`和`pta`的容器。此外，还定义了一个名为`static`的卷索赔，并将其挂载到两个容器上。

1.  使用以下命令部署`panoramic-trekking-app` StatefulSet：

[PRE114]

这将为我们的应用程序创建一个 StatefulSet：

[PRE115]

1.  创建一个`service.yaml`文件，内容如下：

[PRE116]

此服务定义具有`LoadBalancer`类型，以访问具有标签`app: panoramic-trekking-app`的 Pod。端口`80`将可用于访问 Pod 的`web`端口。

1.  使用以下命令部署`panoramic-trekking-app`服务：

[PRE117]

这将创建以下 Service 资源：

[PRE118]

1.  使用以下命令获取 Service 的 IP：

[PRE119]

在以下步骤中存储 IP 以访问 Panoramic Trekking App。

1.  在浏览器中打开 Panoramic Trekking App 的管理部分，网址为`http://$SERVICE_IP/admin`：![图 10.24：管理员登录视图](img/B15021_10_24.jpg)

图 10.24：管理员登录视图

1.  使用用户名`admin`和密码`changeme`登录，并添加新的照片和国家：![图 10.25：管理员设置视图](img/B15021_10_25.jpg)

图 10.25：管理员设置视图

1.  在浏览器中打开 Panoramic Trekking App，网址为`http://$SERVICE_IP/photo_viewer`：![图 10.26：应用程序视图](img/B15021_10_26.jpg)

图 10.26：应用程序视图

照片查看器应用程序显示已从数据库中检索到照片和国家。它还表明应用程序已正确设置并且正常运行。

在这个活动中，您已经将 Panoramic Trekking App 部署到了 Kubernetes 集群。您首先使用其 Helm 图表创建了一个数据库，然后为应用程序创建了 Kubernetes 资源。最后，您从浏览器访问了应用程序，并通过添加新的照片进行了测试。在这个活动结束时，您已经了解了如何使用官方 Helm 图表部署数据库，创建一系列 Kubernetes 资源来连接数据库并部署应用程序，并从集群中收集信息以访问应用程序。该活动中的步骤涵盖了在 Kubernetes 集群中部署容器化应用程序的生命周期。

# 11\. Docker 安全

## 活动 11.01：为 Panoramic Trekking App 设置 seccomp 配置文件

**解决方案**：

有多种方法可以创建一个`seccomp`配置文件，阻止用户执行`mkdir`、`kill`和`uname`命令。以下步骤展示了如何完成这一操作：

1.  如果你本地没有`postgres`镜像，请执行以下命令：

[PRE120]

1.  在你的系统上使用`wget`命令获取默认的`seccomp`配置文件的副本。将你下载的文件命名为`activity1.json`：

[PRE121]

1.  从配置文件中删除以下三个命令，以允许我们进一步锁定我们的镜像。用你喜欢的文本编辑器打开`activity1.json`文件，并从文件中删除以下行。你应该删除*行 1500*到*1504*以删除`uname`命令，*669*到*673*以删除`mkdir`命令，以及*行 579*到*583*以删除`kill`命令的可用性：

[PRE122]

你可以在以下链接找到修改后的`activity1.json`文件：[`packt.live/32BI3PK`](https://packt.live/32BI3PK)。

1.  使用`–-security-opt seccomp=activity1.json`选项在运行镜像时为`postgres`镜像分配一个新配置文件：

[PRE123]

1.  现在你已经登录到正在运行的容器中，测试你已经分配给容器的新配置文件的权限。执行`mkdir`命令在系统上创建一个新目录：

[PRE124]

该命令应该显示一个`Operation not permitted`的输出：

[PRE125]

1.  为了测试你不再能够杀死正在运行的进程，你需要启动一些东西。启动`top`进程并在后台运行。在命令行中输入`top`，然后添加`&`，然后按*Enter*在后台运行该进程。接下来的命令提供了进程命令(`ps`)，以查看容器上正在运行的进程：

[PRE126]

如下输出所示，`top`进程正在以`PID 8`运行：

[PRE127]

注意

基于`postgres`镜像的容器中不可用`ps`和`top`命令。然而，这不会造成任何问题，因为用任意随机的 pid 号运行`kill`命令足以证明该命令不被允许运行。

1.  使用`kill -9`命令杀死`top`进程，后面跟着你想要杀死的进程的 PID 号。`kill -9`命令将尝试强制停止命令：

[PRE128]

你应该看到`Operation not permitted`：

[PRE129]

1.  测试`uname`命令。这与其他命令有些不同：

[PRE130]

你将得到一个`Operation not permitted`的输出：

[PRE131]

这是一个很好的活动，表明如果我们的镜像被攻击者访问，我们仍然可以采取很多措施来限制对它们的操作。

## 活动 11.02：扫描您的全景徒步应用镜像的漏洞

**解决方案：**

有许多方法可以扫描我们的镜像以查找漏洞。以下步骤是使用 Anchore 来验证`postgres-app`镜像是否对我们的应用程序安全的一种方法：

1.  给镜像打标签并将其推送到您的 Docker Hub 仓库。在这种情况下，使用我们的仓库名称给`postgres-app`镜像打标签，并将其标记为`activity2`。我们还将其推送到我们的 Docker Hub 仓库：

[PRE132]

1.  您应该仍然拥有您在本章最初使用的`docker-compose.yaml`文件。如果您还没有运行 Anchore，请运行`docker-compose`命令并导出`ANCHORE_CLI_URL`、`ANCHORE_CLI_URL`和`ANCHORE_CLI_URL`变量，就像您之前做的那样，以便我们可以运行`anchore-cli`命令：

[PRE133]

1.  通过运行`anchore-cli system status`命令来检查 Anchore 应用的状态：

[PRE134]

1.  使用`feeds list`命令来检查 feeds 列表是否都已更新：

[PRE135]

1.  一旦所有的 feeds 都已经更新，添加我们推送到 Docker Hub 的`postgres-app`镜像。使用`anchore-cli`提供的`image add`命令，并提供我们想要扫描的镜像的仓库、镜像和标签。这将把镜像添加到我们的 Anchore 数据库中，准备进行扫描：

[PRE136]

1.  使用`image list`命令来验证我们的镜像是否已经被分析。一旦完成，您应该在`Analysis Status`列中看到`analyzed`这个词：

[PRE137]

1.  使用我们的镜像名称执行`image vuln`命令，以查看在我们的`postgres-app`镜像上发现的所有漏洞的列表。这个镜像比我们之前测试过的镜像要大得多，也要复杂得多，所以当我们使用`all`选项时，会发现很长的漏洞列表。幸运的是，大多数漏洞要么是`Negligible`，要么是`Unknown`。运行`image vuln`命令并将结果传输到`wc -l`命令：

[PRE138]

这将给我们一个漏洞数量的统计。在这种情况下有超过 100 个值：

[PRE139]

1.  最后，使用`evaluate check`命令来查看发现的漏洞是否会给我们通过或失败：

[PRE140]

幸运的是，正如您从以下输出中看到的，我们通过了：

[PRE141]

由于该镜像由一个大型组织提供，他们有责任确保您可以安全使用它，但由于扫描镜像如此容易，我们仍然应该扫描它们以验证它们是否 100%安全可用。

# 12\. 最佳实践

## 活动 12.01：查看 Panoramic Trekking App 使用的资源

**解决方案：**

我们在本章中执行第一个活动的方式有很多种。以下步骤是通过使用`docker stats`命令查看 Panoramic Trekking App 中服务使用的资源的一种方法。在本例中，我们将使用作为 Panoramic Trekking App 的一部分运行的`postgresql-app`服务：

1.  创建一个脚本，将创建一个新表并用随机值填充它。以下脚本正是我们在这种情况下想要的，因为我们想创建一个长时间的处理查询，并查看它如何影响我们容器上的资源。添加以下细节，并使用您喜欢的编辑器将文件保存为`resource_test.sql`：

[PRE142]

*第 1 行*到*第 6 行*创建新表并设置它包含的三行，而*第 8 行到第 14 行*遍历一个新表，用随机值填充它。

1.  如果您还没有 PostgreSQL Docker 镜像的副本，请使用以下命令从受支持的 PostgreSQL Docker Hub 存储库中拉取镜像：

[PRE143]

1.  进入新的终端窗口并运行`docker stats`命令，查看正在使用的`CPU`百分比，以及正在使用的内存和内存百分比：

[PRE144]

在以下命令中，我们没有显示容器 ID，因为我们希望限制输出中显示的数据量：

[PRE145]

1.  要简单测试这个镜像，您不需要将运行的容器挂载到特定卷上，以使用您先前为此镜像使用的数据。切换到另一个终端来监视您的 CPU 和内存。启动容器并将其命名为`postgres-test`，确保数据库可以从主机系统访问，通过暴露运行`psql`命令所需的端口。我们还在此示例中使用环境变量（`-e`）选项指定了临时密码为`docker`：

[PRE146]

1.  在运行测试脚本之前，切换到监视 CPU 和内存使用情况的终端。您可以看到我们的容器已经在没有真正做任何事情的情况下使用了一些资源：

[PRE147]

1.  使用以下命令进入容器内的终端：

[PRE148]

1.  使用`psql`命令发送`postgres-test`容器命令以创建名为`resource_test`的新数据库：

[PRE149]

1.  运行您之前创建的脚本。确保在运行脚本之前包括`time`命令，因为这将允许您查看完成所需的时间：

[PRE150]

我们已经减少了以下代码块中命令的输出。用数据填充`resource_database`表花费了 50 秒：

[PRE151]

1.  移动到运行`docker stats`命令的终端。您将看到一个输出，取决于系统运行的核心数量和可用的内存。正在运行的脚本似乎不太耗费内存，但它正在将容器可用的 CPU 推高到 100%：

[PRE152]

1.  在对 CPU 和内存配置进行更改后运行容器之前，删除正在运行的容器，以确保您有一个新的数据库运行，使用以下命令：

[PRE153]

1.  再次运行容器。在这种情况下，您将将可用的 CPU 限制为主机系统上一半的一个核心，并且由于测试不太耗费内存，将内存限制设置为`256MB`：

[PRE154]

1.  使用`exec`命令进入容器：

[PRE155]

1.  再次，在运行测试之前，创建`resource_test`数据库：

[PRE156]

1.  现在，为了查看对我们的资源所做的更改，限制容器可以使用的资源。再次运行`resource_test.sql`脚本，并通过限制资源，特别是 CPU，我们可以看到现在完成需要超过 1 分钟：

[PRE157]

1.  移动到运行`docker stats`命令的终端。它看起来也会有所不同，因为可用于使用的 CPU 百分比将减半。您对 CPU 所做的更改减慢了脚本的运行，并且似乎也减少了内存的使用：

[PRE158]

这项活动为您提供了一个很好的指示，有时您需要执行平衡的行为，当您监视和配置容器资源时。它确实澄清了您需要了解您的服务正在执行的任务，以及对配置的更改将如何影响您的服务的运行方式。

## 活动 12.02：使用 hadolint 改进 Dockerfile 的最佳实践

**解决方案**

有许多种方法可以执行此活动。以下步骤展示了一种方法：

1.  使用以下`docker pull`命令从`hadolint`存储库中拉取镜像：

[PRE159]

1.  使用`hadolint`来检查我们在本章中一直在使用的`docker-stress` `Dockerfile`并记录所呈现的警告：

[PRE160]

你会收到以下警告：

[PRE161]

与你最初测试镜像时没有真正的变化。然而，在`Dockerfile`中只有三行代码，所以看看是否可以减少`hadolint`呈现的警告数量。

1.  正如本章前面提到的，`hadolint`维基页面将为你提供如何解决所呈现的每个警告的详细信息。然而，如果你逐行进行，应该能够解决所有这些警告。首先呈现的`DL3006`要求标记你正在使用的 Docker 镜像版本，这是 Ubuntu 镜像的新版本。将你的`Dockerfile`的*行 1*更改为现在包括`18.08`镜像版本，如下所示：

[PRE162]

1.  接下来的四个警告都与我们`Dockerfile`的第二行有关。`DL3008`要求固定安装的应用程序版本。在下面的情况下，将 stress 应用程序固定到 1.0.3 版本。`DL3009`指出你应该删除任何列表。这就是我们在下面的代码中添加*行 4*和*行 5*的地方。`DL3015`指出你还应该使用`--no-install-recommends`，确保你不安装不需要的应用程序。最后，`DL3014`建议你包括`-y`选项，以确保你不会被提示验证应用程序的安装。编辑`Dockerfile`如下所示：

[PRE163]

1.  `DL3025`是你的最后警告，并且指出你需要将`CMD`指令以 JSON 格式编写。这可能会导致问题，因为你正在尝试在 stress 应用程序中使用环境变量。为了消除这个警告，使用`sh -c`选项运行`stress`命令。这样应该仍然允许你使用环境变量运行命令：

[PRE164]

你的完整`Dockerfile`，现在遵循最佳实践，应该如下所示：

[PRE165]

1.  现在，再次使用`hadolint`对`Dockerfile`进行检查，不再呈现任何警告：

[PRE166]

1.  如果你想百分之百确定`Dockerfile`看起来尽可能好，进行最后一次测试。在浏览器中打开`FROM:latest`，你会看到`Dockerfile`显示最新更改时的`没有找到问题或建议！`：![图 12.4：docker-stress Dockerfile 现在遵循最佳实践](img/B15021_12_04.jpg)

图 12.4：docker-stress Dockerfile 现在遵循最佳实践

您的`Dockerfiles`可能比本章中呈现的要大得多，但是正如您所看到的，逐行系统地处理将帮助您纠正`Dockerfiles`可能存在的任何问题。使用诸如`hadolint`和`FROM latest`之类的应用程序，以及它们关于如何解决警告的建议，将使您熟悉随着实践而来的最佳实践。这就是我们活动和本章的结束，但是还有更多有趣的内容要学习，所以现在不要停下来。

# 13.监控 Docker 指标

## 活动 13.01：创建 Grafana 仪表板以监控系统内存

**解决方案**：

有许多方法可以执行此活动。以下步骤是一种方法：

1.  确保 Prometheus 正在运行和收集数据，Docker 和`cAdvisor`已配置为公开指标，并且 Grafana 正在运行并配置为使用 Prometheus 作为数据源。

1.  打开 Grafana Web 界面和您在*练习 13.05：在您的系统上安装和运行 Grafana*中创建的`Container Monitoring`仪表板

1.  仪表板顶部和仪表板名称右侧有一个“添加面板”的选项。单击“添加面板”图标以添加新的仪表板面板：![图 13.26：向容器监控仪表板添加新面板](img/B15021_13_26.jpg)

图 13.26：向容器监控仪表板添加新面板

1.  从下拉列表中选择`Prometheus`作为我们将使用的数据源，以生成新的仪表板面板。

1.  在`metrics`部分，添加以下 PromQL 查询，`container_memory_usage_bytes`，仅搜索具有名称值的条目。然后，按每个名称求和，为每个容器提供一条线图：

[PRE167]

1.  根据时间序列数据库中可用的数据量进行调整相对时间（如果需要）。也许将相对时间设置为`15m`。前三个步骤如下图所示：![图 13.27：向容器监控仪表板添加新面板](img/B15021_13_27.jpg)

图 13.27：向容器监控仪表板添加新面板

1.  选择“显示选项”并添加“内存容器使用”作为标题。

1.  如果单击“保存”，您会注意到无法保存面板，因为仪表板已在启动时进行了配置。您可以导出 JSON，然后将其添加到您的配置目录中。单击“共享仪表板”按钮并导出 JSON。选择“将 JSON 保存到文件”并将仪表板文件存储在“/tmp 目录”中：![图 13.28：警告，我们无法保存新的仪表板](img/B15021_13_28.jpg)

图 13.28：警告，我们无法保存新的仪表板

1.  停止运行 Grafana 容器，以便您可以添加到环境中的配置文件。使用以下`docker kill`命令执行此操作：

[PRE168]

1.  您已经在`provisioning/dashboards`目录中有一个名为`ContainerMonitoring.json`的文件。从您的`tmp`目录中复制刚创建的 JSON 文件，并替换`provisioning/dashboards`目录中的原始文件：

[PRE169]

1.  再次启动 Grafana 图像，并使用默认管理密码登录应用程序：

[PRE170]

1.  再次登录到 Grafana，并转到您一直在配置的“容器监控”仪表板。您现在应该在我们的仪表板顶部看到新创建的“内存容器使用情况”面板，类似于以下屏幕截图：![图 13.29：显示内存使用情况的新仪表板面板](img/B15021_13_24.jpg)

图 13.29：显示内存使用情况的新仪表板面板

现在应该更容易监视系统上运行的容器的内存和 CPU 使用情况。仪表板提供了比查看`docker stats`命令更简单的界面，特别是当您开始在系统上运行更多容器时。

## 活动 13.02：配置全景徒步应用程序向 Prometheus 公开指标

**解决方案**：

我们可以以多种方式执行此活动。在这里，我们选择向全景徒步应用程序的 PostgreSQL 容器添加导出器：

1.  如果您没有全景徒步应用程序在运行，请确保至少 PostgreSQL 容器正在运行，以便您可以完成此活动。您还不需要运行 Prometheus，因为您需要先对配置文件进行一些更改。运行以下命令以验证 PostgreSQL 数据库是否正在运行：

[PRE171]

要从您的 PostgreSQL 容器中收集更多指标，您可以在 GitHub 上找到用户`albertodonato`已经创建的导出器。使用其他人已经创建的导出器比自己创建要容易得多。文档和详细信息可以在以下网址找到：[`github.com/albertodonato/query-exporter`](https://github.com/albertodonato)。

1.  上述的 GitHub 帐户有一个很好的分解如何设置配置和指标。设置一个基本的配置文件以开始。通过运行以下`docker inspect`命令找到 PostgreSQL 容器正在运行的 IP 地址。这会给出容器正在运行的内部 IP 地址。您还需要替换您正在运行的容器名称为`<container_name>`：

[PRE172]

您的 IP 地址可能与此处的不同：

[PRE173]

1.  对于这个导出器，您需要设置一些额外的配置来输入到导出器中。首先，在您的工作目录中创建一个名为`psql_exporter_config.yml`的配置文件，并用文本编辑器打开该文件。

1.  在下面的配置文件中输入前四行。这是导出器连接到数据库的方式。您需要提供可以访问数据库的密码以及在上一步中获得的 IP 地址，或者如果为数据库分配了域，则需要提供域：

[PRE174]

1.  将您的第一个指标添加到配置文件中。输入以下行以添加指标名称、规模类型、描述和标签：

[PRE175]

1.  设置一个数据库查询，以收集您想要用于`pg_process`规模的指标详细信息。*第 13 行*显示您想要创建一个数据库查询，*第 14*和*15 行*将结果分配给您之前创建的指标。*第 16*至*23 行*是我们想要在数据库上运行的查询，以便为数据库上运行的进程数量创建一个规模：

psql_exporter_config.yml

[PRE176]

您可以在此处找到完整的代码[`packt.live/32C47K3`](https://packt.live/32C47K3)。

1.  保存配置文件并从命令行运行导出器。导出器将在端口`9560`上公开其指标。挂载您在此活动中创建的配置文件。您还将获得`adonato/query-exporter`图像的最新版本：

[PRE177]

1.  打开一个网络浏览器，使用 URL`http://0.0.0.0:9560/metrics`来查看您为全景徒步应用程序运行的 PostgreSQL 容器设置的新指标：

[PRE178]

1.  进入您安装了 Prometheus 的目录，用您喜欢的文本编辑器打开`prometheus.yml`文件，并添加导出器详细信息，以允许 Prometheus 开始收集数据：

[PRE179]

1.  保存您对`prometheus.yml`文件所做的更改，并再次从命令行启动 Prometheus 应用程序，如下所示：

[PRE180]

1.  如果一切都按照应该的方式进行了，您现在应该在 Prometheus 的`Targets`页面上看到`postgres-web`目标，如图所示：![图 13.30：在 Prometheus 上显示的新的 postgres-web 目标页面](img/B15021_13_25.jpg)

图 13.30：在 Prometheus 上显示的新的 postgres-web 目标页面

这就是活动的结束，也是本章的结束。这些活动应该有助于巩固之前学到的知识，并为您提供了在更加用户友好的方式中收集应用程序和运行系统的指标的经验。

# 14. 收集容器日志

## 活动 14.01：为您的 Splunk 安装创建一个 docker-compose.yml 文件

**解决方案**：

有许多方法可以执行这项活动。以下步骤概述了一种可能的方法。

在这里，您将设置一个`docker-compose.yml`文件，该文件至少会以与本章中一直运行的方式运行您的 Splunk 容器。您将设置两个卷，以便挂载`/opt/splunk/etc`目录，以及`/opt/splunk/var`目录。您需要公开端口`8000`、`9997`和`8088`，以允许访问您的 Web 界面并允许数据转发到 Splunk 实例。最后，您需要设置一些环境变量，以接受 Splunk 许可证并添加管理员密码。让我们开始吧：

1.  创建一个名为`docker-compose.yml`的新文件，并用您喜欢的文本编辑器打开它。

1.  从您喜欢的`Docker Compose`版本开始，并创建要用于挂载`var`和`ext`目录的卷：

[PRE181]

1.  使用`splunk`作为主机名和`splunk/splunk`作为您安装的镜像来设置 Splunk 安装的服务。此外，设置`SPLUNK_START_ARGS`和`SPLUNK_PASSWORD`的环境变量，如下所示：

[PRE182]

1.  最后，挂载卷并公开安装所需的访问 Web 界面和从转发器和容器转发数据的端口：

[PRE183]

1.  运行`docker-compose up`命令，确保一切都正常工作。使用`-d`选项确保它作为后台守护程序在我们系统中运行：

[PRE184]

该命令应返回类似以下的输出：

[PRE185]

1.  一旦您的 Splunk 安装再次运行，就该是让 Panoramic Trekking App 中的一个服务运行起来，这样您就可以将日志转发到 Splunk 进行索引。在使用`docker run`命令时，添加日志驱动程序的详细信息，就像您在本章中之前所做的那样，并确保您包括了正确的`HTTP 事件收集器`的令牌：

[PRE186]

请注意

请注意，我们在`docker run`命令中使用了`-c log_statement=all`，这将确保我们所有的 PostgreSQL 查询都被记录并发送到 Splunk。

1.  登录 Splunk Web 界面并访问`搜索和报告`应用程序。在界面中输入`source="http:docker logs" AND postgres-test`查询，然后按*Enter*。由于您已经给我们的容器打了标签，您应该会看到您的容器带有名称和完整 ID 的标签，因此在搜索中添加`postgres-test`将确保只有您的 PostgreSQL 日志可见：![图 14.48：Splunk 中显示的 PostgreSQL 日志](img/B15021_14_46.jpg)

图 14.48：Splunk 中显示的 PostgreSQL 日志

从前面的屏幕截图中可以看出，我们的日志已经成功地通过 Splunk 流动。请注意在日志条目中添加的标签，就像前面的屏幕截图中所示的那样。

这项活动教会了我们如何在开发项目中使用 Docker Compose 实施日志记录程序。

## 活动 14.02：创建一个 Splunk 应用程序来监视 Panoramic Trekking App

**解决方案**：

有许多方法可以执行这项活动。以下步骤是一种方法。在这里，您将向作为 Panoramic Trekking App 的一部分正在运行的`PostgreSQL`容器添加一个导出器：

1.  确保 Splunk 正在运行，并且您一直在监视的服务已经运行了一段时间，以确保您正在为这项活动收集一些日志。

1.  登录 Splunk Web 界面。从 Splunk 主屏幕上，点击`应用程序`菜单旁边的齿轮图标；您将看到您的 Splunk 环境的`应用程序`页面：![图 14.49：Splunk 环境的应用程序页面](img/B15021_14_49.jpg)

图 14.49：Splunk 环境的应用程序页面

1.  点击“创建”应用按钮并填写表格。表格将类似于以下内容，其中“名称”设置为“全景徒步应用”，“文件夹名称”设置为`panoramic_trekking_app`，“版本”设置为`1.0.0`。点击“保存”以创建新应用：![图 14.50：在 Splunk 中创建您的新应用](img/B15021_14_50.jpg)

图 14.50：在 Splunk 中创建您的新应用

1.  返回到 Splunk 主页，并确保您的“全景徒步应用”从“应用”菜单中可见。点击“全景徒步应用”以打开“搜索和报告”页面，以便您可以开始查询您的数据：![图 14.51：选择全景徒步应用](img/B15021_14_51.jpg)

图 14.51：选择全景徒步应用

1.  在查询栏中输入`source="http:docker logs" AND postgres-test AND INSERT AND is_superuser | stats count`，然后按 Enter 键。搜索将查找作为应用程序的一部分创建的任何“超级用户”。当您的数据出现时，点击“可视化”选项卡，并将其更改为显示单个值的可视化：![图 14.52：在查询栏中输入查询](img/B15021_14_52.jpg)

图 14.52：在查询栏中输入查询

1.  点击屏幕顶部的“另存为”按钮，然后选择“仪表板”面板。当您看到此屏幕时，选择要添加到新仪表板的面板，并将其命名为“PTA 监控”。还要给面板命名为“超级用户访问”，然后点击“保存”：![图 14.53：向仪表板面板添加详细信息](img/B15021_14_53.jpg)

图 14.53：向仪表板面板添加详细信息

1.  当您看到新的仪表板时，点击“编辑”和“添加”面板按钮。选择“新建”，然后选择“单个值”作为可视化类型。将“内容标题”设置为“数据库创建”。添加`source="http:docker logs" AND postgres-test AND CREATE DATABASE | stats count`源字符串，然后点击“保存”。这将通过日志搜索以显示是否有人在 PostgreSQL 数据库上创建了任何数据库，这应该只在设置和创建应用程序时发生：![图 14.54：编辑仪表板面板](img/B15021_14_54.jpg)

图 14.54：编辑仪表板面板

1.  再次点击“新面板”按钮，选择“新建”，然后从可视化中选择“柱状图”。添加一个“内容标题”为“应用使用情况”，添加`source="http:docker logs" AND postgres-test AND SELECT AND photo_viewer_photo earliest=-60m | timechart span=1m count`搜索查询，并点击“保存”。这个搜索将为您提供一段时间内使用应用程序查看照片的人数。

1.  随意移动仪表板上的面板。当您对更改感到满意时，点击“保存”按钮。您的仪表板应该看起来类似于以下内容：![图 14.55：用于监视 PostgreSQL 使用情况的新仪表板面板](img/B15021_14_55.jpg)

图 14.55：用于监视 PostgreSQL 使用情况的新仪表板面板

这个活动帮助您收集 Panoramic Trekking App 的日志数据，并使用 Splunk 以更用户友好的方式显示它。

# 15. 使用插件扩展 Docker

## 活动 15.01：使用网络和卷插件安装 WordPress

**解决方案：**

可以使用以下步骤使用卷和网络插件为数据库和 WordPress 博客创建容器：

1.  使用以下命令创建一个网络：

[PRE187]

这个命令使用 Weave Net 插件创建一个网络，并使用`driver`标志进行指定。此外，卷被指定为`attachable`，这意味着您可以在将来连接到 Docker 容器。最后，容器的名称将是`wp-network`。您应该得到以下输出：

[PRE188]

1.  使用以下命令创建一个卷：

[PRE189]

这个命令使用`vieux/sshfs`插件通过 SSH 创建一个卷。卷的名称是`wp-content`，并传递了`ssh`命令、端口和密码的额外选项：

[PRE190]

1.  使用以下命令创建`mysql`容器：

[PRE191]

这个命令以分离模式运行`mysql`容器，并使用环境变量和`wp-network`连接。

1.  使用以下命令创建`wordpress`容器：

[PRE192]

这个命令以分离模式运行`wordpress`容器，并使用环境变量和`wp-network`连接。此外，容器的端口`80`在主机系统的端口`8080`上可用。

成功启动后，您将有两个运行中的`mysql`和`wordpress`容器：

[PRE193]

![图 15.17：WordPress 和数据库容器](img/B15021_15_15.jpg)

图 15.17：WordPress 和数据库容器

1.  在浏览器中打开`http://localhost:8080`，以检查 WordPress 设置屏幕：![图 15.18：WordPress 设置屏幕](img/B15021_15_18.jpg)

图 15.18：WordPress 设置屏幕

WordPress 设置屏幕验证了 WordPress 是否使用了网络和卷插件。

在这个活动中，您使用 Weave Net 插件创建了一个自定义网络，并使用`sshfs`插件创建了一个自定义卷。您创建了一个使用自定义网络的数据库容器，以及一个使用自定义网络和自定义卷的 WordPress 容器。通过成功的设置，您的 Docker 容器可以通过自定义网络相互连接，并通过 SSH 使用卷。通过这个活动，您已经为一个真实的应用程序使用了 Docker 扩展。现在您可以自信地根据自己的业务需求和技术扩展 Docker。

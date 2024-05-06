# 评估

# 第一章，Docker 概述

1.  Docker 商店：[`store.docker.com/`](https://store.docker.com/)

1.  `$ docker image pull nginx`

1.  Moby 项目

1.  七个月

1.  `$ docker container help`

# 第二章，构建容器映像

1.  错误；它用于向图像添加元数据

1.  您可以附加`CMD`到`ENTRYPOINT`，但不能覆盖周围

1.  正确

1.  快照失败的容器，以便您可以在离开 Docker 主机时进行审查

1.  `EXPOSE`指令用于在容器上公开端口，但不会映射主机上的端口

# 第三章，存储和分发图像

1.  错误；还有 Docker 商店

1.  这允许您在上游 Docker 图像更新时自动更新您的 Docker 图像

1.  是的，它们是（如在本章的示例中所示）

1.  正确；如果您使用命令行登录，您已登录到 Docker for Mac 和 Docker for Windows

1.  您将按名称删除它们，而不是使用图像 ID

1.  端口`5000`

# 第四章，管理容器

1.  `-a`或`--all`

1.  错误；情况正好相反

1.  当您按下*Ctrl + C*时，您将返回到终端；但是，保持容器活动的进程仍在运行，因为我们已经从进程中分离出来，而不是终止它

1.  错误；它在指定的容器内生成一个新进程

1.  您将使用`--network-alias [别名]`标志

1.  运行`docker volume inspect [卷名称]`将为您提供有关卷的信息

# 第五章，Docker Compose

1.  YAML，或 YAML 不是标记语言

1.  `restart`标志与`--restart`标志相同

1.  错误；您可以使用 Docker Compose 在运行时构建图像

1.  默认情况下，Docker Compose 使用存储 Docker Compose 文件的文件夹的名称

1.  您可以使用`-d`标志启动容器的分离模式

1.  使用`docker-compose config`命令将公开 Docker Compose 文件中的任何语法错误

1.  Docker App 将您的 Docker Compose 文件捆绑到一个小的 Docker 图像中，可以通过 Docker Hub 或其他注册表共享，Docker App 命令行工具可以从图像中包含的数据渲染工作的 Docker Compose 文件

# 第六章，Windows 容器

1.  您可以使用 Hyper-V 隔离在最小的 hypervisor 中运行容器

1.  命令是`docker inspect -f "{{ .NetworkSettings.Networks.nat.IPAddress }}” [容器名称]`

1.  错误；在管理 Windows 容器时，您需要运行的 Docker 命令没有区别

# 第七章，Docker Machine

1.  使用`--driver`标志

1.  错误；它会给您命令；相反，您需要运行`eval $(docker-machine env my-host)`

1.  Docker Machine 是一个命令行工具，可用于以简单和一致的方式在许多平台和技术上启动 Docker 主机

# 第八章，Docker Swarm

1.  错误；独立的 Docker Swarm 不再受支持或被视为最佳实践

1.  您需要 Docker Swarm 管理器的 IP 地址，以及用于对抗您的管理器进行身份验证的令牌

1.  您将使用`docker node ls`

1.  您将添加`--pretty`标志

1.  您将使用`docker node promote [node name]`

1.  您将运行`docker service scale cluster=[x] [service name]`，其中`[x]`是您要扩展的容器数量

# 第九章，Docker 和 Kubernetes

1.  错误；您始终可以看到 Kubernetes 使用的图像

1.  `docker`和`kube-system`命名空间

1.  您将使用`kubectl describe --namespace [NAMESPACE] [POD NAME]`

1.  您将运行`kubectl create -f [FILENAME OR URL]`

1.  端口`8001`

1.  它被称为 Borg

# 第十章，在公共云中运行 Docker

1.  错误；他们启动 Docker Swarm 集群

1.  在使用 Amazon Fargate 时，您无需启动 Amazon EC2 实例来运行您的 Amazon ECS 集群

1.  容器选项列在 Azure Web 应用程序服务下

1.  使用命令`kubectl create namespace sock-shop`

1.  通过运行`kubectl -n sock-shop describe services front-end-lb`

# 第十一章，Portainer - Docker 的 GUI

1.  路径为`/var/run/docker.sock`

1.  端口是`9000`

1.  错误；应用程序有自己的定义。在运行 Docker Swarm 时，您可以使用 Docker Compose 并启动一个堆栈

1.  正确；所有统计数据都是实时显示的

# 第十二章，Docker 安全

1.  您将添加`--read-only`标志；或者，如果您想使卷为只读，您将添加`:ro`

1.  在理想的世界中，您只会在每个容器中运行一个进程

1.  通过运行 Docker Bench Security 应用程序

1.  Docker 的套接字文件，可以在`/var/run/docker.sock`找到；而且，如果您的主机系统正在运行 Systemd，则在`/usr/lib/systemd`

1.  错误；Quay 扫描公共和私有图像

# 第十三章，Docker 工作流

1.  nginx（`web`）容器提供网站；WordPress（`WordPress`）容器运行传递给 nginx 容器的代码

1.  `wp`容器运行一个进程，一旦运行就存在

1.  `cAdvisor`仅保留五分钟的度量标准

1.  您将使用`docker-compose down --volumes --rmi all`

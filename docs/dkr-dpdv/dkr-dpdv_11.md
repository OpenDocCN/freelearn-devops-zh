## 第十一章：9：使用 Docker Compose 部署应用程序

在本章中，我们将看看如何使用 Docker Compose 部署多容器应用程序。

Docker Compose 和 Docker Stacks 非常相似。在本章中，我们将专注于 Docker Compose，它在**单引擎模式**下部署和管理多容器应用程序在 Docker 节点上运行。在后面的章节中，我们将专注于 Docker Stacks。Stacks 在**Swarm 模式**下部署和管理多容器应用程序。

我们将把这一章分成通常的三个部分：

+   简而言之

+   深入研究

+   命令

### 使用 Compose 部署应用程序-简而言之

大多数现代应用程序由多个较小的服务组成，这些服务相互交互形成一个有用的应用程序。我们称之为微服务。一个简单的例子可能是一个具有以下四个服务的应用程序：

+   Web 前端

+   排序

+   目录

+   后端数据库

把所有这些放在一起，你就有了一个*有用的应用程序*。

部署和管理大量服务可能很困难。这就是*Docker Compose*发挥作用的地方。

与使用脚本和冗长的`docker`命令将所有内容粘合在一起不同，Docker Compose 允许你在一个声明性的配置文件中描述整个应用程序。然后你可以用一个命令部署它。

一旦应用程序被*部署*，你可以用一组简单的命令*管理*它的整个生命周期。你甚至可以将配置文件存储和管理在版本控制系统中！这一切都非常成熟 :-D

这就是基础知识。让我们深入了解一下。

### 使用 Compose 部署应用程序-深入研究

我们将把深入研究部分分为以下几个部分：

+   Compose 背景

+   安装 Compose

+   Compose 文件

+   使用 Compose 部署应用程序

+   使用 Compose 管理应用程序

#### Compose 背景

一开始是*Fig*。Fig 是一个强大的工具，由一个叫做*Orchard*的公司创建，它是管理多容器 Docker 应用程序的最佳方式。它是一个 Python 工具，位于 Docker 之上，允许你在一个单独的 YAML 文件中定义整个多容器应用程序。然后你可以用`fig`命令行工具部署应用程序。Fig 甚至可以管理整个应用程序的生命周期。

在幕后，Fig 会读取 YAML 文件，并通过 Docker API 部署和管理应用程序。这是一件好事！

事实上，2014 年，Docker 公司收购了 Orchard 并将 Fig 重新命名为*Docker Compose*。命令行工具从`fig`改名为`docker-compose`，自收购以来，它一直是一个外部工具，可以附加到 Docker Engine 上。尽管它从未完全集成到 Docker Engine 中，但它一直非常受欢迎并被广泛使用。

就目前而言，Compose 仍然是一个外部的 Python 二进制文件，您必须在运行 Docker Engine 的主机上安装它。您可以在 YAML 文件中定义多容器（多服务）应用程序，将 YAML 文件传递给`docker-compose`二进制文件，然后 Compose 通过 Docker Engine API 部署它。

是时候看它的表现了。

#### 安装 Compose

Docker Compose 在多个平台上都可用。在本节中，我们将演示在 Windows、Mac 和 Linux 上安装它的*一些*方法。还有更多的安装方法，但我们在这里展示的方法可以让您开始。

##### 在 Windows 10 上安装 Compose

在 Windows 10 上运行 Docker 的推荐方式是*Docker for Windows (DfW)*。Docker Compose 包含在标准的 DfW 安装中。因此，如果您安装了 DfW，您就有了 Docker Compose。

使用以下命令检查是否已安装 Compose。您可以从 PowerShell 或 CMD 终端运行此命令。

```
> docker-compose --version
docker-compose version 1.18.0, build 8dd22a96 
```

如果您需要有关在 Windows 10 上安装*Docker for Windows*的更多信息，请参阅**第三章：安装 Docker**。

##### 在 Mac 上安装 Compose

与 Windows 10 一样，Docker Compose 作为*Docker for Mac (DfM)*的一部分安装。因此，如果您安装了 DfM，您就有了 Docker Compose。

从终端窗口运行以下命令以验证您是否安装了 Docker Compose。

```
$ docker-compose --version
docker-compose version `1`.18.0, build 8dd22a96 
```

如果您需要有关在 Mac 上安装*Docker for Mac*的更多信息，请参阅**第三章：安装 Docker**。

##### 在 Windows Server 上安装 Compose

Docker Compose 作为一个独立的二进制文件安装在 Windows Server 上。要使用它，您需要在 Windows Server 上安装最新的 Docker。

在 PowerShell 终端中键入以下命令以安装 Docker Compose。

为了可读性，该命令使用反引号（`）来转义回车并将命令包装在多行上。

以下命令安装了 Docker Compose 的`1.18.0`版本。您可以安装此处列出的任何版本：https://github.com/docker/compose/releases。只需用您想要安装的版本替换 URL 中的`1.18.0`。

```
> Invoke-WebRequest ` "https://github.com/docker/compose/releases/download/1\
.18.0/docker-compose-Windows-x86_64.exe" `
-UseBasicParsing `
-OutFile $Env:ProgramFiles\docker\docker-compose.exe

Writing web request
Writing request stream... (Number of bytes written: 5260755) 
```

使用`docker-compose --version`命令验证安装。

```
> docker-compose --version
docker-compose version 1.18.0, build 8dd22a96 
```

Compose 现在已安装。只要您的 Windows Server 机器安装了最新版本的 Docker Engine，您就可以开始了。

##### 在 Linux 上安装 Compose

在 Linux 上安装 Docker Compose 是一个两步过程。首先，您使用`curl`命令下载二进制文件。然后使用`chmod`使其可执行。

要使 Docker Compose 在 Linux 上工作，您需要一个可用的 Docker Engine 版本。

以下命令将下载 Docker Compose 的版本`1.18.0`并将其复制到`/usr/bin/local`。您可以在[GitHub](https://github.com/docker/compose/releases)的发布页面上检查最新版本，并将 URL 中的`1.18.0`替换为您想要安装的版本。

该命令可能会在书中跨越多行。如果您在一行上运行该命令，您需要删除任何反斜杠（`\`）。

```
`$` `curl` `-``L` `\`
 `https``:``//``github``.``com``/``docker``/``compose``/``releases``/``download``/``1.18.0``/``docker``-``compose-``````

`uname` `-``s`````-`````uname` `-``m```` `\`
 `-``o` `/``usr``/``local``/``bin``/``docker``-``compose`

`% Total    % Received   Time        Time     Time    Current`
                        `Total`       `Spent`    `Left`    `Speed`
`100`   `617`    `0`   `617`    `0` `--:--:--` `--:--:--` `--:--:--`  `1047`
`100` `8280``k`  `100` `8280``k`    `0`  `0``:``00``:``03`  `0``:``00``:``03` `--:--:--`  `4069``k` 
```

 `Now that you’ve downloaded the `docker-compose` binary, use the following `chmod` command to make it executable.

```
$ chmod +x /usr/local/bin/docker-compose 
```

 `Verify the installation and check the version.

```
$ docker-compose --version
docker-compose version `1`.18.0, build 8dd22a9 
```

 `You’re ready to use Docker Compose on Linux.

You can also use `pip` to install Compose from its Python package. But we don’t want to waste pages showing every possible installation method. Enough is enough, time to move on!

#### Compose files

Compose uses YAML files to define multi-service applications. YAML is a subset of JSON, so you can also use JSON. However, all of the examples in this chapter will be YAML.

The default name for the Compose YAML file is `docker-compose.yml`. However, you can use the `-f` flag to specify custom filenames.

The following example shows a very simple Compose file that defines a small Flask app with two services (`web-fe` and `redis`). The app is a simple web server that counts the number of visits and stores the value in Redis. We’ll call the app `counter-app` and use it as the example application for the rest of the chapter.

```
version: "3.5"
services:
  web-fe:
    build: .
    command: python app.py
    ports:
      - target: 5000
        published: 5000
    networks:
      - counter-net
    volumes:
      - type: volume
        source: counter-vol
        target: /code
  redis:
    image: "redis:alpine"
    networks:
      counter-net:

networks:
  counter-net:

volumes:
  counter-vol: 
```

 `We’ll skip through the basics of the file before taking a closer look.

The first thing to note is that the file has 4 top-level keys:

*   `version`
*   `services`
*   `networks`
*   `volumes`

Other top-level keys exist, such as `secrets` and `configs`, but we’re not looking at those right now.

The `version` key is mandatory, and it’s always the first line at the root of the file. This defines the version of the Compose file format (basically the API). You should normally use the latest version.

It’s important to note that the `versions` key does not define the version of Docker Compose or the Docker Engine. For information regarding compatibility between versions of the Docker Engine, Docker Compose, and the Compose file format, google “Compose file versions and upgrading”.

For the remainder of this chapter we’ll be using version 3 or higher of the Compose file format.

The top-level `services` key is where we define the different application services. The example we’re using defines two services; a web front-end called `web-fe`, and an in-memory database called `redis`. Compose will deploy each of these services as its own container.

The top-level `networks` key tells Docker to create new networks. By default, Compose will create `bridge` networks. These are single-host networks that can only connect containers on the same host. However, you can use the `driver` property to specify different network types.

The following code can be used in your Compose file to create a new *overlay* network called `over-net` that allows standalone containers to connect to it (`attachable`).

```
`networks``:`
  `over``-``net``:`
  `driver``:` `overlay`
  `attachable``:` `true` 
```

 `The top-level `volumes` key is where we tell Docker to create new volumes.

##### Our specific Compose file

The example file we’ve listed uses the Compose v3.5 file format, defines two services, defines a network called counter-net, and defines a volume called counter-vol.

Most of the detail is in the `services` section, so let’s take a closer look at that.

The services section of our Compose file has two second-level keys:

*   web-fe
*   redis

Each of these defines a service in the app. It’s important to understand that Compose will deploy each of these as a container, and it will use the name of the keys as part of the container names. In our example, we’ve defined two keys; `web-fe` and `redis`. This means Compose will deploy two containers, one will have `web-fe` in its name and the other will have `redis`.

Within the definition of the `web-fe` service, we give Docker the following instructions:

*   `build: .` This tells Docker to build a new image using the instructions in the `Dockerfile` in the current directory (`.`). The newly built image will be used to create the container for this service.
*   `command: python app.py` This tells Docker to run a Python app called `app.py` as the main app in the container. The `app.py` file must exist in the image, and the image must contain Python. The Dockerfile takes care of both of these requirements.
*   `ports:` Tells Docker to map port 5000 inside the container (`-target`) to port 5000 on the host (`published`). This means that traffic sent to the Docker host on port 5000 will be directed to port 5000 on the container. The app inside the container listens on port 5000.
*   `networks:` Tells Docker which network to attach the service’s container to. The network should already exist, or be defined in the `networks` top-level key. If it’s an overlay network, it will need to have the `attachable` flag so that standalone containers can be attached to it (Compose deploys standalone containers instead of Docker Services).
*   `volumes:` Tells Docker to mount the counter-vol volume (`source:`) to `/code` (‘target:’) inside the container. The `counter-vol` volume needs to already exist, or be defined in the `volumes` top-level key at the bottom of the file.

In summary, Compose will instruct Docker to deploy a single standalone container for the `web-fe` service. It will be based on an image built from a Dockerfile in the same directory as the Compose file. This image will be started as a container and run `app.py` as its main app. It will expose itself on port 5000 on the host, attach to the `counter-net` network, and mount a volume to `/code`.

> **Note:** Technically speaking, we don’t need the `command: python app.py` option. This is because the application’s Dockerfile already defines `python app.py` as the default app for the image. However, we’re showing it here so you know how it works. You can also use it to override CMD instructions set in Dockerfiles.

The definition of the `redis` service is simpler:

*   `image: redis:alpine` This tells Docker to start a standalone container called `redis` based on the `redis:alpine` image. This image will be pulled from Docker Hub.
*   `networks:` The `redis` container will be attached to the `counter-net` network.

As both services will be deployed onto the same `counter-net` network, they will be able to resolve each other by name. This is important as the application is configured to communicate with the redis service by name.

Now that we understand how the Compose file works, let’s deploy it!

#### Deploying an app with Compose

In this section, we’ll deploy the app defined in the Compose file from the previous section. To do this, you’ll need the following 4 files from https://github.com/nigelpoulton/counter-app:

*   Dockerfile
*   app.py
*   requirements.txt
*   docker-compose.yml

Clone the Git repo locally.

```
$ git clone https://github.com/nigelpoulton/counter-app.git

Cloning into `'counter-app'`...
remote: Counting objects: `9`, `done`.
remote: Compressing objects: `100`% `(``8`/8`)`, `done`.
remote: Total `9` `(`delta `1``)`, reused `5` `(`delta `0``)`, pack-reused `0`
Unpacking objects: `100`% `(``9`/9`)`, `done`.
Checking connectivity... `done`. 
```

 `Cloning the repo will create a new sub-directory called `counter-app`. This will contain all of the required files and will be considered your *build context*. Compose will also use the name of the directory (`counter-app`) as your project name. We’ll see this later, but Compose will pre-pend all resource names with `counter-app_`.

Change into the `counter-app` directory and check the files are present.

```
$ `cd` counter-app
$ ls
app.py  docker-compose.yml  Dockerfile  requirements.txt ... 
```

 `Let’s quickly describe each file:

*   `app.py` is the application code (a Python Flask app)
*   `docker-compose.yml` is the Docker Compose file that describes how Docker should deploy the app
*   `Dockerfile` describes how to build the image for the `web-fe` service
*   `requirements.txt` lists the Python packages required for the app

Feel free to inspect the contents of each file.

The `app.py` file is obviously the core of the application. But `docker-compose.yml` is the glue that sticks all the app components together.

Let’s use Compose to bring the app up. You must run the all of the following commands from within the `counter-app` directory that you just cloned from GitHub.

```
$ docker-compose up `&`

`[``1``]` `1635`
Creating network `"counterapp_counter-net"` with the default driver
Creating volume `"counterapp_counter-vol"` with default driver
Pulling redis `(`redis:alpine`)`...
alpine: Pulling from library/redis
1160f4abea84: Pull `complete`
a8c53d69ca3a: Pull `complete`
<Snip>
web-fe_1  `|`  * Debugger PIN: `313`-791-729 
```

 `It’ll take a few seconds for the app to come up, and the output can be quite verbose.

We’ll step through what happened in a second, but first let’s talk about the `docker-compose` command.

`docker-compose up` is the most common way to bring up a Compose app (we’re calling a multi-container app defined in a Compose file a *Compose app*). It builds all required images, creates all required networks and volumes, and starts all required containers.

By default, `docker-compose up` expects the name of the Compose file to `docker-compose.yml` or `docker-compose.yaml`. If your Compose file has a different name, you need to specify it with the `-f` flag. The following example will deploy an application from a Compose file called `prod-equus-bass.yml`

```
$ docker-compose -f prod-equus-bass.yml up 
```

 `It’s also common to use the `-d` flag to bring the app up in the background. For example:

```
docker-compose up -d

--OR--

docker-compose -f prod-equus-bass.yml up -d 
```

 `Our example brought the app up in the foreground (we didn’t use the `-d` flag), but we used the `&` to give us the terminal window back. This is not normal, but it will output logs directly in our terminal window which we’ll use later.

Now that the app is built and running, we can use normal `docker` commands to view the images, containers, networks, and volumes that Compose created.

```
$ docker image ls
REPOSITORY          TAG         IMAGE ID    CREATED         SIZE
counterapp_web-fe   latest      `96`..6ff9e   `3` minutes ago   `95`.9MB
python              `3`.4-alpine  `01`..17a02   `2` weeks ago     `85`.5MB
redis               alpine      ed..c83de   `5` weeks ago     `26`.9MB 
```

 `We can see that three images were either built or pulled as part of the deployment.

The `counterapp_web-fe:latest` image was created by the `build: .` instruction in the `docker-compose.yml` file. This instruction caused Docker to build a new image using the Dockerfile in the same directory. It contains the application code for the Python Flask web app, and was built from the `python:3.4-alpine` image. See the contents of the `Dockerfile` for more information.

```
FROM python:3.4-alpine           << Base image
ADD . /code                      << Copy app into image
WORKDIR /code                    << Set working directory
RUN pip install -r requirements.txt  << install requirements
CMD ["python", "app.py"]         << Set the default app 
```

 `I’ve added comments to the end of each line to help explain. They must be removed before deploying the app.

Notice how Compose has named the newly built image as a combination of the project name (counter-app), and the resource name as specified in the Compose file (web-fe). Compose has removed the dash (`-`) from the project name. All resources deployed by Compose will follow this naming convention.

The `redis:alpine` image was pulled from Docker Hub by the `image: "redis:alpine"` instruction in the `.Services.redis` section of the Compose file.

The following container listing shows two containers. The name of each is prefixed with the name of the project (name of the working directory). Also, each one has a numeric suffix that indicates the instance number — this is because Compose allows for scaling.

```
$ docker container ls
ID    COMMAND           STATUS    PORTS                   NAMES
`12`..  `"python app.py"`   Up `2` min  `0`.0.0.0:5000->5000/tcp  counterapp_web-fe_1
`57`..  `"docker-entry.."`  Up `2` min  `6379`/tcp                counterapp_redis_1 
```

 `The `counterapp_web-fe` container is running the application’s web front end. This is running the `app.py` code and is mapped to port `5000` on all interfaces on the Docker host. We’ll connect to this in just a second.

The following network and volume listings show the `counterapp_counter-net` and `counterapp_counter-vol` networks and volumes.

```
$ docker network ls
NETWORK ID     NAME                     DRIVER    SCOPE
1bd949995471   bridge                   bridge    `local`
40df784e00fe   counterapp_counter-net   bridge    `local`
f2199f3cf275   host                     host      `local`
67c31a035a3c   none                     null      `local`

$ docker volume ls
DRIVER     VOLUME NAME
<Snip>
`local`      counterapp_counter-vol 
```

 `With the application successfully deployed, you can point a web browser at your Docker host on port `5000` and see the application in all its glory.

![](img/figure9-1.png)

Pretty impressive ;-)

Hitting your browser’s refresh button will cause the counter to increment. Feel free to inspect the app (`app.py`) to see how the counter data is stored in the Redis back-end.

If you brought the application up using the `&`, you will be able to see the `HTTP 200` response codes being logged in the terminal window. These indicate successful requests, and you’ll see one for each time you load the web page.

```
web-fe_1  | 172.18.0.1 - - [09/Jan/2018 11:13:21] "GET / HTTP/1.1" 200 -
web-fe_1  | 172.18.0.1 - - [09/Jan/2018 11:13:33] "GET / HTTP/1.1" 200 - 
```

 `Congratulations. You’ve successfully deployed a multi-container application using Docker Compose!

#### Managing an app with Compose

In this section, we’ll see how to start, stop, delete, and get the status of applications being managed by Docker Compose. We’ll also see how the volume we’re using can be used to directly inject updates to the app’s web front-end.

As the application is already up, let’s see how to bring it down. To do this, replace the `up` sub-command with `down`.

```
$ docker-compose down
 `1`. Stopping counterapp_redis_1  ...
 `2`. Stopping counterapp_web-fe_1 ...
 `3`. redis_1   `|` `1`:signal-handler Received SIGTERM scheduling shutdown...
 `4`. redis_1   `|` `1`:M `09` Jan `11`:16:00.456 `# User requested shutdown...`
 `5`. redis_1   `|` `1`:M `09` Jan `11`:16:00.456 * Saving the final RDB snap...
 `6`. redis_1   `|` `1`:M `09` Jan `11`:16:00.463 * DB saved on disk
 `7`. Stopping counterapp_redis_1  ... `done`
 `8`. counterapp_redis_1 exited with code `0`
 `9`. Stopping counterapp_web-fe_1 ... `done`
`10`. Removing counterapp_redis_1  ... `done`
`11`. Removing counterapp_web-fe_1 ... `done`
`12`. Removing network counterapp_counter-net
`13`. `[``1``]`+  Done          docker-compose up 
```

 `Because we started the app with the `&`, it’s running in the foreground. This means we get a verbose output to the terminal, giving us an excellent insight into how things work. Let’s step through what each line is telling us.

Lines 1 and 2 are stopping the two services. These are the `web-fe` and `redis` services defined in the Compose file.

Line 3 shows that the `stop` instruction sends a `SIGTERM` signal. This is sent to the PID 1 process in each container. Lines 4-6 show the Redis container gracefully handling the signal and shutting itself down. Lines 7 and 8 report the success of stop operation.

Line 9 shows the `web-fe` service successfully stopping.

Lines 10 and 11 show the stopped services being removed.

Line 12 shows the `counter-net` network being removed, and line 13 shows the `docker-compose up` process exiting.

It’s important to note that the `counter-vol` volume was **not** deleted. This is because volumes are intended to be long-term persistent data stores. As such, their lifecycle is entirely decoupled from the containers they serve. Running a `docker volume ls` will show that the volume is still present on the system. If you’d written any data to the volume it would still exist.

Also, any images that were built or pulled as part of the `docker-compose up` operation will still remain on the system. This means future deployments of the app will be faster.

Let’s look at a few other `docker-compose` sub-commands.

Use the following command to bring the app up again, but this time in the background.

```
$ docker-compose up -d
Creating network `"counterapp_counter-net"` with the default driver
Creating counterapp_redis_1  ... `done`
Creating counterapp_web-fe_1 ... `done` 
```

 `See how the app started much faster this time — the counter-vol volume already exists, and no images needed building or pulling.

Show the current state of the app with the `docker-compose ps` command.

```
$ docker-compose ps
Name                  Command               State   Ports
--------------------------------------------------------------------------
counterapp_redis_1    docker-entrypoint...  Up      `6379`/tcp
counterapp_web-fe_1   python app.py         Up      `0`.0.0.0:5000->5000/tcp 
```

 `We can see both containers, the commands they are running, their current state, and the network ports they are listening on.

Use the `docker-compose top` command to list the processes running inside of each service (container).

```
$ docker-compose top
counterapp_redis_1
PID     USER     TIME     COMMAND
------------------------------------
`843`   dockrema   `0`:00   redis-server

counterapp_web-fe_1
PID    USER   TIME             COMMAND
-------------------------------------------------
`928`    root   `0`:00   python app.py
`1016`   root   `0`:00   /usr/local/bin/python app.py 
```

 `The PID numbers returned are the PID numbers as seen from the Docker host (not from within the containers).

Use the `docker-compose stop` command to stop the app without deleting its resources. Then show the status of the app with `docker-compose ps`.

```
$ docker-compose stop
Stopping counterapp_web-fe_1 ... `done`
Stopping counterapp_redis_1  ... `done`

$ docker-compose ps
Name                  Command                      State
---------------------------------------------------------
counterapp_redis_1    docker-entrypoint.sh redis   Exit `0`
counterapp_web-fe_1   python app.py                Exit `0` 
```

 `As we can see, stopping a Compose app does not remove the application definition from the system. It just stops the app’s containers. You can verify this with the `docker container ls -a` command.

You can delete a stopped Compose app with the `docker-compose rm` command. This will delete the containers and networks the app is using, but it will not delete volumes or images. Nor will it delete the application source code (`app.py`, `Dockerfile`, `requirements.txt`, and `docker-compose.yml`) in your project directory.

Restart the app with the `docker-compose restart` command.

```
$ docker-compose restart
Restarting counterapp_web-fe_1 ... `done`
Restarting counterapp_redis_1  ... `done` 
```

 `Verify the operation.

```
$ docker-compose ps
Name                  Command               State   Ports
--------------------------------------------------------------------------
counterapp_redis_1    docker-entrypoint...  Up      `6379`/tcp
counterapp_web-fe_1   python app.py         Up      `0`.0.0.0:5000->5000/tcp 
```

 `Use the `docker-compose down` command to **stop and delete** the app with a single command.

```
$ docker-compose down
Stopping counterapp_web-fe_1 ... `done`
Stopping counterapp_redis_1  ... `done`
Removing counterapp_web-fe_1 ... `done`
Removing counterapp_redis_1  ... `done`
Removing network counterapp_counter-net 
```

 `The app is now deleted. Only its images, volumes and source code remain.

Let’s deploy the app one last time and see its volume in action.

```
$ docker compose up -d
Creating network `"counterapp_counter-net"` with the default driver
Creating counterapp_redis_1  ... `done`
Creating counterapp_web-fe_1 ... `done` 
```

 `If you look in the Compose file, you’ll see that we’re defing a new volume called `counter-vol` and mounting it in to the `web-fe` service at `/code`.

```
`services``:`
  `web``-``fe``:`
  `<``Snip``>`
    `volumes``:`
      `-` `type``:` `volume`
        `source``:` `counter``-``vol`
        `target``:` `/``code`
`<``Snip``>`
`volumes``:`
  `counter``-``vol``:` 
```

 `The first time we deployed the app, Compose checked to see if a volume already existed with this name. It did not, so it created it. You can see it with the `docker volume ls` command.

```
$ docker volume ls
RIVER              VOLUME NAME
`local`               counterapp_counter-vol 
```

 `It’s also worth knowing that Compose builds networks and volumes **before** deploying services. This makes sense, as they are lower-level infrastructure objects that are consumed by services (containers). The following snippet shows Compose creating the network and volume as its first two tasks (even before building and pulling images).

```
$ docker-compose up -d

Creating network `"counterapp_counter-net"` with the default driver
Creating volume `"counterapp_counter-vol"` with default driver
Pulling redis `(`redis:alpine`)`...
<Snip> 
```

 `If we take another look at the service definition for `web-fe`, we’ll see that it’s mounting the counter-app volume into the service’s container at `/code`. We can also see from the Dockerfile that `/code` is where the app is installed and executed from. Net result, our app code resides on a Docker volume.

![](img/figure9-2.png)

This all means we can make changes to files in the volume, from the host side, and have them reflected immediately in the app. Let’s see it.

The next few steps will walk through the following process. We’ll edit the `app.py` file in the project’s working directory so that the app will display different text in the web browser. We’ll copy updated app to the volume on the Docker host. We’ll refresh the app’s web page to see the updated text. This will work because whatever you write to the location of the volume on the Docker host will immediately appear in the volume in the container.

Use you favourite text editor to edit the `app.py` file in the projects working directory. We’ll use `vim` in the example.

```
$ vim ~/counter-app/app.py 
```

 `Change text between the double quote marks (“”) on line 22\. The line starts with `return "What's up..."`. Enter any text you like, as long as it’s within the double-quote marks, and save your changes.

Now that we’ve updated the app, we need to copy it into the volume on the Docker host. Each Docker volume is exposed at a location within the Docker host’s filesystem, as well as a mount point in one or more containers. Use the following `docker volume inspect` command to find where the volume is exposed on the Docker host.

```
$ docker volume inspect counterapp_counter-vol `|` grep Mount

`"Mountpoint"`: `"/var/lib/docker/volumes/counterapp_counter-vol/_data"`, 
```

 `Copy the updated app file to the volume’s mount point on your Docker host. This will make it appear in the `web-fe` container at `/code`. The operation will overwrite the existing `/code/app.py` file in the container.

```
$ cp ~/counterapp/app.py `\`
  /var/lib/docker/volumes/counterapp_counter-vol/_data/app.py 
```

 `The updated app file is now on the container. Connect to the app to see your change. You can do this by pointing your web browser to the IP of your Docker host on port 5000.

Figure 9.3 shows the updated app.

![](img/figure9-3.png)

Obviously you wouldn’t do this in production, but it’s a real time-saver in development.

Congratulations! You’ve deployed and managed a simple multi-container app using Docker Compose.

Before reminding ourselves of the major commands we learned, it’s important to understand that this was a very simple example. Docker Compose is capable of deploying and managing far more complex applications.

### Deploying apps with Compose - The commands

*   `docker-compose up` is the command we use to deploy a Compose app. It expects the Compose file to be called `docker-compose.yml` or `docker-compose.yaml`, but you can specify a custom filename with the `-f` flag. It’s common to start the app in the background with the `-d` flag.
*   `docker-compose stop` will stop all of the containers in a Compose app without deleting them from the system. The app can be easily restarted with `docker-compose restart`.
*   `docker-compose rm` will delete a stopped Compose app. It will delete containers and networks, but it will not delete volumes and images.
*   `docker-compose restart` will restart a Compose app that has been stopped with `docker-compose stop`. If you have made changes to your Compose app since stopping it, these changes will **not** appear in the restarted app. You will need to re-deploy the app to get the changes.
*   `docker-compose ps` will list each container in the Compose app. It shows current state, the command each one is running, and network ports.
*   `docker-compose down` will stop and delete a running Compose app. It deletes containers and networks, but not volumes and images.

### Chapter Summary

In this chapter, we learned how to deploy and manage a multi-container application using Docker Compose.

Docker Compose is a Python application that we install on top of the Docker Engine. It lets us define multi-container apps in a single declarative configuration file and deploy it with a single command.

Compose files can be YAML or JSON, and they define all of the containers, networks, volumes, and secrets that an application requires. We then feed the file to the `docker-compose` command line tool, and Compose instructs Docker to deploy it.

Once the app is deployed, we can manage its entire lifecycle using the many `docker-compose` sub-commands.

We also saw how volumes can be used to mount changes directly into containers.

Docker Compose is very popular with developers, and the Compose file is an excellent source of application documentation — it defies all the services that make up the app, the images they use, ports they expose, networks and volumes they use, and much more. As such, it can help bridge the gap between dev and ops. You should also treat your Compose files as if they were code. This means, among other things, storing them in source control repos.``````````````````````````````````

# 第四章：安装 Docker

有很多种方式和地方可以安装 Docker。有 Windows，有 Mac，显然还有 Linux。但也有云端，本地，笔记本电脑等等...除此之外，我们还有手动安装，脚本安装，基于向导的安装...实际上有很多种方式和地方可以安装 Docker！

但不要让这吓到你！它们都很容易。

在本章中，我们将涵盖一些最重要的安装：

+   桌面安装

+   Docker for Windows

+   Docker for Mac

+   服务器安装

+   Linux

+   Windows Server 2016

+   升级 Docker

+   存储驱动器考虑

我们还将看看如何升级 Docker 引擎并选择适当的存储驱动程序。

### Windows 的 Docker（DfW）

首先要注意的是*Docker for Windows*是 Docker，Inc.的“打包”产品。这意味着它很容易下载，并且有一个漂亮的安装程序。它在 64 位 Windows 10 台式机或笔记本上启动一个单引擎 Docker 环境。

第二件要注意的事情是它是一个社区版（CE）应用。因此不适用于生产。

第三件值得注意的事情是它可能会遇到一些功能滞后。这是因为 Docker，Inc.正在采取“稳定性第一，功能第二”的方法来处理产品。

这三点加起来就是一个快速简单的安装，但不适用于生产。

废话够多了。让我们看看如何安装*Docker for Windows*。

首先，先决条件。*Docker for Windows*需要：

+   Windows 10 专业版|企业版|教育版（1607 周年更新，版本 14393 或更新版本）

+   必须是 64 位 Windows 10

+   *Hyper-V*和*容器*功能必须在 Windows 中启用

+   必须在系统的 BIOS 中启用硬件虚拟化支持

以下将假定硬件虚拟化支持已在系统的 BIOS 中启用。如果没有，您应该仔细遵循您特定机器的程序。

在 Windows 10 中要做的第一件事是确保**Hyper-V**和**容器**功能已安装并启用。

1.  右键单击 Windows 开始按钮，然后选择“应用和功能”。

1.  点击“程序和功能”链接（右侧的一个小链接）。

1.  点击“打开或关闭 Windows 功能”。

1.  勾选`Hyper-V`和“容器”复选框，然后点击“确定”。

这将安装并启用 Hyper-V 和容器功能。您的系统可能需要重新启动。

![图 3.1](img/Figure3-1.png)

图 3.1

*Containers*功能仅在运行 2016 年夏季的 Windows 10 周年更新（版本 14393）或更高版本时可用。

安装了`Hyper-V`和`Containers`功能并重新启动计算机后，现在是安装*Docker for Windows*的时候了。

1.  前往 https://www.docker.com/get-docker 并单击`GET DOCKER COMMUNITY EDITION`链接。

1.  单击`DOCKER CE FOR WINDOWS`部分下方的`Download from Docker Store`链接。这将带您到 Docker Store，您可能需要使用 Docker ID 登录。

1.  单击一个`Get Docker`下载链接。

Docker for Windows 有一个*stable*和*edge*通道。Edge 通道包含更新的功能，但可能不太稳定。

一个名为`Docker for Windows Installer.exe`的安装程序包将被下载到默认的下载目录。

1.  找到并启动在上一步中下载的安装程序包。

按照安装向导的步骤，并提供本地管理员凭据以完成安装。Docker 将自动启动为系统服务，并且 Moby Dock 鲸鱼图标将出现在 Windows 通知区域中。

恭喜！您已经安装了*Docker for Windows*。

打开命令提示符或 PowerShell 终端，尝试以下命令：

```
```

客户端：

版本：18.01.0-ce

API 版本：1.35

Go 版本：go1.9.2

Git 提交：03596f5

构建时间：2018 年 1 月 10 日星期三 20:05:55

OS/Arch：windows/amd64

实验性：false

编排器：swarm

服务器：

引擎：

版本：18.01.0-ce

API 版本：1.35（最低版本 1.12）

Go 版本：go1.9.2

Git 提交：03596f5

构建时间：2018 年 1 月 10 日星期三 20:13:12

OS/Arch：linux/amd64

实验性：false

``` 
```

注意，输出显示`OS/Arch: linux/amd64`是因为默认安装目前会在轻量级的 Linux Hyper-V 虚拟机中安装 Docker 守护程序。在这种情况下，您只能在*Docker for Windows*上运行 Linux 容器。

如果您想运行*本机 Windows 容器*，可以右键单击 Windows 通知区域中的 Docker 鲸鱼图标，然后选择`Switch to Windows containers...`。您也可以在命令行中使用以下命令（位于`\Program Files\Docker\Docker`目录中）来实现相同的功能：

```
C:\Program Files\Docker\Docker> .\dockercli -SwitchDaemon 
```

如果您没有启用`Windows Containers`功能，您将收到以下警报。

![图 3.2](img/figure3-2.png)

图 3.2

如果您已经启用了 Windows 容器功能，切换只需要几秒钟。切换完成后，`docker version`命令的输出将如下所示。

```
C:\> docker version
Client:
 <Snip>

Server:
 Engine:
  Version:      18.01.0-ce
  API version:  1.35 (minimum version 1.24)
  Go version:   go1.9.2
  Git commit:   03596f5
  Built:        Wed Jan 10 20:20:36 2018
  OS/Arch:      windows/amd64
  Experimental: true 
```

`请注意，服务器版本现在显示为`windows/amd64`。这意味着守护程序在 Windows 内核上本地运行，并且只能运行 Windows 容器。

还要注意，系统现在正在运行 Docker 的*实验*版本（`实验性：true`）。如前所述，*Docker for Windows*有一个稳定的通道和一个边缘通道。在撰写本文时，Windows 容器是边缘通道的实验功能。

您可以使用`dockercli -Version`命令检查您正在运行的通道。 `dockercli`命令位于`C:\Program Files\Docker\Docker`中。

```
PS C:\Program Files\Docker\Docker> .\dockercli -Version

Docker for Windows
Version: 18.01.0-ce-win48 (15285)
Channel: edge
Sha1: ee2282129dec07b8c67890bd26865c8eccdea88e
OS Name: Windows 10 Pro
Windows Edition: Professional
Windows Build Number: 16299 
```

以下列表显示了常规 Docker 命令的正常工作。

```
> docker image ls
REPOSITORY    TAG      IMAGE ID      CREATED       SIZE

> docker container ls
CONTAINER ID   IMAGE   COMMAND   CREATED    STATUS    PORTS    NAMES

> docker system info
Containers: 1
 Running: 0
 Paused: 0
 Stopped: 1
Images: 6
Server Version: 17.12.0-ce
Storage Driver: windowsfilter
<Snip> 
```

Windows 的 Docker 包括 Docker Engine（客户端和守护程序）、Docker Compose、Docker Machine 和 Docker Notary 命令行。使用以下命令验证每个是否成功安装：

```
C:\> docker --version
Docker version 18.01.0-ce, build 03596f5 
```

````
C:\> docker-compose --version
docker-compose version 1.18.0, build 8dd22a96 
```

````

C:\> docker-machine --version

docker-machine.exe 版本 0.13.0，构建 9ba6da9

```

````

C:\> notary version

公证

版本：0.4.3

Git 提交：9211198

```

 `### Docker for Mac (DfM)

*Docker for Mac* is also a packaged product from Docker, Inc. So relax, you don’t need to be a kernel engineer, and we’re not about to walk through a complex hack for getting Docker onto your Mac. Installing DfM is ridiculously easy.

What is *Docker for Mac?*

First up, *Docker for Mac* is a packaged product from Docker, Inc. that is based on the Community Edition of Docker. This means it’s an easy way to install a single-engine version of Docker on you Mac. It also means that it’s not intended for production use. If you’ve heard of **boot2docker**, then *Docker for Mac* is what you always wished *boot2docker* was — smooth, simple, and stable.

It’s also worth noting that *Docker for Mac* will not give you the Docker Engine running natively on the Mac OS Darwin kernel. Behind the scenes, the Docker daemon is running inside a lightweight Linux VM. It then seamlessly exposes the daemon and API to your Mac environment. This means you can open a terminal on your Mac and use the regular Docker commands.

Although this works seamlessly on your Mac, don’t forget that it’s Docker on Linux under the hood — so it’s only going work with Linux-based Docker containers. This is good though, as it’s where most of the container action is.

Figure 3.3 shows a high-level representation of the *Docker for Mac* architecture.

![Figure 3.3](img/figure3-3.png)

Figure 3.3

> **Note:** For the curious reader, *Docker for Mac* leverages [HyperKit](https://github.com/docker/hyperkit) to implement an extremely lightweight hypervisor. HyperKit is based on the [xhive hypervisor](https://github.com/mist64/xhyve). *Docker for Mac* also leverages features from [DataKit](https://github.com/docker/datakit) and runs a highly tuned Linux distro called *Moby* that is based on [Alpine Linux](https://alpinelinux.org/%20and%20https://github.com/alpinelinux).

Let’s get *Docker for Mac* installed.

1.  Point your browser to https://www.docker.com/get-docker and click `GET DOCKER COMMUNITY EDITION`.
2.  Click the `Download from Docker Store` option below `DOCKER CE FOR MAC`. This will take you to the Docker Store and you will need to provide your Docker ID and password.
3.  Click one of the `Get Docker CE` download links.

    Docker for Mac has a stable and edge channel. Edge has newer features, at the expense of stability.

    A **Docker.dmg** installation package will be downloaded.

4.  Launch the `Docker.dmg` file that you downloaded in the previous step. You will be asked to drag and drop the Moby Dock whale image into the **Applications** folder.
5.  Open your **Applications** folder (it may open automatically) and double-click the Docker application icon to Start it. You may be asked to confirm the action because the application was downloaded from the internet.
6.  Enter your password so that the installer can create the components that require elevated privileges.
7.  The Docker daemon will now start.

    An animated whale icon will appear in the status bar at the top of your screen while Docker starts. Once Docker has successfully started, the whale will stop being animated. You can click the whale icon to manage DfM.

Now that DfM is installed, you can open a terminal window and run some regular Docker commands. Try the following.

```

$ docker version

客户端：

版本：17.05.0-ce

API 版本：1.29

Go 版本：go1.7.5

Git 提交：89658be

构建日期：星期四，2017 年 5 月 4 日 21:43:09

OS/Arch：darwin/amd64

服务器：

版本：17.05.0-ce

API 版本：1.29（最低版本 1.12）

Go 版本：go1.7.5

Git 提交：89658be

构建日期：星期四，2017 年 5 月 4 日 21:43:09

OS/Arch：linux/amd64

实验性：true

```

 `Notice that the `OS/Arch:` for the **Server** component is showing as `linux/amd64`. This is because the daemon is running inside of the Linux VM we mentioned earlier. The **Client** component is a native Mac application and runs directly on the Mac OS Darwin kernel (`OS/Arch: darwin/amd64`).

Also note that the system is running the experimental version (`Experimental: true`) of Docker. This is because the system is running the *edge* channel which comes with experimental features turned on.

Run some more Docker commands.

```

$ docker --version

Docker 版本 17.05.0-ce，构建 89658be

$ docker image ls

存储库    标签    映像 ID    创建时间    大小

$ docker container ls

容器 ID   映像   命令   创建时间   状态   端口   名称

```

 `Docker for Mac installs the Docker Engine (client and daemon), Docker Compose, Docker machine, and the Notary command line. The following three commands show you how to verify that all of these components installed successfully, as well as which versions you have.

```

$ docker --version

Docker 版本 17.05.0-ce，构建 89658be

```

````

$ docker-compose --version

docker-compose 版本 1.13.0，构建 1719ceb

```

````

$ docker-machine --version

docker-machine 版本 0.11.0，构建 5b27455

```

````

$ notary version

公证

版本：0.4.3

Git 提交：9211198

```

 `### Installing Docker on Linux

Installing Docker on Linux is the most common installation type and it’s surprisingly easy. The most common difficulty is the slight variations between Linux distros such as Ubuntu vs CentOS. The example we’ll use in this section is based on Ubuntu Linux, but should work on upstream and downstream forks. It should also work on CentOS and its upstream and downstream forks. It makes absolutely no difference if your Linux machine is a physical server in your own data center, on the other side of the planet in a public cloud, or a VM on your laptop. The only requirements are that the machine be running Linux and has access to https://get.docker.com.

The first thing you need to decide is which edition to install. There are currently two editions:

*   Community Edition (CE)
*   Enterprise Edition (EE)

Docker CE is free and is the version we’ll be demonstrating. Docker EE is the same as CE, but comes with commercial support and access to other Docker products such as Docker Trusted Registry and Universal Control Plane.

In this example, we’ll use the `wget` command to call a shell script that installs Docker CE. For information on other ways to install Docker on Linux, go to https://www.docker.com and click on `Get Docker`.

> **Note:** You should ensure that your system is up-to-date with the latest packages and security patches before continuing.

1.  Open a new shell on your Linux machine.
2.  Use `wget` to retrieve and run the Docker install script from

`https://get.docker.com` and pipe it through your shell.

```

$ wget -qO- https://get.docker.com/ `|` sh

modprobe：致命：未找到模块 aufs/lib/modules/4.4.0-36-generic

+ sh -c 'sleep 3; yum -y -q install docker-engine'

<剪辑>

如果您想以非 root 用户身份使用 Docker，您应该

现在考虑将您的用户添加到`"docker"`组中

类似于：

sudo usermod -aG docker your-user

请记住，您将不得不注销并重新登录...

```

`*   It is best practice to use non-root users when working with Docker. To do this, you need to add your non-root users to the local `docker` Unix group. The following command shows you how to add the **npoulton** user to the `docker` group and verify that the operation succeeded. You will need to use a valid user account on your own system.

```

$ sudo usermod -aG docker npoulton

$ cat /etc/group `|` grep docker

docker:x:999:npoulton

```

     `If you are already logged in as the user that you just added to the `docker` group, you will need to log out and log back in for the group membership to take effect.`` 

 ``Congratulations! Docker is now installed on your Linux machine. Run the following commands to verify the installation.

```

$ docker --version

Docker 版本`18`.01.0-ce，构建 03596f5

$ docker system info

容器：`0`

运行中：`0`

已暂停：`0`

已停止：`0`

镜像：`0`

服务器版本：`18`.01.0-ce

存储驱动程序：overlay2

后备文件系统：extfs

<Snip>

```

 `If the process described above doesn’t work for your Linux distro, you can go to the [Docker Docs](https://docs.docker.com/engine/installation/) website and click on the link relating to your distro. This will take you to the official Docker installation instructions which are usually kept up to date. Be warned though, the instructions on the Docker website tend use package managers that require a lot more steps than the procedure we used above. In fact, if you open a web browser to https://get.docker.com you will see that it’s a shell script that does all of the installation grunt-work for you — including configuring Docker to automatically start when the system boots.

> **Warning:** If you install Docker from a source other than the official Docker repositories, you may end up with a forked version of Docker. In the past, some vendors and distros chose to fork the Docker project and develop their own slightly customized versions. You need to watch out for things like this, as you could unwittingly end up in a situation where you are running a fork that has diverged from the official Docker project. This isn’t a problem if this is what you intend to do. If it is not what you intend, it can lead to situations where modifications and fixes your vendor makes do not make it back upstream in to the official Docker project. In these situations, you will not be able to get commercial support for your installation from Docker, Inc. or its authorized service partners.

### Installing Docker on Windows Server 2016

In this section we’ll look at one of the ways to install Docker on Windows Server 2016\. We’ll complete the following high-level steps:

1.  Install the Windows Containers feature
2.  Install Docker
3.  Verify the installation

Before proceeding, you should ensure that your system is up-to-date with the latest package versions and security updates. You can do this quickly with the `sconfig` command and choosing option 6 to install updates. This may require a system restart.

We’ll be demonstrating an installation on a version of Windows Server 2016 that does not have the Containers feature or an older version of Docker already installed.

Ensure that the `Containers` feature is installed and enabled.

1.  Right-click the Windows Start button and select `Programs and Features`. This will open the `Programs and Features` console.
2.  Click `Turn Windows features on or off`. This will open the `Server Manager` app.
3.  Make sure the `Dashboard` is selected and choose `Add Roles and Features`.
4.  Click through the wizard until you get to the `Features` page.
5.  Make sure that the `Containers` feature is checked, then complete the wizard. Your system may require a system restart.

Now that the Windows Containers feature is installed, you can install Docker. We’ll use PowerShell to do this.

1.  Open a new PowerShell Administrator terminal.
2.  Use the following command to install the Docker package management provider.

```

> Install-Module DockerProvider -Force

```

 `If prompted, accept the request to install the NuGet provider.` 
`*   Install Docker.

```

> Install-Package Docker -ProviderName DockerProvider -Force

```

 `Once the installation is complete you will get a summary as shown.

```

名称 版本 来源 摘要

---- ------- ------ -------

Docker 17.06.2-ee-6 Docker Docker for Windows Server 2016

```

     `Docker is now installed and configured to automatically start when the system boots.`` ``*   You may want to restart your system to make sure that none of changes have introduced issues that cause your system not to boot. You can also check that Docker automatically starts after the reboot.```

```Docker is now installed and you can start deploying containers. The following two commands are good ways to verify that the installation succeeded.

```

> docker --version

Docker 版本 17.06.2-ee-6，构建 e75fdb8

> docker system info

容器：0

运行中：0

已暂停：0

已停止：0

镜像：0

服务器版本：17.06.2-ee-6

存储驱动程序：windowsfilter

<Snip>

```

 `Docker is now installed and you are ready to start using Windows containers.

### Upgrading the Docker Engine

Upgrading the Docker Engine is an important task in any Docker environment — especially production. This section of the chapter will give you the high-level process of upgrading the Docker engine, as well as some general tips and a couple of upgrade examples.

The high-level process of upgrading the Docker Engine is this:

Take care of any pre-requisites. These can include; making sure your containers have an appropriate restart policy, or draining nodes if you’re using *Services* in Swarm mode. Once you’ve completed any potential pre-requisites you can follow the procedure below.

1.  Stop the Docker daemon
2.  Remove the old version
3.  Install the new version
4.  configure the new version to automatically start when the system boots
5.  Ensure containers have restarted

That’s the high-level process. Let’s look at some examples.

Each version of Linux has its own slightly different commands for upgrading Docker. We’ll show you Ubuntu 16.04\. We’ll also show you Windows Server 2016.

#### Upgrading Docker CE on Ubuntu 16.04

We’re assuming you’ve completed all pre-requisites and your Docker host is ready for the upgrade. We’re also assuming you’re running commands as root. Running commands as root is obviously **not recommended**, but it does keep examples in the book simpler. If you’re not running as root, well done! However, you will have to prepend the following commands with `sudo`.

1.  Update your `apt` package list.

```

$ apt-get update

```

`*   Uninstall existing versions of Docker.

```

$ apt-get remove docker docker-engine docker-ce docker.io -y

```

 `The Docker engine has had several different package names in the past. This command makes sure all older versions get removed.` `*   Install the new version.

There are different versions of Docker and different ways to install each one. For example, Docker CE or Docker EE, both of which can be installed in more than one way. For example, Docker CE can be installed from `apt` or `deb` packages, or using a script on `docker.com`

The following command will use a shell script at `get.docker.com` to install and configure the latest version of Docker CE.

```

$ wget -qO- https://get.docker.com/ | sh

```

`*   Configure Docker to automatically start each time the system boots.

```

$ systemctl enable docker

同步 docker.service 的状态...

执行/lib/systemd/systemd-sysv-install enable docker

$ systemctl is-enabled docker

已启用

```

 `At this point you might want to restart the node. This will make sure that no issues have been introduced that prevent your system from booting in the future.` `*   Make sure any containers and services have restarted.

```

$ docker container ls

容器 ID 图像 命令 创建状态 \

名称

97e599aca9f5 alpine "sleep 1d" 14 分钟前 上线 1 分钟

$ docker service ls

ID 名称 模式 副本 图像

ibyotlt1ehjy prod-equus1 复制 1/1 alpine:latest

``````` 

```Remember, other methods of upgrading and installing Docker exist. We’ve just shown you one way, on Ubuntu Linux 16.04.

#### Upgrading Docker EE on Windows Server 2016

This section will walk you through the process of upgrading Docker on Windows from 1.12.2, to the latest version of Docker EE.

The process assumes you have completed any pre-flight tasks, such as configuring containers with appropriate restart policies and draining Swarm nodes if you’re using Swarm services.

All commands should be ran from a PowerShell terminal.

1.  Check the current version of Docker.

   ```
    > docker version
    Client:
     Version:      1.12.2-cs2-ws-beta
    <Snip>
    Server:
     Version:      1.12.2-cs2-ws-beta 
   ```

`*   Uninstall any potentially older modules provided by Microsoft, and install the module from Docker.

   ```
    > Uninstall-Module DockerMsftProvider -Force

    > Install-Module DockerProvider -Force 
   ```

   `*   Update the `docker` package.

   This command will force the update (no uninstall is required) and configure Docker to automatically start each time the system boots.

   ```
    > Install-Package -Name docker -ProviderName DockerProvider -Update -Force

    Name      Version          Source       Summary
    ----      -------          ------       -------
    Docker    17.06.2-ee-6     Docker       Docker for Windows Server 2016 
   ```

    `You might want to reboot your server at this point to make sure the changes have not introduced any issues that prevent it from restarting in the future.` `*   Check that containers and services have restarted.```

```That’s it. That’s how to upgrade to the latest version of Docker EE on Windows Server 2016.

### Docker and storage drivers

Every Docker container gets its own area of local storage where image layers are stacked and the container filesystem is mounted. By default, this is where all container read/write operations occur, making it integral to the performance and stability of every container.

Historically, this local storage area has been managed by the *storage driver*, which we sometimes call the *graph driver* or *graphdriver*. Although the high-level concepts of stacking image layers and using copy-on-write technologies are constant, Docker on Linux supports several different storage drivers, each of which implements layering and copy-on-write in its own way. While these *implementation differences* do not affect the way we *interact* with Docker, they can have a significant impact on *performance* and *stability*.

Some of the *storage drivers* available for Docker on Linux include:

*   `aufs` (the original and oldest)
*   `overlay2` (probably the best choice for the future)
*   `devicemapper`
*   `btrfs`
*   `zfs`

Docker on Windows only supports a single storage driver, the `windowsfilter` driver.

Selecting a storage driver is a *per node* decision. This means a single Docker host can only run a single storage driver — you cannot select the storage driver per-container. On Linux, you set the storage driver in `/etc/docker/daemon.json` and you need to restart Docker for any changes to take effect. The following snippet shows the storage driver set to `overlay2`.

```
{
 "storage-driver": "overlay2"
} 
```

`> **Note:** If the configuration line is not the last line in the configuration file, you will need to add a comma to the end.

If you change the storage driver on an already-running Docker host, existing images and containers will not be available after Docker is restarted. This is because each storage driver has its own subdirectory on the host where it stores image layers (usually below `/var/lib/docker/<storage-driver>/...`). Changing the storage driver obviously changes where Docker looks for images and containers. Reverting the storage driver to the previous configuration will make the older images and containers available again.

If you need to change your storage driver, and you need your images and containers to be available after the change, you need to save them with `docker save`, push the saved images to a repo, change the storage driver, restart Docker, pull the images locally, and restart your containers.

You can check the current storage driver with the `docker system info` command:

```
$ docker system info
<Snip>
Storage Driver: overlay2
 Backing Filesystem: xfs
 Supports d_type: `true`
 Native Overlay Diff: `true`
<Snip> 
```

`Choosing which storage driver, and configuring it properly, is important in any Docker environment — especially production. The following list can be used as a **guide** to help you choose which storage driver to use. However, you should always consult the latest support documentation from Docker, as well as your Linux provider.

*   **Red Hat Enterprise Linux** with a 4.x kernel or higher + Docker 17.06 and higher: `overlay2`
*   **Red Hat Enterprise Linux** with an older kernel and older versions of Docker: `devicemapper`
*   **Ubuntu Linux** with a 4.x kernel or higher: `overlay2`
*   **Ubuntu Linux** with an earlier kernel: `aufs`
*   **SUSE Linux Enterprise Server:** `btrfs`

Again, this list should only be used as a guide. Always check the latest support and compatibility matrixes in the Docker documentation, and with your Linux provider. This is especially important if you are using Docker Enterprise Edition (EE) with a support contract.

#### Devicemapper configuration

Most of the Linux storage drivers require little or no configuration. However, `devicemapper` needs configuring in order to perform well.

By default, `devicemapper` uses *loopback mounted sparse files* to underpin the storage it provides to Docker. This is fine for a smooth out-of-the box experience that *just works*. But it’s terrible for production. In fact, it’s so bad that it’s **not supported on production systems!

To get the best performance out of `devicemapper`, as well as production support, you must configure it in `direct-lvm` mode. This significantly increases performance by leveraging an LVM `thinpool` backed by raw block devices.

Docker 17.06 and higher can configure `direct-lvm` for you. However, at the time of writing, it has some limitations. The main ones being; it will only configure a single block device, and it only works for fresh installations. This might change in the future, but a single block device will not give you the best in terms of performance and resiliency.

##### Letting Docker automatically configure `direct-lvm`

The following simple procedure will let Docker automatically configure `devicemapper` for `direct-lvm`.

1.  Add the following storage driver configuration to `/etc/docker/daemon.json`

   ```
    {
    "storage-driver": "devicemapper",
    "storage-opts": [
      "dm.directlvm_device=/dev/xdf",
      "dm.thinp_percent=95",
      "dm.thinp_metapercent=1",
      "dm.thinp_autoextend_threshold=80",
      "dm.thinp_autoextend_percent=20",
      "dm.directlvm_device_force=false"
    ]
    } 
   ```

    `Device Mapper and LVM are complex topics, and beyond the scope of a heterogeneous Docker book like this. However, let’s quickly explain each option:

   *   `dm.directlvm_device` is where you specify the block device. For best performance and availability, this should be a dedicated high-performance device such as a local SSD, or RAID protected high performance LUN from an external storage array.
   *   `dm.thinp_percent=95` allows you to specify how much of the space you want Images and containers to be able to use. Default is 95%.
   *   `dm.thinp_metapercent` sets the percentage of space to be used for metadata storage. Default is 1%.
   *   `dm.thinp_autoextend_threshold` sets the threshold at which LVM should automatically extend the thinpool. The default value is currently 80%.
   *   `dm.thinp_autoextend_percent` is the amount of space that should be added to the thin pool when an auto-extend operation is triggered.
   *   `dm.directlvm_device_force` lets you specify whether or not to format the block device with a new filesystem.` 
`*   Restart Docker.*   Verify that Docker is running and the `devicemapper` configuration is correctly loaded.

   ```
    $ docker version
    Client:
    Version:      18.01.0-ce
    <Snip>
    Server:
    Version:      18.01.0-ce
    <Snip>

    $ docker system info
    <Snipped output only showing relevant data>
    Storage Driver: devicemapper
    Pool Name: docker-thinpool
    Pool Blocksize: 524.3 kB
    Base Device Size: 25 GB
    Backing Filesystem: xfs
    Data file:       << Would show a loop file if in loopback mode
    Metadata file:   << Would show a loop file if in loopback mode
    Data Space Used: 1.9 GB
    Data Space Total: 23.75 GB
    Data Space Available: 21.5 GB
    Metadata Space Used: 180.5 kB
    Metadata Space Total: 250 MB
    Metadata Space Available: 250 MB 
   ```` 

``Although Docker will only configure `direct-lvm` mode with a single block device, it will still perform significantly better than `loopback` mode!

##### Manually configuring devicemapper direct-lvm

Walking you through the entire process of manually configuring `device mapper direct-lvm` is beyond the scope of this book. It is also something that can change and vary between OS versions. However, the following items are things you should know and consider when performing a configuration.

*   **Block devices**. You need to have block devices available in order to configure `direct-lvm` mode. These should be high performance devices such as local SSD or high performance external LUNs. If your Docker environment is on-premises, external LUNs can be on FC, iSCSI, or other block-protocol storage arrays. If your Docker environment is in the public cloud, these can be any form of high performance block storage (usually SSD-based) supported by your cloud provider.
*   **LVM config**. The `devicemapper` storage driver leverages LVM, the Linux Logical Volume Manager. This means you will need to configure the required physical devices (pdev), volume group (vg), logical volumes (lv), and thinpool (tp). You should use dedicated physical volumes and form them into a new volume group. You should not share the volume group with non-Docker workloads. You will also need to configure two logical volumes; one for data and the other for metadata. Create an LVM profile specifying the auto-extend threshold and auto-extend values, and configure monitoring so that auto-extend operations can happen.
*   **Docker config**. Backup the current Docker config file (`/etc/docker/daemon.json`) and then update it as follows. The name of the `dm.thinpooldev` might be different in your environment and you should adjust as appropriate.

   ```
    {
      "storage-driver": "devicemapper",
      "storage-opts": [
      "dm.thinpooldev=/dev/mapper/docker-thinpool",
      "dm.use_deferred_removal=true",
      "dm.use_deferred_deletion=true"
      ]
    } 
   ```

`Once the configuration is saved you can start the Docker daemon.

For more detailed information, see the Docker documentation or talk to your Docker technical account manager.

### Chapter Summary

Docker is available for Linux and Windows, and has a Community Edition (CE) and an Enterprise Edition (EE). In this chapter, we looked at some of the ways to install Docker on Windows 10, Mac OS X, Linux, and Windows Server 2016.

We looked at how to upgrade the Docker Engine on Ubuntu 16.04 and Windows Server 2016, as these are two of the most common configurations.

We also learned that selecting the right *storage driver* is essential when using Docker on Linux in production environments.`````````````````````````````````

## 16：企业工具

在本章中，我们将看一些 Docker 公司提供的企业级工具。我们将看到如何安装它们，配置它们，备份它们和恢复它们。

这将是一个相当长的章节，包含大量逐步的技术细节。我会尽量保持有趣，但这可能会很难:-D

其他工具也存在，但我们将集中在 Docker 公司的工具上。

让我们直接开始。

### 企业工具-简而言之

Docker 和容器已经席卷了应用程序开发世界-构建，发布和运行应用程序从未如此简单。因此，企业想要参与其中并不奇怪。但企业有比典型的前沿开发者更严格的要求。

企业需要以他们可以使用的方式打包 Docker。这通常意味着他们拥有和管理自己的本地解决方案。这还意味着角色和安全功能，使其适应其内部结构，并得到安全部门的认可。这也意味着一切都有一个有意义的支持合同支持。

这就是 Docker 企业版（EE）发挥作用的地方！

Docker EE 是*企业版的 Docker*。它是一套产品，包括一个经过加固的引擎，一个操作界面和一个安全的私有注册表。您可以在本地部署它，并且它包含在支持合同中。

高级堆栈如图 16.1 所示。

![图 16.1 Docker EE](img/figure16-1.png)

图 16.1 Docker EE

### 企业工具-深入挖掘

我们将把本章的其余部分分为以下几个部分：

+   Docker EE 引擎

+   Docker Universal Control Plane (UCP)

+   Docker Trusted Registry (DTR)

我们将看到如何安装每个工具，并在适用的情况下配置 HA，并执行备份和恢复作业。

#### Docker EE 引擎

*Docker 引擎*提供了所有核心的 Docker 功能。像镜像和容器管理，网络，卷，集群，秘密等等。在撰写本文时，有两个版本：

+   社区版（CE）

+   企业版（EE）

就我们而言，最大的两个区别是发布周期和支持。

Docker EE 按季度发布，并使用基于时间的版本方案。例如，2018 年 6 月发布的 Docker EE 将是`18.06.x-ee`。Docker 公司保证每个版本提供 1 年的支持和补丁。

##### 安装 Docker EE

安装 Docker EE 很简单。但是，不同平台之间存在细微差异。我们将向您展示如何在 Ubuntu 16.04 上进行操作，但在其他平台上进行操作同样简单。

Docker EE 是基于订阅的服务，因此您需要一个 Docker ID 和一个活跃的订阅。这将为您提供访问一个独特个性化的 Docker EE 仓库，我们将在接下来的步骤中配置和使用。[试用许可证](https://store.docker.com/editions/enterprise/docker-ee-trial)通常是可用的。

> **注意：** Windows Server 上的 Docker 始终安装 Docker EE。有关如何在 Windows Server 2016 上安装 Docker EE 的信息，请参阅第三章。

您可能需要在以下命令前加上`sudo`。

1.  确保您拥有最新的软件包列表。

```
 $ apt-get update 
```

* 安装访问 Docker EE 仓库所需的软件包。

```
 $ apt-get install -y \
 	  apt-transport-https \
 	  curl \
 	  software-properties-common 
```

* 登录[Docker Store](https://store.docker.com/)并复制您独特的 Docker EE 仓库 URL。

将您的网络浏览器指向 store.docker.com。点击右上角的用户名，然后选择`My Content`。在您的活动 Docker EE 订阅下选择`Setup`。

![](img/figure16-2.png)

从`Resources`窗格下复制您的仓库 URL。

同时下载您的许可证。

![](img/figure16-3.png)

我们正在演示如何为 Ubuntu 设置仓库。但是，这个 Docker Store 页面包含了如何为其他 Linux 版本进行设置的说明。

+   将您独特的 Docker EE 仓库 URL 添加到环境变量中。

```
 $ DOCKER_EE_REPO=<paste-in-your-unique-ee-url> 
```

* 将官方 Docker GPG 密钥添加到所有密钥环中。

```
 $ curl -fsSL "`${``DOCKER_EE_REPO``}`/ubuntu/gpg" | sudo apt-key add - 
```

* 设置最新的稳定仓库。您可能需要用最新的稳定版本替换最后一行的值。

```
 $ add-apt-repository \
    "deb [arch=amd64] $DOCKER_EE_REPO/ubuntu \
    $(lsb_release -cs) \
    stable-17.06" 
```

* 运行另一个`apt-get update`，以从您新添加的 Docker EE 仓库中获取最新的软件包列表。

```
 $ apt-get update 
```

* 卸载先前版本的 Docker。

```
 $ apt-get remove docker docker-engine docker-ce docker.io 
```

* 安装 Docker EE

```
 $ apt-get install docker-ee -y 
```

* 检查安装是否成功。

```
$ docker --version
Docker version `17`.06.2-ee-6, build e75fdb8 
``````````` 

 ```That’s it, you’ve installed the Docker EE engine.

Now you can install Universal Control Plane.

#### Docker Universal Control Plane (UCP)

We’ll be referring to Docker Universal Control Plane as **UCP** for the rest of the chapter.

UCP is an enterprise-grade container-as-a-service platform with an Operations UI. It takes the Docker Engine, and adds all of the features enterprises love and require. Things like; *RBAC, policies, trust, a highly-available control plane,* and a *simple UI*. Under-the-covers, it’s a containerized microservices app that you download and run as a bunch of containers.

Architecturally, UCP builds on top of Docker EE in *Swarm mode*. As shown in Figure 16.4, the UCP control plane runs on Swarm managers, and apps are deployed on Swarm workers.

![Figure 16.4 High level UCP architecture](img/figure16-4.png)

Figure 16.4 High level UCP architecture

At the time of writing, UCP managers have to be Linux. Workers can be a mix of Windows and Linux.

##### Planning a UCP installation

When planning a UCP installation, it’s important to size and spec your cluster appropriately. We’ll look at some of things you should consider.

All nodes in the cluster should have their clocks in sync (e.g. NTP). If they don’t, problems can occur that are a pain to troubleshoot.

All nodes should have a static IP address and a stable DNS name.

By default, UCP managers don’t run user workloads. This is a recommended best practice, and you should enforce it for production environments — it allows managers to focus solely on control plane duties. It also makes troubleshooting easier.

You should always have an odd number of managers. This helps avoid split-brain conditions where managers fail or become partitioned from the rest of the cluster. The ideal number is 3, 5, or 7, with 3 or 5 usually being the best. Having more than 7 can cause issues with the back-end Raft and cluster reconciliation. If you don’t have enough nodes for 3 managers, 1 is better than 2!

If you’re implementing a backup schedule (which you should) and taking regular backups, you might want to deploy 5 managers. This is because Swarm and UCP backup operations require stopping Docker and UCP services. Having 5 managers can help maintain cluster resiliency during such operations.

Manager nodes should be spread across data center availability zones. The last thing you want, is a single availability zone failing and taking all of the UCP managers with it. However, it’s important to connect your managers via high-speed reliable networks. So if your data center availability zones are not connected by good networks, you might be better-off keeping all managers in a single availability zone. As a general rule, if you’re deploying on public cloud infrastructure, you should deploy your managers in availability zones within a single *region*. Spanning *regions* usually involves less-reliable high-latency networks.

You can have as many *worker nodes* as you want — they don’t participate in cluster Raft operations, so won’t impact control plane operations.

Planning the number and size of worker nodes requires an understanding of the apps you plan on running on the cluster. For example, knowing this will help you determine things like how many Windows vs Linux nodes you require. You will also need to know if any of your apps have special requirements and need specialised worker nodes — may be PCI workloads.

Also, although the Docker engine is lightweight and small, the containerized applications you run on your nodes might not be. With this in mind, it’s important to size nodes according to the CPU, RAM, network, and disk I/O requirements of your applications.

Making server sizing requirements isn’t something I like to do, as it’s entirely dependant on *your* workloads. However, the Docker website is currently suggesting the following **minimum** requirements for Docker UCP 2.2.4 on Linux:

*   UCP Manager nodes running DTR: 8GB of RAM with 3GB of disk space
*   UCP Worker nodes: 4GB of RAM with 3GB of free disk space

**Recommended** requirements are:

*   UCP Manager nodes running DTR: 8GB RAM, 4 vCPUs, and 100GB disk space
*   UCP Worker nodes: 4GB RAM 25-100GB of free disk space

Take this with a pinch of salt, and be sure to do your own sizing exercise.

One thing’s for sure — Windows images are **a lot bigger** than Linux images. So be sure to factor this into your sizing.

One final word on sizing requirements. Docker Swarm and Docker UCP make it extremely easy to add and remove managers and workers. New managers are automagically added to the HA control plane, and new workers are immediately available for workload scheduling. Similarly, removing managers and workers is simple. As long as you have multiple managers, you can remove a manager without impacting cluster operations. With worker nodes, you can drain them and remove them from a running cluster. This all makes UCP very forgiving when it comes to changing your managers and workers.

With these considerations in mind, we’re ready to install UCP.

##### Installing Docker UCP

In this section, we’ll walk through the process of installing Docker UCP on the first manager node in a new cluster.

1.  Run the following command from a Linux-based Docker EE node that you want to be the first manager in your UCP cluster.

    A few things to note about the command. The example installs UCP using the `docker/ucp:2.2.5` image, you will want to substitute your desired version. The `--host-address` is the address you will use to access the web UI. For example, if you’re installing in AWS and plan on accessing from your corporate network via the internet, you would enter the AWS public IP.

    The installation is interactive, so you’ll be prompted for further input to complete it.

    ```
     $ docker container run --rm -it --name ucp \
       -v /var/run/docker.sock:/var/run/docker.sock \
       docker/ucp:2.2.5 install \
       --host-address <node-ip-address> \
       --interactive 
    ```

`*   Configure credentials.

    You’ll be prompted to create a username and password for the UCP Admin account. This is a local account, and you should follow your corporate guidelines for choosing the username and password. Be sure you don’t forget it :-D

    *   Subject alternative names (SANs).

    The installer gives you the option to enter a list of alternative IP addresses and names that might be used to access UCP. These can be public and private IP addresses and DNS names, and will be added to the certificates.` 

 `A few things to note about the install.

UCP leverages Docker Swarm. This means UCP managers have to run on Swarm managers. If you install UCP on a node in *single-engine mode*, it will automatically be switched into *Swarm mode*.

The installer pulls all of the images for the various UCP services, and starts containers from them. The following listing shows some of them being pulled by the installer.

```
INFO[0008] Pulling required images... (this may take a while)
INFO[0008] Pulling docker/ucp-auth-store:2.2.5
INFO[0013] Pulling docker/ucp-hrm:2.2.5
INFO[0015] Pulling docker/ucp-metrics:2.2.5
INFO[0020] Pulling docker/ucp-swarm:2.2.5
INFO[0023] Pulling docker/ucp-auth:2.2.5
INFO[0026] Pulling docker/ucp-etcd:2.2.5
INFO[0028] Pulling docker/ucp-agent:2.2.5
INFO[0030] Pulling docker/ucp-cfssl:2.2.5
INFO[0032] Pulling docker/ucp-dsinfo:2.2.5
INFO[0080] Pulling docker/ucp-controller:2.2.5
INFO[0084] Pulling docker/ucp-proxy:2.2.5 
```

 `Some of the interesting ones include:

*   `ucp-agent` This is the main UCP agent. It gets deployed to all nodes in the cluster and is in charge of making sure the required UCP containers are up and running.
*   `ucp-etcd` The cluster’s persistent key-value store.
*   `ucp-auth` Shared authentication service (also used by DTR for single-sign-on).
*   `ucp-proxy` Controls access to the local Docker socket so that unauthenticated clients cannot make changes to the cluster.
*   `ucp-swarm` Provides compatibility with the underlying Swarm.

Finally, the installation creates a couple of root CA’s: one for internal cluster communications, and one for external access. They issue self-signed certs, which are fine for labs and testing, but not production.

To install UCP with certificates from a trusted CA, you will need a certificate bundle with the following three files:

*   `ca.pem` Certificate of the trusted CA (usually one of your internal corporate CA’s).
*   `cert.pem` UCP’s public certificate. This needs to contain all IP addresses and DNS names that the cluster will be accessed by — including any load-balancers that are fronting it.
*   `key.pem` UCP’s private key.

If you have these files, you need to mount them into a Docker volume called `ucp-controller-server-certs`, and use the `--external-ca` flag to specify the volume. You can also change the certificates from the `Admin Settings` page of the web UI after the installation.

The last thing the UCP installer outputs is the URL that you can access it from.

```
<Snip>
INFO[0049] Login to UCP at https://<IP or DNS>:443 
```

 `Point a web browser to that address and login. If you’re using self-signed certificates you’ll need to accept the browser warnings. You’ll also need to specify your license file, which can be downloaded from the `My Content` section of the Docker Store.

Once you’re logged in, you’ll be landed at the UCP Dashboard.

![](img/figure16-5.png)

At this point, you have a single-node UCP cluster.

You can add more manager and worker nodes from the `Add Nodes` link at the bottom of the Dashboard.

Figure 16.6 shows the Add Nodes screen. You can choose to add `managers` or `workers`, and it gives you the appropriate command to run on the nodes you want to add. The example shows the command to add a Linux worker node. Notice that the command is a `docker swarm` command.

![](img/figure16-6.png)

Adding a node will join it to the Swarm and configure the required UCP services on it. If you’re adding managers, it’s recommended to wait between each new addition. This gives Docker a chance to download and run the required UCP containers, as well as allow the cluster to register the new manager and achieve quorum.

Newly added managers are automatically configured into the highly-available (HA) Raft consensus group and granted access to the cluster store. Also, although external load-balancers aren’t generally considered core parts of UCP HA, they provide a stable DNS hostname that masks what’s going on behind the scenes — such as node failures.

You should configure external load-balancers for *TCP pass-through* on port 443, with a custom HTTPS health check for each UCP manager node at `https://<ucp_manager>/_ping`.

Now that you have a working UCP, you should look at the options that can be configured from the `Admin Settings` page.

![Figure 16.7 UCP Admin Settings](img/figure16-7.png)

Figure 16.7 UCP Admin Settings

The settings on this page make up the bulk of the configuration data that is backed as part of the UCP backup operation.

###### Controlling access to UCP

All access to UCP is controlled via the identity management sub-system. This means you need to authenticate with a valid UCP username and password before you can perform any actions on the cluster. This includes cluster administration, as well as deploying and managing services.

We’ve seen this already with UI — we had to log in with a username and password. But the same applies to the Docker CLI — you cannot run unauthenticated commands against UCP from the command line! This is because the local Docker socket on UCP cluster nodes is protected by the `ucp-proxy` service that will not accept unauthorized commands.

Let’s see it.

###### Client bundles

Any node running the Docker CLI is capable of deploying and managing workloads on a UCP cluster, **so long as it presents a valid certificate for a UCP user!**

In this section we’ll create a new UCP user, create and download a certificate bundle for that user, and configure a Docker client to use the certificates. Once we’re done, we’ll explain how it works.

1.  If you aren’t already, login to UCP as `admin`.
2.  Click `User Management` > `Users` and then create a new user.

    As we haven’t discussed roles and grants yet, make the user a Docker EE Admin.

3.  With the new user still selected, click the `Configure` drop-down box and choose `Client Bundle`.![](img/figure16-8.png)

4.  Click the `New Client Bundle +` link to generate and download a client bundle for the user.

    At this point, it’s important to note that client bundles are user-specific. The certificates downloaded will enable any properly configured Docker client to execute commands on the UCP cluster under the identity of the user that the bundle belongs to.

5.  Copy the bundle to the Docker client that you want to configure to manage UCP.
6.  Logon to the client node and perform all of the following commands from that node.
7.  Unzip the contents of the bundle.

    This example uses the Linux `unzip` package to unzip the contents of the bundle to the current directory. Substitute the name of the bundle to match the one in your environment.

    ```
     $ unzip ucp-bundle-nigelpoulton.zip
     Archive:  ucp-bundle-nigelpoulton.zip
     extracting: ca.pem
     extracting: cert.pem
     extracting: key.pem
     extracting: cert.pub
     extracting: env.sh
     extracting: env.ps1
     extracting: env.cmd 
    ```

     `As the output shows, the bundle contains the required `ca.pem`, `cert.pem`, and `key.pem` files. It also includes scripts that will configure the Docker client to use the certificates.` 
`*   Use the appropriate script to configure the Docker client. `env.sh` works on Linux and Mac, `env.ps1` and `env.cmd` work on Windows.

    You’ll probably need administrator/root privileges to run the scripts.

    The example works on Linux and Mac.

    ```
     $ eval "$(<env.sh)" 
    ```

     `At this point, the client node is fully configured.` `*   Test access.

    ```
     $ docker version

      <Snip>

     Server:
      Version:      ucp/2.2.5
      API version:  1.30 (minimum version 1.20)
      Go version:   go1.8.3
      Git commit:   42d28d140
      Built:        Wed Jan 17 04:44:14 UTC 2018
      OS/Arch:      linux/amd64
      Experimental: false 
    ```

     `Notice that the server portion of the output shows the version as `ucp/2.2.5`. This proves the Docker client is successfully talking to the daemon on a UCP node!``` 

 ``Under-the-hood, the script configures three environment variables:

*   `DOCKER_HOST`
*   `DOCKER_TLS_VERIFY`
*   `DOCKER_CERT_PATH`

DOCKER_HOST points the client to the remote Docker daemon on the UCP controller. An example might look like this `DOCKER_HOST=tcp://34.242.196.63:443`. As we can see, access via port 443.

DOCKER_TLS_VERIFY is set to 1, telling the client to use TLS verification in *client mode*.

DOCKER_CERT_PATH tells the Docker client where to find the certificate bundle.

The net result is all `docker` commands from the client will be signed by the user’s certificate and sent across the network to the remote UCP manager. This is shown in Figure 16.9.

![Figure16.9](img/figure16-9.png)

Figure16.9

Let’s switch tack and see how we backup and recover UCP.

###### Backing up UCP

First and foremost, high availability (HA) is not the same as a backup!

Consider the following example. You have a highly available UCP cluster with 5 managers nodes. All manager nodes are healthy and the control plane is replicating. A dissatisfied employee corrupts the cluster (or deletes all user accounts). This *corruption* is automatically replicated to all 5 manager nodes, rendering the cluster broken. There is no way that HA can help you in this situation. What you need, is a backup!

A UCP cluster is made from three major components that need backing up separately:

*   Swarm
*   UCP
*   Docker Trusted Registry (DTR)

We’ll walk you through the process of backing up Swarm and UCP, and we’ll show you how to back up DTR later in the chapter.

Although UCP sits on top of Swarm, they are separate components. Swarm holds all of the node membership, networks, volumes, and service definitions. UCP sits on top and maintains its own databases and volumes that hold things such as users, groups, grants, bundles, license files, certificates, and more.

Let’s see how to **backup Swarm**.

Swarm configuration and state is stored in `/var/lib/docker/swarm`. This includes Raft log keys, and it’s replicated to every manager node. A Swarm backup is a copy of all the files in this directory.

Because it’s replicated to every manager, you can perform the backup from any manager.

You need to stop Docker on the node that you want to perform the backup on. This means it’s probably not a good idea to perform the backup on the leader manager, as a leader election will be instigated. You should also perform the backup at a quiet time for the business — even though stopping Docker on a single manager node isn’t a problem in a multi-manager Swarm, it can increase the risk of the cluster losing quorum if another manager fails during the backup.

Before proceeding, you might want to create a couple of Swarm objects so that you can prove the backup and restore operation work. The example Swarm we’ll be backing up in these examples has an overlay network called `vantage-net` and a Swarm service called `vantage-svc`.

1.  Stop Docker on the Swarm manager node you are performing the backup from.

    This will stop all UCP containers on the node. If UCP is configured for HA, the other managers will make sure the control plane remains available.

    ```
     $ service docker stop 
    ```

`*   Backup the Swarm config.

    The example uses the Linux `tar` utility to perform the file copy. Feel free to use a different tool.

    ```
     $ tar -czvf swarm.bkp /var/lib/docker/swarm/
     tar: Removing leading `/' from member names
     /var/lib/docker/swarm/
     /var/lib/docker/swarm/docker-state.json
     /var/lib/docker/swarm/state.json
     <Snip> 
    ```

    `*   Verify that the backup file exists.

    ```
     $ ls -l
     -rw-r--r-- 1 root   root   450727 Jan 29 14:06 swarm.bkp 
    ```

     `You should rotate, and store the backup file off-site according to your corporate backup policies.` `*   Restart Docker.

    ```
     $ service docker restart 
    `````` 

 ```Now that Swarm is backed up, it’s time to **backup UCP**.

A few notes on backing up UCP before we start.

The UCP backup job runs as a container, so Docker needs to be running for the backup to work.

You can run the backup from any UCP manager node in the cluster, and you only need to run the operation on one node (UCP replicates its configuration to all manager nodes, so backing up from multiple nodes is not required).

Backing up UCP will stop all UCP containers on the manager that you’re executing the operation on. With this in mind, you should be running a highly available UCP cluster, and you should run the operation at a quiet time for the business.

Finally, user workloads running on the manager node will not be stopped. However, it is not recommended to run user workloads on UCP managers.

Let’s backup UCP.

Perform the following command on a UCP manager node. Docker will need to be running on the node.

```
$ docker container run --log-driver none --rm -i --name ucp `\`
  -v /var/run/docker.sock:/var/run/docker.sock `\`
  docker/ucp:2.2.5 backup --interactive `\`
  --passphrase `"Password123"` > ucp.bkp 
```

 `It’s a long command, so let’s step through it.

The first line is a standard `docker container run` command that tells Docker to run a container with no log driver, to remove it when the operation is complete, and to call it `ucp`. The second line mounts the *Docker socket* into the container so that the container has access to the Docker API to stop containers etc. The third line tells Docker to run a `backup --interactive` command inside of a container based on the `docker/ucp:2.2.5` image. The final line creates an encrypted file called `ucp.bkp` and protects it with a password.

A few points worth noting.

It’s a good idea to be specific about the version (tag) of the UCP image to use. This example specifies `docker/ucp:2.2.5`. One of the reasons for being specific, is that it’s recommended to run backup and restore operations with the same version of the image. If you don’t explicitly state which image to use, Docker will use the one tagged as `latest`, which might be different between the time you run the backup command and the time you run the restore.

You should always use the `--passphrase` flag to protect the contents of the backup, and you should definitely use a better password than the one in the example :-D

You should catalogue and make off-site copies of the backup file according to your corporate backup policies. You should also configure a backup schedule and job verification.

Now that Swarm and UCP are backed up, you can safely recover them in the event of disaster. Speaking of which….

###### Recovering UCP

We need to be clear about one thing before we get into the weeds of recovering UCP: Restoring from backup is a last resort, and should only be used when the cluster has been corrupted or all manager nodes have been lost!

You **do not need to recover from a backup if you’ve lost a single manager in an HA cluster**. In that case, you can easily add a new manager and it’ll join the cluster.

We’ll show how to recover Swarm from a backup, and then UCP.

Perform the following tasks from the Swarm/UCP manager node that you wish to recover.

1.  Stop Docker.

    ```
     $ service docker stop 
    ```

`*   Delete any existing Swarm configuration.

    ```
     $ rm -r /var/lib/docker/swarm 
    ```

    `*   Restore the Swarm configuration from backup.

    In this example, we’ll restore from a zipped `tar` file called `swarm.bkp`. Restoring to the root directory is required with this command as it will include the full path to the original files as part of the extract operation. This may be different in your environment.

    ```
     $ tar -zxvf swarm.bkp -C / 
    ```

    `*   Initialize a new Swarm cluster.

    Remember, you are not recovering a manager and adding it back to a working cluster. This operation is to recover a failed Swarm that has no surviving managers. The `--force-new-cluster` flag tells Docker to create a new cluster using the configuration stored in `/var/lib/docker/swarm` on the current node.

    ```
     $ docker swarm init --force-new-cluster
     Swarm initialized: current node (jhsg...3l9h) is now a manager. 
    ```

    `*   Check that the network and service were recovered as part of the operation.

    ```
     $ docker network ls
     NETWORK ID        NAME            DRIVER       SCOPE
     snkqjy0chtd5      vantage-net     overlay      swarm

     $ docker service ls
     ID              NAME          MODE         REPLICAS    IMAGE
     w9dimu8jfrze    vantage-svc   replicated   5/5         alpine:latest 
    ```

     `Congratulations. The Swarm is recovered.` `*   Add new manager and worker nodes to the Swarm, and take a fresh backup.`````

 ```With Swarm recovered, you can now **recover UCP.**

In this example, UCP was backed up to a file called `ucp.bkp` in the current directory. Despite the name of the backup file, it is a Linux tarball.

Run the following commands from the node that you want to recover UCP on. This can be the node that you just recovered Swarm on.

1.  Remove any existing, and potentially corrupted, UCP installations.

    ```
     $ docker container run --rm -it --name ucp \
       -v /var/run/docker.sock:/var/run/docker.sock \
       docker/ucp:2.2.5 uninstall-ucp --interactive

     INFO[0000] Your engine version 17.06.2-ee-6, build e75fdb8 is compatible
     INFO[0000] We're about to uninstall from this swarm cluster.
     Do you want to proceed with the uninstall? (y/n): y
     INFO[0000] Uninstalling UCP on each node...
     INFO[0009] UCP has been removed from this cluster successfully.
     INFO[0011] Removing UCP Services 
    ```

`*   Restore UCP from the backup.

    ```
     $ docker container run --rm -i --name ucp \
       -v /var/run/docker.sock:/var/run/docker.sock  \
       docker/ucp:2.2.5 restore --passphrase "Password123" < ucp.bkp

     INFO[0000] Your engine version 17.06.2-ee-6, build e75fdb8 is compatible
     <Snip>
     time="2018-01-30T10:16:29Z" level=info msg="Parsing backup file"
     time="2018-01-30T10:16:38Z" level=info msg="Deploying UCP Agent Service"
     time="2018-01-30T10:17:18Z" level=info msg="Cluster successfully restored. 
    ```

    `*   Log on to the UCP web UI and ensure that the user created earlier is still present (or any other UCP objects that previously existed in your environment).``

 ``Congrats. You now know how to backup and recover Docker Swarm and Docker UCP.

Let’s shift our attention to Docker Trusted Registry.

#### Docker Trusted Registry (DTR)

Docker Trusted Registry, which we’re going to refer to as DTR, is a secure, highly available on-premises Docker registry. If you know Docker Hub, think of DTR as a private Docker Hub that you can install on-premises and manage yourself.

In this section, we’ll show how to install it in an HA configuration, and how to back it up and perform recovery operations. We’ll show how DTR implements advanced features in the next chapter.

Let’s mention a few important things before getting your hands dirty with the installation.

If possible, you should run your DTR instances on dedicated nodes. You definitely shouldn’t run user workloads on your production DTR nodes.

As with UCP, you should run an odd number of DTR instances. 3 or 5 is best for fault tolerance. A recommended configuration for a production environment might be:

*   3 dedicated UCP managers
*   3 dedicated DTR instances
*   However many worker nodes your application requirements demand

Let’s install and configure a single DTR instance.

##### Install DTR

The next few steps will walk through the process of configuring the first DTR instance in a UCP cluster.

To follow along, you’ll need a UCP node that you will install DTR on, and a load balancer configured to listen on port 443 in TCP passthrough mode with a health check configured for `/health` on port 443\. Figure 16.10 shows a high-level diagram of what we’ll build.

Configuring a load balancer is beyond the scope of this book, but the diagram shows the important DTR-related configuration requirements.

![Figure 16.10 High level single-instance DTR config.](img/figure16-10.png)

Figure 16.10 High level single-instance DTR config.

1.  Log on to the UCP web UI and click `Admin` > `Admin Settings` > `Docker Trusted Registry`.
2.  Fill out the DTR configuration form.
    *   `DTR EXTERNAL URL:` Set this to the URL of your external load balancer.
    *   `UCP NODE:` Select the name of the node you wish to install DTR on.
    *   `Disable TLS Verification For UCP:` Check this box if you’re using self-signed certificates.
3.  Copy the long command at the bottom of the form.
4.  Paste the command into any UCP manager node.

    The command includes the `--ucp-node` flag telling UCP which node to perform the install on.

    The following is an example DTR install command that matches the configuration in Figure 16.10\. It assumes that you already have a load balancer configured at `dtr.mydns.com`

    ```
     $ docker run -it --rm docker/dtr install \
       --dtr-external-url dtr.mydns.com \
       --ucp-node dtr1  \
       --ucp-url https://34.252.195.122 \
       --ucp-username admin --ucp-insecure-tls 
    ```

     `You will need to provide the UCP admin password to complete the installation.` 
`*   Once the installation is complete, point your web browser to your load balancer. You will be automatically logged in to DTR.![Figure 16.11 DTR home page](img/figure16-11.png)

    Figure 16.11 DTR home page` 

 `DTR is ready to use. But it’s not configured for HA.

##### Configure DTR for high availability

Configuring DTR with multiple replicas for HA requires a shared storage backend. This can be NFS or object storage, and can be on-premises or in the public cloud. We’ll walk through the process of configuring DTR for HA using an Amazon S3 bucket as the shared backend.

1.  Log on to the DTR console and navigate to `Settings`.
2.  Select the `Storage` tab and configure the shared storage backend.

    Figure 16.12 shows DTR configured to use an AWS S3 bucket called `deep-dive-dtr` in the `eu-west-1` AWS availability zone. You will not be able to use this example.

    ![Figure 16.12 DTR Shared Storage configuration for AWS](img/figure16-12.png)

    Figure 16.12 DTR Shared Storage configuration for AWS

DTR is now configured with a shared storage backend and ready to have additional replicas.

1.  Run the following command from a manager node in the UCP cluster.

    ```
     $ docker run -it --rm \
       docker/dtr:2.4.1 join \
       --ucp-node dtr2 \
       --existing-replica-id 47f20fb864cf \
       --ucp-insecure-tls 
    ```

     `The `--ucp-node` flag tells the command which node to add the new DTR replica on. The `--insecure-tls` flag is required if you’re using self-signed certificates.

    You will need to substitute the version of the image and the replica ID. The replica ID was displayed as part of the output when you installed the initial replica.` 
`*   Enter the UCP URL and port, as well as admin credentials when prompted.`

 `When the join is complete, you will see some messages like the following.

```
INFO[0166] Join is complete
INFO[0166] Replica ID is set to: a6a628053157
INFO[0166] There are currently 2 replicas in your DTR cluster
INFO[0166] You have an even number of replicas which can impact availability
INFO[0166] It is recommended that you have 3, 5 or 7 replicas in your cluster 
```

 `Be sure to follow the advice and install additional replicas so that you operate an odd number.

You may need to update your load balancer configuration so that it balances traffic across the new replicas.

DTR is now configured for HA. This means you can lose a replica without impacting the availability of the service. Figure 16.13 shows an HA DTR configuration.

![Figure 16.13 DTR HA](img/figure16-13.png)

Figure 16.13 DTR HA

Notice that the external load balancer is sending traffic to all three DTR replicas, as well as performing health checks on all three. All three DTR replicas are also sharing the same external shared storage backend.

In the diagram, the load balancer and the shared storage backend are 3rd party products and depicted as singletons (not HA). In order to keep the entire environment as highly available as possible, you should ensure they have native HA, and that you back up their contents and configurations as well (e.g. make sure the load balancer and storage systems are natively HA, and perform backups of them).

##### Backup DTR

As with UCP, DTR has a native `backup` command that is part of the Docker image that was used to install the DTR. This native backup command will backup the DTR configuration that is stored in a set of named volumes, and includes:

*   DTR configuration
*   Repository metadata
*   Notary data
*   Certificates

**Images are not backed up as part of a native DTR backup**. It is expected that images are stored in a highly available storage backend that has its own independent backup schedule using non-Docker tools.

Run the following command from a UCP manager node to perform a DTR backup.

```
$ `read` -sp `'ucp password: '` UCP_PASSWORD`;` `\`
    docker run --log-driver none -i --rm `\`
    --env `UCP_PASSWORD``=``$UCP_PASSWORD` `\`
    docker/dtr:2.4.1 backup `\`
    --ucp-insecure-tls `\`
    --ucp-username admin `\`
    > ucp.bkp 
```

 `Let’s explain what the command is doing.

The `read` command will prompt you to enter the password for the UCP admin account, and will store it in a variable called `UCP_PASSWORD`. The second line tells Docker to start a new temporary container for the operation. The third line makes the UCP password available inside the container as an environment variable. The fourth line issues the backup command. The fifth line makes it work with self-signed certificates. The sixth line sets the UCP username to “admin”. The last line directs the backup to a file in the current directory called `ucp.bkp`.

You will be prompted to enter the UCP URL as well as a replica ID. You can specify these as part of the backup command, I just didn’t want to explain a single command that was 9 lines long!

When the backup is finished, you will have a file called `ucp.bkp` in your working directory. This should be picked up by your corporate backup tool and managed in-line with your existing corporate backup policies.

##### Recover DTR from backups

Restoring DTR from backups should be a last resort, and only attempted when the majority of replicas are down and the cluster cannot be recovered any other way. If you have lost a single replica and the majority are still up, you should add a new replica using the `dtr join` command.

If you are sure you have to restore from backup, the workflow is like this:

1.  Stop and delete DTR on the node (might already be stopped)
2.  Restore images to the shared storage backend (might not be required)
3.  Restore DTR

Run the following commands from the node that you want to restore DTR to. This node will obviously need to be a member of the same UCP cluster that the DTR is a member of. You should also use the same version of the `docker/dtr` image that was used to create the backup.

1.  Stop and delete DTR.

    ```
     $ docker run -it --rm \
       docker/dtr:2.4.1 destroy \
       --ucp-insecure-tls

     INFO[0000] Beginning Docker Trusted Registry replica destroy
     ucp-url (The UCP URL including domain and port): https://34.252.195.122:443
     ucp-username (The UCP administrator username): admin
     ucp-password:
     INFO[0020] Validating UCP cert
     INFO[0020] Connecting to UCP
     INFO[0021] Searching containers in UCP for DTR replicas
     INFO[0023] This cluster contains the replicas: 47f20fb864cf a6a628053157
     Choose a replica to destroy [47f20fb864cf]:
     INFO[0030] Force removing replica
     INFO[0030] Stopping containers
     INFO[0035] Removing containers
     INFO[0045] Removing volumes
     INFO[0047] Replica removed. 
    ```

     `You’ll be prompted to enter the UCP URL, admin credentials, and replica ID that you want to delete.

    If you have multiple replicas, you can run the command multiple times to remove them all.` 
`*   If the images were lost from the shared backend, you will need to recover them. This step is beyond the scope of the book as it can be specific to your shared storage backend.*   Restore DTR with the following command.

    You will need to substitute the values on lines 5 and 6 with the values from your environment. Unfortunately the `restore` command cannot be ran interactively, so you cannot be prompted for values once the `restore` has started.

    ```
     `$` `read` `-sp` `'ucp password: '` `UCP_PASSWORD``;` `\`
     `docker` `run` `-i` `--rm` `\`
     `--env` `UCP_PASSWORD``=$``UCP_PASSWORD` `\`
     `docker``/``dtr``:``2``.``4``.``1` `restore` `\`
     `--ucp-url` `<``ENTER_YOUR_ucp-url``>` `\`
     `--ucp-node` `<``ENTER_DTR_NODE_hostname``>` `\`
     `--ucp-insecure-tls` `\`
     `--ucp-username` `admin` `\`
     `<` `ucp``.``bkp` 
    ```` 

 ``DTR is now recovered.

Congratulations. You now know how to backup and recover; Swarm, UCP, and DTR.

Time for one final thing before wrapping up the chapter — network ports!

UCP managers, workers, and DTR nodes need to be able to communicate over the network. Figure 16.14 summarizes the port requirements.

![Figure 16.14 UCP cluster network port requirements](img/figure16-14.png)

Figure 16.14 UCP cluster network port requirements

### Chapter Summary

Docker Enterprise Edition (EE) is a suite of products that form an “*enterprise friendly*” container-as-a-service platform. It comprises a hardened Docker Engine, an Operations UI, and a secure registry. All of which can be deployed on-premises and managed by the customer. It’s even bundled with a support contract.

Docker Universal Control Plane (UCP) provides a simple-to-use web UI focussed at traditional enterprise Ops teams. It supports native high availability (HA) and has tools to perform backup and restore operations. Once up and running, it provides a whole suite of enterprise-grade features that we’ll discuss in the next chapter.

Docker Trusted Registry (DTR) sits on top of UCP and provides a highly available secure registry. Like UCP, this can be deployed on-premises within the safety of the corporate *“firewall”*, and provides native tools for backup and recovery.```````````````````````

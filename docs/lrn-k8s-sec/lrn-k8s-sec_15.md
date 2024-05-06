# *第十二章*：分析和检测加密挖矿攻击

随着区块链和加密货币的日益普及，加密挖矿攻击变得越来越引人注目。加密货币是通过在区块链上利用计算资源进行去中心化交易的交易费而获得的。使用计算资源验证交易以赚取加密货币的过程称为加密挖矿，由一个名为加密挖矿软件的软件进行。安全研究人员发现与各种加密挖矿二进制文件相关的黑客事件在受害者的基础设施内运行。Kubernetes 集群的默认开放性以及用于挖矿所需的大量计算能力的可用性使得 Kubernetes 集群成为加密挖矿攻击的完美目标。Kubernetes 集群的复杂性也使得加密挖矿活动难以检测。

由于我们已经介绍了不同的 Kubernetes 内置安全机制和开源工具来保护 Kubernetes 集群，现在我们将看看如何在具体场景中使用它们。在本章中，我们将首先分析几种已知的加密挖矿攻击，然后我们将讨论如何使用开源工具检测加密挖矿攻击的检测机制。最后但同样重要的是，我们将回顾我们在之前章节中讨论的主题，并看看它们应该如何应用来保护我们的环境免受一般攻击。

本章将涵盖以下主题：

+   分析加密挖矿攻击

+   检测挖矿攻击

+   防御攻击

# 分析加密挖矿攻击

在本节中，我们将首先简要介绍加密挖矿攻击，然后分析一些公开披露的加密挖矿攻击。我们希望您了解加密挖矿攻击模式以及使攻击可能的缺陷。

## 加密挖矿攻击简介

区块链构成了加密货币的基础。简而言之，区块链是由表示为区块的数字资产链组成的。这些区块包含有关交易的信息以及谁参与了交易的数字签名。每种加密货币都与一个区块链相关联。验证交易记录的过程称为挖矿。挖矿将历史记录添加到区块链中，以确保区块在未来无法修改。挖矿旨在消耗大量资源，以确保区块链的去中心化属性。通过成功挖矿区块，矿工可以获得与交易相关的交易费。因此，如果你有一台笔记本电脑或个人电脑，你也可以用它来挖矿；但很可能你需要一些专用的 GPU 或专门的硬件，比如**现场可编程门阵列**（**FPGA**）和**专用集成电路**（**ASIC**）来做好挖矿工作。Kubernetes 集群中的资源可用性使它们成为攻击者赚取加密货币的理想目标。

加密挖矿攻击就像在 Wi-Fi 上免费搭车一样。就像你的网络带宽会被免费搭车者分享一样，你的 CPU 或计算资源的一部分（或大部分）将在没有你的同意的情况下被挖矿进程占用。影响也是类似的。如果 Wi-Fi 上的免费搭车者正在使用你的 Wi-Fi 网络通过 BitTorrent 下载电影，你在观看 Netflix 时可能会有不好的体验。当有挖矿进程运行时，同一节点中运行的其他应用程序也会受到严重影响，因为挖矿进程可能会大部分时间占用 CPU。

加密挖矿攻击已经成为黑客最吸引人的攻击之一，因为这几乎是一种确保能够从成功入侵中获益的方式。小偷只来偷或破坏。如果破坏不是入侵的目的，加密挖矿攻击可能是黑客的主要选择之一。

黑客发动加密挖矿攻击的至少两种方式已经被报道。一种是通过应用程序漏洞，比如跨站脚本，SQL 注入，远程代码执行等，使黑客能够访问系统，然后下载并执行挖矿程序。另一种方式是通过恶意容器镜像。当从包含挖矿程序的镜像创建容器时，挖矿过程就会开始。

尽管互联网上有不同类型的加密挖矿二进制文件，但总的来说，矿业过程是计算密集型的，占用大量 CPU 周期。矿业过程有时会加入矿业池，以便以合作的方式进行挖矿。

接下来，让我们看看发生在现实世界中的一些加密挖矿攻击。我们将讨论使攻击可能的漏洞，并研究攻击模式。

## 特斯拉的 Kubernetes 集群上的加密挖矿攻击

2018 年，特斯拉的 Kubernetes 集群遭受了一次加密挖矿攻击，并由 RedLock 报告。尽管这次攻击发生了相当长时间，但我们至少可以从中学到两件事情——使攻击可能的漏洞和攻击模式。

### 漏洞

黑客渗透了没有密码保护的 Kubernetes 仪表板。从仪表板上，黑客获得了一些重要的秘密来访问 Amazon S3 存储桶。

### 攻击模式

黑客们做了相当不错的工作，隐藏了他们的足迹，以避免被发现。以下是一些值得一提的模式：

+   矿业过程没有占用太多 CPU 周期，因此 pod 的 CPU 使用率并不太高。

+   与大多数加密挖矿案例不同，矿业过程没有加入任何知名的矿业池。相反，它有自己的矿业服务器，位于 Cloudflare 后面，这是一个内容交付网络（CDN）服务。

+   矿业过程与矿业服务器之间的通信是加密的。

通过前面的操作，黑客故意试图隐藏加密挖矿模式，以便逃避检测。

## Graboid——一次加密蠕虫攻击

这次加密蠕虫攻击是由 Palo Alto Network Unit42 研究团队在 2019 年底发现的。尽管这次攻击并不是针对 Kubernetes 集群，但它是针对 Docker 守护程序的，这是 Kubernetes 集群中的基石之一。在攻击的一个步骤中，工具包从 Docker Hub 下载包含加密挖矿二进制文件的镜像并启动。这一步也可以应用于 Kubernetes 集群。

### 漏洞

Docker 引擎暴露在互联网上，而且没有进行身份验证和授权配置。攻击者可以轻松地完全控制 Docker 引擎。

### 攻击模式

一旦黑客控制了 Docker 引擎，他们就开始下载一个恶意镜像并启动一个容器。以下是关于恶意容器值得一提的一些模式：

+   恶意容器联系了命令和控制服务器，以下载一些恶意脚本。

+   恶意容器包含了一个 Docker 客户端二进制文件，用于控制其他不安全的 Docker 引擎。

+   恶意容器通过 Docker 客户端向其他不安全的 Docker 引擎发出命令，以下载和启动另一个包含加密挖矿二进制文件的镜像。

根据 Shodan 的数据，超过 2000 个 Docker 引擎暴露在互联网上。前述步骤被重复执行，以便加密挖矿蠕虫传播。

## 吸取的教训

回顾一下我们讨论过的两种已知的加密挖矿攻击，配置错误是使黑客轻松入侵的主要问题之一。加密挖矿具有一些典型的模式，例如，挖矿过程将与挖矿池通信，而挖矿过程通常会占用大量的 CPU 周期。然而，黑客可能会故意伪装他们的挖矿行为以逃避检测。一旦黑客进入 pod，他们可以开始联系命令和控制服务器来下载和执行挖矿二进制文件；另一方面，他们也可以开始侦察。如果您的 Kubernetes 集群中的安全域没有得到适当配置，他们很容易进行横向移动。接下来，让我们使用我们在之前章节介绍的开源工具来检测 Kubernetes 集群中典型的加密挖矿活动。

# 检测加密挖矿攻击

在这一部分，我们将讨论如何使用我们在前几章介绍的一些开源工具来检测 Kubernetes 集群中的加密挖矿活动。我们基于已知的加密挖矿模式来检测加密挖矿活动：高 CPU 使用率，与挖矿池的通信，矿工的执行命令行以及二进制签名。请注意，每个单独的措施都有其自身的局限性。将它们结合起来可以提高检测的效率。然而，仍然存在一些高级的加密挖矿攻击，比如攻击特斯拉的那种。因此，有必要与安全团队合作，为您的 Kubernetes 集群应用全面的检测策略，以覆盖各种入侵。

为了演示每个工具检测加密挖矿，我们模拟一个受害者`nginx` pod：

```
$ kubectl get pods -n insecure-nginx
NAME                              READY   STATUS    RESTARTS   AGE
insecure-nginx-8455b6d49c-z6wb9   1/1     Running   0          163m
```

在`nginx` pod 内部，有一个矿工二进制文件位于`/tmp`目录中：

```
root@insecure-nginx-8455b6d49c-z6wb9:/# ls /tmp
minerd2  perg
```

`minerd2`是挖矿二进制文件。我们可以假设`minerd2`要么被种子化在镜像中，要么从命令和控制服务器下载。首先，让我们看看监控 CPU 使用率如何帮助检测加密挖矿活动。

注意

不建议在生产服务器上运行加密挖矿二进制文件。这仅供教育目的。 

## 监控 CPU 利用率

正如我们在*第十章*中讨论的那样，*Kubernetes 集群的实时监控和资源管理*，资源管理和资源监控对于维护服务的可用性至关重要。加密挖矿通常占用大量 CPU 周期，导致容器或 pod 的 CPU 使用率显着提高。让我们通过比较加密挖矿发生前后`nginx` pod 的 CPU 使用情况来看一个例子：

![图 12.1 - 挖矿前 nginx pod 的 CPU 使用情况在 Grafana 指标中](img/B15566_12_01.jpg)

图 12.1 - 挖矿前 nginx pod 的 CPU 使用情况在 Grafana 指标中

前面的截图显示了由 Prometheus 和 Grafana 监控的`insecure-nginx` pod 的 CPU 使用情况。一般来说，最大的 CPU 使用率小于`0.1`。当执行加密挖矿二进制文件时，你会发现 CPU 使用率急剧上升：

![图 12.2 - 挖矿后 nginx pod 的 CPU 使用情况](img/B15566_12_02.jpg)

图 12.2 - 挖矿后 nginx pod 的 CPU 使用情况

CPU 使用率从平均率`0.07`上升到约`2.4`。无论在幕后发生了什么，这样巨大的 CPU 使用率上升都应立即引起您的注意。很明显，即使有这样的 CPU 激增，也不意味着 pod 内运行着加密挖矿二进制文件。CPU 激增也可能是由其他原因引起的。

另一方面，如果黑客故意限制加密挖矿攻击的进展，就像对特斯拉的攻击一样，CPU 可能只会有一点点上升，很难注意到。接下来，让我们看看 Falco 如何帮助检测加密挖矿活动。

## 检测到矿池的网络流量

典型的加密挖矿进程行为是挖矿进程与同一挖矿池内的其他挖矿进程协作，以便高效地进行挖矿。挖矿进程在挖矿期间与挖矿池服务器进行通信。

在 Falco 的默认规则中，有一个规则用于检测对已知矿工池的出站连接。让我们更仔细地看看这个规则。首先，有一个用于挖矿端口和挖矿域的预定义列表([`github.com/falcosecurity/falco/blob/master/rules/falco_rules.yaml#L2590`](https://github.com/falcosecurity/falco/blob/master/rules/falco_rules.yaml#L2590))：

```
- list: miner_ports
  items: [
        25, 3333, 3334, 3335, 3336, 3357, 4444,
        5555, 5556, 5588, 5730, 6099, 6666, 7777,
        7778, 8000, 8001, 8008, 8080, 8118, 8333,
        8888, 8899, 9332, 9999, 14433, 14444,
        45560, 45700
    ]
- list: miner_domains
  items: [
      "Asia1.ethpool.org","ca.minexmr.com", "monero.crypto-pool.fr",
      ...
      "xmr-jp1.nanopool.org","xmr-us-east1.nanopool.org",
      "xmr-us-west1.nanopool.org","xmr.crypto-pool.fr",
      "xmr.pool.minergate.com"
      ]
```

然后，有一个预定义的网络连接宏用于前述矿工端口和矿工域：

```
- macro: minerpool_other
  condition: (fd.sport in (miner_ports) and fd.sip.name in (miner_domains))
```

除了`minerpool_other`宏之外，还有两个分别用于 HTTP 和 HTTPS 连接的其他宏—`minerpool_http`和`minerpool_https`—它们都结合起来得到主要的检测逻辑：

```
- macro: net_miner_pool
  condition: (evt.type in (sendto, sendmsg) and evt.dir=< and (fd.net != "127.0.0.0/8" and not fd.snet in (rfc_1918_addresses)) and ((minerpool_http) or (minerpool_https) or (minerpool_other)))
```

然后，`net_miner_pool`宏由`检测出站连接到常见矿工池端口`规则使用，以检测出站连接到矿工域：

```
# The rule is disabled by default.
# Note: Falco will send DNS requests to resolve miner pool domains which may trigger alerts in your environment.
- rule: Detect outbound connections to common miner pool ports
  desc: Miners typically connect to miner pools on common ports.
  condition: net_miner_pool and not trusted_images_query_miner_domain_dns
  enabled: true
  output: Outbound connection to IP/Port flagged by cryptoioc.ch (command=%proc.cmdline port=%fd.rport ip=%fd.rip container=%container.info image=%container.image.repository)
  priority: CRITICAL
  tags: [network, mitre_execution]
```

如果有一个正在运行并与列表中定义的矿工域进行通信的加密挖矿进程，警报将被触发，如下所示：

```
19:46:37.939287649: Critical Outbound connection to IP/Port flagged by cryptoioc.ch (command=minerd2 -a cryptonight -o stratum+tcp://monero.crypto-pool.fr:3333 -u 49TfoHGd6apXxNQTSHrMBq891vH6JiHmZHbz5Vx36nLRbz6WgcJunTtgcxno G6snKFeGhAJB5LjyAEnvhBgCs5MtEgML3LU -p x port=37110 ip=100.97.244.198 container=k8s.ns=insecure-nginx k8s.pod=insecure-nginx-8455b6d49c-z6wb9 container=07dce07d5100 image=kaizheh/victim) k8s.ns=insecure-nginx k8s.pod=insecure-nginx-8455b6d49c-z6wb9 container=07dce07d5100 k8s.ns=insecure-nginx k8s.pod=insecure-nginx-8455b6d49c-z6wb9 container=07dce07d5100
```

`检测出站连接到常见矿工池端口`规则很简单。如果这个规则生成了一个警报，你应该把它作为高优先级处理。规则的限制也很明显；您将不得不保持挖矿域和挖矿端口的更新。如果有新的挖矿域可用或者使用了新的挖矿服务器端口，并且它们没有添加到 Falco 列表中，那么规则将无法检测到加密挖矿活动。请注意，该规则默认情况下是禁用的。由于 Falco 需要发送 DNS 请求来解析矿工池域名，这些 DNS 请求将被一些云提供商警报。一个副作用是，像 Cilium 的 Hubble 这样的开源工具可以帮助监控网络流量。

另一种方法是使用白名单方法。如果您知道微服务的出站连接中的目标端口或 IP 块，您可以创建 Falco 规则来警报不在白名单上的任何出站连接的目标 IP 或端口。以下是一个例子：

```
- list: trusted_server_addresses
  items: [...]
- list: trusted_server_ports
  items: [...]
- rule: Detect anomalous outbound connections 
  desc: Detect anomalous outbound connections
  condition: (evt.type in (sendto, sendmsg) and container and evt.dir=< and (fd.net != "127.0.0.0/8" and not fd.snet in (trusted_server_addresses) or not fd.sport in (trusted_server_ports))) 
  output: Outbound connection to anomalous IP/Port(command=%proc.cmdline port=%fd.rport ip=%fd.rip container=%container.info image=%container.image.repository)
  priority: CRITICAL
```

上述规则将警报任何对`trusted_server_ports`或`trusted_server_addresses`之外的 IP 地址或端口的出站连接。鉴于攻击发生在特斯拉，Falco 将警报存在异常连接，即使 IP 地址看起来正常。接下来，让我们看另一个 Falco 规则，根据命令行中的模式来检测潜在的加密挖矿活动。

## 检测已启动的加密挖矿进程

Stratum 挖矿协议是与挖矿服务器进行通信的挖矿过程中最常见的协议。一些挖矿二进制文件允许用户在执行时指定与挖矿池服务器通信的协议。

在 Falco 的默认规则中，有一个规则是基于命令行中的关键字来检测加密二进制文件的执行：

```
- rule: Detect crypto miners using the Stratum protocol
  desc: Miners typically specify the mining pool to connect to with a URI that begins with 'stratum+tcp'
  condition: spawned_process and proc.cmdline contains "stratum+tcp"
  output: Possible miner running (command=%proc.cmdline container=%container.info image=%container.image.repository)
  priority: CRITICAL
  tags: [process, mitre_execution]
```

如果 Falco 检测到任何使用`stratum+tcp`启动的进程并且在进程的命令行中指定了，那么`检测使用 Stratum 协议的加密矿工`规则将引发警报。输出如下：

```
19:46:37.779784798: Critical Possible miner running (command=minerd2 -a cryptonight -o stratum+tcp://monero.crypto-pool.fr:3333 -u 49TfoHGd6apXxNQTSHrMBq891vH6JiHmZHbz5Vx36 nLRbz6WgcJunTtgcxnoG6snKFeGhAJB5LjyAEnvhBgCs5MtEgML3LU -p x container=k8s.ns=insecure-nginx k8s.pod=insecure-nginx-8455b6d49c-z6wb9 container=07dce07d5100 image=kaizheh/victim) k8s.ns=insecure-nginx k8s.pod=insecure-nginx-8455b6d49c-z6wb9 container=07dce07d5100 k8s.ns=insecure-nginx k8s.pod=insecure-nginx-8455b6d49c-z6wb9 container=07dce07d5100
```

执行的`minerd2 -a cryptonight -o stratum+tcp://monero.crypto-pool.fr:3333 -u 49TfoHGd6apXxNQTSHrMBq891vH6JiHmZHbz5Vx36nLRbz6Wgc JunTtgcxnoG6snKFeGhAJB5LjyAEnvhBgCs5MtEgML3LU -p x`命令行包含了`stratum+tcp`关键字。这就是为什么会触发警报。

与其他基于名称的检测规则一样，该规则的限制是显而易见的。如果加密二进制执行文件不包含`stratum+tcp`，则该规则将不会被触发。

上述规则使用了黑名单方法。另一种方法是使用白名单方法，如果您知道将在微服务中运行的进程。您可以定义一个 Falco 规则，当启动任何不在信任列表上的进程时引发警报。以下是一个示例：

```
- list: trusted_nginx_processes
  items: ["nginx"]
- rule: Detect Anomalous Process Launched in Nginx Container
  desc: Anomalous process launched inside container.
  condition: spawned_process and container and not proc.name in (trusted_nginx_processes) and image.repository.name="nginx"
  output: Anomalous process running in Nginx container (command=%proc.cmdline container=%container.info image=%container.image.repository)
  priority: CRITICAL
  tags: [process]
```

上述规则将警报任何在`nginx`容器中启动的异常进程，其中包括加密挖矿进程。最后，让我们看看图像扫描工具如何通过与恶意软件源集成来帮助检测加密挖矿二进制文件的存在。

## 检查二进制签名

加密挖矿二进制文件有时可以被识别为恶意软件。与传统的反病毒软件一样，我们也可以检查运行中的二进制文件的哈希值与恶意软件源的匹配情况。借助图像扫描工具，比如 Anchore，我们可以获取文件的哈希值：

```
root@anchore-cli:/# anchore-cli --json image content kaizheh/victim:nginx files | jq '.content | .[] | select(.filename=="/tmp/minerd2")'
{
  "filename": "/tmp/minerd2",
  "gid": 0,
  "linkdest": null,
  "mode": "00755",
  "sha256": "e86db6abf96f5851ee476eeb8c847cd73aebd0bd903827a362 c07389d71bc728",
  "size": 183048,
  "type": "file",
  "uid": 0
}
```

`/tmp/minerd2`文件的哈希值为`e86db6abf96f5851ee476eeb8c847cd73aebd0bd903827a362c07389d71bc728`。然后，我们可以将哈希值与 VirusTotal 进行比对，VirusTotal 提供恶意软件信息源服务：

```
$ curl -H "Content-Type: application/json" "https://www.virustotal.com/vtapi/v2/file/report?apikey=$VIRUS_FEEDS_API_KEY&resource=e86db6abf96f5851ee476eeb8c847cd73aebd0bd903827a 362c07389d71bc728" | jq .
```

`$VIRUS_FEEDS_API_KEY`是您访问 VirusTotal API 服务的 API 密钥，然后提供以下报告：

```
{
  "scans": {
    "Fortinet": {
      "detected": true,
      "version": "6.2.142.0",
      "result": "Riskware/CoinMiner",
      "update": "20200413"
    },
    ...
    "Antiy-AVL": {
      "detected": true,
      "version": "3.0.0.1",
      "result": "RiskWare[RiskTool]/Linux.BitCoinMiner.a",
      "update": "20200413"
    },
  },
  ...
  "resource": "e86db6abf96f5851ee476eeb8c847cd73aebd0bd903827a362c07389d71bc 728",
  "scan_date": "2020-04-13 18:22:56",
  "total": 60,
  "positives": 25,
  "sha256": "e86db6abf96f5851ee476eeb8c847cd73aebd0bd903827a362c07389d71bc 728",
 }
```

VirusTotal 报告显示，`/tmp/minerd2`已被 25 个不同的信息源报告为恶意软件，如 Fortinet 和 Antiy AVL。通过在 CI/CD 流水线中集成图像扫描工具和恶意软件信息源服务，您可以帮助在开发生命周期的早期阶段检测恶意软件。然而，这种单一方法的缺点是，如果挖矿二进制文件从命令和控制服务器下载到运行的 Pod 中，您将错过加密挖矿攻击。另一个限制是，如果信息源服务器没有关于加密二进制文件的任何信息，您肯定会错过它。

我们已经讨论了四种不同的方法来检测加密挖矿攻击。每种方法都有其自身的优点和局限性；将一些这些方法结合起来以提高其检测能力和检测效果将是理想的。

接下来，让我们回顾一下我们在本书中讨论的内容，并全面运用这些知识来预防一般性的攻击。

# 防御攻击

在前一节中，我们讨论了几种检测加密挖矿活动的方法。在本节中，我们将讨论通过保护 Kubernetes 集群来防御攻击。因此，这不仅涉及防御特定攻击，还涉及防御各种攻击。四个主要的防御领域是 Kubernetes 集群供应、构建、部署和运行时。首先，让我们谈谈保护 Kubernetes 集群供应。

## 保护 Kubernetes 集群供应

有多种方法可以配置 Kubernetes 集群，比如`kops`和`kubeadm`。无论您使用哪种工具来配置集群，每个 Kubernetes 组件都需要进行安全配置。使用`kube-bench`来对您的 Kubernetes 集群进行基准测试，并改进安全配置。确保启用了 RBAC，禁用了`--anonymous-auth`标志，网络连接进行了加密等等。以下是我们在*第六章*中涵盖的关键领域，*保护集群组件*，以及*第七章*，*身份验证、授权和准入控制*：

+   为 Kubernetes 控制平面、`kubelet`等正确配置身份验证和授权

+   保护 Kubernetes 组件之间的通信，例如`kube-apiserver`、`kubelet`、`kube-apiserver`和`etcd`之间的通信

+   为`etcd`启用静态数据加密

+   确保不启动不必要的组件，比如仪表板

+   确保所有必要的准入控制器都已启用，而已弃用的控制器已禁用

通过安全配置 Kubernetes 集群，可以减少黑客轻易入侵您的 Kubernetes 集群的机会，就像特斯拉的集群一样（其中仪表板不需要身份验证）。接下来，让我们谈谈如何保护构建。

## 保护构建

保护 Kubernetes 集群还包括保护微服务。保护微服务必须从 CI/CD 流水线的开始进行。以下是一些关键的对策，如*第八章*中讨论的，*保护 Kubernetes Pod*，以及*第九章*，*DevOps 流水线中的图像扫描*，以在构建阶段保护微服务：

+   妥善处理图像扫描工具发现的微服务漏洞，以减少通过利用应用程序漏洞成功入侵的可能性。

+   对 Dockerfile 进行基准测试，以改进镜像的安全配置。确保镜像中没有存储敏感数据，所有依赖包都已更新等等。

+   扫描镜像中的可执行文件，确保没有恶意软件植入镜像。

+   为工作负载正确配置 Kubernetes 安全上下文。遵循最小特权原则，限制对系统资源的访问，比如使用主机级别的命名空间、主机路径等，并移除不必要的 Linux 能力，只授予必需的能力。

+   不要启用自动挂载服务账户。如果工作负载不需要服务账户，就不要为其创建服务账户。

+   遵循最小特权原则，尝试了解工作负载正在执行的任务，并只授予服务账户所需的特权。

+   遵循最小特权原则，尝试估计工作负载的资源使用情况，并为工作负载应用适当的资源请求和限制。

当然，保护构建也可以扩展到保护整个 CI/CD 流水线，比如源代码管理和 CI/CD 组件。然而，这超出了本书的范围。我们只会建议我们认为最相关的保护 Kubernetes 集群的选项。接下来，让我们谈谈保护部署。

## 保护部署

我们已经在 Kubernetes 集群中的*第七章*、*认证、授权和准入控制*，以及*第八章*、*保护 Kubernetes Pods*中讨论了不同类型的准入控制器，以及正确使用它们的必要性，例如一个镜像扫描准入控制器的示例（*第九章*、*DevOps 流水线中的镜像扫描*）。使用准入控制器和其他内置机制作为工作负载的重要安全门卫。以下是一些关键的对策：

+   为命名空间和工作负载应用网络策略。这可以是限制对工作负载的访问（入站网络策略），也可以是实施最小特权原则（出站网络策略）。当给定一个工作负载时，如果你知道出站连接的目标 IP 块，你应该为该工作负载创建一个网络策略来限制其出站连接。出站网络策略应该阻止任何超出白名单 IP 块的目的地流量，比如从命令和控制服务器下载加密挖矿二进制文件。

+   使用**Open Policy Agent**（**OPA**）来确保只有来自受信任的镜像仓库的镜像被允许在集群中运行。有了这个策略，OPA 应该阻止来自不受信任来源的镜像运行。例如，可能存在包含加密挖矿二进制文件的恶意镜像在 Docker Hub 中，因此您不应该将 Docker Hub 视为受信任的镜像仓库。

+   使用镜像扫描准入控制器来确保只有符合扫描策略的镜像被允许在集群中运行。我们已经在[*第九章*]（B15566_09_Final_ASB_ePub.xhtml#_idTextAnchor277）中谈到了这一点，*DevOps 流水线中的镜像扫描*。可能会发现新的漏洞，并且在部署工作负载时漏洞数据库将会更新。在部署之前进行扫描是必要的。

+   使用 OPA 或 Pod 安全策略来确保具有受限 Linux 功能和对主机级命名空间、主机路径等受限访问权限的工作负载。

+   最好在工作节点上启用 AppArmor，并为部署的每个镜像应用一个 AppArmor 配置文件。当工作负载部署时，会限制 AppArmor 配置文件，尽管实际的保护是在运行时发生的。一个很好的用例是构建一个 AppArmor 配置文件，以允许白名单内的进程运行，当您知道容器内运行的进程时，这样其他进程，比如加密挖矿进程，将被 AppArmor 阻止。

利用准入控制器的力量，为您的工作负载部署构建一个门卫。接下来，让我们谈谈在运行时保护工作负载。

## 保护运行时

很可能，您的 Kubernetes 集群是与黑客作战的前线。尽管我们讨论了不同的策略来保护构建和部署，但所有这些策略最终都旨在减少 Kubernetes 集群中的攻击面。您不能简单地闭上眼睛，假设您的 Kubernetes 集群一切都会好起来。这就是为什么我们在[*第十章*]（B15566_10_Final_ASB_ePub.xhtml#_idTextAnchor305）中谈论资源监控，*Kubernetes 集群的实时监控和资源管理*，以及审计、秘钥管理、检测和取证在[*第十一章*]（B15566_11_Final_ASB_ePub.xhtml#_idTextAnchor324）中，*深度防御*。总结一下在这两章中涵盖的内容，以下是保护运行时的关键对策：

+   部署像 Prometheus 和 Grafana 这样的良好的监控工具，以监控 Kubernetes 集群中的资源使用情况。这对于确保服务的可用性至关重要，而且像加密货币挖矿这样的攻击可能会引发 CPU 使用率的激增。

+   启用 Kubernetes 的审计策略以记录 Kubernetes 事件和活动。

+   确保基础设施、Kubernetes 组件和工作负载的高可用性。

+   使用像 Vault 这样的良好的秘密管理工具来管理和提供微服务的秘密。

+   部署像 Falco 这样的良好的检测工具，以侦测 Kubernetes 集群中的可疑活动。

+   最好有取证工具来收集和分析可疑事件。

你可能注意到保护微服务间通信并未被提及。服务网格是一个热门话题，可以帮助保障微服务及其间通信，但出于两个原因，本书未涵盖服务网格：

+   服务网格会给工作负载和 Kubernetes 集群带来性能开销，因此它们还不是保障服务间通信的完美解决方案。

+   从应用安全的角度来看，可以轻松地强制应用程序在 443 端口上监听，并使用 CA 签名证书进行加密通信。如果微服务还执行身份验证和授权，那么只有受信任的微服务才能访问授权资源。服务网格并非保障服务间通信的不可替代解决方案。

为了防御针对 Kubernetes 集群的攻击，我们需要从头到尾保护 Kubernetes 集群的供应、构建、部署和运行。它们都应被视为同等重要，因为你的防御力取决于最薄弱的环节。

# 总结

在本章中，我们回顾了过去两年发生的一些加密货币挖矿攻击，这引起了对保护容器化环境需求的广泛关注。然后，我们向你展示了如何使用不同的开源工具来检测加密货币挖矿攻击。最后但同样重要的是，我们讨论了如何通过总结前几章的内容来保护你的 Kubernetes 集群免受攻击。

我们希望你理解保护 Kubernetes 集群的核心概念，这意味着保护集群的供应、构建、部署和运行阶段。你也应该对开始使用 Anchore、Prometheus、Grafana 和 Falco 感到满意。

众所周知，Kubernetes 仍在不断发展，并不完美。在下一章中，我们将讨论一些已知的 Kubernetes**常见漏洞和曝光**（**CVEs**）以及一些可以保护您的集群免受未知变体影响的缓解措施。以下一章的目的是为了让您能够应对未来发现的任何 Kubernetes CVEs。

# 问题

+   是什么缺陷导致了特斯拉的 Kubernetes 集群中发生了加密挖矿攻击？

+   如果您是特斯拉的 DevOps，您会采取什么措施来防止加密挖矿攻击？

+   当您在一个容器中看到 CPU 使用率激增时，您能否得出结论说发生了加密挖矿攻击？

+   您能想到一种可以绕过“使用 Stratum 协议检测加密挖矿程序”的 Falco 规则的加密挖矿过程吗？

+   为了保护您的 Kubernetes 集群，您需要保护哪四个领域？

# 进一步阅读

有关本章涵盖的主题的更多信息，请参考以下链接：

+   特斯拉加密挖矿攻击：[`redlock.io/blog/cryptojacking-tesla`](https://redlock.io/blog/cryptojacking-tesla)

+   加密蠕虫攻击：[`unit42.paloaltonetworks.com/graboid-first-ever-cryptojacking-worm-found-in-images-on-docker-hub/`](https://unit42.paloaltonetworks.com/graboid-first-ever-cryptojacking-worm-found-in-images-on-docker-hub/)

+   普罗米修斯：[`prometheus.io/docs/introduction/overview/`](https://prometheus.io/docs/introduction/overview/)

+   Falco：[`falco.org/docs/`](https://falco.org/docs/)

+   VirusTotal API：[`developers.virustotal.com/v3.0/reference`](https://developers.virustotal.com/v3.0/reference)

+   加密挖矿攻击分析：[`kromtech.com/blog/security-center/cryptojacking-invades-cloud-how-modern-containerization-trend-is-exploited-by-attackers`](https://kromtech.com/blog/security-center/cryptojacking-invades-cloud-how-modern-containerization-trend-is-exploited-by-attackers)

+   哈勃：[`github.com/cilium/hubble`](https://github.com/cilium/hubble)

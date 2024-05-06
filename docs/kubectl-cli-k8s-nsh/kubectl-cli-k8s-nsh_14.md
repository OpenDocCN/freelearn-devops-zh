# 第十章：kubectl 最佳实践和 Docker 命令

在上一章中，我们学习了 Helm，这是一个 Kubernetes 包管理器。在本书的最后一章中，我们将学习一些`kubectl`的最佳实践。

在本章中，我们将学习如何使用 shell 别名缩短`kubectl`命令，以及使用`kubectl`命令的其他方便技巧。

我们还将检查一些 Docker 中的等效命令，特别是对于熟悉 Docker 命令并想了解`kubectl`中类似命令的新 Kubernetes 用户来说，这些命令非常有用。

在本章中，我们将涵盖以下主要主题：

+   使用 shell 别名来执行 kubectl 命令

+   kubectl 中的类似 Docker 命令

# 使用 shell 别名来执行 kubectl 命令

每次输入`kubectl`命令都很无聊且耗时。您可以在`Bash`和`Zsh` shell 中使用`kubectl`命令完成，当然会有帮助，但仍然不如使用别名快速。

让我们概述一些方便的`kubectl`命令的列表，并将它们与别名一起使用，您可以将它们放在`zsh_aliases`或`bash_aliases`文件中，具体取决于您使用的 shell：

+   `k`代表`kubectl` - 这是不言而喻的。

+   `kg`代表`kubectl get` - 这对于获取 pod、部署、有状态集、服务、节点和其他详细信息非常有用，如下例命令所示：

```
$ kg nodes
```

上述命令的输出如下截图所示：

![图 10.1 - kg 节点输出](img/B16411_10_001.jpg)

图 10.1 - kg 节点输出

上述截图显示了通过运行`$ kg nodes`命令在集群中列出的可用 Kubernetes 节点的列表。

+   `kd`代表`kubectl describe` - 这对于描述 pod、部署、有状态集、服务、节点等非常有用。

+   `kga`代表`kubectl get all` - 这显示了当前设置的命名空间中的 pod、部署、有状态集、服务和资源的列表。您还可以提供`-n`标志来指定命名空间，或者`-A`来显示所有命名空间中的资源：

```
$ kga
```

上述命令的输出如下截图所示：

![图 10.2 - kga 输出](img/B16411_10_002.jpg)

图 10.2 - kga 输出

上述截图显示了`kga`别名的输出，显示了当前命名空间中找到的资源。

+   `krga` 代表 `kubectl really get all`—这会显示当前设置的命名空间中所有资源的列表，包括 secrets、events 等。您也可以提供 `-n` 标志来指定命名空间，或者 `-A` 来显示所有命名空间中的所有资源。

+   `kp` 代表 `kubectl get pods -o wide`—这会显示当前命名空间中 pods 的列表。`-o wide` 标志会显示给定 pod 的分配 IP 和它所在的节点：

```
$ k get pods
$ kp
```

上述命令的输出显示在以下截图中：

![图 10.3 – kgak get pods 输出](img/B16411_10_003.jpg)

图 10.3 – kgak get pods 输出

上述截图显示了 `k get pods` 和 `kp` 的输出。

+   `kap` 代表 `kubectl get pods -A -o wide`—这是一个类似于 `kp` 的别名，但会显示所有命名空间中的 pods。

+   `ka` 代表 `kubectl apply -f`—您可以使用这个命令来创建/更新一个部署：

```
$ ka nginx.yaml
```

+   `kei` 代表 `kubectl exec -it`—这会执行进入运行中 pod 的 shell：

```
$ kei nginx-fcb5d6b64-x4kwg – bash
```

上述命令的输出显示在以下截图中：

![图 10.4 – kei 输出](img/B16411_10_004.jpg)

图 10.4 – kei 输出

上述截图显示了 `kei nginx-fcb5d6b64-x4kwg bash – bash` 的输出。

+   `ke` 代表 `kubectl exec`—这会在运行的 pod 中执行一个命令：

```
$ ke nginx-fcb5d6b64-x4kwg -- ls -alh
```

上述命令的输出显示在以下截图中：

![图 10.5 – ke 输出](img/B16411_10_005.jpg)

图 10.5 – ke 输出

上述截图显示了 `ke nginx-fcb5d6b64-x4kwg bash – ls -alh` 的输出。

+   `ktn` 代表 `watch kubectl top nodes`—使用这个命令来观察节点的资源消耗：

```
$ ktn
```

上述命令的输出显示在以下截图中：

![图 10.6 – ktn 输出](img/B16411_10_006.jpg)

图 10.6 – ktn 输出

上述截图显示了 `ktn` 的输出，列出了节点及其相应的资源使用情况。

+   `ktp` 代表 `watch kubectl top pods`—使用这个命令来观察 pod 的资源消耗：

```
$ ktp
```

上述命令的输出显示在以下截图中：

![图 10.7 – ktp 输出](img/B16411_10_007.jpg)

图 10.7 – ktp 输出

上述截图显示了 `ktp` 的输出，列出了 pods 及其资源使用情况。

+   `kpf` 代表 `kubectl port-forward`—使用这个命令进行端口转发，以便我们可以从 `localhost` 访问 pod：

```
$ kpf nginx-fcb5d6b64-x4kwg 8080
```

上述命令的输出显示在以下截图中：

![图 10.8 – kpf 输出](img/B16411_10_008.jpg)

图 10.8 – kpf 输出

上述截图显示了设置端口转发到端口`8080`的`kpf`的输出。

+   `kl`代表`kubectl logs` - 这显示了一个 pod 或 deployment 的日志：

```
$ kl deploy/nginx --tail 10
```

上述命令的输出显示在以下截图中：

![图 10.9 - kl 输出](img/B16411_10_009.jpg)

图 10.9 - kl 输出

上述截图显示了`kl`的输出，显示了`nginx`部署的日志。

另外，您还可以将以下内容添加到您的列表中：

+   `d`: `docker`

+   `kz`: `kustomize`

+   `h`: `helm`

以下是`.zsh_aliases`的示例片段：

```
$ cat .zsh_aliases
# aliases
alias a="atom ."
alias c="code ."
alias d="docker"
alias h="helm"
alias k="kubectl"
alias ke="kubectl exec -it"
alias kc="kubectl create -f"
alias ka="kubectl apply -f"
alias kd="kubectl describe"
alias kl="kubectl logs"
alias kg="kubectl get"
alias kp="kubectl get pods -o wide"
alias kap="kubectl get pods --all-namespaces -o wide"
alias ktn="watch kubectl top nodes"
alias ktp="watch kubectl top pods"
alias ktc="watch kubectl top pods --containers"
alias kpf="kubectl port-forward"
alias kcx="kubectx"
alias kns="kubectl-ns"
```

使用别名将帮助您更高效地输入几个字母而不是几个单词。此外，并非所有命令都容易记住，因此使用别名也将有助于克服这一点。

# 在 kubectl 中类似的 Docker 命令

以下是最有用的 Docker 命令列表，以及它们在 kubectl 中的等价物。

获取信息使用以下命令完成：

+   `docker info`

+   `kubectl cluster-info`

获取版本信息使用以下命令完成：

+   `docker version`

+   `kubectl version`

运行一个容器并暴露其端口使用以下命令完成：

+   `docker run -d --restart=always --name nginx -p 80:80 nginx`

+   `kubectl create deployment --image=nginx nginx`

+   `kubectl expose deployment nginx --port=80 --name=nginx`

获取容器日志使用以下命令完成：

+   `docker logs --f <container name>`

+   `kubectl logs --f <pod name>`

进入运行中的容器/ pod shell 使用以下命令完成：

+   `docker exec –it <container name> /bin/bash`

+   `kubectl exec –it <pod name>`

获取容器/ pod 列表使用以下命令完成：

+   `docker ps –a`

+   `kubectl get pods`

停止和删除容器/ pod 使用以下命令完成：

+   `docker stop <container name> && docker rm <container name>`

+   `kubectl delete deployment <deployment name>`

+   `kubectl delete pod <pod name>`

我们现在已经学会了 Docker 用户最有用的 kubectl 命令，这应该加快您学习 kubectl 的速度，并且将成为您日常工作中有用的命令。

# 摘要

在本章的最后，我们通过查看如何使用别名来运行各种 kubectl 命令，然后看到了 kubectl 中 Docker 命令的一些等价物，学习了一些 kubectl 的最佳实践。

使用别名缩短了输入所需的时间，当然，别名比一些长命令更容易记住。

在整本书中，我们学到了很多有用的信息，比如如何安装 `kubectl`；获取有关集群和节点的信息；安装、更新和调试应用程序；使用 `kubectl` 插件；还学习了 Kustomize 和 Helm。

我希望这本书能帮助你掌握 Kubernetes、`kubectl` 和 Helm。

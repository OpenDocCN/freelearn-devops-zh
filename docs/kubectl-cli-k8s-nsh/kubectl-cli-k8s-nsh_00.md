# 前言

本书是为那些初次接触通过命令行管理 Kubernetes 的人提供的全面介绍，将帮助您迅速掌握相关知识。

Kubernetes 是一个用于自动化应用部署、扩展和管理的开源容器编排系统，`kubectl`是一个帮助管理它的命令行工具。

# 本书是为谁准备的

本书适用于 DevOps、开发人员、系统管理员以及所有希望使用`kubectl`命令行执行 Kubernetes 功能的人，他们可能了解 Docker，但尚未掌握使用`kubectl`将容器部署到 Kubernetes 的方法。

# 本书涵盖内容

*第一章*，*介绍和安装 kubectl*，提供了对`kubectl`的简要概述以及如何安装和设置它。

*第二章*，*获取有关集群的信息*，教读者如何获取有关集群和可用 API 列表的信息。

*第三章*，*使用节点*，教读者如何获取有关集群节点的信息。

*第四章*，*创建和部署应用程序*，解释了如何创建和安装 Kubernetes 应用程序。

*第五章*，*更新和删除应用程序*，解释了如何更新 Kubernetes 应用程序。

*第六章*，*调试应用程序*，解释了如何查看应用程序日志，`exec`到容器中。

*第七章*，*使用 kubectl 插件*，解释了如何安装`kubectl`插件。

*第八章*，*介绍 Kustomize for kubectl*，讨论了 Kustomize。

*第九章*，*介绍 Helm for Kubernetes*，讨论了 Helm，Kubernetes 包管理器。

*第十章*，*kubectl 最佳实践和 Docker 命令*，涵盖了`kubectl`最佳实践和`kubectl`中的 Docker 等效命令。

# 要充分利用本书

![Table_16411](img/B16411_Preface_Table1.jpg)

**我们建议通过 GitHub 存储库访问代码（链接在** **下一节中可用）。这样做将帮助您避免与复制和粘贴代码相关的任何潜在错误。**

# 下载示例代码文件

您可以从 GitHub 上下载本书的示例代码文件，网址为[`github.com/PacktPublishing/kubectl-Command-Line-Kubernetes-in-a-Nutshell`](https://github.com/PacktPublishing/kubectl-Command-Line-Kubernetes-in-a-Nutshell)。如果代码有更新，将在现有的 GitHub 存储库中进行更新。

我们还有来自我们丰富的书籍和视频目录的其他代码包，可在[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)上找到。去看看吧！

# 下载彩色图片

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图片。您可以在这里下载：[`static.packt-cdn.com/downloads/9781800561878_ColorImages.pdf`](https://static.packt-cdn.com/downloads/9781800561878_ColorImages.pdf)。

# 使用的约定

本书中使用了许多文本约定。

`文本中的代码`：表示文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。这里有一个例子：“在您的主目录中创建`.kube`目录。”

代码块设置如下：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
  labels:
    app: postgresql
```

当我们希望引起您对代码块的特定部分的注意时，相关行或项目将以粗体显示：

```
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
```

任何命令行输入或输出都是这样写的：

```
$ kubectl version –client --short
Client Version: v1.18.1
```

**粗体**：表示新术语、重要单词或屏幕上看到的单词。例如，菜单或对话框中的单词会在文本中以这种方式出现。这里有一个例子：“我们为节点分配了**标签**和**注释**，并且没有设置**角色**或**污点**。”

提示或重要说明

看起来像这样。

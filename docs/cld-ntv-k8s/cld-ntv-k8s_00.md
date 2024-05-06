# 前言

本书的目的是为您提供构建使用 Kubernetes 的云原生应用程序所需的知识和广泛的工具集。Kubernetes 是一种强大的技术，为工程师提供了强大的工具，以使用容器构建云原生平台。该项目本身不断发展，并包含许多不同的工具来解决常见的场景。

对于本书的布局，我们不会局限于 Kubernetes 工具集的任何一个特定领域，而是首先为您提供默认 Kubernetes 功能的最重要部分的全面摘要，从而为您提供在 Kubernetes 上运行应用程序所需的所有技能。然后，我们将为您提供在第 2 天场景中处理 Kubernetes 安全性和故障排除所需的工具。最后，我们将超越 Kubernetes 本身的界限，探讨一些强大的模式和技术，以构建在 Kubernetes 之上的内容，例如服务网格和无服务器。

# 这本书是为谁准备的

这本书是为初学者准备的，但您应该对容器和 DevOps 原则非常熟悉，以便充分利用本书。对 Linux 有扎实的基础将有所帮助，但并非完全必要。

# 本书涵盖内容

第一章《与 Kubernetes 通信》向您介绍了容器编排的概念以及 Kubernetes 工作原理的基础知识。它还为您提供了与 Kubernetes 集群通信和认证所需的基本工具。

第二章《设置您的 Kubernetes 集群》将指导您通过几种不同的流行方式在本地机器和云上创建 Kubernetes 集群。

第三章《在 Kubernetes 上运行应用程序容器》向您介绍了在 Kubernetes 上运行应用程序的最基本构建块 - Pod。我们将介绍如何创建 Pod，以及 Pod 生命周期的具体内容。

第四章《扩展和部署您的应用程序》回顾了更高级的控制器，这些控制器允许扩展和升级应用程序的多个 Pod，包括自动扩展。

第五章，服务和入口 - 与外部世界通信，介绍了将在 Kubernetes 集群中运行的应用程序暴露给外部用户的几种方法。

第六章，Kubernetes 应用程序配置，为您提供了在 Kubernetes 上运行的应用程序提供配置（包括安全数据）所需的技能。

第七章，Kubernetes 上的存储，回顾了为在 Kubernetes 上运行的应用程序提供持久性和非持久性存储的方法和工具。

第八章，Pod 放置控制，介绍了控制和影响 Kubernetes 节点上 Pod 放置的几种不同工具和策略。

第九章，Kubernetes 上的可观察性，涵盖了在 Kubernetes 上下文中可观察性的多个原则，包括指标、跟踪和日志记录。

第十章，Kubernetes 故障排除，回顾了 Kubernetes 集群可能出现故障的一些关键方式，以及如何有效地对 Kubernetes 上的问题进行分类。

第十一章，Kubernetes 上的模板代码生成和 CI/CD，介绍了 Kubernetes YAML 模板工具和一些常见的 Kubernetes 上的 CI/CD 模式。

第十二章，Kubernetes 安全和合规性，涵盖了 Kubernetes 安全的基础知识，包括 Kubernetes 项目的一些最近的安全问题，以及集群和容器安全的工具。

第十三章，使用 CRD 扩展 Kubernetes，介绍了自定义资源定义（CRD）以及其他向 Kubernetes 添加自定义功能的方法，如操作员。

第十四章，服务网格和无服务器，回顾了 Kubernetes 上的一些高级模式，教您如何向集群添加服务网格并启用无服务器工作负载。

第十五章，Kubernetes 上的有状态工作负载，详细介绍了在 Kubernetes 上运行有状态工作负载的具体内容，包括运行生态系统中一些强大的有状态应用程序的教程。

# 充分利用本书

由于 Kubernetes 基于容器，本书中的一些示例可能使用自出版以来发生了变化的容器。其他说明性示例可能使用在 Docker Hub 中不存在的容器。这些示例应作为运行您自己的应用程序容器的基础。

![](img/Preface_table_1.1.jpg)

在某些情况下，像 Kubernetes 这样的开源软件可能会有重大变化。本书与 Kubernetes 1.19 保持最新，但始终检查文档（对于 Kubernetes 和本书涵盖的任何其他开源项目）以获取最新信息和规格说明。

**如果您使用本书的数字版本，我们建议您自己输入代码或通过 GitHub 存储库（链接在下一节中提供）访问代码。这样做将帮助您避免与复制和粘贴代码相关的任何潜在错误。**

# 下载示例代码文件

您可以从 GitHub 上的[`github.com/PacktPublishing/Cloud-Native-with-Kubernetes`](https://github.com/PacktPublishing/Cloud-Native-with-Kubernetes)下载本书的示例代码文件。如果代码有更新，将在现有的 GitHub 存储库上进行更新。

我们还提供了来自我们丰富书籍和视频目录的其他代码包，可在[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)上找到！

# 下载彩色图片

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。您可以在这里下载：[`www.packtpub.com/sites/default/files/downloads/9781838823078_ColorImages.pdf`](http://www.packtpub.com/sites/default/files/downloads/9781838823078_ColorImages.pdf)。

# 使用的约定

本书中使用了许多文本约定。

`文本中的代码`：指示文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。以下是一个例子：“在我们的情况下，我们希望让集群上的每个经过身份验证的用户创建特权 Pod，因此我们绑定到`system:authenticated`组。”

代码块设置如下：

[PRE0]

当我们希望引起您对代码块的特定部分的注意时，相关行或项目将以粗体显示：

[PRE1]

任何命令行输入或输出都以以下方式编写：

[PRE2]

**粗体**：表示一个新术语，一个重要的词，或者屏幕上看到的词。例如，菜单或对话框中的单词会以这种方式出现在文本中。这是一个例子：“Prometheus 还提供了一个用于配置 Prometheus 警报的**警报**选项卡。”

提示或重要说明

像这样出现。

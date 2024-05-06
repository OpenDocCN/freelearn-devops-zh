# 前言

目前，容器化被认为是实施 DevOps 的最佳方式。虽然 Docker 引入了容器并改变了 DevOps 时代，但 Google 开发了一个广泛的容器编排系统 Kubernetes，现在被认为是容器编排的领先者。本书的主要目标是了解使用 Helm 管理在 Kubernetes 上运行的应用的效率。本书将从简要介绍 Helm 及其如何有益于整个容器环境开始。然后，您将深入了解架构方面，以及学习 Helm 图表及其用例。您将学习如何编写 Helm 图表以实现在 Kubernetes 上自动化应用部署。本书专注于提供围绕 Helm 和自动化的企业就绪模式，涵盖了围绕 Helm 的应用开发、交付和生命周期管理的最佳实践。通过本书，您将了解如何利用 Helm 开发企业模式，以实现应用交付。

# 本书适合对象

本书面向对学习 Helm 以实现在 Kubernetes 上应用开发自动化感兴趣的 Kubernetes 开发人员或管理员。具备基本的 Kubernetes 应用开发知识会很有帮助，但不需要事先了解 Helm。建议具备自动化提供的业务用例的基本知识。

# 本书涵盖内容

*第一章*，*理解 Kubernetes 和 Helm*，介绍了 Kubernetes 和 Helm。您将了解在将应用部署到 Kubernetes 时用户面临的挑战，以及 Helm 如何帮助简化部署并提高生产力。

*第二章*，*准备 Kubernetes 和 Helm 环境*，涵盖了在本地 Kubernetes 集群上使用 Helm 部署应用所需的工具。此外，您还将了解安装后发生的基本 Helm 配置。

*第三章*，*安装您的第一个 Helm 图表*，解释了如何通过安装 Helm 图表将应用部署到 Kubernetes，并涵盖了使用 Helm 部署的应用的不同生命周期阶段。

*第四章*，*理解 Helm 图表*，深入探讨了 Helm 图表的构建模块，并为您提供构建自己的 Helm 图表所需的知识。

*第五章*“构建您的第一个 Helm 图表”，提供了一个构建 Helm 图表的端到端演练。本章从构建利用基本 Helm 构造的 Helm 图表的基本概念开始，并逐渐修改基线配置以包含更高级的 Helm 构造。最后，您将学习如何将图表部署到基本图表存储库

*第六章*“测试 Helm 图表”，讨论了围绕对 Helm 图表进行 linting 和测试的不同方法论。

*第七章*“使用 CI/CD 和 GitOps 自动化 Helm 流程”，探讨了在利用 CI/CD 和 GitOps 模型自动化 Helm 任务方面的高级用例。即，围绕测试、打包和发布 Helm 图表开发一个流程。此外，还介绍了在多个不同环境中管理 Helm 图表安装的方法。

*第八章*“使用 Operator 框架与 Helm”，讨论了在 Kubernetes 上使用 operator 的基本概念，以便利用 operator 框架提供的 operator-sdk 工具从现有的 Helm 图表构建一个 Helm operator。

*第九章*“Helm 安全注意事项”，深入探讨了在使用 Helm 时的一些安全注意事项和预防措施，从安装工具到在 Kubernetes 集群上安装 Helm 图表的整个过程。

# 为了充分利用本书

虽然不是强制性的，因为基本概念在整本书中都有解释，但建议对 Kubernetes 和容器技术有一定了解。

对于本书中使用的工具，第 2-9 章将重点关注以下关键技术：

![](img/B15458_Preface_Table_1.jpg)

这些工具的安装在*第二章*“准备 Kubernetes 和 Helm 环境”中有详细讨论。本书中使用的其他工具是特定于章节的，它们的安装方法在使用它们的章节中进行描述。

如果您使用的是本书的数字版本，我们建议您自己输入代码或通过 GitHub 存储库（链接在下一节中提供）访问代码。这样做将有助于避免与复制/粘贴代码相关的任何潜在错误。

# 下载示例代码文件

您可以从[www.packt.com](http://packt.com)的帐户中下载本书的示例代码文件。如果您在其他地方购买了这本书，您可以访问[www.packtpub.com/support](https://www.packtpub.com/support)并注册，以便文件直接通过电子邮件发送给您。

您可以按照以下步骤下载代码文件：

1.  登录或注册[www.packt.com](http://packt.com)。

1.  选择“支持”选项卡。

1.  点击“代码下载”。

1.  在搜索框中输入书名，然后按照屏幕上的说明进行操作。

文件下载后，请确保使用最新版本的解压缩或提取文件夹：

+   WinRAR/7-Zip 用于 Windows

+   Zipeg/iZip/UnRarX 用于 Mac

+   7-Zip/PeaZip 用于 Linux

该书的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/-Learn-Helm`](https://github.com/PacktPublishing/-Learn-Helm)。如果代码有更新，将在现有的 GitHub 存储库上进行更新。

我们还有来自我们丰富的图书和视频目录的其他代码包，可在 https://github.com/PacktPublishing/上找到。去看看吧！

## 代码实例

本书的实际代码演示视频可在 https://bit.ly/2AEAGvm 上观看。

## 下载彩色图像

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。您可以在这里下载：[`www.packtpub.com/sites/default/files/downloads/9781839214295_ColorImages.pdf`](http://www.packtpub.com/sites/default/files/downloads/9781839214295_ColorImages.pdf)。

## 使用的约定

本书中使用了许多文本约定。

`文本中的代码`：表示文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。这是一个例子：“将下载的`WebStorm-10*.dmg`磁盘映像文件挂载为系统中的另一个磁盘。”

代码块设置如下：

```
html, body, #map {
 height: 100%; 
 margin: 0;
 padding: 0
}
```

当我们希望引起您对代码块的特定部分的注意时，相关的行或项目会以粗体显示：

```
[default]
exten => s,1,Dial(Zap/1|30)
exten => s,2,Voicemail(u100)
exten => s,102,Voicemail(b100)
exten => i,1,Voicemail(s0)
```

任何命令行输入或输出都以以下方式编写：

```
$ mkdir css
$ cd css
```

**粗体**：表示新术语、重要单词或屏幕上看到的单词。例如，菜单或对话框中的单词会以这样的方式出现在文本中。这是一个例子：“从**管理**面板中选择**系统信息**”。

提示或重要说明

以这种方式出现。

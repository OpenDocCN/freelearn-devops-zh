# 前言

Docker 在现代应用程序部署和架构方面是一个改变游戏规则的因素。它现在已经发展成为创新的关键驱动力，超越了系统管理，并对 Web 开发等领域产生了影响。但是，您如何确保您跟上了它所推动的创新？您如何确保您充分发挥了它的潜力？

本书向您展示了如何做到这一点；它不仅演示了如何更有效地使用 Docker，还帮助您重新思考和重新想象 Docker 的可能性。

您还将涵盖基本主题，如构建、管理和存储图像，以及在深入研究 Docker 安全性之前使您信心十足的最佳实践。您将找到与扩展和集成 Docker 相关的一切新颖创新的方法。Docker Compose，Docker Swarm 和 Kubernetes 将帮助您以高效的方式控制容器。

通过本书，您将对 Docker 的可能性有一个广泛而详细的认识，以及它如何无缝地融入到您的本地工作流程中，以及高可用的公共云平台和其他工具。

# 本书适合谁

如果您是 IT 专业人士，并认识到 Docker 在从系统管理到 Web 开发的创新中的重要性，但不确定如何充分利用它，那么本书适合您。

# 要充分利用本书

要充分利用本书，您需要一台能够运行 Docker 的机器。这台机器应至少具有 8GB RAM 和 30GB 可用硬盘空间，配备 Intel i3 或更高版本，运行以下操作系统之一：

+   macOS High Sierra 或更高版本

+   Windows 10 专业版

+   Ubuntu 18.04

此外，您将需要访问以下公共云提供商之一或全部：DigitalOcean，Amazon Web Services，Microsoft Azure 和 Google Cloud。

# 下载示例代码文件

您可以从[www.packt.com](http://www.packt.com)的帐户中下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问[www.packt.com/support](http://www.packt.com/support)并注册，以便文件直接发送到您的邮箱。

您可以按照以下步骤下载代码文件：

1.  在[www.packt.com](http://www.packt.com)登录或注册。

1.  选择“支持”选项卡。

1.  点击“代码下载和勘误”。

1.  在搜索框中输入书名，然后按照屏幕上的说明操作。

文件下载后，请确保使用最新版本的解压缩软件解压缩文件夹。

+   WinRAR/7-Zip 适用于 Windows

+   Zipeg/iZip/UnRarX 适用于 Mac

+   7-Zip/PeaZip 适用于 Linux

本书的代码包也托管在 GitHub 上，网址为 [`github.com/PacktPublishing/Mastering-Docker-Third-Edition`](https://github.com/PacktPublishing/Mastering-Docker-Third-Edition)。如果代码有更新，将在现有的 GitHub 存储库上进行更新。

我们还有其他代码包，来自我们丰富的图书和视频目录，可在 **[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)** 上找到。快去看看吧！

# 下载彩色图像

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。您可以在这里下载：`www.packtpub.com/sites/default/files/downloads/9781789616606_ColorImages.pdf`。

# 代码示例

访问以下链接查看代码运行的视频：

[`bit.ly/2PUB9ww`](http://bit.ly/2PUB9ww)

# 使用的约定

本书中使用了许多文本约定。

`CodeInText`：表示文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。这是一个例子：“第一个文件是`nginx.conf`，其中包含基本的 nginx 配置文件。”

代码块设置如下：

[PRE0]

任何命令行输入或输出都将按以下方式编写：

[PRE1]

粗体：表示新术语、重要单词或屏幕上看到的单词。例如，菜单或对话框中的单词会以这种方式出现在文本中。这是一个例子：“单击“创建”后，您将被带到类似下一个屏幕截图的屏幕。”

警告或重要说明会以这种方式出现。提示和技巧会以这种方式出现。

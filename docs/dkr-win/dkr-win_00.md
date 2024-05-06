# 前言

容器是运行软件的新方式。它们高效、安全、可移植，您可以在 Docker 中运行 Windows 应用程序而无需进行代码更改。Docker 帮助您应对 IT 中的最大挑战：现代化传统应用程序、构建新应用程序、迁移到云端、采用 DevOps 并保持创新。

本书将教会您有关 Windows 上 Docker 的一切，从基础知识到在生产环境中运行高可用负载。您将通过一个 Docker 之旅，从关键概念和在 Windows 上的.NET Framework 和.NET Core 应用程序的简单示例开始。然后，您将学习如何使用 Docker 来现代化传统的 ASP.NET 和 SQL Server 应用程序的架构和开发。

这些示例向您展示了如何将传统的单片应用程序拆分为分布式应用程序，并将它们部署到云端的集群环境中，使用与本地运行时完全相同的构件。您将了解如何构建使用 Docker 来编译、打包、测试和部署应用程序的 CI/CD 流水线。为了帮助您自信地进入生产环境，您将学习有关 Docker 安全性、管理和支持选项的知识。

本书最后将指导您如何在自己的项目中开始使用 Docker。您将学习一些 Docker 实施的真实案例，从小规模的本地应用到在 Azure 上运行的大规模应用。

# 本书适合对象

如果您想要现代化旧的单片应用程序而不必重写它，平稳地部署到生产环境，或者迁移到 DevOps 或云端，那么 Docker 就是您的实现者。本书将为您提供 Docker 的扎实基础，让您能够自信地应对所有这些情况。

# 要充分利用本书

本书附带了大量代码，存储在 GitHub 的`sixeyed/docker-on-windows`仓库中。要使用这些示例，您需要：

+   Windows 10 与 1809 更新，或 Windows Server 2019

+   Docker 版本 18.09 或更高

# 下载示例代码文件

您可以从您在[www.packt.com](http://www.packt.com)的账户中下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问[www.packt.com/support](http://www.packt.com/support)并注册，文件将直接通过电子邮件发送给您。

您可以按照以下步骤下载代码文件：

1.  在[www.packt.com](http://www.packt.com)上登录或注册。

1.  选择“支持”选项卡。

1.  点击“代码下载和勘误”。

1.  在搜索框中输入书名，然后按照屏幕上的说明操作。

下载文件后，请确保使用最新版本的解压或提取文件夹：

+   WinRAR/7-Zip for Windows

+   Zipeg/iZip/UnRarX for Mac

+   7-Zip/PeaZip for Linux

该书的代码包也托管在 GitHub 上：[`github.com/PacktPublishing/Docker-on-Windows-Second-Edition`](https://github.com/PacktPublishing/Docker-on-Windows-Second-Edition)。如果代码有更新，将在现有的 GitHub 存储库中更新。

我们还有来自我们丰富书籍和视频目录的其他代码包，可在**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**上找到。去看看吧！

代码包也可以在作者的 GitHub 存储库中找到：[`github.com/sixeyed/docker-on-windows/tree/second-edition`](https://github.com/sixeyed/docker-on-windows/tree/second-edition)。

# 下载彩色图像

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。你可以在这里下载：`www.packtpub.com/sites/default/files/downloads/9781789617375_ColorImages.pdf`。

# 使用的约定

本书中使用了许多文本约定。

`CodeInText`：表示文本中的代码词，数据库表名，文件夹名，文件名，文件扩展名，路径名，虚拟 URL，用户输入和 Twitter 句柄。这是一个例子：“作为 Azure 门户的替代方案，你可以使用`az`命令行来管理 DevTest 实验室。”

代码块设置如下：

```
<?xml version="1.0" encoding="utf-8"?> <configuration>
  <appSettings  configSource="config\appSettings.config"  />
  <connectionStrings  configSource="config\connectionStrings.config"  /> </configuration>
```

任何命令行输入或输出都以以下方式编写：

```
> docker version
```

**粗体**：表示一个新术语，一个重要的词，或者你在屏幕上看到的词。例如，菜单或对话框中的单词会以这种方式出现在文本中。这是一个例子：“在你做任何其他事情之前，你需要选择切换到 Windows 容器…”

警告或重要提示会以这种方式出现。提示和技巧会以这种方式出现。

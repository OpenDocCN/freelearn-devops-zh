# 第六章：使用 Ansible 进行配置管理

我们已经涵盖了持续交付过程的两个最关键的阶段：提交阶段和自动接受测试。在本章中，我们将专注于配置管理，将虚拟容器化环境与真实服务器基础设施连接起来。

本章涵盖以下要点：

+   介绍配置管理的概念

+   解释最流行的配置管理工具

+   讨论 Ansible 的要求和安装过程

+   使用 Ansible 进行即时命令

+   展示 Ansible 自动化的强大力量与 playbooks

+   解释 Ansible 角色和 Ansible Galaxy

+   实施部署过程的用例

+   使用 Ansible 与 Docker 和 Docker Compose 一起

# 介绍配置管理

配置管理是一种控制配置更改的过程，以使系统随时间保持完整性。即使这个术语并非起源于 IT 行业，但目前它被广泛用来指代软件和硬件。在这个背景下，它涉及以下方面：

+   **应用程序配置**：这涉及决定系统如何工作的软件属性，通常以传递给应用程序的标志或属性文件的形式表达，例如数据库地址、文件处理的最大块大小或日志级别。它们可以在不同的开发阶段应用：构建、打包、部署或运行。

+   **基础设施配置**：这涉及服务器基础设施和环境配置，负责部署过程。它定义了每台服务器应安装哪些依赖项，并指定了应用程序的编排方式（哪个应用程序在哪个服务器上运行以及有多少个实例）。

举个例子，我们可以想象一个使用 Redis 服务器的计算器 Web 服务。让我们看一下展示配置管理工具如何工作的图表。

![](img/886430b5-6e25-4fba-925d-5e18c53eea0d.png)

配置管理工具读取配置文件并相应地准备环境（安装依赖工具和库，将应用程序部署到多个实例）。

在前面的例子中，**基础设施配置**指定了**计算器**服务应该在**服务器 1**和**服务器 2**上部署两个实例，并且**Redis**服务应该安装在**服务器 3**上。**计算器应用程序配置**指定了**Redis**服务器的端口和地址，以便服务之间可以通信。

配置可能因环境类型（QA、staging、production）的不同而有所不同，例如，服务器地址可能不同。

配置管理有许多方法，但在我们研究具体解决方案之前，让我们评论一下一个好的配置管理工具应该具备的特征。

# 良好配置管理的特点

现代配置管理解决方案应该是什么样的？让我们来看看最重要的因素：

+   **自动化**：每个环境都应该自动可再现，包括操作系统、网络配置、安装的软件和部署的应用程序。在这种方法中，修复生产问题意味着自动重建环境。更重要的是，这简化了服务器复制，并确保暂存和生产环境完全相同。

+   **版本控制**：配置的每个更改都应该被跟踪，这样我们就知道是谁做的，为什么，什么时候。通常，这意味着将配置保存在源代码存储库中，要么与代码一起，要么在一个单独的地方。前者的解决方案是推荐的，因为配置属性的生命周期与应用程序本身不同。版本控制还有助于修复生产问题-配置始终可以回滚到先前的版本，并自动重建环境。唯一的例外是基于版本控制的解决方案是存储凭据和其他敏感信息-这些信息永远不应该被检入。

+   **增量更改**：应用配置的更改不应该需要重建整个环境。相反，配置的小改变应该只改变基础设施的相关部分。

+   **服务器配置**：通过自动化，添加新服务器应该像将其地址添加到配置（并执行一个命令）一样快。

+   **安全性**：对配置管理工具和其控制下的机器的访问应该得到很好的保护。当使用 SSH 协议进行通信时，密钥或凭据的访问需要得到很好的保护。

+   **简单性**：团队的每个成员都应该能够阅读配置，进行更改，并将其应用到环境中。属性本身也应尽可能简单，不受更改影响的属性最好保持硬编码。

在创建配置时以及在选择正确的配置管理工具之前，重要的是要牢记这些要点。

# 配置管理工具概述

最流行的配置管理工具是 Ansible、Puppet 和 Chef。它们每个都是一个不错的选择；它们都是开源产品，有免费的基本版本和付费的企业版本。它们之间最重要的区别是：

+   **配置语言**：Chef 使用 Ruby，Puppet 使用其自己的 DSL（基于 Ruby），而 Ansible 使用 YAML。

+   **基于代理**：Puppet 和 Chef 使用代理进行通信，这意味着每个受管服务器都需要安装特殊工具。相反，Ansible 是无代理的，使用标准的 SSH 协议进行通信。

无代理的特性是一个重要的优势，因为它意味着不需要在服务器上安装任何东西。此外，Ansible 正在迅速上升，这就是为什么选择它作为本书的原因。然而，其他工具也可以成功地用于持续交付过程。

# 安装 Ansible

Ansible 是一个开源的、无代理的自动化引擎，用于软件供应、配置管理和应用部署。它于 2012 年首次发布，其基本版本对个人和商业用途都是免费的。企业版称为 Ansible Tower，提供 GUI 管理和仪表板、REST API、基于角色的访问控制等更多功能。

我们介绍了安装过程以及如何单独使用它以及与 Docker 一起使用的描述。

# Ansible 服务器要求

Ansible 使用 SSH 协议进行通信，对其管理的机器没有特殊要求。也没有中央主服务器，因此只需在任何地方安装 Ansible 客户端工具，就可以用它来管理整个基础架构。

被管理的机器的唯一要求是安装 Python 工具和 SSH 服务器。然而，这些工具几乎总是默认情况下在任何服务器上都可用。

# Ansible 安装

安装说明因操作系统而异。在 Ubuntu 的情况下，只需运行以下命令即可：

[PRE0]

您可以在官方 Ansible 页面上找到所有操作系统的安装指南：[`docs.ansible.com/ansible/intro_installation.html`](http://docs.ansible.com/ansible/intro_installation.html)。

安装过程完成后，我们可以执行 Ansible 命令来检查是否一切都安装成功。

[PRE1]

# 基于 Docker 的 Ansible 客户端

还可以将 Ansible 用作 Docker 容器。我们可以通过运行以下命令来实现：

[PRE2]

Ansible Docker 镜像不再得到官方支持，因此唯一的解决方案是使用社区驱动的版本。您可以在 Docker Hub 页面上阅读更多关于其用法的信息。

# 使用 Ansible

为了使用 Ansible，首先需要定义清单，代表可用资源。然后，我们将能够执行单个命令或使用 Ansible playbook 定义一组任务。

# 创建清单

清单是由 Ansible 管理的所有服务器的列表。每台服务器只需要安装 Python 解释器和 SSH 服务器。默认情况下，Ansible 假定使用 SSH 密钥进行身份验证；但是，也可以通过在 Ansible 命令中添加`--ask-pass`选项来使用用户名和密码进行身份验证。

SSH 密钥可以使用`ssh-keygen`工具生成，并通常存储在`~/.ssh`目录中。

清单是在`/etc/ansible/hosts`文件中定义的，它具有以下结构：

[PRE3]

清单语法还接受服务器范围，例如`www[01-22].company.com`。如果 SSH 端口不是默认的 22 端口，还应该指定。您可以在官方 Ansible 页面上阅读更多信息：[`docs.ansible.com/ansible/intro_inventory.html`](http://docs.ansible.com/ansible/intro_inventory.html)。

清单文件中可能有 0 个或多个组。例如，让我们在一个服务器组中定义两台机器。

[PRE4]

我们还可以创建带有服务器别名的配置，并指定远程用户：

[PRE5]

前面的文件定义了一个名为`webservers`的组，其中包括两台服务器。Ansible 客户端将作为用户`admin`登录到它们两台。当我们创建了清单后，让我们发现如何使用它来在许多服务器上执行相同的命令。

Ansible 提供了从云提供商（例如 Amazon EC2/Eucalyptus）、LDAP 或 Cobbler 动态获取清单的可能性。在[`docs.ansible.com/ansible/intro_dynamic_inventory.html`](http://docs.ansible.com/ansible/intro_dynamic_inventory.html)了解更多关于动态清单的信息。

# 临时命令

我们可以运行的最简单的命令是对所有服务器进行 ping 测试。

[PRE6]

我们使用了`-m <module_name>`选项，允许指定应在远程主机上执行的模块。结果是成功的，这意味着服务器是可达的，并且身份验证已正确配置。

可以在[`docs.ansible.com/ansible/modules.htm`](http://docs.ansible.com/ansible/modules.htm)找到 Ansible 可用模块的完整列表。

请注意，我们使用了`all`，以便可以处理所有服务器，但我们也可以通过组名`webservers`或单个主机别名来调用它们。作为第二个例子，让我们只在其中一个服务器上执行一个 shell 命令。

[PRE7]

`-a <arguments>`选项指定传递给 Ansible 模块的参数。在这种情况下，我们没有指定模块，因此参数将作为 shell Unix 命令执行。结果是成功的，并且打印了`hello`。

如果`ansible`命令第一次连接服务器（或服务器重新安装），那么我们会收到密钥确认消息（当主机不在`known_hosts`中时的 SSH 消息）。由于这可能会中断自动化脚本，我们可以通过取消注释`/etc/ansible/ansible.cfg`文件中的`host_key_checking = False`或设置环境变量`ANSIBLE_HOST_KEY_CHECKING=False`来禁用提示消息。

在其简单形式中，Ansible 临时命令的语法如下：

[PRE8]

临时命令的目的是在不必重复时快速执行某些操作。例如，我们可能想要检查服务器是否存活，或者在圣诞假期关闭所有机器。这种机制可以被视为在一组机器上执行命令，并由模块提供的附加语法简化。然而，Ansible 自动化的真正力量在于 playbooks。

# Playbooks

Ansible playbook 是一个配置文件，描述了服务器应该如何配置。它提供了一种定义一系列任务的方式，这些任务应该在每台机器上执行。Playbook 使用 YAML 配置语言表示，这使得它易于阅读和理解。让我们从一个示例 playbook 开始，然后看看我们如何使用它。

# 定义一个 playbook

一个 playbook 由一个或多个 plays 组成。每个 play 包含一个主机组名称，要执行的任务以及配置细节（例如，远程用户名或访问权限）。一个示例 playbook 可能如下所示：

[PRE9]

此配置包含一个 play，其中：

+   仅在主机`web1`上执行

+   使用`sudo`命令获取 root 访问权限

+   执行两个任务：

+   安装最新版本的`apache2`：Ansible 模块`apt`（使用两个参数`name=apache2`和`state=latest`）检查服务器上是否安装了`apache2`软件包，如果没有，则使用`apt-get`工具安装`apache2`

+   运行`apache2`服务：Ansible 模块`service`（使用三个参数`name=apache2`，`state=started`和`enabled=yes`）检查 Unix 服务`apache2`是否已启动，如果没有，则使用`service`命令启动它

在处理主机时，您还可以使用模式，例如，我们可以使用`web*`来寻址`web1`和`web2`。您可以在[`docs.ansible.com/ansible/intro_patterns.html`](http://docs.ansible.com/ansible/intro_patterns.html)了解更多关于 Ansible 模式的信息。

请注意，每个任务都有一个易于阅读的名称，在控制台输出中使用，例如`apt`和`service`是 Ansible 模块，`name=apache2`，`state=latest`和`state=started`是模块参数。在使用临时命令时，我们已经看到了 Ansible 模块和参数。在前面的 playbook 中，我们只定义了一个 play，但可以有很多 play，并且每个 play 可以与不同的主机组相关联。

例如，我们可以在清单中定义两组服务器：`database`和`webservers`。然后，在 playbook 中，我们可以指定应该在所有托管数据库的机器上执行的任务，以及应该在所有 web 服务器上执行的一些不同的任务。通过使用一个命令，我们可以设置整个环境。

# 执行 playbook

当定义了 playbook.yml 时，我们可以使用`ansible-playbook`命令来执行它。

[PRE10]

如果服务器需要输入`sudo`命令的密码，那么我们需要在`ansible-playbook`命令中添加`--ask-sudo-pass`选项。也可以通过设置额外变量`-e ansible_become_pass=<sudo_password>`来传递`sudo`密码（如果需要）。

已执行 playbook 配置，因此安装并启动了`apache2`工具。请注意，如果任务在服务器上做了一些改变，它会被标记为`changed`。相反，如果没有改变，它会被标记为`ok`。

可以使用`-f <num_of_threads>`选项并行运行任务。

# Playbook 的幂等性

我们可以再次执行命令。

[PRE11]

请注意输出略有不同。这次命令没有在服务器上做任何改变。这是因为每个 Ansible 模块都设计为幂等的。换句话说，按顺序多次执行相同的模块应该与仅执行一次相同。

实现幂等性的最简单方法是始终首先检查任务是否尚未执行，并且仅在尚未执行时执行它。幂等性是一个强大的特性，我们应该始终以这种方式编写我们的 Ansible 任务。

如果所有任务都是幂等的，那么我们可以随意执行它们。在这种情况下，我们可以将 playbook 视为远程机器期望状态的描述。然后，`ansible-playbook`命令负责将机器（或一组机器）带入该状态。

# 处理程序

某些操作应仅在某些其他任务更改时执行。例如，假设您将配置文件复制到远程机器，并且只有在配置文件更改时才应重新启动 Apache 服务器。如何处理这种情况？

例如，假设您将配置文件复制到远程机器，并且只有在配置文件更改时才应重新启动 Apache 服务器。如何处理这种情况？

Ansible 提供了一种基于事件的机制来通知变化。为了使用它，我们需要知道两个关键字：

+   `handlers`：指定通知时执行的任务

+   `notify`：指定应执行的处理程序

让我们看一个例子，我们如何将配置复制到服务器并且仅在配置更改时重新启动 Apache。

[PRE12]

现在，我们可以创建`foo.conf`文件并运行`ansible-playbook`命令。

[PRE13]

处理程序始终在 play 结束时执行，只执行一次，即使由多个任务触发。

Ansible 复制了文件并重新启动了 Apache 服务器。重要的是要理解，如果我们再次运行命令，将不会发生任何事情。但是，如果我们更改`foo.conf`文件的内容，然后运行`ansible-playbook`命令，文件将再次被复制（并且 Apache 服务器将被重新启动）。

[PRE14]

我们使用了`copy`模块，它足够智能，可以检测文件是否已更改，然后在这种情况下在服务器上进行更改。

Ansible 中还有一个发布-订阅机制。使用它意味着将一个主题分配给许多处理程序。然后，一个任务通知主题以执行所有相关的处理程序。您可以在以下网址了解更多信息：[`docs.ansible.com/ansible/playbooks_intro.html`](http://docs.ansible.com/ansible/playbooks_intro.html)。

# 变量

虽然 Ansible 自动化使多个主机的事物变得相同和可重复，但不可避免地，服务器可能需要一些差异。例如，考虑应用程序端口号。它可能因机器而异。幸运的是，Ansible 提供了变量，这是一个处理服务器差异的良好机制。让我们创建一个新的 playbook 并定义一个变量。

例如，考虑应用程序端口号。它可能因机器而异。幸运的是，Ansible 提供了变量，这是一个处理服务器差异的良好机制。让我们创建一个新的 playbook 并定义一个变量。

[PRE15]

配置定义了`http_port`变量的值为`8080`。现在，我们可以使用 Jinja2 语法来使用它。

[PRE16]

Jinja2 语言不仅允许获取变量，还可以用它来创建条件、循环等。您可以在 Jinja 页面上找到更多详细信息：[`jinja.pocoo.org/`](http://jinja.pocoo.org/)。

`debug`模块在执行时打印消息。如果我们运行`ansible-playbook`命令，就可以看到变量的使用情况。

[PRE17]

变量也可以在清单文件中的`[group_name:vars]`部分中定义。您可以在以下网址了解更多信息：[`docs.ansible.com/ansible/intro_inventory.html#host-variables`](http://docs.ansible.com/ansible/intro_inventory.html#host-variables)。

除了用户定义的变量，还有预定义的自动变量。例如，`hostvars`变量存储了有关清单中所有主机信息的映射。使用 Jinja2 语法，我们可以迭代并打印清单中所有主机的 IP 地址。

[PRE18]

然后，我们可以执行`ansible-playbook`命令。

[PRE19]

请注意，使用 Jinja2 语言，我们可以在 Ansible 剧本文件中指定流程控制操作。

对于条件和循环，Jinja2 模板语言的替代方案是使用 Ansible 内置关键字：`when`和`with_items`。您可以在以下网址了解更多信息：[`docs.ansible.com/ansible/playbooks_conditionals.html`](http://docs.ansible.com/ansible/playbooks_conditionals.html)。

# 角色

我们可以使用 Ansible 剧本在远程服务器上安装任何工具。想象一下，我们想要一个带有 MySQL 的服务器。我们可以轻松地准备一个类似于带有`apache2`包的 playbook。然而，如果你想一想，带有 MySQL 的服务器是一个相当常见的情况，肯定有人已经为此准备了一个 playbook，所以也许我们可以重用它？这就是 Ansible 角色和 Ansible Galaxy 的用武之地。

# 理解角色

Ansible 角色是一个精心构建的剧本部分，准备包含在剧本中。角色是独立的单元，始终具有以下目录结构：

[PRE20]

您可以在官方 Ansible 页面上阅读有关角色及每个目录含义的更多信息：[`docs.ansible.com/ansible/playbooks_roles.html`](http://docs.ansible.com/ansible/playbooks_roles.html)。

在每个目录中，我们可以定义`main.yml`文件，其中包含可以包含在`playbook.yml`文件中的剧本部分。继续 MySQL 案例，GitHub 上定义了一个角色：[`github.com/geerlingguy/ansible-role-mysql`](https://github.com/geerlingguy/ansible-role-mysql)。该存储库包含可以在我们的 playbook 中使用的任务模板。让我们看一下`tasks/main.yml`文件的一部分，它安装`mysql`包。

[PRE21]

这只是在`tasks/main.yml`文件中定义的任务之一。其他任务负责 MySQL 配置。

`with_items`关键字用于在所有项目上创建循环。`when`关键字意味着任务仅在特定条件下执行。

如果我们使用这个角色，那么为了在服务器上安装 MySQL，只需创建以下 playbook.yml：

[PRE22]

这样的配置使用`geerlingguy.mysql`角色将 MySQL 数据库安装到所有服务器上。

# Ansible Galaxy

Ansible Galaxy 是 Ansible 的角色库，就像 Docker Hub 是 Docker 的角色库一样，它存储常见的角色，以便其他人可以重复使用。您可以在 Ansible Galaxy 页面上浏览可用的角色：[`galaxy.ansible.com/`](https://galaxy.ansible.com/)。

要从 Ansible Galaxy 安装角色，我们可以使用`ansible-galaxy`命令。

[PRE23]

此命令会自动下载角色。在 MySQL 示例中，我们可以通过执行以下命令下载角色：

[PRE24]

该命令下载`mysql`角色，可以在 playbook 文件中后续使用。

如果您需要同时安装许多角色，可以在`requirements.yml`文件中定义它们，并使用`ansible-galaxy install -r requirements.yml`。了解更多关于这种方法和 Ansible Galaxy 的信息，请访问：[`docs.ansible.com/ansible/galaxy.html`](http://docs.ansible.com/ansible/galaxy.html)。

# 使用 Ansible 进行部署

我们已经介绍了 Ansible 的最基本功能。现在，让我们暂时忘记 Docker，使用 Ansible 配置完整的部署步骤。我们将在一个服务器上运行计算器服务，而在第二个服务器上运行 Redis 服务。

# 安装 Redis

我们可以在新的 playbook 中指定一个 play。让我们创建`playbook.yml`文件，内容如下：

[PRE25]

该配置在一个名为`web1`的服务器上执行。它安装`redis-server`包，复制 Redis 配置，并启动 Redis。请注意，每次更改`redis.conf`文件的内容并重新运行`ansible-playbook`命令时，配置都会更新到服务器上，并且 Redis 服务会重新启动。

我们还需要创建`redis.conf`文件，内容如下：

[PRE26]

此配置将 Redis 作为守护程序运行，并将其暴露给端口号为 6379 的所有网络接口。现在让我们定义第二个 play，用于设置计算器服务。

# 部署 Web 服务

我们分三步准备计算器 Web 服务：

1.  配置项目可执行。

1.  更改 Redis 主机地址。

1.  将计算器部署添加到 playbook 中。

# 配置项目可执行

首先，我们需要使构建的 JAR 文件可执行，以便它可以作为 Unix 服务轻松在服务器上运行。为了做到这一点，只需将以下代码添加到`build.gradle`文件中：

[PRE27]

# 更改 Redis 主机地址

以前，我们已将 Redis 主机地址硬编码为`redis`，所以现在我们应该在`src/main/java/com/leszko/calculator/CacheConfig.java`文件中将其更改为`192.168.0.241`。

在实际项目中，应用程序属性通常保存在属性文件中。例如，对于 Spring Boot 框架，有一个名为`application.properties`或`application.yml`的文件。

# 将计算器部署添加到 playbook 中

最后，我们可以将部署配置作为`playbook.yml`文件中的新 play 添加。

[PRE28]

让我们走一遍我们定义的步骤：

+   **准备环境**：此任务确保安装了 Java 运行时环境。基本上，它准备了服务器环境，以便计算器应用程序具有所有必要的依赖关系。对于更复杂的应用程序，依赖工具和库的列表可能会更长。

+   **将应用程序配置为服务**：我们希望将计算器应用程序作为 Unix 服务运行，以便以标准方式进行管理。在这种情况下，只需在`/etc/init.d/`目录中创建一个指向我们应用程序的链接即可。

+   **复制新版本**：将应用程序的新版本复制到服务器上。请注意，如果源文件没有更改，则文件不会被复制，因此服务不会重新启动。

+   **重新启动服务**：作为处理程序，每次复制应用程序的新版本时，服务都会重新启动。

# 运行部署

与往常一样，我们可以使用`ansible-playbook`命令执行 playbook。在此之前，我们需要使用 Gradle 构建计算器项目。

[PRE29]

成功部署后，服务应该可用，并且我们可以在`http://192.168.0.242:8080/sum?a=1&b=2`上检查它是否正常工作。预期地，它应该返回`3`作为输出。

请注意，我们通过执行一个命令配置了整个环境。而且，如果我们需要扩展服务，只需将新服务器添加到清单中并重新运行`ansible-playbook`命令即可。

我们已经展示了如何使用 Ansible 进行环境配置和应用程序部署。下一步是将 Ansible 与 Docker 一起使用。

# Ansible 与 Docker

正如您可能已经注意到的，Ansible 和 Docker 解决了类似的软件部署问题：

+   **环境配置**：Ansible 和 Docker 都提供了配置环境的方式；然而，它们使用不同的方法。虽然 Ansible 使用脚本（封装在 Ansible 模块中），Docker 将整个环境封装在一个容器中。

+   **依赖性**：Ansible 提供了一种在相同或不同的主机上部署不同服务并让它们一起部署的方式。Docker Compose 具有类似的功能，允许同时运行多个容器。

+   **可扩展性**：Ansible 有助于扩展服务，提供清单和主机组。Docker Compose 具有类似的功能，可以自动增加或减少运行容器的数量。

+   **配置文件自动化**：Docker 和 Ansible 都将整个环境配置和服务依赖关系存储在文件中（存储在源代码控制存储库中）。对于 Ansible，这个文件称为`playbook.yml`。在 Docker 的情况下，我们有 Dockerfile 用于环境和 docker-compose.yml 用于依赖关系和扩展。

+   **简单性**：这两个工具都非常简单易用，并提供了一种通过配置文件和一条命令执行来设置整个运行环境的方式。

如果我们比较这些工具，那么 Docker 做了更多，因为它提供了隔离、可移植性和某种安全性。我们甚至可以想象在没有任何其他配置管理工具的情况下使用 Docker。那么，我们为什么还需要 Ansible 呢？

# Ansible 的好处

Ansible 可能看起来多余；然而，它为交付过程带来了额外的好处：

+   **Docker 环境**：Docker 主机本身必须进行配置和管理。每个容器最终都在 Linux 机器上运行，需要内核打补丁、Docker 引擎更新、网络配置等。而且，可能有不同的服务器机器使用不同的 Linux 发行版，Ansible 的责任是确保 Docker 引擎正常运行。

+   **非 Docker 化应用程序**：并非所有东西都在容器内运行。如果基础设施的一部分是容器化的，另一部分以标准方式或在云中部署，那么 Ansible 可以通过 playbook 配置文件管理所有这些。不以容器方式运行应用程序可能有不同的原因，例如性能、安全性、特定的硬件要求、基于 Windows 的软件，或者与旧软件的工作。

+   **清单**：Ansible 提供了一种非常友好的方式来使用清单管理物理基础设施，清单存储有关所有服务器的信息。它还可以将物理基础设施分成不同的环境：生产、测试、开发。

+   **GUI**：Ansible 提供了一个（商业）名为 Ansible Tower 的 GUI 管理器，旨在改进企业的基础设施管理。

+   **改进测试流程**：Ansible 可以帮助集成和验收测试，并可以以与 Docker Compose 类似的方式封装测试脚本。

我们可以将 Ansible 视为负责基础设施的工具，而将 Docker 视为负责环境配置的工具。概述如下图所示：

![](img/a8d7f1ee-0867-4b62-a53b-0ae730381cd1.png)

Ansible 管理基础设施：Docker 服务器、Docker 注册表、没有 Docker 的服务器和云提供商。它还关注服务器的物理位置。使用清单主机组，它可以将 Web 服务链接到其地理位置附近的数据库。

# Ansible Docker playbook

Ansible 与 Docker 集成得很顺利，因为它提供了一组专门用于 Docker 的模块。如果我们为基于 Docker 的部署创建一个 Ansible playbook，那么第一个任务需要确保 Docker 引擎已安装在每台机器上。然后，它应该使用 Docker 运行一个容器，或者使用 Docker Compose 运行一组交互式容器。

Ansible 提供了一些非常有用的与 Docker 相关的模块：`docker_image`（构建/管理镜像）、`docker_container`（运行容器）、`docker_image_facts`（检查镜像）、`docker_login`（登录到 Docker 注册表）、`docker_network`（管理 Docker 网络）和`docker_service`（管理 Docker Compose）。

# 安装 Docker

我们可以使用 Ansible playbook 中的以下任务来安装 Docker 引擎。

[PRE30]

每个操作系统的 playbook 看起来略有不同。这里介绍的是针对 Ubuntu 16.04 的。

此配置安装 Docker 引擎，使`admin`用户能够使用 Docker，并安装了 Docker Compose 及其依赖工具。

或者，您也可以使用`docker_ubuntu`角色，如此处所述：[`www.ansible.com/2014/02/12/installing-and-building-docker-with-ansible`](https://www.ansible.com/2014/02/12/installing-and-building-docker-with-ansible)。

安装 Docker 后，我们可以添加一个任务，该任务将运行一个 Docker 容器。

# 运行 Docker 容器

使用`docker_container`模块来运行 Docker 容器，它看起来与我们为 Docker Compose 配置所呈现的非常相似。让我们将其添加到`playbook.yml`文件中。

[PRE31]

您可以在官方 Ansible 页面上阅读有关`docker_container`模块的所有选项的更多信息：[`docs.ansible.com/ansible/docker_container_module.html`](https://docs.ansible.com/ansible/docker_container_module.html)。

现在我们可以执行 playbook 来观察 Docker 是否已安装并且 Redis 容器已启动。请注意，这是一种非常方便的使用 Docker 的方式，因为我们不需要在每台机器上手动安装 Docker 引擎。

# 使用 Docker Compose

Ansible playbook 与 Docker Compose 配置非常相似。它们甚至共享相同的 YAML 文件格式。而且，可以直接从 Ansible 使用`docker-compose.yml`。我们将展示如何做到这一点，但首先让我们定义`docker-compose.yml`文件。

[PRE32]

这几乎与我们在上一章中定义的内容相同。这一次，我们直接从 Docker Hub 注册表获取计算器镜像，并且不在`docker-compose.yml`中构建它，因为我们希望构建一次镜像，将其推送到注册表，然后在每个部署步骤（在每个环境中）重复使用它，以确保相同的镜像部署在每台 Docker 主机上。当我们有了`docker-compose.yml`，我们就准备好向`playbook.yml`添加新任务了。

[PRE33]

我们首先将 docker-compose.yml 文件复制到服务器，然后执行`docker-compose`。结果，Ansible 创建了两个容器：计算器和 Redis。

我们已经看到了 Ansible 的最重要特性。在接下来的章节中，我们会稍微介绍一下基础设施和应用程序版本控制。在本章结束时，我们将介绍如何使用 Ansible 来完成持续交付流程。

# 练习

在本章中，我们已经介绍了 Ansible 的基础知识以及与 Docker 一起使用它的方式。作为练习，我们提出以下任务：

1.  创建服务器基础设施并使用 Ansible 进行管理。

+   连接物理机器或运行 VirtualBox 机器来模拟远程服务器

+   配置 SSH 访问远程机器（SSH 密钥）

+   在远程机器上安装 Python

+   创建一个包含远程机器的 Ansible 清单

+   运行 Ansible 的临时命令（使用`ping`模块）来检查基础设施是否配置正确

1.  创建一个基于 Python 的“hello world”网络服务，并使用 Ansible 剧本在远程机器上部署它。

+   服务可以与本章练习中描述的完全相同

+   创建一个部署服务到远程机器的剧本

+   运行`ansible-playbook`命令并检查服务是否已部署

# 总结

我们已经介绍了配置管理过程及其与 Docker 的关系。本章的关键要点如下：

+   配置管理是创建和应用基础设施和应用程序的配置的过程

+   Ansible 是最流行的配置管理工具之一。它是无代理的，因此不需要特殊的服务器配置

+   Ansible 可以与临时命令一起使用，但真正的力量在于 Ansible 剧本

+   Ansible 剧本是环境应该如何配置的定义

+   Ansible 角色的目的是重用剧本的部分。

+   Ansible Galaxy 是一个在线服务，用于共享 Ansible 角色

+   与仅使用 Docker 和 Docker Compose 相比，Ansible 与 Docker 集成良好并带来额外的好处

在下一章中，我们将结束持续交付过程并完成最终的 Jenkins 流水线。

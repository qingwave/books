# 前言

欢迎来到*Kubernetes编程*，感谢您选择与我们共度时光。在我们深入研究之前，让我们快速获得一些管理和组织的东西。我们也将分享编写本书的动机。

# 谁应该读这本书

您是开发人员的云端本地人，或AppOps或命名空间管理员希望从Kubernetes中获得最大收益。香草设置不再适合您了，您可能已经了解了[扩展点](http://bit.ly/2XmoeKF)。好。你是在正确的地方。

# 我们为什么写这本书

自2015年初以来，我们两人一直在为Kubernetes做贡献，写作，教学和使用。我们为Kubernetes开发了工具和应用程序，并为Kubernetes开发了几次研讨会。在某些时候我们说，“为什么我们不写一本书？”这将允许更多的人，以他们自己的节奏异步地学习如何编程Kubernetes。我们在这里。我们希望您在阅读本书时能够像编写本书一样有趣。

# 生态系统

在事情的宏伟计划，Kubernetes生态系统仍处于早期阶段。虽然Kubernetes在2018年初确立了自己作为管理容器（及其生命周期）的行业标准，但仍然需要有关如何编写本机应用程序的良好实践。基础构建块（例如[`client-go`](http://bit.ly/2L5cUMu)，自定义资源和云原生编程语言）已到位。然而，大部分知识都是部落的，分散在人们的脑海中，分散在成千上万的Slack频道和StackOverflow答案中。

###### 注意

在撰写本文时，Kubernetes 1.15是最新的稳定版本。编译后的示例应该适用于较旧的版本（低至1.12），但我们将代码基于较新版本的库，对应于1.14。一些更高级的CRD功能需要运行1.13或1.14集群，第9章甚至1.15中的CRD转换。如果您无法访问最近的群集，强烈建议您使用本地工作站上的[Minikube](http://bit.ly/2WT3k1l)或[kind](https://kind.sigs.k8s.io/)。

# 您需要了解的技术

这个中级书籍需要对一些开发和系统管理概念的最小理解。在深入研究之前，您可能需要查看以下内容：

- 包管理

  该本书中的工具通常具有多个依赖项，您需要通过安装某些软件包来满足这些依赖项。因此，需要了解机器上的包管理系统。这可能是*容易*在Ubuntu / Debian的系统，*百胜*在CentOS / RHEL系统或*端口*或*BREW*在MacOS。无论是什么，请确保您知道如何安装，升级和删除软件包。

- 去

  Git已经确立了自己的标准分布式版本控制。如果您已经熟悉CVS和SVN但尚未使用Git，那么您应该这样做。Jon Loeliger和Matthew McCullough（O'Reilly）*使用Git*进行*版本控制*是一个很好的起点。与Git一起，[GitHub网站](http://github.com/)是您开始使用自己的托管存储库的绝佳资源。要了解GitHub，请查看[他们的培训产品](https://services.github.com/)和[相关的交互式教程](http://try.github.io/)。

- 走

  Kubernetes用[Go](http://golang.org/)写的。在过去的几年中，Go已成为许多初创公司和许多与系统相关的开源项目的首选新编程语言。这本书不是教你Go，但它告诉你如何用Go编程Kubernetes。您可以学习浏览各种不同的资源，从[Go网站](https://golang.org/doc)上的在线文档到博客文章，演讲和一些书籍。

# 本书中使用的约定

本书使用以下印刷约定：

- *斜体*

  表示新术语，URL，电子邮件地址，文件名和文件扩展名。

- `Constant width`

  用于程序列表，以及段落内部，用于引用程序元素，如变量或函数名称，数据库，数据类型，环境变量，语句和关键字。也用于命令和命令行输出。

- **Constant width bold**

  显示应由用户按字面输入的命令或其他文本。

- *Constant width italic*

  显示应使用用户提供的值替换的文本或由上下文确定的值。

###### 小费

此元素表示提示或建议。

###### 注意

该元素表示一般性说明。

###### 警告

此元素表示警告或警告。

# 使用代码示例

这个本书可以帮助您完成工作。你可以在GitHub的组织在整本书中使用的代码示例[本书](https://github.com/programming-kubernetes)。

通常，如果本书提供了示例代码，您可以在程序和文档中使用它。除非您复制了大部分代码，否则您无需与我们联系以获得许可。例如，编写使用本书中几个代码块的程序不需要许可。出售或分发O'Reilly书籍中的示例CD-ROM需要获得许可。通过引用本书并引用示例代码来回答问题不需要许可。将本书中的大量示例代码合并到产品文档中需要获得许可。

我们感谢，但不要求，归属。归属通常包括标题，作者，出版商和ISBN。例如：“ Michael Hausenblas和Stefan Schimanski（O'Reilly）的Kubernetes *编程*。版权所有2019 Michael Hausenblas和Stefan Schimanski。“

如果您认为您对代码示例的使用超出了合理使用范围或上述许可范围，请随时通过*permissions@oreilly.com*与我们联系。

本书中使用的Kubernetes清单，代码示例和其他脚本可通过[GitHub获得](https://github.com/programming-kubernetes)。您可以克隆这些存储库，转到相关的章节和配方，并按原样使用代码。

# O'Reilly在线学习

###### 注意

近40年来，[*O'Reilly Media*](http://oreilly.com/)提供技术和业务培训，知识和洞察力，帮助公司取得成功。

我们独特的专家和创新者网络通过书籍，文章，会议和在线学习平台分享他们的知识和专业知识。O'Reilly的在线学习平台为您提供按需访问实时培训课程，深入学习路径，交互式编码环境以及来自O'Reilly和200多家其他出版商的大量文本和视频。有关更多信息，请访问[*http://oreilly.com*](http://oreilly.com/)。

# 如何联系我们

请 向发布商提出有关本书的评论和问题：

- O'Reilly Media，Inc。
- 1005 Gravenstein Highway North
- Sebastopol，CA 95472
- 800-998-9938（在美国或加拿大）
- 707-829-0515（国际或当地）
- 707-829-0104（传真）

我们有一本本书的网页，其中列出勘误表，示例和任何其他信息。您可以通过[*https://oreil.ly/pr-kubernetes*](https://oreil.ly/pr-kubernetes)访问此页面。

请发送电子邮件至*bookquestions@oreilly.com*发表评论或询问有关本书的技术问题。

有关我们的书籍，课程，会议和新闻的更多信息，请访问我们的网站[*http://www.oreilly.com*](http://www.oreilly.com/)。

在Facebook上找到我们：[*http*](http://facebook.com/oreilly)：[*//facebook.com/oreilly*](http://facebook.com/oreilly)

在Twitter上关注我们：[*http*](http://twitter.com/oreillymedia)：[*//twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

在YouTube上观看我们：[*http*](http://www.youtube.com/oreillymedia)：[*//www.youtube.com/oreillymedia*](http://www.youtube.com/oreillymedia)

# 致谢

一个很大的“谢谢你！”向Kubernetes社区致敬，他们开发了这样一款出色的软件，并成为了一大群人 - 开放，善良，随时准备提供帮助。此外，我们非常感谢我们的技术评审员：Ahmed Belgana，Michael Gasch，Dimitris Gkanatsios，Mingding Han，Jess Males，MaxNeunhöffer，Ewout Prangsma和Adrien Trouillaud。您都提供了极具价值和可操作性的反馈，使本书对读者更具可读性和实用性。感谢您的时间和精力！

迈克尔想对他那令人敬畏和支持的家庭表示最深切的谢意：我邪恶的聪明而有趣的妻子Anneliese; 我们的孩子Saphira，Ranya和Iannis; 和我们几乎还是小狗的史努比。

Stefan感谢他的妻子Clelia，当他再次“在书上工作”的时候，他感到非常支持和鼓励。没有她，这本书就不会在这里。如果你在书中发现拼写错误，很可能他们很自豪地由两只猫Nino和Kira贡献。

最后但同样重要的是，两位作者都感谢O'Reilly团队，特别是弗吉尼亚威尔逊，他们在编写本书的过程中引导我们，确保我们按时交付并保证质量。
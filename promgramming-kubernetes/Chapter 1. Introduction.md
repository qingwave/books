# 第1章简介

Kubernetes编程对不同的人来说意味着不同的东西。在本章中，我们将首先确定本书的范围和重点。此外，我们将分享关于我们正在运营的环境以及您需要提供什么样的假设（我们假定了一系列的场景，同时结合了在该场景环境下执行相关的操作，希望您借此了解背后的实现机制，进而能从本书中有所收获。我们将进一步描述我们认为的Kubernetes编程是什么，Kubernetes原生应用程序是什么，并通过查看具体示例，了解它们的特征。我们将讨论controller和operator的基础知识，以及基于事件驱动的Kubernetes的控制平面的原理是怎样的。准备好了么？让我们开始吧。

# Kubernetes编程是什么？

我们假设您可以访问正在运行的Kubernetes集群，例如Amazon EKS，Microsoft AKS，Google GKE或OpenShift其中一个产品。

###### 提示

您将花费大量时间在笔记本电脑或桌面环境中进行*本地*开发; 也就是说，您正在开发的Kubernetes集群是本地的，而不是位于云平台或您的数据中心。在本地开发时，您可以使用多种选项。根据您的操作系统和个人喜好，您可以选择一个（或多个）以下解决方案在本地运行Kubernetes：[kind](https://kind.sigs.k8s.io/)，[k3d](http://bit.ly/2Ja1LaH)或[Docker Desktop](https://dockr.ly/2PTJVLL)。[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#idm46336877072840)

我们假设您是Go程序员 - 也就是说，您具有Go编程语言的经验或至少基本熟悉。如果您对kubernetes或者Go编程语言不熟悉，现在正是一个熟悉起来的好机会：对于Go，我们推荐Alan AA Donovan和Brian W. Kernighan（Addison-Wesley）[*的The Go Programming Language*](https://www.gopl.io/)和Katherine的[*Concurrency in Go*](http://bit.ly/2tdCt5j) Cox-Buday（O’Reilly）。对于 Kubernetes，查看以下一本或多本书：

- [*Kubernetes in Action*](http://bit.ly/2Tb8Ydo) by Marko Lukša (Manning)

- [*Kubernetes: Up and Running*, 2nd Edition](https://oreil.ly/2SaANU4) by Kelsey Hightower et al. (O’Reilly)

- [*Cloud Native DevOps with Kubernetes*](https://oreil.ly/2BaE1iq) by John Arundel and Justin Domingus (O’Reilly)

- [*Managing Kubernetes*](https://oreil.ly/2wtHcAm) by Brendan Burns and Craig Tracey (O’Reilly)

- [*Kubernetes Cookbook*](http://bit.ly/2FTgJzk) by Sébastien Goasguen and Michael Hausenblas (O’Reilly)

  

###### 注意

为什么我们专注于Go中的Kubernetes编程？好吧，可以从一个类比中让您更好的理解：Unix是用C编程语言编写的，如果你想为Unix编写应用程序或工具，你会默认选择C语言.另外，为了扩展和定制Unix - 即使你是使用C以外的语言 - 您至少需要能够阅读C语言程序代码.

现在，Kubernetes和许多相关的云原生技术，从容器运行时到监控，如Prometheus，都是用Go编写的。我们相信大多数原生应用程序都是基于Go的，因此我们在本书中专注于它。如果您更喜欢其他语言，请关注[kubernetes-client](http://bit.ly/2xfSrfT) GitHub组织。这里有各种编程语言实现的客户端，您可以选择自己最喜欢的，同时这些客户端也在不断的扩充中。

在本书的上下文中，我们所谓的“Kubernetes编程”是指：您将开发一个Kubernetes原生应用程序，它直接与API服务器交互，查询资源状态和/或更新其状态。这个Kubernetes原生的应用程序并不是现成的，相比于WordPress或Rocket Chat或您喜欢的企业CRM系统，这些应用通常称为*商用现成*（COTS）应用程序。此外，在[第7章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#ch_shipping)，我们并没有像现实中那样过多的关注运营问题，而是主要关注开发和测试阶段。所以，简而言之，这本书是主要聚焦于开发真正的云原生应用程序。[图1-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#apps-on-kube)可能会帮助您更好地理解它。

![在Kubernetes上运行的不同类型的应用程序](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0101.png)

###### 图1-1。在Kubernetes上运行的不同类型的应用程序

如你所见，我们可以采用不同的方式在Kubernetes上运行应用程序：

1. 拿一个像Rocket Chat这样的COTS并在Kubernetes上运行它。应用程序本身并不知道它在Kubernetes上运行，通常也不需要知道。Kubernetes控制应用程序的生命周期 - 查找节点以运行，提取图像，启动容器，执行运行状况检查，装载卷等等 - 诸如此类操作。
2. 拿一个定制的应用程序，你从头开始编写的东西，无论是否考虑到Kubernetes作为运行时环境，并在Kubernetes上运行它。其他操作类似于上述COTS的操作方式。
3. 我们在本书中关注的案例是云原生或Kubernetes原生应用程序，它确定在Kubernetes上运行并在某种程度上利用Kubernetes API和资源。

针对Kubernetes API进行开发有以下优点：一方面您获得了可移植性，因为您的应用程序现在可以在任何环境中运行（从内部部署到任何公共云提供商），另一方面您可以从Kubernetes提供的简洁的声明机制中受益。

让我们现在看一个具体的例子。

# 一个激动人心的例子

为了演示了Kubernetes原生应用程序的强大功能，让我们假设您要实现`at`程序- 这个程序会在给定时间[安排执行命令](http://bit.ly/2L4VqzU)。

我们把它称做[`cnat`](http://bit.ly/2RpHhON)或云原生`at`，它的工作原理如下。假设你想在2019年7月3日凌晨2点执行命令输出一句话，如`echo "Kubernetes native rocks!"`。我们可以定义下边的声明式yaml文件：

```
$ cat cnat-rocks-example.yaml
apiVersion：cnat.programming-kubernetes.info/v1alpha1
kind：At
metadata：
  name：cnrex
spec：
  schedule： "2019-07-03T02:00:00Z"
  containers：
  - name：shell
    image：centos：7
    command:
    - "bin/bash"
    - "-c"
    - echo "Kubernetes native rocks!"

$ kubectl apply -f cnat-rocks-example.yaml
cnat.programming-kubernetes.info/cnrex created
```

在声明式yaml文件的背后，涉及以下组件：

- 一个自定义资源`cnat.programming-kubernetes.info/cnrex`，用来描述执行计划。
- 一个控制器负责在正确的时间执行调度的命令。

此外，实现一个`kubectl`的命令插件也会很有用，可以执行像`kubectl` `at` `"02:00 Jul 3"` `echo``"Kubernetes native rocks!"` 这样的命令在命令行就可以进行简单处理。不过本书中我们暂不详细介绍如何写命令行插件，如您感兴趣可以参考[Kubernetes文档获取说明](http://bit.ly/2J1dPuN)。

在整本书中，我们将使用此示例来讨论Kubernetes的各个方面，其内部工作机制以及如何扩展它。

在第[8](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)和[9](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#ch_advanced-topics)章我们提供了一些进阶的例子，我们将在集群中模拟一个比萨饼店，包含其相关的比萨和配料对象。有关详细信息，请参阅[“示例：比萨餐厅”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-example)。

# 扩展模式

Kubernetes是一个功能强大且内在可扩展的系统。通常，有多种方法可以自定义和/或扩展Kubernetes：对于控制平面组件（如`kubelet`或者Kubernetes API服务器）可以使用[配置文件和](http://bit.ly/2KteqbA)的参数变量，以及通过许多已定义的扩展点：

- 所谓的[云提供商](http://bit.ly/2FpHInw)，在以往是属于Kubernetes的controller manager核心项目中支持的一部分。从1.11开始，Kubernetes通过提供[自定义`cloud-controller-manager`进程](http://bit.ly/2WWlcxk)与云集成，使在核心项目之外开发云管理控制器成为可能。云提供商允许使用特定于云提供商的工具，如负载均衡器或虚拟机（VM）。
- `kubelet`用于[网络](http://bit.ly/2L1tPzm)，[设备](http://bit.ly/2XthLgM)（如GPU），[存储](http://bit.ly/2x7Unaa)和[容器运行时的](http://bit.ly/2Zzh1Eq)可执行程序插件。
- `kubectl` [可执行程序插件](http://bit.ly/2FmH7mu)。
- API服务器中的访问扩展，例如[带有webhooks](http://bit.ly/2DwR2Y3)的[动态准入控制](http://bit.ly/2DwR2Y3)（参见[第9章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#ch_advanced-topics)）。
- 自定义资源（请参阅[第4章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)）和自定义控制器; 请参阅以下部分。
- 自定义API服务器（请参阅[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)）。
- 调度程序扩展，例如使用[webhook](http://bit.ly/2xcg4FL)来实现您自己的调度决策。
- 使用webhook进行[身份验证](http://bit.ly/2Oh6DPS)。

在本书的上下文中，我们将重点关注自定义资源，控制器，webhook和自定义API服务器，以及Kubernetes [扩展模式](http://bit.ly/2L2SJ1C)。如果您对其他扩展点感兴趣，例如存储或网络插件，请查看[官方文档](http://bit.ly/2Y0L1J9)。

现在您已经对Kubernetes扩展模式和本书的范围有了基本的了解，让我们转到Kubernetes控制平面的核心，看看我们如何扩展它。

# 控制器和Operator

在 在本节中，您将了解Kubernetes中的控制器（controller）和Operator以及它们的工作原理。

根据[Kubernetes术语表](http://bit.ly/2IWGlxz)，*控制器*实现一个控制循环，通过API服务器观察集群的共享状态，并进行更改以尝试将当前状态调整至所需状态。

在我们深入了解控制器的内部工作之前，让我们来定义我们的术语：

- 控制器可以对核心资源（例如deployment或service）执行操作，这些资源通常是控制平面中[Kubernetes控制器管理器的](http://bit.ly/2WUAEVy)一部分，或者可以监视和操作用户定义的自定义资源。
- Operator是在控制器的基础上加入一些运维知识，例如应用程序生命周期管理，在[第4章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)定义的自定义资源会有介绍。

当然，鉴于后者的概念是基于前者，我们首先考虑控制器，然后再深入讨论Operator。

## 控制循环

在 一般来说，控制循环如下所示：

1. 获取资源状态，最好是在事件驱动模式下（采用watch方式，如[第3章所述](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#ch_client-go)）。有关详细信息，请参阅[“事件”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#controller-events)和[“边缘与电平驱动的触发器 Edge-Versus Level-Driven Triggers”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#edge-vs-level)。
2. 更改集群或集群外部的对象的状态。例如，启动pod，创建endpoint或调用云端API。有关详细信息，请参阅[“更改集群内或外部对象”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#controller-change-world)。
3. 通过与API服务器交互，更新步骤1中资源的状态到`etcd`中。有关详细信息，请参阅[“乐观并发”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#optimistic-concurrency)。
4. 重复循环; 回到第1步。

无论您的控制器有多复杂或简单，都将遵循这三个步骤 - 读取资源状态˃更改业务相关的资源状态（集群内或集群外部）˃更新资源状态。让我们深入探讨一下如何在Kubernetes控制器中实现这些步骤。控制循环[如图1-2所示](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#controller-loop-overview)，在这个典型的流程里，控制器的主循环位于中间。该主循环在控制器进程内持续运行。此过程通常在集群中的pod中运行。

![Kubernetes控制循环](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0102.png)

###### 图1-2。Kubernetes控制循环

从架构的角度来看，控制器通常使用以下数据结构（如[第3章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#ch_client-go)详细讨论）：

- Informers

  Informers以可持续和可扩展的方式watch所关注资源的状态。它们还实现了重新同步机制（请参阅[“Informers和Caching”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#informers)以获取详细信息），通过执行周期性状态协调操作，来确保集群状态和缓存在内存中的假定状态间的一致性（例如，不会由于错误或网络问题发生状态漂移/不一致 ）。

- 工作队列 Work queues

  基本上，一个工作队列可被事件处理程序用于处理状态更改的排队，同时工作队列辅助实现了重试功能。在`client-go`包中是通过所提供的[*workqueue* package](http://bit.ly/2x7zyeK)（见[“工作队列”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#workqueue)）来实现的。在更新业务相关的资源状态或写入当前所监控资源的状态（循环中的步骤2和3）时，如果出现错误，或者因其他原因我们不得不在一段时间后重新协调资源状态时，都可以将资源重新排队。

有关Kubernetes作为声明引擎和状态转换的更正式的讨论，请阅读Andrew Chen和Dominik Tornow [撰写的“Kubernetes的力学”](http://bit.ly/2IV2lcb)。

现在让我们仔细看看控制循环，从Kubernetes事件驱动架构开始。

## 事件

Kubernetes控制平面大量使用事件和松耦合组件的原理。其他分布式系统使用远程过程调用（RPC）以触发行为。这与Kubernetes是不同的。Kubernetes控制器watch API服务器中Kubernetes对象的更改：添加，更新和删除。当发生这样的事件时，控制器执行其业务逻辑。

例如，在通过部署一个deployment来启动一个pod的过程中，涉及到许多控制器和其他控制平面组件一起协调工作：

1. 当deployment控制器（位于`kube-controller-manager`组件内）发现（通过deployment informer）用户创建了一个deployment。deployment控制器将在自己的业务逻辑中创建replica set。
2. 当Replica set控制器（同样位于`kube-controller-manager`组件内）发现（通过replica set informer）有新的replica set被创建，随后在自己的运行业务逻辑中，将创建出pod对象。
3. 调度程序（`kube-scheduler`可执行文件） - 它也是一个控制器 - 当他发现pod中`spec.nodeName`字段为空（通过pod informer）。它在自己的业务逻辑中会将pod放入其调度队列中。
4. 与此同时- `kubelet`另一个控制器 - 发现新的pod（通过其pod informer）。但是新pod的`spec.nodeName`字段为空，因此与`kubelet`节点名称不匹配。它忽略了pod并重新进入休眠状态（直到下一个事件）。
5. 调度程序将pod从工作队列中取出，选取一个具有足够可用资源的节点名称，更新到pod中的`spec.nodeName`字段，并将其写入API服务器，以此声明Pod将被调度到所选节点。
6. `kubelet`将被Pod更新事件再次唤醒。它再次将其`spec.nodeName`与自己的节点名称进行比较。此时名称匹配，`kubelet`将启动Pod中定义的所有容器，并将相关信息写入容器状态中，同时返回API服务器来报告容器已启动。
7. Replica set控制器发现Pod信息又变化，但不会触发任何操作。
8. Pod可能会出于某些原因被终止。这时`kubelet`会发现，它通过API服务器交互获取Pod实例对象，设置Pod status中的“terminated”condition对应的状态值，并把它写回API服务器。
9. Replica set控制器注意到已终止的pod，控制器必须创建一个新的Pod替换此pod，来满足声明中的副本数的要求。因此它删除API服务器上已终止的pod并创建一个新的pod。
10. 等等。

如您所见，许多独立的控制循环之间仅通过API服务器上的对象更改以及这些更改通过informers触发的事件进行通信。

这些事件是以watch方式从API服务器发送到控制器内的informers对象中（参见[“watch”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go-watches)） - **<u>也就是说资源的监控是流方式实现的</u>**。所有这些对用户来说几乎是不可见的。甚至API服务器审计机制也不会使这些事件可见; 只有对象更新是可见的。可以通过在控制器中输出日志，当事件触发时就可以观察到。

##### 事件与事件对象

Watch事件和Kubernetes中的事件对象是两件不同的事情：

- Watch事件通过API服务器和控制器之间的HTTP流连接发送，驱动informers来实现。
- 事件对象是类似于pods，deployments或services的资源，具有特殊属性，它具有一小时的生存时间，然后自动清除`etcd`。

`Event Object`事件对象仅仅是用户可见的日志记录机制。许多控制器创建这些事件，以便将其业务逻辑的各个方面传达给用户。例如，`kubelet`报告pod的生命周期事件（即，当容器启动，重新启动和终止时）。

您可以通过`kubectl`列出自己所使用的集群中发生的第二类事件。使用以下命令，您可以看到`kube-system`命名空间中发生了什么：

```
$ kubectl -n kube-system get events
LAST SEEN   FIRST SEEN   COUNT  NAME                                              KIND
3m          3m           1      kube-controller-manager-master.15932b6faba8e5ad   Pod
3m          3m           1      kube-apiserver-master.15932b6fa3f3fbbc            Pod
3m          3m           1      etcd-master.15932b6fa8a9a776                      Pod
…
2m          3m           2      weave-net-7nvnf.15932b73e61f5bc6                  Pod
2m          3m           2      weave-net-7nvnf.15932b73efeec0b3                  Pod
2m          3m           2      weave-net-7nvnf.15932b73e8f7d318                  Pod
```

如果如果您想了解有关事件的更多信息，请阅读Michael Gasch的博客文章[“Events,the DNA of Kubernetes”](http://bit.ly/2MZwbl6)，在那里他提供了更多背后机制和示例。

## Edge-Driven触发与Level-Driven触发

让我们先从更高层次抽象地看看我们如何在控制器中实现业务逻辑，以及为什么Kubernetes选择使用事件（即状态变化）来驱动其逻辑。

有两个原则检测状态变化（事件本身）：

- Edge-Driven触发

   在某个时间点状态发生了改变，触发处理程序执行 - 例如，从无pod到pod运行。

- Level-Driven触发

  定期检查状态，如果满足某些条件（例如，pod运行），则触发处理程序。

后者是一种轮询方式。对于对象数量非常大时它可能会有一些时延，因为对象状态从状态变更的一刻起到被控制器更改之间的延迟取决于轮询的间隔以及API服务器的应答速度。如[“Events 事件”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#controller-events)所述，涉及许多异步控制器，结果是需要很长时间来实现用户期望的系统。

对于许多对象，前一种状态监测的效率更高。延迟主要取决于控制器处理事件中的工作线程数。因此，Kubernetes基于事件（即边缘驱动的触发器）。

在Kubernetes控制平面中，许多组件在API服务器上更改对象，每次更改都会产生一个事件（即一个边缘）。我们将这些组件称为*事件源*或*事件生成器*。另一方面，在控制器的上下文中，我们对消费事件感兴趣 - 即何时以及如何对事件做出反应（通过informers）。

在分布式系统中，有许多actor并行运行，并且事件可能以任何顺序异步进入。当我们有一个错误的控制器逻辑，一些稍微错误的状态机制或外部服务失败时，在这些场景中我们很容易丢失事件，因为状态转换被打断了，我们不能闭环的处理它们。因此，我们必须深入研究如何应对这些错误。

在[图1-3中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#edge-vs-level-overview)您可以看到不同的工作策略：

1. 一个仅使用edge-driven逻辑的示例，其中可能错过第二次状态改变。
2. 一个edge-triggered逻辑的示例，它在处理事件时始终获得最新状态（即level-driven）。换句话说，它的处理逻辑是edge-triggered但是level-driven。
3. 具有附加重新同步的edge-triggered，level-driven逻辑的示例。

![触发选项（边缘与级别）](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0103.png)

###### 图1-3。触发选项（边缘驱动与水平驱动）

策略1无法很好地应对错过的事件，无论是因为破坏的网络使其丢失事件，还是因为控制器本身存在错误或某些外部云API已关闭。想象一下，replica set控制器只有在终止时才会替换pod。缺少事件意味着replica set将始终以较少的pod运行，因为缺少事件代表无法触发该事件对应的协调操作，进而事件对象的状态也无法达到预期。

策略2在收到另一个事件时从这些问题中恢复，因为它基于集群中的最新状态实现其逻辑。对于replica set控制器，它始终将指定的副本计数与群集中正在运行的pod进行比较。当它丢失事件时，它将在下次收到pod更新时替换所有丢失的pod。

策略3增加了持续地重新同步机制（例如，每五分钟）。如果没有pod事件进入，它将至少每五分钟协调一次，即使应用程序运行非常稳定并且不会导致许多pod事件。

鉴于单纯Edge-Driven触发器的不足，Kubernetes控制器通常实施第三种策略。

如果你想了解更多关于触发器的起源以及在Kubernetes中使用协调操作进行level triggering这一设计的动机，请阅读James Bowes的文章[“Level Triggering and Reconciliation in Kubernetes”](http://bit.ly/2FmLLAW)。

总结一下，我们讨论了检测外部变化并对其作出反应的不同抽象方法。[图1-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#controller-loop-overview)控制循环的下一步是更改业务相关的对象，可以是集群内对象也可能是按照规范更改外部对象。我们现在来看看。

## 更改集群对象或外部对象

在这个阶段，控制器改变它正在watch的对象的状态。例如，[controller manager](http://bit.ly/2WUAEVy)中的`ReplicaSet`[控制器](http://bit.ly/2WUAEVy)正在watch pod资源。在每个事件（edge-triggerd）上，它将观察其pod的当前状态，并将其与所需状态进行比较（这里是根据pod资源的状态信息作为依据进行协调操作，所以是level-driven的，反之可以理解，edge-driven方式的触发条件不是看状态信息来的，而是由具体事件触发的）。

由于实际场景中更改资源状态的具体行为与特定领域或任务相关的，因此在业务逻辑层面我们几乎无法提供示例。不过，我们可以继续从ReplicaSet`之前介绍过的控制器来演示。`ReplicaSet`s用于管理deployments部署，相应控制器的底线是：维护用户定义数量的相同pod副本。也就是说，如果pods数量少于用户指定的pods数（例如，因为pods已经死亡或者副本值已经增加），控制器将启动新pods。但是，如果有太多的pod，它会选择一些终止。控制器的整个业务逻辑可参考[*replica_set.go*包](http://bit.ly/2L4eKxa)，以下摘录的Go代码为状态改变的相关操作：

```go
// manageReplicas checks and updates replicas for the given ReplicaSet.
// It does NOT modify <filteredPods>.
// It will requeue the replica set in case of an error while creating/deleting pods.
func (rsc *ReplicaSetController) manageReplicas(
	filteredPods []*v1.Pod, rs *apps.ReplicaSet,
) error {
    diff := len(filteredPods) - int(*(rs.Spec.Replicas))
    rsKey, err := controller.KeyFunc(rs)
    if err != nil {
        utilruntime.HandleError(
        	fmt.Errorf("Couldn't get key for %v %#v: %v", rsc.Kind, rs, err),
        )
        return nil
    }
    if diff < 0 {
        diff *= -1
        if diff > rsc.burstReplicas {
            diff = rsc.burstReplicas
        }
        rsc.expectations.ExpectCreations(rsKey, diff)
        klog.V(2).Infof("Too few replicas for %v %s/%s, need %d, creating %d",
        	rsc.Kind, rs.Namespace, rs.Name, *(rs.Spec.Replicas), diff,
        )
        successfulCreations, err := slowStartBatch(
        	diff,
        	controller.SlowStartInitialBatchSize,
        	func() error {
        		ref := metav1.NewControllerRef(rs, rsc.GroupVersionKind)
                err := rsc.podControl.CreatePodsWithControllerRef(
            	    rs.Namespace, &rs.Spec.Template, rs, ref,
                )
                if err != nil && errors.IsTimeout(err) {
                	return nil
                }
                return err
            },
        )
        if skippedPods := diff - successfulCreations; skippedPods > 0 {
            klog.V(2).Infof("Slow-start failure. Skipping creation of %d pods," +
            	" decrementing expectations for %v %v/%v",
            	skippedPods, rsc.Kind, rs.Namespace, rs.Name,
            )
            for i := 0; i < skippedPods; i++ {
                rsc.expectations.CreationObserved(rsKey)
            }
        }
        return err
    } else if diff > 0 {
        if diff > rsc.burstReplicas {
            diff = rsc.burstReplicas
        }
        klog.V(2).Infof("Too many replicas for %v %s/%s, need %d, deleting %d",
        	rsc.Kind, rs.Namespace, rs.Name, *(rs.Spec.Replicas), diff,
        )

        podsToDelete := getPodsToDelete(filteredPods, diff)
        rsc.expectations.ExpectDeletions(rsKey, getPodKeys(podsToDelete))
        errCh := make(chan error, diff)
        var wg sync.WaitGroup
        wg.Add(diff)
        for _, pod := range podsToDelete {
            go func(targetPod *v1.Pod) {
                defer wg.Done()
                if err := rsc.podControl.DeletePod(
                	rs.Namespace,
                	targetPod.Name,
                	rs,
                ); err != nil {
                    podKey := controller.PodKey(targetPod)
                    klog.V(2).Infof("Failed to delete %v, decrementing " +
                    	"expectations for %v %s/%s",
                    	podKey, rsc.Kind, rs.Namespace, rs.Name,
                    )
                    rsc.expectations.DeletionObserved(rsKey, podKey)
                    errCh <- err
                }
            }(pod)
        }
        wg.Wait()

        select {
        case err := <-errCh:
            if err != nil {
                return err
            }
        default:
        }
    }
    return nil
}
```

您可以看到控制器在此行代码处 `diff` `:= len(filteredPods) - int(*(rs.Spec.Replicas))`计算行当前replicaset对象的期望对象状态之间的差异，然后根据具体结果实现两种逻辑：

- `diff` `<` `0`：副本太少; 必须创建更多的pod。
- `diff` `>` `0`：副本太多; 必须删除pod。

它还实施了一种策略，在`getPodsToDelete`方法中，用来选择将被删除的pods 。

但是，更改资源状态并不一定意味着资源本身必须是Kubernetes集群的一部分。换句话说，控制器可以改变位于Kubernetes之外的资源的状态，例如云存储服务。例如，[AWS Service Operator](http://bit.ly/2ItJcif)允许您管理AWS资源。除此之外，它还允许您管理S3存储桶 - 即，S3控制器正在监控存在于Kubernetes之外的资源（S3存储桶），状态更改反映了其生命周期中的具体阶段：创建了一个S3存储桶，在某些时候删除。

这应该说对您使用自定义控制器是给出了一个范例，您不仅可以管理核心资源（如pod）和自定义资源（如我们的`cnat`示例），还可以计算或存储Kubernetes之外的资源。这使控制器具有非常灵活和强大的集成机制，提供了跨平台和环境使用资源的统一方法。

## 乐观并发

在[“控制循环”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#controller-loop)，我们 在步骤3中讨论了控制器 - 根据规则，在更新集群对象和（或）外部世界之后将结果写入步骤1中触发控制器执行的那个资源的状态中。

这和其他任何写入操作一样（步骤2中）都可能出错。在分布式系统中，此控制器可能只是更新资源的众多控制器之一。由于写冲突，并发写入可能会失败。

为了更好地了解正在发生的事情，让我们退一步看看[图1-4](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#scheduling-archs)。[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#idm46336867946120)

![调度分布式系统中的体系结构](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0104.png)

###### 图1-4。调度分布式系统中的体系结构

这里定义了Omega的并行调度器架构，如下所示：

> 我们的解决方案是围绕共享状态构建的新并行调度程序体系结构，使用无锁的乐观并发控制来实现可扩展性和性能可伸缩性。这种架构正在谷歌的下一代集群管理系统Omega中使用。

Kubernetes继承了[Borg](http://bit.ly/2XNSv5p)的许多特性和经验教训，而这个特定的事务控制平面功能来自Omega：为了在没有锁的情况下执行并发操作，Kubernetes API服务器使用乐观并发。

简而言之，这意味着如果API服务器检测到并发写入尝试，它将拒绝后两次写入操作。然后由客户端（控制器，调度程序`kubectl`等）来处理冲突并可能重试写操作。

以下演示了Kubernetes中乐观并发的概念：

```go
var err error
 forretries：=0 ;retries <10 ;retries ++ {
    foo，err =client.Get ("foo"，metav1.GetOptions {})
    if err != nil {
        break
    }

    //更新foo资源或其他外部资源

    _，err =client.Update (foo )
    if err != nil && errors.IsConflict(err) {
        continue
    } else if err != nil{
        break
    }
}
```

这段代码展示了一个重试逻辑，在每次迭代中获取最新的`foo`对象，然后尝试更新外部和（或）`foo`状态以达到期望的`foo`状态。在`Update`调用之前完成的更改是乐观的。

`foo`来自`client.Get`调用的返回，该对象包含一个*资源版本*属性（是`ObjectMeta`结构的一部分- 请参阅[“ObjectMeta”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#ObjectMeta)以获取详细信息），它将在之后的写操作中被etcd识别，如果在我们调用`client.Update`更新操作的同时，集群中有另外一个操作更新了`foo`对象（意味着另一个程序通过client.Get取得了和我们这里resource version 相同的对象，但是先于我们对该对象做了更新）。如果是这种情况，我们的重试循环将获得*resource version冲突错误*。这意味着乐观并发逻辑失败。换句话说，`client.Update`调用也是乐观无锁的的。

###### 注意

Resource version实际上是`etcd`键/值对的版本version。每个对象的资源版本是Kubernetes中包含整数的字符串。这个整数直接来自`etcd`。`etcd`维护一个计数器，每次修改一个键（保存对象的序列化）的值时，该计数器都会增加。

在整个API machinery代码中，资源版本resource version（或多或少因此）作为任意字符串来处理，但在其上有一些排序。存储整数只是当前`etcd`存储后端的实现细节。

让我们看一个具体的例子。想象一下，您的客户端不是集群中唯一会修改pod的程序。有另外一个程序，即`kubelet`不断修改某些字段，因为容器不断崩溃。现在你的控制器读取到pod对象的最新状态，如下所示：

```
kind: Pod
metadata:
  name: foo
  resourceVersion: 57
spec:
  ...
status:
  ...
```

现在假设控制器需要几秒钟来更新操作。七秒钟后，它尝试更新它读取的pod - 例如，它设置了一个注释。同时，`kubelet`已发现另一个容器重启并更新了pod的状态; 也就是说，`resourceVersion`已经增加到58。

控制器在更新请求中发送的对象具有`resourceVersion: 57`。API服务器尝试`etcd`使用该值设置pod 的密钥。`etcd`发现资源版本不匹配，并报告资源版本57与58冲突。更新失败。

此示例我们想说的是，对于您的控制器，您负责实施重试策略并处理乐观操作下可能的失败。您永远不知道还有谁可能在操纵状态，无论是其他自定义控制器还是核心控制器（如deployment控制器）。

其实质是：*资源版本的冲突错误在控制器中完全正常。需要在编程时考虑到他们并优雅地处理*。

重要的是要指出乐观并发非常适合基于level-based逻辑，因为通过使用基于level-based的逻辑，您可以重新运行控制循环（请参阅[“Edge-Versus Level-Driven Triggers”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#edge-vs-level)）。当该循环的再次运行将可能自动撤消掉先前因为乐观锁失败场景下出现的问题，同时将尝试将业务相关的集群内部或外部对象的状态更新为最新状态。

让我们继续讨论自定义控制器的特定情况（以及自定义资源）：Operators。

## Operators

Operators作为Kubernetes中的一个概念，是由CoreOS于2016年提出。在他的开创性博客文章[“Introducing Operators: Putting Operational Knowledge into Software”](http://bit.ly/2ZC4Rui)[中](http://bit.ly/2ZC4Rui)，CoreOS公司CTO Brandon Philips将operators定义如下：

> 一个现场可靠性工程师（SRE）是一个通过编写软件来操作应用程序的人。他们是工程师，开发人员，知道如何专门为特定应用领域开发软件。由此产生的软件向原有的应用程序中加入了领域相关知识。
>
> […]
>
> 我们将这类新的软件称为Operators。Operator是一个特定于应用程序的控制器，它扩展了Kubernetes API，以代表Kubernetes用户创建，配置和管理复杂有状态应用程序的实例。它建立在基本的Kubernetes资源和控制器概念的基础上，但包括领域或特定于应用程序的知识，用来自动执行常见任务。

在本书的上下文中，我们将使用Philips描述的Operators，确切说，要求满足以下三个条件（参见[图1-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#operator-conceptual)）：

- 您希望将一些特定于相关领域的操作知识，赋予自动化执行。
- 这种操作知识的最佳实践是已知的并且可以明确 - 例如，在Cassandra Operator中，何时以及如何重新平衡节点，或者在service mesh的operator中，如何创建一条路由。
- 在operator的上下文中涉及的组件是：
  - 一组*自定义资源定义*（CRD），以及与某特定领域结合后产生的资源描述schema和自定义资源对象实例（CR）。
  - 自定义控制器，监控自定义资源（CR），可能还有核心资源。例如，自定义控制器可能会启动一个pod。

![运营商的概念](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0105.png)

###### 图1-5。operator的概念

Operator从2016年的概念性工作和原型设计到Red Hat（在2018年收购CoreOS并不断发展）的[OperatorHub.io](https://operatorhub.io/)的推出已经[走了很长一段路](http://bit.ly/2x5TSNw)。在[图1中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#operatorhub)可以看到[图1-6](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#operatorhub)的截图。 2019年中期，有大约17个operators，可供使用。

![OperatorHub.io屏幕截图](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0106.png)

###### 图1-6。OperatorHub.io截图

# 摘要

在第一章中，我们定义了本书的范围以及我们对您的期望。我们解释了在本书中我们所谓的Kubernetes编程，以及Kubernetes原生应用程序所想表达的含义。作为后续示例的准备，我们还提供了对控制器和operator的高级介绍。

所以，既然你已经知道了本书的内容以及如何从中获益，那么让我们深入探讨。在下一章中，我们将详细介绍Kubernetes API，API服务器的内部工作方式，以及如何使用命令行工具与API进行交互`curl`。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#idm46336877072840-marker)有关此主题的更多信息，请参阅Megan O'Keefe的[ “适用于MacOS的Kubernetes开发人员工作流程”](http://bit.ly/2WXfzu1)，*Medium*，2019年1月24日; 和Alex Ellis的博客文章[ “Be KinD to yourself”](http://bit.ly/2XkK9C1)，2018年12月14日。

[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#idm46336867946120-marker)来源：[ “Omega：适用于大型计算集群的灵活，可扩展的调度程序”](http://bit.ly/2PjYZ59)，作者：Malte Schwarzkopf等人，Google AI，2013。
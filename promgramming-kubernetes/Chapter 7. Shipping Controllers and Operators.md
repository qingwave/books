# 第7章发布控制器和Operators

现在您已经熟悉了自定义控制器的开发，接下来我们将讨论如何使您的自定义控制器和operators在生产中使用。在本章中，我们将讨论控制器和operator的操作运行方面，演示如何打包它们，引导您完成在生产中使用控制器的最佳实践，并确保您的这个扩展不会破坏Kubernetes集群的安全和性能。

# 生命周期管理和打包

在本节我们考虑operator的生命周期管理。也就是说，我们将讨论如何打包和发布您的控制器或operator，以及如何进行升级。当您准备将operator发送给用户时，您需要提供一种安装方法。为此，您需要打包相应的组件，例如YAML配置清单来描述控制器可执行程序如何部署（通常定义一个Kubernetes deployment），以及CRD清单和安全相关资源配置清单，例如service account和必要的RBAC权限。一旦您的目标用户开始使用运行某个版本的operator，您还需要有一个机制来升级它，考虑版本控制和潜在的平滑升级。

让我们从最简单的开始：打包和交付operator，以便用户可以直接安装它。

## 打包：挑战

Kubernetes通过在一个配置清单中声明对资源的需求，通常清单用YAML编写，用YAML文件来进行资源声明也有一些缺点。例如，在对容器化应用程序打包时，YAML清单是静态的，所有值都是固定的。这意味着，如果要更改[部署清单中](http://bit.ly/2WZ1uRD)的容器映像，则必须创建一个新YAML清单。

让我们看一个具体的例子。假设您在名为*mycontroller.yaml*的YAML清单中声明了以下Kubernetes deployment，用来安装用户的自定义控制器：

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: mycustomcontroller
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: customcontroller
    spec:
      containers:
      - name: thecontroller
        image: example/controller:0.1.0
        ports:
        - containerPort: 9999
        env:
        - name: REGION
          value: eu-west-1
```

想象一下，环境变量`REGION`定义了控制器的某些运行时属性，例如托管服务等其他服务的可用性。换句话说，虽然默认值`eu-west-1`可能是合适的，但用户应该可以根据自己的需要来覆盖它。

现在，由于YAML清单*mycontroller.yaml*本身是一个静态文件，其中所有值都是在编写时定义好的 （并且像`kubectl`这种客户端本身不支持对清单中的内容进行改变），那么如何满足用户想通过变量覆盖的方式替换清单中的值呢？例如前面的例子中，用户可以在运行时设置`REGION`值为`us-east-2`？

解决Kubernetes部署时，YAML清单静态值的限制，有一些模板化的工具可以选择（例如Helm）或是可以在执行时接收用户提供值或运行时属性的Kustomize。

## Helm

[Helm](https://helm.sh/)，自称是Kubernetes 的软件包管理工具，最初由Deis开发，现在是一个云原生计算基金会（[CNCF](https://www.cncf.io/)）项目，主要贡献者来自微软，谷歌和Bitnami（现在是VMware的一部分）。

Helm通过定义和使用所谓的charts，有效地参数化YAML清单，帮助您安装和升级Kubernetes应用程序。以下是[示例charts模板](http://bit.ly/2XmLk3R)的摘录：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "flagger.fullname" . }}
...
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ template "flagger.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ template "flagger.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      serviceAccountName: {{ template "flagger.serviceAccountName" . }}
      containers:
        - name: flagger
          securityContext:
            readOnlyRootFilesystem: true
            runAsUser: 10001
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

如您所见，变量写成`{{ ._Some.value.here_ }}`格式，与[Go模板](http://bit.ly/2N2Q3DW)一样。

安装chart，可以运行`helm install`命令。虽然Helm有多种查找和安装charts的方法，但其中最简单的方法是使用官方稳定的chart：

```shell
# get the latest list of charts:
$ helm repo update

# install MySQL:
$ helm install stable/mysql
Released smiling-penguin

# list running apps:
$ helm ls
NAME             VERSION   UPDATED                   STATUS    CHART
smiling-penguin  1         Wed Sep 28 12:59:46 2016  DEPLOYED  mysql-0.1.0

# remove it:
$ helm delete smiling-penguin
Removed smiling-penguin
```

为了打包控制器，您需要为它创建一个Helm chart并将其发布到某个地方，默认情况下将其发布到[Helm Hub](https://hub.helm.sh/)访问公共存储库，[如图7-1所示](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#helm-hub)。

![Helm Hub屏幕截图，显示公开的Helm图表](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0701.png)

###### 图7-1。Helm Hub屏幕截图显示了公开的Helm charts

关于进一步了解如何创建Helmchart，可阅读以下资源：

- Bitnami发布的一片文章 [“How to Create Your First Helm Chart”](http://bit.ly/2ZIlODJ)
- 如果要将图表保留在自己的组织中，请[使用S3作为Helm存储库”](http://bit.ly/2KzwLDY)。
- Helm官方文档：[“chart最佳实践指南”](http://bit.ly/31GbayW)。

Helm很受欢迎，部分原因是它的易用性。然而，一些人认为目前的Helm架构有一些[安全风险](http://bit.ly/2WXM5vZ)。好消息是社区正在积极致力于解决这些问题。

## Kustomize

[Kustomize](https://kustomize.io/) 遵循熟悉的Kubernetes API，提供一种声明性方法来配置Kubernetes清单文件。它于[2018年中期推出，](http://bit.ly/2L5Ec5f)现在是Kubernetes SIG CLI项目。

你可以本机[安装](http://bit.ly/2Y3JeCV)Kustomize，或者，如果你有一个较新的`kubectl`版本（1.14以上），它被[集成](http://bit.ly/2IEYqRG)在`kubectl`工具中通过`-k`命令启用。

Kustomize允许您对原始YAML清单文件自定义，而无需修改原始清单。这是如何做到的呢？我们假设您要打包`cnat`自定义控制器; 你定义了一个名为*kustomize.yaml*的文件，它看起来像：

```yaml
imageTags:
  - name: quay.io/programming-kubernetes/cnat-operator
    newTag: 0.1.0
resources:
- cnat-controller.yaml
```

现在，您可以将此应用于*cnat-controller.yaml*文件，使用以下内容：

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: cnat-controller
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: cnat
    spec:
      containers:
      - name: custom-controller
        image: quay.io/programming-kubernetes/cnat-operator
```

使用`kustomize build`（*cnat-controller.yaml*文件将保持不变！） - 然后输出：

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: cnat-controller
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: cnat
    spec:
      containers:
      - name: custom-controller
        image: quay.io/programming-kubernetes/cnat-operator:0.1.0
```

`kustomize build`可以自动将所有的[自定义](http://bit.ly/2LbCDTr)值替换，它的输出可以作为`kubectl apply`命令的输入，

有关Kustomize的更详细的演示以及使用方法，请查看以下资源：

- SébastienGoasguen的博文[“Configuring Kubernetes Applications with kustomize"](http://bit.ly/2JbgJOR).
- Kevin Davin的帖子[“Kustomize—The right way to do templating in Kubernetes”](http://bit.ly/2JpJgPm).
- 视频[“TGI Kubernetes 072：Kustomize和朋友们”](http://bit.ly/2XoHm6C)，在那里你可以看到Joe Beda如何使用它。

考虑到Kustomize在`kubectl`中的原生支持，越来越多的用户可能会采用它。请注意，虽然它解决了一些自配置清单的定制化问题，但生命周期管理的其他方面（如验证和升级）可能需要您将Kustomize与其他一些技术一起使用，比如Google的[CUE](http://bit.ly/32heAZl)等语言。

为了总结打包这个主题，让我们回顾一下大家使用的其他一些解决方案。

## 其他打包选项

一些常见的打包选择 - 以及[新兴](http://bit.ly/2X553FE)的选择：

- UNIX工具

  在为了实现对原Kubernetes清单值定制，你可以在shell脚本中使用一系列的命令行工具，如`sed`，`awk`或`jq`。这是一种比较流行的解决方案，至少在Helm出现之前，也可能是最广泛选择 - 因为它最大限度地减少了依赖性，并且在* nix 环境中相当便携。

- 传统的配置管理系统

  您可以使用任何传统的配置管理系统（例如Ansible，Puppet，Chef或Salt）来打包和交付您的operator。

- 云原生语言

  新一代所谓的[云原生编程语言](http://bit.ly/2Rwh5lu)，如Pulumi和Ballerina，提供对Kubernetes原生应用程序的打包和生命周期管理等。

- [ytt](https://get-ytt.io/)

  ytt是另外一个YAML模板工具，它本身就是Google配置语言[Starlark](http://bit.ly/2NaqoJh)的改版。它在YAML结构上进行语义操作，并侧重于可重用性。

- [Ksonnet](https://ksonnet.io/)

  一个 用于Kubernetes清单的配置管理工具，最初由Heptio（现在的VMware）开发，Ksonnet已被弃用，并且不再更新，因此使用它需要您自担风险。

阅读更多关于Jesse Suen的帖子[“Kubernetes配置管理状态：一个未解决的问题”中](http://bit.ly/2N9BkXM)的讨论。

现在我们已经讨论了常见的打包方式，让我们来看看打包和发布制器和operator的最佳实践。

## 打包的最佳实践

当您打包并发布operator时，请确保您了解以下最佳做法。无论您选择哪种机制（Helm，Kustomize，shell脚本等），这些都适用：

- 提供适当的访问控制权限：这意味着在最小权限的基础上为控制器定义专用service account以及RBAC权限; 有关详细信息，请参阅[“获得权限”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#crds-rbac)。
- 考虑自定义控制器的范围：它会在一个命名空间或多个命名空间中查看CR吗？查看Alex Ellis关于不同方法的利弊对比[Twitter对话](http://bit.ly/2ZHd5S7)。
- 测试并对控制器进行分析，以便了解其它的空间占用和可扩展性。例如，Red Hat已经将一组详细的要求与OperatorHub [贡献](http://bit.ly/2IEplx4)指南中的说明放在一起。
- 使确保CRD和控制器都有详细记录，最好使用[godoc.org](https://godoc.org/)上提供的内联文档和一组用法示例; 请参阅Banzai Cloud的[银行金库](http://bit.ly/2XtfPVB)运营商获取灵感。

## 生命周期管理

与打包和发布相比，生命周期管理包含的内容将更广泛和全面。其基本的出发点是从全流程出发进行考虑，从开发到发布再到升级，并尽可能自动化。CoreOS（后来被Red Hat收购）是这一领域的引领者：在operator中加入生命周期管理的相关逻辑。换句话说：operator中不仅包含自定义控制器，还需要给operator加入领域相关的运维逻辑，以便operator可以执行安装和升级。这是这个operator也就成为了一个业务领域专有的operator。实际上，operator 框架中已经包含了这部分（operatorSDK也来自于这个框架），在[“operator SDK”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#operator-sdk)也有所讨论（即：OLM，[operator生命周期管理器](http://bit.ly/2HIfDcR)）。

Jimmy Zelinskie是OLM背后的主导者，他曾[这样](http://bit.ly/2KEfoSu)说：

> OLM为operator开发者做了很多工作，但它也解决了一个很少有人想过的重要问题：operator作为Kubernetes中最主要的扩展方式，如何长期有效地对其管理？

简而言之，OLM提供了一种声明性的方式来安装和升级operator及其依赖项，再结合Helm等打包方案作为辅助。您可根据自己的实际需求决定是购买成熟的OLM解决方案还是为版本控制和升级挑战创建临时解决方案; 但是，在每个层面提前制定一些策略，这是必不可少的（例如，Red Hat对operator hub中的[认证过程](http://bit.ly/2KBlymy)，对于任何重要的部署方案都是必须提供的）。

# 生产中使用

在本节我们将回顾并讨论如何使您的自定义控制器和operator用于生产环境。以下是从较高的层次列出的清单：

- 使用Kubernetes [deployment](http://bit.ly/2q7vR7Y)或DaemonSet来管控您的自定义控制器，以便它们在失败时自动重启。
- 通过专用健康检查点以获得是否活跃和是否就绪状态的探测。这与上一步结合使您的操作加弹性可控。
- 考虑使用高可用方式，以确保即使您的控制器pod崩溃，另外一个Pod可以立刻接管。但需注意，状态同步要做好。
- 提供对资源的访问控制，例如service account和role，应用最小权限原则; 有关详细信息，请参阅[“获取权限”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#crds-rbac)。
- 考虑采用自动构建，自动化测试。[“自动构建和测试”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#crds-perf)中提供了更多提示。
- 主动进行监测和日志记录; 详细内容请参阅[“自定义控制器和可观察性”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#o11y)。

我们还建议您仔细阅读上述文章[“提高可用性的Kubernetes operator开发指南”](http://bit.ly/31P7rPC)以了解更多信息。

## 获得权限

您的自定义控制器是Kubernetes控制平面的一部分。它需要读取资源状态，在Kubernetes内部也可能在外部创建资源，并读取相关资源的状态。鉴于此，自定义控制器需要配置一系列正确的权限，通过基于角色的访问控制（RBAC）来进行相关设置。本节将介绍这方面的内容。

首先要做的事情：创建一个[专用的service account](http://bit.ly/2RwoSQp)给您的operator。换句话说：*不要*使用命名空间中名为default的service account。[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#idm46336853931352)

为了简化操作，您可以将必要的RBAC规则定义在一个ClusterRole实例中，再通过`RoleBinding`将其绑定到特定命名空间，从而在跨命名空间中重用ClusterRole角色，可参阅[使用RBAC授权](http://bit.ly/2LdVFsj)介绍。

使用最小授权原则，仅分配给控制器执行工作所需的最小权限。例如，如果控制器仅管理pod，则无需为其提供列出或创建deployment或service的权限。此外，一般来说保控制器不需要安装CRD和admission webhook。所以，控制器不应该有管理CRD和webhook的权限。

如[第6章所述](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#ch_operator-solutions)，用于创建自定义控制器的通用工具，通常提供了命令来生成RBAC规则。例如，Kubebuilder会根据对应的operator生成[以下](http://bit.ly/2RRCyFO) RBAC文件：

```shell
$ ls -al rbac/
total 40
drwx------  7 mhausenblas  staff   224 12 Apr 09:52 .
drwx------  7 mhausenblas  staff   224 12 Apr 09:55 ..
-rw-------  1 mhausenblas  staff   280 12 Apr 09:49 auth_proxy_role.yaml
-rw-------  1 mhausenblas  staff   257 12 Apr 09:49 auth_proxy_role_binding.yaml
-rw-------  1 mhausenblas  staff   449 12 Apr 09:49 auth_proxy_service.yaml
-rw-r--r--  1 mhausenblas  staff  1044 12 Apr 10:50 rbac_role.yaml
-rw-r--r--  1 mhausenblas  staff   287 12 Apr 10:50 rbac_role_binding.yaml
```

在*rbac_role.yaml中，*您可以看到关于自动生成的RBAC角色和角色绑定详细的设置：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: null
  name: manager-role
rules:
- apiGroups:
  - apps
  resources:
  - deployments
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups:
  - apps
  resources:
  - deployments/status
  verbs: ["get", "update", "patch"]
- apiGroups:
  - cnat.programming-kubernetes.info
  resources:
  - ats
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups:
  - cnat.programming-kubernetes.info
  resources:
  - ats/status
  verbs: ["get", "update", "patch"]
- apiGroups:
  - admissionregistration.k8s.io
  resources:
  - mutatingwebhookconfigurations
  - validatingwebhookconfigurations
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups:
  - ""
  resources:
  - secrets
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups:
  - ""
  resources:
  - services
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

看看Kubebuilder生成的这些属于`v1`版本的权限，你可能会有点吃惊。[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#idm46336853898344)最佳实践告诉我们，如果一个控制器没有充分的理由，它不应该做以下几点：

- 对代码中读取的资源赋予写权限。例如，如果你只是查看service和deployment，应该删除 verbs中的`create`，`update`，`patch`，和`delete`中方法。
- 读取所有secret; 也就是说，没有充分理由，只应该列出对你所需secret的访问，而不是全部。
- 写`MutatingWebhookConfigurations`或`ValidatingWebhookConfigurations`。这相当于访问集群中所有资源。
- 写`CustomResourceDefinition`s。虽然，在上表显示的集群角色中没有这样做，这里想建议：CRD创建应该由单独的进程完成，不要放在控制器中进行。
- 操作无关资源的/ status子资源（请参阅[“子](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-subresources)资源[”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-subresources)）。例如，此处的deployment不由`cnat`控制器管理，不应赋权对depolyment的操作。

当然，Kubebuilder实际上无法理解您的控制器代码的业务逻辑。因此，生成的RBAC规则过于宽松也就不足为奇了。我们建议按照前面的检查表仔细检查权限并将其减少到最小。

###### 警告

赋予读取系统中所有secret的权限相当于允许访问所有serviceaccount 的token信息。换句话即可以访问集群中所有的密码。赋予对`MutatingWebhookConfigurations`或`ValidatingWebhookConfigurations`的写访问权限，意味着允许您拦截和操作系统中的每个API请求。这将使得Kubernetes集群大门大开。两者显然都非常危险，并被是反常规的做法，因此尽可能要避免。

为了避免拥有太多的权力（即将访问权限限制为最小）可考虑使用[audit2rbac](http://bit.ly/2IDW1qm)。此工具使用审核日志生成一组适当的权限，从而实现更安全的设置和更少的麻烦。

从*rbac_role_binding.yaml*您可以了解到：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: null
  name: manager-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: manager-role
subjects:
- kind: ServiceAccount
  name: default
  namespace: system
```

有关RBAC及相关工具的更多最佳实践，请访问[*RBAC.dev*](https://rbac.dev/)，这是一个致力于Kubernetes RBAC的网站。现在让我们继续讨论自定义控制器的测试和性能考虑因素。

## 自动构建和测试

作为云原生下的最佳实践，应该考虑给自定义控制器增加自动化构建。这常被称为*持续构建*或*持续集成*（CI），包括单元测试，集成测试，构建容器镜像，甚至包括完整性或[冒烟](http://bit.ly/1Z9jXp5)测试。云原生计算基金会（CNCF）维护着许多可用的开源CI工具的[列表](http://bit.ly/2J2vy4L)。

在构建控制器时，应尽可能少地消耗计算资源，同时尽可能多地为客户端提供服务。每个CR，基于您定义的CRD，是客户端的代理。但是，你怎么知道它消耗了多少，它是否有内存泄漏，以及它的扩展性如何？

一旦自定义控制器的开发逐步稳定后，就可以开展大量测试。这些测试可包括但不限于以下内容：

- 对于性能方面的测试，可以使用[Kubernetes本身](http://bit.ly/2X556g8)以及[kboom](http://bit.ly/2Fuy4zU)工具，它提供有关扩展和资源占用空间的数据。
- 压力测试，请参考这些[在Kubernetes](http://bit.ly/2KBZmZc)中的测试，他们的目标是测试较长期的使用情况，从几个小时到几天，看相关资源是否有泄漏，如文件或内存。

作为最佳实践，这些测试应该是CI流水线的一部分。换句话说，从开始就设计好自动构建控制器，测试和打包的流程。对于一个具体的示例，您可查看MarkoMudrinić的一个不错的帖子[“Spawning Kubernetes Clusters in CI for Integration and E2E tests”](http://bit.ly/2FwN1RU).

接下来，我们将介绍快速排除故障方面的最佳实践：内置可监测性。

## 自定义控制器和可监测性

在本节中，我们将介绍自定义控制器的可监测性方面，特别是日志记录和监控。

### 日志

确保您提供了足够的日志信息以帮助进行[故障排除](http://bit.ly/2WXD85D)（特别是在生产中）。与其他容器化中的配置一样，日志信息发送到`stdout`，可以针对单个pod使用`kubectl logs`命令或以聚合方式来获取日志。对于聚合方式来说，在云提供商中会有特定的解决方案，例如Google Cloud中的Stackdriver或AWS中的CloudWatch，此外也可以使用定制化的解决方案，如采用Elasticsearch-Logstash-Kibana / Elasticsearch-Fluentd-Kibana等技术栈。关于这个主题更多内容可以参见SébastienGoasguen和Michael Hausenblas（O'Reilly）的[*Kubernetes Cookbook*](http://bit.ly/2FTgJzk)。

让我们看一下`cnat`自定义控制器日志的示例：

```json
{ "level":"info",
  "ts":1555063927.492718,
  "logger":"controller",
  "msg":"=== Reconciling At" }
{ "level":"info",
  "ts":1555063927.49283,
  "logger":"controller",
  "msg":"Phase: PENDING" }
{ "level":"info",
  "ts":1555063927.492857,
  "logger":"controller",
  "msg":"Checking schedule" }
{ "level":"info",
  "ts":1555063927.492915,
  "logger":"controller",
  "msg":"Schedule parsing done" }
```

如何记录日志：一般情况下，我们推荐[结构化记录](http://bit.ly/31TPRu3)和可调整的日志级别，至少包含`debug`和`info`。在Kubernetes代码库中广泛使用了两种方法，除非你有充分的理由，否则你应该考虑使用它们：

- 实现`logger`接口，例如，参考[*httplog.go*](http://bit.ly/2WWV54w)，每一个具体类型有一个实现（如：`respLogger`），包装了状态和错误信息。
- [`klog`](http://bit.ly/31OJxUu)， 一个fork自谷歌的`glog`的结构化日志库，Kubernetes内部项目几乎全部在使用它，很值得了解下。

需要记录什么内容：一般是要将业务逻辑中的详细信息记录到日志中。例如，定义在[*at_controller.go文件中*](http://bit.ly/2Fpo5Mi)的，由Operator SDK实现的cnat控制器的，使用如下方式定义日志对象：

```go
reqLogger := log.WithValues("namespace", request.Namespace, "at", request.Name)
```

然后在业务逻辑中，在`Reconcile(request reconcile.Request)`方法中：

```go
case cnatv1alpha1.PhasePending:
  reqLogger.Info("Phase: PENDING")
  // As long as we haven't executed the command yet, we need to check if it's
  // already time to act:
  reqLogger.Info("Checking schedule", "Target", instance.Spec.Schedule)
  // Check if it's already time to execute the command with a tolerance of
  // 2 seconds:
  d, err := timeUntilSchedule(instance.Spec.Schedule)
  if err != nil {
    reqLogger.Error(err, "Schedule parsing failure")
    // Error reading the schedule. Wait until it is fixed.
    return reconcile.Result{}, err
  }
  reqLogger.Info("Schedule parsing done", "Result", "diff", fmt.Sprintf("%v", d))
  if d > 0 {
    // Not yet time to execute the command, wait until the scheduled time
    return reconcile.Result{RequeueAfter: d}, nil
  }
  reqLogger.Info("It's time!", "Ready to execute", instance.Spec.Command)
  instance.Status.Phase = cnatv1alpha1.PhaseRunning
```

这段Go代码让您了解该记录哪些内容，特别是何时使用`reqLogger.Info`和`reqLogger.Error`。

了解了日志，让我们继续讨论相关主题：监控指标！

### 监控，仪表和审计

[Prometheus](https://prometheus.io/)是一个很好的开源，可用于容器环境的的监控解决方案，它可以跨环境（内部部署和公有云中）使用。显然对每个事件都发出警报是不切实际的，因此您可能想要考虑谁需要了解哪种事件。例如，您可以制定一个策略，即由基础架构管理员处理与节点相关的事件或与命名空间相关的事件，而Pod级别的事件进一步划分给命名空间管理员或开发人员来处理。在这种情况下，我们可以使用[Grafana](https://grafana.com/) 进行可视化的指标收集; 有关在Grafana中可视化的查看Prometheus指标的示例，请参[见图7-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#grafana_prometheus)，该示例取自[Prometheus文档](http://bit.ly/2Oi4YcA)。

如果您正在使用服务网格Service Mesh - 例如，基于[Envoy代理](https://envoy.com/)（如Istio或App Mesh）或Linkerd，那么在基础设施层面一般不需要配置或者可以通过最少的配置来实现。否则，您必须通过使用相应的库（例如[Prometheus](http://bit.ly/2xb2qmv)提供的库）来定义代码中的相关指标。您可能会对2019年初推出服务网状接口（[SMI](https://smi-spec.io/)）项目感兴趣，该项目旨在为基于CR和控制器的服务网格提供标准化接口。

![在Grafana中可视化的Prometheus指标](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0702.png)

###### 图7-2。在Grafana中可视化Prometheus指标

KubernetesAPI服务器提供的另一个有用功能是[审计](http://bit.ly/2O4WBkL)，它允许您记录影响集群的一系列活动。审计策略中提供了不同的策略，从无记录到记录事件元数据，请求正文和响应正文。您可以选择简单的日志后端实现或使用webhook与第三方系统集成。

# 总结

本章通过讨论控制器和operator在运维操作层面（包括打包，安全性和性能）的一些使用方法，使您更多的了解如何将operator用于生产环境。

到此为止，我们已经介绍了开发和使用自定义Kubernetes控制器和operator的基础知识，接下来我们将介绍另一种扩展Kubernetes的方法：开发自定义API服务器。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#idm46336853931352-marker)另请参阅Luc Juggery的帖子[ “Kubernetes Tips：Using a ServiceAccount”](http://bit.ly/2X0fjKK)，详细讨论service account的使用情况。

[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#idm46336853898344-marker)我们给Kubebuilder项目反馈了[问题748](http://bit.ly/2J7Qys4)。
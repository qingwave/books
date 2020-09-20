# 第2章 Kubernetes API基础知识

在本章中，我们将向您介绍Kubernetes API的基础知识。这包括深入了解API服务器的内部工作机制，API本身以及如何从命令行与API进行交互。我们将向您介绍Kubernetes API概念，例如资源和资源类型，以及分组和版本。

# API服务器

Kubernetes由一组具有不同角色的节点（集群中的机器）组成，[如图2-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#k8s-arch-overview)所示：主节点上的控制平面由API服务器，控制管理器和调度程序组成。API服务器是中央管理实体，也是与分布式存储组件`etcd`直接对话的唯一组件。

API服务器具有以下核心职责：

- 提供Kubernetes API服务。此API可以被集群内主要组件，工作节点以及Kubernetes原生应用程序在集群内部使用，也可由客户端程序外部调用，如：`kubectl`。
- 作为集群组件的代理（例如Kubernetes仪表板），或流式传输日志，服务端口或提供`kubectl exec`会话。

提供API意味着：

- 读取状态：获取单个对象，列出它们以及流式地更改对象
- 操作状态：创建，更新和删除对象

状态存储在`etcd`中。

![Kubernetes建筑概述](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0201.png)

###### 图2-1。Kubernetes架构预览

Kubernetes的核心是API服务器。但是API服务器如何工作？我们可先将API服务器视为黑盒子并仔细查看其HTTP接口，然后我们再深入其中讨论API服务器的内部工作原理。

## API服务器的HTTP接口

从客户的视角来看，API服务器公开了一个RESTful HTTP API，支持JSON或[*protocol buffer*](http://bit.ly/1HhFC5L)（简称*protobuf*）协议的payload，protobuf主要用于集群内部通信，用于提升性能。

API服务器HTTP接口使用以下[HTTP verbs](https://mzl.la/2WX21hL)（或HTTP方法）处理HTTP请求以查询和操作Kubernetes资源：

- HTTP `GET`用于检索具有特定资源（例如某个pod）或资源集合或列表（例如，命名空间中的所有pods）的数据。
- HTTP `POST`用于创建资源，例如services或deployments。
- HTTP `PUT`用于更新现有资源 - 例如，更改pod的容器镜像。
- HTTP `PATCH`用于现有资源的部分更新。阅读Kubernetes文档中的 [“Use a JSON merge patch to update a Deployment”](http://bit.ly/2Xpbi6I) ，以了解有关可用策略和含义的更多信息。
- HTTP `DELETE` 用于以不可恢复的方式删除资源。

我们看一下Kubernetes api，以 [1.14 API为例](http://bit.ly/2IVevBG)，你可以看到不同的HTTP方法。例如，要使用等效的CLI命令列出当前命名空间中的pod `kubectl -n` `THE_NAMESPACE` `get pods`，您将执行`GET方法 /api/v1/namespaces/*THENAMESPACE*/pods`（参见[图2-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-list-pods)）。

![运行中的API服务器HTTP接口：列出给定命名空间中的pod](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0202.png)

###### 图2-2。运行中的API服务器HTTP接口：列出给定命名空间中的pod

有关如何从Go程序调用API服务器HTTP接口的介绍，请参阅[“客户端库”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go)。

## API术语

在我们深入API业务逻辑前，让我们首先定义Kubernetes API服务器涉及到的术语：

- 资源类型Kind

  一个资源实体的类型。每个对象都有一个字段`Kind`（具体的`kind` 值在JSON中的小写的，`Kind`在Golang中是首字母大写），它告诉如`kubectl`之类客户端，这个资源对象代表一个pod。kind可以分为三类：
  
  - 对象可以是系统中*的持久化实体* - 例如，`Pod`或`Endpoints`。对象具有名称，其中许多都存在于命名空间中。
  - 列表是一种或多种实体的集合。列表具有有限的公共metadta。比如`PodList`s或`NodeList`s。当你执行命令**kubectl get pods**，将会得到以上结果。
  - 针对对象和非持久化实体执行的特定操作，如`/binding`或`/scale`。对于discovery api，Kubernetes使用`APIGroup`和`APIResource`; 对于错误结果，它使用`Status`。

在Kubernetes程序中，一个资源类型直接对应于Golang类型。因此，作为Golang类型，资源类型的表述形式为单数，并且首字母需要大写。

- API group

  一组逻辑上相关的资源类型组成的集合。例如，所有批量操作的对象，如`Job`或`ScheduledJob`都属于批处理API group。

- version

  每API group可以存在多个版本，实际中这也是常见的。例如，一个api group最初是以`v1alpha1`出现，然后升级为`v1beta1`版本，最终发布稳定版本`v1`。一个对象可以作为一个特定的版本创建（比如：`v1beta1`）也可以在查询时指定不同于创建时指定的api group版本进行检索，只要这个版本是被api group 支持的。API服务器将进行无损转换，为返回请求版本中的对象。从集群用户的角度来看，版本只是同一对象的不同表示。

###### 提示

对于“ 一个资源实例在集群中存在一个`v1`版本的对象，另外还有一个`v1beta1`的对象。”，这种情况是不存在的，每个资源实例只有一个对象存在，可以按照用户的期望，既可以作为`v1`版本返回也可以作为`v1beta1`版本返回。

- 资源

  资源通常是一个小写的、复数的单词（例如，`pods`）标识出一组HTTP endpoints（路径），其暴露系统中某个对象类型的CRUD（创建，读取，更新，删除）语义。常见路径是：
  
  - 根目录，例如*... / pods*，列出该类型的所有实例
  
  - 各个命名资源的路径，例如*... / pods / nginx*
  
  通常，这些HTTP请求接收和返回的是同一种资源类型（第一个列子为`PodList`，在第二个例子为 `Pod`）。但在其他情况下（例如，在出现错误的情况下），会返回一个`Status`类型的对象。
  
  除了具有完整CRUD语义的主要资源之外，资源还可以有其他HTTP路径来执行特定操作（例如，*... / pod / nginx / port-forward*，*... / pod / nginx / exec*，或*... / pod / nginx / logs*）。我们调用这些*子资源 sub resource*（参见[“子资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-subresources)）。这些通常实现自定义协议而不是REST - 例如，通过WebSockets或命令式API进行某种流式连接。
  
    

###### 提示

资源和资源类型经常混在一起。请注意明确的区别：

- 资源对应于HTTP路径。
- 资源类型是这些端点返回和接收的对象，以及持久化在`etcd`中的对象。

资源始终是要和API group和version放在一起说的，统称为*GroupVersionResource*（或GVR）。GVR唯一地定义HTTP路径。例如，`default`命名空间中的具体路径是*/ apis / batch / v1 / namespaces / default / jobs*。[图2-3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#gvr)显示了一个命名空间下的资源示例GVR，a `Job`。

![Kubernetes API  - 集团，版本，资源（GVR）](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0203.png)

###### 图2-3。Kubernetes API - GroupVersionResource（GVR）

与`jobs`GVR示例相反，集群范围的资源（如节点或命名空间）本身在路径中没有*$ NAMESPACE*部分。例如，`nodes`GVR示例可能如下所示：*/ api / v1 / nodes*。请注意，出现在其他资源HTTP路径中的命名空间本身也是资源，可通过路径*/ api / v1 / namespaces进行访问*。

与GVR类似，每种资源类型都存在于API group中，具有版本，并通过*GroupVersionKind*（GVK）进行识别。

##### 共栖- 生活在多个API组中的类型

同名的类型，不仅可以在不同的*版本中*共存，也可以在不同的API group中存在。例如，`Deployment`在扩展组extensions group中以alpha类型开始，最终在其自己的组`apps.k8s.io`中升级为稳定版本。我们称之为共栖。虽然在Kubernetes中不常见，但有一些：

- `Ingress`，`NetworkPolicy`在`extensions`和`networking.k8s.io`
- `Deployment`，`DaemonSet`，`ReplicaSet`在`extensions`和`apps`
- `Event` 在核心core组和 `events.k8s.io`

GVK和GVR是相关的。GVK在GVR标识的HTTP路径下提供。将GVK映射到GVR的过程称之为REST mapping。我们将看到`RESTMappers`在Golang中实现[“REST Mapping”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#RESTMapping)。

从全局的角度来看，API资源空间在逻辑上形成了一个具有顶级节点的树，包括*/ api*，*/ apis*和一些非层次化端点，例如*/ healthz*或*/ metrics*。此API空间的示例呈现[如图2-4](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-space-tree)所示。请注意，确切的层次和路径与Kubernetes版本相关，随着时间将日趋稳定。

![一个示例Kubernetes API空间](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0204.png)

###### 图2-4。一个Kubernetes API空间示例

## Kubernetes API版本控制

对于可扩展性原因，Kubernetes支持不同API路径下的多个API版本，例如*/ api / v1*或*/ apis / extensions / v1beta1*。不同的API版本意味着不同级别的稳定性和支持：

- *Alpha*级别（例如`v1alpha1`）通常在默认情况下禁用; 对功能的支持可能随时被删除，恕不另行通知，并且只能在短期测试群集中使用。
- *Beta*级别（例如`v2beta3`）默认情况下启用，这意味着代码已经过充分测试; 但是，对象的语义可能会在随后的beta或稳定版本中以不兼容的方式发生变化。
- *稳定*（通常可用或GA）级别（例如`v1`）对于许多后续版本，将出现在已发布的软件中。

让我们来看看如何构造HTTP API空间：在顶层我们区分核心组 - 即*/ api / v1*下面的所有内容- 以及以命名组形式的路径下，如：*/ apis / `$NAME`/ `$VERSION`*。

###### 注意

由于历史原因，核心小组位于`/api/v1`下，而不是人们所期望的*/ apis / core / v1*下。因为在引入API组的概念之前，核心组已经存在。

API服务器公开了第三种类型的HTTP路径 - 非资源对齐的路径：集群范围的实体，例如*/ metrics*，*/ logs*或*/ healthz*。此外，API服务器支持watch; 也就是说，您可以添加`?watch=true`特定请求，并将API服务器转换为[watch模式](http://bit.ly/2x5PnTl)，代替通常设定时间间隔去轮询资源的模式。

## 声明性状态管理

大多数API需要区分*期望的*对象状态和当前对象的*状态*。规格specification，或简称为spec，就是对一种资源的期望状态的完整描述，并且通常被持久化存储下来，通常会使用`etcd`来存储。

###### 注意

为什么我们说“通常会是`etcd`”？好吧，有的Kubernetes发行版和产品，如[k3s](https://k3s.io/)或微软的AKS，已经取代或正在努力替换`etcd`其他东西。得益于Kubernetes控制平面的模块化架构，这实现起来很方便。

让我们在API服务器的上下文中多讨论一些规格Spec（期望状态）与状态（当前观察到的状态）。

规格描述了您所需的资源状态，您需要通过命令行工具例如`kubectl`，或者通过Go代码以编程方式获取。当前资源的状态（或者说资源当前被观察到的状态），是由控制平面管理的。可以时由核心组件（如控制管理器controller manager）或您自己的自定义控制管理器custom controller（请参阅[“控制器和operator”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#ch_controllers-operators)）。例如，在部署deployment时，您可以指定您希望始终运行20个应用程序副本。deployment控制器是控制平面中控制管理器的一部分，它读取您提供的deployment规格并创建一个副本集replica set，然后控制权交由replica set管理：它创建相应数量的pod，最终（通过`kubelet`）将容器在工作节点上启动。如果任何副本失败，deployment控制器会在status状态信息中反馈给您。这就是我们所谓的*声明式状态管理* - 也就是说，声明所需的状态并让Kubernetes处理剩下的事情。

我们将在下一节中看到更多声明式状态管理的内容，我们会从命令行开始探索API。

# 在命令行中访问API

在本节我们将使用`kubectl`和`curl`演示访问Kubernetes API。如果您不熟悉这些CLI工具，现在是安装它们并试用它们的好时机。

首先，让我们看一下资源的期望状态和当前观察到的状态。我们以一个每个集群都会存在的属于控制平面的CoreDNS插件为例进行演示，该插件在`kube-system`命名空间中（旧版本的Kubernetes中为Kube-dns插件）（此输出只显示了我们所关注重要部分）：

```shell
$ kubectl -n kube-system get deploy / coredns -o =yaml
apiVersion：apps / v1
kind：Deployment
metadata：
  name：coredns
  namespace：kube-system
  ...
spec：
  template：
    spec：
      containers：
      - name：coredns
        image：602401143452.dkr.ecr.us-east-2.amazonaws.com/eks/coredns:v1.2.2
  ...
status：
  replicas：2
  conditions：
  - type：Available
    status："True"
    lastUpdateTime："2019-04-01T16:42:10Z"
  ...
```

正如您从此`kubectl`命令中看到的那样，在`spec`的部分中，您将定义一些特征，例如要使用的容器映像以及要并行运行的副本数，并在本`status`节中看到了当前运行的有多少个副本。

为了执行与CLI相关的操作，在本章的其余部分中，我们将使用批处理操作作为运行示例。让我们首先在终端中执行以下命令：

```sh
$ kubectl proxy --port =8080
Starting to serve on 127.0.0.1:8080
```

此命令将Kubernetes API代理到本地计算机，并且还负责身份验证和授权。它允许我们通过HTTP直接发出请求并接收JSON返回结果。让我们通过启动第二个终端来查询的`v1` 下的资源：

```shell
$ curl http://127.0.0.1:8080/apis/batch/v1
 {
  "kind"："APIResourceList"，
   "apiVersion"："v1"，
   "groupVersion"："batch/v1"，
   "resources"：[
    {
      "name"："jobs"，
       "singularName"：""，
       "namespaced"：true，
       "kind"："Job"，
       "verbs"：[
        "create"，
         "delete"，
         "deletecollection"，
         "get"，
         "list"，
         "patch"，
         "update"，
         "watch"
      ]，
       "categories"：[
        "all"
      ]
    }，
     {
      "name"："jobs/status"，
       "singularName"：""，
       "namespaced"：true，
       "kind"："Job"，
       "verbs"：[
        "get"，
         "patch"，
        "update"
      ]
    }
  ]
}
```

###### 提示

除了使用`curl`与`kubectl proxy`命令一起使用来对Kubernetes API进行直接HTTP API访问之外。您可以改为使用`kubectl get --raw`命令：例如，替换`curl http://127.0.0.1:8080/apis/batch/v1`为`kubectl get --raw /apis/batch/v1`。

将以上信息与`v1beta1`版本进行比较，注意到在查看*http://127.0.0.1:8080/apis/batch* 时，您可以获得批处理API组的受支持版本列表。

`v1beta1`版本返回信息如下：

```shell
$ curl http://127.0.0.1:8080/apis/batch/v1beta1
 {
  "kind"："APIResourceList"，
   "apiVersion"："v1"，
   "groupVersion"："batch/v1beta1"，
   "resources"：[
    {
      "name"："cronjobs"，
       "singularName"：""，
       "namespaced"：true，
       "kind"："CronJob"，
       "verbs"：[
        "create"，
         "delete"，
         "deletecollection"，
         "get"，
         "list"，
         "patch"，
         "update"，
         "watch"
      ]，
       "shortNames"：[
        "cj"
      ]，
       "categories"：[
        "all"
      ]
    }，
     {
      "name"："cronjobs/status"，
       "singularName"：""，
       "namespaced"：true，
       "kind"："CronJob"，
       "verbs"：[
        "get"，
         "patch"，
        "update"
      ]
    }
  ]
}
```

如您所见，该`v1beta1`版本还包含`cronjobs`资源，对应的类型为`CronJob`。在撰写本文时，cronjob尚未升级到`v1`版本。

如果您想了解集群中支持哪些API资源，包括它们的类型，是否为命名空间，以及它们的短名称（主要用于`kubectl`命令行），您可以使用以下命令：

```shell
$ kubectl api-resources
NAME                   SHORTNAMES APIGROUP NAMESPACED   KIND
bindings                                   true         Binding
componentstatuses      cs                  false        ComponentStatus
configmaps             cm                  true         ConfigMap
endpoints              ep                  true         Endpoints
events                 ev                  true         Event
limitranges            limits              true         LimitRange
namespaces             ns                  false        Namespace
nodes                  no                  false        Node
persistentvolumeclaims pvc                 true         PersistentVolumeClaim
persistentvolumes      pv                  false        PersistentVolume
pods                   po                  true         Pod
podtemplates                               true         PodTemplate
replicationcontrollers rc                  true         ReplicationController
resourcequotas         quota               true         ResourceQuota
secrets                                    true         Secret
serviceaccounts        sa                  true         ServiceAccount
services               svc                 true         Service
controllerrevisions               apps     true         ControllerRevision
daemonsets             ds         apps     true         DaemonSet
deployments            deploy     apps     true         Deployment
...
```

使用以下命令，对于确定集群中所支持的不同资源版本非常有用：

```shell
$ kubectl api-versions
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
appmesh.k8s.aws/v1alpha1
appmesh.k8s.aws/v1beta1
apps/v1
apps/v1beta1
apps/v1beta2
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1beta1
crd.k8s.amazonaws.com/v1alpha1
events.k8s.io/v1beta1
extensions/v1beta1
networking.k8s.io/v1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```

# API服务器如何处理请求

现在您对面向外部的HTTP接口有了一定的理解，那么我们将重点关注API服务器的内部工作原理。[图2-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-high-level-flow)显示了一个高层视角观察API服务器对请求的处理。

![Kubernetes API服务器请求处理概述](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0205.png)

###### 图2-5。Kubernetes API服务器请求处理概览

所以实际上当HTTP请求到达Kubernetes API时会发生什么呢？在较高的层面上，发生以下交互过程：

1. HTTP请求由注册的过滤器链处理在`DefaultBuildHandlerChain()`方法中。该链定义在[*k8s.io/apiserver/pkg/server/config.go中*](http://bit.ly/2x9t27e)，稍后将详细讨论。它将在之前提到的请求上应用一系列过滤操作。每一个过滤器将传递和携带各自的信息到上下文中，准确地来说`ctx.RequestInfo`，这里`ctx`就是Go语言中的[context](https://golang.org/pkg/context)Go（这里场景假设为，通过认证的用户） -如果用户未通过认证，它返回一个适当的HTTP响应代码并说明原因（例如，用户身份验证失败时返回[`401`响应](https://httpstatuses.com/401)）。
2. 接下来，根据HTTP路径，代码[*k8s.io/apiserver/pkg/server/handler.go中*](http://bit.ly/2WUd0c6)的多路复用器将HTTP请求路由到相应的处理程序。
3. 每个API group对应注册了一个处理程序 - 有关详细信息，请参阅[*k8s.io/apiserver/pkg/endpoints/groupversion.go*](http://bit.ly/2IvvSKA)和[*k8s.io/apiserver/pkg/endpoints/installer.go*](http://bit.ly/2Y1eySV)。它接受HTTP请求以及context上下文（例如，用户和访问权限）以及将所需的对象从`etcd`存储中检索出来。

现在让我们仔细看看[*server / config.go中*](http://bit.ly/2LWUUnQ)设置的过滤器DefaultBuildHandlerChain()，以及其中发生的事情：

```go
func DefaultBuildHandlerChain(apiHandler http.Handler, c *Config) http.Handler {
    h := WithAuthorization(apiHandler, c.Authorization.Authorizer, c.Serializer)
    h = WithMaxInFlightLimit(h, c.MaxRequestsInFlight,
          c.MaxMutatingRequestsInFlight, c.LongRunningFunc)
    h = WithImpersonation(h, c.Authorization.Authorizer, c.Serializer)
    h = WithAudit(h, c.AuditBackend, c.AuditPolicyChecker, LongRunningFunc)
    ...
    h = WithAuthentication(h, c.Authentication.Authenticator, failed, ...)
    h = WithCORS(h, c.CorsAllowedOriginList, nil, nil, nil, "true")
    h = WithTimeoutForNonLongRunningRequests(h, LongRunningFunc, RequestTimeout)
    h = WithWaitGroup(h, c.LongRunningFunc, c.HandlerChainWaitGroup)
    h = WithRequestInfo(h, c.RequestInfoResolver)
    h = WithPanicRecovery(h)
    return h
}
```

所有包都在[*k8s.io/apiserver/pkg中*](http://bit.ly/2LUzTdx)。更具体地审查：

- `WithPanicRecovery()`

  处理recovery并且记录panic。定义在[*server / filters / wrap.go中*](http://bit.ly/2N0zfNB)。

- `WithRequestInfo()`

  给context上下文附一个`RequestInfo`。定义在[*endpoints/filters/requestinfo.go*](http://bit.ly/2KvKjQH).。

- `WithWaitGroup()`

  将所有非长时间运行的请求添加到等待组; 用于优雅关机。定义在[*server / filters / waitgroup.go中*](http://bit.ly/2ItnsD6)。

- `WithTimeoutForNonLongRunningRequests()`

  设置非长期运行的请求的超时时间（如大多数的`GET`，`PUT`，`POST`，和`DELETE`请求），相对的长期运行的请求有watch和proxy请求。方法定义在[*server / filters / timeout.go中*](http://bit.ly/2KrKk8r)。

- `WithCORS()`

  提供[CORS](https://enable-cors.org/)跨域的实现。CORS是跨域资源共享的缩写，是一种允许嵌入HTML页面的JavaScript将XMLHttpRequests发送到与JavaScript发起的域不同的域的机制。在[*server / filters / cors.go中*](http://bit.ly/2L2A6uJ)定义。

- `WithAuthentication()`

  尝试将对请求进行身份验证，用户可以是人员或机器用户，并将用户信息存储在提供的上下文中。成功时，`Authorization`将从请求中删除HTTP header信息。如果身份验证失败，则返回HTTP `401`状态代码。在[*endpoints/filters/authentication.go*](http://bit.ly/2Fjzr4b)定义。

- `WithAudit()`

  使用所有传入请求加入审核日志记录信息功能的装饰器处理程序。审计日志条目包含诸如请求的源IP，用户调用操作和请求的命名空间之类的信息。在[*admission/audit.go*](http://bit.ly/2XpQN9U)定义。

- `WithImpersonation()`

  通过检查尝试更改用户的请求（类似于`sudo`）来处理用户模拟。在[*endpoints/filters/impersonation.go*](http://bit.ly/2L2UETP)定义。

- `WithMaxInFlightLimit()`

  限制已已发起未结束的请求数量。在[*server / filters / maxinflight.go中*](http://bit.ly/2IY4unl)定义。

- `WithAuthorization()`

  通过调用授权模块检查权限，并将所有授权请求传递给多路复用器，多路复用器将请求分派给正确的处理程序。如果用户没有足够的权限，则返回HTTP `403`状态代码。Kubernetes现在使用基于角色的访问控制（RBAC）。定义在[*endpoints/filters/authorization.go*](http://bit.ly/31M2NSA)。

在通用处理程序链处理完后（[图2-5中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-high-level-flow)的第一个框），实际的请求处理开始（即执行请求处理程序的语义）：

- 对于请求*/*，*/ version*，*/ apis*，*/ healthz*和其他nonRESTful API的请求将被直接处理。

- 对于RESTful资源的请求进入请求管道，包括：

  - admission

    传入的对象进入admission处理链。这个链中有20种不同admission插件。[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#idm46336866991944)每个插件都可以是修改阶段的一部分（参见[图2-5中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-high-level-flow)的第三个框），或是验证阶段的一部分（参见图中的第四个框），或两者兼而有之。

    在修改阶段，可以改变传入的请求payload; 例如，镜像拉取策略被设置为`Always`，`IfNotPresent`，或`Never`取决于admission配置。

    第二个admission阶段纯粹是为了验证; 例如，验证pod中的安全设置，或者在创建该命名空间中的对象之前验证是否存在命名空间。

  - validation

    根据通用验证逻辑检查传入对象，该逻辑对系统中的每个对象类型都存在。例如，检查字符串格式以验证服务名称中仅使用有效的DNS兼容字符，或者pod中的所有容器名称是唯一的。
  
  - `etcd`-端的CRUD逻辑
  
    这里实现了我们在[“API服务器的HTTP接口”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-http-interface)看到的不同方法; 例如，更新逻辑从`etcd`中读取对象，检查没有其他用户在[“乐观并发”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#optimistic-concurrency)意义上修改了对象，如果没有，则将请求对象写入`etcd`。

我们将在以下章节中更详细地研究所有这些步骤; 例如：

- 自定义资源

  验证在 [“Validating Custom Resources”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-validation)，admission在 [“Admission Webhooks”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#admission-webhooks)中，还有常规的CRUD语义在[第4章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)

- Golang原生资源

  Validation in [“Validation”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-validation), admission in [“Admission”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-admission), and the implementation of CRUD semantics in [“Registry and Strategy”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-registry)

# 摘要

在本章中，我们首先将Kubernetes API服务器作为黑盒子进行了讨论，并查看了其HTTP接口。然后你学习了如何在命令行上与那个黑盒子进行交互，最后我们打开了黑盒子并探索了它的内部工作原理。到目前为止，您应该知道API服务器如何在内部工作，以及如何使用命令行工具`kubectl`与资源进行交互以进行资源探索和操作。

现在是时候将手动交互留在我们身后的命令行，并开始使用Go： `client-go`（Kubernetes“标准库”的核心）进行编程API服务器访问。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#idm46336866991944-marker)在一个Kubernetes 1.14集群，这些是（以此顺序）： `AlwaysAdmit`, `NamespaceAutoProvision`, `NamespaceLifecycle`, `NamespaceExists`, `SecurityContextDeny`, `LimitPodHardAntiAffinityTopology`, `PodPreset`, `LimitRanger`, `ServiceAccount`, `NodeRestriction`, `TaintNodesByCondition`, `AlwaysPullImages`, `ImagePolicyWebhook`, `PodSecurityPolicy`, `PodNodeSelector`, `Priority`, `DefaultTolerationSeconds`, `PodTolerationRestriction`, `DenyEscalatingExec`, `DenyExecOnPrivileged`, `EventRateLimit`, `ExtendedResourceToleration`, `PersistentVolumeLabel`, `DefaultStorageClass`, `StorageObjectInUseProtection`, `OwnerReferencesPermissionEnforcement`, `PersistentVolumeClaimResize`, `MutatingAdmissionWebhook`, `ValidatingAdmissionWebhook`, `ResourceQuota`, and `AlwaysDeny`.
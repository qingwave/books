# 第3章client-go基础知识

我们现在将重点关注Go语言下的Kubernetes编程接口。您将学习如何通过Kubernetes API访问我们熟知的原生类型（如pod，service和deployment）。在后面的章节中，这些技术将扩展到用户定义的类型。但是，我们首先看一下每个Kubernetes集群都必不可少的所有API对象。

# 存储库

Kubernetes项目在GitHub上的*kubernetes*组织下提供了许多可供第三方使用的Git存储库。您需要注意在导入Golang包时要使用别名*k8s.io / ...*（不是*github.com/kubernetes/...*）来将所有这些library导入到您的项目中。我们将在以下部分介绍这些最重要的Git存储库。

## 客户端库

该Go中的Kubernetes编程接口主要由*k8s.io/client-go*库组成（为简洁起见，我们将其称之为`client-go`）。*client-go*是一个典型的Web服务客户端库，支持所有正式发布的KubernetesAPI类型。它可以用来执行常见的 REST方法：

- Create
- Get
- List
- Update
- Delete
- Patch

这些REST方法中的每一个都使用[“API服务器的HTTP接口”实现](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-http-interface)。此外，它还支持`Watch`操作，这是类Kubernetes的API的一个特别之处，也是与其他API主要区别之一。

[`client-go`](http://bit.ly/2RryyLM) 源码可以GitHub上访问（参见[图3-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#github-client-go)），在Go代码中使用*k8s.io/client-go* 作为包名导入。它与Kubernetes的代码是配套使用的; 也就是说，对于每个Kubernetes `1.x.y`版本，都有一个带有匹配标记`kubernetes-1.x.y`的client-go版本。

![Github上的`client-go`存储库](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0301.png)

###### 图3-1。GitHub上的client-go存储库

另外，版本方面client-go还有一套语义化版本控制。例如，`client-go` 9.0.0匹配Kubernetes 1.12版本，`client-go` 10.0.0匹配Kubernetes 1.13版本，依此类推。未来可能会有更细粒度的版本定义。除了Kubernetes API对象的客户端代码外，`client-go`还包含许多通用代码。也可以被[第4章中的](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)用户定义的API对象使用到。有关软件包列表，请参[见图3-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#github-client-go)。

所有软件包都将各司其职，我们会从大多数调用Kubernetes API的代码中看到，*tools / clientcmd /*将被用来从`kubeconfig`文件中创建一个客户端配置，而*kubernetes /则*用于创建实际的Kubernetes API客户端。我们会在后续的示例中进一步看到使用细节。在此之前，让我们快速浏览一下其他相关的存储库和软件包。

## Kubernetes API类型

如我们已经看到，`client-go`拥有客户端接口。pod，service和deployment等对象的Kubernetes API Go类型位于[其自己的存储库中](http://bit.ly/2ZA6dWH)。在Go代码中通过导入`k8s.io/api`来访问它们。

Pod属于遗留API组中`v1`版本的一部分（通常也称为“核心”组）。因此，Go语言中`Pod`代码定义在k8s.io/api/core/v1包中，与Kubernetes中的所有其他API类型类似。看到 [图3-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#github-api)显示了包列表，其中大多数都与Kubernetes API组及其版本相对应。

具体的Go类型是定义在*types.go*文件中（例如，*k8s.io / api / core / v1 / type.go*）。此外，也会有其他文件，其中大部分是由代码生成器自动生成的。

![Github上的API存储库](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0302.png)

###### 图3-2。GitHub上的API存储库

## API Machinery

还有一个常被用到的第三方存储库名为[API Machinery](http://bit.ly/2xAZiR2)，在Go语言中使用`k8s.io/apimachinery`包路径导入。这个库实现Kubernetes类API操作的所有通用方法。API Machinery不仅限于实现对容器的管理，也可用于为在线商店或任何其他特定领域的业务来构建API。

虽然我们将在Kubernetes-native Go代码中看到很多属于API Machinery库的软件包。其中一个非常重要的包*k8s.io/apimachinery/pkg/apis/meta/v1*我们需要单独提一下，它包含了许多通用的API类型如`ObjectMeta`，`TypeMeta`，`GetOptions`，和`ListOptions`（参见[图3-3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#github-apimachinery)）。

![Github上的API Machinery存储库](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0303.png)

###### 图3-3。GitHub上的API Machinery存储库

## 创建和使用客户端

现在我们知道了创建Kubernetes客户端对象的步骤，意味着我们可以访问Kubernetes集群中的资源。我们假设您可以通过本机访问一个kubernetes集群（即，`kubectl`已正确设置并配置好了凭据信息），以下代码演示了如何在Go项目中使用`client-go`包：

```go
import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/tools/clientcmd"
    "k8s.io/client-go/kubernetes"
)

kubeconfig = flag.String("kubeconfig", "~/.kube/config", "kubeconfig file")
flag.Parse()
config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
clientset, err := kubernetes.NewForConfig(config)

pod, err := clientset.CoreV1().Pods("book").Get("example", metav1.GetOptions{})
```

代码导入`meta/v1`包以获取访问权限`metav1.GetOptions`。此外，它从`client-go`中导入`clientcmd`以便读取和解析 kubeconfig（即配置了服务器名称，凭据等的客户端所需的配置信息）。然后，它导入`client-go`中的`kubernetes`，来使用Kubernetes的client sets 操作相应的kubernetes资源。

kubeconfig文件的默认位置位于用户Home目录的*.kube / config*中。`kubectl`也是从此文件中获取Kubernetes集群凭据。

之后kubeconfig 将被`clientcmd.BuildConfigFromFlags`读取和解析。为了突出主要逻辑我们在整个代码中省略了对错误err的处理，但是错误处理还是需要提前考虑到的，如果kubeconfig格式不正确，将会返回相应的语法错误。由于语法错误在Go代码中很常见，也可以参照下边的代码检查错误，如下所示：

```go
config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
if err != nil {
    fmt.Printf("The kubeconfig cannot be loaded: %v\n", err
    os.Exit(1)
}
```

从`clientcmd.BuildConfigFromFlags`我们得到一个`rest.Config`，您可以在*k8s.io/client-go/rest*包中找到。将这个config传递给`kubernetes.NewForConfig`方法，来创建实际的Kubernetes *客户端集合*。之所以被称为*客户端集和，*因为它包含了多个客户端共同来访问所有本地Kubernetes资源。

在集群的pod中运行可执行文件时，`kubelet`将自动将service account装入容器的*/var/run/secrets/kubernetes.io/serviceaccount*路径下。它等效于刚刚提到的kubeconfig文件，我们可以通过`rest.InClusterConfig()`方法很容易地获取`rest.Config`这个配置。你可能经常会看到下列组合`rest.InClusterConfig()`和`clientcmd.BuildConfigFromFlags()`一起使用的代码，这段代码不仅支持从集群内部的pod中获取config配置，也支持在集群之外通过~/.kube/config配置文件或者`KUBECONFIG`环境变量的方式获取config配置：

```go
config, err := rest.InClusterConfig()
if err != nil {
    // fallback to kubeconfig
    kubeconfig := filepath.Join("~", ".kube", "config")
    if envvar := os.Getenv("KUBECONFIG"); len(envvar) >0 {
        kubeconfig = envvar
    }
    config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
    if err != nil {
        fmt.Printf("The kubeconfig cannot be loaded: %v\n", err
        os.Exit(1)
    }
}
```

在下面的示例代码，我们使用方法`clientset.CoreV1()`首先选择核心core组`v1`版本的资源，再选择命名空间`"book"下的pod资源，最后指定该资源名称为"example"：

```go
 pod, err := clientset.CoreV1().Pods("book").Get("example", metav1.GetOptions{})
```

请注意，只有最后一个Get方法会实际访问服务器。前两部选择CoreV1`和`Pods`选择客户的操作只是做了参数的设置，（这通常被称为构建者模式，这里构建了一个请求）。

`Get`调用将HTTP `GET`请求发送到服务器上的*/ api / v1 / namespaces / book / pods / example*，这是在kubeconfig中设置的。如果Kubernetes API服务器返回HTTP`200`应答码，则表明请求成功，响应的报文将携带编码过的pod对象，一般有两种协议格式JSON或protobuf，`client-go`默认采用JSON格式。

###### 注意

你可以通过在创建客户端之前修改REST配置，来为访问Kubernetes原生资源客户端启用protobuf：

```go
cfg, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
cfg.AcceptContentTypes = "application/vnd.kubernetes.protobuf,
                          application/json"
cfg.ContentType = "application/vnd.kubernetes.protobuf"
clientset, err := kubernetes.NewForConfig(cfg)
```

请注意，[第4章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)介绍的自定义资源不支持protobuf。

## 版本控制和兼容性

Kubernetes API是版本控制的。我们在上一节中看到过pods属于`v1`核心组。核心组实际上现在只有一个版本中。对于其他组，例如，`apps`组，存在于`v1`，`v1beta2,`和`v1beta1`多个版本（撰写本文时）。如果您查看[*k8s.io/api/apps*](http://bit.ly/2L1Nyio)包，您将找到这些版本的所有API对象。在[*k8s.io/client-go/kubernetes/typed/apps*](http://bit.ly/2x45Uab)包中，您将看到所有这些版本的客户端实现。

所有以上看到的内容都只是从客户端一侧定义的。目前还没有提到Kubernetes集群及其API服务器的。可以想象如果我们使用客户端定义好的，但是API服务器不支持的API group版本进行访问时，调用将会失败。客户端作为某个版本的硬编码实现，当应用程序开发人员使用时必须选择正确的API group版本才能与手头的集群成功通信。有关API group兼容性保证的更多信息，请参阅[“API版本和兼容性保证”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#api-versions)。

兼容性的第二个方面API服务器的API 元数据，这些结构我们使用`client-go`是与API服务器对话时会用得到。例如，CRUD操作时的Options结构，如`CreateOptions`，`GetOptions`，`UpdateOptions`，和`DeleteOptions`。另一个重要的是`ObjectMeta`（在[“ObjectMeta”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#ObjectMeta)中详细讨论），它是[所有资源类型的一个 属性](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#ObjectMeta)。这些元数据的结构在增加新功能时都会经常被扩展; 我们通常把元数据方面的功能称为*API machinery功能*。在这些元数据结构定义的源代码中，通过注释来描述该功能何时被视为alpha或beta。相同的API兼容性保证适用于任何其他API字段。

在下面的示例中，`DeleteOptions`结构在包[*k8s.io/apimachinery/pkg/apis/meta/v1/types.go中*](http://bit.ly/2MZ9flL)定义：

```go
// DeleteOptions may be provided when deleting an API object.
type DeleteOptions struct {
    TypeMeta `json:",inline"`

    GracePeriodSeconds *int64 `json:"gracePeriodSeconds,omitempty"`
    Preconditions *Preconditions `json:"preconditions,omitempty"`
    OrphanDependents *bool `json:"orphanDependents,omitempty"`
    PropagationPolicy *DeletionPropagation `json:"propagationPolicy,omitempty"`

    // When present, indicates that modifications should not be
    // persisted. An invalid or unrecognized dryRun directive will
    // result in an error response and no further processing of the
    // request. Valid values are:
    // - All: all dry run stages will be processed
    // +optional
    DryRun []string `json:"dryRun,omitempty" protobuf:"bytes,5,rep,name=dryRun"`
}
```

最后一个字段，`DryRun`在Kubernetes 1.12中添加为alpha，在1.13中添加为beta（默认启用）。早期版本中的API服务器无法理解它。根据功能的不同，传递这样的选项可能会被忽略甚至被拒绝。因此，拥有一个`client-go`与集群版本相距不太远的版本非常重要。

###### 提示

到底k8s.io/api包中资源的哪个版本拥有哪些级别的属性字段，可以访问不同发行版本的kubernetes源代码来查看，例如，可以访问[`release-1.13`分支中的](http://bit.ly/2Yrhjgq) Kubernetes 1.13 版本的代码。Alpha级别的字段会在其注释中写明。

这里有一份[生成的API文档](http://bit.ly/2YrfiB2)，方便查询。它与*k8s.io/api包中的*信息相同。

最后，许多alpha和beta功能都有相应的[功能开关](http://bit.ly/2RP5nmi)（请在此处查看[主要功能开关](http://bit.ly/2FPZPTT)）。功能相关的讨论在[问题反馈中](http://bit.ly/2YuHYcd)有所记录。

Kubernetes集群和`client-go`版本关系矩阵在`client-go`存储库的 [README中](http://bit.ly/2RryyLM)发布（[参见表3-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go-compatibility)）。

|                | Kubernetes 1.9 | Kubernetes 1.10 | Kubernetes 1.11 | Kubernetes 1.12 | Kubernetes 1.13 | Kubernetes 1.14 | Kubernetes 1.15 |
| :------------- | :------------- | :-------------- | :-------------- | :-------------- | :-------------- | :-------------- | --------------- |
| client-go 6.0  | ✓              | +–              | +–              | +–              | +–              | +–              | +–              |
| client-go 7.0  | +–             | ✓               | +–              | +–              | +–              | +–              | +–              |
| client-go 8.0  | +–             | +–              | ✓               | +–              | +–              | +–              | +–              |
| client-go 9.0  | +–             | +–              | +–              | ✓               | +–              | +–              | +–              |
| client-go 10.0 | +–             | +–              | +–              | +–              | ✓               | +–              | +–              |
| client-go 11.0 | +–             | +–              | +–              | +–              | +–              | ✓               | +–              |
| client-go 12.0 | +–             | +–              | +–              | +–              | +–              | +–              | ✓               |
| client-go HEAD | +–             | +–              | +–              | +–              | +–              | +–              | +–              |

- ✓：`client-go`和Kubernetes版本具有相同的功能和相同的API组版本。
- `+`：`client-go`具有Kubernetes群集中可能不存在的功能或API组版本。这可能是因为`client-go`增加了新功能，或者是Kubernetes删除了旧的，已弃用的功能。但是，属于它们的功能交集（即大多数API）都可以使用。
- `–`：`client-go`明显与Kubernetes集群不兼容。

从[表3-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go-compatibility)表明`client-go`库与它对应的集群版本之间需要匹配支持。在版本偏差的情况下，开发人员必须仔细衡量他们使用哪些功能和哪些API group，以及应用程序所访问的集群版本是否支持这些功能。

在[表3-1中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go-compatibility)，`client-go`列出了版本。我们简单地提到[“客户端库”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go)即`client-go`正式发版中使用语义化（semver）版本控制，即当Kubernetes的次版本号（1.13.2中的13）递增时，通过递增`client-go`的主版本号。Kubernetes 1.4时`client-go`发布1.0，现在`client-go `12.0与Kubernetes 1.15版本对应（在撰写本文时）。

这里的semver版本控制仅适用于`client-go`自身，而不适用于API Machinery或API存储库。相反，后者采用了Kubernetes版本进行标记，[如图3-4所示](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go-versioning)。请参见[“Vendoring”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#vendoring)看看如何将*k8s.io/client-go*，*k8s.io/apimachinery*和*k8s.io/api*在加入您项目的vendor中 。

![客户端版本控制](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0304.png)

###### 图3-4。客户端版本控制

## API版本和兼容性保证

如果您使用代码访问不同版本的集群，那么在上一节中看到，选择正确的API group版本可能至关重要。Kubernetes中所有APIgroup都是版本化的。一种常见的Kubernetes风格的版本控制方案，它由alpha，beta和GA（一般可用性）版本组成。

模式是：

- `v1alpha1`，`v1alpha2`，`v2alpha1`，等有称为*alpha版本*并被认为是不稳定的 这意味着：
  - 他们可能会以任何不相容的方式随时丢弃或改变。
  - Kubernetes版本变更，数据可能会被丢弃，丢失或无法访问。
  - 如果管理员未手动选择，则默认情况下通常会禁用它们。
- `v1beta1`，`v1beta2`，`v2beta1`，等等，都称为*beta版本*。他们正在走向稳定，这意味着：
  - 在有beta升级为stable时，不会直接变更为stable而去掉beta版本，它们将与相应的稳定API版本同时存在，至少存在于一个Kubernetes版本。
  - 它们通常不会以不兼容的方式改变，但没有严格的保证。
  - 存储在beta版中的对象不会被删除或无法访问。
  - 默认情况下，Beta版本通常在群集中默认启用。但这可能取决于所使用的Kubernetes版本或云提供商。
- `v1`，`v2`等等是稳定的版本，通常可用的API; 这意味着：
  - 他们会一直保留下来。
  - 它们将保证兼容性。

###### 提示

Kubernetes有正式的[功能下线政策](http://bit.ly/2FOrKU8)。您可以在[Kubernetes社区GitHub上](http://bit.ly/2XKPWAX)找到更多关于API设计兼容性的更多详细信息。

与API group version相关，要记住两个要点：

- API 分组和版本对于一个API资源都是比不可少的，例如pod或service定义中可以看得到。除API组和版本外，API资源可能具有正交版本的单个字段; 例如，稳定API中的某个字段也可能是新引入的alpha属性，在字段的注释中一般都会有说明信息。上文中对API组不同版本级别的规则描述同样适用于此类字段。例如：

  - 稳定API中的一个alpha字段可能会变得不兼容，数据会丢失或随时被废弃。例如，`ObjectMeta.Initializers`就是这样一个alpha 字段，并会在不久的将来废弃（在1.14中已弃用）：

    ```go
    // DEPRECATED - initializers are an alpha field and will be removed
    // in v1.15.
    Initializers *Initializers `json:"initializers,omitempty"
    ```

  - alpha字段通常在默认情况下处于禁用状态，必须在API服务器中启用相应的feature gate，如下所示：

    ```go
    type JobSpec struct {
        ...
        // This field is alpha-level and is only honored by servers that
        // enable the TTLAfterFinished feature.
        TTLSecondsAfterFinished *int32 `json:"ttlSecondsAfterFinished,omitempty"
    }
    ```

  - API服务器对不同的alpha字段的处理行为也是因字段而异的。如果未启用相应的功能开关feature gate，有些alpha字段会被拒绝，有些会被忽略。这些具体的处理行为会在字段注释中被描述（参见`TTLSecondsAfterFinished`上文的示例）。

- 此外，API组的版本在访问API时起到作用。在同一资源的不同版本之间，由API服务器会完成即时转换。也就是说，您可以创建一个版本的对象（例如`v1beta1`），在后续访问时请求一个不同版本的对象（例如`v1`），而无需在您的应用程序中进行任何进一步的工作。这对于构建向后兼容和向前兼容的应用程序非常方便。

  - 存储在`etcd`中的每个对象被标记了特定版本号。默认情况下，这称为该资源的*存储版本*。虽然对于不同版本的Kubernetes这个存储版本是可以更改的，但存储的对象版本不会自动更新（在撰写此文时）。因此，在删除旧版本支持之前，集群管理员必须确保在更新Kubernetes集群时及时进行迁移。目前还没有通用的迁移机制，不同版本的Kubernetes发行版迁移方法也会有所不同。
  - 不过，应用程序开发人员来说，无需关注这个细节。API服务器的即时转换将确保应用程序从集群中访问到对象是一致的。应用程序甚至不会注意到正在使用哪个存储版本。存储版本控制这个细节在应用程序编写Go代码时是透明的，无需关注的。

# Go中的Kubernetes对象

在[“创建和使用客户端”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)，我们了解了如何为核心组创建客户端从而访问Kubernetes集群中的pod。在下文中，我们想要更详细地了解一下pod或其他Kubernetes资源，就此而言，我们将深入到Go代码的世界中进一步探索。

Kubernetes资源 - 或者更准确地说是对象 - 作为资源类型Kind[1的](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#idm46336866123400)实例在API服务器中是作为作为资源来管理，对应到Go语言中则是structs。不同的资源类型，它们的字段属性当然是不同。但另一方面，他们也会有部分相同的结构定义。

从Golang语言类型系统的角度来看，Kubernetes对象实现了一个名为的Go接口 `runtime.Object`定义在包*k8s.io/apimachinery/pkg/runtime*，这个接口非常简单：

```go
// Object interface must be supported by all API types registered with Scheme.
// Since objects in a scheme are expected to be serialized to the wire, the
// interface an Object must provide to the Scheme allows serializers to set
// the kind, version, and group the object is represented as. An Object may
// choose to return a no-op ObjectKindAccessor in cases where it is not
// expected to be serialized.
type Object interface {
    GetObjectKind() schema.ObjectKind
    DeepCopyObject() Object
}
```

这里，`schema.ObjectKind`（来自*k8s.io/apimachinery/pkg/runtime/schema*包）是另一个简单的接口：

```go
// All objects that are serialized from a Scheme encode their type information.
// This interface is used by serialization to set type information from the
// Scheme onto the serialized version of an object. For objects that cannot
// be serialized or have unique requirements, this interface may be a no-op.
type ObjectKind interface {
    // SetGroupVersionKind sets or clears the intended serialized kind of an
    // object. Passing kind nil should clear the current setting.
    SetGroupVersionKind(kind GroupVersionKind)
    // GroupVersionKind returns the stored group, version, and kind of an
    // object, or nil if the object does not expose or provide these fields.
    GroupVersionKind() GroupVersionKind
}
```

换句话说，Go中的Kubernetes对象是这样一种数据结构，可以：

- 返回一个GroupVersionKind和设置一个GroupVersionKind
- 可以被深拷贝

一个*深拷贝*是数据结构的克隆，使其不与原始对象共享任何内存。它被用于在代码中需要修改对象属性数据但又不对原始对象进行修改的场景。有关如何在Kubernetes中实现深层复制的详细信息，请参阅有关代码生成的[“全局标记”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#global-tags)。

简而言之，Kubernetes对象存储自身的类型并且允许对原对象进行克隆。

## TypeMeta

虽然`runtime.Object`只是一个接口，我们想知道它实际中的实现是怎样的。来自*k8s.io/api包的* Kubernetes对象通过嵌入*k8s.io/apimachinery/meta/v1包中*的`metav1.TypeMeta`结构来实现`schema.ObjectKind`接口中的getter和setter方法 ：

```go
// TypeMeta describes an individual object in an API response or request
// with strings representing the type of the object and its API schema version.
// Structures that are versioned or persisted should inline TypeMeta.
//
// +k8s:deepcopy-gen=false
type TypeMeta struct {
    // Kind is a string value representing the REST resource this object
    // represents. Servers may infer this from the endpoint the client submits
    // requests to.
    // Cannot be updated.
    // In CamelCase.
    // +optional
    Kind string `json:"kind,omitempty" protobuf:"bytes,1,opt,name=kind"`

    // APIVersion defines the versioned schema of this representation of an
    // object. Servers should convert recognized schemas to the latest internal
    // value, and may reject unrecognized values.
    // +optional
    APIVersion string `json:"apiVersion,omitempty"`
}
```

理解了这层概念，我们将用Go语言定义一个pod的数据结构，如下所示：

```go
// Pod is a collection of containers that can run on a host. This resource is
// created by clients and scheduled onto hosts.
type Pod struct {
    metav1.TypeMeta `json:",inline"`
    // Standard object's metadata.
    // +optional
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // Specification of the desired behavior of the pod.
    // +optional
    Spec PodSpec `json:"spec,omitempty"`

    // Most recently observed status of the pod.
    // This data may not be up to date.
    // Populated by the system.
    // Read-only.
    // +optional
    Status PodStatus `json:"status,omitempty"`
}
```

如您所见，`TypeMeta`被嵌入到了Pod结构中。此外，pod类型具有JSON标记，同时声明`TypeMeta`为内联。

###### 注意

这个`",inline"`标签实际上将与Golang JSON encoders / decoders一起使用：嵌入式结构将自动内联。

这在[YAML en / decoder *go-yaml / yaml中*](http://bit.ly/2ZuPZy2)是不同的，它在非常早期的Kubernetes代码中与JSON并行使用。我们从那时起继承了[内联标签](http://bit.ly/2IUGwcC)，但今天它只是文档而没有任何影响。

定义在 *k8s.io/apimachinery/pkg/runtime/serializer/yaml*包中的YAML序列化代码使用sigs.k8s.io/yaml包中编码和解码功能，这些双向的编码和解码大体上是将YAML转换为接口 `interface{}`，再通过JSON的编码和解码将其转换为Go语言数据结构。

这里是一个pod的YAML描述文件，Kubernetes用户会比较熟悉：[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#idm46336865870888)

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: example
spec:
  containers:
  - name: hello
    image: debian:latest
    command:
    - /bin/sh
    args:
    - -c
    - echo "hello world"; sleep 10000
```

版本对应的是TypeMeta.APIVersion字段`，资源类型对应的是`TypeMeta.Kind字段。

##### 核心小组因历史原因而异

Pods以及许多其他类型最早被添加到Kubernetes的核心组中- 现在我们也称为*遗留组* - 这个组没有名称，它的名称是空字符串表示。因此，对应的`apiVersion`只是“ `v1`”。

之后，API分组的概念被加入到Kubernetes中，并且以斜杠分隔的组名称被添加到`apiVersion`。在这种情况下`apps`，版本将是`apps/v1`。因此，现在看来`apiVersion`字段从名称看解读它描述的就不准确了; 它实际存储了API组名称和版本两个信息。这是由于历史原因造成的，当时定义`apiVersion`时只有核心组，而这些其他API组都还不存在。

运行[“创建和使用客户端”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)的示例以从集群中获取pod时，请注意客户端返回的pod对象实际上并未设置类型和版本集。基于`client-go`库编写的客户端程序有这样一个约定俗成的惯例，就是这些字段在内存中是空的，只有当它们被转换成到JSON或protobuf时，它们才会被自动填充实际值。然而，这是由客户端自动完成的，或者更准确地说，是由版本控制的序列化逻辑完成的。

##### 幕后花絮：GO TYPE类型，Packages，Kinds和Goup Names是如何关联起来的？

你可能想知道客户端如何知道该怎样填写`TypeMeta`中的类型和APIgroup 字段的。虽然这个问题起初听起来微不足道，但它不是：

- 看上去Yaml中定义的资源类型Kind只是Go语言中对应类型名称，貌似是通过反射从对象派生。这基本上是正确的 - 可能在99％的情况下 - 但也有例外（在[第4章中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)您将了解自定义资源，这就是个例外）。
- Yaml中的group看上去就是Go代码对应包名称（`apps`API group的类型在*k8s.io/api/apps*中声明）。这通常是一致的，但并非在所有情况下都一致：还记得上文核心组组名为空。另外，`rbac.authorization.k8s.io`组的类型位于*k8s.io/api/rbac中*，而不是*k8s.io/api/rbac.authorization.k8s.io中*。

如何填写`TypeMeta`各个字段，以及这其中涉及到的概念，将在[“Schema”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#scheme)中更详细地讨论。

换句话说，基于`client-go`开发的应用程序检查Golang类型的对象以确定当前的对象。这可能与其他框架的实现是不同，例如Operator SDK（请参阅[“Operator SDK”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#operator-sdk)）。

## ObjectMeta

除了`TypeMeta`之外，大多数顶级对象都有一个`metav1.ObjectMeta`类型的字段，同样来自*k8s.io/apimachinery/pkg/meta/v1*包：

```go
type ObjectMeta struct {
    Name string `json:"name,omitempty"`
    Namespace string `json:"namespace,omitempty"`
    UID types.UID `json:"uid,omitempty"`
    ResourceVersion string `json:"resourceVersion,omitempty"`
    CreationTimestamp Time `json:"creationTimestamp,omitempty"`
    DeletionTimestamp *Time `json:"deletionTimestamp,omitempty"`
    Labels map[string]string `json:"labels,omitempty"`
    Annotations map[string]string `json:"annotations,omitempty"`
    ...
}
```

JSON或YAML定义中，这些字段在*元数据*下。例如，对于上一个pod，`metav1.ObjectMeta`存储的结构为：

```yaml
metadata:
  namespace: default
  name: example
```

通常，它包含所有元数据的信息，如名称，命名空间，资源版本（不要与API组版本混淆），多个时间戳，以及众所周知的标签和注释都是`ObjectMeta`其中的一部分。有关字段的深入讨论，请参阅`ObjectMeta`[“类型剖析”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#anatomy-of-CRD-types)。

前面在[“乐观并发”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#optimistic-concurrency)讨论了资源版本。它几乎不会从`client-go`代码中读取或写入。但它是Kubernetes中使得整个系统运转的一个重要的组成部分。`resourceVersion`是`ObjectMeta`的一部分，因为嵌入`ObjectMeta`结构的每个对象，都对应于`etcd`一个键，而`resourceVersion`就是这个键对应的modified index。

## 规格Spec和状态Status

最后，几乎每个顶级对象都有一个`spec`和一个`status`部分。此约定来自Kubernetes API的声明性质：`spec`描述了用户需求、期望，而`status`描述的是该期望的结果，通常由系统中的控制器填充status中的相关信息。有关Kubernetes中控制器的详细讨论，请参阅[“控制器和operator”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#ch_controllers-operators)。

`spec`和`status`的这一约定惯例也有少数例外- 例如，核心组中的endpoint，或RBAC对象中的ClusterRole。

# 客户端集合

在[“创建和使用客户端”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)的示例中，我们看到`kubernetes.NewForConfig(config)`为我们提供了一个*客户端集合*。客户端集合提供对多个API组和资源的客户端的访问。在此情况下使用`kubernetes.NewForConfig(config)`，这个在*k8s.io/client-go/kubernetes*包中定义的方法，我们可以获得*k8s.io/api*包中定义的所有API组和资源。基本上也就是 Kubernetes API服务器提供的整套资源了。当然也还有少数例外情况，例如`APIServices`（对于聚合的API服务器）和`CustomResourceDefinition`（参见[第4章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)）未含在其中。

在[第5章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)，我们将解释如何从API类型定义（在本例中为*k8s.io/api*）来生成这些客户端集合。使用自定义API的第三方项目将不仅仅使用Kubernetes客户端集合。所有客户端集合的共同点是都会用到：REST配置（例如，`clientcmd.BuildConfigFromFlags("", *kubeconfig)`方法返回的值，如示例中所示）。

该客户端设置*k8s.io/client-go/kubernetes/typed中的*主界面，*Kubernetes*本地资源如下所示：

```go
type Interface interface {
    Discovery() discovery.DiscoveryInterface
    AppsV1() appsv1.AppsV1Interface
    AppsV1beta1() appsv1beta1.AppsV1beta1Interface
    AppsV1beta2() appsv1beta2.AppsV1beta2Interface
    AuthenticationV1() authenticationv1.AuthenticationV1Interface
    AuthenticationV1beta1() authenticationv1beta1.AuthenticationV1beta1Interface
    AuthorizationV1() authorizationv1.AuthorizationV1Interface
    AuthorizationV1beta1() authorizationv1beta1.AuthorizationV1beta1Interface

    ...
}
```

在这个接口中曾经有过无版本的方法 - 例如，`Apps() appsv1.AppsV1Interface`但是从基于Kubernetes 1.14的`client-go`11.0 开始，它们被弃用了。如前所述，应用程序非常明确地使用API组的版本是一种很好的做法。

##### 过去版本化的客户端和内部客户端

在过去，Kubernetes有所谓的*内部*客户。这些对象称为“内部”的对象使用了通用的内存版本，其中包含与线上版本之间的转换。

原本希望是从正在使用的实际API版本中抽象出控制器代码，并能够通过单行更改切换到另一个版本。实际上，实现转换代价巨大，额外复杂性以及此转换代码所需具备相关对象语义方面的知识量要求很高，从实践中这种模式不值得推荐。

此外，客户端和API服务器之间从未进行任何类型的自动协商。即使使用内部类型和客户端，控制器也是硬编码实现的。因此，对于使用版本化API的类型，在客户端和服务器之间的出现版本偏差时，采用内部类型的控制器将不再兼容。

在最近的Kubernetes版本中，大量代码被重写去掉了这些内部版本。今天，在*k8s.io/api*和*k8s.io/client-go*中已经没有内部版本了。

客户端集合还允许访问Discovery客户端（它被`RESTMappers`使用;请参阅[“REST Mapping”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#RESTMapping)和[“从命令行使用API”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-cli)）。

在每个`GroupVersion`方法中（例如`AppsV1beta1`），我们找到API组的资源 - 例如：

```go
type AppsV1beta1Interface interface {
    RESTClient() rest.Interface
    ControllerRevisionsGetter
    DeploymentsGetter
    StatefulSetsGetter
}
```

 `RESTClient`方法作为一个通用*REST客户端*，存在于每个资源的一个接口中，如：

```go
// DeploymentsGetter has a method to return a DeploymentInterface.
// A group's client should implement this interface.
type DeploymentsGetter interface {
    Deployments(namespace string) DeploymentInterface
}

// DeploymentInterface has methods to work with Deployment resources.
type DeploymentInterface interface {
    Create(*v1beta1.Deployment) (*v1beta1.Deployment, error)
    Update(*v1beta1.Deployment) (*v1beta1.Deployment, error)
    UpdateStatus(*v1beta1.Deployment) (*v1beta1.Deployment, error)
    Delete(name string, options *v1.DeleteOptions) error
    DeleteCollection(options *v1.DeleteOptions, listOptions v1.ListOptions) error
    Get(name string, options v1.GetOptions) (*v1beta1.Deployment, error)
    List(opts v1.ListOptions) (*v1beta1.DeploymentList, error)
    Watch(opts v1.ListOptions) (watch.Interface, error)
    Patch(name string, pt types.PatchType, data []byte, subresources ...string)
        (result *v1beta1.Deployment, err error)
    DeploymentExpansion
}
```

根据资源的范围 - 即，是集群还是命名空间作用域 - Getter接口（此处`DeploymentGetter`）可能有也可能没有`namespace`参数。

在`DeploymentInterface`中定义了访问资源的所有方法。其中大多数顾名思义，不过有些方法接下来我们一起重点关注一下。

## 状态的子资源：UpdateStatus

Deployment有一个所谓的*状态子资源*。这意味着`UpdateStatus`方法使用后缀为/status的一个额外HTTP路径。这样一来，路径*/ apis / apps / v1beta1 / namespaces / ns/ deployments /name*上的更新只能更改deployment的spec，而路径/ *apis / apps / v1beta1 / namespaces / ns/ deployments /name/ status*只能更改对象的状态status。这对于规范资源的更新操作，设置不同的权限非常有用一般情况规格spec（提供给人来操作）和状态status更新（由控制器操作完成）。

默认情况下，`client-gen`（参见[“client-gen Tags”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#clientgen-tags)）生成`UpdateStatus()`方法。该方法的存在并不能保证资源实际上支持子资源。这在我们使用CRD的[“子资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-subresources)时，这将非常重要。

## 列表和删除

`DeleteCollection` 允许我们一次删除一个命名空间下的多个对象。`ListOptions`这个参数允许我们定义应删除哪些对象使用*字段选择器*或*标签选择器*：

```go
type ListOptions struct {
    ...

    // A selector to restrict the list of returned objects by their labels.
    // Defaults to everything.
    // +optional
    LabelSelector string `json:"labelSelector,omitempty"`
    // A selector to restrict the list of returned objects by their fields.
    // Defaults to everything.
    // +optional
    FieldSelector string `json:"fieldSelector,omitempty"`

    ...
}
```

## Watch

`Watch` 提供了一个事件接口，用于接收对象的所有更改事件（如添加，删除和更新）。`watch.Interface`定义在*k8s.io/apimachinery/pkg/watch*包中的*内容*，如下所示：

```go
// Interface can be implemented by anything that knows how to watch and
// report changes.
type Interface interface {
    // Stops watching. Will close the channel returned by ResultChan(). Releases
    // any resources used by the watch.
    Stop()

    // Returns a chan which will receive all the events. If an error occurs
    // or Stop() is called, this channel will be closed, in which case the
    // watch should be completely cleaned up.
    ResultChan() <-chan Event
}
```

`watch`接口的结果通道返回三种事件：

```go
// EventType defines the possible types of events.
type EventType string

const (
    Added    EventType = "ADDED"
    Modified EventType = "MODIFIED"
    Deleted  EventType = "DELETED"
    Error    EventType = "ERROR"
)

// Event represents a single event to a watched resource.
// +k8s:deepcopy-gen=true
type Event struct {
    Type EventType

    // Object is:
    //  * If Type is Added or Modified: the new state of the object.
    //  * If Type is Deleted: the state of the object immediately before
    //    deletion.
    //  * If Type is Error: *api.Status is recommended; other types may
    //    make sense depending on context.
    Object runtime.Object
}
```

虽然直接使用这个接口很诱人，但实际上从Informers的角度看这是不鼓励的（参见[“线人和缓存”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#informers)）。Informers是此事件接口和带有索引查找的内存缓存的组合。这是比较最常见的watch使用方法。在Informers中，通过客户端首先调用`List`方法获取所有对象的集合（这些对象将作为缓存的基线），然后`Watch`不断更新缓存。它能够正确处理错误情况 - 可以从网络方面的错误或其他群集问题中自行恢复。

## 客户端扩展

`DeploymentExpansion` 实际上是一个空接口。过去被设计用于添加自定义客户端行为，但现在很难用于Kubernetes。替而代之，可以使用客户端生成器以声明方式添加自定义方法（请参阅[“client-gen标签”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#clientgen-tags)）。

需要注意，所有的这些方法`DeploymentInterface`既不会校验`TypeMeta`的字段`Kind`和`APIVersion`的有效性，也不需要在执行`Get()`和`List()`方法时设置这些字段（参见[“TypeMeta”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#TypeMeta)）。这些字段将在编码解码时被自动填充实际值。

## 客户选项

有必要来看看我们在创建客户端集合时可以设置哪些不同选项。在[“版本控制和兼容性”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#versioning-capability)之前的介绍中，我们看到我们可以切换到原生Kubernetes类型的protobuf协议格式。Protobuf比JSON（信息压缩和客户端和服务器的CPU负载）更有效，因此更受欢迎。

出于调试目的和指标的可读性，区分访问API服务器的不同客户端通常很有帮助。如需这样做，我们可以在REST配置中设置*用户代理*字段。默认值为`binary/version (os/arch) kubernetes/commit`; 例如，`kubectl`将使用这样的用户代理`kubectl/v1.14.0 (darwin/amd64) kubernetes/d654b49`。如果该模式不足以进行设置，则可以像这样定制：

```go
cfg, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
cfg.AcceptContentTypes = "application/vnd.kubernetes.protobuf,application/json"
cfg.UserAgent = fmt.Sprintf(
    "book-example/v1.0 (%s/%s) kubernetes/v1.0",
    runtime.GOOS, runtime.GOARCH
)
clientset, err := kubernetes.NewForConfig(cfg)
```

其他REST配置中经常被覆盖的值是客户端*速率限制*和*超时的值*：

```go
// Config holds the common attributes that can be passed to a Kubernetes
// client on initialization.
type Config struct {
    ...

    // QPS indicates the maximum QPS to the master from this client.
    // If it's zero, the created RESTClient will use DefaultQPS: 5
    QPS float32

    // Maximum burst for throttle.
    // If it's zero, the created RESTClient will use DefaultBurst: 10.
    Burst int

    // The maximum length of time to wait before giving up on a server request.
    // A value of zero means no timeout.
    Timeout time.Duration

    ...
}
```

该`QPS`值默认为`5`请求每秒，burst为`10`。

该timeout没有默认值，至少在客户端REST配置中没有。默认情况下，对于非长时间运行的请求，Kubernetes API服务器将在60秒后超时。长时间运行的请求例如watch请求，或者对子资源的请求例如*/ exec*请求，*/ portforward*或*/ proxy*等将不会有超时。

##### 优雅的关机和对连接错误的适应能力

请求可以分为长期运行和非长期运行。watch是长时间运行，同时`GET`，`LIST`，`UPDATE`等都是非长期运行。许多子资源（例如，用于日志流，exec，端口转发）也是长期运行的。

当重新启动Kubernetes API服务器时（例如，在更新期间），它会等待最多60秒才能正常关闭。在此期间，它可以响应非长时间运行的请求，然后终止。当API服务器终止时，长时间运行的请求（如正在进行的watch连接）将被切断。

非长时间运行的请求无论如何都会受到60秒的限制（然后它们会超时）。因此，从客户端的角度来看，关闭是优雅的。

通常，应用程序代码应始终提前考虑请求失败的情况，并且对这些非致命性错误进行响应。在分布式系统中，连接错误是正常的，没什么好担心的。但需要特别注意小心处理错误并从中恢复。

对于watch操作错误处理尤为重要。watch长时间运行，但它们可能随时出现故障。下一节对informers的描述中提供了一个围绕watch的实现，可以优雅地处理错误 - 它可以自动从断开连接的错误中恢复，创建出新的连接。而应用程序代码无需关注。

# Informers和缓存

[“客户端集合”中的](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#clientsets)客户端接口包括`Watch`方法，它提供了一个接口对事件作出反应，这些对象的事件有（添加，删除，更新）。Informers为最常见的watch用例提供了更高级别的编程接口：在内存中缓存和按名称或内存中的其他属性快速，索引查找对象。

每次控制器从API服务器访问对象的操作都会在系统上产生高负载。使用informers把对象缓存在内存中是解决此问题的方法。此外，informers几乎可以实时响应对象的变化，而不需要轮询请求。

[图3-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#informers-figure)显示了informers的概念模式; 具体来说：

- 从API服务器获取输入作为事件。
- 提供一个类似客户端的接口`Lister`，用于从内存缓存中列出对象。
- 为对应的添加事件，删除事件和更新事件，注册事件处理程序。
- 实现内存数据缓存。

![线人](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0305.png)

###### 图3-5。Informers

Informers还有高级错误处理行为：当长时间运行的watch连接发生故障时，它们会通过尝试启用另一个watch来接管事件流程，从而在不丢失任何事件的情况下从错误中恢复。如果中断很长，并且API服务器丢失了事件，因为`etcd`在新的监视请求成功之前可能会将它们从数据库中清除，则informers将 重新list所有对象。

与*relists一起的，还有一个可配置的重新*同步周期，用于内存缓存和业务逻辑之间的协调：每次经过此周期后，将为所有对象调用已注册的事件处理程序。通常以分钟为单位（例如，10或30分钟）。

###### 警告

重新同步单纯是在内存中，*不会触发对服务器的调用*。过去不是这样的，但[最终被改成这样了，](http://bit.ly/2FmeMge)因为watch机制的错误处理行为已经得到了优化和改进，可以不在需要relist操作。

所有这些高级且经过实战验证的错误处理行为也是推荐使用informer而不是直接使用`Watch()`方法的一个很好的理由。Informer在Kubernetes本身随处可见，它是Kubernetes API设计中的主要架构概念之一。

虽然informers比轮询更受欢迎，但它们会在API服务器上产生负载。一个可执行程序应该只为每个GroupVersionResource实例化一个informer。可以通过使用*共享的informer工厂*类，创建出共享的informers。

共享的informer工厂允许在应用程序中为相同的资源共享信息。换句话说，不同的控制循环可以使用相同的watch连接与与API服务器交互。例如，`kube-controller-manager`是Kubernetes集群的一个主要组件（参见[“API服务器”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server)）它内部有多达两位数控制器。但是对于每个资源来说（例如，pod），他们共享一个informer。

###### 提示

始终使用共享的informer工厂类来实例化informers。不要尝试手动实例化informers。这样做的额外开销很小，如果不使用共享informers的控制器程序可能在某个地方为同一资源打开多个watch连接。

有了REST配置（请参阅[“创建和使用客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)），使用客户端集合可以很方便的创建共享的informer工厂。informers由代码生成器生成，并作为`client-go`（k8s.io/client-go/informers）包中的一部分为标准Kubernetes资源而提供：

```go
import (
    ...
    "k8s.io/client-go/informers"
)
...
clientset, err := kubernetes.NewForConfig(config)
informerFactory := informers.NewSharedInformerFactory(clientset, time.Second*30)
podInformer := informerFactory.Core().V1().Pods()
podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(new interface{}) {...},
    UpdateFunc: func(old, new interface{}) {...},
    DeleteFunc: func(obj interface{}) {...},
})
informerFactory.Start(wait.NeverStop)
informerFactory.WaitForCacheSync(wait.NeverStop)
pod, err := podInformer.Lister().Pods("programming-kubernetes").Get("client-go")
```

该示例显示了如何获取pod的共享informer。

你可以看到informers允许添加三种情况的事件处理程序*添加*，*更新*和*删除*。这通常用于触发控制器的业务逻辑- 即再次处理某个对象（请参阅[“控制器和operator”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#ch_controllers-operators)）。这些处理程序通常只是将修改后的对象添加到工作队列中。

##### 其他事件处理程序和内部存储更新逻辑

不要将这些处理程序与informer内部存储库的更新逻辑（可通过示例最后一行中的lister访问）混淆。informer的逻辑是始终更新其内存缓存，但刚才描述的其他事件处理程序可以有别的操作，它是由拥有informer的应用程序来决定的。

另外，这里可以根据需要添加许多事件处理程序。之所以抽象出共享informer工厂的概念是因为这个逻辑在许多控制器中是很常见，每个控制循环都会注册事件处理程序然后将对象添加到它们自己的工作队列中。

注册处理程序后，必须启动共享的informers。在后台其实是通过Go协程来执行对API服务器的实际调用。`Start`方法（拥有一个控制停止的channel通道来控制生命周期）实际启动了这些Go协程，`WaitForCacheSync()`方法内部调用了客户端的`List`方法并等待第一次List调用完成。如果控制器逻辑执行的前提要求缓存对象全部被填充好，则此`WaitForCacheSync`调用至关重要，应该在同步调用返回后再向后执行。

通常，watch的事件接口在后台执行时会有一定的延迟滞后。在提前做过容量规划的程序中，这种滞后并不是很大。另外，建立指标metric来衡量这种滞后也是一种很好的做法。但是滞后确实是存在，所以应用程序代码逻辑中应该提前考虑，或加入相应的处理逻辑。

###### 警告

Informers的滞后可导致控制器通过`client-go`在API服务器上对资源状态的修改与informer一侧保存的资源状态出现竞态或不一致的情况。

如果控制器更改了对象，则同一进程中的informer必须等到相应的事件到达，然后将结果更新内存中。此过程需要一点时间，还有可能在前一个更改变为可见之前，另一个控制器触发启动了另一个控制循环来运行。

在该示例中，重新同步设置为间隔30秒，重新同步将影响到注册到`UpdateFunc`方法上的全部事件，此操作可以让控制器逻辑检测出资源的状态，并将现有状态协调到期望的状态。另外，可以通过比较`ObjectMeta.resourceVersion`字段，来区分出是不是自于重新同步操作的更新事件。

###### 注意

确定一个合适的的重新同步间隔取决于实际情况。例如，30秒非常短。在许多情况下，几分钟，甚至30分钟，也会是一个不错的选择。在最坏的情况下，30分钟意味着需要30分钟才能通过触发执行协调方法中的逻辑（例如，由于不正确的错误处理导致的信号丢失）。

另请注意，示例中最后一行`Get("client-go")`是访问的内存; 而不是访问API服务器。内存中存储的对象是无法直接修改。需要通过客户端集合来进行对资源的写操作。然后，informer将从API服务器获取事件并将最新的资源状态更新到内存存储中。

##### 永远不要改变informer的对象

记住从Listers传递给事件处理程序的任何对象都是由informer专有的，这一点非常重要。如果你以任何方式改变了它，这将引起informer中存储数据的不一致问题，而且一般难以调试。在更改对象之前，先进行深拷贝（请参阅[“Go中的Kubernetes对象”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#kube-objects)）。

一般来说：在修改对象之前，总是问自己谁拥有这个对象或其中的数据结构。根据经验：

- informer和listers拥有他们返回的对象。因此，消费者必须在修改对象之前进行深拷贝。
- 客户端返回的对象是归调用者拥有。
- 转换返回的共享对象。如果调用者拥有输入对象，name输出对象不归它拥有。

`NewSharedInformerFactory`示例中的informer构造函数将资源的所有对象缓存在存储的所有名称空间中。如果这对于应用程序来说太多了，那么有一个具有更大灵活性的替代构造函数：

```go
// NewFilteredSharedInformerFactory constructs a new instance of
// sharedInformerFactory. Listers obtained via this sharedInformerFactory will be
// subject to the same filters as specified here.
func NewFilteredSharedInformerFactory(
    client versioned.Interface, defaultResync time.Duration,
    namespace string,
    tweakListOptions internalinterfaces.TweakListOptionsFunc
) SharedInformerFactor

type TweakListOptionsFunc func(*v1.ListOptions)
```

这个方法允许我们能够指定一个命名空间，并传递一个`TweakListOptionsFunc`方法，这将允许对`ListOptions`数据结构进行修改，从而影响到客户端List`和`Watch方法调用的结果。例如，它可用于设置*标签选择器*或*字段选择器*。

informers是控制器的重要组成之一。在[第6章中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#ch_operator-solutions)我们将看到`client-go`基于典型的控制器的外观。在客户端和informers之后，第三个主要构成是工作队列。我们现在来看看吧。

## 工作队列

一个工作队列是一种数据结构。您可以按队列预定义的顺序添加元素并从队列中取出元素。形式上，这种队列称为*优先级队列*。`client-go`为构建控制器提供了一个强大的实现在[*k8s.io/client-go/util/workqueue*](http://bit.ly/2IV0JPz)中。

准确地说，那个工作队列的包目录下，包含许多可供用于不同目的的实现。这些实现的公有接口如下所示：

```go
type Interface interface {
    Add(item interface{})
    Len() int
    Get() (item interface{}, shutdown bool)
    Done(item interface{})
    ShutDown()
    ShuttingDown() bool
}
```

这里通过`Add(item)`添加一个条目，`Len()`返回长度，`Get()`返回一个具有最高优先级的条目（并且它会阻塞直到取到一个条目）。当控制器完成处理后，`Get()`方法返回的每个条目都需要再执行一次`Done(item)`调用。同时，重复`Add(item)`将条目标记为脏，以便在`Done(item)`被调用时对其进行读取。

以下队列类型派生自此通用接口：

- `DelayingInterface`可以在一段时间之后添加条目。这样可以更容易在故障后延迟一段时间再重新将条目入排而不会陷入当前状态下的死循环中：

  ```go
  type DelayingInterface interface {
      Interface
      // AddAfter adds an item to the workqueue after the
      // indicated duration has passed.
      AddAfter(item interface{}, duration time.Duration)
  }
  ```

- `RateLimitingInterface`限速队列。它扩展了`DelayingInterface:`

  ```go
  type RateLimitingInterface interface {
      DelayingInterface
  
      // AddRateLimited adds an item to the workqueue after the rate
      // limiter says it's OK.
      AddRateLimited(item interface{})
  
      // Forget indicates that an item is finished being retried.
      // It doesn't matter whether it's for perm failing or success;
      // we'll stop the rate limiter from tracking it. This only clears
      // the `rateLimiter`; you still have to call `Done` on the queue.
      Forget(item interface{})
  
      // NumRequeues returns back how many times the item was requeued.
      NumRequeues(item interface{}) int
  }
  ```

  这里最有趣的是`Forget(item)`方法：它代表条目不再入队。通常，在在成功处理条目后调用它。

  速率限制算法可以传递给构造函数`NewRateLimitingQueue`。在同一个包中定义了几个速率限制器，例如`BucketRateLimiter`， `ItemExponentialFailureRateLimiter`， `ItemFastSlowRateLimiter`和 `MaxOfRateLimiter`。有关更多详细信息，请参阅包文档。大多数控制器只使用`DefaultControllerRateLimiter() *RateLimiter`功能，它可以提供：

  - 指数方式的限速算法，当每次遇到错误延迟时间会翻倍，时间延迟的范围是5毫秒到1,000秒
  - 最大速率为每秒10项，burst为100项

根据您的实际情况，您可以自定义值。对于某些控制器应用，每个条目最大延迟1,000秒也是可以的。

# 深入API Machinery

API Machinery存储库实现了Kubernetes类型type系统的基础。但究竟这种类型系统指什么呢？首先说说什么类型呢？

术语 *类型 type* *实际上并不存在于API Machinery的术语中。相反，它指的是*资源类型kinds。

## Kinds

资源类型分为API组和版本，正如我们在[“API术语”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#terminology)已经看到的那样。因此，API Machinery存储库中的核心术语是GroupVersionKind，简称*GVK*。

在Go中，每个GVK对应一个Go类型。反之，一个Go类型可以属于多个GVK。

Kinds没与HTTP路径不是一对一关系。有许多资源类型都有HTTP REST访问路径，用于访问给定类型的对象。但是也有没有任何HTTP访问路径的资源类型（例如，[*admission.k8s.io/*](http://bit.ly/2XJXBQD) v1beta1.AdmissionReview，用于调用webhook）。还有许多资源类型有过个访问路径可以返回 - 例如，[*meta.k8s.io / v1.Status*](http://bit.ly/31Ktjvz)，由所有资源访问后都会返回Status用来报告状态，如错误。

通过约定，资源类型采用[驼峰命名方式](http://bit.ly/31IqMSC),并且采用[单数](http://bit.ly/31IqMSC)。根据具体情况，它们的具体格式不同。对于CustomResourceDefinition自定义资源类型，它必须是DNS路径标签（RFC 1035）。

## Resources

与“kinds”并行，正如我们在[“API术语”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#terminology)看到的那样，存在*资源*的概念。资源分为组和版本，产生了术语*GroupVersionResource*或*简称GVR*。

每个GVR对应一个HTTP（基本）路径。GVR用于标识Kubernetes API的REST访问路径。例如，GVR *apps / v1.deployments*映射到*/ apis / apps / v1 / namespaces / namespace/ deploymentments*。

客户端库使用此映射来构造访问GVR的HTTP路径。

##### 了解资源是命名空间还是集群范围

您必须知道GVR是命名空间还是群集范围才能知道HTTP路径。例如，deployment属于命名空间，因此将命名空间作为其HTTP路径的一部分。其他GVR，例如*rbac.authorization.k8s.io/v1.clusterroles*，是集群范围的; 例如，可以在*apis / rbac.authorization.k8s.io / v1 / clusterroles中*访问集群角色。

通过惯例，资源是小写和复数，通常对应于并行类型的多个单词。它们必须符合DNS路径标签格式（RFC 1025）。由于资源会直接映射到HTTP路径，因此这是可以理解的。

## REST Mapping

将GVK映射到GVR称为*REST映射*。

 `RESTMapper`是一个[Golang接口](http://bit.ly/2Y7wYS8)，使我们能够将GVK返回成一个GVR：

```go
RESTMapping(gk schema.GroupKind, versions ...string) (*RESTMapping, error)
```

`RESTMapping`类型如下所示：

```go
type RESTMapping struct {
    // Resource is the GroupVersionResource (location) for this endpoint.
    Resource schema.GroupVersionResource.

    // GroupVersionKind is the GroupVersionKind (data format) to submit
    // to this endpoint.
    GroupVersionKind schema.GroupVersionKind

    // Scope contains the information needed to deal with REST Resources
    // that are in a resource hierarchy.
    Scope RESTScope
}
```

此外， `RESTMapper`还提供了许多便利功能：

```go
// KindFor takes a partial resource and returns the single match.
// Returns an error if there are multiple matches.
KindFor(resource schema.GroupVersionResource) (schema.GroupVersionKind, error)

// KindsFor takes a partial resource and returns the list of potential
// kinds in priority order.
KindsFor(resource schema.GroupVersionResource) ([]schema.GroupVersionKind, error)

// ResourceFor takes a partial resource and returns the single match.
// Returns an error if there are multiple matches.
ResourceFor(input schema.GroupVersionResource) (schema.GroupVersionResource, error)

// ResourcesFor takes a partial resource and returns the list of potential
// resource in priority order.
ResourcesFor(input schema.GroupVersionResource) ([]schema.GroupVersionResource, error)

// RESTMappings returns all resource mappings for the provided group kind
// if no version search is provided. Otherwise identifies a preferred resource
// mapping for the provided version(s).
RESTMappings(gk schema.GroupKind, versions ...string) ([]*RESTMapping, error)
```

这里，部分GVR意味着并非所有字段都被设置。例如，假设您输入`kubectl get pods`。在这种情况下，没有指定组和版本信息。一个`RESTMapper`在有足够的信息情况下可以设法将其映射成`版本为v1 Pods`。

对于前面的deployment示例，`RESTMapper`将把[*apps / v1.Deployment*](http://bit.ly/2IujaLU)映射到*apps / v1.deployments*作为一个命名空间内的资源。

`RESTMapper`接口有多种不同的实现方式。对于客户端应用程序最重要的一个是基于discovery的[`DeferredDiscoveryRESTMapper`](http://bit.ly/2XroxUq) 位于包*k8s.io/client-go/restmapper*，它使用来自Kubernetes API服务器的发现信息来动态构建REST映射。它还可以与非核心资源（如自定义资源）一起使用。

## Scheme

最后一个我们希望大家了解的Kubernetes系统的核心概念是[*scheme*](http://bit.ly/2N1PGJB)，它位于*k8s.io/apimachinery/pkg/runtime*包。

scheme将Golang世界与独立于具体实现的GVKs联系起来。scheme的主要特征是将Golang类型映射到可能的GVK：

```go
func (s *Scheme) ObjectKinds(obj Object) ([]schema.GroupVersionKind, bool, error)
```

正如我们看到[“Kubernetes Objects in Go”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#kube-objects)，通过schema.ObjectKind方法返回一个object对象，一个object可以通过GetObjectKind() 方法返回它的api group和资源类型信息。但是，这些值在大多数时间都是空的，因此对于识别对象而言是无用的。

相反，scheme通过反射获取给定对象的Golang类型，并将其映射到Golang类型已注册GVK。当然，必须首先将Golang类型注册到scheme中：

```go
scheme.AddKnownTypes(schema.GroupVersionKind{"", "v1", "Pod"}, &Pod{})
```

scheme不仅用于注册Golang类型及它对应的GVK，还用于存储转换函数和默认操作（参见[图3-6](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#scheme-spider)）。我们将在[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)中更详细地讨论转换和默认操作。它也是实现编码和解码的数据来源。

![该方案将Golang数据类型与GVK，转换和违约者联系起来](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0306.png)

###### 图3-6。scheme将Golang数据类型与GVK，转换和默认操作关联起来

对于Kubernetes核心类型，在*k8s.io/client-go/kubernetes/scheme*包[中的`client-go`客户端集合中](http://bit.ly/2FkXDn2)有一个[预定义的scheme](http://bit.ly/2FkXDn2)，所有类型都已预先注册。实际上，每个客户端集都由 `client-gen`代码生成器生成（参见[第5章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)）在生成的scheme子包中，包含客户端集合中所有组和版本对应的资源类型。

我们深入探讨了API Machinery的概念。如果你只能记得关于这些概念的一件事，那请记住下图[如图3-7所示](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#scheme-restmapper-http-path)。

![从Golang类型到GVK到GVR再到HTTP路径 - 简而言之，API机制](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0307.png)

###### 图3-7。从Golang类型到GVK到GVR再到HTTP路径 

# Vendoring

我们本章在看到*k8s.io/client-go*，*k8s.io/api*和*k8s.io/apimachinery*包的使用是Golang进行 Kubernetes编程中的核心部分。Golang采用*vendoring*来管理第三方应用程序的源代码库。

Vendoring在Golang社区中还在不断的变化。在撰写本文时，有几个常见的包依赖管理工具，例如*godeps*，*dep*和*glide*。与此同时，Go 1.12正在支持Go Modules，modules可能会成为未来Go社区的标准包管理方式，但目前尚未在Kubernetes生态系统中完全用起来。

现在大多数项目都使用`dep`或`glide`。Kubernetes本身*github.com/kubernetes/kubernetes*在1.15开发周期中切换到了的Go module。以下注释与所有这些销售工具相关。

*k8s.io/**存储库中主要使用了*Godeps / Godeps.json文件。重要的是要强调对于切换依赖管理工具可能会破坏现有的功能。

有关*k8s.io/client-go，k8s.io* / *api*和*k8s.io/apimachinery*的已发布tag彼此之间是兼容的，请参阅[“客户端库”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go)。

## Glide

使用 `glide`的项目，glide可以读取*Godeps / Godeps.json*文件内容在依赖有变化的时候。事实证明这非常可靠：开发人员只需要声明正确的*k8s.io/client-go*版本，`glide`会自动选择正确版本的*k8s.io/apimachinery，k8s.io* / *api*和其他依赖项。

对于GitHub上的一些项目，*glide.yaml*文件可能如下所示：

```yaml
package: github.com/book/example
import:
- package: k8s.io/client-go
  version: v10.0.0
...
```

使用 glide install -v`将“k8s.io/client-go”及其依赖项下载到本地vendor目录中。这里，`-v`意味着从vendor/库中删除已存在的包。这正是是我们希望的。

如果您通过编辑*glide.yaml*更新`client-go`到新版本，`glide update -v`将再次以正确的版本下载新的依赖项。

## DEP

`dep`通常被认为比glide更强大、更先进。长期以来它被视为继任者`glide`的生态系统，似乎注定要成为Go 的依赖管理工具。在撰写本文时，其未来尚不清楚，Go module似乎也是未来的方向。

在`client-go`上下文中，了解以下几个限制非常重要`dep`：

- `dep`在第一次运行时会读取*Godeps / Godeps.json* 当执行`dep init`时。
- `dep`在以后的调用中不会读取*Godeps / Godeps.json* 如，执行`dep ensure -update`。

这意味着在*Godep.toml中*更新版本`client-go`时，依赖关系的解析可能是错误的。这点不太好，因为它要求开发人员显式地并且通常手动声明*所有*依赖项。

一个可用的*Godep.toml*文件如下所示：

```toml
[[constraint]]
  name = "k8s.io/api"
  version = "kubernetes-1.13.0"

[[constraint]]
  name = "k8s.io/apimachinery"
  version = "kubernetes-1.13.0"

[[constraint]]
  name = "k8s.io/client-go"
  version = "10.0.0"

[prune]
  go-tests = true
  unused-packages = true

# the following overrides are necessary to enforce
# the given version, even though our
# code does not import the packages directly.
[[override]]
  name = "k8s.io/api"
  version = "kubernetes-1.13.0"

[[override]]
  name = "k8s.io/apimachinery"
  version = "kubernetes-1.13.0"

[[override]]
  name = "k8s.io/client-go"
  version = "10.0.0"
```

###### 警告

*Gopkg.toml*文件不但要明确指定k8s.io/apimachinery*和*k8s.io/api的版本，还需要指定它们的覆盖信息。在项目初期，即使没有从这两个存储库中显式导入包的情况下，这也是必需的。在这种情况下，如果没有加入覆盖的信息`dep`会忽略开头的约束，开发人员会碰到依赖报错的情况。

甚至*Gopkg.toml*文件中配置的信息在技术上说也不完全正确，因为它不完整，它没有声明`client-go`库所需其他依赖。一旦`client-go` 依赖的上游库出现了不兼容，就会产生新的依赖问题，因此，如果您使用`dep`依赖项管理，需要注意这一点。

## Go Modules

Go模块是Golang中依赖管理的未来。它们在Go 1.11中得到[初步支持](http://bit.ly/2FmBp3Y)，并进一步稳定在1.12。许多命令，如`go run`和`go get`，通过设置`GO111MODULE=on`环境变量来使用Go模块。在Go 1.13中，这将是默认设置。

Go模块由项目根目录中的*go.mod*文件驱动。以下是[第8章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)*github.com/programming-kubernetes/pizza-apiserver*项目的*go.mod*文件的摘录：

```shell
module github.com/programming-kubernetes/pizza-apiserver

require (
    ...
    k8s.io/api v0.0.0-20190222213804-5cb15d344471 // indirect
    k8s.io/apimachinery v0.0.0-20190221213512-86fb29eff628
    k8s.io/apiserver v0.0.0-20190319190228-a4358799e4fe
    k8s.io/client-go v2.0.0-alpha.0.0.20190307161346-7621a5ebb88b+incompatible
    k8s.io/klog v0.2.1-0.20190311220638-291f19f84ceb
    k8s.io/kube-openapi v0.0.0-20190320154901-c59034cc13d5 // indirect
    k8s.io/utils v0.0.0-20190308190857-21c4ce38f2a7 // indirect
    sigs.k8s.io/yaml v1.1.0 // indirect
)
```

`client-go` `v11.0.0`匹配的Kubernetes 1.14，更早版本没有对Go模块的明确支持。如前面的示例所示，仍然可以将Go module与Kubernetes库一起使用。

只要`client-go`和其他Kubernetes存储库不提供go.mod*文件（至少在Kubernetes 1.15之前），只有通过手动方式选择正确的依赖版本。也就是说，你需要在`client-go`中从*Godeps / Godeps.json*文件中自己匹配的依赖的版本。

有一点需要注意，前一个示例中有的依赖的版本号是根据当前代码的tag计算的模拟版本号，还有的是当找不到代码tag时采用`v0.0.0`作前缀。不太好用的地方是，手动修改了依赖的引用版本后，再下次执行Go module命令又将替换为伪版本号。

`client-go` `v12.0.0`匹配Kubernetes 1.15，我们发布了一个*go.mod*文件只用了Go module没有使用其他依赖管理工具（参见[相应的提案文档](http://bit.ly/2IZ9MPg)）。文中的*go.mod*文件包含所有依赖项，您可以直接使用该go.mod文件。在以后的版本中，也可能会更改版本标记的方案，比如将目前伪版本替换为它们semver标签。但在撰写本文时，尚未完全决定或实现。

# 总结

在本章中，我们的重点介绍了Go中的Kubernetes编程接口。我们讨论了访问众所周知的核心类型的Kubernetes API，即每个Kubernetes集群附带的标准API对象。

理解了这个，我们已经介绍了Kubernetes API的基础知识及其在Go中的表示。现在，我们已准备好继续讨论自定义资源这一主题，它是理解operator的前提。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#idm46336866123400-marker)请参阅[ “API术语”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#terminology)。

[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#idm46336865870888-marker) `kubectl` `explain` `pod`允许您在API服务器中查询对象的模式，包括字段文档。
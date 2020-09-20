# 第5章自动生成代码

在本章中，您将学习如何在Go项目中使用Kubernetes代码生成器编写自定义资源。代码生成器在标准Kubernetes资源的实现中经常使用，我们将在这里使用相同的生成器。

# 为什么使用代码生成

Go是一种简单的语言设计。它缺乏更高级甚至类元编程的机制，即以通用（类型无关）的方式表达不同数据类型的算法。在Go语言中是通过使用外部代码生成器来实现的。

在Kubernetes开发过程的早期阶段，随着更多资源被添加到系统中，必须重写越来越多的代码。代码生成使得这项工作变得更加容易。最早使用的是[Gengo库](http://bit.ly/2L9kwNJ)，后来，基于Gengo，[*k8s.io / code-generator*](http://bit.ly/2Kw8I8U)被开发出来作为一整套代码生成器。我们将在以下部分中使用这些生成器生成CR。

# 调用生成器

通常，代码生成器在每个控制器项目中以大致相同的方式调用。只有包，组名和API版本不同。调用*k8s.io/code-generator/generate-groups.sh*或像*hack / update-codegen.sh*这样的bash脚本是创建CR对应的Go类型代码的最简单方法（参见[本书的GitHub存储库）](http://bit.ly/2J0s2YL)）。

注意，由于一些特殊要求和历史原因，某些项目要求直接调用代码生成器可执行文件。一般情况下，为CR创建控制器，从*k8s.io/code-generator*仓库调用*generate-groups.sh*脚本是最简单直接的方法：

```shell
$ vendor/k8s.io/code-generator/generate-groups.sh all \
    github.com/programming-kubernetes/cnat/cnat-client-go/pkg/generated
    github.com/programming-kubernetes/cnat/cnat-client-go/pkg/apis \
    cnat:v1alpha1 \
    --output-base "${GOPATH}/src" \
    --go-header-file "hack/boilerplate.go.txt"
```

这里，`all`的含义为创建 CR所需的四类代码：

- `deepcopy-gen`

  生成`func` `(t *T)` `DeepCopy()` `*T`和`func` `(t *T)` `DeepCopyInto(*T)`方法。

- `client-gen`

  创建类型化的客户端集合。

- `informer-gen`

  为CR创建informer，提供基于事件的接口来处理服务器上CR的更改事件。

- `lister-gen`

  为CR创建lister，一个只读缓存层，用于`GET`和`LIST`请求。

最后两个是创建控制器的基础（参见[“控制器和operator”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#ch_controllers-operators)）。这四个代码生成器通过采用与标准Kubernetes控制器相同的代码生成机制，为用户创建功能齐全的用于生产级别的控制器提供了强大的基础保障。

###### 注意

*k8s.io/code-generator中*还有一些生成器，主要用于其他场景。例如，如果您要构建自己的聚合API服务器（请参阅[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)），除了版本化类型之外，您还需要创建使用内部类型，并且必须定义默认函数。您可以通过从*k8s.io/code-generator*调用[*generate-internal-groups.sh*](http://bit.ly/2L9kSE3)脚本来访问这两个生成器，它们与此相关：

- `conversion-gen`

  创建用于内部和外部类型转换的函数 。

- `defaulter-gen`

  处理字段默认值。

现在让我们详细看看`generate-groups.sh`脚本的参数：

- 第二个参数是生成的clients，listers和informer的父包名。
- 第三个参数是API 资源类型定义所在的基础包的名称。
- 第四个参数是以空格分隔的列表，其中每一项为一个API组和版本信息。
- `--output-base` 作为标志传递给所有生成器，以定义找到包的根目录。
- `--go-header-file` 指定版权文件，使我们能够将版权标题放入生成的代码中。

某些生成器，例如`deepcopy-gen`，直接在API组包内创建文件。被生成的文件遵循统一的命名规则都以*zz_generated*作为前缀，这便于将它们从版本控制中排除（例如，通过*.gitignore*文件），由于目前代码生成器相关的Go工具开发的还不是很完善，大多数项目会对生成的文件进行检查。[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#idm46336860111320)

项目[*k8s.io/sample-controller*](http://bit.ly/2UppsTN)介绍了一种控制器的开发模式- `sample-controller`作为一种新的模式被推荐来代替Kubernetes内置的控制器模式 - 开发的第一步将从调用以下脚本生成代码开始：

```shell
$ hack/update-codegen.sh
```

`cnat`的例子我们将采用`sample-controller开发模式结合client-go`的方式进行[“详见sample-controller例子”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#cnat-client-go)。

###### 提示

通常，除了[`hack/update-codegen.sh`](http://bit.ly/2J0s2YL)脚本之外，我们还会见到另一个名为[`hack/verify-codegen.sh`](http://bit.ly/2IXUWsy)脚本。

这个脚本会调用`hack/update-codegen.sh`脚本，首先检查生成的文件是已经被更改，只要有一个文件对应的相关代码发生了改变，脚本会返回非零代码，并终止。

这在持续集成（CI）脚本中非常有用：如果开发人员意外修改了文件或者文件刚刚过时，CI会发现并报告。

# 使用标记控制生成器

虽然代码生成器的某些行为是通过前面描述的命令行标志控制的（例如指定要处理的包路径），但是更多属性是通过Go文件中的*标记*控制的。标记是一种特殊格式的Go语言注释，格式如下：

```go
// +some-tag
// +some-other-tag=value
```

有两种标记：

- 全局标记定义在文件*doc.go*中，定义的位置在`package`定义的上方
- 局部标记定义在每个类型声明的上方（例如，在结构体定义之上）

根据标记规则，注释的位置非常重要。

##### 准确地参照示例（包括注释块位置）

有许多标记的注释必须位于类型定义的上方（或者全局标记必须在包定义的上方），然而另外一些标记注释必须与类型定义或包定义分开（必须用空行分隔）。例如：

```go
// +second-comment-block-tag

// +first-comment-block-tag
type Foo struct {
}
```

产生这些差别主要是出于历史原因：Kubernetes中的API文档生成器无法区别普通文档注释和用于生成代码的标记注释，它仅会将第一个注释块导出到文档中。因此，位于第一个注释块中的标记将出现在API HTML文档中。

虽然代码生成器版本都有所改进，但是在标记的处理逻辑方面并不总是一致的，而且错误处理也没有那么完善。所以，在参照示例时尽可能与例子中的代码保持一致，例如，空行可能很重要。

## 全局标记

全局标记被写入包定义的*doc.go*文件中。通常pkg / apis / group/ version/doc.go文件的内容如下所示：

```go
// +k8s:deepcopy-gen=package

// Package v1 is the v1alpha1 version of the API.
// +groupName=cnat.programming-kubernetes.info
package v1alpha1
```

文件的第一行将告诉deepcopy-gen生成器，`默认情况下为该包中的每个类型创建深拷贝方法。如果某些类型不需要深拷贝方法，可以使用局部标记对该类型关闭深拷贝`// +k8s:deepcopy-gen=false`。如果您不启用包范围的深拷贝，则需要通过以下方式为每个所需的类型单独设置深拷贝`// +k8s:deepcopy-gen=true`。

第二个标记`// +groupName=example.com`定义了API组的完全限定名称。如果Go代码父包名称与组名称不一致，则需要通过此标记指明。

这个样例文件实际上来自[`cnat client-go`示例的*pkg / apis / cnat / v1alpha1 / doc.go*文件](http://bit.ly/2L6M9ad)（请参阅[“详见sample-controller例子”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#cnat-client-go)）。示例中`cnat`是父包，`cnat.programming-kubernetes.info`是组名。

使用`// +groupName`标记，客户端代码生成器（请参阅[“通过client-gen创建的类型化客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#clientgen-client)）将使用正确的HTTP路径*/apis/foo.project.example.com*生成客户端。除此`+groupName`之外，还有一个名为`+groupGoName`的标记，可以用来自定义Go语言中对应的变量名称和结构体名称。例如，默认情况下，生成器将使用首字母大写的方式来标识，对应到示例中为`Cnat`。对于“Cloud Native At。”，可能我们期望使用标识符CNAt，这就可以通过标记 `// +groupGoName=CNAt`来实现，`（虽然我们在这个例子中没有这样做`），本例中client-gen生成的结果如下所示：

```go
type Interface interface {
    Discovery() discovery.DiscoveryInterface
    CNatV1() atv1alpha1.CNatV1alpha1Interface
}
```

## 局部标记

局部标记直接写在API类型之上或其上方的第二个注释块中。以下是[cnat示例](http://bit.ly/31QosJw)的*types.go*文件中的主要类型：

```go
// AtSpec defines the desired state of At
type AtSpec struct {
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"`
    // Command is the desired command (executed in a Bash shell) to be executed.
    Command string `json:"command,omitempty"`
    // Important: Run "make" to regenerate code after modifying this file
}

// AtStatus defines the observed state of At
type AtStatus struct {
    // Phase represents the state of the schedule: until the command is executed
    // it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"`
    // Important: Run "make" to regenerate code after modifying this file
}

// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// At runs a command at a given schedule.
type At struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   AtSpec   `json:"spec,omitempty"`
    Status AtStatus `json:"status,omitempty"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// AtList contains a list of At
type AtList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []At `json:"items"`
}
```

接下来，我们将围绕这个示例的标记来讲解。

###### 提示

在本例中，API文档位于第一个注释块中，而我们将标记放入第二个注释块中。如果您使用某种工具提取Go文档注释，这样会区分标记注释和文档注释，标记注释不会进入API文档。

## deepcopy-gen标记

生成深拷贝方法，通常会在全局标记中设置`// +k8s:deepcopy-gen=package`标记设置为所有类型启用深拷贝（请参阅[“全局标记”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#global-tags)），如果某些类型不需要生成深拷贝方法，可以在局部标记中设置为针对该类型不开启深拷贝。

例如，如果我们在API类型包中有一个帮助类结构体定义（通常为了保持API包简洁，不建议这样做），这时必须禁用生成Helper类的深拷贝代码：

```go
// +k8s:deepcopy-gen=false
//
// Helper is a helper struct, not an API type.
type Helper struct {
    ...
}
```

## runtime.Object和DeepCopyObject

这是一个特殊的深拷贝标记，我们来详细解释一下：

```go
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
```

在[“Kubernetes Objects in Go”中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#kube-objects)我们知道`runtime.Object`s必须实现 runtime.Object`接口的`DeepCopyObject()方法。原因是Kubernetes中的通用代码必须能够创建对象的深拷贝。这个标记帮我来达成。

##### 历史背景

在1.8之前，在scheme实现中（参见[“scheme”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#scheme)）也保留了对类型相关的的深拷贝方法的引用，它通过基于反射方式的深拷贝实现。这两种机制导致出现了许多重要的错误但是很难被发现原因。因此，Kubernetes后来采用了静态对象深拷贝方式，为`runtime.Object`接口中的`DeepCopyObject`方法生成深拷贝方法。

`DeepCopyObject()`方法调用生成的`DeepCopy`方法。生成的DeepCopy方法根据类型的不同，会有很多（`DeepCopy()` `*T` 因T不同而各异）。DeepCopyObject的签名DeepCopyObject() runtime.Object：

```go
func (in *T) DeepCopyObject() runtime.Object {
    if c := in.DeepCopy(); c != nil {
        return c
    } else {
        return nil
    }
}
```

将局部标记`//+k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object`放在顶级类型的Go结构体上方，这将告诉`deepcopy-gen`代码生成器为runtime.Object对象创建这样一个DeepCopyObject()方法。

###### 提示

在前面的例子中，`At`和`AtList`都是顶级类型，因为它们都实现了`runtime.Object`接口。

根据经验，顶级类型将嵌入`metav1.TypeMeta`类型。

如果我们在API类型定义中，将类型属性字段定义成接口类型，那么这种情况下，这个接口类型也是需要支持深拷贝的：

```go
type SomeAPIType struct {
  Foo Foo `json:"foo"`
}
```

如我们所知，API类型必须是可深拷贝的，因此`Foo`字段也必须支持深拷贝。如果我们希望采取一种通用的方式，而不是在Foo接口中增加DeepCopyFoo() Foo 深拷贝方法，该如何实现呢？

```go
type Foo interface {
    ...
    DeepCopyFoo() Foo
}
```

这种情况下，可以使用上述类似的deepcopy-gen标记：

```go
// +k8s:deepcopy-gen:interfaces=<package>.Foo
type FooImplementation struct {
    ...
}
```

在Kubernetes源码中也有一些除了`runtime.Object之外的例子：

```go
// +k8s:deepcopy-gen:interfaces=.../pkg/registry/rbac/reconciliation.RuleOwner
// +k8s:deepcopy-gen:interfaces=.../pkg/registry/rbac/reconciliation.RoleBinding
```

## 客户端标记

这节介绍一些用于控制`client-gen`的标记，有一个标记在之前`At`和`AtList`中我们已经见到过：

```go
// +genclient
```

它告诉`client-gen`为这个类型创建一个客户端（这个标记是可选的）。请注意，在`List`API对象的类型定义上是不需要加也不能加这个标记的。

在我们的`cnat`示例中，我们使用*/ status*子资源并使用`UpdateStatus`客户端的方法更新CR的状态（请参阅[“状态子资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#status-subresource)）。如果对于没有状态字段或没有将spec和状态拆分的CR的实例。在这些情况下，可以指定以下标记将避免生成`UpdateStatus()`方法：

```go
// +genclient:noStatus
```

###### 警告

如果没有明确使用这个// +genclient:noStatus标记，`client-gen`默认就会生成`UpdateStatus()`方法。但是，关于spec-status拆分的另外重要的一点是，只有在CustomResourceDefinition定义中实际启用了*/ status*子资源时，spec-status拆分才有效（请参阅[“子资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-subresources)）。

单独在客户端中生成UpdateStatus方法可能是没有效果的。还需要配合相关的定义一起来正确配置。

客户端代码生成器必须选择正确的HTTP路径，可以是命名空间的资源，也可以时集群范围的资源。对于集群范围的资源，您必须使用标记：

```go
// +genclient:nonNamespaced
```

默认会生成命名空间作用域的客户端。同样，这个设置也要与CRD定义中的作用域范围设置相匹配。对于特殊用途的客户端，您可能还希望详细控制生成的HTTP方法。您可以使用几个标记来操作，例如：

```go
// +genclient:noVerbs
// +genclient:onlyVerbs=create,delete
// +genclient:skipVerbs=get,list,create,update,patch,delete,watch
// +genclient:method=Create,verb=create,
// result=k8s.io/apimachinery/pkg/apis/meta/v1.Status
```

前三个可以顾名思义，但最后一个需要一些解释。

上面这些标记最终给类型定义了创建的HTTP方法，它的返回值不是类型本身，而是返回一个 `metav1.Status`结构的对象。对于这个例子中的CR来说，这没有实际意义，但在实际场景中，通过以上几种标记方式，用户可以扩展出符合自己需求的HTTP资源，由API服务器提供服务（参见[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)）。

`// +genclient:method=`标记是一种常见的扩展资源的方法。在[“Scale subresource”中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#scale-subresource)我们描述了如何为CR启用*/ scale*子资源。以下是通过标记创建相应的客户端的方法：

```go
// +genclient:method=GetScale,verb=get,subresource=scale,\
//    result=k8s.io/api/autoscaling/v1.Scale
// +genclient:method=UpdateScale,verb=update,subresource=scale,\
//    input=k8s.io/api/autoscaling/v1.Scale,result=k8s.io/api/autoscaling/v1.Scale
```

第一个标记创建了get方法 `GetScale`。第二个创建了set方法 `UpdateScale`。

###### 注意

所有CR对象的 */ scale*子资源在调用时接收和返回的`Scale`对象都应该是*autoscaling / v1*这个组下的Scale类型实例。在Kubernetes API中，由于历史原因，有些资源使用不是这个类型。

## informer-gen和lister-gen

 `informer-gen`和`lister-gen`这两个代码生成器，处理的也是`// +genclient`标记，与`client-gen`是一样的。在设置好客户端生成标记后，每种类型都会自动生成与客户端匹配的*informer和listers*（如果是通过*k8s.io/code-generator/generate-group.sh*脚本调用全部代码生成器来生成的话）。

Kubernetes代码生成相关的文档会随着时间的推移肯定慢慢完善。有关不同生成器的更多信息，通常查看Kubernetes本身的示例很有帮助的 - 例如，[k8s.io / api](http://bit.ly/2ZA6dWH)和[OpenShift API类型](http://bit.ly/2KxpKnc)。这两个存储库都有许多高级用例。

此外，可以直接查看代码生成项目本身。`deepcopy-gen`在[*main.go*](http://bit.ly/2x9HmN4)文件中有一些文档可供查看。`client-gen`在[Kubernetes贡献者文档中](http://bit.ly/2WYNlns)提供了一些文档。`informer-gen`而`lister-gen`目前还没有进一步的文件，但*generate-groups.sh*中有[如何使用](http://bit.ly/31MeSHp)。

# 总结

在本章中，我们向您展示了如何将Kubernetes代码生成器用于CR。有了这一点，我们现在转向更高级别的抽象工具 - 即编写自定义控制器和operator的解决方案，使您能够专注于业务逻辑。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#idm46336860111320-marker) Go工具不会自动运行生成代码，当缺乏定义和需要定义源文件和生成文件之间依赖关系的方法时。
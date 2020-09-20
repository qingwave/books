# 第4章使用自定义资源

在 本章我们向您介绍自定义资源（CR），这是整个Kubernetes生态系统中使用的重要扩展机制之一。

自定义资源一般是对内部配置对象进行少量的配置声明，不包含任何控制器的逻辑 - 纯粹以声明方式定义。对于希望提供Kubernetes原生API体验的Kubernetes之上的许多重要开发项目，自定义资源发挥着核心作用。比如服务网格，如Istio，Linkerd 2.0和AWS App Mesh，它们都是采用自定义资源方式实现的。

还记得[第1章中的](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#intro) “定时操作的例子” 吗？它的核心是有一个如下所示的CR：

```yaml
apiVersion: cnat.programming-kubernetes.info/v1alpha1
kind: At
metadata:
  name: example-at
spec:
  schedule: "2019-07-03T02:00:00Z"
status:
  phase: "pending"
```

从1.7版本起，Kubernetes集群提供了自定义资源。它们与Kubernetes 标准API资源存储在相同的`etcd`实例中，并由相同的Kubernetes API服务器提供服务。[如图4-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#apiextensions-apiserver)所示，当自定资源类型的请求进入API服务器后，判断它们不属于以下两类请求，则请求由`apiextensions-apiserver`处理：

- 由聚合的API服务器处理请求（参见[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)）。
- 本地Kubernetes标准资源请求。

![Kubernetes API服务器内部的API Extensions服务器API](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0401.png)

###### 图4-1。Kubernetes API服务器内的API Extensions API服务器

一个CustomResourceDefinition（CRD），本身就是Kubernetes资源。它描述了群集中的可用CR。对于上述示例的自定义资源CR，相应的自定义资源定义CRD如下所示：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  group: cnat.programming-kubernetes.info
  names:
    kind: At
    listKind: AtList
    plural: ats
    singular: at
  scope: Namespaced
  subresources:
    status: {}
  version: v1alpha1
  versions:
  - name: v1alpha1
    served: true
    storage: true
```

这里，CRD的名称 - `ats.cnat.programming-kubernetes.info`必须与名称中的复数名称定义plural值完全匹配。这个被称为ats的自定义资源，它的资源类型为At，属于cnat.programming-kubernetes.info API 组，是一个命名空间下的资源。

如果在群集中创建此CRD，`kubectl`将自动检测资源，用户可以通过以下方式访问它：

```shell
$ kubectl get ats
NAME                                         CREATED AT
ats.cnat.programming-kubernetes.info         2019-04-01T14:03:33Z
```

# Discovery 机制

上一个 kubectl命令获取ats资源，其背后的原理是`kubectl`通过使用来自API服务器的discovery接口来获取新资源。我们深入看一下这个discovery机制。

通过在`kubectl`命令中设置日志的详细级别为7，我们可以看到它如何获取资源类型：

```shell
$ kubectl get ats -v=7
... GET https://XXX.eks.amazonaws.com/apis/cnat.programming-kubernetes.info/
                                      v1alpha1/namespaces/cnat/ats?limit=500
... Request Headers:
... Accept: application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json
      User-Agent: kubectl/v1.14.0 (darwin/amd64) kubernetes/641856d
... Response Status: 200 OK in 607 milliseconds
NAME         AGE
example-at   43s
```

详细的discovery步骤是：

1. 最初，`kubectl`不知道资源类型`ats`是什么。
2. 因此，`kubectl`通过请求*/ apis*这个HTTP Path，向API服务器查询出当前所有API组。
3. 接下来，针对上一步得到的每一个API组，`kubectl`通过请求*/ apis /group version* group，获取所有API组中包含的全部资源。
4. 然后，`kubectl`将给定类型`ats`转换为以下三元组：
   - 组（`cnat.programming-kubernetes.info`）
   - 版本（`v1alpha1`）
   - 资源（`ats`）。

discovery 接口提供了最后一步执行转换时所需的所有信息：

```shell
$ http localhost:8080/apis/
{
  "groups": [{
    "name": "at.cnat.programming-kubernetes.info",
    "preferredVersion": {
      "groupVersion": "cnat.programming-kubernetes.info/v1",
      "version": "v1alpha1“
    },
    "versions": [{
      "groupVersion": "cnat.programming-kubernetes.info/v1alpha1",
      "version": "v1alpha1"
    }]
  }, ...]
}

$ http localhost:8080/apis/cnat.programming-kubernetes.info/v1alpha1
{
  "apiVersion": "v1",
  "groupVersion": "cnat.programming-kubernetes.info/v1alpha1",
  "kind": "APIResourceList",
  "resources": [{
    "kind": "At",
    "name": "ats",
    "namespaced": true,
    "verbs": ["create", "delete", "deletecollection",
      "get", "list", "patch", "update", "watch"
    ]
  }, ...]
}
```

这一切都是由discovery中的`RESTMapper`实现的。这个`RESTMapper`类型我们在[“REST Mapping”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#RESTMapping)中有详细的介绍。

###### 警告

`kubectl`命令为了提升执行效率，在*〜/ .kubectl*目录对资源进行了缓存，从而不必在每次调用discovery接口时都重复获取数据。此缓存失效时间为10分钟。因此，CRD如果有更改的话，通过命令行查询时，最多有10分钟的延迟。

# 类型定义

现在让我们更详细地看一下CRD和及其功能：如`cnat`示例中所示，CRD资源在Kubernetes API服务器内是属于`apiextensions.k8s.io/v1beta1`这个API组，是由`apiextensions-apiserver`进程提供服务。

CRD的结构如下所示：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: name
spec:
  group: group name
  version: version name
  names:
    kind: uppercase name
    plural: lowercase plural name
    singular: lowercase singular name # defaulted to be lowercase kind
    shortNames: list of strings as short names # optional
    listKind: uppercase list kind # defaulted to be kindList
    categories: list of category membership like "all" # optional
  validation: # optional
    openAPIV3Schema: OpenAPI schema # optional
  subresources: # optional
    status: {} # to enable the status subresource (optional)
    scale: # optional
      specReplicasPath: JSON path for the replica number in the spec of the
                        custom resource
      statusReplicasPath: JSON path for the replica number in the status of
                          the custom resource
      labelSelectorPath: JSON path of the Scale.Status.Selector field in the
                         scale resource
  versions: # defaulted to the Spec.Version field
  - name: version name
    served: boolean whether the version is served by the API server # defaults to false
    storage: boolean whether this version is the version used to store object
  - ...
```

可以看到许多字段是可选的或默认的。我们将在以下部分中更详细地解释这些字段。

在一个CRD对象被创建后，`kube-apiserver`内部的`apiextensions-apiserver`将检查其名称并确定是否与其他资源冲突或者它本身的字段属性是否合法。检测之后，检测结果会在CRD的状态字段中体现，例如：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  group: cnat.programming-kubernetes.info
  names:
    kind: At
    listKind: AtList
    plural: ats
    singular: at
  scope: Namespaced
  subresources:
    status: {}
  validation:
    openAPIV3Schema:
      type: object
      properties:
        apiVersion:
          type: string
        kind:
          type: string
        metadata:
          type: object
        spec:
          properties:
            schedule:
              type: string
          type: object
        status:
          type: object
  version: v1alpha1
  versions:
  - name: v1alpha1
    served: true
    storage: true
status:
    acceptedNames:
      kind: At
      listKind: AtList
      plural: ats
      singular: at
    conditions:
    - lastTransitionTime: "2019-03-17T09:44:21Z"
      message: no conflicts found
      reason: NoConflicts
      status: "True"
      type: NamesAccepted
    - lastTransitionTime: null
      message: the initial names have been accepted
      reason: InitialNamesAccepted
      status: "True"
      type: Established
    storedVersions:
    - v1alpha1
```

您可以看到spec定义中未设置值的字段是取默认值，并在状态中反映为可接受的名称。此外，还设置了以下条件：

- `NamesAccepted` 描述了spec中给定的名称是否一致且没有冲突。
- `Established`描述了API服务器可以为在`status.acceptedNames`下定义的资源提供服务，状态“True”时表示正常提供服务。

请注意，在创建CRD后，例如API服务器已经可以正常给这个CRD资源提供Restful服务，这时也可以更改CRD中的某些字段。例如，您可以添加名称缩写或列信息。在这种情况下，尽管spec存在冲突，但CRD的established状态为True，会使用旧名称提供服务 。再看`NamesAccepted`这个条件的状态将是False的，表示spec名称和已经接受（提供服务）的名称是不同的。

# 自定义资源的高级功能

在本节中，我们将讨论自定义资源的高级功能，例如验证或子资源。

## 验证自定义资源

API服务器可以在CR被创建和更新时进行验证。在OpenAPI v3 scheme](http://bit.ly/2RqtN5i)中定义了可以别验证的CRD字段。

每当请求创建或修改一个CR对象时，会将JSON对象与spec中字段根据此规范进行验证，如果出现错误，则会返回给用户一个在HTTP代码`400`响应，正文返回冲突字段。[图4-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#apiextensions-apiserver-validation)显示了在`apiextensions-apiserver`内处理程序对请求进行验证的流程。

可以在验证相关的admission webhooks中实现更复杂的验证。[图4-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#apiextensions-apiserver-validation)显示了在本节中描述的基于OpenAPI的验证之后直接调用这些webhook。在[“Admission Webhooks”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#admission-webhooks)，我们将看到如何实施和部署admission webhook。到时，我们将研究将结合其他资源在内一起完成验证的过程，那将远超出OpenAPI v3验证。幸运的是，对于许多场景来说，OpenAPI v3模式就足够了。

![验证步骤在`apiextensions-apiserver`的处理程序堆栈中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0402.png)

###### 图4-2。apiextensions-apiserver的处理程序流程中的验证步骤

该OpenAPI语言基于[JSON Schema标准](http://bit.ly/2J7aIT7)，该[标准](http://bit.ly/2J7aIT7)使用JSON / YAML本身来表示。下面一个例子：

```yaml
type: object
properties:
  apiVersion:
    type: string
  kind:
    type: string
  metadata:
    type: object
  spec:
    type: object
    properties:
      schedule:
        type: string
        pattern: "^\d{4}-([0]\d|1[0-2])-([0-2]\d|3[01])..."
      command:
        type: string
    required:
    - schedule
    - command
  status:
    type: object
    properties:
      phase:
        type: string
required:
- metadata
- apiVersion
- kind
- spec
```

这个值实际上是JSON对象; [1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#idm46336863073336)即，它是一个字符串map，而不是一个列表或一个数值。此外，它有（除了`metadata`，`kind`，和`apiVersion`，作为描述资源的元数据）两个额外属性：`spec`和`status`。

这两个属性也都是JSON对象。`spec`有需要的业务属性`schedule`和`command`，这两者都是字符串。`schedule`必须匹配ISO日期的模式（在这里用正则表达式描述）。可选的 `status`属性有一个名为`phase`的字符串字段。

##### OPENAPI V3架构，完整性及其未来

OpenAPI v3模式曾经是CRD中的可选模式。在Kubernetes 1.14之前，它们仅用于服务器端验证。从这个角度看，它们可能是不完整的 - 换句话说，它们可能没有指定所有字段。

从Kubernetes 1.15开始，CRD schema定义将作为Kubernetes API服务器OpenAPI规范的一部分发布。这个也将被`kubectl`用于客户端验证。客户端验证会遇到未知字段会报错。例如，当用户键入`foo:bar`对象并且OpenAPI schema未验证通过`foo`时，`kubectl`将拒绝对该对象的操作。因此，需要使用完整的OpenAPI schema 定义。

最后，[将来会修订自定义资源实例](http://bit.ly/2WY8lKY)。这意味着 - 类似于本地Kubernetes资源类型的pod-未知（未指定）字段将不会被持久化。这不仅对数据一致性很重要，而且对安全性也很重要。这也是CRD的OpenAPI schema应该完整的另一个原因。

有关完整参考，请参阅[OpenAPI v3 schema文档](http://bit.ly/2RqtN5i)。

手动创建OpenAPI schema可能很繁琐。幸运的是，正在进行中的代码生成器工作会使这类需求方便的实现：Kubebuilder项目（ 参见这里[“Kubebuilder”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#kubebuilder)）已经在[*sig.k8s.io/controller-tools中*](http://bit.ly/2J00kvi)开发了`crd-gen`，并在逐渐扩展和丰富，以便它可以在其他地方使用。[crd-schema-gen`](http://bit.ly/31N0eQf)fork自`crd-gen`，是代码生成schema的一个项目。

## 缩写名称和分类

正如标准资源，自定义资源可能具有长资源名称。它们在API级别上很棒，但在CLI中输入很繁琐。CR也可以有短名称，就像`daemonsets`可以查询的本机资源一样`kubectl get ds`。这些缩写名称也称为别名，每个资源可以包含任意数量的别名。

查看所有可用的别名，使用如下`kubectl api-resources`命令：

```shell
$ kubectl api-resources
NAME                   SHORTNAMES  APIGROUP NAMESPACED  KIND
bindings                                    true        Binding
componentstatuses      cs                   false       ComponentStatus
configmaps             cm                   true        ConfigMap
endpoints              ep                   true        Endpoints
events                 ev                   true        Event
limitranges            limits               true        LimitRange
namespaces             ns                   false       Namespace
nodes                  no                   false       Node
persistentvolumeclaims pvc                  true       PersistentVolumeClaim
persistentvolumes      pv                   false       PersistentVolume
pods                   po                   true        Pod
statefulsets           sts         apps     true        StatefulSet
...
```

`kubectl`获取别名也是通过discovery 接口（参见[“Discovery Information”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#discovery)）。请看下面例子：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  ...
  shortNames:
  - at
```

接下来， `kubectl get at`命令将列出默认命名空间中的所有`cnat` 的CR实例。

另外，CR-与任何其他资源一样 - 都是属于分类的一部分。当我们使用all`分类，如`kubectl get all。它将列出集群中所有用户可用的资源，如pod和service。

集群中自定义的CR可以通过设置以下`categories`字段加入已有分类或创建自己的分类：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  ...
  categories:
  - all
```

这个设置分为all，`kubectl get all`命令也会在命名空间中列出自定义资源`cnat` 的CR实例。

## 打印列

`kubectl`命令行工具使用服务器端来组织`kubectl get` 查询到的打印输出内容。这意味着API服务器在查询返回中渲染好需要显示的列以及每行中的值。

自定义资源通过`additionalPrinterColumns`字段，支持对服务器端定义所需打印的列信息。之所以被称为“额外”，因为第一列始终是对象的名称。列定义如下：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  additionalPrinterColumns: (optional)
  - name: kubectl column name
    type: OpenAPI type for the column
    format: OpenAPI format for the column (optional)
    description: human-readable description of the column (optional)
    priority: integer, always zero supported by kubectl
    JSONPath: JSON path inside the CR for the displayed value
```

该`name`字段是列名，`type`是OpenAPI schema规范中[数据类型](http://bit.ly/2N0DSY4)部定义的一种，并且`format`（定义同上）是可选的，可以被用于`kubectl`或其他客户端来使用。

此外，`description`是一个可选的供使用者查看的字符串。`priority`字段用于kubectl控制显示细节。在撰写本文时（使用Kubernetes 1.14），目前仅支持零，其他具有更高优先级的列将不被显示。

最后，`JSONPath`定义要显示的值。它描述CR内部的简单JSON路径。这里，“简单”意味着它支持对象属性的语法`.spec.foo.bar`，但不支持循环遍历数组或类似的更复杂用法来描述的JSON路径。

了解这些概念，我们看看示例CRD的 `additionalPrinterColumns`定义：

```yaml
additionalPrinterColumns: #(optional)
- name: schedule
  type: string
  JSONPath: .spec.schedule
- name: command
  type: string
  JSONPath: .spec.command
- name: phase
  type: string
  JSONPath: .status.phase
```

`kubectl` get 操作将得到如下`cnat`资源：

```shell
$ kubectl get ats
NAME  SCHEDULER             COMMAND             PHASE
foo   2019-07-03T02:00:00Z  echo "hello world"  Pending
```

接下来，我们来看看子资源。

## 子资源

我们在[“Status Subresources：UpdateStatus”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go-subresource)提到了子资源。子资源是特殊的HTTP 方法，使用附加到普通资源的HTTP路径的后缀。例如，pod标准HTTP路径是*/ api / v1 / namespace / namespace/ pods /name*。Pod有许多子资源，例如*/ logs*，*/ portforward*，*/ exec*和*/ status*。相应的子资源HTTP路径是：

- */ api / v1 / namespace /* `namespace`*/ pods /* `name`*/ logs*
- */ api / v1 / namespace /* `namespace`*/ pods /* `name`*/ portforward*
- */ api / v1 / namespace /* `namespace`*/ pods /* `name`*/ exec*
- */ api / v1 / namespace /* `namespace`*/ pods /* `name`*/ status*

子资源端点使用与主资源端点不同的协议。

在撰写本文时，自定义资源支持两个子资源：*/ scale*和*/ status*。两者都是可选的，即必须在CRD中明确启用它们。

### 状态子资源

该*/status*子资源用于将CR实例中由控制器管控的状态和用户管控的spec分区开来。这样做的主要目的是控制权分离：

- 用户通常不应该写状态字段。
- 控制器不应写入spec字段。

用于访问控制的RBAC机制目前达不到这么详细地级别。这些规则始终是针对每个资源来说的。该*/status*子资源通过提供两个不同的HTTP path来解决这个问题。每个都可以独立地使用RBAC规则进行控制。这通常称为*spec-status拆分*。以下是以`ats`资源为例，该规则适用于*/ status*子资源（同时`"ats"`与主资源匹配）：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: ...
rules:
- apiGroups: [""]
  resources: ["ats/status"]
  verbs: ["update", "patch"]
```

包含*/ status*子资源的资源（包括自定义资源）语义和之前有所不同，这里它既是一个子资源也代表了主资源：

- 在对主资源对应的HTTP方法进行调用时，它会忽略status中的属性内容。（例如创建和更新时）
- 同样，调用*/ status*子资源对应的HTTP方法时，只有status中的内容会起作用，其他信息将被忽略。另外，对于*/ status* 执行创建操作也是无效的。
- 每当改变非`metadata`和非`status`的内容时（即改变spec的内容），对主资源对应的HTTP方法调用，会将`metadata.generation`值增加。这将给控制器的一个信号，代表用户改变了spec的内容。

注意，通常`spec`和`status`在更新请求时两部分内容都会提供，不过从技术角度说，提供其中一个就足够了。

另请注意，对*/ status*子资源操作时，将忽略*状态*之外的所有其他内容更改，包括标签或注释等元数据。

启用自定义资源的spec-status状态，按照如下设置：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
spec:
  subresources:
    status: {}
  ...
```

注意`status`，在以上YAML片段中的字段被分配了空对象。如果将status写作：

```yaml
subresources:
  status:
```

将导致验证错误，因为在YAML中，这样写的结果将给`status赋`null`值`，而null 将验证不通过。

###### 警告

启用spec-status拆分是一个非兼容性的API变化。旧的控制器将写入主资源对应的HTTP
方法。一旦拆分启用，他们不会意识到status内容将被忽略。同样，在拆分之前，新的控制器无法写入新的*/status*子资源对应的HTPP方法。

在Kubernetes 1.13及更高版本中，可以为每个版本配置子资源。这允许我们以兼容方式引入*/ status*子资源：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
spec:
  ...
  versions:
  - name: v1alpha1
    served: true
    storage: true
  - name: v1beta1
    served: true
    subresources:
      status: {}
```

这将对`v1beta1`版本启用*/ status*子资源，但不能用于`v1alpha1`。

###### 注意

从Kubernetes的乐观并发锁机制来看（参见[“乐观并发”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#optimistic-concurrency)）与主资源相同，对于子资源的并发操作也需要参看resource version 字段，也就是说，`status`并且`spec`共享相同的resource version计数器，*/status*更新可能与写入主资源操作冲突，反之亦然。换句话说，存储层上没有分割`spec`和分割`status`。

### 扩缩子资源

可用于自定义资源的第二个子资源是*/ scale*。*/scale*子资源可以看做是主资源的视图（投影）[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#idm46336862332104)，我们只需要查看和修改副本值即可，其他值不需要关心。这个 subresource以Kubernetes中的部署和副本集等资源而闻名，显然可以扩容和缩容。

`kubectl scale`命令使用*/ scale*子资源; 例如，以下内容将修改给定实例中的指定副本值：

```shell
$ kubectl scale --replicas=3 your-custom-resource -v=7
I0429 21:17:53.138353   66743 round_trippers.go:383] PUT
https://host/apis/group/v1/your-custom-resource/scale
```

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
spec:
  subresources:
    scale:
      specReplicasPath: .spec.replicas
      statusReplicasPath: .status.replicas
      labelSelectorPath: .status.labelSelector
  ...
```

kubectl命令将更新，`spec.replicas`所代表的副本值，之后返回。

*/ status*子资源中使用标签选择器时，不能对标签选择器本身的值进行修改，只能读取。这里标签选择器的目的只是用来作为条件查找目标对象的。例如，`ReplicaSet`控制器计算满足此选择器的相应pod。

标签选择器是可选的。如果您的自定义资源语义不适合标签选择器，就不要为其指定JSON路径。

在前面`kubectl scale --replicas=3 ...`的示例值`3`写入`spec.replicas`。当然，可以使用任何其他简单的JSON路径; 例如，`spec.instances`或者`spec.size`，根据实际情况也可以这样写。

##### 副本整数值与创建和删除副本的控制器

我们讨论了在自定义资源中读取和设置副本值。实际上真正的删除或创建操作，必须由自定义控制器实现（请参阅[“控制器和operator”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#ch_controllers-operators)）。

`Scale`定义在`autoscaling/v1`API组。这是它的结构：

```go
type Scale struct {
    metav1.TypeMeta `json:",inline"`
    // Standard object metadata; More info: https://git.k8s.io/
    // community/contributors/devel/api-conventions.md#metadata.
    // +optional
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // defines the behavior of the scale. More info: https://git.k8s.io/community/
    // contributors/devel/api-conventions.md#spec-and-status.
    // +optional
    Spec ScaleSpec `json:"spec,omitempty"`

    // current status of the scale. More info: https://git.k8s.io/community/
    // contributors/devel/api-conventions.md#spec-and-status. Read-only.
    // +optional
    Status ScaleStatus `json:"status,omitempty"`
}

// ScaleSpec describes the attributes of a scale subresource.
type ScaleSpec struct {
    // desired number of instances for the scaled object.
    // +optional
    Replicas int32 `json:"replicas,omitempty"`
}

// ScaleStatus represents the current status of a scale subresource.
type ScaleStatus struct {
    // actual number of observed instances of the scaled object.
    Replicas int32 `json:"replicas"`

    // label query over pods that should match the replicas count. This is the
    // same as the label selector but in the string format to avoid
    // introspection by clients. The string will be in the same
    // format as the query-param syntax. More info about label selectors:
    // http://kubernetes.io/docs/user-guide/labels#label-selectors.
    // +optional
    Selector string `json:"selector,omitempty"`
}
```

实例化后将如下所示：

```yaml
metadata:
  name: cr-name
  namespace: cr-namespace
  uid: cr-uid
  resourceVersion: cr-resource-version
  creationTimestamp: cr-creation-timestamp
spec:
  replicas: 3
  status:
    replicas: 2
    selector: "environment = production"
```

请注意，主资源和*/ scale*子资源的乐观并发锁机制相同。也就是说，对主资源写入可能与*/ scale*写入冲突，反之亦然。

# 从开发视角看自定义资源

可以使用许多Golang实现的客户端访问自定义资源。我们将关注于：

- 使用`client-go`动态客户端（请参阅[“动态客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#dynamic-client)）
- 使用typed客户端：
  - 由[kubernetes-sigs / controller-runtime提供](http://bit.ly/2ZFtDKd)，被Operator SDK和Kubebuilder使用( [“controller-runtime Client of Operator SDK and Kubebuilder”](https://learning.oreilly.com/library/view/Programming+Kubernetes/9781492047094/ch04.html#controller-runtime))
  - 如 [*k8s.io/client-go/kubernetes中*](http://bit.ly/2FnmGWA)`使用client-gen`生成的那样（参见[“通过client-gen创建的类型客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#clientgen-client)）

具体选择使用哪个客户端主要取决于要编写代码时的考虑，尤其需求逻辑和实现的复杂性（例如，支持动态特性和支持在编译时未知的GVK）。

前面的客户列表：

- 降低处理未知GVK的灵活性。
- 增加类型安全性。
- 增加了他们提供的Kubernetes API功能的完整性。

## 动态客户端

在[*k8s.io/client-go/dynamic中的*](http://bit.ly/2Y6eeSK)动态客户端是与GVK无关的。除了[*unstructured.Unstructured之外*](http://bit.ly/2WYZ6oS)，它甚至不使用任何Go类型，它只包装了 `json.Unmarshal`和它的输出。

动态客户端既不使用scheme也不使用RESTMapper。这意味着开发人员必须通过以GVR的形式提供资源（请参阅[“资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#resources)）来手动指定有关的三元组数据：

```go
schema.GroupVersionResource{
  Group: "apps",
  Version: "v1",
  Resource: "deployments",
}
```

如果可以使用REST config（请参阅[“创建和使用客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)），可以在一行中创建动态客户端：

```go
client, err := NewForConfig(cfg)
```

对给定GVR的REST访问非常简单：

```go
client.Resource(gvr).
   Namespace(namespace).Get("foo", metav1.GetOptions{})
```

这使您可以`foo`在给定的命名空间中进行部署。

###### 注意

您必须知道资源的范围（即，它是命名空间还是集群作用域）。集群范围的资源只是忽略了`Namespace(namespace)`调用。

动态客户端的输入和输出`*unstructured.Unstructured`是一个对象，它与`json.Unmarshal`在解析对象时输出的数据结构相同，其内部主要包含以下结构：

- 对象用表示`map[string]interface{}`。
- 数组表示为`[]interface{}`。
- 原始类型`string`，`bool`，`float64`，或`int64`。

方法`UnstructuredContent()`提供对非结构化对象内部的数据结构的访问（我们也可以访问`Unstructured.Object`）。在包中有一些帮助方法，可以从对象中方便的检索字段 - 例如：

```go
name, found, err := unstructured.NestedString(u.Object, "metadata", "name")
```

这行代码将返回deployment的名称`"foo"`。如果实际找到该字段（不为空的的值），`found`值为true。`err`代表可能的异常发生（如，这个例子中，如果不是字符串）。其他是一些通用的帮助方法，Copy结尾的返回原始对象的深拷贝，NoCopy不是深拷贝：

```go
func NestedFieldCopy(obj map[string]interface{}, fields ...string)
  (interface{}, bool, error)
func NestedFieldNoCopy(obj map[string]interface{}, fields ...string)
  (interface{}, bool, error)
```

还有一些针对不同类型的类型转换方法，如果失败则返回错误：

```go
func NestedBool(obj map[string]interface{}, fields ...string) (bool, bool, error)
func NestedFloat64(obj map[string]interface{}, fields ...string)
  (float64, bool, error)
func NestedInt64(obj map[string]interface{}, fields ...string) (int64, bool, error)
func NestedStringSlice(obj map[string]interface{}, fields ...string)
  ([]string, bool, error)
func NestedSlice(obj map[string]interface{}, fields ...string)
  ([]interface{}, bool, error)
func NestedStringMap(obj map[string]interface{}, fields ...string)
  (map[string]string, bool, error)
```

最后是一个通用的setter方法：

```go
func SetNestedField(obj, value, path...)
```

动态客户端在Kubernetes中用于通用控制器，如垃圾回收控制器，它删除父项已消失的对象。垃圾回收控制器可以与系统中的任何资源一起使用，因此可以广泛使用动态客户端。

## 类型化的客户端

类型化的客户端不使用`map[string]interface{}`类似的通用数据结构，而是使用真实的Golang类型，每个GVK都有对应的Golang类型。它们更易于使用，大大提高了类型安全性，并使代码更简洁，更易读。缺点是，它们的灵活性较低，因为必须在编译时知道已有的类型，并生成这些客户端，这会增加复杂性。

在进入类型化客户端的两个实现之前，让我们看看Golang类型系统中各种类型的表示（有关Kubernetes类型系统背后的理论，请参阅[“深入API Machinery”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#api-machinery-core)）。

### 一个类型的构成

一个资源类型对应于Golang的结构体。通常结构体和资源类型是同名的，只不过名称采用驼峰方式命名（虽然从技术上讲它不必是），并且被放置在与这个GVK的组和版本相对应的包中。常见的惯例是把这个GVK 放到类似*group*/ *version*.*Kind* 层次的Go包中：

```go
pkg / apis / group / version
```

名为*Kind* Golang结构体被定义在文件*types.go中*。

对应于GVK的每个Golang类型内都嵌入`TypeMeta`结构（来自于包[*k8s.io/apimachinery/pkg/apis/meta/v1中*](http://bit.ly/2Y5HdWT)）。`TypeMeta`包括`Kind`和`ApiVersion`两个字段：

```go
type TypeMeta struct {
    // +optional
    APIVersion string `json:"apiVersion,omitempty" yaml:"apiVersion,omitempty"`
    // +optional
    Kind string `json:"kind,omitempty" yaml:"kind,omitempty"`
}
```

此外，每个顶级的资源类型（ 就是指具有自己的HTTP Restful访问方法的资源，它可能对应一个或多个GVR（参见[“REST Mapping”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#RESTMapping)） ） 具有存储名称，属于命名空间作用域的资源拥有命名空间以及元数据相关的字段。所有这些构成了Golang结构体`ObjectMeta`，存储在在[*k8s.io/apimachinery/pkg/apis/meta/v1*](http://bit.ly/2XSt8eo)包中：

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

除此之外，还有许多其他字段。我们强烈建议您阅读 [extensive inline documentation](http://bit.ly/2IutNyh)，这里描述了Kubernetes对象的核心功能。

Kubernetes顶级类型（即那些嵌入了TypeMeta和ObjectMeta，被持久化在etcd中的类型`）它们之间看起来非常相似，因为它们通常都有 `spec和status字段。看一下[*k8s.io/kubernetes/apps/v1/types.go*](http://bit.ly/2RroTFb)中deployment的例子：

```go
type Deployment struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec DeploymentSpec `json:"spec,omitempty"`
    Status DeploymentStatus `json:"status,omitempty"`
}
```

尽管对于不同类型来说spec`和`status中的实际内容是不同的，但是将资源的信息分为spec和status两个部分是Kubernetes中的一个约定俗成。我们推荐遵循这种CRD结构的划分方法。一些CRD功能甚至依赖于这种结构; 例如，自定义资源的*/ status*子资源（请参阅[“状态子资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#status-subresource)） 当启用时，对于该自定义资源的操作，将仅作用于`status`字段下的内容。并且不能将它重命名。

### GOLANG 包结构

如我们所见，Golang类型通常定义在名为types.go的文件中，该文件放在包*pkg / apis /*group */version中*。除了这个文件，我们再来看看其他的文件。其中一些是由开发人员手动编写的，而另一些是使用代码生成器生成的。详细信息请参见[第5章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)。

名称为*doc.go*的文件描述了API的目的，同时定义了包级别的全局代码生成标记：

```go
// Package v1alpha1 contains the cnat v1alpha1 API group
//
// +k8s:deepcopy-gen=package
// +groupName=cnat.programming-kubernetes.info
package v1alpha1
```

register.go包含帮助代码，将自定义资源Golang类型注册到scheme中（请参阅[“scheme”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#scheme)）：

```go
package version

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/schema"

    group "repo/pkg/apis/group"
)

// SchemeGroupVersion is group version used to register these objects
var SchemeGroupVersion = schema.GroupVersion{
    Group: group.GroupName,
    Version: "version",
}

// Kind takes an unqualified kind and returns back a Group qualified GroupKind
func Kind(kind string) schema.GroupKind {
    return SchemeGroupVersion.WithKind(kind).GroupKind()
}

// Resource takes an unqualified resource and returns a Group
// qualified GroupResource
func Resource(resource string) schema.GroupResource {
    return SchemeGroupVersion.WithResource(resource).GroupResource()
}

var (
    SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
    AddToScheme   = SchemeBuilder.AddToScheme
)

// Adds the list of known types to Scheme.
func addKnownTypes(scheme *runtime.Scheme) error {
    scheme.AddKnownTypes(SchemeGroupVersion,
        &SomeKind{},
        &SomeKindList{},
    )
    metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
    return nil
}
```

*zz_generated.deepcopy.go*定义了自定义资源的顶级Golang类型的深拷贝方法（例如，之前示例代码中的`SomeKind`与`SomeKindList`）。另外，其他子结构的深拷贝方法（如`spec`和`status`）。

因为本例*doc.go*文件中使用的标签为`+k8s:deepcopy-gen=package`，默认该包下的所有对象都会生成深拷贝; 也就是说，如果某个类型不希望生成深拷贝，需要在类型定义上方单独设置标记`+k8s:deepcopy-gen=false`。有关详细信息，请参阅[第5章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)，尤其是[“deepcopy-gen标记”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#deepcopy-tags)。

### 通过CLIENT-GEN创建类型化客户端

与API包的位置*pkg / apis / group/version*对应，客户端生成器`client-gen`创建的类型化客户端代码默认的生成路径为（参见[第5章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)的详细信息，特别是[“客户端根标签”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#clientgen-tags)），*pkg /generated/ clientset / versioned*（pkg /client/clientset/versioned 旧版本的生成器生成代码的路径）。准确地说，生成的是客户端的集合。它包含多种API组，版本和资源。

客户端集合的[文件](http://bit.ly/2GdcikH)如下所示：

```go
// Code generated by client-gen. DO NOT EDIT.

package versioned

import (
    discovery "k8s.io/client-go/discovery"
    rest "k8s.io/client-go/rest"
    flowcontrol "k8s.io/client-go/util/flowcontrol"

    cnatv1alpha1 ".../cnat/cnat-client-go/pkg/generated/clientset/versioned/
)

type Interface interface {
    Discovery() discovery.DiscoveryInterface
    CnatV1alpha1() cnatv1alpha1.CnatV1alpha1Interface
}

// Clientset contains the clients for groups. Each group has exactly one
// version included in a Clientset.
type Clientset struct {
    *discovery.DiscoveryClient
    cnatV1alpha1 *cnatv1alpha1.CnatV1alpha1Client
}

// CnatV1alpha1 retrieves the CnatV1alpha1Client
func (c *Clientset) CnatV1alpha1() cnatv1alpha1.CnatV1alpha1Interface {
    return c.cnatV1alpha1
}

// Discovery retrieves the DiscoveryClient
func (c *Clientset) Discovery() discovery.DiscoveryInterface {
   ...
}

// NewForConfig creates a new Clientset for the given config.
func NewForConfig(c *rest.Config) (*Clientset, error) {
    ...
}
```

客户端集合定义了名为Interface的接口，并且为每个版本提供对API组客户端接口的访问 - 例如，本例中CnatV1alpha1Interface：

```go
type CnatV1alpha1Interface interface {
    RESTClient() rest.Interface
    AtsGetter
}

// AtsGetter has a method to return a AtInterface.
// A group's client should implement this interface.
type AtsGetter interface {
    Ats(namespace string) AtInterface
}

// AtInterface has methods to work with At resources.
type AtInterface interface {
    Create(*v1alpha1.At) (*v1alpha1.At, error)
    Update(*v1alpha1.At) (*v1alpha1.At, error)
    UpdateStatus(*v1alpha1.At) (*v1alpha1.At, error)
    Delete(name string, options *v1.DeleteOptions) error
    DeleteCollection(options *v1.DeleteOptions, listOptions v1.ListOptions) error
    Get(name string, options v1.GetOptions) (*v1alpha1.At, error)
    List(opts v1.ListOptions) (*v1alpha1.AtList, error)
    Watch(opts v1.ListOptions) (watch.Interface, error)
    Patch(name string, pt types.PatchType, data []byte, subresources ...string)
        (result *v1alpha1.At, err error)
    AtExpansion
}
```

可以使用NewForConfig帮助方法创建客户端集合的实例。这与[“创建和使用客户端”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)讨论的，创建标准Kubernetes资源[的客户端](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)类似：

```go
import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/tools/clientcmd"

    client "github.com/.../cnat/cnat-client-go/pkg/generated/clientset/versioned"
)

kubeconfig = flag.String("kubeconfig", "~/.kube/config", "kubeconfig file")
flag.Parse()
config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
clientset, err := client.NewForConfig(config)

ats := clientset.CnatV1alpha1Interface().Ats("default")
book, err := ats.Get("kubernetes-programming", metav1.GetOptions{})
```

如您所见，代码生成机制允许我们以与操作标准Kubernetes资源相同的方式为自定义资源编写逻辑。也可以使用像informer这样的高级工具; 参见[第5章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)`informer-gen`。

## operator SDK和Kubebuilder的controller-runtime客户端

对于为了完整起见，我们想快速浏览一下第三种客户端，它可以作为[“从开发视角看自定义资源”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-dev)的第二个选项。`controller-runtime`项目为[第6章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#ch_operator-solutions)介绍的operator解决方案Operator SDK和Kubebuilder提供了基础。它包含了一个客户端，正用到了之前“[类行的构成”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#anatomy-of-CRD-types)介绍的Go结构。

与先前[“通过client-gen客户端创建的类型化客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#clientgen-client)，和[“动态客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#dynamic-client)相比，这个客户端是一个实例，它能够处理在给定scheme中注册的任何类型。

它使用来自API服务器的discovery将类型映射到HTTP路径。请注意，[第6章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#ch_operator-solutions)将更详细地介绍那两个operator的框架是如何使用这个客户端的。

以下是使用`controller-runtime`的示例：

```go
import (
    "flag"

    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/tools/clientcmd"

    runtimeclient "sigs.k8s.io/controller-runtime/pkg/client"
)

kubeconfig = flag.String("kubeconfig", "~/.kube/config", "kubeconfig file path")
flag.Parse()
config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)

cl, _ := runtimeclient.New(config, client.Options{
    Scheme: scheme.Scheme,
})
podList := &corev1.PodList{}
err := cl.List(context.TODO(), client.InNamespace("default"), podList)
```

客户端对象的`List()`方法接受任何实现了runtime.Object`接口对象，这个对象事先需要在给定的scheme中注册，这个例子中是由`client-go将所有标准Kubernetes类型默认注册到了scheme中。在内部过程来看，客户端使用传入的scheme将Golang类型`*corev1.PodList`转换为GVK。在第二步中，该`List()`方法使用discovery接口来获取pod的GVR，即 `schema.GroupVersionResource{"", "v1", "pods"}`，因此访问*/ api / v1 / {namespace} / default / pods* 就可以得到该命名空间中的pod列表。

自定义资源同样适用。主要区别是给定的scheme对象中需要提前注册自定义资源对应的Go类型：

```go
import (
    "flag"

    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/tools/clientcmd"

    runtimeclient "sigs.k8s.io/controller-runtime/pkg/client"
    cnatv1alpha1 "github.com/.../cnat/cnat-kubebuilder/pkg/apis/cnat/v1alpha1"
)

kubeconfig = flag.String("kubeconfig", "~/.kube/config", "kubeconfig file")
flag.Parse()
config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)

crScheme := runtime.NewScheme()
cnatv1alpha1.AddToScheme(crScheme)

cl, _ := runtimeclient.New(config, client.Options{
    Scheme: crScheme,
})
list := &cnatv1alpha1.AtList{}
err := cl.List(context.TODO(), client.InNamespace("default"), list)
```

注意`List()`命令调用没有变化。

可以想象一下，您编写了一个operator使用客户端访问许多不同类型的资源。如果采用（[“通过client-gen创建类型化客户端](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#clientgen-client)”）类型化客户端，您需要创建许多不同的客户端传递给operator，使得代码非常复杂。相比之下，`controller-runtime`这里只需要一个可以操作所有类型的客户端，这些类型都在给定的scheme注册好。

所有三种类型的客户端都有其用途、各有利弊。在处理未知对象的通用控制器中，只能使用动态客户端。在强调类型安全的场景中，类型化的客户端可以保证代码执行的正确性，生成类型化的客户端非常适合。Kubernetes项目本身有很多贡献者，即使代码的稳定性非常重要，即使它被很多人扩展和重写。如果希望方便和快速对您来说更重要些，`controller-runtime`客户端是一个不错的选择。

# 总结

我们向您介绍了自定义资源，Kubernetes生态系统中重要的扩展机制。到目前为止，您应该很好地了解它们的功能和限制以及几种可用的客户端。

现在让我们继续使用代码生成来管理所述资源。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#idm46336863073336-marker)不要在这里混淆Kubernetes和JSON对象。后者只是字符串映射的另一个术语，用于JSON和OpenAPI的上下文中。

[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#idm46336862332104-marker) “投影”在这里意味着`scale`对象是主要资源的投影，因为它只显示某些字段并隐藏其他所有字段。
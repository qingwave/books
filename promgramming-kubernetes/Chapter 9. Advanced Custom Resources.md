# 第9章高级自定义资源

在本章中，我们将向您介绍有关CR的高级主题：版本控制，转换和许可控制器。

对于多个版本，CRD变得更加严重，并且与基于Golang的API资源的区别要小得多。当然，同时复杂性在开发和维护方面都有相当大的增长，但在操作上也是如此。我们将这些功能称为“高级”，因为它们将CRD从清单（即纯粹声明性）转移到Golang世界（即进入真正的软件开发项目）。

即使您不打算构建自定义API服务器而是打算直接切换到CRD，我们强烈建议您不要跳过[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)。围绕高级CRD的许多概念在自定义API服务器的世界中具有直接对应物，并且受它们的推动。阅读[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)将使本章更容易理解。

此处显示和讨论的所有示例的代码都可以通过[GitHub存储库获得](http://bit.ly/2RBSjAl)。

# 自定义资源版本控制

在[第8章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)我们了解了如何通过不同的API版本提供资源。在自定义API服务器的示例中，披萨资源存在于版本中`v1alpha1`并且`v1beta1`同时存在（参见[“示例：比萨餐厅”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-example)）。在自定义API服务器内部，请求中的每个对象首先从API端点版本转换为内部版本（请参阅[“内部类型和转换”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-internal-types)和[图8-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-conversions-figure)），然后转换回外部版本进行存储，并转换为回复一个回复。该转换机制由转换函数实现，其中一些是手动编写的，一些是生成的（参见[“转换”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-conversion)）。

版本控制API是一种强大的机制，用于调整和改进API，同时保持旧客户端的兼容性。版本在Kubernetes的每个地方都发挥着核心作用，将alpha API推向beta，最终推广到普通可用性（GA）。在此过程中，API通常会更改结构或进行扩展。

很长一段时间，版本控制只是一个功能[第8章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)介绍的聚合API服务器。任何严重的API最终都需要进行版本控制，因为打破与API的使用者的兼容性是不可接受的。

幸运的是，最近在Kubernetes中添加了CRD版本 - 在Kubernetes 1.14中作为alpha版本，并在1.15中升级为beta版。请注意，转换需要*结构化的* OpenAPI v3验证模式（请参阅[“验证自定义资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-validation)）。结构模式基本上就像Kubebuilder这样的工具。我们将在[“结构模式”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-structural-schema)讨论技术细节。

我们将向您展示版本控制如何在这里工作，因为它将在不久的将来在CR的许多严肃应用中发挥核心作用。

## 修改比萨餐厅

至了解CR转换的工作原理，我们将重新实现[第8章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)的披萨餐厅示例，这次仅仅使用CRD - 即没有涉及聚合的API服务器。

对于转换，我们将专注于`Pizza`资源：

```
apiVersion: restaurant.programming-kubernetes.info/v1alpha1
kind: Pizza
metadata:
  name: margherita
spec:
  toppings:
  - mozzarella
  - tomato
```

此对象应在`v1beta1`版本中具有不同的浇头切片表示：

```
apiVersion: restaurant.programming-kubernetes.info/v1beta1
kind: Pizza
metadata:
  name: margherita
spec:
  toppings:
  - name: mozzarella
    quantity: 1
  - name: tomato
    quantity: 1
```

在使用时`v1alpha1`，重复浇头用于代表额外的奶酪比萨饼，我们`v1beta1`通过使用每个浇头的数量字段来实现这一点。浇头的顺序无关紧要。

我们希望实现这种转换 - 转换`v1alpha1`为`v1beta1`和转换。不过，在我们这样做之前，让我们将API定义为CRD。请注意，我们不能在同一个集群中拥有相同GroupVersion的聚合API服务器和CRD。因此，请确保在继续使用CRD之前删除[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)中的APIServices。

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: pizzas.restaurant.programming-kubernetes.info
spec:
  group: restaurant.programming-kubernetes.info
  names:
    kind: Pizza
    listKind: PizzaList
    plural: pizzas
    singular: pizza
  scope: Namespaced
  version: v1alpha1
  versions:
  - name: v1alpha1
    served: true
    storage: true
    schema: ...
  - name: v1beta1
    served: true
    storage: false
    schema: ...
```

CRD定义了两个版本：`v1alpha1`和`v1beta1`。我们将前者设置为存储版本（参见[图9-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-conversion-resource-versioning)），这意味着要存储的每个对象首先被`etcd`转换为`v1alpha1`。

![转换和存储版本](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0901.png)

###### 图9-1。转换和存储版本

由于当前定义了CRD，我们可以创建一个对象`v1alpha1`并将其检索为`v1beta1`，但两个API端点都返回相同的对象。这显然不是我们想要的。但我们很快就会改善这一点。

但在我们这样做之前，我们将在群集中设置CRD并创建一个margherita披萨：

```
apiVersion: restaurant.programming-kubernetes.info/v1alpha1
kind: Pizza
metadata:
  name: margherita
spec:
  toppings:
  - mozzarella
  - tomato
```

我们注册前面的CRD，然后创建margherita对象：

```
$ kubectl create -f pizza-crd.yaml
 $ kubectl create -f margherita-pizza.yaml
```

正如所料，我们为两个版本找回了相同的对象：

```
$ kubectl获得披萨margherita -o yaml
apiVersion：restaurant.programming-kubernetes.info/v1beta1
亲切：披萨
元数据：
  creationTimestamp： "2019-04-14T11:39:20Z"
  一代：1
  名称：玛格丽塔
  命名空间：pizza-apiserver
  resourceVersion： "47959"
  selfLink：/apis/restaurant.programming-kubernetes.info/v1beta1/namespaces/pizza-apiserver/
  比萨饼/雏菊
  uid：f18427f0-5ea9-11e9-8219-124e4d2dc074
规格：
  配料：
  - 奶酪
  - 番茄
```

Kubernetes使用规范版本顺序; 那是：

- `v1alpha1`

  不稳定：可能会随时离开或更改，并且通常默认情况下禁用。

- `v1beta1`

  走向稳定：至少在一个版本中平行存在`v1`; 合同：没有不兼容的API更改。

- `v1`

  稳定或一般可用（GA）：将保持良好，并将兼容。

GA版本按顺序排在第一位，然后是beta版，然后是alpha版，主要版本从高到低排序，次要版本排序相同。每个不符合此模式的CRD版本都是最后一个，按字母顺序排序。

在我们的例子中，前面`kubectl get pizza`因此返回`v1beta1`，尽管创建的对象是版本`v1alpha1`。

## 转换Webhook架构

现在让我们从加入转换`v1alpha1`到`v1beta1`和背部。CRD转换是通过Kubernetes中的webhook实现的。流程[如图9-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-conversion-webhook)所示：

1. 客户端（例如，我们的`kubectl get pizza margherita`）请求版本。
2. `etcd` 已将对象存储在某个版本中。
3. 如果版本不匹配，则将存储对象发送到webhook服务器以进行转换。webhook返回转换对象的响应。
4. 转换后的对象将被发送回客户端。

![转换Webhook](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0902.png)

###### 图9-2。转换webhook

我们必须实现这个webhook服务器。在此之前，让我们看一下webhook API。Kubernetes API服务器发送`ConversionReview`API组中的对象`apiextensions.k8s.io/v1beta1`：

```
type ConversionReview struct {
    metav1.TypeMeta `json:",inline"`
    Request *ConversionRequest
    Response *ConversionResponse
}
```

请求字段在发送到webhook的有效负载中设置。响应字段在响应中设置。

请求如下所示：

```
type ConversionRequest struct {
    ...

    // `desiredAPIVersion` is the version to convert given objects to.
    // For example, "myapi.example.com/v1."
    DesiredAPIVersion string

    // `objects` is the list of CR objects to be converted.
    Objects []runtime.RawExtension
}
```

该`DesiredAPIVersion`字符串具有`apiVersion`我们所知的通常格式`TypeMeta`：*group/version*。

objects字段有许多对象。这是一个切片，因为对于一个比萨饼的列表请求，webhook将收到一个转换请求，该切片是列表请求的所有对象。

webhook转换并设置响应：

```
type ConversionResponse struct {
    ...

    // `convertedObjects` is the list of converted versions of `request.objects`
    // if the `result` is successful otherwise empty. The webhook is expected to
    // set apiVersion of these objects to the ConversionRequest.desiredAPIVersion.
    // The list must also have the same size as input list with the same objects
    // in the same order (i.e. equal UIDs and object meta).
    ConvertedObjects []runtime.RawExtension

    // `result` contains the result of conversion with extra details if the
    // conversion failed. `result.status` determines if the conversion failed
    // or succeeded. The `result.status` field is required and represents the
    // success or failure of the conversion. A successful conversion must set
    // `result.status` to `Success`. A failed conversion must set `result.status`
    // to `Failure` and provide more details in `result.message` and return http
    // status 200. The `result.message` will be used to construct an error
    // message for the end user.
    Result metav1.Status
}
```

结果状态告诉Kubernetes API服务器转换是否成功。

但是在请求管道中我们实际调用了转换webhook？我们可以期待什么样的输入对象？为了更好地理解这一点，请查看[图9-3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-conversion-crd-webhook-calls)中的常规请求管道：所有这些实心和条纹圆都是在*k8s.io/apiserver*代码中进行转换的地方。

![转换webhook调用CR](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0903.png)

###### 图9-3。转换webhook调用CR

与聚合的自定义API服务器（请参阅[“内部类型和转换”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-internal-types)）相比，CR不使用内部类型，而是直接在外部API版本之间进行转换。因此，只有那些黄色圆圈实际上在[图9-4中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-conversion-crd-pipeline)进行转换; 实心圆圈是CRD的NOOP。换句话说：CRD转换只发生在和从`etcd`。

![CR转换的地方](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0904.png)

###### 图9-4。CR转换的地方

因此，我们可以假设我们的webhook将从请求管道中的这两个位置调用（参见[图9-3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-conversion-crd-webhook-calls)）。

另请注意，修补程序请求会在冲突时自动重试（更新无法重试，并且它们会直接向调用方回复错误）。每次重试都包含读取和写入`etcd`（[图9-3中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-conversion-crd-webhook-calls)的黄色圆圈），因此每次迭代会导致两次调用webhook。

###### 警告

[“转化”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-conversion)关于转化关键性的所有警告也适用于此：转化必须正确。错误很快导致数据丢失和API的不一致行为。

在我们开始实施webhook之前，关于webhook可以做什么并且必须避免的最后一些话：

- 请求和响应中对象的顺序不得更改。
- `ObjectMeta` 标签和注释除外不得变异。
- 转换是全部或全部：要么所有对象都成功转换，要么全部失败。

## 转换Webhook实现

同我们背后的理论，我们准备开始实施webhook项目。您可以在[存储库中](http://bit.ly/2IHXKLn)找到源代码，其中包括：

- 作为HTTPS Web服务器的webhook实现
- 许多端点：
  - */ convert / v1beta1 / pizza*在`v1alpha1`和之间转换披萨对象`v1beta1`。
  - */ admit / v1beta1 / pizza*将该`spec.toppings`字段默认为马苏里拉奶酪，番茄，萨拉米香肠。
  - */ validate / v1beta1 / pizza*验证每个指定的顶部是否具有相应的浇头对象。

最后两个端点是入场webhooks，将在[“Admission Webhooks”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#admission-webhooks)中详细讨论。相同的webhook二进制文件将同时用于准入和转换。

在`v1beta1`这些路径不应该与混淆`v1beta1`我们的餐厅API组，但它意味着作为`apiextensions.k8s.io`API组的版本，我们支持作为网络挂接。有一天`v1`会支持webhook API，[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#idm46336844085864)此时我们将添加相应的`v1`另一个端点，以支持旧的（截至今天）和新的Kubernetes集群。可以在CRD清单中指定webhook支持的版本。

让我们来看看这个转换webhook实际上是如何工作的。之后，我们将深入探讨如何将webhook部署到真正的集群中。再次注意，webhook转换在1.14中仍然是alpha，必须使用`CustomResourceWebhookConversion`功能门手动启用，但它在1.15中以beta版的形式提供。

## 设置HTTPS服务器

该第一步是启动支持传输层安全性或TLS（即HTTPS）的Web服务器。Kubernetes中的Webhooks需要HTTPS。转换webhook甚至需要Kubernetes API服务器针对CRD对象中提供的CA包成功检查的证书。

在示例项目中，我们使用了作为*k8s.io/apiserver*一部分的安全服务库。它提供您可能用于部署`kube-apiserver`或聚合API服务器二进制文件的所有TLS标志和行为。

该*k8s.io/apiserver*安全服务代码遵循`options-config`模式（请参阅[“选项和配置模式和启动管道”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-options-config)）。将代码嵌入到您自己的二进制文件中非常容易：

```
func NewDefaultOptions() *Options {
    o := &Options{
        *options.NewSecureServingOptions(),
    }
    o.SecureServing.ServerCert.PairName = "pizza-crd-webhook"
    return o
}

type Options struct {
    SecureServing options.SecureServingOptions
}

type Config struct {
    SecureServing *server.SecureServingInfo
}

func (o *Options) AddFlags(fs *pflag.FlagSet) {
    o.SecureServing.AddFlags(fs)
}

func (o *Options) Config() (*Config, error) {
    err := o.SecureServing.MaybeDefaultWithSelfSignedCerts("0.0.0.0", nil, nil)
    if err != nil {
        return nil, err
    }

    c := &Config{}

    if err := o.SecureServing.ApplyTo(&c.SecureServing); err != nil {
        return nil, err
    }

    return c, nil
}
```

在二进制文件的main函数中，此`Options`结构体被实例化并连接到标志集：

```
opt := NewDefaultOptions()
fs := pflag.NewFlagSet("pizza-crd-webhook", pflag.ExitOnError)
globalflag.AddGlobalFlags(fs, "pizza-crd-webhook")
opt.AddFlags(fs)
if err := fs.Parse(os.Args); err != nil {
    panic(err)
}

// create runtime config
cfg, err := opt.Config()
if err != nil {
    panic(err)
}

stopCh := server.SetupSignalHandler()

...

// run server
restaurantInformers.Start(stopCh)
if doneCh, err := cfg.SecureServing.Serve(
    handlers.LoggingHandler(os.Stdout, mux),
    time.Second * 30, stopCh,
); err != nil {
    panic(err)
} else {
    <-doneCh
}
```

我们用三条路径代替三个点来设置HTTP多路复用器，如下所示：

```
// register handlers
restaurantInformers := restaurantinformers.NewSharedInformerFactory(
    clientset, time.Minute * 5,
)
mux := http.NewServeMux()
mux.Handle("/convert/v1beta1/pizza", http.HandlerFunc(conversion.Serve))
mux.Handle("/admit/v1beta1/pizza", http.HandlerFunc(admission.ServePizzaAdmit))
mux.Handle("/validate/v1beta1/pizza",
    http.HandlerFunc(admission.ServePizzaValidation(restaurantInformers)))
restaurantInformers.Start(stopCh)
```

由于路径上的比萨验证webhook */ validate / v1beta1 / pizza*必须知道集群中现有的顶级对象，我们为`restaurant.programming-kubernetes.info`API组实例化一个共享的informer工厂。

现在我们来看看后面的实际转换webhook实现`conversion.Serve`。它是一个普通的Golang HTTP处理函数，意味着它获取请求和响应编写器作为参数。

请求正文包含`ConversionReview`API组中的对象`apiextensions.k8s.io/v1beta1`。因此，我们必须首先从请求中读取正文，然后解码字节切片。我们使用API Machinery的解串器来完成此操作：

```
func Serve(w http.ResponseWriter, req *http.Request) {
    // read body
    body, err := ioutil.ReadAll(req.Body)
    if err != nil {
        responsewriters.InternalError(w, req,
          fmt.Errorf("failed to read body: %v", err))
        return
    }

    // decode body as conversion review
    gv := apiextensionsv1beta1.SchemeGroupVersion
    reviewGVK := gv.WithKind("ConversionReview")
    obj, gvk, err := codecs.UniversalDeserializer().Decode(body, &reviewGVK,
        &apiextensionsv1beta1.ConversionReview{})
    if err != nil {
        responsewriters.InternalError(w, req,
          fmt.Errorf("failed to decode body: %v", err))
        return
    }
    review, ok := obj.(*apiextensionsv1beta1.ConversionReview)
    if !ok {
        responsewriters.InternalError(w, req,
          fmt.Errorf("unexpected GroupVersionKind: %s", gvk))
        return
    }
    if review.Request == nil {
        responsewriters.InternalError(w, req,
          fmt.Errorf("unexpected nil request"))
        return
    }

    ...
}
```

此代码使用编解码器工厂`codecs`，该工厂派生自方案。该方案必须包括*apiextensions.k8s.io/v1beta1*的类型。我们还添加了我们的餐厅API组的类型。传递的`ConversionReview`对象将我们的披萨类型嵌入一个`runtime.RawExtension`类型 - 更多关于在一秒钟内。

首先让我们创建我们的方案和编解码器工厂：

```
import (
    apiextensionsv1beta1 "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1"
    "github.com/programming-kubernetes/pizza-crd/pkg/apis/restaurant/install"
    ...
)

var (
    scheme = runtime.NewScheme()
    codecs = serializer.NewCodecFactory(scheme)
)

func init() {
    utilruntime.Must(apiextensionsv1beta1.AddToScheme(scheme))
    install.Install(scheme)
}
```

A `runtime.RawExtension`是嵌入在另一个对象的字段中的类似Kubernetes的对象的包装器。它的结构实际上非常简单：

```
type RawExtension struct {
    // Raw is the underlying serialization of this object.
    Raw []byte `protobuf:"bytes,1,opt,name=raw"`
    // Object can hold a representation of this extension - useful for working
    // with versioned structs.
    Object Object `json:"-"`
}
```

另外，`runtime.RawExtension`还有特殊的JSON和protobuf编组两种方法。此外，`runtime.Object`转换为内部类型（即自动编码和解码）时，转换为动态时存在特殊逻辑。

在这种CRD的情况下，我们没有内部类型，因此转换魔法不起作用。仅`RawExtension.Raw`填充发送到webhook进行转换的披萨对象的JSON字节切片。因此，我们必须解码这个字节切片。再次注意，一个`ConversionReview`可能携带许多对象，这样我们就必须遍历所有对象：

```
// convert objects
review.Response = &apiextensionsv1beta1.ConversionResponse{
    UID: review.Request.UID,
    Result:  metav1.Status{
        Status: metav1.StatusSuccess,
    },
}
var objs []runtime.Object
for _, in := range review.Request.Objects {
    if in.Object == nil {
        var err error
        in.Object, _, err = codecs.UniversalDeserializer().Decode(
            in.Raw, nil, nil,
        )
        if err != nil {
            review.Response.Result = metav1.Status{
                Message: err.Error(),
                Status:  metav1.StatusFailure,
            }
            break
        }
    }

    obj, err := convert(in.Object, review.Request.DesiredAPIVersion)
    if err != nil {
        review.Response.Result = metav1.Status{
            Message: err.Error(),
            Status:  metav1.StatusFailure,
        }
        break
    }
    objs = append(objs, obj)
}
```

该`convert`调用`in.Object`使用所需的API版本作为目标版本进行实际转换。请注意，我们会在第一个错误发生时立即中断循环。

最后，我们`Response`在`ConversionReview`对象中设置字段，并使用API Machinery的响应编写器将其写回请求的响应主体，该编写器再次使用我们的编解码器工厂来创建序列化器：

```
if review.Response.Result.Status == metav1.StatusSuccess {
    for _, obj = range objs {
        review.Response.ConvertedObjects =
          append(review.Response.ConvertedObjects,
            runtime.RawExtension{Object: obj},
          )
    }
}

// write negotiated response
responsewriters.WriteObject(
    http.StatusOK, gvk.GroupVersion(), codecs, review, w, req,
)
```

现在，我们必须实施实际的披萨转换。在上面的所有这些管道之后，转换算法是最简单的部分。它只是检查，我们实际上得到了已知版本的比萨饼对象，然后从执行转换`v1beta1`到`v1alpha1`反之亦然：

```
func convert(in runtime.Object, apiVersion string) (runtime.Object, error) {
    switch in := in.(type) {
    case *v1alpha1.Pizza:
        if apiVersion != v1beta1.SchemeGroupVersion.String() {
            return nil, fmt.Errorf("cannot convert %s to %s",
              v1alpha1.SchemeGroupVersion, apiVersion)
        }
        klog.V(2).Infof("Converting %s/%s from %s to %s", in.Namespace, in.Name,
            v1alpha1.SchemeGroupVersion, apiVersion)

        out := &v1beta1.Pizza{
            TypeMeta: in.TypeMeta,
            ObjectMeta: in.ObjectMeta,
            Status: v1beta1.PizzaStatus{
                Cost: in.Status.Cost,
            },
        }
        out.TypeMeta.APIVersion = apiVersion

        idx := map[string]int{}
        for _, top := range in.Spec.Toppings {
            if i, duplicate := idx[top]; duplicate {
                out.Spec.Toppings[i].Quantity++
                continue
            }
            idx[top] = len(out.Spec.Toppings)
            out.Spec.Toppings = append(out.Spec.Toppings, v1beta1.PizzaTopping{
                Name: top,
                Quantity: 1,
            })
        }

        return out, nil

    case *v1beta1.Pizza:
        if apiVersion != v1alpha1.SchemeGroupVersion.String() {
            return nil, fmt.Errorf("cannot convert %s to %s",
              v1beta1.SchemeGroupVersion, apiVersion)
        }
        klog.V(2).Infof("Converting %s/%s from %s to %s",
          in.Namespace, in.Name, v1alpha1.SchemeGroupVersion, apiVersion)

        out := &v1alpha1.Pizza{
            TypeMeta: in.TypeMeta,
            ObjectMeta: in.ObjectMeta,
            Status: v1alpha1.PizzaStatus{
                Cost: in.Status.Cost,
            },
        }
        out.TypeMeta.APIVersion = apiVersion

        for i := range in.Spec.Toppings {
            for j := 0; j < in.Spec.Toppings[i].Quantity; j++ {
                out.Spec.Toppings = append(
                  out.Spec.Toppings, in.Spec.Toppings[i].Name)
            }
        }

        return out, nil

    default:
    }
    klog.V(2).Infof("Unknown type %T", in)
    return nil, fmt.Errorf("unknown type %T", in)
}
```

请注意，在转换的两个方向上，我们只需复制`TypeMeta`并将`ObjectMeta`API版本更改为所需的版本，然后转换topping slice，这实际上是结构上不同的对象的唯一部分。

如果有更多版本，则需要在所有版本之间进行另一次双向转换。或者，当然，我们可以使用中心版本聚合API服务器（请参阅[“内部类型和转换”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-internal-types)），而不是实现与所有受支持的外部版本的转换。

## 部署转换Webhook

我们现在想要部署转换webhook。你可以在[GitHub上](http://bit.ly/2KEx4xo)找到所有的清单。

CRD的转换webhooks在集群中启动并放在服务对象后面，该服务对象由CRD清单中的转换webhook规范引用：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: pizzas.restaurant.programming-kubernetes.info
spec:
  ...
  conversion:
    strategy: Webhook
    webhookClientConfig:
      caBundle: BASE64-CA-BUNDLE
      service:
        namespace: pizza-crd
        name: webhook
        path: /convert/v1beta1/pizza
```

CA捆绑包必须与webhook使用的服务证书匹配。在我们的示例项目中，我们使用[Makefile](http://bit.ly/2FukVac)使用OpenSSL生成证书，并使用文本替换将它们插入到清单中。

请注意，Kubernetes API服务器假定webhook支持所有指定版本的CRD。每个CRD也只有一个这样的webhook。但由于CRD和转换webhook通常由同一个团队拥有，这应该足够了。

另请注意，当前*apiextensions.k8s.io/v1beta1* API 中的服务端口必须为443 。但是，该服务可以将此映射到webhook pod使用的任何端口。在我们的示例中，我们将443映射到8443，由webhook二进制文件提供服务。

## 看到行动中的转换

现在 我们了解转换webhook如何工作以及它如何连接到集群，让我们看看它的实际运行情况。

我们假设您已经检查了示例项目。此外，我们假设您有一个启用了webhook转换的集群（通过1.14集群中的功能门或通过1.15+集群，默认情况下启用了webhook转换）。获得这样一个集群的一种方法是通过[kind项目](http://bit.ly/2X75lvS)，它提供对Kubernetes 1.14.1和本地*kind-config.yaml*文件的支持，以启用webhook转换的alpha功能门（[“Kubernetes是什么意思？”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#programming-kubernetes-meaning)链接了一个开发集群的其他选项数量）：

```
kind: Cluster
apiVersion: kind.sigs.k8s.io/v1alpha3
kubeadmConfigPatchesJson6902:
- group: kubeadm.k8s.io
  version: v1beta1
  kind: ClusterConfiguration
  patch: |
    - op: add
      path: /apiServer/extraArgs
      value: {}
    - op: add
      path: /apiServer/extraArgs/feature-gates
      value: CustomResourceWebhookConversion=true
```

然后我们可以创建一个集群：

```
$ kind create cluster --image kindest / node-images：v1.14.1 --config kind-config.yaml
 $ export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
```

现在我们可以部署[我们的清单](http://bit.ly/2KEx4xo)：

```
$ cd pizza-crd
 $ cd 清单/部署
 $ make
 $ kubectl create -f ns.yaml
 $ kubectl create -f pizza-crd.yaml
 $ kubectl create -f topping-crd.yaml
 $ kubectl create -f sa.yaml
 $ kubectl create -f rbac.yaml
 $ kubectl create -f rbac-bind.yaml
 $ kubectl create -f service.yaml
 $ kubectl create -f serve-cert-secret.yaml
 $ kubectl create -f deployment.yaml
```

这些清单包含以下文件：

- *ns.yaml*

  创建`pizza-crd`命名空间。

- *比萨饼crd.yaml*

  指定`restaurant.programming-kubernetes.info`API组中的披萨资源，包括`v1alpha1`和`v1beta1`版本以及webhook转换配置，如前所示。

- *平顶crd.yaml*

  指定同一API组中的浇头CR，但仅限于`v1alpha1` 版本。

- *sa.yaml*

  介绍`webhook`服务帐户。

- *rbac.yaml*

  定义读取，列出和观察浇头的角色。

- *RBAC-bind.yaml*

  将早期的RBAC角色绑定到`webhook`服务帐户。

- *service.yaml*

  定义`webhook`服务，将webhook pod的端口443映射到8443。

- *服务-CERT-secret.yaml*

  包含webhook pod使用的服务证书和私钥。该证书还可以直接用作前面的披萨CRD清单中的CA捆绑包。

- *deployment.yaml*

  启动webhook pods，传递`--tls-cert-file`和`--tls-private-key`服务证书秘密。

在此之后，我们最终可以创建一个margherita披萨：

```
$ cat ../examples/margherita-pizza.yaml
apiVersion：restaurant.programming-kubernetes.info/v1alpha1
亲切：披萨
元数据：
  名称：玛格丽塔
规格：
  配料：
  - 奶酪
  - 番茄
$ kubectl创建../examples/margherita-pizza.yaml
pizza.restaurant.programming-kubernetes.info/margherita创建
```

现在，通过转换webhook，我们可以在两个版本中检索相同的对象。首先明确在`v1alpha1`版本中：

```
$ kubectl获取pizzas.v1alpha1.restaurant.programming-kubernetes.info \
    margherita -o yaml
apiVersion：restaurant.programming-kubernetes.info/v1alpha1
亲切：披萨
元数据：
  creationTimestamp： "2019-04-14T21:41:39Z"
  一代：1
  名称：玛格丽塔
  命名空间：pizza-crd
  resourceVersion： "18296"
  比萨饼/雏菊
  uid：15c1c06a-5efe-11e9-9230-0242f24ba99c
规格：
  配料：
  - 奶酪
  - 番茄
状态： {}
```

然后相同的对象`v1beta1`显示不同的浇头结构：

```
$ kubectl获取pizzas.v1beta1.restaurant.programming-kubernetes.info \
    margherita -o yaml
apiVersion：restaurant.programming-kubernetes.info/v1beta1
亲切：披萨
元数据：
  creationTimestamp： "2019-04-14T21:41:39Z"
  一代：1
  名称：玛格丽塔
  命名空间：pizza-crd
  resourceVersion： "18296"
  比萨饼/雏菊
  uid：15c1c06a-5efe-11e9-9230-0242f24ba99c
规格：
  配料：
  - 名称：莫扎里拉
    数量：1
  - 名字：番茄
    数量：1
状态： {}
```

同时，在webhook pod的日志中，我们看到此转换调用：

```
I0414 21：46：28.639707 1 convert.go:35]转换pizza-crd / margherita
  来自restaurant.programming-kubernetes.info/v1alpha1
  到restaurant.programming-kubernetes.info/v1beta1
10.32.0.1  -   -  [14 / Apr / 2019：21：46：28 +0000]
  “POST / convert / v1beta1 / pizza？timeout = 30s HTTP / 2.0”200 968
```

因此，webhook正在按预期完成其工作。

# 入场Webhooks

在[“自定义API服务器的用例”中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#uc-custom-api-server)我们讨论了聚合API服务器比使用CR更好的选择的用例。给出的很多原因是关于使用Golang实现某些行为的自由，而不是限制在CRD清单中的声明性功能。

我们在上一节中已经看到Golang如何用于构建CRD转换webhook。类似的机制用于在Golang中添加CRD的自定义许可。

基本上，我们与聚合API服务器中的自定义许可插件具有相同的自由度（请参阅[“许可”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-admission)）：存在变异和验证许可的webhook，并且它们在与本机资源相同的位置调用，[如图9](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-admission-pipeline)所示[-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-admission-pipeline)。

![CR请求管道中的准入](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0905.png)

###### 图9-5。CR请求管道中的准入

我们在[“验证自定义资源”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-validation)看到了基于OpenAPI的CRD验证。在[图9-5中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-admission-pipeline)，验证在标记为“验证”的框中完成。之后调用验证准入webhooks，之前是变异录取webhooks。

在配额之前，准入webhook几乎在准入插件订单的末尾。入场webhooks在Kubernetes 1.14中是测试版，因此可在大多数集群中使用。

###### 小费

对于入场webhooks API的v1，计划允许最多两次通过入场链。这意味着早期的许可插件或webhook可以在一定程度上取决于后来的插件或webhook的输出。因此，未来这种机制将变得更加强大。

## 餐厅示例中的入学要求

该 餐厅示例使用多项内容：

- `spec.toppings`如果是`nil`莫扎里拉奶酪，西红柿和萨拉米香肠，则默认为空。
- 应该从CR JSON中删除未知字段，而不是持久存储`etcd`。
- `spec.toppings` 必须仅包含具有相应顶部对象的浇头。

前两个用例是变异的; 第三个用例纯粹是验证。因此，我们将使用一个变异webhook和一个验证webhook来实现这些步骤。

###### 注意

[通过OpenAPI v3验证模式](http://bit.ly/2ZFH8JY)进行[本机默认](http://bit.ly/2ZFH8JY)工作正在进行中。OpenAPI有一个`default`字段，API服务器将来会应用它。此外，丢弃未知字段将成为每个资源的标准行为，由Kubernetes API服务器通过[称为修剪](http://bit.ly/2Xzt2wm)的[机制完成](http://bit.ly/2Xzt2wm)。

修剪在Kubernetes 1.15中以beta版的形式提供。默认计划在1.16中作为测试版提供。当目标集群中的两个功能都可用时，可以在没有任何webhook的情况下实现前面列表中的两个用例。

## 入场Webhook架构

入场webhooks 在结构上与我们在本章前面看到的转换webhook非常相似。

它们部署在集群中，在服务映射端口443后面放置到pod的某个端口，并使用`AdmissionReview`API组中的审阅对象进行调用`admission.k8s.io/v1beta1`：

```
---
// AdmissionReview describes an admission review request/response.
type AdmissionReview struct {
    metav1.TypeMeta `json:",inline"`
    // Request describes the attributes for the admission request.
    // +optional
    Request *AdmissionRequest `json:"request,omitempty"`
    // Response describes the attributes for the admission response.
    // +optional
    Response *AdmissionResponse `json:"response,omitempty"`
}
---
```

它`AdmissionRequest`包含我们用于接纳属性的所有信息（参见[“实现”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#admission-plug-in-implementation)）：

```
// AdmissionRequest describes the admission.Attributes for the admission request.
type AdmissionRequest struct {
    // UID is an identifier for the individual request/response. It allows us to
    // distinguish instances of requests which are otherwise identical (parallel
    // requests, requests when earlier requests did not modify etc). The UID is
    // meant to track the round trip (request/response) between the KAS and the
    // WebHook, not the user request. It is suitable for correlating log entries
    // between the webhook and apiserver, for either auditing or debugging.
    UID types.UID `json:"uid"`
    // Kind is the type of object being manipulated.  For example: Pod
    Kind metav1.GroupVersionKind `json:"kind"`
    // Resource is the name of the resource being requested.  This is not the
    // kind.  For example: pods
    Resource metav1.GroupVersionResource `json:"resource"`
    // SubResource is the name of the subresource being requested.  This is a
    // different resource, scoped to the parent resource, but it may have a
    // different kind. For instance, /pods has the resource "pods" and the kind
    // "Pod", while /pods/foo/status has the resource "pods", the sub resource
    // "status", and the kind "Pod" (because status operates on pods). The
    // binding resource for a pod though may be /pods/foo/binding, which has
    // resource "pods", subresource "binding", and kind "Binding".
    // +optional
    SubResource string `json:"subResource,omitempty"`
    // Name is the name of the object as presented in the request.  On a CREATE
    // operation, the client may omit name and rely on the server to generate
    // the name.  If that is the case, this method will return the empty string.
    // +optional
    Name string `json:"name,omitempty"`
    // Namespace is the namespace associated with the request (if any).
    // +optional
    Namespace string `json:"namespace,omitempty"`
    // Operation is the operation being performed
    Operation Operation `json:"operation"`
    // UserInfo is information about the requesting user
    UserInfo authenticationv1.UserInfo `json:"userInfo"`
    // Object is the object from the incoming request prior to default values
    // being applied
    // +optional
    Object runtime.RawExtension `json:"object,omitempty"`
    // OldObject is the existing object. Only populated for UPDATE requests.
    // +optional
    OldObject runtime.RawExtension `json:"oldObject,omitempty"`
    // DryRun indicates that modifications will definitely not be persisted
    // for this request.
    // Defaults to false.
    // +optional
    DryRun *bool `json:"dryRun,omitempty"`
}
```

相同的`AdmissionReview`对象用于改变和验证准入webhooks。唯一的区别是，在突变的情况下，`AdmissionResponse`可以有一个字段`patch`和`patchType`，网络挂接已接收到响应之后有要在Kubernetes API服务器内施用。在验证案例中，这两个字段在响应时保持为空。

这里我们目的最重要的领域是`Object`字段，它与前面的转换webhook一样 - 使用`runtime.RawExtension`类型来存储披萨对象。

我们还获取更新请求的旧对象，并且可以检查是否为只读但在请求中更改的字段。我们在这个例子中没有这样做。但是在Kubernetes中会遇到很多情况，例如，对于pod的大多数字段，实现了这样的逻辑，因为在创建pod之后你无法更改它的命令。

变异webhook返回的补丁必须是`Patch`Kubernetes 1.14中的JSON类型（参见RFC 6902）。此修补程序描述了如何修改对象以满足所需的不变量。

请注意，最佳做法是在最后验证验证webhook中的每个变异webhook更改，至少如果这些强制属性对于该行为很重要。想象一下，其他一些变异的webhook触及对象中的相同字段。然后你不能确定变异的变化会持续到变异入场链的结束。

目前没有订单变异除了字母顺序之外的webhooks。目前正在进行讨论，以便在未来以某种方式改变这种状况。

为了验证webhooks，显然，顺序并不重要，Kubernetes API服务器甚至会并行调用验证webhook以减少延迟。相反，变异webhooks会为每个通过它们的请求增加延迟，因为它们是按顺序调用的。

常见的延迟 - 当然严重依赖于环境 - 大约是100毫秒。因此，按顺序运行许多webhook会导致用户在创建或更新对象时会遇到相当大的延迟。

## 注册入场Webhooks

入场webhooks未在CRD清单中注册。原因是它们不仅适用于CRD，也适用于任何类型的资源。您甚至可以将自定义录取webhook添加到标准Kubernetes资源中。

而是有注册对象：`MutatingWebhookRegistration`和`ValidatingWebhookRegistration`。它们只在种类名称上有所不同：

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  name: restaurant.programming-kubernetes.info
webhooks:
- name: restaurant.programming-kubernetes.info
  failurePolicy: Fail
  sideEffects: None
  admissionReviewVersions:
  - v1beta1
  rules:
  - apiGroups:
    - "restaurant.programming-kubernetes.info"
    apiVersions:
    - v1alpha1
    - v1beta1
    operations:
    - CREATE
    - UPDATE
    resources:
    - pizzas
  clientConfig:
    service:
      namespace: pizza-crd
      name: webhook
      path: /admit/v1beta1/pizza
    caBundle: CA-BUNDLE
```

这将注册我们`pizza-crd`从入院突变对我们资源的两个版本本章开头网络挂接`pizza`的API组`restaurant.programming-kubernetes.info`，和HTTP动词`CREATE`及`UPDATE`（其中包括补丁以及）。

webhook配置中还有其他方法来限制匹配资源 - 例如，命名空间选择器（以排除例如控制平面命名空间以避免引导问题）以及具有通配符和子资源的更高级资源模式。

最后但并非最不重要的是失败模式，可以是`Fail`或者`Ignore`。它指定如果由于其他原因无法访问或失败webhook时要执行的操作。

###### 警告

如果以错误的方式部署，则入场webhook可能会破坏群集。允许webhook匹配核心类型可以使整个集群无法运行。必须特别注意为非CRD资源调用入场webhook。

具体来说，最好从webhook中排除控制平面和webhook资源本身。

## 实施招生Webhook

同我们在本章开头的转换webhook上所做的工作，不难添加录取功能。我们还看到路径*/ admit / v1beta1 / pizza*和*/ validate / v1beta1 / pizza*在`pizza-crd-webhook`二进制文件的main函数中注册：

```
mux.Handle("/admit/v1beta1/pizza", http.HandlerFunc(admission.ServePizzaAdmit))
mux.Handle("/validate/v1beta1/pizza", http.HandlerFunc(
admission.ServePizzaValidation(restaurantInformers)))
```

两个HTTP处理程序实现的第一部分看起来几乎与转换webhook相同：

```
func ServePizzaAdmit(w http.ResponseWriter, req *http.Request) {
    // read body
    body, err := ioutil.ReadAll(req.Body)
    if err != nil {
        responsewriters.InternalError(w, req,
          fmt.Errorf("failed to read body: %v", err))
        return
    }

    // decode body as admission review
    reviewGVK := admissionv1beta1.SchemeGroupVersion.WithKind("AdmissionReview")
    decoder := codecs.UniversalDeserializer()
    into := &admissionv1beta1.AdmissionReview{}
    obj, gvk, err := decoder.Decode(body, &reviewGVK, into)
    if err != nil {
        responsewriters.InternalError(w, req,
          fmt.Errorf("failed to decode body: %v", err))
        return
    }
    review, ok := obj.(*admissionv1beta1.AdmissionReview)
    if !ok {
        responsewriters.InternalError(w, req,
          fmt.Errorf("unexpected GroupVersionKind: %s", gvk))
        return
    }
    if review.Request == nil {
        responsewriters.InternalError(w, req,
          fmt.Errorf("unexpected nil request"))
        return
    }

    ...
}
```

在验证webhook的情况下，我们必须连接informer（用于检查集群中是否存在浇头）。只要未同步informer，我们就会返回内部错误。未同步的线人有不完整的数据，因此可能不知道配料，虽然披萨有效，但披萨会被拒绝：

```
func ServePizzaValidation(informers restaurantinformers.SharedInformerFactory)
    func (http.ResponseWriter, *http.Request)
{
    toppingInformer := informers.Restaurant().V1alpha1().Toppings().Informer()
    toppingLister := informers.Restaurant().V1alpha1().Toppings().Lister()

    return func(w http.ResponseWriter, req *http.Request) {
        if !toppingInformer.HasSynced() {
            responsewriters.InternalError(w, req,
              fmt.Errorf("informers not ready"))
            return
        }

        // read body
        body, err := ioutil.ReadAll(req.Body)
        if err != nil {
            responsewriters.InternalError(w, req,
              fmt.Errorf("failed to read body: %v", err))
            return
        }

        // decode body as admission review
        gv := admissionv1beta1.SchemeGroupVersion
        reviewGVK := gv.WithKind("AdmissionReview")
        obj, gvk, err := codecs.UniversalDeserializer().Decode(body, &reviewGVK,
            &admissionv1beta1.AdmissionReview{})
        if err != nil {
            responsewriters.InternalError(w, req,
              fmt.Errorf("failed to decode body: %v", err))
            return
        }
        review, ok := obj.(*admissionv1beta1.AdmissionReview)
        if !ok {
            responsewriters.InternalError(w, req,
              fmt.Errorf("unexpected GroupVersionKind: %s", gvk))
            return
        }
        if review.Request == nil {
            responsewriters.InternalError(w, req,
              fmt.Errorf("unexpected nil request"))
            return
        }

        ...
    }
}
```

与webhook转换案例一样，我们已经设置了方案和编解码器工厂以及许可API组和我们的餐厅API组：

```
var (
    scheme = runtime.NewScheme()
    codecs = serializer.NewCodecFactory(scheme)
)

func init() {
    utilruntime.Must(admissionv1beta1.AddToScheme(scheme))
    install.Install(scheme)
}
```

有了这两个，我们解码嵌入式披萨对象（这次只有一个，没有切片）`AdmissionReview`：

```
// decode object
if review.Request.Object.Object == nil {
    var err error
    review.Request.Object.Object, _, err =
      codecs.UniversalDeserializer().Decode(review.Request.Object.Raw, nil, nil)
    if err != nil {
        review.Response.Result = &metav1.Status{
            Message: err.Error(),
            Status:  metav1.StatusFailure,
        }
        responsewriters.WriteObject(http.StatusOK, gvk.GroupVersion(),
          codecs, review, w, req)
        return
    }
}
```

然后我们可以进行实际的变异录入（`spec.toppings`两个API版本的默认）：

```
orig := review.Request.Object.Raw
var bs []byte
switch pizza := review.Request.Object.Object.(type) {
case *v1alpha1.Pizza:
    // default toppings
    if len(pizza.Spec.Toppings) == 0 {
        pizza.Spec.Toppings = []string{"tomato", "mozzarella", "salami"}
    }
    bs, err = json.Marshal(pizza)
    if err != nil {
        responsewriters.InternalError(w, req,
          fmt.Errorf"unexpected encoding error: %v", err))
        return
    }

case *v1beta1.Pizza:
    // default toppings
    if len(pizza.Spec.Toppings) == 0 {
        pizza.Spec.Toppings = []v1beta1.PizzaTopping{
            {"tomato", 1},
            {"mozzarella", 1},
            {"salami", 1},
        }
    }
    bs, err = json.Marshal(pizza)
    if err != nil {
        responsewriters.InternalError(w, req,
          fmt.Errorf("unexpected encoding error: %v", err))
        return
    }

default:
    review.Response.Result = &metav1.Status{
        Message: fmt.Sprintf("unexpected type %T", review.Request.Object.Object),
        Status:  metav1.StatusFailure,
    }
    responsewriters.WriteObject(http.StatusOK, gvk.GroupVersion(),
      codecs, review, w, req)
    return
}
```

或者，我们可以使用转换webhook中的转换算法，然后仅针对其中一个版本实现默认。这两种方法都是可能的，哪种更有意义取决于上下文。这里，默认很简单，可以实现两次。

最后一步是计算补丁 - 原始对象（`orig`以JSON格式存储）与新默认对象之间的差异：

```
// compare original and defaulted version
ops, err := jsonpatch.CreatePatch(orig, bs)
if err != nil {
    responsewriters.InternalError(w, req,
        fmt.Errorf("unexpected diff error: %v", err))
    return
}
review.Response.Patch, err = json.Marshal(ops)
if err != nil {
    responsewriters.InternalError(w, req,
    fmt.Errorf("unexpected patch encoding error: %v", err))
    return
}
typ := admissionv1beta1.PatchTypeJSONPatch
review.Response.PatchType = &typ
review.Response.Allowed = true
```

我们使用[JSON-Patch库](http://bit.ly/2IKxwIk)（[Matt Baird的一个](http://bit.ly/2xfBIsN)带有[关键修复](http://bit.ly/2XxKfWP)的分支）从原始对象`orig`和修改后的对象派生补丁`bs`，两者都作为JSON字节切片传递。或者，我们可以直接操作非类型化的JSON数据并手动创建JSON-Patch。同样，它取决于上下文。使用diff库很方便。

然后，就像在webhook转换中一样，我们通过使用先前创建的编解码器工厂将响应写入响应编写器来结束：

```
responsewriters.WriteObject(
    http.StatusOK, gvk.GroupVersion(), codecs, review, w, req,
)
```

验证webhook非常相似，但它使用共享informer中的toppings lister来检查顶部对象是否存在：

```
switch pizza := review.Request.Object.Object.(type) {
case *v1alpha1.Pizza:
    for _, topping := range pizza.Spec.Toppings {
        _, err := toppingLister.Get(topping)
        if err != nil && !errors.IsNotFound(err) {
            responsewriters.InternalError(w, req,
              fmt.Errorf("failed to lookup topping %q: %v", topping, err))
            return
        } else if errors.IsNotFound(err) {
            review.Response.Result = &metav1.Status{
                Message: fmt.Sprintf("topping %q not known", topping),
                Status:  metav1.StatusFailure,
            }
            responsewriters.WriteObject(http.StatusOK, gvk.GroupVersion(),
              codecs, review, w, req)
            return
        }
    }
    review.Response.Allowed = true
case *v1beta1.Pizza:
    for _, topping := range pizza.Spec.Toppings {
        _, err := toppingLister.Get(topping.Name)
        if err != nil && !errors.IsNotFound(err) {
            responsewriters.InternalError(w, req,
              fmt.Errorf("failed to lookup topping %q: %v", topping, err))
            return
        } else if errors.IsNotFound(err) {
            review.Response.Result = &metav1.Status{
                Message: fmt.Sprintf("topping %q not known", topping),
                Status:  metav1.StatusFailure,
            }
            responsewriters.WriteObject(http.StatusOK, gvk.GroupVersion(),
              codecs, review, w, req)
            return
        }
    }
    review.Response.Allowed = true
default:
    review.Response.Result = &metav1.Status{
        Message: fmt.Sprintf("unexpected type %T", review.Request.Object.Object),
        Status:  metav1.StatusFailure,
    }
}
responsewriters.WriteObject(http.StatusOK, gvk.GroupVersion(),
      codecs, review, w, req)
```

## 入场Webhook in Action

我们 通过在集群中创建两个注册对象来部署两个准入webhook：

```
$ kubectl create -f validatingadmissionregistration.yaml
 $ kubectl create -f mutatingadmissionregistration.yaml
```

在此之后，我们再也无法制作带有未知浇头的比萨饼：

```
$ kubectl create -f ../examples/margherita-pizza.yaml
服务器出错"../examples/margherita-pizza.yaml"：创建时出错：
  录取webhook "restaurant.programming-kubernetes.info"否认了请求：
    打顶"tomato"未知
```

同时，在webhook日志中我们看到：

```
I0414 22:45:46.873541 1 pizzamutation.go:115]违约披萨-dd / in
  版本admission.k8s.io/v1beta1，Kind = AdmissionReview
10.32.0.1  -   -  [14 / Apr / 2019：22：45：46 +0000]
  “POST / admit / v1beta1 / pizza？timeout = 30s HTTP / 2.0”200 871
10.32.0.1  -   -  [14 / Apr / 2019：22：45：46 +0000]
  “POST / validate / v1beta1 / pizza？timeout = 30s HTTP / 2.0”200 956
```

在示例文件夹中创建浇头后，我们可以再次创建margherita披萨：

```
$ kubectl create -f ../examples/topping-tomato.yaml
 $ kubectl create -f ../examples/topping-salami.yaml
 $ kubectl create -f ../examples/topping-mozzarella.yaml
 $ kubectl create -f ../examples /margherita-pizza.yaml
pizza.restaurant.programming-kubernetes.info/margherita创建
```

最后但并非最不重要的是，让我们检查默认是否按预期工作。我们想要创建一个空的披萨：

```
apiVersion: restaurant.programming-kubernetes.info/v1alpha1
kind: Pizza
metadata:
  name: salami
spec:
```

这应该是默认为萨拉米香肠披萨，它是：

```
$ kubectl create -f ../examples/empty-pizza.yaml
pizza.restaurant.programming-kubernetes.info/salami created
$ kubectl get pizza salami -o yaml
apiVersion: restaurant.programming-kubernetes.info/v1beta1
kind: Pizza
metadata:
  creationTimestamp: "2019-04-14T22:49:40Z"
  generation: 1
  name: salami
  namespace: pizza-crd
  resourceVersion: "23227"
  uid: 962e2dda-5f07-11e9-9230-0242f24ba99c
spec:
  toppings:
  - name: tomato
    quantity: 1
  - name: mozzarella
    quantity: 1
  - name: salami
    quantity: 1
status: {}
```

Voilà，一种萨拉米香肠披萨，含有我们所期望的所有配料。请享用！

在结束本章之前，我们希望了解`apiextensions.k8s.io/v1`CRD 的API组版本（即，nonbeta，一般可用性） - 即结构模式的引入。

# 结构模式和CustomResourceDefinitions的未来

从Kubernetes 1.15，OpenAPI v3验证模式（参见[“验证自定义资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-validation)）正在为CRD发挥更重要的作用，因为如果使用任何这些新功能，必须指定模式：

- CRD转换（[见图9-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-conversion-webhook)）
- 修剪（参见[“修剪与保留未知领域”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-pruning)）
- 默认（参见[“默认值”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#crd-defaulting)）
- OpenAPI架构[发布](http://bit.ly/2RzeA1O)

严格地说，模式的定义仍然是可选的，并且每个现有的CRD都将继续工作，但是如果没有模式，您的CRD将被排除在任何新功能之外。

此外，指定的模式必须遵循某些规则，以强制指定的类型在遵守[Kubernetes API约定](http://bit.ly/2Nfd9Hn)的意义上实际上是理智的。我们称之为*结构模式*。

## 结构模式

结构模式是遵循以下规则的OpenAPI v3验证模式（请参阅[“验证自定义资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-validation)）：

1. 模式`type`为根，对象节点的每个指定字段（通过`properties`或`additionalProperties`在OpenAPI中）以及对于数组节点中的每个项（通过OpenAPI）指定非空类型（`items`在OpenAPI中），但以下情况除外：
   - 一个节点 `x-kubernetes-int-or-string: true`
   - 一个节点 `x-kubernetes-preserve-unknown-fields: true`
2. 用于在对象中的每个字段，并在阵列中，这是设置内的每个项目`allOf`，`anyOf`，`oneOf`，或`not`，该架构还指定那些逻辑junctors外部的场/项目。
3. 该方案不设`description`，`type`，`default`，`additionProperties`，或`nullable`内`allOf`，`anyOf`，`oneOf`，或者`not`，与两种模式的例外`x-kubernetes-int-or-string: true`（见[“IntOrString和RawExtensions”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#intorstring-section)）。
4. 如果`metadata`指定，则仅限制`metadata.name`和`metadata.generateName`允许。

这是一个非结构性的例子：

```
properties:
  foo:
    pattern: "abc"
  metadata:
    type: object
    properties:
      name:
        type: string
        pattern: "^a"
      finalizers:
        type: array
        items:
          type: string
          pattern: "my-finalizer"
anyOf:
- properties:
    bar:
      type: integer
      minimum: 42
  required: ["bar"]
  description: "foo bar object"
```

由于以下违规行为，它不是结构模式：

- 缺少根的类型（规则1）。
- `foo`缺少的类型（规则1）。
- `bar`内部`anyOf`未指定（规则2）。
- `bar`的`type`是内`anyOf`（规则3）。
- 描述在`anyOf`（规则3）内设定。
- `metadata.finalizer` 可能不受限制（规则4）。

相比之下，以下相应的架构是结构性的：

```
type: object
description: "foo bar object"
properties:
  foo:
    type: string
    pattern: "abc"
  bar:
    type: integer
  metadata:
    type: object
    properties:
      name:
        type: string
        pattern: "^a"
anyOf:
- properties:
    bar:
      minimum: 42
  required: ["bar"]
```

`NonStructural`在CRD 的条件中报告违反结构模式规则。

验证自己的的模式`cnat`在例如[“确认自定义资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-validation)，并在该模式[比萨饼CRD例子](http://bit.ly/31MrFcO)确实是结构。

## 修剪与保留未知领域

CRD传统上存储任何（可能经过验证的）JSON `etcd`。这意味着未指定字段（如果存在的OpenAPI V3验证架构在所有）将持续。这与本地Kubernetes资源（如pod）形成鲜明对比。如果用户指定了一个字段`spec.randomField`，那么API服务器HTTPS端点将接受该字段，但在将该pod写入之前将其删除（我们称之为*修剪*）`etcd`。

如果定义了结构OpenAPI v3验证模式（在全局`spec.validation.openAPIV3Schema`或每个版本中），我们可以通过设置`spec.preserveUnknownFields`为启用修剪（在创建和更新时删除未指定的字段）`false`。

我们来看看这个`cnat`例子。[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#idm46336839329896)使用Kubernetes 1.15集群，我们启用修剪：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  ...
  preserveUnknownFields: false
```

然后我们尝试创建一个具有未知字段的实例：

```
apiVersion: cnat.programming-kubernetes.info/v1alpha1
kind: At
metadata:
  name: example-at
spec:
  schedule: "2019-07-03T02:00:00Z"
  command: echo "Hello, world!"
  someGarbage: 42
```

如果我们检索这个对象`kubectl get at example-at`，我们看到该`someGarbage`值被删除：

```
apiVersion: cnat.programming-kubernetes.info/v1alpha1
kind: At
metadata:
  name: example-at
spec:
  schedule: "2019-07-03T02:00:00Z"
  command: echo "Hello, world!"
```

我们说这`someGarbage`已被*修剪*。

从Kubernetes 1.15开始，*apiextensions / v1beta1中*提供修剪，但默认为关闭; 也就是说，`spec.preserveUnknownFields`默认为`true`。在*apiextensions / v1中*，`spec.preserveUnknownFields: true`不允许创建新的CRD 。

## 控制修剪

随着`spec.preserveUnknownField: false`在对于该类型和所有版本的所有CR，都启用了CRD修剪。但是，可以通过`x-kubernetes-preserve-unknown-fields: true`OpenAPI v3验证模式选择不修剪JSON子树：

```
type: object
properties:
  json:
    x-kubernetes-preserve-unknown-fields: true
```

该字段`json`可以存储任何JSON值，而无需修剪任何内容。

可以部分指定允许的JSON：

```
type: object
properties:
  json:
    x-kubernetes-preserve-unknown-fields: true
    type: object
    description: this is arbitrary JSON
```

使用此方法，仅允许对象类型值。

为每个指定的属性（或`additionalProperties`）再次启用修剪：

```
type: object
properties:
  json:
    x-kubernetes-preserve-unknown-fields: true
    type: object
    properties:
      spec:
        type: object
        properties:
          foo:
            type: string
          bar:
            type: string
```

有了这个，价值：

```
json:
  spec:
    foo: abc
    bar: def
    something: x
  status:
    something: x
```

将被修剪为：

```
json:
  spec:
    foo: abc
    bar: def
  status:
    something: x
```

这意味着修剪*something*指定`spec`对象中的字段（因为指定了“spec”），但外部的所有内容都没有。`status`未指定未修剪的内容。`status.*something*`

## IntOrString和RawExtensions

那里结构模式不够表达的情况。其中之一是*多态*字段 - 可以是不同类型的字段。我们`IntOrString`从本地Kubernetes API类型中了解到。

它可以`IntOrString`使用`x-kubernetes-int-or-string: true`模式中的指令在CRD中使用。同样，`runtime.RawExtensions`可以使用声明`x-kubernetes-embedded-object: true`。

例如：

```
type: object
properties:
  intorstr:
    type: object
    x-kubernetes-int-or-string: true
  embedded:
    x-kubernetes-embedded-object: true
    x-kubernetes-preserve-unknown-fields: true
```

这声明：

- 一个名为的字段`intorstr`包含整数或字符串
- 一个名为的字段`embedded`包含类似Kubernetes的对象，例如完整的pod规范

有关这些指令的所有详细信息，请参阅[CRD官方文档](http://bit.ly/2Lnmw61)。

我们要讨论的最后一个主题取决于结构模式是默认的。

## 默认值

在原生Kubernetes类型，通常默认某些值。只有通过改变录取webhooks（参见[“Admission Webhooks”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#admission-webhooks)），CRD才能实现默认。但是，从Kubernetes 1.15开始，直接通过上一节中描述的OpenAPI v3架构向CRD 添加了默认支持（参见[设计文档](http://bit.ly/2ZFH8JY)）。

###### 注意

从1.15开始，这仍然是一个alpha功能，这意味着默认情况下它被禁用特色门`CustomResourceDefaulting`。但随着升级到beta，可能在1.16，它将在CRD中无处不在。

要默认某些字段，只需通过`default`OpenAPI v3架构中的关键字指定默认值。在向类型添加新字段时，这非常有用。

`cnat`从[“验证自定义资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-validation)中的示例的模式开始，假设我们想要使容器图像可自定义，但默认为`busybox`图像。为此，我们将`image`字符串类型字段添加到OpenAPI v3架构，并将默认值设置为`busybox`：

```
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
      image:
        type: string
        default: "busybox"
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

如果用户在未指定图像的情况下创建实例，则会自动设置该值：

```
apiVersion: cnat.programming-kubernetes.info/v1alpha1
kind: At
metadata:
  name: example-at
spec:
  schedule: "2019-07-03T02:00:00Z"
  command: echo "hello world!"
```

在创建时，它会自动变为：

```
apiVersion: cnat.programming-kubernetes.info/v1alpha1
kind: At
metadata:
  name: example-at
spec:
  schedule: "2019-07-03T02:00:00Z"
  command: echo "hello world!"
  image: busybox
```

这看起来非常方便，并显着改善了CRD的用户体验。而且，`etcd`从API服务器读取时，所有保留的旧对象将自动继承新字段。[3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#idm46336838744152)

请注意，`etcd`不会重写持久化对象（即自动迁移）。换句话说，在读取时，默认值仅在运行时添加，并且仅在由于其他原因更新对象时保留。

# 摘要

入场和转换webhooks使CRD达到完全不同的水平。在这些功能出现之前，CR主要用于小型，不那么严重的用例，通常用于配置和API兼容性不那么重要的内部应用程序。

使用webhooks，CR看起来更像是本机资源，具有很长的生命周期和强大的语义。我们已经了解了如何实现不同资源之间的依赖关系以及如何设置字段的默认值。

此时，您可能对现有CRD中可以使用这些功能的位置有很多想法。我们很想看到未来基于这些功能的社区创新。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#idm46336844085864-marker) `apiextensions.k8s.io`并且`admissionregistration.k8s.io`都计划在Kubernetes 1.16中升级到v1。

[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#idm46336839329896-marker)我们使用`cnat`示例而不是披萨示例，因为前者的结构简单 - 例如，只有一个版本。当然，所有这些都可以扩展到多个版本（即一个模式版本）。

[3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#idm46336838744152-marker)例如，via`kubectl get ats -o yaml`。
# 第6章编写operator

到目前为止，在第一章[“控制器和operator”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#ch_controllers-operators)我们从概念上了解了自定义控制器和operator，在[第5章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)，我们讨论了如何使用Kubernetes代码生成器 - 一种在低层次处理这个问题的方法。在本章中，我们将介绍三个开发自定义控制器和操作员的方案，并介绍更多的可选方案。

使用本章中讨论的3个解决方案中任何一个都会帮助您避免编写大量重复代码，并使您能够专注于业务逻辑，而不是框架代码。它应该会让您更快地上手并提高您的工作效率。

###### 注意

目前（2019年中期），我们在本章中讨论到的operator开发工具仍在迅速发展。您在此处看到的某些命令或输出可能与您在使用时看到的不完全相同。基于这点考虑，我们建议您始终使用相应工具的最新版本，密切关注相应的问题跟踪，邮件列表和Slack通道，寻找和解决您遇到的问题。

对于本文讨论的解决方案，网上已经有一些[比较](http://bit.ly/2ZC5fZT)。这里我们不会向您推荐具体的解决方案，但是我们鼓励您自己评估和比较它们，并选择最适合您的那个。

# 准备

我们将使用`cnat`（云原生的`at`示例，详见[“示例”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#mot-example) ）作为本章中不同解决方案的示例。如果你想使用这个列子，请检查您的环境是否具备：

1. 安装了Go1.12或更高版本并正确设置。
2. 能够正常访问Kubernetes1.12或以上版本的集群并且正确配置了`kubectl`，能够访问（可以通过`kind`或`k3d`本机搭建集群，也可以通过远程访问云服务提供商的集群）
3. `git clone`我们提供的[GitHub存储库](http://bit.ly/2N3R6U4)。这个代码是完整、可用的，可以结合下文介绍的命令来操作。请注意，我们将会从头开始演示。如果您想查看结果，可以直接克隆存储库并执行命令来安装CRD，安装CR并启动自定义控制器。

做完准备工作，让我们开始编写operator：我们将介绍三种方式`sample-controller`，Kubebuilder和Operator SDK。

准备好了吗？让我们开始吧！

# sample-controller方式

让我们基于[*k8s.io/sample-controller*](http://bit.ly/2UppsTN)来实现`cnat`，它[直接](http://bit.ly/2Yas9HK)使用了`client-go`库。`sample-controller`使用[*k8s.io/code-generator*](http://bit.ly/2Kw8I8U)生成一个类型化的客户端，informers，listers，以及深拷贝方法。每当自定义控制器中的API类型发生变化时（ 例如，在自定义资源中添加新字段）您必须使用*update-codegen.sh*脚本来重新生成源代码文件（请参阅GitHub中的[源代码](http://bit.ly/2Fq3Td1)）。

###### 警告

您可能已经注意到*k8s.io*被用作整本书中的基本URL。我们在[第3章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#ch_client-go)介绍了它的用法; 需要注意，它实际上是*kubernetes.io*的别名，在Go语言的包管理中，它被解析为*github.com/kubernetes*。而*k8s.io*不会自动转换。所以，对于*k8s.io / sample-control*包路径对应的源代码路径应该查看[*github.com/kubernetes/sample-controller*](http://bit.ly/2UppsTN)。

好的，让我们按照`sample-controller`的方式，使用`client-go`实现我们的[`cnat`](http://bit.ly/2RpHhON)operator。（请参阅[我们仓库中的相应目录](http://bit.ly/2N3R6U4)）

## 引导

开始执行**go get k8s.io/sample-controller** 命令，将下载源代码和依赖项到我们本机，它将被下载到$ GOPATH / src / k8s.io / sample-controller目录中。

将*sample-controller*目录的内容复制到您选择的目录中（例如，我们在repo中使用*cnat-client-go*），您可以顺序运行以下命令来构建和运行基本的控制器（包含默认实现，还没有加入cnat业务逻辑）：

```shell
# build custom controller binary:
$ go build -o cnat-controller .

# launch custom controller locally:
$ ./cnat-controller -kubeconfig=$HOME/.kube/config
```

此命令将启动自定义控制器，等待您注册CRD并创建自定义资源。在第二个终端，输入：

```shell
$ kubectl apply -f artifacts/examples/crd.yaml
```

想确认CRD已正确注册，可以执行命令：

```shell
$ kubectl get crds
NAME                            CREATED AT
foos.samplecontroller.k8s.io    2019-05-29T12:16:57Z
```

您应该在返回信息中看到很多CRD，实际的返回的CRD信息列表与您的环境相关，但是其中应该包含*foos.samplecontroller.k8s.io*。

接下来，我们创建自定义资源实例*foo.samplecontroller.k8s.io/example-foo*，并检查控制器是否正常执行了它的业务逻辑：

```shell
$ kubectl apply -f artifacts/examples/example-foo.yaml
foo.samplecontroller.k8s.io/example-foo created

$ kubectl get po,rs,deploy,foo
NAME                                           READY   STATUS    RESTARTS   AGE
pod/example-foo-5b8c9679d8-xjhdf               1/1     Running   0          67s

NAME                                           DESIRED   CURRENT   READY AGE
replicaset.extensions/example-foo-5b8c9679d8   1         1         1     67s

NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/example-foo              1/1     1            1           67s

NAME                                           AGE
foo.samplecontroller.k8s.io/example-foo        67s
```

可以看到，它正常执行了！我们现在可以继续实现cnat业务逻辑。

## 业务逻辑

下一步实现业务逻辑，我们首先将现有目录*pkg / apis / samplecontroller*重命名为*pkg / apis / cnat*，然后创建我们自己的CRD和自定义资源，如下所示：

```shell
$ cat artifacts/examples/cnat-crd.yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  group: cnat.programming-kubernetes.info
  version: v1alpha1
  names:
    kind: At
    plural: ats
  scope: Namespaced

$ cat artifacts/examples/cnat-example.yaml
apiVersion: cnat.programming-kubernetes.info/v1alpha1
kind: At
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: example-at
spec:
  schedule: "2019-04-12T10:12:00Z"
  command: "echo YAY"
```

请注意，每当API类型发生更改时（例如，当您向`At` CRD类型添加新字段时），您必须执行*update-codegen.sh*脚本，如下所示：

```shell
$ ./hack/update-codegen.sh
```

这将自动生成以下内容：

- *pkg/apis /cnat/v1alpha1/zz_generated.deepcopy.go*
- pkg/generated/*

在业务逻辑方面，我们在operator中实现了两个部分：

- 在[*types.go中*](http://bit.ly/31QosJw)我们修改`AtSpec`结构添加相应的字段，例如`schedule`和`command`。请注意，每当您在此处更改某些内容时都必须运行`update-codegen.sh`，以便重新生成相关文件。
- 在[*controller.go中，*](http://bit.ly/31MM4OS)我们更改`NewController()`和`syncHandler()`函数以及添加帮助函数，包括创建pods和检查调度时间。

在*types.go中*，请注意代表`At`资源三个阶段的三个常量：在预定的时间之前处于`PENDING`，然后开始执行时`RUNNING`状态，最后完成是`DONE`状态：

```go
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

const (
    PhasePending = "PENDING"
    PhaseRunning = "RUNNING"
    PhaseDone    = "DONE"
)

// AtSpec defines the desired state of At
type AtSpec struct {
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"`
    // Command is the desired command (executed in a Bash shell) to be
    // executed.
    Command string `json:"command,omitempty"`
}

// AtStatus defines the observed state of At
type AtStatus struct {
    // Phase represents the state of the schedule: until the command is
    // executed it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"`
}
```

请注意显式定义的标记用法`+k8s:deepcopy-gen:interfaces`（请参阅[第5章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)），用于自动生成相应的源代码。

我们现在可以实现自定义控制器的业务逻辑。也就是说，我们在 [controller.go](http://bit.ly/31MM4OS)中实现三个状态`PhasePending`、`PhaseRunning`和`PhaseDone`的状态转换。

在[“工作队列”中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#workqueue)我们介绍了`client-go`提供的工作队列。现在，我们来实践一下：在文件*controller.go*中`processNextWorkItem()`方法（准确地说，在[176行至186行](http://bit.ly/2WYDbyi) ）有以下（生成）的代码：

```go
if when, err := c.syncHandler(key); err != nil {
    c.workqueue.AddRateLimited(key)
    return fmt.Errorf("error syncing '%s': %s, requeuing", key, err.Error())
} else if when != time.Duration(0) {
    c.workqueue.AddAfter(key, when)
} else {
    // Finally, if no error occurs we Forget this item so it does not
    // get queued again until another change happens.
    c.workqueue.Forget(obj)
}
```

这个代码片段展示了`syncHandler()`是如何调用我们的自定义函数（尚未编写的，稍后解释）它处理以下三种逻辑：

1. 第一个`if`分支当错误发生时，通过`AddRateLimited()`方法调用重新入队操作。
2. 第二个分支，`else if`通过`AddAfter()`方法调用重新入队操作以避免hot-looping。
3. 最后一种情况`else`是，该事件对象已成功处理，使用Forget()方法丢弃。

现在我们已经对通用处理有了充分的了解，让我们继续讨论特定于业务逻辑的功能。关键是上述`syncHandler()`功能，我们正在实现自定义控制器的业务逻辑。它有以下签名：

```go
// syncHandler compares the actual state with the desired state and attempts
// to converge the two. It then updates the Status block of the At resource
// with the current status of the resource. It returns how long to wait
// until the schedule is due.
func (c *Controller) syncHandler(key string) (time.Duration, error) {
    ...
}
```

此`syncHandler()`函数实现以下状态转换：[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#idm46336858995208)

```go
...
// If no phase set, default to pending (the initial phase):
if instance.Status.Phase == "" {
    instance.Status.Phase = cnatv1alpha1.PhasePending
}

// Now let's make the main case distinction: implementing
// the state diagram PENDING -> RUNNING -> DONE
switch instance.Status.Phase {
case cnatv1alpha1.PhasePending:
    klog.Infof("instance %s: phase=PENDING", key)
    // As long as we haven't executed the command yet, we need
    // to check if it's time already to act:
    klog.Infof("instance %s: checking schedule %q", key, instance.Spec.Schedule)
    // Check if it's already time to execute the command with a
    // tolerance of 2 seconds:
    d, err := timeUntilSchedule(instance.Spec.Schedule)
    if err != nil {
        utilruntime.HandleError(fmt.Errorf("schedule parsing failed: %v", err))
        // Error reading the schedule - requeue the request:
        return time.Duration(0), err
    }
    klog.Infof("instance %s: schedule parsing done: diff=%v", key, d)
    if d > 0 {
        // Not yet time to execute the command, wait until the
        // scheduled time
        return d, nil
    }

    klog.Infof(
       "instance %s: it's time! Ready to execute: %s", key,
       instance.Spec.Command,
    )
    instance.Status.Phase = cnatv1alpha1.PhaseRunning
case cnatv1alpha1.PhaseRunning:
    klog.Infof("instance %s: Phase: RUNNING", key)

    pod := newPodForCR(instance)

    // Set At instance as the owner and controller
    owner := metav1.NewControllerRef(
        instance, cnatv1alpha1.SchemeGroupVersion.
        WithKind("At"),
    )
    pod.ObjectMeta.OwnerReferences = append(pod.ObjectMeta.OwnerReferences, *owner)

    // Try to see if the pod already exists and if not
    // (which we expect) then create a one-shot pod as per spec:
    found, err := c.kubeClientset.CoreV1().Pods(pod.Namespace).
        Get(pod.Name, metav1.GetOptions{})
    if err != nil && errors.IsNotFound(err) {
        found, err = c.kubeClientset.CoreV1().Pods(pod.Namespace).Create(pod)
        if err != nil {
            return time.Duration(0), err
        }
        klog.Infof("instance %s: pod launched: name=%s", key, pod.Name)
    } else if err != nil {
        // requeue with error
        return time.Duration(0), err
    } else if found.Status.Phase == corev1.PodFailed ||
        found.Status.Phase == corev1.PodSucceeded {
        klog.Infof(
            "instance %s: container terminated: reason=%q message=%q",
            key, found.Status.Reason, found.Status.Message,
        )
        instance.Status.Phase = cnatv1alpha1.PhaseDone
    } else {
        // Don't requeue because it will happen automatically
        // when the pod status changes.
        return time.Duration(0), nil
    }
case cnatv1alpha1.PhaseDone:
    klog.Infof("instance %s: phase: DONE", key)
    return time.Duration(0), nil
default:
    klog.Infof("instance %s: NOP")
    return time.Duration(0), nil
}

// Update the At instance, setting the status to the respective phase:
_, err = c.cnatClientset.CnatV1alpha1().Ats(instance.Namespace).
    UpdateStatus(instance)
if err != nil {
    return time.Duration(0), err
}

// Don't requeue. We should be reconcile because either the pod or
// the CR changes.
return time.Duration(0), nil
```

此外，为了设置informer和控制器，我们实现NewController()方法：

```go
// NewController returns a new cnat controller
func NewController(
    kubeClientset kubernetes.Interface,
    cnatClientset clientset.Interface,
    atInformer informers.AtInformer,
    podInformer corev1informer.PodInformer) *Controller {

    // Create event broadcaster
    // Add cnat-controller types to the default Kubernetes Scheme so Events
    // can be logged for cnat-controller types.
    utilruntime.Must(cnatscheme.AddToScheme(scheme.Scheme))
    klog.V(4).Info("Creating event broadcaster")
    eventBroadcaster := record.NewBroadcaster()
    eventBroadcaster.StartLogging(klog.Infof)
    eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{
        Interface: kubeClientset.CoreV1().Events(""),
    })
    source := corev1.EventSource{Component: controllerAgentName}
    recorder := eventBroadcaster.NewRecorder(scheme.Scheme, source)

    rateLimiter := workqueue.DefaultControllerRateLimiter()
    controller := &Controller{
        kubeClientset: kubeClientset,
        cnatClientset: cnatClientset,
        atLister:      atInformer.Lister(),
        atsSynced:     atInformer.Informer().HasSynced,
        podLister:     podInformer.Lister(),
        podsSynced:    podInformer.Informer().HasSynced,
        workqueue:     workqueue.NewNamedRateLimitingQueue(rateLimiter, "Ats"),
        recorder:      recorder,
    }

    klog.Info("Setting up event handlers")
    // Set up an event handler for when At resources change
    atInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: controller.enqueueAt,
        UpdateFunc: func(old, new interface{}) {
            controller.enqueueAt(new)
        },
    })
    // Set up an event handler for when Pod resources change
    podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: controller.enqueuePod,
        UpdateFunc: func(old, new interface{}) {
            controller.enqueuePod(new)
        },
    })
    return controller
}
```

为了使它工作，我们还需要两个帮助方法：一个用来计算距离调度执行的时间间隔，如下所示：

```go
func timeUntilSchedule(schedule string) (time.Duration, error) {
    now := time.Now().UTC()
    layout := "2006-01-02T15:04:05Z"
    s, err := time.Parse(layout, schedule)
    if err != nil {
        return time.Duration(0), err
    }
    return s.Sub(now), nil
}
```

另一个使用`busybox`容器镜像创建一个带有要执行命令的pod ：

```go
func newPodForCR(cr *cnatv1alpha1.At) *corev1.Pod {
    labels := map[string]string{
        "app": cr.Name,
    }
    return &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      cr.Name + "-pod",
            Namespace: cr.Namespace,
            Labels:    labels,
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{
                {
                    Name:    "busybox",
                    Image:   "busybox",
                    Command: strings.Split(cr.Spec.Command, " "),
                },
            },
            RestartPolicy: corev1.RestartPolicyOnFailure,
        },
    }
}
```

这里您可以稍作熟悉，在本章稍后我们会在`syncHandler()`中再次使用到这两个帮助方法和业务逻辑的基本流程。

请注意，从`At`资源的角度来看，pod是辅助资源，控制器必须确保清理这些pod，否则它们将会孤立的存在。

现在，通过`sample-controller`您领略了如何开发operator，但通常您希望专注于创建业务逻辑而不是处理框架代码。因此，您可以选择另外两个项目：Kubebuilder和Operator SDK，来简化这一过程。让我们看看如何使用它们实现`cnat`。

# Kubebuilder

[Kubebuilder](http://bit.ly/2I8w9mz)，由Kubernetes特殊兴趣小组（SIG）API Machinery创建并维护，是一个工具和一套库，使您能够以简单有效的方式构建operator。深度了解Kubebuilder的最佳资源是在线[Kubebuilder文档](https://book.kubebuilder.io/)，它将向您介绍内部组件和用法。不过，我们将重点关注使用Kubebuilder 实现我们的operator[`cnat`](http://bit.ly/2RpHhON)（请参阅[我们的Git存储库中的相应目录](http://bit.ly/2Iv6pAS)）。

首先，确保已安装所有依赖项 - 即[dep](http://bit.ly/2x9Yrqq)，[kustomize](http://bit.ly/2Y3JeCV)（参见[“Kustomize”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#kustomize)）和[Kubebuilder本身](http://bit.ly/32pQmfu)：

```shell
$ dep version
dep:
 version     : v0.5.1
 build date  : 2019-03-11
 git hash    : faa6189
 go version  : go1.12
 go compiler : gc
 platform    : darwin/amd64
 features    : ImportDuringSolve=false

$ kustomize version
Version: {KustomizeVersion:v2.0.3 GitCommit:a6f65144121d1955266b0cd836ce954c04122dc8
          BuildDate:2019-03-18T22:15:21+00:00 GoOs:darwin GoArch:amd64}

$ kubebuilder version
Version: version.Version{
  KubeBuilderVersion:"1.0.8",
  KubernetesVendor:"1.13.1",
  GitCommit:"1adf50ed107f5042d7472ba5ab50d5e1d357169d",
  BuildDate:"2019-01-25T23:14:29Z", GoOs:"unknown", GoArch:"unknown"
}
```

我们将从头开始编写cnat operator。首先，创建一个目录（我们在我们*的仓库*中使用*cnat-kubebuilder*），在此目录下执行所有其他命令。

###### 警告

在撰写本文时，Kubebuilder正在发布新版本（v2）。由于它还不稳定，我们会使用[版本v1](https://book-v1.book.kubebuilder.io/)的命令和设置。

## 引导

创建cnat` operator的第一步，我们首先执行`init命令（请注意，这可能需要几分钟，具体取决于您的环境）：

```shell
$ kubebuilder init \
              --domain programming-kubernetes.info \
              --license apache2 \
              --owner "Programming Kubernetes authors"
Run `dep ensure` to fetch dependencies (Recommended) [y/n]?
y
dep ensure
Running make...
make
go generate ./pkg/... ./cmd/...
go fmt ./pkg/... ./cmd/...
go vet ./pkg/... ./cmd/...
go run vendor/sigs.k8s.io/controller-tools/cmd/controller-gen/main.go all
CRD manifests generated under 'config/crds'
RBAC manifests generated under 'config/rbac'
go test ./pkg/... ./cmd/... -coverprofile cover.out
?       github.com/mhausenblas/cnat-kubebuilder/pkg/apis        [no test files]
?       github.com/mhausenblas/cnat-kubebuilder/pkg/controller  [no test files]
?       github.com/mhausenblas/cnat-kubebuilder/pkg/webhook     [no test files]
?       github.com/mhausenblas/cnat-kubebuilder/cmd/manager     [no test files]
go build -o bin/manager github.com/mhausenblas/cnat-kubebuilder/cmd/manager
```

完成此命令后，Kubebuilder已经为operator搭建了框架，生成了一堆文件，从自定义控制器到CRD样例。您的目录结构应该类似于以下内容（为清晰起见，排除了vendor目录）：

```shell
$ tree -I vendor
.
├── Dockerfile
├── Gopkg.lock
├── Gopkg.toml
├── Makefile
├── PROJECT
├── bin
│   └── manager
├── cmd
│   └── manager
│       └── main.go
├── config
│   ├── crds
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_image_patch.yaml
│   │   └── manager_prometheus_metrics_patch.yaml
│   ├── manager
│   │   └── manager.yaml
│   └── rbac
│       ├── auth_proxy_role.yaml
│       ├── auth_proxy_role_binding.yaml
│       ├── auth_proxy_service.yaml
│       ├── rbac_role.yaml
│       └── rbac_role_binding.yaml
├── cover.out
├── hack
│   └── boilerplate.go.txt
└── pkg
    ├── apis
    │   └── apis.go
    ├── controller
    │   └── controller.go
    └── webhook
        └── webhook.go

13 directories, 22 files
```

接下来，我们创建一个API资源，使用create api命令创建一个自定义控制器（这应该比上一个命令快，但仍然需要一点时间）：

```shell
$ kubebuilder create api \
              --group cnat \
              --version v1alpha1 \
              --kind At
Create Resource under pkg/apis [y/n]?
y
Create Controller under pkg/controller [y/n]?
y
Writing scaffold for you to edit...
pkg/apis/cnat/v1alpha1/at_types.go
pkg/apis/cnat/v1alpha1/at_types_test.go
pkg/controller/at/at_controller.go
pkg/controller/at/at_controller_test.go
Running make...
go generate ./pkg/... ./cmd/...
go fmt ./pkg/... ./cmd/...
go vet ./pkg/... ./cmd/...
go run vendor/sigs.k8s.io/controller-tools/cmd/controller-gen/main.go all
CRD manifests generated under 'config/crds'
RBAC manifests generated under 'config/rbac'
go test ./pkg/... ./cmd/... -coverprofile cover.out
?       github.com/mhausenblas/cnat-kubebuilder/pkg/apis        [no test files]
?       github.com/mhausenblas/cnat-kubebuilder/pkg/apis/cnat   [no test files]
ok      github.com/mhausenblas/cnat-kubebuilder/pkg/apis/cnat/v1alpha1  9.011s
?       github.com/mhausenblas/cnat-kubebuilder/pkg/controller  [no test files]
ok      github.com/mhausenblas/cnat-kubebuilder/pkg/controller/at       8.740s
?       github.com/mhausenblas/cnat-kubebuilder/pkg/webhook     [no test files]
?       github.com/mhausenblas/cnat-kubebuilder/cmd/manager     [no test files]
go build -o bin/manager github.com/mhausenblas/cnat-kubebuilder/cmd/manager
```

让我们看看发生了什么变化，重点关注以下两个目录：

```shell
$ tree config/ pkg/
config/
├── crds
│   └── cnat_v1alpha1_at.yaml
├── default
│   ├── kustomization.yaml
│   ├── manager_auth_proxy_patch.yaml
│   ├── manager_image_patch.yaml
│   └── manager_prometheus_metrics_patch.yaml
├── manager
│   └── manager.yaml
├── rbac
│   ├── auth_proxy_role.yaml
│   ├── auth_proxy_role_binding.yaml
│   ├── auth_proxy_service.yaml
│   ├── rbac_role.yaml
│   └── rbac_role_binding.yaml
└── samples
    └── cnat_v1alpha1_at.yaml
pkg/
├── apis
│   ├── addtoscheme_cnat_v1alpha1.go
│   ├── apis.go
│   └── cnat
│       ├── group.go
│       └── v1alpha1
│           ├── at_types.go
│           ├── at_types_test.go
│           ├── doc.go
│           ├── register.go
│           ├── v1alpha1_suite_test.go
│           └── zz_generated.deepcopy.go
├── controller
│   ├── add_at.go
│   ├── at
│   │   ├── at_controller.go
│   │   ├── at_controller_suite_test.go
│   │   └── at_controller_test.go
│   └── controller.go
└── webhook
    └── webhook.go

11 directories, 27 files
```

请注意在*config/crds/*路径中生成了*cnat_v1alpha1_at.yaml*文件，这是CRD的定义文件。在config / samples /路径中生成了cnat_v1alpha1_at.yaml文件，这是一个CR样例（与CRD文件是同名的）。此外，在*pkg /中*我们看到生成了许多新文件，最重要的是*apis / cnat / v1alpha1 / at_types.go*和*controller / at / at_controller.go*，我们将在下面修改这两个文件。

接下来，我们在Kubernetes中创建一个专用的命名空间`cnat`并将其设置为默认命名空间，（推荐总是使用专用的命名空间，而不是`default`）：

```shell
$ kubectl create ns cnat && \
  kubectl config set-context $(kubectl config current-context) --namespace=cnat
```

安装CRD：

```shell
$ make install
go run vendor/sigs.k8s.io/controller-tools/cmd/controller-gen/main.go all
CRD manifests generated under 'config/crds'
RBAC manifests generated under 'config/rbac'
kubectl apply -f config/crds
customresourcedefinition.apiextensions.k8s.io/ats.cnat.programming-kubernetes.info created
```

在本机启动operator：

```shell
$ make run
go generate ./pkg/... ./cmd/...
go fmt ./pkg/... ./cmd/...
go vet ./pkg/... ./cmd/...
go run ./cmd/manager/main.go
{"level":"info","ts":1559152740.0550249,"logger":"entrypoint",
  "msg":"setting up client for manager"}
{"level":"info","ts":1559152740.057556,"logger":"entrypoint",
  "msg":"setting up manager"}
{"level":"info","ts":1559152740.1396701,"logger":"entrypoint",
  "msg":"Registering Components."}
{"level":"info","ts":1559152740.1397,"logger":"entrypoint",
  "msg":"setting up scheme"}
{"level":"info","ts":1559152740.139773,"logger":"entrypoint",
  "msg":"Setting up controller"}
{"level":"info","ts":1559152740.139831,"logger":"kubebuilder.controller",
  "msg":"Starting EventSource","controller":"at-controller",
  "source":"kind source: /, Kind="}
{"level":"info","ts":1559152740.139929,"logger":"kubebuilder.controller",
  "msg":"Starting EventSource","controller":"at-controller",
  "source":"kind source: /, Kind="}
{"level":"info","ts":1559152740.139971,"logger":"entrypoint",
  "msg":"setting up webhooks"}
{"level":"info","ts":1559152740.13998,"logger":"entrypoint",
  "msg":"Starting the Cmd."}
{"level":"info","ts":1559152740.244628,"logger":"kubebuilder.controller",
  "msg":"Starting Controller","controller":"at-controller"}
{"level":"info","ts":1559152740.344791,"logger":"kubebuilder.controller",
  "msg":"Starting workers","controller":"at-controller","worker count":1}
```

保留正在运行的终端会话，打开一个新的会话，再执行命令安装CRD，并创建示例自定义资源，如下所示：

```shell
$ kubectl apply -f config/crds/cnat_v1alpha1_at.yaml
customresourcedefinition.apiextensions.k8s.io/ats.cnat.programming-kubernetes.info
configured

$ kubectl get crds
NAME                                   CREATED AT
ats.cnat.programming-kubernetes.info   2019-05-29T17:54:51Z

$ kubectl apply -f config/samples/cnat_v1alpha1_at.yaml
at.cnat.programming-kubernetes.info/at-sample created
```

现在查看以下刚才执行`make run`的会话，会看到以下输出：

```shell
...
{"level":"info","ts":1559153311.659829,"logger":"controller",
  "msg":"Creating Deployment","namespace":"cnat","name":"at-sample-deployment"}
{"level":"info","ts":1559153311.678407,"logger":"controller",
  "msg":"Updating Deployment","namespace":"cnat","name":"at-sample-deployment"}
{"level":"info","ts":1559153311.6839428,"logger":"controller",
  "msg":"Updating Deployment","namespace":"cnat","name":"at-sample-deployment"}
{"level":"info","ts":1559153311.693443,"logger":"controller",
  "msg":"Updating Deployment","namespace":"cnat","name":"at-sample-deployment"}
{"level":"info","ts":1559153311.7023401,"logger":"controller",
  "msg":"Updating Deployment","namespace":"cnat","name":"at-sample-deployment"}
{"level":"info","ts":1559153332.986961,"logger":"controller",#
  "msg":"Updating Deployment","namespace":"cnat","name":"at-sample-deployment"}
```

这说明我们设置成功！现在我们已经完成了基础框架的搭建并成功启动了`cnat` operator，我们可以继续：使用Kubebuilder 实现业务逻辑`cnat`。

## 业务逻辑

首先，我们将[*config / crds / cnat_v1alpha1_at.yaml*](http://bit.ly/2N1jQNb)和[*config / samples / cnat_v1alpha1_at.yaml更改*](http://bit.ly/2Xs1F7c)为我们自己的`cnat`对应的CRD定义和CR样例，使用与[“sample-controller示例”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#cnat-client-go)相同的结构。

在业务逻辑方面，我们在operator中实现了两个部分：

- 在[*pkg / apis / cnat / v1alpha1 / at_types.go文件中，*](http://bit.ly/31KNLfO)我们修改`AtSpec`结构增加相应的字段，例如`schedule`和`command`。请注意，每当您在此处更改某些内容时都必须运行`make`，以便重新生成相关文件。Kubebuilder使用Kubernetes生成器（在[第5章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)描述）并发布一组它自己的生成器（例如，生成CRD 定义文件）。
- 在[*pkg / controller / at / at_controller.go中，*](http://bit.ly/2Iwormg)我们修改了`Reconcile(request reconcile.Request)`方法，按照`Spec.Schedule`中在定义的时间点去创建pod。

*at_types.go*文件：

```go
const (
    PhasePending = "PENDING"
    PhaseRunning = "RUNNING"
    PhaseDone    = "DONE"
)

// AtSpec defines the desired state of At
type AtSpec struct {
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"`
    // Command is the desired command (executed in a Bash shell) to be executed.
    Command string `json:"command,omitempty"`
}

// AtStatus defines the observed state of At
type AtStatus struct {
    // Phase represents the state of the schedule: until the command is executed
    // it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"`
}
```

在*at_controller.go*文件中，我们实现三个阶段之间的状态转变，`PENDING`到`RUNNING`到`DONE`：

```go
func (r *ReconcileAt) Reconcile(req reconcile.Request) (reconcile.Result, error) {
    reqLogger := log.WithValues("namespace", req.Namespace, "at", req.Name)
    reqLogger.Info("=== Reconciling At")
    // Fetch the At instance
    instance := &cnatv1alpha1.At{}
    err := r.Get(context.TODO(), req.NamespacedName, instance)
    if err != nil {
        if errors.IsNotFound(err) {
            // Request object not found, could have been deleted after
            // reconcile request—return and don't requeue:
            return reconcile.Result{}, nil
        }
            // Error reading the object—requeue the request:
        return reconcile.Result{}, err
    }

    // If no phase set, default to pending (the initial phase):
    if instance.Status.Phase == "" {
        instance.Status.Phase = cnatv1alpha1.PhasePending
    }

    // Now let's make the main case distinction: implementing
    // the state diagram PENDING -> RUNNING -> DONE
    switch instance.Status.Phase {
    case cnatv1alpha1.PhasePending:
        reqLogger.Info("Phase: PENDING")
        // As long as we haven't executed the command yet, we need to check if
        // it's already time to act:
        reqLogger.Info("Checking schedule", "Target", instance.Spec.Schedule)
        // Check if it's already time to execute the command with a tolerance
        // of 2 seconds:
        d, err := timeUntilSchedule(instance.Spec.Schedule)
        if err != nil {
            reqLogger.Error(err, "Schedule parsing failure")
            // Error reading the schedule. Wait until it is fixed.
            return reconcile.Result{}, err
        }
        reqLogger.Info("Schedule parsing done", "Result", "diff",
            fmt.Sprintf("%v", d))
        if d > 0 {
            // Not yet time to execute the command, wait until the scheduled time
            return reconcile.Result{RequeueAfter: d}, nil
        }
        reqLogger.Info("It's time!", "Ready to execute", instance.Spec.Command)
        instance.Status.Phase = cnatv1alpha1.PhaseRunning
    case cnatv1alpha1.PhaseRunning:
        reqLogger.Info("Phase: RUNNING")
        pod := newPodForCR(instance)
        // Set At instance as the owner and controller
        err := controllerutil.SetControllerReference(instance, pod, r.scheme)
        if err != nil {
            // requeue with error
            return reconcile.Result{}, err
        }
        found := &corev1.Pod{}
        nsName := types.NamespacedName{Name: pod.Name, Namespace: pod.Namespace}
        err = r.Get(context.TODO(), nsName, found)
        // Try to see if the pod already exists and if not
        // (which we expect) then create a one-shot pod as per spec:
        if err != nil && errors.IsNotFound(err) {
            err = r.Create(context.TODO(), pod)
            if err != nil {
            // requeue with error
                return reconcile.Result{}, err
            }
            reqLogger.Info("Pod launched", "name", pod.Name)
        } else if err != nil {
            // requeue with error
            return reconcile.Result{}, err
        } else if found.Status.Phase == corev1.PodFailed ||
                  found.Status.Phase == corev1.PodSucceeded {
            reqLogger.Info("Container terminated", "reason",
                found.Status.Reason, "message", found.Status.Message)
            instance.Status.Phase = cnatv1alpha1.PhaseDone
        } else {
            // Don't requeue because it will happen automatically when the
            // pod status changes.
            return reconcile.Result{}, nil
        }
    case cnatv1alpha1.PhaseDone:
        reqLogger.Info("Phase: DONE")
        return reconcile.Result{}, nil
    default:
        reqLogger.Info("NOP")
        return reconcile.Result{}, nil
    }

    // Update the At instance, setting the status to the respective phase:
    err = r.Status().Update(context.TODO(), instance)
    if err != nil {
        return reconcile.Result{}, err
    }

    // Don't requeue. We should be reconcile because either the pod
    // or the CR changes.
    return reconcile.Result{}, nil
}
```

请注意，最后的`Update`调用操作在*/ status*子资源上（参见[“Status subresource”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#status-subresource)）而不是整个CR。因此，在这里我们遵循spec-status分离的最佳实践。

当我们创建了CR实例 `example-at`，我们就会在本地执行的operator的会话中看到以下输出：

```shell
$ make run
...
{"level":"info","ts":1555063897.488535,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063897.488621,"logger":"controller",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063897.4886441,"logger":"controller",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T10:12:00Z"}
{"level":"info","ts":1555063897.488703,"logger":"controller",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 10:12:00 +0000 UTC with a diff of 22.511336s"}
{"level":"info","ts":1555063907.489264,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063907.489402,"logger":"controller",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063907.489428,"logger":"controller",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T10:12:00Z"}
{"level":"info","ts":1555063907.489486,"logger":"controller",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 10:12:00 +0000 UTC with a diff of 12.510551s"}
{"level":"info","ts":1555063917.490178,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063917.4902349,"logger":"controller",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063917.490247,"logger":"controller",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T10:12:00Z"}
{"level":"info","ts":1555063917.490278,"logger":"controller",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 10:12:00 +0000 UTC with a diff of 2.509743s"}
{"level":"info","ts":1555063927.492718,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063927.49283,"logger":"controller",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063927.492857,"logger":"controller",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T10:12:00Z"}
{"level":"info","ts":1555063927.492915,"logger":"controller",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 10:12:00 +0000 UTC with a diff of -7.492877s"}
{"level":"info","ts":1555063927.4929411,"logger":"controller",
  "msg":"It's time!","namespace":"cnat","at":
  "example-at","Ready to execute":"echo YAY"}
{"level":"info","ts":1555063927.626236,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063927.626303,"logger":"controller",
  "msg":"Phase: RUNNING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063928.07445,"logger":"controller",
  "msg":"Pod launched","namespace":"cnat","at":"example-at",
  "name":"example-at-pod"}
{"level":"info","ts":1555063928.199562,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063928.199645,"logger":"controller",
  "msg":"Phase: DONE","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063937.631733,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063937.631783,"logger":"controller",
  "msg":"Phase: DONE","namespace":"cnat","at":"example-at"}
...
```

想验证我们的自定义控制器是否已完成其工作，可以执行以下命令：

```shell
$ kubectl get at,pods
NAME                                                  AGE
at.cnat.programming-kubernetes.info/example-at        11m

NAME                 READY   STATUS        RESTARTS   AGE
pod/example-at-pod   0/1     Completed     0          38s
```

太棒了！`example-at-pod` 已被创建，现在我们来看看命令执行的结果：

```shell
$ kubectl logs example-at-pod
YAY
```

以上我们完成自定义控制器的开发后，并在本地执行。您可能希望将operator构建成镜像，在Kubernetes部署。可以使用以下命令生成容器镜像，并将其推送到镜像仓库，例如 *quay.io/pk/cnat*：

```shell
$ export IMG=quay.io/pk/cnat:v1

$ make docker-build

$ make docker-push
```

在了解如何使用Kubebuilder之后，让我们来看看如何使用operator SDK做同样的事情。

# Operator SDK

为了更容易构建Kubernetes应用程序，CoreOS / Red Hat将operator和代码框架整合在一起。[Operator SDK](http://bit.ly/2KtpK7D)就是其中的一部分，它使开发人员无需深入了解Kubernetes API即可开发出operators。

Operator SDK提供了构建，测试和打包operator的工具。SDK中包含了更多其他可用的功能，特别是在测试时，不过在这里我们专注于使用SDK 实现我们的运算符[`cnat`](http://bit.ly/2RpHhON)（请参阅[我们的Git存储库中的相应目录](http://bit.ly/2FpCtE9)）。

首先要做的事情是：确保[安装Operator SDK](http://bit.ly/2ZBQlCT)并检查所有依赖项是否可用：

```shell
$ dep version
dep:
 version     : v0.5.1
 build date  : 2019-03-11
 git hash    : faa6189
 go version  : go1.12
 go compiler : gc
 platform    : darwin/amd64
 features    : ImportDuringSolve=false

 $ operator-sdk --version
operator-sdk version v0.6.0
```

## 引导

现在开始按如下方式创建cnat operator：

```shell
$ operator-sdk new cnat-operator && cd cnat-operator
```

接下来，和Kubebuilder非常相似，我们添加一个API - 或简单地说：初始化自定义控制器，如下所示：

```shell
$ operator-sdk add api \
               --api-version =cnat.programming-kubernetes.info/v1alpha1 \
               --kind =At

$ operator-sdk add controller \
               --api-version =cnat.programming-kubernetes.info/v1alpha1 \
               --kind =At
```

这些命令将产生所需的框架代码以及一些辅助功能，如深拷贝方法DeepCopy()`，`DeepCopyInto()`和`DeepCopyObject()。

现在 我们可以在Kubernetes集群中创建自动生成的CRD：

```shell
$ kubectl apply -f deploy/crds/cnat_v1alpha1_at_crd.yaml

$ kubectl get crds
NAME                                             CREATED AT
ats.cnat.programming-kubernetes.info             2019-04-01T14:03:33Z
```

我们在本地启动自定义控制器`cnat`。启动后，它就可以开始处理请求：

```shell
$ OPERATOR_NAME=cnatop operator-sdk up local --namespace "cnat"
INFO[0000] Running the operator locally.
INFO[0000] Using namespace cnat.
{"level":"info","ts":1555041531.871706,"logger":"cmd",
  "msg":"Go Version: go1.12.1"}
{"level":"info","ts":1555041531.871785,"logger":"cmd",
  "msg":"Go OS/Arch: darwin/amd64"}
{"level":"info","ts":1555041531.8718028,"logger":"cmd",
  "msg":"Version of operator-sdk: v0.6.0"}
{"level":"info","ts":1555041531.8739321,"logger":"leader",
  "msg":"Trying to become the leader."}
{"level":"info","ts":1555041531.8743382,"logger":"leader",
  "msg":"Skipping leader election; not running in a cluster."}
{"level":"info","ts":1555041536.1611362,"logger":"cmd",
  "msg":"Registering Components."}
{"level":"info","ts":1555041536.1622112,"logger":"kubebuilder.controller",
  "msg":"Starting EventSource","controller":"at-controller",
  "source":"kind source: /, Kind="}
{"level":"info","ts":1555041536.162519,"logger":"kubebuilder.controller",
  "msg":"Starting EventSource","controller":"at-controller",
  "source":"kind source: /, Kind="}
{"level":"info","ts":1555041539.978822,"logger":"metrics",
  "msg":"Skipping metrics Service creation; not running in a cluster."}
{"level":"info","ts":1555041539.978875,"logger":"cmd",
  "msg":"Starting the Cmd."}
{"level":"info","ts":1555041540.179469,"logger":"kubebuilder.controller",
  "msg":"Starting Controller","controller":"at-controller"}
{"level":"info","ts":1555041540.280784,"logger":"kubebuilder.controller",
  "msg":"Starting workers","controller":"at-controller","worker count":1}
```

在我们创建CR *ats.cnat.programming-kubernetes.info* 之前自定义控制器日志信息不会再发生滚动。接下来我们创建CR实例：

```shell
$ cat deploy/crds/cnat_v1alpha1_at_cr.yaml
apiVersion: cnat.programming-kubernetes.info/v1alpha1
kind: At
metadata:
  name: example-at
spec:
  schedule: "2019-04-11T14:56:30Z"
  command: "echo YAY"

$ kubectl apply -f deploy/crds/cnat_v1alpha1_at_cr.yaml

$ kubectl get at
NAME                                             AGE
at.cnat.programming-kubernetes.info/example-at   54s
```

## 业务逻辑

在业务逻辑方面，我们在operator中实现了两个部分：

- 在[*pkg / apis / cnat / v1alpha1 / at_types.go中，*](http://bit.ly/31Ip2sF)我们修改`AtSpec`struct添加相应的字段，例如`schedule`和`command`，并使用`operator-sdk generate k8s`重新生成代码，同时使用`operator-sdk generate openapi` 命令生成OpenAPI。
- 在[*pkg / controller / at / at_controller.go中，*](http://bit.ly/2Fpo5Mi)我们修改了`Reconcile(request reconcile.Request)`方法，按照`Spec.Schedule`中在定义的时间点去创建pod。

详细地修改如下，文件*at_types.go*：

```go
// AtSpec defines the desired state of At
// +k8s:openapi-gen=true
type AtSpec struct {
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"`
    // Command is the desired command (executed in a Bash shell) to be executed.
    Command string `json:"command,omitempty"`
}

// AtStatus defines the observed state of At
// +k8s:openapi-gen=true
type AtStatus struct {
    // Phase represents the state of the schedule: until the command is executed
    // it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"`
}
```

在*at_controller.go*文件z红我们实现了三个阶段的状态转换，`PENDING`到`RUNNING`到`DONE`。

###### 注意

[`controller-runtime`](http://bit.ly/2ZFtDKd)是另一个属于SIG API Machinery的项目，这个库的目标在于以Go语言包的形式提供一套通用的低级API功能来构建控制器。有关详细信息，请参阅[第4章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)。

由于Kubebuilder和Operator SDK都使用了controller-runtime，`Reconcile()`函数实际上是相同的：

```go
func (r *ReconcileAt) Reconcile(request reconcile.Request) (reconcile.Result, error) {
    the-same-as-for-kubebuilder
}
```

当CR `example-at`被创建后，我们会看到本地operator有以下输出：

```shell
$ OPERATOR_NAME=cnatop operator-sdk up local --namespace "cnat"
INFO[0000] Running the operator locally.
INFO[0000] Using namespace cnat.
...
{"level":"info","ts":1555044934.023597,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044934.023713,"logger":"controller_at",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044934.0237482,"logger":"controller_at",
  "msg":"Checking schedule","namespace":"cnat","at":
  "example-at","Target":"2019-04-12T04:56:00Z"}
{"level":"info","ts":1555044934.02382,"logger":"controller_at",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 04:56:00 +0000 UTC with a diff of 25.976236s"}
{"level":"info","ts":1555044934.148148,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044934.148224,"logger":"controller_at",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044934.148243,"logger":"controller_at",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T04:56:00Z"}
{"level":"info","ts":1555044934.1482902,"logger":"controller_at",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 04:56:00 +0000 UTC with a diff of 25.85174s"}
{"level":"info","ts":1555044944.1504588,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044944.150568,"logger":"controller_at",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044944.150599,"logger":"controller_at",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T04:56:00Z"}
{"level":"info","ts":1555044944.150663,"logger":"controller_at",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 04:56:00 +0000 UTC with a diff of 15.84938s"}
{"level":"info","ts":1555044954.385175,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044954.3852649,"logger":"controller_at",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044954.385288,"logger":"controller_at",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T04:56:00Z"}
{"level":"info","ts":1555044954.38534,"logger":"controller_at",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 04:56:00 +0000 UTC with a diff of 5.614691s"}
{"level":"info","ts":1555044964.518383,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044964.5184839,"logger":"controller_at",
  "msg":"Phase: PENDING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044964.518566,"logger":"controller_at",
  "msg":"Checking schedule","namespace":"cnat","at":"example-at",
  "Target":"2019-04-12T04:56:00Z"}
{"level":"info","ts":1555044964.5186381,"logger":"controller_at",
  "msg":"Schedule parsing done","namespace":"cnat","at":"example-at",
  "Result":"2019-04-12 04:56:00 +0000 UTC with a diff of -4.518596s"}
{"level":"info","ts":1555044964.5186849,"logger":"controller_at",
  "msg":"It's time!","namespace":"cnat","at":"example-at",
  "Ready to execute":"echo YAY"}
{"level":"info","ts":1555044964.642559,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044964.642622,"logger":"controller_at",
  "msg":"Phase: RUNNING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044964.911037,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044964.9111192,"logger":"controller_at",
  "msg":"Phase: RUNNING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044966.038684,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044966.038771,"logger":"controller_at",
  "msg":"Phase: DONE","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044966.708663,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044966.708749,"logger":"controller_at",
  "msg":"Phase: DONE","namespace":"cnat","at":"example-at"}
...
```

在这里你可以看到operator的三个状态：直到时间戳`1555044964.518566`一直在`PENDING`，然后`RUNNING`，然后`DONE`。

要验证自定义控制器的功能并检查操作结果，请输入：

```shell
$ kubectl get at,pods
NAME                                                  AGE
at.cnat.programming-kubernetes.info/example-at        23m

NAME                 READY   STATUS        RESTARTS   AGE
pod/example-at-pod   0/1     Completed     0          46s

$ kubectl logs example-at-pod
YAY
```

以上我们完成自定义控制器的开发后，并在本地执行。您可能希望将operator构建成镜像，在Kubernetes部署。可以使用以下命令生成容器镜像：

```shell
$ operator-sdk build $REGISTRY/PROJECT/IMAGE
```

这里是有关Operator SDK及其示例的更多资源：

- Toader Sebastian在BanzaiCloud上发表的[“Kubernetes Operator SDK完整指南”](http://bit.ly/2RqkGSf)
- Rob Szumski的博客文章[“Building a Kubernetes Operator for Prometheus and Thanos”](http://bit.ly/2KvgHmu)
- 来自Cloudark on ITNEXT的[“Kubernetes Operator Development Guidelines for Improved Usability”](http://bit.ly/31P7rPC) 

为了总结本章，我们来看一些编写自定义控制器和operator的替代方法。

# 其他方法

除了我们讨论过的方法之外，您可以看看以下项目，库和工具：

- [Metacontroller](https://metacontroller.app/)

  Metacontroller的基本思想是为您提供状态和变化的声明性规范，使用JSON接口，基于级别触发的协调循环。也就是说，您将收到描述观察状态的JSON并返回描述所需状态的JSON。这对于在Python或JavaScript等动态脚本语言中快速开发自动化特别有用。除了简单的控制器，Metacontroller还允许您将API组合成更高级别的抽象 - 例如，[BlueGreenDeployment](http://bit.ly/31KNTfi)。

- [KUDO](https://kudo.dev/)

  类似对于Metacontroller，KUDO提供了一种声明性方法来构建Kubernetes operator，涵盖整个应用程序生命周期。简而言之，是Mesosphere将Apache Mesos框架经验迁移到了Kubernetes。KUDO易于使用，几乎不需要编码; 实质上，您必须指定的是Kubernetes 配置清单的集合，其中包含了何时执行等内置逻辑。

- [Rook操作工具包](http://bit.ly/2J34faw)

  这个是一个实现operator的通用库。它起源于Rook Operator，现在已被分拆成一个独立项目。

- [ericchiang / K8S](http://bit.ly/2ZHc5h0)

  这是一个由Eric Chiang精简的Go客户端，使用Kubernetesprotobuf协议。它的行为类似于官方的Kubernetes `client-go`，但只导入两个外部依赖项。虽然它有一些限制(例如，在[集群访问配置方面](http://bit.ly/2ZBQIxh))它是一个简单易用的Go包。

- [`kutil`](http://bit.ly/2Fq3ojh)

  AppsCode通过`kutil`提供Kubernetes `client-go`附加组件。

- 基于CLI客户端的方法

  一个客户端方法，主要用于实验和测试，是以编程方式使用用`kubectl`（例如，[kubecuddler](http://bit.ly/2L3CDoi)库）。

###### 注意

虽然在本书中主要介绍了使用Go编程语言编写operator，但您也可以使用其他语言编写。两个值得关注的例子是Flant创建的[Shell-operator](http://bit.ly/2ZxkZ0m)，它使您能够在shell脚本中编写operator，以及Zalando创建的[Kopf（Kubernetes运算符框架）](http://bit.ly/2WRXU6Q)，一个Python框架和库。

正如本章开头所提到的，operator领域正在迅速发展，越来越多的参与者以代码和最佳实践的形式分享他们的知识，因此请关注这里的新工具。请一定要看看网上资源和论坛，比如`#kubernetes-operators`，`#kubebuilder`和`#client-go-docs`等Kubernetes Slack 上的频道，学习新的方法、讨论问题，当遇到困难时得到帮助。

# 未来方向

目前operator开发仍然被认为将是最受欢迎和广泛使用的方式。在Kubernetes生态中，在CR和控制器方面，有几个特殊兴趣小组。最主要的是SIG [API Machinery](http://bit.ly/2RuTPEp)，目前它拥有CR和控制器，负责[Kubebuilder](http://bit.ly/2I8w9mz)项目。operator SDK已经努力在与Kubebuilder API的对齐，因此你也会看到很多重合的地方。

# 总结

在本章中，我们了解了不同的工具，可以帮助您更有效地编写自定义控制器和operator。过去，遵循`sample-controller`是唯一的选择，但是现在可以使用Kubebuilder和Operator SDK，可以让您专注于自定义控制器的业务逻辑而不需要太多的关注于框架代码。另外一点幸运的是，这两个工具共享了很多API和代码，因此从一个工具转移到另一个工具也会比较方便。

现在，让我们看看如何发布我们的劳动成果 - 即如何打包和发布我们本章编写的控制器。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#idm46336858995208-marker)我们只在这里展示相关部分; 函数本身有很多其他的样板代码，我们并不关心它们。
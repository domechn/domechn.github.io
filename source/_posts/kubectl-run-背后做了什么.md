---
title: kubectl run 背后做了什么
subtitle: whats_run_in_kubectl
date: 2019-08-29 16:18:17
tags:
- kubernetes
---

这篇文章随便写写，只写流程，不写原理

## 执行 kubectl run

### Api Server

1. 本地验证，确保不合法的请求（比如：创建不支持的资源或者格式不对等等）会快速失败，不会发给 api-server，减轻服务端压力

2. 准备向 api-server 发送 HTTP 请求，将修改后的数据进行序列化。但是 URI PATH 是什么呢？这里就依赖资源内的 apiVersion 这个值，再加上资源类型，kubectl 就能在 api 列表中找到应该发往的地址。api 列表可以通过 api-server 的 /apis 这个 URL 获取，获取之后会在本地缓存一份，提高效率。

3. api-server 肯定不会接受不合法的请求，所以 kubectl 还要在请求之前设置好认证信息。认证信息一般可以从 ~/.kube/config 中获取，它支持 4 种。

    * tls: 需要使用 x509 证书
    * token: 在 HEADER 中添加 Authorization
    * basic username password: 基本的账号密码认证
    * openid: 类似于 token，openid 由用户事先手动设置

4. 这时 api-server 已经成功接收到了请求。它会判断我们是不是有权限操作这个资源。那么怎么验证呢，在 api-server 启动的时候，可以通过参数 --authorization_mode 进行设置，这个值有 4 种。

    * webhook: 与集群外的 HTTPS 服务交互
    * ABAC: 静态文件定义的策略
    * RBAC: 动态配置的策略
    * Node: 每个 kubelet 只能访问自己节点上的资源

    如果配置了多种授权方式，只要其中一种能通过那么请求就能继续。

5. 授权通过了，但是现在仍然还不能够向 etcd 中写数据，还需要经过`准入控制链`这道关卡。准入控制链由 `Admission Controller` 控制。官方标准的准入控制链有近 10 个之多，而且支持自定义扩展。不同于授权，准入控制链一旦有一个验证失败，那么请求就会被拒绝。以下介绍三个准入控制器。

    * SecurityContextDeny: 禁止创建设置了 Security Context 的 Pod
    * ResourceQuota: 限制某个 Namespace 下总系统资源的占用量和资源的数量
    * LimitRanger: 限制某个 Namespace 下单个资源的占用量

6. 通过上面所有验证后，api-server 将 kubectl 提交的数据反序列化，然后保存到 etcd 中。

### InitializerConfiguration

1. 虽然数据已经持久化到 etcd 中了，但 apiserver 还无法完全看到或调度它，在此之前还要执行一系列 `Initializers`。`Initializers` 会在资源对外可用之前执行某些逻辑。比如 `将 Sidecar` 注入到暴露 80 端口的 Pod 中，或者加上特定的 `annotation` 等。`InitializerConfiguration` 资源对象允许你声明某些资源类型应该运行哪些Initializers。

### Contorller

1. 数据已经保存到 etcd 中了，并且初始化逻辑也完成了，下面就需要 k8s 中的 `Controller` 来完成资源的创建了。各个 Controller 会监听各自负责的资源，比如 `Deployment Controller` 就会监听 `Deployment` 资源的变化。当 api-server 将资源保存到 etcd 后，Controller 发现了资源的变化，然后就根据变化类型会调用相应回调函数。每个 Controller 都会尽力将资源当前的状态逐步转化为 etcd 中保存的状态。

2. 当所有的 Controller 正常运行后，etcd 中就会保存一个 Deployment、一个 ReplicaSet 和 三个 Pod 资源记录，并且可以通过 kube-apiserver 查看。然而，这些 Pod 资源现在还处于 Pending 状态，因为它们还没有被调度到集群中合适的 Node 上运行。这个问题最终要靠调度器（`Scheduler`）来解决。

### Scheduler

1. Scheduler 会将待调度的 Pod 按照特定的算法和调度策略绑定到集群中某个合适的 Node 上，并将绑定信息写入 etcd 中（它会过滤其 PodSpec 中 NodeName 字段为空的 Pod）。

2. Scheduler 一旦找到了合适的节点，就会创建一个 `Binding` 对象，该对象的 `Name` 和 `Uid` 与 Pod 相匹配，并且其 `ObjectReference` 字段包含所选节点的名称，然后通过 POST 请求发送给 apiserver。

3. 当 kube-apiserver 接收到此 Binding 对象时,会更新 Pod 资源中的以下字段:

    * 将 `NodeName` 的值设置为 `ObjectReference` 中的 `NodeName`。

    * 添加相关的注释。

    * 将 `PodScheduled` 的 `status` 值设置为 `True`。

### Kubelet

在 Kubernetes 集群中，每个 Node 节点上都会启动一个 Kubelet 服务进程，该进程用于处理 Scheduler 下发到本节点的任务，管理 Pod 的生命周期，包括挂载卷、容器日志记录、垃圾回收以及其他与 Pod 相关的事件。

1. Kubelet 每隔 20s，会通过 `NodeName` 向 api-server 发送查询请求来获取自身 Node 上所要运行的 Pod 清单。获取到数据后会和自身内部缓存比较来获取有差异的 Pod 列表。并开始同步这些 Pod。

    * 记录 Pod 启动相关的 `Metrics`

    * 生成一个 `PodStatus` 对象，它表示 Pod 当前阶段的状态。PodStatus 的值取决于：一、`PodSyncHandlers` 会检查 Pod 是否应该运行在 Node，如果不应该 `PodStatus` 会有 `Phase` 变成 `PodFailed`。二、接下来 PodStatus 会由 `init 容器` 和`应用容器`的状态共同来决定。

2. 生成 PodStatus 之后（Pod 中的 status 字段），Kubelet 就会将它发送到 Pod 的状态管理器，该管理器的任务是通过 apiserver 异步更新 etcd 中的记录。

3. 接下来运行一系列准入处理器来确保该 Pod 是否具有相应的权限，被准入控制器拒绝的 Pod 将一直保持 Pending 状态。

4. 如果 Kubelet 启动时指定了 `cgroups-per-qos` 参数，Kubelet 就会为该 Pod 创建 cgroup 并进行相应的资源限制。这是为了更方便地对 Pod 进行服务质量（QoS）管理。

5. 然后为 Pod 创建相应的目录，包括 Pod 的目录（`/var/run/kubelet/pods/<podID>`），该 Pod 的卷目录（`<podDir>/volumes`）和该 Pod 的插件目录（`<podDir>/plugins`）。

6. 卷管理器会挂载 `Spec.Volumes` 中定义的相关数据卷，然后等待是否挂载成功。根据挂载卷类型的不同，某些 Pod 可能需要等待更长的时间（比如 NFS 卷）。

7. 从 apiserver 中检索 `Spec.ImagePullSecrets` 中定义的所有 Secret，然后将其注入到容器中。

### CRI

1. 上步之后大量的初始化工作都已经完成，容器已经准备好开始启动了。Kubelet 通过容器运行时接口与容器运行时（默认是 `Docker`）交互。第一次启动 Pod 时，Kubelet 会创建 `sandbox`，sandbox 作为 Pod 中所有的容器的基础容器，为 Pod 中的每个业务容器提供了大量的 Pod 级别资源，这些资源都是 Linux 命名空间（包括网络命名空间，IPC 命名空间和 PID 命名空间）。

### CNI

1. 接着 Kubelet 会为 Pod 创建网络环境，来保证跨主机之间 `Pod 和 Pod`，`Pod 和 Service` 的通信。当 Kubelet 为 Pod 创建网络时，它会将创建网络的任务交给 `CNI` 插件。CNI 表示容器网络接口（Container Network Interface），和容器运行时的运行方式类似，它也是一种抽象，允许不同的网络提供商为容器提供不同的网络实现。不同的 CNI 插件运行原理会有不同，可以参考对应的文章。

### 启动容器

所有网络都配置完成后，接下来就开始真正启动业务容器了！

1. 一旦 `sanbox` 完成初始化并处于 active 状态，Kubelet 就可以开始为其创建容器了。首先启动 `PodSpec` 中定义的 init 容器，然后再启动业务容器。

2. 首先拉取容器的镜像。如果是私有仓库的镜像，就会利用 `PodSpec` 中指定的 `Secret` 来拉取该镜像。

3. 然后通过 `CRI` 接口创建容器。`Kubelet` 向 `PodSpec` 中填充了一个 `ContainerConfig` 数据结构（在其中定义了命令，镜像，标签，挂载卷，设备，环境变量等待），然后通过 protobufs 发送给 CRI 接口。对于 Docker 来说，它会将这些信息反序列化并填充到自己的配置信息中，然后再发送给 Dockerd 守护进程。在这个过程中，它会将一些元数据标签（例如容器类型，日志路径，dandbox ID 等待）添加到容器中。

4. 接下来会使用 `CPU` 管理器来约束容器，这是 Kubelet 1.8 中新添加的 alpha 特性，它使用 `UpdateContainerResources CRI` 方法将容器分配给本节点上的 CPU 资源池。

最后容器开始真正启动。

如果 `Pod` 中配置了容器生命周期钩子（`Hook`），容器启动之后就会运行这些 Hook。Hook 的类型包括两种：Exec（执行一段命令） 和 HTTP（发送HTTP请求）。如果 PostStart Hook 启动的时间过长、挂起或者失败，容器将永远不会变成 `running` 状态。

## 总结

上文所述的创建 Pod 整个过程的流程图如下所示：

![流程](/images/kubectl_run/short.svg)

参考：

* <https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw>

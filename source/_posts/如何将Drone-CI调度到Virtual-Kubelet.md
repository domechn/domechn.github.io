---
title: 如何将Drone CI调度到Virtual Kubelet
subtitle: drone_virtual_kublet
date: 2019-11-18 23:53:30
tags:
- devops
- kubernetes
- drone
- ci/cd
---

## 什么是 virtual kubelet

以下是来自 [Virtual Kubelet](https://github.com/virtual-kubelet/virtual-kubelet#virtual-kubelet) 项目的文档的中文翻译。

> Virtual Kubelet 是开源的 Kubernetes kubelet 的实现，它可以伪装成 Kubelet 将 Kubernetes 连接到其他 API。这样就允许 Node 的背后由其他服务支撑，例如：ACI, AWS Fargate, IoT Edge。Virtual Kubelet 的主要作用是扩展无服务器（Serverless）平台，让它能够与 Kubernetes 通信。

以 `阿里云ECI` 为例（以下均以 ECI 代指 Virtual Kubelet），ECI 是阿里云的弹性容器实例。可以将每台 ECI 实例看作是一个 `Container`，所以它的创建和销毁是很廉价的。同时它拥有启动快（秒级）、成本低（按运行的秒数收费）、弹性能力强等特点。通过 Virtual Kubelet 提供的 [Kubernetes API](https://github.com/virtual-kubelet/virtual-kubelet#current-features)，ECI 就能和 K8S 交互，我们就可以在 ECI 上执行创建 Pod 或者删除 Pod 等操作。

## 使用 virtual kubelet 执行 drone 任务的好处

在使用 virtual kubelet 之前，为了不影响业务的稳定性。我们的 K8S 集群中开了几台固定的 ECS 实例专门给 CI 使用（给这些 Node 打上了污点，所以业务的服务不会调度到上面）。

这种做法虽然在不影响业务的情况下也保证了 CI 的稳定运行，但是它会造成一定程度的浪费。因为 CI 本身不像大多数业务服务，需要一天 24 小时的运行。CI 的场景是白天需要很多资源，但是到了晚上几乎不消耗任何资源。所以可以说这些机器有接近 1/3 的时间是在浪费 💰 的。

虽说 K8S 本身也提供了动态扩容机制，可以设置很少的固定资源再通过 CA 动态扩容集群来减少资源消耗。但是 CA 的启动速度（分钟级）满足不了 CI 这种时效要求高的场景。

## 如何让 Drone 兼容 ECI

### HostPath

关于 Drone In K8S 的运行模式，可以翻看我之前写的文章 [Drone 在 K8S 中执行一次构建都经历了什么](https://domc.me/2019/10/22/ck34o2dab0007u79ki5yxke0l/)。

简单来说，`Drone Server` 接收到构建任务后，会在其运行的 `Namespace`（假设为 CICD）下创建一个 `Job`，该 Job 会创建一个随机名称的 Namespace，再在创建出来的 Namespace 按配置文件中的顺序执行每个 Step，每个 `Step` 就是一个 `Pod`。这些 Pods 之间通过 `HostPath` 类型的 `Volume` 来交换文件。

![执行过程](/images/drone_virtual_kubelet/origin.jpg)

那么问题来了，ECI 是不支持 HostPath Volume 的。它只支持 EmptyDir、NFS 和 ConfigFile（也就是 ConfigMap 和 Secret）。

所以要如何解决之前提的 Pods 之间使用 HostPath 交换文件的问题呢？首先想到了是通过 `Mutating Webhook` 将 HostPath 替换成 NFS，这样每个 Pod 之间使用 NFS 共享文件，这样带来了 NFS 文件的清理问题，不像之前 `Pipeline` 执行完之后可以直接使用 `os.Remove(path)` 来清理文件，使用 NFS 后需要实现额外 `Cornjob` 来清理 NFS 上的琐碎的文件。这样便增添了服务之间的关系复杂度。

好在在浏览 Drone 社区相关信息后发现了 Drone 发布了 1.6 版本。在 1.6 版本之后，Drone 为 K8S 实现了单独的 [Runner](https://github.com/drone-runners/drone-runner-kube)。

与之前执行 Job 的方式不同，新的执行方式是 Drone Server 接收到构建信息后会将构建信息存入基于内存的 `Queue` 中，runner 会向 Server 拉取构建信息，然后将构建信息解析成**一个** `Pod`，每个 `Step` 是一个 `Container`。为了保证 Step 执行的顺序行（因为 Pod 创建的时候 Container 执行是无序的），Kube-Runner 将每个还未轮到执行的 Step 的 `Image` 设置成了 `placeholder`，placeholder 是一个占位 image，该 image 不停地 sleep 不作任何操作。等到前置 Step 执行完成了，Kube-Runner 会将下个要执行的 Step 的 Image 由 placeholder 改成其对应的 image。通过上述操作来完成执行的顺序性。

![执行过程](/images/drone_virtual_kubelet/latest.jpg)

因为所有的 Step 都在一个 Pod 内，所以它们的数据就可以使用 EmptyDir 来共享。这便解决了之前的 HostPath 兼容性的问题。

### Privileged Context

在执行 CI 时，重要一步就是构建镜像。以 docker 为例。使用 docker 构建镜像就需要用到 `Docker Deamon`，Docker Deamon 可以使用宿主机上的或者可以在 Container 内启动一个 Docker Deamon，这里就形成了两种不同的模式，也就是 `Docker Outside Docker` 和 `Docker In Docker`。

因为 Docker Outside Docker 需要挂载宿主机的文件，所以自然在这种情况下是无法使用的。而 Docker In Docker 因为需要在容器内启动 Docker Deamon，所以需要 Privileged 权限，遗憾的是目前 ECI 中并不支持 Container 使用 Privileged Context。所有这两种方法在当前情况下都无法有效地构建镜像。

那么如何解决这种问题？

1. 通过一个服务将某台主机的 docker.sock 通过 TCP 的方式暴露出来，再通过 TCP 的方式访问 Docker Deamon。
2. 使用 [Kaniko](https://github.com/GoogleContainerTools/kaniko) 来构建镜像。

这里，我们选用了 kaniko 作为构建工具。

### Knaiko

![Knaiko](https://raw.githubusercontent.com/GoogleContainerTools/kaniko/master/logo/Kaniko-Logo.png)

> Knaiko 是从容器或 Kubernetes 集群内部的 Dockerfile 构建容器映像的工具。不依赖 Docker 守护程序，而是完全在用户空间中执行 Dockerfile 中的每个命令。这样就可以在无法轻松或安全地运行 Docker 守护程序的环境（例如标准Kubernetes集群）中构建容器映像。

Kaniko 执行器首先根据 Dockerfile 中的 `FROM` 一行命令解析基础镜像，按照 Dockerfile 中的顺序来执行每一行命令，在每执行完一条命令之后，会在用户目录空间中产生一个文件系统的快照，并与存储于内存中的上一个状态进行对比，若有改变，则将其认为是对基础镜像进行的修改，并以新层级的形式对文件系统进行增加扩充，并将修改写入镜像的元数据中。在执行完 Dockerfile 中的每一条指令之后， Kaniko 执行器将最终的镜像文件推送到指定的镜像仓库。

Kaniko 可以在不具有 `ROOT` 权限的环境下，完全在用户空间中执行解压文件系统，执行构建命令以及生成文件系统的快照等一系列操作，以上构建的过程完全没有引入 docker 守护进程以及CLI的操作。

到这，构建镜像问题也解决了。接下来就可以将整个构建调度到 ECI 了。

## 实际过程中遇到的问题

虽说上面已经为 Drone 兼容 ECI 的目标做了很多事情，但也只是理论上的。实际在操作中仍然遇到了很多问题。

因为 Drone 在执行一次构建时需要不断的 Update Pod，而目前看来似乎 ECI 的 Update 机制做的不是很完善，在 Update 过程中出现了很多问题。

1. 在 Update 时，所有 Pod 的 `Environment` 都丢失。

    ```bash
    > k exec -it -n beta nginx env
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    NGINX_VERSION=1.17.5
    NJS_VERSION=0.3.6
    PKG_RELEASE=1~buster
    KUBERNETES_PORT_443_TCP_PROTO=tcp
    KUBERNETES_PORT_443_TCP_ADDR=172.21.0.1
    KUBERNETES_PORT_443_TCP_PORT=443
    KUBERNETES_PORT_443_TCP=tcp://172.21.0.1:443
    KUBERNETES_SERVICE_HOST=172.21.0.1
    TESTHELLO=test
    KUBERNETES_SERVICE_PORT_HTTPS=443
    KUBERNETES_SERVICE_PORT=443
    KUBERNETES_PORT=tcp://172.21.0.1:443
    TERM=xterm
    HOME=/root

    > k edit pod -n beta nginx
    pod/nginx edited

    > k exec -it -n beta nginx env
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    TERM=xterm
    HOME=/root
    ```

2. Pod 中每个 Container 的 Environment 数量不能超过 92 个。因为 Drone 将每个 Step 执行的关键信息都保存在 Env 中，导致每个 Container 需要包含大量 Env，因此在调度 Pod 到 ECI 上时出现了 `ExceedParamError`。

3. Pod Update 时 EmptyDir 中的文件会丢失。Drone 通过 EmptyDir 来共享每步获得或者修改的文件，EmptyDir 丢失导致 CI 失败。

4. Pod Update 某个 `Image` 时，K8S 内显示 Pod 更新成功，但 ECI 的接口返回了失败。

    ```bash
    Normal   SuccessfulMountVolume  2m5s                 kubelet, eci                    MountVolume.SetUp succeeded for volume "drone-dggd12eq2mlm5zeevff9"
    Normal   SuccessfulMountVolume  2m5s                 kubelet, eci                    MountVolume.SetUp succeeded for volume "drone-sw7d8bri9rpn0tjr90bp"
    Normal   SuccessfulMountVolume  2m5s                 kubelet, eci                    MountVolume.SetUp succeeded for volume "default-token-l6jsc"
    Normal   Started                2m4s (x4 over 2m4s)  kubelet, eci                    Started container
    Normal   Pulled                 2m4s                 kubelet, eci                    Container image "registry-vpc.cn-hangzhou.aliyuncs.com/drone-git:latest" already present on machine
    Normal   Created                2m4s (x4 over 2m4s)  kubelet, eci                    Created container
    Normal   Pulled                 2m4s (x3 over 2m4s)  kubelet, eci                    Container image "drone/placeholder:1" already present on machine
    Warning  ProviderInvokeFailed   104s                 virtual-kubelet/pod-controller  SDK.ServerError
    ErrorCode: UnknownError
    Recommend:
    RequestId: 94D6A5E0-9F90-47EF-99E9-64DFAE37XXXX
    ```

5. ECI 中 Pod Update 的策略和 K8S 中的不一致，K8S 中 Update Pod 时，只会 Kill 掉更改过的 Container，并进行替换。而在 ECI 上，会 Kill 掉所有正在运行的 Container，在进行替换。

    ```bash
    Type    Reason                 Age                 From          Message
    ----    ------                 ----                ----          -------
    Normal  Pulling                111s                kubelet, eci  pulling image "nginx"
    Normal  Pulled                 102s                kubelet, eci  Successfully pulled image "nginx"
    Normal  Pulling                101s                kubelet, eci  pulling image "redis"
    Normal  Pulled                 97s                 kubelet, eci  Successfully pulled image "redis"
    Normal  SuccessfulMountVolume  69s (x2 over 112s)  kubelet, eci  MountVolume.SetUp succeeded for volume "test-volume"
    Normal  Killing                69s                 kubelet, eci  Killing container with id containerd://image-2:Need to kill Pod
    Normal  Killing                69s                 kubelet, eci  Killing container with id containerd://image-3:Need to kill Pod
    Normal  Pulled                 69s (x2 over 111s)  kubelet, eci  Container image "busybox" already present on machine
    Normal  SuccessfulMountVolume  69s (x2 over 112s)  kubelet, eci  MountVolume.SetUp succeeded for volume "default-token-l6jsc"
    Normal  Pulling                68s                 kubelet, eci  pulling image "mongo"
    Normal  Started                46s (x5 over 111s)  kubelet, eci  Started container
    Normal  Created                46s (x5 over 111s)  kubelet, eci  Created container
    Normal  Pulled                 46s                 kubelet, eci  Successfully pulled image "mongo"
    ```

    感觉它的更新方式就是把 Pod 删了重新创建，这里只更新了 image-2 ，但是它把 image-3 也 Kill 掉了，还重新 Pull 了 image-1 的镜像。

尽管上述问题在我们向阿里云反馈之后，一部分得到修复。但是 Drone 没执行一步都需要 Update Pod 的操作，对 ECI 来说都需要很大的代价。

所以我想是否能改变 Kube Runner 的执行方式来优化整个 CI 执行的流程。

> ps: 还不是要改代码。。。

在 [drone-runner-eci](https://github.com/domgoer/drone-runner-eci) 中，我通过在 CI 启动时就确定好所有的镜像，然后在执行每个 `Step` 时，`Attach` 到容器中执行 `Commands` 的方式来避免 `Pod Update` 的巨大开销。

但是这样做就要求每个 `Image` 都必须有 `shell`，好在我们的配置是中心化管理的，所以改起来还算方便。

实际执行时，仍然遇到一个尴尬的问题，就是 Pod 的 Spec 过大（每个 Step 大概都有 100 多个 Env，大概有 10 多个 Step），在 virtual kubelet 向阿里云 ECI 提交创建请求时被阿里云的网关拒绝，返回了 414 。。。。

于是只能接着优化。因为 Pod 中大多数 Env 都是相同的，所以我将他们合并在一起放到了 Pod 的 Annotations 中，再通过 DownwardAPI 将 Pod Annotations 映射到文件，再在执行每步 Step 之前将它们 export 成环境变量。

> 我太难了

于是 Pod 的 Spec 瞬间少了许多，也就解决了上面的 414 问题。

目前 [drone-runner-eci](https://github.com/domgoer/drone-runner-eci) 已在我们生产环境稳定运行，想要体验弹性 CI 的朋友也可以尝试尝试。

## 持续优化

解决了上面提到的问题，Drone 也算能在 ECI 上运行了。但仍然有许多优化的空间。

### 镜像拉取

不同与在宿主机上创建 Pod，每次创建时可以使用宿主机上的镜像缓存。每次在 ECI 上创建 Pod 都需要拉取镜像，这便加慢了构建的速度，为了解决这一现象，阿里云也提供了方案 [imagecache](https://help.aliyun.com/document_detail/141241.html?spm=a2c4g.11174283.6.614.36ed4b5bSDZ6jm)。

> imagecache 运行用户事先将需要用到的镜像作为云盘快照缓存，在创建ECI容器组实例时基于快照创建，避免或减少镜像层下载，从而提升ECI容器组实例创建速度。

### 资源优化

ECI 给 Drone 提供了一个巨大的便利，就是在限制运行资源时不需要为每个 Container 设置 resources。

比如有两个 Step（Container），每个都需要 1vCPU 和 1Gi Mem，那么调度这个 Pod 就需要 node 上有超过 2vCPU 和 2Gi Mem 的资源，但又因为大多数时候 CI 构建任务是串行的，这两个 Step 不需要同时执行，也就是说在这种场景下其实只需要 1vCPU 和 1Gi Mem 就足够了，上面这种情况就造成了资源的浪费，但在宿主机上并没有办法很好的解决这个问题。不过在 ECI 中每个 Pod 会独占一台 ECI 实例，所以所有 Pod 内的所有 Container 都可以享受 ECI 的全额配置。

这种资源分配方式和资源大小可以通过 Pod `Annotations` 来指定：

```yaml
annotations:
    k8s.aliyun.com/eci-cpu: 1
    k8s.aliyun.com/eci-memory: 1Gi
```

这样 Pod 就能在一个 1vCPU 和 1Gi Mem 的 ECI 实例中稳定运行完两个 Step。

## 总结

总的来说，`Virtual Kubelet` 可能是目前最好的弹性 CI 解决方案之一。(ps: 在 ECI 的产品文档中也包含了 Jenkins 和 Gitlab CI 的[最佳实践](https://help.aliyun.com/document_detail/98298.html?spm=a2c4g.11174283.6.601.4c6a4b5bNIYtpx))

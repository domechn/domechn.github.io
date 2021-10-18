---
title: 'ExitCode: 128 之无任何错误信息'
subtitle: exit_code_128
date: 2019-11-21 02:04:24
tags:
- issues
- ci/cd
- drone
- devops
- kubernetes
---

今天在使用 `Drone CI` 构建项目时出现一个异常奇怪的错误。

![error](/images/exitcode-128/drone-error.png)

在 `clone` 某分支代码时构建直接失败退出了。

首先我排查了 `rpc error`，发现这个错误出现的原因是正在获取 `Pod` 的 `Log` 的过程中，Pod 被删除了。

当时我就觉得很奇怪，就算因为 `Step` 执行失败要删除 Pod，那也会先停止 `getLog`，而不是在 getLog 的过程中直接删除 Pod。

于是使用了 `kubectl describe` 了这个 Pod，发现该 Container 的 `Exit Code` 是 128。说明是该 Container 执行时出错，自己退出的。

但是从上图的日志中来看 Container 应该是执行成功了，并没有出现错误的日志。那为什么该 Container 的退出码是 128 而非 0 呢。

于是我开始怀疑是不是 `git checkout -b master` 时出现了错误，于是我在 Docker 容器中模拟了该 Step 的步骤，`git checkout -b master | echo $?`，发现即使是显示 `Already on master` 它的退出码也是 0。于是排除了是 git 出错的可能性。

那么问题出在哪呢？

经过一段时间排查，我发现只有当 `Image` 为 `registry.cn-hangzhou.aliyuncs.com/xxx/xxxx:xxx` 是才会出现这种不正常情况。

原来是因为我们集群里还运行着另一个服务，该服务会在 **Pod 被创建时**判断 Image 的类型，将这种使用了公网地址的 Image，替换成使用内网地址 `registry-vpc.cn-hangzhou.aliyuncs.com/xxx/xxxx:xxx`。再加上我刚好对 `drone-runner-kube` 做过一些[优化](https://github.com/domgoer/drone-runner-kube/commit/d2b8fd19b69206ece37e00754b3d1b92fb58ad11)（提前设置初始镜像，减少 pod update 次数）。导致了这两个服务冲突。

![steps](/images/exitcode-128/steps.jpg)

在 runner 创建 pod 时，先将 image 设为了公网地址（Step 1），接着 image 被另一个服务设置成了内网地址，并提交到 k8s 中运行了。然后 runner 通过对比发现 pod 的 image 不是自己刚开始设置的 image，于是就 update 了这个 pod（Step 4）。

因为 Update Image 的缘故导致了该 Step 没有按期望运行。所以出现了这个默默奇妙的错误。

解决方法：将 Image 的初始地址设置成内网地址，这样另一个服务就不会去修改该 Pod 的 Image，也就不会出现该错误。

---
title: ArgoCD Monorepo 性能优化之路（二）
subtitle: argocd_mono_repo_performance_optimization_second
date: 2024-09-02 12:50:24
tags:
  - kubernetes
  - argocd
  - gitops
  - monorepo
---

> 本文中 ArgoCD 为 2.8.x 版本。

在 [argocd-monorepo性能优化之路](/2024/08/03/argocd_mono_repo_performance_optimization/) 中，我们介绍了 ArgoCD Monorepo 性能优化的第一部分，主要是从优化 Git Repo 以及关闭 Application Auto Sync 改成通过 Github Action Workflow 触发应用 sync，从而降低 ArgoCD 负载。

虽然经过上面的优化后，argoCD 能在日常使用时，99% 的情况下不出现性能问题，然而还是存在一些极端场景下的性能问题，导致 sync 时耗时过长（比如正常情况下 sync app 耗时 25s (p90)，但是有时候会耗时 3min(p99)）。

本篇将继续介绍其他优化策略，优化 ArgoCD sync 的速度和稳定性和降低 sync 时使用的资源，从而增强 ArgoCD 并发 sync 多个 Application 的能力。

以下是导致 argoCD sync 突然很慢的原因以及优化的策略：

## 原因

1. Sync App 时，argoCD application-controller 中 k8s cluster 缓存已经过期，argoCD 需要先刷新 cluster 缓存，然后再 sync app，如果 cluster 中 resources 数量过多，会导致 cluster 缓存刷新时间过长，从而导致 sync app 耗时过长。
2. 如果 Application 使用了 sidecar plugin，那么在 sync app 时，argoCD repo-server 会把整个 repo 压缩成 tarball，然后传输给 sidecar plugin，如果 repo 过大，会导致压缩和解压缩耗时过长，以及在这过程中会占用大量的 CPU 和内存资源，导致 argoCD repo-server 没有足够的资源同时处理多个 sync 请求。
3. 如果 repo server 长时间没有接收到请求，它内部的 git 代码可能落后太久，导致接受到 sync 请求时，会花费较长的时间 git fetch 代码（这取决于 repo 更新的频繁程度，如果是 monorepo 可能一天会有大量的 git commit），增加 argoCD sync 的耗时。

## 优化策略

### 1. 优化 argoCD application-controller 中 cluster 的缓存策略

argoCD application-controller 提供了环境变量 `ARGOCD_CLUSTER_CACHE_RESYNC_DURATION` 来控制 cluster 缓存的过期时间，默认是 12h 将所有 cluster 的缓存都过期，然后直到接受到新的 sync 或者 refresh 请求，再刷新这个 application 所在的 cluster 的缓存（会拉取 k8s cluster 中所有的资源，以及集群的 metadata 等）。同时这个过程是同步的，也就是说在 cluster 缓存刷新过程中，所有接受到的 app sync 请求，都会一直等，直到 cluster 缓存刷新完成。

> 如果集群中资源过多，比如集群中有 200k 个资源，那么刷新 cluster 缓存可能会耗时 3～5min，

如果集群中资源过多，可以适当调大 `ARGOCD_CLUSTER_CACHE_RESYNC_DURATION` 的值，减少 cluster 缓存刷新的频率（或者将它设置成 0，不过期 cluster 缓存），从而减少 sync app 时等待 cluster 缓存刷新的情况。

### 2. 优化 argoCD repo-server 中 sidecar plugin 的压缩策略

这其实是影响 argoCD 性能的主要原因。如果 repo-server 数量不多，导致多个 sync 请求打到同一个 repo-server pod 上，pod 的 CPU 很快就会被占满（这取决于 git repo 大小，如果 repo 本身并不大，这个问题其实也并不明显。如之前的文章中提到的，目前我们 repo 大小为 150Mi，每次 tarball 需要 3s，在处理单个请求时，cpu 都会被占满（4vCPU））。

当然，这可以通过增加 repo-server 数量来缓解，但是仍然有可能出现负载不均衡的情况。而且增加 repo-server 数量，也无发解决业务侧批量 sync 整个集群所有服务的情况。sync 的 app 数量过多时，所有 repo-server pod 必然会被占满，导致后续的 sync 请求被阻塞。

然而不幸的是，argoCD 并没有提供优化 sidecar plugin 的压缩策略的环境变量，所以我们只能通过修改 argoCD 源码来实现。

早期在 argoCD 中。提供了 `argocd.argoproj.io/manifest-generate-paths` 配置，可以设置在 argoCD Application 的 Annotation 中。这原本是用来处理 webhook 请求，如果 argoCD 接入了 github webhook，那么在用户 push code 后，github 会发送 push event 到 argoCD server，argoCD 再判断变动的文件是否在 `argocd.argoproj.io/manifest-generate-paths` 路径中，如果是，argoCD 会自动触发 sync，否则就忽略。

这个值往往就对应了渲染这个 Application 中所有 l8s resources 时所需要用到的文件。

所以我们可以复用这个配置，修改 argoCD 代码，让它一旦发现 Application 中有 `argocd.argoproj.io/manifest-generate-paths` 配置，就不再压缩整个 repo，而是只压缩并传输 `argocd.argoproj.io/manifest-generate-paths` 配置中的文件。

在优化完成后，我们的 argoCD repo-server 的 CPU 使用率明显下降 95%，tarball 耗时从 3s -> 400ms，同时 argoCD sync 的速度也有了明显的提升（20s -> 3s）。

### 3. 优化 argoCD repo-server 中 git fetch 的策略

通过观察 argoCD 的 grafana dashboard，我们发现 git fetch 的 metrics 偶尔会出现异常，拉取代码会超过 20s（repo 变动频发，且有 450k+ 的 commit），而正常情况下耗时都在 1s 以内。

因此我们猜想可能是因为请求被打到的 repo-server 长时间没有接收到请求，导致 git 代码落后太久，导致接受到 sync 请求时，会花费较长的时间 git fetch 代码。

所以我们对 argoCD repo-server 代码也做了一些优化。

当 repo-server 中的 git repo 长时间（超过 5min）没有被更新时，主动触发 git fetch，从而保证 repo-server 中的代码是最新的，减少之后 git fetch 的耗时。

### 优化 argoCD repo-server 中 git gc 的策略

因为 monorepo 中变动频发，，可能导致 repo-server 中的 git 目录里有很多松散对象或者包文件，这可能会触发 git 的 auto gc，我们不希望在正常的使用过程中出现 git gc，导致 repo-server 不可用。因此我们重新 build 了 argoCD repo-server 镜像，在其中添加了 `.gitconfig` 文件，禁用了 git gc。

然后再修改 argoCD 代码，增加了一个 grpc function，通过调用这个 function 主动触发 git gc。在 git gc 时，repo server 会将自己的 readinessProbe 主动失败，等这个 pod 被移除出 k8s service 后，再执行 git gc，gc 完后再加入 k8s service。

最后再通过 k8s cronjob 每天定时执行 git gc 任务。执行时，会获取所有 repo-server pod，然后通过 grpc 挨个请求 pod ip 触发 git gc。

## 总结

通过上面的优化策略，我们进一步优化了 argoCD 处理大量并发 sync app 的性能，并提高了 sync app 的速度，使其能够满足我们的性能和稳定性要求。

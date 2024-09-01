---
title: ArgoCD Monorepo 性能优化之路
subtitle: argocd_mono_repo_performance_optimization
date: 2024-08-03 23:47:13
tags:
  - kubernetes
  - argocd
  - gitops
  - monorepo
---

> 本文中 ArgoCD 为 2.8.x 版本。

在这篇博客中，我将分享我们在使用 ArgoCD 和 Monorepo 的过程中遇到的性能问题以及我们是如何解决这些问题的，最终实现在 ArgoCD 中使用一个 Monorepo 稳定部署超过 100k+ 应用的。

## 为什么使用 Monorepo

### 通过目录结构划分环境和集群，方便权限控制

通过 Prow 和 OWNERS 文件，只有特定的人或 team 能 `/lgtm` 合并特定 `业务/环境/集群/应用` 的变更 PR。

### 方便批量变更

因为所有部署文件都在一个仓库中，当有新的 feature 需要 enable 时，可以通过脚本批量更新所有部署文件。

### 方便新环境，新集群上线

有新环境或新集群需要上线时，批量拷贝文件夹并替换全局变量即可同步所有 infra 组件/应用 到新集群。

### 当前用法

因业务需要，我们会在每个业务的账号中部署一套 ArgoCD，因此有多个 ArgoCD 集群。不过 98% 以上的应用使用同一个 Monorepo。

以下是常用的几个 ArgoCD 集群以及用量。

![argo-cd-1](/images/argocd_monorepo/argo-cd-1.png)
![argo-cd-2](/images/argocd_monorepo/argo-cd-2.png)
![argo-cd-3](/images/argocd_monorepo/argo-cd-3.png)

#### 然后通过 Apps of App 模式部署应用

在 ArgoCD 中用一个 Global Application 去创建各个控制部署资源的 Application，再由这些 Application 去创建部署资源（Deployment，Service 等）。

#### 目前我们只使用了 Application

虽然我们也需要将一份配置同时部署到多个集群，但我们没有使用 ApplicationSet。如果是镜像集群，我们会修改镜像集群中 Application 的 source，直接指向主集群的 gitops 目录。

如果是灾备集群，它会有单独的配置，在上游的发布系统中，发布某个服务的主集群时，会一同修改这个集群里的配置。

---

但是众所周知，ArgoCD 对 Monorepo 的支持非常差。我们做了很多努力来优化它的性能，使其能够满足我们的性能和稳定性要求。

## 优化策略

我们通过以下几个方面来优化 ArgoCD 的性能。

### Monorepo 层面

- **减少 Monorepo 体积**：只存放部署文件。
- **定时清理 Git 的 commit 记录**：每次发版会创建一个 commit，当 commit 数量太多时，会严重增大 repo 的大小，导致 repo server 拉取速度变慢。我们会在 repo commits 数量超过 1M 时，备份 repo，然后清空 commit 记录。

### ArgoCD 层面

- **增加 repo server 节点，关闭 repo server 的 HPA**：

  每次 ArgoCD `refresh` 或 `sync` 应用时都会请求 repo server 获取 `helm/kustomize` 渲染后的 K8s YAML 文件。由于每个 repo server pod 对同一个 repo 一次只能处理一个请求，而我们使用的是 mono repo（只有一个 repo），因此 repo server 的数量限制了 ArgoCD 的 sync 并发数。增加 repo server 的数量可以明显提高发版高峰期的 sync 效率。

  但是，repo server 每次启动时会全量 clone repo（以我们现在的 mono repo 为例，有 40 万个 commits，虽然代码只占 150M，但仓库总大小超过 4Gi，全量克隆一次需要花费 3-4 min）。如果触发 HPA 导致 repo server 频繁扩缩容，反而会影响 sync 性能。因此，我们通常会为 repo server 配置足够的 replicas 以应对日常发版需求。当需要批量变更应用时，再手动扩容 repo server。

- **关闭 auto refresh，将 appResyncPeriod 设置成 0**：

  我们关闭了所有 Applications 的 auto sync，以避免 ArgoCD 大量主动 refresh + sync Application，导致各个组件压力过大，无法处理正常用户的同步请求。

  将 [appResyncPeriod](https://argo-cd.readthedocs.io/en/stable/faq/#how-often-does-argo-cd-check-for-changes-to-my-git-or-helm-repository) 设置为 0 意味着 application controller 不会主动请求 refresh Application，从而减轻各个组件的压力，这对性能优化也非常重要。

#### 关闭 auto sync 后，如何在 commit 之后触发 Application 的 sync 呢 ？

我们使用 `GitHub Action Workflow` 来监听文件变动，然后找到对应的 Application 并触发相应的 sync.

这得益于目录结构的规划，使我们能很方便的解析出变动文件对应的 Application。

目录结构如下:

```markdown
├── app|infra # 分别对于业务应用和 infra 组件
│ ├── $project # 所属的项目
│ │ ├── $env # 所属的环境
│ │ | |-- $cluster # 所在的集群
│ │ | | |-- kustomize|helm # 使用的 argocd plugin
│ │ | | | |-- $app # app 名称
│ │ | | | | |-- values.yaml # 具体的配置文件
│ │ | | | | |-- application.yaml # 用于生成这个 app 的 argo Applications
```

**Github Action Workflow 的执行步骤**:

1. 解析 commit 中变动的文件，找出变动的 应用类型 (app or infra), project, env, cluster，根据这些信息找到这个集群的 Global Application。
2. Sync Global Application。因为应用的 Application 会存储一些信息，需要先把应用的 Application 的 spec 同步上。
3. 通过 `argocd app sync $app` sync 具体应用的 Application。

这里还存在一个问题，即 Global Application 会管理大量的应用（可能有几千个）。这会导致同步 Global Application 非常慢。此外，当多个用户在同一个集群内同时发布应用时，所有人都需要同步这个 Global Application，必然会造成拥堵，导致大家都在这一步等待。

因此我们将这一步也进行了优化。

我们更改了 Global Application 的 sync 方式，不通过 argocd app sync，而是直接执行 kubectl apply。

在 GitHub Action Workflow 中，我们会先找到发生变动的应用。然后直接渲染出它的所有 Application（一般有多个，因为一个应用可能有多个 overlays）。接着，从 ArgoCD 所在的 Kubernetes 集群中列出这个应用的所有 Applications，并进行差异比较，找出不一样的 Applications，执行 `kubectl apply`。这样相当于变相刷新了 Global Application。

流程图如下：

![argocd-sync-workflow](/images/argocd_monorepo/argo-cd-workflow.jpg)

经过上述优化后，我们能做到当一个 argocd 管理同一个 repo 中接近 7k 应用时，用户的发版也能在 30-50s 完成。（其中 sync global app 10s 左右，argocd sync app 在 15-35s 左右）
而且不会影响各个用户之间的发版。

#### 但是仍然存在影响 argocd 稳定性的问题

在完成上述的性能优化后，尽管 argocd 已经能满足业务发版的要求，绝大多数情况下，发版的速度都能满足业务的要求。但不幸的是，仍然存在一些会引起 argocd 抖动的问题。

在业务 Kubernetes 集群内资源变动时，ArgoCD 会对资源所属的 Applications 进行刷新。刷新过程中，controller 会检查缓存，如果缓存存在则结束处理；如果缓存不存在，则会请求 repo server 重新渲染资源。大多数情况下，刷新 Applications 会命中缓存，但在一些极端情况下，可能无法命中缓存，从而导致请求 fallback 到 repo server。由于 repo server 渲染资源非常耗时且占用大量 CPU，当大量渲染请求涌向 repo server 时，会导致其 CPU 被占满，甚至出现 OOM 情况，进而导致正常的同步请求无法处理，用户发布超时。可能造成这问题的原因包括：

1. 当业务 DevOps 批量操作集群内数据时，例如批量重启 deployments，ArgoCD 监听到 deployments 被修改或者 pods 被删除/创建时，会找到相应的 Applications 并进行刷新。这会导致短时间内大量的 applications 被刷新。
2. 当 DevOps 升级业务集群时，由于需要替换 nodes，这将导致大量 pods 被重建，进而致短时间内大量的 Applications 被刷新。
3. 一些经常被 operators 更新的 CRD 也会触发 Applications 刷新。例如，KEDA 会频繁更改 HPA 的 spec，导致 Applications 被频繁刷新。

**优化的方法**

argocd 2.8 之后提供了参数，可以忽略掉集群内一些资源的特定字段的监听：

```yaml
# argo-cd helm values.yaml

# 比如这里配置了所有资源的 `.status` 和 `.metadata.resourceVersion` 的变更，都不会触发 argocd refresh
server:
  config:
    resource.customizations.ignoreResourceUpdates.all: |
      jsonPointers:
      - /status
      - /metadata/resourceVersion
```

但这样仍然不能完全解决问题，因为总有些漏网之鱼，比如新部署了一个 operator 会频繁修改它的 CRD spec 之类的。因此，我们还需要想其他办法来根治这个问题。

然后我们想到，因为我们禁用了所有 Applications 的 auto sync，所以其实这个监听资源变动 + refresh Application 的机制，对我们来说可有可无。

因此我们魔改了 argocd controller 的代码：

如果这个监听到资源的变动，但是这个资源所属的 application 没有开启 auto sync 或者没有正在被操作（正在被手动 sync 中），那么就不 refresh Application。

至此这个问题，才被彻底解决。

改动的 PR:

[Improve performance for refreshing apps by domechn · Pull Request #1 · domechn/argo-cd (github.com)](https://github.com/domechn/argo-cd/pull/1)

---

**除此之外还要保证 repo server 不会因为短时间大量请求被打到 OOM**

首先 repo server OOM 必然导致当前正在处理的 argocd sync 请求失败，其次因为 repo server 重启后要重新拉代码，会很大延长下次 argocd sync 的时间，影响发版稳定性。

不过 argocd 提供了 `reposerver.parallelism.limit` 这个参数，可以限制 repo server 同一时间并发处理渲染请求的数量。这个值是一个经验值，可能要根据 mono repo 大小和 repo server pod resource 来调整。以我们自己的经验来看，我们的 repo 在 4G 左右，repo server resource limit 给的是 12Gi4vCPU, sidecar plugin 是 6Gi4vCPU，那么它设置成 15，repo server 几乎就不会被打到 OOM。

<br/>

这样一番优化完后，argocd 的性能表现终于变得稳定，几乎很少有抖动，而且理论上之后性能也不会太受到 application 数量增长的影响（也行还会有，但目前没发现）

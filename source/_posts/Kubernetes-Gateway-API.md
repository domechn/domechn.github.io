---
title: Kubernetes Gateway API
date: 2021-05-10 00:14:15
subtitle: k8s-gateway-api
tags:
  - kubernetes
  - network
---

## 什么是 Kubernetes Gateway API

Kubernetes Gateway API（以下简称 `KGA`）是由 sig-network 小组提出的，用于帮助用户在 K8S 中对服务网络进行建模的资源的集合。

它通过提供标准化的接口，允许各个供应商提供自己的实现方式，来帮助用户规划集群内的服务网络。

## 演进历史

最初 K8S 提供了 `Ingress` API 来让用户可以定义集群的入口网络。随着流量管理（熔断，限流，灰度）等需求的增加，最初的 `Ingress` 的定义已经无法满足，因此社区开始分裂出了不同的实现方式。

- 利用 `CRD` 扩展 `Ingress`，例如：https://dave.cheney.net/paste/ingress-is-dead-long-live-ingressroute.pdf
- 利用 `Annotation` 扩展 `Ingress`，例如：[nginx-controller](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/)

这些方式虽然能解决现有的问题，但也导致了

- 没有统一的标准，各个 ingress-controller 按自己的方式实现
- 没有可移植性

因此 KGA 应运而生，KGA 需要能让用户很方便的使用以下功能

### 流量管理

- HTTP 流量能够按 `Header`, `URI` 路由
- 流量按权重路由
- 流量镜像
- TCP 和 UDP 等路由

### 面向角色的设计

- 将 Routing 和 Service 的角色分离

### 扩展性

- 支持层级配置，用户可以在上层配置中添加或覆盖下层配置中的属性

### 灵活一致性

- KGA 提供了三种级别的 API
  - core
    - 必须实现
  - extended
    - 可以不实现，如果实现必须保证移植性
  - custom
    - 各个 controller 可以自定义，且不需要提供可移植保证

## KGA

为满足上述功能，KGA 提供了以下几种 API

- **GatewayClass**

  指定 KGA 的实现方式，（比如 `istio` or `nginx`），类似于 `StorageClassess`

- **Gateway**

  指定 Route 应该使用哪个 GatewayClass

- **Route**

  定义路由的具体规则，有 `HTTPRoute`, `TCPRoute`, `UDPRoute`

  指定符合规则的流量应该路由到哪个 `Service`

关于各个 API 的具体字段，可以查看该[文档](https://gateway-api.sigs.k8s.io/api-types/gatewayclass/)

![网关API的资源](https://kubernetes.io/blog/2021/04/22/evolving-kubernetes-networking-with-the-gateway-api/gateway-api-resources.png)

## 示例

有以下常用的情况

foo 小组需要将多个服务部署在 `foo` namespace 中，并要求按照 http 请求能按照 uri 路由到不同的 Service

````yaml

kind: HTTPRoute
apiVersion: networking.x-k8s.io/v1alpha1
metadata:
name: foo-route
namespace: foo
labels:
gateway: external-https-prod
spec:
hostnames:

- "foo.example.com"
  rules:
- matches:
  - path:
    type: Prefix
    value: /login
    forwardTo:
  - serviceName: foo-auth
    port: 8080
- matches:
  - path:
    type: Prefix
    value: /home
    forwardTo:
  - serviceName: foo-home
    port: 8080
- matches: - path:
  type: Prefix
  value: /
  forwardTo: - serviceName: foo-404
  port: 8080`` - bar 小组将服务部署在 `bar` namespace 中，并需要 canary 发布他们的服务 - 将流量按照 9:1 的比例分别发送到, bar-v1 和 bar-v2 - 当请求的 Header 中带有 `env: canary` 时，将流量发布到 bar-v2 - ![The routing rules configured for the bar-v1 and bar-v2 Services](https://kubernetes.io/blog/2021/04/22/evolving-kubernetes-networking-with-the-gateway-api/httproute.png) -``yaml
  kind: HTTPRoute
  apiVersion: networking.x-k8s.io/v1alpha1
  metadata:
  name: bar-route
  namespace: bar
  labels:
  gateway: external-https-prod
  spec:
  hostnames:
- "bar.example.com"
  rules:
- forwardTo:
  - serviceName: bar-v1
    port: 8080
    weight: 90
  - serviceName: bar-v2
    port: 8080
    weight: 10
- matches: - headers:
  values:
  env: canary
  forwardTo: - serviceName: bar-v2
  port: 8080`` - 当两个团队创建好了 `HTTPRoute` 后，该如何将入口流量匹配到这些规则上呢？这里就需要用到 `Gateway` 了 - infra 团队可以创建如下 `Gateway` 来绑定 `Routes` -``yaml
  kind: Gateway
  apiVersion: networking.x-k8s.io/v1alpha1
  metadata:
  name: prod-web
  spec:
  gatewayClassName: acme-lb
  listeners:
- protocol: HTTPS
  port: 443
  routes:
  kind: HTTPRoute
  selector:
  matchLabels:
  gateway: external-https-prod
  namespaces:
  from: All
  tls:
  certificateRef:
  name: admin-controlled-cert``` - 可以看到，`Gateway `中使用 `selector.matchLabels`来选中所有 namespace 下带有`gateway: external-https-prod`Label 的`HTTPRoute` - 同时`Gateway`中指定了`gatewayClassName`为`acme-lb`，所以这些 route 规则就会被同步到 acme-lb 上 - 因此只要将 `foo.example.com`和`bar.example.com` 的 dns 解析到 acme-lb 的 externalIP 上，就可以完成流量的路由
  - 以下是这些 API 的拓补图
    - ![schema](https://gateway-api.sigs.k8s.io/images/schema-uml.svg)
- # Reference
  - https://kubernetes.io/blog/2021/04/22/evolving-kubernetes-networking-with-the-gateway-api/
  - https://gateway-api.sigs.k8s.io/
````

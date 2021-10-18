---
title: Cilium 从 0 到 0.1
date: 2021-10-17 16:40:13
tags:
- kubernetes
- cilium
- cni
- network
---

*本文基于 `Cilium 1.10` 编写*

最近在做 Cilium 相关的技术调研，也趁着周末有时间，将最近的调研结果总结成了这篇文章，顺便理一下自己的思路。这篇文章会和大家分享一些 Cilium 相关的知识，包括以下几个主要部分

- eBPF 的基础知识
- Cilium 对 eBPF 的应用
- Cilium 中一些 Futures 的使用
- 在 Cilium 中如何做 TroubleShooting

当然在开始将 eBPF 和 Cilium 之前，会还是会简单介绍一下以下一些基础知识

- 在传统网络协议栈中，Linux 接收数据包的过程
- CNI 中的网络是如何通信

那么接下来开始这篇文章的主要内容

## Linux 处理数据包的过程

### 接收

这里会简单介绍接收数据包的过程，即数据包是如何从网络被送到内核协议栈的

![receive packet](/images/cilium_0_0_1/network_data_path.jpg)

1. 首先当数据包从网络到达网卡（NIC）
2. 数据包被复制（通过 DMA）到内核内存的环形缓冲区（Ring Buffer）
3. NIC 产生硬件中断（IRQ）让 CPU 知道数据包在内存中
4. CPU 根据注册好的中断函数，找到并调用 NIC Driver 中对应的函数
5. NIC Driver 会禁用硬件中断，之后有数据包继续到 NIC 后，NIC 会直接把数据包写到内存中，不再会产生硬件中断
6. NIC Driver 启动软中断（Soft IRQ），让内核中的 `ksoftirqd` 进程调用 Driver 中的 poll 函数读取 NIC DMA 到内存的数据
7. 通过 Driver 中的函数将读取到的数据，转换成 `skb` 格式（内核规定的格式）
8. GRO 将多个 `skb` 合并成大小相同的 skb
9. `skb` 数据被被协议栈相关的函数处理成协议层的格式
10. 等内存中所有的数据包被处理完成后，NIC Driver 启动 NIC 的硬件终端

### 发送

1. 数据包经过网络协议栈的封装成 `skb` 后会被放到网卡的发送队列中
2. 内核通知网卡发送数据包
3. 网卡发送完成后发送中断给 CPU
4. CPU 收到中断后进行 skb 的清理工作

## CNI 中的网络通信

这里会简单介绍下在 Kubernetes 中，Pod 和 Pod 之间是如何通信的。那么在 K8S 中存在两种 Pod 之间通信的情况，`同节点 Pod 通信` 和 `跨节点 Pod 通信`

在讨论 Pod 和 Pod 通信之前，先了解下 Pod 的数据包是如何发送到 Host 节点上的。

Pod 在 Kubernetes 中是指一组 Containers 的统称，Container 技术本质上是使用了 Linux 的 Namespace 技术。在 Pod 中的所有 Containers 通过共享同一个 Namespace 来实现它们之间的网络共享以及和外部网络的隔离

但被网络隔离起来 Containers 总有访问其他服务的需求，因此它们发出的数据包必定是要能送达到外部的，为了满足这种需求当然有很多方案都能做到，比如将物理网卡或者虚拟网卡 attach 到 Namespace 中

但这里主要介绍的还是大部分 CNI 所采用的方案，那就是 veth-pair

### veth-pair

veth-pair 就是一对的虚拟设备接口，它都是成对出现的，两端彼此相连，一端发送的数据会被另一端收到。正因为有这个特性，它常常充当着一个桥梁，连接着各种虚拟网络设备

如果想要让 Namespace 中的服务能够把数据报文发送到外部，可以创建出一对 veth-pair，一端放在 Namespace 中，另一端放在 Host 中，这样当 Namespace 中的服务发送报文时，报文就会被发送到 Root Namespace （ Host ）中，这样 Host 可以根据 route 表中的路由规则将报文转发到目标服务

当然上面这段只是理想情况，实际上还有很多东西没有考虑进去，比如网络通信的四要素，`src_ip`, `src_mac`, `dst_ip` 和 `dst_mac`，没有这四种信息数据报文自然是无法送达到目的地的。在上述场景中 `src_ip`（Namespace 中被 attach 的 veth-pair 的网卡的 ip）, `src_mac`（同上） 和 `dst_ip` 都是已知的，所以接下来的问题就是如何获取 `dst_mac`

> 在网络通信中 `dst_mac` 是指 nextHop 的地址，如果走 L3 转发那么 dst_mac 则为网关地址，如果走 L2 交换，`dst_mac` 则为目的地址

### 同节点 Pod 通信

如上面所说，同节点 Pod 之间的通信的问题就是如何解决 `dst_mac` 的问题

不同的 CNI 有不同的解决方案。

比如在 `Flannel` 中，Flannel 会在 Host 上创建一个 `NetworkBridge` (`cni0`)，然后将 Pod 的 IP 设置成 `24` 位，`veth-pair` 的一端 attach 到 Pod 中，另一端 attach 到 cni0 上。因为同节点上的 Pod 都是 24 位 IP，所以 Pod 之间通信走二层交换，src Pod 可以通过 arp 请求获取到 dst Pod 的 mac 地址，这样也就解决了 `dst_mac` 的问题

> NetworkBridge 可以看作是虚拟的交换机，attach 到 network bridge 上的设备之间可以相互通信

![flannel_cni0](/images/cilium_0_0_1/flannel_cni0.jpg)

但这篇文章主要会介绍另一种模式（之后所有内容也是基于该模式作为基础），该模式也在 `Calico` 或者是 `Cilium` 中使用

在 `Calico` 或者 `Cilium` 中，Pod 的 IP 会被设置成 32 位。因此在这种情况下，Pod 访问任何其他 IP 都是走 L3 路由

![calico_veth](/images/cilium_0_0_1/calico_veth.jpg)

上面是我从 kubernetes 集群中获取到的数据，cni 为 Calico

从 Pod 中的路由表可以看到，默认网关的地址为 `169.254.1.1`，因此如果要从 Pod1 访问其他服务需要先获取到 `169.254.1.1` 的 mac 地址，但很明显这个 mac 地址是无法通过 arp 请求获取到的

那么如何解决这个问题呢？事实上 Calico 和 Cilium 采用了不同的方案

**Calico**

Calico 使用了 `proxy_arp` 来解决，简单来说开启 proxy_arp 的网络设备可以被当作是一个 arp 网关，当它接收到 arp 请求时，它会把自己的 mac 地址回复给请求者。因为 veth-pair 的缘故，Pod 中 eth0 发出的请求都会被送到它所绑定的 `veth*` 中。

因此给该 `veth*` 开启 proxy_arp 之后，veth* 就能够把它的 mac 回复给 Pod，这样数据报文就能被送出来，当数据包文被送到 Host 中后，再根据 Host 内的本地路由表，将数据报文送到对应的 Pod 挂在 Host 上的 veth-pair 设备上了

**Cilium**

Cilium 的做法则是直接在 Pod 中写一条指向 `169.254.1.1` 的 static arp 规则。这样 Pod 可以直接拿到 nextHop 的 `dst_mac`。

这样做的好处是整个请求流程简单，但需要通过一个外部 agent 去维护 arp 规则和 Pod 之间的关系，也带来了一定的维护成本。

### 跨节点 Pod 通信

本篇文章不会着重去介绍跨节点通信时用到的，比如 `Overlay` 或者是 `Underlay` 之类这些网络技术

而且 Cilium 着重解决的也不是这方面的问题，且 Cilium 在解决跨节点传输的问题上用的也都是非常主流的技术，比如 `VxLan` 或者是 `BGP`，这里就不展开说了

如果对这部分感兴趣的话，推荐大家去看 Calico 这部分的文档

## eBPF 基础知识

在结束了上面的铺垫之后，接下来开始介绍本次的主角 eBPF

目前谈到的 BPF 技术分两种，`cBPF` 和 `eBPF`

`cBPF` 诞生于 1997 年，内核 2.1.75 版本，是类 Unix 系统上数据链路层的一种原始接口，提供原始链路层封包的收发。`tcpdump` 的底层正是采用 cBPF 作为底层包过滤技术

`eBPF` 诞生于 2014 年，内核 3.18 版本，eBPF 新的设计针对现代硬件进行了优化，所以 eBPF 生成的指令集比旧的 BPF 解释器生成的机器码执行得更快

cBPF 现在已经基本废弃，目前内核只会运行 eBPF，内核会将加载的 cBPF 字节码透明地转换成 eBPF 再执行（本文以下皆用 BPF 指代 eBPF）

BPF 拥有以下优点：

- 在内核中运行沙箱程序，而无需修改内核源码或者加载内核模块，将 Linux 内核变成可编程之后，就能基于现有的（而非增加新的）抽象层来打造更加智能、 功能更加丰富的基础设施软件，而不会增加系统的复杂度，也不会牺牲执行效率和安全性
- 可以热重启，并且内核会帮助管理状态，在不会引起流量中断（traffic interrupt）的前提下，原子地替换运行中的程序
- 内核原生，不需要倒入第三方内核模块
- 安全性，内核会检查 BPF 程序确保它不会造成内核崩溃

同时 BPF 拥有以下几种特性：

1. BPF map：高效的 kv 存储
2. 辅助函数：可以方便地使用内核功能或和内核交互
3. 尾调用：高效地调用其他 BPF 程序
4. 安全加固原语
5. 支持 object pinning，实现持久存储
6. 支持 offload 到网卡

接下来会对这些特性做介绍

### BPF Map

为了让 BPF 能够持久化状态，内核提供了驻留在内核空间的高效 kv 存储器（BPF map），BPF map 可以被 BPF 程序访问，可以在多个 BPF 程序之间共享（共享的程序之间不一定要求是相同的类型），也可以通过 fd 的形式被用户空间的程序访问。因此用户程序可以使用 fd 相关的 api 方便的操作 map

> 共享的程序之间不一定要求是相同的类型指： tracing programs 也可以和 networking programs 程序共享 map

![bpf_map](/images/cilium_0_0_1/bpf_map.png)

但 fd 受到进程的生命周期的影响，使得 map 的共享等操作实现起来变得复杂,为了解决这个问题，内核开发了 object pinning 功能，该功能能将 map 的 fd 能保留住，不会随着进程退出而被删除

### 辅助函数

使得 BPF 能够通过一组内核定义的函数调用来从内核中查询数据，或者将数据推送到内核

> 不同类型的 BPF 程序能够使用的辅助函数可能是不同的

### 尾调用 (Tail Calls)

![bpf_tailcall](/images/cilium_0_0_1/bpf_tailcall.png)

BPF 支持一个 BPF 程序可以调 用另一个 BPF 程序，并且调用完成后不用返回到原来的程序（只有相同类型的 BPF 程序才可以尾调用）

和普通函数调用相比，这种调用方式开销最小，因为它是用长跳转（longjump）实现的，复用了原来的栈帧

使用场景

- 可以通过尾调用结构化地解析网络头
- 在运行时原子地添加或替换功能，即动态地改变 BPF 程序的执行行为

### offload

BPF 网络程序，尤其是 tc 和 XDP BPF 程序在内核中都有一个 offload 到硬件的接口，这 样就可以直接在网卡上执行 BPF 程序

本文接下来会讨论在 Cilium 得到广泛使用的两个 BPF 子系统， XDP 和 tc 子系统

### XDP

XDP 为Linux内核提供了高性能、可编程的网络数据路径

XDP hook 位于网络驱动的快速路径上，XDP 程序直接从接收缓冲区中将包拿下来，此时 Driver 还没有将数据包转换成 `skb`，因此该数据包的元信息还没有被解析出。理论上这是软件层最早可以处理包的位置

同时因为 XDP hook 运行在网络驱动的快速路径的原因，运行 XDP BPF 程序必需得到网络驱动的支持

![network_xdp_data_path](/images/cilium_0_0_1/network_xdp_data_path.jpg)

> xdp program 会在第 7 步到第 8 步之间执行

同时 XDP 运行在内核态，并不会绕过内核，这带来了以下好处

- 它可以复用所有上游开发的内核网络驱动，用户空间工具，以及其他一些内核接触设施（例如 BPF 辅助函数在调用自身时可以使用系统路由表，socket 等）
- XDP 在访问硬件是与内核其他部分相同的安全模型
- 无需跨越内核和用户空间
- 可以复用 TCP/IP 协议栈
- 不需要显式地将专门 CPU 分配给 XDP，它可以支持 `不停轮询` 或者是 `中断驱动` 的模式

XDP BPF 程序能够修改数据包的内容。同时 XDP 为每个数据包提供了 256 个字节的 headroom，XDP BPF 程序可以对这部分进行修改，比如在数据包前面添加自定义元数据，该部分数据对内核协议栈不可见，但是对 tc BPF 程序可见

```clang
struct xdp_buff {
    void *data;
    void *data_end;
    void *data_meta;
    void *data_hard_start;
    struct xdp_rxq_info *rxq;
};
```

这个结构是 XDP 程序获取到的数据包的格式

- `data`: 指向数据包起始位置
- `data_end`: 指向数据包结尾
- `data_hard_start`: 指向 hardroom 开始位置
- `data_meta`: 指向 `meta` 信息开始， 刚开始 `data_meta` 和 `data` 相同，随着 meta 信息增加，`data_meta` 开始向 `data_hard_start` 靠近
- `rxq`: 字段指向某些额外的、和每个接收队列相关的元数据

从上面可以得出 `xdp_buff` 的结构

`data_hard_start |___| data_meta |___| data |___| data_end`

以及这些 data 字段之间的关系

`data_hard_start <= data_meta <= data < data_end`

**XDP BPF 程序返回码**

XPD BPF 程序运行结束后会返回一个状态码，告诉驱动如何处理这个数据包

- `XDP_DROP`: 在 Driver 层将该数据包丢弃
- `XDP_PASS`: 将这个包送到内核网络协议栈，这和没有 XDP 时默认的包处理行为是一样的
- `XDP_TX`: 在收到该数据包的网卡上，将该数据包再发出去（数据包一般会被修改）
- `XDP_REDIRECT`: 和 `XDP_TX` 类似，不过是在另一张网卡上将数据包发出去
- `XDP_ABORTED`: 表示程序发生异常，行为和 XDP_DROP 一致，但是会经过 trace_xdp_exception tracepoint，可以通过 tracing 工具来监控这种非正常行为

**XDP 使用案例**

- `DDoS 防御、防火墙`: 得益于 XDP 能够在最早的位置拿到数据包，然后用 `XDP_DROP` 命令驱动将包丢弃，XDP 能够实现非常高效的防火墙策略，对于 DDoS 攻击的场景来说非常理想
- `转发和负载均衡`: 通过 `XDP_TX` 和 `XDP_REDIRECT` 两个动作实现。`XDP_TX` 能够实现发卡模式的负载均衡器
- `栈前过滤/处理`: 对于不符合要求的流量可以尽早使用 `XDP_DROP` 丢弃，比如节点只接受 TCP 流量，那么对于 UDP 请求可以直接丢弃掉。同时 XDP 能够在 NIC Driver 分配 skb 之前修改数据包的内容，对于某些 Overlay 场景（需要封装和解封数据包）来说很有用，并且 XDP 能够在数据包的前面 push 元数据，且该部分对内核协议栈不可见
- `流抽样和监控`：XDP 可以将流量进行分析，对于异常流量可以放到 BPF map 中，提供给其他进程用于分析

> 一些智能网卡（例如支持 Netronome’s nfp 驱动的网卡）实现了 xdpoffload 模式，允许将整个 BPF/XDP 程序 offload 到硬件，因此程序在网卡收到包时就直接在网卡进行处理，不过在该模式中某些 BPF map 类型 和 BPF 辅助函数是不能用的

### tc

除了 XDP 等类型的程序之外，BPF 还可以用于内核数据路径的 tc (traffic control，流量控制)层

tc 和 XDP 主要有以下三点不同之处:

**输入上下文**

相较于 XDP，tc 在流量路径中位于更加靠后的位置（skb 分配之后）。因此对 tc BPF 程序而言，它的输入的上下文是 `sk_buff`, 而非 `xdp_buff`，所以 tc ingress BPF 程序可以利用 `sk_buff` 中内核处理好的包的元数据。当然内核处理这些元数据是需要开销的，包括协议栈执行的缓冲区分配，元数据的提取和其他处理等过程，而 xdp_buff 不需要访问这些元数据，因为 XDP hook 在这之前被调用，所以这是 XDP 和 tc hook 性能差距的重要原因之一

**是否依赖驱动支持**

因为 tc BPF 程序运行在网络栈通用层中的 hook 点，所以它们不需要驱动做任何支持

**触发点**

tc BPF 程序在数据路径上的 ingress 和 egress 都可以触发

XDP BPF 程序只能在 ingress 点触发

#### tc ingress

![network_tc_data_path](/images/cilium_0_0_1/network_tc_data_path.jpg)

tc ingress hook 位于 GRO 之后，处理协议之前最早的处理点

#### tc egress

tc egress hook 运行的位置是，内核将数据包交给 NIC Driver 之前最晚的位置，这个地方在传统 iptables 防火墙 `POSTROUTING` 链之后，但是在 GSO 引擎处理之前

**tc BPF 程序执行模式**

在 tc 中，有以下 4 种组件

- `qdisc`: Linux 排队规则，根据某种算法完成限速、整形等功能
- `class`: 用户定义的流量类别
- `classifier`: 分类器，分类规则。传统的 tc 方案中，classifier 和 action modules 之间是分开的，每个分类器能 attach 多个 action，当匹配到这个分类器时这些 action 就会执行。除此之外，它不仅能够读取 skb 元数据和包数据，还能任意**修改**这两者，最后结束 tc 处理过程，返回一个返回码
- `action modules`: 要对包执行什么动作

当需要给某个网络设备挂载 tc BPF 程序时，需要执行以下操作

- 为网络设备创建 `qdisc`
- 创建 `class`，并 attach 到 `qdisc`
- 创建 `filter (classifier)`，并 attach 到 `qdisc`
  - filter  对网络设备上的流量进行分类，并将包分发（dispatch）到前面定义的不同 class
  - filter 会对每个包进行过滤，返回下列值之一
    - `0`: 表示 mismatch，如果后面有其他 filters，则继续向下执行
    - `-1`: 执行这个 filter 上的默认 `class`
    - `其他`: 表示一个 classid，接下来系统会将数据包送到这个 `class`
- 给 `filter` 添加 `action`
  - 例如，将选中的包丢弃（drop），或者将流量镜像到另一个网络设备等等

**cls_bpf classifier**

cls_bpf 是一种分类器，相比于其它类型的 tc 分类器，它有一个优势：能够使用 direct-action 模式

**direction-action**

如上文所说，tc BPF 程序的执行模式是一个 classifier attach 多个 actions，classifier 负责匹配流量，然后将匹配到的流量交给 action 执行

todo

**tc BPF 程序返回码**

- `TC_ACT_UNSPEC`: 结束当前程序的处理，不指定接下来的操作（内核会根据情况执行下一步操作）。对于以下三种情况，默认操作分别为。
  - 当 `cls_bpf` 被 attach 了多个 tc BPF 程序时，继续下一个 tc BPF 程序
  - 当 `cls_bpf` 被 attach 了 offloaded tc BPF 程序（和 offloaded XDP 程序类似）时，cls_bpf 会返回 `TC_ACT_UNSPEC`，内核会执行下一个没有被 offloaded BPF 程序（一张 NIC 只能 offloaded 一个程序）
  - 当只有单个 tc BPF 程序时，返回该 code 通知内核继续执行 skb 处理，不会带来其他副作用
- `TC_ACT_OK`: 结束当前程序的处理，并告诉内核下一个执行的 tc BPF 程序
- `TC_ACT_SHOT`: 通知内核丢弃数据包，返回 `NET_XIT_DROP` 给调用方表示包被丢弃
- `TC_ACT_STOLEN`: NET_XIT_DROP` 给调用方包被丢弃，返回 `NET_XMIT_SUCCESS` 给调用方，假装这个包被正确发送
- `TC_ACT_REDIRECT`: 使用这个返回码并加上 `bpf_redirect()` 辅助函数，允许重定向一个 `skb` 到同一个或另一个网络设备的 ingress 或 egress 路径。对目标网络设备没有额外的要求，只要本身是 一个网络设备就行了，在目标设备上不需要运行 `cls_bpf` 实例或其他限制

## Reference

- [Cilium Documentation](https://docs.cilium.io/en/stable/)
- [基于 BPF/XDP 实现 K8s Service 负载均衡 (LPC, 2020)](http://arthurchiao.art/blog/cilium-k8s-service-lb-zh/#1-k8s-%E7%BD%91%E7%BB%9C%E5%9F%BA%E7%A1%80%E8%AE%BF%E9%97%AE%E9%9B%86%E7%BE%A4%E5%86%85%E6%9C%8D%E5%8A%A1%E7%9A%84%E5%87%A0%E7%A7%8D%E6%96%B9%E5%BC%8F)

---
title: Cilium 从 0 到 0.1
subtitle: cilium_0_to_0_1
date: 2021-10-17 16:40:13
tags:
- kubernetes
- cilium
- cni
- network
---

*本文基于 `Cilium 1.10` 编写*

这篇文章会和大家分享一些 Cilium 相关的知识，包括以下几个主要部分

- eBPF 的基础知识
- Cilium 对 eBPF 的应用
- Cilium 中一些 Features 的使用
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

Cilium 的做法则是过在 `veth*` 上 attach `tc ingress bpf 程序`，为 Pod 的所有 arp 请求都返回 veth* 的 mac

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

**tc ingress**

![network_tc_data_path](/images/cilium_0_0_1/network_tc_data_path.jpg)

tc ingress hook 位于 GRO 之后，处理协议之前最早的处理点

**tc egress**

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

Cilium 中就使用了 cls_bpf 分类器，它在部署 cls_bpf 服务时，对于给定的 hook 点只会 attach 一个程序，且用的正是 direct-action模式

**direction-action**

如上文所说，传统 tc BPF 程序的执行模式是 classifier 分类和 action modules 执行是分开的，一个 classifier attach 多个 actions，classifier 负责匹配流量，然后将匹配到的流量交给 action 执行

但是对于很多场景，eBPF classifier 已经有足够的能力完成完成任务处理，无需再 attach 额外的 qdisc 或 class 了

所以，为了

- 避免因套用 tc 原有流程而引入一个功能单薄的 action
- 简化那些 classifier 独自就能完成所有工作的场景
- 提升性能

社区为 tc 引入了一个新的 flag：direct-action，简写 da。 这个 flag 用在 filter 的 attach time，告诉系统： classifier 的返回值应当被解读为 action 类型的返回值。这意味着 classifier 加载的 eBPF 现在可以返回 action code 了。比如在要求丢包的场景中，就不需要再引入一个 action 执行 drop 操作，可以直接在 classifier 中完成

这样带了的好处

- 性能提升，因为 tc 子系统无需再调用到额外的 action 模块 ，而后者是在内核之外的
- 程序结构更加简单易用

**tc BPF 程序返回码**

- `TC_ACT_UNSPEC`: 结束当前程序的处理，不指定接下来的操作（内核会根据情况执行下一步操作）。对于以下三种情况，默认操作分别为。
  - 当 `cls_bpf` 被 attach 了多个 tc BPF 程序时，继续下一个 tc BPF 程序
  - 当 `cls_bpf` 被 attach 了 offloaded tc BPF 程序（和 offloaded XDP 程序类似）时，cls_bpf 会返回 `TC_ACT_UNSPEC`，内核会执行下一个没有被 offloaded BPF 程序（一张 NIC 只能 offloaded 一个程序）
  - 当只有单个 tc BPF 程序时，返回该 code 通知内核继续执行 skb 处理，不会带来其他副作用
- `TC_ACT_OK`: 结束当前程序的处理，并告诉内核下一个执行的 tc BPF 程序
- `TC_ACT_SHOT`: 通知内核丢弃数据包，返回 `NET_XIT_DROP` 给调用方表示包被丢弃
- `TC_ACT_STOLEN`: NET_XIT_DROP` 给调用方包被丢弃，返回 `NET_XMIT_SUCCESS` 给调用方，假装这个包被正确发送
- `TC_ACT_REDIRECT`: 使用这个返回码并加上 `bpf_redirect()` 辅助函数，允许重定向一个 `skb` 到同一个或另一个网络设备的 ingress 或 egress 路径。对目标网络设备没有额外的要求，只要本身是 一个网络设备就行了，在目标设备上不需要运行 `cls_bpf` 实例或其他限制

**tc 使用案例**

- `为容器落实策略`: 在容器网络中，veth-pair 一端连接着 Namespace，另一段连接着 Host。容器内所有的网络都会经过 Host 端的 veth 设备，因此在 veth 设备上的 tc ingress 和 egress hook 点可以 attach tc BPF 程序。此时容器内发出的流量都会经过 veth 的 tc ingress hook，进入容器的流量都会经过 veth 的 tc egress hook（对于 veth 这样的虚拟设备，XDP 在该场景下并不合适，因为内核在这里只操作 skb，而通用 XDP 有几个限制，导致无法操作克隆的 skb。而且 XDP 无法处理 egress 流量）
- `转发和负载均和`: 使用场景和 XDP 很类似，只是目标更多的是在东西向容器流量而不是南北向。tc 还可以在 egress 方向使用，例如对容器的 egress 流量做 NAT 和 负载均衡，整个过程对容器是透明的。由于在内核网络栈的实现中，egress 流量已经是 sk_buff 形式的了，因此很适合 tc BPF 对其进行重写（rewrite）和重定向（redirect）。 使用 bpf_redirect() 辅助函数，BPF 就可以接管转发逻辑，将包推送到另一个网络设 备的 ingress 或 egress 路径上
- `流抽样和监控`: tc BPF 程序可以同时 attach 到 ingress 和 egress，另外，这两个 hook 都在（通用）网络栈的更低层，这使得可以监控每台节点 的所有双向网络流量。Cilium 中会使用 tc BPF 能够给数据包添加自定义 annotations 的特性，对被 drop 的包打上 annotations，标记它所属的容器以及被 drop 的原因，提供了丰富的信息

## Cilium 与 eBPF

上文已经描述了 Pod 中的流量是如何顺利发送到 Host 的 veth 上的，接下来会介绍在 Cilium 中流量的完整路径

*当前环境基于 Legacy Host Routing 模式*

在开始之前，先观察一下当集群中安装了 Cilium 之后，会多出来哪些东西，这里可以直接进入 cilium agent 的 Pod，执行网络命令，因为 agent 使用的是 `hostNetwork: true`，所以它和主机是共享网络的，因此在 Pod 中查看到的网络设备也就是主机上的网络设备

```shell
$ k exec -it -n kube-system cilium-qv2cb -- ip a
18: cilium_net@cilium_host: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default qlen 1000
    link/ether fa:f3:f0:8d:5e:ae brd ff:ff:ff:ff:ff:ff
    inet6 fe80::f8f3:f0ff:fe8d:5eae/64 scope link
       valid_lft forever preferred_lft forever
19: cilium_host@cilium_net: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default qlen 1000
    link/ether 3e:87:e2:f3:58:47 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.118/32 scope link cilium_host
       valid_lft forever preferred_lft forever
    inet6 fe80::3c87:e2ff:fef3:5847/64 scope link
       valid_lft forever preferred_lft forever
```

```shell
$ k exec -it -n kube-system cilium-qv2cb -- iptables -L
Chain CILIUM_FORWARD (1 references)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere             /* cilium: any->cluster on cilium_host forward accept */
ACCEPT     all  --  anywhere             anywhere             /* cilium: cluster->any on cilium_host forward accept (nodeport) */
ACCEPT     all  --  anywhere             anywhere             /* cilium: cluster->any on lxc+ forward accept */
ACCEPT     all  --  anywhere             anywhere             /* cilium: cluster->any on cilium_net forward accept (nodeport) */

Chain CILIUM_INPUT (1 references)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere             mark match 0x200/0xf00 /* cilium: ACCEPT for proxy traffic */

Chain CILIUM_OUTPUT (1 references)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere             mark match 0xa00/0xfffffeff /* cilium: ACCEPT for proxy return traffic */
MARK       all  --  anywhere             anywhere             mark match ! 0xe00/0xf00 mark match ! 0xd00/0xf00 mark match ! 0xa00/0xe00 /* cilium: host->any mark as from host */ MARK xset 0xc00/0xf00
```

可以看到分别多出来了 2 个网络设备 `cilium_host` 和 `cilium_net`（如果使用了 overlay 模式，还会有 `cilium_vxlan`），以及 3 条 iptables 规则 `CILIUM_FORWARD`, `CILIUM_INPUT` 和 `CILIUM_OUTPUT`

**cilium_host 和 cilium_net**

`cilium_host` 和 `cilium_net` 是一对 veth pair

通过 `route -n` 查看路由表发现，`cilium_host` 负责处理本机 pod 流量之间的路由

```
10.0.1.0        10.0.1.118        255.255.255.0   UG    0      0        0 cilium_host
10.0.1.118        0.0.0.0         255.255.255.255 UH    0      0        0 cilium_host
```

那么当流量从同节点的 Pod1 发送到 Pod2 时，需要用到这个路由表吗？答案是不需要，之前讨论 tc 的时候有说到，tc 可以使用辅助函数获取路由表的内容，并且支持直接 redirect 流量到另一张网卡，那么只需要在 Pod1 的 lxc (veth pair) 上 attach tc ingress BPF 程序就可以直接将流量发送到 Pod2 的 lxc

那么 `cilium_host` 这个虚拟网络设备的用处是什么呢？那就是用来接收跨节点的 Pod 或者是外网到 Pod 的流量。当有非本地发送过来到 Pod 的流量时，Host 上的 eth0 网卡上的 tc ingress 程序会对流量做某些处理（如果是 Pod 访问外网的回程报文则是 SNAT 还原，如果是 Pod 之间的访问则不需要操作），处理完成后将数据报文交给内核路由系统，最终送到 `cilium_host` 中，再由 cilium_host 上的 tc BPF 程序将流量 redirect 到 Pod 的 lxc

除此之外，Cilium 还在 Host 的出口网卡上 attach 了 tc ingress 和 egress hook

### tc BPF Program

以上了解了在 Cilium 中通信会依赖三个网络设备，分别是 `eth0`, `cilium_host` 和 `lxc`

那么在来看下这些设备上都被 attach 了哪些 tc hook

通过 `tc filter show dev <dev_name> (ingress|egress)` 命令可以查看到 attach 的 program 名称

**lxc**

- ingress: `to-container`
- egress: `from-container`

`to-container`

- 提取数据包中的 identity 信息（该信息由 Cilium 设置，包含了数据包所属的 namespace，service，pod 和 container 信息）
- 检查接收容器 ingress policy
- 如果检查通过，将数据包发送到 lxc 的对端，也就是 pod 的网卡

`from_container`

- 如果访问的是 service ip，先对 service ip 做负载均衡
- 执行 DNAT，将 dst_ip 替换成 pod ip
- 将数据包交给内核路由系统

**cilium_host**

- ingress: `to-host`
- egress: `from-host`

`to-host`

- 解析这个包所属的 identity，并存储到包的结构体中
- 验证 dst 的 ingress policy

`from-host`

- 尾调用 dst endpoint 的 ingress BPF 程序 (to-container)

**eth0**

- ingress: `from-netdev`
- egress: `to-netdev`

`from-netdev`

- 解析这个包所属的 identity (还原SNAT)，并存储到包的结构体中
- 将包交给内核转发

`to-netdev`

- 在某些场景下对数据包做 SNAT

因此基于以上的前提，大概可以画出集群内的流量路径

![cilium_legacy](/images/cilium_0_0_1/cilium_legacy_data_path.png)

> 上图引用自 https://www.infoq.cn/article/p9vg2g9t49kpvhrckfwu

1. 跨节点 Pod to Pod
  Pod 发出的流量经过 lxc 上 tc ingress hook 的处理后发给内核协议栈进行路由查找，然后通过 eth0 的 egress 发出到达 Node2 的 eth0
  Node2 接收流量后，先经过 Node2 物理口 eth0 的 tc ingress hook 的 eBPF 处理，然后送给 kernel stack 进行路由查找，发给 cilium_host 接口，流量经过 cilium_host 的 tc egress hook 点后，最后 redirect 到目的 Pod 的宿主机 lxc 接口上，最终送给目的 pod。
2. 同节点 Pod to Pod
  同节点 Pod 之间的流量可以通过 lxc 接口上的 tc ingress hook 直接 redirect 到目标 Pod，这种方法可以绕过内核网络协议栈，将数据包送往目标 Pod 的 lxc 接口，进而送给目标 Pod
3. NodePort Local Endpoint
  流量从 Node2 的 eth0 进入后，经过 tc ingress hook 的 DNAT 处理后，将流量交给内核协议栈路由，内核将数据包转发给 cilium_host 接口，cilium_host 的 tc egress hook 处理后将流量 redirect 到 Pod 的 lxc 接口，进而送给 Pod
  回程数据包经过 ；xc 的 tc ingress hook 反 DNAT 后，将流量重定向到 eth0，过程不经过 kernel，最后经过 eth0 的 tc egress hook 返回给请求者
4. NodePort Remote Endpoint
  eth0 的 tc ingress hook 进行 DNAT 将 nodeport ip 转换成 pod ip，然后 tc egress hook 经过 SNAT 将数据包的 src ip 换成 node ip 后将流量发出
5. Pod 访问外网
  Pod 发出流量后，经过 lxc 的 tc ingress hook 处理后，送给内核协议栈进行路由查找，确定需要从 eth0 发出，因此流量回发给物理口 eth0，而经过物理口 eth0 的 tc egress hook 做 SNAT，将源地址转换成 Node 节点的物理接口的 IP 和 Port 发出。
  从外网回来的反向流量，经过 Node 节点物理口 eth0 tc ingress hook 进行 SNAT 还原，然后将流量发给内核协议栈查路由，流量流到 cilium_host 接口后，经过 tc egress eBPF 程序，直接识别出是发给 Pod 的流量，将流量直接 redirect 到目的 Pod 的宿主机 lxc 接口上，最终反向流量回到 Pod
6. 主机访问本 Pod
  主机访问 Pod 流量使用 cilium_host 口发出，所以在 tc egress hook 点 eBPF 程序直接 redirect 到目的 Pod 的宿主机 lxc 接口上，最终发给 Pod。
  反向流量，从 Pod 发出到宿主机的接口 lxc，经过 tc ingress hook eBPF 识别送给 kernel stack，回到宿主机上

那么上面提到的 Cilium 在 iptables 中创建的三条链是用来做什么的呢？

因为在 `Legacy` 模式中数据包在某些情况下仍然需要进入内核协议栈，因此仍然需要用到 iptables。通过 `FORWARD` + `POSTROUTING` 链将数据包从 lxc 接口送到 `cilium_host`，而 `INPUT` 链则是为了处理 L7 Network Policy Proxy 的情况，这个部分下面会讲

### BPF Host Routing

在 5.10 内核以后，Cilium 新增了 eBPF Host-Routing 功能，该功能更加速了 eBPF 数据面性能，新增了 `bpf_redirect_peer` 和 `bpf_redirect_neigh` 两个 redirect 方式

- `bpf_redirect_peer`
  可以理解成 `bpf_redirect` 的升级版，其将数据包直接送到 veth pair Pod 里面接口 `eth0` 上，而不经过宿主机的 `lxc` 接口，这样实现的好处是数据包少进入一次 cpu backlog queue 队列，该特性引入后，路由模式下，Pod -> Pod 性能接近 Node -> Node 性能
- `bpf_redirect_neigh`
  用来填充 pod egress 流量的 src 和 dst mac 地址
  流量无需经过 kernel 的 route 协议栈
  处理过程
    - 首先会查找路由，`ip_route_output_flow()`
    - 将 skb 和匹配的路由条目（dst entry）关联起来，`skb_dst_set()`
    - 然后调用到 `neighbor subsystem`，`ip_finish_output2()`
        - 填充 neighbor 信息，即 `src/dst MAC` 地址
        - 保留 `skb->sk` 信息，因此物理网卡上的 qdisc 都能访问到这个字段

在引入了该特性之后，流量路径发生了较大的变化

![cilium_bpf_data_path](/images/cilium_0_0_1/cilium_bpf_data_path.png)

可以看到在该场景下，除了本机访问本地 Pod，其他流量均不会经过内核协议栈，`cilium_host` 和 `cilium_net` 上

1. 跨节点 Pod to Pod
  Pod 发出的流量通过 lxc 的 tc ingress hook 中的 `redirect_neigh` 直接通过 eth0 的 egress 发出到达 Node2 的 eth0
  Node2 的 eth0 接收流量后，先经过 Node2 物理口 eth0 的 tc ingress hook 的 eBPF 处理，然后送给 Pod2 的 lxc 上的 tc ingress hook 中的 `redirect_peer` 直接把数据送到 Pod2 的 eth0
2. 同节点 Pod to Pod
  不同于 Legacy 模式，直接将数据包送到目的 Pod 的 eth0 网卡

以下这张图会更加直观的表示在 BPF Host Routing 模式下数据路径的简单

![ebpf_hostrouting](/images/cilium_0_0_1/ebpf_hostrouting.png)

## Features

### Network Policy

Cilium 支持基于 `Layer 3/4/7` 的网络策略，同时支持 `allow` 和 `deny` 两种模式

> 相同的规则，deny 的优先级更高

Cilium 除了兼容 Kuberentes 的 `NetworkPolicy` API，也提供了 CRD 用于定义网络策略 `CiliumNetworkPolicy`（具体字段请看[官方文档](https://docs.cilium.io/en/stable/policy/intro/#rule-basics)，这里不做介绍）

Cilium 提供了三种网络策略模式

- `default`: 默认模式，如果一个 `endpoint` 设置了一个 `ingress`，那么不符合这个规则的 ingress 请求都会被拒绝。`egress` 同理。不过 `deny` 规则不同，如果一个 `endpoint` 只设置了 deny，那么只有命中 deny 规则的请求会被拒绝，其他还是会被放行，如果一个 `endpoint` 没有设置任何 rule，那么它的网络不受任何限制
- `always`: 如果一个 `endpoint` 没有设置任何 rule，那么它无法访问其他服务，也无法被其他服务访问
- `never`: 不启动 Cilium 的网络策略，所有服务之间都能互相或者和外部访问

**Layer 3**

- `Labels Based`: 根据 Pod labels 选择对应的 ip 然后过滤
- `Services Based`: 根据 Service labels 选择对于的 ip 然后过滤
- `Entities Based`: Cilium 内置了以下字段来指定特定的流量
  - `host`: 本节点的流量
  - `remote-node`: 集群内其他节点的 Endpoint
  - `cluster`: 集群内部 Endpoint
  - `init`: 在引导阶段还没有被解析的 Endpoint
  - `health`: Cilium 用来检查集群状态的 Endpoint
  - `unmanaged`: 不是由 Cilium 管理的 Endpoint
  - `world`: 所有外部流量，允许 world 流量相当于允许 `0.0.0.0/0`
  - `all`: 以上所有
- `IP/CIDR Based`: 基于 IP 过滤
- `DNS Based`: 基于 dns 解析后的 IP 过滤，Cilium 会在 agent 中运行 dns proxy，然后缓存 dns 解析后的 ip list，如果 dns name 满足 allow 或者 deny 规则，那么所有发往这些 ip list 的请求都会被过滤

**Layer 4**

- `Port`: 按端口号过滤
- `Protocol`: 按协议过滤，支持 `TCP`, `UDP` 和 `ANY`

**Layer 7**

- `HTTP`: 支持根据请求 `Path`, `Method`, `Host`, `Headers` 过滤
- `DNS`: 不同于 Layer 3 中的 `DNS Based` 过滤，这是直接过滤 DNS 请求（因为是在 Layer 7 的缘故）
- `Kafka`: 支持根据 `Role` (`produce`, `consume`), `Topic`, `ClientID`, `APIKey` 和 `APIVersion` 过滤

注意：使用了 Layer 7 Network Policy 之后，所有请求都会被转发到 Cilium agent 的 Proxy 中，该 Proxy 由 Envoy 提供，在 Legacy Host Routing 模式下，数据路径会变成下面这样

![cilium_bpf_endpoint](/images/cilium_0_0_1/cilium_bpf_endpoint.svg)

上图描述了 Endpoint to Endpoint 使用了 Layer 7 NetworkPolicy 时的数据路径，上半部分为默认情况下的数据路径，下半部分为使用了 socket 增强之后数据的路径

### Bandwidth Manager

Cilium 还提供了带宽限制的功能

可以通过在 Pod annotation 中添加 `kubernetes.io/ingress-bandwidth: "10M"` 或者 `kubernetes.io/egress-bandwidth: "10M"` 来限制单个 Pod 的带宽。这样 Pod 的出口和入口带宽会被限制在 `10Mbit/s`

不过目前该功能还无法和 Layer 7 NetworkPolicy 同时工作，两者都使用的话会导致带宽限制失效

## Trouble Shooting

这部分会介绍一些工具来方便排查网络问题，主要是 cilium cli 的使用

可以 exec 到 cilium pod 中使用 cilium 命令行进行调试

**查看集群网络状态**

`cilium status`

```shell
KVStore:                Ok   Disabled
Kubernetes:             Ok   1.21+ (v1.21.2-eks-0389ca3) [linux/amd64]
Kubernetes APIs:        ["cilium/v2::CiliumClusterwideNetworkPolicy", "cilium/v2::CiliumEndpoint", "cilium/v2::CiliumNetworkPolicy", "cilium/v2::CiliumNode", "core/v1::Namespace", "core/v1::Node", "core/v1::Pods", "core/v1::Service", "discovery/v1::EndpointSlice", "networking.k8s.io/v1::NetworkPolicy"]
KubeProxyReplacement:   Disabled
Cilium:                 Ok   1.10.3 (v1.10.3-4145278)
NodeMonitor:            Listening for events on 8 CPUs with 64x4096 of shared memory
Cilium health daemon:   Ok
IPAM:                   IPv4: 1/254 allocated from 10.0.1.0/24,
BandwidthManager:       Disabled
Host Routing:           Legacy
Masquerading:           Disabled
Controller Status:      30/30 healthy
Proxy Status:           OK, ip 10.0.1.118, 0 redirects active on ports 10000-20000
Hubble:                 Ok   Current/Max Flows: 4095/4095 (100.00%), Flows/s: 12.47   Metrics: Ok
Encryption:             Disabled
Cluster health:         5/5 reachable   (2021-10-18T15:01:07Z)
```

**抓包**

`cilium monitor`

在 BPF 场景下因为数据路径和传统网络栈发生较大改变，如果不熟悉这套模式在使用比如 `tcpdump` 等工具抓包调试时可以产生一些问题。（比如在 BPF Host Routing 下，lxc 接口是无法抓到回程报文的）

好在 cilium 提供了一套工具用于分析数据包，方便开发者进行问题排查

**NetworkPolicy Tracing**

如果集群中使用了较多的网络策略，有可能导致某些情况下请求命中了意料之外的 NetworkPolicy 导致失败

Cilium 也提供了 policy tracing 的功能用来追踪请求命中 `NetworkPolicy` 的情况

`cilium policy trace`

```shell
# 验证从 default ns 下 xwing 发出的流量，到 default ns 下带有 deathstar label 的 endpoint 的 80 端口的流量会命中哪些 cnp
$ cilium policy trace --src-k8s-pod default:xwing -d any:class=deathstar,k8s:org=expire,k8s:io.kubernetes.pod.namespace=default --dport 80
level=info msg="Waiting for k8s api-server to be ready..." subsys=k8s
level=info msg="Connected to k8s api-server" ipAddr="https://10.96.0.1:443" subsys=k8s
----------------------------------------------------------------
Tracing From: [k8s:class=xwing, k8s:io.cilium.k8s.policy.serviceaccount=default, k8s:io.kubernetes.pod.namespace=default, k8s:org=alliance] => To: [any:class=deathstar, k8s:org=empire, k8s:io.kubernetes.pod.namespace=default] Ports: [80/ANY]

Resolving ingress policy for [any:class=deathstar k8s:org=empire k8s:io.kubernetes.pod.namespace=default]
* Rule {"matchLabels":{"any:class":"deathstar","any:org":"empire","k8s:io.kubernetes.pod.namespace":"default"}}: selected
    Allows from labels {"matchLabels":{"any:org":"empire","k8s:io.kubernetes.pod.namespace":"default"}}
      Labels [k8s:class=xwing k8s:io.cilium.k8s.policy.serviceaccount=default k8s:io.kubernetes.pod.namespace=default k8s:org=alliance] not found
1/1 rules selected
Found no allow rule
Ingress verdict: denied

Final verdict: DENIED
```

**Dns Based NetworkPolicy Debug**

随着 DNS 查询结果的变化，FQDN Policy 的拦截结果也在变，导致这部分难以 debug

可以使用 `cilium fqdn cache list` 查看当前 dns proxy 中缓存了哪些 dns-ip

如果流量是允许的，那么这些 IPs 应该存在于 local identities，`cilium identity list | grep <IP>` 应该返回结果

**Hubble**

Cilium 还提供了 Hubble 用来加强网络监控和报警，Hubble 提供了以下功能

- 可视化
  - 服务的调用关系
  - 支持 HTTP，kafka 协议
- 监控和报警
  - 网络监控
    - 过去一段时间内是否有网络通信失败，通信失败的原因
    - 应用监控
      - `4xx` 和 `5xx` HTTP Response code 出现的频率，出现在哪些服务
      - http 调用的 `latency`
  - 安全性监控
    - 是否有因为 NetworkPolicy Deny 失败的请求
    - 哪些服务，有接收集群外发来的请求
    - 哪些服务要求解析某个特定的 dns

## Reference

- [Cilium Documentation](https://docs.cilium.io/en/stable/)
- [网易轻舟对 CIlium 容器网络的探索和实践](https://www.infoq.cn/article/p9vg2g9t49kpvhrckfwu)
- [Kubernetes service load-balancing at scale with BPF & XDP](https://linuxplumbersconf.org/event/7/contributions/674/)

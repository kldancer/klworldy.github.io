+++
title = 'ryu+ovs 在容器网络中的应用'
date = 2025-09-13T00:53:56+08:00
draft = false
tags= ["网络","sdn","cni"]
categories= ["网络"]
author= ["kl"]
+++

# 前言

博主之前从事过基于ovs的自研cni的研发，当时是选择ryu作为ovs的南向控制端，这次来好好的剖析一下该ryu-controller程序的架构、工作流程和思想。

# 项目概述

**功能定位**: 本程序基于 Ryu SDN 框架构建，是 CNI项目的核心网络控制组件。OVS CNI 组件的控制端，负责通过 OpenFlow 协议向 OVS 交换机下发流表规则。
* 
![ryu-in-cni-01.png](/img/ryu-in-cni-01.png)

# 整体架构
![ryu-in-cni-02.png](/img/ryu-in-cni-02.png)

### **1. RyuController 类**

主控制器，继承自 `RyuApp` 和 `FlowManager`，负责：

- OpenFlow 交换机连接管理
- 数据包处理（Packet-In 事件）
- 流表生成和下发
- 与 cni-ctl 的交互

### 2. **FlowManager 类**

流表管理器，负责：

- 流表的增删改查
- 流表匹配字段和动作的格式化
- 异步请求和回调管理

### **3. DefaultFlowGenerator 类**

默认流表生成器，负责初始化 OVS 流表结构。

**核心流表表** (FlowTable):

```python
DEFAULT_TABLE = 0              # 默认入口表
CT_SVC_TRACE_TABLE = 2         # Service 追踪表
CT_EP_TABLE = 3                # 连接跟踪端点表
STATIC_TABLE = 4               # 静态规则表
FILTER_TABLE = 5               # 过滤表
LEARN_TABLE = 6                # 学习表
CT_CHECK_TABLE = 10            # 连接状态检查表
CT_SVC_TABLE = 12              # Service 连接跟踪表
BASE_LEARN_TABLE = 15          # 基础学习表
CT_COMMIT_TABLE = 20           # 连接提交表
CT_NAT_TABLE = 30              # NAT 处理表
LOCAL_TABLE = 60               # 本地处理表
NORMAL_TABLE = 70              # 转发表
EXTERNAL_TABLE = 80            # 外部处理表
```

**寄存器用途 (reg0-reg7)**

```python
reg0        # 下一跳表号
reg1        # NAT 标记 (1=经过 NAT)
reg2        # VLAN ID
reg3        # Service Group ID
reg4        # Endpoint 索引
reg5        # NAT 数据包标记 (1=经过 NAT)
reg6        # CT Zone (conntrack)
reg7        # 隧道标记
reg8        # 出口端口(mix mode)
```

### **4. NetworkPolicyAPI 类**

网络策略检查接口，与 cni-ctl 通信。

**关键数据结构 - NetworkPolicy**:

```python
class NetworkPolicy:
    allowed              # 是否允许该流量
    src / dst            # 源/目标 IP
    src_mac / dst_mac    # 源/目标 MAC
    src_type / dst_type  # IP 类型（POD/NODE/SERVICE/EXTERNAL/LOCAL）
    src_on_local / dst_on_local  # 是否在本地节点
    eps                  # 端点信息（用于 Service）
    mix_mode             # 混合模式（overlay/underlay）
```

### **5. Tracker 类**

流量跟踪器，通过嗅探和分析网络流量来：

- 学习源 MAC 地址
- 追踪流量路径
- 建立流量轨迹映射

### **6. ControllerAPI 类**

暴露HTTP REST API 接口，以供手动调用接口来操作流表，以及监控监察、Prometheus指标暴露。

# 数据流处理流程

## **关键流程 : Packet-In 处理流程**

1. 交换机中的包，在目的MAC尚未被学习（未匹配到流表项）的状况下，会将数据包进行`flooding`泛洪，交换机会生成一个`Packet-in`消息，将该数据包及其相关信息发送给控制器。
2. 控制器收到数据包，首先将包计数器加1，然后根据包解析的结果，来处理不同类型的数据包，包括ARP、UDP、IP包。
3. 根据数据包的源和目的IP地址、MAC地址、端口信息、是否为回复包以及是否执行NAT等信息，访问cni-ctl的GRPC`/route/exist`接口获得，端到端通信的七元组规则。
4. 根据以上数据发送或丢弃数据包并**生成流表到表3**，主要包括**是否允许流量通过（policy.allowed）**、是否需要进行NAT转换（policy.dst_type）、是否需要将流量转发到服务**（svc流量生成流表到表0）**（policy.eps）、是否需要进行源端口和目的端口的转换（policy.src_port）等。

ovs控制器处理请求首包，然后向cni-ctl查询该数据包的路由方式，计算并下发正向及反向流表至**表3（svc发至表0）**，最后将数据包送回ovs按照一定的指令转发。控制器与ovs进行异步通信，数据包路由查询均通过cni-ctl缓存得到，所以首包的处理耗时主要在TCP连接建立，耗时一般在4至7ms。
![ryu-in-cni-03.png](/img/ryu-in-cni-03.png)

# 关键技术点

## **Conntrack 集成**

conntrack（Connection Tracking）是 Linux 内核网络协议栈中一个非常重要的模块，主要用于**跟踪网络连接的状态**。它的核心作用是：

- **状态保持：** 记录每个网络连接的详细状态信息，例如源IP、目的IP、源端口、目的端口、协议、以及连接所处阶段（新建、建立、关闭等）。
- **报文过滤和修改依据：** 许多网络功能（如 NAT、防火墙）都依赖 conntrack 来对属于同一连接的报文进行一致的处理。

**它在 Linux 内核中的作用：**

conntrack 是 netfilter 框架的一部分，而 netfilter 负责 Linux 防火墙（iptables / nftables）以及 NAT 等功能。当一个报文进入或离开网络栈时，netfilter 的钩子点会检查 conntrack 记录，以确定该报文属于哪个连接，以及该连接的当前状态。

**它在 OVS 中的作用：**

OVS 作为软交换机，为了实现更复杂的网络功能（如防火墙、NAT、负载均衡），也需要 conntrack 的能力。OVS 可以通过其 OpenFlow 流表来**利用内核的 conntrack 模块**。这意味着 OVS 本身不实现一套完整的 conntrack，而是通过特定的 OpenFlow 动作（ct 动作）将报文递交给内核的 conntrack 模块进行处理和状态跟踪。

**常见ct_state的含义：**

- **trk (tracked)：** 表示这个报文已经被 conntrack 跟踪。这是所有 ct_state 标志的基础。如果一个报文没有 trk 标志，说明它没有经过 conntrack 处理。
- **new (new connection)：** 表示这个报文是某个新连接的第一个报文。例如，TCP 的 SYN 包。
- **est (established)：** 表示这个报文属于一个已建立的连接。例如，TCP 连接经过三次握手后，后续的数据传输报文都会被标记为 est。
- **rpl (reply)：** 表示这个报文是连接的回复方向的报文。例如，如果 conntrack 记录的是客户端到服务器的连接，那么服务器到客户端的报文就会被标记为 rpl。
- **related：** 表示这个报文与某个已存在的连接相关，但不是该连接的一部分。例如，FTP 数据连接与控制连接就是 related 关系。
- **inv (invalid)：** 表示这个报文被 conntrack 认为是无效的，可能因为报文格式错误、状态不匹配等。
- **snat / dnat：** 表示连接进行了源 NAT (SNAT) 或目的 NAT (DNAT)。
- **!src / !dst：** 表示连接没有进行源 NAT 或目的 NAT。

## **热缓存优化**

对 TCP SYN/ACK 进行缓存，加快判断是否为回包：

- TCP 三次握手本质：
    - A -> B: SYN (A发起，dst=B:port)
    - B -> A: SYN-ACK (B回复)
    - A -> B: ACK (确认)
- 控制器的判别条件比较粗糙：默认用是否带 ACK 来判断“回包”（ack => is_reply True）。
  但是首包（SYN）和之后的包在异步/重传/网络延迟/流经 NAT 的场景下，控制器看到的顺序或字段可能复杂（例如 NAT 改变地址/端口、controller 转发 PacketOut 丢失某些 reg/标记），使得仅用 ACK 位容易误判。

所以需要会把 "B:port" 记录为“热目标”，这样才不会把A到B的ACK判断为回包。如果错判于是可能只创造“回程”流或以回包优先逻辑处理，导致正向流尚未/不正确下发，出现一向流或丢包。

```python
if l4_pkt.has_flags(tcp.TCP_SYN) and not l4_pkt.has_flags(tcp.TCP_ACK):
    set_hot_cache(f'{ip_pkt.dst}:{dst_port}')  # 记录新连接

if has_hot_cache(f'{ip_pkt.src}:{src_port}'):  # 检查是否回包
    is_reply = True
```

## 流表自学习

从业务网卡或隧道网卡进入的数据包将**通过表4送至表15，**学习生成回程的表项下发到表6。支持tcp、udp、sctp协议。**学习的方式是，将源和目标的IP、端口、Mac地址、隧道目标IP进行交换。**

```bash
cookie=0x0, duration=1448.857s, table=15, n_packets=7, n_bytes=546, priority=30,tcp 
	actions=learn(table=6,idle_timeout=60,priority=10,eth_type=0x800,nw_proto=6,
	NXM_OF_IP_SRC[]=NXM_OF_IP_DST[],
	NXM_OF_IP_DST[]=NXM_OF_IP_SRC[],
	NXM_OF_TCP_SRC[]=NXM_OF_TCP_DST[],
  load:0x3c->NXM_NX_REG0[0..7],load:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],load:NXM_NX_TUN_IPV4_SRC[]->NXM_NX_TUN_IPV4_DST[]),resubmit(,20)
```

## 智能网卡流表卸载

核心思想是：**只有当流表规则的匹配字段和执行动作都在智能网卡支持的硬件能力范围内时，才可能被卸载。** 任何超出硬件支持范围的匹配或动作，都将导致流表无法被卸载，而只能在 OVS 软件路径中处理。

OVS 流表分为逻辑上的 OpenFlow 流表和数据路径上的实际流表。
![ryu-in-cni-04.png](/img/ryu-in-cni-04.png)
- **用户态流表 (OpenFlow 流表)：**
    - **查看方式：** ovs-ofctl dump-flows <bridge_name>
    - **含义：** 这是 OpenFlow 控制器下发给 OVS 的流表。它以 OpenFlow 协议规定的格式存在于 OVS 的用户空间 ovs-vswitchd 进程中。这些流表定义了数据包在 OVS 虚拟交换机中的完整逻辑处理路径，包括匹配条件、优先级、动作、跳转到其他表等。它们是高级别的、抽象的规则。
    - **作用：** 指导 OVS 如何处理数据包。当数据包到达 OVS 时，它会首先尝试匹配这些用户态流表。
- **内核态流表 (Datapath Flows 或 Fast Path Flows)：**
    - **查看方式：** ovs-appctl dpctl/dump-flows 或 ovs-appctl dpctl/dump-flows -m type=offloaded
    - **含义：** 当数据包匹配到用户态流表并被处理后，OVS 会将这个处理结果（匹配条件和最终动作）缓存起来，以内核数据路径能够理解的更精简、更高效的格式存储在内核中。这就是“内核态流表”。它通常是用户态流表的一个优化过的、简化的版本，用于加速后续相同流量的处理。
    - **作用：** 提供“快速路径”处理。如果后续有相同特征的数据包到来，可以直接在内核态匹配到这些缓存的流，而无需再次查询用户态的复杂流表，从而提高转发效率。
- **卸载流表 (type=offloaded)：**
    - **查看方式：** ovs-appctl dpctl/dump-flows -m type=offloaded
    - **含义：** 这正是智能网卡卸载后，**OVS 数据路径层面对那些已成功卸载到硬件的流的内部记录。** 当 OVS 或 DPDK 成功地将一条内核态流下发给智能网卡硬件进行处理时，OVS 内部会将这条流标记为 offloaded。这条命令显示的就是 OVS 认为已经卸载到硬件的流。
- **TC Flower Offload：** 这是一个 Linux 内核功能，允许 OVS 将其 OpenFlow 流表转换为 tc flower 过滤器，并由网卡驱动将其卸载到支持 TC Flower 卸载的智能网卡硬件上。
    - **查看方式：**tc filter show dev <interface> ingress
    - **含义：**对于许多智能网卡，tc flower 过滤器是它们将流规则卸载到硬件的一种标准 Linux 内核接口。也就是说，OVS 可能会将一些它认为可卸载的流，通过 netlink 接口转换为 tc flower 过滤器，然后内核的 tc 子系统会尝试将这些过滤器下发给网卡驱动。如果网卡驱动支持，就会将其编程到硬件。
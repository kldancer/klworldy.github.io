+++
title = 'kube-ovn vpc网络设计概览'
date = 2025-01-20T15:00:52+08:00
draft = false
tags= ["网络","kube-ovn"]
categories= ["网络"]
author= ["kl"]
+++

# 前言

临近过年，总算是闲下来了，博主自9月份入职新公司后，一直在做容器网络相关的工作，有了一些积累，乘着年前闲暇做做梳理。

一个CNI 网络插件的设计，我认为可以大致分为控制面的设计和数据面的设计。如何管理IPAM、如何维护k8s cluster、node、pod、svc等等信息属于控制面，如何设计并维护容器网络的数据路径datapath这属于数据面；本篇文章，我想探讨一下kube-ovn这个项目对容器网络数据路径的设计。

# 数据面总体设计拓扑

为了直观清晰，先上图：

![image.png](/img/kube-ovn-vpc_1.png)

VPC、Subnet是kube-ovn中的核心概念，对应到ovn层面就是三层逻辑路由器、二层逻辑交换机。kube-ovn通过这样的对应关系来构建容器网络的逻辑网络拓扑。

## 默认VPC网络**：解读 kube-ovn 默认网络架构**

在 kube-ovn 的部署中，默认容器网络通过创建一个默认 VPC 网络实现。以 Overlay 模式为例，系统会自动生成以下资源：

- **VPC**：ovn-cluster
- **子网**：ovn-default 和 join

它们的逻辑关系如图所示可以类比为一个真实的路由器连接两个交换机。

### 容器网络之间通信

`ovn-cluster` 路由器通过其`ovn-cluster-ovn-default`端口连接`ovn-default`交换机，端口的 IP 地址为 10.16.0.1/16，这也是默认容器网络的网段。

在默认网络中，Pod 的主机网络侧 veth 设备被接入到 OVS 虚拟网桥 `br-int`，相当于 Pod 内部网络侧的 veth 设备被连接到 `ovn-default` 交换机的某个端口。

这种网络拓扑的设计使得同一子网内的 Pod 像连接在同一交换机上的设备，可以直接通过二层网络通信。

### 容器网络与主机网络通信

join子网是专用于默认容器网络与主机网络之间的通信而设计的。

首先，在每个节点上创建 `ovn0` 网络设备，并将其接入 OVS 虚拟网桥 `br-int`。在逻辑层中，这表现为 `ovn-cluster` 路由器通过 `ovn-cluster-join` 端口连接到 `join` 交换机，而 `join` 交换机分别与各节点的 `ovn0` 设备相连。

其次，在`ovn-cluster`路由器中存在路由规则：`ovn-nbctl lr-route-add router "0.0.0.0/0" 100.64.0.1` ，该规则指示所有未匹配到其他路由的容器网络流量发送到 join 网段。同时node主机上也存在路由`ip route add 10.16.0.0/16 via 100.64.0.1` ，该路由将主机发往容器网络的数据包转发到 ovn0 设备。

最终，`ovn-cluster` 路由器连接了 `ovn-default` 和 `join` 两个交换机：

- `ovn-default` 交换机管理默认容器网络 Pod 的通信。
- `join` 交换机负责主机网络与容器网络的数据传递。

通过上述路由和网络拓扑设计，实现了容器网络与主机网络间的无缝互通，形成一个闭环的数据流路径。

```bash
root@kf1:~# kubectl ko nbctl show ovn-cluster 
router 5f8722a2-a1aa-4479-a8a4-62c8f43bc75e (ovn-cluster)
    port ovn-cluster-ovn-default
        mac: "36:0c:84:c9:23:d1"
        networks: ["10.16.0.1/16"]
    port ovn-cluster-join
        mac: "ee:9e:29:f0:57:df"
        networks: ["100.64.0.1/16"]

root@kf1:~# kubectl ko nbctl show ovn-default 
switch 36bcf45e-63a3-48f4-b60f-0bee0ab9fb96 (ovn-default)
    port ovn-default-ovn-cluster
        type: router
        router-port: ovn-cluster-ovn-default
    port coredns-575999dc5b-d522m.kube-system
        addresses: ["aa:0a:7c:73:b9:86 10.16.0.4"]
    port kube-ovn-pinger-bb5c4.kube-system
        addresses: ["ae:31:8d:cb:42:02 10.16.0.11"]
    port kube-ovn-pinger-ln924.kube-system
        addresses: ["d6:87:97:40:0b:21 10.16.0.5"]
    port coredns-575999dc5b-lcphj.kube-system
        addresses: ["36:ba:27:08:22:3f 10.16.0.2"]

root@kf1:~# kubectl ko nbctl show join 
switch dc21dda6-7741-48d5-ab24-044a8793d92d (join)
    port join-ovn-cluster
        type: router
        router-port: ovn-cluster-join
    port node-kf2
        addresses: ["42:10:5a:62:b7:a9 100.64.0.2"]
    port node-kf1
        addresses: ["9e:55:09:21:ff:77 100.64.0.3"]        
```

## 自定义VPC网络**：扩展与隔离的灵活设计**

在默认 VPC 网络之外，**kube-ovn** 支持创建自定义 VPC 网络。与默认网络相比，自定义 VPC 网络的最大特点是完全隔离：

- **逻辑架构**

每个自定义 VPC 都拥有独立于 ovn-cluster 的逻辑路由器，并可以自定义不同的网段。

- **独立性与拓扑一致性**

自定义 VPC 的网络拓扑设计与默认 VPC 网络相同，但逻辑上是完全分离的。

## 外部网络**：自定义 VPC 的网关设计**

虽然自定义 VPC 的网段可以自由配置，但其默认状态下与默认网络、主机网络完全隔离，仅支持自身网段内的通信。为了扩展其能力，kube-ovn 设计了 **自定义 VPC 网关**，实现自定义 VPC 对集群外部的访问。

**网关配置流程：**

1. **NAT 网关 Pod 的创建**

在目标 VPC 的子网下，创建一个 nat-gw Pod 实例。

1. **外部网卡处理**

    - 如果外部网络指定了 VLAN，需要为物理网卡创建 VLAN 子网卡。
    - 使用 macvlan 为物理网卡生成子网卡。
    - 通过 Multus-CNI 的多网卡功能，将生成的子网卡接入到 nat-gw 网关 Pod 的网络命名空间，作为其第二张网卡。

2. **路由规则配置**

   自定义vpc的逻辑路由器会有路由规则`ovn-nbctl lr-route-add router "0.0.0.0/0" 54.54.0.254`，该规则指示所有未匹配其他路由的流量发送到 nat-gw 网关 Pod。

最后我们就可通过在网关pod里下发特定的路由、iptabels 规则，实现EIP、FIP等功能。

## Underlay 网络**：直接对接物理网络**

underlay网络没有VPC概念，kube-ovn设计了`ProviderNetwork` 提供了主机网卡到物理网络映射的抽象；`Vlan` 提供了vlan和`ProviderNetwork`的绑定。最后创建 `subnet`只需要绑定`vlan`资源即可；

**网络拓扑**

- **网卡接入与桥接**

Underlay 业务网卡接入到相应的网桥设备，同时该网桥设备与 br-int 网桥互联。

- **逻辑表现**

每个 Underlay 子网对应一个逻辑交换机，交换机的端口连接 Pod 的网络设备和本地业务网卡网络。

```bash
[root@node152 ~]# kubectl ko vsctl node152    show    
    Bridge br-ens4f0
        Port br-ens4f0
            Interface br-ens4f0
                type: internal
        Port patch-localnet.net-9e0crrxa-to-br-int
            Interface patch-localnet.net-9e0crrxa-to-br-int
                type: patch
                options: {peer=patch-br-int-to-localnet.net-9e0crrxa}
        Port ens4f0
            trunks: [0, 134, 300]
            Interface ens4f0
        Port patch-localnet.net-dn74fi18-to-br-int
            Interface patch-localnet.net-dn74fi18-to-br-int
                type: patch
                options: {peer=patch-br-int-to-localnet.net-dn74fi18}
    Bridge br-int
        fail_mode: secure
        datapath_type: system
        Port patch-br-int-to-localnet.net-dn74fi18
            Interface patch-br-int-to-localnet.net-dn74fi18
                type: patch
                options: {peer=patch-localnet.net-dn74fi18-to-br-int}
        Port patch-br-int-to-localnet.net-9e0crrxa
            Interface patch-br-int-to-localnet.net-9e0crrxa
                type: patch
                options: {peer=patch-localnet.net-9e0crrxa-to-br-int}  
        ...      
```

# 总结

自定义VPC主要用于多租户场景，强调灵活性和隔离性。和主机网络无法互通的特性使得自定义VPC无法支持例如节点和 Pod 互访，NodePort 功能，基于网络访问的健康检查和 DNS 能力。下期会探讨引入ebpf技术来加强自定义VPC的能力。
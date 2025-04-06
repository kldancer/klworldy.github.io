+++
title = '探究kube-ovn 逻辑网络数据架构'
date = 2025-04-06T10:56:00+08:00
draft = false
tags= ["网络","Linux kernel","kube-ovn"]
categories= ["网络"]
author= ["kl"]
+++

## 同vpc、subnet不同node的pod流量

同vpc子网不同node下pod在OVN逻辑层面：

在vpc逻辑路由器下，存在子网逻辑交换机，pod在该交换机下均有对应的逻辑端口，逻辑端口绑定信息中也记录着端口所在的节点`chassis` ，逻辑数据路径，逻辑端口编号等信息。以上信息在ovs中转化为对应流表。从而支持同vpc子网下pod的跨节点流量。

```bash
root@l1:~# kubectl get pod -owide 
NAME                       READY   STATUS              RESTARTS   AGE   IP         NODE   NOMINATED NODE   READINESS GATES
nginx-o-2-6bf6f745-ssdqp   1/1     Running             0          29m   11.0.1.3   l2     <none>           <none>
nginx-o-658f89b7f-zvvf5    1/1     Running             0          42m   11.0.1.2   l1     <none>           <none>

root@l1:~# kubectl get vpc 
NAME          ENABLEEXTERNAL   ENABLEBFD   STANDBY   SUBNETS                              EXTRAEXTERNALSUBNETS   NAMESPACES
ovn-cluster   false            false       true      ["ovn-default","u-subnet1","join"]                          
test-vpc-1    false            false       true      ["net1"]                                                    ["ns1"]
test-vpc-2    false            false       true      ["net2"]                                                    ["ns2"]
test-vpc-3    false            false       true      ["net3"]                                                    ["ns3"]
test-vpc-4    false            false       true      ["net4"]                                                    ["ns4"]
root@l1:~# kubectl get subnets.kubeovn.io net1 
NAME   PROVIDER               VPC          PROTOCOL   CIDR          PRIVATE   NAT     DEFAULT   GATEWAYTYPE   V4USED   V4AVAILABLE   V6USED   V6AVAILABLE   EXCLUDEIPS     U2OINTERCONNECTIONIP
net1   net1.kube-system.ovn   test-vpc-1   IPv4       11.0.1.0/24   false     false   false     distributed   2        251           0        0             ["11.0.1.1"]   

root@l1:~# kubectl ko nbctl show test-vpc-1 
router 6ebf2ebb-6a59-4a54-a3e1-ab7a4bd07846 (test-vpc-1)
    port test-vpc-1-net1
        mac: "c2:f1:67:d8:cb:a5"
        networks: ["11.0.1.1/24"]
root@l1:~# kubectl ko nbctl show net1
switch 412b1b0d-26fe-4dc6-969d-ee0035f0164b (net1)
    port net1-test-vpc-1
        type: router
        router-port: test-vpc-1-net1
    port nginx-o-658f89b7f-zvvf5.default
        addresses: ["62:90:be:72:b5:6b 11.0.1.2"]
    port nginx-o-2-6bf6f745-ssdqp.default
        addresses: ["4a:9e:88:41:bf:25 11.0.1.3"]
        
root@l1:~# kubectl ko sbctl list  Port_Binding  nginx-o-658f89b7f-zvvf5.default 
_uuid               : 35d165d4-9d25-47cb-8106-c9b9df469cf7
additional_chassis  : []
additional_encap    : []
chassis             : 9643d87b-87c9-4deb-a78d-b96ad2d9f619
datapath            : e9e83e8e-7f07-41a3-880c-071e65203150
encap               : []
external_ids        : {associated_sg_default-securitygroup="false", ls=net1, pod="default/nginx-o-658f89b7f-zvvf5", vendor=kube-ovn}
gateway_chassis     : []
ha_chassis_group    : []
logical_port        : nginx-o-658f89b7f-zvvf5.default
mac                 : ["62:90:be:72:b5:6b 11.0.1.2"]
mirror_rules        : []
nat_addresses       : []
options             : {}
parent_port         : []
port_security       : []
requested_additional_chassis: []
requested_chassis   : []
tag                 : []
tunnel_key          : 2
type                : ""
up                  : true
virtual_parent      : []

root@l1:~# kubectl ko sbctl list  Port_Binding  nginx-o-2-6bf6f745-ssdqp.default
_uuid               : f1b99dea-e149-4bd9-8f26-6096c0934423
additional_chassis  : []
additional_encap    : []
chassis             : fce4d24e-3ac7-48b4-a249-28f4b1f75516
datapath            : e9e83e8e-7f07-41a3-880c-071e65203150
encap               : []
external_ids        : {associated_sg_default-securitygroup="false", ls=net1, pod="default/nginx-o-2-6bf6f745-ssdqp", vendor=kube-ovn}
gateway_chassis     : []
ha_chassis_group    : []
logical_port        : nginx-o-2-6bf6f745-ssdqp.default
mac                 : ["4a:9e:88:41:bf:25 11.0.1.3"]
mirror_rules        : []
nat_addresses       : []
options             : {}
parent_port         : []
port_security       : []
requested_additional_chassis: []
requested_chassis   : []
tag                 : []
tunnel_key          : 3
type                : ""
up                  : true
virtual_parent      : []

root@l1:~# kubectl ko sbctl list Chassis 
_uuid               : fce4d24e-3ac7-48b4-a249-28f4b1f75516
encaps              : [81ecd7fa-0459-496a-bfb0-dea4acd489e8]
external_ids        : {vendor=kube-ovn}
hostname            : l2
name                : "1da489fa-8291-47c2-a744-576e79192e53"
nb_cfg              : 0
other_config        : {ct-no-masked-label="true", datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", mac-binding-timestamp="true", ovn-bridge-mappings="ens192:br-ens192", ovn-chassis-mac-mappings="ens192:f6:c5:21:84:16:bc", ovn-cms-options="", ovn-ct-lb-related="true", ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-timeout-ms="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

_uuid               : 9643d87b-87c9-4deb-a78d-b96ad2d9f619
encaps              : [516d6784-8089-4f54-a676-8594cdc0f6b0]
external_ids        : {vendor=kube-ovn}
hostname            : l1
name                : "018d58f8-8aab-4a2f-9f90-0bc95ad422fb"
nb_cfg              : 0
other_config        : {ct-no-masked-label="true", datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", mac-binding-timestamp="true", ovn-bridge-mappings="ens192:br-ens192", ovn-chassis-mac-mappings="ens192:e2:a8:a0:8d:e0:7e", ovn-cms-options="", ovn-ct-lb-related="true", ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-timeout-ms="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

root@l1:~# kubectl ko sbctl list Encap
_uuid               : 516d6784-8089-4f54-a676-8594cdc0f6b0
chassis_name        : "018d58f8-8aab-4a2f-9f90-0bc95ad422fb"
ip                  : "192.168.80.128"
options             : {csum="true"}
type                : geneve

_uuid               : 81ecd7fa-0459-496a-bfb0-dea4acd489e8
chassis_name        : "1da489fa-8291-47c2-a744-576e79192e53"
ip                  : "192.168.80.129"
options             : {csum="true"}
type                : geneve

root@l1:~# kubectl ko sbctl list Datapath_Binding 
_uuid               : e9e83e8e-7f07-41a3-880c-071e65203150
external_ids        : {logical-switch="412b1b0d-26fe-4dc6-969d-ee0035f0164b", name=net1}
load_balancers      : []
tunnel_key          : 8
        
```

**在ovs中可以过滤到该条规则，其中信息存在以下要点：**

- cookie：为nginx-o-2 Port_Binding 的uuid前缀
- reg15：用于存储数据包的**目标逻辑端口的 OFPort (OpenFlow Port) 编号**。这个编号是在 OVN 的逻辑层面确定的。具体查看`tunnel_key`值
- metadata：OVN 用它来存储数据包当前所处的**逻辑数据路径 (Logical Datapath) 的标识符 (ID)**。这个 ID 唯一地标识了一个逻辑交换机 (Logical Switch) 或逻辑路由器 (Logical Router)。

```bash
root@l1:/kube-ovn# ovs-ofctl dump-flows br-int | grep 'actions=.*output:2' 
cookie=0xf1b99dea, duration=4188.293s, table=39, n_packets=58, n_bytes=5104, idle_age=119, priority=100,
	reg15=0x3,
	metadata=0x8 actions=
		load:0x8->NXM_NX_TUN_ID[0..23],set_field:0x3->tun_metadata0,move:NXM_NX_REG14[0..14]->NXM_NX_TUN_METADATA0[16..30],output:2,resubmit(,40)
```

>
> 💡排查流表Tips
>
> ```shell
> # 1. 先查看隧道OVS物理端口编号
> root@l1:/kube-ovn# ovs-ofctl show br-int | grep ovn-
>  2(ovn-1da489-0): addr:82:b6:65:de:f9:b9
> # 2. 再筛选转发到目的端口的动作
> ovs-ofctl dump-flows br-int | grep 'actions=.*output:2'
> ```



>
> 💡 OVN逻辑端口、OVS物理端口
>
> 在这个例子中，pod的逻辑端口号为3，而在对于节点中ovs-ofctl show br-int得到的对应端口号为7，为什么不一样呢？
>
> - 在**发送节点l1**上：
>
>   - **ovn-controller**知道目标逻辑端口**3**在远程节点**l2**。
>   - 它生成 OVS 流表，匹配**reg15=3**，并将逻辑端口**3**编码到 Geneve 隧道元数据中，然后通过代表隧道的 OVS 物理端口（比如**l1**上的端口**2**）发送出去。
>
> - 在**接收节点l2**上：
>
>   - **ovs-vswitchd**收到 Geneve 包，解封装，并将隧道元数据（包含目标逻辑端口**3**）提供给 OVN 处理流程。
>
>   - **ovn-controller**在**l2**上知道：OVN 逻辑端口**3**对应的是本地 OVS 接口**baf0a678f203_h**。
>
>   - **ovn-controller**也知道：本地**ovs-vswitchd**给接口**baf0a678f203_h**分配的物理端口号是**7**。
>
>   - 因此，**ovn-controller**会生成 OVS 流表，在处理完该数据包的逻辑流程（如 ACL、负载均衡等）后，对于需要发送给逻辑端口**3**的包，最终执行的动作是**output:7**。
>
>     ```bash
>     root@l2:/kube-ovn# ovs-ofctl dump-flows br-int | grep '0xf1b99dea' 
>     cookie=0xf1b99dea, duration=6354.389s, table=65, n_packets=66, n_bytes=5548, idle_age=2285, priority=100,reg15=0x3,metadata=0x8 actions=output:7
>     ```
>
> - **OVN Logical Port** 是 OVN 用来思考和传递 "要去哪个 Pod" 的**逻辑地址**。
>
> - **OVS Physical Port onl2** 是**l2**节点上的 OVS 用来实际把包从**br-int**扔给对应 Pod 的物理网卡的**本地门牌号**



## underlay 逻辑网络架构

```bash
root@l1:~# kubectl get pod  -owide 
NAME                       READY   STATUS              RESTARTS   AGE    IP         NODE   NOMINATED NODE   READINESS GATES
nginx-u-d76c9fdff-8dhnr    1/1     Running             0          127m   172.17.0.3 l1     <none>           <none>
nginx-u-2-686c945ddc-jb8kg 1/1     Running             0          5s     172.17.0.4 l2     <none>           <none>

root@l1:~/test# kubectl get subnets.kubeovn.io  u-subnet1
NAME        PROVIDER   VPC           PROTOCOL   CIDR            PRIVATE   NAT     DEFAULT   GATEWAYTYPE   V4USED   V4AVAILABLE   V6USED   V6AVAILABLE   EXCLUDEIPS       U2OINTERCONNECTIONIP
u-subnet1   ovn        ovn-cluster   IPv4       172.17.0.0/16   false     false   false     distributed   2        65531         0        0             ["172.17.0.1"]   

root@l1:~/test# kubectl ko nbctl show  u-subnet1 
switch b2dfe539-a6f2-4621-9419-e0b6c3b6be52 (u-subnet1)
    port nginx-u-d76c9fdff-8dhnr.default
        addresses: ["46:d5:8f:21:33:cc 172.17.0.3"]
    port nginx-u-2-686c945ddc-jb8kg.default
        addresses: ["36:0c:78:6c:49:71 172.17.0.4"]
    port localnet.u-subnet1
        type: localnet
        tag: 10
        addresses: ["unknown"]
        
root@l1:~/test# kubectl ko sbctl list  Port_Binding nginx-u-d76c9fdff-8dhnr.default 
_uuid               : 5889e864-c48e-40cd-a2be-0c5f0425c608
additional_chassis  : []
additional_encap    : []
chassis             : 9643d87b-87c9-4deb-a78d-b96ad2d9f619
datapath            : f09cb18d-6cc6-4be1-ae17-1bc8301b602f
encap               : []
external_ids        : {ls=u-subnet1, pod="default/nginx-u-d76c9fdff-8dhnr", vendor=kube-ovn}
gateway_chassis     : []
ha_chassis_group    : []
logical_port        : nginx-u-d76c9fdff-8dhnr.default
mac                 : ["46:d5:8f:21:33:cc 172.17.0.3"]
mirror_rules        : []
nat_addresses       : []
options             : {}
parent_port         : []
port_security       : []
requested_additional_chassis: []
requested_chassis   : []
tag                 : []
tunnel_key          : 3
type                : ""
up                  : true
virtual_parent      : []

root@l1:~/test# kubectl ko sbctl list  Port_Binding nginx-u-2-686c945ddc-jb8kg.default
_uuid               : 2950194d-1c9c-44d7-b81b-3ae1300aeac2
additional_chassis  : []
additional_encap    : []
chassis             : fce4d24e-3ac7-48b4-a249-28f4b1f75516
datapath            : f09cb18d-6cc6-4be1-ae17-1bc8301b602f
encap               : []
external_ids        : {ls=u-subnet1, pod="default/nginx-u-2-686c945ddc-jb8kg", vendor=kube-ovn}
gateway_chassis     : []
ha_chassis_group    : []
logical_port        : nginx-u-2-686c945ddc-jb8kg.default
mac                 : ["36:0c:78:6c:49:71 172.17.0.4"]
mirror_rules        : []
nat_addresses       : []
options             : {}
parent_port         : []
port_security       : []
requested_additional_chassis: []
requested_chassis   : []
tag                 : []
tunnel_key          : 4
type                : ""
up                  : true
virtual_parent      : []    

root@l1:~/test# kubectl ko sbctl list Datapath_Binding  f09cb18d-6cc6-4be1-ae17-1bc8301b602f 
_uuid               : f09cb18d-6cc6-4be1-ae17-1bc8301b602f
external_ids        : {logical-switch="b2dfe539-a6f2-4621-9419-e0b6c3b6be52", name=u-subnet1}
load_balancers      : []
tunnel_key          : 12

root@l1:/kube-ovn# ovs-vsctl show 
5013150e-5cfe-47e5-8cfc-01f9d02d20c2
    Bridge br-int
        fail_mode: secure
        datapath_type: system
        Port mirror0
            Interface mirror0
                type: internal
        Port e8d8e67b335b_h
            Interface e8d8e67b335b_h
        Port "518cf0f85dde_h"
            Interface "518cf0f85dde_h"
        Port ovn-1da489-0
            Interface ovn-1da489-0
                type: geneve
                options: {csum="true", key=flow, remote_ip="192.168.80.129"}
        Port patch-br-int-to-localnet.u-subnet1
            Interface patch-br-int-to-localnet.u-subnet1
                type: patch
                options: {peer=patch-localnet.u-subnet1-to-br-int}
        Port ovn0
            Interface ovn0
                type: internal
        Port br-int
            Interface br-int
                type: internal
        Port "6161fdbc64b9_h"
            Interface "6161fdbc64b9_h"
    Bridge br-ens192
        Port patch-localnet.u-subnet1-to-br-int
            Interface patch-localnet.u-subnet1-to-br-int
                type: patch
                options: {peer=patch-br-int-to-localnet.u-subnet1}
        Port ens192
            trunks: [0, 10]
            Interface ens192
        Port br-ens192
            Interface br-ens192
                type: internal
    ovs_version: "3.1.6"
```

在 OVN 的**逻辑层面**，`localnet`类型的端口代表了一个 OVN 逻辑交换机（这里是`u-subnet1`）与节点**本地物理网络**之间的连接点。它告诉 OVN：“属于**u-subnet1**这个逻辑网络的流量，如果要与外部物理网络交互，应该通过这个`localnet`端口进出。”

**指定物理网络连接:**

- **type：localnet**: 表明这是一个连接到本地网络的特殊逻辑端口。
- **tag: 10**：它指示 OVN：
    - 当`u-subnet1`逻辑交换机内的流量（例如来自 Pod**172.17.0.3**）需要通过`localnet.u-subnet1`端口**流向物理网络**时，这些流量在离开 OVS （最终从**ens192**出去）时必须被打上**VLAN Tag 10**。
    - 当带有**VLAN Tag 10**的流量从物理网络（通过**ens192**）**进入 OVS**时，应该被引导到`localnet.u-subnet1`端口，并进入**u-subnet1**逻辑交换机进行处理（此时 VLAN Tag 通常会被 OVS 剥离）。
- **addresses: ["unknown"]**: 对于`localnet`端口，通常设置为**unknown**，因为 OVN 不需要学习连接到物理网络侧的设备的 MAC 地址，这个任务由物理网络设备或 OVS 的 Provider Bridge (**br-ens192**) 的普通 L2 学习机制完成。

当nginx-u-d76c9fdff-8dhnr要访问 VLAN 10 中跨节点的另一个设备（nginx-u-2-686c945ddc-jb8kg）或外部网络：

1. **Pod 发包:**Pod 发出数据包，通过 veth pair 进入**l1**节点的**br-int**。
2. **br-intOVN 逻辑处理:**
    - OVN 流表识别出数据包来自逻辑端口**nginx-u-...**，属于逻辑交换机**u-subnet1**(**metadata**可能被设置为**u-subnet1**的 Datapath ID)。
    - OVN 进行 ACL 检查 (NetworkPolicy)。
    - OVN 查找目标 MAC/IP。如果目标不在**u-subnet1**的已知 Pod 中（即需要发往物理网络），OVN 逻辑流表会决定将数据包导向**u-subnet1**的出口——**逻辑端口localnet.u-subnet1**。
3. **通过patch口连接:**
    - **ovn-controller**将”发送到逻辑端口`localnet.u-subnet1`”这个逻辑动作，翻译为 OVS 物理动作：将数据包发送到`patch-br-int-to-localnet.u-subnet1`这个 OVS 端口。
4. **进入br-ens192：**数据包通过 patch link 到达**br-ens192**上的`patch-localnet.u-subnet1-to-br-int`端口。
5. **br-ens192处理与 VLAN Tagging:**
    - **br-ens192**收到来自 patch 口的数据包。因为它知道这个 patch 口与 OVN 中配置了**tag: 10**的**localnet.u-subnet1**相关联，所以**br-ens192**会将这个数据包视为属于 VLAN 10。
    - **br-ens192**将数据包从物理端口**ens192**发出。因为**ens192**配置为 trunk 允许 VLAN 10，并且 OVS 知道这个包属于 VLAN 10，所以 OVS 会在发送前给数据包**打上 VLAN 10 的 Tag**。
6. **物理网络传输：** 带 VLAN Tag 10 的数据包在物理网络中传输。

>
> 💡 Underlay 和 Overlay 网络在 OVN/Kube-OVN 中处理方式的不同。
>
> - overlay：VPC有对应的逻辑路由器，逻辑路由器上的端口记录着连接逻辑交换机（子网）的网段信息。
> - underlay：只有逻辑交换机，Underlay 网络的核心设计是 Pod 直接使用底层物理网络的 IP，其**默认网关和路由决策主要由外部物理网络设备（物理路由器）负责**，而不是由 OVN 的逻辑路由器来处理。OVN 的角色更侧重于将 Pod 的 veth 接口正确地桥接到对应的物理网络（或 VLAN）上。
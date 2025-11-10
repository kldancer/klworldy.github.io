+++
title = '探索SDN实操_05_sdn Ovn实操'
date = 2025-08-10T15:49:04+08:00
draft = false
tags= ["网络","sdn"]
categories= ["网络"]
author= ["kl"]
+++

# 前言

### OVN 架构核心组件和角色

OVN 是 Open vSwitch (OVS) 的一个扩展，它提供了一个更高级别的、声明式的接口来定义虚拟网络，并自动将这些定义转换为底层的 OpenFlow 规则。其核心思想是提供一个平台，允许用户通过逻辑网络概念来操作网络，而无需直接与 OpenFlow 规则交互。

OVN 的核心组件包括：

1. **OVN 北向数据库 (OVN Northbound DB / OVN-NBDB)**：
    - **角色**：这是 OVN 的**控制平面**接口，也是网络管理员或云管理平台 (如 OpenStack, Kubernetes) 与 OVN 交互的地方。
    - **内容**：存储逻辑网络模型的高级抽象，例如：
        - 逻辑交换机 (Logical Switches)
        - 逻辑路由器 (Logical Routers)
        - 逻辑端口 (Logical Ports)
        - ACLs (Access Control Lists)
        - NAT 规则
        - QoS 策略等。
    - **特点**：它不关心物理网络的细节，只关注“期望的状态”。所有的配置都通过 ovn-nbctl 工具或北向 API (例如 OVN Kubernetes 集成中的 K8s API) 写入到这个数据库。
2. **ovn-northd 守护进程**：
    - **角色**：OVN 的**中央控制逻辑**。它持续监控 OVN-NBDB 的变化。
    - **功能**：作为 NBDB 和 SBDB 之间的翻译器。当 NBDB 中的逻辑配置发生变化时，ovn-northd 会将这些高级逻辑概念翻译成更具体、但仍是抽象的“逻辑流” (Logical Flows) 和其他信息，并将这些信息写入 OVN 南向数据库。它进行复杂的路由计算、ACL 评估等，以决定数据包在逻辑网络中应该如何转发。
3. **OVN 南向数据库 (OVN Southbound DB / OVN-SBDB)**：
    - **角色**：这是 OVN 的**数据平面**抽象。
    - **内容**：存储由 ovn-northd 翻译而来的“逻辑流”（Logical Flows），以及物理网络拓扑信息（如哪个 hypervisor 上运行着哪些逻辑端口）。
        - Chassis 表：记录每个 hypervisor (物理节点) 的信息。
        - Port_Binding 表：记录逻辑端口绑定到哪个物理节点和哪个 OVS 端口。
        - Logical_Flow 表：存储由 ovn-northd 生成的、描述数据包在逻辑网络中如何转发的规则（类似于 OpenFlow 流规则，但仍是抽象的逻辑层）。
    - **特点**：它仍然不直接包含 OpenFlow 规则，而是包含足够的信息让底层的 ovn-controller 知道如何生成 OpenFlow 规则。
4. **ovn-controller 守护进程**：
    - **角色**：在每个**计算节点 (hypervisor)** 上运行的代理。它是 OVN 数据平面的执行者。
    - **功能**：
        1. 监控 OVN-SBDB，获取与当前计算节点相关的逻辑流和端口绑定信息。
        2. 将这些逻辑流**实时翻译成 OpenFlow 规则**，并下发到本地 OVS 实例。
        3. 管理本地 OVS 网桥（通常是 br-int，一个集成网桥），创建和配置连接到虚拟机的 OVS 端口 (VIF)。
        4. 向 SBDB 报告本节点的物理端口信息（例如，哪个 OVS 端口连接到哪个 VM）。
    - **特点**：负责将抽象的逻辑网络最终具象化为物理的 OpenFlow 规则，从而控制数据包的转发。

### OVN 工作流总结

1. 管理员/云平台通过 ovn-nbctl 或 API 配置 **OVN-NBDB** (定义逻辑交换机、逻辑端口等)。
2. ovn-northd 监听 **OVN-NBDB** 的变化，将其翻译成逻辑流，并写入 **OVN-SBDB**。
3. 每个计算节点上的 ovn-controller 监听 **OVN-SBDB** 中与自己相关的逻辑流和端口绑定。
4. ovn-controller 将这些逻辑流转换为具体的 **OpenFlow 规则**，并下发到本地 OVS 实例，同时配置 OVS 端口。
5. 数据包在 OVS 网桥上根据这些 OpenFlow 规则进行转发。

# **实践一：OVN 环境搭建与操作**

## 1. 安装OVN

```bash
# 安装 OVN (Fedora/CentOS)
# 确保如果ovn-nbctl, ovn-sbctl, ovn-northd, ovn-controller命令存在
sudo yum install -y openvswitch ovn ovn-central ovn-host

# 启动 ovsdb-server 和 ovn-northd (如果它们没有自动启动的话)
# 通常安装后会自动启动，但为了确保，可以检查状态
sudo systemctl start ovsdb-server
sudo systemctl enable ovsdb-server
sudo systemctl start ovn-northd
sudo systemctl enable ovn-northd

# 检查 OVN 数据库状态
sudo ovn-nbctl show
sudo ovn-sbctl show
```

## 2.  **启动 ovn-controller 并连接虚拟机 (模拟)**

```bash
echo "2. Setting up OVS for ovn-controller and simulating VMs..."

# 创建 OVS 集成网桥 br-int (如果还没有)
# ovn-controller 会自动管理这个网桥
sudo ovs-vsctl add-br br-int

# 启动 ovn-controller (如果它没有自动启动的话)
# ovn-controller 需要知道 OVN-SBDB 的地址,
# 需要将 OVS 节点的 external_ids:ovn-encap-type 和 external_ids:ovn-encap-ip 配置正确，以便 ovn-controller 知道如何与 OVN Central 通信。
sudo ovs-vsctl set Open_vSwitch . external_ids:ovn-remote=unix:/run/ovn/ovnsb_db.sock
sudo ovs-vsctl set Open_vSwitch . external_ids:ovn-encap-type=geneve
sudo ovs-vsctl set Open_vSwitch . external_ids:ovn-encap-ip=192.168.1.139
# 
sudo systemctl start ovn-controller
sudo systemctl enable ovn-controller
sudo journalctl -u ovn-controller -b -f

# 模拟 vm1 (对应 lp-web1)
sudo ip netns add vm1
sudo ip link add vm1-vif type veth peer name vm1-br-int
sudo ip link set vm1-vif netns vm1
sudo ip netns exec vm1 ip link set vm1-vif up
sudo ip netns exec vm1 ip addr add 10.0.0.10/24 dev vm1-vif # 与lp-web1的IP一致
sudo ip netns exec vm1 ip link set lo up

sudo ip link set vm1-br-int up
sudo ovs-vsctl add-port br-int vm1-br-int -- set Interface vm1-br-int external_ids:iface-id=lp-web1
# 这里的 external_ids:iface-id=lp-web1 是关键，它告诉 ovn-controller 这个 OVS 端口对应哪个逻辑端口

# 模拟 vm2 (对应 lp-web2)
sudo ip netns add vm2
sudo ip link add vm2-vif type veth peer name vm2-br-int
sudo ip link set vm2-vif netns vm2
sudo ip netns exec vm2 ip link set vm2-vif up
sudo ip netns exec vm2 ip addr add 10.0.0.11/24 dev vm2-vif # 与lp-web2的IP一致
sudo ip netns exec vm2 ip link set lo up

sudo ip link set vm2-br-int up
sudo ovs-vsctl add-port br-int vm2-br-int -- set Interface vm2-br-int external_ids:iface-id=lp-web2
# 同样的 iface-id 绑定

echo "VMs simulated and connected to br-int. Waiting for ovn-controller to configure..."
# 等待 ovn-controller 几秒钟，让它处理配置

```

## 3. **使用 ovn-nbctl 手动创建一个逻辑交换机和几个逻辑端口**

```bash
echo "1. Creating logical switch and logical ports in OVN-NBDB..."

sudo ip netns exec vm1 ip link show vm1-vif # 看 MAC 地址
sudo ip netns exec vm2 ip link show vm2-vif # 看 MAC 地址
# 4e:52:1b:26:54:b5
# 9a:02:98:21:d6:53

# 创建逻辑交换机 ls-web
sudo ovn-nbctl ls-add ls-web

# 为 ls-web 创建两个逻辑端口
# lp-web1 with MAC 00:00:00:00:00:01 and IP 10.0.0.10/24
sudo ovn-nbctl lsp-add ls-web lp-web1
sudo ovn-nbctl lsp-set-addresses lp-web1 "4e:52:1b:26:54:b5 10.0.0.10"

# lp-web2 with MAC 00:00:00:00:00:02 and IP 10.0.0.11/24
sudo ovn-nbctl lsp-add ls-web lp-web2
sudo ovn-nbctl lsp-set-addresses lp-web2 "9a:02:98:21:d6:53 10.0.0.11"

# 查看 OVN-NBDB 中的逻辑网络配置
echo "OVN-NBDB logical network configuration:"
sudo ovn-nbctl show
```

## 4. **观察 ovn-controller 自动生成的 OVSDB 记录和 OpenFlow 流表**

```bash
# 观察 OVSDB Port_Binding 表：
echo "OVN-SBDB Port_Binding table (should show lp-web1 and lp-web2 bound to this chassis):"
# sudo ovn-sbctl list Chassis
# sudo ovn-sbctl list Port_Binding
# sudo ovn-sbctl list Logical_Flow 
sudo ovn-sbctl show

# 观察 OVSDB br-int 端口信息：
echo "OVS 'br-int' port configuration:"
sudo ovs-vsctl show
sudo ovs-ofctl show br-int

# 观察 OpenFlow 流表：
echo "OpenFlow flows on 'br-int' (generated by ovn-controller):"
sudo ovs-ofctl dump-flows br-int
```

## 5. 测试联通性

```bash
echo "Testing connectivity between vm1 and vm2..."
sudo ip netns exec vm1 ping -c 3 10.0.0.11
```

## 6. **使用 ovn-trace 工具模拟数据包路径**

### **模拟 vm1 (lp-web1) 发送 ARP 请求给 vm2 (lp-web2) 的过程：**

1. ls-web 交换机对入站 ARP 包进行端口安全检查。
2. 识别为广播流量，并将出端口设置为 _MC_flood 多播组。
3. 数据包进入 _MC_flood 多播组，进行泛洪。
4. 数据包不会被发送回 lp-web1 (源端口)。
5. 数据包被发送到 lp-web2，经过出站端口安全检查后，最终成功转发

```bash
echo "4. Tracing an ARP request from lp-web1 to lp-web2 (logical flow)..."
# ovn-trace 参数：
# -v: 详细输出
# <logical-datapath>: 逻辑交换机的UUID或名称 (ls-web)
# 'inport="lp-web1", eth.src=00:00:00:00:00:01, eth.dst=ff:ff:ff:ff:ff:ff, eth.type=0x806, arp.tpa=10.0.0.11'
# 解释：来自lp-web1，源MAC 00:00:00:00:00:01，广播目的MAC，ARP类型，目标IP 10.0.0.11
sudo ovn-trace -v ls-web 'inport=="lp-web1" && eth.src==4e:52:1b:26:54:b5 && eth.dst==ff:ff:ff:ff:ff:ff && eth.type==0x806 && arp.tpa==10.0.0.11'

# 结果
# arp,reg14=0x1,vlan_tci=0x0000,dl_src=4e:52:1b:26:54:b5,dl_dst=ff:ff:ff:ff:ff:ff,arp_spa=0.0.0.0,arp_tpa=10.0.0.11,arp_op=0,arp_sha=00:00:00:00:00:00,arp_tha=00:00:00:00:00:00
ingress(dp="ls-web", inport="lp-web1")
--------------------------------------
 0. ls_in_check_port_sec (northd.c:9399): 1, priority 50, uuid 1d79ecbe
    reg0[15] = check_in_port_sec();
    next;
 5. ls_in_pre_lb (northd.c:6170): eth.mcast, priority 110, uuid ca90f0a4
    next;
28. ls_in_l2_lkup (northd.c:10051): eth.mcast, priority 70, uuid 78deefaf
    outport = "_MC_flood";
    output;

multicast(dp="ls-web", mcgroup="_MC_flood")
-------------------------------------------

    egress(dp="ls-web", inport="lp-web1", outport="lp-web1")
    --------------------------------------------------------
            /* omitting output because inport == outport && !flags.loopback */

    egress(dp="ls-web", inport="lp-web1", outport="lp-web2")
    --------------------------------------------------------
         3. ls_out_pre_lb (northd.c:6172): eth.mcast, priority 110, uuid a87097a6
            next;
        11. ls_out_check_port_sec (northd.c:5863): eth.mcast, priority 100, uuid becf5915
            reg0[15] = 0;
            next;
        12. ls_out_apply_port_sec (northd.c:5874): 1, priority 0, uuid 5e874079
            output;
            /* output to "lp-web2", type "" */

```

### **模拟 vm1 (lp-web1) 发送 ICMP 请求给 vm2 (lp-web2) 的过程 (在 ARP 之后)：**

1. ls-web 对入站数据包执行端口安全检查。
2. 由于是单播流量，ls-web 在其内部查找 MAC 地址表，根据目的 MAC 地址 9a:02:98:21:d6:53 确定出站端口是 lp-web2。
3. 数据包被转发到 lp-web2 进行出站处理。
4. ls-web 对出站数据包执行端口安全检查。
5. 最终，数据包成功从 lp-web2 发送出去，到达目标设备 10.0.0.11 (MAC 9a:02:98:21:d6:53)。

```bash
echo "Tracing an ICMP request from lp-web1 to lp-web2 (logical flow, assuming MACs are learned)..."
# 假设 MAC 地址已经学习到
sudo ovn-trace -v ls-web \
'inport == "lp-web1" && eth.src == 4e:52:1b:26:54:b5 && eth.dst == 9a:02:98:21:d6:53 && ip4.src == 10.0.0.10 && ip4.dst == 10.0.0.11 && ip.proto == 1'

# 输出
# icmp,reg14=0x1,vlan_tci=0x0000,dl_src=4e:52:1b:26:54:b5,dl_dst=9a:02:98:21:d6:53,nw_src=10.0.0.10,nw_dst=10.0.0.11,nw_tos=0,nw_ecn=0,nw_ttl=0,nw_frag=no,icmp_type=0,icmp_code=0

# icmp,reg14=0x1,vlan_tci=0x0000,dl_src=4e:52:1b:26:54:b5,dl_dst=9a:02:98:21:d6:53,nw_src=10.0.0.10,nw_dst=10.0.0.11,nw_tos=0,nw_ecn=0,nw_ttl=0,nw_frag=no,icmp_type=0,icmp_code=0

ingress(dp="ls-web", inport="lp-web1")
--------------------------------------
 0. ls_in_check_port_sec (northd.c:9399): 1, priority 50, uuid 1d79ecbe
    reg0[15] = check_in_port_sec();
    next;
28. ls_in_l2_lkup (northd.c:10292): eth.dst == 9a:02:98:21:d6:53, priority 50, uuid 27ae7e1f
    outport = "lp-web2";
    output;

egress(dp="ls-web", inport="lp-web1", outport="lp-web2")
--------------------------------------------------------
11. ls_out_check_port_sec (northd.c:5866): 1, priority 0, uuid a6a2fdde
    reg0[15] = check_out_port_sec();
    next;
12. ls_out_apply_port_sec (northd.c:5874): 1, priority 0, uuid 5e874079
    output;
    /* output to "lp-web2", type "" */

```

## 7. 清理环境

```bash
echo "Cleaning up OVN and OVS environment..."

# 停止 ovn-controller 和 ovn-northd (如果需要完全停止 OVN)
sudo systemctl stop ovn-controller
sudo systemctl stop ovn-northd

# 清理 OVN-NBDB 中的逻辑配置
sudo ovn-nbctl -- --if-exists lsp-del lp-web1
sudo ovn-nbctl -- --if-exists lsp-del lp-web2
sudo ovn-nbctl -- --if-exists ls-del ls-web

# 删除 OVS 网桥
sudo ovs-vsctl -- --if-exists del-br br-int

# 删除网络命名空间
sudo ip netns del vm1 2>/dev/null
sudo ip netns del vm2 2>/dev/null

# 清理 veth 设备 (ovs-vsctl del-br 会自动处理连接到网桥的端口)
sudo ip link del vm1-vif 2>/dev/null
sudo ip link del vm2-vif 2>/dev/null
sudo ip link del vm1-br-int 2>/dev/null
sudo ip link del vm2-br-int 2>/dev/null

echo "Cleanup complete."
```

## 输出

```bash
root@t1:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:0e:9f:a2 brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 192.168.1.139/24 brd 192.168.1.255 scope global dynamic noprefixroute ens160
       valid_lft 78012sec preferred_lft 78012sec
    inet6 fe80::20c:29ff:fe0e:9fa2/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
19: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 0a:ad:42:9e:cf:cd brd ff:ff:ff:ff:ff:ff
20: br-int: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 06:ed:26:ee:44:8c brd ff:ff:ff:ff:ff:ff
21: vm1-br-int@if22: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UP group default qlen 1000
    link/ether 4a:68:03:6d:80:08 brd ff:ff:ff:ff:ff:ff link-netns vm1
    inet6 fe80::4868:3ff:fe6d:8008/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
23: vm2-br-int@if24: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UP group default qlen 1000
    link/ether 4e:50:8f:8d:7b:8c brd ff:ff:ff:ff:ff:ff link-netns vm2
    inet6 fe80::4c50:8fff:fe8d:7b8c/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
root@t1:~# sudo ovn-sbctl show
Chassis "c3743e5d-400b-45fc-960c-80d7c87a2739"
    hostname: t1
    Encap geneve
        ip: "192.168.1.139"
        options: {csum="true"}
    Port_Binding lp-web2
    Port_Binding lp-web1
root@t1:~# sudo ovs-vsctl show
31288f48-4746-4314-a541-834cfd827ede
    Bridge br-int
        fail_mode: secure
        datapath_type: system
        Port vm2-br-int
            Interface vm2-br-int
        Port br-int
            Interface br-int
                type: internal
        Port vm1-br-int
            Interface vm1-br-int
    ovs_version: "3.4.0-2.fc41"
root@t1:~# sudo ovs-ofctl show br-int
OFPT_FEATURES_REPLY (xid=0x2): dpid:000006ed26ee448c
n_tables:254, n_buffers:0
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
 1(vm1-br-int): addr:4a:68:03:6d:80:08
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 2(vm2-br-int): addr:4e:50:8f:8d:7b:8c
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 LOCAL(br-int): addr:06:ed:26:ee:44:8c
     config:     PORT_DOWN
     state:      LINK_DOWN
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
root@t1:~# sudo ovs-ofctl dump-flows br-int
root@t1:~# sudo ip netns exec vm1 ping -c 3 10.0.0.11
PING 10.0.0.11 (10.0.0.11) 56(84) 字节的数据。
64 字节，来自 10.0.0.11: icmp_seq=1 ttl=64 时间=0.576 毫秒
64 字节，来自 10.0.0.11: icmp_seq=2 ttl=64 时间=0.079 毫秒
^C
--- 10.0.0.11 ping 统计 ---
已发送 2 个包， 已接收 2 个包, 0% packet loss, time 1060ms
rtt min/avg/max/mdev = 0.079/0.327/0.576/0.248 ms

```
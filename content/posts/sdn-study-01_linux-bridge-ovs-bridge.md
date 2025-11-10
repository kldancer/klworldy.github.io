+++
title = '探索SDN实操_01_Linux bridge & OVS bridge'
date = 2025-07-15T15:08:26+08:00
draft = false
tags= ["网络","sdn"]
categories= ["网络"]
author= ["kl"]
+++

# 前言

准备开个新坑吧，系统的过一遍SDN领域的相关技术，记录一些实验实操。

> 💡手动创建网络命名空间，用veth pair连接它们，并通过Linux bridge或OVS bridge将它们互联。尝试配置IP地址并ping通。


## **使用 Linux Bridge 互联**

```bash
# 1. 创建网络命名空间
sudo ip netns add ns1
sudo ip netns add ns2

# 2. 创建 veth pair
sudo ip link add veth1 type veth peer name veth1_br
sudo ip link add veth2 type veth peer name veth2_br

# 3. 将 veth pair 的一端放入命名空间
sudo ip link set veth1 netns ns1
sudo ip link set veth2 netns ns2

# 4. 激活命名空间内的 veth 设备
sudo ip netns exec ns1 ip link set veth1 up
sudo ip netns exec ns2 ip link set veth2 up

# 5. 配置命名空间内的 IP 地址
sudo ip netns exec ns1 ip addr add 192.168.1.10/24 dev veth1
sudo ip netns exec ns2 ip addr add 192.168.1.20/24 dev veth2

# 6. 创建 Linux Bridge
sudo brctl addbr mybridge
sudo ip link set mybridge up

# 7. 将 veth pair 的另一端连接到 Linux Bridge
sudo brctl addif mybridge veth1_br
sudo brctl addif mybridge veth2_br

# 8. 激活连接到 Bridge 的 veth 设备
sudo ip link set veth1_br up
sudo ip link set veth2_br up

# 9. 测试连通性
sudo ip netns exec ns1 ping -c 3 192.168.1.20

# 10. 清理
sudo ip netns del ns1
sudo ip netns del ns2
sudo ip link del mybridge
sudo ip link del veth1_br # veth1_br 会自动删除 veth1
sudo ip link del veth2_br # veth2_br 会自动删除 veth2
```

![sdn-study-01_linux-bridge-ovs-bridge_1.png](/img/sdn-study-01_linux-bridge-ovs-bridge_1.png)

结果如下：
```bash
root@t1:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:f6:6f:4c brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 192.168.31.172/24 brd 192.168.31.255 scope global dynamic noprefixroute ens160
       valid_lft 42118sec preferred_lft 42118sec
    inet6 fd00:6868:6868::e3f/128 scope global dynamic noprefixroute 
       valid_lft 42121sec preferred_lft 42121sec
    inet6 fd00:6868:6868:0:20c:29ff:fef6:6f4c/64 scope global noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fef6:6f4c/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: veth1_br@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master mybridge state UP group default qlen 1000
    link/ether b2:75:52:f7:53:a4 brd ff:ff:ff:ff:ff:ff link-netns ns1
    inet6 fe80::b075:52ff:fef7:53a4/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
5: veth2_br@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master mybridge state UP group default qlen 1000
    link/ether 6e:06:4f:b9:36:40 brd ff:ff:ff:ff:ff:ff link-netns ns2
    inet6 fe80::6c06:4fff:feb9:3640/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
7: mybridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 6e:06:4f:b9:36:40 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::d0c4:bdff:fe69:4536/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
root@t1:~# ip netns list
ns1 (id: 0)
ns2 (id: 1)
root@t1:~# brctl show 
bridge name     bridge id               STP enabled     interfaces
mybridge                8000.6e064fb93640       no              veth1_br
                                                        veth2_br
root@t1:~#  sudo ip netns exec ns1 ping -c 3 192.168.1.20
PING 192.168.1.20 (192.168.1.20) 56(84) 字节的数据。
64 字节，来自 192.168.1.20: icmp_seq=1 ttl=64 时间=0.032 毫秒
64 字节，来自 192.168.1.20: icmp_seq=2 ttl=64 时间=0.173 毫秒
^C
--- 192.168.1.20 ping 统计 ---
已发送 2 个包， 已接收 2 个包, 0% packet loss, time 1052ms
rtt min/avg/max/mdev = 0.032/0.102/0.173/0.070 ms
root@t1:~#  sudo ip netns exec ns1 ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether a2:d7:7d:02:be:bb brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.1.10/24 scope global veth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a0d7:7dff:fe02:bebb/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
root@t1:~#  sudo ip netns exec ns2 ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
6: veth2@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether de:31:e9:31:98:6d brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.1.20/24 scope global veth2
       valid_lft forever preferred_lft forever
    inet6 fe80::dc31:e9ff:fe31:986d/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
```

## **使用 OVS Bridge 互联**

```bash
# 安装Open vSwitch (如果尚未安装)
sudo yum update
sudo yum install openvswitch # CentOS/RHEL
sudo systemctl start openvswitch
sudo systemctl enable openvswitch
sudo systemctl status openvswitch
#sudo apt install openvswitch-switch # Debian/Ubuntu

# 1. 创建网络命名空间
sudo ip netns add ns3
sudo ip netns add ns4

# 2. 创建 veth pair
sudo ip link add veth3 type veth peer name veth3_ovs
sudo ip link add veth4 type veth peer name veth4_ovs

# 3. 将 veth pair 的一端放入命名空间
sudo ip link set veth3 netns ns3
sudo ip link set veth4 netns ns4

# 4. 激活命名空间内的 veth 设备
sudo ip netns exec ns3 ip link set veth3 up
sudo ip netns exec ns4 ip link set veth4 up

# 5. 配置命名空间内的 IP 地址
sudo ip netns exec ns3 ip addr add 192.168.2.10/24 dev veth3
sudo ip netns exec ns4 ip addr add 192.168.2.20/24 dev veth4

# 6. 创建 OVS Bridge
sudo ovs-vsctl add-br ovsbr0

# 7. 将 veth pair 的另一端连接到 OVS Bridge
sudo ovs-vsctl add-port ovsbr0 veth3_ovs
sudo ovs-vsctl add-port ovsbr0 veth4_ovs

# 8. 激活连接到 OVS Bridge 的 veth 设备
sudo ip link set veth3_ovs up
sudo ip link set veth4_ovs up

# 9.激活 OVS Bridge
sudo ip link set ovsbr0 up

# 10. 测试连通性
sudo ip netns exec ns3 ping -c 3 192.168.2.20

# 11.  清理
sudo ip netns del ns3
sudo ip netns del ns4
sudo ovs-vsctl del-br ovsbr0
sudo ip link del veth3_ovs # veth3_ovs 会自动删除 veth3
sudo ip link del veth4_ovs # veth4_ovs 会自动删除 veth4
```

![sdn-study-01_linux-bridge-ovs-bridge_2.png](/img/sdn-study-01_linux-bridge-ovs-bridge_2.png)

结果如下：
```bash
root@t1:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:f6:6f:4c brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 192.168.31.172/24 brd 192.168.31.255 scope global dynamic noprefixroute ens160
       valid_lft 41491sec preferred_lft 41491sec
    inet6 fd00:6868:6868::e3f/128 scope global dynamic noprefixroute 
       valid_lft 41493sec preferred_lft 41493sec
    inet6 fd00:6868:6868:0:20c:29ff:fef6:6f4c/64 scope global noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fef6:6f4c/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: veth1_br@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master mybridge state UP group default qlen 1000
    link/ether b2:75:52:f7:53:a4 brd ff:ff:ff:ff:ff:ff link-netns ns1
    inet6 fe80::b075:52ff:fef7:53a4/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
5: veth2_br@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master mybridge state UP group default qlen 1000
    link/ether 6e:06:4f:b9:36:40 brd ff:ff:ff:ff:ff:ff link-netns ns2
    inet6 fe80::6c06:4fff:feb9:3640/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
7: mybridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 6e:06:4f:b9:36:40 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::d0c4:bdff:fe69:4536/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
8: veth3_ovs@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UP group default qlen 1000
    link/ether d2:9b:80:4a:02:39 brd ff:ff:ff:ff:ff:ff link-netns ns3
    inet6 fe80::d09b:80ff:fe4a:239/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
10: veth4_ovs@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UP group default qlen 1000
    link/ether fa:95:c4:96:e1:bc brd ff:ff:ff:ff:ff:ff link-netns ns4
    inet6 fe80::f895:c4ff:fe96:e1bc/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
12: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether aa:61:c7:91:86:39 brd ff:ff:ff:ff:ff:ff
13: ovsbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether e2:5a:b6:fa:a5:42 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::e05a:b6ff:fefa:a542/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
root@t1:~# ovs-vsctl show 
a6d87171-af75-4028-97f2-335866e29fd2
    Bridge ovsbr0
        Port ovsbr0
            Interface ovsbr0
                type: internal
        Port veth3_ovs
            Interface veth3_ovs
        Port veth4_ovs
            Interface veth4_ovs
    ovs_version: "3.4.0-2.fc41"
root@t1:~# ovs-ofctl show ovsbr0
OFPT_FEATURES_REPLY (xid=0x2): dpid:0000e25ab6faa542
n_tables:254, n_buffers:0
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
 1(veth3_ovs): addr:d2:9b:80:4a:02:39
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 2(veth4_ovs): addr:fa:95:c4:96:e1:bc
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 LOCAL(ovsbr0): addr:e2:5a:b6:fa:a5:42
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
root@t1:~# sudo ip netns exec ns3 ping -c 3 192.168.2.20
PING 192.168.2.20 (192.168.2.20) 56(84) 字节的数据。
64 字节，来自 192.168.2.20: icmp_seq=1 ttl=64 时间=0.502 毫秒
^C
--- 192.168.2.20 ping 统计 ---
已发送 1 个包， 已接收 1 个包, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.502/0.502/0.502/0.000 ms
```
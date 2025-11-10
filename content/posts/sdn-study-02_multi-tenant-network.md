+++
title = '探索SDN实操_02_ovs-bridge模拟一个简单的多租户网络'
date = 2025-07-20T15:08:26+08:00
draft = false
tags= ["网络","sdn"]
categories= ["网络"]
author= ["kl"]
+++

# 前言

> 💡 实操一下使用 ovs-vsctl 来管理OVS。包括创建/删除网桥，添加/删除端口，设置接口类型和VLAN Tag等。搭建一个包含多个OVS网桥的实验环境，模拟一个简单的多租户网络

```bash
#1. 创建 OVS 网桥
echo "Creating OVS bridges..."
sudo ovs-vsctl add-br ovsbr_tenantA
sudo ovs-vsctl add-br ovsbr_tenantB
sudo ip link set ovsbr_tenantA up
sudo ip link set ovsbr_tenantB up
echo "OVS bridges created."

#2. 创建网络命名空间 (模拟虚拟机)
echo "Creating network namespaces (simulating VMs)..."
sudo ip netns add hostA1
sudo ip netns add hostA2
sudo ip netns add hostB1
sudo ip netns add hostB2
echo "Network namespaces created."

#3. 为每个命名空间创建 veth pair 并配置
echo "Configuring Tenant A (hostA1, hostA2)..."
# hostA1 (VLAN 10)
sudo ip link add vethA1 type veth peer name vethA1_ovs
sudo ip link set vethA1 netns hostA1
sudo ip netns exec hostA1 ip link set vethA1 up
sudo ip netns exec hostA1 ip addr add 10.10.10.10/24 dev vethA1
sudo ip link set vethA1_ovs up
sudo ovs-vsctl add-port ovsbr_tenantA vethA1_ovs tag=10

# hostA2 (VLAN 20)
sudo ip link add vethA2 type veth peer name vethA2_ovs
sudo ip link set vethA2 netns hostA2
sudo ip netns exec hostA2 ip link set vethA2 up
sudo ip netns exec hostA2 ip addr add 10.10.20.10/24 dev vethA2
sudo ip link set vethA2_ovs up
sudo ovs-vsctl add-port ovsbr_tenantA vethA2_ovs tag=20
echo "Tenant A configured."

echo "Configuring Tenant B (hostB1, hostB2)..."
# hostB1 (VLAN 30)
sudo ip link add vethB1 type veth peer name vethB1_ovs
sudo ip link set vethB1 netns hostB1
sudo ip netns exec hostB1 ip link set vethB1 up
sudo ip netns exec hostB1 ip addr add 10.10.30.10/24 dev vethB1
sudo ip link set vethB1_ovs up
sudo ovs-vsctl add-port ovsbr_tenantB vethB1_ovs tag=30

# hostB2 (VLAN 40)
sudo ip link add vethB2 type veth peer name vethB2_ovs
sudo ip link set vethB2 netns hostB2
sudo ip netns exec hostB2 ip link set vethB2 up
sudo ip netns exec hostB2 ip addr add 10.10.40.10/24 dev vethB2
sudo ip link set vethB2_ovs up
sudo ovs-vsctl add-port ovsbr_tenantB vethB2_ovs tag=40
echo "Tenant B configured."

#4. 验证 OVS 配置
echo "Verifying OVS bridges and ports..."
sudo ovs-vsctl show
echo "OVS configuration shown above."

#5. 测试连通性
#5.1 Tenant A 内部测试 (不同 VLAN 不通):
echo "Testing Tenant A connectivity (hostA1 ping hostA2 - should FAIL due to VLAN isolation)..."
sudo ip netns exec hostA1 ping -c 3 10.10.20.10
echo "Ping test for hostA1 to hostA2 completed. Expected to fail."

#5.2 Tenant B 内部测试 (不同 VLAN 不通):
echo "Testing Tenant B connectivity (hostB1 ping hostB2 - should FAIL due to VLAN isolation)..."
sudo ip netns exec hostB1 ping -c 3 10.10.40.10
echo "Ping test for hostB1 to hostB2 completed. Expected to fail."

#5.3 跨租户测试 (不同网桥不通):
echo "Testing cross-tenant connectivity (hostA1 ping hostB1 - should FAIL due to separate bridges)..."
sudo ip netns exec hostA1 ping -c 3 10.10.30.10
echo "Ping test for hostA1 to hostB1 completed. Expected to fail."

#5.4 同网桥相同VLAN互通
echo "Adding hostA3 to Tenant A (VLAN 10) for internal VLAN connectivity test..."
sudo ip netns add hostA3
sudo ip link add vethA3 type veth peer name vethA3_ovs
sudo ip link set vethA3 netns hostA3
sudo ip netns exec hostA3 ip link set vethA3 up
sudo ip netns exec hostA3 ip addr add 10.10.10.11/24 dev vethA3 # 不同 IP
sudo ip link set vethA3_ovs up
sudo ovs-vsctl add-port ovsbr_tenantA vethA3_ovs tag=10
echo "hostA3 configured."

echo "Testing Tenant A internal VLAN 10 connectivity (hostA1 ping hostA3 - should SUCCEED)..."
sudo ip netns exec hostA1 ping -c 3 10.10.10.11
echo "Ping test for hostA1 to hostA3 completed. Expected to succeed."

#6. 清理实验环境
echo "Cleaning up all resources..."
# 删除 OVS 网桥 (会自动删除连接到它们的端口)
sudo ovs-vsctl del-br ovsbr_tenantA
sudo ovs-vsctl del-br ovsbr_tenantB

# 删除命名空间
sudo ip netns del hostA1
sudo ip netns del hostA2
sudo ip netns del hostA3 # 如果你创建了 hostA3
sudo ip netns del hostB1
sudo ip netns del hostB2

# 清理可能残留的 veth 设备 (通常删除 OVS bridge 会自动处理，但手动确保)
sudo ip link del vethA1_ovs 2>/dev/null
sudo ip link del vethA2_ovs 2>/dev/null
sudo ip link del vethA3_ovs 2>/dev/null
sudo ip link del vethB1_ovs 2>/dev/null
sudo ip link del vethB2_ovs 2>/dev/null

echo "Cleanup complete."
```

![sdn-study-02_multi-tenant-network.png](/img/sdn-study-02_multi-tenant-network.png)

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
       valid_lft 37588sec preferred_lft 37588sec
    inet6 fd00:6868:6868::e3f/128 scope global dynamic noprefixroute 
       valid_lft 37590sec preferred_lft 37590sec
    inet6 fd00:6868:6868:0:20c:29ff:fef6:6f4c/64 scope global noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fef6:6f4c/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
14: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether aa:61:c7:91:86:39 brd ff:ff:ff:ff:ff:ff
15: ovsbr_tenantA: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 8e:d4:f2:41:44:48 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::8cd4:f2ff:fe41:4448/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
16: ovsbr_tenantB: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether ba:39:2b:2f:d0:45 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::b839:2bff:fe2f:d045/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
17: vethA1_ovs@if18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UP group default qlen 1000
    link/ether ee:3f:23:f0:3e:3d brd ff:ff:ff:ff:ff:ff link-netns hostA1
    inet6 fe80::ec3f:23ff:fef0:3e3d/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
19: vethA2_ovs@if20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UP group default qlen 1000
    link/ether b2:67:7e:7b:cf:1e brd ff:ff:ff:ff:ff:ff link-netns hostA2
    inet6 fe80::b067:7eff:fe7b:cf1e/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
21: vethB1_ovs@if22: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UP group default qlen 1000
    link/ether 46:31:6f:bc:84:7f brd ff:ff:ff:ff:ff:ff link-netns hostB1
    inet6 fe80::4431:6fff:febc:847f/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
23: vethB2_ovs@if24: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UP group default qlen 1000
    link/ether 5a:bf:a3:de:60:2a brd ff:ff:ff:ff:ff:ff link-netns hostB2
    inet6 fe80::58bf:a3ff:fede:602a/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
       
root@t1:~# sudo ovs-vsctl show
a6d87171-af75-4028-97f2-335866e29fd2
    Bridge ovsbr_tenantB
        Port ovsbr_tenantB
            Interface ovsbr_tenantB
                type: internal
        Port vethB2_ovs
            tag: 40
            Interface vethB2_ovs
        Port vethB1_ovs
            tag: 30
            Interface vethB1_ovs
    Bridge ovsbr_tenantA
        Port ovsbr_tenantA
            Interface ovsbr_tenantA
                type: internal
        Port vethA2_ovs
            tag: 20
            Interface vethA2_ovs
        Port vethA3_ovs
            tag: 10
            Interface vethA3_ovs
        Port vethA1_ovs
            tag: 10
            Interface vethA1_ovs
    ovs_version: "3.4.0-2.fc41"

root@t1:~# ovs-ofctl show ovsbr_tenantB 
OFPT_FEATURES_REPLY (xid=0x2): dpid:0000ba392b2fd045
n_tables:254, n_buffers:0
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
 1(vethB1_ovs): addr:46:31:6f:bc:84:7f
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 2(vethB2_ovs): addr:5a:bf:a3:de:60:2a
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 LOCAL(ovsbr_tenantB): addr:ba:39:2b:2f:d0:45
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
root@t1:~# ovs-ofctl show ovsbr_tenantA
OFPT_FEATURES_REPLY (xid=0x2): dpid:00008ed4f2414448
n_tables:254, n_buffers:0
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
 1(vethA1_ovs): addr:ee:3f:23:f0:3e:3d
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 2(vethA2_ovs): addr:b2:67:7e:7b:cf:1e
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 3(vethA3_ovs): addr:1e:59:d4:16:e2:e1
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 LOCAL(ovsbr_tenantA): addr:8e:d4:f2:41:44:48
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0

root@t1:~# sudo ip netns exec hostA1 ping -c 3 10.10.20.10
ping: connect: 网络不可达
root@t1:~# sudo ip netns exec hostB1 ping -c 3 10.10.40.10
ping: connect: 网络不可达
root@t1:~# sudo ip netns exec hostA1 ping -c 3 10.10.30.10
ping: connect: 网络不可达
root@t1:~# sudo ip netns exec hostA1 ping -c 3 10.10.10.11
PING 10.10.10.11 (10.10.10.11) 56(84) 字节的数据。
64 字节，来自 10.10.10.11: icmp_seq=1 ttl=64 时间=0.568 毫秒
^C
--- 10.10.10.11 ping 统计 ---
已发送 1 个包， 已接收 1 个包, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.568/0.568/0.568/0.000 ms
```
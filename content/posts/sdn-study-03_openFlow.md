+++
title = '探索SDN实操_03_OpenFlow协议流表实操'
date = 2025-07-22T22:34:05+08:00
draft = false
tags= ["网络","sdn"]
categories= ["网络"]
author= ["kl"]
+++
# 前言


> 💡 学习一些流表相关


- **Match (匹配字段)**：定义了数据包的哪些属性（如源/目的 MAC、IP、端口、VLAN ID 等）需要被匹配。
- **Action (动作)**：定义了当数据包匹配到某个流规则后，控制器或交换机应该对数据包执行的操作（如转发到端口、丢弃、修改字段、封装/解封装 VLAN 等）。
- **Flow Rule (流规则)**：是 Match 和 Action 的组合。它告诉 OpenFlow 交换机：“如果一个数据包满足这些匹配条件，就执行这些动作。”
- **Flow Table (流表)**：OpenFlow 交换机内部用于存储流规则的集合。
- **Pipeline (多级流表)**：OpenFlow 1.0 版本只有一个流表。从 OpenFlow 1.1 开始引入了多级流表（Pipeline）概念，数据包可以按顺序经过多个流表进行处理。每个流表可以有不同的匹配和动作逻辑。这提供了更大的灵活性，例如，第一个流表可以处理安全策略，第二个流表处理路由，第三个流表处理 QoS 等。

手动下发流规则，实现：

- 让两个端口互通（模拟二层交换机）。
- 丢弃来自某个MAC地址的流量。
- 将流量从一个端口转发到另一个端口。

# 实操

## 环境搭建

```bash
#1. 创建 OVS 网桥
sudo ovs-vsctl add-br ovsbr0
sudo ovs-vsctl set bridge ovsbr0 protocols=OpenFlow13 # 明确指定使用 OpenFlow 1.3
sudo ip link set ovsbr0 up

#2. 创建命名空间和连接到 OVS
# host1
sudo ip netns add host1
sudo ip link add veth1 type veth peer name veth1_ovs
sudo ip link set veth1 netns host1
sudo ip netns exec host1 ip link set veth1 up
sudo ip netns exec host1 ip addr add 192.168.1.10/24 dev veth1
sudo ip link set veth1_ovs up
sudo ovs-vsctl add-port ovsbr0 veth1_ovs
# host2
sudo ip netns add host2
sudo ip link add veth2 type veth peer name veth2_ovs
sudo ip link set veth2 netns host2
sudo ip netns exec host2 ip link set veth2 up
sudo ip netns exec host2 ip addr add 192.168.1.20/24 dev veth2
sudo ip link set veth2_ovs up
sudo ovs-vsctl add-port ovsbr0 veth2_ovs
# host3
sudo ip netns add host3
sudo ip link add veth3 type veth peer name veth3_ovs
sudo ip link set veth3 netns host3
sudo ip netns exec host3 ip link set veth3 up
sudo ip netns exec host3 ip addr add 192.168.1.30/24 dev veth3
sudo ip link set veth3_ovs up
sudo ovs-vsctl add-port ovsbr0 veth3_ovs
#3. 获取 OVS 端口号
sudo ovs-ofctl show ovsbr0 --protocols=OpenFlow13
# 输出示例如下：
root@t1:~# ovs-ofctl show ovsbr0 --protocols=OpenFlow13
OFPT_FEATURES_REPLY (OF1.3) (xid=0x2): dpid:0000ca7651a9a549
n_tables:254, n_buffers:0
capabilities: FLOW_STATS TABLE_STATS PORT_STATS GROUP_STATS QUEUE_STATS
OFPST_PORT_DESC reply (OF1.3) (xid=0x3):
 1(veth1_ovs): addr:6e:9b:0b:7e:46:6d
     config:     0
     state:      LIVE
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 2(veth2_ovs): addr:0e:fb:27:b1:04:d5
     config:     0
     state:      LIVE
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 3(veth3_ovs): addr:d2:9b:80:4a:02:39
     config:     0
     state:      LIVE
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 LOCAL(ovsbr0): addr:ca:76:51:a9:a5:49
     config:     0
     state:      LIVE
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (OF1.3) (xid=0x9): frags=normal miss_send_len=0
```

<aside>
💡

模拟二层交换机，我们需要一个“学习”和“泛洪”的逻辑。最简单的方式是，如果数据包的目的 MAC 地址未知，就泛洪（flood）到所有端口。如果已知，就直接转发。

</aside>

```bash
ececho "Adding default flow rules for L2 learning and forwarding..."

# Table 0: 优先级为0的默认规则，丢弃所有流量
# 确保没有其他规则匹配的流量被丢弃，防止意外转发
sudo ovs-ofctl add-flow ovsbr0 "table=0,priority=0,actions=drop"

# Table 0: 泛洪 ARP 流量，确保 IP 地址解析正常
sudo ovs-ofctl add-flow ovsbr0 "table=0,priority=100,dl_type=0x0806,actions=FLOOD"

# Table 0: 泛洪 IPv6 NDP (邻居发现协议) 流量
sudo ovs-ofctl add-flow ovsbr0 "table=0,priority=100,dl_type=0x86dd,nw_proto=58,actions=FLOOD"

# Table 0: 默认动作，如果有学习功能，就泛洪。
# 对于简单的 L2 学习，我们会使用 controller 或者更高级的 OVS 功能。
# 这里我们先用最简单的，如果没匹配到上面的ARP/NDP，就直接发送给控制器让控制器来处理
# 但由于我们没有控制器，这个动作在这里会丢弃非ARP/NDP的未知流量，因为没有控制器处理。
# 后面我们会用明确的规则来替代这个。
# sudo ovs-ofctl add-flow ovsbr0 "table=0,priority=1,actions=normal" # 'normal' 动作模拟传统交换机行为，自动学习MAC地址，但可能不适用于所有OpenFlow版本和场景。

# 为了简化，我们暂时不依赖 'normal' 动作，而是手动配置。
# 先清空所有流规则，确保我们从头开始
sudo ovs-ofctl --protocols=OpenFlow13 del-flows ovsbr0
```

## 流表下发

```bash
#1. 让两个端口互通（模拟二层交换机）,我们将 host1 和 host2 互通。这意味着来自 host1 的流量如果目标是 host2，就转发到 veth2_ovs；反之亦然。
# 获取 MAC 地址：
sudo ip netns exec host1 ip link show veth1
sudo ip netns exec host2 ip link show veth2
sudo ip netns exec host3 ip link show veth3
# 得到
#host1 (veth1) MAC: a2:d7:7d:02:be:bb 
#host2 (veth2) MAC: de:31:e9:31:98:6d 
#host3 (veth3) MAC: 6e:fd:e7:0f:f5:18

# 下发流规则：
echo "Configuring L2 communication between host1 (port 1) and host2 (port 2)..."
# host1 to host2
sudo ovs-ofctl --protocols=OpenFlow13 add-flow ovsbr0 "table=0,priority=100,in_port=1,dl_dst=de:31:e9:31:98:6d ,actions=output:2"
# host2 to host1
sudo ovs-ofctl --protocols=OpenFlow13 add-flow ovsbr0 "table=0,priority=100,in_port=2,dl_dst=a2:d7:7d:02:be:bb,actions=output:1"

# 添加 ARP 泛洪规则，让 IP 地址解析正常工作
sudo ovs-ofctl --protocols=OpenFlow13 add-flow ovsbr0 "table=0,priority=90,dl_type=0x0806,actions=FLOOD"

# 默认丢弃规则，确保除了明确允许的流量外，其他流量都被丢弃
sudo ovs-ofctl --protocols=OpenFlow13 add-flow ovsbr0 "table=0,priority=0,actions=drop"

echo "L2 rules added. Testing connectivity between host1 and host2..."
# 测试，应该能 ping 通
sudo ip netns exec host1 ping -c 3 192.168.1.20

#2. 丢弃来自某个 MAC 地址的流量
echo "Adding flow rule to drop traffic from host3 (MAC: 6e:fd:e7:0f:f5:18)..."
# 优先级要高于之前的 L2 转发规则
sudo ovs-ofctl --protocols=OpenFlow13 add-flow ovsbr0 "table=0,priority=200,dl_src=6e:fd:e7:0f:f5:18,actions=drop"
echo "Drop rule added. Testing connectivity from host3 to host1 (should FAIL)..."
# 测试，应该 ping 不通
sudo ip netns exec host3 ping -c 3 192.168.1.10

#3. 将流量从一个端口转发到另一个端口
echo "Deleting previous host1 to host2 forwarding rule..."
sudo ovs-ofctl --protocols=OpenFlow13 del-flows ovsbr0 "table=0,in_port=1,dl_dst=de:31:e9:31:98:6d "

echo "Adding flow rule to redirect ALL traffic from host1 (port 1) to host3 (port 3)..."
# 这个规则的优先级可以和 ARP 规则保持一致或更高，以确保它先匹配
sudo ovs-ofctl --protocols=OpenFlow13 add-flow ovsbr0 "table=0,priority=150,in_port=1,actions=output:3"
echo "Redirect rule added. Testing connectivity from host1 to host2 (should FAIL) and host1 to host3 (should succeed if host3 processes it)..."

# 测试
sudo ip netns exec host1 ping -c 3 192.168.1.20
# 应该 ping 不通
sudo ip netns exec host1 ping -c 3 192.168.1.30
# 应该 ping 不通，由于我们之前设置了 host3 的流量丢弃规则，host3 回包会被丢弃。

# 查看流规则
sudo ovs-ofctl --protocols=OpenFlow13 dump-flows ovsbr0
```

### 验证

```bash
root@t1:~# ovs-ofctl dump-flows --protocols=OpenFlow13 ovsbr0
 cookie=0x0, duration=5.551s, table=0, n_packets=0, n_bytes=0, priority=200,dl_src=6e:fd:e7:0f:f5:18 actions=drop
 cookie=0x0, duration=35.665s, table=0, n_packets=0, n_bytes=0, priority=100,in_port="veth1_ovs",dl_dst=de:31:e9:31:98:6d actions=output:"veth2_ovs"
 cookie=0x0, duration=35.645s, table=0, n_packets=0, n_bytes=0, priority=100,in_port="veth2_ovs",dl_dst=a2:d7:7d:02:be:bb actions=output:"veth1_ovs"
 cookie=0x0, duration=35.625s, table=0, n_packets=2, n_bytes=84, priority=90,arp actions=FLOOD
 cookie=0x0, duration=35.606s, table=0, n_packets=3, n_bytes=266, priority=0 actions=drop
 
root@t1:~# sudo ip netns exec host1 ping -c 3 192.168.1.20
PING 192.168.1.20 (192.168.1.20) 56(84) 字节的数据。
64 字节，来自 192.168.1.20: icmp_seq=1 ttl=64 时间=9.33 毫秒
64 字节，来自 192.168.1.20: icmp_seq=2 ttl=64 时间=0.175 毫秒
^C
--- 192.168.1.20 ping 统计 ---
已发送 2 个包， 已接收 2 个包, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.175/4.754/9.334/4.579 ms
root@t1:~# 
root@t1:~# sudo ip netns exec host3 ping -c 3 192.168.1.10
PING 192.168.1.10 (192.168.1.10) 56(84) 字节的数据。
^C
--- 192.168.1.10 ping 统计 ---
已发送 2 个包， 已接收 0 个包, 100% packet loss, time 1050ms

# 修改后
root@t1:~# ovs-ofctl dump-flows --protocols=OpenFlow13 ovsbr0
 cookie=0x0, duration=50.440s, table=0, n_packets=0, n_bytes=0, priority=200,dl_src=6e:fd:e7:0f:f5:18 actions=drop
 cookie=0x0, duration=4.307s, table=0, n_packets=0, n_bytes=0, priority=150,in_port="veth1_ovs" actions=output:"veth3_ovs"
 cookie=0x0, duration=80.534s, table=0, n_packets=0, n_bytes=0, priority=100,in_port="veth2_ovs",dl_dst=a2:d7:7d:02:be:bb actions=output:"veth1_ovs"
 cookie=0x0, duration=80.514s, table=0, n_packets=2, n_bytes=84, priority=90,arp actions=FLOOD
 cookie=0x0, duration=80.495s, table=0, n_packets=3, n_bytes=266, priority=0 actions=drop
root@t1:~# sudo ip netns exec host1 ping -c 3 192.168.1.20
PING 192.168.1.20 (192.168.1.20) 56(84) 字节的数据。
^C
--- 192.168.1.20 ping 统计 ---
已发送 3 个包， 已接收 0 个包, 100% packet loss, time 2047ms

root@t1:~# 
root@t1:~# sudo ip netns exec host1 ping -c 3 192.168.1.30
PING 192.168.1.30 (192.168.1.30) 56(84) 字节的数据。
^C
--- 192.168.1.30 ping 统计 ---
已发送 3 个包， 已接收 0 个包, 100% packet loss, time 2079ms
```

## 多级流表下发

```bash
# 使用多级流表
#1. 清空所有流规则
sudo ovs-ofctl --protocols=OpenFlow13 del-flows ovsbr0

#2. Table 0: 安全策略 (例如，丢弃来自 host3 的流量)
# 丢弃来自 host3 的流量，然后跳到下一个表
sudo ovs-ofctl --protocols=OpenFlow13 add-flow ovsbr0 "table=0,priority=200,dl_src=6e:fd:e7:0f:f5:18,actions=drop"
# 如果不匹配上面的规则，就转发到 table=1
sudo ovs-ofctl --protocols=OpenFlow13 add-flow ovsbr0 "table=0,priority=0,actions=goto_table:1"

#3. Table 1: 转发策略 (例如，host1 到 host2 转发)
# ARP 泛洪
sudo ovs-ofctl --protocols=OpenFlow13 add-flow ovsbr0 "table=1,priority=90,dl_type=0x0806,actions=FLOOD"
# host1 to host2
sudo ovs-ofctl --protocols=OpenFlow13 add-flow ovsbr0 "table=1,priority=100,in_port=1,dl_dst=de:31:e9:31:98:6d,actions=output:2"
# host2 to host1
sudo ovs-ofctl --protocols=OpenFlow13 add-flow ovsbr0 "table=1,priority=100,in_port=2,dl_dst=a2:d7:7d:02:be:bb,actions=output:1"
# Table 1 的默认丢弃规则
sudo ovs-ofctl --protocols=OpenFlow13 add-flow ovsbr0 "table=1,priority=0,actions=drop"
# 查看
ovs-ofctl dump-flows --protocols=OpenFlow13 ovsbr0

#4. 测试多级流表：
echo "Testing multi-table setup..."
sudo ip netns exec host3 ping -c 3 192.168.1.10 # Should FAIL
sudo ip netns exec host1 ping -c 3 192.168.1.20 # Should SUCCEED

#5. 清理实验环境
echo "Cleaning up all resources..."
sudo ovs-vsctl del-br ovsbr0
sudo ip netns del host1
sudo ip netns del host2
sudo ip netns del host3
sudo ip link del veth1_ovs 2>/dev/null
sudo ip link del veth2_ovs 2>/dev/null
sudo ip link del veth3_ovs 2>/dev/null
echo "Cleanup complete."
```

### 验证

```bash
root@t1:~# ovs-ofctl dump-flows --protocols=OpenFlow13 ovsbr0
 cookie=0x0, duration=24.941s, table=0, n_packets=0, n_bytes=0, priority=200,dl_src=6e:fd:e7:0f:f5:18 actions=drop
 cookie=0x0, duration=24.922s, table=0, n_packets=0, n_bytes=0, priority=0 actions=goto_table:1
 cookie=0x0, duration=24.884s, table=1, n_packets=0, n_bytes=0, priority=100,in_port="veth1_ovs",dl_dst=de:31:e9:31:98:6d actions=output:"veth2_ovs"
 cookie=0x0, duration=24.865s, table=1, n_packets=0, n_bytes=0, priority=100,in_port="veth2_ovs",dl_dst=a2:d7:7d:02:be:bb actions=output:"veth1_ovs"
 cookie=0x0, duration=24.903s, table=1, n_packets=0, n_bytes=0, priority=90,arp actions=FLOOD
 cookie=0x0, duration=24.847s, table=1, n_packets=0, n_bytes=0, priority=0 actions=drop
root@t1:~# sudo ip netns exec host3 ping -c 3 192.168.1.10
PING 192.168.1.10 (192.168.1.10) 56(84) 字节的数据。
^C
--- 192.168.1.10 ping 统计 ---
已发送 3 个包， 已接收 0 个包, 100% packet loss, time 2049ms

root@t1:~# sudo ip netns exec host1 ping -c 3 192.168.1.20
PING 192.168.1.20 (192.168.1.20) 56(84) 字节的数据。
64 字节，来自 192.168.1.20: icmp_seq=1 ttl=64 时间=0.752 毫秒
64 字节，来自 192.168.1.20: icmp_seq=2 ttl=64 时间=0.148 毫秒
^C
--- 192.168.1.20 ping 统计 ---
已发送 2 个包， 已接收 2 个包, 0% packet loss, time 1065ms
rtt min/avg/max/mdev = 0.148/0.450/0.752/0.302 ms
root@t1:~# 
```

## 整体拓扑
![img.png](/img/sdn-study-03_openFlow.png.png)

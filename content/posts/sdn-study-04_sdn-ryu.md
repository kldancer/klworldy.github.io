+++
title = '探索SDN实操_04_sdn Ryu Controller'
date = 2025-08-02T00:52:44+08:00
draft = false
tags= ["网络","sdn"]
categories= ["网络"]
author= ["kl"]
+++

# 前言

## SDN 核心思想：控制平面与数据平面分离

SDN（软件定义网络）最核心的思想就是**控制平面 (Control Plane)** 与 **数据平面 (Data Plane)** 的分离。

- **数据平面 (Data Plane / Forwarding Plane)**：
    - **职责**：负责实际的数据包转发，也就是“怎么走”。
    - **组成**：由网络设备（如交换机、路由器）构成。这些设备现在被称为**数据平面设备**或**转发器 (Forwarders)**。
    - **特点**：它们变得“愚蠢”且可编程。它们不再自行决定数据包的转发路径，而是严格遵循**控制平面**下发的指令（流规则）进行操作。
    - **南向接口 (Southbound Interface)**：是数据平面设备与控制平面通信的接口。它允许控制器向数据平面设备下发流规则，并接收数据平面设备的事件（如 Packet-In）。OpenFlow 是最广为人知和广泛使用的南向接口协议。
- **控制平面 (Control Plane / Control Layer)**：
    - **职责**：负责网络的“大脑”，决定数据包的转发逻辑和策略，也就是“往哪走”。
    - **组成**：由一个或多个**网络控制器 (Network Controllers)** 组成（例如 OpenDaylight, ONOS, Ryu, Floodlight 等）。
    - **特点**：集中化管理，具有网络的全局视图。它根据网络管理员的策略或应用程序的需求，计算出最佳的转发路径和规则，然后通过南向接口下发给数据平面设备。
    - **北向接口 (Northbound Interface)**：是控制器与上层应用 (Applications) 或业务编排系统通信的接口。它允许应用程序向控制器请求网络服务、配置网络策略或获取网络状态。REST API 是最常见的北向接口形式。

**总结**：

- 传统网络：控制平面和数据平面紧密耦合在每个网络设备中。
- SDN 网络：将控制平面从数据平面设备中抽离出来，集中到控制器中。控制器通过南向接口控制数据平面，并通过北向接口暴露网络服务给上层应用。

### 南向接口 (Southbound Interface) 和 北向接口 (Northbound Interface)

- **南向接口 (Southbound Interface)**：
    - **目的**：控制器 <-> 转发设备。
    - **功能**：控制器向转发设备下发流规则、查询设备状态；转发设备向控制器报告事件（如 Packet-In，当遇到未知数据包时）。
    - **例子**：OpenFlow, NETCONF, OVSDB 等。OpenFlow 是目前 SDN 领域最主流和最具代表性的南向接口协议。
- **北向接口 (Northbound Interface)**：
    - **目的**：控制器 <-> 上层应用。
    - **功能**：应用程序通过北向接口向控制器请求网络资源、配置网络策略（例如“创建一条带宽为 100Mbps 的隧道”），或者获取网络拓扑、流量统计等信息。
    - **例子**：RESTful API (最常见), Java API, Python SDK 等。

## Ryu控制器实操


>💡 用 Python 编写一个简单的 Ryu L2 Learning Switch


### 环境搭建

```bash
#1. 清理旧环境
sudo ovs-vsctl del-br ovsbr0
sudo ip netns del host1
sudo ip netns del host2
sudo ip netns del host3
sudo ip link del veth1_ovs 2>/dev/null
sudo ip link del veth2_ovs 2>/dev/null
sudo ip link del veth3_ovs 2>/dev/null

#2. 创建 OVS 网桥
sudo ovs-vsctl add-br ovsbr0
# 将 OVS 网桥连接到控制器（Ryu 默认监听 6653 端口）
sudo ovs-vsctl set-controller ovsbr0 tcp:127.0.0.1:6653
sudo ovs-vsctl set bridge ovsbr0 protocols=OpenFlow13 # 使用 OpenFlow 1.3
sudo ip link set ovsbr0 up

#3. 创建命名空间和连接到 OVS
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
```

### 安装Ryu

Ryu最新版本为4.34，早就19年就不维护了，很多依赖都不适配当前最新版本，python还是乖乖用3.9，不然不好安装。

```bash
# 尝试删除旧的虚拟环境
rm -rf /root/venvs/ryu 2>/dev/null
# 卸载全局安装的Ryu（如果存在）
pip uninstall ryu dnspython eventlet -y 2>/dev/null
sudo dnf update
sudo dnf install -y git make gcc zlib-devel bzip2 bzip2-devel readline-devel sqlite sqlite-devel openssl-devel xz xz-devel libffi-devel
# pyenv
curl https://pyenv.run | bash
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init -)"\nfi' >> ~/.bashrc
source ~/.bashrc
pyenv --version
pyenv install 3.9.18 # 可以选择最新的3.9.x版本
pyenv global 3.9.18 # 将这个版本设置为全局默认，或者只在项目目录设置local
python --version
python -m venv ~/ryu_env
source ~/ryu_env/bin/activate
pip uninstall eventlet -y
pip uninstall dnspython -y
pip install eventlet==0.25.0 dnspython==1.16.0
pip install ryu==4.34
```

### 程序

```python
# l2_learning_switch.py

from ryu.base import app_manager
from ryu.controller import ofp_event
from ryu.controller.handler import CONFIG_DISPATCHER, MAIN_DISPATCHER
from ryu.controller.handler import set_ev_cls
from ryu.ofproto import ofproto_v1_3
from ryu.lib.packet import packet
from ryu.lib.packet import ethernet
from ryu.lib.packet import ether_types

class L2LearningSwitch(app_manager.RyuApp):
    OFP_VERSIONS = [ofproto_v1_3.OFP_VERSION]

    def __init__(self, *args, **kwargs):
        super(L2LearningSwitch, self).__init__(*args, **kwargs)
        # mac_to_port 字典用于存储 MAC 地址到端口的映射
        # 格式: {dpid: {mac_addr: port_no}}
        self.mac_to_port = {}

    @set_ev_cls(ofp_event.EventOFPSwitchFeatures, CONFIG_DISPATCHER)
    def switch_features_handler(self, ev):
        """
        处理交换机连接事件。
        当交换机连接到控制器时，控制器会收到此事件。
        在这里，我们安装一个默认的 Table-miss 流规则，
        将所有未知流量发送到控制器进行处理。
        """
        datapath = ev.msg.datapath
        ofproto = datapath.ofproto
        parser = datapath.ofproto_parser

        self.logger.info("Switch connected: dpid=%s", datapath.id)

        # 安装默认的 table-miss 流规则
        # 优先级为 0 的规则，匹配所有数据包
        # 动作: 将数据包发送到控制器 (OFPP_CONTROLLER)
        # max_len 用于 Packet-In 消息的截断长度
        match = parser.OFPMatch()
        actions = [parser.OFPActionOutput(ofproto.OFPP_CONTROLLER,
                                          ofproto.OFPCML_NO_BUFFER)]
        self.add_flow(datapath, 0, match, actions) # priority 0 for table-miss

    def add_flow(self, datapath, priority, match, actions, buffer_id=None):
        """
        辅助函数，用于向交换机下发流规则。
        """
        ofproto = datapath.ofproto
        parser = datapath.ofproto_parser

        inst = [parser.OFPInstructionActions(ofproto.OFPIT_APPLY_ACTIONS,
                                             actions)]
        if buffer_id:
            mod = parser.OFPFlowMod(datapath=datapath, buffer_id=buffer_id,
                                     priority=priority, match=match,
                                     instructions=inst)
        else:
            mod = parser.OFPFlowMod(datapath=datapath, priority=priority,
                                     match=match, instructions=inst)
        datapath.send_msg(mod)

    @set_ev_cls(ofp_event.EventOFPPacketIn, MAIN_DISPATCHER)
    def _packet_in_handler(self, ev):
        """
        处理 Packet-In 事件。
        当交换机收到一个数据包，并且没有匹配到任何流规则时，
        它会根据 table-miss 规则将数据包发送到控制器。
        """
        msg = ev.msg
        datapath = msg.datapath
        ofproto = datapath.ofproto
        parser = datapath.ofproto_parser

        in_port = msg.match['in_port'] # 数据包进入的端口

        pkt = packet.Packet(msg.data)
        eth = pkt.get_protocols(ethernet.ethernet)[0]

        # 忽略 LLDP 和 STP 流量
        if eth.ethertype == ether_types.ETH_TYPE_LLDP:
            return

        dst = eth.dst # 目的 MAC 地址
        src = eth.src # 源 MAC 地址
        dpid = datapath.id # 交换机 ID

        self.logger.info("packet in %s %s %s %s", dpid, src, dst, in_port)

        # 学习源 MAC 地址和入端口的映射关系
        # 如果是第一次见到这个 MAC 地址，就记录下来
        self.mac_to_port.setdefault(dpid, {})
        self.mac_to_port[dpid][src] = in_port

        # 检查目的 MAC 地址是否已经学习到
        if dst in self.mac_to_port[dpid]:
            # 如果目的 MAC 地址已知，则将数据包转发到对应的端口
            out_port = self.mac_to_port[dpid][dst]
        else:
            # 如果目的 MAC 地址未知，则泛洪 (flood) 到所有端口
            # OFPP_FLOOD 表示发送到所有非入端口和非 STP 端口
            out_port = ofproto.OFPP_FLOOD

        actions = [parser.OFPActionOutput(out_port)]

        # 如果目的 MAC 已知，并且不是泛洪，就安装流规则
        # 以便后续相同目的 MAC 的流量不再发送到控制器
        if out_port != ofproto.OFPP_FLOOD:
            match = parser.OFPMatch(in_port=in_port, eth_dst=dst)
            # 优先级为 1，高于默认的 table-miss 规则 (优先级 0)
            self.add_flow(datapath, 1, match, actions, msg.buffer_id)
        else:
            # 如果是泛洪，直接发送数据包，不安装流规则 (因为泛洪流量可能不规律)
            # 如果 buffer_id 有效，使用 buffer_id，否则直接发送数据
            data = None
            if msg.buffer_id == ofproto.OFP_NO_BUFFER:
                data = msg.data
            out = parser.OFPPacketOut(datapath=datapath, buffer_id=msg.buffer_id,
                                      in_port=in_port, actions=actions, data=data)
            datapath.send_msg(out)
```

### 执行

```bash
# 运行 Ryu 控制器
ryu-manager l2_learning_switch.py
#1. 初始状态：此时 OVS 网桥上只有一条默认的 table-miss 规则 (priority=0, actions=CONTROLLER)。
root@t1:~# sudo ovs-ofctl --protocols=OpenFlow13 dump-flows ovsbr0 
 cookie=0x0, duration=8.972s, table=0, n_packets=0, n_bytes=0, priority=0 actions=CONTROLLER:65535

#2. host1 ping host2 (第一次)：
root@t1:~#  sudo ip netns exec host1 ping -c 3 192.168.1.20
PING 192.168.1.20 (192.168.1.20) 56(84) 字节的数据。
64 字节，来自 192.168.1.20: icmp_seq=3 ttl=64 时间=0.693 毫秒

--- 192.168.1.20 ping 统计 ---
已发送 3 个包， 已接收 1 个包, 66.6667% packet loss, time 2066ms
rtt min/avg/max/mdev = 0.693/0.693/0.693/0.000 ms
root@t1:~# 

# 同时，ryu程序有输出
(ryu_env) root@t1:~# ryu-manager l2_learning_switch.py
loading app l2_learning_switch.py
loading app ryu.controller.ofp_handler
instantiating app l2_learning_switch.py of L2LearningSwitch
instantiating app ryu.controller.ofp_handler of OFPHandler
Switch connected: dpid=64033405859660

packet in 64033405859660 a2:d7:7d:02:be:bb ff:ff:ff:ff:ff:ff 1
packet in 64033405859660 de:31:e9:31:98:6d a2:d7:7d:02:be:bb 2
packet in 64033405859660 a2:d7:7d:02:be:bb ff:ff:ff:ff:ff:ff 1
packet in 64033405859660 a2:d7:7d:02:be:bb de:31:e9:31:98:6d 1
packet in 64033405859660 a2:d7:7d:02:be:bb de:31:e9:31:98:6d 1

#3. 检查 OVS 流规则：这些是 Ryu 根据学习到的 MAC 地址自动下发的流规则，优先级为 1。
root@t1:~# sudo ovs-ofctl --protocols=OpenFlow13 dump-flows ovsbr0 
 cookie=0x0, duration=10.081s, table=0, n_packets=3, n_bytes=182, priority=1,in_port="veth2_ovs",dl_dst=a2:d7:7d:02:be:bb actions=output:"veth1_ovs"
 cookie=0x0, duration=9.032s, table=0, n_packets=2, n_bytes=140, priority=1,in_port="veth1_ovs",dl_dst=de:31:e9:31:98:6d actions=output:"veth2_ovs"
 cookie=0x0, duration=33.009s, table=0, n_packets=5, n_bytes=322, priority=0 actions=CONTROLLER:65535
 
#4. host1 ping host2 (第二次)：
root@t1:~#  sudo ip netns exec host1 ping -c 3 192.168.1.20
PING 192.168.1.20 (192.168.1.20) 56(84) 字节的数据。
64 字节，来自 192.168.1.20: icmp_seq=1 ttl=64 时间=0.303 毫秒
64 字节，来自 192.168.1.20: icmp_seq=2 ttl=64 时间=0.262 毫秒
64 字节，来自 192.168.1.20: icmp_seq=3 ttl=64 时间=0.138 毫秒

--- 192.168.1.20 ping 统计 ---
已发送 3 个包， 已接收 3 个包, 0% packet loss, time 2052ms
rtt min/avg/max/mdev = 0.138/0.234/0.303/0.070 ms

#5. 同理 host1 ping host3 (第一次)：Ryu 再次收到 Packet-In，学习 host3 的 MAC 地址，并为 host1 到 host3 的流量下发新的流规则。
sudo ip netns exec host1 ping -c 3 192.168.1.30
```

**总结 Ryu 的行为模式：**

1. **第一次见到的流量**：当 OVS 收到一个数据包，并且没有任何流规则匹配它时（只会匹配到默认的 priority=0, actions=CONTROLLER 规则），OVS 会将这个数据包的头部（以及一部分数据）封装成 Packet-In 消息发送给 Ryu 控制器。
2. **学习**：Ryu 控制器会分析 Packet-In 消息，特别是数据包的源 MAC 地址和它进入 OVS 的端口号，并将这个 MAC-to-Port 映射存储在自己的 self.mac_to_port 字典中。
3. **决策**：
    - 如果数据包的目的 MAC 地址在 self.mac_to_port 中已知，Ryu 就知道应该将数据包转发到哪个端口。
    - 如果目的 MAC 地址未知（例如 ARP 广播），Ryu 就会决定泛洪数据包到除了入端口之外的所有端口。
4. **下发流规则**：如果 Ryu 成功地学习到了源 MAC 和目的 MAC 的映射关系，并且决定直接转发（而不是泛洪），它就会通过 OpenFlow 协议向 OVS 网桥**下发一条新的流规则**。这条流规则告诉 OVS：“以后如果收到来自这个端口、目的 MAC 是这个地址的数据包，就直接转发到那个端口，**不用再问我了**。”
5. **数据平面转发**：一旦流规则被下发并安装到 OVS 的流表中，后续所有符合该流规则的数据包都将由 OVS **直接在数据平面进行高速转发**，而不再需要经过控制器。这就是 SDN “控制平面与数据平面分离”的精髓所在：控制器只负责初始的决策和规则下发，实际的数据转发则由数据平面设备高效完成。
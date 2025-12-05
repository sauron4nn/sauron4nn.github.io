# 使用OpenvSwitch完成基于BFD的主备链路切换

在没有接触到交换机之前，我对计算机网络的理解是按照上学时接触的《计算机网络》进行的。
网络方面使用的最多的是Linux系统，所以我对于Linux系统上网络的配置基本就是那几套。

使用`iptables`配置防火墙，使用`ip link`命令创建接口和隧道，使用`ip route`命令查看和修改路由表等。

但是当我需要将一台普通的Linux设备和一台交换机对接时，才发现交换机上简单的配置在Linux
原始环境中这么难以处理，比如标题中的问题，如果纯粹使用上面提到的`iptables`和`iproute2`
工具，那估计还得搭配各种小程序、shell脚本等等，问题最终可能也会解决，但手段繁琐且容易出问题。

于是我发现了`Open vSwitch`这个项目，看介绍是一个软件实现的虚拟交换机，它正好能优雅的解决
我标题中的问题。

## 实验环境

本次试验基于3个openwrt24虚拟机，理论上其他Linux发行版也行，只需安装openvswitch软件包即可。

在openwrt中安装openvswitch很简单，使用opkg安装即可。

```shell
opkg update && opkg install openvswitch
```

安装完成之后启动openvswitch服务（我不清楚为什么这个不默认启动）

```shell
ovs-ctl start
```

3台虚拟机均处于同一网络上，分配IP为

A: 192.168.1.1、B: 192.168.1.2、C: 192.168.1.3

## 实验目的

我需要搭建一个3台虚拟机构成的网络环境，分别记为A、B、C。其中A和C使用一条链路直连，A和B之间连接一条链路，B和C之间连接一条链路。
B实际上进行中转，为A，C提供第二条虚拟直连链路。

```text
           +---+
           | B |
           +-+-+
          /     \
         /       \
        /         \
       /           \
+-----+             +-----+
|  A  |---- L1 -----|  C  |
+-----+             +-----+
   ^                   ^
   |                   |
   +------- L2 --------+
    Virtual Direct Path
       (via B Relay)
```

## 实验步骤

整个大致方案是A和C之间建立两条VxLan隧道，每条VxLan隧道都配置bfd，在A、C两端bond成一个接口，bond策略选择主备`active-backup`，bond接口优先选择`L2`（这样我们可以在B上控制通断来检查主备倒换功能）。

首先配置A、C的Bond接口（以A为例，C将IP反过来即可）：

```bash
ovs-vsctl add-br br-switch

ovs-vsctl add-bond br-switch bond0 vxlan_l1 vxlan_l2 bond_mode=active-backup lacp=false

```

这里由于我们还没有设置接口`vxlan_l1`和`vxlan_l2`，所以会报错，不过没关系，我们马上设置两个接口：

```bash
ovs-vsctl set interface vxlan_l1 type=vxlan options:remote_ip=192.168.1.3 options:key=9999 options:dst_port=4789
ovs-vsctl set interface vxlan_l2 type=vxlan options:remote_ip=192.168.1.2 options:key=9901 options:dst_port=4789
```

现在我们设置`vxlan_l2`的bfd参数：

```bash
ovs-vsctl set interface vxlan_l2 \
          bfd:enable=true \
		  bfd:min_rx=100 \
		  bfd:min_tx=100 \
		  bfd:forward_if_rx=true
```

最后我们设置bond接口的优先接口：

```bash
ovs-vsctl set port bond0 other_config:bond-primary=vxlan_l2
```

A、C的配置完成

检查bfd和bond状态，如果一切顺利的话会得到：

```text
root@OpenWrt:~# ovs-appctl bfd/show vxlan_l2
  Forwarding: true
  Detect Multiplier: 3
  Concatenated Path Down: false
  TX Interval: Approx 100ms
  RX Interval: Approx 100ms
  Detect Time: now +264ms
  Next TX Time: now +91ms
  Last TX Time: now -8ms

  Local Flags: none
  Local Session State: up
  Local Diagnostic: Control Detection Time Expired
  Local Discriminator: 0x54f2413d
  Local Minimum TX Interval: 100ms
  Local Minimum RX Interval: 100ms

  Remote Flags: none
  Remote Session State: up
  Remote Diagnostic: Control Detection Time Expired
  Remote Discriminator: 0x3aaaeee9
  Remote Minimum TX Interval: 100ms
  Remote Minimum RX Interval: 100ms
  Remote Detect Multiplier: 3

root@OpenWrt:~# ovs-appctl bond/show bond0
---- bond0 ----
bond_mode: active-backup
bond may use recirculation: no, Recirc-ID : -1
bond-hash-basis: 0
lb_output action: disabled, bond-id: -1
updelay: 0 ms
downdelay: 0 ms
lacp_status: off
lacp_fallback_ab: false
active-backup primary: vxlan_l2
active member mac: 9a:77:1b:c5:ae:2f(vxlan_l2)

member vxlan_l1: enabled
  may_enable: true

member vxlan_l2: enabled
  active member
  may_enable: true

```

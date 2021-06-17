# net-isolation

Isolated net by namespace in Linux!

## 简介

&emsp;&emsp;为了限制服务器中某些进程进行网络访问，或者不干扰Host上的其他进程的网络空间等．需要开辟出一个隔离的网络，专门服务于这些(单个或集群式)进程．

&emsp;&emsp;基于现阶段版本Linux提供的iproute2工具集(新)，net-tools(旧)，brctl，iptables等命令．该网络隔离的方法也被docker底层用于在Linux系统上的网络子模块的隔离．

&emsp;&emsp;鉴于未研究过新iproute2命令，在网络配置上面尽量使用旧式的命令．

## 步骤

- ip netns add xxns # 创建名为xxns的网络namepsace，docker中名字为docker0

- ip netns exec xxns bash # 在xxns空间里面运行bash

- ip link add name veth-ix type veth peer name veth-iy # 创建一对veth设备: veth-ix, veth-iy

- ip link set veth-iy netns xxns # 将veth-iy端加入到xxns空间中，你可以通过ip link set veth-iy name eth0命令在该空间中对网卡进行重命名

- ip netns exec xxns ifconfig veth-iy 192.168.1.2 netmask 255.255.255.0 broadcast 192.168.1.255 # 配置并启动veth-iy设备

- brctl addbr xxbr # 创建名为xxbr的网桥

- brctl addif xxbr veth-iy # 将veth-ix设备加入xxbr网桥中，进行桥接

- brctl show # 使用该命令查看接入网桥的设备

- ifconfig xxbr 192.168.1.1 netmask 255.255.255.0 broadcast 192.168.1.255 # 给xxbr网桥设置ip

- ip netns exec xxns route add default gw 192.168.1.1 # 在xxns空间设置网关(网桥ip)

- ifconfig veth-ix 0.0.0.0 # 桥接模式必要的操作，注意不要遗忘，接几个设备，设置几个设备

- route -n # 查看本机的路由表是否有异常(这里自己确认)

- ping 192.168.1.1 && ping 192.168.1.2 && ip netns exec xxns ping 192.168.1.2 # 假设这是本机ip，这里应该都能ping通，如果不通，那我就不知道你那边啥情况了O_O．

- # 当前应是，主机能连接到xxns这个网络中，xxns网络中的程序能连接主机；接下来，将主机外部网络接入xxns该网络空间．

- sysctl -w net.ipv4.ip_forward=1 # Linux默认都是关闭端口转发的，只有作为路由器才打开．

- # 下面两个操作，配置同一主机上不同网卡设备的转发，假设主机网卡叫eno1;添加iptables FORWARD规则，并启动路由转发功能

- iptables -A FORWARD --out-interface eno1 --in-interface xxbr -j ACCEPT

- iptables -A FORWARD --in-interface eno1 --out-interface ntw2-br -j ACCEPT

- # 添加iptables NAT规则;忘记NAT叫什么了吗?Network Address Translation，网络地址转换，路由器很重要的一部分

- iptables -t nat -A POSTROUTING --source 192.168.1.0/24 --out-interface eno1 -j MASQUERADE

- # 好了，完成！

- ip netns exec xxns ping 192.168.0.2 # ping通另一个电脑，同时接入外网．

- 支持的话，给个star吧！~_~

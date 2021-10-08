简介：本文介绍了openswan连接AWS Site to Site VPN的过程供参考。

前置知识：网络路由、网络协议、IPSec VPN、Linux系统等。请自行学习。

# AWS Site to Site VPN 介绍

[AWS site to site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html) 是用来连接私有IDC网络与AWS VPC的一种低成本方案。其中AWS端由AWS提供托管服务，客户段可以采用Cisco等路由器设备，也可以采用Openswan这类软件VPN搭建。本文主要介绍Openswan软件类VPN方式。关于其他VPN设备可以参考相关设备文档。

# 搭建环境介绍

VPN环境如图所示：

![VPN 环境](https://docs.aws.amazon.com/zh_cn/vpn/latest/s2svpn/images/vpn-how-it-works-vgw.png)

`Virtual Private Gateway` 是AWS VPC 的边界路由器，`Customer Gateway` 是 On-premises的边界路由器。

其中 AWS 侧 VPC CIDR为 `172.31.0.0/16`， on-premises 侧 CIDR 为 `10.50.0.0/16`。

为了方便测试，本次on-premises采用不同region的EC2 代替。 


# AWS侧配置

## 创建Customer Gateway

AWS Console配置的`Customer Gateway (cgw)` 其实是on-premises设备信息的"注册"，在AWS上登记on-premises路由器信息。 所以创建过程很简单。只需要提供IP或者证书、路由类型。

因为我们用EC2搭建openswan，而openswan暂不支持BGP。 所以路由类型我们选择`Static`，IP Address填入 EC2的IP地址。

![CGW create](https://github.com/kealiu/kealiu.github.io/raw/master/images/2021-10-08-AWS%20Site%20to%20Site%20VPN%20with%20on-premise%20Openswan%20or%20libreswan/cgw-create.png)

Tips: 这里的IP地址填入后就不能更改。建议采用可长期使用的IP地址（如AWS上的EIP），不要采用动态分配的地址（如ADSL分配的公网地址）。

## 创建 Virtual Private Gateway

`Virtual Private Gateway (vgw)` 是 VPC的边界路由器。我们在创建完毕后需要`attach`到某一个AWS VPC。过程如下

- 创建vgw

![创建vgw](https://github.com/kealiu/kealiu.github.io/raw/master/images/2021-10-08-AWS%20Site%20to%20Site%20VPN%20with%20on-premise%20Openswan%20or%20libreswan/vgw-create.png)

- attach到vpc

![attach到vpc](https://github.com/kealiu/kealiu.github.io/raw/master/images/2021-10-08-AWS%20Site%20to%20Site%20VPN%20with%20on-premise%20Openswan%20or%20libreswan/vgw-attach-to-vpc.png)

在弹出的对话框中选择你的目标VPC。 注意：一个VPC仅可attach 一个vgw。

## 创建site to site VPN connection

在AWS VPC console中，找到`Site to Site VPN`，然后新建。内容如下。其中vgw/cgw 选择之前新建的。

![vpn-create-1](https://github.com/kealiu/kealiu.github.io/raw/master/images/2021-10-08-AWS%20Site%20to%20Site%20VPN%20with%20on-premise%20Openswan%20or%20libreswan/vpn-create-1.png)
![vpn-create-1](https://github.com/kealiu/kealiu.github.io/raw/master/images/2021-10-08-AWS%20Site%20to%20Site%20VPN%20with%20on-premise%20Openswan%20or%20libreswan/vpn-create-2.png)

VPN创建好后，可以在VPC detail 页面，看到两个Tunnel的公网IP。

## 下载VPN配置文件

AWS为了方便on-premise配置，提供了大量现成的配置文件，一般情况下，只需要选择对应的配置文件，下载后无脑配置即可。

![vpn-download-config](https://github.com/kealiu/kealiu.github.io/raw/master/images/2021-10-08-AWS%20Site%20to%20Site%20VPN%20with%20on-premise%20Openswan%20or%20libreswan/vpn-download-config.png)
![vpn-download-config](https://github.com/kealiu/kealiu.github.io/raw/master/images/2021-10-08-AWS%20Site%20to%20Site%20VPN%20with%20on-premise%20Openswan%20or%20libreswan/vpn-download-config-1.png)

## 修改路由

为了在VPN可用的时候，路由表能够自动更新，需要在VPC内相应的 subnet 路由表中启用路由传播。如图

![routtable-propagation](https://github.com/kealiu/kealiu.github.io/raw/master/images/2021-10-08-AWS%20Site%20to%20Site%20VPN%20with%20on-premise%20Openswan%20or%20libreswan/routetable-propagation.png)

# On-premise 配置

## 安装openswan/libreswan

[Openswan](https://openswan.org) 没有提供官方的Centos/Ubuntu发行版。需要自行下载、编译。具体过程可以参考官方文档。

为了简单起见，我们安装了openswan的分支 [libreswan](https://libreswan.org/)。`libreswan` 有官方的yum/apt 源，可以快速、可靠安装。如过是ubuntu：

```
sudo apt-get update
sudo apt-get install -y libreswan

# 检查安装结果
ipsec --version
```

## 配置IPSec

打开AWS上下载的openswan配置文件，可以看到两个Tunnel的配置。两个Tunnel是为了实现高可靠性，我们的测试没有同时启用两个，仅采用一个Tunnel。所以可以任选一个Tunnel，按文档进行配置。

1. 编辑 `/etc/sysctl.conf`，在文件末尾添加:
    ```
    net.ipv4.ip_forward = 1
    net.ipv4.conf.default.rp_filter = 0
    net.ipv4.conf.default.accept_source_route = 0
    ```
1. 执行 `sysctl -p`，让配置生效
1. 编辑 `/etc/ipsec.conf`，确保包含如下内容
    ```
    include /etc/ipsec.d/*.conf
    ```
1. 编辑 `/etc/ipsec.d/aws.conf`，按AWS上下载的文档，复制&粘贴配置。然后修改部分配置文件:
    1. `leftsubnet=`后面填入on-premise的CIDR（本例中的`10.50.0.0/16`) ；
    1. `rightsubnet=`填入AWS VPC CIDR （本例中的`17.31.0.0/16`) 。所以，配置中的`left`都代表本地，`right`都代表远端。
    1. 特别提醒：最新版本的openswan不再需要 `auth=esp`内容，请确保配置不包含该内容。
    ```
    conn Tunnel1
    	authby=secret
    	auto=start
    	left=%defaultroute
    	leftid=1.1.1.1
    	right=2.2.2.2
    	type=tunnel
    	ikelifetime=8h
    	keylife=1h
    	phase2alg=aes128-sha1;modp1024
    	ike=aes128-sha1;modp1024
    	keyingtries=%forever
    	keyexchange=ike
    	leftsubnet=<LOCAL NETWORK>
    	rightsubnet=<REMOTE NETWORK>
    	dpddelay=10
    	dpdtimeout=30
    	dpdaction=restart_by_peer
    ```
1. 编辑 `/etc/ipsec.d/aws.secrets` ，按AWS下载的配置文件填入内容:
    ```
    1.1.1.1 2.2.2.2: PSK "PSWD_**********"
    ```

## 启用IPSec VPC

为了确保连接正常，请确保机器UDP 端口500 (如果采用NAT穿透，还需端口 4500) 双向开放。
```
sudo systemctl enable ipsec.service
sudo systemctl start ipsec.service

# 检查运行状态
sudo systemctl status ipsec.service
```

## on-premise 路由更新

为了让前往AWS VPC (`172.31.0.0/16`)的网络流量都走VPN，需要更新路由表。由于我们的测试环境使用的AWS VPC，所以我们更新了VPC路由表，添加 `172.31.0.0/16` 指向我们的openswan EC2。

# 测试验证

首先，确保两边安全组/防火墙都开启了 `ICMP`。 然后，再AWS VPC以及on-premise启动机器，互相`ping`。 可以看到都能`ping`通。

# References
- [AWS Site to Site VPN 快速启动](https://docs.aws.amazon.com/vpn/latest/s2svpn/SetUpVPNConnections.html#vpn-configure-routing)
- [VPN静态路由及路由传播](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-edit-static-routes.html)
- [VPN端口需求](https://aws.amazon.com/premiumsupport/knowledge-center/vpn-tunnel-phase-1-ike/)
- [Openswan](https://openswan.org/)
- [libreswan](https://libreswan.org/)


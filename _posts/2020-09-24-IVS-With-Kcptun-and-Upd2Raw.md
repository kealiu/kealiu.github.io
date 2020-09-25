简介： 介绍AWS IVS服务，并DEMO如何推流到IVS。结合国内网络情况，介绍一种基于KCPTUN双向加速的网络推流加速方案。

# AWS IVS 服务

## 简介

Amazon Interactive Video Service（IVS）是一项AWS托管的实时视频流服务，技术源于Twitch游戏直播平台，可为用户提供：

- 简单易用的创建直播频道，并马上开始视频直播。
- 通过超低延迟的视频直播技术，打造极致的直播互动体验。
- 将直播视频分发到各大视频直播平台，以及电脑、手机等各种设备。
- 方便的集成到自己的网站或者应用程序。

Amazon IVS让用户专注于构建自己的交互式直播应用程序，提供极致用户体验。 借助Amazon IVS，用户无需管理基础设施或开发、配置视频相关的应用组件，即可实现安全、可靠、成本低廉的直播服务。

Amazon IVS支持RTMPS流。 RTMPS是RTMP（实时消息协议）的基于TLS/SS的安全升级版本。 RTMP是广泛用于视频网络传输的行业标准协议之一。

## 创建IVS服务

登陆AWS控制台，查找IVS，进入IVS控制台。可以直接点击`Create Channle`进行频道创建。

![新建IVS流](https://github.com/kealiu/kealiu.github.io/raw/master/images/2020-09-24-IVS-With-Kcptun-and-Upd2Raw/CreateNew.png)

参数很简单，就是名字、高清（HD）还是标清（SD）、极低时延还是标准时延、是否需要认证。测试场景天下名字保留默认即可。

## 测试验证

在IVS控制台，点开对应的IVS `channel` ，可以看到具体的推流地址以及推流key

![IVS 配置信息](https://github.com/kealiu/kealiu.github.io/raw/master/images/2020-09-24-IVS-With-Kcptun-and-Upd2Raw/IVSConfiguration.png)

基于MacBook进行直播测试。由于国内网络环境，`ultra-low`极低时延很难连接上。需要使用 `Basic` SD标清 及 `Standard` 正常时延来保证稳定视屏流进行测试验证。

### ffmpeg 推流

注意替换下面命令中的`aws-ivs-ingest.global-contribute.live-video.net` 和 `sk_token_here` 替换为自己`channel`的 `ingest server` 及 `Stream key`值

具体参数的意义以及取值请参考ffmpeg官网文档。 

```
sudo ffmpeg -f avfoundation -framerate 30 -i "0" -c:v libx264 -b:v 6000K -maxrate 6000K -pix_fmt yuv420p -s 1920x1080 -profile:v main -preset veryfast -g 120 -x264opts "nal-hrd=cbr:no-scenecut" -acodec aac -ab 160k -ar 44100 -f flv -flvflags no_duration_filesize rtmps://aws-ivs-ingest.global-contribute.live-video.net:443/app/sk_token_here
```

注意，由于AWS IVS 采用了 `rtmps`协议，`ffmpeg`必须是支持  TLS/SSL 的版本。请注意编译参数。

### LiveStream 验证

通过打开LiveStream可以实现stream的实时监控，及时了解视频流情况。 注意，考虑时延时，这里的监控视频流，与自己所处的网络环境有关，不代表真实用户的时延。

![LiveStream](https://github.com/kealiu/kealiu.github.io/raw/master/images/2020-09-24-IVS-With-Kcptun-and-Upd2Raw/LiveStream.png)

中国境内本次测试大概 20s 左右的时延。毕竟，推流到海外、再拉流会境内，时延很难理想。

## IVS价格

IVS价格包含两部分：`channel`租用费，以及直播流量费。具体参考[AWS IVS 定价 官网](https://aws.amazon.com/cn/ivs/pricing/)

### `channel`租用费

基本频道类型的最大输入为 1.5 Mbps 和 480p 分辨率 (SD)。标准频道的最大输入则是 8.5 Mbps 和 1080p 分辨率 (full HD)。

频道类型 | 每小时价格
---|---
标准 | 2.00 USD
基本 | 0.20 USD

### 直播流量费

针对直播视频输出，用户要为 IVS 将直播视频传输到观众的持续时间付费。

注意：账单区域指的是直播视频的传输起点位置，而不是观众所在位置。

|  分辨率 |   传输时数 每月   |    北 美   |    欧洲    |    巴西   | 日本、 中国香港 新加坡， 和泰国 |  中国台湾 |    韩国    |  澳大利亚 |
|:-------:|:-----------------:|:----------:|:----------:|:---------:|:-------------------------------:|:---------:|:----------:|:---------:|
|    SD   |  前 10000 个小时  | 0.0375 USD | 0.0375 USD | 0.060 USD |            0.065 USD            | 0.080 USD |  0.125 USD | 0.085 USD |
|         | 超过 10000 个小时 |  0.035 USD |  0.035 USD | 0.055 USD |            0.060 USD            | 0.075 USD | 0.1175 USD | 0.080 USD |
|    HD   |  前 10000 个小时  |  0.075 USD |  0.075 USD | 0.120 USD |            0.130 USD            | 0.160 USD |  0.250 USD | 0.170 USD |
|         | 超过 10000 个小时 |  0.070 USD |  0.070 USD | 0.110 USD |            0.120 USD            | 0.150 USD |  0.235 USD | 0.160 USD |
| Full HD |  前 10000 个小时  |  0.150 USD |  0.150 USD | 0.240 USD |            0.260 USD            | 0.320 USD |  0.500 USD | 0.340 USD |
|         | 超过 10000 个小时 |  0.140 USD |  0.140 USD | 0.220 USD |            0.240 USD            | 0.300 USD |  0.470 USD | 0.320 USD |

# KCPTUN 加速方案

注意：如果能正常推流，时延满意，不需要使用`KCPTUN`代理进行加速优化

## KCPTUN简介

`Kcptun` 是一个非常简单和快速的，基于 KCP 协议的 UDP 隧道，它可以将 TCP 流转换为 KCP+UDP 流。而 KCP 是一个快速可靠协议，能以比 TCP 浪费10%-20%的带宽的代价，换取平均延迟降低 30%-40%，且最大延迟降低三倍的传输效果。

`Kcptun` 是 KCP 协议的一个简单应用，可以用于任意 TCP 网络程序的传输承载，以提高网络流畅度，降低掉线情况。由于 Kcptun 使用 Go 语言编写，内存占用低（经测试，在64M内存服务器上稳定运行），而且适用于所有平台，甚至 Arm 平台。


![KCPTUN原理图](https://github.com/xtaci/kcptun/raw/master/kcptun.png)

更多信息请访问 [kcptun github](https://github.com/xtaci/kcptun) 了解。

## 搭建方式

以下`kcptun`搭建配置仅供参考，由于每个人所处的网络环境千差万别，适合自己的参数必须要自己尝试调整。

`kcptun` 需要额外增加一个带公网IP的代理节点，用做中间转发节点。我们假设代理节点的公网IP为  `5.5.5.5`。同时，假设本地代理IP为 `1.1.1.1`


### 服务端 (IP `5.5.5.5`)

注意替换下面命令中的`aws-ivs-ingest.global-contribute.live-video.net` 和 `KcpTunToken` 替换为自己`channel`的 `ingest server` 和 自己的 `kcptun` 密码

- 配置文件 kcp.conf

```json
{
  "listen": "0.0.0.0:55555",
  "target": "aws-ivs-ingest.global-contribute.live-video.net:443",
  "key": "KcpTunToken",
  "crypt": "none",
  "mode": "fast",
  "mtu" : 1350,
  "sndwnd" : 2048,
  "rcvwnd" : 1024,
  "datashard":10,
  "parityshard":5,
  "sockbuf" : 16777215
}
```

- EC2 安全组

安全组开放 `custom udp` 协议的 `55555` 端口

- 运行

`kcptun-server -c kcp.conf`

### 客户端 (IP: `1.1.1.1`)

- 配置文件 kcp.conf

注意替换下面命令中的 `KcpTunToken` 替换为 自己的 `kcptun` 密码

```
{
  "localaddr":"0.0.0.0:443",
  "remoteaddr":"5.5.5.5:55555",
  "key":"KcpTunToken",
  "crypt":"none",
  "mode":"fast2",
  "mtu":1350,
  "sndwnd":1024,
  "rcvwnd":2048,
  "sockbuf":16777215
}
```

- 运行

`kcptun-client -c kcp.conf`

### IVS 更新

注意下文中的 `aws-ivs-ingest.global-contribute.live-video.net` 替换为 自己的 `ingest server` 地址

修改  `/etc/hosts`, 添加一行：

`1.1.1.1	aws-ivs-ingest.global-contribute.live-video.net`

然后推流测试。 

```
sudo ffmpeg -f avfoundation -framerate 30 -i "0" -c:v libx264 -b:v 6000K -maxrate 6000K -pix_fmt yuv420p -s 1920x1080 -profile:v main -preset veryfast -g 120 -x264opts "nal-hrd=cbr:no-scenecut" -acodec aac -ab 160k -ar 44100 -f flv -flvflags no_duration_filesize rtmps://aws-ivs-ingest.global-contribute.live-video.net:443/app/sk_token_here
```

通过 kcptun 加速后，本次测试大概是要达到 10s 左右，远好与无加速情况下的时延。

## UDP2RAW 优化

由于网络运营商往往对UDP数据包进行乐限流、封锁等操作，如果`kcptun` 连接不上或者性能极差，可以进一步通过`udp2raw` 来将`kcptun` 的udp流量伪装成TCP流量来实现性能提升。

注意，`udp2raw`不是必须。如果`kcptun`能正常使用，则完全没有必须添加`udp2raw`

### UDP2RAW简介

`udp2raw` 通过raw socket给UDP包加上TCP或ICMP header，进而绕过UDP屏蔽或QoS，或在UDP不稳定的环境下提升稳定性。可以有效防止在使用`kcptun`或者`finalspeed`的情况下udp端口被运营商限速。支持心跳保活、自动重连，重连后会恢复上次连接，在底层掉线的情况下可以保持上层不掉线。同时有加密、防重放攻击、信道复用的功能。


![UDP2RAW原理图](https://github.com/wangyu-/udp2raw-tunnel/raw/unified/images/image0.PNG)

更多信息请访问 [udp2raw github](https://github.com/wangyu-/udp2raw-tunnel) 了解。


以下`udp2raw`搭建配置仅供参考，由于每个人所处的网络环境千差万别，适合自己的参数必须要自己尝试调整。


### 服务端 (IP `5.5.5.5`)


- 配置文件 udp2raw.conf

注意替换下面命令中的 `Udp2RawToken` 替换为 自己的 `udp2raw` 密码

```
-s
-k Udp2RawToken
-l 0.0.0.0:56666
-r 5.5.5.5:55555
--raw-mode faketcp
--cipher-mode none
-a
```

- 安全组

安全组开放 `custom tcp` 协议的 `56666` 端口

- 运行

`udp2raw --conf-file udp2raw.conf`

### 客户端 (IP `1.1.1.1` )

- 配置文件udp2raw.conf 

注意替换下面命令中的 `Udp2RawToken` 替换为 自己的 `udp2raw` 密码

```
-c
-k Udp2RawToken
-l 0.0.0.0:56666
-r 5.5.5.5:56666
--raw-mode faketcp
--cipher-mode none
```

- 运行

`udp2raw --conf-file udp2raw.conf`

### KCPTUN 配置更新

因为kcptun流量需要经过`udp2raw`来进行TCP伪装，所以， `kcptun` 的`remoteaddr` 将配置为刚刚配置的 `udp2raw` 服务监听地址。
```
{
  "localaddr":"0.0.0.0:443",
  "remoteaddr":"1.1.1.1:56666",
  "key":"KcpTunToken",
  "crypt":"none",
  "mode":"fast2",
  "mtu":1350,
  "sndwnd":1024,
  "rcvwnd":2048,
  "sockbuf":16777215
}
```

### 测试验证

`ffmpeg` 命令不用改，只需要修改 `/etc/hosts` 域名指向即可。

# 总结

本文介绍来AWS IVS服务并进行了DEMO，来实现时延的直播。可以看到，IVS非常简单易用，而且使用的标准协议，方便各类设备接入。 

然后，考虑到国内到海外的时延情况，介绍了通过KCPTUN/UDP2RAW双向加速的方式，对服务进行来优化提升性能。


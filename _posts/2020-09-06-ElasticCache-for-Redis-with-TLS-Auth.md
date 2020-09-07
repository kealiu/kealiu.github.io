# AWS ElasticCache for Redis

## AWS ElasticCache for Redis 介绍

官方文档介绍
> Amazon ElastiCache 是一种Web服务，可轻松地在云中设置、管理和扩展分布式内存数据存储或缓存环境。它提供高性能、可扩展且具有成本效益的缓存解决方案。同时，它有助于消除部署和管理分布式缓存环境的复杂性。
> 现有的使用 Redis 的应用程序可以几乎不进行任何修改就使用 ElastiCache。您的应用程序只需要有关您已部署的 ElastiCache 节点的主机名和端口号的信息。

> Amazon ElastiCache for Redis 是速度超快的内存数据存储，能够提供亚毫秒级延迟来支持 Internet 范围内的实时应用程序。适用于 Redis 的 ElastiCache 基于开源 Redis 构建，可与 Redis API 兼容，能够与 Redis 客户端配合工作，并使用开放的 Redis 数据格式来存储数据。自我管理型 Redis 应用程序可与适用于 Redis 的 ElastiCache 无缝配合使用，无需更改任何代码。适用于 Redis 的 ElastiCache 兼具开源 Redis 的速度、简单性和多功能性与 Amazon 的可管理性、安全性和可扩展性，能够在游戏、广告技术、电子商务、医疗保健、金融服务和物联网领域支持要求最严苛的实时应用程序。

## AWS Redis 创建 

![Create ElasticCache for Redis](https://github.com/kealiu/kealiu.github.io/raw/master/images/2020-09-03-Redis/createNewRedis.png)

首先，选择Redis类型，默认是非集群版本。如果要选用集群模式，请勾选cluster mode。

> 集群 与 非集群 区别简介
> - 非集群：一主多从结构，支持多数据库
> - 集群：为了解决Redis容量问题，支持数据分片，不支持多数据库

创建需要填写对应的Redis名称，可选的描述。其他的Redis版本，端口，参赛组一般默认即可。 

对于实例类型，后期可以调整，可以根据现在的实际需求合理选择。

副本数一般选择2，实现3副本。

![Advanced and Security option](https://github.com/kealiu/kealiu.github.io/raw/master/images/2020-09-03-Redis/advancedAndSecurity.png)

高级设置中，可以选择子网以及放置的AZ。

安全选项种，可以选择对应的安全组，请确保能让client访问到redis的对应端口。

如果期望加密存储，请选择Encrytion at-rest。并选择加密使用的key。可以由AWS默认提供，也可以自己选择对应的key。

如果期望传输加密，请选择Encrytion in-transit，AWS将启用TLS加密传输。同时，启用加密传输的前提下，还可以进一步添加token以提升安全性。

![Data Related Options](https://github.com/kealiu/kealiu.github.io/raw/master/images/2020-09-03-Redis/dataRelated.png)
在新建Redis时，可以选择导入已有的dump数据到Redis。导入的dump数据文件应该已经存在于S3，并且使用 `bucketName/path/to/dump/file` 的格式引用。

为了数据安全，我们一般做定期备份。我们可以选择备份间隔与备份时间。

最后，我们还可以选择维护（一般是升级）窗口，以及通知机制（使用SNS进行通知）

## 连接到AWS ElasticCache for Redis

点击redis事例查看详情。可以看到连接串。一般为 `master.name.user.region.cache.amazonaws.com.cn:6379`。

我们通过redis-cli来验证我们的联通性。登录到一台可以访问该redis的机器，安装redis-tools，然后通过redis-cli来进行访问。 以ubuntu为例

```
sudo apt-get update
sudo apt-get instsall -y redis-tools
redis-cli -h master.replace.me.with.the.right.region.cache.amazonaws.com.cn
```

如成功登陆，就可以开始正常是用redis了。

## AWS Redis 价格

[AWS Elasticsearch 价格表](https://aws.amazon.com/elasticache/pricing/)。与AWS EC2类似，基于实例类型进行收费。同样也提供RI实例。

# AWS ElasticCache for Redis 安全

AWS 负责基础设施以及Redis本身的安全，但是使用安全由用户负责。具体来说，有以下几个方面的安全：
1. 传输加密，即通过TLS/SSL来进行加密。
2. 连接认证，即Redis AUTH功能
3. 存储加密，即落盘时加密。 

## AWS Redis 启用 TLS

传输中加密是可选功能，只能在创建 Redis 复制组时在复制组中启用。

通过传输中加密功能，ElastiCache 为您提供了在不同位置之间移动数据时用来保护数据的工具。由于在终端节点加密和解密数据时需要进行一些处理，因此启用传输中加密会对性能产生一些影响。应对使用和不使用传输中加密的数据进行基准测试，以确定对使用案例的性能影响。


## AWS Redis 启用 Auth

Redis AUTH 是Redis的原生功能。如果要开通Redis AUTH，必须启用TLS。

开启了AUTH的话，在创建的时候需要设置Token（密码），在每次连接 Redis 服务器时，要先使用 AUTH 命令进行认证。认证之后才能使用其他 Redis 命令。

如果 AUTH 命令给定的密码 token 和创建Redis时的token相符的话，服务器会返回 OK 并开始接受命令输入。

另一方面，假如token不匹配的话，服务器将返回一个错误，并要求客户端需重新输入token。

## AWS Redis jedis 实例

jedis 是 Redis 官方推荐的的 Java 客户端开发包，主要是对Redis命令进行砺封装。

Jedis使用阻塞的I/O，且其方法调用都是同步的，程序流需要等到sockets处理完I/O才能执行，不支持异步。同时不是线程安全的，所以推荐使用连接池来使用Jedis。
 
### 搭建简单的java环境

我们通过`redis-cli` 以及 简单的java code 来验证 Elasticcache Redis 的TLS/AUTH功能。

#### 安装依赖

```
# 安装redis-tools 以及 stunnel4(TSL/SSL 代理)
apt-get install redis-tools stunnel4 -y

# install java deps
apt-get install openjdk-8-jdk maven -y

```

#### 通过`redis-cli`验证

由于`redis-cli` 不支持TLS，需要借用 `stunnel4` TSL 代理来进行下中转。在此不对`stunnel4` 进行细节介绍，仅仅过一下怎么搭建。

主要是修改stunnel4的配置文件，添加一个`[redis-cli]`代理项。注意把`connect`的值 改为对应的 ElasticCache for Redis 对应的 `hostname` 与 `redis` 端口

```
sudo vi /etc/stunnel/stunnel.conf

    fips = no
    setuid = nobody
    setgid = nogroup
    pid = /var/run/pids/stunnel.pid
    debug = 7
    delay = yes
    [redis-cli]
      client = yes
      accept = 127.0.0.1:6379
      connect = managed_redis_hostname_or_ip:managed_redis_port

sudo systemctl restart stunnel4
```

然后，直接通过 `redis-cli` 进行连接。

```
redis-cli -h localhost
redis>auth <token-here>
OK
```

### 构建并运行测试

本节我们通过简单的java code来进行测试。相关的代码可以从`github` 获取


```

git clone https://github.com/kealiu/redisauthtls.git

cd redisauthtls

# 修改代码，src/main/java/com/mycompany/app/App.java 
vi src/main/java/com/mycompany/app/App.java 
# 把 `master.of.redis` 以及 `token`  改为正确的值

# build
mvn dependency:go-offline
mvn package

```

构建完成后，即可以与`jedis` client一起运行测试。

```
java -cp ~/.m2/repository/redis/clients/jedis/3.3.0/jedis-3.3.0.jar:target/my-app-1.0-SNAPSHOT.jar com.mycompany.app.App
```

## 总结

Amazon ElastiCache for Redis 可与开源 Redis 数据格式、Redis API 兼容，并与 Redis 客户端配合使用。您可以将自我管理型 Redis 工作负载迁移到 ElastiCache for Redis，而无需更改任何代码。

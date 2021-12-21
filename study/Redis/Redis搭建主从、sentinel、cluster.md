# Table of Contents

* [Redis搭建主从、sentinel、cluster](#redis搭建主从、sentinel、cluster)
    * [环境准备](#环境准备)
    * [主从复制搭建](#主从复制搭建)
        * [配置](#配置)
        * [测试](#测试)
    * [sentinel搭建](#sentinel搭建)
        * [配置](#配置-1)
        * [测试](#测试-1)
    * [cluster搭建](#cluster搭建)
        * [配置](#配置-2)
        * [测试](#测试-2)


# Redis搭建主从、sentinel、cluster

## 环境准备

**系统**：MacOS Version:10.15.7

**Redis版本**：6.2.5

[Redis下载地址](https://redis.io/download)

安装Redis
```
// 进入解压完的Redis目录
make
```

安装完后将`src`目录的`redis-cli`拷贝至`/usr/local/bin`实现全局`redis-cli`指令，`redis-server`和`redis-sentinel`同理


## 主从复制搭建

### 配置

将Redis根目录下的`redis.conf`文件拷贝两份分别取名为`redis6380.conf`和`redis6381.conf`用来测试。

配置内容文本太大，拎出其中需要改动的几点讲解（以默认配置改成redis6380.conf为例，其他端口的对应配置）：

```
port 6379 -> port 6380

pidfile /var/run/redis_6379.pid -> pidfile /var/run/redis_6380.pid

// 具体数据存放处自己定
dir ./ -> dir /Users/zhuweijie/app/redis-data/6380/
```

以两个配置文件分别启动redis，命令如下（以6380为例）：

```
redis-server redis6380.conf
```

通过如下命令分别访问两个redis server:

```
redis-cli -p 6380
```

通过`info`命令查看两个server的信息（两个现在都是master）：

![](http://img.yelizi.top/36ef7e3b-57e6-4ce3-a1d2-e7f82c0d728c.jpg$xyz)

在6381下执行如下命令做从server连接到6380：

```
slaveof 127.0.0.1 6380
```

再来看info，6380显示有一个从库，6381已经显示为从库并且连接到6380：

![](http://img.yelizi.top/901e79a6-9131-4a79-95d2-27c1571859ff.jpg$xyz)


### 测试

在6380执行`set a 1`，在6381执行`get a`：

```
// 6380 
127.0.0.1:6380> set a 1
OK

// 6381
127.0.0.1:6381> get a
"1"
```

在6381直接执行`set b 2`：

```
127.0.0.1:6381> set b 2
(error) READONLY You can't write against a read only replica.
```

测试断开6380，6381开始不断打印如下日志（未实现高可用）：

```
43616:S 06 Sep 2021 13:56:09.662 * MASTER <-> REPLICA sync started
43616:S 06 Sep 2021 13:56:09.662 # Error condition on socket for SYNC: Connection refused
43616:S 06 Sep 2021 13:56:10.694 * Connecting to MASTER 127.0.0.1:6380
43616:S 06 Sep 2021 13:56:10.694 * MASTER <-> REPLICA sync started
43616:S 06 Sep 2021 13:56:10.694 # Error condition on socket for SYNC: Connection refused
43616:S 06 Sep 2021 13:56:11.731 * Connecting to MASTER 127.0.0.1:6380
43616:S 06 Sep 2021 13:56:11.731 * MASTER <-> REPLICA sync started
43616:S 06 Sep 2021 13:56:11.731 # Error condition on socket for SYNC: Connection refused
43616:S 06 Sep 2021 13:56:12.758 * Connecting to MASTER 127.0.0.1:6380
43616:S 06 Sep 2021 13:56:12.758 * MASTER <-> REPLICA sync started
43616:S 06 Sep 2021 13:56:12.758 # Error condition on socket for SYNC: Connection refused
```


## sentinel搭建

### 配置

定义两个个sentinel的配置文件，分别叫`sentinel23679.conf`和`sentinel23680.conf`（配置10s失联就切换主从）：

```
port 26379
sentinel monitor mymaster 127.0.0.1 6380 2
sentinel down-after-milliseconds mymaster 10000
sentinel failover-timeout mymaster 30000
sentinel parallel-syncs mymaster 1
```

启动命令如下（其他端口启动类似）：

```
redis-sentinel sentinel23679.conf
```

### 测试

当前6380是主库，6381是从库，依旧直接断开6380看sentinel的效果，6381的server控制台日志：

```
43616:S 06 Sep 2021 14:31:38.776 * Connecting to MASTER 127.0.0.1:6380
43616:S 06 Sep 2021 14:31:38.776 * MASTER <-> REPLICA sync started
43616:S 06 Sep 2021 14:31:38.776 # Error condition on socket for SYNC: Connection refused
43616:M 06 Sep 2021 14:31:39.720 * Discarding previously cached master state.
43616:M 06 Sep 2021 14:31:39.720 # Setting secondary replication ID to cce5e3acc9de15d1348e6e8f7a9d3067525b7001, valid up to offset: 29991. New replication ID is dc27a0ceb725101cd90d616ce72e20c6f3fe4b72
43616:M 06 Sep 2021 14:31:39.720 * MASTER MODE enabled (user request from 'id=22 addr=127.0.0.1:64085 laddr=127.0.0.1:6381 fd=12 name=sentinel-63e99348-cmd age=62 idle=0 flags=x db=0 sub=0 psub=0 multi=4 qbuf=188 qbuf-free=65342 argv-mem=4 obl=45 oll=0 omem=0 tot-mem=82980 events=r cmd=exec user=default redir=-1')
43616:M 06 Sep 2021 14:31:39.724 # CONFIG REWRITE executed with success.
^@ multi=4 qbuf=188 qbuf-free=65342 argv-mem=4 obl=45 oll=0 omem=0 tot-mem=82980 events=r cmd=exec user=default redir=-1')
```

连接主库10s失败后被切换为主库，查看info（身份已经转变为master）：

![](http://img.yelizi.top/f839e9b2-4603-4e7b-83ad-cba80740c1dd.jpg$xyz)


再重启6380看看是什么效果：

![](http://img.yelizi.top/e4137208-2341-4c31-b266-4fa48f3c8a31.jpg$xyz)

由于6381已经是主库，6380重启自动转变为从库，至此，实现了高可用。



## cluster搭建

### 配置

另起三个端口测试集群（集群最少3个主库），6382和6383和6384。进行如下配置：

```
port 6379 -> port 6382

pidfile /var/run/redis_6379.pid -> pidfile /var/run/redis_6382.pid

// 具体数据存放处自己定
dir ./ -> dir /Users/zhuweijie/app/redis-data/6382/

cluster-enabled yes
cluster-config-file nodes-6382.conf
cluster-node-timeout 15000
```

分别启动三个服务。启动后和正常启动有点不一样，多了个cluster标志。

![](http://img.yelizi.top/f87c2e8f-d537-4b97-9c8e-06e160d8146a.jpg$xyz)


启动完后通过如下命令创建分片：

```
redis-cli --cluster create 127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384
```

展示如下就是分片完成了：

```
>>> Performing hash slots allocation on 3 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
M: f0d19e19abd3e4a4e5d591f161cc9f6401f60715 127.0.0.1:6382
   slots:[0-5460] (5461 slots) master
M: 30d25ea7930f3c39cf20747b11e3478ec7126d3d 127.0.0.1:6383
   slots:[5461-10922] (5462 slots) master
M: 2c5f36d8ba4cb75009c70f33e21f99bff330996c 127.0.0.1:6384
   slots:[10923-16383] (5461 slots) master
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 127.0.0.1:6382)
M: f0d19e19abd3e4a4e5d591f161cc9f6401f60715 127.0.0.1:6382
   slots:[0-5460] (5461 slots) master
M: 2c5f36d8ba4cb75009c70f33e21f99bff330996c 127.0.0.1:6384
   slots:[10923-16383] (5461 slots) master
M: 30d25ea7930f3c39cf20747b11e3478ec7126d3d 127.0.0.1:6383
   slots:[5461-10922] (5462 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

这时候去看对应的node.conf也能看出来了（以6382为例）：

```
2c5f36d8ba4cb75009c70f33e21f99bff330996c 127.0.0.1:6384@16384 master - 0 1630913929747 3 connected 10923-16383
30d25ea7930f3c39cf20747b11e3478ec7126d3d 127.0.0.1:6383@16383 master - 0 1630913929019 2 connected 5461-10922
f0d19e19abd3e4a4e5d591f161cc9f6401f60715 127.0.0.1:6382@16382 myself,master - 0 1630913929000 1 connected 0-5460
vars currentEpoch 3 lastVoteEpoch 0
```


### 测试

`redis-cli` 通过 `-c` 启动将支持集群的使用。

如下命令进入：

```
redis-cli -c -p 6382
```

进行测试：

```
127.0.0.1:6382> set aa 11
OK
127.0.0.1:6382> set bb 22
-> Redirected to slot [8620] located at 127.0.0.1:6383
OK
127.0.0.1:6383> keys *
1) "bb"
```

设置 `aa` 分片直接在6382，因此直接OK，但设置 `bb` 时，算出来slot是8620应该分到6383上，因此直接转到6383，并且命令行都直接切换为6383，在6383查看key已经有了 `bb`。


退出，不以`-c`访问，再测试：

```
127.0.0.1:6382> set bb 22
(error) MOVED 8620 127.0.0.1:6383
127.0.0.1:6382>
```

报错，并提示数据分片属于6383，数据的隔离性没有问题。
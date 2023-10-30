# Redis-集群高可用与实践

## 集群注意

集群不支持处理多个键的命令，因为key位于多个键上，性能不佳。不在一个slot下的键值，是不能使用mget,mset等多键操作。lua脚本不被支持。

可以通过{}来定义组的概念，从而使key中{}内相同内容的键值对放到一个slot中去。

```shell
host.docker.internal:6384> mset k3{group} v3 k4{group} v4
OK
host.docker.internal:6385> get k3{group}
-> Redirected to slot [12148] located at host.docker.internal:6384
"v3"
host.docker.internal:6384> get k4{group}
"v4"
host.docker.internal:6384> mget k3{group} k4{group}
1) "v3"
2) "v4"
host.docker.internal:6384> get v3
-> Redirected to slot [9423] located at host.docker.internal:6381
(nil)
host.docker.internal:6381> get v4
-> Redirected to slot [5160] located at host.docker.internal:6385
(nil)
```

Redis 集群目前无法做数据库选择， 默认在 0 数据库。

```shell
# redis-cli -p 6380 -a 1234 -c
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
127.0.0.1:6380> keys *
(empty array)
127.0.0.1:6380> get k1
-> Redirected to slot [12706] located at host.docker.internal:6384
"v1"
host.docker.internal:6384> keys *
1) "k1"
host.docker.internal:6384> mget k1 k2
(error) CROSSSLOT Keys in request don't hash to the same slot
host.docker.internal:6384> mget k1
1) "v1"
host.docker.internal:6384> mget k2
-> Redirected to slot [449] located at host.docker.internal:6385
1) "v2"
```

## 集群节点的划分

节点数量至少为 6 个才能保证组成完整高可用的集群，其中三个为主节点，三个为从节点。三个主节点会分配槽（也就是分片），处理客户端的命令请求，而从节点可用在主节点故障后，顶替主节点。

## 实践

### 准备

镜像：redis:latest

容器网络组：my-bridge

容器配置文件：redis_cluster_container.sh
```shell
for port in $(seq 6380 6385);
do
mkdir -p /Users/gaea/shell/redis/node-${port}/conf
touch /Users/gaea/shell/redis/node-${port}/conf/redis.conf
cat  << EOF > /Users/gaea/shell/redis/node-${port}/conf/redis.conf
port ${port}
requirepass 1234
bind 0.0.0.0
protected-mode no
daemonize no
appendonly yes
cluster-enabled yes
cluster-config-file nodes-${port}.conf
cluster-node-timeout 5000
cluster-replica-validity-factor 0
cluster-announce-ip host.docker.internal
cluster-announce-port ${port}
cluster-announce-bus-port 1${port}
EOF
done
```
为6380-6385端口的6个容器创建redis服务，服务的配置在node-$PORT下的conf。注意cluster-enabled等配置。
- port：节点端口；
- bind: 可访问网络地址 0.0.0.0 指本网络所有ip
- requirepass：添加访问认证；
- masterauth：如果主节点开启了访问认证，从节点访问主节点需要认证；
- protected-mode：保护模式，默认值 yes，即开启。开启保护模式以后，需配置 bind ip 或者设置访问密码；关闭保护模式，外部网络可以直接访问；
- daemonize：是否以守护线程的方式启动（后台启动），默认 no；
- appendonly：是否开启 AOF 持久化模式，默认 no；
- cluster-enabled：是否开启集群模式，默认 no；
- cluster-config-file：集群节点信息文件；
- cluster-node-timeout：集群节点连接超时时间；
- cluster-slave-validity-factor: 如果一个从节点设为0, 无论其与对应的主节点断开连接多长时间, 仍然可以取代主节点成为新的主节点. 如果设为正数, 通过node timeout * factor(该参数设置的值) 得出最大允许断开连接时间(maximum disconnection time). 如果这是一个从节点, 与其对应的主节点断开连接超过这个时间的话, 将不会尝试取代而成为新的主节点. 举个例子, 如果node timeout 设为5秒, factor 设为10, 那这个从节点与其主节点断开连接超过50秒后将不会尝试取代原主节点. 注意这个参数设为任意非0值可能会出现因某主节点失效但没有从节点允许取而代之而导致集群无法运作. 这种情况下集群仅当原主节点恢复正常并重新加入到集群中才能重新提供服务.
- cluster-announce-ip：集群节点 IP，这里需要特别注意一下，如果要对外提供访问功能，需要填写宿主机的 IP，如果填写 Docker 分配的 IP（172.x.x.x），可能会导致外部无法正常访问集群；host.docker.internal是所有容器服务都需要经过宿主机网络中转再去访问其他容器服务，会有些耗费时间，这样是为了避免容器ip不在同一网段就无法访问的问题，也可以通过加入网桥解决。
- cluster-announce-port：集群节点映射端口；
- cluster-announce-bus-port：集群节点总线端口。
　　每个 Redis 集群节点都需要打开两个 TCP 连接。一个用于为客户端提供服务的正常 Redis TCP 端口，例如 6379。还有一个基于 6379 端口加 10000 的端口，比如 16379。


docker常用的网络组管理命令：
```shell
docker network ls
docker network create my-bridge
docker network connect $networkName $containerName
docker network inspect my-bridge
```

### 启动容器与集群

启动容器：redis_cluster_run.sh
```shell
for port in $(seq 6380 6385); \
do \
   docker run -it -d -p ${port}:${port} -p 1${port}:1${port} \
  --privileged=true -v /Users/gaea/shell/redis/node-${port}/conf/redis.conf:/usr/local/etc/redis/redis.conf \
  --privileged=true -v /Users/gaea/shell/redis/node-${port}/data:/data \
  --restart always --name redis-${port} --net my-bridge \
  --sysctl net.core.somaxconn=1024 redis redis-server /usr/local/etc/redis/redis.conf
done
```
容器开放的端口6380-6385用于redis服务，与宿主机映射端口一致。1$PORT是Redis总线端口，不允许外部访问，用于节点间故障迁移等。并将数据挂载到本地的data中，比如rdb、aof等文件。

进入容器：
```shell
docker exec -it redis-6380 /bin/sh
```

创建集群：
```shell
redis-cli  -a 1234 --cluster create host.docker.internal:6380 host.docker.internal:6381 host.docker.internal:6382 host.docker.internal:6383 host.docker.internal:6384 host.docker.internal:6385   --cluster-replicas 1
```
每个主节点有一个副本节点，当主宕机后将由从替代，主恢复后也只能作为新主的从节点。

注意，这种集群模式生成的集群，slave关系已经创建，并且初始化了主备节点和相互通信等配置，碰到主故障时也可以自动切换到从。

--cluster-replicas 1 表示我们希望为集群中的每个主节点创建一个从节点。分配原则尽量保证每个主数据库运行在不同的IP地址，每个从库和主库不在一个IP地址上。

一个 Redis 集群包含 16384 个插槽（hash slot）， 数据库中的每个键都属于这 16384 个插槽的其中一个，

集群使用公式 CRC16(key) % 16384 来计算键 key 属于哪个槽， 其中 CRC16(key) 语句用于计算键 key 的 CRC16 校验和。集群中的每个节点负责处理一部分插槽。

### 检查与测试集群

测试：
```shell
redis-cli -h host.docker.internal -p 6380 -c -a 1234
```
注意，不确定是由于cluster-announce-ip是host.docker.internal，还是端口映射问题。在宿主机不可以连接集群容器。
redis-cli的-c参数表示通过集群形式访问，也就是客户端不需要知道key落在哪个节点，访问任意节点，redis都能自动重定向key的操作到对应的节点上。

检查集群：
```shell
host.docker.internal:6380> cluster nodes
6b4b8be480d4da9d56a4377db122d30d9dd00215 host.docker.internal:6382@16382 master - 0 1676273596000 3 connected 10923-16383
1aff20eaf32b5754377fbcf7ec72d82e04521ee4 host.docker.internal:6384@16384 slave 6b4b8be480d4da9d56a4377db122d30d9dd00215 0 1676273597134 3 connected
ac72418c71ffbab84594c07c376ebb8718c64c79 host.docker.internal:6385@16385 slave 0454789355b9222148c7e9901eaaba7137e465b5 0 1676273597622 1 connected
0454789355b9222148c7e9901eaaba7137e465b5 host.docker.internal:6380@16380 myself,master - 0 1676273597000 1 connected 0-5460
8ef733235773bd64a44ccd5f5b0f4cf75fb011a0 host.docker.internal:6383@16383 slave 18f0e906e8be0bf5ed300c5d43e1eaaee5829940 0 1676273596622 2 connected
18f0e906e8be0bf5ed300c5d43e1eaaee5829940 host.docker.internal:6381@16381 master - 0 1676273596000 2 connected 5461-10922
```
表头为`<id> <ip:port> <flags> <master> <ping-sent> <pong-recv> <config-epoch> <link-state> <slot> <slot> ... <slot>`

`cluster info`也可以显示集群信息
```shell
host.docker.internal:6385> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:11
cluster_my_epoch:10
cluster_stats_messages_ping_sent:24236
cluster_stats_messages_pong_sent:23989
cluster_stats_messages_sent:48225
cluster_stats_messages_ping_received:23989
cluster_stats_messages_pong_received:24054
cluster_stats_messages_received:48043
total_cluster_links_buffer_limit_exceeded:0
```

故障转移测试
```shell
redis-cli -h host.docker.internal -p 6380 -c -a 1234
host.docker.internal:6380> get k1
-> Redirected to slot [12706] located at host.docker.internal:6382
(nil)
host.docker.internal:6382> set k1 v1
OK
host.docker.internal:6382> get k1
"v1"
host.docker.internal:6382> cluster nodes
2b02072472b7a0665e4cb392df9984d5162bf534 host.docker.internal:6380@16380 master - 0 1676278542000 1 connected 0-5460
cf7b74361ef75ba9ae0e165df4ade316bb50b93f host.docker.internal:6384@16384 slave 37c1d85f60c928fe58668bbcec0d7d8de8a31af0 0 1676278542545 3 connected
1984d41dfec08dd17c48c971cf3f5bd84c0bab23 host.docker.internal:6385@16385 slave 2b02072472b7a0665e4cb392df9984d5162bf534 0 1676278541658 1 connected
37c1d85f60c928fe58668bbcec0d7d8de8a31af0 host.docker.internal:6382@16382 master - 0 1676278542632 3 connected 10923-16383
2f9ecde6f02283eaf0a2a35343607a52d0dc6bf4 host.docker.internal:6383@16383 slave ba0cb606cd99db363d16f01abd28f93423255c29 0 1676278542129 2 connected
ba0cb606cd99db363d16f01abd28f93423255c29 host.docker.internal:6381@16381 myself,master - 0 1676278541000 2 connected 5461-10922
host.docker.internal:6382> exit
关闭redis-6382服务器，制造主节点故障。6382的原从节点6384晋升为主节点，6382显示断开连接。
redis-cli -h host.docker.internal -p 6380 -c -a 1234
host.docker.internal:6380> cluster nodes
1984d41dfec08dd17c48c971cf3f5bd84c0bab23 host.docker.internal:6385@16385 slave 2b02072472b7a0665e4cb392df9984d5162bf534 0 1676278649000 1 connected
ba0cb606cd99db363d16f01abd28f93423255c29 host.docker.internal:6381@16381 master - 0 1676278650000 2 connected 5461-10922
37c1d85f60c928fe58668bbcec0d7d8de8a31af0 host.docker.internal:6382@16382 master,fail - 1676278627839 1676278627537 3 disconnected
2b02072472b7a0665e4cb392df9984d5162bf534 host.docker.internal:6380@16380 myself,master - 0 1676278649000 1 connected 0-5460
cf7b74361ef75ba9ae0e165df4ade316bb50b93f host.docker.internal:6384@16384 master - 0 1676278650663 7 connected 10923-16383
2f9ecde6f02283eaf0a2a35343607a52d0dc6bf4 host.docker.internal:6383@16383 slave ba0cb606cd99db363d16f01abd28f93423255c29 0 1676278650000 2 connected
重新开启6382服务器，6382连接，作为6384的从
host.docker.internal:6380> cluster nodes
1984d41dfec08dd17c48c971cf3f5bd84c0bab23 host.docker.internal:6385@16385 slave 2b02072472b7a0665e4cb392df9984d5162bf534 0 1676279012552 1 connected
ba0cb606cd99db363d16f01abd28f93423255c29 host.docker.internal:6381@16381 master - 0 1676279012455 2 connected 5461-10922
37c1d85f60c928fe58668bbcec0d7d8de8a31af0 host.docker.internal:6382@16382 slave cf7b74361ef75ba9ae0e165df4ade316bb50b93f 0 1676279014448 7 connected
2b02072472b7a0665e4cb392df9984d5162bf534 host.docker.internal:6380@16380 myself,master - 0 1676279013000 1 connected 0-5460
cf7b74361ef75ba9ae0e165df4ade316bb50b93f host.docker.internal:6384@16384 master - 0 1676279014000 7 connected 10923-16383
2f9ecde6f02283eaf0a2a35343607a52d0dc6bf4 host.docker.internal:6383@16383 slave ba0cb606cd99db363d16f01abd28f93423255c29 0 1676279013469 2 connected
```

## 主要参考

[redis 集群](https://www.jianshu.com/p/25b5e4601a71)

[redis 集群与常见错误](https://blog.csdn.net/weixin_60223449/article/details/125398004)
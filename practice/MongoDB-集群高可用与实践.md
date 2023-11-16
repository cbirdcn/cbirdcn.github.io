# MongoDB-集群高可用与实践

## 原理与概念

### 副本集replic_set

因为主从模式有数据读写不一致和不可自动灾备切换问题，所以MongoDB抛弃了主从模式。

副本集是主从模式的一种改良。一组mongod进程维护同一个数据集合。因为副本集存在数据冗余，在多台服务器保存数据能避免一台服务器故障导致不可用或丢数据。但是这也增加了服务器数量导致成本上升。

![MongoDB副本集架构](https://syushin-blog-oss.oss-cn-shenzhen.aliyuncs.com/blog-img/mongodb/replica-set.png)

特点
- 多副本集群在故障的时候，可以使用同步完的副本自动恢复服务。
- 支持读写分离，类似主从模式，将读分流到副本(secondary)，降低主(primary)的读写压力。
- primary接收请求处理写操作记录到oplog，再同步log到secondary。secondary定期从primary获取oplog并本地执行，从而保证数据一致。
- primary挂掉时，会自动选举得到新的primary继续提供服务。如果需要读请求直接到secondary，需要客户端提供配置参数优先读取secondary
- 由于副本集出现故障的时候，存活的节点必须大于副本集节点总数的一半，否则无法选举主节点，或者主节点会自动降级为从节点，整个副本集变为只读。所以可以提供Arbiter仲裁节点参与投票，但是不接收复制的数据，也不能成为活跃节点
- 副本之间有心跳检测，可以感知集群状态

### 分片shard

多副本可以解决可用性问题，但是当数据量非常大时，数据量和吞吐量会对单机的性能造成较大的压力，会影响到单机的CPU、内存等，所以还是需要分片集群。

分片是将数据库进行拆分，将其分散在不同的机器上的过程。

Sharding 集群中有以下三大模块
- Config Server：配置中心。管理所有shard的节点、shard数据路由信息。不能单点，也是一个replicaSet，默认需要配置3个Config Server节点。
- Mongos路由：代理层。提供对外应用访问，所有操作均通过无状态的mongos执行。mongos通过散列算法将请求路由到对应的数据分片。一般有多个mongos节点。提供数据迁移和数据自动平衡。
- sharding分片：数据层， 存储应用数据记录。一般有多个Mongod节点，达到数据分片目的。

![MongoDB分片集架构](https://syushin-blog-oss.oss-cn-shenzhen.aliyuncs.com/blog-img/mongodb/sharding-particular.png)

#### shard原理

选一个字段（或者多个字段组合也可以）用来做 Key，这个 Key 可以是你任意指定的一个字段。

我们把 Sharding Key 作为输入，按照特点的分片策略计算出一个值，值的集合形成了一个值域，我们按照固定步长去切分这个值域，每一个片叫做 Chunk ，每个 Chunk 出生的时候就和某个 Shard 绑定起来，这个绑定关系存储在配置中心里。

所以，我们看到 MongoDB 用 Chunk 再做了一层抽象层，隔离了用户数据和 Shard 的位置，用户数据先按照分片策略算出落在哪个 Chunk 上，由于 Chunk 某一时刻只属于某一个 Shard，所以自然就知道用户数据存到哪个 Shard 了。

![MongoDB集群分片写入原理](https://syushin-blog-oss.oss-cn-shenzhen.aliyuncs.com/blog-img/mongodb/sharding-write.gif)

实际情况下，mongos 不需要每次都和 Config Server 交互，大部分情况下只需要把 Chunk 的映射表 cache 一份在 mongos 的内存，就能减少一次网络交互，提高性能。

而Chunk的作用是，可以通过进程`splitting`将大Chunk切分为更小的Chunk，再通过后台进程`Balancing`迁移Chunk，从而均衡各个shard server的负载。这个过程是自动进行的。当存储需求变大，集群上服务器的chunk数量变多，每个服务器的chunk数据严重失衡时，就会进行chunk的迁移。

MongoDB 支持两种 Sharding Strategy：
- Hashed Sharding （哈希分片）。拥有相近sharding-key的文档很可能不会存储在同一个数据块中，因此数据的分离性更好一些。所以速度快，但是排序慢。
- Range Sharding （以范围为基础的分片）。相近sharding-key的文档很可能存储在同一个数据块中，因此也会存储在同一个分片中。数据不均匀容易出现热点问题，增加单机负载，范围查询快。

## 实践-副本集

构建三副本，以第一个副本作为Primary，其他作为secondary，构建集群rs。

### 准备挂载目录(本地测试环境需要，正式环境不需要)

将容器的配置、数据库、日志都可以挂载到宿主机，需要准备一个位置存放这些文件

比如`/Users/gaea/docker/mongodb/`

```shell
# 创建配置文件目录
mkdir -p /Users/gaea/docker/mongodb/conf
# 创建数据文件目录(3个)
mkdir -p /Users/gaea/docker/mongodb/data/{1..3}
# 创建日志文件目录（3个）
mkdir -p /Users/gaea/docker/mongodb/logs/{1..3}
```

注意，这里的key、db、log将可以被宿主机、所有容器内部使用，也就是说容器建立时，这里的conf被映射到容器内的conf并启动。

### 生成通用秘钥文件

集群中所有机器都需要用同一个key文件进行认证

```shell
openssl rand -base64 756 > /Users/gaea/docker/mongodb/conf/mongo.key
chmod 600 /Users/gaea/docker/mongodb/conf/mongo.key
```

注意访问权限不能设置太高，比如644会被提示`permissions are too open`，600就足够。

auth.key文件举例：

```
6LQhRGZOEqUy5a8A5lZZ3Kul4PPgrJGLZ7uiVID8crq1HNeDvEKtz048bjZyJxGs
Ru2Cc4x0Wxu8Qod6Z3ttoJnOAZ0Mf9BRhZ66uhiyCwjLpp8bNkdegoZ6+eWog0pn
7mBVA8vXzoSwdzHibZ0JfMrm3+ZfhKCdNPQPDHNGvtdualCrhnWJRbK0/T7u+aOM
O79ijmFunvUrPxATizMFAGMKipU36oYaYspTaSTxp8wpop077LeIeqarfP57lz03
3tuRrrrppu38F5huPYjQBE8VPlo4ALel0abXSOsMkgFKTrGmEKyAni6zK9o3djGG
wlhtrbEoFgdHP3WUOsSodnU4R/C8eZ1bWuI/5dosoygofgwL8OK2XpwfWMaX0Smd
IiGmeda+Y05crrqct86Sx5+dmOEJDEJo9Ws8U0ZGof0aOTCiKsYg61JJhkNOYiPm
4uYvIc8AT3i+I0aWGNzThasKzBc+zwVIt5Jm4vc/FQOWlYMO85+WGLHzN1faJMxL
Xc11s83+snFGA5T86EZLIVlUtme+9n+CwnyfalBPfDvQUN5lKXKLnoLPy328W5Fe
xIRWyJMoF64H9s0NRfBvVKABNC4rDCvDEqaA1gLhNKYtY7jBVpqrOCeA6BlGMwkw
dxRF6cq3upmfH58sTt1S3JpbIoqBH2tP1EVTWABOpm2/+jpSpHZeeXz/6M8c0rDA
MZtsGy8tGUf6LSLWQILZXbC/dvTqQorkePp2VDQuKl6zubD4CgnfLBR4mYRDv/9e
WgeUcHUc/BuVWCi9BKGXbu06vJMnFd6Aev8HA2+uDx2bhvSpXCILCuMNTjP5UcvF
iknRsgdjSI6KpMSu7UCtHVz5Ugn4b6QcZkG/TlLqQD8MMGN+4bFegsqW5U5ue3/3
kyIFOFNhrXhHdWFSNzGJrTgvD6PMmjUshM4GBvN0fn88kH4hNsbCjqq+HpLYYBJ1
gDg0M67SkbhEVKl+pVsdoGnJbbclonHZ0aD7JLxiePQJ9/aQ
```

### 配置文件

```yaml
# /Users/gaea/docker/mongodb/conf/mongo.conf
# 日志文件
storage:
  # mongod 进程存储数据目录，此配置仅对 mongod 进程有效
  dbPath: /data/db
systemLog:
  destination: file
  logAppend: true
  path: /data/logs/mongo.log
#  网络设置
# net:
#   port: 27017  #端口号，如果所有容器都使用此端口才开启，可以在docker-compose中配置
#  bindIp: 127.0.0.1    #绑定ip，注释掉则默认0.0.0.0，表示开放给所有ip访问
replication:
  replSetName: configsvr #副本集名称
sharding:
  clusterRole: configsvr # 集群角色，这里配置的角色是配置节点
security:
  authorization: enabled #是否开启认证
  keyFile: /Users/gaea/docker/mongodb/conf/mongo.key #keyFile路径
```

如果要求每个服务器都开放端口27017可以在此配置，如果要让每个容器开放不同的端口并能映射到宿主机的同样端口，要么就在启动容器时映射到宿主机不同端口，或者这里就不配置同样的端口，而是在docker-compose中配置端口。

authorization必须为开启状态，否则容器创建好以后不能添加admin用户

由于容器创建时，conf文件夹在容器和宿主机之间映射，所以容器内的keyFile也就是前面宿主机创建的那个auth.key

### docker-compose

编写docker-compose.yml文件。

不同version的语法不同。

生成3个容器，使用未经过修改的mongo:latest镜像。生产环境要用自定义镜像，可以看下面"通过Dockerfile构建自定义镜像"一节。

ports端口映射，每个容器都在27017端口运行mongodb，服务之间互相访问也都是27017，但是映射到宿主机的端口不同。

networks使用default自动生成桥接网络`mongodb_default`，让三个容器互相连通

volumes挂载路径和前面创建的配置文件路径、数据库和日志路径有关，conf用于宿主机给容器共享文件，db和logs是容器给宿主机共享文件。

entrypoint入口，用之前的配置文件启动每个mongod服务

expose暴露端口，ports映射，如果每个容器都是同样的暴露端口27017，需要映射到不同的主机端口可以这样写，注意端口映射时宿主机在前。

networks网络。让三个容器都在同一个网段交互。选择default则表示自动生成网络"mongodb_default"。driver表示网络模式，比如"host","dridge"。host表示所有容器使用和宿主机一样的ip，bridge表示所有容器使用同一网段的不同ip然后桥接(转发)到宿主机。如果用host需要让不同容器暴露的端口是不一致的，而bridge则需要让一个容器开放SSH22端口给宿主机访问。

注意，此时只是分别生成了三个允许通过auth访问的mongodb容器，还没有初始化集群，也没有添加用户。

```yaml
# docker-compose.yml
version: "2.0"
services:
  mongo-1:
    image: mongo
    hostname: mongo-1
    ports:
      - 27011:27017
    expose:
      - 27017
    privileged: true
    networks:
      - default
    volumes:
      - /Users/gaea/docker/mongodb/conf:/data/configdb/conf
      - /Users/gaea/docker/mongodb/data/1:/data/db
      - /Users/gaea/docker/mongodb/logs/1:/data/logs
    entrypoint: [ "/usr/bin/mongod", "--bind_ip_all", "--replSet", "rs", "--config", "/data/configdb/conf/mongo.conf"]
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: 123456
  mongo-2:
    image: mongo
    hostname: mongo-2
    ports:
      - 27012:27017
    expose:
      - 27017
    privileged: true
    networks:
      - default
    volumes:
      - /Users/gaea/docker/mongodb/conf:/data/configdb/conf
      - /Users/gaea/docker/mongodb/data/2:/data/db
      - /Users/gaea/docker/mongodb/logs/2:/data/logs
    entrypoint: [ "/usr/bin/mongod", "--bind_ip_all", "--replSet", "rs", "--config", "/data/configdb/conf/mongo.conf"]
  mongo-3:
    image: mongo
    hostname: mongo-3
    ports:
      - 27013:27017
    expose:
      - 27017
    privileged: true
    networks:
      - default
    volumes:
      - /Users/gaea/docker/mongodb/conf:/data/configdb/conf
      - /Users/gaea/docker/mongodb/data/3:/data/db
      - /Users/gaea/docker/mongodb/logs/3:/data/logs
    entrypoint: [ "/usr/bin/mongod", "--bind_ip_all", "--replSet", "rs", "--config", "/data/configdb/conf/mongo.conf"]

networks:
  default:
    driver: bridge

```

启动容器
```shell
docker-compose up -d
```

如果出现错误可以通过命令检查错误日志，或者进入前面的logs路径下查看日志文件：
```shell
docker logs --tail=10 CONTAINER_ID
```

另外，如果需要重新生成容器，有必要删除data中的文件，防止上次的操作在下次生成容器后仍然存在

### 添加admin用户

进入主mongo容器，这里就是mongo-1

```shell
// 进入mongoshell，自动连接到本机的27017，如果自定义的端口需要--port参数
# mongosh
Current Mongosh Log ID:	6554c5b4dd7624af9077b9ed
Connecting to:		mongodb://127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.0.1
Using MongoDB:		7.0.2
Using Mongosh:		2.0.1

For mongosh info see: https://docs.mongodb.com/mongodb-shell/

// 需要进入admin库才能添加账号或auth认证
test> use admin
switched to db admin

// 账号还未存在
admin> db.auth("root","123456")
MongoServerError: Authentication failed.

// 集群初始化，每个容器的名称因为是同一个docker-compose中且在同一个网桥下所以可以作为ip的别名保持网络连通，端口还没经过映射所以都是27017
admin> rs.initiate({ _id: "rs", members: [{_id:1,host:"mongo-1:27017"},{_id:2,host:"mongo-2:27017"},{_id:3,host:"mongo-3:27017"}]})
{ ok: 1 }

// 注意，副本集的数据初始化和同步是需要时间的，所以虽然rs已经初始化返回ok，但是当前mongosh还以为自己是other或者是secondary身份，因为还没确定好谁是primary。此时创建用户会提示not primary
rs [direct: other] admin> db.createUser(
... { 
...     user:'root',
...     pwd:'123456',
...     roles:[{ role:'userAdminAnyDatabase', db: 'admin'}, 'readWriteAnyDatabase','clusterAdmin']
... });
MongoServerError: not primary

// 稍等几秒，新的命令行提示符应该就变成primary了
rs [direct: other] admin> <Enter>

rs [direct: secondary] admin> <Enter>

rs [direct: secondary] admin> <Enter>

// 对变身primary的shell执行createUser操作
rs [direct: primary] admin> db.createUser(
... { 
...     user:'root',
...     pwd:'123456',
...     roles:[{ role:'userAdminAnyDatabase', db: 'admin'}, 'readWriteAnyDatabase','clusterAdmin']
... });
{
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1700055086, i: 4 }),
    signature: {
      hash: Binary.createFromBase64("tolZuxKGc2pVyyU46eAoM9/ZWEQ=", 0),
      keyId: Long("7301678474622664726")
    }
  },
  operationTime: Timestamp({ t: 1700055086, i: 4 })
}

// 用账号密码登录，稍等一下，此时去其他副本也应该认证通过
rs [direct: primary] admin> db.auth("root","123456")
{ ok: 1 }

// 查看集合不再弹出 MongoServerError: Authentication failed.
rs [direct: primary] admin> show collections
system.keys
system.users
system.version

// 查看rs集群配置。
// 注意host名规定了在客户端连接到集群时也必须用此名，而不能使用ip:port的格式
rs [direct: primary] admin> rs.conf()
{
  _id: 'rs',
  version: 1,
  term: 1,
  members: [
    {
      _id: 1,
      host: 'mongo-1:27017',
      arbiterOnly: false,
      buildIndexes: true,
      hidden: false,
      priority: 1,
      tags: {},
      secondaryDelaySecs: Long("0"),
      votes: 1
    },
    {
      _id: 2,
      host: 'mongo-2:27017',
      arbiterOnly: false,
      buildIndexes: true,
      hidden: false,
      priority: 1,
      tags: {},
      secondaryDelaySecs: Long("0"),
      votes: 1
    },
    {
      _id: 3,
      host: 'mongo-3:27017',
      arbiterOnly: false,
      buildIndexes: true,
      hidden: false,
      priority: 1,
      tags: {},
      secondaryDelaySecs: Long("0"),
      votes: 1
    }
  ],
  configsvr: true,
  protocolVersion: Long("1"),
  writeConcernMajorityJournalDefault: true,
  settings: {
    chainingAllowed: true,
    heartbeatIntervalMillis: 2000,
    heartbeatTimeoutSecs: 10,
    electionTimeoutMillis: 10000,
    catchUpTimeoutMillis: -1,
    catchUpTakeoverDelayMillis: 30000,
    getLastErrorModes: {},
    getLastErrorDefaults: { w: 1, wtimeout: 0 },
    replicaSetId: ObjectId("6554c5d7d6d8e76c72524ac2")
  }
}

// 查看集群状态，注意members中mongo-1是PRIMARY主节点，提供增删查改服务。其他都是SECONDARY备节点，只提供读，需要`db.getMongo().setReadPref('secondary')`
rs [direct: primary] admin> rs.status()
{
  set: 'rs',
  date: ISODate("2023-11-15T13:44:39.757Z"),
  myState: 1,
  term: Long("1"),
  syncSourceHost: '',
  syncSourceId: -1,
  configsvr: true,
  heartbeatIntervalMillis: Long("2000"),
  majorityVoteCount: 2,
  writeMajorityCount: 2,
  votingMembersCount: 3,
  writableVotingMembersCount: 3,
  optimes: {
    lastCommittedOpTime: { ts: Timestamp({ t: 1700055879, i: 1 }), t: Long("1") },
    lastCommittedWallTime: ISODate("2023-11-15T13:44:39.444Z"),
    readConcernMajorityOpTime: { ts: Timestamp({ t: 1700055879, i: 1 }), t: Long("1") },
    appliedOpTime: { ts: Timestamp({ t: 1700055879, i: 1 }), t: Long("1") },
    durableOpTime: { ts: Timestamp({ t: 1700055879, i: 1 }), t: Long("1") },
    lastAppliedWallTime: ISODate("2023-11-15T13:44:39.444Z"),
    lastDurableWallTime: ISODate("2023-11-15T13:44:39.444Z")
  },
  lastStableRecoveryTimestamp: Timestamp({ t: 1700055867, i: 1 }),
  electionCandidateMetrics: {
    lastElectionReason: 'electionTimeout',
    lastElectionDate: ISODate("2023-11-15T13:21:39.323Z"),
    electionTerm: Long("1"),
    lastCommittedOpTimeAtElection: { ts: Timestamp({ t: 1700054487, i: 1 }), t: Long("-1") },
    lastSeenOpTimeAtElection: { ts: Timestamp({ t: 1700054487, i: 1 }), t: Long("-1") },
    numVotesNeeded: 2,
    priorityAtElection: 1,
    electionTimeoutMillis: Long("10000"),
    numCatchUpOps: Long("0"),
    newTermStartDate: ISODate("2023-11-15T13:21:39.428Z"),
    wMajorityWriteAvailabilityDate: ISODate("2023-11-15T13:21:39.955Z")
  },
  members: [
    {
      _id: 1,
      name: 'mongo-1:27017',
      health: 1,
      state: 1,
      stateStr: 'PRIMARY',
      uptime: 1464,
      optime: { ts: Timestamp({ t: 1700055879, i: 1 }), t: Long("1") },
      optimeDate: ISODate("2023-11-15T13:44:39.000Z"),
      lastAppliedWallTime: ISODate("2023-11-15T13:44:39.444Z"),
      lastDurableWallTime: ISODate("2023-11-15T13:44:39.444Z"),
      syncSourceHost: '',
      syncSourceId: -1,
      infoMessage: '',
      electionTime: Timestamp({ t: 1700054499, i: 1 }),
      electionDate: ISODate("2023-11-15T13:21:39.000Z"),
      configVersion: 1,
      configTerm: 1,
      self: true,
      lastHeartbeatMessage: ''
    },
    {
      _id: 2,
      name: 'mongo-2:27017',
      health: 1,
      state: 2,
      stateStr: 'SECONDARY',
      uptime: 1391,
      optime: { ts: Timestamp({ t: 1700055877, i: 1 }), t: Long("1") },
      optimeDurable: { ts: Timestamp({ t: 1700055877, i: 1 }), t: Long("1") },
      optimeDate: ISODate("2023-11-15T13:44:37.000Z"),
      optimeDurableDate: ISODate("2023-11-15T13:44:37.000Z"),
      lastAppliedWallTime: ISODate("2023-11-15T13:44:39.444Z"),
      lastDurableWallTime: ISODate("2023-11-15T13:44:39.444Z"),
      lastHeartbeat: ISODate("2023-11-15T13:44:38.037Z"),
      lastHeartbeatRecv: ISODate("2023-11-15T13:44:38.629Z"),
      pingMs: Long("0"),
      lastHeartbeatMessage: '',
      syncSourceHost: 'mongo-1:27017',
      syncSourceId: 1,
      infoMessage: '',
      configVersion: 1,
      configTerm: 1
    },
    {
      _id: 3,
      name: 'mongo-3:27017',
      health: 1,
      state: 2,
      stateStr: 'SECONDARY',
      uptime: 1391,
      optime: { ts: Timestamp({ t: 1700055877, i: 1 }), t: Long("1") },
      optimeDurable: { ts: Timestamp({ t: 1700055877, i: 1 }), t: Long("1") },
      optimeDate: ISODate("2023-11-15T13:44:37.000Z"),
      optimeDurableDate: ISODate("2023-11-15T13:44:37.000Z"),
      lastAppliedWallTime: ISODate("2023-11-15T13:44:39.444Z"),
      lastDurableWallTime: ISODate("2023-11-15T13:44:39.444Z"),
      lastHeartbeat: ISODate("2023-11-15T13:44:38.036Z"),
      lastHeartbeatRecv: ISODate("2023-11-15T13:44:38.630Z"),
      pingMs: Long("0"),
      lastHeartbeatMessage: '',
      syncSourceHost: 'mongo-1:27017',
      syncSourceId: 1,
      infoMessage: '',
      configVersion: 1,
      configTerm: 1
    }
  ],
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1700055879, i: 1 }),
    signature: {
      hash: Binary.createFromBase64("0GRdY4nXq+y0EtqfSuAAmPaM8uE=", 0),
      keyId: Long("7301678474622664726")
    }
  },
  operationTime: Timestamp({ t: 1700055879, i: 1 })
}

```

到这里集群就可用了，在MongoDB Compass中可以通过`mongodb://root:123456@localhost:27011/?directConnection=true&authMechanism=DEFAULT`连接到PRIMARY

注意，如果连接字符串是单一连接（directConnection）到secondary时，将无法做写操作。比如`mongodb://root:123456@localhost:27012/?directConnection=true&authMechanism=DEFAULT&readPreference=secondary&replicaSet=rs`。`readPreference=secondary`时，当前连接可以读，如果没有这个选项，甚至不能读。

注意，在容器中可以通过`mongodb://root:123456@mongo-1:27017,mongo-2:27017,mongo-3:27017/?replicaSet=rs`链接到集群，但是在宿主机中无法通过`mongodb://root:123456@localhost:27011,localhost:27012,localhost:27013/?replicaSet=rs`连接。

原因是前面`rs.conf()`中看到的host名是`mongo-1:27017`，这就要求在客户端连接到集群时也必须用此名，而不能使用`ip:port`的格式。所以在容器内通过ip也无法连接到集群，可以通过`docker inspect 386736e09b66 | grep IPAddress`查找容器的IP，并通过一个容器连接`mongosh mongodb://172.18.0.2:27011,172.18.0.4:27012,172.18.0.3:27013`确认。

此时要在宿主机中连接到集群，只有两个办法：
- 通过ssh隧道连接到其中一个mongoX主机（需要提前开放ssh 22端口），然后通过这个容器连接到集群。
- 将容器使用的网络docker network mode改为"host"，然后更新rs配置，才能在宿主机使用"localhost:port"访问集群，但是host模式下所有容器的IP是相同的，那就不能所有容器用同一个端口了

[stackoverflow: unable-to-connect-to-local-mongodb-docker-cluster-with-compass](https://stackoverflow.com/questions/72739581/unable-to-connect-to-local-mongodb-docker-cluster-with-compass)

原文在[connection-string](https://www.mongodb.com/docs/manual/reference/connection-string/#self-hosted-replica-set-with-members-on-different-machines)

> Self-Hosted Deployment Connection String Examples
> ...
> Self-Hosted Replica Set with Members on Different Machines
> ...
> Replica Set
> ...
> For a replica set, specify the hostname(s) of the mongod instance(s) as listed in the replica set configuration.

### 增删节点

```shell
rs.add("ip:端口)
rs.remove("ip:端口")
// 仲裁
rs.addArb("ip:端口")
```

### 其他

#### 通过Dockerfile构建自定义镜像

在生产环境不能通过挂载Volumes的形式向容器提供配置文件，需要将配置文件、认证文件等打入镜像中，然后在docker-compose中用这个自定义镜像部署容器。

下面介绍打镜像过程，以认证文件auth.key为例。

构建Dockerfile文件
```yaml
FROM mongo:3.4.10
COPY auth.key /tmp/
RUN chown -R mongodb:mongodb /tmp/auth.key
RUN chmod 600 /tmp/auth.key
```

然后以当前Dockerfile文件构建镜像，镜像名和版本设定为`mongo-repl:v1.0`。`.`表示寻找当前路径下的Dockerfile文件
```shell
docker build -t mongo-repl:v1.0 .
```

这样auth.key就被打入了镜像mongo-repl中。

检查镜像
```shell
docker images
```

docker-compose做容器编排时，就可以使用镜像`image: mongo-repl:v1.0`了，也可以直接使用`/tmp/auth.key`文件了

## 分片集群

TODO

## 主要参考

[Deploy a Replica Set in the Terminal](https://www.mongodb.com/docs/manual/tutorial/deploy-replica-set/)

[手把手教你搭建MongoDB集群搭建](https://www.cnblogs.com/jiagooushi/p/16477696.html)

[Docker搭建Mongodb集群](http://bazingafeng.com/2017/06/19/create-mongodb-replset-cluster-using-docker/)

[使用docker-compose创建mongoDB副本集](https://chunlife.top/2021/09/16/%E4%BD%BF%E7%94%A8docker-compose%E5%88%9B%E5%BB%BAmongoDB%E5%89%AF%E6%9C%AC%E9%9B%86/)

[docker 部署mongoDB集群与读写分离 ](https://www.cnblogs.com/ejiyuan/p/17255287.html)

[分片集群](https://www.cnblogs.com/syushin/p/15633183.html)
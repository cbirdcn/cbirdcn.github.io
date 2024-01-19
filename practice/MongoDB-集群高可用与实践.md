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

## 常用命令行操作

### 数据库

#### 管理用户权限

```shell
use admin // 一般都将用户权限放在admin库
db.createUser( // 为集群添加用户、设置密码、操作权限。账号密码都是字符串。操作权限为所有数据库以及集群管理员。
... { 
...     user:'root',
...     pwd:'123456',
...     roles:[{ role:'userAdminAnyDatabase', db: 'admin'}, 'readWriteAnyDatabase','clusterAdmin']
... });
use admin
switched to db admin
db.auth("root", "123456")
rs [direct: primary] admin> show users // 查看当前库中所有的user和权限
[
  {
    _id: 'admin.root',
    userId: new UUID("937bc297-8986-40b4-862d-d8f39329159c"),
    user: 'root',
    db: 'admin',
    roles: [
      { role: 'readWriteAnyDatabase', db: 'admin' },
      { role: 'clusterAdmin', db: 'admin' },
      { role: 'userAdminAnyDatabase', db: 'admin' }
    ],
    mechanisms: [ 'SCRAM-SHA-1', 'SCRAM-SHA-256' ]
  }
]
db.removeUser("userName") // 删除用户
```

#### 管理数据库

```shell
# mongosh 
Current Mongosh Log ID:	655d696d47d945605b977836
Connecting to:		mongodb://127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.0.1
Using MongoDB:		7.0.2
Using Mongosh:		2.0.1

For mongosh info see: https://docs.mongodb.com/mongodb-shell/

rs [direct: secondary] test> use admin
switched to db admin
rs [direct: secondary] admin> db.auth("root", 123456) // 用户名和密码都是字符串类型
TypeError: Expected string.
rs [direct: secondary] admin> db.auth("root", "123456")
{ ok: 1 }
rs [direct: secondary] admin> show dbs // 所有数据库
admin   172.00 KiB
config  248.00 KiB
local     1.50 MiB
rs [direct: secondary] admin> db.getName() // 当前数据库名
admin
rs [direct: secondary] admin> db.getMongo() // 获取当前数据库的连接字符串
mongodb://127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.0.1
rs [direct: secondary] admin> use t // 使用新库
switched to db t
rs [direct: secondary] t> show dbs // 新库不会在use时创建，只有真正插入数据时才会创建
admin   172.00 KiB
config  248.00 KiB
local     1.50 MiB
rs [direct: secondary] t> db.createCollection("collName", {size: 1, capped: true, max: 2}); // 在读写分离的rs集群中，对secondary执行写操作是错误的
MongoServerError: not primar
rs [direct: secondary] t> rs.status() // 得到集群的主节点信息
...
members: [
    {
      _id: 1,
      name: 'mongo-1:27017',
      health: 1,
      state: 1,
      stateStr: 'PRIMARY',
      uptime: 4075,
      optime: { ts: Timestamp({ t: 1700624347, i: 1 }), t: Long("2") },
      optimeDurable: { ts: Timestamp({ t: 1700624346, i: 1 }), t: Long("2") },
      optimeDate: ISODate("2023-11-22T03:39:07.000Z"),
      optimeDurableDate: ISODate("2023-11-22T03:39:06.000Z"),
      lastAppliedWallTime: ISODate("2023-11-22T03:39:07.258Z"),
      lastDurableWallTime: ISODate("2023-11-22T03:39:06.258Z"),
      lastHeartbeat: ISODate("2023-11-22T03:39:07.330Z"),
      lastHeartbeatRecv: ISODate("2023-11-22T03:39:06.864Z"),
      pingMs: Long("0"),
      lastHeartbeatMessage: '',
      syncSourceHost: '',
      syncSourceId: -1,
      infoMessage: '',
      electionTime: Timestamp({ t: 1700620281, i: 1 }),
      electionDate: ISODate("2023-11-22T02:31:21.000Z"),
      configVersion: 1,
      configTerm: 2
    },
...
rs [direct: secondary] t> exit // 退出当前节点的连接
```

### 集合

#### 集合管理

重新进入主节点的连接，执行集合的增删操作

```shell
# mongosh mongo-1:27017 // 注意：要用集群中的节点名称，而不是ip
Current Mongosh Log ID:	655d781ad0663a69b556ae18
Connecting to:		mongodb://mongo-1:27017/?directConnection=true&appName=mongosh+2.0.1
Using MongoDB:		7.0.2
Using Mongosh:		2.0.1

For mongosh info see: https://docs.mongodb.com/mongodb-shell/

rs [direct: primary] test> use admin
switched to db admin
rs [direct: primary] admin> db.auth("root", "123456")
{ ok: 1 }
rs [direct: primary] admin> use t
switched to db t
// 创建集合名称为"collName"，capped表示固定容量(类似环形队列，一旦集合达到size字节数，它就会开始覆盖旧条目),capped=true时需要指定集合的最大字节数size，超过最大字节数的操作会覆盖。max表示集合可容纳最大文档数，超过max的操作会覆盖。autoIndexId为bool类型表示是否自动创建索引_id。
rs [direct: primary] t> db.createCollection("collName", {size: 1, capped: true, max:2}) // 显式创建集合
{ ok: 1 }
rs [direct: primary] t> show dbs // 只有写操作后，新库才会建立
admin   172.00 KiB
config  248.00 KiB
local     1.46 MiB
t         8.00 KiB
rs [direct: primary] t> db.collName.isCapped() // 判断是否固定容量的集合
true
rs [direct: primary] t> db.getCollectionNames(); // 获取所有collectionNames
[ 'collName' ]
rs [direct: primary] t> db.collName.drop() // 删除collection
true
rs [direct: primary] t> db.getCollectionNames();
[]
```

#### 在capped集合中写文档可能发生覆盖

```go
rs [direct: primary] t> db.collName.insertOne({}) // 插入第一个document
{
  acknowledged: true,
  insertedId: ObjectId("655d80fbd0663a69b556ae19")
}
rs [direct: primary] t> db.collName.findOne() // 查询集合中的一个document
{ _id: ObjectId("655d80fbd0663a69b556ae19") }
rs [direct: primary] t> db.collName.insertOne({}) // 插入第二个document
{
  acknowledged: true,
  insertedId: ObjectId("655d8119d0663a69b556ae1a")
}
rs [direct: primary] t> db.collName.find() // find()查询集合中的所有document（带[]）。因为size大于1个字节，所以新插入的数据会发生覆盖
[ { _id: ObjectId("655d8119d0663a69b556ae1a") } ]
rs [direct: primary] t> db.collName.drop()
true
rs [direct: primary] t> db.createCollection("collName", {size: 1000, capped: true, max:2}) // 让size足够大，但max不足，最多允许两个document
rs [direct: primary] t> db.collName.insertOne({})
{
  acknowledged: true,
  insertedId: ObjectId("655d846cd0663a69b556ae22")
}
rs [direct: primary] t> db.collName.insertOne({})
{
  acknowledged: true,
  insertedId: ObjectId("655d846dd0663a69b556ae23")
}
rs [direct: primary] t> db.collName.find()
[
  { _id: ObjectId("655d846cd0663a69b556ae22") },
  { _id: ObjectId("655d846dd0663a69b556ae23") }
]
rs [direct: primary] t> db.collName.insertOne({})
{
  acknowledged: true,
  insertedId: ObjectId("655d848ed0663a69b556ae24")
}
rs [direct: primary] t> db.collName.find() // max=2，最多只有2行document，所以发生了覆盖
[
  { _id: ObjectId("655d846dd0663a69b556ae23") },
  { _id: ObjectId("655d848ed0663a69b556ae24") }
]
```

#### 查看集合信息信息
```shell
rs [direct: primary] t> db.printCollectionStats() // 打印当前db所有聚集索引的状态。或者用db.collName.stats()
collName
{
  ok: 1,
  capped: true,
  max: 2,
  wiredTiger: {
    ...
  },
  shards: {
    ...
  },
  sharded: false,
  size: 44,
  count: 2,
  numOrphanDocs: 0,
  storageSize: 36864,
  totalIndexSize: 36864,
  totalSize: 73728,
  indexSizes: { _id_: 36864 },
  avgObjSize: 22,
  maxSize: 1000,
  ns: 't.collName',
  nindexes: 1,
  scaleFactor: 1
}
rs [direct: primary] t> db.collName.count() // 获取集合中文档条数
DeprecationWarning: Collection.count() is deprecated. Use countDocuments or estimatedDocumentCount.
2
rs [direct: primary] t> db.collName.countDocuments() // 获取集合中文档条数
2
rs [direct: primary] t> db.collName.dataSize() // 集合数据空间大小
44
rs [direct: primary] t> db.collName.totalSize() // 集合整体占据空间大小
73728
rs [direct: primary] t> db.collName.storageSize() // 聚集集合占据空间大小
36864
rs [direct: primary] t> db.collName.renameCollection("coll_name") // 重命名集合
{ ok: 1 }
```

#### 聚集集合查询

部分命令已在之前出现过

```shell
rs [direct: primary] t> db.coll_name.find() // 获取所有记录，shell默认只显示20条，需要"it"命令迭代。或者修改每页显示数量DBQuery.shellBatchSize= 50
[
  { _id: ObjectId("655d846dd0663a69b556ae23") },
  { _id: ObjectId("655d848ed0663a69b556ae24") }
]
rs [direct: primary] t> db.coll_name.find({}) // 查询所有，没有exp或exp={}，都表示查询所有，类似于没有where语句的SQL
[
  { _id: ObjectId("655d846dd0663a69b556ae23") },
  { _id: ObjectId("655d848ed0663a69b556ae24") }
]
rs [direct: primary] t> db.coll_name.findOne()
{ _id: ObjectId("655d846dd0663a69b556ae23") }
```

条件查询

```shell
rs [direct: primary] t> db.coll_name.insertOne({"name": "himongoih", "age": 23}) // 为了后续操作，插入两条记录。后面再说插入
{
  acknowledged: true,
  insertedId: ObjectId("655daa37d0663a69b556ae26")
}
rs [direct: primary] t> db.coll_name.insertOne({"name": "himongoih", "age": 25})
{
  acknowledged: true,
  insertedId: ObjectId("655daa4cd0663a69b556ae27")
}
rs [direct: primary] t> db.coll_name.distinct("name") // 对某列去重后的此列数据列表
[ 'himongoih' ]
rs [direct: primary] t> db.coll_name.find({"name": "himongoih"}) // 条件查询所有
[
  {
    _id: ObjectId("655daa37d0663a69b556ae26"),
    name: 'himongoih',
    age: 23
  },
  {
    _id: ObjectId("655daa4cd0663a69b556ae27"),
    name: 'himongoih',
    age: 25
  }
]
rs [direct: primary] t> db.coll_name.find({"age": {$lte: 23}}) // and操作。查看age<=23的记录。注意$lte是操作符而不是字符串。{列:{$操作符:值}}这样的格式
[
  {
    _id: ObjectId("655daa37d0663a69b556ae26"),
    name: 'himongoih',
    age: 23
  }
]
rs [direct: primary] t> db.coll_name.find({"age": {$lte: 20}}) // 查询不到返回空数据

rs [direct: primary] t> db.coll_name.find({"age": {$lte: "23"}}) // 注意：值的类型是严格区分整形和字符串的，不做兼容。也就等同于查询不到的结果

rs [direct: primary] t> db.coll_name.find({"age": {$gte: 20, $lt: 26}}) // 查询20<=age<26的记录。注意，and关系的操作符写在同一个{}中
[
  {
    _id: ObjectId("655daa37d0663a69b556ae26"),
    name: 'himongoih',
    age: 23
  },
  {
    _id: ObjectId("655daa4cd0663a69b556ae27"),
    name: 'himongoih',
    age: 25
  }
]
rs [direct: primary] t> db.coll_name.find({$or: [{"age": {$gte: 30}}, {"age": { $lt: 24}}]}) // or操作。格式为{$or: [expression1, expression2]}，其中exp分别是{列: {$操作符: 值}}的格式。这样可以支持where field1 >= 30 or field2 < 24 这种操作。
[
  {
    _id: ObjectId("655daa37d0663a69b556ae26"),
    name: 'himongoih',
    age: 23
  }
]
rs [direct: primary] t> db.coll_name.find({"name": /mongo/}) // 模糊查询。类似where name like "%mongo%"
[
  {
    _id: ObjectId("655daa37d0663a69b556ae26"),
    name: 'himongoih',
    age: 23
  },
  {
    _id: ObjectId("655daa4cd0663a69b556ae27"),
    name: 'himongoih',
    age: 25
  }
]
rs [direct: primary] t> db.coll_name.find({"name": /mongo/}, {"name": true}) // 只获取指定列的数据，注意`_id`也会返回。格式find(exp, {"field1": true, "field2": true})
[
  { _id: ObjectId("655daa37d0663a69b556ae26"), name: 'himongoih' },
  { _id: ObjectId("655daa4cd0663a69b556ae27"), name: 'himongoih' }
]
```

对结果集过滤

```shell
rs [direct: primary] t> db.coll_name.find().sort({"age": -1}) // 对查询结果按列降序排列
[
  {
    _id: ObjectId("655daa4cd0663a69b556ae27"),
    name: 'himongoih',
    age: 25
  },
  {
    _id: ObjectId("655daa37d0663a69b556ae26"),
    name: 'himongoih',
    age: 23
  }
]
rs [direct: primary] t> db.coll_name.find().limit(1) // limit过滤（默认升序）
[
  {
    _id: ObjectId("655daa37d0663a69b556ae26"),
    name: 'himongoih',
    age: 23
  }
]
rs [direct: primary] t> db.coll_name.find().sort({"age": -1}).limit(1) // 降序后limit
[
  {
    _id: ObjectId("655daa4cd0663a69b556ae27"),
    name: 'himongoih',
    age: 25
  }
]
rs [direct: primary] t> db.coll_name.find().limit(1).skip(0) // 分页过滤：limit+skip
[
  {
    _id: ObjectId("655daa37d0663a69b556ae26"),
    name: 'himongoih',
    age: 23
  }
]
rs [direct: primary] t> db.coll_name.find().limit(1).skip(1)
[
  {
    _id: ObjectId("655daa4cd0663a69b556ae27"),
    name: 'himongoih',
    age: 25
  }
]
rs [direct: primary] t> db.coll_name.find().skip(0) // 单纯skip过滤就是获取从skip到total
[
  {
    _id: ObjectId("655daa37d0663a69b556ae26"),
    name: 'himongoih',
    age: 23
  },
  {
    _id: ObjectId("655daa4cd0663a69b556ae27"),
    name: 'himongoih',
    age: 25
  }
]
rs [direct: primary] t> db.coll_name.find().count() // 过滤记录数
2
```

### 索引

```shell
rs [direct: primary] t> db.coll_name.createIndex({"age": 1, "name": -1}) // 按照某列升序(1)或降序(-1)索引。也可以是ensureIndex。
[ 'age_1_name_-1' ]
rs [direct: primary] t> db.coll_name.getIndexes() // 获取所有索引
[
  { v: 2, key: { _id: 1 }, name: '_id_' },
  { v: 2, key: { age: 1, name: -1 }, name: 'age_1_name_-1' }
]
rs [direct: primary] t> db.coll_name.createIndex({"age": 1}, {unique: true, expireAfterSeconds:10}) // 创建索引时，设定options。注意capped集合不允许给定索引过期时间
MongoServerError: Cannot create TTL index on a capped collection
rs [direct: primary] t> db.coll_name.createIndex({"age": 1}, {unique: true})
age_1
rs [direct: primary] t> db.coll_name.insertOne({"name": "himongoih", "age": 23}) // 唯一索引不允许重复插入
MongoServerError: E11000 duplicate key error collection: t.coll_name index: age_1 dup key: { age: 23 }
rs [direct: primary] t> db.coll_name.getIndexes()
[
  { v: 2, key: { _id: 1 }, name: '_id_' },
  { v: 2, key: { age: 1, name: -1 }, name: 'age_1_name_-1' },
  { v: 2, key: { age: 1 }, name: 'age_1', unique: true }
]
rs [direct: primary] t> db.coll_name.dropIndex("age_1") // 删除指定索引
{
  nIndexesWas: 3,
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1700641028, i: 2 }),
    signature: {
      hash: Binary.createFromBase64("VPTfFd0Z2K/DZ9BNoR1oKv5rdek=", 0),
      keyId: Long("7301886247960576022")
    }
  },
  operationTime: Timestamp({ t: 1700641028, i: 2 })
}
rs [direct: primary] t> db.coll_name.getIndexes()
[
  { v: 2, key: { _id: 1 }, name: '_id_' },
  { v: 2, key: { age: 1, name: -1 }, name: 'age_1_name_-1' }
]
rs [direct: primary] t> db.coll_name.dropIndexes() // 删除除`_id`外的所有索引
{
  nIndexesWas: 2,
  msg: 'non-_id indexes dropped for collection',
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1700641097, i: 2 }),
    signature: {
      hash: Binary.createFromBase64("ENkZpE0LIwMySjm0qpv1RbNjG0s=", 0),
      keyId: Long("7301886247960576022")
    }
  },
  operationTime: Timestamp({ t: 1700641097, i: 2 })
}
rs [direct: primary] t> db.coll_name.getIndexes()
[ { v: 2, key: { _id: 1 }, name: '_id_' } ]
```

### 写操作

```shell
rs [direct: primary] t> db.coll_name.updateOne({"age": 23}, {$set: {"age": 22, "name": "mongo"}}) // updateOne操作。$set操作符。可以修改多列值。
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}
rs [direct: primary] t> db.coll_name.find()
[
  { _id: ObjectId("655daa37d0663a69b556ae26"), name: 'mongo', age: 22 },
  {
    _id: ObjectId("655daa4cd0663a69b556ae27"),
    name: 'himongoih',
    age: 25
  }
]
rs [direct: primary] t> db.coll_name.updateOne({"age": 23}, {$set: {"age": 22, "name": "mongo"}}) // updateOne操作。$set操作符。可以修改多列值。格式：(filter, update_exp, params)
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}
rs [direct: primary] t> db.coll_name.updateOne({"age": 28}, {$set: {"age": 27, "name": "mongohi"}}, {upsert: true}) // updateOne支持upsert操作，如果filter没有获取到记录，且upsert为true，就将insert数据。如果filter能获取到就update数据。update_exp的写法是支持$inc。
{
  acknowledged: true,
  insertedId: ObjectId("655dbe4a07f1ff52718d8b6d"),
  matchedCount: 0,
  modifiedCount: 0,
  upsertedCount: 1
}
rs [direct: primary] t> db.coll_name.find() // 注意，如果不是capped集合，将会出现3条记录，这里22岁的mongo记录被丢弃了，并插入了新记录27岁的mongohi。
[
  {
    _id: ObjectId("655daa4cd0663a69b556ae27"),
    name: 'himongoih',
    age: 25
  },
  {
    _id: ObjectId("655dbe4a07f1ff52718d8b6d"),
    age: 27,
    name: 'mongohi'
  }
]
rs [direct: primary] t> db.coll_name.updateOne({"age": 28}, {$inc: {"age": 2}}, {upsert: true}) // 自增更新：让age=age+2。因为是upsert，所以会覆盖老数据。
{
  acknowledged: true,
  insertedId: ObjectId("655dc0c607f1ff52718da86e"),
  matchedCount: 0,
  modifiedCount: 0,
  upsertedCount: 1
}
rs [direct: primary] t> db.coll_name.find() // 插入了28岁数据后，又自增+2，并覆盖了25岁数据。注意：新数据没有name。
[
  {
    _id: ObjectId("655dbe4a07f1ff52718d8b6d"),
    age: 27,
    name: 'mongohi'
  },
  { _id: ObjectId("655dc0c607f1ff52718da86e"), age: 30 }
]
rs [direct: primary] t> db.coll_name.updateOne({"age": 28}, {$set: {"age": 27, "name": "mongohi"}}, {upsert: true})
{
  acknowledged: true,
  insertedId: ObjectId("655dc0ea07f1ff52718daa1d"),
  matchedCount: 0,
  modifiedCount: 0,
  upsertedCount: 1
}
rs [direct: primary] t> db.coll_name.find()
[
  { _id: ObjectId("655dc0c607f1ff52718da86e"), age: 30 },
  {
    _id: ObjectId("655dc0ea07f1ff52718daa1d"),
    age: 27,
    name: 'mongohi'
  }
]
rs [direct: primary] t> db.coll_name.updateOne({"age": {$gt: 20}}, {$inc: {"age": 2}, $set: {"name": "mongohi"}}, {upsert: true}) // 当$set和$inc同时出现在updateOne中
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}
rs [direct: primary] t> db.coll_name.find()
[
  {
    _id: ObjectId("655dc0c607f1ff52718da86e"),
    age: 32,
    name: 'mongohi'
  },
  {
    _id: ObjectId("655dc0ea07f1ff52718daa1d"),
    age: 27,
    name: 'mongohi'
  }
]
rs [direct: primary] t> db.coll_name.updateOne({"age": {$gt: 30}}, {$inc: {"age": 2}, $set: {"name": "mongohi"}}, {upsert: true})
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}
rs [direct: primary] t> db.coll_name.find()
[
  {
    _id: ObjectId("655dc0c607f1ff52718da86e"),
    age: 34,
    name: 'mongohi'
  },
  {
    _id: ObjectId("655dc0ea07f1ff52718daa1d"),
    age: 27,
    name: 'mongohi'
  }
]
rs [direct: primary] t> db.coll_name.updateOne({"age": {$gt: 34}}, {$inc: {"age": 2}, $set: {"name": "mongohi"}}, {upsert: true}) // 如果是upsert的新数据，给了$inc但是没有$set的列，将会从默认初始值基础上inc，比如这里age初始值=0，inc后=2
{
  acknowledged: true,
  insertedId: ObjectId("655dc38e07f1ff52718dc9b4"),
  matchedCount: 0,
  modifiedCount: 0,
  upsertedCount: 1
}
rs [direct: primary] t> db.coll_name.find()
[
  {
    _id: ObjectId("655dc0ea07f1ff52718daa1d"),
    age: 27,
    name: 'mongohi'
  },
  {
    _id: ObjectId("655dc38e07f1ff52718dc9b4"),
    age: 2,
    name: 'mongohi'
  }
]
rs [direct: primary] t> db.coll_name.deleteOne({"age": 2}) // 删除一行
{ acknowledged: true, deletedCount: 0 }
rs [direct: primary] t> db.coll_name.find()
[
  {
    _id: ObjectId("655dc0ea07f1ff52718daa1d"),
    age: 27,
    name: 'mongohi'
  }
]
rs [direct: primary] t> db.coll_name.findAndModify({query: {"age": 27}, update: {"age": 28}, upsert: true}) // 查找并修改。有很多params。注意update需要提供所有列名否则会出现列丢失。或者通过$set、$unset等操作符只影响指定的列。
{ _id: ObjectId("655dc0ea07f1ff52718daa1d"), age: 27, name: 'mongohi' }
rs [direct: primary] t> db.coll_name.find()
[ { _id: ObjectId("655dc0ea07f1ff52718daa1d"), age: 28 } ]
rs [direct: primary] t> db.coll_name.findAndModify({query: {"age": 28}, update: {$set: {"name": "mongo"}}, upsert: true}) // 查找并修改指定的列
{ _id: ObjectId("655dc0ea07f1ff52718daa1d"), age: 28 }
rs [direct: primary] t> db.coll_name.find()
[
  { _id: ObjectId("655dc0ea07f1ff52718daa1d"), age: 28, name: 'mongo' }
]
rs [direct: primary] t> db.coll_name.findAndModify({query: {"age": 28}, update: {$unset: {"name": ""}}, upsert: true}) // 查找并删除指定列。使用$unset。格式{$unset: {field1: "", field2: ""}}
{ _id: ObjectId("655dc0ea07f1ff52718daa1d"), age: 28, name: 'mongo' }
rs [direct: primary] t> db.coll_name.find()
[ { _id: ObjectId("655dc0ea07f1ff52718daa1d"), age: 28 } ]
rs [direct: primary] t> db.coll_name.findAndModify({ query: { "age": 28 }, sort: { "age": 1 }, update: {$inc: { "age": 2 }}, remove: true }) // 找到并删除记录。remove优先于update生效
{ _id: ObjectId("655dc0ea07f1ff52718daa1d"), age: 28 }
rs [direct: primary] t> db.coll_name.find()
rs [direct: primary] t> db.coll_name.findAndModify({ query: { "age": 28 }, sort: { "age": 1 }, update: {$inc: { "age": 2 }}, remove: true, upsert: true}) // remove优先于upsert生效
null
rs [direct: primary] t> db.coll_name.find()

rs [direct: primary] t> db.coll_name.findAndModify({ query: { "age": 28 }, sort: { "age": 1 }, update: {$inc: { "age": 2 }}, upsert: true}) // 查找不到没有modify成功返回null。但是upsert插入成功。这里会有歧义。所以在此慎用upsert。
null
rs [direct: primary] t> db.coll_name.find()
[ { _id: ObjectId("655dca0307f1ff52718e1519"), age: 30 } ]
```

### 操作符

除了上面用到的操作符，还有一些
```shell
{ $exists: <boolean> } // 查询存在指定字段的文档
{ $not: { <operator-expression> } } // 不匹配的。where not
{ $nor: [ { <expression1> }, { <expression2> }, ... { <expressionN> } ] } // 与所有exp都不匹配的。where exp1 not... and exp2 not...
{ $min: { <field1>: <value1>, ... } } // 与$set是同类型操作符，如果目标值比field小，则把对应的field更新成目标值。注意相等也不会更新。
{ $currentDate: { <field1>: <typeSpecification1>, ... } } // 设置指定字段为当前时间
```

举例：

```shell
rs [direct: primary] t> db.coll_name.insertOne({"age": 20, "name": "mongo"}) // 插入记录
{
  acknowledged: true,
  insertedId: ObjectId("655dce34d0663a69b556ae2a")
}
rs [direct: primary] t> db.coll_name.find()
[
  { _id: ObjectId("655dce34d0663a69b556ae2a"), age: 20, name: 'mongo' }
]
rs [direct: primary] t> db.coll_name.updateOne({"age": 20}, {$min: {"age": 21}}) // 如果目标值比field大，不能更新
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 0,
  upsertedCount: 0
}
rs [direct: primary] t> db.coll_name.find()
[
  { _id: ObjectId("655dce34d0663a69b556ae2a"), age: 20, name: 'mongo' }
]
rs [direct: primary] t> db.coll_name.updateOne({"age": 20}, {$min: {"age": 20}}) // 如果目标值与field相等，也不能更新，因为更新没有意义（看modifiedCount=0）
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 0,
  upsertedCount: 0
}
rs [direct: primary] t> db.coll_name.find()
[
  { _id: ObjectId("655dce34d0663a69b556ae2a"), age: 20, name: 'mongo' }
]
rs [direct: primary] t> db.coll_name.updateOne({"age": 20}, {$min: {"age": 19}}) // 目标值比field小，就把field更新成目标值
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}
rs [direct: primary] t> db.coll_name.find()
[
  { _id: ObjectId("655dce34d0663a69b556ae2a"), age: 19, name: 'mongo' }
]
rs [direct: primary] t> db.coll_name.updateOne({"name": "mongo"}, {$currentDate: {"last_login": {$type: "timestamp"}}}) // 更新指定字段为当前时间，需要提供当前时间的类型
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}
rs [direct: primary] t> db.coll_name.find()
[
  {
    _id: ObjectId("655dce34d0663a69b556ae2a"),
    age: 19,
    name: 'mongo',
    last_login: Timestamp({ t: 1700647339, i: 2 }) // 注意Timestamp的存储方式
  }
]
rs [direct: primary] t> db.coll_name.updateOne({"name": "mongo"}, {$currentDate: {"last_login": {$type: "date"}}}) // 更新指定字段为当前时间，时间类型为date
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}
rs [direct: primary] t> db.coll_name.find()
[
  {
    _id: ObjectId("655dce34d0663a69b556ae2a"),
    age: 19,
    name: 'mongo',
    last_login: ISODate("2023-11-22T10:03:37.782Z") // date类型的存储方式
  }
]
```

## 编码

### mongo-driver

建议go1.20+配合mongoDB3.6+

[MongoDB Go Driver](https://github.com/mongodb/mongo-go-driver)

`go.mongodb.org/mongo-driver`是官方提供的MongoDB驱动。

示例代码

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"errors"

	"go.mongodb.org/mongo-driver/bson" // 提供bson结构
	"go.mongodb.org/mongo-driver/mongo" // 管理连接等
	"go.mongodb.org/mongo-driver/mongo/options" // 各种选项参数
	// "go.mongodb.org/mongo-driver/bson/primitive" // bson转换主键
)

// const MONGODB_URI = "mongodb://root:123456@host.docker.internal:27011/?maxPoolSize=10&minPoolSize=2&maxConnecting=2&w=mojority"
const MONGODB_URI = "mongodb://root:123456@host.docker.internal:27011/?directConnection=true"
const MONGODB_URI_WITHOUT_CREDENTIAL = "mongodb://host.docker.internal:27011/?directConnection=true"
const MONGODB_USERNAME = "root"
const MONGODB_PASSWORD = "123456"
const DB_GLOBAL = "global"
const COLL_USER_ACCOUNT = "user_account"

var (
	client *mongo.Client
	db *mongo.Database
	coll *mongo.Collection
	err error
)

func Connect(ctx context.Context, opts *options.ClientOptions) (*mongo.Client, error) {
	// client, err := mongo.Connect(ctx, options.Client().ApplyURI(MONGODB_URI)) // 将账号密码作为连接字符串的方式
	client, err = mongo.Connect(ctx, opts) // 将账号密码作为options的方式。client是全局变量
	if err != nil {
		panic(err)
	}
	// 关闭连接。应用中可以不关闭。
	// defer func() {
	// 	if err := client.Disconnect(context.TODO()); err != nil {
	// 		panic(err)
	// 	}
	// }()

	// 注意：Connect()没有实际连接到服务器，只是在本地创建一个连接对象，需要请求（比如ping）后才确认连接是否联通。
	err = client.Ping(ctx, nil)
	if err != nil {
		panic(err)
	}
	// 到这里不会出现err==nil的情况，因为前面panic了，但是一般还是要返回
	return client, err
}

// 获取客户端连接，没有则初始化，有则直接使用
// GetMongoClient和Connect部分，及全局变量client可以独立成一个util.go包，导入进项目
func GetMongoClient(ctx context.Context, opts *options.ClientOptions) (*mongo.Client, error) {
	if client == nil {
		return Connect(ctx, opts)
	}
	return client, nil
}

// 类型定义（表声明）的好处是，可以选择性读写特定的列，并且避免编写列名和语句等防止出错
// 当然也可以嵌套结构体
// 在生产时，也应该独立到一个单独的model.go包中
type CollUserAccount struct {
	Uid string `bson:"_id", json:"uid"`	// 注意：bson规定了，写入到mongo对应的字段名。如果明确了`_id`对应的字段就不会再自动生成`_id`
	Name string `bson:"name", json:"name"`
}

// 为filter设定一个全新的struct，不能用CollStruct，因为没有给值的字段会被当做零值进行filter
// 但是如果很多条件怎么办？所以从条件查询的角度来说，bson.M更适合作为filter条件，只不过每次要注意一下拼写
type CollUserAccountFilterByUid struct {
	Uid string `bson:"uid"`
}

// 添加一行struct数据
func AddOneData(ctx context.Context, coll *mongo.Collection, documents interface{}) (*mongo.InsertOneResult, error){ // 注意structData传入时必须用指针，返回值是*mongo.InsertOneResult，如果需要res.InsertedID就是interface{}类型
	return coll.InsertOne(ctx, documents)
}

// 添加一行struct数据
func AddManyData(ctx context.Context, coll *mongo.Collection, documents []interface{}) (*mongo.InsertManyResult, error){ // 可以拿到所有插入的主键：res.InsertedIDs
	return coll.InsertMany(ctx, documents)
}

func UpsertOneData(ctx context.Context, coll *mongo.Collection, filter bson.D, update bson.D, opts *options.UpdateOptions) (*mongo.UpdateResult, error){
	return coll.UpdateOne(ctx, filter, update, opts) // 注意opts是指针
}

// 注意如果查询不到，就无法给指定变量赋值为解码的数据，并且返回的error就是mongo.ErrNoDocuments
// 也能添加options，比如skip，limit等，但是暂时不加，因为涉及迭代，单独处理
func FindOneDataToBson(ctx context.Context, coll *mongo.Collection, filter bson.D, result *bson.M) (error){
	return coll.FindOne(ctx, filter).Decode(result)
}

func FindOneDataToStructByBsonFilter(ctx context.Context, coll *mongo.Collection, filter bson.D, result *CollUserAccount) (error){ // FindOne()返回*mongo.SingleResult，再Decode()返回error
	return coll.FindOne(ctx, filter).Decode(result)
}

func FindOneDataToStructByStructFilter(ctx context.Context, coll *mongo.Collection, filter *CollUserAccountFilterByUid, result *CollUserAccount) (error){ // FindOne()返回*mongo.SingleResult，再Decode()返回error
	return coll.FindOne(ctx, filter).Decode(result)
}

func DeleteManyData(ctx context.Context, coll *mongo.Collection, filter bson.M) (*mongo.DeleteResult, error){
	return coll.DeleteMany(ctx, filter)
}

// 获取结果的一页数据
func FindManyDataToStructByBsonFilter(ctx context.Context, coll *mongo.Collection, filter bson.D, opts ...*options.FindOptions) (*mongo.Cursor, error){ // 注意可变参数的类型写法
	return coll.Find(ctx, filter, opts...)
}

// 迭代Find()所有结果到一个结果集中，需要提供分页迭代参数
/*
举例：
skipPage := 0 // 跳过了多少页，跳过的记录数就是skipPage*limit
total := 4 // 初始总数量，如果没给就用默认初始值，可以定为100
totalInc := 4 // 如果查完total还没查完，就让total=total+totalInc。可以理解为总量翻倍扩容，则直接让total作为inc。
limit := 2 // 每次查limit条记录，如果有一次没查到记录就停止循环
*/
const DEFAULT_ITERATOR_INITIAL_TOTAL_COUNT = 100
const DEFAULT_ITERATOR_INITIAL_TOTAL_MAX_COUNT = 100000
func IteratorFindManyDataToStructByBsonFilter(ctx context.Context, coll *mongo.Collection, filter bson.D, opts *options.FindOptions, startSkipPage, initialTotalCount, totalInc, limit int64)  ([]CollUserAccount, error){
	// 
	findManyRes := make([]CollUserAccount, 0, initialTotalCount)
	if initialTotalCount < 0 {
		return findManyRes, errors.New("iterator find initialTotalCount must >= 0")
	}
	if initialTotalCount >= DEFAULT_ITERATOR_INITIAL_TOTAL_MAX_COUNT {
		return findManyRes, errors.New("iterator find initialTotalCount over max")
	}
	if limit <= 0 {
		return findManyRes, errors.New("iterator find limit must > 0")
	}
	if initialTotalCount == 0 {
		initialTotalCount = DEFAULT_ITERATOR_INITIAL_TOTAL_COUNT
	}
	if totalInc <= 0 {
		totalInc = initialTotalCount
	}
	total := initialTotalCount
	skipPage := startSkipPage

	var cursor *mongo.Cursor
	defer func(){
		if err = cursor.Close(ctx); err != nil {
			fmt.Println(err)
		}
	}()
	
	for {
		findManyResTemp := make([]CollUserAccount, 0, limit)
		opts.SetSkip(int64(skipPage * limit))
		opts.SetLimit(int64(limit))
		cursor, err = FindManyDataToStructByBsonFilter(ctx, coll, filter, opts)
		if err != nil {
			return findManyRes, err
		}
		if err = cursor.All(ctx, &findManyResTemp); err != nil { // 即使读取的数量少于limit就结束了，也不会报错，当读取结果为空时表示查询结束
			return findManyRes, err
		}
		if len(findManyResTemp) > 0 {
			if skipPage * limit + limit >= total {
				total = total + totalInc
			}
			skipPage++
			findManyRes = append(findManyRes, findManyResTemp...)
		} else {
			break
		}
	}
	return findManyRes, err
}


func main() {
	// 创建客户端连接。
	ctx := context.TODO()
	credential := options.Credential{
		Username: MONGODB_USERNAME,
		Password: MONGODB_PASSWORD,
	}
	opts := options.Client().ApplyURI(MONGODB_URI_WITHOUT_CREDENTIAL).SetAuth(credential)
	client, err = GetMongoClient(ctx, opts) // 已声明全局变量

	// 指定db和集合
	coll = client.Database(DB_GLOBAL).Collection(COLL_USER_ACCOUNT)

	// 数据
	uid := "13732795"
	name := "Bob"

	// 删除所有数据
	fmt.Println("----------------")
	fmt.Println("DeleteManyData")
	deleteRes, err := DeleteManyData(ctx, coll, bson.M{})
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(deleteRes.DeletedCount)


	// 直接插入一条数据，如果数据出现唯一约束错误，会返回error
	fmt.Println("----------------")
	fmt.Println("AddOneData")
	insertRes, err := AddOneData(ctx, coll, &CollUserAccount{ // 传入变量地址
		Uid: uid,
		Name: name,
	})
	if err != nil {
		fmt.Println(err)
	}
	// 返回类型为*mongo.InsertOneResult，如果要返回主键就是insertRes.InsertedID类型是interface{}。注意不是InsertedId
	// 如果要把"_id"用到mongo中，应该用bson.D{{"_id",res.InsertedID}}包裹起来，或转成类型primitive.ObjectID
	fmt.Println(insertRes)
	fmt.Println(insertRes.InsertedID)
	// fmt.Println(insertRes.InsertedID.(primitive.ObjectID).Hex()) // 不同版本的mongo-driver处理主键ID的方式不同，过去将bsonID转成字符串：interface{}->底层类型bsonID->十六进制字符串
	fmt.Println(insertRes.InsertedID.(string))


	// 直接插入多条数据，如果数据出现唯一约束错误，会返回error
	fmt.Println("----------------")
	fmt.Println("AddManyData")
	insertManyRes, err := AddManyData(ctx, coll, []interface{}{
		&CollUserAccount{ // 传入变量地址
			Uid: uid + "a",
			Name: name,
		},
		&CollUserAccount{
			Uid: uid + "b",
			Name: name,
		},
	})
	if err != nil {
		fmt.Println(err)
	}
	// 返回类型为*mongo.InsertOneResult，如果要返回主键就是insertRes.InsertedID类型是interface{}。注意不是InsertedId
	// 如果要把"_id"用到mongo中，应该用bson.D{{"_id",res.InsertedID}}包裹起来，或转成类型primitive.ObjectID
	fmt.Println(insertManyRes)
	fmt.Println(insertManyRes.InsertedIDs)
	for _, v := range insertManyRes.InsertedIDs {
		fmt.Println(v)
	}


	// 不存在则插入
	fmt.Println("----------------")
	fmt.Println("UpsertOneData")
	// filter := bson.D{{"_id", uid}} // 注意bson.D{}是固定格式，里面才是表达式，所以不能少写最外层的{}
	filter := bson.D{{"uid", uid}} // 注意：不提供"_id"，也没指定哪个字段是主键时，mongo创建记录时将自动生成一个"_id"字段
	update := bson.D{{"$setOnInsert", bson.D{{"uid", uid}, {"name", name}}}}
	update_opts := options.Update().SetUpsert(true) // 当match失败时，如果upsert=true，将创建新数据。注意：返回*options.UpdateOptions
	update_record, err := UpsertOneData(ctx, coll, filter, update, update_opts) // 注意：UpdateOne返回的类型和InsertOne返回的类型不同，所以返回值不能赋值给同一个变量
	// update_record, err := coll.UpdateOne(context.TODO(), filter, update, update_opts)
	if err != nil {
		fmt.Println(err)
	} else {
		// 如果filter已匹配，则match=1,upsert=0表示匹配到所以不插入不更新。
		// 如果filter未匹配，则match=0,upsert=1表示未匹配到所以插入。
		fmt.Println(update_record.MatchedCount) // 如果发生err，将无法取得count，因为res将是值为<nil>的interface{}。报错：type interface{} has no field or method MatchedCount
		fmt.Println(update_record.UpsertedCount) // upserted行数
	}


	fmt.Println("----------------")
	fmt.Println("FindOneDataToBson")
	var result bson.M
	err = FindOneDataToBson(ctx, coll, bson.D{{"uid", uid}}, &result) // 传入地址
	// err = coll.FindOne(context.TODO(), bson.D{{"uid", uid}}).Decode(&result)
	if err != nil {
		if err == mongo.ErrNoDocuments {
			fmt.Printf("No document was found with the uid %s\n", uid)
		} else {
			fmt.Println(err)
		}
	}
	fmt.Println(result) // 如果返回err出错，result就不会被赋值


	fmt.Println("----------------")
	fmt.Println("FindOneDataToStructByBsonFilter")
	var resStruct1 = CollUserAccount{}
	err = FindOneDataToStructByBsonFilter(ctx, coll, bson.D{{"uid", uid}}, &resStruct1) // 传入地址。用bson.D作为filter的方式
	// err = coll.FindOne(context.TODO(), bson.D{{"uid", uid}}).Decode(&resStruct1)
	if err != nil {
		if err == mongo.ErrNoDocuments {
			fmt.Printf("No document was found with the uid %s\n", uid)
		} else {
			fmt.Println(err)
		}
	}
	fmt.Println(resStruct1)


	fmt.Println("----------------")
	fmt.Println("FindOneDataToStructByStructFilter")
	var resStruct2 = CollUserAccount{}
	// 用struct查询时，不能直接在Struct中添加filter作为条件。因为Struct中没添加的字段也会作为零值参与filter过程的，会出现错误。所以需要单独为filter设定一个新的struct才行
	err = FindOneDataToStructByStructFilter(ctx, coll, &CollUserAccountFilterByUid{Uid: uid}, &resStruct2) // 传入地址。用struct作为filter的方式
	// err = coll.FindOne(context.TODO(), bson.D{{"uid", uid}}).Decode(&resStruct2)
	if err != nil {
		if err == mongo.ErrNoDocuments {
			fmt.Printf("No document was found with the uid %s\n", uid)
		} else {
			fmt.Println(err)
		}
	}
	fmt.Println(resStruct2)


	// 查询多个文档，并迭代输出
	fmt.Println("----------------")
	fmt.Println("FindManyDataToStructByBsonFilter")
	// 注意skip表示跳过的数量，limit表示此次查询的总数。所以skip0limit1就是查1条。但是skip跳过时还是要一条条地去跳过，skip数量太大效率不好
	// 官方未实现深度分页功能，如果要用skip，需要自己在迭代过程中调整skip和limit值
	/*
	逻辑：skip是跳过的记录数=pageSize * page，limit是要读的数量
	//第一页
	db.collection.find().skip(0).limit(2) // 跳过0条记录，读2条
	//第二页
	db.collection.find().skip(2).limit(2) // 跳过2条记录，读2条
	//第三页
	db.collection.find().skip(4).limit(2) // 跳过4条记录，读2条
	*/
	iteratorFilter := bson.D{{"name", name}}
	startSkipPage := 0
	initialTotalCount := 4
	totalInc := 4
	limit := 2
	findManyRes := make([]CollUserAccount, 0, initialTotalCount)
	findOpts := options.Find() // 技巧：不需要传递多个*mongo.FindOptions，可以定义一个options.Find()返回*mongo.FindOptions结构体，后面的所有SetSort、SetSkip等都只是对这个FindOptions的修改，所以传入一个FindOptions就够了
	findOpts.SetSort(bson.D{{"_id", 1}}) // 不需要SetSkip和SetLimit，内部会自动拼接
	findManyRes, err = IteratorFindManyDataToStructByBsonFilter(ctx, coll, iteratorFilter, findOpts, int64(startSkipPage), int64(initialTotalCount), int64(totalInc), int64(limit))
	if err != nil {
		fmt.Println(err)
	}
	for _, findLine := range findManyRes {
		fmt.Println(findLine)
	}

	
	fmt.Println("----------------")
	fmt.Println("json")
	jsonData, err := json.MarshalIndent(result, "", "    ")
	fmt.Printf("%s\n", jsonData)
	if err != nil {
		fmt.Println(err)
	}
	fmt.Printf("%s\n", jsonData)

  
}


/*
第一遍运行：
# go run connectMongo.go
----------------
DeleteManyData
0
----------------
AddOneData
&{13732795}
13732795
13732795
----------------
AddManyData
&{[13732795a 13732795b]}
[13732795a 13732795b]
13732795a
13732795b
----------------
UpsertOneData
0
1
----------------
FindOneDataToBson
map[_id:ObjectID("6560816566ff417aaa128c1d") name:Bob uid:13732795]
----------------
FindOneDataToStructByBsonFilter
{6560816566ff417aaa128c1d Bob}
----------------
FindOneDataToStructByStructFilter
{6560816566ff417aaa128c1d Bob}
----------------
FindManyDataToStructByBsonFilter
{13732795 Bob}
{13732795a Bob}
{13732795b Bob}
{6560816566ff417aaa128c1d Bob}
----------------
json
{
    "_id": "6560816566ff417aaa128c1d",
    "name": "Bob",
    "uid": "13732795"
}
{
    "_id": "6560816566ff417aaa128c1d",
    "name": "Bob",
    "uid": "13732795"
}




第二遍运行：
# go run connectMongo.go
----------------
DeleteManyData
4
----------------
AddOneData
&{13732795}
13732795
13732795
----------------
AddManyData
&{[13732795a 13732795b]}
[13732795a 13732795b]
13732795a
13732795b
----------------
UpsertOneData
0
1
----------------
FindOneDataToBson
map[_id:ObjectID("6560817866ff417aaa128d1a") name:Bob uid:13732795]
----------------
FindOneDataToStructByBsonFilter
{6560817866ff417aaa128d1a Bob}
----------------
FindOneDataToStructByStructFilter
{6560817866ff417aaa128d1a Bob}
----------------
FindManyDataToStructByBsonFilter
{13732795 Bob}
{13732795a Bob}
{13732795b Bob}
{6560817866ff417aaa128d1a Bob}
----------------
json
{
    "_id": "6560817866ff417aaa128d1a",
    "name": "Bob",
    "uid": "13732795"
}
{
    "_id": "6560817866ff417aaa128d1a",
    "name": "Bob",
    "uid": "13732795"
}
*/

```

解释一下代码。

`mongo`包用于管理连接，提供Connect()、Disconnect()等方法。Connect()返回一个数据库连接。

```go
// ctx用于提供context限制，比如超时时间。
func Connect(ctx context.Context, opts ...*options.ClientOptions) (*Client, error)
```

`mongo/options`包提供连接需要的选项参数，比如`options.Client().ApplyURI("mongodb://localhost:27017")`

注意Connect()调用后，没有立刻请求DB，需要等首次访问DB才会发出请求。所以往往init_db的过程除了Connect()之外，还包含一次`connection.Ping()`的过程，确认是否真正可以连接到数据库。

使用Connect()返回的连接connection访问数据库。访问DB和collection的方式为`client.Database("global").Collection("user_account")`

查询方式：

```go
var result bson.M
err = coll.FindOne(context.TODO(), bson.D{{"uid", uid}}).Decode(&result)
```

bson格式是二进制的json。`bson`包提供对数据bson化的方法。`bson.D`表示一个文档(Document)，并且是有序的，所以格式为`bson.D{{k1, v1}, {k2, v2}}`。`bson.M`则是无序的文档，格式为`bson.M{k1:v1, k2:v2}`。

因为查询结果可以是多类型多字段的，所以一般用bson.M类型预定义一个变量，然后将查询结果写入预定义变量的内存中（注意result参数是指针），这样就避免在不知道结果集结构的情况下声明结果集struct。

注意，如果查询结果为空，是作为error返回的。所以`err!=nil`时，也可能是结果为空，需要判断`err==mongo.ErrNoDocuments`

其他见代码内注释

代码和常用写法可以参考：
```go
// 官方mongo：https://pkg.go.dev/go.mongodb.org/mongo-driver/mongo
// 官方options：https://pkg.go.dev/go.mongodb.org/mongo-driver/mongo/options
// github：https://github.com/ypankaj007/golang-mongodb-restful-starter-kit
// 知乎：https://zhuanlan.zhihu.com/p/144308830
```

## 主要参考

[Deploy a Replica Set in the Terminal](https://www.mongodb.com/docs/manual/tutorial/deploy-replica-set/)

[手把手教你搭建MongoDB集群搭建](https://www.cnblogs.com/jiagooushi/p/16477696.html)

[Docker搭建Mongodb集群](http://bazingafeng.com/2017/06/19/create-mongodb-replset-cluster-using-docker/)

[使用docker-compose创建mongoDB副本集](https://chunlife.top/2021/09/16/%E4%BD%BF%E7%94%A8docker-compose%E5%88%9B%E5%BB%BAmongoDB%E5%89%AF%E6%9C%AC%E9%9B%86/)

[docker 部署mongoDB集群与读写分离 ](https://www.cnblogs.com/ejiyuan/p/17255287.html)

[分片集群](https://www.cnblogs.com/syushin/p/15633183.html)
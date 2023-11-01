# 队列 RabbitMQ 实践

## 概念

RabbitMQ只是队列结构的消息代理，用来接受、存储和转发二进制数据消息。

队列只受主机的内存和磁盘限制，它本质上是一个大的**消息缓冲区**。许多生产者可以向一个队列发送消息，而许多消费者可以尝试从一个队列接收数据。

在一般图例中，P是生产者，C是消费者。中间的方框是一个队列——RabbitMQ代表消费者保存的消息缓冲区。

建议消费(推)，不要轮询(拉)消息。确保您的消费者消费来自队列的消息，而不是使用基本的get操作。

### 连接、通道、队列的关系

生产者、消费者与RabbitMQ server建立连接Connection。AMQP 连接通常是长连接。AMQP 是一个使用 TCP 提供可靠投递的应用层协议。AMQP 使用认证机制并且提供 TLS（SSL）保护。当一个应用不再需要连接到 AMQP 代理的时候，需要优雅的释放掉 AMQP 连接，而不是直接将 TCP 连接关闭。

在Connection中可能存在多个Channel通道，可以把通道理解成共享一个 TCP 连接的多个轻量化连接。

消息通过连接发送到服务器上首先接收到的为交换机（Exchange），在没有使用交换机的简单模式中，实际上则使用的是默认的交换机（AMQP-Default）。

在使用交换机前我们需要对交换机进行声明（declared ），声明之后我们还需要将消息队列绑定到交换机上。

交换机根据消息信息以及交换机绑定的队列信息，分发到相应的消息队列（Queue）上。

最后RabbitMQ server通过消费端的连接发送到消费者客户端上。

RabbitMQ消息模型的核心思想是生产者永远不会直接向队列发送任何消息。实际上，通常生产者甚至不知道消息是否将被传递到任何队列。根据确定的规则，RabbitMQ将会决定消息该投递到哪个队列。 这些规则称为路由键（routing key），队列通过路由键绑定到交换机上。

### 死信队列 Dead Letter Queue

[Dead Letter Exchanges](https://www.rabbitmq.com/dlx.html)

过期时间TTL表示可以对消息设置预期的时间，在这个时间内都可以被消费者接收获取；过了之后消息将自动被删除。RabbitMQ可以对消息和队列设置TTL。如果上述两种方法同时使用，则消息的过期时间以两者之间TTL较小的那个数值为准。消息在队列的生存时间一旦超过设置的TTL值，就称为dead message被投递到死信队列， 消费者将无法再收到该消息。

而死信队列DLX，全称为Dead-Letter-Exchange , 可以称之为死信交换机，也有人称之为死信邮箱。

当消息在一个队列中变成死信(dead message)之后，它能被重新发送到另一个交换机中，这个交换机就是DLX ，绑定DLX的队列就称之为死信队列。

消息变成死信，可能是由于以下的原因：
- 消息被拒绝
- 消息过期(注意：不是队列过期，如果队列过期，队列内的消息不是死信)
- 队列达到最大长度（消息溢出）

DLX也是一个正常的交换机，和一般的交换机没有区别，它能在任何的队列上被指定，实际上就是设置某一个队列的属性。

当这个队列中存在死信时，Rabbitmq就会自动地将这个消息重新发布到设置的DLX上去，进而被路由到另一个队列，即死信队列。

要想使用死信队列，只需要在定义队列的时候设置队列参数 x-dead-letter-exchange 指定交换机即可。

有可能形成消息死信循环。例如，当将死信消息队列发送到默认交换机而没有指定死信路由键时，就会发生这种情况。如果在整个周期中没有拒绝消息，则丢弃此类周期中的消息(即两次到达同一队列的消息)。

默认情况下，在内部未打开发布者确认的情况下重新发布死信消息。因此，在集群RabbitMQ环境中使用DLX并不能保证是安全的。消息在发布到DLX目标队列后立即从原始队列中删除。这确保了不会产生过多的消息积累，从而耗尽代理资源。但是，如果目标队列无法接受消息，则可能会丢失消息。

死信处理过程向每条死信消息的头部添加一个名为x-death的数组。此数组包含每个死信事件的条目，由一对{queue, reason}标识。每个这样的条目都是一个由几个字段组成的表:
- queue:消息变成死信之前所在的队列的名称
- reason:死信的原因(下文将进一步描述)
- time:称为死信的日期和时间，为64位AMQP 0-9-1时间戳
- exchange:消息发布到的交换机(注意，如果消息多次被封为死信，则此交换机为死信交换机)
- routing-keys:发布消息时使用的路由密钥(包括抄送密钥CC但不包括密件抄送密钥BCC)
- count:由于reason，此消息在此队列中成为死信的次数
- original-expiration(如果消息由于`per-message TTL`而成为死信):消息的原始过期属性`expiration`。从死信消息中删除过期属性，以防止它在路由到的任何队列中再次过期。

新条目被添加到x-death数组的开头。在x-death已经包含具有相同队列和死字原因的条目的情况下，它的count字段将被增加并移动到数组的开头。

reason描述了为什么消息是死信，包括
- rejected:消息被拒绝，队列参数设置为false
- expired:消息TTL已过期
- maxlen:超过了允许的最大队列长度
- delivery_limit:消息返回的次数超过限制(由仲裁队列的策略参数delivery-limit设置)。

### 延迟队列

延迟队列存储的对象是对应的延迟消息；所谓`延迟消息` 是指当消息被发送以后，并不想让消费者立刻拿到消息，而是等待特定时间后，消费者才能拿到这个消息进行消费。

在RabbitMQ中延迟队列可以通过 过期时间 + 死信队列 来实现，也就是说生产者将消息写入队列1，直到消息过期都没有消费并被RabbitMQ转入队列2，消费者一直在等着消费队列2，这样就实现了延迟队列。

延迟队列的应用场景
- 在电商项目中的支付场景；如果在用户下单之后的几十分钟内没有支付成功；那么这个支付的订单算是支付失败，要进行支付失败的异常处理（将库存加回去），这时候可以通过使用延迟队列来处理
- 在系统中如有需要在指定的某个时间之后执行的任务都可以通过延迟队列处理

### 消息确认与事务

确认并且保证消息被送达，提供了两种方式：发布确认和事务。

两者不可同时使用。在channel为事务时，不可引入确认模式；同样channel为确认模式下，不可使用事务。

```go
func (ch *Channel) Tx() error
```

Tx将服务器上的Channel置于事务模式。遵循此方法的所有发布和确认将为单个队列自动提交或回滚。调用`Channel.TxCommit` 或 `Channel.TxRollback`以离开此事务并立即开始新事务。

跨多个队列的原子性没有定义为队列声明，并且绑定不包括在事务中。

当Channel处于事务中时，作为强制或立即交付的发布行为没有定义。

Channel一旦进入事务模式，就不能从事务模式中取出。为非事务性语义使用不同的通道。

```go
func (ch *Channel) TxCommit() error
```

TxCommit自动提交单个队列的所有发布和确认，并立即启动一个新事务。

在没有调用Channel.Tx的情况下调用此方法是一个错误。

```go
func (ch *Channel) TxRollback() error
```

TxRollback自动回滚单个队列的所有发布和确认，并立即启动新事务。

在没有调用Channel的情况下调用此方法。Tx是一个错误。

### 消息追踪Trace

消息中心的消息追踪需要使用Trace实现，Trace是Rabbitmq用于记录每一次发送的消息，方便使用Rabbitmq的开发者调试、排错。可通过插件形式提供可视化界面。

Trace启动后会自动创建系统Exchange：amq.rabbitmq.trace ,每个队列会自动绑定该Exchange，绑定后发送到队列的消息都会记录到Trace日志。

查看插件列表：`rabbitmq-plugins list`

启用trace插件：`rabbitmq-plugins enable rabbitmq_tracing`

打开trace开关：`rabbitmqctl trace_on`

安装插件并开启 trace_on 之后，会发现多个 exchange：amq.rabbitmq.trace ，类型为：topic。(后台查看可能需要重新登录)

在后台->Admin->Tracing->添加一个trace->All traces中，可以看到Trace log files

通过账号密码就能看到追踪日志，比如这里用匿名exchange生产和消费了"hello"消息

```json

================================================================================
2023-10-31 13:52:45:102: Message published

Node:         rabbit@55b3aad61783
Connection:   172.17.0.1:49450 -> 172.17.0.2:5672
Virtual host: /
User:         admin
Channel:      1
Exchange:     
Routing keys: [<<"hello">>]
Routed queues: [<<"hello">>]
Properties:   [{<<"content_type">>,longstr,<<"text/plain">>}]
Payload: 
Hello World!

================================================================================
2023-10-31 13:52:45:105: Message received

Node:         rabbit@55b3aad61783
Connection:   172.17.0.1:47552 -> 172.17.0.2:5672
Virtual host: /
User:         admin
Channel:      1
Exchange:     
Routing keys: [<<"hello">>]
Queue:        hello
Properties:   [{<<"content_type">>,longstr,<<"text/plain">>}]
Payload: 
Hello World!
```

### 集群

[【深度知识】RabbitMQ的四种集群架构](https://cloud.tencent.com/developer/article/1795578)

[Clustering Guide](https://www.rabbitmq.com/clustering.html)

TODO

### amqp的消息（amqp.Publishing）

AMQP 模型中的消息（Message）对象是带有属性（Attributes）的。

- Content type（内容类型）
- Content encoding（内容编码）
- Routing key（路由键）
- Delivery mode (persistent or not)  投递模式（持久化 或 非持久化）
- Message priority（消息优先权）
- Message publishing timestamp（消息发布的时间戳）
- Expiration period（消息有效期）
- Publisher application id（发布应用的ID）

AMQP 的消息除属性外，也含有一个有效载荷 - Payload（消息实际携带的数据），它被 AMQP 代理当作不透明的字节数组来对待。消息代理不会检查或者修改有效载荷。消息可以只包含属性而不携带有效载荷。

消息能够以持久化的方式发布，AMQP 代理会将此消息存储在磁盘上。如果服务器重启，系统会确认收到的持久化消息未丢失。简单地将消息发送给一个持久化的交换机或者路由给一个持久化的队列，并不会使得此消息具有持久化性质：它完全取决与消息本身的持久模式（persistence mode）。将消息以持久化方式发布时，会对性能造成一定的影响（就像数据库操作一样，健壮性的存在必定造成一些性能牺牲）。

### 消息确认 acknowledge

由于网络的不确定性和应用失败的可能性，处理确认回执（acknowledgement）就变的十分重要。有时我们确认消费者收到消息就可以了，有时确认回执意味着消息已被验证并且处理完毕，例如对某些数据已经验证完毕并且进行了数据存储或者索引操作。

这种情形很常见，所以 AMQP 0-9-1 内置了一个功能叫做消息确认（message acknowledgements），消费者用它来确认消息已经被接收或者处理。如果一个应用崩溃掉（此时连接会断掉，所以 AMQP 代理亦会得知），而且消息的确认回执功能已经被开启，但是消息代理尚未获得确认回执，那么消息会被从新放入队列（并且在还有其他消费者存在于此队列的前提下，立即投递给另外一个消费者）。

协议内置的消息确认功能将帮助开发者建立强大的软件。

## 命令 rabbitmqctl

列出交换器：`rabbitmqctl list_exchanges`

查看状态：`rabbitmqctl status`

查看集群状态：`rabbitmqctl cluster_status`

rabbitmq_management 插件可以启用作为web管理后台，会占用15672端口

rabbitmqctl 官方指令可见 [rabbitmqctl(8)](https://www.rabbitmq.com/rabbitmqctl.8.html)

## API

### 创建通道channel

```go
conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
ch, err := conn.Channel()
```

创建一个通道，大多数RabbitMQ提供的API都驻留在通道中

### Channel.QueueDeclare 声明队列

```go
func (ch *Channel) QueueDeclare(name string, durable, autoDelete, exclusive, noWait bool, args Table) (Queue, error)
```

不带任何参数的 queueDeclare 方法默认创建一个由 RabbitMQ 命名的(类似这种 amq.gen-LhQzlgv3GhDOv8PIDabOXA 名称，这种队列也称之为匿名队列〉、排他的、自动删除 的、非持久化的队列。

RabbitMQ不允许你用不同的参数重新定义一个已经存在的队列，任何试图这样做的程序都会返回一个错误。并且应用到生产者和消费者的参数应该是一致的，比如durable。

queueDeclare方法的参数详细说明
- queue 队列的名称。
- durable: 设置是否持久化。为 true 则设置队列为持久化。**持久化的队列会存盘，在服务器重启的时候可以保证不丢失相关信息**。
- exclusive 设置是否排他。为 true 则设置队列为排他的。**如果一个队列被声明为排 他队列，该队列仅对首次声明它的连接可见，并在连接断开时自动删除。**这里需要注意 三点:排他队列是基于连接(connection) 可见的，同一个连接的不同信道 (Channel) 是可以同时访问同一连接创建的排他队列; "首次"是指如果一个连接己经声明了排他队列，其他连接是不允许建立同名的排他队列的，这个与普通队列不同:即使该队列是持久化的，一旦连接关闭或者客户端退出，该排他队列都会被自动删除，这种队列适用于一个客户端同时发送和读取消息的应用场景。
- autoDelete: 设置是否自动删除。为 true 则设置队列为自动删除。自动删除的前提是: **至少有一个消费者连接到这个队列，之后所有与这个队列连接的消费者都断开时，才会自动删除**。不能把这个参数错误地理解为: "当连接到此队列的所有客户端断开时，这个队列自动删除"，因为生产者客户端创建这个队列，或者**没有消费者客户端与这个队列连接时，都不会自动删除这个队列**。
- no-wait: 新增。是否需要等待返回。TRUE 表示不需要服务端的任何返回，no-wate之后，紧接着使用声明的队列时有可能会发生异常情况。
- argurnents: 设置队列的其他一些参数，如 x-message-ttl 、x-expires 、x-max-length 、x-max-length-bytes、 x-dead-letter-exchange、 x-deadletter-routing-key、 x-max-priority 等。
  - x-message-ttl，队列中消息的ttl。必须是非负 32 位整数 (0 <= n <= 2^32-1) ，以毫秒为单位表示 TTL 的值，这会影响队列中每条消息的 TTL。如果要为单一消息设置 TTL 需要在 message 中设置 expiration。
  - x-expires，队列的ttl。
  - x-max-length，队列最大长度。
  - x-max-length-bytes，队列中最大字节数。
  - x-dead-letter-exchange，当出现死信时指定使用死信队列的交换器
  - x-deadletter-routing-key，如果您将消息发布到具有foo路由键的交换器，并且该消息是死信的，则将其发布到具有foo路由键的死信交换器。如果声明消息最初到达的队列时，将x-dead-letter-routing-key设置为bar，则将消息发布到具有bar路由键的死信交换中。
  - 等

### queueDeclarePassive 检测队列是否存在

```go
func (ch *Channel) QueueDeclarePassive(name string, durable, autoDelete, exclusive, noWait bool, args Table) (Queue, error)
```

### queueDelete 删除队列

```go
func (ch *Channel) QueueDelete(name string, ifUnused, ifEmpty, noWait bool) (int, error)
```

- queue 表示队列的名称
- ifEmpty 设置为 true 表示在队列为空(队列里面没有任何消息堆积)的情况下才能够删除。
- noWait 不需要等待返回

### queuePurge 清空队列数据

```go
func (ch *Channel) QueuePurge(name string, noWait bool) (int, error)
```

清空队列中的内容(purge)，而不删除队列本身，类似MySQL的truncate。

### Channel.PublishWithContext 发布

```go
func (ch *Channel) PublishWithContext(ctx context.Context, exchange, key string, mandatory, immediate bool, msg Publishing) error
```

从客户端发送一个publish到服务器上的exchange。

- exchange和key：当您希望将单个消息传递到单个队列时，可以使用队列名称（q.Name）的routingKey将消息发布到默认交换机（空字符串""）。这是因为每个声明的队列都有一个到默认交换器的隐式路由。
- mandatory：强制为true，并且没有绑定与routingKey匹配的队列时，或者当立即标志immediate为真并且匹配队列上没有消费者准备好接受交付时，Publishing可能无法送达。
- 返回：当通道、连接或套接字关闭时，可能会返回一个错误。有没有错误并不表明服务器是否已收到此发布。

### Channel.Qos 消息质保配置

```go
func (ch *Channel) Qos(prefetchCount, prefetchSize int, global bool) error
```

Qos控制服务器在接收到交付ack之前将尝试在网络上为消费者保留多少消息或多少字节。Qos的目的是确保服务器和客户端之间的网络缓冲区保持满。在Consume()消费之前配置。

如果预取计数大于零，则服务器将在收到确认之前向消费者传递相同数量的消息。当消费者使用noAck启动时，服务器将忽略此选项，因为不需要或发送任何确认。

如果预取size大于0，服务器将尝试在接收到来自消费者的确认之前至少保持那么多字节的交付刷新到网络。当消费者使用noAck启动时，此选项将被忽略。

当global为true时，这些Qos设置适用于同一连接上所有通道上的所有现有和未来的消费者。当为假时，通道。Qos设置将适用于该通道上所有现有和未来的消费者。

要在不同连接上从同一队列消费的消费者之间获得循环行为，请将预取计数设置为1，服务器上的下一个可用消息将被传递给下一个可用的消费者。

如果你的消费者工作时间是相当一致的，并且不超过你的网络往返时间的两倍，那么你将看到显著的吞吐量改进，从预取计数2开始，或者根据RabbitMQ上的基准测试描述的稍微大一点。

### Channel.Consume 消费

```go
func (ch *Channel) Consume(queue, consumer string, autoAck, exclusive, noLocal, noWait bool, args Table) (<-chan Delivery, error)
```

Consume立即开始传递队列中的消息。不间断地返回Delivery结构数据，直到Channel.Cancel, Connection.Close, Channel.Close, or an AMQP exception发生。所以需要迭代返回的数据。

AMQP协议中的所有交付都必须确认Ack。消费者应该在成功接收数据后调用Delivery.Ack发出确认消息。在成功处理发货后回复。如果消费者被取消或通道或连接被关闭，任何未确认的交付将在同一队列的末尾重新排队。

当autoAck(也称为noAck)为true时，服务器将在向网络写入交付之前向该消费者确认交付。当autoAck为true时，消费者不应该调用deliver.ack。自动确认交付意味着，如果消费者在服务器交付后无法处理它们，则可能会丢失一些交付。如果autoAck为false，需要在每一次迭代中都响应msg.Ack(false)

当exclusive为true时，服务器将确保这是该队列的唯一消费者。当exclusive为false时，服务器将在多个消费者之间公平地分发交付。

当noWait为true时，不要等待服务器确认请求并立即开始交付。如果无法使用，则会引发通道异常并关闭通道。

当通道或连接关闭时，所有缓冲消息和机上消息将被丢弃。RabbitMQ将对未被确认的消息进行排队。换句话说，以这种方式丢弃的消息不会丢失。

### amqp.Publishing 消息结构

消息分为属性和消息（字节数组）。举例

```go
amqp.Publishing{
	// Copy all the properties
	ContentType:     msg.ContentType,
	ContentEncoding: msg.ContentEncoding,
	DeliveryMode:    msg.DeliveryMode,
	Priority:        msg.Priority,
	CorrelationId:   msg.CorrelationId,
	ReplyTo:         msg.ReplyTo,
	Expiration:      msg.Expiration,
	MessageId:       msg.MessageId,
	Timestamp:       msg.Timestamp,
	Type:            msg.Type,
	UserId:          msg.UserId,
	AppId:           msg.AppId,

	// Custom headers
	Headers: msg.Headers,

	// And the body
	Body: msg.Body,
}
```
其中，消息属性为
```go
type Publishing struct {
	// Application or exchange specific fields,
	// the headers exchange will inspect this field.
	Headers Table

	// Properties
	ContentType     string    // MIME content type，如"text/plain", "application/json"
	ContentEncoding string    // MIME content encoding
	DeliveryMode    uint8     // Transient (0 or 1) or Persistent (2)，暂存还是永久存储
	Priority        uint8     // 0 to 9，优先级，数字越大代表优先级越高，消息如果没有设置priority优先级字段，那么priority字段值默认为0
	CorrelationId   string    // correlation identifier，透传Id，将RPC响应与请求关联起来。一般设置为每个请求的唯一值。客户端等待回调队列上的数据，当消息出现时，他检查correlationId，如果它和从请求返回的值匹配，就进行响应。
	ReplyTo         string    // address to to reply to (ex: RPC)，可以用回调队列的名字
	Expiration      string    // message expiration spec，过期时间（毫秒）
	MessageId       string    // message identifier
	Timestamp       time.Time // message timestamp
	Type            string    // message type name
	UserId          string    // creating user id - ex: "guest"
	AppId           string    // creating application id

	// The application specific payload of the message
	Body []byte
}

```

### 交互数据结构

```go
type Delivery struct {
	Acknowledger Acknowledger // the channel from which this delivery arrived

	Headers Table // Application or header exchange table

	// Properties
	ContentType     string    // MIME content type
	ContentEncoding string    // MIME content encoding
	DeliveryMode    uint8     // queue implementation use - non-persistent (1) or persistent (2)
	Priority        uint8     // queue implementation use - 0 to 9
	CorrelationId   string    // application use - correlation identifier
	ReplyTo         string    // application use - address to reply to (ex: RPC)
	Expiration      string    // implementation use - message expiration spec
	MessageId       string    // application use - message identifier
	Timestamp       time.Time // application use - message timestamp
	Type            string    // application use - message type name
	UserId          string    // application use - creating user - should be authenticated user
	AppId           string    // application use - creating application id

	// Valid only with Channel.Consume
	ConsumerTag string

	// Valid only with Channel.Get
	MessageCount uint32

	DeliveryTag uint64
	Redelivered bool
	Exchange    string // basic.publish exchange
	RoutingKey  string // basic.publish routing key

	Body []byte
}
```
```go
type Return struct {
	ReplyCode  uint16 // reason
	ReplyText  string // description
	Exchange   string // basic.publish exchange
	RoutingKey string // basic.publish routing key

	// Properties
	ContentType     string    // MIME content type
	ContentEncoding string    // MIME content encoding
	Headers         Table     // Application or header exchange table
	DeliveryMode    uint8     // queue implementation use - non-persistent (1) or persistent (2)
	Priority        uint8     // queue implementation use - 0 to 9
	CorrelationId   string    // application use - correlation identifier
	ReplyTo         string    // application use - address to to reply to (ex: RPC)
	Expiration      string    // implementation use - message expiration spec
	MessageId       string    // application use - message identifier
	Timestamp       time.Time // application use - message timestamp
	Type            string    // application use - message type name
	UserId          string    // application use - creating user id
	AppId           string    // application use - creating application

	Body []byte
}
```

### Channel.QueryBind 将交换器绑定到队列

```go
func (ch *Channel) QueueBind(name, key, exchange string, noWait bool, args Table) error
```
QueueBind将交换器绑定到队列，这样当发布routing key与绑定routing key匹配时，对交换器的发布将被路由到队列。

绑定key的含义取决于交换类型。比如扇出交换会忽略了它的值。

## 容器

TODO

开放5672端口，需要账号密码访问。

连接字符串：`amqp://admin:admin@host.docker.internal:5672/`

## 后台

官方[Management Plugin](https://www.rabbitmq.com/management.html)

TODO

## 实践

### 使用默认Exchange的hello例子

生产者（消息发布者）sender.go
```go
package main

import (
	"context"
	"log"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

// 检查每个amqp调用的返回值
func failOnError(err error, msg string) {
	if err != nil {
		log.Panicf("%s: %s", msg, err)
	}
}

func main() {
	// 连接到 RabbitMQ 服务器
	conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
	failOnError(err, "Failed to connect to RabbitMQ")
	defer conn.Close()

	// 创建一个通道，用于完成任务的大多数API都驻留在通道中
	ch, err := conn.Channel()
	failOnError(err, "Failed to open a channel")
	defer ch.Close()

	// 要发送，我们必须声明一个要发送到的队列
	// 声明一个队列是幂等的——只有当它不存在时才会创建它。
	q, err := ch.QueueDeclare(
		"hello", // name
		false,   // durable
		false,   // delete when unused
		false,   // exclusive
		false,   // no-wait
		nil,     // arguments
	)
	failOnError(err, "Failed to declare a queue")

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	body := "Hello World!"
	// 将消息发布到声明的队列
	err = ch.PublishWithContext(ctx,
		"",     // exchange
		q.Name, // routing key
		false,  // mandatory
		false,  // immediate
		amqp.Publishing{
			ContentType: "text/plain",
			Body:        []byte(body),	// 消息内容是一个字节数组
		})
	failOnError(err, "Failed to publish a message")
	log.Printf(" [x] Sent %s\n", body)
}

```


消费者（消息接收者）receiver.go

```go
package main

import (
	"log"

	amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
	if err != nil {
		log.Panicf("%s: %s", msg, err)
	}
}

func main() {
	conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
	failOnError(err, "Failed to connect to RabbitMQ")
	defer conn.Close()

	ch, err := conn.Channel()
	failOnError(err, "Failed to open a channel")
	defer ch.Close()

	q, err := ch.QueueDeclare(
		"hello", // name
		false,   // durable
		false,   // delete when unused
		false,   // exclusive
		false,   // no-wait
		nil,     // arguments
	)
	failOnError(err, "Failed to declare a queue")

	msgs, err := ch.Consume(
		q.Name, // queue
		"",     // consumer
		true,   // auto-ack
		false,  // exclusive
		false,  // no-local
		false,  // no-wait
		nil,    // args
	)
	failOnError(err, "Failed to register a consumer")

	var forever chan struct{}

	go func() {
		for d := range msgs {
			log.Printf("Received a message: %s", d.Body)
		}
	}()

	log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
	<-forever
}
```

### 使用配置过的工作队列应用到多生产消费者场景

```go
// task.go
package main

import (
        "context"
        "log"
        "os"
        "strings"
        "time"

        amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Panicf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        q, err := ch.QueueDeclare(
                "task_queue", // name
                true,         // durable
                false,        // delete when unused
                false,        // exclusive
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare a queue")

        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        body := bodyFrom(os.Args)
        err = ch.PublishWithContext(ctx,
                "",           // exchange
                q.Name,       // routing key
                false,        // mandatory
                false,
                amqp.Publishing{
                        DeliveryMode: amqp.Persistent,
                        ContentType:  "text/plain",
                        Body:         []byte(body),
                })
        failOnError(err, "Failed to publish a message")
        log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "hello"
        } else {
                s = strings.Join(args[1:], " ")
        }
        return s
}
```


```go
// worker.go
package main

import (
        "bytes"
        "log"
        "time"

        amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Panicf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        q, err := ch.QueueDeclare(
                "task_queue", // name
                true,         // durable
                false,        // delete when unused
                false,        // exclusive
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare a queue")

        err = ch.Qos(
                1,     // prefetch count // 预取数量，超过1表示客户端还没返回ack时，服务器就可以再次向客户端发送第二轮迭代的数据。
                0,     // prefetch size
                false, // global
        )
        failOnError(err, "Failed to set QoS")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                false,  // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        var forever chan struct{}

        go func() {
                for d := range msgs {
                        log.Printf("Received a message: %s", d.Body)
                        dotCount := bytes.Count(d.Body, []byte("."))
                        t := time.Duration(dotCount)
                        time.Sleep(t * time.Second)
                        log.Printf("Done")
                        d.Ack(false) // 每次迭代都要ack
                }
        }()

        log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
        <-forever
}
```

运行：
```shell
// shell1 消费者 等待直到消息到来
# go run 2_worker.go
2023/10/30 09:55:25  [*] Waiting for messages. To exit press CTRL+C
2023/10/30 09:55:25 Received a message: hello
2023/10/30 09:55:25 Done
2023/10/30 09:58:58 Received a message: world
2023/10/30 09:58:58 Done
```
```shell
// shell2 生产者 等待直到自动发出hello消息，或手动发送其他消息
# go run 2_task.go
2023/10/30 09:58:44  [x] Sent hello
# go run 2_task.go world
2023/10/30 09:58:58  [x] Sent world
```

### 使用发布订阅模式(fanout扇出)的实时日志收集器例子

相反，生产者只能向交换器发送消息。交换是一件非常简单的事情。它一边接收来自生产者的消息，另一边将它们推送到队列中。交换器必须确切地知道如何处理它收到的消息。是否应该将它附加到特定的队列中?是否应该将它附加到许多队列中?或者它应该被丢弃。该规则由交换类型定义。

交换器:生产者只能向交换器发送消息。交换的一端接收来自生产者的消息，另一端将它们推送到队列中。

有几种交换类型可用:直接，主题(模式匹配)，标题和扇出(direct, topic, headers and fanout.)

fanout扇出，它只是将它接收到的所有消息广播到它所知道的所有队列。

交换器和队列之间的关系称为绑定。

对于日志发布订阅系统来说，我们希望听到所有的日志消息，而不仅仅是其中的一个子集。我们也只对当前流动的消息感兴趣，而不是旧的消息。

首先，无论何时连接到Rabbit，我们都需要一个新的空队列。要做到这一点，我们可以创建一个随机名称的队列，或者，更好的是——让服务器为我们选择一个随机的队列名称。

其次，一旦断开消费者的连接，队列就会被自动删除。

当声明它的连接关闭时，队列将被删除，因为它被声明为排他的。

总结一下，发布订阅模式就是用fanout的交换器，将数据保存到匿名队列，以求得到实时日志。

```go
// emit_log.go
package main

import (
        "context"
        "log"
        "os"
        "strings"
        "time"

        amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Panicf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs",   // name
                "fanout", // type
                true,     // durable
                false,    // auto-deleted
                false,    // internal
                false,    // no-wait
                nil,      // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        body := bodyFrom(os.Args)
        err = ch.PublishWithContext(ctx,
                "logs", // exchange
                "",     // routing key // 不提供队列名，让Rabbit为我们提供一个随机队列，例如amq.gen-JzTY20BRgKO-HjmUJj0wLg
                false,  // mandatory
                false,  // immediate
                amqp.Publishing{
                        ContentType: "text/plain",
                        Body:        []byte(body),
                })
        failOnError(err, "Failed to publish a message")

        log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "hello"
        } else {
                s = strings.Join(args[1:], " ")
        }
        return s
}
```

```go
// receive_logs.go

package main

import (
        "log"

        amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Panicf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs",   // name
                "fanout", // type
                true,     // durable
                false,    // auto-deleted
                false,    // internal
                false,    // no-wait
                nil,      // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when unused
                true,  // exclusive
                false, // no-wait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

        // QueueBind将交换器绑定到队列，这样当发布routing key与绑定routing key匹配时，对交换器的发布将被路由到队列。
        err = ch.QueueBind(
                q.Name, // queue name // 从Rabbit生成的随机队列获取数据
                "",     // routing key
                "logs", // exchange
                false,
                nil,
        )
        failOnError(err, "Failed to bind a queue")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        var forever chan struct{}

        go func() {
                for d := range msgs {
                        log.Printf(" [x] %s", d.Body)
                }
        }()

        log.Printf(" [*] Waiting for logs. To exit press CTRL+C")
        <-forever
}
```

### 路由绑定，让指定的队列处理指定路由key的消息

用routing-key将某个或某些队列绑定到交换器上。这可以简单地理解为:队列对来自此交换的消息感兴趣。

绑定Channel.QueueBind可以接受一个额外的routing_key参数。为避免与Channel.Publish混淆，我们将称它为绑定key。

上一教程中的日志系统将所有消息广播给所有消费者。我们希望对其进行扩展，以允许根据消息的严重程度过滤消息。例如，我们可能希望将日志消息写入磁盘的脚本只接收关键错误，而不是在警告或信息日志消息上浪费磁盘空间。我们将采用直接交换的方式。直接交换背后的路由算法很简单——消息被送到与其绑定key完全匹配的队列中。

使用相同的绑定key将交换器绑定到多个队列是完全合法的。

```go
// emit_log_direct.go
package main

import (
        "context"
        "log"
        "os"
        "strings"
        "time"

        amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Panicf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs_direct", // name
                "direct",      // type
                true,          // durable
                false,         // auto-deleted
                false,         // internal
                false,         // no-wait
                nil,           // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        body := bodyFrom(os.Args)
        err = ch.PublishWithContext(ctx,
                "logs_direct",         // exchange
                severityFrom(os.Args), // routing key
                false, // mandatory
                false, // immediate
                amqp.Publishing{
                        ContentType: "text/plain",
                        Body:        []byte(body),
                })
        failOnError(err, "Failed to publish a message")

        log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
        var s string
        if (len(args) < 3) || os.Args[2] == "" {
                s = "hello"
        } else {
                s = strings.Join(args[2:], " ")
        }
        return s
}

func severityFrom(args []string) string {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "info"
        } else {
                s = os.Args[1]
        }
        return s
}
```

```go
// receive_logs_direct.go
package main

import (
        "log"
        "os"

        amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Panicf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs_direct", // name
                "direct",      // type
                true,          // durable
                false,         // auto-deleted
                false,         // internal
                false,         // no-wait
                nil,           // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when unused
                true,  // exclusive
                false, // no-wait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

        if len(os.Args) < 2 {
                log.Printf("Usage: %s [info] [warning] [error]", os.Args[0])
                os.Exit(0)
        }
        for _, s := range os.Args[1:] {
                log.Printf("Binding queue %s to exchange %s with routing key %s",
                        q.Name, "logs_direct", s)
                err = ch.QueueBind(
                        q.Name,        // queue name
                        s,             // routing key
                        "logs_direct", // exchange
                        false,
                        nil)
                failOnError(err, "Failed to bind a queue")
        }

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto ack
                false,  // exclusive
                false,  // no local
                false,  // no wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        var forever chan struct{}

        go func() {
                for d := range msgs {
                        log.Printf(" [x] %s", d.Body)
                }
        }()

        log.Printf(" [*] Waiting for logs. To exit press CTRL+C")
        <-forever
}
```

这里生产者使用routing-key（如果不提供就使用"info"）将exchange通过direct绑定到匿名queue。

消费者同样，提供exchange和routing-key绑定到对应的匿名queue，就可以进行消费。

```shell
# go run 4_receive_logs_direct.go info warn
// 生成匿名queue，并通过两个routing-key "info" "warn" 将队列amq.gen-Fm0riArPWxSiuNufFlJnyA 绑定到交换器logs_direct
2023/10/31 06:30:14 Binding queue amq.gen-Fm0riArPWxSiuNufFlJnyA to exchange logs_direct with routing key info
2023/10/31 06:30:14 Binding queue amq.gen-Fm0riArPWxSiuNufFlJnyA to exchange logs_direct with routing key warn
2023/10/31 06:30:14  [*] Waiting for logs. To exit press CTRL+C
// 等待生产者传输数据到exchange及存储到匿名queue...
2023/10/31 06:30:31  [x] a warning
2023/10/31 06:58:37  [x] a info
```

```shell
# go run 4_emit_log_direct.go warn "a warning"
// 生产者将log发送到交换器logs_direct，并指定routing-key为命令行指定参数 "warn"
2023/10/31 06:30:31  [x] Sent a warning
# go run 4_emit_log_direct.go info "a info"
2023/10/31 06:58:37  [x] Sent a info
# go run 4_emit_log_direct.go error "a error" 
// 因为QueueBind时没有为exchange指定"error"的routing-key，所有没有队列可以接收"error"消息，消费者自然也就无法消费"error"
2023/10/31 06:57:07  [x] Sent a error
```

### 使用Topic主题模式（发布订阅）过滤路由关键字

发送到topic的消息不能有任意的routing_key——它必须是一个用点分隔的单词列表。单词可以是任何东西，但通常它们指定与信息相关的一些特征。下面是一些有效的路由关键字示例:`"quick.orange.rabbit"`。routing key中可以有任意多的单词，最多不超过255字节。

binding key也必须是相同的形式。top背后的逻辑与direct类似——使用特定路由键发送的消息将被传递到使用匹配绑定键绑定的所有队列。但是，绑定键有两种重要的特殊情况:
- `*(star)`只能代替一个单词。
- `# (hash)`可以代替零个或多个单词。

注意
- 当绑定中不使用特殊字符`*(star)`和`#(hash)`时，主题交换的行为就像直接交换一样。
- 当队列使用`#(hash)`绑定键绑定时，它将接收所有消息，而不管路由键是什么，就像fanout交换一样。

在日志系统中使用top。假设日志的routing key将包含两个单词:`<facility>.<severity>`。

```go
// emit_log_topic.go
package main

import (
        "context"
        "log"
        "os"
        "strings"
        "time"

        amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Panicf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs_topic", // name
                "topic",      // type
                true,         // durable
                false,        // auto-deleted
                false,        // internal
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        body := bodyFrom(os.Args)
        err = ch.PublishWithContext(ctx,
                "logs_topic",          // exchange
                severityFrom(os.Args), // routing key
                false, // mandatory
                false, // immediate
                amqp.Publishing{
                        ContentType: "text/plain",
                        Body:        []byte(body),
                })
        failOnError(err, "Failed to publish a message")

        log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
        var s string
        if (len(args) < 3) || os.Args[2] == "" {
                s = "hello"
        } else {
                s = strings.Join(args[2:], " ")
        }
        return s
}

func severityFrom(args []string) string {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "anonymous.info"
        } else {
                s = os.Args[1]
        }
        return s
}
```

```go
// receive_logs_topic.go
package main

import (
        "log"
        "os"

        amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Panicf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs_topic", // name
                "topic",      // type
                true,         // durable
                false,        // auto-deleted
                false,        // internal
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when unused
                true,  // exclusive
                false, // no-wait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

        if len(os.Args) < 2 {
                log.Printf("Usage: %s [binding_key]...", os.Args[0])
                os.Exit(0)
        }
        for _, s := range os.Args[1:] {
                log.Printf("Binding queue %s to exchange %s with routing key %s",
                        q.Name, "logs_topic", s)
                err = ch.QueueBind(
                        q.Name,       // queue name
                        s,            // routing key
                        "logs_topic", // exchange
                        false,
                        nil)
                failOnError(err, "Failed to bind a queue")
        }

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto ack
                false,  // exclusive
                false,  // no local
                false,  // no wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        var forever chan struct{}

        go func() {
                for d := range msgs {
                        log.Printf(" [x] %s", d.Body)
                }
        }()

        log.Printf(" [*] Waiting for logs. To exit press CTRL+C")
        <-forever
}
```

```shell

# go run 5_receive_logs_topic.go "kern.*" "*.critical"
2023/10/31 08:00:27 Binding queue amq.gen-O_37tbDOW3gbUyDRcZ8Bsw to exchange logs_topic with routing key kern.*
2023/10/31 08:00:27 Binding queue amq.gen-O_37tbDOW3gbUyDRcZ8Bsw to exchange logs_topic with routing key *.critical
2023/10/31 08:00:27  [*] Waiting for logs. To exit press CTRL+C
2023/10/31 08:00:39  [x] A critical kernel error
2023/10/31 08:40:48  [x] A critical kernel error
```

```shell
# go run 5_emit_log_topic.go kern.critical "A critical kernel error"

# go run 5_emit_log_topic.go kern. "A critical kernel error"
2023/10/31 08:40:48  [x] Sent A critical kernel error

# go run 5_emit_log_topic.go kern "A critical kernel error"
// routing-key 无法匹配，没有队列可以处理
2023/10/31 08:41:08  [x] Sent A critical kernel error
```

消费者通过`ExchangeDeclare`注册名为`logs_topic`类型为`topic`的exchange。声明一个匿名队列。并将匿名队列用命令行参数routing key `"kern.*" "*.critical"` 绑定到交换器`logs_topic`。

生产者生产的消息会带着routing-key来到exchange，如果routing-key匹配消费者指定的topic规则，则将消息交给队列处理，否则忽略。

### 构建RPC系统

构建一个RPC系统:一个客户端和一个可扩展的RPC服务器。由于我们没有任何值得分发的耗时任务，因此我们将创建一个返回斐波那契数的虚拟RPC服务。

一般来说，在RabbitMQ上做RPC是很容易的。客户机发送请求消息，服务器使用响应消息进行应答。为了接收响应，我们需要在请求中发送一个回调队列地址。我们可以使用默认队列。

消息属性中Correlation Id的作用：为每个客户端创建一个回调队列。在该队列中接收到响应后，不清楚该响应属于哪个请求。这就是使用correlation_id属性的时候。我们会为每个请求设置一个唯一的值。稍后，当我们在回调队列中接收到消息时，我们将查看此属性，并基于此将响应与请求进行匹配。如果我们看到一个未知的correlation_id值，我们可以安全地丢弃消息-它不属于我们的请求。您可能会问，为什么我们应该忽略回调队列中的未知消息，而不是失败并出现错误?这是由于服务器端存在race condition的可能性。虽然不太可能，但RPC服务器可能在向我们发送答案之后，但在为请求发送确认消息之前就会死亡。如果发生这种情况，重新启动的RPC服务器将再次处理请求。这就是为什么在客户机上我们必须优雅地处理重复响应的原因，理想情况下RPC应该是幂等的。

我们的RPC将这样工作:

当客户端启动时，它创建一个匿名独占回调队列。
对于RPC请求，客户端发送具有两个属性的消息:reply_to和correlation_id，前者设置为回调队列，后者设置为每个请求的唯一值。
请求被发送到rpc_queue队列。
RPC工作器(又名:服务器)正在该队列上等待请求。当出现请求时，它执行任务，并使用reply_to字段中的队列将带有结果的消息发送回客户机。
客户端等待回调队列上的数据。当出现消息时，它会检查correlation_id属性。如果它与请求中的值匹配，则将响应返回给应用程序。

```go
// rpc_server.go
package main

import (
        "context"
        "log"
        "strconv"
        "time"

        amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Panicf("%s: %s", msg, err)
        }
}

func fib(n int) int {
        if n == 0 {
                return 0
        } else if n == 1 {
                return 1
        } else {
                return fib(n-1) + fib(n-2)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        q, err := ch.QueueDeclare(
                "rpc_queue", // name
                false,       // durable
                false,       // delete when unused
                false,       // exclusive
                false,       // no-wait
                nil,         // arguments
        )
        failOnError(err, "Failed to declare a queue")

        err = ch.Qos(
                1,     // prefetch count
                0,     // prefetch size
                false, // global
        )
        failOnError(err, "Failed to set QoS")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                false,  // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        var forever chan struct{}

        go func() {
                ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
                defer cancel()
                for d := range msgs {
                        n, err := strconv.Atoi(string(d.Body))
                        failOnError(err, "Failed to convert body to integer")

                        log.Printf(" [.] fib(%d)", n)
                        response := fib(n)

                        err = ch.PublishWithContext(ctx,
                                "",        // exchange
                                d.ReplyTo, // routing key
                                false,     // mandatory
                                false,     // immediate
                                amqp.Publishing{
                                        ContentType:   "text/plain",
                                        CorrelationId: d.CorrelationId,
                                        Body:          []byte(strconv.Itoa(response)),
                                })
                        failOnError(err, "Failed to publish a message")

                        d.Ack(false)
                }
        }()

        log.Printf(" [*] Awaiting RPC requests")
        <-forever
}
```

```go
// rpc_client.go
package main

import (
        "context"
        "log"
        "math/rand"
        "os"
        "strconv"
        "strings"
        "time"

        amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Panicf("%s: %s", msg, err)
        }
}

func randomString(l int) string {
        bytes := make([]byte, l)
        for i := 0; i < l; i++ {
                bytes[i] = byte(randInt(65, 90))
        }
        return string(bytes)
}

func randInt(min int, max int) int {
        return min + rand.Intn(max-min)
}

func fibonacciRPC(n int) (res int, err error) {
        conn, err := amqp.Dial("amqp://admin:admin@host.docker.internal:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when unused
                true,  // exclusive
                false, // noWait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        corrId := randomString(32)

        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        err = ch.PublishWithContext(ctx,
                "",          // exchange
                "rpc_queue", // routing key
                false,       // mandatory
                false,       // immediate
                amqp.Publishing{
                        ContentType:   "text/plain",
                        CorrelationId: corrId,
                        ReplyTo:       q.Name,
                        Body:          []byte(strconv.Itoa(n)),
                })
        failOnError(err, "Failed to publish a message")

        for d := range msgs {
                if corrId == d.CorrelationId {
                        res, err = strconv.Atoi(string(d.Body))
                        failOnError(err, "Failed to convert body to integer")
                        break
                }
        }

        return
}

func main() {
        rand.Seed(time.Now().UTC().UnixNano())

        n := bodyFrom(os.Args)

        log.Printf(" [x] Requesting fib(%d)", n)
        res, err := fibonacciRPC(n)
        failOnError(err, "Failed to handle RPC request")

        log.Printf(" [.] Got %d", res)
}

func bodyFrom(args []string) int {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "30"
        } else {
                s = strings.Join(args[1:], " ")
        }
        n, err := strconv.Atoi(s)
        failOnError(err, "Failed to convert arg to integer")
        return n
}
```

```shell
# go run 5_rpc_server.go
2023/10/31 09:56:40  [*] Awaiting RPC requests
// 1. 声明一个队列"rpc_queue"作为请求队列，启动Consume消费队列rpc_queue
// 1. 由于队列没有数据就等待rpc_client的请求到来并将数据写入队列rpc_queue，此时启动rpc_client.go
2023/10/31 09:56:48  [.] fib(30)
// 3. 消费者从rpc_queue获取到请求数据
// 3. 请求数据经过很短的回调函数处理后，将response带着CorrelationId参数replyTo请求来的队列。逻辑再次进入rpc_client.go
```

```shell
# go run 5_rpc_client.go
2023/10/31 09:56:48  [x] Requesting fib(30)
// 2. 声明一个匿名队列作为响应队列，启动Consume消费匿名队列中rpc_server reply的数据
// 2. 响应队列还没有数据，就发布带有参数"CorrelationId=randomString"和"ReplyTo=匿名队列"的请求到rpc_server（也就是写入队列rpc_queue）
2023/10/31 09:56:48  [.] Got 832040
// 2. 发送请求后就进入迭代匿名队列的reply消息过程，等待服务器将reply消息写入匿名队列，此时逻辑进入rpc_server.go的回调函数处理阶段。
// 4. 当读取到匿名队列中数据时，判断收到的CorrelationId是否和请求中生成的randomString是一致的，一致则表示响应已返回，回调完成。
```

代码执行过程，如shell执行结果中1-2-3-4步骤所示。双方是同步通信，彼此分别要监听送达和响应队列的数据，并分别做出回调函数和后续业务逻辑的处理。

注意，请求队列名是确定的，但是响应队列应该是匿名的。这样可以避免两个不同的响应发往同一个响应队列。

另外，由于rpc_server的消费过程是并发的，以及使用了forever channel上锁。所以只要队列没有关闭，`for ... range ch.Consumes` 就能不断循环消费，这和go中无限循环Channel的道理是一致的。

## 其他

官方服务器文档 [Server Operator Documentation](https://www.rabbitmq.com/admin-guide.html)

包含了安装、工具、配置、认证、监控、集群、副本与高可用、分布式等方面

## 主要参考

[AMQP 介绍](https://www.cnblogs.com/wuyongyin/p/15033836.html)

[官方教程](https://www.rabbitmq.com/tutorials/tutorial-one-go.html)

[RabbitMQ的应用场景以及基本原理介绍](https://zhuanlan.zhihu.com/p/304756549)

[RabbitMQ 高级应用](https://www.cnblogs.com/lusaisai/p/13022090.html)
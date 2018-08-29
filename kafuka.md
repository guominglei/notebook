# kafuka

## 简介
Linkedin 开源的可持久化，分布式 ，基于发布/订阅 模式的消息系统。

### 性能优势：
    - TB级消息持久化时间复杂度O(1)。顺序写硬盘。说明：内存中到一定阀值才flush到硬盘中。所以持久化不是立即的。
    - 高吞吐量 单机单进程100K/s 消息传输
    - 服务器间可以水平分区。分布式消费。保证分区内的信息是有序传输的。
    - 支持实时、异步 两种消息处理方式。

### kafka 架构
    - Broker  ，kafka把服务器称为broker
    - Topic 消息类别（消息队列）。可以理解为订阅的主题。一个消息必须属于某个主题。一个主题里的消息物理上可以分开存储（patrition）。逻辑上可以保存在多个broker 中。主题发布前需要定义好主题的内容格式（json格式的 key 名称，value 类型）。不符合格式的数据是发布不了的。
    - Partition  物理存储单元（文件夹）。序号从0开始
    - Producer  发布消息到kafka broker
    - Consumer  消息消费者从kafka broker中获取消息。
    - Consumer Group  每个消费者属于一个特定的Consumer Group。
    - zookeeper   负责控制broker之间的partition 平衡。和consumer 间的消费平衡。

### kafka 消息删除策略：
    - 基于时间             
    - 基于partition物理空间大小。

    读写高效率原因：
        消息顺序写入硬盘保证 O(1) 持久化
        consumer Group 保存offset偏移量。方便读取查找。顺序消费。没有锁机制。

    Producer 路由机制    
        生产者可以定义一个分片机制（负载均衡机制）。把消息均匀的分配到不同的partition中。消息可以指定一个key值。

    Consumer Group 概念
        同一个消息，在一个消费组内只能消费一次。不同的业务，需要使用不同的消费组。每个消费组内维持本组内的消息偏移量。保证消息的顺序读取。
        消息组播 就是说多个消费 隶属于不同的消费组。
        消息的单播 就是说一个消费一个消费组。

### kafka消息分发类别
    - at most once  最多一次。可能丢失
    - at least one    可能多次，不会丢失
    - exactly once   肯定一次，不会丢失
    
    生产者在push 数据到broker时，会生成一个主键。
    如果网络有问题，本次push(commit)不成功那么可以根据这个主键重复push(commit)
    消费者从broker pull数据时，根据当前消费组里的offset 来顺序读取消息。
    消费者可以选择是否修改这个offset.修改动作就是commit 。如果本次消费没有修改offset 那么下次依旧会消费这个条数据。

### partition 分片备份机制。
    为了提高数据可靠性。多个partition可以组成一个备份组。一个备份组有一个主，多个从。
    一个消息先发送给备份组里的主partition。 当这个消息被主partition 当前同步列表里的从partittion都复制了。就认为消息提交成功。
    注意：
        组内从partition 不会一起同步数据。只要主partition接到消息时，正在和主partittion 进行消息同步操作的所有从partition 都同步了当前消息。那么当前消息才算提交 发布 成功。主从更新采用的是批量更新的形式。

#### partition平衡算法：
    前提partition 数量大于broker 数量
    1、所有broker(N个) ，待分配partition（M个） 排序
    2、第I个partition分配到 i mod N 余数序号的broker中
    3、第J个partition的第j个从分配 分配到 (i+j) mod N 的余数序号的broker中

    Kafka选举一个broker作为controller，这个controller通过watch Zookeeper检测所有的broker failure，并负责为所有受影响的parition选举leader，再将相应的leader调整命令发送至受影响的broker，
    zookeeper 动态记录每个leader partition 的ISR(in-sync replicas)队列。只有在这个ISR队列中的partition 才有资格进行leader 选举。

### consumer 消息平衡消费。
    某个固定的partition 只会被固定的某个consumer消费。为了避免通信开销。
    平衡工作或者分配工作通过zookeeper操作的。
#### 平衡步骤：
    - consumser 在 consumer group 中注册自己。
    - consumser 在 consumer group 给自己添加个观察事件 ，
        方便动态更新自己在 consumer group中的状态
    - broker 在 consumer group 中给自己添加一个观察事件，
        方便动态更新自己在 consumer group中的状态
    - 如果 consumer 对事件有过滤，也会在consumer group 中注册这个过滤条件
    - consumer 开始再平衡。注意每增加一个consumer 或则 broker 时候，都会引起再平衡
        平衡算法。就是平均算法。
        1、将topic 下的所有partition 排序 记做，Pt
        2、将consumer group 下的所有consumer 排序记做，Ct
        3、N = len(Pt)/len(Ct) 向上取整。
        4、将 i * N 到  (i+1) * N的 partition 分配给 Ct 集合中的第i个consumer
                        

#### 调用API处理消息
    用户也可以调用底层api 自己实现读取消息行为。（Low level comsumper）
    优势：
        自己定义读取行为例如。
        * 同一消息读取多次
        * 读取某个topic 的部分partition
        * 管理事务。
    劣势就是比较麻烦。
#### 步骤：
    - 找到一个可用的broker。并且找到涉及的的每个partition 的leader
    - 找出每个partition的follower
    - 定义好请求，描述如何获取数据。
    - 获取数据
    - 实时监控leader变化。做出相应的变化。

## 初体验

    下载地址：https://www.confluent.io/download-center/
    下载zip 文件然后解压。

### 文件结构
    * confluent-3.1.1/bin/        
        # Driver scripts for starting/stopping services
    * confluent-3.1.1/etc/        # Configuration files
    * confluent-3.1.1/share/java/ # Jars
### 命令
#### 1、启动zookeeper 进程
   
    ./bin/zookeeper-server-start ./etc/kafka/zookeeper.properties

#### 2、启动kafka进程
    ./bin/kafka-server-start ./etc/kafka/server.properties

#### 3、启动注册器
    ./bin/schema-registry-start ./etc/schema-registry/schema-registry.properties

#### 4、启动shell producter
    ./bin/kafka-avro-console-produr --broker-list localhost:9092 --topic test --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"f1","type":"string"}]}’
    
    随后输入 
    {‘f1’:’msgxx’}
    {‘f1’:’msggg’}
    crtl+c 关闭

#### 5、启动shell consumer
    ./bin/kafka-avro-console-consumer --topic test --zookeeper localhost:2181 --from-beginning
    消费端会看到
    {‘f1’:’msgxx’}
    {‘f1’:’msggg’} 
    crtl + 关闭

#### 6、定义一个topic
    ./bin/kafka-avro-console-producer --broker-list localhost:9092,localhost:9093 -topic r_topic1 --property value.schema='{"type":"record","name":"r_topic1","fields":[{"name":"f1","type":"string"}]}'

#### 7、查看分区情况
    ./bin/kafka-topics --describe --zookeeper localhost:2181
    ./bin/kafka-topics --describe --zookeeper localhost:2181 --topic g_topic

## 配置文件
### kafka  server 配置参数
    
    broker.id = 0   #集群中的唯一标示 
    port = 9092     #默认监控地址
    host.name = xxx # 服务名称 
    num.network.threads=2 # 服务器处理网络请求最大的线程数
    num.io.threads = 8 #服务器处理磁盘IO的线程数。
    background.threads= 4 # 后台进程数
    queued.max.requests = 500 # 网络io线程最大等待队列
    socket.send.buffer.bytes = 1024 # socket发送buffer 
    socket.receive.buffer.bytes = 1024 # socket接收buffer
    socket.request.max.bytes = 1024 # socket请求最大字节数
    # zookeeper
    enable.zookeeper =true # 允许注册到zookeeper 服务器
    zookeeper.connect =  localhost:2182/kafka #  zookeeper 服务器地址信息
    zookeeper.connection.timeout.ms = 60 # 连接zookeeper timeout时间
    zookeeper.session.timeout.ms = xx # zk server session 过期时间
    zookeeper.sync.time.ms = xx # zk follower 和zk leader 同步间隔
    # log 
    log.flush.interval.messages = 1000 # 内存日志刷新到硬盘策略。消息条数到1000 就写硬盘。
    log.flush.interval.ms = 10*1000 # 内存日志刷新到硬盘策略。间隔10秒写硬盘。 和消息条数两者任意个成立都会执行。
    log.flush.scheduler.interval.ms = 2*1000 # 定期检查日志是否需要写入磁盘
    log.cleaner.enable = true or false# 开启日志清理
    log.cleanup.policy = delete or compat# 日志清理策略。
    log.retention.hours  =168 # 日志保存168小时（7天）
    log.retention.bytes =  xx   # 日志总存储能力多少字节的
    log.index.size.max.bytes = xx # 日志索引文件大小
    log.index.interval.bytes = xx # 日志索引计算用到的缓冲区   
    log.dir = xx # 日志目录地址
    log.segment.bytes = xx # 单个日志文件大小
    log.roll.hours = 24 #几个小时对日志进行一个切分
    # topic
    num.partitions = 2 # 每个topic 分区个数
    auto.create.topics.enable=true # 是否允许自动创建topic
    default.replication.factor = 1 # 分区默认的备份数量
    message.max.bytes = xx # 消息体最大数据量
    # partition
    controller.socket.timeout.ms = xx # controller 和主从连接超时时间。
    controller.message.queue.size=10 # 主分区到broker消息队列 
    replica.lag.time.max.ms = xx # 从分区响应主分区最长时间。超过就把从分区从主分区的同步队列中删除。
    controlled.shutdown.enable = false # 是否允许控制器关闭分区。
    controlled.shutdown.max.retries = 3 # 控制器关闭分区尝试次数
    controlled.shutdown.retry.backoff.ms = xx # 尝试关闭时间间隔
    replica.socket.timeout.ms = xx # partition leader 和 replica 的timeout’
    replica.socket.receive.buffer.bytes = xx #
    replica.fetch.max.bytes = xx # 从分区一次最多从主分区获取多少数据。
    replica.fetch.min.bytes = xx # 从分区一次最少从主分区获取多少数据。
    replica.fetch.wait.max.ms = 500 # 分区同步最多等待时间
    num.replica.fetcher=1  # 同步进程数量
    replia.high.watermark.checkpoint.interval.ms = xx # 
    auto.leader.rebalance.enable = false # 是否开启broker平衡策略
    leader.imbalance.per.broker.percentage= 10 # broker平衡操作阀值
    leader.imbalance.check.interval.seconds = xx # broker 平衡检查周期
    offset.metadata.max.bytes = xx # 客户端保留的offset信息。

### 生产者配置文件

    client.id = xx  #  
    bootstrap.servers = xx # broker 节点信息 形如192.168.203.234:9092,xx
    request.required.acks = 0 # 0 不保证消息到达确认。只管发送。
                                                  1 等待leader 确认。
                                                  -1 等到leader确认。并进行主从间复制。最高级别
    request.timeout.ms = xx # 发布消息最大等待时间
    send.buffer.bytes = xx # 
    key.serializer.class = xx # 序列号工作类
    partitioner.class =  xx # 分区策略类
    compression.codec = none # 数据压缩类型
    compression.topics = null #  对指定的专题进行压缩
    message.send.max.retries = 3 # 发送消息失败重复次数
    retry.backoff.ms = 100 # 失败后等待多久再次尝试发送。
    topic.metadata.refresh.interval.ms = xx # 生产者定时更新topic元信息的时间间隔 ，若是设置为0，那么会在每个消息发送后都去更新数据
    queue.buffering.max.ms= xx # 消息发送最大延时
    queue.buffering.max.messages = xx # 异步模式下消息最大队列
    queue.enqueue.timeout.ms = -1 # 异步模式下进入队列等待时间 
    batch.num.messages = xx # 异步模式下批量发送信息的量。

### 消费者配置文件

    group.id = xx # 消费组id
    consumer.id = xx # 当前消费者id
    client.id  = xx # 
    #zookeeper 
    zookeeper.connect = 192.xx.xx.xx:2182 # zkserver 地址
    zookeeper.session.timeout.ms = xx # zk 心跳包
    zookeeper.connection.timeout.ms = xx # zk 连接超时
    zookeeper.sync.time.ms = xx # zk leader  和 follower 同步频率
    auto.offset.reset = largest or  smallest # offset重置
    socket.timeout.ms = xx # socket超时时间
    socket.receive.buffer.bytes= xx 
    fetch.message.max.bytes = xx 
    auto.commit.enable = True, # true时，Consumer会在消费消息后将offset同步到zookeeper，这样当Consumer失败后，新的consumer就能从zookeeper获取最新的offset 
    auto.commit.interval.ms = 60*1000  # 自动提交间隔
    queued.max.message.chunks = xx 
    rebalance.max.retries  = xx # 消费平衡重试次数
    rebalance.backoff.ms = xx # 重试间隔
    refresh.leader.backoff.ms = xx # 选举leader 间隔
    fetch.min.bytes = xx # 最小获取消息字节数
    fetch.wait.max.ms = xx # 当数据不满足最小字节数。等待时间
    consumer.timeout.ms = -1 # 指定时间内没有消息，抛出异常。 -1标示不受限制

## 集群

问题1：1个topic如何映射到不同的partition? 映射发生时机？

      通过partition 分配算法。映射发生在生产端。

问题2：topic的partition 是否可以分配到不同的broker中。
      
      topic的partition可以分配到不同的broker中。分配策略分片数量对broker数量取模。

问题3：不同的broker 如何做到均衡。 集群模式下客户端怎样连接broker。
    
    broker 的均衡有两种。
    一种是已有节点坏了。
        1、如果broker 配置项auto.leader.rebalance.enable =True 。那么zookeeper 会开始分片leader 选举（只有主节点在坏的broker）。然后开始从分片的分配。
        2、人工开启再平衡工作。
    另一种是新添加节点。人工开启再平衡。

问题4：集群模式下客户端怎样连接broker。
     
    读写都通过partition的leader分片。从分片不参与实际操作。
    集群模式下。生成者和消费者如何配置服务器地址？
    生成者，需要配所有的broker节点信息。kafka集群中的任何一个broker,都可以向producer提供
    metadata信息,这些metadata中包含"集群中存活的servers列表"/"partitions leader列表"等信息(请参看zookeeper中的节点信息). 当producer获取到metadata信息之后, producer将会和Topic下所有partition leader保持socket连接;消息由producer直接通过socket发送到broker,中间不会经过任何"路由层".事实上,消息被路由到哪个partition上,有producer客户端决定.比如可以采用"random""key-hash""轮询"等,如果一个topic中有多个partitions,那么在producer端实现"消息均衡分发"是必要的.在producer端的配置文件中,开发者可以指定partition路由的方式.
    消费者，需要配所有的zookeeper节点信息。消费者的offset发送给zookeeper。让zookeeper保存。

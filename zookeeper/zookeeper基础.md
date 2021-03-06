# zookeeper 
## 基础
    zookeeper 通过 数据结构(znode) + 原语（） + watch机制 来实现信息同步服务的。
    
### znode:
    是一个树状结构。类似于linux文件系统。
    znode 通过路径来引用。路径必须是绝度路径。 路径是唯一的。字符编码是unicode。除了系统保留的根节点。其他名称不限制

    znode 数据由三部分组成。
    1、stat 状态信息，描述znode版本信息，权限信息。
    2、data 与该znode关联的数据。（数据信息最大不能超过1M。通常以KB为单位。）
    3、children 该znode下的子节点。
    
    znode 每个节点，都有自己的访问控制列表(ACL)这个列表规定了用户的权限。限制了特定用户的操作。

#### znode 节点数据结构
    时间戳 Zxid  64位。 高32位记录leader id。 低32位是递增计数。 每次leader 的更改。时间戳都会变更。 
    
    cZxid 节点创建时间 
    mZxid 节点修改时间   
    pZxid 节点及其子节点最后一次修改时间 

    version 节点数据版本号
    cversion 子节点版本号
    aversion 节点的acl版本号

    ctime  创建时间
    mtime  修改时间
    ephemeralOwner    如果是临时节点，值为会话id。 如果是永久节点值为0
    dataLength  节点数据长度
    numChildren  子节点长度  指的是路径深度？还是所有子节点的数量？


### 节点类型
    znode 有两种节点类型。一种是临时。另一种是永久的。
    临时节点，存在于一个会话（session）中。会话结束就被删除自动。临时节点不允许创建子节点。
    永久节点 生命周期不依赖于会话。只有客户端执行了删除操作才删除。

    节点是有顺序的。每次创建节点，zookeeper总会在路径结尾处添加一个递增的计数。只个计数对于此节点的父节点是唯一的。
    
    节点总共有4类
    1、持久化节点
    2、顺序自动编号持久化节点
    3、临时节点
    4、顺序自动编号临时节点
    
    问题：
        顺序节点不判断名称吗？不怕重复？


### 节点基本操作
    create          创建
    delete          删除  必须有版本号 如果版本号不匹配，操作失败
    exists          判断节点是否存在并返回元数据
    getACL/setACL   获取 设置 znode 的ACL
    getChildren     获取所有子节点
    getData/setData 获取 设置 节点数据  设置是必须要有版本号 如果版本号不匹配，操作失败
    sync            客户端的znode视图和zookeeper同步
    以上操作是非阻塞的。


### watch 机制
    znode节点的创建，修改，删除。事件通过watch 通知给客户端。只会通知一次。即客户端收到watch事件后，进行处理。如果还想监听这个节点的特定Watch事件，必须进行再次watch注册。要不然下次节点变动，不会通知到这个客户端。
    注意：如果客户端注册了观察事件，在事件到来之前断线了。事件通知已经发送出去了。再次连接上后，已经发送的事件不会回放。

#### watch 类型
    1、监控节点数据变化
        getData, exists 这两个操作可以实现监控节点数据变化。
        getData 操作在被监控节点数据更新或者节点删除时候，发送通知。节点创建时不能发送通知
        exists  操作在被监控节点创建，删除 或者 数据更新时发送通知。
    2、监控节点下的子节点变化
        getChildren 操作在被监控节点的子节点创建 删除 或者被监控节点自己被删除时，发送通知。

#### Watch 处理事件类型
    1、连接状态事件。
    2、节点事件。 就是节点变更事件。 是一次性的。处理完本次事件后，需要重复注册。


## 安装
### 下载地址
    http://mirrors.shu.edu.cn/apache/zookeeper/
    py库：http://www.jb51.net/article/133173.htm
### doc 地址
    https://zookeeper.apache.org/doc/r3.4.11/

### 配置文件
    tickTime=2000           服务器客户端直接心跳包时间. 最小的session维持时间是2倍的心跳包时间。
    dataDir=/data/tmp/zookeeper     数据存放地点
    clientPort=2181         监听客户端连接端口
    initLimit=5             5 * 2000 或则 10 秒 从节点和主节点,最长心跳时间
    syncLimit=2             2 * 2000 从节点和主节点同步数据最长应答时间
    server.1=host:port1:port2   
    .1 服务器编号， host 服务域名或则iP 
    port1 leader选举接口， port2 服务器之间通信接口  
    服务器dataDir下需要有个myid的文件。对应.1 myid文件里的内容就是1

### 服务器启动
    1、解压文件后，bin目录添加的path中。
    2、创建配置文件
    3、zkServer.sh start xx.cfg     启动
    4、zkServer.sh status xx.cfg    查看状态
    5、zkServer.sh stop xx.cfg      删除

### 客户端基本使用
    启动命令客户端：
        zkCli.sh -server host:port
    查看节点
    ls / 查看根目录文件
    创建节点
    create /zk_test  haha  创建目录 /zk_test 并把数据haha存进去
    # 获取节点数据
    get /zk_test       获取节点数据
    # 更新节点
    set /zk_test rick  更新节点数据
    # 删除节点
    delete /zk_test    删除节点

## 应用场景
### 分布式锁服务
    原理是： 
        以节点id大小作为判断。最小的持有锁 
        在特定的节点下。创建顺序子节点。判断新节点id是否是子节点中最小的那个。 如果是最小的,即最先创建的,那本节点持有锁。如果不是不持有锁。

    问题
    1：羊群效应。
        如果大量客户端收到同一个事件通知。单实际上只有很少的一部分需要处理这个事件。只有一个客户端会成功获取到锁。但是zookeeper服务需要维护大量的观察事件。浪费了
        zookeeper的服务能力。
      方案：
        优化通知条件。zookeeper服务消息机制。
        只在最小子节点消失(删除）时，通知次小节点客户端。其他客户端不通知。
        疑问：客户端如何实现的？
    2：可恢复的异常
       不能处理因连接丢失而导致的create节点操作失败。
       可以根据节点名称来判断下。节点是否创建成功了。
    3：不可恢复的异常。
        临时节点过了会话期被删除了。只能有程序自己控制了。

### 共享锁
    实现原理
    1、 利用节点名称唯一性，来实现锁机制。
        想获取锁就创建。
        创建成功了就获取到锁。创建不成功。获取不到锁。处理完毕。删除节点。
    2、利用顺序节点。来实现。

### 数据发布与订阅
    数据发布与订阅即配置管理。将数据发布到zk节点上。供订阅者动态获取数据。实现配置信息的集中式管理和动态更新

### 统一命名服务
    利用zk节点路径唯一性。客户端根据指定的名字来获取资源服务地址。较为常见的是指定某个路径。
    路径下存储服务器列表。客户端根据路径获取相关列表信息。

### 发送通知/协调
    新的心跳检测机制。通过检测zk上的节点。来判断服务是否正常
    新的系统调度。通过修改zk节点上的数据。利用watch机制完成数据通知
    新的汇报模式。子任务创建临时节点。完成后临时节点删除。管理者实时知道子节点的情况。
    降低系统间的耦合度。

### 集群管理
    集群机器监控。 
    1、客户端对指定节点进行watch. 节点有变化，及时通知客户端。
    2、创建临时子节点。一旦客户端与服务器的会话期结束。该节点就删除。
    Master选举
    1、利用路径唯一性。都创建/master路径。但是只有一个能成功
    2、动态Master选举。利用顺序临时节点。
        允许路径创建成功。但是成为主节点的是，最先创建的节点。客户端需要判断

### 队列管理
    1、同步队列。只有队列中的所有成员聚齐了。这个队列才可用。
        创建指定节点。节点数据存储成员数量。每个成员对应指定节点下的子节点。
        a、查看是否有start节点。如果有说明已经完成同步。
        b、如果没有，查看子节点数量和之前设置的数量是否一致。如果一致，创建start节点。

    2、生成者消费者模型。按照FIFO方式进行工作
       利用顺序节点的性质。在指定节点下，生成时，创建顺序节点。消费是，读取节点顺序最小的。

## 权限管理 ACL
    访问控制即ACL。由客户端和服务器双方公共完成的。
    ACL数据由三部分组成 1、权限perms 2、验证模式scheme 3、具体内容expression:ids
    权限perms
        是一个int型数据 1-5 分别表示 setacl(最高权限), delete, create, write, read
        java API 有三种权限
        OPEN_ACL_UNSAFE  所有人开放
        CREATOR_ALL_ACL  创建者所有权限。
        READ_ACL_UNSAFE  只有只读权限
    验证模式scheme
        Digest  client由用户名和密码验证
        Host    client由主机名验证
        Ip      client由IP地址验证
        World   固定用户anyone, 为所有用户开放
    具体内容expression:ids
        当验证模式是 Digest     ids 为 用户名:密码
        当验证模式是 Host       ids 为 host
        当验证模式是 ip         ids 为 ip

## 原理
    zookeeper核心是原子广播机制。通过实现zab协议实现的。
    zab协议有两种模式1、恢复模式。2、广播模式
    恢复模式：
        当前leader失效。进入恢复模式。
        1、选举出新的leader。
        2、从节点和新leader进行数据的同步。
    广播模式:
        leader 和 follower 完成了数据的同步。进入广播状态。

    客户端可以连接到 leader follower 任意一个。
    如果连接的是follower,数据更新操作会被转到leader上。由leader发起决议。让follower进行表决。同意过半，视为通过。待服务器持久化后，响应客户端。数据操作成功。
    注意，
    只要服务器持久化成功。就返回。还是待符合半数以上持久化成功后，返回。需要确认。
    当客户端连接的那台zk服务器，收到了commit信息后。zk服务器向client发送操作成功。


    leader 已经commit 的数据，再此leader恢复后，依旧把commit信息发送到说有机器中。
    提议可以在没有发送时，leader 宕机了。那此leader 恢复后，未发出的提议。会被忽略。
    如何保证呢？
        选举leader规则是选举有提议编号(zxid编号最大)。新leader会把 提议编号进行修改
        前32位是leader编号。+1 后边的32位序列号为由0 开头。
        相当于，把当前未被commit的提议，内容不变，换了个zxid。从新进行预提交表决。表决通过后，再进行commit操作。
        从节点，只会接受，大于自己接受到的提议编号的提议。

    添加observer节点。观察节点不参与投票和leader选举。其作用就是，处理客户端的连接。提高了处理ZK系统整体能力。
    每增加一个从机，就需要多一份发送提议 + 一份投票 +一份comit 系统内部通信增多。
    observer可以获取leader的本地数据。 如果发生了数据不一致（新旧leader导致的），可以通过client sync 命令来实现数据的同步












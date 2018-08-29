# mongodb
## 数据备份
### 1、备份
    备份整个数据库
    a、mongodump -d 数据库名称 -c 集合名称 -o 存放地址（到目录）
    导出数据
    b、mongoexport -d 数据库名称 -c 集合名称 -o 导出的数据存放地址（到具体文件）。
    mongoexport -d uaas_init -c m_info_gathering -o /Users/mingleiguo/mongo-db/backup/in_out/m_info_gathering.json

### 2、恢复
    a、mongorestore -d 数据库名称 -c 集合名称 -dir 存放地址/*.bson（到目录）
    导出数据
    b、mongoimport -d 数据库名称 -c 集合名称 -o 导出的数据存放地址（到具体文件）。
    mongoimport -d mapi -c m_info_gathering /Users/mingleiguo/mongo-db/backup/in_out/m_info_gathering.json

说明：
    
    dump出来的数据格式是bson二进制文件，可读性差些，不同版本之间可能有兼容性问题。
    但是数据索引都有。
    export出来的数据是json/csv/tsv可读性好，不同版本之间通用性好。
    但是只有数据没有索引信息。

    mongodump 有一个配套的机制。来保证在dump数据期间，产生的新的操作记录完整的保存下来。
    这个机制就是oplog。oplog 只有在集群中才能使用。oplog 存放于local 库中oplog.rs。 oplog机制只在全库备份时可用。 实现原理就是固定大小的collection。
    记录在dump开始到dump结束时，这段时间内的操作记录。

## 安全机制
### mongodb 写操作执行过程：
    insert/update/remove/save  等操作更新集合中数据时候，先写内存journal 日志，再修改内存中的数据。 注意此时内存中的数据并没有持久化到磁盘上。每100ms journal日志会写磁盘。每60秒内存变更的数据会持久化到磁盘上。

    mongodb 写安全机制：写安全机制，由客户端设置。用于控制写入安全级别的机制。

    unacknowledged  非确认式写入。服务器收到写请求后，就返回。服务器到底执行成功与否不清楚。
    acknowledged确认式写入。服务器收到请求，并且服务器写入journal日志后注意未持久化到硬盘就返回。
    journaled 日志写入。journal 日志作用相当于redo日志。用于故障恢复和持久化。 2.0以上版本默认开启。journal文件位于journal目录中，只能以追加方式添加数据，文件名以j._开头。每100ms 或则30ms flush到磁盘上。写操作要等到操作记录存储到journal日志并且持久化到硬盘后才返回结果。这种结果可以有效保障服务器宕机。
    replica acknowledged  复制集确认式写入。写操作不仅得到主节点的写入确认，还需要得到从节点的写入确认。这里可以设置写入节点确认个数

    python 客户端pymongo  mongoengine  使用的时候是在connect 中传入参数
    w = 3 代表副本集中，2个从节点完成的写日志操作。 1个主节点完成写日志操作。
    w =0 代表非确认式写入。
    w =1 代表确认式写入
    wtimeout = 10  表示等待多久。 和w参数一起配合使用。 超时会报错。
    j =True , 标示使用 journal日志方式
    fsync =True ， 代表写入到磁盘。如果mongodb实例已经开启了journaling 那么这个操作和j  一样。这两种不能同时使用。版本2.2以上。fsync=True  和 j= True 作用相等。两者选一个。 也可以这样理解，fsync 已经废弃了。

## 视图VIEW
    view视图。 除了占用system.views 下的定义空间外，不占用其他存储空间。
    当用户查询的时候，根据视图定义先查询一遍。获取相关数据后再展示给前端。
    注意：
    1、只读，不能写
    2、不能使用map reduce,  全文索引。不能geonear 
    3、视图不能提高查询效率。查询效率决定于源集合内的索引情况。
    4、视图数据根据源数据变化而变化。

    使用场景：
    1、权限控制。 
    2、简化查询逻辑。

    db.createView(
         “view_name”,                            # 视图名称
         “source_collection_name”,       #  源集合名词
         {
              $match:{ key: value}           #   查询条件
          }
    )

## 副本集
    mongodb集群有下边三种形式：
    1、主从 。两个实例就行。一个运行master，一个运行slave模式。现在不推荐使用
    2、副本集。推荐使用
    3、分片。分片一般和副本集联合使用。

    副本集：
    一个主节点，多个从节点。主节点失效，从节点会选举一个作为主节点。如果所有的从节点都宕机了。那么主节点变为从节点。整个副本集停止服务。
    一个副本集可以最多支持12个成员，但是只有7个成员可以参与投票。
    如何进行读写分离呢？读写分离是做到客户端了。
    命令行下：读的话，直接连具体的从机器，进入到具体的数据库中，然后敲db.slaveOk() 就能读数据了注意，当前连接有效。

### Python 客户端，可以指定读的方式。目前有5种方式：
    primary
        主节点，默认模式，读操作只在主节点，如果主节点不可用，报错或者抛出异常。
        对应 mongoengine  ReadPreference.PRIMARY

    primaryPreferred
        首选主节点，大多情况下读操作在主节点，如果主节点不可用，如故障转移，读操作在从节点。
        对应 mongoengine  ReadPreference.PRIMARY_PREFERRED

    secondary
        从节点，读操作只在从节点， 如果从节点不可用，报错或者抛出异常。
        对应 mongoengine  ReadPreference.SECONDARY

    secondaryPreferred
        首选从节点，大多情况下读操作在从节点，特殊情况（如单主节点架构）读操作在主节点。
        对应 mongoengine  ReadPreference.SECONDARY_PREFERRED

    nearest
        最邻近节点，读操作在最邻近的成员，可能是主节点或者从节点，关于最邻近的成员请参考
        对应 mongoengine  ReadPreference.NEAREST
    
    说明：
        primary 节点写入数据，secondary 通过读取primary的oplog得到复制信息，开始复制数据并且将复制信息写入到自己的oplog中。secondary节点如果某个操作失败了。就停止再同步。
        同步过程中，secondary 自己的local库的oplog.rs集合找出最近的时间戳，检查Primary节点的local库里的oplog.rs集合找出大于secondary时间戳的记录。
        优先级为0的节点，不会成为主点。
        隐藏节点，可以进行主节点选择投票。但是不会提供服务。只供数据备份用。当数据量大的时候，新加节点数据恢复时间比较长。
        仲裁节点。只处理主节点选举。主节点数+从节点数+仲裁节点数 的和必须是奇数

### 具体例子
测试环境：
#### 1、准备配置文件。
    下边的是master配置， 拷贝slave.conf, slave2.conf修改dbpath,logpath, port
    dbpath=/Users/mingleiguo/mongo-db/RepSet/data/master
    logpath=/Users/mingleiguo/mongo-db/RepSet/logs/master.log
    bind_ip=10.200.27.185
    port=27021
    replSet=rick
    #auth=True
#### 2、运行
     分别运行 mongd  -f  xx.conf
#### 3、配置副本集
    $>mongo 10.200.27.185:27021
    $>rs.initiate()
    $>rs.add('10.200.27.185:27021’)
    $>rs.add('10.200.27.185:27022’)
    $>rs.add('10.200.27.185:27023’)
    $>rs.conf() # 查看副本集配置
    $>rs.status() # 查看集合状态

#### 4、写数据 通过rs.status 可以查看出那个实例是主节点。
    $>mongo 10.200.27.185:27021
    $>use tiku
    $>db.createCollection(‘user’)
    $>db.user.insert({})

#### 5、从节点上查看数据
    $>mongo 10.200.27.185:27022
    $>rs.slaveOk()  # 没有这个，从节点不能查看数据
    $>use tiku
    $>db.user.find()

    默认从节点只能从主节点同步数据。可以通过设置，是从节点之间同步数据，节省同步时间。
    conf =  rs.conf()
    conf.chainingAllowed=false
    rs.reconfig(conf) 

## 数据分片
    分片，只是针对集合来说，按某个映射键来把集合内的数据，分别存入不同的mongodb实例中。对前台数据操作没有啥影响。分片，就是为了增加整个系统的ram。提高数据的读写速度。
    分片集群，指的是，每个分片内，又由多个mongodb实例，主组一个副本集。增加了系统的稳定性。
    数据以chunk形式存储，chunk最大值为 64MB。
    mongos 节点，分片路由节点，客户端都连接到mongos节点。由于所有客户端的请求都会经过这类节点（mongos通过分片算法知道请求的key位于哪个分片中，然后mogos再和具体的分片建立连接，查询，返回结果到mongos,mongos最后再返回结果到客户端。），所以，生成环境下，mongos 都是多个节点一起工作（多个节点如何一起工作？前端加一个负载均衡的比如ha, lvs, dns诸如此类）。

### 配置节点
    mongod配置文件节点，存储集群配置文件信息。用于存储分片集群的元数据。
    
    这些元数据包含整个集群的数据集合data sets与后端shards的对应关系。
    query router使用这些元数据来将客户端请求定位到后端相应的shards。
    生产环境的分片集群正好有3个config servers。
    config servers里的数据非常重要，如果config servers全部挂掉，整个分片集群将不可用。
    在生产环境中，每个config server都必须放置到不同的服务器上，并且每个分片集群的config server不能共用，必须分开部署。
### 分片节点
    mongod分片节点。 具体存储数据

    分片算法 
        1、分段 分段又分，人为分段（tag），和系统自己分。
        2、hashed
     
    分片key值的选择。已有数据的分片，选择的键值必须是索引键。  
    1、单个字段
    2、多个字段组成。如果对已经存在的数据进行分片。这几个字段在集合中是联合索引。
    
    说明：
        key和映射算法固定后，并且也有数据了。更改映射规则可以吗？答案是：在不删除数据的前提下不可以，删除数据后可以更改。
        变通的方式，
        1、先把数据export出来，
        2、然后删除集合。
        3、创建新的分片集合，
        4、导入import数据。

        表数据大于256G的不能被分片。
### 实例
    操作步骤：
    1、开启服务器实例
    a、开启配置文件服务器
    mongod -f ShardRepSet/conf/config.conf -configsvr

    b、开启分片路由服务器。
    mongos --configdb config/10.200.27.185:27021 --port 27020      
    # 需要说明的是， config/10.200.27.185:27021  
    config 是配置文件复制集的名称, / 后边的ip:port 是配置服务器信息， 多个配置信息由逗号分隔。

    c、开启存储节点
    mongod -f ShardRepSet/conf/shard1.conf  --shardsvr
    mongod -f ShardRepSet/conf/shard2.conf  --shardsvr

    2、连接mongos 进行分片配置
    $>mongo  10.200.27.185:27020
    # 添加分片
    $>sh.addShard(“10.200.27.185:27022”)
    $>sh.addShard("10.200.27.185:27023’)
    # sh.addShard("spock/server-1:27017,server-2:27017,server-4:27017”) # 添加副本集合
    # 对具体库开启分片功能
    $>sh.enableSharding('tiku’)
    # 对具体集合配置分片键值和算法。
    $>sh.shardCollection(’tiku.info',{age:1}) # 默认算法
    $>sh.shardCollection(’tiku.info',{age:”hashed"}) # hashed
    # 进入具体库
    $> use tiku
    $>db.printShardingStatus() # 查看分片状态。
    # 添加信息，查看分片情况如何
    $>for (var i=1; i< 10000;i++){ db.info.insert({age:i,"name":"g"+i}); }
    $>sh.help()  # 查看关于分片的相关函数。
    $>sh.status() # 查看集群信息

## mongoengine 学习
### 1、切换/（多）数据库
    class User(Document):
          name = StringField()
          meta = {
               'db_alias': 'user-db'
               'collection': 'book_catalog_draft’,
       'strict': False
          }
    self.switch_db(‘user-db-bk’)
    self.save()

### 2、切换/（多）集合
    self.switch_collection(‘collection_bk’)
    self.save()

###3、切换库并且切换表
mongoengine 
    swith_db switch_collection不能完成即切库又切表的需求。所以需要另写一个方法来完成。

    class Info(Document):
        name = StringField(max_length=255,verbose_name=u'姓名')
        age = IntField(default=0)
        updated_at = DateTimeField(default=None)
        meta = {
            'collection': 'user_info_draft',
            'db_alias': "draft",
            'strict': False
        }
        def switch_db_collection(self, db_alias, collection_alias,
            keep_created=True):
            """
                即切换数据库，有切换集合
            """
            with switch_db(self.__class__, db_alias) as cls:
                db = cls._get_db()
                with switch_collection(
                    self.__class__, collection_alias) as coll:
                    collection = coll._get_collection()
            self._get_collection = lambda: collection
            self._get_db = lambda: db
            self._collection = collection
            self._created = True if not keep_created else self._created
            self.__objects = self._qs
            self.__objects._collection_obj = collection
            return self

        def sync_2(self):
            import copy
            try:
                self.updated_at = datetime.datetime.utcnow()
                print "save draft"
                print "draft db:{} \t collection:{}".format(
                        self._get_db().name, self._collection
                    )
                self.save()
                print "###########################################"
                self.switch_db_collection("online",'user_info')
                self.updated_at = datetime.datetime.utcnow()
                print "save online"
                print "online db:{} \t collection:{}".format(
                        self._get_db().name, self._collection.name
                    )
                self.save(force_insert=True)
                print "###########################################"
                self.switch_db_collection('draft','user_info_draft')
                print "after db:{} \t collection:{}".format(
                        self._get_db().name, self._collection.name
                    )
            except Exception as e:
                print traceback.format_exc()

### 4、普通连接方式
     connect(
         db='test’,
         username='user’,
         password='12345’,
         host='mongodb://admin:qwerty@localhost/production’
     )

### 5、副本集连接方式
     from mongoengine import connect
      connect('dbname', replicaset='rs-name’)
      # MongoDB URI-style connect
      connect(host='mongodb://localhost/dbname?replicaSet=rs-name’)

### 6、读写分离
    读写分离，写不用特殊设置，默认就是主节点，为啥，因为只有一个主节点。初始化连接的时候就是连接的主连接。
    a: 实例化连接的时候指定
     con = connect(
           "tiku”,
           host="10.200.27.185”,
           port=27021,
           replicaset="rick”,
           read_preference=ReadPreference.SECONDARY
      )
    b: 查询的时候更改
    item = cls.objects(
           read_preference=ReadPreference.SECONDARY,
           name=name).get()
    print item

### 7、安全写入。相当于写入确认吧。

## WiredTiger引擎事务
mongodb 事务不支持多文档之间的事务操作。单文档之间的事务，通过两次提交来完成的。具体如下：
mongodb  wiredTiger 引擎事务原理。wiredtriger 事务涉及到的相关技术

wirtedtriger 事务对象结构如下：
wt_transaction {
      transaction_id:     事务唯一性id，表示数据版本号
      snapshot_object:  当前事务开始或者操作时刻其他正在执行并且未提价的事务集合。用于事务隔离
      operation_array: 本次事务中已经执行的操作列表，用于事务回滚。
      redo_log_buf:    操作日志缓存区。用于事务提交后的持久化。
      state:  事务当前状态。
}

1、mvcc（多版本并发控制）
基于key/value 中value值的链表。这个链表单元中存储当前版本操作的事务ID和操作后修改的值。数据结构如下：
wt_mvcc {
     transaction_id： 本次修改事务的ID。
     value：本次修改后的值。
}
事务只能查看当前事务和本次事务之前已经提交的数据。 未提交、删除的数据不能访问。

2、snapshot（事务快照）
事务开始或者进行操作之间对整个wt引擎内部正在执行或者将要执行的事务进行一次快照。保存当时整个引擎所有事务的状态。
snapshot_object = {
     snap_min = T1,          # 最小事务ID
     snap_max = T5,         # 最大事务ID
     snap_array = {T1, T4, T5}    # 正在执行的事务序列。
}
以上边这个快照来看。新的事务T6， 只能查看小于T1事务的数据。假如，快照建立后，T1事务提交了。那么T6 也不能访问T1修改的数据。

全局事务管理器
wt_txn_gloabl = {
    current_id： 全局写事务ID，递增。当前事务ID就是更具这个值来设置的。
    oldest_id：系统中最早产生，并且还在执行的写事务ID。
    transaction_array: 系统事务对象数组。保存系统中所有事务对象。
    scan_count: 正在扫描transaction_array数组的线程事务数，用于建立snapshot过程的无锁并发。
}

创建快照时候需要扫描全局事务管理器中的transaction_array， 由于创建快照是个频繁的工作。所以并没有使用锁机制。而是使用了变量scan_count。当scan_count变量为小于等于0的时候才允许进行transaction_array 扫描工作。扫描工作用来确认oldest_id 的值。确认过后，scan_count 变量设置为-1。供其他事务进行扫描。

3、 redo log（重做日志） 
操作日志是基于key/value形如：
{
     Operation = Insert
     Key = name,
     value= ‘rick’,
}

问题：为何没有表名称？
系统将操作日志写入到事务的redo_log_buf中。事务提交后，redo_log_buf 信息写入到系统的redo_log_file，定期把redo_log_file持久化到磁盘中。数据恢复时，可以根据提供的检查点的位置，执行redo_log中的动作即可。

WT引擎定义了一个LSN(Log Sequence Number)日志序列号。来管理日志。

wt-lsn = {
     file: 那个日志文件中。
     offset: 文件中的偏移量。
}

日志数据格式
一个事务对应WT日志的一个logrec对象。事务中的每次操作记录成一个logop对象。一个logrec对象对应多个logop对象。
logrec对象中包涵了事务ID，更改对象id。 然后就是一连串的logop对象

日志分为4类
1、建立监测点（ logrec_checkpoint）的操作日志
2、普通事务操作日志(logrec_commit) 根据 key/value操作方式分为
    a、log_put  。增加或则修改key value 操作
    b、log_remove 单个KEY删除操作
    c、范围删除日志

3、btree page 同步刷盘操作日志。 logger_file_sync
4、引擎外部使用的日志 logrec_message

多事务并发下。系统redo_log_file 如何工作的?
wt_log = {
     
     active_slot:  准备就绪并且可以合并loggec 的slot_buffer对象。
     slot_pool: 所有slot_buffer 对象数组。包括正在合并，准备合并，闲置的slot_buffer

}
slot_buffer结构
wt_log_slot = {
     state: 当前slot状态 ,ready, done, written, free
     buf : 缓存合并logrec的临时缓冲区
    group_size: 需啊哟提交的数据长度。
    slot_start_offset: Log file偏移量
}
slot_buffer 是有大小限制的。事务日志按根据先后顺序和大小来填充slot_buffer大小。如果数据大小小于buffer长度，但是大于当前剩余长度，那么等待新的slot_buffer对象。如果数据大小大于buffer大小，直接写磁盘文件。并且slot_buffer长度加大一倍。

事务过程：

1、准备阶段：
创建一个事务对象，并把这个对象加入到全局事务管理器。通过事务配置信息确认事务的隔离级别和redo log 的刷盘方式。并将事务状态设为执行状态。
最后判断隔离级别。如果隔离级别是snapshot 那么本次事务开始前创建一个snapshot快照。
2、事务执行：
主要操作事务对象中的operation_array， redo_log_buf。
a、创建一个MVCC list 中的值单元对象。
b、验证事务ID，更改事务状态为HAS_TXN_ID。
c、将本次事务ID更新到A步骤中的mvcc list单元里mvcc 版本号。
d、创建一个operation对象。并将这个对象的值指向步骤A创建的mvcc list 单元，并且将这个operation对象加入到operation_array中。
e、将步骤a 中创建的对象添加到mvcc list  表头。
f、写一条redo log到本次事务对象的redo_log_buf中。

3、事务提交 ：
a、将事务对象的redo_log_buf中的数据写入到 redo_log_file中去。并持久化到磁盘上。
b、清除事务对象的snapshot对象。并将事务对象中事务状态设置为WT_TNX_NONE。保证其他事务在创建事务snapshot时本次事务状态为已提交。可使用

4、事务回滚：
a、遍历整个operation_array，对每个数组单元对应的updated事务状态进行更改，设置为WT_TXN_ABORTED。标示改修改回滚了。其他读事务在进行MVCC读操作的时候，不再读取整个值。整个过程无锁。

事务的隔离级别：

read_uncommited  未提交读
未提交读指的是读取了还为commit的数据。wt  引擎将事务丢向中的snop_objects.snap_array设置为空。在读取mvcc list中的版本值时总读第一个 

read_commited 提交读
在本次事务执行过程中，在当前事务读取数据前，其他事务提交了对要读取数据的修改。那么这个修改可以在本次事务中生效。

snapshot_lsolation 快照隔离
在本次事务执行前，对系统进行快照。在本次事务执行过程中，对数据的访问严格依赖这个快照。事务开始前的数据可以使用，事务执行过程中commit的修改，对本次事务无效。

有说是，事务修改的数据，只在mvcc 列表中，实际的文档不修改。这个问题需要确认下。？

## 用户权限管理

    每个库里边都有一个Users 集合。admin 库里的Users配置 优先级大于 一般库里的Users配置
    用户有两个层级管理。一种是在admin库下边建立的用户，这时候use admin 后，db.getUsers() 。如果权限够，可以到具体某个库下边建立用户。另一种，就是具体某个库下边建立用户。在具体库里边建立用户的好处是，mongo -u rick -p 123456 库名  可以认证成功。否则还需要切换一次数据库的操作。

    假如同一个库uaas 。
    在admin 库下建立了一个用户 us_r 的用户可以对uaas进行读操作。 
    在uaas 库下建立了一个用户 g_r 的用户可以对uaas进行读操作。

    在uaas 库下，db.getUsers() 只显示g_r
    在admin库下，db.getUsers() 只显示us_r

    进入库的区别
    $mongo -u g_r -p xx uaas 
    $ db
    uaas

    $mongo -u us_r -p xx admin
    $ db
    admin
    $ use uaas

    如果想要使用身份认证，需要在配置文件中添加auth=True

### 具体命令例子：
    前提：use admin
    1、创建用户
    db.createUser({user:”rick”, pwd:’123456’, roles:[{role:”read”, db:”uaas"}]})

    2、删除用户
    db.removeUser(“rick”) # 参数是用户名

    3、用户列表，
    说明：只显示当前库下的用户，admin 库里设置的用户不显示。
    db.getUsers()

    4、用户认证
    方法1
    $ mongo
    $>use admin
    $>db.auth(‘rick’,’123456’)
    1 成功， 0 失败
    $> use db_name  # 切换到具体数据库开始自己的操作
    $
    方法2
    $ mongo -u rick -p 123456 admin   # admin是库名称

    5、更新数据
    db.updateUser({’rick’, {roles:[{role:’dbAdmin’, db:’uaas'}]}}) # 需要说明的是，roles里边要写全，现有的权限也需要加上，这个操作是覆盖操作。如果不写以前的就只有当前的

## 性能优化
    1、为data，journal日志，log日志分别使用单独的物理卷。 这三个都需要写磁盘，不同的物理卷，可以提高写的速度。提高性能。
    2、WiredTiger引擎下。使用XFS文件系统。因为Ext 系统内部的journal 和WireTiger 有冲突。高并发不理想。
    3、WiredTiger引擎下，默认60秒做一次checkpoint。即对内存中的改动的数据进行写磁盘操作。如果内存中的数据过大（大于128G）那么整个checkpoint时间会过长。 checkpoint 期间数据写入性能有问题。目前建议不能超过64G。
    4、关闭transparent huge pages。 
    mongodb单个记录占内存空间都比较小。所以没有必要开启内存大页。
    5、增加系统文件描述符的限制，因为mongodb对每个数据库文件和每个客户端连接都需要一个文件描述符来标示。
    6、禁止使用NUMA （非一致存储访问结构，多个CPU使用组合起来成为一个模块，多个模块组合起来就是NUMA。各个模块内不使用自己的内存。模块之间的内存数据通过连接模块来进行读取需要等待。通过连接模块读取数据比读取模块内的内存慢3倍。）。因为使用的了会出现性能问题。
    7、更改vm.swappiness 的值。 合理进行内存和硬盘之间数据的进出。0 为尽力不进行swap 都在内存中。100就是尽力使用硬盘不使用内存。默认是60。
    8、字段名称要简略。这样可以少占内存。提高效率。
    9、使用TTL来自动删除过期数据减少内存使用。
    10、合理设置oplog日志大小。避免日志没被写进去。或则日志量过大。

## map-reduce
### 命令总览
    db.runCommand({
         mapreduce: collection_name,          # 对那个集合进行操作
         map: map_function,                         # Map 映射函数
         reduce: reduce_function,                 #  reduce 统计函数
         query: query_params,                      # 过滤条件
         sort: sort_params                             # 排序条件
         limit: limit_num,                                # 限制数量
         out:out_collection_name,                 # 把统计结果存放于那个集合中去
         keeptemp: true|false,                       # 是否保留临时集合
         finalize: finalize_funciton,                 # 最终处理函数（对reduce返回结果进行最终整理后存入结果集合中去）
         scope: params_object,                     # 向 map, reduce, finalize 导入外部变量
         verbose:true,                                    # 显示详细时间统计信息。
    })     

    执行流程：
    1、对指定集合进行查询，找出符合筛选条件的文档。
    2、对第一步结果进行map方法采集
    3、对第二步结果进行reduce方法汇总。
    4、对第三步结果进行finalize方法处理。
    5、最终结果输出到指定的地方。终端？临时集合？固定集合
    6、断开连接，删除执行过程中的临时数据。

### map-reduce 实例
    db.orders.mapReduce(
         function(){emit(this.location, 1)}, # map
         function(key, values){return Array.sum(values)}, # reduce
         {
              query:{status:”1”}, # 查询
              out:’order_totals’,  # 结果存放处
          }  
    )

### map-reduce ,group 的区别？
    db.orders.group({
         key: {“localtion”:true},
         initial: {“names”:[]},
         reduce: function R(val, out){
                       var isExist=false;
                      for(var i=0;i<out.names.length;i++){
                             var cur = out.names[i];          
                             if (cur==val.name){
                                   isExists=True;
                                   break;
                                }
                        }
                       if(!isExist){
                             out.names.push(val.name)
                           }
              },
         finalize: function Finalize(out){ return out;}
    })

小提示：
1、使用Group或MapReduce时，如果一个分类只有一个元素，那么Reduce函数将不会执行，但Finalize函数还是会执行的。这时你要在Finalize函数中考虑一个元素与多个元素返回结果的一致性（比如，你把问题二中插入一个grade=3的数据看看，执行返回的grade=3时还有names集合吗？）。

## 查看性能
### 1、explain() 查询语句执行分析器
     db.user.info.find({name: ‘rick’,}).hint({name:1}).explain()
### 2、慢查询日志
     慢查询日志是一个固定大小的集合。如果要加大集合大小。需要先把现有的集合删除了。然后再创建一个新的集合。具体步骤如下：
     关闭慢查询：
     db.setProfilingLevel(0)
     删除老的集合
     db.system.profile.drop()
     创建新的集合# 大小是10M 单位是字节
     db.createCollection(’system.profile’, {capped:true, size:10*1024*1024}) 
     重新开启
      db.setProfilingLevel(1)

     开启慢查询日志：
     db.setProfillingLevel(2) 
     # 0 不开启
     # 1 开启，默认是执行时间大于100ms 的才记录。db.setProfillingLevel(1，500)  执行时间大于500毫秒的记录到慢查询日志中 
     # 2 所以命令都记录。

     需要注意的是：
     上边的命令只对单个库有效。
     要想使所有的库都生效，需要对每个库都运行上述命令。
     或则在启动实例的时候使用参数 - - profile=1  - - slowms= 600。 
     或则在配置文件中写入 profile=1， slowms= 600

     查看慢查询配置
     db.getProfillingStatus()

     查询慢查询日志信息：
     db.system.profile.find({mills:{$gt:600}})  # 执行时间大于600ms的

     每条信息结果如下：
      ts                开始执行时间
      reslen          返回结果集的大小
      scanned      查询扫描的记录数
      nreturned    本次查询实际返回的结果集
      responseLength   返回数据大小
      mills               命令消耗时间
      op                  操作类型 query, insert, update, remove, getmore, command
      ns                  操作的集合名称
      lockStats       锁信息。R全局读锁， W全局写锁，r特定库读锁, w特定库写锁

      type：  
     COLLSCAN #全表扫描
     IXSCAN #索引扫描
     FETCH #根据索引去检索指定document
     SHARD_MERGE #将各个分片返回数据进行merge
     SORT #表明在内存中进行了排序（与老版本的scanAndOrder:true一致）
     LIMIT #使用limit限制返回数
     SKIP #使用skip进行跳过
     IDHACK #针对_id进行查询
     SHARDING_FILTER #通过mongos对分片数据进行查询
     COUNT #利用db.coll.explain().count()之类进行count运算     
     COUNTSCAN #count不使用Index进行count时的stage返回
     COUNT_SCAN #count使用了Index进行count时的stage返回
     SUBPLA #未使用到索引的$or查询的stage返回
     TEXT #使用全文索引进行查询时候的stage返回
     PROJECTION #限定返回字段时候stage的返回 
       
### 3、mongostat 工具
     mongostat --host 10.200.27.185 --port 27021
     定期打印出如下的信息
 insert query update delete getmore command dirty used flushes vsize   res qrw arw net_in net_out conn  set repl                time
    *0    *0     *0     *0       0     4|0  0.0% 0.0%       0 3.17G 32.0M 0|0 0|0   485b   44.3k    5 rick  PRI Jul  4 11:00:42.212

### 4、db.stats()
    查看当前数据库的信息比如 记录数，数据库大小，索引数量等等
    {
         "db" : "tiku",
         "collections" : 1,
         "views" : 0,
         "objects" : 3,
         "avgObjSize" : 48,
         "dataSize" : 144,
         "storageSize" : 36864,
         "numExtents" : 0,
         "indexes" : 1,
         "indexSize" : 36864,
         "ok" : 1
    }

### 5、db.serverStatus()
    {
         "host" : "bogon:27021",
         "version" : "3.4.5",
         "process" : "mongod",
         "pid" : NumberLong(12936),
         "uptime" : 613,
         "uptimeMillis" : NumberLong(612105),
         "uptimeEstimate" : NumberLong(612),
         "localTime" : ISODate("2017-07-04T03:09:06.020Z"),
         "asserts" : {
              "regular" : 0,
              "warning" : 0,
              "msg" : 0,
              "user" : 6,
              "rollovers" : 0
         },
         "connections" : {
              "current" : 5,
              "available" : 199,
              "totalCreated" : 11
         },
         "extra_info" : {
              "note" : "fields vary by platform",
              "page_faults" : 3877
         },
         "globalLock" : {
              "totalTime" : NumberLong(612045000),
              "currentQueue" : {
                   "total" : 0,
                   "readers" : 0,
                   "writers" : 0
              },
              "activeClients" : {
                   "total" : 23,
                   "readers" : 0,
                   "writers" : 0
              }
         },
         "locks" : {
              "Global" : {
                   "acquireCount" : {
                        "r" : NumberLong(4692),
                        "w" : NumberLong(139),
                        "R" : NumberLong(1),
                        "W" : NumberLong(4)
                   }
              },
              "Database" : {
                   "acquireCount" : {
                        "r" : NumberLong(2573),
                        "w" : NumberLong(62),
                        "R" : NumberLong(5),
                        "W" : NumberLong(18)
                   }
              },
              "Collection" : {
                   "acquireCount" : {
                        "r" : NumberLong(1271)
                   }
              },
              "Metadata" : {
                   "acquireCount" : {
                        "w" : NumberLong(61)
                   }
              },
              "oplog" : {
                   "acquireCount" : {
                        "r" : NumberLong(1299),
                        "w" : NumberLong(62)
                   }
              }
         },
         "network" : {
              "bytesIn" : NumberLong(382625),
              "bytesOut" : NumberLong(686986),
              "physicalBytesIn" : NumberLong(382625),
              "physicalBytesOut" : NumberLong(686986),
              "numRequests" : NumberLong(2364)
         },
         "opLatencies" : {
              "reads" : {
                   "latency" : NumberLong(588380951),
                   "ops" : NumberLong(180)
              },
              "writes" : {
                   "latency" : NumberLong(0),
                   "ops" : NumberLong(0)
              },
              "commands" : {
                   "latency" : NumberLong(68801),
                   "ops" : NumberLong(1001)
              }
         },
         "opcounters" : {
              "insert" : 0,
              "query" : 3,
              "update" : 0,
              "delete" : 0,
              "getmore" : 179,
              "command" : 1002
         },
         "opcountersRepl" : {
              "insert" : 0,
              "query" : 0,
              "update" : 0,
              "delete" : 0,
              "getmore" : 0,
              "command" : 0
         },
         "repl" : {
              "hosts" : [
                   "10.200.27.185:27021",
                   "10.200.27.185:27022",
                   "10.200.27.185:27023"
              ],
              "arbiters" : [
                   "10.200.27.185:27024"
              ],
              "setName" : "rick",
              "setVersion" : 7,
              "ismaster" : true,
              "secondary" : false,
              "primary" : "10.200.27.185:27021",
              "me" : "10.200.27.185:27021",
              "electionId" : ObjectId("7fffffff0000000000000008"),
              "lastWrite" : {
                   "opTime" : {
                        "ts" : Timestamp(1499137738, 1),
                        "t" : NumberLong(8)
                   },
                   "lastWriteDate" : ISODate("2017-07-04T03:08:58Z")
              },
              "rbid" : 1280618692
         },
         "storageEngine" : {
              "name" : "wiredTiger",
              "supportsCommittedReads" : true,
              "readOnly" : false,
              "persistent" : true
         },
         "wiredTiger" : {
              "uri" : "statistics:",
              "LSM" : {
                   "application work units currently queued" : 0,
                   "merge work units currently queued" : 0,
                   "rows merged in an LSM tree" : 0,
                   "sleep for LSM checkpoint throttle" : 0,
                   "sleep for LSM merge throttle" : 0,
                   "switch work units currently queued" : 0,
                   "tree maintenance operations discarded" : 0,
                   "tree maintenance operations executed" : 0,
                   "tree maintenance operations scheduled" : 0,
                   "tree queue hit maximum" : 0
              },
              "async" : {
                   "current work queue length" : 0,
                   "maximum work queue length" : 0,
                   "number of allocation state races" : 0,
                   "number of flush calls" : 0,
                   "number of operation slots viewed for allocation" : 0,
                   "number of times operation allocation failed" : 0,
                   "number of times worker found no work" : 0,
                   "total allocations" : 0,
                   "total compact calls" : 0,
                   "total insert calls" : 0,
                   "total remove calls" : 0,
                   "total search calls" : 0,
                   "total update calls" : 0
              },
              "block-manager" : {
                   "blocks pre-loaded" : 23,
                   "blocks read" : 75,
                   "blocks written" : 135,
                   "bytes read" : 335872,
                   "bytes written" : 839680,
                   "bytes written for checkpoint" : 839680,
                   "mapped blocks read" : 0,
                   "mapped bytes read" : 0
              },
              "cache" : {
                   "application threads page read from disk to cache count" : 14,
                   "application threads page read from disk to cache time (usecs)" : 89626,
                   "application threads page write from cache to disk count" : 0,
                   "application threads page write from cache to disk time (usecs)" : 0,
                   "bytes belonging to page images in the cache" : 90848,
                   "bytes currently in the cache" : 157312,
                   "bytes not belonging to page images in the cache" : 66464,
                   "bytes read into cache" : 84119,
                   "bytes written from cache" : 558149,
                   "checkpoint blocked page eviction" : 0,
                   "eviction calls to get a page" : 40,
                   "eviction calls to get a page found queue empty" : 40,
                   "eviction calls to get a page found queue empty after locking" : 0,
                   "eviction currently operating in aggressive mode" : 0,
                   "eviction empty score" : 0,
                   "eviction server candidate queue empty when topping up" : 0,
                   "eviction server candidate queue not empty when topping up" : 0,
                   "eviction server evicting pages" : 0,
                   "eviction server slept, because we did not make progress with eviction" : 0,
                   "eviction server unable to reach eviction goal" : 0,
                   "eviction state" : 16,
                   "eviction walks abandoned" : 0,
                   "eviction worker thread active" : 0,
                   "eviction worker thread created" : 0,
                   "eviction worker thread evicting pages" : 0,
                   "eviction worker thread removed" : 0,
                   "eviction worker thread stable number" : 0,
                   "failed eviction of pages that exceeded the in-memory maximum" : 0,
                   "files with active eviction walks" : 0,
                   "files with new eviction walks started" : 0,
                   "force re-tuning of eviction workers once in a while" : 0,
                   "hazard pointer blocked page eviction" : 0,
                   "hazard pointer check calls" : 0,
                   "hazard pointer check entries walked" : 0,
                   "hazard pointer maximum array length" : 0,
                   "in-memory page passed criteria to be split" : 0,
                   "in-memory page splits" : 0,
                   "internal pages evicted" : 0,
                   "internal pages split during eviction" : 0,
                   "leaf pages split during eviction" : 0,
                   "lookaside table insert calls" : 0,
                   "lookaside table remove calls" : 0,
                   "maximum bytes configured" : 3758096384,
                   "maximum page size at eviction" : 0,
                   "modified pages evicted" : 0,
                   "modified pages evicted by application threads" : 0,
                   "overflow pages read into cache" : 0,
                   "overflow values cached in memory" : 0,
                   "page split during eviction deepened the tree" : 0,
                   "page written requiring lookaside records" : 0,
                   "pages currently held in the cache" : 30,
                   "pages evicted because they exceeded the in-memory maximum" : 0,
                   "pages evicted because they had chains of deleted items" : 0,
                   "pages evicted by application threads" : 0,
                   "pages queued for eviction" : 0,
                   "pages queued for urgent eviction" : 0,
                   "pages queued for urgent eviction during walk" : 0,
                   "pages read into cache" : 29,
                   "pages read into cache requiring lookaside entries" : 0,
                   "pages requested from the cache" : 4062,
                   "pages seen by eviction walk" : 0,
                   "pages selected for eviction unable to be evicted" : 0,
                   "pages walked for eviction" : 0,
                   "pages written from cache" : 67,
                   "pages written requiring in-memory restoration" : 0,
                   "percentage overhead" : 8,
                   "tracked bytes belonging to internal pages in the cache" : 20854,
                   "tracked bytes belonging to leaf pages in the cache" : 136458,
                   "tracked dirty bytes in the cache" : 79626,
                   "tracked dirty pages in the cache" : 2,
                   "unmodified pages evicted" : 0
              },
              "connection" : {
                   "auto adjusting condition resets" : 88,
                   "auto adjusting condition wait calls" : 3840,
                   "files currently open" : 18,
                   "memory allocations" : 43782,
                   "memory frees" : 42449,
                   "memory re-allocations" : 10773,
                   "pthread mutex condition wait calls" : 9903,
                   "pthread mutex shared lock read-lock calls" : 6191,
                   "pthread mutex shared lock write-lock calls" : 803,
                   "total fsync I/Os" : 155,
                   "total read I/Os" : 737,
                   "total write I/Os" : 238
              },
              "cursor" : {
                   "cursor create calls" : 74,
                   "cursor insert calls" : 106,
                   "cursor next calls" : 562,
                   "cursor prev calls" : 15,
                   "cursor remove calls" : 1,
                   "cursor reset calls" : 4028,
                   "cursor restarted searches" : 0,
                   "cursor search calls" : 4559,
                   "cursor search near calls" : 9,
                   "cursor update calls" : 0,
                   "truncate calls" : 0
              },
              "data-handle" : {
                   "connection data handles currently active" : 15,
                   "connection sweep candidate became referenced" : 0,
                   "connection sweep dhandles closed" : 0,
                   "connection sweep dhandles removed from hash list" : 23,
                   "connection sweep time-of-death sets" : 23,
                   "connection sweeps" : 61,
                   "session dhandles swept" : 0,
                   "session sweep attempts" : 31
              },
              "lock" : {
                   "checkpoint lock acquisitions" : 11,
                   "checkpoint lock application thread wait time (usecs)" : 0,
                   "checkpoint lock internal thread wait time (usecs)" : 8,
                   "handle-list lock eviction thread wait time (usecs)" : 2201,
                   "metadata lock acquisitions" : 11,
                   "metadata lock application thread wait time (usecs)" : 0,
                   "metadata lock internal thread wait time (usecs)" : 17,
                   "schema lock acquisitions" : 27,
                   "schema lock application thread wait time (usecs)" : 4,
                   "schema lock internal thread wait time (usecs)" : 1,
                   "table lock acquisitions" : 0,
                   "table lock application thread time waiting for the table lock (usecs)" : 0,
                   "table lock internal thread time waiting for the table lock (usecs)" : 0
              },
              "log" : {
                   "busy returns attempting to switch slots" : 0,
                   "consolidated slot closures" : 88,
                   "consolidated slot join active slot closed" : 0,
                   "consolidated slot join races" : 0,
                   "consolidated slot join transitions" : 88,
                   "consolidated slot joins" : 97,
                   "consolidated slot transitions unable to find free slot" : 0,
                   "consolidated slot unbuffered writes" : 0,
                   "log bytes of payload data" : 21013,
                   "log bytes written" : 26240,
                   "log files manually zero-filled" : 0,
                   "log flush operations" : 5931,
                   "log force write operations" : 6620,
                   "log force write operations skipped" : 6545,
                   "log records compressed" : 13,
                   "log records not compressed" : 59,
                   "log records too small to compress" : 25,
                   "log release advances write LSN" : 13,
                   "log scan operations" : 5,
                   "log scan records requiring two reads" : 0,
                   "log server thread advances write LSN" : 75,
                   "log server thread write LSN walk skipped" : 1529,
                   "log sync operations" : 88,
                   "log sync time duration (usecs)" : 837044,
                   "log sync_dir operations" : 1,
                   "log sync_dir time duration (usecs)" : 30282,
                   "log write operations" : 97,
                   "logging bytes consolidated" : 25856,
                   "maximum log file size" : 104857600,
                   "number of pre-allocated log files to create" : 2,
                   "pre-allocated log files not ready and missed" : 1,
                   "pre-allocated log files prepared" : 2,
                   "pre-allocated log files used" : 0,
                   "records processed by log scan" : 10,
                   "total in-memory size of compressed records" : 25074,
                   "total log buffer size" : 33554432,
                   "total size of compressed records" : 11977,
                   "written slots coalesced" : 0,
                   "yields waiting for previous log file close" : 0
              },
              "reconciliation" : {
                   "fast-path pages deleted" : 0,
                   "page reconciliation calls" : 67,
                   "page reconciliation calls for eviction" : 0,
                   "pages deleted" : 0,
                   "split bytes currently awaiting free" : 0,
                   "split objects currently awaiting free" : 0
              },
              "session" : {
                   "open cursor count" : 50,
                   "open session count" : 19,
                   "table alter failed calls" : 0,
                   "table alter successful calls" : 0,
                   "table alter unchanged and skipped" : 0,
                   "table compact failed calls" : 0,
                   "table compact successful calls" : 0,
                   "table create failed calls" : 0,
                   "table create successful calls" : 0,
                   "table drop failed calls" : 0,
                   "table drop successful calls" : 0,
                   "table rebalance failed calls" : 0,
                   "table rebalance successful calls" : 0,
                   "table rename failed calls" : 0,
                   "table rename successful calls" : 0,
                   "table salvage failed calls" : 0,
                   "table salvage successful calls" : 0,
                   "table truncate failed calls" : 0,
                   "table truncate successful calls" : 0,
                   "table verify failed calls" : 0,
                   "table verify successful calls" : 0
              },
              "thread-state" : {
                   "active filesystem fsync calls" : 0,
                   "active filesystem read calls" : 0,
                   "active filesystem write calls" : 0
              },
              "thread-yield" : {
                   "application thread time evicting (usecs)" : 0,
                   "application thread time waiting for cache (usecs)" : 0,
                   "page acquire busy blocked" : 0,
                   "page acquire eviction blocked" : 0,
                   "page acquire locked blocked" : 0,
                   "page acquire read blocked" : 0,
                   "page acquire time sleeping (usecs)" : 0
              },
              "transaction" : {
                   "number of named snapshots created" : 0,
                   "number of named snapshots dropped" : 0,
                   "transaction begins" : 1072,
                   "transaction checkpoint currently running" : 0,
                   "transaction checkpoint generation" : 11,
                   "transaction checkpoint max time (msecs)" : 60,
                   "transaction checkpoint min time (msecs)" : 29,
                   "transaction checkpoint most recent time (msecs)" : 38,
                   "transaction checkpoint scrub dirty target" : 0,
                   "transaction checkpoint scrub time (msecs)" : 0,
                   "transaction checkpoint total time (msecs)" : 425,
                   "transaction checkpoints" : 11,
                   "transaction checkpoints skipped because database was clean" : 0,
                   "transaction failures due to cache overflow" : 0,
                   "transaction fsync calls for checkpoint after allocating the transaction ID" : 11,
                   "transaction fsync duration for checkpoint after allocating the transaction ID (usecs)" : 19962,
                   "transaction range of IDs currently pinned" : 0,
                   "transaction range of IDs currently pinned by a checkpoint" : 0,
                   "transaction range of IDs currently pinned by named snapshots" : 0,
                   "transaction sync calls" : 0,
                   "transactions committed" : 72,
                   "transactions rolled back" : 1000
              },
              "concurrentTransactions" : {
                   "write" : {
                        "out" : 0,
                        "available" : 128,
                        "totalTickets" : 128
                   },
                   "read" : {
                        "out" : 0,
                        "available" : 128,
                        "totalTickets" : 128
                   }
              }
         },
         "mem" : {
              "bits" : 64,
              "resident" : 32,
              "virtual" : 3252,
              "supported" : true,
              "mapped" : 0,
              "mappedWithJournal" : 0
         },
         "metrics" : {
              "commands" : {
                   "_isSelf" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(2)
                   },
                   "buildInfo" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(4)
                   },
                   "dbStats" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(1)
                   },
                   "find" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(2)
                   },
                   "getLog" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(2)
                   },
                   "getMore" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(179)
                   },
                   "getnonce" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(1)
                   },
                   "isMaster" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(11)
                   },
                   "listCollections" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(1)
                   },
                   "listDatabases" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(1)
                   },
                   "ping" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(1)
                   },
                   "replSetGetRBID" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(1)
                   },
                   "replSetGetStatus" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(12)
                   },
                   "replSetHeartbeat" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(592)
                   },
                   "replSetUpdatePosition" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(356)
                   },
                   "serverStatus" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(15)
                   },
                   "whatsmyuri" : {
                        "failed" : NumberLong(0),
                        "total" : NumberLong(2)
                   }
              },
              "cursor" : {
                   "timedOut" : NumberLong(0),
                   "open" : {
                        "noTimeout" : NumberLong(0),
                        "pinned" : NumberLong(1),
                        "total" : NumberLong(1)
                   }
              },
              "document" : {
                   "deleted" : NumberLong(0),
                   "inserted" : NumberLong(0),
                   "returned" : NumberLong(62),
                   "updated" : NumberLong(0)
              },
              "getLastError" : {
                   "wtime" : {
                        "num" : 0,
                        "totalMillis" : 0
                   },
                   "wtimeouts" : NumberLong(0)
              },
              "operation" : {
                   "scanAndOrder" : NumberLong(0),
                   "writeConflicts" : NumberLong(1)
              },
              "queryExecutor" : {
                   "scanned" : NumberLong(0),
                   "scannedObjects" : NumberLong(62)
              },
              "record" : {
                   "moves" : NumberLong(0)
              },
              "repl" : {
                   "executor" : {
                        "counters" : {
                             "eventCreated" : 7,
                             "eventWait" : 7,
                             "cancels" : 668,
                             "waits" : 0,
                             "scheduledNetCmd" : 1525,
                             "scheduledDBWork" : 2,
                             "scheduledXclWork" : 0,
                             "scheduledWorkAt" : 2189,
                             "scheduledWork" : 0,
                             "schedulingFailures" : 0
                        },
                        "queues" : {
                             "networkInProgress" : 0,
                             "dbWorkInProgress" : 0,
                             "exclusiveInProgress" : 0,
                             "sleepers" : 4,
                             "ready" : 0,
                             "free" : 9
                        },
                        "unsignaledEvents" : 4,
                        "eventWaiters" : 0,
                        "shuttingDown" : false,
                        "networkInterface" : "\nNetworkInterfaceASIO Operations' Diagnostic:\nOperation:    Count:   \nConnecting    0        \nIn Progress   0        \nSucceeded     607      \nCanceled      0        \nFailed        918      \nTimed Out     0        \n\n"
                   },
                   "apply" : {
                        "attemptsToBecomeSecondary" : NumberLong(1),
                        "batches" : {
                             "num" : 0,
                             "totalMillis" : 0
                        },
                        "ops" : NumberLong(0)
                   },
                   "buffer" : {
                        "count" : NumberLong(0),
                        "maxSizeBytes" : NumberLong(268435456),
                        "sizeBytes" : NumberLong(0)
                   },
                   "initialSync" : {
                        "completed" : NumberLong(0),
                        "failedAttempts" : NumberLong(0),
                        "failures" : NumberLong(0)
                   },
                   "network" : {
                        "bytes" : NumberLong(0),
                        "getmores" : {
                             "num" : 0,
                             "totalMillis" : 0
                        },
                        "ops" : NumberLong(0),
                        "readersCreated" : NumberLong(0)
                   },
                   "preload" : {
                        "docs" : {
                             "num" : 0,
                             "totalMillis" : 0
                        },
                        "indexes" : {
                             "num" : 0,
                             "totalMillis" : 0
                        }
                   }
              },
              "storage" : {
                   "freelist" : {
                        "search" : {
                             "bucketExhausted" : NumberLong(0),
                             "requests" : NumberLong(0),
                             "scanned" : NumberLong(0)
                        }
                   }
              },
              "ttl" : {
                   "deletedDocuments" : NumberLong(0),
                   "passes" : NumberLong(10)
              }
         },
         "ok" : 1
    }


### 5、db.currentOp()  # 类似于mysql 的 show processlist
    数据当前操作命令
    {
         "inprog" : [
              {
                   "desc" : "conn5",
                   "threadId" : "0x700008b24000",
                   "connectionId" : 5,
                   "client" : "10.200.27.185:61335",
                   "active" : true,
                   "opid" : 3664, # 当前进程id
                   "secs_running" : 1,
                   "microsecs_running" : NumberLong(1224412),
                   "op" : "getmore",
                   "ns" : "local.oplog.rs",
                   "query" : {
                        "getMore" : NumberLong("14999006599"),
                        "collection" : "oplog.rs",
                        "maxTimeMS" : NumberLong(5000),
                        "term" : NumberLong(8),
                        "lastKnownCommittedOpTime" : {
                             "ts" : Timestamp(1499138238, 1),
                             "t" : NumberLong(8)
                        }
                   },
                   "originatingCommand" : {
                        "find" : "oplog.rs",
                        "filter" : {
                             "ts" : {
                                  "$gte" : Timestamp(1498818722, 1)
                             }
                        },
                        "tailable" : true,
                        "oplogReplay" : true,
                        "awaitData" : true,
                        "maxTimeMS" : NumberLong(60000),
                        "term" : NumberLong(8)
                   },
                   "planSummary" : "COLLSCAN",
                   "numYields" : 0,
                   "locks" : {
                       
                   },
                   "waitingForLock" : false,
                   "lockStats" : {
                        "Global" : {
                             "acquireCount" : {
                                  "r" : NumberLong(2)
                             }
                        },
                        "Database" : {
                             "acquireCount" : {
                                  "r" : NumberLong(1)
                             }
                        },
                        "oplog" : {
                             "acquireCount" : {
                                  "r" : NumberLong(1)
                             }
                        }
                   }
              }
         ],
    …
    ok: 1
    }

### 6、db.killOp(opid)
     删除某个进程 按上例子，db.killOp(3664)

## 日常命令总结
### 1、根据数组长度进行筛选
    db.getCollections(“xx”).find({
         field:{$size:2}
    })
    找出字段field长度为2的记录。
### 2、查找字段是否存在
    db.getCollections(“xx”).find({
     field:{$exists:1}
    }) 

### 3、类型嵌套
    db.getCollections(“xx”).find({
    ’user.name’: ‘rick'
    }) 

### 4、批量更新
    db.getCollections(“xx”).update({find}, {$set:{}}, false, true) 

    db.getCollection('app_question_ref').update(
    {"app_name":"afenti_yx", "extras.grade":{$exists:1}}
    ,{"$set":{"deleted_at":"2017-10-24 15:16:10"}},
    false,true
    )

###

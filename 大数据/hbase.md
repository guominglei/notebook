# hbase
    hbase 是一个高可靠性、高性能、面向列、可伸缩的分布式存储系统

## hbase 节点类型
    
    Master  主节点 负责存储table和region的meta信息（存储表的定义信息）管理region信息
        1、管理表的增删改查操作
        2、管理Regionserver的负载均衡。调整region分布
            2.1、region split后， 负责新的region分布
            2.2、RegionServer 停机后。负责失效的节上的region迁移。
    RegionServer 数据存储节点 负责存储具体数据

    zk  存储那些regionservier 存放有region的meat信息。主要提供给客户端使用。
        zk的作用：
            1.存放HBase的meta表数据、Region与RegionServer的关系、以及RegionServer的访问地址等信息。
                当HBase集群启动后，Master和RegionServer会分别向Zookeeper进行注册，并且在Zookeeper中保存meta表数据、Region与RegionServer的关系，以及RegionServer的访问地址等信息。meta表中维护着TableName与RowKey、RowKey与Region的关系。
            2.保证Master的高可用，当状态为Active的Master无法对外提供服务时，会将状态为StandBy的Master切换为Active状态。
            3.实时监控RegionServer，当某个RegionServer节点无法提供服务时将会通知Master，由Master进行RegionServer上的Region迁移。

## hbase存储机制
    RegionServer是存储节点。管理着region信息。
    每个表初始时只有一个region。当某个列族的压缩后的数据大于一个阈值时（默认256M。新的切策略是和regionserver中的region多少相关。如果当前region个数是1 。那么阈值就是 flush_size * 2。否则就是设置的最大值）时会水平分成两个region（分成两个region后。旧的region会下线。master可能会把这俩新region分到不同的regionserver 上达到负载平衡）
    
    region如何标识：
        1、<表名, start_rowkey, 创建时间>
        2、region 的end_rowkey 存储在 region的root表和 table的meta表里
    region如何查找：
        1、客户端利用表名通过zk的 /hbase/meta-region-server 获取到那些regionserver存放有region的meta信息。
        2、客户端根据rowkey 去请求 存放有region meta 信息的 regionserver（含有hbase:meta表）。 获取到具体的region 存放信息
        3、客户端根据具体的region存放信息 遍历具体文件。获取到实际内容。
        注意。客户端会缓存region meta信息。

        疑问？hbase:meta只会存放到一个server中吗？ 多个server如何分配？

    region 是分布式存储的最小单元。 单不是实际存储的最小单元。
        实际存储的最小单元是按列族划分形成的（store）。每一个store由一个memstore(内存写缓冲临时文件，到阈值后刷新到store file文件中。) 和多个store file 组成。 数据的安全性由hlog保证。


    store hbase的存储的核心。由 memstore 和storefile 构成
    log 
        每个HRegionServer中都会有一个HLog对象，HLog是一个实现Write Ahead Log的类，每次用户操作写入MemStore的同时，也会写一份数据到HLog文件，HLog文件定期会滚动出新，并删除旧的文件(已持久化到StoreFile中的数据)。当HRegionServer意外终止后，HMaster会通过Zookeeper感知，HMaster首先处理遗留的HLog文件，将不同region的log数据拆分，分别放到相应region目录下，然后再将失效的region重新分配，领取到这些region的HRegionServer在Load Region的过程中，会发现有历史HLog需要处理，因此会Replay HLog中的数据到MemStore中，然后flush到StoreFiles，完成数据恢复。

        疑问？ hmaster 如何能知道 regionserver 的log 里记录的内容？ LOG 存储在远端的hfs上。目录路文件格式固定。

### HBase处理读请求的流程
    1.HBase Client连接Zookeeper，根据TableName和RowKey从Zookeeper中查询这些记录对应存放的Region以及所关联的RegionServe。
    2.HBase Client请求这些RegionServer并找到对应的Region。
    3.如果Region的MemoryStore中存在目标的RowKey则直接从MemoryStore中进行查询，否则从StoreFile中进行查询。
     
### HBase处理写请求的流程
    1.HBase Client连接Zookeeper，根据TableName找到其Region列表及其关联的RegionServer。
    2.然后通过一定的算法计算出数据要写入的Region，并依次连接要写入的Region其对应的RegionServer。
    3.然后把数据写入到HLog和Region的MemoryStore中。
    4.每当MemoryStore中的大小达到128M时，会生成一个StoreFile，并写入到HDFS中。
    5.当StoreFile的数量超过一定时，会进行StoreFile的合并，将多个StoreFile文件合并成一个StoreFile，当StoreFile的文件大小超过指定的阈值时，会进行Region的切分，由Master将新的Region分配给合适的RegionServer进行管理（负载均衡）

    HBase Client会在第一次读取或写入时才需要连接Zookeeper，会将Zookeeper中的相关数据缓存到本地，往后直接从本地进行查询，当Zookeeper中的信息发生改变时，将会通过通知机制去通知HBase Client进行更新。

## 注意点
    jdk 要使用 版本8的。

## 列族

## 命名空间(namespace)
    
    命名空间指的是一个表的逻辑分组。同一分组中的表有类似用途。
    好处：
        1、资源的配额管理。限制一个命名空间可以使用多少 RegionServer Table。 如何操作？
        2、命名空间的安全管理。 如何做？
        3、RegionServer组。一个命名或者命名下的某个表可以指定到固定到一组或者多组RegionServer下。保证数据隔离性。 如何操作？

    系统默认有两个命名空间 。
        一个是hbase。这个是系统自己用的。
        一个是default。创建的表如果不指定命名空间。默认都到default里。


## 二级索引
    注意：自己单独为每个索引建了一个索引表。所以数据最少写了2份一份是索引表。另一份是真实数据。
         需要额外的存储空间，属于一种以空间换时间的方式。
    
    索引表的存储位置角度。分为 全局索引和本地索引。存储在哪里呢？
    索引表的可更改性角度。分为 可以更改和不可以更改。默认的都是可更改的。
    索引表的创建方式角度。分为 同步创建和异步创建。

    全局索引：
        默认索引类型， 适合多度少写。
        每次更改都要变更索引表。由于数节点和索引节点可能不在统一个节点上。带来了性能的消耗。
        搜索的字段和查询条件要是没有建立索引。将会全表扫描。

        覆盖索引: 
            索引覆盖其实就是将INCLUDE里面包含的列都存储到索引表里面，当检索的时候就可以从索引表里直接带回这些列值。
            注意索引列和索引覆盖列的区别：
                1、索引列在索引表里面是以rowkey的形式存在，多个索引列以某个约定的字节分割然后一起存储在rowkey里面，也就是说当索引列有很多个的时候，rowkey的长度也相应会变长，大小取决于索引列值的大小。
                2、而索引覆盖列，是存储在索引表的列族中。

                列族和rowkey的区别？

    本地索引：
        适合多写少读。
        索引数据和数据表的数据存放在统一个节点上。减少网络通信。 即使查询的字段不是索引字段。索引表也会引用（如何引用的？）
        开启本地索引需要更改hbase配置
        <property>
            <name>hbase.master.loadbalancer.class</name>
            <value>org.apache.phoenix.hbase.index.balancer.IndexLoadBalancer</value>
        </property>
        <property>
            <name>hbase.coprocessor.master.classes</name>
            <value>org.apache.phoenix.hbase.index.master.IndexMasterObserver</value>
        </property>

        #高能预警：如果使用的是Phoenix 4.3+的版本的话还需要在HBase集群的每个regionserver节点的hbase-site.xml中添加如下配置并重启HBase集群。
        <property>
            <name>hbase.coprocessor.regionserver.classes</name>
            <value>org.apache.hadoop.hbase.regionserver.LocalIndexMerger</value>
        </property>


### 索引优化 未验证
    1.index.builder.threads.max:（默认值：10）
        根据主表的更新来确定更新索引表的线程数

    2.index.builder.threads.keepalivetime：（默认值：60）
        builder线程池中线程的存活时间

    3.index.write.threads.max:（默认值：10）
        更新索引表时所能使用的线程数(即同时能更新多少张索引表)，其数量最好与索引表的数量一致

    4.index.write.threads.keepalivetime（默认值：60）
         更新索引表的线程所能存活的时间

    5.hbase.htable.threads.max（默认值：2147483647）
         每张索引表所能使用的线程(即在一张索引表中同时可以有多少线程对其进行写入更新)，增加此值可以提高更新索引的并发量

    6.hbase.htable.threads.keepalivetime（默认值：60）
         索引表上更新索引的线程的存活时间

    7.index.tablefactoy.cache.size（默认值：10）
         允许缓存的索引表的数量
         增加此值，可以在更新索引表时不用每次都去重复的创建htable，由于是缓存在内存中，所以其值越大，其需要的内存越多

### 过滤器

    比较运算符  
        = 相等， 
        <  小于
        <= 小于等于
        != 不等
        > 大于
        >= 大于等于
    比较规则：
        binary:rick   完全相等
        binaryprefix: ri  列值前缀为ri
        substring:rick  列值里包含rick
        regexstring:r*ck  符合正则规则的

    
    1、所有列都查找。只返回符合条件的列。其他列不返回
        ValueFilter(比较运算符, 比较规则)
        例如 ValueFilter(=, 'binary:rick')
    2、指定单列查找。返回符合查询条件的rowkey对应的所有列
        SingleColumnValueFilter(列族，列名， 运算符， 规则)
        例如： 
            SingleColumnValueFilter('info', 'address', =,'substring:rick')  # 有列名 
                除了info:address 列这一行其他列也返回。
            SingleColumnValueFilter('name', '', =,'substring:rick')         # 没有列名
                除了name 列这一行其他列也返回

    3、查找指定的列。 只返回符合条件的列。 有别于第2个
        ColumnValueFilter(列族，列名， 运算符， 规则)
        例如： 
            ColumnValueFilter('info', 'address', =,'substring:rick')  # 有列名  只返回 info:address 列
            ColumnValueFilter('name', '', =,'substring:rick')         # 没有列名
    4、行键过滤 返回 符合条件的所有行（行里所有列）
        RowFilter(运算符， 规则)

    5、对列族进行过滤 返回结果是列族名称。 查找包含名称为xx 的列族名称。
        FamilyFilter(运算符， 规则) 
    6、对列名进行过滤 。返回结果是列名的全称  列族:列名 形式的。  查找包含名称为xx 的子列。
        QualifierFilter(运算符， 规则) 

    7、基于列名的范围进行过滤 只返回符合条件的
        ColumnRangeFilter(start_value, 是否包含下限(ture/false), end_value, 是否包含上限(true/false))

    8、 只返回所有行。每行只返回第一个列。
        FirstKeyOnlyFilter() 如何做的？
        KeyOnlyFilter()  只返回第一个有值的列。  如何做的？


### shell 命令：
    1、创建命名空间
        create_namespace 'rick'
    2、查看某个命名空间
        describe_namespace  'rick'
    2、列出所有的命名空间
        list_namespace
    3、修改命名空间
        添加命名空间属性
        alter_namespace  'rick', {'METHOD => 'set', 'author'=>'rickgml'}
        删除命名空间属性
        alter_namespace  'rick', {'METHOD => 'unset', 'author'=>'rickgml'}
    4、删除命名空间
        drop_namespace 'rick'
        注意。删除命名空间前需要先把命名空间下的表都删除了。
    5、创建表
        create 'rick:userinfo', 'name', 'age'
    6、添加记录
        put 'rick:userinfo', 'rick', 'name', 'guo'
        put 'rick:userinfo', 'rick', 'age', '12'
        # rick:userinfo 是表名
        # rick 是 rowkey
        # name/age 是列名
        # guo/12 是列的值
    7、获取记录
        get 'rick:userinfo', 'rick'
    9、遍历
        9.1 遍历全表
            scan 'account'
        9.2 遍历整个列族
            scan 'account', {"COLUMN" => "name"}
        9.3 遍历列族中的某个列
            scan 'account', {"COLUMN"=> "info:address"}
        9.4
            scan 'account', {"STARTROW" => "rick_1", "LIMIT"=> 10, "STOPROW" => "rick_100", "FILTER" =>"ValueFilter(=,'binary:rick')"}
            从rowkey 为rick_1行开始 到 rowkey 为 rick_100 为止。 找出值为rick 的 10条记录。
            过滤条件有 
                "ValueFilter(=,'substring:6')"  包含有6的
                "ColumnPrefixFilter('birth')"  列前缀是 brith的
                "PrefixFilter('E')"  rowkey 前缀是 E 的

    10、查看表的行数
        count 'rick:userinfo'
    11、删除表(先停表，然后删除)
        disable 'rick:userinfo'
        drop 'rick:userinfo'
    12、添加列族
        alter 'account', 'id'
    13、删除列族
        alter 'account', {"NAME"=>'id', "METHOD"=>'delete'}
    14、查看表定义
        desc 'account'
    15、删除某条记录的某个子列
        delete 'account', 'rick', 'info:address'
    16、删除某行记录
        deleteall 'account', 'rick'
    17、查看所有表
        list

# phoenix
    是什么？方便操作hbase。 操作SQL化。

    将SQl查询编译为 HBase扫描
    确定扫描Rowkey的最佳开始和结束位置
    扫描并行执行
    将where子句推送到服务器端的过滤器
    通过协处理器进行聚合操作
    完美支持HBase二级索引创建
    DML命令以及通过DDL命令创建和操作表和版本化增量更改。
    容易集成：如Spark，Hive，Pig，Flume和Map Reduce。

## Phoenix与HBase的关系
 
    Phoenix与HBase中的表是独立的，两者之间没有必然的关系。
    Phoenix与HBase集成后会创建六张系统表：SYSTEM.CATALOG、SYSTEM.FUNCTION、SYSTEM.LOG、SYSTEM.SEQUENCE、SYSTEM.STATS，其中SYSTEM.CATALOG表用于存放Phoenix创建表时的元数据。
    Phoenix创建表时会自动调用HBase客户端创建相应的表，并且在SYSTEM.CATALOG系统表中记录Phoenix创建表时的元数据，其主键的值对应HBase的RowKey，非主键的列对应HBase的Column（列族不指定时为0，且列会进行编码）
    如果是通过Phoenix创建的表，那么必须通过Phoenix客户端来对表进行操作，因为通过Phoenix创建的表其非主键的列会进行编码。

## hbase phoenix
### 环境搭建
    1、 安装jdk8 注意版本。
    2、到 https://hbase.apache.org/downloads.html 下载hbase。 我下载的是 2.3.2
    3、到 https://phoenix.apache.org/download.html 下载hbase 对应的phoenix。 由于HBASE下载的是2.3.2。所以选择  5.0.0-HBase-2.0
    4、到 https://repo1.maven.org/maven2/org/apache/htrace/htrace-core/3.1.0-incubating/htrace-core-3.1.0-incubating.jar  下载 htrace-core-3.1.0-incubating.jar 包。（hbase 由于2.0 版本以后 不带 这个 jar包了 所以运行 phoenix 的时候会报错）
    5、解压hbase, phoenix。
    6、拷贝 phoenix 目录下的 phoenix-5.0.0-HBase-2.0-server.jar 到 hbase目录的 lib 下 
    7、拷贝下载的 htrace-core-3.1.0-incubating.jar  到 hbase目录的 lib 下 
    9、修改 hbase的配置文件 hbase/conf/hbase_site.xml.
        数据存储位置。
        <property>
            <name>hbase.rootdir</name>
            <value>file:///Users/guo/big_data/hbase_data</value>
        </property>
        zk 的数据存放位置
        <property>
            <name>hbase.zookeeper.property.dataDir</name>
            <value>/Users/guo/big_data/zk_data</value>
        </property>

        并把配置文件 拷贝到 phoenix/bin 目录下。

    10、进入 hbase/bin下运行 ./start_server.sh 启动hbase
    11、浏览网页 http://localhost:16010/  如果正常说明正常。
    12、进入phoenix/bin 目录 执行 ./sqlline.py localhost
        如果不报错误。 并且执行 !list 显示出的连接是open 的说明启动成功了。

### phoenix 和 命名空间表
    
    hbase 里的namespace 对应 phoenix里的schema。
    
    1、要想在phoenix里使用命名空间。 hbase需要修改参数。
        <property>
            <name>phoenix.schema.isNamespaceMappingEnabled</name>
            <value>true</value>
        </property>
        <property>
            <name>phoenix.schema.mapSystemTablesToNamespace</name>
            <value>true</value>
        </property>

    2、在hbase里创建命名空间。
        create_namespace "rick"
    
    3、在phoenix里创建 schema
        create schema if not exists "rick";
        create schema if not exists "default";

    4、剩下的和下边的shell 命令一致 不需要单独说明

### 映射旧表
    列族里只有一个元素。 如何映射暂时不知道
        假如hbase 的表结果是  name, pwd  两列
        尝试了：
            1、create table "default"."account"("ROW" varchar primary key, "name" varchar, "pwd" varchar)column_encoded_bytes=0
            列名出来了。但是数据为空
            2、create table "default"."account"("ROW" varchar primary key, "name"."" varchar, "pwd"."" varchar)column_encoded_bytes=0
            没有列明。但是数据显示出来了。

    列族里有多个元素。
        假如hbase的表结构是 info 列族下有 name, age 两个子列。
        create table "default"."account"("ROW" varchar primary key, "info"."name" varchar, "info"."pwd" varchar)column_encoded_bytes=0
        column_encoded_bytes=0 的含义是不进行列压缩。
        映射成功。

    hbase存在的旧表。phoenix进行建表映射时，只是在，catalog 表中建立了表的meta信息。 phoenix 映射成功后。 运行删除表命令后。 hbase上的表也会被删除。 这个需要注意下。 

### phoenix shell
    1、创建命名空间里的表
        说明 字段要是不加 ""。在hbase里的字段名称将是大写的。
        create table "rick"."userinfo"("id" integer primary key, "name" varchar(256), "age" integer default 0);

    2、添加数据（注意字符串的值必须用单引号）
        upsert into "rick"."userinfo" values(2, 'minglei',  21); 

    3、搜索数据
        select * from "rick"."userinfo" where "name"='guo';

    4、根据主键更新字段。
        upsert into "rick"."userinfo"("id", "name") values(3, 'guominglei');

    5、删除数据
        delete from "rick"."userinfo" where "id" = 1;

    6、修改表结构
        添加字段
        alter table "rick"."userinfo" add sex integer default 0;
            说明：字段 sex 由于没有加 "" 所以最终创建的字段名称将是SEX
        删除字段
        alter table "rick"."userinfo" drop column sex;
            说明: phoenix shell 报错了。 但是再搜索数据的时候。sex字段确实是删除的。
    7、创建索引
        create index "rick_userinfo_name" on "rick"."userinfo"("name");
        说明：
            需要更改hbase配置文件
            <property>
                <name>hbase.regionserver.wal.codec</name>
                <value>org.apache.hadoop.hbase.regionserver.wal.IndexedWALEditCodec</value>
            </property>

        查看索引是否创建成功？
            phoenix shell 下 
                !tables
                !indexes "rick"."userinfo"
            bash shell 下  list 命令
    7.1 创建覆盖索引
        create index "rick_cover_name_age" on "rick"."userinfo"("name") include("age");
        explain select "name","age" from "rick"."userinfo" where "name" = 'guominglei';  发现可以命中刚刚建立的好的覆盖索引

    7.2 创建本地索引
        create local index "rick_account_index" on "rick"."account"("name") include("age");
    
    7.3 异步创建索引(数据已经大量存在了) 未检验，需要实际操作。
        步骤1：create index "rick_cover_name_age" on "rick"."userinfo"("name") include("age") async;
        步骤2： 
            ${HBASE_HOME}/bin/hbase org.apache.phoenix.mapreduce.index.IndexTool
              --schema MY_SCHEMA --data-table MY_TABLE --index-table ASYNC_IDX
              --output-path ASYNC_IDX_HFILES

    8、删除索引
        drop index "rick_userinfo_name" on "rick"."userinfo";

    9、重建索引
        alter index "rick_cover_name_age" on "rick"."userinfo" rebuild;

    9、查看是否使用索引了。
        explain 里 full scan 是全表扫描没有用到索引。 range scan 用到了索引了。
        explain select * from "rick"."userinfo" where "name" = 'rick';   不使用索引
        explain select "name" from "rick"."userinfo" where "name" = 'guominglei'; 可以使用索引

        为啥？hbase rowkey 的缘故。

### 小细节
    phoenix 的queryserver 启动的时候要指定hbase的配置文件路径。 要不然，使用namespace 的时候报错！


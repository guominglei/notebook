#hadoop
## map reduce shuffle 机制
    MapReduce 中。map阶段处理的数据通过shuffle操作后，传递给reduce 阶段。

### shuffle
    shuffle 数据分区，排序，写文件
    具体流程：
    1、map task 搜集map 方法操作过的输出kv对。这时数据在内存缓冲区中。
    2、从内存缓冲区不断写到磁盘临时文件中。可以有多个文件，至少有一个。
    3、多个临时文件合并成大的文件。
    4、数据从内存到磁盘，以及文件合并过程中调用partitioner进行分组和针对key 进行排序。
    5、reduce task 根据自己的分区号，去各个map task 机器上获取相应分区的map 结果磁盘文件。
    6、reduce task 把不同map task 的相同分区文件进行合并 （归并排序）
    7、合并文件结束后，shuffle 过程结束。
    缓冲区越大，shuffle消耗时间越小。

### yarn  （yet another resource negotiator）
    yarn 是一个资源调度平台，负责为运算程序提供服务器运算资源。
    yarn 不清楚用户提交的程序的运行机制
    yarn 只提供运算资源的调度。用户程序向yarn 申请资源，yarn分配资源
    yarn 的管理者是 ResourceManager
    yarn 提供运算资源的角色是NodeManager

    ResourceManager
    基于应用程序对集群资源的需求进行调度的yarn集群主控节点。负责协调和管理整个集群的资源。响应用户提交的不同类型应用程序的解析 调度 监控等工作。ResourceManager会为每个application 启动一个applicationmaster 。applicationmaster分散在各个nodeManager节点

    ResourceManager 有两部分：
    调度器(Scheduler), 
    应用程序管理器(ApplicationsManager, ASM)

    NodeManager
    yarn集群资源提供者，执行应用程序的容器提供者。监控应用程序的资源使用情况。并通过心跳向集群资源调度器进行汇报

    ApplicationMaster 申请容器，监控任务
    对应一个应用程序， 向资源调度器申请执行任务的资源容器，运行任务，监控整个任务的执行。跟踪整个任务的状态，处理任务失败以及异常情况。

    Container
    抽象出来的逻辑资源单位。MapReduce程序的所有task都在同一个容器中执行完毕。容器大小可以调节。

    ASM
    ApplicationsManager 应用程序管理器ASM负责管理整个系统中所有应用程序，包括应用程序提交，与调度器协商资源以启动ApplicationMaster监控ApplicationMaster 运行状态并在失败时重启

    Scheduler 
    根据应用程序的资源需求进行资源分配，不参与应用程序具体的执行和监控等工作。资源分配的单位是Container,调度器是一个可插拔的组件。

#### yarn 作业流程：
    1、用户向yarn提交应用程序，其中包括ApplicationMaster程序， 启动ApplicationMaster的命令，用户程序等。
    2、ResourceManager 为该程序分配第一个Container。并与对应的NodeManager通讯，要求这个Container中启动应用程序ApplicationMaster.
    3、ApplicationMaster首先向ResourceManager 注册，这样用户可以直接通过ResourceManager 查看应用程序运行状态，然后将为各个任务申请资源，并监控它的运行状态，直到运行结束重复4-7步骤
    4、ApplicationMaster 采用轮询方式通过RPC协议向ResourceManager 申请和领取资源。
    5、ApplicationMaster申请到资源后，便和NodeManager通讯，要求它启用任务。
    6、NodeManager 为任务设置好运行环境（环境变量，jar包，二进制程序等等），将任务启动命令写入一个脚本中通过运行脚本启动任务。
    7、各个人物通过某个RPC协议向ApplicationMaster汇报自己的状态和进度，以让ApplicationMaster随时掌握各个任务的运行状态，从而可以在任务失败后重启任务
    8、应用程序运行完成后，AM向RM注销并关闭自己。

## hadoop单机版环境搭建
### 1、安装java
     export JAVA_HOME
### 2、下载hadoop源码。解压到相应的目录中。
### 3、修改配置文件。
    配置文件位于项目目录中的。/etc/hadoop中。

    core-site.xml  
        <configuration>
            <property>
                <name>hadoop.tmp.dir</name>
                <value>file:/usr/local/hadoop/tmp</value>
                <description>Abase for other temporary directories.</description>
            </property>
            <property>
                <name>fs.defaultFS</name>
                <value>hdfs://localhost:9000</value>
            </property>
        </configuration>

    hdfs-site.xml
        <configuration>
            <property>
                <name>dfs.replication</name>
                <value>1</value>
            </property>
            <property>
                <name>dfs.namenode.name.dir</name>
                <value>dir_path</value>
            </property>
            <property>
                <name>dfs.datanode.data.dir</name>
                <value>dir_path</value>
            </property>
        </configuration>

    mapped-site.xml
        <property>
            <name>mapreduce.framework.name</name>
            <value>yarn</value>
        </property>

    yarn-site.xml
        <property>
            <name>yarn.resourcemanager.hostname</name>
            <value>Master</value>
        </property>
        <property>
            <name>yarn.nodemanager.aux-services</name>
            <value>mapreduce_shuffle</value>
        </property>

### 启动命令：
    - 首先格式化namenode
        bin/hdfs namenode -format
    - 启动hdfs
        sbin/start-dfs.sh
    - 启动yarn.sh
        sbin/sart-yarn.sh
    - 查看进程是否启动
    jps 

## hdfs shell命令
    下边的shell命令只是shell下HDFS文件系统的命令
### 查看文件信息：
    ls 命令
    hadoop fs -ls /   查看当前hdfs系统根目录下的文件
    df 
    hadoop fs -df -h /  查看hdfs系统空间

    du
    hadoop fs -du -s -h dir_path  统计文件夹空间大小

    count 
    hadoop fs -count  dir_path  统计文件夹下的文件数

    setrep
    hadoop fs -setrep num file_path  设置指定文件副本数量

    chmod
    hadoop fs -chmod  777 file_path  设置文件权限

    chown  
    hadoop fs -chown  user:gourp  file_path  更改用户权限组

### 目录操作：
    mkdir 
    hadoop fs -mkdir -p path   在HDFS上创建目录

    删除目录
    rmdir
    hadoop fs -rmdir 删除目录 

### 本地和hdfs文件交互
    复制本地文件到hdfs：
    put 
    hadoop fs -put  local_source_path hdfs_dist_path      把本地文件放到hdfs中指定的位置 等同于

    copyFromLocal
    hadoop fs -copyFromLocal local_source_path hdfs_dist_path      把本地文件放到hdfs中指定的位置 
    
    获取hdfs文件到本地：
    get
    hadoop fs -get  hdfs_dist_path   从hdfs中指定的目录下载下来

    copyToLocal
    hadoop fs -copyToLocal  hdfs_dist_path  从hdfs中指定的目录下载下来

    把本地文件剪切粘贴到hdfs中
    moveFromLocal
    hadoop fs -moveFromLocal local_path  hdfs_dist_path

    把hdfs 中文件移动到本地  当前执行目录
    moveToLocal
    hadoop fs -moveToLocal  hdfs_source_path

    追加文件到已有文件
    appendToFile
    hadoop fs -appendToFile local_source_path  hdfs_dist_path   本地文件

### HDFS 系统内文件操作
    cp
    hadoop fs -cp  source_path  dist_path  HDFS系统内文件拷贝

    mv 
    hadoop fs -mv source_path  dist_path HDFS系统内文件拷贝

    rm
    hadoop fs -rm source_path    hdfs 系统内删除文件

    cat 
    hadoop fs -cat  hdfs_file_path  查看文件

    tail 
    hadoop fs -tail  hdfs_file_path 显示末尾

    text
    hadoop fs -text hdfs_file_path  以字符形式打印

## HDFS 如何保证数据完整性？
    1、数据按固定长度（默认512字节）来生成CRC校验和（4字节）。
    2、数据存储：datanode 收到数据后，检验成功了。才存储数据。并把数据转发给下一个datanode 节点。
    3、数据读取：客户端在收到数据后，也会验证CRC。如果验证成功，会给datanode发送数据片验证成功。datanode会更新此数据片的最后验证时间。
    4、datanode节点 有单独的后台进程定期来检查本节点内的数据片数据的完整性。如有错误数据片，给namenode 发消息告知此片数据有问题，然后本节点会丢弃这片数据。namenode会重新分配一个datanode来存储这个已经损伤的数据片。

    LocalFileSystem 来执行客户端的校验和验证。
    如果想不对数据进行校验，可以通过配置使用RawLocalFileSystem 
    bzip2 格式的压缩文件是支持分片的。其他格式的压缩文件不支持压缩分片。即，HDFS里的文件除了bzip2格式的压缩文件支持查询外，其他的都不支持。因为无法查询。
    LZO  压缩算法支持分片。通过索引实现切分。
    HDFS 里边存储的都是源文件。即不压缩的。
    注意：
        map 运算后的数据是可以压缩形式输出的。
    avro 独立于编程语言的数据序列化系统

## avro介绍
    avro 一个数据对象序列化系统。特点。支持二进制序列话，快速处理大量数据。可以使用动态语言处理avro数据。
    avro 依赖模式（schema）来实现数据结构定义（类定义）。avro对象（类的实例对象）。
    模式以json 对象来标示。
    网络通信中，由模式到数据是序列化 发送过程。 由数据到模式反序列化 接收过程。 序列化编码支持二进制编码（编号后数据量比json格式的小）和json编码。模式解释时，深度优先。从左到右遍历。


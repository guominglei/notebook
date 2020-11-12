# hive

## 元数据。 表结构定义
    Hive 中的元数据通常包括：表的名字，表的列和分区及其属性，表的属性（内部表和 外部表），表的数据所在目录
    Metastore 默认存在自带的 Derby 数据库中。缺点就是不适合多用户操作，并且数据存 储目录不固定。数据库跟着 Hive 走，极度不方便管理
    Hive 和 MySQL 之间通过 MetaStore 服务交互

## 元数据支持中文
   修改MySQL中的 meta信息
    1、修改表字段注解和表注解
        alter table COLUMNS_V2 modify column COMMENT varchar(256) character set utf8；
        alter table TABLE_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8； 
    2、修改分区字段注解
        alter table PARTITION_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8 ;
        alter table PARTITION_KEYS modify column PKEY_COMMENT varchar(4000) character set utf8;
    3、修改索引注解
        alter table INDEX_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;

    修改数据库连接支持编码
        <property>
            <name>javax.jdo.option.ConnectionURL</name>
            <value>jdbc:mysql://IP:3306/db_name?createDatabaseIfNotExist=true&amp;useUnicode=true&characterEncoding=UTF-8</value>
            <description>JDBC connect string for a JDBC metastore</description>
        </property>

## 内部表，外部表
    内部表。默认创建的表类型。 元数据和行数据都存储在hive系统中。 删除内部表时候，元数据和行数据都会被删除
    外部表。（场景：行数据先于hive中的表存在。） hive系统中只存储了表的元数据。 行数据存储于原先的hfs上。 删除表时，只删除元数据。外部存储的行数据不删除。

    hive 不允许删除存在有表的数据库。


# HIVE—索引、分区和分桶的区别
## 索引
    Hive支持索引，但是Hive的索引与关系型数据库中的索引并不相同，比如，Hive不支持主键或者外键。
    Hive索引可以建立在表中的某些列上，以提升一些操作的效率，例如减少MapReduce任务中需要读取的数据块的数量。

    为什么要创建索引？
    Hive的索引目的是提高Hive表指定列的查询速度。
    没有索引时，类似'WHERE tab1.col1 = 10'的查询，Hive会加载整张表或分区，然后处理所有的rows，
    但是如果在字段col1上面存在索引时，那么只会加载和处理文件的一部分。与其他传统数据库一样，增加索引在提升查询速度时，
    会消耗额外资源去创建索引表和需要更多的磁盘空间存储索引。
 
## 分区

    为了对表进行合理的管理以及提高查询效率，Hive可以将表组织成“分区”。
    分区是表的部分列的集合，可以为频繁使用的数据建立分区，这样查找分区中的数据时就不需要扫描全表，这对于提高查找效率很有帮助。
    分区是一种根据“分区列”（partition column）的值对表进行粗略划分的机制。
    Hive中每个分区对应着表很多的子目录，将所有的数据按照分区列放入到不同的子目录中去。

    为什么要分区？
    庞大的数据集可能需要耗费大量的时间去处理。在许多场景下，可以通过分区的方法减少每一次扫描总数据量，这种做法可以显著地改善性能。
    数据会依照单个或多个列进行分区，通常按照时间、地域或者是商业维度进行分区。比如vido表，分区的依据可以是电影的种类和评级，另外，按照拍摄时间划分可能会得到更一致的结果。为了达到性能表现的一致性，对不同列的划分应该让数据尽可能均匀分布。最好的情况下，分区的划分条件总是能够对应where语句的部分查询条件。
    Hive的分区使用HDFS的子目录功能实现。每一个子目录包含了分区对应的列名和每一列的值。但是由于HDFS并不支持大量的子目录，这也给分区的使用带来了限制。我们有必要对表中的分区数量进行预估，从而避免因为分区数量过大带来一系列问题。
    Hive查询通常使用分区的列作为查询条件。这样的做法可以指定MapReduce任务在HDFS中指定的子目录下完成扫描的工作。HDFS的文件目录结构可以像索引一样高效利用。

 
## 分桶（桶表）
    桶是通过对指定列进行哈希计算来实现的，通过哈希值将一个列名下的数据切分为一组桶，并使每个桶对应于该列名下的一个存储文件。

    为什么要分桶？
    在分区数量过于庞大以至于可能导致文件系统崩溃时，我们就需要使用分桶来解决问题了。
    分区中的数据可以被进一步拆分成桶，不同于分区对列直接进行拆分，桶往往使用列的哈希值对数据打散，并分发到各个不同的桶中从而完成数据的分桶过程。

    注意，hive使用对分桶所用的值进行hash，并用hash结果除以桶的个数做取余运算的方式来分桶，保证了每个桶中都有数据，但每个桶中的数据条数不一定相等。
    哈希函数的选择依赖于桶操作所针对的列的数据类型。除了数据采样，桶操作也可以用来实现高效的Map端连接操作。
    记住，在数据量足够大的情况下，分桶比分区，更高的查询效率。

## 总结
    索引和分区最大的区别就是索引不分割数据库，分区分割数据库。
    索引其实就是拿额外的存储空间换查询时间，但分区已经将整个大数据库按照分区列拆分成多个小数据库了。
    分区和分桶最大的区别就是分桶随机分割数据库，分区是非随机分割数据库。
    因为分桶是按照列的哈希函数进行分割的，相对比较平均；而分区是按照列的值来进行分割的，容易造成数据倾斜。
    其次两者的另一个区别就是分桶是对应不同的文件（细粒度），分区是对应不同的文件夹（粗粒度）。
    注意：普通表（外部表、内部表）、分区表这三个都是对应HDFS上的目录，桶表对应是目录里的文件


# hive 和 hbase 的区别

## 两者分别是什么？  
 　   Hive是一个构建在Hadoop基础设施之上的数据仓库。通过Hive可以使用HQL语言查询存放在HDFS上的数据。HQL是一种类SQL语言，这种语言最终被转化为Map/Reduce. 虽然Hive提供了SQL查询功能，但是Hive不能够进行交互查询--因为它只能够在Haoop上批量的执行Hadoop。

    HBase是一种Key/Value系统，它运行在HDFS之上。和Hive不一样，Hbase的能够在它的数据库上实时运行，而不是运行MapReduce任务。Hbase被分区为表格，表格又被进一步分割为列簇。列簇必须使用schema定义，列簇将某一类型列集合起来（列不要求schema定义）。例如，“message”列簇可能包含：“to”, ”from” “date”, “subject”, 和”body”. 每一个 key/value对在Hbase中被定义为一个cell，每一个key由row-key，列簇、列和时间戳。在Hbase中，行是key/value映射的集合，这个映射通过row-key来唯一标识。Hbase利用Hadoop的基础设施，可以利用通用的设备进行水平的扩展。

## 两者的特点
　　Hive帮助熟悉SQL的人运行MapReduce任务。因为它是JDBC兼容的，同时，它也能够和现存的SQL工具整合在一起。运行Hive查询会花费很长时间，因为它会默认遍历表中所有的数据。虽然有这样的缺点，一次遍历的数据量可以通过Hive的分区机制来控制。分区允许在数据集上运行过滤查询，这些数据集存储在不同的文件夹内，查询的时候只遍历指定文件夹（分区）中的数据。这种机制可以用来，例如，只处理在某一个时间范围内的文件，只要这些文件名中包括了时间格式。

　　HBase通过存储key/value来工作。它支持四种主要的操作：增加或者更新行，查看一个范围内的cell，获取指定的行，删除指定的行、列或者是列的版本。版本信息用来获取历史数据（每一行的历史数据可以被删除，然后通过Hbase compactions就可以释放出空间）。虽然HBase包括表格，但是schema仅仅被表格和列簇所要求，列不需要schema。Hbase的表格包括增加/计数功能。

## 限制
　　Hive目前不支持更新操作。另外，由于hive在hadoop上运行批量操作，它需要花费很长的时间，通常是几分钟到几个小时才可以获取到查询的结果。Hive必须提供预先定义好的schema将文件和目录映射到列，并且Hive与ACID不兼容。

　　HBase查询是通过特定的语言来编写的，这种语言需要重新学习。类SQL的功能可以通过Apache Phonenix实现，但这是以必须提供schema为代价的。另外，Hbase也并不是兼容所有的ACID特性，虽然它支持某些特性。最后但不是最重要的--为了运行Hbase，Zookeeper是必须的，zookeeper是一个用来进行分布式协调的服务，这些服务包括配置服务，维护元信息和命名空间服务。

## 应用场景
    Hive适合用来对一段时间内的数据进行分析查询，例如，用来计算趋势或者网站的日志。Hive不应该用来进行实时的查询。因为它需要很长时间才可以返回结果。
    Hbase非常适合用来进行大数据的实时查询。Facebook用Hbase进行消息和实时的分析。它也可以用来统计Facebook的连接数。

## 总结
　　Hive和Hbase是两种基于Hadoop的不同技术--Hive是一种类SQL的引擎，并且运行MapReduce任务，Hbase是一种在Hadoop之上的NoSQL 的Key/vale数据库。当然，这两种工具是可以同时使用的。就像用Google来搜索，用FaceBook进行社交一样，Hive可以用来进行统计查询，HBase可以用来进行实时查询，数据也可以从Hive写到Hbase，设置再从Hbase写回Hive。

## 下载连接
    hadoop:
        https://mirror.bit.edu.cn/apache/hadoop/
    hive:
        https://mirror.bit.edu.cn/apache/hive/

## 下载MySQL dev （mysql-connector-java-5.1.49.tar）
    https://dev.mysql.com/downloads/file/?id=496254
    解压后，把 mysql-connector-java-5.1.49-bin.jar 拷贝到 hive/lib 目录下。

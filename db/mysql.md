# mysql
## 修改线上表
    既然是线上修改，就是不想影响线上数据的正常访问。
    1、创建一个临时表，这个临时表的结构和目标表修改前的结构相同。
    2、对临时表进行alert操作。
    3、在目标表上创建insert, update, delete触发器.目标表有变动，也提现到临时表上。
    4、把目标表的数据拷贝到临时表上
    5、临时表替换目标表。

## 数据库事务处理并发数
TPS - Transactions Per Second（每秒传输的事物处理个数），即服务器每秒处理的事务数，如果是InnoDB会显示，没有InnoDB就不会显示。
TPS = (COM_COMMIT + COM_ROLLBACK)/UPTIME
    
    use information_schema;
    select VARIABLE_VALUE into @num_com from GLOBAL_STATUS where VARIABLE_NAME ='COM_COMMIT';
    select VARIABLE_VALUE into @num_roll from GLOBAL_STATUS where VARIABLE_NAME ='COM_ROLLBACK';
    select VARIABLE_VALUE into @uptime from GLOBAL_STATUS where VARIABLE_NAME ='UPTIME';
    select (@num_com+@num_roll)/@uptime;
    
QPS - Queries Per Second（每秒查询处理量）MyISAM 引擎
QPS = QUESTIONS/UPTIME
    
    use information_schema;
    select VARIABLE_VALUE into @num_queries from GLOBAL_STATUS where VARIABLE_NAME ='QUESTIONS';
    select VARIABLE_VALUE into @uptime from GLOBAL_STATUS where VARIABLE_NAME ='UPTIME';
    select @num_queries/@uptime;

## 查询例子
###找最大值
    mysql> select * from user_info;
    +------+----------+-----+---------+
    | name | day      | num | address |
    +------+----------+-----+---------+
    | rick | 20160208 |  10 | HN      |
    | rick | 20160209 |  99 | HN      |
    | rick | 20160209 |  80 | bk      |
    | gml  | 20160208 |  30 | GZ      |
    | gml  | 20160209 |  30 | GZ      |
    | guo  | 20160209 |  30 | GZ      |
    | Zxg  | 20160209 |  30 | GH      |
    原始数据如上：需要找出每天按人统计最多的num。的信息。
    即每天sum(num)最多的人。
    1 按天、name 分组出每天每人的信息。然后sum(num) 注意 group by day,name order by day,name
    2 把这些记录按sum(num)倒叙排列。
    3  把这些记录按天显示。
    2，3 步骤主要是解决 max(num) 获取到的name 不正确。group 总是把第一次获取到name 作为最后结果的name。

    select name, day,num from (select * from (select name,day,sum(num) as num from user_info group by day,name order by day,name) as a order by num desc) as b group by day;
    
    前N条
    select * from aa as a where N > (select count(*) from aa where day = a.day and num > a.num ) order by a.day,a.score desc;

## 范式
    第一范式：属性不能再分隔。即字段最小化。（原子性）
    第二范式：主键依赖。字段必须依赖主键。 （不相干的数据分开放）
    第三范式：消除冗余。非主键外的所有字段必须互不依赖。
    第四范式：消除表中的多值依赖。
    范式作用：避免数据的冗余和插入 删除 更新带来的异常。

## sql 注入
    执行裸写 sql 语句时候，需要注意sql 注入风险。
    解决方式如下：
        sql = "SELECT * FROM user_contacts WHERE username = %s"
        cursor = connection.cursor()
        cursor.execute(sql, [user]) 
    请注意在cursor.execute() 的SQL语句中使用“%s”，而不要在SQL内直接添加参数。 如果你使用这项技术，数据库基础库将会自动添加引号，同时在必要的情况下转意你的参数。
    sql 语句不要拼接。只需要提供占为符。最后的拼接工作交给数据库自己。

## DML对innodb索引影响
    首先需要了解一下innodb索引结构，关键下面几点：
    1，索引结构是B+tree；
    2，主键索引的索引结构和数据一起，即数据放到主键索引B+tree的叶子节点中；
    3，辅助索引的索引结构也是B+tree，但叶子节点存储关键字和对应的主键，不直接存储数据；
    基于前面三点，下面我们重点分析一下DML操作对innodb 索引的影响：
###1、innser操作
    在主键索引clusterd b+tree上插入数据；
    在每个辅助索引secondery b+tree上插入主键；
###2、delete操作
    在主键索引clusterd b+tree上删除数据；
    在每个辅助索引secondery b+tree上删除主键；
###3、主键update操作
    在主键索引clusterd b+tree上删除原有的数据；
    在主键索引clusterd b+tree上插入新的数据；
    在每个辅助索引secondery b+tree上删除之前的主键；
    在每个辅助索引secondery b+tree上插入新的主键；
###4、辅助索引update操作
    在主键索引clusterd b+tree上更新数据；（只需更新对应的叶子的数据部分，主键索引结构记录无需变更）
    在每个辅助索引secondery b+tree上删除原先的记录；（辅助索引发生变化，原先的记录要先删除，重新插入到B+树中）
    在每个辅助索引secondery b+tree上插入原先的记录；（主键不变化，即辅助索引结构记录的叶子数据不变）

## 视图View
    视图是个虚拟表（show tables  可以看出来。视图定义信息放到了mysql information_schema库  views表中）。
    视图不能创建索引。视图名在库内需要唯一。
    视图的数据，只有在查询的时候才会产生。

###视图有几个数据执行策略
    * 使用MERGE策略，MySQL会先将输入的查询语句和视图的声明语句进行合并，然后执行合并后的语句并返回。但是如果输入的查询语句中不允许包含一些聚合函数如: MIN, MAX, SUM, COUNT, AVG, etc., or DISTINCT, GROUP BY, HAVING, LIMIT, UNION, UNION ALL, subquery。同样如果视图声明没有指向任何数据表，也是不允许的。如果出现以上任意情况, MySQL默认会使用UNDEFINED策略。
    * 使用TEMPTABLE策略，MySQL先基于视图的声明创建一张temporary table，当输入查询语句时会直接查询这张temporary table。由于需要创建temporary table来存储视图的结果集, TEMPTABLE的效率要比MERGE策略低，另外使用temporary table策略的视图是无法更新的。
    * 使用UNDEFINED策略，如果创建视图的时候不指定策略，MySQL默认使用此策略。UNDEFINED策略会自动选择使用上述两种策略中的一个，优先选择MERGE策略，无法使用则转为TEMPTABLE策略。
###命令
    创建视图
    create ALGORITHM=MERGE view view_name as select * from xx;
    删除视图
    drop view view_name;

## 数据导出，导入
    mac 下 mysql默认安装在/usr/local/mysql 下。
###1、mysqldump方式（数据格式不变)
    导出某个表
        mysqldump -uroot -pxx database table> 导出数据存放绝对地址.sql
    导出某个库里的所有表
        mysqldump -uroot -pxx database > 导出数据存放绝对地址.sql 
    导入数据
        1、mysql -uroot -pxx database< 导入数据的绝对地址
        2、mysql -uroot -pxx 
        进入数据库命令行，切换数据库后，执行 source 导入数据的绝对地址。

###2、select 方式 (数据格式有变化)
    导出数据到某个文件中。
        select * from xx into outfile "xx.sql"  fields terminated by "," enclosed by "\"";
    导入数据
        load data infile "/tmp/task.sql" into table task fields terminated by "," enclosed by "\”";
    需要注意的是数据分隔信息的更改。这点比较重要。

## 常见问题
### blob/text column 默认值
    mysql 报错 ERROR 1101 (42000): BLOB/TEXT column can’t have a default value
    MySQL 在创建 ci_sessions 表的时候报错：ERROR 1101 (42000): BLOB/TEXT column can’t have a default value
    text或blob字段不允许有缺省值，这是由于strict mode导致的，只要在my.cnf中去掉
    sql-mode=”STRICT_TRANS_TABLES, NO_AUTO_CREATE_USER, NO_ENGINE_SUBSTITUTION”
    就行了。
## SQLAlchemy
### session 中对象的状态
    Transient  状态  实例对象即不在session 里，也不存贮在数据库中。当前对象只有一个映射关系。即这个对象对应的类和数据库中的那张表示对应的。
    Pending 状态   实例对象(transient状态对象)已经add到session中。注意，如果没有人为flush或则commit ，当前对象没有写入数据库中。
    Persistent  状态  实例对象(Pending状态) 已经写入到数据库中。注意只有flush commit这两个操作才能写入数据库。事务处理正确，不回滚。事务处理异常，可以回滚。
    Deleted  状态  实例对象在数据库中被删除。注意只有flush commit 这两个操作可以。事务处理正确，不回滚。事务处理异常，可以回滚。
    Detached 状态 实例对象和数据库中的记录解除绑定。当前对象，只能使用已经加载到的属性，未加载的其他属性不能获取。detached 后，就获取不到。
    比如，书籍的对象，有个作者信息外链。未detached 前，可以通过，book.aouther 来获取到作者信息。detached 后，如果之前没有book.auother 就获取不到

###session 里变量
    session.new   所有已经通过session.add(obj) 操作的obj 列表。 此obj 处于pending状态。 所有处于Pending状态的对象
    session.dirty   所有已经写入到数据库中的更改对象列表。所有处于Persistent 状态的对象
    session.deleted 所有已经在数据库中删除的对象列表。所有处于Deleted状态的对象
    session.identity_map   对上边对象做一个dict（主键和对象）

### 动态添加属性方法
    from sqlalchemy.ext.hybrid import hybrid_property, hybrid_method
    class Interval(Base):
        __tablename__ = 'interval'
        id = Column(Integer, primary_key=True)
        start = Column(Integer, nullable=False)
        end = Column(Integer, nullable=False)
        def __init__(self, start, end):
            self.start = start
            self.end = end
        @hybrid_property
        def length(self):
            return self.end - self.start
        @hybrid_method
        def contains(self, point):
            return (self.start <= point) & (point <= self.end)
        @hybrid_method
        def intersects(self, other):
            return self.contains(other.start) | self.contains(other.end)
    Session().query(Interval).filter_by(length=5)
    Session().query(Interval).filter_by(length>5)
    i1.contains(15)

## peewee

###1、读写分离配置
    使用第三方库 playhouse 
    from peewee import *
    from playhouse.read_slave import ReadSlaveModel
    from playhouse.pool import PooledMySQLDatabase
    master = PooledMySQLDatabase(stale_timeout=600, max_connections=32, **parse(ZyConfig.mysql_url))
    slave1=  PooledMySQLDatabase(stale_timeout=600, max_connections=32, **parse(ZyConfig.slave1_url))
    slave2=  PooledMySQLDatabase(stale_timeout=600, max_connections=32, **parse(ZyConfig.slave2_url))
    class BaseModel(ReadSlaveModel):
         class Meta:
              database = master  
              read_slaves = (slave1, slave2)

###2、编写migration 
    from playhouse.migrate import *
    title_field = CharField(default=‘')
    with master.transction():
         migrate(
              # 添加字段
              migrator.add_column(’table_name’, ‘field_name’, title_field),
              # 删除字段
              migrator.drop_column(’table_name’, ‘field_name’),
              # 字段重命名
              migrator.rename_column(’table_name’, ‘old_field_name’, ’new_field_name’),
              # 字段可为空
              migrator.drop_not_null(’table_name’, ‘field_name’),
              # 字段不能为空
              migrator.add_not_null(’table_name’, ‘field_name’),
              # 创建索引            表名          字段名称列表，                     是否是唯一性索引  
              migrator.add_index(’table_name’,(‘field_name_1’,’field_name_2’,..),False),
              #  创建成的索引名称就是 table_name_field_name_1_field_name_2
              # 删除索引
              migrator.drop_index(’table_name’, ’index_name')
         )

###3、由现有的库表生成model
     python -m pwiz -engine=mysql piano> model.py

###4、断开重连机制
     from peewee import *
     from playhouse.shortcuts import RetryOperationalError
     class MyDB(RetryOperationalError, MySQLDatabase):
          pass
     db = MyDB(xx)

## sqlite
###1.客户端/服务器程序
    如果你有许多的客户端程序要通过网络访问一个共享的数据库, 你应当考虑用一个客户端/服务器数据库来替代SQLite. SQLite可以通过网络文件系统工作, 但是因为和大多数网络文件系统都存在延时, 因此执行效率不会很高. 此外大多数网络文件系统在实现文件逻辑锁的方面都存在着bug(包括Unix 和windows). 如果文件锁没有正常的工作, 就可能出现在同一时间两个或更多的客户端程序更改同一个数据库的同一部分, 从而导致数据库出错. 因为这些问题是文件系统执行的时候本质上存在的bug, 因此SQLite没有办法避免它们.
    经验告诉我们, 应该避免在许多计算机需要通过一个网络文件系统同时访问同一个数据库的情况下使用SQLite.
###2.高流量网站
    SQLite通常情况下用作一个网站的后台数据库可以很好的工作. 但是如果你的网站的访问量大到你开始考虑采取分布式的数据库部署, 那么你应当毫不犹豫的考虑用一个企业级的客户端/服务器数据库来替代SQLite.
###3.超大的数据集
    当你在SQLite中开始一个事务处理的时候(事务处理会在任何写操作发生之前产生, 而不是必须要显示的调用BEGIN...COMMIT), 数据库引擎将不得不分配一小块脏页(文件缓冲页面)来帮助它自己管理回滚操作. 每1MB的数据库文件SQLite需要256字节. 对于小型的数据库这些空间不算什么, 但是当数据库增长到数十亿字节的时候, 缓冲页面的尺寸就会相当的大了. 如果你需要存储或修改几十GB的数据, 你应该考虑用其他的数据库引擎.
###4.高并发访问
    SQLite对于整个数据库文件进行读取/写入锁定. 这意味着如果任何进程读取了数据库中的某一部分, 其他所有进程都不能再对该数据库的任何部分进行写入操作. 同样的, 如果任何一个进程在对数据库进行写入操作, 其他所有进程都不能再读取该数据库的任何部分. 对于大多数情况这不算是什么问题. 在这些情况下每个程序使用数据库的时间都很短暂, 并且不会独占, 这样锁定至多会存在十几毫秒. 但是如果有些程序需要高并发, 那么这些程序就需要寻找其他的解决方案了。 

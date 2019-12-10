# postgresql

## 新特性
    服务器接收到请求时，会fork出一个进程，来实现的并发。注意这点的不同。
    用户即角色。创建用户时，默认这个角色有登陆权限。创建角色时，需要显示给角色赋值登陆权限。否则不能登陆

    数据库是通过复制指定数据库模板初始化的。默认有俩模板。如果在模板1这个库里新建一张表A。创建新库时指定模板为模板1。那么新库里也会有表A。

    数据和索引分开存储。一条数据需要两次读取才能取到。一次是读索引。另一次是取数据

    表数据，索引数据，1G一块。文件名加序号。
### 系统列
    每个表有如下系统列
    oid   一行对象的ID 
    tableoid 一行对象对应的表ID
    ctid  行版本所在的物理位置
    xmin    插入事务ID
    xmax    删除事务ID
    cmin    插入事务的命令标识符
    cmax    删除事务的命令标识符 

### 列
    serial  自增字段 . 进程间不安全。不能保证并发请求
    PRIMARY KEY 主键
    price numeric CHECK (price > 0)  字段约束

### 表之间可以继承
    父表的访问权限，不会被子表所继承。
    父表的索引，字段限制之类的，不会被子表继承。子表继承的只是父表的字段定义。

### 可以对行数据进行权限限制。限制那些人可以看到那些行。

### 模式
    默认有一个.public模式。如果不指定具体模式，使用的就是public模式。
    模式搜索路径默认是 1、当前用户同名模式， 2、public模式
    表实际上创建在模式下（模式名.表名）
        优点:
            1、允许多个用户使用同一个数据库。相互不干扰。
                如何实现？
                    表物理文件存储在对应的模式下的。不同模式的相同表名。对应的物理存储文件不同
                    https://yq.aliyun.com/articles/360154
                    https://blog.csdn.net/chichichichi/article/details/82189138
            2、把数据库对象组织成逻辑组，让它们便于管理。 暂时没想明白
            3、第三方应用存放在不同模式中。这样它们就不会和其它对象的名字冲突
        模式不能嵌套。

#### 模式命令
    # 创建模式
    CREATE SCHEMA myschema;
    # 删除模式  注意，模式下的表、都已经删除了
    DROP SCHEMA myschema;
    # 删除模式，及其所有的表及其他对象
    DROP SCHEMA myschema CASCADE;
    # 创建一个他人可以用的模式   ？？
    CREATE SCHEMA schemaname AUTHORIZATION username;
    # 查看搜索模式
    show search_path;
    # 查看模式列表
    \n

### 表分区
    1、创建一个主表。只有表字段。索引，限制啥都不加
    2、创建继承于主表的子表（分区表，继承表）。字段不添加。
    3、为分区表增加表约束以定义每个分区中允许的键值。 2，3 步骤例如：
        CREATE TABLE measurement_y2006m02 (
            CHECK ( logdate >= DATE '2006-02-01' AND logdate < DATE '2006-03-01' )
        ) INHERITS (measurement);
    4、子表（分区表，继承表）进行索引 主键约束等等其他操作
    5、向主表进行写入。如何分散到各个子表数据？
        1、可以给主表建立一个触发器。这样数据就分散存储到其他表里了。
        2、可以给主表定义规则
            CREATE RULE measurement_insert_y2006m02 AS
            ON INSERT TO measurement WHERE
                ( logdate >= DATE '2006-02-01' AND logdate < DATE '2006-03-01' )
            DO INSTEAD
                INSERT INTO measurement_y2006m02 VALUES (NEW.*);
    6、确保在postgresql.conf中constraint_exclusion配置参数没有被禁用。如果它被禁用，查询将不会被按照期望的方式优化。

## 索引
    b-tree 
        等值或范围查询。 结果可以指定排序。
        可以声明唯一性索引。其他不行
    hash
        等值查询。WAL 写提前策略不支持 hash索引。 数据库崩溃后需要reindex 数据库重新建立索引
        结果不能指定排序。只能返回和实现相关的排序
    GiST 
        通用搜索树。地理位置邻近点查询
        结果不能指定排序。只能返回和实现相关的排序
    GIN
        倒排序索引。
        只支持列为 字符串数组类型。数组内每一个元素。都会有一个索引。没有权重设置。
        结果不能指定排序。只能返回和实现相关的排序
        大批量插入（初始化时）。可以先删除索引。初始化数据后再创建索引
    BRIN 
        块范围索引
        结果不能指定排序。只能返回和实现相关的排序

    表达式索引
        如果某列查询时候，需要基于字段的函数表达式结果的判断。那么普通的字段索引将不起作用。
             CREATE INDEX test1_lower_col1_idx ON test1 (lower(col1));
            对字段 col1 做 lower的表达式索引。 这样查询的时候
             SELECT * FROM test1 WHERE lower(col1) = 'value'; 就可以命中索引
    部分索引
        表内的部分行（只有符合定义好规则的）数据有索引。其他没有索引。


## 并行查询
    max_parallel_workers_per_gather 控制一次查询最多可以并行多少。但是需要注意
    总工作进程由max_worker_processes限制。  
    所以可用的数量 max_parallel_workers_per_gather >= worker = max_worker - worker_now

    开启并行条件
    max_parallel_workers_per_gather > 0
    dynamic_shared_memory_type != none  进程间共享内存

    不能使用并行的情况
    1、隔离级别为序列化的。
    2、查询运行在一个已经存在的查询中。
    3、使用了任何被标记为PARALLEL UNSAFE的函数的查询。(用户自己定义的函数)
    4、查询可能被中途暂停的查询。
    5、查询中要写数据或者对数据进行锁定。

## 查询优化
    1、大量写入后。 要运行 ANALYZE 命令。这样可以更新pg_class 表内的信息。这样查询优化器，就可以使用最新的统计数据。选择更优的索引。

## mac 版本安装
    
    安装最新版本的
        brew install postgresql

    查询可用版本
        brew search postgresql

    安装指定版本的
        brew search postgresql@10

    定制shell命令
        vi ~/.bash_profile

        # 添加环境变量
        export PATH="/usr/local/opt/postgresql@10/bin:$PATH"
        export LDFLAGS="-L/usr/local/opt/postgresql@10/lib"
        export CPPFLAGS="-I/usr/local/opt/postgresql@10/include"
        export PKG_CONFIG_PATH="/usr/local/opt/postgresql@10/lib/pkgconfig"
        
        # 添加别名
        alias pg-start='brew services start postgresql@10'
        alias pg-stop='brew services stop postgresql@10'
        alias pg-restart='brew services restart postgresql@10'
        alias pg-status='pg_ctl -D /usr/local/var/postgresql@10 status'

### 相关文件说明
    config file: /usr/local/var/postgresql@10/postgresql.conf;
    client authentication config: /usr/local/var/postgresql@10/pg_hba.conf;
    config folder: /usr/local/var/postgresql@10;
    bin folder: /usr/local/Cellar/postgresql@10/10.10/bin;
    database location: /usr/local/var/postgresql@10

## 安装smlar插件
    可以根据汉明距离进行查询
    git clone git://sigaev.ru/smlar
    cd smlar
    USE_PGXS=1 PGUSER=postgres make && make install

### 使用第三方插件

    create extension smlar;  

## 常用命令
    
    查看版本信息
    postgres -V

    列出当前数据库的连接信息
    \conninfo

    创建用户
    CREATE ROLE postgres WITH LOGIN PASSWORD '123456';

    给用户授权
    GRANT ALL PRIVILEGES ON DATABASE dbname TO dbuser;

    更改用户密码
    \password user:xxx

    设置当前用户密码
    \password: xxx

    查看数据库列表
    \l

    创建数据库
    CREATE DATABASE 'image'  OWNER=rick ENCODING=UTF8 ;

    删除库
    DROP DATABASE postgres;

    切换数据库
    \c xx

    列出当前库中所有的表
    \d 

    查看某个具体的表结构
    \d xx

    建表
    create table hm(
        id int, 
        hmval bit(64),
    );

insert into hm select id, val::int8::bit(64) from (select id, sqrt(random())::numeric*9223372036854775807*2-9223372036854775807::numeric as val from generate_series(1,10000) t(id)) t;

    删除表
    drop table hm

    列出psql命令列表
    \? 

    查看某条命令的解释
    \h select

    查看用户
    \du
    
    打开文本编辑器
    \e

    退出
    \q

    查看已经安装的插件
    \dT

    查看当前所属模式
    \dn

# 统计信息
    
    pg_class 表统计的信息更新频率不太高。

    relpages 指的是表的磁盘空间占用量。 只是预估
    reltuples 表的行数。只是预估。

    select relpages from pg_class where relname='hm'



    SELECT attname, n_distinct, most_common_vals FROM pg_stats WHERE tablename = 'hm';



select * from hm1 where length(replace(bitxor(bit'0100110110001110001000100110011001001001100111011101111011111111', hmval)::text,'0','')) < 3;


    select    
        *,    
        smlar( hmarr, '{1_01001101,2_10001110,3_00100010,4_01100110,5_01001001,6_10011101,7_11011110,8_11111111}')    
      from    
        hm1    
      where    
        hmarr % '{1_01001101,2_10001110,3_00100010,4_01100110,5_01001001,6_10011101,7_11011110,8_11111111}'      
        and length(replace(bitxor(bit'0100110110001110001000100110011001001001100111011101111011111111', hmval)::text,'0','')) < 3 
      limit 100;  


    create index idx_hm3 on hm3 using gin(hmarr _text_sml_ops );  



## python 客户端

    pip install psycopg2





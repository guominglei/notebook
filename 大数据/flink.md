# flink

# 默认web地址是
    http://localhost:8081

## pyflink

    pip install apache-Flink

## 导出数据
    执行本地Python 代码。需要把依赖包放到 Python的 site-packages/pyflink/lib
    更好的方式是在代码里把要引入的jar包给指定了。这样远端和本地都能运行。
    t_env.get_config().get_configuration().set_string("pipeline.jars", "file:///Users/guo/Downloads/mysql-connector-java-8.0.22/mysql-connector-java-8.0.22.jar;file:///Users/guo/Downloads/flink-connector-jdbc_2.11-1.11.2.jar")

###  导出数据到MySQL
    需要注意的地方是， 
    版本小于 8 配置信息是：
        'connector.driver' = 'com.mysql.jdbc.Driver',
        MySQL 的驱动 mysql-connector-java-5.1.49.jar

    版本大于等于8 配置信息时
        'connector.driver' = 'com.mysql.cj.jdbc.Driver',
        MySQL 的驱动 mysql-connector-java-8.0.22.jar
        下载地址是 https://dev.mysql.com/downloads/connector/j/

    另外需要说明的是。flink不会自己创建表格。表格需要自己单独创建。

### 导出到hbase


## 提交作业

    1、提交一个Python Table的作业:
        ./bin/flink run -py WordCount.py
    2、提交一个有多个依赖的Python Table的作业:
        ./bin/flink run -py examples/python/table/batch/word_count.py \
                            -pyfs file:///user.txt,hdfs:///$namenode_address/username.txt
    
    3、提交一个Python Table的作业，并指定依赖的jar包。多个jar包 空格分隔？: 
        ./bin/flink run -py examples/python/table/batch/word_count.py -j <jarFile>
    
    4、提交一个有多个依赖的Python Table的作业，Python作业的主入口通过pym选项指定:
        ./bin/flink run -pym batch.word_count -pyfs examples/python/table/batch
    
    5、提交一个指定并发度为16的Python Table的作业:
        ./bin/flink run -p 16 -py examples/python/table/batch/word_count.py
    
    6、提交一个关闭flink日志输出的Python Table的作业:

        ./bin/flink run -q -py examples/python/table/batch/word_count.py
        提交一个运行在detached模式下的Python Table的作业:

    ./bin/flink run -d -py examples/python/table/batch/word_count.py
    提交一个运行在指定JobManager上的Python Table的作业:

    ./bin/flink run -m myJMHost:8081 \
                        -py examples/python/table/batch/word_count.py
    提交一个运行在有两个TaskManager的per-job YARN cluster的Python Table的作业:

    ./bin/flink run -m yarn-cluster \
                             -py examples/python/table/batch/word_count.py
# sentry
## sentry 是啥？
    sentry是一个错误信息收集系统。既然是异常消息收集系统。那么高并发下的可靠性就不太能保证。这点要清楚。
    错误信息在sentry系统中的处理流程：
    web接受消息 -> 分发到redis队列/RabbitMQ -> celery 分发消息到worker 把消息回写到mysql -> mysql ->web展示 通过查询mysql展示数据

## 依赖条件
    1、python 2.7.9 以上
    2、pip 8.1 +
    3、python-setuptools, python-dev, libxslt1-dev, gcc, libffi-dev, libjpeg-dev, libxml2-dev, libxslt-dev,libyaml-dev, libpq-dev
    4、redis
    5、postgresql

## 安装：
    pip install sentry
    出现的问题：
    1、是否安装postgresql postgresql-dev
    2、pip版本低 sudo python -m pip install -U pip
 
## 运行：
    1、初始化
        sentry init /home/mingleiguo/sentry_test/
        在指定目录下创建.sentry目录，目录下创建config.yml sentry.conf.py文件
        config.yml sentry系统相关配置 例如邮件服务， redis 服务， 系统物理文件存储位置
        sentry.conf.py 是web相关配置就是django的配置文件。
    2、更改配置文件。默认使用的数据库是postgresql。可以更改配置文件使用成mysql。注意数据库配置是提供给django使用的。
    3、初始化数据库 顺道创建用户 用来登录使用
        sentry django migrate
    4、运行web 默认端口是9000
    sentry run web
    5、运行 worker 用来接收客户端上传过来的数据
    sentry run worker
    6、运行shell ?用来干嘛的不知道
    sentry run cron
    7、在web中查看dns，在Python客户端中。添加测试数据。再页面中查看。
    8、删除过期数据  sentry cleanup --days=30

## supervisord配置文件

    [program:sentry-web]
    directory=/www/sentry/
    environment=SENTRY_CONF="/etc/sentry"
    command=/www/sentry/bin/sentry start
    autostart=true
    autorestart=true
    redirect_stderr=true
    stdout_logfile=syslog
    stderr_logfile=syslog

    [program:sentry-worker]
    directory=/www/sentry/
    environment=SENTRY_CONF="/etc/sentry"
    command=/www/sentry/bin/sentry run worker
    autostart=true
    autorestart=true
    redirect_stderr=true
    stdout_logfile=syslog
    stderr_logfile=syslog

    [program:sentry-cron]
    directory=/www/sentry/
    environment=SENTRY_CONF="/etc/sentry"
    command=/www/sentry/bin/sentry run cron
    autostart=true
    autorestart=true
    redirect_stderr=true
    stdout_logfile=syslog
    stderr_logfile=syslog
     sentry queues list 查看当前队列情况。

## sentry相关概念

    上下文Context   信息发送的时候可以人为添加必要的额外信息
    tags  人为定义的key value 对。用于标示信息 便于统计和搜索
    python 客户端如何使用？
        client.captureMessage(msg, tags={key:value, key:value})
    user  
        client.user_context(data={id:xx, username:xx, email:xx, ip_address:xx})
    extra
      client.extra_context(data={})

    通知
    1、警报
    2、工作流 状态变动提醒

    信息汇总
    1、根据权限
    2、根据exception 信息栈 模块、文件、
    3、自己定义fingerprint

    搜索
    token:value 格式

    node storage  存储 key/value 数据  gzip压缩后存入nodestore_node 表中 。 nodestore_node 和 sentry_message 有啥关系？

    write buffers 数据缓存 批量写入数据库。

    Time-series storage  存储聚合数据 例如 10分钟发生多少次。

    Throttles and Rate Limiting   控制消息频率。同个项目一分钟接受多少个消息。一个消息就是一个event?
    redis 默认存储数据， 频率数据可以在 config.yml  中 system.rate-limit: 500

## 杂记

    如何创建一个新的用户？
        
        sentry createuser

    如何创建一个项目？
        用户分配项目？
        用户权限分配？

    信息如何存储？
        
        消息存储在数据库中 sentry——message 表中
    如何分发信息？
        
        客户端在发送消息时指定具体项目。这个消息只存在于具体的某个项目中。项目之间互不干扰。

    消息和event的关系？
        
        一个消息就是一个event

## 客户端

### 发送方式
    1、sentry 客户端有针对flask的插件
        可以在flask程序中捕获异常，然后发送给sentry服务器。页面展示效果好
    2、监听现有log日志文件。展示效果差
### 例子
    client = raven.Clinet(dns)
    client.extra_context(data={"server": server_name})
    client.captureMessage(line, tags={"api":"userapi"})

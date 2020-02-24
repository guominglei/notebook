# celery

## 源码分析

## broker 是redis 数据传递

### 默认key
    
    _kombu.binding.celeryev         event队列
        类型为set,  记录节点ID  消费开启后，会把本身节点ID添加进去
        \x06\x16 是分隔符
        通信时，以消息订阅方式来实现。比如 
        celeryev
        celeryev/worker.heartbeat 

    _kombu.binding.celery.pidbox
        类型为set, celery remote control 节点信息。 
        例如：
            \x06\x16\x06\x16celery@guodeMBP.lan.celery.pidbox
        通信时，以消息订阅方式来实现。比如：
            celery.pidbox
   
    unacked_mutex
        类型  string  锁ID

    _kombu.binding.celery
        类型 set 记录 消息队列配置情况 默认为 celery\x06\x16\x06\x16celery

    # 任务
    celery  类型 list 默认的消息队列 优先级为0

    celery\x06\x163  优先级队列 其中 \x06\x16 是分隔符 3 是优先级
    celery\x06\x166  优先级队列 其中 \x06\x16 是分隔符 6 是优先级
    celery\x06\x169  优先级队列 其中 \x06\x16 是分隔符 9 是优先级

    # 任务确认
    unacked_index
        类型 有序集合set  (time, task_id) 
    unacked 
        类型 hash table  {task_id: task_message}

### 启动worker后有10个连接  默认连接池就是10个 
    估计连接池使用的不太合理


## Request 对象
    
    数据何时解析的？

    两个消息机制。
        1、send_event  内部使用 通过Redis订阅消息实现的
            task-revoked 事件，消息类型为 task 。以 - 分割。
        2、signal 机制供外部使用

## Connection
## Transport
## Channel
## Queue

## Worker

    初始化流程：
    1、hub
        生成 kombu Hub实例。
    2、pool
    3、consumer
        1、connection
        2、events
        3、heart
        4、mingle
        5、tasks
        6、control
        7、gossip
        8、event_loop

### 依赖组件
#### hub
#### pool
#### Consumer
##### 依赖组件
    
    connection  管理 consumer broker的连接

        创建 connection
        调用transport的register_with_event_loop把 conn 注册到hub上。
            hub添加on_tick事件（循环每次执行pool前先执行），如果Channel里的qos允许接收新事件。
                那么就把channel对应的fd,从epool里删除。再从新添加。并且发送brpop命令看是否可以获取到新的任务。
            hub添加定期事件。10秒做一次qos，看是否有未完成的任务没有(ack_later,任务有可能会重复做。)
            hub做客户端健康检查。

    
    events  管理worker监控事件 通过 Redis 进行事件传递
        初始化 consumer 实例的event_dispatcher (celery.events.dispatcher.EventDispatcher)实例
    
    Mingle  worker间发现、同步时钟协议

    gossip  worker间选master协议
    
    heart   发送worker心跳包
        发送给谁？

    control 远程命令管理服务

    tasks  管理consumer消费消息
        1、搜集任务生成strategies map
        2、
        qos

    evloop  启动consumer事件循环

        注册消息回调函数处理接收过来的消息。
        开始事件循环。

    agent

### pool 流程
    
    _inqueue    工作线程从这个queue里获取到任务。
    _outqueue   结果线程从这个queue里获取处理结果任务。
    _taskqueue  上层发送过来的任务队列

    _pool =[WorkerProcess(worker)]
        生成工作worker 并且启动worker
        worker通过 循环读取 _inqueue 看是否有任务到来。 
    
    worker_hander : 监控 pool里的workerprocess list
        定期清理工作线程。工作线程达到工作数量或者内存占用到限额了(worker本次任务执行完毕后退出。每次执行完毕后都会进行内存统计)。会退出。

    taskhander
        从taskqueue里获取任务。然后通过预设的put方法。把任务添加到_inqueue里供工作线程取用。
        如果没有任务
            向_outqueue 结果队里发送None，标识handler没有任务了。
            pool里有几个worker，就向_inqueue 发送几个None, 告诉worker 没有任务了。 


### 逻辑

    1、任务确认机制和QOS kombu (QOS, channel，multichannel 合起来完成的)
        默认是worker接收到任务。就可以需要进行任务确认。 
            删除  unacked_index 集合里的 task_id
            删除  unacked  hash table 里的 消息

        如果任务使用的是，ack_later 策略。 等任务处理完毕后再进行任务确认。
            假如任务处理时间比较长，会定期检查。会把在未确认的前N条记录重新做一次。 
            1、删除 unacked_index 里的task_id, 
            2、获取 unacked 里的任务信息。 在unacked里删除。 
            3、把任务重新放入消息队列里。

        unacked_mutex 数据恢复Redis锁。在确认是否有需要

    2、worker 获取任务机制。
        event_loop 每次执行 pool前，先看本channel的qos是否允许处理数据。
        如果允许。
            那么把channel对应的Redis fd, 先从pool里删除。然后再添加。并且发送 brpop celery,celery 3,celery 6 celery 9。 命令。看是否有新的任务到来。
            0，3，6，9 是优先级
        如果不允许。
            本次循环不做处理。




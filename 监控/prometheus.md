# prometheus 监控系统
## Python client 
    github: https://github.com/prometheus/client_python
    pip: pip install pip install prometheus_client

## 统计类型
### Histogram 直方图
    base_name = api_time
    bucket 阈值默认单位是秒。
    每个统计项数据：
    {base_name}_bucket 每个bucket 一条。共有N条。
    {base_name}_count 一条
    {base_name}_sum  一条
#### 查询P99耗时
    histogram_quantile(0.99, api_time_bucket{method_name="QA.view.api.v8.dialog.dialog_comment"})
#### 查询QPS
    查询1分钟内的请求量
    rate(api_time_count{method_name="QA.view.api.v8.dialog.dialog_comment"}[1m])

#### 统计机器QPS
    sum by (container_name)(rate(api_time_count[1m]))
#### 统计接口QPS
    sum(rate(api_time_count{method_name="QA.view.api.v8.dialog.dialog_comment"}[1m]))
#### 统计接口、机器、QPS
    sum by (container_name) (rate(api_time_bucket{method_name="QA.view.api.v8.dialog.dialog_comment"}[1m]))

#### histogram 和 summary 的比较
    不同：
        histogram 的 quantile 是根据 bucket 里的数据实时计算的。
        summary的quantile是计算好的。
    相同：
        都有 {base_name}_count，{base_name}_sum

## 动态配置节点
### file_fd_configs + consul_template
    blog: https://blog.51cto.com/xujpxm/1964878
    指定prometheus 配置文件目录。根据consul 里的key/value 信息。
    动态生成配置文件内容。达到服务动态发现

### consul_sd_configs
    代码: 
    blog:https://blog.51cto.com/1000682/2363038

## 获取数据方式 pull & push
    pull默认方式。不能跨网段。
    push客户端定期向pushgateway 发送格式好的监控数据。
        promethus节点从gatewaypull数据
        优点：避免了跨网段问题。
        问题：
            1、pushgateway 可能出现单点故障和瓶颈
            2、丧失了监控节点健康检查功能。
            3、除非手动清除gateway数据。否则gateway里的数据永远都会给promethus节点
               如果客户端不再给gateway push数据。那promethus节点采集到的数据就是最
               后一次push那个时间点的数据。之后的数据有问题。

## 本地存储 tsdb
    1、俩个小时一个block（内存中到俩小时后。刷盘）。
        block里有chuncks(样本数据)，
        meta.json（元数据）
         {
            "ulid": "01DJNTVX7GZ2M1EKB4TM76APV8",   # block id 
            "minTime": 1566237600000,               # 最小时间
            "maxTime": 1566244800000,               # 最大时间
            "stats": {
                "numSamples": 30432619,             # 样本数
                "numSeries": 65064,                 # 时间序数
                "numChunks": 255203                 # chunk数
            },
            "compaction": {
                "level": 1,                    # 压缩级别。级别越高，压缩次数越多
                "sources": [
                    "01DJNTVX7GZ2M1EKB4TM76APV8" # 原始block ulid
                ]
            },
            "version": 1
        } 
        index（索引文件）
            帮助通过metric name, labels找到chunk文件以及chunk文件中的位置
            toc 表 index总入口。记录其他index表的偏移量 8bit 。
            符号表，可以理解为一个set集合。只记录不同的。在写数据前先在toc表中记录偏移量
            时序列表。记录时序对应的chunk
            标签表。 记录标签组合的情况
            postings表。记录标签+时序关联关系。
            查找过程：
            先根据时间找出具体的block。然后读取block的index文件。只读取最后的52字节即可。6 * 8（index表 + 上述5张表） + 4(CRC校验)。获取到上述5张表的信息。
            1、从符号表中。找到匹配的符号ID
            2、从标签表中找根据符号id对应的标签
            3、从postings表中找出标签+时序对应的时序记录
            4、从时序记录中找出具体的chunk信息。

       后台会对多个block进行压缩。形成更大时间跨度的block数据。

    2、使用WAL(write ahead log) 预写日志系统。
    3、通过API删除的时序数据。删除记录保存在tombstone里。block里的数据做标记不删除。直到过期。整块删除。
    4、block时间，超过日志保留时间。直接丢弃。
        默认保留15天。 block块时间为2小时。最大时间段为36小时。
    5、数据压缩技术 delta-of-delta + xor
        block里 1K为一个chunk。 每个chunk有个一个开始时间，和初始值
        
        delta-of-delta 对时间进行压缩。 delta = (Tn - Tn-1)-(Tn-1 - Tn-2)
            1、第一个数据的时间戳t0，完整存储
            2、第二个数据的时间戳t1,实际存储的是t1-t0
            3、后续的时间到来时，实际存储的是(Tn - Tn-1)-(Tn-1 - Tn-2)
            如果值为0，使用1个bit存储0。
            如果值在[-8191,8192], 用16bit存储数据。前2位 10作为标示
            如果值在[-65535,65536],用20bit存储数据。前3位 110作为标示。17位存储其他值
            如果值在[-524287,524288],用24位存储数据。前4位 1110作为标示。20位存储
            如果超出上述值，用68位标示，前4位 1111作为标示。64位存储实际值
        xor 对数值进行压缩    value = Vn & Vn-1 然后进行压缩（数值相差不大相同位比较多）
            1、时序中第一个值不压缩，直接保存
            2、第二个数据之后，value = Vn & Vn-1
            如果值为0，标示两个值相同。用1个bit位存储
            如果值不为0，用2个bit控制位。
                10 标示结果中的非0播放被前一个xor运算结果包含。
                11 。需要5bit存储xor结果中前置0的数量。
                        6bit存储中间非0位的长度
                        最后存储中间非0位。
        缺点就是。查询时必须得对整个chunk进行解压缩。进行数据的还原。
    6、mmap技术。查询时数据不在内存中再读盘。不是内存数据库哦。
    7、按时间倒排索引。

## 数据存储到远端 influxdb中
    https://docs.influxdata.com/influxdb/v1.7/supported_protocols/prometheus

    配置文件里添加规则
    remote_write:
      - url: "http://xxxx:8086/api/v1/prom/write?db=prometheus"
    # remote_read:
    #   - url: "http://xxxx:8086/api/v1/prom/read?db=prometheus"

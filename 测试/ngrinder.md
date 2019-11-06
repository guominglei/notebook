# ngrinder
    github: http://naver.github.io/ngrinder/
    带WEB节面，controller-agent分布式结构的强大的压力测试工具。

# 常用测试工具比较
    JMeter
        基于UI操作，容易上手，但是不具备编程能力。其次JMeter基于线程模拟数千用户几乎不可能。
    Tsung
        基于Erlang，能模拟上千用户并且易于扩展。但是基于XML的DSL，描述场景能力弱，而且需要
        大量的数据处理才知道测试结果。
    Locust
        基于python的gevent，能模拟百万个用户。但是需要对python有一定理解。
    Loadrunner
        这个可以说是应用最多的一个，很方便，但是还是太重。往后的方向肯定是客户端工具逐步向平台化发展，所以loadrunner注定慢慢被淘汰（个人拙见）。而且不开源，扩展性不高，收费。
    nGrinder
        单节点支持3000并发、支持分布式、可监控被测服务器、可录制脚本、开源、平台化


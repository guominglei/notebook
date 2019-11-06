# consul 
    支持多数据中心分布式高可用的服务发现和配置共享软件。go语言开发。consul功能比较全。
    1、服务注册和发现机制。
    2、分布一致性协议实现 raft。
    3、健康检查 gossip。
    4、key/value键值对存储。
    5、多数据中心方案 多端口监听。

# 使用场景
    1、docker 实例的注册与配置共享
    2、coreos实例的注册与配置共享
    3、vitess集群
    4、saas应用的配置共享
    5、于confd服务集成，动态生成nginx和haproxy配置文件。

# 和其他系统的比较
## 相同点
    架构相识，都有服务节点。服务节点的操作都要求达到节点的仲裁数
    强一致性。
## 不同点
    1、zk, etcd 只提供一个原始的k/v值存储。需要自己书写服务发现功能。
      consul自带服务发现。
    2、consul通过gossip协议完成健康检查。zk需要通过客户端自己完成。etcd无健康检查
    3、consul，etcd用raft完成一致性。zk使用 paxos协议。 raft较paxos简单
    4、consul多服务数据中心。内外网服务采用不同的端口号避免单点故障。但是需要考虑延时问题。
       zk,etcd都是单端口监听。
    5、consul，zk都支持http， dns 协议接口。但zk集成比较复杂。 etcd只支持http。
    6、consul有web管理页面。etcd。zk无管理界面。

# 安装
    https://www.consul.io/downloads.html

# python 客户端
    pip install python-consul
    docs: https://python-consul.readthedocs.io/en/latest/

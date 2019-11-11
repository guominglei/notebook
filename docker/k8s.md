# k8s 
## k8s和swarm区别
    swarm董容器，k8s更懂管理。
    1、swarm偏重于容器的部署。k8s偏重于应用的部署。
        pod 里的docker可以共享volume, network namespace。这样就可以试同一个pod里的不同docker直接更好的交流。多个不同功能的docker部署在同一个pod里。
    2、k8s的docker调动策略比swarm多。
        swarm只有宿主机负载，已运行的容器多少，随机指定宿主机。
        k8s 端口冲突策略，容器挂载卷冲突，指定特定宿主机策略。等
    3、swarm和docker集成的好。安装简单。 k8s开发了许多组件。安装比较复杂。适应更多场景
    4、swarm 创建 管理 调度的最小力度是 docker。
       k8s 最小力度是pod。pod由 一个或者多个为实现某个特定功能放在一起的docker组成。
    5、 swarm 负载均衡采用 nginx + consul 缺点要么从新生成nginx镜像。要不手动更改nginx配置文件。
        k8s负载均衡做的好。通过service进行均衡。service是pod的访问入口。他指向一组有相同label的多个pod。
    6、swarm，k8s都支持灰度发布。swarm update时，docker镜像逐个全部替换为新的。k8s可以人为控制docker更新范围。
    7、k8s伸缩性能好。k8s可以控制pod使用的宿主机的cpu，内存等。这样k8s可以动态控制pod数量。swarm不能控制。
    8、swarm是docker官方推出的。k8s是Google开源的。 
    9、swarm集群，只有2层交互。容器启动是毫秒级别。 k8s是5层交互。容器启动是秒级别。

# k8s 核心组件
    etcd 保存集群状态
    apiserver 提供资源操作入口。
    controller manager负责维护集群状态。比如故障检测，滚动更新 等
    schedule 资源调度 按预先设置的调度策略进行pod调度到相应的node节点上
    kubelet 负责维护容器的生命周期，同时也负责管理挂载卷和网络
    container runtime 负责镜像管理以及pod和容器的真正运行
    kube-proxy 负责为service提供cluster内部的服务发现和负责均衡

# k8s 概念
    node 节点。可以是虚拟机。也可以是物理机
    Pod 最小调度单元。 
        pod内的 docker 共享文件系统和网络地址
    Lable 一个label 是一个key/value对。附着在资源上比如pod。便于筛选资源
    selector 通过匹配label来定义资源之间的关系
    namespace 资源前缀
    RC 复制控制器
    RS 副本集 现在代替RC 
    Deployment  部署 
        升级或则版本回退过程是两个副本集。旧的副本集pod数逐步减少。新的副本集pod数逐步增加。
    Service 服务
        pod 的IP是随时漂移的。外部无法固定访问。每个service有一个固定的虚拟IP。内部通过这个
        虚拟IP进行访问。 每个node上有一个负载均衡器(kube-proxy)。
    Job 任务 
    DaemonSet
        每个节点node上都会有一个daemonset类型的pod 比如日志，存储，监控等
    PetSet
        有状态pod。 
        适合于mysql之类的服务。当pod重启后，原来的存储没有了。或则是找不到了。这就不允许。
        petset就是解决，将不确定的pod和确定的存储关联起来。 目前处于测试阶段。
    Volume 存储卷 pod内所有docker共享
    PV（持久卷）PVC（持久卷声明） k8s提供存储逻辑抽象能力。




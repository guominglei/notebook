# locust
    使用requests库作为客户端进行请求的发送。使用 gevent, 非阻塞IO实现请求并发。
    单机可以实现数千并发。分布式测试。可以完成大规模的并发测试。
    优点：
        1、可以通过Python编写模拟用户行为的代码，简单易用。
        2、分布式可扩展，能够支持上百万用户。
        3、自带Web界面。
        4、不仅能测试web系统，也可以测试其它系统。

## 安装 
    pip install locust

## 书写测试脚本
    from locust import HttpLocust, TaskSet, task
    TOKEN = "DRY_wSEmcZgF1lHDRly43w|fhRpiw46jMu_2Ln4g_M5Dg"
    class ContributeTest(TaskSet):
        @task
        def get_contribute_list(self):
            url = "/v8/ugc/contribute/mine?token={}".format(TOKEN)
            self.client.get(url)
    class WebSiteUser(HttpLocust):
        host= 'xx'
        task_set = ContributeTest
        # 虚拟用户执行两次任务的间隔
        min_wait = 10     # 毫秒 
        max_wait = 20     # 毫秒
    执行流程：
        1、先执行ContributeTest中的on_start（只执行一次），作为初始化。没有定义可以忽略；
        2、从ContributeTest中随机挑选（如果定义了任务间的权重关系，那么就按照权重关系随机挑选）一个任务执行；
        3、根据Locust类中min_wait和max_wait定义的间隔时间范围（如果TaskSet类中也定义了min_wait或者max_wait，以TaskSet中的优先），在时间范围中随机取一个值，休眠等待；
        4、重复2~3步骤，直到测试任务终止

## 运行
    带WEB界面:
        locust -f xx.py --host=https://minglei-api.beta.crucio.hecdn.com/

        查看WEB界面
            http://localhost:8089
            1、设置虚拟用户数，设置并发数
            2、查看测试结果
    不带WEB界面:
        locust -f xx.py --no-web -c10  -r10*10 -t 1m --csv=xxxx
        说明： c 虚拟用户数， 
              r 为每秒启动多少次虚拟用户。可以理解为并发数。
              t 为持续时间
              csv 测试结果会在测试脚本当前目录下创建一个指定名称的csv文件。

    分布式测试：
        主从机必须运行相同的测试代码。 主机负责收集测试数据。从机负责进行压测测试。
        主机运行： locust -f xx.py --master
        从机带WEB界面运行： locust -f xx.py --slave --master-host=主机IP --host=xx
        从机带不带界面运行： locust -f xx.py --slave --master-host=主机IP --no-web --host=xx

locust  -f contribute.py  --host=https://minglei-api.beta.crucio.hecdn.com --no-web -c 10 -r 100 -t 1m --csv=list

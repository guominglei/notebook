# buildbot

## buildbot 环境搭建
     源码位置：git : https://github.com/buildbot/

###1、master
     a、建立虚拟环境：
          virtualenv --no-site-packages bb-master
          cd bb-master
     b、安装 buildbot master
          pip install buildbot
     c、安装buildbot www插件  web插件
          pip install buildbot-www
     d、安装 buildbot waterfall_view插件  web瀑布流插件
          pip install buildbot-waterfall_view
     e、安装buildbot console_view插件  web命令行插件
          pip install buildbot-console_view
     c,d,e 这三步骤如果不操作，那么照着官方的例子运行不起来。因为默认的master.cfg中有如下配置：
     c['www'] = dict(
          port=8010,
          plugins=dict(waterfall_view={}, console_view={})
     )
     创建 master
     buildbot create-master master (此处的master是一个路径)
     创建master 配置文件
     cp master/master.cfg.simple  master/master.cfg 这里使用系统默认的配置文件
     上边的步骤都完成后，运行 buildbot start master
     如果显示:
     Following twistd.log until startup finished..
     The buildmaster appears to have (re)started correctly.
     说明master启动成功了。日志文件位于 master/twistd.log
     可以访问 http://localhost:8010 来查看master的web界面
     buildbot reconfig master  更新master的配置文件重启 相当于reload吧我想

###2、worker
     a、建立虚拟环境：
          virtualenv --no-site-packages bb-worker
          cd bb-worker
     b、安装 buildbot worker
          pip install buildbot-worker
     c、创建worker
          ./bin/buildbot-worker create-worker worker localhost example-worker pass
     d、运行worker
          ./bin/buildbot-worker start worker
     看到：
     Following twistd.log until startup finished..
     The buildbot-worker appears to have (re)started correctly.
     说明成功了。

## 配置文件讲解
以master.cfg 配置文件为例
     
     change_source 需要编译和测试的代码源。
     scheduler 调度器有许多类型。比如 
          1、singlebranchscheduler  只有关注branch。不需要写change_filter。定期检查，如果有更新，执行
          2、anybranchscheduler 所有分支上。可以写change_filter。 定期检查，如果有更新，执行
          3、dependentscheduler 依赖某个调度器的调度器。
          4、forcescheduler  人员web页面手工触发。
          5、periodicscheduler 定期触发。
          6、nightlyscheduler period 调度器的一个变种。可以理解为 crontab那种方式的。
          7、trigglerable scheduler  触发器调度。触发条件是factory 中的 step。例子如下：
               checkin_factory.addStep(
                    steps.Trigger(schedulerNames=['build-all-platforms'],                                     
                         waitForFinish=True))
     builder buildbot具体执行单元。绑定需要用到的slave(就是woker)和需要执行的一系列命令。当build被scheduler触发后，会通过tcp去通知slave开始工作了。
     factory 是build的对slave的工作内容配置逻辑。也是buildbot的核心业务逻辑所在。
     slave 定义worker 名称 密码和 master 监听的端口号。 启动worker时候需要指定master端口号 worker 的名称 密码。 

1个worker 可以运行许多个任务。每个任务任务名称不一样。创建的文件夹也不一样。
如何运行实例？比如运行一个 python manage.py runserver  但是在这之前，如何在第一的时候安装环境依赖包呢？

一个master 可以有多个更新源。

以前 master(调度器进程)  slave（工作进程）  。现在都称为 worker

主从模式 即 一个master 多个worker这个是标准形式。

多主模式 即多个master组成一个群体。一个master配置UI、调度器。其他master配置一部分的builder。每个builder 在对应若干个worker.  

问题1、如果一个master挂掉了。是不是就是说这个master对应的buider 不起作用了呢？

可以有多进程master协同工作。
1、协同工作的master配置文件中都需要开通多进程模式。
c['multiMaster'] = True 开启多进程模式 。
2、多进程间数据共享通过MySQL。
3、一个master配置www web界面、更新源、调度器。其他协同工作的master只配置builder以及和builder相关的factory worker. 注意名称别重复了。
4、 master之间通过wamp 协议进行通信。所以需要一个 wamp 的 router 。文档推荐用的是crossbar。即想运行多master模式就需要启动一个crossbar进程。这个crossbar进程负责在master进程间分发消息。配置文件需要添加 mq 相关配置。注意utf8编码。如果文件是utf8的那么router_url 需要是u’’。 这个坑不小。

注意事项：
1、测试机器上运行环境需要人为创建。
2、master中的worker负载均衡
     1、 人为配置，某个master 配置固定的worker
     2、master 地址使用域名。worker 连接master 的时候使用域名连接。 通过DNS实现负载均衡。

## 邮件提醒

当测试、编译 结束后，可以发送邮件到指定用户。来通知用户这个任务的执行情况。
buildbot 的插件reporters已经有这个功能了。
用时候初始化一个reporters.MailNotifier实例。然后挂载到master配置文件上就行了。
假如 mail是 reporters.MailNotifier实例
mail = reporters.MailNotifier(
     fromaddr="guoml0126@126.com”,
     sendToInterestedUsers=False,
     extraRecipients=["guominglei@baofeng.com"],
     relayhost="smtp.126.com",
     #smtpPort="",
     smtpUser="guoml01",
     smtpPassword=“xx”,
)

c['services'] = []
c['services'].append(mail)

默认的事件是(failing, passing, warnings) 
下边是事件的所有定义
all
Always send mail about builds. Equivalent to (change, failing, passing, problem, warnings, exception).
warnings
Equivalent to (warnings, failing).
(list of strings). A combination of:
change
Send mail about builds which change status.
failing
Send mail about builds which fail.
passing
Send mail about builds which succeed.
problem
Send mail about a build which failed when the previous build has passed.
warnings
Send mail about builds which generate warnings.
exception
Send mail about builds which generate exceptions. 

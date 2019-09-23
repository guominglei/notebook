# 同步异步， 阻塞非阻塞
    同步异步针对的是指令。指令是否及时返回结果。
    阻塞非阻塞针对的是进程 线程 程序执行体。执行体是否继续执行其他代码块。

# python2 生成器，协程是一种写法。
# python3之后生成器写法不变。
    协程在原有生成器的基础上添加了标志位。用来标识是生成器协程还是原生协程。

# 生成器
    def Fibonacci(n):
        a, b = 0, 1
        while n > 0:
            yield b
            a, b = b, a + b
            n -= 1
# 协程
    types.coroutine 修饰器，把func设置了__code__.co_flags= CO_ITERABLE_COROUTINE
    用types.coroutine 定义的协程，称为基于生成器的协程
    async def func  编译时，func __code__.co_flags=CO_COROUTINE
    称为原生协程
    两种区别如下：
    原生协程不能有yield yield from。 yield 一个对象。 yield from 一个生产器
    原生协程被垃圾回收时，如果它从来没有被吊用过。会抛出RuntimeWarning.
    原生协程没有实现__iter__ 和 __next__ 方法
    生成器协程不能yield from 原生协程。 原生协程，生成器协程，不能混用
    原生协程，只能由 await 调用（如果对象定义了__await__方法就是 awaitable 对象）

# python3 协程 concurrent
    # 通过信号量来完成执行块之间的转换。
    # CPU计算工作可以有线程池，进程池两种方式。

# python3 异步 asyncio
    两种方式定义
    方式1：
    @asyncio.coroutine
    def say_hello():
        print("say hello")
        r = yield from asyncio.sleep(1)
        print("say hello")
    方式2：
    async def hello():
        print('Hello world! (%s)' % threading.currentThread())
        await asyncio.sleep(2)
        print('Hello again! (%s)' % threading.currentThread())

    结果存在 future 里。但是啥时候能获取到呢？没有看到 send 数据。或则 future.get_result

## coroutine 迭代
    def coroutine(func):
        @functools.wraps(func)
        def coro(*args, **kw):
            res = func(*args, **kw)
            if (base_futures.isfuture(res) or inspect.isgenerator(res) or
                isinstance(res, CoroWrapper)):
                res = yield from res
            elif _AwaitableABC is not None:
                # If 'func' returns an Awaitable (new in 3.5) we
                # want to run it.
                try:
                    await_meth = res.__await__
                except AttributeError:
                    pass
                else:
                    if isinstance(res, _AwaitableABC):
                        res = yield from await_meth()
            return res
        coro._is_coroutine = _is_coroutine  # For iscoroutinefunction().
        return coro

# Task
    把一个协程包装到一个fature中。 func-> fature -> task
    初始化task实例时候，会调用loop.call_soon(self._step) 
    把task._step包装成handler放入loop的ready 队列里。

## _step() 方法开始执行具体工作。一直不太明白。呵呵
    1、result = coro.send(None)  向 func send(None)。让func开始工作。
    2、如果抛出stopIteration 异常，说明内部fature执行结束。结果返回了。
        调用set_result方法，
            存储result结果。
            调用 loop.call_soon,把future下的callbacks添加到 loop._ready队列中
    3、判断 result结果
        如果 result 有 _asyncio_future_blocking 属性，代表是异步
            如果result loop 和 task loop 不同。报错
            如果 _asyncio_future_blocking=True 
                代表之前这个result是阻塞着的。
                把_asyncio_future_blocking=False。
                并把task._wakeup 注册到result的call_back 队列中。等待result运行结束后唤醒task

## _wakeup
    1、获取future 结果
    2、执行_step()

# concurrent.futures.Fauture
# asyncio.futures.Fature
     fature有三种状态，pending, cancelled, finished。
    1、asyncio.futures.Fature
    2、concurrent.futures.Fature 
    aio.Fature 和con.Fature aio.fature 兼容con.fature。区别
    1、aio Fature不是线程安全
    2、aio Fature的result,exception 没有带timeout参数。
    3、通过，add_done_callback函数，注册callback。回调函数肯定会执行。不管fature是取消了，还是完成了。
        future状态不是pending. callback回放到loop的ready队列中。等待执行。
        futuer状态是pending. callback放到future，回调队列里。等fature状态变为cancel或则finish再放到loop.ready对列中
    4、con Fature 有wait()，as_completed（）函数。 aio Fature 没有

    aio Fature, 独有字段，_asyncio_future_blocking = False 用于在Task._step 中区分返回结果。

## _scheddule_callbacks
    callbacks 队列置为空
    把原来callback添加到loop.ready 队列中。

## set_result 逻辑
    给result赋值。
    调用_scheddule_callbacks 把callback添加到loop.ready 队列中。

## set_exception 逻辑
    设置exception信息
    设置状态为finish
    调用_scheddule_callbacks 把callback添加到loop.ready 队列中。

## cancel 逻辑
    设置状态为 cancel
    调用_scheddule_callbacks 把callback添加到loop.ready 队列中。

## add_done_callback 逻辑
    如果fature状态为pending。函数放入call_back列表中。
    如果fature状态为cancel，finish 调用loop.call_soon。 把函数包装成handler 放入loop.ready队列中等待执行。

## remove_done_callback
    把 call_back 队列里 和 指定 函数 相同的 删除。

## --iter-- 逻辑
    如果没有完成， 
        _asyncio_future_blocking = True
        yield self
    判断是否完成了。
    返回结果。

## --await-- == --iter--

# BaseEventLoop
## run_forerver 逻辑
    1、检查执行环境
    2、执行run_once
## run_once 逻辑
    scheduled 就是callback队列
    1、如果scheduled队列长度超过100，并且需要取消的比例超过50%。
       整理scheduled队列，把该取消的任务取消了
    2、把scheduled里delayed任务删除了。
    3、处理IO事件
    4、处理延期事件（timeout）。scheduled队列的分发事件。
        把执行时间大于等于当前时间的事件放在ready队列里。
    5、执行ready队列里的事件。

# run_until_complete 逻辑
    1、参数是task或则 future。检查参数是否合法。
    2、调用loop.create_task。
        主要是future, loop的赋值。把task._step 添加的loop的ready对列中。
    3、给task添加callback。用于结果返回。结束loop循环。
    4、执行run_forever

# create_task逻辑
    有task factory就用指定的，工厂函数创建task。
    没有就用默认的。 

# call_soon 逻辑
    1、把执行函数封装成一个handler
    2、把handler放到loop的ready队列里。

# call_at 逻辑
    创建一个timer hander（有执行时间点）把这个time hander放入scheduled中。

# call_later 逻辑
    deal_time = now + delay 
    执行call_at(deal_time, callback, *args)

# ThreadPoolExecutor线程池，用于执行耗时任务。增加吞吐量。
    数据如何同步的？
    最大线程数是 CPU * 5


mongo 异步库 
    motor, 利用pymongo 实现tornado，和 python3 两个事件循环。
    motorengine。只实现了tornado事件循环。

redis 异步库 aioredis
mysql 异步库 aiomysql

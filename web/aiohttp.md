#aiohttp
    即可以做server。又可以做异步请求的http client。 

    中文例子
        https://segmentfault.com/a/1190000014587452
    doc:
        https://aiohttp.readthedocs.io/en/stable/
    git:
        https://github.com/aio-libs/aiohttp

    aiohttp + uvloop 。 用uvloop事件循环。替换aiohttp Python 自己的ioloop。可以实现大并发。
    uvloop:
        pip 安装: pip install uvloop
        替换loop
            import asynico
            import uvloop
            asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())

    但是http数据解析有点慢。可以使用 Httptools 
    httptools: 
        pip 安装：pip install httptools
        源码：https://github.com/MagicStack/httptools.git




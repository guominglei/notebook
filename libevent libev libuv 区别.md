# libevent libev libuv 区别
    libevent
        默认事件优先级相同。可以通过设置来更改。
        通过优先级队列来完成事件循环（event_loop
    libev
        默认事件优先级相同。可以通过设置来更改。
        通过优先级队列来完成事件循环（event_loop）

    libuv
        没有优先级概念。
        通过event_base来完成事件切换

    所有事件相event_base和loop都不是线程安全的，一个event_base或loop实例只能在用户的一个线程内访问（一般是主线程），注册到event_base或者loop的event都是串行访问的，即每个执行过程中，会按照优先级顺序访问已经激活的事件，执行其回调函数。所以在仅使用一个event_base或loop的情况下，回调函数的执行不存在并行关系同。

    libev 
        优点：
            是系统异步模型的简单封装，基本上来说，它解决了 epoll ，kqueuq 与 select 之间 API 不同的问题。保证使用 livev 的 API 编写出的程序可以在大多数 *nix 平台上运行。
            例如：
                libev 在 socket 发生读写事件时，只告诉你，“XX socket 可以读/写了，自己看着办吧”。往往我们需要自己申请内存并调用 read(3) 或者 write(3) 来响应 I/O 事件。
        缺点：
            1、使用起来比较麻烦。
                基本只是封装了 Event Library，用起来有诸多不便。
                比如 accept(3) 连接以后需要手动 setnonblocking 。
                从 socket 读写时需要检测 EAGAIN 、EWOULDBLOCK 和 EINTER。
                这也是大多数人认为异步程序难写的根本原因。
            2、没有异步 DNS 解析，这一点一直广为垢病。
            3、完全是单线程的。

    libuv 
        优点：
            1、处处回调。只需要关注回调即可。底层处理逻辑已经封装好了。
            libuv 是 joyent 给 Node 做的一套 I/O Library 。而这也导致了 libuv 最大的特点就是处处回调。基本上只要有可能阻塞的地方，libuv 都使用回调处理。这样做实际上大大减轻了程序员的工作量。因为当回调被 call 的时候，libuv 保证你有事可做，这样 EAGAIN 和 EWOULDBLOCK 之类的 handle 就不是程序员的工作了，libuv 会默默的帮你搞定。

            当接口可读时，libuv 会调用你的 allocate callback 来申请内存并将读到的内容写入。当读取完毕后，libuv 会 call 你为这个 socket 设置的回调函数，在参数中带着这个 buffer 的信息。你只需要负责处理这个 buffer 并且free 掉就OK了。因为是从 buffer 中读取数据，在你的 callback 被调用时数据已经 ready 了，所以程序员也就不用考虑阻塞的问题了。

            而对写的处理则更显巧妙。libuv 没有 write callback ，如果你想写东西，直接 generate 一个 write request 连着要写的 buffer 一起丢给 libuv ，libuv 会把你的 write request 加进相应 socket 的 write queue ，在 I/O 可写时按顺序写入。

            2、有异步的 DNS 解析，解析结果也是通过回调的方式通知程序。
            3、需要多线程库支持，因为其在内部维护了一个线程池来 handle 诸如 getaddrinfo(3) 这样的无法异步的调用。

    C 没有闭包，所以确定读写上下文是 libuv 的使用者需要面对的问题。否则程序面对汹涌而来的 buffer 也不能分得清哪个是哪个的数据。在这一点的处理上，libuv 跟 libev 一样，都是使用了一个 void *data 来解决问题。你可以用 data 这个 member 存储任何东西，这样当 buffer 来的时候，只需要简单的把 data cast 到你需要的类型就 OK 了。

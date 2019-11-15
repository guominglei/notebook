# mmap
    共享内存。将文件映射到一块内存区域中。最开始只申请虚拟地址。不调用真正的页。当对页进行引用的时候。会引起一个缺页中断，再将页面调入内存中。避免内存浪费

## 优点
    像操作文件一样，操作内存。适合大文件 减少文件拷贝次数

## 缺点
    内存申请是按页来的。
        1、如果实际数据小于一页的。那页内剩余的内存就浪费了。
        2、假如申请了3页空间，但是只有一页半的数据容量。前一页半数据的更改可以反映到磁盘文件了。第二页空闲的半页可以写入。但是数据不会反映在磁盘里。第三页不能读写。
        3、如果想把第二页剩余的半页写入磁盘。写数据前先更改文件大小然后内存数据才会同步到磁盘里）
    频繁申请mmap。并且每次size 不同。会照成缺少足够的连续内存地址。


## 用法
    1、匿名映射 主要用于父子进程间的共享内存。
    2、磁盘文件映射进程的虚拟空间。

## 映射区改动策略
    MAP_SHARED      修改数据，所有映射到同一个文件的进程，都可以看到。 内存会定期回写到磁盘文件里？
    MAP_PRIVATE     修改数据只有本进程，(如果有修改先拷贝一份原始的到新地址上。改动发生在新地址上 copy on write 写入复制)
    MAP_ANONYMOUS   匿名映射 父子进程内通信  fd 为 -1 或者为 /dev/zero文件

## mmap会写时机
    1、内存不够
    2、进程退出
    3、调用msync（触发会硬盘写） 或者 munmap（解除映射关系，依旧会调用mysync完成数据回写。） 
    4、靠系统脏页定期回写机制？

    问题？ 
        prometheus_client 没有发现有 调用munmap, msync 的情况。如何写入到文件中去的？
        被动等待系统脏页回写磁盘？

## 多个进程映射同一份文件如何做到的？
    通过swap cache机制完成的。发生缺页中断，先从 swap cache里找如果没有再读盘。
    每个文件在内存中有一个inode结构根据文件内的偏移量就能确认page。然后更新进程表

## 扩容
    1、munmap 解除映射关系
    2、物理文件扩容
    3、从新映射文件到目标大小


## python 相关实例
    1、https://github.com/grandinquisitor/python-mmap-bloomfilter
    2、https://github.com/dragondjf/jsonmmap

## 利用mmap做一个本地多进程共享的LRUcache
    难点：
        1、数据大小可能不一致。
            对策，存储数据块可以开放给用户让用户自己定义 这样数据大小统一。便于数据覆盖
        2、数据快速查找
            两个文件， 
            index  记录1、key, 2、data文件里的偏移量 3、过期时间
            data   记录 1、数据实际大小，2、实际数据
        3、更新策略
            1、获取最新的索引信息
            index 文件不满 接着读偏移量之后的。
            index 文件满了。
                读开始位置数据。
                    拿出来和本地存储的首位数据比对 看淘汰了几个。
                    然后再读取最新的数据。
            2、加锁更新



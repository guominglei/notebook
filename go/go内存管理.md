# go内存管理
    go 内存管理基于tcmalloc, 使用连续虚拟地址，以页8k为单位，多级缓存管理
    # 内存分配策略：
    1、极小对象 <16byte 直接从当前P的mcache上的tiny 缓存上分配
    2、小对象 16byte < size <= 32k byte 当前P所在的mcache 上对应slot的空闲列表中分配。无空闲列表则会继续向mcentral申请。如果还没有向mheap申请 
    3、大对象 >32K 直接通过mheap申请
    
    # 内存管理级别
    mspan  记录起始地址，预分配页数， 
    |
    mcache  
    每个go线程分配了mcache管理。不需要加锁操作（线程独享）。
    每个mcache有67个mspan数组。存储不同级别大小的mspan。
    这么说mspan 里的大小都是固定的？或则这么说，同一个mspan里的页相同。
    |
    mcentral
    mcache申请空间失败，向mcentral申请
    |
    mheap
    mheap拥有虚拟地址，同时拥有67个不同级别的mcentral


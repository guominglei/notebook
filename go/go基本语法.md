# 包的引用 作用域

    公有函数、变量 首字母大写。不同包可以引用
    私有函数、变量 首字母小写。本包可见，不同包不能引用。

# 1、引用第三方包，需要引用全路径。即在gopath目录下的相对路径。

    import “project1/package1/file1”

## 2、可以给引用的包起个别名

    import file1 "project1/package1/file1"
    file1 是别名  后边的是包名全称

## 3、引用多个包

    import (
        "project1/package1/file1"
        "project2/package2/file2"
    )
    # 特殊操作
    ## 点操作
    import . "fmt" 
    在调用fmt包的函数时候，可以省略fmt. 直接使用函数名调用    
    ## _操作
    import _ "包名"
    _操作其实是引入该包，不调用该包内的函数，但是调用该包的init函数

## 4、包的定义

使用package 关键字来声明一个包。声明语句是所在源文件的第一行非注释语句。
一个目录只能定义一个包（package），包名推荐和源文件所在目录名称保持一致。
    
        package firsttest  # 声明一个名为firsttest的包
        package main       # main 包，程序启动执行的入口

# 变量

## 1、基本定义
    var i int  # var 关键字， i 为变量名  int是变量类型 
    int 默认值为 0
    string 默认值为“”
    bool 默认值为 false

## 2、变行定义

    var a, b, c int  #  a,b,c 都是 int型的数据 默认值为0
    var (
            a int,      # a = 0
            b string,   # b = ""
            c bool,     # c = false
        )
    var (
            a,b,c = int # a=0 b=0 c=0
            d string    # d=""
        )
    var a = A"
    var a, b = 0, "B"
    a := 3  # 只能在函数体内进行定义赋值

## 3、常量

    const x int = 3
    const x,y int = 1,2
    const (
            a byte = "A"
            b string = "B"
            d int = 1

        )
    # 自动推导类型
    const x = 1 #   x int = 1
    # 组合定义时，复用表达式
    const (
            a = 3               # a = 3
            b                   # b = 3
            c                   # c = 3
            d = len("asbd")     # d = 4
            e                   # e = 4 
            f                   # f = 4
            g,h,i = 7,8,9       # g=7, h=8, i = 9
            x,y,z               # x=7, y=8, z=9 
        )

## 使用iota（自动递增枚举常量）
    # 1、单独赋值，iota数值不增加
    const a int = iota  # a = 0 
    const b int = iota  # b = 0
    const c int = iota  # c = 0
    # 2、组合赋值，iota数值增加
    const (
            a unit8 = iota  # a = 0
            b int = iota    # b = 1
            c float = iota  # c = 2
        )
    # 3、即使iota不在常量数组内第一个开始引用。也会按组内常量数量进行递增
    const (
            a = "A"
            b = "C"
            c = iota    # c = 2
        )
    # 4、iota 数据类型跟上一个数据相同
    const (
            a byte = iota   # a unit8 = 0
            b               # b unit8 = 1
            c rune = iota   # c int32 = 2
            d               # d int32 = 3
        )
    # 5、自增在一个常量组内有效。跳出常量组 iota 变为0
    const (
            a = iota        # a = 0
            b               # b = 1
            c               # c = 2
        )
    const (
            e = iota        # e = 0
            f               # f = 1
        )
    # 6、可以定制iota 步进值
    const (
            a = (iota + 2) * 3  # a = (0+2)*3
            b                   # b = (1+2)*3
            c                   # c = (2+2)*3
        )
# 数组
    数组声明中带有长度信息长度固定。数组元素默认值为0 。传递参数时会复制

    var a [3]int = [3]int{0,1,2}       # a = [0,1,2]
    var b [3]int = [3]int{}            # b = [0,0,0]
    # 先声明后初始化
    var c [3]int                       #  
    c = [3]int{}                       # c = [0, 0, 0]
    c = [3]int{0,0,0}                  # c = [0, 0, 0]
    # 
    d := [3]int{}                      # d = [0, 0, 0]
    # 声明时不指定长度
    var e = [...] int{1,2,3,4}         # 4 = [1,2,3,4] 长度根据初始化元素长度来。
    
    # 切片使用 
    # 连续切片  
    # 新数组的容量 = （cap(slice) - startindex) 即原来数组的容量-索引值
    s := []int{0,1,2,3,4,5}            # s = [0,1,2,3,4,5]  len=5 cap=5
    a := s[1:3]                        # a = [1,2,]  len=2, cap=4
    b := s[:4]                         # b = [0,1,2,3] len=4, cap=5
    c := s[1:]                         # c = [1,2,3,4] len=4, cap=4
    d := s[1:1]                        # d = [] len=0, cap=4
    e := s[:]                          # e = [0,1,2,3,4] len=5, cap=5
    
    # 跳跃切片
    s := make([]int, 5, 10)             # s=[]
    a := s[9:10:10]                     # a:[0] len:1, cap:1
    b := s[:3:5]                        # b:[0,0,0] len:3 cap:5
    slice[start:end:cap]
    容量大小的计算公式为 cap - start

    # 添加元素
    s := []string{}
    s = append(s, "a")              # 添加一个元素
    s = append(s, "b","c","d")      # 添加一列元素
    t "= []string{"e","f","g"}
    s = append(s, t...)                # 把t作为一列添加进s
    s = append(s, t[:2]...)            # 把t作为一列添加进s  
    # s = [a,b,c,d,e,f,g,f,g]
    s[0]= "A"
    s[len(s)-1] = "G"
    # s = ["A", "b", "c", "d", "e", "f", "g", "e", "G"]

    # 删除元素,
    func deleteByAppend(){
        i := 3
        s := []int{1,2,3,4,5,6,7}
        s = append(s[:i], s[i+1:]...)
    }
    func deleteByCopy(){
        i := 3
        s := []int{1,2,3,4,5,6,7}
        copy(s[i:], s[i+1:])
        s = s[:len(s)-1]
    } 

## 字典
    # 指定指定key, value类型。 随后初始化
    var m map[int]int = make(map[int]int)
    m[0] = 0

    # 指定key, value类型 和初始化值
    m := map[int]int{
        0:0,
        1:1,
    }

    # 只指定key类型， value类型自动适配
    mb := map[string]S{
        "a":S{1,2},
        "b":S{3,4},
    }
    # 修改
    m[0] = 3
    m[4] = 3
    a := mb["a"]
    c, is_exists := mb["c"]  # c 为值， is_exists为是否存在这个值。
    # 遍历
    for key,value :=range mb{
        fmt.Println(key, value)
    }

## 结构体 struct
    go 没有类的概念。结构体就是定义数据模型的。相对于其他语言类定义的数据模型
    go 没有继承的概念？
    定义一个结构体，里边有三个数据，A 为int， B， c 为string
    type S struct{
        A int,
        B,c string
    }
    var s S = S{0, "1", "2"}
    var s1 S = S{B: "1", A: 0}
    var a S
    var b = S{}
    结构体初始化：
    方式1 
        var a = new(S)
        a.A = 1
        a.B = "b"
        a.c = "c"
    方式2
        b := S{0, "1", "2"}
    方式3
        c := S{B: "1", A: 0, c:"c"}

## 指针 Pointer
    # 通过 & 获取对象指针
    var i int = 1
    pi := &i 
    a := []int{0, 1, 2}
    pa := &a
    var s *S = &S{0, "1", "2"}

## 通道 channel
    两个协程之间传递指定类型的值用来同步和通讯。信道里边的数据是有序的先进新消费

    # 有缓冲， 够3个才能接受，消费不了不接受新的消息
    var ch chan int = make(chan int, 3) 
    # 没有缓冲
    var ch chan int = make(chan int)
    var c0 chan int   # c0 是个双向通道
    var c1 chan<- int # c1 是个单项通道 发送 数据是int 
    var c2 <-chan int # c2 是个单项通道 接受 数据是int
   
    # 关闭通道
    close(ch) 
    通道关闭了只能读不能写

    # 循环读取管道数据
    # 方法1
    for msg := range ch{
        fmt.Println(msg)
        if msg == "send end"{   //  send end 是channel最后一个信息 不这样做。range可以形成死锁。比需得有打破这个死锁机制
            close(ch)
        }
    }
    # 方法2
    loop:
        for{
            select{
                case x, ok := <-ch:
                    if !ok{
                        close(ch)
                        break loop
                    }
                    fmt.Println(x)
            }
        }

## 接口 interface 
    interface机制有别于其他语言机制。非强制行。
    一个interface{}类型的变量包含了2个指针，一个指针指向值的类型，另外一个指针指向实际的值
    interface 比较 需要这两个指针都相对才行。
    如下实例。 接口Speek 定以了两个方法。
    但是Human 实例只实现了一个方法SayHi。
    var Any interface{}  空接口类型可以代表所有数据类型。就是其他语言的any 类型

    // 定义接口
    type Speek interface {
        SayHi()
        SayBye()
    }
    // 定义数据对象
    type Human struct {
        name  string
        age   int
        phone string
    }
    // 数据对象实现接口
    func (man *Human) SayHi() {
        fmt.Printf("Hi, I am %s-%d-%s\n", man.name, man.age, man.phone)
    }
    func main() {
        // 初始化
        man := Human{"rick", 10, "123456"}
        man.SayHi()
    }
    接口对象如何区分？

## 流程控制
    
    # if 判断条件可以不用()扩起来，但是执行体必须用{}扩起来
    if x > 10{
        fmt.Println("xx")
    }  

    # switch
    switch x /= 3; x{
        case x == 1:
            fmt.Println("1")
        case x == 2:
            fmt.Println("2")
        default:
            fmt.Println("3") 
    }  
    # switch 中每个分支执行过后，就跳出循环了。但是执行语句中添加 fallthrough 语句后，执行本分支后，接着进行后边的判断。
    switch x /= 3; x{
        case x == 1:
            fmt.Println("1")
            fallthrough
        case x == 2:
            fmt.Println("2")
            fallthrough
        default:
            fmt.Println("3") 
            fallthrough
    }
    上边例子改为fallthrough版后，最少会打印出3；1，3； 2，3； 这三种情况

    # for 循环
    小括号可以省略 大括号不能省略
    for i :=0; i < 10; i++ {}
    for i :=0; i< 10; {}
    for i :=0; ; i++ {}
    for ; i < 10; i++ {}
    for i :=0;; {}
    for {}
    # for range 相当于  enumarate([])
    # for range 只支持遍历 数组， 数组指针，slice, string, map, channel 类型数据
    a := [5]int{1,2,3,4,5}
    for index,value := range a {
        fmt.Println(index, value)
    }
    for index := range a {
        fmt.Println(index)
    }

## select 选择通道
    select 用于当前协程从一组可能的通讯中选择一个进一步处理
    当没有case 可以执行即多个通道都没有数据传递，select 语句会阻塞。直到有某个通道有数据传递过来为止
    case 只能操作ch的IO数据
    # sample 1 
    ch1 := make(chan int, 1)
    ch2 := make(chan int, 1)
    select {
        case <- ch1:
            fmt.Println("ch1 send")
        case <- ch1:
            fmt.Println("ch2 send")
    }
    # sample2  实现timeout
    timeout := make(chan bool, 1)
    func(){
        time.Sleep(1e8)
        timeout <- true
    }
    go func()
    ch := make(chan int)
    select {
        case <- ch:
            fmt.Println("ch")
        case <- timeout:
            fmt.Println("timeout")
        defatul:
            fmt.Println("default")
    }

## defer 延迟执行
    # defer 语句调用函数，将调用的函数加入到defer栈中， 栈中函数在defer所在的主函数返回时执行，执行顺序是先进后出，后进先出
    package main
    func main() {
        defer print(0)
        defer print(1)
        defer print(2)
    }
    // 输出 2 , 1, 0 
    # defer 在函数返回后执行，可以修改函数返回值
    func f() (i int) {
        defer func() {
            i *= 5
        }()
        return 3
    }
    # defer用于释放资源
    释放锁
    mu.Lock()
    defer mu.Unlock()
    关闭channel
    ch <- "hello"
    defer close(ch)
    关闭流
    f, err := os.Open("file.txt")
    defer f.Close()
    关闭数据库连接
    db, err := sql.Open("mysql", "user:pwd@tcp(ip:port)/database")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    # defer 用于恐慌的截获
    panic 用于产生异常，recover用于捕获异常。 recover 只能在defer语句中使用
    func p() {
        panic("exception ..")
    }
    func f() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Println("catch:", r)
            }
        }()
        p()
        fmt.Println("normal")
    }

## goto 跳转语句
    goto 用于在一个函数内部运行跳转到指定标签的代码处，不能跳转到其他函数中定义的标签
    goto 模拟循环
    func main() {
        i := 0
    loop:
        i++
        if i < 5 {
            goto loop
        }
        fmt.Println(i)
    }
    goto 模拟 continue, break
    func main() {
    i, sum := 0, 0
    head:
        for ; i <= 10; i++ {
            if i < 5 {
                i++
                goto head // continue
            }
            if i > 9 {
                goto tail // break
            }
            sum += i
        }
    tail:
        fmt.Println(sum) // print 35
    }

## function 函数
    func f(a int, c ...string){} 
    // c 代表是可变参数 可变参数只能放在函数的最后一个参数
    函数可以返回任意数量的返回值。 和python相同
    函数返回结果参数， 可以像变量那样命名和使用
    func f3()(a int, b string){
        a = 1
        b = "B"
        return   //  相当于 ruturn a, b 因为 返回类型和变量名已经定义过了。
    }
    func f2(a,b,c int) {}
    可变参数如何传递？请看下边例子
    func TestMultiParam(t *testing.T) {
        valueArray := []string{"1", "2", "3", "4", "5"}
        result := valueArray[0: 3]
        t.Log(result)
        multiParam(result...)  // 这里就是我们平时需要用到的
    }
    func multiParam(args ...string) {
        print(args)
    }

## 闭包 closure
     匿名函数，闭包，函数值

## init函数
    init 函数用于程序执行前做包的初始化工作的函数
    init函数没有参数没有返回值
    init函数在main函数执行前自动执行，不能被其他函数执行
    一个包或者go源文件可以有多个init函数。
        同一个go文件的init函数调用顺序是从上到下的
        同一个package中按go源文件名字符串比较从小到大执行调用各个文件中的init函数
        不同package，如果互不依赖的，按照main包中先 import的后调用的顺序调用其包中的init函数
        如果package存在依赖，则先调用最早被依赖的package的init函数

## test
    测试文件必须以_test.go结尾. 测试源文件需要引用 testing 包
    go test 

## 调试
    gdb 

## context
    解决一个请求，需要多个协程来完成。数据之间的共享问题

## 交叉编译
    Mac 下编译 Linux 和 Windows 64位可执行程序
    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build 
    CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build 
    Linux 下编译 Mac 和 Windows 64位可执行程序
    CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build 
    CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build

## map 只可以遍历数据，不能更改数据。修改需要从新赋值，切片可以就地更改数据
    目前 Golang 中的 map 实现不能保证元素地址固 定不变。内部的 hashable 在扩容过程中将移动元素位置，所 以如果 map 元素可以取地址，则保存外部指针变量中的地址 将可能变为野指针。map 元素取不了地址，就不能更改其属性。

## 易错点
    # type 只能使用在interface对象上
    # 函数多个返回值，有一个返回值有指定命名，其他返回值也得命名。如果返回值有多个
    # defer 在函数返回值返回前执行。 defer 先进后出，后进先出
    # select 随机性原则
        有一个case有return 则立刻执行
        多个case都能return 则随机抽取一个执行
        没有case能return 执行default
    # map线程安全 
    # 
 







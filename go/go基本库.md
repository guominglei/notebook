# 字符编码
    byte 是数组，可以原地更改，性能比string好。
    string  数据不可更改， 数值不能为nil, 
    rune 是字符的数字编码 int
    string 和 byte 在使用中应该通过指针来进行转换。要不然性能上会有不小的损失。

# reflect
    反射库，可以用来获取指定变量的类型
    reflect.TypeOf(i)
# fmt
    # 格式化字符串
    Sprint(str)  返回string 
    Sprintf(format, parmas)  返回string  
    Sprintln(st1, str1)  返回string 
    # 格式化字符输出到终端
    Print(str)  直接打印到终端
    Printf(format, parmas) 直接打印到终端
    Println(str1, str2) 直接打印到终端
    # 格式化字符串写入文件
    Fprint()    写入指定文件 F指的是file 
    Fprintf()   写入指定文件
    Fprintln()  写入指定文件
    # 生成错误信息
    Errorf(format, params) 返回error  返回一个错误，错误内容是format格式的
    # 从标准输入端获取数据
    Scan()      
    从标准终端获取，参数之间可以是空格分隔，解析数量够了。就不再接受了。 
    也可以是以回车键来分隔。
    Scanln()   不能用回车键来分隔参数。只能接受一个回车键。 
    Scanf()     可以指定参数之间的分隔符
    # 从给定的字符串中解析数据 用空格来分隔数据,回车
    Sscan() 
    Sscanf()
    Sscanln()  多个回车，解析有问题
    # 从文件或则其他IO流中获取数据 用空格来分隔数据
    Fscan()
    Fscanf()
    Fscanln()   多个回车解析有问题

# strings
    字符串拼接 最好使用下边这种形式，因为效率会非常高       
            b := bytes.Buffer{}
            b.WriteString("[")
            b.WriteString(v)
            b.WriteString("]")
            s = b.String()

# file 
    #os.file 
    读文件
    file.Wirte(b []byte) 写二进制文件
    file.WriteAt(b []byte, seek int) 偏移量写二进制文件 
    file.WriteString(s string) 写字符串
    写文件
    file.Read(b []byte) 读固定大小的字节
    file.ReadAt(b []byte, seek) 读固定大小的字节
    #io
    io.EOF 错误
    读文件
    写文件
    #bufio
    读文件
    写文件
    io/ioutil
    TempDir(dir, prefix string ) dir_path, err 指定目录下创建文件
    TempFile(dir, prefix string) file, err 指定目录下创建文件
    ReadDir(dirname string)  返回指定目录的目录信息的有序列表。
    ReadFile(file_path)]([]byte ,err) 读文件所有
    WriteFile(file_path, data []byte, perm) error 写文件
    ReadAll(io.Reader)([]byte, err) 一次性读取完毕

# path
# path/filepath

# time

    ## 格式化输出
    简写时间格式必须是 2006-01-02 15:04:05 换做 2016-01-02 15:04:05 就不能正确显示
    年月日，时分秒数值必须是2006 01 02 15 04 05 换成其他都不好使

# json

    # 方式1
    对象 转换到 bytes
    content_byte, _ := json.Marshal(content)
    bytes 转换到 对象
    json.Unmarshal(data, &content)
    # 方式2
    io 转换到 对象
    decoder := json.NewDecoder(resp.Body)
    decoder.Decode(&content)
    对象 转换到 io (bytes 直接发送到io)
    encoder := json.NewEncoder(io)
    encoder.Encode(&result)

# net 
    go协程的socket书写起来都是阻塞形式的。go runtime 内部实现是IO多路复用的。当这个协程在监听某个端口时，这个协程是阻塞的让出执行。其他协程可以执行。当事件到来时，go runtime 会唤醒相应的协程处理事件


# 自定义错误
    定义一个 struct 然后实现 Error() string {} 方法。





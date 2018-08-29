# bee go web框架
    restful 框架
    json 数据是 json.dumps() 后的字符串
    jsonp 数据是脚本代码可以js直接运行

## 基本命令
    bee new 项目名称  在你的GOPATH/src目录下创建新的项目 
    bee api 项目名称  在你的GOPATH/src目录下创建新的项目 
        比 new 命令创建的项目少了static views
    bee run 通过 fsnotify 随时监控文件的变动 自动重新编译运行文件。默认端口8080。 必须在项目源代码目录下运行
    bee pack 项目源代码目录下运行 打包代码用于部署
    bee generate 从数据库自动生成model
    bee migrate 数据库迁移命令

## controller
    ### 1、 router
    #### 固定路由
        beego.Router("/", &controller.MainController{})
    #### 正则路由
        beego.Router(“/api/:id”, &controllers.RController{})
        参数id 获取方式
        this.Ctx.Input.Param(":id")
    #### 自定义方法以及restful规则
        beego.Router("/api/list",&RestController{},"*:ListFood")  * 所有方法
        beego.Router("/simple",&SimpleController{},"get:GetFunc;post:PostFunc")
        一个uRL可以根据method来匹配不同的方法。不同选择之间用;来分隔 
        beego.Router("/api",&RestController{},"get,post:ApiFunc")
        多个httpmethod对应一个处理函数
    #### 自动匹配
        # 形式1
        /object/login   调用 ObjectController 中的 Login 方法
        get参数通过url?来传递，解析this.Input().Get("id") 
        post参数通过body传递， 解析this.ParseForm(&u) 
        # 形式2
        /object/blog/2013/09/12  调用 ObjectController 中的 Blog 方法，
        参数如下：map[0:2013 1:09 2:12]
        获取this.Ctx.Input.Params()["0"] == 2013
        获取this.Ctx.Input.Params()["1"] == 09
        获取this.Ctx.Input.Params()["2"] == 12
        # 获取请求后缀
        this.Ctx.Input.Param(":ext") 可以获取请求uRL的后缀比如 .json, 
        /controller/simple.html
        /controller/simple.json
        /controller/simple.xml
        使用这种形式，参数只能以?key=value形式来传递 或则放到body里边
        只需要在rooter.go中
        beego.AutoRouter(&controllers.ObjectController{})
        this.Ctx.Input.Param(":ext")
    #### 注解路由
        注解路由是自动匹配路由的升级版。
        在controller里边添加uRL和方法映射关系。
        router.go 中加入
        beego.Include(&CMSController{}) 就行了。
    #### namespace 
        可以添加前缀，namespace 可以嵌套

    ### 2、参数获取
    #### get url?key=value
        this.GetString("id")
        this.GetStrings(key string) []string
        this.GetInt(key string) (int64, error)
        this.GetBool(key string) (bool, error)
        this.GetFloat(key string) (float64, error)
        # 如果数据类型不是想要的，可以先获取数据，然后根据需要自己转格式
        this.Input().Get("id") 
    #### get object/:id
        this.Ctx.Input.Param(":id")
    #### post DATA
        this.ParseForm(&u)
    #### 直接解析body里边数据解析
        配置文件里添加
            copyrequestbody = true 
        controller里边
            var ob models.Object
            json.Unmarshal(this.Ctx.Input.RequestBody, &ob)
            objectid := models.AddOne(ob)
    #### 处理文件上传
        配置文件 调整文件缓存
        maxmemory = 1<<22
        beego有两个函数可以读取文件
        f, h, err := c.GetFile("uploadname")
        c.SaveToFile("uploadname", "static/upload/" + h.Filename) 

    ### 3、通用错误处理
    ####
        修改框架默认 401、403、404、500、503 处理方法
        beego.ErrorHandler("404",page_not_found)
    #### 把自己写的错误处理方法统一放到 controller里边。例如：
        beego.ErrorController(&controllers.ErrorController{}) // 定义错误处理函数

    ### 提前终止执行  this 是controller 实例
        this.StopRun()  

## model
    ### 表名
    类名 默认首字母大写。 对应的表名是 小写，用_连接
    比如 AuthUser 是类名 。对应的表名就是  auth_user
    如果想人为更改表名，让类实现TableName这个方法
    func (u *User) TableName() string {
        return "auth_user"
    }
    ### 添加索引
    func (u *User) TableIndex() [][]string {
        return [][]string{
            []string{"Id", "Name"},
        }
    }   
    ### 设置多字段唯一性
    func (u *User) TableUnique() [][]string {
        return [][]string{
            []string{"Name", "Email"},
        }
    }
    ### 修改引擎
    func (u *User) TableEngine() string {
        return "INNODB"
    }
    ### 字段属性设置
    ##### 默认值
        `orm:"default(xx)"`
    ##### 字段名称
        `orm:"column(xx)"`
    ##### 字符串长度
        `orm:"size(60)"`
    ##### Int Float 精度
        `orm:"digits(10);decimals(4)"`
    ##### 允许为空 
        `orm:"null"`
    ##### 唯一性
        `orm:"unique"`
    ##### 时间字段默认值
        `orm:"auto_now_add;type(datetime)"` 只有在创建的时候才添加
        `orm:"auto_now;type(datetime)"` 每次更新都修改
        使用批量修改命令不成功
        type可以是date datetime

    ### 模型关系
     on_delete()  在删除管理关系时候
     set_null   关联对象删除了。设置为空
     do_nothing 关联关系删除了。啥也不做
    #### 1对1关系
        type User struct {                      type Profile{
            Profile *Profile `orm:rel(one)`       User *User `orm:reverse(One)`
            }                                        }                                                                   
    #### 1对多关系
        type User struct {
            Post []*Post  `orm:"reverse(many)"`
        }
        type Post struct {
            User *User `orm:"rel(fk);on_delete(do_nothing)"`  relForeignKey 对应主键
        }
    #### 多对多关系
        type Post struct {
            Tags []*Tag `orm:"rel(m2m);on_delete(set_null)"`
        }
        type Tag struct {
            Posts []*Post `orm:"reverse(many)"`
        }
    ##### 多对多关系通过第三方表实现
        可以通过

    ### 查询
        # 裸查询
        o := NewOrm()
        var r RawSeter
        r = o.Raw("UPDATE user SET name = ? WHERE name = ?", "testing", "slene")
        # 查询
        o := orm.NewOrm()
        // 获取 QuerySeter 对象，user 为表名
        qs := o.QueryTable("user")
        qs.Filter("id", 1)
        // 查询关联对象属性
        qs.Filter("profile__age", 18)
        qs.Filter("profile__age__gt", 18)
        qs.Filter("profile__age__in", 17,18)
        qs.Filter("name__contains", "r")
        qs.Filter("profile_id", nil)  不存在
        qs.Filter("profile_isnull", true)  为空
        // limit
        qs.Limit(10) # 
        qs.Limit(10, 20) # [10,20)
        // offset 
        qs.Offset(10)
        // group by
        qs.GroupBy("id", "age")
        // orderby
        qs.OrderBy("-id") 降续
        qs.OrderBy("id") 升续
        //存在
        qs.Exists()
        // 数量
        qs.Count()
        // distinct
        qs.Distinct("name")
        // 获取多个结果值
        var users []*User
        qs.All(&users)
        //获取一个结果值
        var users User
        qs.One(&user)
        // 指定字段
        qs.One(&user, "Id", "Title")
        # 查询2
            objects := orm.NewOrm()
            user = User{Id: id}
            err = objects.Read(&user)
    ### 事务
    o := NewOrm()
    err := o.Begin()
    // 事务处理过程
    ...
    ...
    // 此过程中的所有使用 o Ormer 对象的查询都在事务处理范围内
    if SomeError {
        err = o.Rollback()
    } else {
        err = o.Commit()
    } 

    ### 切库
        orm.RegisterDataBase("db1", "mysql", "root:root@/orm_db1?charset=utf8")
        orm.RegisterDataBase("db2", "mysql", "root:root@/orm_db2?charset=utf8")
        o1 := orm.NewOrm()
        o1.Using("db1")
        o2 := orm.NewOrm()
        o2.Using("db2")

## 缓存
    安装：go get github.com/astaxie/beego/cache
    使用：
    import (
        "github.com/astaxie/beego/cache"
    )  
    生成实例：
    bm, err := cache.NewCache("memory", `{"interval":60}`)
    使用
    bm.Put("astaxie", 1, 10*time.Second)
    bm.Get("astaxie")
    bm.IsExist("astaxie")
    bm.Delete("astaxie")
    引擎分类：
    1、memory
    bm, err := cache.NewCache("memory", `{"interval":60}`)
    2、file
    bm, err := cache.NewCache("file", `{"CachePath":"./cache","FileSuffix":".cache","DirectoryLevel":2,"EmbedExpiry":120}`)
    3、Redis
    bm, err := cache.NewCache("redis", `{"key":"collectionName","conn":":6039","dbNum":"0","password":"thePassWord"}`)
    4、memcache
    bm, err := cache.NewCache("memcache", `{"conn":"127.0.0.1:11211"`)

    要是自己想实现一个缓存引擎，只需要实现 Cache 接口就行了



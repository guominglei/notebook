# django

## model 和 数据库
### 查询技巧
#### select_related  prefetch_related
    select_related 通过 join 方式。
        把一对一，一对多。相关的数据，一次性的读取出来。减少数据库查询次数。
    prefetch_related 分别查询多个表。然后用python处理关联关系。
    
    sql层面：
        filter 返回符合条件的结果。
        exclude 返回不符合条件的结果
        values 返回指定字段. 数据是字典。
        values_list 返回指定字段。数据是元组 。
    
    Python实例层面加载:
        only  只实例指定字段的数据。
        defer 延缓实例化(Python实例层面加载)指定字段。

### F, Q 方法的区别
    Q: 条件值对象。方便查询条件的生成。
    F: 原始值对象。
        常规：
            item = M.objects.get(id=xx)
            item.num += 1
            item.save()
        F :
            M.objects.get(id=xx).update(num = F('num')+1)

### 读写分离

    db-router
    class AuthRouter(object):
        """
        A router to control all database operations on models in the
        auth application.
        """
        def db_for_read(self, model, **hints):
            """
                根据数据模型的原型 库标签 来判断库
            """
            #
            # if model._meta.app_label in ("stat","stat_cms","mobile_production"):
            #     database = "{}_read".format(model._meta.app_label)
            #     logging.debug("read from {}".format(database))
            #     return database
            # else:
            #     return None
            return "read"
        def db_for_write(self, model, **hints):
            """
            根据数据模型的原型 库标签 来判断库
            """
            # if model._meta.app_label in ("stat","stat_cms","mobile_production"):
            #     database = "{}_write".format(model._meta.app_label)
            #     logging.debug("write from {}".format(database))
            #     return database
            # else:
            #     return
            return "write"

    settings.py 添加
    #DATABASE_ROUTERS = ['Piano.db_router.AuthRouter',]


### 使用数据库命令查询
    1、使用模型
        models.objects.raw("select * from table_name where name=%s", ["rick"])
    
    2、单数据库，或则使用默认default 直接使用connection
        from django.db import connection
        with connection.cursor() as Cursor:
            Cursor.execute("select * from table_name where name=%s", ["rick"])
    
    3、多数据库使用connections
        from django.db import connections
        with connections['db_alias'].cursor() as Cursor:
            Cursor.execute("select * from table_name where name=%s", ["rick"])

#### 模型manage的extra方法
        extra(self, select=None, where=None, params=None, tables=None, order_by=None, select_params=None)
        
        Entry.objects.extra(select={'new_id': "select col from sometable where othercol > %s"}, select_params=(1,))
        
        Entry.objects.extra(where=['headline=%s'], params=['Lennon'])
          
        Entry.objects.extra(where=["foo='a' OR bar = 'a'", "baz = 'a'"])

        Entry.objects.extra(select={'new_id': "select id from tb where id > %s"}, select_params=(1,), order_by=['-nid'])


### 模型变更
    1、makemigrations 根据模型的变动。来生成迁移文件
        python manage.py makemigrations
    2、执行迁移文件
        python manage.py migrate

### 利用旧表生成模型
    1、生成模型
        python manage.py inspectdb
        python manage.py inspectdb > models.py  
        python manage.py inspectdb table-name > models.py
        默认生成的模型 managed=False
    2、 完成迁移
        python manage.py migrate

## view 
    FBV function base view .
        def login(request):
            pass

    CBV class base view
    class Login(view.View):
        def get(self,request, *args, **kwargs):
            pass
        def post(self,request, *args, **kwargs):
            pass

## 国际化
    django-admin makemessages --locale=de --extension xhtml

## 中间件
    中间件定义5个方法：
    process_request  路由分发前处理的
    process_view  路由分发后的，view函数处理前
    process_template_response 有render 模板返回后。
    process_response Response 返回时执行
    precess_exception 有异常处理

    中间件注册顺序。就是调用顺序。由上到下依次处理request, 再由下向上处理response.

## cache

    1、开发调试：django.core.cache.backends.dummy.DummyCache
    2、本地内存：django.core.cache.backends.locmem.LocMemCache
    3、本地文件：django.core.cache.backends.filebased.FileBasedCache
    4、数据库：django.core.cache.backends.db.DatabaseCache
    5、memcached :
        django.core.cache.backends.memcached.MemcachedCache 
        django.core.cache.backends.memcached.PyLibMCCache
    配置：
    BACKEND: 'django.core.cache.backends.memcached.MemcachedCache',
    LOCATION:  [
            ('10.10.10.10:11211', 10),  # 设置远端Memcache服务器IP和端口以及优先级
            ('10.10.10.11:11211', 20),
        ],
    TIMEOUT：缓存的默认过期时间，以秒为单位，默认是300秒。
    OPTIONS：
        MAX_ENTRIES：缓存允许的最大条目数，超过这个数则旧值会被删除，默认是300。
        CULL_FREQUENCY：当达到MAX_ENTRIES 的时候,被删除的条目比率。 实际比率是1/CULL_FREQUENCY，默认是3。
    KEY_PREFIX：缓存key的前缀（默认空）
    VERSION：缓存key的版本（默认1）
    KEY_FUNCTION：生成key的函数（默认函数会生成为：[前缀:版本:key])

    选择合适的缓存策略
    添加 middleware
    MIDDLEWARE = [
        'django.middleware.cache.UpdateCacheMiddleware',    # 更新数据，timeout
        'django.middleware.common.CommonMiddleware',        # 
        'django.middleware.cache.FetchFromCacheMiddleware', # 获取cache
    ]

    view 里添加
        from django.views.decorators.cache import cache_page
        
        @cache_page(60 * 15, key_prefix="site1")
        def my_view(request):
            ...

    template 
        {% load cache %}
        {% cache 500 sidebar request.user.username %}
            .. sidebar for logged in user ..
        {% endcache %}

## 单元test
    from django.test import TestCase
    from app01.models import *
    class AuthorTestCase(TestCase):
        # 测试开始前的工作
        def setUp(self):
            auths = Author.objects.all().values()
            print(auths)

        # 测试结束的收尾工作
        def tearDown(self):
            Author.objects.filter(name="Steven").delete()
            auths = Author.objects.all().values()
            print(auths)

        # 自己定义的测试方法，必须以"test_"开头
        def test_insert_data(self):
            Author.objects.create(name="Steven", hobby="骑行")
            auths = Author.objects.all().values()
            print(auths)
    输出：
    <QuerySet []>
    <QuerySet [{'name': 'Steven', 'id': 1, 'hobby': '骑行'}]>
    <QuerySet []>

    说明：
        1. 对于每一个测试方法都会将setUp()和tearDown()方法执行一遍
        2. Django会在数据库中自动新建一个测试数据库来进行数据库方面的测试，默认在测试完成后销毁。所以不用担心它会影响你实际的生成数据库！

    命令：
        1. 测试项目中所有的应用
        python3 manage.py test
        2. 测试项目中单独的应用
        python3 manage.py test app01
        3. 运行项目中某个应用的测试文件中的一个Case
        python3 manage.py test app01.test2.AuthorTestCase
        4. 运行项目中某个应用的测试文件中的一个Case中的其中一个测试方法
        python3 manage.py test app01.test2.AuthorTestCase.test_insert_data
        5. 运行单元测试结束时不自动删除测试数据库（保留测试数据库）
        python3 manage.py test app01 --keepdb

## 脚本环境

    import sys,os,django
    sys.path.append(os.path.dirname(os.path.dirname(os.path.abspath(__file__)))) #把manage.py所在目录添加到系统目录  
    os.environ['DJANGO_SETTINGS_MODULE'] = 'jcsbms.settings' #设置setting文件  
    django.setup()#初始化Django环境  

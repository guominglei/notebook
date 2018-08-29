# flask

## flasgger
    接口页面测试框架。网页版浏览接口。

## 上下文环境 context 
shell 中使用App 环境
1 、 with app.app_context():
               xx

2、 使用命令。命令中调用App中自己定义的方法。

## url映射
和endpoint 相关的有两个map。
第一个map的作用是根据URL找到endpoint。 
第二个map的作用是根据 endpoint 找到相应的 view 函数

## werkzeug 
    Python wsgi 服务器组件

    服务器处理流程：
    服务器server继承关系
    BaseServer-> TCPServer- > HTTPServer-> BaseWSGIServer

    实例化wsigserver 然后  运行server_forever() 
    server_forever() ：
         1、清空线程事件
         2、处理select 事件
         3、_handle_request_noblock() 

    _handle_request_noblock() ：
    1、socke.accecpt
    2、verify_request
    3、process_request

    process_request
    1、finshi_request(request,client_addr)
    2、 shutdown_reqeust(request)

    finish_request:
         实例化一个wsgirequesthandler

    处理函数继承关系
    BaseRequestHandler-> StreamRequestHandler-> BaseHTTPRequestHandler-> WSGIRequestHandler     

    wsgirequestHandler  实例化方法中调用  setup(). handler() , finish()

     WSGIRequestHandler.handler -> WSGIRequestHandler.handle_one_request -> WSGIRequestHandler.run_wsgi

    run_wsgi():

    1、make_environ
    2、app(environ, start_response)  

    想要开发wsgi 服务器，只需要定义一个app 并且实现 __ call__ 方法即可。
    在flask 中 app.py 中的Flask 类定义中实现了 __call__ 方法 。
    在flask中 RequestContext 实例的match_request 方法 负责URL-> 到 endpoint 的映射工作。

## 多APP
    from werkzeug.wsgi import DispatcherMiddleware
    from biubiu.app import create_app
    from biubiu.admin.app import create_app as create_admin_app
    application = DispatcherMiddleware(create_app(), {
        '/admin': create_admin_app()
    })

### DispatcherMiddleware源码
    根据path来分发。
    class DispatcherMiddleware(object):
        """Allows one to mount middlewares or applications in a WSGI application.
        This is useful if you want to combine multiple WSGI applications::
            app = DispatcherMiddleware(app, {
                '/app2':        app2,
                '/app3':        app3
            })
        """
        def __init__(self, app, mounts=None):
            self.app = app
            self.mounts = mounts or {}
        def __call__(self, environ, start_response):
            script = environ.get('PATH_INFO', '')
            path_info = ''
            while '/' in script:
                if script in self.mounts:
                    app = self.mounts[script]
                    break
                script, last_item = script.rsplit('/', 1)
                path_info = '/%s%s' % (last_item, path_info)
            else:
                app = self.mounts.get(script, self.app)
            original_script_name = environ.get('SCRIPT_NAME', '')
            environ['SCRIPT_NAME'] = original_script_name + script
            environ['PATH_INFO'] = path_info
            return app(environ, start_response)

    为啥要使用stack形式的? 主要是跑脚本和测试用。
    因为多app中。实际中只有一个app可以运行。所以stack里还是只有一个。


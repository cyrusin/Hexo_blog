title: "Tornado与线程池"
date: 2016-04-07 15:30:29
tags: [技术,Python,Tornado]
categories: Tornado
---
Tornado本身的设计目标是单线程异步非阻塞，要想很好的发挥它的性能最好使用异步IO，并且Tornado本身也提供了异步的`AsyncHttpClient`的实现，配合`gen.coroutine`和`yield`，可以让请求异步执行从而不阻塞当前线程，对于单线程服务器来说，`阻塞(blocking)`和`同步的sleep`这种会挂起线程的动作都是服务器的噩梦，因为只有一个线程，所以任何等待都会影响服务器对于其他请求的处理。

异步非阻塞对于第三方IO是http请求的情况还好，毕竟可以使用Tornado提供的异步实现，但是对于有些数据库的IO，则需要异步库的支持，比如针对MongoDB的Motor等。但是第三方异步库的质量也是参差不齐，在实际的工程中，假如没有特别好的异步库的时候，还是只能使用同步的库。

假如只能使用同步阻塞的IO，为了提高性能，可以考虑在Tornado中使用线程池。线程池可以考虑使用`concurrent.futures`里的`ThreadPoolExecutor`。

1. 在RequestHandler的http请求处理方法(get/post等)中使用线程池
=======
线程池为RequestHandler持有，请求处理逻辑中的耗时/阻塞任务可以提交给线程池处理，主循环逻辑可以继续处理其他请求，线程池内的任务处理完毕后，会通过回调注册callback到ioloop，ioloop可以通过执行callback恢复挂起的请求处理逻辑。

    
    import tornado.web
    import tornado.gen
    from tornado.concurrent import run_on_executor
    from concurrent.futures import ThreadPoolExecutor
    class HasBlockTaskHandler(tornado.web.RequestHandler):
        executor = ThreadPoolExecutor(4) #起线程池，由当前RequestHandler持有
        @tornado.gen.coroutine
        def get(self):
            ...
            result = yield self.block_task() #block_task将提交给线程池运行
            ... #继续处理
            self.write(response)
        @run_on_executor
        def block_task(self):
            time.sleep(5) #也可能是其他耗时／阻塞型任务
            return result #直接return结果即可

2. 项目使用全局线程池，将http请求处理方法(get/post等)整个托管给线程池
========
假如某个RequestHandler中的http method中有可能导致主处理逻辑阻塞的任务，直接将该http method整个托管给线程池执行。

   
    import functools
    import tornado.ioloop
    import tornado.web
    from concurrent.futures import ThreadPoolExecutor
    EXECUTOR = ThreadPoolExecutor(max_workers=4)#全局线程池
    def unblock(http_method):
        #必须添加该装饰器，表明当前方法结束后，并不finish该请求
        #Tornado请求执行的流程默认是: initialize()->prepare()->http_method(get/post等)->finish()
        #当用unblock装饰器装饰后，http_method实际是执行下面的_wrapper()方法，在_wrapper中我们只是将原始的
        #http_method提交给线程池处理，所以还没有执行完该http_method，所以还不能finish该请求
        @tornado.web.asynchronous 
        @functools.wraps(http_method)
        def _wrapper(self, *args, **kwargs):
            #以下的callback必须在主线程执行
            #self.write(),self.finish()等都不是线程安全的
            def callback(future):
                self.write(future.result())
                self.finish()
            _future = EXECUTOR.submit(
                functools.partial(http_method, self, *args, **kwargs)  
            )
            tornado.ioloop.IOLoop.instance().add_future(_future, callback)
        return _wrapper
    class BlockHandler(tornado.web.RequestHandler):
        @unblock
        def get(self): #该方法将被提交到线程池中运行
            ...
            #直接return，该结果即future.result(), 后续将被self.write(result)
            #不要在子线程中执行self.write(),因为这并非线程安全的方法
            #通过ioloop.IOLoop.instance().add_callback的方式，将其交给主线程执行
            ＃ioloop提供的add_callback是线程安全的
            return result

3. GIL的问题
=====
GIL使得Python多线程无法充分利用多核，并且同一时刻只有一个线程工作，那用多线程还有什么意义？这里面其实有一个问题需要注意：web服务通常是IO密集型的，当我们使用线程池的时候，其实大多都是在有同步阻塞的IO任务且没有很好的异步库的时候使用，GIL在当前线程被阻塞在IO任务上时，是可以被释放从而给其他线程运行的机会的，所以使用线程池还是可以大大的提升性能。

总之，在单线程Tornado中使用同步阻塞的IO是一个需要认真对待的问题，对于单个进程来说，同步阻塞的IO意味着当前服务进程（对Tornado来说其实就是主线程）对于IO异常情况（比如有某个第三方请求响应超慢）的承受能力很差，一个请求慢，其后所有的请求都会滞后。但异步或线程池就不会出现这种情况。

4. 参考
====
1. [使用Tornado让你的请求异步非阻塞](http://www.dongwm.com/archives/shi-yong-tornadorang-ni-de-qing-qiu-yi-bu-fei-zu-sai/)
2. [Blocking tasks in Tornado](https://lbolla.info/blog/2013/01/22/blocking-tornado)
3. [Tornado的多线程封装](https://github.com/nikoloss/iceworld)


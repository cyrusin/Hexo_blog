title: "两种协程的调度方式: trampoline、thread scheduler"
date: 2015-11-06 15:48:26
tags: [Python,技术]
categories: Python
---
协程是用户态内的,或者准确点说是线程内部的一种上下文切换技术,由于协程切换是在用户态下完成的,所以省去了线程切换时频繁出入内核态的资源开销,可以形成一种很高效的协作式并发技术。

这个简短的视频介绍了一些有关协程、并发之类的东西,很有意义。

[Coroutines, event loops, and the history of Python generators](https://www.youtube.com/watch?v=b7R3-_ViNxk)

从里面学习到两种很好的协程的调度方式。把代码拿过来分享一下。

##Coroutine trampoline
这种方式下的协程调度比较好理解,就是从一个初始状态开始,一条执行线索不断的在多个协程之间切换,就好像多个协程协作完成一项任务。
代码:

    def co_trampoline(generators, start, init_data):
        coroutines = dict() #上下文映射,其实就是从名字到协程对象的映射
        for g in generators:
            if g is start: #初始协程
                coroutine = g(init_data)
                #curr, data => 当前需要切换到的协程, 传给协程的数据
                curr, data = coroutine.next()
            else:
                coroutine = g()
                coroutine.next()
                coroutines[coroutine.__name__] = coroutine
    
        while True: #一定是循环的执行
            try:
                #每个协程抛出下一个将要执行的协程和数据,相当于上下文切换
                curr, data = coroutines[curr].send(data)
            except StopIteration:
                break

##Thread scheduler
这种方式类似一种用协程去模拟多个线程的并发模式,所以叫"Thread scheduler"。
代码:

    from collections import defaultdict
    def thread_scheduler(generators):
        #协程要消费的数据,比如实际应用时可以是一个任务队列、消息队列之类的
        thread_data = defaultdict(list)
        threads = [g() for g in generators]
        while True:
            try:
                for t in threads:
                    data = thread_data[t.__name__]
                    #协程t(模拟一个"线程")执行,并返回t下一步的协程和其数据
                    consumer, data = t.send(data)
                    thread_data[consumer].append(data) #任务队列增加,等待调度
            except StopIteration:
                break

假如我的`generators`里有三个协程`c1`, `c2`, `c3`, 那么以上代码的执行顺序就是`c1->c2->c3->c1->c2->c3...`, 不停的循环执行,每一个协程内部可以根据处理的过程不同,返回不同的下一步的协程。就像一个带状态转移的有限状态机一样。

##协程的实现
之前说了协程像一个状态转移的有限状态机,其实现大致类似下面的样子,上面说的两种协程调度方式都可以依赖这种协程的实现方式:

    def coroutine():
        while True:
            state, somedata = process(data) 
            if state == 1:
                data = yield ('c1', somedata)
            elif state == 2:
                data = yield ('c2', somedata)
            else:
                break
            ...



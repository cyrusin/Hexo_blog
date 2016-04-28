title: "Python多线程join()的实现细节"
date: 2016-04-28 14:44:24
tags: [Python,技术]
categories: Python 
---
写Python多线程都知道当前线程调用`a.join()`后，会阻塞直到线程a运行结束，看了一下`threading`模块的源码,
了解了一下实现的原理。

每一个新开启的线程内部都维护着一个`Condition`类型的条件变量，对线程a进行`join()`，其实是`wait()`在线程a
内部的条件变量上，当线程a执行结束时，会通过`notify_all()`通知所有`join()`的线程，则阻塞的线程被唤醒，恢复执行。

以下是源码:

    self.__block = Condition(Lock()) #线程内部维护的Contition变量
    def __stop(self):
        if not hasattr(self, '_Thread__block'):
            return
        self.__block.acquire()
        self.__stopped = True
        self.__block.notify_all() #唤醒所有join()的线程
        self.__block.release()
    def join(self, timeout=None): #可以添加timeout
        ...
        self.__block.acquire()
        try:
            if timeout is None:
                while not self.__stopped:
                    self.__block.wait() #等待线程运行结束
        else:
            deadline = _time() + timeout
            while not self.__stopped:
                delay = deadline - _time()
                if delay <= 0:
                    break
                self.__block.wait(delay)
        finally:
            self.__block.release()

由于是`Condition`，所以线程可以被`join()`多次，最终都会由`notify_all()`唤醒。

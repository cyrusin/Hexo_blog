title: "ioloop的blocking_signal"
date: 2016-08-04 16:07:10
tags: [Python, Tornado]
categories: Tornado
---
这里的blocking signal里的blocking并不是传统意义上的针对IO的blocking, 尽管这可能是引起ioloop阻塞的一个原因之一。在这里，blocking指的是ioloop在epoll返回之后开始依次处理各监听文件句柄上的IO事件时，直到下一次进入epoll调用的这段时间的ioloop的状态。我们知道Tornado是单线程的，在处理完某次epoll调用返回的读写就绪事件之前，Tornado无法启动下次epoll监听，所以这段时间理论上是越短越好，这样，ioloop可以充分及时的获取就绪文件句柄，不会影响整体IO性能。然而在实际的使用过程中，难免会出现某次处理时间过长，从而导致ioloop的blocking时间过长的现象。此时，假如有大量连接到达或者有多个IO事件等待处理，ioloop是不能及时获取的，进而会影响性能。

Tornado在ioloop中给我们提供了针对blocking_signal的处理方法，通过singal实现软中断，当ioloop的blocking时间达到阈值时，允许我们注册信号处理程序，我们可以借此实现一些额外的功能，比如当blocking超过一定时间时，log出当前调用栈，这样后续可以分析出到底哪块儿调用耗费了太多时间。

使用这个功能时，可以在ioloop启动之前针对ioloop的实例调用`set_blocking_signal_threshold`方法，这个方法的实现是:

    def set_blocking_signal_threshold(self, seconds, action):
        if not hasattr(signal, "setitimer"):
            gen_log.error("set_blocking_signal_threshold requires a signal module "
                          "with the setitimer method")
            return
        self._blocking_signal_threshold = seconds
        if seconds is not None:
            signal.signal(signal.SIGALRM,
                          action if action is not None else signal.SIG_DFL)

从代码可以看出，主要是注册了针对signal.SIGALRM信号的处理程序，这个程序有我们自己来定义。以log出当前调用栈为例:

    def log_stack(self, signal, frame):
        """Signal handler to log the stack trace of the current thread.

        For use with `set_blocking_signal_threshold`.
        """
        gen_log.warning('IOLoop blocked for %f seconds in\n%s',
                        self._blocking_signal_threshold,
                        ''.join(traceback.format_stack(frame)))

即标准的信号处理程序的定义方法，log出当前调用栈的话，就直接记日志就行。

Tornado是如何在ioloop中触发signal的，看ioloop的`start`方法的实现:

    def start(self):
       ...
       try:
            while True:
            ...
            if self._blocking_signal_threshold is not None:
            # clear alarm so it doesn't fire while poll is waiting for events.
            signal.setitimer(signal.ITIMER_REAL, 0, 0) # 1
            try:
                event_pairs = self._impl.poll(poll_timeout)
            except Exception as e: 
                ...
            if self._blocking_signal_threshold is not None:
                signal.setitimer(signal.ITIMER_REAL, self._blocking_signal_threshold, 0) # 2
            #处理event_pairs
        except:
            ....

ioloop的`start`方法的大致框架如上所示，可以看出其实现技巧和注意点是在调用epoll之前，务必将timer清除，防止signal的软中断错误的打断epoll对IO事件的监听，然后在epoll返回之后，假如之前有调用过`set_blocking_signal_threshold`方法的话，`_blocking_signal_threshold`不为None，则会设置`signal.setitimer`这个定时器。而这个`signal.ITIMER_REAL`对应的定时器则会在`self._blocking_signal_threshold`时间(假如IO事件的处理到这个阈值还没结束)之后发出`signal.SIGALRM`信号。我们注册的signal.SIGALRM的信号处理程序就会被调用。

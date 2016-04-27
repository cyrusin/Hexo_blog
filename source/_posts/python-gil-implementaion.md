title: "GIL的实现细节"
date: 2016-04-27 15:01:23
tags: [Python,技术]
categories: Python 
---
GIL
===
熟悉Python的人对GIL这货可定都不陌生, **全局解释器锁(Global Interpreter Lock)**简称**GIL**, 这货是
Python多线程的核心机制。由于Python的线程实际是操作系统的原生线程,多个线程同时执行同一段字节码可能
会导致很多问题(比如: 内存管理的引用计数需要线程安全机制的保护),于是使用GIL这把大锁锁住其他线程,保
证同一时刻只有一个线程可以解释执行字节码。关于GIL的更多分析,
可以看[David Beazley大神的研究](http://www.dabeaz.com/GIL/)。本文主要分析下**CPython**的GIL在**Linux**上
基于**pthread**的实现细节,看完这些源码后能够对GIL有更深入的理解。

GIL的定义
===
有人可能会想,从GIL的介绍来看不就是一个互斥锁吗?有什么好分析的,直接pthread_mutex_lock()/pthread_mutex_unlock()
不就行了,其实不然,pthread原生的mutex lock是不足以实现我们的要求的,GIL的目的是让多个线程按一定条件并发执行
而非简单互斥,CPython源码的注释也说了:

    /* A pthread mutex isn't sufficient to model the Python lock type
     * because, according to Draft 5 of the docs (P1003.4a/D5), both of the
     * following are undefined:
     *  -> a thread tries to lock a mutex it already has locked
     *  -> a thread tries to unlock a mutex locked by a different thread
     * pthread mutexes are designed for serializing threads over short pieces
     * of code anyway, so wouldn't be an appropriate implementation of
     * Python's locks regardless.
     *
     * The pthread_lock struct implements a Python lock as a "locked?" bit
     * and a <condition, mutex> pair.  In general, if the bit can be acquired
     * instantly, it is, else the pair is used to block the thread until the
     * bit is cleared.     9 May 1994 tim@ksr.com
     */
    来自CPython源码(Python/thread_pthread.h), Python版本: 2.7.8

从以上注释来看, 由于Linux上的POSIX线程实现有些未定义行为, 并且mutex lock只适用于同步线程对于小段临界
区代码的访问，所以并不适合作为GIL的原生实现。而Python的GIL实际是一个`<condition, mutex>`对, 并用这个
条件变量和互斥锁来保护一个`locked`状态变量。下面来看GIL的真面目:

    typedef struct {
        char             locked; /* 0=unlocked, 1=locked */
        /* a <cond, mutex> pair to handle an acquire of a locked lock */
        pthread_cond_t   lock_released;
        pthread_mutex_t  mut;
    } pthread_lock;

以上这个结构体就是GIL的定义, 可以看出, `locked`用来指示是否上锁, `1`表示已有线程上锁, `0`表示锁空闲。
而`lock_released`和`mutex`来同步对`locked`的访问。

GIL的获取与释放
===
从GIL的定义来看,线程对GIL的操作本质上就是通过修改`locked`状态变量来获取或释放GIL。所以主要的操作有两个:
1. PyThread_acquire_lock()
2. PyThread_release_lock()

下面分别看看起具体实现:

    /*获取GIL*/
    int  PyThread_acquire_lock(PyThread_type_lock lock, int waitflag) 
    {
        int success;
        pthread_lock *thelock = (pthread_lock *)lock;
        int status, error = 0;
        status = pthread_mutex_lock( &thelock->mut ); /*先获取mutex, 获得操作locked变量的权限*/
        success = thelock->locked == 0;
        if ( !success && waitflag ) { /*已有线程上锁,*/
            while ( thelock->locked ) {
                /*通过pthread_cond_wait等待持有锁的线程释放锁*/
                status = pthread_cond_wait(&thelock->lock_released,
                                           &thelock->mut);
            }
            success = 1;
        }
        if (success) thelock->locked = 1; /*当前线程上锁*/
        status = pthread_mutex_unlock( &thelock->mut ); /*解锁mutex, 让其他线程有机会进入临界区等待上锁*/
        if (error) success = 0;
        return success; 
    }

以上就是获取GIL的实现,可以看到,线程在其他线程已经获取GIL的时候,需要通过pthread_cond_wait()等待获取GIL的线程释放GIL。

    /*释放GIL*/
    void PyThread_release_lock(PyThread_type_lock lock)
    {
        pthread_lock *thelock = (pthread_lock *)lock;
        int status, error = 0;
        status = pthread_mutex_lock( &thelock->mut ); /*通过mutex获取操作locked变量的权限*/
        thelock->locked = 0; /*实际的释放GIL的操作, 就是将locked置为0*/
        status = pthread_mutex_unlock( &thelock->mut ); /*解锁mutex*/
        status = pthread_cond_signal( &thelock->lock_released ); /*这步非常关键, 通知其他线程当前线程已经释放GIL*/
    }

以上就是释放GIL的过程,特别注意最后一步, 通过pthread_cond_signal()通知其他等待(pthread_cond_wait())释放GIL的线程, 
让这些线程可以获取GIL。

获取与释放GIL的时机
===
分析了GIL的获取与释放的实现机制后,我们来看看CPython解释器会在什么时候获取与释放GIL。我们知道,
GIL是用来同步多线程使得同一时刻只有一个线程可以解释执行字节码的,显然多线程下,一个线程执行一段
时间之后就要释放GIL让其他线程有执行的机会,而且从获取与释放GIL的实现来看,只有持有GIL的线程主动
释放GIL,其他线程才有机会获取GIL执行自己的任务。那么到底多长时间之后会释放GIL呢？

首先我们来了解一下CPython解释器解释执行字节码的主循环,这个主循环位于`Python/ceval.c`文件的
函数`PyEval_EvalFrameEx()`内。这个函数的大体结构是:

    {
        ...
        for (;;) { /*解释器主循环*/
            ...
            /*_Py_Ticker是个数字, 比如100, 这段代码使得大概每100次才会进入if内的代码*/
            if (--_Py_Ticker < 0) { 
                ...
                _Py_Ticker = _Py_CheckInterval;
                ...
                if (interpreter_lock) { /*开始进行释放GIL与获取GIL的操作*/
                    ...
                    PyThread_release_lock(interpreter_lock); /*释放GIL*/
                    /*获取GIL, 同时注意到这个操作是紧跟着释放GIL的, 
                    这个使得常常当前线程释放了GIL,紧接着又重新获取了GIL*/
                    PyThread_acquire_lock(interpreter_lock, 1); 
                }
            }
            ...
            switch (opcode) { /*opcode就是字节码指令*/
                case: ...
            }
        }
    }

以上代码给出了解释器主循环的代码的整体结构,可以看出,在一个大的循环中逐个解析字节码指令,但是
需要注意的是每次循环开始都会检查一下`_Py_Ticker`的值,这个值可以通过:

    python -c 'import sys;print sys.getcheckinterval()'

查看,默认是100,这个值可以认为是执行的字节码条数的一个计数器,严格上来说应该并不完全等于字节码
条数,可以但是可以这么理解,就是一个当前线程执行了多久的指示器。可以看出,这个数字在执行字节码的
过程中是递减的,而每次进入一条新的字节码之前都会检查这个数字,当这个数字小于0的时候,就开始进入if
块内部的代码,而内部的代码会释放GIL。所以,这个周期性的计数器小于0就是我们释放GIL的时机之一。

Python释放GIL的时机之一:
>有一个周期性计数的计数器,不断递减,保证Python线程可以在执行一段时间之后释放GIL

仔细分析就会发现问题,假如在解析执行字节码的过程中当前线程遇到了一个IO操作,却由于等待数据而被阻塞到了
该IO操作上,由于只有主动释放GIL,其他线程才有机会运行,那这样显然是白白浪费了时间,从GIL的设计来看,当当前
线程阻塞在IO操作上时,此时给其他线程运行的机会并没有什么问题,因为GIL只是用来同步线程执行字节码的,并非
一般的互斥共享资源的互斥锁。在阻塞操作之前让出GIL,其他线程可以继续执行,而当前线程可以继续执行阻塞型的
操作,当该阻塞型的操作完成之后,再次试图获取GIL,继续执行余下的字节码。Python的设计者已经考虑到了这样的问题:

    /* Interface for threads.
    A module that plans to do a blocking system call (or something else
     that lasts a long time and doesn't touch Python data) can allow other
      threads to run as follows:
        ...preparations here...
        Py_BEGIN_ALLOW_THREADS
        ...blocking system call here...
        Py_END_ALLOW_THREADS
        ...interpret result here...
    */
    [Python/ceval.h]

从以上注释来看,Python允许在执行block型的system call(或者其他不会操作Python Data的调用)之前allow其他
线程执行(Py_BEGIN_ALLOW_THREADS),完事儿后,再重新尝试获取GIL(Py_END_ALLOW_THREADS)。
Py_BEGIN_ALLOW_THREADS和Py_END_ALLOW_THREADS是两个宏,其完整定义是:

    #define Py_BEGIN_ALLOW_THREADS { \
                            PyThreadState *_save; \
                            _save = PyEval_SaveThread();

    #define Py_END_ALLOW_THREADS    PyEval_RestoreThread(_save); \
                    }

而PyEval_SaveThread()和PyEval_RestoreThread()里会分别释放GIL和获取GIL。看一个Python实现中的具体应用的例子:

    static PyObject * file_read(...)
    {
        ...
        for(;;)
        {
            ...
            FILE_BEGIN_ALLOW_THREADS(f) //释放GIL
            errno = 0;
            chunksize = Py_UniversalNewlineFread(BUF(v) + bytesread,
            buffersize - bytesread, f->f_fp, (PyObject *)f);
            interrupted = ferror(f->f_fp) && errno == EINTR;
            FILE_END_ALLOW_THREADS(f) //获取GIL
            ...
        }
        ...
    }

`file_read`是`f.read()`对应的C的实现。可以看到代码里在实际进行读操作之前会尝试释放GIL。

由此可以看出,Python释放GIL的第二个时机:
>在IO操作等可能会引起阻塞的system call之前,可以暂时释放GIL,但在执行完毕后,必须重新获取GIL

总结
===
1. 单核机器上,GIL是比较好的多线程并发的机制,但在多核机器上,GIL使得**计算密集型**Python应用无法充分利用多核;
2. IO密集型多线程Python应用可以通过GIL获得良好的性能,因为在IO操作时可以暂时释放GIL;
3. 通常情况下,Python通过周期性的check决定是否释放GIL;
4. 在通过C扩展Python时,可以谨慎处理GIL,在不操作Python原生对象的时候,可以尝试Py_BEGIN_ALLOW_THREADS和Py_END_ALLOW_THREADS。

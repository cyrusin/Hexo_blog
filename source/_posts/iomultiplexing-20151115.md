title: "Linux的IO多路复用: select/poll/epoll"
date: 2015-11-15 16:31:10
tags: [网络IO, 技术, 网络编程, Linux]
categories: 服务器
---
Linux里的IO多路复用是有效提高IO效率的技术。主要有select、poll、epoll三种。

select
======
select调用的函数接口是:

    int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);

参数说明:

> nfds: fdset中最大描述符值加1, fdset是一个位数组, 大小为__FD_SETSIZE(1024), 位数组的每一位表示该描述符是否被检查
> readfds, writefds, exceptfds: 三个位数组, 非别对应监听不同类型读写及错误事件的描述符。可被内核修改用于通知相应事件
> timeout: 超时时间, 可被内核修改, 则其值是超时剩余时间

基本原理:

> 用户进程调用select后阻塞, select对应内核态的sys_select, 将readfds, writefds, exceptfds对应的fd_set拷贝到内核, 内核通过轮询的方式检查注册的描述符, 对于事件发生的描述符记录到临时fd_set, 最终会拷贝到用户进程空间，轮询完毕后返回。
> 没有事件发生时, 睡眠到超时返回。然后进行下次轮询
> select对于监视的描述符有数量限制, __FD_SETSIZE(1024)个

poll
=====
poll的函数接口:

    int poll(struct pollfd *fds, nfds_t fds, int timeout);

    struct pollfd {
        int fd; /*描述符*/
        short events; /*关注的事件*/
        short revents; /*内核报告的实际发生的事件*/
    };

参数说明:

> fds: 结构体数组, 每一个结构体表示一个描述符及该描述符上的关注事件、发生的事件

基本原理:

> 用户进程向内核传递fds数组, 不像select一样有描述符数目限制
> poll对应的是内核的sys_poll, 依然是对fds数组中的每个pollfd进行poll, 检查其事件发生, 修改revents字段报告该事件
> poll返回后, 逐一检查fds数组中每个pollfd的revents

epoll
=====
epoll的函数接口:

    int epoll_create(int size);
    int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
    int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);

    typedef union epoll_data {
        void *ptr;
        int fd;
        __uint32_t u32;
        __uint64_t u64;
    }epoll_data_t;

    typedef struct epoll_event {
        __uint32_t events;
        epoll_data data;
    };

参数说明:

> size: epoll并不限制监视的文件描述符数目, 这只是对内核的初始化建议
> events: epoll_wait的第二个参数用于存放结果

基本原理:

> 用户进程通过epoll_create创建epoll描述符, 返回的int型就是epoll描述符的值, -1表示创建失败
> 通过epoll_ctl为某个fd进行添加、删除、修改事件, 即op为EPOLL_CTL_ADD, EPOLL_CTL_DEL, EPOLL_CTL_MOD, 关联的事件为event
> epoll_wait等待io事件, 该调用会返回活跃描述符(发生了监听事件的描述符)

epoll相对于select和poll的优点
====

1. 相对于select, epoll监视描述符没有数量限制, 理论上是你的进程可以打开的描述符的最大数目;
2. IO效率并不随监视描述符数目增加而明显下降, **epoll并不像select和poll那样轮询**, 而是**通过给每个监视的文件描述符注册回调函数**实现, 当某个监视的文件描述符上发生了监视的事件后, 触发回调将自己写入结果中, 这样每次epoll返回的总是活跃的文件描述符集, 避免了在select和poll的使用中用户依然需要逐一检查描述符导致的性能下降;
3. 对监视事件有电平触发和边沿触发两种触发方式, 其中边沿触发的性能极高, 只是应用逻辑会稍复杂;
4. 内核和用户进程通过共享内存的方式传递信息, 主要是内核和用户空间通过Linux的mmap映射同一块内存, 这样减少了多次内存拷贝的性能开销。

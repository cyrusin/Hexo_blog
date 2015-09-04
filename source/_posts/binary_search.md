title: "二分查找思想搞定轮播视频匹配问题"
date: 2015-08-25 14:49:00
tags: [算法,项目,二分查找]
categories: 技术
---

之前两个月的时间全组投入到了我们自己的TV launcher的开发, 这算是比较重要的项目, 涉及到公司未来OTT的战略性产品, 所以重视程度比较高。
我负责了轮播服务的后端开发, 当然整个技术栈还是以Tornado/Redis/MongoDB为主。经常听到兄弟们抱怨学习了那么多算法, 进BAT也得面试算法什么的, 有个啥子用, 撸了这么多年代码, 用不到啥算法, 不也照样撸的风声水起吗。哥下面就小小展示一下, 其实算法这东西是有用的, 你带着算法和数据结构的一些思想思考问题, 和你仅仅凭经验写东西, 结果是很不一样的。

所谓轮播:指的是像电视台那样的一天24小时的滚动播放,由服务端生成播放列表(频道\频道的视频列表等), 并计算好起始时间,用户每次打开Launcher时, 客户端请求频道的当前播放视频列表, 服务端会返回一定数量的视频列表给客户端, 这些视频每一个都有begin_time和end\_time标记起始时间, 同时服务端返一个current\_time的当前时间戳, 客户端匹配播放当前播放的视频。

我们用MongoDB做持久化存储, 服务端会通过定时任务生成每个频道的某一时间段的视频播放列表, 并为每个视频信息标记begin_time和end_time。考虑到这种列表的东西很适合缓存到Redis, 所以我们每次会把生成的视频按begin_time排序将这个列表整个塞到Redis, 构成一个Redis的List。这样的话问题简化成了: 每次客户端来请求某一频道的当前播放视频列表(比如Redis中的List长为500个, 客户端每次要当前时间点前20后30共51个), 我们从Redis中找到该频道对应的视频列表, 查询当前播放视频, 并返回当前视频的前后视频组成的视频列表。

典型的**查找问题**:
>    输入: `current_time(UNIX timestamp)`
>    待查找的列表: `有序的Redis列表, 每一个元素都是一个视频信息的JSON串`
>    输出: `满足begin_time <= current_time < end_time的在列表中的索引及反序列化的视频信息`

与纯二分查找的问题略有差异, 但基本思想确都是一致的。其实一开始的时候有想法写成顺序查找的版本, 但是后来分析一下, 很明显顺序查找的方法会随着时间推移, 列表末尾的那些视频会慢死你。

代码如下:
    
    def binary_search_t(L, cur_time, trans_func=json.loads):
        '''binary_search_t(list, timestamp, function) -> index, transfered_tartget_element

        use binary search algorithm to find target element in sorted list,
        each element needs to be transfered to sortable object within 'trans_func'.
        'L': list contains some elements that sorted by some attribute(e.g. 'begin_time')
        'cur_time': target value(current timestamp)
        'trans_func': function used to transfer element to be sortable, default is json.loads
        e.g. L=["{'begin_time':...,'end_time':...}","{...}"...]
        '''
        if not L:
            return -1, None

        low = 0
        high = len(L) - 1
        while low <= high:
            mid = (low + high) >> 1
            try:
                mid_val = trans_func(L[mid])
                begin_time = mid_val.get('begin_time')
                end_time = mid_val.get('end_time')
            except Exception, e:
                logging.exception(e)
                #存在非法数据,停止搜索
                return -1, None

            if begin_time <= cur_time:
                if end_time > cur_time:
                    return mid, mid_val
                else:
                    low = mid + 1
            else:
                high = mid - 1
        return -1, None 



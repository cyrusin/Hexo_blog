title: "利用Redis作周期性访问失败计数功能"
date: 2016-03-29 16:46:39
tags: [技术,Redis,Python]
categories: Redis
---
每次进入应用客户端时，都需要进行后端鉴权服务，接口会调用某牌照方的鉴权接口，根据用户的MAC地址决定用户是否有权限登陆使用服务。由于调用的接口不是很稳定，有时会出现连续一段时间误判，导致终端大量用户无法使用APP，所以决定在接口这边做一个策略：

>统计一段时间内的第三方鉴权接口鉴权失败数量，当超过某一阈值时，接口暂时对用户请求返回成功。

由于只是周期性的计数，比如十分钟，所以当第三方服务异常，连续大量用户请求失败的时候，只要同时做好监控报警工作，及时上报给第三方，同时，并不影响用户使用服务。第三方发现后可以及时处理，处理正常后，接口又可以继续以牌照方的响应为准，所以基本也不违背广电总局可管可控的原则。

这种周期性计数功能，使用Redis最好不过。

方案是使用Redis的`INCR`命令作自增，同时注意“周期性”这个限制是通过`EXPIRE`命令控制key的有效期，但需注意，使用事务保证命令序列的原子性，防止出现`EXPIRE`命令执行失败，导致缓存的数据永久有效的现象。

实际使用时，在缓存的key上使用一定的技巧，如：统计十分钟内的数据，在生成key的时候，用当前时间（以秒为单位）除以600（十分钟），把结果拼到key里，这样可以保证：即使Redis事务执行都没有成功， 依然可以每十分钟更新一次计数器，使得当第三方接口恢复正常后，用户的请求依然以第三方接口的响应为准。

以下是代码：

    current_time = int(time.time())
    #COUNTER_INTERVAL_S是计数周期
    #当前时间除以计数周期, 保证计数周期的有效性
    time_tick = current_time / COUNTER_INTERVAL_S
    key = ''.join(['counter:device_login:failed:', str(time_tick)])
    redis_client = redis_cache.client()
    failed_num = redis_client.get(key)
    if failed_num and int(failed_num) > MAX_FAILED: #超过阈值, 直接返回默认结果
        return self.write(default_content)
    else: #没有超过阈值, 计数器自增
        try:
            #注意使用事务
            p = redis_client.pipeline()
            p.incr(key, 1)
            p.expire(key, COUNTER_INTERVAL_S)
            status = p.execute()[0]
        except Exception, e:
            logging.exception(e)
        return self.write(content)


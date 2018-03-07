---
title: python学习笔记-redis multi watch实现锁库存
tags:
    - 电商
categories:
    - python
date: 2016-05-11 00:52:32
---

python 关于redis的基本操作网上已经很多了，这里主要介绍点个人觉得有意思的内容1.redis的事务操作以及watch 乐观锁；后面描述2.tornado下异步使用redis的方式
     
redis是单进程单线程模型，本身应对外部请求的是单任务的，也是多线程安全的，这个大家都应该知道的， 所以才会经常有人用redis做计数服务。
     
首先redis 的事务处理只能使用pipeline：In redis-py MULTI and EXEC can only be used through a Pipeline object.
http://stackoverflow.com/questions/31769163/what-are-equivalent-functions-of-multi-and-exec-commands-in-redis-py
 　　
很多人喜欢在处理电商库存，例如秒杀活动的时候用redis做计数器，不过下面这段代码采用了事务来控制库存，是不是有点问题呢？
```
if __name__ == "__main__":
    with sms_redis.pipeline() as pipe:
        while 1:
            try:
                # 事务开始
                pipe.multi()
                count = int(sms_redis.get('stock_count'))
                if count > 0:  # 有库存
                    pipe.set('stock_count', count - 1)
                # 事务结束
                pipe.execute()
                # 把命令推送过去
                break
            except Exception:
                traceback.print_exc()
                continue
```
问题在于误以为pipe.multi() 事务处理执行后，就可以锁住该库存，这里强调下：
redis 本身没有悲观锁的概念，也就是说对于客户的是无法锁住redis中的值的，当然你可以通过其它曲径实现，比如SETNX， 这里先不谈；
redis 的事务可以认为是一段命令的批量执行和执行结果一次性返还，事务操作本身没有问题，执行过程不会中断，但是在pipe.execute() 的时候事务才真正向redis_server 提交，但是很遗憾在redis_server 执行之前库存都有机会被其它客户端修改，完全起不到锁库存概念；
 
那么如何才能实现锁库存呢？watch是一个很好的解决方案：

watch 字面就是监视的意思，这里可以看做为数据库中乐观锁的概念，谁都可以读，谁都可以修改，但是修改的人必须保证自己watch的数据没有被别人修改过，否则就修改失败了；
 
```
if __name__ == "__main__":
    with sms_redis.pipeline() as pipe:
        while 1:
            try:
                #关注一个key
                pipe.watch('stock_count’)
                count = int(pipe.get('stock_count'))
                if count > 0:  # 有库存
                    # 事务开始
                    pipe.multi()
                    pipe.set('stock_count', count - 1)
                    # 事务结束
                    pipe.execute()
                    # 把命令推送过去
                break
            except Exception:
                traceback.print_exc()
                continue
```
　
如果在watch后值被修改，在执行pipe.execute()的时候会报异常WatchError: Watched variable changed.
这里可以简单的实现基于redis中的锁库存的逻辑。
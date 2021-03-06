---
title: redis原理分析
tags:
    - redis
categories:
    - 运维
date: 2016-04-11 23:43:08
---

edis的使用大家都很熟悉，可能除了watch 锁，pipeline，订阅发布用的少点，不过网上也有大量的教材和例子，这里想聊聊redis中的一些原理。
 
### redis 提供了两种持久化方式，一种是RDB，一种是AOF；
RDB 是指在制定的时间间隔生成数据集的快照，
AOF持久化记录服务器执行的所有写命令，并在服务器重启时，重新执行这些命令来恢复数据
 
### redis 启动过程
-（1）初始化服务器变量，设置服务器默认配置
-（2）读取配置文件的配置，覆盖默认配置
-（3）初始化服务器功能模块
-（4）从RDB或者AOF中重载数据
-（5）网络监听前的准备工作
-（6）开启事件监听，循环接受客户端请求

![](https://images2015.cnblogs.com/blog/564050/201611/564050-20161120194610342-913835037.png)

### redis持久化方案分析
#### RDB持久化
将内存中的数据快照保持到磁盘中，redis重启的时候重新载入RDB文件来还原数据库。
a）RDB 保存
 如果已经存在RDB文件，会用新文件替换旧文件， 在保存RDB的过程中，redis主进程阻塞，无法响应客户端请求。
为了避免阻塞主线程，redis提供了rdbSaveBackground函数，新建子进程调用RDB save，完成RDB保存后再发生消息通知主进程。在此期间主进程可以继续处理新的客户端请求。
b）RDB读取
redis启动的时候会根据配置的持久化模式，决定是否读取RDB文件，并将其保存到内存中。
在此过程中，每载入1000个key，就处理一次已经等待处理的客户端请求，不过目前只能处理订阅功能的命令， 其他一律返回错误信息。因为发布订阅功能不需要写入数据，不需要保存在redis数据库中
 
缺点：
1.保存频率过低，宕机时会导致数据丢失
2.保存频率过高，可能由于数据集过大导致操作耗时，短时间无法处理客户端请求
 
#### AOF持久化
将所有的写命令记录下来，达到记录数据库状态的目的
- 保存
（1）将客户端请求的命令转化为网络协议格式
（2）将协议内容字符串追加到变量server.aof_buf 中
（3）当AOF系统达到设定的条件时，会调用aof_fsync将数据写入磁盘
           这里的设定的条件，就是AOF性能的关键，其主要有3种类型
          1.AOF_FSYNC_NO:不保存
               该过程中命令只会追加到server.aof_buf中，但是不会执行写入磁盘，当redis被正常关闭，AOF功能关闭，或者buf 缓存写满了，或者定时保存操作执行，这3种情况下都会阻塞主进程，导致客户端请求失败。
          2.AOF_FSYNC_EVERYSECS 每秒保存一次
               后台子进程调用写入保存，不会阻塞主进程，发送宕机最大丢失数据2s内
          3.AOF_FSYNC_ALWAYS 没执行一个命令保存一次
               保证每条命令都保存，数据不丢失，但是会影响性能，因为每一次操作都会阻塞主进程
 
 
AOF提供了重写机制，可以减少命令
 
### redis rehash
redis 的rehash机制和php的rehash机制不相同；
php使用阻塞型rehash，在此期间不能对hash 表做任何操作，而redis不能，redis操作频繁，对性能要求高。
 
redis 采用渐进式rehash方式
redis中会保存两个hash数组，正常的操作只会针对h[0],h[1]做rehash之用；
在rehash过程中，所有的写都切换到h[1],读操作先针对h[0],都去不到再去都去h[1],

![](https://images2015.cnblogs.com/blog/564050/201611/564050-20161120194646263-783016722.png)

参考http://blog.csdn.net/a600423444/article/details/8944601
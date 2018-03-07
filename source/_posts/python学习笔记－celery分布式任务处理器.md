---
title: python学习笔记－celery分布式任务处理器
tags:
    - 电商
categories:
    - python
date: 2016-05-29 00:05:36
---

celery是用python写的一个异步的任务框架，功能非常强大，具体的说明可以查看官网，这里主要提供点demo让你迅速使用该框架
 
### 环境安装
默认安装好了redis
pip install celery
redis 用来作为任务消息的载体
 

### tasks.py
 ```
import sys
reload(sys)
sys.setdefaultencoding('utf-8’)  # 不加这句话，打印中文log会出错
 
from celery import Celery
 
celery = Celery('tasks', broker='redis://127.0.0.1:6379/0') ＃选择本地redis db＝0 作为消息载体, 第一个参数为任务名称
from celery.utils.log import get_task_logger # 倒入celery 中的log模块
logger = get_task_logger(__name__)
 
@celery.task(bind=True, max_retries=10,
             default_retry_delay=1 * 6)  # bind 表示开启, max_retries 是重新尝试的次数,default_retry_delay 是默认的间隔时间，尝试的时间
def exec_task_order_overtime(self, order_id):  # 订单到期后,执行订单失效的任务 
    try:
        logger.info('===================> exec_task_order_overtime order_id=%s' % order_id)
        success = BaseHandler.context_services.order_overtime_task_service.process_over_time(order_id)
        if success is False:
            logger.error(
                '<================order_overtime_task_service.process_over_time Failed, order_id=%s' % order_id)
            raise Return(False)
        else:
            logger.info(
                '<=================order_overtime_task_service.process_over_time Success, order_id=%s' % order_id)
    except Exception as exc:
        logger.info('exec_task_order_overtime retry, order_id=%s' % order_id)
        raise self.retry(exc=exc, countdown=3)  # 3 秒后继续尝试, 此处的countdown 优先级高于装饰器中的default_retry_delay
 ```
 

该文件路径下执行命令， 此后celery 开始作为消费之执行任务
$celery worker -A tasks --loglevel=info
 
生产者呢？执行下面语句后，redis中的db=0 的 celery key是一个 list 类型， 里面存放着执行任务，如果celery没有开启可以清晰看到；开启了celery可能已经被执行了
```
from celery import Celery
 
celery = Celery('tasks', broker='redis://127.0.0.1:6379/0') #消息载体
push_task_id = celery.send_task('tasks.exec_task_order_overtime'
                                , [order_id] # 参数，必须为list，具体可见源码，第三个可以为dict，我们这里没有使用
                                , countdown=10)  #延时多久执行 推送消息 
```
疑问1:

　　有的人会奇怪为什么exec_task_order_overtime有self， 有时候发现没有，区别在与装饰器task，如果@celery.task， 不是装饰器函数调用，则没有self，The bind argument to the task decorator will give access to self (the task type instance)
 
 
疑问2:
　　celery 的构造函数的参数，第一个为模块名，也就是本文件名称， 第二个为redis载体地址，看官网介绍
The first argument to Celery is the name of the current module, this only needed so names can be automatically generated when the tasks are defined in the __main__module.
The second argument is the broker keyword argument, specifying the URL of the message broker you want to use. Here using RabbitMQ (also the default option).
 
疑问3：
　　celery内存不足时，没有反馈机制：
我们知道socket网络传输中，当接受端来不及处理的时候，发送端会阻塞；celery 完全没有相关机制；
我们不需要发送端阻塞，当然也不能，但是我们的celery来不及处理时，应该缓存一些在redis中，虽然会造成消息处理不及时，但是也不至于内存不足的问题出现，这里可能需要自己做一些处理；
例如：retry次数控制尽可能少，任务执行失败后记录下来，根据业务需要比如10分钟后再次抛入redis消息队列中
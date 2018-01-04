---
title: libevent学习
tags:
  - C++
categories:
  - C++
date: 2015-07-11 23:12:55
---

uscxml基于libevent库，使用到了里面到定时器，socket等功能

### Libevent 亮点：
事件驱动（event-driven），高性能;
轻量级，专注于网络，不如ACE那么臃肿庞大；
跨平台，支持Windows、Linux、*BSD和Mac Os；
支持多种I/O多路复用技术， epoll、poll、dev/poll、select和kqueue等；
支持I/O，定时器和信号等事件；
注册事件优先级；


### Reactor的事件处理机制

#### 反应机制原理
首先来回想一下普通函数调用的机制：程序调用某函数?函数执行，程序等待?函数将结果和控制权返回给程序?程序继续处理。

Reactor释义“反应堆”，是一种事件驱动机制。和普通函数调用的不同之处在于：应用程序不是主动的调用某个API完成处理，而是恰恰相反，Reactor逆置了事件处理流程，`应用程序需要提供相应的接口并注册到Reactor上，如果相应的时间发生，Reactor将主动调用应用程序注册的接口`，这些接口又称为“回调函数”。使用Libevent也是想Libevent框架注册相应的事件和回调函数；当这些时间发声时，Libevent会调用这些回调函数处理相应的事件（I/O读写、定时和信号）。

 这个概念非常常见，nodejs的事件机制，python等事件机制（tornado ioloop）c#等委托事件，linux的epoll机制
 
#### 优点
1）响应快，不必为单个同步时间所阻塞，虽然Reactor本身依然是同步的；
2）编程相对简单，可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程/进程的切换开销；
3）可扩展性，可以方便的通过增加Reactor实例个数来充分利用CPU资源；
4）可复用性，reactor框架本身与具体事件处理逻辑无关，具有很高的复用性；

### 使用libevent
#### 例子
1）首先初始化libevent库，并保存返回的指针
```
struct event_base * base = event_init();
```
实际上这一步相当于初始化一个Reactor实例；在初始化libevent后，就可以注册事件了。

2）初始化事件event，设置回调函数和关注的事件
```
evtimer_set(&ev, timer_cb, NULL);
```
事实上这等价于调用
```
event_set(&ev, -1, 0, timer_cb, NULL);
```

3）设置event从属的event_base

 ```
 event_base_set(base, &ev);  
```
这一步相当于指明event要注册到哪个event_base实例上；

4）是正式的添加事件的时候了
```
event_add(&ev, timeout);
```
基本信息都已设置完成，只要简单的调用event_add()函数即可完成，其中timeout是定时值；
这一步相当于调用Reactor::register_handler()函数注册事件。


5）程序进入无限循环，等待就绪事件并执行事件处理
```
event_base_dispatch(base);
```
代码实例
```
struct event ev;
struct timeval tv;
void time_cb(int fd, short event, void *argc)
{
    printf("timer wakeup/n");
    event_add(&ev, &tv); // reschedule timer
}
int main()
{
    struct event_base *base = event_init();
    tv.tv_sec = 10; // 10s period
    tv.tv_usec = 0;
    evtimer_set(&ev, time_cb, NULL);
    event_add(&ev, &tv);
    event_base_dispatch(base);
}
```
#### 事件处理流程

当应用程序向libevent注册一个事件后，libevent内部是怎么样进行处理的呢？下面就给出了这一基本流程。

   - 首先应用程序准备并初始化event，设置好事件类型和回调函数；这对应于前面第步骤2和3；
   - 向libevent添加该事件event。对于定时事件，libevent使用一个`最小根堆管理`，key为超时时间；对于Signal和I/O事件，libevent将其放入到等待链表（wait list）中，这是一个双向链表结构；
   - 程序调用event_base_dispatch()系列函数进入无限循环，等待事件。
 
 
 以select()函数为例；每次循环前libevent会检查定时事件的最小超时时间tv，根据tv设置select()的最大等待时间，以便于后面及时处理超时事件；
 
当select()返回后，首先检查超时事件，然后检查I/O事件；
Libevent将所有的就绪事件，放入到激活链表中；
然后对激活链表中的事件，调用事件的回调函数执行事件处理；
![](/images/cpp/libevent_1.jpg)

### 源码

1）头文件
主要就是event.h：事件宏定义、接口函数声明，主要结构体event的声明；
2）内部头文件
xxx-internal.h：内部数据结构和函数，对外不可见，以达到信息隐藏的目的；
3）libevent框架
event.c：event整体框架的代码实现；
4）对系统I/O多路复用机制的封装
epoll.c：对epoll的封装；
select.c：对select的封装；
devpoll.c：对dev/poll的封装;
kqueue.c：对kqueue的封装；
5）定时事件管理
min-heap.h：其实就是一个以时间作为key的小根堆结构；
6）信号管理
signal.c：对信号事件的处理；
7）辅助功能函数
evutil.h 和evutil.c：一些辅助功能函数，包括创建socket pair和一些时间操作函数：加、减和比较等。
8）日志
log.h和log.c：log日志函数
9）缓冲区管理
evbuffer.c和buffer.c：libevent对缓冲区的封装；
10）基本数据结构
compat/sys下的两个源文件：queue.h是libevent基本数据结构的实现，包括链表，双向链表，队列等；_libevent_time.h：一些用于时间操作的结构体定义、函数和宏定义；
11）实用网络库
http和evdns：是基于libevent实现的http服务器和异步dns查询库；


### Timer小根堆

 Libevent使用堆来管理Timer事件，其key值就是事件的超时时间，源代码位于文件min_heap.h中。
所有的数据结构书中都有关于堆的详细介绍，向堆中插入、删除元素时间复杂度都是O(lgN)，N为堆中元素的个数，而获取最小key值（小根堆）的复杂度为O(1)。堆是一个完全二叉树，基本存储方式是一个数组。
 


学习资料：
http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod
http://libevent.org/
https://www.cnblogs.com/lfsblack/p/5498556.html （推荐）
http://blog.chinaunix.net/uid-8048969-id-5008922.html
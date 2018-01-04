---
title: shared_ptr线程安全性分析
tags:
  - C++
  - boost
categories:
  - C++
date: 2015-10-15 01:49:52
---

### shared_ptr的线程安全性
boost官方文档对shared_ptr线程安全性的正式表述是：shared_ptr对象提供与内置类型相同级别的线程安全性。【shared_ptrobjects offer the same level of thread safety as built-in types.】具体是以下三点。
	- 同一个shared_ptr对象可以被多线程同时读取。【A shared_ptrinstance can be "read" (accessed using only const operations)simultaneously by multiple threads.】
	- 不同的shared_ptr对象可以被多线程同时修改（即使这些shared_ptr对象管理着同一个对象的指针）。【Different shared_ptr instances can be "written to"(accessed using mutable operations such as operator= or reset) simultaneouslyby multiple threads (even when these instances are copies, and share the samereference count underneath.) 】
	- 任何其他并发访问的结果都是无定义的。【Any other simultaneous accesses result in undefined behavior.】
    
第一种情况是对对象的并发读，自然是线程安全的。
第二种情况下，如果两个shared_ptr对象A和B管理的是不同对象的指针，则这两个对象完全不相关，支持并发写也容易理解。但如果A和B管理的是同一个对象P的指针，则A和B需要维护一块共享的内存区域，该区域记录P指针当前的引用计数。对A和B的并发写必然涉及对该引用计数内存区的并发修改，这需要boost做额外的工作，也是本文分析的重点。

另外weak_ptr和shared_ptr紧密相关，用户可以从weak_ptr构造出shared_ptr，也可以从shared_ptr构造weak_ptr，但是weak_ptr不涉及到对象的生命周期。由于shared_ptr的线程安全性是和weak_ptr耦合在一起的，本文的分析也涉及到weak_ptr。

下面先从总体上看一下shared_ptr和weak_ptr的实现。

### shared_ptr的结构图
以下是从boost源码提取出的shared_ptr和weak_ptr的类图。
![](/images/cpp/ptr_1.png)

我们首先忽略虚线框内的weak_ptr部分。最高层的shared_ptr就是用户直接使用的类，它提供shared_ptr的构造、复制、重置(reset函数）、解引用、比较、隐式转换为bool等功能。它包含一个指向被管理对象的指针，用来实现解引用操作，并且组合了一个shared_count对象，用来操作引用计数。
但shared_count类还不是引用计数类，它只是包含了一个指向引用计数类sp_counted_base的指针，功能上是对sp_counted_base操作的封装。shared_count对象的创建、复制和删除等操作，包含着对sp_counted_base的增加和减小引用计数的操作。
最后sp_counted_base类才保存了引用计数，并且对引用计数字段提供无锁保护。它也包含了一个指向被管理对象的指针，是用来删除被管理的对象的。sp_counted_base有三个派生类，分别处理用户指定Deleter和Allocator的情况：
1. sp_counted_impl_p：用户没有指定Deleter和Allocator
2. sp_counted_impl_pd：用户指定了Deleter，没有指定Allocator
3. sp_counted_impl_pda：用户指定了Deleter和 Allocator
创建指针P的第一个shared_ptr对象的时候，子对象shared_count同时被建立， shared_count根据用户提供的参数选择创建一个特定的sp_counted_base派生类对象X。之后创建的所有管理P的shared_ptr对象都指向了这个独一无二的X。
然后再看虚线框内的weak_ptr就清楚了。weak_ptr和shared_ptr基本上类似，只不过weak_ptr包含的是weak_count子对象，但weak_count和shared_count也都指向了sp_counted_base。
如果上面的文字还不够清楚，下面的代码就能说明问题。
```
shared_ptr<SomeObject> SP1(new SomeObject());
shared_ptr<SomeObject> SP2=SP1;
weak_ptr<SomeObject> WP1=SP1;
```
执行完以上代码后，内存中会创建以下对象实例，其中红色箭头表示指向引用计数对象的指针，黑色箭头表示指向被管理对象的指针。
![](/images/cpp/ptr_2.png)

从上面可以清楚的看出，SP1、SP2和WP1指向了同一个sp_counted_impl_p对象，这个sp_counted_impl_p对象保存引用计数，是SP1、SP2和WP1等三个对象共同操作的内存区。多线程并发修改SP1、SP2和WP1，有且只有sp_counted_impl_p对象会被并发修改，因此sp_counted_impl_p的线程安全性是shared_ptr以及weak_ptr线程安全性的关键问题。而sp_counted_impl_p的线程安全性是在其基类sp_counted_base中实现的。下面将着重分析sp_counted_base的代码。
### 引用计数类sp_counted_base
幸运的是，sp_counted_base的代码量很小，下面全文列出来，并添加有注释。
```
class sp_counted_base
{
private:
     // 禁止复制
    sp_counted_base( sp_counted_base const & );
    sp_counted_base & operator= ( sp_counted_baseconst & );
 
     // shared_ptr的数量
    long use_count_;  
     // weak_ptr的数量+1
    long weak_count_;      
 
public:
     // 唯一的一个构造函数，注意这里把两个计数都置为1
    sp_counted_base(): use_count_( 1 ), weak_count_( 1 ){    }
 
     // 虚基类，因此可以作为基类
    virtual ~sp_counted_base(){    }
 
     // 子类需要重载，用operator delete或者Deleter删除被管理的对象
    virtual void dispose() = 0;
 
     // 子类可以重载，用Allocator等删除当前对象
    virtual void destroy(){
        delete this;
    }
 
    virtual void * get_deleter( sp_typeinfo const & ti ) = 0;
 
     // 这个函数在根据shared_count复制shared_count的时候用到
     // 既然存在一个shared_count作为源，记为A，则只要A不释放，
     // use_count_就不会被另一个线程release()为1。
     // 另外，如果一个线程把A作为复制源，另一个线程释放A，执行结果是未定义的。
     void add_ref_copy(){
        _InterlockedIncrement( &use_count_ );
    }
 
     // 这个函数在根据weak_count构造shared_count的时候用到
     // 这是为了避免通过weak_count增加引用计数的时候，
     // 另外的线程却调用了release函数，清零use_count_并释放了指向的对象
    bool add_ref_lock(){
        for( ;; )
        {
            long tmp = static_cast< long const volatile& >( use_count_ );
            if( tmp == 0 ) return false;
 
            if( _InterlockedCompareExchange( &use_count_, tmp + 1, tmp ) == tmp )return true;
        }
    }
 
    void release(){
        if( _InterlockedDecrement( &use_count_ ) == 0 )
        {
              // use_count_从1变成0的时候，
              // 1. 释放对象
              // 2. 对weak_count_执行一次递减操作。这是因为在初始化的时候（use_count_从0变1时），weak_count初始值为1
            dispose();
            weak_release();
        }
    }
 
    void weak_add_ref(){
        _InterlockedIncrement( &weak_count_ );
    }
 
     // 递减weak_count_；且在weak_count为0的时候，把自己删除
    void weak_release(){
        if( _InterlockedDecrement( &weak_count_ ) == 0 )
        {
            destroy();
        }
    }
 
     // 返回引用计数。注意如果用户没有额外加锁，引用计数完全可能同时被另外的线程修改掉。
    long use_count() const{
        return static_cast<long const volatile &>( use_count_ );
    }
};
```
代码中的注释已经说明了一些问题，这里再重复一点：use_count_字段等于当前shared_ptr对象的数量，weak_count_字段等于当前weak_ptr对象的数量加1。
首先不考虑weak_ptr的情况。根据对shared_ptr类的代码分析（代码没有列出来，但很容易找到），shared_ptr之间的复制都是调用add_ref_copy和release函数进行的。假设两个线程分别对SP1和SP2进行操作，操作的过程无非是以下三种情况：
1. SP1和SP2都递增引用计数，即add_ref_copy被并发调用，也就是两个_InterlockedIncrement（&use_count_)并发执行，这是线程安全的。
2. SP1和SP2都递减引用计数，即release被并发调用，也就是_InterlockedDecrement(&use_count_ )并发执行，这也是线程安全的。只不过后执行的线程负责删除对象。
3.  SP1递增引用计数，调用add_ref_copy；SP2递减引用计数，调用release。由于SP1的存在，SP2的release操作无论如何都不会导致use_count_变为零，也就是说release中if语句的body永远不会被执行。因此，这种情况就化简为_InterlockedIncrement（&use_count_)和_InterlockedDecrement( &use_count_ )的并发执行，仍然是线程安全的。

然后考虑weak_ptr。如果是weak_ptr之间的操作，或者从shared_ptr构造weak_ptr，都不涉及到use_count_的操作，只需要调用weak_add_ref和weak_release来操作weak_count_。与上面的分析相同，`_InterlockedIncrement和_InterlockedDecrement`保证了weak_add_ref和weak_release并发操作的线程安全性。但如果存在从weak_ptr构造shared_ptr的操作，则需要考虑在构造weak_ptr的过程中，被管理的对象已经被其他线程被释放的情况。如果从weak_ptr构造shared_ptr仍然是通过add_ref_copy函数完成的，则可能发生以下错误情况：
 <table>
 <tr>
 <td></td>
 <td>线程1，从weak_ptr创建shared_ptr</td>
 <td>线程2，释放目前唯一存在的shared_ptr</td>
 </tr>
 <tr>
 <td>1</td>
 <td>判断use_count_大于0，等待执行add_ref_copy</td>
 <td></td>
 </tr>
  <tr>
 <td>2</td>
 <td></td>
 <td>调用release，use_count--。发现use_count为0，删除被管理的对象</td>
 </tr>
   <tr>
 <td>3</td>
 <td>
开始执行add_ref_copy，导致 use_count递增。
发生错误,use_count==1，但是对象已经被删除了</td>
 <td></td>
 </tr>
 </table>
 
我们自然会想，线程1在第三行结束后，再判断一次use_count是否为1，如果是1，认为对象已经删除，判断失败不就可以了吗。其实是行不通的，下面是一个反例。
 




<table>
 <tr>
 <td></td>
 <td>线程1，从weak_ptr创建shared_ptr</td>
 <td>线程2，释放目前唯一存在的shared_ptr</td>
 <td>线程3，从weak_ptr创建shared_ptr</td>
 </tr>
 <tr>
 <td>1</td>
 <td>判断use_count_大于0，等待执行add_ref_copy</td>
 <td></td>
  <td></td>
 </tr>
  <tr>
 <td>2</td>
 <td></td>
 <td>调用release，use_count--。发现use_count为0，删除被管理的对象</td>
 <td>判断use_count_大于0，等待执行add_ref_copy</td>
 </tr>
   <tr>
 <td>3</td>
 <td></td>
 <td>调用release，use_count--。发现use_count为0，删除被管理的对象</td>
 <td></td>
 </tr>
 <tr>
 <td>4</td>
 <td>开始执行add_ref_copy，导致 use_count递增。</td>
 <td></td>
 <td></td>
 </tr>
 <tr>
 <td>5</td>
 <td></td>
 <td></td>
 <td>执行add_ref_copy，导致 use_count递增。</td>
 </tr>
 <tr>
 <td>6</td>
 <td>发现use_count_ != 1，判断执行成功。
发生错误,use_count==2，但是对象已经被删除了</td>
 <td></td>
 <td>发现use_count_ != 1，判断执行成功。
发生错误,use_count==2，但是对象已经被删除了</td>
 </tr>
 
 </table>

实际上，boost从weak_ptr构造shared_ptr不是调用add_ref_copy，而是调用add_ref_lock函数。add_ref_lock是典型的无锁修改共享变量的代码，下面再把它的代码复制一遍，并添加证明注释。
```
    bool add_ref_lock(){
        for( ;; )
        {
            // 第一步，记录下use_count_
            long tmp = static_cast< long const volatile& >( use_count_ );
            // 第二步，如果已经被别的线程抢先清0了，则被管理的对象已经或者将要被释放，返回false
            if( tmp == 0 ) return false;
            // 第三步，如果if条件执行成功，
         // 说明在修改use_count_之前,use_count仍然是tmp，大于0
            // 也就是说use_count_在第一步和第三步之间，从来没有变为0过。
            // 这是因为use_count一旦变为0，就不可能再次累加为大于0
            // 因此，第一步和第三步之间，被管理的对象不可能被释放，返回true。
            if( _InterlockedCompareExchange( &use_count_, tmp + 1, tmp ) == tmp )return true;
        }
    }
 ```

在上面的注释中，用到了一个没有被证明的结论，“use_count一旦变为0，就不可能再次累加为大于0”。下面四条可以证明它。
1.   use_count_是sp_counted_base类的private对象，sp_counted_base也没有友元函数，因此use_count_不会被对象外的代码修改。
2.   成员函数add_ref_copy可以递增use_count_，但是所有对add_ref_copy函数的调用都是通过一个shared_ptr对象执行的。既然存在shared_ptr对象，use_count在递增之前一定不是0。
3.   成员函数add_ref_lock可以递增use_count_，但正如add_ref_lock代码所示，执行第三步的时候，tmp都是大于0的，因此add_ref_lock不会使use_count_从0递增到1
4.   其它成员函数从来不会递增use_count_
至此，我们可以放下心来，只要add_ref_lock返回true，递增引用计数的行为就是成功的。因此从weak_ptr构造shared_ptr的行为也是完全确定的，要么add_ref_lock返回true，构造成功，要么add_ref_lock返回false，构造失败。
综上所述，多线程通过不同的shared_ptr或者weak_ptr对象并发修改同一个引用计数对象sp_counted_base是线程安全的。而sp_counted_base对象是这些智能指针唯一操作的共享内存区，因此最终的结果就是线程安全的。
其它操作
前面我们分析了，不同的shared_ptr对象可以被多线程同时修改。那其它的问题呢，同一个shared_ptr对象可以对多线程同时修改吗？我们必须要注意到，前面所有的同步都是针对引用计数类sp_counted_base进行的，shared_ptr本身并没有任何同步保护。我们看下面boost文档举出来的非线程安全的例子

```
// thread A
p3.reset(new int(1));
// thread B
p3.reset(new int(2)); // undefined, multiple writes
```
下面是shared_ptr类相关的代码
```
template<class Y>
void reset(Y * p)
{
     this_type(p).swap(*this);
}
 
void swap(shared_ptr<T> & other)
{
     std::swap(px, other.px);
     pn.swap(other.pn);
}
```
可以看到，reset执行了两个修改成员变量的操作，thread A和thread B的执行结果可能是非法的。。
但是仿照内置对象的语义，boost提供了若干个原子函数，支持通过这些函数并发修改同一个shared_ptr对象。这包括atomic_store、atomic_exchange、atomic_compare_exchange等。以下是实现的代码，不再详细分析。
```
template<class T>
void atomic_store( shared_ptr<T> * p, shared_ptr<T> r ){
    boost::detail::spinlock_pool<2>::scoped_lock lock( p );
    p->swap( r );
}
 
template<class T>
shared_ptr<T> atomic_exchange( shared_ptr<T> * p, shared_ptr<T> r ){
    boost::detail::spinlock & sp = boost::detail::spinlock_pool<2>::spinlock_for( p );
 
    sp.lock();
    p->swap( r );
    sp.unlock();
 
    return r;
}
 
template<class T>
bool atomic_compare_exchange( shared_ptr<T> * p, shared_ptr<T> * v, shared_ptr<T> w ){
 
    boost::detail::spinlock & sp = boost::detail::spinlock_pool<2>::spinlock_for( p );
    sp.lock();
    if( p->_internal_equiv( *v ) ){
        p->swap( w );
        sp.unlock();
        return true;
    }
    else{
        shared_ptr<T> tmp( *p );
        sp.unlock();
        tmp.swap( *v );
        return false;
    }
}
```
总结
正如boost文档所宣称的，boost为shared_ptr提供了`与内置类型同级别的线程安全性`。这包括：
1. 同一个shared_ptr对象可以被多线程同时读取。
2. 不同的shared_ptr对象可以被多线程同时修改。
3. 同一个shared_ptr对象不能被多线程直接修改，但可以通过原子函数完成。

如果把上面的表述中的"shared_ptr"替换为“内置类型”也完全成立。
最后，整理这个东西的时候我也发现有些关键点很难表述清楚，这也是由于线程安全性本身比较难严格证明。如果想要完全理解，还是建议阅读shared_ptr完整的代码。shared_ptr在windows下的源代码我已经单独从boost中提取了出来，整理成了单独的文件，且去掉了不相关的条件编译指令。

参考 http://blog.csdn.net/jiangfuqiang/article/details/8292906
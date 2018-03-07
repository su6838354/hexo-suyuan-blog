---
title: python学习笔记-functools.wraps 装饰器
tags:
    - python
categories:
    - python
date: 2016-06-03 23:51:58
---

wraps其实没有实际的大用处， 就是用来解决装饰器导致的原函数名指向的函数 的属性发生变化的问题；
装饰器装饰过函数func， 此时func不是指向真正的func，而是指向装饰器中的装饰过的函数
 
```
import sys

debug_log = sys.stderr

def trace(func):
        if debug_log:
                def callf(*args, **kwargs):
                        """A wrapper function."""
                        debug_log.write('Calling function: {}\n'.format(func.__name__))
                        res = func(*args, **kwargs)
                        debug_log.write('Return value: {}\n'.format(res))
                        return res
                return callf
        else:
                return func

@trace
def square(x):
        """Calculate the square of the given number."""
        return x * x 
```
这里的 square 其实指向的是 calls， 可以用help(square)或者 `square.__name__ `看下。
square 被装饰后的函数其实已经是另外一个函数了（函数名等函数属性会发生改变）
 
如果使用wraps进行修饰
```
def trace(func):
        if debug_log:
          @functools.wraps(func)
                def callf(*args, **kwargs):
                        """A wrapper function."""
                        debug_log.write('Calling function: {}\n'.format(func.__name__))
                        res = func(*args, **kwargs)
                        debug_log.write('Return value: {}\n'.format(res))
                        return res
                return callf
        else:
                return func
```
此时 用trace 装饰的 square 的属性就不会变化了，可以help(square) 看看

 
原因：我们把wraps的装饰的代码翻译如下，其等价为：
```
def trace(func):
        if debug_log:
                def _callf(*args, **kwargs):
                        """A wrapper function."""
                        debug_log.write('Calling function: {}\n'.format(func.__name__))
                        res = func(*args, **kwargs)
                        debug_log.write('Return value: {}\n'.format(res))
                        return res

                callf = functools.update_wrapper(_callf, wrapped = func,assigned = functools.WRAPPER_ASSIGNMENTS,updated = functools.WRAPPER_UPDATES)

                return callf
        else:
                return func
```
update_wrapper做的工作很简单，就是用参数wrapped表示的函数对象（例如：square）的一些属性（如：`__name__、 __doc__`）覆盖参数wrapper表示的函数对象（例如：callf，这里callf只是简单地调用square函数，因此可以说callf是 square的一个wrapper function）的这些相应属性。

 
因此，本例中使用wraps装饰器“装饰”过callf后，callf的`__doc__、__name__`等属性和trace要“装饰”的函数square的这些属性完全一样。
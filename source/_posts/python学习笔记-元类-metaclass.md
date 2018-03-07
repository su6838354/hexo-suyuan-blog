---
title: python学习笔记-元类__metaclass__
tags:
    - python
categories:
    - python
date: 2016-03-28 00:05:06
---


type 其实就是元类，type 是python 背后创建所有对象的元类
 
python 中的类的创建规则：
假设创建Foo 这个类
 ```
class Foo(Bar):
　　def __init__():
　　　　pass
 ```
- Foo中有__metaclass__这个属性吗？如果有，Python会在内存中通过__metaclass__创建一个名字为Foo的类对象，他是一个类，但是本身类就是对象，一个python文件模块也属于一个对象。
如果Python没有找到__metaclass__，它会继续在Bar（父类）中寻找__metaclass__属性，并尝试做和前面同样的操作。
- 如果Python在任何父类中都找不到__metaclass__，它就会在模块层次中去寻找__metaclass__，并尝试做同样的操作。
- 如果还是找不到__metaclass__,Python就会用内置的type来创建这个类对象。

用途：元类的主要目的就是为了当创建类时能够自动地改变类，元类的主要用途是创建API。一个典型的例子是Django ORM。它允许你像这样定义：
Flask sqlalchemy ORM 类的定义也是通过继承一个被创建的类Base， 并且要注意sqlalchemy 中所有的表类必须继承一个Base 对象，不然继承后创建的Base的表将无法实现ORM映射；
 sqlalchemy 实例： 
```
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()


class HotWordType(Base):
    # 表名称
    __tablename__ = 'hotWordType'
    # id typeName
    id = Column(Integer, primary_key=True)  # 主键
    typeName = Column(String(20), nullable=False)  # 类型名
    hotWord = relationship('HotWord', backref='hotWordType')
```
declarative_base 中的元类源码：
```
The new base class will be given a metaclass that produces
appropriate :class:`~sqlalchemy.schema.Table` objects and makes
the appropriate :func:`~sqlalchemy.orm.mapper` calls based on the
information provided declaratively in the class and any subclasses
of the class.

class DeclarativeMeta(type):
    def __init__(cls, classname, bases, dict_):
        if '_decl_class_registry' not in cls.__dict__:
            _as_declarative(cls, classname, cls.__dict__)
        type.__init__(cls, classname, bases, dict_)

    def __setattr__(cls, key, value):
        _add_attribute(cls, key, value) 
```
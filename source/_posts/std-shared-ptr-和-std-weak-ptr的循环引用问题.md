---
title: 'std::shared_ptr 和 std::weak_ptr的循环引用问题'
tags:
  - C++
  - boost
categories:
  - C++
date: 2015-10-11 00:12:01
---

在std::shared_ptr被引入之前，C++标准库中实现的用于管理资源的智能指针只有std::auto_ptr一个而已。std::auto_ptr的作用非常有限，因为它存在被管理资源的所有权转移问题。

这导致多个std::auto_ptr类型的局部变量不能共享同一个资源，这个问题是非常严重。
智能指针内存管理要解决的根本问题是：一个堆对象（或则资源，比如文件句柄）在被多个对象引用的情况下，何时释放资源的问题。

何时释放很简单，就是在最后一个引用它的对象被释放的时候释放它。关键的问题在于无法确定哪个引用它的对象是被最后释放的。

std::shared_ptr确定最后一个引用它的对象何时被释放的基本想法是：对被管理的资源进行引用计数，当一个shared_ptr对象要共享这个资源的时候，该资源的引用计数加1，当这个对象生命期结束的时候，再把该引用技术减少1。这样当最后一个引用它的对象被释放的时候，资源的引用计数减少到0，此时释放该资源。下边是一个shared_ptr的用法例子：

```
#include <iostream>
#include <memory>

class Woman;
class Man{
private:
	std::weak_ptr<Woman> _wife;
	//std::shared_ptr<Woman> _wife;
public:
	void setWife(std::shared_ptr<Woman> woman){
		_wife = woman;
	}

	void doSomthing(){
		if(_wife.lock()){
		}
	}

	~Man(){
		std::cout << "kill man\n";
	}
};

class Woman{
private:
	//std::weak_ptr<Man> _husband;
	std::shared_ptr<Man> _husband;
public:
	void setHusband(std::shared_ptr<Man> man){
		_husband = man;
	}
	~Woman(){
		std::cout <<"kill woman\n";
	}
};


int main(int argc, char** argv){
	std::shared_ptr<Man> m(new Man());
	std::shared_ptr<Woman> w(new Woman());
	if(m && w) {
		m->setWife(w);
		w->setHusband(m);
	}
	return 0;
}
```

在Man类内部会引用一个Woman，Woman类内部也引用一个Man。当一个man和一个woman是夫妻的时候，他们直接就存在了`相互引用`问题。

man内部有个用于管理wife生命期的shared_ptr变量，也就是说wife必定是在husband去世之后才能去世。同样的，woman内部也有一个管理husband生命期的shared_ptr变量，也就是说husband必须在wife去世之后才能去世。这就是循环引用存在的问题：husband的生命期由wife的生命期决定，wife的生命期由husband的生命期决定，最后两人都死不掉，违反了自然规律，`导致了内存泄漏`。

解决std::shared_ptr循环引用问题的钥匙在`weak_ptr`手上。
weak_ptr对象引用资源时不会增加引用计数，但是它能够通过lock()方法来判断它所管理的资源是否被释放。另外很自然地一个问题是：既然weak_ptr不增加资源的引用计数，那么在使用weak_ptr对象的时候，资源被突然释放了怎么办呢？答案是你根本不能直接通过weak_ptr来访问资源。那么如何通过weak_ptr来间接访问资源呢？答案是：在需要访问资源的时候weak_ptr为你生成一个shared_ptr，shared_ptr能够保证在shared_ptr没有被释放之前，其所管理的资源是不会被释放的。创建shared_ptr的方法就是lock()方法。

细节：shared_ptr实现了operator bool() const方法来判断一个管理的资源是否被释放。
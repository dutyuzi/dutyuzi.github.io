---
layout:     post
title:      "单例模式"
subtitle:   "The singleton patterne"
date:        2017/01/05  21:01:39 
author:     "MaJ"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 设计模式
---

### 前言
很多时候，编写的类的属于工具性质，不需要存储与自身有关的数据，这种情况下，每次都去new一个对象，就会增加开销，代码质量就很低。其实只需要实例化一个对象就可以，简单的方法是采取全局或者静态变量的方式，这种方式会影响封装性。
单例模式就是用来解决以上问题的。具体实现方法如下：

1、将默认的构造函数声明为私有，防止被外部new。声明私Singleton* instance;

2、声明公有静态方法 getInstance(),new一个instance，返回该instance的指针。


### C++实现代码

类定义：

	#ifndef _SINGLETON_H_
	#define _SINGLETON_H_
	
	
	class Singleton{
	public:
		static Singleton* getInstance();
	
	private:
		Singleton();
		~Singleton();
		//把复制构造函数和=操作符也设为私有,防止被复制
		Singleton(const Singleton&);
		Singleton& operator=(const Singleton&);
	
		static Singleton* instance;
	};
	
	#endif

类实现：

	#include "Singleton.h"
	
	Singleton::Singleton(){
	
	}
	
	Singleton::~Singleton(){

        if (instance ！= NULL)
        {
	       delete instance；
	    }
	}

	Singleton::Singleton(const Singleton&){
	
	}
	
	Singleton& Singleton::operator=(const Singleton&){
	
	}
	
	//在此处初始化
	Singleton* Singleton::instance = new Singleton();
	Singleton* Singleton::getInstance(){
		return instance;
	}

类使用：

	#include "Singleton.h"
	#include <stdio.h>
	int main(){
		Singleton* singleton1 = Singleton::getInstance();
		Singleton* singleton2 = Singleton::getInstance();
	
		if (singleton1 == singleton2)
			fprintf(stderr,"singleton1 = singleton2\n");
	
		return 0;
	}


### 总结
	
单例模式常常与工厂模式结合使用，因为工厂只需要创建产品实例就可以了，在多线程的环境下也不会造成任何的冲突，因此只需要一个工厂实例就可以了。





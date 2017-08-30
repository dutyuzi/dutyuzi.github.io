---
layout:     post
title:      "C++对象模型"
subtitle:   "C++ Object Model"
date:        2017/8/30 16:57:02 
author:     "MaJ"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - C++
---
---
## 虚函数表

C++中虚函数的作用主要是为了实现多态机制，多态简单来说就是指在继承层次中，父类的指针可以具有多种形态——当它指向某个子类对象时，通过它能够调用到子类的函数，而非父类的函数。

	class Base {     virtual void print(void);    }
	class Drive1 :public Base{    virtual void print(void);    }
	class Drive2 :public Base{    virtual void print(void);    }

---
	Base * ptr1 = new Base; 
	Base * ptr2 = new Drive1;  
	Base * ptr3 = new Drive2;

---

	ptr1->print(); //调用Base::print()
	prt2->print();//调用Drive1::print()
	prt3->print();//调用Drive2::print()

这是一种运行期的多态，即父类指针唯有在运行时候才知道所指的真正类型是什么。这种运行期的决议，是通过**虚函数表**来实现的。

## 使用指针访问虚表
如果我们丰富Base类，使其拥有多个虚函数

	class Base
	{
	public:
	 
	    Base(int i) :baseI(i){};
	
	    virtual void print(void){ cout << "调用了虚函数Base::print()"; }
	
	    virtual void setI(){cout<<"调用了虚函数Base::setI()";}
	
	    virtual ~Base(){}
	 
	private:
	 
	    int baseI;
	
	};

当一个类本身定义了虚函数或者其父类有虚函数时，为了支持多态机制，编译器将为该类添加一个虚函数指针（vptr）。虚函数指针一般都放在对象内存布局的第一个位置上，这是为了保证在多层继承或者多重继承的情况下以最高的效率取到虚函数表。
当vptr在对象内存的最前面时，对象的地址即为虚函数指针地址。我们可以取得虚函数指针的地址：

	Base b(1000);
	int * vptrAdree = (int *)(&b);  
	cout << "虚函数指针（vprt）的地址是：\t"<<vptrAdree << endl;
我们强行把类对象的地址转换为int*类型，取得了虚函数指针的地址，***虚函数指针指向虚函数表***。虚函数表中存放的是一系列虚函数的地址，虚函地址出现的顺序与类中虚函数声明的顺序一致。对虚函数指针地址取值，可以得到虚函数表的地址，也是虚函数表中第一个虚函数的地址：

 	typedef void(*Fun)(void);
    Fun vfunc = (Fun)*( (int *)*(int*)(&b));
    cout << "第一个虚函数的地址是：" << (int *)*(int*)(&b) << endl;
    cout << "通过地址，调用虚函数Base::print()：";
    vfunc();

## 对象模型概述
在c++中，有两种数据成员：static和nonstatic以及三种类成员函数：static、nonstatic和virtual：
下面的类包含了上面5种类型的数据或者函数：


	class Base
	{
	public:
	 
	    Base(int i) :baseI(i){};
	  
	    int getI(){ return baseI; }
	 
	    static void countI(){};
	 
	    virtual void print(void){ cout << "Base::print()"; }
	 
	    virtual ~Base(){}
	 
	private:
	 
	    int baseI;
	 
	    static int baseS;
	};

### 简单对象模型

这个模型非常地简单粗暴。在该模型下，对象由一系列的指针组成，每一个指针都指向一个数据成员或成员函数，也即是说，每个数据成员和成员函数在类中所占的大小是相同的，都为一个指针的大小。这样有个好处——很容易算出对象的大小，不过赔上的是空间和执行期效率。

### 表格驱动模型

这个模型在简单对象模型的基础上又添加一个间接层，它把类中的数据分成了两个部分：数据部分与函数部分，并使用两张表格，一张存放数据本身，一张存放函数的地址（也即函数比成员多一次寻址），而类对象仅仅含有两个指针，分别指向上面这两个表。这样看来，对象的大小是固定为两个指针大小。这个模型也没有用于实际应用于真正的C++编译器上。

### 非继承下的C++对象模型
概述：在此模型下，nonstatic数据成员被置于每个类对象中，而static数据成员被置于类对象外。static函数和nonstatic函数也都放在类对象之外。而对应虚函数，则通过虚函数表+虚指针来支持,具体如下：
- 每个类生成一个表格，称为虚表（virtual table，简称vtbl）。虚表中存放着一堆指针，这些指针指向该类的每一个虚函数。
- 每个类对象都拥有一个虚表指针（vptr），由编译器为其生成，大多数编译器会把vptr放在一个类对象的最前端。
- 虚表的前面设置了一个指向type_info的指针，用来支持RTTI（Run Time Type Identification，运行时类型识别）。

在此模型下，Base的对象模型如图：

![img](/img/objectmodel.png)

### 其他
待续。。。

# 参考
[http://blog.jobbole.com/101583/](http://blog.jobbole.com/101583/)

---
layout:     post
title:      "linux进程和线程的区别"
subtitle:   "process and thread"
date:        2017/02/03  11:01:39 
author:     "MaJ"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - other
---

### 一句话
进程是系统***资源分配***的基本单位，线程是系统***任务调度***的基本单位。

### 理解
进程是程序执行时的一个实例，即它是程序已经执行到课中程度的数据结构的汇集。从内核的观点看，进程的目的就是担当分配系统资源（CPU时间、内存等）的基本单位。

线程是进程的一个执行流，是CPU调度和分派的基本单位，它是比进程更小的能独立运行的基本单位。一个进程由几个线程组成（拥有很多相对独立的执行流的用户程序共享应用程序的大部分数据结构），线程与同属一个进程的其他的线程共享进程所拥有的全部资源。

进程有独立的地址空间，一个进程崩溃后，在保护模式下不会对其它进程产生影响，而线程只是一个进程中的不同执行路径。线程有自己的堆栈和局部变量，但线程***没有单独的地址空间***，一个线程死掉就等于整个进程死掉，所以多进程的程序要比多线程的程序健壮，但在进程切换时，耗费资源较大，效率要差一些。但对于一些要求同时进行并且又要共享某些变量的并发操作，只能用线程，不能用进程。

### 参考
[http://www.ruanyifeng.com/blog/2013/04/processes_and_threads.html](http://www.ruanyifeng.com/blog/2013/04/processes_and_threads.html "有意思的解释")
[http://http://blog.csdn.net/forrest2009/article/details/6413756](http://http://blog.csdn.net/forrest2009/article/details/6413756 "linux下进程和线程的区别")






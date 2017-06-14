---
layout:     post
title:      "linux进程间通信"
subtitle:   "Communication between Linux processes"
date:        2017/06/14  15:29:00 
author:     "MaJ"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - linux
---
---

# 前言 #

进程间通信（IPC，InterProcess Communication）是指在不同进程之间传播或交换信息。IPC的方式通常有管道（包括无名管道和命名管道）、消息队列、信号量、共享存储、Socket、Streams等。其中 Socket和Streams支持不同主机上的两个进程IPC。

linux下的进程通信手段基本上是从Unix平台上的进程通信手段继承而来的。而对Unix发展做出重大贡献的两大主力AT&T的贝尔实验室及BSD（加州大学伯克利分校的伯克利软件发布中心）在进程间通信方面的侧重点有所不同。前者对Unix早期的进程间通信手段进行了系统的改进和扩充，形成了“system V IPC”，通信进程局限在单个计算机内；后者则跳过了该限制，形成了基于套接口（socket）的进程间通信机制。Linux则把两者继承了下来，如图示：

![img](/img/IPC.jpg)

# 管道 #
## 特点  ##

1. 它是半双工的（即数据只能在一个方向上流动），具有固定的读端和写端。
2. 它只能用于具有亲缘关系的进程之间的通信（也是父子进程或者兄弟进程之间）。
3. 它可以看成是一种特殊的文件，对于它的读写也可以使用普通的read、write 等函数。但是它不是普通的文件，并不属于其他任何文件系统，并且只存在于内存中。

## 函数 ##
	#include <unistd.h>
	int pipe(int fd[2]);    // 返回值：若成功返回0，失败返回-1

当一个管道建立时，它会创建两个文件描述符：fd[0]为读而打开，fd[1]为写而打开。如下图：

![img](/img/pipe1.png)

要关闭管道只需将这两个文件描述符关闭即可。

## 示例代码 ##
单个进程中的管道几乎没有任何用处。所以，通常调用 pipe 的进程接着调用 fork，这样就创建了父进程与子进程之间的 IPC 通道。如下图所示：

![img](/img/pipe2.png)

若要数据流从父进程流向子进程，则关闭父进程的读端（fd[0]）与子进程的写端（fd[1]）；反之，则可以使数据流从子进程流向父进程。

	#include<stdio.h>
	#include<unistd.h>
	
	int main()
	{
	    int fd[2];  // 两个文件描述符
	    pid_t pid;
	    char buff[100];
	
	    if(pipe(fd) < 0)  // 创建管道
	        printf("Create Pipe Error!\n");
	
	    if((pid = fork()) < 0)  // 创建子进程
	        printf("Fork Error!\n");
	    else if(pid > 0)  // 父进程
	    {
	        close(fd[0]); // 关闭读端
	        write(fd[1], "hello world\n", 100);
	        printf("父进程发送:%s", "hello world\n");
	    }
	    else
	    {
	        close(fd[1]); // 关闭写端
	        read(fd[0], buff, 100);
	        printf("子进程接收:%s", buff);
	    }
	
	    return 0;
	}

**运行结果**

![img](/img/rpipe.jpg)


### 参考
[select、poll、epoll之间的区别总结](http://www.cnblogs.com/Anker/p/3265058.html)

[select,poll,epoll区别](http://blog.csdn.net/ithomer/article/details/5971779)

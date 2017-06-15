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

进程间通信（IPC，InterProcess Communication）是指在不同进程之间传播或交换信息。IPC的方式通常有管道（包括无名管道和命名管道）、消息队列、信号量、共享存储、Socket等。其中 Socket支持不同主机上的两个进程IPC。

linux下的进程通信手段基本上是从Unix平台上的进程通信手段继承而来的。而对Unix发展做出重大贡献的两大主力AT&T的贝尔实验室及BSD（加州大学伯克利分校的伯克利软件发布中心）在进程间通信方面的侧重点有所不同。前者对Unix早期的进程间通信手段进行了系统的改进和扩充，形成了“system V IPC”，通信进程局限在单个计算机内；后者则跳过了该限制，形成了基于套接口（socket）的进程间通信机制。Linux则把两者继承了下来，如图示：

![img](/img/IPC.jpg)

# 一、管道 #
管道，通常指**无名管道**，是 UNIX 系统IPC最古老的形式。
## 特点  ##

1. 它是半双工的（即数据只能在一个方向上流动），具有固定的读端和写端。
2. 它只能用于**具有亲缘关系**的进程之间的通信（也是父子进程或者兄弟进程之间）。
3. 它可以看成是一种特殊的文件，对于它的读写也可以使用普通的read、write 等函数。但是它不是普通的文件，并不属于其他任何文件系统，并且只存在于内存中。

## 相关API ##
	#include <unistd.h>
	int pipe(int fd[2]);    // 返回值：若成功返回0，失败返回-1

当一个管道建立时，它会创建两个文件描述符：fd[0]为读而打开，fd[1]为写而打开。如下图：

![img](/img/pipe1.png)

要关闭管道只需将这两个文件描述符关闭即可。

## 示例代码 ##
**fork()函数介绍**

在Linux 中，创造新进程的方法只有一个，就是fork函数。其他一些库函数，如system()，看起来似乎它们也能创建新的进程，如果能看一下它们的源码就会明白，它们实际上也在内部调用了fork。包括我们在命令行下运行应用程序，新的进程也是由shell调用fork制造出来的。

	/* fork_test.c */
	#include<sys/types.h>
	#inlcude<unistd.h>
	main()
	{
	pid_t pid;
	 
	/*此时仅有一个进程*/
	pid=fork();
	/*此时已经有两个进程在同时运行*/
	if(pid<0)
	printf("error in fork!");
	else if(pid==0)
	printf("I am the child process, my process ID is %d/n",getpid());
	else
	printf("I am the parent process, my process ID is %d/n",getpid());
	}
	编译并运行：
	$gcc fork_test.c -o fork_test
	$./fork_test
	I am the parent process, my process ID is 1991
	I am the child process, my process ID is 1992

看这个程序的时候，头脑中必须首先了解一个概念：在语句pid=fork()之前，只有一个进程在执行这段代码，但在这条语句之后，就变成两个进程在执行了，这两个进程的代码部分完全相同，将要执行的下一条语句都是if(pid==0)……。

两个进程中，原先就存在的那个被称作“父进程”，新出现的那个被称作“子进程”。父子进程的区别除了进程标志符（process ID）不同外，变量pid的值也不相同，pid存放的是fork的返回值。fork调用的一个奇妙之处就是它仅仅被调用一次，却能够返回两次，它可能有三种不同的返回值：

- 在父进程中，fork返回新创建子进程的进程ID；
- 在子进程中，fork返回0；
- 如果出现错误，fork返回一个负值；

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

	root@ubuntu:~/mytest# ./pipe 
	父进程发送:hello world
	root@ubuntu:~/mytest# 子进程接收:hello world

# 二、FIFO #
FIFO，也称为**命名管道**，它是一种文件类型
## 特点 ##
1. FIFO可以在**无关的进程**之间交换数据，与无名管道不同。
2. FIFO有路径名与之相关联，它以一种特殊设备文件形式存在于文件系统中。
## 相关API ##
	#include <sys/stat.h>
	// 返回值：成功返回0，出错返回-1
	int mkfifo(const char *pathname, mode_t mode);
其中的 mode 参数与open函数中的 mode 相同。一旦创建了一个 FIFO，就可以用一般的文件I/O函数操作它。

当 open 一个FIFO时，是否设置非阻塞标志（O_NONBLOCK）的区别：



- 若没有指定O_NONBLOCK（默认），只读 open 要阻塞到某个其他进程为写而打开此 FIFO。类似的，只写 open 要阻塞到某个其他进程为读而打开它。

- 若指定了O_NONBLOCK，则只读 open 立即返回。而只写 open 将出错返回 -1 如果没有进程已经为读而打开该 FIFO，其errno置ENXIO。

## 示例代码 ##

***read_fifo.c***

	#include<stdio.h>
	#include<stdlib.h>
	#include<errno.h>
	#include<fcntl.h>
	#include<sys/stat.h>

	int main()
	{
		int fd;
		int len;
		char buf[1024];
	
		if(mkfifo("fifo1", 0666) < 0 && errno!=EEXIST) // 创建FIFO管道
			perror("Create FIFO Failed");
	
		if((fd = open("fifo1", O_RDONLY)) < 0)  // 以读打开FIFO
		{
			perror("Open FIFO Failed");
			exit(1);
		}
		
		while((len = read(fd, buf, 1024)) > 0) // 读取FIFO管道
			printf("Read message: %s", buf);
	
		close(fd);  // 关闭FIFO文件
		return 0;
	}

***write_fifo.c***

	#include<stdio.h>
	#include<stdlib.h>   // exit
	#include<fcntl.h>    // O_WRONLY
	#include<sys/stat.h>
	#include<time.h>     // time
	
	int main()
	{
		int fd;
		int n, i;
		char buf[1024];
		time_t tp;
	
		printf("I am %d process.\n", getpid()); // 说明进程ID
		
		if((fd = open("fifo1", O_WRONLY)) < 0) // 以写打开一个FIFO 
		{
			perror("Open FIFO Failed");
			exit(1);
		}
	
		for(i=0; i<10; ++i)
		{
			time(&tp);  // 取系统当前时间
			n=sprintf(buf,"Process %d's time is %s",getpid(),ctime(&tp));
			printf("Send message: %s", buf); // 打印
			if(write(fd, buf, n+1) < 0)  // 写入到FIFO中
			{
				perror("Write FIFO Failed");
				close(fd);
				exit(1);
			}
			sleep(1);  // 休眠1秒
		}
	
		close(fd);  // 关闭FIFO文件
		return 0;
	}

在两个终端里用 gcc 分别编译运行上面两个文件，可以看到输出结果如下：

	root@ubuntu:~/mytest# ./write_fifo 
	I am 9561 process.
	Send message: Process 9561's time is Wed Jun 14 09:07:31 2017
	Send message: Process 9561's time is Wed Jun 14 09:07:32 2017
	Send message: Process 9561's time is Wed Jun 14 09:07:33 2017
	Send message: Process 9561's time is Wed Jun 14 09:07:34 2017
	Send message: Process 9561's time is Wed Jun 14 09:07:35 2017
	Send message: Process 9561's time is Wed Jun 14 09:07:36 2017
	Send message: Process 9561's time is Wed Jun 14 09:07:37 2017
	Send message: Process 9561's time is Wed Jun 14 09:07:38 2017
	Send message: Process 9561's time is Wed Jun 14 09:07:39 2017
	Send message: Process 9561's time is Wed Jun 14 09:07:40 2017

---

	root@ubuntu:~/mytest# ./read_fifo 
	Read message: Process 9561's time is Wed Jun 14 09:07:31 2017
	Read message: Process 9561's time is Wed Jun 14 09:07:32 2017
	Read message: Process 9561's time is Wed Jun 14 09:07:33 2017
	Read message: Process 9561's time is Wed Jun 14 09:07:34 2017
	Read message: Process 9561's time is Wed Jun 14 09:07:35 2017
	Read message: Process 9561's time is Wed Jun 14 09:07:36 2017
	Read message: Process 9561's time is Wed Jun 14 09:07:37 2017
	Read message: Process 9561's time is Wed Jun 14 09:07:38 2017
	Read message: Process 9561's time is Wed Jun 14 09:07:39 2017
	Read message: Process 9561's time is Wed Jun 14 09:07:40 2017

# 三、消息队列 #
## 特点 ##

1. 消息队列是面向记录的，其中的消息具有特定的格式以及特定的优先级。
2. 消息队列独立于发送与接收进程。进程终止时，消息队列及其内容并不会被删除。
3. 消息队列可以实现消息的随机查询,消息不一定要以先进先出的次序读取,也可以按消息的类型读取。

## 相关API ##

	#include <sys/msg.h>
	// 创建或打开消息队列：成功返回队列ID，失败返回-1
	int msgget(key_t key, int flag);
	// 添加消息：成功返回0，失败返回-1
	int msgsnd(int msqid, const void *ptr, size_t size, int flag);
	// 读取消息：成功返回消息数据的长度，失败返回-1
	int msgrcv(int msqid, void *ptr, size_t size, long type,int flag);
	// 控制消息队列：成功返回0，失败返回-1
	int msgctl(int msqid, int cmd, struct msqid_ds *buf);

在以下两种情况下，msgget将创建一个新的消息队列：

- 如果没有与键值key相对应的消息队列，并且flag中包含了IPC_CREAT标志位。
- key参数为IPC_PRIVATE。

函数msgrcv在读取消息队列时，type参数有下面几种情况：

- type == 0，返回队列中的第一个消息；
- type > 0，返回队列中消息类型为 type 的第一个消息；
- type < 0，返回队列中消息类型值小于或等于 type 绝对值的消息，如果有多个，则取类型值最小的消息。
可以看出，type值非 0 时用于以非先进先出次序读消息。也可以把 type 看做优先级的权值。

## 示例代码 ##
下面写了一个简单的使用消息队列进行IPC的例子，服务端程序一直在等待特定类型的消息，当收到该类型的消息以后，发送另一种特定类型的消息作为反馈，客户端读取该反馈并打印出来。

***msg_server.c***

	#include <stdio.h>
	#include <stdlib.h>
	#include <sys/msg.h>
	#include <unistd.h>

	// 用于创建一个唯一的key
	#define MSG_FILE "/etc/passwd"
	
	// 消息结构
	struct msg_form {
		long mtype;
		char mtext[256];
	};
	
	int main()
	{
		int msqid;
		key_t key;
		struct msg_form msg;
		
		// 获取key值
		if((key = ftok(MSG_FILE,'z')) < 0)
		{
			perror("ftok error");
			exit(1);
		}
	
		// 打印key值
		printf("Message Queue - Server key is: %d.\n", key);
	
		// 创建消息队列
		if ((msqid = msgget(key, IPC_CREAT|0777)) == -1)
		{
			perror("msgget error");
			exit(1);
		}
	
		// 打印消息队列ID及进程ID
		printf("My msqid is: %d.\n", msqid);
		printf("My pid is: %d.\n", getpid());
	
		// 循环读取消息
		for(;;) 
		{
			msgrcv(msqid, &msg, 256, 888, 0);// 返回类型为888的第一个消息
			printf("Server: receive msg.mtext is: %s.\n", msg.mtext);
			printf("Server: receive msg.mtype is: %d.\n", msg.mtype);
	
			msg.mtype = 999; // 客户端接收的消息类型
			sprintf(msg.mtext, "hello, I'm server %d", getpid());
			msgsnd(msqid, &msg, sizeof(msg.mtext), 0);
		}
		return 0;
	}

***msg_client.c***

	#include <stdio.h>
	#include <stdlib.h>
	#include <sys/msg.h>
	#include <unistd.h>

	// 用于创建一个唯一的key
	#define MSG_FILE "/etc/passwd"
	
	// 消息结构
	struct msg_form {
		long mtype;
		char mtext[256];
	};
	
	int main()
	{
		int msqid;
		key_t key;
		struct msg_form msg;
	
		// 获取key值
		if ((key = ftok(MSG_FILE, 'z')) < 0) 
		{
			perror("ftok error");
			exit(1);
		}
	
		// 打印key值
		printf("Message Queue - Client key is: %d.\n", key);
	
		// 打开消息队列
		if ((msqid = msgget(key, IPC_CREAT|0777)) == -1) 
		{
			perror("msgget error");
			exit(1);
		}
	
		// 打印消息队列ID及进程ID
	    printf("My msqid is: %d.\n", msqid);
		printf("My pid is: %d.\n", getpid());
	
		// 添加消息，类型为888
		msg.mtype = 888;
		sprintf(msg.mtext, "hello, I'm client %d", getpid());
		msgsnd(msqid, &msg, sizeof(msg.mtext), 0);
	
		// 读取类型为999的消息
		msgrcv(msqid, &msg, 256, 999, 0);
		printf("Client: receive msg.mtext is: %s.\n", msg.mtext);
		printf("Client: receive msg.mtype is: %d.\n", msg.mtype);
		return 0;
	}


在两个终端里用 gcc 分别编译运行上面两个文件，可以看到输出结果如下：

	root@ubuntu:~/mytest# ./msg_server 
	Message Queue - Server key is: 2046888585.
	My msqid is: 0.
	My pid is: 9703.
	Server: receive msg.mtext is: hello, I'm client 9726.
	Server: receive msg.mtype is: 888.

---
	root@ubuntu:~/mytest# ./msg_client 
	Message Queue - Client key is: 2046888585.
	My msqid is: 0.
	My pid is: 9726.
	Client: receive msg.mtext is: hello, I'm server 9697.
	Client: receive msg.mtype is: 999.

# 四、信号量 #
**信号量（semaphore）**与已经介绍过的 IPC 结构不同，它是一个**计数器**。信号量用于实现进程间的互斥与同步，而不是用于存储进程间通信数据。
## 特点 ##

1. 信号量用于进程间同步，若要在进程间传递数据需要结合共享内存。
2. 信号量基于操作系统的 PV 操作，程序对信号量的操作都是原子操作。
3. 每次对信号量的 PV 操作不仅限于对信号量值加 1 或减 1，而且可以加减任意正整数。
4. 支持信号量组。

最简单的信号量是只能取 0 和 1 的变量，这也是信号量最常见的一种形式，叫做二值信号量（Binary Semaphore）。而可以取多个正整数的信号量被称为通用信号量。

## 相关API ##

Linux 下的信号量函数都是在通用的信号量数组上进行操作，而不是在一个单一的二值信号量上进行操作。

	#include <sys/sem.h>
	// 创建或获取一个信号量组：若成功返回信号量集ID，失败返回-1
	int semget(key_t key, int num_sems, int sem_flags);
	// 对信号量组进行操作，改变信号量的值：成功返回0，失败返回-1
	int semop(int semid, struct sembuf semoparray[], size_t numops);  
	// 控制信号量的相关信息
	int semctl(int semid, int sem_num, int cmd, ...);

当semget创建新的信号量集合时，必须指定集合中信号量的个数（即num_sems），通常为1；如果是引用一个现有的集合，则将num_sems指定为 0 。

在semop函数中，sembuf结构的定义如下：

	struct sembuf 
	{
	    short sem_num; // 信号量组中对应的序号，0～sem_nums-1
	    short sem_op;  // 信号量值在一次操作中的改变量
	    short sem_flg; // IPC_NOWAIT, SEM_UNDO
	}

其中 sem_op 是一次操作中的信号量的改变量：

- 若sem_op > 0，表示进程释放相应的资源数，将 sem_op 的值加到信号量的值上。如果有进程正在休眠等待此信号量，则换行它们。

- 若sem_op < 0，请求 sem_op 的绝对值的资源。

  - 如果相应的资源数可以满足请求，则将该信号量的值减去sem_op的绝对值，函数成功返回。
  - 当相应的资源数不能满足请求时，这个操作与sem_flg有关。
     - sem_flg 指定IPC_NOWAIT，则semop函数出错返回EAGAIN。
     - sem_flg 没有指定IPC_NOWAIT，则将该信号量的semncnt值加1，然后进程挂起直到下述情况发生：
          - 当相应的资源数可以满足请求，此信号量的semncnt值减1，该信号量的值减去sem_op的绝对值。成功返回；
          - 此信号量被删除，函数smeop出错返回EIDRM；
          - 进程捕捉到信号，并从信号处理函数返回，此情况下将此信号量的semncnt值减1，函数semop出错返回EINTR
- 若sem_op == 0，进程阻塞直到信号量的相应值为0：

  - 当信号量已经为0，函数立即返回。
  - 如果信号量的值不为0，则依据sem_flg决定函数动作：
     - sem_flg指定IPC_NOWAIT，则出错返回EAGAIN。
     - sem_flg没有指定IPC_NOWAIT，则将该信号量的semncnt值加1，然后进程挂起直到下述情况发生：
         - 信号量值为0，将信号量的semzcnt的值减1，函数semop成功返回；
         - 此信号量被删除，函数smeop出错返回EIDRM；
         - 进程捕捉到信号，并从信号处理函数返回，在此情况将此信号量的semncnt值减1，函数semop出错返回EINTR
      
在semctl函数中的命令有多种，这里就说两个常用的：

- SETVAL：用于初始化信号量为一个已知的值。所需要的值作为联合semun的val成员来传递。在信号量第一次使用之前需要设置信号量。

- IPC_RMID：删除一个信号量集合。如果不删除信号量，它将继续在系统中存在，即使程序已经退出，它可能在你下次运行此程序时引发问题，而且信号量是一种有限的资源。

## 示例代码 ##

	#include<stdio.h>
	#include<stdlib.h>
	#include<sys/sem.h>
	
	// 联合体，用于semctl初始化
	union semun
	{
		int              val; /*for SETVAL*/
		struct semid_ds *buf;
		unsigned short  *array;
	};
	
	// 初始化信号量
	int init_sem(int sem_id, int value)
	{
		union semun tmp;
		tmp.val = value;
		if(semctl(sem_id, 0, SETVAL, tmp) == -1)
		{
			perror("Init Semaphore Error");
			return -1;
		}
		return 0;
	}
	
	// P操作:
	//	若信号量值为1，获取资源并将信号量值-1 
	//	若信号量值为0，进程挂起等待
	int sem_p(int sem_id)
	{
		struct sembuf sbuf;
		sbuf.sem_num = 0; /*序号*/
		sbuf.sem_op = -1; /*P操作*/
		sbuf.sem_flg = SEM_UNDO;
	
		if(semop(sem_id, &sbuf, 1) == -1)
		{
			perror("P operation Error");
			return -1;
		}
		return 0;
	}
	
	// V操作：
	//	释放资源并将信号量值+1
	//	如果有进程正在挂起等待，则唤醒它们
	int sem_v(int sem_id)
	{
		struct sembuf sbuf;
		sbuf.sem_num = 0; /*序号*/
		sbuf.sem_op = 1;  /*V操作*/
		sbuf.sem_flg = SEM_UNDO;
	
		if(semop(sem_id, &sbuf, 1) == -1)
		{
			perror("V operation Error");
			return -1;
		}
		return 0;
	}
	
	// 删除信号量集
	int del_sem(int sem_id)
	{
		union semun tmp;
		if(semctl(sem_id, 0, IPC_RMID, tmp) == -1)
		{
			perror("Delete Semaphore Error");
			return -1;
		}
		return 0;
	}
	
	
	int main()
	{
		int sem_id;  // 信号量集ID
		key_t key;  
		pid_t pid;
	
		// 获取key值
		if((key = ftok(".", 'z')) < 0)
		{
			perror("ftok error");
			exit(1);
		}
	
		// 创建信号量集，其中只有一个信号量
		if((sem_id = semget(key, 1, IPC_CREAT|0666)) == -1)
		{
			perror("semget error");
			exit(1);
		}
	
		// 初始化：初值设为0资源被占用
		init_sem(sem_id, 0);
	
		if((pid = fork()) == -1)
			perror("Fork Error");
		else if(pid == 0) /*子进程*/ 
		{
			sleep(2);
			printf("Process child: pid=%d\n", getpid());
			sem_v(sem_id);  /*释放资源*/
		}
		else  /*父进程*/
		{
			sem_p(sem_id);   /*等待资源*/
			printf("Process father: pid=%d\n", getpid());
			sem_v(sem_id);   /*释放资源*/
			del_sem(sem_id); /*删除信号量集*/
		}
		return 0;
	}

输出结果：
	
	root@ubuntu:~/mytest# ./sem 
	Process child: pid=9943
	Process father: pid=9942

上面的例子如果不加信号量，则父进程会先执行完毕。这里加了信号量让父进程等待子进程执行完以后再执行。

# 五、共享内存 #
**共享内存（Shared Memory）**，指两个或多个进程共享一个给定的存储区。
## 特点 ##
1. 共享内存是最快的一种 IPC，因为进程是直接对内存进行存取。
2. 因为多个进程可以同时操作，所以需要进行同步。
3. 信号量+共享内存通常结合在一起使用，信号量用来同步对共享内存的访问

## 相关API ##
	
	#include <sys/shm.h>
	// 创建或获取一个共享内存：成功返回共享内存ID，失败返回-1
	int shmget(key_t key, size_t size, int flag);
	// 连接共享内存到当前进程的地址空间：成功返回指向共享内存的指针，失败返回-1
	void *shmat(int shm_id, const void *addr, int flag);
	// 断开与共享内存的连接：成功返回0，失败返回-1
	int shmdt(void *addr); 
	// 控制共享内存的相关信息：成功返回0，失败返回-1
	int shmctl(int shm_id, int cmd, struct shmid_ds *buf);


当用shmget函数创建一段共享内存时，必须指定其 size；而如果引用一个已存在的共享内存，则将 size 指定为0 。

当一段共享内存被创建以后，它并不能被任何进程访问。必须使用shmat函数连接该共享内存到当前进程的地址空间，连接成功后把共享内存区对象映射到调用进程的地址空间，随后可像本地空间一样访问。

shmdt函数是用来断开shmat建立的连接的。注意，这并不是从系统中删除该共享内存，只是当前进程不能再访问该共享内存而已。

shmctl函数可以对共享内存执行多种操作，根据参数 cmd 执行相应的操作。常用的是IPC_RMID（从系统中删除该共享内存）。



## 示例代码 ##

下面这个例子，使用了【共享内存+信号量+消息队列】的组合来实现服务器进程与客户进程间的通信。

- 共享内存用来传递数据；
- 信号量用来同步；
- 消息队列用来 在客户端修改了共享内存后 通知服务器读取。

***shmServer.c***

	#include<stdio.h>
	#include<stdlib.h>
	#include<sys/shm.h>  // shared memory
	#include<sys/sem.h>  // semaphore
	#include<sys/msg.h>  // message queue
	#include<string.h>   // memcpy
	
	// 消息队列结构
	struct msg_form {
	    long mtype;
	    char mtext;
	};
	
	// 联合体，用于semctl初始化
	union semun
	{
	    int              val; /*for SETVAL*/
	    struct semid_ds *buf;
	    unsigned short  *array;
	};
	
	// 初始化信号量
	int init_sem(int sem_id, int value)
	{
	    union semun tmp;
	    tmp.val = value;
	    if(semctl(sem_id, 0, SETVAL, tmp) == -1)
	    {
	        perror("Init Semaphore Error");
	        return -1;
	    }
	    return 0;
	}
	
	// P操作:
	//  若信号量值为1，获取资源并将信号量值-1 
	//  若信号量值为0，进程挂起等待
	int sem_p(int sem_id)
	{
	    struct sembuf sbuf;
	    sbuf.sem_num = 0; /*序号*/
	    sbuf.sem_op = -1; /*P操作*/
	    sbuf.sem_flg = SEM_UNDO;
	
	    if(semop(sem_id, &sbuf, 1) == -1)
	    {
	        perror("P operation Error");
	        return -1;
	    }
	    return 0;
	}
	
	// V操作：
	//  释放资源并将信号量值+1
	//  如果有进程正在挂起等待，则唤醒它们
	int sem_v(int sem_id)
	{
	    struct sembuf sbuf;
	    sbuf.sem_num = 0; /*序号*/
	    sbuf.sem_op = 1;  /*V操作*/
	    sbuf.sem_flg = SEM_UNDO;
	
	    if(semop(sem_id, &sbuf, 1) == -1)
	    {
	        perror("V operation Error");
	        return -1;
	    }
	    return 0;
	}
	
	// 删除信号量集
	int del_sem(int sem_id)
	{
	    union semun tmp;
	    if(semctl(sem_id, 0, IPC_RMID, tmp) == -1)
	    {
	        perror("Delete Semaphore Error");
	        return -1;
	    }
	    return 0;
	}
	
	// 创建一个信号量集
	int creat_sem(key_t key)
	{
		int sem_id;
		if((sem_id = semget(key, 1, IPC_CREAT|0666)) == -1)
		{
			perror("semget error");
			exit(-1);
		}
		init_sem(sem_id, 1);  /*初值设为1资源未占用*/
		return sem_id;
	}
	
	
	int main()
	{
		key_t key;
		int shmid, semid, msqid;
		char *shm;
		char data[] = "this is server";
		struct shmid_ds buf1;  /*用于删除共享内存*/
		struct msqid_ds buf2;  /*用于删除消息队列*/
		struct msg_form msg;  /*消息队列用于通知对方更新了共享内存*/
	
		// 获取key值
		if((key = ftok(".", 'z')) < 0)
		{
			perror("ftok error");
			exit(1);
		}
	
		// 创建共享内存
		if((shmid = shmget(key, 1024, IPC_CREAT|0666)) == -1)
		{
			perror("Create Shared Memory Error");
			exit(1);
		}
	
		// 连接共享内存
		shm = (char*)shmat(shmid, 0, 0);
		if((int)shm == -1)
		{
			perror("Attach Shared Memory Error");
			exit(1);
		}
	
	
		// 创建消息队列
		if ((msqid = msgget(key, IPC_CREAT|0777)) == -1)
		{
			perror("msgget error");
			exit(1);
		}
	
		// 创建信号量
		semid = creat_sem(key);
		
		// 读数据
		while(1)
		{
			msgrcv(msqid, &msg, 1, 888, 0); /*读取类型为888的消息*/
			if(msg.mtext == 'q')  /*quit - 跳出循环*/ 
				break;
			if(msg.mtext == 'r')  /*read - 读共享内存*/
			{
				sem_p(semid);
				printf("%s\n",shm);
				sem_v(semid);
			}
		}
	
		// 断开连接
		shmdt(shm);
	
	    /*删除共享内存、消息队列、信号量*/
		shmctl(shmid, IPC_RMID, &buf1);
		msgctl(msqid, IPC_RMID, &buf2);
		del_sem(semid);
		return 0;
	}

***shmClient.c***

	#include<stdio.h>
	#include<stdlib.h>
	#include<sys/shm.h>  // shared memory
	#include<sys/sem.h>  // semaphore
	#include<sys/msg.h>  // message queue
	#include<string.h>   // memcpy
	
	// 消息队列结构
	struct msg_form {
	    long mtype;
	    char mtext;
	};
	
	// 联合体，用于semctl初始化
	union semun
	{
	    int              val; /*for SETVAL*/
	    struct semid_ds *buf;
	    unsigned short  *array;
	};
	
	// P操作:
	//  若信号量值为1，获取资源并将信号量值-1 
	//  若信号量值为0，进程挂起等待
	int sem_p(int sem_id)
	{
	    struct sembuf sbuf;
	    sbuf.sem_num = 0; /*序号*/
	    sbuf.sem_op = -1; /*P操作*/
	    sbuf.sem_flg = SEM_UNDO;
	
	    if(semop(sem_id, &sbuf, 1) == -1)
	    {
	        perror("P operation Error");
	        return -1;
	    }
	    return 0;
	}
	
	// V操作：
	//  释放资源并将信号量值+1
	//  如果有进程正在挂起等待，则唤醒它们
	int sem_v(int sem_id)
	{
	    struct sembuf sbuf;
	    sbuf.sem_num = 0; /*序号*/
	    sbuf.sem_op = 1;  /*V操作*/
	    sbuf.sem_flg = SEM_UNDO;
	
	    if(semop(sem_id, &sbuf, 1) == -1)
	    {
	        perror("V operation Error");
	        return -1;
	    }
	    return 0;
	}
	
	
	int main()
	{
		key_t key;
		int shmid, semid, msqid;
		char *shm;
		struct msg_form msg;
		int flag = 1; /*while循环条件*/
	
		// 获取key值
		if((key = ftok(".", 'z')) < 0)
		{
			perror("ftok error");
			exit(1);
		}
	
		// 获取共享内存
		if((shmid = shmget(key, 1024, 0)) == -1)
		{
			perror("shmget error");
			exit(1);
		}
	
		// 连接共享内存
		shm = (char*)shmat(shmid, 0, 0);
		if((int)shm == -1)
		{
			perror("Attach Shared Memory Error");
			exit(1);
		}
	
		// 创建消息队列
		if ((msqid = msgget(key, 0)) == -1)
		{
			perror("msgget error");
			exit(1);
		}
	
		// 获取信号量
		if((semid = semget(key, 0, 0)) == -1)
		{
			perror("semget error");
			exit(1);
		}
		
		// 写数据
		printf("***************************************\n");
		printf("*                 IPC                 *\n");
		printf("*    Input r to send data to server.  *\n");
		printf("*    Input q to quit.                 *\n");
		printf("***************************************\n");
		
		while(flag)
		{
			char c;
			printf("Please input command: ");
			scanf("%c", &c);
			switch(c)
			{
				case 'r':
					printf("Data to send: ");
					sem_p(semid);  /*访问资源*/
					scanf("%s", shm);
					sem_v(semid);  /*释放资源*/
					/*清空标准输入缓冲区*/
					while((c=getchar())!='\n' && c!=EOF);
					msg.mtype = 888;  
					msg.mtext = 'r';  /*发送消息通知服务器读数据*/
					msgsnd(msqid, &msg, sizeof(msg.mtext), 0);
					break;
				case 'q':
					msg.mtype = 888;
					msg.mtext = 'q';
					msgsnd(msqid, &msg, sizeof(msg.mtext), 0);
					flag = 0;
					break;
				default:
					printf("Wrong input!\n");
					/*清空标准输入缓冲区*/
					while((c=getchar())!='\n' && c!=EOF);
			}
		}
	
		// 断开连接
		shmdt(shm);
	
		return 0;
	}


在两个终端里用 gcc 分别编译运行上面两个文件，可以看到输出结果如下：

	root@ubuntu:~/mytest# ./shmServer 
	majian1992.cc
	root@ubuntu:~/mytest# 

---
	
	root@ubuntu:~/mytest# ./shmClient 
	***************************************
	*                 IPC                 *
	*    Input r to send data to server.  *
	*    Input q to quit.                 *
	***************************************
	Please input command: r
	Data to send: majian1992.cc
	Please input command: q
	root@ubuntu:~/mytest# 

# 六、Socket #
Socket起源于Unix，而Unix/Linux基本哲学之一就是“一切皆文件”，都可以用“打开open –> 读写write/read –> 关闭close”模式来操作。我的理解就是Socket就是该模式的一个实现，socket即是一种特殊的文件，一些socket函数就是对其进行的操作（读/写IO、打开、关闭）。
## 特点 ##
前面说到的进程间的通信，所通信的进程都是在同一台计算机上的，而使用socket进行通信的进程可以是同一台计算机的进程，也是可以是通过网络连接起来的不同计算机上的进程。

## 相关API ##

	#include <sys/types.h>
	#include <sys/socket.h>
	#include <unistd.h>
	int socket(int domain, int type, int protocol);
	int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
	int listen(int sockfd, int backlog);
	int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
	int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);


	ssize_t read(int fd, void *buf, size_t count);
	ssize_t write(int fd, const void *buf, size_t count);
	
	
	ssize_t send(int sockfd, const void *buf, size_t len, int flags);
	ssize_t recv(int sockfd, void *buf, size_t len, int flags);
	
	ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
	               const struct sockaddr *dest_addr, socklen_t addrlen);
	ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
	                 struct sockaddr *src_addr, socklen_t *addrlen);
	
	ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
	ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);\


socket函数对应于普通文件的打开操作。普通文件的打开操作返回一个文件描述字，而socket()用于创建一个socket描述符（socket descriptor），它唯一标识一个socket。这个socket描述字跟文件描述字一样，后续的操作都有用到它，把它作为参数，通过它来进行一些读写操作。

正如可以给fopen的传入不同参数值，以打开不同的文件。创建socket的时候，也可以指定不同的参数创建不同的socket描述符，socket函数的三个参数分别为：

domain：即协议域，又称为协议族（family）。常用的协议族有，AF_INET、AF_INET6、AF_LOCAL（或称AF_UNIX，Unix域socket）、AF_ROUTE等等。协议族决定了socket的地址类型，在通信中必须采用对应的地址，如AF_INET决定了要用ipv4地址（32位的）与端口号（16位的）的组合、AF_UNIX决定了要用一个绝对路径名作为地址。
type：指定socket类型。常用的socket类型有，SOCK_STREAM、SOCK_DGRAM、SOCK_RAW、SOCK_PACKET、SOCK_SEQPACKET等等（socket的类型有哪些？）。
protocol：故名思意，就是指定协议。常用的协议有，IPPROTO_TCP、IPPTOTO_UDP、IPPROTO_SCTP、IPPROTO_TIPC等，它们分别对应TCP传输协议、UDP传输协议、STCP传输协议、TIPC传输协议（这个协议我将会单独开篇讨论！）。
注意：并不是上面的type和protocol可以随意组合的，如SOCK_STREAM不可以跟IPPROTO_UDP组合。当protocol为0时，会自动选择type类型对应的默认协议。

当我们调用socket创建一个socket时，返回的socket描述字它存在于协议族（address family，AF_XXX）空间中，但没有一个具体的地址。如果想要给它赋值一个地址，就必须调用bind()函数，否则就当调用connect()、listen()时系统会自动随机分配一个端口。


## 示例代码 ##

***TCPServer.c***

	#include<stdio.h>
	#include<stdlib.h>
	#include<string.h>
	#include<errno.h>
	#include<sys/types.h>
	#include<sys/socket.h>
	#include<netinet/in.h>
	
	#define MAXLINE 4096
	
	int main(int argc, char** argv)
	{
	    int    listenfd, connfd;
	    struct sockaddr_in     servaddr;
	    char    buff[4096];
	    int     n;
	
	    if( (listenfd = socket(AF_INET, SOCK_STREAM, 0)) == -1 ){
	    printf("create socket error: %s(errno: %d)\n",strerror(errno),errno);
	    exit(0);
	    }
	
	    memset(&servaddr, 0, sizeof(servaddr));
	    servaddr.sin_family = AF_INET;
	    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	    servaddr.sin_port = htons(6666);
	
	    if( bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) == -1){
	    printf("bind socket error: %s(errno: %d)\n",strerror(errno),errno);
	    exit(0);
	    }
	
	    if( listen(listenfd, 10) == -1){
	    printf("listen socket error: %s(errno: %d)\n",strerror(errno),errno);
	    exit(0);
	    }
	
	    printf("======waiting for client's request======\n");
	    while(1){
	    if( (connfd = accept(listenfd, (struct sockaddr*)NULL, NULL)) == -1){
	        printf("accept socket error: %s(errno: %d)",strerror(errno),errno);
	        continue;
	    }
	    n = recv(connfd, buff, MAXLINE, 0);
	    buff[n] = '\0';
	    printf("recv msg from client: %s\n", buff);
	    close(connfd);
	    }
	
	    close(listenfd);
	}

***TCPClient.c***

	#include<stdio.h>
	#include<stdlib.h>
	#include<string.h>
	#include<errno.h>
	#include<sys/types.h>
	#include<sys/socket.h>
	#include<netinet/in.h>
	
	#define MAXLINE 4096
	
	int main(int argc, char** argv)
	{
	    int    sockfd, n;
	    char    recvline[4096], sendline[4096];
	    struct sockaddr_in    servaddr;
	
	    if( argc != 2){
	    printf("usage: ./client <ipaddress>\n");
	    exit(0);
	    }
	
	    if( (sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0){
	    printf("create socket error: %s(errno: %d)\n", strerror(errno),errno);
	    exit(0);
	    }
	
	    memset(&servaddr, 0, sizeof(servaddr));
	    servaddr.sin_family = AF_INET;
	    servaddr.sin_port = htons(6666);
	    if( inet_pton(AF_INET, argv[1], &servaddr.sin_addr) <= 0){
	    printf("inet_pton error for %s\n",argv[1]);
	    exit(0);
	    }
	
	    if( connect(sockfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0){
	    printf("connect error: %s(errno: %d)\n",strerror(errno),errno);
	    exit(0);
	    }
	
	    printf("send msg to server: \n");
	    fgets(sendline, 4096, stdin);
	    if( send(sockfd, sendline, strlen(sendline), 0) < 0)
	    {
	    printf("send msg error: %s(errno: %d)\n", strerror(errno), errno);
	    exit(0);
	    }
	
	    close(sockfd);
	    exit(0);
	}

当然上面的代码很简单，也有很多缺点，这就只是简单的演示socket的基本函数使用。其实不管有多复杂的网络程序，都使用的这些基本函数。上面的服务器使用的是迭代模式的，即只有处理完一个客户端请求才会去处理下一个客户端的请求，这样的服务器处理能力是很弱的，现实中的服务器都需要有并发处理能力！为了需要并发处理，服务器需要fork()一个新的进程或者线程去处理请求等。

# 参考 #
[Linux fork()返回值说明](http://blog.csdn.net/lovenankai/article/details/6874475)

[进程间通信（IPC）](http://songlee24.github.io/2015/04/21/linux-IPC/)

[深刻理解Linux进程间通信（IPC）](https://www.ibm.com/developerworks/cn/linux/l-ipc/)

[Linux Socket编程（不限Linux）](http://www.cnblogs.com/skynet/archive/2010/12/12/1903949.html)

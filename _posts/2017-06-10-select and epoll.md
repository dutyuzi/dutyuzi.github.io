---
layout:     post
title:      "selsec、poll与epoll的区别"
subtitle:   "The difference between selsec, poll and epoll"
date:        2017/06/10  10:01:00 
author:     "MaJ"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - other
---
---
select，poll，epoll都是IO多路复用的机制。I/O多路复用就通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作.
但select，poll，epoll本质上都是**同步I/O**，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

与多进程和多线程技术相比，I/O多路复用技术的最大优势是系统开销小，系统不必创建进程/线程，也不必维护这些进程/线程，从而大大减小了系统的开销。

三种IO模型，都属于多路IO就绪通知，提供了对大量文件描述符就绪检查的高性能方案，只不过实现方式有所不同：
### select 
select 调用如下图所示

![img](/img/select.jpg)

**select函数**

	#include <sys/select.h>
	#include <sys/time.h>
	
	int select(int maxfd,fd_set *readset,fd_set *writeset,fd_set *exceptset,const struct timeval *timeout)
	返回值：就绪描述符的数目，超时返回0，出错返回-1

函数参数介绍如下：

- 第一个参数maxfd指定待测试的描述字个数，它的值是待测试的最大描述字加1，描述字0、1、2...maxfd-1均将被测试因为文件描述符是从0开始的。
- 中间的三个参数readset、writeset和exceptset指定我们要让内核测试读、写和异常条件的描述字。如果对某一个的条件不感兴趣，就可以把它设为空指针。struct fd_set可以理解为一个集合，这个集合中存放的是文件描述符，可通过以下四个宏进行设置：

          void FD_ZERO(fd_set *fdset);           //清空集合

          void FD_SET(int fd, fd_set *fdset);   //将一个给定的文件描述符加入集合之中

          void FD_CLR(int fd, fd_set *fdset);   //将一个给定的文件描述符从集合中删除

          int FD_ISSET(int fd, fd_set *fdset);   // 检查集合中指定的文件描述符是否可以读写 

- timeout告知内核等待所指定描述字中的任何一个就绪可花多少时间。其timeval结构用于指定这段时间的秒数和微秒数。

         struct timeval{

                   long tv_sec;   //seconds

                   long tv_usec;  //microseconds

       };

这个参数有三种可能：

（1）永远等待下去：仅在有一个描述字准备好I/O时才返回。为此，把该参数设置为空指针NULL。

（2）等待一段固定时间：在有一个描述字准备好I/O时返回，但是不超过由该参数所指向的timeval结构中指定的秒数和微秒数。

（3）根本不等待：检查描述字后立即返回，这称为轮询。为此，该参数必须指向一个timeval结构，而且其中的定时器值必须为0。

**测试程序**：

服务器端：

	#include <stdio.h>
	#include <stdlib.h>
	#include <string.h>
	#include <errno.h>
	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <sys/select.h>
	#include <sys/types.h>
	#include <netinet/in.h>
	#include <arpa/inet.h>
	#include <unistd.h>
	#include <assert.h>
	
	#define IPADDR      "127.0.0.1"
	#define PORT        8787
	#define MAXLINE     1024
	#define LISTENQ     5
	#define SIZE        10
	
	typedef struct server_context_st
	{
	    int cli_cnt;        /*客户端个数*/
	    int clifds[SIZE];   /*客户端的个数*/
	    fd_set allfds;      /*句柄集合*/
	    int maxfd;          /*句柄最大值*/
	} server_context_st;
	static server_context_st *s_srv_ctx = NULL;
	/*===========================================================================
	 * ==========================================================================*/
	static int create_server_proc(const char* ip,int port)
	{
	    int  fd;
	    struct sockaddr_in servaddr;
	    fd = socket(AF_INET, SOCK_STREAM,0);
	    if (fd == -1) {
	        fprintf(stderr, "create socket fail,erron:%d,reason:%s\n",
	                errno, strerror(errno));
	        return -1;
	    }
	
	    /*一个端口释放后会等待两分钟之后才能再被使用，SO_REUSEADDR是让端口释放后立即就可以被再次使用。*/
	    int reuse = 1;
	    if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse)) == -1) {
	        return -1;
	    }
	
	    bzero(&servaddr,sizeof(servaddr));
	    servaddr.sin_family = AF_INET;
	    inet_pton(AF_INET,ip,&servaddr.sin_addr);
	    servaddr.sin_port = htons(port);
	
	    if (bind(fd,(struct sockaddr*)&servaddr,sizeof(servaddr)) == -1) {
	        perror("bind error: ");
	        return -1;
	    }
	
	    listen(fd,LISTENQ);
	
	    return fd;
	}
	
	static int accept_client_proc(int srvfd)
	{
	    struct sockaddr_in cliaddr;
	    socklen_t cliaddrlen;
	    cliaddrlen = sizeof(cliaddr);
	    int clifd = -1;
	
	    printf("accpet clint proc is called.\n");
	
	ACCEPT:
	    clifd = accept(srvfd,(struct sockaddr*)&cliaddr,&cliaddrlen);
	
	    if (clifd == -1) {
	        if (errno == EINTR) {
	            goto ACCEPT;
	        } else {
	            fprintf(stderr, "accept fail,error:%s\n", strerror(errno));
	            return -1;
	        }
	    }
	
	    fprintf(stdout, "accept a new client: %s:%d\n",
	            inet_ntoa(cliaddr.sin_addr),cliaddr.sin_port);
	
	    //将新的连接描述符添加到数组中
	    int i = 0;
	    for (i = 0; i < SIZE; i++) {
	        if (s_srv_ctx->clifds[i] < 0) {
	            s_srv_ctx->clifds[i] = clifd;
	            s_srv_ctx->cli_cnt++;
	            break;
	        }
	    }
	
	    if (i == SIZE) {
	        fprintf(stderr,"too many clients.\n");
	        return -1;
	    }
	101 }
	
	static int handle_client_msg(int fd, char *buf) 
	{
	    assert(buf);
	    printf("recv buf is :%s\n", buf);
	    write(fd, buf, strlen(buf) +1);
	    return 0;
	}
	
	static void recv_client_msg(fd_set *readfds)
	{
	    int i = 0, n = 0;
	    int clifd;
	    char buf[MAXLINE] = {0};
	    for (i = 0;i <= s_srv_ctx->cli_cnt;i++) {
	        clifd = s_srv_ctx->clifds[i];
	        if (clifd < 0) {
	            continue;
	        }
	        /*判断客户端套接字是否有数据*/
	        if (FD_ISSET(clifd, readfds)) {
	            //接收客户端发送的信息
	            n = read(clifd, buf, MAXLINE);
	            if (n <= 0) {
	                /*n==0表示读取完成，客户都关闭套接字*/
	                FD_CLR(clifd, &s_srv_ctx->allfds);
	                close(clifd);
	                s_srv_ctx->clifds[i] = -1;
	                continue;
	            }
	            handle_client_msg(clifd, buf);
	        }
	    }
	}
	static void handle_client_proc(int srvfd)
	{
	    int  clifd = -1;
	    int  retval = 0;
	    fd_set *readfds = &s_srv_ctx->allfds;
	    struct timeval tv;
	    int i = 0;
	
	    while (1) {
	        /*每次调用select前都要重新设置文件描述符和时间，因为事件发生后，文件描述符和时间都被内核修改啦*/
	        FD_ZERO(readfds);
	        /*添加监听套接字*/
	        FD_SET(srvfd, readfds);
	        s_srv_ctx->maxfd = srvfd;
	
	        tv.tv_sec = 30;
	        tv.tv_usec = 0;
	        /*添加客户端套接字*/
	        for (i = 0; i < s_srv_ctx->cli_cnt; i++) {
	            clifd = s_srv_ctx->clifds[i];
	            /*去除无效的客户端句柄*/
	            if (clifd != -1) {
	                FD_SET(clifd, readfds);
	            }
	            s_srv_ctx->maxfd = (clifd > s_srv_ctx->maxfd ? clifd : s_srv_ctx->maxfd);
	        }
	
	        /*开始轮询接收处理服务端和客户端套接字*/
	        retval = select(s_srv_ctx->maxfd + 1, readfds, NULL, NULL, &tv);
	        if (retval == -1) {
	            fprintf(stderr, "select error:%s.\n", strerror(errno));
	            return;
	        }
	        if (retval == 0) {
	            fprintf(stdout, "select is timeout.\n");
	            continue;
	        }
	        if (FD_ISSET(srvfd, readfds)) {
	            /*监听客户端请求*/
	            accept_client_proc(srvfd);
	        } else {
	            /*接受处理客户端消息*/
	            recv_client_msg(readfds);
	        }
	    }
	}
	
	static void server_uninit()
	{
	    if (s_srv_ctx) {
	        free(s_srv_ctx);
	        s_srv_ctx = NULL;
	    }
	}
	
	static int server_init()
	{
	    s_srv_ctx = (server_context_st *)malloc(sizeof(server_context_st));
	    if (s_srv_ctx == NULL) {
	        return -1;
	    }
	
	    memset(s_srv_ctx, 0, sizeof(server_context_st));
	
	    int i = 0;
	    for (;i < SIZE; i++) {
	        s_srv_ctx->clifds[i] = -1;
	    }
	
	    return 0;
	}
	
	int main(int argc,char *argv[])
	{
	    int  srvfd;
	    /*初始化服务端context*/
	    if (server_init() < 0) {
	        return -1;
	    }
	    /*创建服务,开始监听客户端请求*/
	    srvfd = create_server_proc(IPADDR, PORT);
	    if (srvfd < 0) {
	        fprintf(stderr, "socket create or bind fail.\n");
	        goto err;
	    }
	    /*开始接收并处理客户端请求*/
	    handle_client_proc(srvfd);
	    server_uninit();
	    return 0;
	err:
	    server_uninit();
	    return -1;
	}

客户端：

	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <stdio.h>
	#include <string.h>
	#include <stdlib.h>
	#include <sys/select.h>
	#include <time.h>
	#include <unistd.h>
	#include <sys/types.h>
	#include <errno.h>
	
	#define MAXLINE 1024
	#define IPADDRESS "127.0.0.1"
	#define SERV_PORT 8787
	
	#define max(a,b) (a > b) ? a : b
	
	static void handle_recv_msg(int sockfd, char *buf) 
	{
	printf("client recv msg is:%s\n", buf);
	sleep(5);
	write(sockfd, buf, strlen(buf) +1);
	}
	
	static void handle_connection(int sockfd)
	{
	char sendline[MAXLINE],recvline[MAXLINE];
	int maxfdp,stdineof;
	fd_set readfds;
	int n;
	struct timeval tv;
	int retval = 0;
	
	while (1) {
	
	FD_ZERO(&readfds);
	FD_SET(sockfd,&readfds);
	maxfdp = sockfd;
	
	tv.tv_sec = 5;
	tv.tv_usec = 0;
	
	retval = select(maxfdp+1,&readfds,NULL,NULL,&tv);
	
	if (retval == -1) {
	return ;
	}
	
	if (retval == 0) {
	printf("client timeout.\n");
	continue;
	}
	
	if (FD_ISSET(sockfd, &readfds)) {
	n = read(sockfd,recvline,MAXLINE);
	if (n <= 0) {
	fprintf(stderr,"client: server is closed.\n");
	close(sockfd);
	FD_CLR(sockfd,&readfds);
	return;
	}
	
	handle_recv_msg(sockfd, recvline);
	}
	}
	}
	
	int main(int argc,char *argv[])
	{
	int sockfd;
	struct sockaddr_in servaddr;
	
	sockfd = socket(AF_INET,SOCK_STREAM,0);
	
	bzero(&servaddr,sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(SERV_PORT);
	inet_pton(AF_INET,IPADDRESS,&servaddr.sin_addr);
	
	int retval = 0;
	retval = connect(sockfd,(struct sockaddr*)&servaddr,sizeof(servaddr));
	if (retval < 0) {
	fprintf(stderr, "connect fail,error:%s\n", strerror(errno));
	return -1;
	}
	
	printf("client send to server .\n");
	write(sockfd, "hello server", 32);
	
	handle_connection(sockfd);
	
	return 0;
	}

select的几大缺点：

- 每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大

- 同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大

- select支持的文件描述符数量太小了，默认是1024

### poll

poll的机制与select类似，与select在本质上没有多大差别，管理多个描述符也是进行轮询，根据描述符的状态进行处理，但是poll没有最大文件描述符数量的限制。poll和select同样存在一个缺点就是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。

**poll函数**

	# include <poll.h>
	int poll ( struct pollfd * fds, unsigned int nfds, int timeout);

pollfd结构体定义如下：

	struct pollfd {
	
	int fd;         /* 文件描述符 */
	short events;         /* 等待的事件 */
	short revents;       /* 实际发生了的事件 */
	} ; 

**测试程序**：

服务器端程序如下：

	#include <stdio.h>
	#include <stdlib.h>
	#include <string.h>
	#include <errno.h>
	
	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <poll.h>
	#include <unistd.h>
	#include <sys/types.h>
	
	#define IPADDRESS   "127.0.0.1"
	#define PORT        8787
	#define MAXLINE     1024
	#define LISTENQ     5
	#define OPEN_MAX    1000
	#define INFTIM      -1
	
	//函数声明
	//创建套接字并进行绑定
	static int socket_bind(const char* ip,int port);
	//IO多路复用poll
	static void do_poll(int listenfd);
	//处理多个连接
	static void handle_connection(struct pollfd *connfds,int num);
	
	int main(int argc,char *argv[])
	{
	    int  listenfd,connfd,sockfd;
	    struct sockaddr_in cliaddr;
	    socklen_t cliaddrlen;
	    listenfd = socket_bind(IPADDRESS,PORT);
	    listen(listenfd,LISTENQ);
	    do_poll(listenfd);
	    return 0;
	}
	
	static int socket_bind(const char* ip,int port)
	{
	    int  listenfd;
	    struct sockaddr_in servaddr;
	    listenfd = socket(AF_INET,SOCK_STREAM,0);
	    if (listenfd == -1)
	    {
	        perror("socket error:");
	        exit(1);
	    }
	    bzero(&servaddr,sizeof(servaddr));
	    servaddr.sin_family = AF_INET;
	    inet_pton(AF_INET,ip,&servaddr.sin_addr);
	    servaddr.sin_port = htons(port);
	    if (bind(listenfd,(struct sockaddr*)&servaddr,sizeof(servaddr)) == -1)
	    {
	        perror("bind error: ");
	        exit(1);
	    }
	    return listenfd;
	}
	
	static void do_poll(int listenfd)
	{
	    int  connfd,sockfd;
	    struct sockaddr_in cliaddr;
	    socklen_t cliaddrlen;
	    struct pollfd clientfds[OPEN_MAX];
	    int maxi;
	    int i;
	    int nready;
	    //添加监听描述符
	    clientfds[0].fd = listenfd;
	    clientfds[0].events = POLLIN;
	    //初始化客户连接描述符
	    for (i = 1;i < OPEN_MAX;i++)
	        clientfds[i].fd = -1;
	    maxi = 0;
	    //循环处理
	    for ( ; ; )
	    {
	        //获取可用描述符的个数
	        nready = poll(clientfds,maxi+1,INFTIM);
	        if (nready == -1)
	        {
	            perror("poll error:");
	            exit(1);
	        }
	        //测试监听描述符是否准备好
	        if (clientfds[0].revents & POLLIN)
	        {
	            cliaddrlen = sizeof(cliaddr);
	            //接受新的连接
	            if ((connfd = accept(listenfd,(struct sockaddr*)&cliaddr,&cliaddrlen)) == -1)
	            {
	                if (errno == EINTR)
	                    continue;
	                else
	                {
	                   perror("accept error:");
	                   exit(1);
	                }
	            }
	            fprintf(stdout,"accept a new client: %s:%d\n", inet_ntoa(cliaddr.sin_addr),cliaddr.sin_port);
	            //将新的连接描述符添加到数组中
	            for (i = 1;i < OPEN_MAX;i++)
	            {
	                if (clientfds[i].fd < 0)
	                {
	                    clientfds[i].fd = connfd;
	                    break;
	                }
	            }
	            if (i == OPEN_MAX)
	            {
	                fprintf(stderr,"too many clients.\n");
	                exit(1);
	            }
	            //将新的描述符添加到读描述符集合中
	            clientfds[i].events = POLLIN;
	            //记录客户连接套接字的个数
	            maxi = (i > maxi ? i : maxi);
	            if (--nready <= 0)
	                continue;
	        }
	        //处理客户连接
	        handle_connection(clientfds,maxi);
	    }
	}
	
	static void handle_connection(struct pollfd *connfds,int num)
	{
	    int i,n;
	    char buf[MAXLINE];
	    memset(buf,0,MAXLINE);
	    for (i = 1;i <= num;i++)
	    {
	        if (connfds[i].fd < 0)
	            continue;
	        //测试客户描述符是否准备好
	        if (connfds[i].revents & POLLIN)
	        {
	            //接收客户端发送的信息
	            n = read(connfds[i].fd,buf,MAXLINE);
	            if (n == 0)
	            {
	                close(connfds[i].fd);
	                connfds[i].fd = -1;
	                continue;
	            }
	           // printf("read msg is: ");
	            write(STDOUT_FILENO,buf,n);
	            //向客户端发送buf
	            write(connfds[i].fd,buf,n);
	        }
	    }
	}

客户端代码如下：

	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <stdio.h>
	#include <string.h>
	#include <stdlib.h>
	#include <poll.h>
	#include <time.h>
	#include <unistd.h>
	#include <sys/types.h>
	
	#define MAXLINE     1024
	#define IPADDRESS   "127.0.0.1"
	#define SERV_PORT   8787
	
	#define max(a,b) (a > b) ? a : b
	
	static void handle_connection(int sockfd);
	
	int main(int argc,char *argv[])
	{
	    int                 sockfd;
	    struct sockaddr_in  servaddr;
	    sockfd = socket(AF_INET,SOCK_STREAM,0);
	    bzero(&servaddr,sizeof(servaddr));
	    servaddr.sin_family = AF_INET;
	    servaddr.sin_port = htons(SERV_PORT);
	    inet_pton(AF_INET,IPADDRESS,&servaddr.sin_addr);
	    connect(sockfd,(struct sockaddr*)&servaddr,sizeof(servaddr));
	    //处理连接描述符
	    handle_connection(sockfd);
	    return 0;
	}
	
	static void handle_connection(int sockfd)
	{
	    char    sendline[MAXLINE],recvline[MAXLINE];
	    int     maxfdp,stdineof;
	    struct pollfd pfds[2];
	    int n;
	    //添加连接描述符
	    pfds[0].fd = sockfd;
	    pfds[0].events = POLLIN;
	    //添加标准输入描述符
	    pfds[1].fd = STDIN_FILENO;
	    pfds[1].events = POLLIN;
	    for (; ;)
	    {
	        poll(pfds,2,-1);
	        if (pfds[0].revents & POLLIN)
	        {
	            n = read(sockfd,recvline,MAXLINE);
	            if (n == 0)
	            {
	                    fprintf(stderr,"client: server is closed.\n");
	                    close(sockfd);
	            }
	            write(STDOUT_FILENO,recvline,n);
	        }
	        //测试标准输入是否准备好
	        if (pfds[1].revents & POLLIN)
	        {
	            n = read(STDIN_FILENO,sendline,MAXLINE);
	            if (n  == 0)
	            {
	                shutdown(sockfd,SHUT_WR);
	        continue;
	            }
	            write(sockfd,sendline,n);
	        }
	    }
	}

### epoll

epoll是在2.6内核中提出的，是之前的select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

epoll操作过程需要三个接口，分别如下：

	#include <sys/epoll.h>
	int epoll_create(int size);
	int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
	int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

1） int epoll_create(int size);
　　创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大。这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

（2）int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
　　epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。第一个参数是epoll_create()的返回值，第二个参数表示动作，用三个宏来表示：
EPOLL_CTL_ADD：注册新的fd到epfd中；
EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
EPOLL_CTL_DEL：从epfd中删除一个fd；
第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：

	struct epoll_event {
	  __uint32_t events;  /* Epoll events */
	  epoll_data_t data;  /* User data variable */
	};

events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

（3） int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
　　等待事件的产生，类似于select()调用。参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。

3、工作模式

epoll对文件描述符的操作有两种模式：LT（level trigger）和ET（edge trigger）。LT模式是默认模式，LT模式与ET模式的区别如下：

- LT模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。

- ET模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。

ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

**测试程序**：

服务器代码如下：

	#include <stdio.h>
	#include <stdlib.h>
	#include <string.h>
	#include <errno.h>
	
	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <arpa/inet.h>
	#include <sys/epoll.h>
	#include <unistd.h>
	#include <sys/types.h>
	
	#define IPADDRESS   "127.0.0.1"
	#define PORT        8787
	#define MAXSIZE     1024
	#define LISTENQ     5
	#define FDSIZE      1000
	#define EPOLLEVENTS 100
	
	//函数声明
	//创建套接字并进行绑定
	static int socket_bind(const char* ip,int port);
	//IO多路复用epoll
	static void do_epoll(int listenfd);
	//事件处理函数
	static void
	handle_events(int epollfd,struct epoll_event *events,int num,int listenfd,char *buf);
	//处理接收到的连接
	static void handle_accpet(int epollfd,int listenfd);
	//读处理
	static void do_read(int epollfd,int fd,char *buf);
	//写处理
	static void do_write(int epollfd,int fd,char *buf);
	//添加事件
	static void add_event(int epollfd,int fd,int state);
	//修改事件
	static void modify_event(int epollfd,int fd,int state);
	//删除事件
	static void delete_event(int epollfd,int fd,int state);
	
	int main(int argc,char *argv[])
	{
	    int  listenfd;
	    listenfd = socket_bind(IPADDRESS,PORT);
	    listen(listenfd,LISTENQ);
	    do_epoll(listenfd);
	    return 0;
	}
	
	static int socket_bind(const char* ip,int port)
	{
	    int  listenfd;
	    struct sockaddr_in servaddr;
	    listenfd = socket(AF_INET,SOCK_STREAM,0);
	    if (listenfd == -1)
	    {
	        perror("socket error:");
	        exit(1);
	    }
	    bzero(&servaddr,sizeof(servaddr));
	    servaddr.sin_family = AF_INET;
	    inet_pton(AF_INET,ip,&servaddr.sin_addr);
	    servaddr.sin_port = htons(port);
	    if (bind(listenfd,(struct sockaddr*)&servaddr,sizeof(servaddr)) == -1)
	    {
	        perror("bind error: ");
	        exit(1);
	    }
	    return listenfd;
	}
	
	static void do_epoll(int listenfd)
	{
	    int epollfd;
	    struct epoll_event events[EPOLLEVENTS];
	    int ret;
	    char buf[MAXSIZE];
	    memset(buf,0,MAXSIZE);
	    //创建一个描述符
	    epollfd = epoll_create(FDSIZE);
	    //添加监听描述符事件
	    add_event(epollfd,listenfd,EPOLLIN);
	    for ( ; ; )
	    {
	        //获取已经准备好的描述符事件
	        ret = epoll_wait(epollfd,events,EPOLLEVENTS,-1);
	        handle_events(epollfd,events,ret,listenfd,buf);
	    }
	    close(epollfd);
	}
	
	static void
	handle_events(int epollfd,struct epoll_event *events,int num,int listenfd,char *buf)
	{
	    int i;
	    int fd;
	    //进行选好遍历
	    for (i = 0;i < num;i++)
	    {
	        fd = events[i].data.fd;
	        //根据描述符的类型和事件类型进行处理
	        if ((fd == listenfd) &&(events[i].events & EPOLLIN))
	            handle_accpet(epollfd,listenfd);
	        else if (events[i].events & EPOLLIN)
	            do_read(epollfd,fd,buf);
	        else if (events[i].events & EPOLLOUT)
	            do_write(epollfd,fd,buf);
	    }
	}
	static void handle_accpet(int epollfd,int listenfd)
	{
	    int clifd;
	    struct sockaddr_in cliaddr;
	    socklen_t  cliaddrlen;
	    clifd = accept(listenfd,(struct sockaddr*)&cliaddr,&cliaddrlen);
	    if (clifd == -1)
	        perror("accpet error:");
	    else
	    {
	        printf("accept a new client: %s:%d\n",inet_ntoa(cliaddr.sin_addr),cliaddr.sin_port);
	        //添加一个客户描述符和事件
	        add_event(epollfd,clifd,EPOLLIN);
	    }
	}
	
	static void do_read(int epollfd,int fd,char *buf)
	{
	    int nread;
	    nread = read(fd,buf,MAXSIZE);
	    if (nread == -1)
	    {
	        perror("read error:");
	        close(fd);
	        delete_event(epollfd,fd,EPOLLIN);
	    }
	    else if (nread == 0)
	    {
	        fprintf(stderr,"client close.\n");
	        close(fd);
	        delete_event(epollfd,fd,EPOLLIN);
	    }
	    else
	    {
	        printf("read message is : %s",buf);
	        //修改描述符对应的事件，由读改为写
	        modify_event(epollfd,fd,EPOLLOUT);
	    }
	}
	
	static void do_write(int epollfd,int fd,char *buf)
	{
	    int nwrite;
	    nwrite = write(fd,buf,strlen(buf));
	    if (nwrite == -1)
	    {
	        perror("write error:");
	        close(fd);
	        delete_event(epollfd,fd,EPOLLOUT);
	    }
	    else
	        modify_event(epollfd,fd,EPOLLIN);
	    memset(buf,0,MAXSIZE);
	}
	
	static void add_event(int epollfd,int fd,int state)
	{
	    struct epoll_event ev;
	    ev.events = state;
	    ev.data.fd = fd;
	    epoll_ctl(epollfd,EPOLL_CTL_ADD,fd,&ev);
	}
	
	static void delete_event(int epollfd,int fd,int state)
	{
	    struct epoll_event ev;
	    ev.events = state;
	    ev.data.fd = fd;
	    epoll_ctl(epollfd,EPOLL_CTL_DEL,fd,&ev);
	}
	
	static void modify_event(int epollfd,int fd,int state)
	{
	    struct epoll_event ev;
	    ev.events = state;
	    ev.data.fd = fd;
	    epoll_ctl(epollfd,EPOLL_CTL_MOD,fd,&ev);
	}

客户端代码如下：

	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <stdio.h>
	#include <string.h>
	#include <stdlib.h>
	#include <sys/epoll.h>
	#include <time.h>
	#include <unistd.h>
	#include <sys/types.h>
	#include <arpa/inet.h>
	
	#define MAXSIZE     1024
	#define IPADDRESS   "127.0.0.1"
	#define SERV_PORT   8787
	#define FDSIZE        1024
	#define EPOLLEVENTS 20
	
	static void handle_connection(int sockfd);
	static void
	handle_events(int epollfd,struct epoll_event *events,int num,int sockfd,char *buf);
	static void do_read(int epollfd,int fd,int sockfd,char *buf);
	static void do_read(int epollfd,int fd,int sockfd,char *buf);
	static void do_write(int epollfd,int fd,int sockfd,char *buf);
	static void add_event(int epollfd,int fd,int state);
	static void delete_event(int epollfd,int fd,int state);
	static void modify_event(int epollfd,int fd,int state);
	
	int main(int argc,char *argv[])
	{
	    int                 sockfd;
	    struct sockaddr_in  servaddr;
	    sockfd = socket(AF_INET,SOCK_STREAM,0);
	    bzero(&servaddr,sizeof(servaddr));
	    servaddr.sin_family = AF_INET;
	    servaddr.sin_port = htons(SERV_PORT);
	    inet_pton(AF_INET,IPADDRESS,&servaddr.sin_addr);
	    connect(sockfd,(struct sockaddr*)&servaddr,sizeof(servaddr));
	    //处理连接
	    handle_connection(sockfd);
	    close(sockfd);
	    return 0;
	}
	
	
	static void handle_connection(int sockfd)
	{
	    int epollfd;
	    struct epoll_event events[EPOLLEVENTS];
	    char buf[MAXSIZE];
	    int ret;
	    epollfd = epoll_create(FDSIZE);
	    add_event(epollfd,STDIN_FILENO,EPOLLIN);
	    for ( ; ; )
	    {
	        ret = epoll_wait(epollfd,events,EPOLLEVENTS,-1);
	        handle_events(epollfd,events,ret,sockfd,buf);
	    }
	    close(epollfd);
	}
	
	static void
	handle_events(int epollfd,struct epoll_event *events,int num,int sockfd,char *buf)
	{
	    int fd;
	    int i;
	    for (i = 0;i < num;i++)
	    {
	        fd = events[i].data.fd;
	        if (events[i].events & EPOLLIN)
	            do_read(epollfd,fd,sockfd,buf);
	        else if (events[i].events & EPOLLOUT)
	            do_write(epollfd,fd,sockfd,buf);
	    }
	}
	
	static void do_read(int epollfd,int fd,int sockfd,char *buf)
	{
	    int nread;
	    nread = read(fd,buf,MAXSIZE);
	        if (nread == -1)
	    {
	        perror("read error:");
	        close(fd);
	    }
	    else if (nread == 0)
	    {
	        fprintf(stderr,"server close.\n");
	        close(fd);
	    }
	    else
	    {
	        if (fd == STDIN_FILENO)
	            add_event(epollfd,sockfd,EPOLLOUT);
	        else
	        {
	            delete_event(epollfd,sockfd,EPOLLIN);
	            add_event(epollfd,STDOUT_FILENO,EPOLLOUT);
	        }
	    }
	}
	
	static void do_write(int epollfd,int fd,int sockfd,char *buf)
	{
	    int nwrite;
	    nwrite = write(fd,buf,strlen(buf));
	    if (nwrite == -1)
	    {
	        perror("write error:");
	        close(fd);
	    }
	    else
	    {
	        if (fd == STDOUT_FILENO)
	            delete_event(epollfd,fd,EPOLLOUT);
	        else
	            modify_event(epollfd,fd,EPOLLIN);
	    }
	    memset(buf,0,MAXSIZE);
	}
	
	static void add_event(int epollfd,int fd,int state)
	{
	    struct epoll_event ev;
	    ev.events = state;
	    ev.data.fd = fd;
	    epoll_ctl(epollfd,EPOLL_CTL_ADD,fd,&ev);
	}
	
	static void delete_event(int epollfd,int fd,int state)
	{
	    struct epoll_event ev;
	    ev.events = state;
	    ev.data.fd = fd;
	    epoll_ctl(epollfd,EPOLL_CTL_DEL,fd,&ev);
	}
	
	static void modify_event(int epollfd,int fd,int state)
	{
	    struct epoll_event ev;
	    ev.events = state;
	    ev.data.fd = fd;
	    epoll_ctl(epollfd,EPOLL_CTL_MOD,fd,&ev);
	}

对于第一个缺点，epoll的解决方案在epoll_ctl函数中。每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程中只会拷贝一次。

对于第二个缺点，epoll的解决方案不像select或poll一样每次都把current轮流加入fd对应的设备等待队列中，而只在epoll_ctl时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表）。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用schedule_timeout()实现睡一会，判断一会的效果，和select实现中的第7步是类似的）。

对于第三个缺点，epoll没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。


### 总结：

（1）select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用epoll_wait不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候只要判断一下就绪链表是否为空就行了，这节省了大量的CPU时间。这就是回调机制带来的性能提升。

（2）select，poll每次调用都要把fd集合从用户态往内核态拷贝一次，并且要把current往设备等待队列中挂一次，而epoll只要一次拷贝，而且把current往等待队列上挂也只挂一次（在epoll_wait的开始，注意这里的等待队列并不是设备等待队列，只是一个epoll内部定义的等待队列）。这也能节省不少的开销。

### 参考
[select、poll、epoll之间的区别总结](http://www.cnblogs.com/Anker/p/3265058.html)

[select,poll,epoll区别](http://blog.csdn.net/ithomer/article/details/5971779)

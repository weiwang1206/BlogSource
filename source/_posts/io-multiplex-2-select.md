#IO多路复用---select

**select**是最简单的最基础的IO多路复用技术，用句时髦的话叫 “de facto method"，已经成为事实的一种方法。闲话少说，我们直接来看看这个函数。

	//linux system
	#include<sys/select.h>
	int select(int maxfdp,fd_set *readfds,fd_set *writefds,fd_set *errorfds,struct timeval *timeout)
	
	struct timeval{
		long tv_sec; /second
		long tv_usec;/micro seconds
	}

	
以及配套使用的宏

	#include<sys/select.h>
	int FD_ISSET(int fd, fd_set * fdset)
	
	int FD_CLR(int fd, fd_set * fdset)
	int FD_SET(int fd, fd_set * fdset)
	int FD_ZREO(fd_set * fdset)
	
		
	
关于select的用法，直接上最简单版本的代码，代码是最好的解释~~~

	main()  
	{  
    	int sock;  
    	int *file;  
    	struct fd_set fds;
    	struct timeval timeout={3,0}; //select等待3秒，3秒轮询，要非阻塞就置0  
    	char buffer[256]={0}; //256字节的接收缓冲区  
    	/* 假定已经建立TCP listen socket 
    	sock=socket(...); 
    	bind(...); 
    	file=open(...); */  
    	while(1)  
    	{  
        	FD_ZERO(&fds); //每次循环都要清空集合，否则不能检测描述符变化  
        	FD_SET(sock,&fds); //添加描述符  
        	FD_SET(file,&fds); //同上  
        	maxfdp=sock>fp?sock+1:fp+1;//描述符最大值加1，其实如果描述符集合不变，这句可以放在循环外
        	select(maxfdp,&fds,NULL,NULL,&timeout))   //select使用  
        	
        	if (ret < 0)
        	{  
            	perror("select");  
            	break;  
        	} else if (ret == 0) {  
            	printf("timeout\n");  
            	continue;  
        	}
        	
        	if (FD_ISSET(sock, &fds)) {监听socket有数据可读
        		int s = accept(...) //建立tcp链接
        		//...
        	}
        	if (FD_ISSET(file, &fds)) {//文件有可读数据
        		int num = read(file,...)
        	}
        	
     	}//end while  
	}//end main

fds是需要内核监听的文件描述符集合，类型是fd\_set结构体。select函数的作用是监听多个描述符的事件，因此我们这里假设有两个描述符，一个是server端的socket套接字sock，和文件描述符fd。 然后进入while死循环。在while循环中，每次都需要情况fds，因为fds是用来标记哪个描述有未处理的事件，因此每次调用select之前要先情况，并且重新添加需要监测的描述符，使用FD_SET函数，将sock和file加入fds（**NOTE：这也是select函数效率较低的原因之一**）。然后计算出描述符中最大的值+1，作为select的参数之一，我们这里只有两个描述符，因此简单比较一个即可，如果有多个描述符，还需要计算出。然后调用select函数，第一个参数是maxfdp，第二个参数是我们关心可读事件的描述符集合，第三个是关心的可写的描述符集合，第四个是关心异常事件的描述符集合，我们这里只关注可读事件，因此将第二个参数传入fds，最后一个参数是time_out，在APUE中，time_out有三种值
- NULL, 永远等待。直到描述符有事件发生或收到一个错误信号。
- time_out->tv_sec = 0, time_out->tv_usec = 0, 不等待，立即返回，不阻塞。
- time_out->tv_sec != 0, time_out->tv_usec = !0, 等待制定的时间或有事件发生就返回。

当select函数返回后，如果返回值小于0， 就出错了；如果等于0，意味着等待超时并在这段事件没有事件发生。

如果ret大于0，就是我们希望出现的结果了，YES。但是我们监听了很多描述符，到底是哪个描述符有事件呢，我们并不知道，select函数也没告诉我们。select只将fds完全返回，并将其中有事件的置位，没有事件的置0，因此我们需要逐一判断到底哪个描述符有事件了（**这就是另一个select效率低下的原因**）。在本代码中，我们有两个描述符，因此使用FD_ISSET函数判断了两次，如果有很多的描述符参与，则需要使用循环监测每一个。如果FD_ISSET返回非0，则该描述符有事件发生，我们可以执行响应的操作。然后循环重新开始，清除fds，重新添加描述符。

这就是select函数的用法，很简单吧！配合多进程，我们可以写一个**并发量不太大**的webServer，代码见[selectServer.c](http://)


导致**select并发量不太大的原因**主要是因为：

1. 描述符数量限制，硬编码在内核中，最大1024个
2. 每次调用都需要将整个fdset从用户空间拷贝到内核空间，返回时再从内核空间拷贝的用户空间，这是非常耗时的
3. 当select进入内核态后，也需要遍历fdset，来寻找哪个描述符的可读或可写被置位了，这种遍历对于大规模的fdset，效率非常低。
	关于select的实现可以参考，[select实现分析](http://blog.csdn.net/lizhiguo0532/article/details/6568964#comments)
4. select返回后，需要对整个fdset进行遍历才知道哪些fd被置位了（可读或可写），效率较低。


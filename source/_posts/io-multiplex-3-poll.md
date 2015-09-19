#IO多路复用---poll

poll函数和select函数非常类似，内核实现也是一样的。区别就是函数调用接口不同，并且poll没有了select对于描述符数量（最大1024）大小的限制。

	#include <poll.h>
	int poll (struct pollfd *__fds, nfds_t __nfds, int __timeout);
	
	struct pollfd{
		int fd;              //文件描述符
		short events;    //请求的事件
		short revents;   //返回的事件
	};
	
	//events, 后面带*号的只能作为描述字的返回结果存储在revents中，而不能作为测试条件用于events中	POLLIN		普通或优先级带数据可读
	POLLRDNORM	普通数据可读
	POLLRDBAND	优先级带数据可读
	POLLPRI		高优先级数据可读
	POLLOUT		普通数据可写
	POLLWRNORM	普通数据可写
	POLLWRBAND	优先级带数据可写
	POLLERR		发生错误 *
	POLLHUP		发生挂起 *
	POLLNVAL	描述字不是一个打开的文件 *
	
\_\_nfds表示 __fds的数量，timeout为超时，-1表示永远等待，0表示不等待，>0表示等待的毫秒数。使用方法如下：

	int main (int argc, char **argv)
	{                               
        int sfd1, sfd2, sfd3;   

        struct pollfd Events [3];
        int retval;              
        char buff [256];         
        int readcnt;             

        sfd1 = open (...);
        sfd2 = open (...);
        sfd3 = open (...);

                  

        memset (Event, 0, sizeof(Events));

        Event[0].fd = sfd1;
        Event[0].events = POLLIN | POLLERR;     /*关心读取和出错事件*/

        Event[1].fd = sfd2;
        Event[1].events = POLLIN | POLLERR;     /*关心读取和出错事件*/

        Event[2].fd = sfd3;
        Event[2].events = POLLIN | POLLERR;     /*关心读取和出错事件*/

        while (1) {
                /*等待事件*/
                retval = poll ((struct pollfd *)&Events, 3, 5000);
                if (retval < 0) {                                 
                        perror ("poll");
                        exit (EXIT_FAILURE);
                }
                if (retval == 0) {
                        prinntf ("no data in 5 seconds.\n");
                        continue;
                }

                for (i = 0; i < 3; i++) {
                        /*检查错误*/
                        if (Events[i].revents & POLLERR) {
                                printf ("device error!\n");
                                exit (EXIT_FAILURE);
                        }

                        /*检查是否存在传递的数据(可读)*/
                        if (Events[i].revents & POLLIN) {
                                readcnt = read (Events[i].fd, buff, 256);
                                write (Events[i].fd, buff, readcnt);
                        }
                }
        }

        close (sfd1);
        close (sfd2);
        colse (sfd3);
	}
	
可以看到poll和select的程序模型基本是一样的。唯一区别就是传入关心的描述符集和传出事件的方式不一样。而且poll不需要在循环中每次都重新设置events，但是只是不需要在用户程序中重复设置，仍然需要每次都传入。

####poll的缺陷

- 和select相同，用户态和内核态之间传输描述符集合及其事件的开销太大。
- 返回的事件并不是只是有事件的描述符，而是所有描述符集合，需要进行遍历查找。


**至此，经典的IO复用函数select和poll就简单介绍完了。在2000年左右，随着互联网的蓬勃发展，Client数目爆发式增长，服务器需要并发的数量已经不是几百几千这个数量级了，因此提出了[C10K Problem](http://http://www.kegel.com/c10k.html)，指出select和poll已经不能胜任当前的服务器并发了，随后也提出了新的性能更好的IO复用函数，即Linux平台上的epoll，和FreeBSD平台的kqueue。**
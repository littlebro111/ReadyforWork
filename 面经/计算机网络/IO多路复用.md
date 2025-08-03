# 什么是I/O多路复用

多路是指多个网络连接，复用是指同一个线程。I/O多路复用就是单个线程/进程可以监听多个文件描述符。一旦某个描述符就绪，就可以通知应用程序进行相应的读写操作；没有文件描述符就绪时会阻塞应用程序，交出cpu。通过减少运行的线程/进程，就可以有效的减少上下文切换的消耗。其实，select、poll、epoll本质上都是同步I/O模型，它们都需要在读写事件就绪之后自己负责读写，即这个数据读写过程是阻塞的。

# 为什么需要I/O多路复用

根本原因是为了实现高性能服务器。

如果不使用I/O多路复用，主要可以采取的方法有以下几种：

* 服务端采用单线程和阻塞I/O，当accept一个请求后，在recv或send调用阻塞时，将无法accept其他请求（必须等上一个请求处理完recv或send），没法处理并发；

* 服务器端采用多线程和阻塞I/O，当accept一个请求后，开启线程进行recv，可以完成并发处理，但随着请求数增加需要增加系统线程，大量的线程占用很大的内存空间，并且线程切换会带来很大的开销，而且1万个线程中真正发生读写事件的线程数不会超过20%，每次accept都开一个线程也是一种资源浪费；
* 服务器端采用单线程和非阻塞I/O，当accept一个请求后，就将其描述符加入一个fds集合中，每次轮询一遍fds集合recv(非阻塞)数据，没有数据则立即返回错误，但每次轮询所有fd（包括没有发生读写事件的fd）会很浪费cpu。

因此最终就采取I/O多路复用技术来解决单线程监听多个网络连接的情况。

# IO多路复用的三种实现方式

## select

### select函数接口

```c++
#include <sys/select.h>
#include <sys/time.h>
 
#define FD_SETSIZE 1024
#define NFDBITS (8 * sizeof(unsigned long))
#define __FDSET_LONGS (FD_SETSIZE/NFDBITS)
 
// 数据结构 (bitmap)
typedef struct {
    unsigned long fds_bits[__FDSET_LONGS];
} fd_set;

// 计时器
struct timeval {
    long tv_sec; // seconds
    long tv_usec; // microseconds
};

// API
int select(
    int max_fd, 
    fd_set *readset, 
    fd_set *writeset, 
    fd_set *exceptset, 
    struct timeval *timeout
);                              // 返回值就绪描述符的数目
 
FD_ZERO(int fd, fd_set* fds);   // 清空集合
FD_SET(int fd, fd_set* fds);    // 将给定的描述符加入集合
FD_ISSET(int fd, fd_set* fds);  // 判断指定描述符是否在集合中 
FD_CLR(int fd, fd_set* fds);    // 将给定的描述符从文件中删除 
```

### select使用示例

```c++
void useSelect(int listenfd, int readfd, int writefd) {
	/*
	* 这里进行一些初始化的设置，
	* 包括socket建立，地址的设置等,
	*/

	fd_set rset, wset;
	struct timeval timeout;
	int maxfdpl = 0;  // 用于记录最大的fd，在轮询中时刻更新即可
    int nfds = 0; // 记录就绪的事件，可以减少遍历的次数
    
	// 初始化比特位
	FD_ZERO(&rset);
	FD_ZERO(&wset);
    while (1) {
		FD_SET(readfd, &rset);
        FD_SET(writefd, &wret);
        maxfdpl = max(readfd, writefd) + 1;
        // 阻塞获取
		// 每次需要把fd从用户态拷贝到内核态
		// nfds = select(maxfdpl, &rset, &wret, NULL, &timeout);
        nfds = select(maxfdpl, &rset, &wret, NULL, NULL);
        // 每次需要遍历所有fd，判断有无读写事件发生
        for (int i = 0; i <= maxfdpl && nfds; ++i) {
            if (i == listenfd) {
				--nfds;
				// 这里处理accept事件
				FD_SET(i, &rset); // 将客户端socket加入到集合中
        	}
			if (FD_ISSET(i, &rset)) {
				--nfds;
				// 这里处理read事件
			}
			if (FD_ISSET(i, &wret)) {
				--nfds;
				// 这里处理write事件
			}
		}
	}
}
```

## poll

### poll函数接口

```c++
#include <poll.h>

// 数据结构
struct pollfd {
    int fd;                         // 需要监视的文件描述符
    short events;                   // 需要内核监视的事件
    short revents;                  // 实际发生的事件
};

// API
int poll(struct pollfd fds[], nfds_t nfds, int timeout); // 返回值就绪描述符的数目
```

### poll使用示例

```c++
// 先宏定义长度
#define MAX_POLLFD_LEN 4096

void usePoll(int listenfd, int readfd, int writefd) {
	/*
	* 在这里进行一些初始化的操作，
	* 比如初始化数据和socket等。
	*/

	int nfds = 0;
	pollfd fds[MAX_POLLFD_LEN];
	memset(fds, 0, sizeof(fds));
	fds[0].fd = listenfd;
	fds[0].events = POLLRDNORM;
 	int maxfdpl = 0;  // 队列的实际长度，是一个随时更新的，也可以自定义其他的
 	int timeout = 0;
	while (1) {
		// 阻塞获取
		// 每次需要把fd从用户态拷贝到内核态
		nfds = poll(fds, maxfdpl + 1, timeout);
		if (fds[0].revents & POLLRDNORM) {
			// 这里处理accept事件
			connfd = accept(listenfd);
			// 将新的描述符添加到读描述符集合中
            if (--nfds <= 0) break;
		}
		// 每次需要遍历所有fd，判断有无读写事件发生
		for (int i = 1; i <= maxfdpl && nfds; ++i) {     
			if (fds[i].revents & POLLRDNORM) { 
				sockfd = fds[i].fd;
				if ((n = read(sockfd, buf, MAXLINE)) <= 0) {
					// 这里处理read事件
					if (n == 0) {
						close(sockfd);
						fds[i].fd = -1;
					}
				} else {
					// 这里处理write事件     
				}
				--nfds;
			}
		}
	}
}
```

## epoll

### epoll函数接口

#### epoll_create

epoll_create函数原型如下：

```c++
int epoll_create(int size);
```

函数说明：

- 参数size：从Linux 2.6.8以后就不再使用，但是必须设置一个大于0的值。
- 返回值：调用成功返回一个非负值的文件描述符fd（本文称之为epollfd），调用失败返回-1，这个epollfd主要是供下面两个函数使用。

#### epoll_ctl

可以使用epoll_ctl函数将我们需要检测事件的文件描述符fd绑定到epoll_create创建的epollfd上，或者是修改一个已经绑定上去的fd的事件类型，再或者是在不需要时将fd从epollfd上解绑。

epoll_ctl函数原型如下：

```c++
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

函数说明：

* 参数epfd：调用epoll_create函数创建的epollfd。
* 参数op：操作类型，取值有EPOLL_CTL_ADD、EPOLL_CTL_MOD和EPOLL_CTL_DEL，分别表示向epollfd上添加、修改和移除一个其他fd，当取值是EPOLL_CTL_DEL，第四个参数event忽略不计，可以设置为NULL。
* 参数fd：即需要被操作的fd。
* 参数event：是一个epoll_event结构体的地址，epoll_event结构体定义下面会详细介绍。
* 返回值：调用成功返回0，调用失败返回-1，你可以通过errno错误码获取具体的错误原因。

epoll_event结构体定义如下：

```c++
struct epoll_event {
    uint32_t     events; // 需要检测的fd事件，取值与poll函数一样
    epoll_data_t data; // 用户自定义数据
};

typedef union epoll_data {
    void    *ptr;
    int      fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
```

epoll_event支持的事件宏如下表：

|    事件宏    |                  描述                   |
| :----------: | :-------------------------------------: |
|   EPOLLIN    |    数据可读（包括普通数据&优先数据）    |
|   EPOLLOUT   |    数据可写（包括普通数据&优先数据）    |
|  EPOLLRDHUP  | TCP连接被对端关闭，或者对端关闭了写操作 |
|   EPOLLPRI   |    高优先级数据可读，例如TCP带外数据    |
|   EPOLLERR   |                  错误                   |
|   EPOLLHUP   |                  挂起                   |
|   EPOLLET    |              边缘触发模式               |
| EPOLLONESHOT |       最多触发其上注册的事件一次        |

#### epoll_wait

设置好某个fd上需要检测事件并将该fd绑定到epollfd上去后，就可以调用epoll_wait来检测事件。

epoll_wait函数原型如下：

```c++
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

函数说明：

* 参数epfd：调用epoll_create函数创建的epollfd。
* 参数events：是一个epoll_event结构数组的首地址，这是一个输出参数，函数调用成功后，events中存放的是与就绪事件相关epoll_event结构体数组。
* 参数maxevents：数组元素的个数。
* 参数timeout：超时时间，单位是毫秒，如果设置为0，epoll_wait会立即返回。
* 返回值：调用成功会返回有事件的fd数目；如果返回0表示超时；调用失败返回-1。

### epoll实现原理

当一个进程调用epoll_create函数时，Linux内核会创建一个eventpoll结构体，每一个epoll对象都有一个独立的eventpoll结构体，这个结构体会在内核空间中创造独立的内存，用于存储使用epoll_ctl方法向epoll对象中添加进来的事件。

eventpoll结构体的定义如下：

```c++
struct eventpoll {
	/* Protect the access to this structure */
	spinlock_t lock;
    
	/*
	* This mutex is used to ensure that files are not removed
	* while epoll is using them. This is held during the event
	* collection loop, the file cleanup path, the epoll file exit
	* code and the ctl operations.
	*/
	struct mutex mtx;
 
	/* Wait queue used by sys_epoll_wait() */
	wait_queue_head_t wq;
 
	/* Wait queue used by file->poll() */
	wait_queue_head_t poll_wait;
 
	 /* List of ready file descriptors */
	struct list_head rdllist; // 双向链表中保存着将要通过epoll_wait返回给用户的、满足条件的事件
 
	/* RB tree root used to store monitored fd structs */
	struct rb_root rbr; // 红黑树根节点，这棵树存储着所有添加到epoll中的事件，也就是这个epoll监控的事件
 
	/*
	* This is a single linked list that chains all the "struct epitem" that
	* happened while transferring ready events to userspace w/out
	* holding ->lock.
	*/
	struct epitem *ovflist;

	/* wakeup_source used when ep_scan_ready_list is running */
	struct wakeup_source *ws;

	/* The user that created the eventpoll descriptor */
	struct user_struct *user;
 
	struct file *file;
 
	/* used to optimize loop detection check */
	int visited;
	struct list_head visited_list_link;
 };
```

在eventpoll结构体中，有两个数据结构比较值得关注：红黑树和双向链表。

epoll使用红黑树作为索引结构（对应rbr），从而可以方便地添加和移除fd，而且便于搜索，能够高效的识别出重复的事件。

epoll使用双向链表来实现就绪链表（对应rdllist），用于存储就绪的事件。当调用epoll_wait函数时，仅需要观察这个双向链表里有没有数据即可，有数据就返回，没有数据就保持睡眠，等到timeout时间到达即使链表中没有数据也会返回。

在epoll中，对于每一个事件都会建立一个epitem结构体，红黑树中的每个节点都是基于epitem结构体中的rdllink成员，双向链表中的每个节点都是基于epitem结构体中的rbn成员。

epitem结构体的定义如下：

```c++
struct epitem {
	/* RB tree node used to link this structure to the eventpoll RB tree */
	struct rb_node rbn;
	
	/* List header used to link this structure to the eventpoll ready list */
	struct list_head rdllink;

	/*
	* Works together "struct eventpoll"->ovflist in keeping the
	* single linked chain of items.
	*/
	struct epitem *next;

    /* The file descriptor information this item refers to */
	struct epoll_filefd ffd;

    /* Number of active wait queue attached to poll operations */
	int nwait;

    /* List containing poll wait queues */
	struct list_head pwqlist;

    /* The "container" of this item */
	struct eventpoll *ep;

	/* List header used to link this item to the "struct file" items list */
	struct list_head fllink;

	/* wakeup_source used when EPOLLWAKEUP is set */
	struct wakeup_source __rcu *ws;

	/* The structure that describe the interested events and the source fd */
	struct epoll_event event;
};
```

综上所述：在执行epoll_create函数时，会创建一个eventpoll结构体，其中包含红黑树和双向链表；执行epoll_ctl函数时，如果是增加文件描述符，则检查其在红黑树中是否存在，存在就立即返回，不存在就将其添加到树干上，然后向内核注册回调函数，用于当中断事件来临时向就绪链表中插入数据；执行epoll_wait函数时，直接返回就绪链表里的数据即可。

### epoll工作模式

epoll模型既支持水平触发（LT），也支持边缘触发（ET），默认是水平触发（LT）。

* 水平触发（Level_triggered）：当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据一次性全部读写完(如读写缓冲区太小)，那么下次调用 epoll_wait()时，它还会通知你在没读写完的文件描述符上继续读写，当然如果你一直不去读写，它会一直通知你，如果系统中有大量你不需要读写的就绪文件描述符，而它们每次都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率。

* 边缘触发（Edge_triggered）：当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你，这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符。

ET模式只支持no-block socket，因为ET模式，当有数据时，只会被触发一次，所以每次读取数据时，一定要一次性把数据读取完，所以需要设置一个whlie循环调用recv函数读取数据直到recv出错，错误码EWOULDBLOCK或者EAGAIN（此时表示该socket上的本次数据已经全部读完），但如果recv是阻塞模式，那么如果没有数据时，将会阻塞，导致程序卡死，所以这里只允许非阻塞模式。如果使用LT模式，则不用，可以根据业务一次性收取固定的字节数，或者收完为止。

**LT模式和ET模式的优缺点：**

- ET模式

    缺点：应用层业务逻辑复杂，容易遗漏事件，很难用好。

    优点：相对LT模式效率比较高。一触发立即处理事件。

- LT模式：

    优点：编程更符合用户直觉，业务层逻辑更简单。

    缺点：效率比ET低。

### epoll使用示例

```c++
#include <sys/types.h> 
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>
#include <sys/time.h>
#include <vector>
#include <errno.h>
#include <sys/epoll.h>

int main() {
    // 创建一个侦听socket
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1) {
        std::cout << "create listen socket error." << std::endl;
        return -1;
    }

    // 初始化服务器地址
    struct sockaddr_in bindaddr;
    bindaddr.sin_family = AF_INET;
    bindaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    bindaddr.sin_port = htons(3000);
    if (bind(listenfd, (struct sockaddr *)&bindaddr, sizeof(bindaddr)) == -1) {
        std::cout << "bind listen socket error." << std::endl;
        close(listenfd);
        return -1;
    }

    // 启动侦听
    if (listen(listenfd, SOMAXCONN) == -1) {
        std::cout << "listen error." << std::endl;
        close(listenfd);
        return -1;
    }
    
    std::cout << "listen fd." << listenfd << std::endl;
    
    // epoll_create
    int epollfd = epoll_create(1);
    if(-1 == epollfd) {
        std::cout << "epoll_create error." << std::endl;
        close(listenfd);
        return -1;
    }

    epoll_event listen_fd_event;
    listen_fd_event.data.fd = listenfd;
    listen_fd_event.events = EPOLLIN;
    
    if(-1 == epoll_ctl(epollfd, EPOLL_CTL_ADD, listenfd, &listen_fd_event)) {
        std::cout << "epoll_ctl error." << std::endl;
        close(listenfd);
        return -1;
    }
    
    while (true) {
        epoll_event epoll_events[1024];

        int n = epoll_wait(epollfd, epoll_events, 1024, 1000);
        
        if (n < 0) {
            // 被信号中断
            if (errno == EINTR) 
                continue;
            // 出错,退出
            break;
        } else if (n == 0){
            // 超时,继续
            continue;
        }
        
        for(int i = 0; i < n; i++) {
            std::cout << "events." << epoll_events[i].events << std::endl;
            std::cout << "fd." << epoll_events[i].data.fd << std::endl;
            if(epoll_events[i].events & EPOLLIN) {
                if(epoll_events[i].data.fd == listenfd) {
                    // 侦听socket的可读事件，则表明有新的连接到来
                    struct sockaddr_in clientaddr;
                    socklen_t clientaddrlen = sizeof(clientaddr);
                    // 接受客户端连接
                    int clientfd = accept(listenfd, (struct sockaddr *)&clientaddr, &clientaddrlen);
                    if (clientfd == -1) {           
                        //接受连接出错，退出程序
                        break;
                    }
                    
                    epoll_event client_fd_event;
                    client_fd_event.data.fd = clientfd;
                    client_fd_event.events = EPOLLIN;
                    
                    if(-1 == epoll_ctl(epollfd, EPOLL_CTL_ADD, clientfd, &client_fd_event)) {
                        std::cout << "epoll_ctl error." << std::endl;
                        close(clientfd);
                        return -1;
                    }
                
                    //只接受连接，不调用recv收取任何数据
                    std:: cout << "accept a client connection, fd: " << clientfd << std::endl;
                } else {

                    // 有数据可以读
                    char recvbuf[32] = {0};
                    int clientfd = epoll_events[i].data.fd;
                    std::cout << "before recv: " << n << std::endl;
                    int length = recv(clientfd, recvbuf, 32, 0);
                    std::cout << "after recv: " << n << std::endl;
                    
                    if (length <= 0 && errno != EINTR) {
                        // 收取数据出错了
                        std::cout << "recv data error, clientfd: " << clientfd << std::endl;                            
                        close(clientfd);
                        continue;
                    }
                    std::cout << "clientfd: " << clientfd << ", recv data: " << recvbuf << std::endl;   
                    
                    // send data
                    int ret = send(clientfd, recvbuf, strlen(recvbuf), 0);
                    if (ret != strlen(recvbuf)) {
                        std::cout << "send data error., clientfd: " << clientfd << std::endl;   
                        close(clientfd);
                    }
                }                   
            }
        }
    }
    //关闭侦听socket
    close(listenfd);

    return 0;
}
```

## select、poll与epoll的区别

* 对于select和poll来说，所有文件描述符都是在用户态被加入其文件描述符集合的，每次调用都需要将整个集合拷贝到内核态，返回时再把就绪的fd集合从内核态拷贝到用户态，当fd很多时开销极大；epoll通过epoll_ctl将整个文件描述符集合维护在内核态，内核通过一个事件表直接管理用户感兴趣的所有事件，在调用epoll_wait时，仅需要观察就绪链表中有没有数据即可。但是每次添加文件描述符的时候都需要执行一个系统调用。

    系统调用的开销是很大的，在有很多短期活跃连接的情况下，由于这些大量的系统调用开销，epoll可能会慢于select和poll。

* select使用线性表描述文件描述符集合，其支持的文件描述符数量太小了，32位机器默认是1024，64位机器默认2048；poll使用链表来描述，没有最大连接数的限制；epoll底层通过红黑树来描述，并且维护一个就绪链表，将事件表中已经就绪的事件添加到这里，在调用epoll_wait时，仅观察这个list中有没有数据即可。

* select和poll的最大开销来自内核判断是否有文件描述符就绪这一过程：每次执行select或poll调用时，它们会采用轮询的方式，遍历整个文件描述符集合去判断各个文件描述符是否有活动；epoll则不需要去以这种方式检查，当有活动产生时，会自动触发epoll回调函数通知epoll文件描述符，然后内核将这些就绪的文件描述符放到就绪链表中等待epoll_wait被调用后处理。

* select和poll都只能工作在相对低效的LT模式下，而epoll同时支持LT和ET模式。

综上，当监测的fd数量较小，且各个fd都很活跃的情况下，建议使用select和poll；当监听的fd数量较多，且单位时间仅部分fd活跃的情况下，使用epoll会明显提升性能。

# 参考资料

[1] [近四十场面试汇聚成的超全Web服务器面经总结](https://blog.csdn.net/ClaireSy/article/details/122436929)

[2] [socket通信之epoll模型](https://blog.csdn.net/u022812849/article/details/109771916)

[3] [Epoll原理解析](https://blog.csdn.net/armlinuxww/article/details/92803381?ops_request_misc=%7B%22request%5Fid%22%3A%22165812573416780366539068%22%2C%22scm%22%3A%2220140713.130102334..%22%7D&request_id=165812573416780366539068&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-92803381-null-null.142^v32^control,185^v2^tag_show&utm_term=epoll&spm=1018.2226.3001.4187)

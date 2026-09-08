---
title: I/O 复用
date: 2023/9/15 10:59:00
categories:
- UNIX
tags:
- C/C++
- UNIX
- NetWork
---

### 1. 概述
- **I/O** 复用可以使一个进程监听多个 I/O 事件 (包括基本的读写事件) 的发生，而不是每个事件都对应一个阻塞线程，以提升程序的 I/O 效率。可以设置：当可读事件发生时，函数返回；

	- 可读事件包含以下几种：

		- 内核缓冲区字节数大小大于或等于其低水位标记 `SO_RCVLOWAT` （前文提到默认为 **1 byte**）

		- **socket** 连接被关闭

		- 监听 **socket** 上有新的连接请求

	- 可写事件包含以下几种：

		- 发送缓冲区字节大于或等于 `SO_SNDLOWAT` 
		- **socket** 上的写操作被关闭
		- **socket** 使用非阻塞的 `connect` 连接成功或超时、
		- **socket** 上有未处理的错误。可以使用 `getsockopt` 读取和清除该错误


###  2. **Linux** 库中的 I/O 复用
#### 2.1 select

- 前三项表示对 **读**、**写** 和 **异常** 事件的监听，**fd** 按照 `fd_set` 结构传入，监听的事件被触发时，函数返回并将传入的 **fd** 修改后从内核复制到用户区。函数成功调用返回 **就绪的文件描述符数量** ，若超时则返回 **0** ，如果在等待期间接收到信号则返回 **-1** 并设置 **errno** 为 `EINTR` 

	```c++ 
	    #include <sys/select>
	    int select(int nfds, fd_set* rdfds, fd_set* wrfds, fd_set* excpfds, struct timeval* timeout);
	```

-  `fd_set` 的结构在此不进行深究，其内部使用 **bitmap** 。可以用以下几组函数设置

	```c++
		#include <sys/select.h>
		void FD_ZERO(fd_set* fdset);		// 清除所有fd位
		void FD_CLR(int fd, fd_set* fdset);	// 清除指定fd位
		void FD_SET(int fd, fd_set* fdset); 
		int FD_ISSET(int fd, fd_set* fdset); // 判断指定fd位是否设置
	```


- 使用 `select` 的时候可以把监听 **fd** 放入 **fd** 集合中一起处理，之后借助上述几个函数判断是什么事件使 `select` 返回，再逐一处理。注意内核会修改传入的 **fd** 集合，每次使用 `select` 的时候都应该清除触发的 **fd** 位，不然会导致同一个事件重复触发。

#### 2.2 poll

- 与上述 select 函数略有不同，这次 **fd** 的触发状态不再通过 `fd_set` 结构进行传递，而是通过 `pollfd` 类型的数组进行传递。而 `n_fd_t` 其实是 `unsigned long` 的封装，代表 **fd** 的数目。同时注意 `timeout` 参数类型不同。不过返回值逻辑与 `select` 相同 

	```c++
		#include <poll.h>
		int poll(struct pollfd* fds, n_fds_t nfds, int timeout);
	
		struct pollfd {
	        int fd;
	        short events;
	        short revents; // 这里会被内核修改
	    }
	```


#### 2.3 epoll

- `epoll` 的实现和用法与上述两个函数十分不同，它无需全量传入监听事件给内核，而是将事件放在一个额外的内核事件表中，所以需要多一个文件描述符表示内核事件表

	```c++
		#include <sys/epoll.h>
		int epoll_create(int size); // size 只是给内核的一个提示
	```
	
- 之后就可以向内核事件表中注册事件
    ```c++
    #include <sys/epoll.h>
    int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);
    ```
  - 其中 `op` 指定操作类型，代表对于内核事件表的操作
  
      - `EPOLL_CTL_ADD`
      - `EPOLL_CTL_MOD`
      - `EPOLL_CTL_DEL`
      
  - `event` 用于指定事件
  
      ```C++
      	struct epoll_event {
              __uint32_t events; // 事件类型, EPOLLIN, EPOLLET, EPOLLONESHOT 等
              epoll_data_t data; // 存储返回数据的联合体, 一般内部的 fd 用的最多
          };
      
      	typedef struct union epoll_data {
              void* ptr;
              int fd;
              uint32_t u32;
              uint64_t u64;
          } epoll_data_t;
      ```
  
      

- 设置完事件之后就可以像前两种 **I/O** 复用函数一样调用，返回值逻辑同于上两种 **I/O** 复用函数。注意这里的 events 需要提供一个缓冲区供内核存放事件。

	```c++
		int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
	```

- **LT** 与 **ET** 的区别在于后者只会通知一次。另外注册了 **EPOLLONESHOT** 的文件描述符，只会被触发一次 (不管是什么事件)，事件处理完后要重新注册以便于收到下一次通知。

#### 2.4 三个 I/O 复用函数的比较

- **select** 和 **poll** 都需要不断地将监听事件传入内核再等待其写回用户空间， **epoll** 只需要注册一次，但是需要额外打开一张内核事件表。
- **epoll** 内核中监听事件靠的是回调函数实现，同时只向用户空间传回被触发的事件，时间复杂度 **O(1)**
- **select** 监听 **fd** 最大值的典型值为 **1024**，后二者一般为能打开的最大文件描述符数。
- 前二者只有 **LT** 模式，而 **epoll** 还可以支持 **ET** 



###  3. I/O 复用使用场景

- 对于非阻塞的 `connect`, 连接失败会产生可写事件。这时就可以用上述 **I/O** 复用函数进行监听，接着用 `getsockopt` 读取错误码并清除错误。


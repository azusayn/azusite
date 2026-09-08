---
title: Unix Socket编程基本API
date: 2023/8/12 13:28:00
categories:
- UNIX
tags:
- C/C++
- UNIX
- NetWork
---

### 1. Socket地址 API
#### 1.1 地址结构
​	地址结构定义在`<netinet/in.h>`中

- **IPv4**

	```c++
	    struct sockaddr_in {
	        uint8_t 	sin_len;
	        sa_family_t sin_family;
	        in_addr 	sin_addr;
	        char 		sin_zero[8];	
	    }
	
	    struct in_addr {
	        in_addr_t s_addr; // 32-bit IPv4 地址
	    }
	```
	
	1）注意 `in_addr ` 被定义为结构体却只有一个字段是由于历史原因：在子网划分技术出现之前，过去的 `struct in_addr` 存储着四个8位值或者两个16位值，便于划分 **A类、B类、C类**地址，而现在已经不需要。
	
	2） `sin_zero` 字段作为结构体填充，一般置为零。
	
- **IPv6**， 以及 **UNIX**  本地域地址（ 即在本机的 socket 之间进行通信，不用经过协议栈 ）

  ```c
  	struct sockaddr_in6{}
  	struct sockaddr_un{}
  ```

  

- **通用Socket地址结构**
	
	```c++
	    struct sockaddr {
	        uint8_t 	sa_len;
	        sa_family_t sa_family;
	        char		sa_data[14];
	    }
	
		struct sockaddr_storage {
	        sa_family_t sa_family;
	        unsigned long int __ss_align;     // 内存对齐
	        char __ss_padding[128 - sizeof(__ss_align)];
	    }
	```
	
	1）事实上调用套接字相关 **API** 的时候，函数定义的地址参数为 `struct sockaddr*` , 需要将上述地址结构指针强制转换再传入。
	
	2）由于 `sockaddr` 并不能放下 **IPv6** 或者 **UNIX** 本地域地址，所以有了第二种地址结构 `sockaddr_storgae` 
	



#### 1.2 字节排序函数

1. 在计算机存储中，需要区分不同的字节存储顺序，以使机器能够按照正确的逻辑顺序读取数据。字节序分为 **大端序** 和 **小端序** ，不同的主机可能拥有不同的 **主机字节序** 。例如某个十六进制值 **0x1234567** 的每个字节按照不同顺序的排列：


|    address    | 0x100 | 0x101 | 0x102 | 0x103 |
| :-----------: | :---: | :---: | :---: | :---: |
|  big endian   |  01   |  23   |  45   |  67   |
| little endian |  67   |  45   |  23   |  01   |

2. 网际协议提倡按 <u>大端序</u> 作为 **网络字节序** 。同时为了保证网络传输中的各个数据是正确的字节序，**UNIX** 提供字节顺序转换函数。

```c
    // h: host, n: net
    // l: long, s: short
    uint16_t htons(uint16_t);
    uint16_t htonl(uint16_t);
    uint16_t ntohs(uint16_t);
    uint16_t ntohl(uint16_t);
```



#### 1.3 IP 地址转换函数

- 下列三个函数，将 **IPv4** 数据在 **char*** 和网络字节序整数之间进行转换

    ```c
        #include <arpa/inet.h>
    
        // 将点十分制的地址转换为 in_addr_t, 失败则返回宏 INADDR_NONE
        in_addr_t inet_addr(const char* inp);
    
        // 与 inet_addr 作用相同, 成功返回 true, 否则 false
        int inet_aton(const char* cp, struct in_addr* inp);
    
        // 将网络字节序整数转换为 char*, 注意返回的指针指向的是内部的一个静态变量
        char* inet_ntoa(struct inaddr_t in);
	```

- 下面的新函数完成同样的功能，同时适用于 **IPv6** 地址

    ```c
        #include <arpa/inet.h>
    	
    	// 成功返回1, 若失败返回 0 且设置 errno
       	int inet_pton(int af, const char* src, void* dst);
    	
    	// 成功返回存储地址的地址, 失败则返回 NULL 并设置 errno
    	const char* inet_ntop(int af, const void* src, char* dst, socklen_t cnt);
    ```

- 另外新函数中指定目标存储单元大小的大小可以根据提供的宏进行定义

	```c
		#include <netinet/in.h>
		#define INET_ADDRSTRLEN 16
		#define INET6_ADDRSTRLEN 46
	```

### 2.   基本的网络连接

#### 2.1 初始化 **socket** 

- 返回值：成功则返回 **fd**， 否则返回 **-1** 并设置 `errno`
- `domain` : 指定 **IP** 协议或者 **UNIX** 本地域协议
- `type` : 用于指定 **运输层** 协议。自 **Linux** 内核 **2.6.17** 开始可以将这个值同 `SOCK_NONBLOCK` 或者 `SOCK_CLOEXEC`  **相与** ，以获得 `fcntl` 的功能。其中 `SOCK_CLOEXEC` 用于在子进程中关闭该 **socket** 
- `protocol` : 通常设置为 **0**，除非还需要选择更具体的协议

``` c++
	#include <sys/types.h>
	#include <sys/socket.h>
	int socket(int domain, int type, int protocol);
```

#### 2.2 命名 **socket**

为一个 **socket** 绑定地址被称为 **socket** 的命名。这个操作在客户端并不是必须的，默认情况下系统会自动为客户端的 **socket** 分配通信端口。 

- 返回值：成功返回 **0**， 否则返回 **-1** 并设置 **errno**。**errno** 有以下两种：
	- `EACCES` : 被绑定地址是受保护地址，比如除了 **root** 用户以外访问 0 - 1024 端口设置这个 **errno** 
	- `EADDRINUSE` ：端口正在被使用

- `addr` ：如前文所述，应该将专用的地址结构在这里进行类型转换
- `addrlen` : 因为不同专用地址结构的大小不同，需要传入专用地址的大小。可以通过 `sizeof` 实现。另外 `socklen_t` 其实是一个 `unsigned int`

```c++
 	int bind(int sockfd, const struct sockaddr* addr, socklen_t addrlen);
```



#### 2.3 监听端口与接受连接

- `listen` 。我们把调用过 `listen` 处于 **LISTEN** 状态的 **socket** 称之为 **监听socket** 。将 **ESTABLISHED** 状态的 socket 称之为 **连接socket**，它们会被放入监听队列中等待下文 `accept` 函数处理

	- 返回值：成功返回 **0**， 否则返回 **-1**，并设置 **errno**。 
	- `backlog` : 提示内核最大的连接数量。在内核 **2.2** 版本之前，这个连接数包括 **SYN_RCVD** 和 **ESTBLISHED** 两种状态的连接，现在只包括后者。 

	```c++
		#include <sys/socket>
		int listen(int sockfd, int backlog);
	```

- `accept` , 从监听队列取出一个 **socket** 进行处理

	- 返回值：成功则返回 **fd**， 否则返回 **-1** 并设置 `errno` 

	- `addr` :  **连接socket** 的地址，监听 **socket** 的 **IP** 地址可以设置为 `INETADDR_ANY` 代表监听某端口任何地址的连接

	```c++
		#include <sys/socket>
		int accept(int sockfd, struct sockaddr* addr, socklen_t* addrlen);
	```
	

#### 2.4 发起连接

- `connect` 

	- 返回值：成功返回 **0**，失败返回 **-1** 并设置 **errno** ，分为以下两种
		- `ECONNREFUSED`
		- `ETIMEOUT` 

	```c++
		int connect(int sockfd, const struct sockaddr* addr, socklen_t addrlen);
	```


#### 2.5 关闭连接

- 可以通过通用的文件接口关闭 **socket**

  ```c++
      int close(int fd);
  ```

- 或者 **socket** 专用的函数，`howto` 可以选择性关闭 **socket** 的读写端，有以下三种宏可以选择

	- `SHUT_RD`
	- `SHUT_WR`
	- `SHUT_RDWR`
	
	```c++
		int shutdown(int fd, int howto);
	```
	
- 注意 `close` 只会把 **fd** 上的引用减 **1** ， 只有对文件的引用为 **0** 时，才会释放资源。而后者会直接释放资源。

#### 2.6 socket 选项

对于文件可以通过 `fcntl` 设置文件描述符的属性，而对于 **socket** 也有专门的函数。

- `level` 指定选项属于的协议 （也可以是协议无关的通用选项 `SOl_SOCKET`）, 例如 **IPv4** 和 **IPv6** 分别为 `IPPROTO_IP` 和 `IPPROTO_IPV6` 
- `option_name` 即为具体的选项名称
- `option_value` 和 `option_len` 根据不同的选项有不同的设置方法

```c++
	int getsockopt(int fd, int level, int option_name, 
        	void* option_value, socklen_t* restrict option_len);
	int setsockopt(int fd, int level, int option_name, 
        	const void* option_value, socklen_t* restrict option_len);
```

接下来我们讨论几个重要的 **socket** 选项

- `SO_REUSEADDR` : 该选项使处于 **TIME_WAIT** 状态的 **socket** 的端口能被立即重用
- `SO_REVBUF` 和 `SO_SNDBUF` ： 设置 **TCP** 接收缓冲区和发送缓冲区的大小。不过最终的值可能会受到系统的调度
- `SO_RCVLOWAT` 和 `SO_SNDLOWAT` ：**I/O** 复用系统通过低水位标记 （一般为一个字节）来判断缓冲区是否可读，一般情况下低水位指的都是一个字节
- `SO_LINGER` : 用于控制 `close` 在关闭 **TCP** 连接时的行为，具体的讨论会放在之后的文章中。

### 3. 数据读写

#### 3.1 专用读写函数

对于 socket 上的数据读写同样可以用普通文件的读写函数，但是专门的 socket 读写函数带有对数据读写的控制

- 对于  **TCP** 数据流的读写

	- 其中 `flags` 有几个典型值：
		- `MSG_MORE` ： 提示内核还有更多数据要写入，于是内核会等待数据写入后再一并发送
		- `MSG_OOB` ： 专门接收带外数据，带外数据一般处于与普通数据不同的缓冲区中

	```c++
		#include <sys/types>
		#include <sys/socket.h>
		ssize_t recv(int fd, void* buf, size_t len, int flags);
	    ssize_t send(int fd, const void* buf, size_t len, int flags);
	```

- 对于 **UDP** 数据

	可以看到函数命名上都带上了介词 `from` 和 `to`，暗示 **UDP** 协议没有源地址和源端口，所以对于连接好的 **fd** ，我们必须手动指定目标 **socket** 地址结构。实际上如果将地址结构置为 **NULL** ，同样也可以读写流式数据。

	```c++
		#include <sys/types>
		#include <sys/socket.h>
		ssize_t recvfrom(int scokfd, void* buf, size_t len, int flags, 
	    	struct sockaddr* addr, socklen_t* len);
		ssize_t sendto(int scokfd, const void* buf, size_t len, int flags, 
	    	struct sockaddr* addr, socklen_t* len);
	```

#### 3.2 通用读写函数

- socket 编程接口还提供了一组适用于 **TCP** 和 **UDP** 的系统调用

	```c++
		#include <sys/socket.h>
		ssize_t recvmsg(int fd, struct msghdr* msg, int flag);
		ssize_t sendmsg(int fd, struct msghdr* msg, int flag);
	```

	其中 `struct msghdr` 是一个专门的数据结构，注意在 **TCP** 中 `msg_name` 这个字段没有任何意义，应当设置为 `NULL` 。其中`struct iovec*` 类型的指针指向一个 `iovec` 数组，而每个 `iovec` 结构都指向一片内存。它们代表着若干块分散的内存，使用上述函数进行读写分别成为 **分散读** 和 **集中写** 。

	```c++
		struct msghdr {
	        void* 		  msg_name;			// socket 地址结构
	        socklen_t 	  msg_namelen;
	        struct iovec* msg_iov;
	        int 		  msg_iovlen;
	        void* 		  msg_control; 		// 辅助数据
	        socklen_t 	  msg_controllen; 
	        int 		  msg_flags;		// 根据 flag 参数自动设定
	    }
	
		struct iovec {
	        void*  iov_base;
	        size_t iov_len;
	    }
	```

	

### 4. 获取网络信息

#### 4.1 静态

- 可以根据一个连接的 **socket** 的文件描述符获取 **socket** 结构的地址。**peer** 和 **sock** 分别代表 **对端** 和和 **本端** 的地址。

	```c++
		int getsockname(int fd, struct sockaddr* addr, socklen_t* len);
		int getpeername(int fd, struct sockaddr* addr, socklen_t* len);
	```


#### 4.2 动态

以下介绍的函数可以根据主机名字的信息获取 **IP** 和端口信息，或者反过来查询。系统会在本地 **/etc/services** 和 **/etc/hosts** 进行查询，若查不到，则会向 **DNS** 服务器发起请求

- `gethostbyname` 和 `gethostbyaddr` ，这两个函数根据主机名字或者地址获取主机的更多信息

	```c++
		#include <netdb.h>
		struct hostent* gethostbyname(const char* name);
		struct hostent* gethostbyaddr(const void* addr, size_t len, itn type);
		
		struct hostent {
	        char* 	h_name;		
	        char** 	h_aliases;  // 主机别名列表
	        int 	h_addrtype; // 地址族
	        int 	h_length; 
	        char** 	h_addr_list; // ip地址列表
	    }
	```

- `getservbyname` 和 `getservbyport` ， 这两个函数通过本地的  **/etc/services** 文件进行查询。

- 实际上存在 `getaddrinfo` 和 `getnameinfo` 这两个函数，但功能大致类似，不多讨论。



### 5. 更多思考

1. 注意到 `accept` 函数和 `bind` 函数中传入地址的方式一个是指针，一个是传值。

---
title: 服务器调制、调试、测试与系统监测
date: 2023/9/18 00:02:00
categories:
- UNIX
tags:
- shell
- UNIX
- NetWork
---

### 1. 调制

#### 1.1 文件描述符限制调整

**Linux** 的一个优秀特性就是内核微调，包括了最大文件描述符个数的设置。最大文件描述符个数，分为用户的限制和系统级限制。

- 如果是用户级别的限制

	- 可以用以下指令进行查看
    
	  ```shell
	  	ulimit -n
	  ```
	  
	- 通过 `ulimit` 设置为最大限制，但是只针对当前会话有效

        	ulmit -SHn max-file-number
    
  - 如果要永久有效，应该进入 **/etc/security/limits.conf ** 手动设置
	
		```shell
			hard nofile max-file-header # 硬限制
			soft nofile max-file-header
		```
	
		
	
- 系统级别的限制

	
	- 临时设置
        ```shell
            sysctl -w fs.file-max=max-file-header 
        ```

	- 永久设置 (修改 **/etc/sysctl.conf**)
	
	  ```shell
	  	fs.file-max=max-file-header # 之后需要使用 sysctl -p 使设置生效
	  ```
	
	  

#### 1.2 内核参数设置

内核的参数大部分都可以在 **/proc/sys** 中查看，使用 `sysctl -a` 可以看到所有的内核参数。接下来我们关注几个重要的参数

- **/proc/sys/fs** ：其下保存着文件系统有关的信息
	- **/file-max **：系统级文件描述符限制
	- **/epoll/max_user_watches** ：所有打开的 **epoll** 内核事件最大值

- **/proc/sys/net**
	- **/core/somaxconn** :设置 **Established** 状态的 **socket** 的最大值
	- **/ipv4/tcp_max_syn_backlog** : 与上者类似，但是包括了 **SYN_RCVD** 状态的 **socket** 
	- **/ipv4/tcp_wmen**：指定一个 **socket** 的 **tcp** 写缓冲区的最小值，默认值与最大值
	- **/ipv4/tcp_rmen**：指定一个 **socket** 的 **tcp** 读缓冲区的最小值，默认值与最大值
	- **/ipv4/tcp_syncookies**：代表通过对 **syn** 的缓存，防止同一地址的重复同步报文，导致 `listen` 监听队列溢出

### 2. gdb调试

#### 2.1 多进程调试的方法

1. 指定进程调试，在 **gdb** 中指定 **[PID]** 后打断点进行调试 

    ```gdb
        (gdb) attach [PID] 
    ```

2. 设置在 `fork` 调用之后是否跟随子进程（设置 **[parent]** 或者 **[child]**）

    ```
    	(gdb) set follow-fork-mode [parent/child] 
    ```

#### 2.2 多线程调试

- 有以下指令可以帮助调试线程

	```shel
		(gdb) info threads
		(gdb) thread [ID]  // 调试 [ID] 线程
	    (gdb) set scheduler-locking [off/on/step] 
	```

	注意第三个指令 `off` 为默认值，`on` 表示只运行当前线程，`step` 表示单步调试的时候只有当前线程能执行



### 3. 压力测试

[如何进行压力测试？](https://www.loadview-testing.com/zh-hans/blog/%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E4%B8%8E%E5%8E%8B%E5%8A%9B%E6%B5%8B%E8%AF%95%E4%B8%8E%E8%B4%9F%E8%BD%BD%E6%B5%8B%E8%AF%95/)

- 压力测试程序有多种实现方法，其中等待服务器响应时用 **I/O** 可以让测试服务器施加尽可能多的压力，而不是浪费过多 `CPU` 资源在处理响应连接上
- 另外可以通过一些常见的测试程序进行测试
	- Apache JMeter
	- LoadRunner
	- BlazeMeter
	- Gatling

### 4. 系统监测工具

- **tcpdump** 

	- 可以指定类型，方向或协议进行过滤

		- 类型（包括 `net`，`host`，`port` 等）

			```shell
				tcpdump net 192.168.31.1/24
			```

		- 方向 （ `src` 和 `dest`）

			```shel
				tcpdump dest port 8080
			```

		- 协议（ 比如 `icmp` ）

			```she
				tcpdump icmp
			```

			

	- 表达式之间可以进行逻辑运算（`and`，`or`，`not`），复杂的表达式用单引号包裹，内部可以使用括号分组

		```shel
			tcpdump 'src 192.168.31.217 and (dest port 8080 or 22)'
		```

		

	- 同时支持对包内部的信息进行过滤，比如过滤得到 **syn** 报文。因为报文的第 **14** 个字节的第 2 位正是 **syn** 标志

		```shell
			# 格式 -> tcpdump 'tcp[BYTES] & [NUMBER] != 0' 
			tcpdump 'tcp[13] & 2 != 0'
		```
		
	- 同时支持许多选项

	  ```shell
	      -n # 使用IP地址表示主机名
	      -i # 指定要监听的网口, 可以使用 -i any
	      -v # 显示更详细的信息, 比如显示 tcp 中的 TTL 和 TOS 信息
	      -t # 不打印时间戳
	      -e # 显示以太网帧头部
	      -X # 十六进制显示数据以及对应的 ASCII 字符
	      -s # 设置抓包截取长度
	      -S # 以绝对值显示 tcp 报文段序号
	      -w # 将数据重定向某个文件
	      -r # 从文件读取数据
	  ```
	
	  
	
- **lsof** 
	
	即 **list open file**  
	
	- `-i` 选项，查看套接字
	
		```shell
			# lsof -i [IP_PROTO] [TRANS_PROTO][@HOSTNAME/IP_ADDR]:[SERVICE/PORT]
		    # 示例
		    lsof -i@192.169.31.217:80
		```
	
	- `-u` 显示打开的所有套接字
	
		```shell
			lsof -u
		```
	
	- `-c` 显示指定程序打开的套接字
	
		```shell
			lsof -c ssh
		```
	
	- `-p` 显示指定进程打开的套接字
	
		```shell
			lsof -p [PID] 
		```
	
	- `-t` 显示打开了套接字的进程
	
		
	
- **nc** 
	
	即 **netcat** ，**debian** 中安装可以使用
	
	```shel
		apt install ncat
	```
	
	 之后我们就可以开始使用这个指令，有如下选项
	
	```shell
	    nc 	[HOSTNAME] [PORT] 		# 以客户端模式启动
	    nc -l [PORT]  				# 以服务端启动, 并监听指定端口 [PORT]
	    nc -lk [PORT] 				# 在 -l 基础上加上 -k 选项可以重复接受连接
	    nc -C 						# 使用 CRLF 作为行结束符
	```
	
	
	
- **strace**

	跟踪系统调用以及发送的信号

	```shell
		-c 	# 统计系统调用的执行时间、次数和出错次数
		-f 	# 跟踪由 fork 产生的子进程
		-t 	# 附加时间信息
		-e 	# 指定表达式, 详细见下文
	```

	`-e` 选项中被指定的表达式形式如下

	```shell
		[QUALIFIER]=[!][VALUE1], [!][VALUE2], ... # `!` 代表取反
	```

	- **[qualifier]** 有如下几种常见值
		- **trace**
		- **signal**
		- **read** 或 **write**
	- **[VALUE]** 有如下一些取值
		- **file**
		- **process**
		- **network**
		- **ipc**
		- **signal** (或者直接指定特定信号值，比如 **SIGINT** )

	接下来是一些使用 `-e` 选项的示例

	```shell
		strace -e trace=set 		# set代表了 open, write, read, close 四种简单调用
		strace -e trace=file
		strace -e trace=network
		strace -e trace=signal
		strace -e read=3, 5 		# 输出从文件描述符读到的数据
		strace -e signal=SIGINT
	```

	

- **netstat** 

	网络统计工具，可以统计网卡上的全部连接（相当于 `lsof`）以及路由表和网卡等信息。

	```shell
		-n 	# 使用 IP 号表示主机
		-a  # 统计的结果中包含监听套接字
		-r  # 显示路由信息
		-i  # 显示网卡接口的数据流量
		-c  # 每隔一秒输出一次
		-p  # 显示 socket 所属的进程的 PID
	```

	

- **vmstat**

	显示系统资源的指令 

	```shell
		-f 		 # 显示自启动以来系统 fork 的次数
		-s 		 # 显示内存相关信息以及系统活动信息
		-d 		 # 显示磁盘相关信息
		-p 		 # 统计分区信息
		-S 		 # 指定单位显示
		[delay]	 # 设置采样间隔秒数, 直接替换 [delay]
		[count]  # 设置统计次数, 直接替换 [count]
	```

	

- **ifstat** 

	检测流量速度

	```shell
		-a # 检测系统上所有网卡接口
		-i # 指定网卡接口
		-t # 加上时间戳
		-b # 以 bit/s 作为单位
	```

	

- **mpstat**

	**mp** 指 **multi-processor** , 用于监控 **CPU** 状况；下载 **sysstat** 包含这个指令

	```shell
	 	# 格式 -> mpstat -[CPU_ID | ALL] [INTERVAL] [COUNT]
	```

	比如我们可以这样使用

	```shell
	    mpstat -ALL 1 10 # 1s 输出一次, 总共十次
	```

	
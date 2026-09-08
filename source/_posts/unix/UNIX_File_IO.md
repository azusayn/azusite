---
title: UNIX 文件 I/O
date: 2023/9/13 16:58:00
categories:
- UNIX
tags:
- C/C++
- UNIX
---

### 1. 常用的文件操作符

包括 `open`, `read`,  `write`,  `lseek`,  `close`

- 标准输入输出和错误流有 `STDIN_FILENO` 、`STDOUT_FILENO` 和 `STDERR_FILENO` 等宏定义可以使用
- 一个进程的文件描述符最大值小于 `OPEN_MAX` （ 见 **POSIX** 限制）

- 使用 `open` 函数获取文件的描述符，成功则返回 **fd** , 否则返回 **-1** 。`oflag` 可以使用宏定义指定选项，例如 `O_RDWR | O_CREAT`

	```c
		#include <fcntl.h>
		int open(const char* path, int oflag, ...);
		
		int openat(int fd, const char* path, int oflag, ...);
	```

- 使用 `create` 函数创建文件

	```c
		int creat(const char* path, mode_t mode);
		// 实际上这个函数等价于 open(path, O_RDWR|O_CREAT|O_TRUNC, mode);
	```

- `close` 用于关闭文件，实际上进程终止时内核会自动释放所有打开的文件，并释放文件上的所有记录锁。

	```c
	int close(int fd);
	```

- 使用 `lseek` 为文件设置偏移量，下次读取文件会从这个位置开始读取数据；参数 `where` 有三个宏定义选项：

	- `SEEK_SET`，将文件偏移量设置为 `offset`
	- `SEEK_CUR`，将文件偏移量设置为 **当前值加上** `offset`
	- `SEEK_END`，将文件偏移量设置为 **文件长度加上** `offset`

	```c
		off_t lseek(int fd, off_t offset, int where);
	```

	有几点需要注意：

	- 除了以 `O_APPEND` 选项的 `open` 打开的文件之外，偏移量都是 **0** 
	- 另外不可打开管道或者套接字，否则返回 **-1** 。
	- 偏移量可以为负值，所以判断 `lseek` 是否执行成功只能判断是否为 **-1**
	- 偏移量大于文件长度则会形成空洞，但空洞的数据不分配内存
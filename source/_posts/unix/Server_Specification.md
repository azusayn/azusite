---
title: Linux 服务器规范
date: 2023/9/13 22:52:00
categories:
- UNIX
tags:
- C/C++
- UNIX
- NetWork
---

### 1. 系统日志

```c
	#include <syslog.h>
	void syslog (int priority, const char* message);
	
	void openlog(const char* ident, int logopt, int facility); // 比起 syslog() 提供更详细的参数选择

	int setlogmask
        
    void closelog();
```



### 2. 查看与设置系统资源限制

- `struct rlimt` 结构体存储着资源限制值，超过其值时内核分别对应发送 `SIGXFSZ` 和 `SIGXCPU` 信号

	```c
		struct rlimit {
	        rlim_t rlim_cur; // 建议值
	        rlim_t rlim_max; // 硬性要求最大值
	    }
	```

- 通过以下函数查看与设置：

	```c
        #include <sys/resource.h>
        int getrlimit(int resource, struct rlimit* rlim);
        int setrlimit(int resource, struct rlimit* rlim);
	```

- 另外可以通过 **shell** 命令 **ulimit** 设置当前**shell** 环境下资源限制，对当前 **shell** 启动的所有后续进程有效

### 3. 改变当前进程的工作目录与根目录

- 查看与修改工作目录

	```c
		#include <unistd.h>
		char* getcwd(char* buf, size_t size);
		int chdir(const char* path);
	```

- 修改根目录

	```c
		#include <unistd.h>
		int chroot(const char* path);
	```

	

### 4. 服务器程序后台化

- 直接使用 **Linux** 提供的库函数，设置 `nochdir` 为 `false` 会改变当前 **工作目录** 。`noclose` 代表是否关闭原来的标准输入输出以及错误流

	```c
		#include <unistd.h>
		int daemon(int nochdir, int noclose);
	```

- 手动实现，从父进程 `fork` 出子进程之后将父进程退出，将输入输出以及错误流都重定向到 **/dev/null** 中

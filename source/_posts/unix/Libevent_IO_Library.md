---
title: Libevent 入门
date: 2023/9/15 16:30:00
categories:
- UNIX
tags:
- C/C++
- UNIX
- NetWork
- CMake
---

### 0. 前言

- 初稿 *2023-9-17*，**Libevent** 的版本 **2.2.1-alpha-dev**，在 **Debian 12** 环境上进行测试
- 官方网站有详细的文档可以参考 [libevent.org](https://libevent.org/libevent-book/)

- 索引
	- [1. 安装](#install)
	- [2. 基本方法](#bu)
	- [3. 进阶方法](#au)

### 1. <span id="install">安装</span>

- 从 **github** 拉取，并安装前置 **openssl** 库

	```shell
		git clone https://github.com/libevent/libevent.git 
		sudo apt-get install libssl-dev libmbedtls-dev     # libevent 依赖于 openssl 
	```

- 在 **CMake** + **gcc 12.2.0** 的环境下进行编译

	```shell
		cd libevent
		mkdir build && cd build
		cmake -G "Unix Makefiles" ..  # 生成 makefile 
		make
		make verify 				  # 可以跳过的测试步骤
	```

- 接着将当前文件夹下的 **lib** 中的库移动到 **/usr/lib** 下，**include** 中的文件移动到 **/usr/include**；源代码根目录下的 **include** 移动到 **/usr/include** 中。

- 接着可以在 **CMakeLists.txt** 中引用，

	```cmake
		cmake_minimum_required(VERSION 3.0.0)
	    project(Test VERSION 0.1.0 LANGUAGES C CXX)
	
	    include_directories(/usr/include/libevent) # 我把libevent文件放在了单独的文件夹中
	    link_directories(/usr/lib/libevent)		
	
	    add_executable(Main main.cc)
	
	    target_link_libraries(Main
	        event
	    )
	```

- 我们可以使用以下函数检查当前环境 **Libevent** 支持的后端方法以及版本

  ```c++
  	const char** event_get_supported_methods(void);
  	const char** event_get_version();
  ```

### 2. <span id="bu">基本方法</span>

#### 2.1 初始化

- **libevent** 中通过下列方法初始化和清理一个 **reactor** 模型

	```c++
		struct event_base* event_base_new();
		void event_base_free(struct event_base *base);
	```

- 如果需要更多客制化，也可以通过 `event_base_new_with_config()`  进行初始化，通过一个 `event_config` 结构体设置更多初始化参数。

  - 函数声明

  	```c++
  		struct event_base* event_base_new_with_config(struct event_config* cfg);	
  	```

  - 通过专门的函数设置 `event_config` 结构体。以下方法成功返回 **0** , 否则返回 **-1**

  	- 初始化与释放

  		```c++
  			// 初始化
  			struct event_config *event_config_new(void);
  			// 已经传入 event_base 实例的 event_config 实例可以直接释放
  			void event_config_free(struct event_config *cfg);	
  		```
  		
  	- 设置不需要的方法，比如可以传入 `"epoll"`
  	
  		```c++
  			// 1. 设置不需要的方法, 比如可以传入"epoll"
  			int event_config_avoid_method(struct event_config *cfg, const char *method);
  		```
  	
  	 - 让 event 保证支持下列参数
  	
  		```c++
  			// 2. 让 libevent 保证支持以下参数的一种或多种 (可以进行‘或’运算进行组合)
  		    enum event_method_feature {
  		        EV_FEATURE_ET = 0x01,
  		        EV_FEATURE_O1 = 0x02,    // 保证增删事件复杂度 O(1)
  		        EV_FEATURE_FDS = 0x04,   // 支持除了 socket 以外的普通文件描述符
  		    };
  		    int event_config_require_features(struct event_config *cfg,
  		                                      enum event_method_feature feature);
  		```
  	
  	 - 更多运行时参数
  	
  		```c++
  			// 3. 更多运行时参数
  		    enum event_base_config_flag {
  		        EVENT_BASE_FLAG_NOLOCK = 0x01,
  		        EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
  		        EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,
  		        EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,
  		        EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10,
  		        EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
  		    };
  		    int event_config_set_flag(struct event_config *cfg,
  		        enum event_base_config_flag flag);
  		```
  	
  	 - 控制线程使用的 **CPU** 数，但是只有 **Windows** 的**IOCP** 支持
  	
  		```c++
  			int event_config_set_num_cpus_hint(struct event_config *cfg, int cpus);
  		```
  	
  	 - *预防 **Priority Inversion** 
  	
  		```c++
  		    int event_config_set_max_dispatch_interval(struct event_config *cfg,
  		        const struct timeval *max_interval, int max_callbacks,
  		        int min_priority);
  		```
  
- 对于初始化的模型，可以设置 `event_base` 的优先级并检查

	```c++
		int event_base_priority_init(struct event_base *base, int n_priorities);
		int event_base_get_npriorities(struct event_base *base);
	```

- 检查某个初始化的 `event_base` 结构体使用的方法

  ```c++
      const char *event_base_get_method(const struct event_base *base);
      enum event_method_feature event_base_get_features(const struct event_base *base);
  ```

- 对于 `fork` 产生的子进程中的 event_base，需要使用以下函数进行重新初始化

	```c++
		int event_reinit(struct event_base *base);
	```

#### 2.2 事件注册

##### 2.2.1 一般的注册方式

- 对于一个事件有多个状态，根据官方文档 

	> Events have similar lifecycles. Once you call a Libevent function to set up an event and associate it with an event base, it becomes **initialized**. At this point, you can add, which makes it **pending** in the base. When the event is pending, if the conditions that would trigger an event occur (e.g., its file descriptor changes state or its timeout expires), the event becomes **active**, and its (user-provided) callback function is run. If the event is configured **persistent**, it remains pending. If it is not persistent, it stops being pending when its callback runs. You can make a pending event non-pending by deleting it, and you can add a non-pending event to make it pending again.

- 使用 `event_new()` 函数创建事件，`event_free()` 释放事件；其中创建事件需要的参数

	- `base` ：指向创建的 `event_base` 实例

	- `fd`：监听的事件描述符

	- `what` : 指定事件类型（什么情况下被触发）以及附加选项，包含以下几种宏定义

		```c++
			#define EV_TIMEOUT      0x01   // 经过一定时间之后事件变为 Activated 状态
		    #define EV_READ         0x02	
		    #define EV_WRITE        0x04
		    #define EV_SIGNAL       0x08
		    #define EV_PERSIST      0x10	// 被激活后仍然保持 Pending 状态
		    #define EV_ET           0x20
		```

	- `cb` 和 `arg`，回调函数以及其传入参数

	```c++
	    typedef void (*event_callback_fn)(evutil_socket_t, short, void *);
	
	    struct event *event_new(struct event_base *base, evutil_socket_t fd,
	        short what, event_callback_fn cb,
	        void *arg);
	
	    void event_free(struct event *event);
	```

- 根据文档，创建的事件将会处于 **Initialized** 状态，但是不会被触发，除非我们将它设置为 **Pending** 状态。同时已经触发回调函数的事件将需要重新设置为 **Pending** 状态，除非它被设置了 `EV_PERSIST` 

	```c++
		int event_add(struct event *ev, const struct timeval *tv);
	```

- 删除事件将会使它们变为 **Non-Pending** 状态

	```c++
		int event_del(struct event *ev);
	```

- 同时可以设置事件的优先级

    ```c++
		int event_priority_set(struct event *event, int priority);
    ```

	

##### 2.2.2 便捷的函数

为了方便使用，针对特定事件类型，**Libevent** 设置特殊的宏定义或者函数便于使用。如下是两个特化的事件注册函数

- 定时事件

	```c++
	    #define evtimer_new(base, callback, arg) \
	        event_new((base), -1, 0, (callback), (arg))
	```

- 信号事件，注意一个进程只能有一个信号事件（详细请查阅操作系统原理）

	```c++
		#define evsignal_new(base, signum, cb, arg) \
	    	event_new(base, signum, EV_SIGNAL|EV_PERSIST, cb, arg)
	```

#### 2.3 设置事件监听

当我们注册好要监听的事件以及属性之后，我们便可以开始监听事件。前者类似于在内核事件表中注册事件，后者则更像是调用 `epoll_wait` 进行监听

##### 2.3.1 开启监听

- `event_base_loop()` ，其中的 `flags` 可以设置为下列三个值：

	- `EVLOOP_ONCE` ：运行到事件完成之后便返回
	- `EVLOOP_NONBLOCK`： 只是检查事件是否有事件就绪，直接返回
	- `EVLOOP_NO_EXIT_ON_EMPTY`： 就算就绪队列为空也不会返回

	```c++
		int event_base_loop(struct event_base *base, int flags);
	```

- `event_base_dispatch()` 可以看作是 `event_base_loop` 的简化版本，相当于设置了`EVLOOP_ONCE` 

	```c++
		int event_base_dispatch(struct event_base *base);
	```

	

##### 2.3.2 关闭监听

一般的事件监听 (没有设置 `EVLOOP_NO_EXIT_ON_EMPTY` 的前提下) 都会在就绪事件队列为空的时候主动退出，但假如我们想在队列中还有事件的时候就强行退出，可以使用以下两个函数；与上述开启监听的两个函数类似，这两个函数只是调用接口略有不同

- `event_base_loopexit` ：设置一段时间之后退出

	```c++
		    int event_base_loopexit(struct event_base *base,
	                            	const struct timeval *tv);
	```

- `event_base_loopbreak` ：相当于 `event_base_loopexit(struct event_base *base, NULL)` ，立即退出

	```c++
	    int event_base_loopbreak(struct event_base *base);	
	```

- 通过下列两个函数检查事件监听是否正常退出

	```c++
		int event_base_got_exit(struct event_base *base);
	    int event_base_got_break(struct event_base *base);
	```


### 3. <span id="au">进阶方法</span>

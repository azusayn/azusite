---
title: 在 CMake 工具链中加入vcpkg
date: 2023/12/18 16:40:00 
categories:
- C/C++
tags:
- C/C++
- CMake
- Deep Learning
---
###  摘要

这篇文章继续记录我与 **C++** 生态打架的精彩记录，主要记录一些配置文件的编写，便于日后参考第三方库的安装。



### 安装 vcpkg

与终端交互中完成如下配置（刚装的系统会有一些基础依赖需要按照报错提示自行安装）

```bash
	# 下载并初始化
	git clone https://github.com/microsoft/vcpkg.git
	cd vcpkg && ./bootstrap-vcpkg.sh
	# 设置环境变量并刷新
	export VCPKG_ROOT=/path/to/vcpkg # 这里要自己替换路径, 个人喜欢放到 ~/.bashrc 中永久保存
	export PATH=$VCPKG_ROOT:$PATH
    source ~/.bashrc
```



### 开始使用 vcpkg

1. 这里参考官方的例子先给出基本的 **CMakeLists.txt** 文件和 **main.cc** 文件

	**CMakeLists.txt**

	```cmake
	    cmake_minimum_required(VERSION 3.10)
	
	    project(HelloWorld)
	
	    find_package(fmt CONFIG REQUIRED)
	
	    add_executable(Main main.cc)
	
	    target_link_libraries(HelloWorld PRIVATE fmt::fmt)
	```

	**main.cc**

	```c++
		#include <fmt/core.h>
	
	    int main() {
	        fmt::print("Hello World!\n");
	        return 0;
	    }
	```

	

2. 由于网络时不时会遭受神秘力量的攻击, 这里可以通过设置环境变量更换镜像源 

```bash
    export X_VCPKG_ASSET_SOURCES=x-azurl,http://106.15.181.5/
    source ~/.bashrc
```



3. 为了能让 **CMake** 工具链加入 **vcpkg**，需要设置  `CMKAE_TOOLCHAIN_FILE`  这个值，官网给出了三种方法

- 在 CMakePresets.json 中进行设置

```cmake
    {
        "version": 3,
        "configurePresets": [
            {
                "name": "default",
                "binaryDir": "${sourceDir}/build",
                "cacheVariables": {
                    "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
                }
            }
        ]
    }
```

- 在 CMakeLists.txt 中进行设置，不过要写在 `project()` 这一行之前

```cmake
	set(CMAKE_TOOLCHAIN_FILE "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake")
```

- 在使用cmake指令进行工程构建的时候加入 `-D` 选项

```bash
	cmake -DCMAKE_TOOLCHAIN_FILE="$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
```



4. 在希望使用 **vcpkg** 的项目中使用如下指令初始化 **vcpkg** 管理

```bash
    vcpkg new --application
```




5. 之后可以使用如下命令进行依赖的添加，比如我这里需要添加 **fmt** 库，回车之后会发现 **vcpkg.json** 中多了这一项依赖

```bash
    vcpkg add port fmt 
```



6. 之后就可以使用 **cmake** 指令进行构建编译了，加上 `--preset` 选项设置配置文件为上述 **CMakePresets.json** 中的 **default** 配置

```bash
    mkdir build && cd build
    cmake --preset=default ..
    cmake --build .
```



7. 构建成功即可运行（实际上大概率遇到一堆小问题，这里就即兴发挥了 :happy:

```bash
	./Main
	# Hello World!
```



### 注

- 实际上 **vcpkg** 安装包有两种模式：**classic** 和 **manifest** (上文使用的就是这种方式，也是官方推荐的 )

	- **manifest:** 相当于每次安装库的位置都是项目 **build** 文件夹内


	- **classic:** 相当于全局安装，库安装在在 `VCPKG_ROOT`的文件夹中；在设置了 **vcpkg** 工具链的项目中， 每次安装都会从这个全局目录中寻找缓存，大大加快安装速度；该模式安装使用另一条指令

		```bash
			vcpkg install <lib> 
		```


- 另外贴上 **vcpkg** 页面 https://learn.microsoft.com/en-us/vcpkg/

​      






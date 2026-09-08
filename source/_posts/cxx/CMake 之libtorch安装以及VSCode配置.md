---
title: LibTorch 安装以及 VSCode 配置
date: 2023/11/8 12:04:00 
categories:
- C/C++
tags:
- C/C++
- Deep Learning
- CMake
---
###  前言 

最近在学 **CMake** 和 **UNIX** ，不幸被老师抓去做 **CV** 和深度学习，（~~摸鱼的时候~~） 正巧发现 **Pytorch** 有开放的  **C++ API** 可以使用，便想着装来玩玩（~~一装装一下午~~）

**LibTorch** 是深度学习框架 **PyTorch** 开放的 **C++ API** ，本文将使用 **CMake**，针对 **Linux** 环境以及 **Windows** 环境对 **LibTorch** 库进行安装。目的是记录一些踩过的坑，并对 **LibTorch** 简洁的 **README**做一点补充解释  (~~他们甚至让我用 **cmake-gui**~~ ) 

原文档 -> https://pytorch.org/cppdocs/installing.html



### 前置条件

- 首先电脑上一定要装有 **CMake** 以及并配置好某个版本的 **C++** 编译器路径

- 附加地，如果想要在 **VSCode** 中愉快地写代码，需要在扩展安装 **CMake** 、 **CMakeTools** 、 **C\C++** 这三个插件，以便识别 **CMake** 工程和代码提示；喜欢终端可以直接跳过这一项

- 附加地，如果需要显卡计算，电脑上一定要先安装好 **CUDA** 和 **cuDNN** 库，并且在下文的下载步骤中选择好对应 **CUDA** 版本的库下载

 

### Linux 环境

1. 在官网 https://pytorch.org/get-started/locally/ 根据交互界面获取分发地址，下载并解压

    ```bash
        # 我的 CUDA 版本是 11.8, pre 代表预览版本
        curl -O https://download.pytorch.org/libtorch/cu118/libtorch-shared-with-deps-2.1.0%2Bcu118.zip
        unzip libtorch-shared-with-deps-2.1.0%2Bcu118.zip
    ```

2. 编写 **CMakeLists.txt** 文件

    ```cmake
        cmake_minimum_required(VERSION 3.10)
        project(example-app)
    
        find_package(Torch REQUIRED)
        set(CMAKE_CXX_FLAGS, "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS}")
    
        add_executable(example-app example-app.cpp)
    
        target_link_libraries(example-app "${TORCH_LIBRARIES}")
        set_property(TARGET example-app PROPERTY CXX_STANDARD 17)
    
        # target_include_directories(example-app PUBLIC "${TORCH_INCLUDE_DIRS}")
    
        # The following code block is suggested to be used on Windows.
        # According to https://github.com/pytorch/pytorch/issues/25457,
        # the DLLs need to be copied to avoid memory errors.
        if (MSVC)
          file(GLOB TORCH_DLLS "${TORCH_INSTALL_PREFIX}/lib/*.dll")
          add_custom_command(TARGET example-app
                             POST_BUILD
                             COMMAND ${CMAKE_COMMAND} -E copy_if_different
                             ${TORCH_DLLS}
                             $<TARGET_FILE_DIR:example-app>)
        endif (MSVC)
    ```

3. 编写 **example-app.cpp** 检查环境是否配置正确，我在官方给出代码的基础上多加了一句对 **CUDA** 的检查

    ```c++
        #include <torch/torch.h>

        int main() {
            torch::Tensor tensor = torch::rand({2, 3});
            std::cout << tensor << std::endl;
            std::cout << std::boolalpha << torch::cuda::is_available() << std::endl;
            return 0;
        }  
    ```

4. 编译

- 官方给出的方法

    ```bash
        # 设置下载的 libtorch 文件夹绝对路径
        cmake .. -DCMAKE_PREFIX_PATH="absolute/path/to/libtorch" 
    ```

- 如果嫌麻烦也可以直接把路径写在 **CMakeLists.txt** 里，或者在 **VSCode** 的 **CMake** 插件中设置附加生成选项。比如前者可以这么写：

    ```cmake
    	set(CMAKE_PREFIX_PATH "absolute/path/to/libtorch" )
    	# 或者把路径加入环境变量(我取名为 LIBTORCH)，用如下方式调用 
    	set(CMAKE_PREFIX_PATH "$ENV{LIBTORCH}") 
    ```

- 接着就可以直接使用下面指令生成了，或者可以通过 **CMake** 插件的按钮生成或者运行

	```cmake
		cmake --build .
	```

	

### Windows 环境

1. 在官网 https://pytorch.org/get-started/locally/ 根据交互界面获取分发地址，直接点击下载；

    ```bash
        # 安装 CPU 的 release 版本; Debug 版本会带有调试信息，但编译会稍慢。
        https://download.pytorch.org/libtorch/cpu/libtorch-win-shared-with-deps-2.1.0%2Bcpu.zip
    ```

2. 接着我们可以使用和 **Linux** 环境下一样的源文件和 **CMakeLists.txt**。


3. 编译方法与 **Linux** 环境下一样

    

### VSCode 代码提示

- 按照以上的步骤安装完之后会发现 **VSCode** 无法给出代码提示，于是我们就需要配置 **.vscode/c_cpp_properties.json** 文件,  在 `"includePath"` 中添加头文件的位置，**VSCode** 会检查这个文件并进行代码提示。

    ```json
    	{
            "configurations": [{
                "name": "Linux",
                "includePath": [
                    "${workspaceFolder}/**",
                    "/absolute/path/to/libtorch/**"  
                ]
            }],
            "version": 4
        }
    ```

- 如果在 **Windows** 环境下这里可以加入环境变量简化路径，在上文 **Windows**  环境下我配置了 `LIBTORCH` 这个环境变量，于是可以把头文件位置写成:

    ```json
    	 "${env:LIBTORCH}/**"  
    ```

    


### 注

- 如果用 **CMake** 插件的按钮进行生成出错，可以用在 **CMakeLists.txt** 中使用 `message(${CMAKE_PREFIX_PATH})` 检查环境变量是否正确导入。如果没有大概是因为 **VSCode** 保存着先前的环境变量。可以重启 **VSCode**，删除 **build** 文件夹重新生成 。（可见命令行的重要性 :XD）

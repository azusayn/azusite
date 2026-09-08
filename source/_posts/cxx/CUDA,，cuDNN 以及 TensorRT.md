---
title: CUDA，cuDNN 和 TensorRT
date: 2023/12/26 16:16:00 
categories:
- C/C++
tags:
- C/C++
- CMake
- Deep Learning
---

###  摘要

- **CUDA** (Compute Unified Device Architecture) 是 **N** 卡上的计算框架，在这个框架上可以装载 **cuDNN** 或者 **TensorRT** 进行深度学习模型的加速。前者主要用于模型训练的加速，而后者适用于部署时模型推理的加速。

- 这篇文章主要记录 **Linux** 下 **CUDA**，**cuDNN** 以及 **TensorRT** 的安装；

	

### CUDA

- 先去 [ CUDA 的页面 ]( https://developer.nvidia.com/cuda-downloads?target_os=Linux ) 进行安装，另外一提 **/etc/issue** 中记录了当前 **Linux** 的发行版本，可以对照选择下载连接。因为是在 **docker** 中运行这个 **CUDA** 环境，所以不需要安装驱动，而是依靠物理主机 ( 我的 **Windows** ) 上的驱动 。在 **x86 > Debian 12 > deb (local) ** 的选择下有了这些指令

	```bash
	    wget https://developer.download.nvidia.com/compute/cuda/12.3.1/local_installers/cuda-repo-debian12-12-3-local_12.3.1-545.23.08-1_amd64.deb
	    sudo dpkg -i cuda-repo-debian12-12-3-local_12.3.1-545.23.08-1_amd64.deb
	    sudo cp /var/cuda-repo-debian12-12-3-local/cuda-*-keyring.gpg /usr/share/keyrings/
	    sudo add-apt-repository contrib # 这个指令可能需要先 apt install software-properties-common
	    sudo apt-get update
	    sudo apt-get -y install cuda-toolkit-12-3
	```
	



### cuDNN

- 官方给出了 [完整说明](https://docs.nvidia.com/deeplearning/cudnn/install-guide/index.html#install-linux)，在 **CUDA** 的基础上我们需要先安装 **zlib** 依赖

	```bash
		apt install zlib1g
	```

- 接下来要去 [cuDNN 主页](https://developer.nvidia.com/cudnn) 手动下载安装包，其中 **tar** 安装包适用于所有 **Linux** 分发版本，所以我选择了 **tar** 包

- 解压下载的 **tar** 包

	```bash
		tar -xvf cudnn-linux-x86_64-8.9.7.29_cuda12-archive.tar.xz
	```

- 将文件放到 **CUDA** 目录中并设置文件权限

	```bash
	    sudo cp cudnn-linux-x86_64-8.9.7.29_cuda12-archive/include/cudnn*.h /usr/local/cuda/include
	    sudo cp -P cudnn-linux-x86_64-8.9.7.29_cuda12-archive/lib/libcudnn* /usr/local/cuda/lib64
	    sudo chmod a+r /usr/local/cuda/include/cudnn*.h /usr/local/cuda/lib64/libcudnn*
	```

	

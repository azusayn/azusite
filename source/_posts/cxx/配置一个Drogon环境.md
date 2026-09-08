---
title: 配置一个 Drogon 环境
date: 2023/11/25 20:09:00 
categories:
- C/C++
tags:
- C/C++
- CMake
- web
---
###  前言 

偶然发现了 **TechEmpower** 这个后端框架评分网站，发现了一位重量级选手，2020年综合评分的冠军 [**Drogon**](https://www.techempower.com/benchmarks/#hw=ph&test=composite&section=data-r19)，而这两年 **Just.js** 和 **Rust** 的两个框架霸榜，某 **++** 语言只能靠边观摩。今年更是因为测试不规范，被抬出了party。今天发现之前装框架的时候忘记装数据库接口了，这次干脆给自己写个 **Dockerfile** 打造一个专用的开发环境，也方便之后参考。



### 镜像源

我把我的镜像上传到了 **Docker Hub**，有缘人可以拉下来玩。当然我一般还会自己配置 **ssh** 和 **VSCode** 开发，这里为了整洁就不将配置写进去了。

```bash
 	docker pull azusaing/drogon:basic # 解压后 1.79 GiB
```



###  菜谱

花了我 3 分钟 build 的菜谱

```dockerfile
    # +----------------------------------+
    # |  @author:   azusaings@gmail.com  | 
    # |  @date:     2023-11-25           | 
    # |  @version:  0.0.1                |
    # +----------------------------------+

    FROM debian:latest
    WORKDIR /root
    RUN apt update                                                                          && \
    # basic tools
        apt install -y                                                                         \    
        git                                                                                    \
        gcc                                                                                    \
        g++                                                                                    \
        cmake                                                                                  \
        valgrind                                                                               \
        ssh                                                                                    \
        vim                                                                                    \
        wget                                                                                && \   
    # drogon dependencies                                                                        
        apt install -y                                                                         \
        libjsoncpp-dev                                                                         \
        uuid-dev                                                                               \   
        zlib1g-dev                                                                             \
        openssl                                                                                \
        libssl-dev                                                                             \
        postgresql-server-dev-all                                                              \
        default-libmysqlclient-dev                                                             \
        libsqlite3-dev                                                                         \
        libhiredis-dev                                                                      && \
    # drogon                    
        git clone https://github.com/drogonframework/drogon                                 && \
        cd drogon                                                                           && \
        git submodule update --init                                                         && \
        mkdir build                                                                         && \
        cd build                                                                            && \
        cmake ..                                                                            && \
        make -j $(nproc)                                                                    && \
        make install                                                                        && \
        cd ../..                                                                            && \ 
        rm -rf drogon/                                                                      && \
    # curl            
        wget https://github.com/curl/curl/releases/download/curl-8_4_0/curl-8.4.0.tar.gz    && \
        tar -xzvf curl-8.4.0.tar.gz                                                         && \                                 
        cd curl-8.4.0                                                                       && \
        ./configure --with-openssl --prefix=/usr/                                           && \     
        make -j $(nproc)                                                                    && \
        make install                                                                        && \
        cd ..                                                                               && \ 
        rm -rf curl-8.4.0/ curl-8.4.0.tar.gz                                                && \
        apt remove -y wget                                                                  && \
        apt autoclean                                                                       

```




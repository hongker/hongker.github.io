---
title: docker介绍与安装
date: 2018-12-11 22:02:10
tags: docker
---
本篇文章主要介绍微服务的利器：Docker的使用。
[TOC]

## 什么是docker
引用百度百科的说明：
>Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。

简单的说，就是将应用程序与运行环境合为一体，以docker容器的方式运行。

更多内容请查看 docker官方地址: [https://www.docker.com](https://www.docker.com)

## Docker的优点
- 应用隔离性
使用Docker运行应用，通过使用一个容器运行一个应用，实现应用间的解耦，避免单服务器直接运行多个应用，方便管理与维护。

- 环境标准化
一个docker镜像不管在何处运行，其内部环境都是一致不变的，故不必担心应用移植时发生环境不对导致应用运行异常的情况。

- 部署简单化
一条run命令即可运行一个容器，so easy。

## 安装Docker CE
以下教程以ubuntu18.04 作为运行环境进行安装，其他环境请查看 [官方文档](https://docs.docker.com/install/)

- 移除旧版本(非必要)
```
sudo apt-get remove docker docker-engine docker.io
```

- 安装前提软件
```
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```
- 添加官方GPG Key
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

- 开始安装Docker CE(社区版)
```
sudo apt-get install docker-ce
```

- 查看Docker 版本
```
sudo docker version
```
输出如下:
```
Client:
 Version:           18.09.0
 API version:       1.39
 Go version:        go1.10.4
 Git commit:        4d60db4
 Built:             Wed Nov  7 00:49:01 2018
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.0
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.10.4
  Git commit:       4d60db4
  Built:            Wed Nov  7 00:16:44 2018
  OS/Arch:          linux/amd64
  Experimental:     false
```

至此安装成功!

## 其他说明
### 免去sudo
鉴于每次docker命令都要使用超级权限，所以每次都得在前面加sudo,为此我们通过以下命令，去掉烦人的sudo.
- 如果还没有 docker group 就添加一个
```
sudo groupadd docker
 ```

- 将用户加入该 group 内

 ```
sudo usermod -aG docker $USER
```

- 重启Docker
```
sudo service docker restart
```

- 切换当前会话到新 group
```
newgrp - docker
```
运行成功后，登出当前用户重新登录。再次输入 `docker verion`，即可。

### 使用中国镜像
国外的镜像拉取比较慢，我们通过配置中国官方镜像，可以明显的提高镜像拉取速度。
这里选择修改 /etc/docker/daemon.json 文件的方式。添加上 registry-mirrors 键值。
- 打开文件
```
sudo vim /etc/docker/daemon.json
```
- 写入以下内容
```
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
```

- 重启Docker
```
sudo systemctl restart docker
```
- 查看是否生效
通过`docker info`查看是否生效，如下:
```
Registry Mirrors:
 https://registry.docker-cn.com/

```
配置已生效。

下一篇将介绍docker的相关命令。
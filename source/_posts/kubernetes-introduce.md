---
title: kubernetes介绍与安装
date: 2021-04-06 20:00:33
tags: kubernetes
---
本文作为kubernetes系列的开篇，主要介绍一些相关概念。

## 什么是kubernetes
>kubernetes，简称K8s。是一个开源的，用于管理云平台中多个主机上的容器化的应用，Kubernetes的目标是让部署容器化的应用简单并且高效（powerful）,Kubernetes提供了应用部署，规划，更新，维护的一种机制。

在服务化时代，每个公司都会有很多个分散的项目，我们通过k8s来统一管理这些应用，不但可以提高服务的部署效率，还能提高服务的横向扩展能力。

## k8s & docker

- docker: 应用容器引擎，作为目标程序的载体，可移植，达到运行环境一致性。
- k8s: 容器集群管理系统，实现容器集群的自动化部署，扩缩容，负载均衡等。


在我看来,docker就像“出租车”，k8s就是“出租车管理公司”。

## 相关概念
### Master
集群控制节点。负责整个集群服务的的管理和控制。Mater节点上运行以下组件：
- kube-apiserver: http服务，对k8s中所有资源进行管理(增删改查)，是集群控制入口进程。kubectl是访问apiserver的客户端工具。

- kube-control-manager: apiserver为对外开放的应用服务，与之相对的后台服务就是kube-control-manager,负责k8s里所有资源对象的控制。

- kube-schedule: 资源调度器，负责调度pod到node节点上。
- etcd: 存储各种资源的状态数据。
### Node
除master节点外的其他机器节点。是集群的工作负载节点。Node节点上运行以下组件：
- kubelet: 负责pod对应的容器创建、启动、停止、资源监控等任务。
- kube-proxy: 实现service的通信与负载均衡功能。
- docker: 容器runtime。

### Deployment
当我们需要创建pod来运行应用的时候，不是直接创建Pod,而是创建一个Deployment，用它负责创建和更新应用。可以理解为pod的"管家"。

### Service
因为pod可能会不停的创建和销毁，其IP也会随之变化，我们想要在外部固定的访问pod上运行的应用，就需要service来暴露pod里的应用。

service定义了一组pod的逻辑集合和访问策略，不管内部的pod如何变化，我们只要保证service的访问方式不变，其内部会通过服务注册/发现来定位到pod应用。

### Pod
- 在k8s里最小的逻辑运行单元，简单的说就是容器集合。
- 相同pod里运行的多个容器，共享uts、net、ipc空间等。


## 安装k8s集群环境
通过`kind`安装k8s cluster,下载：
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.10.0/kind-linux-amd64
chmod +x ./kind
mv ./kind /usr/bin/kind
```

- 创建集群
```
kind create cluster
```

- 查看集群信息
```
kubectl cluster-info
```

- 查看节点
```
kubectl get nodes
```
- 删除集群
```
kind delete cluster
```

k8s环境已准备就绪。下一章将介绍`如何用k8s部署go的web应用`，敬请期待！
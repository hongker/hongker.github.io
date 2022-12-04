---
title: kubenetes架构设计初探
date: 2022-12-04 21:31:58
tags: kubernetes
---

本文将初步探索k8s的架构设计。

## 架构图
![architecture](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)

## 组件说明
### 控制平面组件(ControlPlaneComponents)
主要运行在Master节点，为k8s集群做出全局决策，如资源调度、服务检测、集群事件响应等。

- kube-apiserver
该组件管理着k8s的开放接口，是k8s集群的前端控制，负责处理内部集群与外部客户端的请求。

- etcd
该组件为高可用的键值存储数据库，负责存储k8s集群的所有资源配置数据。

- kube-schedule
该组件负责监听新创建且未指定Node的Pods,并选择某个Node来运行Pod。

调度策略由Pod定义的资源需求、软硬件需求、亲和性、反亲和性、工作负载等因素决定。


- kube-controller-manager
该组件负责运行控制器进程，包含多个控制器，主要包括：
    - 节点控制器(NodeController): 负责在节点出现故障时进行通知和响应。
    - 任务控制器(JobController): 监测代表一次性任务的Job对象，然后创建Pods来运行任务直至完成。
    - 端点分片控制器(EndpointSliceController): 填充EndpointSlice对象，提供Service与Pod之间的链接。
    - 服务账号控制器(ServiceAccountController): 为新的命名空间创建默认的服务账号。


### Node组件
运行在每个worker节点上，负责维护运行Pod以及提供k8s运行环境

- kubelet
该组件通过各类机制提供给它的PodSpecs，确保描述的容器处理运行状态且健康。

- kube-proxy
运行在每个节点上的网络代理，负责流量转发，允许集群内部或外部直接与pod进行通信。

- Container Runtime
k8s支持多种容器运行环境，如containerd,docker等

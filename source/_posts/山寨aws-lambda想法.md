---
title: 山寨aws lambda想法
date: 2019-04-10 11:07:35
tags:
---

### 结构
根据[AWS re:Invent 2018: [REPEAT 1] A Serverless Journey: AWS Lambda Under the Hood](https://www.youtube.com/watch?v=QdzV04T_kec)。
aws lambda 可以分为Controller Plane 和 Data Plane两大部分
Data Plane主要负责函数的调用，分为Synchronous Invoke 和 Asynchronous Invoke & Events
Synchronous Invoke 分为以下几部分

#### Front End Invoke
此模块负责编排所有请求，包括同步和异步的,对请求进行鉴权

#### Counting Service
负责维护函数的当前并发数，aws宣称其使用 Quorum based protocol
aws将其设计成一个高吞吐低延时(1.5ms)的服务
目前看来是所有用户的调用都会经过这个Counting Service来做限流

#### Worker Manager
管理Worker，跟踪sandbox的idle/busy状态，调度请求到可用的容器，或者选择Worker来创建容器
如果容量不够，会请求Placement Service获取新的Worker
当worker 和 sandbox 变成idle的时候会spin down来节省资源
同时会优先使用warm sandbox
有一个问题是当worker manger扩容的时候，warm sandbox的复用率会降低
同时如何防止用户的代码在后台运行

#### Worker
创建和管理sandbox，设置sandbox的cpu 和 memory limit, 当sandbox上的请求结束之后通知worker manager


#### Placement Service
放置sandbox到worker上，原则是一个worker要放置尽量多不同的sandbox以达到最大的资源使用率。
根据后述的演讲，它貌似是管理worker的，worker manager 从placement service 租用worker。
问题：如果这个Placement Service是选择worker来放置sandbox的，那么貌似承担了一部分worker manager的职责


#### 想法
目前他们所用的sandbox[Firecracker](https://github.com/firecracker-microvm/firecracker)也开源出来了。  
根据这个架构，可以一个个模块地实现一个lambda，从worker开始
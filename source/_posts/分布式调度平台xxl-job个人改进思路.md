---
title: 分布式调度平台xxl-job个人改进思路
date: 2019-9-8 23:02:42
tags:
- Java
- xxl-job
- 任务调度
- 分布式
- 高可用
categories:
- Java
---

# 分布式调度平台xxl-job个人改进思路   
> 本人**刚入门**后端开发，__错误之处请批评指正__  
> 被导师安排的🌚  
> 本人于2019年9月6日与同事进行的分享  

## 1 xxl-job是什么  
### 1.1 xxl-job是什么  
* 轻量级、易扩展的分布式任务调度框架
* 通过Cron表达式配置计划任务
	`0 0/30 9-18 ? * MON-FRI` 朝九晚六每半个小时执行
* 支持多语言（Java、Shell、Python、NodeJS、PHP、PowerShell等，需要执行器部署环境支持），任务逻辑可在Web界面编写代码，或在执行器编写代码

### 1.2 常见任务调度框架
* Quartz  
    Java常用计划任务框架，虽然Quartz可以基于数据库实现作业的高可用，但分布式并行调度方面有所欠缺。
* elastic-job  
    当当开发的弹性分布式任务调度系统，功能丰富强大，采用zookeeper实现分布式协调，实现任务高可用以及分片。
* xxl-job  
    是大众点评员工徐雪里于2015年发布的分布式任务调度平台，是一个轻量级分布式任务调度框架，其核心设计目标是开发迅速、学习简单、轻量级、易扩展。

### 1.3 xxl-job 与 elastic-job 怎么选  

xxl-job
* 核心设计目标是开发迅速、学习简单、轻量级、易扩展
* 登记在用公司数>228家
* 开箱即用
* 持续更新，社区活跃、文档齐全

![xxl-job GitHub](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/github-xxl-job.png)

elastic-job
* 初衷面向高并发复杂业务
* 部署、配置、使用相对复杂
* Last commit 一年半前
* Latest release 2017.07.10

![elastic-job GitHub](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/github-elastic-job.png)

### 1.4 xxl-job 单节点基本架构

* 调度中心业务无关
* 执行器业务相关，实现业务逻辑
* 使用GLUE模式可以直接在监控界面上写业务代码
![xxl-job单节点架构图]()

## 2 xxl-job HA  

### 2.1 调度中心HA & 执行器HA  

前提：
* 每个执行器节点需要在配置中配置所有调度中心地址

缺点：
* 调度中心地址写死，不便于节点数量调整

![HA without Nginx]()

### 2.2 引入`Nginx`  

已有条件：  
* 调度中心通过数据库维护执行器信息，即执行器无须向所有调度中心注册，只须向其中一个节点注册  

引入后：  
* 实现负载均衡、故障迁移等  
* 修改Nginx配置即可调整节点  
* 执行器注册到调度中心走Nginx，调度中心调度任务仍然直接访问执行器  

![HA with Nginx]()

### 2.3 执行器集群 & 可用性  

* 执行器节点部署简单，不依赖数据库，只须确保可访问调度中心以及配置文件正确即可
* 配置任务时，可以配置各种路由策略（包括：第一个、最后一个、轮询、随机、一致性HASH、最不经常使用、最近最久未使用、故障转移、忙碌转移等），调度时任务根据策略路由到对应执行器节点
* 执行器高可用：直接水平扩展

### 2.4 调度中心集群 & 可用性

* 调度中心集群的节点之间相对独立、透明，不会感知调度中心其他节点的存在，通过分布式锁进行同步
* 调度中心各节点数据库等配置保持一致
* 调度中心各节点所在主机的时钟必须同步（使用NTP）
* 调度中心高可用涉及数据库、Nginx（可选）


## 3 原生实现特点与问题  

### 3.1 任务调度原生实现细节  

* 任务设置cron表达式并启动时，根据cron表达式计算任务下次调度时间戳（epoch milliseconds），并存到数据库中。
* 调度中心每秒读取数据库中“下次调度时间”在未来5s之内的任务,筛选条件即`trigger_next_time < System.currentTimeMillis() + 5000`，任务进入调度队列后，计算下次执行时间并更新数据。
* 如果`下次调度时间`早于当前时间，即`过期任务`。若过期超过5s，从当前时间计算下次执行时间；若过期小于5s，立即调度一次并从当前时间计算下次执行时间。  

![mission-table](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/mission-table.png)

### 3.2 原生实现如何保证多个调度中心节点不会重复调度任务

* 通过分布式锁保证多个调度中心节点不会重复调度任务
* 如果节点在执行临界区代码时宕机，数据库Connection断开，MySQL连接所对应的线程结束，与该线程相关的事务结束，即释放锁
* 粗粒度锁，实现简单  

![分布式锁流程图]()

![transaction](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/transaction.png)
上面为3个节点的调度中心集群，其中一个节点在临界区中执行任务调度逻辑，另外两个节点因事务阻塞而阻塞  


### 3.3 调度中心原生实现存在的问题

因为多种执行器路由策略（第一个、随机、轮询、哈希、LRU、分片等）的存在，执行器集群中各个节点可以得到充分利用  

但调度中心目前的任务调度实现方式存在以下问题：  
* 每次调度周期获取“计划执行时间”在未来5s内的任务，如果任务数量过多，节点可能无法在下一个调度周期前完成上一个周期的调度，即一个调度周期实际花费的时间超过5s，就有可能会产生过期任务。

    ![timeline]()
* 如果任务数量过多且线程池的任务队列使用有界队列，很大概率会使线程池执行拒绝策略，抛出java.util.concurrent.RejectedExecutionException，导致任务调度失败。如果切换为无界队列，可能会导致任务积压，引发一系列问题。  
    ![java-thread-pool](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/java-thread-pool-executor.jpg)
    ![executor-service](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/java-executor-service.png)同一时间任务过多时（任务执行时间过于集中），可能会导致线程池触发拒绝策略

* 由于分布式锁的存在任务调度多个节点串行化，不能充分利用调度中心集群中的其他节点资源。
    ![serialized]()




## 4 任务调度逻辑改进方案  




## 5 高可用架构改进  


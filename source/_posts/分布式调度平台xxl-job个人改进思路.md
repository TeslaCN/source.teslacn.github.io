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
![xxl-job单节点架构图](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/xxl-job%E5%8D%95%E8%8A%82%E7%82%B9%E5%9F%BA%E6%9C%AC%E6%9E%B6%E6%9E%84.svg)

## 2 xxl-job HA  

### 2.1 调度中心HA & 执行器HA  

前提：
* 每个执行器节点需要在配置中配置所有调度中心地址

缺点：
* 调度中心地址写死，不便于节点数量调整

![HA without Nginx](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/xxl-job-without-nginx.svg)

### 2.2 引入`Nginx`  

已有条件：  
* 调度中心通过数据库维护执行器信息，即执行器无须向所有调度中心注册，只须向其中一个节点注册  

引入后：  
* 实现负载均衡、故障迁移等  
* 修改Nginx配置即可调整节点  
* 执行器注册到调度中心走Nginx，调度中心调度任务仍然直接访问执行器  

![HA with Nginx](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/xxl-job-with-nginx.svg)

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

![分布式锁流程图](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/xxl-job%E8%B0%83%E5%BA%A6%E9%80%BB%E8%BE%91.svg)

![transaction](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/transaction.png)
上面为3个节点的调度中心集群，其中一个节点在临界区中执行任务调度逻辑，另外两个节点因事务阻塞而阻塞  


### 3.3 调度中心原生实现存在的问题

因为多种执行器路由策略（第一个、随机、轮询、哈希、LRU、分片等）的存在，执行器集群中各个节点可以得到充分利用  

但调度中心目前的任务调度实现方式存在以下问题：  
* 每次调度周期获取“计划执行时间”在未来5s内的任务，如果任务数量过多，节点可能无法在下一个调度周期前完成上一个周期的调度，即一个调度周期实际花费的时间超过5s，就有可能会产生过期任务。

    ![timeline](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/timeline.svg)
* 如果任务数量过多且线程池的任务队列使用有界队列，很大概率会使线程池执行拒绝策略，抛出java.util.concurrent.RejectedExecutionException，导致任务调度失败。如果切换为无界队列，可能会导致任务积压，引发一系列问题。  
    ![java-thread-pool](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/java-thread-pool-executor.jpg)
    ![executor-service](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/java-executor-service.png)同一时间任务过多时（任务执行时间过于集中），可能会导致线程池触发拒绝策略

* 由于分布式锁的存在任务调度多个节点串行化，不能充分利用调度中心集群中的其他节点资源。
    ![serialized](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/xxl-job-scheduler.svg)




## 4 任务调度逻辑改进方案  

### 4.1 任务调度逻辑修改方案

* __实现任务调度负载均衡__。维护一个调度中心在线节点列表，从0开始按顺序给每个节点分配一个数值作为分片值。任务被调度时，根据id（或其他算法）路由到不同的调度中心
* __更细粒度的锁__。调度任务时，通过CAS的方式更新任务“下次调度时间”，更新成功后再将任务推到调度队列；若CAS失败，则意味着任务可能已被其他节点调度（原因可能是调度中心在线节点数量新增或减少，任务路由也有所变动）
* __维护调度中心在线节点列表__。在应用层或MySQL Event Scheduler删除过期节点（可能宕机或断网没有正常注销 ）  

![xxl-job-improved](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/xxl-job-improved.svg)



## 5 高可用架构改进  

### 5.1 `Nginx`--调度中心集群唯一出口  

作为调度中心唯一出口，Nginx部署需要高可用方案

Keepalived
* VRRP（虚拟路由冗余协议）
* 管理LVS (Linux Virtual Server)
* 心跳检测、故障切换

Nginx + Keepalived 模式：
* 主备模式
* 互为主备

#### 5.1.1 Nginx + Keepalived 主备模式

* 正常情况下，Master节点对外服务，Backup节点处于空闲状态  
* 如果Backup收不到Master节点的心跳包，则认为Master节点出现故障，此时VIP(Virtual IP)切换到Backup节点对外提供服务

缺点
* Master节点正常工作时，Backup节点处于空闲状态，资源利用率低

![nginx-master-backup](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/nginx-master-backup.svg)主节点工作，备份节点空闲

![nginx-master-down](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/nginx-master-down.svg)主节点宕机，备份节点工作

#### 5.1.2 Nginx + Keepalived 互为备份  

* 正常情况下，多个节点对应多个VIP(Virtual IP)对外提供服务
* DNS能够提供简单的负载均衡服务（例如轮询）
* 如果某个VIP对应的节点出现故障，则将该VIP切换到其他正常工作的节点

![nginx-double-backup](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/nginx-double-backup.svg)两个节点正常工作，负载均衡

![nginx-double-master-down](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/nginx-double-master-down.svg)其中一个节点宕机，请求集中在另一个节点

### 5.2 `MySQL`高可用  
> 本人对MySQL的多节点部署不熟悉，此处仅仅简单提一下

* MySQL目前有很多集群方案可选，适用于各种场景。由于xxl-job框架使用MySQL实现分布式锁，如果不修改调度中心任务调度的实现，除了考虑一致性、可用性的同时，还需要考虑集群方案须支持调度中心实现分布式锁的方式。
* 个人理解，任务调度需要避免停机或遗漏，在CAP中满足P的情况下尽量提高A；xxl-job集群总体架构相对简明，选择MySQL集群方案时，在满足AP的情况下，优先考虑无侵入性、部署与维护相对简单的方案，避免调度系统技术复杂度过高  

常见MySQL集群方案  

1. MySQL (内建复制功能) + Keepalived
2. MHA (MySQL-Master-HA)
3. PXC (Percona XtraDB Cluster)
4. MyCat / Cobar 数据库中间件
5. MGR (MySQL Group Replication)

本人理解，方案1比较适合目前的场景。
1. 由于xxl依赖MySQL实现分布式锁，数据写入必须只在同一个数据库节点进行；
2. 调度中心每秒都会读取数据库，如果不能保证强一致性，可能会发生任务重复调用，在实际情况中，强一致性难以保证；  

综上2点，不做读写分离；  

1. 在保证P的情况下尽量提高A
2. 技术复杂度相对较低

MySQL + Keepalived
* Master节点正常工作时，持续发送心跳包到Backup节点
* Slave节点通过MySQL内建复制功能同步Master节点的数据，Master宕机时有可能
* 读写不分离，均落在单一节点
* Master节点宕机时，VIP将切换到Slave节点，Slave节点成为新的Master节点

![mysql-master-backup](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/mysql-keepalive.svg)主从同步，Binlog之类的方式

![mysql-master-down](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/mysql-master-down.svg)主节点宕机，备份节点工作。后续主节点恢复工作可能需要手动进行。


### 5.3 方案最终架构

![最终集群方案](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/%E5%88%86%E5%B8%83%E5%BC%8F%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E4%B8%AA%E4%BA%BA%E6%94%B9%E8%BF%9B%E6%80%9D%E8%B7%AF/%E6%9C%80%E7%BB%88%E9%9B%86%E7%BE%A4%E6%96%B9%E6%A1%88.svg)

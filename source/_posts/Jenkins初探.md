---
title: Jenkins 初探
date: 2019-10-30 08:56:31
tags:
- Jenkins
- CI/CD
categories:
- Jenkins
- CI/CD
---

# Jenkins 初探

> [Jenkins](https://jenkins.io)
> 是开源CI&CD软件领导者， 提供超过1000个插件来支持构建、部署、自动化， 满足任何项目的需要。

## 1 为什么选Jenkins

之前尝试过
* Git/GitHub Hook
* GitLab CI/CD


## 2 实践步骤
> 使用Jenkins配置一条简单流水线，并从GitHub拉取一个Golang项目代码、编译、构建Docker镜像、并部署到ECS

### 2.1 前置条件

1 安装并启动Jenkin。安装方式：
* [官网直接下载](https://jenkins.io/zh/download/)
* 包管理工具
* [Jenkins Docker](https://hub.docker.com/r/jenkins/jenkins)

2 安装Jenkins插件并完成配置：Docker plugin, Go Plugin

### 2.2 步骤

#### 2.2.1 新建任务

选择**构建一个自由风格的软件项目**
![新建任务]()

#### 2.2.2 配置源码管理、构建环境等

GitHub项目：[have-you-ordered](https://github.com/scau-ltd/have-you-ordered.git)

`源码管理`选择Git并输入仓库地址

`构建环境`勾上`Set up Go programming language tools`并选择相应版本

此处需要提前在系统设置中配置


#### 2.2.3 编写构建步骤

构建步骤：

1. 执行shell
```bash
go env -w GOPROXY=https://goproxy.cn,direct
go build cmd/orderserver/orderserver.go
```
2. Build / Publish Docker Image
![Build / Publish Docker Image]()

3. Run Docker Containers
![Run Docker Containers]()

#### 2.2.4 开始构建

控制台输出 Console Output  
![控制台输出]()

```

```
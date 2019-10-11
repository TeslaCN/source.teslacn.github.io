---
title: 基础镜像选择问题 | Alpine作为基础镜像的坑
date: 2019-10-11 22:48:06
tags: 
- Golang
- Docker
- Alpine
categories:
- Docker
- 踩坑
---

# 基础镜像选择问题 | `Alpine`作为基础镜像的坑

## 1 大致流程

* `alpine`镜像体积只有5MB，作为`Docker`下最小的Linux镜像，很适合打造一些轻量级镜像。但`alpine`底层使用`musl-libc`，兼容性与`glibc`有一定差距。

1. 用`Golang`编写了一个简单的hello, world程序，`Dockerfile`使用`alpine`作为基础镜像`FROM alpine:latest`能够正常运行；
2. 基于Gin框架Web应用，相似的Dockerfile文件，build完成后run的时候报了一些不明原因的错误，最后基础镜像改为`centos`或`alpine-glibc`后(`FROM centos:latest`)重新构建的镜像可以正常运行。

## 2 问题复现

### 2.1 使用`alpine`作为基础镜像构建Golang `hello, world`应用，`ENTRYPOINT`使用`sh`运行程序

#### 2.1.1 编写`hello, world` Dockerfile，基础镜像alpine

hello.go
```Golang
package main

import (
	"fmt"
	"os"
	"flag"
)

var config = flag.String("config", "go & docker", "[-config value]")

func main() {
	flag.Parse()
	fmt.Printf("os.Args: %s\n", os.Args)
	fmt.Printf("hello, %s\n", *config)
}
```

```dockerfile
FROM alpine:latest
ENV PARAMS=""
WORKDIR /
ADD hello /
ENTRYPOINT [ "sh", "-c", "./hello $PARAMS" ]
```
#### 2.1.2 构建镜像

`docker build -t hellogo:0.9 .`

```
Sending build context to Docker daemon  2.152MB

Step 1/5 : FROM alpine:latest
 ---> 961769676411
Step 2/5 : ENV PARAMS=""
 ---> Using cache
 ---> 3d8a4e790bbf
Step 3/5 : WORKDIR /
 ---> Using cache
 ---> f693abea2696
Step 4/5 : ADD hello /
 ---> Using cache
 ---> 156c71a0b934
Step 5/5 : ENTRYPOINT [ "sh", "-c", "./hello $PARAMS" ]
 ---> Using cache
 ---> c5d4aa11a311
Successfully built c5d4aa11a311
Successfully tagged hellogo:0.9
```
#### 2.1.3 运行

`docker run --rm hellogo:0.9`

输出结果
```
os.Args: [./hello]
hello, go & docker
```

由此可见，**运行hello, world没啥问题**

### 2.2 使用`alpine`作为基础镜像构建web应用，`ENTRYPOINT`使用`sh`运行程序

#### 2.2.1 编写Dockerfile，基础镜像alpine

```Dockerfile
FROM alpine:latest
WORKDIR /
ENV PARAMS=""
ADD orderserver /
ADD configs /configs
EXPOSE 60001
ENTRYPOINT ["sh", "-c", "/orderserver $PARAMS"]
```

#### 2.2.2 构建镜像

执行命令`docker build -t orderserver:build-with-alpine .`
```
Sending build context to Docker daemon  23.83MB

Step 1/7 : FROM alpine:latest
 ---> 961769676411
Step 2/7 : WORKDIR /
 ---> Running in 4e77f58acc2a
Removing intermediate container 4e77f58acc2a
 ---> 576bb22ca7c6
Step 3/7 : ENV PARAMS=""
 ---> Running in 7d601c97afad
Removing intermediate container 7d601c97afad
 ---> d1bbcbd11e4f
Step 4/7 : ADD orderserver /
 ---> e804183d1957
Step 5/7 : ADD configs /configs
 ---> 5f22f0e5e099
Step 6/7 : EXPOSE 60001
 ---> Running in 849525af1bad
Removing intermediate container 849525af1bad
 ---> 354f12aa7c3d
Step 7/7 : ENTRYPOINT ["sh", "-c", "/orderserver $PARAMS"]
 ---> Running in 4119e9b8543a
Removing intermediate container 4119e9b8543a
 ---> 40c84a5e3673
Successfully built 40c84a5e3673
Successfully tagged orderserver:build-with-alpine
```
#### 2.2.3 启动镜像

`docker run -P --rm --name orderserver0 -v /docker/data/have-you-ordered/configs:/mounted -e PARAMS="-config /mounted/config.json" orderserver:build-with-alpine`

`docker : sh: /orderserver: not found`

镜像因为不明原因报了找不到orderserver文件

### 2.3 使用`alpine`作为基础镜像构建，`ENTRYPOINT`直接执行目标二进制文件，有更详细的报错信息，但仍然难以排查原因

#### 2.3.1 编写`Dockerfile`，`ENTRYPOINT`直接执行目标程序

```Dockerfile
FROM alpine:latest
WORKDIR /
ENV PARAMS=""
ADD orderserver /
ADD configs /configs
EXPOSE 60001
#ENTRYPOINT ["sh", "-c", "/orderserver $PARAMS"]
ENTRYPOINT ["/orderserver"]
```

#### 2.3.2 build

`docker build -t orderserver:for-test .`


```
Sending build context to Docker daemon  23.83MB

Step 1/7 : FROM alpine:latest
 ---> 961769676411
Step 2/7 : WORKDIR /
 ---> Running in 94a464e58a2a
Removing intermediate container 94a464e58a2a
 ---> a2bd2111ff1a
Step 3/7 : ENV PARAMS=""
 ---> Running in d836703cad25
Removing intermediate container d836703cad25
 ---> 1beadaa8423a
Step 4/7 : ADD orderserver /
 ---> a10094c7eebb
Step 5/7 : ADD configs /configs
 ---> 6e80ce8fb8cd
Step 6/7 : EXPOSE 60001
 ---> Running in c70a05d12a5f
Removing intermediate container c70a05d12a5f
 ---> c9d7ea756259
Step 7/7 : ENTRYPOINT ["/orderserver"]
 ---> Running in 430fd5268cf0
Removing intermediate container 430fd5268cf0
 ---> 9bb02c756cd0
Successfully built 9bb02c756cd0
Successfully tagged orderserver:for-test
```


#### 2.3.3 运行镜像

报错信息：
`docker : standard_init_linux.go:211: exec user process caused "no such file or directory"`

与使用sh执行相比，报错信息更详细，不过貌似对问题排查仍然没什么帮助


### 2.4 使用`frolvlad/alpine-glibc`作为基础镜像构建并运行

#### 2.4.1 编写`Dockerfile`，基础镜像`alpine-glibc`

```Dockerfile
FROM frolvlad/alpine-glibc:latest
WORKDIR /
ENV PARAMS=""
ADD orderserver /
ADD configs /configs
EXPOSE 60001
ENTRYPOINT ["sh", "-c", "/orderserver $PARAMS"]
```

#### 2.4.2 构建镜像

`docker build -t orderserver:using-glibc .`

```
Sending build context to Docker daemon  23.83MB

Step 1/7 : FROM frolvlad/alpine-glibc:latest
 ---> e8076a77c225
Step 2/7 : WORKDIR /
 ---> Running in fc67e5d9e5fa
Removing intermediate container fc67e5d9e5fa
 ---> 2ced8d6cdaf2
Step 3/7 : ENV PARAMS=""
 ---> Running in fbb048b38832
Removing intermediate container fbb048b38832
 ---> bec33277d8a1
Step 4/7 : ADD orderserver /
 ---> 7df727ecfc51
Step 5/7 : ADD configs /configs
 ---> 5dc592e179c3
Step 6/7 : EXPOSE 60001
 ---> Running in 001fa109df4e
Removing intermediate container 001fa109df4e
 ---> e1a8e1f7a580
Step 7/7 : ENTRYPOINT ["sh", "-c", "/orderserver $PARAMS"]
 ---> Running in 06fc223499e5
Removing intermediate container 06fc223499e5
 ---> 0c819ac833c6
Successfully built 0c819ac833c6
Successfully tagged orderserver:using-glibc
```

#### 2.4.3 启动镜像

`docker run -P --rm --name orderserver0 -v /docker/data/have-you-ordered/configs:/mounted -e PARAMS="-config /mounted/config.json" orderserver:using-glibc`

看到一下信息代表Web应用能够正常启动了，底下报错只是因为没有配置Elasticsearch
```
2019/10/11 09:27:48 {{[http://es.0:49204]  } :60001 having-meal 1h}
os.Args: [/orderserver -config /mounted/config.json]
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)
[GIN-debug] GET    /api                      --> have-you-ordered/internal/app/orderserver.ApiHelloGo (3 handlers)
[GIN-debug] GET    /api/ordered/:date        --> have-you-ordered/internal/app/orderserver.ApiOrdered (3 handlers)
[GIN-debug] POST   /api/order                --> have-you-ordered/internal/app/orderserver.PostOrder (3 handlers)
[GIN-debug] POST   /api/fake-order           --> have-you-ordered/cmd/orderserver/app.Run.func1 (3 handlers)
[GIN-debug] GET    /api/dashboard/agg-by-day --> have-you-ordered/internal/app/orderserver.AggHistogram (3 handlers)
[GIN-debug] POST   /api/control/fetch        --> have-you-ordered/internal/app/orderserver.ManuallyFetch (3 handlers)
[GIN-debug] Listening and serving HTTP on :60001
2019/10/11 09:27:48 Start Fetch Interval: 3600 sec
2019/10/11 09:27:48 Error getting response: dial tcp 127.0.0.1:49204: connect: connection refused
2019/10/11 09:27:48 Fetch interval error: runtime error: invalid memory address or nil pointer dereference
```

## 3 总结

1. 选择基础镜像需要谨慎
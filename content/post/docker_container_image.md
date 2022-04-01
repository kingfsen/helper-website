---
title: "Docker容器与镜像结构"
date: 2019-04-21T10:19:15+08:00
description: "深入理解Docker容器与镜像结构"
thumbnail: "img/docker.jpg"
Tags:
- docker
Categories:
- docker
---

**容器与镜像概念**

Docker镜像是由一系列的层(Layer)组成的，每一个层对应的都是Dockerfile文件中的一个指令，Dockerfile中指令的行数越多，
镜像的层就会越多，镜像的每一层都是只读的，除了可写的容器层(容器运行时添加的一层)，例如下面的这个Dockerfile。

```Dockerfile
FROM ubuntu:15.04
COPY . /app
RUN make /app
CMD python /app/app.py
```

上面这个Dockerfile包含了四个指令，每个指令都创建了一层，每一层跟它之前的层均不同，这些层类似栈结构，新创建的层总是在栈顶端，当创建了一个Docker容器的时候，
将会增加一个新的可写层到栈顶，可写层并不会体现在Dockerfile中，而是通过`docker create`、`docker run`等命令创建容器时Docker自动添加的。可写层又称为容器层，
在一个运行的Docker容器中，创建新文件、删除文件、修改已存在的文件等操作都是在容器层进行的，容器就是镜像的一个运行实例而已。

{{< figure src="/blog/docker_data/002.jpg" >}}

容器与镜像主要的不同就在于顶层的可写层，所有的改变都发生在可写层，多个容器可以共享使用可写层下面的这些只读镜像层，下面这个图就展示了多个容器共享了同一份镜像层。

{{< figure src="/blog/docker_data/003.jpg" >}}

Docker通过存储驱动来管理镜像层和可写容器层的内容，每种存储驱动的实现方式可能有所不同，但所有的驱动都是使用镜像层叠以及copy-on-write策略，对于copy-on-write，
学习Java的应该都了解，因为JDK中有线程安全的`CopyOnWriteArrayList`类，适用于读多、写少的并发场景。

**容器大小**

我们可以通过命令`docker ps -s`查看容器在磁盘上的大约占用空间，其中与大小有关的字段`size`以及`virtual`。

```Docker
[root@q7osq ~]# docker ps -s
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                    NAMES               SIZE
c7e0e4971607        nginx:latest             "nginx -g 'daemon of…"   21 hours ago        Up 21 hours         80/tcp                   proxy               2B (virtual 109MB)
67a6c20ec7ed        wonderfall/isso:latest   "run.sh"                 3 weeks ago         Up 3 weeks          0.0.0.0:8080->8080/tcp   isso                0B (virtual 68.2MB)
```

- **size**
  
    可写层使用的磁盘空间
  
- **virtual**
  
    容器的镜像层加上可写层占用的磁盘空间，多个容器可能共享了部分或者全部镜像层，所以计算多个容器占用空间的时候，不能直接简单的将每个容器的`virtual`相加，
    因为这些容器共享了一份通用的镜像层。
  
所有运行中的容器占用的总磁盘空间是容器的`size`与`virtual`的一些组合计算，如果多个容器运行完全相同的镜像，那么这些容器占用的磁盘空间等于每个容器可写层`size`加上
一份镜像层大小(`virtual` - `size`)，结果=`SUM(size) + (virtual - size)`。

**copy-on-write策略**

cow是一种共享并复制的高效策略，在一个镜像中，如果一个文件或者目录存在于lower层中(类似栈结构)，那么一个upper层(包括可写层)需要读取，仅仅是读取已存在的这个文件或者目录。
在容器创建后者运行的时候，一个层首次需要修改这个文件，那么这个文件会直接从lower层复制到当前层并且修改它。

当通过`docker pull`拉取镜像时或者使用一个不在本地的镜像启动容器时，Docker会自动单独拉取镜像的每一层，并且存储到本地文件系统中，在Linux中一般就是`/var/lib/docker`目录。

```Docker
$ docker pull ubuntu:15.04

15.04: Pulling from library/ubuntu
1ba8ac955b97: Pull complete
f157c4e5ede7: Pull complete
0b7e98f84c4c: Pull complete
a3ed95caeb02: Pull complete
Digest: sha256:5e279a9df07990286cce22e1b0f5b0490629ca6d187698746ae5e28e604a640e
Status: Downloaded newer image for ubuntu:15.04
```

镜像的每一层都存储在Docker机器的指定目录，具体的存放目录与当前Docker采用的storage driver有关，比如当前主机安装的Docker采用的storage driver是aufs，那么层的存放路径为
`/var/lib/docker/aufs/layers/`。

```bash
$ ls /var/lib/docker/aufs/layers
1d6674ff835b10f76e354806e16b950f91a191d3b471236609ab13a930275e24
5dbb0cbe0148cf447b9464a358c1587be586058d9a4c9ce079320265e2bb94e7
bef7199f2ed8e86fa4ada1309cfad3089e0542fec8894690529e4c04a7ca2d73
ebf814eccfe98f2704660ca1d844e4348db3b5ccc637eb905d4818fbfb00a06a
```

从Docker1.10开始，镜像每一层的目录名与层的LayerID不一致，具体的镜像层的组织关系在后面的Docker存储驱动会详解介绍。


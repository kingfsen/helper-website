---
title: "Docker容器数据文件管理"
date: 2019-04-19T16:53:36+08:00
description: "Docker容器数据管理bind mount、volumes、tmpfs mount三种方式详解介绍"
Tags:
- docker
Categories:
- docker
---

在Docker容器内部创建的文件默认存储在可写的容器层，容易产生几个问题:

- 当容器不存在时，数据文件不能持久化，同时这些数据文件不方便在容器之外被其他进程使用。
- 当容器运行的时候容器可写层严重依赖宿主机，不能轻易移动这些数据文件到其他地方。
- 在容器层写数据文件需要存储驱动(storage driver)来管理文件系统，存储驱动使用Linux内核提供的联合文件系统，
与`data volumes`直接将文件写到宿主机文件系统相比，性能降低。

Docker为容器提供了两种方式将数据文件存储到宿主机上，即使容器停止运行或者被删除数据文件都可以持久化，这两种方式分别为
`volumes`与`bind mounts`，当然如果Docker容器在Linux运行，也可以使用`tmpfs mounts`。不管使用哪种mount方式，数据文件在Docker容器内部文件系统中是相同的，
要么是一个文件夹，或者一个独立的文件。



![docker_data](https://gitee.com/JunWuZhiJing/blog-images/raw/master/blog/docker_data/001.png)


- **volumes** 存储在Docker安装目录下，在Linux上默认指的是`/var/lib/docker/volumes`，Docker会自动管理，非Docker进程不应该修改这些文件系统，`volumes`是在Docker中是最好的数据持久化方式。
- **bind mount** 存储在宿主机文件系统的任何地方，宿主机上的非Docker进程或者Docker容器都可以在任何时候修改它。
- **tmpfs mount** 始终存储在宿主机系统内存中，不会被写到宿主机的文件系统中。


**详细介绍**

- **volumes** <br/>
	Docker会自动创建并管理`volumes`，当然可以通过命令`docker volumes create`明确的创建一个`volume`，当成功创建了一个volume时，它存储在宿主机的某个目录下，
	当把这个`volume`挂载到一个Docker容器时，这个目录自然会挂载到容器内部，`volumes`与`bind mount`工作方式类似，除了`volumes`是被Docker自动管理以及隔离性，
	两者没什么区别。
	
	```bash
	[root@ins ~]# docker volume create mm
	mm
	[root@ins ~]# docker volume create
	cc0613fe5a32273134a76e5670f166f6e248634e909d64cf00061130086f5ae5
	[root@ins ~]# docker volume ls
	DRIVER              VOLUME NAME
	local               cc0613fe5a32273134a76e5670f166f6e248634e909d64cf00061130086f5ae5
	local               mm
	[root@ins ~]# 
	```
	
	一个`volume`可以同时挂载到多个Docker容器，当没有任何Running状态的容器使用这个`volume`，这个`volume`仍然有效并且不会被自动删除，除非通过执行命令
	`docker volume prune`进行删除。当挂载一个`volume`时，这个`volume`可能匿名或者有一个名字，当它首次挂载到容器中的时候如果Docker发现该`volume`没有一个明确的名字，
	则会给它分配一个随机的名字，这个名字在Docker宿主机上是唯一的。
	
	`volumes`支持`volume driver`，允许通过driver将数据存储到远程机器或者云厂商等。

- **bind mounts** <br/>
	这种方式与`volumes`相比，有一些功能限制。当使用`bind mounts`时，宿主机上的一个文件或者目录被挂载到Docker容器中，这个文件或者目录通过它在
	宿主机上的完整路径名被引用，他们在宿主机上不是必须存在的，在需要的时候Docker会自动创建它。`bind mounts`非常高效，但是他们依赖于宿主机文件系统明确的目录结构，
	同时通过`Docker CLI`命令无法直接管理这些`bind mounts`。
	
	> * 注意：正在运行的容器中进程可以直接改变宿主机上的文件系统，包括创建、修改以及删除重要的文件或者目录，会引发安全风险问题，影响宿主机上运行的其它非Docker进程，请注意控制权限。
	
- **tmpfs mounts** <br/>
	这种方式不能将数据持久化到磁盘，一个`tmpfs`可以被一个容器在整个生命周期内使用，用于存储一些非持久状态或者敏感数据，比如swarm services使用`tmpfs`将`secrets`
	挂载到service的容器中。
	
`volumes`与`bind mounts`都能通过`-v` 或者 `--volume` flag参数挂载到容器中，对于 `tmpfs mount`，可以使用`--tmpfs` flag参数，在Docker17.06以及更高版本中，
推荐使用`--mount`，对于这三种方式`--mount`语法更明确。

**三种方式优点**

`volumes`可以在多个运行的容器之间共享，解耦Docker容器与宿主机文件系统，支持存储远程以及云厂商，方便在不同的Docker机器上迁移数据。

`bind mounts`可以让多个容器共享宿主机文件，比如Docker就是通过将/etc/resolv.conf挂载到每个容器方式实现DNS解决方案，开发的时候可以共享项目源代码，在容器内编译运行。

`tmpfs`保存敏感数据，非持久化数据，由于保存在内存中，相比文件系统性能更高。

> * 如果将一个空的`volumes`挂载到容器内的某个目录，如果该目录中已经有一些文件或者目录，那么这些文件或者目录会直接复制到`volumes`中。
> * 如果将一个`bind mount`或者非空volume挂载到容器的某个目录，这个目录中已经存在文件或者目录，那么这个目录中的文件或者目录会被mount覆盖，被覆盖的文件或者目录只是暂时被隐藏，当移除挂载时即可恢复。

**命令使用**

早期，Docker都是通过flag `-v`或者`--volume`给单机容器实现挂载，而swarm service则是通过 flag `--mount`实现，在Docker 17.06版本开始，`--mount`也适用于单机容器挂载，该命令的
语法更灵活明确，在使用`-v`或者`--volume`时尽量使用`--mount`代替。

- **-v or --volume**<br/>
  `-v db:/var/lib/mysql`，通过英文冒号分隔，如果volume有名字，冒号之前的是volume，如果volume匿名，则直接 `-v /var/lib/mysql`，如果需要控制容器读写volume权限，可以
  `-v db:/var/lib/mysql:ro`
  
- **--mount**<br/>
  `--mount`包含许多以英文逗号分隔的key-value键值对，它的语法比`-v`以及`--volume`更详细，key的顺序无关紧要，主要包含以下key。
  
    > * `type` 它的值可以为 `volume`、`bind`或者`tmpfs`
    > * `source` 对于已命名的volume，`source`即为volume名字，volume匿名，则该值为空，`source`也可定义为`src`
    > * `destination ` 指定挂载到容器中的path路径，可以定义为`dst`、`destination`或者`target`
    > * `readonly` 如果存在，则被挂载的volume在容器中只能读
    > * `volume-opt` 可选参数，可以定义多次，key-value形式
   
*volume使用*
 
创建一个volume

```Docker
docker volume create sunjinfu
```

查看volume

```Docker
docker inspect sunjinfu
[
    {
        "CreatedAt": "2019-04-20T15:00:12+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/sunjinfu/_data",
        "Name": "sunjinfu",
        "Options": {},
        "Scope": "local"
    }
]
```

移除volume

```Docker
docker volume rm sunjinfu
```

移除所有未使用的volume

```Docker
docker volume prune
```

将volume通过`-v`挂载到容器

```Docker
docker run -d --name proxy -v sunjinfu:/app nginx:latest
```

将volume通过`--mount`挂载到容器，注意逗号之间不要有空格

```Docker
docker run -d --name proxy --mount source=sunjinfu,target=/app nginx:latest
```

查看容器的Mount信息

```Docker
"Mounts": [
    {
        "Type": "volume",
        "Name": "sunjinfu",
        "Source": "/var/lib/docker/volumes/sunjinfu/_data",
        "Destination": "/app",
        "Driver": "local",
        "Mode": "z",
        "RW": true,
        "Propagation": ""
    }
]
```

volume driver

在当前宿主机启动容器，需要将volume数据写到可以通过ssh登录的另外一台机器97.82.193.73上，
这就需要volume driver支持，首先下载并安装volume driver `vieux/sshfs`

```Docker
docker plugin install --grant-all-permissions vieux/sshfs
```


查看plugin

```Docker
docker plugin ls
ID                  NAME                 DESCRIPTION               ENABLED
377c95d6f6fa        vieux/sshfs:latest   sshFS plugin for Docker   true
```

创建volume

```Docker
docker volume create --driver vieux/sshfs -o sshcmd=root@97.82.193.73:/data/test -o password=root123456 sshvolume
```

查看volume，其driver不再是local

```Docker
docker volume ls
DRIVER               VOLUME NAME
vieux/sshfs:latest   sshvolume
local                sunjinfu
```

创建容器挂载sshvolume

```Docker
docker run -d --name ssh-container --volume-driver vieux/sshfs \
--mount src=sshvolume,target=/app,volume-opt=sshcmd=root@97.82.193.73:/data/test,volume-opt=password=root123456 \
nginx:latest
```

进入容器在/app目录下新建一个文件ok.txt，然后登录97.82.193.73查看路径/data/test下是否存在。

```Docker
docker exec -it 4667aabfb5fd sh
# cd /app
# ls
# touch ok.txt
# ls
ok.txt
# exit
```

ssh root@97.82.193.73登录到机器查看路径/data/test发现ok.txt文件存在，volume driver正常工作。